# Codex Configuration Optimization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Apply the officially recommended GPT-5.6 model, reasoning, output, sandbox, and profile defaults while preserving all unrelated Codex integrations and providing safe rollback.

**Architecture:** Keep general defaults in the user-level configuration, apply game-engine-specific quality settings in the trusted repository, and isolate high-speed, deep-analysis, routine, and scan workloads into explicit profiles. Project configuration has higher precedence than profiles, so overlapping values inside this repository use the desktop model picker or explicit CLI overrides. Leave model context, compaction, and tool-output limits unset so the live Codex model catalog remains authoritative.

**Tech Stack:** Codex CLI 0.144.5, TOML, JSON Schema, PowerShell, Python 3 `tomllib`, Git.

## Global Constraints

- Preserve every unrelated MCP, plugin, app, marketplace, desktop, notification, shell-environment, authentication, and project-trust setting. Correct only the target repository's stale trust path when necessary.
- Do not add production dependencies.
- Use GPT-5.6 Sol / Medium globally and Sol / High in this repository.
- Reserve Sol / Max for the explicit Deep profile outside higher-precedence project overrides, or for an explicit per-session override.
- Use Terra / Medium for Fast and Scan, and Luna / Medium for Routine.
- Keep `model_verbosity = "low"` globally and in routine work.
- Do not set `model_context_window`, `model_auto_compact_token_limit`, `model_auto_compact_token_limit_scope`, `tool_output_token_limit`, or `model_supports_reasoning_summaries`.
- Create a hash-verified timestamped backup bundle for every existing user config and profile before changing them, and record profiles that did not yet exist.
- Do not use subagents for execution unless the user explicitly asks for them.

---

### Task 1: Capture the baseline and create rollback material

**Files:**
- Read: `$CODEX_HOME/config.toml`
- Read: `$CODEX_HOME/fast.config.toml`
- Read: `$CODEX_HOME/deep.config.toml`
- Read: `$CODEX_HOME/scan.config.toml`
- Read if present: `$CODEX_HOME/routine.config.toml`
- Create: `$CODEX_HOME/backups/codex-config-optimization-<yyyyMMdd-HHmmss>/`

**Interfaces:**
- Consumes: Current Codex configuration files and their SHA-256 hashes.
- Produces: One immutable rollback bundle containing the global config, every existing profile, and an existence/hash manifest used by final verification.

- [x] **Step 1: Confirm the repository and user configuration baseline**

Run:

```powershell
git status --short
$codexHome = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { Join-Path $HOME '.codex' }
$configFiles = @(
  'config.toml',
  'fast.config.toml',
  'deep.config.toml',
  'scan.config.toml'
) | ForEach-Object { Join-Path $codexHome $_ }
Get-FileHash -Algorithm SHA256 -LiteralPath $configFiles
```

Expected: Git reports only the worktree ignore rule and design/plan documentation changes; all four user files return SHA-256 hashes.

- [x] **Step 2: Verify the baseline parses before copying it**

Run:

```powershell
@'
import os, pathlib, tomllib
root = pathlib.Path(os.environ.get("CODEX_HOME", pathlib.Path.home() / ".codex"))
paths = [
    root / "config.toml",
    root / "fast.config.toml",
    root / "deep.config.toml",
    root / "scan.config.toml",
]
for path in paths:
    with path.open("rb") as handle:
        tomllib.load(handle)
    print(f"OK {path}")
'@ | python -
```

Expected: Four `OK` lines and exit code 0.

- [x] **Step 3: Create and verify the rollback copy**

Run:

```powershell
$stamp = Get-Date -Format 'yyyyMMdd-HHmmss'
$codexHome = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { Join-Path $HOME '.codex' }
$backupRoot = Join-Path $codexHome "backups\codex-config-optimization-$stamp"
New-Item -ItemType Directory -Path $backupRoot | Out-Null
$manifest = [ordered]@{
  createdAt = (Get-Date).ToString('o')
  routineConfigExisted = Test-Path -LiteralPath (Join-Path $codexHome 'routine.config.toml')
  files = [ordered]@{}
}
foreach ($name in 'config.toml', 'fast.config.toml', 'deep.config.toml', 'scan.config.toml', 'routine.config.toml') {
  $source = Join-Path $codexHome $name
  if (-not (Test-Path -LiteralPath $source)) { continue }
  $backup = Join-Path $backupRoot $name
  Copy-Item -LiteralPath $source -Destination $backup
  $sourceHash = (Get-FileHash -Algorithm SHA256 -LiteralPath $source).Hash
  $backupHash = (Get-FileHash -Algorithm SHA256 -LiteralPath $backup).Hash
  if ($sourceHash -ne $backupHash) { throw "Backup hash mismatch: $name" }
  $manifest.files[$name] = [ordered]@{ sha256 = $sourceHash }
}
$manifest | ConvertTo-Json -Depth 4 |
  Set-Content -LiteralPath (Join-Path $backupRoot 'manifest.json') -Encoding utf8NoBOM
$backupRoot
```

Expected: One timestamped directory containing four byte-identical config files and `manifest.json`; the manifest records that Routine did not exist before this change.

### Task 2: Apply user-level defaults and workload profiles

**Files:**
- Modify: `$CODEX_HOME/config.toml`
- Modify: `$CODEX_HOME/fast.config.toml`
- Modify: `$CODEX_HOME/deep.config.toml`
- Modify: `$CODEX_HOME/scan.config.toml`
- Create: `$CODEX_HOME/routine.config.toml`

**Interfaces:**
- Consumes: The verified backup from Task 1 and the exact values in the approved design.
- Produces: A safe global Power default plus four explicit workload profiles.

- [x] **Step 1: Apply the global model, output, safety, and concurrency values**

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

Remove the global `review_model` override so `/review` uses the current session model. Remove only the ignored `js_repl = false` feature entry and remove `[features]` only if that table becomes empty.

- [x] **Step 2: Apply the Fast profile**

Use this complete content:

```toml
model = "gpt-5.6-terra"
model_reasoning_effort = "medium"
plan_mode_reasoning_effort = "medium"
model_verbosity = "low"
model_reasoning_summary = "none"
personality = "pragmatic"
service_tier = "fast"
```

- [x] **Step 3: Apply the Deep profile**

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

- [x] **Step 4: Apply the Scan profile**

Use this complete content:

```toml
model = "gpt-5.6-terra"
model_reasoning_effort = "medium"
plan_mode_reasoning_effort = "medium"
model_verbosity = "low"
model_reasoning_summary = "none"
personality = "none"
sandbox_mode = "read-only"
```

- [x] **Step 5: Create the Routine profile**

Use this complete content:

```toml
model = "gpt-5.6-luna"
model_reasoning_effort = "medium"
plan_mode_reasoning_effort = "medium"
model_verbosity = "low"
model_reasoning_summary = "none"
personality = "pragmatic"
```

- [x] **Step 6: Parse all five user configuration files**

Run:

```powershell
@'
import os, pathlib, tomllib
root = pathlib.Path(os.environ.get("CODEX_HOME", pathlib.Path.home() / ".codex"))
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
- Modify: `$CODEX_HOME/config.toml` only if the normalized repository root is missing or stale in `[projects]`
- Create: `.codex/config.toml`

**Interfaces:**
- Consumes: The user-level Sol / Medium default from Task 2.
- Produces: A trusted-project override that raises implementation and planning effort without increasing answer verbosity.

- [x] **Step 1: Confirm exact repository trust**

Run `git rev-parse --show-toplevel`, normalize the resulting absolute path for Windows, and ensure the user config contains the exact repository root:

```toml
[projects.'<normalized-repository-root>']
trust_level = "trusted"
```

Preserve every unrelated trust entry. Project `.codex/config.toml` is ignored until this exact repository is trusted.

- [x] **Step 2: Create the project configuration**

Use this complete content:

```toml
#:schema https://developers.openai.com/codex/config-schema.json

model = "gpt-5.6-sol"
model_reasoning_effort = "high"
plan_mode_reasoning_effort = "xhigh"
model_verbosity = "low"
model_reasoning_summary = "concise"
```

- [x] **Step 3: Parse the repository configuration**

Run:

```powershell
@'
import pathlib, tomllib
path = pathlib.Path(".codex/config.toml")
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

- [x] **Step 4: Verify project-layer provenance**

Open a new CLI task in the repository and run `/debug-config`, or call app-server `config/read` with the exact repository `cwd` and `includeLayers = true`.

Expected: No project-disabled warning. The origins for `model`, `model_reasoning_effort`, `plan_mode_reasoning_effort`, `model_verbosity`, and `model_reasoning_summary` are `project` and point to the repository `.codex` folder.

### Task 4: Validate effective behavior and review scope

**Files:**
- Verify: `$CODEX_HOME/config.toml`
- Verify: `$CODEX_HOME/fast.config.toml`
- Verify: `$CODEX_HOME/deep.config.toml`
- Verify: `$CODEX_HOME/routine.config.toml`
- Verify: `$CODEX_HOME/scan.config.toml`
- Verify: `.codex/config.toml`
- Verify: `docs/superpowers/specs/2026-07-18-codex-config-optimization-design.md`

**Interfaces:**
- Consumes: All configuration layers produced by Tasks 2 and 3.
- Produces: Current evidence that the CLI loads each layer and that only intended repository files changed.

- [x] **Step 1: Validate the normal configuration**

Run:

```powershell
codex doctor --json
codex features list
codex debug models
```

Expected: Configuration load reports `ok`; model output advertises Sol, Terra, Luna and the configured reasoning levels; no removed `js_repl` override remains.

- [x] **Step 2: Validate each explicit profile**

Run:

```powershell
$profileValidationRoot = Join-Path ([System.IO.Path]::GetTempPath()) 'codex-profile-validation'
New-Item -ItemType Directory -Force -Path $profileValidationRoot | Out-Null
Push-Location $profileValidationRoot
try {
  codex --profile fast debug prompt-input "profile validation"
  codex --profile deep debug prompt-input "profile validation"
  codex --profile routine debug prompt-input "profile validation"
  codex --profile scan debug prompt-input "profile validation"
} finally {
  Pop-Location
}
```

Expected: Every command exits 0 and emits JSON that parses successfully. Fast and Routine expose `workspace-write`; Deep and Scan expose `read-only`. Run this check outside the repository because project configuration has higher precedence than profiles. This command proves runtime loading and permission composition; validate each configured model and reasoning value directly from the parsed TOML.

- [x] **Step 3: Validate against the current official JSON Schema**

Run a local JSON Schema validator already available in the bundled workspace runtime against the six TOML documents after converting them to objects. Do not install a dependency. If the runtime has no validator, use `codex doctor --json` plus Python `tomllib` as the validation boundary and report that limitation.

Expected: Six valid documents, or an explicit report that runtime and parser validation passed while an independent schema library was unavailable.

- [x] **Step 4: Review the final repository diff**

Run:

```powershell
git diff --check
git status --short
git diff -- .codex/config.toml .gitignore docs/superpowers/specs/2026-07-18-codex-config-optimization-design.md docs/superpowers/plans/2026-07-18-codex-config-optimization.md
```

Expected: No whitespace errors; only the project config, worktree ignore rule, and the two documentation files are changed or untracked.

- [x] **Step 5: Commit the repository-scoped configuration and documentation**

Run:

```powershell
git add -- .codex/config.toml .gitignore docs/superpowers/specs/2026-07-18-codex-config-optimization-design.md docs/superpowers/plans/2026-07-18-codex-config-optimization.md
git commit -m "chore: optimize Codex model configuration"
```

Expected: The feature branch contains only the four listed repository files. User-level configuration files remain outside Git.

### Rollback procedure

Restore the four pre-existing files from the verified bundle and remove Routine only when the manifest records that it did not exist before this change:

```powershell
$codexHome = if ($env:CODEX_HOME) { $env:CODEX_HOME } else { Join-Path $HOME '.codex' }
$backupRoot = Join-Path $codexHome 'backups\codex-config-optimization-<timestamp>'
$manifest = Get-Content -Raw -LiteralPath (Join-Path $backupRoot 'manifest.json') |
  ConvertFrom-Json
foreach ($file in $manifest.files.PSObject.Properties) {
  $name = $file.Name
  Copy-Item -Force -LiteralPath (Join-Path $backupRoot $name) -Destination (Join-Path $codexHome $name)
  $restoredHash = (Get-FileHash -Algorithm SHA256 -LiteralPath (Join-Path $codexHome $name)).Hash
  if ($restoredHash -ne $file.Value.sha256) { throw "Restore hash mismatch: $name" }
}
if (-not $manifest.routineConfigExisted -and (Test-Path -LiteralPath (Join-Path $codexHome 'routine.config.toml'))) {
  Remove-Item -LiteralPath (Join-Path $codexHome 'routine.config.toml')
}
codex doctor --json
```

Expected: Restored files match the SHA-256 values in `manifest.json`, Routine returns to its recorded prior existence state, and `config.load` reports `ok`.
