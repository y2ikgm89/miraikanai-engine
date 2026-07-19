# Miraikanai Engine 独自Particle／VFX Platformアーキテクチャ規約

- 文書版: 1.0
- 作成日: 2026-07-20
- 最終更新日: 2026-07-20
- 調査基準日: 2026-07-20
- 対象: 2D／3D Particle、CPU／GPU simulation、VFX Asset、Compiler、Renderer連携、Editor、AI Authoring、Mobile
- 状態: プロジェクト公式の規範設計レビュー版。C1 CPU経路とC2 GPU経路のProduction昇格は実装後のQualification待ち
- 上位文書: [AIネイティブ独自ゲームエンジン 設計計画書](./2026-07-18-ai-native-game-engine-authoring-design.md)
- 機能範囲: [Miraikanai Engine 2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)
- Runtime正本: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- Renderer正本: [Miraikanai Engine Rendering／Render Graphアーキテクチャ規約](./2026-07-19-rendering-render-graph-architecture-design.md)
- Asset正本: [Miraikanai Engine Asset Pipeline／Content Package規約](./2026-07-19-asset-pipeline-content-packaging-design.md)
- Authoring正本: [Miraikanai Engine Authoring Model／Project State規約](./2026-07-19-authoring-model-project-state-design.md)
- Editor規約: [Miraikanai Engine Editor／Workspace／UX規約](./2026-07-19-editor-workspace-ux-design.md)
- Editor UI Framework規約: [Miraikanai Engine 独自Editor UI Framework／Shellアーキテクチャ規約](./2026-07-20-editor-ui-framework-architecture-design.md)
- Collision正本: [Miraikanai Engine Collision／Colliderアーキテクチャ規約](./2026-07-19-collision-collider-architecture-design.md)
- モバイル規約: [Miraikanai Engine モバイルPlatformアーキテクチャ規約](./2026-07-19-mobile-platform-architecture-design.md)
- 実行可能契約: [Miraikanai Engine 実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)
- AIガバナンス: [Miraikanai Engine AI実装・保守ガバナンス規約](./2026-07-19-ai-engine-development-governance-design.md)

## 1. 結論

Miraikanai EngineのParticle／VFXは、**一つの型付きVFX Asset／Graph IRを、2D／3DおよびCPU／GPU向けの専用実行Artifactへoffline compileする独自Platform**として実装する。

- UnityのBuilt-in Particle SystemとVisual Effect Graphのような二つの独立製品へ分けない。
- Godotの`CPU/GPU × 2D/3D`を別々のAuthoring modelとして重複実装しない。
- Unreal Engine Niagaraの任意性をそのまま再現せず、C1／C2ごとのclosed Node Catalog、上限、型、failureを持たせる。
- 初心者向けModule Stack、上級者向けGraph、AI自然言語編集は、すべて同じ`VfxSystemDocumentV1`のProjectionとする。
- C1はWindows／Mobileで成立するCPU simulationを先に実装し、C2でGPU compute、Mesh、Ribbon、Depth／SDF collision、Vector Field、Sub-emitter、Particle Lightを追加する。
- VFXはPresentationであり、Particleの位置、衝突、GPU Event、Lightをauthoritative Gameplay、Damage、Save、Navigation、AI perceptionへ逆入力しない。
- Game／AI／Project C++へD3D12、Vulkan、Metalのresource、shader、counter、barrier、native pointerを公開しない。
- Shipping RuntimeにVFX Graph compiler、shader source、任意code evaluator、JITを含めない。

Gameの実行CodeはC++23のままである。VFX GraphはGame用Script VMではなく、Engine-ownedのclosed型付き表現をoffline compileし、RuntimeではC++ kernelまたは事前Cook済みGPU shaderを実行する。

本書でいう「公式推奨」は、Unity、Epic、Godot、Microsoft、Khronos、AppleがMiraikanai Engine固有の設計を推奨したという意味ではない。各一次資料で確認した実装可能性と制約を基に、本プロジェクトが公式規約として採用した判断を意味する。

## 2. 決定権と境界

| 主題 | 正本 |
|---|---|
| VFX Asset、Emitter、Graph、Node Catalog、VFX IR、CPU／GPU execution plan、VFX Runtime、VFX AI／Editor、VFX budget／failure／test | 本書 |
| C1／C2で製品として提供する2D／3D表現範囲とMilestone | 2D／3D機能計画 |
| Tick、Presentation command、queue、memory parent domain、global performance cap、Asset promotion phase | Runtime規約 |
| `RenderSnapshot`、Render Graph、resource／pass／barrier、Material／Shader pipeline、Backend、device loss | Renderer規約 |
| Source／Derived／Cooked Artifact、Catalog、Package、hot reload transaction | Asset規約 |
| `ProjectChangeSet`、revision、Undo／Redo、唯一のCommit経路 | Authoring規約 |
| Collider、Physics query、Contact／Trigger、authoritative collision | Collision規約 |
| Mobile Capability Signature、Vulkan／Metal最低線、thermal、GPU／CPU aggregate budget | モバイル規約 |
| Widget、Dock、MiraUI、keyboard／accessibility、AI Semantic Interface | Editor UI Framework規約 |
| MCD、Capability、Operation、Diagnostic projection、Provider Schema | 実行可能契約規約 |

本書は次を変更しない。

- Runtimeのexactly 60 Hz、`T90_PresentationBuild`、`T110_Publish`、`R00`～`R70`。
- Rendering CPU／upload内の`VFX CPU simulation` 32 MiB。
- Transparent／VFX全体のGPU P95 1.50 ms。
- RendererのTarget別HLSL／DXIL／SPIR-V／MSL Cook経路。
- Authoring stateをC++ `AuthoringCommandGateway`だけがCommitする原則。
- Collision callback中にVFXを直接呼ばない原則。

VFXはPhysics Backend、World ECS、Editor widget、Graphics Backendを所有しない。必要な入力はvalue型Presentation event、immutable transform／collision proxy snapshot、Asset lease、Render Graph Portだけから取得する。

## 3. 採用判断

### 3.1 比較した三方式

| 方式 | 長所 | 重大な問題 | 判断 |
|---|---|---|---|
| 2D CPU、2D GPU、3D CPU、3D GPUを別Systemとして実装 | 各経路を局所的に理解しやすい | Asset、Editor、AI Operation、Node、Testが4重化し、機能差と変換損失が蓄積する | 不採用 |
| 一つの完全汎用Graph Runtimeで全処理を動的解決 | 最大の拡張性 | 任意control flow、VM overhead、Target差、初心者UX、AI検証、Shipping安全性が過大 | C1／C2では不採用 |
| 単一Asset／型付きIR＋Target別specialization | Authoringを共有し、Runtime layoutとshaderをDimension／Backendごとに最適化できる | Compiler、Node capability、conformance testが必要 | **採用** |

同じAsset形式を使うことと、Runtimeで常に3D dataを持つことは同義ではない。Compilerは2D ArtifactからZ、Quaternion、3D collision fieldを除去し、3D Artifactだけへ3D attributeを配置する。CPUとGPUも同じ実行objectを分岐させず、別`VfxExecutionArtifactV1`としてCookする。

### 3.2 有名Engineから確認した構成

| Engine | 公式資料で確認した構成 | Miraikanaiへの判断 |
|---|---|---|
| Unity 6.0 | InspectorのModuleで構成するBuilt-in Particle Systemと、GPUで大規模Effectを扱うVisual Effect Graphを併用できる。Built-inはCollision、Sub Emitter、Light、Trail等のModuleを持つ | 初心者向けModuleと高性能Graphの両方は必要だが、正規Assetを二系統へ分けない |
| Unreal Engine 5.8 | NiagaraはSystem、Emitter、Module、Parameterを持ち、Emitter単位でCPU／GPU Sim Targetを選ぶ。System／Emitter処理はGPU ParticleでもCPU上に残り、Emitter数、pool、bounds、scalabilityが性能要因になる | System／Emitter／Stage／Renderer分離、予算、pool、fixed boundsを採用する一方、任意VMと無制限Emitterを避ける |
| Godot 4.6 | `GPUParticles2D`、`CPUParticles2D`、`GPUParticles3D`、`CPUParticles3D`を別Nodeとして提供する。GPUからCPUへ変換すると複数Draw Pass、turbulence、sub-emitter、trail、attractor、collision等を失う | 2D専用最適化は必要だが、Authoring sourceとCapability検査を共通化し、変換損失をCook時に明示する |

これらのAsset名、UI配置、Node名、file形式、既定値、Graph外観はコピーしない。MiraikanaiはAI検証、ProjectChangeSet、Dimension specialization、closed Capabilityを中心に独自設計する。

## 4. 成熟度と対象範囲

| Level | 到達点 |
|---|---|
| C0 Foundation | MCD、Asset schema、Node Catalog、Graph Validator、VFX IR、CPU／GPU Artifact descriptor、Cost model、Preview contract、Diagnostic、Qualification harness |
| C1 First Playable | 2D／3D CPU simulation、Point／Line／Area emitter、rate／burst、lifetime、velocity／acceleration／drag、Color／Size／Rotation over Life、Sprite／Flipbook、2D Sprite、3D Billboard、basic trail、fixed-seed preview、bounds／count／overdraw debug |
| C2 Production | GPU compute、Mesh particle、Ribbon、Depth／SDF／VFX proxy collision、Vector Field、Sub-emitter／GPU Event、Particle Light、quality variant、GPU sorting、VFX bake cache、approved custom operator |
| C3 Research | Fluid／volumetric simulation、GPU mesh shader専用経路、ray-traced particle、cross-machine visual lockstep、runtime graph mutation、user-authored arbitrary GPU code |

C1 ArtifactはC2／C3 Nodeを受理しない。C2機能がSourceに含まれる場合、Target Profileに対応Artifactがなくても無言で除去せず、明示したfallback graphが存在する場合だけ別variantとしてCookする。

### 4.1 TargetとPromotion順

| Target | C1 | C2 | Production表示条件 |
|---|---|---|---|
| Windows x64／D3D12 | CPU 2D／3D必須 | GPU必須 | CPU conformance、DXIL compile、D3D12 validation、performance／device recovery |
| Android arm64／Vulkan | CPU 2D必須、CPU 3D scalable subset | Standard以上でGPU | AVP、Adreno／Mali実機、thermal、Vulkan validation、package test |
| iOS／iPadOS arm64／Metal | CPU 2D必須、CPU 3D scalable subset | Standard以上でGPU | A12 baseline、Metal実機、background／surface recovery、thermal、archive test |

Mobile BaselineでGPU Particleを必須化しない。GPU variantはDevice Capability Signature、Target Profile、Quality Profileの全条件が一致した場合だけ選択する。

## 5. Platform構成、Module、Directory

```text
schemas/mira/vfx/
engine/vfx/
  contracts/
  core/
    graph/
    parameters/
    curves/
    random/
    budget/
  compiler/
    validation/
    ir/
    specialization/
    cpu_plan/
    gpu_codegen/
  runtime/
    instances/
    scheduler/
    snapshots/
  cpu/
    kernels/
    storage/
    jobs/
  gpu/
    passes/
    storage/
    sorting/
  render/
    sprite/
    billboard/
    trail/
    mesh/
    ribbon/
    light/
  diagnostics/
  extensions/
authoring/vfx/
editor/panels/vfx/
tools/vfx_compiler/
tools/vfx_qualification/
tests/vfx/
```

| CMake Target | 責務 |
|---|---|
| `mira_vfx_contracts` | 生成value type、enum、Command、Snapshot、Port |
| `mira_vfx_core` | Graph共通意味、parameter、curve、RNG、budget |
| `mira_vfx_compiler` | Source validation、IR、specialization、Artifact生成。Shipping Gameへlinkしない |
| `mira_vfx_cpu` | Engine-owned CPU kernel、SoA storage、job |
| `mira_vfx_gpu` | Backend非依存GPU pass template、buffer plan、shader codegen input |
| `mira_vfx_render` | VFX batchからRenderer packet／Render Graph passを生成 |
| `mira_vfx_runtime` | instance lifecycle、command consume、CPU scheduling、GPU instance descriptor |
| `mira_vfx_authoring` | Document、ChangeSet projection、cost preview、cook request |
| `mira_editor_vfx` | Stack／Graph／Timeline／Preview／Profiler panel |
| `mira_vfx_qualification` | conformance、golden、benchmark、fault fixture |

`mira_vfx_core`、`mira_vfx_cpu`、`mira_vfx_gpu`はGraphics Backend header、Box2D、Jolt、Editor UIへ依存しない。`mira_vfx_gpu`はRendererのBackend非依存Graph／Shader Portだけを使う。D3D12、Vulkan、Metalへの変換はRenderer Adapterが所有する。

Project C++は`mira.runtime.vfx`のCommand／parameter APIだけを通常利用する。`mira.vfx.gpu.backend`、particle buffer pointer、shader compiler objectを公開Moduleにしない。

## 6. 正規ObjectとAsset契約

### 6.1 Object一覧

| Object | 所有者 | Source保存 | Runtime保存 |
|---|---|---:|---:|
| `VfxSystemDocumentV1` | Authoring Model | する | 直接読まない |
| `VfxEmitterV1` | `VfxSystemDocumentV1` | する | Cook済みrecordだけ |
| `VfxGraphV1` | Authoring Model | する | しない |
| `VfxCurveV1`／`VfxGradientV1` | Authoring Model | する | LUTまたはkey artifact |
| `VfxBudgetProfileV1`／`VfxQualityProfileV1` | Target／Project Profile | する | Cooked manifest |
| `VfxGraphIrV1` | Compiler | しない | Development artifactだけ |
| `VfxSystemArtifactManifestV1` | Asset Pipeline | Derivedとして保存 | System Instanceがleaseする |
| `VfxExecutionArtifactV1` | Asset Pipeline | Emitter／Dimension／Target／Quality別Derived | 対応Emitterがleaseして読む |
| `VfxSystemHandle`／`VfxEmitterHandle` | VFX Runtime | しない | `{index32,generation32}` |
| `VfxCpuInstanceStateV1` | VFX Runtime | しない | Play session内 |
| GPU buffer／counter／indirect argument | Renderer Backend | しない | submission lifetime内 |
| `VfxBatchSnapshotV1` | VFX Runtime | しない | `T110`からRender frame内 |

### 6.2 `VfxSystemDocumentV1`

```text
VfxSystemDocumentV1
  document_header
  system_id: StableId
  spatial_domain: d2 | d3 | portable_2d_3d
  loop_mode: once | loop | continuous
  duration_seconds: optional finite f32
  seed_policy: fixed | per_instance | from_presentation_event
  fixed_seed: optional u64
  public_parameters: VfxParameterV1[0..64]
  event_inputs: VfxEventInputV1[0..16]
  emitters: VfxEmitterV1[1..32]
  bounds_policy: VfxBoundsPolicyV1
  quality_profile_ref: StableId
  budget_profile_ref: StableId
  visual_style_roles: StableId[0..16]
  capability_requirements: CapabilityId[0..32]
```

- C1はEmitterを最大16、C2は最大32とする。
- Systemは少なくとも一つの`enabled=true` Emitterを必要とし、全Emitter disabledのSystemをRuntime Instance化しない。
- `once`／`loop`は`duration_seconds`を必須とし、範囲を`[1/60, 3600]`秒とする。
- `continuous`は`duration_seconds`を持たない。
- `portable_2d_3d`はDimension-polymorphic Nodeだけを許可し、Cook時に2D／3D Artifactを別々に生成する。
- Hybrid SceneでDimension-specific Systemを利用する場合、Instance側が`d2`または`d3`を明示する。表示layerや親Entityから推測しない。
- System参照cycleを禁止する。Sub-emitterはC2で深さと生成数を別上限に通す。

### 6.3 `VfxEmitterV1`

| Field | 型／規則 |
|---|---|
| `emitter_id` | System内一意のStableId |
| `enabled` | bool |
| `execution_policy` | `auto \| cpu_required \| gpu_required \| dual_fallback` |
| `simulation_space` | `local \| world`。Instance実行中に変更不可 |
| `max_particles` | `uint32`、選択Budget内 |
| `priority` | `uint8`。Presentation drop順にだけ使用し、critical権限を与えない |
| `rate_q32` | Q32.32 particle／second |
| `bursts` | 最大256、`tick_offset:uint32`と`count:uint32` |
| `lifetime_seconds` | min／maxともfinite、`1/60`～`3600`、min≤max |
| `prewarm_seconds` | C1 `[0,2]`、C2 `[0,5]`。Cook時にstep数を計算 |
| `max_events_per_tick` | C1は0。C2は`0..min(max_particles,4096)` |
| `spawn_graph` | `VfxGraphV1`、Stage=`particle_spawn` |
| `update_graph` | `VfxGraphV1`、Stage=`particle_update` |
| `event_graph` | C2だけ、Stage=`particle_event` |
| `render_outputs` | C1は1～2、C2は1～4 |
| `fixed_bounds` | GPU Productionでは必須。2DはRect、3DはAABB |

`rate_q32`の60 Hz scheduling、`division_remainder`、burst、drop順は2D／3D機能計画のParticle Budget規則をそのまま使用する。Rate、Burst、Event spawnの合計を一つのSpawn admissionへ通し、入口ごとに上限を回避できないようにする。

`prewarm_seconds>0`は`loop`／`continuous`かつScene／Assetの`preload`対象だけに許可する。Play中に初めて動的spawnするone-shotは0でなければならない。PrewarmはLoading boundaryで非表示Instanceへ行い、active frameへ120／300 stepを一括投入しない。即時に過去状態が必要なone-shotはC2 `VfxBakeCacheV1`を使う。

### 6.4 `VfxGraphV1`、Node、Edge

```text
VfxGraphV1
  graph_id: StableId
  stage: particle_spawn | particle_update | particle_event
  nodes: VfxNodeV1[1..stage_limit]
  edges: VfxEdgeV1[0..stage_limit]
  output_bindings: VfxOutputBindingV1[0..64]
  event_routes: VfxEventRouteV1[0..16]
  editor_layout: VfxGraphLayoutV1

VfxNodeV1
  node_id: StableId
  node_type_id: VfxNodeTypeId
  node_schema_version: u32
  literal_fields: {FieldId -> closed typed value}[0..32]
  random_slot_u32: optional

VfxEdgeV1
  edge_id: StableId
  source_node_id: StableId
  source_port_id: PortId
  target_node_id: StableId
  target_port_id: PortId

VfxOutputBindingV1
  attribute_id: VfxAttributeId
  source_node_id: StableId
  source_port_id: PortId

VfxEventRouteV1
  route_id: StableId
  event_pulse_node_id: StableId
  event_pulse_port_id: PortId
  target_emitter_id: StableId
  child_count: u8
  payload_bindings: {EventFieldId -> typed Node output}[0..16]
```

- Node CatalogはNode typeごとに許可Stage、Dimension、version、typed input／output port、literal field、defaultの有無、attribute read／write、Capability、CPU kernel ID、GPU implementation IDを定義する。
- 必須input portは型一致するedgeまたは明示literalを一つだけ持つ。optional portだけCatalog既定値を使える。未接続必須port、未知port、同一target portへの複数edgeを拒否する。
- Edgeは同じGraph内だけを接続し、implicit cast、名前によるport解決、配列indexの自動拡張を行わない。
- `output_bindings`はattribute当たり一つだけ許可する。複数sourceを合成する場合は一つのbindingへ列挙せず、Catalogの`Combine*` Nodeで順序を明示する。
- Spawn Graphは`Position`と`Lifetime`を必ずbindする。その他の未bind初期値は、2D Velocity `vec2f(0,0)`、3D Velocity `vec3f(0,0,0)`、Color `(1,1,1,1)`、2D／Billboard Size `vec2f(1,1)`、Mesh Size `vec3f(1,1,1)`、Rotation／AngularVelocity `0`、FlipbookFrame `0`である。Update Graphの未bind attributeは入力値を保持する。
- `event_routes`は`particle_event`だけが持てる。trigger portはCatalog内部型`event_pulse`、`child_count`は1～8、payloadは256 byte以下でなければならない。`event_pulse`はPublic Parameter、literal、保存field、render outputに使用できない。
- `OnDeath`、`OnVisualCollision`、`ConditionRising`だけがC2の標準`event_pulse` sourceである。`ConditionRising`は前stepのboolをCompiler-declared persistent attributeとして保持し、trueが連続する間は再発火しない。
- `editor_layout`はNode位置、group、comment、折り畳み状態だけを持ち、semantic content hashとCook invalidationから除外する。ただしProjectRevision、Undo／Redo、共同編集の対象には含める。
- Canonical serializationはStableId byte昇順、field ID昇順、Edge ID昇順とし、表示名、挿入順、画面座標をCompiler順序へ使わない。

### 6.5 ParameterとEvent input

`VfxParameterV1`は次を必須とする。

| Field | 規則 |
|---|---|
| `parameter_id` | Stable field ID。表示名を参照に使わない |
| `value_type` | 7.1節のclosed type |
| `default_value` | 型一致、finite、Asset refは存在必須 |
| `range` | 数値型はmin／max、enumは許可集合 |
| `update_rate` | `instance_start \| presentation_frame` |
| `semantic_role` | color、intensity、size、speed等の登録済みID |
| `exposure` | `hidden \| inspector \| gameplay \| ai` |

`presentation_frame` parameterは`SetVfxParameterV1`で次の`T90`に反映する。Particleごとの直接write、pointer、array、string、Entity object、native resourceをParameterにしない。

`VfxEventInputV1`は登録済みPresentation Event Type IDと最大256 byteのclosed payload schemaを持つ。VFXが任意World componentをqueryするData InterfaceはC1／C2に設けない。

Event payloadは`SpawnVfxSystemV1`で新しいSystem Instanceを作る時に一度captureし、そのInstance内ではimmutableである。`EventField` Nodeはこのcaptureだけを読む。同じtickの異なるEvent payloadを一つのEmitterへ暗黙mergeせず、別Instanceまたは明示したaggregate Presentation Eventへする。

### 6.6 Seed policy

- `fixed`は`fixed_seed`を必須とし、全Instanceでその値を使う。
- `per_instance`は`LE64(SHA256("MIRA_VFX_INSTANCE_SEED_V1\0" || project_seed_le64 || instance_sequence_le64)[0..7])`でRuntimeが導出する。
- `from_presentation_event`はEvent schemaに`vfx_seed:u64`を必須とし、不足時はSpawn Commandを拒否する。
- `instance_sequence`は`T90`でcanonical merge済み`SpawnVfxSystemV1`順に単調増加し、thread completion順を使わない。

外部AIまたはProject dataがRuntimeの`resolved_seed` fieldを直接書かない。Editor Previewは表示中のseedを固定／copyできる。

## 7. Graph、型、Stage意味

### 7.1 Closed value type

```text
bool
i32
u32
u64
f32
vec2f
vec3f
vec4f
color_linear_rgba
quaternionf
spatial_vector
curve1_ref
gradient_ref
texture2d_ref
mesh_ref
material_ref
```

- `spatial_vector`はSource Graphだけのpolymorphic typeで、2D Artifactでは`vec2f`、3D Artifactでは`vec3f`へ必ずspecializeする。
- 2D rotationは+Z軸周りのradian scalarとし、Quaternionを受理しない。
- 3D Billboard rotationはview-facing plane上のradian scalar、Mesh particle rotationはC2のnormalized Quaternionとする。
- 距離はmeter、時間はsecond、角度はradian、色はlinear RGBAを正規単位とする。
- NaN、Inf、非正規化Quaternion、型の暗黙scalar／vector変換を拒否する。

### 7.2 Stage

| Stage | 実行時点 | 読取 | 書込 |
|---|---|---|---|
| `emitter_control` | `T90`でEmitterごと | Parameter、Event、Emitter transform、tick | spawn count、Emitter local state |
| `particle_spawn` | 新規Particleごと | spawn ID、seed、Parameter、Emitter transform | 初期Particle attribute |
| `particle_update` | alive Particleごと、fixed step | 現attribute、Parameter、time | 次attribute、alive flag |
| `particle_event` | C2条件成立時 | Particle attribute、visual collision result | bounded internal VFX event |
| `render_output` | Snapshot／Render packet構築時 | 最終attribute、Material parameter | Renderer bindingだけ |

GraphはStageごとに有向非循環でなければならない。前frame値は`PreviousAttribute`としてCompilerが宣言したstate fieldだけから読み、Graph edgeのcycleで表現しない。

`emitter_control`はC1／C2ともEngine-ownedのrate、burst、event admission planであり、Source Documentに任意Emitter Graphを持たせない。`render_output`も`render_outputs` descriptorからCompilerが作るEngine-owned binding stageで、任意Graphではない。Emitter制御を拡張する場合はParameterまたはPresentation Eventを使い、任意control-flow VMを追加しない。

一つのattributeへ複数Nodeがwriteする場合、`CombineAdd`、`CombineMultiply`、`CombineMin`、`CombineMax`のいずれかを明示し、入力をNode StableId順に評価する。暗黙last-write-winsを禁止する。

### 7.3 C1 Node Catalog

| 分類 | C1 Node |
|---|---|
| Input | Constant、Parameter、Age、NormalizedAge、Lifetime、SpawnId、SimulationStep、EmitterTransform、EventField |
| Math | Add、Subtract、Multiply、DivideSafe、Min、Max、Clamp、Abs、SqrtSafe、Lerp、Remap、Dot、Length、NormalizeSafe、Select |
| Random | Uniform01、Range、UnitDirection2D、UnitDirection3D、RandomColorGradient |
| Curve | SampleCurve、SampleGradient |
| Spawn shape 2D | Point、LineSegment、Rectangle、CirclePerimeter、Disk |
| Spawn shape 3D | Point、LineSegment、Box、SphereSurface、SphereVolume、Cone |
| Initialize | Position、Velocity、Lifetime、Color、Size、Rotation、AngularVelocity、FlipbookFrame |
| Update | IntegrateVelocity、Acceleration、Gravity、LinearDrag、ColorOverLife、SizeOverLife、RotationOverLife、KillByAge |
| Output | Sprite2D、Billboard3D、PortableFacingSprite、Flipbook、BasicTrail |

`DivideSafe`は分母絶対値が`1e-8`未満ならNode parameterで明示したfallbackを返す。`NormalizeSafe`も長さが`1e-8`未満なら明示fallback vectorを返す。Compilerが0除算を黙って0へ置換しない。

### 7.4 C2追加Node

- Curl／value noise、turbulence、Vector Field sampling。
- Mesh surface／volume emission。
- Depth collision、Global SDF collision、VFX collision proxy。
- Collision bounce／friction／kill、visual collision event。
- Sub-emitter event、GPU event。
- Mesh、Ribbon、Particle Light output。
- Depth fade、soft particle、distortion parameter。
- Camera distance／quality parameter、LOD branch。
- 承認済み`VfxExtensionOperatorV1`。

Scene color sampling、arbitrary texture write、ray tracing、mesh shader、unbounded bindless、GPU readback eventはC2 Node Catalogへ含めない。

### 7.5 Graph上限

| 項目 | C1 | C2 |
|---|---:|---:|
| Node／Emitter | 256 | 1,024 |
| Edge／Emitter | 1,024 | 4,096 |
| Persistent custom attribute／Emitter | 16 | 64 |
| Curve／Emitter | 32 | 128 |
| Key／Curve | 64 | 256 |
| Public parameter／System | 64 | 64 |
| Render output／Emitter | 2 | 4 |

Node／Edge上限を超えたGraphは分割を提案できるが、Compilerが自動で複数Emitterへ意味変更しない。

## 8. Curve、Gradient、Random

### 8.1 CurveとGradient

- Curve key timeはnormalized `[0,1]`、厳密昇順で、最初を0、最後を1とする。
- interpolationは`step`、`linear`、`cubic_hermite`のclosed enumとする。
- cubic tangentはfiniteで、weighted tangentはC1で受理しない。
- Gradientはlinear RGBAのkeyを最大64持ち、同じtimeのcolor／alpha keyを許可しない。
- Source colorはstraight alpha、Render outputでMaterial blend契約に応じてpremultiplyする。

GPU用LUTは64、128、256、512、1,024 sampleの最小サイズをCompilerが昇順に試す。各key区間の端点、中央、四分点でSource評価との差を測り、scalarは`max(1e-4, source_range * 1e-3)`、colorは各channel `1/1024`以下を満たす最小サイズを選ぶ。1,024で満たせなければGPU variantを拒否し、CPU variantへ黙って切り替えない。

### 8.2 `VfxCounterRngV1`

VFX乱数はper-particleの`std::mt19937` stateを保持しない。CPU／GPU共通のcounter-based `Philox4x32-10`を`VfxCounterRngV1`として使う。

```text
emitter_seed32 =
  LE32(SHA256("MIRA_VFX_EMITTER_SEED_V1\0" || emitter_id_canonical_bytes)[0..3])

counter =
  [low32(particle_spawn_id),
   high32(particle_spawn_id),
   simulation_step_u32,
   random_slot_u32]

key =
  [low32(system_seed_u64) xor emitter_seed32,
   high32(system_seed_u64)]
```

- `simulation_step_u32`はParticleローカルの`age_steps`である。Spawn Stageは0、最初のUpdate／Event評価は1とし、Global tickの下位32 bitを使わない。
- Random Nodeは作成時にEmitter内の全Stageを通じて一意の`random_slot_u32`を取得し、そのEmitter revision内で削除後も再利用しない。Node複製は新しいslotを得る。
- Philox出力4 laneをscalar／vector成分へlane 0から順に割り当てる。一つのNodeが4値を超えて要求することをSchemaで禁止する。
- `u32`から`[0,1)`の`f32`への変換は`(u >> 8) * 0x1p-24f`とする。
- System Seed、Emitter ID、Spawn ID、Step、Random Slotが同じなら整数乱数出力はCPU／GPUで同じでなければならない。

これはPresentation専用RNGであり、Runtime規約のauthoritative `DeterministicRngV1`を置き換えない。GPUの浮動小数点simulation全体はbitwise replay対象外である。

## 9. CompilerとCooked Artifact

### 9.1 Compiler pipeline

`VfxCompiler`は次を固定順で実行する。

1. Document、StableId、field range、Asset reference、Target Profileを検証する。
2. Stage、型、Dimension、attribute read／write、Graph cycleを検証する。
3. Node Catalog versionとCapability requirementを検証する。
4. Emitter／System／ProjectのBudget、memory、spawn、renderer、overdraw見積りを検証する。
5. Node StableIdをtie-breakとしてcanonical topological orderを作る。
6. constant folding、dead-node elimination、attribute livenessを行う。浮動小数点演算の再結合はしない。
7. `d2`／`d3`をspecializeし、未使用Dimension fieldを除去する。
8. 9.2節の規則でCPU／GPU execution variantを解決する。
9. CPUは`VfxCpuProgramV1`、GPUはportable shader IRとRender Graph pass templateへ生成する。
10. RendererのShader pipelineでDXIL／SPIR-V／MSLをTarget別にoffline compile／validationする。
11. Artifact manifest、resource estimate、interface hash、source hash、compiler hash、test fixture hashを生成する。
12. Asset Pipelineのtransactional CookでReady generationへ置き、全closure検証後だけpromotion候補にする。

Compiler warningを成功条件の代替にしない。unsupported Node、layout overflow、shader compile failure、fallback不足、Budget超過は該当variantを失敗させる。

### 9.2 Execution target解決

Compilerは各enabled Emitterについて独立に次を解決する。同じSystem内のCPU／GPU Emitter混在は許可するが、Parameter、System seed、Instance transform、lifecycleを共有するだけでParticle storageを共有しない。

| 条件 | 解決 |
|---|---|
| C1 Asset | CPU |
| `cpu_required` | CPU capabilityとBudgetがなければ失敗 |
| `gpu_required` | C2、compute、storage buffer、atomic counter、indirect draw、shader variantが全てなければ失敗 |
| `dual_fallback` | CPUとGPUを両方Cookし、Target Quality Manifestが一方を正規選択する |
| `auto`かつGPU-only Nodeを使用 | GPU。GPU非対応Targetには明示fallback graphが必要 |
| `auto`かつpeak alive見積り4,096以上 | GPU capabilityがあればGPU、なければCPU Budget内の場合だけCPU |
| `auto`かつpeak alive見積り4,095以下 | CPU |
| `mobile_baseline` | CPUを正規選択。限定GPUはDevice Qualification Receiptで明示enableしたProfileだけ |

Runtimeがframeごとの負荷を見てCPU／GPU実行中stateを移送しない。Target／Qualityごとの選択はCooked `VfxSystemArtifactManifestV1.emitter_artifact_refs`に保存し、Instance開始時に一意に決める。

### 9.3 `VfxExecutionArtifactV1`

```text
VfxSystemArtifactManifestV1
  system_id
  source_content_hash
  target_profile_id
  quality_profile_id
  emitter_artifact_refs[1..32]
  parameter_layout_hash
  lifecycle_descriptor
  closure_hash

VfxExecutionArtifactV1
  artifact_schema_version
  system_id
  emitter_id
  source_content_hash
  compiler_build_hash
  node_catalog_version
  target_profile_id
  quality_profile_id
  spatial_domain: d2 | d3
  execution_target: cpu | gpu
  parameter_layout_hash
  attribute_layout_hash
  renderer_interface_hash
  cpu_program_hash: optional
  shader_package_refs[]
  render_pass_template_ids[]
  fixed_bounds
  resource_estimate
  capability_requirements[]
  fallback_artifact_ref: optional
  verification_receipt_refs[]
```

System manifestは全enabled Emitterの該当Dimension ArtifactがReadyの場合だけReadyになる。Artifact keyはSystem ID、Emitter ID、Dimension、execution targetに加え、Asset規約のTarget、Quality、Toolchain、Source dependency closureを含む。Source hashまたはCompiler hashが異なるArtifactを同一keyで再利用しない。

### 9.4 CPU programの性質

`VfxCpuProgramV1`は任意bytecodeやGame Scriptではない。Engine binaryへ事前compileされたclosed C++ kernel IDとimmutable parameter blockを、Compilerが型検査済み順序へ並べた実行計画である。

- loop、recursion、file／network／OS API、reflection、allocation instructionを持たない。
- Kernelごとにread attribute、write attribute、scratch byte、Dimension、StageをManifestへ固定する。
- Runtimeは未知Kernel ID、Manifest hash不一致、parameter block size不一致をAsset promotion時に拒否する。
- Shipping packageへVFX CompilerまたはNode sourceを含めない。

## 10. Runtime lifecycle、Command、Phase

### 10.1 Instance state

```text
Unloaded -> Ready -> Prewarming -> Running -> Draining -> Complete
               |          |          |          |
               +----------+----------+-> Stopped+
Running <-> Paused
Any live state -> Faulted
```

- `Ready`はArtifact lease、instance slot、parameter block、必要buffer容量が確保済みである。
- `Prewarming`はpreload対象の非表示Instanceだけが入り、固定stepをLoading budget内で処理する。
- `prewarm_seconds = 0`でも同じ`T90`内のzero-work `Prewarming`遷移を通し、別の初期化経路を作らない。
- CPU Prewarmは共有Worker Poolで通常fixed-step kernelを実行する。GPU Prewarmは一publish当たり最大8 `VfxGpuTickRecordV1`をhidden computeとしてRendererへ渡し、対応submission fence完了後だけ次batchへ進む。全step完了前にdrawまたは`Running`へ移らない。
- GPU／Render surfaceが利用できないLoading経路ではGPU Prewarmを開始せず、Cooked CPU fallbackがあればそれを選び、なければScene loadを`VfxCapabilityUnavailable`で失敗させる。300 stepを一dispatchへ圧縮しない。
- `Running`はspawnとupdateを行う。
- `Draining`は新規spawnを止め、alive Particleが0になるまでupdateする。
- `Stopped`はalive Particleを破棄し、次のpublish境界でresource retireへ送る。
- `Paused`はage、rate accumulator、simulation stepを進めない。
- `Faulted`は新規処理を止め、Diagnosticを発行し、authoritative Worldを変更しない。

### 10.2 Runtime Command

| Command | 必須field | 適用 |
|---|---|---|
| `SpawnVfxSystemV1` | asset、transform source、event payload、parameter block、priority。seedは6.6節でEngineが解決 | `T90` |
| `StopVfxSystemV1` | handle、`immediate \| drain` | `T90` |
| `PauseVfxSystemV1`／`ResumeVfxSystemV1` | handle | `T90` |
| `SetVfxParameterV1` | handle、parameter ID、typed value | 次の`T90` |
| `SetVfxTransformV1` | handle、finite transform | 次の`T90` |

すべてRuntime規約の`PresentationCommand` envelope、canonical merge、capacity、drop policyに従う。VFX Commandにcritical bitを設定できない。

### 10.3 Phase接続

1. `T70`までにGameplay／Collisionがauthoritative結果からPresentation Eventを生成する。
2. `T90`がEventとCommandをlatchし、VFX Instance操作、rate／burst admission、CPU fixed-step jobを行う。
3. CPU jobは固定partitionのprivate outputへ書き、`T90`終了前にjoinする。
4. VFXはWorldへwrite backせず、`VfxBatchSnapshotV1`だけをPresentation bufferへ構築する。
5. `T110`で全`RenderSnapshot`の`vfx_batch[]`として一度だけpublishする。
6. Rendererは`R00`でSnapshotをleaseし、CPU draw packetまたはGPU pass instanceを構築する。
7. GPU simulation／drawは`R30`でRender Graphへcompileし、`R50`で明示dependency順にsubmitする。

`T90`中にVFXが生成したSub-emitter requestは次Presentation frameへ送る。同じStageへの再入実行を禁止する。

`VfxBatchSnapshotV1`はpointerではなく次のvalue recordを持つ。

```text
VfxBatchSnapshotV1
  snapshot_tick: u64
  cpu_draw_batches[]
  gpu_emitter_records[]

VfxGpuEmitterRecordV1
  system_instance_handle
  emitter_id
  artifact_version
  current_transform
  event_payload
  parameter_blocks[1..8]
  last_tick_sequence: u64
  tick_records[0..8]

VfxGpuTickRecordV1
  tick_id: u64
  first_reserved_spawn_id: u64
  external_spawn_count: u32
  sub_emitter_spawn_quota: u32
  emitter_transform
  parameter_block_index: u8
  simulation_step_count: u8 = 1
```

`parameter_blocks`は直近8 tickで参照する異なるrevisionだけをdeduplicateし、`parameter_block_index`で参照する。VFX RuntimeはRunning／Draining中のGPU Emitterごとに毎fixed tick一つのrecordを生成し、直近8 tickをPresentation stateへ保持する。`external_spawn_count`はrate、burst、Presentation Eventから当該tickにadmission済みの数、`sub_emitter_spawn_quota`は当該EmitterをtargetとするGPU内部Event用に同じProject Spawn budgetから予約した上限である。予約範囲は`first_reserved_spawn_id`から両者の合計数で、外部spawnを先、Sub-emitterを親Emitter ID／親Spawn ID／Event Node ID順に割り当て、未使用quotaのIDはgapとして再利用しない。

RendererはInstanceごとの`last_consumed_tick_sequence`より新しいrecordだけをtick昇順に処理し、各recordにつき`1/60 s`を一step実行する。同じSnapshotを120 Hz等で再描画してもsimulationを二重実行しない。30 fps描画では通常2 recordを一frameで処理する。Paused中はrecordを生成せず、Resume後の最初のrecordを一stepとして扱う。

Renderer停止等で8 tickを超えるgapが生じた場合、欠落stepを巨大dispatchで追いつかせない。ambient／loop Instanceは最新tickからvisual restartし、one-shotは再発火せず`VfxGpuTimelineGap`を記録する。これはPresentation縮退であり、Gameplay stateへ影響しない。

## 11. CPU Simulation

### 11.1 Storage

CPU ParticleはStructure of Arraysで保持する。

- allocation単位は256 Particleの64 byte aligned chunk。
- `max_particles`から必要chunk数をInstance開始時に確保し、Particle spawn／update／deathでheap allocationしない。
- mandatory fieldは`spawn_id:u64`、`age_steps:u32`、`lifetime_steps:u32`、`alive:u32`である。
- Position、previous position、velocity、color、size、rotation等はGraph livenessで必要なarrayだけ配置する。
- 2Dは`vec2f`、3Dは`vec3f`を別layoutにし、2D arrayへpadding目的のZを保存しない。
- Asset／Material／Texture参照をParticleごとに持たず、Emitter／Output bindingへ集約する。
- `VfxSystemHandle`、`VfxEmitterHandle`以外のraw pointerをCommand、Snapshot、job captureへ保存しない。

### 11.2 Updateとstable compaction

1. Alive Particleを`spawn_id`昇順のlogical orderで保持する。
2. 256 Particle chunkを連続rangeへ分割し、共有Worker Poolへ固定順で提出する。
3. 各jobはprivate dead maskとevent bufferへ書く。
4. join後、chunk index昇順でalive countのprefix sumを作る。
5. 次bufferへstable compactし、同じ入力とBuildでParticle順が変わらないようにする。
6. 当該tickでBudget admission済みのParticleをSpawn ID昇順に空slotへ作り、age 0のまま初期attributeを設定する。
7. 空chunkだけをInstance poolへ返し、memory domain外へ解放しない。

swap-and-pop、thread completion順のappend、unordered reductionを使用しない。Scalar reference kernelとoptimized kernelは同じFixtureで比較する。

### 11.3 Integration

- fixed stepは`1/60 s`。
- CPU VFX kernelは浮動小数点のreassociationと暗黙FMA contractionを禁止するTarget Execution Profileを使い、Compiler／flagはArtifact Receiptへ保存する。
- Velocity integrationは`v_next = v + a * dt`、Positionは`p_next = p + v_next * dt`のsemi-implicit Eulerに固定する。
- Linear dragは`v_next *= max(0, 1 - drag_per_second * dt)`とする。
- `drag_per_second`は`[0,60]`、acceleration magnitudeはProject Profile上限、Position／Velocityはfiniteでなければならない。
- Spawn時に選んだ`lifetime_seconds`は`lifetime_steps = clamp(ceil(lifetime_seconds * 60), 1, 216000)`へ一度だけ量子化する。`Age`／`Lifetime` Nodeが返す秒値はそれぞれ`age_steps / 60.0f`、`lifetime_steps / 60.0f`である。
- 既存ParticleはUpdate評価前に`age_steps`を1増やし、`age_steps >= lifetime_steps`ならdead候補にする。最終attributeとdeath／collision条件から同stepのEventを確定した後にdeadをcompactする。`age_steps`は最大216,000なので`u32` wrapを許さない。
- 各fixed stepは「既存alive Particleのage増加／update／death判定、visual event確定、stable compact、新規Particle spawn」の順とする。新規Particleは生成tickではupdateせず、次tickからageを進める。
- Render interpolationを使うEmitterだけ`previous_position`をlivenessへ追加し、Render frame alphaでprevious／currentをLerpする。

C1でIntegratorをNodeごとに変更できない。C2で別Integratorを追加する場合はNode Catalog version、golden、CPU／GPU visual toleranceを別に持つ。

### 11.4 Bounds

CPU Emitterはalive Position、Size、Trailをstable reductionし、毎tick dynamic boundsを生成できる。Reductionはchunk index順とし、NaN／Infを検出したInstanceをFaultedへ移す。

GPU ArtifactはProductionでfinite fixed boundsを必須とする。Compilerのmotion envelopeで証明できる場合は提案値を生成できるが、Authoring sourceへ自動CommitせずPreviewで人間またはPolicy承認を得る。Developmentではbounds外Particleを`VfxBoundsEscape`として計測する。

## 12. GPU SimulationとRender Graph

### 12.1 前提Capability

GPU variantは次をすべて必要とする。

- compute shader。
- read／write storage buffer。
- 32 bit atomic counter。
- indirect draw argument buffer。
- Rendererが提供する明示barrier／queue dependency。
- Target用の検証済みShader Package。

Wave size、subgroup幅、mesh shader、ray tracing、unbounded bindlessを前提にしない。`portable_mobile_v1`のShader Capability intersection内で生成する。

### 12.2 Persistent buffer

| Buffer | 内容 |
|---|---|
| `ParticleAttributesA/B` | liveness後のSoA attribute。必要fieldだけ |
| `AliveIndicesA/B` | 現在／次frameのalive slot |
| `DeadIndices` | 再利用可能slot |
| `EmitterParameters` | immutable／frame parameter block |
| `SpawnRequests` | admitted rate／burst／event spawn |
| `EventCandidates` | 現stepのC2内部VFX event候補。CPU readback不可 |
| `SubEmitterReady`／`SubEmitterNext` | 同一GPU execution island内で、前stepからspawn可能なevent／現stepで確定したeventのping-pong。target Emitter IDを持つ |
| `SortKeys`／`SortIndices` | sort有効時だけ |
| `IndirectArguments` | draw instance count等 |
| `Counters` | alive、dead、spawn accepted、event、overflow |

Counter storageはBuffer内にEngineが明示確保し、Backend固有の暗黙counter lifetimeへ依存しない。GPU bufferをCPUが毎frame readbackしてdraw countまたはeventを決めない。

### 12.3 Pass順序

```text
VfxGpuResetCounters
-> VfxGpuUpdateAndCompact
-> VfxGpuEventResolve          # C2、必要時だけ
-> VfxGpuAdmitAndSpawn
-> VfxGpuSort                  # 必要時だけ
-> VfxGpuBuildIndirect
-> VfxDraw
```

`VfxGpuResetCounters`はnext-alive、event-candidate、`SubEmitterNext`、spawn、drop、indirect等のstep一時counterだけを初期化し、現在のalive／dead stateと`SubEmitterReady`を破棄しない。Event Resolveは現stepの候補を`SubEmitterNext`へ確定し、Admit／Spawnは`SubEmitterReady`だけを`sub_emitter_spawn_quota`まで消費する。step末にReady／Nextをswapするため、生成eventを同じstepへ再入させない。quota超過eventは規定順の末尾からdropし、次stepへ繰り越さない。

新しい`VfxGpuTickRecordV1`がなければsimulation Passを作らず、既存stateから`VfxGpuBuildIndirect`と`VfxDraw`だけを行う。複数recordがあるframeはtick昇順にReset→Update／Compact→Event→Spawn→Ready／Next swapをrecordごとに反復し、全step後にSort→Indirect→Drawを一度行う。

各PassはRendererの`RenderPassDescriptor`、resource access、queue class、declared costを持つ。Compilerはread-before-write、unordered write、counter容量、indirect usageをRender Graphへ明示する。Backend最適化でPassをmergeできても意味順を変更しない。Budget admission済みSpawn ID rangeをGPU容量不足で全て生成できない場合もIDを再利用せず、生成できなかった数をdropとして記録し、次tickへ繰り越さない。

### 12.4 Backend mapping

| Backend | Storage／Compute | Indirect draw | 同期 |
|---|---|---|---|
| D3D12 | Buffer UAV／SRV、Compute Dispatch | `ExecuteIndirect`とEngine command signature | Enhanced BarrierまたはRendererの検証済みlegacy path、queue fence |
| Vulkan | storage buffer、compute pipeline | `vkCmdDrawIndirect`／indexed variant | Render Graphからstage／access mask、queue dependency |
| Metal | `MTLBuffer`、compute pass | indirect argument bufferによるdraw | encoder boundary、resource usage、fence／event／completion |

Backend native object、counter offset、descriptor、argument tableはRenderer Adapter内部に留める。VFX Graphへnative resource stateを保存しない。

### 12.5 GPU sorting

| Mode | 用途 | 規則 |
|---|---|---|
| `none` | additive、opaque-like cutout | Particle sortなし |
| `spawn_order` | 2D、trail、順序重視 | Spawn ID順。上限内だけ |
| `view_depth` | alpha 3D | view depth降順、同bucketはSpawn ID順 |
| `emitter_only` | 大量Particle | Emitter単位だけRendererがsort |

`view_depth`はEmitter 65,536、Project 262,144 Particleまでとし、超えるGraphは`emitter_only`またはadditive Materialを選ばなければCookを拒否する。GPU sort結果はvisual-onlyで、Replay digestへ含めない。

## 13. Renderer、2D／3D specialization

### 13.1 C1 output

| Output | 2D | 3D |
|---|---|---|
| `Sprite2D` | Canvas transform、layer、order、pivot | 不可 |
| `Billboard3D` | 不可 | screen-facing、axis-locked、velocity-aligned |
| `PortableFacingSprite` | `Sprite2D`へspecialize | `Billboard3D`へspecialize |
| `Flipbook` | Sprite texture sheet | Billboard texture sheet |
| `BasicTrail` | 2D polyline／strip | camera-facing 3D strip |

C1 blend modeは`premultiplied_alpha`、`additive`、`multiply`とする。straight-alpha Source textureはImport時にMaterial契約へ合わせ、Shader内で毎Particleの不定なpremultiply判断をしない。

### 13.2 C2 output

- instanced Mesh particle。
- Ribbon。
- Particle Light。
- Distortion parameterを持つSprite／Billboard。
- lit VFX Material。

Particle LightはRenderer light listへだけ出力する。Shadow casting、GI injection、Physics、Navigation、AI perceptionはC2で禁止する。Project 32、Emitter 4の上限を超えるLightを生成しない。

### 13.3 Composition

- 3D Transparent／World VFXはRenderer規約の3D Transparent段で描画する。
- 2D World Particleは2D World Layerへ入り、Canvas layer／orderを使う。
- Pixel-locked Particleは明示`pixel_locked=true`とPoint sampling／integer scaleを必要とし、World dynamic resolutionから除外する。
- UI ParticleはC2の別Capabilityであり、C1のWorld VFXをUI tree内へ直接埋め込まない。
- Toon／Realistic／`pixel_diorama`は`vfx_unlit`／`vfx_lit` Material roleとVisual Style Profileで解決し、simulation coreを分岐しない。

## 14. Collision、Event、Gameplay分離

### 14.1 C1

C1 Particleはper-particle Physics collisionを行わない。Impact、explosion、footstep、damage等はCollision／Gameplayがauthoritative結果を確定し、そのPresentation EventからVFXをspawnする。

### 14.2 C2 visual collision

| Source | 対象 | 制約 |
|---|---|---|
| Scene depth | 3D GPU | 現Viewに見えるsurfaceだけ。off-screen正確性を保証しない |
| Global SDF | 3D GPU | Rendererが提供するversioned SDF snapshotだけ |
| `VfxCollisionProxy2D` | 2D CPU／GPU | Collisionがpublishしたprimitive／segmentのimmutable presentation copy |
| `VfxCollisionProxy3D` | 3D CPU／GPU | primitive／convex proxyのimmutable presentation copy |

VFXはBox2D／Jolt World、Body pointer、Contact callbackを直接queryしない。Proxyは前tickまでのauthoritative stateから作るため、visual collisionに一tick以上の差があり得る。

```text
VfxCollisionProxySnapshotV1
  snapshot_tick: u64
  generation: u64
  proxy2d[0..4096]
  proxy3d[0..2048]
  acceleration_artifact_handle
```

- 2D Proxy shapeはCircle、CapsuleSegment、AABB、8頂点以下のConvexPolygon、3DはSphere、Capsule、Box、32 plane以下のConvexだけとする。Triangle mesh、heightfield、vendor shape pointerをSnapshotへ含めない。
- Collision SubsystemがStableId、finite transform、shape、`vfx_collision_priority:u8`、friction／restitution `[0,1]`をvalue recordへ変換し、2D uniform gridまたは3D BVHのimmutable acceleration artifactと共に`T110`でpublishする。
- Windows上限は2D 4,096、3D 2,048、Mobile上限は2D 1,024、3D 512 Proxyである。静的SceneのCook見積り超過は拒否し、動的超過は`priority desc, proxy StableId asc`の末尾を除外して`VfxCollisionProxyOverflow`を発行する。
- CPU／Proxy GPU collisionはprevious→current Positionのswept point／sphereを一fixed step当たり最大1 hitだけ解決する。半径はOutputのcollision radius fieldで明示し、Sprite画像のalphaやMesh boundsから推測しない。
- 一つのstepで複数surfaceを反復解決せず、最小TOI、同値ならProxy StableId昇順を選ぶ。bounce、friction、kill、EventのいずれかをGraphで明示し、未指定時はpositionをhit pointへclampしてnormal速度を0にする。
- Scene depthはOpaque depth完成後のGPU passだけで使い、画面外、透明surface、別Viewの正確性を保証しない。Global SDFとProxy artifactはSnapshot generationをRender Graph resource dependencyへ含める。

### 14.3 Event

- CPU visual collision Eventはbounded internal VFX Eventとして次Presentation frameのSub-emitterにだけ使える。
- GPU EventはGPU buffer内でSub-emitterへだけ配送し、CPU readbackしない。
- CPU／GPUとも生成したEventを現在のfixed stepへ再入させず、最短で次fixed stepのSpawn admissionへ渡す。
- Sub-emitter深さは2、親Particle一つ当たり生成数は8とする。CPUは次`T90`の通常Spawn admissionへ通し、GPUはtarget Emitterの`VfxGpuTickRecordV1.sub_emitter_spawn_quota`としてProject Spawn budgetを事前予約する。未使用quotaを別Systemへ事後再配分しない。
- Event候補は`親Emitter StableId asc, 親Particle Spawn ID asc, Event Route StableId asc`でcanonical化し、`max_events_per_tick`超過分を末尾からdropして次stepへ繰り越さない。CPUは`VfxEventOverflow`を即時集計し、GPUはon-device overflow counterへ記録する。
- Sub-emitter edgeの親／target Emitterは同じexecution targetでなければならない。GPU→CPUはreadbackを、CPU→GPUは別delivery timingを必要とするため、Compilerがcross-target edgeを拒否する。両方を同じImpactから起動する場合は一つのauthoritative Presentation Eventから二つのEmitter／Systemを独立spawnする。
- VFX EventからDamage、Quest、Save、GameplayDefinition state、Audioの正規Eventを生成しない。
- 同じImpactでAudioとVFXが必要な場合、authoritative Impact EventをAudioとVFXがそれぞれ購読する。

## 15. Determinism、Save、Replay、Bake

### 15.1 CPU repeatability

CPU C1は、同じBuild Manifest、Target Execution Profile、Artifact hash、seed、Command列、fixed tick列に対して同じParticle count、Spawn ID、attribute byte列を出力する。optimized kernelはscalar referenceとbitwise一致をC1 Gateとする。

異なるCPU ISA、Compiler version、GPU Backend間のbitwise一致は保証しない。Target間はReference imageと統計値の許容差で検証する。

### 15.2 GPU

GPUは同じseedから同じcounter RNG整数を得るが、浮動小数点、sort、driver、thread scheduleを含む全simulationはbitwise replay対象外である。GPU Particle bufferをReplay digest、authoritative Save、network stateへ含めない。

Profile／Diagnostic buildはoverflow、alive数、処理時間の観測目的に限り低頻度の非同期counter readbackを許可する。通常frameのdraw、VFX Event、Gameplay判断、Save、Replay、Networkはreadback結果へ依存してはならない。

### 15.3 Save

通常のone-shot VFXは保存しない。永続ambient Systemだけ次を保存できる。

```text
PersistentVfxDescriptorV1
  system_asset_id
  instance_stable_id
  spatial_domain
  parameter_block
  system_seed
  start_tick
  paused
```

alive Particle、GPU buffer、sort state、trail pointを保存しない。Load後は最大120 fixed step、すなわち2秒だけprewarmする。2秒を超える過去状態は正確再現せず、loop phaseを`(load_tick-start_tick) mod duration_ticks`から設定してvisual restartする。連続性がGameplayまたは映像編集で必須のEffectはVFX ParticleではなくAnimation／Baked VFX Cacheを使う。

### 15.4 `VfxBakeCacheV1`

C2 Editorはoffline simulationをFlipbook、Mesh／Ribbon cache、またはframeごとのbounded VFX attribute cacheへBakeできる。CacheはSource Artifact hash、seed、fixed delta、frame count、Target formatを持ち、Source変更でinvalidateする。Shippingでruntime compilerやsimulation sourceを必要としない。

## 16. Memory、Budget、性能

### 16.1 Windows Reference Profile

| 項目 | C1 CPU | C2 CPU | C2 GPU |
|---|---:|---:|---:|
| Active Emitter | Project 256 | Project 256 | Project 1,024 |
| Alive Particle | Project 65,536／Emitter 8,192 | Project 65,536／Emitter 8,192 | Project 1,048,576／Emitter 262,144 |
| Spawn rate | Project 20,000／s／Emitter 4,096／s | Project 20,000／s／Emitter 4,096／s | Project 524,288／s／Emitter 131,072／s |
| Burst／tick | Project 8,192／Emitter 2,048 | Project 8,192／Emitter 2,048 | Project 131,072／Emitter 32,768 |
| Trail point | Project 16,384／Trail 64 | Project 16,384／Trail 64 | Project 262,144／Trail 256 |
| Internal VFX event／tick | 0 | Project 16,384／Emitter 2,048 | Project 65,536／Emitter 4,096 |
| Child emitter | 0 | depth 2、親Particle当たり8 | depth 2、親Particle当たり8 |
| Particle Light | 0 | Project 32／Emitter 4 | Project 32／Emitter 4 |
| CPU memory | 32 MiB | 32 MiB | metadataはRendering CPU budget内 |
| GPU memory | 0 | 0 | persistent 128 MiB＋transient 64 MiB |
| P95 | CPU simulation 0.75 ms | CPU simulation 0.75 ms | simulation＋draw 0.75 ms |

- C1 32 MiBはRuntime規約のRendering CPU／upload内`VFX CPU simulation`へchargeする。
- C2 GPU memoryはRendererのpersistent／transient Domainへchargeし、Texture／Meshは各Asset Domainへ別途chargeする。
- 0.75 msは独立予約ではない。CPUは`PostPhysics＋Presentation`、GPUはTransparent／VFX 1.50 msの内数である。
- Project／Emitter上限の両方を検査し、Editor／AI proposal／Cookでは超過を拒否する。
- `VfxSystemInstance` slot pool上限は`min(4096, CPU Active Emitter上限 + GPU Active Emitter上限)`に固定する。各live Systemは有効なEmitterを最低1つ持ち、各Emitterがexecution target側のslotを消費するため、System数でEmitter上限を迂回できない。

### 16.2 Resource accounting

Emitter容量は`capacity = ceil(max_particles / 256) * 256`へ丸める。Compilerはattribute liveness後の各SoA arrayについて`align_up(sizeof(attribute) * capacity, 64)`を計算し、次をArtifactの`resource_estimate`へ保存する。

- CPU persistentはattribute current／nextの2面、alive index current／next、dead mask、rate state、event private buffer、Trail point、Instance metadataの総和である。
- GPU persistentは`ParticleAttributesA/B`、`AliveIndicesA/B`、`DeadIndices`、`SubEmitterReady/Next`、parameter、counter、indirect argument、persistent Trailを含む。
- GPU transientは`EventCandidates`、sort key／index／scratch、prefix-sum scratch、draw packet scratchの同時live peakである。Render Graph aliasを見込んだ値とaliasなしworst caseの両方をReceiptへ残し、Hard cap判定は検証済みalias後peakを使う。
- Event memoryは`max_events_per_tick * compiled_payload_stride`を各buffer面へ計上する。payload strideは16 byte alignment、最大256 byteである。
- Texture、Mesh、Material、Shader PackageはVFX memoryへ重複計上せず、それぞれのAsset Domainへ一度だけchargeする。

Compiler estimateより実測peakが大きいArtifactは`VfxMemoryEstimateMismatch`でQualification失敗とし、余剰Global budgetで黙って許可しない。

### 16.3 Mobile candidate ceiling

実機Qualification前にProduction Capabilityを誇張しないため、次を初期ceilingとする。実機Receiptは値を下げられるが、上げる場合はProfile改訂と再Qualificationを必要とする。

| Quality | Backend | Active Emitter | Alive Project／Emitter | Spawn/s Project／Emitter | Burst/tick Project／Emitter | Trail point Project／Trail | Event/tick Project／Emitter |
|---|---|---:|---:|---:|---:|---:|---:|
| `mobile_baseline` | CPU | 128 | 16,384／2,048 | 5,000／1,024 | 2,048／512 | 4,096／32 | 0 |
| `mobile_baseline`限定Profile | GPU | 128 | 65,536／16,384 | 32,768／8,192 | 8,192／2,048 | 32,768／64 | 8,192／1,024 |
| `mobile_standard` | CPU | 192 | 32,768／4,096 | 10,000／2,048 | 4,096／1,024 | 8,192／48 | 16,384／2,048 |
| `mobile_standard` | GPU | 512 | 262,144／65,536 | 131,072／32,768 | 32,768／8,192 | 65,536／128 | 32,768／2,048 |
| `mobile_high` | CPU | 256 | 65,536／8,192 | 20,000／4,096 | 8,192／2,048 | 16,384／64 | 32,768／4,096 |
| `mobile_high` | GPU | 768 | 524,288／131,072 | 262,144／65,536 | 65,536／16,384 | 131,072／192 | 65,536／4,096 |

| Quality | CPU VFX memory | GPU persistent＋transient | Particle Light | VFX P95 CPU／GPU |
|---|---:|---:|---:|---:|
| `mobile_baseline` | 12 MiB | 限定Profileだけ16＋8 MiB | 0 | 0.40／0.40 ms |
| `mobile_standard` | 20 MiB | 48＋24 MiB | Project 8／Emitter 2 | 0.60／0.60 ms |
| `mobile_high` | 32 MiB | 96＋48 MiB | Project 16／Emitter 4 | 0.75／0.75 ms |

Mobile aggregate CPU、GPU working set、process physical footprint、thermal ruleはモバイル規約を同時に満たす。VFX ceiling内でもaggregate budgetを超えるPackageはCookを拒否する。

### 16.4 Overdraw

Profile buildでpipeline statisticsが利用できる場合、`particle pixel shader invocations / internal output pixel`をoverdraw ratioとする。非対応環境ではdiagnostic runだけ1/4 resolution accumulation passを使う。

| Ratio | 動作 |
|---|---|
| `<=4.0` | 合格 |
| `>4.0` | warning、Cost panelへ表示 |
| `>8.0` | AI proposal／Cook拒否 |
| Stress fixture P95 `>8.0` | C2 Promotion失敗 |

### 16.5 Runtime overflowとdegradation

- Editor preview、AI ChangeSet、CookはBudget超過を`VfxBudgetExceeded`で拒否し、黙ってspawn scaleを下げない。
- Shippingの突発的超過だけ、`priority desc, screen_influence desc, emitter StableId asc, particle_spawn_id asc`で末尾spawnを生成しない。
- drop数は`ParticleSpawnDropped{emitter,tick,reason,count}`へ集計し、次tickへ繰り越さない。
- Quality variant切替は同じexecution backend内のspawn rate、Material、Trail、Light、collision品質をframe boundaryで変更する。
- active Particle stateをCPUとGPU間で移送しない。Backend fallbackが必要な場合、新Instanceからfallback Artifactを使い、既存Instanceはdrainまたは明示restartする。
- Thermal governorはVFX Qualityを一段ずつ下げられるが、Project dataから無効化できない。

## 17. Asset promotion、Hot Reload、Device fault

### 17.1 Promotion

- Source CommitとVFX Cookを同一transactionにしない。
- VFX Artifact closureはGraph、Curve、Texture、Mesh、Material、Shader Package、Target／Quality Manifestを含む。
- Closure全体がReadyになってからRuntime規約のAsset generationとしてpromotionする。
- CPU ArtifactとRenderer shader／pipeline interface hashが一致しないgenerationを部分promotionしない。

### 17.2 Existing Instance

- 新Artifact promotion後に作るInstanceは新generationを使う。
- 実行中Instanceは旧Artifact leaseをCompleteまで保持する。
- parameter-only調整は`SetVfxParameterV1`で行い、Source Graph hot swapでalive Particle layoutを置換しない。
- Editor PreviewはGraph変更後に明示Restartし、新Artifactを使う。Restart前の画面を新仕様の結果として表示しない。
- old generationは全CPU job、RenderSnapshot、GPU submission serial、Instance leaseが終了してからretireする。

### 17.3 Device loss／surface loss

- GPU device fault時は新規GPU VFX submitを停止し、GPU Particle bufferの回収を試みずRenderer recoveryへ従う。
- device recreate後、persistent ambient Systemはdescriptorからrestartする。
- one-shot GPU VFXを再発火してGameplay eventを重複表示しない。
- CPU VFX stateはWorld lifecycleが維持される場合に保持できるが、surface不在中はRender packetを生成しない。
- Mobile backgroundでGPU activityを停止し、復帰時はsurface generationとArtifact leaseを再検証する。

## 18. AI Authoring、自然言語、手動／C++拡張

### 18.1 Capability

```text
capability.vfx.system_v1
capability.vfx.particle_cpu_v1
capability.vfx.particle_gpu_v1
capability.vfx.sprite_2d_v1
capability.vfx.billboard_3d_v1
capability.vfx.trail_v1
capability.vfx.mesh_ribbon_v1
capability.vfx.visual_collision_v1
capability.vfx.particle_light_v1
capability.vfx.bake_cache_v1
capability.vfx.extension_operator_v1
```

AIはEngine Backend名、descriptor、UAV、Vulkan barrier、Metal encoder、thread countを選ばない。Game上の目的、Dimension、規模、Target、Style、Gameplayとの関係からCapabilityを選び、9.2節のCompilerがexecution targetを確定する。

### 18.2 Authoring Operation

| Operation ID | 動作 | Risk |
|---|---|---|
| `operation.vfx.inspect_system` | Asset、Graph、Budget、Artifact、Diagnosticを読む | R0 |
| `operation.vfx.validate_changeset` | Schema、Graph、Capability、Costを検査 | R0 |
| `operation.vfx.preview_changeset` | fixed seedでPreview／差分／Costを生成 | R0 |
| `operation.vfx.create_system` | System Documentを作成 | R2 |
| `operation.vfx.create_emitter` | 型付きEmitterを追加 | R2 |
| `operation.vfx.update_emitter`／`delete_emitter` | Emitterを変更／削除 | R2 |
| `operation.vfx.add_node`／`update_node`／`delete_node` | closed Catalog Nodeを編集 | R2 |
| `operation.vfx.connect_nodes`／`disconnect_nodes` | 型付きEdgeを編集 | R2 |
| `operation.vfx.set_curve`／`set_gradient` | keyとinterpolationを編集 | R2 |
| `operation.vfx.set_output` | Renderer bindingを設定 | R2 |
| `operation.vfx.generate_fallback` | Target別fallback案を作る | R2 |
| `operation.vfx.capture_bounds` | Preview計測からfixed bounds案を作る | R1。Commitは別R2 |
| `operation.vfx.run_qualification` | read-only fixture reportを生成 | R1 |
| `operation.vfx.propose_extension_operator` | Project C++／portable shader拡張を提案 | R4 |

Write Operationは`ProjectChangeSet`を生成するだけで、live Particle buffer、GPU resource、ProjectRevisionを直接変更しない。

### 18.3 Level 0で確認する内容

| 不足事項 | 扱い |
|---|---|
| Effectの意味。炎、煙、魔法、雨、爆発、軌跡等 | High Impact |
| Sceneが2D、3D、Hybridのどれか | 既存Sceneから一意なら自動、曖昧ならBlocking |
| 常時表示か一回だけか、概算同時数 | High Impact |
| Mobileを含むTarget | Project Profileから取得、未設定ならBlocking |
| Gameplay判定を必要とするか | Blocking。必要ならGameplay／Collision Entityを別設計 |
| Realistic、Toon、Pixel等のStyle | Visual Style Profileから取得、未設定ならHigh Impact |

初心者へCPU／GPU、thread、buffer、shader model、sort algorithmを質問しない。未指定の小規模EffectはC1 CPU、固定seed Preview、`premultiplied_alpha`またはEffectに適したadditive、Particle Lightなしを仮定し、理由と変更影響をDecision Ledgerへ記録する。

### 18.4 手動編集

- Stack、Graph、Inspector、Curve、Timelineの編集は同じtyped Operationへ変換する。
- Source JSONを外部IDEで変更した場合も三者比較、Schema、Graph、Budget、Cookを経てCommitする。
- 人間lockしたNode、Curve、Parameter、EmitterをAIは変更しない。
- Graph全置換ではなくStableIdを保った最小Operationを優先する。

### 18.5 C++とShader拡張

C1ではProject custom operatorを許可しない。C2の`VfxExtensionOperatorV1`は次をすべて満たす場合だけ使用できる。

- MCDで入力、出力、Stage、Dimension、attribute read／write、parameter range、scratch byte、determinism classを宣言する。
- CPU実装はNativeGameModuleのC++23 functionとしてbuildし、bounded SoA spanだけを受け取る。
- function内allocation、World query、Physics call、file／network、logging、GPU API、static mutable stateを禁止する。
- GPU実装はRenderer規約のportable HLSL subsetへoffline compileし、全対象Targetでvalidationする。
- `portable`宣言にはCPU referenceとGPU implementationの両方を必須とする。
- AI生成SourceはSource sandbox、review、test、Promotionを通し、R4承認なしに有効化しない。
- Backend固有Operatorが必要な場合はProject AssetではなくEngine Node Catalog変更として別ADRを必要とする。

Project C++は通常、Particle bufferを直接操作せず、VFX Command、Parameter、Presentation Eventを生成する。これで表現できず、Profileでhot pathまたは固有表現の必要性が証明された場合だけExtension Operatorを使う。

## 19. Editor UXとAI統合

VFX Editorは独自MiraUI上のDock可能Panelとして実装し、次を同じWorkspaceまたは別Windowで配置できる。

- System／Emitter一覧。
- Beginner Module Stack。
- Advanced typed Graph。
- Inspector。
- Curve／Gradient Editor。
- Timeline、Play、Pause、Step、Restart、Scrub。
- 2D Canvas Preview／3D Scene Preview切替。
- Seed、Quality、Target、CPU／GPU Artifact表示。
- Bounds、velocity、spawn shape、collision proxy overlay。
- Particle count、spawn／drop、CPU／GPU time、memory、overdraw、draw call、sort cost。
- Compile／Validation／Budget Diagnostic。
- AI Partner Panelと提案Diff。

StackとGraphは別Sourceを持たない。Stackで表現できないGraphは`Advanced` badgeを表示するが、Stackへ戻すためにNodeを破棄しない。

Previewの初期値はseed固定、60 Hz、Reference Qualityである。random seed modeへ切り替えた場合も現在seedを表示し、問題を再現できるようにする。Preview、Cost、Cooked ArtifactのTarget／Qualityが異なる場合は常時表示する。

AI Semantic Snapshotは選択System／Emitter／Node、型、range、Diagnostic、Budget差、lock、current ProjectRevisionをStableId付きで提供する。AIがscreen座標でNodeやCurveを操作しない。

## 20. DiagnosticとFailure

### 20.1 Closed Diagnostic ID

```text
VfxSchemaInvalid
VfxMissingReference
VfxGraphCycle
VfxTypeMismatch
VfxStageViolation
VfxDimensionMismatch
VfxCapabilityUnavailable
VfxNodeCatalogMismatch
VfxBudgetExceeded
VfxMemoryExceeded
VfxMemoryEstimateMismatch
VfxOverdrawExceeded
VfxBoundsInvalid
VfxBoundsEscape
VfxCollisionProxyOverflow
VfxCompileFailed
VfxShaderCompileFailed
VfxArtifactInterfaceMismatch
VfxArtifactStale
VfxInstancePoolFull
VfxSpawnDropped
VfxEventOverflow
VfxGpuCounterOverflow
VfxGpuTimelineGap
VfxGpuFault
VfxExtensionRejected
VfxDeviceRecoveryRestarted
```

Diagnosticは`system_id`、`emitter_id`、optional Node／Field ID、Project revision、Artifact hash、Target、Quality、actual、limit、remediation IDを持つ。表示名や自由文だけで対象を識別しない。

### 20.2 Failure policy

| Failure | Source／Authoring | Runtime |
|---|---|---|
| Schema、Graph、型、Dimension不正 | ChangeSet全体reject | Artifactをloadしない |
| Capability／fallback不足 | 対象Target Cook失敗 | Instanceを開始しない |
| Budget／memory／overdraw超過 | Preview／AI proposal／Cook reject | 突発spawnだけ規定順でdrop |
| Artifact memory estimateより実測peakが大きい | Qualificationとpromotionをreject | DevelopmentはInstanceをFaulted、Shippingは未昇格Artifactを含めない |
| Shader compile／validation失敗 | 該当Asset generationをReadyにしない | last valid generationをDevelopmentで明示継続 |
| Artifact interface hash不一致 | promotion reject | 旧generationを維持 |
| CPU NaN／Inf | Preview失敗、Node／Particleを特定 | InstanceをFaulted、World不変 |
| GPU counter overflow | Qualification失敗 | 新規spawn停止、Instance fault、counter記録 |
| Instance pool full | Cost validation失敗 | lowest-priority new spawnを拒否 |
| GPU device fault | Source不変 | Renderer recovery、ambient restart、one-shot非再発火 |
| Extension違反 | Source Promotion拒否 | Extensionをloadしない |

Developmentのerror material、bounds overlay、counter readbackはDiagnostic buildだけに許可し、Shippingの通常pathへ含めない。

## 21. Test、Qualification、完了条件

### 21.1 Contract／Compiler

- MCD→JSON Schema／C++／Provider projectionのround-trip。
- unknown field、duplicate ID、missing reference、cycle、type mismatch、Stage違反、Dimension違反のnegative fixture。
- Node StableId順canonical compileとArtifact hash再現。
- constant folding／dead-node elimination前後のscalar reference一致。
- curve／gradient LUT error bound。
- Philox公式test vectorとCPU／HLSL／SPIR-V／MSL整数出力一致。
- C1全Nodeの2D／3D、CPU specialization matrix。
- C2全NodeのTarget Capability、fallback、shader validation matrix。
- CPU／GPU混在SystemのArtifact manifest、同一target Sub-emitter、cross-target Sub-emitter拒否。
- SoA／GPU bufferの計算値とinstrumented peakを比較し、過小estimateを必ず拒否。
- malformed／adversarial Graphに対するfuzz、timeout、memory cap。

### 21.2 CPU

- Point、area、burst、loop、pause、drain、parameter update、prewarm。
- 1、255、256、257、8,192 Particleのchunk boundary。
- fixed seedを100回実行し、count、Spawn ID、attribute hash一致。
- `lifetime_seconds`のstep量子化、Spawn時age 0、最初のUpdate時age 1、death Event後compactの境界。
- scalar／optimized kernel bitwise一致。
- stable compaction、job count変更、worker completion順変更でも一致。
- update中のheap allocation 0、out-of-bounds 0、use-after-free 0。
- 10分soakで32 MiBとP95 0.75 ms以内。

### 21.3 GPU／Renderer

- Reset→Update→Compact→Event→Spawn→Sort→Indirect→Drawのresource hazard validation。
- zero Particle、capacity丁度、capacity+1、counter wrap前拒否。
- D3D12 debug layer／DRED fixture、Vulkan validation、Metal API validation。
- CPU readbackなしでindirect countが正しい。
- 2D Sprite、3D Billboard、Flipbook、Trail、Mesh、Ribbon、Lightのgolden。
- depth／SDF／proxy collisionのvisual tolerance。
- Proxy shape／capacity、最小TOI tie-break、1 hit上限、dynamic overflow順、Snapshot generation mismatch拒否。
- `SubEmitterReady/Next`で同step再入がなく、quota超過Eventと未使用予約IDを規定どおりdrop／gap化する。
- resize、surface loss、device fault、quality切替、Artifact retire。
- 120 Hzで同一Snapshotを再描画してもstepが増えず、30 fpsで2 recordを順序どおり処理し、9 tick gapで規定restart／non-replayになる。
- persistent 128 MiB＋transient 64 MiB、P95 simulation＋draw 0.75 ms、overdraw Gate。

### 21.4 Reference Effect

| Fixture | 必須検証 |
|---|---|
| 2D hit spark | Burst、additive、fixed seed、pixel／non-pixel camera |
| 2D fire／smoke | rate、Flipbook、Color／Size over Life、layer order |
| 2D trail | moving emitter、local／world、64 point上限 |
| 3D hit spark | Billboard、velocity alignment、depth sort |
| 3D fire／smoke | alpha、soft depth C2、bounds、overdraw |
| Rain／snow | large bounds、quality scaling、Mobile thermal |
| Magic projectile | authoritative projectileとPresentation VFX分離 |
| Explosion | one authoritative EventからAudio／Camera／VFXを独立配送 |
| GPU mesh／ribbon | indirect draw、sort、C2 Budget |

### 21.5 AI／Editor／Asset

- 自然言語から2D／3D Effectを作り、同じGraphをStack／Graph／Inspectorで往復。
- AI proposalと手動操作が同じcanonical ChangeSetになる。
- lock、stale revision、Budget、unsupported Target、Gameplay collision要求を正しく拒否／質問する。
- invalid GraphをCommitせず、Undo／Redo 10,000回後にDocument hash一致。
- Source変更、Cook失敗、last valid generation、Preview restart、old Artifact retire。
- Extension Source sandbox、R4 approval、forbidden API scan、Target compile。

### 21.6 Promotion Gate

#### C0完了

- Schema、Node Catalog、IR、Compiler descriptor、Diagnostic、Cost modelが生成／round-trip testに合格する。
- VFX CoreがGraphics Backend、Physics Backend、Editor UIへ依存しないnegative dependency testに合格する。
- invalid／adversarial GraphでProject stateを失わない。

#### C1 Production完了

- Windowsの2D／3D CPU Reference Effectがvisual、determinism、memory、performance Gateを満たす。
- Mobile Baselineの2D CPU vertical sliceが実機memory、thermal、10分performance、2時間enduranceに合格する。
- AI、手動Editor、Project C++ Commandの三経路が同じVFX Asset／Runtime結果へ収束する。
- Particleをauthoritative Gameplayへ使う依存をSchema／link／runtime testで拒否する。

#### C2 Production完了

- D3D12、Vulkan、MetalのGPU Artifactが各実機／validation／shader／device recovery Gateに合格する。
- Mesh、Ribbon、visual collision、Sub-emitter、Particle Light、quality variantがBudget内でReference Effectに合格する。
- GPU readbackなし、Shipping runtime compilerなし、Backend native型漏出なしをbinary／dependency scanで証明する。

## 22. Version、互換性、更新

- `VfxSystemDocumentV1`、Node Catalog、IR、Artifactは別々のversionを持つ。
- Source schema major変更は明示migration tool、before／after hash、semantic diff、golden更新を必要とする。
- unknown Node、unknown attribute、unknown blend、unknown execution policyを既定値へ置換しない。
- Node削除は少なくとも一つのProject release cycleでdeprecated diagnosticを持ち、migrationが全fixtureに合格してから行う。
- ArtifactはCompiler build、Target、Quality、Node Catalog、Shader toolchainのいずれかが変われば再Cookする。
- Unity、Unreal、GodotのParticle Asset互換importはC1／C2対象外である。将来Importerを作る場合も外部意味をMiraikanai Graphへloss report付きで変換し、外部Runtimeを埋め込まない。

## 23. 公式資料と採用根拠

### 23.1 有名Engine

- [Unity 6.0: Choosing your particle system solution](https://docs.unity3d.com/cn/current/Manual/ChoosingYourParticleSystem.html)
- [Unity 6.0: Particle System modules](https://docs.unity3d.com/kr/current/Manual/ParticleSystemModules.html)
- [Unreal Engine 5.8: Niagara Overview](https://dev.epicgames.com/documentation/en-us/unreal-engine/overview-of-niagara-effects-for-unreal-engine)
- [Unreal Engine 5.8: Emitter Settings](https://dev.epicgames.com/documentation/unreal-engine/emitter-settings-reference-for-niagara-effects-in-unreal-engine)
- [Unreal Engine: Niagara Scalability and Best Practices](https://dev.epicgames.com/documentation/en-us/unreal-engine/scalability-and-best-practices-for-niagara)
- [Unreal Engine 5.8: Niagara Renderers](https://dev.epicgames.com/documentation/unreal-engine/render-module-reference-for-niagara-effects-in-unreal-engine?lang=en-US)
- [Godot 4.6: GPUParticles2D](https://docs.godotengine.org/en/4.6/classes/class_gpuparticles2d.html)
- [Godot 4.6: CPUParticles2D](https://docs.godotengine.org/en/4.6/classes/class_cpuparticles2d.html)
- [Godot 4.6: GPUParticles3D](https://docs.godotengine.org/en/4.6/classes/class_gpuparticles3d.html)
- [Godot 4.6: CPU／GPU 3D Particle conversion](https://docs.godotengine.org/en/4.6/tutorials/3d/particles/creating_a_3d_particle_system.html)

### 23.2 Graphics APIとAlgorithm

- [Microsoft: ID3D12GraphicsCommandList::ExecuteIndirect](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-executeindirect)
- [Microsoft: Direct3D 12 UAV Counters](https://learn.microsoft.com/en-us/windows/win32/direct3d12/uav-counters)
- [Vulkan: vkCmdDrawIndirect](https://registry.khronos.org/vulkan/specs/latest/man/html/vkCmdDrawIndirect.html)
- [Apple Metal: MTLDrawPrimitivesIndirectArguments](https://developer.apple.com/documentation/metal/mtldrawprimitivesindirectarguments)
- [Apple Metal: Compute passes](https://developer.apple.com/documentation/metal/compute-passes)
- [Apple Metal: Indirect command encoding](https://developer.apple.com/documentation/metal/indirect_command_encoding)
- [Random123／Philox](https://github.com/DEShawResearch/random123)
- [NVIDIA GPU Gems 3: High-Speed, Off-Screen Particles](https://developer.nvidia.com/gpugems/gpugems3/part-iv-image-effects/chapter-23-high-speed-screen-particles)

外部資料は実現可能性、API semantics、性能上の注意を確認する一次資料である。Miraikanai固有のAsset、Node、IR、Budget、AI Operation、Diagnostic、Promotion Gateは本書を正本とする。
