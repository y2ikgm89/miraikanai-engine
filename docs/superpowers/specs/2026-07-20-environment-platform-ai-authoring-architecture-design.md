# Miraikanai Engine Environment Platform／AI Authoringアーキテクチャ規約

- 日付: 2026-07-20
- 状態: 設計レビュー用正式仕様。既存Environment計画を型付きSource、AI Intent、Operation、Preview、Runtime Compiler、Qualificationへ具体化
- 対象: Sky、Atmosphere、Fog、Cloud、Environment Lighting、AI／Editor Authoring
- Lighting規約: [Miraikanai Engine Lighting／AI Authoringアーキテクチャ規約](./2026-07-20-lighting-ai-authoring-architecture-design.md)

## 1. 結論

Miraikanai EngineのEnvironmentは、Sky、Atmosphere、Fog、Cloud、Environment Lightingを別々の描画設定として公開せず、version付き`EnvironmentProfileV1`からTarget別Derived Artifactを生成する独立Platformとして実装する。

人間またはAIは「薄い朝靄」「湿った洞窟」「火星に似た夕焼け」のような意味Intent、登録済みPreset、範囲検証可能な公開Parameterを編集できる。AIへLUT解像度、ray-march step、froxel grid、history weight、Render Graph resource、shader sourceを選ばせない。`EnvironmentIntentResolver`が意味Intentを決定論的なPresetと型付きOverrideへ変換し、`AuthoringCommandGateway`がSchema、semantic、Capability、Target、budget、lock、base revisionを検証してからだけCommitする。

本Platformの完成条件は、見た目が描画できることだけではない。自然言語とEditor操作が同じtyped Operationへ収束し、Before／After Preview、予測cost、fallback、closed Diagnostic、Undo、Cook、device recovery、Reference Scene、AI Evalを通過して初めて当該CapabilityをProduction表示する。

## 2. 決定権と対象外

| 主題 | 正本 |
|---|---|
| Environment Source型、Preset、Intent、AI Operation、Validator、Preview、Compiler、Artifact、Budget、Diagnostic、Qualification | 本書 |
| C1／C2／C3の製品到達点と全体実装順序 | 2D／3D機能計画 |
| RenderSnapshot、Render Graph、GPU resource、pass、barrier、Backend、device loss | Rendering／Render Graph規約 |
| MCD Type／Operation／Capability、Provider Schema projection | 実行可能契約規約 |
| Math storage／semantic type、unit／space／normalization、Transform／Velocity、failure | [Math／Core Utilities規約](./2026-07-20-ai-readable-math-core-utilities-architecture-design.md) |
| ChangeSet、Commit、Undo、journal、recovery | Authoring Model／Project State規約 |
| Weather Snapshot、降水、温度、風、Snow Surface | Weather／Snow Surface規約 |
| Water Surface、Water Volume、水中固有吸収／散乱 | Water Surface Platform規約 |
| Visual StyleのPalette、style-critical field、Profile lock | Material／Visual Style／AI Authoring規約 |
| Sun／Moonの`LightSourceV1`、物理単位、Light Intent／Resolver／Runtime Snapshot | Lighting／AI Authoring規約 |
| Target、thermal、mobile fallback | モバイルPlatform規約 |

本書は次を所有しない。

- Gameplay visibility、AI perception、stealth判定。必要な場合は描画結果と独立したtyped Gameplay ruleを使う。
- 流体力学による雲／煙／気象前線simulation。
- VFXの局所煙、炎、爆発。長時間の空間FogはEnvironment、短寿命EffectはVFXが所有する。
- Water内部の水中Fog。境界選択はWater Volumeが所有し、Environmentは入射Sky／Light Snapshotだけを提供する。
- 複数大気天体、large-world cloud streaming、hardware ray traced environment。C3 Research Gate後だけ扱う。

## 3. 設計原則

1. **Intent first**: 初心者とAIは意味Intentを選び、Engineが物理値へ解決する。
2. **Typed escape hatch**: 上級者は公開物理Parameterを明示編集できるが、単位、範囲、Capability、Riskを回避できない。
3. **Derived internals**: GPU構成、履歴、sample数、format、fallback順はEngineがTarget Profileから導出する。
4. **One source**: Editor、AI、CLI、MCPは同じSource型とOperationを使い、簡易AI専用形式を持たない。
5. **Preview before impact**: High Impact変更はBefore／After、Cost、Fallbackを確認してからCommitする。
6. **No visual authority leak**: 描画Fog、Cloud shadow、GPU densityをGameplayの権威入力へ使わない。
7. **Bounded generation**: collection、距離、密度、resource、job、更新頻度へHard上限を持つ。
8. **Stable degradation**: 品質低下でArt Direction、Gameplay visibility、human lockを変更しない。

## 4. Platform構成とデータフロー

```text
Natural Language／Inspector／Command Palette
  -> EnvironmentIntentV1またはtyped Environment Operation
  -> EnvironmentIntentResolver
  -> EnvironmentChangeSet
  -> Schema／Semantic／Capability／Target／Budget／Lock Validator
  -> EnvironmentPreviewReceiptV1
  -> AuthoringCommandGateway
  -> EnvironmentProfileV1 revision
  -> EnvironmentCompiler
  -> Target別EnvironmentArtifactSetV1
  -> R10 atomic promotion
  -> EnvironmentPresentationSnapshotV1
  -> Render Graph Environment passes
```

`EnvironmentIntentResolver`はProject状態を変更しないpure queryである。Resolver出力は候補Preset、型付きOverride、仮定、質問、予測Capabilityを持つが、Commit権限を持たない。Write Operationは`ProjectChangeSet`を生成するだけで、live World、GPU resource、ProjectRevisionを直接変更しない。

## 5. Source Model

### 5.1 `EnvironmentProfileV1`

```text
EnvironmentProfileV1
  profile_id: StableId
  schema_version: 1
  source_revision: uint64
  display_name: string[1..128]
  semantic_intent: EnvironmentIntentV1
  preset_id: EnvironmentPresetId
  quality_policy: EnvironmentQualityPolicyV1
  sky: SkySourceV1
  atmosphere: AtmosphereSourceV1
  fog: FogSourceV1
  local_fog_volumes: LocalFogVolumeSourceV1[0..128]
  cloud: CloudSourceV1
  lighting: EnvironmentLightingSourceV1
  weather_binding: nullable<WeatherBindingV1>
  style_binding: nullable<VisualStyleBindingV1>
  field_locks: EnvironmentFieldLockV1[0..256]
```

`source_revision`はDocument revisionと一致させる。Runtime generation、LUT handle、GPU descriptor、history stateをSourceへ保存しない。未知Field、未知enum、NaN／Inf、単位なし物理量、Target上限を超えるcollectionを拒否する。

### 5.2 `SkySourceV1`

| Field | 型／範囲 |
|---|---|
| `mode` | `solid \| gradient \| hdri \| physical_atmosphere` |
| `solid_color` | nullable linear RGB、各0～64 |
| `gradient_zenith／horizon／ground` | nullable linear RGB、各0～64 |
| `hdri_asset` | nullable HDR cubemap Asset reference |
| `rotation_radians` | `[0, 2π)` |
| `intensity_multiplier` | 0～64 |
| `sun_light_id`／`moon_light_id` | nullable Light StableId |

`physical_atmosphere`はC2 Capabilityを必要とする。C1で指定された場合、Projectが明示許可した`precomputed_sky_cubemap` fallbackが存在する時だけCookできる。Light参照はLighting規約の`LightSourceV1`かつDirectional Lightに限り、同一Lightをsunとmoonへ二重指定しない。Environmentは参照先Lightの物理量、色、Shadow、Runtime Snapshotを複製しない。

`solid`では`solid_color`だけ、`gradient`では3色だけ、`hdri`では`hdri_asset`だけを必須とする。`physical_atmosphere`では色とHDRIをnullにし、`AtmosphereSourceV1.mode=physical`を必須とする。modeに属さないpayloadを保持して暗黙復帰に使わず、Profile切替はPresetまたはChangeSet履歴から行う。

### 5.3 `AtmosphereSourceV1`

```text
AtmosphereSourceV1
  mode: disabled | physical
  preset_id: reference_earth_v1 | custom_planet_v1
  custom_planet: nullable<CustomAtmosphereBodyV1>
```

`reference_earth_v1`では`custom_planet=null`、`custom_planet_v1`では全Fieldを持つ`CustomAtmosphereBodyV1`を必須とする。

`ReferenceEarthV1`は次をsource dataとして保存する。距離はmeter、係数は`m^-1`であり、shader内へkilometer値として保存し直さない。

| 項目 | `ReferenceEarthV1` |
|---|---|
| ground／top radius | 6,360,000 m／6,460,000 m |
| ground albedo | linear RGB `(0.3, 0.3, 0.3)` |
| Rayleigh scattering／scale height | `(5.802, 13.558, 33.100) × 10^-6 m^-1`／8,000 m |
| Rayleigh absorption | `(0, 0, 0) m^-1` |
| Mie scattering／absorption／scale height／g | `3.996 × 10^-6 m^-1`／`4.400 × 10^-6 m^-1`／1,200 m／0.8 |
| Ozone absorption | `(0.650, 1.881, 0.085) × 10^-6 m^-1` |
| Ozone density | 25,000 mで1、上下15,000 mで0となるtriangle profile |
| Sun angular radius | 0.267° |

`preset_id`は`reference_earth_v1 \| custom_planet_v1`のclosed enumとする。`custom_planet_v1`はground radius、top radius、全scattering／absorption、density profile、phase g、ground albedoを必須とし、Earth値からの部分補完を禁止する。

`top_radius > ground_radius > 0`、全係数finiteかつ0以上、`-0.999 <= g <= 0.999`を必須とする。登録済みPresetの選択はR2、custom coefficientの直接編集はR3で、Before／After Previewと明示承認を必要とする。

### 5.4 `FogSourceV1`

```text
FogSourceV1
  mode: disabled | height_distance | volumetric
  density_input: direct_extinction | target_visibility
  target_visibility_m: nullable<float32>
  base_extinction_m_inv: nullable<float32>
  height_falloff_m_inv: float32
  base_height_m: float32
  albedo_source: fixed_linear | inherit_sky | palette_token
  scattering_albedo_linear: nullable<LinearRgb>
  palette_token: nullable<StablePaletteToken>
  anisotropy_g: float32
  max_distance_m: float32
  ambient_injection: float32
  local_light_injection: float32
```

範囲は次に固定する。

| Field | 範囲 |
|---|---|
| `target_visibility_m` | 5～1,000,000 m |
| `base_extinction_m_inv` | 0～10 `m^-1` |
| `height_falloff_m_inv` | 0～10 `m^-1` |
| `base_height_m` | ±1,000,000 m |
| `scattering_albedo_linear` | 各0～1 |
| `anisotropy_g` | -0.99～0.99 |
| `max_distance_m` | 1～1,000,000 m |
| `ambient_injection`／`local_light_injection` | 0～4 |

Height Fogは次で定義する。

```text
e = clamp(-height_falloff_m_inv * (world_y_m - base_height_m), -80, 80)
extinction(world_y_m) = base_extinction_m_inv * exp(e)
optical_depth = clamp(extinction * distance_m, 0, 80)
transmittance = exp(-optical_depth)
```

`target_visibility_m`は均一Fogにおいて背景contrastが5%へ低下する距離と定義し、Resolverは`base_extinction_m_inv = 2.995732 / target_visibility_m`を候補値にする。Height分布がある場合はbase height上の水平rayにこの定義を適用し、Preview Receiptへ測定条件を記録する。

`density_input=direct_extinction`では`base_extinction_m_inv`だけ、`target_visibility`では`target_visibility_m`だけを必須とし、他方をnullにする。両方または両方nullを`EnvironmentFogVisibilityConflict`で拒否する。`albedo_source=fixed_linear`ではlinear RGBだけ、`palette_token`では登録済みPalette tokenだけ、`inherit_sky`では両方nullを必須とする。

既定は`density_input=direct_extinction`、`base_extinction_m_inv=0.01`、`height_falloff_m_inv=0.10`、`base_height_m=0`、`albedo_source=fixed_linear`、albedo `(0.9,0.9,0.9)`、anisotropy `0.2`、max distance 10,000 mとする。

### 5.5 `LocalFogVolumeSourceV1`

```text
LocalFogVolumeSourceV1
  volume_id: StableId
  shape: box | sphere | ellipsoid | cylinder
  transform: Transform3f
  blend_distance_m: float32
  density_operation: add | subtract | replace
  density_multiplier: float32
  albedo_linear: LinearRgb
  anisotropy_g: float32
  emission_linear: LinearRgb
  noise_asset: nullable<Texture3DRef>
  noise_scale_m: float32
  noise_velocity_m_s: Velocity3f
  priority: int32
  semantic_role: ground_mist | room_haze | dust_shafts | cave_fog | exclusion | custom
```

Volume寸法は各axis 0.01～100,000 m、blend distanceは0～Volume最短半径、density multiplierは`add／replace`で0～10、`subtract`で-10～0とする。C2 Mediumはvisible 32、Highは128を上限とし、Source全体上限も128とする。上限超過を黙って間引かず、Preview時点で`EnvironmentLocalFogLimitExceeded`を返す。

`noise_velocity_m_s`はVolume local space、Cloudの`wind_velocity_m_s`はWorld spaceの`Velocity3f`とし、MCD descriptorへframeを必須にする。両者の暗黙変換と裸のvector公開を禁止する。

### 5.6 `CloudSourceV1`

| Field | 型／範囲 |
|---|---|
| `mode` | `disabled \| layer_2d \| volumetric` |
| `layer_texture_asset` | nullable 2D cloud Texture source |
| `bottom_height_m` | 0～100,000 |
| `top_height_m` | bottom＋1～200,000 |
| `max_view_distance_m` | 1～100,000 |
| `coverage` | 0～1 |
| `density_multiplier` | 0～10 |
| `weather_map_asset` | nullable 512² `RGBA8_UNORM` Derived Asset source |
| `base_noise_asset` | nullable 128³ `R8_UNORM` Derived Asset source |
| `detail_noise_asset` | nullable 32³ `R8_UNORM` Derived Asset source |
| `wind_velocity_m_s` | World spaceの`Velocity3f`、各axis ±200 m/s |
| `cast_cloud_shadow` | bool |

既定bottom／topは1,500 m／5,000 m、最大view distanceは100,000 m、coverage 0.50、density multiplier 1.0とする。`layer_2d`はProject AssetまたはEngine同梱Reference texture、`volumetric`はweather／base／detail noiseの3 Assetを必要とする。nullの場合は同じversionの`ReferenceCloudAssetsV1`を一式使用し、一部だけを暗黙補完しない。C1は`layer_2d`まで、C2は`volumetric`を許可する。AIはnoise textureのtexel、ray-march step、blue-noise phase、shadow map resolutionを生成または指定しない。

### 5.7 `EnvironmentLightingSourceV1`

```text
EnvironmentLightingSourceV1
  ibl_mode: baked | dynamic_incremental
  diffuse_irradiance_enabled: bool
  specular_prefilter_enabled: bool
  exposure_mode: automatic | manual
  exposure_compensation_ev: float32
  manual_ev100: nullable<float32>
  min_ev100: float32
  max_ev100: float32
```

compensationは-8～8 EV、manual EV100は-16～32、automatic出力範囲は`min_ev100 < max_ev100`かつ各-16～32とする。`manual`では`manual_ev100`を必須、`automatic`ではnullを必須とする。Automatic ExposureはEV100 -16～32を256 log-luminance binへ量子化し、0.5～99.5 percentile、18% middle gray、brighten 1.5 EV/s、darken 3.0 EV/sを`ExposureRuntimeProfile::ReferenceV1`へ固定する。`dynamic_incremental`はC2 `capability.environment.dynamic_ibl_v1`を必要とする。

C1 IBLはdiffuse 32²×6 `RGBA16_FLOAT`、specular 256²×6・9 mip `RGBA16_FLOAT`、BRDF LUT 256² `RG16_FLOAT`をDerived Artifactとして生成する。HDR sourceを除くpersistent IBL setを8 MiB以下にする。AIはcubemap convolution結果、histogram bin、percentile、adaptation rateを直接編集しない。

### 5.8 Policy、Binding、Lock

```text
EnvironmentQualityPolicyV1
  minimum_tier: low | medium | high
  preferred_tier: low | medium | high
  fallback_policy: deny | require_approval | allow_preapproved
  budget_profile_id: StableId

WeatherBindingV1
  weather_profile_id: StableId
  minimum_revision: uint64
  channels: set<wind | cloud_coverage | cloud_density | precipitation | temperature>

VisualStyleBindingV1
  visual_style_profile_id: StableId
  minimum_revision: uint64

EnvironmentFieldLockV1
  field_id: uint32
  locked_at_project_revision: uint64
  owner_kind: human | project_policy
  reason: string[1..256]
```

Tier順を`low < medium < high`とし、`minimum_tier <= preferred_tier`を必須とする。`allow_preapproved`はTarget Profileに記録済みの具体的fallback集合だけに適用し、AIの一般的な「おまかせ」を恒久許可へ変換しない。Bindingのminimum revisionを満たさないSnapshotはstaleとする。LockはMCDのstable Field IDを参照し、JSON pointerや表示名を正本にしない。同じField IDのlock重複を拒否する。

## 6. 自然言語IntentとPreset

### 6.1 `EnvironmentIntentV1`

```text
EnvironmentIntentV1
  setting: outdoor | indoor | cave | underwater_boundary | space_transition | stylized
  time_of_day: inherit | dawn | day | dusk | night
  visibility: clear | light_haze | light_mist | fog | dense_fog
  vertical_distribution: uniform | ground_hugging | elevated_layer | local_only
  light_shafts: none | subtle | pronounced
  color_intent: inherit_sky | neutral | warm | cool | palette_token
  cloud_intent: none | thin | scattered | overcast | storm_like
  motion_intent: still | gentle | windy | inherit_weather
  target_visibility_m: nullable<float32>
  palette_token: nullable<StablePaletteToken>
```

`underwater_boundary`はWater Volumeの生成要求ではない。Cameraが水中へ入る可能性を示すadvisory intentであり、水中ProfileはWater規約のOperationで別に作成する。

### 6.2 Reference Preset

| Preset | Intent | Fog source | Capability |
|---|---|---|---|
| `clear_day_v1` | clear、uniform、shaftなし | visibility 20,000 m、falloff 0 | C1 |
| `temperate_morning_mist_v1` | light_mist、ground_hugging、subtle | visibility 800 m、falloff 0.12、base 1 m | C1 |
| `humid_distance_haze_v1` | light_haze、uniform、subtle | visibility 2,000 m、falloff 0.015 | C1 |
| `dense_ground_fog_v1` | dense_fog、ground_hugging、subtle | visibility 50 m、falloff 0.25、base 1 m | C1 |
| `interior_dust_shafts_v1` | indoor、local_only、pronounced | global density 0、`dust_shafts` Local Volume | C2 |
| `overcast_volumetric_v1` | fog、overcast、subtle | volumetric Fog＋coverage 0.9 | C2 |
| `stylized_tinted_fog_v1` | stylized、palette_token | Profile値＋必須Palette token | C1／C2 |
| `reference_earth_atmosphere_v1` | outdoor、inherit time | `ReferenceEarthV1` atmosphere | C2 |

PresetはSourceの複製ではなく、version付き定数とResolver ruleである。Preset適用時は既存human lockを維持し、lock済みfieldを上書きする必要がある候補を生成しない。

`interior_dust_shafts_v1`は選択中のroom／region boundsをbox shapeへ使い、density multiplier 0.015、albedo `(0.78,0.72,0.62)`、anisotropy 0.7、blend distanceを最短半径の10%とする。bounded regionを一意に取得できない場合はshapeを推測せずBlocking質問を返す。Built-in Cloud PresetはEngine同梱の`ReferenceCloudAssetsV1`を参照し、Project Assetが明示指定された場合だけ置き換える。AIはnoise texelを生成しない。

Intentの既定解決規則は次に固定する。

| Intent | 解決規則 |
|---|---|
| `clear／light_haze／light_mist／fog／dense_fog` | target visibility 20,000／2,000／800／200／50 m |
| `uniform` | height falloff 0 |
| `ground_hugging` | Presetに値がなければheight falloff 0.10 `m^-1` |
| `elevated_layer` | Global Height Fogへ近似せずLocal Fog Volume候補 |
| `local_only` | Global density 0、Local Fog Volume必須 |
| shaft `none／subtle／pronounced` | anisotropy 0.2／0.5／0.7。`pronounced`はC2 Volumetric Fog必須 |
| color `neutral／warm／cool` | fixed linear RGB `(0.9,0.9,0.9)`／`(0.95,0.88,0.75)`／`(0.78,0.86,0.95)` |
| `inherit_sky` | `albedo_source=inherit_sky` |
| `palette_token` | 登録済みPalette token必須 |
| cloud `none／thin／scattered／overcast／storm_like` | coverage 0／0.2／0.5／0.9／1.0 |

`storm_like`は見た目のCloud intentであり、雨、雷、風、Gameplay weatherを暗黙生成しない。`gentle／windy／inherit_weather`はversion付きWeather bindingがある場合だけ風向と速度を解決する。bindingがなく`still`以外を要求された場合は一つのBlocking質問を返す。

`time_of_day`は既存のversion付きTime-of-Day／sun animation sourceへ渡すsemantic phaseであり、時刻、緯度、経度を暗黙生成しない。既存sourceがなく`inherit`以外を要求された場合、登録済みScene Presetを提示するか一つのBlocking質問を返す。

### 6.3 Intent Resolver

Resolver入力は、Intent、Target Profile、Camera far、Scene bounds、sun／moon、Visual Style、Weather binding、現在Profile、field lock、Contract set hashである。出力は最大3候補とし、各候補が次を持つ。

```text
EnvironmentIntentCandidateV1
  candidate_id
  preset_id
  typed_overrides
  required_capabilities
  estimated_cost
  assumptions
  blocking_questions
  confidence: low | medium | high
  contract_set_hash
```

Resolverは次の順で選ぶ。

1. human lock、Target、minimum Quality、Capabilityで不可能な候補を除外する。
2. 明示された距離、色、時間帯、屋内外を満たすPresetを選ぶ。
3. Visual StyleとWeather bindingを適用する。
4. Budget内で最小Capabilityの候補を優先する。
5. 同点はPreset IDのbyte順で安定化する。

候補が一つでconfidence high、変更がR2以下、必要Previewが成功し、ユーザーの現在の委任範囲内なら追加質問なしに提案できる。候補が0、相互に異なるHigh Impact候補が複数、Target未設定、Gameplay visibilityとの混同、human lock競合のいずれかでは質問を一つだけ返す。

初心者へRayleigh、Mie、phase g、froxel、LUT、sample count、texture format、history weightを質問しない。

## 7. AI CapabilityとTyped Operation

### 7.1 Capability

```text
capability.environment.core_v1
capability.environment.sky_hdri_v1
capability.environment.height_fog_v1
capability.environment.ibl_baked_v1
capability.environment.atmosphere_lut_v1
capability.environment.aerial_perspective_v1
capability.environment.volumetric_fog_v1
capability.environment.local_fog_volume_v1
capability.environment.volumetric_cloud_v1
capability.environment.dynamic_ibl_v1
capability.environment.cloud_shadow_v1
capability.environment.intent_resolver_v1
```

`capabilities.search`はtitle、tag、Target、maturity、summaryだけを返し、`capabilities.read`が選択CapabilityのSource型、Operation、Constraint、Budget、Preset、valid／invalid例を返す。未昇格CapabilityをManifestへ載せず、AIに選択させない。

### 7.2 Authoring Operation

| Operation ID | 動作 | Risk |
|---|---|---|
| `operation.environment.inspect_profile` | Source、Artifact、lock、Diagnosticを読む | R0 |
| `operation.environment.list_presets` | 利用可能PresetとCapabilityを読む | R0 |
| `operation.environment.resolve_intent` | Intentから候補を生成する | R0 |
| `operation.environment.validate_changeset` | Schema、semantic、Capability、budget、lockを検査 | R0 |
| `operation.environment.preview_changeset` | Before／After、debug view、costを生成 | R0 |
| `operation.environment.estimate_cost` | Target別CPU／GPU／memory／rebuild costを返す | R0 |
| `operation.environment.create_profile` | Profile Documentを作成 | R2 |
| `operation.environment.apply_preset` | 登録済みPresetを適用 | R2 |
| `operation.environment.set_intent` | `EnvironmentIntentV1`を更新 | R2 |
| `operation.environment.set_sky` | Solid／Gradient／HDRI／Physicalを設定 | R2 |
| `operation.environment.set_sun_moon_link` | Directional Light参照を設定 | R2 |
| `operation.environment.set_height_distance_fog` | C1 Fogを一つの不変条件付きOperationで更新 | R2 |
| `operation.environment.set_volumetric_fog` | C2 Fog公開Parameterを更新 | R2 |
| `operation.environment.create_local_fog_volume` | Local Volumeを作成 | R2 |
| `operation.environment.update_local_fog_volume` | shape、density、material意味を更新 | R2 |
| `operation.environment.delete_local_fog_volume` | Local Volumeを削除 | R2 |
| `operation.environment.set_atmosphere_preset` | 登録済み大気Presetを設定 | R2 |
| `operation.environment.set_custom_atmosphere` | 全係数を明示した任意planetを提案 | R3＋明示承認 |
| `operation.environment.set_cloud_layer` | 2D／Volumetric Cloud Sourceを更新 | R2 |
| `operation.environment.set_lighting` | IBL／Exposure sourceを更新 | R2 |
| `operation.environment.bind_weather` | version付きWeather sourceを関連付ける | R2 |
| `operation.environment.generate_fallback` | Target別fallback候補を作成 | R1。Commitは別R2 |
| `operation.environment.bake` | IBL／Cloud Derived Artifact jobを要求 | R2 |
| `operation.environment.run_qualification` | read-only fixture reportを生成 | R1 |

自由形式の`set_environment_json_pointer`、raw shader、console variable、Render Graph pass編集をProviderへ公開しない。複数fieldに不変条件があるFog、Atmosphere、Cloudを細かな`SetComponentField`列へ分解しない。

### 7.3 Field Authority

| Field群 | AI | 人間／Editor | Engine |
|---|---|---|---|
| Intent、Preset、公開Parameter | ChangeSetとして提案可 | 編集可 | 検証 |
| human lock済みField | 読取のみ | lock解除後に編集 | 強制維持 |
| custom atmosphere係数 | R3提案のみ | 明示承認 | 完全性検証 |
| minimum Quality Tier | High Impact提案 | 承認 | Target検証 |
| LUT／froxel／step／format | 編集不可 | 編集不可 | 導出 |
| history weight／破棄条件 | 編集不可 | 編集不可 | 導出 |
| Runtime generation／GPU handle | 非公開 | debug読取のみ | 所有 |
| fallback選択 | 候補提案のみ | Policyを承認 | Capabilityとbudgetから選択 |

lock Operationはtrusted Editorだけが発行し、AI Providerへ公開しない。AIがlock競合を回避するため別Fieldへ意図を歪めることを禁止する。

### 7.4 Level 0自然言語

AIは既存SceneとProject Profileから次を解決する。

| 不足事項 | 扱い |
|---|---|
| Target Profile | 未設定ならBlocking |
| 屋内／屋外 | Scene boundsとSky設定から一意なら自動、曖昧ならHigh Impact |
| 視程が見た目だけかGameplayにも影響するか | 曖昧ならBlocking |
| 時間帯 | 既存sun linkがあれば継承、なければHigh Impact |
| Realistic／Toon／Pixel等 | Visual Style Profileから取得、未設定ならHigh Impact |
| Volumetric品質 | AIが質問せずTargetとQuality Policyから導出 |

「不気味に」「映画的に」だけでは密度や色を確定しない。既存Visual Style、Scene semantic、参考画像から一意に解決できなければ、最大3候補のPreviewを提示する。

### 7.5 ChangeSet例

```text
intent:
  setting: outdoor
  time_of_day: dawn
  visibility: light_mist
  vertical_distribution: ground_hugging
  light_shafts: subtle
  color_intent: cool
  cloud_intent: scattered
  motion_intent: inherit_weather
  target_visibility_m: 600

resolved:
  preset_id: temperate_morning_mist_v1
  operations:
    - operation.environment.set_intent
    - operation.environment.apply_preset
    - operation.environment.set_height_distance_fog
  required_capabilities:
    - capability.environment.core_v1
    - capability.environment.height_fog_v1
```

AIの説明文ではなく、Canonical ChangeSet hash、Preview Receipt hash、base revision、approval nonceを承認対象にする。

## 8. ValidationとRisk

Validatorは次の順で実行する。

1. MCD schema、closed enum、collection上限、finite、単位。
2. StableId、Asset revision、Light／Weather／Style参照。
3. Atmosphere半径、係数、Fog range、Cloud layer、Local Volume shapeのDomain invariant。
4. human lock、Decision lock、base revision。
5. Capability dependency、Target intersection、Quality fallback。
6. GPU memory、Environment pass、persistent／transient peak、Artifact job cost。
7. Scene bounds、camera near／far、`fog_far > near`。
8. Weather、Water、VFX、Visual Styleとのcross-profile invariant。
9. Preview Receipt freshnessと必要承認。

次をHigh ImpactとしてBefore／After Previewと明示承認を必須にする。

- Project minimum Quality Tierの変更。
- `custom_planet_v1`の作成または係数変更。
- Scene全体のtime-of-day、sun／moon link、Sky modeの変更。
- `target_visibility_m < 100`、または現在値から4倍を超える視程変更。
- Volumetric Cloud／Fogを新たに必須Capabilityへする変更。
- Gameplay visibilityにも影響すると解釈される要求。

Gameplay visibilityを求められた場合、Environment ChangeSetだけで完了扱いにせず、別のauthoritative Gameplay ruleを設計して両方を一つのimpact summaryへ載せる。

## 9. PreviewとEditor UX

`EnvironmentPreviewReceiptV1`は次を持つ。

```text
EnvironmentPreviewReceiptV1
  receipt_id
  contract_set_hash
  base_project_revision
  canonical_changeset_hash
  target_profile_id
  quality_tier
  camera_set_hash
  before_capture_refs[1..3]
  after_capture_refs[1..3]
  debug_view_refs
  visibility_measurements
  estimated_gpu_p95_ms
  estimated_cpu_p95_ms
  estimated_peak_memory_bytes
  artifact_rebuild_units
  fallback_summary
  diagnostics
  expires_at
```

Preview cameraは最低でもground view、elevated view、代表Gameplay cameraを使用する。該当しないcameraは省略できるが、1件未満にしない。Debug Viewはtransmittance、extinction、froxel slice、Local Volume bounds、Cloud coverage、history validity、IBL generationを表示する。

Editorは次を同じtyped Operationへ変換する。

- Beginner: Intent、Preset、視程、時間帯、霧の高さ、雲量。
- Advanced: Atmosphere coefficient、Fog physical parameter、Local Volume、Cloud source、IBL。
- AI Partner: 候補、理由、仮定、Before／After、Cost、Fallback、Diagnostic。
- Profiler: Environment pass、memory、history reset、LUT／IBL rebuild。

AIとEditorでPreview Renderer、Preset、Validatorを分けない。PreviewとShippingが異なるCapabilityの場合はPreviewへ明示watermarkと差分理由を表示する。

## 10. Compiler、Artifact、Runtime

### 10.1 Compiler

`EnvironmentCompiler`はSourceとTarget Profileから次を生成する。

```text
EnvironmentArtifactSetV1
  artifact_set_id
  source_profile_id
  source_revision
  contract_set_hash
  target_profile_id
  quality_tier
  sky_artifact
  atmosphere_lut_artifact
  fog_profile_artifact
  cloud_artifact
  ibl_artifact
  fallback_manifest
  resource_receipt
```

同じcanonical Source、Contract set、Target Profile、Toolchainから同じArtifact hashを生成する。Derived binaryをSource control上の正本にせず、source hashから再生成可能にする。

### 10.2 Atmosphere LUT

| LUT | 解像度／format | ray-march step |
|---|---|---:|
| Transmittance | 256×64 `RGBA16_FLOAT` | 40 |
| Multiple scattering | 32×32 `RGBA16_FLOAT` | 20 |
| Sky view | 200×100 `RGBA16_FLOAT` | 30 |
| Aerial perspective | 32×32×32 `RGBA16_FLOAT` | 30 |

step数は`AtmosphereLutProfile::ReferenceV1`のexact値で、Quality Profile以外から変更できない。旧generationは新しい4枚がreadyになるまでliveとし、R10でatomic promotionする。

各rayはjittered stratified midpointを使い、光学的厚さの上限またはplanet boundary到達で終了する。Transmittanceはatmosphere generation変更、Multiple scatteringはtransmittance／light generation変更、Sky viewはobserver altitude／sun direction／atmosphere generation変更、Aerial perspectiveはcamera frustum／sun／atmosphere generation変更で再生成する。

### 10.3 Volumetric Fog

| 項目 | Medium | High |
|---|---:|---:|
| 最大grid | 160×90×64 | 240×135×96 |
| XY寸法 | `min(ceil(internal/12), 160×90)` | `min(ceil(internal/8), 240×135)` |
| Z分布 | near～fog far、隣接slice幅比1.2 | 同左 |
| Local Fog Volume | 32 | 128 |
| 注入local light | 64 | 128 |
| current／history scattering | 各`RGBA16_FLOAT` | 各`RGBA16_FLOAT` |
| extinction | `R16_FLOAT` | `R16_FLOAT` |

`fog_far = min(max_distance_m, camera_far)`とし、`fog_far <= near`をCompile errorにする。

N sliceの最初の幅を`Δz0 = (fog_far-near) * (1.2-1) / (1.2^N-1)`、以後`Δzi = Δz0 * 1.2^i`とする。AIとProject C++はslice数、分布比、resource formatを変更できない。

### 10.4 Volumetric Cloud

Mediumは1/4-linear、最大480×270、primary 48 step、light 6 stepとする。Highは1/2-linear、最大960×540、primary 96 step、light 8 stepとする。view segment `[entry, exit]`のstep長を`(exit-entry)/N`、sample位置を`entry + (i + blue_noise_0_1) * step_length`とし、sun方向のlight sampleも同じuniform ruleを使う。blue-noise phaseを256 frameで循環し、opacity 0.995またはscene depthでearly-outする。

Cloud shadowはMedium 1,024² `R16_FLOAT`を4 frameごと、High 2,048²を2 frameごとに更新する。sun directionが1°以上変化、またはweather generation変更時は即時更新する。

### 10.5 Dynamic IBL

Dynamic IBLは6 face capture、1 diffuse convolution、9 specular mipのexactly 16 work unitとする。staging setへ最大1 unit／frameを実行し、全unit ready時だけR10でgenerationを切り替える。1 frame 0.25 ms soft capを超えた作業は次frameへ送るが、部分generationを公開しない。

### 10.6 History

CloudとVolumetric Fogの履歴は次で全破棄する。

- 明示camera cut。
- world-origin rebase。
- internal width／height変更。
- non-jitter projection変更。
- Environment generation変更。
- sun directionが1 frameで5°以上変化。
- weather coverage／density fieldの最大絶対差が0.20以上。

破棄frameはhistory weight 0、次の7 frameで`min(0.90, valid_frames / 8)`まで増加させる。AI、人間、Project C++はweightや破棄条件を編集できない。

## 11. Subsystem境界

### 11.1 Weather

Weatherは風、coverage、density、降水、温度のversion付きSnapshotをpublishする。EnvironmentはCloudとFogのPresentation入力として読むが、Weather Snapshotを書き戻さない。stale generationは前のvalid Snapshotを保持し、Diagnosticを出す。

### 11.2 Water

EnvironmentはSky radiance、sun／moon、Fog color、exposureをWater rendererへ提供する。Water Volumeの選択、水中absorption、scattering、fog、waterlineはWater規約が所有する。水中からEnvironment Fogを二重適用しない。

### 11.3 VFX

短寿命の煙、炎、爆発、魔法はVFXが所有する。VFXから永続Local Fogへ変換する暗黙経路を持たない。変換が必要なAuthoring操作は新しいLocal Fog Volume Sourceを作るR2 ChangeSetとしてPreviewする。

### 11.4 Visual Style

Visual Styleは許可Palette token、fog saturation範囲、contrast、Pixel Diorama composite policy、style-critical lockを提供する。Environment ResolverはStyle Profileを変更せず、その制約内の候補だけを返す。Quality fallbackでRealisticからToon等へ画風を変更しない。

### 11.5 Gameplay

Render Fog、Cloud shadow、exposure、GPU transmittanceをPhysics、Navigation、AI perception、stealth、hit判定へ入力しない。Gameplay visibilityが必要なら独立したProfileとQueryを使い、Environmentの`target_visibility_m`はPresentation値であることをSchema descriptionへ明記する。

## 12. Quality、Budget、Fallback

| Tier | Atmosphere | Fog | Cloud |
|---|---|---|---|
| Low | Precomputed sky cubemap | Height／distance fog | 2D cloud layer |
| Medium | Atmosphere LUT | 最大160×90×64 froxel | 1/4-linear、48／6 step |
| High | Atmosphere LUT＋dynamic IBL | 最大240×135×96 froxel | 1/2-linear、96／8 step＋shadow |

Environmentのpersistent、history、当該pass専用transientを合算した同時peak capはLow 32 MiB、Medium 128 MiB、High 256 MiBとする。MediumをC2 reference acceptanceに使い、Atmosphere／Environment pass全体を2.00 ms GPU P95 soft capへ収める。Highも全体14.00 ms GPU frame P95 soft targetを免除しない。

Texture、Render target、transientはRuntime規約の所属memory classへ一度だけ計上する。Render Graph aliasで同じphysical heapを共有するresourceはcommitted physical byteを一度だけ数えるが、aliasを見込んだ未証明の0 byte見積りを禁止する。Preview Receiptはlogical resource内訳とphysical peakの両方を表示する。

Fallback順は次に固定する。

1. Cloud shadow更新頻度。
2. Cloud light step。
3. Cloud primary stepとrender scale。
4. Volumetric Fog XY、次にZ解像度。
5. Local light injection数。
6. Dynamic IBLをbaked IBLへ。
7. Volumetric Cloudを2D layerへ。
8. Volumetric FogをHeight／distance Fogへ。
9. Atmosphere LUTをprecomputed sky cubemapへ。

Fallbackは同じArt Direction、Palette、sun／moon意味、target visibility、Gameplay ruleを維持する。Project minimum Tier未満なら黙って低下せず、不足Capability、予測cost、利用可能fallbackを提示して承認を得る。

## 13. Diagnostic、Failure、Recovery

最低限、次のclosed Diagnostic IDを定義する。

- `EnvironmentPresetNotFound`
- `EnvironmentIntentAmbiguous`
- `EnvironmentIntentUnsupported`
- `EnvironmentInvalidUnit`
- `EnvironmentInvalidRange`
- `EnvironmentFogVisibilityConflict`
- `EnvironmentFogFarBeforeNear`
- `EnvironmentAtmosphereIncomplete`
- `EnvironmentAtmosphereRadiusInvalid`
- `EnvironmentLocalFogLimitExceeded`
- `EnvironmentCloudLayerInvalid`
- `EnvironmentCloudAssetMissing`
- `EnvironmentCapabilityUnavailable`
- `EnvironmentTargetFallbackMissing`
- `EnvironmentBudgetExceeded`
- `EnvironmentDerivedFieldWriteForbidden`
- `EnvironmentHistoryPolicyLocked`
- `EnvironmentHumanLockConflict`
- `EnvironmentPreviewRequired`
- `EnvironmentPreviewStale`
- `EnvironmentWeatherBindingStale`
- `EnvironmentStyleConflict`
- `EnvironmentGenerationMismatch`
- `EnvironmentIblPromotionIncomplete`

Validation、Compile、Bake失敗時はlive Profileとlive Artifact generationを変更しない。新Artifactの一部だけreadyでも公開しない。Device loss後はSourceとArtifact Receiptからpersistent resourceを再生成し、Fog／Cloud historyを破棄する。一回限りの雲イベントやVFXを再発火しない。

## 14. Test、AI Eval、Qualification

### 14.1 Contract

- 全Source型のvalid、boundary、invalid、unknown field、NaN／Inf、unit mismatch fixture。
- 全OperationのSchema projection、C++／TypeScript round-trip、MCP／Provider conformance。
- human lock、stale revision、Capability不足、budget超過、partial custom planetのnegative test。
- 同一Inputから同一Intent候補順、ChangeSet hash、Artifact hashを得るdeterminism test。

### 14.2 Resolver Eval

最低限、次のPromptをholdoutを含むEval setへ持つ。

| Prompt | 期待 |
|---|---|
| 「森の朝。地面に薄い霧、光芒は控えめ」 | morning mist候補、ground_hugging、subtle、R2 |
| 「視界50mの濃霧」 | target visibility 50 m、High Impact Preview |
| 「洞窟の一部だけ埃っぽい光」 | global Fogを増やさずLocal Volume候補、C2 |
| 「火星のような夕焼け」 | stylized tint候補を優先し、custom planet係数を推測しない |
| 「敵から見えなくなる霧」 | PresentationとGameplay visibilityを分離してBlocking質問 |
| 「スマホでUE級の雲を最大品質」 | Target／budget不成立を説明しfallback提示 |
| 「もっと映画的に」 | Style／Sceneから一意でなければ最大3 Preview |

Evalは正しいOperation、質問数、不要な専門用語質問なし、lock尊重、budget、Diagnostic、説明とChangeSet一致を測る。見た目の主観評価だけを合格条件にしない。

### 14.3 Visual／Performance

- Clear day、morning mist、dense fog、interior local fog、sunset atmosphere、overcast cloudのReference Scene。
- Camera cut、resolution change、world-origin rebase、sun急変、weather急変のhistory fixture。
- Transparent、Water、Pixel Diorama、VFX、UI compositeとのintegration capture。
- Reference GPUのgolden image、luminance ordering、transmittance monotonicity、NaN／Infなし。
- MediumでEnvironment 2.00 ms GPU P95、128 MiB peak、10分soak。
- Low／Medium／High fallbackでtarget visibilityとArt Direction invariantを維持。
- Device loss、missing cloud asset、Bake cancel、OOM injectionから最後のvalid generationへ復旧。

### 14.4 Promotion Gate

Capabilityは次をすべて満たした場合だけManifestへ掲載する。

1. MCD Type、Operation、Capability、Diagnosticが生成できる。
2. C++ semantic／budget Validatorがpositive／negative fixtureを通る。
3. EditorとAIが同じChangeSetを生成するconformanceが通る。
4. Preview ReceiptとShipping ArtifactのSource／Contract hashが一致する。
5. Target別Reference Sceneがvisual、performance、memory、recovery Gateを通る。
6. User documentationとAI Tool descriptionが同じschema versionを参照する。

## 15. 段階実装

| Level | 到達点 |
|---|---|
| C0 Contract | Source型、Intent、Preset、Operation、Validator、Resolver fixture、headless Preview Receipt |
| C1 Environment Core | Solid／Gradient／HDRI、sun／moon link、baked IBL、Height／distance Fog、Exposure |
| C2 Environment Advanced | Atmosphere LUT、Aerial Perspective、Volumetric Fog、Local Volume、Volumetric Cloud、Dynamic IBL、Cloud Shadow |
| C3 Research | Multiple atmosphere body、weather front、large-world cloud streaming、optional hardware ray traced environment |

C0はRenderer未完成でもSchema、Resolver、ChangeSet、Diagnosticをheadlessで検証できる。C1／C2を一括実装せず、C0→C1→C2の各GateでAIと手動編集の往復、Preview、Cook、playable resultを成立させる。

## 16. 完了条件

- 自然言語、Inspector、Command Paletteが同じtyped Operationへ収束する。
- AIはCapability discovery後に存在するPreset、Field、Operationだけを使用する。
- Intentから物理値への変換がversion付きResolver ruleとして再現可能である。
- High Impact変更がfresh Preview Receiptと承認なしにCommitされない。
- AIがDerived Field、human lock、Gameplay authorityを変更できない。
- Low／Medium／Highで明示FallbackとBudgetを持つ。
- invalid input、OOM、device loss、missing asset、stale generationで最後のvalid状態を失わない。
- Reference Scene、AI Eval、Contract conformance、performance、memory、recovery testが通る。

## 17. 一次資料

- [Production Ready Atmosphere Rendering, EGSR 2020](https://sebh.github.io/publications/egsr2020.pdf)
- [Towards Unified and Physically-Based Volumetric Lighting in Frostbite, SIGGRAPH 2015](https://advances.realtimerendering.com/s2015/)
- [Real Shading in Unreal Engine 4, SIGGRAPH 2013](https://cdn2.unrealengine.com/Resources/files/2013SiggraphPresentationsNotes-26915738.pdf)

外部資料は実現可能性と既知の構成を確認するために使う。Miraikanai固有のSource型、Intent、Preset、Operation、Risk、Budget、Diagnostic、Promotion Gateは本書を正本とし、既存EngineのProject形式やEditor APIを採用しない。
