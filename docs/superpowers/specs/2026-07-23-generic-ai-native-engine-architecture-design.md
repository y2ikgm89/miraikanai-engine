# Generic AI-Native Engine Architecture Design

- 作成日: 2026-07-23
- 状態: ユーザー承認済み設計。Architecture正本へ反映中
- 対象: Engine Core、Pack、Game Project、World／Runtime、Gameplay、AI Discovery、Product Qualification
- 非対象: Engine実装、Capability Activation、実機Qualification、Release／Shipping実行

## 1. 結論

Miraikanai Engineは、Shooterを共通基盤として一般化する設計を採らない。製品構造を次の4層へ固定する。

1. `Generic Engine Core`
2. `Reusable Feature Packs`
3. `Genre Packs`（任意）
4. `Game Projects`

`Reference Games`は第5のRuntime層ではない。各層を検証するために選定された通常のGame Projectであり、Production RuntimeからFixtureまたはReference Gameへの依存を禁止する。

AI Control PlaneはGameplay層ではなく横断Control Planeである。Editor内AI、first-party local inference、cloud Provider、外部MCP Client、CLIは、同じMiraikanai Contract Definition（MCD）、型付きOperation、Validation、Staging、Receiptを使用する。

Shooter-firstは実装順序として維持できるが、Core、Editor、AI、Project C++、Project Shader、Build、Test、Package、Network、Releaseの成立条件にShooter Packを置いてはならない。

## 2. 層と依存規則

依存方向は利用側から提供側への一方向とする。

```text
Game Project
  -> optional Genre Pack
  -> Feature Pack
  -> Generic Engine Core
```

次を機械検査する。

- CoreはFeature Pack、Genre Pack、Game Project、Reference Game、Fixtureを参照しない。
- Feature PackはCoreおよび別Feature Packだけを参照できる。Feature間依存はDAGでなければならない。
- Feature PackはGenre Pack、Game Project、Reference Game、Fixtureを参照しない。
- Genre PackはCoreとFeature Packだけを参照できる。Genre Pack同士を直接依存させない。
- 複数Genreの合成はGame Projectが行い、Genre Pack間の隠れた推移依存にしない。
- Game ProjectはGenre Packを使わずFeature Packを直接構成できる。
- Reference Game／Fixtureは検証入力として各層を利用できるが、Production artifactへ逆流しない。
- Profileは独立Packではなく、所有Packのversion／hashに含まれる構成単位とする。

## 3. Pack契約

`DomainPackManifest`を廃止し、正規型を`PackManifestV1`とする。`pack_kind`は`feature | genre`のclosed enumとする。

```text
PackManifestV1
  pack_id
  pack_version
  pack_kind: feature | genre
  content_hash
  minimum_engine_contract_ref
  supported_target_profile_refs[]
  required_capability_refs[]
  required_feature_pack_refs[]
  provided_capability_refs[]
  public_contract_refs[]
  component_schema_refs[]
  game_system_spec_refs[]
  authoring_operation_refs[]
  runtime_port_refs[]
  configuration_profile_refs[]
  composition_recipe_refs[]
  source_template_refs[]
  validator_refs[]
  test_scenario_refs[]
  example_refs[]
  counterexample_refs[]
  ai_vocabulary_refs[]
  ai_planning_recipe_refs[]
  performance_profile_refs[]
  migration_step_refs[]
  license_ref
  provenance_ref
```

Pack全Recipeに共通するFeature依存だけを`PackManifestV1.required_feature_pack_refs[]`へ置く。選択したcompositionだけが必要とする条件依存は次の正規型へ置く。

```text
CompositionRecipeV1
  recipe_id
  recipe_version
  recipe_hash
  owner_pack_ref
  required_capability_refs[]
  required_feature_pack_refs[]
  configuration_profile_refs[]
  game_spec_template_refs[]
  action_role_set_refs[]
  source_template_refs[]
  validator_refs[]
  qualification_fixture_refs[]
  fallback_recipe_ref: CompositionRecipeRef | null
```

選択Recipeのeffective closureはManifest共通Feature、Recipe required Feature、両者の推移Feature DAGの和集合である。canonical Recipe／owner Pack／resolved Pack version＋hashから`closure_hash`を生成し、Preview、Project ChangeSet、Cook、Qualification、Save／Replayへ伝播する。未選択Recipeの条件依存はinstall closureへ追加しない。missing、version／hash conflict、Target不適合、unqualified dependency、fallback cycleはRecipe applyを原子的に拒否し、last-valid Recipe activation、Project revision、registry、closure hash、Artifactを維持する。

`PackManifestV1.validator_refs[]`と`test_scenario_refs[]`はPack内で利用可能なValidator／fixtureのidentity inventoryであり、全Recipe共通の実行gateではない。Recipe apply／qualificationで実行するのは選択済み`CompositionRecipeV1.validator_refs[]`と`qualification_fixture_refs[]`だけである。未選択RecipeのValidator、fixture、条件依存を選択Recipeのclosureまたはgateへ追加しない。

Feature Packは再利用可能なPublic Contract、Schema、Validator、Runtime Port、AI vocabulary、reference implementation、contract fixtureを提供する。Genre PackはFeature Packを組み合わせるcomposition recipe、Genre vocabulary、GameSpec template、Profile、reference scenarioを提供する。Genre Packは新しい汎用Core契約を作らない。

Shooter Packは`pack_kind=genre`とし、次のFeature Packを必要に応じて構成する。

- Combat: Damage、Vital、Faction
- Ranged Combat: Weapon、Shot、Projectile
- Encounter／Spawn
- Scoring
- Pickup／Grant
- Interaction
- Character Locomotion
- Path Following
- Scenario／Stage Flow（有限Gameplay用Recipeだけの条件依存）

`Ready | Playing | Paused | Result`、Shooter Action Role、Shooter固有Camera／Audio／LOD ProfileはShooter Genre Packが所有する。

Shooter manifestの全Recipe共通依存はRanged CombatとそのFeature DAGだけとする。Encounter、Scoring、Pickup、Interaction、Character Locomotion、Path Following、Scenario／Stage、Perceptionは、それらを使用するRecipeの`required_feature_pack_refs[]`／`required_capability_refs[]`へ置く。AI敵、移動、Score、Pickup、finite Stageを持たないminimal Shooter recipeをvalidとし、既存2D／TPS reference Recipeは従来のeffective closureとPerception bindingを維持する。

minimal target-practiceは既存`genre.shooter.top_down_2d | genre.shooter.third_person_3d`のexact 1件とPack-owned immutable `ShooterTargetProviderTemplateV1`を選ぶ。Production bindingはowner-typed `ShooterTargetProviderBindingDocumentV1`、qualification fixtureはProject Documentではない`ShooterTargetProviderFixtureBindingRecordV1`として分離し、Profile／templateへProject identityを埋め込まない。Productionはcurrent Projectとexact equalityのtagged `project_owned` owner、binding／template／implementation System／Save Replay ref／version／hashを必須とし、cross-project／self-assert spoofを拒否する。`fixture_only`はexact Fixture Registry ownerだけを許可しProduction Source／Registry／Save／Packageへ昇格しない。Ranged Combat ownerのMCD refsは`type.feature.ranged_combat.collision_query_port`と`type.feature.ranged_combat.shot_hit_event` version 1である。create／update／select Operation、Binding Registry、Save／Replay、Compile Manifestの`selected_provider_binding_set_hash`を閉じ、World／Physics／PerceptionなしProduction Project provider fixtureまで検証する。

次の旧identityを廃止する。

| 旧identity | 移行先 |
|---|---|
| `capability.gameplay.shooter_core` | 上記Feature Capabilityの明示集合 |
| `domain.action_2d` | `genre.shooter.top_down_2d` |
| `domain.tps_single_player` | `genre.shooter.third_person_3d` |
| `DomainPackManifest` | `PackManifestV1` |
| `domain_composition_profile_refs[]` | `composition_recipe_refs[]` |

## 4. Project、World、Scenario

### 4.1 Runtime entry

`ProjectManifest`の単数`root scene`前提を廃止する。各Projectは`runtime_entry_point_refs[1..64]`を持ち、各active Targetでdefault entryを厳密に一つ解決する。

```text
RuntimeEntryPointV1
  entry_point_id
  entry_kind: world | ui | headless
  target_selector_ref
  default_for_selected_targets: bool
  world_ref: WorldDocumentRef | null
  ui_document_ref: UiDocumentRef | null
  startup_game_system_refs[]
  activation_policy_ref
```

`runtime_entry_point_refs[]`はexact `DocumentRef<RuntimeEntryPointDocumentV1>`であり、entryは本文をManifestへ埋め込まない。`RuntimeEntryPointDocumentV1`、`RuntimeTargetSelectorDocumentV1`、`RuntimeEntryActivationPolicyDocumentV1`は共通Document headerの`relative_path`と`content_hash`を持つProject-owned Sourceである。entryの`target_selector_ref`と`activation_policy_ref`は各Documentへのexact ref／schema version／content hashで解決する。

三Documentは`DocumentRef.stable_id == header.document_id == payload.entry_point_id | selector_id | policy_id`を必須とする。`selected_runtime_entry_point_hash`はDocument hashでなく、hash Fieldを持たない`RuntimeEntryPointV1` payloadのMCD canonical semantic hashだけを意味する。Document hashはrefの`content_hash`、selector／policy hashは各hash Field自身を除くpayload hashであり、相互代用しない。

```text
RuntimeTargetSelectorV1
  selector_id
  selector_version
  selector_hash
  target_profile_refs[1..64]

RuntimeEntryActivationPolicyV1
  policy_id
  policy_version
  policy_hash
  readiness_timeout_ms: uint32[1..120000]
  failure_semantics: reject_activation_keep_last_valid | fault_session_reverse_teardown
  cancel_semantics: preparing_before_first_publish | not_cancelable
  explicit_deactivation_semantics: graceful_reverse_teardown | immediate_reverse_teardown
```

selectorはexact `DocumentRef<TargetProfileDocument>`だけをStable ID byte順で持ち、wildcard、tag、表示名、platform名、active Targetの現在値によるlookupを許可しない。duplicate、missing／removed、schema／hash不一致、明示選択Targetの非membershipをtyped rejectする。entry／selector／policyのcreate／updateとroot scene migrationの七OperationはProject closed CatalogとMCD Operation Registryへ、canonical `operation.*` ID、`McdContractRefV1` input／output、AuthoringCommandGateway authority、R2／R3、Authoring side effect、key付きidempotency、authoring transaction、pre／postcondition、列挙error、30,000／120,000 ms timeout、Preview／Validation／Commitを結ぶ`RuntimeEntryMutationReceiptV1`まで完全登録する。

tagged branchは次を強制する。

- `world`: `world_ref`だけをbranch固有必須refにし、`startup_game_system_refs[0..128]`を許可する。
- `ui`: `ui_document_ref`だけをbranch固有必須refにし、`startup_game_system_refs[0..128]`を許可する。Worldは不要。
- `headless`: `startup_game_system_refs[1..128]`を必須にし、World／UI／surfaceを要求しない。

各active Targetについて`default_for_selected_targets=true`のentryだけを対象に`target_selector_ref`の被覆を検査し、default 0件または2件以上を拒否する。default selector集合とactive Target集合はset equalityで一致させ、inactive extra Targetも拒否する。non-default entryのselector overlapは許可し、benchmark、menu、game、server等を同Targetへ複数登録できる。Runtimeはdefaultまたは明示選択を推測せず、実際の選択結果を`selected_runtime_entry_point_ref`、`selected_runtime_entry_point_hash`、`target_selector_hash`、`activation_policy_hash`、`entry_branch_closure_hash`としてCompile Manifestへ保存する。owner-typed Provider Bindingを一件以上選んだ時だけ`selected_provider_binding_set_hash`をcanonical presentにし、entry closureへ入れる。World／Topology／streaming hashは選択branchが`world`で対応Sourceが存在する場合だけcanonical presentとし、`ui`／`headless`ではcanonical omissionにする。

`entry_branch_closure_hash`は次の順序付きcanonical inputだけから計算する。

```text
entry_kind
selected_runtime_entry_point_hash
target_selector_hash
activation_policy_hash
target_profile_hash
game_system_dependency_graph_hash
system_implementation_set_hash
selected_provider_binding_set_hash?
world_document_hash?
ui_document_hash?
spatial_topology_hash?
world_streaming_plan_hash?
startup_system_closure_hash?
```

選択branchの全startup system closureは存在する全推移hard dependency、Implementation Variant、State owner relation、Target compatibilityを含む。startup systemsが空でもbranch rootは上記共通hashとbranch固有hashで確定する。Runtime Packageはこのclosureを格納する外側Artifactであり、Package自身のhashをclosure inputへ含めない。未選択branch Fieldや未選択Recipe inventoryも含めない。

`PlayPreparing`は選択entry、Runtime package、System Graph、Target Planと、そのbranch固有closureだけを検証する。`world`はWorld closure、`ui`はUI closure、`headless`はstartup system closureを持つ。branch activation setの準備後に、runtime session配下へWorld instance、UI session、startup systemsをbranch別optionalとして生成する。`Starting`はbranch activation setを準備し、選択された場合だけPresentation child／surfaceを準備する。`surface_unavailable`はApplication stateでなくoptional Presentation child stateで、simulationを暗黙停止しない。`surface_generation`はsurfaceを持つPresentationだけのtagged fieldであり、headless branchへ偽surfaceを作らない。Host／Platform control threadがouter loopを所有し、window threadを暗黙ownerにしない。Input SourceとPresentationが未登録ならinput／render workを省略し、strict headlessではWindow／Surface／RenderSnapshot／Render thread dependencyを0件にする。

stop、fault、restartは実際にactivateしたstartup／UI／World／presentation dependency DAGを常にreverse teardownする。activation policyの明示deactivation値はgraceful／immediateな要求だけを選び、stop／fault時のteardown省略や依存順の再定義には使用しない。Project Document identity／selector-policyの二fixtureとRuntime branch activation／reverse teardownの二fixtureを`RuntimeEntryOwnerIntegrationManifestV1`のexact ref／hash mappingで束ね、`fixture.integration.project-runtime-entry.owner-resolution`を合格させる。

旧`root scene`はmigration stagingで`default_for_selected_targets=true`の明示的な`world` entryへ変換し、各active Targetへdefaultを厳密に一件生成する。実行時の暗黙migration、`Level` alias、`ui`／`headless`への近似変換を禁止する。

### 4.2 Generic World

`WorldDocumentV1`は空間、Scene、global composition、persistent entity、任意のspatial topologyだけを所有する。

- `scene_document_refs`は`0..65,535`とし、procedural-only Worldを許可する。
- `spatial_topology_definition_ref`は`0..1`とする。
- Level、Objective、Completion、Encounter、player／party transferをCore Worldから除去する。
- World activation、Scene activation、Cell streamingはObjectiveやResultを要求しない。

`SpatialTopologyDefinitionV1`は`space_nodes[]`、`transition_edges[]`、`activation_entry_refs[]`を持つ。entryは0件を許可し、到達不能spaceは`intentionally_isolated=true`で明示する。World ownerはMCD `type.world.spatial_transition_destination` version 1を登録し、exact World／Topology／Edge／target Space ref、各version／hash、Edge policyに従うoptional／required Anchorを一recordへ閉じる。transition Requestはそのtyped refを参照しFieldを複製しない。transition payloadはtyped `transfer_subject_refs[]`を使用し、Player／Partyを固定しない。

Streaming interestはclosedなPlayer enumではなく、owner登録済みtyped interest-source contract refの集合を使用する。observer、entity、camera、scripted anchor等を登録可能とし、unknown／removed ownerを拒否する。

`WorldAuthoringPlanV1.affected_world_refs`は既存World編集branchで1～64件、新規World作成branchでは`create_document_kinds`がWorld作成を厳密に一件宣言する時だけ0件を許可する。新規Worldの`WorldAuthoringContextV1`はCommit成功後にexact `world_ref`付きで生成し、未発行IDを推測しない。

`ProceduralWorldDefinitionV1`と`WorldAuthoringPlanV1`は同じ`selected_validation_provider_bindings[]`のexact binding ref／hash／output MCD schema集合を持つ。truth tableはabsent+empty=skip、absent+nonempty=reject、selected+required output不足=reject、selected+valid=execute／accept、selected+stale／invalid／failure=Delta全rejectである。hard closureへ含めるのはWorld／Tile／Blockoutが明示選択したgeneric dependency bindingだけであり、Renderer／Collision／Navigationを名前で常時列挙しない。同seed／input／Target／Toolchain／binding集合は同じDelta／Artifact hashを再生成し、失敗時はlast-valid Source／Artifact／World generationを維持する。五truth-table fixtureと`fixture.world.procedural-core-only`を必須にする。

### 4.3 Optional Scenario／Stage Feature

有限Stage、entry／exit、Objective、Completionが必要なProjectだけがScenario／Stage Feature Packを使用する。

```text
StageDefinitionV1
  stage_id
  world_ref: WorldDocumentRef | null
  content_source_refs[]
  entry_anchor_refs[]
  exit_anchor_refs[]
  stage_game_system_refs[]
  objective_definition_refs[]
  spawn_definition_refs[]
  transition_policy_refs[]
  completion_mode: none | explicit_outcomes
  completion_contract: CompletionContractV1 | null
  save_replay_policy_ref
  fallback_contract
```

`completion_mode=none`ではcompletion contract、objective、resultを要求しない。Stage ScopeはFeature Packが`scope.feature.scenario_stage.instance`として登録する。Coreは`level_instance`を列挙しない。

`world_ref`はrequired nullableである。world Runtime Entryへ結ぶStageだけがexact World refを必須とし、UI-only／headless Stageは`world_ref=null`とowner-typed `content_source_refs[]`を使用できる。`entry_anchor_refs[]`、`exit_anchor_refs[]`、spatial spawn fieldは`world_ref=null`で0件、world branchでだけ参照kindを検証する。

`spawn_definition_refs[]`はspatial spawnだけに使用し、`world_ref=null`では0件にする。worldless Stageの非空間処理は`stage_game_system_refs[]`とowner-typed contentを使う。`StageContentActivationPolicyV1.content_activation_scope`は`none | entry_anchor_closure | listed_content_refs`であり、`none`はcontent／anchor 0件のsystem-only headless Stageをvalidにする。

Stage transition destinationはMCD `type.feature.scenario_stage.transition_destination` version 2のtagged unionとし、`stage | runtime_entry | world_space | ui | headless | session_end`ごとにexact 1 branchだけを許可する。Request V2はdestinationを持たずexact Policy ref／hashだけを参照する。`runtime_entry`は任意entry、`ui`／`headless`は対応entry kind、`world_space`はworld entry ref／payload semantic hashとWorld-owned `type.world.spatial_transition_destination` refを必須にする。UI Document、headless systems、World／AnchorをStage内で直接指定してRuntime Entry closureを迂回しない。Policy／Request／Destination／PortのMCD ref集合を`StageTransitionContractManifestV1`とPack Manifestでset equality検証し、六kind round-trip、branch外Field、required anchor欠落、stale World／Topology／Space／Edge／hashをfixture化する。`fixture.feature.scenario_stage.worldless-ui`と`fixture.feature.scenario_stage.worldless-headless`はstable IDとしてManifest inventoryと選択qualificationへ登録する。

## 5. Gameplay、Scope、Locomotion

### 5.1 System scope

旧版`GameSystemSpecV1.runtime_instance_scope`のclosed enumを、現行`runtime_scope_type_ref: RuntimeScopeTypeRefV1`へ置換する。

Core登録Scopeは次に限定する。

- `scope.core.application`
- `scope.core.runtime_session`
- `scope.core.world`
- `scope.core.entity`
- `scope.core.ui_session`

```text
RuntimeScopeTypeCatalogV1
  catalog_id: runtime_scope.catalog.active
  catalog_schema_version: 1
  catalog_version
  catalog_hash
  contract_set_hash
  dependency_registry_ref: RuntimeScopeDependencyRegistryRefV1(
    registry_id, registry_revision, registry_content_hash)
  dependency_registry_hash
  entries[5..4096]:
    scope_type_ref: RuntimeScopeTypeRefV1(scope_type_id, scope_type_version, scope_type_hash)
    instance_key_schema_ref: McdContractRefV1(kind=type)
    owner_ref: RuntimeScopeOwnerRefV1(owner_id, owner_revision, owner_content_hash)
    lifetime_ref: McdContractRefV1(kind=policy)
    save_replay_policy_ref: McdContractRefV1(kind=policy)
    activation_condition_ref: McdContractRefV1(kind=policy)
    deactivation_condition_ref: McdContractRefV1(kind=policy)
```

Core entryは次のexact 5 rowと完全一致する。

| `scope_type_ref` | `instance_key_schema_ref` | `owner_ref` | `lifetime_ref` | `save_replay_policy_ref` | `activation_condition_ref` | `deactivation_condition_ref` |
|---|---|---|---|---|---|---|
| `scope.core.application` | `type.runtime_scope.key.application_singleton` | `owner.core.runtime` | `policy.runtime_scope.lifetime.process` | `policy.runtime_scope.save_replay.application_none` | `policy.runtime_scope.activation.process_started` | `policy.runtime_scope.deactivation.process_stopping` |
| `scope.core.runtime_session` | `type.runtime_scope.key.runtime_session_uuidv7` | `owner.core.runtime` | `policy.runtime_scope.lifetime.runtime_session` | `policy.runtime_scope.save_replay.runtime_session` | `policy.runtime_scope.activation.entry_ready` | `policy.runtime_scope.deactivation.stop_or_fault` |
| `scope.core.world` | `type.runtime_scope.key.world_instance` | `owner.core.world` | `policy.runtime_scope.lifetime.world_instance` | `policy.runtime_scope.save_replay.world` | `policy.runtime_scope.activation.world_branch_ready` | `policy.runtime_scope.deactivation.world_branch_teardown` |
| `scope.core.entity` | `type.runtime_scope.key.entity_stable_id` | `owner.core.runtime_ecs` | `policy.runtime_scope.lifetime.entity` | `policy.runtime_scope.save_replay.entity_owner_state` | `policy.runtime_scope.activation.entity_created` | `policy.runtime_scope.deactivation.entity_destroyed` |
| `scope.core.ui_session` | `type.runtime_scope.key.ui_session_uuidv7` | `owner.core.ui` | `policy.runtime_scope.lifetime.ui_session` | `policy.runtime_scope.save_replay.ui_session` | `policy.runtime_scope.activation.ui_branch_ready` | `policy.runtime_scope.deactivation.ui_branch_teardown` |

表はID表示だけで、保存値はscope version／hash、MCD version／Contract set hash、owner revision／content hashを全cellに持つ。active Owner／Dependency Registryは全refを実体recordとして解決し、Registry ref／hashのexact equalityとself-excluding canonical Registry hashを検証する。entryはscope ID byte順にsortし、Catalog hashはdomain separator、catalog ID、schema／catalog version、Contract set hash、Dependency Registry ref canonical bytes／同値hash、entry count、七typed refのcanonical bytesをexact inputにして自己hashを除外する。Feature Packは`scope.feature.<feature>.instance`、Genre Packは自身の内部だけで使用する`scope.genre.<genre>.<scope>.instance`を登録できる。Scenario Stage、Encounter、Scoring、Shooter Game Flowは各owner文書のexact 7-Field rowを登録し、Shooter Game Flowのexact scope IDは`scope.genre.shooter.game_flow.instance`である。Core／FeatureからGenre scopeへの依存、duplicate、unknown owner、unavailable／removed owner、各dependency version／hash不一致を四Scope Diagnosticでtyped rejectする。

旧`runtime_instance_scope`はcurrent validatorが常にrejectし、offline `operation.runtime_scope.migrate_game_system_v1_to_v2`だけが旧Contract setを読む。Operationはexact MCD input／output type、OfflineProjectMigrator authority、R3、Authoring transaction、pre／postcondition、120,000 ms timeout、`RuntimeScopeMigrationReceiptV1`を持ち、Source／Save／Replay identity mappingとephemeral generation非移行を記録する。旧enum aliasをcurrent resolverへ残さない。

### 5.2 InteractionとGame Flow

共通Interaction rejectionは`game_flow_disallowed`を持たない。`operation_eligibility_policy_ref`によるowner判定と、genericな`policy_denied`を使用する。Shooter Game FlowはShooter Pack内だけで`scope.genre.shooter.game_flow.instance`を読むInteraction policyを提供する。

`InteractionSnapshotV1@1.rejection_reason=game_flow_disallowed`は`InteractionSnapshotV1@2.rejection_reason=policy_denied`へversioned clean migrationする。@2で旧enumをaliasとして受理せず、owner／policy／schema hashを一意に解決できないSource／Save／Replayはmigrationを拒否してlast-validを維持する。

Saveは登録済みState ownerと`SaveReplayContractV1`が宣言したfieldだけを保存する。共通Runtimeが常にGame Flow stateを保存してはならない。

### 5.3 Locomotion

Navigation Path FollowingをCharacter Motorへ固定しない。

```text
MotionExecutorPortV1
  executor_capability_ref: McdContractRefV1(kind=capability)
  movement_profile_schema_ref: McdContractRefV1(kind=type)
  accepted_intent_schema_refs[1..64]: McdContractRefV1(kind=type)
  transport_message_schema_ref:
    McdContractRefV1(id=type.navigation.motion_executor_intent_batch, version=1, contract_set_hash)
  resolved_motion_schema_ref: McdContractRefV1(kind=type)
  compatibility_predicate_ref: McdContractRefV1(kind=policy)
  failure_diagnostic_refs[1..32]
```

Character MotorはC1で適格化するProviderの一つである。Vehicle、flying、swimming、RTS unit、board token等は別Capability／Providerとして同じPortへ接続できる。Physics CoreはKinematic Character Motorを必須Core機能にせず、Character Locomotion FeatureのEngine-owned reference implementationとして提供する。

Navigationはこの7 Field型、`MotionExecutorProviderCatalogV1`／record、`MotionExecutorIntentBatchV1`の唯一のcanonical ownerである。Catalog recordはprovider identity／version／content hash、tagged owner identity、production／fixture usage、7 Field descriptor、implementation System、Target、Qualification Receiptを持つ。Physics production、board-token／RTS fixtureは同じCatalogへ登録し、fixtureをProductionへ選択しない。Path Followingは`executor_capability_ref`とProvider-owned `movement_profile_ref`を要求し、compatibility policyがintent schema、movement profile、Targetを検証する。path progressは`resolved_motion_schema_ref`の結果だけで判定する。

Batch entriesは1～16件で、各entryが`intent_schema_ref: McdContractRefV1`、payload refまたはinline value identityのtagged union、payload hash、producer Game System ref、proposal IDを持つ。accepted set検査はbatch envelopeでなくentries schema集合だけへ適用する。type spoof、payload hash mismatch、duplicate proposal、noncanonical order、root-motionあり／なしをfixture化する。Gameplay schemaは`type.feature.character_locomotion.gameplay_motion_intent`、Navigationは`type.navigation.movement_intent`、Animationは`type.animation.root_motion_proposal` version 1である。Provider-private派生入力をpublic accepted setへ混ぜない。

`game_system.engine.character_locomotion.binding`はexact MCD共通Envelopeと`runtime_scope_type_ref={scope.core.entity,v1,hash}`を持つfully registered `GameSystemSpecV1`である。Feature-owned Selection Stateを所有し、上記3 proposal schemaをCommandとして受理し、Navigation-owned batchを`emitted_port_message_type_refs`でT40に発行する。Eventとして登録しない。role、3 requirement、dependency、Implementation policy、Save／Replay、budget、fixture、3 invariantは全てversion／content hash付きactive recordを持つ。Animationを含む全proposalはbinding経由とし、TransformまたはPhysics stateを直接writeしない。

Physics Character Motor Providerの7 FieldはCapability `capability.motion_executor.physics_character_motor`、Profile `type.physics.character_motor_profile`、accepted intent exact 3件、transport batch、resolved `type.physics.character_resolved_motion`、compatibility policy、failure diagnosticsで固定する。root-motion execution modeがProviderのaccepted policyにない場合はactivationを拒否する。board-token／RTS fixture Providerも7 Fieldとowner／usage／Target／dimension／diagnosticを完全登録し、Physics dependency 0件を検証する。

`feature.character_locomotion@1`のPack installはPhysicsを要求しない。Physics Character MotorはC1 reference recipe／qualification providerの一つに限る。Animation root motionはCharacter Locomotion bindingを経由する`RootMotionProposalV1`であり、Providerへ直接提出しない。`gameplay_motor`旧modeはprovider-neutral modeへclean migrationする。T40はselected executor、T50はactive Physics providerがある場合だけ実行する。missing／incompatible provider、stale result、provider failureはtyped failureとし、last-valid stateを維持する。

## 6. ClockとPause

60 HzはC1／C2で唯一のqualified baselineとして維持するが、Core schema／phase ABIへliteralとして固定しない。

```text
SimulationCadenceProfileV1
  profile_id
  cadence_kind: fixed | explicit_step
  fixed_rate_numerator_hz: uint32 | null
  fixed_rate_denominator: uint32 | null
  max_catch_up_steps
  overrun_policy
  physics_substep_profile_ref: null | ProfileRef
```

- `fixed`は既約な正の有理数rateを必須にする。
- `explicit_step`はrate fieldをnullにする。
- C1／C2 active Profileは`fixed, 60/1, max_catch_up_steps=4`だけとする。
- alternate rate、explicit step、別physics substepはQualification前に`capability_unavailable`とする。
- tick durationは有理数式から求め、切り捨てnanosecondの反復加算やfloat蓄積を使わない。
- VFX、Input、Replay、TimerはProfile ref／hashを記録し、60を独自に再定義しない。

Pauseは任意Capabilityとする。`PausePolicyV1.support_mode`はC1で`unsupported | global`を許可し、将来のscoped pauseは別Qualificationにする。`support_mode=unsupported`のProjectへPause commandを適用せず、typed `pause_not_supported`を返す。Pause ownerはCoreの`scope.core.runtime_session`を使い、Weapon、Encounter、Shooter Game Flowを共通規約へ記載しない。

## 7. AIが理解できる上位仕様

AIの「理解した」という文章を状態にしない既存方針を維持し、上位入力をfield-level MCDへ閉じる。

### 7.1 `GameIntentDraftV1`

Disposableな入力解決RecordでありProject正本ではない。

```text
GameIntentDraftV1
  draft_id
  project_ref: ProjectRef | null
  source_input_evidence_refs[]
  requested_experience_statements[]
  candidate_requirement_statements[]
  candidate_constraint_statements[]
  ambiguity_refs[]
  unsupported_candidate_refs[]
  created_at
```

### 7.2 `GameBriefV1`

人間確認済みの制作意図を表す。

```text
GameBriefV1
  brief_id
  project_id
  experience_summary
  experience_loop_kinds[]
  audience_requirement_refs[]
  target_profile_refs[]
  control_requirement_refs[]
  presentation_requirement_refs[]
  content_requirement_refs[]
  persistence_requirement_refs[]
  online_requirement_refs[]
  accessibility_requirement_refs[]
  business_and_distribution_requirement_refs[]
  explicit_non_goal_refs[]
  accepted_assumption_refs[]
  decision_refs[]
  unresolved_question_refs[]
```

`experience_loop_kinds[]`は`finite | endless | turn_based | continuous_simulation | tool_like | mixed`のnon-empty subsetとする。Player、Character、Combat、Objective、Completion、World、Genre Packは必須Fieldにしない。

### 7.3 `GameSpecDocumentV1`

```text
GameSpecDocumentV1
  game_spec_id
  game_brief_ref
  requirement_refs[]
  runtime_entry_point_refs[]
  feature_pack_refs[]
  genre_pack_refs[]
  game_system_spec_refs[]
  gameplay_definition_refs[]
  content_plan_refs[]
  test_scenario_refs[]
  budget_profile_refs[]
  target_profile_refs[]
  save_replay_contract_refs[]
  decision_refs[]
  unsupported_capability_refs[]
```

`genre_pack_refs[]`、World、controllable actor、Objective、Completionは0件を許可する。RequirementからCapability、Pack、System、Implementation、Test、Artifactまでの追跡を`GameUnderstandingClosureV1`が検査する。

## 8. AI Catalog、Context、Operation

全公開機能から`AiCatalogEntryV1`を生成する。

```text
AiCatalogEntryV1
  catalog_entry_id
  subject_ref
  subject_version
  subject_hash
  owner_ref
  architecture_layer
  purpose
  non_goals[]
  input_contract_refs[]
  output_contract_refs[]
  state_owner_refs[]
  runtime_scope_type_refs[]
  phase_and_lifetime_refs[]
  read_set_refs[]
  write_set_refs[]
  dependency_refs[]
  incompatibility_refs[]
  target_support_refs[]
  maturity_ref
  budget_refs[]
  allowed_operation_refs[]
  risk_and_approval_refs[]
  diagnostic_and_remediation_refs[]
  example_refs[]
  counterexample_refs[]
  test_fixture_refs[]
  deprecation_and_migration_refs[]
  public_project_sdk_refs[]
```

Pack Discoveryは次のexact OperationをMCDへ登録する。

- `operation.packs.search`
- `operation.packs.read`
- `operation.packs.resolve_dependencies`
- `operation.packs.explain_composition`
- `operation.packs.plan_apply`
- `operation.packs.preview_apply`
- `operation.packs.validate`
- `operation.packs.plan_remove`

`AiTaskContextCapsuleV1`はGame Brief／Spec、選択Catalog entry、Project Slice、依存、制約、Target、Budget、Diagnostic、許可Operation、選択理由、source hash、token estimate、omitted range、continuationを持つread-only／Disposable projectionとする。AIへ全Schema、全Project、全設計書、Security署名内部を一括送信しない。Gatewayがsemantic projectionを返し、Authorization、Provenance、Receipt検証はGateway内で強制する。

Model familyまたはHost brandをEngine機能分岐にしない。同じMCD／Operationへ適合するProvider／ModelをProfileとConformance Receiptで有効化する。Strict Tool Callingへ適合しないModelはread-onlyまたはproposal-onlyに制限し、自然言語をCommit Operationへ推測変換しない。

## 9. Product ScopeとQualification

現時点で未実装のPlatform／大規模機能を、汎用Schemaが存在することだけで対応済みと表示しない。offline single-player、Windows-first、60 Hz baseline等の現在Scopeは許容するが、恒久Core制約にしない。

最低Reference／holdout集合を次へ固定する。

- Shooter top-down 2D: 最初のGenre vertical slice
- Shooter third-person 3D: 同じFeature契約の3D検証
- Puzzle／Dialogue 2D: Combatなし
- Platformer 2D: Character Locomotionの別Genre検証
- Turn-based zero-character: Player Character、Physics、Level completionを要求しない
- Endless continuous simulation: Completion／Resultなし
- UI-only／tool-like: Worldなし
- Headless simulation: Surface／UIなし

Phase 4 AI MVPはShooterだけで汎用性を主張しない。少なくともCombatなし、zero-characterまたはWorldなしの中立Fixtureを一つ同じAI／手動往復Gateへ含める。`general_production_2d`または`general_production_3d`は複数Genre／構造のQualification後だけ表示する。

Network transport、replication、authority、rollbackはGenre-neutral Future Capabilityとする。`small-cooperative-multiplayer`と`rollback-competitive-networking`はShooter Capabilityへ依存しない。`large-session-network-shooter`だけがShooter Genreをconsumerとして参照できる。

Release closureはTarget単位に評価する。Windows／Android／Apple／headlessの個別Releaseを別Target未完了で停止させない。全Target coverage labelは個別Releaseとは別のPortfolio projectionとする。

## 10. 必須Conformance Gate

1. Shooter Packを未installまたは削除した状態でCore、Editor、AI、Project C++、Project Shader、Test、Build、Packageが成功する。
2. Core／Feature文書とProduction dependency graphからShooter Genre IDへの依存が0件である。
3. Genre PackなしでFeature Packだけを使うProjectがvalidである。
4. UI-only ProjectがWorld／Levelなし、headless Projectがsurfaceなしでvalidである。
5. finite、endless、turn-based、continuous simulation、tool-like、zero-characterをGame Brief／Specで表現できる。
6. Path FollowingがCharacter Motor以外のconformance stubへ接続できる。
7. C1は60/1だけをqualifiedに保ちながら、Core schema変更なしに別Cadence Profileを追加できる。
8. Pack追加／更新／削除の依存、影響、migration、fallbackをAIがexact IDで説明できる。
9. Local小型Modelを含むConformanceで、未知IDの最終提出0、`question_required`推測回避0、禁止Operation提出0を満たす。
10. Target-local Release closureと全Target Portfolio labelが別判定になる。

## 11. 公式設計から採用する原則

- Unreal EngineのModule／Plugin依存階層とProject-owned module
- Unity Package Manager／Assembly Definitionのmanifest、dependency、version、sample、test、documentation
- GodotのNode／Scene composition、Resource data、GDExtension、機械可読CLI
- O3DE Gem、Reflection Context、Asset dependency graph

外部EngineのAPI、class hierarchy、serialization、package format互換は採用しない。MiraikanaiのMCD、Stable ID、typed Operation、Gateway、Receiptを正本とする。

## 12. 完了判定

本設計の文書反映だけではEngine実装、Capability Activation、Shippingを証明しない。Architecture正本、Product Registry、current execution plan、AI Eval、Reference Fixture、依存検査が本設計へ同期し、旧identity／旧必須前提へのactive参照が0件になった時点で「計画書更新完了」とする。判定は引き続き`planning_go / implementation_and_shipping_no_go`である。
