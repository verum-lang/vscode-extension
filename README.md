# Verum Language Support for VS Code

Official Visual Studio Code extension for the [Verum](https://verum-lang.org)
programming language — refinement-type validation, CBGR memory-safety
analysis, LSP integration, and end-to-end command palette / task
provider / dashboard surfaces.

**Current version**: `0.3.0` (2026-04-20).
See [`CHANGELOG.md`](./CHANGELOG.md) for the full release history.

## Features

### Syntax highlighting

Grammar-driven — the TextMate patterns are derived from
`grammar/verum.ebnf`, so what the editor colours is exactly what
the parser accepts:

- Refinement types: `Int{i > 0}`, `List<T>{len(self) > 0}`.
- CBGR reference tiers: `&T`, `&mut T`, `&checked T`, `&unsafe T`.
- Attribute families: `@verify(proof)`, `@hint("split")`,
  `@derive(...)` (member-by-member), `@device(...)`, `@inline(...)`.
- Tagged literals: `sql#"..."`, `rx#"..."`, `json#"..."`,
  `mat#[...]`, 30+ built-in tags plus user-defined.
- Interpolated strings: `f"Hello, {name:format}!"`, triple-quoted
  raw multiline `"""..."""`.
- Proof vocabulary: `theorem`, `lemma`, `proof`, `calc`, `have`,
  `suffices`, `qed`, plus tactic combinators.
- Semantic tokens from the LSP on top of the TextMate pattern for
  precise highlighting of CBGR references, protocols, refinement
  variables, etc.

### Code snippets (100+)

- **Functions** — `fn`, `async-fn`, `fn-using`, `fn-contract`,
  `meta-fn`.
- **Types** — `type`, `type-refined`, `type-sigma`, `newtype`,
  `type-tensor`.
- **Protocols** — `protocol`, `implement`, `implement-for`.
- **Context** — `context`, `provide`, `using-group`.
- **Verification** — `fn-requires`, `fn-ensures`, `theorem`,
  `lemma`, `proof`; all five shipping `@verify` strategies
  (`runtime` / `proof` / `compare` / `cubical` / `dependent`).
- **Control flow** — `if`, `match`, `for`, `while`, `loop`,
  `try-catch`, `errdefer`.
- **Async** — `spawn`, `stream`, `select`, `nursery`.
- **Testing** — `test`, `test-async`, `bench`.

### Refinement-type validation

- **On-type validation** through `verum/validateRefinement` with
  200 ms debouncing and concrete counter-examples.
- **Three latency budgets** via `verum.lsp.validationMode`:
  `quick` (<100 ms, default), `thorough` (<1 s), `complete`
  (unbounded — reserved for background CI/CD).
- **Cache** with client-controlled capacity (`cacheMaxEntries`) and
  TTL (`cacheTtlSeconds`); hot-swapped on
  `workspace/didChangeConfiguration` — **no server restart needed**
  for any setting change.

Example:

```verum
fn divide(x: Int, y: Int{i != 0}) -> Int {
    x / y
}

let result = divide(10, 0);
// ↑ Error: y = 0 does not satisfy `i != 0`
//   Counterexample attached to the diagnostic.
```

### Quick fixes

Six categories served through `verum/getQuickFixes` and the
standard `textDocument/codeAction` surface:

- `runtime_check` — wrap in `Result<T, E>` / `NonZero.try_from(y)?`.
- `inline_refinement` — add `{i != 0}` constraint to parameter.
- `sigma_type` — convert to dependent pair `(v, proof)`.
- `assertion` — insert `assert(...)` with panic.
- `weaken_refinement` — relax over-strict constraints.
- `promote_to_checked` — `&T` → `&checked T` with escape-proof
  comment (driven by `verum/promoteToChecked`).

### CBGR reference analysis

- **Structured hover** on any `&` / `&mut` / `&checked` / `&unsafe`
  sigil. The bubble reports tier, mutability, runtime cost,
  syntactic context (value expression vs type position — the
  latter doesn't charge any runtime cost), escape-analysis verdict,
  and whether the reference can be promoted to `&checked T`.
- **Inlay hints** (opt-in via `verum.cbgr.showOptimizationHints`)
  render compact badges next to references: `0ns` for promotable,
  `~15ns` for Tier-0 references that can't be promoted. Nothing in
  type positions, nothing on already-optimised `&checked` /
  `&unsafe` references. Full detail lives in the tooltip.
- **Escape-analysis panel** — the code-action *"View escape analysis
  details"* emits the `verum.showEscapeAnalysis` command, which
  repositions the cursor on the sigil and opens the hover bubble.

### Commands

| Command | Shortcut | Effect |
|---------|----------|--------|
| `Verum: Run Current File` | `Cmd/Ctrl+Shift+R` | `verum run <file>` in an editor terminal. |
| `Verum: Run Test` | — | `verum test <file> --filter <fn>` at the cursor. |
| `Verum: Verify Function Contracts` | — | Re-verify `@verify(...)` obligations under the cursor. |
| `Verum: Show Escape Analysis` | — | Shows the full CBGR escape-analysis report for the `&` sigil at the cursor. |
| `Verum: Promote to &checked Reference` | `Cmd/Ctrl+Alt+C` | Upgrade `&T` → `&checked T` with proof comment. |
| `Verum: Add Runtime Check (Result<T, E>)` | — | Wrap in a runtime fallback when escape analysis fails. |
| `Verum: Infer Refinement Type` | `Cmd/Ctrl+Alt+R` | Ask the server for a refinement suggestion. |
| `Verum: Validate Refinement at Cursor` | `Cmd/Ctrl+Alt+V` | One-shot SMT validation at the cursor. |
| `Verum: Profile Current File` | — | Run `verum profile --all --export=json`. |
| `Verum: Open Profile Dashboard` | — | Focus the webview. |
| `Verum: Format Document` | `Cmd/Ctrl+Shift+F` | Delegates to LSP formatter. |
| `Verum: Restart Language Server` | — | Kill and respawn `verum lsp`. |
| `Verum: Show Language Server Status` | — | Current state + crash count. |

All SMT-backed commands call through to dedicated custom JSON-RPC
methods (`verum/validateRefinement`, `verum/promoteToChecked`,
`verum/inferRefinement`, `verum/getEscapeAnalysis`,
`verum/getProfile`). The server isolates Z3 on a dedicated
`verum-smt-worker` OS thread so these methods are fully
Send-safe — see the [LSP architecture note](https://verum-lang.org/docs/tooling/lsp#custom-verum-json-rpc-methods--architecture).

### Tasks

The extension contributes a task provider (`Terminal → Run Task...`):

- `Verum: build`  → `verum build`
- `Verum: run`    → `verum run <active-file>`
- `Verum: test`   → `verum test`
- `Verum: check`  → `verum check`

### Debug adapter

- `verum.debug.defaultTier` — `"interpreter"` (VBC, full
  step/breakpoint support) or `"aot"` (LLVM, reduced).
- `verum.debug.dapServerPath` — override the DAP binary; empty
  falls back to `verum.lsp.serverPath`.

## Requirements

- **`verum` CLI** on `$PATH`. Install with
  `cargo install --path crates/verum_cli` (or from a release build).
  The extension runs `verum lsp` and `verum dap` as subprocess
  transports.
- **Z3** (or CVC5 — configurable via `verum.lsp.smtSolver`) for
  refinement validation. The `verum` binary links against whichever
  SMT backend was enabled at build time.
- **VS Code 1.75+**.

## Installation

### From the VS Code Marketplace

1. Open VS Code.
2. Extensions (`Cmd/Ctrl+Shift+X`).
3. Search for **"Verum Language"**.
4. Install.

### From VSIX

```bash
code --install-extension verum-language-0.3.0.vsix
```

### From source (for contributors)

```bash
cd internal/vscode-extension
npm install
npm run compile
code --install-extension .
```

## Configuration

All settings live under the `verum.*` namespace. The extension
forwards every one of them to the server through
`initializationOptions` on start and
`workspace/didChangeConfiguration` on change, so edits take effect
**without a server restart**.

### LSP

| Setting | Default | Meaning |
|---------|---------|---------|
| `verum.lsp.enable` | `true` | Master switch. |
| `verum.lsp.serverPath` | `"verum"` | Path to the `verum` binary (the LSP runs via `verum lsp`). |
| `verum.lsp.enableRefinementValidation` | `true` | SMT-backed refinement validation while typing. |
| `verum.lsp.validationMode` | `"quick"` | `quick` / `thorough` / `complete` latency budget. |
| `verum.lsp.showCounterexamples` | `true` | Attach concrete witness values to refinement errors. |
| `verum.lsp.maxCounterexampleTraces` | `5` | Cap on execution-trace steps attached to a counter-example. |
| `verum.lsp.showInlayHints` | `true` | Inline inferred types. |
| `verum.lsp.diagnosticDelay` | `200` (ms) | Debounce between keystroke and validation. |
| `verum.lsp.smtSolver` | `"auto"` | `auto` / `z3` / `cvc5`. `auto` lets the compiler's capability router choose. |
| `verum.lsp.smtTimeout` | `50` (ms) | Per-obligation SMT timeout. |
| `verum.lsp.cacheValidationResults` | `true` | Cache SMT results keyed by goal hash. |
| `verum.lsp.cacheTtlSeconds` | `300` | Cache entry TTL. |
| `verum.lsp.cacheMaxEntries` | `1000` | Cache capacity. On downsize the oldest entries evict first. |
| `verum.lsp.trace.server` | `"off"` | `messages` / `verbose` dumps LSP traffic to the output channel. |

### CBGR

| Setting | Default | Meaning |
|---------|---------|---------|
| `verum.cbgr.enableProfiling` | `false` | Enable CBGR per-deref profiling (~15 ns overhead). |
| `verum.cbgr.showOptimizationHints` | `false` | Opt-in inlay hints on every `&` reference. Off by default — the hover bubble already shows this information on demand. |

### Verification

| Setting | Default | Meaning |
|---------|---------|---------|
| `verum.verification.showCostWarnings` | `true` | Warn when an obligation exceeds the slow threshold. |
| `verum.verification.slowThresholdMs` | `5000` | Slow-verification threshold. |

See [`package.json`](./package.json) for every knob, including
code-lens, formatting, semantic-highlighting, and debug settings.

## Troubleshooting

### Language Server Not Starting

1. Verify `verum` is on `$PATH`:
   ```bash
   which verum
   # should print ~/.cargo/bin/verum (or wherever it was installed)
   ```
2. If it isn't, either install with `cargo install --path
   crates/verum_cli` or point `verum.lsp.serverPath` at the
   absolute path of your build.
3. Check the output channel: `View → Output → Verum Language
   Server` — server traces and crash reports land there.
4. `Command Palette → "Verum: Restart Language Server"` re-launches.

### Validation feels slow

- Keep `verum.lsp.validationMode` at `"quick"` (the default) while
  editing — switch to `"thorough"` only for on-save runs.
- Increase `verum.lsp.diagnosticDelay` (default 200 ms) if you type
  very fast and the solver can't keep up.
- Raise the cache: `verum.lsp.cacheMaxEntries` to `4000`,
  `cacheTtlSeconds` to `1800`.
- Reduce the counter-example trace depth:
  `verum.lsp.maxCounterexampleTraces = 2`.

### SMT solver errors

- Confirm the solver is bundled in your `verum` binary:
  `verum smt-info`.
- Tune `verum.lsp.smtSolver` — `auto` is the safe default;
  `z3` / `cvc5` force a specific backend.

## Development

### Prerequisites

- Node.js ≥ 18, npm ≥ 9.
- VS Code ≥ 1.75.

### Build & run

```bash
cd internal/vscode-extension
npm install
npm run compile
```

To iterate on the extension itself:

1. `npm run watch` (continuous TypeScript compilation).
2. Open `internal/vscode-extension` in VS Code.
3. Press `F5` to launch the Extension Development Host.

### Tests

```bash
npm test          # full VS Code-integration suite
npm run lint      # ESLint
npm run compile   # type-check only
```

### Directory layout

```
internal/vscode-extension/
├── src/
│   ├── extension.ts              # entry point, activates the LSP client
│   ├── refinementValidator.ts    # on-type validation wrapper
│   ├── codeActionProvider.ts     # quick-fix code-actions
│   ├── inlayHintsProvider.ts     # CBGR / type inlay hints
│   ├── dashboardWebview.ts       # profile dashboard
│   └── debugAdapter.ts           # DAP descriptor factory
├── syntaxes/verum.tmLanguage.json
├── snippets/verum.json
├── language-configuration.json
└── package.json
```

### Packaging

```bash
npm run package                    # produces verum-language-0.3.0.vsix
code --install-extension verum-language-0.3.0.vsix
```

## Contributing

Contributions welcome — see
[`CONTRIBUTING.md`](../../CONTRIBUTING.md) for the workflow.

## License

MIT — see [`LICENSE`](./LICENSE).

## Links

- [Verum Language documentation](https://verum-lang.org/docs)
- [Tooling → VS Code extension](https://verum-lang.org/docs/tooling/vscode-extension)
- [Tooling → LSP](https://verum-lang.org/docs/tooling/lsp)
- [Tooling → CLI](https://verum-lang.org/docs/tooling/cli)
- [Issue tracker](https://github.com/verum-lang/verum/issues)

See [`CHANGELOG.md`](./CHANGELOG.md) for the full release history.
