# Miraikanai Engine Rendering／Render Graphアーキテクチャ規約

- 文書版: 2.6
- 作成日: 2026-07-19
- 最終更新日: 2026-07-20
- 対象: 2D／3D Rendering、Render Snapshot、Render Graph、GPU resource、GPU-driven visibility、Temporal Reconstruction、Ray Tracing、Path Tracing、Neural Rendering、D3D12／Vulkan／Metal Adapter
- 状態: プロジェクト公式の規範設計レビュー版
- 機能範囲: [Miraikanai Engine 2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)
- 基盤規約: [Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)
- Runtime規約: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- Asset規約: [Miraikanai Engine Asset Pipeline／Content Package規約](./2026-07-19-asset-pipeline-content-packaging-design.md)
- Lighting規約: [Miraikanai Engine Lighting／AI Authoringアーキテクチャ規約](./2026-07-20-lighting-ai-authoring-architecture-design.md)
- Post Process規約: [Miraikanai Engine Post Process／AI Authoringアーキテクチャ規約](./2026-07-20-post-process-ai-authoring-architecture-design.md)
- LOD規約: [Miraikanai Engine AI可読LODアーキテクチャ規約](./2026-07-20-ai-readable-lod-architecture-design.md)
- Particle／VFX規約: [Miraikanai Engine 独自Particle／VFX Platformアーキテクチャ規約](./2026-07-20-particle-vfx-architecture-design.md)
- Water規約: [Miraikanai Engine Water Surface Platformアーキテクチャ規約](./2026-07-20-water-surface-platform-architecture-design.md)
- Weather／Snow規約: [Miraikanai Engine Weather／Snow Surfaceアーキテクチャ規約](./2026-07-20-weather-snow-surface-architecture-design.md)
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
- temporal input、reconstruction、frame generation、latency markerの共通契約
- anti-aliasのsample／resolve、spatial／temporal排他、ViewFamily単位Plan、history／layer分離
- raster、ray tracing、path tracing、neural renderingのCapabilityとfallback
- Target Capability検証、device loss、fallback、telemetry

Backend API、allocator、shader compiler、DLSS／XeSS／FSR／MetalFX等のSDKはAdapterまたはToolとして利用するが、Project、AI、GameplayDefinition、NativeGameModuleへnative objectを公開しない。Portable Rasterを全Targetの基準経路とし、高度機能は同じEngine-owned入力とRender Graphへ接続する交換可能Capabilityとして段階昇格する。

## 2. 決定権と対象外

| 主題 | 正本 |
|---|---|
| RenderSnapshot、Render Graph、resource／pass contract、Backend mapping | 本書 |
| Light Source、物理単位、Intent、Profile、Resolver、selection／clusterの論理契約 | Lighting／AI Authoring規約 |
| Shadowの意味、技法、品質、成熟度 | 2D／3D機能計画 |
| Material、Shader、Visual Style、AI Authoringの意味と品質 | Material／Visual Style／AI Authoring規約 |
| 非AA Post effectのIntent、Profile、Node Catalog、stage、history、Volume、品質 | Post Process／AI Authoring規約 |
| Anti-aliasのUser／AI向け意味Goal、Authoring Intent、製品成熟度 | 2D／3D機能計画 |
| `ResolvedAntiAliasingPlanV1`、sample／resolve、排他、history、実行Pass、Qualification | 本書 |
| Sky、Atmosphere、Fog、Cloud、Environment LightingのSource／AI Authoring／Compiler／品質 | Environment Platform規約 |
| VFX Asset／Graph／CPU・GPU simulation／VFX renderer binding／budget | Particle／VFX規約 |
| Water Body／Surface／Wave／Flow／Query／Underwater／water固有budget | Water規約 |
| Weather input／Snow Surface field／stamp／snow固有budget | Weather／Snow規約 |
| GPU memory、handle、submission retire、frame phase、budget | 基盤／Runtime規約 |
| Android Vulkan、Apple Metal minimum、surface lifecycle、thermal | Mobile規約 |
| Asset import、Shader／Texture／Mesh cook、streaming | Asset規約 |

C1はPortable RasterだけをProduction必須経路とする。C2はGPU-driven visibility、Temporal Reconstruction、Frame Generation、選択的Ray Traced Shadow／Reflectionを個別CapabilityとしてProduction昇格できる。C3はRTGI、Path Tracing、Ray Reconstruction、Neural Denoising／Radiance Cache／Shaderを実装対象に含めるが、機能ごとのQualification前にActive Capability CatalogやAI Toolへ掲載しない。

Mesh Shader、Work Graph、unbounded bindless、Ray Tracing、Neural Renderingを最低Targetの必須条件にしない。GPU visibility、generated frame、neural outputをauthoritative gameplayへ使用しない。Runtime shader／model compile、Runtime学習、AIによる任意Render pass／HLSL／model weight生成を禁止する。

### 2.1 単一Authoring SourceとTarget別実行Artifact

Miraikanai Engineは、軽量用と高品質用に別Material、別Scene、別Visual Style、別Gameplay Sourceを持つ方式を採用しない。人間とAIは一つのAuthoring Document集合、Material IR、Visual Style、`LodIntentV1`、Render Representation intentを編集し、Runtime CompilerがTarget ProfileとProduction Capability Manifestから次の実行Artifact closureを生成する。

| Artifact class | 対象 | 必須特性 |
|---|---|---|
| Portable Raster | Windows C1と全Targetの意味基準 | Native Raster、CPU direct／instancing、LOD0／CPU selector fallback、Vendor SDK不要 |
| Mobile Raster | Android／Apple | 同じMaterial／Style意味、tile／bandwidth／thermal向けlayout、offline shader／texture specialization |
| Advanced Raster | Qualified desktop | GPU indirect、HZB、HLOD、Hybrid Deferred等を個別Capabilityとして追加 |
| Optional Advanced Attachment | 対応Profileだけ | Temporal／FG、RT、Path、Neural、Vendor Provider。個別に除去可能 |

これらは別Authoring pipelineではなく、同じSourceから生成されるTarget別Derived Artifactである。Artifact間のbinary layout一致は要求しないが、Material parameter、Style lock、critical cue、Gameplay visibilityの意味一致を要求する。Source変更時は依存closureを再Cookし、TargetごとのArtifact／Interface hash、fallback、budget、Qualification Receiptを更新する。

Packageは選択Targetに必要なArtifact closureだけを含める。起動時のfeature probeは事前Cook済み候補から選択するだけで、Runtime shader compile、Source変換、未承認Provider downloadを行わない。必須Artifactがない、Interface hashが一致しない、fallbackが未Cookの場合は上位機能だけを黙って外さず、Package／Scene loadをtyped errorで拒否する。Projectが明示的に許可した下位Quality Profileがあり、そのProfileも独立Qualification済みの場合だけ再選択できる。

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
| `rendering_visibility` | CPU／GPU culling、HZB、LOD／HLOD、indirect packet、visibility history |
| `render_graph` | resource／pass DAG、lifetime、alias、barrier plan |
| `rendering_materials` | Material IR、Shading Model、parameter layout、pipeline key |
| `rendering_visual_styles` | Cooked Style Manifest、layer composition |
| `rendering_shadows` | Shadow Intent／Graphのresolved Plan、Technique Catalog、cache／page residency、debug semantics |
| `rendering_temporal` | motion／depth／exposure／mask、TAA／upscale、history、frame generation入力 |
| `rendering_raytracing` | acceleration input、ray query／pipeline、RT shadow／reflection／GI、path tracing |
| `rendering_neural` | model manifest、neural feature plan、denoise／reconstruction／radiance cache contract |
| `rendering_provider_adapters` | DirectSR、DLSS／Streamline、XeSS、FSR、MetalFX等の隔離Adapter |
| Backend Adapter | native device、queue、resource、descriptor、PSO、present |

Backend AdapterとProvider AdapterはWorld、Authoring Model、GameplayDefinitionへlinkしない。`rendering_core`はPhysics／Navigation／Audioを呼ばず、`RenderSnapshot`とAsset leaseだけを読む。Provider AdapterはEngine-owned frame inputとBackend Portだけを受け、World、Material Graph、Swapchain policy、fallback順を所有しない。

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
  light_snapshot: LightSnapshotV1
  environment
  water_batch[]
  weather_presentation
  snow_surface_batch[]
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

Game Viewは一つの`ViewFamily`を基準とする。2D split viewは2D／3D機能計画5.6節のC2 Capabilityとして同一ViewFamily内のbounded Viewへ解決し、3D split view、stereo／XRはC3までMCDで拒否する。Editor Scene View、thumbnailはGame Viewとresource budgetを別telemetryへ記録し、Shippingへ含めない。

`ViewLodContextV1`はprojection、render extent、FOV／orthographic span、view transform、camera cut、Quality ProfileからLOD規約の`projected_error_px_q16`と`projected_coverage_px_q16`を生成する。distanceだけをMesh LOD thresholdとして保存せず、CPU／GPUが同じ量子化、境界包含、enter／exit比較を使用する。Editor／shadow／reflection Viewはそれぞれ独立したselection stateを持ち、Game Viewの履歴またはtierを再利用しない。

### 4.3 Renderable packet

RenderableはAsset version、transform、bounds、material instance、layer、visibility flags、object Stable rendering IDを持つ。Mesh／Sprite vertex、Material graph、text sourceをSnapshotへcopyしない。

`RenderObjectKey`は`{ renderable_type_id, pipeline_key, material_key, geometry_key, stable_render_id }`のcanonical tupleとする。TransparentはStyleが定めるdepth／priorityとStable IDでsortし、pointer、submission orderをtie-breakに使わない。

### 4.4 `RenderRepresentationPlanV1`

大量配置を個別drawのまま受理してBackend任せにしない。Runtime CompilerはRuntime規約の`RuntimeRepresentationPlanV1`からTarget別`RenderRepresentationPlanV1`をCookし、各Source集合を次のいずれかへ明示分類する。

| Representation | 対象 | C1経路 | C2経路 |
|---|---|---|---|
| `individual` | 個別Material／animation／interactionを持つ重要object | stable packet sort＋direct draw | GPU visibility＋indirect |
| `instanced` | 同一geometry／material interfaceで個別transformだけが主に異なる集合 | CPU cull＋instance buffer＋direct instanced draw | GPU cull＋compaction＋indirect |
| `spatial_batch` | cell内のstatic decorative集合 | offline batch、cell単位cull | HLOD／streaming＋GPU visibility |
| `presentation_batch` | Sprite、glyph、VFX等の非authoritative表示 | domain packet batch | indirect／domain固有GPU path |

PlanはLOD規約のTarget別`LodResolutionPlanV1`を参照し、Source StableId集合、spatial cell、mobility、interaction、geometry／material key、Mesh／Representation／Material／VFX／Surface LOD plan、HLOD chain、bounds、maximum resident／visible instance、Target fallback、visual equivalence hashを持つ。Domainごとのtierを一つの共通`lod_index`へ畳み込まない。

- static／decorative objectはCook時にinstance／spatial batch候補へする。総配置数が多いことだけでSourceを拒否しない。
- mutable Physics、個別Damage、interaction、Save対象をstatic batchへ混入させない。描画をinstance化してもauthoritative Entity identityは失わない。
- HLODはstatic `decorative_instance`だけを対象とし、proxy／impostor／occlusion geometryをGameplay Collision、Navigation、Damage、AI perceptionへ公開しない。
- Material parameter差で無制限にbatchを分割せず、instance parameter blockまたは承認済みMaterial variantへ解決する。見た目を統合できない場合はindividualを維持する。
- C1 `cpu_direct_v1`もinstancing、spatial culling、LODを必須とし、C2 GPU pathがないと大量Sceneが成立しない設計にしない。
- AIは「大量」をdraw削減だけで解決済みと表示せず、CPU extract、visible instance、draw／dispatch、triangle／pixel、GPU memory、overdrawを同じfixtureで測定する。

### 4.5 2D packetとExecution Plan

2D製品機能、`Renderer2DExecutionPlanV1`、Sprite／Tilemap Profile、公式Backend対応、fallback、fixtureは2D／3D機能計画5.0～5.7節を正本とする。Rendererは`SpriteRendererComponentV1`または`TileChunkArtifactV1`から、Asset version、bounds、layer／order／Y-sort、material instance、atlas page、mask／blend、Stable rendering IDを持つpacketを抽出する。Source rect配列、Tile ID配列、Texture handle、native descriptor indexをSnapshotへcopyしない。

Portable C1は`cpu_direct_v1`のdomain pathとしてCPU bounds cull、Sprite LOD、canonical sort、instance buffer、bounded resource table、direct instanced drawを実行する。TilemapはC1から専用chunk packet producerを持ち、occupied boundsでcullし、Cook済みdraw spanを同じCanvas batchへ送る。C2 `gpu_indirect_v1`はCPUと同じbounds、LOD量子化、batch key、Stable ID tie-breakからvisibility／draw listを生成し、CPU readback結果をGameplayへ返さない。

2D resource bindingはEngine正規値`bounded_table | indexed_table`とし、D3D12 descriptor table／direct heap、Vulkan descriptor set／indexing、Metal bounded binding／argument bufferへAdapterが変換する。Backend高速化は起動時feature query、Target Capability Manifest、Qualification Receiptの積でだけ選び、`indexed_table -> bounded_table`、`gpu_indirect -> cpu_instanced -> cpu_direct`の事前Cook済みfallbackを保持する。driver名またはVendor名だけで選択せず、fallback切替時もSprite／Tile／animation eventの意味を変えない。

## 5. Render resource model

### 5.1 Resource descriptor

`RenderResourceDesc`を次で固定する。

| Field | 規則 |
|---|---|
| `resource_id` | Graph instance内`uint32` |
| `kind` | `texture \| buffer \| acceleration_structure` |
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

`pass_type`と`execute_template_id`はCooked Capability Catalogのclosed IDである。Engine Templateに加え、隔離compile、interface／budget／Target validation、人間Review、Promotion Receiptに合格した`ProjectShadowTechniqueV1`だけがCook前にProject固有IDを登録できる。AI、GameplayDefinition、Project dataはRuntimeに任意GPU callback、shader binary、native barrierを追加できない。

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
| `R30` | Base、RT、reconstruction、UI、generationを含むRender Graphをcompile、resource／descriptor予約 |
| `R40` | workerごとに事前割当command allocator／bufferへrecord |
| `R50` | real frameをqueue dependency順でsubmitし、serialとProvider frame IDを発行 |
| `R60` | surface generationを再検査し、UI合成、任意generated frame、real frameをProvider present planどおり提示 |
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
- `ExecuteIndirect`、Mesh Shader、Sampler Feedback、VRS、Work Graph、Cooperative Vectorは個別feature queryとQualification Receiptがある場合だけ使う。
- DRED、device removed reason、submission breadcrumbsをDeviceFaultReportへcopyする。

### 8.3 Vulkan

- AndroidはMobile規約のVulkan 1.1＋Android Vulkan Profile 2022を最低線とする。
- Synchronization、layout、queue ownershipはGraph barrier planから生成し、implicit orderへ依存しない。
- VMAはallocation primitiveとして隔離する。
- Dynamic Rendering等、minimum profile外の機能はCapability bitを持ち、未対応Deviceで自動使用しない。
- descriptor indexing、indirect count、mesh shader、fragment shading rate、ray query／ray tracing pipelineは個別feature bitと実機Receiptを必要とする。
- Validation Layer fixtureをDevelopment／CIで実行し、Shipping packageへlayerを含めない。

### 8.4 Metal

- Apple family 5／A12 baselineをMobile規約どおり適用する。
- `MTLHeap` resourceはGraph lifetimeとalias planに従い、heap resourceのhazard tracking特性を暗黙仮定しない。
- pass間hazardはfence／event／encoder boundaryへ明示写像する。
- drawableをframe外へ保持せず、background／surface loss時はsubmit／presentを停止する。
- Metal feature tableをTarget Capability lockへ固定する。
- Indirect Command Buffer、Mesh Shading、MetalFX、Ray Tracing、Metal 4 Machine Learningは`supportsFamily`と個別API queryの両方を検査し、Marketing上のdevice世代名だけで有効化しない。

## 9. 2D／3D／UI composition

C1の共通composition順を次で固定する。未使用Passは除去できるが、順序の意味を変えない。

```text
Shadow／Depth preparation
-> 3D Opaque／Lighting／AO／Reflection # Post Process規約のP00／P05接続
-> Environment／Fog／Cloud
-> Water Depth／Surface／Underwater Composite  # C2。C1 Waterは次の3D Transparentへ統合
-> 3D Transparent／World VFX
-> 2D World Layer
-> World Post Process                 # ResolvedPostProcessPlanV1のP15～P70
-> Pixel-locked Layer
-> Game UI／Text
-> Accessibility／Debug Overlay       # P80／P90接続
-> Final Composite
```

- `pixel_2d`とpixel-locked layerはPoint sampling、integer scale、temporal history除外をStyle contractとして維持する。
- Toon、Realistic、`pixel_diorama`はMaterial Shading Modelとcomposition profileを切り替え、Renderer coreのnative APIを分岐公開しない。
- UI／Textはdisplay resolutionでrenderし、World dynamic resolutionの対象外とする。
- UI／Text／pixel-locked layerをWorld motion vector、Temporal Reconstruction、Frame Generation用HUD-less colorへ混ぜない。ProviderがUI textureを要求する場合はFinal Compositeから分離して渡す。
- Editor overlayはShipping Snapshotへ含めない。
- GPU VFX、occlusion、exposure結果をauthoritative gameplayへ戻さない。
- Water GPU波／SSR／foamとSnow GPU fieldをauthoritative gameplayへ戻さない。Water Surface QueryとGameplay Surface Stateは各SubsystemのCPU正規契約を使う。

Post ProcessのNode順、入力色空間、AA接続、Volume blend、history reset、UI／pixel-locked除外は[Post Process／AI Authoring規約](./2026-07-20-post-process-ai-authoring-architecture-design.md)を正本とする。Rendererは`ResolvedPostProcessPlanV1`をCatalog登録済みPass Templateへ展開し、ProfileやAIから任意Pass順を受け取らない。

Editor WindowではScene／Game Viewのversioned texture composite後に`MirakanUiDrawPacketV1`をdisplay resolutionで描画する。MirakanUiはD3D12 command list、descriptor、GPU addressを所有せず、本書のRendering Portとsubmission lifetimeへ従う。Editor UI Packet、primitive、clip、atlas、surface generationの正本はEditor UI Framework規約に置く。

## 10. Material、Shader、Pipeline

Material IR、Shading Model、Visual Style、Semantic Catalog、AI Operation、Preview／Explain／Validatorの機能詳細は[Material／Visual Style／AI Authoring規約](./2026-07-20-material-visual-style-ai-authoring-architecture-design.md)を正本とする。本書ではRuntime契約を次に固定する。

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
- `LightIntentV1`、`LightingStyleProfileV1`とEngineが生成した`ResolvedLightPlanV1`／`LightSnapshotV1`
- `PostProcessIntentV1`、`PostProcessProfileV1`とEngineが生成した`ResolvedPostProcessPlanV1`
- `ShadowIntentV1`、`ShadowStyleProfileV1`、`ShadowGraphV1`、承認済み`ProjectShadowTechniqueV1`
- `AntiAliasingIntentV1`とEngineが生成した`ResolvedAntiAliasingPlanV1`のread-only Preview
- Target／Quality Profile
- 登録済みRender featureとPass Template
- cost予測、thumbnail、debug view、GPU capture参照、validation

Lightの物理単位、Role、Intent、Profile、Resolver、selection／cluster、Operation、PreviewはLighting規約、非AA Post effectのIntent、Profile、Node、Volume、Operation、PreviewはPost Process規約、Environment／Fog／CloudはEnvironment Platform規約を正本とする。本書は各Subsystemの解決済みPlanを登録済みRender Graph TemplateとTarget resourceへ変換する境界だけを所有する。

AIはresource barrier、heap offset、descriptor index、native format、queue signal値、shader binaryを指定しない。Custom HLSLはC2のR3 Native／Shader Source ChangeSetであり、隔離compile、interface validation、instruction／resource budget、全Target test、人間承認を必須とする。

AAについてAIが指定できるのは意味Goal、許可方式、Scope、Target／Quality selector、fallbackだけである。sample location、resolve shader、jitter sequence、history weight、reactive mask生成、Provider native option、Pipeline keyを指定しない。`camera_profile` ScopeもCamera単独のnative surfaceを変更せず、所属ViewFamilyのPlan候補としてResolverへ渡す。

AIが大量配置を作る場合は、個別object数だけでなく`RenderRepresentationPlanV1`のindividual／instanced／spatial／presentation内訳、resident／visible peak、推定draw／triangle／pixel、memory、Target fallbackをPreviewへ表示する。instance化、batch、LOD、HLOD、streamingはGameplay identityやinteractionを変更しない範囲で自動提案できる。object削除、interaction削除、敵味方の可視性やGameplay結果を変える案はRenderer最適化として自動Commitしない。

EditorのRender Graph viewerは論理Pass、resource lifetime、alias、barrier、queue、costを表示するが、表示上のdragでShipping graphを任意改変しない。登録済みGraph Templateのparameter変更だけをChangeSetへ変換する。

### 13.1 Shadow拡張境界

Shadowの意味、Authoring Level、Node Catalog、Resolver、Quality Profile、Backend成熟度は2D／3D機能計画を正本とする。本書は`ResolvedShadowPlanV1`をRender Graphへ安全に展開する境界を所有する。

- L0／L1は登録済みGraph Templateのparameterだけを変更する。
- L2 `ShadowGraphV1`は型付きNodeをoffline compileし、closed Pass Templateとresource descriptorへ解決する。Graph nodeからnative command、shader entry、barrierを直接生成しない。
- L3 `ProjectShadowTechniqueV1`はRenderer全体の差替えではなく、`ShadowTechniquePortV1`の入力Semanticから`shadow_attenuation_linear`を生成するProject moduleである。
- Technique PassはCook前に固定ID、Shader Interface hash、resource／queue access、最大dispatch／draw、persistent／transient byte、fallback、debug channelをManifestへ記録する。
- TechniqueはShadow専用snapshot viewとversioned Assetだけを受け、World、Gameplay state、Entity pointer、Backend native objectを受けない。
- Runtime code／shader compile、未宣言resource access、Graph compile後のPass追加、Target別の秘密fallbackを禁止する。

Render Graph compilerはTechnique Manifestの宣言を通常Passと同じcycle、hazard、lifetime、alias、queue、memory validationへ通す。Manifest申告とshader reflection／実行時使用が一致しないArtifactはpromotionせず、Running中の不一致は`ShadowTechniqueValidationFailed`で該当Planを停止する。事前承認済みfallbackがあれば次FrameのGraph Instanceから切り替え、なければRenderer faultへ遷移する。

## 14. Capability成熟度と公式選択

Renderer Capabilityは実装有無とProduction保証を分離し、`Unavailable | Experimental | Qualified | Production`の状態をTarget、Quality、Backend、SDK generationごとに持つ。`Production`以外をProject既定、AI推奨、Shipping必須Capabilityにしない。

| Product段階 | 必須Capability | 任意Capability |
|---|---|---|
| C1 Portable Production | Forward+、2D CPU bounds／instanced batch、Tile chunk、CPU frustum／LOD、FXAA、`mirakan_taa_v1`、Forward+ MSAA 2x／4x、dynamic resolution、native present | Backend固有の同値最適化 |
| C2 Scalable Production | 2D GPU visibility／indirect／indexed binding／streaming Tilemap、3D GPU visibility、HZB、indirect、HLOD、Hybrid Deferred、SMAA 1x、Temporal Reconstruction Port | MSAA 8x、DLSS／XeSS／FSR／MetalFX、Frame Generation、RT Shadow／Reflection、Mesh Shader |
| C3 Advanced | RTGI、Path Tracing、Ray Reconstruction、Neural Denoising／Radiance Cache／Shaderの共通契約と個別Qualification | Work Graph、large-world virtual geometry、vendor research SDK |

起動時に`RendererCapabilitySignatureV1`を作る。最低fieldはBackend、API／shader version、GPU／driver identity、feature bit、memory budget、display mode、SDK／model generation、signed artifact hashである。`RendererProfileResolver`はProject要求、Target Profile、Capability Signature、Qualification Receiptから一つの`ResolvedRendererProfileV1`を決定し、次を保存する。

- raster path、visibility path、shadow／reflection／GI path
- temporal reconstruction provider、frame generation provider、latency provider
- render／display extent、Quality、memory／pass budget
- fallback chain、選択理由、拒否されたCapabilityと理由
- SDK、model、driver、Profile、benchmark baseline ID

Userが設定画面でProviderを変更した場合も同じResolverを通す。対応表示だけで未Qualification Providerを有効化せず、Benchmark結果だけでProjectの必須画風を変更しない。

## 15. GPU-driven visibilityとgeometry pipeline

### 15.1 正規段階

| Path | 内容 | 必須fallback |
|---|---|---|
| `cpu_direct_v1` | CPU frustum、`projected_error_px_q16` LOD、hysteresis、stable packet sort、instancing、direct draw | なし。C1基準 |
| `gpu_indirect_v1` | CPUと同じ量子化規則によるGPU frustum／LOD、前frame HZB occlusion、compaction、indirect draw | `cpu_direct_v1` |
| `gpu_meshlet_v1` | meshlet cull、indirect mesh drawまたはmesh shader | `gpu_indirect_v1` |
| `gpu_work_graph_v1` | Work Graph／同等機能によるdynamic work expansion | `gpu_indirect_v1` |

C2 Desktopでは`gpu_indirect_v1`を正式実装対象とする。Meshlet、Mesh Shader、Work GraphはFeatureが存在するだけで選ばず、同一fixtureでCPU／GPU frame、memory、quality、driver stabilityを比較して15.4節の昇格条件を満たした場合だけ使用する。

### 15.2 Visibility data

GPU inputは`VisibilityInstanceV1`のSoA bufferとし、geometry generation、current／previous transform、bounds、量子化済みlevel error／enter／exit threshold、previous presentation tier、material packet、layer、stable render IDだけを持つ。Entity、Component pointer、Gameplay tag、Simulation tier、authoritative relevanceを含めない。

HZBは当該Viewの直前に完了したreal frame depthから構築する。camera cut、surface generation、projection、render extent、occluder policy変更時にhistoryをinvalid化し、invalid frameはfrustumだけで描画する。occlusion結果は一frame以上のconservative hysteresisを持ち、near plane交差、巨大bounds、前frame未存在objectを強制visibleにする。

GPU outputはbounded visible index、Domain別presentation tier、indirect argument、overflow counterである。上限超過時はStable ID順に黙って欠落させず、事前Cook／Previewでは拒否し、Shippingの突発超過ではC1 CPU fallbackへ次frameから切り替えて`MIRAKAN-LOD-CAPACITY_EXCEEDED`を記録する。

### 15.3 Backend mapping

- D3D12は`ExecuteIndirect`、Vulkanは`vkCmdDrawIndexedIndirectCount`相当、MetalはIndirect Command Bufferへ写像する。
- descriptor indexing／dynamic resource heap非対応時はEngine-owned tableとpacket groupingを使う。
- meshletはAsset規約のportable cluster artifactを読み、Backend native commandをCooked Assetへ保存しない。
- Work GraphはRenderer Graphの代替ではない。Shader内work expansionだけを担当し、resource lifetime、queue、barrier、budgetはRender Graphが所有する。

### 15.4 最適化昇格

新経路は同じScene、input trace、Quality、output extent、warm stateで基準経路と比較し、frame P95が5%以上かつ0.20 ms以上改善し、GPU／CPU memory peak、allocation、shader variant、visual fixture、device faultが悪化しない場合だけ既定候補へ昇格する。改善が一GPUだけの場合はそのCapability Signature限定Profileとし、他Vendorへ一般化しない。

## 16. Anti-aliasing、Temporal Reconstruction、Frame Generation、Vendor SDK

### 16.0 Anti-aliasing PlanとResolver

`RendererProfileResolver`は`AntiAliasingIntentV1`を直接実行せず、ViewFamilyごとに一つの`ResolvedAntiAliasingPlanV1`へ解決する。同じViewFamilyのCameraはraster sample count、temporal Provider、jitter、render／display extentを共有する。Split ViewでCameraごとに異なる方式が必要な場合は別ViewFamily／別render targetとしてbudgetを再検証し、一つのsurfaceへ異なるMSAA sample countを混在させない。

| Field | 規則 |
|---|---|
| `source_intent_id`／`source_revision` | 解決元Intentを特定し、stale Planを拒否する |
| `view_family_id`／`scope_resolution` | Project／Camera Profileから最終ViewFamily Scopeへ解決した結果 |
| `raster_samples` | `1 \| 2 \| 4 \| 8`。2以上はForward+のQualified attachment／pipelineだけ |
| `spatial_method` | `off \| fxaa \| smaa_1x`。一つだけ |
| `temporal_method` | `off \| mirakan_taa_v1 \| mirakan_taau_v1 \| qualified_provider_id`。一つだけ |
| `render_extent`／`display_extent` | Target ProfileとProvider制限内。UI／pixel-locked extentを含めない |
| `jitter_policy` | Engine-owned closed ID。AI／Project dataはsample列を保持しない |
| `history_reset_mask` | camera cut、teleport、extent、surface、projection、Provider、model、AA方式変更のclosed bit |
| `excluded_layers` | `ui \| text \| pixel_locked`を必須集合とし、World historyへ混入させない |
| `required_capabilities` | Backend、renderer、sample count、motion／depth／mask、Provider、HDRのCapability ID |
| `predicted_cost` | pass GPU、bandwidth、persistent／transient byte、Pipeline variant数 |
| `fallback_chain` | 順序、意味差、User通知、必要な再構築境界 |
| `decision_trace` | 選択理由、却下method、Constraint／Receipt／baseline ID |

方式互換は次のclosed tableを正本とする。`Qualified`はTarget／Backend／driver／QualityごとのReceiptを意味し、API feature bitだけでは利用可能にしない。

| Method | 成熟度 | Renderer／入力 | 主用途と制限 |
|---|---|---|---|
| `none` | C1 diagnostic | 全Raster | bit-exact検査、AA対象外layer、User明示指定だけ。AI自動最適化は禁止 |
| `fxaa` | C1 Portable | 全Raster、resolved color | 最低costのspatial fallback。Tone map後、UI合成前に実行 |
| `mirakan_taa_v1` | C1 Portable 3D | motion、depth、exposure、jitter、history | native extentのtemporal AA。Pixel／UI／VR low-latency Intentでは候補外 |
| `msaa_2x`／`msaa_4x` | C1 Forward+ | sample可能なcolor／depth、全対象PSOのsample一致 | Geometry edgeとalpha-to-coverage。Shader／specular aliasを単独では解決しない |
| `smaa_1x` | C2 Portable | resolved color | Temporal禁止かつFXAAより鮮明さを優先するspatial候補 |
| `msaa_8x` | C2 optional | Forward+、High／offline実機Gate | 自動選択禁止。memory／bandwidth／tile storeを個別測定 |
| `mirakan_taau_v1` | C2 | `TemporalFrameInputV1` | Engine基準temporal upscale |
| Qualified DLSS／XeSS／FSR／MetalFX | C2 optional | Provider別Input／署名／license／driver | Providerごとに独立Qualification |

正規組合せは次とする。

- `raster_samples > 1`と`temporal_method != off`を同時に使用しない。Unity URPの排他とForward／Deferred差を含む複数BackendのPortable基準として固定する。
- MSAAとFXAA／SMAAの併用は既定で禁止し、Target限定Qualificationが単独方式よりAA fixtureを改善しbudget内である場合だけProfileへ登録する。
- Hybrid DeferredのGBufferをMSAA化しない。MSAAはForward+ ViewFamilyだけに許可し、Deferred／Hybrid ViewFamilyはspatialまたはtemporal方式を使う。
- MSAA 2x／4xはC1、8xはC2 optionalとする。8xをLow／Medium／MobileのAuto候補にしない。
- Alpha-to-coverageはMSAA有効時のmasked Material Capabilityであり、transparent blend、texture、specular aliasが改善するとは表示しない。
- TAA／TAAU／temporal Providerはpre-tonemap scene-linear HDRで実行し、FXAA／SMAAはTone map後かつUI／text／pixel-locked final composite前に実行する。
- Dynamic resolution変更、Camera cut、teleport、projection／surface／AA方式／Provider変更ではhistoryを全破棄する。MSAA sample count変更はRender Graph attachmentとPipeline keyをSettings Apply／Loading境界で再構築する。

Resolverは`UnsupportedByRenderer | UnsupportedByTarget | InvalidCombination | MissingMotionVectors | MissingTemporalInput | PixelLockedTemporalForbidden | MsaaSampleUnsupported | ProviderNotQualified | BudgetExceeded | ScopeConflict | RebuildBoundaryRequired`を`AntiAliasingResolutionErrorV1`のclosed codeとして返す。未知方式を近い名前へ補正せず、利用可能候補、拒否理由、必要Capability／Receipt、意味差を`RemediationV1`へ含める。

### 16.1 Engine-owned frame contract

`TemporalFrameInputV1`を全Providerの唯一の入力契約とする。

| Field | 規則 |
|---|---|
| `frame_id`／`present_id` | real frameごとに単調増加。generated frameへ新しいsimulation IDを割り当てない |
| `color` | pre-tonemap scene-linear HDR、render extent |
| `depth` | Engine reversed-Z metadata付き。AdapterがProvider形式へ変換 |
| `motion` | current pixelからprevious real frame pixelへの`RG16_FLOAT`、jitterを含めない |
| `exposure`／`pre_exposure` | finite、同じreal frameのTone Map契約 |
| `reactive_mask` | transparent、particle、emissive変化を`[0,1]`で表す |
| `composition_mask` | RT、procedural、履歴保護を解除すべき領域 |
| `hudless_color`／`ui_color` | Frame Generationが要求する場合だけdisplay extentで提供 |
| `jitter` | render pixel単位、`(-0.5,0.5)` |
| `render_extent`／`display_extent` | 0禁止、Provider制限内 |
| `history_reset` | camera cut、teleport、extent／Provider／surface／model変更時にtrue |

Opaque、masked、skinned、deformed、vertex animated objectはmotion vectorを書く。Transparent／Particleは可能ならmotion、常に適切なreactive maskを書く。UI、text、pixel-locked layerはWorld motion／historyへ含めない。

### 16.2 Provider Catalog

| Provider | 初期lock／用途 | Platform |
|---|---|---|
| `mirakan_taau_v1` | Engine基準TAA／temporal upscale | 全Target |
| `directsr_v1` | DirectSR経由SR。Frame Generation／Ray Reconstructionを所有しない | Windows D3D12 |
| `streamline_2_11_1` | DLSS SR／RR／FG、Reflex。Production署名binaryだけShipping可 | Windows D3D12。Vulkanは別Gate |
| `xess_3_0_1` | XeSS SR／FG、XeLL | Windows D3D12 |
| `fsr_sdk_2_3_0` | FSR Upscaling 4.1.1、FG 4.0.1、Ray Regeneration 1.1.0。Technique別に昇格 | Windows D3D12。VulkanはTechnique別Gate |
| `metalfx_v1` | Temporal／Spatial、Frame Interpolation、Denoised Upscaling | Apple Metal |

上表はReview済み初期versionである。実装開始時に公式配布元から取得し、source／binary SHA-256、署名、license、Notice、runtime依存、driver条件を`RendererProviderLockV1`へ固定する。配布物が取得不能、license不承認、署名不一致なら当該ProviderをBuildへ含めず、versionを推測しない。

ShippingでProvider／modelの無承認OTA更新を禁止する。更新は旧／新versionの同一fixture、bridge baseline、visual／latency／memory／fault比較、SBOM、署名、人間承認を必要とする。

### 16.3 Selectionと排他

- 一つのViewFamilyで有効なSuper Resolution ProviderとFrame Generation Providerは各一つだけである。
- DLSS familyを使う場合はStreamline、XeSS FGはXeLL、FSR familyはFSR SDK、AppleはMetalFXの同世代統合を既定とし、異なるVendorのFrame Generation／Latency wrapperを混在させない。
- DirectSRはSRだけを必要とし、同じframeでDLSS／XeSS／FSR direct Adapterを使用しない場合に選ぶ。
- Provider切替はLoading／Settings Apply境界でpresentをdrainし、旧Context、proxy swapchain、historyを完全破棄してsurface generationを増やす。
- AA方式、MSAA sample count、jitter policyの切替もLoading／Settings Apply境界で行う。変更対象ViewFamilyのpresentをdrainし、history／resolved attachment／関連Pipelineを世代更新する。
- Pause、全画面Menu、Loading、camera cut直後、window resize中、base real frameが60 fps hard gateを満たさない場合はFrame Generationを停止する。

### 16.4 Frame Generationの評価

generated frameをSimulation rate、real render FPS、CPU／GPU base frame P95へ加算しない。`real_fps`、`displayed_fps`、`generated_ratio`、`present_latency`を別metricにする。Frame Generationはbase real frameのCPU／GPU P95が16.67 ms以下、10分deadline miss 0、real render rate 60 fps以上の場合だけ評価対象にする。

Desktopのinput-to-photonは、固定した1000 Hz USB input injectorから240 Hz以上のVRR display上の指定flash領域までをphotodiodeで測る。VSync／VRR／HDR／display overdrive、解像度、real render rate、Provider latency modeを同一にし、warm-up後240入力×5 run、nearest-rank P95のmedianを採る。Provider有効時のP95が無効時より5.00 msを超えて悪化せず、かつ50.00 ms以下の場合だけFrame Generationを推奨表示する。光学測定receiptがない環境では`input-to-photon qualified`を表示せず、CPU markerやPresent時刻だけで代用しない。

## 17. Ray Tracing、Path Tracing、Neural Rendering

### 17.1 共通Ray Tracing Port

`RayTracingPortV1`はAcceleration Structure build／update、ray query、ray dispatch、shader table／function table、scratch、compaction、timestampだけを公開する。D3D12 DXR、Vulkan Ray Tracing、Metal Ray TracingへAdapterが写像し、Material／Lightingは同じEngine-owned IRを使う。

BLAS inputはAsset規約のportable ray geometry、TLAS inputはcurrent RenderSnapshotから作る。native acceleration structureをTarget横断Assetとして保存しない。static BLAS compaction、dynamic refit／rebuild判断、scratch lifetime、queue ownershipはRender Graph planへ含める。

### 17.2 機能段階

| Capability | 初期用途 | 必須fallback |
|---|---|---|
| `rt_shadow_v1` | 選択Light／casterのshadow | CSM／atlas／SDF |
| `rt_reflection_v1` | glossy reflection、off-screen補完 | probe＋SSR |
| `rtgi_v1` | bounded radiance cache／DDGI | lightmap＋irradiance／reflection probe |
| `path_trace_reference_v1` | Editor Reference、Photo、golden生成 | raster preview |
| `path_trace_runtime_v1` | Qualification済み高性能PC runtime | Hybrid Raster／RT |

RTGIはEngine-owned `RadianceCachePortV1`を正本とし、DDGI、RTXGI、FSR Radiance Caching等をAdapter候補にできる。初期`rtgi_medium_v1`は最大32,768 active probe、1 frame最大1,024 probe update、1 probe 64 ray、persistent＋scratch 256 MiB以下、GPU P95 3.00 ms以下とする。上限超過時はprobe updateを次のreal frameへ繰り越し、allocation拡張や未宣言のray削減を行わない。Preview／Beta SDKは`Experimental`から開始し、Production Projectへ暗黙導入しない。

Path TracerはMaterial BSDF、Light、Camera、Geometry、Environmentの同じCooked contractを使い、sample count、bounce、Russian roulette、MIS、accumulation、denoiser、convergenceをProfileへ保存する。初期Profileは次に固定する。

| Profile | Sampling／Path | Gate |
|---|---|---|
| `path_trace_reference_v1` | 4,096 spp、max depth 8、Russian roulette開始depth 3、next-event estimation＋MIS有効、FP32 accumulation、fixture固定seed、denoiser無効 | 2,048 spp画像との差がlinear RGB RMSE 0.005以下かつSSIM 0.995以上、peak memoryをreceipt化 |
| `path_trace_preview_v1` | 64 spp、max depth 6、Russian roulette開始depth 3、MIS有効、承認済みdenoiser任意 | Editor専用。reference表記禁止 |
| `path_trace_runtime_v1` | real frameあたり1 spp、max depth 4、Russian roulette開始depth 2、MIS有効、temporal＋spatial denoise | base real 60 fps hard Gate、4,096 spp referenceとの品質Gate、Hybrid Raster／RT fallback |

sample sequence、seed、light sampling table、floating-point modeはreceiptへ保存する。Editor Referenceはinteractive frame deadlineではなくsamples／pixel、収束時間、reference error、peak memoryをGateにする。Runtime Path Tracingは別途real 60 fps Gateを満たすまでProduction表示しない。

### 17.3 Neural Rendering

`NeuralRenderModelV1`はmodel ID、semantic input／output、architecture version、weight format、weight SHA-256、training data／license provenance、quantization、required feature、scratch／persistent byte、inference soft cap、fallbackを持つ。初期Production上限は一ViewFamily合計でpersistent＋scratch 512 MiB以下、inference GPU P95 2.00 ms以下とし、超過時は非Neural fallbackへ切り替える。Runtime download、Runtime training、未署名weight、arbitrary operator、network accessを禁止する。

初期Adapter対象はDLSS Ray Reconstruction、FSR Ray Regeneration、MetalFX Denoised Upscaling、D3D12 Cooperative Vector、Metal 4 Machine Learningである。Vulkan相当機能はKhronos extensionが対象実機でProduction合格した後だけ追加する。Neural outputは非Neural temporal／spatial denoiserまたはraw accumulated referenceへfallbackできなければShipping必須機能にしない。

## 18. Failure、Fallback、Security

`RendererProviderErrorV1`は少なくとも`NotInstalled | UnsupportedDevice | UnsupportedDriver | SignatureInvalid | LicenseNotApproved | VersionMismatch | MissingInput | InvalidFormat | InitializationFailed | ExecutionFailed | HistoryInvalid | SwapchainConflict | BudgetExceeded | DeviceFault`を持つ。AA Intent解決前の互換／排他／Scope失敗は16.0節の`AntiAliasingResolutionErrorV1`を使い、Provider障害と混同しない。

- 起動時のProvider失敗は`ResolvedRendererProfileV1`の承認済みfallback順で次候補を選び、理由を表示する。
- Running中のProvider failureは新規Providerへその場で差し替えず、generationを停止し、次のLoading境界でContextを再生成する。安全なreal frame presentを継続できなければRenderer faultへ遷移する。
- RT／Neural pass失敗は同じframeの未宣言fallbackを差し込まない。次frameのGraphを承認済みRaster／非Neural Profileで再compileする。
- Provider DLL／modelはabsolute canonical path、expected SHA-256、OS署名またはVendor署名を検査する。Developmentのunsigned artifactをShippingへ含めない。
- Providerのtelemetry、crash dump、OTA、network機能は既定無効とし、必要な場合はPrivacy／Security Reviewと明示User consentを別に要求する。

## 19. PerformanceとTelemetry

Runtime規約のGPU P95 14.00 ms、hard 16.67 msとPass別soft capを維持する。Temporal ReconstructionやFrame Generationを使わないNative RasterでC1 hard gateへ合格する。高度ProfileもPass soft cap合計とheadroomを超えてはならない。

最低telemetryは次である。

- AA Intent／Plan ID、方式、raster sample count、render／display extent、jitter／history generation
- AA pass GPU P50／P95／max、bandwidth、persistent／transient byte、resolve回数、Pipeline variant数
- edge alias energy、edge spread、temporal shimmer、ghost persistence、history reset理由、fallback回数

- CPU extract、cull、LOD、Graph compile、record、submit
- Pass別GPU timestampとpipeline statistics
- visible／culled／draw／dispatch／triangle／sprite／glyph count
- 2D batch key、instances／batch、Tile authored／resident／requested／visible chunk、draw span、dirty rebuild、atlas page／occupancy／padding waste
- Representation別authored／resident／visible count、instance batch数、instances／draw、spatial cell／HLOD遷移、individual fallback理由
- LOD class／tier別transition、境界往復、minimum residency違反、projected error、geometry residency miss、fallback reason
- HZB test／visible／false-negative guard、indirect command、meshlet、overflow
- transient peak、alias saving、GPU budget／usage／eviction
- descriptor、PSO cache hit、shader variant
- upload／readback bytes、streaming miss、promotion wait
- barrier、queue wait、async overlap
- history invalidation、surface generation、device fault
- Provider、SDK／model generation、real／display extent、real／displayed fps、generated ratio、present latency
- motion／reactive／composition coverage、history reset、upscale／FG GPU／CPU cost
- BLAS／TLAS build／refit／compaction、ray count、ray depth、RT pass、denoiser、radiance cache、path tracing convergence
- Shadow Plan hash、atlas occupancy、cache hit／invalidation、dirty update、virtual page request／resident／evict、coarse fallback、filter sample、Technique別GPU／memory

10分soakでCPU／GPU deadline、memory、descriptor、submission serial backlogを検証する。GPU timestamp未対応／不安定DeviceではAvailabilityを記録し、0 msとして合格させない。最適化はBefore／After、同一input trace、Profile、driver、SDK、capture、visual diffを一つの`RendererOptimizationReceiptV1`へ保存する。

Frame平均値だけでなくfirst-use hitchをProduction Gateに含める。Cooked Asset closureから必須pipeline key集合を決定論的に列挙し、起動／loading中にbackground PSO作成を開始する。操作可能frameの公開条件はcritical pipeline準備完了、または事前承認済み意味同等fallbackの準備完了である。120秒runではpipeline miss、compile wait、default material代替、draw skipの件数と最長stallを記録し、critical pipelineのruntime compile／blocking miss 0件をC1／C2 hard Gateとする。非critical background作成はStreaming priorityで実行し、Runtime規約のCriticalSimulation／CriticalRenderを飢餓させない。

高度CapabilityはPortable Rasterと同じScene／input／output意味で比較する。Advanced経路が速くても、Portable fallbackのC1 Gate、Mobile Artifact、device recovery、Package起動を退行させる変更は共通Sourceへ昇格しない。共通改善とTarget限定改善を別Receiptにし、Target限定改善はCapability Signature一致時だけ選択する。

## 20. TestとRelease Gate

Temporal／RT／Neuralの画質比較は`RendererVisualReceiptV1`へ固定する。比較画像をlinear Rec.709 RGB32Fへ変換し、UI／pixel-locked領域は別maskでbit-exact、3D領域はSSIM 0.950以上かつnormalized RGB RMSE 0.025以下、NaN／Inf 0 pixelを合格条件とする。disocclusion mask内でabsolute RGB error 0.10超が3 real frameを超えて残るpixelを0件とする。Path Referenceだけは17.2節の厳格値を使う。同じframe ID、camera、exposure、jitter、dynamic extent、Provider preset、driver、SDK／model hashをreceiptへ保存し、閾値変更はBefore／After corpusとADRを必要とする。

AA方式ごとに`AntiAliasingVisualReceiptV1`を追加する。4x linear-resolution SSAA downsampleを静止Referenceとし、AA Offを動的Baselineにする。UI／text／pixel-locked maskはbit-exact、NaN／Inf 0 pixelを必須とする。edge mask内alias energyはOff比20%以上低減、FXAA／SMAA／MSAAのedge spread P95は1.50 display pixel以下、TAA／TAAU／temporal Providerは2.00 display pixel以下、temporal方式のshimmer energy P95はOff比30%以上低減する。disocclusion ghostは前段の3 real frame上限を共有する。方式が対象としないspecular／texture／transparent aliasは失敗扱いにせず`unaddressed_alias_class`へ列挙し、Resolver Previewが補助Capabilityまたは別方式を提示する。

- Graph cycle、read-before-write、unordered write、subresource overlap、history invalidationのunit／property test
- 同一Graph入力からcanonical compile plan hash一致
- D3D12 Enhanced／legacy、Vulkan、Metalへのaccess／barrier conformance
- D3D12 Debug Layer／GPU validation、Vulkan Validation、Metal validationのzero-error fixture
- 2D pixel、Realistic、Toon、Pixel Dioramaのgolden imageと許容差
- AA Off／FXAA／SMAA 1x／MSAA 2x・4x・8x／Mirakan TAA／Mirakan TAAUのthin geometry、foliage、alpha scissor／blend、specular、particle、emissive、skinning、急加速、camera cut、dynamic extent、HDR fixture
- AA Scope conflict、TAA＋MSAA、Deferred MSAA、unsupported sample count、missing motion／depth、pixel-locked temporal、Settings Apply外rebuildのnegative test
- D3D12／Vulkan／MetalでMSAA color／depth sample count、Pipeline key、resolve order、alpha-to-coverage、surface loss後再構築が一致するBackend conformance
- RTX 3060、RX 6600、Android minimum／reference、A12／referenceのNative Raster Performance
- RTX 5070、RX 9070、Arc B580の1440p60、1080p120、Vendor feature別Qualification
- resize、alt-tab、HDR／SDR切替、surface loss、device removed fault injection
- GPU OOM、descriptor exhaustion、pipeline miss、corrupt shader、stale Asset generation
- resource last-use serial前に破棄されないlifetime test
- CPU direct／GPU indirect／meshletの同一visible result、occlusion history、camera cut、overflow、fallback
- FOV、projection、resolution、dynamic extentごとの`projected_error_px_q16` golden値、CPU／GPU tier一致、enter／exit hysteresis、camera cut再選択
- `individual | instanced | spatial_batch | presentation_batch`のCook分類、Gameplay identity保持、mutable objectのstatic batch混入拒否、C1 CPU instancing
- HLOD Source順序変更時のcluster／Artifact hash一致、interactive／Physics／Save／animation混入拒否、HLOD on／offのGameplay結果一致
- Runtime規約の2D／3D Integrated Scale Fixtureで大量配置、spawn、streaming、敵味方VFXが同時発生してもNative Raster hard gateとvisual equivalenceを満たす
- Native TAA／DirectSR／DLSS／XeSS／FSR／MetalFXのmotion、depth、exposure、reactive、UI、HDR、dynamic extent、camera cut
- 一つのFG ProviderだけがSwapchainを所有し、Menu／Pause／Loading／resize／base miss時に停止するconformance
- Provider署名、hash、version、missing DLL、unsupported driver、initialization／execution failure、switch teardown
- RT Shadow／Reflection／GIのRaster fallback、BLAS／TLAS lifetime、dynamic geometry、alpha、device fault
- Path Tracing reference sceneの収束、Material／Light parity、deterministic seed、denoiserなし／あり比較
- Neural model hash、license、input schema、memory／latency、corrupt／unsigned model、非Neural fallback
- Shadow Graph cycle／上限／unsupported node、Technique未宣言access、Interface hash不一致、Target fallback欠落のnegative test
- conventional／cached／virtual／ray-traced Shadow Planを同じ`shadow_attenuation_linear`契約へ接続するBackend conformance
- UI／text／pixel-locked layerがdynamic resolution、Temporal Reconstruction、Frame Generationで劣化しないtest
- AIが未登録Pass、native resource、unsupported Target feature、arbitrary modelを生成できないconformance

性能runはRuntime規約の120秒×5と10分soakを使い、Mobileは30分thermalと2時間enduranceを追加する。Provider更新は旧／新versionを同じcommit、scene、trace、driverで測るbridge baselineを必須とする。

C1 RendererはPortable Rasterで2D top-down sliceと3D compact arenaを全Targetの該当Qualityへ描画し、Graph validation、device recovery、memory／frame budget、manual／AI authoringを満たした時点で完了する。C2／C3は各Capabilityが本章の個別Gateへ合格した場合だけProductionへ昇格する。

## 21. 段階導入

| 段階 | 成果物 | Promotion Gate |
|---|---|---|
| R0 | Contract、`AntiAliasingIntentV1`／Plan、FXAA、Mirakan TAA、D3D12 MSAA 2x／4x、CPU LOD reference selector、hysteresis、LOD0 fallback、Render Graph、telemetry、capture、Native Raster fixture | D3D12 C1 2D／3D、AA visual／resolve、LOD metric golden |
| R1 | Vulkan／Metal Portable Raster、FXAA／MSAA 2x・4x Mobile Profile | 実機AA visual、tile memory／bandwidth、thermal／endurance |
| R2 | GPU LOD selector、indirect、HZB、HLOD、geometry residency／streaming、geometry artifact | CPU selector一致、HLOD Gameplay分離、全Vendor baseline |
| R3 | SMAA 1x、MSAA 8x、`TemporalFrameInputV1`、Mirakan TAAU、DirectSR、DLSS、XeSS、FSR、MetalFX | AA／temporal visual、latency、failure、signature |
| R4 | Frame Generation、latency provider、UI separation | real 60 fps、present／input latency |
| R5 | RT Shadow／Reflection、acceleration lifecycle、denoiser | Raster fallback、RT Vendor matrix |
| R6 | RTGI／Radiance Cache | indoor／outdoor／dynamic fixture |
| R7 | Editor Reference Path Tracer | Material parity、convergence、golden |
| R8 | Neural denoise／reconstruction／radiance cache／shader | model provenance、quality、memory、latency |
| R9 | Runtime Path Tracing、Work Graph、large-world virtual geometry | 専用C3 Gate |

後段Capabilityを先行実装して前段のNative Raster、instrumentation、fallback、reference fixtureを省略しない。R3以降はProviderごとに独立task／Receiptを持ち、一つのSDK合格を他SDKの合格として扱わない。

## 22. 一次資料

### 22.1 Graphics API

- [Direct3D 12 Programming Guide](https://learn.microsoft.com/en-us/windows/win32/direct3d12/directx-12-programming-guide)
- [DirectX 12 Agility SDK Downloads](https://devblogs.microsoft.com/directx/directx12agility/)
- [D3D12 Enhanced Barriers](https://microsoft.github.io/DirectX-Specs/d3d/D3D12EnhancedBarriers.html)
- [D3D12 Render Passes](https://microsoft.github.io/DirectX-Specs/d3d/RenderPasses.html)
- [D3D12 Work Graphs](https://microsoft.github.io/DirectX-Specs/d3d/WorkGraphs.html)
- [DirectSR](https://microsoft.github.io/DirectX-Specs/DirectSR/DirectSR.html)
- [HLSL Cooperative Vectors](https://microsoft.github.io/hlsl-specs/proposals/0029-cooperative-vector/)
- [Vulkan Performance Samples](https://docs.vulkan.org/samples/latest/README.html)
- [Vulkan Descriptor Management](https://docs.vulkan.org/samples/latest/samples/performance/descriptor_management/README.html)
- [Vulkan Ray Tracing Guide](https://docs.vulkan.org/guide/latest/extensions/ray_tracing.html)
- [Metal Performance and Settings](https://developer.apple.com/documentation/metal/improving-your-games-graphics-performance-and-settings)
- [MetalFX](https://developer.apple.com/documentation/metalfx)
- [Metal Ray Tracing](https://developer.apple.com/documentation/metal/accelerating-ray-tracing-using-metal)
- [Metal Feature Set Tables](https://developer.apple.com/metal/Metal-Feature-Set-Tables.pdf)

### 22.2 Provider SDK

- [NVIDIA Streamline](https://github.com/NVIDIA-RTX/Streamline)
- [NVIDIA DLSS Integration](https://github.com/NVIDIA-RTX/Streamline/blob/main/docs/ProgrammingGuideDLSS.md)
- [NVIDIA DLSS Frame Generation Integration](https://github.com/NVIDIA-RTX/Streamline/blob/main/docs/ProgrammingGuideDLSS_G.md)
- [AMD FSR SDK](https://gpuopen.com/manuals/fsr_sdk/)
- [AMD FSR Temporal Upscaling Inputs](https://gpuopen.com/manuals/fsr_sdk/techniques/super-resolution-temporal/)
- [Intel XeSS SDK](https://github.com/intel/xess)
- [Intel XeSS Frame Generation Guide](https://github.com/intel/xess/blob/main/doc/xess_fg_developer_guide_english.md)

### 22.3 有名Engineの大量表現経路

- [Unity 6: GPU Resident Drawer](https://docs.unity3d.com/6000.1/Documentation/Manual/WhatsNewUnity6Preview.html)
- [Unity: GPU instancing](https://docs.unity3d.com/Manual/gpu-instancing-enable.html)
- [Unity Entities: Entity Command Buffer](https://docs.unity.cn/Packages/com.unity.entities%401.0/manual/systems-entity-command-buffers.html)
- [Unreal Engine: Mass Gameplay Overview](https://dev.epicgames.com/documentation/en-us/unreal-engine/overview-of-mass-gameplay-in-unreal-engine)
- [Unreal Engine: HLOD Overview](https://dev.epicgames.com/documentation/unreal-engine/hierarchical-level-of-detail-overview-in-unreal-engine)
- [Unreal Engine: World Partition](https://dev.epicgames.com/documentation/unreal-engine/world-partition-in-unreal-engine)
- [Godot: Optimization using MultiMeshes](https://docs.godotengine.org/en/stable/tutorials/performance/using_multimesh.html)
- [Godot: Optimization using Servers](https://docs.godotengine.org/en/stable/tutorials/performance/using_servers.html)

これらから、通常object経路とは別にECS／Mass／Server、instancing、LOD／HLOD、streaming、poolingを組み合わせる一般的な構造を採用する。ただし、外部Engineの個別APIや「大量なら表示を消す」という判断を模倣せず、MiraikanaiではScale intent、Gameplay fidelity floor、Representation Plan、統合実測GateをAI制作経路の必須契約にする。

### 22.4 有名Engineの性能・実行経路

- [Unreal Engine: Scalability Reference](https://dev.epicgames.com/documentation/en-us/unreal-engine/scalability-reference-for-unreal-engine)
- [Unreal Engine: Lumen Performance Guide](https://dev.epicgames.com/documentation/en-us/unreal-engine/lumen-performance-guide-for-unreal-engine)
- [Unreal Engine: PSO Precaching](https://dev.epicgames.com/documentation/en-us/unreal-engine/pso-precaching-for-unreal-engine)
- [Unity: Choose a render pipeline](https://docs.unity3d.com/Manual/choose-a-render-pipeline.html)
- [Unity: Burst compilation](https://docs.unity3d.com/Manual/script-compilation-burst.html)
- [Godot: Renderers overview](https://docs.godotengine.org/en/stable/tutorials/rendering/renderers.html)
- [Godot: General optimization](https://docs.godotengine.org/en/stable/tutorials/performance/general_optimization.html)

これらから、品質Profileと対象hardwareに応じた実行経路、hot pathのcompiler／data-oriented最適化、計測主導のbottleneck選択、PSO first-use hitchの事前除去を採用する。ただし、Miraikanaiでは外部EngineのAPI、設定項目数、互換性区分をそのまま移植せず、単一Authoring SourceからTarget別artifactを生成し、Portable基準経路と意味同等fallbackを維持する。

### 22.5 Anti-aliasingとAI Tool設計

- [Unreal Engine 5.8: Anti-Aliasing and Upscaling](https://dev.epicgames.com/documentation/en-us/unreal-engine/anti-aliasing-and-upscaling-in-unreal-engine)
- [Unreal Engine 5.8: Rendering Project Settings](https://dev.epicgames.com/documentation/unreal-engine/rendering-settings-in-the-unreal-engine-project-settings)
- [Unreal Engine 5.8: Unreal MCP](https://dev.epicgames.com/documentation/unreal-engine/unreal-mcp-in-unreal-editor)
- [Unity 6.3 URP: Add anti-aliasing](https://docs.unity3d.com/6000.3/Documentation/Manual/urp/anti-aliasing.html)
- [Godot stable: 3D antialiasing](https://docs.godotengine.org/en/stable/tutorials/3d/3d_antialiasing.html)
- [Godot stable: Renderer feature comparison](https://docs.godotengine.org/en/stable/tutorials/rendering/renderers.html)

| 公式Engine | 公式仕様で確認した構成 | Miraikanaiへ採用する判断 |
|---|---|---|
| Unreal Engine 5.8 | TSR、TAAU、FXAA、MSAAを選択できる。MSAAはForward Renderer限定で2x／4x／8x、geometry edge中心でmaterial／texture／transparent aliasを単独では解決しない。Project／Scalability設定に分かれる | temporal／spatial／multisampleを別Capabilityにし、MSAAをForward+限定、2x／4xと8xを別成熟度にする。方式だけで全alias解決と表示しない |
| Unity 6.3 URP | FXAA／SMAA／TAAはCamera、MSAAはURP Assetで設定する。TAAはMSAA、Camera Stacking、Dynamic Resolutionと非互換。MobileではFXAAを推奨し、StoreAndResolve非対応端末でOpaque Textureを使うとMSAAが無効になる | Project／Camera入力をViewFamily Planへ統合し、MSAA×temporalをPortable禁止にする。Mobile既定をFXAAとし、scene color sampling／store・resolve非互換を黙って無効化しない |
| Godot stable | MSAAとSSAAは全Renderer、FXAA／SMAAはForward+／Mobile、TAA／FSR2はForward+だけ。MSAAは2x／4x／8xを持ち、FXAAは低cost、TAAはtemporal安定、MSAAは低blur用途として説明される | Renderer／Target互換表を正本化し、Goalから候補を選ぶ。複数方式の無条件併用ではなく、単独方式と比較したQualificationを必須にする |
| Unreal MCP 5.8 Experimental | Toolset、JSON Schema、Tool search／describe／callによる段階的Discoveryを提供するが、Experimentalで変更可能性があり、localhost接続自体は認証境界ではない | typed MCD Operationと段階的Discoveryは採用し、Commit／Provider activation／Pipeline rebuildはToolへ投影しない。Gateway authorityとrevision検証を別に強制する |

これらから、方式とRenderer／Platformの互換表、MSAA 2x／4x／8x、spatial／temporalの画質・cost差、TAAとMSAAのPortable排他、Camera／Project設定Scope、型付きTool schema、段階的Tool discoveryを採用する。外部Engineのconsole variable、Camera Component、Pipeline Asset、ProjectSettingsをMiraikanaiの正本にはせず、`AntiAliasingIntentV1`からViewFamily単位Planを生成し、Gateway検証とTarget別Qualificationを必須にする。

これらのAPI／SDKは機能、入力、Capability、性能上の注意の根拠である。MiraikanaiのRender Graph schema、Provider選択、fallback、budget、AI公開範囲、Production表示を外部API／SDKへ委ねるものではない。
