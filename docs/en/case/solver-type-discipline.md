# Solver and Type Discipline

The solver explores Petri net reachability from the type declarations in rules. The correctness of the search results depends on whether the type declarations accurately reflect domain constraints. The solver is faithful to the declared world; constraints outside the declarations are not subject to inference.

This case study covers the technique of eliminating shortcuts (paths shorter than expected) returned by the solver through type refinement.

## Implicit Dependencies and Structural Guarantees

The execution order of procedural programs creates implicit dependencies. The following is typical of a procedure that "creates an account and issues a session":

```rust
fn create_account(&mut self, input: CreateAccountInput) -> Result<CreateAccountOutput, ...> {
    let did = self.generate_did(...)?;
    self.store_account(...)?;              // ← What if this fails?
    let tokens = self.issue_tokens(did)?;  // ← Tokens can still be issued even if store fails
    Ok(CreateAccountOutput { did, tokens })
}
```

`issue_tokens` depends on `store_account` succeeding, but that dependency rests on two implicit assumptions:

1. `store_account` executes before `issue_tokens` (line order).
2. Failure of `store_account` propagates as an error (the `?` operator).

Both depend on programmer attention. If the library has `store_account` fail silently without throwing an exception, `issue_tokens` would issue a token for a non-existent account.

The same constraint is expressed structurally in rule types:

| Procedural guarantee | Type-based guarantee |
|---|---|
| Line order puts store first | `issue_tokens` requires `account` |
| Error propagation via `?` | When store fails, `account` is absent from the marking; `issue_tokens` cannot fire |

**Execution order dependency is created by the absence of types.** Intermediate states that are not declared as types do not exist in the solver's search space. In procedural code, "having passed through that line" is the evidence of success. In rules, only "that type being present in the marking" serves as evidence.

## Interpreting Shortcuts

When the solver returns a path shorter than expected, it indicates a type deficiency.

| Symptom | Cause | Fix |
|---|---|---|
| Read and write return the same type | The fact of write completion does not exist as a type | Add a storage evidence type |
| Different operations fire with the same requires | Input type granularity is too coarse | Add operation-specific preconditions to requires |
| Protected resource reached without authentication | Capability not declared | Add capability to the rule |
| Goal reached in a single step | Intermediate state does not exist as a type | Chain intermediate outputs via produces/requires |
| Another endpoint's path cuts into the goal | Using a shared fact directly as the goal | Replace with semantic fact / completion token |

**A shortcut is a diagnostic of declaration deficiency, not a solver bug.**

## Convergence Stages

Iterative type refinement until the solver's path converges to one:

| Path count | Meaning | Next action |
|---|---|---|
| 0 | Unreachable. No producer present | Check missing facts and add rules |
| 1 | Converged. Types sufficiently reflect domain constraints | Done |
| 2+ | Branching. Multiple paths produce the same goal | Classify the cause using the table below |

Cause classification for branching:

| Cause | Example | Fix |
|---|---|---|
| Difference in path length | A 2-step path (handle → resolve → profile) and a 1-step path (did → profile) coexist | Solver picks the shorter. If only the longer is correct, the shorter's requires are insufficient |
| Insufficient granularity | Read and write return the same type | Separate the write evidence type |
| Missing precondition | Protected resource reached without authentication | Add capability |
| Shared fact contamination | Another endpoint produces the same `access-jwt` and cuts in | Replace with semantic fact |

The solver prioritizes the shortest path. When multiple paths of different depths coexist, the shorter path is likely an invalid shortcut that holds due to insufficient type constraints.

## Storage Evidence Types

### When read and write return the same type

If the operation that stores an account to the DB (`store_account`) and the operation that reads an account from the DB (`get_account`) both produce the same `account` type, the solver discovers a shortcut via the read operation.

In the context of "new creation," the fact that it was stored in the DB is required, and cannot be substituted by reading an existing record.

As a fix, add an evidence type specific to the store operation:

```kdl
rule "store_account" {
    requires output="did"
    produces output="account"
    produces output="account-record"    // storage evidence: obtainable only from store
}
```

By including `account-record` in the goal, the solver takes only the path through `store_account` as a solution. `get_account` produces `account` but not `account-record`, so the shortcut is eliminated.

## Semantic Facts and Completion Tokens

### Direct dependency on shared facts

Using shared facts such as `access-jwt`, `refresh-jwt`, `did`, and `session-output` directly as goals or in requires makes it easy for other endpoints or rules to cut into the shortest path.

For example, if both `createSession` and `refreshSession` produce `access-jwt`, then when verifying an endpoint with `access-jwt` as the goal, both paths appear and the intended path cannot be identified.

### Limiting paths with endpoint-specific facts

Rather than using shared facts directly, introduce two kinds of endpoint-specific facts.

**Semantic fact**: a fact representing the semantic completion of an operation. Converts shared facts into endpoint-specific context.

```kdl
// Using the shared fact (did) directly allows other endpoints to cut in
rule "generate_did" {
    requires output="handle"
    produces output="did"           // shared fact
    produces output="created-did"   // semantic fact: this endpoint's context
}
```

**Completion token**: a fact indicating that the entire endpoint has completed. Used as the goal for the endpoint.

```kdl
rule "issue_session_pair_from_account_record" {
    requires output="account-record"
    produces output="access-jwt"
    produces output="refresh-jwt"
    produces output="create-account-complete"   // completion token
}
```

### Operational Pattern

1. raw input / shared fact → convert to endpoint-specific token via request rule
2. Limit the requires of write / mutation rules to semantic facts only
3. For endpoint completion verification, specify the completion token as the goal

```
// createAccount path
validate_handle      : input → validated-handle
check_handle_unique  : validated-handle → handle-unique
generate_did         : handle-unique → created-did, account-create-ready
store_account        : created-did, account-create-ready → account-record
issue_session_pair   : account-record → create-account-complete
```

Because each rule's requires/produces connects via endpoint-specific facts, other endpoints' rules cannot cut into the path. The solver converges to this single 5-step path when searching with `create-account-complete` as the goal.

## Trust Boundary for External Dependencies

Storage evidence and semantic facts are what the solver verifies within the category: the type chain inside the category. The outside of the category (axiom) is trusted but not verified.

| Scope | Solver involvement | Example |
|---|---|---|
| category | Full enumeration of paths, proof of reachability | Type chain between rules, coverage of intermediate states |
| axiom | Trusted. Not verified | Internal implementation of imported libraries, external API responses, DB write guarantees |

You can declare that a rule in a category passes through an axiom operation using `composed-of`:

```kdl
rule "store_account" {
    requires output="created-did"
    requires output="account-create-ready"
    composed-of "datum.record.create"    // declares axiom dependency
    produces output="account"
    produces output="account-record"     // storage evidence
}
```

`composed-of` declares "this rule passes through an external operation." Whether `datum.record.create` is an INSERT into an RDB or a PUT into a KV store is not within the solver's responsibility. The solver's responsibility covers the category-internal structure: "to obtain `account-record`, there is no path other than through a rule that uses `datum.record.create`."

### Trust Levels and Type Design

At integration points with external systems, the trustworthiness of the response influences the type design.

| Trust level | Example | Type handling |
|---|---|---|
| Full trust | Pure computation within the same process | Produce directly via produces |
| Conditional trust | Return value of an imported library | Declare the library's type boundary as an axiom and bring the return type into the category |
| Requires verification | External API response | Insert a verification rule into requires. Do not trust the response directly |

If you treat an external API response directly as `produces`, subsequent rules can fire even when the API returns an invalid value. Inserting a verification rule creates a "verified response" type within the category, preventing unverified responses from proceeding.

## Guidance for Interpreting Solver Output

| Solver report | Interpretation | Action |
|---|---|---|
| Shortest path is shorter than expected | Type constraints insufficient; intermediate state is implicit | Add the missing intermediate type |
| Multiple paths coexist at different depths | The shorter path may hold due to a type deficiency | Inspect each step of the shorter path; check for missing requires that should be present |
| Another endpoint's rule cuts in | Using a shared fact as the goal | Use a completion token as the goal; replace direct dependency on shared facts with semantic facts |
| Unreachable | No producer, or requires are excessive | Use missing facts to identify the deficiency |
