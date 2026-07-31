# ChatGPT Pro Tunnel-only Delivery Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace every Browser file-delivery and selector-fallback branch in `collaborating-with-chatgpt-pro` with one fail-closed Secure MCP Tunnel read path and an exact visible `Pro` response-performance gate.

**Architecture:** Keep the existing fixed-profile Tunnel lifecycle helper unchanged. Rewrite the authorization, artifact, route, transcript, and stop contracts as intentionally breaking Tunnel-only schemas, then make the top-level workflow consume only those schemas. Convert the static validator before the contract rewrite so the current legacy behavior produces an observed RED failure and the final behavior produces GREEN.

**Tech Stack:** Markdown Skill contracts, PowerShell 7 static validation and helper tests, Python `quick_validate.py`, YAML metadata, Codex in-app Browser for non-upload UI acceptance.

## Global Constraints

- `browser_file_uploads.mode` is the fixed literal `denied`; its allowlist is always empty.
- Local Artifact content is never attached, uploaded, pasted into a Browser prompt, archived, split, bundled, or added as a Project Source.
- `artifact.delivery` and Transcript delivery mode accept only `secure-mcp-tunnel`.
- The exact Browser app identity is `G Workspace Readonly`, backed by Profile `g-workspace-readonly`, read-only root `G:\workspace`, and required `list`／`read` capabilities.
- A stopped Tunnel is started only by the existing `scripts/ensure_secure_mcp_tunnel.ps1`; that helper and its tests are not behaviorally changed.
- The composer `応答性能` control must expose and select exact option `Pro`; collapsed button text `Pro` is the authoritative selected-state signal.
- The plan badge, inferred model, `非常に高い`, hidden selector, API, another browser, another Project, or another mode is not a substitute.
- Any lifecycle, app, root, target, Tool, coverage, or `Pro` verification failure ends as `blocked` with no file-delivery fallback.
- No compatibility adapter, legacy schema parser, dual-write, or migration is added.
- Official OpenAI facts remain separate from this personal Skill's project decisions.

---

### Task 1: Replace legacy static predicates and establish RED

**Files:**
- Modify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1`
- Test: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1`

**Interfaces:**
- Consumes: UTF-8 text from `SKILL.md`, the five direct reference contracts, lifecycle helper, and lifecycle tests.
- Produces: named boolean results `BrowserUploadsAlwaysDenied`, `TunnelOnlyArtifactDelivery`, `NoLegacyChatDeliverySchema`, `TunnelFailureBlocks`, `ExactBrowserAppIdentity`, `ResponsePerformanceProRequired`, `NoProductSelectedProFallback`, and `TranscriptTunnelOnlyEvidence`.

- [ ] **Step 1: Replace the authorization and delivery predicates**

Replace the union-shaped upload and multi-mode delivery checks with these contract checks:

```powershell
$browserUploadsAlwaysDenied = Has-All $promptContract @(
    '(?ms)browser_file_uploads:\s*\r?\n\s+mode:\s*denied\s*\r?\n\s+allowlist:\s*\[\]'
    '(?is)browser_file_uploads.{0,500}(?:fixed|always).{0,120}`denied`'
    '(?is)(?:attach|upload|paste|Project Source).{0,500}(?:never|prohibited|forbidden)'
) -and ($promptContract -notmatch '(?ms)browser_file_uploads:\s*\r?\n\s+mode:\s*scoped \| specific \| denied')
Add-Result 'BrowserUploadsAlwaysDenied' $browserUploadsAlwaysDenied

$tunnelOnlyArtifactDelivery = Has-All $artifactContract @(
    '(?m)^\s+delivery:\s+secure-mcp-tunnel\s*$'
    '(?is)Tunnel-only'
    '(?is)required_tools:\s*\[list,\s*read\]'
)
Add-Result 'TunnelOnlyArtifactDelivery' $tunnelOnlyArtifactDelivery
```

- [ ] **Step 2: Add explicit legacy-schema rejection**

Add structural rejection for the removed enum and branches:

```powershell
$legacyDeliverySchema = @(
    'delivery:\s*secure-mcp-tunnel\s*\|'
    'mode:\s*secure-mcp-tunnel\s*\|'
    '(?m)^\s*chat_delivery:\s*$'
    '(?m)^\s*model_selection_mode:\s*product-selected-pro\s*$'
    '(?m)^\s*quality_reasoning_control_'
)
$noLegacyChatDeliverySchema = $true
foreach ($pattern in $legacyDeliverySchema) {
    if ($allContractText -match $pattern) {
        $noLegacyChatDeliverySchema = $false
        break
    }
}
Add-Result 'NoLegacyChatDeliverySchema' $noLegacyChatDeliverySchema
Add-Result 'NoProductSelectedProFallback' (
    $allContractText -notmatch '(?m)^\s*model_selection_mode:\s*product-selected-pro\s*$'
)
```

Do not reject prose that names a prohibited Browser action; reject only legacy schema fields and union values.

- [ ] **Step 3: Add fail-closed, app-identity, Pro, and Transcript predicates**

```powershell
$tunnelFailureBlocks = Has-All $allContractText @(
    '(?is)Tunnel.{0,240}(?:unavailable|incomplete|failure).{0,240}`blocked`'
    '(?is)(?:never|do not).{0,180}(?:attach|upload|chat fallback)'
)
Add-Result 'TunnelFailureBlocks' $tunnelFailureBlocks

$exactBrowserAppIdentity = Has-All $allContractText @(
    '(?is)G Workspace Readonly'
    [regex]::Escape('G:\workspace')
    '(?is)read-only'
    '(?is)required.{0,80}`list`.{0,80}`read`'
)
Add-Result 'ExactBrowserAppIdentity' $exactBrowserAppIdentity

$responsePerformanceProRequired = Has-All $allContractText @(
    '(?is)`応答性能`'
    '(?is)(?:option|選択肢).{0,120}`Pro`'
    '(?is)collapsed.{0,120}(?:button|ボタン).{0,120}`Pro`'
    '(?is)(?:plan badge|Profile).{0,180}(?:not|does not|代替)'
)
Add-Result 'ResponsePerformanceProRequired' $responsePerformanceProRequired

$transcriptTunnelOnlyEvidence = Has-All $transcriptContract @(
    '(?m)^\s+mode:\s*secure-mcp-tunnel\s*$'
    '(?m)^\s+browser_file_uploads:\s*denied\s*$'
    '(?m)^\s+upload_attempted:\s*false\s*$'
    '(?m)^\s+response_performance_selected:\s*Pro\s*$'
    '(?m)^\s+response_performance_verified:\s*true\s*$'
)
Add-Result 'TranscriptTunnelOnlyEvidence' $transcriptTunnelOnlyEvidence
```

- [ ] **Step 4: Remove predicates that positively require deleted behavior**

Delete `ChatManifestBranch`, `ModeSpecificBranchExclusivity`,
`ChatCompletenessProtocol`, `ChatReceiptPrimaryPromptBoundary`,
`TunnelFailureFallbackBoundary`, `ProductSelectedProBoundary`,
`ProductSelectedProPositiveEvidence`, `ProductSelectedProEvidenceSeparation`,
and `NoUnconditionalProductSelectedPro`. Retain lifecycle, no-secret,
unexpected-write, prompt-quality, completion, Browser route, and no-pinned-model
checks that do not require legacy delivery.

- [ ] **Step 5: Run validator and verify RED**

Run:

```powershell
pwsh -NoProfile -File "$env:USERPROFILE\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1"
```

Expected: non-zero exit and at least the eight new result names print `FAIL`.
The failure must be a contract mismatch, not a PowerShell parse or missing-file
error.

- [ ] **Step 6: Commit the RED validator change in the personal Skill only if it is Git-backed**

The personal Skill is not Git-backed in this environment, so record the RED
command and output in the working transcript and do not create a repository
commit for this task.

### Task 2: Rewrite prompt authorization and artifact delivery as Tunnel-only

**Files:**
- Modify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\references\prompt-generation-contract.md`
- Modify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\references\artifact-delivery-contract.md`

**Interfaces:**
- Consumes: `already-running | started | blocked` lifecycle result from `ensure_secure_mcp_tunnel.ps1`.
- Produces: fixed-deny `task_contract`, single-mode `artifact` manifest, exact `project_route`, and Tool-read completeness protocol used by `SKILL.md` and Transcript.

- [ ] **Step 1: Replace the Task Contract schema**

Use this exact canonical authorization block:

```yaml
task_contract:
  browser_file_uploads:
    mode: denied
    allowlist: []
  tunnel_reads:
    mode: scoped | specific
    allowlist: []
  tunnel_writes:
    mode: denied
    allowlist: []
```

State that `browser_file_uploads` is non-overridable and that the Tunnel read
allowlist alone grants access to Local Artifact content.

- [ ] **Step 2: Replace Project route selection evidence**

Use:

```yaml
project_route:
  browser: codex-in-app-browser
  url: exact ChatGPT Project URL
  expected_title: visible Project title
  required_memory: project-only | disabled
  response_performance:
    control_display_name: 応答性能
    required_option: Pro
    option_visibility: visible
    option_selected: true
    collapsed_button_text: Pro
  browser_app:
    display_name: G Workspace Readonly
    description_root: G:\workspace
    access: read-only
  fallback: deny
```

Remove `product-selected-pro`, `quality_reasoning_control_*`, and every
selector-not-exposed success branch. Keep an optional observed model display
label only as non-authoritative Transcript metadata.

- [ ] **Step 3: Rewrite the artifact contract around one manifest**

Replace the file with focused sections: immutable no-upload authority,
canonical Tunnel manifest, Browser app preflight, Tool read protocol,
completeness reconciliation, unexpected-write incident, and fail-closed
handling. The canonical manifest is:

```yaml
artifact:
  id: stable task-local identifier
  repo_relative_path: path shown to ChatGPT
  canonical_path: verified path under allowed root
  role: why this artifact is material
  revision: commit, version, or not-applicable
  sensitivity: non-sensitive | approved-private | blocked
  delivery: secure-mcp-tunnel
  expected_bytes: integer
  sha256: lowercase hex
  required_tools: [list, read]
```

- [ ] **Step 4: Define completeness without a chat receipt phase**

Require exact equality of expected and read Artifact IDs, zero failed IDs,
successful required Tool calls, and target paths under the allowed root.
`expected_bytes` and `sha256` identify the locally adjudicated manifest;
ChatGPT must not claim it independently recomputed a digest unless an observed
Tool returned that digest.

Define one complete primary prompt containing Goal, path/manifest metadata,
criteria, output schema, and stop rule, but no Local Artifact content. ChatGPT
performs Tool reads before analysis in the same response. Do not define
attachment batches, receipt-only prompts, or `BEGIN_REVIEW`.

- [ ] **Step 5: Scan these contracts for legacy canonical fields**

Run:

```powershell
rg -n "delivery:\s*secure-mcp-tunnel\s*\||mode:\s*secure-mcp-tunnel\s*\||^\s*chat_delivery:|^\s*model_selection_mode:\s*product-selected-pro|^\s*quality_reasoning_control_" `
  "$env:USERPROFILE\.agents\skills\collaborating-with-chatgpt-pro\references\prompt-generation-contract.md" `
  "$env:USERPROFILE\.agents\skills\collaborating-with-chatgpt-pro\references\artifact-delivery-contract.md"
```

Expected: no matches.

### Task 3: Rewrite the Skill workflow and Transcript/stop contracts

**Files:**
- Modify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\SKILL.md`
- Modify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\references\transcript-contract.md`
- Modify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\references\adjudication-and-stop-rules.md`
- Modify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\references\response-completion-gate.md`

**Interfaces:**
- Consumes: Task Contract, Tunnel manifest, exact Project route, lifecycle result, Browser app and Tool evidence.
- Produces: executable seven-phase workflow, canonical Tunnel-only Transcript, and terminal blocker rules.

- [ ] **Step 1: Replace the top-level execution workflow**

Use these phases in order:

```text
1. inspect current task state and build fixed-deny Task Contract
2. run fixed-profile lifecycle helper before Browser navigation
3. verify the routed Project, sign-in, and required memory
4. select exact response performance Pro and verify collapsed Pro button
5. verify and activate exact G Workspace Readonly app
6. send one complete metadata-only primary prompt and require list/read Tools
7. reconcile coverage, adjudicate locally, or stop blocked
```

The workflow must say that lifecycle `blocked`, app mismatch, missing Tool,
unreachable path, incomplete coverage, or unavailable `Pro` cannot choose a
chat-delivery fallback.

- [ ] **Step 2: Replace route verification semantics**

Require exact visible `Pro` menu option selection for every send and follow-up.
The only success evidence is:

```yaml
response_performance_control: 応答性能
response_performance_selected: Pro
response_performance_verified: true
collapsed_button_text: Pro
```

State explicitly that the plan badge and an inferred or hidden model selector
do not satisfy the gate.

- [ ] **Step 3: Replace the Transcript schema**

Use:

```yaml
route:
  browser: codex-in-app-browser
  destination: ChatGPT Project title
  memory: project-only | disabled
  execution_date: YYYY-MM-DD
  response_performance_control: 応答性能
  response_performance_selected: Pro
  response_performance_verified: true
  model_display_name: observed visible label | not-exposed
  browser_app_display_name: G Workspace Readonly
evidence_delivery:
  mode: secure-mcp-tunnel
  browser_file_uploads: denied
  upload_attempted: false
  tunnel_profile: g-workspace-readonly
  allowed_root: G:\workspace
  required_tools: [list, read]
  tool_calls: []
  expected_artifact_ids: []
  read_artifact_ids: []
  failed_artifact_ids: []
  completeness: complete | incomplete
```

Remove every chat delivery branch and receipt/`BEGIN_REVIEW` evidence section.

- [ ] **Step 4: Tighten terminal blockers**

Add lifecycle, exact Browser app identity, allowed root, required Tool,
target-reachability, read-coverage, unexpected-write, and exact `Pro`
verification failures to the terminal blocker table. The action is always
`blocked`; it never mutates the delivery mode or Browser route.

- [ ] **Step 5: Simplify response completion**

Remove `BEGIN_REVIEW` from the prompt kinds. Keep completion checks for the
single primary prompt, continuation requests, and ordinary follow-ups. A Tool
read failure or incomplete manifest is a semantic blocker, not a reason to ask
for attachment delivery.

- [ ] **Step 6: Scan all runtime contracts for legacy canonical fields**

Run:

```powershell
rg -n "delivery:\s*secure-mcp-tunnel\s*\||mode:\s*secure-mcp-tunnel\s*\||^\s*chat_delivery:|^\s*model_selection_mode:\s*product-selected-pro|^\s*quality_reasoning_control_" `
  "$env:USERPROFILE\.agents\skills\collaborating-with-chatgpt-pro\SKILL.md" `
  "$env:USERPROFILE\.agents\skills\collaborating-with-chatgpt-pro\references"
```

Expected: no matches.

### Task 4: Reach GREEN and verify regressions

**Files:**
- Verify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro`
- Verify: `G:\workspace\development\GameEngine\miraikanai-engine\docs\superpowers\specs\2026-07-31-chatgpt-pro-tunnel-only-delivery-design.md`
- Verify: `G:\workspace\development\GameEngine\miraikanai-engine\docs\superpowers\plans\2026-07-31-chatgpt-pro-tunnel-only-delivery.md`

**Interfaces:**
- Consumes: all Task 1–3 outputs.
- Produces: current evidence for static contract validity, helper regression safety, Skill package validity, and repository cleanliness.

- [ ] **Step 1: Run the contract validator**

```powershell
pwsh -NoProfile -File "$env:USERPROFILE\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1"
```

Expected: exit 0 and every result, including the eight breaking-contract
results, prints `PASS`.

- [ ] **Step 2: Run all lifecycle helper tests**

```powershell
pwsh -NoProfile -File "$env:USERPROFILE\.agents\skills\collaborating-with-chatgpt-pro\scripts\test_ensure_secure_mcp_tunnel.ps1"
```

Expected: exit 0 with 28 passed and 0 failed.

- [ ] **Step 3: Run the official Skill quick validator**

```powershell
python "$env:USERPROFILE\.codex\skills\.system\skill-creator\scripts\quick_validate.py" `
  "$env:USERPROFILE\.agents\skills\collaborating-with-chatgpt-pro"
```

Expected: exit 0 and `Skill is valid!`.

- [ ] **Step 4: Parse metadata and verify direct references**

```powershell
$skillRoot = Join-Path $env:USERPROFILE '.agents\skills\collaborating-with-chatgpt-pro'
$frontmatter = Get-Content -Raw -LiteralPath (Join-Path $skillRoot 'SKILL.md')
if ($frontmatter -notmatch '(?s)^---\r?\nname:\s*collaborating-with-chatgpt-pro\r?\ndescription:.+?\r?\n---') {
    throw 'Invalid SKILL.md frontmatter'
}
$refs = [regex]::Matches($frontmatter, '\]\((references/[^)]+)\)')
foreach ($ref in $refs) {
    $path = Join-Path $skillRoot ($ref.Groups[1].Value -replace '/', '\')
    if (-not (Test-Path -LiteralPath $path -PathType Leaf)) {
        throw "Missing direct reference: $path"
    }
}
```

Expected: exit 0 with no output.

- [ ] **Step 5: Run non-transmitting Browser UI acceptance**

Open the required Project in the Codex in-app Browser without sending a
message. Verify the visible Project title, memory mode, `応答性能` menu option
`Pro`, collapsed button text `Pro`, Browser app display name
`G Workspace Readonly`, and description root `G:\workspace`.

Expected: all visible signals match. Do not open a file chooser, attach a file,
or add a Project Source.

- [ ] **Step 6: Gate the transmitting Tool-read acceptance**

A real Browser ChatGPT Tool-read test sends a message containing a specific
local path and manifest metadata. Immediately before that send, obtain
action-time user confirmation naming the destination Project and the exact
non-sensitive test path. If confirmation is not obtained, do not send and
report this acceptance check as not run rather than passing it.

Expected after confirmation: the response uses `G Workspace Readonly`
`list`／`read` without upload, reads only the allowlisted test Artifact, and
returns complete Tool-read evidence. Any mismatch is `blocked`.

- [ ] **Step 7: Review complete personal-Skill changes**

Because the personal Skill is outside Git, compare current files against a
task-local pre-edit copy or inspect every changed file in full. Confirm no
unrelated lifecycle helper, test, metadata, Profile, or repository policy
change occurred.

- [ ] **Step 8: Run repository checks**

```powershell
git diff --check
git status --short
git diff --stat
git log -2 --oneline
```

Expected: no unstaged repository change beyond the implementation-plan commit;
the latest two repository commits are the design and implementation plan.

- [ ] **Step 9: Commit the implementation plan**

```powershell
git add -- docs/superpowers/plans/2026-07-31-chatgpt-pro-tunnel-only-delivery.md
git commit -m "docs: plan ChatGPT Pro tunnel-only delivery"
```

Expected: one repository commit containing only this plan.
