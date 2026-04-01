---
name: clean-code
description: Use when applying clean code refactoring to one or more files. Accepts a single file, a folder (processes source files recursively), or no argument (processes entire project). Orchestrates parallel clean-code-single-file agents in batches of 10, auto-detects project lint/build tools, tracks progress via progress.json, verifies actual file modifications via git diff, and retries silently failed agents. Works with any programming language. Trigger on "clean code", "refactor", "code cleanup", "improve readability", "코드 정리", "클린코드", "리팩토링".
---

# Clean Code Orchestrator

Orchestrates clean code refactoring using the `clean-code-single-file` agent. Handles single files directly via Direct mode, or uses Batch mode for folders and projects with progress tracking and verification.

## Pre-flight: Permissions Check

Before anything else, use AskUserQuestion (in the user's language) to confirm:

- This skill modifies source files via sub-agents. For smooth operation, `--dangerously-skip-permissions` is recommended.
- Ask: "Is `--dangerously-skip-permissions` enabled? (A) Yes, proceed / (B) No, stop so I can restart with the flag"
- If A → continue to Input Modes
- If B → output `claude --dangerously-skip-permissions` and stop execution

This check happens once at the very start, before determining Direct or Batch mode. Do NOT create a Task for this step.

## Input Modes

Determine the mode from user input:

| Input | Mode | Behavior |
|-------|------|----------|
| Single source file path | **Direct** | Dispatch one agent, verify, done |
| Folder path | **Batch** | Collect source files under folder, batch process |
| No path / "all" | **Batch** | Collect all source files in project |

### Exclude Option

User can specify files or folders to exclude (e.g., "exclude tests/", "config/ 제외해줘"). Parse these from the user's message and add them to the default exclude list in file discovery.

## Direct Mode

Create these Tasks and execute in order:

### Task #1: Detect verification tools

Search the project root for build/lint configuration (see "Project Verification Tool Detection" below). Store the detected commands.

### Task #2: Dispatch agent and wait for completion

```
Agent("Apply clean code refactoring to <file_path>. Formatter: <detected_formatter>. Verification commands: <detected_lint_command>, <detected_build_command>",
      subagent_type="clean-code:clean-code-single-file")
```

If no formatter or verification commands were detected, omit them from the prompt.

### Task #3: Verify results

1. Run `git diff --name-only` to confirm the file was actually modified
2. If verification commands exist, run them from the project root
3. If errors found, attempt to fix (2 attempts max)
4. Report result to user

## Batch Mode

Create Tasks dynamically and execute in order:

### Task #1: Detect verification tools

Same as Direct mode Task #1. Detect once, pass to all agents.

### Task #2: Collect target files via git ls-files

Run `git ls-files` to collect source files. If the user specified a path, scope to that path.

**Exclude patterns** (filter out from results):
- Test files: paths containing `test`, `spec`, `__tests__`
- Config files: `*.json`, `*.yaml`, `*.yml`, `*.toml`, `*.config.*`
- Generated files: `*.min.*`, `*.generated.*`, `*.d.ts`
- Lock files: paths containing `lock`
- Non-source files: `*.md`, `*.txt`, `*.csv`, `*.svg`, `*.png`, `*.jpg`
- User-specified exclusions

Sort files by path for deterministic ordering. Store the file list.

### Task #3: Create progress.json

Create `.clean-code/progress.json`:

```json
{
  "target": "<path or 'project'>",
  "totalFiles": 42,
  "batchSize": 10,
  "completed": [],
  "failed": [],
  "currentBatch": 0,
  "verificationCommands": {
    "lint": "<detected or null>",
    "build": "<detected or null>"
  },
  "status": "in_progress"
}
```

If resuming from a previous session (progress.json already exists), read it and skip to the next incomplete batch.

### Task #3.5: Collect baseline errors (before any changes)

If verification commands were detected, run them now **before any refactoring** and store the output. This baseline is passed to each agent so they can distinguish pre-existing errors from errors they introduced. This prevents agents from wasting time reporting or trying to fix errors that existed before the refactoring.

### Task #4..N: Batch dispatch and verify (repeated)

**Dynamic batch sizing:** Instead of fixed batches of 10, distribute files evenly across batches. Calculate: `batch_size = ceil(totalFiles / ceil(totalFiles / 10))`. For example, 16 files → 8+8 instead of 10+6. This ensures balanced workload across batches.

**Dispatch phase:**

Create parallel Agent calls for the current batch:
```
Agent("Apply clean code refactoring to <file_path>. Formatter: <formatter_cmd>. Verification commands: <lint_cmd>, <build_cmd>. Baseline errors (pre-existing, ignore these): <baseline_errors>",
      subagent_type="clean-code:clean-code-single-file")
```

**Verification phase** (after all agents in the batch complete):

1. Run `git diff --name-only` and check which batch files were actually modified
2. Classify each file:
   - **modified**: File appears in `git diff --name-only` → actually changed
   - **no_changes_needed**: Agent reported "no changes needed" or "skipped all principles" AND file is not in diff → legitimately clean
   - **silently_failed**: Agent reported applying changes but file is NOT in diff → re-queue for retry (max 1 retry per file)
3. Collect each agent's report for the final summary (store which principles were applied per file)
4. Update progress.json: add verified files to `completed`, update `currentBatch`
5. Log any silently_failed files (they go back into the queue)

Do NOT run lint/build after each batch — this is deferred to final verification to avoid redundant checks (each agent already self-verifies).

**Progress report** (output to user after each batch):

```
## Batch N/M Complete
- Modified: [file1, file2, ...]
- No changes needed: [file3, ...]
- Silently failed: [file4, ...] (re-queued)
- Failed: [file5 (reason)]
- Progress: 20/42 files (48%)
```

### Task #(last): Final verification and summary report

When all files are processed (`completed.length + failed.length >= totalFiles`):

1. Run detected lint command from project root (if available)
2. Run detected build command from project root (if available)
3. If errors found, attempt to fix (2 attempts max)
4. Run `git diff --stat` to collect change statistics
5. Aggregate all agent reports to build the summary
6. Delete the `.clean-code/` directory entirely (`rm -rf .clean-code/`)
7. Output the final summary report

**Final summary report format:**

```
## Clean Code Summary
- Files processed: N | Modified: X | No changes needed: Y | Failed: Z
- Total lines changed: +A / -B

## Principle Hit Rate (how many files needed this fix)
- Naming: X/N (%)
- Stepdown Rule: X/N (%)
- Parameter Count: X/N (%)
- Abstraction Level: X/N (%)
- Error Handling: X/N (%)
- DRY Within File: X/N (%)
- Dead Code Removal: X/N (%)
- Reduce Nesting: X/N (%)
- Consistent Patterns: X/N (%)
- Magic Numbers: X/N (%)

## High-Change Files (review recommended)
1. path/to/file (+lines/-lines)
2. path/to/file (+lines/-lines)
3. path/to/file (+lines/-lines)

## Failed
- path/to/file: [reason]

## Verification
- lint: PASS / FAIL / SKIPPED
- build: PASS / FAIL / SKIPPED
```

The Principle Hit Rate shows how many files needed each fix. For example, "Naming: 12/16 (75%)" means 12 out of 16 files had naming issues that were improved. A high percentage indicates a systemic pattern across the codebase.

## Project Verification Tool Detection

Search the project root for these files and extract lint, build, and formatter commands:

| File | Lint | Build | Formatter |
|------|------|-------|-----------|
| `pubspec.yaml` | `dart analyze` | `dart compile` (if applicable) | `dart format <file>` |
| `package.json` | `scripts.lint` | `scripts.build` / `scripts.typecheck` | `prettier --write <file>` (if prettier config exists) or `eslint --fix <file>` |
| `pyproject.toml` | `ruff check .` / `flake8` / `pylint` | `mypy .` | `ruff format <file>` or `black <file>` |
| `Cargo.toml` | `cargo clippy` | `cargo check` | `rustfmt <file>` |
| `go.mod` | `go vet ./...` / `golangci-lint run` | `go build ./...` | `gofmt -w <file>` |
| `Makefile` | `make lint` / `make check` | `make build` | `make fmt` (if target exists) |
| `.swift-format` or `Package.swift` | `swiftlint` (if available) | `swift build` | `swift-format -i <file>` |
| `build.gradle` / `pom.xml` | project-specific | project-specific | `google-java-format <file>` (if available) |

**Fallback**: If the project is not in the table above, look for formatter config files in the project root (`.prettierrc`, `.editorconfig`, `.clang-format`, `.php-cs-fixer.php`, etc.). If found, use the corresponding formatter. If nothing is found, skip formatting.

If multiple config files exist (e.g., a project with both `package.json` and `Makefile`), prefer the most specific one for the detected source files.

**The formatter command is passed to each agent alongside lint/build commands.** Agents apply the formatter to their single file after refactoring, before lint verification.

## State File Update Rule

When updating `.clean-code/progress.json`, always follow this pattern:

1. **Read** the current file with Read tool
2. **Parse** the full JSON in memory
3. **Modify** only the fields that changed (e.g., `completed`, `currentBatch`, `status`)
4. **Write** back the complete JSON with ALL original fields preserved

Never construct the JSON from scratch — always start from the parsed original to prevent field omission.

## Error Recovery

- Progress file persists at `.clean-code/progress.json`
- Re-running the skill resumes from where it left off (already completed files are skipped)
- If the session is interrupted mid-batch, the current batch's incomplete files will be re-processed on resume

## Important

- Each agent works on ONE file independently — no cross-file changes
- Public interfaces (exported functions, types, public methods) are never modified
- The orchestrator (this skill) handles verification and error fixing, not the agents
- Verification tool detection happens ONCE and results are passed to all agents
