# ChatGPT Pro Standalone MCP Route Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:executing-plans` to execute this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Personal Skill `collaborating-with-chatgpt-pro`の唯一のBrowser destinationをfresh standalone ChatGPT chatへ後方互換性なく切り替え、visible `Pro`とSecure MCP Tunnel-backed `G Workspace Readonly`でLocal Artifactをuploadせず読めることを静的検証と実ブラウザ試験で証明する。

**Architecture:** CodexはChatGPT Desktop app内蔵Browserを操作するcontrol planeであり、Browser版ChatGPTがread-only MCP clientである。Taskごとのmetadata-only Artifact manifestをpromptに渡し、本文はexact allowlistの`list_allowed_directories`／`list_directory`／`search`／`fetch`で取得する。Tool実行はvisible UIだけに依存せず、sanitized Tunnel telemetry deltaとArtifact completenessを結合して判定する。

**Tech Stack:** Markdown Skill contracts, PowerShell 7 static validator and lifecycle suite, Python 3.12／pytest Local Filesystem MCP tests, ChatGPT Desktop app built-in Browser, browser ChatGPT, Secure MCP Tunnel.

## Global Constraints

- 実装正本は[承認済み設計](../specs/2026-07-31-chatgpt-pro-standalone-mcp-route-design.md)とする。
- 現行Skill、five reference contracts、validator、repository `AGENTS.md`だけをroute変更対象とし、過去のplan／review／実測記録本文は改変しない。
- 旧`project_route`のalias、移行変換、dual schema、Project fallbackを実装しない。
- Browser routeは`https://chatgpt.com/`から始めるfresh standalone chatだけとし、pathが`/g/g-p-`で始まるrouteは成功扱いしない。
- Browser版ChatGPTのvisible `応答性能: Pro`、collapsed `Pro`、exact app `G Workspace Readonly`を送信前に毎回確認する。
- Local Artifact本文のattachment、upload、prompt paste、Project Source、別app、API、Chrome、Tunnel write、低い応答性能へのfallbackを禁止する。
- Tunnel Profile、allowed root `G:\workspace`、exact four Tool catalog、Tunnel executable、lifecycle helper、Local MCP implementationは変更しない。
- Transcript／repositoryへ`tunnel_id`、request ID、Profile本文、credential、cookie、account identifier、Local secretを保存しない。
- 個人Skill配下はこのrepository外なのでrepository commitに含めず、絶対pathで個別検証する。
- live acceptanceが失敗した場合は`blocked`を正確に記録し、別routeで成功を装わない。

---

### Task 1: Add RED predicates for the breaking route contract

**Files:**
- Modify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1`
- Test: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1`

**Consumes:** current Project-bound Skill contracts and the approved design.

**Produces:** named failing checks which distinguish the required standalone route, forbidden Project gates, and Tunnel evidence from the old contract.

- [x] **Step 1: Add the standalone-route shape check without editing production contracts**

After the current route verification block, add a check equivalent to:

```powershell
$standaloneChatRouteRequired = Has-All $prompt @(
    '(?ms)^chat_route:\s*$'
    '(?ms)^\s+browser:\s*codex-in-app-browser\s*$'
    '(?ms)^\s+destination_type:\s*standalone-chat\s*$'
    '(?ms)^\s+start_url:\s*https://chatgpt\.com/\s*$'
    '(?ms)^\s+new_chat_required:\s*true\s*$'
    '(?ms)^\s+project_membership:\s*forbidden\s*$'
    '(?ms)^\s+control_display_name:\s*応答性能\s*$'
    '(?ms)^\s+required_option:\s*Pro\s*$'
    '(?ms)^\s+collapsed_button_text:\s*Pro\s*$'
    '(?ms)^\s+display_name:\s*G Workspace Readonly\s*$'
    '(?ms)^\s+description_root:\s*G:\\workspace\s*$'
    '(?ms)^\s+access:\s*read-only\s*$'
    '(?ms)^\s+fallback:\s*deny\s*$'
)
Add-Result 'StandaloneChatRouteRequired' $standaloneChatRouteRequired
```

- [x] **Step 2: Add explicit absence checks for the deleted Project contract**

Build current-contract text only from `SKILL.md` and the five active references. Add separate results:

```powershell
$projectRouteForbidden = -not (Has-Any $allContractText @(
    '(?m)^project_route:\s*$'
    '(?m)^\s*destination:\s*ChatGPT Project title\s*$'
    'https://chatgpt\.com/g/g-p-'
))
Add-Result 'ProjectRouteForbidden' $projectRouteForbidden

$projectMemoryGateForbidden = -not (Has-Any $allContractText @(
    '(?m)^\s*memory:\s*project-only \| disabled\s*$'
    '(?i)memory mode\s+`プロジェクトのみ`'
    '(?i)verify the routed Project'
))
Add-Result 'ProjectMemoryGateForbidden' $projectMemoryGateForbidden
```

Keep the phrase `Project Source` allowed only inside deny rules; do not use a blanket search for the word `Project`.

- [x] **Step 3: Add telemetry, no-upload, exact-app, Pro, and no-fallback predicates**

Require the transcript schema and the owner rules with named results:

```powershell
$tunnelTelemetryEvidenceRequired = (Has-All $transcript @(
    '(?ms)^\s*tunnel_telemetry:\s*$'
    '(?m)^\s+baseline_tools_call_count:\s*integer\s*$'
    '(?m)^\s+final_tools_call_count:\s*integer\s*$'
    '(?m)^\s+tools_call_delta:\s*integer\s*$'
    '(?m)^\s+baseline_log_sequence:\s*integer\s*$'
    '(?m)^\s+forward_event_delta:\s*integer\s*$'
    '(?m)^\s+telemetry_sanitized:\s*true\s*$'
)) -and (Has-All $completion @(
    '(?is)Tunnel.{0,180}(?:delta|増分).{0,260}Artifact completeness'
    '(?is)(?:delta|増分).{0,120}(?:0|zero).{0,220}`blocked`'
))
Add-Result 'TunnelTelemetryEvidenceRequired' $tunnelTelemetryEvidenceRequired
```

Add or retain distinct `ResponsePerformanceProRequired`, `ExactBrowserAppIdentity`, `BrowserUploadsAlwaysDenied`, and `NoRouteFallback` checks. Each result must test its canonical owner rather than pass because the same sentence is duplicated across every reference.

- [x] **Step 4: Run the validator and capture RED**

Run:

```powershell
& 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1'
```

Expected: exit code `1`; at least `FAIL StandaloneChatRouteRequired`, `FAIL ProjectRouteForbidden`, `FAIL ProjectMemoryGateForbidden`, and `FAIL TunnelTelemetryEvidenceRequired`. Confirm no production contract changed in Task 1.

---

### Task 2: Replace the active Skill and reference contracts

**Files:**
- Modify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\SKILL.md`
- Modify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\references\prompt-generation-contract.md`
- Modify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\references\artifact-delivery-contract.md`
- Modify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\references\response-completion-gate.md`
- Modify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\references\transcript-contract.md`
- Modify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\references\adjudication-and-stop-rules.md`
- Modify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1`

**Consumes:** Task 1 RED output.

**Produces:** one active standalone Browser route, one telemetry-aware completion rule, and zero current Project route aliases.

- [x] **Step 1: Replace the Skill route phase**

In `SKILL.md`, replace the Project identity／memory phase with a standalone route phase that requires:

```markdown
1. Open `https://chatgpt.com/` in the ChatGPT Desktop app's built-in Browser.
2. Start a fresh standalone chat and verify that its path does not begin with `/g/g-p-`.
3. Before every primary send and follow-up, visibly select `応答性能: Pro` and verify the collapsed button text is exactly `Pro`.
4. Select the exact app `G Workspace Readonly`; verify its visible description is rooted at `G:\workspace` and read-only.
5. If standalone identity, `Pro`, app identity, or Tunnel evidence cannot be verified, stop as `blocked` without fallback.
```

Keep Codex as Browser control plane and browser ChatGPT as MCP client. Remove exact Project URL/title, Project chat creation, and Project-memory verification from active workflow text.

- [x] **Step 2: Make the prompt contract the canonical route owner**

Replace `project_route:` with the exact `chat_route:` YAML from the approved design. Add a metadata-only context manifest whose item shape is:

```yaml
artifact_manifest:
  - artifact_id: root-relative identifier
    role: task-specific role
    revision: observed revision or digest basis
    expected_bytes: observed integer
    sha256: observed lowercase hex
    required_tools:
      - list_allowed_directories
      - fetch
    evaluation_criteria:
      - task-specific completeness predicate
```

State that Local Artifact body, quoted excerpts, encoded payloads, and substitute summaries are forbidden in the prompt. Require only task-relevant exact allowlist entries.

- [x] **Step 3: Align artifact delivery and completion ownership**

In `artifact-delivery-contract.md`, replace Project-route wording with standalone route and retain the four exact read-only Tools. Make `search` metadata-first and `fetch` content-returning. Keep attachment, upload, paste, Project Source, and Tunnel write as unconditional denies.

In `response-completion-gate.md`, define success as all of:

```text
standalone route verified
+ visible Pro verified
+ exact app verified
+ tools_call_delta > 0
+ forward_event_delta > 0
+ every required Artifact passes bytes/digest/completeness checks
+ no unauthorized Tool or delivery path observed
```

Define a visible Tool card as supplemental evidence when exposed. A card must not replace telemetry or completeness; a missing card must not independently fail an otherwise fully corroborated turn.

- [x] **Step 4: Replace transcript route fields and add sanitized telemetry**

In `transcript-contract.md`, replace the current route block with:

```yaml
route:
  browser: codex-in-app-browser
  destination_type: standalone-chat
  start_url: https://chatgpt.com/
  new_chat_required: true
  project_membership: forbidden
  execution_date: YYYY-MM-DD
  model_display_name: observed visible label | not-exposed
  response_performance_control: 応答性能
  response_performance_selected: Pro
  response_performance_verified: true
  collapsed_button_text: Pro
  browser_app_display_name: G Workspace Readonly
  browser_app_description_root: G:\workspace
  browser_app_access: read-only
```

Under delivery evidence, add:

```yaml
tunnel_telemetry:
  baseline_tools_call_count: integer
  final_tools_call_count: integer
  tools_call_delta: integer
  baseline_log_sequence: integer
  forward_event_delta: integer
  telemetry_sanitized: true
```

Explicitly forbid retaining Tunnel IDs, request IDs, Profile contents, credentials, or secrets. Replace `tunnel_tool_calls: count` with the canonical delta block rather than keeping both schemas.

- [x] **Step 5: Align adjudication and terminal blockers**

In `adjudication-and-stop-rules.md`, make every primary／follow-up completion check revalidate fresh standalone route, visible `Pro`, exact app, Tunnel delta, and Artifact completeness. Remove standalone chat from forbidden fallbacks; add Project chat and any path beginning with `/g/g-p-` as forbidden destinations. Preserve no-API, no-upload, no-lower-mode, and no-write rules.

- [x] **Step 6: Make all new validator predicates GREEN**

Remove or replace `BrowserProjectMemoryProVisualVerification`. Keep the existing `ResponsePerformanceSelectionPolicy` and `TranscriptObservedProModeDate` only if their patterns match the new canonical schema without accepting Project state. Replace `ResourceObservationShape`'s legacy `tunnel_tool_calls: count` expectation with the six-field `tunnel_telemetry` schema.

Run:

```powershell
& 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1'
```

Expected: exit code `0`; all eight approved-design predicates are `PASS`; `TOTAL_FAILURES=0`.

---

### Task 3: Align the Miraikanai repository destination policy

**Files:**
- Modify: `AGENTS.md`
- Test: `AGENTS.md`

**Consumes:** Task 2 canonical `chat_route` contract.

**Produces:** repository-specific destination selection that adds no second schema and no Project-only exception.

- [x] **Step 1: Replace the ChatGPT Pro collaboration route section**

Keep the invocation trigger and repository authority boundaries. Replace the exact Project URL/title/memory rules with:

```markdown
- Use the Codex in-app Browser and start each distinct outcome in a fresh standalone chat at `https://chatgpt.com/`.
- Before sending, visibly verify that the chat is not Project-bound, the response-performance control is selected as `Pro`, the collapsed control reads `Pro`, and the exact `G Workspace Readonly` app is selected.
- Treat any path beginning with `/g/g-p-`, Project memory, Project Source, another app, API, Chrome, a lower response-performance option, or file attachment/upload/paste as forbidden fallbacks.
```

Link behavior ownership back to the Global Skill. Keep repository Artifact access under `G:\workspace`, Task Contract authority, local diff validation, and the retention policy unchanged.

- [x] **Step 2: Check required and forbidden active text**

Run:

```powershell
$agents = Get-Content -Raw -LiteralPath 'AGENTS.md'
if ($agents -notmatch 'fresh standalone chat') { throw 'standalone route missing' }
if ($agents -notmatch 'response-performance control is selected as `Pro`') { throw 'Pro gate missing' }
if ($agents -match 'g-p-6a69aacb1ae88191a27dd74eeb166569/project') { throw 'legacy Project URL remains' }
if ($agents -match 'memory mode\s+`プロジェクトのみ`') { throw 'legacy Project memory gate remains' }
```

Expected: exit code `0`.

---

### Task 4: Run the complete local regression suite

**Files:**
- Test: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1`
- Test: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\test_ensure_secure_mcp_tunnel.ps1`
- Test: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\tests`
- Test: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\readonly_local_files`

**Consumes:** Tasks 2–3 implementation.

**Produces:** current evidence that route documentation changes did not regress Tunnel lifecycle or Local Filesystem MCP behavior.

- [x] **Step 1: Run the static Skill validator**

```powershell
& 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1'
```

Expected: `TOTAL_FAILURES=0`.

- [x] **Step 2: Run the Tunnel lifecycle suite**

```powershell
& 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\test_ensure_secure_mcp_tunnel.ps1'
```

Expected: exit code `0`; all lifecycle cases pass. Do not include Profile contents or generated Tunnel identifiers in durable evidence.

- [x] **Step 3: Run the Local MCP test suite and self-test**

```powershell
$serverRoot = 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server'
$venvPython = Join-Path $serverRoot '.venv\Scripts\python.exe'
& $venvPython -m pytest (Join-Path $serverRoot 'tests') -q
& $venvPython -m readonly_local_files.server --self-test
```

Expected: pytest exit code `0`; self-test reports the exact Tool catalog `list_allowed_directories`, `list_directory`, `search`, `fetch` and no write Tool.

- [x] **Step 4: Run the system Skill validator**

```powershell
$venvPython = 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\.venv\Scripts\python.exe'
& $venvPython 'C:\Users\y2ikg\.codex\skills\.system\skill-creator\scripts\quick_validate.py' 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro'
```

Expected: exit code `0` and a valid Skill package.

---

### Task 5: Run a standalone Browser ChatGPT Pro acceptance

**Files:**
- Read: `C:\Users\y2ikg\.codex\plugins\cache\openai-bundled\browser\26.721.81911\skills\control-in-app-browser\SKILL.md`
- Run: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\ensure_secure_mcp_tunnel.ps1`
- Read: one existing known acceptance Artifact under `G:\workspace`
- Test: ChatGPT Desktop app built-in Browser, fresh standalone browser ChatGPT chat

**Consumes:** Task 4 passing local suite.

**Produces:** one fail-closed End-to-End disposition based on visible route state, sanitized Tunnel delta, and exact Artifact completeness.

- [x] **Step 1: Load Browser instructions and ensure the Tunnel lifecycle**

Read `browser:control-in-app-browser` completely before Browser calls. Run the lifecycle helper exactly as documented by the Skill:

```powershell
& 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\ensure_secure_mcp_tunnel.ps1'
```

Expected: healthy existing Tunnel is reused; absent/unhealthy Tunnel is started and becomes ready; a hard lifecycle failure stops the acceptance.

- [x] **Step 2: Derive the known Artifact manifest locally**

Resolve one existing `known.md` acceptance Artifact under `G:\workspace`. Record only its root-relative ID, byte count, lowercase SHA-256, and non-secret marker in task-local evidence:

```powershell
$known = Get-ChildItem -LiteralPath 'G:\workspace' -Recurse -File -Filter 'known.md' |
    Where-Object { $_.FullName -notmatch '\\.git\\' } |
    Select-Object -First 1
if (-not $known) { throw 'known.md acceptance Artifact not found' }
$manifest = [ordered]@{
    artifact_id = $known.FullName.Substring('G:\workspace\'.Length).Replace('\', '/')
    expected_bytes = $known.Length
    sha256 = (Get-FileHash -Algorithm SHA256 -LiteralPath $known.FullName).Hash.ToLowerInvariant()
}
$manifest | ConvertTo-Json
```

Do not paste the file body into the browser prompt.

- [x] **Step 3: Capture sanitized Tunnel baseline**

From the currently running tunnel client's loopback admin telemetry, record only:

```yaml
baseline_tools_call_count: integer
baseline_log_sequence: integer
captured_at: ISO-8601 with offset
```

Obtain the concrete read-only metrics/log endpoints from the installed tunnel client's current help/schema before querying; do not guess ports or endpoint paths. Discard Tunnel IDs, request IDs, Profile body, credentials, and unrelated log messages immediately.

- [x] **Step 4: Start and verify the exact Browser route**

Using only the in-app Browser, navigate to `https://chatgpt.com/`, start a fresh standalone chat, and verify:

```yaml
destination_type: standalone-chat
project_membership: forbidden
response_performance_control: 応答性能
response_performance_selected: Pro
collapsed_button_text: Pro
browser_app_display_name: G Workspace Readonly
browser_app_description_root: G:\workspace
browser_app_access: read-only
```

Stop as `blocked` if URL/state is Project-bound, `Pro` is not visibly selected, or exact app identity cannot be verified.

- [x] **Step 5: Send one metadata-only acceptance prompt**

Require browser ChatGPT to call `list_allowed_directories`, then `fetch` for the exact `artifact_id`, and return only:

- observed allowed root;
- observed Artifact ID, byte count, SHA-256, and marker;
- read completeness or exact failure;
- unique completion marker `STANDALONE_PRO_MCP_ACCEPTANCE_20260731`.

Do not attach/upload a file, paste Artifact content, add Project Source, call API, switch app, or lower response performance.

- [ ] **Step 6: Reconcile final telemetry and Artifact output**

Capture final `tools/call` count and forward events strictly after `baseline_log_sequence`, then calculate:

```text
tools_call_delta = final_tools_call_count - baseline_tools_call_count
forward_event_delta = count(sanitized matching forward events after baseline_log_sequence)
```

Pass only when both deltas are positive and the response's Artifact ID, bytes, digest, marker, and completeness match the local manifest. A visible Tool card may be recorded as supplemental evidence. Any zero delta, mismatch, partial read, or inferred response is `blocked`.

> **Current blocker (2026-08-01):** Initial telemetry and response observations did
> not preserve the active Transcript contract's per-call `call_id`, exact-one
> requirement mapping, or duplicate／unauthorized-extra reconciliation. They are
> historical and non-dispositive. A strict retry stopped before navigation because
> `iab` was unavailable in the subagent Browser inventory; lifecycle was `ready`,
> sends／uploads／fallbacks were 0. Current live acceptance is `blocked`／
> `incomplete`. Resume in a parent context with `iab` and rerun a fresh acceptance
> with call-level reconciliation for every observed call.

- [x] **Step 7: Finalize Browser tabs as the last Browser operation**

After the final visible evidence read, call the Browser tab finalization method. Make no Browser call afterward in this execution turn.

---

### Task 6: Record evidence, review the full change, and commit repository files

**Files:**
- Modify: `docs/superpowers/specs/2026-07-31-chatgpt-pro-standalone-mcp-route-design.md`
- Modify: `docs/superpowers/plans/2026-07-31-chatgpt-pro-standalone-mcp-route.md`
- Modify: `docs/reviews/README.md`
- Verify: `AGENTS.md`

**Consumes:** Tasks 1–5 exact outputs.

**Produces:** a compact non-normative audit record, checked plan completion state, and one repository commit without transient consultation data.

- [x] **Step 1: Record only current implementation and acceptance evidence**

Append a compact implementation record to the approved design: changed active contracts, named validator result, lifecycle/pytest/self-test/Skill-validator result, browser route, visible mode, sanitized Tunnel deltas, Artifact completeness, upload count `0`, and final `pass` or `blocked` disposition.

Update `docs/reviews/README.md` before deleting any transient prompt/response/screenshot data. Include audit ID/date, standalone route and `Pro`, scope, input count and digest, valid-gap count, affected active contracts, closure result, exact terminal marker, response digest when available, and retention disposition. Never promote this review summary to Architecture or implementation evidence.

- [x] **Step 2: Mark plan checkboxes from actual evidence**

Check only completed steps. If live acceptance is blocked, leave its unmet pass predicate unchecked and record the exact blocker; local implementation tasks may still be checked independently.

- [x] **Step 3: Scan current contracts for stale route aliases and unfinished markers**

Run:

```powershell
$skillRoot = 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro'
$active = @(
    Join-Path $skillRoot 'SKILL.md'
    Join-Path $skillRoot 'references\prompt-generation-contract.md'
    Join-Path $skillRoot 'references\artifact-delivery-contract.md'
    Join-Path $skillRoot 'references\response-completion-gate.md'
    Join-Path $skillRoot 'references\transcript-contract.md'
    Join-Path $skillRoot 'references\adjudication-and-stop-rules.md'
)
$stale = Select-String -LiteralPath $active -Pattern '^project_route:|ChatGPT Project title|g-p-6a69aacb1ae88191a27dd74eeb166569/project|memory: project-only \| disabled'
if ($stale) { $stale; throw 'stale Project route remains' }
$unfinishedPattern = (@('TO' + 'DO', 'T' + 'BD', 'FIX' + 'ME') -join '|')
$unfinished = Select-String -LiteralPath ($active + @(
    'AGENTS.md',
    'docs/superpowers/specs/2026-07-31-chatgpt-pro-standalone-mcp-route-design.md',
    'docs/superpowers/plans/2026-07-31-chatgpt-pro-standalone-mcp-route.md'
)) -Pattern $unfinishedPattern
if ($unfinished) { $unfinished; throw 'unfinished marker remains' }
```

Expected: exit code `0`.

- [x] **Step 4: Run final repository checks and inspect the complete diff**

```powershell
git diff --check
git status --short
git diff --stat
git diff -- AGENTS.md docs/superpowers/specs/2026-07-31-chatgpt-pro-standalone-mcp-route-design.md docs/superpowers/plans/2026-07-31-chatgpt-pro-standalone-mcp-route.md docs/reviews/README.md
```

Confirm no unrelated file changed, no absent implementation is claimed, and evidence labels distinguish official facts, user requirement, project decision, and measured result.

- [x] **Step 5: Stage and commit only repository-owned changes**

```powershell
git add -- AGENTS.md docs/superpowers/specs/2026-07-31-chatgpt-pro-standalone-mcp-route-design.md docs/superpowers/plans/2026-07-31-chatgpt-pro-standalone-mcp-route.md docs/reviews/README.md
git diff --cached --check
git diff --cached --stat
git commit -m 'docs: use standalone ChatGPT Pro MCP route'
```

Expected: commit succeeds. Personal Skill files remain outside this repository commit but have passing current validators from Task 4.
