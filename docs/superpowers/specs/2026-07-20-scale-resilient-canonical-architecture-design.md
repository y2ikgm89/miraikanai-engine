# Miraikanai Engine Scale-Resilient Canonical Architecture規約

- 文書版: 1.0
- 作成日: 2026-07-20
- 最終更新日: 2026-07-20
- 対象: 中規模／大規模Game制作、World、Entity、Asset、Authoring、Build、Save、AI Context、将来のDistributed Authority
- 状態: ユーザーReview待ちの規範設計案
- 上位目的: [AIネイティブ独自ゲームエンジン 設計計画書](./2026-07-18-ai-native-game-engine-authoring-design.md)
- 基盤規約: [Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)
- Authoring正本: [Miraikanai Engine Authoring Model／Project State規約](./2026-07-19-authoring-model-project-state-design.md)
- Runtime正本: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- World正本: [Miraikanai Engine World／Level／Map／AI Authoringアーキテクチャ規約](./2026-07-20-world-level-map-ai-authoring-architecture-design.md)
- Contract正本: [Miraikanai Engine 実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)
- LOD正本: [Miraikanai Engine AI可読LODアーキテクチャ規約](./2026-07-20-ai-readable-lod-architecture-design.md)
- Game Project配置正本: [Miraikanai Engine AI可読Game Project配置・命名規約](./2026-07-20-ai-readable-game-project-layout-naming-design.md)
- 技術識別子正本: [Miraikanai Engine AI可読命名・技術識別子規約](./2026-07-20-ai-readable-engine-naming-convention-design.md)

## 1. 結論

Miraikanai Engineは、中規模用と大規模用に別のProject形式、World object、Entity identity、Save形式、AI Operationを作らない。

人間とAIが編集する正本を規模非依存のCanonical Sourceへ固定し、Target、Quality、World、Population、Content、Authoring、Authorityの宣言されたScale Envelopeから、Engine-owned Compilerが次のTarget別Derived Planを生成する。

- World partition／streaming／activation plan
- Entity representation／simulation LOD／pool reservation plan
- Asset dependency／residency／partial cook plan
- HLOD／render visibility plan
- Physics／Navigation partition plan
- Authoring shard／index／bounded AI context plan
- Build／cook work partition plan
- 将来のreplication／authority partition plan

規模拡大によって変更してよいものはDerived PlanとRuntime representationである。Source Stable ID、Gameplay meaning、Save identity、Requirement、Decision、State owner、Capability contractを変更してはならない。

大規模機能をPhase 0へ空実装しない。中規模実装が大規模化を阻害するshortcutを持たないことをContract testで固定し、Large World、Massive Simulation、Large Project Authoring、Distributed Authorityは、それぞれ専用仕様、prototype、Qualificationが成立した後だけ個別にActivatedする。

## 2. 本書の決定権

### 2.1 本書が決定するもの

本書は次の正本である。

- 規模を一つの「小／中／大」文字列へ縮退させないScale axis
- `ProjectScaleEnvelopeV1`の共通Envelope
- Canonical Source、Derived Plan、Runtime State、Evidenceの一方向関係
- 規模変更で維持するcross-Subsystem invariant
- 中規模から大規模へ移行する際のidentity、Save、Replay、Migration、fallback規則
- AIがScaleを検索、読取、提案、解決、説明、検証するOperation
- Scale CapabilityのActivation、Qualification、rollback規則
- 下位四仕様の責務と着手順序

### 2.2 本書が変更しないもの

次は既存の所有文書を正本とする。

| 主題 | 正本 |
|---|---|
| UUIDv7、Artifact hash、Runtime handle、Memory、Pointer | 基盤規約 |
| ProjectRevision、Document、Shard、ChangeSet、Commit、Undo、Migration | Authoring Model規約 |
| Runtime phase、State writer、Command／Event／Snapshot、Budget | Runtime規約 |
| World／Scene／Level／Region／Cell／Topology／Streaming Plan | World規約 |
| LOD metric、Policy、fidelity、fallback、Receipt | LOD規約 |
| MCD Type／Operation／Capability／Diagnostic／Projection | Contract規約 |
| Asset Source／Derived／Package／VFS／Cook／Patch | Asset Pipeline規約 |
| Multiplayer／Distributed Authorityの具体的Network方式 | 将来のDistributed Authority専用仕様 |

同じ主題で矛盾する場合、本書はScale横断不変条件を決定し、各所有文書はDomain内の具体的な型、phase、budget、backendを決定する。下位仕様は本書の不変条件を緩和できず、本書は下位仕様の安全条件を概略表現で上書きできない。

## 3. 対象と対象外

### 3.1 対象

- Level制2D／3D Gameの中規模Production
- Content全体がmemoryへ収まらず、Level境界でresident setを切り替えるProject
- Continuousまたはinterest-source based streamingを必要とするLarge World
- Full EntityだけではTarget budgetを満たさない大量Population
- Scene、Asset、Build、Cook、Packageを分割しなければ制作できないProject
- 複数のAuthoring worker／人間／AIが異なるDocumentを変更するProject
- 将来のclient／serverまたはserver shard authorityを追加できるSource境界

### 3.2 現在Activatedしないもの

次は本書だけでは利用可能にならない。

- continuous origin rebasing
- planetary／space scale coordinate
- cross-server authoritative migration
- client prediction／reconciliation
- replication graph／interest management
- distributed transaction
- live multi-user Scene editing
- remote Build farm scheduling
- distributed Asset service

該当する専用仕様、Threat Model、State machine、failure test、Target benchmarkが存在しない場合、Capability Catalogは`NotActivated`を返す。AI、Editor、Project、Providerが独自Defaultで有効化してはならない。

## 4. 調査結果と採用判断

調査基準日は2026-07-20とし、外部EngineのAPI名や実装をMiraikanaiの正規契約へコピーしない。採用するのは大規模制作で実証された責務分離の原則である。

### 4.1 Unity

Unity Entities 1.4は、GameObject／MonoBehaviour authoringをBakerでEntityへ変換し、SubSceneからEntity Sceneを生成する。Archetype／Chunkへ同一Component構成を集約し、System queryから処理する。Unity 6 Addressablesはaddressとdependencyから非同期Asset loadを行う。

採用する原則:

- Authoring representationとRuntime representationの分離
- Source変更からDerived Entity Sceneを生成するBaking境界
- 同一構成dataをChunkへ集約するdata-oriented execution
- Asset dependencyを含む非同期load

採用しない点:

- 通常GameObject WorldとECS Worldを別の正規Game modelとして併存させる
- Authoring objectとRuntime entityの対応をEditor慣習だけへ依存させる
- SubScene identityをGameplay Level identityまたはSave identityと同一視する

### 4.2 Unreal Engine

Unreal Engine 5.8のWorld Partitionはpersistent Levelをstreaming cellへ分割し、Streaming Sourceからload／unloadする。One File Per ActorはActor instanceを別fileへ保存し、Data LayerはEditor／RuntimeのWorld layerを分離する。HLODはunloaded cellのstatic contentをproxyへ置換し、Builder CommandletはWorld全体を常時loadせずHLOD、Navigation、PCG等を処理する。Mass EntityはFragment、Archetype、Chunk、Processorを分離する。Large World CoordinatesはCPUのdouble variantとGPU向け高精度表現を持つ。

採用する原則:

- Source editing shard、Runtime streaming cell、HLOD proxyを別identityにする
- Streaming sourceとruntime grid／partition policyを分離する
- Static presentation proxyをauthoritative gameplayへ逆入力しない
- 大規模Build／Cookをheadless、iterative、partition単位で実行する
- 大量Entityのdataとprocessingを分離する
- 大規模座標をMath、Physics、Renderer、VFXまでcross-cuttingに扱う

採用しない点:

- 一つの固定grid方式をCanonical World Contractへ埋め込む
- Actor file、Data Layer、runtime cell、HLOD Layerの組合せをAIが慣習から推測する
- Presentation HLOD、network relevancy、simulation LODを一つの距離値で決める

### 4.3 Godot

Godot 4.5はSceneTree／Node／Resourceを高水準Authoringに用い、大量instanceではRenderingServer／PhysicsServerとMultiMeshを低水準経路として利用する。MultiMeshは大きな一括描画を可能にする一方、個別instance cullingを自動提供しない。Large World Coordinatesはdouble precision buildであり、memory／performance costを伴う。ResourceLoaderはbackground loadを提供する。

採用する原則:

- 通常object経路を維持しつつ、実測後だけ大量data経路へ変換する
- 低水準Server／Backend handleをSourceへ保存しない
- 大量instanceのbatch boundaryをspatialに分割し、all-or-nothing cullingを避ける
- double precisionを全Targetへ無条件適用せず、明示Capabilityとcostで選ぶ

採用しない点:

- 高水準Nodeから低水準Serverへの移行判断をProject codeへ委ねる
- SceneTree外objectを別Save／AI modelとして管理する
- large coordinate precisionをBuild optionだけで意味決定する

### 4.4 採用方式

次の三方式を比較した。

| 方式 | 長所 | 破綻要因 | 判断 |
|---|---|---|---|
| 固定World PartitionをPublic Contractにする | 実装とtoolを早期に一本化しやすい | Grid以外のTopology、2D、Portal、space、Target差をSourceへ固定する | 不採用 |
| Authoring objectとECS objectを二つの正本にする | 通常制作と大量処理を個別最適化しやすい | identity、Save、Debug、AI Contextが二重化する | 不採用 |
| Canonical SourceからDomain別Derived Planを生成する | 中規模と大規模が同じ意味modelを使い、Target別最適化を隔離できる | Compiler、Receipt、同値fixtureが必須 | 採用 |

## 5. 規範語とScale用語

本書の「必須」「禁止」「だけ」は機械検証または明示Review Gateを持つ規範である。「推奨」は代替を選ぶADRと同値fixtureを要求する。「候補」はCapability CatalogへProductionとして掲載できない。

### 5.1 四層

| 層 | 正規内容 | 永続 | 人間／AI編集 |
|---|---|---:|---:|
| Canonical Source | Requirement、World、Entity、Asset metadata、GameplayDefinition、Scale Intent、Decision | する | ChangeSet経由だけ可 |
| Derived Plan | Cell、HLOD、Representation、Residency、Cook work、将来Authority partition | ArtifactRef付きで保存可 | 直接編集禁止 |
| Runtime State | generation handle、queue、active cell、resident resource、ECS chunk | Saveへ直接保存しない | Runtime ownerだけ |
| Evidence | Trace、Benchmark、Diff、Explanation、Qualification Receipt | hash付きで保存 | 追記生成だけ |

Runtime StateまたはEvidenceからSourceへ自動で値を書き戻さない。EngineまたはAIはEvidenceからSource ChangeSetを提案できるが、通常のValidation、Preview、Approval、Commitを迂回できない。

### 5.2 Scale axis

「大規模」を単一boolまたは一つのenumへ縮退させない。`ProjectScaleEnvelopeV1`は次の五axisを独立に持つ。

#### World axis

| 値 | 意味 |
|---|---|
| `bounded_level` | 一つのLevel activation groupが明示境界内に収まる |
| `explicit_level_graph` | Portal／loading transitionでbounded Levelを切り替える |
| `continuous_partitioned` | Play中にinterest sourceからCell residencyが連続変化する |
| `planetary_or_space` | bounded local floatだけでは宣言precisionを維持できない |

#### Population axis

| 値 | 意味 |
|---|---|
| `full_entity` | peak active authoritative objectをFull Entityで処理できる |
| `pooled_or_batched` | 意味を変えずpool、batch、instanceでTarget budgetを満たす |
| `simulation_lod` | 契約済みactive／reduced／dormant／aggregate state遷移が必要 |
| `distributed_simulation` | authoritative populationがprocessまたはserver境界を越える |

#### Content axis

| 値 | 意味 |
|---|---|
| `single_working_set` | 選択Targetの必須Content closureが一つのbounded working setへ収まる |
| `incremental_partitioned` | dependency単位の増分Import／Cook／Packageが必要 |
| `partial_cook_streamed` | Region／Cell／chunk単位の選択CookとRuntime deliveryが必要 |
| `distributed_content_service` | remote worker／cache／content serviceがBuild成立条件になる |

#### Authoring axis

| 値 | 意味 |
|---|---|
| `single_writer` | 同時に一つのAuthoring writerだけが変更する |
| `optimistic_multi_writer` | 異なるChangeSetをbase revisionとfield単位でmerge判定する |
| `partition_owned_multi_writer` | Region／Document／Shard ownershipとleaseが制作成立条件になる |
| `federated_repository` | 複数repositoryまたはsite間同期が制作成立条件になる |

#### Authority axis

| 値 | 意味 |
|---|---|
| `single_process` | 一つのGame processがauthoritative simulationを所有する |
| `client_server` | Serverがauthorityを所有しclientへreplicateする |
| `sharded_server` | 複数authoritative server間でownershipを移動する |

`client_server`と`sharded_server`はDistributed Authority専用仕様が承認されるまで`NotActivated`である。Single-playerのbackground worker、Render process、Asset workerはGameplay Authorityを持たないため`single_process`のままである。

各axis表の列挙順をschema上の意味順序0～3とする。ただしwire値は上記文字列enumであり、ordinal integerをSource、Save、Operationへ保存しない。比較ValidatorはContract compilerが生成するclosed順序表を使い、文字列比較、追加順、Provider出力順から大小を推測しない。

### 5.3 表示用Scale class

EditorとAIは五axisから次の`scale_class`を決定的に導出できる。ただし、Capability解決は表示値でなく各axisと数値Envelopeを使用する。

1. Authorityが`client_server`または`sharded_server`なら`distributed_candidate`。
2. Authorityが`single_process`で、Worldが`continuous_partitioned`以上、Populationが`simulation_lod`以上、Contentが`partial_cook_streamed`以上、Authoringが`partition_owned_multi_writer`以上のいずれかなら`large_local_candidate`。
3. 上記以外で、Worldが`explicit_level_graph`、Populationが`pooled_or_batched`、Contentが`incremental_partitioned`、Authoringが`optimistic_multi_writer`のいずれかなら`medium_candidate`。
4. すべて最初の値なら`compact_reference`。

`scale_class`とTarget別`qualification_status = predicted | optimization_required | qualified | not_activated`を別に表示する。`medium_candidate／qualified`だけをMedium Production、`large_local_candidate／qualified`だけをLarge Local Productionと表示できる。C1 vertical slice、推定costだけ、未実測TargetをProductionと表示しない。

この順序以外のheuristic、Project名、Genre、総file数だけでScale classを決めない。

## 6. `ProjectScaleEnvelopeV1`

Authoring Document kindを`ProjectScaleEnvelopeDocument`、payloadを`ProjectScaleEnvelopeDocumentV1`へ固定する。一つのProjectは現在revisionでexactly oneのactive `ProjectScaleEnvelopeDocument`を`ProjectManifest`から参照し、`ProjectProfileDocument`はそのDocument IDと`scale_envelope_id`を参照する。二件以上、参照なし、Project不一致、stale Document revisionをrejectする。

`ProjectScaleEnvelopeDocumentV1`はexactly oneの`ProjectScaleEnvelopeV1`、`WorldScaleIntentV1`、`ContentScaleIntentV1`、`AuthoringScaleIntentV1`、`AuthorityScaleIntentV1`を含み、1件以上の既存`RuntimeScaleIntentV1`をStable ID参照する。Intent recordはDocument内の順序や配列indexで参照せず、自身のUUIDv7 Stable IDを使う。

全Targetの最小値でSource表現を削らず、TargetごとにQualification statusを持つ。

### 6.1 共通Field

| Field | Contract |
|---|---|
| `scale_envelope_id` | UUIDv7 `StableId` |
| `project_id` | 親Project UUIDv7 |
| `schema_version` | `uint32`、現在版だけを保存 |
| `source_revision` | Envelopeを確定した`ProjectRevision` |
| `target_profile_refs` | exact StableId＋revision／hash、1～16件 |
| `world_axis` | 5.2節のclosed enum |
| `population_axis` | 5.2節のclosed enum |
| `content_axis` | 5.2節のclosed enum |
| `authoring_axis` | 5.2節のclosed enum |
| `authority_axis` | 5.2節のclosed enum |
| `world_intent_ref` | `WorldScaleIntentV1` StableId＋revision |
| `population_intent_refs` | `RuntimeScaleIntentV1` StableId set、1件以上 |
| `content_intent_ref` | `ContentScaleIntentV1` StableId＋revision |
| `authoring_intent_ref` | `AuthoringScaleIntentV1` StableId＋revision |
| `authority_intent_ref` | `AuthorityScaleIntentV1` StableId＋revision |
| `gameplay_fidelity_floor_refs` | Requirement／Game System exact MCD ref、1件以上 |
| `integrated_fixture_refs` | `TestScenarioDocument` StableId set、1件以上 |
| `decision_refs` | Scale判断を所有するDecision StableId set |

表示用`scale_class`をSourceへ重複保存しない。Projectionが返す場合は五axisとEnvelope hashから導出し、導出規則versionを付ける。

### 6.2 必須数値Envelope

次は「分からない」を0、最大値、無制限、空optionalで表さない。値が確定しない場合は`MissingScaleEnvelope` Blocking Diagnosticとし、Cook／Shipping Qualificationへ進めない。

| Intent | 必須値 |
|---|---|
| `WorldScaleIntentV1` | Dimension、Topology、authored bounds／Region数、最大active範囲、最大移動速度、teleport有無、要求position tolerance、Target |
| `RuntimeScaleIntentV1` | total authored、peak live、peak active authoritative、peak spawn／tick、peak visible、interaction radius、simultaneous VFX、fidelity floor |
| `ContentScaleIntentV1` | Source Asset数／bytes、Derived closure bytes、install bytes、peak CPU／GPU resident bytes、locale数、patch envelope |
| `AuthoringScaleIntentV1` | concurrent writer数、最大Document／Shard数、ChangeSet operation上限、Preview／Commit latency budget、branch／merge policy |
| `AuthorityScaleIntentV1` | authority mode、process／server数、connection上限、ownership transfer有無、tick／latency／bandwidth envelope |

単位はMath／Core Utilities規約のsemantic typeを使用する。Wire上primitiveが必要な場合だけ`_m`、`_mps`、`_bytes`、`_ms`、`_hz`を使用する。非finite、負数、逆転range、Target未指定、fidelity floor空をrejectする。

新しいIntent recordの最低Fieldを次へ固定する。

#### `WorldScaleIntentV1`

| Field | Contract |
|---|---|
| `world_scale_intent_id` | UUIDv7 |
| `world_ref` | World Stable ID＋Document revision／hash |
| `dimension` | `two_d | three_d | hybrid` |
| `topology_mode` | World axisと同じclosed enum |
| `authored_extent` | `WorldExtentV1`。proceduralでもQualification上限を必須とする |
| `maximum_simultaneously_active_extent` | Dimension付きfinite extent、authored extent以下 |
| `maximum_travel_speed_mps` | canonical decimal string、0以上、fraction最大3桁 |
| `teleport_policy` | `not_required | bounded_destination_set | arbitrary_valid_region` |
| `teleport_ready_deadline_ms` | teleportを要求するvariantで1以上 |
| `required_position_resolution_m` | canonical decimal string、正数、fraction最大9桁 |
| `target_profile_refs` | EnvelopeのTarget subset、1件以上 |

`WorldExtentV1`は`two_d { width_m, height_m } | three_d { width_m, height_m, depth_m } | hybrid { two_d_extent, three_d_extent }`のtagged unionである。各lengthは指数表記なしのcanonical decimal string、正数、fraction最大3桁とし、絶対originまたは座標表現を意味しない。`maximum_simultaneously_active_extent`も同じtagged unionを使い、各dimensionが`authored_extent`以下でなければならない。

`planetary_or_space`でも「無限」を保存しない。proceduralまたは拡張可能Worldは、現在Qualificationする最大extent、speed、active set、precisionを値として宣言し、超過要求は新Envelope revisionと再Qualificationを必要とする。

#### `ContentScaleIntentV1`

| Field | Contract |
|---|---|
| `content_scale_intent_id` | UUIDv7 |
| `source_asset_count` | `uint64`、1以上 |
| `source_asset_bytes` | `uint64`、1以上 |
| `derived_closure_bytes_by_target` | Target ref→`uint64`、全Targetを含む |
| `install_bytes_by_target` | Target ref→`uint64`、全Targetを含む |
| `peak_cpu_resident_bytes_by_target` | Target ref→`uint64`、Target budget以下 |
| `peak_gpu_resident_bytes_by_target` | Graphics Target ref→`uint64`、Target budget以下 |
| `locale_count` | `uint32`、1以上 |
| `maximum_patch_bytes_by_target` | Target ref→`uint64` |
| `delivery_mode` | Content axisと同じclosed enum |

#### `AuthoringScaleIntentV1`

| Field | Contract |
|---|---|
| `authoring_scale_intent_id` | UUIDv7 |
| `concurrent_writer_count` | `uint32`、1以上 |
| `maximum_document_count` | `uint64`、1以上 |
| `maximum_scene_shard_count` | `uint64`、1以上 |
| `maximum_operations_per_changeset` | `uint32`、1～4,096 |
| `preview_latency_budget_ms` | `uint32`、1以上 |
| `commit_latency_budget_ms` | `uint32`、1以上 |
| `writer_mode` | Authoring axisと同じclosed enum |
| `conflict_policy` | `reject_overlap | field_disjoint_merge_candidate`。最終Commit前に全Validatorを再実行 |

#### `AuthorityScaleIntentV1`

| Field | Contract |
|---|---|
| `authority_scale_intent_id` | UUIDv7 |
| `authority_mode` | Authority axisと同じclosed enum |
| `simulation_tick_hz` | `uint32`、1以上 |
| `process_or_server_count` | `uint32`。`single_process`ではexactly 1、Network variantでは1以上 |
| `maximum_connection_count` | Network variantだけで1以上 |
| `maximum_round_trip_latency_ms` | Network variantだけで1以上 |
| `maximum_aggregate_bandwidth_bytes_per_second` | Network variantだけで1以上 |
| `allows_authority_transfer` | `single_process`ではfalse。Network variantは専用仕様のclosed policyで決定 |

`AuthorityScaleIntentV1`の`single_process` variantでは`process_or_server_count = 1`、`allows_authority_transfer = false`とし、connection、latency、bandwidth Fieldを存在させない。`client_server`／`sharded_server` variantは専用仕様未Activated中にSchema validationまではできるが、resolve、cook、play、packageを`MIRAKAN-SCALE-DISTRIBUTED_AUTHORITY_NOT_ACTIVATED`で拒否する。

### 6.3 Envelopeの変更

Scale Envelope変更は通常の`ProjectChangeSet`であり、次を必須とする。

- expected Project／Document revision
- Before／After axisと数値
- 影響するTarget、Capability、Artifact、Fixture、Decision closure
- Gameplay fidelity floor差分
- 再Cook／再Qualification範囲
- Save／Replay／Package互換性判定
- rollback先のlast valid Envelope revision

数値を下げて性能合格を作る変更は自動最適化ではない。敵数、味方数、Damage、Collision、Navigation、Goal、spawn timing、World範囲、同時接続数を下げる場合、既存の`GameplayScaleChangeProposalV1`と人間承認を必須とする。

## 7. Canonical Source不変条件

### 7.1 Identity

1. World、Scene、Level、Region、Entity、Asset、Rule、UI、Scale IntentはUUIDv7 `StableId`を持つ。
2. Stable IDはrename、Directory移動、Scene／Shard／Cell移動、repartition、recook、HLOD化、instance化、simulation LOD、server shard変更で変えない。
3. MCD Contractはexact `McdContractRefV1`、Derived Artifactは`ArtifactRefV1`、Runtime objectはgeneration handleを使う。
4. Runtime handle、pointer、GPU address、Vendor ID、Plan-local IDをSourceまたはSaveへ保存しない。
5. Plan-local IDを永続参照する場合、owner `ArtifactRefV1`とPlan schema version／hashを同時保存し、別Planの同じ数値と比較しない。

### 7.2 Source／Derived分離

- Scene Shardはcollaborative edit単位であり、Gameplay LevelまたはRuntime Cellではない。
- LevelはGameplay単位であり、Scene fileまたはStreaming Cellではない。
- Region／Partition IntentはSourceであり、Cell boundaryはTarget別Derived Planである。
- HLOD、Navmesh、Physics aggregate、ECS Chunk、GPU instance、replication listはDerived／Runtimeである。
- Derived Planの再生成だけでsemantic Source diff、Save identity変更、AI Decision変更を作らない。
- SourceからDerivedへのCompiler input closureと出力hashをReceiptへ保存する。

### 7.3 State authority

- Presentation visibility、HLOD、GPU occlusion、frame rate、network relevancyからGameplay authorityを決めない。
- Simulation LODはGameplay契約済みstate transitionだけを使い、distanceまたはvisibilityだけでenemy、Damage、Collision、Navigation、Goal参加を削らない。
- Runtime Stateのauthoritative writer、consume phase、lease、versionをRuntime規約へ割り当てる。
- Cell unload、Entity dormant、server transfer中もSave対象stateのownerを一意にする。
- Owner不在、二重owner、owner generation不一致ではstateをpublishしない。

### 7.4 Reference closure

- Source cross-Document／cross-Region参照はStable IDを使う。
- Cookerは参照を同じactivation groupへまとめる、soft dependencyへ変換する、またはunsupported cycleとして拒否する。
- Runtimeはunloaded objectのpointerを保持せず、Stable IDからversion付きresolver／leaseを取得する。
- Activation groupはauthoritative closureをall-or-nothingでpublishする。
- Presentation-only dependencyは明示fallbackを持つ場合だけ遅延可能である。

### 7.5 Save、Replay、Migration

- SaveはStable ID、exact Contract ref、Source／Level version、authoritative State fieldを保存する。
- SaveへCell ID、Shard ID、HLOD ID、ECS Chunk、Runtime handle、GPU／Physics／Nav Backend handleを保存しない。
- Replay headerはProject revision、Contract set hash、Scale Envelope hash、World／Streaming／Representation Plan hashを記録する。
- Repartition後もSource Stable IDとauthoritative replay digestを維持する。
- Schema変更は不変`field_id`によるoffline一方向Migratorを必須とし、削除Field IDを再利用しない。
- Migration失敗時はSource／Save原本を変更せず、旧Engineまたはlast valid migrated copyで回復できるReceiptを残す。

## 8. Domain別Resolverと統合

`ScaleManager`、`LargeWorldManager`、`OptimizationManager`のような万能Runtime ownerを作らない。

各Domainは同じ`ProjectScaleEnvelopeV1`と自身のIntentを読み、次のPlanを所有する。

| Domain | Source | Derived Plan | Runtime owner |
|---|---|---|---|
| World | World／Topology／Partition Intent | `WorldStreamingPlanV1` | World／Level lifecycle owner |
| Runtime | Runtime Scale Intent、Game System Contract | `RuntimeRepresentationPlanV1` | RuntimeOrchestrator配下のDomain owner |
| LOD | `LodIntentV1`、Domain Policy | `LodResolutionPlanV1` | 各Presentation／Simulation owner |
| Asset | Asset metadata、dependency、Content Intent | Asset closure／residency／package plan | Asset Runtime |
| Render | Material、Style、World／LOD Plan | Render Representation／Graph | Renderer |
| Physics | Collider／Physics Source、World Plan | Physics partition／aggregate | Physics Platform |
| Navigation | Navigation Source、World Plan | Nav Artifact／tile activation | Navigation Platform |
| Authoring | Document、Shard、Authoring Intent | Context index／ownership／work plan | Authoring Service |
| Build | Source DAG、Target、Content Intent | Build／Cook work graph | Build Gateway |
| Network | Authority Intent、Game System Contract | Replication／authority plan | 未Activated |

`ScalePlanSetV1`は各Plan本文を埋め込まず、exact ArtifactRef、Source revision、Target Profile、Capability signature、dependency edge、Qualification statusを束ねるManifestである。

Runtime開始前に全required PlanのSource revision、Contract set、Target、Capability signature、dependency hashが一致しなければならない。一件でもstale、missing、unqualifiedなら新Plan setをpublishせず、last valid playableを維持する。

## 9. AI可読契約

### 9.1 AIが理解する対象

AIへ公開するのは次である。

- Scale axisの意味と現在値
- 数値Envelopeと単位
- Gameplay fidelity floor
- Source Stable IDとbounded dependency closure
- Target別Resolved Planの要約
- Capability availability／maturity／Qualification
- cost、fallback、未達理由、Evidence
- ChangeSetへ使用できるtyped Operation

AIへ公開しないもの:

- native pointer／address
- Runtime handle
- Vendor object
- raw ECS Chunk memory
- GPU／Physics／Navigation Backend ID
- unsigned Derived binary
- credential、signing key
- World全体または全Projectの無制限dump

### 9.2 MCD Operation

次をMCDの唯一のScale Operation集合とする。

| Operation | Risk | 結果 |
|---|---|---|
| `operation.scale.search` | R0 | axis、Intent、Plan、Fixture候補をbounded検索 |
| `operation.scale.read_envelope` | R0 | exact Envelope、Decision、fidelity、Qualificationを取得 |
| `operation.scale.dependencies` | R0 | inbound／outbound Source／Plan／Fixture closureを取得 |
| `operation.scale.resolve_preview` | R1 | Sourceを変更せずTarget別Plan差分とcostを予測 |
| `operation.scale.explain_plan` | R0 | Plan選択理由、fallback、Evidence、反証条件を取得 |
| `operation.scale.propose_envelope_change` | R2 | typed `ProjectChangeSet`候補を生成 |
| `operation.scale.validate_transition` | R0 | medium→large、Plan切替、fallbackの不変条件を検査 |

MCP aliasは技術識別子規約に従い`mirakan.scale.search`、`mirakan.scale.read_envelope`、`mirakan.scale.dependencies`、`mirakan.scale.resolve_preview`、`mirakan.scale.explain_plan`、`mirakan.scale.propose_envelope_change`、`mirakan.scale.validate_transition`とする。

ProviderへProject Commit、Plan write、Capability activate、baseline緩和、Source直接write、server authority移動を公開しない。

### 9.3 Bounded Context

全Query結果は次を返す。

- `project_revision`
- `contract_set_hash`
- `scale_envelope_id`／revision／hash
- `target_profile_ref`
- `capability_signature_hash`
- `authoring_index_revision`
- `query_hash`
- 選択Itemと選択理由
- `omitted_ranges`
- `continuation_cursor`
- Evidence／Qualification Receipt hash

AIはWorld全体、全Entity、全AssetをPromptへ投入しない。Task anchor、Region／Level／System Stable ID、field mask、dependency depth、byte／token budgetから`SceneSliceV1`とScale projectionを作る。

別revisionへのfallback、名前だけの参照確定、omitted rangeの存在を隠した要約、stale Receiptによる成功表示を禁止する。

### 9.4 Explanation Receipt

`ScaleExplanationReceiptV1`は次を必須とする。

| Field | 意味 |
|---|---|
| `receipt_id` | UUIDv7 |
| `source_revision` | 入力ProjectRevision |
| `scale_envelope_hash` | 入力Envelope |
| `target_profile_ref` | 解決Target |
| `plan_set_hash` | 説明対象Plan |
| `selected_strategies` | partition、batch、LOD、streaming等のclosed strategy ID |
| `rejected_strategies` | 候補、拒否Requirement、Evidence |
| `fidelity_proof_refs` | Gameplay不変fixture |
| `cost_measurement_refs` | CPU／GPU／memory／I/O／build／network |
| `fallback_chain` | 順序付きPlan ref |
| `qualification_status` | `predicted | optimization_required | qualified | not_activated` |
| `invalidated_by` | Source、Target、Toolchain、Capability、baselineのhash条件 |

自由文理由だけで`qualified`にしない。AI向け自然言語説明はこのReceiptから生成するProjectionである。

### 9.5 Diagnostic

Scale Domainの最低Diagnostic codeを次へ固定する。

| Code | 条件 |
|---|---|
| `MIRAKAN-SCALE-MISSING_ENVELOPE` | 必須axis、Intent、数値、Fixtureがない |
| `MIRAKAN-SCALE-AMBIGUOUS_REQUIREMENT` | 「大規模」「多数」「高速」等を数値／意味へ解決できない |
| `MIRAKAN-SCALE-UNQUALIFIED_CAPABILITY` | 必要CapabilityにTarget Receiptがない |
| `MIRAKAN-SCALE-PLAN_STALE` | Source／Contract／Target／Capability hash不一致 |
| `MIRAKAN-SCALE-FIDELITY_VIOLATION` | Gameplay意味を無承認で低下する |
| `MIRAKAN-SCALE-REFERENCE_CLOSURE_INVALID` | cross-boundary参照をactivation／fallbackへ解決できない |
| `MIRAKAN-SCALE-BUDGET_EXCEEDED` | hard CPU／GPU／memory／I/O／queue／build budget超過 |
| `MIRAKAN-SCALE-PARTIAL_ACTIVATION_REJECTED` | authoritative closureの部分publish要求 |
| `MIRAKAN-SCALE-DISTRIBUTED_AUTHORITY_NOT_ACTIVATED` | Network／server authorityを未承認で要求 |

未知値を近いenum、0、最大値、現在TargetのDefaultへ補正しない。Diagnosticはexpected、actual、location、Requirement、Target、Remediation候補を持つ。

## 10. 中規模から大規模への非破壊遷移

### 10.1 許可する変更

- Partition IntentとTarget Profileを追加し、新しいDerived Cell Planを生成する
- 同じEntity Sourceからinstance、batch、HLOD、simulation LOD Artifactを生成する
- Asset closureをRegion／chunkへ分割する
- Scene Entityを別Authoring Shardへre-shardする
- Build／Cook graphを複数bounded work itemへ分割する
- Target別Quality／residency／fallbackを追加する

### 10.2 禁止する変更

- Large用Entity typeへSourceを一括変換し、元のStable IDを失う
- Medium用SaveとLarge用Saveを別Schemaへforkする
- Cell境界を親子関係、Quest、Damage、Navigation authorityの意味へ使う
- HLODまたはGPU instanceをSave対象Entityとして扱う
- AIが性能のため敵数、World範囲、Collision、Goalを無承認で削る
- Large TargetのためMedium fallbackを削除する
- Medium TargetのためLarge Source表現を破棄する
- Build shard、Scene shard、Runtime cell、server shardを同一identityにする
- unqualified Derived Planを`Production`表示する

### 10.3 意味同値

同じSource revisionと入力traceを共通範囲で実行した場合、Medium PlanとLarge Planは次を一致させる。

- authoritative Game System state digest
- Save fieldとStable ID
- Input→Command→Eventの順序
- Goal、Damage、Collision、Navigation result
- deterministic random stream
- Level／Region transition outcome
- Replay first divergenceが存在しないこと

Presentation pixel、animation sample、particle、HLOD mesh、streaming timingはbitwise一致対象外にできるが、Style／visual tolerance、critical cue floor、event timing、fallback Gateを満たす。

## 11. Large Worldへの準備境界

現在のC1／Medium Sourceは`WorldPosition2f`／`WorldPosition3f`を明示的なLevel／Region coordinate space内で使う。裸の`Vec*`、implicit global origin、無制限world floatをPublic Sourceへ保存しない。

`continuous_partitioned`または`planetary_or_space`をActivatedする前に、専用Large World仕様で次を一意に決定する。

- Canonical global position representation
- Region／cell＋local positionまたはdouble／fixed representationの採否
- origin shift／rebase phase
- Physics、Navigation、Renderer、Audio、VFX、Camera、Save、Replay変換
- cross-region Transformとparent制約
- precision toleranceとTarget別cost
- teleport／high-speed traversal
- floating-point conversion report
- Multiplayer authorityとの順序

専用仕様が存在しない間、EngineはRegion-local bounded Sourceを維持し、continuous large-worldをCapability gapとして拒否する。`double`を念のため全Runtimeへ導入せず、`WorldPosition3f`をplanetary scaleに誤用しない。

この境界により、Medium実装はlocal coordinate spaceを明示するだけでLarge World representationへ移行でき、具体方式を未検証のまま固定しない。

## 12. Massive Entity／Simulationへの準備境界

大量Entityを通常objectの個数上限だけで拒否しない。SourceはStable ID、Game System Contract、Scale Intent、fidelity floorを維持し、Derived Planが次を選ぶ。

- Full Entity
- pooled Full Entity
- SoA／archetype chunk
- instanced presentation
- reduced-frequency simulation
- dormant state record
- aggregate simulation
- HLOD／render proxy
- CPU／GPU VFX Artifact

各Representationはclosed strategy ID、entry／exit Predicate、owner、State mapping、Save mapping、recovery、fallback、budgetを持つ。距離だけでRepresentationを決めず、Gameplay relevance、interaction、authority、Target、visible working set、critical cueを入力にする。

`distributed_simulation`はsingle-process simulation LODの延長として暗黙実装しない。Server authority、partition、cross-shard event、transaction、save、recoveryを専用仕様で決定するまでNotActivatedとする。

## 13. Large Project Authoring／Buildへの準備境界

### 13.1 Authoring

既存の`SceneEntityShardDocument`上限4,096 Entity recordまたは8 MiB、Stable ID検索、Merkle root、`SceneSliceV1`、incremental Indexを維持する。

規模拡大でShard上限だけを増やさない。Entity数、Component size、reference closure、変更局所性に基づいてre-shardし、re-shardを`storage_only` diffとしてsemantic diffから分離する。

`partition_owned_multi_writer`をActivatedする前に専用仕様で次を決める。

- Project／Region／Document／Shard ownership lease
- optimistic mergeとの優先順位
- offline writer、lease expiry、clock、identity
- cross-partition ChangeSet
- lock／Decision dependency
- binary Asset checkout
- conflict、rebase、abort、recovery
- audit、Review、Promotion

### 13.2 Build／Cook

Build Gatewayを唯一の入口とし、Project file数またはWorld全体を一processへloadすることを要求しない。

Build／Cook work itemは少なくとも次を持つ。

- work item Stable ID
- Source／dependency closure hash
- Target／Configuration／Toolchain
- required Capability
- input／output ArtifactRef
- memory／time／I/O budget
- lease／deadline／cancellation
- retry class
- deterministic cache key
- provenance／SBOM

Remote workerまたはBuild farmは、signed work manifest、content-addressed input、network allowlist、output verification、untrusted worker Threat Model、duplicate completion、partial failureを決定する専用仕様なしにActivatedしない。

## 14. Distributed Authorityへの準備境界

Canonical Sourceは将来Networkingを許すが、現在のsingle-player Runtimeへreplication field、RPC、client role、server pointerを埋め込まない。

Game System Stateは今から次を明示する。

- State owner
- authoritative／presentation分類
- Command／Event／Snapshot
- fixed phase
- deterministic input
- Save field
- Stable ID
- conflict policy

Distributed Authority専用仕様は最低限、次を決定する。

- trust／threat model
- client／server／shard identity
- transport／session／encryption
- replication schemaとfield ownership
- interest／relevancy／priority／bandwidth
- prediction／reconciliation／rollback
- server tick／clock／deadline
- cross-shard entity migration
- distributed save／checkpoint
- reconnect／host failure／version compatibility
- abuse／cheat／privacy
- load／loss／latency／soak fixture

これらが存在しない間、AIはMultiplayer Requirementを`MIRAKAN-SCALE-DISTRIBUTED_AUTHORITY_NOT_ACTIVATED`として返し、single-player actor、NPC Navigation、local workerをnetwork-readyと説明しない。

## 15. FailureとRecovery

| Failure | 正規動作 |
|---|---|
| Envelope不足／曖昧 | Blocking Diagnostic。AIが値を推測せず質問または選択肢を返す |
| Derived Plan compile失敗 | 新Planをpublishせずlast valid Plan setとSourceを維持 |
| Cell／Activation dependency不足 | authoritative group全体を非Activeにし、空World／無衝突状態を公開しない |
| stale Source／Target／Contract | 結果破棄、current revisionで再計画 |
| Budget超過 | `OptimizationRequired`。baselineまたはfidelityを自動緩和しない |
| HLOD／Presentation Artifact不足 | 承認済みvisual fallback。Gameplay Sourceは維持 |
| Simulation LOD state復元失敗 | Full／last valid stateへfallback、不可ならActivationを拒否 |
| partial Cook／Package失敗 | last valid packageを維持し、partial manifestをShippingへ昇格しない |
| Migration失敗 | 原本を変更せず失敗Receiptとrecovery pathを返す |
| Authoring conflict | field／Document／Decision closure Diffを返し、部分Commitしない |
| Remote worker不正／不一致 | Artifactを隔離し、Trusted cache／Sourceへpublishしない |
| Authority capability未Activated | Play／Packageをfail-closedし、single-player fallbackが意味同等の場合だけ別案を提示 |

OOM、device lost、process crash、network interruption時もSourceまたはlast valid Packageを削除しない。RecoveryがSource meaningを変更する場合は別ChangeSetとApprovalを必要とする。

## 16. Securityと権限

- Scale QueryはR0、PreviewはR1、Envelope ChangeはR2とする。
- Native code、Build worker、Pack、Authority schema、Releaseは既存Risk classを緩和しない。
- AI ProviderへSource write、Commit、Capability Activation、Server ownership、Releaseを公開しない。
- Scene Slice、Scale Envelope、ReceiptはProject policyに従ってredactし、credential／personal data／license-restricted SourceをContextへ含めない。
- Remote workerはuntrusted input／outputとしてhash、signature、path、network、Toolchain、SBOMを検証する。
- Clientが提示するScale、position、Entity count、authority、Receiptを信頼しない。
- DiagnosticとExplanationは機密Source本文を含めず、Stable Evidence IDとbounded locationを返す。

## 17. VerificationとQualification

### 17.1 共通fixture

| Fixture | 検証 |
|---|---|
| `scale_envelope_contract_v1` | 全axis／tagged union／単位／range／missing／invalid／Projection同値 |
| `medium_to_large_same_source_v1` | 同じSourceをMedium／Large PlanへCookしauthoritative digest一致 |
| `repartition_identity_v1` | Region／Cell／Shard再分割後もStable ID、Save、Decision、semantic hash維持 |
| `plan_stale_rejection_v1` | Source／Target／Contract／Capability変更後のstale Plan拒否 |
| `activation_closure_atomicity_v1` | dependency不足でauthoritative closureを部分publishしない |
| `representation_fidelity_v1` | Full／pooled／chunk／dormant／aggregate切替のState、Save、Replay一致 |
| `medium_fallback_non_regression_v1` | Large Capability無効時にMedium基準経路が非退行 |
| `large_context_bounded_ai_v1` | 100万EntityでStable ID検索、Slice、closure、omitted range、stale rejection |
| `incremental_cook_equivalence_v1` | clean／incremental／partial CookのArtifact closure hash一致 |
| `last_valid_recovery_v1` | Compiler、Cook、Migration、Activation失敗でSourceとlast valid維持 |
| `scale_explanation_grounding_v1` | 全Plan判断がRequirement、Evidence、Receiptへ追跡可能 |
| `unactivated_authority_rejection_v1` | client／server／shard要求を全Provider／Editor／Cookで拒否 |

### 17.2 Medium Production Gate

`medium_candidate／qualified`表示には次をすべて必要とする。

- 2Dまたは3Dの一つ以上のProject固有`ProjectScaleEnvelopeV1`
- explicit Level graph、Save、Replay、Package
- Content全体とactive working setの分離
- incremental Import／Cook／Build
- Project固有Integrated Scale Fixture
- Target別CPU／GPU／memory／I/O／queue budget
- 2時間endurance run
- clean Save→現行Schema migration→load
- AI bounded ContextからLevel／Entity／Asset変更を提案し、手動変更を保持
- last valid Project／Package recovery

C1の5分／15分vertical sliceだけでMedium Productionを宣言しない。Project固有fixtureは実作品の最大同時体験を含み、Subsystem最大値を別runへ分離しない。

### 17.3 Large Local Gate

`large_local_candidate／qualified`表示にはMedium Gateに加えて次を必要とする。

- Large World、Massive Simulation、partial Cook、partition-owned Authoringのうち利用する専用仕様とCapability Receipt
- continuous traversal／teleport／camera cutまたは大量PopulationのProject固有trace
- partition boundary、cross-reference、load deadline、memory pressure、recovery
- same SourceのMedium fallbackまたは意味同等なbounded fallback
- repartition後のStable ID／Save／Replay
- large Context Slice、incremental Index、partial Diff
- clean／incremental／partial Cook Artifact同値
- 10分×3性能run、2時間endurance、failure injection

利用しないLarge Capabilityまで実装する必要はない。例えば大量Simulationだけを持つbounded ArenaはLarge World Capabilityを要求しない。

### 17.4 Distributed Large Gate

本書では定義しない。Distributed Authority専用仕様、Threat Model、server実機、loss／latency／abuse／recovery fixture、人間承認が揃うまでGate自体をActive Catalogへ掲載しない。

### 17.5 CI拒否条件

- Scale axisと数値Envelopeの不一致
- AI mutable fieldにtyped Operationがない
- SourceへDerived／Runtime IDを保存
- Source Stable IDがrepartitionで変化
- Plan-local IDのowner hash不足
- Runtime handleをSave／Replayへ保存
- PresentationからGameplay authorityへの逆入力
- stale Plan／ReceiptによるQualified表示
- fidelity floorを下げた無承認最適化
- Medium fallbackなしのLarge Capability Promotion
- 未Activated AuthorityのSchema／Tool／Package公開
- fixture、Target、metric、baseline、Receiptのいずれか不足

## 18. 段階設計と文書分割

本書を上位正本とし、次の順序で独立仕様を作成する。

| 順序 | 文書 | 決定する範囲 | Entry Gate |
|---:|---|---|---|
| 1 | 本書 `2026-07-20-scale-resilient-canonical-architecture-design.md` | Scale axis、Envelope、四層、不変条件、AI、共通Gate | 既存Review setとユーザー方向承認 |
| 2 | `2026-07-20-large-world-streaming-coordinate-architecture-design.md` | global／local座標、Region、Cell、origin、streaming、cross-region | 本書承認、Math／World／Runtime正本 |
| 3 | `2026-07-20-massive-entity-simulation-scale-architecture-design.md` | Chunk、Representation、Simulation LOD、aggregate、State復元 | 本書承認、Runtime／LOD／Game System正本 |
| 4 | `2026-07-20-large-project-authoring-build-distribution-architecture-design.md` | ownership、Shard、Index、partial Cook、remote worker境界 | 本書承認、Authoring／Asset／Build正本 |
| 5 | `2026-07-20-distributed-authority-networking-architecture-design.md` | Server authority、replication、prediction、shard、security | 1～4のSource／State境界が安定し、別Threat Model承認 |

文書2～4は相互に独立してReviewできるが、本書のStable ID、Source／Derived、State owner、Envelope、Receiptを共通利用する。文書5はNetworking実装を許可するものではなく、実装計画前に追加のSecurity Reviewを必須とする。

### 18.1 実装Phaseへの配置

- Phase 0: `ProjectScaleEnvelopeV1`、axis、Diagnostic、unactivated rejection、共通fixtureのMCD契約だけ。Large Runtimeの空Classを作らない。
- Phase 1: Authoring Document、Scale Query、Scene Slice、Envelope ChangeSet。
- Phase 2～3: Compact／Medium reference経路と測定。
- Phase 4: AI Scale discovery、Preview、Explanation、ChangeSet提案。
- Phase 8: 個別Qualification済みLarge World／Massive Simulation／Large Authoring Capability。
- Phase 8以後: Distributed Authority専用Gate後の隔離prototype。

## 19. Definition of Done

本書の設計完了は次をすべて満たした時点とする。

1. Scale axis、Envelope、四層、identity、authorityが一意に定義されている。
2. MediumとLargeがCapability maturity、Quality Target、Genreと混同されない。
3. 既存のWorld、Authoring、Runtime、LOD、Asset、Contract規約の所有権と矛盾しない。
4. Source Stable ID、Save、Replay、Migrationがrepartition／Representation変更で維持される。
5. AIがScaleをbounded queryし、typed ChangeSetだけを提案できる。
6. Unknown、unsupported、unqualified、not activatedを別Diagnosticとして返す。
7. Medium→Largeの意味同値fixtureとMedium fallbackがある。
8. Derived Planの理由、cost、fallback、EvidenceをReceiptから説明できる。
9. Large World、Massive Simulation、Large Project、Distributed Authorityの専用仕様境界が重複しない。
10. 専用仕様がないCapabilityをActiveまたはProductionと表示できない。
11. 実装taskへ対象file、contract、dependency、test、budget、rollbackを割り当てられる。
12. Placeholder、暗黙Default、実装段階で再決定する未割当の選択を残していない。

## 20. 主要リスクと確定対策

| リスク | 確定対策 |
|---|---|
| Large対応の抽象化がMedium実装を遅らせる | Phase 0はContractと拒否fixtureだけ。Large Runtime classを作らない |
| Medium／Largeで別data modelになる | Canonical SourceとStable IDを一つにし、Derived Planだけ変える |
| 万能Scale Systemが全Domainを所有する | Domain別Resolverと`ScalePlanSetV1` Manifestだけを共通化 |
| AIが「大規模」から方式を推測する | 五axis、数値Envelope、Blocking Diagnostic、bounded discovery |
| Grid方式へ固定される | SourceはPartition Intent、Target別CompilerがCell Planを決める |
| doubleを全Targetへ導入してcost増大 | Large World専用仕様とTarget QualificationまでNotActivated |
| ECS化でSave／Debug identityを失う | Stable ID↔Runtime handle mapping、SaveはSource identity |
| HLOD／visibilityがGameplayを変える | Presentation→authority逆入力をValidatorとfixtureで拒否 |
| Shard／Cell／server identityが混同される | Authoring、Runtime、AuthorityのID namespaceとowner hashを分離 |
| Partial load／Cookで壊れた状態を公開する | activation closureとPackageをatomic publish |
| Scale最適化で敵数やWorldを削る | fidelity floor、GameplayScaleChangeProposal、人間承認 |
| Large pathだけ進化してMedium fallbackが腐る | ablation、Medium non-regression、同一Source fixture |
| Multiplayerを後付けしてState ownerが崩れる | 今からowner／Command／Event／Snapshotを固定し、Networkは別Gate |
| 長期開発でProject／Saveが開けない | immutable field ID、offline migrator、原本維持、recovery Receipt |
| 外部Engineの機能名を模倣して責務が曖昧になる | 原則だけ採用しMiraikanai固有のIntent／Plan／Receiptへ変換 |

## 21. 一次資料

### 21.1 外部Engine

- Unity Entities 1.4: [Subscenes overview](https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/conversion-subscenes.html)
- Unity Entities 1.4: [ECS authoring and baking workflow](https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/ecs-workflow-example-authoring-baking.html)
- Unity 6: [Addressables](https://docs.unity3d.com/6000.0/Documentation/Manual/com.unity.addressables.html)
- Unreal Engine 5.8: [World Partition](https://dev.epicgames.com/documentation/en-us/unreal-engine/world-partition-in-unreal-engine)
- Unreal Engine 5.8: [One File Per Actor](https://dev.epicgames.com/documentation/en-us/unreal-engine/one-file-per-actor-in-unreal-engine)
- Unreal Engine 5.8: [Data Layers](https://dev.epicgames.com/documentation/en-us/unreal-engine/world-partition---data-layers-in-unreal-engine)
- Unreal Engine 5.8: [World Partition HLOD](https://dev.epicgames.com/documentation/en-us/unreal-engine/world-partition---hierarchical-level-of-detail-in-unreal-engine)
- Unreal Engine 5.8: [World Partition Builder Commandlets](https://dev.epicgames.com/documentation/en-us/unreal-engine/world-partition-builder-commandlet-reference)
- Unreal Engine 5.8: [Mass Entity overview](https://dev.epicgames.com/documentation/en-us/unreal-engine/overview-of-mass-entity-in-unreal-engine)
- Unreal Engine 5.8: [Large World Coordinates](https://dev.epicgames.com/documentation/en-us/unreal-engine/large-world-coordinates-in-unreal-engine-5)
- Unreal Engine 5.7: [Replication Graph](https://dev.epicgames.com/documentation/en-us/unreal-engine/replication-graph-in-unreal-engine)
- Godot 4.5: [CPU optimization](https://docs.godotengine.org/en/4.5/tutorials/performance/cpu_optimization.html)
- Godot 4.5: [Optimization using MultiMeshes](https://docs.godotengine.org/en/4.5/tutorials/performance/using_multimesh.html)
- Godot 4.5: [Thread-safe APIs](https://docs.godotengine.org/en/4.5/tutorials/performance/thread_safe_apis.html)
- Godot 4.5: [Background loading](https://docs.godotengine.org/en/4.5/tutorials/io/background_loading.html)
- Godot 4.5: [Large world coordinates](https://docs.godotengine.org/en/4.5/tutorials/physics/large_world_coordinates.html)

### 21.2 標準

- RFC 9562: [UUID Version 7](https://www.rfc-editor.org/rfc/rfc9562.html)
- RFC 8785: [JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785)
- Khronos glTF 2.0: [Specification](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html)

外部資料のversion更新は本書の判断を暗黙変更しない。新versionの事実、既存不変条件への影響、fixture結果、ADRをReviewしてから採用する。

## 22. 明示的に採用しないもの

- `is_large_game`だけで実装方式を決める
- Genre名または有名作品名からScaleを推測する
- Engine全体を初めからdouble precisionにする
- 一つの固定gridをWorld Source Contractにする
- Medium EntityとLarge Entityを別Source typeにする
- Authoring Shard、Streaming Cell、ECS Chunk、HLOD、Network Shardを同じ単位にする
- SourceにRuntime handle、pointer、GPU／Vendor IDを保存する
- AIへ全World、全Entity、全Assetを無制限に渡す
- AIがDerived Plan、Runtime Cell、ECS Chunk、replication listを直接編集する
- Presentation LODからSimulationまたはNetwork authorityを決める
- Subsystem単体benchmarkの合計をIntegrated Scale合格とする
- 大規模機能を空Class、未接続Schema、常時成功stubとして先行実装する
- 未Activated Multiplayerをsingle-player fallbackで成功扱いする
- performance baseline緩和と実装最適化を同じChangeSetで行う
- Migration失敗時にProject／Save原本を上書きする
