# Animation Contract Forward-Port Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `b7c6540`のAnimation設計だけを、現行`main`のAnimation Owner文書へガバナンス適合状態でforward-portする。

**Architecture:** 旧commitは参照専用にし、最新`main`から`codex/animation-contract-forward-port`を作る。Product、Debug、Mobile変更は移植せず、Animation Ownerが所有するGraph、IK、retarget、LOD、planned-action境界だけを一つのChangeSetにする。

**Tech Stack:** Git、Markdown、PowerShell、ripgrep

## Global Constraints

- `docs/superpowers/specs/2026-07-28-unique-commit-migration-design.md`を要求仕様とする。
- `b7c6540`をcherry-pick、merge、rebaseしない。
- 現行Owner Headerの10 fieldと順序を変更しない。
- MCD、Operation、Capability、実装、Qualificationのcurrent activationを主張しない。
- `docs/plans/`および旧AI Discovery specを追加しない。

---

### Task 1: Animation Graphと評価契約のforward-port

**Files:**
- Modify: `docs/architecture/05-simulation/animation.md:40`
- Test: `docs/architecture/05-simulation/animation.md`

**Interfaces:**
- Consumes: 現行`Typed Animation Graph`、`AnimationInstanceV1`、`RootMotionProposalV1`、Schedulingの`SimulationAdvanceIntervalV1`
- Produces: `AI-readable Graph closure`、Blend Space／同期契約、IK Query／Result Binding、Animation Definition Closure

- [ ] **Step 1: Verify the destination does not already contain the selected sections**

Run:

```powershell
$file = 'docs/architecture/05-simulation/animation.md'
$terms = @('AI-readable Graph closure','Blend Space、同期、event／root motion','IK request、座標空間、評価順','Foot placement query／result binding','MCD／Tool／Debug Binding Definition Closure')
foreach ($term in $terms) {
  if (Select-String -LiteralPath $file -SimpleMatch $term) { throw "Already present: $term" }
}
```

Expected: exit 0 with no output.

- [ ] **Step 2: Add the selected Graph and evaluation sections**

Under the existing section numbers, add this exact structure:

```markdown
#### 2.1.1 AI-readable Graph closure
#### 2.1.2 Blend Space、同期、event／root motion
### 2.4 外部Engine比較から採るCoverage
### 4.1 IK request、座標空間、評価順
### 4.2 Foot placement query／result binding
### 4.3 Retarget、LOD、memory／failure
### 5.1 MCD／Tool／Debug Binding Definition Closure
```

The prose must enforce all of these requirements:

- Graph／State／Transition／Port／ParameterはStable ID、revision、typed value、unit、semantic hashで解決し、layout、locale、display nameをidentityにしない。
- Blend Spaceは1D boundaryおよび2D triangulation tieを決定論的に解決し、weightをfiniteかつ正規化する。
- Gameplay event sourceとroot-motion authorityを一意にし、Presentation sampleからauthoritative eventを生成しない。
- IK requestはspace、subject generation、target／pole、solver、weight、priorityを持ち、canonical producer順で解決する。
- Foot placementはT30 Query Binding、T50 generic execution、T60 Result Binding、T80 consumptionを同じintervalへ束縛し、live Physics queryを禁止する。
- Retarget、LOD、scratch／published pose memory、last-valid／bind-pose failureを別契約として扱う。
- Runtime／semantic Type、Authoring Tool、Debug Bindingの三closureを独立状態として扱い、current Operation集合は`[]`、Debug Bindingは`required_unmaterialized`を維持する。

- [ ] **Step 3: Add focused validation and rejection cases**

In section 6, add fixtures for Blend Space boundary／tie、IK space／generation、foot-placement stale binding、AI-readable completeness／gap, and Definition Closure partial activation. Add explicit rejections for Backend-default blend semantics, display-name command generation, and Debug projection mutation.

- [ ] **Step 4: Run content assertions**

Run:

```powershell
$file = 'docs/architecture/05-simulation/animation.md'
$required = @(
  '#### 2.1.1 AI-readable Graph closure',
  '#### 2.1.2 Blend Space、同期、event／root motion',
  '### 4.2 Foot placement query／result binding',
  '### 5.1 MCD／Tool／Debug Binding Definition Closure',
  'current Operation集合は`[]`',
  '`required_unmaterialized`'
)
foreach ($term in $required) {
  if (-not (Select-String -LiteralPath $file -SimpleMatch $term)) { throw "Missing: $term" }
}
if (Test-Path 'docs/plans') { throw 'Retired docs/plans directory was restored' }
```

Expected: exit 0.

- [ ] **Step 5: Validate and commit**

Run:

```powershell
git diff --check
git diff --stat
git status --short
git add -- docs/architecture/05-simulation/animation.md
git commit -m "docs: forward-port animation contracts"
```

Expected: one-file commit with no whitespace errors.

### Task 2: Review the isolated Animation ChangeSet

**Files:**
- Review: `docs/architecture/05-simulation/animation.md`

**Interfaces:**
- Consumes: Task 1 commit
- Produces: PR-ready Animation-only diff

- [ ] **Step 1: Verify scope**

Run:

```powershell
$changed = @(git diff --name-only main...HEAD)
if ($changed.Count -ne 1 -or $changed[0] -ne 'docs/architecture/05-simulation/animation.md') {
  throw "Unexpected files: $($changed -join ', ')"
}
git diff --check main...HEAD
```

Expected: exit 0.

- [ ] **Step 2: Verify no activation claim was introduced**

Run:

```powershell
$added = git diff --unified=0 main...HEAD -- docs/architecture/05-simulation/animation.md
$forbidden = @('Capability state is active','current callable Operation','implementation=implemented','qualification=pass')
foreach ($term in $forbidden) {
  if ($added -match [regex]::Escape($term)) { throw "Forbidden activation claim: $term" }
}
```

Expected: exit 0.
