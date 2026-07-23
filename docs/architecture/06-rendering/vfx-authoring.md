# Miraikanai Engine VFX Authoring Contract

- 文書ID: mirakan.arch.rendering-vfx-authoring
- 状態: review
- 正本範囲: VFX Source Document、semantic intent／catalog、typed graph、curve／gradient／random、compiler、VFX固有planned authoring action／diagnostic／qualification
- 非正本範囲: Project HLSL Module／Technique、compiled execution artifact／instance／CPU・GPU simulation／render execution、LOD共通selection、Runtime phase／shared capacity、Asset transaction、Tool／SDK version、AI authorization、Evidence envelope、共通Schema／projection。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Render Graph](render-graph.md)、[Materials](materials.md)、[Project Shader](project-shader.md)、[LOD](lod.md)、[VFX runtime](vfx-runtime.md)、[Environment／surfaces](environment-surfaces.md)
- 外部根拠検証日: 2026-07-22

## 1. 結論と所有境界

VFX Authoringは一つのtyped `VfxSystemDocumentV1`を2D／3D、CPU／GPU向けにoffline compileするSource契約を所有する。Beginner Module Stack、typed Graph、AI intentは同じSourceのprojectionであり、別Assetへ分裂しない。Source、compiler、Editorへnative graphics resource、shader compiler object、Particle buffer、任意code evaluatorを公開せず、Shipping packageへGraph compiler、shader source、JITを含めない。

Coreはgenre名ではなくpresentation role、dimension、temporal role、gameplay relation、style、scale、Targetを組み合わせる。VFXはPresentationだけを変更し、Particle位置、visual collision、GPU event、Particle LightをGameplay、Damage、Save、Navigation、AI perceptionへ逆入力しない。

本書はSource schema、closed catalog、validation、specialization、compile手順までを所有する。compiler出力の構造、実行、lifetime、fallbackは[VFX runtimeのcompiled artifact boundary](vfx-runtime.md#2-compiled-artifact-boundary)だけが定義する。Capability maturity、Activation、導入順は[Product Plan](../00-product/product-plan.md)だけが決定し、本書のqualification結果はそのEvidence入力である。

## 2. Source model

### 2.1 System、Emitter、parameter

```text
VfxSystemDocumentV1
  document_header
  system_id: StableId
  effect_intent_ref: StableId
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

`once`／`loop`は`duration_seconds`を必須とし`[1/60,3600]`秒、`continuous`はnullとする。enabled Emitterは最低1件、portable profileは最大16、advanced profileは最大32 Emitterである。Portable Sourceはdimension-polymorphic Nodeだけを持ち、2D／3D Artifactを別生成する。Dimension-specific instanceは`d2 | d3`を明示し、layerや親Entityから推測しない。System参照cycleを拒否する。

`VfxEmitterV1`は次のfieldを持つ。

| Field | 型／不変条件 |
|---|---|
| `emitter_id`／`enabled` | System内一意Stable ID／bool |
| `execution_policy` | `auto \| cpu_required \| gpu_required \| dual_fallback` |
| `simulation_space` | `local \| world`。実行中変更不可 |
| `max_particles` | `u32`、選択Budget内 |
| `semantic_priority` | LOD ownerの`LodSemanticPriorityV1`。Event registry上限を超えない |
| `rate_q32`／`bursts` | Q32.32 particles/s／`{tick_offset,count}`最大256 |
| `lifetime_seconds` | min／maxがfinite、`[1/60,3600]`、min≤max |
| `prewarm_seconds` | portable `[0,2]`、advanced `[0,5]` |
| `max_events_per_tick` | portable 0、advanced `0..min(max_particles,4096)` |
| `spawn_graph`／`update_graph`／`event_graph` | 各stage一致。eventはadvancedだけ |
| `render_outputs` | portable 1～2、advanced 1～4 |
| `sort_mode` | `none \| spawn_order \| view_depth \| emitter_only`。既定は2D出力`spawn_order`、3D出力`view_depth` |
| `fixed_bounds` | GPU Productionでfinite必須。2D Rect／3D AABB |

`sort_mode`のsort実行、view depth粒子上限、超過時のCook拒否は[VFX runtimeのGPU sort契約](vfx-runtime.md#5-gpu-simulationrendercollision)が所有する。上限超過Sourceの解消は`sort_mode`の`emitter_only`への明示変更またはadditive Materialへの明示変更だけで行い、Compilerが`sort_mode`を暗黙変更しない。

Rate、Burst、Event spawnは一つのadmissionへ通す。Prewarmはloop／continuousかつpreload対象だけに許可し、動的one-shotは0とする。過去状態が必要なone-shotは`VfxBakeCacheV1`を使う。Sub-emitterは深さ2、親Particle当たり8、生成eventは次stepへだけ渡す。

`VfxParameterV1`は`parameter_id`、closed `value_type`、型一致finite `default_value`、numeric min／maxまたはenum allowlist、`update_rate: instance_start | presentation_frame`、registered `semantic_role`、`exposure: hidden | inspector | gameplay | ai`を持つ。Particle直接write、pointer、array、string、Entity、native resourceをparameterにしない。

`VfxEventInputV1`はregistered Presentation Event Type IDと最大256 byteのclosed payload schemaを持つ。payloadはinstance開始時にcaptureしてimmutableとし、異なるEventを暗黙mergeしない。`from_presentation_event` seedではpayloadの`vfx_seed:u64`を必須とする。

### 2.2 Graph contracts

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

GraphはstageごとのDAGである。Node Catalogはtypeごとにstage、dimension、version、typed ports、literal、default有無、attribute read／write、Capability、CPU kernel ID、GPU implementation IDを定義する。必須portは型一致edgeまたはliteralを一つだけ持つ。unknown／duplicate port、implicit cast、配列自動拡張、Graph外edge、暗黙last-write-winsを拒否し、複数writeは`CombineAdd | CombineMultiply | CombineMin | CombineMax`をNode Stable ID順で評価する。

SpawnはPosition／Lifetimeを必須bindする。未bind初期値は2D／3D Velocity zero、Color `(1,1,1,1)`、Sprite／Billboard Size `(1,1)`、Mesh Size `(1,1,1)`、Rotation／AngularVelocity／FlipbookFrame 0であり、Update未bind値は保持する。

`event_routes`はevent stageだけ、pulse sourceは`OnDeath | OnVisualCollision | ConditionRising`、`child_count`は1～8、payloadは256 byte以下とする。`ConditionRising`は前step boolをpersistent attributeに持ち、連続trueで再発火しない。event pulseはpublic parameter、literal、保存field、render outputへ使わない。

Canonical serializationはStable ID byte、field ID、Edge IDの昇順であり、表示名、挿入順、screen座標をcompile順へ使わない。`VfxGraphLayoutV1`はProject revision／Undo対象だがsemantic hashとCook invalidationから除く。

Closed value typeは`bool, i32, u32, u64, f32, vec2f, vec3f, vec4f, color_linear_rgba, quaternionf, spatial_vector, curve1_ref, gradient_ref, texture2d_ref, mesh_ref, material_ref`である。`spatial_vector`は2Dで`vec2f`、3Dで`vec3f`へspecializeする。距離meter、時間second、角度radian、色linear RGBAとし、NaN／Inf、非正規Quaternion、暗黙scalar／vector変換を拒否する。

Stageは次に閉じる。previous値はCompiler-declared `PreviousAttribute`だけから読み、Graph edgeでcycleを作らない。

| Stage | 実行時点 | 読取 | 書込 |
|---|---|---|---|
| `emitter_control` | Emitterごと | Parameter、Event、Emitter transform、tick | spawn count、Emitter local state |
| `particle_spawn` | 新規Particleごと | spawn ID、seed、Parameter、Emitter transform | 初期Particle attribute |
| `particle_update` | alive Particleごと、fixed step | 現attribute、Parameter、time | 次attribute、alive flag |
| `particle_event` | advanced条件成立時 | Particle attribute、visual collision result | bounded internal VFX event |
| `render_output` | Snapshot／Render packet構築時 | 最終attribute、Material parameter | Renderer bindingだけ |

`emitter_control`と`render_output`はEngine-ownedであり任意Source Graphを持たない。Node Catalog entryはNode typeごとに許可Stage、Dimension、version、typed input／output port、literal field、default有無、attribute read／write、Capability、CPU kernel ID、GPU implementation IDを定義する。必須inputは型一致edgeまたは明示literalをちょうど一つ持ち、optional inputだけがCatalog defaultを使える。未知port、未接続必須port、同一target portへの複数edge、Graph外edge、implicit cast、名前によるport解決、配列index自動拡張を拒否する。`output_bindings`はattribute当たり一つだけである。

Portable Node CatalogのIDを次に閉じる。

| 分類 | Portable Node ID |
|---|---|
| Input | `Constant, Parameter, Age, NormalizedAge, Lifetime, SpawnId, SimulationStep, EmitterTransform, EventField` |
| Math | `Add, Subtract, Multiply, DivideSafe, Min, Max, Clamp, Abs, SqrtSafe, Lerp, Remap, Dot, Length, NormalizeSafe, Select` |
| Random | `Uniform01, Range, UnitDirection2D, UnitDirection3D, RandomColorGradient` |
| Curve | `SampleCurve, SampleGradient` |
| Spawn shape 2D | `Point, LineSegment, Rectangle, CirclePerimeter, Disk` |
| Spawn shape 3D | `Point, LineSegment, Box, SphereSurface, SphereVolume, Cone` |
| Initialize | `Position, Velocity, Lifetime, Color, Size, Rotation, AngularVelocity, FlipbookFrame` |
| Update | `IntegrateVelocity, Acceleration, Gravity, LinearDrag, ColorOverLife, SizeOverLife, RotationOverLife, KillByAge` |
| Output | `Sprite2D, Billboard3D, PortableFacingSprite, Flipbook, BasicTrail` |

`DivideSafe`は分母絶対値が`1e-8`未満、`NormalizeSafe`は長さが`1e-8`未満ならNode parameterで明示したfallbackを返す。Compilerが0除算を0へ置換しない。Advanced catalogはCurl／value noise、turbulence、Vector Field sampling、Mesh surface／volume emission、Depth／Global SDF／VFX collision proxy、collision bounce／friction／kill、visual collision event、Sub-emitter／GPU event、Mesh／Ribbon／Particle Light output、depth fade／soft particle／distortion parameter、camera distance／quality parameter／LOD branch、qualified `VfxExtensionOperatorV1`を追加する。Scene color sampling、arbitrary texture write、ray tracing、mesh shader、unbounded bindless、GPU readback eventは含めない。

| 上限 | Portable | Advanced |
|---|---:|---:|
| Node／Emitter | 256 | 1,024 |
| Edge／Emitter | 1,024 | 4,096 |
| persistent custom attribute／Emitter | 16 | 64 |
| Curve／Emitter | 32 | 128 |
| Key／Curve | 64 | 256 |
| public parameter／System | 64 | 64 |
| render output／Emitter | 2 | 4 |

超過時にCompilerが意味を変えてEmitter分割しない。

### 2.3 Curve、gradient、seed

`VfxCurveV1`のkey timeは`[0,1]`で厳密昇順、最初0、最後1、interpolationは`step | linear | cubic_hermite`である。tangentはfinite、weighted tangentは不許可。`VfxGradientV1`はlinear RGBA key最大64、同時刻のcolor／alpha重複を拒否する。Sourceはstraight alpha、premultiplyはMaterial blend契約で行う。

GPU LUTは64、128、256、512、1,024 sampleを順に試し、各key区間の端点／中央／四分点でscalar誤差`max(1e-4, source_range*1e-3)`、color各channel誤差`1/1024`以下の最小値を選ぶ。1,024でも不合格ならGPU variantを拒否する。

`VfxCounterRngV1`はCPU／GPU共通`Philox4x32-10`であり、per-particle RNG stateを持たない。

```text
emitter_seed32 = LE32(SHA256("MIRAKAN_VFX_EMITTER_SEED_V1\0" || emitter_id_canonical_bytes)[0..3])
counter = [low32(particle_spawn_id), high32(particle_spawn_id), simulation_step_u32, random_slot_u32]
key = [low32(system_seed_u64) xor emitter_seed32, high32(system_seed_u64)]
random_f32 = (u32_value >> 8) * 0x1p-24f
instance_seed = LE64(SHA256("MIRAKAN_VFX_INSTANCE_SEED_V1\0" || project_seed_le64 || instance_sequence_le64)[0..7])
```

Spawn stage stepは0、最初のUpdate／Eventは1である。random slotはEmitter revision内で再利用せず、1 Node最大4 laneとする。instance sequenceはcanonical merge済みspawn順でありthread completion順ではない。

## 3. Semantic intent、catalog、extension

```text
VfxEffectIntentV1
  intent_id: StableId
  semantic_role_id: VfxSemanticRoleId
  spatial_scope: d2 | d3 | hybrid | portable_2d_3d
  temporal_role: one_shot | loop | continuous
  gameplay_relation: cosmetic | authoritative_event_presentation | visual_collision_only
  style_role_ids: StableId[0..16]
  target_selector
  scale_envelope: {maximum_concurrent_instances, maximum_visible_instances,
                   maximum_spawn_per_second, maximum_burst_per_tick}
  semantic_priority: LodSemanticPriorityV1
  minimum_cue_contract: MinimumCueContractV1
  fallback_policy: semantic_equivalent_only | approval_required | no_fallback
  extension_policy: core_only | qualified_extension_allowed

MinimumCueContractV1
  required_invariants: VfxCueInvariantId[0..16]
  minimum_visible_steps: u32
  minimum_projected_coverage_px_q16: optional u32
  required_contrast_class: optional low | medium | high
```

Cue invariant初期集合は`onset_timing, duration_class, directional_readability, boundary_readability, team_readability, silhouette_class, minimum_visibility, contrast_floor`である。Role Catalogの必須invariantをIntent、Pattern、LOD、Fallback、Qualityのどこでも削除しない。Hybridは同じChangeSetに明示したd2 Systemとd3 Systemを必要とする。

`VfxSemanticRoleCatalogV1`はRole ID、説明、許可space／time、必須Cue、priority上限、Presentation Event、候補Patternを持つ。初期Roleは`impact_confirmation, projectile_presentation, motion_trail, area_boundary_warning, status_loop, environment_ambient, weather_precipitation, interaction_feedback, spawn_despawn_transition, progress_success_feedback`である。

`VfxPatternCatalogV1`はPattern ID、Role、dimension-polymorphic graph descriptor、Capability、Cost formula、Fallback Pattern、Reference Effect、Catalog versionを持つ。初期Patternは`portable_burst, portable_flipbook_loop, portable_motion_trail, portable_area_boundary, portable_status_loop, portable_ambient_field, portable_precipitation, portable_interaction_pulse, portable_spawn_despawn, portable_success_burst`である。Pattern適用は一回限りのSource proposalで、runtime inheritanceを作らず、Pattern ID／version／Intent hash／ChangeSet hashを記録する。

```text
VfxExtensionManifestV1
  extension_id: StableId
  extension_version: SemVer
  extension_content_hash: SHA-256
  owner_ref: exact {owner_id, owner_revision, owner_content_hash}
  semantic_role_ids: VfxSemanticRoleId[1..32]
  operator_ids: VfxExtensionOperatorId[1..64]
  supported_dimensions: set<d2 | d3 | portable_2d_3d>[1..3]
  supported_target_profiles: TargetProfileId[1..32]
  capability_requirements: CapabilityId[1..64]
  cost_model_id: StableId
  determinism_class: cpu_repeatable | visual_only
  fallback_pattern_refs: VfxPatternRef[0..16]
  preserved_cue_invariants: VfxCueInvariantId[0..16]

VfxExtensionManifestRefV1
  extension_id: StableId
  extension_version: SemVer
  extension_content_hash: SHA-256

VfxExtensionQualificationSubjectV1
  qualification_id/version
  owner_ref: exact {owner_id, owner_revision, owner_content_hash}
  extension_ref: exact VfxExtensionManifestRefV1
  target_profile_refs[1..32]
  declared_dimensions[1..3]
  fixture_refs[1..64]: exact fixture ref/version/content_hash
  input_closure_hash
  result: pass | fail
  qualification_subject_hash

VfxExtensionQualificationReceiptV1
  subject: VfxExtensionQualificationSubjectV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=vfx_extension_qualification)

VfxExtensionActivationBindingRefV1
  activation_binding_id/version
  activation_binding_hash: SHA-256

VfxExtensionActivationBindingV1
  activation_binding_id/version
  extension_ref: exact VfxExtensionManifestRefV1
  qualification_receipt_refs[1..64]:
    exact {qualification_id, qualification_version,
           qualification_subject_hash, signed_record_hash}
  activation_binding_hash

VfxExtensionActivationProjectionV1
  projection_id/version
  extension_ref: exact VfxExtensionManifestRefV1
  activation_binding_ref: VfxExtensionActivationBindingRefV1
  activated_target_profile_refs[1..32]
  activated_dimensions[1..3]
  projection_hash: SHA-256
```

`extension_content_hash`はASCII `MIRAKAN_VFX_EXTENSION_MANIFEST_V1`、`qualification_subject_hash`はASCII `MIRAKAN_VFX_EXTENSION_QUALIFICATION_SUBJECT_V1`、`activation_binding_hash`はASCII `MIRAKAN_VFX_EXTENSION_ACTIVATION_BINDING_V1`、`projection_hash`はASCII `MIRAKAN_VFX_EXTENSION_ACTIVATION_PROJECTION_V1`と、それぞれ自己Fieldだけを除くcount／length-framed canonical bytesから計算する。Manifest `owner_ref`はExtensionを寄与するCore／Pack／Project owner recordへexact解決し、Extension ID prefixやsigner自己申告から補完しない。Subject `owner_ref`は`extension_ref`が解決するReceipt-free Manifestの同Fieldとbyte equalityにする。生成順は`receipt-free Manifest → extension ref → Qualification subject → signed Receipt → root外Activation Binding → root外Activation projection`で、Receipt／Binding／Projection／FixtureをManifest hashへ戻さない。

Projectionの`extension_ref`、解決したBindingの`extension_ref`、各Receipt wrapper内Subjectの`extension_ref`はbyte equalityで、各Subject ownerは解決したManifest `owner_ref`とbyte equalityでなければならない。ProjectionのTarget／dimension集合はstrict sort／uniqueとし、各集合は解決したpass Receipt subject群の`target_profile_refs[]`／`declared_dimensions[]`のunionとexact set equalityにする。各Receipt subjectのTarget／dimensionはManifestの`supported_target_profiles`／`supported_dimensions`のsubsetであり、Production候補ではManifestが宣言した全Target／dimensionをProjection集合へ含める。BindingのReceipt refは`qualification_id`／version／subject hash／signed hashのcanonical順で、duplicate、fail、stale、revoked、別owner／extension subjectを拒否する。ProjectionはBinding refをexact一件だけ持ち、Binding refのID／version／hashを解決したBindingと一致させる。Manifestのowner／Target／dimension一Fieldだけを更新したstale Projection、別ownerの有効Subjectまたは別extensionの有効Bindingへのsubstitution、Target／dimensionの欠落・追加・duplicate・順序違反を各一原因fixtureで拒否する。

Manifestは対象外Targetを明示し、Production候補には上記Projectionが解決する各宣言dimension／Targetのpass Receiptを最低1件必要とする。semantic-equivalent intentは同じCueを保つfallbackを必須とし、不足時はCookを拒否する。`VfxExtensionOperatorV1`はtyped ports、stage、dimension、attribute set、bounded scratch、determinismを宣言し、CPUはNative Game Moduleのbounded SoA spanだけ、GPUは[Project Shader](project-shader.md)のS2 `ProjectShaderModuleV1`または`vfx_simulation | vfx_render` Portへ接続するS4 `ProjectShaderTechniqueV1`だけを扱う。allocation、World／Physics、file／network、logging、native GPU API、mutable static stateを禁止する。ExtensionのauthorizationはGovernance ownerへ委譲する。

## 4. Compiler、planned authoring action、preview

`VfxCompiler`は次の固定順で動く。

1. Document、Stable ID、range、Asset ref、Targetを検証する。
2. stage、type、dimension、attribute access、cycleを検証する。
3. Node Catalog versionとCapabilityを検証する。
4. Emitter／System／Projectのdomain cost、memory、spawn、renderer、overdraw estimateを検証する。
5. Node Stable ID tie-breakのcanonical topological orderを作る。
6. constant folding、dead-node elimination、attribute livenessを行い、float演算を再結合しない。
7. d2／d3 specializationで不要fieldを除く。
8. execution policy、Node requirement、Target capability、domain thresholdsからvariantを解決する。
9. CPUはclosed kernel plan、GPUはportable shader IRとEngine-owned Pass TemplateまたはQualification済みProject Shader Module／Technique参照を生成する。
10. [Project Shader](project-shader.md)のoffline compile／Fact／Understanding validationとRenderer pipeline compileを依頼する。
11. source／compiler／interface／resource／fixture hashを生成する。
12. Asset lifecycle ownerへtransactional Cookとclosure promotion候補を渡す。

Execution dispositionは`auto | cpu_required | gpu_required | dual_fallback`に閉じ、Compilerがenabled Emitterごとに独立して次の固定順で解決する。同じSystem内のCPU／GPU EmitterはParameter、System seed、Instance transform、lifecycleだけを共有し、Particle storageを共有しない。

| 条件 | 解決 |
|---|---|
| Portable Asset | CPU |
| `cpu_required` | CPU capabilityとdomain budgetがなければ失敗 |
| `gpu_required` | advanced、compute、storage buffer、atomic counter、indirect draw、shader variantが全てなければ失敗 |
| `dual_fallback` | CPUとGPUを両方Cookし、Target Quality Manifestが一方を正規選択 |
| `auto`かつGPU-only Nodeを使用 | GPU。GPU非対応Targetには明示fallback graphが必要 |
| `auto`かつpeak alive見積り4,096以上 | GPU capabilityがあればGPU、なければCPU domain budget内の場合だけCPU |
| `auto`かつpeak alive見積り4,095以下 | CPU |
| `mobile_baseline` | CPUを正規選択。限定GPUはDevice Qualification Receiptで明示enableしたProfileだけ |

peak alive見積りはEmitterごとに粒子数の整数値`min(max_particles, ceil(rate_q32 × lifetime_seconds.max) + Σ bursts.count + max_events_per_tick)`で算出する。丸めは`ceil`だけを使い、上表の閾値比較はこの値で行う。見積りはauthored fieldだけから決まり、runtime実測やframe負荷を入力にしない。

Runtimeがframe負荷から実行中stateをCPU／GPU間で移送しない。Target／Qualityごとの決定は[Runtime側のcompiled artifact boundary](vfx-runtime.md#2-compiled-artifact-boundary)へ渡し、Instance開始時に一意に選ぶ。Authoringは解決入力とalgorithmだけを所有し、compiled artifact schemaを再定義しない。

Unsupported Node、layout overflow、shader failure、fallback不足、domain budget超過は該当variantを失敗させる。`VfxGraphIrV1`はdevelopment-only compiler intermediateでSource／Runtime保存しない。`VfxBudgetProfileV1`／`VfxQualityProfileV1`はTarget／Project Source、`VfxBakeCacheV1`はSource artifact hash、seed、fixed delta、frame count、Target formatを持つoffline cacheである。

予約候補IDは`operation.vfx.inspect_system, operation.vfx.inspect_semantic_catalog, operation.vfx.validate_changeset, operation.vfx.validate_semantic_preservation, operation.vfx.preview_changeset, operation.vfx.resolve_effect_intent, operation.vfx.set_effect_intent, operation.vfx.apply_pattern, operation.vfx.create_system, operation.vfx.create_emitter, operation.vfx.update_emitter, operation.vfx.delete_emitter, operation.vfx.add_node, operation.vfx.update_node, operation.vfx.delete_node, operation.vfx.connect_nodes, operation.vfx.disconnect_nodes, operation.vfx.set_curve, operation.vfx.set_gradient, operation.vfx.set_output, operation.vfx.generate_fallback, operation.vfx.capture_bounds, operation.vfx.run_qualification, operation.vfx.propose_extension_operator`のexact 24件であり、[Executable contracts](../02-foundation/executable-contracts.md#211-既存domain文書から回収した未登録operation候補)の`planning.operation_family.vfx_authoring@1`だけに属する。これらはMCD Operationではなく、Capability stateは`not_activated`、current MCD／Owner Manifest／Service allowlist／Policy／Validator／Diagnostic／Receipt／Provider／MCP／generated alias／legacy alias集合はすべて`[]`である。`activation.vfx.authoring_operations.v1`が24件を同じContract set transactionで完全登録するまでGatewayはdispatchせず、要求を`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`でSource不変として拒否する。Write／Preview／Qualification等の記述はActivation審査用の予定意味だけであり、現在の`ProjectChangeSetV1`生成、live buffer／resource／revision変更、共通envelope、projection、authorizationを与えない。

Capability IDは`capability.vfx.system, capability.vfx.particle_cpu, capability.vfx.particle_gpu, capability.vfx.sprite_2d, capability.vfx.billboard_3d, capability.vfx.trail, capability.vfx.mesh_ribbon, capability.vfx.visual_collision, capability.vfx.particle_light, capability.vfx.bake_cache, capability.vfx.semantic_intent, capability.vfx.pattern_catalog, capability.vfx.extension_operator`に閉じる。利用可能性はProduct activation manifestを読み、本書からmaturityを推測しない。

Resolverは最大3 Pattern／Graph候補、missing requirement、assumption、Target cost、Cue diffを返す。Scene dimension、Gameplay authority、Target、必須Cue、Extension policyが解決不能なら質問へ停止する。CPU／GPU、thread、buffer、shader model、sort algorithmを初心者へ質問しない。Stack、Graph、Inspector、Curve、Timelineは同じoperationへ収束し、human lockを保つ。

Previewはfixed seed、fixed simulation step、Reference Qualityを既定とし、seed、Target、Quality、artifact kind、bounds、velocity、collision proxy、count、drop、time、memory、overdraw、sort costを表示する。StackとGraphは別Sourceを持たない。`VfxAiAuthoringFixtureV1`は360 caseで、240 clear、40 hybrid、40 ambiguous／high-impact、40 unsupported／adversarialとする。各caseはIntent、allowed Pattern、required question、forbidden operation、Target、Budget、golden Cue、expected Diagnostic、canonical ChangeSet hash集合を持つ。

## 5. Diagnostic、failure、versioning

Authoring／compiler closed Diagnosticは`VfxSchemaInvalid, VfxMissingReference, VfxGraphCycle, VfxTypeMismatch, VfxStageViolation, VfxDimensionMismatch, VfxSemanticRoleUnknown, VfxIntentUnresolved, VfxCueContractViolation, VfxSemanticDrift, VfxPatternUnsupported, VfxExtensionRequired, VfxFallbackApprovalRequired, VfxCapabilityUnavailable, VfxNodeCatalogMismatch, VfxBudgetExceeded, VfxMemoryExceeded, VfxMemoryEstimateMismatch, VfxOverdrawExceeded, VfxBoundsInvalid, VfxCompileFailed, VfxShaderCompileFailed, VfxArtifactInterfaceMismatch, VfxArtifactStale, VfxExtensionRejected`である。各Diagnosticはsystem／emitter／optional Node・Field、Project revision、Artifact hash、Target、Quality、actual、limit、remediation IDを持つ。

Semantic不明／Intent unresolvedはproposalを停止し、Cue driftはPreview／Cookを拒否する。Schema／graph／type／dimension、Capability／fallback、budget／memory／overdraw、shader／interface failureではlive Sourceとlast valid artifactを変えない。unknown Node／attribute／blend／execution policyをdefaultへ置換しない。

System、Intent、Role Catalog、Pattern Catalog、Extension Manifest、Node Catalog、IR、Artifactは独立versionを持つ。Source major変更はmigration tool、before／after hash、semantic diff、golden更新を必要とする。Node削除はdeprecation diagnosticと全fixture migration後に行う。Compiler build、Target、Quality、Node Catalog、shader toolchain変更は再Cookする。外部Particle Asset importerは正本互換ではなくloss report付き変換だけを許す。`VfxCounterRngV1`はPresentation専用で、Runtime ownerのauthoritative `DeterministicRngV1`を置き換えない。

## 6. Qualification

Contract qualificationは全Source型のround-trip、unknown／duplicate／missing／cycle／type／stage／dimension negative fixture、canonical compile hash、optimization前後scalar一致、LUT誤差、Philox test vector、portable／advanced node matrix、CPU／GPU混在、cross-target sub-emitter拒否、resource estimate対instrumented peak、adversarial graph fuzz／timeout／memory capを含む。

Reference Effectは2D hit spark、fire／smoke、trail、3D hit spark、fire／smoke、rain／snow、magic projectile、explosion、GPU mesh／ribbon、crowded battle compositeを含む。AI／Editor qualificationはsemantic role分解、2D／3D／hybrid、Pattern／LOD／Target／Extension Cue diff、manual／AI ChangeSet一致、lock／stale／unsupported／Gameplay collision拒否、Undo／Redo 10,000回、Cook failureとold generation retireを検証する。

AI fixture hard gateはBlocking／High Impact見落とし0、unknown Role／Node／planned semantic action token 0、dimension／temporal／gameplay relation誤解決0、Gameplay逆入力0、Cue破壊0、silent feature removal 0、clear case ChangeSet不一致0、Core外要求のExtension-requiredまたは明示拒否100%、task success 95%以上、不要Blocking質問5%以下である。Evidence envelope、run数、holdout、gradingはAI Verification ownerが決定する。
