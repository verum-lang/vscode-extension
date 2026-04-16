# Verum Language Support for VS Code

Official Visual Studio Code extension for the Verum programming language, providing rich IDE support including refinement type validation, CBGR profiling, and LSP integration.

## Features

### 1. Syntax Highlighting
- Full syntax highlighting for Verum language constructs (100+ patterns)
- Support for refinement types (e.g., `Int{i > 0}`)
- Highlighting for CBGR reference tiers (`&T`, `&checked T`, `&unsafe T`)
- Attribute highlighting (`@verify(proof)`, `@hint("split")`)
- Tagged literals (`sql#"..."`, `rx#"..."`, `mat#[...]`)
- Interpolated strings (`f"Hello, {name}!"`)
- Formal proof keywords (`theorem`, `lemma`, `proof`, tactics)
- Semantic token support for enhanced highlighting

### 2. Code Snippets (100+)
Comprehensive snippets for rapid development:
- **Functions**: `fn`, `async-fn`, `fn-using`, `fn-contract`, `meta-fn`
- **Types**: `type`, `type-refined`, `type-sigma`, `newtype`, `type-tensor`
- **Protocols**: `protocol`, `impl`, `impl-for`
- **Context**: `context`, `provide`, `using-group`
- **Verification**: `fn-requires`, `fn-ensures`, `theorem`, `lemma`, `proof`
- **Control flow**: `if`, `match`, `for`, `while`, `loop`
- **Error handling**: `try-catch`, `result-match`, `option-match`
- **Async**: `async-fn`, `spawn`, `stream`, `select`
- **Testing**: `test`, `test-async`, `bench`

### 3. Real-Time Refinement Validation
- **On-type validation**: Get immediate feedback as you type
- **Concrete counterexamples**: See actual values that violate constraints
- **Debounced validation**: 200ms delay to avoid overwhelming the SMT solver
- **Multiple validation modes**: Quick (<100ms), Thorough (<1s), Complete (unlimited)

Example:
```verum
fn divide(x: Int, y: Int{i != 0}) -> Int {
    x / y
}

let result = divide(10, 0);  // Error: Counterexample: y = 0
```

### 4. Quick Fixes
Six categories of automated code fixes:
- **Runtime check wrapping**: `NonZero.try_from(y)?`
- **Inline refinement**: Add constraint to parameter type
- **Sigma type conversion**: Use dependent pairs
- **Runtime assertion**: Insert `assert!()` statements
- **Weaken refinement**: Relax constraints
- **Promote to &checked**: Convert to checked references

### 5. Inlay Hints
- Display inferred refinement types inline
- Show type information for complex expressions
- Configurable visibility

### 6. Code Actions
- Context-aware refactoring suggestions
- Performance optimization hints
- CBGR tier promotion recommendations

### 7. Commands
- **Promote to &checked** (Cmd/Ctrl+Alt+C): Convert references to zero-overhead checked references
- **Infer Refinement** (Cmd/Ctrl+Alt+R): Auto-generate refinement constraints
- **Validate Refinement** (Cmd/Ctrl+Alt+V): Force validation at cursor
- **Show Verification Profile**: Display verification performance statistics
- **Show CBGR Profile**: Display memory safety overhead analysis

## Requirements

- **Verum LSP Server**: The extension requires `verum-lsp-server` to be installed and available in your PATH.
- **SMT Solver**: Z3 or CVC5 for refinement validation (automatically used by LSP server)

## Installation

### From VS Code Marketplace
1. Open VS Code
2. Go to Extensions (Cmd/Ctrl+Shift+X)
3. Search for "Verum Language"
4. Click Install

### From VSIX
```bash
code --install-extension verum-language-1.0.0.vsix
```

### From Source
```bash
cd editors/vscode
npm install
npm run compile
code --install-extension .
```

## Configuration

### Basic Settings
```json
{
  "verum.lsp.enable": true,
  "verum.lsp.enableRefinementValidation": true,
  "verum.lsp.validationMode": "quick",
  "verum.lsp.showCounterexamples": true,
  "verum.lsp.diagnosticDelay": 200
}
```

### Advanced Settings
```json
{
  "verum.lsp.smtSolver": "z3",
  "verum.lsp.smtTimeout": 50,
  "verum.lsp.cacheValidationResults": true,
  "verum.cbgr.enableProfiling": false,
  "verum.verification.showCostWarnings": true
}
```

See [package.json](./package.json) for all available settings.

## Usage

### Writing Code with Refinements
The extension provides real-time feedback when writing refinement types:

```verum
// Good: Refinement satisfied
fn safe_divide(x: Int, y: Int{i != 0}) -> Int {
    x / y
}

// Error: Counterexample shown
fn main() {
    let divisor = 0;
    safe_divide(10, divisor);  // Counterexample: divisor = 0
}
```

### Quick Fixes
1. Place cursor on error
2. Press `Cmd/Ctrl+.` or click the lightbulb
3. Select from available fixes:
   - Wrap with runtime check (safe)
   - Add inline refinement (breaking)
   - Add assertion (safe)

### Optimizing Performance
The extension can suggest CBGR optimizations:

```verum
// Before: 15ns overhead per check
fn process(data: &List<Int>) { ... }

// Quick fix suggestion: Promote to &checked
fn process(data: &checked List<Int>) { ... }  // 0ns overhead
```

### Viewing Profiles
- **Verification Profile**: Shows SMT solver costs, slow verifications, cache statistics
- **CBGR Profile**: Shows memory safety overhead, optimization opportunities

## Troubleshooting

### Language Server Not Starting
1. Verify `verum-lsp-server` is in your PATH:
   ```bash
   which verum-lsp-server
   ```
2. Check the output channel: View → Output → Verum Language Server
3. Try restarting the server: Command Palette → "Verum: Restart Language Server"

### Slow Validation
1. Reduce validation mode to "quick" in settings
2. Increase diagnostic delay (default 200ms)
3. Disable caching if it causes issues

### SMT Solver Errors
1. Ensure Z3 or CVC5 is installed
2. Check solver timeout settings
3. Review SMT solver logs in output channel

## Development

### Prerequisites
- **Node.js**: 18.0.0 or higher
- **VS Code**: 1.75.0 or higher
- **npm**: 9.0.0 or higher

### Building from Source
```bash
cd editors/vscode
npm install
npm run compile
```

### Running in Development Mode (F5 Launch)
1. Open `editors/vscode` folder in VS Code
2. Press **F5** to launch Extension Development Host
3. A new VS Code window opens with the extension loaded
4. Open any `.vr` file to test syntax highlighting
5. Changes to TypeScript files require reloading the Extension Development Host

### Watch Mode for Active Development
```bash
# Terminal 1: Start TypeScript compiler in watch mode
npm run watch

# Then press F5 in VS Code to launch Extension Development Host
# After making changes, reload the host with Cmd/Ctrl+R
```

### Running Tests
```bash
# Run all tests (requires VS Code to be installed)
npm test

# Run linting only
npm run lint

# Type check only
npm run compile
```

**Note**: Tests run in a special VS Code instance. Ensure no other VS Code instances have the extension loaded.

### Testing Without LSP Server
The extension works for syntax highlighting and snippets without the LSP server. For full functionality:
1. Build the LSP server: `cd crates/verum_lsp && cargo build --release`
2. Add to PATH: `export PATH="$PWD/target/release:$PATH"`
3. Or set in VS Code settings: `"verum.lsp.serverPath": "/path/to/verum-lsp-server"`

### Packaging
```bash
# Create .vsix package
npm run package

# Install the package
code --install-extension verum-language-1.0.0.vsix
```

### Directory Structure
```
editors/vscode/
├── src/
│   ├── extension.ts           # Main entry point
│   ├── refinementValidator.ts # Validation engine
│   ├── codeActionProvider.ts  # Quick fix provider
│   └── inlayHintsProvider.ts  # Type hints
├── syntaxes/
│   └── verum.tmLanguage.json  # TextMate grammar (100+ patterns)
├── snippets/
│   └── verum.json             # Code snippets (100+ snippets)
├── src/test/
│   ├── runTest.ts             # Test runner
│   └── suite/
│       ├── index.ts           # Test suite loader
│       ├── extension.test.ts  # Extension tests
│       ├── refinementValidator.test.ts
│       └── dashboard.test.ts
└── package.json               # Extension manifest
```

## Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](../../CONTRIBUTING.md) for guidelines.

## License

MIT License - see [LICENSE](../../LICENSE) for details.

## Links

- [Verum Language Documentation](https://verum-lang.org/docs)
- [Language Specification](../../docs/detailed/)
- [LSP Server Documentation](../../crates/verum_lsp/README.md)
- [Issue Tracker](https://github.com/verum-lang/verum/issues)

## Changelog

### 1.0.0 (Initial Release)
- Syntax highlighting for `.vr` files
- LSP integration with refinement validation
- Real-time counterexample display
- Quick fix actions (6 categories)
- Inlay hints for type inference
- Commands for optimization and profiling
- Comprehensive configuration options
