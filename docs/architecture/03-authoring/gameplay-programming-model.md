# Miraikanai Engine Gameplay Programming Model

- 文書ID: mirakan.arch.gameplay-programming-model
- 状態: review
- 正本範囲: 構造化GameplayとProject C++の選択境界、GameplayDefinition、GameSystemSpecV1、State owner、typed Command／Event／Snapshot Port、Perception／Interaction contract、Project-defined System、AI実装Plan、Contract codegen、SystemBundleChangeSetV1、実装Variantの検証／Promotion
- 非正本範囲: Native ABI／entry／lifecycle／Target link／Build identity／Packaging、Project transaction、共有Schema基盤、Runtime scheduling／共通budget、外部Tool・SDK・Libraryの固定値、Navigation query／Character Motor／Project固有Interaction結果。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core architecture](../02-foundation/core-architecture.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Naming／Project layout](../02-foundation/naming-project-layout.md)、[C++23 modules](../02-foundation/cpp23-modules.md)、[Project state](project-state.md)、[Editor Workspace UX](editor-workspace-ux.md)、[Native game module](native-game-module.md)
- 外部根拠検証日: 2026-07-21

Miraikanai Engineは、CPU上のEngine／Game実行codeをC++とし、調整頻度の高い挙動と内容を検証可能な構造化`GameplayDefinition`としてAuthoringする。Game Systemは契約固定・実装開放型であり、同じ`GameSystemSpecV1`に対してGameplayDefinition、bounded Project C++、hybrid、Target-specialized setを選べる。

自由にするのはGenre、Core loop、System構成、Algorithm、2D／3D表現である。固定するのはPublic System Contract、State owner、typed Port、lifecycle、Save／Replay意味、Target、Budget参照、Test、failure、fallbackである。Game制作でEngine core、Engine Adapter、署名済みExtensionを変更経路に含めない。公開Capabilityで実現不能な要求は`capability_unavailable`で停止する。

## 1. 構造化dataとC++のdecision matrix

### 1.1 公式実装経路

| 要求 | 第一選択 | C++候補へ進む条件 | 禁止 |
|---|---|---|---|
| Rule、FSM、bounded Behavior Tree、Ability、Quest、Encounter、UI flow、調整値 | `GameplayDefinition` | 公開Capabilityで意味を表せない | Genre名だけでC++を選択 |
| 既存System／Component／Assetのcomposition | Engine Standard System＋Definition | Public Contractに必要なProject Algorithmがない | Engine private APIへfallback |
| Project固有algorithm | Project-defined System＋Definitionまたはhybrid | 公開SDK内でのみ実装可能 | Engine Extensionへ自動昇格 |
| 大量Entityのbatch kernel、SIMD／cache layout支配処理 | Definitionを先にCook／index／layout最適化 | 同一fixtureでBudget超過またはdeadline missを確認 | AI主観だけでhot path認定 |
| Platform／Store／Native SDK統合 | `capability_unavailable` | Game制作Taskでは進まない | Project C++から直接利用 |

選択順は次である。

1. CatalogにあるEngine Standard System、Component、Asset、Capabilityのcomposition。
2. Project GameplayDefinition。
3. Project-defined `GameSystemSpecV1`。
4. 公開SDK内のbounded `NativeGameModule` Variant。
5. 意味同等な実装ができなければ`capability_unavailable`。

性能理由の選択は同じSource revision、input trace、Target Profile、Toolchain lock、clean process、correctness fixtureで比較する。共通budget、測定法、capacity envelope、閾値はRuntime Performance Ownerが決定し、本書へ複写しない。Native候補はPublic Contract、Gameplay fidelity、Save／Replay、failure familyを変えてはならない。

### 1.2 C++実行codeと許可例外

Foundation、World、Serialization、Renderer orchestration、Asset Runtime、Simulation、Audio、Input、UI Runtime、VFX、Gameplay evaluator、Editor native shell、Project `NativeGameModule`はC++で実装する。GPU program、Platform bridge、C ABI、Engine外AI Orchestrator、構造化Authoring fileは役割を限定した例外であり、Gameplay authorityを持たない。

Texture、Mesh、Animation、Audio、Font、Scene、Quest、Weapon値、UI配置をC++ sourceへ埋め込まない。Authoring dataはoffline Cookし、Shipping RuntimeはAuthoring JSONをGameplay hot pathでparseしない。Native C++を選んでもSource断片だけを成果物にせず、Contract、Definition参照、Test、Benchmark、Migration、Build receiptをSystem Bundleへ含める。

## 2. `GameplayDefinition`

本節は`GameplayDefinition`のschema責務、bounded実行意味、Cook境界を決める唯一のDomain ownerである。`GameplayDefinition`はGame固有の挙動と内容を表すversion付き構造化Authoring objectであり、任意codeではなく、Engine-owned evaluatorへ渡す有限、typed、権限制約付きdataである。`GameplayDefinitionSet`は同一Project revisionで整合するDefinitionとdependency closure、`CookedGameplayPackage`はそれを検証／正規化／索引化したPlatform ABI非依存のcanonical Runtime binaryである。

初期Definition kindは次を含む。

- Rule／Event–Condition–Action、Finite State Machine。
- bounded Behavior TreeとBlackboard schema。
- Ability、Status Effect、Cooldown、Cost。
- Quest、Objective、Dialogue、Choice。
- Encounter、Spawn Plan、Wave。
- UI Flow、Screen transition、Input action mapping。
- Camera、Audio、VFXのPresentation cue。
- Item、Weapon、Character、Difficulty、Balance table。

kindごとのField単位schemaの正本所在は次で確定する。Perception／Interactionは本書§2.4、Rule／ECAとFSMは§2.5が所有する。[Shooter Genre Pack](../08-packs/shooter.md)はWeapon、Encounter／Spawn、Pickup、Score等を所有せず、対応するFeature Capabilityのcomposition、Profile、Game Flow、Action roleだけを所有する。Combat、Ranged Combat、Encounter／Spawn、Scoring、Pickup／Grantの旧Shooter詳細は[Gameplay Feature Packs](../08-packs/gameplay-features.md)へcontract identityを維持して移管する。残る汎用kindのV1 schemaは本書または参照するFeature Packが所有し、§2.5のPhase期限までにschema revision、validator、Cooker、fixtureを同時登録する。

Definitionが利用できる操作はMCD Capability IDで列挙する。filesystem、network、process、wall clock、pointer、native SDK、dynamic library、arbitrary reflectionを公開しない。

### 2.1 汎用言語化を防ぐbounded contract

- 任意Source text、eval、FFI、function pointerを持たない。
- 任意loop、recursion、自己書換え、runtime node生成を持たない。
- cycleはtick／phase境界を越える明示State transitionとして表す。
- 一つのState Machine instanceは一phaseに最大一回だけauthoritative transitionできる。
- 同一instanceの評価で複数条件が成立した場合はkindごとのschemaが定めるpriorityとStable ID順でcanonicalに選択する。schema未固定kindをauthoring宣言順やcontainer反復順で代用しない。instance間の実行順とCommand競合の解決は本書§4のCommand受付ContractとRuntime Ownerのcanonical順序規則に従い、本書へ複写しない。
- Behavior TreeはDefinitionごとにnode visit上限を持つ。
- Collection、string、blob、Blackboard slot、active task、queue、memoryはMCD上限を必須とする。
- 待機はcall stack／coroutineで保持せず、`GameplayTask { task_id, state_id, wake_tick, bounded_parameters }`へ保存する。
- 非同期処理はversion付きrequest／resultとして扱う。
- World変更はtyped Commandへ追加し、callbackから直接適用しない。

上限のないDefinitionはEditor PreviewとCookで拒否する。現行Shipping RuntimeはAI Definition変更経路自体を持たず、bounded／unboundedを問わず全generation／mutation要求をdeny-only policyで拒否する。Future entryがactive化された後も、本上限検査を通らないDefinitionは拒否する。

### 2.2 Authoring、Cook、Runtime形式

```text
Natural language／Manual Editor
  -> Requirement Resolver
  -> GameplayDefinitionChangeSet
  -> Structural／Semantic／Capability／Budget validation
  -> Preview Diff
  -> Project Commit
  -> Runtime Compiler
  -> Canonicalize／Resolve／Index／Pack
  -> CookedGameplayPackage
  -> Isolated GameHost Preview
```

Editor形式はMCDを正本とし、text-diffable、version付き、unknown field拒否、Stable ID／Requirement ID／Capability ID／Target／Budget参照を持つ。Runtime Compilerは参照解決、string keyのnumeric ID化、event index、flat array、constant fold、未到達検出、capacity上限計算、Capability／State layout／dependency hash、canonical encodingをofflineで行う。

Cooked packageはformat version、Project revision、Contract set hash、Definition set hash、Capability manifest hash、State layout hash、section metadata、flat definition table、event index、constant pool、Stable ID reference、exact Asset dependency、実行上限、Target Profileを持つ。Pointer、vtable、native padding、Source path、Editor説明、Provider情報を保存しない。

### 2.3 State、Save、Replay、live edit

authoritative Definition instance stateはEngine-owned `GameplayStateStore`だけが持つ。Workerはimmutable Definition／Snapshotからprivate outputを作り、成功時にだけstate deltaとCommandをsealする。SaveはDefinition Stable ID、State／Field ID、task、versionを保存し、Source、pointer、C++ object layoutを保存しない。Replayはinput、accepted async result、state delta、Command、fault、RNG mappingをcanonical順で記録する。

Definition setのlive editはdependency closure全体をStagingし、Capability manifest、State layout、Stable State／node IDが互換で、Cook、semantic fixture、deterministic replayが合格した場合だけRuntime Ownerの安全なboundaryでswap候補にする。不一致時は旧versionを維持して`restart_play`を返す。Native hot unloadをDefinition live editへ含めない。

現行Shipping RuntimeはAIによるbounded data変更を含む全generation／mutation、provider network call、generated contentのdownload／loadをdeny-only policyで拒否し、Project／Save／authoritative Worldを不変に保つ。出荷済みCapabilityとSchemaが許可するbounded data変更を検討できるのは、`future.capability.runtime-structured-data-generation`がactive Product Registry、専用Authority／Threat Model、Target binding、fresh Qualificationを成立させ、本書を同じChangeSetで更新した後だけである。その場合もCapability、Schema、Budgetの追加、C++／Shader／bytecode／binaryの生成／download／load、raw Physics／Render／Network stateの確定、process／shell／filesystem／network／FFIは禁止を維持する。

### 2.4 C1 Perception／Interaction

`capability.gameplay.perception`（成熟度C1。maturityはidentityに含めない）は2D／3D共通のboundedな視覚・聴覚認識を、`capability.gameplay.interaction`（同C1）は対象発見、prompt semantic、利用要求、競合制御を所有する。maturity suffixやschema version suffixをcapability IDへ埋め込まず、成熟度はProduct Plan Registryのtierで表す。Behavior Tree、Utility AI、Squad共有Blackboard、door／switch／pickup／inspect／talkのProject固有結果は本Systemへ暗黙に含めない。

```text
PerceptionProfileV1
  profile_id
  dimension: world_2d | world_3d
  sight_range_m
  horizontal_fov_rad
  vertical_fov_rad
  hearing_range_m
  hearing_strength_threshold_q16
  line_of_sight_query_profile_ref
  detectable_channel_mask
  update_interval_ticks
  memory_duration_ticks
  max_candidates_per_observer
  max_visible_targets_per_observer
  target_selection_policy: nearest | highest_priority_then_nearest
  target_priority_field_ref

PerceptionStimulusEventV1
  stimulus_id
  source_entity_ref
  kind: sight_candidate | sound | damage | scripted
  world_position
  strength_q16
  emitted_tick
  channel

PerceptionSnapshotV1
  observer_ref
  snapshot_tick
  visible_targets[]
  heard_stimuli[]
  remembered_targets[]
  query_generation
  overflow_state: closed bitset { none = 0 | candidates | visible_targets | heard_stimuli | memory }

InteractionDefinitionV1
  interaction_id
  semantic_role
  input_action_id
  prompt_message_key
  priority: int16
  max_range_m
  query_shape_ref
  line_of_sight_policy: required | not_required
  concurrency_policy: exclusive | shared
  activation_command_ref
  state_owner_ref
  save_policy: none | owner_state
  accessibility_cue_refs[]

InteractionRequestV1
  request_id
  actor_ref
  target_ref
  interaction_ref
  requested_tick
  focus_snapshot_generation

InteractionSnapshotV1
  actor_ref
  focused_target_ref
  available_interaction_refs[]
  rejection_reason: none | stale_focus_generation | actor_deactivated | target_deactivated | actor_generation_mismatch | target_generation_mismatch | out_of_range | line_of_sight_blocked | game_flow_disallowed | exclusive_lease_conflict | unavailable_state_owner | unknown_interaction | unknown_input_action
  generation
```

Perceptionは距離、FOV、channel filterで候補を先にbounded化し、Collision正本のversion付きRay／Shape QueryだけでLOSを判定する。sight／hearing rangeはfiniteな0～10,000 m、horizontal FOVは0～2π rad、3D vertical FOVは0～π radとし、0 range／0 FOVは該当sense無効を意味する。2Dは`vertical_fov_rad=0`を必須として判定に使用しない。Render visibility、depth buffer、occlusion query、Camera frustum、Particle、Post Processをauthoritative Perceptionへ入力せず、聴覚はAudio mixer実音量ではなくGameplay Systemが発行する`PerceptionStimulusEventV1`だけを使う。`strength_q16`は0～65,535へ正規化した強度（0＝無効、65,535＝当該channelの最大強度）である。聴覚判定は`hearing_range_m`内かつ`strength_q16 >= hearing_strength_threshold_q16`のstimulusだけをheardとし、C1では距離減衰を適用せず発行値をそのまま比較する。

候補とQueryの生成、Query batch処理、結果の正規化は連続する固定tick phaseで行い、各段のphase割当と実行内容は[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)のphase表を正本とする。正規化済みの結果は次tickのGameplayが読む。visible target、heard stimulus、memoryを非決定的に切らず、priority、距離の量子化値、source `StableId`、stimulus IDのcanonical順で残した結果と`overflow_state`を返す。Perception／Interactionのcanonical順に使う距離の量子化値はmm単位へfloorした非負整数とし、浮動小数点比較を順序決定に使わない。`highest_priority_then_nearest`のpriorityは`target_priority_field_ref`が指すtyped Component fieldのexact参照から読み、`target_priority_field_ref`を持たないProfileは`highest_priority_then_nearest`を選択できない。`overflow_state`は同時発生したcandidates、visible targets、heard stimuli、memoryのoverflow bitを組合せられるclosed bitsetであり、`none = 0`だけをzero値のcanonical表現とする。canonical serializationはflagを宣言順のbit位置で符号化し、unknown bitをrejectしてgeneric fallbackへmapしない。C1 reference Profileはobserver当たりcandidates 64、visible targets 16、heard stimuli 16、memory 32、update interval 1～6 ticks、memory 0～600 ticksを許可する。Perception Systemだけがmemory、last confirmed tick、target `StableId`のauthoritative stateを所有し、Save／Replayにはそれらを保存／記録するが、Physics handle、Query result pointer、render objectは保存しない。

Interactionの`max_range_m`はfiniteな0.1～100 mとする。FocusはCollision Ownerが定義する対象発見用の用途別Sensor Profile（`InteractionDefinitionV1.query_shape_ref`で参照）とversion付きQueryを使い、range、LOS、priority降順、距離の量子化値、target `StableId`の順で決定する。当該Sensor Profileのoverlap／query semanticsは[Collision](../05-simulation/collision.md)が所有し、本書はProfile IDを直書きしない。UIは`prompt_message_key`と`accessibility_cue_refs[]`を提示するだけで、localized文字列やpixel hitからWorldを変更しない。keyboard／controller／touchのUse入力は`InteractionRequestV1`となり、Engine Standard Interaction Systemがactor／target generation、range、LOS、Game Flow、exclusive lease、`state_owner_ref`を再検証して登録済みCommandを発行する。door、switch、pickup等の結果は参照先Game Systemが所有し、common Interaction Systemは任意のProject Componentを書き換えない。

stale Query、target deactivate、range外、LOS遮断、exclusive lease競合は`rejection_reason`によるtyped rejectionとし、別targetへ推測で切り替えない。`rejection_reason`はclosed enumであり、stale focus generation、actor／target deactivate、actor／target generation mismatch、range外、LOS遮断、Game Flow不許可、exclusive lease競合、state owner unavailable、unknown interaction／input actionを別値で返す。canonical serializationは宣言したenum値をそのまま符号化し、unknown enum valueをrejectしてgeneric fallbackへmapしない。Focus QueryからUse確定までは最大1 tickだけ許容し、超過Requestは再Queryを要求する。exclusive leaseは確定Commandを発行するtickだけ有効で、継続占有は参照先Game Systemが別のauthoritative stateとして所有する。Saveは`state_owner_ref`のowner stateだけを対象とし、focus、prompt、lease、Physics handleは保存しない。ReplayはRequest、確定Command、overflow_state、rejection_reasonをcanonical serializationのまま記録して値を保持する。C1 fixtureはdoor、switch、collision pickup、explicit-use pickup、inspectを2D／3Dで同じContractへ通し、screen reader labelを含むAccessibility cue、pause、Level deactivate、同tick競合を検証する。

### 2.5 Rule／ECAとFinite State Machine V1

```text
GameplayRuleEcaDefinitionV1
  definition_id
  schema_version: 1
  input_snapshot_type_refs[1..32]
  evaluation_policy: first_match | all_matches
  max_matches: 1..64
  rules[1..256]:
    rule_id
    priority: int16
    condition_ref
    action_command_refs[1..32]

GameplayFiniteStateMachineV1
  definition_id
  schema_version: 1
  initial_state_id
  states[1..256]:
    state_id
    entry_action_command_refs[0..32]
    exit_action_command_refs[0..32]
  transitions[0..1024]:
    transition_id
    source_state_id
    trigger_event_type_ref
    guard_ref?
    priority: int16
    target_state_id
    action_command_refs[0..32]
```

`condition_ref`と`guard_ref`は登録済みのbounded pure predicateだけを参照し、sealed input snapshotとinstance state以外を読まない。Ruleはpriority降順、同値なら`rule_id`のcanonical UTF-8 byte順で評価する。`first_match`は`max_matches=1`を必須とし、`all_matches`も`max_matches`到達後は同じ順で残した結果とoverflow diagnosticを返す。actionは登録済みtyped Command templateだけであり、全matchの出力を一つのprivate batchとして検証後にatomic publishする。

FSMは一instance、一tickにつき最大一transitionである。active stateに属しtriggerとguardが成立した候補をpriority降順、同値なら`transition_id`のcanonical UTF-8 byte順に並べ、先頭だけを選ぶ。`exit actions -> transition actions -> entry actions`を一つのprivate batchとして検証し、成功時だけactive stateとともにatomic publishする。同tickの再帰transition、entry actionからの同期再評価、container順へのfallbackを禁止する。authoritative FSMのactive state、pending bounded task、accepted triggerはSave／ReplayへStable IDで保存し、runtime indexやpointerを保存しない。

残る汎用kindは次のentry gateまでにV1 schemaを本節へ追加する。期限前でもschema revision、validator、Cooker、migration、fixtureが揃わないkindをProject Source、AI Proposal、Cooked packageで使用できない。

| 汎用kind | schema確定期限 |
|---|---|
| Presentation cue | `phase.ai-authoring-mvp-a` entry gate |
| 汎用Balance table | `phase.ai-authoring-mvp-a` entry gate |
| bounded Behavior Tree／Blackboard | `phase.manual-3d-mvp-b` entry gate |
| Ability／Status Effect／Cooldown／Cost | `phase.manual-3d-mvp-b` entry gate |
| Quest／Objective／Dialogue／Choice | `phase.production-capability` entry gate |
| UI Flow／Screen transition／Input action mapping | `phase.production-capability` entry gate |

## 3. `GameSystemSpecV1`

`GameSystemSpecV1`はMCD kind `game_system`の唯一のDomain ownerである。C++ class、Definition graph、Editor panel、Source directoryは正本ではない。

| Field | 型／規則 |
|---|---|
| MCD共通Envelope | [Executable contracts](../02-foundation/executable-contracts.md)の全Field |
| `system_origin` | `engine_standard \| project_defined \| engine_extension` |
| `semantic_role_ids` | versioned role ID、1～16件 |
| `responsibility_requirement_ids` | Requirement ID、1～64件 |
| `non_responsibility_requirement_ids` | Requirement ID、0～64件 |
| `runtime_instance_scope` | closed scope、厳密に1件 |
| `state_class` | `authoritative \| derived \| presentation_only \| tooling_only` |
| `owned_state_type_refs` | exact MCD Type、0～128件 |
| `read_snapshot_type_refs` | exact MCD Type、0～256件 |
| `accepted_command_type_refs` | exact MCD Type、0～256件 |
| `emitted_event_type_refs` | exact MCD Type、0～256件 |
| `provided_capability_refs` | exact Capability、0～128件 |
| `required_capability_refs` | exact Capability、0～128件 |
| `allowed_phase_ids` | Runtime phase ID、1～16件 |
| `dependency_edges` | `GameSystemDependencyEdgeV1`、0～128件 |
| `implementation_policy` | `GameSystemImplementationPolicyV1` |
| `save_replay_contract_ref` | authoritative State ownerでは必須 |
| `behavior_budget_refs` | Target Profileごとのexact参照 |
| `authoring_surface_ids` | natural language／form／table／graph／timeline／sourceのsubset |
| `fallback_contract` | 意味同等fallbackまたは`no_fallback`理由 |
| `fixture_ids` | Engine-owned／Project fixture、1～128件 |
| `compatibility_invariant_ids` | Predicate ID、1～128件 |
| `extension_policy` | `sealed \| composable \| replaceable` |

一つのSpecは`play_session`、`world_instance`、`level_instance`、`encounter_instance`、`entity_instance`、`ui_session`のいずれか一scopeだけを持つ。複数scopeのStateを所有する場合はSystemを分割し、Stable handleまたはtyped Eventで接続する。

`GameSystemImplementationPolicyV1`は許可Implementation kind、default implementation、Native eligibility、replacement policy、live switch policy、equivalence fixture、required Target、configuration schema、unavailable behaviorを持つ。Native live switchは許可しない。Project overrideもPublic Contract、State、Save field、Replay意味を変更できない。

[Executable contracts](../02-foundation/executable-contracts.md#5-mcd共通envelope)が所有する`GameSystemContractRefV1`をPublic Contractの永続identityに使い、Fieldを再定義しない。Implementation Variant／Bundle／ReceiptはUUIDv7 `StableId`、Cooked package内dispatchはderived `uint32 system_id`を使う。runtime `system_id`をSave、Replay、外部APIへ保存しない。

## 4. State ownershipとtyped ports

Authoritative Typeは同一Contract set内で厳密に一つのactive Game Systemだけが所有する。他Systemはownerへtyped Commandを送り、immutable Snapshotまたはsealed Eventを読む。Derived／Presentation Systemはauthoritative Typeを所有できない。一つのTypeをField単位で複数ownerへ分けず、必要ならType自体を分割する。

typed Portは次の意味を持つ。

| Port | 規則 |
|---|---|
| Command | ownerへ変更意図を送る。受付、順序、overflow、failureをContract化 |
| Event | producer callback成功後にsealし、canonical orderで配送 |
| Snapshot | immutableな観測。consumerから元Stateへwrite-back不可 |
| Capability | Engine-owned operation／query。versionとTarget requirementを持つ |
| Asset／Entity handle | generation付きbounded handle。pointer／pathを公開しない |

`GameSystemDependencyEdgeV1`はTarget System ref、edge kind、Contract Type ref、phase relation、delivery、required、fallbackを持つ。edge kindは`build_link`、`cook_input`、`snapshot_read`、`command_target`、`event_subscription`、`presentation_feed`、`authoring_reference`である。

Build／Cook edgeはDAGを必須とする。同じtickのCommand cycle、同phase再入、callback同期逆呼出し、Presentationからauthoritative判断への逆入力を拒否する。next-boundary cycleを許可する場合は各edgeのqueue contribution、latency上限、overflow failure、Replay fixtureを必須とする。Runtime phaseの定義と実行順自体はRuntime Ownerが所有する。

active Spec集合から`GameSystemDependencyGraphV1`をContract compilerが生成する。GraphはContract set hash、Project revision、System ref／derived ID／origin／scope、Edge／Type／phase／delivery、State owner table、Build／Cook order、producer canonical order、Save／Replay closure、Target別active Variantを持つ。手動編集しない。

`SaveReplayContractV1`はSystem ref、owned State Type、saved／derived Field ref、RNG stream、recorded Command／Event、checkpoint、Migration、unsupported version behavior、state hash policyを持つ。owned authoritative Fieldをsavedまたはderivedのどちらにも分類しないContract、C++ layoutをField IDにするContract、MigrationなしでFieldを削除するContractを拒否する。

## 5. Project-defined systems

Engine Standard System Catalogは開始点でありWhitelistではない。Projectは同じ`GameSystemSpecV1`、State owner、typed Port、Target、Save／Replay、Testを満たすProject-defined Systemを登録できる。

| Origin | ID namespace | Game制作時の変更権限 |
|---|---|---|
| Engine Standard | `game_system.engine.<path>` | Contractはread-only。許可時にVariantを置換可能 |
| Project-defined | `game_system.project.<project_namespace>.<path>` | Schema／Risk Gateを通して追加・変更可能 |
| Engine Extension | `game_system.extension.<package_namespace>.<path>` | 署名済みbaseline内だけ。Game制作AIは生成・変更不可 |

ID構文とProject pathは[Naming／Project layout](../02-foundation/naming-project-layout.md)を参照し、本書で別規則を作らない。Display name、localized title、Genre名、Manager／Controller／Service suffixをidentityまたは責務判定に使わない。State owner、semantic role、Requirement、Portが責務を決める。

`SystemImplementationSetV1`はProject／Targetに対して各active Public Contractを厳密に一つのVariantへ束縛するAuthoring sourceである。Project revision、Contract set hash、Target Profile、Entry、State owner table hash、expected Dependency Graph hash、fallback set refを持つ。EntryはSystem ref、Variant ID／kind、Definition set ref、Native module revision ref、configuration ref、Qualification Receipt、same-contract fallbackを持つ。

同じSystemの重複、active System欠落、未Qualified Variant、Target不一致、Graph hash不一致、DefinitionとNativeの二重writerをCompile errorにする。Project-defined replacementがEngine Standard ownerを置換する場合も同時activeにしない。

`GameSystemCatalogV1`はactive MCD、Project Profile、Capability Manifest、Qualification Receiptから生成するread-only projectionである。System ref、origin、semantic role、localized summary、Capability maturity、Qualified Target、Implementation kind、required Capability、Budget summary、failure、positive／negative example、Contract hashを持つ。Catalog掲載とactive実装を混同せず、未Qualified Targetを成功候補として返さない。

## 6. AI planとcode generation

AIはGameSpecからsemantic roleを抽出し、Catalogを検索し、候補Systemだけを読む。必要なCapabilityだけを追加取得し、既存composition、Project Definition、prequalified Pack、bounded Project Sourceの順に比較する。全System、全Schema、全Backend資料を一つのPromptへ投入しない。未知IDをfuzzy補正せず、候補IDとcurrent Contract hashを持つDiagnosticを返す。

`SystemImplementationPlanV1`はPlan ID、Project revision、Contract set hash、System ref、Requirement、Target、authoring profile、candidate Variant、selected Variant、implementation origin、unmet Capability、Behavior Budget ref、Benchmark／equivalence fixture、Save／Replay impact、Build impact、Risk、assumption、rejected alternative、fallback、optional Code owner assignment ref、dispositionを持つ。

`selected_variant`は`gameplay_definition`、`native_game_module`、`hybrid`、`target_specialized_set`のいずれか、`implementation_origin`は`project_definition | prequalified_pack | project_source`のいずれかである。`disposition`は`ready_to_stage`、`awaiting_code_owner`、`question_required`、`capability_unavailable`、`budget_missing`、`rejected`であり、`ready_to_stage`はCommitまたはPromotion承認ではない。

初心者へC++かDefinitionかを質問しない。`authoring_profile=beginner`ではDefinition-firstとし、`implementation_origin`を`project_definition`またはexact Qualification Receiptを持つ`prequalified_pack`に限定する。どちらでも成立しない場合は`capability_unavailable`で停止し、Native／Shaderを暗黙生成しない。`authoring_profile=advanced`で`project_source`を選ぶPlanは、Native moduleなら`role.code_owner.native_module`、Project Shaderなら`role.code_owner.project_shader`を持ち、closed 9-Field subject、exact scope、current Qualification、`revoked_at=null`が成立する`CodeOwnerAssignmentV1`を必須にする。Policy Serviceが信頼済みrevocation registryの署名済みlatest headをread-backし、Assignment Recordとsubject identityのどちらもcurrent snapshotでactiveな場合だけ受理する。Role欠落／unknown、Source kindとのRole不一致、Scope外、期限切れ／失効、snapshot未検証では`awaiting_code_owner`とし、Source Workerを起動しない。AIはGame要件の不足だけを質問し、実装方式はPlanへ根拠付きで記録する。上級者は同じSystem BundleからGraph、Table、Form、Source、Profilerを開く。人間が編集したFieldまたはSource hunkをAIが無条件に再生成しない。

External Agentが同じPlanを提案しても、Host／Model Conformance、Caller Authorization、Project Source Activation、Code owner、G0–G7、Promotionは別Gateである。standard MCPの`StandardExternalProposalReceiptV1`をSource生成、Code owner Approval、Activationへ読み替えない。

Contract compilerは`GameSystemSpecV1`から次を決定論的に生成する。

- C++ System ID、Type ref、Command／Event／Snapshot binding。
- Component access manifestとNative descriptor skeleton。
- TypeScript strict typeとruntime validator。
- Internal JSON SchemaとProvider／MCP projection。
- Editor Form／Graph／Table metadataとCatalog Entry。
- Dependency Graph fragment、State owner／phase conformance test。
- Save／Replay field projectionとfixture skeleton。
- Human-readable System reference。

Contract compilerはGameplay Algorithm本文を推測生成しない。AI Source WorkerがNative Sourceを提案する場合も、generated public bindingと手書き／AI Sourceを別Artifact分類にし、generated fileの直接編集をCIで拒否する。外部Toolchainのexact pinは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)を参照する。

## 7. Generated bundle

`SystemBundleChangeSetV1`は`ProjectChangeSetV1`、GameplayDefinitionChangeSet、NativeCodeChangeSet、Contract ChangeSet、Asset ChangeSetを置き換えず、複数Authorityの変更をexact hashで結ぶcoordination envelopeである。この文書だけがBundle schemaと状態遷移を所有する。

```text
SystemBundleChangeSetV1
  bundle_id
  project_id
  base_project_revision
  base_source_revision
  contract_set_hash
  target_system_refs[]
  game_system_spec_refs[]
  project_changeset_hashes[]
  gameplay_definition_changeset_hashes[]
  native_code_changeset_hashes[]
  project_shader_changeset_hashes[]
  contract_changeset_hashes[]
  asset_changeset_hashes[]
  migration_artifact_hashes[]
  test_fixture_hashes[]
  implementation_plan_hashes[]
  code_owner_assignment_refs[]
  code_owner_approval_refs[]
  dependency_graph_before_hash
  expected_dependency_graph_after_hash
  required_gate_ids[]
  risk_class
```

Bundle／ProjectはUUIDv7 `StableId`、Systemはexact `GameSystemContractRefV1`、ChangeSet／Artifact／PlanはSHA-256で参照する。Source本文、Asset binary、巨大JSONを埋め込まず、Staging artifact hashとBroker管理relative pathだけを参照する。全参照は同じProject、Contract set、base revisionへ解決しなければならない。

```text
Draft -> Resolved -> Staged -> Validating
  -> AwaitingReview                                      [Sourceなし]
  -> AwaitingCodeOwner -> AwaitingReview                 [Native／Shader Sourceあり]
  -> PromotingSource -> BuildingTrustedArtifact
  -> CommittingProject -> Qualified

Source promote完了前の各非終端state -> FailedBeforeActivation | Superseded
Source promote完了後の失敗 -> InactiveSourcePromoted
```

失敗遷移の判定条件は「base Source revisionのpromoteが完了済みか」だけである。`PromotingSource`中にpromoteが完了しないまま失敗した場合は`FailedBeforeActivation`へ、promote完了後の失敗（`BuildingTrustedArtifact`、`CommittingProject`を含む）は`InactiveSourcePromoted`だけへ遷移し、二つの規則を同じ失敗へ重複適用しない。`RetryProjectActivation`と`RevertProposed`はstateではなく`InactiveSourcePromoted`からの遷移actionであり、前者は同一hashで`CommittingProject`へ再入し、後者はReview済みrevertを別のBundleとして提案する。

Native／Project Shader Sourceを含まないBundleは`AwaitingCodeOwner`、Source promotion／trusted buildを通らずReview後にProject Commitへ進む。Sourceを含むBundleは、Source kindに一致するexact `role_ref`、Scope、Qualification、active期間、`revoked_at=null`を持つAssignmentがなければ`AwaitingCodeOwner`で停止し、全Source ChangeSetのexact DiffとSource revisionへ`CodeOwnerApprovalV1.decision=approved`が揃った後だけ`AwaitingReview`へ進む。Sourceを含むBundleだけが二段階Activationを必須とする。`Qualified`だけをactive Implementationとして表示し、Source promotion済みでもProject revisionが参照しないSourceをGameHost、EditorHost、Shippingへloadしない。

Source repositoryとProject revisionを一つのatomic transactionと偽装しない。Source promotion後にProject Commitが失敗してもSource revisionをdelete、reset、force-moveせずinactiveとして記録し、同一hashで再試行するかReview済みrevertを別commitとして提案する。Projectは直前のactive implementationを維持する。

## 8. ValidationとBuild

Definition／prequalified Pack経路はSchema、semantic、Capability、State owner、dependency、Target、Budgetを検証し、canonical CookとReference evaluator fixtureを実行する。Packはexact package hash、license、Target、Qualification Receipt、revocationを検証し、変更を加えた時点でprequalified扱いを失う。Native経路は同じPublic Contract conformanceに加え、[Native game module](native-game-module.md)が所有するSource境界、C ABI、entry、lifecycle、Target link、Build identity、Preview、Promotion、Package gateを通す。Project Shader経路はShader OwnerのSource／Technique Gateに加え、Nativeと同じCode owner binding原則を使う。

本書はNative ABI field、entry symbol、function table、memory portのbinary layout、compiler／generator、Target別link方式、Build artifact identityを定義しない。これらをGameplayProgrammingModelまたはGenerated Bundleへ複写せず、Native module revision refとReceipt hashだけを持つ。

共通validationは次を含む。

- MCD Envelope、ID／version／Contract set、unknown field拒否。
- State owner exactly-one、typed Port、Build／Cook DAG、same-boundary cycle。
- Component access、Capability、Target、Budget ref、fallback。
- Save／Replay／Migration closureとState hash policy。
- Definition／Native／hybridのsemantic equivalence。
- generated bindingとSource dependency manifestの一致。
- stale base revision、Graph hash、Toolchain lockの拒否。

Native／Project Shader buildが成功しただけでactiveにしない。Project Sourceは信頼済みProcessまたはGPU programで実行されるcodeであり、[AI Security／Approval](../01-governance/ai-security-approval.md)のRisk、Code owner assignment／approval、Review、Promotion authorizationと、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)のEvidence／Receiptを満たす。

## 9. Testingとpromotion

### 9.1 Contract／Runtime／Source／AI fixture

- 全Fieldのvalid／invalid／boundary fixtureとMCD projection整合。
- State owner exactly-one、dependency cycle、undeclared edgeのnegative test。
- Command／Event／Snapshot、phase access、queue、budget conformance。
- transition／Rule／Behavior Tree nodeのauthoring宣言順first-match選択をReference evaluatorで検証する順序fixture。
- Save／Load／Replay state hash、fault、overflow、cancel、restart、Migration。
- DefinitionとNativeのsemantic equivalence、Target-specialized VariantのGameplay fidelity。
- Source format、warning、static analysis、sanitizer、unit、property、fuzz、integration、forbidden API／dependency scanはNative OwnerのGateを参照。
- AIが既存compositionを不要なC++へ昇格せず、Capability不足とbounded Native候補を区別するEval。
- 未知ID、stale Contract、unsupported Target、人間変更を推測補正しないEval。
- Beginner Planの`implementation_origin`が`project_definition | prequalified_pack`だけであり、新規Native／Shader Source Taskが0件になるfixture。
- Code owner Assignment不在、missing／unknown `role_ref`、Native↔Shader wrong-scope Role、期限切れ、`revoked_at` Field省略、non-null `revoked_at`、unknown extra Field、current snapshotのAssignment／subject revoke、current snapshotのmissing／stale／invalidと、別Diff／Source revision／Build ReceiptのApprovalを一原因ずつ拒否し、`revoked_at=null`だけで`awaiting_code_owner`からPromotionへ直行しないfixture。
- External Agentの`StandardExternalProposalReceiptV1`だけでProject Source Activation、Code owner Approval、Promotionを通過できないfixture。

`SystemQualificationReceiptV1`はSystem ref、Variant hash、Dependency Graph hash、Target Profile、fixture、correctness、performance、Save／Replay、fault、Review Receiptを結ぶ。ReceiptなしにCatalog maturityまたはactive implementationを昇格しない。

### 9.2 Promotionとfailure recovery

System Bundleはbase Project revision、base Source revision、Contract setをlockし、全ChangeSet／ArtifactをStagingしてvalidation、Cook、Test、Reviewを終える。Native／Project Shader Sourceがある場合はCode owner Approval、Source promotion、clean trusted build、該当Project Source revision登録の`ProjectChangeSetV1` Commit、read-back verificationの順に進む。

Project Commitは[Project state](project-state.md)のtransactionを使う。Definition、Source、Asset、Migration、Testの一部だけをactiveにしない。Graph、Contract set、Implementation set、Cooked package、Native revision、Receiptのhashが一致しなければ旧Implementationを維持する。

| Failure | 結果 |
|---|---|
| Definition schema／semantic不正 | Commitなし、field Diagnostic |
| Capability不足 | `capability_unavailable`、Engine変更へ昇格しない |
| State ownerなし／複数 | Contract compile失敗 |
| dependency／same-boundary cycle | Bundle拒否 |
| Save contract欠落 | Bundle拒否 |
| stale revision／Contract hash | 再Resolve要求 |
| Code owner不在／Role欠落・unknown・Source kind不一致／失効／Scope外 | `AwaitingCodeOwner`、Source WorkerまたはPromotionを停止 |
| Code owner ApprovalのDiff／Source revision／Build Receipt不一致 | Bundle拒否、Sourceはinactive Staging |
| Cook／Build／Test失敗 | active implementation不変 |
| Source promotion後Project失敗 | inactive Source保持、retry／revert提案 |
| Target Variant未Qualified | 対象Targetで非表示 |
| runtime Port／phase違反 | output破棄、typed session fault |

Capabilityの導入順、Game System一覧の製品Phase、MVP、Pack maturityは[Product Plan](../00-product/product-plan.md)と各Pack Ownerが所有する。本書は空Class群、固定Phase表、Genre固有System一覧を実装開始条件として複写しない。

## 10. 禁止する固定階層とunrestricted scripting runtime

明示的に採用しないものは次である。

- すべてのGameを同じ固定Class hierarchyへ押し込む方式。
- Engine Standard CatalogをProject System禁止Whitelistにする方式。
- Genreごとの巨大`GameManager`、`MapManager`、`CombatManager`。
- Entityごとのvirtual `Update()`、個別heap object、文字列dispatchを標準方式にすること。
- System間でWorld pointer、Container、Vendor objectを共有すること。
- 一つのauthoritative Stateを複数Systemがwriteすること。
- 同phase callbackの同期相互呼出しと循環。
- `.cpp`だけを生成し、Contract、Save、Test、Budgetを後付けすること。
- GameplayDefinitionを任意loop、recursion、eval、FFI、self-modification付き言語へ拡張すること。
- Luau、Lua、C#等のGame scripting runtime、汎用VM、bytecode interpreter、JIT、Game向けFFI。
- `ScriptRuntime`、`ScriptStateStore`、言語別Capability bindingの先回り抽象。
- Shipping RuntimeでのC++、Shader、bytecode、binary生成／download／compile／dynamic load。
- Build成功だけでAI生成Sourceをactiveにすること。
- Target性能のためGameplay意味、enemy数、Damage、goal、spawn timingを黙って変えること。
- Source repositoryとProject revisionを一つのatomic transactionと偽ること。

置換後の正規概念は`GameplayDefinition`、`CookedGameplayPackage`、`GameplayStateStore`、`GameSystemSpecV1`、`SystemBundleChangeSetV1`、`NativeGameModule`である。将来unrestricted scriptingを検討する場合は、本設計の暗黙拡張ではなく、新しいProduct decision、Threat Model、Memory／Performance、Editor／Debugger、全Target gateを必須とする。現行Capabilityとしては`not_activated`であり、Runtime interfaceを実装しない。
