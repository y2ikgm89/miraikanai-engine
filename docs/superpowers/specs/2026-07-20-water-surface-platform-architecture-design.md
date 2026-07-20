# Miraikanai Engine Water Surface Platformアーキテクチャ規約

- 最終更新日: 2026-07-20
- 状態: 設計レビュー用正式仕様
- 対象: C0 Foundation、C1 Bounded Water、C2 Production Water、C3 Research
- 関連規約:
  - [2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)
  - [AI可読LODアーキテクチャ規約](./2026-07-20-ai-readable-lod-architecture-design.md)
  - [Environment Platform／AI Authoring規約](./2026-07-20-environment-platform-ai-authoring-architecture-design.md)
  - [Rendering／Render Graph規約](./2026-07-19-rendering-render-graph-architecture-design.md)
  - [Particle／VFX Platform規約](./2026-07-20-particle-vfx-architecture-design.md)
  - [Physics Platform規約](./2026-07-20-physics-engine-architecture-design.md)
  - [Collision／Collider規約](./2026-07-19-collision-collider-architecture-design.md)
  - [Asset Pipeline規約](./2026-07-19-asset-pipeline-content-packaging-design.md)
  - [Runtime連携規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)

## 1. 結論

Miraikanai EngineのWaterは、一般Transparent Material、VFX、Physicsへ機能を分散させず、**型付きWater Sourceから描画Artifact、CPU Query Artifact、Collision／Gameplay用Volumeをoffline cookする独立Capability**として実装する。

- C1は小規模Scene向けの`bounded_plane`／`mesh_region`、静的水位、Material波、Environment IBL、平面Water Volumeを提供する。
- C2は`lake`／`spline_river`／`ocean`、解析的波、流れ、水深、Reflection、Underwater、CPU Surface Query、浮力連携を提供する。
- C3はFFT ocean、shallow-water solver、相互作用する流体、large-world streamingをResearch Capabilityとして隔離する。
- 飛沫、泡Particle、雨滴、航跡のPresentationはParticle／VFXが所有し、Water PlatformはVFX simulationを所有しない。
- 浮力、泳ぎ、Damage、Navigation costはPhysics／Gameplayが所有し、GPU水面、foam、screen-space reflectionを正規判定へ使用しない。
- Water Source、Editor、AI、Project C++へD3D12、Vulkan、Metalのnative resource、shader、descriptorを公開しない。

Unity HDRPのWater SurfaceとCPU search、Unreal EngineのWater Body／Water Mesh／Buoyancy分離、Godotで一般的なMesh＋Shaderの軽量経路を参考にする。ただしAsset名、UI、Shader、file形式をコピーせず、MiraikanaiのCapability、ChangeSet、Render Graph、Target別Cookへ統合する。

## 2. 決定権と境界

| 主題 | 正本 |
|---|---|
| Water Source、Body、Surface、Volume、Wave、Flow、Depth、Query、Artifact、Budget、Qualification | 本書 |
| LOD Intent、`WaterLodProfileV1` envelope、metric、transition、AI LOD Operation、Qualification Receipt | LOD規約 |
| GPU resource、pass、barrier、reflection view、history、device loss | Rendering規約 |
| Water Material IR、Shading Model、Visual Style | Material／Visual Style／AI Authoring規約 |
| Sky radiance、sun／moon、Environment IBL、地上Fog／Exposure | Environment Platform規約 |
| 飛沫、泡、雨滴、航跡Particle | Particle／VFX規約 |
| 浮力Force、Character swimming、Body motion、Save／Replay | Physics／Gameplay規約 |
| Collision shape、Trigger／Contact、Water Volume overlap | Collision規約 |
| Source／Derived／Cooked Asset、promotion、package | Asset規約 |
| Tick、Snapshot、Command、memory parent domain、global frame budget | Runtime規約 |

Water RuntimeはWorld、Physics Backend、Graphics Backend、Editor widgetを直接queryしない。`WaterSnapshotV1`、`WaterQueryPortV1`、`PresentationEvent`、Asset leaseを境界にする。

## 3. 成熟度

| Level | 到達点 |
|---|---|
| C0 | Schema、validator、cost model、Material interface、Artifact envelope、flat query fixture、Editor preview contract |
| C1 | `bounded_plane`／`mesh_region`、静的水位、dual normal scroll、Fresnel、absorption tint、Environment IBL、flat Water Volume、Splash Event接続 |
| C2 | `lake`／`spline_river`／`ocean`、解析的波、current map、水深、foam mask、probe＋SSR、underwater、CPU Surface Query、buoyancy連携、`WaterLodProfileV1` |
| C3 | FFT spectrum、shallow-water solver、dynamic displacement field、erosion、large-world water、networked fluid state |

C1では一般Mesh／Materialだけでも同じ見た目を作れるが、正式Water Sourceを使用したSceneだけがC2へlosslessに昇格できる。一般MaterialからWater Sourceを推測変換しない。

## 4. ModuleとDirectory

```text
schemas/mirakan/water/
engine/water/
  contracts/
  core/
    bodies/
    waves/
    flow/
    depth/
    budget/
  compiler/
  runtime/
  query/
  render/
  diagnostics/
authoring/water/
editor/panels/water/
tools/water_compiler/
tools/water_qualification/
tests/water/
```

| Target | 責務 |
|---|---|
| `mirakan_water_contracts` | value type、enum、Snapshot、Query、Diagnostic |
| `mirakan_water_core` | body意味、wave／flow評価、priority、budget |
| `mirakan_water_compiler` | Source validation、mesh／depth／query Artifact生成 |
| `mirakan_water_runtime` | instance、Snapshot、quality、lifecycle |
| `mirakan_water_query` | CPU surface／depth／flow query。Physics Backend非依存 |
| `mirakan_water_render` | Water packet、Material binding、Render Graph template |
| `mirakan_water_authoring` | ChangeSet projection、preview、cost、cook request |
| `mirakan_water_qualification` | golden、CPU／GPU一致、performance、fault fixture |

`mirakan_water_query`はRenderer、D3D12、Vulkan、Metalへ依存しない。`mirakan_water_render`はPhysics Worldへ依存しない。

## 5. Source Object

### 5.1 `WaterSystemDocumentV1`

```text
WaterSystemDocumentV1
  schema_version
  system_id
  body[]
  material_profile[]
  quality_profile_id
  target_policy
  fallback_variant[]
```

### 5.2 `WaterBodyV1`

```text
WaterBodyV1
  body_id
  body_kind
  transform
  boundary_source
  surface_profile_id
  volume_profile_id
  wave_profile_id
  flow_profile_id
  depth_source
  priority
  layer_mask
  gameplay_binding
```

`body_kind`は次のclosed enumとする。

| Kind | Level | 境界 |
|---|---|---|
| `bounded_plane` | C1 | finite rectangle |
| `mesh_region` | C1 | manifold surface mesh。vertical wallを水面として受理しない |
| `lake` | C2 | closed polygon、hole最大16 |
| `spline_river` | C2 | non-self-intersecting spline、幅／水位／flow key |
| `ocean` | C2 | camera-relative clipmap、C1の10 km World範囲内 |

同じ位置へ複数Bodyが重なる場合は`priority desc, body_kind specificity desc, body_id asc`で一つを選ぶ。同priorityの同種Bodyが体積交差するSourceはcook errorとする。

### 5.3 SurfaceとVolume

`WaterSurfaceProfileV1`はbase color、absorption、scattering、roughness、normal texture、normal scale、normal scroll、Fresnel F0、foam Material referenceを持つ。距離はmeter、速度はm/s、吸収／散乱は`m^-1`、色はlinear RGBとする。

`WaterVolumeProfileV1`は水位以下のclosed volume、priority、underwater profile、Gameplay Surface IDを持つ。C1のVolumeは平面とfinite boundaryから生成し、visual vertex displacementをColliderへ反映しない。

## 6. WaveとFlow

### 6.1 C1

C1の見た目は最大2枚のnormal textureを独立した方向／速度でscrollし、静的surface meshを変位させない。Water VolumeとGameplay水位は平面である。

### 6.2 C2解析的波

`DirectionalWaveSetV1`は最大8成分を持つ。

```text
DirectionalWaveComponentV1
  direction_xz: normalized vec2f
  amplitude_m: f32
  wavelength_m: f32
  phase_rad: f32
  steepness: f32
```

範囲はamplitude `[0, 20] m`、wavelength `[0.25, 2,000] m`、steepness `[0,1]`とする。位相速度はProject gravity `g`からdeep-water dispersion `omega=sqrt(g*k)`、`k=2*pi/wavelength`で求める。CPU QueryとGPU vertex evaluationは同じ生成定数と式を使う。

ShippingのCPU／GPU一致fixtureはReference sample 65,536点で高さ最大誤差2 mm、normal角度最大0.25°を満たす。超えるTarget Shader variantをProductionへ昇格しない。

`FlowProfileV1`はconstant flowまたはCook済み2D current mapを持つ。current mapのRGはnormalized方向、Bは`[0,1]` strength、profileが最大速度m/sを定める。TextureをGameplayが直接sampleせず、CPU Query Artifactへ同じfieldをquantizeして保存する。

## 7. Depth、Reflection、Underwater

- C1 depthはBodyごとのconstant depthまたはVolume底面までの幾何距離をEditor previewだけで表示する。Gameplay Queryはflat surface overlapだけを返す。
- C2 depthはterrain／mesh sourceからoffline cookしたfieldを使用し、Runtime scene depthを正規水深にしない。
- C1 reflectionはEnvironment IBLだけとする。
- C2 MediumはReflection Probe、HighはProbe＋SSRを許可する。SSR欠落をGameplay、AI visibility、水中判定へ使わない。
- UnderwaterはWater Volume overlapでViewを選び、absorption、scattering、fog、waterline compositeを登録済みRender Graph templateで描画する。
- 複数Volume overlap時はBody選択順を使う。Cameraが境界から5 cm以内では前frameのBodyを保持し、10 cm外へ出た時に解除するhysteresisを持つ。

## 8. CPU QueryとGameplay

```text
WaterQueryRequestV1
  query_id: u64
  position_world_m: vec3f
  body_layer_mask: u64
  snapshot_tick: u64

WaterQueryResultV1
  status
  body_id
  surface_position_world_m
  surface_normal_world
  flow_velocity_mps
  depth_m
  query_generation
```

`status`は`Success | OutsideWater | UnsupportedAtLevel | StaleSnapshot | InvalidRequest | QueueFull | BackendFailure`とする。

- Physicsはfixed tick開始時にpublish済みCPU Query Artifactだけを読む。
- GPU buffer、render depth、SSR、foam、VFX collisionをreadbackしない。
- C1はflat surface／zero flow、C2は解析的波／flow／depthを返す。
- 浮力はPhysics側がQuery結果からForceを計算する。Water PlatformはBodyへForceを加えない。
- Query上限はWindows 16,384／tick、Mobile 4,096／tick。超過は`QueueFull`であり前frame値へ黙ってfallbackしない。
- SaveはBody Asset revision、Gameplay water state、解析波の開始tickを保存し、GPU texture／foam／SSR historyを保存しない。

## 9. RuntimeとRender Graph

```text
T90  Gameplay／Collision EventからWater presentation commandを構築
T100 Water RuntimeがBody／Query snapshotを構築
T110 immutable WaterSnapshotV1をpublish
R20  Water packet／visibility／LOD
R30  registered Water pass templateをRender Graphへ追加
R40  record
R50  submit
```

C1 Waterは3D Transparent stageで描画する。C2は次の登録済み順序を使う。

```text
Opaque／Lighting
-> Environment
-> Water Depth／Surface
-> Underwater／Waterline Composite
-> 3D Transparent／World VFX
```

Water passはscene color／depthを読む場合にaccessを宣言し、implicit framebuffer readへ依存しない。Water resource generationをSnapshotへ含め、device loss後はMaterial、mesh、query Artifact、historyを順に再構築する。

`R20`はLOD規約の`WaterLodProfileV1`とTarget別`LodResolutionPlanV1`からsurface patch density、wave shading、reflection、underwater presentation、foam／spray VFX tierを選ぶ。CPU Surface Query、Water Volume、Gameplay water level、浮力、swimming、Damage、Navigation costをWater LODで変更しない。

## 10. VFX連携

`WaterInteractionEventV1`はauthoritative Collision／Gameplayが生成する。

```text
WaterInteractionEventV1
  event_id
  body_id
  position
  normal
  relative_velocity_mps
  interaction_kind
  vfx_seed
```

`interaction_kind`は`enter | exit | impact | wake_request`とする。VFXはこのEventからSplash、foam、ripple、wakeを生成できるが、その結果をWater Query、Damage、浮力へ戻さない。

## 11. BudgetとFallback

| 項目 | C1 Desktop | C2 Desktop | C2 Mobile Standard |
|---|---:|---:|---:|
| Active Water Body | 32 | 256 | 64 |
| Visible surface | 8 | 32 | 8 |
| Water GPU persistent | 16 MiB | 96 MiB | 32 MiB |
| Water GPU transient peak | 16 MiB | 64 MiB | 24 MiB |
| CPU query Artifact | 4 MiB | 32 MiB | 8 MiB |
| GPU P95 soft cap | 0.35 ms | 1.00 ms | 0.75 ms |

Hard上限超過Sourceはcookを拒否する。Runtime visibility超過は`semantic_priority desc, projected_coverage_px_q16 desc, body_id asc`の末尾Presentationを描画から除外してDiagnosticを発行するが、Water VolumeとGameplay Queryは維持する。

`WaterLodProfileV1`はsurface patch density、波成分数、normal octave、SSR resolution、foam resolution、underwater sample数の順で明示variantを選ぶ。enter／exit hysteresis、minimum residency、C1 surface fallbackを持ち、Body boundary、Gameplay water level、authoritative flow、CPU Query、Visual Styleを変更しない。

## 12. AI／Editor

EditorはWater Body、spline、boundary、flow、depth、wave、Volume、reflection、underwater、budget、Query debugをtyped panelで編集する。AIへ公開するOperationは`CreateWaterBody`、`SetWaterBoundary`、`SetWaveProfile`、`SetFlowProfile`、`SetWaterMaterial`、`SetUnderwaterProfile`、`PreviewWaterCost`とする。

AIはshader source、Render pass、FFT size、GPU resource、Physics Forceを直接生成しない。C1 ProjectにC2 Bodyを提案した場合は必要Capability、Target、cost、fallbackをChangeSetへ明示する。

Water LODの説明、Policy提案、transition Preview、Validation、適用はLOD規約の`operation.lod.*`を使う。Water Operationが独自の距離閾値またはQuality variantを生成せず、`WaterLodProfileV1`のtyped payloadへ収束する。

## 13. Diagnostic

最低限のclosed IDを持つ。

- `WaterInvalidBoundary`
- `WaterBodyOverlapConflict`
- `WaterUnsupportedBodyKind`
- `WaterWaveOutOfRange`
- `WaterDepthCookFailed`
- `WaterQueryQueueFull`
- `WaterQueryStaleSnapshot`
- `WaterCpuGpuMismatch`
- `WaterRenderBudgetExceeded`
- `WaterResourceGenerationMismatch`
- `WaterFallbackMissing`

## 14. Qualificationと完了条件

### C1

- bounded pool、small river-like mesh、2D top-down water tileのgolden image。
- flat Water Volumeのenter／exit EventとSplash VFX。
- D3D12、Vulkan、MetalでMaterial interface hash一致。
- resize、surface loss、device loss後にWater resourceが復旧する。
- Reference sceneでGPU P95 0.35 ms、memory上限を満たす。

### C2

- lake、spline river、oceanを同一Sceneで描画し、priority overlapが規定順になる。
- 解析波のCPU／GPU一致が2 mm／0.25°以内。
- buoyancy fixtureがrender frame rateと独立したfixed tickで安定する。
- probe＋SSR、underwater、waterline、flow、depth、foamがgolden許容差内。
- surface LOD transition、hysteresis、camera cut、patch境界、residency missを検証し、LOD on／offでCPU Query、Volume、buoyancy、Gameplay結果が一致する。
- Water Interaction Event一件からVFX／Audioを独立配送し、Gameplayへ逆入力しない。
- Windows、Android、AppleのTarget別performance、thermal、device recoveryを通過する。

### C3着手Gate

C2が全TargetでProduction昇格し、FFT／shallow-waterの独立Threat Model、memory、determinism、fallback、Editor UX、Reference hardwareが承認されるまでC3 Node、Asset、MCDを公開しない。
