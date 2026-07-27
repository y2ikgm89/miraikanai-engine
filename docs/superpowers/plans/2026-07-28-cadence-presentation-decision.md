# Cadence Presentation Decision Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `35ccd17`からSimulation CadenceとPresentationの責務分離だけを、現行`main`へreview ADRとしてforward-portする。

**Architecture:** 最新`main`から`codex/cadence-presentation-decision`を作る。既存`SimulationAdvanceIntervalV1`とplanning-only alternate cadenceを参照し、新しいcurrent Schema、120／240 Hz Capability、Target Qualificationは追加しない。

**Tech Stack:** Git、Markdown、PowerShell、ripgrep

## Global Constraints

- `docs/superpowers/specs/2026-07-28-unique-commit-migration-design.md`を要求仕様とする。
- `35ccd17`をcherry-pick、merge、rebaseしない。
- ADRは責務分離の判断理由を所有し、interval Schemaとruntime fixed valueを再定義しない。
- Product Planへhigh-refresh Capability、claim、Work Packageを追加しない。
- 60／120／240 Hzの実測済み、qualified、activatedという主張を追加しない。

---

### Task 1: Add the Cadence／Presentation review decision

**Files:**
- Create: `docs/architecture/decisions/2026-07-28-simulation-cadence-presentation-separation.md`
- Modify: `docs/architecture/decisions/README.md:20`
- Test: `docs/architecture/00-product/product-plan.md`

**Interfaces:**
- Consumes: Schedulingの`SimulationCadenceProfileV1`／`SimulationAdvanceIntervalV1`、Input T10 latch、Physics substep boundary
- Produces: `mirakan.decision.simulation-cadence-presentation-separation` review Decision

- [ ] **Step 1: Verify the destination Decision is absent**

Run:

```powershell
if (rg -n -F 'mirakan.decision.simulation-cadence-presentation-separation' docs/architecture) { throw 'Decision ID already exists' }
if (Test-Path 'docs/architecture/decisions/2026-07-28-simulation-cadence-presentation-separation.md') { throw 'Decision path already exists' }
```

Expected: exit 0.

- [ ] **Step 2: Create the ADR with current governance metadata**

Use this exact header:

```markdown
# Miraikanai Engine Simulation Cadence／Presentation Separation Decision

- 文書ID: mirakan.decision.simulation-cadence-presentation-separation
- 状態: review
- 正本範囲: Device reading、Presentation、Simulation Advance、Physics substepを別OwnerのProfile／Intervalとして扱う採用判断
- 非正本範囲: Runtime phase／Schema／固定値、Input Schema、Package／Save／Replay Schema、Subsystem budget、Capability activation、実装Task。各Owner文書を参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Input](../07-platform/input.md)、[Physics](../05-simulation/physics.md)
- 外部根拠検証日: 2026-07-27
- 文書種別: Architecture Decision／runtime timing responsibility
- Decision owner document: `mirakan.arch.runtime-scheduling-lifetime`
- Decision日: 2026-07-28
- Supersedes: none
```

Use the Decision Log template section order. The Decision must state:

- Device reading is an input candidate, not a consumed authoritative Command.
- Presentation is non-authoritative and cannot mutate World、Save、Replay、Gameplay digest.
- Simulation uses the sealed current `SimulationAdvanceIntervalV1`.
- Physics substep is a child of one outer Simulation Advance and creates no extra Input、Timer、event、publication boundary.
- High-refresh Presentation and alternate Simulation cadence remain separate future evaluation subjects.
- `contract_activation_effect = none`.

- [ ] **Step 3: Register the Decision once**

Add this exact Decision Log row:

```markdown
| 2026-07-28 | [Simulation Cadence／Presentation Separation](2026-07-28-simulation-cadence-presentation-separation.md) | `review` | Runtime timing authority separation |
```

- [ ] **Step 4: Verify no Product capability was added**

Run:

```powershell
$productDiff = git diff -- docs/architecture/00-product/product-plan.md
if ($productDiff) { throw 'Product Plan changed unexpectedly' }
$adr = 'docs/architecture/decisions/2026-07-28-simulation-cadence-presentation-separation.md'
$required = @('Device reading','Presentation','SimulationAdvanceIntervalV1','Physics substep','contract_activation_effect = none')
foreach ($term in $required) {
  if (-not (Select-String -LiteralPath $adr -SimpleMatch $term)) { throw "Missing: $term" }
}
git diff --check
```

Expected: exit 0.

- [ ] **Step 5: Commit**

Run:

```powershell
git add -- docs/architecture/decisions/2026-07-28-simulation-cadence-presentation-separation.md docs/architecture/decisions/README.md
git commit -m "docs: separate simulation and presentation cadence"
```

Expected: two-file commit.

### Task 2: Review the isolated Cadence ChangeSet

**Files:**
- Review: `docs/architecture/decisions/2026-07-28-simulation-cadence-presentation-separation.md`
- Review: `docs/architecture/decisions/README.md`

**Interfaces:**
- Consumes: Task 1 commit
- Produces: PR-ready Decision-only diff

- [ ] **Step 1: Verify exact scope**

Run:

```powershell
$expected = @(
  'docs/architecture/decisions/2026-07-28-simulation-cadence-presentation-separation.md',
  'docs/architecture/decisions/README.md'
)
$changed = @(git diff --name-only main...HEAD)
$unexpected = @($changed | Where-Object { $_ -notin $expected })
if ($unexpected.Count -or $changed.Count -ne 2) { throw "Unexpected files: $($changed -join ', ')" }
git diff --check main...HEAD
```

Expected: exit 0.

- [ ] **Step 2: Verify the Decision ID and index are unique**

Run:

```powershell
$idFiles = @(rg -l -F 'mirakan.decision.simulation-cadence-presentation-separation' docs/architecture)
if ($idFiles.Count -ne 1) { throw "Decision ID appears in $($idFiles.Count) files" }
$rowCount = @(Select-String -LiteralPath 'docs/architecture/decisions/README.md' -SimpleMatch '2026-07-28-simulation-cadence-presentation-separation.md').Count
if ($rowCount -ne 1) { throw "Decision row count is $rowCount" }
```

Expected: exit 0.
