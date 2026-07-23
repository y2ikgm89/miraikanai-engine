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

AI、Editor、Project C++はSource Documentとtyped operationを扱い、Runtime cell object、ECS pointer、Renderer resource、Physics／Navigation native handleを直接保存しない。cell activation state／phase／lifetimeはRuntime、capacity／backpressureはPerformance、representation selectionはLODが所有する。

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

`ProceduralWorldDefinitionV1`は`procedural_world_id`（UUIDv7 Stable ID）、exact `generator_contract_ref`、Qualified Definition／Native variantの`generator_implementation_ref`、`seed_policy: fixed | project_parameter | save_slot | session_derived`、`input_definition_refs` 0～1,024件、`layout_constraint_refs` 1～1,024件、exact `output_schema_ref`、正の`max_generation_steps`／`max_output_entities`、Targetごとに厳密に1件の`time_memory_budget_refs`、`determinism_contract_ref`、`validation_fixture_refs` 1～1,024件、`selected_validation_provider_bindings[0..64]`、`failure_policy: retry seed | fallback layout | abort`を持つ。budget値はRuntime capacity ownerを参照する。

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
```

binding配列は`validation_kind`、binding Document Stable IDのcanonical byte順でsortし、duplicateを拒否する。Document bytesのidentityは`binding_document_ref.content_hash`だけが表し、resolved closureを同じhashへ多重定義しない。`resolved_binding_closure_hash`は`SHA-256(ASCII "MIRAKAN_WORLD_PROCEDURAL_BINDING_CLOSURE_V1" || validation_kind || binding_document_refの全canonical bytes || owner_ref/hash || provider_contract_ref || output_schema_ref || required_output_count)`である。Document content hashまたはclosure hashの一方だけが一致しても受理せず、owner、provider、output schemaをlatestへ再解決しない。`required_output_count`は0～1,024で、0はprovider実行を要求するが出力recordを要求しないvalidatorだけに使う。Owner名、Capabilityの存在、output field名からproviderを推測選択しない。

UUID割当前のgenerator出力は`GeneratedWorldSemanticCandidateV1`である。これは`candidate_id`やStable IDを持たず、`procedural_world_ref`、base Project、generator／input／seed／Target／Toolchain／RNG／binding closure、`create_records[]`の連続`uint32 local_id`、local IDだけで結ぶ新規record間edge、既存Stable IDへのupdate／delete、local IDで表すgenerated anchor／portal／validation output、bounds、step count、`semantic_graph_hash`、`candidate_artifact_semantic_hash`を持つ。local IDは1から連続し、0、gap、duplicate、未定義local edge、Stable IDに見えるgenerator生成文字列を拒否する。

`semantic_graph_hash`は`SHA-256(ASCII "MIRAKAN_WORLD_GENERATED_SEMANTIC_GRAPH_V1" || procedural World exact ref || base Project revision || generator contract／implementation hash || input revisions／input hash || seed || Target ref/hash || Toolchain manifest hash || RNG stream manifest hash || binding closure集合 || local-ID正規化create graph || update／delete records || local-ID正規化Anchor／Portal／validation outputs || output bounds || generation step count)`である。`candidate_artifact_semantic_hash`はStable ID allocationやartifact container metadataを除くCook入力のlocal-ID正規化bytesを、ASCII `MIRAKAN_WORLD_GENERATED_ARTIFACT_SEMANTIC_V1`でhashする。random device、wall clock、worker completion順、network response、UUIDをgenerator入力または両hashへ含めない。

determinism gateは同じcanonical inputをfresh processで3回実行し、Gateway／Brokerを一度も呼ばず、各runのlocal-ID正規化semantic graph bytes、canonical record order、`semantic_graph_hash`、`candidate_artifact_semantic_hash`のbyte equalityだけを比較する。一つでも異なれば`MIRAKAN-WORLD-PROCEDURAL_NONDETERMINISTIC`として候補三件を破棄する。retry seedはfailure policyが許可した新しい明示seedでだけ新candidateを作り、同じseedの不一致を成功へ近似しない。

三run一致後、Authoring Command Gatewayをexact一回だけ呼び、次を同じrequest／receiptで発行する。

```text
WorldStableIdAllocationManifestV1
  allocation_request_id: UUIDv7
  project_ref/revision/document_set_hash
  procedural_world_ref
  semantic_graph_hash
  candidate_artifact_semantic_hash
  local_id_count
  mappings[local_id_count]:
    local_id: uint32
    stable_id: UUIDv7
  delta_id: UUIDv7
  manifest_hash: SHA-256

WorldStableIdAllocationReceiptV1
  allocation_operation_ref
  request_hash
  idempotency_key
  allocation_manifest_ref/hash
  project_ref/revision/document_set_hash
  semantic_graph_hash
  local_id_count
  allocated_uuid_count
  gateway_service_ref/hash
  receipt_hash: SHA-256
```

`mappings[]`はlocal ID昇順でexact `1..local_id_count`、Stable ID重複なし、`allocated_uuid_count=local_id_count+1`（mapping分＋`delta_id`）とする。Manifest hashはASCII `MIRAKAN_WORLD_STABLE_ID_ALLOCATION_MANIFEST_V1`、Receipt hashはASCII `MIRAKAN_WORLD_STABLE_ID_ALLOCATION_RECEIPT_V1`と各self hashを除くcanonical bytesから計算する。同じidempotency key＋request hashはbyte-identical Manifest／Receiptを返し、別requestでのkey再利用を拒否する。

`GeneratedWorldDeltaV1`はGateway発行`delta_id`、exact allocation Manifest ref／hash、Receipt ref／hash、候補の全入力、mapping適用済みcreate／update／delete、generated Anchor／Portal、typed validation outputs、output bounds、step count、`semantic_graph_hash`、`candidate_artifact_semantic_hash`、`delta_content_hash`を持つ。Source Delta、全内部ref、validation output、Preview、Cook入力、Commit Receiptは一つのManifest mappingだけを共有し、subsystem別再割当、二度目のGateway call、local ID残存を拒否する。`delta_content_hash`はASCII `MIRAKAN_WORLD_GENERATED_DELTA_V1`とself hashを除く全Fieldをhashし、UUIDを除外した再現性の判定には使わない。既存更新／削除は明示allowlistとexpected revisionを必須とし、World全置換、上限超過、absolute path、native pointer、Cooked Artifact本文を拒否する。

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
  assembly_transform: finite Transform3f with positive scale
  primitive_instances: bounded array[1..256]<{primitive_ref, local_transform}>
  composition_recipe_ref: optional exact recipe ref
  gameplay_anchor_refs: bounded array[0..256]<StableId>
  validation_fixture_refs: bounded array[1..256]<exact fixture ref>
  source_generation: uint64
```

各dimensionは0.01～10,000 m、radial segmentsは3～64、height segmentsは1～64、compact spatial assembly全体は4,096 primitive以下とする。assembly／local transformはfinite translation、normalized rotation、正のscaleで、負scale、NaN／Inf、zero-area surface、暗黙boolean由来のnonmanifold、Colliderとwalkable semanticの矛盾を拒否する。Primitiveは通常のTransform、Material、Renderer、Collision、Navigation SourceへCookし、Blockout専用Runtime objectを作らない。

`CreatePrimitiveMesh | CreateBlockoutAssembly | UpdateBlockoutPrimitive | PromoteBlockoutToMeshAsset`をAI／Editor共通のbounded operationとする。Promotion previewは元Stable ID、generation、pivot、bounds、Material slot、Collider／Navigation semantic、参照元を固定し、承認後に通常Mesh Sourceと対応表をatomic publishする。Sourceが明示選択したconsumer Artifact bindingのall-readyとgeneration一致前にはassembly置換を公開せず、Renderer／Collision／Navigationを固定必須集合にしない。元Sourceを自動削除せず、置換対象はexplicit ChangeSetに列挙する。C1 fixtureはexternal DCCを要求せず、6 primitive、dimension境界、4,096 primitive spatial assembly、Collider／Navigationを選択したcook、Promotion前後のbounds／pivot／Material slot、Undo／Redo、Save／Load、AI／手動operationのafter hash一致を検証する。external DCC Asset 0件のgeneric spatial blockoutのWorld activation smokeを完走できることをGateとする。

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
  validation_fixture_ids: bounded array[1..1024]<exact fixture ID>
  assumptions: bounded array[0..32]<typed assumption>
  blocking_questions: bounded array[0..7]<MapIntentResolutionV1 Question>
  fallback: optional exact fallback ref
  risk_class: closed RiskClassV1
  disposition: ready_to_stage | question_required | capability_unavailable | target_unsupported | budget_missing | rejected
```

`source_change_kinds`は`world_document | scene_composition | topology | partition | procedural_layout | map_presentation`の6種へ閉じる。既存World編集branchは`affected_world_refs[1..64]`を必須とし、すべてのaffected refが`project_revision`に実在しexpected kindと一致しなければならない。新規World作成branchだけは`affected_world_refs=[]`を許可するが、`create_document_kinds`がclosed World document kindを厳密に一件含み、`source_change_kinds`が`world_document`を含むことを必須にする。新規IDはCommit時にGatewayが発行し、Planへ存在しないWorld IDを先行記録しない。procedural branchの`selected_validation_provider_bindings`はCommit対象`ProceduralWorldDefinitionV1`とbinding Document ref/content hash、resolved closure hash、output schemaのset equalityで一致し、Planだけでproviderを追加・削除しない。`disposition=question_required`だけがQuestionを1～7件持ち、その他は0件とする。Planはread-only／proposal-onlyで、Source、Staging、Derived Artifact、Runtime stateを変更せず、Commit／Approval／Receipt権限を持たない。

```text
WorldAuthoringBundleV1
  bundle_id: StableId
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
  fixture_hashes: bounded array[1..1024]<{exact fixture ref, fixture hash}>
  risk_class: closed RiskClassV1
  required_gate_ids: bounded array[0..256]<exact Gate ID>
```

changeset／delta配列は合わせて1件以上を必須とし、owner ref、expected kind、hashの組をcanonical orderで重複なく束ねる。Bundleは変更本文を複写せず、typed refとexact ChangeSet／delta／bundle hashだけを持つimmutable Staging proposalであり、`base_project_revision`不一致を自動rebaseしない。

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

`topology_ref`と`topology_version`は共に存在するか共に省略する。`source_closure_hash`はcanonicalな`source_document_refs`のtyped ref、version、source hashを結ぶ。`continuation`は`omitted_ranges`が1件以上の時だけ存在し、request hash、source closure hash、revision、scopeをbindする。Contextはread-only／Disposable projectionであり、Source Document、ChangeSet／Commit、Save／Replay headerへ保存せず、Commit可能なidentityやOperationとして受理しない。新規World作成中はContextを生成せず、`CreateWorldDocument` Commit成功後の新Project revisionからGateway発行のexact `world_ref`を持つContextだけを生成する。Plan内local ID、表示名、予定pathからWorld IDを推測しない。

Source operationは`CreateWorldDocument`、`CreateSceneDocument`、`ComposeScene`、`MoveEntityToScene`、`SetSpatialTopologyDefinition`、`SetSpatialPartitionIntent`、`SetProceduralWorldDefinition`、`SetMapPresentationDefinition`である。consumer-owned Gameplay operationをWorld namespaceへ登録しない。`GenerateWorldStreamingPlan`はCommit済みSourceからDerived Planを作るCook jobで、Preview／Inspectはread-onlyである。

AI contextはWorld／Scene／Space、Topology、Cell plan、dependency、Target、budgetだけをbounded projectionする。consumer-owned Gameplay stateや進行単位をWorld contextへ合成しない。

複数Documentの変更は`WorldAuthoringBundleV1`のhashを承認済み`ProjectChangeSetV1`から参照して初めてCommitできる。Validate／Previewはread-onlyで、Plan、Bundle、ContextをCommit結果またはqualification evidenceとして扱わない。`WorldQualificationReceiptV1`はCommit済みSource revision、Topology／Scene／Space／Partition hash、Target Profile、Streaming／Navigation／LOD Artifact hash、system graph、fixture、correctness、performance、Review Receiptを結ぶDomain receiptであり、required Gate完了後だけ発行する。ChangeSet本文、Approval、共通Evidence envelopeの正本をWorldへ複写しない。

`CreateCell`、`UpdateCell`、`DeleteCell`、`SetCellIntent`、`replace_streaming_plan`、plan-local `cell_id`をtargetとする任意field writeは公開または登録しない。未知またはaliasのCell write operationをSource editへ近似変換せず`MIRAKAN-WORLD-DERIVED_CELL_WRITE`で失敗させる。Applyは`SpatialPartitionIntentV1`を含むProject Source ChangeSetだけを対象とし、Derived Plan、Cell descriptor、Runtime cellを直接操作しない。

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
- `fixture.world.procedural-determinism`: 同じseed／input／Target／Toolchain／binding集合をfresh processで3回、Gateway call 0件で実行し、local semantic graph bytes／canonical order／`semantic_graph_hash`／`candidate_artifact_semantic_hash`の一致、Generator bound、connectivityを検証する。合格後だけGatewayを一回呼び、mapping全件＋`delta_id`を一つのManifest／Receiptで発行し、Source／validation／Preview／Cook／Commitが同じmappingを使うこと、二回目allocationとlocal ID残存を拒否する。生成後に選択Artifactを削除して同じ入力から再生成し同じ`semantic_graph_hash`になること、削除中／再生成失敗／各hash単独tamperでは既存last-valid ArtifactとWorld generationを維持する。
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
- `fixture.world.authoring-cross-view`: World Outline／Topology Graph／Scene Composition／Spatial View／AIのDomain Operationを64 scenarioで比較し、after-state hash一致100%。
- `fixture.world.authoring-intent`: holdout 240件（明確な6分類各30件、曖昧／High Impact 60件）を3 runし、明確Caseの`selected_kind`正解率97%以上、Blocking Caseの`question_required` recall 100%、存在しないStable IDを含むProposal 0件、World／Scene／Space／Cell identity誤変更0件、Derived／Runtime直接write提案0件。
- finite Gameplay goal／Resultを持たないendless、continuous simulation、procedural-only Worldがvalidである。
