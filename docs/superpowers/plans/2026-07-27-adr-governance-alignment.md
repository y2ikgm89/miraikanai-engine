# ADR Governance Alignment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Separate current Architecture authority from an append-only Architecture Decision Log while preserving the accepted restructure Decision and normalizing the proposed Runtime ECS Decision.

**Architecture:** `docs/architecture/decisions/` remains the central Decision Log with one significant decision per file and a shared `README.md` template/index. Current Schema, fixed values, Gates, and runtime behavior remain in Domain Owner documents; Decision links from those documents are informative rather than Header authority dependencies.

**Tech Stack:** Markdown, PowerShell 7, ripgrep, Git

## Global Constraints

- Preserve `docs/architecture/decisions/2026-07-21-document-system-restructure.md` byte-for-byte; its current Git blob hash is `998c5fa0e3ce824f1438d3e5ba391a1d00feb51e`.
- Keep one significant decision per Decision file; do not create a combined `DECISIONS.md`.
- Keep current Schema, fixed values, runtime behavior, Gates, and qualification details in their existing Owner documents.
- Treat `review` as Proposed and `normative` as Accepted; add `rejected` and `superseded` only for Decision lifecycle.
- Do not change MCD, Operation, Tool, Capability activation, Owner Registry, runtime implementation, or package formats.
- Add no production or documentation dependency.
- Use `apply_patch` for repository file edits.
- Preserve unrelated user changes and stage only the files listed by the current task.
- Keep the active plan in Git while executing it; remove it in Task 5 after the migration is complete so Git history remains the plan record.

---

## File map

- `docs/architecture/decisions/README.md`: Decision Log purpose, lifecycle, generic template, and complete navigation.
- `docs/architecture/01-governance/architecture-governance.md`: normative governance rules separating Decision rationale from current Architecture authority.
- `docs/architecture/README.md`: separate navigation for current Architecture specifications and the Decision Log.
- `docs/architecture/decisions/2026-07-27-architecture-decision-log-governance.md`: approved Decision for this migration; remains `review` until the migration passes Task 4.
- `docs/architecture/decisions/2026-07-22-runtime-ecs-contract.md`: concise proposed Decision containing context, options, choice, consequences, and Owner links.
- Thirty-eight current Architecture documents listed in Task 2: removal of the restructure Decision from Header authority dependencies only.
- `docs/architecture/04-runtime/performance-capacity.md` and `docs/architecture/04-runtime/scheduling-lifetime.md`: removal of the Runtime ECS Decision from Header authority dependencies.
- `docs/superpowers/plans/2026-07-27-adr-governance-alignment.md`: active implementation plan, removed after completion.

---

### Task 1: Establish the Decision Log contract and navigation

**Files:**

- Create: `docs/architecture/decisions/README.md`
- Modify: `docs/architecture/01-governance/architecture-governance.md`
- Modify: `docs/architecture/README.md`

**Interfaces:**

- Consumes: the approved design in `docs/architecture/decisions/2026-07-27-architecture-decision-log-governance.md`
- Produces: a documented Decision lifecycle, a reusable Decision format, and separate current-specification/Decision-Log navigation

- [ ] **Step 1: Run the pre-change checks**

Run:

```powershell
Test-Path -LiteralPath 'docs\architecture\decisions\README.md'
rg -n '^## 8\. Architecture Decision Log' 'docs/architecture/01-governance/architecture-governance.md'
rg -n '^### 3\.10 Decisions$' 'docs/architecture/README.md'
```

Expected:

- `Test-Path` prints `False`.
- The Governance search has no match.
- The root Index search matches line 117.

- [ ] **Step 2: Create the generic Decision Log README**

Create `docs/architecture/decisions/README.md` with these sections and semantics:

```markdown
# Miraikanai Engine Architecture Decision Log

## 1. Purpose

- This directory records why architecturally significant choices were made.
- Current Schema, fixed values, runtime behavior, Gates, and qualification remain in Owner documents.
- The log is append-only after a Decision becomes `normative` or `rejected`.

## 2. Lifecycle

| 状態 | Meaning | Allowed changes |
|---|---|---|
| `review` | Proposed | Review revisions |
| `normative` | Accepted | Relationship/status metadata only |
| `rejected` | Rejected | Relationship/status metadata only |
| `superseded` | Replaced by a newer Decision | Relationship metadata only |

## 3. Decision Log

| Date | Decision | Status | Scope |
|---|---|---|---|
| 2026-07-21 | Architecture Document System Restructure | `normative` | document-system restructure |
| 2026-07-22 | Runtime ECS Contract | `review` | Engine-owned ECS choice |
| 2026-07-27 | Architecture Decision Log Governance | `review` | ADR lifecycle and authority separation |

## 4. Template

1. Required Architecture header fields.
2. Decision owner document, Decision date, Supersedes.
3. Context.
4. Decision drivers.
5. Considered options.
6. Decision.
7. Consequences.
8. Canonical Owner documents.
9. Supersedes/Superseded by.
10. Official or primary sources.

## 5. Update rules

- One significant decision per file.
- Do not rewrite accepted/rejected Decision bodies.
- Create a new Decision for a changed choice and link the supersession.
- Do not use Decision text as the sole authority for a current Contract.
```

Use relative links for all three Decision names. State that this README is navigation/template only and does not itself make an Architecture decision.

- [ ] **Step 3: Add Decision Log governance to Architecture Governance**

Append `## 8. Architecture Decision Log` without renumbering existing sections or changing existing anchors. Add these exact rules:

1. Current Architecture specifications continue to use `review | normative`.
2. Decision records use `review | normative | rejected | superseded`, where the first two map to Proposed and Accepted.
3. `normative` and `rejected` Decision bodies are immutable; only status and supersession relationship metadata may change.
4. Changed choices require a new Decision and bidirectional stable-ID/relative-link supersession.
5. Decision rationale is informative to Domain Owner documents and is not the sole authority for current Contract fields, fixed values, Gates, or runtime behavior.
6. Rejected and superseded records remain discoverable in the Decision Log even when they are not current Architecture authority.
7. `decisions/README.md` owns only lifecycle, template, and navigation.

Extend the Governance header `正本範囲` with `Architecture Decision Logの状態・不変性・現行正本との分離`.

- [ ] **Step 4: Separate Decision navigation in the root Architecture Index**

In `docs/architecture/README.md`:

1. Remove `### 3.10 Decisions` and rows 51–52 from `## 3. 正本一覧`.
2. Insert `## 4. Decision Log` with a link to `decisions/README.md` and a short statement that Decisions own rationale/history, not current Schema or runtime behavior.
3. Renumber the former `## 4. 変更時の入口` to `## 5. 変更時の入口`.
4. Renumber the former `## 5. Indexの更新規則` to `## 6. Indexの更新規則`.
5. Add a `DecisionのContext、比較案、置換履歴` row to the navigation table, pointing first to `decisions/README.md` and then to the relevant Domain Owner.

- [ ] **Step 5: Verify Task 1**

Run:

```powershell
Test-Path -LiteralPath 'docs\architecture\decisions\README.md'
rg -n '^## 8\. Architecture Decision Log' 'docs/architecture/01-governance/architecture-governance.md'
rg -n '^## 4\. Decision Log$|^## 5\. 変更時の入口$|^## 6\. Indexの更新規則$' 'docs/architecture/README.md'
rg -n '^### 3\.10 Decisions$' 'docs/architecture/README.md'
git diff --check
```

Expected:

- The new README exists.
- Governance has exactly one Decision Log section.
- All three root Index headings match.
- `### 3.10 Decisions` has no match.
- `git diff --check` exits successfully.

- [ ] **Step 6: Commit Task 1**

```powershell
git add -- 'docs/architecture/decisions/README.md' 'docs/architecture/01-governance/architecture-governance.md' 'docs/architecture/README.md'
git diff --cached --check
git commit -m "docs: establish architecture decision log"
```

---

### Task 2: Remove historical Decision links from current authority headers

**Files:**

- Modify: `docs/architecture/00-product/product-plan.md`
- Modify: `docs/architecture/01-governance/architecture-governance.md`
- Modify: `docs/architecture/02-foundation/core-architecture.md`
- Modify: `docs/architecture/02-foundation/cpp23-modules.md`
- Modify: `docs/architecture/02-foundation/executable-contracts.md`
- Modify: `docs/architecture/02-foundation/math-core.md`
- Modify: `docs/architecture/02-foundation/memory-pointers.md`
- Modify: `docs/architecture/02-foundation/naming-project-layout.md`
- Modify: `docs/architecture/02-foundation/toolchain-dependencies.md`
- Modify: `docs/architecture/03-authoring/asset-lifecycle.md`
- Modify: `docs/architecture/03-authoring/editor-ui-framework.md`
- Modify: `docs/architecture/03-authoring/editor-workspace-ux.md`
- Modify: `docs/architecture/03-authoring/gameplay-programming-model.md`
- Modify: `docs/architecture/03-authoring/native-game-module.md`
- Modify: `docs/architecture/03-authoring/project-state.md`
- Modify: `docs/architecture/04-runtime/debugging-observability-replay.md`
- Modify: `docs/architecture/04-runtime/performance-capacity.md`
- Modify: `docs/architecture/04-runtime/scheduling-lifetime.md`
- Modify: `docs/architecture/05-simulation/animation.md`
- Modify: `docs/architecture/05-simulation/collision.md`
- Modify: `docs/architecture/05-simulation/navigation.md`
- Modify: `docs/architecture/05-simulation/physics.md`
- Modify: `docs/architecture/06-rendering/camera.md`
- Modify: `docs/architecture/06-rendering/environment-surfaces.md`
- Modify: `docs/architecture/06-rendering/lighting.md`
- Modify: `docs/architecture/06-rendering/lod.md`
- Modify: `docs/architecture/06-rendering/materials.md`
- Modify: `docs/architecture/06-rendering/post-processing.md`
- Modify: `docs/architecture/06-rendering/render-graph.md`
- Modify: `docs/architecture/06-rendering/vfx-authoring.md`
- Modify: `docs/architecture/06-rendering/vfx-runtime.md`
- Modify: `docs/architecture/07-platform/android.md`
- Modify: `docs/architecture/07-platform/apple.md`
- Modify: `docs/architecture/07-platform/audio.md`
- Modify: `docs/architecture/07-platform/input.md`
- Modify: `docs/architecture/07-platform/mobile-common.md`
- Modify: `docs/architecture/07-platform/ui-text-localization-accessibility.md`
- Modify: `docs/architecture/07-platform/windows.md`

**Interfaces:**

- Consumes: the informative/authority distinction defined in Task 1
- Produces: current Architecture headers that no longer classify the accepted restructure Decision as a Contract authority dependency

- [ ] **Step 1: Record the immutable accepted Decision hash and failing reference count**

Run:

```powershell
git hash-object 'docs/architecture/decisions/2026-07-21-document-system-restructure.md'
$needle = '2026-07-21-document-system-restructure.md'
$files = rg -l --glob '*.md' ([regex]::Escape($needle)) docs/architecture |
  Where-Object {
    $_ -notmatch 'docs[/\\]architecture[/\\]decisions' -and
    $_ -notmatch 'README\.md$'
  }
$files.Count
```

Expected:

- Blob hash: `998c5fa0e3ce824f1438d3e5ba391a1d00feb51e`.
- Current specification reference count: `38`.

- [ ] **Step 2: Remove only the historical Decision prefix from each Header dependency**

For every file listed in this task, change line 7 from:

```markdown
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Next dependency]...
```

to:

```markdown
- 依存: [Next dependency]...
```

Do not change any remaining dependency order, label, target, body content, or Domain contract. Do not modify the accepted Decision file.

- [ ] **Step 3: Verify that no current specification Header depends on the accepted Decision**

Run:

```powershell
$needle = '2026-07-21-document-system-restructure.md'
$files = rg -l --glob '*.md' ([regex]::Escape($needle)) docs/architecture |
  Where-Object {
    $_ -notmatch 'docs[/\\]architecture[/\\]decisions' -and
    $_ -notmatch 'README\.md$'
  }
$files.Count
git hash-object 'docs/architecture/decisions/2026-07-21-document-system-restructure.md'
git diff --check
```

Expected:

- Current specification reference count: `0`.
- Accepted Decision hash remains `998c5fa0e3ce824f1438d3e5ba391a1d00feb51e`.
- `git diff --check` exits successfully.

- [ ] **Step 4: Review the mechanical diff**

Run:

```powershell
git diff --numstat
git diff --word-diff=porcelain -- docs/architecture |
  Select-String -Pattern 'document-system-restructure'
```

Expected:

- Exactly 38 files have one removed dependency link each.
- No body paragraph, Schema, fixed value, Gate, or accepted Decision byte changes appear.

- [ ] **Step 5: Commit Task 2**

Stage exactly the 38 files listed in this task, then run:

```powershell
git diff --cached --check
git commit -m "docs: separate ADR rationale from authority dependencies"
```

---

### Task 3: Normalize the proposed Runtime ECS Decision

**Files:**

- Modify: `docs/architecture/decisions/2026-07-22-runtime-ecs-contract.md`
- Modify: `docs/architecture/04-runtime/performance-capacity.md`
- Modify: `docs/architecture/04-runtime/scheduling-lifetime.md`
- Verify only: `docs/architecture/04-runtime/entity-component-system.md`
- Verify only: `docs/architecture/02-foundation/compatibility-evolution.md`
- Verify only: `docs/architecture/00-product/product-plan.md`

**Interfaces:**

- Consumes: Runtime ECS Schema and fixed values from `entity-component-system.md`, qualification from `performance-capacity.md`, migration rules from `compatibility-evolution.md`, and currentization from `architecture-governance.md`
- Produces: a concise Proposed ADR containing only decision context, options, choice, consequences, and Owner links

- [ ] **Step 1: Prove that duplicated implementation details currently exist**

Run:

```powershell
rg -n 'RuntimeComponentLayoutPolicyV1|16 KiB|64-byte|256-byte|90-run|12,600|target不変条件|approved target addendum' `
  'docs/architecture/decisions/2026-07-22-runtime-ecs-contract.md'
```

Expected: matches in the proposed Decision, including the target invariant/addendum sections.

- [ ] **Step 2: Prove that the canonical Owner documents already contain the details**

Run:

```powershell
rg -n 'RuntimeComponentLayoutPolicyV1|16 KiB|64-byte|256-byte' `
  'docs/architecture/04-runtime/entity-component-system.md'
rg -n '90 measured runs|12,600秒|selected 16 KiB layout' `
  'docs/architecture/04-runtime/performance-capacity.md'
rg -n 'source_preserving_recook|Runtime ECS正本化' `
  'docs/architecture/02-foundation/compatibility-evolution.md'
```

Expected: each concept resolves in its Owner document before it is removed from the Decision.

- [ ] **Step 3: Rewrite the Runtime ECS Decision in the generic format**

Keep `状態: review`. Put the required Architecture header fields first, followed by:

```markdown
- 文書種別: Architecture Decision
- Decision owner document: `mirakan.arch.runtime-ecs`
- Decision日: 2026-07-22
- Supersedes: none
```

Use these sections:

1. `Context`: explain the need for a standard Runtime World Entity/Component storage and the separation from Package, Save, Scheduling, Compatibility, and AI authority.
2. `Decision drivers`: data-only Components, deterministic query/structural boundaries, Save/Replay projection, bounded AI explanation, measurable layout behavior.
3. `Considered options`: Engine-owned archetype/SoA; third-party ECS runtime; object-per-Entity virtual update; direct structural mutation during iteration; one combined ECS/Package/Save authority.
4. `Decision`: select Engine-owned archetype/SoA and explicitly reject vendor runtime/API/schema compatibility.
5. `Consequences`: locality and control benefits; implementation/qualification cost; responsibility for migration and evidence.
6. `Canonical Owner documents`: retain the existing seven-owner table, but link rather than repeat fields or values.
7. `Currentization and compatibility`: summarize that Governance and Compatibility own the approval closure and `source_preserving_recook`; do not duplicate the seven-step closure.
8. `Official comparison sources`: retain focused Unity Entities, Flecs, EnTT, and Unreal official/primary links relevant to the chosen option.
9. `Non-goals and relationships`: no activation, implementation task, MCD/Operation registration, or supersession.

Delete the duplicated target invariant list, exact layout values, qualification run counts, exact campaign duration, detailed migration checklist, route enum, and implementation-order list. The resulting ADR must stand alone as a decision but defer all current technical details to Owner documents.

- [ ] **Step 4: Remove Runtime ECS Decision from two authority headers**

In both:

- `docs/architecture/04-runtime/performance-capacity.md`
- `docs/architecture/04-runtime/scheduling-lifetime.md`

remove only:

```markdown
[Runtime ECS契約Decision](../decisions/2026-07-22-runtime-ecs-contract.md)、
```

Keep the Runtime ECS Owner dependency and all remaining dependencies unchanged. Preserve the Product Plan body link at `product-plan.md:127` as an informative rationale link.

- [ ] **Step 5: Verify Task 3**

Run:

```powershell
rg -n 'RuntimeComponentLayoutPolicyV1|16 KiB|64-byte|256-byte|90-run|12,600|target不変条件|approved target addendum' `
  'docs/architecture/decisions/2026-07-22-runtime-ecs-contract.md'
rg -n '^-\s*依存:.*2026-07-22-runtime-ecs-contract\.md' `
  'docs/architecture/04-runtime/performance-capacity.md' `
  'docs/architecture/04-runtime/scheduling-lifetime.md'
rg -n '2026-07-22-runtime-ecs-contract\.md' 'docs/architecture/00-product/product-plan.md'
git hash-object 'docs/architecture/decisions/2026-07-21-document-system-restructure.md'
git diff --check
```

Expected:

- No duplicate implementation-detail match remains in the Runtime ECS Decision.
- Neither Runtime header contains the Decision dependency.
- Product Plan retains one informative link.
- Accepted Decision hash remains `998c5fa0e3ce824f1438d3e5ba391a1d00feb51e`.
- `git diff --check` exits successfully.

- [ ] **Step 6: Commit Task 3**

```powershell
git add -- 'docs/architecture/decisions/2026-07-22-runtime-ecs-contract.md' `
  'docs/architecture/04-runtime/performance-capacity.md' `
  'docs/architecture/04-runtime/scheduling-lifetime.md'
git diff --cached --check
git commit -m "docs: normalize runtime ECS decision"
```

---

### Task 4: Validate the migration and accept ADR governance

**Files:**

- Modify: `docs/architecture/decisions/2026-07-27-architecture-decision-log-governance.md`
- Modify: `docs/architecture/decisions/README.md`
- Verify: all `docs/architecture/**/*.md`

**Interfaces:**

- Consumes: completed Tasks 1–3
- Produces: an Accepted ADR governance Decision with consistent Decision Log status and verified links/IDs

- [ ] **Step 1: Validate unique document IDs and required headers**

Run:

```powershell
$files = Get-ChildItem -LiteralPath 'docs\architecture' -Recurse -File -Filter '*.md' |
  Where-Object { $_.Name -ne 'README.md' }
$rows = foreach ($file in $files) {
  $lines = Get-Content -LiteralPath $file.FullName
  $id = $lines | Where-Object { $_ -match '^- 文書ID:' } | Select-Object -First 1
  $status = $lines | Where-Object { $_ -match '^- 状態:' } | Select-Object -First 1
  $scope = $lines | Where-Object { $_ -match '^- 正本範囲:' } | Select-Object -First 1
  $nonScope = $lines | Where-Object { $_ -match '^- 非正本範囲:' } | Select-Object -First 1
  $deps = $lines | Where-Object { $_ -match '^- 依存:' } | Select-Object -First 1
  $verified = $lines | Where-Object { $_ -match '^- 外部根拠検証日:' } | Select-Object -First 1
  if (!$id -or !$status -or !$scope -or !$nonScope -or !$deps -or !$verified) {
    throw "Missing required header field: $($file.FullName)"
  }
  [pscustomobject]@{ File = $file.FullName; Id = ($id -replace '^- 文書ID:\s*', '') }
}
$duplicates = $rows | Group-Object Id | Where-Object Count -gt 1
if ($duplicates) { $duplicates | Format-Table; throw 'Duplicate document ID' }
```

Expected: no exception.

- [ ] **Step 2: Validate local Markdown file and heading links**

Run this PowerShell validator:

```powershell
$files = Get-ChildItem -LiteralPath 'docs\architecture' -Recurse -File -Filter '*.md'
$errors = [System.Collections.Generic.List[string]]::new()
foreach ($file in $files) {
  $text = Get-Content -Raw -LiteralPath $file.FullName
  foreach ($match in [regex]::Matches($text, '\[[^\]]+\]\((?!https?://)([^)]+)\)')) {
    $raw = $match.Groups[1].Value
    $parts = $raw -split '#', 2
    $relativePath = $parts[0]
    $anchor = if ($parts.Count -eq 2) { $parts[1] } else { '' }
    $target = if ($relativePath) {
      [System.IO.Path]::GetFullPath((Join-Path $file.DirectoryName $relativePath))
    } else {
      $file.FullName
    }
    if (!(Test-Path -LiteralPath $target)) {
      $errors.Add("$($file.FullName): missing $raw")
      continue
    }
    if ($anchor) {
      $targetLines = Get-Content -LiteralPath $target
      $anchors = foreach ($line in $targetLines) {
        if ($line -match '^#{1,6}\s+(.+)$') {
          (($Matches[1].ToLowerInvariant().Trim() -replace '[^\p{L}\p{N}\p{M}\s-]', '') -replace '\s', '-')
        }
      }
      if ($anchor -notin $anchors) {
        $errors.Add("$($file.FullName): missing anchor $raw")
      }
    }
  }
}
if ($errors.Count) { $errors; throw 'Markdown link validation failed' }
```

Expected: no exception.

- [ ] **Step 3: Validate Decision Log closure and authority separation**

Run:

```powershell
$decisionFiles = Get-ChildItem -LiteralPath 'docs\architecture\decisions' -File -Filter '*.md' |
  Where-Object { $_.Name -ne 'README.md' } |
  Select-Object -ExpandProperty Name
$index = Get-Content -Raw -LiteralPath 'docs\architecture\decisions\README.md'
foreach ($name in $decisionFiles) {
  if ($index -notmatch [regex]::Escape($name)) { throw "Decision missing from index: $name" }
}
rg -n '^-\s*依存:.*decisions/' docs/architecture --glob '!docs/architecture/decisions/**'
git hash-object 'docs/architecture/decisions/2026-07-21-document-system-restructure.md'
git diff --check
```

Expected:

- Every Decision file is listed.
- No current Architecture Header depends on a Decision.
- Accepted restructure Decision hash remains `998c5fa0e3ce824f1438d3e5ba391a1d00feb51e`.
- `git diff --check` exits successfully.

- [ ] **Step 4: Accept the ADR governance Decision**

Only after Steps 1–3 pass:

1. Change `- 状態: review` to `- 状態: normative` in `2026-07-27-architecture-decision-log-governance.md`.
2. Change its Decision Log row from `review` to `normative` in `decisions/README.md`.
3. Do not change any other body text in the governance Decision.

- [ ] **Step 5: Re-run the complete validation**

Repeat Steps 1–3, then run:

```powershell
rg -n '^- 状態: normative$' `
  'docs/architecture/decisions/2026-07-27-architecture-decision-log-governance.md'
rg -n 'architecture-decision-log-governance\.md.*`normative`' `
  'docs/architecture/decisions/README.md'
git status --short
```

Expected:

- Both status checks match.
- Only the governance Decision and Decision Log README are modified.

- [ ] **Step 6: Commit Task 4**

```powershell
git add -- 'docs/architecture/decisions/2026-07-27-architecture-decision-log-governance.md' `
  'docs/architecture/decisions/README.md'
git diff --cached --check
git commit -m "docs: accept ADR governance"
```

---

### Task 5: Retire the completed implementation plan

**Files:**

- Delete: `docs/superpowers/plans/2026-07-27-adr-governance-alignment.md`

**Interfaces:**

- Consumes: all accepted commits and successful Task 4 validation
- Produces: no active completed plan in the working tree; the plan remains recoverable from Git history

- [ ] **Step 1: Confirm completion before deleting the plan**

Run:

```powershell
git status --short
git log -4 --oneline
```

Expected:

- Worktree is clean.
- Commits for Tasks 1–4 are present.

- [ ] **Step 2: Delete only the completed plan**

Use `apply_patch` to delete:

```text
docs/superpowers/plans/2026-07-27-adr-governance-alignment.md
```

Do not delete `docs/superpowers/` recursively. Git will naturally omit empty directories.

- [ ] **Step 3: Commit plan retirement**

```powershell
git add -- 'docs/superpowers/plans/2026-07-27-adr-governance-alignment.md'
git diff --cached --check
git commit -m "docs: retire completed ADR migration plan"
git status --short
```

Expected: commit succeeds and the worktree is clean.
