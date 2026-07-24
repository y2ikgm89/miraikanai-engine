# Miraikanai Engine LOD Contract

- 文書ID: mirakan.arch.rendering-lod
- 状態: review
- 正本範囲: LOD intent／policy、representation set／tier、projected-error／importance／pressure入力、hysteresis／transition、geometry／HLOD／simulation／animation／material／VFX／terrain representation selection、LOD固有operation／diagnostic／qualification
- 非正本範囲: representation asset生成／promotion、Render pass／visibility execution、World streaming activation、Simulation behavior、Runtime shared capacity／phase、Tool version、AI authorization、Evidence envelope、共通Schema／projection。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Math／Core utilities](../02-foundation/math-core.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[Animation](../05-simulation/animation.md)、[Physics](../05-simulation/physics.md)、[Navigation](../05-simulation/navigation.md)、[Render Graph](render-graph.md)、[Materials](materials.md)、[World](world.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論と所有境界

LODは距離だけでなく、projected error、semantic importance、view role、quality intent、runtime pressureを入力に、どのrepresentationを選ぶかを一意に所有する。各Subsystemは候補representationとcost／quality metadataを公開し、LOD Resolverが選択、hysteresis、transition、fallbackを決める。

[Render Graph](render-graph.md)は選択済みgeometry／material representationのvisibilityとdraw executionを所有する。[World](world.md)はWorld／Scene／Level Sourceと`WorldStreamingPlanV1`内のplan-local Cell descriptorを所有し、Runtime scheduling／capacity ownerがactivationとpressureを決める。Physics、Navigation、Animationは各Domainのbehavior semanticsを所有し、LODが別のDynamics／Nav／Animation規則を作らない。

Asset import、cook、promotion、generation leaseは[Asset lifecycle](../03-authoring/asset-lifecycle.md)だけが所有する。本書はartifactを生成せず、同一source identityへ紐づく候補artifactの選択条件を定義する。

## 2. 設計判断と共通語彙

LOD語彙を次に固定する。

- `representation`: 同じSource意味を異なるcost／fidelityで表す候補。
- `tier`: 高品質からfallbackまでのcanonical順序を持つ候補識別子。
- `intent`: quality、importance、transition、minimum semanticsのauthoring要求。
- `selection`: View／Domain／subjectごとに解決されたtierと理由。
- `transition`: old／new representationの共存、blend、handoff、history reset条件。
- `pressure`: Runtime ownerが公開するcapacity signal。LODは測定法や閾値を再定義しない。
- `HLOD`: 複数subjectを一つのrender representationへ集約する候補。World identityの置換ではない。

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

## 3. LOD policyとrepresentation contract

### 3.1 `LodIntentV1`

```text
LodIntentV1
  intent_id
  owner_stable_id
  semantic_role
  semantic_priority
  gameplay_fidelity_floor
  visual_fidelity_floor
  style_profile_id
  target_profile_set[]
  expected_peak_resident
  expected_peak_visible
  expected_view_distance_m
  interaction_radius_m
  locked_capabilities[]
  allowed_lod_classes[]
  forbidden_degradations[]
  policy_set_id
  source_revision
```

`expected_view_distance_m`はPreview／fixture入力でruntime switch thresholdではない。gameplay floorはowner-typed authoritative instance数、Damage、Collision、Navigation、goal、spawn timing、input response、critical cueをtyped fieldで持ち、visual floorはsilhouette、material feature、animation cue、pixel-art sampling、minimum visible size、critical VFX outputを持つ。`forbidden_degradations`は登録済みCapability IDだけを許可する。

### 3.2 `LodPolicySetV1`

```text
LodPolicySetV1
  policy_set_id
  policy_version
  intent_id
  mesh_lod_profile?
  sprite_lod_profile?
  representation_lod_profile?
  simulation_lod_contract?
  animation_lod_profile?
  material_lod_profile?
  vfx_lod_profile?
  surface_lod_profiles[]
  residency_lod_profile?
  preset_provenance?: LodPolicyPresetSelectionV1
  policy_locks[]
```

Optional fieldがないDomainをdefault推測しない。未使用classは`allowed_lod_classes`から除外し、必要Profile欠損はCook拒否とする。

### 3.3 `LodResolutionPlanV1`

```text
LodResolutionPlanV1
  plan_id
  source_intent_hash
  source_policy_hash
  target_profile_id
  quality_profile_id
  capability_manifest_hash
  domain_plans[]
  fallback_closure[]
  predicted_cost_before
  predicted_cost_after
  fidelity_checks[]
  unresolved_requirements[]
  status
  compiler_version
```

`domain_plans[]`はDomain固有typed payloadのtagged unionである。unknown tag／major、Target fallback欠損、Source hash不一致を拒否する。Quality ProfileはPresentation実装品質だけを変え、simulation contract、owner-typed authoritative instance数、Damageを変更できない。

`RepresentationDescriptor`はStable ID、Domain kind、source identity、artifact generation ref、quality rank、geometric／semantic error bound、required capability、estimated cost class、dependency closure、transition compatibility、fallback targetを持つ。costの共通測定単位、budget、capacity envelopeは[Runtime performance／capacity](../04-runtime/performance-capacity.md)を参照する。

候補setは同じsource semanticsとversion closureに属し、missing tier、duplicate rank、fallback cycle、incompatible dependency、stale artifactを拒否する。低tierがgameplay collision、navigation reachability、authoritative identityを暗黙に変えない。

## 4. 共通選択契約

Resolver inputはsubject Stable ID、representation set generation、ViewFamily／view role、projection、viewport extent、bounds、importance、occlusion confidence、Target／Quality、Runtime pressure snapshot、previous selectionを含む。

`ViewLodContextV1`はprojection、render extent、FOV／orthographic span、view transform、camera cut、Quality Profileから`projected_error_px_q16`と`projected_coverage_px_q16`を生成する。distanceだけをMesh thresholdとせず、CPU／GPUは同じ量子化／境界包含／enter／exit比較を使う。Editor／shadow／reflection Viewは独立したselection／history stateを持つ。

共通metric IDと用途を次に固定する。

| Metric | 用途 | 禁止用途 |
|---|---|---|
| `projected_error_px_q16` | Mesh／surface geometry detail | Simulation |
| `projected_coverage_px_q16` | Representation、VFX、Material presentation | Gameplay relevancy |
| `distance_mm_u64` | HLOD cell／manual visibility range、bounded surface | FOV／解像度依存のMesh品質判定 |
| `gameplay_relevance_q16` | Simulation tier | Rendering、occlusion |
| `budget_pressure_q16` | 同一fidelity内の候補選択 | Gameplay fidelity floorの緩和 |

Cooked thresholdは整数へ量子化し、CPU／GPUは同じ比較方向、境界包含、fixtureを使う。NaN、Inf、負値、非単調thresholdを受理しない。

選択順はminimum semantics、Target capability、error bound、importance、pressure policy、previous selection／hysteresis、Stable ID tie-breakでcanonicalにする。worker completion順、frame timeの単一sample、hash-map iteration順を使わない。

`LodTransitionRuleV1`は`from_tier`、`to_tier`、`metric_id`、`enter_threshold`、`exit_threshold`、`minimum_residency`、`transition_mode`、`transition_extent`、`camera_cut_policy`、`missing_artifact_policy`を持つ。`minimum_residency`はclosed tagged unionで、Presentation branch=`{kind=presentation_frames, count: positive uint32}`、Simulation branch=`{kind=simulation_advances, count: positive uint32, cadence_profile_ref: SimulationCadenceProfileRefV1}`とする。enter／exitを別に持ち境界往復を防ぐ。Presentation frameとSimulation Advanceを同じunitとして比較せず、Simulation branchのProfile refはRuntime選択値およびSave／Replay closureとbyte equalityにする。Cadence rate、wall time、Genreからcountを換算しない。`transition_mode`は`instant | dither | cross_fade | domain_blend`のclosed enumとし、未対応BackendではProfile登録済みの意味同等fallbackを使う。camera cut、projection／dynamic extent変更では古いvisibility historyを捨て、そのframeに必要なdetailへ即時再選択し、無効historyを理由に低detailを選ばない。

## 5. Mesh／Sprite geometry LOD

`MeshLodProfileV1`は`profile_id`、`source_mode`、`selection_metric = projected_error_px_q16`、`levels[]`、`transition_rule_set`、`quality_overrides[]`、`skin_policy`、`morph_policy`、`section_policy`、`shadow_policy`、`fallback_geometry`を持つ。`source_mode`は`disabled | source_chain | generated_chain | hybrid_chain`である。

`MeshLodLevelV1`は`level_index`、optional `source_asset_id`、optional `target_triangle_ratio_permille`、optional `maximum_object_error_q24`、`maximum_projected_error_px_q16`、`preserve_boundaries`、`preserve_uv_seams`、`preserve_hard_normals`、`preserve_vertex_color_channels[]`、`required_material_interface_hash`、`expected_triangle_count`、`artifact_role`を持つ。

Mesh／Sprite representationはgeometry artifact ref、bounds／silhouette error、vertex／primitive cost class、material interface、skin／morph compatibility、shadow／collision proxy relationを宣言する。LODはgeometry candidateを選び、meshlet／indirect draw／occlusionの実行は[Render Graph](render-graph.md)へ委譲する。

silhouette、UV、normal／tangent、skin weight、sprite pivot／pixel lockに意味差があるtierは明示する。missing material interfaceやanimation bindingをdefaultへ置換せず、compatible fallbackへ戻す。

`SpriteLodProfileV1`は`source_variant | visibility_only | disabled`を持つ。Pixel artのpoint sampling／integer scale／palette／alpha padding／pixel-lockedをQualityより優先し、Gameplay cueは意味同等variantなしに消さず、atlas page／texture mip／Sprite LODを同一indexにせず、stable pivot／bounds／sort／collision非依存を検証する。

## 6. Representation LODとHLOD

`RepresentationLodProfileV1`のtierは`individual | instanced | spatial_proxy | impostor | hidden_presentation`のclosed enumだけを使う。`hidden_presentation`はPresentationだけを隠し、Entity、Collision、Damage、Navigation、Save stateを削除しない。individualからinstancedへ変えてもStable Entity identityとauthoritative event routingを保持する。

`HlodProfileV1`は`profile_id`、`eligibility_rule`、`spatial_partition_profile`、`cluster_limits`、`proxy_mode`、`proxy_geometry_profile`、`proxy_material_profile`、`transition_rule_set`、`streaming_cell_profile`、`fallback_representation`を持つ。`proxy_mode`は`instanced | merged_mesh | simplified_mesh | impostor`である。対象はstatic transform、`decorative_instance`、mutable Physics／Damage／interaction／Save／animation／root motionなし、Style／Material許可、bounded plan-local Cell descriptorとcluster／geometry／material／texture capacity内をすべて満たす。

Clusterは`WorldStreamingPlanV1`の`plan_id`／`artifact_hash`、plan-local `cell_id`、Profile ID、material compatibility、Source Stable ID順から決定論的に生成する。`HlodArtifactV1`はSource Stable ID集合、Source revision hash、bounds、proxy method、geometry／material key、visual error、exact `{plan_id, artifact_hash, cell_id}`、fallbackを持つ。Source、transform、Material、Profile変更時に該当Artifactだけinvalidateする。HLODをEntity identity、Save owner、Physics／Navigation objectの代替にしない。

`WorldStreamingPlanV1`はplan-local Cell descriptorとHLOD artifactのresidency dependencyを記述し、LOD selectionはresident candidateから選ぶ。必要candidateが非residentの場合のrequest／backpressureは[World](world.md)とRuntime ownerへ返し、unbounded blocking loadを起こさない。

## 7. Simulation LOD境界

Simulation representationはfull、reduced、dormant等のDomain-defined behavior candidate refとsemantic guaranteeを公開できる。LODは候補を選択するが、Physics integration、Collision response、Navigation query、authoritative writer、wake conditionの意味は各Simulation Ownerが所有する。

`SimulationLodContractV1`は`contract_id`、`experience_role`、`tiers[]`、`relevance_inputs[]`、`transition_rules[]`、`wake_triggers[]`、`retained_state_schema`、`queued_event_policy`、`authoritative_equivalence_contract`、`forbidden_changes[]`、`reference_fixture_id`を持つ。Target／QualityはこのGameplay契約を自動的に強めたり弱めたりしない。

render tierとsimulation tierを同一indexで結ばない。off-screenでもauthoritative gameplayに必要なsimulationを停止せず、visibleでもcapacity／Targetが許さないsimulation featureを暗黙有効化しない。tier handoffはpublished state、generation、handoff resultを持つ。

## 8. Animation、Material、VFXとの境界

Animationはclip／pose／skin candidateとminimum event／root-motion semanticsを公開し、LODはrepresentationを選ぶ。event、root motion、IK、retargetの意味は[Animation](../05-simulation/animation.md)を参照し、低tierへの遷移でauthoritative eventを欠落させない。

`AnimationLodProfileV1`は`AnimationLodTierV1[]`を持ち、各tierは`tier_id`、`presentation_pose_sample_interval_frames: positive u32`、`interpolation_mode`、`presentation_bone_set`、`skinning_mode`、`shadow_pose_mode`、`required_bones[]`、`required_events[]`を持つ。このintervalはPresentation render frameだけを数え、Simulation Advance、Animation clip timebase tick、秒へ換算しない。required bone／eventを除外するProfileを拒否し、authoritative state machine／root motion／hitbox／authoritative attachment socket／foot contact／event timingを低頻度poseから取得しない。

Materialは各tierのcompatible artifactとfeature reductionを[Materials](materials.md)で宣言する。LODはtierを選ぶがshader variantやparameter意味を再定義しない。

[Materialsが所有する`MaterialLodProfileV1`](materials.md#7-material-lod境界)をexactに消費し、Fieldを再定義しない。Style-critical ramp、alpha semantics、pixel sampling、critical-cue emissiveを削除しない。

`VfxLodProfileV1`は`VfxLodTierV1[]`を持ち、各tierは`tier_id`、`semantic_priority`、`branch_id`、`spawn_scale_permille`、`maximum_alive`、`update_interval={kind=simulation_advances,count: positive u32,cadence_profile_ref: SimulationCadenceProfileRefV1}`、`renderer_outputs[]`、`simulation_target`、`minimum_cue_contract`を持つ。intervalは選択ProfileのSimulation Advance数だけを意味し、render frame、秒、表示Hzへ変換しない。current `fixed_rational_advance_v1` VFX artifactは`count=1`だけを受理し、`count>1`はmulti-advance integrationを定義する新Artifact versionとCadence別Qualificationがactiveになるまで`vfx_lod_update_interval_not_qualified`で拒否する。`critical_gameplay_cue`はshape／timing／minimum visibilityを維持し、ambient effectより先にdropしない。VFX tierはGameplay event数／Damage／Collision／AI perceptionを変更しない。

## 9. Terrain、Foliage、Water／Surface、residency

Terrain、foliage、water、snow／surfaceはDomain Ownerがtile／patch／cluster representation候補、seam constraint、interaction guaranteeを公開し、LODは選択だけを行う。隣接tierのseam、crack、normal／material continuity、interaction proxyの互換性をdescriptorで検証する。

`TerrainLodProfileV1`はscreen-space geometry error、quadtree patch bounds、neighbor level差上限、skirt／stitch policy、material residency、streaming cellを持ち、render patchはCollision height／Nav tile／Gameplay Surface Stateを置き換えない。`FoliageLodProfileV1`はinstance mesh chain、wind tier、shadow tier、impostor、cluster bounds、per-cell instance上限を持ち、Gameplay Collision subsetを描画LODへ追従させない。

`WaterLodProfileV1`はsurface patch density、wave shading、reflection、underwater presentation、foam／spray VFXのTarget別tierを持ち、CPU Surface Query／Water Volume／浮力／swimming／Damage／Navigation costを変更しない。`SnowSurfaceLodProfileV1`はdynamic field update distance、normal／sparkle detail、static mask fallbackを持ち、Gameplay Surface State／friction／movement／foot contact／static coverageを変更しない。降雪particle密度の倍率は`vfx_presentation` classの`VfxLodProfileV1`だけが所有し、`SnowfallVfxProfileV1`の`density_scale`はauthored基準値である。

residency requestはartifact generation、priority intent、deadline class、fallback candidateを持つが、queue、memory reservation、backpressure値を本書で決めない。候補のresidencyが失われた場合はgenerationを再検証し、stale GPU／streaming handleを再利用しない。

`GeometryResidencyLodPlanV1`は`requested`、`resident`、`pending`、`fallback_level`、`byte_cost`、`deadline`、`owner_generation`を持つ。要求detailが非residentなら同じAsset generation内で最も近い意味同等resident levelを使い、別generationをframe内で混在させない。

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
```

`preset_content_hash`はASCII `MIRAKAN_LOD_POLICY_PRESET_V1`と自己hashを除くReceipt-free Preset canonical bytes、Registry hashはASCII `MIRAKAN_LOD_POLICY_PRESET_REGISTRY_V1`、Registry ID／version、Preset ID／version順の全Preset canonical bytesから計算する。selected refsはPreset ID／version／hash順、Binding refsは解決したsubject refの同じ順にstrict sortし、duplicateを拒否する。Activation Projectionのselected ref集合とQualification Bindingが解決する合格かつfreshなsubject集合はexact set equalityで、Receipt／BindingをPreset／Registry hashへ戻さない。Feature／Genre contributionは所有Pack identity、Project contributionはProject owner identityへexact解決する。自己namespace外ID、Coreまたは他Owner entryの上書き、duplicate、unknown、stale owner／version／hash、unqualified entry、Target／Capability不成立をfail closedにする。Registry materializationはProject／Pack dependency closureを解決したCompilerが行い、Generic Engine CoreからPackへのdependency edgeを生成しない。

Core required defaultはGenre／object classを仮定しない`lod.preset.core.primary_subject@1 | lod.preset.core.interactive_subject@1 | lod.preset.core.supporting_subject@1 | lod.preset.core.decorative_subject@1 | lod.preset.core.critical_presentation_cue@1 | lod.preset.core.ambient_presentation@1`のexact六件である。従来defaultは次のexact contributionへ移し、同じ意味とresolved policyをgolden fixtureで維持する。

| 従来suffix | canonical `preset_id@version` | `contribution_layer`／logical owner |
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

表のlogical ownerはmaterialization時にversion／content hashを含むexact `owner_ref`またはPack identityへ解決する。Core互換entryはCore required六件とは別のoptional default、Feature entryは該当Packを選択したProjectだけのcandidateである。いずれも全Projectの暗黙default、名前fallbackにはせず、該当Contributionを選択しないProjectでは候補に現れない。これによりboard／puzzle／simulation／tool ProjectがCharacter、Crowd、Combat vocabularyへ依存せず、必要なPackだけが固有Presetを追加できる。

従来suffixは互換fixtureの説明にだけ残し、新SourceのPreset refまたはaliasとして受理しない。実在する旧bytesを移行する場合はsource schema bytes／Owner／Named Algorithm／immutable fixtureを束縛した別の承認済みschema migrationを先にactivateし、suffixまたはsemantic roleだけで自動変換しない。

`LodPlanPreviewV1`は対象Stable ID／semantic role、exact Preset ref／owner／qualification、変更する／しないLOD class、Target別tier／threshold／fallback、triangle／draw／instance／memory／overdraw／simulation workのBefore／After、visual／silhouette／animation／critical-cue risk、Gameplay fidelity影響、Artifact／tool version／予測・実測区分、blocking Diagnostic／Approvalを示す。表示名、suffix、semantic roleからPresetを推測しない。

上記Registry、Projection、Core／compatibility default、extension fixtureはtarget contractであり、実装済み、active Gateway surface、Production Qualification済みという主張ではない。

LODのcreate／update policy、bind representation、apply preset、set importance、preview selection、explain transition、validate closureは将来のDomain action vocabularyであり、現在のOperation登録ではない。

`operation.lod.propose_policy`と`operation.lod.apply_policy`は[Executable contracts](../02-foundation/executable-contracts.md#211-既存domain文書から回収した未登録operation候補)の`planning.operation_family.lod_authoring@1`に属するexact二候補である。Capability stateは`not_activated`、current MCD／Owner Manifest／Service allowlist／Policy／Validator／Diagnostic／Receipt／Provider／MCP／generated alias／legacy alias集合はすべて`[]`である。前者はActivation後にだけPreset selectionを含むProposalを返す予定で、現在のselection権限ではない。Preset contribution自体のauthoring／activationは別計画語彙`planning.lod.policy_preset_contribution_authoring@1`とし、reserved Operation ID集合`[]`、current MCD／Owner Manifest／Service allowlist／Policy／Validator／Diagnostic／Receipt／Provider／MCP／CLI／Editor／generated alias／legacy alias集合もexact `[]`、Capability stateも`not_activated`である。Foundationが専用Operationをatomic登録するまで二候補からRegistry登録権限を推測しない。`activation.lod.authoring_operations.v1`が二件を同じContract set transactionで完全登録するまでGatewayはdispatchせず、要求を`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`でSource不変として拒否する。候補Policy／Before／After予測と承認済みPolicy ChangeSetはActivation後の予定結果であってcurrent動作ではない。Simulationの`dormant_record`は保持state schema／pending event／wake triggerを含むDomain-owned candidateで、可視性で破棄しない。

Editorはview／Target／Quality／pressure fixtureを切り替え、各subjectのcandidate、selected tier、projected error class、hysteresis state、missing dependency、fallback理由を表示する。Preview selectionをRuntime authoritative stateやProduction qualificationと混同しない。

## 11. Runtime、determinism、telemetry境界

LOD evaluationの実行slot、snapshot、writer、job dependency、handoff lifetimeは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)を参照し、phase表やSimulation Cadenceを複写しない。selectionは同一input snapshotでdeterministicにし、Replayにはinput identityとselection／reasonをDomain projectionとして供給する。

LOD固有telemetryはcandidate／selected tier、transition、thrash、missing／nonresident representation、fallback reasonを公開する。共通frame、memory、queue、capacity、regression測定とEvidence envelopeはRuntime／Governance ownerへ委譲する。

## 12. Validation、failure、qualification

Diagnosticはsubject／policy／representation Stable ID、View／Target、previous／requested／selected tier、error／importance／pressure class、error code、remediationを含む。少なくともinvalid policy、missing candidate、dependency mismatch、unsupported capability、stale generation、transition incompatible、nonresident、selection oscillationを区別する。

| Condition | 結果 |
|---|---|
| 非単調level／threshold、NaN／Inf、unit不一致 | ChangeSet／Cook拒否 |
| fallback closure欠落 | Package promotion拒否 |
| interactive／mutable objectのHLOD混入 | HLOD Cook拒否 |
| generated meshのvisual／deformation error超過 | candidate破棄、Source chain／LOD0維持 |
| GPU selector容量超過／Backend fault | 次frameからCPU direct fallback、Diagnostic |
| LOD Artifact未resident | 同generationのresident fallback、miss記録 |
| simulation wake event欠落／queue overflow | hard failure。eventをdropしない |
| Reference simulation不一致 | Simulation LOD非Promotion |
| critical VFX cue floor未達 | Policy非Promotion |
| Target capacity未達 | Source維持、`OptimizationRequired` |

Diagnostic IDを`MIRAKAN-LOD-SCHEMA_UNKNOWN | MIRAKAN-LOD-SEMANTIC_ROLE_CONFLICT | MIRAKAN-LOD-NON_MONOTONIC | MIRAKAN-LOD-MISSING_FALLBACK | MIRAKAN-LOD-UNSUPPORTED_TRANSITION | MIRAKAN-LOD-GENERATION_ERROR_LIMIT | MIRAKAN-LOD-HLOD_INTERACTIVE_SOURCE | MIRAKAN-LOD-SIMULATION_VISIBILITY_INPUT | MIRAKAN-LOD-SIMULATION_EQUIVALENCE | MIRAKAN-LOD-CRITICAL_CUE_FLOOR | MIRAKAN-LOD-RESIDENCY_MISS | MIRAKAN-LOD-CAPACITY_EXCEEDED | MIRAKAN-LOD-TARGET_UNQUALIFIED`に固定する。

Qualificationは次のDomain fixtureを持つ。

- Domain schemaからgenerated C++／TypeScript／binary descriptor／MCP projectionが同じclosed field／enumを表し、unknown field／enum／majorを拒否する。projection mechanicsはFoundation ownerを使う。
- unit、range、monotonic、fallback closure、Preset version、policy lockのpositive／negative fixture。Human、AI、headless CLIが同じIntentから同じPolicy／Plan hashへ収束する。
- Preset RegistryはCore neutral六件と従来default十二suffixのresolved policyをgolden fixtureで維持し、Genre Pack 0件のneutral Project、qualified `project.board_game.token@1` contributionをpositiveで検証する。unknown ref、Qualification欠落、owner／hash stale、Core ID上書き、同一ID別hash、Pack未選択、表示名／suffixからの暗黙選択を各一原因negativeとしてSource／last-valid Registry不変で拒否する。
- FOV、orthographic／perspective、resolution、dynamic extent、camera cut、split Editor Viewのprojected-error golden値。
- CPU direct／GPU indirectのtier、境界包含、hysteresis一致とsilhouette、normal、UV seam、vertex color、material interface、skinning、morph、shadow error fixture。
- camera path往復時のthrash、pop、cross-fade overdraw、residency miss。
- Source順序を変えてもHLOD cluster／Artifact hash／proxy boundsが一致し、interactive／mutable Physics／Save／animation objectを拒否する。
- cell境界、teleport、camera cut、Artifact promotion、memory pressureの同時発生と、HLOD on／offでCollision、Nav、Damage、Save、Replay結果が一致すること。
- Full referenceと各Production simulation tierのReplay hash、最終count、Damage、goal、pending event一致。enter／exit、minimum residency、wake trigger同時発生、event queue上限、Save／Load直後を検証する。
- camera、frustum、occlusion、VFX visibility、Target／Quality Profileを変えてもauthoritative simulation結果が一致すること。
- Animation required bone、root motion、hitbox、event timing、VFX critical cue floor、pixel artのpoint sampling／integer scale／palette／pixel-locked、Terrain／Foliage／WaterのCollision／Nav／CPU Query非逆入力。
- camera移動、LOD／HLOD遷移、spawn burst、Physics／Navigation／Animation、owner-typed authoritative subject／critical presentation VFX、streaming、Asset promotionを同一runで発生させるIntegrated fixture。run条件とEvidence transportはRuntime／Governance ownerを使う。

Evidence envelopeとgradingは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を参照する。distance-only selection、tier index coupling、silent hide、authoritative behavior loss、phase／budget複写が残る実装はRelease候補にしない。本書はdomain qualification evidenceを出力し、activationと導入順は[Product Plan](../00-product/product-plan.md)が決定する。

[Performance／capacityが所有する`IntegratedScaleFixtureV1`](../04-runtime/performance-capacity.md#13-integrated-fixtureとqualification)を使い、camera移動、LOD／HLOD遷移、spawn burst、Physics／Navigation／Animation、owner-typed authoritative subject／critical presentation VFX、streaming、Asset promotionを同一runで発生させる。`LodQualificationReceiptV1`は`receipt_id`、先に固定したReceipt-free `plan_hash`、`artifact_hashes[]`、`target_profile`、`device／driver`、exact `fixture_id／fixture_version／fixture_content_hash`、`camera_path_hash`、`quality_profile`、`before_metrics`、`after_metrics`、`visual_diff_metrics`、`gameplay_replay_hash`、`fallback_events[]`、`diagnostics[]`、`result`、`toolchain_hash`、`timestamp`を持つ。Receiptまたはwrapper hashをLOD Plan preimageへ戻さない。Evidence envelopeの定義は[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)の正本を使う。

全Domainを一つの`LodProfileV1`や一つの`lod_index`へ畳み込む設計は重複契約として非採用である。Worldの`SpatialPartitionIntentV1`は[World](world.md)が所有し、LODはそのDerived Planに対するrepresentationだけを選ぶ。Gameplay意味の変更が必要な場合はLOD proposalと分離した`GameplayScaleChangeProposalV1`と人間承認を[Executable contracts](../02-foundation/executable-contracts.md)のChangeSet経路へ返し、LODが暗黙Commitしない。
