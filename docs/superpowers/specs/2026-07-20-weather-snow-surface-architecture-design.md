# Miraikanai Engine Weather／Snow Surfaceアーキテクチャ規約

- 最終更新日: 2026-07-20
- 状態: 設計レビュー用正式仕様
- 対象: Weather Presentation input、降雪VFX接続、静的／動的積雪、融雪、足跡表示
- 関連規約:
  - [2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)
  - [Particle／VFX Platform規約](./2026-07-20-particle-vfx-architecture-design.md)
  - [Water Surface Platform規約](./2026-07-20-water-surface-platform-architecture-design.md)
  - [Rendering／Render Graph規約](./2026-07-19-rendering-render-graph-architecture-design.md)
  - [Collision／Collider規約](./2026-07-19-collision-collider-architecture-design.md)
  - [Runtime連携規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)

## 1. 結論

雪は一つのParticle Effectとして実装せず、次の三層へ分離する。

1. `EnvironmentProfile`の風、降水量、温度が型付きWeather inputをpublishする。
2. Particle／VFXが降雪、吹雪、着地煙をPresentationとして描画する。
3. Snow Surfaceが静的Mask、動的積雪、融雪、圧雪、足跡の見た目をMaterialへ供給する。

摩擦、移動速度、足場、雪Damage等はGameplay／Physicsの`GameplaySurfaceState`が所有する。GPU積雪field、Particle数、Particle collisionを正規判定へ使用しない。

Unity、Unreal Engine、Godotと同様に降雪Particleと地表Materialを分離する一方、Miraikanaiでは手作業の暗黙接続ではなく、version付きWeather Snapshot、Presentation Event、Snow Surface Artifactで接続する。

## 2. 決定権

| 主題 | 正本 |
|---|---|
| Weather presentation field、Snow Source、Mask、dynamic field、stamp、melt、budget、qualification | 本書 |
| Particle spawn、simulation、renderer、collision、GPU Artifact | Particle／VFX規約 |
| Material IR、snow surface role、Visual Style | 2D／3D機能計画 |
| Render resource、compute pass、history、device loss | Rendering規約 |
| Contact／Trigger、foot placement、authoritative surface ID | Collision／Gameplay規約 |
| Tick、Event、Snapshot、Save、global budget | Runtime規約 |

本書は汎用気象simulation、雲生成、流体降水、気候モデルを所有しない。C1／C2のWeatherはbounded inputであり、Cloud／Atmosphere規約が見た目を別に解決する。

## 3. 成熟度

| Level | 到達点 |
|---|---|
| C0 | Weather／Snow schema、Material interface、static mask cook、cost model、preview、diagnostic |
| C1 | CPU降雪VFX、静的snow mask、world-up Material blend、authoritative contactからfootprint Decal／VFX、quality scaling |
| C2 | GPU降雪、turbulence、visual collision、paged dynamic snow field、accumulation、melt、compaction、bounded footprint stamp |
| C3 | deformable snow volume、height displacement／tessellation、avalanche、persistent large-world snow、coupled fluid／temperature simulation |

## 4. ModuleとDirectory

```text
schemas/mira/weather/
schemas/mira/snow/
engine/environment/weather/
engine/snow/
  contracts/
  core/
  compiler/
  runtime/
  render/
  diagnostics/
authoring/snow/
editor/panels/snow/
tools/snow_compiler/
tools/snow_qualification/
tests/snow/
```

| Target | 責務 |
|---|---|
| `mira_weather_contracts` | Weather inputとSnapshot value type |
| `mira_snow_contracts` | Snow Source、Artifact、stamp、Snapshot、Diagnostic |
| `mira_snow_core` | accumulation／melt／compaction規則、page選択、budget |
| `mira_snow_compiler` | static mask、receiver、page manifest、Material binding |
| `mira_snow_runtime` | dynamic field、stamp admission、Snapshot、Save seed |
| `mira_snow_render` | Material parameter、compute pass、debug view |
| `mira_snow_authoring` | ChangeSet、paint、preview、cost |

Snow RuntimeはPhysics World、VFX Particle buffer、Graphics Backendを直接queryしない。

## 5. Weather input

```text
WeatherPresentationProfileV1
  profile_id
  precipitation_kind
  precipitation_rate_mmph
  wind_velocity_world_mps
  air_temperature_c
  snow_accumulation_rate_mps
  snow_melt_rate_mps
  transition_seconds
```

`precipitation_kind`は`none | rain | snow | sleet`とする。範囲は降水量`[0,500] mm/h`、風速各axis `[-150,150] m/s`、温度`[-100,100] °C`、積雪／融雪速度`[0,0.01] m/s`、transition `[0,600] s`とする。

Weather変更は`T90_PresentationBuild`で`WeatherPresentationSnapshotV1`へ変換し、VFXとSnow Runtimeが同じgenerationを読む。Gameplay用Weather stateが必要な場合はGameplayDefinitionが独立したauthoritative値を所有し、Presentation Snapshotを逆入力にしない。

## 6. 降雪VFX

C1／C2のParticle実行契約はParticle／VFX規約を正本とする。本書はReference Effectの構成だけを定める。

```text
SnowfallVfxProfileV1
  system_asset_id
  camera_volume_extent_m
  density_scale
  flake_size_range_m
  fall_speed_range_mps
  wind_response
  quality_variant[]
```

- Emitterはcamera-relative finite Boxを使い、infinite boundsを禁止する。
- C1はCPU Billboard、gravity、drag、Color／Size over Lifeを使う。
- C2はGPU Billboard、turbulence、Depth／SDF／VFX proxy visual collisionを許可する。
- Particleが地面に当たった事実から積雪量、摩擦、足跡、Gameplay Eventを生成しない。
- Mobile BaselineはCPU低密度variant、Standard以上だけGPU variantを選択できる。
- 大きい雪片、吹雪、前景flakeは別Emitterへ分け、透明overdrawを一つのEmitterへ隠さない。

## 7. Snow Surface Source

```text
SnowSurfaceDocumentV1
  document_id
  receiver[]
  static_mask[]
  dynamic_field_profile_id
  material_profile_id
  quality_profile_id
  fallback_variant[]
```

`SnowReceiverV1`はreceiver Stable ID、geometry Asset、world bounds、surface layer、static mask、dynamic enabled、priorityを持つ。

C1 static maskはSource textureまたはEditor paint strokeからoffline cookする。Textureはlinear `R8_UNORM`、0がsnowなし、255が完全被覆である。UVがないMeshはworld-projected maskを明示選択しない限りcook errorとする。

Material blendは次を使用する。

- snow coverage
- world normalとup vectorのdot
- receiver allow mask
- height bias
- material-specific snow retention

world-up thresholdだけから雪を生成せず、static／dynamic coverageが0ならsnow layerを描画しない。C1はgeometry displacementを行わない。

## 8. C2 Dynamic Snow Field

`SnowDynamicFieldProfileV1`はworld-aligned paged atlasを定める。

| 項目 | Desktop | Mobile Standard |
|---|---:|---:|
| Page resolution | 128×128 | 128×128 |
| Texel size | 0.25 m | 0.50 m |
| Active page | 256 | 64 |
| Format | `R16G16_UNORM` | `R16G16_UNORM` |
| Channel R | coverage／depth normalized | 同左 |
| Channel G | compaction normalized | 同左 |

Desktop pageは32×32 m、Mobile pageは64×64 mを覆う。page keyはinteger world cell `(x,z)`とfield generationから決め、camera距離だけをidentityにしない。

Profileは`max_depth_m`を`[0.01,2.0] m`で必須とする。coverage／compactionの正規更新値は`uint16` Q0.16として計算し、GPU浮動小数点atomicへ依存しない。Weatherの一step分coverage deltaは`round_ties_to_even(clamp(rate_mps * fixed_step_seconds / max_depth_m, 0, 1) * 65535)`、stamp deltaはSourceのsigned Q0.16へcookし、各passで符号付き32-bit中間値へ加算して`[0,65535]`へsaturateする。これにより同じcommand列からD3D12、Vulkan、Metalで同じfield byte列を要求できる。

Update順は固定する。

```text
SnowResetNewPages
-> SnowApplyWeather
-> SnowApplyStamps
-> SnowResolveMelt
-> SnowBuildMaterialBindings
```

各値は`[0,1]`へsaturateする。降雪、融雪、stampを同じpassのunordered writeへせず、上記pass間のresource dependencyをRender Graphへ宣言する。

page上限を超えた場合は`receiver priority desc, camera distance asc, page key asc`で保持し、末尾をdynamic更新対象外にする。静的maskへ戻す明示fallbackがあるreceiverだけ描画を継続する。

## 9. Footprint／Interaction Stamp

Gameplay／Collisionはauthoritative contactから`SnowSurfaceInteractionEventV1`を生成する。

```text
SnowSurfaceInteractionEventV1
  event_id
  receiver_id
  position_world_m
  direction_world
  radius_m
  depth_delta
  compaction_delta
  stamp_asset_id
  presentation_seed
```

範囲はradius `[0.01,5] m`、delta `[-1,1]`とする。Desktopは1,024 stamp／tick、Mobileは256 stamp／tickを上限とし、`priority desc, event_id asc`の末尾をdropしてDiagnosticへ集計する。dropしたstampを次tickへ繰り越さない。

StampはPresentation専用である。Character friction、foot IK、movement speedはcontactの`Gameplay Surface ID`とauthoritative `GameplaySurfaceState`から決める。

## 10. Accumulation、Melt、Save

- accumulationはWeather Snapshotのrate、receiver retention、fixed presentation stepからcoverageへ加算する。
- air temperatureがProfile melt thresholdを超える場合だけmelt rateを適用する。
- compactionはstampだけが増加させ、自然降雪で減らさない。
- C2のdynamic fieldはPresentation stateであり、SaveへGPU pageを保存しない。
- Persistent snowを復元する場合はWeather profile、start tick、receiver static mask revision、bounded gameplay-authored snow eventsを保存し、load時にfieldを再生成する。
- C3 large-world persistent snowが必要になるまで全stamp履歴を無制限保存しない。

## 11. Material／Render Graph

`snow_surface` Material roleはbase color、normal、roughness、subsurface tint、sparkle、coverage、compactionを受け取る。C1はstatic texture binding、C2はstatic maskとdynamic page atlasを合成する。

Dynamic snow passはWorld opaque Materialより前に完了し、Materialが同frameの確定fieldだけを読む。Pixel-locked 2Dではdynamic atlasをsampleせず、Tile／Spriteの明示snow variantを使う。

device loss時はstatic mask、page manifest、新規empty fieldを復元し、保存済みWeather start tickからbounded warm-upを行う。復旧中にstale GPU pageをsampleしない。

## 12. BudgetとFallback

| 項目 | C1 Desktop | C2 Desktop | C2 Mobile Standard |
|---|---:|---:|---:|
| Snow receiver | 1,024 | 8,192 | 2,048 |
| Dynamic field persistent | 0 | 24 MiB | 6 MiB |
| Dynamic transient | 0 | 8 MiB | 4 MiB |
| Snow update GPU P95 | 0 | 0.35 ms | 0.25 ms |
| Stamp／tick | Decal／VFX budget内 | 1,024 | 256 |

降雪ParticleはParticle／VFX budgetへ別計上する。同じbyteをSnowとTexture Domainへ二重計上せず、Parent GPU capの合算検査を行う。

Fallback順はdynamic footprint detail、field update distance、sparkle、normal detail、降雪densityとする。Gameplay Surface ID、static coverage、Visual Styleを変更しない。

## 13. AI／Editor

EditorはWeather Preview、Snow Receiver、static mask paint、dynamic page、coverage、compaction、stamp、budget、overdrawを表示する。AI Operationは`SetWeatherPresentation`、`CreateSnowReceiver`、`PaintStaticSnowMask`、`SetSnowMaterial`、`EnableDynamicSnow`、`PreviewSnowCost`とする。

AIはGPU texture、Render pass、Particle buffer、Gameplay frictionを直接編集しない。動的積雪を有効にするChangeSetはTarget Capability、page数、memory、GPU cost、fallbackを提示する。

## 14. Diagnostic

- `WeatherValueOutOfRange`
- `SnowReceiverInvalid`
- `SnowMaskMissingUv`
- `SnowDynamicCapabilityMissing`
- `SnowPageBudgetExceeded`
- `SnowStampOverflow`
- `SnowFieldGenerationMismatch`
- `SnowMaterialInterfaceMismatch`
- `SnowFallbackMissing`
- `SnowRenderBudgetExceeded`

## 15. Qualificationと完了条件

### C1

- 2D top-downと3D compact arenaでLow／Medium／Highの降雪densityが選べる。
- static snow maskがPBR、Toon、Pixel Dioramaの各Styleで意図したMaterial roleを維持する。
- authoritative foot contactからfootprint Decal／VFXが生成され、Particle collisionからは生成されない。
- Mobile 10分thermal fixtureとParticle overdraw gateを通る。

### C2

- accumulation、melt、compaction、footprintが固定入力から同じfield hashを生成する。
- page admission、overflow、stamp dropが規定順で再現可能である。
- device loss後にstale pageを参照せず再構築できる。
- D3D12、Vulkan、Metalのgolden image、memory、GPU P95を満たす。
- Gameplay friction fixtureがGPU field、render frame rate、Particle countを変更しても同じ結果になる。

### C3着手Gate

変形geometry、avalanche、large-world persistenceは、C2 Production、Terrain／Streaming設計、Physics連携、Save上限、Mobile fallbackが承認されるまでSource Schemaへ追加しない。
