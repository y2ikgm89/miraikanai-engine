# Architecture Document Restructure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the 47-file dated Engine review set with 42 focused canonical specifications, separate the Codex configuration document, remove completed plans, and prove that the new document graph has no legacy-path, ownership, link, duplication, or unresolved-decision defects.

**Architecture:** The approved Decision at `docs/architecture/decisions/2026-07-21-document-system-restructure.md` is the normative migration map. Work proceeds by responsibility group so every intermediate commit has a reviewable source-to-target boundary; semantic merges use `apply_patch`, pure renames use `git mv`, and mechanical link rewrites run only after all target paths exist.

**Tech Stack:** Markdown, Git, PowerShell 7-compatible validation commands, Context7, official vendor and standards documentation.

## Global Constraints

- Preserve all pre-existing user changes; abort a task if its source files differ unexpectedly from `HEAD`.
- Do not add production dependencies or generated compatibility documents.
- Do not retain dated active filenames, redirect files, old-path aliases, or `docs/superpowers/specs/` compatibility stubs.
- Every active specification must have exactly one H1 and the six required header fields defined by Decision section 7.1.
- Every restructured specification starts with `状態: review`; implementation maturity remains a separate closed value.
- A type, schema, operation, identifier, shared value, phase, budget, approval rule, or evidence envelope has one normative owner only.
- External versions, hashes, licenses, and acquisition URLs are normative only in `docs/architecture/02-foundation/toolchain-dependencies.md`.
- External API behavior must be checked through Context7 first; when Context7 lacks the exact product or version, use the current official primary source and record the fallback.
- `docs/architecture/README.md` is non-normative and contains only navigation, reading order, status, and ownership links.
- No active specification may exceed 1,000 lines.
- The final state contains 42 Engine specifications, one Engine Index, the approved Decision, and `docs/developer-tools/codex/configuration.md`; both completed implementation plans are absent from the worktree.
- All semantic edits use `apply_patch`; `git mv` is allowed for pure path moves, and a scripted replacement is allowed only for a reviewed old-path-to-new-path table.

## Complete Source Manifest

Every pre-existing document is assigned to exactly one task below. A row with multiple targets requires semantic allocation rather than a mechanical move.

| Source | Target／Disposition | Task |
|---|---|---:|
| `docs/superpowers/specs/2026-07-18-ai-native-game-engine-authoring-design.md` | `docs/architecture/00-product/product-plan.md` | 1 |
| `docs/superpowers/specs/2026-07-19-2d-3d-capability-plan.md` | `docs/architecture/00-product/product-plan.md`＋named Subsystem owners | 1 |
| `docs/superpowers/specs/2026-07-20-ai-readable-capability-portfolio-productization-roadmap-design.md` | `docs/architecture/00-product/product-plan.md` | 1 |
| `docs/superpowers/specs/2026-07-19-ai-engine-development-governance-design.md` | `docs/architecture/01-governance/ai-security-approval.md` | 1 |
| `docs/superpowers/specs/2026-07-21-immutable-engine-beginner-ai-approval-design.md` | `docs/architecture/01-governance/ai-security-approval.md` | 1 |
| `docs/superpowers/specs/2026-07-19-ai-verification-evaluation-provenance-design.md` | `docs/architecture/01-governance/ai-verification-provenance.md` | 1 |
| `docs/superpowers/specs/2026-07-19-engine-foundation-architecture-design.md` | `docs/architecture/02-foundation/core-architecture.md`＋`toolchain-dependencies.md` | 2 |
| `docs/superpowers/specs/2026-07-19-executable-contract-schema-codegen-design.md` | `docs/architecture/02-foundation/executable-contracts.md` | 2 |
| `docs/superpowers/specs/2026-07-20-ai-readable-engine-naming-convention-design.md` | `docs/architecture/02-foundation/naming-project-layout.md` | 2 |
| `docs/superpowers/specs/2026-07-20-ai-readable-game-project-layout-naming-design.md` | `docs/architecture/02-foundation/naming-project-layout.md` | 2 |
| `docs/superpowers/specs/2026-07-20-cpp23-modules-import-std-transition-design.md` | `docs/architecture/02-foundation/cpp23-modules.md` | 2 |
| `docs/superpowers/specs/2026-07-20-ai-readable-math-core-utilities-architecture-design.md` | `docs/architecture/02-foundation/math-core.md` | 2 |
| `docs/superpowers/specs/2026-07-20-ai-readable-memory-pointer-architecture-design.md` | `docs/architecture/02-foundation/memory-pointers.md` | 2 |
| `docs/superpowers/specs/2026-07-19-authoring-model-project-state-design.md` | `docs/architecture/03-authoring/project-state.md` | 3 |
| `docs/superpowers/specs/2026-07-19-asset-pipeline-content-packaging-design.md` | `docs/architecture/03-authoring/asset-lifecycle.md` | 3 |
| `docs/superpowers/specs/2026-07-20-asset-import-ai-authoring-editor-ux-design.md` | `docs/architecture/03-authoring/asset-lifecycle.md` | 3 |
| `docs/superpowers/specs/2026-07-20-editor-ui-framework-architecture-design.md` | `docs/architecture/03-authoring/editor-ui-framework.md` | 3 |
| `docs/superpowers/specs/2026-07-19-editor-workspace-ux-design.md` | `docs/architecture/03-authoring/editor-workspace-ux.md` | 3 |
| `docs/superpowers/specs/2026-07-19-cpp-structured-game-data-design.md` | `docs/architecture/03-authoring/gameplay-programming-model.md` | 3 |
| `docs/superpowers/specs/2026-07-20-game-system-ai-codegen-architecture-design.md` | `docs/architecture/03-authoring/gameplay-programming-model.md` | 3 |
| `docs/superpowers/specs/2026-07-19-native-game-module-architecture-design.md` | `docs/architecture/03-authoring/native-game-module.md` | 3 |
| `docs/superpowers/specs/2026-07-19-runtime-integration-lifetime-performance-design.md` | `docs/architecture/04-runtime/scheduling-lifetime.md`＋`performance-capacity.md` | 4 |
| `docs/superpowers/specs/2026-07-20-scale-resilient-canonical-architecture-design.md` | `docs/architecture/04-runtime/performance-capacity.md`＋named Subsystem owners | 4 |
| `docs/superpowers/specs/2026-07-20-ai-readable-debugging-observability-replay-architecture-design.md` | `docs/architecture/04-runtime/debugging-observability-replay.md` | 4 |
| `docs/superpowers/specs/2026-07-19-collision-collider-architecture-design.md` | `docs/architecture/05-simulation/collision.md` | 5 |
| `docs/superpowers/specs/2026-07-20-physics-engine-architecture-design.md` | `docs/architecture/05-simulation/physics.md` | 5 |
| `docs/superpowers/specs/2026-07-20-physics-ai-semantic-capability-catalog-design.md` | `docs/architecture/05-simulation/physics.md` | 5 |
| `docs/superpowers/specs/2026-07-20-navigation-platform-architecture-design.md` | `docs/architecture/05-simulation/navigation.md` | 5 |
| `docs/superpowers/specs/2026-07-19-physics-navigation-animation-architecture-design.md` | `docs/architecture/05-simulation/animation.md`＋`docs/architecture/04-runtime/scheduling-lifetime.md` | 5 |
| `docs/superpowers/specs/2026-07-19-rendering-render-graph-architecture-design.md` | `docs/architecture/06-rendering/render-graph.md` | 6 |
| `docs/superpowers/specs/2026-07-20-material-visual-style-ai-authoring-architecture-design.md` | `docs/architecture/06-rendering/materials.md` | 6 |
| `docs/superpowers/specs/2026-07-20-lighting-ai-authoring-architecture-design.md` | `docs/architecture/06-rendering/lighting.md` | 6 |
| `docs/superpowers/specs/2026-07-20-post-process-ai-authoring-architecture-design.md` | `docs/architecture/06-rendering/post-processing.md` | 6 |
| `docs/superpowers/specs/2026-07-20-ai-readable-lod-architecture-design.md` | `docs/architecture/06-rendering/lod.md` | 6 |
| `docs/superpowers/specs/2026-07-20-world-level-map-ai-authoring-architecture-design.md` | `docs/architecture/06-rendering/world.md` | 6 |
| `docs/superpowers/specs/2026-07-20-particle-vfx-architecture-design.md` | `docs/architecture/06-rendering/vfx-authoring.md`＋`vfx-runtime.md` | 7 |
| `docs/superpowers/specs/2026-07-20-camera-platform-ai-authoring-virtual-production-architecture-design.md` | `docs/architecture/06-rendering/camera.md`＋Product `not_activated` entry | 7 |
| `docs/superpowers/specs/2026-07-20-environment-platform-ai-authoring-architecture-design.md` | `docs/architecture/06-rendering/environment-surfaces.md` | 7 |
| `docs/superpowers/specs/2026-07-20-water-surface-platform-architecture-design.md` | `docs/architecture/06-rendering/environment-surfaces.md` | 7 |
| `docs/superpowers/specs/2026-07-20-weather-snow-surface-architecture-design.md` | `docs/architecture/06-rendering/environment-surfaces.md` | 7 |
| `docs/superpowers/specs/2026-07-19-windows-platform-distribution-design.md` | `docs/architecture/07-platform/windows.md` | 8 |
| `docs/superpowers/specs/2026-07-19-mobile-platform-architecture-design.md` | `docs/architecture/07-platform/mobile-common.md`＋`android.md`＋`apple.md` | 8 |
| `docs/superpowers/specs/2026-07-19-input-action-device-architecture-design.md` | `docs/architecture/07-platform/input.md` | 8 |
| `docs/superpowers/specs/2026-07-19-audio-mixer-spatial-architecture-design.md` | `docs/architecture/07-platform/audio.md` | 8 |
| `docs/superpowers/specs/2026-07-19-ui-text-localization-accessibility-design.md` | `docs/architecture/07-platform/ui-text-localization-accessibility.md` | 8 |
| `docs/superpowers/specs/2026-07-19-domain-pack-future-capability-roadmap.md` | `docs/architecture/08-domain-packs/domain-pack-contract.md`＋Product roadmap | 9 |
| `docs/superpowers/specs/2026-07-20-ai-readable-shooter-gameplay-architecture-design.md` | `docs/architecture/08-domain-packs/shooter.md` | 9 |
| `docs/superpowers/specs/2026-07-18-codex-config-optimization-design.md` | `docs/developer-tools/codex/configuration.md` | 9 |
| `docs/superpowers/specs/README.md` | Replace with `docs/architecture/README.md` | 9 |
| `docs/superpowers/plans/2026-07-18-codex-config-optimization.md` | Remove after `open=0` verification | 11 |
| `docs/superpowers/plans/2026-07-20-existing-documentation-naming-directory-unification.md` | Remove after `open=0` verification | 11 |

---

### Task 1: Record the preflight and create the Product and Governance canon

**Files:**
- Create: `docs/architecture/00-product/product-plan.md`
- Create: `docs/architecture/01-governance/ai-security-approval.md`
- Create: `docs/architecture/01-governance/ai-verification-provenance.md`
- Consume and remove after coverage review:
  - `docs/superpowers/specs/2026-07-18-ai-native-game-engine-authoring-design.md`
  - `docs/superpowers/specs/2026-07-19-2d-3d-capability-plan.md`
  - `docs/superpowers/specs/2026-07-20-ai-readable-capability-portfolio-productization-roadmap-design.md`
  - `docs/superpowers/specs/2026-07-19-ai-engine-development-governance-design.md`
  - `docs/superpowers/specs/2026-07-21-immutable-engine-beginner-ai-approval-design.md`
  - `docs/superpowers/specs/2026-07-19-ai-verification-evaluation-provenance-design.md`

**Interfaces:**
- Consumes: Decision sections 5–8 and all H2 sections of the six source documents.
- Produces: the only Product roadmap, the only AI authorization/approval contract, and the only verification/provenance contract used by every later task.

- [ ] **Step 1: Verify the preflight state**

Run:

```powershell
$specs = Get-ChildItem docs/superpowers/specs -File -Filter *.md
$plans = Get-ChildItem docs/superpowers/plans -File -Filter *.md
"specs=$($specs.Count) plans=$($plans.Count)"
git status --short --branch
```

Expected: `specs=49 plans=3` because the active implementation plan now exists; Git status lists only this plan before it is committed.

- [ ] **Step 2: Verify the target files do not yet exist**

Run:

```powershell
@(
  'docs/architecture/00-product/product-plan.md',
  'docs/architecture/01-governance/ai-security-approval.md',
  'docs/architecture/01-governance/ai-verification-provenance.md'
) | ForEach-Object { "$_=$([bool](Test-Path -LiteralPath $_))" }
```

Expected: all three values are `False`.

- [ ] **Step 3: Write `product-plan.md`**

Create one document with this exact section responsibility:

1. Product intent and AI-native authoring outcome from Master sections 1–3.
2. Non-negotiable clean-implementation principles and explicit non-goals.
3. C0–C3 maturity vocabulary and `not_activated | candidate_locked | qualified | production` transition meaning.
4. 2D／3D Capability portfolio as a compact matrix; do not include Subsystem field lists, vendor defaults, or runtime budgets.
5. MVP scope and the single Phase sequence.
6. Promotion, deactivation, and Product completion gates.
7. Future portfolio entries, including Virtual Production as `not_activated`; no C3 implementation schema.
8. Links to each Subsystem owner and primary Product evidence.

Move technical content from the old 2D／3D plan to the relevant later task instead of copying it here. The resulting file must be below 1,000 lines.

- [ ] **Step 4: Write `ai-security-approval.md`**

Use AI Governance as the base and integrate the immutable-engine document into these sections: trust boundaries; risk levels; task authorization; beginner questions and assumptions; Project data versus Project C++／Shader／Native module; sandbox and credential boundaries; Provider／MCP／CLI roles; preview and human approval; activation and promotion; failure and denial; security fixtures. Remove evaluation dataset, grader, provenance-envelope, and trace-grade definitions because those belong to the verification owner.

- [ ] **Step 5: Write `ai-verification-provenance.md`**

Preserve evaluation lifecycle, public／holdout／adversarial datasets, evidence envelopes, provenance, trace grading, release evidence, retention, and failure behavior. Replace copied authorization rules with a link to `ai-security-approval.md`.

- [ ] **Step 6: Check source-heading coverage before deleting sources**

Run this for each of the six source files and review every printed H2 against Steps 3–5:

```powershell
$files = @(
  '2026-07-18-ai-native-game-engine-authoring-design.md',
  '2026-07-19-2d-3d-capability-plan.md',
  '2026-07-20-ai-readable-capability-portfolio-productization-roadmap-design.md',
  '2026-07-19-ai-engine-development-governance-design.md',
  '2026-07-21-immutable-engine-beginner-ai-approval-design.md',
  '2026-07-19-ai-verification-evaluation-provenance-design.md'
)
foreach ($name in $files) {
  "FILE $name"
  Select-String -LiteralPath "docs/superpowers/specs/$name" -Pattern '^## ' -Encoding utf8 | ForEach-Object Line
}
```

Expected: every H2 is represented by a target section, explicitly moved to a named later owner, or rejected as duplicate. No heading is silently dropped.

- [ ] **Step 7: Remove the six consumed sources and run focused checks**

Use `git rm --` with the six exact source paths. Then run:

```powershell
Get-ChildItem docs/architecture/00-product,docs/architecture/01-governance -File -Filter *.md |
  ForEach-Object { "{0}`t{1}" -f (Get-Content -LiteralPath $_.FullName -Encoding utf8).Count,$_.FullName }
git diff --check
```

Expected: 3 target files, each at or below 1,000 lines; `git diff --check` exits 0.

- [ ] **Step 8: Commit the Product and Governance canon**

```powershell
git add -- docs/architecture/00-product docs/architecture/01-governance docs/superpowers/specs
git commit -m "docs: consolidate product and governance specifications"
```

### Task 2: Split Foundation and consolidate naming

**Files:**
- Create: `docs/architecture/02-foundation/core-architecture.md`
- Create: `docs/architecture/02-foundation/toolchain-dependencies.md`
- Move: `2026-07-19-executable-contract-schema-codegen-design.md` → `executable-contracts.md`
- Create by merge: `docs/architecture/02-foundation/naming-project-layout.md`
- Move: `2026-07-20-cpp23-modules-import-std-transition-design.md` → `cpp23-modules.md`
- Move: `2026-07-20-ai-readable-math-core-utilities-architecture-design.md` → `math-core.md`
- Move: `2026-07-20-ai-readable-memory-pointer-architecture-design.md` → `memory-pointers.md`
- Consume: `2026-07-19-engine-foundation-architecture-design.md`, both naming/layout sources.

**Interfaces:**
- Consumes: Product and Governance links from Task 1.
- Produces: the sole external-version owner and common naming/path rules needed by every later task.

- [ ] **Step 1: Split the Foundation H2 ownership**

Write `core-architecture.md` from old Foundation sections 1, 3–5, 8–9, 12, 14, 16–18, and 20. Move sections 2.1, 10 tool-version facts, 13, 19, and all artifact/version/hash/license tables to `toolchain-dependencies.md`. Sections 6–7 become links to `memory-pointers.md` and Runtime performance owners; section 11 becomes a link to `naming-project-layout.md`; section 15 becomes a link to Runtime observability/performance.

- [ ] **Step 2: Centralize all fixed external facts**

`toolchain-dependencies.md` must own the exact Windows, Android, Apple, CMake, Ninja, compiler, SDK, Node.js, TypeScript, OpenAI SDK, graphics, physics, navigation, audio, text, animation, image, license, hash, and update-Gate tables. Record Context7 IDs for CMake, Box2D, Jolt, and Recast; record Android as an official-document fallback. No later target may repeat these versions.

- [ ] **Step 3: Merge naming and project layout**

Write one document with: vocabulary and acronym rules; public type／operation／diagnostic naming; file and directory naming; Engine versus Game Project roots; Source／Derived／Intermediate／Package placement; generated-file policy; module／namespace／target mapping; invalid examples; lint and migration gates. Fix the two source-level `# 禁止` headings to ordinary subsections.

- [ ] **Step 4: Move the four focused specifications**

Use exact `git mv` commands:

```powershell
git mv -- docs/superpowers/specs/2026-07-19-executable-contract-schema-codegen-design.md docs/architecture/02-foundation/executable-contracts.md
git mv -- docs/superpowers/specs/2026-07-20-cpp23-modules-import-std-transition-design.md docs/architecture/02-foundation/cpp23-modules.md
git mv -- docs/superpowers/specs/2026-07-20-ai-readable-math-core-utilities-architecture-design.md docs/architecture/02-foundation/math-core.md
git mv -- docs/superpowers/specs/2026-07-20-ai-readable-memory-pointer-architecture-design.md docs/architecture/02-foundation/memory-pointers.md
```

Replace their headers and remove copied Toolchain values in favor of `toolchain-dependencies.md` links.

- [ ] **Step 5: Remove consumed sources and verify Foundation ownership**

After H2 coverage review, remove the old Foundation and both naming sources. Run:

```powershell
rg -n "CMake 4\.4\.0|Ninja 1\.13\.2|Node\.js 24\.18\.0|Box2D 3\.1\.1|Jolt(?: Physics)? 5\.6\.0" docs/architecture --glob '*.md'
```

Expected: normative version tables occur only in `toolchain-dependencies.md`; the Decision may contain historical audit evidence. Other specifications may name a dependency without restating the pinned version.

- [ ] **Step 6: Commit Foundation**

```powershell
git add -- docs/architecture/02-foundation docs/superpowers/specs
git commit -m "docs: establish canonical foundation specifications"
```

### Task 3: Consolidate Authoring contracts

**Files:**
- Move: `authoring-model-project-state` → `docs/architecture/03-authoring/project-state.md`
- Merge: Asset Pipeline＋Asset Import → `asset-lifecycle.md`
- Move: Editor UI Framework → `editor-ui-framework.md`
- Move: Editor Workspace UX → `editor-workspace-ux.md`
- Merge: C++ Structured Game Data＋Game System Codegen → `gameplay-programming-model.md`
- Move: Native Game Module → `native-game-module.md`

**Interfaces:**
- Consumes: Foundation contract, naming, toolchain, and Governance links.
- Produces: transaction, asset, editor, gameplay, and native-extension owners used by Runtime and Domain Packs.

- [ ] **Step 1: Move the four one-to-one files**

Use `git mv` for Project State, Editor UI Framework, Editor Workspace UX, and Native Game Module. Add the six-field header and replace old-path references with target owner links only after the target exists.

- [ ] **Step 2: Write `asset-lifecycle.md`**

Use sections in this order: source/import identity; Import Profile and detection; preview and conversion report; reimport and dependency invalidation; Derived/Cooked artifacts; packaging and content addressing; hot reload/promotion; Editor and AI operations; platform specialization; diagnostics; security; qualification. Resolve the current conflict by making import/reimport owned here and package assembly a later stage of the same lifecycle.

- [ ] **Step 3: Write `gameplay-programming-model.md`**

Use sections in this order: decision matrix for structured data versus C++; `GameplayDefinition`; `GameSystemSpecV1`; state ownership and typed ports; project-defined systems; AI plan and code generation; generated bundle; validation and build; testing and promotion; forbidden fixed class hierarchy and unrestricted scripting runtime. Keep Native ABI/build details exclusively in `native-game-module.md`.

- [ ] **Step 4: Remove the four merged sources and verify line counts**

Remove both Asset sources and both gameplay-model sources only after every H2 has a named target section. Run line-count and H1 checks for all six Authoring targets.

- [ ] **Step 5: Commit Authoring**

```powershell
git add -- docs/architecture/03-authoring docs/superpowers/specs
git commit -m "docs: consolidate authoring and gameplay contracts"
```

### Task 4: Split Runtime and absorb the Scale overlay

**Files:**
- Create: `docs/architecture/04-runtime/scheduling-lifetime.md`
- Create: `docs/architecture/04-runtime/performance-capacity.md`
- Move and reduce: `docs/architecture/04-runtime/debugging-observability-replay.md`
- Consume: Runtime, Scale, and Debugging sources.

**Interfaces:**
- Consumes: Product phases, Governance evidence links, Foundation memory/toolchain, and Authoring state rules.
- Produces: the single runtime phase/tick owner, shared capacity owner, and debug/replay owner.

- [ ] **Step 1: Write `scheduling-lifetime.md`**

Move Runtime sections 1–10, 13 ordering semantics, 15 lifecycle recovery, 17 scheduling tests, 18 application rules, and 19 Phase-0 runtime artifacts. Preserve the exact 60 Hz phase sequence once. Add cross-subsystem Physics／Navigation／Animation order from the integration source sections 4, 6, 7, 12, and 16 without copying subsystem schemas.

- [ ] **Step 2: Write `performance-capacity.md`**

Move Runtime sections 11–14 and measurement portions of 16–17. Integrate Scale sections 5–15 and 17–20 as one `ProjectScaleEnvelopeV1`, resolver/backpressure, measurement, transition, and qualification model. Return World cell fields to `world.md`, LOD fields to `lod.md`, and Authoring/build fields to their owners.

- [ ] **Step 3: Reduce `debugging-observability-replay.md` below 1,000 lines**

Keep debugger surfaces, bounded telemetry/query operations, deterministic capture/replay, crash/fault evidence, AI diagnosis, Editor UX, and qualification. Remove copied runtime budgets, authorization definitions, and provenance-envelope fields; link to the three canonical owners.

- [ ] **Step 4: Remove the three sources and verify unique runtime ownership**

Run searches for the fixed tick sequence, shared memory budgets, and `ProjectScaleEnvelopeV1`. Expected: one definition in its owner and only links elsewhere.

- [ ] **Step 5: Commit Runtime**

```powershell
git add -- docs/architecture/04-runtime docs/superpowers/specs
git commit -m "docs: split runtime scheduling and capacity specifications"
```

### Task 5: Rebuild Simulation boundaries

**Files:**
- Move and reduce: `collision.md`
- Merge: Physics＋Physics Semantic Catalog → `physics.md`
- Move: Navigation → `navigation.md`
- Create: `animation.md`
- Consume: Physics／Navigation／Animation integration source.

**Interfaces:**
- Consumes: Runtime order/capacity and Toolchain dependency ownership.
- Produces: four non-overlapping simulation owners.

- [ ] **Step 1: Move Collision and Navigation**

Use `git mv`. Remove pinned dependency versions, shared runtime budgets, and repeated approval rules. Collision owns geometry/collider/query/filter/event only; Navigation owns Grid2D, Navmesh profile/artifact/query/version/lease only.

- [ ] **Step 2: Merge Physics semantics**

Use Physics as the base. Integrate natural-language Intent, capability discovery, ambiguity questions, semantic resolution, preview, diagnostics, and AI eval from the Semantic Catalog into an `AI semantics` section. Keep vendor APIs private and move exact versions to Toolchain.

- [ ] **Step 3: Write `animation.md`**

Move integration-source sections 9–15 into a standalone contract: Animation Asset and graph; 2D clips; 3D skeleton/skin/clip; pose ownership; event and root motion; IK; retargeting; memory/failure; Editor/AI operations; tests and qualification. Runtime order links to `scheduling-lifetime.md`; Physics and Navigation details link to their owners.

- [ ] **Step 4: Remove consumed sources and reduce oversized files**

Remove Physics, Semantic Catalog, Navigation, Collision, and integration sources after coverage review. Ensure Collision and Physics are below 1,000 lines.

- [ ] **Step 5: Commit Simulation**

```powershell
git add -- docs/architecture/05-simulation docs/superpowers/specs
git commit -m "docs: establish simulation subsystem boundaries"
```

### Task 6: Migrate focused Rendering specifications

**Files:**
- Move: Render Graph → `render-graph.md`
- Move: Material → `materials.md`
- Move: Lighting → `lighting.md`
- Move: Post Process → `post-processing.md`
- Move: LOD → `lod.md`
- Move: World → `world.md`

**Interfaces:**
- Consumes: Runtime scheduling/capacity and Simulation links.
- Produces: six focused rendering owners needed by VFX, Camera, and Environment.

- [ ] **Step 1: Move all six files with `git mv`**

Use exact target paths from Decision section 6.5. Add required headers and update titles.

- [ ] **Step 2: Remove cross-owner copies**

Render Graph owns pass/resource/queue/AA execution. Materials owns shader/material/visual-style authoring. Lighting owns physical light semantics. Post Process owns volume/effect composition. LOD owns representation selection. World owns World/Scene/Level/Cell source and streaming-plan authoring. Replace shared budgets, versions, approval rules, and schema-projection definitions with links.

- [ ] **Step 3: Run focused link and line checks, then commit**

```powershell
git add -- docs/architecture/06-rendering docs/superpowers/specs
git commit -m "docs: migrate canonical rendering specifications"
```

### Task 7: Split VFX, trim Camera, and merge Environment surfaces

**Files:**
- Create: `docs/architecture/06-rendering/vfx-authoring.md`
- Create: `docs/architecture/06-rendering/vfx-runtime.md`
- Create: `docs/architecture/06-rendering/camera.md`
- Create: `docs/architecture/06-rendering/environment-surfaces.md`
- Consume: Particle/VFX, Camera/Virtual Production, Environment, Water, Weather sources.

**Interfaces:**
- Consumes: all focused Rendering owners from Task 6.
- Produces: four specifications with no active Virtual Production schema.

- [ ] **Step 1: Split Particle/VFX by source versus execution**

`vfx-authoring.md` receives old sections 1–9, 18–20, and authoring portions of 21–23. `vfx-runtime.md` receives sections 10–17 and runtime qualification portions of 21–23. Define the compiled artifact boundary once and link across the two documents.

- [ ] **Step 2: Trim Camera to active C1/C2 scope**

Keep old Camera sections 1–11 and shared parts of 15–23. Remove sections 12–14 as active implementation contracts. Add one Product link stating Recording／Timecode／Genlock／Virtual Production are `not_activated`; do not retain their schemas, budgets, or APIs.

- [ ] **Step 3: Merge Environment, Water, and Weather**

Use one source model with sections: environment composition; sky/fog/wind; water surface/body profile; weather state; snow/wetness surface response; renderer/material interfaces; runtime compilation; AI/Editor operations; budgets and fallback; diagnostics; qualification. Deduplicate common Operation, preview, lifecycle, and surface-state rules.

- [ ] **Step 4: Remove five consumed sources and verify all four targets are below 1,000 lines**

Run H1, heading-level, line-count, and exact-paragraph duplicate checks for `docs/architecture/06-rendering/`.

- [ ] **Step 5: Commit composite Rendering work**

```powershell
git add -- docs/architecture/06-rendering docs/superpowers/specs
git commit -m "docs: split vfx and unify environment specifications"
```

### Task 8: Split Mobile and migrate Platform specifications

**Files:**
- Move: Windows → `windows.md`
- Create: `mobile-common.md`, `android.md`, `apple.md`
- Move: Input → `input.md`
- Move: Audio → `audio.md`
- Move: UI/Text/Localization/Accessibility → `ui-text-localization-accessibility.md`
- Consume: Mobile source.

**Interfaces:**
- Consumes: Toolchain versions, Runtime budgets, Asset lifecycle, and common Renderer contracts.
- Produces: seven platform owners.

- [ ] **Step 1: Move four focused Platform files**

Use `git mv` for Windows, Input, Audio, and UI/Text. Remove copied tool versions and shared budgets.

- [ ] **Step 2: Write `mobile-common.md`**

Own shared target-profile schema, lifecycle, save/recovery, graphics abstraction boundary, asset delivery contract, touch/safe-area concepts, memory/thermal policy, device workflow, privacy model, and common qualification. It must not contain Gradle, NDK, Xcode, signing, or Store-specific values.

- [ ] **Step 3: Write `android.md`**

Own Android profile, Gradle/AGP/NDK build mapping by reference to Toolchain values, GameActivity/controller/frame-pacing adapters, Vulkan/asset packaging, lifecycle, Play delivery, permissions/privacy, 16 KiB page compatibility, device tests, and release gate.

- [ ] **Step 4: Write `apple.md`**

Own Apple profile, Xcode build mapping by reference to Toolchain values, C ABI/Objective-C++ bridge, Metal/asset packaging, lifecycle, signing/upload separation, Xcode Cloud, privacy manifest, TestFlight/App Store, device tests, and release gate.

- [ ] **Step 5: Remove Mobile source and verify common/platform separation**

Search `mobile-common.md` for `AGP|NDK|Gradle|Xcode|Provisioning|App Store Connect`; expected: only links in the non-owner scope, not normative values or procedures.

- [ ] **Step 6: Commit Platform**

```powershell
git add -- docs/architecture/07-platform docs/superpowers/specs
git commit -m "docs: split mobile and migrate platform specifications"
```

### Task 9: Migrate Domain Packs, Codex material, and build the Index

**Files:**
- Create: `docs/architecture/08-domain-packs/domain-pack-contract.md`
- Move and reduce: Shooter → `docs/architecture/08-domain-packs/shooter.md`
- Move: Codex configuration → `docs/developer-tools/codex/configuration.md`
- Create: `docs/architecture/README.md`
- Consume: Domain Pack source, Shooter source, old Index, Codex source.

**Interfaces:**
- Consumes: every canonical owner path.
- Produces: complete navigation and the final separation between Engine and developer tooling.

- [ ] **Step 1: Write the Domain Pack contract**

Move pack identity, manifest, capability declaration, activation, dependency, qualification, update, and removal rules from the old Domain Pack source. Keep future Capability roadmap entries exclusively in Product Plan.

- [ ] **Step 2: Move and reduce Shooter below 1,000 lines**

Keep it as a reference composition of Gameplay, Input, Collision, Physics, Camera, UI, Audio, Asset, and Runtime contracts. Remove copied subsystem schemas and budgets; link to owners.

- [ ] **Step 3: Move Codex configuration outside Engine Architecture**

Use:

```powershell
New-Item -ItemType Directory -Force -Path docs/developer-tools/codex | Out-Null
git mv -- docs/superpowers/specs/2026-07-18-codex-config-optimization-design.md docs/developer-tools/codex/configuration.md
```

Update its title and status but do not add it to the Engine Index.

- [ ] **Step 4: Write the non-normative Index**

`docs/architecture/README.md` contains: purpose and non-normative warning; required reading order; one row per 42 specification with status and ownership summary; Product-to-Subsystem navigation; external developer-tool separation; document-change rules. It contains no field definitions, fixed values, phase tasks, budgets, or DoD copies.

- [ ] **Step 5: Remove old Domain Pack source and old Index, then verify 42 target specs**

Run:

```powershell
$specs = Get-ChildItem docs/architecture -Recurse -File -Filter *.md |
  Where-Object { $_.FullName -notmatch '[\\/]decisions[\\/]' -and $_.Name -ne 'README.md' }
"engine_specs=$($specs.Count)"
```

Expected: `engine_specs=42`.

- [ ] **Step 6: Commit Domain Packs, Codex separation, and Index**

```powershell
git add -- docs/architecture docs/developer-tools docs/superpowers/specs
git commit -m "docs: finalize canonical architecture document graph"
```

### Task 10: Rewrite all links and required headers mechanically

**Files:**
- Modify: every `docs/architecture/**/*.md`
- Modify only if needed: `docs/developer-tools/codex/configuration.md`

**Interfaces:**
- Consumes: all target paths created in Tasks 1–9.
- Produces: a graph with no legacy path or header defects.

- [ ] **Step 1: Build an explicit old-path-to-new-path replacement table**

Use Decision section 6 as the table. For one-to-many sources, do not mechanical-replace; patch each reference to the exact owner based on the surrounding concept. Mechanical replacement is allowed only for one-to-one moves.

- [ ] **Step 2: Apply and inspect one-to-one replacements**

After the script runs, inspect `git diff --word-diff=plain` for every changed file. Reject replacements inside historical Decision text and external URLs.

- [ ] **Step 3: Add or normalize the six required header fields**

Every active specification must identify its unique Document ID, `review` state, normative scope, non-normative scope, relative dependencies, and evidence verification date. Decision and developer-tool documents are excluded from the active-spec header schema.

- [ ] **Step 4: Verify there are no legacy references**

Run:

```powershell
$legacy = rg -n "docs/superpowers/(specs|plans)|\./2026-07-(18|19|20|21)-" docs/architecture --glob '*.md' --glob '!decisions/**'
if ($LASTEXITCODE -eq 1) { 'legacy_refs=0'; exit 0 }
$legacy
exit 1
```

Expected: `legacy_refs=0`.

- [ ] **Step 5: Commit link and header normalization**

```powershell
git add -- docs/architecture docs/developer-tools
git commit -m "docs: normalize canonical links and document metadata"
```

### Task 11: Remove completed plans and empty legacy directories

**Files:**
- Remove: `docs/superpowers/plans/2026-07-18-codex-config-optimization.md`
- Remove: `docs/superpowers/plans/2026-07-20-existing-documentation-naming-directory-unification.md`
- Remove: this implementation plan after all its checkboxes are complete
- Remove empty `docs/superpowers/specs/` and `docs/superpowers/plans/` directories implicitly through Git.

**Interfaces:**
- Consumes: completed and committed Tasks 1–10.
- Produces: no active legacy plan/spec directory in the final tree.

- [ ] **Step 1: Verify the two historical plans are complete**

Run:

```powershell
foreach ($p in @(
  'docs/superpowers/plans/2026-07-18-codex-config-optimization.md',
  'docs/superpowers/plans/2026-07-20-existing-documentation-naming-directory-unification.md'
)) {
  $open = @(Select-String -LiteralPath $p -Pattern '^- \[ \]' -Encoding utf8).Count
  "$p open=$open"
}
```

Expected: both report `open=0`.

- [ ] **Step 2: Remove the two completed plans**

Use `git rm --` with the two exact paths. Do not archive or create replacement summaries.

- [ ] **Step 3: Complete this plan's checkboxes, then remove it**

The implementation commit history and the approved Decision are the permanent record. Remove `docs/superpowers/plans/2026-07-21-architecture-document-restructure.md` only after Task 12 passes.

### Task 12: Run the full completion audit

**Files:**
- Verify: all final Markdown files and Git state.
- Modify: only files that fail a listed Gate.

**Interfaces:**
- Consumes: complete final tree.
- Produces: evidence that every Decision completion criterion is satisfied.

- [ ] **Step 1: Verify inventory, paths, headers, and headings**

Run a PowerShell audit that asserts: 42 Engine specs; one Index; one Decision; one separated Codex document; no active legacy specs/plans; one H1 per file; no skipped heading level; all six header fields on active specs; unique Document IDs; no file above 1,000 lines.

Expected: every assertion passes and the process exits 0.

- [ ] **Step 2: Verify all relative files and Markdown anchors**

Extract Markdown links, resolve each relative target against its containing directory, decode anchor fragments, and compare normalized anchors with the target's headings.

Expected: `broken_files=0 broken_anchors=0`.

- [ ] **Step 3: Verify source-heading migration coverage**

Read each old source from `git show 1308772^:<path>` or its last pre-removal commit, enumerate every H2, and compare it with the Task 1–9 allocation record and final target content.

Expected: all 47 Engine source documents and all source H2 sections are classified as migrated, moved to a named owner, or rejected as duplicate/non-normative; unclassified count is 0.

- [ ] **Step 4: Verify ownership uniqueness**

Search external versions, phase/tick definitions, shared budgets, approval/authorization definitions, evidence/provenance definitions, and shared contract names. Inspect every match outside its owner and convert copied definitions to links.

Expected: every normative definition has one owner.

- [ ] **Step 5: Verify duplication and ambiguity**

Run exact paragraph duplicate detection for normalized paragraphs of 120 or more characters and character 4-gram comparison for every pair. Review all pairs at or above 0.70. Search for unresolved markers, open choice language, missing owners, contradictory status, and unqualified future behavior.

Expected: exact duplicate count 0; every high-similarity pair is either reduced or documented as responsibility-required; unresolved normative choice count 0.

- [ ] **Step 6: Verify official evidence and Context7 notes**

Check every normative external claim against its exact version in Context7 or an official primary source. Check all retained external URLs for successful resolution. Ensure current/recommended facts include a verification date and are not Architecture constants.

Expected: unsupported claims 0; unreachable normative links 0; undocumented Context7 fallback 0.

- [ ] **Step 7: Verify final diff and Git cleanliness**

Run:

```powershell
git diff --check
git status --short
git diff --stat 1308772..HEAD
```

Expected before final commit: only the intended plan deletion and any Gate fixes are pending; `git diff --check` exits 0.

- [ ] **Step 8: Commit verification fixes and remove the implementation plan**

```powershell
git rm -- docs/superpowers/plans/2026-07-21-architecture-document-restructure.md
git add -- docs
git commit -m "docs: verify canonical architecture documentation"
```

- [ ] **Step 9: Re-run the entire audit after the final commit**

Expected: all Gates still pass, `git status --short` is empty, and no required work remains.
