# Miraikanai Engine Render Graph Contract

- 文書ID: mirakan.arch.rendering-render-graph
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Renderer公開境界、Render Snapshot／View、resource／pass graph、queue／barrier／lifetime execution、transient alias／GPU visibility optimization eligibility、surface composition、2D presentation packet・sorting・batching、visibility／geometry execution、resolved lighting／light-transport execution、anti-aliasing／temporal resource execution、Renderer固有failure／qualification
- 非正本範囲: Project Shader Source／semantic Module／Technique Manifest意味／AI理解、Material／Lighting／Advanced Light Transport／Post Process／Terrain／LOD／Worldのauthoring semanticsとTechnique選択、Runtime phase／shared capacity、Asset transaction、Tool／SDK version、AI authorization、Evidence envelope、共通Schema／projection。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Core Architecture](../02-foundation/core-architecture.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[World](world.md)
- 関連文書: [Product Plan](../00-product/product-plan.md)、[Advanced Rendering／Multiplayer Ownership Decision](../decisions/2026-07-29-advanced-rendering-multiplayer-ownership.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core architecture](../02-foundation/core-architecture.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Editor UI Framework](../03-authoring/editor-ui-framework.md)、[Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[Animation](../05-simulation/animation.md)、[Collision](../05-simulation/collision.md)、[Navigation](../05-simulation/navigation.md)、[Materials](materials.md)、[Project Shader](project-shader.md)、[Lighting](lighting.md)、[Advanced Light Transport](advanced-light-transport.md)、[Environment／Water／Weather／Snow](environment-surfaces.md)、[Terrain／Foliage](terrain-foliage.md)、[VFX Runtime](vfx-runtime.md)、[Post Processing](post-processing.md)、[LOD](lod.md)、[Virtualized／Continuous Geometry](virtualized-continuous-geometry.md)、[World](world.md)、[Camera](camera.md)、[Windows](../07-platform/windows.md)、[Mobile Common](../07-platform/mobile-common.md)、[Android](../07-platform/android.md)、[Apple](../07-platform/apple.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-27

## 1. 結論と所有境界

RendererはProject C++、Gameplay、Editor、AIからnative API object、command list、descriptor index、GPU address、shader binaryを隔離し、Engine-owned handleとimmutable input snapshotだけを受ける。Render Graphはpass、resource、queue、barrier、alias、temporal history、submissionを一意に計画し、Backend Adapterはその計画をnative APIへ写像する。

本書でいう「RHI相当境界」は、`CanonicalRenderExecutionPlanV1`を受ける`GraphicsDevicePort`と、その下にあるprivate Backend Adapterの境界を説明する比較用語である。`RHI`という公開Module、Class、Stable ID、Project APIを別に新設せず、正本名には既存の型とModule名を使う。

宣言的`RenderGraphDefinition`はresource／pass／view／dependencyだけを持ち、`render_graph` moduleがcanonical execution planへcompileする。Engine-owned Pass Templateと[Project Shader](project-shader.md)でQualification済みのTechnique ManifestだけをDefinitionへ展開し、callback外access、native barrier／queue signal、Backend objectを埋め込まない。

Runtime phase、Simulation Advance、job dependency、submission lifetimeは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、共通CPU／GPU／memory budgetと測定法は[Runtime performance／capacity](../04-runtime/performance-capacity.md)だけが決定する。本書はRenderer固有のresource pressure、fallback、correctnessを定義するが、共通値や測定envelopeを複写しない。

Materialのshading意味、Lightの物理意味、GI／reflection／advanced shadow／reference transportのTechniqueとfallback、Post Processのvolume／effect composition、Terrain／Foliage domain、LOD representation選択、World source／streaming planは各同階層Ownerが決定する。Rendererは解決済み入力を実行し、他DomainのSource Documentを解釈しない。

## 2. Module境界と公開Port

ModuleはContracts、Render Extract、Graph Compiler、Resource Registry、Pipeline Cache、Visibility／Geometry、Surface Composite、Backend Adapter、Renderer Qualificationに分ける。Backend AdapterとProvider Adapterはprivateであり、Vendor型、result code、archive、extension structを公開Portへ漏らさない。

`GraphicsDevicePort`はadapter／device／surface Capability query、resource／view／sampler／pipeline作成、descriptor／argument binding、command allocator／buffer取得、logical barrier plan encode、queue submit／serial／completion、present／resize／device fault report、budget／residency telemetryだけを公開する。native型をPort header／MCD／Asset metadata／Project C++へ出さない。

### 2.1 RHI相当境界

論理計画からnative APIまでの一方向関係は次で固定する。

```text
RenderGraphDefinition
  -> Graph Compiler
  -> CanonicalRenderExecutionPlanV1
  -> GraphicsDevicePort
  -> private D3D12 | Vulkan | Metal Backend Adapter
  -> native graphics API
```

| 境界 | 所有するもの | 禁止するもの |
|---|---|---|
| Render Graph／Compiler | pass・resource・dependency、canonical順序、logical queue／barrier／lifetime plan | native command、Vendor feature struct、Backend別pass追加 |
| `GraphicsDevicePort` | Engine-owned descriptor／handle、作成・encode・submit・completion・surface・fault・telemetryの抽象操作 | Project向け描画API化、Graph意味の再解釈、native handle返却 |
| private Backend Adapter | logical planのnative object／command／synchronization／presentationへの写像 | logical pass／resource／edgeの追加・削除・並べ替え、別TargetのCapability推測 |
| Platform Target owner | Target Profile、Platform Capability観測、surface／lifecycle入力 | Renderer plan、resource lifetime、barrier意味の再定義 |

`GraphicsDevicePort`の設計閉包は、具体的なC++ method一覧ではなく次の操作分類と不変条件で判定する。

| 操作分類 | Engine入力 | Engine出力／不変条件 |
|---|---|---|
| Capability／surface | exact Target／Platform Capability ref、surface generation | 正規化済みCapability、generation付きsurface state |
| resource／pipeline lifetime | closed descriptor、Engine handle、artifact hash | generation付きhandle。native objectを返さない |
| binding／command encode | `CanonicalRenderExecutionPlanV1`、logical barrier plan、parameter block | logical planを変更しないBackend work |
| submit／completion／present | Engine queue class、surface generation、work batch | `GpuSubmissionSerial`、typed completion／present result |
| fault／budget telemetry | device generation、submission serial、Engine field mask | stable Renderer diagnostic、正規化済みbudget／residency。native detailはprivate attachment |

PortのC++ ABI、method signature、MCD生成物、Backend実装がRepositoryに存在しない間、この表は設計境界であり実装済みinterfaceではない。AI／Editorは操作分類から未定義method名を生成せず、[Architecture Governance §5.3](../01-governance/architecture-governance.md#53-architecture-explain-projection)のread-only projectionでOwner、関係、状態を確認する。

公開入力は次のinventoryに限定する。以後の節と他文書はこの表のIDだけを参照する。種別は`frame`（毎frameのimmutable入力）、`resolved`（Owner Resolverの解決済みPlan）、`cook`（Cook由来artifact）である。`AntiAliasingIntentV1`等のplanned authoring action入力は§11の正本経由であり、frame入力へ混在させない。

| 公開入力 | 種別 | 正本schemaとOwner定義 |
|---|---|---|
| `RenderSnapshot` | frame | 本書§2.2。published simulation／world stateから抽出したimmutable frame input |
| `ViewFamilyV1` | frame | 本書§2.2の`RenderViewV1`集合。同じsurface、render extent policy、AA plan、exposure familyを共有する |
| `ResolvedMaterialBindingV1` | frame | [Materials](materials.md) §5。`CookedMaterialArtifactGenerationRefV1`、typed Instance／Runtime Override、`MaterialBatchCompatibilityKeyV1`の解決結果 |
| `LightSnapshotV1` | frame | [Lighting](lighting.md) §6。`ResolvedLightSet`の唯一のRenderer公開形で、各View Familyにexact一件の`RenderSnapshot.light_snapshots[]` memberとして受ける |
| `EnvironmentPresentationSnapshotV1` | frame | [Environment／Water／Weather／Snow](environment-surfaces.md) §5。Environment Profileと完成Artifact generationのprojection |
| `WaterSnapshotV1` | frame | [Environment／Water／Weather／Snow](environment-surfaces.md) §5。Water surface／underwater／query generationのprojection |
| `WeatherPresentationSnapshotV1` | frame | [Environment／Water／Weather／Snow](environment-surfaces.md) §2.3。Weather Provider状態の非authoritative presentation projection |
| `SnowSurfaceBatchV1` | frame | [Environment／Water／Weather／Snow](environment-surfaces.md) §5。Snow receiver／page／Material bindingのprojection |
| `VfxBatchSnapshotV1` | frame | [VFX Runtime](vfx-runtime.md) §3。CPU draw batchとGPU emitter advanceのimmutable projection |
| `ResolvedPostProcessPlanV1` | resolved | [Post Processing](post-processing.md)が解決したordered effect composition |
| `ViewLodCandidateSetV1` | frame | 本書§7。LOD選択前のconservative frustum／layer candidate集合。selected tier／occlusionを含まない |
| `ResolvedRepresentationSet` | frame | 本書のframe入力名。[LOD](lod.md)所有の`LodResolutionPlanV1`／`ViewLodContextV1`に基づくruntime選択結果（representationとtransition state）をViewFamilyごとに整列する |
| `ResolvedAntiAliasingPlanV1` | resolved | 本書§9。[Executable contracts](../02-foundation/executable-contracts.md)正本の`AntiAliasingIntentV1`から解決する |
| `ResolvedOutlineExecutionPlanV1` | resolved | 本書§8.1。[Materials](materials.md)の`OutlineStyleProfileV1`をEngine-owned qualified techniqueへ解決した結果 |
| `ResolvedShadowPlanV1` | resolved | [Advanced Light Transport](advanced-light-transport.md) §6。Lighting-owned Shadow authoringのchannel／Technique解決結果 |
| `ResolvedLightTransportPlanV1` | resolved | [Advanced Light Transport](advanced-light-transport.md) §5。channel別Technique／representation／fallbackの解決結果 |
| `RenderRepresentationPlanV1` | cook | 本書§12。Runtime planからCookした分類plan |
| `Renderer2DExecutionPlanV1` | cook | 本書§12。2D packet抽出plan |

SnapshotはSource Stable ID、artifact generation、Transform、bounds、visibility mask、previous presentation stateをEngine handleで参照する。Rendererはauthoritative World／Physics stateを書き戻さない。Animation skin／poseは[Animation](../05-simulation/animation.md)が公開したgeneration付きsnapshotとして読み、GPU skin executionだけを所有する。

[World](world.md)所有の`WorldRepresentationSummaryV1`とactive Cell lineageは`RenderSnapshot.renderable_2d[]`／`renderable_3d[]`の抽出条件として使い、別の公開`WorldRenderPacket`型を作らない。World summary、Cell revision、RenderableのWorld lineageのいずれかが一致しなければSnapshot全体をpublishせず、配列先頭、前frameまたは同名Worldから補完しない。

Renderableの`RenderObjectKey`は`{renderable_type_id, pipeline_key, material_key, geometry_key, stable_render_id}`のcanonical tupleである。Transparent sortはStyleのdepth／priorityとStable IDを使い、pointer／submission orderをtie-breakにしない。

### 2.2 `RenderSnapshot`と`RenderView`

```text
RenderSnapshot
  schema_version
  snapshot_id
  simulation_cadence_profile_ref: SimulationCadenceProfileRefV1
  simulation_advance_interval_hash: SHA-256
  advance_sequence
  project_revision
  world_generation
  asset_generation_id
  view_families[1..64]: ViewFamilyV1
  renderable_2d[]
  renderable_3d[]
  light_snapshots[1..64]: LightSnapshotV1
  environment: nullable<EnvironmentPresentationSnapshotV1>
  water_snapshot: nullable<WaterSnapshotV1>
  weather_presentation: nullable<WeatherPresentationSnapshotV1>
  snow_surface_batch: nullable<SnowSurfaceBatchV1>
  vfx_batch: nullable<VfxBatchSnapshotV1>
  ui_snapshot
  debug_batch
```

Snapshotは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)のpublish contractで全体を一度だけpublish後immutableとする。先頭のCadence Profile ref、Interval hash、advance sequenceは同じpublish対象`SimulationAdvanceIntervalV1`とbyte equalityにし、Presentation側でrateまたはdurationを補完しない。Entity pointer、Component span、native Physics／GPU objectを含めず、ArrayはStable rendering keyでcanonical sortしworker completion順を保存しない。

各`view_families[]` member `f`から`Ref(f)={view_family_id,view_family_generation,view_family_content_hash}`を投影し、`{ls.view_family_ref | ls in light_snapshots[]} = {Ref(f) | f in view_families[]}`をexact set equalityにする。`light_snapshots[]`は`view_family_ref`の三Field tuple順へstrict sortし、duplicate Family ref、missing／extra Family、同ID別generation／hash、配列位置または`latest`による対応を拒否する。各`LightSnapshotV1.project_revision`は`RenderSnapshot.project_revision`とbyte equalityでなければならない。Lightが0件のFamilyもSoA配列が全てemptyのexact一Snapshotを持ち、Family memberを省略しない。Advanced Light TransportへはRequirementの`view_family_ref`とbyte equalityになる一件だけを渡し、別FamilyのSnapshot、配列先頭、表示名または同じTargetから選択しない。

Environment bindingが存在する時、`environment`はnon-nullとし、その`weather_snapshot_ref`は`weather_presentation`のexact refとbyte equalityにする。Water／Snow／VFXをpublishする場合は各SnapshotのCadenceまたはWeather refを同じSimulation Advance／Environment generationへ束縛する。bindingまたは各Domain出力が存在しないFieldはcanonical nullであり、Rendererは配列先頭、直前frame、Source Profile、表示名からSnapshotや既定Environmentを補完しない。

```text
RenderViewV1
  schema_version: 1
  view_id: StableId
  view_generation: positive u64
  frame_view_slot: uint32
  purpose: game | editor | shadow | reflection | thumbnail
  projection:
    perspective | physical_perspective {
      vertical_fov_rad: finite (0, pi)
      near_m: finite positive f64
      far_m: finite f64 greater than near_m
    }
    | orthographic | pixel_orthographic {
        vertical_span_m: finite positive f64
        near_m: finite non-negative f64
        far_m: finite f64 greater than near_m
      }
  view_from_world: finite right-handed Matrix4x4
  render_width_px: positive u32
  render_height_px: positive u32
  viewport_px: finite integer rect inside render extent
  scissor_px: finite integer rect inside viewport
  render_scale: finite Target-Profile-bounded f64
  layer_mask: registered u64 layer mask
  visibility_origin: finite world-space Vec3
  camera_cut: bool
  history_reset: bool
  target_profile_ref: exact TargetProfileRefV1
  quality_profile_ref: exact QualityProfileRefV1
  history_key: StableId
  view_content_hash: SHA-256

RenderViewRefV1
  view_id: StableId
  view_generation: positive u64
  view_content_hash: SHA-256

ViewFamilyV1
  schema_version: 1
  view_family_id: StableId
  view_family_generation: positive u64
  world_scope_ref: exact WorldScopeRefV1
  target_profile_ref: exact TargetProfileRefV1
  quality_profile_ref: exact QualityProfileRefV1
  surface_generation_ref: exact owner-typed ref
  render_extent_policy_ref: exact owner-typed ref
  anti_aliasing_plan_ref: exact ResolvedAntiAliasingPlanRefV1
  exposure_history_key: StableId
  view_refs[1..64]: exact RenderViewRefV1
  view_family_content_hash: SHA-256

ViewFamilyRefV1
  view_family_id: StableId
  view_family_generation: positive u64
  view_family_content_hash: SHA-256

ViewFamilyRequirementV1
  requirement_id: content-derived StableId
  requirement_version: positive u32
  project_revision: positive u64
  view_family_ref: exact ViewFamilyRefV1
  world_scope_ref: exact WorldScopeRefV1
  required_view_refs[1..64]: exact RenderViewRefV1
  required_purposes[1..5]:
    game | editor | shadow | reflection | thumbnail
  target_profile_ref: exact TargetProfileRefV1
  quality_profile_ref: exact QualityProfileRefV1
  requirement_content_hash: SHA-256

ViewFamilyRequirementRefV1
  requirement_id: StableId
  requirement_version: positive u32
  requirement_content_hash: SHA-256
```

`RenderViewRefV1`、`ViewFamilyRefV1`、`ViewFamilyRequirementRefV1`はそれぞれ解決先のidentity、generation／version、content hashと全Fieldでbyte equalityにする。`view_refs[]`と`required_view_refs[]`は`view_id, view_generation`順、`required_purposes[]`は上記closed enum ordinal順へstrict sortし、duplicateを拒否する。FamilyのTarget／Qualityは全Viewとbyte equalityにし、全Viewは同じ`world_scope_ref`のWorldから解決されたcamera／derived Viewでなければならない。Requirementの`world_scope_ref`、Target／Qualityは参照先Familyとbyte equality、View集合はFamilyのView集合のnon-empty subset、purpose集合はそのView集合から投影したpurpose集合とset equalityにする。`exposure_history_key`は同じ露出測定／adaptation historyを共有するViewだけで同一にし、異なるWorld scope、surface generation、Post Process exposure modeまたはmetering scopeを同じkeyへ束縛しない。Exposure parameterのSourceは[Post Processing](post-processing.md)の`ExposureProfileV1`であり、Render Graphはprofile refや同値parameterをFamilyへ複写しない。

`view_id`と`view_generation`がframeを跨いでView identityを表し、`frame_view_slot`は一つの`RenderSnapshot`内のexecution indexに限る。同じSnapshot内ではslotをuniqueとするが、slotの再利用、配列順、surface上の位置からView identityまたはhistoryを復元しない。`view_content_hash`はASCII `MIRAKAN_RENDER_VIEW_V1`、Family hashはASCII `MIRAKAN_VIEW_FAMILY_V1`、Requirement hashはASCII `MIRAKAN_VIEW_FAMILY_REQUIREMENT_V1`と、それぞれ自己hashを除くclosed recordのcount／presence／length-framed canonical bytesからSHA-256する。CameraはSource／Rig／cut intentを公開し、Render Graph Ownerだけが解決済み`RenderViewV1`、Family、Requirementのcanonical recordを公開する。

View capabilityのactivationと導入順は[Product Plan](../00-product/product-plan.md)を参照し、本書はViewごとに独立selection／history stateとdomain qualification evidenceだけを提供する。

## 3. Resource modelとRender Graph

`RenderResourceDesc`を次で固定する。`kind`と`ownership`は独立したclosed enumであり、surface／historyをresource kindへ混在させない。

| Field | 規則 |
|---|---|
| `resource_id` | Graph instance内`uint32` |
| `kind` | `texture \| buffer \| acceleration_structure` |
| `ownership` | `transient \| imported \| persistent \| history \| external_surface` |
| `format` | Engine `PixelFormat` closed enum |
| `extent` | absolute、view-relative、named resource-relative |
| `mip_count`／`array_layers`／`sample_count` | Target Capability内 |
| `usage_flags` | attachment、sampled、storage、copy、indirect等 |
| `memory_class` | device local、upload、readback |
| `initial_access` | imported／persistentだけ必須 |
| `clear_value` | attachment clear時だけ、typed |
| `alias_group` | compilerが生成。Authoring入力不可 |
| `debug_name_id` | shipping identityに使わない |

Texture extent、mip、format、usageの組合せをTargetごとに検証し、0 size、overflow、unknown format、usage不整合、readback attachmentを拒否する。Imported resourceはowner、generation、acquire／release conditionを持ち、Graph外resourceを暗黙captureしない。

上位層が保持できるGPU object identityは`RenderResourceHandle { index32, generation32 }`、`PipelineHandle`、`MaterialHandle`だけである。D3D12 resource、`VkImage`、`MTLTexture`等はBackend registryが単一所有し、すべてのhandle lookupでhandle generation、Backend device generation、Asset versionの3つを検証する。一つでも不一致ならstale handle failureとし、native objectを返さない。

resource／pipeline／materialの再利用または破棄は、最後に使用したgraphics、async compute、copyの全queueそれぞれの`GpuSubmissionSerial`がcompletionを通過した後だけ許す。frame count、一queueのserial、CPU submission完了のいずれも代用できない。Device recreate時はBackend device generationを増加し、registry内の`RenderResourceHandle`、`PipelineHandle`、`MaterialHandle`とpending leaseをbulk invalidateする。旧device generationのhandleは全queue完了後のretire処理に限って参照でき、lookup／reuse／new submissionに使えない。

Pass declarationはStable ID、queue class、read／write／read-write access、subresource range、attachment、pipeline key、side-effect class、optional capability requirementを持つ。Pass callbackが宣言外resourceへ触れること、native barrierを発行すること、別queueへworkを隠すことを禁止する。

`RenderPassDescriptor`は`pass_id`、`pass_type`、`queue_class`、`view_scope`、`accesses[]`、`color_attachments[]`、`depth_stencil_attachment`、`execute_template_id`、`parameter_block`、`declared_cost`を持つ。Runtimeのpass type／execute templateはCooked Capability Catalogのclosed IDである。Project／AIはRuntime GPU callback、shader binary、native barrierを追加できないが、offline Source ChangeSetとして`ProjectShaderTechniqueV1`を提案し、Qualification後にCookerがgenerated execute templateとShader artifactへ変換できる。`ResourceAccess`はresource／subresource、`read | write | read_write`、logical stage／usageを明示し、descriptor外accessはvalidation faultである。

Graph Compilerは少なくとも次を検証する。

- use-before-produce、write conflict、cycle、未宣言access、incompatible format／sample、invalid present path。
- resource lifetime、transient alias overlap、temporal history generation、surface generation。
- queue ownership transfer、wait／signal dependency、barrier completeness、pass culling legality。
- Pipeline interfaceとresource binding reflectionの一致。

compile algorithm `R30_CompileRenderGraph`は次の固定順である。

1. resource、pass、view、Target Capabilityをschema validationする。
2. writer一意性、read-before-write、import initial state、feedback loopを検証する。
3. explicit dependencyとresource hazardからpass DAGを構築する。
4. `pass_id` tie-breakでcanonical topological sortする。
5. resourceのfirst／last useとqueue ownershipを計算する。
6. descriptor完全一致かつlifetime非重複のtransient resourceだけalias候補にする。
7. cross-queue signal／waitとBackend-neutral barrier planを作る。
8. compatible attachment passのmerge候補をBackendへ提示する。
9. transient heap、descriptor、command-list capacityを予約する。
10. compile keyとGraph diagnosticsを出力する。

cycle、uninitialized read、同一subresource unordered write、unsupported format、capacity超過はGraph全体を拒否し、resourceを別formatへ黙って変換しない。

同じGraph Definition、Target Profile、Capability Signature、Quality inputからは同じcanonical pass order、resource plan、pipeline key集合を生成する。worker completion順、pointer値、hash-map iteration順を計画へ使わない。

`CanonicalRenderExecutionPlanV1`（compile済みcanonical pass order、resource plan、queue／barrier plan）は本書がOwnerであるlogical planであり、D3D12等の各Backendはそのconsumerとしてnative mappingだけを所有する。Backendがlogical planへpass、resource、依存edgeを追加・並べ替えすることを禁止する。

## 4. Queue、barrier、lifetime execution

queue classはgraphics、async compute、copyをEngine語彙として公開し、実Deviceで利用不能なclassはGraph compile時に意味を保ったqueueへ統合するか、明示的にunsupportedを返す。Queue間dependencyはGraph edgeからだけ生成し、Backend固有signal値を保存形式やdiagnostic identityにしない。

### 4.1 D3D12 Enhanced Barriers capability

根拠: official-spec — Microsoftの[`D3D12_FEATURE_DATA_D3D12_OPTIONS12`](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/ns-d3d12-d3d12_feature_data_d3d12_options12)ではEnhanced Barriersはoptional capabilityであり、`EnhancedBarriersSupported`のfeature queryが必要である。

根拠: project-decision／provisional — MiraikanaiのD3D12候補BackendはEnhanced Barriersだけを使用し、Legacy Resource Barriersとの二重実装を持たない。この互換範囲はPrototypeと対応Device調査前の候補であり、対応済みとは表現しない。

D3D12 Device作成時にOPTIONS12をqueryし、`EnhancedBarriersSupported == TRUE`を確認してからD3D12 Render Graph capabilityを公開する。`FALSE`、query失敗、必要Agility SDKを読み込めない場合は`UnsupportedDevice`としてRenderer初期化前に拒否する。Legacy Barrierへ暗黙fallbackせず、Graph compileまたはsubmissionを開始しない。Capability Signatureはquery結果、Agility SDK version、Adapter／driver identityを保持し、Qualification Receiptは同じSignatureへ束縛する。

Transient resourceはcompile済みintervalの範囲だけ生存し、aliasはformat／alignment／queue overlap／clear semanticsが互換な場合に限る。Persistent resource、streaming resource、temporal history、swapchain surfaceはgeneration付きleaseで参照し、Device reset、resize、provider change、artifact promotionを跨ぐstale handleを拒否する。

同じphysical memoryを異なるtransient resourceへ再利用するcandidateは、size／alignment／heap／format／usage compatibility、全queueでのlifetime非重複、previous resourceのlast access、next resourceのfirst accessをcompile planへ固定する。D3D12 mappingはprevious／next resource間に必要なaliasing barrierを発行し、next resourceの最初の使用で未初期化領域を全てclear／discard／writeする。barrier省略、resource間のdata inheritance、cross-queue overlap、partial first write、compiler外のmanual aliasを禁止する。根拠はMicrosoftの[D3D12 memory aliasing and data inheritance](https://learn.microsoft.com/en-us/windows/win32/direct3d12/memory-aliasing-and-data-inheritance)とし、他Backendでも同じlogical lifetimeとvalidationを満たすnative同期だけをAdapterが選ぶ。

attachmentをrender passとして表現する最適化はTarget別candidateである。Microsoft公式の[D3D12 render passes](https://learn.microsoft.com/en-us/windows/win32/direct3d12/direct3d-12-render-passes)が示すようにtile-based renderer等でoff-chip memory trafficを抑え得るが、全GPUへ普遍的な高速経路としない。同じattachment load／store／resolve／clear semanticsとvisual oracleを保ち、Target／driver別実測がないcandidateは`not_evaluated`である。

Graph planは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)が公開するRenderer execution slotで実行する。本書は共通phase表、writer順、Simulation Cadenceを再掲せず、slot inputがimmutableであることと、leaseが記録した最後の全queue `GpuSubmissionSerial`完了後にだけ解放することを要求する。

## 5. Frame lifecycle、surface、recovery

Frame contextはinput snapshot revision、ViewFamily revision、Graph plan generation、surface generation、submission serialを束ねる。resize、display-mode change、surface lossはnew generationを発行し、旧generationのworkをretireしてから公開する。旧／新surface resourceの一frame混在を禁止する。

Device loss時は新規submissionを止め、diagnosticをfreezeし、in-flight leaseをfaultedとして回収し、BackendとProviderを再生成する。Source AssetやWorld stateをGPU resourceから復元せず、[Asset lifecycle](../03-authoring/asset-lifecycle.md)のCooked Artifactとpublished snapshotから再構築する。復旧不能ならGameHostを無限retryさせずRenderer faultを公開する。

## 6. 2D／3D／UI composition

World 2D、World 3D、post-processed scene、pixel-locked overlay、Editor UIは別layerとしてGraphへ登録する。depth、color space、alpha convention、sample count、output transferはlayer contractで固定し、UIをtemporal reconstructionやscene exposureへ暗黙入力しない。

Post Processのeffect順、volume blend、history reset要求は[Post Processing](post-processing.md)を正本とする。RendererはそのPlanをCatalog登録済みPass Templateへ展開する。UI／Editor packetのprimitive意味とaccessibilityはそれぞれのOwnerが決定し、本書はsurface compositeとsubmission lifetimeだけを所有する。

`LayerCompositionSummaryV1`は本書がOwnerとして公開するread-only／revisioned projectionであり、最低fieldとしてrevision、view_family_id、layer entry配列（layer ID、depth／color space／alpha convention、sample count、pixel-locked有無、AA除外区分）を持つ。layer IDの語彙は本節のlayer contract登録に従う。[Post Processing](post-processing.md) §7のresolver入力を含む消費側はfield一覧を複写せず書き戻さない。

## 7. Visibilityとgeometry execution

```text
ViewLodCandidateSetRefV1
  candidate_set_id: StableId
  candidate_set_generation: positive u64
  candidate_set_content_hash: SHA-256

ViewLodCandidateSetV1
  schema_version: 1
  candidate_set_id: StableId
  candidate_set_generation: positive u64
  view_id: StableId
  view_generation: positive u64
  view_lod_context_hash: SHA-256
  subjects[0..1048576]:
    subject_stable_id: StableId
    source_bounds_generation: positive u64
    conservative_world_bounds_ref: exact immutable bounds ref
    layer_mask: closed RenderLayerMaskV1
  candidate_set_content_hash: SHA-256
```

RendererはSource／World packetのconservative boundsをclosed View frustumとlayer maskだけで評価し、交差するsubjectをStable ID順へstrict sortしてcandidate setを作る。set hashはASCII `MIRAKAN_VIEW_LOD_CANDIDATE_SET_V1`と自己hashを除くcanonical bytesから計算する。selected representation、projected metric、occlusion history、resident stateをpreimageへ入れない。LOD ContextとView ID／generation／hash、bounds generationが一致しないsetを拒否し、camera cut、projection／extent／World bounds generation変更で新generationを作る。

Visibility executionはこのconservative candidate set、[LOD](lod.md)のResolved Representation、occlusion inputを順に使い、LOD選択後のocclusion、instance compaction、indirect command generationを行う。frustum／layer candidate生成はLOD selectionの前、occlusionは後であり、occlusion結果を同frameまたは次frameのtier／Simulation relevancyへ逆入力しない。representation選択やprojected-error policyをRendererで再計算しない。

GPU-driven pathとCPU reference pathは同じvisible item identity、material binding、geometry generationを生成しなければならない。HZBやocclusion historyはView／surface／projection generationへ束縛し、camera cut、teleport、extent change、history欠損ではconservative visibleへfallbackする。Work expansion機能を使ってもresource lifetime、queue、barrier、budget ownershipはRender Graphから移さない。

実行path IDは`renderer-profile.cpu-direct`、`renderer-profile.gpu-indirect`、`renderer-profile.gpu-meshlet`、`renderer-profile.gpu-work-graph`で、後3者はそれぞれ`renderer-profile.cpu-direct`または`renderer-profile.gpu-indirect`へfallbackできる。HLODのstatic eligibility roleは`decorative_instance`に限り、Gameplay identity／interactionを変更しない。

candidate評価順は`CPU direct conservative frustum／layer oracle -> ViewLodCandidateSetV1 -> LOD Resolver -> GPU representation packet／HZB occlusion／compaction／indirect draw -> meshlet／work graph`とする。GPU indirectはCPU referenceとcandidate／visible Stable ID集合、material／geometry generation、LOD tier、draw argument countが一致しなければならない。argument／count／compaction bufferのcapacity超過はtyped failureでGraph generationを不成立にし、partial command buffer、silent draw drop、CPU再実行を同frameへ挿入しない。Microsoftの[D3D12 ExecuteIndirect sample](https://learn.microsoft.com/en-us/samples/microsoft/directx-graphics-samples/d3d12-execute-indirect-sample-win32/)はcompute visibility cullingとindirect command compactionの実装比較根拠にだけ使用する。

HZBはconservative occlusionだけを許し、false occlusion（referenceでvisibleなitemを不可視にすること）を0にする。occluder selection、depth convention、mip reduction、reprojection、history validityはalgorithm／profile revisionへ含め、camera cut、teleport、projection／extent／surface generation変更、missing／stale historyでは全candidateをvisible側へ倒す。余分にvisibleなitemはperformance metricへ記録できるが、誤って隠したitemをtoleranceで合格させない。

Target／Profileのqualified primary execution pathはexact一件、未選択なら0件とする。CPU pathはsemantic oracle、または別Qualificationされた明示的fallbackになり得るが、旧／新経路の互換layer、silent fallback、runtime auto-tuning、同frameの二重drawを意味しない。fallbackはProfileにexact ref、発動条件、意味同等Receipt、切替generationを持つ場合だけ次Graph generationで選択する。benchmark candidateは非dispatchableである。

### 7.1 Virtualized geometryのtarget execution境界

[Virtualized／Continuous Geometry](virtualized-continuous-geometry.md)がactiveなTargetでは、実行順を`conservative View candidate -> outer LOD representation -> virtual familyのresident hierarchy cut -> occlusion／compaction -> raster`に固定する。Render Graphはexact `VirtualGeometryTargetActivationBindingRefV1`／`VirtualGeometryResolutionPlanRefV1`、同じArtifact generationのimmutable `VirtualGeometryResidencySnapshotRefV1`、`RenderViewV1`から投影済みの`ViewLodContextV1`だけを受ける。Activation Binding、Plan、Snapshot、ViewのTarget／generationが一致しなければGraphを作らない。Source Intent、World Cell、provider option、raw I/O queueを解釈しない。

inner cutはRender Viewごとに解決し、ancestor／descendant排他、surface coverage、resident-only、error bound、Material／deformation interface、candidate／visible capacityを検証する。結果はbounded `VirtualGeometryViewCutSummaryV1`へ投影できるが、`LodTierRefV1`、World state、Simulation input、Save／Replayへ書き戻さない。page requestはowner-qualified hintだけを返し、I/O順、eviction、pool budgetをRendererが決めない。

root unavailable、generation mismatch、Material／deformation incompatibility、capacity超過、provider faultをsubmission前に検出した場合はそのGraph generationを不成立にし、同frameへpartial command、CPU再実行、default Materialまたはfallback passを挿入しない。統合Planにapproved exact discrete fallbackがあれば次のGraph generationで選び、なければRenderer faultとする。virtual pathからCPU directへ戻すだけではgeometry artifactの意味同等fallbackにならないため、Render execution fallbackとrepresentation fallbackを別Diagnosticで扱う。Capabilityが`planning_only`のcurrent Graphへvirtual pass、resource、pipeline、feature flagを登録しない。

## 8. Material、Lighting、Post Processとの実行境界

[Materials](materials.md)がMaterial Domain、Shading Model意味、visual style、Material variant keyを所有し、[Project Shader](project-shader.md)がProject Module／Techniqueのsemantic interface、Source profile、Fact Graph、Understanding Closureを所有する。Rendererはoffline生成済みshader artifact、Technique Manifest、Pipeline Interfaceを検証し、runtime source compileや未登録fallback shaderを行わない。

MCD生成`ShaderInterface`はbinding、constant layout、vertex input、attachment expectationを持ち、Pipeline keyはShader Package、render state、vertex layout、attachment format、sample count、Target featureのcanonical hashとする。

[Lighting](lighting.md)はlight type、photometric quantity、color／temperature、shape、range、shadow intentを所有する。RendererはTarget Capabilityに応じたselection、cluster／tile assignment、shadow pass、lighting passをGraph executionとして所有するが、lightの物理値を別単位へ黙って補正しない。

lighting pipeline profileは本書所有のclosed IDであり、`forward_plus_v1 | hybrid_deferred_v1`に固定する。`forward_plus_v1`はGBufferを持たないclustered forward executionで、§9のMSAA Qualified attachment／pipelineと[Materials](materials.md) §4.2のDecal C1 receiverはこのprofileを前提とする。`hybrid_deferred_v1`は`forward_plus_v1`を基底に一部receiverをGBuffer passで実行するoptional profileで、GBuffer attachmentへMSAAを適用しない（§9）。profileは`ResolvedRendererProfileV1`（§12）がTarget Profile、Quality、Capability Signatureから一意に解決し、Target別の既定と品質Tier対応は各Target Profile文書が本IDを参照して固定する。profileをframe途中で切り替えず、変更はSettings Apply／Loading境界のGraph再構築とする。

[Post Processing](post-processing.md)はvolume resolve、effect order、parameter compositionを所有する。RendererはPlanのEffect Catalog IDまたはQualification済みProject Shader Technique IDからresource requirement、history lease、AA接続、surface compositeを検証し、Plan本文からraw pass、Shader Source、native resource、未承認順序変更を受けない。

### 8.1 Outline intent resolution

`OutlineStyleProfileV1`は[Materials](materials.md)が所有するstyle intentであり、Rendererはそのfieldをpass、resource、geometry expansion、screen-space sampling APIへ直接公開しない。`OutlineTechniqueCatalogV1`のEngine-ownedかつTarget-qualified entryだけを候補にし、Project Shader Techniqueを含む候補も同じQualification boundaryを通す。

```text
OutlineQualifiedTechniqueRefV1
  technique_id: StableId
  technique_version: positive uint32
  technique_content_hash: SHA-256
  qualification_binding_ref:
    exact {binding_id, binding_version, binding_content_hash}

OutlineTechniqueCatalogEntryRefV1
  entry_id: StableId
  entry_version: positive uint32
  entry_content_hash: SHA-256

OutlineTechniqueCatalogEntryV1
  entry_id: StableId
  entry_version: positive uint32
  qualified_technique_ref: exact OutlineQualifiedTechniqueRefV1
  target_profile_ref:
    exact TargetProfileRefV1
  resolved_technique_kind: geometry | screen_space | hybrid
  compatible_world_space: WorldSpaceCompatibilityV1
  required_input_kinds[0..3]: unique subset of {depth, normal, coverage}
  supported_temporal_policies[1..2]:
    unique subset of {no_history, stable_history_required}
  allows_msaa: bool
  allows_spatial_aa: bool
  allows_temporal_aa: bool
  required_capability_refs[0..16]: McdContractRefV1(kind=capability)
  entry_content_hash: SHA-256

OutlineTechniqueCatalogV1
  catalog_id: StableId
  catalog_version: positive uint32
  entries[1..256]: OutlineTechniqueCatalogEntryV1
  catalog_content_hash: SHA-256

OutlineInputAvailabilityV1
  view_family_id: StableId
  graph_generation: positive uint64
  available_input_kinds[0..3]: unique subset of {depth, normal, coverage}
  history_available: bool
  availability_content_hash: SHA-256

ResolvedAntiAliasingPlanRefV1
  view_family_id: StableId
  plan_hash: SHA-256

OutlineResolutionDiagnosticV1
  code:
    profile_invalid | world_incompatible | target_unsupported |
    input_unavailable | aa_incompatible | budget_exceeded |
    fallback_cycle | no_qualified_candidate | stale_input
  source_outline_style_profile_ref: exact OutlineStyleProfileRefV1
  world_space_profile_ref: exact WorldSpaceProfileRefV1
  target_profile_ref:
    exact TargetProfileRefV1
  rejected_catalog_entry_refs[0..256]: OutlineTechniqueCatalogEntryRefV1
  remediation_capability_refs[0..16]: McdContractRefV1(kind=capability)

resolve_outline(
  OutlineStyleProfileV1,
  WorldSpaceProfileRefV1,
  TargetCapabilitySnapshotV1,
  ResolvedAntiAliasingPlanV1,
  OutlineTechniqueCatalogV1,
  OutlineInputAvailabilityV1,
  RendererBudgetEnvelopeV1
) -> ResolvedOutlineExecutionPlanV1 | OutlineResolutionDiagnosticV1

ResolvedOutlineExecutionPlanV1
  source_outline_style_profile_ref: OutlineStyleProfileRefV1
  world_space_profile_ref: exact WorldSpaceProfileRefV1
  target_profile_ref:
    exact TargetProfileRefV1
  qualified_technique_ref: OutlineQualifiedTechniqueRefV1 | null
  source_catalog_entry_ref: OutlineTechniqueCatalogEntryRefV1 | null
  outline_technique_catalog_hash: SHA-256
  resolved_technique_kind: geometry | screen_space | hybrid | disabled
  required_input_kinds[0..3]: unique subset of {depth, normal, coverage}
  aa_plan_ref: exact ResolvedAntiAliasingPlanRefV1
  fallback_trace[]
  predicted_cost
  qualification_fixture_refs[]
  plan_hash: SHA-256
```

`OutlineTechniqueCatalogV1`はEngine-ownedかつQualificationから生成するread-only Catalogであり、Project／AIはentry、Technique、Capability、costを直接追加・変更しない。entryはcontent hash順にstrict sortし、duplicate／stale Qualification Bindingを拒否する。entryのQualification Binding subject Targetはentryの`target_profile_ref`とbyte equalityでなければならない。entry、catalog、availability hashはそれぞれASCII `MIRAKAN_OUTLINE_TECHNIQUE_CATALOG_ENTRY_V1`、`MIRAKAN_OUTLINE_TECHNIQUE_CATALOG_V1`、`MIRAKAN_OUTLINE_INPUT_AVAILABILITY_V1`と自己Fieldを除くlength-framed canonical bytesをSHA-256する。`RendererBudgetEnvelopeV1`は[Runtime performance／capacity](../04-runtime/performance-capacity.md)が公開するread-only revisioned projectionである。

ResolverはProfileのcross-field validationを先に行い、exact World ProfileがProfileとCatalog entryの両`compatible_world_space`を満たすこと、Target、depth／normal／coverage availability、AA／history compatibility、budget、Capability、strict fallback priorityの順で評価する。outputの`aa_plan_ref`は入力`ResolvedAntiAliasingPlanV1`とbyte equalityで、non-disabled outputのCatalog entry／qualified Techniqueはnon-null、`source_catalog_entry_ref`が解決するentryの`qualified_technique_ref`と`target_profile_ref`はPlanの同Fieldとbyte equality、entryのkind／required inputはPlanの同Fieldとexact equalityでなければならない。Catalog entryのrequired inputは`OutlineInputAvailabilityV1.available_input_kinds[]`のsubset、Profileの`temporal_policy`はentryの`supported_temporal_policies[]`に含まれ、history必須時は`history_available=true`、MSAA／spatial／temporal AAは各entryのallow flagを満たさなければならない。`geometry_only`、`screen_space_only`、`hybrid_qualified`、`disabled`の意味を変更する候補、未qualified Technique、missing required input、AA不整合、budget超過、cycleまたは候補なしのfallbackを拒否する。failureは対応する`OutlineResolutionDiagnosticV1.code`で返す。fallbackはProfileの順序を守り、最初のcompatibleかつTarget-qualified profileだけを選び、silent style substitutionをしない。`disabled`は`resolved_technique_kind=disabled`、canonical nullの`source_catalog_entry_ref`／`qualified_technique_ref`、empty `required_input_kinds[]`だけへ解決でき、隠れたoutline passを作らない。

`ResolvedOutlineExecutionPlanV1`はlogical execution projectionであってSourceではない。`outline_technique_catalog_hash`は入力Catalogのcontent hashとbyte equalityにする。Graph Compilerだけが`qualified_technique_ref`をCooked Pass Templateとresource relationへ展開し、native geometry expansion、screen-space implementation、Backend object、descriptorをProject／AI APIへ出さない。source profile、Catalog、World／Target／AA／Capability／budget／Qualificationのいずれかが変わればPlanをstaleとして再解決する。

## 9. Anti-aliasingとtemporal execution

`RendererProfileResolver`は`AntiAliasingIntentV1`をViewFamilyごとの`ResolvedAntiAliasingPlanV1`へ解決する。同じViewFamilyのCameraはraster sample count、temporal Provider、jitter、render／display extentを共有し、異なる方式は別ViewFamily／render targetとする。

| Field | 規則 |
|---|---|
| `source_intent_id`／`source_revision` | 解決元Intent、stale Plan拒否 |
| `plan_hash` | self-excluding canonical resolved Plan hash。`ResolvedAntiAliasingPlanRefV1`は`view_family_id`とこのhashへexactにbindする |
| `view_family_id`／`scope_resolution` | Project／Camera Profileから最終scopeへ解決した結果 |
| `raster_samples` | `1 \| 2 \| 4 \| 8`。2以上はForward+のQualified attachment／pipelineだけ |
| `spatial_method` | `off \| fxaa \| smaa_1x`。一つだけ |
| `temporal_method` | `off`またはexact `upscaler_profile_ref`。後者は`upscaler-profile.mirakan-taa \| upscaler-profile.mirakan-taau`またはQualification済みCatalogのexact Profile IDの一つだけ |
| `render_extent`／`display_extent` | Target／Provider制限内。UI／pixel-locked extentを含めない |
| `jitter_policy` | Engine-owned closed ID。AI／Project dataはsample列を保持しない |
| `history_reset_mask` | camera cut、teleport、extent、surface、projection、Provider、model、AA方式変更のclosed bit |
| `excluded_layers` | `ui \| text \| pixel_locked`を必須集合とする |
| `required_capabilities` | Backend、renderer、sample count、motion／depth／mask、Provider、HDR Capability ID |
| `predicted_cost` | pass GPU、bandwidth、persistent／transient byte、Pipeline variant数 |
| `fallback_chain` | 順序、意味差、User通知、必要な再構築境界 |
| `decision_trace` | 選択理由、却下method、Constraint／Receipt／baseline ID |

方式互換を次のclosed tableに固定する。

| Method | Qualification class | Renderer／入力 | 主用途と制限 |
|---|---|---|---|
| `none` | diagnostic | 全Raster | bit-exact検査、AA対象外layer、User明示だけ |
| `fxaa` | portable | 全Raster、resolved color | spatial fallback。Tone map後、UI合成前 |
| `upscaler-profile.mirakan-taa` | portable 3D | motion、depth、exposure、jitter、history | native extent temporal AA。Pixel／UI／VR low-latencyでは候補外 |
| `msaa_2x`／`msaa_4x` | portable Forward+ | sample可能color／depth、全PSO sample一致 | Geometry edgeとalpha-to-coverage |
| `smaa_1x` | optional | resolved color | Temporal禁止時のspatial候補 |
| `msaa_8x` | optional | Forward+、High／offline実機Gate | 自動選択禁止 |
| `upscaler-profile.mirakan-taau` | optional | `TemporalFrameInputV1` | Engine基準temporal upscale |
| Qualified DLSS／XeSS／FSR／MetalFX | optional | Provider別Input／署名／license／driver | Providerごとに独立Qualification |

`raster_samples > 1`と`temporal_method != off`を同時使用しない。MSAAとFXAA／SMAAの併用は既定禁止、Hybrid Deferred GBufferのMSAAは禁止、MSAA 8xはLow／Medium／MobileのAuto候補外とする。Alpha-to-coverageをtransparent／texture／specular alias対策と表示しない。Temporalはpre-tonemap scene-linear HDR、FXAA／SMAAはTone map後かつUI composite前に実行する。Dynamic resolution、camera cut、teleport、projection／surface／方式／Provider変更ではhistoryを破棄し、MSAA sample変更はSettings Apply／Loading境界でattachment／Pipeline keyを再構築する。

Resolverは`UnsupportedByRenderer | UnsupportedByTarget | InvalidCombination | MissingMotionVectors | MissingTemporalInput | PixelLockedTemporalForbidden | MsaaSampleUnsupported | ProviderNotQualified | BudgetExceeded | ScopeConflict | RebuildBoundaryRequired`を`AntiAliasingResolutionErrorV1`のclosed codeとして返す。未知方式を近似せず、候補、拒否理由、必要Capability／Receipt、意味差をremediationに含める。

`camera_profile`のScopeはCamera単独のnative surfaceを変更せずViewFamilyのPlan候補として解決する。SR provider IDの`upscaler-profile.directsr`／`upscaler-profile.metalfx`はそれぞれ公開Target／input契約に一致する場合だけ選択する。

Temporal historyはViewFamily、algorithm／provider generation、surface generation、extent、projectionへ束縛する。camera cut、teleport、generation／extent／projection／AA方式変更、missing motion inputでは破棄する。Generated frameはauthoritative simulation／render snapshotではなくpresentation outputとして区別し、real frameのmetricへ混ぜない。

`TemporalFrameInputV1`とそのmotion／history execution carrierは本書所有のclosed schemaであり、次を持つ。

```text
RenderResourceGenerationRefV1
  resource_handle:
    { index32: u32, generation32: u32 }
  backend_device_generation: positive u64
  logical_resource_descriptor_hash: SHA-256

TemporalMotionVectorInputV1
  motion_input_id: StableId
  motion_input_version: 1
  view_family_ref: exact ViewFamilyRefV1
  frame_id: positive u64
  present_id: positive u64
  history_reset: bool
  previous_render_view_refs[0..64]:
    sorted unique exact RenderViewRefV1
  current_render_view_refs[1..64]:
    sorted unique exact RenderViewRefV1
  motion_resource_ref: exact RenderResourceGenerationRefV1
  validity_mask_resource_ref:
    null | exact RenderResourceGenerationRefV1
  encoding: screen_space_pixels_per_real_frame_xy_right_down
  includes_camera_motion: true
  includes_object_motion: true
  excludes_projection_jitter: true
  source_snapshot_sequence: positive u64
  previous_source_snapshot_sequence: null | positive u64
  motion_input_content_hash: SHA-256

TemporalMotionVectorInputRefV1
  motion_input_id: StableId
  motion_input_version: 1
  motion_input_content_hash: SHA-256

TemporalExposureSampleV1
  exposure_sample_id: StableId
  exposure_sample_version: 1
  exposure_history_key: StableId
  frame_id: positive u64
  ev100: finite f64
  linear_exposure_multiplier: finite positive f64
  exposure_sample_content_hash: SHA-256

TemporalExposureSampleRefV1
  exposure_sample_id: StableId
  exposure_sample_version: 1
  exposure_sample_content_hash: SHA-256

TemporalHistoryLeaseRefV1
  history_key: StableId
  history_generation: positive u64
  algorithm_or_provider_profile_ref:
    exact McdContractRefV1(kind=profile)
  surface_id: StableId
  surface_generation: positive u64
  surface_content_hash: SHA-256
  render_extent: positive u32 width, positive u32 height
  display_extent: positive u32 width, positive u32 height
  history_content_hash: SHA-256

TemporalFrameInputV1
  schema_version: 1
  view_family_ref: exact ViewFamilyRefV1
  frame_id: positive u64
  present_id: positive u64
  render_extent: positive u32 width, positive u32 height
  display_extent: positive u32 width, positive u32 height
  jitter_offset_px: finite Vec2
  motion_vector_ref: exact TemporalMotionVectorInputRefV1
  depth_ref: exact RenderResourceGenerationRefV1
  exposure_ref: exact TemporalExposureSampleRefV1
  history_ref: exact TemporalHistoryLeaseRefV1
  history_reset_mask: closed ResolvedAntiAliasingPlanV1 reset bits
  temporal_input_content_hash: SHA-256
```

`TemporalMotionVectorInputV1`はRender Graphだけが、同じView Familyのcurrent／previous unjittered View／Projection、`VisibilityInstanceV1`のcurrent／previous presentation transform、[Animation](../05-simulation/animation.md)の同じSource snapshot lineageに束縛されたcurrent／previous poseから生成する。値はcurrent render extent上のpixel単位、real frameあたり、+X右／+Y下で、projection jitterを含めない。Camera motionとobject motionを合成し、skinned／procedural objectのprevious presentation stateを欠くpixelはzero motionに偽装せず`validity_mask_resource_ref`でinvalidにする。initial V1で非対象のMorphを暗黙に処理せず、将来のMorph Capabilityは同じprevious-state契約をend-to-endで満たすまでtemporal inputへ参加させない。`validity_mask_resource_ref=null`は全pixel validの時だけ許す。Engine基準temporal methodはinvalid pixelの旧history weightを0にしてcurrent sampleからseedし、Provider Profileが完全motion validityを要求する場合だけ一件でもinvalidなら`MissingMotionVectors | MissingTemporalInput`でPlanを不成立にする。Gameplay velocity、Physics velocity、generated frame、前回のmotion textureまたはBackend推定から補完しない。

`history_reset=false`ではprevious／current Viewを同じView ID集合とし、current集合を`view_family_ref`のView集合とexact set equality、previous集合を各current Viewの直前generationへexact解決し、previous／current source snapshot sequenceも連続させる。`history_reset=true`ではprevious View集合をempty、previous source sequenceを`null`、validity maskを全pixel invalidにし、current sampleだけでhistoryをseedする。camera cut、teleport、World／surface／extent／projection generation変更、source snapshot非連続またはprevious state欠落ではこのreset branchだけを使い、旧motion／historyを再利用しない。`motion_input_content_hash`、`exposure_sample_content_hash`、`temporal_input_content_hash`はそれぞれ型名のASCII domain separatorと自己hashを除くclosed MCD canonical bytesから計算し、Refは完成recordへexact解決する。

`frame_id`／`present_id`はreal frameごとに単調増加し、generated frameへ新しいsimulation identityを割り当てない。`jitter_offset_px`は`jitter_policy`から導出したEngine-owned値、`history_ref.history_key`は`ViewFamilyV1.exposure_history_key`ではなくAA Planが所有するtemporal history keyへ束縛し、surface identity／generation／hashはFamilyの`surface_generation_ref`解決先とbyte equalityにする。`exposure_ref.exposure_history_key`はFamilyとbyte equality、`history_reset_mask`は`ResolvedAntiAliasingPlanV1`と同じclosed bitである。`RenderResourceGenerationRefV1`は§3のEngine handle、Backend device generation、logical descriptorを一体で検証し、native handle、descriptor index、Provider内部structを公開しない。これらもtarget contractであり、resource、motion producer、history leaseまたはSchemaの実装済み主張ではない。

Providerはprivate Adapterとして統合し、exact version、hash、license、取得元、build optionは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)だけが固定する。未署名artifact、runtime download、runtime training、無承認更新は本書で別規則を複写せず[AI Security／Approval](../01-governance/ai-security-approval.md)とToolchain ownerへ委譲する。

## 10. Ray、path、neural execution capability

Ray query、ray／path transport、neural reconstructionは同じRender Graph resource／queue contractへ従うoptional execution capabilityであり、別Rendererを形成しない。ただしGI、reflection、shadow visibility、reference transportのsemantic channel、Technique ID、reference／preview／runtime role、representation requirement、denoise intent、fallback ladderは[Advanced Light Transport](advanced-light-transport.md)だけが選択する。本書は`ResolvedLightTransportPlanV1`／`LightTransportExecutionRequestV1`を解釈し直さずGraphへ展開する。

本書は`RayTracingPortV1`、`RadianceCachePortV1`、`NeuralRenderModelV1`と、acceleration build／update、logical ray dispatch、scratch、history resource、queue／barrier／submissionを所有する。ALT Profileの`software_ray | hardware_ray | path_traced | hybrid`等をRender Graph profile名へ複写せず、Provider／Backend capabilityからsemantic channelを推測しない。

Capability unavailable、history resource invalid、Provider／device fault時は同一frameへ未宣言passを差し込まない。Rendererはtyped execution failureをALTへ返し、ALTのregistered fallbackが新しいPlanを作った場合だけ次のGraph generationで切り替える。Rendererが独自にraster、screen-space、non-neural、channel disabledへ降格しない。

Acceleration structure、model weight、scratch resourceはgeneration付きArtifact／resourceとして扱う。Project C++やAIへnative acceleration handle、arbitrary operator、network accessを公開しない。reference、preview、runtime candidateのTarget QualificationとProduct claimはALTと[Product Plan](../00-product/product-plan.md)が所有し、Graph実行成功だけから昇格しない。

## 11. Planned action vocabulary、diagnostic、fallback

View／Renderer intent、debug capture要求、qualification run要求はStable IDでないplanned semantic action vocabularyであり、本書のcurrent MCD Operation集合は空である。AAのexact五IDだけは[Executable contracts](../02-foundation/executable-contracts.md#20-ai向けdiscoveryexecution候補のplanning-record未activation)の`planning.operation_family.rendering_aa_discovery@1`に属し、current MCD／Owner Manifest／Service allowlist／Provider／MCP／alias集合は`[]`、Capability stateは`not_activated`である。各ownerのfuture atomic activation work itemが完全なMCD／Service／Policy／Validator／Diagnostic／Receipt／publication closureを登録するまでEditor／AIへdispatchせず、action名からIDを生成しない。Activation後の共通envelope、preview projection、approval classはFoundationとGovernanceの正本を参照し、本書では再定義しない。

Renderer固有diagnosticはGraph／pass／resource／ViewFamily／surface generation、Backend-neutral error code、first failing dependency、fallback dispositionを含む。少なくともgraph invalid、resource exhausted、pipeline unavailable、history invalid、surface lost、device fault、provider unavailableを区別する。native result codeやdriver messageはprivate attachmentとして保存し、stable diagnostic codeにしない。

optimizationのAI／Editor説明は[Runtime performance／capacity §8.4](../04-runtime/performance-capacity.md#84-algorithm-optimization-candidate-qualification)の`OptimizationDecisionProjectionV1`をread-onlyで消費する。AIはnative command／barrier、resource alias、visibility buffer、GPU address、candidate selection、threshold、Receiptを変更せず、raw capture／full traceを直接解釈してselectionを補完しない。Optimization propose／select Operationは現在登録せず、§11のcurrent MCD Operation集合を増やさない。

[Architecture Governance §5.3](../01-governance/architecture-governance.md#53-architecture-explain-projection)の`ArchitectureExplainProjectionV1`は、`RenderGraphDefinition -> CanonicalRenderExecutionPlanV1 -> GraphicsDevicePort -> Backend Adapter -> Target graphics API`のOwner／consumer関係、文書状態、実装状態をread-onlyに説明できなければならない。これはnative object、command stream、driver identity、Project write権限を公開する経路ではない。ProjectionとGeneratorが未materializeの現在は、本書とexact Owner linkがreview用正本であり、AI対応済みまたはtool dispatch可能とは表現しない。

`RendererProviderErrorV1`は`NotInstalled | UnsupportedDevice | UnsupportedDriver | SignatureInvalid | LicenseNotApproved | VersionMismatch | MissingInput | InvalidFormat | InitializationFailed | ExecutionFailed | HistoryInvalid | SwapchainConflict | BudgetExceeded | DeviceFault`のclosed codeを持つ。AAの互換／排他／scope失敗は`AntiAliasingResolutionErrorV1`を使い、Provider障害と混同しない。Running中のProvider failureは同frameで別Providerへ差し替えずgenerationを停止し、次のLoading境界でContextを再生成する。Ray／path／neural failureはALTへtyped failureを返し、同Ownerのfallback解決を経ないGraph変更を行わない。

Renderer-owned Quality fallbackはresolution、optional Renderer effect、AA／temporal execution等の本書が所有する順序付き候補だけから選ぶ。GI／reflection／shadow／reference transport、light-transport denoiseのfallbackは[Advanced Light Transport](advanced-light-transport.md)、Post effectのfallbackは[Post Processing](post-processing.md)へ委譲する。allocation失敗時のsilent quality reduction、draw skip、default material置換、別Domain Techniqueへの切替を禁止する。共通backpressureとcapacity判定は[Runtime performance／capacity](../04-runtime/performance-capacity.md)へ従う。

## 12. 関連契約の配置

Rendererは[Lighting](lighting.md)所有の`LightIntentV1`／`LightingStyleProfileV1`／`ResolvedLightPlanV1`、[Advanced Light Transport](advanced-light-transport.md)所有の`ResolvedLightTransportPlanV1`／`ResolvedShadowPlanV1`／`LightTransportExecutionRequestV1`、[Post Processing](post-processing.md)所有の`PostProcessIntentV1`／`PostProcessProfileV1`／`ResolvedPostProcessPlanV1`、[LOD](lod.md)所有の`LodIntentV1`／`LodResolutionPlanV1`／`ViewLodContextV1`を解釈し直さず実行する。UI primitiveの`MirakanUiDrawPacketV1`は[Editor UI Framework](../03-authoring/editor-ui-framework.md)、`RuntimeRepresentationPlanV1`は[Runtime performance／capacity](../04-runtime/performance-capacity.md)、共通`RemediationV1`は[Executable contracts](../02-foundation/executable-contracts.md)、Provider lockの`RendererProviderLockV1`は[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)が正本である。

`RenderRepresentationPlanV1`はRuntime planからCookされ、各Source集合を`individual | instanced | spatial_batch | presentation_batch`の一つに分類し、Source Stable ID集合、plan-local Cell、mobility／interaction、geometry key、[Materials](materials.md)所有のexact `MaterialBatchCompatibilityKeyV1`、Domain LOD plan、HLOD chain、bounds、resident／visible上限、Target fallback、visual-equivalence hashを持つ。`VisibilityInstanceV1`はgeometry generation、current／previous transform、bounds、量子化済みerror／threshold、previous presentation tier、material packet、layer、stable render IDだけを持つSoAで、Entity／Component pointer／Gameplay tag／Simulation tierを含めない。

LOD representation tierとRenderer cook classの対応を次に固定する。Rendererはこの対応で再分類するだけで、LOD thresholdまたはtierを再選択しない。

| LOD `RepresentationLodProfileV1` tier | `RenderRepresentationPlanV1` class／処理 |
|---|---|
| `individual` | `individual` |
| `instanced` | Source identity集合とexact geometry／material keyが適格なら`instanced`、不適格ならPlan compile拒否 |
| `spatial_proxy` | exact World cell／representation slotとHLOD Artifactがある場合だけ`spatial_batch` |
| `impostor` | exact proxy artifactとMaterial interfaceがある場合だけ`spatial_batch` |
| `hidden_presentation` | 対象Viewの`ResolvedRepresentationSet`からcanonical omission。Source identity、Entity、Save、Simulationは削除しない |

`presentation_batch`はVFX／sprite等の選択済みPresentation outputをRendererが実行都合でbatch化するclassで、LOD tier名ではない。`spatial_proxy | impostor`をartifact欠損時に`individual`へ暗黙変換せず、LOD Planのexact fallbackへ戻す。`hidden_presentation`をvisibility culling result、World unload、Entity deletionとして記録しない。

同じ`MaterialBatchCompatibilityKeyV1`はbatchの必要条件であって十分条件ではない。Rendererはexact geometry generation、vertex／Pass interface、LOD、View、layer、sort class、Target Profileも一致するときだけgroup化し、異なるMaterial keyを同じbatchへ入れない。Material parameter名、値、texture handleを再評価せず、Materialsが宣言したper-render-instance layoutのpayloadだけを参照する。individual／instancedのどちらでもSource Stable ID集合とstable render IDを保ち、batch化をGameplay identityの統合として扱わない。上限超過、stale artifact generation、layout不一致ではgeneration全体をtyped rejectし、silent dropや同一frameの別Material fallbackを行わない。

2Dは`Renderer2DExecutionPlanV1`により`SpriteRendererComponentV1`または`TileChunkArtifactV1`からAsset version、bounds、layer／order／Y-sort、material instance、atlas page、mask／blend、Stable rendering IDを持つpacketを抽出する。Source rect／Tile ID配列、texture handle、native descriptor indexをSnapshotへcopyしない。

### 12.1 2D authority chain

`SpriteRendererComponentV1`は本書がtarget意味を所有するRender-presentation入力であり、sprite Asset revision／Stable Sprite ID、transform binding、color／visibility、layer／order／Y-sort intent、material／mask／blend、Stable rendering IDだけを表す。ECSのstorage／structural transactionは[Runtime ECS](../04-runtime/entity-component-system.md)、Gameplay stateはGameplay Ownerが所有する。current Component Schema、Registry entry、Runtime instanceは存在せず、exact Definition Closureなしに型名からfieldまたはactive Componentを推測しない。

`TileChunkArtifactV1`は[World §10.1](world.md#101-tilemap-sourcecookpublication)のTilemap SourceからCookされるWorld-owned Derived Artifactであり、tile layer、region、Source revision、sprite参照、boundsとpresentation payloadを持つ。RendererはTile ID配列、World Cell membership、Collision shape、Navigation gridを所有せず、Collision／Navigationは同じTilemap Source revisionから独立projectionをCookする。

| 境界 | 一意Owner | Rendererが消費／生成するもの |
|---|---|---|
| Sprite／Atlas Source、import、Cook、revision | [Asset Lifecycle](../03-authoring/asset-lifecycle.md) | exact Sprite／Atlas Artifact refとgeneration |
| Sprite frame／property selection | [Animation](../05-simulation/animation.md) | sealed presentation selection |
| Tilemap Source、chunk、World Cell membership | [World](world.md) | `TileChunkArtifactV1`とbounds |
| Runtime request、generation、lease、residency | [Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md) | ready generationのlease。I/O／evictionは決めない |
| 2D Material／mask／blend compatibility | [Materials](materials.md) | resolved material packetとbatch compatibility key |
| pixel profile、view、projection、subpixel policy | [Camera](camera.md) | exact Render View |
| collider／navigation projection | [Collision](../05-simulation/collision.md)／[Navigation](../05-simulation/navigation.md) | Renderer packetへ混入しない |
| packet抽出、culling、stable sorting、batching、submission | 本書 | `Renderer2DExecutionPlanV1`とimmutable packet |

canonical sort keyはView、composition layer、owner-supplied order、quantized Y-sort value、material／mask ordering class、Stable rendering IDの順を本書が定義し、同値時もStable IDで全順序にする。World、Gameplay、Animation、Materialはsort intentの値を供給できるが、draw orderやbatch orderを再計算しない。batchingはsort順を変えない連続範囲だけに許可し、同じAtlas pageまたはMaterialであることを理由にlayer／mask boundaryを越えて並べ替えない。

2D Qualificationは少なくとも、isometric Y-sortのtie、負座標／subpixel camera、pixel-locked camera、Tile chunk replacementと旧新Asset generation、Atlas hot reload、複数Camera、World 2DとUIのcomposition、mask／blend／2D light boundary、Animation frame切替、resident generation失効、batch上限を含む。同じ入力でpacket集合とsort orderが一致し、stale generation、missing Sprite、Tile Source revision差、batchによる順序変更をtyped failureとして検出するまでcurrent化しない。

`RendererCapabilitySignatureV1`はBackend、API／shader version、GPU／driver identity、feature bit、memory budget、display mode、SDK／model generation、signed artifact hashを持つ。`RendererCapabilityProjectionV1`はそのAuthoring向けredacted projectionであり、Adapter LUID、driver文字列等のnative識別子をEngine build固定のfield maskで除外した集合だけを公開する。Authoring／AI経路はSignature本体を直接読まずこのprojectionを消費する。`ResolvedRendererProfileV1`はProject要求、Target Profile、そのSignature、Qualification Receiptから一意に解決するroot外Derived projectionで、承認済みfallback順を持つ。Receipt subjectは先に固定したCapability Signature／Target artifact closureだけであり、Resolved Profile／Plan hashを含めない。`RendererOptimizationReceiptV1`は同一input trace／Profile／driver／SDKのBefore／After、capture、visual diffを結ぶ。

`RendererCapabilitySignatureV1`は`WindowsCapabilitySignatureV1`または`MobileCapabilitySignatureV1`の別名、継承型、unionではない。Renderer ownerがexact `platform_capability_signature_ref`、`target_profile_ref`、`toolchain_profile_ref`、`backend_adapter_id`、`renderer_artifact_refs[]`をsource bindingとして保持し、次の規則で決定論的に生成する。

| Source | Renderer署名への写像 |
|---|---|
| Target Profile | 許可Backend、既定Renderer profile、Target identityをexact refで束縛 |
| Platform Capability Signature | GPU／driver／graphics feature／shader capability／memory budget／display capabilityの正規化済み観測だけを写像 |
| Toolchain／Provider lock | API／shader version、SDK／model generation、Backend artifact requirementを写像 |
| signed Renderer／Shader artifact manifest | content hashとsignature statusをexact artifact refとして写像 |
| OS／CPU／input／audio／thermal／privacy field | Renderer選択に明示的に必要な正規化fieldを除き署名へ複写しない |

Source ref、source content hash、Backend Adapter generationのいずれかが変われば旧Renderer署名、`RendererCapabilityProjectionV1`、`ResolvedRendererProfileV1`をstaleにする。Qualification Receiptは完成したRenderer署名へ束縛するdownstream evidenceであり、署名生成の入力へ戻さない。Platform ownerはRenderer field setを再定義せず、Renderer ownerはPlatform署名に存在しない観測値を推測しない。

Shadow authoringの`ShadowIntentV1`／`ShadowStyleProfileV1`は[Lighting](lighting.md)、semantic `ShadowGraphV1`、Technique／fallback selection、`ResolvedShadowPlanV1`は[Advanced Light Transport](advanced-light-transport.md)が所有する。`ProjectShadowTechniqueV1`は[Project Shader](project-shader.md)の`ProjectShaderTechniqueV1` specializationで、ALTがTarget／channel／Qualificationを解決する。Rendererは完成した`execution_request_refs[]`をregistered Pass Templateへ展開し、cycle、hazard、resource lifetime、alias、queue、memoryを検証するが、Shadow quality、raster／compute／ray／mixed selection、meaning fallbackを再決定しない。

`ShadowTechniquePortV1`はProject Shaderの`ProjectShaderTechniquePortV1`にある`port_id=shadow` entryをexact reuseし、同名Schemaを作らない。必須出力`shadow_attenuation_linear`、入力semantic、Layer、history、ordering boundaryはPortとALT Planへ閉じる。Plan欠落、Target／generation mismatch、未登録Template、型不一致semantic、上限超過では既定Shadow Graphを生成せずGraph compileをrejectする。

Render Graph compilerは[Project Shader](project-shader.md)が所有するTechnique Manifestを通常Passと同じcycle、hazard、lifetime、alias、queue、memory validationへ通す。Manifest申告とShader Fact Graph、reflection、実行時resource useが一致しないArtifactはpromotionを拒否する。Running中の不一致は汎用`ProjectShaderTechniqueValidationFailed`とDomain projectionを発行し、該当Techniqueのそれ以降のpassとsubmissionを停止する。ShadowではDomain projectionを`ShadowTechniqueValidationFailed`とする。同一frameにfallback passを挿入しない。

PlanがGovernanceで承認されたfallback referenceを持つ場合は、次frameのGraph Instanceからそのfallbackへ決定論的に切り替える。承認済みfallbackがなければRenderer faultへ遷移し、該当Planをretry／resumeしない。承認の成立、scope、署名、期限／失効は[AI Security／Approval](../01-governance/ai-security-approval.md)だけが決め、Rendererはそのexact Governance referenceの検証結果だけを消費する。

`RayTracingPortV1`はacceleration-structure build／update、ray query／dispatch、shader／function table、scratch、compaction、timestampだけを公開する。ALTが要求するradiance cache executionはEngine-owned `RadianceCachePortV1`を介し、native handleをAssetまたはALT Planへ保存しない。`NeuralRenderModelV1`はmodel ID、semantic input／output、architecture version、weight format／SHA-256／provenance、quantization、required feature、scratch／persistent byte、inference cap、fallbackを持ち、runtime download／training／未署名weight／arbitrary operator／network accessを禁止する。モデル選択とlight-transport fallbackはALT、generic reconstruction executionとphysical resourceは本書が所有する。

`RendererVisualReceiptV1`はlinear Rec.709 RGB32F比較、UI／pixel-locked bit-exact mask、3D SSIM／RMSE、NaN／Inf、ghost persistenceとframe／camera／exposure／jitter／extent／Provider／driver／SDK／model hashを保存する。`AntiAliasingVisualReceiptV1`は本書のAA reference／baseline、alias energy、edge spread、shimmer、ghost、`unaddressed_alias_class`を追加するDomain projectionである。

AA metricの算出仕様は`AntiAliasingVisualReceiptV1`のDomain projectionとして本書が正本である。比較bufferと色空間は`RendererVisualReceiptV1`のlinear Rec.709 RGB32F比較を継承する。edge maskは静止Reference（§13の4x SSAA downsample）のrelative luminanceへの3×3 Sobel gradient magnitudeから導出し、導出閾値はEngine buildが固定してfixtureごとにReceiptへ記録する。alias energyはedge mask内のReferenceとの差分二乗和、shimmer energyは静止fixtureの連続real frame間差分二乗和のedge mask内積算とする。edge spreadはedge法線方向でluminance遷移が10%から90%へ達する距離のdisplay pixel長とする。P95の母集団はfixtureごとの全edge mask pixel（shimmerは全比較frame pair）であり、§13の閾値はfixture単位で適用する。同一入力から同一metric値を得るdeterministic実装だけを合格証拠にする。

## 13. Qualification

Qualificationはportable raster referenceを必須とし、次のDomain fixtureを持つ。

algorithm optimization candidateは同じGraph Definition、Render Snapshot、ViewFamily、Target Profile、Capability Signature、driver、Toolchain lock、fixture、input traceで比較し、[Runtime performance／capacity §8.4](../04-runtime/performance-capacity.md#84-algorithm-optimization-candidate-qualification)の`OptimizationDecisionProjectionV1`へread-onlyに投影する。最低metricはCPU／GPU P50／P95／P99、command／indirect argument count、visible-set equality／false occlusion、transient logical／physical peak、alias reuse byte、barrier／validation error、history invalidation数である。Target別に一つのprimaryだけを選び、別Target結果から推測しない。

Backend qualificationは同じlogical conformanceを三Backendへ要求し、API固有の最適化量ではなく意味同等性を先に判定する。

| Backend | Platform source | 必須mapping／evidence |
|---|---|---|
| D3D12 | [Windows](../07-platform/windows.md)の`WindowsCapabilitySignatureV1` | `CanonicalRenderExecutionPlanV1`からEnhanced Barriers、queue、resource、surfaceへ写像し、OPTIONS12 positive／negative、validation、artifact hashを同一Renderer署名へ束縛 |
| Vulkan | [Android](../07-platform/android.md)が生成する[Mobile Common](../07-platform/mobile-common.md)の`MobileCapabilitySignatureV1` | 同じlogical access／barrier／queue ownership／lifetimeを写像し、validation zero-error、offline shader artifact、physical-device resultを同一Renderer署名へ束縛 |
| Metal | [Apple](../07-platform/apple.md)が生成する[Mobile Common](../07-platform/mobile-common.md)の`MobileCapabilitySignatureV1` | 同じlogical resource／hazard／queue／completion／surface lifetimeを写像し、validation、offline library artifact、physical-device resultを同一Renderer署名へ束縛 |

Backend固有render-pass merge、memoryless／tile optimization、native queue統合は、canonical pass／resource／dependencyと観測結果を変えず、Backend conformance Receiptで意味同等性を示す場合だけ許可する。Backend Adapterが不足するlogical featureを別pass、別format、別queueへ黙って置換しない。

- Graph cycle、read-before-write、unordered write、subresource overlap、history invalidationのunit／property test。
- 同一Graph入力からcanonical compile plan hashが一致するdeterminism test。
- D3D12はOPTIONS12のpositive／negative feature queryを検証し、positive DeviceだけでEnhanced Barriersのaccess／barrier conformanceを実行する。negative Deviceはsubmission前に`UnsupportedDevice`となることを検証する。Legacy Barrier laneは持たない。Vulkan、Metalは各APIのaccess／barrier conformanceとvalidation zero-errorを検証する。
- 2D pixel、Realistic、Toon、Pixel Dioramaのgolden image。
- AA Off／FXAA／SMAA 1x／MSAA 2x・4x・8x／Mirakan TAA／TAAUのthin geometry、foliage、alpha scissor／blend、specular、particle、emissive、skinning、急加速、camera cut、dynamic extent、HDR fixture。
- AA scope conflict、TAA＋MSAA、Deferred MSAA、unsupported sample count、missing motion／depth、pixel-locked temporal、Settings Apply外rebuildのnegative test。
- D3D12／Vulkan／MetalでMSAA color／depth sample count、Pipeline key、resolve order、alpha-to-coverage、surface loss後rebuildが一致するBackend conformance。
- resize、alt-tab、HDR／SDR切替、surface loss、device removed fault injection。
- GPU OOM、descriptor exhaustion、pipeline miss、corrupt shader、stale Asset generationとlast-use serial前に破棄されないlifetime test。
- CPU direct／GPU indirect／meshletのvisible result、occlusion history、camera cut、overflow、fallback一致。alias pairごとのlifetime非重複、全queue completion、aliasing barrier、full first-use initialization、HZB false occlusion 0、indirect capacity一件超過、Target別primary exact一件も検査する。
- FOV、projection、resolution、dynamic extent別projected error、CPU／GPU tier、hysteresis、camera cut再選択。
- Cook分類、Gameplay identity保持、mutable objectのstatic batch混入拒否、HLOD Source順序に依存しないArtifact hash、interactive／Physics／Save／animation混入拒否、HLOD on／offのGameplay一致。
- Native TAA／qualified Providerのmotion、depth、exposure、reactive、UI、HDR、dynamic extent、camera cutと、FG Provider一意Swapchain ownership／停止条件。
- Provider署名／hash／missing artifact／unsupported driver／initialization／execution failure／teardown、RT Raster fallback、acceleration structure lifetime、Path convergence／deterministic seed、corrupt／unsigned neural model／non-neural fallback。
- Shadow Graph cycle／上限／unsupported node、未宣言access、interface hash不一致、fallback欠落のnegative testと、各Shadow Planの同一`shadow_attenuation_linear`接続。
- Outlineのgeometry-only／screen-space-only／hybrid-qualified／disabled、Profile／Catalog両方のWorld compatibility、fixed／dynamic resolution、AA off／MSAA／spatial／temporal、depth／normal／coverage欠落、Target／budget fallback、stale profile／input／budget envelope、unqualified technique、fallback cycleのpositive／negative fixture。
- 同じTechnique Manifest／shader reflection／runtime-use trace／Governance referenceから同じ`ShadowTechniqueValidationFailed`を生成し、promotion拒否、該当Plan停止、当該frameへのfallback挿入0件、承認済みfallbackがある場合は次frame切替、ない場合はRenderer faultとなる決定論fixture。
- Project ShaderのRaster／Compute／Ray／mixed Techniqueについて、宣言済みStorage access、Pass DAG、Port入出力、Target fallbackが同じcanonical Graphへcompileされ、Manifest外Pass／Resource／side effect、stale Understanding Closureを拒否するfixture。
- UI／text／pixel-locked layerがdynamic resolution、Temporal Reconstruction、Frame Generationで劣化しないtest。
- AIが未登録Pass、native resource、unsupported Target feature、arbitrary modelを生成できないconformance。

AA visual fixtureは4x linear-resolution SSAA downsampleを静止Reference、AA Offを動的Baselineとする。UI／text／pixel-lockedはbit-exact、NaN／Inf 0 pixel、edge mask内alias energyはOff比20%以上低減、FXAA／SMAA／MSAAのedge spread P95は1.50 display pixel以下、temporal方式は2.00以下、shimmer energy P95はOff比30%以上低減、disocclusion ghostは3 real frameを超えて残らないことを要求する。対象外alias classは失敗でなく明示列挙する。

性能run、visual／replay evidence、receipt envelope、provenance gradingは[Runtime performance／capacity](../04-runtime/performance-capacity.md)と[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を使い、閾値やfieldを複写しない。Renderer固有fixtureはGraph input、expected pass/resource relation、expected output／fallbackだけを所有する。

Release候補はruntime source compile、undeclared resource access、stale generation／Shader Understanding Closure use、critical pipeline miss、device recovery leak、unqualified Provider／Project Technique activationが0件でなければならない。本書はdomain qualification evidenceを出力し、activationと導入順は[Product Plan](../00-product/product-plan.md)が決定する。
