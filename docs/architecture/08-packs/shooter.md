# Miraikanai Engine Shooter Genre Pack

- 文書ID: mirakan.arch.pack-shooter
- 状態: review
- 正本範囲: Shooter固有composition recipe、Genre identity、Profile、Game Flow、Action role、Shooter fixture
- 非正本範囲: Feature CapabilityのPublic Contract／Schema／State owner、Pack lifecycle、Input／Collision／Physics／Navigation／Camera／UI／Audio／LOD、World／Stage、Runtime scheduling、Product roadmapは各Ownerを参照
- 依存: [Product Plan](../00-product/product-plan.md)、[Pack Contract](pack-contract.md)、[Gameplay Feature Packs](gameplay-features.md)、[Scenario／Stage](scenario-stage.md)、[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Collision](../05-simulation/collision.md)、[Physics](../05-simulation/physics.md)、[Navigation](../05-simulation/navigation.md)、[Camera](../06-rendering/camera.md)、[LOD](../06-rendering/lod.md)、[Input](../07-platform/input.md)、[Audio](../07-platform/audio.md)、[Game UI](../07-platform/ui-text-localization-accessibility.md)
- 外部根拠検証日: 2026-07-23

## 1. 結論

Shooterは`pack_kind=genre`のGenre Packであり、Generic Engine Coreまたは単一の共通Gameplay基盤ではない。Shooterは再利用可能なFeature Packをcomposition recipeで組み合わせ、Shooter固有のProfile、Game Flow、Action role、reference scenarioだけを所有する。

正規Genre identityは次の二つである。

- `genre.shooter.top_down_2d`
- `genre.shooter.third_person_3d`

2D／3Dは同じFeature Capabilityを使い、Damage、Vital、Faction、Weapon、Shot、Projectile、Encounter、Scoring、Save／Replayの意味をforkしない。Shooter Packを未installまたは削除した状態でCore、Editor、AI、Project C++、Project Shader、Test、Build、Packageが成功しなければならない。

## 2. Feature Capability mapping

Shooter Genre Packは次のFeature Packを参照する。各Public Contract、Schema、State owner、Validator、Runtime Port、failureはFeature Packが唯一のOwnerであり、本書は再定義しない。

| Feature Pack | Provided Feature Capability | Shooterでの利用 |
|---|---|---|
| `feature.combat` | `capability.gameplay.combat` | Damage、Vital、Faction |
| `feature.ranged_combat` | `capability.gameplay.ranged_combat` | Weapon、Shot、Projectile |
| `feature.encounter_spawn` | `capability.gameplay.encounter_spawn` | Encounter、Spawn |
| `feature.scoring` | `capability.gameplay.scoring` | score、combo、result projection |
| `feature.pickup_grant` | `capability.gameplay.pickup_grant` | Pickup、typed Grant |
| `feature.interaction` | `capability.gameplay.interaction` | interact eligibilityとgeneric `policy_denied` |
| `feature.character_locomotion` | `capability.gameplay.character_locomotion` | controlled actorの移動intentとMotion Executor binding |
| `feature.path_following` | `capability.gameplay.path_following` | enemy／agent route。Character Motorへ固定しない |
| `feature.scenario_stage` | `capability.gameplay.scenario_stage` | 有限Stage、Objective、Completion、transition |

Pack-levelのRanged CombatとそのFeature DAG以外のFeature Capabilityは、必要なcomposition recipeだけが選ぶ。たとえばendless Shooter recipeはScenario／Stageを省略でき、Shooter Packを使用しないtool-like weapon editorはCombat、Encounter、Worldを要求しない。現在の二つのreference fixtureは開始からResultまでの有限Gameを検証するため、表のFeature Pack closureを使用する。

Shooterのenemy targetingは`capability.gameplay.perception`をrequired Capabilityとして参照し、Public ContractとState ownerは[Gameplay Programming Model §2.4](../03-authoring/gameplay-programming-model.md#24-c1-perceptioninteraction)が所有する。Shooterは`PerceptionProfileV1`、`PerceptionSnapshotV1`、Perception memoryを再定義またはwriteしない。

Genre Packは別Genre Packへ依存しない。複数Genreとの合成はGame Projectが行う。

## 3. Genre composition

```text
genre.shooter.top_down_2d
  -> profile.shooter.top_down_2d
  -> Feature Capability closure
  -> Generic Engine Core

genre.shooter.third_person_3d
  -> profile.shooter.third_person_3d
  -> Feature Capability closure
  -> Generic Engine Core
```

次は[Pack Contract](pack-contract.md)が所有するcanonical `PackManifestV1` schemaへShooter値を代入したpartial value projection／具体instanceであり、schemaの再定義ではない。表外FieldもPack Contractどおりexact identityで解決し、Feature契約をpayloadとして内包しない。

```text
PackManifestV1
  pack_id: genre.shooter
  pack_version: 1.0.0
  pack_kind: genre
  required_feature_pack_refs: [feature.ranged_combat@1]
  required_capability_refs: []
  composition_recipe_refs:
    [recipe.shooter.top_down_2d.finite_stage,
     recipe.shooter.third_person_3d.finite_stage,
     recipe.shooter.top_down_2d.endless,
     recipe.shooter.target_practice.minimal]
  configuration_profile_refs:
    [profile.shooter.top_down_2d, profile.shooter.third_person_3d,
     profile.shooter.target_practice]
  public_contract_refs:
    [ShooterPerceptionBindingV1, ShooterGameFlowV1,
     ShooterGameFlowInteractionEligibilityPolicyV1,
     ShooterActionRoleSetV1, ShooterMinimalActionRoleSetV1,
     ShooterTargetProviderTemplateV1, ShooterTargetProviderBindingDocumentV1,
     ShooterTargetProviderSelectionDocumentV1,
     ShooterTargetProviderBindingRegistryV1]
  authoring_operation_refs:
    [operation.shooter.target_provider_binding.create@1,
     operation.shooter.target_provider_binding.update@1,
     operation.shooter.target_provider_binding.select@1]
  validator_refs:
    [validator.genre.shooter.composition@1,
      validator.genre.shooter.perception_binding@1,
      validator.genre.shooter.target_provider_binding.create_semantics@1,
      validator.genre.shooter.target_provider_binding.create_postcondition@1,
      validator.genre.shooter.target_provider_binding.update_semantics@1,
      validator.genre.shooter.target_provider_binding.update_postcondition@1,
      validator.genre.shooter.target_provider_binding.select_semantics@1,
      validator.genre.shooter.target_provider_binding.select_postcondition@1]
  migration_contribution_refs:
    []
  migration_step_refs:
    [migration.genre.shooter.feature_identity@1]
  test_scenario_refs:
    [fixture.product.shooter-2d@1, fixture.product.shooter-arena-3d@1,
     fixture.genre.shooter.endless-top-down-2d@1,
     fixture.genre.shooter.target-practice-minimal@1,
     fixture.genre.shooter.target-practice-minimal-no-perception@1,
     fixture.genre.shooter.target-practice-minimal-project-provider@1]
```

全`@1`表示はManifest保存時に対応するversion／Contract set rootまたはcontent hashを持つexact refへ展開し、裸IDを保存しない。三Operation inventory、Pack ownerのactive MCD Operation集合、Authoring Command Gateway Service allowlist contributionはset equalityである。Manifest `validator_refs[]`はShooter Validator Registryの`owner_ref=owner.genre.shooter` subsetとexact 8-ref set equalityであり、common Operation ValidatorをPack inventoryへ複写しない。Profileは独立PackではなくShooter Packのversion／hashに含まれる。ManifestのValidator／Test一覧はPack inventoryであり、選択Recipeは自身の`validator_refs[]`とroot外`PackRecipeActivationBindingV1`が指すsigned Receiptだけを実行gateにする。Fixture bodyは別Qualification subjectだけが解決し、Production Recipe／Runtime Packageは解決しない。Pack-level requiredはRanged Combatとその推移Feature DAGだけである。Encounter、Scoring、Pickup、Interaction、Character Locomotion、Path Following、Scenario／Stage、Perceptionは、それらを使用するRecipeだけが持つ条件依存である。Runtime Scope legacy migrationはcurrent Pack inventoryへ含めず、`migration_contribution_refs=[]`、migration固有Qualification Binding／Activation Catalog集合もexact `[]`である。次の四`CompositionRecipeV1`は[Pack Contract §3.1](pack-contract.md#31-compositionrecipev1)のcanonical schemaを満たすShooter具体instanceであり、後続するPack Recipe Qualification／Activation三recordは同じ節のcanonical schemaに対するShooter owner制約付きvalue projectionである。いずれもschemaの再定義ではない。

```text
CompositionRecipeV1
  recipe_id: recipe.shooter.top_down_2d.finite_stage
  recipe_version: 1.0.0
  recipe_hash: Sha256(self_excluding_canonical_record)
  owner_pack_local_identity: {pack_id=genre.shooter, pack_version=1.0.0}
  required_capability_refs: [capability.gameplay.perception]
  required_feature_pack_refs:
    [feature.encounter_spawn@1, feature.scoring@1, feature.pickup_grant@1,
     feature.interaction@1, feature.character_locomotion@1,
     feature.path_following@1, feature.scenario_stage@1]
  configuration_profile_refs: [profile.shooter.top_down_2d]
  game_spec_template_refs: [template.shooter.top_down_2d.finite_stage]
  action_role_set_refs: [ShooterActionRoleSetV1]
  source_template_refs: [template.source.shooter.top_down_2d]
  validator_refs: [validator.genre.shooter.composition, validator.genre.shooter.perception_binding]
  fallback_recipe_ref: null

CompositionRecipeV1
  recipe_id: recipe.shooter.third_person_3d.finite_stage
  recipe_version: 1.0.0
  recipe_hash: Sha256(self_excluding_canonical_record)
  owner_pack_local_identity: {pack_id=genre.shooter, pack_version=1.0.0}
  required_capability_refs: [capability.gameplay.perception]
  required_feature_pack_refs:
    [feature.encounter_spawn@1, feature.scoring@1, feature.pickup_grant@1,
     feature.interaction@1, feature.character_locomotion@1,
     feature.path_following@1, feature.scenario_stage@1]
  configuration_profile_refs: [profile.shooter.third_person_3d]
  game_spec_template_refs: [template.shooter.third_person_3d.finite_stage]
  action_role_set_refs: [ShooterActionRoleSetV1]
  source_template_refs: [template.source.shooter.third_person_3d]
  validator_refs: [validator.genre.shooter.composition, validator.genre.shooter.perception_binding]
  fallback_recipe_ref: null

CompositionRecipeV1
  recipe_id: recipe.shooter.top_down_2d.endless
  recipe_version: 1.0.0
  recipe_hash: Sha256(self_excluding_canonical_record)
  owner_pack_local_identity: {pack_id=genre.shooter, pack_version=1.0.0}
  required_capability_refs: [capability.gameplay.perception]
  required_feature_pack_refs:
    [feature.encounter_spawn@1, feature.scoring@1, feature.pickup_grant@1,
     feature.interaction@1, feature.character_locomotion@1,
     feature.path_following@1]
  configuration_profile_refs: [profile.shooter.top_down_2d]
  game_spec_template_refs: [template.shooter.top_down_2d.endless]
  action_role_set_refs: [ShooterActionRoleSetV1]
  source_template_refs: [template.source.shooter.top_down_2d]
  validator_refs: [validator.genre.shooter.composition, validator.genre.shooter.perception_binding]
  fallback_recipe_ref: null

CompositionRecipeV1
  recipe_id: recipe.shooter.target_practice.minimal
  recipe_version: 1.0.0
  recipe_hash: Sha256(self_excluding_canonical_record)
  owner_pack_local_identity: {pack_id=genre.shooter, pack_version=1.0.0}
  required_capability_refs: []
  required_feature_pack_refs: []
  configuration_profile_refs: [profile.shooter.target_practice]
  game_spec_template_refs: [template.shooter.target_practice.minimal]
  action_role_set_refs: [ShooterMinimalActionRoleSetV1]
  source_template_refs: [template.source.shooter.target_practice]
  validator_refs: [validator.genre.shooter.composition]
  fallback_recipe_ref: null

PackRecipeQualificationSubjectV1
  qualification_id
  qualification_version: positive uint32
  owner_pack_ref: exact genre.shooter PackContractRefV1
  recipe_ref/hash: exact CompositionRecipeV1
  target_profile_refs[1..64]
  fixture_refs[1..256]:
    exact {fixture_id, fixture_version, fixture_content_hash}
  input_closure_hash
  result: pass | fail
  qualification_subject_hash: SHA-256

PackRecipeQualificationReceiptV1
  subject: exact completed PackRecipeQualificationSubjectV1 above
  signed_record:
    exact MirakanSignedRecordV1(purpose=pack_recipe_qualification)

PackRecipeActivationBindingV1
  activation_binding_id
  activation_binding_version: positive uint32
  recipe_ref/hash: exact receipt-free subject.recipe_ref/hash
  qualification_receipt_ref: PackRecipeQualificationReceiptRefV1
  activation_binding_hash: SHA-256
```

四Recipeには同logical suffixのQualification subject／Receipt／Activation Bindingをroot外でexact一件ずつ作る。生成順とhash規則は[Pack Contract §3.1](pack-contract.md#31-compositionrecipev1)の共通DAGだけを使い、Shooter固有purpose、Receipt Field、inline wrapperをRecipe／Profile／Pack hashへ追加しない。`Sha256(self_excluding_canonical_record)`も同節の規則により各Receipt-free Recipe recordから計算して保存・検証する。有限Recipeのeffective closureだけが`feature.scenario_stage@1`を含み、endless／minimal Recipeのclosureには含めない。既存2D／TPSとendless RecipeはTask 1時点のFeature closureとPerception bindingをRecipe側で維持する。minimal Recipeのeffective closureはPack共通の`feature.ranged_combat@1 -> feature.combat@1`だけで、AI、Stage、Score、Locomotion、Path、Encounter、Pickup、Interactionを含まない。closure hash不一致または条件Feature unavailableでは該当Recipe applyだけを`MIRAKAN-PACK-RECIPE-DEPENDENCY_UNRESOLVED`で拒否し、他Recipeとlast-valid Projectを変更しない。

composition recipeはFeature Capability、Profile、GameSpec template、Action role、reference scenarioをexact IDで結ぶ。Feature schema、World、Stage、Runtime Scope、Input Action identityをrecipe内で再定義しない。

## 4. Shooter Profile

### 4.1 `profile.shooter.top_down_2d`

- `genre_ref=genre.shooter.top_down_2d`
- orthographic／pixel-safe Camera binding
- `move`、`aim`、primary fire、任意のsecondary fire／reload／pause Action role
- top-down向けCamera、Audio、LOD configuration
- 2D collision／animation／VFX／UIへのbinding
- Qualification ReceiptはProfile外の`PackRecipeActivationBindingV1`から検証する

### 4.2 `profile.shooter.third_person_3d`

- `genre_ref=genre.shooter.third_person_3d`
- third-person Camera、camera collision、reticle binding
- `move`、`look`、primary fire、aim mode、reload、weapon switch、pause Action role
- third-person向けCamera、Audio、LOD configuration
- 3D collision／animation／VFX／UIへのbinding
- Qualification ReceiptはProfile外の`PackRecipeActivationBindingV1`から検証する

### 4.3 `profile.shooter.target_practice`

```text
ShooterTargetPracticeProfileV1
  profile_id: profile.shooter.target_practice
  profile_version: 1.0.0
  profile_hash
  genre_ref: genre.shooter.top_down_2d | genre.shooter.third_person_3d
  target_provider_template_ref
  target_provider_template_hash
  required_action_role_refs: [shooter.fire_primary]
```

一recordの`genre_ref`は既存二Genre identityのexact 1件であり、組込みstationary templateは`genre.shooter.top_down_2d`を選ぶ。stationary／vehicle-mounted／tool-like firing stationのいずれでもmove／look／AI enemyを要求しない。ProfileとtemplateはPack-owned immutable Sourceであり、Project identity、Project target data、implementation System、Save instanceを含めない。

### 4. Provider Binding

Pack-owned templateを次へ固定する。MCD参照はすべて`McdContractRefV1 {id, version, contract_set_hash}`であり、IDへPascalCaseまたは`@1`を埋め込まない。

```text
ShooterTargetProviderTemplateV1
  template_id
  template_version: uint32
  template_hash: SHA-256(self excluding template_hash)
  target_schema_ref: McdContractRefV1(kind=type)
  collision_query_port_ref:
    McdContractRefV1(id=type.feature.ranged_combat.collision_query_port, version=1, contract_set_hash)
  hit_evidence_schema_ref:
    McdContractRefV1(id=type.feature.ranged_combat.shot_hit_event, version=1, contract_set_hash)
  supported_dimensions[1..2]: 2d | 3d
  compatibility_policy_ref: McdContractRefV1(kind=policy)
  allowed_binding_usage[1..2]: project_owned | fixture_only
  required_implementation_role_ref
  save_replay_requirement_ref
```

組込みstationary templateは次のexact値を持つ。これは直前のcanonical `ShooterTargetProviderTemplateV1` schemaへ値を代入したbuilt-in instanceであり、schemaの再定義ではない。

```text
ShooterTargetProviderTemplateV1
  template_id: template.genre.shooter.target_provider.stationary
  template_version: 1
  template_hash: Sha256(self_excluding_template_hash)
  target_schema_ref:
    {id=type.genre.shooter.stationary_target, version=1, contract_set_hash}
  collision_query_port_ref:
    {id=type.feature.ranged_combat.collision_query_port, version=1, contract_set_hash}
  hit_evidence_schema_ref:
    {id=type.feature.ranged_combat.shot_hit_event, version=1, contract_set_hash}
  supported_dimensions: [2d]
  compatibility_policy_ref:
    {id=policy.genre.shooter.stationary_target_2d, version=1, contract_set_hash}
  allowed_binding_usage: [project_owned, fixture_only]
  required_implementation_role_ref:
    {role_id=role.genre.shooter.target_provider, role_version=1, role_hash}
  save_replay_requirement_ref:
    {id=requirement.genre.shooter.target_provider_save_replay, version=1, contract_set_hash}
```

Project固有選択はtemplateを変更せず、owner-typed Project Documentへ分離する。

```text
ShooterTargetProviderBindingDocumentV1
  common Project Document header
  payload: ShooterTargetProviderBindingV1(
    owner_kind=project_owned, usage=project_owned)

ShooterTargetProviderSelectionDocumentV1
  common Project Document header
  selection_id: same StableId as Document header
  project_id: stable parent Project ID
  selected_bindings[0..256]:
    ShooterTargetProviderSelectedBindingSourceRecordV1
  selection_hash: SHA-256

ShooterTargetProviderFixtureBindingRecordV1
  common Fixture Registry record header
  payload: ShooterTargetProviderBindingV1(
    owner_kind=fixture_only, usage=fixture_only)

ShooterTargetProviderBindingV1
  binding_id
  binding_version: uint32
  binding_hash: SHA-256(self excluding binding_hash)
  template_ref
  template_hash
  owner_identity: ShooterTargetProviderOwnerIdentityV1
  usage: project_owned | fixture_only
  implementation_system_ref: UsageTaggedImplementationSystemRefV1
  target_data_ref
  target_data_hash
  selected_target_profile_refs[1..64]
  save_replay_contract_ref
  save_replay_contract_hash

ShooterTargetProviderOwnerIdentityV1
  owner_kind: project_owned | fixture_only
  project_id: StableId | null
  fixture_owner_ref:
    exact {fixture_id, fixture_version, fixture_content_hash} | null

ShooterTargetProviderBindingDocumentRefV1
  stable_id: StableId
  document_revision: uint64
  content_hash: SHA-256
  binding_version: uint32
  binding_hash: SHA-256

ShooterTargetProviderSelectedBindingRecordV1
  runtime_entry_ref
  runtime_entry_content_hash
  runtime_entry_semantic_hash
  recipe_ref/hash
  profile_ref/hash
  target_profile_ref/hash
  binding_document_ref: ShooterTargetProviderBindingDocumentRefV1
  selected_record_hash: SHA-256

ShooterTargetProviderSelectedBindingSourceRecordV1
  runtime_entry_ref
  runtime_entry_content_hash
  runtime_entry_semantic_hash
  recipe_ref/hash
  profile_ref/hash
  target_profile_ref/hash
  binding_document_ref: ShooterTargetProviderBindingDocumentRefV1
  source_record_hash: SHA-256

ShooterTargetProviderSelectionDocumentRefV1
  stable_id
  document_revision
  content_hash
  selection_hash

ShooterTargetProviderBindingRegistryRefV1
  registry_id
  registry_version: uint64
  registry_hash: SHA-256
  project_ref: exact {project_id, project_revision, document_set_hash}

ShooterTargetProviderBindingRegistryV1
  registry_id: shooter.target_provider_binding.registry.active
  registry_version: uint64
  registry_hash: SHA-256
  project_ref: exact {project_id, project_revision, document_set_hash}
  binding_document_refs[0..1024]:
    ShooterTargetProviderBindingDocumentRefV1
  selection_document_ref:
    ShooterTargetProviderSelectionDocumentRefV1
  selected_bindings[0..256]:
    ShooterTargetProviderSelectedBindingRecordV1
```

Binding Documentは共通identity ruleに従い、`DocumentRef.stable_id == header.document_id == payload.binding_id`を必須とし、`owner_kind=project_owned`かつ`usage=project_owned`以外を拒否する。Project payloadはstable `project_id`だけをnon-nullにし、Project revision／document set hashを保存しない。Selection Documentも`DocumentRef.stable_id == header.document_id == selection_id`とparent Project containmentを必須にし、Runtime Entry／Recipe／Profile／Target／Bindingの選択を所有する唯一のProject Source正本である。所有証明はtrusted Document indexのProject containment、両Document headerのparent Project、current Registry Project ID、compile対象Project IDのexact equalityで行う。payload自身のProject IDだけを証拠にせず、cross-project containment、self-assert spoofをrejectする。

fixture bindingはBinding Documentではなく`ShooterTargetProviderFixtureBindingRecordV1`としてactive Fixture Registryだけに存在し、`FixtureRecordRef.stable_id == header.record_id == payload.binding_id`、exact fixture owner、`owner_kind=fixture_only`、`usage=fixture_only`、同ownerの`FixtureImplementationSystemRefV1`＋Fixture System Activation Binding branchを必須にする。Project branchは`GameSystemContractRefV1`＋contract hash＋Game System Activation Bindingだけ、fixture branchはFixture System ref＋Fixture System Activation Bindingだけを持ち、branch Fieldの両立／欠落を拒否する。Project Document／Production Registry／Production Save／Runtime Packageへfixture branchを登録しない。

`binding_hash`は`SHA-256(ASCII "MIRAKAN_SHOOTER_TARGET_PROVIDER_BINDING_V1" || binding_id || binding_version || template ref/hash || owner_kind || stable project_idまたはfixture owner exact ref || usage || canonical UsageTaggedImplementationSystemRefV1 branch || target data ref/hash || canonical selected Target Profile refs || Save Replay contract ref/hash)`である。Project revision、document set hash、Binding Document header／content hash、Registry revision／hash、Compile Manifest hash、Receipt hashを入力に含めないため、Commit後revisionをpayloadへ戻すfixed pointを作らない。

Production applyはProject-owned Binding Documentのexact ref／content hash／binding semantic hash、template ref／hash、`usage=project_owned`のproduction System ref／contract hash、Save Replay contract ref／hashを必須にする。fixture binding、表示名、似たCollider、同じtemplateの別Project bindingへfallbackしない。

`binding_document_refs[]`は`stable_id` UUID byte、document revision、content hash、binding version、binding hash順、Selection SourceとDerived Registryの`selected_bindings[]`はruntime entry Stable ID、entry semantic hash、recipe ID／version／hash、profile ID／version／hash、Target Profile ID／version／hash、binding Stable ID順へstrict sortする。duplicate ref、同一Stable ID＋revisionの異なるhash、同じselection keyの複数record、非canonical orderをSource validation／Registry materialization errorにする。各Source recordはcurrent Binding Document exact一件へ解決し、Derived recordはSource recordとBinding Documentからのみmaterializeする。`source_record_hash`はASCII `MIRAKAN_SHOOTER_TARGET_PROVIDER_SELECTED_BINDING_SOURCE_RECORD_V1`、`selection_hash`はASCII `MIRAKAN_SHOOTER_TARGET_PROVIDER_SELECTION_DOCUMENT_V1`、`selected_record_hash`はASCII `MIRAKAN_SHOOTER_TARGET_PROVIDER_SELECTED_BINDING_RECORD_V1`と各self hashを除く全Fieldのlength-framed canonical bytesから計算する。

RegistryはProject reload時にtrusted Document indexのBinding Document集合とexact Selection Documentからのみ再materializeするDerived closureである。Registry hashはASCII `MIRAKAN_SHOOTER_TARGET_PROVIDER_BINDING_REGISTRY_V1`、registry ID／version、exact Project ref、Selection Document ref、二配列のcountとcanonical record bytesを順に`uint32_be` length framingし、`registry_hash`自身を除外してSHA-256する。`ShooterTargetProviderBindingRegistryRefV1`は四Fieldすべてをexact解決し、IDだけ、latest version、別Projectの同hashへfallbackしない。Registry直接write、RegistryをSourceへ逆投影、select時のDerived recordだけのmutationを禁止する。

Compile Manifestは次のpayloadをmaterializeする。

```text
ShooterSelectedProviderBindingSetHashPayloadV1
  registry_ref: ShooterTargetProviderBindingRegistryRefV1
  selection_document_ref: ShooterTargetProviderSelectionDocumentRefV1
  project_ref: exact {project_id, project_revision, document_set_hash}
  runtime_entry_ref
  runtime_entry_content_hash
  runtime_entry_semantic_hash
  selected_record_count
  selected_records[]: exact ShooterTargetProviderSelectedBindingRecordV1
```

`selected_record_count`は配列長と一致し、recordsはSelection SourceとRegistryの同じcanonical順、全recordは同一entry branchかつSource／Registry memberでなければならない。`selected_provider_binding_set_hash`はASCII `MIRAKAN_SHOOTER_SELECTED_PROVIDER_BINDING_SET_V1`、payloadのself-hashを持たない全Fieldをlength framingしてSHA-256する。Selection Document ref／hash、Registry ref／hash、post-commit Project revision／document set hash、entry Document content hash／semantic hash、Binding Document五Field ref、selected record hashの一Fieldでも変われば別hashとなり、当該entry branch closureへ入れる。Production SaveはSelection Document ref四Field、binding Document ref五Field、template hash、production System ref／contract hash、target data identity、Save Replay contract hashを保存し、Load／Replayは同じclosureまたは明示migrationを要求する。fixture System refを保存しない。

fixture-only binding recordはexact `fixture.genre.shooter.target-practice-minimal` owner、`usage=fixture_only`、`fixture_system.genre.shooter.stationary_target_provider@1`のexact `FixtureImplementationSystemRefV1`＋`activation.fixture_system.genre.shooter.stationary_target_provider@1`、fixture target data、fixture Target Profileを持つ。implementation base recordを次の全Fieldで固定する。これは[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)が所有するcanonical `FixtureImplementationSystemRecordV1` schemaのShooter fixture具体instanceであり、schemaの再定義ではない。空配列もhashへcount 0で含める。

```text
FixtureImplementationSystemRecordV1
  fixture_system_id:
    fixture_system.genre.shooter.stationary_target_provider
  fixture_system_version: 1
  fixture_owner_ref:
    {fixture.genre.shooter.target-practice-minimal,1,fixture_content_hash}
  implementation_artifact_ref:
    {artifact_kind=fixture_system.genre.shooter.stationary_target_provider,
     schema_version=1,sha256}
  read_type_refs:
    [type.genre.shooter.stationary_target@1]
  accepted_command_type_refs: []
  accepted_port_message_type_refs:
    [type.feature.ranged_combat.collision_query_port@1]
  emitted_event_type_refs:
    [type.feature.ranged_combat.shot_hit_event@1]
  emitted_port_message_type_refs: []
  supported_target_profile_refs: [target.headless.host@1]
  fixture_system_content_hash:
    SHA-256(MIRAKAN_FIXTURE_IMPLEMENTATION_SYSTEM_RECORD_V1,
      self-excluding canonical fields)
```

Registry／System ref確定後、`qualification.fixture_system.genre.shooter.stationary_target_provider@1` subjectは上記System ref、同じfixture owner、`target.headless.host@1`、exact `fixture.genre.shooter.target-practice-minimal@1`、input closure、`result=pass`を全Fieldで持つ。canonical signed wrapper発行後だけ、上記Activation BindingがSystem ref＋四FieldQualification Receipt refを保持する。このdeterministic implementationはpredeclared query inputとtarget Stable IDからHit Evidenceを返し、World、Physics、Perception、render visibilityへ依存しない。Fixture System record owner、Bindingの同型`fixture_owner_ref`、Qualification subject、Activation Bindingはexact equalityであり、`owner_kind` enumや`GameSystemOwnerRefV1`との異型比較を行わない。Qualification sandbox内のSave／Load／Replay evidenceだけに使用でき、Production Source／Registry／Save／Packageへの選択を常に拒否する。

Project Binding用MCD Operationは次のclosed 3件であり、自由JSON writeまたはtemplate mutationを提供しない。

```text
ShooterTargetProviderBindingOperationCommonInputV1
  input_type_ref: exact selected named MCD input Type ref
  operation_ref: McdContractRefV1(kind=operation)
  project_ref: exact {project_id, expected_project_revision, document_set_hash}
  contract_set_hash
  operation_intent_hash
  request_hash
  idempotency_key
  preview_policy_ref: McdContractRefV1(kind=policy)
  validation_policy_ref: McdContractRefV1(kind=policy)
  authorization_binding: exact MutationAuthorizationBindingV2(risk_class=R2)

type.genre.shooter.target_provider_binding_create_input
  common
  allocation_scope
  relative_path
  binding_draft_without_binding_id
  binding_draft_hash
  template_ref/hash
  implementation_system_ref:
    exact UsageTaggedImplementationSystemRefV1(
      base_ref.usage=project_owned)
  target_data_ref/hash
  save_replay_contract_ref/hash

type.genre.shooter.target_provider_binding_update_input
  common
  current_binding_document_ref
  expected_document_revision
  before_content_hash
  before_binding_hash
  after_binding_with_same_binding_id
  after_binding_hash

type.genre.shooter.target_provider_binding_select_input
  common
  selection_document_ref/hash
  expected_selection_document_revision
  before_selection_hash
  derived_registry_ref/hash
  runtime_entry_ref/content_hash
  runtime_entry_semantic_hash
  recipe_ref/hash
  profile_ref/hash
  target_profile_ref/hash
  binding_document_ref/content_hash
  binding_hash

type.genre.shooter.target_provider_binding_mutation_result
  disposition: committed | rejected
  committed:
    mutation_kind: create | update
    before_project_ref
    after_project_ref
    binding_before_ref: ShooterTargetProviderBindingDocumentRefV1 | null
    binding_after_ref: ShooterTargetProviderBindingDocumentRefV1
    registry_before_ref: ShooterTargetProviderBindingRegistryRefV1
    registry_after_ref: ShooterTargetProviderBindingRegistryRefV1
    selection_document_before_ref: ShooterTargetProviderSelectionDocumentRefV1
    selection_document_after_ref: ShooterTargetProviderSelectionDocumentRefV1
    preview_receipt_ref/hash
    validation_receipt_ref/hash
    public_publication_marker_ref/hash: exact PublicPublicationMarkerV1
    mutation_receipt_ref/hash
  rejected:
    diagnostics[1..64]: DiagnosticCodeRefV1

type.genre.shooter.target_provider_binding_selection_result
  disposition: selected | rejected
  selected:
    before_project_ref
    after_project_ref
    selection_document_before_ref: ShooterTargetProviderSelectionDocumentRefV1
    selection_document_after_ref: ShooterTargetProviderSelectionDocumentRefV1
    registry_before_ref: ShooterTargetProviderBindingRegistryRefV1
    registry_after_ref: ShooterTargetProviderBindingRegistryRefV1
    selected_binding_ref: ShooterTargetProviderBindingDocumentRefV1
    selected_provider_binding_set_hash
    entry_branch_closure_hash
    preview_receipt_ref/hash
    validation_receipt_ref/hash
    public_publication_marker_ref/hash: exact PublicPublicationMarkerV1
    selection_receipt_ref/hash
  rejected:
    diagnostics[1..64]: DiagnosticCodeRefV1

PreparedShooterTargetProviderBindingMutationReceiptPayloadV1
  publication_binding: exact PreparedReceiptPublicationBindingV1
  operation_ref
  operation_intent_hash
  request_hash
  idempotency_key
  mutation_kind: create | update
  before_project_ref
  after_project_ref
  binding_before_ref: ShooterTargetProviderBindingDocumentRefV1 | null
  binding_after_ref: ShooterTargetProviderBindingDocumentRefV1
  registry_before_ref
  registry_after_ref
  selection_document_before_ref
  selection_document_after_ref
  owner_identity
  template_ref/hash
  implementation_system_ref:
    exact UsageTaggedImplementationSystemRefV1
  target_data_ref/hash
  save_replay_contract_ref/hash
  preview_receipt_payload_ref/hash
  validation_receipt_payload_ref/hash
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1
  diagnostics[0..64]: DiagnosticCodeRefV1
  prepared_payload_hash: SHA-256

ShooterTargetProviderBindingMutationReceiptV1
  published_receipt:
    exact PublishedDomainReceiptV2 whose prepared_domain_receipt_payload_ref/hash
    resolves PreparedShooterTargetProviderBindingMutationReceiptPayloadV1

PreparedShooterTargetProviderBindingSelectionReceiptPayloadV1
  publication_binding: exact PreparedReceiptPublicationBindingV1
  operation_ref
  operation_intent_hash
  request_hash
  idempotency_key
  before_project_ref
  after_project_ref
  selection_document_before_ref: ShooterTargetProviderSelectionDocumentRefV1
  selection_document_after_ref: ShooterTargetProviderSelectionDocumentRefV1
  registry_before_ref
  registry_after_ref
  selected_record: ShooterTargetProviderSelectedBindingRecordV1
  selected_provider_binding_set_hash
  entry_branch_closure_hash
  preview_receipt_payload_ref/hash
  validation_receipt_payload_ref/hash
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1
  diagnostics[0..64]: DiagnosticCodeRefV1
  prepared_payload_hash: SHA-256

ShooterTargetProviderBindingSelectionReceiptV1
  published_receipt:
    exact PublishedDomainReceiptV2 whose prepared_domain_receipt_payload_ref/hash
    resolves PreparedShooterTargetProviderBindingSelectionReceiptPayloadV1
```

各input／output／Receipt type refは`McdContractRefV1 {id, version=1, contract_set_hash}`である。`input_type_ref`は選択したcreate／update／select named Typeとexact equalityで、anonymous sibling shapeを許可しない。`operation_intent_hash`と`request_hash`は`MIRAKAN_OPERATION_INTENT_V2 -> MutationAuthorizationBindingV2 -> MIRAKAN_OPERATION_REQUEST_V2`の唯一のDAGを使い、Approval／Predelegationをfinal request hashへ逆向きにbindしない。createでは`binding_before_ref=null`、updateではbefore／afterのStable IDが同一、selectではSelection DocumentのStable IDが不変でSource recordが厳密に一件変わる。全成功でbefore／after Project IDが同一かつrevisionが一だけ増加する。

Result、Prepared payload、Source／Derived state、private Marker、`PublicCommitClosureV1`、signed Receipt、Public MarkerのProject／Binding／Selection／Registry／selected set hashはexact equalityである。公開schema、hash、順序は[Executable Contracts §8](../02-foundation/executable-contracts.md#8-operation定義)を唯一の正本とし、Shooter側へ同名型を複写しない。唯一の順序は`Preview → Validation → Prepared Candidate／Prepared Commit Envelope → staged postcondition → private durable commit marker read-back → Shooter ownerとreceipt-free Binding／Selection／Registry artifactを持つowner_typed_state_commitのsecret-free PublicCommitClosure candidate → canonical PublishedDomainReceiptV2／MirakanSignedRecordV1 wrapper → PublicCommitClosure＋PublicPublicationMarkerV1＋after Projectのatomic public CAS`である。private Marker時点またはClosure candidate storeだけではSource current head、Registry、Resultを公開せず、wrapper保存／readback後にだけ同じClosure body、Public Marker、after Projectをpublic CASする。成功ResultのPublic Markerとsigned Receiptは同一`PublicCommitClosureRefV1`／完成object hashを解決し、Ref内semantic hashと完成object SHAを相互代用しない。rejected branchはafter state、Closure、signed Receipt、Public Markerをcanonical omissionし、Project、Source Selection、Registry、Save、Compile Manifest、last-valid packageを変更しない。Prepared payload Typeはmutationがexact `type.genre.shooter.prepared_target_provider_binding_mutation_receipt_payload@1`、selectionがexact `type.genre.shooter.prepared_target_provider_binding_selection_receipt_payload@1`であり、最終Receipt Typeと相互代用しない。各payload hashは対応する`MIRAKAN_PREPARED_SHOOTER_TARGET_PROVIDER_*_RECEIPT_PAYLOAD_V1`とself-excluding canonical bytesから計算し、Domain signer／key／algorithm／signature Fieldをinline定義しない。

private Marker後かつwrapper前のcrashは同じsemantic hash／完成object SHAのClosure candidateと、固定materialization key／issued-at／revocation snapshot／deterministic signing profileによる同じwrapper bytesへroll-forwardする。wrapper後かつPublic Marker前は同じClosure＋Marker＋after Projectを同じexpected predecessorへpublic CASする。Public Marker後の別Closure、rollback、alternate signature、二重publication、unsigned prepared payload／Closure candidateだけによるcurrent head更新を拒否する。

同じidempotency key＋request hashのretryはbyte-identical Resultと同じClosure／signed Receipt／Public Marker ref／hashを返し、同じkeyの別requestは`MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE`で拒否する。

三Operationが使うDiagnostic Registry subsetを次へ固定する。Shooter固有九件は完全な`DiagnosticLocalRecordV2`であり、各rowは`diagnostic_version=1`、`owner_local_ref={owner_id=owner.genre.shooter,owner_revision=1,owner_content_hash=SHA-256(MIRAKAN_DIAGNOSTIC_OWNER_LOCAL_IDENTITY_V1, length-framed canonical owner ID／revision)}`、`requirement_local_refs=[]`、`message_key="<diagnostic_id>.message"`、Ownerを含むself-excluding `diagnostic_local_content_hash`を持つ。root確定後だけ同じ三Field Owner ref、`requirement_refs=[]`、別のself-excluding `diagnostic_content_hash`を持つ外部Registry recordへ投影する。共通八件はExecutable Contractsの既存owner付きrecordをexact reuseし、Shooter ownerへ付け替えない。Operationは表の四Fieldがexact equalityの`DiagnosticCodeRefV1`を保存する。

| diagnostic ID | code | severity／category／retryability |
|---|---|---|
| `diagnostic.conflict.revision_mismatch` | `MIRAKAN-CONFLICT-REVISION_MISMATCH` | blocking／conflict／after_change |
| `diagnostic.authorization.denied` | `MIRAKAN-AUTHORIZATION-DENIED` | blocking／permission／never |
| `diagnostic.approval.required` | `MIRAKAN-APPROVAL-REQUIRED` | blocking／permission／after_input |
| `diagnostic.authoring.lock_conflict` | `MIRAKAN-AUTHORING-LOCK_CONFLICT` | blocking／conflict／after_change |
| `diagnostic.mcd.operation_predicate_invalid` | `MIRAKAN-MCD-OPERATION-PREDICATE_INVALID` | blocking／schema／after_change |
| `diagnostic.operation.timeout` | `MIRAKAN-OPERATION-TIMEOUT` | error／infrastructure／transient |
| `diagnostic.operation.rate_limit_exceeded` | `MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED` | error／permission／transient |
| `diagnostic.operation.idempotency_key_reuse` | `MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE` | blocking／conflict／after_input |
| `diagnostic.genre.shooter.target_provider.identity_mismatch` | `MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_IDENTITY_MISMATCH` | blocking／semantic／after_input |
| `diagnostic.genre.shooter.target_provider.owner_mismatch` | `MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_OWNER_MISMATCH` | blocking／permission／after_change |
| `diagnostic.genre.shooter.target_provider.template_mismatch` | `MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_TEMPLATE_MISMATCH` | blocking／semantic／after_change |
| `diagnostic.genre.shooter.target_provider.type_mismatch` | `MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_TYPE_MISMATCH` | blocking／schema／after_input |
| `diagnostic.genre.shooter.target_provider.hash_mismatch` | `MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_HASH_MISMATCH` | blocking／conflict／after_change |
| `diagnostic.genre.shooter.target_provider.target_unsupported` | `MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_TARGET_UNSUPPORTED` | blocking／semantic／after_change |
| `diagnostic.genre.shooter.target_provider.fixture_in_production` | `MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_FIXTURE_IN_PRODUCTION` | blocking／permission／never |
| `diagnostic.genre.shooter.target_provider.registry_invalid` | `MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_REGISTRY_INVALID` | blocking／schema／after_change |
| `diagnostic.genre.shooter.target_provider.receipt_binding_mismatch` | `MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_RECEIPT_BINDING_MISMATCH` | blocking／semantic／after_change |

`ShooterTargetProviderBindingOperationErrorSetV1.refs[17]`は上表順のexact `{diagnostic_id, code, diagnostic_version=1, diagnostic_content_hash}`である。以下の各Operation MCDの`errors[]`はこの17値を実配列として保持し、ErrorSet ref、code文字列、prefix展開を保存しない。

| Operation MCD共通Envelope exact value | Operation固有Field exact value |
|---|---|
| `mcd_version=1; kind=operation; id=operation.shooter.target_provider_binding.create; version=1; status=active; title=Create Shooter Target Provider Binding; description=Create one identity-consistent Project-owned target-provider Binding Document; owners=[owner.genre.shooter]; requirement_refs=[]; rationale_refs=[mirakan.arch.pack-shooter#4-provider-binding]; since_contract_set=1; supersedes=[]; tags=[authoring,genre_shooter,target_provider_binding]` | `operation_kind=command; input_type={type.genre.shooter.target_provider_binding_create_input,1,contract_set_hash}; output_type={type.genre.shooter.target_provider_binding_mutation_result,1,contract_set_hash}; authority={service_id=service.authoring_command_gateway,service_version=1,service_content_hash}; risk_class=R2; side_effects=[authoring]; idempotency=idempotent_with_key; transaction=authoring_changeset; preconditions=[{policy.operation.shooter.target_provider_binding.create.precondition,1,contract_set_hash}]; postconditions=[{policy.operation.shooter.target_provider_binding.create.postcondition,1,contract_set_hash}]; errors=[{diagnostic.conflict.revision_mismatch,MIRAKAN-CONFLICT-REVISION_MISMATCH,1,diagnostic_content_hash},{diagnostic.authorization.denied,MIRAKAN-AUTHORIZATION-DENIED,1,diagnostic_content_hash},{diagnostic.approval.required,MIRAKAN-APPROVAL-REQUIRED,1,diagnostic_content_hash},{diagnostic.authoring.lock_conflict,MIRAKAN-AUTHORING-LOCK_CONFLICT,1,diagnostic_content_hash},{diagnostic.mcd.operation_predicate_invalid,MIRAKAN-MCD-OPERATION-PREDICATE_INVALID,1,diagnostic_content_hash},{diagnostic.operation.timeout,MIRAKAN-OPERATION-TIMEOUT,1,diagnostic_content_hash},{diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash},{diagnostic.operation.idempotency_key_reuse,MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.identity_mismatch,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_IDENTITY_MISMATCH,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.owner_mismatch,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_OWNER_MISMATCH,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.template_mismatch,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_TEMPLATE_MISMATCH,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.type_mismatch,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_TYPE_MISMATCH,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.hash_mismatch,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.target_unsupported,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_TARGET_UNSUPPORTED,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.fixture_in_production,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_FIXTURE_IN_PRODUCTION,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.registry_invalid,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_REGISTRY_INVALID,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.receipt_binding_mismatch,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_RECEIPT_BINDING_MISMATCH,1,diagnostic_content_hash}]; validator_closure_ref={validator_closure.operation.shooter.target_provider_binding.create,1,closure_content_hash}; timeout_ms=30000; rate_limit_policy={policy.authoring.shooter_target_provider_binding.rate_limit,1,contract_set_hash}; audit_level=full_redacted; provider_exposure=mcp_proposal; receipt_type={type.genre.shooter.target_provider_binding_mutation_receipt,1,contract_set_hash}` |
| `mcd_version=1; kind=operation; id=operation.shooter.target_provider_binding.update; version=1; status=active; title=Update Shooter Target Provider Binding; description=Update one Binding Document while preserving Stable identity; owners=[owner.genre.shooter]; requirement_refs=[]; rationale_refs=[mirakan.arch.pack-shooter#4-provider-binding]; since_contract_set=1; supersedes=[]; tags=[authoring,genre_shooter,target_provider_binding]` | `operation_kind=command; input_type={type.genre.shooter.target_provider_binding_update_input,1,contract_set_hash}; output_type={type.genre.shooter.target_provider_binding_mutation_result,1,contract_set_hash}; authority={service_id=service.authoring_command_gateway,service_version=1,service_content_hash}; risk_class=R2; side_effects=[authoring]; idempotency=idempotent_with_key; transaction=authoring_changeset; preconditions=[{policy.operation.shooter.target_provider_binding.update.precondition,1,contract_set_hash}]; postconditions=[{policy.operation.shooter.target_provider_binding.update.postcondition,1,contract_set_hash}]; errors=[{diagnostic.conflict.revision_mismatch,MIRAKAN-CONFLICT-REVISION_MISMATCH,1,diagnostic_content_hash},{diagnostic.authorization.denied,MIRAKAN-AUTHORIZATION-DENIED,1,diagnostic_content_hash},{diagnostic.approval.required,MIRAKAN-APPROVAL-REQUIRED,1,diagnostic_content_hash},{diagnostic.authoring.lock_conflict,MIRAKAN-AUTHORING-LOCK_CONFLICT,1,diagnostic_content_hash},{diagnostic.mcd.operation_predicate_invalid,MIRAKAN-MCD-OPERATION-PREDICATE_INVALID,1,diagnostic_content_hash},{diagnostic.operation.timeout,MIRAKAN-OPERATION-TIMEOUT,1,diagnostic_content_hash},{diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash},{diagnostic.operation.idempotency_key_reuse,MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.identity_mismatch,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_IDENTITY_MISMATCH,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.owner_mismatch,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_OWNER_MISMATCH,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.template_mismatch,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_TEMPLATE_MISMATCH,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.type_mismatch,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_TYPE_MISMATCH,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.hash_mismatch,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.target_unsupported,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_TARGET_UNSUPPORTED,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.fixture_in_production,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_FIXTURE_IN_PRODUCTION,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.registry_invalid,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_REGISTRY_INVALID,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.receipt_binding_mismatch,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_RECEIPT_BINDING_MISMATCH,1,diagnostic_content_hash}]; validator_closure_ref={validator_closure.operation.shooter.target_provider_binding.update,1,closure_content_hash}; timeout_ms=30000; rate_limit_policy={policy.authoring.shooter_target_provider_binding.rate_limit,1,contract_set_hash}; audit_level=full_redacted; provider_exposure=mcp_proposal; receipt_type={type.genre.shooter.target_provider_binding_mutation_receipt,1,contract_set_hash}` |
| `mcd_version=1; kind=operation; id=operation.shooter.target_provider_binding.select; version=1; status=active; title=Select Shooter Target Provider Binding; description=Select one exact Binding in the canonical Selection Document for an entry recipe profile and Target branch; owners=[owner.genre.shooter]; requirement_refs=[]; rationale_refs=[mirakan.arch.pack-shooter#4-provider-binding]; since_contract_set=1; supersedes=[]; tags=[authoring,genre_shooter,target_provider_binding]` | `operation_kind=command; input_type={type.genre.shooter.target_provider_binding_select_input,1,contract_set_hash}; output_type={type.genre.shooter.target_provider_binding_selection_result,1,contract_set_hash}; authority={service_id=service.authoring_command_gateway,service_version=1,service_content_hash}; risk_class=R2; side_effects=[authoring]; idempotency=idempotent_with_key; transaction=authoring_changeset; preconditions=[{policy.operation.shooter.target_provider_binding.select.precondition,1,contract_set_hash}]; postconditions=[{policy.operation.shooter.target_provider_binding.select.postcondition,1,contract_set_hash}]; errors=[{diagnostic.conflict.revision_mismatch,MIRAKAN-CONFLICT-REVISION_MISMATCH,1,diagnostic_content_hash},{diagnostic.authorization.denied,MIRAKAN-AUTHORIZATION-DENIED,1,diagnostic_content_hash},{diagnostic.approval.required,MIRAKAN-APPROVAL-REQUIRED,1,diagnostic_content_hash},{diagnostic.authoring.lock_conflict,MIRAKAN-AUTHORING-LOCK_CONFLICT,1,diagnostic_content_hash},{diagnostic.mcd.operation_predicate_invalid,MIRAKAN-MCD-OPERATION-PREDICATE_INVALID,1,diagnostic_content_hash},{diagnostic.operation.timeout,MIRAKAN-OPERATION-TIMEOUT,1,diagnostic_content_hash},{diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash},{diagnostic.operation.idempotency_key_reuse,MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.identity_mismatch,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_IDENTITY_MISMATCH,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.owner_mismatch,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_OWNER_MISMATCH,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.template_mismatch,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_TEMPLATE_MISMATCH,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.type_mismatch,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_TYPE_MISMATCH,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.hash_mismatch,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.target_unsupported,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_TARGET_UNSUPPORTED,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.fixture_in_production,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_FIXTURE_IN_PRODUCTION,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.registry_invalid,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_REGISTRY_INVALID,1,diagnostic_content_hash},{diagnostic.genre.shooter.target_provider.receipt_binding_mismatch,MIRAKAN-GENRE-SHOOTER-TARGET_PROVIDER_RECEIPT_BINDING_MISMATCH,1,diagnostic_content_hash}]; validator_closure_ref={validator_closure.operation.shooter.target_provider_binding.select,1,closure_content_hash}; timeout_ms=30000; rate_limit_policy={policy.authoring.shooter_target_provider_binding.rate_limit,1,contract_set_hash}; audit_level=full_redacted; provider_exposure=mcp_proposal; receipt_type={type.genre.shooter.target_provider_binding_selection_receipt,1,contract_set_hash}` |

三Operationが参照する七Policyは次の完全なactive MCD recordである。表の二列を連結した値が各record全体であり、暗黙既定値、bare ID、説明文からFieldを補完しない。

| Policy MCD共通Envelope exact value | Policy payload exact value |
|---|---|
| `mcd_version=1; kind=policy; id=policy.operation.shooter.target_provider_binding.create.precondition; version=1; status=active; title=Create Shooter Target Provider Binding Precondition; description=Validate the Project base, binding draft identity, template, project-owned implementation branch, target data, Save Replay contract, authorization, and idempotency snapshot without mutation; owners=[owner.genre.shooter]; requirement_refs=[]; rationale_refs=[mirakan.arch.pack-shooter#4-provider-binding]; since_contract_set=1; supersedes=[]; tags=[genre_shooter,operation_predicate,pure,target_provider_binding]` | `evaluation_mode=pure; side_effects=[]; input_type={id=type.operation.precondition_evaluation_input,version=1,contract_set_hash}; result_type={id=type.operation.predicate_result,version=1,contract_set_hash}` |
| `mcd_version=1; kind=policy; id=policy.operation.shooter.target_provider_binding.create.postcondition; version=1; status=active; title=Create Shooter Target Provider Binding Postcondition; description=Validate the unpublished Binding Document, Registry, Selection closure, prepared Receipt payload, and atomic Project revision increment; owners=[owner.genre.shooter]; requirement_refs=[]; rationale_refs=[mirakan.arch.pack-shooter#4-provider-binding]; since_contract_set=1; supersedes=[]; tags=[genre_shooter,operation_predicate,pure,target_provider_binding]` | `evaluation_mode=pure; side_effects=[]; input_type={id=type.operation.postcondition_evaluation_input,version=2,contract_set_hash}; result_type={id=type.operation.predicate_result,version=1,contract_set_hash}` |
| `mcd_version=1; kind=policy; id=policy.operation.shooter.target_provider_binding.update.precondition; version=1; status=active; title=Update Shooter Target Provider Binding Precondition; description=Validate the current Binding Document identity, revision, content and semantic hashes, project-owned implementation branch, owner, authorization, and Project base without mutation; owners=[owner.genre.shooter]; requirement_refs=[]; rationale_refs=[mirakan.arch.pack-shooter#4-provider-binding]; since_contract_set=1; supersedes=[]; tags=[genre_shooter,operation_predicate,pure,target_provider_binding]` | `evaluation_mode=pure; side_effects=[]; input_type={id=type.operation.precondition_evaluation_input,version=1,contract_set_hash}; result_type={id=type.operation.predicate_result,version=1,contract_set_hash}` |
| `mcd_version=1; kind=policy; id=policy.operation.shooter.target_provider_binding.update.postcondition; version=1; status=active; title=Update Shooter Target Provider Binding Postcondition; description=Validate the same Stable ID at a new unpublished Document revision, Registry rematerialization, prepared Receipt binding, and atomic Project revision increment; owners=[owner.genre.shooter]; requirement_refs=[]; rationale_refs=[mirakan.arch.pack-shooter#4-provider-binding]; since_contract_set=1; supersedes=[]; tags=[genre_shooter,operation_predicate,pure,target_provider_binding]` | `evaluation_mode=pure; side_effects=[]; input_type={id=type.operation.postcondition_evaluation_input,version=2,contract_set_hash}; result_type={id=type.operation.predicate_result,version=1,contract_set_hash}` |
| `mcd_version=1; kind=policy; id=policy.operation.shooter.target_provider_binding.select.precondition; version=1; status=active; title=Select Shooter Target Provider Binding Precondition; description=Validate the Selection Document, current Registry, Runtime Entry, Recipe, Profile, Target, exact Project-owned Binding, authorization, and Project base without mutation; owners=[owner.genre.shooter]; requirement_refs=[]; rationale_refs=[mirakan.arch.pack-shooter#4-provider-binding]; since_contract_set=1; supersedes=[]; tags=[genre_shooter,operation_predicate,pure,target_provider_binding]` | `evaluation_mode=pure; side_effects=[]; input_type={id=type.operation.precondition_evaluation_input,version=1,contract_set_hash}; result_type={id=type.operation.predicate_result,version=1,contract_set_hash}` |
| `mcd_version=1; kind=policy; id=policy.operation.shooter.target_provider_binding.select.postcondition; version=1; status=active; title=Select Shooter Target Provider Binding Postcondition; description=Validate the unpublished Selection source, rematerialized Registry, selected record, entry closure, prepared Receipt payload, and atomic Project revision increment; owners=[owner.genre.shooter]; requirement_refs=[]; rationale_refs=[mirakan.arch.pack-shooter#4-provider-binding]; since_contract_set=1; supersedes=[]; tags=[genre_shooter,operation_predicate,pure,target_provider_binding]` | `evaluation_mode=pure; side_effects=[]; input_type={id=type.operation.postcondition_evaluation_input,version=2,contract_set_hash}; result_type={id=type.operation.predicate_result,version=1,contract_set_hash}` |
| `mcd_version=1; kind=policy; id=policy.authoring.shooter_target_provider_binding.rate_limit; version=1; status=active; title=Shooter Target Provider Binding Rate Limit; description=Bound binding mutations per principal and Project without changing binding semantics; owners=[owner.genre.shooter]; requirement_refs=[]; rationale_refs=[mirakan.arch.pack-shooter#4-provider-binding]; since_contract_set=1; supersedes=[]; tags=[authoring,genre_shooter,rate_limit,target_provider_binding]` | `policy_ref={id=policy.authoring.shooter_target_provider_binding.rate_limit,version=1,contract_set_hash}; scope=principal_project; window_ns=60000000000; max_requests=60; burst=10; exceeded_error_ref={diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash}` |

Contract set snapshot内部では三Operation、七Policy、Service allowlist、Validator／closureの相互edgeをLocalRefで表す。Pack Manifest三Operation LocalRef = Shooter owner active MCD Operation集合 = `service.authoring_command_gateway`へのShooter owner allowlist contributionはexact三件、各Operationのpre／postと共有rate refのunion = Shooter owner Policy local subsetはexact七件である。`provider_exposure=mcp_proposal`の三Operation集合とProvider／MCP projection集合もexact三件でset equalityにする。Pack removal時は同じRegistry transactionでこれらのexact owner subsetを除去する。set root確定後にだけ外部Service／MCD refをmaterializeする。七Policyそれぞれの共通Envelopeまたはpayloadの実在Fieldを一つだけ変えるfixtureはPolicy member hashとset rootを変更し、旧Operation／Manifest／Service／Provider external refを解決不能にする。stale Service hash、Policy kind／version／Contract set root、rate payload不一致をRegistry compile errorにする。

Shooter Validator Registryのowner subsetは次のexact八recordだけである。全recordは`owner_ref=owner.genre.shooter`、`validator_version=1`、exact implementation Artifact ref／hash、表のinput Type ref、表の実配列`error_refs[]`、self-excluding `validator_content_hash`を持つ。`composition_invalid`と`perception_binding_invalid`はそれぞれexact `DiagnosticCodeRefV1` `{diagnostic.genre.shooter.composition.invalid,MIRAKAN-GENRE-SHOOTER-COMPOSITION_INVALID,1,diagnostic_content_hash}`、`{diagnostic.genre.shooter.perception_binding.invalid,MIRAKAN-GENRE-SHOOTER-PERCEPTION_BINDING_INVALID,1,diagnostic_content_hash}`であり、Operationの17-ref setには含めない。

| Shooter-owned Validator ID | exact input Type | exact `error_refs[]` |
|---|---|---|
| `validator.genre.shooter.composition` | `type.genre.shooter.composition_recipe@1` | `diagnostic.genre.shooter.composition.invalid; diagnostic.genre.shooter.profile.scale_underspecified` |
| `validator.genre.shooter.perception_binding` | `type.genre.shooter.perception_binding@1` | `diagnostic.genre.shooter.perception_binding.invalid` |
| `validator.genre.shooter.target_provider_binding.create_semantics` | `type.genre.shooter.target_provider_binding_create_input@1` | `identity_mismatch; owner_mismatch; template_mismatch; type_mismatch; target_unsupported; fixture_in_production` |
| `validator.genre.shooter.target_provider_binding.create_postcondition` | `type.operation.postcondition_evaluation_input@2` | `hash_mismatch; registry_invalid; receipt_binding_mismatch` |
| `validator.genre.shooter.target_provider_binding.update_semantics` | `type.genre.shooter.target_provider_binding_update_input@1` | `identity_mismatch; owner_mismatch; template_mismatch; type_mismatch; target_unsupported; fixture_in_production` |
| `validator.genre.shooter.target_provider_binding.update_postcondition` | `type.operation.postcondition_evaluation_input@2` | `hash_mismatch; registry_invalid; receipt_binding_mismatch` |
| `validator.genre.shooter.target_provider_binding.select_semantics` | `type.genre.shooter.target_provider_binding_select_input@1` | `identity_mismatch; owner_mismatch; template_mismatch; type_mismatch; target_unsupported; fixture_in_production` |
| `validator.genre.shooter.target_provider_binding.select_postcondition` | `type.operation.postcondition_evaluation_input@2` | `hash_mismatch; registry_invalid; receipt_binding_mismatch` |

表中の短いdomain error名はすべて`diagnostic.genre.shooter.target_provider.<name>`の上表17-ref exact四Fieldへ展開し、bare stringを保存しない。`diagnostic.genre.shooter.profile.scale_underspecified`は§14.3のexact ID／code／version／hashである。Manifest `validator_refs[]` = Shooter Validator Registry owner subset = この八record、をID／version／content hashでset equalityとし、missing／extra／generic create名／duplicate／stale hashをPack activation前に拒否する。

三`OperationValidatorClosureV1`を次へ固定する。各validator refは`{validator_id,validator_version=1,validator_content_hash}`である。

| closure／operation | exact validators | reachable errors |
|---|---|---|
| `validator_closure.operation.shooter.target_provider_binding.create`／create v1 | `validator.operation.request_envelope; validator.operation.authorization; validator.operation.approval; validator.operation.revision_and_lock; validator.operation.pure_predicate; validator.operation.timeout_and_rate_limit; validator.genre.shooter.target_provider_binding.create_semantics; validator.genre.shooter.target_provider_binding.create_postcondition` | exact create `errors[]` 17-ref set |
| `validator_closure.operation.shooter.target_provider_binding.update`／update v1 | `validator.operation.request_envelope; validator.operation.authorization; validator.operation.approval; validator.operation.revision_and_lock; validator.operation.pure_predicate; validator.operation.timeout_and_rate_limit; validator.genre.shooter.target_provider_binding.update_semantics; validator.genre.shooter.target_provider_binding.update_postcondition` | exact update `errors[]` 17-ref set |
| `validator_closure.operation.shooter.target_provider_binding.select`／select v1 | `validator.operation.request_envelope; validator.operation.authorization; validator.operation.approval; validator.operation.revision_and_lock; validator.operation.pure_predicate; validator.operation.timeout_and_rate_limit; validator.genre.shooter.target_provider_binding.select_semantics; validator.genre.shooter.target_provider_binding.select_postcondition` | exact select `errors[]` 17-ref set |

各Operationについて`common六Validator error union ∪ 当該semantics Validator error_refs ∪ 当該postcondition Validator error_refs = closure reachable_error_refs = Operation errors[] = ShooterTargetProviderBindingOperationErrorSetV1.refs[17]`をID／code／version／hashでset equalityにする。Result／Receiptの全Diagnosticも同じ17-ref setのsubsetである。Manifest inventory equalityとper-operation error equalityを混ぜず別gateで検証する。missing、到達不能extra、同ID別code、同code別ID、stale Diagnostic／Validator hashを一原因ずつ拒否し、Registry sort／duplicate、selected set hash、Result／Receipt／request bindingを生成するDomain validatorから全domain codeへ実到達するfixtureを持つ。

`implementation_system_ref`は[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md#3-gamesystemspecv2)のusage-tagged unionをexact reuseする。Project-owned branchはactive `GameSystemCatalogV1`の`game_system.project.<project_namespace>.<path>` ref／contract hash／Game System Activation Bindingだけを持ち、`owner_layer=project`、owner identity、Ranged Combat ownerのCollision Query Port command受理、Shot Hit Event emit、Target data read、Save Replay contract、Target Profileを照合する。fixture-only branchはProduction Catalogに属さない`FixtureImplementationSystemRefV1`＋Fixture System Activation Bindingだけを持ち、Registry record、Qualification subject、Bindingの同型fixture ownerを照合する。Genre Pack自身がProject implementationを暗黙生成しない。System branch、owner、State、Port type、Save Replay、Activation Bindingの一つでも不一致ならBinding Registry materialization前にrejectする。

top-down／third-personのenemy finite 2 Profileだけが次のGenre-owned perception recordを使う。target-practice Profileは選択RecipeにPerception Capabilityがなく、このbindingを要求しない。

```text
ShooterPerceptionBindingV1
  binding_id
  enemy_archetype_ref
  perception_profile_ref
  hostile_team_filter_ref
  lost_target_behavior: search_last_known | return_to_route
  fire_intent_policy_ref
```

`perception_profile_ref`はPerception ownerのactive `PerceptionProfileV1`へexactに解決し、target selection policyと`hostile_team_filter_ref`の互換性を検証する。`lost_target_behavior`は上記closed enumだけを許可し、`search_last_known`はbounded Perception memory、`return_to_route`はPath Following portのtyped requestだけを読む。`fire_intent_policy_ref`はvisibleまたは記憶済みtarget evidenceから`RequestFireCommandV1`を生成するpolicyへ解決し、target消失後のfire継続条件と最大Simulation Advance数を明示しなければならない。Render visibility、Camera frustum、Audio mixer、VFX、native Physics objectをtarget／fire authorityにしたbindingを拒否する。

Camera、Audio、LOD Profileのparameter schemaと実行規則は各Subsystem Ownerが所有する。Shooter Profileは参照とGenre固有の組合せだけを所有し、Camera rig、Audio voice、representation selectionをGameplay authorityにしない。

## 5. Shooter Game Flow

Shooter Genre Packが所有するclosed stateは`Ready | Playing | Paused | Result`である。

Shooter Packは`RuntimeScopeTypeCatalogV1`へ次のexact rowを登録する。

| `scope_type_ref` | `instance_key_schema_ref` | `owner_ref` | `lifetime_ref` | `save_replay_policy_ref` | `activation_condition_ref` | `deactivation_condition_ref` |
|---|---|---|---|---|---|---|
| `scope.genre.shooter.game_flow.instance` | `type.runtime_scope.key.genre_shooter_game_flow_uuidv7` | `owner.genre.shooter` | `policy.runtime_scope.lifetime.genre_shooter_game_flow_instance` | `policy.runtime_scope.save_replay.genre_shooter_game_flow` | `policy.runtime_scope.activation.genre_shooter_game_flow_entry_ready` | `policy.runtime_scope.deactivation.genre_shooter_game_flow_stop_or_fault` |

保存値はversion／hash付き`RuntimeScopeTypeRefV1`、`McdContractRefV1`、`RuntimeScopeOwnerRefV1`であり、表のIDだけを永続化しない。全dependencyをactive Scope Registryへ登録する。Shooter内部Systemだけがこのscopeを使用できる。旧generic `play_session`と末尾`.instance`を欠くGenre aliasへGame Flow stateを保存せず、Scope Save identity、Replay identity、ephemeral runtime generationを分離する。

`runtime_scope.migration_contribution.genre.shooter.game_flow@1`はcurrent Pack memberではなく、Gameplay Programming Model §3.1.2のconditional legacy migrationがsigned `LegacyMigrationInventoryV1` gateを満たしてatomic activationされる場合だけ追加できるdestination candidateである。現時点のContribution／Qualification subject／Receipt／Binding／Activation Catalog／migration fixture inventoryはすべてexact `[]`で、`fixture.genre.shooter.runtime_scope_migration@1`もcurrent `test_scenario_refs[]`へ含めない。Activation transactionではShooter ownerが`RuntimeScopeMigrationContributionRegistryV1`へ同recordを登録する。Production contribution destinationはowner `owner.genre.shooter`、Inventoryが束縛したexact legacy Shooter Game Flow System ref／hashを検査するsource match policy、source schema `type.game_system.spec` version 1、destination version 2、legacy value `play_session`、destination `scope.genre.shooter.game_flow.instance`、Shooter-owned auxiliary／identity migration policyを持つReceipt-free recordである。Registry／ContributionRef固定後に`qualification.runtime_scope_migration.genre.shooter.game_flow@1` subject／signed Receiptと`qualification_binding.runtime_scope_migration.genre.shooter.game_flow@1`をroot外で作る。Fixture bodyはsubjectだけが`fixture.genre.shooter.runtime_scope_migration`を解決する。末尾`.instance`欠落alias、同じ`play_session`を持つ非Shooter System、owner／Binding／Receipt／policy hash stale、Save／Replay mapping衝突をShooter Qualificationで拒否し、Core migration table／binary／Fixture inventoryへShooter IDを追加しない。

| From | To | 条件 |
|---|---|---|
| `Ready` | `Playing` | selected entry／Stageのactivationが成功 |
| `Playing` | `Paused` | Pause Capabilityが有効でpolicyが許可 |
| `Paused` | `Playing` | 同じsessionをresume |
| `Playing` | `Result` | Scenario／StageまたはProject-owned policyがtyped outcomeを提出 |
| `Result` | `Ready` | restartが新しいplay attemptを作成 |

表外の遷移とdefault transitionを拒否する。Title、Settings、Menu、LoadingはUI／Runtime entryの状態でありShooter Game Flow stateへ追加しない。Pause非対応Projectでは`Playing -> Paused`を適用せず、Runtime ownerのtyped failureを返す。

Game FlowはFeature Stateを直接writeしない。開始、pause、resume、completion、restartを各Feature ownerへのtyped CommandとSnapshotで接続し、UI visibility、Audio completion、Animation Eventからauthorityを推測しない。Save／Replayは登録済みState ownerと各FeatureのSave／Replay契約が宣言したfieldだけを記録する。

`ShooterGameFlowInteractionEligibilityPolicyV1`は`scope.genre.shooter.game_flow.instance`のsnapshotを読み、Interaction ownerのgeneric eligibility policy contractへallowまたはdenyを返す。deny時のcommon Interaction結果は`policy_denied`であり、Feature側へShooter state、enum、dependencyを追加しない。このpolicyを選択しないneutral InteractionはShooter Game Flowを要求しない。

## 6. Shooter Action role

Shooter Packは次のsemantic roleを提供する。

- `shooter.move`
- `shooter.aim`
- `shooter.look`
- `shooter.fire_primary`
- `shooter.fire_secondary`
- `shooter.aim_mode`
- `shooter.reload`
- `shooter.next_weapon`
- `shooter.previous_weapon`
- `shooter.pause`

`ShooterMinimalActionRoleSetV1`は`shooter.fire_primary`だけを必須とし、他roleをoptionalにする。Genre vocabularyである点とInput ownerによるStable Action ID生成はfull role setと同じである。

Action roleはGenre vocabularyでありRuntime identityではない。[Input](../07-platform/input.md)がProject適用時にActionのStable ID、value schema、binding、remap、snapshotを所有する。Profileは必要roleとInput templateを宣言できるが、Platform key code、device、dead zone、gestureをShooter契約へ固定しない。

## 7. AI composition

AIは[Pack Contract](pack-contract.md)のplanned Pack actionを使い、要求をGenre identity、Profile、Feature Capability closure、Target、fixtureへ解決する。Pack authoringのcurrent MCD Operation集合は空であり、Activationまではmutation要求を`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`で拒否する。2D／3D、finite／endless、input modality、Camera、required Featureが未確定でGame構造が変わる場合は`question_required`とし、類似表示名からGenreやProfileを推測しない。

Previewは少なくとも次をexact IDで示す。

- selected Genre identityとProfile
- Feature Pack dependency closure
- Camera／Audio／LOD binding
- Action roleから生成するInput Action
- Scenario／Stageの有無とcompletion policy
- Target support、budget、fixture、fallback

AIはFeature schemaをShooter schemaとして複写せず、Feature OwnerのCatalog entry、Operation、Validatorを使用する。

## 8. Fixtureとconformance

### 8.1 `fixture.product.shooter-2d`

`genre.shooter.top_down_2d`、`profile.shooter.top_down_2d`、Feature Capability closureを使い、ReadyからPlaying、Paused／resume、Result、restartを検証する。Damage／Vital／Faction、Ranged Combat、Encounter／Spawn、Scoring、Pickup／Grant、Interaction、Character Locomotion、Path Following、Scenario／Stageが各Feature OwnerのPublic Contractだけを使用することを確認する。Perception境界の出入り、`search_last_known`／`return_to_route`、target消失後のfire-intent停止advanceをfixed inputで検証する。

### 8.2 `fixture.product.shooter-arena-3d`

`genre.shooter.third_person_3d`、`profile.shooter.third_person_3d`、同じFeature Capability semanticsを使い、third-person Camera／Audio／LOD binding、3D locomotion、path following、Save／Load、Replay、Resultまでを検証する。target取得／lost-target／fire-intentのEvent sequenceとReplay hashが2Dと同じowner contractに適合することを確認し、Feature schemaをforkしない。

### 8.3 `fixture.genre.shooter.endless-top-down-2d`

`recipe.shooter.top_down_2d.endless`を選択し、Task 1時点のEncounter、Scoring、Pickup、Interaction、Locomotion、Path、Perception closureを維持しつつ、effective closureとclosure hashに`feature.scenario_stage@1`が含まれないこと、World／Scene activation後にObjective／Completion／Result routeなしでPlayingを継続できること、Pack registry上にScenario／Stage FeatureがなくてもRecipe apply／Save／Load／Replayが成功することを検証する。

### 8.4 `fixture.genre.shooter.target-practice-minimal`

`recipe.shooter.target_practice.minimal`、`profile.shooter.target_practice`の`genre_ref=genre.shooter.top_down_2d`、Pack-owned `template.genre.shooter.target_provider.stationary`、Fixture Registryのfixture-only Binding recordを選択し、effective closureが`feature.ranged_combat@1`と推移依存`feature.combat@1`だけであることを検証する。fixture-only Collision Query Port implementationがWorld／Physics／Perceptionなしにdeterministic Hit Evidenceを返し、Fire／Hit／Damage、Qualification sandbox内のSave／Load／Replayが成功する。ただしfixture bindingをProduction Source／Registry／Save／Packageへ選択するnegative caseは必ずrejectする。

`fixture.genre.shooter.target-practice-minimal-no-perception`はPerception Validator／Profile、full 2D／TPS fixture、Scenario／Stage、Scoring、Character Locomotion、Path Following、Encounter、Pickup、Interactionを未installにしたregistryからminimal Recipeだけをapply／qualifyするregression fixtureである。Manifest inventoryに`validator.genre.shooter.perception_binding`やfull fixtureが存在しても選択Recipe gateへ追加されないことを検証する。

`fixture.genre.shooter.target-practice-minimal-project-provider`はProduction modeでProject revision NのOperation inputからBinding Documentをcreateし、`private Marker read-back→owner-typed PublicCommitClosure candidate→canonical signed wrapper→Closure＋Public Marker＋Projectのatomic CAS`をreadbackしてN+1のrevision／document set hashを確認した後、N+1をreload→select→cook→Play→Save／Load→Replayする。各crash window、Closure欠落／差替え、semantic hashと完成object SHAの混同、署名前public、alternate signature、二重publicationをrejectする。続けてBindingと無関係なDocumentだけを編集したN+2を作り、binding semantic hashがN+1と同一のまま、N+2 Registry membershipとCompile closureのrevision／document set hashだけを更新して再compileできることを検証する。payloadへN+1／N+2 revisionを戻さずfixed pointを作らない。Project-defined logical target implementationはWorld、Physics、Perceptionを一切installせず、Ranged Combat ownerのexact Collision Query Port／Shot Hit Event type、Project-owned System、target dataだけでdeterministic Hit Evidenceを返す。Compile Manifestとentry closureが`selected_provider_binding_set_hash`を持ち、binding／template／System／Save Replay／target data hashのいずれかを変更するとpackageとReplay closureがinvalidateされることを検証する。

### 8.5 Negative fixture

- Shooter Packから別Genre Packへのdependencyを拒否する。
- Feature CapabilityのSchema／State ownerをShooter側で再宣言したrecipeを拒否する。
- missing／incompatible `PerceptionProfileV1`、unknown lost-target value、unbounded fire-intent継続を拒否する。
- Shooter role文字列をInput Action identityまたはSave identityにしたProjectを拒否する。
- Shooter PackなしのFeature-only Projectをvalidとする。
- Shooter Pack removal後もCore／Editor／AI／Project source／Build／Packageを成功させる。
- minimal Recipeへ未宣言のPerception／Stage／Score／Locomotionをclosure resolverが追加する実装を拒否する。
- fixture-only bindingのProduction選択、cross-project Document index containment、payload project ID self-assert spoof、current Registry membership欠落、Operation inputのstale base revision／document set hash、Binding identity三者不一致、payloadへのProject revision／document set hash混入、template／implementation／Save Replay hash mismatchに加え、implementation両branch／両方欠落、usageとbranch不一致、fixture owner mismatch、stale Fixture Registry／record／Qualification subject／signed Receipt／Activation Binding hash、Fixture System refのProduction Source／Registry／Save／Replay／Package混入を各一原因で拒否する。
- 既存2D／TPS RecipeからTask 1時点のFeature closureまたはPerception bindingが欠落した場合、qualificationを拒否する。

Capability成熟度、Phase、Target別Activation、Product claimは[Product Plan](../00-product/product-plan.md)だけが所有する。

## 9. Shooter Difficulty Profile detail

Difficultyは`easy`、`normal`、`hard`等の表示名だけでは成立しない。Profileは変更できるField ref、基準値、演算、上限、Gameplay fidelity floorを次のcanonical schemaで明示する。

```text
DifficultyProfileV1
  difficulty_profile_id: StableId
  axes[1..16]:
    axis_field_ref
    base_value
    operation: add | multiply_q16 | replace
    clamp_min
    clamp_max
  gameplay_fidelity_floor_ref
```

`axis_field_ref`は下記C1軸のFieldだけを参照でき、それ以外をvalidatorが拒否する。`multiply_q16`の倍率は[Gameplay Features §3.3](gameplay-features.md)と同域のunsigned Q16 `[0.0625,16]`とする。適用結果は`clamp_min`／`clamp_max`で有限に固定し、NaN／Infを生成しない。`gameplay_fidelity_floor_ref`はDifficultyが下回れないGameplay fidelity(Collision有効、Hit Event非drop、authoritative objectの非削除、Replay決定性)の宣言を指す。

C1で変更可能な軸は次に限定する。

- enemy max Health
- enemy base Damage
- Encounter spawn interval／group count
- Shot Pattern／cadenceの選択
- pickup量
- Score multiplier

Difficulty変更でCollision無効化、Hit Event drop、敵やProjectileの無断削除、Replay非決定化を行わない。

## 10. Input Action Template

Shooter PackはActionの表示名ではなくdomain-specific Semantic roleを提供し、Action identity、Value schema、Binding、Remap、Snapshotは[Input](../07-platform/input.md)へ委譲する。Project適用時はInput ownerを通じてAction `StableId`を生成する。

| Semantic role | Value | 2D | TPS | Required |
|---|---|---:|---:|---:|
| `shooter.move` | axis2 | yes | yes | yes |
| `shooter.aim` | axis2／pointer2 | yes | no | yes for 2D |
| `shooter.look` | axis2 | no | yes | yes for TPS |
| `shooter.fire_primary` | digital | yes | yes | yes |
| `shooter.fire_secondary` | digital | optional | optional | no |
| `shooter.aim_mode` | digital | no | yes | yes for TPS |
| `shooter.reload` | digital | optional | yes | Profile |
| `shooter.next_weapon` | digital | optional | yes | Profile |
| `shooter.previous_weapon` | digital | optional | yes | Profile |
| `shooter.pause` | digital | yes | yes | yes |

Actionから直接Weapon Stateを書き換えず、Input SnapshotをControl／Weapon intent evaluatorがtyped Commandへ変換する。AI、Replay、Controller、Keyboard／Mouse、Touchは同じsemantic Actionを使う。

User Remap、toggle／hold、sensitivity、dead zone、left-handed layoutはInput規約が所有する。Difficulty ProfileがUser Remapを変更しない。

## 11. PresentationとGameplay分離

一つのauthoritative Eventから次を独立配送する。

| Event | UI | Audio | VFX | Camera |
|---|---|---|---|---|
| Fire accepted | ammo／reticle | shot／mechanical | muzzle／shell／trail | recoil channel |
| Shot hit | hit marker | impact | impact／decal | bounded shake |
| Damage applied | Health／damage number | damage cue | hit reaction | bounded shake |
| Defeat | score／result | defeat cue | defeat effect | director request |
| Reload | ammo progress | reload cue | optional prop | none |

Presentation cueの失敗、Voice不足、VFX drop、Camera unavailableでFire、Damage、Score結果を変更しない。critical cueがownerのcapacity内で出せない場合はPresentation Qualificationを失敗させるが、Gameplay Eventをdropしない。

Camera recoilは現在のcontractではPresentation-onlyであり、Gameplay aim、Shot direction、Collision query、Save transformへ戻さない。authoritative recoilは対象外であり、将来scope、activation、schema／Profile判断は[Product Plan](../00-product/product-plan.md)だけが所有する。

## 12. AI Semantic Contract

### 12.1 `ShooterIntentResolutionV1`

```text
ShooterIntentResolutionV1
  requirement_refs[]
  source_terms[]
  canonical_concept_refs[]
  selected_genre_pack_ref
  selected_profile_ref
  weapon_intents[]
  shot_delivery_intents[]
  fire_mode_intents[]
  ammo_reload_intents[]
  damage_intents[]
  team_vital_intents[]
  pickup_intents[]
  encounter_intents[]
  score_intents[]
  game_flow_intents[]
  input_intents[]
  presentation_intents[]
  target_profiles[]
  scale_intent_ref
  assumptions[]
  blocking_questions[]
  high_impact_questions[]
  capability_gaps[]
  forbidden_approximations[]
  validation_scenarios[]
```

ResolverはSource termごとにEvidence Requirement IDとconfidenceを返す。confidenceの平均値でBlocking不足を隠さない。

### 12.2 必須質問

次が未指定で、候補間にGameplay上の差がある場合だけ質問する。

- 2D top-down、TPS、その他のview／movementか
- hitscan、projectile、両方か
- infinite ammoかmagazine／reloadか
- friendly fireを許可するか
- score／combo／high scoreが目的か
- Boss、Wave、stage completionの有無
- 最大同時敵、Projectile、spawn burst
- Windows、Mobile、Controller、Touchの対象
- visual violence、flash、shake、color依存の制約

既定Profileで安全に解決できるLow Impact項目はAssumptionとしてPreviewへ出し、質問しない。

### 12.3 禁止する推測

- 「弾」をParticleだけで実装する。
- 「銃」をhitscanと決める。
- 「連射」をautomatic、burst、Pattern同時発射のどれかへ無言で決める。
- 「強くする」をDamage、cadence、range、spread、ammo、feedbackの全部へ適用する。
- 「弾幕」をVFX particle countだけ増やして完成とする。
- 「軽くする」をProjectile、敵、Damage、Hit判定の削減へ変換する。
- 「オンライン風」「対戦風」からnetworkingを有効にする。
- Camera shake、recoil、hit flashをGameplay aim／Damageへ使う。

### 12.4 Future AI authoring vocabulary（not activated）

この節の九actionは要求分類用のfuture vocabularyであり、MCD `operation` identityではない。従来のname-only `operation.shooter.*`表記は一度もactivateされておらず、current MCD、Pack Manifest、Service allowlist、Provider／MCP Catalogから除外し、legacy aliasとしても読まない。current Shooter Operation setは§4の完全登録済みtarget-provider Binding三件だけである。

| Future vocabulary token | 想定Risk | 想定結果 |
|---|---:|---|
| `future.shooter.action.search_catalog` | R0 | Concept、Capability、Definition、Qualification候補 |
| `future.shooter.action.resolve_intent` | R0 | `ShooterIntentResolutionV1` |
| `future.shooter.action.explain_resolution` | R0 | Field、理由、Assumption、代替、Evidence |
| `future.shooter.action.preview_simulation` | R0 | Headless／visual Preview、Cost、Event trace |
| `future.shooter.action.compose_recipe` | R2 | Feature ref、Profile、Action role、統合Qualification binding |
| `future.shooter.action.configure_game_flow` | R1 | Ready／Pause／Result／Restart差分 |
| `future.shooter.action.configure_difficulty` | R2 | 許可軸、fidelity floor、capacity差分 |
| `future.shooter.action.apply_profile` | R2 | Pack／Input／UI／Asset closure |
| `future.shooter.action.propose_native_variant` | R3 | Native Source、Test、capacity evidence、Promotion案 |

Capability stateは九actionすべて`not_activated`で、要求は`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`としてSource不変で拒否する。future work item `activation.shooter.ai_authoring.v1`は採用するexact Operation set、MCD全Field、named input／result、Service／Policy／Validator／Diagnostic／Receipt、Risk、authorization intent DAG、private-to-public recovery、Qualificationを同じContract set transactionで完全登録するまでactivateしない。Weapon、Fire Mode、Shot、Damage、Vital、Pickup、Encounter、Scoreその他のFeature Source mutationも、Feature authoring Capabilityが別途完全登録されるまで非活性であり、ShooterがFeature contractのFieldを直接writeしない。

AI ProviderへC++ pointer、Runtime handle、unbounded projectile list、raw Physics objectを渡さない。Search／Readは必要なCatalog entryとbounded Snapshotだけを返す。

## 13. Editor／AI UX

Shooter Workspaceは専用別Editorではなく、既存Scene、Outliner、Inspector、Graph、Table、AI Partnerへ次のProjectionを追加する。

- Weapon Inspector: cadence、ammo、reload、Delivery、Damage、Cue
- Shot Pattern Preview: origin、direction、Simulation Advance、capacity、Hit candidate
- Encounter Timeline: Phase、Wave、Spawn、Boss transition
- Damage／Team Matrix: Relation、block、apply、credit
- Score Rule Table: Event、Filter、point、combo、multiplier
- Shooter HUD Preview: keyboard／controller／touch、safe area、locale、color／flash
- Shooter Profiler: Fire request、accepted／rejected、Projectile、query、hit、Damage、queue、pool
- Replay Timeline: Input→Fire→Hit→Damage→Defeat→Scoreのcausal edge

PreviewはSource Definitionと同じC++ evaluator、Collision query normalization、RNG、Target Profileを使う。Editor専用の簡易弾道式を正解系にしない。

AI変更はRequirement、Before／After、Gameplay差分、Presentation差分、capacity impact、Testを分けて表示する。「Damage +10」と「muzzle flashを強くする」を同じ変更としてまとめない。

## 14. Capacity、Memory、Performance

### 14.1 Profileが必須宣言するScale

```text
peak_live_weapon_instance
peak_fire_request_per_simulation_advance
peak_shot_activation_per_simulation_advance
peak_projectile_spawn_per_simulation_advance
peak_live_authoritative_projectile
peak_hitscan_query_per_simulation_advance
peak_shot_hit_per_simulation_advance
peak_damage_event_per_simulation_advance
peak_score_event_per_simulation_advance
peak_live_pickup
peak_pickup_collection_per_simulation_advance
peak_enemy_and_ally
peak_simultaneous_presentation_cue
```

値はGame BriefのScale intentと一致させる。Runtime Compilerが合格しやすい値へ丸めない。

### 14.2 C1組込み最低Fixture

| Profile | active combat actor | live projectile | projectile spawn／Simulation Advance | hitscan query／Simulation Advance | 目標 |
|---|---:|---:|---:|---:|---|
| `profile.shooter.top_down_2d` | 256 | 2,048 | 256 | 128 | 1080p60、authoritative drop 0 |
| `profile.shooter.third_person_3d` | 50 | 256 | 64 | 128 | 1080p60、authoritative drop 0 |

この個数はProduct上限ではない。Project intentが上回る場合は[Performance／capacityが所有する`IntegratedScaleFixtureV1`](../04-runtime/performance-capacity.md#13-integrated-fixtureとqualification)をProject固有Envelopeから生成する。

Shooter Profileは上表の規模値とRecipe bindingだけを所有する。Weapon／Projectile／Damage／Pickup／Event queueのreservation、pool、capacity failure、rollback、Diagnosticは[Gameplay Feature Packs §4～8](gameplay-features.md#4-game-systemとstate-owner)と[Performance／Capacity](../04-runtime/performance-capacity.md)が所有し、本書は挙動または失敗型を再定義しない。

### 14.3 Genre integrated-scale fixture

| Destination fixture | Profile scale input | 正式な検証 |
|---|---|---|
| `fixture.genre.shooter.integrated-scale.top-down-2d` | `profile.shooter.top_down_2d`の§14.1／14.2 exact値 | Genre composition、Profile、`scope.genre.shooter.game_flow.instance`、Perception bindingをFeature owner receiptと統合 |
| `fixture.genre.shooter.integrated-scale.third-person-3d` | `profile.shooter.third_person_3d`の§14.1／14.2 exact値 | Genre composition、Profile、`scope.genre.shooter.game_flow.instance`、Perception bindingをFeature owner receiptと統合 |

両fixtureは[Performance／capacityが所有する`IntegratedScaleFixtureV1`](../04-runtime/performance-capacity.md#13-integrated-fixtureとqualification)へ上表の規模値を入力し、Feature failure semanticsを複製しない。宣言値不足、非finite、負値、required axis欠落はGenre-owned `MIRAKAN-GENRE-SHOOTER-PROFILE-SCALE-UNDERSPECIFIED`でProfile applyを拒否し、last-valid Profile／Recipe／fixture receiptを維持する。

### 14.4 Genre-owned diagnostic

| Diagnostic ID | code | Owner | 条件 | 結果 |
|---|---|---|---|---|
| `diagnostic.genre.shooter.profile.scale_underspecified` | `MIRAKAN-GENRE-SHOOTER-PROFILE-SCALE-UNDERSPECIFIED` | `genre.shooter` | §14.1 required axis欠落、非finite／負値、§14.2 Qualification input未解決 | Profile／Recipe applyを拒否しlast-valid Genre receiptを維持 |

このrowは`diagnostic_version=1`とself-excluding `diagnostic_content_hash`を持つexact `DiagnosticCodeRefV1`である。Feature Definition、State owner、Fire transaction、Projectile／query／queue capacity、Damage、Pickup、Score、Feature Save／Replay、Feature presentation authorityのDiagnosticは本表へ追加せず、[Gameplay Feature Packs §7](gameplay-features.md#7-failureとdiagnostic)を参照する。

## 15. Qualification closure

Shooter Genre PackのQualification owner範囲はcomposition recipe、Profile、Game Flow、Action role、`ShooterPerceptionBindingV1`、Shooter統合fixtureに限定する。Feature schema、closed enum、境界値、Fire transaction、canonical ordering、State owner、Save／Replay、failure、Feature contract fixtureの合否は各Feature ownerが発行したQualification Receiptをconsumeし、Shooter側で再所有または再判定しない。

Shooter側は2D top-down／third-person 3Dの両fixtureで、required receipt closure、Perception target selection、lost-target behavior、fire-intent policy、Game Flow、Action role、Profile bindingが接続できることだけを検証する。Packとしてのinstall／apply／update／remove、Project ChangeSet、qualification receiptは[Pack Contract](pack-contract.md)を使い、Capability成熟度と実装順序は[Product Plan](../00-product/product-plan.md)へ委譲する。Profile固有のCamera、Input、UI、Audio、VFX、Asset templateは各ownerのschemaを参照し、同じFieldまたは固定値を本Packへ再定義しない。

## 16. Feature identity migration

旧Genre由来のFeature identityはaliasとして残さず、Feature ownerのmigration stepで次のexact identityへclean renameする。旧Save／Replay／fixture参照はmigration receiptへ記録し、AI catalogは旧名からGenre dependencyを推論しない。

| Legacy source identity | Exact destination identity | Migration owner |
|---|---|---|
| `ShooterProjectileStateV1` | `RangedProjectileStateV1` | `feature.ranged_combat` |
| `SpawnShooterProjectileCommandV1` | `SpawnRangedProjectileCommandV1` | `feature.ranged_combat` |
| `ShooterProjectileSnapshotV1` | `RangedProjectileSnapshotV1` | `feature.ranged_combat` |
| `fixture.shooter.even-floor` | `fixture.feature.ranged_combat.even_floor` | `feature.ranged_combat` |
| `fixture.shooter.explicit-offsets` | `fixture.feature.ranged_combat.explicit_offsets` | `feature.ranged_combat` |
| `MIRAKAN-SHOOTER-DEFINITION_INVALID` | `MIRAKAN-FEATURE-DEFINITION_INVALID` | referenced Feature owner |
| `MIRAKAN-SHOOTER-CONTRACT_VERSION_MISMATCH` | `MIRAKAN-FEATURE-CONTRACT_VERSION_MISMATCH` | referenced Feature owner |
| `MIRAKAN-SHOOTER-CAPABILITY_UNAVAILABLE` | `MIRAKAN-FEATURE-CAPABILITY_UNAVAILABLE` | referenced Feature owner |
| `MIRAKAN-SHOOTER-STATE_OWNER_CONFLICT` | `MIRAKAN-FEATURE-STATE_OWNER_CONFLICT` | referenced Feature owner |
| `MIRAKAN-SHOOTER-FIRE_TRANSACTION_FAILED` | `MIRAKAN-RANGED-COMBAT-FIRE_TRANSACTION_FAILED` | `feature.ranged_combat` |
| `MIRAKAN-SHOOTER-PROJECTILE_CAPACITY_EXCEEDED` | `MIRAKAN-RANGED-COMBAT-PROJECTILE_CAPACITY_EXCEEDED` | `feature.ranged_combat` |
| `MIRAKAN-SHOOTER-QUERY_CAPACITY_EXCEEDED` | `MIRAKAN-RANGED-COMBAT-QUERY_CAPACITY_EXCEEDED` | `feature.ranged_combat` |
| `MIRAKAN-SHOOTER-AUTHORITATIVE_QUEUE_OVERFLOW` | `MIRAKAN-FEATURE-AUTHORITATIVE_QUEUE_OVERFLOW` | referenced Feature owner |
| `MIRAKAN-SHOOTER-DAMAGE_TARGET_INVALID` | `MIRAKAN-COMBAT-DAMAGE_TARGET_INVALID` | `feature.combat` |
| `MIRAKAN-SHOOTER-PICKUP_GRANT_FAILED` | `MIRAKAN-PICKUP-GRANT-FAILED` | `feature.pickup_grant` |
| `MIRAKAN-SHOOTER-SCORE_OVERFLOW` | `MIRAKAN-SCORING-OVERFLOW` | `feature.scoring` |
| `MIRAKAN-SHOOTER-SAVE_CONTRACT_MISMATCH` | `MIRAKAN-FEATURE-SAVE_CONTRACT_MISMATCH` | referenced Feature owner |
| `MIRAKAN-SHOOTER-REPLAY_DIVERGENCE` | `MIRAKAN-FEATURE-REPLAY_DIVERGENCE` | referenced Feature owner |
| `MIRAKAN-SHOOTER-PRESENTATION_AUTHORITY_VIOLATION` | `MIRAKAN-FEATURE-PRESENTATION_AUTHORITY_VIOLATION` | referenced Feature owner |
| `MIRAKAN-SHOOTER-PROFILE_SCALE_UNDERSPECIFIED` | `MIRAKAN-GENRE-SHOOTER-PROFILE-SCALE-UNDERSPECIFIED` | `genre.shooter` |
| `fixture.shooter.crowded-battle-2d` | `fixture.genre.shooter.integrated-scale.top-down-2d` | `genre.shooter` |
| `fixture.shooter.crowded-battle-3d` | `fixture.genre.shooter.integrated-scale.third-person-3d` | `genre.shooter` |

上記source identityはこのmigration表にexactly one rowだけ存在し、alias／wildcard／prefix fallbackで照合しない。migration stepはsource identityとdestination identityの双方を完全一致で記録し、unknown legacy identityを最も近いdestinationへ推測変換しない。
