# Miraikanai Engine World／Scene／Level／Cell Contract

- 文書ID: mirakan.arch.rendering-world
- 状態: review
- 正本範囲: World／Scene／Levelのsource identity、Cellのplan-local identity、source composition／partition、streaming-plan authoring、Level transition／Loading presentation、Tilemap、Engine-native Blockout、reference closure、procedural source、Map要求resolution、World固有operation／diagnostic／qualification
- 非正本範囲: Runtime cell activation／phase／shared capacity、ECS／Gameplay component schema、Physics／Navigation behavior、Render／LOD execution、Asset transaction、Save／Replay envelope、Tool version、AI authorization、Evidence envelope、共通Schema／projection。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Math／Core utilities](../02-foundation/math-core.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Gameplay programming model](../03-authoring/gameplay-programming-model.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[Collision](../05-simulation/collision.md)、[Physics](../05-simulation/physics.md)、[Navigation](../05-simulation/navigation.md)、[Animation](../05-simulation/animation.md)、[Render Graph](render-graph.md)、[LOD](lod.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論と所有境界

World authoringはWorld、Scene、Level、Cellを異なるidentityと責務で定義し、Source compositionとTarget別streaming planを分離する。AI、Editor、Project C++はSource Documentとtyped operationを扱い、Runtime cell object、ECS pointer、Renderer resource、Physics／Navigation native handleを直接保存しない。

本書はsourceとstreaming-plan authoringだけを所有する。実行時のcell activation state、phase、job、lifetimeは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、capacity／backpressureは[Runtime performance／capacity](../04-runtime/performance-capacity.md)、representation selectionは[LOD](lod.md)が所有する。

Gameplay component、Physics、Collision、Navigation、Animation、Renderingは各OwnerのDomain dataをWorld entity／cellへ参照で結び、World文書が各Schemaを複写しない。Asset save、promotion、package generationは[Asset lifecycle](../03-authoring/asset-lifecycle.md)へ委譲する。

## 2. 正規用語とidentity

- `World`: Project内の空間／content universeとglobal composition root。
- `Scene`: authoring、ownership、collaborationのための再利用可能なsource shard。
- `Level`: gameplay progression、entry／exit、objective／rule scopeを表すlogical composition。
- `Cell`: streaming、residency、activation planの空間／logical partition unit。
- `Entity source`: Stable IDを持つWorld content record。Runtime entity instanceではない。
- `Layer`: visibility／authoring／variant selectionのorthogonal grouping。LevelやCellの別名ではない。
- `Map`: user request語でありcanonical object typeではない。resolverが意図をWorld／Level／Scene／navigation／presentationへ分類する。

World／Scene／Level／Entity source identityはProject Stable ID、source revision、display labelを分離する。path、filename、display name、array indexをidentityにしない。CellはSource Stable IDを持たずPlan-local `uint32`であり、SceneをLevel、LevelをCell、CellをRuntime chunkと同一視しない。

[Executable contracts](../02-foundation/executable-contracts.md#5-mcd共通envelope)が所有するexact `McdContractRefV1`、`ArtifactRefV1`、`GameSystemContractRefV1`を使い、WorldでFieldを再定義しない。Plan-local IDは同じPlan内だけで使い、Plan外のCell参照はPlanの`ArtifactRefV1`と組み合わせる。Levelが構成するGame Systemはtyped Command／Event／Snapshotで接続する。

## 3. 「Map」要求の解決規則

`MapIntentKindV1`は次の6 kindへ閉じる。

| Kind | 例 | 変更対象 |
|---|---|---|
| `world_structure` | 地域接続、町からDungeonへ移動 | `WorldTopologyDefinitionV1` |
| `playable_level` | Stage、boss room | `LevelDefinitionV1`＋World Source |
| `streaming` | seamless load、遠方軽量化 | `SpatialPartitionIntentV1`＋Derived Plan |
| `procedural_layout` | seed生成Dungeon | `ProceduralWorldDefinitionV1` |
| `navigation` | 歩行領域、飛行経路、NavMesh | [Navigation](../05-simulation/navigation.md) Definition／Profile |
| `map_presentation` | minimap、world map、marker、fog | presentation request routing |

`MapIntentResolutionV1`は`request_id`、`candidate_kinds[]`、`selected_kind`、`confidence_q16`、`evidence_requirement_ids[]`、`affected_stable_ids[]`、`blocking_questions[]`、`disposition`を持つ。

| Field | 規則 |
|---|---|
| `request_id` | UUIDv7 Stable ID。Resolution、Plan、Bundle、Provenanceを結ぶ |
| `candidate_kinds` | score降順`{kind, confidence_q16, evidence_requirement_ids}`、1～6件、重複不可 |
| `selected_kind` | `resolved`時は厳密に1件、その他0件 |
| `confidence_q16` | 0～65,535、1.0を65,535とする |
| `evidence_requirement_ids` | 1～64件 |
| `affected_stable_ids` | 実在確認済み、0～1,024件 |
| `blocking_questions` | 0～7件、各Questionは選択肢2～5件、推奨、影響、変更可能性 |
| `disposition` | `resolved | question_required | rejected` |

上位2候補差が9,830未満、Save／Level transition／authoritative state／Target／Asset license／memory capacityへ影響、layoutとpresentationの両解釈、新規／既存LevelをStable IDで特定不能のいずれかなら`question_required`とする。

曖昧な「マップを作る」「マップを開く」に対して新しい万能Map assetを生成しない。空間contentならWorld／Scene／Level、pathfindingなら[Navigation](../05-simulation/navigation.md)、画面表示なら本書の`MapPresentationDefinitionV1`へroutingする。

## 4. Source Document model

`WorldDocument`は既存Authoring ModelのDocument envelopeを使い、次のroot referenceだけを正本として持つ。

| Field | 型／規則 |
|---|---|
| `world_id` | UUIDv7 `StableId` |
| `topology_definition_ref` | exact version、厳密に1件 |
| `scene_document_refs` | 1～65,535件 |
| `level_definition_refs` | 1～16,384件 |
| `partition_intent_refs` | 0～64件、Target-independent |
| `procedural_definition_refs` | 0～1,024件 |
| `global_entity_refs` | 0～65,535件 |
| `world_rule_system_refs` | 0～128件 |
| `default_spawn_ref` | 0または1件 |
| `budget_profile_ref` | 厳密に1件。値と測定法は[Runtime performance／capacity](../04-runtime/performance-capacity.md)を参照 |

WorldDocumentへAsset binary、Navigation mesh、HLOD mesh、Streaming cell payload、C++ source、Vendor objectを埋め込まない。上限を超える場合はWorldまたはRegionを分割し、数値上限だけを拡大しない。

`SceneDocument`は編集競合、ownership、load範囲、diffを制御し、次を必須fieldとする。

| Field | 型／規則 |
|---|---|
| `scene_id` | UUIDv7 `StableId` |
| `document_bounds` | 2D AABBまたは3D AABB、local space |
| `entity_refs` | Stable Entity ID、0～1,048,576件 |
| `level_membership_refs` | 0～256件 |
| `layer_refs` | 0～256件 |
| `source_dependency_refs` | exact versionまたはcontent hash |
| `edit_ownership` | Authoring lock／merge policy |

Scene境界はRuntime Streaming境界を強制しない。CookerはSource IntentとTarget capacityから複数Sceneを一Cellへまとめることも、一Sceneを複数Cellへ分割することもできる。

Entity sourceはStable ID、Transform source、parent ref、Domain component document refs、Layer／tag、authoring metadata refを持つ。Gameplay／Physics／Navigation／Rendering component fieldは各Owner文書を参照し、World共通recordへflattenしない。

Source Documentのrevision、patch、ChangeSet、Undo／Redo、dirty state、mergeは[Project state](../03-authoring/project-state.md)を正本とする。本書はWorld Domain payloadのvalidationとreference関係だけを所有する。

## 5. World topologyとLevel definition

`WorldTopologyDefinitionV1`は`topology_id`、`version`、`world_ref`、`region_nodes[]`、`level_nodes[]`、`portal_edges[]`、`global_entry_refs[]`、`invariant_ids[]`を持つ。`topology_id`、Region、PortalはUUIDv7 `StableId`、World／Level／Anchor参照はUUIDv7 `StableId`＋Document revision、`version`は1から単調増加する`uint32`である。意味変更ごとにversionを増加し、canonical Source bytesのSHA-256をDocumentRefへ記録する。

| Topology field | 規則 |
|---|---|
| `region_nodes` | 0～4,096件。Stable Region ID、parent 0または1件 |
| `level_nodes` | 1～16,384件。Stable Level ID、Region 0または1件 |
| `global_entry_refs` | 1～256件。少なくとも一つがDefault |

Region parent graphはDAGとし、Levelを複数Regionへ同時所属させない。`PortalEdgeV1`は`portal_id`、`from_level_ref`、`from_anchor_ref`、`to_level_ref`、`to_anchor_ref`、`direction`、`condition_definition_ref`、`transition_policy_ref`、`preload_hint`、`fallback`を持つ。`direction`は`one_way | bidirectional`で、bidirectionalはCook時に二つの正規有向edgeへ展開する。存在しない参照、Defaultから到達不能で`intentionally_isolated=true`もないLevel、return path必須Levelの一方向trap、Portal ID重複、Region cycle、複数Region ownership、fallbackなしのTarget非対応必須edgeは`MIRAKAN-WORLD-TOPOLOGY_INVALID`で拒否する。

`LevelDefinitionV1`を次に固定する。LevelはGameplay上のplay可能単位でありgeometry containerではない。

| Field | 型／規則 |
|---|---|
| `level_id` | UUIDv7 `StableId` |
| `version` | 1から単調増加する`uint32`。Save／Replay意味または参照closure変更時に増加 |
| `content_hash` | canonical Source bytesのSHA-256 |
| `world_ref` | 厳密に1件 |
| `source_scene_refs` | 1～4,096件 |
| `entry_anchor_refs` | 1～256件 |
| `exit_anchor_refs` | 0～256件 |
| `level_game_system_refs` | 1～128件 |
| `objective_definition_refs` | 0～256件 |
| `spawn_plan_refs` | 0～1,024件 |
| `encounter_definition_refs` | 0～1,024件 |
| `navigation_profile_refs` | 0～64件 |
| `camera_profile_refs` | 0～64件 |
| `map_presentation_ref` | 0または1件 |
| `streaming_policy_ref` | 厳密に1件 |
| `save_replay_contract_ref` | authoritative Level stateがある場合必須 |
| `behavior_budget_refs` | Target Profileごとに厳密に1件。値はRuntime capacity ownerを参照 |
| `completion_contract` | success／failure／abortのtyped outcome |
| `fallback_contract` | Target非対応時の意味同等fallbackまたは理由 |

`level_game_system_refs`のうち厳密に一つが`level_gameplay` roleのauthoritative ownerである。LevelDefinition自身はObjective進捗、Combat State、Quest State、Character Stateを書かない。

同じSceneを複数Levelで利用できるが、instance identity、override scope、persistent state ownerを明示する。cross-Level refはpersistent ownerまたはtransition payloadを介し、unloaded instance pointerを保存しない。

`LevelRuntimeStateV1`はLevel gameplay Systemが所有し、`level_ref`、`runtime_instance_id`、`lifecycle_state`、`active_entry_ref`、`objective_state_refs[]`、`activated_system_instance_refs[]`、`authoritative_world_delta_ref`、`completion_outcome`を持つ。`lifecycle_state`は`inactive -> preparing -> ready -> activating -> active -> completing -> deactivating -> inactive`と、`preparing | activating | active | completing | deactivating -> faulted`のclosed state machineである。`runtime_instance_id`はgeneration付き`LevelInstanceHandle`で、Source／Save／Replay headerへ保存しない。

## 6. Spatial Partitionとstreaming-plan authoring

`SpatialPartitionIntentV1`はCreatorが編集するSourceであり、`partition_intent_id`、厳密に1件の`world_ref`、`spatial_dimension: 2d | 3d`、`interest_source_kinds`（`player | camera | portal | scripted_anchor | network_authority`のsubset）、physical unitとhysteresisを明示する`activation_radius_policy`、together／separate Stable ID set、1～128件のtyped ordered `priority_rules`、0～4,096件の`always_resident_refs`、Stable Entity／Layer／Sceneの`streamable_refs`、Targetごとに厳密に1件の`target_budget_refs`、typed `failure_policy`を持つ。Cell size、chunk file名、GPU heap offset、Backend page IDをSource Intentへ固定しない。

`WorldStreamingPlanV1`はCookerが生成するDerived Artifactであり、Editor／AIは直接編集しない。fieldは`plan_id`、`source_world_revision`、`contract_set_hash`、`target_profile_id`、`toolchain_manifest_hash`、`partition_intent_hash`、`cell_descriptors[]`、`dependency_edges[]`、`activation_groups[]`、`residency_budget`、`canonical_priority_order`、`fallback_plan_ref`、`artifact_hash`である。`residency_budget`の定義と解決値はRuntime capacity ownerを参照する。

`plan_id`は`SHA-256("mirakan.world_streaming_plan.v1" || source_world_revision || contract_set_hash || target_profile_id || toolchain_manifest_hash || partition_intent_hash)`で生成する32 byte `DerivedPlanId`である。`artifact_hash`はcanonical Plan bytesのSHA-256であり、`plan_id`と置換しない。`fallback_plan_ref`はexact `{plan_id, artifact_hash}`を使う。

`CellDescriptorV1`を次へ固定する。CellのSource Stable IDを新設せず、Plan-local IDとSource identityを分離する。

| Field | 規則 |
|---|---|
| `cell_id` | Plan内canonical orderから1..Nを割り当てる`uint32`。0 invalid、Source identity／Save／別Plan比較へ使用しない |
| `source_refs` | Entity／Scene／Layer／Levelのexact Stable ref、1～1,048,576件 |
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

## 7. Level transition intent

Level transitionはsource／target Level、trigger intent、loading presentation ref、persistent entity／state policy、required Cell set、precondition、failure／cancel policyをSourceとして定義する。実行phase、writer、async job、timeout、Save checkpointはRuntime Ownerへ委譲する。

`LevelTransitionRequestV1`は`request_id: uint64`、exact `source_level_instance_ref`、exact `portal_ref`、`target_level_ref`、`target_entry_anchor_ref`、`requesting_system_ref`、`requested_tick: uint64`、`player_or_party_transfer_refs[0..256]`、`precondition_snapshot_hash`、exact `transition_policy_ref`を持つ。`request_id`はWorld runtime instance内で1から単調増加し0 invalid、Save／別session比較に使わない。Portal無効、Source非active、Target／Anchor不一致、stale condition hashをprefetch前に拒否する。

transition中に旧／新LevelのEntity identityを再利用せず、persistent identityは明示されたownerとhandoff recordを持つ。target dependencyが不足する場合は部分Activationやdefault Levelへ黙って進まず、blocking reasonと登録済みfallbackを返す。

### 7.1 Loading／prefetch presentation

Loading presentationはWorld activationの結果を表示するread-only projectionであり、activationのOwnerではない。初回起動、Level遷移、Save再開は同じ契約を使い、Runtime Orchestratorがdependency closureからPlanを作り、immutable Snapshotを公開する。

```text
LevelTransitionPresentationPolicyV1
  policy_id: StableId
  mode: seamless | overlay | blocking
  loading_ui_document_ref: exact document ref
  input_context_ref: exact input context ref
  audio_snapshot_ref: exact audio snapshot ref
  minimum_display_monotonic_ms: uint32
  cancel_allowed_until: prefetching | resident
  failure_presentation_ref: exact document ref
  retry_policy: none | explicit_user_retry
  accessibility_announcement_policy_ref: exact policy ref

LoadingProgressPlanV1
  plan_id: DerivedPlanId
  subject_kind: initial_boot | level_transition | resume_save
  subject_request_ref: exact generation-bearing typed request ref
  dependency_closure_hash: bytes32
  work_units: bounded array[1..65,535]<{kind, exact_target_ref, positive_weight_q16}>
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
```

`minimum_display_monotonic_ms`は0～2,000で、表示flash抑制だけに使う。各work unitのkindは`io_bytes | artifact_verify | dependency_ready | activation_group | state_transfer`のclosed enumであり、exact対象refと正のweightを必須とする。Runtime Orchestratorは表示開始前に確定したdependency closureの実作業を1～65,535件だけ列挙し、canonical orderで合計65,535へnormalizeする。同じPlan generationの`completed_weight_q16`は0～65,535で単調非減少とし、完了したunitのweightだけを加算する。65,535 unitは各weightが正で正規化可能なら受理し、exact +1の65,536 unitはPlan materialization前にtyped `loading_plan_capacity_exceeded`で拒否する。unitを省略、結合、truncateして成功へ近似せず、previous valid PlanとWorld generationを維持してpartial unitまたはpartial Planを公開しない。fake timer、frame count、UI animation、spinner、未列挙jobを進捗へ混ぜない。closure hashが変わればbarを巻き戻さず、新しいsession、Plan、generationを発行する。

I/O、verify、timeoutは`real_time`／`async_io` domainで進められるが、activation、Character transfer、Save state適用はRuntime Ownerの正規tick boundaryだけでcommitする。Target activation groupとRenderer／Collision／Navigationを含むhard dependency closureがすべてReadyになった後だけ新generationをatomic publishする。activationまたはtransferが失敗した場合はtargetを全rollbackし、旧Levelをactiveのまま維持する。旧Levelを先に破棄して空Worldを露出せず、新Level active後にだけtransferし、その後に旧Levelをdeactivateする。

CancelはPolicyが許可し、phaseが`prefetching | resident`以下の時だけ受理する。以後は`can_cancel=false`である。受理後はinflight I/Oをcancelまたはbounded drainし、leaseとtemporary Artifactを解放してSource LevelまたはTitleを維持する。Retryは`explicit_user_retry`時の人間の明示操作だけで、新request／session IDを発行し、Source revision、Portal condition、Save checksum、Target Capability、storage／memory budgetを再検証する。partial activation、古いSnapshot、古いprogressを再利用しない。

`blocking`はGameplay inputをLoading contextへ切り替え、Cancel／Retry／system UIだけを許可する。`overlay | seamless`でもtransfer前の入力をTargetへ配送しない。Audioは`audio_snapshot_ref`でcontinue／fade／muteを明示し、無音を暗黙defaultにしない。Accessibilityはphase変更を即時通知し、同一phaseの進捗通知を10%境界かつ1秒以上の間隔へ制限する。色だけへ依存せず、progress、phase、Cancel／Retry、failure reasonへkeyboard／controller／screen readerから到達可能にする。Loading UI、tips、Audio fade、読み上げはauthority、progress、timeoutへ逆入力しない。

## 8. 参照と依存closure

全World referenceはStable ID、expected document kind、required／optional、version compatibilityを持つ。CookerはScene nesting、Entity parent、Level composition、Cell membership、Domain component asset、transition targetのclosureをcanonical orderで解決する。

required ref欠損、kind mismatch、cycle、duplicate owner、stale revisionはhard errorとする。optional ref欠損はSourceでfallbackが宣言された場合だけ許可する。Runtimeはdisplay nameやpathからrefを再解決しない。

Asset artifact、Navigation artifact、LOD／HLOD representation、Renderer material／geometryはgeneration付きrefでPlanへ入り、異なるsource／artifact generationを一つのCell activationへ混在させない。

## 9. Procedural World source

Procedural Worldはgenerator Stable ID、typed parameter、seed semantics、input asset refs、bounded output scope、determinism class、generated Source ownership、regeneration／migration policyを持つ。GeneratorはProject Source DocumentへのChangeSetを生成し、Runtime objectやnative resourceを直接生成して正本化しない。

`ProceduralWorldDefinitionV1`は`procedural_world_id`（UUIDv7 Stable ID）、exact `generator_contract_ref`、Qualified Definition／Native variantの`generator_implementation_ref`、`seed_policy: fixed | project_parameter | save_slot | session_derived`、`input_definition_refs` 0～1,024件、`layout_constraint_refs` 1～1,024件、exact `output_schema_ref`、正の`max_generation_steps`／`max_output_entities`、Targetごとに厳密に1件の`time_memory_budget_refs`、`determinism_contract_ref`、`validation_fixture_refs` 1～1,024件、`failure_policy: retry seed | fallback layout | abort`を持つ。budget値はRuntime capacity ownerを参照する。

`GeneratedWorldDeltaV1`は`delta_id`、`procedural_world_ref`、`base_project_revision`、`generator_contract_hash`、`generator_implementation_hash`、`input_hash`、`seed`、`rng_stream_manifest_hash`、`create_records[0..max_output_entities]`、`update_operations[0..max_output_entities * 4]`、`delete_stable_ids[0..max_output_entities]`、`generated_anchor_refs[]`、`generated_portal_refs[]`、`output_bounds`、`generation_step_count`、`output_hash`を持つ。

`delta_id`はTrusted Staging Broker発行UUIDv7 Stable IDである。create recordはGateway割当前の`uint32 local_id`を1から使い、0 invalid、Delta内重複errorとする。検証後にGatewayがlocal IDをStable IDへ対応付け、Sourceへlocal IDを残さない。既存更新／削除は明示allowlistとexpected revisionを必須とし、World全置換、上限超過、absolute path、native pointer、Cooked Artifact本文を拒否する。

同じgenerator version、input revisions、seed、parameterから同じStable ID assignmentとcanonical outputを生成する。random device、wall clock、worker completion順、network responseをdeterministic generatorの入力にしない。

生成結果は通常のScene／Entity／Cell validationとreviewを通り、手編集領域を無断上書きしない。外部Tool／generator versionとartifact hashは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)を参照する。

## 10. Navigation、Simulation、Renderingとの境界

WorldはNavigation build volume／modifier、Physics／Collision component、Animation component、Render／Material／Light componentへのtyped document refを保持できるが、各Domain fieldや挙動を定義しない。Domain artifactのcook／validation resultをCell dependency closureへ取り込む。

Navigation queryやWorld movement、Physics body activation、Animation sampling、Render visibilityは各OwnerとRuntime schedulingが所有する。World Cell stateをそれらのauthoritative simulation stateとして兼用しない。

[LOD](lod.md)はresident candidateからrepresentationを選び、[Render Graph](render-graph.md)はactive Cell由来の`WorldRenderPacket`を実行する。Worldはselection formula、visibility algorithm、render passを所有しない。

`MapPresentationDefinitionV1`は`map_presentation_id: StableId`、`presentation_kind: minimap | world_map | level_map | navigation_overlay`、exact `world_or_level_ref`、`projection_policy: orthographic_2d | authored_2d | projected_3d`、`layer_refs[1..128]`、`marker_style_refs[0..512]`、typed `marker_source_contract_refs`、optional `fog_policy_ref`、`accessibility_profile_refs[1..32]`、exact `localization_namespace_ref`、Targetごとのexact `render_budget_refs`、`fallback_contract`を持つ非authoritative Sourceである。Marker／fog／cursorからWorld／Quest／Objective／Navigation costを直接writeせず、入力はtyped `MapInteractionCommandV1`としてNavigation／Quest ownerへ送る。

### 10.1 Tilemap source、cook、publication

Tile chunkはTilemap内部のedit、cook、culling単位であり、World Cell、Region、Level、activation groupではない。Tilemapはresident／active CellのAsset closureへ参加するが、chunk単独でEntity、Objective、Portal、authoritative gameplayをactivateしない。次の12型をWorldの唯一の正本とする。

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

`TileGridV1`のtexel extentは両軸正、`pixels_per_unit`はfiniteかつ正とする。stagger／hex fieldはorientationと整合しなければならない。C1 orientationは`orthogonal`だけをQualifiedとし、他のorientationと`typed_object_stamp | image | height_semantic`はC2の個別Qualification前に拒否する。`RectI64`／`RectU32`はinclusive min、exclusive maxで、empty、overflow、負のunsigned extentを拒否する。

空cellはrecord欠落で表し、`tile_id=0`等のsentinelを保存しない。chunk extentは各軸8～256の2冪、Referenceは32×32である。`local_coord`は`[0, chunk_extent_tiles)`内で一意とし、cellsを`local_y, local_x, tile_id`、chunksを`chunk_y, chunk_x`でcanonicalizeする。負のWorld tile coordinateはfloor divisionでchunkへ写像し、`local = world - chunk * extent`を必ず非負にする。言語の負剰余を使わない。

C1 boundは一TileSet 65,535 Tile、一Tilemap 64 Layer、全Layer合計16,777,216 occupied cell、一chunk 4,096 draw spanである。Tile animationのdurationは各1～60,000 ms、合計24時間以下とする。重複／範囲外cell、unknown Tile、TileSet revision mismatch、Source bounds／count overflow、unsupported orientation、dangling parent／Tileset、非finite offsetをtyped validation failureとして拒否する。外部global tile ID、配列index、path、表示名をStable Tile IDにしない。

cell `transform`は正方形格子の二面体群D4のclosed enumである。Cookerはcell中心を基準に、Renderer UV、Sprite pivot、Collision polygon、Navigation source、terrain edge／corner tagへ同じD4 transformをちょうど一度適用する。`TileDrawSpanV1.cooked_cell_transform_state`は適用済みを示し、consumerは再適用しない。いずれかのconsumerが同じ変換を表現できない、適用回数または結果hashが一致しない場合は`consumer_transform_mismatch`でclosure全体を失敗させ、Presentationだけを成功させない。`stable_cell_offset`はTilemap ID、Layer ID、World tile coordinate、Tile IDのcanonical hashだけから決め、load順、chunk residency順、worker順をseedにしない。

Tile editはimmutableな新Artifactを、変更region外周1 tile、terrain dependency radius、Collider seam、Navigation overlapまで再Cookする。Renderer、Collision、Navigationのrequired ArtifactがすべてReadyで、dependency hashとsource generationが一致した後だけ、World activation groupとTilemap generationを一つのpublication boundaryでatomic publishする。Presentation-only変更でCollision／Navigationを再利用する場合もexact dependency hashを検証する。一つでもfailed／cancelled／staleなら旧generationを維持し、partial artifact、空Tile、無衝突状態を公開しない。active authoritative regionのCollider／NavigationをPresentationより先にevictせず、Cell all-or-nothing activationをchunk単位へ弱めない。

AIとEditorは同じbounded `TileLayoutCommandV1`を使い、AIが巨大なtile ID配列を直接生成することを拒否する。`region`のinclusive-min／exclusive-max areaはoverflow-safeなwide integerで事前計算し、canonical preflightはcell payloadのscanやcandidate materializationなしに決定論的なexpansion candidate countを導出する。region areaとcandidate countはそれぞれ`max_examined_tile_count`、Target Profileのcommand examined-tile limit、C1全Layer ceiling 16,777,216以下でなければならず、`max_changed_tile_count <= max_examined_tile_count`も必須とする。16,777,216 examined tileは他の二上限も許せば受理し、exact +1の16,777,217、area積overflow、いずれかの上限超過はscan／expansion開始前にtyped `tile_layout_capacity_exceeded`で拒否する。

Engineはaccepted commandだけをexpected revision／generation上で決定論的に展開し、allowed set、接続、到達性、変更数を検証する。capacity failureでregionをtruncate、sample、分割、近似せず、candidate、changed tile、Artifactを部分公開しない。Commit直前に同じ入力から再展開し、canonical preview expansion hashと一致しなければ`preview_commit_hash_mismatch`として拒否する。stale generation、unknown Tile、revision mismatchもtyped rejectionであり、全拒否経路でprevious Tilemap／World generationと既存Renderer／Collision／Navigation Artifactを維持する。

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

各dimensionは0.01～10,000 m、radial segmentsは3～64、height segmentsは1～64、compact Level全体は4,096 primitive以下とする。assembly／local transformはfinite translation、normalized rotation、正のscaleで、負scale、NaN／Inf、zero-area surface、暗黙boolean由来のnonmanifold、Colliderとwalkable semanticの矛盾を拒否する。Primitiveは通常のTransform、Material、Renderer、Collision、Navigation SourceへCookし、Blockout専用Runtime objectを作らない。

`CreatePrimitiveMesh | CreateBlockoutAssembly | UpdateBlockoutPrimitive | PromoteBlockoutToMeshAsset`をAI／Editor共通のbounded operationとする。Promotion previewは元Stable ID、generation、pivot、bounds、Material slot、Collider／Navigation semantic、参照元を固定し、承認後に通常Mesh Sourceと対応表をatomic publishする。Renderer／Collision／Navigation Artifactのall-readyとgeneration一致前にはassembly置換を公開しない。元Sourceを自動削除せず、置換対象はexplicit ChangeSetに列挙する。C1 fixtureはexternal DCCを要求せず、6 primitive、dimension境界、4,096 primitive Level、Collider／Navigation cook、Promotion前後のbounds／pivot／Material slot、Undo／Redo、Save／Load、AI／手動operationのafter hash一致を検証する。external DCC Asset 0件のarenaをTitleからResultまで完走できることをGateとする。

## 11. Authoring bundleとAI／Editor UX

World authoring bundleは対象World／Level／Scene revisions、selected scope、typed Domain document refs、streaming-plan preview ref、validation summaryを束ねる。共通bundle／projection／operation envelopeは[Executable contracts](../02-foundation/executable-contracts.md)の定義を再利用する。

`WorldAuthoringPlanV1`は`plan_id`、`project_revision`、`contract_set_hash`、`map_intent_resolution_hash`、`requirement_ids[1..256]`、`target_profile_ids[1..32]`、`affected_world_refs[1..64]`、`affected_level_refs[0..1024]`、`create_document_kinds[0..64]`、`source_change_kinds[1..6]`、`required_system_refs[0..128]`、`required_capability_refs[0..128]`、`budget_refs[1..64]`、`derived_build_jobs[0..256]`、`validation_fixture_ids[1..1024]`、`assumptions[0..32]`、`blocking_questions[0..7]`、`fallback`、`risk_class`、`disposition`を持つ。`disposition`は`ready_to_stage | question_required | capability_unavailable | target_unsupported | budget_missing | rejected`で、Plan自体にCommit権限はない。

`WorldAuthoringBundleV1`は`bundle_id`、`project_id`、`base_project_revision`、`contract_set_hash`、`map_intent_resolution_hash`、`requirement_ids[]`、`world_document_changeset_hashes[]`、`topology_changeset_hashes[]`、`level_definition_changeset_hashes[]`、`gameplay_definition_changeset_hashes[]`、`system_bundle_hashes[]`、`asset_changeset_hashes[]`、`procedural_delta_hashes[]`、`navigation_intent_changeset_hashes[]`、`map_presentation_changeset_hashes[]`、`expected_derived_artifact_refs[]`、`target_profile_ids[]`、`budget_refs[]`、`fixture_hashes[]`、`risk_class`、`required_gate_ids[]`を持つ。変更本文を埋め込まずStaging hashとStable IDだけを束ねる。

`WorldAuthoringContextV1`は`project_id`、`project_revision`、`contract_set_hash`、optional `authoring_selection_context_hash`、`world_ref`、`scene_refs[0..256]`、`level_refs[0..256]`、`topology_ref`、`topology_version`、optional `viewport_bounds`、`target_profile_refs[1..32]`、`source_document_refs[1..1024]`、`read_only_derived_artifact_refs[0..1024]`、`capability_refs[0..128]`、`budget_refs[1..64]`、`decision_and_lock_refs[0..128]`、`omitted_ranges[0..128]`、`continuation`を持つread-only projectionである。[Project state](../03-authoring/project-state.md)所有の`AuthoringSelectionContextV1`から生成し、Source／Commit／Replay headerへ保存しない。

Worldの変更operationはcreate／update World、Scene、Level、`SetSpatialPartitionIntent`による`SpatialPartitionIntentV1` Source編集、compose Scene、move Entity source、edit Layer、create transitionだけをDomain actionとして登録する。`GenerateWorldStreamingPlan`はCommit済みSourceから`WorldStreamingPlanV1`を生成するCook jobであり、`PreviewWorldStreamingPlan`／`InspectCellDescriptor`はPlan ID／artifact hash／plan-local `cell_id`を取り読み取り結果だけを返す。

Source Domain Operationは`CreateLevelDefinition`、`SetLevelSourceScenes`、`SetLevelEntryExitContract`、`SetLevelGameplayComposition`、`CreatePortal`／`UpdatePortalContract`／`DeletePortal`、`MoveEntityToScene`、`SetSpatialPartitionIntent`、`SetProceduralWorldDefinition`、`SetMapPresentationDefinition`である。複数Documentの変更はexact ChangeSet hashを`WorldAuthoringBundleV1`で束ね、承認後に一つの`ProjectChangeSetV1`としてCommitする。Query／Job IDは`mirakan.worlds.resolve_map_intent`、`mirakan.worlds.validate_bundle`、`mirakan.worlds.preview_bundle`で、Source／Derived stateをwriteしない。

`CreateCell`、`UpdateCell`、`DeleteCell`、`SetCellIntent`、`replace_streaming_plan`、plan-local `cell_id`をtargetとする任意field writeは公開または登録しない。未知またはaliasのCell write operationをSource editへ近似変換せず`MIRAKAN-WORLD-DERIVED_CELL_WRITE`で失敗させる。Applyは[Project state](../03-authoring/project-state.md)のChangeSetと`SpatialPartitionIntentV1`だけを対象とし、Derived Plan／Cell descriptor／Runtime cellを直接操作しない。

Previewは対象revision、composition graph、Cell membership／dependency、Target plan、missing closure、estimated capacity class、fallback、diagnosticを示す。authorization、approval、sandboxは[AI Security／Approval](../01-governance/ai-security-approval.md)だけが決定する。

`WorldQualificationReceiptV1`はSource revision、Topology／Level／Partition hash、Target Profile、Streaming／Navigation／LOD Artifact hash、System graph、fixture、correctness、performance、Review Receiptを結ぶDomain receiptで、共通Evidence envelopeは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)の正本を使う。

## 12. Save、Replay、Migration境界

World Source revisionとRuntime instance stateを分離する。SaveはWorld／Level／Scene／Entity Stable IDとcompatible source／artifact generationへDomain stateを投影し、Source Document、Runtime pointer、Cell handleを丸ごと保存しない。

Save対象の各Level instanceはGatewayが発行するUUIDv7 `LevelSaveInstanceId`を厳密に1件持つ。同じSource Level `StableId`から複数instanceを作る場合もそれぞれ別の`LevelSaveInstanceId`を発行し、Projectの全session／Save／checkpoint lineageを通じて重複／再利用せず、同じ保存instanceのcheckpoint／migrationを跨いで維持する。Save recordはSource LevelのUUIDv7 `StableId`／definition version、`LevelSaveInstanceId`、active entry Anchor `StableId`、authoritative Domain stateを結び、Runtime handleは含めない。missing／duplicate／不正UUIDv7の`LevelSaveInstanceId`をSave／Load validation errorとする。

Loadは各`LevelSaveInstanceId`に対し現sessionの新しいgeneration付き`LevelInstanceHandle`を厳密に1件割り当て、one-to-one remapをload transactionの間だけ保持する。保存前のhandle value／index／generationを復元または優先せず、異なる`LevelSaveInstanceId`を同じhandleへ結び付けない。`LevelSaveInstanceId`はSave identity、`LevelInstanceHandle`はephemeral Runtime identity、Level `StableId`はSource identityであり、相互に置換しない。

checkpoint時刻、recording、Replay envelope、migration evidenceは[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)を参照する。本書はWorld identity mapping、persistent owner、missing／renamed／split sourceのDomain migration ruleだけを提供する。

Migrationはold／new Stable ID mapping、semantic change、default／manual resolution、orphan dispositionを明示し、display nameやpathの近似一致で自動復元しない。

## 13. Diagnostic、failure、qualification

World固有diagnosticはWorld／Scene／Level／Entity Stable ID、Plan ID／plan-local Cell ID、source revision、reference path、Target、error code、remediationを含む。少なくともmissing ref、kind mismatch、composition cycle、duplicate owner、invalid partition、Cell dependency cycle、unbounded closure、stale plan、unsupported Target、migration unresolvedを区別する。

| Closed code | 条件 | 結果 |
|---|---|---|
| `MIRAKAN-WORLD-MAP_INTENT_AMBIGUOUS` | Map意味が一意でない | 候補とblocking questionを返す |
| `MIRAKAN-WORLD-UNKNOWN_STABLE_ID` | World／Level／Portal／Anchor不明 | fuzzy適用せず拒否 |
| `MIRAKAN-WORLD-TOPOLOGY_INVALID` | cycle、trap、unreachable、参照不正 | Cook／Commit拒否 |
| `MIRAKAN-WORLD-LEVEL_OWNER_INVALID` | Level gameplay ownerが0または複数 | Activation拒否 |
| `MIRAKAN-WORLD-STREAMING_PLAN_STALE` | Source／Target／Toolchain hash不一致 | 再Cook要求 |
| `MIRAKAN-WORLD-LOADING_PLAN_STALE` | dependency closure／Plan generation不一致 | 新session／Planを発行 |
| `MIRAKAN-WORLD-LOADING_PLAN_CAPACITY_EXCEEDED` | work unitが65,535件超 | `loading_plan_capacity_exceeded`、previous Plan／World generation維持 |
| `MIRAKAN-WORLD-DEPENDENCY_NOT_RESIDENT` | hard dependency不足 | Cellをactiveにしない |
| `MIRAKAN-WORLD-ACTIVATION_PARTIAL` | activation groupの一部だけ成功 | 全体rollback |
| `MIRAKAN-WORLD-TILE_SOURCE_INVALID` | duplicate／out-of-range cell、unknown Tile、revision mismatch、overflow、unsupported orientation | Source／Cook拒否 |
| `MIRAKAN-WORLD-TILE_LAYOUT_CAPACITY_EXCEEDED` | region area／candidate／changed countがcommand、Target、C1上限超過 | `tile_layout_capacity_exceeded`、scan前に拒否 |
| `MIRAKAN-WORLD-TILE_TRANSFORM_MISMATCH` | D4のconsumer結果／適用回数不一致 | 全consumer closureを拒否 |
| `MIRAKAN-WORLD-TILE_GENERATION_STALE` | Source／Artifact／command generation不一致 | 旧generation維持、再Cook／再Preview |
| `MIRAKAN-WORLD-TILE_ARTIFACT_PARTIAL` | Renderer／Collision／Navigationがall-readyでない | publication拒否 |
| `MIRAKAN-WORLD-TILE_PREVIEW_STALE` | Preview／Commit再展開hash不一致 | Command拒否 |
| `MIRAKAN-WORLD-BUDGET_EXCEEDED` | residency／IO／hitch上限超過 | fallbackまたはtransition中止 |
| `MIRAKAN-WORLD-PROCEDURAL_NONDETERMINISTIC` | 同じ入力でoutput hash不一致 | Artifact拒否 |
| `MIRAKAN-WORLD-PROCEDURAL_INVALID_OUTPUT` | Schema／connectivity／playability不合格 | delta破棄 |
| `MIRAKAN-WORLD-PRESENTATION_AUTHORITY_WRITE` | Map／LOD／visibilityからGameplay write | Build／conformance失敗 |
| `MIRAKAN-WORLD-DERIVED_CELL_WRITE` | Cell／Streaming Planへの直接／未知write operation | Sourceへ変換せず拒否 |
| `MIRAKAN-WORLD-CROSS_CELL_POINTER` | 永続pointer／Vendor handle参照 | Source／Cook拒否 |
| `MIRAKAN-WORLD-BUNDLE_STALE` | base revision／precondition不一致 | 再Resolve要求 |
| `MIRAKAN-WORLD-TARGET_UNSUPPORTED` | 意味同等fallbackなし | 対象Targetを非対応表示 |

Failure時に別Level、Portal、Assetへ名前類似で置換しない。Gameplay意味が変わるfallbackはGameSpec変更として扱う。

Qualificationは次のDomain fixtureを持つ。

- 全Domain schemaのvalid／invalid／boundary、UUIDv7 Stable IDのrename／delete／migration、SceneとLevelの多対多参照とowner一意性。
- `world_authoring_semantics_v1`: 共有Sceneを参照する二Level、一Levelを構成する複数Scene、Targetごとに異なるCell planを同時表現する。
- `MoveEntityToScene`、`SetLevelSourceScenes`、Cell再Cookがidentity／membershipを暗黙変更しないこと。
- Topology reachability／trap／cycle／Target fallback、unknown／stale／cross-cell pointer negative test、Undo／redo／crash recovery／concurrent edit conflict。
- Cell全state transition、cancel、timeout、I/O failure、activation group atomicity、旧Level維持、Level transition／Character transfer／lease解放、Save／Load／Replay state hash。
- Loadingの実作業unit 65,535 exact／65,536 exact +1、weight合計65,535、同Plan単調進捗、capacity failureでpartial unit非公開、closure変更時の新generation、fake timer拒否、Cancel boundary、明示Retry、input／audio／accessibility projectionを検証する。
- Tilemapのempty cell、負座標floor division、canonical cell／chunk順、C1 exact／plus-one bound、examined tile 16,777,216 exact／16,777,217 exact +1のscan前capacity rejection、D4 single transformのRenderer／pivot／Collision／Navigation／terrain一致、stable animation phase、三Artifact all-ready atomic publication、stale generation、Preview／Commit hash一致を検証する。
- Blockoutのdimension／segment／assembly／Level bound、semantic矛盾、通常Domain cook、Promotion all-ready、external DCC 0件fixtureを検証する。
- 同じSource Levelの複数saved instance、checkpoint連鎖、missing／duplicate／不正`LevelSaveInstanceId`、Loadごとの新しい`LevelInstanceHandle`、one-to-one remap、保存handle復元拒否を検証するidentity fixture。
- inactive／resident／active境界からauthoritative処理が漏れず、Presentation／LOD／Camera／GPU結果からauthorityへ逆入力しないこと。
- 同じseed／input／Target／Toolchainから同じprocedural output hash、Generator bound、connectivity、entry-to-objective-to-exit reachability、Physics overlap、spawn safety、Navigation query、invalid output／timeout／unsupported Target fallback、Navigation Artifact削除後の再生成。
- Compact 2D／3D Levelのframe／memory／load／activation hitch、Cell／prefetch比較、worst-case Portal traversal／camera speed、HLOD on／off authority equivalence、cold start／cold streaming。測定法と共有上限はRuntime ownerを使う。
- AI corpusはMapの6分類、ambiguity／high-impact質問、Scene／Level／Cell／Navigation／Presentation分離、context外の表示名／pathからStable IDを推測しないこと、Source intent以外への直接write拒否を含む。
- `CreateCell`／`UpdateCell`／`DeleteCell`／`SetCellIntent`／`replace_streaming_plan`／未知aliasはすべて`MIRAKAN-WORLD-DERIVED_CELL_WRITE`となり、`SpatialPartitionIntentV1`や`WorldStreamingPlanV1`のhashが変化しないnegative fixture。
- `world_authoring_cross_view_v1` 64 scenarioでWorld Outline／Topology Graph／Level Form／Spatial View／AIのDomain Operationとafter-state hashが一致する。
- `world_authoring_intent_v1` holdout 240件（明確な6分類各30件、曖昧／High Impact 60件）を3 runし、明確Caseの`selected_kind`正解率97%以上、Blocking Caseの`question_required` recall 100%、存在しないStableIdを含むProposal 0件、Scene／Level／Cell identity誤変更0件、Derived／Runtime直接write提案0件とする。

共通capacity test、Evidence envelope、Eval、provenanceは[Runtime performance／capacity](../04-runtime/performance-capacity.md)と[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を使う。万能Map asset、path identity、Runtime pointer保存、silent missing-ref repair、phase／budget／Domain schema複写が残る実装はRelease候補にしない。本書はdomain qualification evidenceを出力し、activationと導入順は[Product Plan](../00-product/product-plan.md)が決定する。
