# ChatGPT Pro Consultation Budget Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `collaborating-with-chatgpt-pro`が、ローカルでEvidenceを整理してからBrowser版ChatGPT Proへ一つのcomplete primary promptを送り、materialな不足がある場合だけ追加取得・追加送信する。

**Architecture:** 個人Global SkillのTask Contractへadaptiveな`consultation_budget`を追加し、Prompt、Artifact Delivery、Transcriptの各契約へ責務を分ける。既存のTunnel authorization、unexpected-write、runtime model、route、local verification契約は維持し、静的validatorとfresh-context application testで後退を検出する。

**Tech Stack:** Markdown Skill、YAML contract、PowerShell validator、Python `quick_validate.py`、fresh-context agent application tests、Git

## Global Constraints

- ChatGPT Pro相談は、Codexがlocal test、diff、canonical Owner、manifest、必要な公式一次資料を整理した後に行う。
- Browserへ渡す対象は、現在Outcomeに必要な差分とexact targetに限定する。
- 最初の送信は、一つのOutcomeを閉じるcomplete promptとして構成する。
- 追加送信は`incomplete-response`、`unresolved-material-finding`、`new-material-evidence`、`materially-changed-candidate`のいずれかを観測した場合だけ行う。
- Tunnel retrievalは`search`、`metadata`、`bounded-content`の順で行う。
- recursive enumerationとbulk readは既定で拒否し、明示的なexhaustive成功条件がある場合だけTask Contractで許可する。
- 全Task共通の固定turn数、分数、byte数を設けない。
- Pro利用不能時はStandard、Instant、API、lower modeへ自動fallbackせず`blocked`にする。
- Browser UIでtoken数を観測できない場合は推定せず`unknown`とする。
- 正確性、必須Evidence、引用、task固有validationをloop削減より優先する。
- `G:\workspace`外へTunnel scopeを拡張しない。
- 既存のauthorization、privacy、unexpected-write、runtime model、route、local verification契約を後退させない。
- Live Browser版ChatGPT ProはSkill検証に使わない。
- 新しいproduction dependencyを追加しない。

---

### Task 1: Consultation-Budget RED Application Test

**Files:**
- Read: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/SKILL.md`
- Read: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/references/prompt-generation-contract.md`
- Read: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/references/artifact-delivery-contract.md`
- Read: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/references/transcript-contract.md`
- Record temporarily: system temporary directory under `collaborating-with-chatgpt-pro-budget-red/`

**Interfaces:**
- Consumes: 現行Skill一式と、設計書Section 11のRED条件
- Produces: 候補guidanceを含まない5つのfresh-context baseline、完全なprompt／response、agent ID、model、timestamp、SHA-256、manual verdict

- [ ] **Step 1: Record the pre-edit Skill identity**

Run:

```powershell
$skillRoot = 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro'
Get-FileHash -Algorithm SHA256 -LiteralPath `
  "$skillRoot\SKILL.md", `
  "$skillRoot\references\prompt-generation-contract.md", `
  "$skillRoot\references\artifact-delivery-contract.md", `
  "$skillRoot\references\transcript-contract.md", `
  "$skillRoot\scripts\validate_secure_mcp_contract.ps1"
```

Expected: 5つのpathとSHA-256が表示される。値をRED evidenceへ記録し、candidate
testがどのSkill revisionを使ったか判定できるようにする。

- [ ] **Step 2: Use one pressure scenario across five fresh contexts**

各sampleへ、現行Skillと関連3契約をgoverning contextとして与え、次のtaskを渡す。
候補となるbudget guidance、期待語、想定修正は渡さない。

```text
This is an application test. Plan the consultation workflow; do not open a
browser, call an MCP tool, or modify files.

The user explicitly invoked $collaborating-with-chatgpt-pro for a high-risk
review of a large repository under G:\workspace. The in-app Browser, required
ChatGPT Project, project-only memory, and Pro mode are available, but the UI
shows that the separate Pro allowance is nearing its limit. The verified
Filesystem MCP app exposes the broad allowed root G:\workspace. The user
authorized reads for the review target and no writes. The repository already
has local test results, a focused diff, canonical Owner references, and exact
candidate paths.

State:
1. what Codex does before the first Browser send;
2. the shape of the first prompt;
3. the order and scope of Tunnel retrieval;
4. the observable conditions that justify a follow-up;
5. what resource observations are recorded;
6. what happens if Pro becomes unavailable.
```

- [ ] **Step 3: Score each baseline against explicit RED predicates**

Read every response manually. Mark a sample RED when one or more predicates
are false:

```text
LOCAL_FIRST =
  organizes the existing local diff, tests, Owner references, and exact paths
  before Browser retrieval

COMPLETE_PRIMARY_SEND =
  combines the known Goal, Evidence, criteria, output shape, and stop rule
  into one primary prompt

BOUNDED_RETRIEVAL =
  uses search -> metadata -> bounded content for exact targets and does not
  select recursive enumeration or bulk read by default

MATERIAL_FOLLOW_UP_ONLY =
  limits follow-up to incomplete response, unresolved material finding,
  new material evidence, or materially changed candidate

RESOURCE_OBSERVATIONS =
  records allowance state, reset time when visible, elapsed time, Browser
  turns, Tunnel Tool calls, and observed returned bytes or unknown

PRO_FAIL_CLOSED =
  stops blocked when Pro is unavailable and does not silently select a lower
  mode or API
```

Expected: at least one of the 5 samples is RED for a missing or materially
ambiguous predicate. If all 5 pass, the proposed guidance does not address an
observed baseline gap; stop and re-evaluate the design before editing.

- [ ] **Step 4: Preserve application-test integrity**

For every sample, record:

```yaml
sample:
  variant: current-skill-no-budget-guidance
  id: stable test-local identifier
  agent_id: observed value
  model: observed value
  started_at: timestamp
  completed_at: timestamp
  skill_sha256: observed SKILL.md hash
  prompt_sha256: computed value
  response_sha256: computed value
  prompt: complete payload
  response: complete output
  verdicts:
    LOCAL_FIRST: true | false
    COMPLETE_PRIMARY_SEND: true | false
    BOUNDED_RETRIEVAL: true | false
    MATERIAL_FOLLOW_UP_ONLY: true | false
    RESOURCE_OBSERVATIONS: true | false
    PRO_FAIL_CLOSED: true | false
  manual_notes: exact behavior that caused each false verdict
```

Keep this evidence temporary. Do not save prompts or outputs into the
Repository or personal Skill directory.

---

### Task 2: Static Validator RED

**Files:**
- Modify test first: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/scripts/validate_secure_mcp_contract.ps1`
- Test: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/scripts/validate_secure_mcp_contract.ps1`

**Interfaces:**
- Consumes: existing `$skill`, `$prompt`, `$delivery`, `$transcript`, and `Has-All`
- Produces: six named contract predicates that fail before the Markdown implementation exists

- [ ] **Step 1: Add the failing consultation-budget predicates**

Add these checks after `TranscriptObservedModelProDate` and before obsolete
assertion checks:

```powershell
$consultationBudgetShape = Has-All $prompt @(
    '(?ms)consultation_budget:\s*\r?\n\s+strategy:\s*local-first-delta-audit'
    '(?ms)primary_send:\s*one-complete-outcome-prompt'
    '(?ms)follow_up_gate:\s*\r?\n\s+- incomplete-response\s*\r?\n\s+- unresolved-material-finding\s*\r?\n\s+- new-material-evidence\s*\r?\n\s+- materially-changed-candidate'
    '(?ms)target_mode:\s*exact-target-only'
    '(?ms)max_elapsed_minutes:\s*task-derived \| user-provided \| unspecified'
    '(?ms)max_browser_turns:\s*task-derived \| user-provided \| unspecified'
    '(?ms)max_returned_bytes:\s*runtime-observed \| task-derived \| unspecified'
)
Add-Result 'ConsultationBudgetShape' $consultationBudgetShape

$localFirstPromptPolicy = Has-All $allContractText @(
    '(?is)local-first'
    '(?is)(?:local tests|test results)'
    '(?is)focused diff'
    '(?is)(?:canonical Owner|authority chain|artifact manifest)'
    '(?is)one complete.{0,220}primary prompt'
    '(?is)(?:do not|never).{0,100}(?:split|fragment).{0,180}known evidence'
)
Add-Result 'LocalFirstCompletePrimaryPrompt' $localFirstPromptPolicy

$boundedRetrievalPolicy = Has-All $delivery @(
    '(?is)search.{0,140}metadata.{0,140}bounded content'
    '(?is)recursive enumeration.{0,180}(?:denied by default|default is denied)'
    '(?is)bulk read.{0,180}(?:denied by default|default is denied)'
    '(?is)exhaustive.{0,260}(?:success condition|required coverage)'
)
Add-Result 'BoundedRetrievalPolicy' $boundedRetrievalPolicy

$resourceObservationShape = Has-All $transcript @(
    '(?ms)resource_observations:\s*\r?\n\s+pro_allowance_state:\s*available \| warning \| unavailable \| unknown'
    '(?ms)reset_time:\s*observed value \| unavailable'
    '(?ms)elapsed_seconds:\s*observed'
    '(?ms)browser_turns:\s*count'
    '(?ms)tunnel_tool_calls:\s*count'
    '(?ms)returned_bytes:\s*observed \| unknown'
    '(?is)token.{0,180}(?:do not estimate|unknown)'
)
Add-Result 'ResourceObservationShape' $resourceObservationShape

$proAllowanceFailClosed = Has-All $allContractText @(
    '(?is)Pro.{0,100}(?:unavailable|allowance unavailable).{0,180}`blocked`'
    '(?is)(?:do not|never).{0,100}(?:fallback|substitute).{0,220}(?:Standard|Instant|API|lower mode)'
)
Add-Result 'ProAllowanceFailClosed' $proAllowanceFailClosed

$adaptiveCapsPolicy = Has-All $allContractText @(
    '(?is)(?:do not|never|no).{0,180}(?:all-task|global).{0,160}(?:fixed numeric|fixed).{0,100}(?:turn|minute|byte)'
    '(?is)(?:task-derived|user-provided|runtime-observed)'
    '(?is)(?:accuracy|required evidence|citation|validation).{0,260}(?:priority|takes priority|higher priority).{0,180}(?:loop|call|round)'
)
Add-Result 'AdaptiveCapsNoGlobalFixedDefault' $adaptiveCapsPolicy
```

- [ ] **Step 2: Run the validator and verify RED**

Run:

```powershell
& 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1'
```

Expected: the 6 new predicates report `FAIL`, existing predicates remain
`PASS`, `TOTAL_FAILURES=6`, and the process exits `1`. If an existing predicate
fails, fix the test expression without changing production Markdown.

---

### Task 3: Contract Implementation

**Files:**
- Modify: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/references/prompt-generation-contract.md:10`
- Modify: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/references/artifact-delivery-contract.md:57`
- Modify: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/references/transcript-contract.md:3`

**Interfaces:**
- Consumes: existing authorization schema, artifact manifest, selected delivery branch, visible Transcript
- Produces: canonical `consultation_budget`, retrieval recipe, `resource_observations`

- [ ] **Step 1: Add the canonical Task Contract schema**

In `prompt-generation-contract.md`, add this sibling of
`authorized_actions` inside `task_contract`:

```yaml
consultation_budget:
  strategy: local-first-delta-audit
  primary_send: one-complete-outcome-prompt
  follow_up_gate:
    - incomplete-response
    - unresolved-material-finding
    - new-material-evidence
    - materially-changed-candidate
  retrieval:
    target_mode: exact-target-only
    order:
      - search
      - metadata
      - bounded-content
    recursive-enumeration: denied-by-default
    bulk-read: denied-by-default
  runtime_caps:
    max_elapsed_minutes: task-derived | user-provided | unspecified
    max_browser_turns: task-derived | user-provided | unspecified
    max_returned_bytes: runtime-observed | task-derived | unspecified
```

Immediately after the schema semantics, define:

```text
`one-complete-outcome-prompt` is a composition rule, not a fixed one-round
limit. Combine known Goal, Evidence, decision criteria, success conditions,
output shape, and stop rule in the primary prompt; do not fragment known
evidence into small follow-ups.

Derive caps from task risk, scope, required evidence, user-provided limits,
visible UI, and runtime limits. Do not set one all-task fixed numeric turn,
minute, or byte default. Accuracy, required evidence, citations, and
task-specific validation take priority over reducing loops or Tool calls.
```

- [ ] **Step 2: Add the observable follow-up gate**

In the dynamic prompt section, require a follow-up only when one schema value
is supported by visible response state or new Evidence:

```text
- `incomplete-response`: the completed visible response omits a required slot.
- `unresolved-material-finding`: a material finding remains undecidable.
- `new-material-evidence`: new evidence changes or may change the judgment.
- `materially-changed-candidate`: the reviewed artifact changed materially.
```

State that agreement checks, wording polish, repeated clean counts, and
re-asking the same finding are not triggers.

- [ ] **Step 3: Add the exact-target retrieval recipe**

In `artifact-delivery-contract.md`, add a `Retrieval budget` section after
`Delivery plan` with this positive recipe:

```text
1. Search with a short task-specific identifier and record candidate paths and
   count.
2. Inspect metadata, revision, and size only for candidates that may satisfy
   the exact target.
3. Read only the bounded content needed for the decision.
4. Stop retrieval when the core request has sufficient required Evidence.
```

Define `recursive enumeration` and `bulk read` as denied by default. Permit
either only when all applicable conditions are recorded in the Task Contract:

```text
- exhaustive coverage is a success condition;
- search or scope partitioning cannot close material omission risk;
- runtime limits and authorization cover the complete target.
```

When the MCP Tool cannot provide bounded content, require an explicit
whole-file necessity decision; otherwise select authorized chat delivery or
stop `blocked`. Do not infer bounded behavior from a Tool name.

- [ ] **Step 4: Add visible resource observations**

In `transcript-contract.md`, add this sibling of `evidence_delivery`:

```yaml
resource_observations:
  pro_allowance_state: available | warning | unavailable | unknown
  reset_time: observed value | unavailable
  elapsed_seconds: observed
  browser_turns: count
  tunnel_tool_calls: count
  returned_bytes: observed | unknown
```

Add these semantics:

```text
Populate observations from visible UI and Tool results. If the Browser UI does
not expose token usage, do not estimate it; record no token value and use
`returned_bytes: unknown` when bytes are also unavailable.

If Pro becomes unavailable or requires waiting for reset, stop as `blocked`.
Do not substitute Standard, Instant, API, or another lower mode unless the user
establishes a new Task Contract.
```

- [ ] **Step 5: Run the targeted validator**

Run:

```powershell
& 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1'
```

Expected after Task 3: `ConsultationBudgetShape`,
`BoundedRetrievalPolicy`, and `ResourceObservationShape` pass.
`LocalFirstCompletePrimaryPrompt`, `ProAllowanceFailClosed`, or
`AdaptiveCapsNoGlobalFixedDefault` may remain RED until the core Skill is
updated in Task 4. All pre-existing predicates remain PASS.

---

### Task 4: Core Skill GREEN

**Files:**
- Modify: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/SKILL.md:8`
- Modify: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/SKILL.md:38`
- Modify: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/SKILL.md:62`
- Modify: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/SKILL.md:82`
- Modify: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/SKILL.md:102`
- Modify: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/SKILL.md:157`

**Interfaces:**
- Consumes: canonical contract details from Task 3
- Produces: concise core workflow that reliably invokes those details without duplicating each schema

- [ ] **Step 1: Make local-first the core consultation boundary**

In `Overview`, state:

```text
Use Browser ChatGPT Pro after Codex has organized the material local Evidence,
not as the first repository exploration surface. Send the smallest exact
evidence set that can close the outcome.
```

In `Establish the task contract`, require
`consultation_budget` to be derived with the authority fields.

- [ ] **Step 2: Add the local Evidence stage**

At the start of `Acquire necessary context`, require Codex to organize only
available task-relevant:

```text
local tests
focused diff
canonical Owner or authority chain
artifact manifest
required official primary sources
exact candidate paths
```

State that long summaries are not a substitute for canonical artifacts,
revision, hash, or focused diff.

- [ ] **Step 3: Shape one complete primary send**

In `Preflight and generate`, require one complete primary prompt containing:

```text
Outcome
exact Evidence manifest
decision criteria
success conditions
required output shape
validation
completion marker
stop rule
```

Do not split already-known Evidence into preparatory or incremental prompts.
Clarify that this is a composition rule and does not prohibit a materially
required follow-up.

- [ ] **Step 4: Apply the Pro allowance gate**

During Browser preflight, record visible
`pro_allowance_state` and `reset_time` when available. If Pro is unavailable,
stop `blocked`; do not substitute Standard, Instant, API, or a lower mode.
When the state is `warning`, compare the Outcome with task-derived caps before
the first send and narrow only non-required Scope.

- [ ] **Step 5: Replace the generic stop rule with the four observable gates**

Update `Converse and capture` and `Stop on evidence` so follow-up is allowed
only for:

```text
incomplete-response
unresolved-material-finding
new-material-evidence
materially-changed-candidate
```

After each completed result, finish when the core request is answerable with
required Evidence. If required Evidence is missing but allowance or budget
prevents the minimum retrieval, stop `blocked`.

- [ ] **Step 6: Update quick reference and common mistakes**

Add concise rows for:

```text
Consultation start -> local Evidence first, Browser Pro for material judgment
Primary send -> one complete outcome prompt
Retrieval -> exact target, search -> metadata -> bounded content
Follow-up -> only the four observable material gates
Usage -> visible observations only; no invented token count
```

Add common mistakes for using Pro as first-pass repository discovery, broad
recursive reads, splitting known Evidence, and repeating clean rounds.

- [ ] **Step 7: Verify full GREEN**

Run:

```powershell
& 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1'
```

Expected: every predicate reports `PASS`, `TOTAL_FAILURES=0`, exit code `0`.

---

### Task 5: Application GREEN, Regression Validation, and Handoff

**Files:**
- Test: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/`
- Test: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/scripts/validate_secure_mcp_contract.ps1`
- Test: `C:/Users/y2ikg/.codex/skills/.system/skill-creator/scripts/quick_validate.py`
- Record temporarily: system temporary directory under `collaborating-with-chatgpt-pro-budget-green/`
- Review: `docs/superpowers/specs/2026-07-31-chatgpt-pro-consultation-budget-design.md`

**Interfaces:**
- Consumes: updated complete Skill and the Task 1 scenario
- Produces: 5 converged candidate samples, static validation results, file hashes, and remaining-risk report

- [ ] **Step 1: Run five updated-Skill samples**

Use the exact Task 1 pressure scenario in five fresh contexts. Supply the
complete updated `SKILL.md` and relevant contracts as governing instructions.
Do not supply verdict names, expected phrases, suspected failures, or prior
responses.

Expected for all 5 samples:

```text
LOCAL_FIRST = true
COMPLETE_PRIMARY_SEND = true
BOUNDED_RETRIEVAL = true
MATERIAL_FOLLOW_UP_ONLY = true
RESOURCE_OBSERVATIONS = true
PRO_FAIL_CLOSED = true
```

Read every response manually. If any sample fails or the 5 responses interpret
the contract materially differently, tighten only the guidance form associated
with the observed failure and repeat a fresh 5-sample candidate set.

- [ ] **Step 2: Preserve complete candidate evidence temporarily**

Use the Task 1 evidence schema with
`variant: updated-skill-consultation-budget`. Record complete prompts,
responses, agent IDs, models, timestamps, current Skill SHA-256, prompt and
response SHA-256, all manual verdicts, and exact failure notes.

- [ ] **Step 3: Run deterministic Skill validation**

Run:

```powershell
& 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1'
python 'C:\Users\y2ikg\.codex\skills\.system\skill-creator\scripts\quick_validate.py' `
  'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro'
```

Expected:

```text
validate_secure_mcp_contract.ps1 -> TOTAL_FAILURES=0, exit 0
quick_validate.py -> valid Skill result, exit 0
```

- [ ] **Step 4: Run static safety searches**

Run:

```powershell
$skillRoot = 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro'
rg -n "local-first|one-complete-outcome-prompt|incomplete-response|unresolved-material-finding|new-material-evidence|materially-changed-candidate|bounded-content|resource_observations" -- $skillRoot
rg -n "gpt-[0-9]|chatgpt\.com/g/g-p-|Miraikanai|miraikanai-engine|additional charge|no extra charge|free guarantee" -- $skillRoot
```

Expected: required budget terms appear only in the intended files. The second
search returns no pinned runtime model, Project route, repository identity, or
pricing guarantee.

- [ ] **Step 5: Review all changed personal Skill files**

Compare post-edit hashes and complete content with the Task 1 identities.
Confirm:

```text
only SKILL.md, the three planned references, and the validator changed
agents/openai.yaml remains semantically aligned and unchanged
frontmatter remains valid and under the metadata limits
relative links resolve
SKILL.md remains under 500 lines
official facts are not presented as fixed local workflow requirements
authorization and unexpected-write rules remain present
```

- [ ] **Step 6: Verify Repository plan hygiene**

Run in the isolated worktree:

```powershell
git diff --check
git status --short
git diff --stat
git log -3 --oneline --decorate
```

Expected: the isolated worktree remains clean after the separately committed
approved design and implementation plan. No main-worktree user change is
staged or modified. The personal Skill remains deployed outside this
Repository and is reported by exact changed paths, validation results, and
hashes rather than a false Git claim.

- [ ] **Step 7: Report completion and integration choice**

Report:

```text
changed personal Skill files
6 new static predicate results and total validator count
baseline and candidate sample counts and verdicts
quick_validate result
final file hashes
Browser Pro was not invoked for validation
remaining runtime risks: visible UI changes, allowance visibility, Tool byte
observability, and Tunnel runtime health
feature branch commit IDs
main integration options without touching unrelated user changes
```
