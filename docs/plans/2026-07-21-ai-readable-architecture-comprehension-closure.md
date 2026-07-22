# AI-Readable Architecture Comprehension Closure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:executing-plans` to execute this plan task-by-task. Checkboxes record the state of the canonical destinations, not the retired source paths.

**Provenance:** Reconciled from local checkpoint `0e86c0f7d7bc0cabe36ee7871e9ac9b83e925c86` on `codex/ai-readable-architecture-closure`.

**Status:** Complete. Integration commit `10d741c283cc22766227a6267bc849bf97d1aba6` was merged by PR #7 as `40a7b95ee593aeb8df8aedb86aa9d0fce6d20ddf`; post-merge validation and legacy cleanup passed.

**Goal:** Add the GameHost outer loop, architecture authority discovery, external-engine terminology resolution, Architecture Explain projection, and AI architecture-comprehension evaluation to their current canonical owners so that humans and AI can explain execution order and responsibility from exact Evidence instead of inference.

**Architecture:** Architecture Governance owns metadata, relations, generated Index, and bounded explain projection. Editor／Workspace UX owns external-term input resolution; Project State only consumes resolved canonical identities. Runtime Scheduling owns the GameHost loop. AI Verification／Provenance owns comprehension cases, grading, and release gates. No compatibility alias or second authority source is introduced.

**Tech Stack:** Markdown architecture contracts, future MCD／TypeScript Control Plane implementation, PowerShell, ripgrep, Git.

## Global constraints

- Active architecture inventory is generated from valid `ArchitectureMetadataV1`; a fixed document count is not a normative invariant.
- Authority discovery uses `ArchitectureMetadataV1`, the document relation registry, Shared canonical contracts, and the generated Architecture Index.
- External-engine terms are input-only and never become persistent IDs, compatibility APIs, class aliases, or Project paths.
- Architecture Explain is read-only／Disposable, bounded to exact revision and query conditions, and cannot be committed.
- GameHost remains exactly 60 Hz with rational integer tick boundaries, at most four catch-up ticks, T00～T110, and R00～R70.
- Existing Project State, Pause, Runtime phase, MCD, StableId, typed Operation, Receipt, and Owner contracts remain authoritative.

---

### Task 1: Merge authority discovery into the Control Plane

**Files:**
- Modify: `docs/plans/2026-07-22-architecture-evolution-control-plane-design.md`
- Modify: `docs/plans/2026-07-22-architecture-evolution-control-plane-implementation-plan.md`

**Interfaces:**
- Consumes: `ArchitectureMetadataV1`, document relation registry, Shared canonical contracts, generated Architecture Index.
- Produces: unique authority discovery, path／anchor／hash validation, projection allow-list validation, MCD closure, paired-update gates.

- [x] **Step 1: Keep one authority source**

The Control Plane design maps topic ownership, source path／anchor, content hash, allowed projections, MCD refs, and review owner into the existing metadata／registry model. It explicitly rejects a second manifest Schema or compatibility alias.

- [x] **Step 2: Keep inventory generated**

The generated Index reports the active inventory from valid metadata and does not preserve an obsolete fixed-count invariant.

- [x] **Step 3: Add implementation gates**

The implementation plan covers metadata parsing, relation validation, deterministic Index bytes, paired registry updates, negative fixtures, and baseline read-back.

### Task 2: Resolve external-engine terminology at the input boundary

**Files:**
- Modify: `docs/architecture/03-authoring/editor-workspace-ux.md`
- Modify: `docs/architecture/03-authoring/project-state.md`

**Interfaces:**
- Consumes: Requirement context, canonical concept IDs, Project Capability, Target Profile, Owner registry.
- Produces: `ExternalEngineConceptResolutionV1`, `resolved | question_required | unsupported`.

- [x] **Step 1: Define the resolver**

The Editor owner defines request ID, source engine family, source term, context-summary hash, candidate canonical concepts, optional selected concept, closed status, and Evidence refs.

- [x] **Step 2: Preserve ambiguity**

Unity Scene, Unreal Level, and Godot Scene may resolve to Scene, Level, World Streaming／Cell, UI, or Composition concepts. Owner, Save, transition, Streaming, Target, or Project-structure ambiguity requires a question.

- [x] **Step 3: Enforce the ChangeSet boundary**

Project State accepts only the selected canonical concept, exact StableId, typed Operation, and expected revision. Resolver objects, external paths, hierarchy indexes, unresolved candidates, and unsupported approximations cannot be committed.

### Task 3: Define the GameHost outer loop

**Files:**
- Modify: `docs/architecture/04-runtime/scheduling-lifetime.md`

**Interfaces:**
- Consumes: Application lifecycle, `PausePolicyV1`, T00～T110, R00～R70, `InputSnapshot`, `RenderSnapshot`.
- Produces: `GameHostLoopStateV1`, rational 60 Hz clock, catch-up／input／pause／surface／fault rules.

- [x] **Step 1: Define state and rational clock**

The canonical state separates Gameplay and Debug pause. Tick duration is computed from adjacent integer-rational 60 Hz boundaries; floating point and repeated truncated nanoseconds are prohibited.

- [x] **Step 2: Fix the outer-loop order**

The order is lifecycle latch, one clock sample, transition, one device sample, zero-to-four complete ticks, latest complete snapshot, presentation interpolation, zero-or-one render, retire／wait.

- [x] **Step 3: Close catch-up and input behavior**

Catch-up records dropped wall time and never duplicates press／release edges, text, or gestures across ticks. Replay records sample-to-tick assignment.

- [x] **Step 4: Close pause, lifecycle, surface, headless, and fault behavior**

Debug pause becomes effective only after T110 and a single step completes exactly one T00～T110 sequence. Surface loss preserves the World, Suspend stops work, headless continues authoritative ticks without render, and a faulted tick is never published or rendered.

### Task 4: Generate bounded Architecture Explain Evidence

**Files:**
- Modify: `docs/plans/2026-07-22-architecture-evolution-control-plane-design.md`
- Modify: `docs/plans/2026-07-22-architecture-evolution-control-plane-implementation-plan.md`
- Modify: `docs/architecture/03-authoring/project-state.md`

**Interfaces:**
- Consumes: metadata／relation／Product／Contract registries, committed World／Level／Streaming Source, Target Profile, exact Project revision.
- Produces: `ArchitectureExplainProjectionV1`, `authoring.explain_architecture`.

- [x] **Step 1: Define the projection and bounds**

The Control Plane owns one projection schema with at most 256 entries per category, 1,024 dependency edges, 1,024 Evidence refs, 128 omitted ranges, and 2 MiB canonical encoding.

- [x] **Step 2: Define deterministic generation**

Every entry carries canonical concept ID, Owner, optional phase／lifetime, Source StableId, Source content hash, and Evidence refs. Same input and query produce identical bytes.

- [x] **Step 3: Define omission and continuation**

Truncation is explicit; signed continuation is bound to Project revision, scope, field mask, Target Profile, and Source closure. Stale or mismatched continuation fails closed.

- [x] **Step 4: Keep Project State read-only**

The query requires scope, non-empty field mask, optional Target Profile, and exact revision. Mutation must respecify canonical StableId, typed Operation, and expected revision through the normal Gateway.

### Task 5: Evaluate architecture comprehension independently

**Files:**
- Modify: `docs/architecture/01-governance/ai-verification-provenance.md`

**Interfaces:**
- Consumes: authority metadata closure, `ExternalEngineConceptResolutionV1`, `ArchitectureExplainProjectionV1`, Runtime／World／Game System contracts.
- Produces: `architecture_comprehension`, `ArchitectureComprehensionCaseV1`, `ArchitectureComprehensionFixtureV1`.

- [x] **Step 1: Register the suite and case schema**

Each case records risk, exact revision, question, input projection hash, expected concepts／owners／phase-or-lifetime, required Evidence, forbidden claims, and question requirement.

- [x] **Step 2: Fix the 240-case corpus**

The corpus has 80 direct canonical-term cases, 80 external-engine-term cases, and 80 ambiguous／conflicting／stale-Evidence cases.

- [x] **Step 3: Fix grader and hard gates**

Code-based comparison requires 100% Blocking／High concept, Owner, and phase／lifetime accuracy; fabricated IDs／phases, wrong authority, unsupported claims, question bypass, and stale／omitted Evidence acceptance are zero. Three clean runs use the worst result.

- [x] **Step 4: Separate infrastructure failure**

Unexpected materialization, metadata／projection／contract hash mismatch, or missing cases are infrastructure errors and never dilute the pass／fail denominator.

### Task 6: Close cross-document validation and provenance

**Files:**
- Verify: all Task 1～5 destination files and affected plans.
- Modify: this plan with final validation and merge evidence.

- [x] **Step 1: Verify canonical ownership and consumer counts**

Require one canonical definition per new contract, no second authority Schema, and only the documented consumer references.

- [x] **Step 2: Verify plan freshness**

Reject legacy active-document paths, fixed inventory fields, obsolete execution-mode statements, stale Owner names, and unchecked completed work.

- [x] **Step 3: Run Markdown and repository hygiene checks**

Require strict UTF-8, no BOM／CR／trailing whitespace, balanced fences, resolvable relative links, `git diff --check`, no secret-like values, and no local workspace paths in destination content.

Completed: 57 Markdown files, format／link errors 0, semantic-definition violations 0, PR #3 identifier gaps 0, and secret／local-path matches 0. No build or automated-test manifest exists; validation is documentation-only.

- [x] **Step 4: Record integration evidence**

After validation, record the integration commit／PR and keep this plan as audit evidence. Cleanup of the source branch and worktree occurs only after merge and post-merge read-back.

Completed: [PR #7](https://github.com/y2ikgm89/miraikanai-engine/pull/7) merged expected head `dc00ff381ed1b6c8b1ffeeeb6fcb218a150bd6b4` as `40a7b95ee593aeb8df8aedb86aa9d0fce6d20ddf`. Local／remote `main` matched, the full validation passed again, both owned worktrees were removed, and all legacy／integration branch refs were deleted.
