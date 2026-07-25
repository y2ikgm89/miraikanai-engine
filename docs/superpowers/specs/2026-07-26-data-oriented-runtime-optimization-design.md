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

Every planned top-level record in this design has a Stable ID、positive
version, and self-excluding SHA-256 content hash. Canonical bytes use the MCD
canonical encoding owned by Executable Contracts. Product signed wrappers
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

Every reference resolves by ID、version, and content hash inside the declared
Contract Set or Product Definition. Bare ID、`latest`、ambient current,
same-name substitution, and same-ID／version different-hash are invalid. A
schema shown inline as `exact {...}` is a nested reference shape and does not
create an additional standalone Type or owner.

## 5. C++ value-transfer contract

Memory／Pointers owns the following planned type.

```text
CppValueTransferPolicyV1
  policy_id: StableId
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
rule suppression scope. Generated public APIs and Native ABI do not inherit a
private exception.

Return-by-value relies on standard copy elision and NRVO. Code returns the
named local directly. Explicit move remains valid for ownership hand-off,
container insertion, and a declared sink after the last pre-move access.

## 6. Generic container and allocation contract

`MemoryContractV1` is extended before materialization with the following
required Fields for every container owner and allocation site.

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
  policy_id: StableId
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
    exact {profile_id, profile_version, profile_hash}
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
  profile_id: StableId
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
  campaign_id: StableId
  campaign_version: 1
  profile_ref:
    exact {profile_id, profile_version, profile_hash}
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

## 10. Product Plan integration

Product Plan adds these rows without treating them as implemented or active.

```text
requirement.runtime.ecs-data-oriented-core
fixture.runtime.ecs-data-oriented-core
gate.product.phase-0-ecs-data-oriented-core
risk.product.ecs-data-oriented-regression
```

`requirement.runtime.ecs-data-oriented-core` is owned by Runtime ECS with
Performance／Capacity as qualification owner. It requires:

- accepted `RuntimeComponentLayoutPolicyV1` records for fixture Components;
- one `RuntimeArchetypeLayoutPlanV1` using `ecs_chunk_soa_v1`;
- query、lease, and structural contracts;
- the complete mandatory metric set;
- all hard predicates passing;
- 8／16／32 KiB characterization bound to the same Candidate;
- no Shipping AoS、sparse-set、object graph, or general-heap fallback path.

`fixture.runtime.ecs-data-oriented-core` is a Headless C0 fixture and is also
reused as an input by later Target-specific ECS qualification. Reuse means
rerunning the same semantic scenario on the target Candidate; a Headless
Receipt is never reused as Windows、Android, or Apple performance evidence.

`gate.product.phase-0-ecs-data-oriented-core` uses
`policy.product.same-candidate.v1` and
`policy.evidence.contract-ci.v1`. Phase 0 includes the new Requirement and
Gate. The Phase Fixture Gate evaluates only its declared Requirement、
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

`risk.product.ecs-data-oriented-regression` is `high／critical`. Its trigger is
any missing layout policy、dual Shipping layout、hot allocation、unbounded
archetype growth、missing metric, or target Receipt substitution. Mitigation is
the new Phase 0 Gate plus Target-specific reruns. Containment rejects the
affected ECS Work Package transition and all dependent Runtime activation.

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

The exact Diagnostic IDs are added by their canonical owners during Contract
materialization. The design requires one stable Diagnostic for each condition:

| Condition | Required result |
|---|---|
| Missing value-transfer binding | Build／Contract compile failure |
| Invalid pass form or move use | Static Gate failure |
| Hot callback allocation or fallback | Performance Gate failure; callback result not published |
| Component schema has incompatible access cohorts | `component_schema_revision_required`; no silent split |
| Chunk capacity is zero | Cook／Contract compile failure |
| Unbounded archetype or tag permutation | Layout qualification failure |
| Query cache contains row or address state | Contract／negative fixture failure |
| Direct structural mutation during iteration | Typed rejection |
| Structural capacity failure | Previous World remains published |
| Required metric missing | Qualification failure |
| Wrong Target Receipt reused | Product Gate failure |
| Consumer Inventory unresolved | Clean-break application rejected |
| Retained external consumer discovered | Switch to separately approved finite migration |

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
| Native Game Module | Public generated APIs and persistent factory calls consume the transfer／memory policies |
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
7. Phase 0 contains the new Requirement and Gate; the Gate remains a pure
   `PhaseFixtureBindingRegistryV1` row, while Phase exit independently requires
   current `complete` lifecycle heads for every listed Work Package, including
   E1 and E2.
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
16. Git diff review shows no unrelated edits introduced by this work.

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
