# Miraikanai Engine Game System／AI Code Generationアーキテクチャ規約

- 文書版: 1.2
- 作成日: 2026-07-20
- 最終更新日: 2026-07-21
- 調査基準日: 2026-07-20
- 対象: Game System、AI実装計画、GameplayDefinition、Project C++、System Catalog、Codegen、依存、検証、Promotion
- 状態: プロジェクト公式の規範設計レビュー版。実装待ち
- 上位文書: [AIネイティブ独自ゲームエンジン 設計計画書](./2026-07-18-ai-native-game-engine-authoring-design.md)
- Game実装方式: [Miraikanai Engine C++実行コード・構造化ゲームデータ規約](./2026-07-19-cpp-structured-game-data-design.md)
- 実行可能契約: [Miraikanai Engine 実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)
- Authoring状態: [Miraikanai Engine Authoring Model／Project State規約](./2026-07-19-authoring-model-project-state-design.md)
- Runtime: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- Project C++: [Miraikanai Engine NativeGameModuleアーキテクチャ規約](./2026-07-19-native-game-module-architecture-design.md)
- AI権限: [Miraikanai Engine AI実装・保守ガバナンス規約](./2026-07-19-ai-engine-development-governance-design.md)
- Game制作時のEngine不変境界・初心者承認: [Miraikanai Engine 不変Engine境界・初心者向けAI技術承認規約](./2026-07-21-immutable-engine-beginner-ai-approval-design.md)
- 検証: [Miraikanai Engine AI検証・評価・来歴規約](./2026-07-19-ai-verification-evaluation-provenance-design.md)
- World／Level／Map: [Miraikanai Engine World／Level／Map／AI Authoringアーキテクチャ規約](./2026-07-20-world-level-map-ai-authoring-architecture-design.md)
- Debugging／Replay: [Miraikanai Engine AI可読Debugging／Observability／Replayアーキテクチャ規約](./2026-07-20-ai-readable-debugging-observability-replay-architecture-design.md)
- Shooter Gameplay: [Miraikanai Engine AI可読Shooter Gameplay／Weapon／Projectileアーキテクチャ規約](./2026-07-20-ai-readable-shooter-gameplay-architecture-design.md)

## 1. 結論

Miraikanai Engineの公式Game System方式を、**契約固定・実装開放型（Contract-Stable／Implementation-Open）AI-native typed hybrid architecture**へ固定する。

ゲームのGenre、Core loop、System構成、Component、State、Rule、Algorithm、2D／3D表現を固定しない。Projectは、署名済みEngine baselineに含まれる標準System／Capabilityのcomposition、Project固有`GameplayDefinition`、bounded Project C++ `NativeGameModule`を必要に応じて選べる。一方、System間接続、State所有、MCD Contract ID／Version、Command、Event、Snapshot、Authority、Lifetime、Runtime phase、Save／Replay、Budget、Test、Target fallbackはMiraikanai Contract Definition（MCD）で一意にする。

```text
Natural language／Editor／IDE
  -> Game Brief／GameSpec
  -> GameSystemSpecV1
  -> SystemImplementationPlanV1
  -> SystemBundleChangeSetV1
     -> GameplayDefinition
     -> NativeGameModule
     -> World／Level／Asset／UI
     -> Test／Benchmark／Migration
  -> Validate／Cook／Build／Review
  -> Source activation＋ProjectRevision commit
  -> CookedGameplayPackage＋Game binary
```

制約する対象はゲーム内容ではなく、検証不能な接続方法である。System Catalogを固定Whitelistにせず、検証済みProject固有Systemを第一級Entryとして登録できるようにする。AIは`.cpp`断片を単独生成せず、同じSystem契約に結び付いたDefinition、Source、Build manifest、Save／Migration、Test、Budgetを一つのdependency closureとして提案する。

Shipping RuntimeのCPU実行Codeは引き続きC++23である。GameplayDefinitionは汎用Scriptではなくoffline Cookするbounded dataであり、C++ evaluatorが実行する。表現不能なAlgorithmまたは同一fixtureで必要性を実測したhot pathだけをNativeGameModuleへ実装する。RuntimeでC++、Shader、bytecode、native／managed executableを生成、download、compile、loadしない。

## 2. 決定権と境界

| 主題 | 正本 |
|---|---|
| Game Systemの意味、System Spec、Catalog、実装Variant、System Bundle、依存Graph、自由度原則 | 本書 |
| MCD kind、Type、Operation、State Machine、Schema projection、Contract compiler | 実行可能契約規約 |
| C++とGameplayDefinitionの責務、CookedGameplayPackage、実装選択の一般則 | C++実行コード・構造化ゲームデータ規約 |
| Project Document、ProjectChangeSet、Commit、Undo、Recovery | Authoring Model規約 |
| Tick phase、Command／Event／Snapshot、World lease、queue、memory、performance | Runtime規約 |
| NativeGameModule ABI、Source API、Build、Preview、Target link、Promotion | NativeGameModule規約 |
| AI Task、Risk、Activation、Source Worker、Approval | AI実装・保守ガバナンス規約 |
| Requirement coverage、Test、Eval、Receipt、provenance | AI検証規約 |
| World、Scene、Level、Streaming、procedural generation、Navigation、Map presentation | World／Level／Map規約 |
| Physics、Navigation、Animation、Renderer、UI等のDomain contract | 各Subsystem正式仕様 |

Game SystemはEngine Subsystemの別名ではない。Physics、Renderer、Navigation等はEngine Capabilityを所有し、Combat、Character、Level gameplay等のGame Systemが公開Portを消費する。Game SystemがVendor APIまたはEngine private objectを所有してはならない。

本書はmultiplayer製品機能をC1へ追加しない。ただしAuthority、State owner、Command／Event境界を省略してsingle-player専用の暗黙global stateを作ることも許可しない。将来multiplayerを導入する場合も同じSystem契約を拡張し、既存State所有を推測で変更しない。

## 3. 正規用語

| 用語 | 定義 |
|---|---|
| Engine Capability | Engine-owned C++が提供するversion付きType、Operation、Command、Event、Snapshot |
| Game System | 一つの明示責務、State owner、入出力契約、lifecycleを持つGameplay単位 |
| Engine Standard System | Engineが標準Contract、Reference Definition、Fixtureを提供するSystem |
| Project-defined System | Project namespaceで新しく定義し、MCDと全Gateを通したSystem |
| Engine Extension | 別のEngine製品開発工程で作られ、署名済みEngine baselineへ組み込まれたread-only Capability。Game制作AIは生成、変更、昇格できない |
| Public System Contract | System ID、State、Command、Event、Snapshot、Capability、Save意味の集合 |
| Implementation Variant | 同じPublic System Contractを実装するGameplayDefinition、Native C++、hybrid等の一実装 |
| System Bundle | System変更に必要なAuthoring、Definition、Source、Asset、Test、Migration、Receipt参照のclosure |
| Authoritative State Owner | 特定State Typeを正規に更新できる厳密に一つのSystem |
| Derived State | Authoritative Snapshotから再生成でき、正規Saveへ不要なcacheまたはprojection |
| Presentation State | Render、Audio、VFX、HUD等の非authoritativeな表示状態 |
| System Activation | 検証済みImplementation VariantをProjectRevisionとRuntime packageから参照可能にする操作 |

`Manager`、`Controller`、`Service`、`System`というSuffixだけで責務を判断しない。正規判断は`GameSystemSpecV1`の`semantic_role_ids`、`responsibility_requirement_ids`、State owner、Portで行う。

### 3.1 IDと参照の型

Game Systemでは次の三種類を混同しない。

| 型 | 表現 | 寿命／用途 |
|---|---|---|
| `GameSystemContractRefV1` | `{id: MCD Contract ID, version: uint32, contract_set_hash: SHA-256}` | Public System Contractの永続identity。Save、Replay、Catalog、Graphで使用 |
| `StableId` | 基盤規約のRFC 9562 UUIDv7、128 bit | Implementation Variant、System Implementation Set、Bundle、Authoring Document、Receipt等のProject record |
| runtime `system_id` | `uint32`、0 invalid | 一つのCooked package内だけでdispatchに使う派生番号。永続化、別Package比較、外部API使用を禁止 |

Contract compilerはactive `GameSystemContractRefV1`をMCD IDのUTF-8 byte順に並べ、1から連続するruntime `system_id`を決定論的に割り当て、対応表hashをDependency GraphとNative descriptorへ記録する。Contract ID、version、Contract set hashが同じなら対応表は再生成しても同じになり、一つでも変わればhashを変える。Save／Replayはruntime `system_id`を保存しない。

## 4. 設計原則

### 4.1 自由にするもの

- Game genre、Core loop、Level構成。
- Engine Standard Systemの利用、置換、非利用。
- Project-defined Systemの追加。
- Component、State、GameplayDefinitionのProject Schema。
- Project固有C++ Algorithm。
- 2D、3D、hybrid gameplay space。
- Targetごとの実装Variant。ただし意味同等性Gateを満たすこと。
- 現在の署名済みEngine baselineへ既に含まれるEngine ExtensionとPlatform Adapterの利用。

### 4.2 固定するもの

- MCD Contract ID、version、Contract set hash。
- Authoritative State owner。
- Command、Event、immutable Snapshot。
- Runtime phase、lifetime、authority、budget。
- Save field ID、Replay意味、Migration。
- Dependency edge、Build graph、Target fallback。
- ChangeSet、Review、Promotion、Receipt。
- Test、Diagnostic、failure policy。

### 4.3 Open contract model

Engine Standard System Catalogは便利な開始点であり、利用可能Game Systemの固定Whitelistではない。Catalogは次のSourceを同じ形式で列挙する。

```text
engine_standard
project_defined
engine_extension
```

`engine_extension`は現在の`ImmutableEngineBaselineV1`に署名済みで含まれるCatalog entryの出自を表すだけであり、Game制作中の実装候補または変更権限を表さない。

Project-defined Systemは`game_system.project.<project_namespace>.<lower_snake_path>`を使用する。`project_namespace`はProject作成時に固定するASCII lowercase identifierで、3～32文字、先頭英字、以後英数字またはunderscoreとする。renameしてもSystem IDを変更しない。Engine Standardは`game_system.engine.<lower_snake_path>`、署名済みExtensionは`game_system.extension.<package_namespace>.<lower_snake_path>`を使う。

`lower_snake_path`はdot区切りで、Engine Standardは1～6 segment、Project／Extensionは1～5 segment、各segmentは1～48文字、ASCII lowercase、先頭英字、以後英数字またはunderscoreとする。`package_namespace`も3～48文字で同じidentifier規則を使う。これにより実行可能契約規約の`<kind>.<namespace_path>`における2～8 segment上限を必ず満たす。

表示名、localized title、Genre名をidentityに使わない。Project SystemをEngine Standardへ昇格する場合は新しいEngine IDを発行し、明示Migrationで置換する。同じIDのoriginを変更しない。

### 4.4 Game制作Profileで許可する実装経路

| 順序 | 実装経路 | 変更Risk |
|---:|---|---:|
| 1 | 既存CapabilityとEngine Standard Systemのcomposition | R1／R2 |
| 2 | Project GameplayDefinition | R2 |
| 3 | Project-defined System Contract | Schema互換ならR3、Authority／State意味変更はR4 |
| 4 | NativeGameModule Implementation Variant | R3 |
| 5 | 公開Capability内で実現不能 | `capability_unavailable`で停止 |

前段で表現できないことだけを理由にbounded Project C++を禁止しない。ただし後段を使う場合もPublic System Contract、Target、Budget、Testを省略できない。Game制作TaskからEngine Extension、Engine Adapter、Engine core変更へfallbackしない。

## 5. `GameSystemSpecV1`

`GameSystemSpecV1`はMCD kind `game_system`の正本である。C++ class、GameplayDefinition graph、Editor panel、Source directoryは正本ではない。

### 5.1 Envelope

| Field | 型／規則 |
|---|---|
| MCD共通Envelope | 実行可能契約規約の全Field |
| `system_origin` | `engine_standard \| project_defined \| engine_extension` |
| `semantic_role_ids` | `GameSystemSemanticRoleId`、1～16件。先頭がprimary |
| `responsibility_requirement_ids` | Requirement ID、1～64件 |
| `non_responsibility_requirement_ids` | Requirement ID、0～64件 |
| `runtime_instance_scope` | 5.2節のclosed enum |
| `state_class` | `authoritative \| derived \| presentation_only \| tooling_only` |
| `owned_state_type_refs` | exact MCD Type、0～128件 |
| `read_snapshot_type_refs` | exact MCD Type、0～256件 |
| `accepted_command_type_refs` | exact MCD Type、0～256件 |
| `emitted_event_type_refs` | exact MCD Type、0～256件 |
| `provided_capability_refs` | exact Capability、0～128件 |
| `required_capability_refs` | exact Capability、0～128件 |
| `allowed_phase_ids` | Runtime `TickPhaseId`／UI phase、1～16件 |
| `dependency_edges` | `GameSystemDependencyEdgeV1`、0～128件 |
| `implementation_policy` | `GameSystemImplementationPolicyV1` |
| `save_replay_contract_ref` | authoritative Stateを持つ場合必須 |
| `behavior_budget_refs` | Target Profileごとに1件、1～32件 |
| `authoring_surface_ids` | `natural_language \| form \| table \| graph \| timeline \| source`のsubset |
| `fallback_contract` | Target非対応時の意味同等fallbackまたは`no_fallback`理由 |
| `fixture_ids` | Engine-owned／Project fixture、1～128件 |
| `compatibility_invariant_ids` | Predicate ID、1～128件 |
| `extension_policy` | `sealed \| composable \| replaceable` |

上限を超えるSystemは一つのSystemへ配列上限を追加せず、State ownerと責務が一意になる複数Systemへ分割する。上限変更はContract Schema変更としてR3以上でReviewする。

#### 5.1.1 `GameSystemImplementationPolicyV1`

| Field | 型／規則 |
|---|---|
| `allowed_implementation_kinds` | `gameplay_definition \| native_game_module \| hybrid \| target_specialized_set`の非空subset。`engine_extension`はGame制作時のImplementation Variantではない |
| `default_implementation_ref` | Engine Standardは厳密に1件。Project-definedはQualification前のみ0件可 |
| `native_eligibility` | `not_allowed \| capability_gap_only \| measured_hot_path_or_gap` |
| `replacement_policy` | `not_replaceable \| one_active_variant \| project_override` |
| `live_switch_policy` | `never \| compatible_definition_at_t00`。Nativeは常に`never` |
| `required_equivalence_fixture_ids` | 1～128件 |
| `required_target_profile_ids` | Productが対応を宣言するTarget、1～32件 |
| `configuration_schema_ref` | exact MCD Type、0または1件 |
| `unavailable_behavior` | typed Diagnostic＋fallbackまたはTarget非対応 |

`project_override`でもPublic System Contract、authoritative State、Save field、Replay意味を変更できない。意味変更が必要なら新しいSystem／Type versionとMigrationを作る。

### 5.2 Runtime instance scope

| 値 | Create／Destroy境界 | 例 |
|---|---|---|
| `play_session` | Play開始／停止 | Game flow、global progression projection |
| `world_instance` | Runtime World create／destroy | World rules、global encounter director |
| `level_instance` | Level activation／deactivation | Objective、Level gameplay |
| `encounter_instance` | Encounter start／finish | Wave、boss phase |
| `entity_instance` | Entity spawn／despawn | Character-local Ability owner |
| `ui_session` | UI Runtime開始／終了 | HUD、Map presentation |

一つのSpecは一つのscopeだけを持つ。複数scopeのStateを所有する場合はSystemを分割し、Stable handleまたはtyped Eventで接続する。

### 5.3 Semantic role

`GameSystemSemanticRoleId`は辞書文字列ではなくversion付きCatalog value IDである。これはMCD document kindではなく、MCD Type `type.game_system.semantic_role_id`が検証するfield valueである。初期Engine roleを次に固定する。

```text
game_flow
session
objective
level_gameplay
world_topology
world_generation
character
control
movement
weapon
projectile
combat
damage
vital
targeting
ability
status_effect
ai_behavior
encounter
spawn
score
inventory
equipment
progression
quest
dialogue
ui_flow
hud
map_presentation
camera_direction
```

Projectは`semantic_role.project.<project_namespace>.<name>`をMCD Schema ChangeSetでCatalog valueへ追加できる。新Roleはpositive example、negative example、親となる0～4個のEngine role、質問条件、禁止mappingを必須とする。文字列類似だけで既存Roleへ自動統合しない。

### 5.4 State owner

- Authoritative Typeは同一Contract set内で厳密に一つのactive Game Systemだけが`owned_state_type_refs`へ持つ。
- 他SystemはownerへCommandを送り、immutable SnapshotまたはEventを読む。
- Derived／Presentation Systemはauthoritative Typeを所有できない。
- 同じTypeのFieldごとにownerを分けない。分割が必要ならType自体を分ける。
- Save／Replay対象Stateにowner、version、migration、fault behaviorがなければContract compilerを失敗させる。
- Project-defined replacementがEngine Standard ownerを置換する場合、両方を同時activeにせず、`SystemImplementationSetV1`で一つだけ選択する。

## 6. `GameSystemDependencyEdgeV1`

```text
GameSystemDependencyEdgeV1
  target_system_ref
  edge_kind
  contract_type_refs
  phase_relation
  delivery
  required
  fallback
```

### 6.1 Edge kind

| `edge_kind` | 意味 | Graph規則 |
|---|---|---|
| `build_link` | C++／generated bindingのlink依存 | DAG必須 |
| `cook_input` | Cook時のSource dependency | DAG必須 |
| `snapshot_read` | immutable Snapshotを読む | owner方向を逆転しない |
| `command_target` | ownerへCommandを送る | 同phase再入禁止 |
| `event_subscription` | sealed Eventを受ける | deliveryを明示 |
| `presentation_feed` | authoritative／derivedからPresentationへ値を渡す | Presentationから逆edge禁止 |
| `authoring_reference` | Source document間UUIDv7 `StableId`参照 | dependency closureへ含める |

`delivery`は`same_tick_later_phase \| next_tick \| boundary \| asynchronous_result \| authoring_only`のclosed enumとする。同じtickで互いをCommand targetにするcycle、同じphaseへの再入、callbackによる同期逆呼出しを拒否する。`next_tick` Event cycleは、各edgeのqueue contribution、max latency tick、overflow failureを宣言し、Replay fixtureで決定論を証明した場合だけ許可する。

### 6.2 Dependency graph

`GameSystemDependencyGraphV1`はactive System Spec集合からContract compilerが生成するDerived Artifactであり、手動編集しない。最低限次を含む。

- Contract set hash、Project revision。
- Nodeの`GameSystemContractRefV1`、runtime `system_id`、origin、scope。
- Edge、Type、phase、delivery。
- State owner table。
- Build／Cook topological order。
- Runtime producer canonical order。
- Save／Replay closure。
- Target別active Implementation Variant。

Graph hashをCookedGameplayPackage、NativeGameModule manifest、Build Receipt、Replay headerへ記録する。

### 6.3 `SystemImplementationSetV1`

`SystemImplementationSetV1`はProjectとTargetに対して、各active Public System Contractを厳密に一つのImplementation Variantへ束縛するAuthoring Sourceである。

```text
SystemImplementationSetV1
  implementation_set_id
  project_id
  project_revision
  contract_set_hash
  target_profile_id
  entries[1..4096]
  state_owner_table_hash
  expected_dependency_graph_hash
  fallback_set_ref
```

`implementation_set_id`、`project_id`、`target_profile_id`はUUIDv7 `StableId`である。`fallback_set_ref`はexact `StableId`＋Document revision＋content hashで参照し、表示名またはpathで解決しない。

各Entryは次を持つ。

| Field | 規則 |
|---|---|
| `system_ref` | exact `GameSystemContractRefV1` |
| `implementation_variant_id` | Set内一意のUUIDv7 `StableId` |
| `implementation_kind` | Planのclosed enum |
| `gameplay_definition_set_ref` | Definition／hybridの場合厳密に1件、それ以外0件 |
| `native_module_revision_ref` | Native／hybridの場合厳密に1件、それ以外0件 |
| `configuration_document_ref` | SpecがSchemaを持つ場合厳密に1件 |
| `qualification_receipt_ref` | active化時に厳密に1件 |
| `fallback_variant_ref` | 0または1件。同じPublic Contractだけ |

同じ`system_ref`のEntry重複、active System欠落、未Qualified Variant、Target不一致、Graph hash不一致をCompile errorにする。一つのSystemをTarget内でDefinitionとNativeの二重writerとして登録しない。`fallback_set_ref`は同じProject revisionとContract setに属し、循環してはならない。

## 7. `GameSystemCatalogV1`とAI Discovery

Catalogはactive MCD、Project Profile、Capability Manifest、Qualification Receiptから生成するread-only Projectionである。

### 7.1 Catalog Entry

| Field | 内容 |
|---|---|
| `system_ref` | exact `GameSystemContractRefV1` |
| `origin` | engine／project／extension |
| `semantic_role_ids` | primary＋secondary |
| `title`／`summary` | locale別の短い説明 |
| `maturity` | `unavailable \| research_only \| c0 \| c1 \| c2 \| c3` |
| `target_profile_ids` | Qualification済みTarget |
| `implementation_kinds` | 利用可能Variant |
| `required_capability_ids` | 依存 |
| `budget_summary` | AIが比較できる上限 |
| `failure_summary` | 未対応時のtyped reason |
| `example_ids`／`negative_example_ids` | Golden fixture参照 |
| `contract_set_hash` | stale検出 |

Catalogへ存在することはactive実装を意味しない。`maturity=unavailable`はDiscovery結果から既定除外し、明示include要求時だけ不足理由を返す。未Qualification Targetを成功候補として返さない。

### 7.2 Operation

| Tool／Operation | 種類 | 結果 |
|---|---|---|
| `mirakan.systems.search` | Query | Role、Target、maturity、tagでEntryを検索 |
| `mirakan.systems.read` | Query | 選択SystemのContract、Constraint、Budget、例、非例 |
| `mirakan.systems.plan` | Proposal | `SystemImplementationPlanV1`候補を生成 |
| `mirakan.systems.validate_bundle` | Query／Job | Staging Bundleを検証しDiagnosticを返す |

AIへSystem登録、Source Promotion、Project Commit、Capability Activation Toolを公開しない。新しいProject-defined SystemはSchema／Project ChangeSetとして提案し、GatewayとRisk Gateを通す。

AIは次の順序を守る。

1. GameSpecとProject Profileから必要なsemantic roleを抽出する。
2. `systems.search`でTargetとmaturityを絞る。
3. 選択候補だけ`systems.read`する。
4. 必要なCapabilityだけ`capabilities.search／read`する。
5. 既存composition、Project Definition、bounded Native C++の順に候補を比較し、いずれでも実現不能なら`capability_unavailable`とする。
6. 不足情報がBlocking／High Impactの場合だけ質問する。
7. `SystemImplementationPlanV1`とBundleを提案する。

全System、全Schema、全Backend資料を一つのPromptへ投入しない。AIが未知IDを使った場合はfuzzy修正せず、候補IDと現在Contract hashを持つDiagnosticを返す。

## 8. `SystemImplementationPlanV1`

```text
SystemImplementationPlanV1
  plan_id
  project_revision
  contract_set_hash
  system_ref
  requirement_ids
  target_profile_ids
  candidate_variants[]
  selected_variant
  unmet_capability_ids
  behavior_budget_refs
  benchmark_fixture_ids
  semantic_equivalence_fixture_ids
  save_replay_impact
  build_impact
  risk_class
  assumptions[]
  rejected_alternatives[]
  fallback
  disposition
```

`plan_id`はUUIDv7 `StableId`、`system_ref`はexact `GameSystemContractRefV1`、`requirement_ids`と`unmet_capability_ids`はexact MCD ID＋version、Target／Budget／Fixture参照はUUIDv7 `StableId`＋revision／content hashである。Plan内の表示名または配列indexをidentityに使用しない。

`selected_variant`は次のいずれかとする。

```text
gameplay_definition
native_game_module
hybrid
target_specialized_set
```

`disposition`は`ready_to_stage \| question_required \| capability_unavailable \| budget_missing \| rejected`である。`ready_to_stage`はCommitまたはPromotion承認ではない。

### 8.1 選択順

1. 既存Component、Asset、Engine Standard System、GameplayDefinition、Capabilityのcomposition。
2. Cook、index、flat layout、SoA、batch、Asset layoutの最適化。
3. 表現不能なProject Algorithmまたは計測済みhot pathのNativeGameModule。
4. 公開SDKで意味同等実装ができなければ`capability_unavailable`で停止する。

Genre名、2D／3D、総File数、AIの主観だけでC++を選ばない。

### 8.2 性能判定

`BehaviorBudget`がない場合は`MIRAKAN-SYSTEM-BUDGET_REQUIRED`でBlockする。同じSource revision、input trace、quality、Target Profile、Toolchain、clean process、fixtureを使用し、Runtime規約の継続性能工学loopで比較する。

- P95 timeとpeak memoryが各割当Budgetの80%以下、deadline miss 0、全correctness Gate合格: GameplayDefinitionを維持する。
- 80%超100%以下: DefinitionのCook／index／layoutを最適化し、Native候補と比較する。
- 100%超、deadline miss 1件以上、またはCapabilityで表現不能: Native候補またはhybridを作る。
- Native候補の改善が5%未満または測定noise内: GameplayDefinitionを維持する。

80%と5%は初期Project Policyであり外部Engineの推奨値ではない。Reference hardwareの実測とADRによりSystem別変更を許可するが、AI Task内で閾値を変更しない。正しさ、Gameplay fidelity、visual、startup、hitch、memoryのhard Gateを悪化させた高速化を採用しない。

### 8.3 Target-specialized variant

TargetごとにDefinition／Native実装を変えられるが、次を全Targetで共通にする。

- Public System Contract。
- Command／Event／Snapshot意味。
- Authoritative StateとSave field ID。
- Replayで観測する結果。
- Gameplay fidelity floor。
- Error family。

Target差により意味同等fallbackが存在しない場合、対象Targetを非対応にする。敵数、Damage、collision、goal、spawn timingを黙って変更してTarget対応と表示しない。

### 8.4 C++ onlyとの比較

Shipping CPU executionがC++23である点はC++ only方式と同じである。差は、全Game内容をSourceへ固定するか、調整頻度の高い内容をoffline Cook済みbounded dataとしてC++ evaluatorへ渡すかにある。

| 観点 | 全Game内容をC++ Source化 | 本規約のtyped hybrid |
|---|---|---|
| hot loopの上限性能 | 専用layout、inline、SIMDを直接実装可能 | 同じhot loopをNative Variantへ実装可能 |
| 一般Ruleの実行 | 手書きobject／virtual dispatch次第であり自動的に高速ではない | Cook時にflat array、event index、constant fold、SoA／batchへ変換 |
| 反復速度 | Rule／数値変更でもcompile／link／GameHost restart | Definition変更はvalidate／cook、互換時はT00 swap |
| Code size／build graph | content量に応じSource、template、translation unitが増える | evaluator Codeを共有し、contentはcompact package |
| AI一貫性 | Source、Save、Editor、Testの別生成でdriftしやすい | Contractからbinding、Editor、codec、Test skeletonを同時生成 |
| 自由度 | Engine private／OS APIを含む任意C++まで到達可能 | 公開SDK内のProject C++でGame algorithmを自由化する。Engine変更相当の要求は意図的に未対応停止 |
| 安全性／Target | pointer、phase、Save、fallbackをReviewで維持 | State owner、phase、Budget、Target、Saveを機械Gateで強制 |

Definition evaluatorのindirect dispatch、generic branch、State lookupが計測済みBudgetを超える場合はNativeまたはhybridへ昇格するため、構造化方式のためにhard deadlineを犠牲にしない。逆にC++化だけで高速になるとは仮定せず、同じfixtureで5%以上かつnoiseを超える改善がなければDefinitionを維持する。

したがって公式推奨は「C++を減らす」ことではなく、**Runtimeの共通／hot codeをC++、制作内容をCook済みdata、表現不能または実測hot pathをProject C++**へ置き、両方を同じSystem Contractで交換できるようにすることである。

## 9. `SystemBundleChangeSetV1`

System Bundleは既存のProjectChangeSet、Contract ChangeSet、GameplayDefinitionChangeSet、NativeCodeChangeSet、AssetChangeSetを置き換えない。複数Authorityにまたがる変更をexact hashで結び付けるcoordination envelopeである。

```text
SystemBundleChangeSetV1
  bundle_id
  project_id
  base_project_revision
  base_source_revision
  contract_set_hash
  target_system_refs
  game_system_spec_refs[]
  project_changeset_hashes[]
  gameplay_definition_changeset_hashes[]
  native_code_changeset_hashes[]
  contract_changeset_hashes[]
  asset_changeset_hashes[]
  migration_artifact_hashes[]
  test_fixture_hashes[]
  implementation_plan_hashes[]
  dependency_graph_before_hash
  expected_dependency_graph_after_hash
  required_gate_ids[]
  risk_class
```

`bundle_id`と`project_id`はUUIDv7 `StableId`、`target_system_refs`と`game_system_spec_refs`はexact `GameSystemContractRefV1`、ChangeSet／Artifact／Plan参照はSHA-256 hashである。`required_gate_ids`だけはAI検証規約のMCD Gate IDを使う。

Source本文、Asset binary、巨大JSONをBundleへ埋め込まない。Staging artifactのcontent hashとBroker管理relative pathだけを参照する。

`game_system_spec_refs`または5種類のChangeSet hash arrayの合計は1件以上とし、全参照は同じProject、Contract set、base revisionへ解決する。Native Sourceを含まないBundleは`PromotingSource`と`BuildingTrustedArtifact`を通らず、Review後に`CommittingProject`へ進む。Native Sourceを含むBundleだけが二段階Activationを必須とする。

### 9.1 State machine

```text
Draft
  -> Resolved
  -> Staged
  -> Validating
  -> AwaitingReview
  -> PromotingSource
  -> BuildingTrustedArtifact
  -> CommittingProject
  -> Qualified

各非終端state
  -> FailedBeforeActivation
  -> Superseded

PromotingSource以後の失敗
  -> InactiveSourcePromoted
  -> RetryProjectActivation | RevertProposed
```

`Qualified`だけをactive System implementationとして表示する。Source Promotion済みでもProjectRevisionが参照しないSourceはinactiveであり、GameHost、EditorHost、Shippingへloadしない。

### 9.2 二段階Activation

Source repositoryとAuthoring ProjectRevisionを分散transactionで偽装しない。次の順序へ固定する。

1. Base Project revision、Base Source revision、Contract setをlockする。
2. 全ChangeSetとArtifactをStagingする。
3. Schema、semantic、owner、dependency、Target、budgetを検証する。
4. DefinitionをCookし、Reference evaluator fixtureを実行する。
5. Source WorkerでNative SourceをBuild／Testする。
6. Riskに応じたReview Receiptを得る。
7. Source Promotion Serviceがexact SourceDeltaを昇格し、source revision hashを返す。
8. Trusted Buildが昇格済みSourceからclean artifactを作る。
9. AuthoringCommandGatewayが`RegisterNativeModuleRevision`を含むProjectChangeSetを一revisionとしてCommitする。
10. Project状態をread-backし、Runtime package、Graph、Receiptのhashを照合する。

7の後、9の前に失敗してもSource revisionを削除、reset、force moveしない。inactive Sourceとして記録し、同一hashで再試行するか、別のReview済みrevert commitを提案する。Projectは直前のactive implementationを維持する。

## 10. GameplayDefinition、C++、hybrid

### 10.1 GameplayDefinition向き

- Rule、Event–Condition–Action。
- FSM、bounded Behavior Tree。
- Ability、Status、Cooldown、Cost。
- Quest、Dialogue、Choice。
- Encounter、Spawn Plan、Wave。
- Level Objective、Game flow。
- Item、Weapon、Character、Difficulty、Balance。
- UI flow、Presentation cue。

### 10.2 Native C++向き

- Generic Capabilityで表現できないAlgorithm。
- 大量Entityのbatch kernel。
- SIMD／cache layoutが支配する処理。
- 高頻度Targeting、Damage集計。
- Project固有procedural generation。
- 特殊Character movement。
- deadline missが確認された処理。

### 10.3 Hybrid

Definitionがparameter、state machine、authoring dataを所有し、Native C++がAlgorithmまたはbatch kernelを提供できる。両者は同じCapability ID、Command、Event、Snapshot、Save fieldを使う。DefinitionからC++ function名、pointer、vtable、file pathを参照しない。二つの実装を同時にauthoritative writerとしてactiveにしない。

## 11. Codegen

Contract compilerは`GameSystemSpecV1`から次を決定論的に生成する。

- C++23 System ID、Type ref、Command／Event／Snapshot binding。
- `ComponentAccessManifest`とNativeGameModule System descriptor skeleton。
- TypeScript strict typeとruntime validator。
- Internal JSON Schema 2020-12。
- MCP／Provider projection。
- Editor Form／Graph／Table metadata。
- Game System Catalog Entry。
- Dependency Graph fragment。
- State owner、phase、dependency conformance test。
- Save／Replay field projection。
- Valid／invalid／boundary fixture skeleton。
- Human-readable System reference。

Contract compilerはGameplay Algorithm本文を自動で推測生成しない。AI Source Workerはgenerated public APIとSystem Planを入力にNative Sourceを提案する。Generated bindingとAI生成Sourceを同じArtifact分類にせず、generated fileの直接編集をCIで拒否する。

AIが作るTestだけでAI生成実装を合格させない。Engine-owned invariant、Catalog golden fixture、AI提案のRequirement固有Test、holdout Evalを分離する。

## 12. Runtime規則

- Game SystemはRuntime Orchestratorだけが登録、invoke、停止する。
- State owner以外はauthoritative Stateを直接writeしない。
- World構造変更は`StructuralCommand`、Game State変更はowner Commandへ送る。
- Eventはproducer callback成功後にsealし、canonical orderで配送する。
- Snapshotはimmutableで、consumerから元Stateへwrite-backできない。
- Presentation Systemのvisibility、camera、GPU query、VFX、Audio結果をauthoritative判断へ逆入力しない。
- System callbackは宣言phase、Component access、queue、budgetを超えない。
- System停止後のcallback、job、handle、lease、queue entryを残さない。
- implementation switchはPlay中に任意実行せず、互換なDefinition swapだけT00、Native変更はGameHost restartを基準とする。

## 13. Save、Replay、Migration

- Authoritative Stateを持つSystemは`SaveReplayContractV1`を必須とする。
- Saveは`GameSystemContractRefV1`、State Type／Field ID、Definition revisionを保存する。
- C++ object layout、pointer、function名、Source pathを保存しない。
- Implementation Variant変更はState、Command、Event、Save意味を維持する。
- 意味変更は新System／Type versionと一方向Migrationを必要とする。
- Replay headerへContract set、System dependency graph、active implementation set、Target Profile、RNG stream mappingを記録する。
- Derived cache、Streaming residency、Presentation stateは正規Saveへ保存せず、authoritative Sourceから再生成する。

`SaveReplayContractV1`を次へ固定する。

| Field | 型／規則 |
|---|---|
| `save_replay_contract_id` | UUIDv7 `StableId` |
| `system_ref` | exact `GameSystemContractRefV1`、厳密に1件 |
| `state_type_refs` | Systemのowned authoritative Typeと完全一致、1～128件 |
| `saved_field_refs` | exact Type `McdContractRefV1`＋そのType内で不変の`uint32 field_id`、1～4,096件 |
| `derived_field_refs` | Saveしないが再生成手順を持つField、0～4,096件 |
| `rng_stream_ids` | Replayへseed／draw countを記録するstream、0～256件 |
| `accepted_command_type_refs` | Replay入力として記録するCommand subset |
| `recorded_event_type_refs` | Replay oracleまたは外部結果として記録するEvent subset |
| `checkpoint_boundaries` | `play_start \| level_active \| level_complete \| explicit_checkpoint \| application_inactive`の非空subset |
| `migration_refs` | 旧Contract versionごとの一方向Migrator、0～256件 |
| `unsupported_version_behavior` | `reject_save \| preserve_backup_and_reject` |
| `state_hash_policy_ref` | canonical field順、numeric、RNGを定めるPolicy |

`saved_field_refs`と`derived_field_refs`は重複できない。owned authoritative Fieldがいずれにも分類されないContract、Project C++ object layoutをField IDとして使うContract、MigrationなしでFieldを削除するContractを拒否する。Implementation Variantだけの変更では同じ`SaveReplayContractV1`を維持する。

Migration不能な変更をField defaultまたは空Stateへ黙って変換しない。Project migratorがbackup、旧検証、変換、Diff、新検証、atomic切替を行う。

## 14. Editor／AI UX

System変更はEditorで次の単位にまとめて表示する。

- RequirementとGameSpec上の目的。
- Public System Contract差分。
- State owner／dependency graph差分。
- GameplayDefinition差分。
- C++ Source／Build差分。
- Save／Migration影響。
- Target／Budget／Benchmark。
- Test、Diagnostic、Review、Activation状態。

初心者へC++かDefinitionかを質問しない。AIはゲーム要件を質問し、実装方式をPlanへ記録する。上級者は同じSystem BundleからGraph、Table、Form、Source、Profilerを直接開ける。人間が編集したFieldまたはSource hunkをAIが無条件に再生成しない。

## 15. Diagnosticとfailure

| Code | 条件 | 結果 |
|---|---|---|
| `MIRAKAN-SYSTEM-UNKNOWN_ID` | System／Role ID不明 | 候補とContract hashを返して拒否 |
| `MIRAKAN-SYSTEM-STATE_OWNER_MISSING` | authoritative Stateにownerなし | Contract compile失敗 |
| `MIRAKAN-SYSTEM-MULTIPLE_STATE_OWNERS` | ownerが複数 | Activation拒否 |
| `MIRAKAN-SYSTEM-DEPENDENCY_CYCLE` | Build／Cook／same-tick write cycle | Bundle拒否 |
| `MIRAKAN-SYSTEM-UNDECLARED_EDGE` | Source／Runtime accessがSpec外 | Build／conformance失敗 |
| `MIRAKAN-SYSTEM-PHASE_VIOLATION` | 宣言外phase、same-phase再入 | invoke output破棄、session fault |
| `MIRAKAN-SYSTEM-CONTRACT_MISMATCH` | VariantとPublic Contract不一致 | load／activation拒否 |
| `MIRAKAN-SYSTEM-BUDGET_REQUIRED` | 実装方式判断にBudgetなし | Plan block |
| `MIRAKAN-SYSTEM-VARIANT_UNQUALIFIED` | Target Gate未合格 | 対象Targetで非表示 |
| `MIRAKAN-SYSTEM-SAVE_CONTRACT_MISSING` | authoritative StateにSave契約なし | Bundle拒否 |
| `MIRAKAN-SYSTEM-BUNDLE_STALE` | base revision／contract hash不一致 | 再Resolve要求 |
| `MIRAKAN-SYSTEM-PROMOTION_PARTIAL` | Source昇格後Project未Commit | inactive保持、再試行／revert提案 |
| `MIRAKAN-SYSTEM-CAPABILITY_UNAVAILABLE` | 必須Capabilityなし | 代替候補または明示非対応 |

GatewayまたはAIがDiagnosticを理由に別Systemへ黙ってmappingしない。意味の違うfallbackは人間承認を伴う別GameSpec変更である。

## 16. TestとQualification

### 16.1 Contract

- 全Fieldのvalid／invalid／boundary fixture。
- MCD Contract ID、version、origin、namespace。
- State ownerのexactly-one検査。
- Build／Cook DAGとsame-tick cycle negative test。
- Project-defined Roleのpositive／negative example。
- Provider projectionからInternal validatorへの再検証。

### 16.2 Runtime

- Command／Event／Snapshot conformance。
- Phase、Component access、queue、budget。
- Save／Load／Replay state hash。
- fault、timeout、overflow、cancel、restart。
- Definition implementationとNative implementationのsemantic equivalence。
- Target-specialized VariantのGameplay fidelity。

### 16.3 Source

- Primary／secondary compiler。
- format、warning-as-error、static analysis、ASan。
- unit、property、fuzz、integration。
- undeclared include／import、Module cycle、forbidden API、link import scan。
- clean Buildとartifact／manifest hash。

### 16.4 AI Eval

- 自然言語Requirementから正しいsemantic roleを選ぶ。
- 既存compositionを不要なC++へ昇格しない。
- Capability不足時にProject System／bounded Nativeで解決可能か、`capability_unavailable`かを区別する。
- State owner、dependency、Save、TestをBundleへ含める。
- 未知ID、stale Contract、unsupported Targetを推測補正しない。
- 人間変更を保持した再編集。
- Map／Level／Combat／Characterの責務を混同しない。

`SystemQualificationReceiptV1`はSystem ref、Implementation Variant hash、Graph hash、Target Profile、fixture、correctness、performance、Save／Replay、fault、Review Receiptを結ぶ。ReceiptなしにCatalog maturityまたはactive implementationを昇格しない。

## 17. 初期Engine Standard System

初期CatalogのRoleと実装順を次に固定する。表は固定Class hierarchyではなく、Reference Contractの導入順である。

| Phase | System Contract | 最小責務 |
|---|---|---|
| Phase 0 | Contract fixture only | `game_system` meta-schema、Catalog、owner／graph validator |
| Phase 1 | `game_system.engine.game_flow` | Headless session state、typed transition |
| Phase 3 | `game_system.engine.level_gameplay` | Level entry、objective、completion |
| Phase 3 | `game_system.engine.character` | Character identity、state、movement／combat Port接続 |
| Phase 3 | `game_system.engine.weapon` | trigger、cadence、ammo、reload、Weapon switch、atomic Fire transaction |
| Phase 3 | `game_system.engine.shooter_projectile` | authoritative Projectile spawn、swept query、hit／expire、capacity |
| Phase 3 | `game_system.engine.combat` | Target、Hit Evidence、Team、Damage policy、Damage command |
| Phase 3 | `game_system.engine.vital` | Health、Shield、invulnerability、Damage適用、defeat event |
| Phase 3 | `game_system.engine.pickup` | overlap Evidence、pending grant transaction、one-shot collection |
| Phase 3 | `game_system.engine.score` | score、combo、multiplier、persistent high score |
| Phase 3 | `game_system.engine.ability` | Cost、cooldown、status、typed cue |
| Phase 3 | `game_system.engine.encounter` | Spawn plan、Wave、Boss phase、completion |
| Phase 4 | Project-defined fixture | AIが一つのProject SystemをDefinitionまたはNativeで生成 |
| Phase 6 | 3D variants | 同じContractの3D Physics／Navigation／Animation接続 |
| Phase 8 | Inventory、Progression、Quest、Dialogue、advanced Domain Pack | 個別Qualification後に追加 |

Phase 0で上表のGame実装、空Class、空CMake targetを作らない。Meta-schema、最小fixture、generated binding、negative testだけをFoundation vertical sliceへ含める。

## 18. Definition of Done

- MCDに`game_system` kindとmeta-schemaがある。
- `GameSystemSpecV1`、Implementation Policy、Implementation Set、Catalog、Implementation Plan、Bundle、Dependency Graph、Save／Replay ContractのSchemaとC++／TS projectionがある。
- Engine Standard、Project-defined、署名済みBaseline内のEngine Extensionが同じCatalog形式を使い、後者をGame制作AIが作成・変更できない。
- CatalogがWhitelistではなく、Project namespace登録とActivation Gateを持つ。
- authoritative State ownerが厳密に一つであることをContract compilerが証明する。
- Build／Cook cycle、same-tick write cycle、Presentation逆入力を拒否する。
- AIが`systems.search／read`から必要なSystemだけを取得できる。
- GameplayDefinition、Native、hybrid、Target-specialized Variantが同じPublic Contract conformanceを通る。
- System Bundleの二段階ActivationとSource Promotion後failure recoveryがfault injectionで合格する。
- Save／Replay／MigrationがImplementation Variantから独立している。
- Phase 3のGame Flow、Level、Character、Weapon、Shooter Projectile、Combat、Vital、Score、Ability、EncounterがTitleからResultまで完走する。
- Shooter CoreはWeapon、Projectile、Vital、Pickup、ScoreのState ownerを分離し、Fire／Pickup transaction、Gameplay／Presentation分離、Fire→Hit→Damage→Defeat→Scoreのcausal chainを専用Fixtureで検証する。
- Phase 4で自然言語からSystem Plan、Definition、必要なC++、Test、Bundle、First Playable、再編集を完走する。
- C++化が同一fixture、Budget、noiseを用いた実測で決まり、Genre名またはAI主観で決まらない。
- System変更をRequirement、Contract、Data、Code、Test、Budget、Review単位でEditor表示できる。

## 19. 外部Evidenceと採用判断

調査基準日は2026-07-20である。外部Engineは責務分離と制作上の課題を確認するEvidenceに限定し、API、Object model、既定値、Project formatをMiraikanaiへ模倣しない。

- [Unreal Engine 5.8 Gameplay Framework](https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-framework-in-unreal-engine): GameMode、GameState、PlayerState、Controller、Pawn等を補完するFrameworkとして分離する。Miraikanaiでは固定継承階層ではなくGame System Contractへ一般化する。
- [Unreal Engine 5.8 Gameplay Ability System](https://dev.epicgames.com/documentation/unreal-engine/gameplay-ability-system-for-unreal-engine): Ability、Attribute、Effect、Taskを再利用Frameworkにする。Miraikanaiでは特定Actor／Reflection APIを公開せず、typed CapabilityとDefinitionで所有する。
- [Unreal Engine 5.8 World Partition](https://dev.epicgames.com/documentation/unreal-engine/world-partition-in-unreal-engine): 大規模Worldのpartition、streaming、HLOD、Data Layerを専用責務へ分離する。Miraikanaiでは単一grid実装を公開契約にせず、IntentとTarget別Planへ分ける。
- [Unity 6 Key concepts](https://docs.unity3d.com/Manual/key-concepts.html): GameObject、Component、Sceneによる柔軟なcompositionを確認する。MiraikanaiではProjectごとの暗黙Manager増加を避けるため、State ownerとSystem Catalogを追加する。
- [Unity 6.1 ScriptableObject](https://docs.unity3d.com/6000.1/Documentation/Manual/class-ScriptableObject.html): instanceから独立した共有data assetを確認する。MiraikanaiではGameplayDefinitionとMCDを正本にし、Engine object参照を永続契約へ使わない。
- [Godot stable Scene organization](https://docs.godotengine.org/en/stable/tutorials/best_practices/scene_organization.html): 単一責務、疎結合、依存注入、自己完結sceneの利点を確認する。Miraikanaiでは文書化だけに頼らずGraphとValidatorで強制する。
- [Godot stable Nodes and Scenes](https://docs.godotengine.org/en/stable/getting_started/step_by_step/nodes_and_scenes.html): 小さい部品から再利用可能なsceneをcompositionする自由度を確認する。MiraikanaiではScene aggregationとGame System State ownerを別概念にする。

## 20. 明示的な非採用

- すべてのGameを同じ固定Class hierarchyへ押し込む方式。
- Engine Standard System CatalogをProject System禁止Whitelistにする方式。
- Genreごとの巨大`GameManager`、`MapManager`、`CombatManager`。
- System間でWorld pointer、Container、Vendor objectを共有する方式。
- 一つのauthoritative Stateを複数Systemがwriteする方式。
- 同phase callbackの相互呼出しによる循環。
- `.cpp`だけを生成し、Contract、Save、Test、Budgetを後付けする方式。
- 全GameplayをC++ Sourceへ埋め込むsource-only authoring。
- GameplayDefinitionを任意loop、recursion、eval、FFI付き汎用言語へ拡張する方式。
- Build成功だけでAI生成Sourceをactiveにする方式。
- Source repositoryとProjectRevisionを一つの原子的transactionと偽る方式。
- Target性能のためGameplay意味を黙って削る方式。
- Shipping RuntimeでのC++、Shader、bytecode、binary生成／download／load。
