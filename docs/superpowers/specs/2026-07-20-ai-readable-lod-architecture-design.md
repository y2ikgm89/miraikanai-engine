# Miraikanai Engine AI可読LODアーキテクチャ規約

- 文書版: 1.1
- 作成日: 2026-07-20
- 最終更新日: 2026-07-20
- 対象: Mesh／Sprite LOD、Representation LOD、HLOD、Simulation LOD、Animation LOD、Material LOD、VFX LOD、Terrain／Foliage／Water／Snow Surface LOD、Geometry residency
- 状態: プロジェクト公式の規範設計レビュー版
- 上位文書: [AIネイティブ独自ゲームエンジン 設計計画書](./2026-07-18-ai-native-game-engine-authoring-design.md)
- 機能範囲: [Miraikanai Engine 2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)
- Runtime正本: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- Renderer正本: [Miraikanai Engine Rendering／Render Graphアーキテクチャ規約](./2026-07-19-rendering-render-graph-architecture-design.md)
- Asset正本: [Miraikanai Engine Asset Pipeline／Content Package規約](./2026-07-19-asset-pipeline-content-packaging-design.md)
- Asset Import／AI／Editor正本: [Miraikanai Engine Asset Import／AI Authoring／Editor UXアーキテクチャ規約](./2026-07-20-asset-import-ai-authoring-editor-ux-design.md)
- Particle／VFX正本: [Miraikanai Engine 独自Particle／VFX Platformアーキテクチャ規約](./2026-07-20-particle-vfx-architecture-design.md)
- Water正本: [Miraikanai Engine Water Surface Platformアーキテクチャ規約](./2026-07-20-water-surface-platform-architecture-design.md)
- Weather／Snow正本: [Miraikanai Engine Weather／Snow Surfaceアーキテクチャ規約](./2026-07-20-weather-snow-surface-architecture-design.md)
- Authoring正本: [Miraikanai Engine Authoring Model／Project State規約](./2026-07-19-authoring-model-project-state-design.md)
- 実行可能契約: [Miraikanai Engine 実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)
- World／Level／Map正本: [Miraikanai Engine World／Level／Map／AI Authoringアーキテクチャ規約](./2026-07-20-world-level-map-ai-authoring-architecture-design.md)

## 1. 結論

Miraikanai EngineのLODは、一つの距離値または一つの万能`lod_index`で全Subsystemを切り替える機能にしない。AIと人間が編集する共通`LodIntentV1`、Domainごとの型付きPolicy、Runtime CompilerがTarget別に生成する`LodResolutionPlanV1`、実測結果を保持する`LodQualificationReceiptV1`へ分離する。

```text
Game Brief／Asset role／Visual Style／Scale intent
  -> LodIntentV1
  -> LodPolicySetV1
  -> Target Capability／Budget／Gameplay fidelity floor
  -> LodResolutionPlanV1
  -> Domain Cooked Artifact
  -> Runtime selection／transition
  -> Telemetry／Visual Diff／Replay
  -> LodQualificationReceiptV1
```

LODの目的は、Projectが宣言した見た目とGameplay上の意味を保ちながら、Targetのframe、memory、streaming、thermal budgetへ収めることである。「遠いから敵を消す」「負荷が高いから重要なVFXを落とす」「HLOD proxyをCollisionとして使う」等、Subsystem境界を越えて意味を変える近似を禁止する。

AIはSource Intent、Policy、Preset、Preview付きChangeSetを提案できる。AI、Editor widget、Project C++、ImporterはCooked mesh、HLOD proxy、GPU visibility buffer、simulation tier、Qualification statusを直接変更できない。C++ Gateway、Compiler、Validator、Runtime phaseだけがそれぞれの状態を確定する。

World／Level／Map規約の`SpatialPartitionIntentV1`と`WorldStreamingPlanV1`がCell residencyとActivationを所有し、本書は各Cell内外のRepresentation／HLOD／Geometry residency policyだけを所有する。HLOD proxy、render visibility、camera、GPU queryはCell active判定、Level completion、Enemy／Damage／Objective、Navigation authorityへ逆入力しない。

## 2. 設計判断

比較した方式と採否を次に固定する。

| 方式 | 長所 | 問題 | 採否 |
|---|---|---|---|
| 全Domain共通の万能`LodProfileV1` | API数が少ない | 描画距離がSimulationへ流入し、Domain固有failureとfidelityを表現できない | 不採用 |
| Subsystemごとの完全独立設定 | 各実装が単純 | AIが関係を毎回推測し、Target別全体最適化と説明が分断される | 不採用 |
| 共通Intent＋Domain別Policy＋共通Receipt | 意味を共有し、実行とfailureを分離できる | Contract数が増える | 採用 |

有名EngineのAPI名やAsset形式はコピーしない。Unityの画面占有率と明示的transition、Unreal EngineのLOD Group／HLOD Layer／Mass Simulation LOD分離、Godotのpixel error／bias／hysteresisの長所を、MiraikanaiのChangeSet、MCD、Target Profile、Gameplay fidelity floor、Qualificationへ統合する。

## 3. 決定権と非対象

| 主題 | 正本 |
|---|---|
| LOD共通語彙、Intent、Policy envelope、metric、transition、AI Operation、Receipt、Domain間禁止規則 | 本書 |
| Render Snapshot、CPU／GPU selector、visibility、packet、HLOD描画、GPU fallback | Rendering規約 |
| Entity tier遷移、dormancy、tick、Save／Replay、Gameplay fidelity | Runtime規約 |
| Mesh reduction input／output、Derived Artifact、streaming、promotion | Asset規約 |
| Source LOD chain、generated設定、Import Preview、Reimport conflict | Asset Import／AI／Editor規約 |
| VFX branch、spawn／alive／overdraw budget、critical cue | Particle／VFX規約 |
| Water patch／surface qualityとCPU Query分離 | Water規約 |
| Snow dynamic field／static maskとGameplay Surface State分離 | Weather／Snow規約 |
| Terrain／Foliageの正式Capability着手条件 | Domain Pack／将来Capability規約 |

次は本書のLODへ統合しない。

- Texture mipはAsset residencyであり、Meshの`lod_index`と同じ番号空間を共有しない。ただし`LodResolutionPlanV1`のvisual fidelity floorを入力にTarget別residency planを生成する。
- Audio attenuation、voice virtualization、streaming decode qualityはAudio規約が所有し、Rendererのdistance／visibilityを入力にしない。
- Shadow、Reflection、GIのTechnique選択はRendering／Shadow Resolverが所有する。LODはobjectのsemantic priorityとboundsを渡せるが、Techniqueを直接選ばない。
- UI、Text、accessibility cue、pixel-locked layerはWorld LOD、dynamic resolution、Frame Generationで劣化させない。
- Presentation LODのCollision、Navigation、Damage、AI perceptionへの逆入力を禁止する。

## 4. 共通語彙

### 4.1 LOD class

`LodClassV1`を次のclosed enumとする。同じobjectが複数classのPolicyを持てるが、一つのclassのtierを別classへ暗黙転用しない。

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

### 4.2 semantic priority

`LodSemanticPriorityV1`を次のclosed enumとする。

```text
critical_gameplay_cue
interactive_subject
primary_subject
supporting_subject
decorative
ambient
```

`critical_gameplay_cue`はhit、危険範囲、parry timing、目標状態等、見落とすとGameplay判断を誤るPresentationに使用する。`decorative`または`ambient`であることを理由にauthoritative stateを削除してはならない。正規の`experience_role`と`LodSemanticPriorityV1`が矛盾する場合は`MIRAKAN-LOD-SEMANTIC_ROLE_CONFLICT`で拒否する。

### 4.3 status

LOD CapabilityとProject planは次のstatusを持つ。

```text
Unavailable | Preview | Predicted | OptimizationRequired | Qualified
```

実測Receiptなしに`Qualified`、`最適化済み`、`快適`と表示しない。BackendまたはTarget Capabilityがない場合はSourceを変更せず`Unavailable`または`OptimizationRequired`にする。

## 5. 共通MCD contract

### 5.1 `LodIntentV1`

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

- `expected_view_distance_m`はauthoring previewとfixture生成の入力であり、Mesh LODのruntime切替閾値ではない。
- `gameplay_fidelity_floor`は敵数、味方数、Damage、Collision、Navigation、goal、spawn timing、入力応答、重要cueを列挙できるtyped fieldとする。
- `visual_fidelity_floor`はsilhouette、material feature、animation cue、pixel-art sampling、minimum visible size、critical VFX outputをtyped fieldで持つ。
- `forbidden_degradations`は自由文字列にせず、MCDで登録されたCapability IDだけを持つ。

### 5.2 `LodPolicySetV1`

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
  preset_provenance?
  policy_locks[]
```

Optional fieldがないDomainをdefault値で推測しない。該当classを使用しない場合は`allowed_lod_classes`から除外し、必要なのにProfileがない場合はCookを拒否する。

### 5.3 `LodResolutionPlanV1`

Runtime CompilerはSource Intentを直接実行せず、Targetごとに次をCookする。

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

`domain_plans[]`はDomain固有のtyped payloadを持つtagged unionである。未知のtag、unknown major、Target fallback欠落、Source hash不一致を拒否する。Quality ProfileはPresentationの実装品質だけを変更でき、simulation LOD contract、敵数、Damage等を変更できない。

### 5.4 metric

共通metric IDを次に固定する。

| Metric | 用途 | 禁止用途 |
|---|---|---|
| `projected_error_px_q16` | Mesh／surface geometry detail | Simulation |
| `projected_coverage_px_q16` | Representation、VFX、Material presentation | Gameplay relevancy |
| `distance_mm_u64` | HLOD cell／manual visibility range、bounded surface | FOV／解像度依存のMesh品質判定 |
| `gameplay_relevance_q16` | Simulation tier | Rendering、occlusion |
| `budget_pressure_q16` | 同一fidelity内の候補選択 | Gameplay fidelity floorの緩和 |

Cooked thresholdは整数へ量子化する。CPU／GPUは同じ比較方向、境界包含、量子化fixtureを使用する。NaN、Inf、負値、非単調thresholdを受理しない。

### 5.5 `LodTransitionRuleV1`

```text
LodTransitionRuleV1
  from_tier
  to_tier
  metric_id
  enter_threshold
  exit_threshold
  minimum_residency_units
  transition_mode
  transition_extent
  camera_cut_policy
  missing_artifact_policy
```

- `enter_threshold`と`exit_threshold`を別に持たせ、境界往復を防ぐ。
- Presentationの`minimum_residency_units`はreal frame数、Simulationはfixed tick数であり、同じunitを共有しない。
- `transition_mode`は`instant | dither | cross_fade | domain_blend`のclosed enumとする。対応しないBackendではProfileに記録した意味同等fallbackを使う。
- camera cut、projection変更、dynamic extent変更時は古いvisibility historyを捨て、そのframeに必要なdetailへ即時再選択する。無効なhistoryを理由に低detailを選ばない。

## 6. Mesh／Sprite geometry LOD

### 6.1 `MeshLodProfileV1`

```text
MeshLodProfileV1
  profile_id
  source_mode
  selection_metric = projected_error_px_q16
  levels[]
  transition_rule_set
  quality_overrides[]
  skin_policy
  morph_policy
  section_policy
  shadow_policy
  fallback_geometry
```

`source_mode`は`disabled | source_chain | generated_chain | hybrid_chain`とする。`levels[]`は次を持つ。

```text
MeshLodLevelV1
  level_index
  source_asset_id?
  target_triangle_ratio_permille?
  maximum_object_error_q24?
  maximum_projected_error_px_q16
  preserve_boundaries
  preserve_uv_seams
  preserve_hard_normals
  preserve_vertex_color_channels[]
  required_material_interface_hash
  expected_triangle_count
  artifact_role
```

- LOD0は常にSourceの最高detailとし、Sourceを削除または上書きしない。
- `maximum_projected_error_px_q16`はFOV、render extent、projection、object boundsを含む`ViewLodContextV1`から評価する。
- level index、triangle count、許容誤差は低detail方向へ単調でなければならない。
- Render LODをCollision mesh、Nav source、Damage geometryへ自動転用しない。それらは独立したSource roleとCooked Artifactを持つ。
- Material sectionの削減は`required_material_interface_hash`とVisual Style契約を保つ場合だけ許可する。Material統合とtexture bakeはHLOD／Material Pipelineの別Artifactとする。
- generated reducerはoffline Tool Adapter内へ隔離し、Engine-owned input／output、deterministic seed、version／hash、error reportを使用する。特定Reducer libraryの型を公開契約へ含めない。
- skinned mesh、morph targetの自動生成はdeformation、skin weight、required bone、morph error fixtureへ合格したProfileだけProductionへ昇格する。不合格時はSource chainまたはLOD0へfallbackする。

### 6.2 `SpriteLodProfileV1`

Spriteは自動的に別画風または別samplingへ切り替えない。Profileは`source_variant | visibility_only | disabled`を持ち、次を守る。

- Pixel artのpoint sampling、integer scale、palette、alpha padding、pixel-locked compositionをQuality Tierより優先する。
- Gameplay cue Spriteはminimum visible size未満でも、承認済みicon／outline等の意味同等variantがない限り消さない。
- atlas page、texture mip、Sprite LODを同じindexで結合しない。
- Variant切替はstable pivot、bounds、sorting、collision非依存を検証する。

## 7. Representation LODとHLOD

### 7.1 `RepresentationLodProfileV1`

Representation tierは次のclosed enumを使用する。

```text
individual | instanced | spatial_proxy | impostor | hidden_presentation
```

`hidden_presentation`はPresentationだけを非表示にする値であり、Entity、Collision、Damage、Navigation、Save stateを削除しない。individualからinstancedへ変更してもStable Entity identityとauthoritative event routingを保持する。

### 7.2 `HlodProfileV1`

```text
HlodProfileV1
  profile_id
  eligibility_rule
  spatial_partition_profile
  cluster_limits
  proxy_mode
  proxy_geometry_profile
  proxy_material_profile
  transition_rule_set
  streaming_cell_profile
  fallback_representation
```

`proxy_mode`は`instanced | merged_mesh | simplified_mesh | impostor`とする。自動HLOD対象は次をすべて満たすSourceだけとする。

- static transform
- `decorative_instance`
- mutable Physics、個別Damage、interaction、Save、animation、root motionを持たない
- HLODを許可するVisual Style／Material契約
- bounded cell、cluster object数、vertex数、material数、texture bake budget内

Clusterはspatial cell、Profile ID、material compatibility、Source Stable ID昇順から決定論的に生成する。総配置数やSource配列順をcluster identityに使わない。

`HlodArtifactV1`はSource Stable ID集合、Source revision hash、bounds、proxy method、geometry／material key、visual error、cell、fallbackを持つ。Source、static transform、Material、Profileが変われば該当Artifactだけをinvalidateする。

HLOD proxy、impostor、occlusion proxyはGameplay Collision、Navigation、AI perceptionを所有しない。interactive objectをHLODへ混入した場合は`MIRAKAN-LOD-HLOD_INTERACTIVE_SOURCE`でCookを拒否する。

## 8. Simulation LOD

### 8.1 `SimulationLodContractV1`

Simulation LODは性能ProfileではなくGameplay契約である。TargetやQuality Tierが契約を自動的に強めてはならない。

```text
SimulationLodContractV1
  contract_id
  experience_role
  tiers[]
  relevance_inputs[]
  transition_rules[]
  wake_triggers[]
  retained_state_schema
  queued_event_policy
  authoritative_equivalence_contract
  forbidden_changes[]
  reference_fixture_id
```

Tierは`full | reduced_frequency | dormant_record`とする。

- `full`: 該当Domainの正規fixed tickと全authoritative systemを実行する。
- `reduced_frequency`: 契約したinterval、catch-up、誤差上限で処理する。Domain固有のReference simulationとのequivalence Gateに合格した場合だけ使用できる。
- `dormant_record`: typed stateとpending eventを保持し、契約したwake triggerまでactive systemから外す。Source Entity identityを失わない。

### 8.2 relevanceと遷移

`gameplay_relevance_q16`は次のauthoritative inputだけから計算する。

- player／controlled actor／goalのWorld位置と契約済みinteraction radius
- active Damage、Collision contact、Navigation goal、target lock、quest／script critical flag
- pending authoritative command／event
- Domainが登録したtyped relevance signal

Camera distance、frustum、occlusion、GPU query、screen size、Audio loudness、VFX visibilityを入力にしない。relevance計算、tier遷移、wakeはSimulation tick境界`T00`で行い、Stable ID順、fixed-point比較、minimum residency tickを使用する。

`reduced_frequency`は`exact_batch | analytic_step | event_only`のいずれかのDomain実行方式を明示する。単に複数tickをskipして同じ処理を一度だけ呼ぶ方式を禁止する。

### 8.3 fidelityとTarget

- 敵数、味方数、HP、Damage、Collision、goal参加、入力応答、spawn timingをLODで黙って変更しない。
- tier、dormant state、pending event、wake reasonはSave／Replay対象とする。
- Targetごとに別のsimulation結果を作らない。Target budgetへ収まらない場合はSourceを維持して`OptimizationRequired`にする。
- Reference simulationとReplay hash、最終count、Damage、goal結果が一致しないContractをProductionへ昇格しない。
- Gameplay変更が必要な場合は`GameplayScaleChangeProposalV1`として別ChangeSetと人間承認を必須にする。

## 9. Animation、Material、VFX

### 9.1 `AnimationLodProfileV1`

Animation LODはpresentation poseとskinning costだけを変更する。authoritative root motion、hitbox、weapon socket、foot contact、Gameplay event timingを低頻度poseから取得しない。

```text
AnimationLodTierV1
  tier_id
  pose_sample_interval_ticks
  interpolation_mode
  presentation_bone_set
  skinning_mode
  shadow_pose_mode
  required_bones[]
  required_events[]
```

required bone／eventを除外するProfileを拒否する。Animation graphのauthoritative state machineは正規Simulation tickで進め、Renderer向けpose snapshotだけを間引ける。

### 9.2 `MaterialLodProfileV1`

Material LODはoffline compile済みvariantだけを選択する。Style-critical ramp、alpha semantics、pixel sampling、combat cue emissive等を削除しない。

TierはMaterial interface hash、allowed feature mask、texture residency floor、shadow／depth participation、visual equivalence toleranceを持つ。Runtime shader source生成、任意branch削除、Material mergeをLOD selection内で行わない。

### 9.3 `VfxLodProfileV1`

```text
VfxLodTierV1
  tier_id
  semantic_priority
  branch_id
  spawn_scale_permille
  maximum_alive
  update_interval
  renderer_outputs[]
  simulation_target
  minimum_cue_contract
```

- `critical_gameplay_cue`はshape、timing、minimum visibilityを維持し、ambient effectより先にdropしない。
- VFX tierはGameplay eventの発生数、Damage、Collision、AI perceptionを変更しない。
- CPU／GPU、aggregate emitter、spawn／alive削減は同じSource VFX AssetからCookした承認済みbranchだけを使用する。
- Camera distance、projected coverage、quality、overdraw、budget pressureをPresentation入力にできるが、selection理由とdrop countをtelemetryへ記録する。

## 10. Terrain、Foliage、Water、Snow Surface、residency

### 10.1 Terrain

Terrain正式Capabilityは`TerrainLodProfileV1`にscreen-space geometry error、quadtree patch bounds、neighbor level差上限、skirt／stitch policy、material residency、streaming cellを持たせる。render patchはCollision height、Nav tile、Gameplay Surface Stateを置き換えない。

Terrain Capability着手前はSchema IDをActive Catalogへ掲載しない。C2 vertical slice、streaming、camera cut、境界crack、Collision／Nav独立fixtureに合格した後だけProductionへ昇格する。

### 10.2 Foliage

`FoliageLodProfileV1`はinstance mesh chain、wind tier、shadow tier、impostor、cluster bounds、per-cell instance上限を持つ。Gameplay Collision対象subsetはSource semantic roleから別ArtifactへCookし、描画LODまたはimpostorへ追従させない。

### 10.3 Water

`WaterLodProfileV1`はsurface patch density、wave shading、reflection、underwater presentation、foam／spray VFXのTarget別tierを持つ。CPU Surface Query、Water Volume、浮力、swimming、Damage、Navigation costはWater LODで変更しない。

### 10.4 Snow Surface

`SnowSurfaceLodProfileV1`はdynamic fieldのupdate distance、normal／sparkle detail、降雪VFX density、static mask fallbackをTarget別tierとして持つ。`GameplaySurfaceState`、friction、movement、foot contact、static coverage、Visual StyleはSnow Surface LODで変更しない。

page admissionはsemantic priority、projected coverage、distance、page keyで決定論的に解決し、camera distanceだけをidentityにしない。dynamic pageが未residentまたはbudget外の場合はstatic maskへ戻し、fallbackを持たないreceiverをProductionへ昇格しない。

### 10.5 Geometry residency

`GeometryResidencyLodPlanV1`はrequested、resident、pending、fallback level、byte cost、deadline、owner generationを持つ。

- LOD0 fallback metadataと最低一つのrenderable geometryをPackage closureへ含める。
- 要求detailが未residentの場合は、同じAsset generation内で最も近い意味同等resident levelを使用し、`MIRAKAN-LOD-RESIDENCY_MISS`を記録する。
- frame途中で別Asset generationのLODを混在させない。
- memory pressure時もactive lease、critical cue、minimum visual fidelityを破棄しない。

## 11. PresetとAI contract

### 11.1 Preset

Presetはマジックナンバー集ではなくversion付き`LodPolicyPresetV1`とする。初期semantic presetを次に限定する。

```text
hero_character
interactive_character
crowd_character
interactive_prop
small_decorative_prop
architecture
foliage
terrain
water_surface
vfx_combat_cue
vfx_ambient
pixel_art_sprite
```

Preset適用後も解決されたlevel、threshold、fidelity floor、Target差、fallbackをInspectorとAI Previewへ表示する。Preset名だけをReceiptへ保存せず、Preset versionとresolved policy hashを保存する。

### 11.2 AI Operation

AIとEditorは同じtyped Operationを使用する。

| Operation | 結果 |
|---|---|
| `operation.lod.explain` | 現在のIntent、Policy、選択理由、lock、fallback、status |
| `operation.lod.propose_policy` | Sourceを変更しない候補PolicyとBefore／After予測 |
| `operation.lod.generate_mesh` | Staging Job。Derived Artifact候補とerror report |
| `operation.lod.build_hlod` | Staging Job。cluster preview、proxy、Source集合、risk |
| `operation.lod.preview_transitions` | camera path／Target／Quality別transition traceとvisual preview |
| `operation.lod.validate` | Contract、visual、Gameplay、budget、fallback診断 |
| `operation.lod.apply_policy` | 承認済みPolicy ChangeSet。Derived Artifactを直接含めない |

AIへScreen座標、native Editor pointer、Renderer handle、raw threshold配列だけを公開しない。Stable ID、semantic role、typed field、unit、allowed range、default provenance、Diagnostic IDを返す。

### 11.3 `LodPlanPreviewV1`

Proposalは少なくとも次を表示する。

- 対象Stable IDとsemantic role
- 変更するLOD classと変更しないclass
- 各Targetのresolved tier、threshold、fallback
- triangle、draw、visible instance、GPU／CPU memory、VFX overdraw、simulation workのBefore／After
- visual diff、silhouette／animation／critical cue risk
- Gameplay fidelityへの影響有無
- generated Artifact、tool version、予測か実測か
- blocking Diagnosticと必要Approval

## 12. Editor UX

LOD InspectorはSource Intent、resolved Target plan、runtime telemetryを混在編集させない。

1. **Intent**: semantic role、priority、fidelity floor、Preset、lock
2. **Policy**: Domain別level、metric、transition、fallback
3. **Preview**: camera path、resolution、FOV、Target、Quality、wireframe、HLOD cluster、VFX cue
4. **Diagnostics**: non-monotonic、missing fallback、unsupported backend、Gameplay risk
5. **Evidence**: Artifact hash、visual diff、performance trace、Qualification Receipt

Scene ViewはLOD coloration、forced tier、transition band、projected error、HLOD source／proxy、simulation tierを別overlayとして表示する。forced tierはEditor preview専用であり、Shipping Sourceへ保存しない。

Basic Workspaceはsemantic presetとPreviewを中心にし、Advanced Workspaceは全typed fieldを表示する。どちらも同じ`LodIntentV1`と`LodPolicySetV1`を編集する。

## 13. Runtime phase、determinism、telemetry

| Phase | LOD処理 |
|---|---|
| `T00` | authoritative relevance、Simulation tier遷移、wake／dormancy |
| Domain fixed tick | 選択済みSimulation tierの契約済み処理 |
| `R10` | 同一generation内のready LOD／HLOD Artifact promotion |
| `R20` | ViewごとのMesh／Representation／Material／VFX／Surface LOD選択 |
| `R70` | last-use完了Artifactとtransition stateのretire |

Presentation LODはRuntime Worldへ書き戻さずSave／Replay hashへ含めない。Simulation LODはauthoritative stateでありSave／Replayへ含める。

Telemetryは少なくとも次を持つ。

- class／tier別authored、resident、active、visible count
- object／Domain別transition count、境界往復、minimum residency違反
- requested／resident geometry、residency miss、upload latency
- triangle、draw、instance、material variant、HLOD cluster／proxy
- simulation tier、wake reason、queued event、catch-up cost
- VFX critical／ambient別drop、spawn、alive、overdraw
- fallback reason、unsupported Capability、capacity overflow

## 14. Validationとfailure

| Condition | 結果 |
|---|---|
| 非単調level／threshold、NaN／Inf、unit不一致 | ChangeSet／Cook拒否 |
| fallback closure欠落 | Package promotion拒否 |
| interactive／mutable objectのHLOD混入 | HLOD Cook拒否 |
| generated meshのvisual／deformation error超過 | candidate破棄、Source chain／LOD0維持 |
| GPU selector容量超過／Backend fault | 次frameからCPU direct fallback、Diagnostic |
| LOD Artifact未resident | 同generationのresident fallback、miss記録 |
| simulation wake event欠落／queue overflow | hard failure。eventをdropして継続しない |
| Reference simulation不一致 | Simulation LOD非Promotion |
| critical VFX cue floor未達 | Policy非Promotion |
| Target budget未達 | Source維持、`OptimizationRequired` |
| Gameplay変更でのみ合格可能 | `GameplayScaleChangeProposalV1`と人間承認 |

代表Diagnostic IDを次に固定する。

```text
MIRAKAN-LOD-SCHEMA_UNKNOWN
MIRAKAN-LOD-SEMANTIC_ROLE_CONFLICT
MIRAKAN-LOD-NON_MONOTONIC
MIRAKAN-LOD-MISSING_FALLBACK
MIRAKAN-LOD-UNSUPPORTED_TRANSITION
MIRAKAN-LOD-GENERATION_ERROR_LIMIT
MIRAKAN-LOD-HLOD_INTERACTIVE_SOURCE
MIRAKAN-LOD-SIMULATION_VISIBILITY_INPUT
MIRAKAN-LOD-SIMULATION_EQUIVALENCE
MIRAKAN-LOD-CRITICAL_CUE_FLOOR
MIRAKAN-LOD-RESIDENCY_MISS
MIRAKAN-LOD-CAPACITY_EXCEEDED
MIRAKAN-LOD-TARGET_UNQUALIFIED
```

## 15. TestとQualification

### 15.1 Contract

- MCDからC++、TypeScript、binary descriptor、MCP schemaを生成し、unknown field／enum／majorを拒否する。
- unit、range、monotonic、fallback closure、Preset version、policy lockのpositive／negative fixtureを持つ。
- Human、AI、headless CLIが同じIntentから同じPolicy hashとPlan hashへ収束する。

### 15.2 Mesh／Renderer

- FOV、orthographic／perspective、resolution、dynamic extent、camera cut、split Editor Viewで`projected_error_px_q16`のgolden値を照合する。
- CPU directとGPU indirectが同じinputで同じtierを選び、境界包含とhysteresisが一致する。
- silhouette、normal、UV seam、vertex color、material interface、skinning、morph、shadowのvisual error fixtureを持つ。
- camera pathを往復し、transition thrash、pop、cross-fade overdraw、residency missを測定する。

### 15.3 HLOD／Streaming

- Source順序を変えてもcluster、Artifact hash、proxy boundsが一致する。
- interactive、mutable Physics、Save、animation objectの混入を拒否する。
- cell境界、teleport、camera cut、Artifact promotion、memory pressureを同一区間で発生させる。
- HLOD on／offでGameplay Collision、Nav、Damage、Save、Replay結果が一致する。

### 15.4 Simulation

- Full referenceと各Production tierでReplay hash、最終count、Damage、goal、pending eventが一致する。
- enter／exit境界、minimum residency、wake trigger同時発生、event queue上限、Save／Load直後を検証する。
- camera、frustum、occlusion、VFX visibilityを変更してもSimulation tier結果が変わらない。
- Target／Quality Profileを変えてもauthoritative結果が一致する。

### 15.5 Domain

- Animation required bone、root motion、hitbox、event timingを全tierで照合する。
- VFXはcritical cue floorを維持し、Gameplay event数へ影響しない。
- Pixel artはpoint sampling、integer scale、palette、pixel-locked layerを維持する。
- Terrain／Foliage／WaterはRender LODをCollision、Nav、CPU Queryへ逆入力しない。

### 15.6 Integrated Gate

Project固有`IntegratedScaleFixtureV1`はcamera移動、LOD／HLOD遷移、spawn burst、Physics／Navigation／Animation、敵味方VFX、streaming、Asset promotionを同一runで発生させる。120秒×5、10分soak、Target実機Gateでframe、memory、visual、Replay、critical cueを測定する。

`LodQualificationReceiptV1`は次を持つ。

```text
receipt_id
plan_hash
artifact_hashes[]
target_profile
device／driver
fixture_id／fixture_hash
camera_path_hash
quality_profile
before_metrics
after_metrics
visual_diff_metrics
gameplay_replay_hash
fallback_events[]
diagnostics[]
result
toolchain_hash
timestamp
```

## 16. 成熟度と段階導入

| 段階 | 成果物 | Promotion Gate |
|---|---|---|
| C0 | MCD、Intent／Policy／Plan、Validator、CPU reference metric、headless Preview／Diagnostic | Schema round-trip、golden metric、negative fixture |
| C1 2D | Sprite policy、CPU VFX LOD、critical cue floor、Full／dormant Simulation contract | 2D crowded battle、Replay一致、pixel style維持 |
| C1 3D | Manual／generated static Mesh LOD、CPU selector、hysteresis、Animation presentation LOD、LOD0 fallback | 3D arena、visual error、CPU budget、deterministic Cook |
| C2 | GPU selector、HZB連携、HLOD、geometry streaming、generated skinned／morph Gate、VFX GPU、Water、Snow Surface、Terrain／Foliage、契約済みreduced Simulation | CPU／GPU一致、open-world fixture、Domain equivalence、全Vendor baseline |
| C3 | virtualized geometry、large-world multi-level HLOD、Domain固有aggregation LOD | 個別ADR、portable fallback、実機Qualification |

Product Phaseとの対応を次に固定する。

1. Phase 0はLOD Runtime／Rendererを実装しない。将来Schema IDをActive Catalogへ偽掲載しない。
2. Phase 1で`LodIntentV1`、Policy envelope、ChangeSet validation、headless explanationを実装する。
3. Phase 2でCPU reference metric、Preview trace、Diagnostic projectionを実装する。
4. Phase 3で2D Sprite／VFX／Full・dormant SimulationのC1 vertical sliceを合格させる。
5. Phase 6で3D static Mesh／Animation／Water／static Snow SurfaceのC1 vertical sliceを合格させる。
6. Phase 8でGPU selector、HLOD、streaming、dynamic Snow Surface、Terrain／Foliage、C2 SimulationをCapabilityごとに昇格する。

## 17. Definition of Done

LOD設計は次をすべて満たした時だけ実装完了とする。

- AIと人間が同じtyped Intent／Policy／Operationを使用する。
- 全Domainが共通`lod_index`ではなく、登録済みDomain planを使用する。
- Manual／generated Mesh LOD、HLOD、Simulation、Animation、Material、VFX、Surface、ResidencyのOwnerとfallbackが閉じている。
- CPU／GPU selector、Cook、Preset、Previewが決定論的fixtureへ合格する。
- Presentation LODからGameplayへの逆入力がない。
- Simulation LODがReference simulation、Save、Replay、Target一致へ合格する。
- critical cue、Visual Style、pixel-art契約を維持する。
- Target別Before／Afterとfallbackを持つ最新Receiptなしに`Qualified`と表示しない。

## 18. 一次資料

- [Unity 6.5: Introduction to level of detail](https://docs.unity3d.com/Manual/LevelOfDetail.html)
- [Unity 6.5: LOD Group component reference](https://docs.unity3d.com/Manual/class-LODGroup.html)
- [Unreal Engine 5.8: Static Mesh Automatic LOD Generation](https://dev.epicgames.com/documentation/unreal-engine/static-mesh-automatic-lod-generation-in-unreal-engine)
- [Unreal Engine 5.8: World Partition HLOD](https://dev.epicgames.com/documentation/en-us/unreal-engine/world-partition---hierarchical-level-of-detail-in-unreal-engine)
- [Unreal Engine 5.8: Mass Gameplay](https://dev.epicgames.com/documentation/en-us/unreal-engine/overview-of-mass-gameplay-in-unreal-engine)
- [Unreal Engine 5.8: Nanite Virtualized Geometry](https://dev.epicgames.com/documentation/en-us/unreal-engine/nanite-virtualized-geometry-in-unreal-engine)
- [Godot 4.6: Mesh level of detail](https://docs.godotengine.org/en/4.6/tutorials/3d/mesh_lod.html)
- [Godot 4.6: Visibility ranges](https://docs.godotengine.org/en/4.6/tutorials/3d/visibility_ranges.html)

外部Engineは、screen-space metric、Preset、自動生成、HLOD、Simulation分離が実用上成立するEvidenceとして参照する。Miraikanaiの公開API、Asset、Editor UI、fidelity、Approval、Qualificationは本書の独自契約を正本とする。
