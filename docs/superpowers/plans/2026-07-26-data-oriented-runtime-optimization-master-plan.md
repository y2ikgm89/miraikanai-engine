# Data-Oriented Runtime Optimization Master Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Execute the approved Foundation、Performance、Runtime ECS、Compatibility, and Product architecture updates in one explicit order with one authorized baseline and one final audit.

**Architecture:** This master plan owns orchestration only; it defines no C++、Memory、ECS、Performance、Compatibility, or Product schema. Three subordinate implementation plans remain the detailed task owners, and their outputs converge through an exact 15-commit sequence before the final cross-document Gate.

**Tech Stack:** Markdown architecture contracts, PowerShell 7, ripgrep, Git, superpowers execution skills.

## Global Constraints

- Normative design:
  `docs/superpowers/specs/2026-07-26-data-oriented-runtime-optimization-design.md`.
- Detailed plans:
  - `docs/superpowers/plans/2026-07-26-foundation-value-memory-contract-plan.md`
  - `docs/superpowers/plans/2026-07-26-runtime-ecs-data-oriented-contract-plan.md`
  - `docs/superpowers/plans/2026-07-26-product-qualification-clean-break-plan.md`
- This file owns execution order and cross-plan Gates only. If it repeats an
  owner schema, Field list, metric list, threshold, Diagnostic argument schema,
  or Product row, delete the duplicate and retain the subordinate owner.
- Do not edit Architecture until the user explicitly selects
  `subagent_driven` or `inline` execution and approves an exact baseline.
- The current workspace contains pre-existing tracked and untracked
  Architecture changes. Never infer authorization to stage or commit them.
- Every Architecture target must be tracked and clean at execution start.
- Use `superpowers:using-git-worktrees` to create the implementation worktree
  from the approved baseline.
- The exact execution order is Foundation all tasks → Product Tasks 0–1 →
  Runtime all tasks → Product Tasks 2–4.
- A failed Gate stops the sequence. Do not skip forward, normalize missing
  evidence, or activate a Product／Runtime path.
- Treat Markdown checkboxes as read-only execution criteria. Track progress in
  the session plan; do not edit or commit planning inputs, which would add
  commits outside the exact 15-commit Architecture sequence.
- The final result changes target Architecture only. It does not activate a
  Capability, apply a Product Definition Migration, publish a Runtime Package,
  or claim Shipping qualification.

## File Map

| File | Responsibility |
|---|---|
| `docs/superpowers/specs/2026-07-26-data-oriented-runtime-optimization-design.md` | Approved normative design and 22 completion conditions |
| `docs/superpowers/plans/2026-07-26-foundation-value-memory-contract-plan.md` | C++ transfer、Memory／container、C++23, Native C ABI |
| `docs/superpowers/plans/2026-07-26-product-qualification-clean-break-plan.md` | Baseline owner repair、Performance、Compatibility、Product、final audit |
| `docs/superpowers/plans/2026-07-26-runtime-ecs-data-oriented-contract-plan.md` | ECS layout、Gameplay、Scheduling、Package、Debug、Decision |

## Execution State Machine

```text
baseline_unapproved
  -> baseline_approved
  -> foundation_complete
  -> performance_seed_complete
  -> runtime_complete
  -> product_projection_complete
  -> coordinated_audit_passed
```

No transition is inferred from prose, elapsed time, a partial commit list, or a
green check from another state.

---

### Task 0: Authorize and freeze the implementation baseline

**Files:**
- Verify: entire Git worktree
- Modify: none

**Interfaces:**
- Consumes: explicit user execution-mode choice and exact baseline authority.
- Produces: clean implementation worktree and immutable
  `baseline_commit_sha256`; no Architecture edit.

- [ ] **Step 1: Record the current blocking state**

```powershell
git branch --show-current
git rev-parse HEAD
git status --short
```

Expected before authorization: the current main worktree contains pre-existing
Architecture modifications and the five untracked canonical documents
`architecture-governance.md`, `compatibility-evolution.md`,
`entity-component-system.md`, `persistence-save.md`, and
`runtime-package.md`.

- [ ] **Step 2: Require an explicit execution authorization**

Accept only a user instruction containing both:

```text
execution_mode = subagent_driven | inline
baseline_policy = approve_current_architecture_snapshot | use_named_commit
```

For `use_named_commit`, also require one full 40-hex Git commit ID. Do not
choose the mode, baseline policy, or commit on the user's behalf.

- [ ] **Step 3: Establish the approved baseline outside this plan**

For `approve_current_architecture_snapshot`, stop and let the user create or
explicitly authorize a dedicated baseline commit containing the intended
pre-existing Architecture work. This master plan never creates that commit by
assumption.

For `use_named_commit`, verify the supplied commit exists:

```powershell
if ($approvedBaselineCommit -notmatch '^[0-9a-f]{40}$') {
  throw 'Approved baseline must be one full lowercase 40-hex commit ID.'
}
git cat-file -e "$approvedBaselineCommit^{commit}"
if ($LASTEXITCODE -ne 0) { throw 'Approved baseline commit does not exist.' }
```

- [ ] **Step 4: Create and validate the isolated worktree**

Use `superpowers:using-git-worktrees`, then run inside that worktree:

```powershell
$targets = @(
  'docs/architecture/00-product/product-plan.md',
  'docs/architecture/01-governance/ai-verification-provenance.md',
  'docs/architecture/02-foundation/core-architecture.md',
  'docs/architecture/02-foundation/executable-contracts.md',
  'docs/architecture/02-foundation/memory-pointers.md',
  'docs/architecture/02-foundation/compatibility-evolution.md',
  'docs/architecture/02-foundation/cpp23-modules.md',
  'docs/architecture/03-authoring/gameplay-programming-model.md',
  'docs/architecture/03-authoring/native-game-module.md',
  'docs/architecture/04-runtime/entity-component-system.md',
  'docs/architecture/04-runtime/performance-capacity.md',
  'docs/architecture/04-runtime/scheduling-lifetime.md',
  'docs/architecture/04-runtime/runtime-package.md',
  'docs/architecture/04-runtime/debugging-observability-replay.md',
  'docs/architecture/decisions/2026-07-22-runtime-ecs-contract.md'
)
foreach ($path in $targets) {
  git ls-files --error-unmatch -- $path *> $null
  if ($LASTEXITCODE -ne 0) { throw "Target is not tracked: $path" }
  if (git status --porcelain -- $path) {
    throw "Target is not clean: $path"
  }
}
if (git status --porcelain) { throw 'Implementation worktree is not clean.' }
$baselineCommit = git rev-parse HEAD
if ($baselineCommit.Length -ne 40) { throw 'Baseline is not a full commit ID.' }
```

Expected: all 15 target files are tracked, worktree and index are clean, and
`baselineCommit` identifies the user-approved state.

---

### Task 1: Verify the five planning inputs before execution

**Files:**
- Verify: the design and three detailed plans in the File Map
- Modify: none

**Interfaces:**
- Consumes: approved baseline and committed planning inputs.
- Produces: exact plan dependency closure and zero placeholder findings.

- [ ] **Step 1: Verify every planning input is tracked and clean**

```powershell
$planningInputs = @(
  'docs/superpowers/specs/2026-07-26-data-oriented-runtime-optimization-design.md',
  'docs/superpowers/plans/2026-07-26-data-oriented-runtime-optimization-master-plan.md',
  'docs/superpowers/plans/2026-07-26-foundation-value-memory-contract-plan.md',
  'docs/superpowers/plans/2026-07-26-runtime-ecs-data-oriented-contract-plan.md',
  'docs/superpowers/plans/2026-07-26-product-qualification-clean-break-plan.md'
)
foreach ($path in $planningInputs) {
  git ls-files --error-unmatch -- $path *> $null
  if ($LASTEXITCODE -ne 0) { throw "Planning input is not tracked: $path" }
  if (git status --porcelain -- $path) {
    throw "Planning input is not clean: $path"
  }
}
```

- [ ] **Step 2: Verify cross-plan dependencies are bidirectional**

```powershell
$master = 'docs/superpowers/plans/2026-07-26-data-oriented-runtime-optimization-master-plan.md'
$subplans = @(
  'docs/superpowers/plans/2026-07-26-foundation-value-memory-contract-plan.md',
  'docs/superpowers/plans/2026-07-26-runtime-ecs-data-oriented-contract-plan.md',
  'docs/superpowers/plans/2026-07-26-product-qualification-clean-break-plan.md'
)
$masterText = Get-Content -Raw $master
foreach ($subplan in $subplans) {
  if (-not $masterText.Contains($subplan)) {
    throw "Master does not reference $subplan"
  }
  $subplanText = Get-Content -Raw $subplan
  if (-not $subplanText.Contains($master)) {
    throw "$subplan does not reference the master"
  }
}
```

Expected: the master references all three plans and every plan points back to
the master.

- [ ] **Step 3: Verify structure and placeholder exclusion**

```powershell
$subplans = @(
  'docs/superpowers/plans/2026-07-26-foundation-value-memory-contract-plan.md',
  'docs/superpowers/plans/2026-07-26-runtime-ecs-data-oriented-contract-plan.md',
  'docs/superpowers/plans/2026-07-26-product-qualification-clean-break-plan.md'
)
foreach ($path in $subplans) {
  $content = Get-Content -Raw $path
  if (-not $content.Contains('## Execution Preflight')) {
    throw "Execution preflight missing: $path"
  }
  if (([regex]::Matches($content, '(?m)^- \[ \]')).Count -eq 0) {
    throw "No executable checkboxes: $path"
  }
}
$hits = rg -n -i `
  'T[B]D|T[O]DO|F[I]XME|implement[ ]later|similar[ ]to|Add[ ]appropriate|Write[ ]tests[ ]for[ ]the[ ]above' `
  $subplans
if ($LASTEXITCODE -eq 0) {
  throw "Planning placeholders found:`n$hits"
}
```

Expected: all detailed plans are executable and contain no placeholder
instruction.

---

### Task 2: Execute the Foundation plan

**Files:**
- Execute:
  `docs/superpowers/plans/2026-07-26-foundation-value-memory-contract-plan.md`

**Interfaces:**
- Consumes: `baseline_approved`.
- Produces: `foundation_complete`, four-Type closure, canonical Memory policy,
  C++ consumers, and fixed C ABI adapter boundary.

- [ ] **Step 1: Execute all five Foundation tasks in order**

Use the selected execution skill and do not start Product or Runtime work until
all 28 Foundation checkboxes pass.

- [ ] **Step 2: Verify the exact Foundation commit sequence**

```powershell
$firstPlanCommit = git log --format='%H' `
  --grep='^docs: route data-oriented foundation contracts$' -1
if (-not $firstPlanCommit) { throw 'First Foundation commit was not found.' }
$baselineCommit = git rev-parse "$firstPlanCommit^"
$expected = @(
  'docs: route data-oriented foundation contracts',
  'docs: add value transfer definition closure',
  'docs: define C++ value transfer and layout contracts',
  'docs: bind C++23 value transfer gates',
  'docs: bind native C ABI adapters to memory policy'
)
$actual = @(
  git log "$baselineCommit..HEAD" --reverse --format='%s'
)
if (
  $actual.Count -ne $expected.Count -or
  @(Compare-Object $expected $actual -SyncWindow 0).Count -ne 0
) {
  throw "Foundation commit sequence mismatch:`n$($actual -join "`n")"
}
```

Expected: exactly five commits in the declared order.

---

### Task 3: Establish Product owner integrity and the Performance seed

**Files:**
- Execute Tasks 0–1:
  `docs/superpowers/plans/2026-07-26-product-qualification-clean-break-plan.md`

**Interfaces:**
- Consumes: `foundation_complete`.
- Produces: zero unresolved Architecture owner refs and canonical Performance
  profile／campaign／metrics required by Runtime consumers.

- [ ] **Step 1: Execute Product Tasks 0 and 1 only**

Stop before Product Task 2. Task 0 must close the existing Release Evidence
owner; Task 1 must establish the Performance-owned qualification records.

- [ ] **Step 2: Verify the two additional commits**

```powershell
$firstPlanCommit = git log --format='%H' `
  --grep='^docs: route data-oriented foundation contracts$' -1
if (-not $firstPlanCommit) { throw 'First Foundation commit was not found.' }
$baselineCommit = git rev-parse "$firstPlanCommit^"
$expected = @(
  'docs: resolve product release evidence owner',
  'docs: define ECS data-oriented qualification'
)
$actual = @(
  git log "$baselineCommit..HEAD" --reverse --format='%s' |
    Select-Object -Skip 5
)
if (
  $actual.Count -ne 2 -or
  @(Compare-Object $expected $actual -SyncWindow 0).Count -ne 0
) {
  throw "Product seed sequence mismatch:`n$($actual -join "`n")"
}
```

Expected: the branch has exactly seven plan commits and can supply exact
Performance refs to Runtime.

---

### Task 4: Execute the Runtime ECS plan

**Files:**
- Execute:
  `docs/superpowers/plans/2026-07-26-runtime-ecs-data-oriented-contract-plan.md`

**Interfaces:**
- Consumes: `foundation_complete` and Performance seed.
- Produces: `runtime_complete` across ECS、Gameplay、Scheduling、Package、Debug,
  and the target Decision.

- [ ] **Step 1: Execute Runtime Tasks 1–7 in order**

Tasks 1–6 each end in a commit. Task 7 is the Runtime audit and creates no
commit.

- [ ] **Step 2: Verify the six Runtime commits**

```powershell
$firstPlanCommit = git log --format='%H' `
  --grep='^docs: route data-oriented foundation contracts$' -1
if (-not $firstPlanCommit) { throw 'First Foundation commit was not found.' }
$baselineCommit = git rev-parse "$firstPlanCommit^"
$expected = @(
  'docs: define canonical runtime ECS layout',
  'docs: derive ECS access cohorts from systems',
  'docs: close ECS scheduling allocation boundaries',
  'docs: bind runtime packages to ECS layout closure',
  'docs: project ECS qualification telemetry',
  'docs: approve data-oriented ECS target'
)
$actual = @(
  git log "$baselineCommit..HEAD" --reverse --format='%s' |
    Select-Object -Skip 7
)
if (
  $actual.Count -ne 6 -or
  @(Compare-Object $expected $actual -SyncWindow 0).Count -ne 0
) {
  throw "Runtime sequence mismatch:`n$($actual -join "`n")"
}
```

Expected: the branch has exactly thirteen plan commits.

---

### Task 5: Complete Compatibility and the Product destination projection

**Files:**
- Execute Tasks 2–3:
  `docs/superpowers/plans/2026-07-26-product-qualification-clean-break-plan.md`

**Interfaces:**
- Consumes: `runtime_complete`.
- Produces: fail-closed clean-break procedure and a non-active atomic
  destination Product projection.

- [ ] **Step 1: Execute Product Tasks 2 and 3**

Do not mutate the current source Product Registry rows. Add only the approved
Compatibility closure and destination projection described by the detailed
plan.

- [ ] **Step 2: Verify the final two implementation commits**

```powershell
$firstPlanCommit = git log --format='%H' `
  --grep='^docs: route data-oriented foundation contracts$' -1
if (-not $firstPlanCommit) { throw 'First Foundation commit was not found.' }
$baselineCommit = git rev-parse "$firstPlanCommit^"
$expected = @(
  'docs: close ECS clean-break inventory',
  'docs: project data-oriented ECS product gate'
)
$actual = @(
  git log "$baselineCommit..HEAD" --reverse --format='%s' |
    Select-Object -Skip 13
)
if (
  $actual.Count -ne 2 -or
  @(Compare-Object $expected $actual -SyncWindow 0).Count -ne 0
) {
  throw "Product completion sequence mismatch:`n$($actual -join "`n")"
}
```

Expected: exactly fifteen implementation commits.

---

### Task 6: Run the coordinated final Gate

**Files:**
- Execute Task 4:
  `docs/superpowers/plans/2026-07-26-product-qualification-clean-break-plan.md`
- Verify: all 15 Architecture targets from Task 0

**Interfaces:**
- Consumes: all subordinate plan results.
- Produces: `coordinated_audit_passed` or one explicit failing invariant.

- [ ] **Step 1: Run every Product Task 4 audit step**

This includes the 52-ID inventory, document refs, relative paths, heading
anchors, schema／hash-domain ownership, four-Type closure, Product
bidirectional refs, 90-run campaign arithmetic, 35 metrics, 15 Diagnostics, C
ABI exclusions, Shipping-path exclusions, and indirect-owner review.

- [ ] **Step 2: Verify the exact 15-commit chain**

```powershell
$firstPlanCommit = git log --format='%H' `
  --grep='^docs: route data-oriented foundation contracts$' -1
if (-not $firstPlanCommit) { throw 'First Foundation commit was not found.' }
$baselineCommit = git rev-parse "$firstPlanCommit^"
$expected = @(
  'docs: route data-oriented foundation contracts',
  'docs: add value transfer definition closure',
  'docs: define C++ value transfer and layout contracts',
  'docs: bind C++23 value transfer gates',
  'docs: bind native C ABI adapters to memory policy',
  'docs: resolve product release evidence owner',
  'docs: define ECS data-oriented qualification',
  'docs: define canonical runtime ECS layout',
  'docs: derive ECS access cohorts from systems',
  'docs: close ECS scheduling allocation boundaries',
  'docs: bind runtime packages to ECS layout closure',
  'docs: project ECS qualification telemetry',
  'docs: approve data-oriented ECS target',
  'docs: close ECS clean-break inventory',
  'docs: project data-oriented ECS product gate'
)
$actual = @(
  git log "$baselineCommit..HEAD" --reverse --format='%s'
)
if (
  $actual.Count -ne 15 -or
  @(Compare-Object $expected $actual -SyncWindow 0).Count -ne 0
) {
  throw "Final commit sequence mismatch:`n$($actual -join "`n")"
}
```

- [ ] **Step 3: Verify clean scope and non-activation**

```powershell
$firstPlanCommit = git log --format='%H' `
  --grep='^docs: route data-oriented foundation contracts$' -1
if (-not $firstPlanCommit) { throw 'First Foundation commit was not found.' }
$baselineCommit = git rev-parse "$firstPlanCommit^"
if (git status --porcelain) { throw 'Implementation worktree is not clean.' }
$diff = git diff "$baselineCommit..HEAD" -- docs/architecture
foreach ($forbidden in @(
  'contract_activation_effect = activated',
  'capability_state = active',
  'runtime_activation = enabled'
)) {
  if ($diff.Contains($forbidden)) {
    throw "Unexpected activation claim: $forbidden"
  }
}
git diff --check "$baselineCommit..HEAD"
```

Expected: clean worktree, no whitespace failures, no activation claim, and no
commit outside the exact sequence.

- [ ] **Step 4: Stop on newly discovered baseline conflicts**

If final validation finds a conflict not enumerated in the detailed plans, do
not create a sixteenth implementation commit. Return to the approved design,
amend the affected plan with the exact owner／schema／test change, obtain review,
and resume from the last passing state.

- [ ] **Step 5: Report the result**

Report the approved baseline commit, execution mode, 15 commit IDs, changed
Architecture files, all verification commands and outcomes, Product
non-activation state, Consumer Inventory status, and any remaining risk.
