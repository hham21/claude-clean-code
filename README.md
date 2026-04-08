# clean-code

```
    __       _                                      _      
   / /   ___| | ___  __ _ _ __         ___ ___   __| | ___ 
  / /   / __| |/ _ \/ _` | '_ \ _____ / __/ _ \ / _` |/ _ \
 / /   | (__| |  __/ (_| | | | |_____| (_| (_) | (_| |  __/
/_/     \___|_|\___|\__,_|_| |_|      \___\___/ \__,_|\___|
```

A Claude Code plugin that applies 11 clean code principles to any codebase. Language-agnostic, with parallel batch processing and automatic project tooling detection.

## What It Does

Dispatches parallel agents that refactor your code for readability — not formatting (that's what linters do), but **structural improvements** that linters can't catch:

- Renaming variables to describe *what*, not *how*
- Splitting 50-line functions into focused units
- Eliminating dead code and noise comments
- Extracting magic numbers into named constants
- Flattening deep nesting with guard clauses
- ...and 6 more principles

Each agent works on one file independently, preserving all public interfaces and functionality.

## The 11 Principles

| # | Principle | What it fixes |
|---|-----------|---------------|
| 1 | **Naming** | Vague names like `data`, `res`, `tmp` → descriptive names |
| 2 | **Function Structure** | Functions doing multiple things → single-responsibility, under 30 lines |
| 3 | **Stepdown Rule** | Random function ordering → top-down narrative (public first, helpers below) |
| 4 | **Parameter Count** | Functions with >4 params → options objects, boolean params → split functions |
| 5 | **Abstraction Level Consistency** | Mixed high/low-level code in one function → consistent abstraction |
| 6 | **Error Handling** | Deep nesting → early returns and guard clauses |
| 7 | **DRY Within File** | Code repeated 3+ times → shared helper functions (Rule of Three) |
| 8 | **Dead Code & Noise Removal** | Unused imports, commented-out code, obvious comments → removed |
| 9 | **Reduce Nesting** | Deeply nested logic → flattened with extracted functions |
| 10 | **Consistent Patterns** | Mixed styles within a file → unified patterns |
| 11 | **Magic Numbers/Strings** | `if (retries > 3)` → `if (retries > MAX_RETRIES)` |

### Why only 11?

Robert C. Martin's *Clean Code* contains 110+ principles across 17 chapters. This plugin selects the subset that meets two criteria:

1. **Single-file scope** — each agent works on one file in isolation without cross-file context. Principles that require understanding relationships between files (Law of Demeter, Feature Envy, Dependency Inversion) are excluded.
2. **Mechanically applicable** — the principle can be applied reliably by an AI agent without subjective judgment calls. Principles like Side Effect detection or Command-Query Separation require understanding intent, which leads to inconsistent results.

Architecture-level principles (SRP for classes, OCP, DIP), concurrency patterns, and test-specific heuristics are also out of scope — they require project-wide context that a single-file agent doesn't have.

## Install

```bash
claude plugins:add-marketplace hham21/claude-clean-code
claude plugins:install clean-code@claude-clean-code
```

## Usage

```
# Single file
/clean-code @src/auth.ts

# Folder
/clean-code @src/services/

# Entire project
/clean-code

# With exclusions
/clean-code @src/ exclude utils/legacy.ts

# Only specific principles
/clean-code @src/ only naming, dead code

# Skip principles
/clean-code @src/ skip parameter count, magic numbers
```

## How It Works

```
  /clean-code @src/
         |
         v
+------------------------+
|  Permissions check     |  <-- Recommends --dangerously-skip-permissions
+------------------------+
         |
         v
+------------------------+
|  Detect lint/build     |  <-- Reads package.json, pyproject.toml, etc.
+------------------------+
         |
         v
+------------------------+
|  Collect files         |  <-- git ls-files, excludes tests/config/generated
+------------------------+
         |
         v
+------------------------+
|  Batch 1: N agents     |  <-- Dynamically sized (up to 10 per batch)
|  Batch 2: N agents     |
|  ...                   |
+------------------------+
         |
         v
+------------------------+
|  Final verification    |  <-- lint + build + test check
+------------------------+
         |
         v
+------------------------+
|  Summary report        |  <-- Principle hit rates, high-change files
+------------------------+
```

## Sample Report

```
## Clean Code Summary
- Files processed: 16 | Modified: 12 | No changes needed: 3 | Failed: 1
- Total lines changed: +230 / -225

## Principle Hit Rate (how many files needed this fix)
- Naming: 12/16 (75%)
- Function Structure: 4/16 (25%)
- Dead Code Removal: 7/16 (44%)
- Magic Numbers: 2/16 (13%)
  ...

## High-Change Files (review recommended)
1. src/services/auth.ts (+38/-22)
2. src/utils/parser.ts (+29/-15)
3. src/api/handler.ts (+21/-12)

## Verification
- lint: PASS
- build: PASS
- test: PASS
```

## Safety

- **No functionality changes** — every edit is behavior-preserving
- **Public interfaces untouched** — exported functions, types, and APIs are never modified
- **Self-verification** — each agent re-reads the file and compares public interfaces before/after
- **Lint/build/test validation** — auto-detected project tools verify nothing broke
- **Silent failure retry** — agents that report changes but produce no diff are automatically retried
- **Resumable** — if interrupted, re-run and it picks up where it left off (batch mode)

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- `--dangerously-skip-permissions` recommended for batch mode
- Git repository (uses `git ls-files` for file discovery)

> **Token usage**: Batch mode dispatches a separate agent per file. A 40-file project can consume significant tokens. Start with a small folder to estimate costs before running on the full project.

## Auto-detected Verification Tools

Works with any language. The 11 principles are language-agnostic. When available, project lint/build tools are auto-detected for post-refactoring verification:

| Project Type | Detected Tools |
|-------------|----------------|
| Node.js | `package.json` scripts (lint, build, test, typecheck) |
| Python | ruff, flake8, mypy, pytest |
| Dart | `dart analyze`, `dart test`, `dart format` |
| Rust | `cargo clippy`, `cargo check`, `cargo test` |
| Go | `go vet`, `golangci-lint`, `go test` |
| Swift | `swiftlint`, `swift build`, `swift test` |
| Java/Gradle/Maven | project-specific lint, build, test |
| Make-based | `make lint`, `make check`, `make test` |
| None found | Principles applied without verification |

## License

MIT
