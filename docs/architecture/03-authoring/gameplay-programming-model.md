# Miraikanai Engine Gameplay Programming Model

- 文書ID: mirakan.arch.gameplay-programming-model
- 状態: review
- 正本範囲: 構造化GameplayとProject C++の選択境界、GameplayDefinition、GameSystemSpecV2、State owner、typed Command／Event／Snapshot Port、Perception／Interaction contract、Interaction Space Semantic Registry、Project-defined System、AI実装Plan、Contract codegen、SystemBundleChangeSetV1、実装Variantの検証／Promotion
- 非正本範囲: Native ABI／entry／lifecycle／Target link／Build identity／Packaging、Project transaction、共有Schema基盤、Runtime scheduling／共通budget、外部Tool・SDK・Libraryの固定値、Navigation query／Character Motor／Project固有Interaction結果。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core architecture](../02-foundation/core-architecture.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Naming／Project layout](../02-foundation/naming-project-layout.md)、[C++23 modules](../02-foundation/cpp23-modules.md)、[Project state](project-state.md)、[Editor Workspace UX](editor-workspace-ux.md)、[Native game module](native-game-module.md)
- 外部根拠検証日: 2026-07-21

Miraikanai Engineは、CPU上のEngine／Game実行codeをC++とし、調整頻度の高い挙動と内容を検証可能な構造化`GameplayDefinition`としてAuthoringする。Game Systemは契約固定・実装開放型であり、同じ`GameSystemSpecV2`に対してGameplayDefinition、bounded Project C++、hybrid、Target-specialized setを選べる。

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
3. Project-defined `GameSystemSpecV2`。
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
- cycleはSimulation Advance／phase境界を越える明示State transitionとして表す。
- 一つのState Machine instanceは一phaseに最大一回だけauthoritative transitionできる。
- 同一instanceの評価で複数条件が成立した場合はkindごとのschemaが定めるpriorityとStable ID順でcanonicalに選択する。schema未固定kindをauthoring宣言順やcontainer反復順で代用しない。instance間の実行順とCommand競合の解決は本書§4のCommand受付ContractとRuntime Ownerのcanonical順序規則に従い、本書へ複写しない。
- Behavior TreeはDefinitionごとにnode visit上限を持つ。
- Collection、string、blob、Blackboard slot、active task、queue、memoryはMCD上限を必須とする。
- 待機はcall stack／coroutineで保持せず、`GameplayTask { task_id, state_id, wake_advance_sequence, bounded_parameters }`へ保存する。
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
  simulation_cadence_profile_ref: SimulationCadenceProfileRefV1
  dimension: world_2d | world_3d
  sight_range_m
  horizontal_fov_rad
  vertical_fov_rad
  hearing_range_m
  hearing_strength_threshold_q16
  line_of_sight_query_profile_ref
  detectable_channel_mask
  update_interval_advances
  memory_duration_advances
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
  emitted_advance_sequence
  channel

PerceptionSnapshotV1
  observer_ref
  simulation_cadence_profile_ref: SimulationCadenceProfileRefV1
  simulation_advance_interval_hash: SHA-256
  snapshot_advance_sequence
  visible_targets[]
  heard_stimuli[]
  remembered_targets[]
  query_generation
  overflow_state: closed bitset { none = 0 | candidates | visible_targets | heard_stimuli | memory }

InteractionSpaceSemanticRefV1
  semantic_id: namespace付きStableId
  semantic_version: positive uint32
  semantic_content_hash: SHA-256

InteractionSpaceSemanticContributionV1
  semantic_ref: InteractionSpaceSemanticRefV1
  owner_ref: exact {owner_id, owner_revision, owner_content_hash}
  contribution_layer: feature_pack | genre_pack | project
  binding_kind: spatial | logical | ui
  semantic_description
  search_synonyms[0..32]
  binding_schema_ref: exact McdContractRefV1(kind=type)
  executor_binding_refs[1..16]: exact GameSystemContractRefV1
  required_capability_refs[0..32]: exact McdContractRefV1(kind=capability)
  target_support_refs[1..16]: exact Target Profile entry ref/version/hash

InteractionSpaceSemanticRegistryV1
  registry_id: registry.feature.interaction.space_semantic
  registry_version: positive uint32
  registry_owner_ref: exact {owner_id, owner_revision, owner_content_hash}
  contributions[3..1024]
  registry_content_hash: SHA-256

InteractionSpaceSemanticActivationProjectionV1
  projection_id
  projection_version: positive uint32
  registry_ref: exact {registry_id, registry_version, registry_content_hash}
  selected_semantic_refs[1..1024]: InteractionSpaceSemanticRefV1
  qualification_binding_refs[1..1024]:
    exact {binding_id, binding_version, binding_content_hash}
  projection_content_hash: SHA-256

SpatialInteractionBindingV1
  actor_subject_type_ref: exact McdContractRefV1(kind=type)
  target_subject_type_ref: exact McdContractRefV1(kind=type)
  max_range_m
  query_shape_ref: exact Collision Sensor Profile ref
  line_of_sight_policy: required | not_required

LogicalInteractionBindingV1
  actor_subject_type_ref: exact McdContractRefV1(kind=type)
  target_subject_type_ref: exact McdContractRefV1(kind=type)
  selection_policy_ref: exact McdContractRefV1(kind=policy)

UiInteractionBindingV1
  actor_subject_type_ref: exact McdContractRefV1(kind=type)
  control_subject_type_ref: exact McdContractRefV1(kind=type)
  focus_context_policy_ref: exact McdContractRefV1(kind=policy)

InteractionSpaceBindingV1
  binding_kind: spatial | logical | ui
  payload:
    SpatialInteractionBindingV1
    | LogicalInteractionBindingV1
    | UiInteractionBindingV1

InteractionDefinitionV1
  interaction_id
  semantic_role
  interaction_space_registry_ref:
    exact {registry_id, registry_version, registry_content_hash}
  interaction_space_activation_projection_ref:
    exact {projection_id, projection_version, projection_content_hash}
  interaction_space_ref: InteractionSpaceSemanticRefV1
  space_binding: InteractionSpaceBindingV1
  input_action_id
  prompt_message_key
  priority: int16
  concurrency_policy: exclusive | shared
  activation_command_ref
  state_owner_ref
  operation_eligibility_policy_ref: optional exact policy ref
  save_policy: none | owner_state
  accessibility_cue_refs[]

InteractionRequestV1
  request_id
  actor_ref
  target_ref
  interaction_ref
  interaction_space_ref: InteractionSpaceSemanticRefV1
  requested_advance_sequence
  focus_snapshot_generation

InteractionSnapshotV1
  actor_ref
  focused_target_ref
  interaction_space_ref: InteractionSpaceSemanticRefV1
  available_interaction_refs[]
  rejection_reason: none | stale_focus_generation | actor_deactivated | target_deactivated | actor_generation_mismatch | target_generation_mismatch | out_of_range | line_of_sight_blocked | policy_denied | exclusive_lease_conflict | unavailable_state_owner | unknown_interaction | unknown_input_action
  generation
```

Perceptionは距離、FOV、channel filterで候補を先にbounded化し、Collision正本のversion付きRay／Shape QueryだけでLOSを判定する。sight／hearing rangeはfiniteな0～10,000 m、horizontal FOVは0～2π rad、3D vertical FOVは0～π radとし、0 range／0 FOVは該当sense無効を意味する。2Dは`vertical_fov_rad=0`を必須として判定に使用しない。Render visibility、depth buffer、occlusion query、Camera frustum、Particle、Post Processをauthoritative Perceptionへ入力せず、聴覚はAudio mixer実音量ではなくGameplay Systemが発行する`PerceptionStimulusEventV1`だけを使う。`strength_q16`は0～65,535へ正規化した強度（0＝無効、65,535＝当該channelの最大強度）である。聴覚判定は`hearing_range_m`内かつ`strength_q16 >= hearing_strength_threshold_q16`のstimulusだけをheardとし、C1では距離減衰を適用せず発行値をそのまま比較する。

候補とQueryの生成、Query batch処理、結果の正規化は連続する一回のSimulation Advance phaseで行い、各段のphase割当と実行内容は[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)のphase表を正本とする。正規化済みの結果は次advanceのGameplayが読む。visible target、heard stimulus、memoryを非決定的に切らず、priority、距離の量子化値、source `StableId`、stimulus IDのcanonical順で残した結果と`overflow_state`を返す。Perception／Interactionのcanonical順に使う距離の量子化値はmm単位へfloorした非負整数とし、浮動小数点比較を順序決定に使わない。`highest_priority_then_nearest`のpriorityは`target_priority_field_ref`が指すtyped Component fieldのexact参照から読み、`target_priority_field_ref`を持たないProfileは`highest_priority_then_nearest`を選択できない。`overflow_state`は同時発生したcandidates、visible targets、heard stimuli、memoryのoverflow bitを組合せられるclosed bitsetであり、`none = 0`だけをzero値のcanonical表現とする。canonical serializationはflagを宣言順のbit位置で符号化し、unknown bitをrejectしてgeneric fallbackへmapしない。C1 reference Profileはobserver当たりcandidates 64、visible targets 16、heard stimuli 16、memory 32、update interval 1～6 advances、memory 0～600 advancesを許可する。Profile／Snapshot／Save／Replayは同じCadence Profile ref／hashを持ち、別rateへadvance countを換算しない。Perception Systemだけがmemory、last confirmed advance sequence、target `StableId`のauthoritative stateを所有し、Save／Replayにはそれらを保存／記録するが、Physics handle、Query result pointer、render objectは保存しない。

`InteractionSpaceSemanticRegistryV1`は`feature.interaction@1`を選択したProjectのdependency closureからmaterializeするFeature-owned Registryであり、`registry_owner_ref`はPack Manifestが閉じるexact `owner.feature.interaction` revision／content hashである。Generic Engine Coreの必須ContractまたはCore→Feature dependencyではない。初期必須Contributionは次のexact三件で、Registryから除去できない。Activation ProjectionはProject／Targetの成立Capabilityに応じて一件以上を選び、Collisionなしのlogical／ui Projectへspatialを偽装または強制しない。

| semantic ref | exact owner／layer | binding kind／schema ref | executor binding | required Capability | Target support |
|---|---|---|---|---|---|
| `interaction.space.spatial@1` | `owner.feature.interaction`／`feature_pack` | `spatial`／`{type.feature.interaction.spatial_binding, 1, contract_set_hash}` | exact `GameSystemContractRefV1(game_system.extension.feature.interaction.standard@1)` | `[capability.simulation.collision]` | `[target.windows.desktop, target.android.mobile, target.apple.mobile]` |
| `interaction.space.logical@1` | `owner.feature.interaction`／`feature_pack` | `logical`／`{type.feature.interaction.logical_binding, 1, contract_set_hash}` | exact `GameSystemContractRefV1(game_system.extension.feature.interaction.standard@1)` | `[]` | `[target.windows.desktop, target.android.mobile, target.apple.mobile]` |
| `interaction.space.ui@1` | `owner.feature.interaction`／`feature_pack` | `ui`／`{type.feature.interaction.ui_binding, 1, contract_set_hash}` | exact `GameSystemContractRefV1(game_system.extension.feature.interaction.standard@1)` | `[]` | `[target.windows.desktop, target.android.mobile, target.apple.mobile]` |

表のownerは表示上IDだけを示すが、保存値はPack identityから解決したrevision／content hash付きexact refである。System ref、Type ref、Capability ref、Target refも各typed refのversion／hashを省略せず保存する。`game_system.extension.feature.interaction.standard`は`owner_layer=feature_pack`、exact `owner.feature.interaction`、`runtime_scope_type_ref=scope.core.runtime_session`を持つ共通executorで、World／Collisionはspatial Contributionが選択された時だけPublic Port経由のdependencyとなる。UI semanticはUI toolkit、pixel、rendererをauthorityにせず、登録済みcontrol subjectから同じRequestへ変換できるため、`capability.platform.ui-core`をInteraction Packの必須依存にしない。

Feature／Genre／Projectは自己namespaceへContributionを追加できるが、`contribution_layer=core`またはFixture layer、他Owner namespace、初期三件の上書き、同じsemantic ID／versionの別hashを拒否する。semantic名だけで新しいexecutor、権限、side effectを作れない。

`semantic_content_hash`はASCII `MIRAKAN_INTERACTION_SPACE_SEMANTIC_CONTRIBUTION_V1`と自己hashを除くReceipt-free Contribution canonical bytes、Registry hashはASCII `MIRAKAN_INTERACTION_SPACE_SEMANTIC_REGISTRY_V1`、Registry ID／version、Registry owner ref、semantic ID／version順の全Contribution canonical bytesから計算する。selected refsはsemantic ID／version／hash順、Binding refsは解決したsubject refの同じ順へstrict sortし、duplicateを拒否する。Activation Projectionのselected ref集合と合格かつfreshなQualification Binding subject集合はexact set equalityで、Receipt／BindingをContribution／Registry hashへ戻さない。bare ID、owner／layer／namespace不一致、unknown／stale version／hash、未Qualification、Target／Capability不成立、Registry外またはProjection非選択refをfail closedにする。

`InteractionSpaceBindingV1`は`binding_kind`をdiscriminatorとするclosed tagged unionで、対応するpayloadだけをexact一件持つ。`binding_kind`、active Registryで解決したContributionの`binding_kind`、`binding_schema_ref`、実payload schemaはすべてbyte-exactに一致しなければならない。`spatial`だけが`SpatialInteractionBindingV1`のrange、Collision Sensor Profile、LOSを持てる。`logical`と`ui`へdummy range、無限range、偽World entity、空Query Profile、`line_of_sight_policy=not_required`を保存して空間契約へ近似することを禁止し、unknown fieldとして拒否する。`logical`はowner-typed subjectとpure selection policy、`ui`はowner-typed control subjectとfocus-context policyだけで対象を決め、いずれもCollision、World position、Render visibility、pixel hitをauthorityにしない。`InteractionRequestV1`と`InteractionSnapshotV1`の`interaction_space_ref`はDefinition、Registry、Activation Projectionとexact equalityで、actor／targetは選択branchのsubject typeへ適合しなければならない。`focused_target_ref`と`focus_snapshot_generation`の`focus`は当該space内の選択tokenを意味し、spatial queryまたはUI pixel focusを暗黙に意味しない。

`spatial`の`max_range_m`はfiniteな0.1～100 mとする。Spatial FocusはCollision Ownerが定義する対象発見用の用途別Sensor Profile（`InteractionDefinitionV1.space_binding.payload.query_shape_ref`で参照）とversion付きQueryを使い、range、LOS、priority降順、距離の量子化値、target `StableId`の順で決定する。当該Sensor Profileのoverlap／query semanticsは[Collision](../05-simulation/collision.md)が所有し、本書はProfile IDを直書きしない。UIは`prompt_message_key`と`accessibility_cue_refs[]`を提示するだけで、localized文字列やpixel hitからWorldを変更しない。keyboard／controller／touchのUse入力は`InteractionRequestV1`となり、Engine Standard Interaction Systemがactor／target generation、space binding、optional `operation_eligibility_policy_ref`、exclusive lease、`state_owner_ref`を再検証して登録済みCommandを発行する。range／LOSの再検証は`spatial`だけ、selection／focus-context policyの再検証は対応する`logical`／`ui`だけに行う。policyがないneutral Interactionは追加FeatureまたはGenre stateを要求せず、policy ownerの拒否だけをgeneric `policy_denied`で返す。door、switch、pickup等の結果は参照先Game Systemが所有し、common Interaction Systemは任意のProject Componentを書き換えない。

stale selection、target deactivate、policy拒否、exclusive lease競合は全branchでtyped rejectionとし、range外とLOS遮断は`spatial`だけが返せる。`logical`／`ui`が`out_of_range | line_of_sight_blocked`を返した場合は結果を受理せずContract violationにする。`rejection_reason`はclosed enumであり、stale focus generation、actor／target deactivate、actor／target generation mismatch、range外、LOS遮断、generic policy denial、exclusive lease競合、state owner unavailable、unknown interaction／input actionを別値で返す。canonical serializationは宣言したenum値をそのまま符号化し、unknown enum valueをrejectしてgeneric fallbackへmapしない。選択SnapshotからUse確定までは最大1 Simulation Advanceだけ許容し、超過Requestは同じspaceで再選択を要求する。exclusive leaseは確定Commandを発行するadvanceだけ有効で、継続占有は参照先Game Systemが別のauthoritative stateとして所有する。Saveは`state_owner_ref`のowner stateだけを対象とし、focus、prompt、lease、Physics handleは保存しない。ReplayはRequest、確定Command、overflow_state、interaction space、rejection reasonをcanonical serializationのまま記録して値を保持する。

C1 fixtureでは既存のdoor、switch、collision pickup、explicit-use pickup、inspectを2D／3Dとも`interaction.space.spatial@1`＋`SpatialInteractionBindingV1`へ明示的に束縛する。加えてWorld／Collisionなしのinventory commandまたはturn-based selectionを`interaction.space.logical@1`、headless UI event sourceでも再生できるmenu／dialogue control actionを`interaction.space.ui@1`へ束縛し、三branchのDefinition／Request／Snapshot／Replay round-trip、screen reader labelを含むAccessibility cue、pause、owner scope deactivate、同一advance競合を検証する。branch／semantic／schema mismatch、logical／uiへのrange／LOS／Query field混入、spatialのrange／Query欠落、bare semantic ID、owner／projection／hash stale、UI pixel座標からのauthoritative Commandを一原因ずつrejectする。Shooter Game Flow eligibility policyはShooter Packだけが提供し、common Interaction manifest、Registry初期三件、System Graphへ依存辺を追加しない。

従来草案のflat `max_range_m`／`query_shape_ref`／`line_of_sight_policy`表現はactive Contract Setへ登録された証拠がなく、新Sourceのcompatibility aliasとして受理しない。実在する旧bytesが後からinventoryで証明された場合は、source schema bytes／Owner／Definition hashを束縛した承認済みversioned migrationを別途activateし、全recordを`interaction.space.spatial@1`＋`SpatialInteractionBindingV1`へ明示変換する。名前またはField存在だけでlogical／uiへ推測変換しない。

`InteractionSnapshotV1@1.rejection_reason=game_flow_disallowed`は`InteractionSnapshotV1@2.rejection_reason=policy_denied`へversioned clean migrationする。旧値を@2 aliasとしてdeserializeせず、migrationは登録済みeligibility policy ref、owner、schema hashを要求する。policy ownerを一意に解決できないSource／Save／Replayはmigrationを拒否してlast-valid recordを維持する。fixtureは@1→@2 round-trip、policyなしneutral Interaction、Shooter policyありdeny、@2への旧enum直接入力拒否を検証する。

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

FSMは一instance、一Simulation Advanceにつき最大一transitionである。active stateに属しtriggerとguardが成立した候補をpriority降順、同値なら`transition_id`のcanonical UTF-8 byte順に並べ、先頭だけを選ぶ。`exit actions -> transition actions -> entry actions`を一つのprivate batchとして検証し、成功時だけactive stateとともにatomic publishする。同一advanceの再帰transition、entry actionからの同期再評価、container順へのfallbackを禁止する。authoritative FSMのactive state、pending bounded task、accepted triggerはSave／ReplayへStable IDで保存し、runtime indexやpointerを保存しない。

残る汎用kindは次のentry gateまでにV1 schemaを本節へ追加する。期限前でもschema revision、validator、Cooker、migration、fixtureが揃わないkindをProject Source、AI Proposal、Cooked packageで使用できない。

| 汎用kind | schema確定期限 |
|---|---|
| Presentation cue | `phase.ai-authoring-mvp-a` entry gate |
| 汎用Balance table | `phase.ai-authoring-mvp-a` entry gate |
| bounded Behavior Tree／Blackboard | `phase.manual-3d-mvp-b` entry gate |
| Ability／Status Effect／Cooldown／Cost | `phase.manual-3d-mvp-b` entry gate |
| Quest／Objective／Dialogue／Choice | `phase.production-capability` entry gate |
| UI Flow／Screen transition／Input action mapping | `phase.production-capability` entry gate |

## 3. `GameSystemSpecV2`

`GameSystemSpecV2`はcurrent C++／wire／Source schema名であり、MCD kind `game_system`の唯一のDomain ownerである。MCD logical Type IDはschema majorを埋め込まない`type.game_system.spec`、MCD `version=2`である。`GameSystemSpecV1`と`type.game_system.spec` version 1は§3.1.2のpost-activation source schema labelで、current retained artifact／Contract set member／Catalog／serializer／Editor／AI projection集合はexact `[]`である。実在bytesを束縛したsigned `LegacyMigrationInventoryV1`が成立するまでoffline migration inputとして受理しない。C++ class、Definition graph、Editor panel、Source directoryは正本ではない。

| Field | 型／規則 |
|---|---|
| MCD共通Envelope | [Executable contracts](../02-foundation/executable-contracts.md)の全Field |
| `owner_layer` | `core \| feature_pack \| genre_pack \| project`。依存matrixを機械検証する正本 |
| `owner_ref` | `GameSystemOwnerRefV1 {owner_layer, owner_id, owner_revision, owner_content_hash}`。共通Envelope `owners[]`とexact equality |
| `system_origin` | `engine_standard \| owner_package \| project_defined`。実装供給元の分類でありlayer判定に使わない |
| `semantic_role_refs` | `SemanticRoleRecordRefV1`、1～16件 |
| `responsibility_requirement_refs` | `McdContractRefV1(kind=requirement)`、1～64件。bare IDを保存しない |
| `non_responsibility_requirement_refs` | `McdContractRefV1(kind=requirement)`、0～64件。bare IDを保存しない |
| `runtime_scope_type_ref` | `RuntimeScopeTypeRefV1 {scope_type_id, scope_type_version, scope_type_hash}`。active `RuntimeScopeTypeCatalogV1`のexact entry、厳密に1件 |
| `state_class` | `authoritative \| derived \| presentation_only \| tooling_only` |
| `owned_state_type_refs` | exact MCD Type、0～128件 |
| `read_snapshot_type_refs` | exact MCD Type、0～256件 |
| `accepted_command_type_refs` | exact MCD Type、0～256件 |
| `emitted_event_type_refs` | exact MCD Type、0～256件 |
| `emitted_port_message_type_refs` | exact MCD Type、0～256件。Command／Eventではないtyped Port transportだけ |
| `provided_capability_refs` | exact Capability、0～128件 |
| `required_capability_refs` | exact Capability、0～128件 |
| `allowed_phase_ids` | Runtime phase ID、1～16件 |
| `dependency_edge_refs` | `GameSystemDependencyEdgeRecordRefV1`、0～128件 |
| `implementation_policy_ref` | `GameSystemImplementationPolicyRecordRefV1`、厳密に1件 |
| `save_replay_contract_ref` | `SaveReplayContractRecordRefV1`。`state_class=authoritative`だけ必須、他三classではcanonical omission |
| `behavior_budget_refs` | `BehaviorBudgetRecordRefV1`、Target Profileごとのexact参照 |
| `authoring_surface_ids` | natural language／form／table／graph／timeline／sourceのsubset |
| `fallback_contract` | 意味同等fallbackまたは`no_fallback`理由 |
| `compatibility_invariant_refs` | `CompatibilityInvariantRecordRefV1`、1～128件 |
| `auxiliary_ref_set_hash` | 全補助ref Fieldのdomain-separated canonical hash |
| `extension_policy` | `sealed \| composable \| replaceable` |

非MCD補助record refは次の別型へ閉じ、すべて`id`、`version`、`content_hash`を必須にする。Field名が違う旧inline record、bare ID、`*_ids` aliasをcurrent schemaで読まない。

```text
SemanticRoleRecordRefV1
  id
  version: uint32
  content_hash: SHA-256

OwnerIdentityLocalRefV1
  owner_id
  owner_revision: positive uint64
  owner_content_hash: SHA-256

DiagnosticOwnerLocalRefV1 = wire-compatible alias of OwnerIdentityLocalRefV1
RuntimeScopeOwnerRefV1 = wire-compatible alias of OwnerIdentityLocalRefV1

GameSystemOwnerRefV1
  owner_layer: core | feature_pack | genre_pack | project
  owner_id
  owner_revision: positive uint64
  owner_content_hash: SHA-256

GameSystemDependencyEdgeRecordRefV1
  id
  version: uint32
  content_hash: SHA-256

GameSystemImplementationPolicyRecordRefV1
  id
  version: uint32
  content_hash: SHA-256

SaveReplayContractRecordRefV1
  id
  version: uint32
  content_hash: SHA-256

BehaviorBudgetRecordRefV1
  id
  version: uint32
  content_hash: SHA-256

GameSystemQualificationReceiptRefV1
  qualification_id
  qualification_version: positive uint32
  qualification_subject_hash: SHA-256
  signed_record_hash: SHA-256

GameSystemActivationBindingRefV1
  activation_binding_id
  activation_binding_version: uint32
  activation_binding_hash: SHA-256

CompatibilityInvariantRecordRefV1
  id
  version: uint32
  content_hash: SHA-256

GameSystemAuxiliaryRefSetV1
  semantic_role_refs[1..16]: SemanticRoleRecordRefV1
  responsibility_requirement_refs[1..64]: McdContractRefV1(kind=requirement)
  non_responsibility_requirement_refs[0..64]: McdContractRefV1(kind=requirement)
  dependency_edge_refs[0..128]: GameSystemDependencyEdgeRecordRefV1
  implementation_policy_ref: GameSystemImplementationPolicyRecordRefV1
  save_replay_contract_ref: SaveReplayContractRecordRefV1
    | canonical omission when state_class is not authoritative
  behavior_budget_refs[1..64]: BehaviorBudgetRecordRefV1
  compatibility_invariant_refs[1..128]: CompatibilityInvariantRecordRefV1
  auxiliary_ref_set_hash: SHA-256
```

### Owner identity registry

`DiagnosticOwnerLocalRefV1`、`RuntimeScopeOwnerRefV1`、`GameSystemOwnerRefV1`のidentity三Fieldは同じwire bytesを使う。`GameSystemOwnerRefV1.owner_layer`だけが依存方向を検証する追加Fieldであり、`owner_id`からlayerを推測しない。三型の`owner_content_hash`は既存Diagnostic identity bytesを変更せず、すべて次の一式で計算する。

```text
owner_content_hash =
  SHA-256(
    ASCII "MIRAKAN_DIAGNOSTIC_OWNER_LOCAL_IDENTITY_V1" ||
    uint32_be(len(NFC UTF-8 owner_id)) || NFC UTF-8 owner_id ||
    uint32_be(8) || uint64_be(owner_revision))

OwnerIdentityRecordV1
  owner_ref: OwnerIdentityLocalRefV1
  owner_layer: core | feature_pack | genre_pack | project
  authority_source:
    architecture_document:
      document_id
    | project:
      project_id
  status: active | deprecated | removed
  owner_record_content_hash: SHA-256

OwnerIdentityRegistryV1
  registry_id: owner_identity.registry.active
  registry_version: positive uint32
  record_count: uint32
  records[1..8192]: OwnerIdentityRecordV1
  registry_content_hash: SHA-256

OwnerIdentityRegistryRefV1
  registry_id: owner_identity.registry.active
  registry_version: positive uint32
  registry_content_hash: SHA-256
```

`owner_content_hash`のbinary identityは上式だけであり、JCS numberやlocale文字列へ変換しない。`owner_record_content_hash = SHA-256(ASCII "MIRAKAN_OWNER_IDENTITY_RECORD_V1" || uint32_be(len(RFC 8785 JCS(closed record excluding owner_record_content_hash))) || RFC 8785 JCS(closed record excluding owner_record_content_hash))`とする。`registry_content_hash = SHA-256(ASCII "MIRAKAN_OWNER_IDENTITY_REGISTRY_V1" || uint32_be(len(RFC 8785 JCS(closed registry excluding registry_content_hash))) || RFC 8785 JCS(closed registry excluding registry_content_hash))`とし、Registry preimageはID、version、`record_count`、`owner_id`／revision順へstrict sortした完成recordを持つ。JCS projectionでは全stringをNFC、SHA-256をlowercase hexadecimal exact 64文字、`owner_revision`をcanonical unsigned decimal string、uint32をsafe JSON integerとしてencodeし、unknown Fieldを禁止する。`record_count`は配列長と一致させ、duplicate identity、same identity／different record hash、非canonical order、unknown authority branch、self-hash混入を拒否する。Owner identity hashはlayer、authority source、status、Registry root、Contract set rootを含まず、Owner record／Registryの後段hashをidentityへ戻さない。

Owner revision不変条件は全Ownerへ一律に適用する。全retained Registry revisionで同じ`{owner_id, owner_revision}`を持つ完成`OwnerIdentityRecordV1` bytesと`owner_record_content_hash`はbyte-identicalでなければならない。current Registryは各`owner_id`にexact一つのselected revisionだけを持ち、同じowner IDの複数revision併存を拒否する。`owner_layer`、`authority_source`のbranchまたは値、`status`を一Fieldでも変更する場合は同じ`owner_id`の`owner_revision`をexact `N+1`へ進め、新Owner record、新Owner Registry root、新Foundation Definition Closureを発行する。同じID／revisionのin-place更新、revision飛越し、旧revision再利用、authorityをowner ID prefixから補完することを拒否する。状態遷移は`active -> deprecated -> removed`だけを許可し、逆行には新Owner IDと明示migrationを要求する。retained Project、Save、Replay、Receipt、Diagnostic、Contract setが参照する旧revisionはその旧Registry root／Foundation Closureと共に保持し、current Registryへ旧rowを併存させずselected revisionだけを新revisionへ移す。

Registry row集合またはrecord bytesが変わる場合だけ`registry_version`をexact `N+1`へ進め、同じRegistry versionで別root、同一bytesの不要なversion増加、複数current rootを拒否する。Registry version更新、Owner revision更新、Contract set／Diagnostic／Game System側のowner ref移行、Foundation Definition Closure更新は一つの承認済みdefinition migrationでset equalityを満たす。Owner recordだけ、aliasだけ、説明文だけを先行current化しない。

current初期Registryは`registry_version=1`、`record_count=17`で、全rowが`owner_revision=1`、`status=active`、`authority_source=architecture_document`の次のexact集合である。

| owner_id | owner_layer | authority `document_id` |
|---|---|---|
| `owner.core.asset_lifecycle` | `core` | `mirakan.arch.asset-lifecycle` |
| `owner.core.gameplay_programming_model` | `core` | `mirakan.arch.gameplay-programming-model` |
| `owner.core.navigation` | `core` | `mirakan.arch.simulation-navigation` |
| `owner.core.performance` | `core` | `mirakan.arch.runtime-performance-capacity` |
| `owner.core.physics` | `core` | `mirakan.arch.simulation-physics` |
| `owner.core.project_state` | `core` | `mirakan.arch.project-state` |
| `owner.core.runtime` | `core` | `mirakan.arch.runtime-scheduling-lifetime` |
| `owner.core.runtime_ecs` | `core` | `mirakan.arch.gameplay-programming-model` |
| `owner.core.security_approval` | `core` | `mirakan.arch.ai-security-approval` |
| `owner.core.ui` | `core` | `mirakan.arch.platform-ui-text-localization-accessibility` |
| `owner.core.world` | `core` | `mirakan.arch.rendering-world` |
| `owner.feature.character_locomotion` | `feature_pack` | `mirakan.arch.simulation-navigation` |
| `owner.feature.encounter_spawn` | `feature_pack` | `mirakan.arch.pack-gameplay-features` |
| `owner.feature.interaction` | `feature_pack` | `mirakan.arch.pack-gameplay-features` |
| `owner.feature.scenario_stage` | `feature_pack` | `mirakan.arch.pack-scenario-stage` |
| `owner.feature.scoring` | `feature_pack` | `mirakan.arch.pack-gameplay-features` |
| `owner.genre.shooter` | `genre_pack` | `mirakan.arch.pack-shooter` |

`owner.core.runtime_ecs`のcurrent authority sourceは本書である。将来、承認済みの専用ECS Architecture文書へauthorityを移す場合も上記の一般規則どおり同じOwner IDの`owner_revision`をexact `N+1`へ進め、新Owner record／Registry root／Foundation Definition Closureを発行する。document path変更や表示名変更だけで既存revisionのauthority sourceをin-place更新しない。

`owner.core.ai_security`、`owner.core.runtime_scheduling`、`owner.core.ui_text_accessibility`はcurrent Owner IDではなく、それぞれ`owner.core.security_approval`、`owner.core.runtime`、`owner.core.ui`への説明用旧称である。Registry row、MCD owner、Runtime Scope、Diagnostic、Game Systemへaliasを保存せず、追加Ownerとして数えない。

補助Fieldのarrayは意味上のset、wire上のsorted arrayである。各arrayはrefのlogical ID NFC UTF-8 bytes、version `uint32_be`、content hash bytesの順でstrict sortし、MCD requirement refはID、version、Contract set hashの順でsortする。同じIDのduplicate、同じID／versionの別hash、非canonical orderをrejectし、入力順を意味へ残さない。`implementation_policy_ref`は一件、`save_replay_contract_ref`はenclosing `state_class=authoritative`で一件、それ以外ではField自体をcanonical omissionし、`null`やzero refで代用しない。

`auxiliary_ref_set_hash = SHA-256(ASCII "MIRAKAN_GAME_SYSTEM_AUXILIARY_REF_SET_V1" || uint32_be(length(MCD canonical bytes of GameSystemAuxiliaryRefSetV1 excluding auxiliary_ref_set_hash)) || canonical bytes)`である。各arrayはcountをcanonical bytesに持ち、hash Fieldを唯一のself-exclusion対象にする。`type.game_system.spec` version 2だけが上記Fieldをcurrentとして登録する。全refはactive recordをexactly oneへ解決し、Specにinline payloadを埋め込まない。補助recordの`content_hash`、MCD requirement／type／CapabilityのContract set hash、Scope hash、owner ref、auxiliary set hashのどれか一つでもstaleならSystem Catalog全体をrejectする。

Fixture artifactはProduction `GameSystemSpecV2`／auxiliary hashから分離する。SpecとContract set rootをReceipt-freeで固定した後、次のowner-typed subject、signed Receipt、root外Activation bindingをこの順で生成する。

```text
GameSystemQualificationSubjectV1
  qualification_id
  qualification_version: positive uint32
  owner_ref: GameSystemOwnerRefV1
  system_ref: GameSystemContractRefV1
  system_contract_hash
  target_profile_refs[1..64]
  fixture_refs[1..128]: exact {fixture_id, fixture_version, fixture_content_hash}
  input_closure_hash
  result: pass | fail
  qualification_subject_hash: SHA-256

GameSystemQualificationReceiptV1
  subject: GameSystemQualificationSubjectV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=game_system_qualification)

GameSystemActivationBindingV1
  activation_binding_id
  activation_binding_version: positive uint32
  system_ref: exact GameSystemContractRefV1
  system_contract_hash
  qualification_receipt_refs[1..128]:
    GameSystemQualificationReceiptRefV1
  activation_binding_hash: SHA-256
```

`qualification_subject_hash`はASCII `MIRAKAN_GAME_SYSTEM_QUALIFICATION_SUBJECT_V1`と自己Fieldを除くlength-framed canonical subject bytes、`activation_binding_hash`はASCII `MIRAKAN_GAME_SYSTEM_ACTIVATION_BINDING_V1`と自己Fieldを除くbinding bytesから計算する。Qualification subjectの`owner_ref`、`system_ref`、`system_contract_hash`は`system_ref`が解決するReceipt-free `GameSystemSpecV2`の`owner_ref`、exact Contract ref、self-excluding System contract hashとbyte equality、Activation Bindingの`system_ref`／`system_contract_hash`はReceipt subjectの同Fieldとbyte equalityにする。`GameSystemQualificationReceiptRefV1`の`qualification_id／qualification_version`はwrapper内subjectの同Fieldとbyte equalityで、qualification content identityとsigned record hashは同じwrapperへexact解決する。signed wrapperはAI Verificationの二値規則に従い、`qualification_subject_hash`を含む完成Subject全体のJCS hashを`signed_record.subject_sha256`として署名する。subject payload／hash preimageへwrapperまたはReceipt refを含めない。Production `GameSystemCatalogV1`はReceipt-free Spec refとexact `GameSystemActivationBindingRefV1`を別Fieldで持ち、Bindingが指す署名済みwrapperのsubject、owner、System、Target、contract hash、result=`pass`、freshness／revocationを検証するだけで、`fixture_refs[]`のbody、oracle、pathをRuntime Package、Save、Replay、Production registryへ解決しない。生成順は`Receipt-free GameSystemSpecV2／auxiliary hash → Contract set root／GameSystemContractRefV1 → Qualification subject → signed Receipt → Activation binding／Catalog projection`であり、Receipt、wrapper、Activation bindingをSpec、auxiliary hash、Contract set rootへ戻さない。Fixture集合変更は新subject／Receipt／Activation bindingを発行し、Production Spec hashへFixture refまたはReceipt refを混入しない。正しいSpec refのままsubject owner／contract hashだけを差し替えるcase、BindingのSystem ref／contract hashだけを別subjectへ差し替えるcaseを一原因negative fixtureで拒否する。

Fixture内だけで動くtest double／stub／oracle implementationはProduction `GameSystemSpecV2`を名乗らない。次の別identity、owner、Registry、Qualification contractだけを使用する。

```text
FixtureImplementationSystemRegistryRefV1
  registry_id
  registry_version: positive uint32
  registry_content_hash: SHA-256

FixtureImplementationSystemRefV1
  registry_ref: FixtureImplementationSystemRegistryRefV1
  fixture_system_id
  fixture_system_version: positive uint32
  fixture_system_content_hash: SHA-256

FixtureImplementationSystemRecordV1
  fixture_system_id
  fixture_system_version: positive uint32
  fixture_owner_ref:
    exact {fixture_id, fixture_version, fixture_content_hash}
  implementation_artifact_ref: ArtifactRefV1
  read_type_refs[0..256]: McdContractRefV1(kind=type)
  accepted_command_type_refs[0..256]: McdContractRefV1(kind=type)
  accepted_port_message_type_refs[0..256]: McdContractRefV1(kind=type)
  emitted_event_type_refs[0..256]: McdContractRefV1(kind=type)
  emitted_port_message_type_refs[0..256]: McdContractRefV1(kind=type)
  supported_target_profile_refs[1..64]
  fixture_system_content_hash: SHA-256

FixtureImplementationSystemQualificationSubjectV1
  qualification_id
  qualification_version: positive uint32
  fixture_system_ref: FixtureImplementationSystemRefV1
  fixture_owner_ref:
    exact same fixture ref as the Registry record
  target_profile_refs[1..64]
  fixture_refs[1..128]:
    exact {fixture_id, fixture_version, fixture_content_hash}
  input_closure_hash
  result: pass | fail
  qualification_subject_hash: SHA-256

FixtureImplementationSystemQualificationReceiptV1
  subject: FixtureImplementationSystemQualificationSubjectV1
  signed_record:
    exact MirakanSignedRecordV1(
      purpose=fixture_implementation_system_qualification)

FixtureImplementationSystemQualificationReceiptRefV1
  qualification_id
  qualification_version: positive uint32
  qualification_subject_hash: SHA-256
  signed_record_hash: SHA-256

FixtureImplementationSystemActivationBindingRefV1
  activation_binding_id
  activation_binding_version: positive uint32
  activation_binding_hash: SHA-256

FixtureImplementationSystemActivationBindingV1
  activation_binding_id
  activation_binding_version: positive uint32
  fixture_system_ref: FixtureImplementationSystemRefV1
  qualification_receipt_refs[1..64]:
    FixtureImplementationSystemQualificationReceiptRefV1
  activation_binding_hash: SHA-256

FixtureImplementationSystemRegistryV1
  registry_id: fixture_implementation_system.registry.active
  registry_version: positive uint32
  records[1..4096]: FixtureImplementationSystemRecordV1
  registry_content_hash: SHA-256

UsageTaggedImplementationSystemBaseRefV1
  usage: production | project_owned | fixture_only
  production_system_ref:
    GameSystemContractRefV1
    | canonical omission when usage=fixture_only
  production_system_contract_hash:
    SHA-256
    | canonical omission when usage=fixture_only
  fixture_system_ref:
    FixtureImplementationSystemRefV1
    | canonical omission when usage=production or project_owned

UsageTaggedImplementationSystemRefV1
  base_ref: exact UsageTaggedImplementationSystemBaseRefV1
  production_system_activation_binding_ref:
    GameSystemActivationBindingRefV1
    | canonical omission when base_ref.usage=fixture_only
  fixture_system_activation_binding_ref:
    FixtureImplementationSystemActivationBindingRefV1
    | canonical omission when base_ref.usage=production or project_owned
```

Fixture implementation base recordはID／version順、全Type／Target集合は各typed refのcanonical identity順へstrict sortする。record hashとRegistry hashはそれぞれASCII `MIRAKAN_FIXTURE_IMPLEMENTATION_SYSTEM_RECORD_V1`、`MIRAKAN_FIXTURE_IMPLEMENTATION_SYSTEM_REGISTRY_V1`と自己hashを除くReceipt-free Fieldのlength-framed canonical bytesから計算する。Registry refとSystem refを固定した後、Qualification subject hashをASCII `MIRAKAN_FIXTURE_IMPLEMENTATION_SYSTEM_QUALIFICATION_SUBJECT_V1`、Activation binding hashをASCII `MIRAKAN_FIXTURE_IMPLEMENTATION_SYSTEM_ACTIVATION_BINDING_V1`と各自己Fieldを除くbytesから計算する。signed wrapperは完成subjectだけを署名し、subject／record／Registry hash preimageへwrapper、Receipt ref、Activation bindingを含めない。生成順は`receipt-free base record → Registry hash／FixtureImplementationSystemRefV1 → Qualification subject → signed Receipt → Activation binding`である。duplicate、same-ID／version different-hash、owner不一致、stale Artifact／Type／Target／Receipt、failed／revoked QualificationをActivation binding materialization前に拒否する。

`UsageTaggedImplementationSystemBaseRefV1`はReceipt-freeなSystem identityだけを持ち、discriminatorとbranch Fieldをexactに一致させる。`usage=production | project_owned`では`production_system_ref`と`production_system_contract_hash`だけ、`fixture_only`では`fixture_system_ref`だけを必須とし、Qualification Receipt／Activation Binding refを一件も含めない。`UsageTaggedImplementationSystemRefV1`は完成base refにroot外System Activation Bindingを加えるdownstream qualified-dependency refである。前者ではbaseのSystem refをsubjectにする`production_system_activation_binding_ref`だけ、後者ではbaseのFixture System refをsubjectにするexact `fixture_system_activation_binding_ref`だけを必須とし、両branch、両方欠落、discriminator外Fieldを拒否する。production consumerのtyped ownerは参照先`GameSystemSpecV2.owner_ref`とexact equalityにする。fixture consumerは自身の`fixture_owner_ref {fixture_id,fixture_version,fixture_content_hash}`をFixture Registry record、Qualification subject、Activation bindingが解決する同型Fieldとexact equalityにし、`GameSystemOwnerRefV1`またはowner kind enumとの異型比較を行わない。生成順は`receipt-free System base／System ref → System Qualification subject → signed Receipt → System Activation Binding → UsageTaggedImplementationSystemRefV1`であり、downstream refをSystem base、Fixture Registry、Contract set rootへ戻さない。

Fixture Registry、System ref、Qualification Receipt、Activation bindingはQualification sandboxだけが解決する。Production `GameSystemCatalogV1`、`GameSystemSpecV2`、`SystemImplementationSetV1`、Project Source／Compile Manifest、Dependency Graph、Save、Replay、Cooked／Runtime Packageへ一件も投影しない。fixture refのProduction branch混入、Production `GameSystemContractRefV1`のfixture branch混入、cross-owner、stale Registry／record／subject／Receipt／Activation binding hashを各一原因で拒否し、last-valid Production catalog／packageを不変にする。

一つのSpecはCatalogで解決した一scopeだけを持つ。複数scopeのStateを所有する場合はSystemを分割し、Stable handleまたはtyped Eventで接続する。

### 3.1 `RuntimeScopeTypeCatalogV1`

```text
RuntimeScopeTypeCatalogV1
  catalog_id: runtime_scope.catalog.active
  catalog_schema_version: 1
  catalog_version
  catalog_hash
  contract_set_hash
  dependency_registry_ref: RuntimeScopeDependencyRegistryRefV1
  dependency_registry_hash
  entries[5..4096]:
    scope_type_ref: RuntimeScopeTypeRefV1
    instance_key_schema_ref: McdContractRefV1(kind=type)
    owner_ref: RuntimeScopeOwnerRefV1
    lifetime_ref: McdContractRefV1(kind=policy)
    save_replay_policy_ref: McdContractRefV1(kind=policy)
    activation_condition_ref: McdContractRefV1(kind=policy)
    deactivation_condition_ref: McdContractRefV1(kind=policy)

RuntimeScopeTypeRefV1
  scope_type_id
  scope_type_version: uint32
  scope_type_hash: SHA-256

RuntimeScopeDependencyRegistryRefV1
  registry_id
  registry_revision: uint64
  registry_content_hash: SHA-256

RuntimeScopeCatalogRefV1
  catalog_id
  catalog_schema_version: 1
  catalog_version
  catalog_hash
  contract_set_hash
  dependency_registry_ref: RuntimeScopeDependencyRegistryRefV1
  dependency_registry_hash

RuntimeScopeOwnerRegistryV1
  registry_id
  registry_revision
  registry_content_hash
  records[1..8192]:
    owner_ref: RuntimeScopeOwnerRefV1
    availability: available | unavailable | removed
    owning_component_ref: exact ref/version/content_hash
    allowed_scope_namespaces[1..64]
    activation_service_ref: exact ref/version/content_hash

RuntimeScopeDependencyRegistryV1
  registry_id
  registry_revision
  registry_content_hash
  owner_registry_ref: exact ref/revision/content_hash
  records[1..32768]:
    dependency_kind:
      instance_key_schema | lifetime | save_replay |
      activation | deactivation
    contract_ref: McdContractRefV1
    record_content_hash: SHA-256
    owner_ref: RuntimeScopeOwnerRefV1
    status: active | deprecated | removed

# Conditional legacy migration destination types below;
# not current Contract set members.
RuntimeScopeMigrationContributionRefV1
  contribution_id
  contribution_version: uint32
  contribution_content_hash: SHA-256

RuntimeScopeMigrationContributionV1
  contribution_id
  contribution_version: uint32
  contributor_owner_ref: RuntimeScopeOwnerRefV1
  source_system_match_policy_ref: McdContractRefV1(kind=policy)
  retained_source_mcd_ref: McdContractRefV1(
    id=type.game_system.spec, version=1, contract_set_hash)
  destination_system_schema_ref: McdContractRefV1(
    id=type.game_system.spec, version=2, contract_set_hash)
  legacy_scope_value
  destination_scope_type_ref: RuntimeScopeTypeRefV1
  auxiliary_record_migration_policy_ref: McdContractRefV1(kind=policy)
  identity_mapping_policy_ref: McdContractRefV1(kind=policy)
  contribution_content_hash: SHA-256

RuntimeScopeMigrationContributionRegistryRefV1
  registry_id
  registry_version: uint32
  registry_content_hash: SHA-256

RuntimeScopeMigrationContributionRegistryV1
  registry_id: runtime_scope.migration_contribution.registry.active
  registry_version: 1
  registry_content_hash: SHA-256
  records[4..4096]: RuntimeScopeMigrationContributionV1

RuntimeScopeMigrationQualificationSubjectV1
  qualification_id
  qualification_version: positive uint32
  contribution_ref: RuntimeScopeMigrationContributionRefV1
  owner_ref: RuntimeScopeOwnerRefV1
  fixture_refs[1..64]: exact fixture ref/version/content_hash
  source_and_destination_schema_hashes
  result: pass | fail
  qualification_subject_hash: SHA-256

RuntimeScopeMigrationQualificationReceiptV1
  subject: RuntimeScopeMigrationQualificationSubjectV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=runtime_scope_migration_qualification)

RuntimeScopeMigrationQualificationReceiptRefV1
  qualification_id
  qualification_version: positive uint32
  qualification_subject_hash: SHA-256
  signed_record_hash: SHA-256

RuntimeScopeMigrationQualificationBindingRefV1
  qualification_binding_id
  qualification_binding_version: positive uint32
  qualification_binding_hash: SHA-256

RuntimeScopeMigrationQualificationBindingV1
  qualification_binding_id
  qualification_binding_version: positive uint32
  contribution_ref: RuntimeScopeMigrationContributionRefV1
  qualification_receipt_ref:
    RuntimeScopeMigrationQualificationReceiptRefV1
  qualification_binding_hash: SHA-256

RuntimeScopeMigrationActivationCatalogV1
  catalog_id: runtime_scope.migration_activation_catalog.active
  catalog_version: positive uint32
  entries[4..4096]:
    exact {RuntimeScopeMigrationContributionRefV1,
           RuntimeScopeMigrationQualificationBindingRefV1}
  catalog_hash: SHA-256
```

`RuntimeScopeOwnerRefV1`は[Owner identity registry](#owner-identity-registry)で一度だけ宣言した`OwnerIdentityLocalRefV1`のwire-compatible aliasをforward-useする。このCatalog節は同型を再定義せず、Owner ID／revision／content hashの追加・省略・別hash式を許可しない。

`instance_key_schema_ref`、`lifetime_ref`、`save_replay_policy_ref`、`activation_condition_ref`、`deactivation_condition_ref`は[Executable contracts](../02-foundation/executable-contracts.md#5-mcd共通envelope)の`McdContractRefV1 {id, version, contract_set_hash}`そのものである。`owner_ref`はactive `RuntimeScopeOwnerRegistryV1`へrevision／content hash込みで解決する。CatalogとCatalog refの`dependency_registry_hash`は必ず`dependency_registry_ref.registry_content_hash`と等しい。conditional migrationがActivationされた後だけ、migration inputの`destination_catalog_hash`も`destination_catalog_ref.catalog_hash`と等しくなければならない。`scope_type_hash`は`scope_type_hash`自身を除く当該entryの六依存refを含むMCD canonical bytesのSHA-256であり、表示名、current owner、latest policyへ再解決しない。

Core entryは次のexact 5件だけである。表は読みやすさのためIDだけを示すが、保存値は全cellについて上記typed refであり、Core初版は`scope_type_version=1`、MCD refは`version=1`、ownerはRegistryが固定したexact revision、全refに対応content／Contract set hashを必須とする。IDだけの表文字列をSource／Save／Replay／Receiptへ保存することを禁止する。

| `scope_type_ref` | `instance_key_schema_ref` | `owner_ref` | `lifetime_ref` | `save_replay_policy_ref` | `activation_condition_ref` | `deactivation_condition_ref` |
|---|---|---|---|---|---|---|
| `scope.core.application` | `type.runtime_scope.key.application_singleton` | `owner.core.runtime` | `policy.runtime_scope.lifetime.process` | `policy.runtime_scope.save_replay.application_none` | `policy.runtime_scope.activation.process_started` | `policy.runtime_scope.deactivation.process_stopping` |
| `scope.core.runtime_session` | `type.runtime_scope.key.runtime_session_uuidv7` | `owner.core.runtime` | `policy.runtime_scope.lifetime.runtime_session` | `policy.runtime_scope.save_replay.runtime_session` | `policy.runtime_scope.activation.entry_ready` | `policy.runtime_scope.deactivation.stop_or_fault` |
| `scope.core.world` | `type.runtime_scope.key.world_instance` | `owner.core.world` | `policy.runtime_scope.lifetime.world_instance` | `policy.runtime_scope.save_replay.world` | `policy.runtime_scope.activation.world_branch_ready` | `policy.runtime_scope.deactivation.world_branch_teardown` |
| `scope.core.entity` | `type.runtime_scope.key.entity_stable_id` | `owner.core.runtime_ecs` | `policy.runtime_scope.lifetime.entity` | `policy.runtime_scope.save_replay.entity_owner_state` | `policy.runtime_scope.activation.entity_created` | `policy.runtime_scope.deactivation.entity_destroyed` |
| `scope.core.ui_session` | `type.runtime_scope.key.ui_session_uuidv7` | `owner.core.ui` | `policy.runtime_scope.lifetime.ui_session` | `policy.runtime_scope.save_replay.ui_session` | `policy.runtime_scope.activation.ui_branch_ready` | `policy.runtime_scope.deactivation.ui_branch_teardown` |

Feature／Genre／Project ownerは自身のregistered namespace内へscopeを登録できる。Coreから非Core scopeへの依存、FeatureからGenre／Project scopeへの依存、Genreから別Genre／Project scopeへの依存、あるownerから未宣言の別owner scopeへの依存、owner namespaceを偽装した登録を拒否する。各entryの7 Fieldはすべて必須で、owner availabilityとversion、instance key schema、lifetime、Save／Replay policy schema hash、activation／deactivation conditionをCatalog materializationとRuntime activationの両方で検証する。

`RuntimeScopeOwnerRegistryV1`は上記typed owner recordを、`RuntimeScopeDependencyRegistryV1`は上表で参照する各type／policyの`McdContractRefV1`、record content hash、owner ref／hash、statusをactive recordとして持つ。Owner recordはowner ID／revision、Dependency recordはdependency kind／contract ID／versionのcanonical byte順にstrict sortし、duplicateを拒否する。各Registry content hashはASCII domain separator（`MIRAKAN_RUNTIME_SCOPE_OWNER_REGISTRY_V1`または`MIRAKAN_RUNTIME_SCOPE_DEPENDENCY_REGISTRY_V1`）、自身のID／revision、Dependency Registryではexact Owner Registry ref、record count、全record canonical bytesを順に入力し、`registry_content_hash`自身を除外してSHA-256する。上表5行に現れる全dependencyはこの二Registryへ実体recordを一件ずつ持ち、unknown、duplicate、deprecated、removed、self-asserted ownerを解決済みと扱わない。

以下のMigration Contribution Registry、Contribution、Qualification、Activation Catalogは`operation.runtime_scope.migrate_game_system@1`が将来atomic activationされた後のdestination templateであり、current Runtime Scope Catalog／Owner／Dependency Registryの成立条件ではない。currentのMigration Contribution record／Registry、Qualification subject／Receipt／Binding、Activation Catalogはすべてexact `[]`で、Production／offline migratorは存在せずdispatchしない。activation後はCoreがgeneric Receipt-free schema、hash、resolver、validatorだけを所有し、Feature／Genre／Project ownerが自身のmappingを下向きに登録する。recordは`contribution_content_hash`を除く全Field、RegistryはASCII `MIRAKAN_RUNTIME_SCOPE_MIGRATION_CONTRIBUTION_REGISTRY_V1`、Registry ID／version、record count、contribution ID／version順Receipt-free record canonical bytesを`uint32_be` length framingしてhashし、Registry self hashを除外する。同じcontribution ID、同じ`{source_system_match_policy_ref, legacy_scope_value}`、同じSource Systemに同時matchする複数record、非canonical order、stale owner／schema／policyをRegistry全体のerrorにする。Contribution／Registry確定後の生成順は`receipt-free Contribution → Registry／ContributionRef → Qualification subject → signed Receipt → Qualification binding → root外Activation Catalog`である。subject hashはASCII `MIRAKAN_RUNTIME_SCOPE_MIGRATION_QUALIFICATION_SUBJECT_V1`、binding hashはASCII `MIRAKAN_RUNTIME_SCOPE_MIGRATION_QUALIFICATION_BINDING_V1`、Catalog hashはASCII `MIRAKAN_RUNTIME_SCOPE_MIGRATION_ACTIVATION_CATALOG_V1`と各自己Fieldを除くcount／length-framed canonical bytesから計算する。subject ownerはContribution owner、Binding contribution refはsubjectとbyte equalityで、purpose/typeが異なる`GameSystemQualificationReceiptRefV1`を代用しない。Receipt／Binding／CatalogをContribution／Registry hashへ戻さない。activation後のProduction migratorはActivation Catalogが指すQualification Receiptのsubject／result=`pass`／freshness／revocationだけを検証し、Fixture bodyを解決しない。offline migratorはSource System ref／hash、exact Field `retained_source_mcd_ref`、legacy scope valueを入力にexact一件を選び、0件または2件以上なら推測せず`MIRAKAN-RUNTIME-SCOPE-CONTRIBUTION_INVALID`で拒否する。

activation後のdestination Core contribution候補はCore-owned Systemだけを対象に次の四完全recordである。全rowは`contribution_version=1`、`contributor_owner_ref={owner.core.gameplay_programming_model, destination owner revision, content hash}`、source schema=`{type.game_system.spec,1,source Contract set hash}`、destination schema=`{type.game_system.spec,2,destination Contract set hash}`を持つ。source match policyはCore owner layer、`game_system.engine.*` namespace、signed inventoryが束縛したexact legacy System ref／hashを検証し、同じlegacy valueを持つFeature／Genre／Project Systemへ適用しない。表の四rowはcurrent Registry memberではなく、現時点のcurrent contribution集合はexact `[]`である。

| contribution ID | source match policy | legacy scope | destination scope ref | auxiliary migration policy | identity mapping policy |
|---|---|---|---|---|---|
| `runtime_scope.migration_contribution.core.runtime_session` | `policy.runtime_scope.migration.match_core_runtime_session_system@1` | `play_session` | `{scope.core.runtime_session,1,scope_hash}` | `policy.runtime_scope.migration.auxiliary.core_runtime_session@1` | `policy.runtime_scope.migration.identity.preserve_runtime_session@1` |
| `runtime_scope.migration_contribution.core.world` | `policy.runtime_scope.migration.match_core_world_system@1` | `world_instance` | `{scope.core.world,1,scope_hash}` | `policy.runtime_scope.migration.auxiliary.core_world@1` | `policy.runtime_scope.migration.identity.preserve_world@1` |
| `runtime_scope.migration_contribution.core.entity` | `policy.runtime_scope.migration.match_core_entity_system@1` | `entity_instance` | `{scope.core.entity,1,scope_hash}` | `policy.runtime_scope.migration.auxiliary.core_entity@1` | `policy.runtime_scope.migration.identity.preserve_entity@1` |
| `runtime_scope.migration_contribution.core.ui_session` | `policy.runtime_scope.migration.match_core_ui_session_system@1` | `ui_session` | `{scope.core.ui_session,1,scope_hash}` | `policy.runtime_scope.migration.auxiliary.core_ui_session@1` | `policy.runtime_scope.migration.identity.preserve_ui_session@1` |

表の全policyはactivation transactionが生成するexact `McdContractRefV1(kind=policy,version=1,contract_set_hash)`、destinationは`RuntimeScopeTypeRefV1`である。各row固定後に同logical suffixのQualification subject／Receipt／Bindingを一件ずつ作り、Activation CatalogのContribution集合とRegistryのContributionRef集合をexact set equalityにする。Fixtureは別`RuntimeScopeMigrationQualificationSubjectV1`にだけ存在し、Feature／Genre／Project固有mapping、Fixture、adapter IDをこの表またはCore validator binaryへ追加しない。

entryは`scope_type_ref.scope_type_id`のNFC UTF-8 byte順で厳密にsortし、duplicateを拒否する。`catalog_hash`のexact inputは、ASCII domain separator `MIRAKAN_RUNTIME_SCOPE_CATALOG_V1`、`catalog_id`、`catalog_schema_version`、`catalog_version`、`contract_set_hash`、`dependency_registry_ref`のcanonical bytes、これと同値の`dependency_registry_hash`、entry count、各entryの七typed refをこの順にMCD canonical encodeしたbytesであり、`catalog_hash`自身を除外してSHA-256する。配列順、scope version／hash、owner revision／hash、MCD version／Contract set hashが一つでも異なれば別Catalogである。

| Diagnostic code | 条件 |
|---|---|
| `MIRAKAN-RUNTIME-SCOPE-CATALOG_INVALID` | entry count、sort、duplicate、unknown Field／scope不正 |
| `MIRAKAN-RUNTIME-SCOPE-OWNER_UNAVAILABLE` | owner missing／removed、activation時unavailable |
| `MIRAKAN-RUNTIME-SCOPE-VERSION_HASH_MISMATCH` | Catalog／entry／instance key／Save Replay policyのversionまたはhash不一致 |
| `MIRAKAN-RUNTIME-SCOPE-MIGRATION_CONFLICT` | 旧scope、owner removal、persisted instanceのclean migrationが一意でない |

表の先頭三件だけがcurrent generic Runtime Scope Catalog Diagnosticである。`MIRAKAN-RUNTIME-SCOPE-MIGRATION_CONFLICT`はpost-activation destination candidateで、`MIRAKAN-RUNTIME-SCOPE-CONTRIBUTION_INVALID`、Receipt binding mismatchと共にcurrent Diagnostic Registry／Operation error／Validator closure集合はexact `[]`である。current Catalog materializationは移行候補Diagnosticを返さず、旧Source検出をgeneric Catalog成功へ補正しない。

Catalog materialization、owner removal、Runtime activationのいずれかで失敗した場合はCatalog、System Graph、last-valid active instance、Save／Replay mappingを変更しない。Scope Source identity、Save／Replay instance identity、ephemeral runtime generationを相互に置換しない。

### 3.1.2 Scope依存recordとconditional offline migration

current validatorはSource schema refが`type.game_system.spec` version 2でない入力、`runtime_instance_scope`、legacy enum、bare scope／auxiliary ID、version／hash欠落、current Catalog以外のscope hashを`MIRAKAN-RUNTIME-SCOPE-CATALOG_INVALID`でrejectする。`GameSystemSpecV1` bytesと該当する旧Project／Systemが実在することは現計画からは証明されておらず、current deserializerへaliasを残さず、説明上の旧inline payloadをcurrent migration inputとして受理しない。

`operation.runtime_scope.migrate_game_system@1`は[Executable contracts §8.1.2](../02-foundation/executable-contracts.md#812-conditional-legacy-migration-evidence-gate)のconditional legacy migrationで、current状態は`not_activated`である。このOperationに固有のcurrent MCD／Owner Manifest／Service allowlist／Policy／Validator／migration Diagnostic／Receipt／Provider／MCP／alias／Qualification Binding／Activation Catalog集合はすべてexact `[]`であり、`service.offline_project_migrator`、`capability.authoring.offline_migration`、`profile.isolation.offline_project_migrator`のcurrent集合もexact `[]`である。入力Type、出力Type、Prepared Receipt Type、Migration Contribution／Registry／Qualification型を含む以下の詳細schemaはcurrent Contract set memberではなく、post-activation destination templateである。code block内のOperation／Type／Service／Policy refをcurrent refとして解決または公開しない。

将来Activationするには、実在する旧`GameSystemSpecV1` schema bytes、旧Project／System bytes、source `ContractSetSnapshotV2`、Owner Identity Registry、Named Algorithm Registry、`FoundationDefinitionClosureV1`、全retained artifact ref／hash、正負fixtureを一件も推測せず列挙したsigned `LegacyMigrationInventoryV1`が§8.1.2 gateを満たさなければならない。その同じ承認済みtransactionだけが、Operation／Type／Policy／Validator／migration Diagnostic／Receipt、offline Service／Capability／Isolation Profile、Service allowlist、Contribution Registry／records、Qualification Receipt／Binding、Activation Catalog、Provider／MCP projectionを完全closureとして同時にmaterializeできる。部分Activation、説明文からのlegacy bytes生成、移行前のalias、silent変換を禁止する。Activation後のlogical IDはversion-neutral `operation.runtime_scope.migrate_game_system` version 1とし、source／destination schema majorをIDへ埋め込まず、exact source／destination schema refをinputへ持つ。

```text
RuntimeScopeGameSystemMigrationInputV1
  input_type_ref: McdContractRefV1(
    id=type.runtime_scope.game_system_migration_input, version=1, contract_set_hash)
  operation_ref: McdContractRefV1(
    id=operation.runtime_scope.migrate_game_system, version=1, contract_set_hash)
  before_project_ref:
    exact {project_id, expected_project_revision, document_set_hash}
  operation_intent_hash
  request_hash
  idempotency_key
  source_foundation_definition_closure_ref:
    FoundationDefinitionClosureRefV1
  retained_source_mcd_ref: McdContractRefV1(
    id=type.game_system.spec, version=1, source_contract_set_hash)
  destination_system_schema_ref: McdContractRefV1(
    id=type.game_system.spec, version=2, destination_contract_set_hash)
  source_system_ref/hash
  source_system_payload_hash
  legacy_scope_value
  legacy_auxiliary_payload_hash
  destination_catalog_ref: RuntimeScopeCatalogRefV1
  destination_catalog_hash
  contribution_registry_ref: RuntimeScopeMigrationContributionRegistryRefV1
  contribution_registry_hash
  selected_contribution_ref: RuntimeScopeMigrationContributionRefV1
  resolved_scope_entry: exact seven typed refs
  resolved_auxiliary_ref_set: GameSystemAuxiliaryRefSetV1
  source_instance_identity_mapping_ref/hash
  save_identity_mapping_ref/hash
  replay_identity_mapping_ref/hash
  preview_policy_ref: McdContractRefV1(kind=policy)
  validation_policy_ref: McdContractRefV1(kind=policy)
  mutation_authorization_binding: exact MutationAuthorizationBindingV2(
    risk_class=R3, authority_evidence=approval)

RuntimeScopeGameSystemMigrationResultV1
  disposition: migrated | rejected
  migrated:
    after_project_ref:
      exact {project_id, project_revision, document_set_hash}
    destination_system_source_ref/hash
    destination_system_schema_ref
    runtime_scope_type_ref: RuntimeScopeTypeRefV1
    auxiliary_ref_set_hash
    preview_receipt_ref/hash
    validation_receipt_ref/hash
    public_publication_marker_ref/hash
    migration_receipt_ref/hash
  rejected:
    diagnostics[1..64]: DiagnosticCodeRefV1

PreparedRuntimeScopeMigrationReceiptPayloadV1
  publication_binding: exact PreparedReceiptPublicationBindingV1
  operation_ref: McdContractRefV1(kind=operation)
  operation_intent_hash
  request_hash
  idempotency_key
  before_project_ref
  after_project_ref
  source_system_ref/hash
  source_foundation_definition_closure_ref:
    FoundationDefinitionClosureRefV1
  destination_system_source_ref/hash
  retained_source_mcd_ref
  destination_system_schema_ref
  legacy_scope_value_hash
  legacy_auxiliary_payload_hash
  contribution_registry_ref/hash
  selected_contribution_ref
  seven_dependency_after_refs/hashes
  auxiliary_ref_set_after: GameSystemAuxiliaryRefSetV1
  source_instance_identity_mapping_ref/hash
  save_identity_mapping_ref/hash
  replay_identity_mapping_ref/hash
  preview_receipt_payload_ref/hash
  validation_receipt_payload_ref/hash
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1
  ephemeral_generation_migrated: false
  diagnostics[0..64]: DiagnosticCodeRefV1
  prepared_payload_hash: SHA-256

RuntimeScopeMigrationReceiptV1
  published_receipt:
    exact PublishedDomainReceiptV2 whose
    prepared_domain_receipt_payload_ref/hash resolves
    PreparedRuntimeScopeMigrationReceiptPayloadV1
```

Activation後の`operation_intent_hash`と`request_hash`は[Executable contracts §8.1.2](../02-foundation/executable-contracts.md#812-conditional-legacy-migration-evidence-gate)から参照する唯一の`MIRAKAN_OPERATION_INTENT_V2 -> MutationAuthorizationBindingV2 -> MIRAKAN_OPERATION_REQUEST_V2`を使い、本書では式またはanonymous bindingを再定義しない。Operation、before Project、source Foundation Closure、source／destination schema、Source、Catalog、Contribution Registry／record、scope／auxiliary closure、identity mapping、Preview／Validationはintentへ入り、binding確定後のrequestはintent hashとexact Approval Set evidenceを含む。`source_foundation_definition_closure_ref`はInputおよび選択した`RuntimeScopeMigrationContributionV1`のexact Field `retained_source_mcd_ref`とSource System recordのContract set／Owner refを同時代のContract Set／Owner Identity Registry／Named Algorithm Registryへexact解決する。三つの`retained_source_mcd_ref`（Input、Contribution、Prepared Receipt）はbyte equalityで、同じsource Closureへだけ解決する。Mutation binding、Operation／input／output／Policy／Validator／Diagnostic／destination schema／Contributionのdestination ref／mapping refはdispatch時のdestination Foundation Closureへ解決し、source Closureへdowngradeしない。InputとPrepared Receiptのsource Closure refもbyte equalityにし、Public Receiptのprepared payloadからretained rootをhistorical解決できなければならない。source Closure省略、別source root、複数source root、retained source refの別名化または三者差、source schemaをdestination rootへalias、destination refをretained rootへ混入するcaseを一原因で拒否する。Prepared payload hashはASCII `MIRAKAN_PREPARED_RUNTIME_SCOPE_MIGRATION_RECEIPT_PAYLOAD_V1`とself-excluding canonical bytesから計算する。唯一のsigned subject／wrapperはExecutable Contractsの`PublishedDomainReceiptPayloadV2`／`PublishedDomainReceiptV2`であり、Domain固有Subject／alternate wrapper／inline signer／key／algorithm／signature Fieldを持たない。

Activation後の`migrated`ではbefore／after Project IDが一致しrevisionが一だけ増加し、ResultとReceiptのafter Project、destination Source、schema、scope、auxiliary set、Preview／Validation／Public Publication Marker ref／hashがexact equalityでなければならず、そのMarkerとReceiptは同じPublic Commit Closureへ解決する。Preview、Validation、Prepared Candidate、staged postcondition成功後のpublicationは[Executable Contracts §8](../02-foundation/executable-contracts.md#8-operation定義)をcanonical reuseし、`private Marker read-back → secret-free PublicCommitClosureV1 candidate → signed wrapper read-back → PublicCommitClosureV1＋PublicPublicationMarkerV1＋after Projectのatomic CAS → Result`の順に固定する。Closureの`domain_commitment.kind`は`owner_typed_state_commit`、`domain_owner_ref`はdestination Foundation Closureが解決するexact `owner.core.gameplay_programming_model` ref、committed artifact集合はPrepared payloadが束縛したreceipt-free destination Source／Catalog artifact ref集合とする。Closure Ref／hash規則は同節から参照し、本書で別式を定義しない。Closure bodyまたは同Closureを束縛するsigned wrapperを欠くPublic Marker／after-state current authorityを拒否する。各Receipt payloadのintent／request hashはOperation inputと一致する。`rejected`ではafter Project、destination Source、Public Marker／migration Receiptをcanonical omissionし、Public Commit Closure bodyを公開せず、diagnosticsはOperation `errors[]`とreachable Validator error集合のsubset 1～64件にする。成功Receiptのdiagnosticsは同じclosed setのwarning／nonblocking recordだけを0～64件許可する。

Activation Qualificationでは、同じ`idempotency_key`＋`request_hash`のretryがbyte-identical Resultと同じReceipt ref／hashを返し、同じkeyの別requestを`MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE`で拒否することを検証する。source／destination Closureまたはschema、Contribution、owner row、七dependency、auxiliary record、Save／Replay mappingの非一意、stale hash、identity collision、Receipt binding不一致は全migrationをrejectし、last-valid Source／Catalog／active instanceを維持する。Activation候補のCore fixtureは四Core contributionをSource／Save／Replay round-tripまで検証する。extension固有mapping／adapter／fixtureは各contributor ownerが同じactivation closureへ登録し、Core fixture inventoryへコピーしない。negative fixtureはsource Closure missing／wrong root／destination substitution、source／destination schema逆転、bare ref、Contribution 0件／複数、removed owner、各dependency hash mismatch、auxiliary sort／duplicate／hash mismatch、Receipt request／Project／Source／mapping mismatch、partial migrationを一原因ずつ拒否する。

`GameSystemImplementationPolicyV1`は許可Implementation kind、default implementation、Native eligibility、replacement policy、live switch policy、equivalence fixture、required Target、configuration schema、unavailable behaviorを持つ。Native live switchは許可しない。Project overrideもPublic Contract、State、Save field、Replay意味を変更できない。`configuration_schema_ref`は厳密に一件のMCD Type refを必須とし、構成Fieldを持たないPolicyも省略、`null`、zero ref、名前だけのempty schemaにしない。

構成Fieldを持たないSystem用の共有schemaは次の完全なMCD Type一件だけである。

```text
EmptyGameSystemConfigurationV1
  field_count: exact 0
  additional_fields: forbidden
  canonical_value: exact MCD empty map

MCD Type local record
  mcd_version: 1
  kind: type
  id: type.game_system.empty_configuration
  version: 1
  status: active
  title: Empty Game System Configuration
  description:
    closed zero-field configuration for a System whose implementation
    policy exposes no configurable values
  owners: [owner.core.gameplay_programming_model]
  requirement_refs: []
  rationale_refs: [mirakan.arch.gameplay-programming-model#3-gamesystemspecv2]
  since_contract_set: 2
  supersedes: []
  tags: [configuration,game_system,shared]
  payload_schema: exact EmptyGameSystemConfigurationV1 above
```

このType local recordは`ContractSetMemberLocalRecordV2(member_kind=mcd, local_identity={kind=type,id=type.game_system.empty_configuration,version=1})`として一度だけSnapshotへ含め、self-excluding MCD member hashとset rootを固定後にだけ`McdContractRefV1 {id=type.game_system.empty_configuration,version=1,contract_set_hash}`へmaterializeする。各Policy local payloadはroot前に同じType LocalRef、external Policy recordはroot後に同じexternal refを持つ。Core／Feature／Genre／Projectごとの複写、別ownerの同ID、unknown Fieldを含むempty value、schema ref missing／wrong kind／stale root、PolicyがこのTypeを指しながら構成値を一Fieldでも持つcaseをrejectする。構成Fieldが必要なPolicyはowner自身の別complete MCD Typeを参照し、この共有Typeを拡張しない。

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

`GameSystemDependencyEdgeV1`はTarget Systemまたはpublic Port contract ref、edge kind、Contract Type ref、phase relation、delivery、required、fallbackを持つ。edge kindは`build_link`、`cook_input`、`snapshot_read`、`command_target`、`event_subscription`、`port_target`、`presentation_feed`、`authoring_reference`である。`port_target`はproducerの`emitted_port_message_type_refs[]`とreceiverのpublic Port input Typeを同じexact refへ束縛し、Target Systemのprivate implementationやowner固有proposal schemaを参照しない。

Build／Cook edgeはDAGを必須とする。同一advanceのCommand cycle、同phase再入、callback同期逆呼出し、Presentationからauthoritative判断への逆入力を拒否する。next-boundary cycleを許可する場合は各edgeのqueue contribution、latency上限、overflow failure、Replay fixtureを必須とする。Runtime phaseの定義と実行順自体はRuntime Ownerが所有する。

active Spec集合から`GameSystemDependencyGraphV1`をContract compilerが生成する。GraphはContract set hash、Project revision、System ref／derived ID／owner layer／owner ref／origin／scope、Edge／Type／phase／delivery、State owner table、Build／Cook order、producer canonical order、Save／Replay closure、Target別active Variantを持つ。手動編集しない。

System dependencyのlayer legalityは次のclosed matrixで検証する。`required_feature_pack_refs[]`等で宣言済みでも表外edgeを許可せず、FixtureをProduction graphのowner layerにしない。

| source `owner_layer` | 許可target layer | 常時禁止 |
|---|---|---|
| `core` | `core` | Feature／Genre／Project／Fixture |
| `feature_pack` | `core`、自身またはManifest DAGで宣言したFeature | Genre／Project／Fixture |
| `genre_pack` | `core`、Manifest／Recipeで宣言したFeature | 別Genre／Project／Fixture |
| `project` | `core`、選択済みFeature／Genre、同じProject | 別Project／Fixture |

Contract fixtureはCore→Feature／Genre／Project／Fixture、Feature→Genre、Genre→Genre、全Production→Fixtureを各一原因でrejectする。owner文字列prefixや`system_origin`からlayerを推測せず、`owner_layer`とexact `owner_ref`、Pack／Project containmentを照合する。

`SaveReplayContractV1`はSystem ref、owned State Type、saved／derived Field ref、RNG stream、recorded Command／Event、checkpoint、Migration、unsupported version behavior、state hash policyを持つ。owned authoritative Fieldをsavedまたはderivedのどちらにも分類しないContract、C++ layoutをField IDにするContract、MigrationなしでFieldを削除するContractを拒否する。

## 5. Project-defined systems

Engine Standard System Catalogは開始点でありWhitelistではない。Projectは同じ`GameSystemSpecV2`、State owner、typed Port、Target、Save／Replay、Testを満たすProject-defined Systemを登録できる。

| Owner layer | ID namespace | Game制作時の変更権限 |
|---|---|---|
| Core | `game_system.engine.<path>` | Contractはread-only。許可時にVariantを置換可能 |
| Feature Pack | `game_system.extension.feature.<pack_namespace>.<path>` | 署名済みFeature baseline内だけ。Game制作AIは生成・変更不可 |
| Genre Pack | `game_system.extension.genre.<pack_namespace>.<path>` | 署名済みGenre baseline内だけ。Game制作AIは生成・変更不可 |
| Project | `game_system.project.<project_namespace>.<path>` | Schema／Risk Gateを通して追加・変更可能 |

ID構文とProject pathは[Naming／Project layout](../02-foundation/naming-project-layout.md)を参照し、本書で別規則を作らない。`owner_layer`、`owner_ref.owner_id`、ID namespaceは上表で相互一致し、Feature／Genre ownerが`game_system.engine.*`を使用するSpecをrejectする。Display name、localized title、Genre名、Manager／Controller／Service suffixをidentityまたは責務判定に使わない。State owner、semantic role、Requirement、Portが責務を決める。

`SystemImplementationSetV1`はProject／Targetに対して各active Public Contractを厳密に一つのVariantへ束縛するAuthoring sourceである。Project revision、Contract set hash、Target Profile、Entry、State owner table hash、expected Dependency Graph hash、fallback set refを持つ。EntryはSystem ref、Variant ID／kind、Definition set ref、Native module revision ref、configuration ref、exact `GameSystemActivationBindingRefV1`、same-contract fallbackを持ち、raw Qualification Receipt refまたはFixture refを保持しない。

EntryのSystem refとActivation Bindingが解決する`GameSystemActivationBindingV1.system_ref`、Public Contract hash、Project／Target closureはbyte equalityでなければならず、Bindingが指すsigned Receiptのsubjectも同じSystem／Contract／Targetを持つ。別Systemまたは別Targetの有効Binding、stale Binding version／hash、missing Bindingを拒否する。同じSystemの重複、active System欠落、未Qualified Variant、Target不一致、Graph hash不一致、DefinitionとNativeの二重writerをCompile errorにする。Project-defined replacementがEngine Standard ownerを置換する場合も同時activeにしない。

`GameSystemCatalogV1`はactive Receipt-free MCD、Project Profile、Capability Manifest、root外`GameSystemActivationBindingRefV1`から生成するread-only projectionである。System refとActivation Binding refを別Fieldにし、origin、semantic role、localized summary、Capability maturity、Qualified Target、Implementation kind、required Capability、Budget summary、failure、positive／negative example、Contract hashを持つ。Catalog掲載とactive実装を混同せず、bindingが指すsigned Qualification Receiptで適格化されていないTargetを成功候補として返さない。

## 6. AI planとcode generation

AIはGameSpecからsemantic roleを抽出し、Catalogを検索し、候補Systemだけを読む。必要なCapabilityだけを追加取得し、既存composition、Project Definition、prequalified Pack、bounded Project Sourceの順に比較する。全System、全Schema、全Backend資料を一つのPromptへ投入しない。未知IDをfuzzy補正せず、候補IDとcurrent Contract hashを持つDiagnosticを返す。

`SystemImplementationPlanV1`はPlan ID、Project revision、Contract set hash、System ref、Requirement、Target、authoring profile、candidate Variant、selected Variant、implementation origin、unmet Capability、Behavior Budget ref、Benchmark／equivalence fixture、Save／Replay impact、Build impact、Risk、assumption、rejected alternative、fallback、optional Code owner assignment ref、dispositionを持つ。

`selected_variant`は`gameplay_definition`、`native_game_module`、`hybrid`、`target_specialized_set`のいずれか、`implementation_origin`は`project_definition | prequalified_pack | project_source`のいずれかである。`disposition`は`ready_to_stage`、`awaiting_code_owner`、`question_required`、`capability_unavailable`、`budget_missing`、`rejected`であり、`ready_to_stage`はCommitまたはPromotion承認ではない。

初心者へC++かDefinitionかを質問しない。`authoring_profile=beginner`ではDefinition-firstとし、`implementation_origin`を`project_definition`またはexact Qualification Receiptを持つ`prequalified_pack`に限定する。どちらでも成立しない場合は`capability_unavailable`で停止し、Native／Shaderを暗黙生成しない。`authoring_profile=advanced`で`project_source`を選ぶPlanは、Native moduleなら`role.code_owner.native_module`、Project Shaderなら`role.code_owner.project_shader`を持ち、closed 9-Field subject、exact scope、current Qualification、`revoked_at=null`が成立する`CodeOwnerAssignmentV1`を必須にする。Policy Serviceが信頼済みrevocation registryの署名済みlatest headをread-backし、Assignment Recordとsubject identityのどちらもcurrent snapshotでactiveな場合だけ受理する。Role欠落／unknown、Source kindとのRole不一致、Scope外、期限切れ／失効、snapshot未検証では`awaiting_code_owner`とし、Source Workerを起動しない。AIはGame要件の不足だけを質問し、実装方式はPlanへ根拠付きで記録する。上級者は同じSystem BundleからGraph、Table、Form、Source、Profilerを開く。人間が編集したFieldまたはSource hunkをAIが無条件に再生成しない。

External Agentが同じPlanを提案しても、Host／Model Conformance、Caller Authorization、Project Source Activation、Code owner、G0–G7、Promotionは別Gateである。standard MCPの`StandardExternalProposalReceiptV1`をSource生成、Code owner Approval、Activationへ読み替えない。

Contract compilerは`GameSystemSpecV2`から次を決定論的に生成する。

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
  qualification_receipt_refs[]:
    exact owner-typed signed Qualification Receipt ref/version/content hash
  implementation_plan_hashes[]
  code_owner_assignment_refs[]
  code_owner_approval_refs[]
  dependency_graph_before_hash
  expected_dependency_graph_after_hash
  required_gate_ids[]
  risk_class
```

Bundle／ProjectはUUIDv7 `StableId`、Systemはexact `GameSystemContractRefV1`、ChangeSet／Artifact／PlanはSHA-256で参照する。Source本文、Asset binary、巨大JSONを埋め込まず、Staging artifact hashとBroker管理relative pathだけを参照する。Fixture bodyはowner-typed Qualification recordだけが解決し、Bundleはsigned Receiptのsubject／owner／System／Target／input closure／result／freshnessだけを検証する。全参照は同じProject、Contract set、base revisionへ解決しなければならない。

```text
Draft -> Resolved -> Staged
  -> AwaitingReview -> CommittingProject                 [Sourceなし]
  -> PrepromotionValidating -> BuildingPrepromotionArtifact
     -> TestingPrepromotionCandidate -> AwaitingCodeOwner
     -> AwaitingReview -> PromotingSource -> CommittingProject
                                                            [Native／Shader Sourceあり]
CommittingProject -> FinalValidating -> FinalCooking
  -> BuildingFinalGameCandidate -> TestingFinalCandidate
  -> Packaging -> Qualified

Source promote完了前の各非終端state -> FailedBeforeActivation | Superseded
Source promote完了後かつProject Commit前の失敗 -> InactiveSourcePromoted
Project Commit完了後の最終pipeline失敗 -> CommittedTargetBlocked
```

失敗遷移はSource PromotionとProject Commitの二境界で判定する。`PromotingSource`中にpromoteが完了しないまま失敗した場合は`FailedBeforeActivation`へ、promote完了後かつProject Commit前の失敗は`InactiveSourcePromoted`だけへ遷移する。Project Commitが`N -> N+1`を完了した後の`FinalValidating`以降の失敗は`CommittedTargetBlocked`へ遷移し、Project `N+1`とSource promotion headを維持する。`RetryProjectActivation`と`RevertProposed`はstateではなく`InactiveSourcePromoted`からの遷移actionであり、前者は同一hashで`CommittingProject`へ再入し、後者はReview済みrevertを別のBundleとして提案する。`RetryFinalQualification`は`CommittedTargetBlocked`から同じProject `N+1`、Candidate root、Targetで`FinalValidating`へ再入するactionであり、Project revisionを進めない。

Native／Project Shader Sourceを含まないBundleはprepromotion Source Build／Test、`AwaitingCodeOwner`、Source promotionを通らずReview後にProject Commitへ進む。Sourceを含むBundleは、Source kindに一致するexact `role_ref`、Scope、Qualification、active期間、`revoked_at=null`を持つAssignmentがなければ`PrepromotionValidating`へ進まず、base Project revision `N`のまま停止する。全Source ChangeSetのexact DiffとCandidate Source revisionに対するprepromotion Build／Candidate Testを完了し、そのReceipt集合へ`CodeOwnerApprovalV1.decision=approved`が揃った後だけ`AwaitingReview`からPromotionへ進む。Sourceを含むBundleだけが二段階Activationを必須とする。どちらの経路もProject Commit後の最終pipelineを省略せず、`Qualified`だけをactive Implementationとして表示する。Source promotion済みでもProject revisionが参照しないSource、または最終Packageまで成功していないTargetをGameHost、EditorHost、Shippingへloadしない。

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
- current Catalog fixtureは`entries[5..4096]`、Core exact 5の7-Field row、canonical sort／hash、extension owner登録、Catalog materialization、owner removal、unknown owner、owner unavailable／removed、duplicate、instance-key mismatch、version／Save Replay schema hash mismatchを検証する。`GameSystemSpecV1` version 1の`runtime_instance_scope`→`GameSystemSpecV2` version 2の`runtime_scope_type_ref` clean migrationとmigration conflict fixtureはcurrent suiteへ含めず、signed legacy evidence gate成立後のActivation Qualificationだけへ同じtransactionで追加する。
- Core Systemからextension-owned scopeへの依存、cross-owner未宣言依存、owner namespace偽装を拒否し、同一owner内の登録済みscopeだけを解決するfixture。extension固有scope IDとpositive fixtureは当該ownerが保有する。
- `InteractionSpaceSemanticRegistryV1`の初期Contribution exact三件、owner／namespace／hash、Projection／Qualification set equalityを検証し、CollisionなしProjectでlogical／uiだけを選択・実行でき、spatialだけがCollision Capability／range／Query／LOSを要求するfixture。branch／payload schema mismatch、Projection非選択semantic、logical／uiの空間Field偽装を拒否する。neutral Interactionはeligibility policyなしでvalid、registered generic policy denialは`policy_denied`、任意extension policy未installでもcommon Interaction install／実行が成功する。
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

System Bundleはbase Project revision `N`、base Source revision、Contract set、`PreparedCandidateRefV1`、Candidate rootをlockする。Native／Project Shader Sourceがある場合はProject `N`のままprepromotion Validate／Source Build／Game Candidate Build／Candidate Test、independent review／Code owner Approval、Source promotionの順に進み、該当Project Source revisionを登録する`ProjectChangeSetV1` Commitでだけ`N -> N+1`へ進める。Sourceがない場合はReview後に同じProject Commitへ進む。Commit成功後は両経路ともcommitted Project revision `N+1`を入力に最終Validate／Cook／Game Candidate Build／Candidate Test／Packageを新たに実行し、その全成功Receiptをread-backしてから`Qualified`へ進む。prepromotion ReceiptはPromotion closureのexact Evidenceとしてだけ保持し、最終stageの同種Receiptへ読み替えない。trial Cook／PackageまたはProject `N`のArtifactを最終Package artifactとして流用しない。

Project Commitは[Project state](project-state.md)のtransactionを使い、Graph、Contract set、Implementation set、promoted Source revision、Candidate rootを一つの`N -> N+1`へ束縛する。最終Cooked packageと最終Build／Test Receiptはpost-commit Project `N+1`から生成するためProject Commitのatomic payloadへ含めず、Definition、Source、Asset、Migrationだけを部分Commitしない。最終pipelineのhashまたはrevisionが一致しない場合はProject `N+1`をrollbackせず対象Targetを`CommittedTargetBlocked`にし、直前revisionのPackageを新revisionの成功結果へ流用しない。

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
| prepromotion Validate／Build／Test失敗 | Commitなし、active implementation不変 |
| Source promotion後Project失敗 | inactive Source保持、retry／revert提案 |
| Project Commit後の最終Validate／Cook／Build／Test／Package失敗 | Project revisionとSource promotionを維持し、対象Targetを`CommittedTargetBlocked` |
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

置換後の正規概念は`GameplayDefinition`、`CookedGameplayPackage`、`GameplayStateStore`、`GameSystemSpecV2`、`SystemBundleChangeSetV1`、`NativeGameModule`である。将来unrestricted scriptingを検討する場合は、本設計の暗黙拡張ではなく、新しいProduct decision、Threat Model、Memory／Performance、Editor／Debugger、全Target gateを必須とする。現行Capabilityとしては`not_activated`であり、Runtime interfaceを実装しない。
