# ChatGPT Pro Secure MCP Tunnel Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `collaborating-with-chatgpt-pro`が、検証済みSecure MCP Tunnel-backed appをRepository Evidenceの第一候補として使い、Task Contractに従ってreadとwriteを分離する。

**Architecture:** 個人Global SkillがTunnelのpreflight、Task権限、delivery fallback、Transcript、local再検証を所有し、Repository `AGENTS.md`はProject固有routeとTunnel利用条件だけを保持する。Tunnelは公式のprivate MCP接続経路として扱うが、料金、entitlement、write確認UIはruntime factとして固定しない。

**Tech Stack:** Markdown Skill、YAML agent metadata、Secure MCP Tunnel、Filesystem MCP、PowerShell、Git、`skill-creator` validator、fresh-context agent scenario tests

## Global Constraints

- OpenAI公式のSecure MCP Tunnel guideを外部仕様の正本とする。
- `G:\workspace`外へTunnel scopeを拡張しない。
- App名や説明だけからread-only／write-capableを推測しない。
- `local_writes: true`でないTaskではTunnel writeを許可しない。
- ChatGPT Tool成功表示をRepository状態の正本にせず、Codexがlocal fileとdiffを再検証する。
- 添付方式を安全なfallbackとして維持する。
- 料金、entitlement、confirmation UIを未確認の固定仕様として記載しない。
- 既存のRepository作業中差分を変更、stage、commitしない。
- 新しいproduction dependencyを追加しない。

---

### Task 1: Current-Skill RED Scenarios

**Files:**
- Read: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/SKILL.md`
- Read: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/references/artifact-delivery-contract.md`
- Record temporarily: `$env:TEMP/collaborating-with-chatgpt-pro-secure-mcp-red.md`

**Interfaces:**
- Consumes: 現行Skillと、稼働中Tunnel appが`G:\workspace`へ到達できるという検証済み環境
- Produces: 現行SkillがTunnelを選べないことを示すverbatim baselineと、更新後scenarioの判定基準

- [ ] **Step 1: Define the control scenario**

Fresh-context agentへSkill本文を与えず、次を一回のtaskとして渡す。

```text
IMPORTANT: This is an application scenario. Decide the workflow you would use.

The user explicitly invoked $collaborating-with-chatgpt-pro for a review of
G:\workspace\development\GameEngine\miraikanai-engine. The in-app Browser,
target ChatGPT Project, project-only memory, and Pro mode are available.
A Secure MCP Tunnel-backed Filesystem MCP app is enabled in that ChatGPT
workspace. Its allowed root and tool catalog can be verified at runtime.
The user authorized reads but did not authorize writes. Choose the evidence
delivery path and state the preflight and fallback. Do not perform the task.
```

- [ ] **Step 2: Run five no-guidance control samples**

Use five fresh-context single-shot agents. Do not include the current or
candidate Skill. Record whether each response:

```text
TUNNEL_FIRST
VERIFY_SCOPE_AND_TOOLS
NO_WRITE
SAFE_FALLBACK
```

Expected: the control establishes that the scenario is understandable and at
least one sample selects or considers the verified Tunnel path. If none does,
rewrite the scenario before continuing.

- [ ] **Step 3: Define the current-Skill baseline scenario**

Run the same task with the complete current `SKILL.md` and
`artifact-delivery-contract.md` supplied as governing instructions.

- [ ] **Step 4: Run five current-Skill samples and verify RED**

Expected failure:

```text
CURRENT_SKILL_RED =
  states that the ChatGPT Project cannot read the repository directly
  AND selects attachment/excerpt/bundle without evaluating the verified Tunnel
```

Record the exact sentences and classifications in the temporary RED record.
Do not edit the Skill until this failure is observed.

---

### Task 2: Global Skill Tunnel Access Contract

**Files:**
- Modify: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/SKILL.md`
- Modify: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/references/prompt-generation-contract.md`
- Modify: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/references/artifact-delivery-contract.md`
- Modify: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/references/transcript-contract.md`
- Validate unchanged unless required: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/agents/openai.yaml`

**Interfaces:**
- Consumes: Task Contract、ChatGPT route、current official Secure MCP Tunnel behavior
- Produces: `secure-mcp-tunnel` delivery mode、`tunnel_reads`／`tunnel_writes`権限、Tool Evidence Transcript、chat delivery fallback

- [ ] **Step 1: Add Tunnel authorization fields to the Task Contract**

Replace the `authorized_actions` block in
`references/prompt-generation-contract.md` with:

```yaml
authorized_actions:
  browser_conversation: true
  browser_file_uploads: scoped | specific | denied
  tunnel_reads: scoped | specific | denied
  tunnel_writes: scoped | specific | denied
  local_reads: true | false
  local_writes: true | false
  external_side_effects: true | false
```

Immediately after the schema, add:

```text
Set `tunnel_writes` to `scoped` or `specific` only when `local_writes` is true
and the user requested change, build, fix, or execution. Keep it `denied` for
answer, research, review, diagnose, and plan work.
```

- [ ] **Step 2: Add Tunnel preflight to the prompt contract**

Add an observable Secure MCP preflight requiring:

```text
- visible target app availability in the routed ChatGPT chat;
- allowed-root result covering the task target;
- target reachability through a read-only probe;
- required read tools and, only when authorized, required write tools;
- healthy/ready/connected tunnel evidence when locally available;
- no inference of capabilities from app name or description.
```

- [ ] **Step 3: Generalize artifact delivery into evidence delivery**

In `references/artifact-delivery-contract.md`:

```text
1. Add `secure-mcp-tunnel` to the manifest delivery values.
2. Make a verified Tunnel-backed app the first delivery mode.
3. Require allowed-root, reachability, and tool-catalog checks.
4. Keep chat attachment, inline excerpt, review bundle, split parts, and
   explicitly authorized Project Source as fallbacks.
5. Route native or unsupported formats to the least-lossy fallback.
6. Treat incomplete Tunnel coverage as fallback-or-blocked, never complete.
```

Use the following decision order:

```text
verified secure-mcp-tunnel
  -> exact supported chat attachment
  -> sufficient inline excerpt
  -> review bundle
  -> split parts
  -> explicitly authorized Project Source
  -> blocked
```

- [ ] **Step 4: Replace the unconditional local-access prohibition**

In `SKILL.md`, replace:

```text
A ChatGPT Project cannot read the local repository directly.
```

with a conditional access rule:

```text
A ChatGPT Project can inspect authorized local artifacts through a verified
Secure MCP Tunnel-backed app. Prefer that path when its allowed root,
reachability, and required tools cover the task. Otherwise use the least-lossy
authorized chat delivery mode.
```

Update the required-capabilities, context acquisition, preflight, converse,
adjudication, quick-reference, and common-mistakes sections so that:

```text
- Tunnel availability is verified, not assumed.
- Reads remain task-scoped.
- Tunnel writes require both `local_writes: true` and `tunnel_writes`
  authorization.
- Scope-exceeding, destructive, Git, credential, permission, purchase,
  publish, and release actions are not implied.
- ChatGPT changes require Codex local re-read, diff review, and task-specific
  validation.
```

- [ ] **Step 5: Extend the visible Transcript contract**

Add these fields to `references/transcript-contract.md`:

```yaml
evidence_delivery:
  mode: secure-mcp-tunnel | chat-attachment | inline-excerpt | review-bundle | split-parts | project-source
  visible_app_identity: value | not-applicable
  allowed_scope_verified: true | false | not-applicable
  target_reachability_verified: true | false | not-applicable
  tool_calls: []
  local_write_verification: []
```

Require each material Tool call to record visible Tool name, task-relevant
arguments, result/error, read/write class, and Task Contract authorization.
Exclude tokens, API keys, Tunnel secrets, cookies, account identifiers, and
unneeded absolute paths from durable Transcripts.

- [ ] **Step 6: Review metadata**

Confirm `agents/openai.yaml` still satisfies:

```yaml
interface:
  display_name: "Collaborate with ChatGPT Pro"
  short_description: "Consult ChatGPT Pro with a task-specific prompt"
  default_prompt: "Use $collaborating-with-chatgpt-pro to consult ChatGPT Pro for this request and adjudicate the result."

policy:
  allow_implicit_invocation: false
```

Do not change it unless validation identifies a concrete metadata gap.

---

### Task 3: Repository Tunnel Route

**Files:**
- Modify: `AGENTS.md`

**Interfaces:**
- Consumes: Global Skill access and authorization contracts from Task 2
- Produces: Miraikanai Project route that permits verified Tunnel access without duplicating the Global Skill workflow

- [ ] **Step 1: Replace the obsolete route assertion**

Replace:

```text
- The ChatGPT Project cannot directly access this local repository. Provide
  only the task-relevant excerpts or artifacts required for the consultation.
```

with:

```text
- Prefer the verified Secure MCP Tunnel-backed Filesystem MCP app for
  task-relevant artifacts under `G:\workspace`. Before relying on it, verify
  the allowed root, this repository's reachability, and the required tool
  capabilities. Do not infer read-only or write capability from the app name.
- If Tunnel access is unavailable or incomplete, use the Global Skill's
  authorized chat-delivery fallback. Tunnel reads and writes remain limited by
  the Task Contract, and ChatGPT-originated changes require local diff and
  validation by Codex.
```

- [ ] **Step 2: Verify route-only responsibility**

Confirm the section contains only:

```text
Project URL
expected visible Project title
required memory mode
required Pro mode
new chat per distinct outcome
fallback denial for destination/mode
repository-specific Tunnel scope
```

Remove no Global Skill workflow and add no duplicate Prompt, Transcript, or
adjudication schema.

---

### Task 4: GREEN Scenarios and Static Validation

**Files:**
- Test: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/`
- Test: `AGENTS.md`
- Test: `$env:TEMP/collaborating-with-chatgpt-pro-secure-mcp-red.md`

**Interfaces:**
- Consumes: Updated Global Skill and Repository route
- Produces: Forward-test evidence, static validation, bounded remaining-risk report

- [ ] **Step 1: Run five candidate wording samples**

Use the Task 1 scenario in five fresh contexts with the complete updated
`SKILL.md` and relevant contracts.

Every sample must satisfy:

```text
TUNNEL_FIRST = true
VERIFY_SCOPE_AND_TOOLS = true
NO_WRITE = true
SAFE_FALLBACK = true
```

Read every response manually. If interpretations vary, tighten the positive
decision recipe and repeat until all five converge.

- [ ] **Step 2: Run authorization variation scenarios**

Run one fresh-context sample for each:

```text
review-only + write-capable app
explicit scoped change + write-capable app
Tunnel unavailable + attachable exact files
Tunnel incomplete + non-attachable required native artifact
unexpected out-of-scope write
```

Expected:

```text
review-only -> no write
explicit change -> scoped write + Codex local verification
Tunnel unavailable -> authorized attachment fallback
incomplete coverage -> blocked
unexpected write -> stop further calls + preserve and inspect local diff
```

- [ ] **Step 3: Run the Skill validator**

Run:

```powershell
python 'C:\Users\y2ikg\.codex\skills\.system\skill-creator\scripts\quick_validate.py' 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro'
```

Expected: exit code `0` and a valid Skill result.

- [ ] **Step 4: Run static searches**

Run:

```powershell
rg -n "cannot read the local repository directly|cannot directly access this local repository|Assuming a ChatGPT Project can read the working tree" `
  'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro' `
  '.\AGENTS.md'
rg -n "secure-mcp-tunnel|tunnel_reads|tunnel_writes|allowed root|local verification" `
  'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro' `
  '.\AGENTS.md'
```

Expected: obsolete unconditional assertions produce no matches; new contract
terms are present in the intended files.

- [ ] **Step 5: Verify Repository diff hygiene**

Run:

```powershell
git diff --check
git status --short
git diff --stat
git diff -- AGENTS.md
```

Inspect the complete `AGENTS.md` diff and confirm unrelated existing changes
remain untouched.

- [ ] **Step 6: Verify the Global Skill diff**

The personal Skill is outside this Repository. Compare all changed files
against pre-edit copies or exact pre-edit content captured in Task 1. Confirm:

```text
frontmatter remains valid
relative references resolve
official fact and local policy remain separated
no local secret or account identifier was added
no Project URL or Miraikanai-specific route leaked into the Global Skill
no pricing or confirmation guarantee was added
```

- [ ] **Step 7: Report completion**

Report:

```text
changed Global Skill files
changed Repository route
RED and GREEN scenario counts and outcomes
quick_validate result
Git checks actually run
unrelated pre-existing dirty files left unchanged
remaining runtime risks, including Tunnel health and product UI changes
```
