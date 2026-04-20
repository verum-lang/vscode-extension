# Changelog

All notable changes to the Verum VS Code extension will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.3.0] - 2026-04-20

### Developer-tooling overhaul — full alignment with spec §§1-6

End-to-end integration with the `verum_lsp` and `verum_cli` crates
after the spec-alignment push in those packages. The extension now
activates every surface described in
`docs/detailed/25-developer-tooling.md` — hover, inlay hints, code
actions, dashboard commands, CBGR escape analysis — and every custom
`verum/*` JSON-RPC method backing those surfaces is live.

### Fixed

- **Startup restart-loop eliminated.** The LSP server used to advertise
  `execute_command_provider` for `verum.runFile` / `verum.runTest` /
  `verum.verifyFunction`, which the client already owned. tower-lsp's
  `ExecuteCommandFeature` registered those ids a second time and
  produced the fatal `command 'verum.runFile' already exists` error,
  retried once per second forever. The server no longer claims those
  commands; the inits loop is gone.
- **Stale `verum.lsp.serverPath`.** Bundled default was a hardcoded
  absolute path from an earlier project layout (`~/projects/.../axiom/
  target/release/verum`). Default restored to `"verum"` so it
  resolves through `$PATH` to `~/.cargo/bin/verum` after `cargo install
  --path crates/verum_cli`.
- **CBGR inlay-hints overlay on source.** The server emitted a ~50-char
  block-comment per reference (`/* can promote → &checked T: 0ns
  (saves ~15ns) */`) even in type positions (`fn f(p: &List<T>)`),
  visually shredding files like `term_compose_demo.vr`. See **Changed**
  below for the new behaviour.

### Added — hover, code actions, commands

- **Structured CBGR hover** on every `&` / `&mut` / `&checked` /
  `&unsafe` sigil. The bubble now reports *tier* (0 / 1 / 2),
  *mutability* (shared vs mutable borrow), *runtime cost* (~15ns /
  0ns per deref), *syntactic context* (value expression vs type
  position — the latter charges no runtime cost), and the *escape
  analysis verdict* with promotion availability. Implemented server
  side in `CbgrHintProvider::analyze_at_position` and
  `format_hover_markdown`; the extension's hover handler routes
  directly to it.
- **`verum.showEscapeAnalysis` command.** Triggered from the LSP code
  action *"View escape analysis details"*. Re-dispatches to
  `editor.action.showHover` at the reported position so the rich
  markdown report the server produces lands in the usual hover
  bubble. Added to `package.json` under the `Verum` category with the
  `$(graph)` icon; listed in the command-palette table in the docs.
- **Five custom `verum/*` JSON-RPC methods** now reachable from the
  extension:
  - `verum/validateRefinement` — SMT-backed validation at a position
    with counter-example.
  - `verum/promoteToChecked` — upgrade `&T` → `&checked T` with
    escape-proof comment.
  - `verum/inferRefinement` — infer the tightest refinement for a
    symbol from its usages.
  - `verum/getEscapeAnalysis` — structured CBGR report for a sigil.
  - `verum/getProfile` — compilation + runtime profiling summary.
  The four commands in the palette that previously emitted these
  requests (*Promote to &checked Reference*, *Add Runtime Check*,
  *Infer Refinement Type*, *Validate Refinement at Cursor*) stopped
  returning `MethodNotFound` — the caution block documenting that
  limitation has been removed.

### Added — settings surfaced

Three new `verum.lsp.*` knobs — the client already sent these through
`initializationOptions`, but their names weren't listed in the
settings UI:

- `verum.lsp.cacheTtlSeconds` (default `300`) — TTL of the validation
  cache; server hot-swaps at runtime via `workspace/didChangeConfig`.
- `verum.lsp.cacheMaxEntries` (default `1000`) — cache capacity;
  on downsize the oldest entries evict first.
- `verum.lsp.maxCounterexampleTraces` (default `5`) — cap on
  execution-trace steps attached to a counter-example.

### Changed

- **CBGR inlay-hint labels dramatically shortened.** Replaced
  50-character block comments with one-word badges: `0ns` for
  promotable references and `~15ns` for Tier-0 references that can't
  be promoted. All detail lives in the tooltip and the hover bubble.
  Hints are now context-aware — nothing is rendered in type positions
  or on `&checked` / `&unsafe` sigils.
- **`verum.cbgr.showOptimizationHints` default flipped `true` → `false`.**
  Inlay hints are opt-in; the hover bubble already shows the same
  information on demand. Turning them on is a deliberate "show me
  everything" mode for profiling sessions.
- **Server applies every `initializationOptions` key.** On `initialize`
  and on every `workspace/didChangeConfiguration` the server now reads
  thirteen client-side knobs and reshapes its internals accordingly —
  no server restart required for any of them. The settings currently
  plumbed: `enableRefinementValidation`, `validationMode`,
  `showCounterexamples`, `maxCounterexampleTraces`, `smtSolver`,
  `smtTimeout`, `cacheValidationResults`, `cacheTtlSeconds`,
  `cacheMaxEntries`, `cbgrEnableProfiling`,
  `cbgrShowOptimizationHints`, `verificationShowCostWarnings`,
  `verificationSlowThresholdMs`. The validation cache is resized in
  place; warm entries survive when headroom allows.

### Added — CLI companion (visible through tasks & dashboard)

The bundled task provider and the dashboard webview now call through
to the richer `verum verify` / `verum profile` surfaces that landed
in this release:

- `verum verify` gained `--profile`, `--budget DURATION`, `--export
  PATH`, `--distributed-cache URL`. Budget is enforced *per project*
  (not per file); remaining budget shrinks for each file, and the
  iterator stops once it is exhausted.
- `verum profile` gained `--all` (unified dashboard per spec §6),
  `--sample-rate PERCENT`, `--functions foo,bar` (exact-match filter
  applied upstream so every section agrees on the population),
  `--precision us|ns` with magnitude-aware formatting (`842ns` /
  `41.7µs` / `2.500ms`).
- `verum.toml [verify]` block supplies defaults for all of the above;
  CLI flags always win.

### Architecture note — Send-safe SMT isolation

The custom methods backed by Z3 are only reachable because the server
now runs a dedicated `verum-smt-worker` OS thread. The
`RefinementVerifier` (with its `Rc<ContextInternal>` and
`NonNull<_Z3_pattern>`) never leaves that thread; handlers talk to it
through a `Send + Sync` `SmtWorkerHandle` backed by `mpsc::sync_channel`
+ `tokio::sync::oneshot`. Only owned, Send-safe payloads cross the
await boundary, which lets tower-lsp's `.custom_method` register the
handlers — previously they failed the router's `Future: Send` bound
and fell through to `MethodNotFound`. See
[tooling/lsp.md → Custom verum/* JSON-RPC methods — architecture](../../website/docs/tooling/lsp.md).

### Removed

- The no-op `execute_command` handler on the server (see *Fixed* —
  there is no `execute_command_provider` to dispatch into).
- "Known limitations" admonition in the extension README section
  flagging four commands as `MethodNotFound`-returning; they work now.

## [0.2.0] - 2026-04-17

### Grammar audit — alignment with `grammar/verum.ebnf`

Full rewrite of the TextMate grammar, snippets, and language configuration
against the current EBNF spec. The prior packaging predated the unified
`type X is ...` syntax, the `mount` module statement, raw multiline strings
`"""..."""`, and the full set of proof / async keywords.

### Added

- **Keywords**: `throws`, `errdefer`, `nursery`, `select`, `defer`, `pure`,
  `provide`, `finally`, `recover`, `mount`, `protocol`, `implement`,
  `forall`, `exists`, `tactic`, and all proof combinators
  (`calc`, `have`, `show`, `suffices`, `obtain`, `qed`,
  `repeat`, `first`, `all_goals`, `focus`).
- **Strings**: triple-quoted raw multiline `"""..."""`, byte strings
  `b"..."`, full-featured interpolated strings `f"..."` / `f"""..."""`
  with `{expr:format}` spec highlighting, tagged interpolation
  `${expr}` inside tagged literals.
- **Tagged literals**: `json`, `json5`, `yaml`, `toml`, `xml`, `html`,
  `csv`, `sql`, `rx`, `re`, `regex`, `gql`, `graphql`, `cypher`,
  `sparql`, `url`, `uri`, `email`, `d`, `dur`, `date`, `sh`, `css`,
  `lua`, `asm`, `ip`, `cidr`, `b64`, `hex`, plus arbitrary user tags.
- **Attributes**: `@derive(...)` highlights constituent items;
  `@verify(strategy)` highlights all 9 semantic strategies
  (`runtime` / `static` / `formal` / `proof` / `fast` /
  `thorough` / `reliable` / `certified` / `synthesize`); generic
  `@name(args)` form with parameter-name / value / keyword
  scopes.
- **Snippets**: full set covering contracts, refinements, CBGR tier
  promotion, structured concurrency (`nursery` / `select`),
  `errdefer`, all 9 `@verify` strategies, all tagged-literal
  forms, proof declarations and tactics.
- **Refinement highlighting**: brace-delimited refinement predicates
  with explicit `self` subject and boolean-operator scoping.
- **Folding**: recognises `fn` / `type` / `protocol` / `module` /
  `implement` / `context` / `theorem` / `lemma` / `proof` /
  `tactic` / `nursery` / `select`.

### Changed

- **`mount` replaces `use` / `import`** — snippets and syntax now use
  the grammar's `mount path.to.module;` form (plus `.{A, B}` /
  `.* ` / `as alias` variants).
- **`implement` replaces `impl`** — in folding, indentation, and
  snippet prefixes (`impl`, `impl-generic`, `impl-where`).
- **`smtSolver` default** changed from `z3` to `auto` to match the
  language-level capability router (Z3 for LIA/bitvectors/arrays,
  CVC5 for strings/nonlinear/SyGuS, orchestrated by the compiler).
- **`assert_eq!` / `assert_ne!`** snippets dropped the Rust-style `!`
  — Verum has no macro-bang syntax, these are plain built-ins.
- **`Heap::new(x)` / `List::new()`** → `Heap(x)` / `List.new()`
  (Verum uses `.` as path separator; `Heap` is a constructor, not
  a type with `new`).

### Removed

- Rust-style keywords **`struct`**, **`enum`**, **`trait`**,
  **`newtype`**, **`ref`**, **`move`**, **`drop`**, **`cofix`**,
  **`view`**, **`private`**, **`priv`**, **`and`** / **`or`** /
  **`not`** (Verum uses `&&` / `||` / `!`), **`use`** / **`import`**
  (see `mount`).
- **`r#"..."#`** raw string syntax — the spec replaced this with
  triple-quoted `"""..."""`, which is always raw and multiline.
- **`#[...]`** hash attributes — Verum uses `@` exclusively.
- **`name!`** macro-invocation highlighting — Verum has no `!`
  macros.
- **Rust standard-library types** `Box`, `Rc`, `Arc` from the
  primitive-type list — forbidden in Verum per spec
  ("Semantic Honesty" — use `Heap`, `Shared`).
- Duplicate `problemMatchers` block in `package.json`.

### Fixed

- Refinement predicate variable corrected from `it` (not in EBNF)
  to `self` (spec-conformant).
- Hoist `@derive` / `@verify` / `@attribute-with-args` matchers so
  they anchor before generic attribute fallback — gives
  per-argument highlighting.
- Language configuration no longer auto-pairs `<` / `>` (avoided
  spurious closings on comparison operators); generics still
  surround-pair via `surroundingPairs`.
- Extension now activates on `workspaceContains:**/Verum.toml` so
  features light up as soon as a Verum project is opened, not only
  on the first `.vr` open.

## [1.0.0] - 2026-03-15

### Added
- Initial release of Verum Language Support for VS Code
- Syntax highlighting for `.vr` files with TextMate grammar
  - Keywords: fn, let, is, using, provide, context, protocol
  - Types: Int, Float, Bool, Text, List, Map, Set, Maybe, Result
  - Refinement types: Int{i > 0}, List{len(l) > 0}
  - Operators, comments, strings, numbers
  - Attributes: @verify(proof), @hint("split")
- LSP integration with `verum-lsp-server`
  - Real-time refinement type validation
  - Custom LSP methods: validateRefinement, promoteToChecked, inferRefinement
  - Incremental document synchronization
- Refinement validation features
  - On-type validation with 200ms debouncing
  - Concrete counterexample display
  - Multiple validation modes: quick, thorough, complete
  - SMT solver integration (Z3, CVC5)
- Quick fix actions (6 categories)
  - Runtime check wrapping (Result<T, E>)
  - Inline refinement constraints
  - Sigma type conversion
  - Runtime assertions
  - Refinement weakening
  - Promotion to &checked references
- Code action provider
  - Context-aware quick fixes
  - Refactoring suggestions
  - Performance optimization hints
- Inlay hints provider
  - Inferred refinement types
  - Type information display
  - Configurable visibility
- Commands
  - verum.promoteToChecked (Ctrl+Alt+C)
  - verum.addRuntimeCheck
  - verum.inferRefinement (Ctrl+Alt+R)
  - verum.validateRefinementAtCursor (Ctrl+Alt+V)
  - verum.showVerificationProfile
  - verum.showCbgrProfile
  - verum.restartLanguageServer
- Language configuration
  - Bracket matching
  - Auto-closing pairs
  - Comment toggling
  - Indentation rules
  - Folding markers
- Comprehensive settings
  - LSP server configuration
  - Validation mode selection
  - SMT solver settings
  - CBGR profiling options
  - Diagnostic customization
- Status bar integration
  - Server status indicator
  - Real-time connection status
  - Error notifications
- Output channel for debugging
  - LSP communication tracing
  - Server logs
  - Error messages

### Developer Features
- TypeScript implementation with strict mode
- ESLint configuration
- Comprehensive type definitions
- Modular architecture
  - extension.ts: Main extension logic
  - refinementValidator.ts: Validation engine
  - codeActionProvider.ts: Quick fix provider
  - inlayHintsProvider.ts: Type hints
- Build scripts
  - Compile: `npm run compile`
  - Watch mode: `npm run watch`
  - Package: `npm run package`
  - Publish: `npm run publish`
- Testing infrastructure
  - Unit tests
  - Integration tests
  - Extension host testing

### Documentation
- Comprehensive README.md
- Configuration guide
- Usage examples
- Troubleshooting section
- Development guide
- Changelog

## [Unreleased]

### Added
- **100+ Code Snippets** covering all Verum language constructs:
  - Function declarations (13 variants): `fn`, `async-fn`, `fn-using`, `fn-contract`, `meta-fn`
  - Type definitions (12 variants): `type`, `type-refined`, `type-sigma`, `newtype`, `type-tensor`
  - Protocols and implementations: `protocol`, `impl`, `impl-for`
  - Context system (7 snippets): `context`, `provide`, `using-group`
  - Verification constructs: `theorem`, `lemma`, `proof`, `forall`, `exists`
  - Control flow: `if`, `match`, `for`, `while`, `loop`, `try-catch`
  - Async patterns: `spawn`, `stream`, `select`
  - Testing: `test`, `test-async`, `bench`

- **Enhanced TextMate Grammar** with 57+ new keywords:
  - Async keywords: `async`, `await`, `spawn`, `stream`
  - Proof keywords: `theorem`, `axiom`, `lemma`, `corollary`, `proof`
  - Verification: `requires`, `ensures`, `invariant`, `decreases`
  - Tactics: `calc`, `have`, `show`, `induction`, `smt`, `qed`
  - Reference modifiers: `checked`, `unsafe`
  - Type modifiers: `tensor`, `affine`, `linear`, `newtype`
  - Tagged literals: `sql#"..."`, `rx#"..."`, `mat#[...]`, `vec#[...]`
  - Interpolated strings: `f"Hello, {name}!"`
  - Operators: `|>` (pipeline), `??` (null coalescing), `?.` (optional chain)

- **Semantic Token Types** for LSP-enhanced highlighting:
  - `refinementType`, `contextProvider`, `contextConsumer`
  - `protocol`, `theorem`, `lemma`, `tactic`
  - `cbgrReference`, `checkedReference`, `unsafeReference`
  - `taggedLiteral`, `contractKeyword`, `metaFunction`

- **Semantic Token Modifiers**:
  - `refinement`, `checked`, `unsafe`, `managed`
  - `pure`, `async`, `fallible`, `contextual`
  - `verified`, `proof`, `meta`, `linear`, `affine`

- **Enhanced Language Configuration**:
  - Colorized bracket pairs
  - Auto-closing for angle brackets (generics)
  - `onEnterRules` for doc comments (`///`, `//!`)
  - Folding markers for `fn`, `type`, `protocol`, `theorem`, `proof`
  - Improved indentation rules

- **Comprehensive Local Testing Documentation**:
  - Quick verification checklist
  - Snippet testing guide
  - Syntax highlighting test file
  - Troubleshooting section

### Now Implemented (LSP Production Quality Upgrade)
- Cross-file goto definition, find references, and rename via workspace index
- Completion resolve (deferred documentation lookup)
- CodeLens resolve (lazy reference counting)
- Delta semantic tokens for efficient incremental highlighting
- Deep selection ranges (word → expression → statement → block → function → file)
- Type hierarchy navigation (supertypes/subtypes via protocol impl blocks)
- Diagnostic pull model support
- Inline values for let bindings
- Progress notifications for workspace indexing
- On-type formatting for `=>` (match arms) and `|>` (pipeline alignment)
- Hover provider with type information
- Signature help for function calls
- Call hierarchy support (incoming/outgoing calls)
- Document symbols provider (outline view)
- Workspace symbols provider (global search)
- Definition provider (AST + syntax tree)
- References provider (AST-based categorized search)
- Rename provider (with validation and cross-file support)
- Format document provider (trivia-preserving)
- Code lens for reference counts, run/test/verify actions
- Performance dashboard in sidebar (CBGR profiling)

### Planned Features
- Debugger integration
- Interactive verification mode
- Proof assistant integration
- Git integration for verification caching
- Remote LSP server support
- Multi-root workspace support
- Notebook support for Verum
