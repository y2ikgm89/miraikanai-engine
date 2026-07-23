# D3D12 Backend Implementation Plan

> **Status: executable draft, not yet authorized.** Preflight D0だけは専用Architecture ChangeSet authorizationで実行でき、文書staging／approval／same-definition Rebaseline以外を変更しない。実装Task 2～13はD3D12 technical documentのcurrent human approval、current Product WP lifecycle=`active`、Task Authorizationが揃った後だけ許可する。本書の存在をApproval／WP stateへ読み替えない。

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `CanonicalRenderExecutionPlanV1`をEnhanced BarriersだけでDirect3D 12へencodeし、Windows Editor／GameHostのpackage、fault recovery、Target Qualificationまで閉じるprivate backendを実装する。

**Architecture:** Render Graphがlogical lifetime／hazard／queue intentを所有し、CX0 self-contained Headerのprivate Adapter境界がControl Plane／Runtime ECS E0 baselineへbindしたnative planへ一方向変換する。CMakeは将来のPrimary Named Module `mirakan.rendering.d3d12.adapter`だけを登録する。Source Policy、Runtime Derived、Evidenceを分離し、D3D12 native型、pointer、cache bytesをProject／Save／AI Authoringへ公開しない。

**Tech Stack:** C++23 CX0 self-contained Header、CMake 4.4.0、Ninja 1.13.2、Visual Studio Build Tools 2026 18.8.0 build 12009.203、MSVC toolset v14.51（`cl.exe`／`link.exe` 14.51.36231以上、`/std:c++23preview`）、Windows SDK exact lock、DirectX 12 Agility SDK 1.619.4／SDKVersion 619、DXC 1.9.2602.24、D3D12MA 3.2.0、WARP＋NVIDIA／AMD／Intel実機、CTest。全exact artifact hashは`toolchain-dependencies.md`だけを正本とする。CMake 4.4のExperimental `import std`はCX1 fixtureだけに隔離し、本計画のproduction targetでは使わない。

## Global Constraints

- Preflight D0開始前にcurrent signed Product snapshotの`CurrentControlPlaneBaselineBindingV1`をkind別`bootstrap | rebaseline`完成wrapperとして検証し、Active Definition hash、Baseline Core、Local Schema Catalog、Authority Binding Source Catalog、Toolchain、Trust／revocationをread-backする。さらに`wp.runtime.ecs-e0=complete`のcurrent lifecycle ReceiptとECS artifact／toolchain／test closureを検証する。不一致、stale binding、Approval不正、missing、authoritative source dirtyは`diagnostic.architecture.baseline-mismatch`または`diagnostic.architecture.rebaseline-approval-invalid`で拒否する。Preflight後のTask 2～13はcurrent source treeと分離した一つのauthorized Staging candidateへ累積し、changed-path manifest内のcandidate deltaをdirty sourceと誤判定しない。初回Bootstrap Approval、branch名、`latest`をcurrent authorityとして直接参照しない。
- Preflight D0-A／D0-BはArchitecture ChangeSet authorization下のpre-WP文書staging／approval／rebaselineであり、Runtime／MCD codeを実装しない。D3D12 technical documentのhuman approvalとsame-definition Control Plane Rebaseline／Product baseline rebindingを完成させた後、外部Schedulerがcurrent `wp.runtime.d3d12-backend` row／definition seed、全`requires_work_package_refs[]`の`complete`、Product Owner `mirakan.arch.rendering-render-graph`のcurrent approval、Product Decision、Task Authorizationを検証し、lifecycleを`declared->ready->active`へ進める。Task 2～13はその後だけ開始し、Plan内でWPをactiveへ自己昇格しない。
- MCD contract compilerはRuntime ECS E0 Implementation Plan Task 0の`tools/contract_compiler/**`成果物だけを消費し、`runtime-ecs-e0-v1.json`のcompiler artifact ref／toolchain hash／`runtime_ecs_contract_tests` Receiptとexact一致させる。Task 2はcompilerのunknown field拒否、bound検査、canonical binary生成、negative fixtureへのexact diagnostic発行を前提とする。
- required Target logical ID exact setは`target.windows.editor; target.windows.desktop`、Backend logical IDは`profile.rendering.d3d12`とし、versionをIDへ埋め込まない。Build、package、fixture、Qualification ReceiptをTargetごとに分け、片側成功を他方へ流用しない。
- Windows C1 runtime barrier pathはEnhanced Barriersだけとし、legacy `ResourceBarrier` translator／fallbackを製品binaryへ含めない。
- Agility SDKはstable `Microsoft.Direct3D.D3D12` 1.619.4、`D3D12SDKVersion=619`をexact lockする。Preview SDKを混在させない。
- D3D12MAは3.2.0、一Device一Allocator、`D3D12MA_RECOMMENDED_ALLOCATOR_FLAGS`、thread-safe既定とし、`SINGLETHREADED`を使わない。
- Pipeline keyはkind、shader closure、Root Signature、全fixed-function state、RTV slot順、DSV、sample count、sample quality、Target／Backend Profile hashをcanonical bytesへ含める。
- After-resourceのalias activation `DISCARD`は、before-resource barrierがlayout transitionまたはwrite flushを行う場合に併用しない。
- Descriptor headroom、HDR／SDR tolerance、device recovery、Pipeline Library採用はD3D12 Design §29の固定式を変更せず使う。
- Shipping Configurationのcandidate Packageから`D3D12SDKLayers.dll`を除外し、`D3D12Core.dll`を`.\D3D12\`配下へexact hashで配置する。この検査成功をRelease Activationへ流用しない。
- CX0／CX1成果物はDevelopment、Test、candidate Package、internal Technology Previewだけに使い、Release Activationへ入力しない。本計画ではCX1専用fixture以外の`.ixx`／`.cppm`を作成せず、外部条件不成立時はCX0 Headerを維持する。
- Runtime source shader compile、WARP production fallback、別Adapterへの無通知切替、無限recovery retry、pipeline cacheのUser data化を禁止する。

---

## 1. File map

| Path | Responsibility |
|---|---|
| `docs/architecture/06-rendering/d3d12-backend.md` | D3D12 Backend active正本 |
| `schemas/mirakan/rendering/d3d12/backend-profile.mcd` | Source Policy schema |
| `schemas/mirakan/rendering/d3d12/capability-snapshot.mcd` | Device query projection |
| `schemas/mirakan/rendering/d3d12/encoding-plan.mcd` | pointer-free derived plan／trace |
| `schemas/mirakan/rendering/d3d12/qualification-receipt.mcd` | Hardware／driver／SDK Evidence |
| `engine/rendering/d3d12/include/mirakan/rendering/d3d12/backend.hpp` | CX0 self-contained public private-adapter surface |
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
| `engine/rendering/d3d12/CMakeLists.txt` | CX0 Header target、将来Module名、Agility／D3D12MA binding。production `MODULE_INTERFACE`なし |
| `tests/rendering/d3d12/*_tests.cpp` | Unit／contract tests |
| `tests/rendering/d3d12/fixtures/**` | WARP、mock、negative fixtures |
| `packaging/windows/d3d12-package-policy.json` | DLL path、export、shipping exclusion policy |

設計§7のclosed tableを実装入口へexact複写し、別名targetや互換aliasを作らない。

| Build／test concept | Canonical value |
|---|---|
| production CMake target | `mirakan_rendering_d3d12` |
| aggregate test target | `mirakan_d3d12_backend_tests` |
| CTest label | `rendering.d3d12` |
| CTest names | `rendering.d3d12.contract`; `rendering.d3d12.device`; `rendering.d3d12.queue`; `rendering.d3d12.memory`; `rendering.d3d12.descriptor`; `rendering.d3d12.pipeline`; `rendering.d3d12.barrier`; `rendering.d3d12.encoding`; `rendering.d3d12.surface`; `rendering.d3d12.recovery`; `rendering.d3d12.package`; `rendering.d3d12.qualification` |

全Taskのtargeted verification入口を次に固定する。

```powershell
cmake -S . -B build/d3d12 -G Ninja -DCMAKE_BUILD_TYPE=Debug
cmake --build build/d3d12 --target mirakan_d3d12_backend_tests
ctest --test-dir build/d3d12 -L '^rendering\.d3d12$' --output-on-failure
```

## 2. Core interfaces

```cpp
#pragma once

#include <cstdint>
#include <memory>

namespace mirakan::rendering::d3d12 {
  enum class PipelineKind : std::uint8_t { graphics, compute, mesh };

  struct BackendCreateInfo final {
    CurrentControlPlaneBaselineBindingV1 control_plane_baseline_binding;
    TargetProfileRefV1 target_profile_ref;
    McdContractRefV1 backend_profile_ref;
    IApplicationSurface& application_surface;
    DiagnosticSink& diagnostics;
  };

  class Backend final {
  public:
    static Result<std::unique_ptr<Backend>> create(const BackendCreateInfo&);
    Result<D3d12EncodingPlanV1> compile(const CanonicalRenderExecutionPlanV1&) const;
    Result<SubmissionReceiptV1> submit(const D3d12EncodingPlanV1&);
    Result<PresentReceiptV1> present(SurfaceId);
    Result<DeviceRecoveryReceiptV1> recover(const D3d12DeviceFaultReportV1&);
  };
}
```

上記は公開宣言の抜粋である。実ファイルは`CurrentControlPlaneBaselineBindingV1`、`TargetProfileRefV1`、`McdContractRefV1`、Render Graph plan、Receipt、`Result<T>`の各Ownerが生成または公開するHeaderを明示includeし、推移的includeやforward declarationだけへ依存しない。単独translation unitで`#include <mirakan/rendering/d3d12/backend.hpp>`だけを行うcompile fixtureを必須にする。

`Backend`以外のnative classをexportしない。`D3d12EncodingPlanV1`とReceiptはpointer、COM object、CPU address、GPU virtual addressを含めず、logical resource／pass／submission IDとhashだけを持つ。

型の定義Ownerを次に固定する。

| 型 | 定義Owner |
|---|---|
| `IApplicationSurface` | windows.md正本（Design §18.1のwindow手渡し境界） |
| `Result<T>` | math-core.md正本（`std::expected<T, Error>`） |
| `D3d12DeviceFaultReportV1` | Design §8.3 Evidence |
| `SubmissionReceiptV1`／`PresentReceiptV1`／`DeviceRecoveryReceiptV1` | 本計画新設。Task 2の`encoding-plan.mcd`で定義する |
| `SurfaceId`／`DiagnosticSink` | 本計画新設。wire型ではなく`include/mirakan/rendering/d3d12/backend.hpp`のC++ interface型であり、MCDへ含めない。`SurfaceId`はgeneration付きsurface identity、`DiagnosticSink`はstable diagnostic送出port |

## 3. Tasks

### Preflight D0-A: Control Plane／Runtime ECS E0 baselineとD3D12正本を接続する

**Files:**
- Create: `docs/architecture/06-rendering/d3d12-backend.md`
- Modify: `architecture/registry/document-relations.v1.json`
- Generated read-only input: current `ActiveProductDefinitionClosureV1`と`ProductOperationalStateSnapshotV1`
- Test: `tools/architecture_lint/test/d3d12-document.test.mjs`

**Interfaces:**
- Consumes: current baseline binding、`wp.runtime.d3d12-backend` definition seed／lifecycle head、completed ECS E0 Receipt closure。
- Produces: technical document ID `mirakan.arch.rendering-d3d12-backend`。既存Work Package `wp.runtime.d3d12-backend`はcurrent Product Definitionからread-onlyに消費する。

- [ ] **Step 1: current binding／ECS Receipt closureのmissing／hash mismatchとmissing document testを書く**
- [ ] **Step 2: failureを確認する**
- [ ] **Step 3: D3D12 file bytesを`ArchitectureDocumentCoreV1`へ束縛し、post-bootstrap新規文書専用の`DocumentLifecycleRecordV1.submit_review`（previous Core／Change Manifestなし、from=null、to=`review`）でcurrent lifecycle headを追加する。bootstrap 5 Owner限定の`construction_seed`を使わず、Metadata V2へstate／approval refを書かない**

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

`wp.runtime.d3d12-backend.owner_document_id`はcurrent Active Definitionどおり`mirakan.arch.rendering-render-graph`に維持する。新D3D12文書はtechnical contract Ownerとしてreciprocal relationを持つが、Product operational DecisionのOwner移管を暗黙実施しない。ECS完了後のfull resetでdependencyを壊すため、移管はproof-carry-forward migrationがActiveになるか、別途承認したepoch boundaryの専用Definition migrationへ分離する。

- [ ] **Step 4: architecture lintを実行する**

Expected: Control PlaneとRuntime ECS E0の全hash read-back PASS、active node +1、cycle／redundant／reciprocity error 0。Capability activationは`not_activated`のまま。

### Preflight D0-B: 既存正本reciprocal更新、承認、Rebaselineを実行する（Design §24 step 2–8／§25）

**Files:**
- Modify: `docs/architecture/README.md`
- Modify: `docs/architecture/00-product/product-plan.md`
- Modify: `docs/architecture/01-governance/ai-security-approval.md`
- Modify: `docs/architecture/01-governance/ai-verification-provenance.md`
- Modify: `docs/architecture/02-foundation/core-architecture.md`
- Modify: `docs/architecture/02-foundation/toolchain-dependencies.md`
- Modify: `docs/architecture/02-foundation/executable-contracts.md`
- Modify: `docs/architecture/02-foundation/cpp23-modules.md`
- Modify: `docs/architecture/02-foundation/naming-project-layout.md`
- Modify: `docs/architecture/03-authoring/editor-ui-framework.md`
- Modify: `docs/architecture/04-runtime/performance-capacity.md`
- Modify: `docs/architecture/04-runtime/debugging-observability-replay.md`
- Modify: `docs/architecture/06-rendering/project-shader.md`
- Modify: `docs/architecture/06-rendering/render-graph.md`
- Modify: `docs/architecture/07-platform/windows.md`
- Test: `tools/architecture_lint/test/d3d12-reciprocal.test.mjs`

**Interfaces:**
- Consumes: Design §25の必須変更表（15行）と§24 step 2–8。
- Produces: D3D12 Backend正本と既存正本のreciprocal整合。

- [ ] **Step 1: §25対象文書ごとの未変更検出testを書く**
- [ ] **Step 2: failureを確認する**
- [ ] **Step 3: §25の15行それぞれの必須変更を対象文書へ適用する**

render-graph.mdはEnhanced-only conformance（Design §16.1）、logical plan Owner、`RendererCapabilityProjectionV1`（redacted projection定義）を含める。toolchain-dependencies.mdは`Microsoft.Direct3D.WARP` redistributable pin行を追加する（version／hashはlock ChangeSetでexact確定）。

- [ ] **Step 4: `ui_d3d12_adapter`→`ui_render_graph_adapter`置換を適用する**

cpp23-modules.mdとeditor-ui-framework.mdの表変更、engine/ui module renameを同じChangeSetで行い、旧名を残さない。

- [ ] **Step 5: architecture lintを実行する**

Expected: §25の全15行に対応する変更が同一ChangeSetへ存在し、`ui_d3d12_adapter`出現0件、reciprocity error 0。

- [ ] **Step 6: D3D12 technical documentを承認し、same-definition Rebaseline／Product baseline rebindingを完成する**

Human reviewerはPreflight D0-A／D0-Bの完成bytesとlint Evidenceを審査し、D3D12 technical documentのcurrent lifecycleを`review->approved`へ進める。続いてControl Plane Design §6.1.1のA2→B2→C2→D2→T2→L2→P2を順に完成し、`ControlPlaneRebaselineApprovalV1`、Envelope、Transaction、Productのfresh `ProductOperationalDecisionV1(kind=control_plane_baseline_rebinding_product)`、`ControlPlaneBaselineRebindingV1`、next snapshotを一回のexpected-parent CASでpublishする。Active Product Definition hash、Activation／Decision／Risk、`wp.architecture.control-plane`以外のWP headはparentからbyte-exactに保持する。current bindingのkind=`rebaseline`とD3D12 approved bytesをread-backできるまで、外部Schedulerは`wp.runtime.d3d12-backend`を`ready`／`active`へ進めない。

Expected: same-definition rebaselineとProduct rebindingが完成し、current Product snapshotが新bindingを指す。Definition migration、full reset、D3D12 WPの自己昇格は0件。

### Task 2: MCD contractとnegative schema fixtureを追加する

**Files:**
- Create: `schemas/mirakan/rendering/d3d12/backend-profile.mcd`
- Create: `schemas/mirakan/rendering/d3d12/capability-snapshot.mcd`
- Create: `schemas/mirakan/rendering/d3d12/encoding-plan.mcd`
- Create: `schemas/mirakan/rendering/d3d12/qualification-receipt.mcd`
- Test: `tests/contracts/d3d12_contract_tests.cpp`

**Interfaces:**
- Produces: `D3d12BackendProfileV1`、`D3d12CapabilitySnapshotV1`、`D3d12EncodingPlanV1`、`D3d12BackendQualificationReceiptV1`、`SubmissionReceiptV1`、`PresentReceiptV1`、`DeviceRecoveryReceiptV1`。

- [ ] **Step 1: unknown Field、pointer-like integer、invalid enum、unbounded arrayのfailing fixtureを書く**
- [ ] **Step 2: contract compiler failureを確認する**
- [ ] **Step 3: Design §8／§9／§17／§21の全Field、cardinality、bound、ordering、failureと、本計画§2のReceipt 3型（`encoding-plan.mcd`）をMCDへ定義する**
- [ ] **Step 4: contract testsを実行する**

Expected: positive fixture PASS、negative 4種は各exact diagnostic一件でPASS。

### Task 3: Agility bootstrapとcapability snapshotを実装する

**Files:**
- Create: `engine/rendering/d3d12/include/mirakan/rendering/d3d12/backend.hpp`
- Create: `engine/rendering/d3d12/CMakeLists.txt`
- Create: `engine/rendering/d3d12/device.cpp`
- Create: `engine/rendering/d3d12/diagnostics.cpp`
- Test: `tests/rendering/d3d12/backend_header_compile_tests.cpp`
- Test: `tests/rendering/d3d12/device_tests.cpp`

**Interfaces:**
- Produces: `Backend::create`、`D3d12CapabilitySnapshotV1`。

- [ ] **Step 1: Header単独compile、SDKVersion mismatch、missing D3D12Core、unsupported Enhanced Barriers、query E_INVALIDARG testを書く**
- [ ] **Step 2: HeaderとWARP実装が未作成で失敗することを確認する**
- [ ] **Step 3: self-contained HeaderとFactory→Adapter rank→Device→feature queryのstate machineを実装する**

`mirakan_add_cpp_component()`へ`TARGET mirakan_rendering_d3d12`、`ALIAS mirakan::rendering_d3d12`、`MODULE_NAME mirakan.rendering.d3d12.adapter`、`HEADER_API_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/include"`を登録する。production `MODULE_INTERFACE`、`.ixx`、`.cppm`は指定しない。

Required query failure／unknownはTarget不適合、optional query `E_INVALIDARG`は`unknown`としActivationしない。Enhanced Barriers false／unknown、SM 6.6未満、required format/sample不適合をrejectする。

- [ ] **Step 4: WARP conformance testを実行する**

Expected: `fixture.rendering.d3d12-warp-conformance`だけがWARPを選べ、Production profileでWARPを要求すると`diagnostic.rendering.d3d12.adapter-profile-unsupported`。

### Task 4: Queue、Fence、command allocator lifecycleを実装する

**Files:**
- Create: `engine/rendering/d3d12/queue.cpp`
- Test: `tests/rendering/d3d12/queue_tests.cpp`

**Interfaces:**
- Produces: `GpuSubmissionSerialV1{backend_device_generation,queue_id,fence_value}`（Design §12.2）、completion-gated allocator pool。

- [ ] **Step 1: fence未完了allocator reuse、cross-generation submission、queue wait cycle testを書く**
- [ ] **Step 2: failureを確認する**
- [ ] **Step 3: Direct／Compute／Copy queue、monotonic fence、poolを実装する**
- [ ] **Step 4: 100,000 submission stressを実行する**

Expected: duplicate fence 0、early reset 0、cycleはcompile前に`diagnostic.rendering.d3d12.queue-wait-cycle`で拒否。

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
- Produces: `D3d12BarrierMappingEntryV1`、barrier groups、`D3d12QueueHandoffV1`、`diagnostic.rendering.d3d12.alias-discard-conflict`。

- [ ] **Step 1: 全logical usage、queue別layout、cross-queue handoff、unknown usageのtable-driven testを書く**

Queue testは次の5行をclosed matrixとして含める。

| Case | Exact expected result |
|---|---|
| Compute pass on `compute0` | sampled textureは`COMPUTE_QUEUE_SHADER_RESOURCE` |
| Compute intent merged into `direct0` | sampled textureは`DIRECT_QUEUE_SHADER_RESOURCE` |
| Direct→Compute read-only | Directだけが`DIRECT_QUEUE_GENERIC_READ_COMPUTE_QUEUE_ACCESSIBLE`へtransitionし、Producer SignalとCompute Waitは同じ`GpuSubmissionSerialV1` |
| Compute→Direct write | Computeはqueue-neutral layoutへtransitionしてSignalし、Directが同じserialをWait後にDirect用layoutへtransition |
| illegal queue-specific transition | Compute上の`DIRECT_QUEUE_*`とDirect上の`COMPUTE_QUEUE_*`を`diagnostic.rendering.d3d12.barrier-validation-failed`で拒否 |
- [ ] **Step 2: alias conflict fixtureを追加する**

Fixtureはbefore texture write、write flush、layout transition、same-memory after texture、`LayoutBefore=UNDEFINED`、DISCARD requestを含む。

- [ ] **Step 3: compile failureを確認する**
- [ ] **Step 4: Sync／Access／Layout mapping、Signal／Wait handoff、二段alias initializationを実装する**
- [ ] **Step 5: Debug Layer／GPU validationを有効にしてtestを実行する**

Expected: combined DISCARD planは拒否、separate barrier→Discard/Clear/Copy planはPASS、5行のqueue matrixは全PASS、cross-queue edgeごとにhandoff exact一件、legacy runtime barrier call 0。

### Task 9: Encoding／submission／traceを実装する

**Files:**
- Create: `engine/rendering/d3d12/encoding.cpp`
- Test: `tests/rendering/d3d12/encoding_tests.cpp`

**Interfaces:**
- Produces: `Backend::compile`、`Backend::submit`、bounded `D3d12EncodingTraceV1`。

- [ ] **Step 1: deterministic plan、producer Signal／consumer Wait exact serial、trace bound、continuation testを書く**
- [ ] **Step 2: failureを確認する**
- [ ] **Step 3: Graph→resource→descriptor→barrier／queue handoff→command list→submitのpure compile pathを実装する**
- [ ] **Step 4: shuffled recording worker count 1／2／4／8で比較する**

Expected: plan hash、submission dependency、render output hashが全worker countで一致し、全cross-queue edgeのtraceでproducer serialとconsumer wait refがexact一致する。

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

### Task 12: Runtime／Application candidate Package bindingを実装する

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
- [ ] **Step 4: Development／Shipping Configurationのcandidate Packageをbuildしread-backする**

Expected: Main EXE export 619／`.\D3D12\`、Core DLL exact hash、Development Layers version一致、Shipping Configuration candidateのLayers absent。CX0／CX1ではRelease Activation Receiptを発行しない。

### Task 13: Backend C1 QualificationとCandidate boundaryを閉じる

**Files:**
- Create: `tools/rendering/d3d12_qualification/c1-matrix.v1.json`
- Create: `tools/rendering/d3d12_qualification/expected-gates.v1.json`
- Create: `tests/rendering/d3d12/fixtures/hardware-smoke.mcd`
- Create: `tests/rendering/d3d12/fixtures/device-loss-injection.mcd`
- Test: `tests/rendering/d3d12/qualification_tests.cpp`

**Interfaces:**
- Produces: Target別`D3d12BackendQualificationReceiptV1`。Product Activation writerではない。

- [ ] **Step 1: Backend C1と後段Product Gateをphase付きmatrixへ固定する**

Matrix schemaは`target_id`、`gate_id`、`fixture_id`、`owner_phase_id`、`required_for_task_13`、`hardware_row`、`profile_hash`、`expected_gate_ref`を必須にする。Task 13はPhase 2 Backend C1の`fixture.product.windows-empty-scene`、`fixture.rendering.d3d12-warp-conformance`、`fixture.rendering.d3d12-hardware-smoke`のNVIDIA／AMD／Intel row、`fixture.rendering.d3d12-device-loss-injection`を`target.windows.editor`と`target.windows.desktop`の各profileへ展開し、全required rowを`required_for_task_13=true`にする。合成stressとEditor multi-windowは該当Targetのsubfixture、UMA rowはoptionalでC1判定へ含めない。`fixture.product.shooter-2d`はPhase 3／4、`fixture.product.shooter-arena-3d`はPhase 6、cross-genre／multi-target fixture setはPhase 8の後段入力として同じ内部matrixへ`required_for_task_13=false`で記録する。この内部planning projectionへProduct operational stateを表す`status` Fieldを設けず、未実行をRegistry rowへ保存しない。
- [ ] **Step 2: Backend C1のstatic／schema、conformance、hardware smoke、fault、performance、soakを実行し、raw結果をpreliminary Evidenceとして固定する**
- [ ] **Step 3: Candidate inputとclean candidate Git treeをread-backする**
- [ ] **Step 4: affected source／documentのhuman approvalを得て、same-definition Rebaseline／Product rebindingを完成する**
- [ ] **Step 5: new current bindingでTarget別Backend C1 Receiptを発行し、Evidence storeへpublishする**
- [ ] **Step 6: internal `candidate_locked`作成可否だけを報告する**

Expected: Phase 2 Backend C1必須row全合格時だけinternal candidate input生成。Step 4はControl Plane Design §6.1.1.1のA2→B2→C2→D2→T2→L2→P2を実行し、`wp.runtime.d3d12-backend=active`を含む非Control-Plane stateをbyte-exactに保持する。完成Receiptはnew current binding、Product Definition、Candidate、Target、raw Evidence closureへ束縛してEvidence storeへ置き、Product Gate evaluatorがexact refをread-time解決する。pre-rebaseline Evidenceはpreliminaryでありcompletion／後続Gateへ使わない。Phase 3／4／6／8はEvidence absentのため`unevaluated`、effective result=`fail`となるだけで、Product Registryまたはoperational snapshotを変更しない。文書、binary、単一GPU合格だけで`qualified`／`production`を書かない。`candidate_locked`もProduct state writerのread-back後だけが設定でき、ReleaseはProduct Planのconditional CX2 cutover／CX3 Shipping Work PackageとDecision Gateの責務である。

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

新規`diagnostic.rendering.d3d12.queue-wait-cycle`と`diagnostic.rendering.d3d12.alias-discard-conflict`に旧IDはない。旧ID出現数0、dual emission 0、old parser 0をArchitecture lintで検査する。

## Completion Gate

- current Control Plane baseline bindingをclosed kind別schemaでread-backし、Active Definition、D3D12 definition seed、Runtime ECS E0 completed Receipt closureの全hashをread-back済みである。
- D3D12正本、MCD、CX0 self-contained Header、tests、package policyが同じChangeSetへ存在し、CX1 fixture以外の`.ixx`／`.cppm`が0件である。
- production target `mirakan_rendering_d3d12`、aggregate test target `mirakan_d3d12_backend_tests`、CTest label `rendering.d3d12`が§1とexact一致する。
- Enhanced-only runtime pathでlegacy barrier symbolが0件である。
- Direct／Compute cross-queue edgeが同じ`GpuSubmissionSerialV1`のSignal／Waitと合法queue上のlayout transitionに閉じる。
- Graphics／Compute／Mesh PSO closure、RTV order、Sample Count／Qualityのnegative testsが通る。
- Alias layout transition／write flush＋DISCARD conflictがexact diagnosticで拒否される。
- Descriptor、HDR／SDR、Recovery、Pipeline LibraryがDesign §29の固定Gateを満たす。
- Development packageはSDKVersion一致Layers、Shipping Configuration candidate PackageはLayers 0、Core DLL hash一致である。この結果だけでRelease Activationしていない。
- Phase 2 Backend C1の`fixture.product.windows-empty-scene`、`fixture.rendering.d3d12-warp-conformance`、`fixture.rendering.d3d12-hardware-smoke`のNVIDIA／AMD／Intel row、`fixture.rendering.d3d12-device-loss-injection`がWindows Editor／Desktopの各Targetで完了し、二Target別Receiptが同じCandidate inputへbindする。
- implementation candidate treeに対するsame-definition Rebaseline／Product rebindingがA2→B2→C2→D2→T2→L2→P2で完成し、二Targetのfinal Receiptがnew current bindingへ束縛されている。pre-rebaseline EvidenceをWP completion／後続Gateへ使わない。
- Phase 3／4の`fixture.product.shooter-2d`、Phase 6の`fixture.product.shooter-arena-3d`、Phase 8のC2 matrixは内部Task 13 matrixで`required_for_task_13=false`であり、Evidence absent時はProduct Gateのread-time評価が`unevaluated`／effective `fail`となる。Registry／snapshotを更新せず、Backend C1の完了条件へ混入しない。
- Product state writerは別Gateであり、本計画が`qualified`／`production`を直接設定していない。
- D3D12 technical documentのapproved current lifecycle Headと、Task 13 candidate treeを取り込んだdestination same-definition Rebaseline bindingがcurrentである。完成D3D12 C1 Evidenceに対するProduct Ownerのfresh `ProductOperationalDecisionV1(kind=work_package_owner_acceptance)`後に、外部Schedulerだけが`wp.runtime.d3d12-backend`を`active->complete`へ進める。本計画はOwner acceptanceまたはWP completionを自己発行しない。
