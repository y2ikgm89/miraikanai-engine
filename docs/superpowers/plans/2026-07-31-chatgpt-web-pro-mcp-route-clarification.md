# ChatGPT Web Pro MCP Route Clarification Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Browser版ChatGPTの`Pro`／Secure MCP Tunnel routeを、component ownershipが曖昧にならない現行Skillへ後方互換性なく更新し、Web UIで検証する。

**Architecture:** CodexはChatGPT Desktop app内蔵Browserのcontrol planeであり、ブラウザ版ChatGPTが`G Workspace Readonly`を通じてSecure MCP TunnelのMCP clientになる。browser turnはvisible／collapsed `Pro`を必須とし、MCP appまたはTool evidenceが欠ければ`非常に高い`へ切り替えずfail closedする。

**Tech Stack:** Markdown Skill contracts, PowerShell 7 static validator and lifecycle tests, ChatGPT Desktop app built-in Browser, browser ChatGPT, Secure MCP Tunnel.

## Global Constraints

- Browser版ChatGPTだけがTunnel-backed `G Workspace Readonly` appを使用する。
- CodexはBrowserを操作・検証するcontrol planeであり、TunnelのMCP clientとは書かない。
- Response performanceはbrowser turnごとにexact `Pro`を可視確認する。
- `非常に高い`、`高い`、`中程度`、API、別app、添付、upload、paste、Project Source、Tunnel writeへのfallbackを禁止する。
- `Pro`＋app＋Tool successを同時に確認できない場合は`blocked`とする。
- 過去のplans／measured evidenceは改変しない。current Skill、current references、validator、current `AGENTS.md`だけを更新する。
- 新規dependency、Profile、Tunnel executable、root、Tool catalogは変更しない。

---

### Task 1: Add a failing route-ownership and Pro policy validator

**Files:**
- Modify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1`
- Test: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1`

**Consumes:** current Skill and five reference contracts.

**Produces:** a named static result proving browser ChatGPT is the Tunnel client, Codex is Browser control plane, and visible／collapsed `Pro` is the only response-performance success state.

- [x] **Step 1: Add one failing contract assertion**

Add this result after the existing required-file checks. Do not change current Skill or references in this step.

```powershell
$browserWebProRouteOwnership = Has-All $allContractText @(
    '(?is)browser ChatGPT.{0,260}(?:Secure MCP Tunnel|`G Workspace Readonly`).{0,260}(?:MCP client|Tool call)'
    '(?is)Codex.{0,220}(?:control plane|operat(?:e|es)).{0,180}(?:ChatGPT Desktop app|built-in Browser)'
    '(?is)response_performance_selected:\s*Pro'
    '(?is)collapsed_button_text:\s*Pro'
    '(?is)(?:非常に高い|高い|中程度).{0,220}(?:fallback|substitute).{0,220}`blocked`'
)
Add-Result 'BrowserChatGPTProRouteOwnership' $browserWebProRouteOwnership
```

- [x] **Step 2: Run RED**

Run:

```powershell
& 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1'
```

Expected: exit code `1` and `FAIL BrowserChatGPTProRouteOwnership`; no current contract text has changed.

- [x] **Step 3: Retain only safe RED evidence**

Keep the named failed result and failure count in task-local evidence. Do not store profile contents, Tunnel IDs, process IDs, cookies, account identifiers, or Local Artifact content.

### Task 2: Replace the active route contract without a fallback

**Files:**
- Modify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\SKILL.md`
- Modify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\references\prompt-generation-contract.md`
- Modify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\references\artifact-delivery-contract.md`
- Modify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\references\transcript-contract.md`
- Modify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\references\adjudication-and-stop-rules.md`
- Modify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\references\response-completion-gate.md`
- Modify: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1`
- Modify: `AGENTS.md`

**Consumes:** Task 1 failing assertion and `docs/superpowers/specs/2026-07-31-chatgpt-web-pro-mcp-route-clarification-design.md`.

**Produces:** one active contract with exact `Pro` browser-Web success tuple and no lower-mode fallback; all static checks align to it.

- [x] **Step 1: Define one canonical route sentence in `SKILL.md`**

Replace ambiguous `Codex in-app Browser`／`Codex desktop app` ownership statements with:

```markdown
Codex operates the ChatGPT Desktop app's built-in Browser as the control plane.
Browser ChatGPT is the MCP client that invokes the `G Workspace Readonly` app through Secure MCP Tunnel.
```

Keep `browser:control-in-app-browser` as the required capability name; it is a tool name, not a claim that Codex is the Tunnel client.

- [x] **Step 2: Replace every active response-performance success tuple**

In `SKILL.md`, prompt and transcript contracts, replace normative current-state instances with:

```yaml
response_performance_control: 応答性能
response_performance_selected: Pro
response_performance_verified: true
collapsed_button_text: Pro
```

State that `Pro` is a `user-requirement`, not an OpenAI-wide recommendation. State once in the prompt contract that unavailable `Pro`, absent app, or absent Tool evidence is `blocked`; no lower response-performance option is a fallback.

- [x] **Step 3: Align the five reference contracts and `AGENTS.md`**

The prompt contract owns pre-send Browser identity and `Pro`; artifact delivery owns browser ChatGPT-to-Tunnel read ownership; transcript owns the per-turn snapshot; response completion owns visible Tool completion; adjudication owns `blocked`. Link to the canonical owner rather than duplicating route definitions. In `AGENTS.md`, state that Codex operates the ChatGPT Desktop app built-in Browser and browser ChatGPT uses the verified Tunnel-backed app. Preserve Project, memory, four read-only Tools, no-upload path, and fail-closed rules.

- [x] **Step 4: Make the validator match the new contract**

Replace `非常に高い`／non-Pro success assertions with exact `Pro`, add Web-client and Codex-control-plane assertions, and rename checks so they retain neither `CompatibleMode` nor `non-Pro` wording.

- [x] **Step 5: Run GREEN**

Run:

```powershell
& 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1'
```

Expected: exit code `0`, `PASS BrowserChatGPTProRouteOwnership`, no active `非常に高い` success tuple or `non-Pro` compatibility assertion, and `TOTAL_FAILURES=0`.

### Task 3: Validate the full local contract and browser-Web route

**Files:**
- Test: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1`
- Test: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\test_ensure_secure_mcp_tunnel.ps1`
- Test: `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\.venv\Scripts\python.exe -m readonly_local_files.server --self-test`
- Test: ChatGPT Desktop app built-in Browser, browser ChatGPT Project

**Consumes:** Task 2 active contract.

**Produces:** exact local contract evidence and a browser-Web `Pro` disposition without file upload.

- [x] **Step 1: Run local validators**

Run the static validator, lifecycle suite, and server self-test for `G:\workspace`. Expected: `TOTAL_FAILURES=0`, lifecycle suite all pass, exact four read-only Tools, and no forbidden Tool.

- [x] **Step 2: Load Browser control instructions before Browser work**

Read `browser:control-in-app-browser` completely. Browser work begins only after the local validators pass.

- [x] **Step 3: Run one fresh browser-Web Pro acceptance**

Using the ChatGPT Desktop app built-in Browser controlled by Codex, open the exact Project, create a new browser ChatGPT chat, and visibly verify sign-in, Project title, memory `プロジェクトのみ`, `応答性能: Pro`, collapsed `Pro`, and `G Workspace Readonly` read-only identity. Send one metadata-only prompt requiring the authorized exact Tool sequence. Do not attach, upload, paste Local Artifact content, add Project Source, use another app, or change response performance.

- [x] **Step 4: Record the disposition and finalize Browser tabs**

If exact Tool cards prove the `Pro` route, record the visible calls and success. If `Pro`, app, or Tool calls are unavailable, record only observed mismatch and `blocked`. Never claim browser-Web Pro works from local Tunnel health alone. After the final Browser read, call tab finalization as the last Browser operation.

### Task 4: Record active-policy evidence and commit

**Files:**
- Modify: `docs/superpowers/specs/2026-07-31-chatgpt-web-pro-mcp-route-clarification-design.md`
- Modify: `docs/superpowers/plans/2026-07-31-chatgpt-web-pro-mcp-route-clarification.md`

**Consumes:** local and Browser disposition from Task 3.

**Produces:** one commit that distinguishes `official-spec`, `user-requirement`, and `measured` evidence without rewriting historical records.

- [x] **Step 1: Update only actual evidence**

Record validator, lifecycle, self-test catalog, Browser response-performance, Tool-card evidence, upload count, and final disposition. Do not record secrets, Profile body, process IDs, full prompts, transcripts, or Local Artifact content.

- [x] **Step 2: Verify and commit repository scope**

Run `git diff --check`, `git status --short`, `git diff --stat`, and inspect the complete diff. Stage only `AGENTS.md`, this design, and this plan; then run `git diff --cached --check` and commit with `docs: require ChatGPT web Pro MCP route`. A blocked Browser result is an honest disposition, not a success claim.
