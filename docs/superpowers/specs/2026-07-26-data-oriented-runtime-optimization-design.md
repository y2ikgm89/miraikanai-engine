# Data-Oriented Runtime Optimization Contract Design

## 1. Status

- Date: 2026-07-26
- Scope: C++ value transfer、memory layout、Runtime ECS、performance
  qualification、Product Work Package、clean-break coordination
- Decision: adopt one Engine-owned archetype／chunk SoA Runtime path and
  contract-driven C++ value transfer／allocation rules
- Implementation status: design only
- Compatibility status: internal target design uses clean break; retained
  User data or published consumers require an evidence-backed finite migration
  instead of an implicit compatibility layer

This design coordinates the existing Product、Governance、Foundation、
Authoring、Runtime、Simulation、Rendering、Platform、Pack owners. It does not
activate an MCD type or Operation, publish a Capability, claim that an ECS
implementation exists, or change current owner authority merely by adding
Architecture prose.

The repository audit covered all 52 identified Architecture documents plus the
Architecture Index. Document IDs are unique and all relative Markdown links
resolve. The worktree already contains unrelated uncommitted Architecture
changes; this design preserves them and defines only the additional
coordination required for data-oriented optimization.

## 2. Decision

Miraikanai Engine uses the following single production direction.

1. Public and generated C++ APIs express value transfer, borrowing, mutation,
   bounded range access, and ownership transfer explicitly.
2. Hot traversal uses contiguous inline values or contiguous typed handles.
   A contiguous pointer or handle array is never reported as contiguous object
   storage.
3. Runtime ECS uses Engine-owned archetype chunks with one contiguous column
   per Component type and contiguous row-range callbacks.
4. ECS is SoA across Component types. Fields inside one Component value remain
   one column element; the Engine does not automatically split every field
   into a separate Component.
5. Component boundaries are chosen from semantic ownership and actual System
   access cohorts. Hot／cold splitting is accepted only when integrated
   evidence outweighs additional archetypes、indirection、structural copies,
   and lifecycle complexity.
6. Structural changes are deferred and committed at the existing Runtime
   boundary. Direct structural mutation during iteration is not another
   supported path.
7. General heap allocation and upstream fallback are zero in declared hot
   callbacks. Capacity is reserved before entering the callback.
8. Performance claims require the same Candidate、Contract Set、Toolchain
   lock、Target Profile、fixture、input trace, and correctness oracle.
9. AoS、sparse-set、object-per-Entity, and alternate chunk sizes may exist in
   benchmark-only comparators. They are not Shipping fallback paths, dual
   readers, or compatibility implementations.
10. Internal pre-1.0 aliases、dual layouts、dual serializers、old readers, and
    old writers are removed in one coordinated change after Consumer Inventory
    closure. Committed Source and qualified User data are preserved according
    to the Compatibility owner.

## 3. Considered approaches

### 3.1 Minimal five-document patch

This approach would update only Product Plan、Memory／Pointers、Runtime ECS、
Performance／Capacity, and Compatibility／Evolution.

It is rejected because generated C++ rules would remain disconnected from
Executable Contracts and C++23 Modules, Game System access would remain
disconnected from Gameplay Programming Model, and Product Work Packages would
have no complete consumer closure.

### 3.2 Canonical contract slice

This is the selected approach. Eight canonical documents define the rules and
six direct consumers bind to them. All remaining documents are checked for
conflicting definitions and continue to reference their existing owners.

This approach keeps each rule in one owner while making Product scheduling,
MCD generation, Runtime storage, performance evidence, and clean-break
application form one traceable chain.

### 3.3 Rewrite every Architecture document

This approach is rejected. Repeating the same transfer、layout、allocation,
and threshold rules across 52 owner documents would create competing
authorities and make later changes unsafe. Documents with no direct contract
change must be verified, not mechanically rewritten.

## 4. Owner boundaries

| Concern | Canonical owner | Required consumers |
|---|---|---|
| Product phase、Requirement、Fixture、Gate、Work Package、risk | Product Plan | all implementation owners |
| Cross-layer clean implementation principle | Core Architecture | Foundation、Runtime、Authoring |
| MCD envelope、materialization、generated closure | Executable Contracts | Memory、ECS、Product |
| C++ value transfer、pointer、generic container layout、allocation、OOM | Memory／Pointers | C++23、Native Module、ECS、all hot subsystems |
| Consumer discovery、clean break、recook／rebuild、finite migration | Compatibility／Evolution | Product、ECS、Package、Save |
| Entity、Component、archetype／SoA、query、lease、structural transaction | Runtime ECS | Gameplay、Scheduling、Package、Debug |
| Measurement、regression、promotion、Target qualification | Performance／Capacity | Product、Memory、ECS、all subsystem owners |
| ECS ownership transfer and target canonicalization sequence | Runtime ECS Decision | Governance、Compatibility、Product |

Direct consumers do not copy another owner's Field set、algorithm、numeric
threshold, or failure rule. They bind an exact Contract／Requirement／Profile
ref and add only their domain-specific input or output.

### 4.1 Common identity, reference, and hash rules

Every planned top-level record in this design has a closed logical key、
positive version, and self-excluding SHA-256 content hash. It does not invent
an opaque Stable ID when the existing logical key is already the exact Target、
Contract Set、Component schema, or Candidate binding. Canonical bytes use the
MCD canonical encoding owned by Executable Contracts. Product signed wrappers
continue to use Product Plan's existing RFC 8785 JCS rules; the two encodings
are not substituted for one another.

The MCD Contract Set contains each planned record's Type schema, not an
instance whose `contract_set_ref` points back to that same root. The Contract
Set root is finalized first. Policy、layout、profile, and campaign instances
are then materialized against that immutable root and retained as
content-addressed artifacts. A generated instance is never inserted back into
its own Contract Set preimage. This ordering prevents a Contract Set／instance
hash cycle.

The hash domains are:

| Record | ASCII domain |
|---|---|
| `CppValueTransferPolicyV1` | `MIRAKAN_CPP_VALUE_TRANSFER_POLICY_V1` |
| one value-transfer binding | `MIRAKAN_CPP_VALUE_TRANSFER_BINDING_V1` |
| `RuntimeComponentLayoutPolicyV1` | `MIRAKAN_RUNTIME_COMPONENT_LAYOUT_POLICY_V1` |
| `RuntimeDataOrientedQualificationProfileV1` | `MIRAKAN_RUNTIME_DATA_ORIENTED_QUALIFICATION_PROFILE_V1` |
| `RuntimeDataOrientedQualificationCampaignV1` | `MIRAKAN_RUNTIME_DATA_ORIENTED_QUALIFICATION_CAMPAIGN_V1` |

For each row, the hash is
`SHA-256(ASCII domain || uint32_be(len(canonical bytes excluding only that
record's own hash Field)) || canonical bytes)`. Nested binding hashes are
calculated before the enclosing policy hash. Value-transfer bindings are
strictly sorted and unique by
`{callable_key canonical bytes, subject enum, ordinal presence, ordinal}`.

Existing MCD／Product references continue to resolve by ID、version, and
content hash inside the declared Contract Set or Product Definition. New
content-addressed artifact references contain the exact logical-key Fields
shown in this design plus version and content hash. Bare ID、`latest`、ambient
current, same-name substitution, and same-logical-key／version different-hash
are invalid. A schema shown inline as `exact {...}` is a nested reference
shape and does not create an additional standalone Type or owner.

## 5. C++ value-transfer contract

Memory／Pointers owns the following planned type.

```text
CppValueTransferPolicyV1
  policy_version: 1
  contract_set_ref: ContractSetRefV1
  target_profile_ref:
    exact {target_profile_id, target_profile_version,
           target_profile_content_hash}
  toolchain_lock_sha256: SHA-256
  target_abi_facts:
    data_model: ilp32 | lp64 | llp64
    abi_word_bytes: 4 | 8
    pointer_bytes: 4 | 8
  bindings[1..65536]:
    callable_key:
      api_contract_id: StableId
      api_contract_version: positive uint32
      api_contract_hash: SHA-256
    subject: parameter | return_value
    ordinal: optional uint16
    type_ref: McdContractRefV1(kind=type)
    pointer_contract_id: StableId
    semantic: input | input_output | output | sink | bounded_range
    transfer_form:
      value | const_borrow | mutable_borrow | return_value
      | move_sink | bounded_view | unique_owner
    value_class:
      scalar | enumeration | typed_handle | compact_trivial
      | bounded_view | nontrivial | move_only
    moved_from_policy: not_applicable | destroy_or_assign_only
    binding_hash: SHA-256
  policy_hash: SHA-256
```

`callable_key` is a nested key for the source API contract, not a new
standalone MCD Type and not a hash of the generated C++ signature. This avoids
a cycle in which a generated signature is required to choose that signature.
`pointer_contract_id` resolves exactly once inside `contract_set_ref`; its
ownership、lifetime、access, and invalidation Fields are not duplicated in the
value-transfer policy. Missing、multiple, or cross-Contract-set resolution is a
Contract compile failure.

`target_abi_facts` are compiler-observed facts from the exact Toolchain lock
and Target Profile. They are not caller-selected tuning values. The compiler
verifies `sizeof`、`alignof`, C++ traits, data model, word size, and pointer
size before materializing the policy and rejects a fact mismatch.

The binding rules are closed.

| Semantic | Required form |
|---|---|
| Input scalar、enum、typed generation handle | `value` |
| Input `compact_trivial` value | `value` only when `is_trivially_copyable`, contains no owner, and is at most two Target ABI words |
| Other synchronous input | `const_borrow` represented by `const T&` |
| Input／output value | `mutable_borrow` represented by `T&` |
| Read-only or mutable sequence | `bounded_view` represented by value-passed `std::span<const T>` or `std::span<T>` |
| Ordinary output | `return_value`; recoverable failure uses the existing `Result<T>` form |
| Move-only unique owner sink | `unique_owner` passed by value |
| Non-owner object consumed unconditionally | `move_sink` represented by `T&&`, moved exactly once |

`compact_trivial` is not inferred from a display name or an arbitrary class
size. Contract generation verifies the Target ABI word size、C++ traits、
ownership closure, and exact `sizeof`／`alignof` facts from the locked
Toolchain. A type that fails any predicate is `nontrivial` and uses
`const_borrow` for input.

The generated and First-party C++ Gate rejects:

- `const T&&`;
- returning `std::move(local)` when `local` is eligible for copy elision;
- `std::move` from a `const` object;
- conditional move from a sink;
- access to a moved-from value other than destruction or assignment;
- by-value input of an unqualified nontrivial type;
- `const T&` for a scalar、enum, or typed handle without an approved measured
  exception;
- non-`const` reference parameters that are never written;
- output parameters where an ordinary return value is sufficient;
- `shared_ptr` parameters that do not express and retain shared ownership.

Private implementation may use a measured exception only through an ADR that
contains the exact Target Profile、fixture、baseline, improvement, and static
rule suppression scope. Generated public C++ APIs do not inherit a private
exception.

The Native C ABI is not a direct `transfer_form` projection. It continues to
use Native Game Module's fixed-width standard-layout values、opaque handles、
C function tables, and caller-owned bounded buffers:

- no C ABI declaration contains a C++ reference、`std::span`, STL／PMR type,
  `Result<T>`, exception, or owner wrapper;
- a C++ `bounded_view` becomes the existing C ABI pointer／byte-or-element
  count shape with an explicit call lifetime;
- `unique_owner` and `move_sink` never cross the C ABI; cross-boundary
  ownership uses the existing Memory Port、opaque handle, or caller-owned
  output buffer;
- the generated C++ adapter validates pointer、count、alignment、lifetime, and
  `PointerContractV1`, creates the `std::span` or bounded view only for the
  call, and does not retain it.

The adapter's C++ side must satisfy `CppValueTransferPolicyV1`; the C ABI wire
shape remains owned by Native Game Module. A private exception cannot weaken
either side.

Return-by-value relies on standard copy elision and NRVO. Code returns the
named local directly. Explicit move remains valid for ownership hand-off,
container insertion, and a declared sink after the last pre-move access.

## 6. Generic container and allocation contract

`MemoryContractV1` keeps all existing allocation、budget、phase、thread、OOM,
and telemetry Fields. Before materialization it adds the six layout／access
Fields below and replaces the prose-only `capacity_source` choices with the
shown closed enum. It does not add a second `capacity_source` Field.

```text
storage_layout:
  contiguous_inline | contiguous_handle | chunk_column
  | ring | segmented | node_based | external
element_storage: inline_value | typed_handle | external_adapter
access_pattern:
  sequential | indexed | sparse_lookup | producer_consumer | infrequent
capacity_source:
  profile | cooked_artifact | fixed_abi | measured_p999_headroom
growth_policy:
  pre_reserved_fixed | boundary_grow | budgeted_non_hot | forbidden
address_stability:
  not_required | scope_bound | handle_resolved
hot_path: bool
```

The generic rules are:

1. `std::vector<T>` is the default dynamic container for non-ECS contiguous
   inline values when the default System resource is allowed.
2. `std::pmr::vector<T>` is used only with a Composition Root-injected resource
   whose lifetime exceeds the container. The process-global default PMR
   resource is never changed.
3. ECS columns use the ECS private chunk storage and generated bounded views;
   they are not independent vectors with independent allocations.
4. `std::vector<Handle<T>>` and `std::vector<T*>` provide contiguous references
   only. They do not satisfy a `contiguous_inline` object requirement.
5. Known cardinality is reserved before a hot phase. `reserve()` during a hot
   callback is a contract failure even when no reallocation happens.
6. Reallocation-invalidated pointers、references、iterators, and spans never
   cross the owning scope or structural boundary.
7. `node_based` is prohibited on sequential hot traversal. It is allowed for
   an infrequent control-plane lookup or a measured exception with integrated
   evidence.
8. Pooling is limited to private fixed-size objects with stable churn and a
   measured improvement. Pooling a cheap C++ value without evidence is not an
   optimization.
9. Frame、RenderFrame, and Job scratch use bounded arenas and reset at the
   existing lifetime boundary. Exhaustion does not fall through to the general
   heap.
10. Hot callback allocation count and general upstream fallback count are both
    exact zero.

## 7. Runtime Component layout policy

Runtime ECS owns the following planned type.

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

`access_cohort_hash` is derived from the exact active System manifests、query
refs、phase refs, and read／write sets. It is not a manually assigned
`hot`／`cold` label.

One Component type occupies one SoA column. Fields within an `inline_value`
Component retain the Component's canonical value layout. When one large
Component contains fields used by disjoint System cohorts, the ECS compiler
does not silently rearrange or split the fields. It emits
`component_schema_revision_required`; the Domain owner must choose a semantic
split, update persistence projection and System manifests, and requalify the
new schema.

The layout review applies these rules together:

- keep fields together when they share semantic ownership and are normally
  consumed together;
- separate large cold or infrequently accessed state when hot Systems would
  otherwise load it on every selected row;
- externalize variable-length data、text、native objects, and large immutable
  shared data through typed handles;
- do not use temporary Component add／remove as a frequently toggled Boolean
  when the existing enablement mechanism represents the same semantics;
- reject unbounded tag combinations or shared-value partitions that cause
  archetype explosion;
- do not split merely to reduce `sizeof` when row capacity is already at its
  qualified ceiling or the added indirection costs more;
- preserve one active writer for each authoritative field and one explicit
  persistent projection.

The accepted C1 Runtime layout remains `ecs_chunk_soa_v1`, 16 KiB payload,
64-byte payload alignment, and a 256-byte maximum inline Component. These are
Miraikanai target values, not claims of a universal vendor optimum. The
8／16／32 KiB comparison required by the existing ECS contract remains the
qualification mechanism for changing the value.

## 8. Query, iteration, and structural mutation

The existing Runtime ECS query and structural rules remain authoritative and
gain explicit performance predicates.

1. A cached query stores archetype membership only. It never stores row
   predicate results、Component addresses、leases, or spans.
2. One callback receives exactly one contiguous row range in one chunk.
3. The generated callback exposes only columns in the manifest and only rows
   in the immutable selection mask.
4. Query-plan scratch、selection masks, command buffers, and output packets are
   reserved before callback dispatch.
5. Callback code cannot allocate from the general heap, grow an unbounded
   container, acquire shared ownership, or perform structural mutation.
6. Create、destroy、add、remove, and enablement changes are emitted as bounded
   deltas and merged at the existing structural boundary.
7. The boundary validates all capacity and preconditions before publication.
   Failure leaves the previous World、location table, query cache, and
   publication unchanged.
8. A structural commit invalidates affected addresses、rows、spans、leases,
   and query membership according to the existing epoch rules.
9. Stable identity is a typed generation handle. Chunk ID、row、address, and
   pool slot are not persistent identity.
10. Frequently toggled state uses value or enablement mutation when its
    semantics do not require an archetype transition.

No sparse-set or object graph fallback is retained when the archetype path
fails. Capacity failure is typed and fail-closed.

## 9. Data-oriented qualification profile

Performance／Capacity owns:

```text
RuntimeDataOrientedQualificationProfileV1
  profile_version: 1
  target_profile_ref:
    exact {target_profile_id, target_profile_version,
           target_profile_content_hash}
  contract_set_ref: ContractSetRefV1
  toolchain_lock_sha256: SHA-256
  fixture_id: fixture.runtime.ecs-data-oriented-core
  sample_policy: runtime_ecs_warmup_5x120s_median_p95_10m_soak_v1
  chunk_payload_candidates_bytes: [8192, 16384, 32768]
  scenario_ids:
    sequential_motion
    position_only_projection
    lifetime_only_scan
    structural_burst
    archetype_fragmentation
    query_cache_invalidation
  metric_set: runtime_ecs_data_oriented_metrics_v1
  correctness_oracle:
    runtime_ecs_semantic_publication_failure_atomicity_v1
  profile_hash: SHA-256

RuntimeDataOrientedQualificationCampaignV1
  campaign_version: 1
  profile_ref:
    exact {profile_version, target_profile_ref, contract_set_ref,
           toolchain_lock_sha256, profile_hash}
  artifact_candidate_binding_ref: content-addressed ref
  artifact_candidate_binding_sha256: SHA-256
  input_trace_ref: content-addressed ref
  input_trace_sha256: SHA-256
  sample_artifact_ref: content-addressed ref
  sample_artifact_sha256: SHA-256
  correctness_artifact_ref: content-addressed ref
  correctness_artifact_sha256: SHA-256
  result: pass | fail | infrastructure_error
  campaign_hash: SHA-256
```

The profile is a reusable measurement definition and therefore does not embed
a Product Candidate. The campaign binds one exact `ArtifactCandidateBindingV1`
from Product Plan. The profile's exact Target ref must be one member of that
binding's `target_profile_refs[]`; the Contract Set and Toolchain hashes must
be byte-equal to the binding's fields. The sample and correctness artifacts
both resolve to that same Candidate root、one profile Target、Contract Set、
Toolchain lock、fixture, and input trace. A missing or unequal binding is
`infrastructure_error`, never a performance pass.

One campaign sample artifact contains the deterministic warm-up、all five
120-second measured runs, and the separate 10-minute soak. An individual
120-second run is not a qualification campaign and cannot be referenced by the
Product Gate.

`sample_policy`、`metric_set`, and `correctness_oracle` are closed literals of
profile version 1, not unresolved Registry refs. Their exact members are the
sampling rule、mandatory metric list, and hard-predicate list below. A future
member change requires a profile version increase; prose or a caller-provided
name cannot extend any set.

`runtime_ecs_warmup_5x120s_median_p95_10m_soak_v1` expands to this exact
sampling matrix:

1. Each of the three layout candidates runs each of the six scenarios.
2. Each scenario／layout cell uses five fresh-process runs.
3. Each run discards ten deterministic scenario cycles, then measures exactly
   120 seconds.
4. Each metric's run P95 uses nearest rank `ceil(0.95 * N)` over all valid
   post-warm-up samples. The campaign value is the third value after sorting
   the five run P95 values.
5. Each layout candidate has one additional fresh-process 600-second composite
   soak after ten discarded warm-up cycles. The composite trace repeats the
   six scenarios in the declared `scenario_ids` order with fixed inputs.
6. The campaign therefore contains exactly 90 measured run results and three
   soak results. Its measured duration is exactly 12,600 seconds, excluding
   warm-up and setup.
7. Missing run、missing soak、process reuse、scenario reorder、sample
   substitution、NaN／infinite value, counter overflow, or environment drift
   makes the campaign `infrastructure_error`.

The scenarios use synthetic, bounded Component schemas representing Position、
Velocity、Lifetime, and cold metadata. They are test contracts, not new
Gameplay Component owners.

| Scenario | Required observation |
|---|---|
| `sequential_motion` | Position＋Velocity columns are traversed in contiguous row ranges |
| `position_only_projection` | Velocity、Lifetime, and cold metadata are not exposed to the callback |
| `lifetime_only_scan` | Lifetime traversal does not require Position／Velocity payload access |
| `structural_burst` | bounded create／destroy／add／remove cost and atomic failure behavior |
| `archetype_fragmentation` | archetype count、chunk occupancy、unused bytes、chunk transitions |
| `query_cache_invalidation` | cache hit、miss、rebuild, and invalidation after structural commit |

Mandatory metrics are:

- callback general-heap allocation count;
- callback upstream fallback count;
- reserved、committed、live, and peak bytes;
- chunk count、row capacity、occupied rows、unused payload bytes;
- archetype count and archetype fragmentation;
- selected rows、contiguous work units, and chunk transitions;
- exposed column bytes and useful selected payload bytes;
- query-cache hit、miss、rebuild, and invalidation counts;
- structural moved rows and structural copy bytes;
- handle resolve and lease-validation P50／P95／P99;
- scenario CPU P50／P95／P99;
- semantic result hash、publication hash, and failure atomicity.

Hardware cache misses、branch misses, and memory bandwidth are supplemental
metrics only when the Target Profile supports reproducible counters. A missing
hardware counter never becomes zero and never blocks a Target that has the
complete mandatory metric set.

Hard predicates are:

- callback general-heap allocation count = `0`;
- callback upstream fallback count = `0`;
- callback work unit crossing a chunk boundary = `0`;
- unselected or undeclared column access = `0`;
- semantic、publication, and failure oracle mismatch = `0`;
- stale handle、expired lease, or direct structural mutation accepted = `0`;
- required metric missing = `0`.

The existing Performance owner sampling remains unchanged: deterministic
warm-up, five 120-second runs, median of run P95, and a separate 10-minute
soak. Production and Mobile endurance continue to use their existing owner
rules.

Promotion compares a baseline campaign and candidate campaign whose profile
ref、Target、Contract Set、Toolchain lock、fixture、input trace, sample policy,
and correctness oracle are byte-equal. Their
`artifact_candidate_binding_ref` values are intentionally distinct and name
the before and after artifacts; metric and correctness artifacts within each
campaign remain bound to that campaign's one Candidate.

A new default path or Component split is promoted only when the candidate
campaign passes all hard predicates and either:

- improves integrated P95 by at least 5% and 0.20 ms while peak memory and
  allocation count do not regress by more than 5%; or
- improves peak memory by at least 15% while latency、allocation count、
  correctness、fault、load, and presentation remain within existing gates.

If neither candidate dominates, the current qualified layout remains selected.
No automatic layout switch occurs at Runtime.

The first clean implementation has no previously qualified Shipping ECS
candidate and therefore does not invent a zero baseline or claim an
improvement percentage. Initial qualification must pass the complete
characterization matrix and all hard predicates with the selected 16 KiB
layout. The baseline／candidate promotion rule applies only after that initial
layout has a qualified campaign.

## 10. Product Plan integration

The destination Active Product Definition adds these rows without treating
them as implemented or active.

```text
requirement.runtime.ecs-data-oriented-core
fixture.runtime.ecs-data-oriented-core
gate.product.phase-0-ecs-data-oriented-core
risk.product.ecs-data-oriented-regression
```

These are destination rows of the same approved Product Definition Migration
that applies `RuntimeEcsCanonicalizationChangeSetV1` and its Owner reference
migration. They are not inserted into the current source Definition while
Runtime ECS remains a `review` target and the current ECS Work Package owner is
Scheduling／Lifetime. The source Definition and its operational snapshot
remain byte-equal until the atomic migration is approved and applied.

### 10.1 Exact destination Registry changes

The destination Registry changes are closed:

| Registry | Exact addition or replacement |
|---|---|
| `RequirementRegistryV1` | `{requirement_id=requirement.runtime.ecs-data-oriented-core, owner_document_id=mirakan.arch.runtime-entity-component-system, verification_kind=runtime_ecs_data_oriented_qualification, failure_diagnostic_id=diagnostic.product.ecs-data-oriented-core-failed}` |
| `FixtureRegistryV1` | `{fixture_id=fixture.runtime.ecs-data-oriented-core, owner_document_id=mirakan.arch.runtime-entity-component-system, requirement_refs=[requirement.runtime.ecs-data-oriented-core], target_refs=[target.headless.host,target.windows.desktop,target.android.mobile,target.apple.mobile], minimum_duration_seconds=12600}` |
| `PhaseFixtureBindingRegistryV1` | `{gate_id=gate.product.phase-0-ecs-data-oriented-core, phase_id=phase.foundation, fixture_id=fixture.runtime.ecs-data-oriented-core, evaluated_requirement_refs=[requirement.runtime.ecs-data-oriented-core], target_refs=[target.headless.host], candidate_binding_policy_ref=policy.product.same-candidate.v1, freshness_policy_ref=policy.evidence.contract-ci.v1}` |
| `ProductPhaseRegistryV1` `phase.foundation` | append the new Requirement to `outcome_requirement_refs[]` and the new Gate to `exit_gate_refs[]`; `work_package_refs[]` is unchanged |
| `WorkPackageRegistryV1` | append `{kind=product_fixture, fixture_id=fixture.runtime.ecs-data-oriented-core}` to `provided_fixture_refs[]` of `wp.foundation.memory-pointers`, `wp.runtime.scheduling-core`, `wp.runtime.ecs-e0`, `wp.runtime.ecs-e1-storage`, and `wp.runtime.ecs-e2-query-mutation`; all other Fields remain unchanged except the coordinated Owner replacements below |

The Fixture target set is deliberately broader than the Phase 0 Gate target
set. The Headless Gate evaluates only `target.headless.host`; later Windows、
Android, and Apple qualification reruns the same Fixture on an allowed exact
Target and uses `policy.evidence.target-device.v1`. A Headless Receipt is never
substituted for those Target-specific runs.

The same destination Owner reference migration replaces
`owner_document_id=mirakan.arch.runtime-scheduling-lifetime` with
`mirakan.arch.runtime-entity-component-system` only for
`wp.runtime.ecs-e0`、`wp.runtime.ecs-e1-storage`, and
`wp.runtime.ecs-e2-query-mutation`. It does not change the owners of
`wp.foundation.memory-pointers` or `wp.runtime.scheduling-core`, add a Work
Package／Capability, or alter the existing Work Package dependency chain.

The destination risk row is:

| Field | Exact value |
|---|---|
| `risk_id` | `risk.product.ecs-data-oriented-regression` |
| `owner_document_id` | `mirakan.arch.runtime-entity-component-system` |
| `affected_work_package_refs[]` | `wp.foundation.memory-pointers; wp.runtime.scheduling-core; wp.runtime.ecs-e0; wp.runtime.ecs-e1-storage; wp.runtime.ecs-e2-query-mutation` |
| `trigger` | missing layout policy、dual Shipping layout、hot callback allocation／fallback、unbounded archetype growth、missing campaign cell／metric, or wrong-Target Receipt substitution |
| `likelihood` | `high` |
| `impact` | `critical` |
| `mitigation` | require the destination Phase 0 Gate、complete campaign、hard predicates, and fresh Target-specific reruns |
| `contingency` | reject the affected ECS Work Package transition and dependent Runtime activation; retain the last qualified layout without an alternate Shipping fallback |
| `monitor_gate_refs[]` | `gate.product.phase-0-ecs-data-oriented-core` |
| `genesis_state` | `open` |
| `revisit_gate_or_date` | `{kind=phase_gate, ref=gate.product.phase-0-ecs-data-oriented-core}` |

`requirement.runtime.ecs-data-oriented-core` is owned by Runtime ECS with
Performance／Capacity as qualification owner. It requires:

- accepted `RuntimeComponentLayoutPolicyV1` records for fixture Components;
- one `RuntimeArchetypeLayoutPlanV1` using `ecs_chunk_soa_v1`;
- query、lease, and structural contracts;
- the complete mandatory metric set;
- all hard predicates passing;
- 8／16／32 KiB characterization bound to the same Candidate;
- no Shipping AoS、sparse-set、object graph, or general-heap fallback path.

`fixture.runtime.ecs-data-oriented-core` is evaluated first as a Headless C0
fixture and is rerun by later Target-specific ECS qualification. Reuse means
rerunning the same 12,600-second campaign on the target Candidate; a Headless
Receipt is never reused as Windows、Android, or Apple performance evidence.

`gate.product.phase-0-ecs-data-oriented-core` uses
`policy.product.same-candidate.v1` and
`policy.evidence.contract-ci.v1`. Destination Phase 0 includes the new
Requirement and Gate. The Phase Fixture Gate evaluates only its declared Requirement、
fixture、Target、Candidate binding, and freshness because
`PhaseFixtureBindingRegistryV1` has no Work Package state Field. Phase 0 exit
separately requires every non-deferred Work Package listed by
`ProductPhaseRegistryV1.work_package_refs[]` to have a current `complete`
lifecycle head under the same active Product Definition. Phase 0's listed
Work Packages are non-deferred and include `wp.runtime.ecs-e1-storage` and
`wp.runtime.ecs-e2-query-mutation`. Work Package state is not smuggled into
the Gate schema.

Work Package responsibilities are:

| Work Package | Added completion responsibility |
|---|---|
| `wp.foundation.memory-pointers` | value-transfer policy、container layout Fields、static and negative Gates |
| `wp.runtime.ecs-e0` | type、owner、diagnostic, and Contract closure |
| `wp.runtime.ecs-e1-storage` | chunk SoA、layout policy、capacity、handle、fragmentation metrics |
| `wp.runtime.ecs-e2-query-mutation` | cached query、contiguous dispatch、allocation-free callback、deferred structural transaction |
| later `wp.runtime.ecs-e7-*` | rerun the qualified profile on the exact Target and device Evidence policy |

The exact `risk.product.ecs-data-oriented-regression` row above is the only
Product risk definition for this change. Prose in owner documents references
that row and does not redefine its trigger、severity、mitigation, or
containment.

## 11. Compatibility and clean break

The clean implementation applies to unreleased Engine-internal layout and API
surfaces. It does not authorize silent deletion of committed Project Source、
Asset provenance、published Saves, Native ABI consumers, distributed Packages,
or external API consumers.

Before applying the target contracts, Compatibility／Evolution requires a
complete `CompatibilityConsumerInventoryV1` for the existing ECS
canonicalization subject. Required discovery scopes remain:

- repository worktree;
- reachable Git history;
- release registry;
- distribution registry;
- Native ABI registry;
- external API registry.

The target `CompatibilityChangeSetV1` is
`source_preserving_recook` when every retained consumer can be rebuilt from
committed Source. It has:

```text
old_reader_policy = absent
old_writer_policy = absent
alias_policy = forbidden
source_preservation_policy = retain
regeneration_policy = full_recook | full_rebuild
rollback_policy = source_rebuild
```

The coordinated cutover removes from current boundaries:

- legacy or suffixless type aliases;
- object-address or pool-slot identity;
- pointer-backed inline Component payload;
- per-Entity virtual update storage;
- Runtime `shared_ptr` ownership;
- alternate Shipping AoS／sparse-set storage;
- direct mutation during iteration;
- old query cache entries and persisted row selections;
- dual package／Save／Replay projection of Runtime layout;
- old generated API signatures that violate the value-transfer policy.

Committed Source、Asset import documents, and approved Runtime entry documents
are the only regeneration inputs. Stale cache、old package bytes、raw Runtime
handles、chunk rows、addresses, and old generated bindings are prohibited
inputs.

If any retained release、published Save、Native ABI, or external consumer
cannot be rebuilt, clean break stops. A separately approved
`versioned_reader_migration` or `external_api_deprecation` must define a finite
reader window、target release、telemetry、rollback, and removal Gate. No
compatibility behavior is inferred or left indefinitely active.

## 12. Data and authority flow

```text
Domain Component schemas + Game System access manifests
  -> RuntimeComponentLayoutPolicyV1
  -> RuntimeArchetypeLayoutPlanV1
  -> Runtime World／Package capacity reservation
  -> query cache and contiguous chunk dispatch
  -> allocation-free generated callback
  -> deferred structural delta
  -> boundary validation and atomic publication
  -> sealed snapshot／Save／Replay／Debug projection

PointerContractV1 + MemoryContractV1
  + PointerMemoryConsumerBindingV1 + CppValueTransferPolicyV1
  -> generated C++ API and static rules
  -> Native Module／System callback bindings

RuntimeDataOrientedQualificationProfileV1
  -> same-Candidate metrics and correctness Receipt
  -> Product Phase Gate／Target qualification
  -> Capability activation decision
```

No arrow runs from Debug、AI、Save、Replay, or Presentation back to live memory
layout or authoritative Component state.

## 13. Failure and diagnostics

The canonical owners add the following exact Diagnostics in the same Contract
or Product Definition transaction as their consumers. The code is the
Naming／Project Layout transformation of the ID; alternate underscore codes or
local aliases are not retained.

| Owner | Diagnostic ID／code | Required typed arguments | Required result |
|---|---|---|---|
| `mirakan.arch.memory-pointers` | `diagnostic.memory.value-transfer-binding-missing`／`MIRAKAN-MEMORY-VALUE-TRANSFER-BINDING-MISSING` | `api_contract_id; api_contract_version; subject; ordinal?` | Build／Contract compile failure |
| `mirakan.arch.memory-pointers` | `diagnostic.memory.value-transfer-invalid`／`MIRAKAN-MEMORY-VALUE-TRANSFER-INVALID` | `api_contract_id; subject; ordinal?; rule_id` | Static Gate failure |
| `mirakan.arch.memory-pointers` | `diagnostic.memory.hot-callback-allocation`／`MIRAKAN-MEMORY-HOT-CALLBACK-ALLOCATION` | `campaign_hash; scenario_id; payload_bytes; observed_count` | Performance Gate failure; callback result not published |
| `mirakan.arch.memory-pointers` | `diagnostic.memory.hot-callback-upstream-fallback`／`MIRAKAN-MEMORY-HOT-CALLBACK-UPSTREAM-FALLBACK` | `campaign_hash; scenario_id; payload_bytes; observed_count` | Performance Gate failure; callback result not published |
| `mirakan.arch.runtime-entity-component-system` | `diagnostic.runtime.ecs-component-schema-revision-required`／`MIRAKAN-RUNTIME-ECS-COMPONENT-SCHEMA-REVISION-REQUIRED` | `component_schema_ref; access_cohort_hash` | `component_schema_revision_required`; no silent split |
| `mirakan.arch.runtime-entity-component-system` | `diagnostic.runtime.ecs-chunk-capacity-zero`／`MIRAKAN-RUNTIME-ECS-CHUNK-CAPACITY-ZERO` | `archetype_id; column_layout_hash; payload_bytes` | Cook／Contract compile failure |
| `mirakan.arch.runtime-entity-component-system` | `diagnostic.runtime.ecs-archetype-permutation-unbounded`／`MIRAKAN-RUNTIME-ECS-ARCHETYPE-PERMUTATION-UNBOUNDED` | `component_schema_ref; observed_count; declared_bound` | Layout qualification failure |
| `mirakan.arch.runtime-entity-component-system` | `diagnostic.runtime.ecs-query-cache-invalid`／`MIRAKAN-RUNTIME-ECS-QUERY-CACHE-INVALID` | `query_ref; invalid_member_kind` | Contract／negative fixture failure |
| `mirakan.arch.runtime-entity-component-system` | `diagnostic.runtime.ecs-structural-mutation-during-iteration`／`MIRAKAN-RUNTIME-ECS-STRUCTURAL-MUTATION-DURING-ITERATION` | `system_ref; operation_kind; logical_work_id` | Typed rejection |
| `mirakan.arch.runtime-entity-component-system` | `diagnostic.runtime.ecs-structural-capacity-exceeded`／`MIRAKAN-RUNTIME-ECS-STRUCTURAL-CAPACITY-EXCEEDED` | `boundary_ref; required_bytes; available_bytes` | Previous World remains published |
| `mirakan.arch.runtime-performance-capacity` | `diagnostic.performance.ecs-required-metric-missing`／`MIRAKAN-PERFORMANCE-ECS-REQUIRED-METRIC-MISSING` | `campaign_hash; scenario_id; payload_bytes; metric_id` | Qualification failure |
| `mirakan.arch.product-plan` | `diagnostic.product.ecs-target-receipt-mismatch`／`MIRAKAN-PRODUCT-ECS-TARGET-RECEIPT-MISMATCH` | `campaign_hash; expected_target_ref; actual_target_ref` | Product Gate failure |
| `mirakan.arch.compatibility-evolution` | `diagnostic.compatibility.ecs-consumer-inventory-unresolved`／`MIRAKAN-COMPATIBILITY-ECS-CONSUMER-INVENTORY-UNRESOLVED` | `change_set_ref; discovery_scope; discovery_state` | Clean-break application rejected |
| `mirakan.arch.compatibility-evolution` | `diagnostic.compatibility.ecs-retained-external-consumer`／`MIRAKAN-COMPATIBILITY-ECS-RETAINED-EXTERNAL-CONSUMER` | `change_set_ref; consumer_class; consumer_ref` | Switch to separately approved finite migration |
| `mirakan.arch.runtime-entity-component-system` | `diagnostic.product.ecs-data-oriented-core-failed`／`MIRAKAN-PRODUCT-ECS-DATA-ORIENTED-CORE-FAILED` | `requirement_id; campaign_hash; target_profile_ref; failed_diagnostic_refs[1..64]` | Aggregate Requirement evaluation fails |

Free-form warning text cannot convert any failure into success.

## 14. Coordinated document changes

### 14.1 Canonical documents to update

| Document | Required change |
|---|---|
| Product Plan | Add Requirement、Fixture、Gate、risk, and Work Package completion bindings |
| Core Architecture | Route C++ transfer／container decisions to Memory and ECS layout decisions to Runtime ECS |
| Executable Contracts | Add the fourth planned Foundation type and materialization／closure tests |
| Memory／Pointers | Own `CppValueTransferPolicyV1`, extended `MemoryContractV1`, static rules, metrics, and qualification |
| Compatibility／Evolution | Bind value-transfer／layout cutover to ECS Consumer Inventory and regeneration |
| Runtime ECS | Own `RuntimeComponentLayoutPolicyV1`, access-cohort review, allocation-free callback predicates, and qualification refs |
| Performance／Capacity | Own `RuntimeDataOrientedQualificationProfileV1`, mandatory metrics, sampling, and promotion |
| Runtime ECS Decision | Record the coordinated target decision and updated official evidence without activating it |

### 14.2 Direct consumers to update

| Document | Required binding |
|---|---|
| C++23 Modules | Generated and First-party C++ obey the value-transfer static Gate |
| Gameplay Programming Model | Component schema and Game System manifests supply the access-cohort inputs |
| Native Game Module | C++ adapters and persistent factory calls consume the transfer／memory policies; fixed C ABI shapes remain owned here and expose no C++ reference／STL type |
| Scheduling／Lifetime | Query dispatch reserves before callback and structural deltas apply only at the existing boundary |
| Runtime Package | World capacity and layout plans bind exact ECS／Memory Contract refs |
| Debugging／Observability／Replay | Carry mandatory data-oriented metrics without addresses、rows, or live leases |

### 14.3 Documents verified without normative duplication

Architecture Governance already requires the target ECS owner transfer to
include all reachable MCD、Diagnostic、Game System、generated binding, and
retained artifact refs. AI Verification already owns Requirement fulfillment
and Receipt freshness. Toolchain owns exact Compiler／SDK versions. Project
State、Asset Lifecycle、Persistence／Save、Simulation、Rendering、Platform, and
Pack documents already keep Runtime pointers／rows out of persistent identity
or consume Performance／Memory owner rules.

These documents receive edits only if final cross-document validation finds an
incorrect name、missing direct dependency, or conflicting owner statement.
They do not receive copied policy tables.

The Architecture Index is updated only if a responsibility summary or routing
entry changes. It never becomes a Contract owner.

## 15. Verification

The coordinated document update is complete only when all checks pass.

1. All 52 Architecture document IDs remain unique and all relative links
   resolve.
2. `CppValueTransferPolicyV1` has one canonical definition and every generated
   C++ consumer references it.
3. All prose that says the Pointer／Memory set has three types is updated to the
   four-type set.
4. `MemoryContractV1` Field lists match across Memory、Executable Contracts,
   Product Requirement, and generated closure descriptions.
5. `RuntimeComponentLayoutPolicyV1` and
   `RuntimeDataOrientedQualificationProfileV1` each have one canonical owner.
6. Product Requirement、Fixture、Gate、Work Package、Capability dependency,
   and risk refs resolve bidirectionally with no orphan ID.
7. Destination Phase 0 contains the new Requirement and Gate; the Gate remains
   a pure `PhaseFixtureBindingRegistryV1` row, while Phase exit independently
   requires current `complete` lifecycle heads for every listed Work Package,
   including E1 and E2.
8. Target-specific qualification requires a fresh Target Receipt and cannot
   reuse Headless evidence.
9. Hot callback allocation and fallback predicates are exact zero in Memory、
   ECS、Performance, and Product acceptance.
10. ECS qualification contains all six scenarios and every mandatory metric.
11. 8／16／32 KiB comparison remains benchmark evidence and does not create
    three Shipping layouts.
12. AoS、sparse-set、old alias、old reader、old writer, and direct structural
    mutation are absent from current Shipping success paths.
13. Consumer Inventory remains fail-closed while any scope is unresolved.
14. Existing User data protection is not weakened by the internal clean break.
15. Official external references are primary sources and vendor-specific
    values are labelled as comparison evidence rather than Miraikanai
    authority.
16. The source Product Definition remains byte-equal until the coordinated
    ECS canonicalization、Owner reference migration, and destination Product
    Definition Migration are approved and atomically applied.
17. Each campaign contains exactly 90 measured run results、three soak
    results, and 12,600 measured seconds; no missing cell is normalized to
    zero or pass.
18. Initial qualification does not fabricate a baseline; later promotion
    compares two complete campaigns with byte-equal measurement inputs.
19. Every Diagnostic ID、code、owner, and argument schema in §13 is registered
    once, and ID-to-code transformation is one-to-one.
20. `MemoryContractV1` retains every existing Field, has exactly one
    `capacity_source` Field with the closed values in §6, and adds exactly the
    six declared layout／access Fields.
21. Generated C ABI declarations contain no C++ reference、`std::span`,
    STL／PMR type、`Result<T>`, exception, or owner wrapper; their generated
    C++ adapters still satisfy the transfer and pointer policies.
22. Git diff review shows no unrelated edits introduced by this work.

## 16. Official external basis

The following primary sources justify the selected principles. They do not
define Miraikanai APIs、Schemas、budgets, or activation state.

- C++ Core Guidelines, parameter passing and ownership:
  <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines>
- Current C++ draft, `std::vector` contiguous storage、capacity, and
  invalidation:
  <https://eel.is/c++draft/vector>
- Current C++ draft, copy／move elision:
  <https://eel.is/c++draft/class.copy.elision>
- Current C++ draft, `std::span`:
  <https://eel.is/c++draft/views.span>
- Current C++ draft, polymorphic memory resources:
  <https://eel.is/c++draft/mem.res>
- Microsoft `/Zc:nrvo` documentation:
  <https://learn.microsoft.com/en-us/cpp/build/reference/zc-nrvo>
- Unity Entities 1.4.8 archetypes and chunk arrays:
  <https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/concepts-archetypes.html>
- Unity Entities 1.4.8 chunk fragmentation guidance:
  <https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/performance-chunk-allocations.html>
- Unity Entities 1.4.8 structural changes and sync points:
  <https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/concepts-structural-changes.html>
- Unity Entities 1.4.8 Entity Command Buffer guidance:
  <https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/systems-entity-command-buffers.html>
- Unreal Engine memory and CPU performance considerations:
  <https://dev.epicgames.com/documentation/unreal-engine/common-memory-and-cpu-performance-considerations-in-unreal-engine>
- Unreal Engine performance profiling and contiguous-memory discussion:
  <https://dev.epicgames.com/documentation/unreal-engine/introduction-to-performance-profiling-and-configuration-in-unreal-engine>
- EnTT Entity／Component／System documentation, used only as a sparse-set
  comparison:
  <https://github.com/skypjack/entt/blob/v3.16.0/docs/md/entity.md>

## 17. Non-goals

- Importing Unity、Unreal Engine, or EnTT runtime code or public APIs.
- Treating 16 KiB as a universal CPU cache optimum.
- Creating a general-purpose custom allocator before integrated evidence.
- Pooling every object or banning the System allocator from non-hot Tool paths.
- Splitting every Component field into an independent Component.
- Maintaining AoS and SoA as dual Shipping implementations.
- Persisting chunk、row、pointer、lease, or allocator state.
- Using `std::move` as a general performance annotation.
- Replacing ordinary return values with output parameters.
- Activating a new MCD Operation、Provider Tool, Capability, or Product state.
- Deleting retained User data or published consumers without Inventory and an
  approved migration decision.
