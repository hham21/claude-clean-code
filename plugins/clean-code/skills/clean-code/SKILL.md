---
name: clean-code
description: Use when applying clean code refactoring to one or more files. Accepts a single file, a folder (processes source files recursively), or no argument (processes entire project). Orchestrates parallel clean-code-single-file agents in dynamically-sized batches (up to 10 per batch), auto-detects project lint/build tools, tracks progress via progress.json, verifies actual file modifications via git diff, and retries silently failed agents. Works with any programming language. Trigger on "clean code", "refactor", "code cleanup", "improve readability", "코드 정리", "클린코드", "리팩토링".
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

Also verify the project is a git repository by running `git rev-parse --is-inside-work-tree`. If not, inform the user that this skill requires a git repo (for file discovery and change verification) and stop.

## Input Modes

Determine the mode from user input:

| Input | Mode | Behavior |
|-------|------|----------|
| Single source file path | **Direct** | Dispatch one agent, verify, done |
| Folder path | **Batch** | Collect source files under folder, batch process |
| No path / "all" | **Batch** | Collect all source files in project |

### Exclude Option

User can specify files or folders to exclude (e.g., "exclude tests/", "config/ 제외해줘"). Parse these from the user's message and add them to the default exclude list in file discovery.

### Principle Selection (Optional)

User can specify which principles to apply or skip:
- "only naming, function structure" → apply only those principles
- "skip parameter count, magic numbers" → apply all except those

Parse principle names from the user's message. Pass the selection to each agent as:
`Apply ONLY these principles: [list]` or `Skip these principles: [list]`

If no selection is specified, apply all 11 principles (default behavior).

## Direct Mode

Create these Tasks and execute in order:

### Task #1: Detect verification tools

Search the project root for build/lint configuration (see "Project Verification Tool Detection" below). Store the detected commands.

### Task #2: Dispatch agent and wait for completion

```
Agent("Apply clean code refactoring to <file_path>. Formatter: <detected_formatter>",
      subagent_type="clean-code:clean-code-single-file")
```

If no formatter was detected, omit it from the prompt.

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
- Non-source files: `*.md`, `*.txt`, `*.csv`, `*.svg`, `*.png`, `*.jpg`, `*.gif`, `*.ico`, `*.woff`, `*.woff2`, `*.ttf`, `*.eot`, `*.mp3`, `*.mp4`, `*.zip`, `*.tar`, `*.gz`, `*.wasm`, `*.pyc`, `*.class`, `*.so`, `*.exe`, `*.dll`
- User-specified exclusions

Sort files by path for deterministic ordering. Store the file list.

### Task #3: Create progress.json

Create `.clean-code/progress.json`:

```json
{
  "target": "<path or 'project'>",
  "totalFiles": 42,
  "batchSize": 9,
  "completed": [],
  "failed": [],
  "currentBatch": 0,
  "verificationCommands": {
    "lint": "<detected or null>",
    "build": "<detected or null>",
    "test": "<detected or null>"
  },
  "status": "in_progress"
}
```

If resuming from a previous session (progress.json already exists), read it, re-collect files via `git ls-files`, subtract `completed` and `failed` arrays, and process the remaining files with fresh batching. If a batch was interrupted mid-way, its incomplete files will be re-processed — this is safe because all changes are idempotent (applying the same principle twice produces the same result).

### Task #3.5: Collect baseline errors (before any changes)

If verification commands were detected, run them now **before any refactoring** and store the output. When passing baseline errors to agents, filter to only include errors relevant to that agent's file (grep for the filename in the output). This prevents token waste from sending the full project error log to every agent.

### Task #4..N: Batch dispatch and verify (repeated)

> **NEVER dispatch the next batch until ALL agents in the current batch have returned AND the Checkpoint is written.** Overlapping batches will cause git diff confusion and file corruption.

**Dynamic batch sizing:** Instead of fixed batches of 10, distribute files evenly across batches. Calculate: `batch_size = ceil(totalFiles / ceil(totalFiles / 10))`. For example, 16 files → 8+8 instead of 10+6. This ensures balanced workload across batches.

**Dispatch phase:**

Create parallel Agent calls for the current batch:
```
Agent("Apply clean code refactoring to <file_path>. Formatter: <formatter_cmd>. Baseline errors (pre-existing, ignore these): <baseline_errors>",
      subagent_type="clean-code:clean-code-single-file")
```

> **Important**: Do NOT pass lint/build/test commands to agents. Agents must only run per-file formatters. Project-wide verification (lint, build, test) is the orchestrator's responsibility and runs only during final verification.

**Verification phase** (after all agents in the batch complete):

1. Run `git diff --name-only` and check which batch files were actually modified
2. Classify each file:
   - **modified**: File appears in `git diff --name-only` → actually changed
   - **no_changes_needed**: Agent reported "no changes needed" or "skipped all principles" AND file is not in diff → legitimately clean
   - **silently_failed**: Agent reported applying changes but file is NOT in diff → re-queue for retry (max 1 retry per file). If the retry also fails, add to `failed` with reason "silent failure after retry".
3. Collect each agent's report for the final summary (store which principles were applied per file)

### Checkpoint (MANDATORY after each batch)

Update `.clean-code/progress.json` immediately after the verification phase, BEFORE dispatching the next batch. This is the session resume point — skipping it means interrupted sessions restart from scratch.

1. Read the current `progress.json` with the Read tool
2. Add all verified files (modified + no_changes_needed) to `completed`
3. Add exhausted silently_failed files to `failed`
4. Increment `currentBatch`
5. Write back the complete JSON (follow the State File Update Rule below)
6. Read back the file to confirm the write succeeded

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

When all files are processed (`completed.length + failed.length >= totalFiles`, where `failed` includes silently_failed files that exhausted their retry):

1. Run detected lint command from project root (if available)
2. Run detected build command from project root (if available)
3. Run detected test command from project root (if available)
4. If errors found, attempt to fix (2 attempts max)
5. Run `git diff --stat` to collect change statistics
6. Aggregate all agent reports to build the summary
7. Delete the `.clean-code/` directory entirely (`rm -rf .clean-code/`)
8. Output the final summary report

**Final summary report format:**

```
## Clean Code Summary
- Files processed: N | Modified: X | No changes needed: Y | Failed: Z
- Total lines changed: +A / -B

## Principle Hit Rate (how many files needed this fix)
- Naming: X/N (%)
- Function Structure: X/N (%)
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
- test: PASS / FAIL / SKIPPED
```

The Principle Hit Rate shows how many files needed each fix. For example, "Naming: 12/16 (75%)" means 12 out of 16 files had naming issues that were improved. A high percentage indicates a systemic pattern across the codebase.

## Project Verification Tool Detection

Search the project root for these files and extract lint, build, test, and formatter commands:

| File | Lint | Build | Test | Formatter |
|------|------|-------|------|-----------|
| `pubspec.yaml` | `dart analyze` | `dart compile` (if applicable) | `dart test` | `dart format <file>` |
| `package.json` | `scripts.lint` | `scripts.build` / `scripts.typecheck` | `scripts.test` (add `--bail` if jest) | `prettier --write <file>` (if prettier config exists) or `eslint --fix <file>` |
| `pyproject.toml` | `ruff check .` / `flake8` / `pylint` | `mypy .` | `pytest --tb=short -q` | `ruff format <file>` or `black <file>` |
| `Cargo.toml` | `cargo clippy` | `cargo check` | `cargo test --no-fail-fast` | `rustfmt <file>` |
| `go.mod` | `go vet ./...` / `golangci-lint run` | `go build ./...` | `go test ./...` | `gofmt -w <file>` |
| `Makefile` | `make lint` / `make check` | `make build` | `make test` (if target exists) | `make fmt` (if target exists) |
| `.swift-format` or `Package.swift` | `swiftlint` (if available) | `swift build` | `swift test` | `swift-format -i <file>` |
| `build.gradle` / `pom.xml` | project-specific | project-specific | project-specific | `google-java-format <file>` (if available) |

**Fallback**: If the project is not in the table above, look for formatter config files in the project root (`.prettierrc`, `.editorconfig`, `.clang-format`, `.php-cs-fixer.php`, etc.). If found, use the corresponding formatter. If nothing is found, skip formatting.

If multiple config files exist (e.g., a project with both `package.json` and `Makefile`), prefer the most specific one for the detected source files.

**Only the formatter command is passed to agents.** When passing the formatter command, replace `<file>` with the actual file path (e.g., `prettier --write src/utils.ts`, not `prettier --write <file>`). Lint/build/test commands are used by the orchestrator only during final verification.

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
- The orchestrator (this skill) handles project-wide verification and error fixing, not the agents
- Verification tool detection happens ONCE — the formatter is passed to agents, lint/build are used by the orchestrator only
