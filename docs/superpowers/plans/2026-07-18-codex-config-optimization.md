# Codex Configuration Optimization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Apply the officially recommended GPT-5.6 model, reasoning, output, sandbox, and profile defaults while preserving all unrelated Codex integrations and providing safe rollback.

**Architecture:** Keep general defaults in the user-level configuration, apply game-engine-specific quality settings in the trusted repository, and isolate high-speed, deep-analysis, routine, and scan workloads into explicit profiles. Leave model context, compaction, and tool-output limits unset so the live Codex model catalog remains authoritative.

**Tech Stack:** Codex CLI 0.144.5, TOML, JSON Schema, PowerShell, Python 3 `tomllib`, Git.

## Global Constraints

- Preserve every existing MCP, plugin, app, marketplace, desktop, notification, shell-environment, authentication, and project-trust setting.
- Do not add production dependencies.
- Use GPT-5.6 Sol / Medium globally and Sol / High in this repository.
- Reserve Sol / Max for the explicit Deep profile.
- Use Terra / Medium for Fast and Scan, and Luna / Medium for Routine.
- Keep `model_verbosity = "low"` globally and in routine work.
- Do not set `model_context_window`, `model_auto_compact_token_limit`, `model_auto_compact_token_limit_scope`, `tool_output_token_limit`, or `model_supports_reasoning_summaries`.
- Create a timestamped backup before changing `C:\Users\y2ikg\.codex\config.toml`.
- Do not use subagents for execution unless the user explicitly asks for them.

---

### Task 1: Capture the baseline and create rollback material

**Files:**
- Read: `C:\Users\y2ikg\.codex\config.toml`
- Read: `C:\Users\y2ikg\.codex\fast.config.toml`
- Read: `C:\Users\y2ikg\.codex\deep.config.toml`
- Read: `C:\Users\y2ikg\.codex\scan.config.toml`
- Create: `C:\Users\y2ikg\.codex\config.toml.backup-<yyyyMMdd-HHmmss>`

**Interfaces:**
- Consumes: Current Codex configuration files and their SHA-256 hashes.
- Produces: One immutable rollback copy of the global configuration and a baseline hash map used by final verification.

- [ ] **Step 1: Confirm the repository and user configuration baseline**

Run:

```powershell
git status --short
Get-FileHash -Algorithm SHA256 `
  C:\Users\y2ikg\.codex\config.toml, `
  C:\Users\y2ikg\.codex\fast.config.toml, `
  C:\Users\y2ikg\.codex\deep.config.toml, `
  C:\Users\y2ikg\.codex\scan.config.toml
```

Expected: Git reports only the design and plan documentation changes; all four user files return SHA-256 hashes.

- [ ] **Step 2: Verify the baseline parses before copying it**

Run:

```powershell
@'
import pathlib, tomllib
paths = [
    pathlib.Path(r"C:\Users\y2ikg\.codex\config.toml"),
    pathlib.Path(r"C:\Users\y2ikg\.codex\fast.config.toml"),
    pathlib.Path(r"C:\Users\y2ikg\.codex\deep.config.toml"),
    pathlib.Path(r"C:\Users\y2ikg\.codex\scan.config.toml"),
]
for path in paths:
    with path.open("rb") as handle:
        tomllib.load(handle)
    print(f"OK {path}")
'@ | python -
```

Expected: Four `OK` lines and exit code 0.

- [ ] **Step 3: Create and verify the rollback copy**

Run:

```powershell
$stamp = Get-Date -Format 'yyyyMMdd-HHmmss'
$source = 'C:\Users\y2ikg\.codex\config.toml'
$backup = "$source.backup-$stamp"
Copy-Item -LiteralPath $source -Destination $backup
$sourceHash = (Get-FileHash -Algorithm SHA256 -LiteralPath $source).Hash
$backupHash = (Get-FileHash -Algorithm SHA256 -LiteralPath $backup).Hash
if ($sourceHash -ne $backupHash) { throw 'Backup hash mismatch' }
$backup
```

Expected: One timestamped backup path and exit code 0.

### Task 2: Apply user-level defaults and workload profiles

**Files:**
- Modify: `C:\Users\y2ikg\.codex\config.toml`
- Modify: `C:\Users\y2ikg\.codex\fast.config.toml`
- Modify: `C:\Users\y2ikg\.codex\deep.config.toml`
- Modify: `C:\Users\y2ikg\.codex\scan.config.toml`
- Create: `C:\Users\y2ikg\.codex\routine.config.toml`

**Interfaces:**
- Consumes: The verified backup from Task 1 and the exact values in the approved design.
- Produces: A safe global Power default plus four explicit workload profiles.

- [ ] **Step 1: Apply the global model, output, safety, and concurrency values**

Set these root values while preserving every unrelated key and table:

```toml
model = "gpt-5.6-sol"
model_reasoning_effort = "medium"
plan_mode_reasoning_effort = "high"
model_verbosity = "low"
model_reasoning_summary = "none"
personality = "pragmatic"
approval_policy = "on-request"
approvals_reviewer = "user"
sandbox_mode = "workspace-write"
web_search = "cached"
```

Set:

```toml
[agents]
max_threads = 6
max_depth = 1
```

Remove only the ignored `js_repl = false` entry and remove `[features]` only if that table becomes empty.

- [ ] **Step 2: Apply the Fast profile**

Use this complete content:

```toml
model = "gpt-5.6-terra"
model_reasoning_effort = "medium"
model_verbosity = "low"
model_reasoning_summary = "none"
personality = "pragmatic"
service_tier = "fast"
```

- [ ] **Step 3: Apply the Deep profile**

Use this complete content:

```toml
model = "gpt-5.6-sol"
model_reasoning_effort = "max"
plan_mode_reasoning_effort = "xhigh"
model_verbosity = "medium"
model_reasoning_summary = "concise"
personality = "none"
sandbox_mode = "read-only"
```

- [ ] **Step 4: Apply the Scan profile**

Use this complete content:

```toml
model = "gpt-5.6-terra"
model_reasoning_effort = "medium"
model_verbosity = "low"
model_reasoning_summary = "none"
personality = "none"
sandbox_mode = "read-only"
```

- [ ] **Step 5: Create the Routine profile**

Use this complete content:

```toml
model = "gpt-5.6-luna"
model_reasoning_effort = "medium"
model_verbosity = "low"
model_reasoning_summary = "none"
personality = "pragmatic"
```

- [ ] **Step 6: Parse all five user configuration files**

Run:

```powershell
@'
import pathlib, tomllib
root = pathlib.Path(r"C:\Users\y2ikg\.codex")
for name in (
    "config.toml",
    "fast.config.toml",
    "deep.config.toml",
    "routine.config.toml",
    "scan.config.toml",
):
    path = root / name
    with path.open("rb") as handle:
        tomllib.load(handle)
    print(f"OK {path}")
'@ | python -
```

Expected: Five `OK` lines and exit code 0.

### Task 3: Add the repository-specific quality layer

**Files:**
- Create: `G:\workspace\development\GameEngine\mirakanai-engine\.codex\config.toml`

**Interfaces:**
- Consumes: The user-level Sol / Medium default from Task 2.
- Produces: A trusted-project override that raises implementation and planning effort without increasing answer verbosity.

- [ ] **Step 1: Create the project configuration**

Use this complete content:

```toml
#:schema https://developers.openai.com/codex/config-schema.json

model = "gpt-5.6-sol"
model_reasoning_effort = "high"
plan_mode_reasoning_effort = "xhigh"
model_verbosity = "low"
model_reasoning_summary = "concise"
```

- [ ] **Step 2: Parse the repository configuration**

Run:

```powershell
@'
import pathlib, tomllib
path = pathlib.Path(r"G:\workspace\development\GameEngine\mirakanai-engine\.codex\config.toml")
with path.open("rb") as handle:
    data = tomllib.load(handle)
assert data["model"] == "gpt-5.6-sol"
assert data["model_reasoning_effort"] == "high"
assert data["plan_mode_reasoning_effort"] == "xhigh"
assert data["model_verbosity"] == "low"
assert data["model_reasoning_summary"] == "concise"
print("OK project config")
'@ | python -
```

Expected: `OK project config` and exit code 0.

### Task 4: Validate effective behavior and review scope

**Files:**
- Verify: `C:\Users\y2ikg\.codex\config.toml`
- Verify: `C:\Users\y2ikg\.codex\fast.config.toml`
- Verify: `C:\Users\y2ikg\.codex\deep.config.toml`
- Verify: `C:\Users\y2ikg\.codex\routine.config.toml`
- Verify: `C:\Users\y2ikg\.codex\scan.config.toml`
- Verify: `G:\workspace\development\GameEngine\mirakanai-engine\.codex\config.toml`
- Verify: `G:\workspace\development\GameEngine\mirakanai-engine\docs\superpowers\specs\2026-07-18-codex-config-optimization-design.md`

**Interfaces:**
- Consumes: All configuration layers produced by Tasks 2 and 3.
- Produces: Current evidence that the CLI loads each layer and that only intended repository files changed.

- [ ] **Step 1: Validate the normal configuration**

Run:

```powershell
codex doctor --json
codex features list
codex debug models
```

Expected: Configuration load reports `ok`; model output advertises Sol, Terra, Luna and the configured reasoning levels; no removed `js_repl` override remains.

- [ ] **Step 2: Validate each explicit profile**

Run:

```powershell
codex --profile fast doctor --json
codex --profile deep doctor --json
codex --profile routine doctor --json
codex --profile scan doctor --json
```

Expected: Every command exits 0 with configuration load `ok`.

- [ ] **Step 3: Validate against the current official JSON Schema**

Run a local JSON Schema validator already available in the bundled workspace runtime against the six TOML documents after converting them to objects. Do not install a dependency. If the runtime has no validator, use `codex doctor --json` plus Python `tomllib` as the validation boundary and report that limitation.

Expected: Six valid documents, or an explicit report that runtime and parser validation passed while an independent schema library was unavailable.

- [ ] **Step 4: Review the final repository diff**

Run:

```powershell
git diff --check
git status --short
git diff -- .codex/config.toml docs/superpowers/specs/2026-07-18-codex-config-optimization-design.md docs/superpowers/plans/2026-07-18-codex-config-optimization.md
```

Expected: No whitespace errors; only the project config and the two documentation files are changed or untracked.

- [ ] **Step 5: Commit the repository-scoped configuration and documentation**

Run:

```powershell
git add -- .codex/config.toml docs/superpowers/specs/2026-07-18-codex-config-optimization-design.md docs/superpowers/plans/2026-07-18-codex-config-optimization.md
git commit -m "chore: optimize Codex model configuration"
```

Expected: One commit containing only the three listed repository files. User-level configuration files remain outside Git.
