# Legacy Branch Reconciliation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Preserve the dirty legacy worktree, reconcile every unique architecture-comprehension requirement into the current canonical documents and plans, merge the result to `main`, and remove obsolete worktrees and branches without losing information.

**Architecture:** Treat the old worktree as immutable source evidence after a local checkpoint commit. Port meaning, not paths: current `docs/architecture` owners receive active contracts, current `docs/plans` receive synchronized implementation work, and a disposition ledger proves how each source hunk was preserved, merged, or removed. Cleanup occurs only after the integration PR is merged and post-merge validation passes.

**Tech Stack:** Markdown architecture contracts, PowerShell, ripgrep, Git, GitHub CLI

## Global Constraints

- Do not resurrect active files under the removed legacy `docs/superpowers/specs` hierarchy.
- Do not delete the dirty worktree or either legacy branch before its content is checkpointed and reconciled.
- Keep one normative owner per concept; merge authority-manifest requirements into `ArchitectureMetadataV1` and the relation registry instead of defining a second manifest authority.
- Keep all plans synchronized with canonical paths, names, dependencies, verification commands, and actual completion state in the same commits as their corresponding architecture changes.
- Preserve the current repository-wide `model_reasoning_effort = "xhigh"` decision merged by PR #5.
- Do not create compatibility aliases, redirects, duplicate schema definitions, or old Review-set fixed counts.
- Use the existing architecture validation profile: strict UTF-8, no BOM, LF, no trailing whitespace, balanced fences, resolvable relative Markdown links, no unresolved placeholder markers.
- Do not remove a branch until its tip or its semantically reconciled replacement is reachable from `origin/main` and all associated worktrees are clean.
- Execute Tasks 1–7 from the isolated `.worktrees/legacy-branch-reconciliation` worktree on `codex/legacy-branch-reconciliation`; perform final worktree removal from the main checkout.

---

### Task 1: Checkpoint the dirty source worktree

**Files:**
- Modify and commit in source worktree: `docs/superpowers/specs/2026-07-18-ai-native-game-engine-authoring-design.md`
- Modify and commit in source worktree: `docs/superpowers/specs/2026-07-19-ai-verification-evaluation-provenance-design.md`
- Modify and commit in source worktree: `docs/superpowers/specs/2026-07-19-authoring-model-project-state-design.md`
- Modify and commit in source worktree: `docs/superpowers/specs/2026-07-19-engine-foundation-architecture-design.md`
- Modify and commit in source worktree: `docs/superpowers/specs/2026-07-19-runtime-integration-lifetime-performance-design.md`
- Modify and commit in source worktree: `docs/superpowers/specs/README.md`
- Create and commit in source worktree: `docs/superpowers/plans/2026-07-21-ai-readable-architecture-comprehension-closure.md`

**Interfaces:**
- Consumes: dirty `codex/ai-readable-architecture-closure` worktree at merge commit `c563d63`.
- Produces: one local checkpoint commit whose tree exactly contains the six modified documents and one untracked plan.

- [x] **Step 1: Capture source status and verify the expected seven paths**

Run:

```powershell
$repo = Split-Path -Parent (git rev-parse --path-format=absolute --git-common-dir)
$wt = Join-Path $repo '.worktrees\ai-readable-architecture-closure'
git -C $wt status --short
git -C $wt diff --check
git -C $wt diff --numstat
```

Expected: six modified specification paths, one untracked plan, no whitespace error.

- [x] **Step 2: Stage only the seven source paths**

Run:

```powershell
git -C $wt add -- docs/superpowers/specs/2026-07-18-ai-native-game-engine-authoring-design.md docs/superpowers/specs/2026-07-19-ai-verification-evaluation-provenance-design.md docs/superpowers/specs/2026-07-19-authoring-model-project-state-design.md docs/superpowers/specs/2026-07-19-engine-foundation-architecture-design.md docs/superpowers/specs/2026-07-19-runtime-integration-lifetime-performance-design.md docs/superpowers/specs/README.md docs/superpowers/plans/2026-07-21-ai-readable-architecture-comprehension-closure.md
git -C $wt diff --cached --check
git -C $wt diff --cached --name-status
```

Expected: exactly seven paths staged.

- [x] **Step 3: Create the local checkpoint commit**

Run:

```powershell
git -C $wt commit -m "docs: checkpoint architecture comprehension closure"
git -C $wt rev-parse HEAD
git -C $wt status --short --branch
```

Expected: commit succeeds and source worktree is clean. Do not push this legacy branch.

Completed: local checkpoint `0e86c0f7d7bc0cabe36ee7871e9ac9b83e925c86` contains exactly the seven expected paths; the source worktree is clean and the legacy branch was not pushed.

### Task 2: Build the complete source disposition ledger

**Files:**
- Create: `docs/plans/2026-07-22-legacy-branch-reconciliation-disposition.md`
- Modify: `docs/plans/2026-07-22-legacy-branch-reconciliation-implementation-plan.md`

**Interfaces:**
- Consumes: Task 1 checkpoint commit and the Document System Restructure Decision §12.
- Produces: file-level and hunk-level `Preserved | Merged | Removed` classification with source commit, destination owner, canonical identifier, and verification evidence.

- [x] **Step 1: Record exact source evidence**

Run:

```powershell
$repo = Split-Path -Parent (git rev-parse --path-format=absolute --git-common-dir)
$wt = Join-Path $repo '.worktrees\ai-readable-architecture-closure'
$checkpoint = git -C $wt rev-parse HEAD
git -C $wt show --stat --oneline $checkpoint
git -C $wt show --format= --unified=0 $checkpoint
```

Expected: 20 source hunks containing 448 added and 7 deleted lines (455 changed lines in total), including the 241-line plan, are available for classification.

- [x] **Step 2: Create the disposition ledger**

Write one row for every source diff hunk with these exact columns:

```text
source_commit | source_path | source_hunk | source_line_count | disposition | canonical_destination | canonical_identifiers | evidence
```

Required classifications:

```text
old README authority manifest -> Merged -> ArchitectureMetadataV1 + relation registry + generated Index
old AI-native external terms -> Preserved -> editor-workspace-ux.md + project-state.md consumer rule
old Project State explain projection -> Merged -> Control Plane design/plan + project-state.md consumer rule
old Runtime GameHost loop -> Preserved -> scheduling-lifetime.md
old AI Verification comprehension fixture -> Preserved -> ai-verification-provenance.md
old Core Architecture plan-count edit -> Removed -> superseded by metadata-generated inventory and current implementation plans
old plan -> Merged -> canonical plan under docs/plans with current paths and statuses
```

- [x] **Step 3: Verify ledger closure**

Run a PowerShell audit that sums `source_line_count`, rejects blank disposition/destination/evidence fields, and requires all seven source categories above.

Expected: unclassified hunk count 0 and the source line count equals the checkpoint diff inventory.

Completed: the ledger has 20 rows totaling 455 changed lines, blank required fields 0, and all seven required source categories present.

### Task 3: Reconcile architecture authority and explain projection with the current Control Plane

**Files:**
- Modify: `docs/plans/2026-07-22-architecture-evolution-control-plane-design.md:149-305`
- Modify: `docs/plans/2026-07-22-architecture-evolution-control-plane-implementation-plan.md:53-342`
- Modify: `docs/architecture/03-authoring/project-state.md:134-169`

**Interfaces:**
- Consumes: `ArchitectureMetadataV1`, relation registry, generated Architecture Index, `ProjectRevision`, canonical Source IDs.
- Produces: bounded architecture-explain projection and query without a duplicate normative manifest.

- [x] **Step 1: Run the pre-change missing-contract audit**

Run:

```powershell
rg -n "ArchitectureExplainProjectionV1|authoring.explain_architecture|omitted_ranges" docs/architecture docs/plans
```

Expected: no canonical definition.

- [x] **Step 2: Extend the Control Plane design**

Add `ArchitectureExplainProjectionV1` as a deterministic, disposable projection generated from exact `ArchitectureMetadataV1`, relation-registry, Product-registry, Contract, World, Target, and Source-revision inputs. Preserve the source bounds of 256 entries per category, 1,024 dependency edges, and 2 MiB canonical encoding. Require signed continuation and explicit omitted ranges; prohibit summaries from becoming authority.

State explicitly that the old `NormativeAuthorityManifestV1` is merged into `ArchitectureMetadataV1` plus relation registry and is not an active alias.

- [x] **Step 3: Extend the Control Plane implementation plan**

Add exact schema, parser, generator, CLI query, negative fixture, deterministic-byte, stale-revision, omitted-range, and CI-gate steps for `ArchitectureExplainProjectionV1`. Update the file map and public interfaces with exact paths and signatures.

- [x] **Step 4: Add the Project State consumer rule**

Define `authoring.explain_architecture` as read-only. Require `scope`, `field_mask`, optional Target Profile, and exact Project revision; require subsequent mutation to respecify canonical Stable IDs, typed operations, and expected revision. Reject direct mutation from projection entries, natural-language summaries, stale continuation, or omitted evidence.

- [x] **Step 5: Verify ownership uniqueness**

Run:

```powershell
rg -n "NormativeAuthorityManifestV1|ArchitectureExplainProjectionV1|authoring.explain_architecture" docs/architecture docs/plans
```

Expected: no active `NormativeAuthorityManifestV1` definition; one projection definition in the Control Plane design, one implementation-plan interface, and one Project State consumer rule.

Completed: active old-manifest definitions 0, Control Plane projection definitions 1, implementation interfaces 1, and Project State consumer rules 1.

### Task 4: Port external-engine terminology resolution

**Files:**
- Modify: `docs/architecture/03-authoring/editor-workspace-ux.md:292-331`
- Modify: `docs/architecture/03-authoring/project-state.md:331-343`
- Modify: `docs/plans/2026-07-21-ai-readable-architecture-comprehension-closure.md`

**Interfaces:**
- Consumes: canonical concept IDs, Requirement context, Project Capability, Target Profile, owner registry.
- Produces: `ExternalEngineConceptResolutionV1` and `question_required` boundary.

- [x] **Step 1: Run the pre-change missing-contract audit**

Run:

```powershell
rg -n "ExternalEngineConceptResolutionV1|question_required" docs/architecture/03-authoring
```

Expected: no canonical definition for the external-engine resolver.

- [x] **Step 2: Define the input-only resolver in Editor／Workspace UX**

Add the exact fields from the checkpoint source: request ID, source engine family, source term, context-summary hash, candidate canonical concepts, selected concept, resolution status, and evidence refs. Keep `resolved | question_required | unsupported` as the closed status set.

- [x] **Step 3: Preserve ambiguity and non-authority rules**

Require a question when Scene／Level／Cell／UI／Composition candidates affect owner, Save, transition, streaming, Target, or Project structure. Prohibit persistent one-to-one aliases, external paths/indexes as identity, and implicit approximation of unsupported features.

- [x] **Step 4: Add the Project State acceptance boundary**

Allow only the selected canonical concept, exact target Stable ID, typed operation, and expected revision into a ChangeSet. The resolver object itself remains read-only evidence and cannot be committed.

- [x] **Step 5: Verify the port**

Run:

```powershell
rg -n "ExternalEngineConceptResolutionV1|Unity Scene|Unreal Level|Godot Scene|question_required" docs/architecture/03-authoring
```

Expected: one definition, explicit ambiguity examples, and one ChangeSet boundary.

Completed: resolver definitions 1, explicit cross-engine ambiguity examples 1, and Project State ChangeSet boundaries 1.

### Task 5: Port the GameHost outer loop contract

**Files:**
- Modify: `docs/architecture/04-runtime/scheduling-lifetime.md:84-249`
- Modify: `docs/plans/2026-07-21-ai-readable-architecture-comprehension-closure.md`

**Interfaces:**
- Consumes: Application lifecycle, T00–T110, R00–R70, InputSnapshot, RenderSnapshot, PausePolicyV1.
- Produces: `GameHostLoopStateV1`, rational 60 Hz clock, catch-up/input/pause/surface/fault rules.

- [x] **Step 1: Run the pre-change missing-contract audit**

Run:

```powershell
rg -n "GameHostLoopStateV1|single_tick_step|tick_duration_ns" docs/architecture/04-runtime/scheduling-lifetime.md
```

Expected: no complete GameHost outer-loop definition.

- [x] **Step 2: Define `GameHostLoopStateV1` and the rational clock**

Port the checkpoint fields and use this exact tick-duration formula:

```text
floor((tick_id + 1) * 1,000,000,000 / 60)
- floor(tick_id * 1,000,000,000 / 60)
```

Prohibit floating point and repeated `16,666,666 ns` accumulation.

- [x] **Step 3: Define the outer-loop sequence and catch-up policy**

Fix lifecycle latch, one monotonic sample, transition, one device sample, zero-to-four complete fixed ticks, latest complete snapshot, presentation-only interpolation, zero-or-one render, retire/wait. Record dropped wall time and never duplicate edge/text/gesture input across catch-up ticks.

- [x] **Step 4: Separate pause, debug, lifecycle, surface, and headless states**

Keep Gameplay pause and Debug pause as separate owners. Apply Debug pause only after T110; execute `single_tick_step` as exactly one T00–T110 sequence. Preserve World on surface loss, stop simulation/render/audio only for Suspend, and run authoritative ticks without render phases for headless.

- [x] **Step 5: Add fault and qualification cases**

Do not publish or render a faulted tick. Add 60-tick exact-second, four-step clamp, input-edge nonduplication, Gameplay pause, Debug step, SurfaceUnavailable, Suspend, headless, stale snapshot, and fault fixtures.

- [x] **Step 6: Verify the port**

Run:

```powershell
rg -n "GameHostLoopStateV1|gameplay_clock_mode|debug_execution_mode|single_tick_step|SurfaceUnavailable|tick_duration_ns" docs/architecture/04-runtime/scheduling-lifetime.md
```

Expected: one complete canonical contract and its qualification cases.

Completed: canonical state definitions 1, rational tick formulas 1, and GameHost qualification matrices 1.

### Task 6: Port architecture-comprehension evaluation and synchronize plans

**Files:**
- Modify: `docs/architecture/01-governance/ai-verification-provenance.md:131-248`
- Create: `docs/plans/2026-07-21-ai-readable-architecture-comprehension-closure.md`
- Modify: `docs/plans/2026-07-22-architecture-evolution-control-plane-design.md`
- Modify: `docs/plans/2026-07-22-architecture-evolution-control-plane-implementation-plan.md`
- Modify: `docs/plans/2026-07-22-legacy-branch-reconciliation-disposition.md`
- Modify: `docs/plans/2026-07-22-legacy-branch-reconciliation-implementation-plan.md`

**Interfaces:**
- Consumes: canonical metadata, explain projection, external-term resolution, GameHost contract.
- Produces: `ArchitectureComprehensionCaseV1`, `ArchitectureComprehensionFixtureV1`, updated executable plans, completed disposition evidence.

- [x] **Step 1: Define the evaluation contracts**

Port the 240-case split: 80 direct canonical terms, 80 external-engine terms, 80 ambiguous/conflicting/stale-evidence cases. Each case holds expected canonical concepts, owner entries, phase/lifetime entries, required evidence refs, forbidden claims, and question requirement.

- [x] **Step 2: Define grading and hard gates**

Use code-based canonical-ID comparison rather than string similarity. Require 100% Blocking/High owner and phase/lifetime accuracy; zero fabricated IDs/phases, wrong authority, unsupported claims, question bypass, and stale/omitted evidence acceptance. Run three times and report worst case. Separate runner/materialization/hash failures as infrastructure failures.

- [x] **Step 3: Canonicalize the old implementation plan**

Move the old plan content to `docs/plans/2026-07-21-ai-readable-architecture-comprehension-closure.md`. Replace every legacy path, fixed 46-document count, `NormativeAuthorityManifestV1`, and obsolete execution-mode statement. Preserve the original goal and source commit as provenance. Mark completed only after the destination contract and verification command have passed.

- [x] **Step 4: Synchronize all affected current plans**

Update the Control Plane design and implementation plan with the final names, schema ownership, Task dependencies, negative fixtures, and completion gates. Update the reconciliation plan and disposition ledger with actual checkpoint and integration commit IDs as they become available.

- [x] **Step 5: Verify plan freshness**

Run:

```powershell
rg -n "docs/superpowers/specs|review_set_document_count|NormativeAuthorityManifestV1|Execution mode:.*ai-readable-architecture-closure" docs/plans/2026-07-21-ai-readable-architecture-comprehension-closure.md docs/plans/2026-07-22-architecture-evolution-control-plane-design.md docs/plans/2026-07-22-architecture-evolution-control-plane-implementation-plan.md
```

Expected: no stale path, fixed old count, obsolete manifest type, or obsolete execution-mode statement.

Completed: canonical Case definitions 1, Fixture definitions 1, 80×3 corpus split 1, and forbidden legacy plan markers 0.

### Task 7: Validate, publish, merge, and clean up

**Files:**
- Verify: `docs/**/*.md`
- Commit: all Task 2–6 destination files
- Delete after merge: worktree `.worktrees/ai-readable-architecture-closure`
- Delete after merge: worktree `.worktrees/legacy-branch-reconciliation`
- Delete after merge: local `codex/ai-readable-architecture-closure`
- Delete after merge: local and remote `codex/c1-gameplay-capability-closure`
- Delete after merge: remote `codex/architecture-document-restructure`

**Interfaces:**
- Consumes: all reconciled contracts, plans, ledger, and passing validation evidence.
- Produces: merged `main`, synchronized local checkout, no obsolete worktree or branches.

- [x] **Step 1: Run full documentation validation**

Validate all Markdown as strict UTF-8 without BOM/CR, no trailing whitespace, balanced fences, and resolvable relative `.md` links. Run:

```powershell
git diff --check
$credentialPattern = '(BEGIN (RSA|OPENSSH|EC|DSA) PRIVATE KEY|gh[pousr]_[A-Za-z0-9_]{20,}|AKIA[0-9A-Z]{16}|AIza[0-9A-Za-z_-]{20,})'
$localPathPattern = '(?:C:' + '[/\\]Users[/\\]|G:' + '[/\\]workspace[/\\])'
rg -n --hidden -g '!.git' -g '!.git/**' -g '!.worktrees/**' -g '!.superpowers/**' "$credentialPattern|$localPathPattern" .
```

Expected: validation exit 0 and no secret/local-workspace pattern match.

Completed: 57 Markdown files checked; UTF-8／BOM／CR／trailing-whitespace／fence／relative-link errors 0, `git diff --check` errors 0, and secret／local-workspace matches 0.

- [x] **Step 2: Run semantic closure audits**

Require:

```text
source disposition unclassified count = 0
PR #3 extracted identifier missing count = 0
legacy active docs/superpowers/specs reference count = 0
active NormativeAuthorityManifestV1 definition count = 0
ArchitectureExplainProjectionV1 canonical definition count = 1
ExternalEngineConceptResolutionV1 canonical definition count = 1
GameHostLoopStateV1 canonical definition count = 1
ArchitectureComprehensionCaseV1 canonical definition count = 1
ArchitectureComprehensionFixtureV1 canonical definition count = 1
```

Completed: ledger 20 rows／455 lines／blank 0; PR #3 identifier gaps 0; legacy active-path refs 0; old-manifest active definitions 0; all five canonical definition counts 1.

- [ ] **Step 3: Commit the reconciled implementation**

Run:

```powershell
git add -- docs/architecture docs/plans
git diff --cached --check
git diff --cached --name-status
git commit -m "docs: reconcile architecture comprehension closure"
```

Expected: no source-worktree legacy path is staged.

- [ ] **Step 4: Push and open a draft PR**

Push `codex/legacy-branch-reconciliation`, create a draft PR targeting `main`, include checkpoint SHA, source disposition counts, validation evidence, and cleanup list. Mark ready only when GitHub reports `MERGEABLE/CLEAN` and no required check fails.

- [ ] **Step 5: Merge and verify local main**

Merge with expected head SHA, switch to local `main`, and run `git pull --ff-only origin main`. Re-run full Markdown and semantic closure audits against the merged tree.

- [ ] **Step 6: Remove obsolete worktrees and branches**

Before each deletion, verify the associated worktree is clean and the integration merge commit is on `origin/main`. Run cleanup from the main checkout, never from inside either worktree. Then:

```powershell
$repo = Split-Path -Parent (git rev-parse --path-format=absolute --git-common-dir)
git -C $repo worktree remove -- (Join-Path $repo '.worktrees\ai-readable-architecture-closure')
git -C $repo worktree remove -- (Join-Path $repo '.worktrees\legacy-branch-reconciliation')
git worktree prune
git branch -d codex/ai-readable-architecture-closure
git branch -d codex/c1-gameplay-capability-closure
git push origin --delete codex/c1-gameplay-capability-closure
git push origin --delete codex/architecture-document-restructure
```

Expected: commands succeed without force deletion.

- [ ] **Step 7: Final branch and repository audit**

Run:

```powershell
git status --short --branch
git branch -vv
git worktree list --porcelain
git ls-remote --heads origin
git rev-parse HEAD
git rev-parse origin/main
```

Expected: clean `main`, matching local/remote SHA, no obsolete branch refs, and only intentionally active worktrees.
