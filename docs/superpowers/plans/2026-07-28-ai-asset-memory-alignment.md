# AI Asset Memory Alignment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `8357e1e`と`fc76b04`の責務分離を、現行ADR lifecycleとOwner文書へ適合させてforward-portする。

**Architecture:** 最新`main`から`codex/ai-asset-memory-alignment`を作り、判断理由は新しいreview ADR、現在のSchema／lifecycle境界は既存Owner文書へ置く。旧ADR本文はコピーせず、現行Header、Decision Log、authority分離に合わせて再構成する。

**Tech Stack:** Git、Markdown、PowerShell、ripgrep

## Global Constraints

- `docs/superpowers/specs/2026-07-28-unique-commit-migration-design.md`を要求仕様とする。
- `8357e1e`と`fc76b04`をcherry-pick、merge、rebaseしない。
- ADRは理由と選択だけを所有し、Schema、fixed value、runtime behaviorはOwner文書へ置く。
- Source format採用集合、MCD／Operation、Artifact／Receiptのcurrent存在を主張しない。
- Owner文書は`文書状態: review`、`実装状態: absent`を維持する。

---

### Task 1: Add the AI Asset／Memory／Async review decision

**Files:**
- Create: `docs/architecture/decisions/2026-07-28-ai-asset-memory-async-alignment.md`
- Modify: `docs/architecture/decisions/README.md:20`
- Test: `docs/architecture/decisions/README.md`

**Interfaces:**
- Consumes: Architecture Governance ADR lifecycle、Memory／Pointers、Asset Lifecycle、Runtime Package、Scheduling／Lifetime
- Produces: `mirakan.decision.ai-asset-memory-async-alignment` review Decision

- [ ] **Step 1: Verify the Decision ID and path are absent**

Run:

```powershell
if (rg -n -F 'mirakan.decision.ai-asset-memory-async-alignment' docs/architecture) { throw 'Decision ID already exists' }
if (Test-Path 'docs/architecture/decisions/2026-07-28-ai-asset-memory-async-alignment.md') { throw 'Decision path already exists' }
```

Expected: exit 0.

- [ ] **Step 2: Create the ADR with the current template**

Use this exact header:

```markdown
# Miraikanai Engine AI-readable Asset／Memory／Async Loading Alignment Decision

- 文書ID: mirakan.decision.ai-asset-memory-async-alignment
- 状態: review
- 正本範囲: AI可読性、外部Asset import、Memory／lifecycle、非同期loadを一つの制作・実行経路として扱う採用判断とOwner責務地図
- 非正本範囲: Product Capability成熟度、Phase／Work Package、Schema・Operation・Diagnostic、Toolchain、Runtime budget／phase、実装Task、Capability activation。各Owner文書を参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)
- 外部根拠検証日: 2026-07-27
- 文書種別: Architecture Decision／cross-owner alignment
- Decision owner document: `mirakan.arch.asset-lifecycle`
- Decision日: 2026-07-28
- Supersedes: none
```

Use the Decision Log template sections in this order:

```markdown
## 1. Context
## 2. Decision drivers
## 3. Considered options
## 4. Decision
## 5. Consequences
## 6. Canonical Owner documents
## 7. Supersedes／Superseded by
## 8. Official or primary sources
```

The Decision must select this path:

```text
Source Asset
  -> typed analysis／Import Profile／Import IR
  -> deterministic Cook／immutable Artifact
  -> Catalog／Runtime Package staging
  -> dependency・integrity・capacity validation
  -> Simulation boundaryでatomic publication
  -> generation handle／lease
  -> retire完了とArtifact reachabilityに基づく回収
```

It must state `contract_activation_effect = none`, current named Source formats `[]`, and current AI authoring Operation set `[]`.

- [ ] **Step 3: Register the Decision once**

Add this exact Decision Log row:

```markdown
| 2026-07-28 | [AI-readable Asset／Memory／Async Loading Alignment](2026-07-28-ai-asset-memory-async-alignment.md) | `review` | Asset import, memory lifecycle, async publication |
```

- [ ] **Step 4: Validate Decision structure**

Run:

```powershell
$adr = 'docs/architecture/decisions/2026-07-28-ai-asset-memory-async-alignment.md'
$required = @('Decision owner document: `mirakan.arch.asset-lifecycle`','## 3. Considered options','contract_activation_effect = none','current named Source format集合はexact `[]`','current AI authoring Operation集合はexact `[]`')
foreach ($term in $required) {
  if (-not (Select-String -LiteralPath $adr -SimpleMatch $term)) { throw "Missing: $term" }
}
$count = (rg -l -F 'mirakan.decision.ai-asset-memory-async-alignment' docs/architecture | Measure-Object).Count
if ($count -ne 1) { throw "Decision ID file count is $count" }
git diff --check
```

Expected: exit 0.

- [ ] **Step 5: Commit the Decision and index**

Run:

```powershell
git add -- docs/architecture/decisions/2026-07-28-ai-asset-memory-async-alignment.md docs/architecture/decisions/README.md
git commit -m "docs: record AI asset memory alignment"
```

Expected: two-file commit.

### Task 2: Add minimal Owner connections

**Files:**
- Modify: `docs/architecture/02-foundation/memory-pointers.md:224`
- Modify: `docs/architecture/03-authoring/asset-lifecycle.md:446`
- Modify: `docs/architecture/04-runtime/runtime-package.md:263`
- Modify: `docs/architecture/README.md:144`

**Interfaces:**
- Consumes: Task 1 Decision ID and path
- Produces: Owner-local lifecycle boundaries with no duplicate Schema authority

- [ ] **Step 1: Add owner-local constraints**

Add one focused paragraph to each owner:

- Memory／Pointers: process memory reclamation does not delete persistent Artifacts; generation handle／lease retirement and Artifact reachability are separate conditions.
- Asset Lifecycle: Source format detection does not imply Product adoption; immutable Derived Artifact publication requires exact Source／Toolchain／Target closure.
- Runtime Package: async I/O completion stays in staging until dependency, integrity, capacity, generation, and publication-boundary checks all pass.
- Architecture README Decision Log section: add navigation link to the new ADR without treating it as an Owner document.

- [ ] **Step 2: Verify the owner boundary terms**

Run:

```powershell
$checks = @{
  'docs/architecture/02-foundation/memory-pointers.md' = 'persistent Artifact'
  'docs/architecture/03-authoring/asset-lifecycle.md' = 'Product採用'
  'docs/architecture/04-runtime/runtime-package.md' = 'staging'
  'docs/architecture/README.md' = 'AI-readable Asset／Memory／Async Loading Alignment'
}
foreach ($entry in $checks.GetEnumerator()) {
  if (-not (Select-String -LiteralPath $entry.Key -SimpleMatch $entry.Value)) { throw "Missing $($entry.Value) in $($entry.Key)" }
}
git diff --check
```

Expected: exit 0.

- [ ] **Step 3: Commit the Owner connections**

Run:

```powershell
git add -- docs/architecture/02-foundation/memory-pointers.md docs/architecture/03-authoring/asset-lifecycle.md docs/architecture/04-runtime/runtime-package.md docs/architecture/README.md
git commit -m "docs: connect asset memory lifecycle owners"
```

Expected: four-file commit.

### Task 3: Review the isolated AI Asset ChangeSet

**Files:**
- Review: all six files changed by Tasks 1 and 2

**Interfaces:**
- Consumes: Tasks 1-2 commits
- Produces: PR-ready ADR and Owner connection diff

- [ ] **Step 1: Verify exact scope and absence of activation**

Run:

```powershell
$allowed = @(
  'docs/architecture/02-foundation/memory-pointers.md',
  'docs/architecture/03-authoring/asset-lifecycle.md',
  'docs/architecture/04-runtime/runtime-package.md',
  'docs/architecture/README.md',
  'docs/architecture/decisions/2026-07-28-ai-asset-memory-async-alignment.md',
  'docs/architecture/decisions/README.md'
)
$changed = @(git diff --name-only main...HEAD)
$unexpected = @($changed | Where-Object { $_ -notin $allowed })
if ($unexpected.Count) { throw "Unexpected files: $($unexpected -join ', ')" }
git diff --check main...HEAD
```

Expected: exit 0.
