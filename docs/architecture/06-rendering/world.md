# Miraikanai Engine World／Scene／Space／Cell Contract

- 文書ID: mirakan.arch.rendering-world
- 状態: review
- 正本範囲: World／Scene／Spaceのsource identity、global composition、persistent entity、optional spatial topology、Cellのplan-local identity、partition／streaming-plan authoring、spatial transition／Loading presentation、Tilemap、Engine-native Blockout、procedural source、Map要求resolution
- 非正本範囲: Stage／Objective／Completion／Encounter／Gameplay progression、Runtime cell phase／shared capacity、ECS／Gameplay component schema、Physics／Navigation behavior、Render／LOD execution、Asset transaction、Save／Replay envelope、AI authorizationは各Ownerを参照
- 依存: [Product Plan](../00-product/product-plan.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[Collision](../05-simulation/collision.md)、[Physics](../05-simulation/physics.md)、[Navigation](../05-simulation/navigation.md)、[Animation](../05-simulation/animation.md)、[Render Graph](render-graph.md)、[LOD](lod.md)、optional consumer: [Scenario／Stage Feature Pack](../08-packs/scenario-stage.md)
- 外部根拠検証日: 2026-07-23

## 1. 結論と所有境界

Worldは空間、Scene、global composition、persistent entity、任意のspatial topologyだけを所有する。有限Stage、entry／exit、Gameplay goal、outcome、Spawn、Stage transitionは[Scenario／Stage Feature Pack](../08-packs/scenario-stage.md)だけが所有し、Worldは同Featureを必須にしない。

World activation、Scene activation、Cell streamingはGameplay goalやResultを要求しない。Scene 0件のprocedural-only World、spatial topologyなしのUI補助World、Scenario／Stage Packなしのendless simulationをvalidとする。

AI、Editor、Project C++はSource Documentとtyped operationを扱い、Runtime cell object、ECS pointer、Renderer resource、Physics／Navigation native handleを直接保存しない。cell activation state／phase／lifetimeはRuntime、capacity／backpressureはPerformance、representation selectionはLODが所有する。

## 2. 正規用語とidentity

- `World`: Project内の空間／content universeとglobal composition root。
- `Scene`: authoring、ownership、collaborationのための再利用可能なsource shard。
- `Space`: spatial topology上の場所または区画。Gameplay progression単位ではない。
- `Cell`: streaming、residency、activation planの空間／logical partition unit。
- `Entity source`: Stable IDを持つWorld content record。Runtime entity instanceではない。
- `Layer`: visibility／authoring／variant selectionのorthogonal grouping。
- `Map`: user request語でありcanonical object typeではない。resolverがWorld／Scene／Space／navigation／presentation／optional Scenarioへ分類する。

World／Scene／Space／Entity source identityはProject Stable ID、source revision、display labelを分離する。path、filename、display name、array indexをidentityにしない。CellはSource Stable IDを持たずPlan-local `uint32`であり、Scene、Space、Cell、Runtime chunkを同一視しない。

## 3. 「Map」要求の解決規則

`MapIntentKindV1`は次の6 kindへ閉じる。

| Kind | 例 | 変更対象 |
|---|---|---|
| `world_structure` | 地域接続、町からDungeonへ移動 | `SpatialTopologyDefinitionV1` |
| `scenario_stage` | finite stage、boss room、outcome | [Scenario／Stage Feature Pack](../08-packs/scenario-stage.md) |
| `streaming` | seamless load、遠方軽量化 | `SpatialPartitionIntentV1`＋Derived Plan |
| `procedural_layout` | seed生成Dungeon | `ProceduralWorldDefinitionV1` |
| `navigation` | 歩行領域、飛行経路、NavMesh | [Navigation](../05-simulation/navigation.md) |
| `map_presentation` | minimap、world map、marker、fog | `MapPresentationDefinitionV1` |

`MapIntentResolutionV1`は`request_id`、`candidate_kinds[]`、`selected_kind`、`confidence_q16`、`evidence_requirement_ids[]`、`affected_stable_ids[]`、`blocking_questions[]`、`disposition: resolved | question_required | rejected`を持つ。上位2候補差が9,830未満、Save／spatial transition／authoritative state／Target／Asset license／memory capacityへ影響、layoutとpresentationの両解釈、Stable IDを特定不能のいずれかなら`question_required`とする。

曖昧な「マップを作る」「マップを開く」に万能Map assetを生成しない。空間contentはWorld／Scene／Space、finite gameplayはScenario／Stage、pathfindingはNavigation、画面表示はMap Presentationへroutingする。

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

Entity sourceはStable ID、Transform source、parent ref、Feature component document refs、Layer／tag、authoring metadata refを持つ。Gameplay／Physics／Navigation／Rendering fieldは各Ownerを参照し、World共通recordへflattenしない。

## 5. Spatial topology

```text
SpatialTopologyDefinitionV1
  space_nodes[]
  transition_edges[]
  activation_entry_refs[]
```

`space_nodes[]`はStable Space ID、optional parent、spatial boundsまたはtopological location、`intentionally_isolated`を持つ。parent graphはDAGとする。`activation_entry_refs[]`は0件を許可し、default entryや到達可能なGameplay goalを要求しない。

`transition_edges[]`はsource／target Space、anchor refs、direction、condition policy ref、activation hint、fallback、typed `transfer_subject_refs[]` schema refを持つ。`transfer_subject_refs[]`はCharacter、Player、Partyへ固定せず、Entity、camera observer、simulation agent、board token等のregistered subject contractを参照できる。bidirectional edgeはCook時に二つの正規有向edgeへ展開する。

entryから到達不能なSpaceは`intentionally_isolated=true`を必須にする。missing ref、duplicate edge identity、parent cycle、孤立宣言なしの到達不能Space、fallbackなしのTarget非対応edgeを`MIRAKAN-WORLD-TOPOLOGY_INVALID`で拒否する。Topology activationはScene／Cell dependency closureだけを解決し、Scenario outcomeを評価しない。


## 6. Spatial Partitionとstreaming-plan authoring

`SpatialPartitionIntentV1`はCreatorが編集するSourceであり、`partition_intent_id`、厳密に1件の`world_ref`、`spatial_dimension: 2d | 3d`、`interest_source_kinds`（`player | camera | portal | scripted_anchor | network_authority`のsubset）、physical unitとhysteresisを明示する`activation_radius_policy`、together／separate Stable ID set、1～128件のtyped ordered `priority_rules`、0～4,096件の`always_resident_refs`、Stable Entity／Layer／Sceneの`streamable_refs`、Targetごとに厳密に1件の`target_budget_refs`、typed `failure_policy`を持つ。Cell size、chunk file名、GPU heap offset、Backend page IDをSource Intentへ固定しない。

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

`SpatialTransitionRequestV1`は`request_id`、source Space instance ref、transition edge ref、target Space ref、target anchor ref、requesting system ref、requested tick、typed `transfer_subject_refs[]`、precondition snapshot hash、transition policy refを持つ。表示名から対象を再解決せず、stale condition、inactive source、target／anchor mismatchをprefetch前に拒否する。

transition中に旧／新SpaceのEntity identityを再利用しない。persistent identityは明示ownerとhandoff recordを持つ。target dependency不足時はpartial activationやdefault destinationへ進まず、blocking reasonとfallbackを返す。

### 7.1 Loading／prefetch presentation

`SpatialTransitionPresentationPolicyV1`は`mode: seamless | overlay | blocking`、Loading UI、Input context、Audio snapshot、0～2,000 msのminimum display、cancel boundary、failure presentation、explicit retry、Accessibility announcement policyを持つ。

`LoadingProgressPlanV1`は確定したdependency closureを1～65,535 work unitへcanonicalizeし、positive weight合計を65,535へnormalizeする。`LoadingProgressSnapshotV1`はgeneration、phase、completed weight、current message、cancel／retry可否、typed failureを持つ。同じgenerationの進捗は実完了unitだけで単調非減少とし、fake timer、frame count、spinnerを混ぜない。65,536 unit以上は`loading_plan_capacity_exceeded`でPlan materialization前に拒否し、truncateしない。

I/O／verifyはreal-time domainで進められるが、activationとtyped subject transferは正規tick boundaryでatomic commitする。target closure失敗時は全rollbackしてsource Spaceをactiveのまま維持する。Cancel／Retryはpolicyとphaseを検証し、新request／session identityでSource revision、condition、Target Capability、storage／memory budgetを再検証する。Loading UI、Audio、読み上げはauthorityへ逆入力しない。


## 8. 参照と依存closure

全World referenceはStable ID、expected document kind、required／optional、version compatibilityを持つ。CookerはScene nesting、Entity parent、Space topology、Cell membership、Domain component asset、transition targetのclosureをcanonical orderで解決する。

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

`MapPresentationDefinitionV1`は`map_presentation_id: StableId`、`presentation_kind: minimap | world_map | space_map | navigation_overlay`、exact `spatial_subject_ref`、`projection_policy: orthographic_2d | authored_2d | projected_3d`、`layer_refs[1..128]`、`marker_style_refs[0..512]`、typed `marker_source_contract_refs`、optional `fog_policy_ref`、`accessibility_profile_refs[1..32]`、exact `localization_namespace_ref`、Targetごとのexact `render_budget_refs`、`fallback_contract`を持つ非authoritative Sourceである。Marker／fog／cursorからWorld／Feature State／Navigation costを直接writeせず、入力はtyped `MapInteractionCommandV1`としてNavigation／Quest ownerへ送る。

### 10.1 Tilemap source、cook、publication

Tile chunkはTilemap内部のedit、cook、culling単位であり、World Cell、Region、Space、activation groupではない。Tilemapはresident／active CellのAsset closureへ参加するが、chunk単独でEntity、Feature State、transition、authoritative gameplayをactivateしない。次の12型をWorldの唯一の正本とする。

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

各dimensionは0.01～10,000 m、radial segmentsは3～64、height segmentsは1～64、compact spatial assembly全体は4,096 primitive以下とする。assembly／local transformはfinite translation、normalized rotation、正のscaleで、負scale、NaN／Inf、zero-area surface、暗黙boolean由来のnonmanifold、Colliderとwalkable semanticの矛盾を拒否する。Primitiveは通常のTransform、Material、Renderer、Collision、Navigation SourceへCookし、Blockout専用Runtime objectを作らない。

`CreatePrimitiveMesh | CreateBlockoutAssembly | UpdateBlockoutPrimitive | PromoteBlockoutToMeshAsset`をAI／Editor共通のbounded operationとする。Promotion previewは元Stable ID、generation、pivot、bounds、Material slot、Collider／Navigation semantic、参照元を固定し、承認後に通常Mesh Sourceと対応表をatomic publishする。Renderer／Collision／Navigation Artifactのall-readyとgeneration一致前にはassembly置換を公開しない。元Sourceを自動削除せず、置換対象はexplicit ChangeSetに列挙する。C1 fixtureはexternal DCCを要求せず、6 primitive、dimension境界、4,096 primitive spatial assembly、Collider／Navigation cook、Promotion前後のbounds／pivot／Material slot、Undo／Redo、Save／Load、AI／手動operationのafter hash一致を検証する。external DCC Asset 0件のarenaのWorld activation smokeを完走できることをGateとする。

## 11. Authoring bundleとAI／Editor UX

World authoring bundleは対象World／Scene／Space revision、selected scope、typed Feature document refs、streaming-plan preview、validation summaryを束ねる。`WorldAuthoringPlanV1`はWorld／Scene／Space refs、partition／procedural／presentation change、required Capability、budget、fixture、assumption、blocking question、fallback、risk、dispositionを持つがCommit権限を持たない。

Source operationは`CreateWorldDocument`、`CreateSceneDocument`、`ComposeScene`、`MoveEntityToScene`、`SetSpatialTopologyDefinition`、`SetSpatialPartitionIntent`、`SetProceduralWorldDefinition`、`SetMapPresentationDefinition`である。Stage operationをWorld namespaceへ登録しない。`GenerateWorldStreamingPlan`はCommit済みSourceからDerived Planを作るCook jobで、Preview／Inspectはread-onlyである。

AI contextはWorld／Scene／Space、Topology、Cell plan、dependency、Target、budgetだけをbounded projectionし、Scenario／Stage Featureが選択された場合は同Ownerへのlinkだけを返す。

## 12. Save、Replay、Migration境界

World Source revisionとRuntime instance stateを分離する。SaveはWorld／Scene／Space／persistent Entity Stable IDとcompatible source／artifact generationだけを投影し、Source Document、Runtime pointer、Cell handleを保存しない。

Save／Replay fieldはregistered State ownerとSave／Replay契約が列挙したものだけを含む。WorldはStage instance、Gameplay goal、outcome、Encounter、Game Flowを保存しない。checkpoint、recording、Replay envelope、migration evidenceは[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)を参照する。

## 13. Diagnostic、failure、qualification

World diagnosticはWorld／Scene／Space／Entity Stable ID、Plan ID／plan-local Cell ID、source revision、reference path、Target、error code、remediationを含む。少なくともmissing ref、kind mismatch、topology cycle、duplicate owner、invalid partition、Cell dependency cycle、unbounded closure、stale plan、unsupported Target、migration unresolvedを区別する。

| Diagnostic ID | 条件 | 結果 |
|---|---|---|
| `MIRAKAN-WORLD-UNKNOWN_STABLE_ID` | World／Scene／Space／Anchor不明 | fuzzy適用せず拒否 |
| `MIRAKAN-WORLD-TOPOLOGY_INVALID` | cycle、unknown edge、未宣言isolated Space | Sourceを拒否 |
| `MIRAKAN-WORLD-PARTITION_INVALID` | Cell bound、dependency、activation group不正 | Planをpublishしない |
| `MIRAKAN-WORLD-LOADING_PLAN_CAPACITY_EXCEEDED` | 65,536 unit以上 | previous valid Planを維持 |
| `MIRAKAN-WORLD-ACTIVATION_FAILED` | dependencyまたはsubject transfer失敗 | target rollback、source維持 |

Qualificationは次を含む。

- World／Scene／Space／Cell identity、0 Scene procedural-only、0 entry topology、intentional isolation。
- Cell state transition、cancel、timeout、I/O failure、activation group atomicity、source Space維持、typed subject transfer、Save／Replay state hash。
- Tilemapのgrid／D4 transform／capacity／atomic publicationとAI／manual after hash一致。
- Blockoutのdimension／segment／assembly bound、semantic矛盾、通常Domain cook、Promotion all-ready、external DCC 0件。
- Compact 2D／3D spatial fixtureのframe／memory／load／activation hitch、Cell／prefetch比較、camera speed、HLOD authority equivalence、cold start／streaming。
- Scenario／Stage Packなしのendless、continuous simulation、procedural-only Worldがvalidで、Gameplay goalやResultを要求しない。
