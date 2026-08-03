# AI-native Production Loop Architecture Documentation Reconstruction Execution Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. Multi-agent delegation is not authorized for this run.

**Goal:** Close the current target-design gaps in AI game understanding, creative iteration, First Playable, Evidence freshness, executable Core closure, and type ownership without claiming implementation or retaining pre-materialization compatibility aliases.

**Architecture:** Add one Authoring Owner for the end-to-end Game production loop and keep Governance, Project transaction, Developer Testing, Gameplay, Asset, Core, and Product responsibilities in their existing Owners. Move provisional shapes to the new Owner, make all consumers reference exact canonical types, and keep all current materialized／active／operational sets empty.

**Tech Stack:** Japanese Markdown Architecture Owners, manually maintained Architecture Index, PowerShell／ripgrep read-only validation, Git.

## Global Constraints

- Preserve the required Owner header order from `docs/architecture/01-governance/architecture-governance.md`.
- Preserve Japanese canonical prose and English ASCII technical identities.
- Keep `文書状態: review`, `実装状態: absent`, and `検証状態: design-reviewed` for every affected Owner.
- Do not create C++ implementation, executable MCD Schema, Registry, Fixture artifact, Receipt, Build system, CI workflow, timeline, estimate, or staffing plan.
- Do not retain old candidate Schema, alias, dual reader, migration Operation, or backward-compatibility layer; no materialized consumer exists.
- Keep current `materialized_operations`, `contract_active_operations`, `active_operations`, and `operational_operations` exact `[]`.
- Do not introduce a normative dependency cycle or reverse the Product → Governance → Foundation → Authoring／Runtime direction.
- External facts must cite current official primary sources and be separated from Miraikanai project decisions.
- Update the Architecture Index only after Owner paths, headers, and canonical references are final.
- A document-only change never proves implementation, Qualification, Activation, Release, or Product completion.

---

### Task 1: Add the Game Production Loop Owner

**Files:**
- Create: `docs/architecture/03-authoring/game-production-loop.md`
- Modify: `docs/architecture/README.md`

**Interfaces:**
- Consumes: `ProjectRefV1`, `ProjectRevisionRefV1`, generic Document header, AI Task Authorization, Evidence refs, Developer Test Result refs.
- Produces: `GameProductionSubjectV1`, `GameIntentSessionV1`, `GameIntentDraftV1`, `GameBriefDocumentV1`, `GameQuestionRecordV1`, `GameAssumptionRecordV1`, `GameDecisionRecordV1`, `GameSpecDocumentV1`, `GameRequirementTraceabilityV1`, `GameUnderstandingClosureV1`, `GameExperienceGoalSetV1`, `PlaytestSessionDefinitionV1`, `PlaytestObservationSetV1`, `GameExperienceEvaluationV1`, `GameIterationDecisionV1`, `GameProductionLoopClosureV1` and exact Ref families.

- [ ] **Step 1: Record the failing ownership evidence**

Run:

```powershell
rg -n "GameBriefV1|GameSpecDocumentV1|GameUnderstandingClosureV1|PlaytestObservationSetV1" docs/architecture
```

Expected: Game understanding types occur only as provisional appendix candidates; Playtest observation types are absent.

- [ ] **Step 2: Create the Owner with the exact required header**

Add `mirakan.arch.game-production-loop` with the scope, non-scope, normative dependencies, project-decision evidence class, and `外部根拠確認日: none`. Define the complete record and Ref fields, bounds, canonical sorting, content hashes, state transitions, failure cases, target Operation families, current empty sets, and completion conditions from the approved Design.

- [ ] **Step 3: Add the Owner to the Authoring section of the Index**

Place it after Project State and before Editor Workspace so the reading order is Project transaction → Game production meaning → presentation.

- [ ] **Step 4: Verify Owner header and Index membership**

Run:

```powershell
rg -n "mirakan.arch.game-production-loop|game-production-loop.md" docs/architecture/03-authoring/game-production-loop.md docs/architecture/README.md
```

Expected: one Owner ID definition and one Index link.

- [ ] **Step 5: Commit**

```powershell
git add -- docs/architecture/03-authoring/game-production-loop.md docs/architecture/README.md
git commit -m "docs: add canonical game production loop owner"
```

### Task 2: Bind Project State, Editor, Testing, and Asset consumers

**Files:**
- Modify: `docs/architecture/03-authoring/project-state.md`
- Modify: `docs/architecture/03-authoring/editor-workspace-ux.md`
- Modify: `docs/architecture/03-authoring/developer-testing.md`
- Modify: `docs/architecture/03-authoring/asset-lifecycle.md`
- Modify: `docs/architecture/appendices/project-target-readiness-fixture-catalog.md`

**Interfaces:**
- Consumes: Task 1 canonical record families.
- Produces: bootstrap Project identity boundary, canonical Document-kind bindings, typed Editor workflow states, automated-test／human-observation separation, and exact three-lane Asset boundaries.

- [ ] **Step 1: Make Project bootstrap identity explicit**

In Project State, define that bootstrap allocates a stable `ProjectRefV1` before revision 1 without publishing an aggregate or partial Project. Add Game Brief／Spec／Decision as canonical Document kinds owned by Game Production Loop and remove the candidate-only interpretation.

- [ ] **Step 2: Replace Editor prose-only workflow state**

Bind Intent, Questions, Plan, Result, Playtest, and iterative conversation to exact Task 1 refs. Require a new Authorization for an iteration Task and prohibit conversation summaries from becoming Project authority.

- [ ] **Step 3: Separate automated tests from creative observations**

In Developer Testing, state that `play_scenario` produces technical Result refs consumed by `PlaytestSessionDefinitionV1` but cannot create human Observation, Experience Evaluation, or Gameplay Approval.

- [ ] **Step 4: Connect Asset generation lanes**

Bind `domain_pack_reference | user_provided` to `ai_composed_game`, `external_generated` to `ai_generated_external_content`, and Native／Shader to `ai_generated_project_source`. Preserve the existing provenance, rights, safety, and Code Owner gates.

- [ ] **Step 5: Remove shadow candidates from the readiness catalog**

Replace candidate `GameSpecDocument`／`DecisionLedgerDocument` shape rows with direct canonical Owner references. Do not reproduce fields.

- [ ] **Step 6: Verify consumer-only ownership**

Run:

```powershell
rg -n "GameBriefDocumentV1|GameSpecDocumentV1|PlaytestObservationSetV1|GameExperienceEvaluationV1" docs/architecture/03-authoring docs/architecture/appendices/project-target-readiness-fixture-catalog.md
```

Expected: definitions occur only in `game-production-loop.md`; consumers link to that Owner.

- [ ] **Step 7: Commit**

```powershell
git add -- docs/architecture/03-authoring/project-state.md docs/architecture/03-authoring/editor-workspace-ux.md docs/architecture/03-authoring/developer-testing.md docs/architecture/03-authoring/asset-lifecycle.md docs/architecture/appendices/project-target-readiness-fixture-catalog.md
git commit -m "docs: bind authoring consumers to production loop"
```

### Task 3: Reconstruct AI authority and Evidence freshness

**Files:**
- Modify: `docs/architecture/01-governance/ai-security-approval.md`
- Modify: `docs/architecture/01-governance/ai-verification-provenance.md`
- Modify: `docs/architecture/appendices/ai-security-assumptions-guide.md`
- Modify: `docs/architecture/appendices/ai-evidence-envelope-fixture-catalog.md`
- Modify: `docs/architecture/03-authoring/editor-ui-framework.md`

**Interfaces:**
- Consumes: Task 1 Game Understanding and iteration refs.
- Produces: generic Task Capsule binding, authority separation, `EvidenceEffectiveStateV1`, and no shadow Game Understanding schema.

- [ ] **Step 1: Remove Game meaning from AI Security**

Keep Task Capsule, Authorization, Risk, Consent, Human Gameplay Approval, Code Owner, Provider trust, and Activation. Replace provisional Game Understanding language with an opaque exact Type／Ref binding to the new Authoring Owner; Security must not define Brief／Spec payload semantics.

- [ ] **Step 2: Define Evidence effective state in the parent Owner**

Add `EvidenceEffectiveStateV1 = fresh | expired | revoked | invalid` under AI Verification §10 with priority `invalid > revoked > expired > fresh`. Define schema／signature／purpose／context invalidity, transitive revocation, time／policy expiry, and fresh admissibility exactly.

- [ ] **Step 3: Convert the assumptions guide to a fixture guide**

Delete candidate record shapes and reference the canonical Task 1 types. Retain only beginner explanation, negative scenarios, and security-specific guidance that does not redefine fields.

- [ ] **Step 4: Repair the Evidence catalog reference**

Replace the nonexistent `§10.1` claim with a relative link to the new parent Owner subsection. Require consumers to count only `fresh` Evidence.

- [ ] **Step 5: Update UI Framework context exclusion**

Remove the statement that Game Understanding is provisional and instead require a complete, current, authorized projection binding. Keep incomplete and stale projections rejected.

- [ ] **Step 6: Verify freshness uniqueness and shadow removal**

Run:

```powershell
rg -n "EvidenceEffectiveStateV1|fresh \| expired \| revoked \| invalid|§10\.1を正本|これは許可済みtop-level Schema集合ではない" docs/architecture
```

Expected: one canonical state definition, no stale `§10.1` assertion, and no provisional top-level Game schema disclaimer.

- [ ] **Step 7: Commit**

```powershell
git add -- docs/architecture/01-governance/ai-security-approval.md docs/architecture/01-governance/ai-verification-provenance.md docs/architecture/appendices/ai-security-assumptions-guide.md docs/architecture/appendices/ai-evidence-envelope-fixture-catalog.md docs/architecture/03-authoring/editor-ui-framework.md
git commit -m "docs: close AI context and evidence freshness semantics"
```

### Task 4: Establish Gameplay graph and state-owner authority

**Files:**
- Modify: `docs/architecture/03-authoring/gameplay-programming-model.md`
- Modify: `docs/architecture/04-runtime/scheduling-lifetime.md`
- Modify: `docs/architecture/04-runtime/runtime-package.md`
- Modify: `docs/architecture/03-authoring/native-game-module.md`
- Modify: `docs/architecture/appendices/gameplay-generated-projection-fixture-catalog.md`

**Interfaces:**
- Produces: `GameSystemDependencyGraphV1／RefV1`, `SystemImplementationSetV1／RefV1`, `GameStateOwnerProjectionV1／RefV1`.
- Consumed by: Game Understanding Closure, Scheduling, Runtime Package, Native Module, Cook／Build fixtures.

- [ ] **Step 1: Confirm the current undefined reference**

Run:

```powershell
rg -n "GameSystemDependencyGraphV1|SystemImplementationSetV1" docs/architecture
```

Expected: consumer references exist without a canonical definition.

- [ ] **Step 2: Define graph, implementation, and state-owner records**

In Gameplay Programming Model, define full IDs／versions／hashes, Project revision, Contract set, System nodes, dependency edges, phase, State reads／writes, Command／Event sets, implementation variants, Target binding, and canonical sort. Require DAG, exactly one active authoritative owner per State Type, and exactly one selected compatible implementation per required System.

- [ ] **Step 3: Bind runtime consumers byte-exactly**

Scheduling owns phase execution only; Runtime Package carries the exact refs／hashes; Native Module validates callback manifests against them. Remove any implication that those consumers own the graph shape.

- [ ] **Step 4: Update fixture catalog**

Use the production graph only by exact Ref in Production and isolate fixture-only graph subjects from the Production Contract set.

- [ ] **Step 5: Verify single ownership**

Run:

```powershell
rg -n "GameSystemDependencyGraphV1|GameStateOwnerProjectionV1" docs/architecture/03-authoring/gameplay-programming-model.md docs/architecture/04-runtime docs/architecture/appendices/gameplay-generated-projection-fixture-catalog.md
```

Expected: one definition Owner and only linked consumer uses.

- [ ] **Step 6: Commit**

```powershell
git add -- docs/architecture/03-authoring/gameplay-programming-model.md docs/architecture/04-runtime/scheduling-lifetime.md docs/architecture/04-runtime/runtime-package.md docs/architecture/03-authoring/native-game-module.md docs/architecture/appendices/gameplay-generated-projection-fixture-catalog.md
git commit -m "docs: define canonical gameplay system graph"
```

### Task 5: Close Product claims and compact RPG First Playable

**Files:**
- Modify: `docs/architecture/00-product/product-plan.md`
- Modify: `docs/architecture/00-product/product-lifecycle.md`
- Modify: `docs/architecture/08-packs/rpg.md`
- Modify: `docs/architecture/08-packs/gameplay-features.md`
- Modify: `docs/architecture/appendices/product-execution-registry-proposal.md`

**Interfaces:**
- Consumes: Task 1 loop closure, Task 2 AI generation lanes, Task 4 gameplay graph.
- Produces: `AiGameGenerationClaimScopeV1`, `FirstPlayableDefinitionV1／RefV1`, exact compact RPG requirement and acceptance projections.

- [ ] **Step 1: Define three non-substituting AI generation claims**

Add `ai_composed_game`, `ai_generated_external_content`, and `ai_generated_project_source` to Product claim scope. Require exact lane membership in public claims and forbid cross-lane substitution.

- [ ] **Step 2: Define the compact 2D command RPG First Playable**

Add the complete receipt-free Definition from the approved Design: Title, Map, conversation, command battle, inventory／progress, Save／Load, Settings, Result, locale, input, accessibility, exact Capability／Pack／Target／Runtime Entry／scenario／journey／Evidence requirements.

- [ ] **Step 3: Bind lifecycle acceptance**

Product Lifecycle derives required journey and acceptance subjects from the Definition and refuses partial Project, manual-only, wrong Genre, wrong Target, or wrong Candidate Evidence.

- [ ] **Step 4: Align RPG Feature／Genre ownership**

Gameplay Features owns reusable State／behavior; RPG owns composition; neither owns Product completion. Reference Project content remains a normal Project.

- [ ] **Step 5: Replace the absent execution projection**

In the Product Execution Proposal, reference the exact `FirstPlayableDefinitionRefV1` and its projections. Do not invent schedule, staffing, estimates, or materialized Fixture IDs. Keep Shooter excluded from RPG acceptance.

- [ ] **Step 6: Verify no Shooter substitution**

Run:

```powershell
rg -n "FirstPlayableDefinitionV1|ai_composed_game|ai_generated_external_content|ai_generated_project_source|Shooter.*RPG|RPG.*Shooter" docs/architecture/00-product docs/architecture/08-packs docs/architecture/appendices/product-execution-registry-proposal.md
```

Expected: canonical claims and Definition exist; no text permits Shooter Evidence to satisfy compact RPG acceptance.

- [ ] **Step 7: Commit**

```powershell
git add -- docs/architecture/00-product/product-plan.md docs/architecture/00-product/product-lifecycle.md docs/architecture/08-packs/rpg.md docs/architecture/08-packs/gameplay-features.md docs/architecture/appendices/product-execution-registry-proposal.md
git commit -m "docs: close AI generation and RPG first playable claims"
```

### Task 6: Define Minimum Executable Core and Operation planning

**Files:**
- Modify: `docs/architecture/02-foundation/core-architecture.md`
- Modify: `docs/architecture/02-foundation/executable-contracts.md`
- Modify: `docs/architecture/appendices/executable-contracts-operation-planning-catalog.md`
- Modify: `docs/architecture/04-runtime/runtime-package.md`
- Modify: `docs/architecture/04-runtime/scheduling-lifetime.md`

**Interfaces:**
- Produces: `MinimumExecutableCoreDefinitionV1／RefV1`, `MinimumExecutableCoreQualificationV1／RefV1`, three new planned Operation families.
- Preserves: all current materialized／active／operational sets exact `[]`.

- [ ] **Step 1: Add Minimum Executable Core definition**

Define exact Foundation, Math, Memory, Contract set, Project read, Package reader, loader, Orchestrator, Scheduler, Jobs, ECS, headless GameHost, WorkerHost, Platform Adapter, clock／seed, diagnostic, Toolchain, and Target members.

- [ ] **Step 2: Add qualification target semantics**

Require clean configure／build, contract round-trip, headless load, deterministic advance hash, cancel／fault／shutdown, sanitizer／leak, and wrong-input negative Evidence on one Candidate. Keep all artifacts absent in current state.

- [ ] **Step 3: Register the three planned operation families**

Add `game_intent_understanding`, `game_experience_iteration`, and `game_production_read` with exact candidate IDs, complete planned-family records, current empty projection sets, and atomic activation work items. Do not expose Commit／Approval／Promotion through Provider projections.

- [ ] **Step 4: Bind Runtime Package and Scheduling**

Use the Minimum Core exact Definition in headless boot target semantics without treating it as Product First Playable or target Qualification.

- [ ] **Step 5: Verify current emptiness**

Run:

```powershell
rg -n "MinimumExecutableCoreDefinitionV1|game_intent_understanding|game_experience_iteration|game_production_read|current.*exact `\[\]`" docs/architecture/02-foundation docs/architecture/04-runtime docs/architecture/appendices/executable-contracts-operation-planning-catalog.md
```

Expected: target contracts are present and every current Operation set remains empty.

- [ ] **Step 6: Commit**

```powershell
git add -- docs/architecture/02-foundation/core-architecture.md docs/architecture/02-foundation/executable-contracts.md docs/architecture/appendices/executable-contracts-operation-planning-catalog.md docs/architecture/04-runtime/runtime-package.md docs/architecture/04-runtime/scheduling-lifetime.md
git commit -m "docs: define minimum executable core closure"
```

### Task 7: Repair references and close the Architecture register

**Files:**
- Modify: `docs/architecture/02-foundation/memory-pointers.md`
- Modify: `docs/architecture/04-runtime/performance-capacity.md`
- Modify: `docs/architecture/appendices/architecture-plan-closure-review.md`
- Modify: `docs/reviews/README.md`

**Interfaces:**
- Consumes: all preceding canonical Owner changes.
- Produces: resolved links, sequential review headings, canonical new closure IDs, and durable audit summary.

- [ ] **Step 1: Repair stale exact references**

Point Memory／Pointers to Toolchain §2.6 Build Policy. Point Performance current Operation state to Executable Contracts §8.2. Confirm the Evidence freshness link was repaired by Task 3.

- [ ] **Step 2: Rebuild the Closure Review tail**

Move the old post-§11 `### 7.9`／`### 7.10` content into sequential top-level follow-up sections or the canonical register. Preserve historical dispositions but make current state explicit.

- [ ] **Step 3: Add canonical closure entries**

Record separate IDs for Game production Owner, Playtest iteration, AI generation claim lanes, First Playable Definition, Evidence freshness, Gameplay Graph ownership, Minimum Executable Core, and reference integrity. Mark target-design closure separately from materialization absent.

- [ ] **Step 4: Update the durable review summary**

Record date, local route, current Architecture document count, changed Owner set, valid-gap count, closure result, external ISO route, exact terminal marker, and retention disposition. Do not claim external consultation or implementation.

- [ ] **Step 5: Verify stale tokens are absent**

Run:

```powershell
rg -n "toolchain-dependencies\.md#2-c23-languageとbuild-policy|§11のexact状態|§10\.1を正本|^### 7\.9 |^### 7\.10 " docs/architecture
```

Expected: no matches.

- [ ] **Step 6: Commit**

```powershell
git add -- docs/architecture/02-foundation/memory-pointers.md docs/architecture/04-runtime/performance-capacity.md docs/architecture/appendices/architecture-plan-closure-review.md docs/reviews/README.md
git commit -m "docs: close reconstruction findings and references"
```

### Task 8: Run the completion audit

**Files:**
- Inspect: every file changed by Tasks 1–7
- Inspect: `docs/architecture/README.md`
- Inspect: `docs/architecture/01-governance/architecture-governance.md`

**Interfaces:**
- Proves: Owner uniqueness, DAG closure, reference integrity, current-state honesty, and Design requirement coverage.

- [ ] **Step 1: Run repository-required checks**

```powershell
git diff --check HEAD~7..HEAD
git status --short
git diff --stat HEAD~7..HEAD
git diff HEAD~7..HEAD -- docs/architecture docs/reviews
```

Expected: no whitespace errors; only intended Architecture／review changes.

- [ ] **Step 2: Validate Owner count, IDs, headers, Index, and DAG**

Run the read-only PowerShell validator used by the Architecture audit. Expected: 64 Owner files, 64 unique IDs, 64 Index entries, required headers present, normative missing 0, cycles 0.

- [ ] **Step 3: Validate links and fragments**

Check all local Markdown paths and fragments, including Japanese headings. Expected: missing path 0, unresolved fragment 0, unbalanced code fence 0.

- [ ] **Step 4: Validate type ownership**

Inventory exact `*V1`／`*RefV1` tokens in changed Owners. For each token, inspect its canonical definition or direct Owner link. Expected: no unowned Game production, Gameplay graph, freshness, or Minimum Core types and no duplicate shape definitions.

- [ ] **Step 5: Validate Design coverage**

Read every section of `docs/superpowers/specs/2026-08-03-ai-native-production-loop-architecture-reconstruction-design.md` and map it to changed Owner evidence. Expected: all 16 sections covered; no requirement supported only by the non-normative Design or Plan.

- [ ] **Step 6: Verify current-state language**

```powershell
rg -n "実装状態:|materialized_operations|active_operations|operational_operations|not_materialized|not_activated" docs/architecture
```

Expected: affected Owners remain `implementation=absent`; all new operation and Core artifacts remain unmaterialized／not activated.

- [ ] **Step 7: Commit any audit-only corrections**

If the audit finds a documentation defect, correct only its canonical Owner and affected direct consumers, rerun Steps 1–6, then commit:

```powershell
git add -- docs/architecture docs/reviews/README.md
git commit -m "docs: verify architecture reconstruction closure"
```

If no correction is necessary, do not create an empty commit.
