# Miraikanai Engine LOD Contract

- 文書ID: mirakan.arch.rendering-lod
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: LOD intent／policy、representation set／tier、projected-error／importance／pressure入力、hysteresis／transition、geometry／HLOD／simulation／animation／material／VFX／terrain representation selection、LOD固有operation／diagnostic／qualification
- 非正本範囲: Terrain／Foliage Source／artifact、representation asset生成／promotion／runtime residency、GI／reflection technique、Render pass／visibility execution、World streaming activation、Simulation behavior、Runtime shared capacity／phase、Tool version、AI authorization、Evidence envelope、共通Schema／projection。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[World](world.md)、[Camera](camera.md)、[Render Graph](render-graph.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)
- 関連文書: [Product Plan](../00-product/product-plan.md)、[Initial Morph Capability Boundary Decision](../decisions/2026-08-03-initial-morph-capability-boundary.md)、[Advanced Rendering／Multiplayer Ownership Decision](../decisions/2026-07-29-advanced-rendering-multiplayer-ownership.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Math／Core utilities](../02-foundation/math-core.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Persistence／Save](../04-runtime/persistence-save.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[Animation](../05-simulation/animation.md)、[Physics](../05-simulation/physics.md)、[Collision](../05-simulation/collision.md)、[Navigation](../05-simulation/navigation.md)、[Camera](camera.md)、[Render Graph](render-graph.md)、[Advanced Light Transport](advanced-light-transport.md)、[Terrain／Foliage](terrain-foliage.md)、[Materials](materials.md)、[VFX Authoring](vfx-authoring.md)、[VFX Runtime](vfx-runtime.md)、[Environment／surfaces](environment-surfaces.md)、[Virtualized／Continuous Geometry](virtualized-continuous-geometry.md)、[World](world.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-28

## 1. 結論と所有境界

LODは距離だけでなく、projected error、semantic importance、view role、quality intent、runtime pressureを入力に、どのrepresentationを選ぶかを一意に所有する。各Subsystemは候補representationとcost／quality metadataを公開し、LOD Resolverが選択、hysteresis、transition、fallbackを決める。

[Render Graph](render-graph.md)は選択済みgeometry／material representationのvisibilityとdraw executionを所有する。[World](world.md)はWorld／Scene Sourceと`WorldStreamingPlanV1`内のplan-local Cell descriptorを所有し、Runtime scheduling／capacity ownerがactivationとpressureを決める。Physics、Navigation、Animationは各Domainのbehavior semanticsを所有し、LODが別のDynamics／Nav／Animation規則を作らない。

Asset import、cook、promotionは[Asset lifecycle](../03-authoring/asset-lifecycle.md)、Runtime request／generation／residency／lease／evictionは[Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md)だけが所有する。本書はartifactを生成またはresident化せず、同一source identityへ紐づく候補artifactの選択条件を定義する。

### 1.1 current／target読解とAI可読性

| 観点 | current reading | target contract |
|---|---|---|
| 設計 | `design-reviewed`の計画正本 | 本書と各Owner文書のexact ref／closed ruleへ収束する |
| 実装 | `absent` | materialized MCD Schema、compiler、runtime resolver、projectionが必要 |
| AI概念理解 | vocabulary、Owner、禁止逆入力を説明可能 | `LodAuthoringContextV1`からboundedにread／compare／explainできる |
| AI操作 | LOD operationは`not_activated` | active Contract set、Policy、Approval、Validator、Receiptが揃う場合だけproposal可能 |
| 表現自由度 | Profile候補として設計済み、利用可能機能の主張ではない | exact Profile／tier／transition／fallbackをOwner-qualified extensionで追加できる |

したがって現時点は`conceptually-readable`であって`operationally-readable`ではない。型名、operation候補、Profile記述を実装済み、dispatch可能、Production qualifiedと解釈しない。AIはContextに含まれないsubject、Profile、Target、Quality、pressure、fieldを補完せず、`unknown | omitted | stale | not_activated`を区別する。

## 2. 設計判断と共通語彙

LOD語彙を次に固定する。

- `representation`: 同じSource意味を異なるcost／fidelityで表す候補。
- `tier`: 高品質からfallbackまでのcanonical順序を持つ候補識別子。
- `intent`: quality、importance、transition、minimum semanticsのauthoring要求。
- `selection`: View／Domain／subjectごとに解決されたtierと理由。
- `transition`: old／new representationの共存、blend、handoff、history reset条件。
- `pressure`: Runtime ownerが公開するcapacity signal。LODは測定法や閾値を再定義しない。
- `HLOD`: 複数subjectを一つのrender representationへ集約するWorld aggregation候補。World identityの置換ではない。
- `virtual geometry micro-cluster`: 一つのvirtualized geometry Artifact内部のhierarchy node。HLOD cluster、World Cell、LOD tierではない。

Mesh／Sprite geometryのLOD0は常にSourceの最高detailであり、Sourceを削除／上書きしない。別Domainのtierはclass固有IDで表し、geometry LOD indexを暗黙転用しない。distanceのみ、asset filename、Backend feature名を選択の正本にしない。

`LodClassV1`を次のclosed enumとする。同じobjectは複数classのPolicyを持てるが、一classのtierを別classへ転用しない。

| 値 | 意味 | 主なOwner |
|---|---|---|
| `geometry_detail` | Mesh／Spriteの幾何・表示detail | Asset／Rendering |
| `representation` | individual、instanced、proxy、impostor、非表示 | Rendering |
| `simulation` | Full、契約済み低頻度、dormant record | Runtime／Gameplay |
| `animation_presentation` | pose評価、skinning、presentation bone | Animation／Rendering |
| `material_detail` | 承認済みMaterial feature variant | Material／Rendering |
| `vfx_presentation` | VFX branch、spawn、alive、output detail | Particle／VFX |
| `surface_detail` | Terrain／Foliage／Water／Snow Surfaceの表示detail | Domain Platform |
| `geometry_residency` | geometry cluster／LODのresident集合 | Asset／Rendering |

`LodSemanticPriorityV1`は`critical_gameplay_cue | interactive_subject | primary_subject | supporting_subject | decorative | ambient`のclosed enumである。`critical_gameplay_cue`はhit、危険範囲、parry timing、目標状態等に使用し、decorative／ambientを理由にauthoritative stateを削除しない。`experience_role`と矛盾する場合は`MIRAKAN-LOD-SEMANTIC_ROLE_CONFLICT`で拒否する。

### 2.1 外部Engine比較から採るもの／採らないもの

| 比較対象 | official behavior | 本契約で採る判断 | そのままは採らない判断 |
|---|---|---|---|
| Unreal Engine 5.8 | Static Mesh LODはdistanceではなくscreen sizeで切り替え、HLODは複数Static Meshをproxy mesh／materialへ集約する。[LOD](https://dev.epicgames.com/documentation/en-us/unreal-engine/creating-and-using-lods-in-unreal-engine)、[HLOD](https://dev.epicgames.com/documentation/unreal-engine/hierarchical-level-of-detail-in-unreal-engine?lang=en-US) | screen-space metric、HLODのidentity非置換、Target別profile、fallbackを採る | Actor／Asset型、Backend名、既定screen percentageをMiraikanaiの正本にしない |
| Unity 6 | `LODGroup`はscreen上のsizeでRenderer集合を切り替え、cross-fadeとtransition zoneを持つ。[LODGroup](https://docs.unity3d.com/jp/current/Manual/class-LODGroup.html) | authorがtier集合／遷移範囲を明示できること、ViewごとのPreview、cross-fadeを採る | GameObject hierarchy、global duration、shader uniformを共通LOD契約にしない |
| Godot stable | import時のautomatic Mesh LODはscreen-space metric、Visibility Rangeはmanual HLOD／impostor／hysteresisを提供する。[Mesh LOD](https://docs.godotengine.org/en/stable/tutorials/3d/mesh_lod.html)、[Visibility Range](https://docs.godotengine.org/en/stable/tutorials/3d/visibility_ranges.html) | 生成と選択の分離、FOV／viewportを含むmetric、manual／generated候補の併用、hysteresisを採る | Node visibilityをGameplay relevancyまたはauthoritative stateの削除へ転用しない |

[Unreal EngineのNanite](https://dev.epicgames.com/documentation/en-us/unreal-engine/nanite-technical-details)に相当するvirtualized／continuous geometryは、discrete Mesh LOD／HLODと異なるhierarchy、virtual residency、Backend capability、fallback、toolchain、qualificationを必要とするため、詳細な統合正本を[Virtualized／Continuous Geometry](virtualized-continuous-geometry.md)へ分離する。本書のouter Resolverは一つのrepresentation descriptorを選び、subject aggregation=`individual | instanced | spatial_proxy | impostor | hidden_presentation`とgeometry family=`discrete | virtualized | not_applicable`を別軸のexact target bindingで解決する。qualifiedなHLOD proxy Artifactはgeometry familyとしてvirtualizedを使えるが、HLOD clusterとmicro-clusterのidentityは統合しない。内部のView-local hierarchy cutをtier、hysteresis、Save stateまたはSimulation inputとして所有しない。Capabilityが`planning_only`の現在はvirtualized bindingを登録せず、本書の`geometry_detail`は明示tier chainだけを意味する。Nanite名、meshlet、GPU indirect、resident resourceまたはBackend supportからcontinuous geometry対応を推測しない。

## 3. LOD policyとrepresentation contract

### 3.1 `LodIntentV1`

```text
LodIntentV1
  schema_version: 1
  intent_id: StableId
  intent_version: positive u32
  owner_ref: exact {owner_id, owner_revision, owner_content_hash}
  semantic_role: registered StableId
  semantic_priority: LodSemanticPriorityV1
  gameplay_fidelity_floor_ref: exact owner-typed ref
  visual_fidelity_floor_ref: exact owner-typed ref
  style_profile_ref: exact owner-typed ref
  target_profile_refs[1..16]: TargetProfileRefV1
  expected_peak_resident: u32
  expected_peak_visible: u32
  expected_view_distance_m: finite non-negative f64
  interaction_radius_m: finite non-negative f64
  locked_capability_refs[0..64]: CapabilityRefV1
  allowed_lod_classes[1..8]: sorted unique LodClassV1
  forbidden_degradation_refs[0..64]: registered CapabilityRefV1
  source_revision: positive u64
  intent_content_hash: SHA-256

LodIntentRefV1
  intent_id: StableId
  intent_version: positive u32
  intent_content_hash: SHA-256
```

`intent_content_hash`はASCII `MIRAKAN_LOD_INTENT_V1`と自己hashを除くcanonical bytesから計算する。`expected_peak_visible <= expected_peak_resident`を必須とする。`expected_view_distance_m`はPreview／fixture入力でruntime switch thresholdではない。gameplay floorはowner-typed authoritative instance数、Damage、Collision、Navigation、goal、spawn timing、input response、critical cueをtyped fieldで持ち、visual floorはsilhouette、material feature、animation cue、pixel-art sampling、minimum visible size、critical VFX outputを持つ。IntentからPolicyへのback-referenceを持たず、`policy_set_id`によるhash cycleを作らない。

### 3.2 `LodPolicySetV1`

```text
LodPolicySetV1
  schema_version: 1
  policy_set_id: StableId
  policy_version: positive u32
  intent_ref: exact {intent_id, intent_version, intent_content_hash}
  domain_profile_refs[1..32]: LodDomainProfileRefV1
  selection_table_ref: LodSelectionTableRefV1
  missing_pressure_policy:
    preserve_previous_or_highest_fidelity
  preset_provenance_ref: LodPolicyPresetProvenanceRefV1 | null
  policy_lock_refs[0..64]: registered exact refs
  policy_content_hash: SHA-256

LodPolicySetRefV1
  policy_set_id: StableId
  policy_version: positive u32
  policy_content_hash: SHA-256

LodDomainProfileRefV1
  lod_class: LodClassV1
  subject_scope_ref: exact owner-typed ref
  profile_id: StableId
  profile_version: positive u32
  profile_content_hash: SHA-256

LodOwnerPayloadHeaderV1
  schema_version: positive u32
  payload_content_hash: SHA-256

LodDomainProfileV1
  schema_version: 1
  lod_class: LodClassV1
  subject_scope_ref: exact owner-typed ref
  profile_id: StableId
  profile_version: positive u32
  owner_ref: exact {owner_id, owner_revision, owner_content_hash}
  representation_set_ref: LodRepresentationSetRefV1
  tier_bindings[1..32]:
    tier_id: StableId
    representation_id: plan-local StableId
  owner_payload_type_ref: exact versioned type ref
  owner_payload_content_hash: SHA-256
  required_capability_refs[0..32]: sorted unique CapabilityRefV1
  profile_content_hash: SHA-256

LodTierRefV1
  domain_profile_ref: LodDomainProfileRefV1
  tier_id: StableId
  representation_ref: LodRepresentationRefV1
```

`profile_content_hash`はASCII `MIRAKAN_LOD_DOMAIN_PROFILE_V1`と自己hashを除くcanonical envelope bytesから計算し、Owner payload hashとRepresentation Set refを必ず含む。各Owner payloadは`LodOwnerPayloadHeaderV1`の二Fieldを持ち、型ごとに登録したASCII domain separatorと自己hashを除くcanonical payload bytesから`payload_content_hash`を計算する。Envelopeの`owner_payload_type_ref`はその型とschema versionへexact解決し、`owner_payload_content_hash`はpayloadの`payload_content_hash`とbyte equalityにする。payload内のexact refはすべて解決し、Profile間dependency graphはacyclicでなければならない。

tier bindingは`tier_id`順へstrict sortし、duplicate、Set内にないrepresentation ID、別Set refを拒否する。Owner payloadが`tier_metadata[]`または`tiers[]`を持つ場合、その`tier_id`集合はEnvelopeの`tier_bindings[].tier_id`とset equalityにし、同じrepresentation identityをpayloadへ複写しない。ProfileはTransition Rule、Selection Table、Policy、Plan、Receipt、Activation Bindingを参照しない。`policy_content_hash`はASCII `MIRAKAN_LOD_POLICY_SET_V1`と自己hashを除くcanonical bytesから計算する。`domain_profile_refs[]`は`lod_class, subject_scope_ref, profile_id, profile_version, profile_content_hash`順にstrict sortし、同一`{lod_class, subject_scope_ref}`のduplicateを拒否する。Intentの`allowed_lod_classes[]`とPolicyに存在する`lod_class`集合はset equalityとし、ProfileがないDomainをdefault推測しない。

以下の`MeshLodProfileV1`、`SpriteLodProfileV1`、`RepresentationLodProfileV1`、`HlodProfileV1`、`SimulationLodProfileV1`、`AnimationLodProfileV1`、`MaterialLodProfileV1`、`VfxLodProfileV1`、`TerrainLodProfileV1`、`FoliageLodProfileV1`、`WaterLodProfileV1`、`SnowSurfaceLodProfileV1`は、`LodDomainProfileV1<owner_payload_type>`のOwner-qualified composition名である。共通identity、version、hash、tier binding、Representation Set refを別Fieldとして二重定義しない。Owner payloadは自身のclosed field、unit、range、fallback metadataだけを持つ。ProfileはOwner固有のclosed typed payloadを持てるため、closed Core enumは表現手法を一種類へ固定しない。新しい表現は既存tagの自由文字列拡張ではなく、Owner-qualified payload type version、Capability、fallback、fixtureを追加して拡張する。

### 3.3 `LodResolutionPlanV1`

```text
LodResolutionPlanV1
  schema_version: 1
  plan_id: StableId
  plan_version: positive u32
  source_intent_ref: exact {intent_id, intent_version, intent_content_hash}
  source_policy_ref: exact {policy_set_id, policy_version, policy_content_hash}
  target_profile_ref: TargetProfileRefV1
  quality_profile_ref: QualityProfileRefV1
  capability_manifest_ref: exact {manifest_id, manifest_version, manifest_hash}
  selection_table_ref: LodSelectionTableRefV1
  domain_plans[1..32]: LodDomainResolutionPlanV1
  representation_set_refs[1..32]: LodRepresentationSetRefV1
  fallback_closure[0..256]: LodFallbackEdgeV1
  predicted_cost_before: owner-typed cost projection
  predicted_cost_after: owner-typed cost projection
  fidelity_checks[1..256]: owner-typed check result
  unresolved_requirements[0..256]: RequirementRefV1
  status: resolved | blocked
  compiler_ref: exact {compiler_id, compiler_version, compiler_hash}
  plan_hash: SHA-256

LodResolutionPlanRefV1
  plan_id: StableId
  plan_version: positive u32
  plan_hash: SHA-256

LodDomainResolutionPlanV1
  lod_class: LodClassV1
  domain_profile_ref: LodDomainProfileRefV1
  subject_scope_ref: exact owner-typed ref
  representation_set_ref: LodRepresentationSetRefV1
  owner_payload_type_ref: exact versioned type ref
  owner_payload_hash: SHA-256

LodFallbackEdgeV1
  from_representation_ref: LodRepresentationRefV1
  failure_classes[1..8]:
    unsupported_capability | missing_dependency | stale_generation
    | nonresident | transition_incompatible | target_unqualified
  to_representation_ref: LodRepresentationRefV1
```

`plan_hash`はASCII `MIRAKAN_LOD_RESOLUTION_PLAN_V1`と自己hashを除くReceipt-free canonical bytesから計算する。Qualification Receipt、Activation Binding、runtime pressure snapshot、runtime selectionをpreimageへ戻さない。`domain_plans[]`は`lod_class`とexact Domain Profile refを持つDomain固有typed payloadのtagged unionである。unknown tag／major、Target fallback欠損、Source ref不一致、`status=resolved`でunresolved requirementが1件以上あるPlanを拒否する。Quality ProfileはPresentation実装品質だけを変え、simulation contract、owner-typed authoritative instance数、Damageを変更できない。

```text
LodRepresentationSetV1
  schema_version: 1
  representation_set_id: StableId
  representation_set_version: positive u32
  source_identity_ref: exact owner-typed ref
  representations[1..32]: LodRepresentationDescriptorV1
  set_content_hash: SHA-256

LodRepresentationSetRefV1
  representation_set_id: StableId
  representation_set_version: positive u32
  set_content_hash: SHA-256

LodRepresentationRefV1
  representation_set_ref: LodRepresentationSetRefV1
  representation_id: StableId
  descriptor_content_hash: SHA-256

LodRepresentationDescriptorV1
  representation_id: StableId
  lod_class: LodClassV1
  source_identity_ref: exact owner-typed ref
  artifact_generation_ref: exact {artifact_id, generation, artifact_hash}
  quality_rank: u16
  geometric_error:
    {kind=bounded_m, maximum_object_error_m: finite non-negative f64}
    | {kind=not_applicable}
  semantic_error_bound_ref: exact owner-typed ref
  required_capability_refs[0..32]: sorted unique CapabilityRefV1
  estimated_cost_class_ref: exact owner-typed ref
  dependency_refs[0..64]: sorted unique exact refs
  transition_compatibility_refs[0..32]: sorted unique exact refs
  fallback_representation_id: StableId | null
  descriptor_content_hash: SHA-256
```

`fallback_representation_id`は同じSet内のplan-local IDで、外側`LodRepresentationSetRefV1`をdescriptorへ埋め戻さない。`transition_compatibility_refs[]`はSetより先に存在するOwner-owned semantic compatibility contractだけを参照し、`LodTransitionRuleV1`、本Set ref、Policy、Planを参照しない。各descriptor hashはASCII `MIRAKAN_LOD_REPRESENTATION_DESCRIPTOR_V1`、set hashはASCII `MIRAKAN_LOD_REPRESENTATION_SET_V1`と各自己hashを除くcanonical bytesから計算する。`quality_rank=0`をhighest fidelityとし、値が増えるほど低fidelityである。`representations[]`は`quality_rank, representation_id`順にstrict sortし、duplicate rankを拒否する。`maximum_object_error_m`を定義できないDomainは`not_applicable`を使い、0で偽装しない。`selection_mode=maximum_candidate_error`は全候補の`geometric_error.kind=bounded_m`を必須とし、別metric modeだけが`not_applicable`を受理する。costの共通測定単位、budget、capacity envelopeは[Runtime performance／capacity](../04-runtime/performance-capacity.md)を参照する。

候補setは同じsource semanticsとversion closureに属し、missing tier、duplicate rank、fallback cycle、incompatible dependency、stale artifactを拒否する。低tierがgameplay collision、navigation reachability、authoritative identityを暗黙に変えない。

LOD closureのhash依存順を`Intent／Preset fragment／Representation Set -> Domain Profile dependency DAG -> Transition Rule -> Selection Table -> resolved Policy -> Resolution Plan -> Qualification Receipt／Activation Binding`に固定する。Domain Profile間はOwner payloadのexact refに従ってtopological sortし、cycleを拒否する。同段の相互参照、後段refの前段preimageへの埋戻し、Receipt／BindingからPlanへのback-referenceを拒否する。CompilerはこのDAGと全exact refのbyte equalityを検証し、topological順を推測可能な命名やfile順に依存しない。

## 4. 共通選択契約

Presentation Resolver inputは[Render Graph](render-graph.md#7-visibilityとgeometry-execution)が公開するexact conservative `ViewLodCandidateSetRefV1`、subject Stable ID、exact representation set ref、`ViewLodContextV1`、bounds、semantic priority、Target／Quality、[Performance Ownerのruntime pressure snapshot](../04-runtime/performance-capacity.md#52-lod-budget-pressure-snapshot)、previous selectionを含む。candidate setはSource boundsとclosed view frustum／layerだけで作り、selected tierまたはocclusionを入力にしない。set外subjectはView selection／residency requestを行わずprevious historyをretire可能状態で保持するが、`hidden_presentation`選択、World unload、Simulation relevancyを意味しない。Render Graphのocclusion result、GPU visibility、frame completion順はselection入力にせず、presentation candidate requestの優先度hintとして使う場合もPlan結果と区別する。

```text
ViewLodContextV1
  schema_version: 1
  view_id: StableId
  view_generation: positive u64
  view_purpose: game | editor | shadow | reflection | thumbnail
  projection:
    perspective {vertical_fov_rad, near_m, far_m}
    | orthographic {vertical_span_m, near_m, far_m}
  view_from_world: finite Matrix4x4
  render_width_px: positive u32
  render_height_px: positive u32
  camera_cut: bool
  history_reset: bool
  target_profile_ref: TargetProfileRefV1
  quality_profile_ref: QualityProfileRefV1
  pressure_snapshot_ref: LodBudgetPressureSnapshotRefV1 | null
  context_hash: SHA-256
```

この型は[Render Graph](render-graph.md)が公開する選択済み`RenderViewV1`からLOD Ownerが作るread-only projectionである。`view_id`、`view_generation`、purpose、near／far、Target／QualityはSource Viewとbyte equalityにし、`physical_perspective`は解決済みvertical FOVを持つ`perspective`、`pixel_orthographic`は解決済みvertical spanを持つ`orthographic`へ一意に投影する。`far_m`はfiniteかつ`far_m > near_m`で、candidate setのclosed frustum外を選択対象へ戻さない。Source Camera Profile、Lens、aspect、Post Processを複写せず、`context_hash`はASCII `MIRAKAN_VIEW_LOD_CONTEXT_V1`と自己hashを除くcanonical bytesから計算する。Editor／shadow／reflection／thumbnail Viewは独立したselection／history stateを持つ。

`algorithm.lod.projected_metric.v1`を次に固定する。入力は同じView generationのcandidate setに含まれるView space conservative bounding sphere center `c=(x,y,z)`、radius `r_m >= 0`、候補descriptorの`geometric_error={kind=bounded_m, maximum_object_error_m=e_m >= 0}`で、Math正本どおりCamera forwardは`-Z`とする。`d_m=-z`、`nearest_depth_m=max(near_m, d_m-r_m)`とし、Perspectiveでは`pixel_scale=render_height_px/(2*tan(vertical_fov_rad/2))`、Orthographicでは`pixel_scale=render_height_px/vertical_span_m`とする。

```text
projected_error_px    = e_m * pixel_scale / nearest_depth_m   // perspective
projected_coverage_px = 2 * r_m * pixel_scale / nearest_depth_m

projected_error_px    = e_m * pixel_scale                     // orthographic
projected_coverage_px = 2 * r_m * pixel_scale

projected_error_px_q16    = saturating_u32(ceil(max(0, projected_error_px) * 65536))
projected_coverage_px_q16 = saturating_u32(ceil(max(0, projected_coverage_px) * 65536))
distance_mm_u64           = saturating_u64(ceil(max(0, d_m-r_m) * 1000))
```

Sphereがnear planeへ接触／交差する`d_m-r_m <= near_m`ではprojection variantにかかわらずerror／coverageを`UINT32_MAX`へsaturateし、最詳細側へ保守的に倒す。view transform、bounds、FOV／span、near、extent、計算途中のいずれかがnon-finite、範囲外、overflow-before-saturationならContextまたはcandidateを拒否する。`ceil`は誤差を過小評価しないために固定し、CPU／GPU、Preview／Runtimeは同じNamed Algorithm、canonical fixture、inclusive比較を使う。

共通metric IDと用途を次に固定する。

| Metric | 用途 | 禁止用途 |
|---|---|---|
| `projected_error_px_q16` | Mesh／surface geometry detail | Simulation |
| `projected_coverage_px_q16` | Representation、VFX、Material presentation | Gameplay relevancy |
| `distance_mm_u64` | HLOD cell／manual visibility range、bounded surface | FOV／解像度依存のMesh品質判定 |
| `gameplay_relevance_q16` | Simulation Ownerが公開するSimulation tier入力 | Rendering、visibility、occlusion |
| `budget_pressure_class` | exact Selection Rowの選択 | Gameplay fidelity floorの緩和、数値thresholdへの暗黙倍率 |

`gameplay_relevance_q16`のNamed Algorithm、入力、量子化、authorityはSimulation Domain Profileが所有し、LODがdistance、visibility、Genre名から生成しない。Cooked thresholdは整数へ量子化し、CPU／GPUは同じ比較方向、境界包含、fixtureを使う。NaN、Inf、負値、非単調thresholdを受理しない。

```text
LodSelectionTableV1
  schema_version: 1
  table_id: StableId
  table_version: positive u32
  rows[1..4096]: LodSelectionRowV1
  table_content_hash: SHA-256

LodSelectionTableRefV1
  table_id: StableId
  table_version: positive u32
  table_content_hash: SHA-256

LodSelectionRowV1
  row_id: StableId
  key:
    lod_class: LodClassV1
    subject_scope_ref: exact owner-typed ref
    semantic_priority: LodSemanticPriorityV1
    view_purpose: game | editor | shadow | reflection | thumbnail | not_applicable
    target_profile_ref: TargetProfileRefV1
    quality_profile_ref: QualityProfileRefV1
    budget_pressure_class: normal | elevated | severe | critical
  selection_mode:
    maximum_candidate_error {
      metric_id: projected_error_px_q16
      maximum_error_inclusive_q16: u32
    }
    | metric_band {
      metric_id: projected_coverage_px_q16 | distance_mm_u64
                 | gameplay_relevance_q16
      bands[1..32]: exact non-overlapping full-domain interval -> LodTierRefV1
    }
  ordered_tier_refs[1..32]: lowest-cost acceptable LodTierRefV1 order
  transition_rule_refs[0..64]: exact LodTransitionRuleRefV1
```

Rowはcanonical key bytes、`row_id`順へstrict sortし、key duplicateを拒否する。`table_content_hash`はASCII `MIRAKAN_LOD_SELECTION_TABLE_V1`と自己hashを除くcanonical bytesから計算する。Row keyは全fieldのbyte equalityでexact一件へ解決し、0件／複数件を拒否する。`maximum_candidate_error`は`ordered_tier_refs[]`を先頭から走査し、minimum semantics、Capability、dependency、generationがvalidかつ候補固有のprojected errorが`maximum_error_inclusive_q16`以下となる最初の候補を選ぶ。`metric_band`はfull domainを隙間／重複なく覆う一つのinclusive intervalからtierを選ぶ。`projected_coverage_px_q16`／`gameplay_relevance_q16`はmetric増加時に`quality_rank`を増加させず、`distance_mm_u64`はmetric増加時にrankを減少させない。これに反するbandは非単調として拒否する。`metric_band` branchの`ordered_tier_refs[]`はbandが参照するtierのexact setと一致し、resident fallbackの評価順にも使う。importance、Target／Quality、pressureはRow選択へ明示され、補間、名前fallback、隠れた倍率を使わない。

previous selectionは候補がなおvalidで、対応transition ruleのexit条件とminimum residencyが保持を許す場合だけ優先する。選択候補がnonresidentなら同一generationの明示fallback closureを順に評価する。exact Rowまたは意味同等resident fallbackがない場合は、previous valid tier、次にhighest-fidelity valid tierを維持してDiagnosticを返し、表示を黙って消さない。`missing_pressure_policy`によりpressure refがnull／stale／Target・Quality不一致の場合も同じ保守的経路を使い、`normal`や`critical`を捏造しない。入力集合がunorderedである箇所だけStable ID byte順を最終tie-breakにし、worker completion順、frame timeの単一sample、hash-map iteration順を使わない。

`LodTransitionRuleRefV1`は`{rule_id: StableId, rule_version: positive u32, rule_content_hash: SHA-256}`である。`LodTransitionRuleV1`は同三Field、`from_tier_ref／to_tier_ref: LodTierRefV1`、`metric_id`、`enter_threshold_inclusive`、`exit_threshold_inclusive`、`minimum_residency`、`transition_mode`、`transition_extent`、`camera_cut_policy`、`missing_artifact_policy`を持つ。thresholdの型はQ16 metric=`u32`、distance=`u64`でmetric IDと一致させる。`minimum_residency`はclosed tagged unionで、Presentation branch=`{kind=presentation_frames, count: positive u32}`、Simulation branch=`{kind=simulation_advances, count: positive u32, cadence_profile_ref: SimulationCadenceProfileRefV1}`とする。enter／exitを別に持ち境界往復を防ぐ。`camera_cut_policy`はPresentationの`reset_and_reselect`またはSimulationの`not_applicable`、`missing_artifact_policy`は`use_exact_fallback_closure | preserve_previous_and_report | block_selection`のclosed enumである。`rule_content_hash`はASCII `MIRAKAN_LOD_TRANSITION_RULE_V1`と自己hashを除くcanonical bytesから計算する。

`transition_extent`は`instant -> {kind=none}`、`dither | cross_fade -> {kind=metric_band, begin_inclusive:metric-typed integer, end_inclusive:metric-typed integer} | {kind=presentation_frames,count:positive u32}`、`domain_blend -> {kind=domain_handoff, handoff_contract_ref:exact owner-typed ref}`だけを許可する。別modeのbranch、zero幅、逆順bandを拒否する。Presentation frameとSimulation Advanceを同じunitとして比較せず、Simulation branchのProfile refはRuntime選択値およびSave／Replay closureとbyte equalityにする。Cadence rate、wall time、Genreからcountを換算しない。未対応BackendではProfile登録済みの意味同等fallbackを使う。camera cut、projection variant／dynamic extent変更では古いPresentation historyを捨て、そのframeに必要なdetailへ即時再選択し、無効historyを理由に低detailを選ばない。Simulation transitionへcamera cutを入力しない。

## 5. Mesh／Sprite geometry LOD

`MeshLodProfileV1 = LodDomainProfileV1<MeshLodProfilePayloadV1>`とし、payloadは`selection_metric = projected_error_px_q16`、`quality_binding_refs[]`、`skin_compatibility_ref`、`section_compatibility_ref`、`shadow_policy_ref`、plan-local `fallback_geometry_id`を持つ。Transition RuleはProfile hash完成後に`LodTierRefV1`へ結ぶため、ProfileへRule refを埋め戻さず`LodSelectionTableV1`だけが参照する。Source chain／generated chain／hybrid、triangle ratio、boundary／UV seam／normal／vertex-color／skin保持、simplifier Toolは[Asset lifecycle](../03-authoring/asset-lifecycle.md)の`MeshImportSettingsV1`と`MeshLodGenerationProfileRefV1`だけが所有し、本Profileへ複写しない。

`MeshLodLevelV1`は`level_index: u8[0..15]`、exact `representation_ref`、`maximum_projected_error_px_q16`、`required_material_interface_hash`、`expected_triangle_count: u64`、`artifact_role`を持つ。levelは生成指示でなく、Assetが公開済みartifactへ付与した選択metadataである。LOD0は最高detail Source representationへexact解決し、欠損または別Source generationを拒否する。

Mesh／Sprite representationはgeometry artifact ref、bounds／silhouette error、vertex／primitive cost class、material interface、skin compatibility、shadow／collision proxy relationを宣言する。initial V1、C1、C2はMorph Targetを非対象とし、`morph_compatibility_ref`、empty placeholder、zero-weight representationまたはMorph対応表示を作らない。将来Morph Capabilityを採択する場合だけ[Asset lifecycle](../03-authoring/asset-lifecycle.md)のend-to-end closureを満たす新しいversioned LOD payloadで追加する。LODはgeometry candidateを選び、meshlet／indirect draw／occlusionの実行は[Render Graph](render-graph.md)へ委譲する。

silhouette、UV、normal／tangent、skin weight、sprite pivot／pixel lockに意味差があるtierは明示する。missing material interfaceやanimation bindingをdefaultへ置換せず、compatible fallbackへ戻す。

`SpriteLodProfileV1 = LodDomainProfileV1<SpriteLodProfilePayloadV1>`とし、payloadの`selection_kind`は`source_variant | visibility_only | disabled`のclosed enumである。Pixel artのpoint sampling／integer scale／palette／alpha padding／pixel-lockedをQualityより優先し、Gameplay cueは意味同等variantなしに消さず、atlas page／texture mip／Sprite LODを同一indexにせず、stable pivot／bounds／sort／collision非依存を検証する。

本節のdiscrete tier chainは、micro-cluster hierarchyをViewごとに細分化するinner continuous cut、virtual page residency、Backend固有virtualized meshを含まない。[Virtualized／Continuous Geometry](virtualized-continuous-geometry.md)がactiveとなるTargetでも本chainをexact fallbackとして必須にし、outer Resolverだけがvirtualized familyとdiscrete representationを切り替える。Future Capabilityの採用前にcontinuous path用field、空Artifact、Backend名だけのcapability flagを追加しない。

## 6. Representation LODとHLOD

`RepresentationLodProfileV1 = LodDomainProfileV1<RepresentationLodProfilePayloadV1>`とする。payloadの`tier_metadata[1..32]`は`tier_id`、`representation_mode: individual | instanced | spatial_proxy | impostor | hidden_presentation`、exact `identity_preservation_contract_ref`だけを持つ。`hidden_presentation`はPresentationだけを隠し、Entity、Collision、Damage、Navigation、Save stateを削除しない。individualからinstancedへ変えてもStable Entity identityとauthoritative event routingを保持する。

`HlodProfileV1 = LodDomainProfileV1<HlodProfilePayloadV1>`とし、payloadはexact `eligibility_rule_ref`、exact `spatial_partition_profile_ref`、exact `cluster_limits_ref`、`proxy_mode: instanced | merged_mesh | simplified_mesh | impostor`、exact `proxy_geometry_profile_ref`、exact `proxy_material_profile_ref`、exact `streaming_cell_profile_ref`、plan-local `fallback_representation_id`だけを持つ。fallback IDはEnvelopeのRepresentation Set内へ解決する。Transition RuleはProfileへ埋め戻さずSelection Tableが参照する。対象はstatic transform、`decorative_instance`、mutable Physics／Damage／interaction／Save／animation／root motionなし、Style／Material許可、bounded plan-local Cell descriptorとcluster／geometry／material／texture capacity内をすべて満たす。

Clusterは`WorldStreamingPlanV1`のexact ref、plan-local `cell_id`／`representation_slot_id`、Profile ref、material compatibility、Source Stable ID順から決定論的に生成する。`HlodArtifactV1`はSource Stable ID集合、Source revision hash、bounds、proxy method、geometry／material key、visual error、exact `{world_streaming_plan_ref, cell_id, representation_slot_id}`、fallbackを持つ。Source、transform、Material、Profile変更時に該当Artifactだけinvalidateする。HLODをEntity identity、Save owner、Physics／Navigation objectの代替にしない。

[World](world.md#6-spatial-partitionとstreaming-plan-authoring)の`WorldStreamingPlanV1`はplan-local Cell descriptor、representation slot、residency dependency slotまでを記述し、HLOD artifactそのものをPlan hashへ戻さない。生成順は`receipt-free World Streaming Plan -> exact Plan ref -> HLOD Artifact`の一方向とする。LOD selectionはresident candidateから選ぶ。必要candidateが非residentの場合のrequest／backpressureはWorldとRuntime ownerへ返し、unbounded blocking loadを起こさない。

## 7. Simulation LOD境界

Simulation representationはfull、reduced、dormant等のDomain-defined behavior candidate refとsemantic guaranteeを公開できる。LODは候補を選択するが、Physics integration、Collision response、Navigation query、authoritative writer、wake conditionの意味は各Simulation Ownerが所有する。`SimulationLodProfileV1 = LodDomainProfileV1<SimulationLodProfilePayloadV1>`とし、payloadはexact `contract_ref: SimulationLodContractRefV1`だけを持つ。現行の[Physics](../05-simulation/physics.md)、[Collision](../05-simulation/collision.md)、[Navigation](../05-simulation/navigation.md)は`full`以外のqualified candidateを宣言していないため、距離／visibility／pressureからreduced／dormantを推測してはならない。

`SimulationLodContractRefV1`は`{contract_id: StableId, contract_version: positive u32, contract_content_hash: SHA-256}`である。これはDomain Profileと別identityを持つSimulation Owner契約であり、`LodDomainProfileV1`の別名ではない。`SimulationLodContractV1`は同三Field、exact `experience_role_ref`、`tiers[1..32]: SimulationLodCandidateDescriptorV1`、exact `relevance_algorithm_ref`、plan-local `transition_eligibility_specs[0..64]`、sorted unique exact `wake_trigger_refs[0..64]`、exact `retained_state_schema_ref`、exact `queued_event_policy_ref`、exact `authoritative_equivalence_contract_ref`、sorted unique exact `forbidden_change_refs[0..64]`、exact `reference_fixture_ref`を持ち、hashはASCII `MIRAKAN_SIMULATION_LOD_CONTRACT_V1`と自己hashを除くcanonical bytesから計算する。Contract完成後のexact Transition RuleはSelection Tableだけが参照し、ContractへRule refを埋め戻さない。各`SimulationLodCandidateDescriptorV1`はplan-local `candidate_id`、`descriptor_content_hash`、exact Domain owner ref、exact semantic guarantee ref、exact retained state projection schema ref、exact wake condition ref、exact handoff contract ref、exact equivalence evidence requirement ref、plan-local fallback candidate IDを必須とする。Profile Envelopeのtier ID集合、Contractのcandidate ID集合、Representation Setのbehavior representation ID集合は一対一対応を検証する。外部`SimulationLodCandidateRefV1`は完成後の`{contract_ref, candidate_id, descriptor_content_hash}`であり、Contract hashをdescriptorへ埋め戻さない。full未満のcandidateはDomain OwnerがこれらをmaterializeしてQualificationするまで候補集合に入らない。Target／QualityはこのGameplay契約を自動的に強めたり弱めたりしない。

render tierとsimulation tierを同一indexで結ばない。off-screenでもauthoritative gameplayに必要なsimulationを停止せず、visibleでもcapacity／Targetが許さないsimulation featureを暗黙有効化しない。tier handoffはpublished state、generation、handoff resultを持つ。

```text
SimulationLodSaveProjectionV1
  schema_version: 1
  projection_id: StableId
  projection_version: positive u32
  owner_ref: exact Simulation Domain owner ref
  contract_ref: exact SimulationLodContractRefV1
  records[0..1048576]:
    subject_persistent_ref: exact owner-typed ref
    retained_state_projection_ref: exact owner-typed ref
    queued_event_projection_ref: exact owner-typed ref
    wake_condition_state_ref: exact owner-typed ref
    last_committed_tier_ref: LodTierRefV1
    handoff_generation: u64
  projection_content_hash: SHA-256
```

recordsは`subject_persistent_ref`のcanonical bytes順へstrict sortしduplicateを拒否する。`projection_content_hash`はASCII `MIRAKAN_SIMULATION_LOD_SAVE_PROJECTION_V1`と自己hashを除くcanonical bytesから計算する。`last_committed_tier_ref`はdiagnostic／handoff検証用で、Load後のtier強制指定ではない。SaveはView、distance、occlusion、pressure snapshot、resident handleを含めず、[Persistence／Save](../04-runtime/persistence-save.md#4-reconstruction)のDomain Projection Bindingへ本Projectionをexact membershipとして結ぶ。Loadはretained state／queued event／wake conditionをDomain Ownerが検証し、fullまたは明示last-valid semantic stateでpublicationした後、fresh Contextから再選択する。復元不能ならfull fallback、fullも復元不能ならactivationを拒否する。

## 8. Animation、Material、VFXとの境界

Animationはclip／pose／skin candidateとminimum event／root-motion semanticsを公開し、LODはrepresentationを選ぶ。event、root motion、IK、retargetの意味は[Animation](../05-simulation/animation.md)を参照し、低tierへの遷移でauthoritative eventを欠落させない。

`AnimationLodProfileV1 = LodDomainProfileV1<AnimationLodProfilePayloadV1>`とし、payloadの`tiers[1..32]: AnimationLodTierV1`をEnvelopeのtier bindingへ結ぶ。各tierは`tier_id`、`presentation_pose_sample_interval_frames: positive u32`、exact `interpolation_profile_ref`、exact `presentation_bone_set_ref`、exact `skinning_profile_ref`、exact `shadow_pose_profile_ref`、sorted unique exact `required_bone_refs[0..256]`、sorted unique exact `required_event_refs[0..256]`を持つ。`LodPolicySetV1`はclass=`animation_presentation`のfull exact `LodDomainProfileRefV1`で参照する。このintervalはPresentation render frameだけを数え、Simulation Advance、Animation clip timebase tick、秒へ換算しない。未登録profile ref、required bone／eventを除外するProfileを拒否し、authoritative state machine／root motion／hitbox／authoritative attachment socket／foot contact／event timingを低頻度poseから取得しない。

Materialは各tierのcompatible artifactとfeature reductionを[Materials](materials.md)で宣言する。LODはtierを選ぶがshader variantやparameter意味を再定義しない。

[Materialsが所有する`MaterialLodProfileV1 = LodDomainProfileV1<MaterialLodProfilePayloadV1>`](materials.md#7-material-lod境界)をclass=`material_detail`のfull exact `LodDomainProfileRefV1`で消費し、Owner payload Fieldを再定義しない。Material Sourceが使うnarrow `MaterialLodProfileRefV1={profile_id, profile_version, profile_content_hash}`はfull refの同三Fieldとbyte equalityにし、`lod_class`と`subject_scope_ref`をIDや表示名から補完しない。選択tierの`material_artifact_generation_ref`、`material_interface_hash`、Target／Quality refはgeometry representation／Render packetとbyte equalityにする。Style-critical ramp、alpha semantics、pixel sampling、critical-cue emissiveを削除しない。

`VfxLodProfileV1 = LodDomainProfileV1<VfxLodProfilePayloadV1>`とし、payloadの`tiers[1..32]: VfxLodTierV1`をEnvelopeのtier bindingへ結ぶ。VFX Source／Artifactのnarrow `VfxLodProfileRefV1={profile_id, profile_version, profile_content_hash}`はclass=`vfx_presentation`のfull Domain refの同三Fieldとbyte equalityにする。各tierは`tier_id`、`semantic_priority: LodSemanticPriorityV1`、`branch_id: StableId`、`spawn_scale_permille: u16[0..1000]`、`maximum_alive: u32`、`update_interval={kind=simulation_advances,count: positive u32,cadence_profile_ref: SimulationCadenceProfileRefV1}`、sorted unique `renderer_output_role_ids[0..4]`、`simulation_target: cpu | gpu`、exact `minimum_cue_contract_ref`を持つ。Payload hashはASCII `MIRAKAN_VFX_LOD_PROFILE_PAYLOAD_V1`と自己hashを除くcanonical bytesから計算し、Envelope hashは§3.2の共通式を使う。Quality Profile bindingがLOD Profile refを持つ一方向とし、LOD ProfileからQuality refを埋め戻さない。VFX Quality binding、System Source、System Manifest、Emitter Artifact、LOD Planが参照するQuality／LOD refはbyte equalityにする。

intervalは選択ProfileのSimulation Advance数だけを意味し、render frame、秒、表示Hzへ変換しない。current `fixed_rational_advance_v1` VFX artifactは`count=1`だけを受理し、`count>1`はmulti-advance integrationを定義する新Artifact versionとCadence別Qualificationがactiveになるまで`vfx_lod_update_interval_not_qualified`で拒否する。`critical_gameplay_cue`はshape／timing／minimum visibilityを維持し、ambient effectより先にdropしない。VFX tierはGameplay event数／Damage／Collision／AI perceptionを変更しない。

## 9. Terrain、Foliage、Water／Surface、residency

Terrain、foliage、water、snow／surfaceはDomain Ownerがtile／patch／cluster representation候補、seam constraint、interaction guaranteeを公開し、LODは選択だけを行う。隣接tierのseam、crack、normal／material continuity、interaction proxyの互換性をdescriptorで検証する。

`TerrainLodProfileV1 = LodDomainProfileV1<TerrainLodProfilePayloadV1>`とし、payloadの`tier_metadata[1..32]`は`tier_id`、`maximum_projected_error_px_q16: u32`、exact `patch_bounds_profile_ref`、exact `seam_profile_ref`、exact `material_residency_floor_ref`、exact `streaming_cell_role_ref`を持ち、payload全体に`neighbor_rank_delta_max: u8[0..31]`とexact `gameplay_surface_invariance_contract_ref`を持つ。`FoliageLodProfileV1 = LodDomainProfileV1<FoliageLodProfilePayloadV1>`とし、payloadの`tier_metadata[1..32]`は`tier_id`、exact `wind_profile_ref`、exact `shadow_profile_ref`、nullable exact `impostor_profile_ref`、exact `cluster_bounds_profile_ref`、`maximum_instances_per_cell: positive u32`を持ち、payload全体にexact `gameplay_collision_invariance_contract_ref`を持つ。render patchはCollision height／Nav tile／Gameplay Surface Stateを置き換えず、Gameplay Collision subsetを描画LODへ追従させない。

Terrain／FoliageのSource／artifact／identityは[Terrain／Foliage](terrain-foliage.md)が所有し、本書は同Ownerが公開するrepresentation candidateとLOD Profileだけを消費する。`future.capability.production-terrain`と`future.capability.production-foliage`は独立した`planning_only` entryであり、上記Schema名、Owner link、候補metadataはActive Capability、Operation、Provider、Cook artifact、Runtime candidate、Qualification Receiptの存在を意味しない。Terrain、Foliage、GIまたは別domain supportを一つの複合Capability IDから推測する経路を定義しない。

[Environment／surfacesが所有する`WaterLodProfileV1`と`SnowSurfaceLodProfileV1`](environment-surfaces.md)は同文書のclosed payload Schemaを使う。LODはEnvelopeのtier bindingを選ぶだけで、CPU Surface Query／Water Volume／浮力／swimming／Damage／Navigation cost、Gameplay Surface State／friction／movement／foot contact／static coverageを変更しない。降雪particle密度の倍率は`vfx_presentation` classの`VfxLodProfileV1`だけが所有し、`SnowfallVfxProfileV1`の`density_scale`はauthored基準値である。

residency requestはartifact generation、priority intent、deadline class、fallback candidateを持つが、queue、memory reservation、backpressure値を本書で決めない。候補のresidencyが失われた場合はgenerationを再検証し、stale GPU／streaming handleを再利用しない。

`GeometryResidencyLodSnapshotV1`は`snapshot_id`、positive `snapshot_generation`、exact `representation_set_ref`、sorted unique `requested_representation_refs[]`／`resident_representation_refs[]`／`pending_representation_refs[]`、nullable exact `selected_fallback_ref`、`estimated_byte_cost:u64`、exact `deadline_class_ref`、`owner_generation:u64`、`snapshot_content_hash`を持つ。hashはASCII `MIRAKAN_GEOMETRY_RESIDENCY_LOD_SNAPSHOT_V1`と自己hashを除くcanonical bytesから計算する。三集合は同じRepresentation Setへ解決し、resident／pendingの交差を拒否する。要求detailが非residentなら同じAsset generation内でFallback closure上の最初の意味同等resident representationを使い、別generationをframe内で混在させない。queue、reservation、実測byte authorityはRuntime／Asset Ownerにあり、本Snapshotの予測値から再定義しない。

## 10. Preset、AI contract、Editor UX

LOD Preset定義はOwner-qualified contributionとしてRegistryに保持し、Projectはexact `LodPolicyPresetSelectionV1`と解決済みPolicyだけをProject-owned assetへ保存する。Preset名から数値やtierを推測せず、resolved policyをPreviewで表示する。

Preset vocabularyはCore closed enumにせず、次のOwner-qualified Registryへ合成する。

```text
LodPolicyPresetRefV1
  preset_id: namespace付きStableId
  preset_version: positive uint32
  preset_content_hash: SHA-256

LodPolicyPresetV1
  preset_ref: LodPolicyPresetRefV1
  owner_ref: exact {owner_id, owner_revision, owner_content_hash}
  contribution_layer: core | feature_pack | genre_pack | project
  semantic_description
  applicability_schema_ref
  policy_fragment_ref
  required_capability_refs[]
  target_support_refs[]

LodPolicyPresetRegistryV1
  registry_id: registry.lod.policy_preset
  registry_version
  presets[1..1024]
  registry_content_hash: SHA-256

LodPolicyPresetActivationProjectionV1
  projection_id
  projection_version: positive uint32
  registry_ref: exact {registry_id, registry_version, registry_content_hash}
  selected_preset_refs[1..1024]: LodPolicyPresetRefV1
  qualification_binding_refs[1..1024]:
    exact {binding_id, binding_version, binding_content_hash}
  projection_content_hash: SHA-256

LodPolicyPresetSelectionV1
  registry_ref: exact {registry_id, registry_version, registry_content_hash}
  activation_projection_ref:
    exact {projection_id, projection_version, projection_content_hash}
  preset_ref: LodPolicyPresetRefV1
  preset_qualification_binding_ref:
    exact {binding_id, binding_version, binding_content_hash}

LodPolicyPresetProvenanceRefV1
  registry_ref: exact {registry_id, registry_version, registry_content_hash}
  preset_ref: LodPolicyPresetRefV1
```

`preset_content_hash`はASCII `MIRAKAN_LOD_POLICY_PRESET_V1`と自己hashを除くReceipt-free Preset canonical bytes、Registry hashはASCII `MIRAKAN_LOD_POLICY_PRESET_REGISTRY_V1`、Registry ID／version、Preset ID／version順の全Preset canonical bytesから計算する。`policy_fragment_ref`はPreset／Registry／resolved Policy refを持たないReceipt-free template fragmentへだけ解決する。selected refsはPreset ID／version／hash順、Binding refsは解決したsubject refの同じ順にstrict sortし、duplicateを拒否する。Activation Projectionのselected ref集合とQualification Bindingが解決する合格かつfreshなsubject集合はexact set equalityで、Receipt／BindingをPreset／Registry hashへ戻さない。Resolved Policyは`LodPolicyPresetProvenanceRefV1`だけをReceipt-free provenanceとしてhashへ含め、Activation Projection／Qualification BindingをPolicy／Plan preimageへ戻さない。Project-owned `LodPolicyPresetSelectionV1`とPreviewが外側でfreshness／qualificationを検証する。Feature／Genre contributionは所有Pack identity、Project contributionはProject owner identityへexact解決する。自己namespace外ID、Coreまたは他Owner entryの上書き、duplicate、unknown、stale owner／version／hash、unqualified entry、Target／Capability不成立をfail closedにする。Registry materializationはProject／Pack dependency closureを解決したCompilerが行い、Generic Engine CoreからPackへのdependency edgeを生成しない。

initial V1のCore required defaultはGenre／object classを仮定しない`lod.preset.core.primary_subject@1 | lod.preset.core.interactive_subject@1 | lod.preset.core.supporting_subject@1 | lod.preset.core.decorative_subject@1 | lod.preset.core.critical_presentation_cue@1 | lod.preset.core.ambient_presentation@1`のexact六件である。次のexact contributionはinitial optional catalogであり、各logical ownerとresolved policyをcanonical fixtureで固定する。

| semantic label | canonical `preset_id@version` | `contribution_layer`／logical owner |
|---|---|---|
| `hero_character` | `lod.preset.hero_character@1` | `feature_pack`／`feature.character_locomotion@1` |
| `interactive_character` | `lod.preset.interactive_character@1` | `feature_pack`／`feature.character_locomotion@1` |
| `crowd_character` | `lod.preset.crowd_character@1` | `feature_pack`／`feature.character_locomotion@1` |
| `interactive_prop` | `lod.preset.interactive_prop@1` | `feature_pack`／`feature.interaction@1` |
| `small_decorative_prop` | `lod.preset.small_decorative_prop@1` | `core`／`mirakan.arch.rendering-lod` |
| `architecture` | `lod.preset.architecture@1` | `core`／`mirakan.arch.rendering-lod` |
| `foliage` | `lod.preset.foliage@1` | `core`／`mirakan.arch.rendering-lod` |
| `terrain` | `lod.preset.terrain@1` | `core`／`mirakan.arch.rendering-lod` |
| `water_surface` | `lod.preset.water_surface@1` | `core`／`mirakan.arch.rendering-lod` |
| `vfx_combat_cue` | `lod.preset.vfx_combat_cue@1` | `feature_pack`／`feature.combat@1` |
| `vfx_ambient` | `lod.preset.vfx_ambient@1` | `core`／`mirakan.arch.rendering-lod` |
| `pixel_art_sprite` | `lod.preset.pixel_art_sprite@1` | `core`／`mirakan.arch.rendering-lod` |

表のlogical ownerはmaterialization時にversion／content hashを含むexact `owner_ref`またはPack identityへ解決する。Core entryはCore required六件とは別のoptional default、Feature entryは該当Packを選択したProjectだけのcandidateである。いずれも全Projectの暗黙default、名前fallbackにはせず、該当Contributionを選択しないProjectでは候補に現れない。これによりboard／puzzle／simulation／tool ProjectがCharacter、Crowd、Combat vocabularyへ依存せず、必要なPackだけが固有Presetを追加できる。

initial V1 Sourceは完全な`LodPolicyPresetRefV1`だけを受理する。表のsemantic label、suffix、display nameまたはsemantic roleをPreset ref、aliasまたは自動選択keyとして定義しない。

```text
LodAuthoringContextV1
  schema_version: 1
  context_id: StableId
  project_ref: exact {project_id, project_revision, document_set_hash}
  contract_set_ref: ContractSetRefV1
  subjects[1..256]:
    subject_ref: exact owner-typed ref
    intent_ref: exact LodIntentRefV1
    policy_ref: exact LodPolicySetRefV1 | null
    representation_set_refs[0..32]: LodRepresentationSetRefV1
  target_profile_refs[1..16]: TargetProfileRefV1
  quality_profile_refs[1..16]: QualityProfileRefV1
  view_fixture_refs[0..32]: exact View fixture refs
  pressure_fixture_refs[0..16]: exact Pressure fixture refs
  available_operation_ids[0..64]: registered OperationId
  field_mask: registered bounded field set
  omitted_subject_count: u32
  continuation_cursor: opaque | null
  invalidation_conditions[1..32]: registered condition refs
  context_content_hash: SHA-256

LodPlanPreviewV1
  schema_version: 1
  preview_id: StableId
  context_ref: exact {context_id, context_content_hash}
  candidate_plan_ref: exact LodResolutionPlanRefV1
  subject_results[1..256]: owner-typed bounded preview results
  aggregate_before_after: owner-typed cost projections
  risk_refs[0..256]: exact risk refs
  blocking_diagnostic_refs[0..256]: DiagnosticCodeRefV1
  required_approval_refs[0..64]: exact approval refs
  completeness: complete | partial
  omitted_ranges[0..64]: typed range descriptors
  preview_content_hash: SHA-256
```

Context／Preview hashはそれぞれASCII `MIRAKAN_LOD_AUTHORING_CONTEXT_V1`／`MIRAKAN_LOD_PLAN_PREVIEW_V1`と自己hashを除くcanonical bytesから計算する。Contextはbounded read-only projectionで、Project revision、Contract set、Field mask対象、fixture、selection contextのいずれかが変わればstaleにする。partial Contextを完全なProject closureと表示せず、cursor、omitted count、invalidation conditionを必ず返す。AIへnative pointer、raw GPU handle、全Project dump、live mutable stateを渡さない。

`subject_results[]`は対象Stable ID／semantic role、exact Preset ref／owner／qualification、変更する／しないLOD class、Target／Quality／View／pressure別tier／integer threshold／fallback、triangle／draw／instance／memory／overdraw／simulation workのBefore／After、visual／silhouette／animation／critical-cue risk、Gameplay fidelity影響、Artifact／tool version／予測・実測区分を示す。表示名、suffix、semantic roleからPresetを推測しない。authorはexact Profile、tier順、threshold、transition、fallback、Target／Quality bindingを明示的に構成できる一方、AIはContext外のProfile、未登録operation、欠損値を補完しない。

上記Registry、Projection、Context／Preview、Core／compatibility default、extension fixtureはtarget contractであり、実装済み、active Gateway surface、Production Qualification済みという主張ではない。

LODのcreate／update policy、bind representation、apply preset、set importance、preview selection、explain transition、validate closureは将来のDomain action vocabularyであり、現在のOperation登録ではない。

`operation.lod.propose_policy`と`operation.lod.apply_policy`は[Executable contracts](../02-foundation/executable-contracts.md#211-既存domain文書から回収した未登録operation候補)の`planning.operation_family.lod_authoring@1`に属するexact二候補である。Capability stateは`not_activated`、current MCD／Owner Manifest／Service allowlist／Policy／Validator／Diagnostic／Receipt／Provider／MCP／generated alias／legacy alias集合はすべて`[]`である。前者はActivation後にだけPreset selectionを含むProposalを返す予定で、現在のselection権限ではない。Preset contribution自体のauthoring／activationは別計画語彙`planning.lod.policy_preset_contribution_authoring@1`とし、reserved Operation ID集合`[]`、current MCD／Owner Manifest／Service allowlist／Policy／Validator／Diagnostic／Receipt／Provider／MCP／CLI／Editor／generated alias／legacy alias集合もexact `[]`、Capability stateも`not_activated`である。Foundationが専用Operationをatomic登録するまで二候補からRegistry登録権限を推測しない。`activation.lod.authoring_operations.v1`が二件を同じContract set transactionで完全登録するまでGatewayはdispatchせず、要求を`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`でSource不変として拒否する。候補Policy／Before／After予測と承認済みPolicy ChangeSetはActivation後の予定結果であってcurrent動作ではない。Simulationの`dormant_record`は保持state schema／pending event／wake triggerを含むDomain-owned candidateで、可視性で破棄しない。

Editorはview／Target／Quality／pressure fixtureを切り替え、各subjectのcandidate、selected tier、projected error class、hysteresis state、missing dependency、fallback理由を表示する。Preview selectionをRuntime authoritative stateやProduction qualificationと混同しない。

## 11. Runtime、determinism、telemetry境界

LOD evaluationの実行slot、snapshot、writer、job dependency、handoff lifetimeは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)を参照し、phase表やSimulation Cadenceを複写しない。selectionは同一input snapshotでdeterministicにする。Presentation Geometry／Material／VFX selectionはauthoritative Replay rootへ入れずDebug captureへ供給する。authoritative behaviorへ関与するSimulation LOD transitionだけは、exact Contract／candidate ref、input context hash、previous／selected tier、transition reason、handoff generationをOwner-typed receipt-free Domain Replay Projectionとして[Persistence／Save](../04-runtime/persistence-save.md#5-replay-projection)へ供給する。

LOD固有telemetryはPlan／Context hash、candidate／selected tier、integer metric／threshold、pressure class、transition、thrash、missing／nonresident representation、fallback reason、completeness／gapを公開する。共通frame、memory、queue、capacity、regression測定とEvidence envelopeはRuntime／Governance ownerへ委譲する。

## 12. Validation、failure、qualification

Diagnosticはsubject／policy／representation Stable ID、View／Target、previous／requested／selected tier、error／importance／pressure class、error code、remediationを含む。少なくともinvalid policy、missing candidate、dependency mismatch、unsupported capability、stale generation、transition incompatible、nonresident、selection oscillationを区別する。

| Condition | 結果 |
|---|---|
| 非単調level／threshold、NaN／Inf、unit不一致 | ChangeSet／Cook拒否 |
| Selection Row 0件／複数、band gap／overlap | Plan compile拒否 |
| pressure snapshot null／stale／Target・Quality不一致 | previous／highest-fidelity維持、typed Diagnostic |
| fallback closure欠落 | Package promotion拒否 |
| interactive／mutable objectのHLOD混入 | HLOD Cook拒否 |
| generated meshのvisual／deformation error超過 | candidate破棄、Source chain／LOD0維持 |
| GPU selector容量超過／Backend fault | 次frameからCPU direct fallback、Diagnostic |
| LOD Artifact未resident | 同generationのresident fallback、miss記録 |
| simulation wake event欠落／queue overflow | hard failure。eventをdropしない |
| Reference simulation不一致 | Simulation LOD非Promotion |
| critical VFX cue floor未達 | Policy非Promotion |
| Target capacity未達 | Source維持、`OptimizationRequired` |

Diagnostic IDを`MIRAKAN-LOD-SCHEMA_UNKNOWN | MIRAKAN-LOD-SEMANTIC_ROLE_CONFLICT | MIRAKAN-LOD-NON_MONOTONIC | MIRAKAN-LOD-SELECTION_ROW_MISSING | MIRAKAN-LOD-SELECTION_ROW_AMBIGUOUS | MIRAKAN-LOD-METRIC_BAND_INVALID | MIRAKAN-LOD-PRESSURE_CONTEXT_INVALID | MIRAKAN-LOD-MISSING_FALLBACK | MIRAKAN-LOD-UNSUPPORTED_TRANSITION | MIRAKAN-LOD-GENERATION_ERROR_LIMIT | MIRAKAN-LOD-HLOD_INTERACTIVE_SOURCE | MIRAKAN-LOD-SIMULATION_VISIBILITY_INPUT | MIRAKAN-LOD-SIMULATION_EQUIVALENCE | MIRAKAN-LOD-CRITICAL_CUE_FLOOR | MIRAKAN-LOD-RESIDENCY_MISS | MIRAKAN-LOD-CAPACITY_EXCEEDED | MIRAKAN-LOD-TARGET_UNQUALIFIED`に固定する。

Qualificationは次のDomain fixtureを持つ。

- Domain schemaからgenerated C++／TypeScript／binary descriptor／MCP projectionが同じclosed field／enumを表し、unknown field／enum／majorを拒否する。projection mechanicsはFoundation ownerを使う。
- unit、range、monotonic、fallback closure、Selection Row exact一件、bandの全域一回coverage、Preset version、policy lockのpositive／negative fixture。Human、AI、headless CLIが同じIntentから同じPolicy／Receipt-free Plan hashへ収束する。
- Preset RegistryはCore neutral六件とinitial optional十二entryのresolved policyをgolden fixtureで固定し、Genre Pack 0件のneutral Project、qualified `project.board_game.token@1` contributionをpositiveで検証する。unknown ref、Qualification欠落、owner／hash stale、Core ID上書き、同一ID別hash、Pack未選択、表示名／suffixからの暗黙選択を各一原因negativeとしてSource／last-valid Registry不変で拒否する。
- FOV端値、near-plane交差、orthographic／perspective／physical／pixel projection、resolution、dynamic extent、camera cut、split Editor Viewについて`algorithm.lod.projected_metric.v1`のfloating intermediate、ceil Q16、saturation、境界包含がCPU／GPU／Previewで一致するgolden値。
- CPU／GPUのconservative frustum／layer oracleが同じ`ViewLodCandidateSetV1` Stable ID／bounds generation集合を生成し、set外subjectがhidden tier、Simulation relevancy、residency requestへ変換されず、HZB／occlusion差がLOD selectionを変えないfixture。
- CPU direct／GPU indirectのtier、境界包含、hysteresis一致とsilhouette、normal、UV seam、vertex color、material interface、skinning、morph、shadow error fixture。
- camera path往復時のthrash、pop、cross-fade overdraw、residency miss。
- Source順序を変えてもHLOD cluster／Artifact hash／proxy boundsが一致し、interactive／mutable Physics／Save／animation objectを拒否する。
- cell境界、teleport、camera cut、Artifact promotion、pressure class遷移、stale pressureの同時発生と、HLOD on／offでCollision、Nav、Damage、Save、Replay結果が一致すること。
- Full referenceと各Production simulation tierのReplay hash、最終count、Damage、goal、pending event一致。enter／exit、minimum residency、wake trigger同時発生、event queue上限、Save／Load直後のfull／last-valid復元とfresh再選択を検証する。
- camera、frustum、occlusion、VFX visibility、Target／Quality Profileを変えてもauthoritative simulation結果が一致すること。
- Animation required bone、root motion、hitbox、event timing、VFX critical cue floor、pixel artのpoint sampling／integer scale／palette／pixel-locked、Terrain／Foliage／WaterのCollision／Nav／CPU Query非逆入力。
- camera移動、LOD／HLOD遷移、spawn burst、Physics／Navigation／Animation、owner-typed authoritative subject／critical presentation VFX、streaming、Asset promotionを同一runで発生させるIntegrated fixture。run条件とEvidence transportはRuntime／Governance ownerを使う。
- Unreal screen-size／HLOD、Unity screen-size／cross-fade、Godot screen-space Mesh LOD／Visibility Rangeを比較fixtureの期待カテゴリに使うが、外部Engineの既定値やBackend結果とのbyte equalityを合格条件にしない。MiraikanaiのPlan hash、semantic floor、Target fallback、Save／Replay invariantを合格条件にする。

Evidence envelopeとgradingは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を参照する。distance-only selection、tier index coupling、silent hide、authoritative behavior loss、phase／budget複写が残る実装はRelease候補にしない。本書はdomain qualification evidenceを出力し、activationと導入順は[Product Plan](../00-product/product-plan.md)が決定する。

[Performance／capacityが所有する`IntegratedScaleFixtureV1`](../appendices/performance-scale-catalog-proposal.md#13-integrated-fixtureとqualification)を使い、camera移動、LOD／HLOD遷移、spawn burst、Physics／Navigation／Animation、owner-typed authoritative subject／critical presentation VFX、streaming、Asset promotionを同一runで発生させる。`LodQualificationReceiptV1`は`receipt_id／version／content_hash`、exact `LodResolutionPlanRefV1`、`artifact_generation_refs[]`、`target_profile_ref`、`quality_profile_ref`、`device_profile_ref`、`driver_ref`、exact `fixture_id／fixture_version／fixture_content_hash`、`camera_path_hash`、`before_metrics`、`after_metrics`、`visual_diff_metrics`、`gameplay_replay_hash`、`fallback_events[]`、`diagnostics[]`、`result`、`toolchain_ref`、`timestamp`を持つ。Receiptまたはwrapper hashをLOD Plan preimageへ戻さない。Evidence envelopeの定義は[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)の正本を使う。

全Domainを一つの`LodProfileV1`や一つの`lod_index`へ畳み込む設計は重複契約として非採用である。Worldの`SpatialPartitionIntentV1`は[World](world.md)が所有し、LODはそのDerived Planに対するrepresentationだけを選ぶ。Gameplay意味の変更が必要な場合はLOD proposalと分離した`GameplayScaleChangeProposalV1`と人間承認を[Executable contracts](../02-foundation/executable-contracts.md)のChangeSet経路へ返し、LODが暗黙Commitしない。
