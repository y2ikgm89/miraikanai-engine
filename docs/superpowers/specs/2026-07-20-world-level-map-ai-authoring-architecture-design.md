# Miraikanai Engine World／Level／Map／AI Authoringアーキテクチャ規約

- 文書版: 1.1
- 作成日: 2026-07-20
- 調査基準日: 2026-07-20
- 対象: World、Scene、Level、Topology、Streaming、Procedural Generation、Navigation接続、Map Presentation、AI Authoring
- 状態: プロジェクト公式の規範設計レビュー版。実装待ち
- 上位文書: [AIネイティブ独自ゲームエンジン 設計計画書](./2026-07-18-ai-native-game-engine-authoring-design.md)
- Game System: [Miraikanai Engine Game System／AI Code Generationアーキテクチャ規約](./2026-07-20-game-system-ai-codegen-architecture-design.md)
- Authoring状態: [Miraikanai Engine Authoring Model／Project State規約](./2026-07-19-authoring-model-project-state-design.md)
- Runtime: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- Math／座標: [Miraikanai Engine AI可読Math／Core Utilitiesアーキテクチャ規約](./2026-07-20-ai-readable-math-core-utilities-architecture-design.md)
- 2D／3D能力計画: [Miraikanai Engine 2D／3D Capability導入計画](./2026-07-19-2d-3d-capability-plan.md)
- Asset: [Miraikanai Engine Asset Pipeline／Content Packaging規約](./2026-07-19-asset-pipeline-content-packaging-design.md)
- Navigation: [Miraikanai Engine Navigation Platformアーキテクチャ規約](./2026-07-20-navigation-platform-architecture-design.md)
- Physics／Animation連携: [Miraikanai Engine Physics／Navigation／Animationアーキテクチャ規約](./2026-07-19-physics-navigation-animation-architecture-design.md)
- LOD: [Miraikanai Engine AI可読LODアーキテクチャ規約](./2026-07-20-ai-readable-lod-architecture-design.md)
- UI: [Miraikanai Engine UI／Text／Localization／Accessibility規約](./2026-07-19-ui-text-localization-accessibility-design.md)
- Camera: [Miraikanai Engine Camera／Platform／AI Authoring／Virtual Productionアーキテクチャ規約](./2026-07-20-camera-platform-ai-authoring-virtual-production-architecture-design.md)
- 検証: [Miraikanai Engine AI検証・評価・来歴規約](./2026-07-19-ai-verification-evaluation-provenance-design.md)
- Debugging／Replay: [Miraikanai Engine AI可読Debugging／Observability／Replayアーキテクチャ規約](./2026-07-20-ai-readable-debugging-observability-replay-architecture-design.md)

## 1. 結論

Miraikanai Engineの公式World／Level方式を、**Source IntentとTarget別Derived Planを分離した、契約駆動の分割Authoring／Streaming architecture**へ固定する。

`Map`を一つのSystem、Document、巨大File、C++ classの名前として扱わない。自然言語の「マップ」は必ず次のいずれかへ解決する。

```text
world_structure
playable_level
streaming
procedural_layout
navigation
map_presentation
```

正本と責務を次のように分ける。

```text
WorldDocument
  -> authoring aggregate、UUIDv7 StableId、Entity／Component、Document分割

SceneDocument
  -> collaborative edit shard。Gameplay上のLevelではない

WorldTopologyDefinitionV1
  -> Region、Level、Portal、接続条件の論理Graph

LevelDefinitionV1
  -> play可能単位のentry、exit、objective、spawn、encounter、rule

SpatialPartitionIntentV1
  -> 何を同時にresidentにしたいかというSource Intent

WorldStreamingPlanV1
  -> Target ProfileごとにCookerが生成するread-only Derived Artifact

ProceduralWorldDefinitionV1
  -> seed、generator version、constraint、inputによる決定論的Source recipe

NavigationDefinition／Profile
  -> 移動可能性のSource Intent。Navigation ArtifactはDerived

MapPresentationDefinitionV1
  -> minimap、world map、marker、fog表示。Gameplay authorityを持たない
```

初期製品C1の標準は、明示境界を持つcompact 2D／3D Levelである。World Partition、HLOD、大規模連続World、runtime procedural generationは設計上のescape pathを先に確保するが、個別Qualificationが完了するまでC2／Phase 8以降とする。大規模World機能をPhase 0の空実装、巨大Schema、固定gridとして先行実装しない。

AIはLevelの見た目だけを生成しない。Requirement、Topology、Level rule、Spatial intent、Entity／Asset変更、Navigation要求、Map presentation、Budget、Testを一つの`WorldAuthoringBundleV1`として提案する。Source Documentを直接上書きせず、Staging、Validation、Preview、Review、ProjectChangeSet Commitを経由する。

## 2. 決定権と境界

| 主題 | 正本 |
|---|---|
| World／Scene／Levelの意味、Topology、Partition Intent、Streaming Plan、Procedural World、Map Presentation | 本書 |
| Document、ProjectChangeSet、Revision、Undo、Recovery、Collaborative edit | Authoring Model規約 |
| Game System、Level gameplay、State owner、System Bundle | Game System規約 |
| Runtime World、lease、phase、StructuralCommand、queue、budget | Runtime規約 |
| Navigation Source／Artifact、Query、Agent Profile、Platform | Navigation Platform規約 |
| Asset Source、Cook、Package、Stable Asset ID、dependency closure | Asset Pipeline規約 |
| Render visibility、LOD／HLOD、Camera、UI | 各Subsystem正式仕様 |
| AI権限、Risk、Activation、Receipt | Governance／Verification規約 |

World／LevelはPhysics、Navigation、Combat、Quest、Camera、UIのStateを所有しない。LevelはそれらのSystemをexact `GameSystemContractRefV1`で構成し、typed Command／Event／Snapshotで接続する。

`WorldDocument`はEditorとCookerが読むAuthoring aggregateであり、Shipping Runtime objectではない。`RuntimeWorld`はCook済みArtifactからRuntime Orchestratorが生成する実行状態であり、Document pointerまたはEditor objectを保持しない。

## 3. 正規用語

| 用語 | 定義 |
|---|---|
| World | 複数Level、Region、Portal、global placementをまとめる論理空間 |
| Scene | Authoring／collaboration／差分管理のDocument shard |
| Level | entry、objective、completion、exitを持つplay可能なGame単位 |
| Region | Topology上のLevel grouping。Streaming cellではない |
| Portal | LevelまたはRegion間の有向接続と遷移条件 |
| Spatial Partition | Runtime residencyを計画する空間分割Intent |
| Cell | Cook後Streaming Planの最小residency単位 |
| Residency | CellのCPU／GPU／Asset／Physics／Navigation資源が利用可能な状態 |
| Activation | Resident contentをauthoritative gameplayへ参加させる境界 |
| Procedural World | version付きgeneratorとseedから再現できるSource／staged生成結果 |
| Navigation | Agentごとの移動可能性、Query、経路Artifact |
| Map Presentation | minimap、world map、marker、fog等の非authoritative表示 |
| World Source Intent | Creatorが意味として編集するDocument／Definition |
| World Derived Plan | Target、Budget、Source IntentからCookerが生成するArtifact |

`Scene`、`Level`、`Cell`を相互の別名として扱わない。一つのSceneに複数LevelのSourceを置くことも、一つのLevelを複数Sceneへ分割することも許可するため、identityとlifecycleを別にする。

### 3.1 IDと参照の型

World関連のIDを次へ固定する。

| 型 | 表現 | 用途 |
|---|---|---|
| Source `StableId` | 基盤規約のRFC 9562 UUIDv7、128 bit | World、Scene、Topology、Region、Level、Portal、Anchor、Partition Intent、Procedural Definition、Map Presentation、Authoring Plan／Bundle |
| `McdContractRefV1` | `{id, version, contract_set_hash}` | Game System、Capability、Type、Operation等のMCD契約 |
| `ArtifactRefV1` | `{artifact_kind, schema_version, sha256}` | Streaming、Navigation、HLOD、Cooked payload等のDerived Artifact |
| Plan-local ID | `uint32`、0 invalid | 一つの`WorldStreamingPlanV1`内のCell／Activation Group。Plan外では`ArtifactRefV1`と組み合わせる |
| Runtime handle | index 32 bit＋generation 32 bit、0 invalid | Level instance、Entity、Cell runtime instance。Save／Source／別sessionへ保存しない |

表示名、相対path、配列index、dotted aliasをidentityに使用しない。Source `StableId`はAuthoringCommandGatewayだけが発行し、rename、Scene移動、Level移動、再Cookで変更または再利用しない。MCDのdotted Contract IDをUUIDv7 `StableId`として扱わず、逆にSource UUIDをMCD IDとして登録しない。

## 4. 「Map」要求の解決規則

### 4.1 `MapIntentKindV1`

| 値 | 例 | 変更対象 |
|---|---|---|
| `world_structure` | 地域をつなぐ、町からDungeonへ移動する | `WorldTopologyDefinitionV1` |
| `playable_level` | Stageを作る、boss roomを追加する | `LevelDefinitionV1`＋World Source |
| `streaming` | seamlessに読み込む、遠方を軽くする | `SpatialPartitionIntentV1`＋Derived Plan |
| `procedural_layout` | seedでDungeonを生成する | `ProceduralWorldDefinitionV1` |
| `navigation` | 歩ける領域、飛行経路、NavMesh | Navigation Definition／Profile |
| `map_presentation` | minimap、world map、marker、fog | `MapPresentationDefinitionV1` |

### 4.2 Resolver

AI Gatewayは自然言語を`MapIntentResolutionV1`へ正規化する。

```text
MapIntentResolutionV1
  request_id
  candidate_kinds[]
  selected_kind
  confidence_q16
  evidence_requirement_ids[]
  affected_stable_ids[]
  blocking_questions[]
  disposition
```

| Field | 規則 |
|---|---|
| `request_id` | UUIDv7 `StableId`。Intent Resolution、Plan、Bundle、Provenanceを結ぶ |
| `candidate_kinds` | score降順の`{kind, confidence_q16, evidence_requirement_ids}`、1～6件、kind重複不可 |
| `selected_kind` | `resolved`時は厳密に1件、その他0件 |
| `confidence_q16` | 0～65,535。1.0を65,535とする |
| `evidence_requirement_ids` | 1～64件 |
| `affected_stable_ids` | 実在確認済みID、0～1,024件 |
| `blocking_questions` | 0～7件。各Questionは選択肢2～5件、推奨、影響、変更可能性を持つ |
| `disposition` | `resolved \| question_required \| rejected` |

次のいずれかなら`question_required`にする。

- 上位2候補の`confidence_q16`差が9,830未満。
- Save互換、Level遷移、authoritative State、Target対応、Asset license、memory budgetへ影響する。
- 「マップを生成」のようにlayoutと表示地図の両方へ解釈できる。
- 新Level作成か既存Level変更かをUUIDv7 `StableId`で特定できない。

Blockingでないlayout detail、装飾密度、既定themeはProject ProfileとGameSpecの明示defaultを使える。AIが`MapManager`、`MapSystem`、`MapData`という曖昧な新規型へ逃がしてはならない。

## 5. Source Document model

### 5.1 `WorldDocument`

`WorldDocument`は既存Authoring Model規約のDocument envelopeを使い、次のroot referenceだけを正本として持つ。

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
| `budget_profile_ref` | 厳密に1件 |

WorldDocumentへAsset binary、Navigation mesh、HLOD mesh、Streaming cell payload、C++ source、Vendor objectを埋め込まない。上限を超える場合はWorldまたはRegionを分割し、数値上限だけを拡大しない。

### 5.2 `SceneDocument`

SceneDocumentの目的は編集競合、ownership、load範囲、diffを制御することである。必須Fieldを次へ固定する。

| Field | 型／規則 |
|---|---|
| `scene_id` | UUIDv7 `StableId` |
| `document_bounds` | 2D AABBまたは3D AABB、local space |
| `entity_refs` | Stable Entity ID、0～1,048,576件 |
| `level_membership_refs` | 0～256件 |
| `layer_refs` | 0～256件 |
| `source_dependency_refs` | exact versionまたはcontent hash |
| `edit_ownership` | Authoring lock／merge policy |

Scene境界はRuntime Streaming境界を強制しない。CookerはSource IntentとTarget Budgetから複数Sceneを一Cellへまとめることも、一Sceneを複数Cellへ分割することもできる。

## 6. `WorldTopologyDefinitionV1`

```text
WorldTopologyDefinitionV1
  topology_id
  version
  world_ref
  region_nodes[]
  level_nodes[]
  portal_edges[]
  global_entry_refs[]
  invariant_ids[]
```

`topology_id`、Region、PortalはUUIDv7 `StableId`、`world_ref`とLevel／Anchor参照はUUIDv7 `StableId`＋Document revision、`version`は1から単調増加する`uint32`である。Topologyの意味を変更するCommitごとにversionを増加し、canonical Source bytesのSHA-256をDocumentRefへ記録する。

### 6.1 Node

| Field | 規則 |
|---|---|
| `region_nodes` | 0～4,096件。Stable Region ID、parent 0または1件 |
| `level_nodes` | 1～16,384件。Stable Level ID、Region 0または1件 |
| `global_entry_refs` | 1～256件。少なくとも一つがDefault |

Region parent graphはDAGとする。Levelを複数Regionへ同時所属させない。分類を重ねたい場合はTagまたはData Layerを使い、lifecycle ownershipを複数化しない。

### 6.2 `PortalEdgeV1`

```text
PortalEdgeV1
  portal_id
  from_level_ref
  from_anchor_ref
  to_level_ref
  to_anchor_ref
  direction
  condition_definition_ref
  transition_policy_ref
  preload_hint
  fallback
```

`direction`は`one_way | bidirectional`である。BidirectionalはCook時に二つの正規有向edgeへ展開する。Portal conditionはbounded GameplayDefinitionで評価し、C++ function名またはEditor callbackを保存しない。

Topology compilerは次を拒否する。

- 存在しないLevel／Anchor／Definition参照。
- Default entryから到達不能で、`intentionally_isolated=true`もないLevel。
- return path必須のLevelからの一方向trap。
- Portal ID重複、Region cycle、同じLevelの複数Region ownership。
- Target非対応Levelへのfallbackなしの必須edge。

## 7. `LevelDefinitionV1`

LevelDefinitionはGameplay上のplay可能単位であり、geometry containerではない。

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
| `behavior_budget_refs` | Target Profileごとに厳密に1件 |
| `completion_contract` | success／failure／abortのtyped outcome |
| `fallback_contract` | Target非対応時の意味同等fallbackまたは理由 |

`level_game_system_refs`のうち厳密に一つが`level_gameplay` roleのauthoritative ownerである。LevelDefinition自身はObjective進捗、Combat State、Quest State、Character Stateを書かない。

### 7.1 `LevelRuntimeStateV1`

Level gameplay Systemが次を所有する。

```text
LevelRuntimeStateV1
  level_ref
  runtime_instance_id
  lifecycle_state
  active_entry_ref
  objective_state_refs[]
  activated_system_instance_refs[]
  authoritative_world_delta_ref
  completion_outcome
```

`runtime_instance_id`はgeneration付き`LevelInstanceHandle`であり、Source、Save、Replay headerへ永続化しない。Saveは`level_ref`のUUIDv7 `StableId`、Level version、State fieldだけを記録する。

`lifecycle_state`は次のclosed state machineである。

```text
inactive
  -> preparing
  -> ready
  -> activating
  -> active
  -> completing
  -> deactivating
  -> inactive

preparing | activating | active | completing | deactivating
  -> faulted
```

`faulted`から自動で`active`へ戻さない。Transition policyに従い、直前Level維持、checkpoint復帰、session abortのいずれかを明示する。

## 8. Spatial PartitionとStreaming

### 8.1 `SpatialPartitionIntentV1`

Creatorが編集するのは結果CellではなくIntentである。

| Field | 型／規則 |
|---|---|
| `partition_intent_id` | UUIDv7 `StableId` |
| `world_ref` | 厳密に1件 |
| `spatial_dimension` | `2d | 3d` |
| `interest_source_kinds` | `player | camera | portal | scripted_anchor | network_authority`のsubset |
| `activation_radius_policy` | physical unitとhysteresisを明示 |
| `grouping_constraints` | together／separate UUIDv7 `StableId` set |
| `priority_rules` | typed ordered rules、1～128件 |
| `always_resident_refs` | 0～4,096件 |
| `streamable_refs` | Stable Entity／Layer／Scene参照 |
| `target_budget_refs` | Targetごとに厳密に1件 |
| `failure_policy` | preload failure時のtyped behavior |

Source IntentへCell size、chunk file名、GPU heap offset、backend page IDを固定しない。必要ならTarget Profile別override policyとして宣言し、共通意味を変えない。

### 8.2 `WorldStreamingPlanV1`

World Streaming PlanはCookerが生成するDerived Artifactであり、EditorまたはAIが直接編集しない。

```text
WorldStreamingPlanV1
  plan_id
  source_world_revision
  contract_set_hash
  target_profile_id
  toolchain_manifest_hash
  partition_intent_hash
  cell_descriptors[]
  dependency_edges[]
  activation_groups[]
  residency_budget
  canonical_priority_order
  fallback_plan_ref
  artifact_hash
```

`plan_id`は`SHA-256("mirakan.world_streaming_plan.v1" || source_world_revision || contract_set_hash || target_profile_id || toolchain_manifest_hash || partition_intent_hash)`で生成する32 byte `DerivedPlanId`である。`artifact_hash`は完成したcanonical Plan bytesのSHA-256であり、入力identityの`plan_id`と出力integrityの`artifact_hash`を置換しない。`fallback_plan_ref`はexact `{plan_id, artifact_hash}`を使う。

`CellDescriptorV1`を次へ固定する。

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

一Planは1～1,048,576 Cell、1 activation groupは1～4,096 Cell、Plan全体の`source_refs`合計は16,777,216件を上限とする。さらにTarget ProfileのArtifact byte budgetを必須とし、件数内でもbyte budget超過は失敗させる。上限超過はPlan generation errorであり、Cellを黙って欠落させない。Cell descriptorは生pointer、Vendor resource handle、absolute pathを持たない。

Cell候補をMath規約のcanonical finite number表現による`bounds.min.x, min.y, min.z, bounds.max.x, max.y, max.z`昇順、次にUUID byte順で作る`source_refs` Merkle root、最後に`payload_hash`のbyte順で並べ、1から`cell_id`を割り当てる。2Dではzを正規化した`+0`とする。同一sort keyが二つあるPlanを重複Cellとして拒否する。Activation Groupは所属`cell_id`昇順列のlexicographic順で1から`activation_group_id`を割り当てる。

同じSource revision、Target Profile、Toolchain Manifestからは同じPlan hashを生成する。非決定要因を使用した場合はCookを失敗させる。

### 8.3 Cell state machine

```text
unloaded
  -> prefetching
  -> resident
  -> activating
  -> active
  -> deactivating
  -> evictable
  -> unloaded

prefetching | resident | activating | active | deactivating | evictable
  -> failed
```

Runtime OrchestratorだけがCell stateを変更する。

- `resident`: payloadと依存が利用可能だがauthoritative gameplayへ未参加。
- `active`: activation group全体がboundaryで成功し、Entity／Physics／Navigation／Systemが参加。
- Activationはall-or-nothingである。途中失敗したCell群を部分activeにしない。
- `active` Cellだけがauthoritative gameplayを実行する。
- `deactivating`後は新Commandを受けず、queue drain、job cancel、lease releaseを完了する。
- `failed`はtyped reasonを返し、fallback planまたは旧Level維持へ進む。

HLOD、occlusion、render visibility、camera frustum、GPU queryはPresentation／Derived stateである。これらをEnemy spawn、Damage、Objective、Navigation authority、Save判定へ逆入力しない。

### 8.4 Budget

Target Profileごとに最低限次を宣言する。

```text
max_resident_cpu_bytes
max_resident_gpu_bytes
max_inflight_io_bytes
max_prefetch_ms_p95
max_activation_ms_p95
max_main_thread_hitch_ms
max_active_entity_count
max_active_physics_body_count
max_active_navigation_tile_count
```

Budget超過時にsilent quality reductionまたはCell欠落を行わない。`budget_exceeded` Diagnostic、fallback plan、Level transition中止のいずれかへ進む。

## 9. Level transition

`LevelTransitionRequestV1`を次へ固定する。

| Field | 規則 |
|---|---|
| `request_id` | World runtime instance内で1から単調増加する`uint64`。0 invalid、Save／別session比較禁止 |
| `source_level_instance_ref` | 現在active Level、厳密に1件 |
| `portal_ref` | Topology内Portal、厳密に1件 |
| `target_level_ref`／`target_entry_anchor_ref` | Portal解決結果と完全一致 |
| `requesting_system_ref` | Game FlowまたはLevel gameplay System |
| `requested_tick` | `uint64` |
| `player_or_party_transfer_refs` | generation付きtyped runtime handle、0～256件 |
| `precondition_snapshot_hash` | Portal condition評価に使ったimmutable Snapshot |
| `transition_policy_ref` | timeout、presentation、failure、checkpointを定めるexact ref |

同じsource instanceとtransfer対象に未完了Requestを複数許可しない。Portalが無効、Source Levelがactiveでない、Target／Anchor不一致、condition hashがstaleの場合はprefetch前に拒否する。

Level transitionの正規順序を固定する。

1. Portal／Game Flow Systemが`LevelTransitionRequestV1`を発行する。
2. Runtime OrchestratorがSource／Target Level、Portal、condition、Target supportを検証する。
3. Target LevelのStreaming Planとdependency closureをprefetchする。
4. Definition、Asset、Physics、Navigation、System implementationのhashを照合する。
5. Target activation groupを`resident`まで準備する。
6. Tick boundaryでTarget Levelを`activating`し、失敗時は全target activationをrollbackする。
7. Targetが`active`になった後だけCharacter transfer Commandを適用する。
8. Transition completion Eventをsealする。
9. 旧Levelを`deactivating`し、lease／job／queue解放後にevictableへ進める。
10. Save checkpointが必要なら新Level active後のauthoritative Stateを保存する。

6または7が失敗した場合、旧Levelをactiveのまま維持する。旧Levelを先に破棄してLoading失敗後に空Worldへ落とさない。Seamless presentationはこのauthority順序を変えない。

## 10. 参照と依存closure

- Source Document／Level間はUUIDv7 `StableId`、MCD契約は`McdContractRefV1`、Derived Artifactは`ArtifactRefV1`で参照する。Cell間参照は同じPlan内の`uint32 cell_id`だけを使い、Plan外ではPlanの`ArtifactRefV1`を必須とする。
- Cross-cell C++ pointer、Scene object pointer、Vendor object参照を保存しない。
- hard dependencyはActivation前にclosure全体をresidentにする。
- soft dependencyはfallbackまたはtyped unavailable behaviorを必須とする。
- circular Asset／Cook dependencyを拒否する。
- Runtime handleはgeneration付きで、Cell deactivate後のstale accessをDiagnosticにする。
- Global entityはWorld scope ownerを一つだけ持ち、各Levelへ複製しない。
- Cross-Level persistent Characterはplay-sessionまたはworld-instance Systemが所有し、Level transferをCommandで行う。

## 11. Procedural World

### 11.1 `ProceduralWorldDefinitionV1`

| Field | 型／規則 |
|---|---|
| `procedural_world_id` | UUIDv7 `StableId` |
| `generator_contract_ref` | exact version |
| `generator_implementation_ref` | Qualified DefinitionまたはNative variant |
| `seed_policy` | `fixed | project_parameter | save_slot | session_derived` |
| `input_definition_refs` | 0～1,024件 |
| `layout_constraint_refs` | 1～1,024件 |
| `output_schema_ref` | exact generated delta schema |
| `max_generation_steps` | 正の整数上限 |
| `max_output_entities` | 正の整数上限 |
| `time_memory_budget_refs` | Targetごとに厳密に1件 |
| `determinism_contract_ref` | RNG stream、canonical order、numeric policy |
| `validation_fixture_refs` | 1～1,024件 |
| `failure_policy` | retry seed／fallback layout／abort |

GeneratorはSource `WorldDocument`を直接変更しない。`GeneratedWorldDeltaV1`をStagingへ出力し、Schema、bounds、connectivity、overlap、Asset、Navigation、Budget、playabilityを検証してからProjectChangeSetへ変換する。

```text
GeneratedWorldDeltaV1
  delta_id
  procedural_world_ref
  base_project_revision
  generator_contract_hash
  generator_implementation_hash
  input_hash
  seed
  rng_stream_manifest_hash
  create_records[0..max_output_entities]
  update_operations[0..max_output_entities * 4]
  delete_stable_ids[0..max_output_entities]
  generated_anchor_refs[]
  generated_portal_refs[]
  output_bounds
  generation_step_count
  output_hash
```

`delta_id`はTrusted Staging Brokerが発行するUUIDv7 `StableId`である。`create_records`はGatewayが未使用StableIdを割り当てる前の`uint32 local_id`を1から使い、0をinvalid、Delta内重複をerrorとする。検証成功後にGatewayが`local_id -> UUIDv7 StableId`対応を返し、Sourceへlocal IDを残さない。既存StableIdの更新／削除はDefinitionの明示allowlistとexpected Document revisionを必須とする。Generatorが既存World全体を置換するDelta、上限を超えたRecord、absolute path、native pointer、Cooked Artifact本文を含むDeltaを拒否する。

### 11.2 Runtime generation

C1／Phase 4のAI生成はEditor／Cook時だけである。Shipping Runtime generationはPhase 9の独立Gateとし、次を追加で満たす場合だけ許可する。

- Cook済みgenerator implementation以外のCodeを生成、download、compile、loadしない。
- 入力、seed、generator version、output hashをSave／Replayへ記録する。
- CPU、memory、step、output countにhard boundがある。
- timeout、invalid output、unsupported Targetで検証済みfallbackへ移る。
- Runtime generated deltaは許可済みComponent／Asset／System schemaのsubsetだけを作る。

## 12. Navigationとの境界

Level／WorldはNavigation Artifactを所有、編集、保存しない。

- Source: geometry semantic、walkable／blocked／cost／link Intent、Agent Profile。
- Derived: NavMesh、grid、volume、graph、tile、acceleration structure。
- Runtime: Navigation PlatformがArtifactをloadし、version付きQuery／Resultを提供する。
- Level gameplayは移動要求をNavigation Command／Queryへ渡し、backend objectを読まない。
- Streaming activation groupは必要Navigation tileをhard dependencyとして宣言できる。
- Navigation tileが未residentならCellをactiveにせず、明示fallbackがあるAgentだけ別方式を使う。
- HLODまたはrender proxyをNavigation Sourceへ使わない。

Procedural outputはNavigation source semanticsを含められるが、Navigation Artifact hashをSourceとして保存しない。

## 13. `MapPresentationDefinitionV1`

Map PresentationはUI／Presentation Systemであり、World／Level gameplayのauthoritative ownerではない。

| Field | 型／規則 |
|---|---|
| `map_presentation_id` | UUIDv7 `StableId` |
| `presentation_kind` | `minimap | world_map | level_map | navigation_overlay` |
| `world_or_level_ref` | 厳密に1件 |
| `projection_policy` | `orthographic_2d | authored_2d | projected_3d` |
| `layer_refs` | 1～128件 |
| `marker_style_refs` | 0～512件 |
| `marker_source_contract_refs` | typed Snapshot／Eventのみ |
| `fog_policy_ref` | 0または1件 |
| `accessibility_profile_refs` | 1～32件 |
| `localization_namespace_ref` | 厳密に1件 |
| `render_budget_refs` | Targetごとに厳密に1件 |
| `fallback_contract` | map非表示、簡易表示等の明示 |

Marker、fog、cursor、camera viewはauthoritative World Stateのprojectionである。PresentationからWorld Entity、Quest、Objective、Navigation costを直接writeしない。Playerが地図上で目的地を指定する場合はtyped `MapInteractionCommandV1`をNavigation／Quest ownerへ送り、ownerが受理または拒否する。

Fog of warにGameplay意味がある場合、visibility／knowledgeを専用authoritative Game Systemが所有し、Map PresentationはSnapshotを表示する。UI textureまたはreveal animationをSaveへ保存しない。

## 14. `WorldAuthoringBundleV1`

World変更を複数の既存ChangeSetへまたがって追跡するcoordination envelopeとする。

### 14.1 `WorldAuthoringPlanV1`

```text
WorldAuthoringPlanV1
  plan_id
  project_revision
  contract_set_hash
  map_intent_resolution_hash
  requirement_ids[1..256]
  target_profile_ids[1..32]
  affected_world_refs[1..64]
  affected_level_refs[0..1024]
  create_document_kinds[0..64]
  source_change_kinds[1..6]
  required_system_refs[0..128]
  required_capability_refs[0..128]
  budget_refs[1..64]
  derived_build_jobs[0..256]
  validation_fixture_ids[1..1024]
  assumptions[0..32]
  blocking_questions[0..7]
  fallback
  risk_class
  disposition
```

`plan_id`はUUIDv7 `StableId`、Requirement／System／Capability参照はexact `McdContractRefV1`、World／Level／Target／Budget／Fixture参照はUUIDv7 `StableId`＋revision／content hashである。表示名または配列indexを参照に使わない。

`source_change_kinds`は本書4.1節の6分類subsetである。`disposition`は`ready_to_stage | question_required | capability_unavailable | target_unsupported | budget_missing | rejected`とする。`ready_to_stage`にはMap intentが`resolved`、全Stable refが実在またはCreate対象、全Targetに意味同等fallback、全Budget／fixtureが存在することを必須とする。PlanはCommit権限ではない。

```text
WorldAuthoringBundleV1
  bundle_id
  project_id
  base_project_revision
  contract_set_hash
  map_intent_resolution_hash
  requirement_ids[]
  world_document_changeset_hashes[]
  topology_changeset_hashes[]
  level_definition_changeset_hashes[]
  gameplay_definition_changeset_hashes[]
  system_bundle_hashes[]
  asset_changeset_hashes[]
  procedural_delta_hashes[]
  navigation_intent_changeset_hashes[]
  map_presentation_changeset_hashes[]
  expected_derived_artifact_refs[]
  target_profile_ids[]
  budget_refs[]
  fixture_hashes[]
  risk_class
  required_gate_ids[]
```

`bundle_id`と`project_id`はUUIDv7 `StableId`、ChangeSet／Delta／Artifact／Fixture参照はSHA-256、Target／Budget参照はUUIDv7 `StableId`＋revision／content hash、Requirement／Gate参照はexact MCD ID＋versionである。

Bundleは変更本文を埋め込まず、Staging SHA-256とUUIDv7 `StableId`を参照する。9種類のSource ChangeSet／Bundle／Delta hash arrayの合計は1件以上とし、全参照は同じProject、base revision、Contract set、Map intentへ解決する。AIはBundle全体をProposalとして作れるが、Source Documentを直接Commitできない。

### 14.2 Domain／AI operation

World変更の正規Domain Operationを次へ固定する。GUI、keyboard command、AI Proposal、bulk toolは同じOperationを生成し、View固有callbackまたは任意field writeを正規経路にしない。

| Domain Operation | 原子的に変更する意味 | 暗黙に変更しないもの |
|---|---|---|
| `CreateLevelDefinition` | Level ID、World、Source Scene、entry／exit、System、Budget、fallbackの初期contract | Entity、Portal、Cell |
| `SetLevelSourceScenes` | LevelとScene shardのmembership集合 | Entity永続化owner、Scene bounds、Cell |
| `SetLevelEntryExitContract` | entry／exit Anchorとcompletion／fallback contract | Topology edge、Objective進捗 |
| `SetLevelGameplayComposition` | authoritative Level gameplay owner、Objective／Spawn／Encounter／Profile参照 | 各SystemのRuntime State |
| `CreatePortal`／`UpdatePortalContract`／`DeletePortal` | Level／Anchor間edge、condition、transition、fallback | Character transfer、Runtime activation |
| `MoveEntityToScene` | Authoring規約のScene永続化owner、subtree Shard record、明示したroot parent | Level membership、StableId、subtree内部parent、Cell |
| `SetSpatialPartitionIntent` | Target-independent residency intent | Target別Cell layout、HLOD、Runtime residency |
| `SetProceduralWorldDefinition` | seed、generator、constraint、bound、fallbackを持つSource recipe | accepted World Source、Runtime generation |
| `SetMapPresentationDefinition` | minimap／world map／marker／fogのPresentation Source | Quest、Objective、Navigation cost |

Operationは対象StableId、expected Document revision、precondition hash、変更前後の閉じたcontract sectionを持つ。entryだけ、exitだけ、Portal片側だけのように中間不整合を作るfield列へ分解せず、複数Documentにまたがる場合は`WorldAuthoringBundleV1`がexact ChangeSet hashを束ねる。

| Operation | 種類 | 結果 |
|---|---|---|
| `mirakan.worlds.search` | Query | World／Level／Region／roleを検索 |
| `mirakan.worlds.read` | Query | 選択Source、Topology、Budget、Target supportを取得 |
| `mirakan.worlds.resolve_map_intent` | Query／Proposal | `MapIntentResolutionV1` |
| `mirakan.worlds.plan_change` | Proposal | `WorldAuthoringPlanV1` |
| `mirakan.worlds.validate_bundle` | Query／Job | Staging BundleのDiagnostic |
| `mirakan.worlds.preview_bundle` | Job | isolated previewとcomparison artifact |

AIへ`commit_world`、`activate_cell`、`replace_streaming_plan`、`write_navmesh`を公開しない。CommitはAuthoringCommandGateway、Runtime activationはRuntime Orchestrator、Derived Artifact promotionはCook／Build serviceだけが行う。

### 14.3 Authoring sequence

1. GameSpec、Project Profile、Target、既存World／Level UUIDv7 `StableId`をResolveする。
2. Map intentを一つ以上の正規kindへ分解する。
3. 影響するSource Documentとdependency closureだけを読む。
4. World Authoring Plan、Budget、Risk、fallbackを作る。
5. BundleをStagingし、Schema／semantic／reference／licenseを検証する。
6. Procedural outputがあればisolated deltaとして生成する。
7. CookerがTarget別Streaming、Navigation、LOD、Package Artifactを生成する。
8. Headless playability、transition、Save／Replay、performanceを検証する。
9. EditorでSource差分、visual preview、Topology／Budget差分を比較する。
10. 承認後に一つのProjectChangeSetとしてCommitし、read-backする。

## 15. AI-readable UX

Editorは同じSourceを次のViewで編集する。

| View | 主用途 | 正本 |
|---|---|---|
| World Outline | World／Region／Level／Scene階層 | Stable ref projection |
| Topology Graph | Level／Portal接続 | Topology Definition |
| Level Form | Entry、objective、System、profile | Level Definition |
| Spatial View | Entity placement、bounds、anchor | World／Scene Document |
| Streaming Inspector | Cell、residency、budget、dependency | Derived Plan、read-only |
| Navigation Overlay | walkability、cost、tile、query | Source＋Derived comparison |
| Map Presentation Preview | minimap／world map／marker | Presentation Definition |
| Bundle Review | Requirement、Diff、Test、Budget、Risk | Staging Bundle |

World向けのbounded Contextを`WorldAuthoringContextV1`へ固定する。

```text
WorldAuthoringContextV1
  project_id
  project_revision
  contract_set_hash
  authoring_selection_context_hash optional
  world_ref
  scene_refs[0..256]
  level_refs[0..256]
  topology_ref
  topology_version
  viewport_bounds optional
  target_profile_refs[1..32]
  source_document_refs[1..1024]
  read_only_derived_artifact_refs[0..1024]
  capability_refs[0..128]
  budget_refs[1..64]
  decision_and_lock_refs[0..128]
  omitted_ranges[0..128]
  continuation
```

AI contextは現在選択中のWorld、Scene、Level、Viewport範囲、Target Profile、Contract hashに限定する。全World payload、全Asset、全CellをPromptへ投入しない。検索結果からUUIDv7 `StableId`を選び、必要なDocument fragmentだけを取得する。`WorldAuthoringContextV1`はAuthoring規約の`AuthoringSelectionContextV1`とCommit済みSourceから生成するread-only projectionであり、保存、Commit、Replay headerへ含めない。

全Viewは上部Context barにProject revision、World／Scene／Level Stable ref、Target、lock、`Source | Staging | Derived read-only | Runtime`区分を表示する。Sourceを編集する操作だけがDomain Operationを生成でき、Streaming InspectorのCell、Navigation Artifact、HLOD、Runtime handleをdrag、Inspector edit、AI instructionからSourceへ書き戻さない。選択同期はStableIdとContext hashで行い、screen coordinate、表示row、Object名、Hierarchy pathをAIまたはUI Automationの操作対象にしない。

Sceneの永続化owner、Level membership、Target別Cell assignmentを別関係として表示する。EntityをScene間移動してもLevel membershipを暗黙変更せず、LevelへSceneを追加してもEntity recordを移動しない。Source Sceneの一つを複数Levelが参照する場合、どのLevel Contextで編集しているかを表示し、Level固有変更と共有Scene変更の影響範囲をCommit前に比較する。

人間が移動したEntity、修正したPortal、locked Scene、Source hunkをAIが全面再生成で消さない。Proposalはprecondition hashを持ち、staleなら再baseを要求する。

## 16. Save、Replay、Migration

### 16.1 保存するもの

- World／LevelのUUIDv7 `StableId`とdefinition version。
- Active Levelの`LevelSaveInstanceId`とentry Anchor `StableId`。`LevelSaveInstanceId`はsave対象instance生成時にGatewayが発行するUUIDv7で、load時に新しい`LevelInstanceHandle`へ対応付ける。Runtime handle自体は保存しない。
- authoritative Level／World Game System State。
- persistent Entity StateとWorld delta。
- Procedural seed、generator contract／implementation version、input hash、accepted output hash。
- Portal／Topologyのauthoritative unlock state。ただしTopology Source自体ではない。
- Game意味を持つknowledge／visibility State。

### 16.2 保存しないもの

- Cell residency、prefetch queue、IO offset。
- Streaming Plan内部index。
- HLOD／render proxy／occlusion result。
- Navigation Artifact、tile cache、path result。
- minimap texture、marker animation、UI layout cache。
- Document pointer、C++ object address、Vendor handle。

Source／Contract version変更時はexact `McdContractRefV1`と、そのType内で不変の`uint32 field_id`による一方向Migrationを行う。Level ID、Portal ID、Anchor IDをrenameだけで再発行しない。削除する場合はSave migrationでreplacement、terminal outcome、unsupportedのいずれかを明示する。

Replay headerへWorld revision、Topology version、Level version、Streaming Plan hash、Navigation Artifact hash、Procedural seed／output hash、System dependency graph hashを記録する。Streaming timingそのものはauthorityへ影響させず、Replayで同じResidency scheduleを要求しない。

## 17. Diagnosticとfailure

| Code | 条件 | 結果 |
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

Failure時に別Level、別Portal、別Assetへ名前類似で自動置換しない。Gameplay意味が変わるfallbackはGameSpec変更とReviewを必要とする。

## 18. TestとQualification

### 18.1 Contract／authoring

- 全Schemaのvalid／invalid／boundary fixture。
- UUIDv7 `StableId`、rename、delete、migration。
- SceneとLevelの多対多参照、owner一意性。
- `world_authoring_semantics_v1` fixtureで、共有Sceneを参照する二つのLevel、一つのLevelを構成する複数Scene、Targetごとに異なるCell planを同時に表現する。
- `MoveEntityToScene`、`SetLevelSourceScenes`、Cell再Cookが互いのidentityとmembershipを暗黙変更しない。
- Topology reachability、trap、cycle、Target fallback。
- unknown／stale／cross-cell pointer negative test。
- Undo／redo、crash recovery、concurrent edit conflict。

### 18.2 Runtime

- Cell全state transition、cancel、timeout、IO failure。
- activation group atomicity、旧Level維持。
- Level transition、Character transfer、queue／lease解放。
- Save／Load／Replay state hash。
- inactive／resident／active境界でauthoritative処理が漏れないこと。
- Presentation、LOD、Camera、GPU結果からauthorityへ逆入力しないこと。

### 18.3 Procedural／Navigation

- 同じseed、input、Target、Toolchainで同じoutput hash。
- Generator step／time／memory／entity hard bound。
- Connectivity、entry-to-objective-to-exit reachability。
- Physics overlap、spawn safety、Navigation query。
- invalid output、timeout、unsupported Targetのfallback。
- Navigation Artifact削除後の決定論的再生成。

### 18.4 Performance

- Compact 2D／3D Levelのframe、memory、load、activation hitch。
- Cell size／prefetch policyのTarget別comparison。
- Peak CPU／GPU／IO memory。
- worst-case Portal traversalとcamera speed。
- HLODあり／なしのvisual floorとauthority同等性。
- warm cacheだけでなくcold start／cold streaming。

### 18.5 AI Eval

- 「マップ」が6分類のどれかを正しく解決する。
- ambiguityとhigh-impact変更で質問する。
- Scene、Level、Cell、Navigation、Presentationを混同しない。
- `WorldAuthoringContextV1`に含まれない表示名またはHierarchy pathからStableIdを推測しない。
- World Outline、Topology Graph、Level Form、Spatial View、AIから同じ意味変更を行う`world_authoring_cross_view_v1` 64 scenarioで、正規Domain Operationとafter state hashが一致する。
- 既存UUIDv7 `StableId`と人間変更を保持する。
- Level生成にObjective、entry／exit、Budget、Testを含める。
- C++だけ、geometryだけ、screenshotだけの不完全Proposalを拒否する。
- unavailable CapabilityまたはTargetを推測で成功扱いしない。
- Derived read-onlyまたはRuntime対象への変更要求をSource Operationへ近似せず拒否し、編集可能なSource Intent候補を説明する。

`world_authoring_intent_v1`はholdout 240件を持ち、明確な6分類各30件、曖昧またはHigh Impact 60件で構成する。3 runの最悪値で、明確Caseの`selected_kind`正解率97%以上、Blocking Caseの`question_required` recall 100%、存在しないStableIdを含む最終Proposal 0件、Scene／Level／Cell identity誤変更0件、Derived／Runtime直接write提案0件をPromotion Gateとする。

`WorldQualificationReceiptV1`はSource revision、Topology／Level／Partition hash、Target Profile、Streaming／Navigation／LOD Artifact hash、System graph、fixture、correctness、performance、Review Receiptを結ぶ。

## 19. 導入Phase

| Phase | 導入内容 | Gate |
|---|---|---|
| Phase 0 | Meta-schema、ID／参照taxonomy、最小Topology／Level fixture、negative test | 実Game実装なし |
| Phase 1 | Headless World／Scene／Level Document、ProjectChangeSet、Runtime World lifecycle | T00／T01相当 |
| Phase 2 | World Outline、Scene、Topology Graph、Level Form、Streaming Inspector、cross-view selection／Undo | Windows空Level edit→play→save→cook→package、`world_authoring_cross_view_v1` |
| Phase 3 | Compact 2D Level、Portal transition、Level gameplay、Asset／Navigation／Camera／UI接続 | C1 2D First Playable |
| Phase 4 | AI Map intent resolver、World Bundle、Level生成、preview、再編集 | AI First Playable＋holdout Eval |
| Phase 6 | Compact 3D Level、3D Navigation／Physics／Camera、同一Level Contract | C1 3D |
| Phase 8 | World Partition、HLOD、大規模World、advanced procedural authoring | 個別C2 Gate |
| Phase 9 | bounded runtime procedural generation | Shipping security／determinism Gate |

Phase 3までのSource modelはPhase 8の大規模Worldへ移行できるUUIDv7 `StableId`、Intent、Derived Plan分離を持つ。ただしPhase 8 ArtifactをPhase 0から生成しない。

## 20. Definition of Done

- World、Scene、Level、Region、Portal、Cell、Navigation、Map Presentationの定義とownerが重複しない。
- `MapIntentKindV1`とResolverが曖昧な要求を推測実行しない。
- World／Scene Document、Topology、Level、Partition Intent、Streaming Plan／Cell、Level Transition、Procedural Definition／Generated Delta、Map Presentation、Authoring Plan／BundleのSchemaがある。
- Source IntentとTarget別Derived Artifactを相互に編集しない。
- Level gameplay ownerが厳密に一つである。
- Cell activation groupがall-or-nothingで、失敗時に旧Levelを維持する。
- Cross-cell pointer、Presentation authority write、stale Planを自動検出する。
- Compact 2D／3D Levelが同じContractでTitleからResultまで完走する。
- AIがRequirementからWorld Bundle、Target Cook、Preview、Test、Commit proposal、再編集を完走する。
- `AuthoringSelectionContextV1`と`WorldAuthoringContextV1`から、GUI、keyboard、UI Automation、AIが同じStableId／revision／Domain Operationへ収束する。
- Scene永続化owner、Level membership、Cell assignmentをcross-view編集、Undo、再Cook後も混同しない。
- `world_authoring_semantics_v1`、`world_authoring_cross_view_v1`、`world_authoring_intent_v1`が各Gateを通る。
- Procedural outputが決定論、bound、connectivity、playability Gateを通る。
- Navigation、LOD、Map PresentationがDerived／Presentation境界を越えてauthorityへ逆入力しない。
- Save／ReplayがStreaming residencyとBackend objectから独立する。
- Targetごとのmemory、IO、activation hitch、entity／physics／navigation上限が継続計測される。
- Phase 8機能をC1対応と誤表示せず、Qualification ReceiptがCatalog maturityを決める。

## 21. 外部Evidenceと採用判断

調査基準日は2026-07-20である。外部Engineは責務分離と制作課題を確認するEvidenceに限定し、Object model、既定grid、Scene format、API名をMiraikanaiの公開契約へ複製しない。

- [Unreal Engine 5.8 World Partition](https://dev.epicgames.com/documentation/en-us/unreal-engine/world-partition-in-unreal-engine): large worldの自動data management、distance-based streaming、grid cell、HLOD、Data Layerを専用機能として扱うことを確認した。Miraikanaiでは特定gridを契約化せず、Source IntentとTarget別Planへ分ける。
- [Unreal Engine 5.8 One File Per Actor](https://dev.epicgames.com/documentation/en-us/unreal-engine/one-file-per-actor-in-unreal-engine): Actor単位の外部保存がLevel file競合を減らす一方、encoded file名とdangling referenceをEditor内Changelistで検証する必要を確認した。MiraikanaiではScene Shard、StableId、reference closure、ProjectChangeSet Reviewへ一般化する。
- [Unreal Engine 5.8 Level Streaming](https://dev.epicgames.com/documentation/en-us/unreal-engine/level-streaming-in-unreal-engine): persistent Levelと複数Levelのload／unloadによるWorld構成を確認した。MiraikanaiではLevel、Scene、Cellのidentityを分離し、Runtime OrchestratorだけがActivationする。
- [Unity 6.3 LTS Key concepts](https://docs.unity3d.com/6000.3/Documentation/Manual/key-concepts.html): SceneがEnvironmentとMenuを含むcontent単位で、複数Sceneを組み合わせられることを確認した。MiraikanaiではAuthoring shardとGameplay Levelを同一語へ固定しない。
- [Unity 6.3 LTS LoadSceneMode.Additive](https://docs.unity3d.com/6000.3/Documentation/ScriptReference/SceneManagement.LoadSceneMode.Additive.html): 現在Sceneをunloadせず追加Sceneをloadする実用的要求を確認した。MiraikanaiではAPI模倣ではなくCell residencyとActivation state machineへ一般化する。
- [Unity 6.3 LTS Undo API](https://docs.unity3d.com/6000.3/Documentation/ScriptReference/Undo.html): property、Component、Object作成／破棄、parent変更をEditor Undoへ明示登録する経路を確認した。Miraikanaiでは操作種別ごとのAPI分岐を`ProjectChangeSet`とDomain Operationへ統合する。
- [Godot 4.7 Scene organization](https://docs.godotengine.org/en/4.7/tutorials/best_practices/scene_organization.html): self-contained scene、疎結合、依存注入、単一責務を推奨する設計を確認した。MiraikanaiではState owner、Stable reference、Graph validatorで機械検証する。
- [Godot 4.7 Nodes and Scenes](https://docs.godotengine.org/en/4.7/getting_started/step_by_step/nodes_and_scenes.html): node treeをsceneとして保存し、sceneを再利用・instance化するcompositionを確認した。MiraikanaiではWorld Document compositionとRuntime System ownershipを分ける。
- [Godot 4.7 PackedScene](https://docs.godotengine.org/en/4.7/classes/class_packedscene.html)および[Running code in the editor](https://docs.godotengine.org/en/4.7/tutorials/plugins/running_code_in_the_editor.html): 永続化されるNode集合が`owner`で決まり、Editor script変更にはunsaved表示またはUndo対応が必要なことを確認した。MiraikanaiではScene Shardの`scene_id`を永続化ownerとし、dirty stateとUndoをGatewayへ集約する。

採用判断を次へ固定する。

| 外部で実証された原則 | Miraikanaiでの採用 | 複製しないもの |
|---|---|---|
| Component／propertyを構造化Editor APIとUndoで変更する | MCD field、Domain Operation、ProjectChangeSet、inverse Operation | GameObject／Component class hierarchy、Unity Scene format |
| 大規模Worldを分割保存し、Streamingを専用機能にする | Scene Shard、Spatial Intent、Target別Derived Cell、reference closure validation | Actor／UObject、OFPA filename、固定World Partition grid、`.umap` |
| Scene compositionと永続化ownerを明示する | Recipe、Scene永続化owner、Level membership分離、`MoveEntityToScene` | Node／PackedScene tree、`owner` property名、`.tscn` |
| Editor操作にTransaction／dirty／Undoを要求する | Gateway Commit、revision、Recovery、inverse ChangeSet | 外部Engine固有transaction bufferまたはcallback |

## 22. 明示的な非採用

- `MapSystem`または`MapManager`一つにWorld、Level、Streaming、Navigation、UIを集約する方式。
- Scene、Level、Cellを同じidentity／lifecycleとして固定する方式。
- World全体を一つの巨大binary／JSON／C++ translation unitへ保存する方式。
- EditorのSource DocumentへCook済みCell、NavMesh、HLOD、backend handleを書き戻す方式。
- Target別Streaming cell layoutを共通Source contractにする方式。
- Cross-cell／Cross-Level pointerまたはVendor objectの永続化。
- Camera、render visibility、HLOD、GPU queryでEnemy、Damage、Objectiveを決める方式。
- Target budget超過時にEntity、Objective、Portalを黙って削除する方式。
- AIが「マップ」を曖昧なままgeometry生成またはUI生成へ決め打ちする方式。
- AIがscreen coordinate、Hierarchy path、表示名だけで対象を選び、`.unity`、`.umap`、`.tscn`相当のSource fileを直接書き換える方式。
- Scene永続化owner、Level membership、Streaming Cellを一つのparent／folder／layer関係から推測する方式。
- Procedural generatorがWorld正本を直接上書きする方式。
- seed、version、bound、fallbackを持たないprocedural generation。
- Shipping RuntimeでC++、Shader、bytecode、native／managed executableを生成、download、compile、loadする方式。
- World Partition／large-world機能をC1達成前に空Class群として先行実装する方式。
