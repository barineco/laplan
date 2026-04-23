# IR パイプライン walkthrough

resolver.lex の具体関数を 1 本選び、Lex₀ から最終出力までの全段と、そこから inverse で戻ってくる復路を並べて追う文書です。個々の層の役割は [ir.md](ir.md) で総論として扱っており、本書はある関数が層を降りて言語コードになり、戻ってくる経路を通しで読むための指示台帳です。

## 本編: checkNeeds

`checkNeeds` は runtime resolver の 9 関数のうち最も短いものです。body が 1 行で、全 21 言語で roundtrip が成功するため、各層の対応を歪みなく観察できます。

### Lex₀: resolver.lex の KDL 宣言

`axiom/resolver.lex` には `checkNeeds` が以下のように書かれています。

```kdl
fn "checkNeeds" {
  param "goal" type="GoalSpec"
  return-type "CheckNeedsOutput"
  app "lang.checkNeedsForGoal" {
    $local.goal
  }
}
```

パラメータ 1 つ、返り値型 1 つ、body は 1 つの関数適用 `lang.checkNeedsForGoal($local.goal)` のみです。`$local.goal` は `var "local.goal"` ノードの糖衣で、FnExpr の `Var` variant に対応します。これが最も浅い表現で、KDL パーサを通って `parse_resolver_lex` に入った段階で Lex₁ 層に上がります。

### Lex₁: FnDef

`parse_resolver_lex` は KDL を `Vec<FnDef>` に変換します。`checkNeeds` に対応する値は概ね以下の形です。

```
FnDef {
  name: "checkNeeds",
  params: [("goal", "GoalSpec")],
  return_type: "CheckNeedsOutput",
  body: Algebraic(
    FnExpr::App("lang.checkNeedsForGoal", [FnExpr::Var("local.goal")]),
  ),
}
```

body は `FnBody::Algebraic(FnExpr)` で保持され、`FnExpr::App` が関数適用を示します。`local.` / `lang.` といった NSID prefix は Lex₁ 時点では残っており、emit の際に各言語の命名規則に従って除去されます。

### Lex₂: RuntimeFn

Lex₂ 経由の言語 (rust / python 等) では、Lex₁ の `FnExpr` を `lowering::fn_to_lex2` で `RuntimeFn` に降格します。`checkNeeds` のような単一 App 式は以下のような `Stmt` 列になります。

```
RuntimeFn {
  name: "checkNeeds",
  params: [("goal", "GoalSpec")],
  return_type: "CheckNeedsOutput",
  body: [
    Stmt::Return(
      Expr::Call("checkNeedsForGoal", [Expr::Var("goal")]),
    ),
  ],
}
```

body は `Vec<Stmt>` になり、末尾が `Stmt::Return` で閉じます。NSID prefix も emit 時に剥がされる形で準備されます。

### haskell 向け出力 (Lex₁ 直接 emit)

haskell は `functional {}` を持つため Lex₁ を直接 emit します。`template_engine_fn::fn_template_emit_runtime` が対応します。

```haskell
checkNeeds :: GoalSpec -> CheckNeedsOutput
checkNeeds goal =
  checkNeedsForGoal goal
```

`FnExpr::App` が haskell の space 適用にそのまま対応し、Lex₂ を経由しない分、関数型の式木がほぼそのまま出力に残ります。

### rust 向け出力 (Lex₂ 経由、brace 系)

rust は Lex₂ に降格してから `template_engine::template_emit_fn` で emit されます。以下は emit の原文そのままで、インデントはタブ 1 つです (rustfmt 整形前の template 直出力)。

```rust
pub fn checkNeeds(goal: GoalSpec) -> CheckNeedsOutput {
	return checkNeedsForGoal(goal);
}
```

body の `Vec<Stmt>` が brace ブロックに展開され、`Stmt::Return` が `return ...;` になります。単一 App 式は Lex₁ と Lex₂ のどちらから出発しても形が素直に残ります。

### python 向け出力 (Lex₂ 経由、indent 系)

python も Lex₂ 経由ですが、block 境界が indent で示されます。

```python
def checkNeeds(goal: GoalSpec):
    return checkNeedsForGoal(goal)
```

形は rust と同型ですが、`def` 宣言と indent の扱いが mapping.lex の `fn-open` / `indent` 設定で決まります。inverse 側が block 終端を検出するため、python の mapping.lex には `«dedent»` directive が配置されています (詳細は [mapping-template.md](../reference/mapping-template.md) を参照)。

### inverse: 戻り経路

rust の出力を source として inverse を走らせた場合、以下の順で層を逆に遡ります。

1. `ast_inverse::engine` の pattern matcher が `pub fn checkNeeds(...) { ... }` 全体を走査し、`MatchResult` の `consumed` バイト数と captures を返す
2. captures から `RuntimeFn` を組み立て、body を `Vec<Stmt>` として取り出す
3. `reverse_lower::reverse_lower` が body 全体 (`Stmt::Return` 1 つ) を `FnBody::Algebraic(FnExpr::App(...))` に復元する
4. 得られた `FnDef` が元の Lex₁ 表現と一致する

`checkNeeds` はこの往復が全 21 言語で成功する代表例です。どの段でも残余が発生せず、`Stmt::Raw` / `Expr::Raw` への fallback も起きません。

## 補章: classifyCandidate

`classifyCandidate` は `axiom/resolver.lex:50-138` にある関数で、case 分岐・filter・let-in が入れ子になります。全段を追いきるには紙幅が足りないため、本書では rust 向け出力を材料に「どこで Lex₁ 表現に畳み込みきれないか」を指さす形で扱います。

### 降格で崩れる構造

Lex₁ の body は case 節と filter と let-in を組み合わせた 1 つの式でしたが、Lex₂ に降格すると以下のように破片化します。

- case 節は If 連鎖へ展開される
- filter は `for` + `if` + 可変 list 操作に展開される
- let-in の各束縛は `Stmt::Let` / `Stmt::Mut` の列になる

これが forward 側で意図的に捨てる情報です。rust 向けの emit は以下のようになります (抜粋)。

```rust
pub fn classifyCandidate(marking: Marking, recipes: HashMap<String, RecipeEntry>,
    requirements: HashMap<String, RecipeRequirements>,
    acc: (Vec<SolveCandidate>, Option<BoundaryKind>), id: String)
    -> (Vec<SolveCandidate>, Option<BoundaryKind>) {
	let candidates = (acc).0;
	let lastBoundary = (acc).1;
	let requirement = requirements[id.as_str()].clone();
	let mut missingCap = vec![].to_owned();
	for capability in &requirement.requires_capability {
		if !(marking.contains_key(capability.as_str())) {
			missingCap = { let mut v = missingCap; v.push(capability.clone()); v };
		}
	}
	if !(missingCap.runtime_is_empty()) {
		return (candidates, Just(Some(BoundaryKind::Capability(missingCap))));
	} else {
		// ... (以下、recipes の有無で分岐するさらなる if-else)
	}
}
```

`$local.lastBoundary` は rust 出力では `lastBoundary` に、list の空判定は `runtime_is_empty` メソッド呼び出しに展開されています。case と filter が双方 `for` + `if` チェーンに展開され、let-in の束縛は `let` / `let mut` の列に開かれています。

### inverse で残余になりやすい箇所

この source を inverse に通すと、`Vec<Stmt>` までは pattern match が通っても、`reverse_lower` で Lex₁ 形に戻せない部分が残ります。典型的な候補は以下です。

- `{ let mut v = missingCap; v.push(capability.clone()); v }` のような block 式 (Lex₁ の `Concat` にはまとめられず、Lex₂ の Stmt 列として保持される)
- `requirements[id.as_str()].clone()` の `.as_str()` と `.clone()` の method 連鎖 (Lex₁ 側に `MethodChain` variant がなく、App に畳み込む適切な軸がないと Lex₂ のまま残る)
- `Just(Some(BoundaryKind::Capability(missingCap)))` の入れ子 Construct (Lex₁ の `Construct` で原理上は表現可能だが、source 側の省略記号や型脱落で失敗することがある)

これらに該当する Expr は `compiler/ir/src/reverse_lower.rs` の `lift_expr` で `None` を返す match arm に落ち、呼び出し側の `recognize_*` 系が body 全体の Algebraic 化を諦めて `FnBody::Procedural(Vec<Stmt>)` を選びます。結果として、この関数は inverse 経路の出口で Lex₁ ではなく Lex₂ のまま保持されます。

### 要約

`classifyCandidate` を全 21 言語で Lex₁ まで戻し切るには、pattern match 段で各言語の block 式 / method 連鎖 / 入れ子 Construct を構造化して拾い、lift 段で `MethodChain` 等を受け入れる arm を拡張する二点が揃う必要があり、どちらかが欠けると残余が Lex₂ に留まります。本書ではそこまで踏み込まず、上記 3 箇所を目印とします。

## 関連 docs

- IR 層の総論: [ir.md](ir.md)
- Lex 階層の位置付け: [../reference/layers.md](../reference/layers.md)
- mapping.lex の置換記法: [../reference/mapping-template.md](../reference/mapping-template.md)
- forward emit の実装: [synthesis.md](synthesis.md)
