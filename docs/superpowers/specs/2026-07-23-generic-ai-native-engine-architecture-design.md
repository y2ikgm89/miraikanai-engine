# Generic AI-Native Engine Architecture Design

- 作成日: 2026-07-23
- 状態: ユーザー承認済み設計。Architecture正本へ反映中（2026-07-24契約DAG追補）
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
  authoring_operation_refs[]: exact MCD operation ref/version/set root
  runtime_port_refs[]
  configuration_profile_refs[]
  composition_recipe_refs[]
  source_template_refs[]
  validator_refs[]: exact validator ID/version/content hash
  test_scenario_refs[]: exact fixture ID/version/content hash
  example_refs[]
  counterexample_refs[]
  ai_vocabulary_refs[]
  ai_planning_recipe_refs[]
  performance_profile_refs[]
  migration_step_refs[]: exact migration ID/version/content hash
  migration_contribution_refs[]:
    exact registry/contribution ID/version/content hash
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
  qualification_receipt_refs[]:
    exact signed Qualification Receipt ref/version/content hash
  fallback_recipe_ref: CompositionRecipeRef | null

PackRecipeQualificationRecordV1
  qualification_id/version/content_hash
  owner_pack_ref
  recipe_ref/hash
  target_profile_refs[]
  fixture_refs[]:
    exact fixture ID/version/content hash
  input_closure_hash
  result: pass | fail
  signed_receipt: exact MirakanSignedRecordV1
```

選択Recipeのeffective closureはManifest共通Feature、Recipe required Feature、両者の推移Feature DAGの和集合である。canonical Recipe／owner Pack／resolved Pack version＋hashから`closure_hash`を生成し、Preview、Project ChangeSet、Cook、Qualification、Save／Replayへ伝播する。未選択Recipeの条件依存はinstall closureへ追加しない。missing、version／hash conflict、Target不適合、unqualified dependency、fallback cycleはRecipe applyを原子的に拒否し、last-valid Recipe activation、Project revision、registry、closure hash、Artifactを維持する。

`PackManifestV1`はPack payload内のOperation、migration step／contribution、Validator、fixtureを全件列挙する。Manifest Operation集合、Pack ownerのactive MCD Operation集合、Trusted Service allowlist contributionはset equalityである。`validator_refs[]`と`test_scenario_refs[]`はPack内で利用可能なidentity inventoryであり、全Recipe共通の実行gateではない。Recipe apply／qualificationで実行するのは選択済み`CompositionRecipeV1.validator_refs[]`と`qualification_receipt_refs[]`だけである。Production Source／Recipe／Registry／Runtime PackageはFixture bodyを解決せず、別owner-typed Qualification recordのsigned Receiptだけをsubject／owner／Target／result／freshnessで検証する。

Validatorは異種集合を一つにしない。Manifest Validator inventoryはOwner Validator Registry subsetとValidator ID／version／content hashでset equalityにし、各Operationはclosure ValidatorのDiagnostic union、`reachable_error_refs[]`、Operation `errors[]`をDiagnostic ID／code／version／content hashで別途set equalityにする。

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

minimal target-practiceは既存`genre.shooter.top_down_2d | genre.shooter.third_person_3d`のexact 1件とPack-owned immutable `ShooterTargetProviderTemplateV1`を選ぶ。Production bindingはowner-typed `ShooterTargetProviderBindingDocumentV1`、選択正本は`ShooterTargetProviderSelectionDocumentV1`、qualification fixtureはProject Documentではない`ShooterTargetProviderFixtureBindingRecordV1`として分離する。Binding Registryはtrusted Document index内のBinding＋Selection Sourceからreload時に決定的に再materializeするDerived closureで、Registry直接writeやRegistry→Source逆投影を禁止する。Compile／Save／ReplayはSelection Document ref／hash、Binding Document ref／hash、RegistryRef、selected record set hashを束縛する。create／update／selectは三つの完全な`McdOperationContractV1`、named typed input／Result／Prepared semantic Receipt payload／canonical `PublishedDomainReceiptV2` wrapper、exact `MutationAuthorizationBindingV2`、pure pre／post policy、Service／rate policy、17 exact Diagnostic実配列、validator reachable error set equality、request hash／idempotencyを持つ。Shooter ManifestのValidator inventoryとShooter owner Registry subsetはcomposition／perception二件と三Operationのsemantics／postcondition六件のexact八件である。publicationはprivate durable marker後にsigned wrapperを保存／readbackし、その後だけPublic Marker＋after Projectを一つのpublic CASで発行する。N create→N+1 reload／select／compile、unrelated N+2でbinding hash不変、cross-project／self-assert spoof、各crash windowをfixture化する。`fixture_only`はexact Fixture Registry ownerだけを許可しProduction Source／Registry／Save／Packageへ昇格しない。

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

RuntimeTargetSelectorHashPayloadV1
  selector_id
  selector_version
  target_profile_ref_count
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

selectorはexact `DocumentRef<TargetProfileDocument>`だけをStable ID／schema／content hash順で持ち、wildcard、tag、表示名、platform名、active Targetの現在値によるlookupを許可しない。`target_profile_ref_count`は配列長と一致し、duplicate／same-ID different-hash／非canonical orderを拒否する。`selector_hash=SHA-256(ASCII "MIRAKAN_RUNTIME_TARGET_SELECTOR_V1" || uint32_be(length(MCD canonical bytes of RuntimeTargetSelectorHashPayloadV1)) || payload bytes)`であり、payloadはhash Fieldを持たない。ID、version、count、ref全Fieldをbindし、ID-only mutation、count mismatch、旧hash再利用を拒否する。entry／selector／policyのcreate／updateとroot scene migrationの七Operation、Scope migration一OperationはMCD Registryへ完全`McdOperationContractV1`として登録する。八recordは共通Envelope、exact input／output／rate-limit／receipt MCD ref、version／hash付きauthority、R2／R3、side effect、idempotency、transaction、16件のpure `policy` pre／post refs、closed `DiagnosticCodeRefV1` set、timeoutを全件明示する。Diagnostic refはID／code／version／content hashをcanonical Registryへexact解決し、各Operationのvalidator error union、reachable errors、`errors[]`をset equalityにする。`request_hash`はExecutable Contractsのcurrent V2定義だけを参照し、root migration ReceiptはWorldをcontent refだけ、selector／policy／entryをcontent ref＋owner固有semantic hashで記録する。Project ownerは七exact Operation refだけを参照し、metadataを複写しない。

tagged branchは次を強制する。

- `world`: `world_ref`だけをbranch固有必須refにし、`startup_game_system_refs[0..128]`を許可する。
- `ui`: `ui_document_ref`だけをbranch固有必須refにし、`startup_game_system_refs[0..128]`を許可する。Worldは不要。
- `headless`: `startup_game_system_refs[1..128]`を必須にし、World／UI／surfaceを要求しない。

各active Targetについて`default_for_selected_targets=true`のentryだけを対象に`target_selector_ref`の被覆を検査し、default 0件または2件以上を拒否する。default selector集合とactive Target集合はset equalityで一致させ、inactive extra Targetも拒否する。non-default entryのselector overlapは許可し、benchmark、menu、game、server等を同Targetへ複数登録できる。Runtimeはdefaultまたは明示選択を推測せず、実際の選択結果を`selected_runtime_entry_point_ref`、`selected_runtime_entry_point_hash`、`target_selector_hash`、`activation_policy_hash`、`entry_branch_closure_hash`としてCompile Manifestへ保存する。owner-typed Provider Bindingを一件以上選んだ時だけ`selected_provider_binding_set_hash`をcanonical presentにし、post-commit Project revision／document set hash、Registry ref／hash、Binding Document content hash、revision非依存semantic hashをentry closureへ入れる。Binding payloadへProject revisionを戻さない。World／Topology／streaming hashは選択branchが`world`で対応Sourceが存在する場合だけcanonical presentとし、`ui`／`headless`ではcanonical omissionにする。

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

旧`root scene`はhash付き`RootSceneMigrationPlanRefV1`が唯一のpreimageである。Planはlegacy closure、World／selector／activation policy／runtime entryのexact四Document mutation、各active Targetのdefault bindingを持つ。各mutationは`create {allocation_intent, plan_local_ref}`または`update {current_ref, before_hash, preserved_identity}`のclosed branchであり、Stable ID allocation件数とmappingはcreate branch数だけに一致する。Preview、Prepared Candidate、postcondition、Prepared Receipt payloadは同じPlan／allocation mappingを束縛し、private durable marker→canonical signed wrapper→Public Marker＋四Documentの順で一括publishする。実行時の暗黙migration、`Level` alias、`ui`／`headless`への近似変換を禁止する。

### 4.2 Generic World

`WorldDocumentV1`は空間、Scene、global composition、persistent entity、任意のspatial topologyだけを所有する。

- `scene_document_refs`は`0..65,535`とし、procedural-only Worldを許可する。
- `spatial_topology_definition_ref`は`0..1`とする。
- Level、Objective、Completion、Encounter、player／party transferをCore Worldから除去する。
- World activation、Scene activation、Cell streamingはObjectiveやResultを要求しない。

`SpatialTopologyDefinitionV1`は`space_nodes[]`、`transition_edges[]`、`activation_entry_refs[]`を持つ。entryは0件を許可し、到達不能spaceは`intentionally_isolated=true`で明示する。World ownerはMCD `type.world.spatial_transition_destination` version 1を登録し、exact World／Topology／Edge／target Space ref、各version／hash、Edge policyに従うoptional／required Anchorを一recordへ閉じる。transition Requestはそのtyped refを参照しFieldを複製しない。transition payloadはtyped `transfer_subject_refs[]`を使用し、Player／Partyを固定しない。

Streaming interestはclosedなPlayer enumではなく、owner登録済みtyped interest-source contract refの集合を使用する。observer、entity、camera、scripted anchor等を登録可能とし、unknown／removed ownerを拒否する。

`WorldAuthoringPlanV1.affected_world_refs`は既存World編集branchで1～64件、新規World作成branchでは`create_document_kinds`がWorld作成を厳密に一件宣言する時だけ0件を許可する。新規Worldの`WorldAuthoringContextV1`はCommit成功後にexact `world_ref`付きで生成し、未発行IDを推測しない。

`ProceduralWorldDefinitionV1`、`WorldAuthoringPlanV1`、precommit `GeneratedWorldSemanticCandidateV1`は同じ`selected_validation_provider_bindings[]`のexact Binding Document ref/content hash、別の`resolved_binding_closure_hash`、output MCD schema集合を持つ。Production recordはFixture bodyでなくexact signed Qualification Receiptを参照する。fresh 3-runはGateway／Brokerを呼ばず、UUIDを持たない連続local ID semantic graph bytes、canonical order、`semantic_graph_hash`、candidate Artifact semantic hashだけのbyte equalityを要求する。

candidateはexact `{project_id, expected_project_revision, document_set_hash}`、immutable `candidate_hash`、`local_id_count`を持つ。合格後、caller-issued `allocation_request_id`をintent subjectへ含め、Authoring Command Gatewayの完全登録済みinternal `operation.world.allocate_generated_stable_ids@1`をexact一回呼ぶ。named Input／Result／Receipt、R3 `MutationAuthorizationBindingV2`、pure pre／post Policy、closed Diagnostic／Validator closure、rate／timeout、Service allowlist、`trusted_internal` exposureを一つのContract set transactionへ登録する。Manifest／prepared Receipt／Publication Stateは同じProject triple、candidate hash、intent／request identityをbindし、`allocated_uuid_count=local_id_count+1`（mapping＋`delta_id`）を検証する。private durable marker→canonical signed wrapper→Public Marker＋Projectの順だけでatomic publishし、crash retryは同じwrapper／Resultへroll-forwardする。

Source、validation output、Preview、Cook、Commitは同じmappingを共有し、二回目allocation、subsystem別mapping、local ID残存を拒否する。final `GeneratedWorldDeltaV1`はallocation ref／hash、両candidate hash、mapping適用済みrecord、self-excluding content hashを持つ。truth tableはabsent+empty=skip、absent+nonempty=reject、selected+required output不足=reject、selected+valid=execute／accept、selected+stale／invalid／failure=Delta全rejectである。hard closureへ含めるのはWorld／Tile／Blockoutが明示選択したgeneric dependency bindingだけであり、Renderer／Collision／Navigationを名前で常時列挙しない。失敗時はlast-valid Source／Artifact／World generationを維持する。五truth-table fixtureと`fixture.world.procedural-core-only`を必須にする。

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

Stage transition destinationはMCD `type.feature.scenario_stage.transition_destination` version 2のinjective tagged unionとし、`stage | runtime_entry | world_space | ui | headless | session_end`ごとにexact 1 branchだけを許可する。`runtime_entry`はworld／nonspatial、`world_space`はworld／spatial、`ui`と`headless`は各専用branchだけで表す。`StageTransitionContractRefSetV1`はStage owner四refとWorld owner一refのexact 5 refだけを持つ。

cross-owner検証はReceiptを含まない`StageTransitionCrossOwnerCandidateV1`を先にhashし、同じcandidate hashへ三owner Receiptと五gate Receiptを発行し、最後にcandidate＋八Receiptから`final_closure_hash`を作る。Receiptはfinal hashをpayload／subject／signature preimageに含めない。Runtime ownerのregistered `BoundaryDeliveryContractV1`はsealed payloadを次のeligible `T00_BoundaryApply`へexactly once配送し、World／spatial／Locomotion／UI／headlessを仮定しない。このRuntime contractをexact五refへ六件目として混ぜず、owned public inventory、runtime port inventory、cross-owner MCD set、Port closure、Gameplay Features aggregateを五つのlike-for-like gateで個別比較する。

Stage lifecycle Systemは`game_system.extension.feature.scenario_stage`、`owner_layer=feature_pack`、exact Feature owner ref、`system_origin=owner_package`である。Scenario／Stage authoring Operationは完全登録されていないためcurrent setとManifest inventoryを空、Capabilityを`not_activated`とする。旧候補名をaliasとして読まず、将来は一つのexact Operation setと§8.1の認可／Contract Set／publication／honest-activation closureを同じactivation transactionへ登録する。

## 5. Gameplay、Scope、Locomotion

### 5.1 System scope

旧版`GameSystemSpecV1.runtime_instance_scope`のclosed enumを、現行`GameSystemSpecV2.runtime_scope_type_ref: RuntimeScopeTypeRefV1`へ置換する。MCD logical IDは`type.game_system.spec`のままversion 2をcurrentとし、V1はoffline migration inputだけに残す。

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

表はID表示だけで、保存値はscope version／hash、MCD version／Contract set hash、owner revision／content hashを全cellに持つ。active Owner／Dependency Registryは全refを実体recordとして解決し、Registry ref／hashのexact equalityとself-excluding canonical Registry hashを検証する。entryはscope ID byte順にsortし、Catalog hashはdomain separator、catalog ID、schema／catalog version、Contract set hash、Dependency Registry ref canonical bytes／同値hash、entry count、七typed refのcanonical bytesをexact inputにして自己hashを除外する。extension ownerは自身のregistered namespaceへscopeを追加できる。Scenario Stage、Encounter、Scoring、Shooter Game Flowの7-Field rowと旧scope migration record／fixtureは各owner文書だけが所有する。Coreからextension scopeへの依存、cross-owner未宣言依存、duplicate、unknown owner、unavailable／removed owner、各dependency version／hash不一致をtyped rejectする。

旧`runtime_instance_scope`はcurrent validatorが常にrejectし、offline `operation.runtime_scope.migrate_game_system` version 1だけが旧Contract setを読む。logical operation／type／policy IDへschema majorを埋め込まず、inputのsource schema ref=`type.game_system.spec` version 1、destination version 2で指定する。Coreはgeneric `RuntimeScopeMigrationContributionRegistryV1`とCore-owned contribution四件だけを所有し、Pack mapping／adapter／fixtureは各ownerが下向きに登録する。typed Result／Receiptはrequest／idempotency、before／after Project、Source、Contribution、scope／auxiliary refs、Save／Replay mapping、Preview／Validation／Commitをbindし、rejected diagnostics 1～64、Receipt diagnostics 0～64、ephemeral generation非移行を記録する。旧enum aliasをcurrent resolverへ残さない。

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

Kinematic Motion ExecutorはC1で適格化するProviderの一つである。Vehicle、flying、swimming、RTS unit、board token等は別Capability／Providerとして同じPortへ接続できる。Physics Coreはgenre／Feature非依存のgeneric Provider／adapterだけを提供する。

Navigationはこの7 Field型、`MotionExecutorProviderCatalogV1`／record、`MotionExecutorProviderRecordRefV1`、`MotionExecutorIntentBatchV1`、generic `MotionIntentContributionBindingRegistryV1`の唯一のcanonical ownerである。RecordRefはCatalog ID／version／hash／Contract setとprovider ID／version／content hashを一つにbindし、Batch、Selection State、Save、Replayが同じ型を使う。Catalog recordはtagged owner identity、production／fixture usage、7 Field descriptor、implementation System、Target、Qualification Receiptを持つ。Coreはprovider-neutral resolverだけを所有し、owner固有proposal schema／adapter／binding／positive fixtureは各PackまたはProjectが登録する。Path Followingは`executor_capability_ref`とProvider-owned `movement_profile_ref`を要求し、compatibility policyがintent schema、movement profile、Targetを検証する。path progressは`resolved_motion_schema_ref`の結果だけで判定する。

Batch entriesは1～16件で、各entryが`intent_schema_ref: McdContractRefV1`、payload refまたはinline value identityのtagged union、payload hash、producer Game System ref、proposal IDを持つ。accepted set検査はbatch envelopeでなくentries schema集合だけへ適用する。type spoof、payload hash mismatch、duplicate proposal、noncanonical order、root-motionあり／なしをfixture化する。Gameplay schemaは`type.feature.character_locomotion.gameplay_motion_intent`、Navigationは`type.navigation.movement_intent`、Animationは`type.animation.root_motion_proposal` version 1である。Provider-private派生入力をpublic accepted setへ混ぜない。

`game_system.extension.feature.character_locomotion.contribution`は`owner_layer=feature_pack`としてFeature-owned proposal／adapter contributionだけを所有する。Navigation／Core-owned `game_system.engine.navigation.motion_intent_batch_publisher`が全owner contributionをcanonical mergeし、`MotionExecutorIntentBatchV1`をT40に一度だけ発行する。`MotionIntentContributionBindingRecordRefV1`、RegistryRef、`MotionIntentBindingSelectionDocumentV1`をCompile／Save／Replay closureへ保存し、Character、RTS、board-token、Animationがgeneric publisherをforkしない。Physicsなしboard-token／RTSとFeature PackなしPath fixtureを必須にする。

Physics Kinematic Motion Providerの7 FieldはCapability `capability.motion_executor.physics_kinematic`、Profile `type.physics.kinematic_motion_profile`、generic Navigation intent／contribution envelope、transport batch、resolved `type.physics.kinematic_resolved_motion`、compatibility policy、failure diagnosticsで固定する。owner固有proposalはPack-owned adapter recordを介し、Physics descriptorへFeature type IDをhard-codeしない。board-token／RTS fixture Providerも7 Fieldとowner／usage／Target／dimension／diagnosticを完全登録し、Physics dependency 0件を検証する。

Physics AI意味解決もobject名をCore enumへ固定しない。Core-owned `PhysicsIntentRoleRegistryV1`は六roleのowner、五axis allowed set、Capability、fixture、self-excluding record hashを完全materializeし、Record内へ自身のhash付きRoleRefを埋め戻さない。`PhysicsIntentRoleSelectionDocumentV1`をProject Source正本、Registry／Resolutionを派生とする。旧enumは`PhysicsIntentRoleMigrationContributionRegistryV1`と完全なMCD migration Operation／Prepared payload／Receipt／Validator／Manifestを介してexact一件へ移行し、近似Core roleへ変換しない。

`feature.character_locomotion@1`のPack installはPhysicsを要求しない。Physics Kinematic Motion ProviderはC1 reference recipe／qualification providerの一つに限る。Animation root motionもgeneric contribution registryからselected executorへ届き、Providerへ直接提出しない。`gameplay_motor`旧modeはprovider-neutral modeへclean migrationする。T40はselected executor、T50はactive Physics providerがある場合だけ実行する。missing／incompatible provider、stale result、provider failureはtyped failureとし、last-valid stateを維持する。

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

Pack Discoveryの旧八actionは完全なMCD登録を持たないため、current Operation set、Manifest、Service allowlist、Provider／MCP Catalogを空にし、Capabilityを`not_activated`とする。旧`operation.packs.*`名をcurrent identityまたはlegacy aliasとして読まない。future vocabularyは`future.packs.action.*`だけに置き、future work item `activation.pack.ai_operations.v1`が採用するexact set、named input／Result／Receipt、Service／Policy／Validator／Diagnostic、Risk、intent DAG、private-to-public publication、Qualificationを同じContract set transactionで閉じるまで要求をfail closedに拒否する。

[Executable Contracts](../../architecture/02-foundation/executable-contracts.md) §§20～21.1の159候補（当初Discovery／Execution 67件＋回収済みDomain authoring／selection 92件）は18の`PlannedOperationFamilyV1`に属し、各familyのcurrent MCD／Manifest／Service／Policy／Validator／Diagnostic／Receipt／Provider／MCP Tool／alias集合はすべて明示的な空配列、stateは`not_activated`である。current active 14（bootstrap八＋Domain六）とはdisjointで、family固有work itemによるatomic Activation以外の部分登録を許可しない。

`AiTaskContextCapsuleV1`はGame Brief／Spec、選択Catalog entry、Project Slice、依存、制約、Target、Budget、Diagnostic、許可Operation、選択理由、source hash、token estimate、omitted range、continuationを持つread-only／Disposable projectionとする。AIへ全Schema、全Project、全設計書、Security署名内部を一括送信しない。Gatewayがsemantic projectionを返し、Authorization、Provenance、Receipt検証はGateway内で強制する。

Model familyまたはHost brandをEngine機能分岐にしない。同じMCD／Operationへ適合するProvider／ModelをProfileとConformance Receiptで有効化する。Strict Tool Callingへ適合しないModelはread-onlyまたはproposal-onlyに制限し、自然言語をCommit Operationへ推測変換しない。

### 8.1 Contract set、request、Commitの非循環DAG

`ContractSetSnapshotV2`だけをcurrent正本とし、closed local-member unionはMCD、Diagnostic、Trusted Service、Validator、Operation Validator Closureの全normative recordを持つ。Snapshot内部はkind別`ContractSetLocalRefV1`とself-excluding local hashだけを使い、Operation／Service allowlist／Validator closure／Diagnosticの相互edgeに外部set rootやrecord hashを入れない。set root preimageはmember kind、kind固有local identity、local hashをkind→ID→versionのcanonical byte順でstrict sortする。生成順は`LocalRef解決 → self-excluding local record hash → closed member set root → external McdContractRef／DiagnosticCodeRef／TrustedServiceRef／ValidatorRecordRef／closure ref materialization`である。Operation表はroot確定後のexternal projectionで、compiler inputは全intra-set refをkind別LocalRefへ機械投影し、Field／cardinality／logical identityのset equalityをfixtureで検証する。

Operation認可は`MIRAKAN_OPERATION_INTENT_V2 -> MutationAuthorizationBindingV2 -> MIRAKAN_OPERATION_REQUEST_V2`の一方向DAGだけを使う。intent payloadは選択named input type、Operation、Risk、全semantic input Fieldを持ち、`operation_intent_hash`、final `request_hash`、authorization binding全体を除外してself-excluding hashを作る。Authorization／Approval／Predelegationは同じintent hashをsubjectにし、final request hashを参照しない。binding確定後、選択inputの`request_hash`だけを除く全実在Fieldからfinal hashを作る。Project-boundはexact Project ref、projectlessはworkspace／catalog／resource等のOwner登録refを使い、sentinel／null Projectを捏造しない。V2は未Activationのため同名で修正し、循環shapeのcompat readerを作らない。retained V1だけをoffline migration record／candidateとして扱い、状態変更Operationは完全登録まで`capability_unavailable`とする。

mutationは`Preview → Validation → PreparedCandidate／PreparedCommitEnvelope → staged postcondition → private durable commit marker → canonical PublishedDomainReceiptV2／MirakanSignedRecordV1 wrapper → PublicPublicationMarkerV1＋public state／Result`の一方向DAGである。private Markerまでは外部current stateを変更しない。signed wrapperを保存／readbackした後だけPublic Markerとafter stateを一つのpublic CASで発行し、unsigned payload、private Marker、receipt-store単独存在をauthorityにしない。postconditionがMarker／公開Receiptを読む循環、Markerがsigned Receipt／Public Markerをhashする循環を禁止する。

`PreparedCandidateV1`はID、schema、Staging root、before／proposed-after state、prepared Artifact集合からself-excluding content hashを作り、外部`PreparedCandidateRefV1`を完成後にmaterializeする。private Marker後／wrapper保存前のcrashは固定materialization key、issued-at、revocation snapshot、key context、deterministic signing profileから同じwrapper bytesへroll-forwardし、wrapper後／Public Marker前は同じexpected predecessorへpublic CASをroll-forwardする。既存wrapperはbyte equalityで再利用し、alternate signature、二重publication、overwrite、public後rollbackを禁止する。MarkerなしPrepared artifactは非公開廃棄する。

R2 mutation inputは`MutationAuthorizationBindingV2`の`approval | predelegated`を厳密に一つ、R3以上はapprovalだけを持つ。欠落／expired／scope mismatchは全Domainで`MIRAKAN-APPROVAL-REQUIRED`とし、Operation error、Validator reachable error、Manifest inventoryをset equalityにする。

Input roleはclosed MCD kind `type`の`SemanticActionRoleRefV1`で表し、未定義kindやbare文字列を受理しない。owner-typed `SemanticActionCommandBindingContributionV1`をcanonical mergeしてDerived `SemanticActionCommandBindingRegistryV1`を再materializeし、Production recordはowner Qualification Receiptだけを参照する。Project Sourceの`SemanticActionBindingSelectionDocumentV1`を変更するOperationは本Taskで完全登録されていないためcurrent setを空、Capabilityを`not_activated`とする。Compile／Save／Replayはcurrent Sourceが存在する場合にSelection Document、Action Map、RegistryRef、全RecordRef、role type、Command Systemをexact closureとして保存し、reload後のSource→Registry→Record equalityを再検査する。

## 9. Product ScopeとQualification

現時点で未実装のPlatform／大規模機能を、汎用Schemaが存在することだけで対応済みと表示しない。offline single-player、Windows-first、60 Hz baseline等の現在Scopeは許容するが、恒久Core制約にしない。

性能正本は`ProjectScaleEnvelopeV2`で、`WorkloadDomainTypeRegistryV1`とowner-typed `WorkloadDomainIntentV1`を束ねる。World／spatialは一件でも`required`なら必須、全domainが`forbidden`なら禁止、`optional`だけならnull／exact World intentの双方を許可する。UI-only、strict headless、tooling、resource-onlyは偽World／Gameplay floorなしでvalidである。semantic modeは`authoritative_equivalence | presentation_fidelity | functional_contract | resource_slo | none`のtagged ruleで、`none`は登録済みtool／resource domainだけに許可する。旧V1は完全なmigration Operation／Prepared payload／Receipt／fixtureでV2へ移し、current Source／projectionへ残さない。Qualificationは全owner dimensionを同時発生させ、未登録dimension、schema／unit／hash不一致、必要なequivalence／fidelity／functional／SLO欠落を開始前に拒否する。

最低Reference／holdout集合を次へ固定する。

- Shooter top-down 2D: 最初のGenre vertical slice
- Shooter third-person 3D: 同じFeature契約の3D検証
- Puzzle／Dialogue 2D: Combatなし
- Platformer 2D: Character Locomotionの別Genre検証
- Turn-based zero-character: Character、Physics、Pack-owned completionを要求しない
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
4. UI-only ProjectがWorld／spatial contentなし、headless Projectがsurfaceなしでvalidである。
5. finite、endless、turn-based、continuous simulation、tool-like、zero-characterをGame Brief／Specで表現できる。
6. Path FollowingがCharacter Motor以外のconformance stubへ接続できる。
7. C1は60/1だけをqualifiedに保ちながら、Core schema変更なしに別Cadence Profileを追加できる。
8. Pack AI Operationが未Activationの間はPack追加／更新／削除要求をfail closedに拒否し、将来activation時は依存、影響、migration、fallbackをexact IDで説明できる。
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
