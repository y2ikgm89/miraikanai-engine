# D3D12 Backend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `CanonicalRenderExecutionPlanV1`をEnhanced BarriersだけでDirect3D 12へencodeし、Windows Editor／GameHostのpackage、fault recovery、Target Qualificationまで閉じるprivate backendを実装する。

**Architecture:** Render Graphがlogical lifetime／hazard／queue intentを所有し、`mirakan.rendering.d3d12.adapter`がControl Plane baselineへbindしたnative planへ一方向変換する。Source Policy、Runtime Derived、Evidenceを分離し、D3D12 native型、pointer、cache bytesをProject／Save／AI Authoringへ公開しない。

**Tech Stack:** C++23 Modules、CMake 4.4.0、Ninja 1.13.2、MSVC Build Tools 18.8.0、Windows SDK exact lock、DirectX 12 Agility SDK 1.619.4／SDKVersion 619、DXC 1.9.2602.24、D3D12MA 3.2.0、WARP＋NVIDIA／AMD／Intel実機、CTest。

## Global Constraints

- 開始前に`architecture/baselines/control-plane-v1.json`をread-backし、全hash不一致とdirty treeを`diagnostic.architecture.baseline-mismatch`で拒否する。
- Target logical IDは`target.windows.desktop`、Backend logical IDは`profile.rendering.d3d12`とし、versionをIDへ埋め込まない。
- Windows C1 runtime barrier pathはEnhanced Barriersだけとし、legacy `ResourceBarrier` translator／fallbackを製品binaryへ含めない。
- Agility SDKはstable `Microsoft.Direct3D.D3D12` 1.619.4、`D3D12SDKVersion=619`をexact lockする。Preview SDKを混在させない。
- D3D12MAは3.2.0、一Device一Allocator、`D3D12MA_RECOMMENDED_ALLOCATOR_FLAGS`、thread-safe既定とし、`SINGLETHREADED`を使わない。
- Pipeline keyはkind、shader closure、Root Signature、全fixed-function state、RTV slot順、DSV、sample count、sample quality、Target／Backend Profile hashをcanonical bytesへ含める。
- After-resourceのalias activation `DISCARD`は、before-resource barrierがlayout transitionまたはwrite flushを行う場合に併用しない。
- Descriptor headroom、HDR／SDR tolerance、device recovery、Pipeline Library採用はD3D12 Design §29の固定式を変更せず使う。
- Shipping packageから`D3D12SDKLayers.dll`を除外し、`D3D12Core.dll`を`.\D3D12\`配下へexact hashで配置する。
- Runtime source shader compile、WARP production fallback、別Adapterへの無通知切替、無限recovery retry、pipeline cacheのUser data化を禁止する。

---

## 1. File map

| Path | Responsibility |
|---|---|
| `docs/architecture/06-rendering/d3d12-backend.md` | D3D12 Backend active正本 |
| `schemas/rendering/d3d12/backend-profile.mcd` | Source Policy schema |
| `schemas/rendering/d3d12/capability-snapshot.mcd` | Device query projection |
| `schemas/rendering/d3d12/encoding-plan.mcd` | pointer-free derived plan／trace |
| `schemas/rendering/d3d12/qualification-receipt.mcd` | Hardware／driver／SDK Evidence |
| `engine/rendering/d3d12/backend.ixx` | public private-adapter module interface |
| `engine/rendering/d3d12/device.cpp` | Agility、Factory、Adapter、Device、Capability |
| `engine/rendering/d3d12/queue.cpp` | Queue、Fence、Allocator／List pool |
| `engine/rendering/d3d12/memory.cpp` | D3D12MA、budget、resource、alias allocation |
| `engine/rendering/d3d12/descriptors.cpp` | heap partition、generation、retire |
| `engine/rendering/d3d12/pipelines.cpp` | binding、Root Signature、PSO key／library |
| `engine/rendering/d3d12/barriers.cpp` | logical hazard→Enhanced Barrier mapping |
| `engine/rendering/d3d12/encoding.cpp` | command list record／submit plan |
| `engine/rendering/d3d12/surface.cpp` | Swap Chain、Present、HDR、resize |
| `engine/rendering/d3d12/recovery.cpp` | DRED、device generation recovery |
| `engine/rendering/d3d12/diagnostics.cpp` | HRESULT→stable diagnostic mapping |
| `engine/rendering/d3d12/CMakeLists.txt` | module target、Agility／D3D12MA binding |
| `tests/rendering/d3d12/*_tests.cpp` | Unit／contract tests |
| `tests/rendering/d3d12/fixtures/**` | WARP、mock、negative fixtures |
| `packaging/windows/d3d12-package-policy.json` | DLL path、export、shipping exclusion policy |

## 2. Core interfaces

```cpp
export module mirakan.rendering.d3d12.adapter;

export namespace mirakan::rendering::d3d12 {
  enum class PipelineKind : std::uint8_t { graphics, compute, mesh };

  struct BackendCreateInfo final {
    ArtifactRefV1 control_plane_baseline_ref;
    TargetProfileRefV1 target_profile_ref;
    McdContractRefV1 backend_profile_ref;
    NativeWindowPort& window_port;
    DiagnosticSink& diagnostics;
  };

  class Backend final {
  public:
    static Result<std::unique_ptr<Backend>> create(const BackendCreateInfo&);
    Result<D3d12EncodingPlanV1> compile(const CanonicalRenderExecutionPlanV1&) const;
    Result<SubmissionReceiptV1> submit(const D3d12EncodingPlanV1&);
    Result<PresentReceiptV1> present(SurfaceId);
    Result<DeviceRecoveryReceiptV1> recover(DeviceFaultV1);
  };
}
```

`Backend`以外のnative classをexportしない。`D3d12EncodingPlanV1`とReceiptはpointer、COM object、CPU address、GPU virtual addressを含めず、logical resource／pass／submission IDとhashだけを持つ。

## 3. Tasks

### Task 1: Control Plane baselineとD3D12正本を接続する

**Files:**
- Create: `docs/architecture/06-rendering/d3d12-backend.md`
- Modify: `architecture/registry/document-relations.v1.json`
- Modify: `architecture/registry/product.v1.json`
- Test: `tools/architecture_lint/test/d3d12-document.test.mjs`

**Interfaces:**
- Consumes: `architecture/baselines/control-plane-v1.json`。
- Produces: document ID `mirakan.arch.rendering-d3d12-backend`、Work Package `wp.runtime.d3d12-backend`。

- [ ] **Step 1: baseline mismatchとmissing document testを書く**
- [ ] **Step 2: failureを確認する**
- [ ] **Step 3: `state=review`、`approval_ref=null`のD3D12正本を追加する**

Metadataのdirect `requires`は`["mirakan.arch.platform-windows"]`だけとする。次のreciprocal integrationを同じChangeSetで両側へ追加する。

| Peer | Contract IDs |
|---|---|
| `mirakan.arch.architecture-governance` | `contract.architecture.change-set` |
| `mirakan.arch.compatibility-evolution` | `contract.compatibility.backend-profile`; `contract.compatibility.rebuild-only-cache` |
| `mirakan.arch.toolchain-dependencies` | `contract.toolchain.dependency-lock`; `contract.toolchain.runtime-artifact-ref` |
| `mirakan.arch.rendering-render-graph` | `contract.rendering.canonical-execution-plan`; `contract.rendering.d3d12-encoding-plan` |
| `mirakan.arch.runtime-persistence-save` | `contract.persistence.renderer-intent-projection` |
| `mirakan.arch.runtime-package` | `contract.package.d3d12-runtime-binding`; `contract.package.runtime-package-manifest` |
| `mirakan.arch.platform-application-package-release` | `contract.package.application-assembly`; `contract.release.package-validation-receipt` |

Windows metadataには`contract.rendering.platform-surface-handoff`と`contract.rendering.target-capability-snapshot`をreciprocalに登録する。D3D12 document IDを他の`requires`へ追加しないためcycleを作らない。

同じChangeSetで`wp.runtime.d3d12-backend.owner_document_id`を暫定Owner `mirakan.arch.rendering-render-graph`から新Owner `mirakan.arch.rendering-d3d12-backend`へ置換する。新文書追加前または別ChangeSetでownerだけを変更しない。

- [ ] **Step 4: architecture lintを実行する**

Expected: active node +1、cycle／redundant／reciprocity error 0。Capability activationは`not_activated`のまま。

### Task 2: MCD contractとnegative schema fixtureを追加する

**Files:**
- Create: `schemas/rendering/d3d12/backend-profile.mcd`
- Create: `schemas/rendering/d3d12/capability-snapshot.mcd`
- Create: `schemas/rendering/d3d12/encoding-plan.mcd`
- Create: `schemas/rendering/d3d12/qualification-receipt.mcd`
- Test: `tests/contracts/d3d12_contract_tests.cpp`

**Interfaces:**
- Produces: `D3d12BackendProfileV1`、`D3d12CapabilitySnapshotV1`、`D3d12EncodingPlanV1`、`D3d12QualificationReceiptV1`。

- [ ] **Step 1: unknown Field、pointer-like integer、invalid enum、unbounded arrayのfailing fixtureを書く**
- [ ] **Step 2: contract compiler failureを確認する**
- [ ] **Step 3: Design §8／§9／§17／§21の全Field、cardinality、bound、ordering、failureをMCDへ定義する**
- [ ] **Step 4: contract testsを実行する**

Expected: positive fixture PASS、negative 4種は各exact diagnostic一件でPASS。

### Task 3: Agility bootstrapとcapability snapshotを実装する

**Files:**
- Create: `engine/rendering/d3d12/backend.ixx`
- Create: `engine/rendering/d3d12/device.cpp`
- Create: `engine/rendering/d3d12/diagnostics.cpp`
- Test: `tests/rendering/d3d12/device_tests.cpp`

**Interfaces:**
- Produces: `Backend::create`、`D3d12CapabilitySnapshotV1`。

- [ ] **Step 1: SDKVersion mismatch、missing D3D12Core、unsupported Enhanced Barriers、query E_INVALIDARG testを書く**
- [ ] **Step 2: WARP testが未実装で失敗することを確認する**
- [ ] **Step 3: Factory→Adapter rank→Device→feature queryのstate machineを実装する**

Required query failure／unknownはTarget不適合、optional query `E_INVALIDARG`は`unknown`としActivationしない。Enhanced Barriers false／unknown、SM 6.6未満、required format/sample不適合をrejectする。

- [ ] **Step 4: WARP conformance testを実行する**

Expected: `fixture.rendering.d3d12-warp-conformance`だけがWARPを選べ、Production profileでWARPを要求すると`diagnostic.rendering.d3d12.adapter-profile-unsupported`。

### Task 4: Queue、Fence、command allocator lifecycleを実装する

**Files:**
- Create: `engine/rendering/d3d12/queue.cpp`
- Test: `tests/rendering/d3d12/queue_tests.cpp`

**Interfaces:**
- Produces: `SubmissionIdV1{device_generation,queue_id,fence_value}`、completion-gated allocator pool。

- [ ] **Step 1: fence未完了allocator reuse、cross-generation submission、queue wait cycle testを書く**
- [ ] **Step 2: failureを確認する**
- [ ] **Step 3: Direct／Compute／Copy queue、monotonic fence、poolを実装する**
- [ ] **Step 4: 100,000 submission stressを実行する**

Expected: duplicate fence 0、early reset 0、cycleはcompile前にexact diagnosticで拒否。

### Task 5: D3D12MA resource／budget／aliasingを実装する

**Files:**
- Create: `engine/rendering/d3d12/memory.cpp`
- Test: `tests/rendering/d3d12/memory_tests.cpp`

**Interfaces:**
- Produces: generation付き`D3d12ResourceIdV1`、budget snapshot、alias allocation plan。

- [ ] **Step 1: one allocator、thread-safe flag、budget pressure、overlap lifetime、invalid initial layout testを書く**
- [ ] **Step 2: failureを確認する**
- [ ] **Step 3: `D3D12MA::CreateAllocator`と`CreateResource3` pathを実装する**

AllocatorはDeviceごと一つ、`D3D12MA_RECOMMENDED_ALLOCATOR_FLAGS`、`SINGLETHREADED`なし。custom poolは登録済みdomainだけ、block size 0を既定とする。Aliased resourcesは最大Size／Alignmentを計算し、同時利用をcompile時に拒否する。

- [ ] **Step 4: 75／85／95／100% pressure fixtureを実行する**

Expected: cache eviction→new allocation rejectionの順、existing Source／published frame破損0。

### Task 6: Descriptor heapと20% headroom Gateを実装する

**Files:**
- Create: `engine/rendering/d3d12/descriptors.cpp`
- Test: `tests/rendering/d3d12/descriptor_tests.cpp`

**Interfaces:**
- Produces: generation付きdescriptor range、retire fence、peak telemetry。

- [ ] **Step 1: stale handle、partition overflow、mid-frame grow、retire前reuse testを書く**
- [ ] **Step 2: failureを確認する**
- [ ] **Step 3: fixed partitionとgeneration validatorを実装する**
- [ ] **Step 4: Design §29.2の3 fixture×30 runを実行する**

Expected: 各heap peak/capacity `<=0.80`、rejection 0、stale diagnostic 0。超過時はProfile revisionなしにcapacityを変更しない。

### Task 7: Root Signature、Pipeline key、kind別PSO closureを実装する

**Files:**
- Create: `engine/rendering/d3d12/pipelines.cpp`
- Test: `tests/rendering/d3d12/pipeline_tests.cpp`

**Interfaces:**
- Produces: `D3d12PipelineKeyV1`、Root Signature artifact、Pipeline Library adapter。

- [ ] **Step 1: sample quality omission、RTV reorder、invalid shader pair、duplicate subobject、mesh input-layout testを書く**
- [ ] **Step 2: old keyがcollisionしてfailureになることを確認する**
- [ ] **Step 3: Design §15.2のcanonical keyを実装する**

GraphicsはVS必須、HS/DS pair、ComputeはCSのみ、MeshはMS必須。RTVはslot 0..7、未使用UNKNOWN。Sample Count／Quality両方をhashし、formatごとのquality queryを検証する。

- [ ] **Step 4: `D3DX12ParsePipelineStream`とEngine validatorを実行する**

Expected: positive graphics／compute／mesh各1 PASS、negative 5種がそれぞれ一件のstable diagnostic。

- [ ] **Step 5: Design §29.5の60-run Pipeline Library Gateを実行する**

Expected: median比、p95、rebuild成功率の式をReceiptへ保存。未達では通常PSO pathを選ぶ。

### Task 8: Enhanced Barrier mappingとalias discard conflictを実装する

**Files:**
- Create: `engine/rendering/d3d12/barriers.cpp`
- Test: `tests/rendering/d3d12/barrier_tests.cpp`
- Create: `tests/rendering/d3d12/fixtures/alias-discard-conflict.mcd`

**Interfaces:**
- Produces: `D3d12BarrierMappingEntryV1`、barrier groups、`diagnostic.rendering.d3d12.alias-discard-conflict`。

- [ ] **Step 1:全logical usageのtable-driven testとunknown usage testを書く**
- [ ] **Step 2: alias conflict fixtureを追加する**

Fixtureはbefore texture write、write flush、layout transition、same-memory after texture、`LayoutBefore=UNDEFINED`、DISCARD requestを含む。

- [ ] **Step 3: compile failureを確認する**
- [ ] **Step 4: Sync／Access／Layout mappingと二段alias initializationを実装する**
- [ ] **Step 5: Debug Layer／GPU validationを有効にしてtestを実行する**

Expected: combined DISCARD planは拒否、separate barrier→Discard/Clear/Copy planはPASS、legacy runtime barrier call 0。

### Task 9: Encoding／submission／traceを実装する

**Files:**
- Create: `engine/rendering/d3d12/encoding.cpp`
- Test: `tests/rendering/d3d12/encoding_tests.cpp`

**Interfaces:**
- Produces: `Backend::compile`、`Backend::submit`、bounded `D3d12EncodingTraceV1`。

- [ ] **Step 1: deterministic plan、queue wait、trace bound、continuation testを書く**
- [ ] **Step 2: failureを確認する**
- [ ] **Step 3: Graph→resource→descriptor→barrier→command list→submitのpure compile pathを実装する**
- [ ] **Step 4: shuffled recording worker count 1／2／4／8で比較する**

Expected: plan hash、submission dependency、render output hashが全worker countで一致。

### Task 10: Surface、Present、HDRを実装する

**Files:**
- Create: `engine/rendering/d3d12/surface.cpp`
- Test: `tests/rendering/d3d12/surface_tests.cpp`

**Interfaces:**
- Produces: `Backend::present`、generation付きSurface、HDR／SDR receipt。

- [ ] **Step 1: resize、occlusion、tearing mismatch、HDR monitor move、stale Surface testを書く**
- [ ] **Step 2: failureを確認する**
- [ ] **Step 3: flip model、color-space query、Present flag policyを実装する**
- [ ] **Step 4: 10,000 resizeとDesign §29.3 visual testを実行する**

Expected: non-finite pixel 0、ΔE／clip率が全閾値内、stale generation use 0。

### Task 11: DREDとone-shot recoveryを実装する

**Files:**
- Create: `engine/rendering/d3d12/recovery.cpp`
- Test: `tests/rendering/d3d12/recovery_tests.cpp`

**Interfaces:**
- Produces: `Backend::recover`、private DRED attachment、`DeviceRecoveryReceiptV1`。

- [ ] **Step 1: removed／reset／driver-internal各10 injection、二回目failure、artifact missing testを書く**
- [ ] **Step 2: failureを確認する**
- [ ] **Step 3: published Source snapshotからnew generationを再構築する**
- [ ] **Step 4: Design §29.4の30 injectionを各Adapter rowで実行する**

Expected: Editor 30/30、p95<=10.0s、max<=15.0s。二回目retry 0、Adapter silent switch 0。

### Task 12: Runtime／Application Package bindingを実装する

**Files:**
- Modify: `docs/architecture/04-runtime/runtime-package.md`
- Modify: `docs/architecture/07-platform/application-package-release.md`
- Create: `packaging/windows/d3d12-package-policy.json`
- Test: `tests/packaging/windows/d3d12_package_tests.cpp`

**Interfaces:**
- Produces: `D3d12RuntimePackageBindingV1`、`PackageValidationReceiptV1`。

- [ ] **Step 1: wrong SDKVersion、missing slash、EXE-root DLL、Layers in Shipping、hash mismatch testを書く**
- [ ] **Step 2: failureを確認する**
- [ ] **Step 3: Design §32のbindingとassembly validatorを実装する**
- [ ] **Step 4: Development／Shipping packageをbuildしread-backする**

Expected: Main EXE export 619／`.\D3D12\`、Core DLL exact hash、Development Layers version一致、Shipping Layers absent。

### Task 13: Full QualificationとCandidate boundaryを閉じる

**Files:**
- Create: `qualification/rendering/d3d12/c1-matrix.v1.json`
- Create: `qualification/rendering/d3d12/expected-gates.v1.json`
- Test: `tests/rendering/d3d12/qualification_tests.cpp`

**Interfaces:**
- Produces: Target別`D3d12QualificationReceiptV1`。Product Activation writerではない。

- [ ] **Step 1: WARP、NVIDIA、AMD、Intel、negative rowをmatrixへ固定する**
- [ ] **Step 2: static／schema、conformance、fault、performance、soakを実行する**
- [ ] **Step 3: Receipt hashとCandidate inputをread-backする**
- [ ] **Step 4: Product registryの`candidate_locked`作成可否だけを報告する**

Expected: 必須row全合格時だけcandidate input生成。文書、binary、単一GPU合格だけで`qualified`／`production`を書かない。

## Appendix A: Diagnostic clean migration

| Old ID | New stable ID |
|---|---|
| `MIRAKAN-D3D12-AGILITY_RUNTIME_MISMATCH` | `diagnostic.rendering.d3d12.agility-runtime-mismatch` |
| `MIRAKAN-D3D12-ADAPTER_NOT_FOUND` | `diagnostic.rendering.d3d12.adapter-not-found` |
| `MIRAKAN-D3D12-ADAPTER_PROFILE_UNSUPPORTED` | `diagnostic.rendering.d3d12.adapter-profile-unsupported` |
| `MIRAKAN-D3D12-DEVICE_CREATE_FAILED` | `diagnostic.rendering.d3d12.device-create-failed` |
| `MIRAKAN-D3D12-CAPABILITY_QUERY_FAILED` | `diagnostic.rendering.d3d12.capability-query-failed` |
| `MIRAKAN-D3D12-ENHANCED_BARRIERS_REQUIRED` | `diagnostic.rendering.d3d12.enhanced-barriers-required` |
| `MIRAKAN-D3D12-QUEUE_CREATE_FAILED` | `diagnostic.rendering.d3d12.queue-create-failed` |
| `MIRAKAN-D3D12-DESCRIPTOR_CAPACITY_EXCEEDED` | `diagnostic.rendering.d3d12.descriptor-capacity-exceeded` |
| `MIRAKAN-D3D12-DESCRIPTOR_STALE` | `diagnostic.rendering.d3d12.descriptor-stale` |
| `MIRAKAN-D3D12-RESOURCE_STALE` | `diagnostic.rendering.d3d12.resource-stale` |
| `MIRAKAN-D3D12-MEMORY_BUDGET_EXCEEDED` | `diagnostic.rendering.d3d12.memory-budget-exceeded` |
| `MIRAKAN-D3D12-BARRIER_MAPPING_MISSING` | `diagnostic.rendering.d3d12.barrier-mapping-missing` |
| `MIRAKAN-D3D12-BARRIER_VALIDATION_FAILED` | `diagnostic.rendering.d3d12.barrier-validation-failed` |
| `MIRAKAN-D3D12-PIPELINE_UNAVAILABLE` | `diagnostic.rendering.d3d12.pipeline-unavailable` |
| `MIRAKAN-D3D12-SURFACE_STALE` | `diagnostic.rendering.d3d12.surface-stale` |
| `MIRAKAN-D3D12-PRESENT_MODE_UNSUPPORTED` | `diagnostic.rendering.d3d12.present-mode-unsupported` |
| `MIRAKAN-D3D12-PRESENT_FAILED` | `diagnostic.rendering.d3d12.present-failed` |
| `MIRAKAN-D3D12-DEVICE_REMOVED` | `diagnostic.rendering.d3d12.device-removed` |
| `MIRAKAN-D3D12-DEVICE_RECOVERY_FAILED` | `diagnostic.rendering.d3d12.device-recovery-failed` |

新規`diagnostic.rendering.d3d12.alias-discard-conflict`に旧IDはない。旧ID出現数0、dual emission 0、old parser 0をArchitecture lintで検査する。

## Completion Gate

- Control Plane baselineの全hashをread-back済みである。
- D3D12正本、MCD、C++ module、tests、package policyが同じChangeSetへ存在する。
- Enhanced-only runtime pathでlegacy barrier symbolが0件である。
- Graphics／Compute／Mesh PSO closure、RTV order、Sample Count／Qualityのnegative testsが通る。
- Alias layout transition／write flush＋DISCARD conflictがexact diagnosticで拒否される。
- Descriptor、HDR／SDR、Recovery、Pipeline LibraryがDesign §29の固定Gateを満たす。
- Development packageはSDKVersion一致Layers、Shipping packageはLayers 0、Core DLL hash一致である。
- NVIDIA／AMD／Intel required matrixとWARP conformanceが完了し、Target別Receiptが同じCandidate inputへbindする。
- Product state writerは別Gateであり、本計画が`qualified`／`production`を直接設定していない。
