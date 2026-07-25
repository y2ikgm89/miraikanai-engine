# Foundation C++ Value Transfer and Memory Contract Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Update the Foundation architecture plans so one closed C++ value-transfer policy, one extended memory-layout contract, and one unchanged fixed C ABI boundary can be implemented without aliases or competing definitions.

**Architecture:** Memory／Pointers is the sole semantic owner of value transfer, pointer lifetime, container layout, allocation, and the four new Memory diagnostics. Executable Contracts owns MCD envelope and materialization only; Core and C++23 route to that owner, while Native Game Module keeps the fixed C ABI and applies the policy only in its generated C++ adapter.

**Tech Stack:** Markdown architecture contracts, C++23 vocabulary, MCD canonical records, PowerShell 7, ripgrep, Git.

## Global Constraints

- Normative design: `docs/superpowers/specs/2026-07-26-data-oriented-runtime-optimization-design.md`.
- Orchestration entry:
  `docs/superpowers/plans/2026-07-26-data-oriented-runtime-optimization-master-plan.md`.
- Execution requires an isolated worktree or an explicitly approved baseline in
  which every file listed by this plan is tracked and clean. The current
  workspace contains pre-existing Architecture edits; do not absorb them into
  this plan's commits.
- Do not edit or stage unrelated dirty-worktree changes.
- Before each commit, require `git diff --cached --name-only` to contain only that task's listed files.
- Use `git commit --only -- <paths>`; never commit all staged or working-tree changes.
- No pre-1.0 alias, suffixless schema alias, dual generated signature, dual manifest, old reader, old writer, or C++ Header fallback.
- `CppValueTransferPolicyV1` is the fourth Pointer／Memory definition-closure type.
- `MemoryContractV1` retains every existing Field, has exactly one `capacity_source`, and adds exactly six layout／access Fields.
- Cheap input-by-value means scalar, enum, typed generation handle, or trivially copyable non-owner value of at most two locked Target ABI words.
- Other synchronous input uses `const T&`; ranges use value-passed `std::span`; ordinary output uses return value; unconditional non-owner sinks use `T&&`; unique owners use value.
- Reject `const T&&`, `std::move` from `const`, conditional sink moves, moved-from reuse except destroy／assign, and `return std::move(local)` when copy elision applies.
- C ABI declarations contain no C++ reference, `std::span`, STL／PMR type, `Result<T>`, exception, or owner wrapper.
- Do not add a dependency or change Toolchain pins.

## Execution Preflight

Before Task 1, use `superpowers:using-git-worktrees` and validate the selected
implementation worktree:

```powershell
$targets = @(
  'docs/architecture/02-foundation/core-architecture.md',
  'docs/architecture/02-foundation/executable-contracts.md',
  'docs/architecture/02-foundation/memory-pointers.md',
  'docs/architecture/02-foundation/cpp23-modules.md',
  'docs/architecture/03-authoring/native-game-module.md'
)
foreach ($path in $targets) {
  git ls-files --error-unmatch -- $path *> $null
  if ($LASTEXITCODE -ne 0) { throw "Target is not in the approved baseline: $path" }
  if (git status --porcelain -- $path) { throw "Target has pre-existing changes: $path" }
}
if (git diff --cached --name-only) { throw 'Index is not empty.' }
```

If this fails, stop before editing. The user must identify or authorize a
baseline commit containing the intended current Architecture state; never
create that baseline by committing pre-existing changes on their behalf.

## File Map

| File | Responsibility in this plan |
|---|---|
| `docs/architecture/02-foundation/core-architecture.md` | Cross-layer routing and clean-break invariant |
| `docs/architecture/02-foundation/executable-contracts.md` | Four-type closure, root-first materialization, generated projections |
| `docs/architecture/02-foundation/memory-pointers.md` | Canonical schemas, rules, diagnostics, telemetry, qualification |
| `docs/architecture/02-foundation/cpp23-modules.md` | Generated／First-party C++ static gate consumer |
| `docs/architecture/03-authoring/native-game-module.md` | Fixed C ABI and generated C++ adapter consumer |

---

### Task 1: Route Core C++ and data-layout decisions to their canonical owners

**Files:**
- Modify: `docs/architecture/02-foundation/core-architecture.md:21-25`
- Modify: `docs/architecture/02-foundation/core-architecture.md:97-106`
- Modify: `docs/architecture/02-foundation/core-architecture.md:723-731`

**Interfaces:**
- Consumes: approved clean-break rule and the owner split in design §§2, 4, 5, and 6.
- Produces: Core routing statements; no duplicate Field list or numeric threshold.

- [ ] **Step 1: Run the pre-edit contract assertion**

```powershell
$path = 'docs/architecture/02-foundation/core-architecture.md'
if (rg -q 'CppValueTransferPolicyV1' $path) {
  throw 'Precondition failed: Core already contains the new routing token.'
}
Write-Error 'Expected failure: Core does not yet route C++ value transfer.'
```

Expected: FAIL with `Core does not yet route C++ value transfer`.

- [ ] **Step 2: Add exact owner routing without copying schemas**

Under `## 2. 後方互換性を持たないClean実装`, add:

```markdown
Data-oriented Runtime最適化も同じclean-break規則に従う。C++ value transferとgeneric container／allocationは[Memory／Pointers](memory-pointers.md)、Component列、archetype、query dispatch、structural transactionは[Runtime ECS](../04-runtime/entity-component-system.md)、共通measurementとpromotionは[Performance／Capacity](../04-runtime/performance-capacity.md)だけが所有する。CoreはそれらのField、threshold、Diagnosticを複写しない。旧C++ signature、旧layout、AoS／sparse-set Shipping fallback、dual reader／writerを同じcurrent境界へ残さない。
```

Under `## 8. C++とModule境界`, add:

```markdown
Generated／First-party C++ APIは[Memory／Pointers](memory-pointers.md)の`CppValueTransferPolicyV1`へexactに閉じる。Module、Header、Native adapterが独自のby-value／`const T&`／`T&&`／bounded-view規則を持たず、C ABIは[Native Game Module](../03-authoring/native-game-module.md)の固定幅値、opaque handle、caller-owned bufferへ分離する。
```

Under `## 12. Test、CI、AI生成物`, add:

```markdown
CIは生成C++ signature、Pointer／Memory manifest、C ABI adapterが同じContract Set、Target Profile、Toolchain lockの`CppValueTransferPolicyV1`へ解決することを検査する。missing／extra binding、別Target policy、C ABIへのC++ reference／STL流出、旧signature aliasをBuild failureにする。
```

- [ ] **Step 3: Verify Core routes but does not redefine**

```powershell
$path = 'docs/architecture/02-foundation/core-architecture.md'
$count = (rg -c 'CppValueTransferPolicyV1' $path)
if ($count -ne 2) { throw "Expected 2 routing references, got $count" }
if (rg -q 'target_abi_facts:|transfer_form:|storage_layout:' $path) {
  throw 'Core duplicated an owner schema.'
}
git diff --check -- $path
```

Expected: PASS; two routing references and zero schema definitions.

- [ ] **Step 4: Review and commit only Core**

```powershell
$path = 'docs/architecture/02-foundation/core-architecture.md'
git diff -- $path
git add -- $path
$staged = @(git diff --cached --name-only)
if ($staged.Count -ne 1 -or $staged[0] -ne $path) { throw "Unexpected staged paths: $($staged -join ', ')" }
git commit --only -m "docs: route data-oriented foundation contracts" -- $path
```

Expected: one-file commit.

---

### Task 2: Extend the Executable Contracts definition closure to four types

**Files:**
- Modify: `docs/architecture/02-foundation/executable-contracts.md:276-299`
- Modify: `docs/architecture/02-foundation/executable-contracts.md:1889-1914`
- Modify: `docs/architecture/02-foundation/executable-contracts.md:1982-1998`
- Modify: `docs/architecture/02-foundation/executable-contracts.md:2518-2564`

**Interfaces:**
- Consumes: `CppValueTransferPolicyV1`, `PointerContractV1`, `MemoryContractV1`, `PointerMemoryConsumerBindingV1`.
- Produces: exact four-type local closure, root-first instance materialization, four generated manifests, closure tests.

- [ ] **Step 1: Prove the current closure is still three-type**

```powershell
$path = 'docs/architecture/02-foundation/executable-contracts.md'
if (-not (rg -q 'PointerContractV1.*MemoryContractV1.*PointerMemoryConsumerBindingV1' $path)) {
  throw 'Existing three-type closure was not found.'
}
if (rg -q 'CppValueTransferPolicyManifest\.bin' $path) {
  throw 'Precondition failed: fourth manifest already exists.'
}
Write-Error 'Expected failure: four-type closure is not materialized.'
```

Expected: FAIL with `four-type closure is not materialized`.

- [ ] **Step 2: Replace §7.1.1 with an exact four-type closure**

Keep its existing owner and activation disclaimers, then make the normative set:

```text
PointerMemoryDefinitionClosureV1
  local_type_refs[4]:
    PointerContractV1
    MemoryContractV1
    PointerMemoryConsumerBindingV1
    CppValueTransferPolicyV1
  local_type_refs are strict-sorted unique McdContractLocalRefV1(kind=type)
  closure_hash: SHA-256
```

Add these rules immediately below:

```markdown
四TypeのField意味は[Memory／Pointers](memory-pointers.md)だけが所有する。本書はMCD共通Envelope、local record、member hash、Contract Set root、external `McdContractRefV1(kind=type)`へのmaterializationを所有し、Field一覧を再定義しない。

Contract Setは四Type schemaを先に閉じてrootを確定する。`CppValueTransferPolicyV1` instanceはそのimmutable root、exact Target Profile、Toolchain lockに対して後からmaterializeし、自身を同じContract Set preimageへ戻さない。missing／extra／duplicate Type、root確定前instance、別root Type ref、same logical key／version different hashをcompile errorにする。
```

- [ ] **Step 3: Extend compiler and C++ projection outputs**

In `## 14. Contract compiler` and `### 17.1 C++23`, require this exact generated set:

```text
PointerContractManifest.bin
MemoryContractManifest.bin
PointerMemoryConsumerBindingManifest.bin
CppValueTransferPolicyManifest.bin
```

Add:

```markdown
Compilerはsource API contractの`callable_key`とparameter／return subjectを全量列挙し、`CppValueTransferPolicyV1.bindings[]`と正逆set equalityを検査してからC++ signatureを生成する。生成signature hashをpolicy inputへ戻さず、source API contract hashをkeyにする。C ABI projectionはC++ signature projectionと別branchであり、Native Game Module ownerの固定shapeだけを生成する。
```

- [ ] **Step 4: Add exact negative closure tests to Definition of Done**

Append these checks to `## 22. Contract compilerのDefinition of Done`:

```markdown
- Pointer／Memory closureがexact四Typeで、三Type旧closure、五Type以上、missing／extra／duplicate、wrong kind、cross-root refを拒否する。
- source API subject集合とvalue-transfer binding集合がexact equalityで、unbound parameter／return、orphan binding、ordinal重複を拒否する。
- Contract Set root確定前のpolicy instance、instanceからrootへのback-edge、generated signature hashをinput keyにするcycleを拒否する。
- C++ Module／Header projectionとC ABI projectionを分離し、C ABIへreference、`std::span`、STL／PMR、`Result<T>`、exception、owner wrapperを出力しない。
```

- [ ] **Step 5: Remove stale three-item wording in this file**

```powershell
$path = 'docs/architecture/02-foundation/executable-contracts.md'
rg -n '三Contract|三種manifest|三Type|three' $path
```

For every result referring to Pointer／Memory closure or manifests, replace the cardinality with exact four and list all four names. Do not alter unrelated three-member contracts.

- [ ] **Step 6: Verify and commit only Executable Contracts**

```powershell
$path = 'docs/architecture/02-foundation/executable-contracts.md'
foreach ($token in @(
  'CppValueTransferPolicyV1',
  'CppValueTransferPolicyManifest.bin',
  'local_type_refs[4]',
  'source API subject集合とvalue-transfer binding集合'
)) {
  if (-not (rg -q --fixed-strings $token $path)) { throw "Missing token: $token" }
}
git diff --check -- $path
git add -- $path
$staged = @(git diff --cached --name-only)
if ($staged.Count -ne 1 -or $staged[0] -ne $path) { throw "Unexpected staged paths: $($staged -join ', ')" }
git commit --only -m "docs: add value transfer definition closure" -- $path
```

Expected: one-file commit and exact four-type closure.

---

### Task 3: Make Memory／Pointers the complete semantic owner

**Files:**
- Modify: `docs/architecture/02-foundation/memory-pointers.md:56-85`
- Modify: `docs/architecture/02-foundation/memory-pointers.md:246-299`
- Modify: `docs/architecture/02-foundation/memory-pointers.md:300-392`
- Modify: `docs/architecture/02-foundation/memory-pointers.md:393-407`

**Interfaces:**
- Consumes: exact Target Profile, Toolchain lock, `McdContractRefV1(kind=type)`, existing pointer／memory contracts.
- Produces: canonical `CppValueTransferPolicyV1`, extended `MemoryContractV1`, four Diagnostics, metrics, tests, and four manifests.

- [ ] **Step 1: Run the missing-schema assertion**

```powershell
$path = 'docs/architecture/02-foundation/memory-pointers.md'
foreach ($token in @('storage_layout','element_storage','access_pattern','growth_policy','address_stability','hot_path')) {
  $pattern = '^\s+' + [regex]::Escape($token) + ':'
  if (rg -q $pattern $path) { throw "Precondition failed: $token already present" }
}
Write-Error 'Expected failure: MemoryContractV1 lacks the six layout/access Fields.'
```

Expected: FAIL with the six-Field message.

- [ ] **Step 2: Replace §6.2 with the full retained-plus-extended schema**

Use this exact Field set:

```text
MemoryContractV1
  contract_id
  charge_domain_id
  memory_resource_class
  allocation_policy:
    system | monotonic_arena | size_class_pool
    | streaming_cache | external_adapter
  alignment
  capacity_source:
    profile | cooked_artifact | fixed_abi | measured_p999_headroom
  hard_limit_bytes
  steady_state_allocation_count
  allowed_phases[]
  allowed_threads[]
  reset_or_release_event
  upstream_fallback: forbidden | budgeted_system
  oom_behavior
  telemetry_tag_id
  source_owner
  storage_layout:
    contiguous_inline | contiguous_handle | chunk_column
    | ring | segmented | node_based | external
  element_storage: inline_value | typed_handle | external_adapter
  access_pattern:
    sequential | indexed | sparse_lookup | producer_consumer | infrequent
  growth_policy:
    pre_reserved_fixed | boundary_grow | budgeted_non_hot | forbidden
  address_stability:
    not_required | scope_bound | handle_resolved
  hot_path: bool
```

State explicitly that all fifteen existing Fields are retained,
`capacity_source` remains one Field whose prose choices become the closed enum,
and the final six layout／access Fields are additions.

- [ ] **Step 3: Add the closed generic container and allocation rules**

Immediately after `MemoryContractV1`, add these ten normative rules:

```markdown
1. `std::vector<T>` is the default dynamic container for non-ECS contiguous inline values only when the default System memory resource is allowed by the exact `MemoryContractV1`.
2. `std::pmr::vector<T>` is allowed only with a Composition Root-injected resource whose lifetime exceeds the container. Never change the process-global default PMR resource.
3. ECS columns use ECS-private chunk storage and generated bounded views. They are not independent vectors with independent allocations.
4. `std::vector<Handle<T>>` and `std::vector<T*>` provide contiguous references only; they never satisfy `storage_layout=contiguous_inline`.
5. Reserve known cardinality before entering a hot phase. Calling `reserve()` inside a hot callback is a Contract failure even if no reallocation occurs.
6. A pointer、reference、iterator, or span invalidated by reallocation never crosses its owning scope or structural boundary.
7. `storage_layout=node_based` is prohibited for sequential hot traversal. It is allowed only for infrequent control-plane lookup or a measured exception with integrated Evidence.
8. Pooling is limited to private fixed-size objects with stable churn and measured improvement. Do not pool a cheap C++ value without Evidence.
9. Frame、RenderFrame, and Job scratch use bounded arenas reset at the existing lifetime boundary. Exhaustion never falls through to the general heap.
10. Hot callback general-heap allocation count and upstream fallback count are both exactly `0`.
```

Add a compact decision table mapping each container owner to
`storage_layout`, `element_storage`, `access_pattern`, `capacity_source`,
`growth_policy`, `address_stability`, `hot_path`, and the owning
`MemoryContractV1.contract_id`. Do not prescribe one container globally
without this per-owner record.

- [ ] **Step 4: Add `### 6.3 CppValueTransferPolicyV1`**

Insert this exact schema:

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

`callable_key` identifies the source API contract and never hashes the
generated C++ signature. `pointer_contract_id` resolves exactly once within
`contract_set_ref`; missing, multiple, or cross-Contract-set resolution is a
compile failure. ABI facts come only from the exact Target／Toolchain and are
verified against `sizeof`, `alignof`, C++ traits, data model, word size, and
pointer size before materialization.

Compute each `binding_hash` as:

```text
SHA-256(
  ASCII "MIRAKAN_CPP_VALUE_TRANSFER_BINDING_V1"
  || uint32_be(len(canonical binding bytes excluding binding_hash))
  || canonical binding bytes excluding binding_hash
)
```

Strict-sort and deduplicate bindings by
`{callable_key canonical bytes, subject enum, ordinal presence, ordinal}`, then
compute `policy_hash` with the same framing and ASCII domain
`MIRAKAN_CPP_VALUE_TRANSFER_POLICY_V1`, excluding only `policy_hash`.
Materialize instances only after the immutable Contract Set root exists; never
insert an instance whose `contract_set_ref` points back into that root's
preimage.

Include this closed rule table:

| Subject semantics | C++ form |
|---|---|
| scalar／enum／typed generation handle input | value |
| trivially copyable non-owner input up to two Target ABI words | value |
| other synchronous input | `const T&` |
| input/output | `T&` |
| bounded sequence | value-passed `std::span<const T>` or `std::span<T>` |
| ordinary output | return value or existing `Result<T>` |
| move-only unique owner sink | value |
| unconditional non-owner sink | `T&&`, moved exactly once |

Add this exact static rejection list:

```text
const T&&
return std::move(local) when local is copy-elision eligible
std::move from a const object
conditional move from a sink
moved-from access other than destruction or assignment
by-value input of an unqualified nontrivial type
const T& for scalar, enum, or typed handle without an approved measured exception
non-const reference parameter that is never written
output parameter where an ordinary return value is sufficient
shared_ptr parameter that neither expresses nor retains shared ownership
```

Return-by-value relies on copy elision and NRVO, so return a named local
directly. Explicit move remains valid for ownership hand-off, container
insertion, and a declared sink after the final pre-move access. A private
measured exception requires an ADR containing the exact Target Profile,
fixture, baseline, improvement, and static suppression scope; generated public
APIs never inherit it.

- [ ] **Step 5: Renumber existing generated-output and AI subsections**

Rename current `### 6.3 生成物` to `### 6.4 生成物` and current
`### 6.4 AI生成規則` to `### 6.5 AI生成規則`.

Change the generated manifest set to:

```text
PointerContractManifest.bin
MemoryContractManifest.bin
PointerMemoryConsumerBindingManifest.bin
CppValueTransferPolicyManifest.bin
```

Add AST／clang-tidy rule generation for every rejection in Step 4.

- [ ] **Step 6: Add four exact Memory diagnostics**

In `## 7. FailureとDiagnostic`, register:

```text
diagnostic.memory.value-transfer-binding-missing
MIRAKAN-MEMORY-VALUE-TRANSFER-BINDING-MISSING
arguments = api_contract_id, api_contract_version, subject, ordinal?

diagnostic.memory.value-transfer-invalid
MIRAKAN-MEMORY-VALUE-TRANSFER-INVALID
arguments = api_contract_id, subject, ordinal?, rule_id

diagnostic.memory.hot-callback-allocation
MIRAKAN-MEMORY-HOT-CALLBACK-ALLOCATION
arguments = campaign_hash, scenario_id, payload_bytes, observed_count

diagnostic.memory.hot-callback-upstream-fallback
MIRAKAN-MEMORY-HOT-CALLBACK-UPSTREAM-FALLBACK
arguments = campaign_hash, scenario_id, payload_bytes, observed_count
```

The first two are Build／static failures. The latter two are Performance Gate
failures and do not publish callback output.

- [ ] **Step 7: Extend metrics, negative tests, introduction order, and DoD**

Add these exact requirements:

```markdown
- container ownerごとのstorage layout、element storage、access pattern、capacity source、growth policy、address stability、hot-path分類をtelemetryへ投影する。
- hot callbackのgeneral-heap allocation countとupstream fallback countは両方exact 0。
- source callable subject集合とbinding集合のset equality、`const T&&`、const move、conditional sink move、moved-from reuse、NRVO阻害、C ABIへのC++型流出をnegative fixtureで拒否する。
- `PointerContractV1`、`MemoryContractV1`、`PointerMemoryConsumerBindingV1`、`CppValueTransferPolicyV1`を同じPhase 0 definition closureへ追加する。
- Definition of Doneは四Type、四manifest、全generated C++ consumer、C ABI adapter分離を検査する。
```

- [ ] **Step 8: Verify canonical ownership and commit only Memory／Pointers**

```powershell
$path = 'docs/architecture/02-foundation/memory-pointers.md'
foreach ($token in @(
  'CppValueTransferPolicyV1',
  'CppValueTransferPolicyManifest.bin',
  'MIRAKAN_CPP_VALUE_TRANSFER_POLICY_V1',
  'MIRAKAN_CPP_VALUE_TRANSFER_BINDING_V1',
  'std::vector<T>',
  'std::pmr::vector<T>',
  'MIRAKAN-MEMORY-VALUE-TRANSFER-BINDING-MISSING',
  'MIRAKAN-MEMORY-HOT-CALLBACK-UPSTREAM-FALLBACK',
  'measured_p999_headroom',
  'contiguous_inline',
  'destroy_or_assign_only'
)) {
  if (-not (rg -q --fixed-strings $token $path)) { throw "Missing token: $token" }
}
$capacityDefinitions = (rg -n '^  capacity_source:' $path | Measure-Object).Count
if ($capacityDefinitions -ne 1) { throw "Expected one schema capacity_source, got $capacityDefinitions" }
git diff --check -- $path
git add -- $path
$staged = @(git diff --cached --name-only)
if ($staged.Count -ne 1 -or $staged[0] -ne $path) { throw "Unexpected staged paths: $($staged -join ', ')" }
git commit --only -m "docs: define C++ value transfer and layout contracts" -- $path
```

Expected: one-file commit; Memory／Pointers contains the sole full schemas.

---

### Task 4: Bind C++23 generation and static gates

**Files:**
- Modify: `docs/architecture/02-foundation/cpp23-modules.md:320-358`
- Modify: `docs/architecture/02-foundation/cpp23-modules.md:466-486`
- Modify: `docs/architecture/02-foundation/cpp23-modules.md:504-538`

**Interfaces:**
- Consumes: exact `CppValueTransferPolicyV1` and four generated manifests.
- Produces: C++ Module／Header generation gate; no local transfer schema.

- [ ] **Step 1: Assert the consumer binding is missing**

```powershell
$path = 'docs/architecture/02-foundation/cpp23-modules.md'
if (rg -q 'CppValueTransferPolicyV1' $path) { throw 'Precondition failed: policy already bound.' }
Write-Error 'Expected failure: C++23 plan lacks the value-transfer consumer gate.'
```

Expected: FAIL.

- [ ] **Step 2: Add exact generated／First-party C++ rules**

Under `## 12. AI生成C++とMCD`, add:

```markdown
Generated／First-party C++の全public callable subjectは[Memory／Pointers](memory-pointers.md)の`CppValueTransferPolicyV1`へexact解決する。AI、hand-written Module、CX0 Header、CX3 Moduleで別規則を持たず、missing bindingを推測補完しない。

AST Gateは`const T&&`、const objectからのmove、conditional sink move、destroy／assign以外のmoved-from access、unqualified nontrivial by-value input、scalar／enum／typed handleの理由なき`const T&`、未writeのnon-const ref、returnで足りるoutput parameter、NRVO対象localの`return std::move(local)`を拒否する。
```

Under `### 16.4 正しさ`, add manifest／source／generated signature hash closure and
explicit C ABI separation.

- [ ] **Step 3: Extend Phase 0 artifacts and DoD**

Add `CppValueTransferPolicyManifest.bin` to Phase 0 outputs and require:

```markdown
- source API subject集合、policy binding集合、generated signature集合が同じContract Set／Target／Toolchainで一対一。
- C ABI Headerはfixed-width value、opaque handle、function table、caller-owned bufferだけを持ち、C++ reference／STL typeを0件にする。
- CX3 cutover時に旧Header signature、alias Module、dual generated APIを残さない。
```

- [ ] **Step 4: Verify no schema duplication and commit**

```powershell
$path = 'docs/architecture/02-foundation/cpp23-modules.md'
if (-not (rg -q 'CppValueTransferPolicyV1' $path)) { throw 'Policy binding missing.' }
if (-not (rg -q 'CppValueTransferPolicyManifest\.bin' $path)) { throw 'Manifest missing.' }
if (rg -q 'target_abi_facts:|transfer_form:|binding_hash:' $path) { throw 'C++23 duplicated the Memory schema.' }
git diff --check -- $path
git add -- $path
$staged = @(git diff --cached --name-only)
if ($staged.Count -ne 1 -or $staged[0] -ne $path) { throw "Unexpected staged paths: $($staged -join ', ')" }
git commit --only -m "docs: bind C++23 value transfer gates" -- $path
```

Expected: one-file commit.

---

### Task 5: Keep Native C ABI fixed and bind its C++ adapter

**Files:**
- Modify: `docs/architecture/03-authoring/native-game-module.md:65-83`
- Modify: `docs/architecture/03-authoring/native-game-module.md:197-260`
- Modify: `docs/architecture/03-authoring/native-game-module.md:546-580`

**Interfaces:**
- Consumes: `CppValueTransferPolicyV1`, `PointerContractV1`, `MemoryContractV1`.
- Produces: C ABI／C++ adapter mapping, persistent factory rules, static and runtime fixtures.

- [ ] **Step 1: Assert the adapter policy binding is missing**

```powershell
$path = 'docs/architecture/03-authoring/native-game-module.md'
if (rg -q 'CppValueTransferPolicyV1' $path) { throw 'Precondition failed: policy already bound.' }
Write-Error 'Expected failure: Native adapter does not yet consume the policy.'
```

Expected: FAIL.

- [ ] **Step 2: Add the exact C ABI non-projection rule**

Under `### 4.1 C ABI entry`, add:

```markdown
`CppValueTransferPolicyV1`はC ABI shapeを直接生成しない。C ABIは固定幅standard-layout value、opaque handle、C function table、明示pointer＋byte-or-element count、caller-owned output bufferだけを使う。C++ reference、`std::span`、STL／PMR、`Result<T>`、exception、owner wrapperをABIへ出さない。
```

- [ ] **Step 3: Bind generated C++ adapters and persistent factories**

Under `### 5.1 公開API`, add:

```markdown
Generated C++ adapterは[Memory／Pointers](../02-foundation/memory-pointers.md)の`CppValueTransferPolicyV1`、`PointerContractV1`、`MemoryContractV1`へexact解決する。C ABIのpointer／count／alignment／call lifetimeを検証してcall-scope `std::span`またはgenerated bounded viewへ変換し、return後に保持しない。`unique_owner`／`move_sink`はABIを越えず、Memory Port、opaque handle、caller-owned bufferを使う。

Persistent factoryはdefault allocator、global `new`、default PMRへfallbackせず、exact Memory Portと`unique_owner` policyを使う。C++ adapterのprivate measured exceptionはC ABI validationまたはpublic C++ policyを緩和しない。
```

- [ ] **Step 4: Add static, fuzz, and negative fixtures**

In §§12–13 add:

```markdown
- C ABI Header AST scanでreference、`std::span`、STL／PMR、`Result<T>`、exception、owner wrapperが0件。
- pointer nullability、count overflow、alignment mismatch、call-return後view access、Memory Port mismatchを各一原因で拒否する。
- source C++ callable binding、generated adapter、C ABI descriptor、Contract Set、Target Profile、Toolchain lockのhash不一致でModule全体をloadしない。
- old signature alias、dual descriptor、old readerをConsumer Inventoryなしで追加しない。
```

- [ ] **Step 5: Run the integrated Foundation verification**

```powershell
$files = @(
  'docs/architecture/02-foundation/core-architecture.md',
  'docs/architecture/02-foundation/executable-contracts.md',
  'docs/architecture/02-foundation/memory-pointers.md',
  'docs/architecture/02-foundation/cpp23-modules.md',
  'docs/architecture/03-authoring/native-game-module.md'
)
foreach ($file in $files) {
  git diff --check -- $file
}
$owners = @(rg -l '^CppValueTransferPolicyV1$' $files | ForEach-Object { $_ -replace '\\','/' })
if ($owners.Count -ne 1 -or $owners[0] -ne 'docs/architecture/02-foundation/memory-pointers.md') {
  throw "Expected one full schema owner, got: $($owners -join ', ')"
}
foreach ($file in $files) {
  if (-not (rg -q 'CppValueTransferPolicyV1' $file)) { throw "Missing policy route: $file" }
}
```

Expected: one full schema owner and all five files reference the policy.

- [ ] **Step 6: Commit only Native Game Module**

```powershell
$path = 'docs/architecture/03-authoring/native-game-module.md'
git add -- $path
$staged = @(git diff --cached --name-only)
if ($staged.Count -ne 1 -or $staged[0] -ne $path) { throw "Unexpected staged paths: $($staged -join ', ')" }
git commit --only -m "docs: bind native C ABI adapters to memory policy" -- $path
```

Expected: one-file commit and a complete Foundation plan result.
