# Miraikanai Engine Lighting Contract

- 文書ID: mirakan.arch.rendering-lighting
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Light Source／Component、light type／shape、photometric quantity／unit／color、attenuation／range、shadow intent、Lighting semantic intent／resolver、Lighting固有operation／diagnostic／qualification
- 非正本範囲: Render pass／cluster／queue／shadow execution、Material shading、Environment composition、Runtime shared capacity、Tool version、AI authorization、Evidence envelope、共通Schema／projection。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Render Graph](render-graph.md)、[Materials](materials.md)
- 関連文書: [Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Math／Core utilities](../02-foundation/math-core.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Render Graph](render-graph.md)、[Materials](materials.md)、[Post Processing](post-processing.md)、[Environment／Surfaces](environment-surfaces.md)、[World](world.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-21

## 1. 結論と所有境界

LightingはLightの物理意味とauthoring intentをEngine-owned contractに固定する。人間とAIはlight type、shape、photometric quantity、color、attenuation、range、shadow intentを編集し、Renderer Backend、cluster layout、shadow atlas、pass、descriptorを指定しない。

[Render Graph](render-graph.md)は解決済みLight setのselection、cluster／tile assignment、shadow／lighting passとqueue executionを所有する。[Materials](materials.md)はsurface responseを所有する。本書はEnvironmentやCameraのsource modelを再定義せず、revision付き`EnvironmentLightingSummaryV1`／exposure contextをtyped inputとして消費する。

共通Source revision、ChangeSet、operation envelope、projection、approval、Evidenceは[Project state](../03-authoring/project-state.md)、[Executable contracts](../02-foundation/executable-contracts.md)、Governance文書を参照し、本書でfieldやruleを複写しない。

## 2. Module境界

ModuleはLighting Contracts、Light Source Authoring、Photometric Validation、Intent Resolver、Runtime Light Snapshot Bridge、Editor／AI Projection、Lighting Qualificationに分ける。Runtime BridgeはWorld sourceを直接編集せず、published component revisionからimmutable `ResolvedLightSet`を生成する。`ResolvedLightSet`の唯一のRenderer公開形は§6の`LightSnapshotV1`であり、別名のRenderer入力objectを設けない。

Rendererとの公開境界にはBackend-neutral light data、stable source identity、generation、view eligibility hint、shadow intentだけを渡す。native light object、shadow resource、cluster index、GPU addressはprivate execution stateである。

## 3. 正本データモデル

`LightSourceV1`を次に固定する。

| Field | 型／規約 |
|---|---|
| `light_id` | 永続Stable ID。Entity／GPU indexを使わない |
| `revision` | 楽観的排他とChangeSetのbase revision |
| `compatible_world_space` | `WorldSpaceCompatibilityV1`。Light Sourceはstructural dimensionを所有しない |
| `light_type` | `directional \| point \| spot \| rectangle \| disk` |
| `mobility` | `static \| stationary \| dynamic` |
| `enabled` | 明示bool。強度0を無効化表現にしない |
| `transform_ref` | WorldのStable Transform ID。Directionalは位置を意味に使わない |
| `radiometry` | §4のtagged union |
| `color_source` | §4のtagged union |
| `range_m` | 有限局所光の影響半径。Directionalでは禁止 |
| `spot_shape` | Spotだけのinner／outer angle |
| `source_radius_m` | Point／Spotの有限Emitter半径 |
| `emitter_shape` | Rectangle／Diskの物理寸法 |
| `channel_mask` | 32-bit論理Lighting Channel。予約bitを検証 |
| `role` | `key`等の意味role。描画計算へ直接使わない |
| `group_ids` | Semantic group Stable ID、最大8 |
| `importance` | `critical \| important \| decorative` |
| `priority_u8` | 同一importance内0～255 |
| `shadow_intent_ref` | version付きShadow Intent、nullable |
| `cookie_asset_id` | 検証済みTexture artifact、nullable |
| `ies_asset_id` | 検証済みIES artifact、nullable、optional capability |
| `target_policy_ref` | Target別上限／fallback policy |
| `human_lock_mask` | AIが変更できないfield集合 |

`LightComponent`はexact `WorldSpaceProfileRefV1`を持つWorldにだけattachでき、`LightSourceV1.compatible_world_space`とWorld Profileの一致を検証する。portable profileではWorld Profileから導く2Dで`directional | point`、3Dで`directional | point | spot`を許可する。2D spot／area／IESと3D rectangle／disk／IESはoptional capabilityとして個別qualificationし、activation／maturityは[Product Plan](../00-product/product-plan.md)が決定する。World Profile、Light compatibility、type／shapeの不正組合せをSchemaで拒否し、Light Sourceの既定やasset名からdimensionを推測しない。

`LightComponent`はWorld entityにSource Definition refとinstance-local overrideを結び、Source Assetを複製しない。override可能fieldはSourceが宣言し、instanceからlight typeやunit semanticsを変更しない。

### 3.1 `LightIntentV1`

```text
LightIntentV1
  intent_id
  scope
  role
  subject_group_ids[]
  mood
  contrast
  coverage
  direction_intent
  softness_intent
  color_intent
  mobility_intent
  importance
  target_selector
  quality_selector
  fallback_priority[]
  constraints
  base_revision
```

閉じた語彙は`role: key | fill | rim | environment | practical | accent | effect | decorative`、`mood: neutral | warm | cool | dramatic | soft | high_key | low_key | toon_banded | pixel_crisp | custom_profile`、`contrast: flat | balanced | strong`、`coverage: subject | local | room | outdoor | world`、`direction_intent: camera_left | camera_right | front | back | above | below | world_direction | match_reference`、`softness_intent: hard | balanced | soft`、`mobility_intent: prefer_static | allow_stationary | require_dynamic`とする。

`constraints`はhuman lock保持、追加光源最大数、変更可能group、Shadow非増加、Gameplay-critical material可読性等のtyped constraintだけを実行条件にできる。自由文は説明に限る。

### 3.2 `LightingStyleProfileV1`

Profile ID／version／parent、Default role recipe、`VisualStyleProfile` ref／整合constraint、Targetごとの許可light type／shadow tier、Exposure接続、2D normal lighting／Pixel Art quantization方針、color temperature／saturation／contrast range、importance別fallback priority、Preview fixture／Qualification refを持つ。Toonを使う場合はVisual Styleが指すexact `ToonShadingProfileRefV1`をinputとして消費するだけで、band、ramp、specular、rim、shadow、energy parameterを複写・上書きしない。継承は最大4段、cycle禁止、各fieldは`inherit | replace`を明示し、配列を暗黙appendしない。

### 3.3 `ResolvedLightPlanV1`

Resolver出力は`base_project_revision`、`world_id`、exact Target Profile refs、Intent／Profile／Catalog／Target Capabilityのversion／hash、exact `WorldSpaceProfileRefV1`、適用時はexact `ToonShadingProfileRefV1`、追加／更新／削除する`LightSourceV1` exact patchとその`affected_light_ids[]`、解決したphysical quantity／color／type／range／mobility／channel／importance／priority、Shadow Intent ref、selection／cluster上限のworst-case proof、予測CPU／GPU時間／persistent・transient byte、保持lock／未充足constraint、採用／棄却／理由／fallback chain、必要Asset cook／Preview／approval／Qualification、`plan_hash`／expiryを持つ。Toon responseはPlanにparameter展開せず、profile refとそのTarget-qualified fixture結果だけを記録する。

```text
ResolvedLightPlanRefV1
  plan_hash: SHA-256
  base_project_revision: positive u64
  world_id: StableId
  target_profile_refs[1..64]
  affected_light_ids[1..4096]: StableId
  expires_at
```

`ResolvedLightPlanRefV1`は一つの保存済みread-only `ResolvedLightPlanV1`へexact解決し、全fieldを解決先とbyte equalityにする。`affected_light_ids[]`はStable ID byte順、Target refsはProfile ID／version／content hash順へstrict sortし、duplicateを拒否する。[Environment／Surfaces](environment-surfaces.md)のTime-of-Day candidateがsun／moon変更を伴う場合、このrefだけをcompanion planとして保持し、Light patch、物理量またはfallbackをEnvironment側へ複写しない。

PlanはProjectを変更しない。ChangeSet Commit後だけSourceを更新し、Catalog、Target Profile、base revisionのいずれかが変われば`MIRAKAN-LIGHTING-STALE-PLAN`として再解決する。approval mechanicsは[AI Security／Approval](../01-governance/ai-security-approval.md)を参照する。

### 3.4 Shadow authoring

LightingはSource側の`ShadowIntentV1`と`ShadowStyleProfileV1`を所有し、Render GraphはDerived側の`ShadowGraphV1`と`ResolvedShadowPlanV1`を所有する。Source objectへpass、queue、resource、descriptor、native techniqueを保存しない。

```text
ShadowIntentV1
  shadow_intent_id, revision, source_light_refs[]
  coverage: subject | local | view | world
  softness: hard | balanced | soft
  quality_intent: disabled | contact | standard | high | ray_qualified | project_technique
  project_technique_ref: optional ProjectShadowTechniqueV1 ref
  target_profile_refs[], fallback_priority[], base_project_revision

ShadowStyleProfileV1
  shadow_style_profile_id, revision, visual_style_profile_ref
  allowed_quality_intents[], default_quality_by_importance{}
  capability_requirement_refs[], budget_ref, fallback_refs[]
  preview_fixture_refs[], qualification_policy_ref
```

`quality_intent=project_technique`だけが`project_technique_ref`を必須とし、他値では禁止する。`ray_qualified`と`project_technique`はTarget Capabilityとfresh Qualification Receiptを必須とする。Source Intent／Style ProfileはReceipt-freeで、Receiptは先に固定したTechnique／Target artifact closureをsubjectにするroot外Activation Bindingからだけ解決し、Intent／Profile hashまたはReceipt subjectへResolved Plan hashを戻さない。`fallback_priority[]`は同Profileの`fallback_refs[]`だけを重複なしで順序付け、未登録fallback、空の必須fallback、Renderer都合のsilent downgradeを拒否する。ResolverはSource refs、Profile、Target、Capability、budget、Qualificationを同一revisionで固定し、Render Graphへexact refだけを渡す。

### 3.5 Production Lighting Bake／Probe

Lightmap、irradiance probe、reflection probeはすべてDerived Assetであり、Source truthではない。AIは`BakeLighting`を要求できるが、bake texelまたはprobe payloadを直接生成・編集しない。Bake／Probeの唯一のowner schemaを次に固定する。

```text
LightingBakeProfileV1
  profile_id
  target_profile_ref
  lightmap_texel_density_per_m
  max_lightmap_dimension
  max_atlas_count
  irradiance_probe_spacing_m
  reflection_probe_resolution
  compression_profile_ref
  bake_budget_ref

LightingBakeArtifactV1
  source_revision
  geometry_hash
  material_hash
  light_hash
  environment_hash
  bake_profile_hash
  lightmap_atlas_refs[]
  lightmap_binding_refs[]
  irradiance_probe_volume_refs[]
  reflection_probe_artifact_refs[]
  validation_report_ref
  artifact_hash

LightmapBindingV1
  mesh_submesh_stable_id
  lightmap_uv_set
  lightmap_atlas_ref: exact ArtifactRefV1
  atlas_rect
  decode_scale_bias

IrradianceProbeVolumeV1
  bounds
  spacing_m
  priority
  blend_distance_m
  validity_mask

ReflectionProbeDefinitionV1
  bounds_shape: sphere | box
  bounds
  capture_origin
  priority
  blend_distance_m
  receiver_channel_mask
```

Static Meshのlightmap UVは専用UV set、0～1範囲、finite、triangle overlap許容率0%、Cook後解像度で4 texel以上のchart paddingを必須とする。C2 reference値は32 texel／m、atlas最大4,096²、1 Level最大64 atlas、irradiance probe間隔0.5～16 m、reflection cubemap 128²×6である。Source Meshが満たさない場合、Importerは生成候補とLoss Reportを提示できるが、既存UVを無断置換しない。

各`LightmapBindingV1.lightmap_atlas_ref`は親`LightingBakeArtifactV1`の`lightmap_atlas_refs[]`への所属をexact Artifact identityで検証し、display name、path、配列indexからatlasを推測しない。`atlas_rect`はfiniteなnormalized `{u_min, v_min, u_max, v_max}`で、全成分を0～1、`u_min < u_max`、`v_min < v_max`とし、参照atlasのextent外または面積0を拒否する。`lightmap_binding_refs[]`内で同じbinding refの反復、または同じ`mesh_submesh_stable_id`を複数回束縛する`duplicate binding`を`MIRAKAN-LIGHTING-BAKE_ARTIFACT_INVALID`としてArtifact promotion前に拒否する。

Probe重複はpriority、volume、Stable IDの順に決定する。未配置領域はC1 Environment IBLへfallbackし、local reflection probeをC1 global IBLと混同しない。geometry、Materialのbaked contribution、static／stationary Light、Environment、Bake Profile、baker toolchain version、quality settingsからSource dependency hashをcanonicalに構成し、そのいずれかのhash変更時だけ該当World Cell Artifactをinvalidateする。

ArtifactはWorld Cell dependencyへ登録し、同一CellのLightmap、irradiance、reflectionをactivation group単位でall-or-nothingにresident化する。UV／probe／Cell dependency closureが不完全、artifactがmissing／stale／corrupt、またはreadback hash不一致なら`MIRAKAN-LIGHTING-BAKE_CLOSURE_INCOMPLETE`、`MIRAKAN-LIGHTING-BAKE_ARTIFACT_INVALID`、`MIRAKAN-LIGHTING-BAKE_CELL_ACTIVATION_FAILED`のtyped failureとし、黒Scene、旧Artifact継続、partial atlas／probe適用を禁止する。同じTargetでQualification済みのrealtime Light＋Environment IBLへ明示的にfallbackし、fallback不可ならCell activationを拒否する。

Bakeは同じSource dependency hash、Toolchain、Profile、Targetで同じ`artifact_hash`を得るdeterministic jobである。`validation_report_ref`はUV overlap、leak、seam、probe light leak、dynamic object blend、Cell境界、cold streaming、memory、bake timeを検査したVisual／performance receiptを参照する。fixed Exposureのoffline reference、realtime fallback、Target実機を比較し、Gameplay、Physics、Navigation、Save stateがBake Artifactの存在、residency、結果に依存しないことをQualificationする。

## 4. 物理単位と数値規約

`radiometry`は次のtagged unionのどれか一つだけを持つ。

| Light type | 正規入力 | 単位 |
|---|---|---|
| Directional | `illuminance_lux` | lux |
| Point | `luminous_flux_lm` | lumen |
| Spot | `luminous_flux_lm`またはqualified IES | lumen／candela distribution |
| Rectangle／Disk | `luminance_nit` | cd/m² |

単位なし`intensity`を正本にせず、Light type／shapeに対応するquantityを使う。Portable Pointのscalar referenceを次に固定する。

```text
delta_m = light_position - surface_position
distance_sq_m2 = dot(delta_m, delta_m)
distance_m = sqrt(distance_sq_m2)
d2 = max(distance_sq_m2, max(source_radius_m, 0.01 m)^2)
x  = saturate(distance_m / range_m)
w  = x < 1 ? (1 - x^4)^2 : 0
I_cd = luminous_flux_lm / (4 * pi)
E_lux = I_cd * w / d2
```

Spotは単位積分が1となる`SpotDistributionV1`をCook時に生成し、その角度分布へ総光束を配る。inner／outer angleを単純乗算して総光束を変化させない。CPU scalar reference、HLSL、MSL、SPIR-V経路は同一fixtureで相対誤差を検証する。`source_radius_m`はEmitter geometryまたはStyle ProfileからResolverが決め、AIがsingularity回避値を直接指定しない。

`color_source`は`linear_rgb`（scene-linear D65基準、各成分0以上）、`cct`（kelvin 1000～20000、tint -1～1）、`profile_default`（`LightingStyleProfileV1`のrole既定値）のtagged unionである。同じLightへRGBとCCTを同時保存しない。display transferやtone mappingは[Post Processing](post-processing.md)の責務でありLight colorへ焼き込まない。

Geometry制約を次に固定する。

- `range_m`は0より大きくTarget上限以下。
- Point／Spotの`source_radius_m`は有限な0以上かつ`range_m`以下。
- Spotは`0 <= inner_angle_deg < outer_angle_deg < 179`。
- Rectangle／Diskは0より大きい寸法を持つ。
- Directionalはrange、position attenuation、IESを持たない。
- Cookie／IESはAsset type、dimension、license、cook statusを検証する。
- NaN、Inf、denormal依存、負の放射量を拒否する。

違反は`MIRAKAN-LIGHTING-UNIT-MISMATCH`、`MIRAKAN-LIGHTING-COLOR-AMBIGUOUS`、`MIRAKAN-LIGHTING-GEOMETRY-INVALID`、`MIRAKAN-LIGHTING-ASSET-UNQUALIFIED`の該当Domain failureを返す。

座標、角度、距離、normalization、finite validationは[Math／Core utilities](../02-foundation/math-core.md)を使う。source unitをBackend unitへ変換する処理はprivate Adapterに閉じ、round-trip時に元のquantity／unitを保持する。

## 5. Light eligibilityとRenderer境界

Lightingはview layer、channel、mobility、importance、shadow intent、Target capability requirementを宣言する。どのLightを各Viewへ投入するか、cluster overflow時の実行順、shadow resource割当は[Render Graph](render-graph.md)が所有する。

RendererはLightの物理値をexecution上限へ収めるため黙ってclampしない。capacity不足時はSource identityと拒否／fallback理由を返し、事前宣言されたshadow disable、quality reduction、view exclusionだけを適用する。共通budget、reservation、backpressure、測定値は[Runtime performance／capacity](../04-runtime/performance-capacity.md)を参照する。

Material側はResolved Light semanticsをshading inputとして消費し、Light type、unit、attenuationを再解釈しない。Light linkingはstable layer／channel semanticsを使い、Material名、Entity名、Backend indexによるbindingを禁止する。

## 6. Lighting Intent Resolver

Lighting intentはsubject、mood／readability、key／fill／rim relation、contrast、time／environment context、physical constraint、Target scopeをtyped axesで表す。曖昧な「明るく」「映画的」等は、既存Light編集、Light追加、exposure／post-process変更の候補を区別して質問する。

ResolverはProject revision、selected World／Level scope、existing Light refs、Material／Environment context refs、Target Profile、assumption／question、candidate changes、compatibility resultを束ねたDomain resolutionを返す。共通hash、revision、disposition、projection fieldは[Executable contracts](../02-foundation/executable-contracts.md)の正本を使う。

同じ入力とCatalog revisionから同じcandidate orderを返し、Entity列挙順やviewportの一時状態へ依存しない。物理constraintとartistic intentが競合する場合はsilent conversionをせず、意味差を示した代替案を返す。

`LightIntentResolverV1`は次の純粋関数契約に固定する。

```text
resolve(
  LightIntentV1,
  LightingStyleProfileV1,
  VisualStyleProfile,
  WorldSpaceProfileRefV1,
  ToonShadingProfileV1 | null,
  EnvironmentLightingSummaryV1,
  MaterialReadabilitySummaryV1,
  SceneLightingSummaryV1,
  TargetCapabilitySnapshotV1,
  LightingBudgetEnvelopeV1,
  PolicySnapshotV1
) -> ResolvedLightPlanV1 | LightingDiagnosticSetV1
```

解決順は(1) Schema／Stable ID／base revision／権限、(2) exact World Profile／scope／subject／human lock、(3) Visual Style／Toon response／Environmentのrole recipe、(4) Target適合Light／Shadow／Assetの絞り込み、(5) 物理量／色／配置candidate生成、(6) readability／budget評価、(7) fallback chain、(8) Plan／reason／cost／risk／Preview差分の固定順とする。Toon Profileがnullである場合、Toon Shading Modelを使うMaterialを対象にした`toon_banded` candidateを生成しない。

Renderer入力の`LightSnapshotV1`は`generation`、`view_family_id`、`compact_light_ids[]`、`type_and_flags[]`、`position_and_range[]`、`direction_and_cone[]`、`color_and_radiometry[]`、`shape_parameters[]`、`channel_masks[]`、`shadow_plan_refs[]`、`source_revisions[]`を持つimmutableな論理SoAである。GPU packingはMCD生成`LightGpuRecordV1`とBackend Adapterの所有とし、Snapshotにnative handle／descriptor／GPU addressを含めない。compact indexはgeneration内だけ有効で、Save／Replay／DiagnosticはStable `light_id`を使う。

`LightSnapshotV1`の抽出・検証・Render Graph入力としての実行はProduct上の`capability.rendering.render-graph-core`に含む。一方、`ResolvedLightPlanV1`を作るIntent discovery／explain surfaceは`planning.operation_family.lighting_discovery`の別activation境界であり、render-graph-coreまたはEnvironment capabilityの有効化から推測して公開しない。

## 7. AI／Editor operation

create light、update physical property、apply lighting intent、bind Source Definition、set shadow intent、preview、explain、validateはStable IDでないDomain planned action vocabularyであり、それ自体はMCD Operationまたはcurrent callable surfaceではない。Lightingの完全IDは次段落の九reserved candidateだけで、current集合は空である。完全登録済みOperation identityはOwner Manifest／MCD／Service equalityでのみ成立し、action名から生成しない。Applyが将来Activationされた場合もGatewayを通じてProject ChangeSetへ変換し、Runtime componentやRenderer resourceを直接変更しない。

`operation.lighting.resolve_intent`／`operation.lighting.explain_plan`は`planning.operation_family.lighting_discovery`のreserved candidateであり、current canonical IDではない。九候補のcurrent MCD／Manifest／Service／Provider／MCP Tool／alias集合は空、Capability stateは`not_activated`である。`activation.lighting.discovery_operations.v1`がfamily全体をatomic activateする場合、それぞれ`ResolvedLightPlanV1`／`LightingPlanExplanationV1`を返し、Light Sourceへwriteしない。

Activation後のPreviewは対象revision、World／Level scope、affected Light Stable IDs、before／after physical values、Renderer compatibility、fallback、diagnosticを表示する。Explainは入力intent、unit conversion、採用candidate、assumption、未解決questionを返す。authorization classとhuman approvalは[AI Security／Approval](../01-governance/ai-security-approval.md)だけが決める。

Activation後の`SceneLightingSummaryV1`はScene／View Family／Target／Visual Style／EnvironmentのID／version、role／type／importance別Light数、Critical Light、上限／現在値／予測cost／overflow、Shadow tier分布、human lock数、active Diagnostic上位数、詳細取得用Stable ID／continuation tokenだけをboundedに返す。`LightingPlanExplanationV1`はLightごとのIntent fieldからSource fieldへの対応、代替案の棄却理由、予測cost、視覚risk、fallbackで失われるcueを返す。

`LightingChangeSetProposalV1`は[Executable contracts](../02-foundation/executable-contracts.md)のProposal envelopeにbase revision、typed Light差分、対象Stable IDs、risk、Preview hash、必要Approvalを載せるDomain projectionであり、直接Commitしない。`LightingDiagnosticSetV1`は共通Diagnostic envelopeに本書のclosed IDとLight property pathを載せる。`EnvironmentLightingSummaryV1`は[Environment／Surfaces](environment-surfaces.md)、`MaterialReadabilitySummaryV1`は[Materials](materials.md) §8が定義・公開するread-only／revisioned projectionである。`TargetCapabilitySnapshotV1`、`LightingBudgetEnvelopeV1`、`PolicySnapshotV1`もそれぞれのOwnerが公開する同種のprojectionであり、Lightingはいずれの内容も複写または書き戻さない。

## 8. Diagnosticとfailure

Lighting固有diagnosticはLight Stable ID、property path、type／shape／unit、Target、error code、remediationを含む。少なくともinvalid quantity、unit mismatch、invalid color、degenerate shape、unsupported light type、shadow intent unsupported、stale source、Renderer rejectionを区別する。

| Closed ID | 意味 |
|---|---|
| `MIRAKAN-LIGHTING-SCHEMA-INVALID` | 型、enum、required field不正 |
| `MIRAKAN-LIGHTING-UNIT-MISMATCH` | Light typeと物理単位が不一致 |
| `MIRAKAN-LIGHTING-COLOR-AMBIGUOUS` | RGBとCCT等が重複 |
| `MIRAKAN-LIGHTING-GEOMETRY-INVALID` | range、angle、shape、transform不正 |
| `MIRAKAN-LIGHTING-ASSET-UNQUALIFIED` | Cookie／IES artifact未検証 |
| `MIRAKAN-LIGHTING-TARGET-UNSUPPORTED` | Target Capabilityで実行不能 |
| `MIRAKAN-LIGHTING-WORLD_INCOMPATIBLE` | exact World ProfileとLight／Toon compatibilityが不一致 |
| `MIRAKAN-LIGHTING-BUDGET-EXCEEDED` | Planが割当capacityを満たさない |
| `MIRAKAN-LIGHTING-CRITICAL-DROPPED` | Critical Light維持不能 |
| `MIRAKAN-LIGHTING-RUNTIME-OVERFLOW` | 動的selection／cluster上限超過 |
| `MIRAKAN-LIGHTING-STALE-PLAN` | revision／Catalog／Target変更 |
| `MIRAKAN-LIGHTING-LOCK-CONFLICT` | human lockとIntentが競合 |
| `MIRAKAN-LIGHTING-NONDETERMINISTIC` | 同一入力でPlan hash不一致 |
| `MIRAKAN-LIGHTING-BAKE_CLOSURE_INCOMPLETE` | UV、probe、World Cell dependency closureが不完全 |
| `MIRAKAN-LIGHTING-BAKE_ARTIFACT_INVALID` | hash不一致、stale、missing、corrupt Derived Artifact |
| `MIRAKAN-LIGHTING-BAKE_CELL_ACTIVATION_FAILED` | activation groupのall-or-nothing residencyまたはqualified fallback不能 |

missing Source、invalid physical value、unsupported capabilityをdefault light、arbitrary intensity、shadowなしへ黙って置換しない。fallbackはSourceまたはProject profileに宣言され、意味差と適用scopeを表示する。

## 9. Qualificationと完了条件

Qualificationは次のDomain fixtureを持つ。

- 全tagged unionのpositive／negative fixture、lux／lumen／candela／nit変換、Point attenuationの距離別golden、Spot distributionの単位積分、RGB／CCT変換の色差。
- Profile継承／循環／merge、同一入力同一Plan hash、human lock、stale revision、unsupported Target。
- CPU／GPUのZ境界、cluster所属、overflow selection、2D sprite per-light selectionの一致と安定性。
- D3D12／Vulkan／Metalで同じlogical Planを実行し、Light snapshot generation／Compact ID lifetime、device loss、surface resize、Target切替を検証する。
- 最大Light数、最大cluster index、dynamic churnのsoak。run条件はRuntime capacity ownerを使う。
- 2D unlit、2D normal map、pixel art、PBR室内／屋外、Toon、透明物、VFX混在、極端な小／大scale Sceneでbeauty、contribution、shadow、clusterを比較する。Toonはanalytic band／rampのsignal・filter・clamp・channel境界、key／fill／rim／accent、cast／receive shadow、energy cap、World Profile／Target fallbackをMaterial側fixtureと同じexact profile refで検証する。
- AI corpusは「主人公を暖かく、背景を冷たく」「低価格Androidで雰囲気を極力保つ」「pixel artをぼかさず夜にする」「既存の人手調整Key Lightを変えない」「Shadow costを増やさず敵を読みやすくする」を含み、Schema妥当性、lock保持、Target適合、Plan再現性、説明、visual metricで判定する。
- Production BakeはUV overlap／padding、atlas exact／+1、binding atlas membership、finite／normalized／non-empty rect、duplicate binding拒否、probe priority／uncovered IBL、dependency hash invalidation、同一入力のartifact hash、Cell境界／cold streaming、missing／stale／corrupt、partial activation拒否、qualified realtime fallback、offline／fallback／Target比較、Visual／performance receiptをfixture化する。

visual／numeric Evidence、Eval、provenance envelopeは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を参照する。本書はLighting inputとexpected physical／semantic resultだけを所有し、共通gradeやreceipt fieldを再掲しない。

単位なしintensity、type不一致field、Backend enumのpublic露出、Renderer実行規則の複写、silent clamp／fallbackが残る実装はRelease候補にしない。本書はdomain qualification evidenceを出力し、activationと導入順は[Product Plan](../00-product/product-plan.md)が決定する。
