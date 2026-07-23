# Miraikanai Engine AI-Readable Direct3D 12 Backend Design

- 作成日: 2026-07-22
- 状態: 詳細設計レビュー待ち
- 採用方針: Backend-neutral Render Graph＋private D3D12 Backend＋MCD projection（ユーザー承認済み）
- 対象Baseline: current Product snapshot内`CurrentControlPlaneBaselineBindingV1`からread-backしたexact architecture tree、Windows `target.windows.editor; target.windows.desktop`各profile version 1、Agility SDK 1.619.4／SDKVersion 619、D3D12MA 3.2.0
- 対象AI: `GameAuthoringProfileV1`と`EngineMaintenanceProfileV1`を分離
- 外部根拠検証日: 2026-07-23
- 上位横断計画: [Architecture Evolution Control Plane Design](2026-07-22-architecture-evolution-control-plane-design.md)
- 対象外: Engine実装code、MCD compiler実装、Vulkan／Metal Backend詳細設計、既存仕様の承認状態変更

## 1. 結論

MiraikanaiのDirect3D 12 Backendは、Project、Gameplay、Editor、Game Authoring AIへnative APIを公開する拡張面ではない。Backend-neutralな`CanonicalRenderExecutionPlanV1`を、検証可能な`D3d12EncodingPlanV1`へ一意に変換し、private AdapterがDXGI／D3D12へencodeするEngine製品実装である。

設計は次の4層に固定する。

```text
Authoring Semantic Layer
  -> RenderGraphDefinitionV1
  -> CanonicalRenderExecutionPlanV1
  -> D3d12EncodingPlanV1
  -> private DXGI／D3D12 execution
```

- Authoring AIはRender intent、Shader semantic、logical resource、Pass DAGだけを扱う。
- Developer AIはR4／A2の`EngineMaintenanceProfileV1`でD3D12正本、MCD、private source、fixtureを扱う。
- D3D12 native object、descriptor handle、GPU virtual address、queue fence値、HRESULTはProject／Save／Package Schemaへ保存しない。
- Toolchainの固定、Runtime capability、Engine採用Policy、実測Qualificationを別dimensionとして扱う。
- Windows C1はEnhanced Barriersだけを実行pathとし、legacy barrier runtime fallbackを持たない。
- Backendの全判断は公式制約、公式推奨、Engine判断、実測Profileのいずれかへ分類し、出典不明の「best practice」を正本へ入れない。

## 2. 完了条件

本設計のArchitecture反映は、次をすべて満たした状態を完了とする。

1. `docs/architecture/06-rendering/d3d12-backend.md`がD3D12固有mapping、lifetime、failure、qualificationの一意Ownerになる。
2. Render Graphはlogical plan、D3D12 Backendはnative mapping、WindowsはHWND／OS lifecycle、Toolchainはversion pinだけを所有する。
3. Authoring AIとDeveloper AIが別Profile、別Tool Catalog、別Authorizationであり、Authoring AIからD3D12 operationが0件である。
4. Adapter選択、Device作成、Capability query、Queue、Fence、Command Allocator、Descriptor、Memory、Barrier、Pipeline、Surface、Present、Device lossがclosed state／typed contractで閉じる。
5. C1必須Capabilityとoptional Capabilityが一意に定義され、未対応機能を別機能へ黙って近似しない。
6. 全logical stage／access／usage／resource kindがEnhanced Barrierのsync／access／layoutまたは明示的なinvalidへ完全写像される。
7. Descriptor、Resource、Command Allocator、Command List、Upload／Readback segment、Swap Chain bufferの再利用条件がqueue completionから一意に決まる。
8. HRESULT、Debug Layer、GBV、DRED、DXGI statusがstable diagnosticとprivate attachmentへ一意に写像される。
9. Device loss後の停止、Evidence freeze、destroy、recreate、rehydrate、fault終了がclosed state machineになる。
10. `D3d12EncodingTraceV1`からGraph pass、logical access、native barrier group、descriptor range、command list、submission serial、presentを前後方向へ追跡できる。
11. MCDからC++ type、validator、AI read projection、diagnostic、fixtureを生成でき、COM pointerまたはnative handleをwire型へ含めない。
12. NVIDIA、AMD、Intel、WARP Development referenceのpositive／negative／fault fixtureと、Debug Layer／GBV zero-error Gateが定義される。
13. Legacy barrier、multi-adapter、runtime shader compile、arbitrary native plugin、manual residency、runtime defragmentationのC1非採用が明記される。
14. 全外部根拠にURL、確認日、適用対象、Miraikanai判断との差分が記録される。
15. 実装が存在しない状態でTarget適合、性能、回復成功を合格済みと表示しない。

## 3. 非目標

- Direct3D 12 APIをAuthoring AIまたはProject extension surfaceにすること。
- Unreal Engine、Unity、GodotのRHI／Plugin API互換を作ること。
- Vulkan／Metalを仮想的に想定して最大公約数RHIを再設計すること。
- C1でlinked／unlinked multi-adapter、explicit multi-node、cross-adapter resourceを実装すること。
- C1でlegacy Resource Barrierをruntime fallbackとして実装すること。
- C1でmanual `Evict`／`MakeResident`、runtime defragmentation、GPU upload heapをProduction利用すること。
- ProjectまたはAIへraw command callback、native resource import、descriptor index、root parameter、register／space、compiler optionを公開すること。
- Hardwareまたはdriver固有の最適化を実測なしにProduction既定へすること。
- PIX、RenderDoc、Nsight、Radeon GPU Profiler等の外部Toolを正本またはRelease依存にすること。

## 4. 判断Authorityと根拠

各規則は次の`decision_authority`を一つだけ持つ。

| Authority | 意味 | 必須根拠 | 変更条件 |
|---|---|---|---|
| `external_requirement` | API／runtime／driverのvalidity制約 | Microsoft API／DirectX-Specsのexact URLと確認日 | 外部仕様更新とconformance再検証 |
| `external_recommendation` | Microsoftまたは採用Libraryが明示する推奨 | 公式文書の推奨箇所 | Engine比較、Target実測、回帰 |
| `miraikan_engine_decision` | MiraikanaiのOwner／security／complexity判断 | 比較案、採用理由、非採用理由 | Architecture reviewと全影響fixture |
| `measured_profile_decision` | Device／Target別の性能選択 | Qualification Receipt、driver／SDK／hardware identity | Evidence freshness切れまたは再測定 |

```text
D3d12DecisionRecordV1 {
  decision_id
  decision_authority
  subject_contract_id
  miraikan_interpretation
  target_profile_ids[]
  external_evidence_refs[]
  qualification_receipt_refs[]
}

D3d12ExternalEvidenceV1 {
  evidence_id
  evidence_authority: external_requirement | external_recommendation
  source_uri
  source_version_or_date
  verified_at
  extracted_constraint_id
}
```

外部仕様にない数値を`external_recommendation`と表示しない。Microsoftが複数の正しい方式を許す場合、選択結果は`miraikan_engine_decision`または`measured_profile_decision`とする。Engineの選択が外部API制約または推奨に依存する場合も、`decision_authority`を複合値にせず、外部事実を`external_evidence_refs`で参照する。`external_requirement`／`external_recommendation`は1件以上のExternal Evidenceを必須、`measured_profile_decision`は1件以上のQualification Receiptを必須とする。

## 5. 有名Engineとの比較と採用判断

| Engine | 公式構造から確認した点 | Miraikanaiで採用 | 採用しない点 |
|---|---|---|---|
| Unreal Engine | High-level RendererをRDG、RHI、D3D12RHIへ分離。RDGがresource lifetime、async compute、transition、validation、trace可視化を所有 | Graph宣言、private Backend、早期validation、machine-readable trace | Game制作層からD3D12RHI native escape hatchへ到達する方式 |
| Unity | SRP上のRenderGraphがhandle、setup／compile／execute、resource lifetimeを所有。Render Graph Viewerを提供 | handle、Graph phase、resource lifetime、Viewer相当trace | Native PluginへDevice、Swap Chain、Queue、Fence、Resourceを公開する方式 |
| Godot | Scene、RenderingServer、RenderingDevice、D3D12 driverを分離。上位はopaque RIDを使用 | Server／Device／Driver分離、opaque identity、Backend別private実装 | Projectから低水準RenderingDeviceを直接操作する方式 |

有名Engineはいずれも低水準APIを抽象化するが、型の意味、AI権限、公式根拠、failure remediationをLLM向けcontractとして閉じてはいない。Miraikanaiは抽象化だけでなく、次を追加する。

- Stable Contract IDとversion。
- `decision_authority`とofficial evidence ref。
- Authoring／Developer AIのTool Catalog分離。
- deterministic encoding planとtrace。
- stable diagnostic、negative fixture、Target Qualification Receipt。
- unknown field／unknown mappingのfail-closed rejection。

## 6. Ownerと権限境界

### 6.1 正本Owner

| 主題 | Owner |
|---|---|
| logical resource／pass／queue class／hazard／canonical Graph | Render Graph |
| D3D12 Adapter、Device、Queue、Fence、Descriptor、Memory、Barrier、PSO、Swap Chain、DRED | D3D12 Backend |
| HWND、monitor、DPI、session／power event、application lifecycle | Windows Platform |
| Agility SDK、DXC、D3D12MA、Windows SDKのversion／hash／license | Toolchain／Dependencies |
| common CPU／GPU／memory budget、measurement envelope | Runtime Performance／Capacity |
| Project Shader semantic、Fact、Target artifact、Qualification | Project Shader |
| AI authorization、R4／A2、human approval | AI Security／Approval |
| Evidence envelope、freshness、Receipt | AI Verification／Provenance |

D3D12 Backendは上位Domainの意味、Product activation、共通Budget、OS package、Shader Source意味を再定義しない。

### 6.2 AI Profile

`GameAuthoringProfileV1`は次だけを取得できる。

- `RendererCapabilityProjectionV1`
- `ResolvedRendererProfileV1`
- Backend-neutral errorとremediation
- Project Shader／Material／Render GraphのAuthoring Operation

`RendererCapabilityProjectionV1`は、render-graph.md正本の`RendererCapabilitySignatureV1`からAdapter LUID、driver文字列等のnative識別子をfield maskで除外したAuthoring向けredacted projectionであり、定義は§25のrender-graph.md変更で追加する。

`D3d12*` type、native error、driver message、adapter LUID、raw trace、Backend sourceはGame Authoring Tool Catalogへ出さない。

`EngineMaintenanceProfileV1`はR4／A2で次を取得できる。

- D3D12正本、MCD、private source slice、公式根拠。
- exact Toolchain lock、Target Profile、Capability snapshot。
- mapping explain、encoding trace、fault report、fixture result。
- Engine ChangeSetのplan／proposal。

Developer AIは実機Receipt生成、合格宣言、Approval、Activation、Releaseを自己実行できない。Domain ownerと独立Reviewerのread-backを必須とする。

## 7. ModuleとDirectory

```text
engine/rendering/
├─ contracts/                    # Backend-neutral public contracts
├─ render_graph/                 # canonical graph compiler
├─ resources/                    # Engine handle registry
├─ pipelines/                    # Engine pipeline identity/cache policy
├─ surfaces/                     # Backend-neutral surface lease
└─ d3d12/                        # private D3D12 Backend実装
   └─ include/mirakan/rendering/d3d12/backend.hpp  # CX0 public surface

tools/rendering/d3d12_qualification/
schemas/mirakan/rendering/d3d12/
```

`engine/rendering/d3d12/`配下のexact file構成（CX0 public Header、実装unit、tests、fixtures）は§34に従い[D3D12 Backend Implementation Plan](2026-07-22-d3d12-backend-implementation-plan.md)の§1 File mapを正本とし、本書は再掲しない。bootstrap、capability、queue、descriptor、memory、barrier、pipeline、surface、diagnostics、qualificationの責務分割は同File mapのResponsibility列が所有する。

設計と実装計画を結ぶ名称は次のclosed tableに固定し、aliasまたは別綴りを作らない。

| Concept | Canonical name／path |
|---|---|
| CX0 public surface | `engine/rendering/d3d12/include/mirakan/rendering/d3d12/backend.hpp` |
| future Primary Named Module | `mirakan.rendering.d3d12.adapter` |
| implementation root | `engine/rendering/d3d12/` |
| MCD root | `schemas/mirakan/rendering/d3d12/` |
| Qualification root | `tools/rendering/d3d12_qualification/` |
| window handoff | `IApplicationSurface`（Windows Platform Owner） |
| backend Qualification evidence | `D3d12BackendQualificationReceiptV1` |
| submission identity | `GpuSubmissionSerialV1{backend_device_generation, queue_id, fence_value}` |
| diagnostic namespace | `diagnostic.rendering.d3d12.*` |
| production CMake target | `mirakan_rendering_d3d12` |
| aggregate test target | `mirakan_d3d12_backend_tests` |
| CTest label | `rendering.d3d12` |

実装計画§1／§2、CMake、MCD、test registrationはこの表をexact複写する。表と異なる旧型名、別directory、別target名を互換aliasで吸収せず、Architecture lintで拒否する。

CX0のcomposition rootはself-containedな`backend.hpp`だけをincludeする。CMake Componentは将来のPrimary Named Module名`mirakan.rendering.d3d12.adapter`だけを登録し、CX1の隔離fixtureを除いてproduction `.ixx`／`.cppm`を作成しない。CX2 cutover後は同名Named Moduleだけをcomposition rootへ公開する。D3D12 header、WRL／COM helper、DXGI type、D3D12MA typeはCX0ではprivate implementation、CX2以降はGlobal Module Fragmentまたはprivate implementationだけでincludeする。

現在の`mirakan::ui_d3d12_adapter`／`mirakan.ui.d3d12.adapter`は廃止し、`mirakan::ui_render_graph_adapter`／`mirakan.ui.render_graph.adapter`へ置換する。UIは`MirakanUiDrawPacketV1`からlogical UI Passを生成し、vertex／index upload、pipeline、descriptor、clip encode、submissionはD3D12 Backendが所有する。

## 8. Canonical MCD Contract

### 8.1 Engine Source Policy

| Type | Class | Owner／Writer | 永続性 |
|---|---|---|---|
| `D3d12BackendProfileV1` | Source Policy | Engine build／human-approved R4 | Engine baselineへ保存 |
| `D3d12FeatureRequirementV1` | Source Policy | D3D12 Backend | Engine baselineへ保存 |
| `D3d12BindingLayoutPolicyV1` | Source Policy | D3D12 Backend＋Project Shader consumer review | Engine baselineへ保存 |
| `D3d12BarrierMappingTableV1` | Source Policy | D3D12 Backend＋Render Graph consumer review | Engine baselineへ保存 |
| `D3d12PresentPolicyV1` | Source Policy | D3D12 Backend＋Windows consumer review | Engine baselineへ保存 |

### 8.2 Runtime Derived

| Type | 生成元 | 用途 | 保存規則 |
|---|---|---|---|
| `D3d12AdapterCandidateV1` | DXGI enumeration | deterministic selection | Session内だけ |
| `D3d12CapabilitySnapshotV1` | Device query | Profile resolve／diagnostic | Receipt attachmentだけ |
| `D3d12DeviceCreatePlanV1` | Profile＋candidate | bootstrap transaction | Session内だけ |
| `D3d12QueuePlanV1` | Profile＋Capability | queue topology | Device generation内 |
| `D3d12DescriptorPlanV1` | Profile＋Pipeline closure | heap partition | Device generation内 |
| `D3d12MemoryPlanV1` | Profile＋budget | allocator／pool／ring | Device generation内 |
| `D3d12EncodingPlanV1` | Canonical Graph＋all plans | native encode input | Frame diagnostic用、Projectへ保存禁止 |

### 8.3 Evidence

| Type | 内容 |
|---|---|
| `D3d12EncodingTraceV1` | Graph、pass、resource、barrier、descriptor、command list、submissionのbounded trace |
| `D3d12DeviceFaultReportV1` | Device removed reason、DRED、last completed serial、live generation |
| `D3d12BackendQualificationReceiptV1` | Target／adapter／driver／SDK／Profile／fixture closure |

MCDはnative pointer、COM identity、CPU／GPU descriptor handle、GPU virtual address、HRESULT message pointerをFieldに持たない。Native値が必要なEvidenceはprivate binary attachmentへ格納し、MCDはattachment hash、size、content type、redaction classだけを参照する。

## 9. C1 Windows D3D12 Baseline

| 項目 | C1決定 | `decision_authority` | `external_evidence_refs`の役割 |
|---|---|---|---|
| Runtime | Agility SDK 1.619.4、SDKVersion 619 | `miraikan_engine_decision` | official release identity、app-local activation制約 |
| Adapter API | DXGI 1.6、`EnumAdapterByGpuPreference(DXGI_GPU_PREFERENCE_HIGH_PERFORMANCE)` | `miraikan_engine_decision` | enumeration APIとpreferenceの意味 |
| Minimum feature level | `D3D_FEATURE_LEVEL_12_0` | `miraikan_engine_decision` | Device creationとFeature Levelの意味 |
| Shader model | 6.6以上 | `miraikan_engine_decision` | Shader Model queryの意味 |
| Root Signature | 1.1必須 | `miraikan_engine_decision` | version queryと1.1 flag semantics |
| Resource Binding Tier | Tier 2以上 | `miraikan_engine_decision` | Tier queryとdescriptor制約 |
| Resource Heap Tier | Tier 1／2両対応。D3D12MAにheap分離を委譲 | `miraikan_engine_decision` | Tier queryとD3D12MA推奨flag |
| Enhanced Barriers | `D3D12_FEATURE_DATA_D3D12_OPTIONS12::EnhancedBarriersSupported == TRUE`必須 | `miraikan_engine_decision` | `D3D12_FEATURE_D3D12_OPTIONS12` queryとoptional support制約 |
| Node | node 0、node mask 1だけ使用 | `miraikan_engine_decision` | node maskのAPI semantics |
| Queue | Direct 1、Compute 1、Copy 1、Normal priority | `miraikan_engine_decision` | Queue type／priorityのvalidity制約 |
| Frames in flight | 3 | `miraikan_engine_decision` | なし。Engine capacity policy |
| Presentation model | DXGI flip model | `external_recommendation` | Microsoftのflip model推奨 |
| Swap Chain parameters | `FLIP_DISCARD`、buffer 3、sample count 1 | `miraikan_engine_decision` | flip-model validityとtearing制約 |
| Descriptor binding | RS 1.1 generated table、unbounded bindless禁止 | `miraikan_engine_decision` | heap／root signature制約 |
| Memory allocator | D3D12MA 3.2.0、1 Allocator／Device | `miraikan_engine_decision` | D3D12MAのAllocator scopeと推奨flag |
| Barrier path | Enhancedだけ。legacy runtime path 0 | `miraikan_engine_decision` | Enhanced Barrier support／API semantics |
| Multi-adapter | 非対応。追加nodeを使用しない | `miraikan_engine_decision` | なし。C1 scope policy |
| WARP | Development／conformanceだけ。Production fallback禁止 | `miraikan_engine_decision` | WARP adapterのAPI semantics |
| Ray／Mesh／VRS／Work Graph | optional Capability。C1 baseline非必須 | `miraikan_engine_decision` | 各optional feature queryの意味 |

Feature levelは性能を表さず、optional featureを包含しない。Device作成成功だけをCapability合格にせず、全必須`CheckFeatureSupport`とformat／sample queryを実行する。

## 10. Bootstrap state machine

```text
uninitialized
  -> sdk_verified
  -> diagnostics_configured
  -> factory_created
  -> adapters_enumerated
  -> adapter_selected
  -> device_created
  -> capabilities_resolved
  -> queues_created
  -> allocators_created
  -> descriptors_created
  -> surfaces_ready
  -> running

any pre-running failure -> bootstrap_rejected
running device fault    -> quiescing -> evidence_frozen -> rebuilding | faulted
```

順序を変更しない。

1. EXE exportの`D3D12SDKVersion`、`D3D12SDKPath`、app-local runtime hashをBuild Receiptと照合する。
2. Development／ProfileではDRED、Debug Layer、GBV policyをDevice作成前に設定する。
3. `CreateDXGIFactory2`でProfileに応じたfactory debug flagを設定する。
4. Adapter候補を列挙し、candidate snapshotをcanonicalizeする。
5. selection policyで一Adapterを選ぶ。
6. feature level 12_0でDeviceを作る。
7. 必須／optional capability queryを固定順で実行する。
8. Capability snapshotをProfileへ照合する。
9. Queue、Fence、D3D12MA、Descriptor heap、Pipeline cacheを作る。
10. WindowごとのSurfaceを作成する。
11. 空Graph validation／clear／present／read-back smoke合格後だけ`running`を公開する。

途中失敗時に低いFeature Level、System D3D12、legacy barrier、WARPへ黙ってretryしない。許可された別Adapter候補を試す場合も、各candidateのfailureとselection traceを保持し、同じProfileを満たす候補に限る。

## 11. Adapter selectionとCapability query

### 11.1 Selection

候補を次の順に評価する。

1. Userが明示したAdapter LUIDが存在し、同じProfileへ適合すれば選択する。
2. それ以外は`DXGI_GPU_PREFERENCE_HIGH_PERFORMANCE`順に列挙する。
3. Software／Remote AdapterをProduction候補から除外する。
4. `D3D12CreateDevice(candidate, FL12_0)` probeと必須queryを実行する。
5. 適合候補のenumeration order、LUID byte順をtie-breakにして一つ選ぶ。

VRAM容量だけで選ばない。Vendor IDを優先度へ使わない。Adapter overrideがstale／missing／不適合なら別Adapterへ黙って切り替えず、候補と不足RequirementをUserへ返す。明示的な`automatic`選択時だけ次候補へ進む。

WARPは`fixture.rendering.d3d12-warp-conformance`で明示したDevelopment Jobだけ選択可能とする。実GPU初期化失敗時のProduction fallbackにはしない。

### 11.2 Query set

`D3d12CapabilitySnapshotV1`は少なくとも次を記録する。

- Adapter LUID、Vendor／Device／Subsystem／Revision ID、dedicated／shared memory。
- Driver version、Agility SDK／runtime hash、D3D12 device interface revision。
- maximum Feature Level、Shader Model、Root Signature version。
- `D3D12_OPTIONS`～使用する最大OPTIONS revisionのquery resultとHRESULT。
- Resource Binding Tier、Resource Heap Tier、Tiled Resources Tier。
- Enhanced Barriers、Wave Ops、Native 16-bit、Typed UAV、Conservative Raster、ROV。
- Raytracing、Mesh Shader、VRS、Sampler Feedback、Work Graph等のoptional tier。
- Architecture／UMA／CacheCoherentUMA、GPU VA support。
- 全Engine `PixelFormat`のformat support、plane count、sample count／quality。
- DXGI tearing、display color space、HDR／Advanced Color capability。
- local／non-local memory budget、usage、reservation。

Queryが`E_INVALIDARG`を返した場合、構造体revisionがRuntimeに未対応であることとFeature非対応を区別する。必須query不明は不適合、optional query不明は`unknown`であり`unsupported`と混同しない。ただし`unknown` CapabilityをActivationしない。

## 12. Queue、Fence、Command lifecycle

### 12.1 Queue topology

| Queue ID | Native type | 所有work |
|---|---|---|
| `direct0` | `D3D12_COMMAND_LIST_TYPE_DIRECT` | graphics、present前transition、qualified compute fallback |
| `compute0` | `D3D12_COMMAND_LIST_TYPE_COMPUTE` | Graphがasync computeと解決したworkだけ |
| `copy0` | `D3D12_COMMAND_LIST_TYPE_COPY` | upload、readback、streaming copy |

全QueueはNormal priority、disable timeout flagなし、node mask 1とする。Global realtime priorityを使用しない。

Async ComputeはQueueが作成できるだけでは有効化しない。Driver／GPU／workload別Qualification Receiptがあり、Directへ統合したreference pathとcorrectness一致し、性能回帰がない場合だけ`async_compute_qualified`とする。不適合時はGraph compile時にDirectへ統合し、同frameで動的に切り替えない。

### 12.2 Submission identity

```text
GpuSubmissionSerialV1 {
  backend_device_generation
  queue_id
  fence_value
}
```

Fence値はQueueごとに1から単調増加し、0をinvalidとする。永続Artifact、Project identity、Save、Replayへ保存しない。全resource／descriptor／allocator／ring segmentは最後に使用した全Queueのserial集合を持つ。

### 12.3 Allocator／List pool

- Command Allocatorは`queue_id × recording_thread_id × frame_slot`でpoolする。
- 対応Queue fence completion前にAllocatorをresetしない。
- Command Listはclose後にsubmitし、List object自体はsubmit後にreset候補へ戻せるが、Allocatorはcompletionまで保持する。
- Recording中、closed、submitted、completed、retiredをclosed stateとする。
- CPU waitはframe slot再利用、readback要求、surface quiesce、device teardownだけで許可する。Pass間同期にCPU waitを使わない。
- Queue間同期はGraph edgeから生成したQueue signal／waitだけとし、Backendが隠れたwaitを追加しない。

Frames in flightは3固定で、frame slotをframe numberの剰余だけで再利用せず、そのslotが保持する全Queue serial completionを検証する。

## 13. Resource、Memory、D3D12MA

### 13.1 Allocation domain

| Domain | 方針 |
|---|---|
| Persistent general GPU resource | D3D12MA default pool |
| Persistent RT／DS／UAV critical resource | high residency priority custom pool、計測後にcommitted指定可能 |
| Render Graph transient | aliasable placed resource＋Graph lifetime plan |
| Upload | persistently mapped upload buffer＋frame-segmented virtual allocation |
| Readback | persistently mapped readback buffer＋request-segmented allocation |
| External surface | DXGI所有Back Buffer。D3D12MA外 |
| Acceleration structure | optional Capability専用pool、general bufferとidentity分離 |

DeviceごとにD3D12MA Allocatorを一つ作り、`D3D12MA_RECOMMENDED_ALLOCATOR_FLAGS`をbaselineとする。`SINGLETHREADED`を使用しない。Resource作成はEnhanced Barrierのinitial layoutを取る`CreateResource3`を使い、`ID3D12Device10`取得失敗をTarget不適合とする。

Default poolを既定とし、custom poolはresource separation、residency priority、statistics、aliasingの明示理由があるDomainだけに限定する。Pool block sizeを根拠なく固定せず、0でAllocatorへ委譲する。実測により固定する場合は`measured_profile_decision`へ昇格する。

### 13.2 Budget／residency

- `IDXGIAdapter3::QueryVideoMemoryInfo`とD3D12MA Budgetを同一sampling windowで取得する。
- soft／hard pressureと共通Budget値はRuntime Performance Ownerから参照する。
- 非必須allocationは`WITHIN_BUDGET`で要求し、超過時は作成しない。
- 必須allocationはrebuild可能cacheを規定順でevict後、一度だけretryする。
- retry失敗時はGraph／surface generationをpublishしない。
- C1はmanual `Evict`／`MakeResident`を使わない。OS demotionとresidency priorityを使用し、manual residencyは別Profile／Qualification後に追加する。
- C1はruntime defragmentationを行わない。Loading境界でもSourceからrebuild可能なpool再生成を優先する。
- GPU upload heapはC1 Productionで使用しない。将来Capabilityとして独立Qualificationする。

### 13.3 Initializationとalias

Placed resourceの既存memory内容を0と仮定しない。最初のread前にfull write、clear、copy、またはdiscard付きactivationを必須とする。Graph Compilerが証明できないpartial initializationを拒否する。

Alias前resourceをdeactivateし、全hazardを完了してwriteをvisibleにした後、Alias後resourceを`UNDEFINED`から必要layoutへactivateする。Alias後resourceは内容保持を前提にせず、最初の有効read前にinitializeする。

## 14. Descriptor model

### 14.1 Heap topology

Device generationごとに次を一度だけ作る。

| Heap | Shader visible | C1 capacity | Partition |
|---|---:|---:|---|
| CBV／SRV／UAV | yes | 262,144 | persistent 131,072、transient 32,768×3、recovery 32,768 |
| Sampler | yes | 2,048 | persistent 1,024、transient 256×3、recovery 256 |
| RTV | no | page 4,096、max 65,536 | generation付きpage allocator |
| DSV | no | page 2,048、max 16,384 | generation付きpage allocator |
| CPU CBV／SRV／UAV staging | no | page 16,384、max 262,144 | descriptor master／copy source |

CapacityはMicrosoft必須値ではなくC1 Engine decisionである。変更はProfile version、memory impact、descriptor exhaustion fixtureを必要とする。

一Command ListでCBV／SRV／UAV heapとSampler heapを一つずつだけbindし、Pass間でshader-visible heapを切り替えない。Shader-visible heapをCPU copy sourceに使わず、CPU staging heapからcopyする。

### 14.2 Identity／lifetime

```text
DescriptorHandleV1 {
  heap_class
  index
  generation
  backend_device_generation
}
```

Native CPU／GPU handleをMCDへ格納しない。Persistent slotの再利用は最後の全Queue serial completion後だけ許可する。Transient segmentは対応frame slotの全Queue completion後に一括resetする。Descriptorが参照するResource generationとBackend device generationをbinding時に検証する。

Heap枯渇時は別heapへ黙って切り替えず`diagnostic.rendering.d3d12.descriptor-capacity-exceeded`でGraph compileを拒否する。Frame途中のheap grow、descriptor table再base、silent resource dropを禁止する。

## 15. Binding、Root Signature、PSO

### 15.1 Generated binding layout

Project Shaderはregister／space／root parameterを指定しない。`ShaderInterface`と`D3d12BindingLayoutPolicyV1`から次を生成する。

| Space | 所有 | 用途 |
|---:|---|---|
| 0 | Engine | frame global、immutable catalog |
| 1 | Engine | view／surface |
| 2 | Material | material parameter／texture |
| 3 | Draw／Object | per-draw／instance |
| 4 | Pass | pass-local SRV／UAV／CBV |
| 5 | Ray／optional | qualified optional feature |

Root Signature 1.1のEngine baselineを次に固定する。

1. Frame CBV root descriptor。
2. View CBV root descriptor。
3. 最大16 DWORDのPass root constants。
4. Generated CBV／SRV／UAV descriptor table。
5. Generated sampler descriptor table。

Engine hard capは32 DWORDで、D3D12の64 DWORD上限より厳しくする。未使用Shader stageはdeny flagを付ける。Immutable samplerはCatalogにexact宣言がある場合だけstatic samplerにし、dynamic samplerはSampler heapへ置く。

RS 1.1のstatic／volatile flagはResourceの更新契約から生成する。Staticと宣言したdescriptorまたはdataを許可期間中に変更しない。Project／AIはflagを指定できない。

### 15.2 Pipeline identity

```text
D3d12PipelineKeyV1 = sha256(McdCanonicalBinaryV1(
  pipeline_kind,
  shader_artifact_set_hash,
  shader_interface_hash,
  root_signature_hash,
  node_mask,
  pipeline_flags,
  graphics_state_or_null,
  target_profile_hash,
  backend_profile_hash
))

pipeline_kind = graphics | compute | mesh

graphics_state_or_null {
  blend_state_hash
  sample_mask
  rasterizer_state_hash
  depth_stencil_state_hash
  input_layout_hash
  index_buffer_strip_cut_value
  primitive_topology_type
  stream_output_hash
  view_instancing_hash
  render_target_count: 0..8
  render_target_formats[8]  # slot 0..7の固定順。未使用slotはUNKNOWN
  depth_stencil_format
  sample_count
  sample_quality
}
```

`pipeline_kind`ごとのshader／subobject closureを次に固定する。

| kind | 必須shader | 任意shader | 禁止shader／state |
|---|---|---|---|
| `graphics` | `VS` | `PS`; `GS`; `HS`＋`DS`のpair | `CS`; `AS`; `MS`。`HS`と`DS`の片側だけを禁止 |
| `compute` | `CS` | なし | `VS/PS/GS/HS/DS/AS/MS`と`graphics_state_or_null`のnon-null値 |
| `mesh` | `MS` | `AS`; `PS` | `VS/GS/HS/DS/CS`、input layout、index strip cut、stream output |

`graphics`と`mesh`は`graphics_state_or_null`を必須、`compute`は`null`を必須とする。`render_target_count=N`ではslot `0..N-1`がexact format、slot `N..7`が`DXGI_FORMAT_UNKNOWN`でなければならない。Arrayをsetとしてsortせずslot順をidentityへ含める。`sample_count`と`sample_quality`は`DXGI_SAMPLE_DESC`の独立Fieldで、両方をhashへ含める。`sample_count=1`では`sample_quality=0`だけを受理し、MSAAは対象formatごとに`D3D12_FEATURE_MULTISAMPLE_QUALITY_LEVELS`で`NumQualityLevels > sample_quality`を確認する。

Pipeline State Streamは同一subobject typeの重複、derived subobjectの競合、kindに不正なsubobjectを作成前に拒否する。`D3DX12ParsePipelineStream`のvalidation結果とEngine closure validatorの両方をconformance fixtureへ保存する。Cached PSO bytesとPipeline Library bytesはrebuild可能cacheであり、`D3d12PipelineKeyV1`の入力にしない。

Runtime source compileを行わない。Critical PipelineはPackage buildで列挙し、起動前またはLoading境界で作成する。Running中のcritical PSO missはfallback shaderへ置換せずRenderer faultまたは事前登録済み同意味fallbackへ次generationから切り替える。

Pipeline LibraryはCapability queryでsupportを確認し、Adapter LUID、driver version、Agility runtime hash、Backend Profile hash、Root Signature hashでcacheを分離する。Library load／deserialize失敗はcacheだけを破棄して同じsigned Shader artifactから再生成する。Pipeline cacheをUser dataまたはPackage identityにしない。

## 16. Enhanced Barrier mapping

### 16.1 原則

Windows C1はEnhanced Barrier native pathだけを持つ。`D3D12_RESOURCE_STATES`へのruntime translatorを実装しない。既存Render Graph Qualificationの「Enhanced／legacy」は次へ修正する。

- Production conformance: Enhanced Barrierだけ。
- legacy: Microsoft互換性理解用のoffline reference fixtureに限定し、Runtime artifactへ含めない。

Enhanced Barriersがoptionalであるため、`D3D12_FEATURE_DATA_D3D12_OPTIONS12::EnhancedBarriersSupported`がfalse／unknownならTarget不適合とし、legacyへfallbackしない。

### 16.2 Mapping contract

```text
D3d12BarrierMappingEntryV1 {
  logical_stage_mask
  logical_access
  logical_usage
  resource_kind
  queue_class
  sync_mask
  access_mask
  texture_layout
  discard_policy
  split_policy
  validity
  evidence_ref
}
```

主要mappingの「後続access側要件」を次に固定する。この表は`SyncAfter`／`AccessAfter`／`LayoutAfter`を引くための表であり、`Before`側を暗黙に含まない。

| Logical usage | `SyncAfter` | `AccessAfter` | `LayoutAfter` |
|---|---|---|---|
| color attachment write | `RENDER_TARGET` | `RENDER_TARGET` | `RENDER_TARGET` |
| depth／stencil write | `DEPTH_STENCIL` | `DEPTH_STENCIL_WRITE` | `DEPTH_STENCIL_WRITE` |
| depth／stencil read | `DEPTH_STENCIL` | `DEPTH_STENCIL_READ` | `DEPTH_STENCIL_READ` |
| pixel shader sample | `PIXEL_SHADING` | `SHADER_RESOURCE` | `DIRECT_QUEUE_SHADER_RESOURCE` |
| vertex／mesh shader sample | exact shading stage | `SHADER_RESOURCE` | `DIRECT_QUEUE_SHADER_RESOURCE` |
| compute shader sample | `COMPUTE_SHADING` | `SHADER_RESOURCE` | Queue対応`SHADER_RESOURCE` |
| UAV graphics／compute | exact shading stage | `UNORDERED_ACCESS` | Queue対応`UNORDERED_ACCESS` |
| copy source | `COPY` | `COPY_SOURCE` | Queue対応`COPY_SOURCE` |
| copy destination | `COPY` | `COPY_DEST` | Queue対応`COPY_DEST` |
| resolve source | `RESOLVE` | `RESOLVE_SOURCE` | `RESOLVE_SOURCE` |
| resolve destination | `RESOLVE` | `RESOLVE_DEST` | `RESOLVE_DEST` |
| present | `NONE` | `NO_ACCESS` | `PRESENT` |
| indirect argument | `EXECUTE_INDIRECT` | `INDIRECT_ARGUMENT` | Bufferのためlayoutなし |
| vertex buffer | `VERTEX_SHADING` | `VERTEX_BUFFER` | Bufferのためlayoutなし |
| index buffer | `INDEX_INPUT` | `INDEX_BUFFER` | Bufferのためlayoutなし |
| constant buffer | consumer stage | `CONSTANT_BUFFER` | Bufferのためlayoutなし |
| acceleration structure read | `RAYTRACING` | `RAYTRACING_ACCELERATION_STRUCTURE_READ` | Bufferのためlayoutなし |
| acceleration structure build | build stage | read／write exact mask | Bufferのためlayoutなし |

「Queue対応」layoutは`D3d12BarrierMappingEntryV1.queue_class`をkey入力として解決し、`direct0`実行では`DIRECT_QUEUE_*`、`compute0`実行では`COMPUTE_QUEUE_*`を選ぶ。§12.1によりasync compute未Qualificationでcompute workが`direct0`へ統合された場合も同規則を適用し、compute shader sampleは`DIRECT_QUEUE_SHADER_RESOURCE`へ解決する。

cross-queue resource edgeはbarrierだけで同期したとみなさず、次の`D3d12QueueHandoffV1`をencoding planへexact一件生成する。

```text
D3d12QueueHandoffV1 {
  resource_ref
  producer_queue_id
  producer_signal_serial: GpuSubmissionSerialV1
  handoff_layout
  consumer_queue_id
  consumer_wait_serial_ref: exact producer_signal_serial
  transition_queue_id: optional exact queue
  consumer_layout
}
```

1. Producerは最後のwrite／readと必要なflushを完了し、Producer queueで合法なqueue-neutral layout、またはDirect→Compute read-only handoff専用の`DIRECT_QUEUE_GENERIC_READ_COMPUTE_QUEUE_ACCESSIBLE`へtransitionする。
2. Producer submission完了後にproducer queueが`producer_signal_serial.fence_value`をSignalし、Consumer queueは`consumer_wait_serial_ref`が指す同じserialをWaitする。serialの`backend_device_generation`と`queue_id`はProducerとexact一致させ、CPU polling、別fence値、submission完了順を依存表現に使わない。
3. Wait後に別layoutが必要なら、`transition_queue_id`で`LayoutBefore`／`LayoutAfter`の双方が合法なbarrierを記録してからConsumer accessを開始する。Direct／Compute共用read layoutへ入る／出るtransitionはDirect queueだけに置き、Compute queueではaccessだけを行う。
4. Compute→Direct write handoffはCompute queueでqueue-neutral layoutへ移してSignalし、Direct queueがWait後にDirect用layoutへtransitionする。Compute queueへ`DIRECT_QUEUE_*`、Direct queueへ`COMPUTE_QUEUE_*` transitionを記録しない。
5. 合法なhandoff layout、transition queue、acyclic Signal／Wait edgeを一意に作れない場合はasync computeをDirectへcompile-time統合し、それも不可能なら`diagnostic.rendering.d3d12.barrier-validation-failed`でGraph compileを拒否する。同frameの動的queue切替は行わない。

このprotocolの公式制約は§27のEnhanced Barriers／Command・Fence Evidenceへbindし、queue ownership、fence、wait、transitionの全Fieldを`D3d12EncodingTraceV1`へ残す。

Barrier生成は次の順で一意に決める。

1. 直前のlogical accessから`SyncBefore`／`AccessBefore`／`LayoutBefore`を引く。resource activationで先行accessがない場合だけ`SyncBefore = NONE`、`AccessBefore = NO_ACCESS`を許可する。
2. 後続logical accessから上表の`After`側を引く。Presentは後続GPU accessがないため`SyncAfter = NONE`、`AccessAfter = NO_ACCESS`とし、先行workのexact scopeは`SyncBefore`／`AccessBefore`に保持する。
3. `SyncBefore = NONE`なら`AccessBefore = NO_ACCESS`、`SyncAfter = NONE`なら`AccessAfter = NO_ACCESS`を必須とする。`NONE`は他Sync bitと結合しない。
4. 同一subresourceでbarrierが連続する場合、後続barrierの`SyncBefore`は先行barrierの`SyncAfter`を完全に含む。含まないplanをrejectする。
5. Textureはqueue typeと`LayoutBefore`／`LayoutAfter`のcompatibilityを検査する。Bufferはlayoutを持たず、Global Barrierでtexture layoutを変更しない。

`COMMON`をunknown usageのfallbackにしない。TextureのQueue-specific layoutを別Queueで使う場合はQueue ownership edgeとcompatible layout transitionを必須にする。Compute QueueからRender Target／Depth Stencil／Present layoutへtransitionしない。

Syncは必要な最小stage集合から生成し、理由なく`ALL`を使わない。Mappingに存在しないStage／Access／Usage combinationはGraph compile errorにする。

### 16.3 Hazard／alias／split

- Read-after-readでlayout互換ならbarrierを生成しない。
- Write-after-read、read-after-write、write-after-writeはexact access／sync hazardを生成する。
- Bufferとsimultaneous-access textureでも、一Writerと依存read／writeのhazardを検証する。
- Global Barrierはtexture layoutを変更できないため、UAV memory orderingまたはalias memory visibilityだけに使用する。
- Aliasはbefore resource deactivate、after resource `UNDEFINED` activation、必要なdiscard／initializeを一組として生成する。
- After resourceのactivation barrierで`D3D12_TEXTURE_BARRIER_FLAG_DISCARD`を使えるのは、`LayoutBefore=UNDEFINED`で、同じalias memoryを先行使用した全before-resource barrierがlayout transitionを行わず、かつwrite flushを必要としない場合だけである。いずれかがlayout transitionまたはwrite flushを行う場合は同一barrier groupでDISCARDを併用せず、先行barrier完了後に別の`DiscardResource`／Clear／Copy initializationを発行する。
- `fixture.rendering.d3d12-alias-discard-conflict`は、before textureがwrite後にlayout transition／flushを要求し、after textureが`UNDEFINED` activation＋DISCARDを要求する入力を与える。Compilerはcombined barrier planを`diagnostic.rendering.d3d12.alias-discard-conflict`で拒否し、二段planだけを受理する。
- C1はsplit barrierを生成しない。Full barrierでcorrectnessを固定し、splitは実測Profileと独立conformance後に追加する。

## 17. Encoding planとFrame data flow

```text
RenderGraphDefinitionV1
  -> CanonicalRenderExecutionPlanV1
  -> validate Backend Profile／Capability
  -> resolve resource allocation／alias
  -> resolve descriptor ranges／binding layout
  -> map logical hazards to Enhanced Barrier groups
  -> partition command lists by queue／recording batch
  -> generate queue signal／wait
  -> D3d12EncodingPlanV1
  -> parallel record
  -> close／validate
  -> submit
  -> present
  -> D3d12EncodingTraceV1
```

`D3d12EncodingPlanV1`は少なくとも次を持つ。

- Backend device／surface／Graph generation。
- ordered command batch、queue、pass Stable ID集合。
- Resource handleからprivate registry lookupするgeneration check。
- Barrier groupと元のlogical hazard edge。
- Descriptor allocation rangeと元Shader Interface binding。
- Pipeline keyとQualification済みartifact ref。
- Queue wait／signal edgeとsubmission serial reservation。
- Present plan、surface generation、color／tearing／latency policy。
- Capacity reservationとfailure disposition。

Plan生成後はimmutableとし、record workerがqueue、barrier、descriptor、pipelineを追加しない。Record中に不足が判明した場合はPlan全体を失敗させ、同frameへ隠れたallocationまたはfallback passを挿入しない。

## 18. Surface、Swap Chain、Present

### 18.1 Ownership

Windows PlatformはHWND、monitor、DPI、lifecycle eventを所有する。D3D12 Backendはその`IApplicationSurface`からDXGI Swap Chain、Back Buffer、RTV、Presentを所有する。一top-level HWNDにつき一つのSwap Chainとsurface generationを持つ。

### 18.2 C1 policy

| 項目 | 決定 |
|---|---|
| Swap effect | `DXGI_SWAP_EFFECT_FLIP_DISCARD` |
| Buffer count | 3 |
| Sample count | 1。MSAAはoffscreen resolve後にpresent |
| SDR | `R8G8B8A8_UNORM`＋sRGB／Rec.709 policy |
| HDR／Advanced Color | `R16G16B16A16_FLOAT`＋scRGBをgeneral C1 path |
| Tearing | DXGI query成功＋windowed＋sync interval 0だけ |
| Frame latency | waitable object使用。Game 2、Editor 3 |
| Exclusive fullscreen | C1非採用。borderless window |

Flip modelはMicrosoft推奨として採用し、flip discard、3 buffer、waitable objectはMiraikanai C1 decisionとする。HDRはMicrosoftのgeneral-purpose推奨であるFP16 scRGBを採用する。HDR非対応display、window跨ぎ、color space変更ではnew surface generationを作る。

Present modeを次に閉じる。

| Mode | Sync interval | Flags | 不適合時 |
|---|---:|---|---|
| `vsync_on` | 1 | 0 | Surface fault |
| `vsync_off_tearing` | 0 | `ALLOW_TEARING` | Unsupported。silent vsync化禁止 |
| `benchmark_unthrottled` | 0 | 条件に応じtearing | Development／Qualificationだけ |

Resize／DPI／monitor／HDR変更では新規submissionを当該surfaceへ停止し、旧Back Bufferを参照する全serial完了後にRTVとBack Bufferを破棄し、同じcreation flagsで`ResizeBuffers`またはSwap Chain再作成を行う。古いsurface generationのPacket／Planを拒否する。

## 19. Editor UI integration

`MirakanUiDrawPacketV1`はD3D12を知らない。`ui_render_graph_adapter`は次のlogical resource／passだけを生成する。

- immutable vertex／index upload request。
- glyph／image texture logical binding。
- clip tableとscissor plan。
- registered UI pipeline IDとtyped parameter。
- display-resolution UI render target。

D3D12 Backendがupload ring、descriptor、root signature、PSO、barrier、submissionを解決する。UI専用Device、Queue、Descriptor heap、Swap Chainを作らない。Scene tone map後、Present前のUI Passとして同じGraphへ接続する。

Device loss中はUI GPU frameを生成せず、Editor UI規約どおりAuthoring journal／Workspaceをflushする。Recovery UIはD3D12に依存しないWin32 entryとする。

## 20. DiagnosticとDevice loss

### 20.1 Diagnostic setup

Development／ProfileではDevice作成前に次を設定する。

- D3D12 Debug Layer。
- GPU-based validation。通常Developmentのbounded fixtureとProfileに限定し、全Editor soakで常時有効にしない。
- DRED auto-breadcrumb、page fault reporting。
- DXGI factory debug。
- InfoQueue message callback／break policy。

Shipping packageへD3D12SDKLayers.dll、raw debug callback、source path、unredacted object名を含めない。DRED取得可能性はShippingでも維持するが、privacy／size policyに従う。

### 20.2 Stable diagnostic

```text
diagnostic.rendering.d3d12.agility-runtime-mismatch
diagnostic.rendering.d3d12.adapter-not-found
diagnostic.rendering.d3d12.adapter-profile-unsupported
diagnostic.rendering.d3d12.device-create-failed
diagnostic.rendering.d3d12.capability-query-failed
diagnostic.rendering.d3d12.enhanced-barriers-required
diagnostic.rendering.d3d12.queue-create-failed
diagnostic.rendering.d3d12.queue-wait-cycle
diagnostic.rendering.d3d12.descriptor-capacity-exceeded
diagnostic.rendering.d3d12.descriptor-stale
diagnostic.rendering.d3d12.resource-stale
diagnostic.rendering.d3d12.memory-budget-exceeded
diagnostic.rendering.d3d12.barrier-mapping-missing
diagnostic.rendering.d3d12.barrier-validation-failed
diagnostic.rendering.d3d12.pipeline-unavailable
diagnostic.rendering.d3d12.surface-stale
diagnostic.rendering.d3d12.present-mode-unsupported
diagnostic.rendering.d3d12.present-failed
diagnostic.rendering.d3d12.device-removed
diagnostic.rendering.d3d12.device-recovery-failed
diagnostic.rendering.d3d12.alias-discard-conflict
```

旧`MIRAKAN-D3D12-*` 19 IDは同じcondition名の上記dotted IDへ一回限りでclean replaceする。Alias、dual emission、旧ID parserを残さない。`queue-wait-cycle`（Graph compile時のqueue wait cycle拒否）と`alias-discard-conflict`は新規IDであり旧対応行を持たない。

HRESULT値、API名、source location、driver message、DRED nodeはprivate attachmentでありstable diagnostic identityにしない。同じContract failureはdriver wordingに依存せず同じDiagnostic ID、subject ID、remediationを返す。

### 20.3 Recovery state machine

```text
running
  -> fault_detected
  -> submissions_stopped
  -> evidence_frozen
  -> surfaces_detached
  -> gpu_objects_released
  -> device_recreated
  -> capabilities_revalidated
  -> resources_rehydrated
  -> surfaces_recreated
  -> running

any recreate／revalidate／rehydrate failure -> faulted
```

Device removal検出後は新規submissionを停止し、`GetDeviceRemovedReason`、DRED、last submitted／completed serial、Graph／surface／resource generationをfreezeする。GPU objectからSourceを復元せず、Cooked Artifactとpublished snapshotから再構築する。

同一sessionの自動recoveryは一回だけ許可する。再失敗、Capability変化、Adapter変更、required artifact欠損ではGameHostをfault終了し、EditorはWin32 recovery entryを提示する。無限retry、同frame継続、WARP fallback、別Adapterへの無通知切替を禁止する。

## 21. AI向けDiscoveryとTrace

### 21.1 Developer AI Operation

| Operation | Authority | 結果 |
|---|---|---|
| `operation.d3d12.read_profile` | R0／A2 | exact Backend Profile、根拠、非目標 |
| `operation.d3d12.read_capability` | R0／A2 | bounded Capability snapshot、requirement diff |
| `operation.d3d12.explain_mapping` | R0／A2 | logical accessからbarrier／descriptor／pipelineへの根拠付き写像 |
| `operation.d3d12.read_trace` | R0／A2 | bounded encoding trace、omitted range、continuation |
| `operation.d3d12.validate_plan` | R0 Job／A2 | Schema、lifetime、mapping、capacity、Target diagnostic |
| `operation.d3d12.run_fixture` | R2 Job／A2 | isolated fixture request。Source writeなし |
| `operation.d3d12.plan_change` | R1／A2 | Sourceを変更しないEngine change plan |
| `operation.d3d12.propose_change` | R4／A2 | expected revisionへのEngine ChangeSet。Commit／Approvalなし |

全結果はEngine baseline hash、D3D12 Backend Profile hash、Toolchain lock hash、Target Profile hash、Capability snapshot hash、query hashを持つ。Traceは既定256 pass／1,024 resource／4,096 barrier、最大1,024 pass／8,192 resource／32,768 barrierとし、超過時は`omitted_ranges`とcontinuationを返す。

### 21.2 Trace correlation

```text
Graph pass ID
  <-> command batch ID
  <-> command list ID
  <-> barrier group ID
  <-> descriptor allocation ID
  <-> pipeline key
  <-> submission serial
  <-> present ID
```

すべてStable／generation付きEngine identityで相関し、pointer、CPU descriptor address、GPU addressをquery keyにしない。DRED breadcrumbはcommand list／pass markerを介して上記identityへprojectionする。

## 22. Failure policy

| Failure | 結果 | 禁止fallback |
|---|---|---|
| Agility runtime hash／SDKVersion不一致 | 起動拒否 | System D3D12 |
| User Adapter override不適合 | 候補と不足Requirementを表示 | 自動別Adapter |
| Automatic候補全不適合 | 起動拒否 | WARP Production |
| Enhanced Barriers未対応 | Target不適合 | legacy barrier |
| Required format／sample不足 | Profile不適合 | 別format／sampleへsilent変換 |
| Descriptor capacity不足 | Graph compile拒否 | frame途中heap grow／switch |
| Memory hard cap超過 | cache eviction後一度retry、再失敗 | draw skip／default resource |
| Barrier mapping欠落 | Graph compile拒否 | COMMON／ALL fallback |
| Pipeline miss | Loading境界で生成またはfault | runtime source compile |
| Surface generation stale | Plan／Packet拒否 | 古いBack Buffer使用 |
| Present mode unsupported | Settings apply拒否 | silent mode変更 |
| Device removed | Evidence freeze後一回recovery | 無限retry／Adapter切替 |
| Recovery再失敗 | GameHost fault終了／Editor recovery | Authoring Source破棄 |

## 23. Qualification

### 23.1 Static／Schema

- D3D12／DXGI／D3D12MA typeがprivate directory外へ出ないinclude／module／symbol scan。
- Authoring Tool Catalogに`operation.d3d12.*`が0件であるnegative gate。
- 全MCD valid／invalid／round-trip／canonical hash fixture。
- 全logical stage／access／usage／resource kindのmapping coverage 100%。unknown combination rejection。
- Root Signature 32 DWORD cap、register space一意性、descriptor range overlapなし。
- Descriptor capacity partition合計一致、generation／device generation validation。

### 23.2 Backend conformance

- Adapter enumeration、override、automatic selection、software／remote rejection、WARP explicit-only。
- Required／optional／unknown Capability、query `E_INVALIDARG`、format／sample matrix。
- Direct／Compute／Copy Queue、cross-queue wait、serial monotonic、Allocator premature reset rejection。
- Enhanced Barrierのtexture／buffer／global、queue-specific layout、UAV、alias、discard、Present。
- Resource create／destroy、D3D12MA default／custom、placed／committed、upload／readback、budget failure。
- Descriptor stale、resource stale、heap exhaustion、frame segment reuse、device generation invalidation。
- Root Signature／PSO／Pipeline Library cold／warm／corrupt／driver change。
- Flip discard、resize、DPI、Alt+Tab、sleep、multi-window、tearing、HDR／SDR、monitor移動。

### 23.3 Diagnostic／fault

- Debug Layer、GBV、InfoQueueのzero-error positive fixture。
- 未宣言resource、stale descriptor、use-after-free、invalid barrier、allocator early resetのnegative fixture。
- GPU page fault、invalid descriptor、TDR相当、Present device removedのDRED capture。
- Device loss一回recovery、二回目fault終了、Capability変化、artifact欠損。
- Shipping closureからSDK Layers、raw source path、unredacted object名を除外するscan。

### 23.4 Hardware matrix

| Class | Reference | 役割 |
|---|---|---|
| NVIDIA | RTX 3060以上の固定driver profile | discrete baseline |
| AMD | RX 6600以上の固定driver profile | discrete baseline |
| Intel | Arc A-series固定driver profile | third-vendor baseline |
| UMA | 必須Profileを満たすqualified UMA device | optional row。UMA／CacheCoherentUMA shared budget検証用であり、C1 `qualified`判定の必須入力ではない |
| WARP | app-local WARP redistributable（`Microsoft.Direct3D.WARP`）＋SDK固定 | deterministic Development reference、性能Gate対象外 |
| Negative | Enhanced BarriersまたはSM 6.6不適合Device／mock | startup rejection |

Hardware model、driver version、OS build、SDK hash、Profile hashをReceiptへ固定する。WARP合格を実GPU合格、単一Vendor合格をWindows合格、Debug Layer zero-errorだけをvisual／performance合格とみなさない。

WARPはinbox実装がOS buildごとに変わるため、OS build下限指定だけではbyte同一性を担保しない。`Microsoft.Direct3D.WARP` redistributableのversionとhashをtoolchain-dependencies.mdのlock表でexact pinし（現時点未固定。pin追加は§25のtoolchain変更で行う）、`fixture.rendering.d3d12-warp-conformance`はapp-local WARP DLLを明示ロードして、そのhashをQualification Receiptへ記録する。

### 23.5 Performance／soak

- 8時間Editor multi-window soak、resize／HDR／Play開始停止／device recovery。
- 合成stress fixture（Phase 2）と`fixture.product.shooter-2d`／`fixture.product.shooter-arena-3d`（各Phase 3／6完了後の再Qualification。§29.2の段階化に従う）のcold／warm PSO、descriptor peak、transient alias、upload／readback peak。
- Direct統合pathとasync compute pathの同一output、frame time、queue overlap。
- Memory pressure 75／85／95／100%相当でcache eviction、allocation rejection、recovery。
- 10,000 surface resize、100 device recreation fault run、1,000 pipeline cache invalidation。

共通測定window、warm-up、percentile、regression thresholdはRuntime Performance Ownerを参照し、本書で複写しない。

### 23.6 Phase-owned Qualification

Backend実装の完了とProduct content統合を次の四Gateへ分離する。後段Gateの未実行をPhase 2 Backend C1のfailureにしない。各Phaseの完成ReceiptはEvidence storeへ同じBackend Profile／Candidate hash付きでpublishし、Product operational projectionがGate評価時にexact refをread-time解決する。Definition RegistryへReceiptや実行状態を書き戻さない。

| Gate | Required fixture／evidence | Owner phase | Phase 2 completion input |
|---|---|---|---|
| Backend C1 | `fixture.product.windows-empty-scene`、`fixture.rendering.d3d12-warp-conformance`、`fixture.rendering.d3d12-hardware-smoke`のNVIDIA／AMD／Intel row、`fixture.rendering.d3d12-device-loss-injection`。合成stressとEditor multi-windowをsubfixtureとして含む | Phase 2 `phase.editor-runtime` | required |
| Product integration | `fixture.product.shooter-2d` | Phase 3 `phase.manual-2d`とPhase 4 `phase.ai-authoring-mvp-a` | not required。Evidence absentならProduct Gateはread-time `unevaluated`／effective `fail` |
| 3D integration | `fixture.product.shooter-arena-3d` | Phase 6 `phase.manual-3d-mvp-b` | not required。Evidence absentならProduct Gateはread-time `unevaluated`／effective `fail` |
| C2 matrix | Product Registryのcross-genre／multi-target fixture set | Phase 8 `phase.production-capability` | not required。Evidence absentならProduct Gateはread-time `unevaluated`／effective `fail` |

Task 13はBackend C1だけを実行する。source binding下の測定結果はpreliminary Evidenceに固定し、candidate treeのsame-definition Rebaseline／Product rebinding後、destination current bindingへTarget別`D3d12BackendQualificationReceiptV1`を発行する。Product integration、3D integration、C2 matrixは同じ数値式を再利用するが、各Product Work PackageのCandidate／Target Receiptであり、D3D12 Backend Work Packageの完了条件へ逆流させない。

## 24. Architecture／MCD反映順序

1. Architecture GovernanceのMetadata V2 grammar確定後、D3D12 file bytesを`ArchitectureDocumentCoreV1`へ束縛し、post-bootstrap新規文書branchの`DocumentLifecycleRecordV1.submit_review`（previous Core／Change Manifestなし、from=null、to=`review`）でcurrent headを追加する。bootstrap 5 Owner限定の`construction_seed`を使わず、state／approval refをmetadataへ保存しない。
2. Render Graphに`CanonicalRenderExecutionPlanV1`、D3D12 consumer境界、Enhanced-only Qualificationを明記する。
3. WindowsからD3D12 resource／frame／device lossを新Ownerへexact linkする。
4. ToolchainのAgility／DXC／D3D12MA pinと本書のCapability利用を接続する。
5. Editor UIの`ui_d3d12_adapter`を`ui_render_graph_adapter`へ置換し、D3D12 ownershipを除去する。
6. Executable ContractsへD3D12 MCD kindとA2-only projection ruleを登録する。
7. AI Securityへ`EngineMaintenanceProfileV1`のD3D12 Operation authorityを接続する。
8. AI VerificationへD3D12 Receipt、hardware matrix、freshnessを接続する。
9. MCD Source Policy、Runtime Derived、Evidence typeとfixtureを追加する。
10. [D3D12 Backend Implementation Plan](2026-07-22-d3d12-backend-implementation-plan.md)のTaskを順に実行する。
11. WARP／mockによるcontract implementationから開始し、NVIDIA／AMD／Intel実機へ拡張する。
12. 全Gate合格後にWindows C1 Candidateを作る。文書承認だけでCapabilityをactivateしない。

## 25. 既存正本への影響

| Existing spec | 必須変更 |
|---|---|
| `architecture/README.md` | Rendering正本へD3D12 Backendを追加。生成件数を更新 |
| `00-product/product-plan.md` | Phase 2 Windows vertical sliceのD3D12 Target Gate参照 |
| `01-governance/ai-security-approval.md` | A2-only D3D12 Tool Catalog、Authoring非公開negative gate |
| `01-governance/ai-verification-provenance.md` | Backend Qualification Receipt、hardware／driver／SDK freshness |
| `02-foundation/core-architecture.md` | Render Graph Portからprivate D3D12 Adapterへの依存方向 |
| `02-foundation/toolchain-dependencies.md` | pin維持、Agility export／package inspection、D3D12MA 3.2.0 API確認、`Microsoft.Direct3D.WARP` redistributableのversion／hash pin追加（値は未固定。lock ChangeSetでexact確定） |
| `02-foundation/executable-contracts.md` | D3D12 MCD kind、A2-only operation projection、trace bound |
| `02-foundation/cpp23-modules.md` | `mirakan.rendering.d3d12.adapter`、UI target rename |
| `02-foundation/naming-project-layout.md` | D3D12 private directoryとVendor表記例外 |
| `03-authoring/editor-ui-framework.md` | `ui_d3d12_adapter`廃止、UI Render Graph Adapter、header path矛盾修正 |
| `04-runtime/performance-capacity.md` | D3D12 budget telemetry projectionとReference measurement接続 |
| `04-runtime/debugging-observability-replay.md` | Encoding trace、DRED attachment、device generation相関 |
| `06-rendering/project-shader.md` | generated D3D12 binding layout／RS／PSO artifact binding |
| `06-rendering/render-graph.md` | logical plan Owner、D3D12 consumer、Enhanced-only conformance、`RendererCapabilityProjectionV1`（`RendererCapabilitySignatureV1`のAuthoring向けredacted projection、native識別子除外field mask付き）の定義追加 |
| `07-platform/windows.md` | HWND OwnerとD3D12 Surface Owner分離、new spec link |

## 26. Architecture lint追加

1. D3D12／DXGI／D3D12MA symbolが許可directory外にない。
2. D3D12 canonical Contract Ownerが一つである。
3. Authoring Tool CatalogにA2-only D3D12 Operationがない。
4. 全logical Render AccessにD3D12 mappingまたはinvalid ruleがある。
5. `COMMON`／`ALL`をunknown fallbackとして使うmappingがない。
6. Enhanced-only Profileにlegacy runtime barrier symbol／fixture activationがない。
7. Descriptor partition合計、capacity、frame segment数が整合する。
8. Pipeline keyに`pipeline_kind`、kind別shader／subobject closure、RTV slot 0～7順、sample count、sample quality、全fixed-function hashがある。
9. Alias activationのDISCARDがbefore-resource layout transitionまたはwrite flushと同じplanへ併用されていない。
10. D3D12 typeをProject／Save Schemaへ保存せず、Packageにはtyped artifact refだけを保存している。
11. 旧`MIRAKAN-D3D12-*`、`windows_desktop_v1`、`d3d12_warp_conformance_v1`が0件である。
12. External evidence ref、authority、verified dateが全Decisionにある。
13. cross-queue edgeごとに`D3d12QueueHandoffV1`がexact一件あり、Producer SignalとConsumer Waitが同じ`GpuSubmissionSerialV1`を参照し、transition queueで両layoutが合法である。
14. Task 13の内部matrixでPhase 3／4／6／8 fixtureは`required_for_task_13=false`であり、Product Registry／operational snapshotに未実行stateを書かない。Evidence absent時の後段Gateはread-time `unevaluated`／effective `fail`となる。
15. `mirakan_rendering_d3d12`、`mirakan_d3d12_backend_tests`、`rendering.d3d12`が§7のclosed tableと一致する。

## 27. 公式一次資料と適用

| 対象 | 一次資料 | 適用 |
|---|---|---|
| Agility SDK | [Microsoft NuGet `Microsoft.Direct3D.D3D12` 1.619.4](https://www.nuget.org/packages/Microsoft.Direct3D.D3D12/1.619.4)、[Getting Started](https://devblogs.microsoft.com/directx/gettingstarted-dx12agility/) | stable 1.619.4／SDKVersion 619のartifact identity、EXE export、app-local runtime、Development layer分離。version／nupkg hashはToolchain lockだけが所有 |
| Capability | [CheckFeatureSupport](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-checkfeaturesupport)、[Hardware Feature Levels](https://learn.microsoft.com/en-us/windows/win32/direct3d12/hardware-feature-levels) | Device作成とoptional query分離、Feature Levelを性能扱いしない |
| Enhanced Barriers | [DirectX-Specs Enhanced Barriers](https://microsoft.github.io/DirectX-Specs/d3d/D3D12EnhancedBarriers.html) | sync／access／layout分離、optional query、alias／discard ordering、queue-specific layout、Direct限定transition、cross-queue fence／wait、validation |
| Command／Fence | [Executing and synchronizing command lists](https://learn.microsoft.com/en-us/windows/win32/direct3d12/executing-and-synchronizing-command-lists)、[Recording command lists](https://learn.microsoft.com/en-us/windows/win32/direct3d12/recording-command-lists-and-bundles) | Queue／Fence ownership、Allocator reuse condition |
| Descriptor | [Descriptors overview](https://learn.microsoft.com/en-us/windows/win32/direct3d12/descriptors-overview)、[Shader-visible heaps](https://learn.microsoft.com/en-us/windows/win32/direct3d12/shader-visible-descriptor-heaps) | handle lifetime、heap、heap switch制約 |
| Root Signature | [Root Signature limits](https://learn.microsoft.com/en-us/windows/win32/direct3d12/root-signature-limits)、[Root Signature 1.1](https://learn.microsoft.com/en-us/windows/win32/direct3d12/root-signature-version-1-1) | DWORD cost、static／volatile semantics |
| Pipeline | [Pipeline State Subobject Type](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/ne-d3d12-d3d12_pipeline_state_subobject_type)、[Pipeline State Stream](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_pipeline_state_stream_desc)、[Managing Graphics Pipeline State](https://learn.microsoft.com/en-us/windows/win32/direct3d12/managing-graphics-pipeline-state-in-direct3d-12)、[Pipeline Library](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device1-createpipelinelibrary) | kind別subobject closure、RTV slot順、SampleDesc Count／Quality、cache support query、rebuildable cache |
| Presentation | [For best performance, use DXGI flip model](https://learn.microsoft.com/en-us/windows/win32/direct3ddxgi/for-best-performance--use-dxgi-flip-model)、[Variable refresh rate](https://learn.microsoft.com/en-us/windows/win32/direct3ddxgi/variable-refresh-rate-displays) | flip model、tearing query、Present flag整合 |
| HDR | [Use DirectX with Advanced Color](https://learn.microsoft.com/en-us/windows/win32/direct3darticles/high-dynamic-range) | FP16 scRGB general path、color space query |
| Device fault | [Use DRED to diagnose GPU faults](https://learn.microsoft.com/en-us/windows/win32/direct3d12/use-dred) | Device作成前設定、breadcrumb／page fault、private Evidence |
| Memory | [D3D12MA 3.2.0 documentation](https://gpuopen-librariesandsdks.github.io/D3D12MemoryAllocator/html/)、[official repository releases](https://github.com/GPUOpen-LibrariesAndSDKs/D3D12MemoryAllocator/releases)、[resource aliasing](https://gpuopen-librariesandsdks.github.io/D3D12MemoryAllocator/html/resource_aliasing.html) | release／header identity、`D3D12MA_RECOMMENDED_ALLOCATOR_FLAGS`、one Allocator／Device、thread-safe default、Enhanced initial layout、budget／pool／alias後の全resource初期化規則 |
| Unreal | [Render Dependency Graph](https://dev.epicgames.com/documentation/en-us/unreal-engine/render-dependency-graph-in-unreal-engine)、[D3D12DynamicRHI](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/D3D12RHI/ID3D12DynamicRHI) | RDG／RHI分離、validation／trace、native escape非採用 |
| Unity | [Render Graph fundamentals](https://docs.unity.cn/Packages/com.unity.render-pipelines.core%4017.0/manual/render-graph-fundamentals.html)、[D3D12 native interface](https://github.com/Unity-Technologies/NativeRenderingPlugin/blob/master/PluginSource/source/Unity/IUnityGraphicsD3D12.h) | handle／Graph phase、native plugin公開非採用 |
| Godot | [Internal rendering architecture](https://docs.godotengine.org/en/stable/engine_details/architecture/internal_rendering_architecture.html)、[RenderingServer](https://docs.godotengine.org/en/stable/classes/class_renderingserver.html) | Server／Device／driver分離、opaque identity |

上表は2026-07-22に公式一次資料へ再照合した。外部資料はAPI／Libraryの合法性、version identity、推奨flagだけを所有する。descriptor 20% headroom、HDR／SDR tolerance、30／60 run、recovery時間、8時間soakはMicrosoft、AMD、D3D12MAの推奨値ではなく、§29で固定するMiraikanai Qualification policyである。

## 28. Riskとmitigation

| Risk | Mitigation |
|---|---|
| D3D12詳細がRender Graphと重複 | logical semanticsとnative mappingでOwnerを分離し、同名型を再定義しない |
| MCDがnative API wrapperになる | native handle禁止、Source Policy／Derived Plan／EvidenceだけをSchema化 |
| AI Contextが巨大化 | operation別bounded slice、field mask、continuation、exact hash |
| 公式推奨とEngine判断を混同 | `decision_authority`とevidence refを必須化 |
| Enhanced-onlyで対応GPUが狭まる | C1 Targetとして明示し、不適合を正しく拒否。Legacy fallbackを隠さない |
| Descriptor固定capacity不足 | exhaustionをfail closed、Profile versionと実測で明示更新 |
| Async Computeが回帰 | Direct reference pathとTarget別Receiptがある場合だけ有効化 |
| Device recoveryがSourceを破損 | GPU stateから復元せずCooked Artifact＋published snapshotから再構築 |
| UIが別D3D12 ownerを作る | UI AdapterをRender Graphへ統合し、Device／Queue／Descriptor所有を禁止 |
| Pipeline cacheがdriver更新で壊れる | driver／SDK／Profile hashで分離し、失敗時はcacheだけ破棄 |

## 29. Qualification measurement contract

MicrosoftのAPI仕様はMiraikanai製品の数値合否閾値を規定しない。次の値は公式API制約を満たした上で採用するMiraikanai固有のBackend C1 Qualification policyであり、測定開始後に変更しない。変更はBackend Profile revision、理由、旧新比較Receiptを必要とする。Product contentの昇格は§23.6の後段Gateが別に所有する。

### 29.1 共通測定条件

- Reference hardware一台につきclean boot後に5 warm-up run、30 measured runを行う。
- Driver、OS build、Adapter LUID、Agility runtime hash、Backend Profile hash、Target Profile hash、fixture hashをReceiptへ固定する。
- Median、p95、maximumは30 measured runから計算する。失敗runを標本から除外しない。
- NVIDIA、AMD、Intelの各required matrix rowが同じGateを満たさなければC1を`qualified`にしない。

### 29.2 Descriptor headroom

測定fixtureは§23.6とProduct RegistryのPhaseに従い段階化する。Phase 2のBackend C1 descriptor入力は`fixture.product.windows-empty-scene`、合成stress fixture `fixture.rendering.d3d12-synthetic-stress`（procedural描画負荷でdescriptor／transient alias圧を再現する）、Editor multi-windowの3 fixtureとする。`fixture.product.shooter-2d`はPhase 3／4、`fixture.product.shooter-arena-3d`はPhase 6の完了後に同一Gate式で再Qualificationし、新しいper-WP ReceiptをEvidence storeへ追加する。既存Receiptを更新せず、存在しないPhaseのcontent fixtureをBackend C1 candidate生成の前提にしない。

各fixtureで、shader-visible CBV／SRV／UAV heapとSampler heapを別々に測る。各heapの`peak_live_descriptors / configured_capacity`が全runで`<= 0.80`、allocation rejection 0、stale descriptor diagnostic 0を合格条件とする。すなわち最低20% headroomを要求する。0.80超過時はcapacityを暗黙growせずProfile revisionを上げ、同じ30-run matrixを再実行する。

### 29.3 HDR／SDR visual tolerance

固定camera、固定exposure、固定seed、UI overlayなしのlinear FP16 captureを正本inputとする。SDR referenceとHDR tone-map後SDR projectionを同一1920×1080 cropで比較し、CIEDE2000のmedian `<= 1.0`、p95 `<= 2.0`、maximum `<= 5.0`、非有限pixel 0、clipped pixel率 `<= 0.1%`を全fixtureで満たす。意図的なHDR highlight差はmask artifactへ明示登録し、mask面積はframeの2%以下、mask hashをReceiptへ含める。Mask外のfailureを目視承認で上書きしない。

### 29.4 Device recovery

各required Adapter／driver rowで`DXGI_ERROR_DEVICE_REMOVED`、`DXGI_ERROR_DEVICE_RESET`、`DXGI_ERROR_DRIVER_INTERNAL_ERROR`を各10回、合計30回injectする。Editorは30/30でSource stateを保持して新Device generationへ復帰し、recovery開始から最初のvalid Presentまでp95 `<= 10.0 s`、maximum `<= 15.0 s`を満たす。GameHostは自動recovery一回だけを許可し、復帰不能時は15秒以内にsupport bundleを書いてclean fault終了する。二回目retry、Adapter無通知切替、Source state破損は一件で不合格とする。

### 29.5 Pipeline Library採用Gate

同じsigned Shader／PSO manifestを使い、cacheなしcold start 30 runとvalid library warm start 30 runを交互に測る。Pipeline Library pathをProductionで有効化できるのは、Loading boundaryまでのPSO preparation medianがcold median比`<= 0.85`、p95が`<= 1.05`、application launchからFirst Presentまでのp95が回帰せず、library deserialize failure後のcache-only rebuildが100%成功する場合だけである。条件未達ではlibraryをPackageまたはUser dataへ残さず、signed Shader artifactからの通常PSO生成を使用する。

### 29.6 Receiptでのみ解決する入力

Reference hardwareのexact driver version、Async ComputeをqualifiedにするAdapter／workload集合、各runの実測値はReceiptで解決する。これらを未決の設計値として扱わず、Receipt欠落時は対応optional pathをActivationしない。

## 30. Review checklist

- Authoring AIとDeveloper AIのProfile／Tool／Authorityが完全に分離されている。
- D3D12 Backend固有概念に一意Ownerがある。
- Toolchain pin、Runtime capability、Engine Policy、Evidenceが別dimensionである。
- Adapter selectionとfallbackが決定論的である。
- Enhanced-onlyとlegacy非採用が矛盾していない。
- Queue／Fence／Allocator／List／frame slotの再利用条件がcompletionへ結び付く。
- Descriptor capacity、partition、generation、retire条件が一意である。
- D3D12MA、alias、initialization、Budget、residencyのC1境界が一意である。
- Root Signature、register space、PSO key、Pipeline cache invalidationが一意である。
- 全logical accessがEnhanced Barrierへ写像される。
- Surface、Present、tearing、HDR、resize、multi-window規則が一意である。
- Device loss state machineに無限retryまたはsilent fallbackがない。
- AI traceがpointerなしでSource→Graph→native encode→submissionを追跡できる。
- 有名Engineの事実とMiraikanai判断を混同していない。
- 外部一次資料の確認日と適用箇所が明記されている。
- 実装なしで合格済みと表現していない。

## 31. Control Plane baseline binding

D3D12 ChangeSetの最初の入力は`D3d12BackendChangeSetV1.control_plane_baseline_binding`へinline保存したcurrent `CurrentControlPlaneBaselineBindingV1`である。kind=`bootstrap | rebaseline`のclosed branchから完成Approval／Envelope／TransactionとBaseline Coreをread-backし、Active Product Definition、Local Schema Catalog、Authority Binding Source Catalog、Toolchain、Trust／revocation、authoritative source Git treeを照合する。authorized Staging candidateはsource treeのdescendantかつchanged-path manifest内として別検証し、candidate deltaをdirty sourceと誤判定しない。Reader側でField集合を再列挙せず、missing／additional Field、stale current head、hash差、source dirty／Staging逸脱のいずれかで`diagnostic.architecture.baseline-mismatch`または`diagnostic.architecture.rebaseline-approval-invalid`を発行して停止する。初回Bootstrap Approvalを後続Rebaselineの代わりにしない。

値を本文へ仮記入しない。Preflight後にControl Planeがcurrent Product snapshotへpublishしたsource bindingを`D3d12BackendChangeSetV1.control_plane_baseline_binding`へbyte-exactに格納し、Task 2～13のStaging／preliminary Evidenceはそのsource bindingとcandidate ancestryを消費する。candidate tree承認後はControl Plane §6.1.1.1のsame-definition Rebaseline／Product rebindingを完成し、Target別final Receiptだけがdestination current bindingを消費する。旧binding EvidenceはpreliminaryのままWP completion／後続Gateへ使用せず、別branch名、日時、`latest`、現在HEADの推測で代用しない。

### 31.1 PreviewとRelease Activationの分離

D3D12のCX0 Header実装とCX1 Module fixtureはDevelopment、Test、candidate Package、internal Technology PreviewのEvidence生成だけに使う。Shipping Configurationで内部candidateをbuildしてもRelease Activation入力にはできず、`capability.rendering.d3d12-cx0`のTarget stateを`qualified`または`production`へ昇格しない。

Product cutoverは[Product Plan](../architecture/00-product/product-plan.md)のdeferred `wp.foundation.cpp23-cx2-cutover`／`capability.foundation.cpp23-cx2-cutover`と`wp.foundation.cpp23-cx3-shipping`／`capability.foundation.cpp23-cx3-shipping`、対応するPlanning Decision Gateだけが再開できる。CMakeの非Experimental `import std`、正式`/std:c++23`、全TargetのBuild／Tooling／ABI／Package／Release Receiptのいずれかが未成立ならCX0を維持し、HeaderからModule、またはcandidateからReleaseへ暗黙昇格しない。

## 32. Five-owner and package closure

| Owner | D3D12が消費／供給するexact boundary |
|---|---|
| Architecture Governance | D3D12 document metadata、Owner uniqueness、relation、lint diagnostic。D3D12はGovernance schemaを再定義しない |
| Compatibility／Evolution | `D3d12BackendProfileV1` revision、Agility／driver／pipeline cacheのartifact-class policy、breaking classification。Pipeline cacheはrebuild-only |
| Persistence／Save | D3D12 native object、descriptor、resource state、Pipeline LibraryをSaveしない。Source policyとlogical renderer intentだけをDomain projectorから保存 |
| Runtime Package | `backend_profile_ref`、Agility runtime artifact ref、signed Shader artifact refs、Root Signature／PSO manifest refs、D3D12MA license/SBOM refをexact closureへ含める |
| Application Package／Release | Runtime Package closureをTarget layoutへ写し、Agility DLL、main EXE export、shipping exclusion、package validation Receiptを所有する |

`RuntimePackageManifestV1`のWindows D3D12 bindingを次に固定する。

```text
D3d12RuntimePackageBindingV1 {
  target_ref: target.windows.editor | target.windows.desktop
  backend_profile_ref
  agility_package_ref          # Microsoft.Direct3D.D3D12 1.619.4 exact nupkg artifact
  agility_sdk_version: 619
  agility_core_dll_ref         # exact D3D12Core.dll bytes
  shader_artifact_refs[]
  root_signature_artifact_refs[]
  pipeline_manifest_ref
  d3d12ma_source_ref           # 3.2.0 exact source/license/SBOM
  qualification_policy_ref
}
```

BindingはTargetごとに一件を作り、exact Target setは`target.windows.editor; target.windows.desktop`である。両BindingのBackend／Agility／Shader closureはsame Candidateへ閉じるが、Editor package／process／surface fixtureとDesktop GameHost package／process／surface fixture、Qualification Receiptを別々に発行する。片側Receipt、同じbinary hash、WARP成功だけで反対Targetをqualifiedにしない。

`ApplicationPackageAssemblyManifestV1`は次を検証する。

1. Main EXEが`D3D12SDKVersion=619`と、末尾`\`を持つ相対subdirectory `D3D12SDKPath=.\D3D12\`をexportする。
2. `D3D12Core.dll`はEXE直下でなく宣言した`D3D12\`配下にあり、Runtime Packageのexact artifact hashと一致する。
3. Development packageだけが同一SDKVersionの`D3D12SDKLayers.dll`を含められる。Shipping Configurationのcandidate Packageでは存在をhard errorにする。
4. Header include orderはAgility NuGet includeをWindows SDK includeより前にし、link libraryはWindows SDKの`d3d12.lib`を使う。
5. Package read-backはEXE export、DLL path、file hash、signer、shipping exclusionを検査し、`PackageValidationReceiptV1`へ記録する。

本節のShipping Configurationはpackage layoutのnegative／read-back検証用candidateであり、CX0／CX1のRelease Activation、配布、Product labelを許可しない。公開Releaseへ使えるのはProduct PlanのCX3 Shipping Gateと全Target Release Receiptが成立した後だけである。

Agility SDK preview、別SDKVersionのLayers、EXE直下DLL、System32 DLLへの暗黙fallback、Package外shader／PSOはすべて不合格である。

## 33. Stable ID migration

- `windows_editor_v1`は`target.windows.editor`、`windows_desktop_v1`は`target.windows.desktop`へ置換し、各profile version 1をTarget Profile Registryへ置く。
- `d3d12_warp_conformance_v1`は`fixture.rendering.d3d12-warp-conformance`へ置換する。
- 旧`MIRAKAN-D3D12-*` 19件は§20.2の同condition dotted diagnosticへ置換する。
- Backend logical IDは`profile.rendering.d3d12`、versionは`D3d12BackendProfileV1.profile_version`へ置く。

MigrationはControl Plane Implementation Plan Appendix DとD3D12 Implementation Plan Appendix Aで一回だけ実行し、alias、dual emission、旧parserを残さない。

## 34. Implementation authority

Exact file、C++ interface、test、fixture、command、expected result、Package validation、PSO closure、alias negative case、Qualification runは[D3D12 Backend Implementation Plan](2026-07-22-d3d12-backend-implementation-plan.md)を正本とする。本設計と実装計画が矛盾した場合は実装を停止し、設計をreview stateへ戻して同じChangeSetで解消する。
