# Miraikanai Engine Rendering／Render Graphアーキテクチャ規約

- 文書版: 1.1
- 作成日: 2026-07-19
- 対象: 2D／3D Rendering、Render Snapshot、Render Graph、GPU resource、D3D12／Vulkan／Metal Adapter
- 状態: プロジェクト公式の規範設計レビュー版
- 機能範囲: [Miraikanai Engine 2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)
- 基盤規約: [Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)
- Runtime規約: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- Asset規約: [Miraikanai Engine Asset Pipeline／Content Package規約](./2026-07-19-asset-pipeline-content-packaging-design.md)
- UI規約: [Miraikanai Engine UI／Text／Localization／Accessibility規約](./2026-07-19-ui-text-localization-accessibility-design.md)
- Editor UI Framework規約: [Miraikanai Engine 独自Editor UI Framework／Shellアーキテクチャ規約](./2026-07-20-editor-ui-framework-architecture-design.md)
- Windows規約: [Miraikanai Engine Windows Platform／Distribution規約](./2026-07-19-windows-platform-distribution-design.md)
- Mobile規約: [Miraikanai Engine モバイルPlatformアーキテクチャ規約](./2026-07-19-mobile-platform-architecture-design.md)

## 1. 結論

Miraikanai EngineのRendererは、Worldを直接読む描画APIでも、AIがGPU commandを組み立てる仕組みでもない。Simulationがpublishしたimmutable `RenderSnapshot`、versioned Asset、Engine-owned Material／Visual Styleから描画packetを抽出し、宣言的`RenderGraphDefinition`をcompileしてD3D12、Vulkan、Metalへ変換する独自Subsystemである。

Rendererが正規に所有するものは次である。

- Platform非依存のRender resource／pass／access model
- 2D、3D、UI、VFX、Environmentのcomposition順
- visibility、LOD、batch、pipeline key
- resource lifetime、alias、barrier、queue dependency
- GPU handle generation、submission serial、deferred destruction
- Target Capability検証、device loss、fallback、telemetry

Backend API、allocator、shader compilerはAdapterまたはToolとして利用するが、Project、AI、GameplayDefinition、NativeGameModuleへnative objectを公開しない。

## 2. 決定権と対象外

| 主題 | 正本 |
|---|---|
| RenderSnapshot、Render Graph、resource／pass contract、Backend mapping | 本書 |
| Light、Shadow、Atmosphere、Fog、Cloud、VFX、Material、Visual Styleの機能と品質 | 2D／3D機能計画 |
| GPU memory、handle、submission retire、frame phase、budget | 基盤／Runtime規約 |
| Android Vulkan、Apple Metal minimum、surface lifecycle、thermal | Mobile規約 |
| Asset import、Shader／Texture／Mesh cook、streaming | Asset規約 |

C1／C2ではray tracing、path tracing、mesh shader必須化、GPU-driven occlusionによるauthoritative gameplay、Runtime shader source compile、AIによる任意Render pass／HLSL生成を行わない。Ray tracing、large-world virtual geometry、neural renderingはC3 Research Capabilityであり、MCDへ公開しない。

## 3. Module境界

```text
RenderSnapshot
  -> Render Extract
  -> Visibility／LOD／Packet Build
  -> Render Graph Instance
  -> Graph Validation／Compile
  -> Backend Command Plan
  -> D3D12 | Vulkan | Metal Adapter
  -> Submission／Present／Retire
```

| Module | 所有 |
|---|---|
| `rendering_contracts` | Platform非依存型、enum、handle、error |
| `rendering_core` | extract、visibility、LOD、packet、view |
| `render_graph` | resource／pass DAG、lifetime、alias、barrier plan |
| `rendering_materials` | Material IR、Shading Model、parameter layout、pipeline key |
| `rendering_visual_styles` | Cooked Style Manifest、layer composition |
| Backend Adapter | native device、queue、resource、descriptor、PSO、present |

Backend AdapterはWorld、Authoring Model、GameplayDefinitionへlinkしない。`rendering_core`はPhysics／Navigation／Audioを呼ばず、`RenderSnapshot`とAsset leaseだけを読む。

## 4. RenderSnapshotとView

### 4.1 Snapshot header

```text
RenderSnapshot
  schema_version
  snapshot_id
  tick_id
  project_revision
  world_generation
  asset_generation_id
  view_family[]
  renderable_2d[]
  renderable_3d[]
  light[]
  environment
  vfx_batch[]
  ui_snapshot
  debug_batch
```

Snapshotは`T110`で全体を一度だけpublishし、その後immutableである。Entity pointer、Component span、native Physics／GPU objectを含めない。ArrayはStable rendering keyでcanonical sortし、worker completion順を保存しない。

### 4.2 `RenderView`

| Field | 型／規則 |
|---|---|
| `view_id` | frame内一意の`uint32` |
| `purpose` | `game \| editor \| shadow \| reflection \| thumbnail` |
| `projection` | perspective／orthographicとfinite parameter |
| `view_transform` | right-handed、meter、finite |
| `viewport_px`／`scissor_px` | surface範囲内 |
| `render_scale` | Target Profile範囲 |
| `layer_mask` | registered 64-bit layer |
| `visibility_origin` | camera-relative origin、C1ではWorld原点と同一 |
| `history_key` | temporal historyのStable key |
| `quality_profile_id` | Cooked Profile |

Game Viewは一つの`ViewFamily`を基準とし、split screen、stereo／XRはC3までMCDで拒否する。Editor Scene View、thumbnailはGame Viewとresource budgetを別telemetryへ記録し、Shippingへ含めない。

### 4.3 Renderable packet

RenderableはAsset version、transform、bounds、material instance、layer、visibility flags、object Stable rendering IDを持つ。Mesh／Sprite vertex、Material graph、text sourceをSnapshotへcopyしない。

`RenderObjectKey`は`{ renderable_type_id, pipeline_key, material_key, geometry_key, stable_render_id }`のcanonical tupleとする。TransparentはStyleが定めるdepth／priorityとStable IDでsortし、pointer、submission orderをtie-breakに使わない。

## 5. Render resource model

### 5.1 Resource descriptor

`RenderResourceDesc`を次で固定する。

| Field | 規則 |
|---|---|
| `resource_id` | Graph instance内`uint32` |
| `kind` | `texture \| buffer \| acceleration_data_reserved` |
| `ownership` | `transient \| imported \| persistent \| history \| external_surface` |
| `format` | Engine `PixelFormat` closed enum |
| `extent` | absolute、view-relative、またはnamed resource-relative |
| `mip_count`／`array_layers`／`sample_count` | Target Capability内 |
| `usage_flags` | attachment、sampled、storage、copy、indirect等 |
| `memory_class` | device local、upload、readback |
| `initial_access` | imported／persistentだけ必須 |
| `clear_value` | attachment clear時だけ、typed |
| `alias_group` | compilerが生成。Authoring入力不可 |
| `debug_name_id` | shipping identityに使わない |

Texture extent、mip、format、usageの組合せをTargetごとに検証する。0 size、overflow、unknown format、usage不整合、readback attachmentを拒否する。

### 5.2 Handleとnative object

Game／Engine上位層は`RenderResourceHandle { index32, generation32 }`、`PipelineHandle`、`MaterialHandle`だけを持つ。D3D12 resource、VkImage、MTLTexture等はBackend registryが単一所有する。handle lookupはgeneration、Backend device generation、Asset versionを検査する。

GPU objectの破棄は最後に使用した全queueの`GpuSubmissionSerial`完了後だけ行う。frame countだけを根拠に解放しない。Device recreate時はgenerationを増やし、旧handleを全てinvalidにする。

## 6. Render Graph

### 6.1 Pass descriptor

```text
RenderPassDescriptor
  pass_id
  pass_type
  queue_class
  view_scope
  accesses[]
  color_attachments[]
  depth_stencil_attachment
  execute_template_id
  parameter_block
  declared_cost
```

`pass_type`と`execute_template_id`はEngine Capability Catalogのclosed IDである。AI、GameplayDefinition、Project dataは任意GPU callback、shader binary、native barrierを指定できない。

`ResourceAccess`はresource／subresource、`read | write | read_write`、logical stage、logical usageを明示する。Passがdescriptorにないresourceへaccessした場合はDevelopment validation faultである。

### 6.2 Compile algorithm

`R30_CompileRenderGraph`は次を固定順で行う。

1. resource、pass、view、Target Capabilityをschema検証する。
2. writer一意性、read-before-write、import initial state、feedback loopを検証する。
3. 明示dependencyとresource hazardからpass DAGを構築する。
4. `pass_id`をtie-breakにcanonical topological sortする。
5. resourceごとのfirst／last useとqueue ownershipを計算する。
6. descriptor完全一致かつlifetime非重複のtransient resourceだけalias候補にする。
7. queue間signal／waitとBackend非依存barrier planを作る。
8. compatible attachment passのmerge候補をBackendへ提示する。
9. transient heap、descriptor、command-list上限を予約する。
10. compile keyとGraph diagnosticsを出力する。

cycle、uninitialized read、同一subresourceのunordered write、unsupported format、budget超過はgraph全体を拒否する。resourceを黙って別formatへ変換しない。

### 6.3 Feedback、history、external surface

- 同一Pass内feedbackは登録済みTemplateとTarget featureが明示対応する場合だけ許可する。
- Temporal historyは`history_key`、format、extent、algorithm version、camera cut、surface generationを持つ。
- camera cut、extent／format／quality変更、device recreate、180 frame未使用でhistoryをdiscardする。
- Swapchain／CAMetalLayer drawableは`external_surface`としてimportし、Graphが所有・保持しない。
- Surface generationが変わったFrameのrecord／submitを拒否し、新generationでGraphを再compileする。

## 7. Frame lifecycle

Runtime規約の8 phaseを唯一の順序とする。

| Phase | Renderer処理 |
|---|---|
| `R00` | publish済みSnapshotをlease。途中構築Snapshotは読まない |
| `R10` | 必要Asset generation全体をatomic promotion |
| `R20` | View、visibility、LOD、packet、light clusterを構築 |
| `R30` | Render Graphをcompile、resource／descriptor予約 |
| `R40` | workerごとに事前割当command allocator／bufferへrecord |
| `R50` | queue dependency順でsubmit、serialを発行 |
| `R60` | surface generationを再検査してpresent |
| `R70` | completion取得、deferred destruction、pool再利用 |

Record workerはGraph plan外のbarrier、resource create／destroy、Asset loadを行わない。Backend callback内でWorld／AI／Asset cookerを呼ばない。

## 8. Backend contract

### 8.1 共通Port

`GraphicsDevicePort`は次の機能だけを公開する。

- Adapter／device／surface Capability query
- resource／view／sampler／pipeline作成
- descriptor／argument binding
- command allocator／buffer取得
- logical barrier planのencode
- queue submit、serial、completion
- present、resize、device fault report
- budget／residency telemetry

native型はPort header、MCD、Asset metadata、Project C++へ出さない。

### 8.2 D3D12

- Agility SDK 1.619.4と固定DXCを使用する。
- Enhanced Barriers supportを起動時検査し、対応時はlogical barrierをEnhanced Barrierへ写像する。非対応Reference pathはlegacy resource barrier conformanceを別fixtureで保持する。
- Render Pass APIはsupport queryとattachment semanticsが適合する場合だけBackend optimizationとして使い、Graph意味を変更しない。
- D3D12MAはheap／resource allocationだけに使い、residency、budget、lifetime policyはEngineが所有する。
- DRED、device removed reason、submission breadcrumbsをDeviceFaultReportへcopyする。

### 8.3 Vulkan

- AndroidはMobile規約のVulkan 1.1＋Android Vulkan Profile 2022を最低線とする。
- Synchronization、layout、queue ownershipはGraph barrier planから生成し、implicit orderへ依存しない。
- VMAはallocation primitiveとして隔離する。
- Dynamic Rendering等、minimum profile外の機能はCapability bitを持ち、未対応Deviceで自動使用しない。
- Validation Layer fixtureをDevelopment／CIで実行し、Shipping packageへlayerを含めない。

### 8.4 Metal

- Apple family 5／A12 baselineをMobile規約どおり適用する。
- `MTLHeap` resourceはGraph lifetimeとalias planに従い、heap resourceのhazard tracking特性を暗黙仮定しない。
- pass間hazardはfence／event／encoder boundaryへ明示写像する。
- drawableをframe外へ保持せず、background／surface loss時はsubmit／presentを停止する。
- Metal feature tableをTarget Capability lockへ固定する。

## 9. 2D／3D／UI composition

C1の共通composition順を次で固定する。未使用Passは除去できるが、順序の意味を変えない。

```text
Shadow／Depth preparation
-> 3D Opaque／Lighting
-> Environment／Fog／Cloud
-> 3D Transparent／World VFX
-> 2D World Layer
-> World Post Process
-> Pixel-locked Layer
-> Game UI／Text
-> Accessibility／Debug Overlay
-> Final Composite
```

- `pixel_2d`とpixel-locked layerはPoint sampling、integer scale、temporal history除外をStyle contractとして維持する。
- Toon、Realistic、`pixel_diorama`はMaterial Shading Modelとcomposition profileを切り替え、Renderer coreのnative APIを分岐公開しない。
- UI／Textはdisplay resolutionでrenderし、World dynamic resolutionの対象外とする。
- Editor overlayはShipping Snapshotへ含めない。
- GPU VFX、occlusion、exposure結果をauthoritative gameplayへ戻さない。

Editor WindowではScene／Game Viewのversioned texture composite後に`MiraUiDrawPacketV1`をdisplay resolutionで描画する。MiraUIはD3D12 command list、descriptor、GPU addressを所有せず、本書のRendering Portとsubmission lifetimeへ従う。Editor UI Packet、primitive、clip、atlas、surface generationの正本はEditor UI Framework規約に置く。

## 10. Material、Shader、Pipeline

Material IR、Shading Model、Visual Styleの機能詳細は2D／3D機能計画を正本とする。本書ではRuntime契約を次に固定する。

- Shader sourceはEngine／承認済みProject sourceからoffline compileする。
- Cooked Shader Packageはtarget、entry ID、shader model、interface hash、binary hash、compiler hashを持つ。
- `ShaderInterface`はbinding、constant layout、vertex input、attachment expectationをMCDから生成する。
- Material InstanceはDefinitionのcompile featureを増やさず、parameter blockとAsset referenceだけを変える。
- Pipeline keyはShader Package、render state、vertex layout、attachment format、sample count、Target featureのcanonical hashである。
- pipeline cache missはrender threadでshader compileせず、事前Cookまたはbackground PSO creationを使う。
- Shipping packageに未承認shader source、runtime compiler、fallback debug shaderを含めない。

Interface hash不一致、missing pipeline、invalid Materialは該当Asset generationのpromotionを拒否する。すでにRunning中のunexpected failureでは明示的なerror materialをDevelopmentだけに表示し、ShippingではScene loadを失敗させる。

## 11. Memory、Descriptor、Streaming

Windows ReferenceのGPU 5.5 GiB内訳とRendering CPU 256 MiBは基盤規約、Frame transientとqueueはRuntime規約を正本とする。

- transient texture／bufferはGraph lifetimeからaliasし、resource名やPass adjacencyだけでaliasしない。
- uploadはbounded ringまたはstaging allocationをsubmission serialでretireする。
- readbackは明示Requestと上限を持ち、frameごとの無条件readbackを禁止する。
- descriptor／argument tableはframe generationとsubmission lifetimeを持つ。
- Texture／Mesh streamingはAsset generationを跨ぐ混在をせず、resident mip／LOD変更だけをversion内のStreaming stateとして扱う。
- emergency reserveを通常PSO、texture、render targetへ使用しない。

OOM前はTarget Profileの順序どおりtexture mip、streaming距離、shadow、transient resolutionを下げる。Style-critical Point／integer scale、Toon ramp等を別Styleへ変えるfallbackは禁止する。必須resource予約に再失敗したFrameはsubmitせず、明示Renderer faultへ遷移する。

## 12. Device lossとRecovery

```text
Ready -> FaultDetected -> SubmissionStopped -> DiagnosticsCaptured
-> NativeObjectsDestroyed -> DeviceRecreated -> AssetsRehydrating
-> FirstValidGraph -> Ready
```

- Device fault検出後は新規submitとAsset promotionを停止する。
- 完了不明GPU workを待ち続けず、Backend fault policyでdeviceを破棄する。
- Snapshot、Project、CPU-side Asset catalogは保持するがnative handleを全invalid化する。
- critical Shader、UI、fallback resourcesから再構築し、その後visible Assetをpriority順にrehydrateする。
- 同一sessionで2回目のdevice loss、または60秒以内にFirstValidGraphへ戻れない場合はGameHostを正常終了扱いに偽装せずfault終了する。
- Editorは別Processのため、GameHost crash／device loss後もProject編集を維持する。

Mobile background／surface lossはdevice lossと区別し、Mobile規約のsurface generation／lifecycleに従う。

## 13. AI／Editor操作

AIと人間へ公開するのは次だけである。

- Camera、Light、Environment、Fog、Cloud、Post、VFX、Material、Visual Styleのtyped Authoring object
- Target／Quality Profile
- 登録済みRender featureとPass Template
- cost予測、thumbnail、debug view、GPU capture参照、validation

AIはresource barrier、heap offset、descriptor index、native format、queue signal値、shader binaryを指定しない。Custom HLSLはC2のR3 Native／Shader Source ChangeSetであり、隔離compile、interface validation、instruction／resource budget、全Target test、人間承認を必須とする。

EditorのRender Graph viewerは論理Pass、resource lifetime、alias、barrier、queue、costを表示するが、表示上のdragでShipping graphを任意改変しない。登録済みGraph Templateのparameter変更だけをChangeSetへ変換する。

## 14. PerformanceとTelemetry

Runtime規約のGPU P95 14.00 ms、hard 16.67 msとPass別soft capを維持する。最低telemetryは次である。

- CPU extract、cull、LOD、Graph compile、record、submit
- Pass別GPU timestampとpipeline statistics
- visible／culled／draw／dispatch／triangle／sprite／glyph count
- transient peak、alias saving、GPU budget／usage／eviction
- descriptor、PSO cache hit、shader variant
- upload／readback bytes、streaming miss、promotion wait
- barrier、queue wait、async overlap
- history invalidation、surface generation、device fault

10分soakでCPU／GPU deadline、memory、descriptor、submission serial backlogを検証する。GPU timestamp未対応／不安定DeviceではAvailabilityを記録し、0 msとして合格させない。

## 15. TestとRelease Gate

- Graph cycle、read-before-write、unordered write、subresource overlap、history invalidationのunit／property test
- 同一Graph入力からcanonical compile plan hash一致
- D3D12 Enhanced／legacy、Vulkan、Metalへのaccess／barrier conformance
- D3D12 Debug Layer／GPU validation、Vulkan Validation、Metal validationのzero-error fixture
- 2D pixel、Realistic、Toon、Pixel Dioramaのgolden imageと許容差
- RTX 3060、RX 6600、Android minimum／reference、A12／referenceのPerformance
- resize、alt-tab、HDR／SDR切替、surface loss、device removed fault injection
- GPU OOM、descriptor exhaustion、pipeline miss、corrupt shader、stale Asset generation
- resource last-use serial前に破棄されないlifetime test
- UI／text／pixel-locked layerがdynamic resolutionやTAAで劣化しないtest
- AIが未登録Pass、native resource、unsupported Target featureを生成できないconformance

C1 Rendererは、2D top-down sliceと3D compact arenaを全Targetの該当Qualityで描画し、Graph validation、device recovery、memory／frame budget、manual／AI authoringを満たした時点で完了する。

## 16. 一次資料

- [Direct3D 12 Programming Guide](https://learn.microsoft.com/en-us/windows/win32/direct3d12/directx-12-programming-guide)
- [D3D12 Enhanced Barriers](https://microsoft.github.io/DirectX-Specs/d3d/D3D12EnhancedBarriers.html)
- [D3D12 Render Passes](https://microsoft.github.io/DirectX-Specs/d3d/RenderPasses.html)
- [Managing Graphics Pipeline State in Direct3D 12](https://learn.microsoft.com/en-us/windows/win32/direct3d12/managing-graphics-pipeline-state-in-direct3d-12)
- [Vulkan Synchronization](https://docs.vulkan.org/spec/latest/chapters/synchronization.html)
- [Vulkan Render Pass／Dynamic Rendering](https://docs.vulkan.org/spec/latest/chapters/renderpass.html)
- [Metal Resource Synchronization](https://developer.apple.com/documentation/metal/resource-synchronization)
- [Metal Work Submission](https://developer.apple.com/documentation/metal/work-submission)
- [Metal Feature Set Tables](https://developer.apple.com/metal/Metal-Feature-Set-Tables.pdf)

これらのAPIは明示同期とresource lifetimeの根拠であり、MiraikanaiのRender Graph schema、Pass Template、composition、budgetを外部APIへ委ねるものではない。
