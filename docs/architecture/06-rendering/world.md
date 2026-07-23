# Miraikanai Engine World／Scene／Space／Cell Contract

- 文書ID: mirakan.arch.rendering-world
- 状態: review
- 正本範囲: World／Scene／Spaceのsource identity、global composition、persistent entity、optional spatial topology、Cellのplan-local identity、partition／streaming-plan authoring、spatial transition／Loading presentation、Tilemap、Engine-native Blockout、procedural source、Map要求resolution
- 非正本範囲: consumer-owned Gameplay progression、Runtime cell phase／shared capacity、ECS／Gameplay component schema、Physics／Navigation behavior、Render／LOD execution、Asset transaction、Save／Replay envelope、AI authorizationは各Ownerを参照
- 依存: [Product Plan](../00-product/product-plan.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[Collision](../05-simulation/collision.md)、[Physics](../05-simulation/physics.md)、[Navigation](../05-simulation/navigation.md)、[Animation](../05-simulation/animation.md)、[Render Graph](render-graph.md)、[LOD](lod.md)
- 外部根拠検証日: 2026-07-23

## 1. 結論と所有境界

Worldは空間、Scene、global composition、persistent entity、任意のspatial topologyだけを所有する。Gameplay goal、outcome、spawn、進行単位はconsumer-owned stateであり、World activationへ必須にしない。

World activation、Scene activation、Cell streamingはGameplay goalやResultを要求しない。Scene 0件のprocedural-only World、spatial topologyなしのUI補助World、有限Gameplay進行を持たないcontinuous simulationをvalidとする。

AI、Editor、Project C++はSource Documentとtyped change primitiveを扱い、Runtime cell object、ECS pointer、Renderer resource、Physics／Navigation native handleを直接保存しない。cell activation state／phase／lifetimeはRuntime、capacity／backpressureはPerformance、representation selectionはLODが所有する。

## 2. 正規用語とidentity

- `World`: Project内の空間／content universeとglobal composition root。
- `Scene`: authoring、ownership、collaborationのための再利用可能なsource shard。
- `Space`: spatial topology上の場所または区画。Gameplay progression単位ではない。
- `Cell`: streaming、residency、activation planの空間／logical partition unit。
- `Entity source`: Stable IDを持つWorld content record。Runtime entity instanceではない。
- `Layer`: visibility／authoring／variant selectionのorthogonal grouping。
- `Map`: user request語でありcanonical object typeではない。resolverがWorld structure／Scene composition／Space／navigation／presentationへ分類する。

World／Scene／Space／Entity source identityはProject Stable ID、source revision、display labelを分離する。path、filename、display name、array indexをidentityにしない。CellはSource Stable IDを持たずPlan-local `uint32`であり、Scene、Space、Cell、Runtime chunkを同一視しない。

## 3. 「Map」要求の解決規則

`MapIntentKindV1`は次の6 kindへ閉じる。

| Kind | 例 | 変更対象 |
|---|---|---|
| `world_structure` | 地域接続、町からDungeonへ移動 | `SpatialTopologyDefinitionV1` |
| `scene_composition` | 複数Sceneの配置、再利用source shard | `SceneDocumentV1`＋World composition |
| `streaming` | seamless load、遠方軽量化 | `SpatialPartitionIntentV1`＋Derived Plan |
| `procedural_layout` | seed生成Dungeon | `ProceduralWorldDefinitionV1` |
| `navigation` | 歩行領域、飛行経路、NavMesh | [Navigation](../05-simulation/navigation.md) |
| `map_presentation` | minimap、world map、marker、fog | `MapPresentationDefinitionV1` |

```text
MapIntentResolutionV1
  request_id: UUIDv7 StableId
  candidate_kinds: bounded array[1..6]<{
    kind: MapIntentKindV1,
    confidence_q16: uint16,
    evidence_requirement_ids: bounded array[1..64]<exact requirement ID>
  }>
  selected_kind: optional MapIntentKindV1
  confidence_q16: uint16
  evidence_requirement_ids: bounded array[1..64]<exact requirement ID>
  affected_stable_ids: bounded array[0..1024]<existing exact StableId>
  blocking_questions: bounded array[0..7]<{
    question_id: StableId,
    choices: bounded array[2..5]<{choice_id, impact, changeability}>,
    recommended_choice_id: exact choice_id
  }>
  disposition: resolved | question_required | rejected
```

`candidate_kinds`は`confidence_q16`のscore降順、同点時は`MapIntentKindV1`のclosed enum順でcanonicalizeし、同じkindの重複を許さない。すべての`confidence_q16`は0～65,535で、top-level値は先頭候補の値と一致させる。各Evidence配列は1～64件で重複不可とし、`affected_stable_ids`はResolution対象のexact Project revisionに実在することをGatewayが検証する。unknown／deleted／revision不一致のIDを候補へ補正せず、`MIRAKAN-WORLD-UNKNOWN_STABLE_ID`で拒否する。

`selected_kind`と`disposition`はtagged ruleである。`resolved`は先頭候補と同じ`selected_kind`を厳密に1件、`blocking_questions`を0件持つ。`question_required`は`selected_kind`を持たず、選択肢2～5件、推奨、影響、後からの変更可能性を備えたQuestionを1～7件持つ。`rejected`は`selected_kind`とQuestionを持たない。上位2候補差が9,830未満、Save／spatial transition／authoritative state／Target／Asset license／memory capacityへ影響、layoutとpresentationの両解釈、Stable IDを特定不能のいずれかなら`question_required`とする。

曖昧な「マップを作る」「マップを開く」に万能Map assetを生成しない。空間contentはWorld／Scene／Space、source shard構成はScene composition、pathfindingはNavigation、画面表示はMap Presentationへroutingする。

## 4. Source Document model

`WorldDocumentV1`は次のroot referenceだけを所有する。

```text
WorldDocumentV1
  world_id
  scene_document_refs[0..65535]
  global_composition_refs[]
  persistent_entity_refs[]
  spatial_topology_definition_ref: SpatialTopologyDefinitionV1 | null
```

`scene_document_refs[]`は0件を許可し、procedural-only Worldをvalidとする。`spatial_topology_definition_ref`は0または1件であり、Gameplay progressionを暗黙生成しない。WorldへAsset binary、Navigation mesh、HLOD mesh、Streaming payload、C++ source、Vendor objectを埋め込まない。

`SceneDocumentV1`は`scene_id`、optional `document_bounds`、`entity_refs[0..1048576]`、`space_membership_refs[]`、`layer_refs[]`、`source_dependency_refs[]`、`edit_ownership`を持つ。Scene境界はRuntime streaming境界を強制せず、Cookerは複数Sceneを一Cellへまとめることも一Sceneを複数Cellへ分割することもできる。

Entity sourceはStable ID、Transform source、parent ref、owner-typed component document refs、Layer／tag、authoring metadata refを持つ。Gameplay／Physics／Navigation／Rendering fieldは各Ownerを参照し、World共通recordへflattenしない。

## 5. Spatial topology

```text
SpatialTopologyDefinitionV1
  space_nodes[]
  transition_edges[]
  activation_entry_refs[]
```

`space_nodes[]`はStable Space ID、optional parent、spatial boundsまたはtopological location、`intentionally_isolated`を持つ。parent graphはDAGとする。`activation_entry_refs[]`は0件を許可し、default entryや到達可能なGameplay goalを要求しない。

`transition_edges[]`はsource／target Space、anchor refs、direction、condition policy ref、activation hint、fallback、typed `transfer_subject_refs[]` schema refを持つ。`transfer_subject_refs[]`はCharacter、Player、Partyへ固定せず、Entity、camera observer、simulation agent、board token等のregistered subject contractを参照できる。bidirectional edgeはCook時に二つの正規有向edgeへ展開する。

entryから到達不能なSpaceは`intentionally_isolated=true`を必須にする。missing ref、duplicate edge identity、parent cycle、孤立宣言なしの到達不能Space、fallbackなしのTarget非対応edgeを`MIRAKAN-WORLD-TOPOLOGY_INVALID`で拒否する。Topology activationはScene／Cell dependency closureだけを解決し、consumer-owned outcomeを評価しない。


## 6. Spatial Partitionとstreaming-plan authoring

`SpatialPartitionIntentV1`はCreatorが編集するSourceであり、`partition_intent_id`、厳密に1件の`world_ref`、`spatial_dimension: 2d | 3d`、`interest_source_contract_refs[1..128]`、physical unitとhysteresisを明示する`activation_radius_policy`、together／separate Stable ID set、1～128件のtyped ordered `priority_rules`、0～4,096件の`always_resident_refs`、Stable Entity／Layer／Sceneの`streamable_refs`、Targetごとに厳密に1件の`target_budget_refs`、typed `failure_policy`を持つ。Cell size、chunk file名、GPU heap offset、Backend page IDをSource Intentへ固定しない。

各`interest_source_contract_ref`はowner登録済みのtyped contractへexactに解決し、observer、entity、camera、portal observer、scripted anchor、simulation agent等を提供できる。Player／Party／Characterをclosed enumまたは必須sourceにせず、unknown、duplicate、owner unavailable／removed、dimension不一致を`MIRAKAN-WORLD-PARTITION_INVALID`で拒否する。Runtimeはcontractが返すgeneration付きposition／bounds／priorityだけを消費し、owner stateをWorldへ複写しない。

`WorldStreamingPlanV1`はCookerが生成するDerived Artifactであり、Editor／AIは直接編集しない。fieldは`plan_id`、`source_world_revision`、`contract_set_hash`、`target_profile_id`、`toolchain_manifest_hash`、`partition_intent_hash`、`cell_descriptors[]`、`dependency_edges[]`、`activation_groups[]`、`residency_budget`、`canonical_priority_order`、`fallback_plan_ref`、`artifact_hash`である。`residency_budget`の定義と解決値はRuntime capacity ownerを参照する。

`plan_id`は`SHA-256("mirakan.world_streaming_plan.v1" || source_world_revision || contract_set_hash || target_profile_id || toolchain_manifest_hash || partition_intent_hash)`で生成する32 byte `DerivedPlanId`である。`artifact_hash`はcanonical Plan bytesのSHA-256であり、`plan_id`と置換しない。`fallback_plan_ref`はexact `{plan_id, artifact_hash}`を使う。

`CellDescriptorV1`を次へ固定する。CellのSource Stable IDを新設せず、Plan-local IDとSource identityを分離する。

| Field | 規則 |
|---|---|
| `cell_id` | Plan内canonical orderから1..Nを割り当てる`uint32`。0 invalid、Source identity／Save／別Plan比較へ使用しない |
| `source_refs` | Entity／Scene／Layer／Spaceのexact Stable ref、1～1,048,576件 |
| `bounds` | finite 2D AABBまたは3D AABB |
| `cpu_bytes`／`gpu_bytes`／`asset_bytes` | 各`uint64`、実Artifact size |
| `physics_bytes`／`navigation_bytes` | 各`uint64` |
| `hard_dependency_cell_refs` | 0～4,096件、Plan全体でDAG |
| `soft_dependency_cell_refs` | 0～4,096件、各refにfallback必須 |
| `activation_group_id` | 厳密に1件 |
| `priority_key` | Source policyから生成したcanonical fixed-width key |
| `payload_hash` | SHA-256 |

一Planは1～1,048,576 Cell、1 activation groupは1～4,096 Cell、Plan全体の`source_refs`合計は16,777,216件を上限とする。Target ProfileのArtifact byte capacityを満たせない場合はPlan generationを失敗させ、Cellを欠落させない。Cell descriptorは生pointer、Vendor resource handle、absolute pathを持たない。

Cell候補をcanonical finite `bounds.min.x, min.y, min.z, bounds.max.x, max.y, max.z`、次にUUID byte順の`source_refs` Merkle root、最後に`payload_hash`のbyte順でsortし、1から`cell_id`を割り当てる。2Dのzは`+0`、同一sort keyは重複Cellとして拒否する。Activation Groupは所属`cell_id`昇順列のlexicographic順で1から`activation_group_id`を割り当てる。同じSource revision、Target Profile、Toolchain Manifestから同じPlan hashを生成し、非決定要因を使ったCookを失敗させる。

共通memory、I/O、worker、queue budget、measurement、backpressureは[Runtime performance／capacity](../04-runtime/performance-capacity.md)だけが所有する。Planはcapacity classとfallback intentを参照するだけで数値を複写しない。Runtime OrchestratorだけがPlanからCell residency／activationを実行する。

Runtime-owned Cell lifecycleの`failed`状態、retry／rollback／evict遷移は[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)へ委譲し、World Source／`CellDescriptorV1`のfieldとして保存しない。

## 7. Spatial transition intent

`SpatialTransitionPolicyV1`はpresentation policy ref、persistent subject policy ref、required activation set policy ref、precondition ref、failure policy、cancel policyを持つ。実行phase、writer、async job、timeout、Save checkpointはRuntime Ownerへ委譲する。

World ownerはStageその他のconsumerが再定義しないexact destination型を次へ固定し、MCDへ`{id=type.world.spatial_transition_destination, version=1, contract_set_hash}`として登録する。

```text
SpatialTransitionDestinationV1
  destination_id: StableId
  destination_version: uint32
  destination_hash: SHA-256(self excluding destination_hash)
  world_ref: exact WorldDocumentRef(stable_id, schema_version, content_hash)
  topology_ref: exact SpatialTopologyDefinitionRef(version, content_hash)
  transition_edge_ref: exact TransitionEdgeRef(version, content_hash)
  target_space_ref: exact SpaceRef(version, content_hash)
  target_anchor_ref: AnchorRef(version, content_hash) | null

SpatialTransitionDestinationRefV1
  destination_id
  destination_version
  destination_hash
  destination_type_ref:
    McdContractRefV1(id=type.world.spatial_transition_destination, version=1, contract_set_hash)
```

World、Topology、Edge、target Spaceは同じWorld／Topology revision closureへ属し、Edgeのsource／targetと`target_space_ref`、各content hashが一致しなければならない。Edgeの`target_anchor_requirement=forbidden | optional | required`を唯一のdiscriminatorとし、forbiddenはnull、optionalはnullまたは同target SpaceのAnchor、requiredはnon-null exact Anchorを要求する。表示名、path、current World、latest Space／Edge、別Worldの同名Anchorへ再解決しない。

`SpatialTransitionRequestV1`は`request_id`、source Space instance ref、exact `SpatialTransitionDestinationRefV1`、requesting system ref、requested tick、typed `transfer_subject_refs[]`、precondition snapshot hash、transition policy ref／hashを持つ。Edge／Space／AnchorをRequestへ複製しない。表示名から対象を再解決せず、stale World／Topology／Edge／Space／Anchor、hash mismatch、inactive sourceをprefetch前に拒否する。

transition中に旧／新SpaceのEntity identityを再利用しない。persistent identityは明示ownerとhandoff recordを持つ。target dependency不足時はpartial activationやdefault destinationへ進まず、blocking reasonとfallbackを返す。

### 7.1 Loading／prefetch presentation

`SpatialTransitionPresentationPolicyV1`は`mode: seamless | overlay | blocking`、Loading UI、Input context、Audio snapshot、0～2,000 msのminimum display、cancel boundary、failure presentation、explicit retry、Accessibility announcement policyを持つ。

```text
LoadingProgressPlanV1
  plan_id: DerivedPlanId
  subject_kind: initial_activation | spatial_transition | resume_save
  subject_request_ref: exact generation-bearing typed request ref
  selected_dependency_binding_refs[0..65535]: WorldDependencyBindingRefV1
  dependency_closure_hash: bytes32
  work_units: bounded array[1..65,535]<{
    kind: io_bytes | artifact_verify | dependency_ready | activation_group | state_transfer,
    exact_target_ref: exact generation-bearing typed ref,
    positive_weight_q16: uint16 in 1..65,535
  }>
  total_weight_q16: 65535

LoadingProgressSnapshotV1
  loading_session_id: generation-bearing runtime ID
  progress_plan_ref: exact {plan_id, dependency_closure_hash, generation}
  phase: validating | prefetching | verifying | resident | activating | transferring | complete | failed | cancelled
  completed_weight_q16: uint16
  current_item_message_key: localization key
  can_cancel: bool
  can_retry: bool
  failure_reason: optional typed failure
  generation: uint64

WorldDependencyBindingRefV1
  binding_id
  binding_version
  binding_hash
  owner_ref
  owner_hash
  dependency_kind: artifact | game_system | service
  dependency_ref
  dependency_content_hash
  dependency_generation
  readiness_policy_ref: McdContractRefV1(kind=policy)
```

`subject_kind`と`subject_request_ref`はtagged ruleで、initial activation、spatial transition、Save resumeの選択kindとexact request kind／generationを一致させる。dependency bindingはbinding ID／version順でstrict sortし、Source／Target選択から生成した集合とset equalityにする。work unit kindは上記5種だけで、各unitはexact対象refと正のweightを必須とする。確定したdependency closureのcanonical dependency順、同一dependency内のkind enum順、target ref canonical byte順で並べ、同じ`{kind, exact_target_ref}`を重複させない。positive weight合計は厳密に65,535とする。65,535 unitは全weightが正で正規化可能なら受理し、65,536 unit以上は`MIRAKAN-WORLD-LOADING_PLAN_CAPACITY_EXCEEDED`としてPlan materialization前に拒否する。省略、結合、truncateで成功へ近似せず、previous valid PlanとWorld generationを維持してpartial Plan／unitを公開しない。

`completed_weight_q16`は0～65,535で、同じexact Plan generationでは完了済みunitのweightだけを加算して単調非減少とする。`phase=complete`は`completed_weight_q16=65,535`かつ`can_cancel=false`かつ`can_retry=false`、`phase=failed`は`failure_reason`を厳密に1件、その他phaseは0件とする。`can_cancel=true`はPolicyが許可した`prefetching | resident`だけ、`can_retry=true`はtyped failureとexplicit retry policyがある`failed`だけである。Snapshotのsession generation、`progress_plan_ref.generation`、`generation`は一致し、fake timer、frame count、UI animation、spinner、未列挙jobを進捗へ混ぜない。

Source、Target、Toolchain、partition intentのいずれかのhashが変わったPlanは`MIRAKAN-WORLD-STREAMING_PLAN_STALE`、dependency closureまたはgenerationが変わったProgress Plan／Snapshotは`MIRAKAN-WORLD-LOADING_PROGRESS_PLAN_STALE`で拒否する。barを巻き戻して古いgenerationを再利用せず、新しいPlan、request、session、generationを発行する。

I/O／verifyはreal-time domainで進められるが、activationとtyped subject transferは正規tick boundaryでatomic commitする。選択Source／Targetが明示した`selected_dependency_binding_refs[]`だけをhard dependency closureへ入れ、各bindingのowner、Artifact／System ref、generation、readiness predicateをexactに検証する。Renderer、Collision、Navigationその他のOwnerを名前で常時追加しない。選択bindingがall-readyになった後だけactivation group全体をpublishする。一部だけ成功した場合は`MIRAKAN-WORLD-ACTIVATION_PARTIAL`としてtarget全体をrollbackし、source Spaceとlast-valid generationをactiveのまま維持する。旧Spaceを先に破棄して空Worldを露出せず、target active後にsubjectをtransferし、その後にsourceをdeactivateする。

Cancelはpolicyが許可しphaseが`prefetching | resident`以下の時だけ受理し、activating以後または不許可時は`MIRAKAN-WORLD-LOADING_CANCEL_REJECTED`としてWorld state不変のtyped rejectionを返す。受理後はinflight I/Oをcancelまたはbounded drainし、leaseとtemporary Artifactを解放する。Retryは明示操作だけで新request／session identityを発行し、Source revision、condition、artifact checksum、Target Capability、storage／memory budgetを再検証する。不合格は`MIRAKAN-WORLD-LOADING_RETRY_REVALIDATION_FAILED`とし、partial activation、古いSnapshot、古いprogressを使用しない。Loading UI、Audio、読み上げはauthorityへ逆入力しない。


## 8. 参照と依存closure

全World referenceはStable ID、expected document kind、required／optional、version compatibilityを持つ。CookerはScene nesting、Entity parent、Space topology、Cell membership、Domain component asset、transition targetのclosureをcanonical orderで解決する。

required ref欠損、kind mismatch、cycle、duplicate owner、stale revisionはhard errorとする。optional ref欠損はSourceでfallbackが宣言された場合だけ許可する。Runtimeはdisplay nameやpathからrefを再解決しない。

Asset artifact、Navigation artifact、LOD／HLOD representation、Renderer material／geometryはgeneration付きrefでPlanへ入り、異なるsource／artifact generationを一つのCell activationへ混在させない。

## 9. Procedural World source

Procedural Worldはgenerator Stable ID、typed parameter、seed semantics、input asset refs、bounded output scope、determinism class、generated Source ownership、regeneration／migration policyを持つ。GeneratorはProject Source DocumentへのChangeSetを生成し、Runtime objectやnative resourceを直接生成して正本化しない。

`ProceduralWorldDefinitionV1`は`procedural_world_id`（UUIDv7 Stable ID）、positive `definition_version`、exact `owner_ref={owner_id,owner_revision,owner_content_hash}`、exact `generator_contract_ref`、Qualified Definition／Native variantの`generator_implementation_ref`、`seed_policy: fixed | project_parameter | save_slot | session_derived`、`input_definition_refs` 0～1,024件、`layout_constraint_refs` 1～1,024件、exact `output_schema_ref`、正の`max_generation_steps`／`max_output_entities`、Targetごとに厳密に1件の`time_memory_budget_refs`、`determinism_contract_ref`、`selected_validation_provider_bindings[0..64]`、`failure_policy: retry seed | fallback layout | abort`、self-excluding `definition_content_hash`を持つReceipt-free Sourceである。`definition_content_hash`はASCII `MIRAKAN_PROCEDURAL_WORLD_DEFINITION_V1`と自己Fieldだけを除くlength-framed canonical bytesから計算する。Qualification Receipt／BindingをDefinitionまたはcontent hashへ含めない。budget値はRuntime capacity ownerを参照する。Fixture bodyはowner-typed Procedural World Qualification subjectだけが解決し、Production Definition／Runtime Packageはroot外Activation projectionからReceiptのsubject／Target／result／freshnessだけを検証する。

```text
ProceduralValidationProviderBindingV1
  validation_kind:
    physics_overlap | spawn_safety | navigation_query | owner_extension
  binding_document_ref:
    exact owner-typed ProviderBindingDocumentRef including content_hash
  resolved_binding_closure_hash: SHA-256
  provider_contract_ref: McdContractRefV1
  output_schema_ref: McdContractRefV1(kind=type)
  required_output_count: uint32
  owner_ref
  owner_hash

ProceduralWorldDefinitionRefV1
  procedural_world_id: StableId
  definition_version: positive uint32
  definition_content_hash: SHA-256

BlockoutAssemblyRefV1
  assembly_id: StableId
  assembly_version: positive uint32
  assembly_content_hash: SHA-256

WorldAuthoringBundleRefV1
  bundle_id: StableId
  bundle_version: positive uint32
  bundle_content_hash: SHA-256

WorldSourceQualificationSubjectRefV1
  subject_kind: procedural_world | blockout | world_authoring_bundle
  subject_ref:
    procedural_world: ProceduralWorldDefinitionRefV1
    | blockout: BlockoutAssemblyRefV1
    | world_authoring_bundle: WorldAuthoringBundleRefV1

WorldSourceQualificationSubjectV1
  qualification_id
  qualification_version: positive uint32
  owner_ref: exact {owner_id, owner_revision, owner_content_hash}
  subject: WorldSourceQualificationSubjectRefV1
  target_profile_refs[1..64]
  fixture_refs[1..1024]:
    exact {fixture_id, fixture_version, fixture_content_hash}
  input_closure_hash: SHA-256
  result: pass | fail
  qualification_subject_hash: SHA-256

WorldSourceQualificationReceiptV1
  subject: WorldSourceQualificationSubjectV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=world_source_qualification)

WorldSourceQualificationReceiptRefV1
  qualification_id
  qualification_version: positive uint32
  qualification_subject_hash: SHA-256
  signed_record_hash: SHA-256

WorldSourceActivationBindingRefV1
  activation_binding_id
  activation_binding_version: positive uint32
  activation_binding_hash: SHA-256

WorldSourceActivationBindingV1
  activation_binding_id
  activation_binding_version: positive uint32
  subject: exact WorldSourceQualificationSubjectRefV1
  qualification_receipt_refs[1..1024]:
    WorldSourceQualificationReceiptRefV1
  activation_binding_hash: SHA-256

WorldSourceActivationProjectionV1
  projection_id
  projection_version: positive uint32
  entries[1..4096]:
    exact {WorldSourceQualificationSubjectRefV1,
           WorldSourceActivationBindingRefV1}
  projection_hash: SHA-256
```

binding配列は`validation_kind`、binding Document Stable IDのcanonical byte順でsortし、duplicateを拒否する。Document bytesのidentityは`binding_document_ref.content_hash`だけが表し、resolved closureを同じhashへ多重定義しない。`resolved_binding_closure_hash`は`SHA-256(ASCII "MIRAKAN_WORLD_PROCEDURAL_BINDING_CLOSURE_V1" || validation_kind || binding_document_refの全canonical bytes || owner_ref/hash || provider_contract_ref || output_schema_ref || required_output_count)`である。Document content hashまたはclosure hashの一方だけが一致しても受理せず、owner、provider、output schemaをlatestへ再解決しない。`required_output_count`は0～1,024で、0はprovider実行を要求するが出力recordを要求しないvalidatorだけに使う。Owner名、Capabilityの存在、output field名からproviderを推測選択しない。

全subject kindの生成順は`receipt-free Source／Bundle → base ref → WorldSourceQualificationSubjectV1 → signed Receipt → WorldSourceActivationBindingV1 → root外Activation projection`である。三base Refは同じID／positive version／self-excluding content hashを持つ完成baseへexact一件解決し、latest fallbackや別kind substitutionを許可しない。subject hashはASCII `MIRAKAN_WORLD_SOURCE_QUALIFICATION_SUBJECT_V1`、binding hashはASCII `MIRAKAN_WORLD_SOURCE_ACTIVATION_BINDING_V1`、projection hashはASCII `MIRAKAN_WORLD_SOURCE_ACTIVATION_PROJECTION_V1`と各自己Fieldを除くcount／length-framed canonical bytesから計算する。subject ownerはbranchごとにRefが解決する`ProceduralWorldDefinitionV1.owner_ref`、`BlockoutAssemblyV1.owner_ref`、`WorldAuthoringBundleV1.owner_ref`とbyte equality、Binding subjectはReceipt subjectとbyte equalityでなければならない。Projection `entries[]`はsubject kindのclosed ordinal、subject logical ID／version／content hash、Binding ID／version／hash順へstrict sortし、duplicate subject／Binding refと同じsubjectへの複数Bindingを拒否する。Production Source、Plan、BundleはFixture body、Receipt、Bindingを自身のhash preimageへ含めず、root外Projectionが指すsigned Receiptだけを検証する。Fixture集合の変更は新subject／Receipt／Binding／Projectionを発行し、Production hashへFixture refまたはReceipt refを混入しない。discriminator外Ref、ID／version／hash欠落、正しいbase refのままsubject ownerだけを差し替えるcase、Binding subjectだけを別baseへ差し替えるcase、Projection entryのduplicate／same-subject別Binding／順序違反を一原因で拒否する。

UUID割当前のgenerator出力は次のimmutable `GeneratedWorldSemanticCandidateV1`である。`candidate_id`やStable IDを持たず、exact Project tripleとcontent hashだけで参照する。

```text
GeneratedWorldSemanticCandidateV1
  candidate_project_ref:
    {project_id, expected_project_revision, document_set_hash}
  procedural_world_ref: exact ref/revision/content hash
  generator_contract_ref/hash
  generator_implementation_ref/hash
  input_revision_refs[0..1024]
  input_closure_hash
  seed
  target_profile_ref/hash
  toolchain_manifest_hash
  rng_stream_manifest_hash
  selected_binding_closure_refs[0..64]
  determinism_gate_validation:
    GeneratedWorldDeterminismGateValidationV1
  local_id_count: uint32
  create_records[local_id_count]:
    consecutive local_id plus owner-typed payload
  local_edges[]: endpoints are defined local_id values only
  update_records[]: exact existing StableId plus expected revision
  delete_records[]: exact existing StableId plus expected revision
  generated_anchor_portal_records[]: local_id references only
  validation_outputs[]: owner-typed bounded values
  output_bounds
  generation_step_count
  semantic_graph_hash: SHA-256
  candidate_artifact_semantic_hash: SHA-256
  candidate_hash: SHA-256

GeneratedWorldSemanticCandidateRefV1
  candidate_project_ref:
    {project_id, expected_project_revision, document_set_hash}
  candidate_content_ref: exact private immutable content ref
  candidate_hash: SHA-256

GeneratedWorldDeterminismGateValidationV1
  gate_validation_version: 1
  gate_execution_id: caller-issued UUIDv7
  gate_owner_ref:
    exact {owner_id=owner.core.world,owner_revision,owner_content_hash}
  gate_implementation_artifact_ref:
    {artifact_kind=world_fresh_process_determinism_validator,
     schema_version=1,sha256}
  canonical_input_closure_hash:
    exact candidate input_closure_hash
  run_count: 3
  runs[3]:
    run_ordinal: 1 | 2 | 3
    sandbox_boot_nonce: bytes32
    fresh_process_instance_hash: SHA-256
    gateway_or_broker_call_count: 0
    normalized_semantic_graph_bytes_hash: SHA-256
    canonical_record_order_hash: SHA-256
    semantic_graph_hash: SHA-256
    candidate_artifact_semantic_hash: SHA-256
  result: pass
  gate_validation_hash: SHA-256
```

`local_id_count`は`create_records[]`のlengthと一致し、local IDは1から連続する。0、gap、duplicate、未定義local edge、Stable IDに見えるgenerator生成文字列を拒否する。`semantic_graph_hash`はASCII `MIRAKAN_WORLD_GENERATED_SEMANTIC_GRAPH_V1`、exact candidate Project triple、procedural World ref、generator／input／seed／Target／Toolchain／RNG／binding closure、local-ID正規化create graph、update／delete、Anchor／Portal／validation output、bounds、step countを各`uint32_be` length framingしてSHA-256する。`candidate_artifact_semantic_hash`はStable ID allocationやartifact container metadataを除くCook入力のlocal-ID正規化bytesをASCII `MIRAKAN_WORLD_GENERATED_ARTIFACT_SEMANTIC_V1`でhashする。

三fresh processはcandidateになる前の同じcanonical generator inputから、上記candidate bodyのうち`determinism_gate_validation`と`candidate_hash`を除いたrun outputを生成する。各runの`semantic_graph_hash`は生成内容と`validation_outputs[]`を含むが、後段のgate validation、process identity、candidate container metadataを含めない。gate runnerは一回のcaller-issued `gate_execution_id`を固定し、各fresh sandboxが起動時に一度だけ生成した`bytes32` nonceから`fresh_process_instance_hash=SHA-256(ASCII "MIRAKAN_WORLD_FRESH_PROCESS_INSTANCE_V1" || gate_execution_id || run_ordinal || sandbox_boot_nonce || gate_implementation_artifact_ref.sha256 || generator implementation hash)`を計算し、三値とnonceのdistinct set equality、ordinal exact `[1,2,3]`、call count 0を検証する。`gate_validation_hash`はASCII `MIRAKAN_WORLD_DETERMINISM_GATE_VALIDATION_V1`と自己Fieldだけを除く完成validation canonical bytesをcount／length frameしてSHA-256する。gate outputはReceiptではなくcandidate-local typed validation evidenceであり、署名済みQualification Receipt、candidate ref、Stable ID allocation refを含めない。

三runのnormalized semantic graph bytes、record order、`semantic_graph_hash`、`candidate_artifact_semantic_hash`がそれぞれbyte equalityである場合だけ`result=pass`のgate validationを一件作り、一致したrun outputとvalidationを合わせてcandidateをmaterializeする。`candidate_hash`はASCII `MIRAKAN_WORLD_GENERATED_SEMANTIC_CANDIDATE_V1`と自身を除く全FieldのMCD canonical bytesを`uint32_be` length framingしてSHA-256する。生成順は`three receipt-free run outputs → gate validation → immutable candidate → candidate ref → Stable-ID allocation request`である。Project ID、expected revision、document set hash、gate implementation、input closure、三run evidenceの一Fieldでも変われば別candidateである。Refは完成後に外部materializeし、candidate recordへ自己参照で埋め戻さない。random device、wall clock、worker completion順、network response、UUIDをgenerator入力、`semantic_graph_hash`、`candidate_artifact_semantic_hash`へ含めない。fresh sandbox identityはgate evidence／candidate hashだけに含め、生成semantic bytesへ混ぜない。

determinism gateは一つでもhash不一致、process identity重複、ordinal missing／extra、Gateway／Broker call count非0、owner／implementation／input closure不一致なら`MIRAKAN-WORLD-PROCEDURAL_NONDETERMINISTIC`として三run outputを破棄し、candidateを作らない。gate validation missing／extra、未定義旧`determinism_gate_receipt_ref`、run一件差し替え、別inputでpassしたvalidation、gate hash不一致をallocation前にrejectする。retry seedはfailure policyが許可した新しい明示seedでだけ新run集合を作り、同じseedの不一致を成功へ近似しない。

三run一致後、callerはUUIDv7 `allocation_request_id`を発行し、次のnamed inputから共通`OperationIntentPayloadV2`／requestを作ってAuthoring Command Gatewayをexact一回だけ呼ぶ。Gatewayがrequest IDを生成またはcandidate hashから導出してはならない。

```text
WorldStableIdAllocationInputV1
  input_type_ref:
    exact {id=type.world.stable_id_allocation_input,
           version=1, contract_set_hash}
  operation_ref:
    exact {id=operation.world.allocate_generated_stable_ids,
           version=1, contract_set_hash}
  contract_set_hash:
    exact input_type_ref.contract_set_hash = operation_ref.contract_set_hash
  allocation_request_id: caller-issued UUIDv7
  idempotency_key: exact allocation_request_id
  candidate_ref: GeneratedWorldSemanticCandidateRefV1
  candidate_project_ref:
    {project_id, expected_project_revision, document_set_hash}
  candidate_hash: exact candidate_ref.candidate_hash
  local_id_count: uint32
  allocated_uuid_count: uint32
  precondition_policy_refs[1]:
    exact {id=policy.operation.world.stable_id_allocation.precondition,
           version=1, contract_set_hash}
  postcondition_policy_refs[1]:
    exact {id=policy.operation.world.stable_id_allocation.postcondition,
           version=1, contract_set_hash}
  rate_limit_policy_ref:
    exact {id=policy.world.stable_id_allocation.rate_limit,
           version=1, contract_set_hash}
  operation_intent_hash:
    exact hash of the canonical OperationIntentPayloadV2 projection below
  authorization_binding: exact MutationAuthorizationBindingV2(risk_class=R3)
  request_hash: SHA-256

WorldStableIdAllocationManifestV1
  allocation_request_id: same caller-issued UUIDv7
  operation_intent_hash
  request_hash
  idempotency_key: exact allocation_request_id
  candidate_project_ref:
    {project_id, expected_project_revision, document_set_hash}
  candidate_hash
  procedural_world_ref
  semantic_graph_hash
  candidate_artifact_semantic_hash
  local_id_count
  allocated_uuid_count
  mappings[local_id_count]:
    local_id: uint32
    stable_id: UUIDv7
  delta_id: UUIDv7
  manifest_hash: SHA-256

PreparedWorldStableIdAllocationReceiptPayloadV1
  publication_binding: exact PreparedReceiptPublicationBindingV1
  allocation_operation_ref
  operation_intent_hash
  request_hash
  allocation_request_id
  idempotency_key: exact allocation_request_id
  candidate_project_ref:
    {project_id, expected_project_revision, document_set_hash}
  candidate_hash
  public_after_project_ref:
    {project_id, project_revision, document_set_hash}
  allocation_manifest_ref/hash
  semantic_graph_hash
  candidate_artifact_semantic_hash
  local_id_count
  allocated_uuid_count
  gateway_service_ref/hash
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1
  prepared_receipt_payload_hash: SHA-256

WorldStableIdAllocationReceiptV1
  published_receipt:
    exact PublishedDomainReceiptV2 whose prepared_domain_receipt_payload_ref/hash
    resolves PreparedWorldStableIdAllocationReceiptPayloadV1

WorldStableIdAllocationPublicationProjectionV1
  allocation_request_id
  operation_intent_hash
  request_hash
  candidate_project_ref:
    {project_id, expected_project_revision, document_set_hash}
  candidate_hash
  allocation_manifest_ref/hash
  signed_receipt_ref/hash
  public_publication_marker_ref/hash
  generated_delta_ref/hash
  local_id_count
  allocated_uuid_count
  public_after_project_ref:
    {project_id, project_revision, document_set_hash}
  publication_projection_hash: SHA-256

WorldStableIdAllocationResultV1
  disposition: committed | rejected
  committed:
    candidate_project_ref
    candidate_hash
    public_after_project_ref
    allocation_manifest_ref/hash
    signed_receipt_ref/hash
    public_publication_marker_ref/hash: exact PublicPublicationMarkerV1
    local_id_count
    allocated_uuid_count
  rejected:
    diagnostics[1..64]: exact DiagnosticCodeRefV1 from Operation errors[]
```

`operation_intent_hash`と`request_hash`は[Executable contracts](../02-foundation/executable-contracts.md)の唯一の`MIRAKAN_OPERATION_INTENT_V2 -> MutationAuthorizationBindingV2 -> MIRAKAN_OPERATION_REQUEST_V2` DAGを使う。このDomainはsibling intent型を定義しない。canonical `OperationIntentPayloadV2`は`input_type_ref=WorldStableIdAllocationInputV1.input_type_ref`、`operation_ref=WorldStableIdAllocationInputV1.operation_ref`、`risk_class=R3`、`semantic_input_fields`をInputの`contract_set_hash`、`allocation_request_id`、`idempotency_key`、`candidate_ref`、`candidate_project_ref`、`candidate_hash`、`local_id_count`、`allocated_uuid_count`、`precondition_policy_refs[]`、`postcondition_policy_refs[]`、`rate_limit_policy_ref`のfield ID／presence discriminator込みexact projectionとする。Inputに実在する意味Fieldを追加／削除した場合はこのprojectionも同じschema生成で変わり、手書きallowlistで省略しない。`operation_intent_hash`、`request_hash`、`authorization_binding`だけを除外する。Input、Manifest、prepared Receipt、root外Publication Projectionは同じintent hashをbyte equalityで持つ。Approvalはこのintent hashへbindし、R3なのでPredelegationを拒否し、final request hashを先に要求しない。

`mappings[]`はlocal ID昇順でexact `1..local_id_count`、Stable ID重複なし、`allocated_uuid_count=local_id_count+1`（mapping分＋`delta_id`）とする。Manifest hash、prepared Receipt payload hash、Publication Projection hashはそれぞれASCII `MIRAKAN_WORLD_STABLE_ID_ALLOCATION_MANIFEST_V1`、`MIRAKAN_WORLD_STABLE_ID_ALLOCATION_RECEIPT_PAYLOAD_V1`、`MIRAKAN_WORLD_STABLE_ID_ALLOCATION_PUBLICATION_PROJECTION_V1`と各self hashを除くcanonical bytesを`uint32_be` length framingして計算する。同じidempotency key＋request hashはbyte-identical Manifest／signed Receipt／Public Marker／Resultを返し、別requestでのkey再利用を拒否する。Prepared payloadの`publication_binding.before_state_ref/hash`は`candidate_project_ref`、`staged_after_state_ref/hash`は`public_after_project_ref`とbyte equalityにする。

このOperationが満たす一つの検証可能な規範を次の完全なMCD Requirement recordへ固定する。

| Requirement MCD共通Envelope exact value | Requirement payload exact value |
|---|---|
| `mcd_version=1; kind=requirement; id=requirement.world.generated_stable_id_allocation_atomic; version=1; status=active; title=Atomic Generated World Stable-ID Allocation; description=Require one generated World candidate to publish its allocation mapping, signed receipt, marker, and Project revision as one exactly-once closure; owners=[owner.core.world]; requirement_refs=[]; rationale_refs=[mirakan.arch.rendering-world#9-procedural-world-source]; since_contract_set=1; supersedes=[]; tags=[allocation,atomic,stable_id,world]` | `normative_level=must; priority=blocking; statement=For one immutable candidate, Project triple, operation intent, request, and idempotency key, publish either zero public objects on failure or exactly one mutually bound mapping, generated Delta, signed Receipt, Public Marker, and after Project closure; scope=[artifact.world.generated_delta,phase.authoring.publication]; verification_methods=[gate.world.stable_id_allocation.atomic_publication,gate.world.stable_id_allocation.crash_recovery]; acceptance_criteria=[predicate.world.stable_id_allocation.prepublication_failure_public_count_zero,predicate.world.stable_id_allocation.public_closure_exactly_once]; failure_code={diagnostic.world.stable_id_publication_conflict,MIRAKAN-WORLD-STABLE_ID-PUBLICATION_CONFLICT,1,diagnostic_content_hash}; source_refs=[{authority=project_normative,ref=mirakan.arch.rendering-world#9-procedural-world-source}]; introduced_by=changeset.task2.final_contract_closure` |

RequirementはContract set内部で`ContractSetLocalRefV1(kind=requirement,id=requirement.world.generated_stable_id_allocation_atomic,version=1)`として参照し、root確定後だけ`McdContractRefV1`へmaterializeする。共通EnvelopeまたはRequirement payloadの実在Fieldを一つだけ変えるfixtureはRequirement member hashとset rootを変更し、旧Operation／Policy／Diagnostic external refを解決不能にする。

このinternal Operationを次の一件の完全recordとしてcurrent Contract setへ登録する。

```text
McdOperationContractV1
  MCD common envelope:
    mcd_version=1
    kind=operation
    id=operation.world.allocate_generated_stable_ids
    version=1
    status=active
    title=Allocate Generated World Stable IDs
    description=Allocate one canonical Stable-ID mapping and atomically publish
      the generated World delta into the bound Project
    owners=[owner.core.world]
    requirement_refs=[
      {id=requirement.world.generated_stable_id_allocation_atomic,
       version=1,contract_set_hash}]
    rationale_refs=[mirakan.arch.rendering-world#9-procedural-world-source]
    since_contract_set=1
    supersedes=[]
    tags=[internal_gateway,procedural,stable_id,world]
  operation_kind: command
  input_type:
    {id=type.world.stable_id_allocation_input,version=1,contract_set_hash}
  output_type:
    {id=type.world.stable_id_allocation_result,version=1,contract_set_hash}
  authority:
    {service_id=service.authoring_command_gateway,
     service_version=1,service_content_hash}
  risk_class: R3
  side_effects: [authoring, source]
  idempotency: idempotent_with_key
  transaction: authoring_changeset
  preconditions:
    [{id=policy.operation.world.stable_id_allocation.precondition,
      version=1,contract_set_hash}]
  postconditions:
    [{id=policy.operation.world.stable_id_allocation.postcondition,
      version=1,contract_set_hash}]
  errors:
    exact WorldStableIdAllocationDiagnosticSetV1
  validator_closure_ref:
    {closure_id=validator_closure.operation.world.stable_id_allocation,
     closure_version=1,closure_content_hash}
  timeout_ms: 120000
  rate_limit_policy:
    {id=policy.world.stable_id_allocation.rate_limit,
     version=1,contract_set_hash}
  audit_level: full_redacted
  provider_exposure: trusted_internal
  receipt_type:
    {id=type.world.stable_id_allocation_receipt,
     version=1,contract_set_hash}
```

`type.world.stable_id_allocation_input`、`type.world.stable_id_allocation_result`、`type.world.stable_id_allocation_receipt`、`type.world.prepared_stable_id_allocation_receipt_payload`はversion 1の別々のexact Type LocalRefで、対応する上記named schemaの全Fieldを投影する。prepared payload Typeと最終Receipt Typeを相互代用せず、anonymous sibling、inline signature、unsigned receipt aliasを生成しない。

Operationが参照する三Policyは次の完全なactive MCD recordである。表の二列を連結した値が各record全体であり、暗黙既定値、bare ID、説明文からFieldを補完しない。

| Policy MCD共通Envelope exact value | Policy payload exact value |
|---|---|
| `mcd_version=1; kind=policy; id=policy.operation.world.stable_id_allocation.precondition; version=1; status=active; title=World Stable-ID Allocation Precondition; description=Validate the exact immutable candidate, Project base, request identity, allocation counts, determinism evidence, authorization, and idempotency snapshot without mutation; owners=[owner.core.world]; requirement_refs=[{id=requirement.world.generated_stable_id_allocation_atomic,version=1,contract_set_hash}]; rationale_refs=[mirakan.arch.rendering-world#9-procedural-world-source]; since_contract_set=1; supersedes=[]; tags=[operation_predicate,pure,stable_id,world]` | `evaluation_mode=pure; side_effects=[]; input_type={id=type.operation.precondition_evaluation_input,version=1,contract_set_hash}; result_type={id=type.operation.predicate_result,version=1,contract_set_hash}` |
| `mcd_version=1; kind=policy; id=policy.operation.world.stable_id_allocation.postcondition; version=1; status=active; title=World Stable-ID Allocation Postcondition; description=Validate the unpublished mapping, Manifest, Delta, prepared Receipt payload, Project document-set hash, counts, and atomic publication candidate; owners=[owner.core.world]; requirement_refs=[{id=requirement.world.generated_stable_id_allocation_atomic,version=1,contract_set_hash}]; rationale_refs=[mirakan.arch.rendering-world#9-procedural-world-source]; since_contract_set=1; supersedes=[]; tags=[operation_predicate,pure,stable_id,world]` | `evaluation_mode=pure; side_effects=[]; input_type={id=type.operation.postcondition_evaluation_input,version=2,contract_set_hash}; result_type={id=type.operation.predicate_result,version=1,contract_set_hash}` |
| `mcd_version=1; kind=policy; id=policy.world.stable_id_allocation.rate_limit; version=1; status=active; title=World Stable-ID Allocation Rate Limit; description=Bound stable-ID allocation requests per Project without changing allocation semantics; owners=[owner.core.world]; requirement_refs=[{id=requirement.world.generated_stable_id_allocation_atomic,version=1,contract_set_hash}]; rationale_refs=[mirakan.arch.rendering-world#9-procedural-world-source]; since_contract_set=1; supersedes=[]; tags=[authoring,rate_limit,stable_id,world]` | `policy_ref={id=policy.world.stable_id_allocation.rate_limit,version=1,contract_set_hash}; scope=project; window_ns=60000000000; max_requests=4; burst=1; exceeded_error_ref={diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash}` |

preconditionはexact Input、immutable candidate、before Project snapshot、authorization binding、idempotency snapshotだけを受ける。postconditionはrequest hash、Prepared Candidate、before snapshot、未公開staged after Project、Manifest、Delta、prepared Receipt payloadだけを受ける。両Policyは評価中にRegistry、clock、network、UUID generator、private／public Marker、final Receiptをqueryしない。Contract set内部では三Policyの相互refをLocalRefへ投影し、Manifest／Operation／World owner Policy subsetをexact三件でset equalityにする。三recordの実在Fieldを一つだけ変えるfixtureはPolicy member hashとset rootを変更する。

`WorldStableIdAllocationDiagnosticSetV1`は次のexact `DiagnosticCodeRefV1 {diagnostic_id, code, diagnostic_version=1, diagnostic_content_hash}`集合である。

| Diagnostic ID | code |
|---|---|
| `diagnostic.conflict.revision_mismatch` | `MIRAKAN-CONFLICT-REVISION_MISMATCH` |
| `diagnostic.authorization.denied` | `MIRAKAN-AUTHORIZATION-DENIED` |
| `diagnostic.approval.required` | `MIRAKAN-APPROVAL-REQUIRED` |
| `diagnostic.authoring.lock_conflict` | `MIRAKAN-AUTHORING-LOCK_CONFLICT` |
| `diagnostic.mcd.operation_predicate_invalid` | `MIRAKAN-MCD-OPERATION-PREDICATE_INVALID` |
| `diagnostic.operation.timeout` | `MIRAKAN-OPERATION-TIMEOUT` |
| `diagnostic.operation.rate_limit_exceeded` | `MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED` |
| `diagnostic.operation.idempotency_key_reuse` | `MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE` |
| `diagnostic.world.generated_candidate_invalid` | `MIRAKAN-WORLD-GENERATED_CANDIDATE_INVALID` |
| `diagnostic.world.stable_id_project_binding_mismatch` | `MIRAKAN-WORLD-STABLE_ID-PROJECT_BINDING_MISMATCH` |
| `diagnostic.world.stable_id_allocation_count_mismatch` | `MIRAKAN-WORLD-STABLE_ID-ALLOCATION_COUNT_MISMATCH` |
| `diagnostic.world.stable_id_manifest_invalid` | `MIRAKAN-WORLD-STABLE_ID-MANIFEST_INVALID` |
| `diagnostic.world.stable_id_receipt_signing_failed` | `MIRAKAN-WORLD-STABLE_ID-RECEIPT_SIGNING_FAILED` |
| `diagnostic.world.stable_id_publication_conflict` | `MIRAKAN-WORLD-STABLE_ID-PUBLICATION_CONFLICT` |

World固有六件は次の完全な`DiagnosticLocalRecordV2`である。全rowのexact Owner identity `DiagnosticOwnerLocalRefV1`は`{owner_id=owner.core.world,owner_revision=1,owner_content_hash=SHA-256(MIRAKAN_DIAGNOSTIC_OWNER_LOCAL_IDENTITY_V1, length-framed canonical owner_id／owner_revision)}`であり、表の短記をこの三Fieldへ必ずmaterializeする。World固有alias型を定義しない。World ManifestのDiagnostic local subsetとDiagnostic Registryの同Owner subsetもこの六件でset equalityにする。local recordの`owner_local_ref`と`requirement_local_refs[]`はroot前後ともLocal identity／LocalRefのまま保持し、root確定後に作る外部`DiagnosticCodeRecordV1` projectionだけがexact external Owner refと対応Requirementのexact `McdContractRefV1`をmaterializeする。

| Diagnostic LocalRef | exact `owner_local_ref` | code | severity | category | message_key | exact `requirement_local_refs[]` | retryability | diagnostic_local_content_hash |
|---|---|---|---|---|---|---|---|---|
| `{diagnostic.world.generated_candidate_invalid,1}` | `DiagnosticOwnerLocalRefV1` | `MIRAKAN-WORLD-GENERATED_CANDIDATE_INVALID` | `blocking` | `schema` | `diagnostic.world.generated_candidate_invalid.message` | `[{kind=requirement,id=requirement.world.generated_stable_id_allocation_atomic,version=1}]` | `after_input` | `SHA-256(MIRAKAN_DIAGNOSTIC_LOCAL_RECORD_V2, self-excluding local fields)` |
| `{diagnostic.world.stable_id_project_binding_mismatch,1}` | `DiagnosticOwnerLocalRefV1` | `MIRAKAN-WORLD-STABLE_ID-PROJECT_BINDING_MISMATCH` | `blocking` | `conflict` | `diagnostic.world.stable_id_project_binding_mismatch.message` | `[{kind=requirement,id=requirement.world.generated_stable_id_allocation_atomic,version=1}]` | `after_change` | `SHA-256(MIRAKAN_DIAGNOSTIC_LOCAL_RECORD_V2, self-excluding local fields)` |
| `{diagnostic.world.stable_id_allocation_count_mismatch,1}` | `DiagnosticOwnerLocalRefV1` | `MIRAKAN-WORLD-STABLE_ID-ALLOCATION_COUNT_MISMATCH` | `blocking` | `semantic` | `diagnostic.world.stable_id_allocation_count_mismatch.message` | `[{kind=requirement,id=requirement.world.generated_stable_id_allocation_atomic,version=1}]` | `after_input` | `SHA-256(MIRAKAN_DIAGNOSTIC_LOCAL_RECORD_V2, self-excluding local fields)` |
| `{diagnostic.world.stable_id_manifest_invalid,1}` | `DiagnosticOwnerLocalRefV1` | `MIRAKAN-WORLD-STABLE_ID-MANIFEST_INVALID` | `blocking` | `schema` | `diagnostic.world.stable_id_manifest_invalid.message` | `[{kind=requirement,id=requirement.world.generated_stable_id_allocation_atomic,version=1}]` | `after_change` | `SHA-256(MIRAKAN_DIAGNOSTIC_LOCAL_RECORD_V2, self-excluding local fields)` |
| `{diagnostic.world.stable_id_receipt_signing_failed,1}` | `DiagnosticOwnerLocalRefV1` | `MIRAKAN-WORLD-STABLE_ID-RECEIPT_SIGNING_FAILED` | `error` | `infrastructure` | `diagnostic.world.stable_id_receipt_signing_failed.message` | `[{kind=requirement,id=requirement.world.generated_stable_id_allocation_atomic,version=1}]` | `transient` | `SHA-256(MIRAKAN_DIAGNOSTIC_LOCAL_RECORD_V2, self-excluding local fields)` |
| `{diagnostic.world.stable_id_publication_conflict,1}` | `DiagnosticOwnerLocalRefV1` | `MIRAKAN-WORLD-STABLE_ID-PUBLICATION_CONFLICT` | `blocking` | `conflict` | `diagnostic.world.stable_id_publication_conflict.message` | `[{kind=requirement,id=requirement.world.generated_stable_id_allocation_atomic,version=1}]` | `after_change` | `SHA-256(MIRAKAN_DIAGNOSTIC_LOCAL_RECORD_V2, self-excluding local fields)` |

生成順は`Owner local identity → diagnostic_local_content_hash → Diagnostic member_hash → Contract set root → 外部DiagnosticCodeRecordV1.diagnostic_content_hash／DiagnosticCodeRefV1`である。外部hashはroot付きRequirement refを含むためlocal hash／member hashと別値であり、等値を要求せず、外部hashをlocal payloadへ戻さない。六recordそれぞれについて`owner_local_ref`を含むcode以外の実在Fieldを一つだけ変えるfixtureを作り、local hash、member hash、set root、外部content hash／四Fieldrefがすべて再生成されることを検証する。

`OperationValidatorClosureV1`はcommon request、authorization、approval、revision／lock、pure predicate、timeout／rate-limitの六Validatorと、`validator.world.generated_candidate_semantics`、`validator.world.stable_id_allocation_postcondition`、`validator.world.stable_id_allocation_publication`のexact九件を持つ。各Validator recordはversion／implementation Artifact／input Type／content hashと上表の明示subsetを持ち、九recordのerror union = closure `reachable_error_refs[]` = Operation `errors[]`を四Fieldset equalityで検査する。candidate validatorはcandidate／Project／count三error、postcondition validatorはcount／Manifest error、publication validatorはReceipt signing／publication errorだけを所有し、common八errorは対応common Validatorへ一件ずつ割り当てる。missing／extra／duplicate／stale ref、説明文からのerror合成をContract set compile failureにする。

World ownerの完全Manifestを次へ固定する。

```text
WorldStableIdAllocationOperationManifestV1
  manifest_id: world.stable_id_allocation.operation_manifest
  manifest_version: 1
  operation_local_refs[1]:
    {kind=operation,id=operation.world.allocate_generated_stable_ids,version=1}
  type_local_refs[3]:
    {kind=type,id=type.world.stable_id_allocation_input,version=1}
    {kind=type,id=type.world.stable_id_allocation_result,version=1}
    {kind=type,id=type.world.stable_id_allocation_receipt,version=1}
  requirement_local_refs[1]:
    {kind=requirement,
     id=requirement.world.generated_stable_id_allocation_atomic,version=1}
  policy_local_refs[3]:
    {kind=policy,
     id=policy.operation.world.stable_id_allocation.precondition,version=1}
    {kind=policy,
     id=policy.operation.world.stable_id_allocation.postcondition,version=1}
    {kind=policy,id=policy.world.stable_id_allocation.rate_limit,version=1}
  validator_closure_local_refs[1]:
    {validator_closure.operation.world.stable_id_allocation,1}
  diagnostic_local_refs[14]:
    exact common eight plus World-specific six listed above
  trusted_service_local_ref: {service.authoring_command_gateway,1}
  trusted_service_allowlist_operation_local_refs[1]:
    {kind=operation,id=operation.world.allocate_generated_stable_ids,version=1}
  provider_projection_local_refs: []
  mcp_tool_local_refs: []
  manifest_hash: SHA-256
```

`operation_local_refs[] = World owner active MCD Operation subset = Authoring Command GatewayのWorld owner allowlist contribution`はexact一件、`policy_local_refs[] = Operation pre／post／rate refs`はexact三件、`diagnostic_local_refs[] = Validator error union = closure reachable errors = Operation errors[]`はexact 14件である。World固有Diagnostic六件はDiagnostic RegistryのWorld owner subset、Requirement一件はWorld owner Requirement subsetとそれぞれset equalityにする。`provider_exposure=trusted_internal`なのでProvider／MCP集合はexact空である。Manifest hashはASCII `MIRAKAN_WORLD_STABLE_ID_ALLOCATION_OPERATION_MANIFEST_V1`と自己hashを除く全Fieldのcanonical bytesから計算し、missing／extra／duplicate／order／one-field mutationでcompileをfail closedにする。

postcondition成功後、Prepared Envelope、candidate／validation／prepared Receipt payload、staged Manifest／Delta／Receipt-free after Project、`PrivateDurableCommitMarkerV1`をprivate durable transactionへcommitする。Marker readback後にprepared World payloadを参照するcanonical `PublishedDomainReceiptV2`／`MirakanSignedRecordV1` wrapperをput-if-absentし、署名済みwrapper readback後だけ`PublicPublicationMarkerV1`、Receipt-free after Project revision、Document index、root外`WorldStableIdAllocationPublicationProjectionV1`を一つのpublic CASでatomic publishする。Public Markerの`public_after_state_ref/hash`、Published payloadの`after_state_ref/hash`、private Markerの`staged_after_state_ref/hash`はすべてprepared payloadの`public_after_project_ref`へexact解決する。Publication Projectionだけがsigned Receipt ref／hashとPublic Marker ref／hashを持ち、そのどちらからも参照されない。ProjectionのManifest／Project triple／candidate hash／request identity／mapping countはprepared payload、signed Receipt、Public Marker、public Projectの同値とbyte equalityにする。

private Marker前のfailureはpublic objectを0件、private Marker後かつ署名前のcrashは固定materialization key／issued-at／revocation snapshot／deterministic signing profileで同じwrapper bytesをroll-forward、署名後かつPublic Marker前のcrashは同じexpected predecessorへpublic CASをroll-forwardする。Public Marker後はrollback、alternate signature、二重publicationを許可しない。mappingまたは`delta_id`を部分公開せず、Project head、Document index、generated Delta、Manifest、Receipt、Public Marker、Publication Projectionは同じtransactionでall-or-nothingに可視化する。

`GeneratedWorldDeltaV1`はGateway発行`delta_id`、exact allocation Manifest ref／hash、候補の全入力、mapping適用済みcreate／update／delete、generated Anchor／Portal、typed validation outputs、output bounds、step count、`semantic_graph_hash`、`candidate_artifact_semantic_hash`、`delta_content_hash`を持つReceipt-free staged recordである。signed Receipt ref／hash、private／public Marker、Publication ProjectionをFieldにもhash preimageにも含めない。`delta_content_hash`はASCII `MIRAKAN_WORLD_GENERATED_DELTA_V1`とself hashを除く全Fieldをhashする。private MarkerはこのReceipt-free Delta ref／hashをstaged after-stateの一部として固定し、その後に作るsigned ReceiptとDeltaの結合は`PublicPublicationMarkerV1`およびroot外`WorldStableIdAllocationPublicationProjectionV1`だけが所有する。Source Delta、全内部ref、validation output、Preview、Cook入力、Commit Receiptは一つのManifest mappingだけを共有し、subsystem別再割当、二度目のGateway call、local ID残存を拒否する。UUIDを除外した再現性の判定にはDelta hashを使わない。既存更新／削除は明示allowlistとexpected revisionを必須とし、World全置換、上限超過、absolute path、native pointer、Cooked Artifact本文を拒否する。

生成結果は通常のScene／Entity／Cell validationとreviewを通り、手編集領域を無断上書きしない。Schema不一致、unknown ref、bounds／step／entity上限超過、topology cycle、空間connectivity不成立は常時検証する。条件provider検証のtruth tableを次へ固定する。

| selected binding | 対応output | 結果 |
|---|---|---|
| absent | empty | canonical skip。provider、placeholder、failureを生成しない |
| absent | non-empty | output owner不明としてDelta全体reject |
| selected | `count < required_output_count` | required output欠落としてDelta全体reject |
| selected、Document ref/content hash／resolved closure hash／schema valid | required count以上で全output valid | providerを一回実行しvalidation成功 |
| selectedだがDocument／owner／closure／schema stale、またはprovider result invalid／failure | 任意 | `MIRAKAN-WORLD-PROCEDURAL_INVALID_OUTPUT`でDelta全体reject |

一bindingでもrejectならcreate／update／deleteを一件もpublishせず、Project revision、last-valid Source、Derived Artifact、World generationを維持する。同じgenerator version、input revisions、seed、Target、Toolchain、binding setで再生成した場合、provider選択を含むlocal semantic graphとcandidate Artifact semantic hashが一致しなければならない。binding Document content hash、resolved closure hash、owner hash、provider Contract set hash、output schema hash、validation output hash、semantic graph hash、allocation mapping hashを各一Fieldだけtamperするnegative fixtureを持ち、別hashの一致で補償しない。再生成中のArtifact削除、provider timeout、hash mismatch、unsupported Target、fallback layoutもtyped resultとし、失敗時はlast-validを維持する。外部Tool／generator versionとartifact hashは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)を参照する。

## 10. Navigation、Simulation、Renderingとの境界

WorldはNavigation build volume／modifier、Physics／Collision component、Animation component、Render／Material／Light componentへのtyped document refを保持できるが、各Domain fieldや挙動を定義しない。Domain artifactのcook／validation resultをCell dependency closureへ取り込む。

Navigation queryやWorld movement、Physics body activation、Animation sampling、Render visibilityは各OwnerとRuntime schedulingが所有する。World Cell stateをそれらのauthoritative simulation stateとして兼用しない。

[LOD](lod.md)はresident candidateからrepresentationを選び、[Render Graph](render-graph.md)はactive Cell由来の`WorldRenderPacket`を実行する。Worldはselection formula、visibility algorithm、render passを所有しない。

`MapPresentationDefinitionV1`は`map_presentation_id: StableId`、`presentation_kind: minimap | world_map | space_map | navigation_overlay`、exact `spatial_subject_ref`、`projection_policy: orthographic_2d | authored_2d | projected_3d`、`layer_refs[1..128]`、`marker_style_refs[0..512]`、typed `marker_source_contract_refs`、optional `fog_policy_ref`、`accessibility_profile_refs[1..32]`、exact `localization_namespace_ref`、Targetごとのexact `render_budget_refs`、`fallback_contract`を持つ非authoritative Sourceである。Marker／fog／cursorからWorld state、consumer-owned Gameplay state、Navigation costを直接writeせず、入力はtyped `MapInteractionCommandV1`として登録済みcommand ownerへ送る。

Map／World presentation、LOD、visibility、Camera、GPU resultからWorld、Space、Cell、persistent entity、consumer-owned stateへ直接writeする実装は`MIRAKAN-WORLD-PRESENTATION_AUTHORITY_WRITE`としてBuild／conformanceを失敗させる。presentation projectionの欠落、degradation、frame dropはauthoritative activation、transfer、Source revisionを変更しない。

### 10.1 Tilemap source、cook、publication

Tile chunkはTilemap内部のedit、cook、culling単位であり、World Cell、Region、Space、activation groupではない。Tilemapはresident／active CellのAsset closureへ参加するが、chunk単独でEntity、consumer-owned state、transition、authoritative gameplayをactivateしない。次の12型をWorldの唯一の正本とする。

```text
TileGridV1
  tile_texel_extent: uint2
  pixels_per_unit: positive finite float
  origin: top_left | bottom_left | center
  axis: x_right_y_down | x_right_y_up
  stagger_axis: none | x | y
  stagger_index: none | even | odd
  hex_side_length_texels: uint32

TileSetAssetV1
  tile_set_id: StableId
  revision: AssetRevision
  grid: TileGridV1
  sprite_table_ref: exact revision ref
  tiles: bounded array[1..65535]<TileDefinitionV1>
  terrain_rule_sets: bounded array[0..4096]<TerrainRuleSetV1>
  custom_property_schema_ref: optional exact schema ref

TileDefinitionV1
  tile_id: StableId
  sprite_id: StableId
  animation_frames: bounded array[0..256]<{sprite_id, duration_ms}>
  material_slot_ref: exact material slot ref
  pivot_offset_m: finite Displacement2f
  collision_shape_ref: optional exact source ref
  collision_tag_ref: optional typed tag ref
  navigation_area_ref: optional typed area ref
  navigation_blocked: bool
  terrain_edge_tags: bounded array[0..8]<typed tag ref>
  terrain_corner_tags: bounded array[0..8]<typed tag ref>
  custom_properties: bounded array[0..32]<typed property>

TerrainRuleSetV1
  rule_set_id: StableId
  terrain_tag_ref: typed tag ref
  adjacency: edge | edge_and_corner
  rules: bounded array[1..65535]<{neighbor_tag_pattern, candidate_tile_ids[1..65535], positive_weights[1..65535]}>
  dependency_radius_tiles: uint8
  deterministic_tie_break: canonical_tile_id_then_rule_id

TileSetRevisionV1
  tile_set_asset_id: AssetId<TileSet>
  asset_revision: AssetRevision
  source_content_hash: bytes32

TilemapAssetV1
  tilemap_id: StableId
  orientation: orthogonal | isometric | staggered | hexagonal
  tile_set_sources: bounded array[1..256]<TileSetRevisionV1>
  layers: bounded array[1..64]<TileLayerV1>
  chunk_extent_tiles: uint2
  source_bounds: optional RectI64
  generation: uint64

TileLayerV1
  layer_id: StableId
  kind: tile | group | typed_object_stamp | image | height_semantic
  parent_layer_ref: optional StableId
  chunks: sparse ordered map<int2, TileChunkSourceV1>
  visible: bool
  locked: bool
  opacity_q16: uint16
  tint_linear: finite Color4f
  blend_mode_ref: exact blend ref
  parallax: finite float2
  offset_m: finite Displacement2f
  collision_contribution: none | enabled
  navigation_contribution: none | area | obstacle

TileChunkSourceV1
  chunk_coord: int2
  cells: bounded array[0..65536]<TileCellSourceV1>
  generation: uint64

TileCellSourceV1
  local_coord: uint2
  tile_set_revision_ref: exact TileSetRevisionV1 ref
  tile_id: StableId
  transform: identity | rotate_90 | rotate_180 | rotate_270 | flip_x | flip_y | transpose | anti_transpose
  animation_phase_policy: synchronized | stable_cell_offset

TileChunkArtifactV1
  tilemap_ref: exact {tilemap_id, revision}
  layer_id: StableId
  chunk_coord: int2
  source_generation: uint64
  occupied_bounds: RectU32
  renderer_artifact_key: ArtifactKey
  draw_spans: bounded array[0..4096]<TileDrawSpanV1>
  collision_artifact_key: optional ArtifactKey
  navigation_artifact_key: optional ArtifactKey
  dependency_hashes: bounded array[1..1024]<bytes32>
  content_sha256: bytes32

TileDrawSpanV1
  canvas_batch_key: exact Canvas Batch Key
  instance_offset: uint32
  instance_count: positive uint32
  local_bounds: RectF32
  cooked_cell_transform_state: d4_applied_once

TileLayoutCommandV1
  command_id: StableId
  target_tilemap_ref: exact {tilemap_id, expected_revision, expected_generation}
  region: RectI64
  operation: paint | erase | fill | stamp | apply_rule | replace_by_query
  rule_set_ref: optional exact TerrainRuleSetV1 ref
  stamp_asset_ref: optional exact Asset revision ref
  seed: uint64
  allowed_tile_ids: bounded set[0..65535]<StableId>
  allowed_terrain_tag_refs: bounded set[0..4096]<typed tag ref>
  connectivity_constraints: bounded array[0..256]<typed constraint>
  reachability_constraints: bounded array[0..256]<typed constraint>
  max_examined_tile_count: positive uint32
  max_changed_tile_count: positive uint32
  preview_expansion_hash: bytes32
```

`TileGridV1`のtexel extentは両軸正、`pixels_per_unit`はfiniteかつ正とする。stagger／hex fieldはorientationと整合しなければならない。一Tilemapの全`tile_set_sources`が参照するTileSetは`TileGridV1`の全fieldが一致しなければならず、この一致したgridをTilemapのcell格子の正本とする。不一致は`MIRAKAN-WORLD-TILE_SOURCE_INVALID`としてSource／Cookを拒否し、変換で近似しない。C1 orientationは`orthogonal`だけをQualifiedとし、他のorientationと`typed_object_stamp | image | height_semantic`はC2の個別Qualification前に拒否する。`RectI64`／`RectU32`はinclusive min、exclusive maxで、empty、overflow、負のunsigned extentを拒否する。

空cellはrecord欠落で表し、`tile_id=0`等のsentinelを保存しない。chunk extentは各軸8～256の2冪、Referenceは32×32である。`local_coord`は`[0, chunk_extent_tiles)`内で一意とし、cellsを`local_y, local_x, tile_id`、chunksを`chunk_y, chunk_x`でcanonicalizeする。負のWorld tile coordinateはfloor divisionでchunkへ写像し、`local = world - chunk * extent`を必ず非負にする。言語の負剰余を使わない。

C1 boundは一TileSet 65,535 Tile、一Tilemap 64 Layer、全Layer合計16,777,216 occupied cell、一chunk 4,096 draw spanである。Tile animationのdurationは各1～60,000 ms、合計24時間以下とする。重複／範囲外cell、unknown Tile、TileSet revision mismatch、TileSet grid不一致、Source bounds／count overflow、unsupported orientation、dangling parent／Tileset、非finite offsetをtyped validation failureとして拒否する。外部global tile ID、配列index、path、表示名をStable Tile IDにしない。

cell `transform`は正方形格子の二面体群D4のclosed enumである。Cookerはcell中心を基準に、Renderer UV、Sprite pivot、Collision polygon、Navigation source、terrain edge／corner tagへ同じD4 transformをちょうど一度適用する。`TileDrawSpanV1.cooked_cell_transform_state`は適用済みを示し、consumerは再適用しない。いずれかのconsumerが同じ変換を表現できない、適用回数または結果hashが一致しない場合は`consumer_transform_mismatch`でclosure全体を失敗させ、Presentationだけを成功させない。`stable_cell_offset`はTilemap ID、Layer ID、World tile coordinate、Tile IDのcanonical hashだけから決め、load順、chunk residency順、worker順をseedにしない。

Tile editはimmutableな新Artifactを、変更region外周1 tile、terrain dependency radius、明示選択されたconsumer bindingのdependency radiusまで再Cookする。Tilemap Sourceが選択したRenderer、Collision、Navigationその他のrequired bindingだけがすべてReadyで、dependency hashとsource generationが一致した後、World activation groupとTilemap generationを一つのpublication boundaryでatomic publishする。未選択consumerをhard closureへ追加しない。Presentation-only変更で選択済みauthoritative Artifactを再利用する場合もexact dependency hashを検証する。一つでもfailed／cancelled／staleなら旧generationを維持し、partial artifact、空Tile、無衝突状態を公開しない。active authoritative regionの選択ArtifactをPresentationより先にevictせず、Cell all-or-nothing activationをchunk単位へ弱めない。

AIとEditorは同じbounded `TileLayoutCommandV1`を使い、AIが巨大なtile ID配列を直接生成することを拒否する。`region`のinclusive-min／exclusive-max areaはoverflow-safeなwide integerで事前計算し、canonical preflightはcell payloadのscanやcandidate materializationなしに決定論的なexpansion candidate countを導出する。region areaとcandidate countはそれぞれ`max_examined_tile_count`、Target Profileのcommand examined-tile limit、C1全Layer ceiling 16,777,216以下でなければならず、`max_changed_tile_count <= max_examined_tile_count`も必須とする。16,777,216 examined tileは他の二上限も許せば受理し、exact +1の16,777,217、area積overflow、いずれかの上限超過はscan／expansion開始前にtyped `tile_layout_capacity_exceeded`で拒否する。

Engineはaccepted commandだけをexpected revision／generation上で決定論的に展開し、allowed set、接続、到達性、変更数を検証する。capacity failureでregionをtruncate、sample、分割、近似せず、candidate、changed tile、Artifactを部分公開しない。Commit直前に同じ入力から再展開し、canonical preview expansion hashと一致しなければ`preview_commit_hash_mismatch`として拒否する。stale generation、unknown Tile、revision mismatchもtyped rejectionであり、全拒否経路でprevious Tilemap／World generationと既存の明示選択consumer Artifactを維持する。

### 10.2 Engine-native 3D Blockout

```text
PrimitiveMeshSourceV1
  primitive_id: StableId
  kind: box | sphere | capsule | cylinder | plane | ramp
  dimensions_m: finite float3
  radial_segments: uint8
  height_segments: uint8
  uv_policy: generated_world_scale | generated_normalized
  material_instance_ref: exact Asset revision ref
  collision_semantic: none | solid | walkable | interaction
  navigation_semantic: ignore | walkable | obstacle
  mobility: static | movable

BlockoutAssemblyV1
  assembly_id: StableId
  assembly_version: positive uint32
  owner_ref: exact {owner_id, owner_revision, owner_content_hash}
  assembly_transform: finite Transform3f with positive scale
  primitive_instances: bounded array[1..256]<{primitive_ref, local_transform}>
  composition_recipe_ref: optional exact recipe ref
  gameplay_anchor_refs: bounded array[0..256]<StableId>
  source_generation: uint64
  assembly_content_hash: SHA-256
```

`assembly_content_hash`はASCII `MIRAKAN_BLOCKOUT_ASSEMBLY_V1`と自己Fieldだけを除くReceipt-free canonical bytesから計算する。Qualificationは完成Assembly refをsubjectにする`WorldSourceQualificationSubjectV1(subject_kind=blockout)`からroot外Bindingを作り、Receipt／BindingをAssemblyへ戻さない。各dimensionは0.01～10,000 m、radial segmentsは3～64、height segmentsは1～64、compact spatial assembly全体は4,096 primitive以下とする。assembly／local transformはfinite translation、normalized rotation、正のscaleで、負scale、NaN／Inf、zero-area surface、暗黙boolean由来のnonmanifold、Colliderとwalkable semanticの矛盾を拒否する。Primitiveは通常のTransform、Material、Renderer、Collision、Navigation SourceへCookし、Blockout専用Runtime objectを作らない。

`CreatePrimitiveMesh | CreateBlockoutAssembly | UpdateBlockoutPrimitive | PromoteBlockoutToMeshAsset`はAI／Editor共通のbounded blockoutを説明するplanned semantic action vocabularyであり、Stable ID、MCD Operation、current `WorldSourceChangePrimitiveKindV1` discriminatorではない。current MCD／Owner Manifest／Service allowlist／Provider／MCP Tool集合はこれら四actionについてexact `[]`、Capability stateは`not_activated`である。future work item `activation.world.blockout_authoring.v1`が採用するexact外側MCD Operationとtyped `ProjectChangePrimitiveV1` branch、Policy／Validator／Diagnostic／Receipt／publication closureを同じContract set transactionで登録するまでdispatchしない。Activation後のPromotion previewは元Stable ID、generation、pivot、bounds、Material slot、Collider／Navigation semantic、参照元を固定し、承認後に通常Mesh Sourceと対応表をatomic publishする。Sourceが明示選択したconsumer Artifact bindingのall-readyとgeneration一致前にはassembly置換を公開せず、Renderer／Collision／Navigationを固定必須集合にしない。元Sourceを自動削除せず、置換対象はexplicit ChangeSetに列挙する。C1 fixtureはexternal DCCを要求せず、6 primitive、dimension境界、4,096 primitive spatial assembly、Collider／Navigationを選択したcook、Promotion前後のbounds／pivot／Material slot、Undo／Redo、Save／Load、Activation後のAI／手動change primitiveのafter hash一致を検証する。external DCC Asset 0件のgeneric spatial blockoutのWorld activation smokeを完走できることをGateとする。

## 11. Authoring bundleとAI／Editor UX

World authoring bundleは対象World／Scene／Space revision、selected scope、owner-typed document refs、streaming-plan preview、validation summaryを束ねる。

```text
WorldAuthoringPlanV1
  plan_id: StableId
  project_revision: exact Project revision
  contract_set_hash: bytes32
  map_intent_resolution_hash: bytes32
  requirement_ids: bounded array[1..256]<exact requirement ID>
  target_profile_ids: bounded array[1..32]<exact Target Profile ID>
  affected_world_refs: bounded array[0..64]<exact World ref>
  affected_scene_refs: bounded array[0..1024]<exact Scene ref>
  affected_space_refs: bounded array[0..1024]<exact Space ref>
  affected_topology_refs: bounded array[0..64]<exact SpatialTopologyDefinitionV1 ref>
  affected_owner_typed_document_refs: bounded array[0..1024]<exact owner-typed document ref>
  create_document_kinds: bounded array[0..64]<closed owner document kind>
  source_change_kinds: bounded array[1..6]<closed source change kind>
  required_system_refs: bounded array[0..128]<exact system ref>
  required_capability_refs: bounded array[0..128]<exact Capability ref>
  selected_validation_provider_bindings:
    bounded array[0..64]<{
      exact binding_document_ref including content_hash,
      resolved_binding_closure_hash,
      output_schema_ref: McdContractRefV1}>
  budget_refs: bounded array[1..64]<exact budget ref>
  derived_build_jobs: bounded array[0..256]<typed build job>
  assumptions: bounded array[0..32]<typed assumption>
  blocking_questions: bounded array[0..7]<MapIntentResolutionV1 Question>
  fallback: optional exact fallback ref
  risk_class: closed RiskClassV1
  disposition: ready_to_stage | question_required | capability_unavailable | target_unsupported | budget_missing | rejected
```

`source_change_kinds`は`world_document | scene_composition | topology | partition | procedural_layout | map_presentation`の6種へ閉じる。既存World編集branchは`affected_world_refs[1..64]`を必須とし、すべてのaffected refが`project_revision`に実在しexpected kindと一致しなければならない。新規World作成branchだけは`affected_world_refs=[]`を許可するが、`create_document_kinds`がclosed World document kindを厳密に一件含み、`source_change_kinds`が`world_document`を含むことを必須にする。新規IDはCommit時にGatewayが発行し、Planへ存在しないWorld IDを先行記録しない。procedural branchの`selected_validation_provider_bindings`はCommit対象`ProceduralWorldDefinitionV1`とbinding Document ref/content hash、resolved closure hash、output schemaのset equalityで一致し、Planだけでproviderを追加・削除しない。`disposition=question_required`だけがQuestionを1～7件持ち、その他は0件とする。PlanはReceipt-free read-only／proposal-onlyで、Source、Staging、Derived Artifact、Runtime stateを変更せず、Commit／Approval／Receipt権限を持たない。

```text
WorldAuthoringBundleV1
  bundle_id: StableId
  bundle_version: positive uint32
  owner_ref: exact {owner_id, owner_revision, owner_content_hash}
  project_id: StableId
  base_project_revision: exact Project revision
  contract_set_hash: bytes32
  map_intent_resolution_hash: bytes32
  requirement_ids: bounded array[1..256]<exact requirement ID>
  world_document_changeset_hashes: bounded array[0..64]<{exact World ref, ChangeSet hash}>
  scene_document_changeset_hashes: bounded array[0..1024]<{exact Scene ref, ChangeSet hash}>
  space_definition_changeset_hashes: bounded array[0..1024]<{exact Space ref, ChangeSet hash}>
  topology_changeset_hashes: bounded array[0..64]<{exact SpatialTopologyDefinitionV1 ref, ChangeSet hash}>
  owner_typed_document_changeset_hashes: bounded array[0..1024]<{expected document kind, exact owner-typed document ref, ChangeSet hash}>
  system_bundle_hashes: bounded array[0..128]<{exact system ref, bundle hash}>
  asset_changeset_hashes: bounded array[0..1024]<{exact Asset ref, ChangeSet hash}>
  procedural_delta_refs: bounded array[0..1024]<
    {exact ProceduralWorldDefinitionV1 ref,
     delta_id,
     allocation_manifest_ref/hash,
     semantic_graph_hash,
     candidate_artifact_semantic_hash,
     delta_content_hash}>
  navigation_intent_changeset_hashes: bounded array[0..1024]<{exact navigation owner ref, ChangeSet hash}>
  map_presentation_changeset_hashes: bounded array[0..1024]<{exact MapPresentationDefinitionV1 ref, ChangeSet hash}>
  expected_derived_artifact_refs: bounded array[0..1024]<exact generation-bearing ArtifactRefV1>
  target_profile_ids: bounded array[1..32]<exact Target Profile ID>
  budget_refs: bounded array[1..64]<exact budget ref>
  risk_class: closed RiskClassV1
  required_gate_ids: bounded array[0..256]<exact Gate ID>
  bundle_content_hash: SHA-256
```

changeset／delta配列は合わせて1件以上を必須とし、owner ref、expected kind、hashの組をcanonical orderで重複なく束ねる。`bundle_content_hash`はASCII `MIRAKAN_WORLD_AUTHORING_BUNDLE_V1`と自己Fieldだけを除くReceipt-free canonical bytesから計算する。Bundleは変更本文を複写せず、typed refとexact ChangeSet／delta／bundle hashだけを持つimmutable Staging proposalであり、`base_project_revision`不一致を自動rebaseしない。Qualificationは完成Bundle refをsubjectにする`WorldSourceQualificationSubjectV1(subject_kind=world_authoring_bundle)`とroot外Binding／Projectionへ分離し、Receipt／Fixture／BindingをBundle hashへ戻さない。

```text
WorldAuthoringContextV1
  project_id: StableId
  project_revision: exact Project revision
  contract_set_hash: bytes32
  authoring_selection_context_hash: optional bytes32
  world_ref: exact World ref
  scene_refs: bounded array[0..256]<exact Scene ref>
  space_refs: bounded array[0..256]<exact Space ref>
  topology_ref: optional exact SpatialTopologyDefinitionV1 ref
  topology_version: optional exact version
  viewport_bounds: optional finite spatial bounds
  target_profile_refs: bounded array[1..32]<exact Target Profile ref>
  source_document_refs: bounded array[1..1024]<{expected document kind, exact owner-typed document ref, source_hash: bytes32}>
  source_closure_hash: bytes32
  read_only_derived_artifact_refs: bounded array[0..1024]<exact generation-bearing ArtifactRefV1>
  capability_refs: bounded array[0..128]<exact Capability ref>
  budget_refs: bounded array[1..64]<exact budget ref>
  decision_and_lock_refs: bounded array[0..128]<exact decision or lock ref>
  omitted_ranges: bounded array[0..128]<bounded field range>
  continuation: optional hash-bound continuation
```

`topology_ref`と`topology_version`は共に存在するか共に省略する。`source_closure_hash`はcanonicalな`source_document_refs`のtyped ref、version、source hashを結ぶ。`continuation`は`omitted_ranges`が1件以上の時だけ存在し、request hash、source closure hash、revision、scopeをbindする。Contextはread-only／Disposable projectionであり、Source Document、ChangeSet／Commit、Save／Replay headerへ保存せず、Commit可能なidentityやchange primitiveとして受理しない。新規World作成中はContextを生成せず、`create_world_document` primitive Commit成功後の新Project revisionからGateway発行のexact `world_ref`を持つContextだけを生成する。Plan内local ID、表示名、予定pathからWorld IDを推測しない。

World Source変更の内部`WorldSourceChangePrimitiveKindV1`は`create_world_document | create_scene_document | compose_scene | move_entity_to_scene | set_spatial_topology_definition | set_spatial_partition_intent | set_procedural_world_definition | set_map_presentation_definition`のexact八discriminatorである。これは`ProjectChangeSetV1`内のtyped `ProjectChangePrimitiveV1` branchであってMCD Operation ID、Manifest row、Service allowlist、Provider／MCP Tool、aliasではなく、primitive名から`operation.*`を生成しない。Worldのcurrent MCD Operationは完全登録済み`operation.world.allocate_generated_stable_ids@1` exact一件、§20のWorld Discovery六件は別の`not_activated` planning familyである。consumer-owned Gameplay change primitiveをWorld branchへ登録しない。`generate_world_streaming_plan`はCommit済みSourceからDerived Planを作るCook job kind、Preview／Inspectはread-only internal actionであり、いずれもMCD Operation identityではない。

AI contextはWorld／Scene／Space、Topology、Cell plan、dependency、Target、budgetだけをbounded projectionする。consumer-owned Gameplay stateや進行単位をWorld contextへ合成しない。

複数Documentの変更は`WorldAuthoringBundleV1`のhashを承認済み`ProjectChangeSetV1`から参照して初めてCommitできる。Validate／Previewはread-onlyで、Plan、Bundle、ContextをCommit結果またはqualification evidenceとして扱わない。`WorldQualificationReceiptV1`はCommit済みSource revision、Topology／Scene／Space／Partition hash、Target Profile、Streaming／Navigation／LOD Artifact hash、system graph、fixture、correctness、performance、Review Receiptを結ぶDomain receiptであり、required Gate完了後だけ発行する。ChangeSet本文、Approval、共通Evidence envelopeの正本をWorldへ複写しない。

`create_cell`、`update_cell`、`delete_cell`、`set_cell_intent`、`replace_streaming_plan`、plan-local `cell_id`をtargetとする任意field writeは`WorldSourceChangePrimitiveKindV1`へ存在せず、公開または登録しない。未知またはaliasのCell write primitive／OperationをSource editへ近似変換せず`MIRAKAN-WORLD-DERIVED_CELL_WRITE`で失敗させる。Applyは`SpatialPartitionIntentV1`を含むProject Source ChangeSetだけを対象とし、Derived Plan、Cell descriptor、Runtime cellを直接操作しない。

## 12. Save、Replay、Migration境界

World Source revisionとRuntime instance stateを分離する。SaveはWorld／Scene／Space／persistent Entity Stable IDとcompatible source／artifact generationだけを投影し、Source Document、Runtime pointer、Cell handleを保存しない。

Save／Replay fieldはregistered State ownerとSave／Replay契約が列挙したものだけを含む。Worldはconsumer-owned Gameplay goal、outcome、progression、flow stateを保存しない。checkpoint、recording、Replay envelope、migration evidenceは[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)を参照する。

## 13. Diagnostic、failure、qualification

World diagnosticはWorld／Scene／Space／Entity Stable ID、Plan ID／plan-local Cell ID、source revision、reference path、Target、error code、remediationを含む。少なくともmissing ref、kind mismatch、topology cycle、duplicate owner、invalid partition、Cell dependency cycle、unbounded closure、stale plan、unsupported Target、migration unresolvedを区別する。

| Diagnostic ID | 条件 | 結果 |
|---|---|---|
| `MIRAKAN-WORLD-MAP_INTENT_AMBIGUOUS` | Map意味が一意でない | 候補とblocking questionを返す |
| `MIRAKAN-WORLD-UNKNOWN_STABLE_ID` | World／Scene／Space／Anchor不明 | fuzzy適用せず拒否 |
| `MIRAKAN-WORLD-TOPOLOGY_INVALID` | cycle、unknown edge、未宣言isolated Space | Sourceを拒否 |
| `MIRAKAN-WORLD-PARTITION_INVALID` | Cell bound、dependency、activation group不正 | Planをpublishしない |
| `MIRAKAN-WORLD-STREAMING_PLAN_STALE` | Source／Target／Toolchain／partition hash不一致 | stale Planを破棄し再Cook要求 |
| `MIRAKAN-WORLD-LOADING_PROGRESS_PLAN_STALE` | dependency closure／generation不一致 | 旧Snapshotを破棄し新session開始 |
| `MIRAKAN-WORLD-LOADING_PLAN_CAPACITY_EXCEEDED` | 65,536 unit以上 | previous valid Planを維持 |
| `MIRAKAN-WORLD-LOADING_CANCEL_REJECTED` | activating以後またはPolicy不許可 | World state不変のtyped rejection |
| `MIRAKAN-WORLD-LOADING_RETRY_REVALIDATION_FAILED` | Source／artifact／Capability／budget再検証不合格 | partial stateを使わずfailure表示維持 |
| `MIRAKAN-WORLD-DEPENDENCY_NOT_RESIDENT` | hard dependency不足 | activation groupをactiveにしない |
| `MIRAKAN-WORLD-ACTIVATION_PARTIAL` | activation groupの一部だけ成功 | target全体rollback、source維持 |
| `MIRAKAN-WORLD-BUDGET_EXCEEDED` | residency／I/O／hitch上限超過 | fallbackまたはactivation中止 |
| `MIRAKAN-WORLD-PROCEDURAL_NONDETERMINISTIC` | 同じcanonical入力でdelta semantic／Artifact hash不一致 | Delta／Artifact拒否 |
| `MIRAKAN-WORLD-PROCEDURAL_INVALID_OUTPUT` | Schema／bounds／connectivity／simulation validation不合格 | Delta全体破棄 |
| `MIRAKAN-WORLD-PRESENTATION_AUTHORITY_WRITE` | Map／World presentation、LOD、visibilityからauthoritative write | Build／conformance失敗 |
| `MIRAKAN-WORLD-DERIVED_CELL_WRITE` | Cell／Streaming Planへの直接／未知write operation | Sourceへ変換せず拒否 |
| `MIRAKAN-WORLD-CROSS_CELL_POINTER` | 永続pointer／Vendor handle参照 | Source／Cook拒否 |
| `MIRAKAN-WORLD-BUNDLE_STALE` | base revision／precondition不一致 | 再Resolve要求 |
| `MIRAKAN-WORLD-TARGET_UNSUPPORTED` | 意味同等fallbackなし | 対象Targetを非対応表示 |
| `MIRAKAN-WORLD-TILE_SOURCE_INVALID` | Tile／grid／orientation／revision不正 | Source／Cook拒否 |
| `MIRAKAN-WORLD-TILE_LAYOUT_CAPACITY_EXCEEDED` | region／candidate／changed count上限超過 | scan前に拒否 |
| `MIRAKAN-WORLD-TILE_TRANSFORM_MISMATCH` | D4 consumer結果／適用回数不一致 | 全consumer closure拒否 |
| `MIRAKAN-WORLD-TILE_GENERATION_STALE` | Source／Artifact／command generation不一致 | 旧generation維持 |
| `MIRAKAN-WORLD-TILE_ARTIFACT_PARTIAL` | 明示選択されたconsumer Artifact bindingがall-readyでない | publication拒否 |
| `MIRAKAN-WORLD-TILE_PREVIEW_STALE` | Preview／Commit再展開hash不一致 | Command拒否 |

Qualificationは次を含む。

- 全World schemaのvalid／invalid／boundary、UUIDv7 Stable IDのrename／delete／migration、World／Scene／Spaceの多対多ref、0 Scene procedural-only、0 entry topology、intentional isolation。
- greenfield `WorldAuthoringPlanV1`が`affected_world_refs=[]`＋World作成kind厳密1件でvalid、既存編集は1～64件必須、Commit前Context生成／推測World IDを拒否し、Commit後だけexact `world_ref`付きContextを生成するfixture。
- Playerなしでcamera observer、scripted anchor、simulation agentのtyped interest-source contractを登録するpositive fixtureと、unknown／removed owner、duplicate、dimension mismatchのnegative fixture。
- `fixture.world.authoring-semantics`: 共有Sceneを参照する複数World composition、複数Sceneからなる一World、Targetごとに異なるCell plan、Scene移動とCell再Cookがidentity／membershipを暗黙変更しないこと。
- Topology reachability／cycle／Target fallback、unknown／stale／cross-cell pointer、Undo／Redo、crash recovery、concurrent edit conflict。
- Cell全state transition、timeout、I/O failure、activation group atomicity、source Space維持、typed subject transfer、Save／Load／Replay state hash。
- `contract.world.loading-progress`: initial activation、Space transition、Save resumeで同じPlan／Snapshot契約を使い、0／10／99／100%の実作業由来進捗、cold I/O、verify failure、0／2,000 ms minimum display、source Space維持を検証する。
- Loading work unit 65,535 exact／65,536 exact+1、weight合計65,535、同Plan単調進捗、capacity failure時のpartial unit非公開、closure変更時の新generation、fake timer拒否、prefetch中Cancel、activating以後の`MIRAKAN-WORLD-LOADING_CANCEL_REJECTED`、明示Retryと`MIRAKAN-WORLD-LOADING_RETRY_REVALIDATION_FAILED`、lease／temporary Artifact解放、input／audio／keyboard／controller／screen reader projection。
- fixtureがRenderer／Collision／Navigationを例として明示選択したhard closureでは一要素failureを注入し、`MIRAKAN-WORLD-ACTIVATION_PARTIAL`、target全rollback、source Space／last-valid generation維持、stale Snapshot／progress再利用0件を検証する。別fixtureは任意owner bindingを選択して同じgeneric contractを通し、未選択ownerをhard closureへ暗黙追加しない。
- Tilemapのempty cell、負座標floor division、canonical cell／chunk順、C1 exact／plus-one bound、D4 single transform、stable animation phase、明示選択consumer Artifact集合のall-ready atomic publication、stale generation、Preview／Commit hash一致。
- Blockoutのdimension／segment／assembly bound、semantic矛盾、通常Domain cook、Promotion all-ready、external DCC 0件。
- `fixture.world.procedural-determinism`: 同じseed／input／Target／Toolchain／binding集合をfresh processで3回、Gateway call 0件で実行し、local semantic graph bytes／canonical order／`semantic_graph_hash`／`candidate_artifact_semantic_hash`の一致、Generator bound、connectivityを検証する。合格後だけcaller-issued request IDでGatewayを一回呼び、candidate／intent／request／Manifest／prepared Receipt／Publication ProjectionのProject tripleとcandidate hash、`local_id_count`、`allocated_uuid_count=local_id_count+1`、mapping全件＋`delta_id`を照合する。private Marker→canonical signed wrapper→Public Marker＋Projectのcrash window三箇所、同key retry byte equality、別request reuse、署名前public、partial Project／mapping publication、二回目allocation、local ID残存を拒否する。さらにPublic Markerのafter stateをProjectionへ置換するcycle fixture、Published payload／private Marker／prepared payloadのoperation、intent、request、idempotency、before／after stateのいずれか一Fieldだけを置換するfixtureを各一原因で拒否する。生成後に選択Artifactを削除して同じ入力から再生成し同じ`semantic_graph_hash`になること、削除中／再生成失敗／各hash単独tamperでは既存last-valid ArtifactとWorld generationを維持する。
- `fixture.world.procedural-provider-absent-empty`: bindingなし＋output 0件がcanonical skipで成功し、provider／placeholderを生成しない。
- `fixture.world.procedural-provider-absent-output`: bindingなし＋output 1件をDelta全rejectしlast-validを維持する。
- `fixture.world.procedural-provider-selected-missing-output`: selected bindingの`required_output_count=1`＋output 0件を全rejectする。
- `fixture.world.procedural-provider-selected-valid`: exact binding Document ref/content hash／resolved closure hash／output schema＋valid outputで一回だけ実行し成功する。
- `fixture.world.procedural-provider-selected-invalid`: stale binding／owner／schema／hash、provider failure、invalid outputを各単独原因で全rejectしlast-validを維持する。
- `fixture.world.procedural-core-only`: Physics、Navigation、spawn provider、selected binding、対応outputをすべて0件にしたprocedural-only Worldがcore schema／connectivity／bound検証だけで成功し、同seed再生成で同じDelta／Artifact hashになり、missing provider failure、偽Artifact、placeholder queryを生成しない。
- Spatial transition fixtureは全consumerで`type.world.spatial_transition_destination`をround-tripし、anchor optional／requiredのpositive、branch外Field、required anchor欠落、stale World／Topology／Space／Edge／Anchor、destination hash mismatchを各一原因で拒否する。
- `fixture.world.presentation-authority`: Map／World presentation、LOD、visibility、Camera、GPU結果からauthoritative writeを試みる全経路が`MIRAKAN-WORLD-PRESENTATION_AUTHORITY_WRITE`となり、Source／Runtime state hashが不変である。
- Compact 2D／3D spatial fixtureでframe／memory／load／activation hitch、Cell／prefetch比較、camera speed、HLOD authority equivalence、cold start／streamingを測定する。
- AI corpusはMapの6分類（world structure、scene composition、streaming、procedural layout、navigation、map presentation）、ambiguity／high-impact質問、World／Scene／Space／Cell／Navigation／Presentation分離、context外の表示名／pathからStable IDを推測しないこと、Source intent以外への直接write拒否を含む。
- `CreateCell`／`UpdateCell`／`DeleteCell`／`SetCellIntent`／`replace_streaming_plan`／未知aliasはすべて`MIRAKAN-WORLD-DERIVED_CELL_WRITE`となり、`SpatialPartitionIntentV1`と`WorldStreamingPlanV1`のhashが変化しないnegative fixture。
- `fixture.world.authoring-cross-view`: World Outline／Topology Graph／Scene Composition／Spatial View／AIの`WorldSourceChangePrimitiveKindV1`を64 scenarioで比較し、after-state hash一致100%。
- `fixture.world.authoring-intent`: holdout 240件（明確な6分類各30件、曖昧／High Impact 60件）を3 runし、明確Caseの`selected_kind`正解率97%以上、Blocking Caseの`question_required` recall 100%、存在しないStable IDを含むProposal 0件、World／Scene／Space／Cell identity誤変更0件、Derived／Runtime直接write提案0件。
- finite Gameplay goal／Resultを持たないendless、continuous simulation、procedural-only Worldがvalidである。
