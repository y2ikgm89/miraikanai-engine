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

minimal target-practiceは既存`genre.shooter.top_down_2d | genre.shooter.third_person_3d`のexact 1件と、version／hash付き`ShooterTargetProviderV1` bindingを選ぶ。Production applyはProject-owned compatible providerのexact ref／hashだけを許可する。組込み`provider.genre.shooter.fixture.stationary_target@1`は2D、fixture-only、deterministic Collision Query／Hit Evidence providerであり、World、Physics、Perception、render visibilityへ依存せず、`fixture.genre.shooter.target-practice-minimal`と`fixture.genre.shooter.target-practice-minimal-no-perception`だけで使用できる。後者はManifest inventoryにPerception Validator／full fixtureが存在してもminimal Recipe gateへ追加されないことを検証する。

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

selectorはexact `DocumentRef<TargetProfileDocument>`だけをStable ID byte順で持ち、wildcard、tag、表示名、platform名、active Targetの現在値によるlookupを許可しない。duplicate、missing／removed、schema／hash不一致、明示選択Targetの非membershipをtyped rejectする。selector／policyのcreate／updateとroot scene migrationは別のversioned Project Operationであり、entry本文へのuntyped field writeで代用しない。

tagged branchは次を強制する。

- `world`: `world_ref`だけをbranch固有必須refにし、`startup_game_system_refs[0..128]`を許可する。
- `ui`: `ui_document_ref`だけをbranch固有必須refにし、`startup_game_system_refs[0..128]`を許可する。Worldは不要。
- `headless`: `startup_game_system_refs[1..128]`を必須にし、World／UI／surfaceを要求しない。

各active Targetについて`default_for_selected_targets=true`のentryだけを対象に`target_selector_ref`の被覆を検査し、default 0件または2件以上を拒否する。default selector集合とactive Target集合はset equalityで一致させ、inactive extra Targetも拒否する。non-default entryのselector overlapは許可し、benchmark、menu、game、server等を同Targetへ複数登録できる。Runtimeはdefaultまたは明示選択を推測せず、実際の選択結果を`selected_runtime_entry_point_ref`、`selected_runtime_entry_point_hash`、`target_selector_hash`、`activation_policy_hash`、`entry_branch_closure_hash`としてCompile Manifestへ保存する。World／Topology／streaming hashは選択branchが`world`で対応Sourceが存在する場合だけcanonical presentとし、`ui`／`headless`ではcanonical omissionにする。

`entry_branch_closure_hash`は次の順序付きcanonical inputだけから計算する。

```text
entry_kind
selected_runtime_entry_point_hash
target_selector_hash
activation_policy_hash
target_profile_hash
game_system_dependency_graph_hash
system_implementation_set_hash
world_document_hash?
ui_document_hash?
spatial_topology_hash?
world_streaming_plan_hash?
startup_system_closure_hash?
```

選択branchの全startup system closureは存在する全推移hard dependency、Implementation Variant、State owner relation、Target compatibilityを含む。startup systemsが空でもbranch rootは上記共通hashとbranch固有hashで確定する。Runtime Packageはこのclosureを格納する外側Artifactであり、Package自身のhashをclosure inputへ含めない。未選択branch Fieldや未選択Recipe inventoryも含めない。

`PlayPreparing`は選択entry、Runtime package、System Graph、Target Planと、そのbranch固有closureだけを検証する。`world`はWorld closure、`ui`はUI closure、`headless`はstartup system closureを持つ。branch activation setの準備後に、runtime session配下へWorld instance、UI session、startup systemsをbranch別optionalとして生成する。`surface_generation`はsurfaceを持つbranchだけのtagged stateであり、headless branchへ偽surfaceを作らない。Host／Platform control threadがouter loopを所有し、window threadを暗黙ownerにしない。Input SourceとPresentationが未登録ならinput／render workを省略し、strict headlessではWindow／Surface／RenderSnapshot／Render thread dependencyを0件にする。

stop、fault、restartは実際にactivateしたstartup／UI／World／presentation dependency DAGを常にreverse teardownする。activation policyの明示deactivation値はgraceful／immediateな要求だけを選び、stop／fault時のteardown省略や依存順の再定義には使用しない。`fixture.integration.project-runtime-entry.owner-resolution`とworld／UI／headlessのstop／fault／restart fixtureでDocument owner、branch owner、reverse teardownを検証する。

旧`root scene`はmigration stagingで`default_for_selected_targets=true`の明示的な`world` entryへ変換し、各active Targetへdefaultを厳密に一件生成する。実行時の暗黙migration、`Level` alias、`ui`／`headless`への近似変換を禁止する。

### 4.2 Generic World

`WorldDocumentV1`は空間、Scene、global composition、persistent entity、任意のspatial topologyだけを所有する。

- `scene_document_refs`は`0..65,535`とし、procedural-only Worldを許可する。
- `spatial_topology_definition_ref`は`0..1`とする。
- Level、Objective、Completion、Encounter、player／party transferをCore Worldから除去する。
- World activation、Scene activation、Cell streamingはObjectiveやResultを要求しない。

`SpatialTopologyDefinitionV1`は`space_nodes[]`、`transition_edges[]`、`activation_entry_refs[]`を持つ。entryは0件を許可し、到達不能spaceは`intentionally_isolated=true`で明示する。transition payloadはtyped `transfer_subject_refs[]`を使用し、Player／Partyを固定しない。

Streaming interestはclosedなPlayer enumではなく、owner登録済みtyped interest-source contract refの集合を使用する。observer、entity、camera、scripted anchor等を登録可能とし、unknown／removed ownerを拒否する。

`WorldAuthoringPlanV1.affected_world_refs`は既存World編集branchで1～64件、新規World作成branchでは`create_document_kinds`がWorld作成を厳密に一件宣言する時だけ0件を許可する。新規Worldの`WorldAuthoringContextV1`はCommit成功後にexact `world_ref`付きで生成し、未発行IDを推測しない。

procedural generator outputは、Worldがexact provider ref／hashを明示選択した時だけ、そのProviderが宣言したoptional Physics source、spawn、Navigation sourceのtyped output schemaとowner availabilityを検証する。該当output refまたはownerが未選択なら検査をcanonical skipし、Core-only procedural Worldをvalidにする。hard closureへ含めるのは明示選択されたProvider outputだけであり、`fixture.world.procedural-core-only`はPhysics／spawn／Navigation providerなしで同一seed再生成hashとlast-valid維持を検証する。

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

Stage transition destinationは`StageTransitionDestinationV1`のtagged unionとし、`stage | runtime_entry | world_space | ui | headless | session_end`ごとにexact 1 branchだけを許可する。spatial anchorは`world_space`、または解決先Stageがworld branchかつentry-anchor closureを要求する場合だけ存在できる。UI／headless／session endへ偽World／anchorを追加しない。`fixture.feature.scenario_stage.worldless-ui`と`fixture.feature.scenario_stage.worldless-headless`はstable IDとしてManifest inventoryと選択qualificationへ登録する。

## 5. Gameplay、Scope、Locomotion

### 5.1 System scope

`GameSystemSpecV1.runtime_instance_scope`のclosed enumを、versioned `RuntimeScopeTypeCatalogV1`参照へ置換する。

Core登録Scopeは次に限定する。

- `scope.core.application`
- `scope.core.runtime_session`
- `scope.core.world`
- `scope.core.entity`
- `scope.core.ui_session`

```text
RuntimeScopeTypeCatalogV1
  catalog_version
  catalog_hash
  entries[5..4096]:
    scope_type_ref
    instance_key_schema_ref
    owner_ref
    lifetime_ref
    save_replay_policy_ref
    activation_condition_ref
    deactivation_condition_ref
```

Core entryは次のexact 5 rowと完全一致する。

| `scope_type_ref` | `instance_key_schema_ref` | `owner_ref` | `lifetime_ref` | `save_replay_policy_ref` | `activation_condition_ref` | `deactivation_condition_ref` |
|---|---|---|---|---|---|---|
| `scope.core.application` | `scope_key.core.application.singleton@1` | `owner.core.runtime` | `lifetime.core.process@1` | `save_replay.scope.core.application.none@1` | `activation.scope.core.application.process_started@1` | `deactivation.scope.core.application.process_stopping@1` |
| `scope.core.runtime_session` | `scope_key.core.runtime_session.uuidv7@1` | `owner.core.runtime` | `lifetime.core.runtime_session@1` | `save_replay.scope.core.runtime_session@1` | `activation.scope.core.runtime_session.entry_ready@1` | `deactivation.scope.core.runtime_session.stop_or_fault@1` |
| `scope.core.world` | `scope_key.core.world.instance@1` | `owner.core.world` | `lifetime.core.world_instance@1` | `save_replay.scope.core.world@1` | `activation.scope.core.world.branch_ready@1` | `deactivation.scope.core.world.branch_teardown@1` |
| `scope.core.entity` | `scope_key.core.entity.stable_id@1` | `owner.core.runtime_ecs` | `lifetime.core.entity@1` | `save_replay.scope.core.entity.owner_state@1` | `activation.scope.core.entity.created@1` | `deactivation.scope.core.entity.destroyed@1` |
| `scope.core.ui_session` | `scope_key.core.ui_session.uuidv7@1` | `owner.core.ui` | `lifetime.core.ui_session@1` | `save_replay.scope.core.ui_session@1` | `activation.scope.core.ui_session.branch_ready@1` | `deactivation.scope.core.ui_session.branch_teardown@1` |

entryは`scope_type_ref` canonical byte順にsortし、`catalog_hash=SHA-256(catalog_version || canonical entries)`とする。Feature Packは`scope.feature.<feature>.instance`、Genre Packは自身の内部だけで使用する`scope.genre.<genre>.<scope>.instance`を登録できる。Scenario Stage、Encounter、Scoring、Shooter Game Flowは各owner文書のexact 7-Field rowを登録し、Shooter Game Flowのexact scope IDは`scope.genre.shooter.game_flow.instance`である。Core／FeatureからGenre scopeへの依存、duplicate、unknown owner、unavailable／removed owner、instance-key schema不一致、Save／Replay schema hash不一致を`MIRAKAN-RUNTIME-SCOPE-CATALOG_INVALID`、`MIRAKAN-RUNTIME-SCOPE-OWNER_UNAVAILABLE`、`MIRAKAN-RUNTIME-SCOPE-VERSION_HASH_MISMATCH`、`MIRAKAN-RUNTIME-SCOPE-MIGRATION_CONFLICT`でtyped rejectする。`GameSystemSpecV1@1.runtime_instance_scope`は`GameSystemSpecV1@2.runtime_scope_type_ref`へclean migrationし、`play_session`等の旧enum aliasは残さない。ScopeのSource／Save identity、Replay identity、ephemeral runtime generationを相互に置換しない。

### 5.2 InteractionとGame Flow

共通Interaction rejectionは`game_flow_disallowed`を持たない。`operation_eligibility_policy_ref`によるowner判定と、genericな`policy_denied`を使用する。Shooter Game FlowはShooter Pack内だけで`scope.genre.shooter.game_flow.instance`を読むInteraction policyを提供する。

`InteractionSnapshotV1@1.rejection_reason=game_flow_disallowed`は`InteractionSnapshotV1@2.rejection_reason=policy_denied`へversioned clean migrationする。@2で旧enumをaliasとして受理せず、owner／policy／schema hashを一意に解決できないSource／Save／Replayはmigrationを拒否してlast-validを維持する。

Saveは登録済みState ownerと`SaveReplayContractV1`が宣言したfieldだけを保存する。共通Runtimeが常にGame Flow stateを保存してはならない。

### 5.3 Locomotion

Navigation Path FollowingをCharacter Motorへ固定しない。

```text
MotionExecutorPortV1
  executor_capability_ref
  movement_profile_schema_ref
  accepted_intent_schema_refs[]
  resolved_motion_schema_ref
  compatibility_predicate_ref
  failure_diagnostic_refs[]
```

Character MotorはC1で適格化するProviderの一つである。Vehicle、flying、swimming、RTS unit、board token等は別Capability／Providerとして同じPortへ接続できる。Physics CoreはKinematic Character Motorを必須Core機能にせず、Character Locomotion FeatureのEngine-owned reference implementationとして提供する。

Navigationはこの6 Field型の唯一のcanonical ownerである。Path Followingは`executor_capability_ref`とProvider-owned `movement_profile_ref`を要求し、compatibility predicateがintent schema、movement profile、Targetを検証する。path progressは`resolved_motion_schema_ref`の結果だけで判定する。

selected compositionがT40へ直接提出し得るintent schema集合は、selected Providerの`accepted_intent_schema_refs[]`のsubsetでなければならない。Gameplay direct schemaは`mirakan.schema.feature.character_locomotion.GameplayMotionIntentV1@1`、Navigationは`mirakan.schema.navigation.MovementIntentV1@1`、Animation root motionは`mirakan.contract.animation.root_motion_proposal@1`である。Provider-private派生入力をpublic accepted setへ混ぜない。

`game_system.engine.character_locomotion.binding`は`scope.core.entity`のfully registered `GameSystemSpecV1`であり、`mirakan.state.feature.character_locomotion.MotionExecutorSelectionStateV1@1`を所有し、上記3 proposal schemaを受理して`mirakan.contract.feature.character_locomotion.MotionExecutorIntentBatchV1@1`をT40で発行する。MCD共通Envelope、origin、role、responsibility／non-responsibility、read／write型、Capability、phase、dependency edge、Implementation policy、Save／Replay、budget、authoring surface、fallback、fixture、invariant、extension policyを全件登録し、TransformまたはPhysics stateを直接writeしない。

Physics Character Motor Providerの6 Fieldは、Capability `capability.motion_executor.physics_character_motor`、Profile `mirakan.schema.physics.CharacterMotorProfileV1@1`、accepted intent exact 3件、resolved motion `mirakan.schema.physics.PhysicsCharacterResolvedMotionV1@1`、compatibility predicate、failure diagnosticsで固定する。root-motion execution modeがProviderのaccepted policyにない場合はactivationを拒否する。board-token／RTS fixture Providerも6 Field、Target／dimension predicate、diagnosticを完全登録し、Physics dependency 0件を検証する。

`feature.character_locomotion@1`のPack installはPhysicsを要求しない。Physics Character MotorはC1 reference recipe／qualification providerの一つに限る。Animation root motionはselected Motion Executorへの`RootMotionProposalV1` proposalであり、`gameplay_motor`旧modeはprovider-neutral modeへclean migrationする。T40はselected executor、T50はactive Physics providerがある場合だけ実行する。missing／incompatible provider、stale result、provider failureはtyped failureとし、last-valid stateを維持する。

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
