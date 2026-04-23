# VSCode 拡張と Vue UI

`extension/` は VSCode 拡張として `.lex` のシンタックスハイライト、diagnostics、semantic tokens、inlay hints を提供します。Petri net エディタは `packages/ui-render` が生成する Vue 3 ビルドを webview に読み込む構成です。

## パッケージ構成

```
laplan/
├── packages/
│   ├── core/                @laplan/core (wasm-pack dual target)
│   ├── ui-model/            @laplan/ui-model (グラフ型、trait 界面)
│   ├── ui-render/           @laplan/ui-render (自前 SVG ノードエディタ)
│   ├── ui-store/            @laplan/ui-store (KDL 永続化)
│   ├── ui-web/              @laplan/ui-web (Vite SPA)
│   ├── ui-tauri/            @laplan/ui-tauri (Tauri v2)
│   ├── neco-view2d-wasm/    viewport の WASM binding
│   ├── neco-nodegraph/      neco-nodegraph の TS 参照
│   ├── neco-edge-routing/   neco-edge-routing の TS 参照
│   └── neco-view2d-svg/     neco-view2d-svg の TS 参照
└── extension/               VS Code 拡張 (thin adapter)
```

主要パッケージの責務:

| パッケージ | 役割 |
|---|---|
| `@laplan/core` | `diagnose_workspace` の WASM 提供 (web / nodejs dual target) |
| `@laplan/ui-model` | `PetriNetGraph`、`NeedsClassification` 等の TS 型と trait 界面 |
| `@laplan/ui-render` | Vue 3 自前 SVG ノードエディタ (`<PetriNetEditor>` を export) |
| `@laplan/ui-store` | `meta view.graph.*` の KDL 永続化と Pinia store |
| `@laplan/ui-web` | Vite SPA (ブラウザで `.lex` を読み込んで編集・エクスポート) |
| `@laplan/ui-tauri` | Tauri v2 (ネイティブ compiler 直リンク、ファイル watch、autosave) |
| `neco-view2d-wasm` | `neco-view2d` の wasm-pack binding (viewport pan/zoom) |

## ビルド

```bash
# WASM コアのビルド
pnpm --filter @laplan/core run build:web
pnpm --filter @laplan/core run build:node

# UI パッケージのビルド
pnpm --filter @laplan/ui-render run build

# VSCode 拡張のパッケージング
cd extension
npm run build:wasm    # @laplan/core の node/web target を更新
npm run compile       # tsc + packages/ui-render/dist/ を out/webview/ へコピー
npm run package       # .vsix を生成
```

成果物:

- `packages/core/dist/`: WASM + JS バインディング (web / nodejs 各ターゲット)
- `packages/ui-render/dist/`: webview として読み込む Vite ビルド出力
- `extension/out/webview/`: `packages/ui-render/dist/` のコピー
- `extension/laplan-lex-<version>.vsix`: 配布用パッケージ

## WebUI と Tauri

```bash
# ブラウザ SPA
pnpm --filter @laplan/ui-web run dev     # 開発サーバ起動
pnpm --filter @laplan/ui-web run build   # dist/ を生成 (静的ホスティング可)

# Tauri デスクトップアプリ
pnpm --filter @laplan/ui-tauri run tauri dev
pnpm --filter @laplan/ui-tauri run tauri build
```

## 拡張の構成

```
extension/
├── src/
│   ├── extension.ts            activation event + command 登録
│   ├── diagnostics.ts          WASM 呼出 → VSCode Diagnostic Provider
│   ├── graph/
│   │   ├── petriNetPanel.ts    WebviewPanel 管理
│   │   └── webviewHtml.ts      CSP nonce 挿入
│   ├── inlayHints.ts
│   ├── semanticTokens.ts
│   └── wasmLoader.ts
└── wasm/
    └── src/lib.rs              WASM API (extension/wasm crate)
```

`petriNetPanel.ts` は `out/webview/index.html` を読み込み、asset URI を書き換えて CSP nonce を注入してから `WebviewPanel` を作成します。webview の実体は `packages/ui-render` のビルド出力です。

`packages/ui-render/vite.config.ts` は `build.modulePreload = false` を指定しています。これで CSP 置換対象を `script` / `stylesheet` / inline `style` に限定できます。

## WASM API

`@laplan/core` の `diagnose_workspace` が公開 API です。

```ts
diagnose_workspace(files_json: string): string
```

返値 (JSON):

```ts
type WorkspaceReport = {
  diagnostics: Diagnostic[];
  lints:        Lint[];
  connections:  Connection[];
  graph: {
    transitions:  GraphTransition[];
    parallel_dag: ParallelDagData;
    subtypes:     Subtype[];
    cratis:       CratisEntry[];
  };
  needs: NeedsClassification[];
};
```

`NeedsClassification` は 5 層分類 (`Matched` / `MultiPath` / `PrunedByBoundary` / `PrunedByRefinement` / `MissingFact`) で、エッジ色と tooltip に対応します。詳細は [architecture/solver.md](../architecture/solver.md) を参照してください。

## host と webview のメッセージ

host → webview:

| メッセージ | 内容 |
|---|---|
| `updateGraph` | 最新の `PetriNetGraph` とベースソース |
| `updateMeta` | デコード済みの `view/graph/*.lex` メタビューファイル |
| `externalChange` | webview 外でワークスペースが変更されたことを通知 |

webview → host:

| メッセージ | 内容 |
|---|---|
| `openFile` | 対応する `.lex` を開く |
| `applyLexEdit` | `WorkspaceEdit` 経由で rule スタブの追加・削除を適用 |
| `applyMetaEdit` | `view/graph/<nsid>.lex` を作成または上書きして保存 |

`.lex` の保存で diagnostics が再実行されます。`view/graph/*.lex` の保存でメタビューが再読み込みされます。

## meta 永続化

ノードの座標・viewport 状態は `view/graph/<nsid>.lex` に保存します。本体 `.lex` には座標情報を含めません。

```
workspace/
  car/parse_v1.lex               本体 (ロジック)
  view/graph/car.parse_v1.lex    座標・viewport (meta)
```

`@laplan/ui-store` の `saveMetaView()` が KDL を生成し、`loadMetaView()` が復元します。型宣言は `axiom/view/` に置かれています。詳細は [reference/axiom/view.md](../reference/axiom/view.md) を参照してください。

## 機能一覧

| 機能 | 実装 |
|---|---|
| シンタックスハイライト | `syntaxes/` (TextMate) |
| diagnostics | `diagnose_workspace` → VSCode Diagnostic Provider |
| semantic tokens | WASM API 経由 |
| inlay hints | WASM API 経由で型注釈を挿入 |
| Petri net エディタ | `PetriNetPanel` + `packages/ui-render` Vue 3 build |
| コマンド: `laplan.showGraph` 等 | `extension.ts` |

## `.vsix` のインストール

```bash
code --install-extension extension/laplan-lex-0.2.0.vsix
```

バージョンは `extension/package.json` の `version` と同期します。

## トラブルシューティング

| 症状 | 対処 |
|---|---|
| `out/webview/` が空 | `pnpm --filter @laplan/ui-render run build` を実行してから `npm run compile` |
| extension がロードされない | `.vsix` の再インストール、VSCode 再起動 |
| diagnostics が更新されない | VSCode の `Developer: Reload Window` |
| Petri net が表示されない | webview の console を `Help > Toggle Developer Tools` で確認 |

## 関連

- [architecture/solver.md](../architecture/solver.md): diagnose / lint の中身
- [architecture/synthesis.md](../architecture/synthesis.md): WASM binding の生成
- [reference/axiom/view.md](../reference/axiom/view.md): `axiom/view/` の型宣言
