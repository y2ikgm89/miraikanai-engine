# Gameplay Generated Projection／Fixture Candidate Catalog

- 文書ID: mirakan.appendix.gameplay-generated-projection-fixture-catalog
- 文書種別: Owner supplement
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 親Owner: [Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)
- 正本範囲: Gameplay schema、Registry、generated projection、migration record、AI generation、fixture、promotionのreview候補詳細
- 非正本範囲: Gameplay Programming Modelの選択境界、Game System意味、State owner、typed Port、Runtime ECS current authority。親Ownerと各Subsystem Ownerが決定する
- 規範依存: [親Owner](../03-authoring/gameplay-programming-model.md)
- 関連文書: [Project State](../03-authoring/project-state.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Runtime ECS](../04-runtime/entity-component-system.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-21

> 本書は分離前Owner文書の具体Schema、Registry、generated projection、fixture候補を保持する。親OwnerのProgramming Model、State ownership、Port意味、Runtime ECS移管条件を上書きせず、生成ArtifactまたはQualification Receiptが存在しない候補をactive、generated、promotedとして扱わない。

> 以下の見出し番号は、親Ownerの論点番号との対応を明示するために維持する。欠番は親Ownerが所有する規範であり、本書に補完しない。

## 分離前Owner節から抽出した候補record

### 2.4 C1 Perception／Interaction

<a id="InteractionDefinitionV1"></a>
<a id="InteractionRequestV1"></a>
<a id="InteractionSnapshotV1"></a>
<a id="InteractionSpaceBindingV1"></a>
<a id="InteractionSpaceSemanticActivationProjectionV1"></a>
<a id="InteractionSpaceSemanticContributionV1"></a>
<a id="InteractionSpaceSemanticRefV1"></a>
<a id="InteractionSpaceSemanticRegistryV1"></a>
<a id="LogicalInteractionBindingV1"></a>
<a id="SpatialInteractionBindingV1"></a>
<a id="UiInteractionBindingV1"></a>

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
| `interaction.space.spatial@1` | `owner.feature.interaction`／`feature_pack` | `spatial`／`{type.feature.interaction.spatial_binding, 1, contract_set_hash}` | exact `GameSystemContractRefV1(game_system.extension.feature.interaction.standard@1)` | `[capability.simulation.collision_query@1]` | `[target.windows.desktop, target.android.mobile, target.apple.mobile]` |
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

`InteractionSnapshotV1@1.rejection_reason`はinitial V1から`policy_denied`をcanonical値とする。過去draftの`game_flow_disallowed`、alias、dual reader、migration fixtureをcurrent planへ登録せず、Source／Save／Replayで旧enum値を拒否する。fixtureはpolicyなしneutral Interaction、Shooter policyありdeny、旧enum直接入力拒否を検証する。

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

Registry row集合またはrecord bytesが変わる場合だけ`registry_version`をexact `N+1`へ進め、同じRegistry versionで別root、同一bytesの不要なversion増加、複数current rootを拒否する。初回公開後のRegistry version更新、Owner revision更新、Contract set／Diagnostic／Game System側のowner ref移行、Foundation Definition Closure更新は、[Governance Migration Proposals](governance-migration-proposals.md#1-post-public-definition-migration-binding-candidate)のOwner reference migration manifest、Compatibility Consumer Inventory、Compatibility Change、全Evidence Requirementのpass satisfaction binding、Definition Migration binding候補を含む一つの承認済みdefinition migrationでset equalityを満たす。initial V1へこのmigrationを適用せず、Owner recordだけ、aliasだけ、説明文だけを先行current化しない。

current初期Registryは`registry_version=1`、`record_count=18`で、全rowが`owner_revision=1`、`status=active`、`authority_source=architecture_document`の次のexact集合である。

| owner_id | owner_layer | authority `document_id` |
|---|---|---|
| `owner.core.asset_lifecycle` | `core` | `mirakan.arch.asset-lifecycle` |
| `owner.core.collision` | `core` | `mirakan.arch.simulation-collision` |
| `owner.core.gameplay_programming_model` | `core` | `mirakan.arch.gameplay-programming-model` |
| `owner.core.navigation` | `core` | `mirakan.arch.simulation-navigation` |
| `owner.core.performance` | `core` | `mirakan.arch.runtime-performance-capacity` |
| `owner.core.physics` | `core` | `mirakan.arch.simulation-physics` |
| `owner.core.project_state` | `core` | `mirakan.arch.project-state` |
| `owner.core.runtime` | `core` | `mirakan.arch.runtime-scheduling-lifetime` |
| `owner.core.runtime_ecs` | `core` | `mirakan.arch.runtime-entity-component-system` |
| `owner.core.security_approval` | `core` | `mirakan.arch.ai-security-approval` |
| `owner.core.ui` | `core` | `mirakan.arch.platform-ui-text-localization-accessibility` |
| `owner.core.world` | `core` | `mirakan.arch.rendering-world` |
| `owner.feature.character_locomotion` | `feature_pack` | `mirakan.arch.simulation-navigation` |
| `owner.feature.encounter_spawn` | `feature_pack` | `mirakan.arch.pack-gameplay-features` |
| `owner.feature.interaction` | `feature_pack` | `mirakan.arch.pack-gameplay-features` |
| `owner.feature.scenario_stage` | `feature_pack` | `mirakan.arch.pack-scenario-stage` |
| `owner.feature.scoring` | `feature_pack` | `mirakan.arch.pack-gameplay-features` |
| `owner.genre.shooter` | `genre_pack` | `mirakan.arch.pack-shooter` |

`owner.core.runtime_ecs`のinitial V1 authority documentは[Runtime ECS](../04-runtime/entity-component-system.md)である。本Catalog候補はそのexact document IDを直接参照し、Gameplay Programming ModelをRuntime ECS authorityとして登録しない。旧Owner revision、source／target authority、Owner-reference migration、aliasまたはdual Registryをinitial V1へ作らない。Catalogの存在はOwner Registry、Foundation Definition Closure、実装またはactive MCDがmaterialize済みであることを意味しない。

`owner.core.ai_security`、`owner.core.runtime_scheduling`、`owner.core.ui_text_accessibility`はcurrent Owner IDではなく、それぞれ`owner.core.security_approval`、`owner.core.runtime`、`owner.core.ui`への説明用旧称である。Registry row、MCD owner、Runtime Scope、Diagnostic、Game Systemへaliasを保存せず、追加Ownerとして数えない。

補助Fieldのarrayは意味上のset、wire上のsorted arrayである。各arrayはrefのlogical ID NFC UTF-8 bytes、version `uint32_be`、content hash bytesの順でstrict sortし、MCD requirement refはID、version、Contract set hashの順でsortする。同じIDのduplicate、同じID／versionの別hash、非canonical orderをrejectし、入力順を意味へ残さない。`implementation_policy_ref`は一件、`save_replay_contract_ref`はenclosing `state_class=authoritative`で一件、それ以外ではField自体をcanonical omissionし、`null`やzero refで代用しない。

`auxiliary_ref_set_hash = SHA-256(ASCII "MIRAKAN_GAME_SYSTEM_AUXILIARY_REF_SET_V1" || uint32_be(length(MCD canonical bytes of GameSystemAuxiliaryRefSetV1 excluding auxiliary_ref_set_hash)) || canonical bytes)`である。各arrayはcountをcanonical bytesに持ち、hash Fieldを唯一のself-exclusion対象にする。`type.game_system.spec` version 1だけが上記Fieldをcurrentとして登録する。全refはactive recordをexactly oneへ解決し、Specにinline payloadを埋め込まない。補助recordの`content_hash`、MCD requirement／type／CapabilityのContract set hash、Scope hash、owner ref、auxiliary set hashのどれか一つでもstaleならSystem Catalog全体をrejectする。

Fixture artifactはProduction `GameSystemSpecV1`／auxiliary hashから分離する。SpecとContract set rootをReceipt-freeで固定した後、次のowner-typed subject、signed Receipt、root外Activation bindingをこの順で生成する。

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

`qualification_subject_hash`はASCII `MIRAKAN_GAME_SYSTEM_QUALIFICATION_SUBJECT_V1`と自己Fieldを除くlength-framed canonical subject bytes、`activation_binding_hash`はASCII `MIRAKAN_GAME_SYSTEM_ACTIVATION_BINDING_V1`と自己Fieldを除くbinding bytesから計算する。Qualification subjectの`owner_ref`、`system_ref`、`system_contract_hash`は`system_ref`が解決するReceipt-free `GameSystemSpecV1`の`owner_ref`、exact Contract ref、self-excluding System contract hashとbyte equality、Activation Bindingの`system_ref`／`system_contract_hash`はReceipt subjectの同Fieldとbyte equalityにする。`GameSystemQualificationReceiptRefV1`の`qualification_id／qualification_version`はwrapper内subjectの同Fieldとbyte equalityで、qualification content identityとsigned record hashは同じwrapperへexact解決する。signed wrapperはAI Verificationの二値規則に従い、`qualification_subject_hash`を含む完成Subject全体のJCS hashを`signed_record.subject_sha256`として署名する。subject payload／hash preimageへwrapperまたはReceipt refを含めない。Production `GameSystemCatalogV1`はReceipt-free Spec refとexact `GameSystemActivationBindingRefV1`を別Fieldで持ち、Bindingが指す署名済みwrapperのsubject、owner、System、Target、contract hash、result=`pass`、freshness／revocationを検証するだけで、`fixture_refs[]`のbody、oracle、pathをRuntime Package、Save、Replay、Production registryへ解決しない。生成順は`Receipt-free GameSystemSpecV1／auxiliary hash → Contract set root／GameSystemContractRefV1 → Qualification subject → signed Receipt → Activation binding／Catalog projection`であり、Receipt、wrapper、Activation bindingをSpec、auxiliary hash、Contract set rootへ戻さない。Fixture集合変更は新subject／Receipt／Activation bindingを発行し、Production Spec hashへFixture refまたはReceipt refを混入しない。正しいSpec refのままsubject owner／contract hashだけを差し替えるcase、BindingのSystem ref／contract hashだけを別subjectへ差し替えるcaseを一原因negative fixtureで拒否する。

Fixture内だけで動くtest double／stub／oracle implementationはProduction `GameSystemSpecV1`を名乗らない。次の別identity、owner、Registry、Qualification contractだけを使用する。

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

`UsageTaggedImplementationSystemBaseRefV1`はReceipt-freeなSystem identityだけを持ち、discriminatorとbranch Fieldをexactに一致させる。`usage=production | project_owned`では`production_system_ref`と`production_system_contract_hash`だけ、`fixture_only`では`fixture_system_ref`だけを必須とし、Qualification Receipt／Activation Binding refを一件も含めない。`UsageTaggedImplementationSystemRefV1`は完成base refにroot外System Activation Bindingを加えるdownstream qualified-dependency refである。前者ではbaseのSystem refをsubjectにする`production_system_activation_binding_ref`だけ、後者ではbaseのFixture System refをsubjectにするexact `fixture_system_activation_binding_ref`だけを必須とし、両branch、両方欠落、discriminator外Fieldを拒否する。production consumerのtyped ownerは参照先`GameSystemSpecV1.owner_ref`とexact equalityにする。fixture consumerは自身の`fixture_owner_ref {fixture_id,fixture_version,fixture_content_hash}`をFixture Registry record、Qualification subject、Activation bindingが解決する同型Fieldとexact equalityにし、`GameSystemOwnerRefV1`またはowner kind enumとの異型比較を行わない。生成順は`receipt-free System base／System ref → System Qualification subject → signed Receipt → System Activation Binding → UsageTaggedImplementationSystemRefV1`であり、downstream refをSystem base、Fixture Registry、Contract set rootへ戻さない。

Fixture Registry、System ref、Qualification Receipt、Activation bindingはQualification sandboxだけが解決する。Production [Gameplay Programming Model §3.0.1](../03-authoring/gameplay-programming-model.md#301-system-graphimplementation-setstate-owner-projection)の`GameSystemDependencyGraphV1`、`SystemImplementationSetV1`、`GameStateOwnerProjectionV1`、`GameSystemCatalogV1`、`GameSystemSpecV1`、Project Source／Compile Manifest、Save、Replay、Cooked／Runtime Packageへ一件も投影しない。fixture refのProduction branch混入、Production `GameSystemContractRefV1`のfixture branch混入、cross-owner、stale Registry／record／subject／Receipt／Activation binding hashを各一原因で拒否し、last-valid Production catalog／packageを不変にする。

一つのSpecはCatalogで解決した一scopeだけを持つ。複数scopeのStateを所有する場合はSystemを分割し、Stable handleまたはtyped Eventで接続する。

### 3.1.2 Scope依存record

`GameSystemSpecV1`は`type.game_system.spec` version 1として、`runtime_scope_type_ref`とowner-typed auxiliary record refを最初から持つ。validatorは`runtime_instance_scope`、legacy enum、bare scope／auxiliary ID、version／hash欠落、current Catalog以外のscope hashを`MIRAKAN-RUNTIME-SCOPE-CATALOG_INVALID`でrejectする。

対応Schema、Project／System bytes、reader／writer、Registry、Operation、Receiptが一度もmaterializeされていないため、過去draftのinline scope、旧`GameSystemSpecV1` migration、legacy Runtime Scope migration Operation、Migration Contribution／Registry／Qualification型、offline Service／Capability／Isolation Profile、aliasをcurrent planへ定義しない。過去表記はimmutable ADRまたはreview transcriptだけに履歴として残し、current Source、Editor、AI projection、Compile、Save、Replayの入力として受理しない。

Scope contractを公開／materializeした後に変更する場合だけ、Compatibility Ownerのconsumer inventoryとversioned migration判断を新しいChangeSetで追加する。現時点のcurrent schema、Catalog、validator、fixtureはinitial V1だけを対象とする。

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

Definition／prequalified Pack経路はSchema、semantic、Capability、State owner、dependency、Target、Budgetを検証し、canonical CookとReference evaluator fixtureを実行する。Packはexact package hash、license、Target、Qualification Receipt、revocationを検証し、変更を加えた時点でprequalified扱いを失う。Native経路は同じPublic Contract conformanceに加え、[Native game module](../03-authoring/native-game-module.md)が所有するSource境界、C ABI、entry、lifecycle、Target link、Build identity、Preview、Promotion、Package gateを通す。Project Shader経路はShader OwnerのSource／Technique Gateに加え、Nativeと同じCode owner binding原則を使う。

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
- current Catalog fixtureは`entries[5..4096]`、Core exact 5の7-Field row、canonical sort／hash、extension owner登録、Catalog materialization、owner removal、unknown owner、owner unavailable／removed、duplicate、instance-key mismatch、version／Save Replay schema hash mismatchを検証する。`GameSystemSpecV1`はinitial schemaから`runtime_scope_type_ref`を必須とし、旧inline scope、legacy migration、migration conflict fixtureをcurrent suiteまたはOperation inventoryへ追加しない。
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

Project Commitは[Project state](../03-authoring/project-state.md)のtransactionを使い、Graph、Contract set、Implementation set、promoted Source revision、Candidate rootを一つの`N -> N+1`へ束縛する。最終Cooked packageと最終Build／Test Receiptはpost-commit Project `N+1`から生成するためProject Commitのatomic payloadへ含めず、Definition、Source、Asset、Migrationだけを部分Commitしない。最終pipelineのhashまたはrevisionが一致しない場合はProject `N+1`をrollbackせず対象Targetを`CommittedTargetBlocked`にし、直前revisionのPackageを新revisionの成功結果へ流用しない。

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
