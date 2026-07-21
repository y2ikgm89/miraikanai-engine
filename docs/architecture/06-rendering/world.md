# Miraikanai Engine World／Scene／Level／Cell Contract

- 文書ID: mirakan.arch.rendering-world
- 状態: review
- 正本範囲: World／Scene／Levelのsource identity、Cellのplan-local identity、source composition／partition、streaming-plan authoring、Level transition intent、reference closure、procedural source、Map要求resolution、World固有operation／diagnostic／qualification
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

## 3. 「Map」要求の解決規則

Map要求は少なくともworld authoring、level composition、scene shard、navigation data、presentation／UIの候補へ分類する。Resolverは対象Project revision、requested task、scope、candidate canonical types、question／assumption、affected ownersを返す。

曖昧な「マップを作る」「マップを開く」に対して新しい万能Map assetを生成しない。空間contentならWorld／Scene／Level、pathfindingなら[Navigation](../05-simulation/navigation.md)、画面表示なら将来のUI／Camera Ownerへroutingし、本書ではpresentation schemaを定義しない。

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

## 7. Level transition intent

Level transitionはsource／target Level、trigger intent、loading presentation ref、persistent entity／state policy、required Cell set、precondition、failure／cancel policyをSourceとして定義する。実行phase、writer、async job、timeout、Save checkpointはRuntime Ownerへ委譲する。

transition中に旧／新LevelのEntity identityを再利用せず、persistent identityは明示されたownerとhandoff recordを持つ。target dependencyが不足する場合は部分Activationやdefault Levelへ黙って進まず、blocking reasonと登録済みfallbackを返す。

## 8. 参照と依存closure

全World referenceはStable ID、expected document kind、required／optional、version compatibilityを持つ。CookerはScene nesting、Entity parent、Level composition、Cell membership、Domain component asset、transition targetのclosureをcanonical orderで解決する。

required ref欠損、kind mismatch、cycle、duplicate owner、stale revisionはhard errorとする。optional ref欠損はSourceでfallbackが宣言された場合だけ許可する。Runtimeはdisplay nameやpathからrefを再解決しない。

Asset artifact、Navigation artifact、LOD／HLOD representation、Renderer material／geometryはgeneration付きrefでPlanへ入り、異なるsource／artifact generationを一つのCell activationへ混在させない。

## 9. Procedural World source

Procedural Worldはgenerator Stable ID、typed parameter、seed semantics、input asset refs、bounded output scope、determinism class、generated Source ownership、regeneration／migration policyを持つ。GeneratorはProject Source DocumentへのChangeSetを生成し、Runtime objectやnative resourceを直接生成して正本化しない。

同じgenerator version、input revisions、seed、parameterから同じStable ID assignmentとcanonical outputを生成する。random device、wall clock、worker completion順、network responseをdeterministic generatorの入力にしない。

生成結果は通常のScene／Entity／Cell validationとreviewを通り、手編集領域を無断上書きしない。外部Tool／generator versionとartifact hashは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)を参照する。

## 10. Navigation、Simulation、Renderingとの境界

WorldはNavigation build volume／modifier、Physics／Collision component、Animation component、Render／Material／Light componentへのtyped document refを保持できるが、各Domain fieldや挙動を定義しない。Domain artifactのcook／validation resultをCell dependency closureへ取り込む。

Navigation queryやWorld movement、Physics body activation、Animation sampling、Render visibilityは各OwnerとRuntime schedulingが所有する。World Cell stateをそれらのauthoritative simulation stateとして兼用しない。

[LOD](lod.md)はresident candidateからrepresentationを選び、[Render Graph](render-graph.md)はactive Cell由来の`WorldRenderPacket`を実行する。Worldはselection formula、visibility algorithm、render passを所有しない。

## 11. Authoring bundleとAI／Editor UX

World authoring bundleは対象World／Level／Scene revisions、selected scope、typed Domain document refs、streaming-plan preview ref、validation summaryを束ねる。共通bundle／projection／operation envelopeは[Executable contracts](../02-foundation/executable-contracts.md)の定義を再利用する。

World operationはcreate／update World、Scene、Level、Cell intent、compose Scene、move Entity source、edit Layer、generate partition plan、create transition、preview closure、explain Map resolution、validateをDomain actionとして登録する。Applyは[Project state](../03-authoring/project-state.md)のChangeSetを通じ、Runtime cellを直接操作しない。

Previewは対象revision、composition graph、Cell membership／dependency、Target plan、missing closure、estimated capacity class、fallback、diagnosticを示す。authorization、approval、sandboxは[AI Security／Approval](../01-governance/ai-security-approval.md)だけが決定する。

## 12. Save、Replay、Migration境界

World Source revisionとRuntime instance stateを分離する。SaveはWorld／Level／Scene／Entity Stable IDとcompatible source／artifact generationへDomain stateを投影し、Source Document、Runtime pointer、Cell handleを丸ごと保存しない。

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
| `MIRAKAN-WORLD-DEPENDENCY_NOT_RESIDENT` | hard dependency不足 | Cellをactiveにしない |
| `MIRAKAN-WORLD-ACTIVATION_PARTIAL` | activation groupの一部だけ成功 | 全体rollback |
| `MIRAKAN-WORLD-BUDGET_EXCEEDED` | residency／IO／hitch上限超過 | fallbackまたはtransition中止 |
| `MIRAKAN-WORLD-PROCEDURAL_NONDETERMINISTIC` | 同じ入力でoutput hash不一致 | Artifact拒否 |
| `MIRAKAN-WORLD-PROCEDURAL_INVALID_OUTPUT` | Schema／connectivity／playability不合格 | delta破棄 |
| `MIRAKAN-WORLD-PRESENTATION_AUTHORITY_WRITE` | Map／LOD／visibilityからGameplay write | Build／conformance失敗 |
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
- inactive／resident／active境界からauthoritative処理が漏れず、Presentation／LOD／Camera／GPU結果からauthorityへ逆入力しないこと。
- 同じseed／input／Target／Toolchainから同じprocedural output hash、Generator bound、connectivity、entry-to-objective-to-exit reachability、Physics overlap、spawn safety、Navigation query、invalid output／timeout／unsupported Target fallback、Navigation Artifact削除後の再生成。
- Compact 2D／3D Levelのframe／memory／load／activation hitch、Cell／prefetch比較、worst-case Portal traversal／camera speed、HLOD on／off authority equivalence、cold start／cold streaming。測定法と共有上限はRuntime ownerを使う。
- AI corpusはMapの6分類、ambiguity／high-impact質問、Scene／Level／Cell／Navigation／Presentation分離、context外の表示名／pathからStable IDを推測しないこと、Source intent以外への直接write拒否を含む。
- `world_authoring_cross_view_v1` 64 scenarioでWorld Outline／Topology Graph／Level Form／Spatial View／AIのDomain Operationとafter-state hashが一致する。
- `world_authoring_intent_v1` holdout 240件（明確な6分類各30件、曖昧／High Impact 60件）を3 runし、明確Caseの`selected_kind`正解率97%以上、Blocking Caseの`question_required` recall 100%、存在しないStableIdを含むProposal 0件、Scene／Level／Cell identity誤変更0件、Derived／Runtime直接write提案0件とする。

共通capacity test、Evidence envelope、Eval、provenanceは[Runtime performance／capacity](../04-runtime/performance-capacity.md)と[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を使う。万能Map asset、path identity、Runtime pointer保存、silent missing-ref repair、phase／budget／Domain schema複写が残る実装はRelease候補にしない。Capability maturityと導入順は[Product Plan](../00-product/product-plan.md)だけが所有する。
