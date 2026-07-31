# ChatGPT Project Secure MCP Tunnel Acceptance Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Browser版ChatGPTに新規Project `Secure MCP Tunnel 検証`を作成し、既存Secure MCP Tunnel-backed `G Workspace Readonly`のread-only Tool discoveryと実Artifact readを公式手順に沿って検証する。

**Architecture:** Local MCP Server、固定Tunnel Profile、既存developer-mode appは変更せず、Local contract、MCP Inspector、ChatGPT connection metadata、新規Project chatの順に検証する。BrowserへはArtifact本文を送らず、固定manifestだけを渡し、`fetch`結果のbytes／SHA-256／markerとsanitized Tunnel telemetry deltaをLocal evidenceへ照合する。

**Tech Stack:** PowerShell 7、CPython 3.14、MCP Python SDK 2.0.0、pytest 9.1.1、`@modelcontextprotocol/inspector`、OpenAI `tunnel-client`、Browser版ChatGPT、Secure MCP Tunnel、Codex Browser control

## Global Constraints

- 実装正本は[承認済み設計](../specs/2026-08-01-chatgpt-project-secure-mcp-tunnel-acceptance-design.md)とする。
- 新規ChatGPT Project名はexact `Secure MCP Tunnel 検証`、memoryは`プロジェクトのみ`とする。
- Browser appは既存exact `G Workspace Readonly`だけを使い、重複Appを作成しない。
- Tunnel Profileは`g-workspace-readonly`、allowed rootはexact `G:\workspace`とする。
- Tool catalogはexact `list_allowed_directories`、`list_directory`、`search`、`fetch`とし、write Toolを許可しない。
- Browser attachment、upload、Local Artifact本文のprompt paste、Project Source、alternate app、standalone-chat fallback、公開HTTPS fallbackを禁止する。
- 既存ChatGPT Project、既存chat、既存App、Tunnel Profile、Local MCP実装、現行Personal Skill contract、repository `AGENTS.md`を変更しない。
- `tunnel_id`、request ID、Profile本文、API key、cookie、account ID、Local secretをdurable evidenceへ保存しない。
- 初回`Pro` chatでTool catalogがsessionへ公開されなかった実測後は、同じProject内の新規chatでresponse performanceだけをApp互換の非`Pro` modelへ変更する。別条件は変えず、一度だけ再試行する。
- 失敗時は別経路へ切り替えず、last verified stateとminimum resume actionを`blocked`として記録する。

---

### Task 1: Establish the immutable Artifact manifest

**Files:**
- Read: `docs/superpowers/specs/2026-08-01-chatgpt-project-secure-mcp-tunnel-acceptance-design.md`

**Interfaces:**
- Consumes: approved existing-App Refresh design
- Produces: approved design plus `artifact_id`, `expected_bytes`, `source_sha256`, and `marker` used by Tasks 2–4

- [ ] **Step 1: Verify the approved design and fixed acceptance Artifact**

Run from the repository root:

```powershell
$workspaceRoot = 'G:\workspace'
$artifact = Join-Path $workspaceRoot 'development\GameEngine\miraikanai-engine\docs\superpowers\specs\2026-08-01-chatgpt-project-secure-mcp-tunnel-acceptance-design.md'
if (-not (Test-Path -LiteralPath $artifact -PathType Leaf)) {
    throw 'acceptance Artifact missing'
}
$design = Get-Content -Raw -LiteralPath $artifact
if ($design -notmatch '(?m)^- 状態: `approved-for-implementation`$') {
    throw 'design is not approved for implementation'
}
```

Expected: exit code `0`.

- [ ] **Step 2: Derive the manifest without reading it into the Browser prompt**

Run:

```powershell
$workspaceRoot = 'G:\workspace'
$artifact = Join-Path $workspaceRoot 'development\GameEngine\miraikanai-engine\docs\superpowers\specs\2026-08-01-chatgpt-project-secure-mcp-tunnel-acceptance-design.md'
$marker = '# ChatGPT Project Secure MCP Tunnel Acceptance Design'
$manifest = [ordered]@{
    artifact_id = $artifact.Substring($workspaceRoot.Length + 1).Replace('\', '/')
    expected_bytes = (Get-Item -LiteralPath $artifact).Length
    source_sha256 = (Get-FileHash -Algorithm SHA256 -LiteralPath $artifact).Hash.ToLowerInvariant()
    marker = $marker
}
$manifest | ConvertTo-Json
```

Expected: one root-relative `artifact_id`, positive `expected_bytes`, 64-character lowercase `source_sha256`, and the exact non-secret marker. Keep this object in task-local memory only.

- [ ] **Step 3: Verify a clean repository baseline before execution**

Run:

```powershell
git diff --check
git status --short
git diff --stat
```

Expected: all commands succeed and `git status --short` is empty. If unrelated user changes exist, preserve them and record their paths before continuing.

---

### Task 2: Prove the Local MCP and Tunnel contract before Browser work

**Files:**
- Run: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1`
- Run: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\test_ensure_secure_mcp_tunnel.ps1`
- Run: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\ensure_secure_mcp_tunnel.ps1`
- Test: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\tests`
- Run: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\src\readonly_local_files\server.py`

**Interfaces:**
- Consumes: fixed profile `g-workspace-readonly` and exact four-Tool Local Server
- Produces: current Local test evidence and lifecycle result with `health`, `ready`, `profile_verified`, `lock_verified`, `catalog_verified`, `allowed_root_verified` all true

- [ ] **Step 1: Run the static Skill contract validator**

Run:

```powershell
& 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1'
```

Expected: exit code `0` and `TOTAL_FAILURES=0`. This validates the existing personal Skill package only; it does not authorize its standalone Browser route for this acceptance.

- [ ] **Step 2: Run the Tunnel lifecycle regression suite**

Run:

```powershell
& 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\test_ensure_secure_mcp_tunnel.ps1'
```

Expected: exit code `0`; every lifecycle case passes.

- [ ] **Step 3: Run all Local MCP tests and self-test**

Run:

```powershell
$serverRoot = 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server'
$python = Join-Path $serverRoot '.venv\Scripts\python.exe'
& $python -m pytest (Join-Path $serverRoot 'tests') -q
& $python -m readonly_local_files.server --self-test
```

Expected: pytest exit code `0`; self-test returns one JSON line with `allowed_root` equal to `G:\workspace`, `annotations_exact: true`, the exact four-Tool catalog, and empty `forbidden_tools`.

- [ ] **Step 4: Ensure exactly one healthy fixed Tunnel runtime**

Run:

```powershell
$ensure = & 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\ensure_secure_mcp_tunnel.ps1' | ConvertFrom-Json
if ($ensure.status -notin @('already-running', 'started')) { throw $ensure.reason }
foreach ($field in 'health','ready','profile_verified','lock_verified','catalog_verified','allowed_root_verified') {
    if ($ensure.$field -ne $true) { throw "Tunnel check failed: $field" }
}
$ensure | Select-Object status,reason,health,ready,profile_verified,lock_verified,catalog_verified,allowed_root_verified
```

Expected: exit code `0`; no Profile content or identifier is printed.

---

### Task 3: Inspect the Local MCP transport with the official Inspector CLI

**Files:**
- Read: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\src\readonly_local_files\server.py`
- Run: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\.venv\Scripts\python.exe`
- Read: the fixed acceptance Artifact from Task 1 through MCP only

**Interfaces:**
- Consumes: Task 2 passing Local runtime and Task 1 `artifact_id`
- Produces: Inspector evidence for initialize／`tools/list`, valid read, invalid input, path escape denial, and unknown write Tool denial

- [ ] **Step 1: Confirm Node.js and npm runner availability**

Run:

```powershell
node --version
npx --version
```

Expected: both exit code `0`. If unavailable, load the bundled workspace Node runtime and rerun once; if still unavailable, stop `blocked` before Browser work.

- [ ] **Step 2: List the exact Tool catalog through Inspector CLI**

Run from the MCP server directory:

```powershell
$python = 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\.venv\Scripts\python.exe'
& npx.cmd -y '@modelcontextprotocol/inspector@latest' --cli $python -m readonly_local_files.server -- --method tools/list
```

Expected: exit code `0`; exactly `list_allowed_directories`, `list_directory`, `search`, `fetch` with explicit input/output schemas and read-only annotations.

- [ ] **Step 3: Call the allowed-root and Artifact read Tools**

Run:

```powershell
$python = 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\.venv\Scripts\python.exe'
$artifactId = 'development/GameEngine/miraikanai-engine/docs/superpowers/specs/2026-08-01-chatgpt-project-secure-mcp-tunnel-acceptance-design.md'
& npx.cmd -y '@modelcontextprotocol/inspector@latest' --cli $python -m readonly_local_files.server -- --method tools/call --tool-name list_allowed_directories
& npx.cmd -y '@modelcontextprotocol/inspector@latest' --cli $python -m readonly_local_files.server -- --method tools/call --tool-name list_directory --tool-arg 'path=development/GameEngine/miraikanai-engine/docs/superpowers/specs'
& npx.cmd -y '@modelcontextprotocol/inspector@latest' --cli $python -m readonly_local_files.server -- --method tools/call --tool-name search --tool-arg 'query=2026-08-01-chatgpt-project-secure-mcp-tunnel-acceptance-design.md'
& npx.cmd -y '@modelcontextprotocol/inspector@latest' --cli $python -m readonly_local_files.server -- --method tools/call --tool-name fetch --tool-arg "id=$artifactId"
```

Expected: allowed root `G:\workspace`; directory and search results contain the acceptance design; fetch returns matching `id`, `metadata.source_bytes`, `metadata.source_sha256`, and marker text.

- [ ] **Step 4: Verify invalid and forbidden operations fail closed**

Run each command separately and retain only the typed error category:

```powershell
$python = 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\.venv\Scripts\python.exe'
& npx.cmd -y '@modelcontextprotocol/inspector@latest' --cli $python -m readonly_local_files.server -- --method tools/call --tool-name fetch --tool-arg 'id=../outside.txt'
& npx.cmd -y '@modelcontextprotocol/inspector@latest' --cli $python -m readonly_local_files.server -- --method tools/call --tool-name write_file --tool-arg 'path=forbidden.txt'
```

Expected: path escape returns a typed boundary error; `write_file` returns method/tool not found. Neither command creates or changes a file.

---

### Task 4: Refresh the existing ChatGPT app and create the isolated Project

**Files:**
- Read: `C:\Users\y2ikg\.codex\plugins\cache\openai-bundled\browser\26.721.81911\skills\control-in-app-browser\SKILL.md`
- External state: existing ChatGPT developer-mode app `G Workspace Readonly`
- External state: new ChatGPT Project `Secure MCP Tunnel 検証`

**Interfaces:**
- Consumes: Task 2 healthy Tunnel and Task 3 exact Tool discovery
- Produces: refreshed existing app metadata and one new isolated ChatGPT Project with `プロジェクトのみ` memory

- [ ] **Step 1: Initialize Browser control for the target URL**

Before Browser control, query available tools for a purpose-built ChatGPT connector. Because no connector provides ChatGPT Developer mode／Project UI management, use the Browser skill. Initialize the Browser runtime, select the browser for `https://chatgpt.com/projects`, read its complete documentation once, and reuse that binding for all following steps.

Expected: an authenticated ChatGPT tab is available. If authentication is required, ask the user to sign in in the selected browser and stop until they confirm.

- [ ] **Step 2: Verify Developer mode and open the existing connection**

Navigate through ChatGPT Settings → Security and login and verify Developer mode is enabled. Open [ChatGPT Plugins](https://chatgpt.com/plugins), locate exact `G Workspace Readonly`, and verify its visible description states root `G:\workspace` and read-only access.

Expected: exact app exists once. Do not create or delete an app.

- [ ] **Step 3: Refresh and inspect the existing connection metadata**

Select `Refresh` on the existing connection and inspect the discovered Tool metadata.

Expected: exact Tool names `list_allowed_directories`, `list_directory`, `search`, `fetch`; read-only annotations; no write Tool. If metadata is missing after one Refresh, capture the visible error and stop `blocked` without recreating the app.

- [ ] **Step 4: Create the new isolated Project**

Open `https://chatgpt.com/projects`, create one Project with exact title `Secure MCP Tunnel 検証`, and choose memory `プロジェクトのみ` when prompted.

Expected: the Project opens with the exact title and Project-only memory. If a Project with that exact title already exists from an interrupted run, reuse it only when it has no unrelated chats or sources; otherwise stop and ask before creating a suffixed duplicate.

- [ ] **Step 5: Start one new Project chat and add the exact app**

Start a new chat inside the Project, open its Tools menu, and add exact `G Workspace Readonly`.

Expected: the composer visibly shows the selected app. Task 4ではresponse performanceを変更せず、添付、Project Sources、別Appも追加しない。Task 5の初回`Pro`診断がTool非公開で終わった場合だけ、承認済み設計の一変数再試行へ進む。

---

### Task 5: Run the Project-chat Tool acceptance and reconcile evidence

**Files:**
- Read through MCP: `development/GameEngine/miraikanai-engine/docs/superpowers/specs/2026-08-01-chatgpt-project-secure-mcp-tunnel-acceptance-design.md`
- External state: new Project chat from Task 4
- Read-only endpoint: `http://127.0.0.1:8080/metrics`
- Read-only endpoint: `http://127.0.0.1:8080/ui`

**Interfaces:**
- Consumes: Task 1 manifest, Task 4 selected exact app, and Task 2 healthy Tunnel
- Produces: one `pass` or `blocked` disposition with visible Tool evidence, matching Artifact metadata, and sanitized Tunnel delta

- [ ] **Step 1: Capture the sanitized Tunnel baseline**

Read the loopback-only `/metrics` and `/ui` surfaces before sending. Record only the current aggregate `tools/call` count, last visible log sequence, and capture time. Do not retain Tunnel IDs, request IDs, Profile content, credentials, or unrelated log text.

Expected shape:

```yaml
baseline_tools_call_count: integer
baseline_log_sequence: integer
captured_at: ISO-8601 with offset
```

If these sanitized values cannot be derived from the current surfaces, record telemetry as unavailable and use the visible ChatGPT Tool cards plus exact fetch metadata as the required Tool evidence; do not infer a call from the response text alone.

- [ ] **Step 2: Select one App-compatible non-Pro model for the corrected retry**

The initial `Pro` attempt is retained as measured diagnostic evidence: the exact app pill was visible, but the session exposed no required Tool and produced no Tool card. Start one new chat in the same Project, change only response performance to a non-`Pro` model that the current UI presents as App-compatible, then reselect exact `G Workspace Readonly`.

Expected: exact Project, app, prompt, Artifact, memory, and safety constraints remain unchanged. Record the visible non-`Pro` model label. If no App-compatible non-`Pro` option is available, stop `blocked` without changing the Tunnel or app.

- [ ] **Step 3: Send one metadata-only corrected acceptance prompt**

Send the following prompt after verifying exact `G Workspace Readonly` is still selected:

```text
G Workspace Readonlyだけを使って、次を順に実行してください。
1. list_allowed_directoriesで許可rootを確認する。
2. list_directoryで development/GameEngine/miraikanai-engine/docs/superpowers/specs を確認する。
3. fetchで development/GameEngine/miraikanai-engine/docs/superpowers/specs/2026-08-01-chatgpt-project-secure-mcp-tunnel-acceptance-design.md を読む。

添付、Project Source、Web検索、別Appは使わないでください。最終回答には、観測したroot、Artifact ID、metadata.source_bytes、metadata.source_sha256、先頭見出し、各Toolの成功／失敗、および完了marker CHATGPT_PROJECT_MCP_ACCEPTANCE_20260801 だけを記載してください。Toolが利用できなければ推測せず、利用できないTool名とエラーを記載してください。
```

Expected: this corrected chat visibly invokes `list_allowed_directories`, `list_directory`, and `fetch` in order. No local Artifact content appears in the prompt. Do not send a third acceptance attempt.

- [ ] **Step 4: Wait for verified completion and inspect Tool evidence**

Wait until generation is complete. Inspect every visible Tool card and final response. Record exact Tool names, call status, and typed error or `none` without retaining the fetched body.

Expected: exactly one successful call for each required Tool and no unexpected Tool.

- [ ] **Step 5: Reconcile response metadata with the Local manifest**

Pass only when all are equal:

```text
observed root == G:\workspace
observed Artifact ID == manifest.artifact_id
observed source_bytes == manifest.expected_bytes
observed source_sha256 == manifest.source_sha256
observed heading == manifest.marker
completion marker == CHATGPT_PROJECT_MCP_ACCEPTANCE_20260801
```

Any missing Tool card, mismatch, partial read, extra Tool, inferred value, or unavailable exact app yields `blocked`.

- [ ] **Step 6: Capture the sanitized final Tunnel delta**

Read `/metrics` and `/ui` again and calculate only:

```text
tools_call_delta = final_tools_call_count - baseline_tools_call_count
forward_event_delta = matching forward events after baseline_log_sequence
```

When telemetry is available, require both deltas to be positive. When it is unavailable under Step 1, require the three successful visible Tool cards and exact fetch metadata match instead. Finalize Browser tabs after the final visible evidence read and make no later Browser call in this execution.

---

### Task 6: Record compact evidence and complete repository verification

**Files:**
- Modify: `docs/superpowers/specs/2026-08-01-chatgpt-project-secure-mcp-tunnel-acceptance-design.md`
- Modify: `docs/superpowers/plans/2026-08-01-chatgpt-project-secure-mcp-tunnel-acceptance.md`
- Modify: `docs/reviews/README.md`

**Interfaces:**
- Consumes: Tasks 2–5 exact current outputs
- Produces: one compact audit record, final checked plan state, and repository commit containing no transient Browser transcript or secret

- [ ] **Step 1: Append the measured implementation record**

Append to the design only: Local validator/test counts, Inspector result, Browser Project title/memory, exact app, discovered Tool catalog, required Tool outcomes, metadata comparison, telemetry availability/deltas, upload count `0`, terminal marker, and final `pass` or `blocked` disposition.

- [ ] **Step 2: Update the compact review ledger**

Add one entry to `docs/reviews/README.md` with audit ID `CHATGPT-PROJECT-MCP-20260801`, date, Browser／Tunnel route, scope, one input Artifact and its digest, valid-gap count, affected non-normative design only, closure result, exact terminal marker, response digest when available, and retention disposition. Do not retain the full prompt, response, screenshots, attachment archive, Tunnel IDs, request IDs, or secrets.

- [ ] **Step 3: Mark only evidenced plan checkboxes complete**

Check each completed step. If Browser acceptance is blocked, leave unmet success predicates unchecked and state the exact blocker in the measured record.

- [ ] **Step 4: Run final repository checks and inspect the complete diff**

Run:

```powershell
git diff --check
git status --short
git diff --stat
git diff -- docs/superpowers/specs/2026-08-01-chatgpt-project-secure-mcp-tunnel-acceptance-design.md docs/superpowers/plans/2026-08-01-chatgpt-project-secure-mcp-tunnel-acceptance.md docs/reviews/README.md
```

Expected: no Architecture Owner or implementation claim changed; no unresolved placeholder; no secret or transient transcript retained.

- [ ] **Step 5: Stage and commit only repository-owned evidence**

Run:

```powershell
git add -- docs/superpowers/specs/2026-08-01-chatgpt-project-secure-mcp-tunnel-acceptance-design.md docs/superpowers/plans/2026-08-01-chatgpt-project-secure-mcp-tunnel-acceptance.md docs/reviews/README.md
git diff --cached --check
git diff --cached --stat
git commit -m 'docs: verify ChatGPT Project MCP route'
```

Expected: commit succeeds. Report checks that could not run and any remaining product-side risk.
