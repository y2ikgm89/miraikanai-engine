# Runtime ECS Data-Oriented Contract Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Update the Runtime architecture plans so Component SoA layout, cached archetype queries, allocation-free callbacks, bounded structural mutation, package capacity, and observability form one fail-closed ECS contract.

**Architecture:** Runtime ECS is the sole semantic owner of Component layout and structural behavior. Gameplay Programming Model supplies exact access-cohort inputs; Scheduling／Lifetime reserves work before dispatch and commits bounded deltas; Runtime Package proves the cooked layout／memory closure; Debugging consumes mandatory metric IDs without owning thresholds; the ECS decision record captures the approved target without activating it.

**Tech Stack:** Markdown architecture contracts, MCD canonical records, C++23／ECS terminology, PowerShell 7, ripgrep, Git.

## Global Constraints

- Normative design: `docs/superpowers/specs/2026-07-26-data-oriented-runtime-optimization-design.md`.
- Prerequisites:
  - complete `docs/superpowers/plans/2026-07-26-foundation-value-memory-contract-plan.md`;
  - complete Task 1 of `docs/superpowers/plans/2026-07-26-product-qualification-clean-break-plan.md` so the Performance-owned profile and metric literals exist before Runtime consumers bind them.
- Execution requires an isolated worktree or an explicitly approved baseline in
  which every file listed by this plan is tracked and clean. The current
  workspace contains pre-existing tracked and untracked Architecture work;
  never roll it into these commits implicitly.
- This plan consumes Performance IDs but must not redefine sampling or
  promotion thresholds.
- Do not edit or stage unrelated dirty-worktree changes.
- Before each commit, require `git diff --cached --name-only` to contain only that task's listed files.
- Use `git commit --only -- <paths>`; never commit all staged or working-tree changes.
- One Component type is one SoA chunk column; fields remain in the Component's canonical value layout.
- A semantic field split requires a new Component schema revision, updated persistence projection and System manifests, and requalification. The compiler never silently transposes fields.
- The accepted C1 layout is `ecs_chunk_soa_v1`, 16 KiB payload, 64-byte payload alignment, and a 256-byte maximum inline Component.
- The 8／16／32 KiB candidates are qualification inputs, not simultaneous Shipping storage paths.
- Cached queries contain archetype membership only. They never cache row predicate results, Component addresses, leases, spans, or persisted row selections.
- Query-plan scratch, masks, command buffers, and output packets are reserved before callback dispatch. General-heap allocation and upstream fallback inside a callback are both exactly zero.
- Structural operations are bounded deltas committed only at the existing structural boundary. Any failure preserves the previously published World, location table, query cache, and output.
- No Shipping AoS, sparse-set, object graph, per-Entity virtual update, Runtime `shared_ptr`, or general-heap fallback remains.

## Execution Preflight

After the Foundation plan and Product／Qualification Task 1, then before this
plan's Task 1, validate the implementation worktree:

```powershell
$targets = @(
  'docs/architecture/04-runtime/entity-component-system.md',
  'docs/architecture/03-authoring/gameplay-programming-model.md',
  'docs/architecture/04-runtime/scheduling-lifetime.md',
  'docs/architecture/04-runtime/runtime-package.md',
  'docs/architecture/04-runtime/debugging-observability-replay.md',
  'docs/architecture/decisions/2026-07-22-runtime-ecs-contract.md'
)
foreach ($path in $targets) {
  git ls-files --error-unmatch -- $path *> $null
  if ($LASTEXITCODE -ne 0) { throw "Target is not in the approved baseline: $path" }
  if (git status --porcelain -- $path) { throw "Target has pre-existing changes: $path" }
}
if (git diff --cached --name-only) { throw 'Index is not empty.' }
```

If this fails, stop before editing and obtain an explicitly approved baseline
commit. In particular, do not add the current untracked Runtime ECS／Package
documents to a plan commit without that authorization.

## File Map

| File | Responsibility in this plan |
|---|---|
| `docs/architecture/04-runtime/entity-component-system.md` | Canonical layout policy, query predicates, diagnostics, qualification references |
| `docs/architecture/03-authoring/gameplay-programming-model.md` | Exact access-cohort inputs and semantic schema-revision workflow |
| `docs/architecture/04-runtime/scheduling-lifetime.md` | Pre-dispatch reservation and structural publication boundary |
| `docs/architecture/04-runtime/runtime-package.md` | Cooked layout／memory closure and capacity validation |
| `docs/architecture/04-runtime/debugging-observability-replay.md` | Mandatory metric consumption and sealed evidence projection |
| `docs/architecture/decisions/2026-07-22-runtime-ecs-contract.md` | Approved target decision, evidence, and non-activation statement |

---

### Task 1: Make Runtime ECS the canonical Component-layout owner

**Files:**
- Modify: `docs/architecture/04-runtime/entity-component-system.md:52-121`
- Modify: `docs/architecture/04-runtime/entity-component-system.md:122-184`
- Modify: `docs/architecture/04-runtime/entity-component-system.md:185-299`
- Modify: `docs/architecture/04-runtime/entity-component-system.md:401-460`

**Interfaces:**
- Consumes: `ComponentSchemaRefV1`, active Game System manifests, query／phase refs, read／write sets, exact qualification profile ref.
- Produces: `RuntimeComponentLayoutPolicyV1`, layout review, query／callback predicates, six Runtime Diagnostics, one aggregate Product Diagnostic, and qualification closure.

- [ ] **Step 1: Run the missing-owner assertion**

```powershell
$path = 'docs/architecture/04-runtime/entity-component-system.md'
if (rg -q '^RuntimeComponentLayoutPolicyV1$' $path) {
  throw 'Precondition failed: Runtime ECS already owns the planned policy.'
}
Write-Error 'Expected failure: RuntimeComponentLayoutPolicyV1 is not defined.'
```

Expected: FAIL with the missing policy message.

- [ ] **Step 2: Add the exact canonical policy schema**

Immediately after the existing Component schema rules, add:

```text
RuntimeComponentLayoutPolicyV1
  policy_version: 1
  component_schema_ref: ComponentSchemaRefV1
  storage_class:
    inline_value | tag | external_handle | derived_index
  access_manifest_refs[1..4096]:
    sorted unique RuntimeComponentAccessManifestRefV1
  access_cohort_hash: SHA-256
  update_frequency:
    every_advance | periodic | event_driven | load_only
  structural_frequency:
    stable | bounded_transition | burst_candidate
  inline_bytes: uint32
  inline_alignment_bytes: uint32
  layout_disposition:
    accepted_inline | accepted_tag | accepted_external
    | component_schema_revision_required
  qualification_profile_ref:
    exact {profile_version, target_profile_ref, contract_set_ref,
           toolchain_lock_sha256, profile_hash}
  policy_hash: SHA-256
```

State that `access_cohort_hash` is derived from the exact active System
manifests, query refs, phase refs, and read／write sets. It is never a manually
assigned hot／cold label.

Compute `policy_hash` as:

```text
SHA-256(
  ASCII "MIRAKAN_RUNTIME_COMPONENT_LAYOUT_POLICY_V1"
  || uint32_be(len(canonical policy bytes excluding policy_hash))
  || canonical policy bytes excluding policy_hash
)
```

Materialize the policy only after the immutable Contract Set root exists. Bare
ID, `latest`, ambient current, same-name substitution, and
same-logical-key／version different-hash refs are invalid.

- [ ] **Step 3: Add the closed layout-review algorithm**

Add these normative rules:

```markdown
- one Component type occupies one SoA column;
- fields of an `inline_value` Component retain canonical value layout;
- colocate fields with common semantic ownership and access cohort;
- move large cold／infrequent state out of a hot Component only through a Domain-approved schema revision;
- externalize variable-length data、text、native objects, and large immutable shared data through typed handles;
- use value or enablement mutation for frequently toggled state when semantics do not require an archetype transition;
- reject unbounded tag combinations and shared-value partitions;
- do not split only to reduce `sizeof` when row capacity is already at its qualified ceiling or indirection costs more;
- preserve one active writer and one explicit persistent projection per authoritative field.
```

If disjoint System cohorts use fields in one large Component, emit
`component_schema_revision_required`; do not automatically rearrange or split
the fields.

- [ ] **Step 4: Close the accepted layout and query predicates**

In the archetype／query sections, state:

```markdown
The sole accepted C1 Shipping layout is `ecs_chunk_soa_v1` with a 16384-byte payload, 64-byte payload alignment, and a 256-byte maximum inline Component. The existing 8192／16384／32768-byte comparison can change that target only through a new qualified profile and contract revision; it does not create Runtime fallback paths.

A cached query stores archetype membership only. One generated callback receives one contiguous row range in one chunk, exposes only declared columns and immutable-selected rows, and cannot cross a chunk boundary. Query scratch、selection masks、command buffers, and output packets are fully reserved before dispatch.

Callback code cannot allocate from the general heap, use an upstream allocator fallback, grow an unbounded container, acquire shared ownership, or perform structural mutation. Create、destroy、add、remove, and enablement changes become bounded deltas for the structural boundary.
```

- [ ] **Step 5: Add exact failure atomicity and identity rules**

Require the structural boundary to validate every capacity and precondition
before publication. Failure leaves the previous World, location table, query
cache, output packet, and publication hash unchanged. A successful commit
invalidates affected addresses, rows, spans, leases, and query membership under
the existing epoch rules.

Keep typed generation handles as stable identity. State that chunk IDs, rows,
addresses, and pool slots are not identity. Remove or explicitly prohibit every
Shipping AoS／sparse-set／object-graph fallback statement in this file.

- [ ] **Step 6: Register six Runtime Diagnostics**

Add these exact pairs, arguments, and results:

```text
diagnostic.runtime.ecs-component-schema-revision-required
MIRAKAN-RUNTIME-ECS-COMPONENT-SCHEMA-REVISION-REQUIRED
arguments = component_schema_ref, access_cohort_hash
result = component_schema_revision_required; no silent split

diagnostic.runtime.ecs-chunk-capacity-zero
MIRAKAN-RUNTIME-ECS-CHUNK-CAPACITY-ZERO
arguments = archetype_id, column_layout_hash, payload_bytes
result = Cook／Contract compile failure

diagnostic.runtime.ecs-archetype-permutation-unbounded
MIRAKAN-RUNTIME-ECS-ARCHETYPE-PERMUTATION-UNBOUNDED
arguments = component_schema_ref, observed_count, declared_bound
result = Layout qualification failure

diagnostic.runtime.ecs-query-cache-invalid
MIRAKAN-RUNTIME-ECS-QUERY-CACHE-INVALID
arguments = query_ref, invalid_member_kind
result = Contract／negative fixture failure

diagnostic.runtime.ecs-structural-mutation-during-iteration
MIRAKAN-RUNTIME-ECS-STRUCTURAL-MUTATION-DURING-ITERATION
arguments = system_ref, operation_kind, logical_work_id
result = Typed rejection

diagnostic.runtime.ecs-structural-capacity-exceeded
MIRAKAN-RUNTIME-ECS-STRUCTURAL-CAPACITY-EXCEEDED
arguments = boundary_ref, required_bytes, available_bytes
result = Previous World remains published
```

- [ ] **Step 7: Add the aggregate Requirement Diagnostic and qualification refs**

Register:

```text
diagnostic.product.ecs-data-oriented-core-failed
MIRAKAN-PRODUCT-ECS-DATA-ORIENTED-CORE-FAILED
arguments = requirement_id, campaign_hash, target_profile_ref,
            failed_diagnostic_refs[1..64]
result = aggregate Requirement evaluation fails
```

Reference, but do not redefine,
`RuntimeDataOrientedQualificationProfileV1`,
`RuntimeDataOrientedQualificationCampaignV1`,
`fixture.runtime.ecs-data-oriented-core`, the six scenarios, complete mandatory
metric set, hard predicates, and 8／16／32 KiB characterization.

- [ ] **Step 8: Verify canonical ownership and commit only Runtime ECS**

```powershell
$path = 'docs/architecture/04-runtime/entity-component-system.md'
foreach ($token in @(
  'RuntimeComponentLayoutPolicyV1',
  'MIRAKAN_RUNTIME_COMPONENT_LAYOUT_POLICY_V1',
  'component_schema_revision_required',
  'ecs_chunk_soa_v1',
  'MIRAKAN-RUNTIME-ECS-QUERY-CACHE-INVALID',
  'MIRAKAN-RUNTIME-ECS-STRUCTURAL-CAPACITY-EXCEEDED',
  'MIRAKAN-PRODUCT-ECS-DATA-ORIENTED-CORE-FAILED'
)) {
  if (-not (rg -q --fixed-strings $token $path)) { throw "Missing token: $token" }
}
$fallbackClaims = rg -n -i '(fallback (to|uses?) (AoS|sparse-set|object graph)|(AoS|sparse-set|object graph) fallback is (enabled|retained|allowed))' $path
if ($LASTEXITCODE -eq 0) { throw "Shipping alternate layout claim remains:`n$fallbackClaims" }
git diff --check -- $path
git add -- $path
$staged = @(git diff --cached --name-only)
if ($staged.Count -ne 1 -or $staged[0] -ne $path) { throw "Unexpected staged paths: $($staged -join ', ')" }
git commit --only -m "docs: define canonical runtime ECS layout" -- $path
```

Expected: one-file commit and one full `RuntimeComponentLayoutPolicyV1` owner.

---

### Task 2: Derive access cohorts from Gameplay System contracts

**Files:**
- Modify: `docs/architecture/03-authoring/gameplay-programming-model.md:318-390`

**Interfaces:**
- Consumes: active `GameSystemSpecV2` records, query refs, phase refs, read／write sets, Component schemas.
- Produces: deterministic access-manifest closure and an explicit Domain-owned schema revision path; no local layout schema.

- [ ] **Step 1: Assert that exact access-cohort derivation is absent**

```powershell
$path = 'docs/architecture/03-authoring/gameplay-programming-model.md'
if (rg -q 'access_cohort_hash' $path) { throw 'Precondition failed: derivation already present.' }
Write-Error 'Expected failure: Game System contracts do not yet derive the ECS access cohort.'
```

Expected: FAIL.

- [ ] **Step 2: Add compiler inputs and deterministic closure**

Under the `GameSystemSpecV2` compiler rules, add:

```markdown
Runtime ECSへ渡す`access_cohort_hash`は、同じActive Product Definitionに属するactive `GameSystemSpecV2`、exact query refs、phase refs、Component read／write setsをcanonical sortしてSHA-256化する。inactive、draft、別Contract Set、別TargetのSystemを混ぜない。手書き`hot`／`cold` labelやsource-order hashを入力にしない。

各Componentの`RuntimeComponentAccessManifestRefV1`集合はstrict-sorted uniqueで、missing System、orphan query、undeclared column access、writer conflictをContract compile failureにする。
```

- [ ] **Step 3: Add the semantic split workflow**

Require `component_schema_revision_required` to stop layout compilation. The
Domain owner must:

1. approve a semantically named Component schema revision;
2. update System read／write manifests and queries;
3. update the authoritative persistence projection;
4. recook Source-derived artifacts;
5. rerun the complete data-oriented qualification campaign.

State that Gameplay Programming Model neither auto-splits fields nor owns
`RuntimeComponentLayoutPolicyV1`.

- [ ] **Step 4: Verify owner separation and commit**

```powershell
$path = 'docs/architecture/03-authoring/gameplay-programming-model.md'
foreach ($token in @('access_cohort_hash','GameSystemSpecV2','component_schema_revision_required','RuntimeComponentAccessManifestRefV1')) {
  if (-not (rg -q --fixed-strings $token $path)) { throw "Missing token: $token" }
}
if (rg -q '^RuntimeComponentLayoutPolicyV1$' $path) { throw 'Gameplay duplicated the ECS schema.' }
git diff --check -- $path
git add -- $path
$staged = @(git diff --cached --name-only)
if ($staged.Count -ne 1 -or $staged[0] -ne $path) { throw "Unexpected staged paths: $($staged -join ', ')" }
git commit --only -m "docs: derive ECS access cohorts from systems" -- $path
```

Expected: one-file commit with a deterministic producer contract.

---

### Task 3: Reserve callback work and commit structural changes at Scheduling boundaries

**Files:**
- Modify: `docs/architecture/04-runtime/scheduling-lifetime.md:643-715`
- Modify: `docs/architecture/04-runtime/scheduling-lifetime.md:716-760`

**Interfaces:**
- Consumes: query plan, immutable selection mask, chunk row range, reserved buffers, structural delta capacity.
- Produces: allocation-free dispatch and all-or-nothing structural publication; no ECS layout definition.

- [ ] **Step 1: Prove the allocation predicates are incomplete**

```powershell
$path = 'docs/architecture/04-runtime/scheduling-lifetime.md'
$missing = @('query-plan scratch','upstream fallback','contiguous row range') |
  Where-Object { -not (rg -q --fixed-strings $_ $path) }
if ($missing.Count -eq 0) { throw 'Precondition failed: scheduling predicates already complete.' }
Write-Error "Expected failure: missing scheduling predicates: $($missing -join ', ')"
```

Expected: FAIL and at least one missing predicate.

- [ ] **Step 2: Add the exact pre-dispatch reservation contract**

At the existing ECS execution boundary, require:

```markdown
Query-plan scratch、selection masks、per-worker command buffers、merge scratch, and output packets are capacity-validated and reserved before callback dispatch. Each callback receives exactly one contiguous row range within one chunk and only its declared columns. The callback performs zero general-heap allocations, zero upstream allocator fallbacks, zero shared-ownership acquisition, and zero container growth.
```

Capacity comes from the exact Runtime Package／World plan and
`MemoryContractV1`; a missing reservation rejects dispatch rather than growing
inside the callback.

- [ ] **Step 3: Close structural commit failure behavior**

Require create／destroy／add／remove／enablement changes to be emitted as bounded
deltas. The boundary sorts／merges them deterministically, validates the full
result and capacity, then commits once. Any rejected delta, capacity overflow,
or stale generation:

- publishes no partial mutation;
- preserves previous World, location table, query cache, and output;
- emits the owning Runtime ECS Diagnostic;
- emits `diagnostic.runtime.ecs-structural-capacity-exceeded` for exhausted
  structural reservation;
- never retries through another storage backend or the general heap.

- [ ] **Step 4: Extend positive and negative scheduling fixtures**

Add fixtures for:

```text
one chunk / one contiguous work unit
multi-chunk query / one work unit per chunk
selection mask hides unselected rows
undeclared column access
direct structural mutation during iteration
command buffer capacity exceeded
merge scratch capacity exceeded
general-heap allocation inside callback
upstream fallback inside callback
failure atomicity across World, location table, query cache, output
```

- [ ] **Step 5: Verify and commit only Scheduling／Lifetime**

```powershell
$path = 'docs/architecture/04-runtime/scheduling-lifetime.md'
foreach ($token in @(
  'query-plan scratch',
  'upstream allocator fallback',
  'one contiguous row range',
  'structural-capacity-exceeded'
)) {
  if (-not (rg -q --fixed-strings $token $path)) { throw "Missing token: $token" }
}
git diff --check -- $path
git add -- $path
$staged = @(git diff --cached --name-only)
if ($staged.Count -ne 1 -or $staged[0] -ne $path) { throw "Unexpected staged paths: $($staged -join ', ')" }
git commit --only -m "docs: close ECS scheduling allocation boundaries" -- $path
```

Expected: one-file commit and fail-closed callback dispatch.

---

### Task 4: Bind Runtime Packages to exact layout and memory closures

**Files:**
- Modify: `docs/architecture/04-runtime/runtime-package.md:29-95`
- Modify: `docs/architecture/04-runtime/runtime-package.md:192-260`

**Interfaces:**
- Consumes: `RuntimeWorldCapacityRecordV1`, Construction Set root, Contract Set root, `RuntimeArchetypeLayoutPlanV1`, `RuntimeComponentLayoutPolicyV1`, `MemoryContractV1`.
- Produces: exact transitive binding and loader rejection rules; no duplicate layout or memory schema.

- [ ] **Step 1: Assert the new layout policy is not in the package closure**

```powershell
$path = 'docs/architecture/04-runtime/runtime-package.md'
if (rg -q 'RuntimeComponentLayoutPolicyV1' $path) { throw 'Precondition failed: package already binds the policy.' }
Write-Error 'Expected failure: Runtime Package lacks the Component layout policy binding.'
```

Expected: FAIL.

- [ ] **Step 2: Document the exact content-addressed binding chain**

Extend the existing `RuntimeWorldCapacityRecordV1`／World build closure without
duplicating owner Fields:

```text
Package Candidate
  -> exact RuntimeWorldCapacityRecordV1 ref
  -> exact Construction Set root
  -> RuntimeArchetypeLayoutPlanV1 refs
  -> RuntimeComponentLayoutPolicyV1 refs
  -> exact Contract Set root
  -> MemoryContractV1 and PointerMemoryConsumerBindingV1 refs
```

The Package and loader validate that all refs resolve to the same Target
Profile, Contract Set, Toolchain lock, Component schema versions, and Candidate
root. State explicitly that this is an exact transitive closure; Runtime
Package does not copy the policy or memory Field lists.

- [ ] **Step 3: Add capacity and clean-break loader rejection**

Reject before World construction:

```markdown
- missing／extra／duplicate layout policy or archetype layout refs;
- a Component schema hash that differs from its layout policy;
- zero rows after alignment and payload calculation;
- unbounded archetype permutation or structural delta capacity;
- a query／command／output reservation not covered by capacity records;
- old AoS／sparse-set／object-graph package sections;
- old generated signatures, pointer-backed inline payload, or persisted row selections;
- any fallback to global `new`, default PMR, or a second Shipping storage backend.
```

Failure publishes neither a partial World nor a repaired Package.

- [ ] **Step 4: Bind qualification without duplicating thresholds**

Require the loader／capacity qualification receipt to reference the exact
`RuntimeDataOrientedQualificationCampaignV1` for the same Candidate, Target,
Contract Set, Toolchain, fixture, and input trace. Runtime Package consumes the
receipt and hard-predicate result; Performance／Capacity remains the owner of
sampling, metrics, and promotion.

- [ ] **Step 5: Verify and commit only Runtime Package**

```powershell
$path = 'docs/architecture/04-runtime/runtime-package.md'
foreach ($token in @(
  'RuntimeComponentLayoutPolicyV1',
  'RuntimeDataOrientedQualificationCampaignV1',
  'exact transitive closure',
  'old AoS',
  'default PMR'
)) {
  if (-not (rg -q --fixed-strings $token $path)) { throw "Missing token: $token" }
}
if (rg -q '^RuntimeComponentLayoutPolicyV1$|^MemoryContractV1$' $path) {
  throw 'Runtime Package duplicated an owner schema.'
}
git diff --check -- $path
git add -- $path
$staged = @(git diff --cached --name-only)
if ($staged.Count -ne 1 -or $staged[0] -ne $path) { throw "Unexpected staged paths: $($staged -join ', ')" }
git commit --only -m "docs: bind runtime packages to ECS layout closure" -- $path
```

Expected: one-file commit with exact package-to-contract closure.

---

### Task 5: Project mandatory ECS evidence through Debugging

**Files:**
- Modify: `docs/architecture/04-runtime/debugging-observability-replay.md:156-220`
- Modify: `docs/architecture/04-runtime/debugging-observability-replay.md:674-735`

**Interfaces:**
- Consumes: Performance-owned `runtime_ecs_data_oriented_metrics_v1`, ECS Diagnostics, sealed campaign artifacts.
- Produces: counter registration and read-only evidence projection; no thresholds or authority back-edge.

- [ ] **Step 1: Assert the mandatory metric-set consumer is absent**

```powershell
$path = 'docs/architecture/04-runtime/debugging-observability-replay.md'
if (rg -q 'runtime_ecs_data_oriented_metrics_v1' $path) { throw 'Precondition failed: metric set already consumed.' }
Write-Error 'Expected failure: Debugging does not yet project the ECS metric set.'
```

Expected: FAIL.

- [ ] **Step 2: Register the complete mandatory metric families**

Reference `runtime_ecs_data_oriented_metrics_v1` and register counters for:

```text
callback general-heap allocation and upstream fallback
reserved, committed, live, and peak bytes
chunk count, row capacity, occupied rows, unused payload bytes
archetype count and archetype fragmentation
selected rows, contiguous work units, and chunk transitions
exposed column bytes and useful selected payload bytes
query-cache hit, miss, rebuild, and invalidation
structural moved rows and structural copy bytes
handle resolve and lease-validation P50/P95/P99
scenario CPU P50/P95/P99
semantic result hash, publication hash, and failure atomicity
```

Use the exact metric IDs declared by Performance／Capacity. Do not introduce
local aliases, sampling durations, percentile algorithms, or pass thresholds.

- [ ] **Step 3: Add sealed evidence and authority rules**

Require every sample to carry the exact campaign, scenario, payload candidate,
Candidate, Target, Contract Set, Toolchain, fixture, and input trace identity.
Missing members remain missing; Debugging never normalizes them to zero.

Replay and Debug read sealed snapshots only. No counter, panel, replay event,
or AI summary can mutate live Component layout, query membership, structural
capacity, or authoritative state.

- [ ] **Step 4: Add qualification negative fixtures**

Cover missing metric, NaN／infinite value, counter overflow, wrong Target,
wrong Candidate, process reuse, missing campaign cell, and Debug-to-Runtime
authority back-edge. Map missing mandatory metrics to
`diagnostic.performance.ecs-required-metric-missing`.

- [ ] **Step 5: Verify and commit only Debugging／Observability／Replay**

```powershell
$path = 'docs/architecture/04-runtime/debugging-observability-replay.md'
foreach ($token in @(
  'runtime_ecs_data_oriented_metrics_v1',
  'callback general-heap allocation',
  'archetype fragmentation',
  'diagnostic.performance.ecs-required-metric-missing',
  'authority back-edge'
)) {
  if (-not (rg -q --fixed-strings $token $path)) { throw "Missing token: $token" }
}
git diff --check -- $path
git add -- $path
$staged = @(git diff --cached --name-only)
if ($staged.Count -ne 1 -or $staged[0] -ne $path) { throw "Unexpected staged paths: $($staged -join ', ')" }
git commit --only -m "docs: project ECS qualification telemetry" -- $path
```

Expected: one-file commit and a read-only complete metric projection.

---

### Task 6: Record the approved Runtime ECS target without activating it

**Files:**
- Modify: `docs/architecture/decisions/2026-07-22-runtime-ecs-contract.md:45-125`

**Interfaces:**
- Consumes: approved design, official external basis, canonical owner split, clean-break migration.
- Produces: target-decision addendum and implementation verification; no Product activation.

- [ ] **Step 1: Assert the approved policy is not recorded**

```powershell
$path = 'docs/architecture/decisions/2026-07-22-runtime-ecs-contract.md'
if (rg -q 'RuntimeComponentLayoutPolicyV1' $path) { throw 'Precondition failed: decision already records the policy.' }
Write-Error 'Expected failure: ECS decision lacks the approved data-oriented target.'
```

Expected: FAIL.

- [ ] **Step 2: Add a dated approved-target addendum**

Record the 2026-07-26 approval with:

- one Component／one SoA column and semantic schema revision;
- 16 KiB／64-byte／256-byte accepted C1 values;
- cached archetype membership only;
- allocation-free, no-fallback callbacks;
- bounded structural deltas and atomic publication;
- exact Foundation, Runtime ECS, Performance, Compatibility, and Product owner split;
- source-preserving recook clean break with inventory requirement;
- complete 90-run, three-soak, 12,600-second qualification.

Set `外部根拠検証日` to `2026-07-26` and add these exact primary references to
the comparison-evidence section:

```text
https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines
https://eel.is/c++draft/vector
https://eel.is/c++draft/class.copy.elision
https://eel.is/c++draft/views.span
https://eel.is/c++draft/mem.res
https://learn.microsoft.com/en-us/cpp/build/reference/zc-nrvo
https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/concepts-archetypes.html
https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/performance-chunk-allocations.html
https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/concepts-structural-changes.html
https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/systems-entity-command-buffers.html
https://dev.epicgames.com/documentation/unreal-engine/common-memory-and-cpu-performance-considerations-in-unreal-engine
https://dev.epicgames.com/documentation/unreal-engine/introduction-to-performance-profiling-and-configuration-in-unreal-engine
https://github.com/skypjack/entt/blob/v3.16.0/docs/md/entity.md
```

Retain the existing official Flecs query comparison if its pinned version and
link still resolve. Label every vendor source as comparison evidence only. Do
not copy vendor wording or claim a vendor API, chunk size, storage backend, or
the Miraikanai numeric target is universally optimal.

- [ ] **Step 3: Preserve decision-vs-activation separation**

Add:

```markdown
この承認はtarget contractと実装計画の承認であり、current Active Product Definition、Capability state、Work Package lifecycle head、Runtime Package、Shipping pathを変更しない。Activationにはcanonical Architecture更新、complete Consumer Inventory、同一Candidateのqualification Receipt、atomic Product Definition Migrationが別途必要である。
```

- [ ] **Step 4: Verify decision size, links, and commit**

```powershell
$path = 'docs/architecture/decisions/2026-07-22-runtime-ecs-contract.md'
foreach ($token in @(
  'RuntimeComponentLayoutPolicyV1',
  '12,600',
  'source-preserving recook',
  'current Active Product Definition'
)) {
  if (-not (rg -q --fixed-strings $token $path)) { throw "Missing token: $token" }
}
$lineCount = (Get-Content $path).Count
if ($lineCount -gt 1000) { throw "Decision exceeds 1000 lines: $lineCount" }
$content = Get-Content -Raw $path
foreach ($match in [regex]::Matches($content, '\]\((?!https?://|#|mailto:)([^)#]+)(?:#[^)]+)?\)')) {
  $link = $match.Groups[1].Value
  $resolved = Join-Path (Split-Path $path) $link
  if (-not (Test-Path $resolved)) { throw "Broken relative link: $link" }
}
git diff --check -- $path
git add -- $path
$staged = @(git diff --cached --name-only)
if ($staged.Count -ne 1 -or $staged[0] -ne $path) { throw "Unexpected staged paths: $($staged -join ', ')" }
git commit --only -m "docs: approve data-oriented ECS target" -- $path
```

Expected: one-file commit, valid links, and no activation claim.

---

### Task 7: Audit the complete Runtime contract slice

**Files:**
- Verify: all six files in this plan
- Verify without modification: `docs/architecture/04-runtime/persistence-save.md`

**Interfaces:**
- Consumes: Tasks 1–6 and the Foundation plan result.
- Produces: evidence that Runtime contracts have one owner per schema, complete consumers, and no Shipping fallback.

- [ ] **Step 1: Verify schema ownership and direct consumers**

```powershell
$files = @(
  'docs/architecture/04-runtime/entity-component-system.md',
  'docs/architecture/03-authoring/gameplay-programming-model.md',
  'docs/architecture/04-runtime/scheduling-lifetime.md',
  'docs/architecture/04-runtime/runtime-package.md',
  'docs/architecture/04-runtime/debugging-observability-replay.md',
  'docs/architecture/decisions/2026-07-22-runtime-ecs-contract.md'
)
$owners = @(rg -l '^RuntimeComponentLayoutPolicyV1$' $files | ForEach-Object { $_ -replace '\\','/' })
if ($owners.Count -ne 1 -or $owners[0] -ne 'docs/architecture/04-runtime/entity-component-system.md') {
  throw "Unexpected layout policy owners: $($owners -join ', ')"
}
foreach ($file in $files) {
  git diff --check -- $file
}
```

Expected: Runtime ECS is the sole full schema owner.

- [ ] **Step 2: Verify the no-fallback and allocation predicates**

```powershell
$required = @{
  'docs/architecture/04-runtime/entity-component-system.md' = @('ecs_chunk_soa_v1','general heap','upstream fallback')
  'docs/architecture/04-runtime/scheduling-lifetime.md' = @('query-plan scratch','one contiguous row range')
  'docs/architecture/04-runtime/runtime-package.md' = @('old AoS','default PMR')
}
foreach ($entry in $required.GetEnumerator()) {
  foreach ($token in $entry.Value) {
    if (-not (rg -q --fixed-strings $token $entry.Key)) { throw "Missing $token in $($entry.Key)" }
  }
}
```

Expected: PASS.

- [ ] **Step 3: Verify Persistence remains a projection-only consumer**

```powershell
$path = 'docs/architecture/04-runtime/persistence-save.md'
foreach ($token in @('raw RuntimeEntityHandle','chunk location','live pointer','Save／Replay projection')) {
  if (-not (rg -qi --fixed-strings $token $path)) { throw "Persistence audit needs manual review for: $token" }
}
if (rg -q '^RuntimeComponentLayoutPolicyV1$' $path) { throw 'Persistence duplicated the ECS owner schema.' }
git diff --exit-code -- $path
```

Expected: no change introduced by this plan. If the user already had a
pre-existing diff in this file, compare the pre-task blob recorded before
execution instead of requiring a clean Git diff.

- [ ] **Step 4: Record verification evidence**

```powershell
git log --oneline -6
git status --short
```

Expected: six scoped commits from this plan; unrelated user changes remain
uncommitted and intact.
