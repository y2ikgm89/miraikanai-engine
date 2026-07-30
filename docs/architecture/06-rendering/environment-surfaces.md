# Miraikanai Engine Environment／Surface Contract

- 文書ID: mirakan.arch.rendering-environment-surfaces
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Environment composition、Sky／Atmosphere／Fog／Cloud、Weather presentation、Water body／surface／query、Snow／wetness surface response、単一Source root、domain compiler／artifact、domain budget／fallback／diagnostic／qualification
- 非正本範囲: Terrain／Foliage Source／artifact、GI／reflection／advanced shadow／reference transportのTechnique／fallback、Light／Material／VFX semantics、Render Graph共通pass／resource／history lifetime、LOD共通selection、Physics／Gameplay surface authority、Runtime phase／shared capacity、Asset transaction、AI authorization、Evidence envelope、共通Schema／projection。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[World](world.md)、[Lighting](lighting.md)、[Materials](materials.md)
- 関連文書: [Product Plan](../00-product/product-plan.md)、[Advanced Rendering／Multiplayer Ownership Decision](../decisions/2026-07-29-advanced-rendering-multiplayer-ownership.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Math／Core utilities](../02-foundation/math-core.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Collision](../05-simulation/collision.md)、[Physics](../05-simulation/physics.md)、[Render Graph](render-graph.md)、[Advanced Light Transport](advanced-light-transport.md)、[Terrain／Foliage](terrain-foliage.md)、[Materials](materials.md)、[Lighting](lighting.md)、[VFX authoring](vfx-authoring.md)、[VFX runtime](vfx-runtime.md)、[LOD](lod.md)、[World](world.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-28

## 1. 結論と単一Source model

Environment、Water、Weather／Snowは一つの`EnvironmentSurfaceDocumentV1`をSource rootとする。Sky、Fog、Water Body、Weather input、Snow receiverを独立Project設定へ分散させず、同じrevision、ChangeSet、lock、Target validation、Preview、Cook closureで扱う。旧来のdomain名付き型はroot内のtyped sectionであり、独立した正本や別lifecycleではない。

TerrainとFoliageは[Terrain／Foliage](terrain-foliage.md)の別Source rootであり、本書へ追加しない。本書はweather、water、snow／wetness、surface conditionをtyped inputとして渡し、Terrain／Foliageのtile、species、placement、identity、artifactを所有しない。GI／reflection／advanced shadow／reference transportは[Advanced Light Transport](advanced-light-transport.md)が本書のradiance／volumetric／water／surface summaryを消費してTechniqueとfallbackを解決する。本書はray、screen-space、probe、path等のTechniqueを選択しない。

```text
EnvironmentSurfaceDocumentV1
  document_header
  document_id: StableId
  source_revision: u64
  environment_profiles: EnvironmentProfileV1[1..64]
  world_environment_bindings: EnvironmentWorldBindingV1[0..256]
  weather_profiles: WeatherPresentationProfileV1[0..64]
  water_system: nullable<WaterSystemDocumentV1>
  snow_surface: nullable<SnowSurfaceDocumentV1>
  shared_style_bindings[]
  target_policy
  field_locks: EnvironmentFieldLockV1[0..256]
```

```text
EnvironmentWorldBindingV1
  binding_id: StableId
  binding_version: positive u32
  binding_content_hash: SHA-256
  world_id: existing WorldDocument StableId
  environment_profile_id: StableId

EnvironmentWorldBindingRefV1
  binding_id: StableId
  binding_version: positive u32
  binding_content_hash: SHA-256

EnvironmentProfileRefV1
  environment_document_ref: exact DocumentRef<EnvironmentSurfaceDocumentV1>
  profile_id: StableId
  profile_schema_version: positive u32
  profile_source_revision: u64
  profile_content_hash: SHA-256
```

`EnvironmentSurfaceDocumentV1`は[Project state](../03-authoring/project-state.md)の正規Project Documentであり、Document indexと`source/worlds/environment/<document_id>/environment-surface.mirakan.json`へ登録する。Source revisionはDocument revisionと一致する。`EnvironmentWorldBindingV1.environment_profile_id`は同じenclosing Document内のProfileへ解決し、`world_id`は同じProject revisionのWorldへ解決する。World側は`global_composition_refs[]`へexact `EnvironmentWorldBindingRefV1`を0または1件だけ持ち、binding追加、削除、Profile切替はEnvironment DocumentとWorld Documentを同じ`ProjectChangeSetV1`でatomicに変更する。Project全体で同じWorldを指すbindingは最大1件であり、bindingが0件なら当該WorldのEnvironment presentationは明示的にabsentとする。配列先頭、最初のProfile、display name、Editor selectionから既定Profileを推測しない。

`binding_content_hash`はASCII `MIRAKAN_ENVIRONMENT_WORLD_BINDING_V1`と自己hashを除く全Fieldのcanonical bytesを`uint32_be` length framingしてSHA-256する。`EnvironmentProfileV1.profile_content_hash`はASCII `MIRAKAN_ENVIRONMENT_PROFILE_V1`と自己hashを除く全Profile Fieldのcanonical bytesを同じlength framingでSHA-256する。`EnvironmentProfileRefV1`は全五Fieldを解決先とbyte equalityにするruntime／Preview／Evidence用のexact revisioned refであり、World Sourceの長寿命selectionには使わない。Runtime generation、GPU handle、LUT／history、derived fieldをSourceへ保存しない。未知field／enum、NaN／Inf、単位なし物理量、collection超過を拒否する。AI、Editor、CLIは同じplanned semantic actionとSourceを使い、raw JSON pointer、shader、Render pass、native resourceを操作しない。Environment authoringのcurrent MCD Operation集合は空であり、Activationまではwrite要求を`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`で拒否する。

本書はPresentationだけを所有する。Fog、GPU water、Snow field、precipitation ParticleをGameplay visibility、friction、Damage、Navigation、AI perceptionへ逆入力しない。Authoritative `GameplaySurfaceState`、contact、Weather rule、buoyancy forceはGameplay／Physics ownersへ委譲し、本書ではその構造やlifecycleを再定義しない。

Capability maturity、Activation、roadmapはProduct Plan、共通phase／lifetimeはRuntime scheduling、共通capacity envelope／測定／backpressureはRuntime performanceだけが所有する。本書のBudgetはdomain固有Source／artifact ceilingである。

### 1.1 設計判定と明示Gap

本計画の推奨判定は「Environment Light Mixer相当のEnvironment Profile authoring設計として採用、実装済み／Activation済みとは扱わない」である。現在の境界を次に固定する。

| 観点 | 設計上の判定 | current非主張 |
|---|---|---|
| AI理解性 | Stable ID／version／hash、closed intent／Preset section、Source／effective分離、origin／reason、bounded候補、closed primitiveで設計を閉じる | Markdown記述だけをMCD、JSON Schema、Provider、callable Toolまたは実行可能validatorとみなさない |
| Environment／Lighting／World連携 | Time-of-Dayはexact Source Sample、sun／moon set equality、単一`ProjectChangeSetV1`でatomicに閉じる | Environmentだけの部分Apply、Editor Lightコピー、未登録Lighting discoveryを成功扱いしない |
| Environment／Post Process責務 | EnvironmentはSky／Atmosphere／Fog／Cloud／IBL、Post Processは露出／metering／adaptation／toneを単独所有する | 同名露出Field、Environment hint、Camera／Lighting側の複製所有を認めない |
| Runtime／Target連携 | Render Snapshot、VFX Weather ref、World binding、exact budget envelope、C1 Core／C2 advanced fallbackへ接続する | 未計測budgetをQualification済み値とせず、Core構成要素の循環fallbackを認めない |
| UI／Product scope | Environment Panelは`basic`／`detailed`で同じcanonical fieldを表示し、Environment authoring 26候補だけをinitial version 1計画集合とする | Panel fixture、Operation family、Preset Catalog、Time Providerは未materialized。Water／Weather／Snow mutationはcurrent Product closure外 |

この表は実装Task、実装順序、期間、担当、依存導入を定義しない。設計状態と非主張を要約するだけで、具体的Activation条件は§6、Product状態はProduct Plan／Execution Registryを正本とする。

## 2. Environment composition、Sky、Fog、Wind

### 2.1 Environment profileとpolicy

```text
EnvironmentProfileV1
  profile_id: StableId
  schema_version: 1
  source_revision: u64
  profile_content_hash: SHA-256
  display_name: string[1..128]
  semantic_intent: EnvironmentIntentV1
  applied_preset_provenance_ref: nullable<EnvironmentPresetRefV1>
  quality_policy: EnvironmentQualityPolicyV1
  sky: SkySourceV1
  atmosphere: AtmosphereSourceV1
  fog: FogSourceV1
  local_fog_volumes: LocalFogVolumeSourceV1[0..128]
  cloud: CloudSourceV1
  lighting: EnvironmentLightingSourceV1
  time_of_day_binding: nullable<TimeOfDayPresentationBindingV1>
  weather_binding: nullable<WeatherBindingV1>
  style_binding: nullable<VisualStyleBindingV1>
```

```text
EnvironmentQualityPolicyV1
  minimum_tier: low | medium | high
  preferred_tier: low | medium | high
  fallback_policy: deny | require_approval | allow_preapproved
  budget_profile_ref:
    exact {budget_profile_id, budget_profile_version, budget_profile_content_hash}

WeatherBindingV1
  weather_profile_ref: exact WeatherPresentationProfileRefV1
  channels: set<wind | cloud_coverage | cloud_density | precipitation | temperature>

VisualStyleBindingV1
  visual_style_profile_ref: exact VisualStyleProfileRefV1

TimeOfDayPresentationBindingV1
  source_ref: PresentationTimeSourceRefV1
  sun_light_id: LightSourceV1 StableId
  moon_light_id: nullable<LightSourceV1 StableId>
  named_phase_q32:
    dawn: u32
    day: u32
    dusk: u32
    night: u32

PresentationTimeSourceRefV1
  provider_ref: exact PresentationTimeSourceProviderRefV1
  source_id: StableId
  source_version: positive u32
  source_content_hash: SHA-256

PresentationTimeSourceProviderRefV1
  provider_id: StableId
  provider_version: positive u32
  provider_content_hash: SHA-256
  owner_manifest_ref: exact registered Owner Manifest ref

PresentationTimeSampleV1
  source_ref: exact PresentationTimeSourceRefV1
  world_id: StableId
  sample_sequence: positive u64
  solar_phase_q32: u32
  discontinuity: bool
  sample_content_hash: SHA-256

PresentationTimeSampleRefV1
  source_ref: exact PresentationTimeSourceRefV1
  world_id: StableId
  sample_sequence: positive u64
  sample_content_hash: SHA-256

EnvironmentPresetRefV1
  catalog_ref: exact {catalog_id, catalog_version, catalog_content_hash}
  preset_id: StableId
  preset_version: positive u32
  preset_content_hash: SHA-256

EnvironmentFieldLockV1
  field_id: u32
  locked_at_project_revision: u64
  owner_kind: human | project_policy
  reason: string[1..256]
```

Tierは`low < medium < high`、minimum≤preferred。preapproved fallbackはTarget manifestに列挙したものだけ、exact ref不一致はstale、lockはstable Field IDを参照し重複を拒否する。`VisualStyleProfileRefV1`と`PaletteTokenRefV1`は[Materials](materials.md)の正本型だけを使う。`PresentationTimeSourceProviderRefV1`はExecutable Contractsの同一Contract setで登録されたProviderとOwner Manifestへ全Fieldでexact解決し、Source refはそのProviderが公開するimmutableなSource定義へexact解決する。Providerは同じWorldに対して単調増加する`sample_sequence`と正規化周期`solar_phase_q32`を持つ`PresentationTimeSampleV1`を公開する。Provider／Sourceのどちらかが未登録、stale、Owner Manifest不一致ならSource unavailableとして扱う。Sample hashはASCII `MIRAKAN_PRESENTATION_TIME_SAMPLE_V1`と自己hashを除く全Fieldのlength-framed canonical bytesからSHA-256し、Sample ref全Fieldを解決先とbyte equalityにする。Engine wall clock、Editor時刻、現在Camera、Light transformから時刻を推測しない。四つのnamed phaseは相互に異なる値で、Presetの`dawn | day | dusk | night`はこの表だけからsolar phaseへ解決する。`discontinuity=true`ではEnvironment／Cloud／Fog historyをresetする。Authorization semanticsはGovernance ownerへ委譲する。

### 2.2 Sky、atmosphere、fog

```text
SkySourceV1
  mode: solid | gradient | hdri | physical_atmosphere
  solid_color: nullable<LinearRgb>
  gradient_zenith: nullable<LinearRgb>
  gradient_horizon: nullable<LinearRgb>
  gradient_ground: nullable<LinearRgb>
  hdri_asset: nullable<HdrCubemapRef>
  rotation_radians: f32
  intensity_multiplier: f32
  sun_light_id: nullable<LightSourceV1 StableId>
  moon_light_id: nullable<LightSourceV1 StableId>
```

Colorはlinear RGB各0～64、rotation `[0,2π)`、intensity `[0,64]`である。Modeに対応するpayloadだけを必須にし、inactive payloadを保持しない。physical atmosphereは`AtmosphereSourceV1.mode=physical`を必要とし、fallbackは明示`precomputed_sky_cubemap`だけ。Lightの量／色／shadowは[Lighting](lighting.md)を参照する。

```text
AtmosphereSourceV1
  mode: disabled | physical
  preset_id: reference_earth_v1 | custom_planet_v1
  custom_planet: nullable<CustomAtmosphereBodyV1>
```

`ReferenceEarthV1`はground／top radius 6,360,000／6,460,000 m、ground albedo `(0.3,0.3,0.3)`、Rayleigh scattering `(5.802,13.558,33.100)*10^-6 m^-1`／scale height 8,000 m／absorption zero、Mie scattering `3.996*10^-6`／absorption `4.400*10^-6 m^-1`／scale height 1,200 m／g 0.8、Ozone absorption `(0.650,1.881,0.085)*10^-6 m^-1`、25,000 mで1・上下15,000 mで0のtriangle density、sun angular radius 0.267 degreeをSource定数とする。

`CustomAtmosphereBodyV1`はradius、全scattering／absorption、density profile、phase g、ground albedoを全件必須としEarth値で部分補完しない。`top_radius > ground_radius > 0`、係数finiteかつnon-negative、`g ∈ [-0.999,0.999]`である。

```text
FogSourceV1
  mode: disabled | height_distance | volumetric
  density_input: direct_extinction | target_visibility
  target_visibility_m: nullable<f32>
  base_extinction_m_inv: nullable<f32>
  height_falloff_m_inv: f32
  base_height_m: f32
  albedo_source: fixed_linear | inherit_sky | palette_token
  scattering_albedo_linear: nullable<LinearRgb>
  palette_token_ref: nullable<PaletteTokenRefV1>
  anisotropy_g: f32
  max_distance_m: f32
  ambient_injection: f32
  local_light_injection: f32
```

Rangeはvisibility 5～1,000,000 m、extinction／falloff 0～10 `m^-1`、base height ±1,000,000 m、albedo各0～1、g -0.99～0.99、max distance 1～1,000,000 m、injection 0～4である。density inputは対応fieldを一つだけ、albedo sourceも対応payloadだけ持つ。

```text
e = clamp(-height_falloff_m_inv * (world_y_m - base_height_m), -80, 80)
extinction(world_y_m) = base_extinction_m_inv * exp(e)
optical_depth = clamp(extinction * distance_m, 0, 80)
transmittance = exp(-optical_depth)
base_extinction_m_inv = 2.995732 / target_visibility_m
```

Visibilityはbase heightのuniform horizontal rayでcontrast 5%となる距離である。Reference defaultはextinction 0.01、falloff 0.10、base height 0、fixed albedo `(0.9,0.9,0.9)`、g 0.2、max distance 10,000 m。

```text
LocalFogVolumeSourceV1
  volume_id: StableId
  shape: box | sphere | ellipsoid | cylinder
  transform: Transform3f
  blend_distance_m: f32
  density_operation: add | subtract | replace
  density_multiplier: f32
  albedo_linear: LinearRgb
  anisotropy_g: f32
  emission_linear: LinearRgb
  noise_asset: nullable<Texture3DRef>
  noise_scale_m: f32
  noise_velocity_m_s: local-space Velocity3f
  priority: i32
  semantic_role: ground_mist | room_haze | dust_shafts | cave_fog | exclusion | custom
```

Dimension各axis 0.01～100,000 m、blendは最短半径以下、multiplierはadd／replace 0～10、subtract -10～0。visible limitはmedium 32／high 128、Source limit 128である。

### 2.3 Cloud、environment lighting、weather input

```text
CloudSourceV1
  mode: disabled | layer_2d | volumetric
  layer_texture_asset: nullable<Texture2DRef>
  bottom_height_m: f32
  top_height_m: f32
  max_view_distance_m: f32
  coverage: f32
  density_multiplier: f32
  reference_asset_set_ref: nullable<ReferenceCloudAssetsRefV1>
  weather_map_asset: nullable<512² RGBA8_UNORM ref>
  base_noise_asset: nullable<128³ R8_UNORM ref>
  detail_noise_asset: nullable<32³ R8_UNORM ref>
  wind_velocity_m_s: world-space Velocity3f
  cast_cloud_shadow: bool

ReferenceCloudAssetsRefV1
  asset_set_id: StableId
  asset_set_version: positive u32
  asset_set_content_hash: SHA-256
  weather_map_asset: exact 512² RGBA8_UNORM ref
  base_noise_asset: exact 128³ R8_UNORM ref
  detail_noise_asset: exact 32³ R8_UNORM ref
```

Bottomは0～100,000 m、top=bottom+1～200,000 m、view distance 1～100,000 m、coverage 0～1、density 0～10、wind各axis ±200 m/s。Referenceはbottom 1,500、top 5,000、distance 100,000、coverage 0.5、density 1.0。Volumetricは`reference_asset_set_ref`一件かProject指定の3 assets全件のexactly-oneを要求する。Reference set使用時は個別3 fieldをcanonical null、Project assets使用時はReference refをcanonical nullとし、部分補完、異なるset versionの混在、Reference refと個別assetの併記を拒否する。

```text
EnvironmentLightingSourceV1
  ibl_mode: baked | dynamic_incremental
  diffuse_irradiance_enabled: bool
  specular_prefilter_enabled: bool
```

露出、metering、adaptation、tone mappingは[Post Processing](post-processing.md)の`ExposureProfileV1`だけがSource ownerである。Environment Source、Preset、Intent、Summary、fallbackは露出値を持たず、Environment Panelから露出を変更する場合もPost Processの別Proposalへ明示的に遷移し、Environment ChangeSetへ複写しない。Baked IBLはdiffuse 32²×6 `RGBA16_FLOAT`、specular 256²×6・9 mip `RGBA16_FLOAT`、BRDF LUT 256² `RG16_FLOAT`、HDR sourceを除くpersistent set 8 MiB以下である。

`EnvironmentLightingSummaryV1`は本書がOwnerとして公開するread-only／revisioned projectionであり、[Lighting](lighting.md) §6の`LightIntentResolverV1`入力である。最低fieldとしてrevision、exact `EnvironmentProfileRefV1`、sky kind、sun／moon Light Stable ID、`ibl_mode`、diffuse／specular IBL有効状態、合成後実効cloud coverage、`cast_cloud_shadow`を持つ。露出またはPost Process fieldを含めない。revisionは入力SourceのDocument revision、Weather合成generation、nullable Presentation Time Sample hashから決定的に導出し、同一入力から同一revisionを生成する。消費側はfield一覧を複写せず、Summary経由でSourceへ書き戻さない。

```text
WeatherPresentationProfileV1
  profile_id: StableId
  schema_version: 1
  source_revision: u64
  profile_content_hash: SHA-256
  precipitation_kind: none | rain | snow | sleet
  precipitation_rate_mmph
  wind_velocity_world_mps
  air_temperature_c
  cloud_coverage
  cloud_density_multiplier
  snow_accumulation_rate_mps
  snow_melt_rate_mps
  transition_seconds

WeatherPresentationProfileRefV1
  environment_document_ref: exact DocumentRef<EnvironmentSurfaceDocumentV1>
  profile_id: StableId
  profile_schema_version: positive u32
  profile_source_revision: u64
  profile_content_hash: SHA-256
```

```text
WeatherPresentationSnapshotV1
  snapshot_id: StableId
  generation: positive u64
  snapshot_content_hash: SHA-256
  environment_profile_ref: exact EnvironmentProfileRefV1
  weather_profile_ref: exact WeatherPresentationProfileRefV1
  precipitation_kind
  precipitation_rate_mmph
  wind_velocity_world_mps
  air_temperature_c
  effective_cloud_coverage
  effective_cloud_density_multiplier
  snow_accumulation_rate_mps
  snow_melt_rate_mps

WeatherPresentationSnapshotRefV1
  snapshot_id: StableId
  generation: positive u64
  snapshot_content_hash: SHA-256
```

Rangeはprecipitation `[0,500] mm/h`、wind各axis `[-150,150] m/s`、temperature `[-100,100] C`、cloud coverage `[0,1]`、cloud density multiplier `[0,10]`、accumulation／melt `[0,0.01] m/s`、transition `[0,600] s`。Profile hashはASCII `MIRAKAN_WEATHER_PRESENTATION_PROFILE_V1`と自己hashを除く全Fieldのlength-framed canonical bytesからSHA-256し、Ref全五Fieldを解決先とbyte equalityにする。`WeatherBindingV1.channels`が`cloud_coverage`を含む場合は`CloudSourceV1.coverage`をweather値でoverrideし、`cloud_density`を含む場合は`CloudSourceV1.density_multiplier`へ`cloud_density_multiplier`を乗算する。いずれも`transition_seconds`で補間し、channel未選択のfieldはCloudSourceV1のauthored値を維持する。§5のweather coverage／density history破棄条件はこの合成後の実効値を入力とする。`WeatherPresentationSnapshotV1`は同generationをEnvironment、VFX、Snowへpublishし、Gameplay weatherへ逆入力しない。VFXとSnowはSnapshot refを消費し、Weather fieldを複写した別generationを作らない。

## 3. Water surface／body profile

```text
WaterSystemDocumentV1
  schema_version
  system_id
  bodies: WaterBodyV1[]
  material_profiles[]
  gravity_mps2
  quality_profile_id
  target_policy
  fallback_variants[]

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

Body kindは`bounded_plane` finite rectangle、`mesh_region` manifold non-vertical surface、`lake` closed polygon・hole最大16、`spline_river` non-self-intersecting splineとwidth／water-level／flow keys、`ocean` camera-relative clipmapである。Overlap selectionは`priority desc, body-kind specificity desc, body_id asc`、同priority／同kindのvolume intersectionはCook error。

`WaterSurfaceProfileV1`はbase color、absorption／scattering `m^-1`、roughness、normal texture／scale／scroll、Fresnel F0、foam Material refを持つ。`WaterVolumeProfileV1`はbelow-surface closed volume、priority、underwater profile、Gameplay Surface IDを持つ。Visual displacementをCollider／Gameplay water levelへ反映しない。

Baseline surfaceは最大2 normal textureを独立方向／速度でscrollしstatic meshを変位させない。`DirectionalWaveSetV1`は`DirectionalWaveComponentV1[0..8]`を持つ。

```text
DirectionalWaveComponentV1
  direction_xz: normalized vec2f
  amplitude_m: f32        # [0,20]
  wavelength_m: f32       # [0.25,2000]
  phase_rad: f32
  steepness: f32          # [0,1]

k = 2*pi/wavelength
omega = sqrt(gravity_mps2 * k)
```

`gravity_mps2`は波分散専用のscalar f32 Source定数であり、範囲`[0.1,100] m/s^2`、既定9.80665とする。PhysicsのWorld Profile gravityを暗黙参照せず、2D／3D Projectで同じfieldを使う。CPU QueryとGPU vertexは同じ生成定数／式を使う。各componentの時刻位相`omega*t`はScheduling Ownerの`SimulationAdvanceIntervalV1.interval.logical_duration_seconds`から得た同じtime stateをCPUがf64で累積し`[0,2π)`へ折り畳んだ定数としてGPUへ渡し、大きな累積引数のままsinを評価しない。duration-null Cadenceではqualified advance-driven wave policyなしに進めない。空間位相はworld-origin rebase後のcamera近傍座標で評価する。一致fixtureはorigin±2,048 m、開始advanceからexact 24時間相当のProfile／Interval列へ固定した65,536 reference pointsでheight error最大2 mm、normal angle最大0.25 degreeを満たす。`FlowProfileV1`はconstant flowまたはCooked current mapを持ち、RG normalized direction、B `[0,1]` strength、profile max speedを使う。Gameplayはtextureを直接sampleせず同じfieldのCPU artifactを読む。

Depthは[Terrain／Foliage](terrain-foliage.md)またはWorld／Mesh ownerのexact offline surface sourceを使い、scene depthをcanonical water depthにしない。Waterはsurface normal、roughness、absorption、IOR、radiance／receiver summaryを[Advanced Light Transport](advanced-light-transport.md)へ渡し、reflection Technique、IBL／probe／screen-space／ray fallbackを本書で選ばない。UnderwaterはWater Volume overlapからabsorption／scattering／fog／waterlineを選ぶ。Boundary 5 cm以内はprevious Body保持、10 cm外で解除する。

```text
WaterQueryRequestV1
  query_id: u64
  position_world_m: vec3f
  body_layer_mask: u64
  simulation_cadence_profile_ref: SimulationCadenceProfileRefV1
  simulation_advance_interval_hash: SHA-256
  snapshot_advance_sequence: u64

WaterQueryResultV1
  status: Success | OutsideWater | UnsupportedAtLevel | StaleSnapshot | InvalidRequest | QueueFull | BackendFailure
  body_id
  surface_position_world_m
  surface_normal_world
  flow_velocity_mps
  depth_m
  query_generation
```

`WaterQueryPortV1`はpublish済みCPU artifactだけを読み、GPU buffer／render depth／SSR／foam／VFX collisionをreadbackしない。RequestのProfile ref／Interval hash／advance sequenceは同じpublish snapshotとbyte equalityにする。Baselineはflat surface／zero flow、advancedはanalytic wave／flow／depthを返す。Windows 16,384 query／advance、Mobile 4,096／advanceで、超過は`QueueFull`と`WaterQueryQueueFull`を返し前frame値へfallbackしない。`StaleSnapshot`は`WaterQueryStaleSnapshot`を発行し、`InvalidRequest | BackendFailure | UnsupportedAtLevel | OutsideWater`を`Success`へ置換しない。PhysicsだけがQuery結果からbuoyancy forceを計算し、WaterはBodyへForceを加えない。

SaveはBody Asset revision、Gameplay water state、解析波のCadence Profile ref／hash、開始advance sequence、exact elapsed-time accumulatorだけを永続化し、GPU texture、foam、SSR historyを保存しない。Body Asset revisionまたはCadence Profile不一致は現在値へ黙って読み替えずmigrationまたはload failureとし、解析波は保存したtime stateから同じ生成定数／式で再構築する。Gameplay stateのschema、Save／Replay envelope、checkpointはGameplay／Runtime Ownerを参照し、本書で複写しない。

`WaterInteractionEventV1`はauthoritative Collision／Gameplayだけが生成し、次の7 required scalar fieldをちょうど一つずつ持つ。optional field、配列、unknown fieldを許可しない。

```text
WaterInteractionEventV1
  event_id: u64
  body_id: StableId
  position: world-space vec3f
  normal: normalized world-space vec3f
  relative_velocity_mps: world-space vec3f
  interaction_kind: enter | exit | impact | wake_request
  vfx_seed: u64
```

non-finite position／velocity、非正規normal、存在しないBody、unknown `interaction_kind`、duplicate `event_id`をschema／reference failureとしてEvent全体ごと拒否し、既定kind／seedを生成しない。同一advanceの配送は[Runtimeのcanonical message order](../04-runtime/scheduling-lifetime.md#10-message-mergeasync-acceptancerandomness)を使い、worker completion順で並べない。Replay envelopeは受理済みEventの7 fieldとaccept advance sequence／Interval hashを記録し、同じ`event_id`／`vfx_seed`でVFX／Audioへ再配送する。Splash／foam／ripple／wakeはPresentationであり、Water Query、Damage、buoyancyへ戻さず、GPU生成結果をReplay／Saveへ含めない。

## 4. Weather state、snow／wetness response

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

Camera volumeはfinite boxである。`density_scale`はauthored基準値であり、LODによる降雪particle密度の倍率は`vfx_presentation` classの`VfxLodProfileV1`だけが所有する。BaselineはCPU Billboard、gravity／drag／Color・Size over Life、advancedはGPU Billboard、turbulence、visual collision。Particle hitからaccumulation、friction、footprint、Gameplay eventを生成しない。

```text
SnowSurfaceDocumentV1
  document_id
  receivers: SnowReceiverV1[]
  static_masks[]
  dynamic_field_profile_id
  material_profile_id
  quality_profile_id
  fallback_variants[]
```

`SnowReceiverV1`はreceiver Stable ID、geometry Asset、world bounds、surface layer、static mask、dynamic enabled、priorityを持つ。Static maskはlinear R8_UNORM、0＝none、255＝full。UVなしMeshはworld projectionを明示しない限りCook error。Material coverageはstatic／dynamic mask、world normal dot up、allow mask、height bias、`retention_q16`を掛け、coverage 0ならsnow layerを描画しない。baselineはgeometry displacementなし。

`SnowDynamicFieldProfileV1`はworld-aligned paged atlas、`max_depth_m ∈ [0.01,2.0]`、`melt_threshold_c: f32 ∈ [-100,100]`（既定0）、`retention_q16: Q0.16 u16`（既定65,535）をclosed fieldとして持つ。`retention_q16`は`SnowApplyWeather`、`melt_threshold_c`は`SnowResolveMelt`だけが読む。Page resolutionは128²、desktop texel 0.25 m／active 256、mobile standard 0.50 m／active 64、format R16G16_UNORM、R=coverage／depth、G=compaction。page keyはinteger world cell `(x,z)`とfield generationでありcamera distanceをidentityにしない。

Coverage／compactionはQ0.16 `u16`である。

```text
weather_delta = round_ties_to_even(
  clamp(rate_mps * logical_duration_seconds * retention_q16
        / (max_depth_m * 65535), 0, 1) * 65535)
next = saturate_to_u16(signed_i32(current) + signed_q0_16_delta)
```

Pass順は`SnowResetNewPages -> SnowApplyWeather -> SnowApplyStamps -> SnowResolveMelt -> SnowBuildMaterialBindings`。unordered同pass writeを禁止する。page overflowは`receiver priority desc, camera distance asc, page key asc`末尾を更新対象外にし、explicit static-mask fallbackがあるreceiverだけ描画を続ける。

```text
SnowSurfaceInteractionEventV1
  event_id
  cadence_profile_ref: SimulationCadenceProfileRefV1
  simulation_advance_interval_hash: SHA-256
  advance_sequence: positive u64
  receiver_id
  position_world_m
  direction_world
  radius_m
  depth_delta
  compaction_delta
  stamp_asset_id
  presentation_seed
```

Snow field updateはScheduling Ownerの一件の`SimulationAdvanceIntervalV1`を消費し、EventのProfile ref／interval hash／advance sequenceは同recordとbyte equalityにする。`logical_duration_seconds`は同recordのnon-null exact rational durationだけを使い、Cadence rate、render frame、wall time、表示名、直前値から推測しない。duration-nullのturn-based／explicit-stepでは`SnowApplyWeather`と時間依存meltを進めず、0秒へ変換した計算結果として扱わない。受理済みstampだけをcanonical順で適用し、時間依存Snow更新は対応するCadence別Qualificationが追加されるまで`cadence_profile_not_qualified`とする。

Radius `[0.01,5] m`、delta `[-1,1]`、desktop 1,024／Simulation Advance、mobile 256／Simulation Advance。Overflowは`priority desc, event_id asc`末尾drop、繰越なし。AccumulationはWeather rate×`retention_q16`×同recordのduration、meltはair temperatureが`melt_threshold_c`超過時、compactionはstampだけが増やす。Dynamic GPU pages、elapsed accumulator、start sequence、全stamp履歴はPresentation-onlyでありSave contractからcanonical omissionとする。Loadはstatic maskとcurrent Weather snapshotから空のdynamic fieldを明示的に開始し、load前のfootprint／compaction再現を保証済みと表示しない。Replayでvisual field再現を要求するProfileは全`SimulationAdvanceIntervalV1` recordと受理済みEvent三Fieldを記録し、欠落時はcomplete visual Replayを名乗らない。

WetnessはWeather rain／sleet、Water interaction、receiver allow maskからMaterial ownerのsurface semanticへtyped inputを出す。Snow coverage／compactionと同じreceiver identityを使うが値をaliasせず、Gameplay friction／surface stateを変更しない。

## 5. Renderer／Material interfaceとruntime compilation

EnvironmentはSky radiance、sun／moon refs、Fog／Cloud、IBL、Waterへ入射lightを提供し、Advanced Light Transportへexact radiance／participation summaryを渡す。露出はEnvironment入力でなく、World HDRを測光するPost Process stageである。WaterはSurface／Underwater packetと`WaterSnapshotV1`、Snowはcoverage／compaction bindingを出す。Materialsは正規semantic role `surface.water`と`surface.snow`を所有し、本書はMaterial IR／Shading Modelを複写しない。Terrain／Foliageはsnow／wetness receiver、vegetation response bindingを消費するが、weather／surface condition authorityを持たない。VFXはshort-lived smoke／precipitation／splashを所有し、VFXからLocal Fog／Snow Sourceへ暗黙変換しない。

Rendererへ渡すEnvironment-owned frame inputを次の四型に閉じる。いずれもSourceでなく、同じpublished Simulation AdvanceとEnvironment artifact generationから作るimmutable projectionである。

```text
EnvironmentPresentationSnapshotV1
  generation: positive u64
  environment_profile_ref: exact EnvironmentProfileRefV1
  artifact_set_ref: exact EnvironmentArtifactSetRefV1
  weather_snapshot_ref: nullable<WeatherPresentationSnapshotRefV1>
  presentation_time_sample_ref: nullable<PresentationTimeSampleRefV1>
  sun_light_id: nullable<LightSourceV1 StableId>
  moon_light_id: nullable<LightSourceV1 StableId>
  selected_fallback_refs[]

WaterSnapshotV1
  generation: positive u64
  source_revision: u64
  cadence_profile_ref: SimulationCadenceProfileRefV1
  simulation_advance_interval_hash: SHA-256
  advance_sequence: positive u64
  active_body_refs[]
  surface_batch_refs[]
  underwater_batch_refs[]
  query_generation: positive u64

SnowSurfaceBatchV1
  generation: positive u64
  source_revision: u64
  weather_snapshot_ref: nullable<WeatherPresentationSnapshotRefV1>
  receiver_binding_refs[]
  page_manifest_ref
  material_binding_refs[]

EnvironmentArtifactSetRefV1
  artifact_set_id: StableId
  generation: positive u64
  artifact_set_content_hash: SHA-256
```

`EnvironmentArtifactSetRefV1`と`WeatherPresentationSnapshotRefV1`はそれぞれ完成Artifact setとSnapshotへ全三Fieldでexact解決する。`PresentationTimeSampleRefV1`も解決先Sampleのsource ref、World、sequence、hashとbyte equalityにする。Artifact set hashはASCII `MIRAKAN_ENVIRONMENT_ARTIFACT_SET_V1`と自己hash／`resource_receipt`を除くreceipt-free closed record、Weather hashはASCII `MIRAKAN_WEATHER_PRESENTATION_SNAPSHOT_V1`と自己hashを除くclosed recordのcanonical bytesを`uint32_be` length framingしてSHA-256する。`EnvironmentPresentationSnapshotV1.weather_snapshot_ref`、`SnowSurfaceBatchV1.weather_snapshot_ref`、VFXが消費するWeather refは、Weather bindingが存在する時に同じ一件へbyte equalityで解決し、存在しない時は全てcanonical nullとする。Time-of-Day bindingが存在する時は`EnvironmentPresentationSnapshotV1.presentation_time_sample_ref`をpublishに使った同一Sampleへexact解決し、bindingなしではcanonical nullとする。`WaterSnapshotV1`のCadence三値は同じpublish対象`SimulationAdvanceIntervalV1`とbyte equalityにする。Rendererは四型のfieldを再解釈、補完、相互変換せず、Source Profileへ書き戻さない。

Advanced Light Transportへ渡すEnvironment-owned radiance inputを次のimmutable projectionに閉じる。

```text
EnvironmentRadianceSummaryV1
  schema_version: 1
  summary_id: content-derived StableId
  generation: positive u64
  project_revision: positive u64
  world_scope_ref: exact WorldScopeRefV1
  target_profile_ref: exact TargetProfileRefV1
  quality_profile_ref: exact QualityProfileRefV1
  environment_profile_ref: exact EnvironmentProfileRefV1
  artifact_set_ref: exact EnvironmentArtifactSetRefV1
  weather_snapshot_ref: nullable<WeatherPresentationSnapshotRefV1>
  presentation_time_sample_ref: nullable<PresentationTimeSampleRefV1>
  radiance_sources[1..6]:
    source_kind:
      sky | atmosphere | ibl_diffuse | ibl_specular | cloud | volumetric
    participation:
      diffuse_indirect | specular_indirect | volumetric_receive
    availability: available | unavailable
    source_generation_ref: exact owner-typed generation ref
  summary_content_hash: SHA-256

EnvironmentRadianceSummaryRefV1
  summary_id: StableId
  generation: positive u64
  summary_content_hash: SHA-256
```

`radiance_sources[]`は`source_kind, participation` enum ordinal順へstrict sortしてduplicateを拒否し、同じEnvironment generationで有効または明示unavailableなradiance source closureとset equalityにする。Profile、Artifact、Weather、Time refsは同じ`EnvironmentPresentationSnapshotV1` generationの各refとbyte equalityにする。Refは解決先三Fieldとbyte equalityにし、summary hashはASCII `MIRAKAN_ENVIRONMENT_RADIANCE_SUMMARY_V1`と自己hashを除くclosed recordのcount／presence／length-framed canonical bytesからSHA-256する。ALTはこのprojectionから露出、Light Source、Material responseまたはTechniqueを生成しない。

`EnvironmentCompiler`はcanonical Source、Contract set、Target、Toolchainからdeterministic artifactを生成する。

```text
EnvironmentArtifactSetV1
  artifact_set_id
  generation: positive u64
  artifact_set_content_hash: SHA-256
  source_profile_id
  source_revision
  contract_set_hash
  target_profile_ref: exact TargetProfileRefV1
  quality_tier
  sky_artifact
  atmosphere_lut_artifact
  fog_profile_artifact
  cloud_artifact
  ibl_artifact
  fallback_manifest
  resource_receipt
```

同じclosureからWater mesh／depth／query、Snow static-mask／receiver／page manifest、Weather snapshot descriptorも同一Cook closureへ生成する。一部Artifactだけをpublishせず、全ready時にatomic generation promotionする。Source hashから再生成可能にしDerived binaryを正本にしない。`EnvironmentPresentationSnapshotV1`、`WeatherPresentationSnapshotV1`、`WaterSnapshotV1`、`SnowSurfaceBatchV1`はgeneration付きimmutable valueである。

`AtmosphereLutProfile::ReferenceV1`のLUTはTransmittance 256×64 `RGBA16_FLOAT`／40 step、Multiple scattering 32×32／20、Sky view 200×100／30、Aerial perspective 32³／30である。Jittered stratified midpointを使い、optical-depth capまたはplanet boundaryで終了する。Dependency変化時に個別再生成するが4枚readyまで旧generationを保持する。

Volumetric Fogはmedium最大160×90×64、high 240×135×96、XY=`min(ceil(internal/12),160×90)`または`min(ceil(internal/8),240×135)`、Z width ratio 1.2。current／history scatteringは各`RGBA16_FLOAT`、extinctionは`R16_FLOAT`。`fog_far=min(max_distance,camera_far)`かつ`fog_far>near`である。

```text
delta_z0 = (fog_far-near) * (1.2-1) / (1.2^N-1)
delta_zi = delta_z0 * 1.2^i
```

Volumetric Cloud mediumは1/4-linear最大480×270、primary／light 48／6 step、highは1/2-linear最大960×540、96／8 step。`step_length=(exit-entry)/N`、sample=`entry+(i+blue_noise_0_1)*step_length`、phaseは256 frames cycle、opacity 0.995またはscene depthでearly-outする。Cloud shadowはmedium 1,024² R16F／4 frames、high 2,048²／2 frames、sun 1 degreeまたはweather generation変化で即時更新する。

Dynamic IBLは6 face＋1 diffuse＋9 specular mip＝16 work units、最大1 unit／frame、全readyでatomic swapする。unit soft ceilingは0.25 ms、超過workを次frameへ送りpartial generationを公開しない。

Fog／Cloud historyはcamera cut、world-origin rebase、internal extent、non-jitter projection、Environment generation、sun 1 frame 5 degree、weather coverage／density max delta 0.20で破棄する。破棄frame weight 0、次7 framesで`min(0.90, valid_frames/8)`まで増やす。編集不可のderived policyである。

Water／SnowのOwner payloadを次に閉じる。両Profileは[LOD §3.2](lod.md#32-lodpolicysetv1)の共通identity、Representation Set、tier binding、hashを再定義しない。

```text
WaterLodProfileV1 =
  LodDomainProfileV1<WaterLodProfilePayloadV1>

WaterLodProfilePayloadV1
  schema_version: 1
  tier_metadata[1..32]:
    tier_id: StableId
    surface_patch_density_profile_ref: exact owner-typed ref
    wave_shading_profile_ref: exact owner-typed ref
    specular_transport_participation_ref:
      exact WaterSpecularTransportParticipationRefV1
    underwater_presentation_profile_ref: exact owner-typed ref | null
    foam_spray_vfx_tier_ref: exact LodTierRefV1 | null
  gameplay_water_invariance_contract_ref: exact owner-typed ref
  payload_content_hash: SHA-256

SnowSurfaceLodProfileV1 =
  LodDomainProfileV1<SnowSurfaceLodProfilePayloadV1>

SnowSurfaceLodProfilePayloadV1
  schema_version: 1
  tier_metadata[1..32]:
    tier_id: StableId
    dynamic_field_update_distance_mm: u64
    normal_detail_profile_ref: exact owner-typed ref
    sparkle_detail_profile_ref: exact owner-typed ref
    static_mask_fallback_ref: exact owner-typed ref
  gameplay_surface_invariance_contract_ref: exact owner-typed ref
  payload_content_hash: SHA-256
```

`specular_transport_participation_ref`はWaterのroughness／normal／coverage／receiver意味だけを持つEnvironment-owned summaryである。reflection Technique、quality tier、history、fallback、Target supportは[Advanced Light Transport](advanced-light-transport.md)が解決し、このLOD payloadへ複写しない。

各tier ID集合はEnvelopeの`tier_bindings[].tier_id`とset equalityにする。payload hashはそれぞれASCII `MIRAKAN_WATER_LOD_PROFILE_PAYLOAD_V1`／`MIRAKAN_SNOW_SURFACE_LOD_PROFILE_PAYLOAD_V1`と自己hashを除くcanonical payload bytesから計算し、Envelopeの`owner_payload_content_hash`とbyte equalityにする。WaterからVFX tierへの依存は一方向であり、VFX ProfileからWater Profileを参照してはならない。exact owner-typed refのSchemaまたはCapabilityが未登録ならProfileをmaterializeせず、値を推測しない。

Water orderingはOpaque／Lighting、Environment、Water Depth／Surface、Underwater／Waterline、Transparent／World VFXである。Water resource generationをsnapshot dependencyへ含め、LODはEnvironment-owned `WaterLodProfileV1`／`LodResolutionPlanV1`だけからpatch density、wave shading、specular transport participation summary、underwater、foam／spray presentation tierを選ぶ。reflection Technique／fallbackは選ばない。SnowはEnvironment-owned `SnowSurfaceLodProfileV1`だけからupdate distance、normal／sparkle、static fallbackを選び、降雪particle密度は`vfx_presentation` classの`VfxLodProfileV1`だけから選ぶ。CPU query、Volume、Gameplay water level、Snow page identity／stampをLODで変えない。

Snow computeはWorld opaque Materialより前に完了し、same-frame finalized fieldだけをsampleする。Pixel-locked 2Dはatlasでなくexplicit Tile／Sprite snow variantを使う。Device loss時はSource／receiptからpersistent Environment、Water material／mesh／query、Snow mask／manifest／empty fieldを復元し、historyを破棄、bounded warm-upし、one-shot VFXを再発火しない。

## 6. AI／Editor operation、preview、validation

Environmentの予約候補は`operation.environment.inspect_profile, operation.environment.explain_effective, operation.environment.list_presets, operation.environment.resolve_intent, operation.environment.validate_changeset, operation.environment.preview_changeset, operation.environment.estimate_cost, operation.environment.create_profile, operation.environment.set_world_binding, operation.environment.apply_preset, operation.environment.set_intent, operation.environment.set_sky, operation.environment.set_sun_moon_link, operation.environment.set_height_distance_fog, operation.environment.set_volumetric_fog, operation.environment.create_local_fog_volume, operation.environment.update_local_fog_volume, operation.environment.delete_local_fog_volume, operation.environment.set_atmosphere_preset, operation.environment.set_custom_atmosphere, operation.environment.set_cloud_layer, operation.environment.set_lighting, operation.environment.bind_weather, operation.environment.generate_fallback, operation.environment.bake, operation.environment.run_qualification`のexact 26件であり、[Executable contracts](../02-foundation/executable-contracts.md#211-既存domain文書から回収した未登録operation候補)の`planning.operation_family.environment_authoring@1`だけに属する。Capability stateは`not_activated`、current MCD／Owner Manifest／Service allowlist／Policy／Validator／Diagnostic／Receipt／Provider／MCP／generated alias／legacy alias集合はすべて`[]`である。`activation.environment.authoring_operations.v1`が26件を同じContract set transactionで完全登録するまでGatewayはdispatchせず、要求を`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`でSource不変として拒否する。過去draftの25件集合、alias、部分Activationを禁止する。Waterの`CreateWaterBody, SetWaterBoundary, SetWaveProfile, SetFlowProfile, SetWaterMaterial, SetUnderwaterProfile, PreviewWaterCost`とWeather／Snowの`SetWeatherPresentation, CreateSnowReceiver, PaintStaticSnowMask, SetSnowMaterial, EnableDynamicSnow, PreviewSnowCost`はStable IDでないfuture action labelであり、current Operation／planning candidateへ推測しない。`wp.rendering.environment-c2`と`capability.environment.*`はEnvironment Profileのauthoring／runtime presentationだけをProduct closureとし、Water／Weather／Snow mutation capabilityを成立済みとみなさない。三領域のSource／Runtime schemaは設計契約として維持するが、別Owner family、Operation全件、Policy、Validator、Receipt、Provider、Target Qualificationが同じActivationで登録されるまではread-only／not activatedである。`operation.lod.*`も不完全prefixであり、LOD ownerのexact二候補だけを別familyで扱う。Write／ChangeSetの意味は各familyのActivation後だけ有効である。

OperationからProject mutation／Jobへの閉じ方を次に固定する。

| Operation群 | canonical result | Project mutation |
|---|---|---|
| `inspect_profile, list_presets` | bounded Source／Catalog projection | なし |
| `explain_effective` | `EnvironmentEffectiveStateV1`＋`EnvironmentPlanExplanationV1` | なし |
| `resolve_intent, validate_changeset, preview_changeset, estimate_cost` | bounded resolution／Proposal validation／Preview Receipt | なし |
| `create_profile, set_world_binding, apply_preset, set_intent, set_sky, set_sun_moon_link, set_height_distance_fog, set_volumetric_fog, create_local_fog_volume, update_local_fog_volume, delete_local_fog_volume, set_atmosphere_preset, set_custom_atmosphere, set_cloud_layer, set_lighting, bind_weather, generate_fallback` | `EnvironmentChangeSetProposalV1` | 下記exact `EnvironmentChangePrimitiveV1` branch |
| `bake` | immutable Source revisionを入力にするoffline Job Proposal | なし |
| `run_qualification` | immutable Artifact／Target closureを入力にするQualification Job Proposal | なし |

`EnvironmentChangePrimitiveV1`のclosed branchは`CreateEnvironmentProfile, SetEnvironmentWorldBinding, ApplyEnvironmentPreset, SetEnvironmentIntent, SetEnvironmentSky, SetEnvironmentSunMoonLink, SetEnvironmentHeightDistanceFog, SetEnvironmentVolumetricFog, CreateEnvironmentLocalFogVolume, UpdateEnvironmentLocalFogVolume, DeleteEnvironmentLocalFogVolume, SetEnvironmentAtmospherePreset, SetEnvironmentCustomAtmosphere, SetEnvironmentCloudLayer, SetEnvironmentLighting, BindEnvironmentWeather, SetEnvironmentFallbackSelection`である。全branchは`primitive_id`、exact Environment Document ref、target Profile／Volume／binding Stable ID、expected Environment Document revision、`locked_field_ids[]`、`declared_cost_ref`を共通fieldとし、payloadを次のclosed型へ固定する。

| branch | exact payload |
|---|---|
| `CreateEnvironmentProfile` | 完全な`EnvironmentProfileV1`一件 |
| `SetEnvironmentWorldBinding` | before／after `EnvironmentWorldBindingV1`とbefore／after World `global_composition_refs[]` |
| `ApplyEnvironmentPreset` | 完全な`EnvironmentPresetApplicationV1` |
| `SetEnvironmentIntent` | 完全な`EnvironmentIntentV1` |
| `SetEnvironmentSky` | 完全な`SkySourceV1` |
| `SetEnvironmentSunMoonLink` | before／after Skyおよび`TimeOfDayPresentationBindingV1`のsun／moon ID |
| `SetEnvironmentHeightDistanceFog`／`SetEnvironmentVolumetricFog` | 完全な`FogSourceV1` |
| `CreateEnvironmentLocalFogVolume`／`UpdateEnvironmentLocalFogVolume` | 完全な`LocalFogVolumeSourceV1`一件 |
| `DeleteEnvironmentLocalFogVolume` | exact Volume Stable IDとbefore content hash |
| `SetEnvironmentAtmospherePreset`／`SetEnvironmentCustomAtmosphere` | 完全な`AtmosphereSourceV1` |
| `SetEnvironmentCloudLayer` | 完全な`CloudSourceV1` |
| `SetEnvironmentLighting` | 完全なIBL-only `EnvironmentLightingSourceV1` |
| `BindEnvironmentWeather` | nullableな完全`WeatherBindingV1` |
| `SetEnvironmentFallbackSelection` | 完全な`EnvironmentQualityPolicyV1`と選択fallback ref集合 |

```text
EnvironmentPresetApplicationV1
  base_profile_ref: exact EnvironmentProfileRefV1
  preset_ref: exact EnvironmentPresetRefV1
  preset_section_replacements[1..11]: EnvironmentSectionReplacementV1
  explicit_section_overrides[0..11]: EnvironmentSectionOverrideV1
  result_profile: complete EnvironmentProfileV1
```

両section unionのbranchとpayloadを次に閉じる。

| branch | complete payload |
|---|---|
| `semantic_intent` | `EnvironmentIntentV1` |
| `quality_policy` | `EnvironmentQualityPolicyV1` |
| `sky` | `SkySourceV1` |
| `atmosphere` | `AtmosphereSourceV1` |
| `fog` | `FogSourceV1` |
| `local_fog_volumes` | `LocalFogVolumeSourceV1[0..128]` |
| `cloud` | `CloudSourceV1` |
| `lighting` | `EnvironmentLightingSourceV1` |
| `time_of_day_binding` | `nullable<TimeOfDayPresentationBindingV1>` |
| `weather_binding` | `nullable<WeatherBindingV1>` |
| `style_binding` | `nullable<VisualStyleBindingV1>` |

Compilerはbase ProfileへCatalog entryとbyte equalityなPreset replacement、明示overrideの順で適用してresult Profileとのbyte equalityを検証し、`applied_preset_provenance_ref`へPreset refを保存する。PresetはRuntime inheritanceではなく、一回のChangeSetで完全なSourceへmaterializeされるため、後のCatalog更新で既存Profileをsilent変更しない。

`EnvironmentSectionReplacementV1`／`EnvironmentSectionOverrideV1`は上表のclosed tagged unionを使う。同じbranchの重複を拒否し、branch ID順へstrict sortする。OverrideはPreset entryが置換するbranchだけに許可し、変更対象のstable Field lockを一件でも含めば全Applicationを拒否する。raw JSON pointer、untyped object、不完全Source fragment、unknown branch、暗黙型変換を拒否する。各primitive branchは[Project state](../03-authoring/project-state.md)の`ProjectChangePrimitiveV1` owner namespaceへ登録する。`SetEnvironmentWorldBinding`はEnvironment Document内bindingと対象Worldの`global_composition_refs[]`を同じChangeSetで変更し、片側だけのmutationを拒否する。Operation名からprimitive名を生成せず、Activation closureが上表のOperation→result→branch coverage 100%を検証する。

```text
EnvironmentChangeSetProposalV1
  proposal_id: StableId
  proposal_version: positive u32
  proposal_content_hash: SHA-256
  base_project_revision: positive u64
  environment_document_ref: exact DocumentRef<EnvironmentSurfaceDocumentV1>
  world_id: StableId
  target_profile_refs[1..64]
  primitives[1..256]: EnvironmentChangePrimitiveV1
  companion_lighting_changeset_ref: nullable<LightingChangeSetProposalRefV1>
  risk_summary
  preview_receipt_ref: nullable<exact EnvironmentPreviewReceiptRefV1>
  expires_at
```

これは共通Proposal envelopeのDomain projectionであり、直接Commitしない。Time-of-Day変更がLight Source変更を伴う場合、exact `LightingChangeSetProposalRefV1`を必須とし、両Proposalのbase Project revision、World、Target集合、expiryをbyte equalityにする。Environment primitive、Lighting primitive、必要なWorld binding primitiveを一つの`ProjectChangeSetV1`へmaterializeしてatomic commitし、片方だけのPreview、Approval、Apply、Commit、retryを拒否する。sun／moonの`affected_light_ids[]`はbindingのnon-null sun／moon ID集合とexact set equalityにし、追加Lightを同じcompanionへ混在させない。

Environment Capability IDは`capability.environment.core, capability.environment.sky_hdri, capability.environment.height_fog, capability.environment.ibl_baked, capability.environment.atmosphere_lut, capability.environment.aerial_perspective, capability.environment.volumetric_fog, capability.environment.local_fog_volume, capability.environment.volumetric_cloud, capability.environment.dynamic_ibl, capability.environment.cloud_shadow, capability.environment.intent_resolver`に閉じる。共通`capabilities.search`／`capabilities.read` projectionはExecutable contracts ownerを参照し、maturityを本書へ複写しない。

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
  target_visibility_m: nullable<f32>
  palette_token_ref: nullable<PaletteTokenRefV1>
```

Reference intent mappingを次に閉じる。

| Intent | Resolver定数／規則 |
|---|---|
| `clear／light_haze／light_mist／fog／dense_fog` | target visibility 20,000／2,000／800／200／50 m |
| `uniform` | height falloff 0 |
| `ground_hugging` | Presetに値がなければheight falloff 0.10 `m^-1` |
| `elevated_layer` | Global Height Fogへ近似せずLocal Fog Volume候補 |
| `local_only` | Global density 0、Local Fog Volume必須 |
| shaft `none／subtle／pronounced` | anisotropy 0.2／0.5／0.7。`pronounced`はVolumetric Fog必須 |
| color `neutral／warm／cool` | fixed linear RGB `(0.9,0.9,0.9)`／`(0.95,0.88,0.75)`／`(0.78,0.86,0.95)` |
| `inherit_sky` | `albedo_source=inherit_sky` |
| `palette_token` | 登録済みPalette token必須 |
| cloud `none／thin／scattered／overcast／storm_like` | coverage 0／0.2／0.5／0.9／1.0 |

Reference Presetは次のversioned constantsとResolver ruleであり、Source copyではない。

```text
EnvironmentPresetCatalogV1
  catalog_id: StableId
  catalog_version: positive u32
  entries[1..256]: EnvironmentPresetEntryV1
  catalog_content_hash: SHA-256

EnvironmentPresetEntryV1
  preset_id: StableId
  preset_version: positive u32
  preset_content_hash: SHA-256
  section_replacements[1..11]: EnvironmentSectionReplacementV1
  required_capability_refs[0..32]
  reference_cloud_assets_ref: nullable<ReferenceCloudAssetsRefV1>
```

Catalog entriesは`preset_id` byte順へstrict sortし重複を拒否し、全entryに`semantic_intent` replacementをexact一件要求する。`preset_content_hash`はASCII `MIRAKAN_ENVIRONMENT_PRESET_ENTRY_V1`と自己hashを除く全Field、Catalog hashはASCII `MIRAKAN_ENVIRONMENT_PRESET_CATALOG_V1`と自己hashを除く全Fieldのlength-framed canonical bytesをSHA-256する。Preset適用はentryが列挙する完全sectionだけを置換して他sectionを維持する。表の八件は`preset_version=1`のbuilt-in entryであり、表の定数、完全なtyped section replacement、Capability、Reference Asset refがFixture Catalog artifactで一致するまでActivationしない。

| Preset | Intent | Fog source | Capability |
|---|---|---|---|
| `fixture.environment.clear-day` | clear、uniform、shaftなし | visibility 20,000 m、falloff 0 | Environment core |
| `fixture.environment.temperate-morning-mist` | light_mist、ground_hugging、subtle | visibility 800 m、falloff 0.12、base 1 m | Environment core |
| `fixture.environment.humid-distance-haze` | light_haze、uniform、subtle | visibility 2,000 m、falloff 0.015 | Environment core |
| `fixture.environment.dense-ground-fog` | dense_fog、ground_hugging、subtle | visibility 50 m、falloff 0.25、base 1 m | Environment core |
| `fixture.environment.interior-dust-shafts` | indoor、local_only、pronounced | global density 0、`dust_shafts` Local Volume | Volumetric Fog／Local Volume |
| `fixture.environment.overcast-volumetric` | fog、overcast、subtle | volumetric Fog＋coverage 0.9 | Volumetric Fog／Cloud |
| `fixture.environment.stylized-tinted-fog` | stylized、palette_token | Profile値＋必須Palette token | Environment core／advanced |
| `fixture.environment.reference-earth-atmosphere` | outdoor、inherit time | `ReferenceEarthV1` atmosphere | Atmosphere LUT |

Preset適用は既存human lockを維持する。`EnvironmentSectionOverrideV1`はlockされていないPreset replacement branchにだけ許可し、complete type、closed enum／range／Capability／Target／fallbackを通常のValidatorで再検査する。lock済みfieldの上書き、Preset replacement外branchの暗黙補完、unknown override、`fixture.environment.stylized-tinted-fog`のexact Palette token ref欠落、`fixture.environment.reference-earth-atmosphere`へのcustom係数混在を拒否する。明示Target visibilityは完全なFog overrideにできるがPreset定数自体を変更しない。

`fixture.environment.interior-dust-shafts`は選択中room／region boundsをbox shapeへ使い、density multiplier 0.015、albedo `(0.78,0.72,0.62)`、anisotropy 0.7、blend distanceを最短半径の10%とする。bounded regionを一意に取得できなければshapeを推測せずBlocking質問へ停止する。Built-in Cloud Presetはexact `ReferenceCloudAssetsRefV1`一式を参照し、Project Assetが明示指定された場合だけ全件置換する。部分置換とnoise texel生成を禁止する。

`storm_like`はCloud intentであり雨、雷、風、Gameplay weatherを暗黙生成しない。`gentle／windy／inherit_weather`はexact Weather bindingがある場合だけ風向／速度を解決し、bindingなしで`still`以外ならBlocking質問を一つ返す。`time_of_day`はexact `TimeOfDayPresentationBindingV1`とfresh `PresentationTimeSampleV1`がある場合だけ、そのSourceとsun／moon Lightへ渡す。binding／sampleなし、ref不一致、sample sequence巻戻り、Skyとbindingのsun／moon集合差、Lighting Resolver unavailableのいずれかで`inherit`以外なら、登録済みPreset提示またはBlocking質問を一つ返し、Environment fieldだけを部分変更しない。

`EnvironmentIntentResolverV1`は次の純粋関数契約に固定する。

```text
resolve(
  EnvironmentIntentV1,
  EnvironmentProfileV1,
  EnvironmentWorldBindingV1,
  EnvironmentPresetCatalogV1,
  VisualStyleProfileRefV1 | null,
  WeatherPresentationSnapshotRefV1 | null,
  PresentationTimeSampleV1 | null,
  TargetCapabilitySnapshotV1,
  EnvironmentBudgetEnvelopeV1,
  PolicySnapshotV1,
  ResolvedLightPlanRefV1 | null
) -> EnvironmentIntentCandidateV1[0..3] | EnvironmentDiagnosticSetV1
```

入力Profile、Style、Weather、Time、Target、Budget、Policy、companion Planは同じbase Project revision／World／Target closureへ解決する。Budget envelopeがmissing／stale／別Worldならcostを推測せずDiagnosticへ停止する。

```text
EnvironmentIntentCandidateV1
  candidate_id: StableId
  base_project_revision: positive u64
  world_binding_ref: exact EnvironmentWorldBindingRefV1
  target_profile_refs[1..64]
  preset_ref: nullable<EnvironmentPresetRefV1>
  section_overrides[0..11]: EnvironmentSectionOverrideV1
  required_capability_refs[0..32]
  estimated_cost: EnvironmentCostEstimateV1
  assumptions[0..32]
  blocking_questions[0..7]
  confidence_q16: u16
  companion_lighting_plan_ref: nullable<ResolvedLightPlanRefV1>
  expires_at
  contract_hash: SHA-256

EnvironmentCostEstimateV1
  budget_envelope_ref: exact EnvironmentBudgetEnvelopeRefV1
  estimated_gpu_p95_us: nonnegative u64
  estimated_persistent_bytes: nonnegative u64
  estimated_transient_peak_bytes: nonnegative u64
  artifact_rebuild_units: nonnegative u32
  confidence_q16: u16
  estimate_content_hash: SHA-256
```

このtyped candidateを`EnvironmentChangeSetProposalV1`へ投影する。`companion_lighting_plan_ref`はTime-of-Day candidateがsun／moon Source変更を必要とする時だけ必須で、[Lighting](lighting.md)のResolver出力を参照するだけとしLight patchを複写しない。Planのbase Project revision、World、Target、expiryはcandidateとbyte equality、`affected_light_ids[]`はbindingが指定するnon-null sun／moonのexact集合とset equalityでなければならない。Resolver順はlock／Target／minimum quality除外、explicit intent match、Style／Weather／Time-of-Day binding、minimum capability、Preset ID byte tie-break。最大3候補で、Gameplay visibility混同、lock conflict、Targetなし、必要なcompanion Plan欠落は質問へ停止する。

`operation.environment.inspect_profile`はauthored Sourceだけを返し、実効値を装わない。`operation.environment.explain_effective`は次のread-only projectionを返す。

```text
EnvironmentEffectiveStateV1
  base_project_revision: positive u64
  world_binding_ref: exact EnvironmentWorldBindingRefV1
  environment_profile_ref: exact EnvironmentProfileRefV1
  visual_style_profile_ref: nullable<exact VisualStyleProfileRefV1>
  weather_snapshot_ref: nullable<exact WeatherPresentationSnapshotRefV1>
  presentation_time_sample_ref: nullable<exact PresentationTimeSampleRefV1>
  target_profile_refs[1..64]
  budget_envelope_ref: exact EnvironmentBudgetEnvelopeRefV1
  sky: SkySourceV1
  atmosphere: AtmosphereSourceV1
  fog: FogSourceV1
  local_fog_volumes[0..128]: LocalFogVolumeSourceV1
  cloud: CloudSourceV1
  lighting: EnvironmentLightingSourceV1
  selected_fallback_refs[0..32]
  field_origins[1..512]: EnvironmentEffectiveOriginV1
  companion_lighting_plan_ref: nullable<ResolvedLightPlanRefV1>
  diagnostics
  effective_state_content_hash: SHA-256

EnvironmentEffectiveOriginV1
  field_id: stable u32
  source_kind:
    authored_source | applied_preset_provenance | visual_style | weather |
    presentation_time | target | budget | fallback | companion_lighting
  exact_origin_ref: closed source-kind-matched ref

EnvironmentPlanExplanationV1
  effective_state_content_hash: SHA-256
  decision_records[1..512]:
    {field_id, authored_value_hash, effective_value_hash, origin_ref, rule_id, disposition: accepted | rejected, reason_code}
  omitted_candidate_reasons[0..64]
  blocking_questions[0..7]
```

`EnvironmentEffectiveOriginV1.exact_origin_ref`は`authored_source -> EnvironmentProfileRefV1`、`applied_preset_provenance -> EnvironmentPresetRefV1`、`visual_style -> VisualStyleProfileRefV1`、`weather -> WeatherPresentationSnapshotRefV1`、`presentation_time -> PresentationTimeSampleRefV1`、`target -> exact Target Profile ref`、`budget -> EnvironmentBudgetEnvelopeRefV1`、`fallback -> exact Product fallback Registry ref`、`companion_lighting -> ResolvedLightPlanRefV1`のclosed対応とし、kindとref branch不一致を拒否する。`field_origins[]`と`decision_records[]`は`field_id` byte順、fallback refsはfallback ID順へstrict sortし重複を拒否する。Source、Preset、Weather、Style、Time、Target、Budget、fallback、companion Lightingのどれが値を決めたかを一件ずつ示し、screen label、現在Camera、Editor preview、Renderer native stateを根拠にしない。露出はこのprojectionに含めず、Post Process Ownerへのexact navigation relationだけをUIが別表示できる。

```text
EnvironmentPreviewReceiptV1
  receipt_id: StableId
  receipt_content_hash: SHA-256
  contract_set_hash
  base_project_revision
  canonical_changeset_hash
  target_profile_ref:
    exact TargetProfileRefV1
  budget_envelope_ref: exact EnvironmentBudgetEnvelopeRefV1
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

EnvironmentPreviewReceiptRefV1
  receipt_id: StableId
  receipt_content_hash: SHA-256
  base_project_revision: positive u64
  canonical_changeset_hash: SHA-256
  target_profile_ref:
    exact TargetProfileRefV1
  expires_at
```

Receipt hashはASCII `MIRAKAN_ENVIRONMENT_PREVIEW_RECEIPT_V1`と自己hashを除く全Fieldのlength-framed canonical bytesからSHA-256する。Refはreceipt identity／hash、base revision、ChangeSet hash、Target ref、expiryを解決先とbyte equalityにする。Ground／elevated／representative gameplay cameraを最低1件使う。Environment／Water／Snowを別Preview lifecycleにせず、同じroot revision、Target、fallback summaryへ結ぶ。

Validator固定順はSchema／enum／collection／finite／unit、ID／revision／Asset ref、domain invariant、lock／base revision、Capability／Target／fallback、domain cost、Scene bounds／camera、cross-profile invariant、fresh Previewである。Custom atmosphere全係数、scene-wide sky／sun、visibility 100 m未満または4倍超、required volumetric、dynamic Snow、Gameplay visibility混同はBefore／After Previewを必須にする。Authorization／approval判定はGovernance ownerへ委譲する。

EditorのEnvironment Panelはexact `basic | detailed`の二表示modeだけを持つ。`basic`はIntent／Preset／World binding／visibility／time／cloud／cost／fallback／diagnostic、`detailed`は同じcanonical fieldにAtmosphere／Fog／Volume／IBLと`EnvironmentEffectiveStateV1`のorigin／reasonを追加する。表示modeはSourceへ保存せず、別の簡易値や既定値を持たない。Activation後だけ各actionを完全登録済みMCD Operationへ解決する。露出はPost Process Ownerへのread-only status／navigationだけで、Environment actionへ変換しない。Water body／flow／depth／wave／query、Weather、Snow receiver／mask／page／stampはSource projectionを表示できるが、別Owner familyが完全登録されるまではread-only Gap表示とし、future action labelからOperationを推測しない。AIはLUT、froxel、step、format、history、GPU texture、Particle buffer、Physics force／frictionを編集しない。

### 6.1 有名Engineとの設計比較

比較はUI名や内部object modelの互換目標ではなく、制作導線と責任分離の根拠に使う。

| 比較Engineの公開モデル | 採用する要点 | Miraikanaiでの差分 |
|---|---|---|
| [Unreal Engine 5.8 Environment Light Mixer](https://dev.epicgames.com/documentation/en-us/unreal-engine/environment-light-mixer-in-unreal-engine?application_version=5.8) | Sky Light、最大二つのAtmosphere Directional Light、Sky Atmosphere、Volumetric Cloudを一つのdockable windowへ集約し、Minimal／Normal／Normal+Advancedでproperty表示量を切り替える | Environment Panelへ集約し詳細度を変えても同じcanonical field refを維持する。一般Lightは[UE 5.8 Light Mixer](https://dev.epicgames.com/documentation/en-us/unreal-engine/using-the-light-mixer-in-unreal-engine?application_version=5.8)と同様にLighting Ownerへ分離し、Time-of-Dayだけexact companion Planで連携する |
| [Unity 6 Lighting window](https://docs.unity3d.com/6000.0/Documentation/Manual/lighting-window.html)、[HDRP 17.0 Visual Environment](https://docs.unity3d.com/Packages/com.unity.render-pipelines.high-definition@17.0/manual/Override-Visual-Environment.html)、[Volume](https://docs.unity3d.com/Packages/com.unity.render-pipelines.high-definition@17.0/manual/volume-component.html) | Scene単位のEnvironment lightingと、Profile参照によるglobal／local Volume、weight／priority／blendを区別する | World bindingとProfile refをSourceで明示し、Local Fogだけをbounded typed volumeにする。全Environment fieldをgeneric override stackへせず、同priority時の任意評価順やscene object列挙へ正本性を依存させない |
| [Godot 4.7 WorldEnvironment](https://docs.godotengine.org/en/4.7/classes/class_worldenvironment.html)と[Environment／post-processing](https://docs.godotengine.org/en/4.7/tutorials/3d/environment_and_post_processing.html) | SceneのEnvironment resource、Camera override、Editor preview sun／skyを別優先度・別用途として扱い、露出／被写界深度はEnvironmentからCameraAttributesへ分離する | WorldはEnvironment bindingを最大一件だけ持ち、Camera／Post Processは各Ownerのexact overrideに分離する。露出をEnvironmentへ複写せず、Editor previewをRuntime EnvironmentやSource既定値へ昇格しない |

共通して有効なのは「環境関連設定の一箇所集約」「Profile／resource参照」「Scene／World scope」「詳細度切替」「光源との明示連携」である。本設計はさらに、AI向けにStable ID／revision／hash、typed intent、bounded candidate、read-only projection、Operation→Change primitive coverage、fallback ID、Source／Derived／Runtime境界を必須にする。これによりAIは画面label、Actor／Node名、配列順、現在Camera、Editor previewから編集対象や権限を推測しない。

## 7. Domain budgetとfallback

Environment tier semanticsはLow／Core＝precomputed sky＋height fog＋2D cloud＋baked diffuse／specular IBL、Medium＝atmosphere LUT＋160×90×64 fog＋1/4 cloud、High＝dynamic IBL＋240×135×96 fog＋1/2 cloud／shadowである。Environment persistent＋history＋exclusive transient peakはLow 32 MiB、Medium 128 MiB、High 256 MiB、Medium domain GPU P95 soft ceiling 2.00 ms。これらは[Runtime performance／capacity](../04-runtime/performance-capacity.md)が公開する同revision／Target／World／View Familyの`EnvironmentBudgetEnvelopeV1`内数であり、Resolver、Preview、QualificationはそのrevisionをReceiptへ記録する。Alias後physical peakとlogical内訳を共に記録し、未証明aliasを0 byte扱いしない。

Environment fallback順はcloud-shadow update frequency、light steps、primary steps／scale、Fog XY→Z、local light count、dynamic→baked IBL、volumetric→2D cloud、volumetric→height fog、atmosphere LUT→precomputed cubemap。同じPalette、sun／moon meaning、target visibility、Gameplay ruleを維持し、minimum tier未満はsilent fallbackしない。

Product Planのexact fallback IDとの対応は次に固定する。`fallback.rendering.ibl-baked`は`dynamic_incremental -> baked`と同じdiffuse／specular有効意味、Palette、sun／moon linkを維持し、`diagnostic.rendering.ibl-baked-selected`を発行する。`fallback.rendering.environment-core`はadvanced要素をLow／Coreのprecomputed sky＋height fog＋2D cloud＋baked IBLへ落とし、Palette、sun／moon meaning、target visibility、Gameplay ruleを維持して`diagnostic.rendering.environment-fallback-selected`を発行する。`capability.environment.core`、`height_fog`、`ibl_baked`はCore構成要素であり相互fallbackせず、いずれかが不成立なら`fallback.capability.unavailable`でfail closedにする。必要Asset不在、target visibilityまたはminimum tierを維持不能、未登録中間variantの場合は`preserves_semantics=true`を主張せずfallback全体を拒否する。別順序、循環fallback、silent fallback、Product fallback IDなしの成功扱いを禁止する。

| Water ceiling | Baseline desktop | Advanced desktop | Advanced mobile standard |
|---|---:|---:|---:|
| active body／visible surface | 32／8 | 256／32 | 64／8 |
| GPU persistent／transient | 16／16 MiB | 96／64 MiB | 32／24 MiB |
| CPU query artifact | 4 MiB | 32 MiB | 8 MiB |
| GPU P95 soft ceiling | 0.35 ms | 1.00 ms | 0.75 ms |

Water visibility overflowは`semantic_priority desc, projected_coverage_px_q16 desc, body_id asc`末尾のPresentationだけを除外し、Volume／queryを維持する。Fallback順はsurface patch、wave component、normal octave、SSR、foam、underwater sampleで、Body boundary、water level、flow、query、Visual Styleを変えない。

| Snow ceiling | Baseline desktop | Advanced desktop | Advanced mobile standard |
|---|---:|---:|---:|
| receiver | 1,024 | 8,192 | 2,048 |
| field persistent／transient | 0 | 24／8 MiB | 6／4 MiB |
| update GPU P95 | 0 | 0.35 ms | 0.25 ms |
| stamp／Simulation Advance | VFX／Decal owner | 1,024 | 256 |

Snow fallback順はfootprint detail、field update distance、sparkle、normal detail。Gameplay Surface ID、static coverage、Visual Styleを維持し、missing／nonresident dynamic pageはexplicit static maskへ戻す。Precipitation particleはVFX domainへ一度だけchargeし、その密度fallbackは`VfxLodProfileV1`が所有する。すべてのdomain ceilingはRuntime ownerのaggregate envelopeを免除しない。

## 8. Diagnostic、failure、qualification

Environment closed Diagnosticは`EnvironmentPresetNotFound, EnvironmentPresetRefStale, EnvironmentIntentAmbiguous, EnvironmentIntentUnsupported, EnvironmentInvalidUnit, EnvironmentInvalidRange, EnvironmentFogVisibilityConflict, EnvironmentFogFarBeforeNear, EnvironmentAtmosphereIncomplete, EnvironmentAtmosphereRadiusInvalid, EnvironmentLocalFogLimitExceeded, EnvironmentCloudLayerInvalid, EnvironmentCloudAssetMissing, EnvironmentCapabilityUnavailable, EnvironmentTargetFallbackMissing, EnvironmentBudgetEnvelopeMissing, EnvironmentBudgetEnvelopeStale, EnvironmentBudgetExceeded, EnvironmentDerivedFieldWriteForbidden, EnvironmentHistoryPolicyLocked, EnvironmentHumanLockConflict, EnvironmentPreviewRequired, EnvironmentPreviewStale, EnvironmentWeatherBindingStale, EnvironmentStyleBindingStale, EnvironmentStyleConflict, EnvironmentTimeSourceMissing, EnvironmentTimeSourceStale, EnvironmentTimeSampleOutOfOrder, EnvironmentSunMoonLinkMismatch, EnvironmentCompanionLightingPlanMissing, EnvironmentCompanionLightingPlanStale, EnvironmentCompanionLightingScopeMismatch, EnvironmentGenerationMismatch, EnvironmentIblPromotionIncomplete`。

Waterは`WaterInvalidBoundary, WaterBodyOverlapConflict, WaterUnsupportedBodyKind, WaterWaveOutOfRange, WaterDepthCookFailed, WaterQueryQueueFull, WaterQueryStaleSnapshot, WaterCpuGpuMismatch, WaterRenderBudgetExceeded, WaterResourceGenerationMismatch, WaterFallbackMissing`。Weather／Snowは`WeatherValueOutOfRange, SnowReceiverInvalid, SnowMaskMissingUv, SnowDynamicCapabilityMissing, SnowPageBudgetExceeded, SnowStampOverflow, SnowFieldGenerationMismatch, SnowMaterialInterfaceMismatch, SnowFallbackMissing, SnowRenderBudgetExceeded`である。

Validation、Compile、Bake failureではlive root revisionとlive generationを変えず、partial artifactを公開しない。stale bindingはlast valid snapshot＋Diagnostic、budget overflowは規定fallbackかreject、device lossはSource／receiptから再生成する。

Contract qualificationは全Source／operationのvalid／boundary／invalid／unknown／NaN／unit、lock／stale／Capability／fallback／partial custom body、resolver candidate順／ChangeSet／artifact hash determinismを含む。8 Reference Presetはそれぞれ同一Inputから同じcandidate順、closed section override、ChangeSet hashと上表のexact constantsを得るfixtureを持ち、lock競合、unknown／duplicate override、Palette token欠落、bounds不明、Cloud Asset部分置換をnegative fixtureとする。Time-of-Day fixtureはSource ref／sample／named phase、Sky link、Lighting Plan／ChangeSet proposalの同一revision／World／Target／expiryとsun／moon exact set equality、atomic Commit、片側失敗時Source不変を検証する。Effective-state fixtureはSource、Preset、Weather、Style、Time、fallbackごとのorigin／reasonと同一入力hash determinismを検証し、露出field混入をnegative fixtureにする。Environment fixturesはclear day、morning mist、dense fog、local dust、sunset atmosphere、overcast cloud、camera cut／extent／origin rebase／sun／weather history、Water／Transparent／Pixel／VFX／UI compositeである。

Water fixturesはbounded pool、river-like mesh、2D tile、lake／river／ocean overlap、flat Volume enter／exit、analytic CPU／GPU tolerance、buoyancy frame-rate independence、probe／SSR／underwater／waterline／flow／depth／foam、LOD invariance、device recoveryを含む。Query fixtureは全7 status、16,384／4,096境界、QueueFull非fallback、stale generationを検査する。Save／load fixtureはBody Asset revision、Gameplay water state、解析波開始advanceから同じqueryを復元し、GPU texture／foam／SSR historyがpayloadにないことを検査する。Interaction fixtureは各closed kindとinvalid field、同一Simulation Advance内のcanonical order、Replay後の7 field／VFX seed一致、VFX／Audio独立配送、Gameplay逆入力0を検査する。Snow fixturesは2D／3D precipitation、Style variants、authoritative contact、exact `SimulationAdvanceIntervalV1`列を固定したfield hash、duration-null時のtime pass停止、page／stamp order、Save canonical omission／load reset、complete／incomplete visual Replay判定、device recovery、Gameplay friction invariance、LOD／static fallbackである。

AI evalはmorning mist、50 m fog、local cave dust、stylized planet request、Gameplay visibility separation、mobile over-budget cloud、ambiguous cinematic request、Water capability mismatch、dynamic Snow costを含み、正しいoperation、質問、lock、fallback、Diagnostic、説明とChangeSet一致を測る。Qualification receiptのTarget、run、Evidence envelope、gradingはAI Verification ownerへ委譲する。
