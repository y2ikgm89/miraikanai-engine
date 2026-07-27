# Physics AI Catalog Proposal

- 文書ID: mirakan.appendix.physics-ai-catalog-proposal
- 文書種別: proposal appendix
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 親Owner: [Physics](../05-simulation/physics.md)
- 正本範囲: Physics intent、role、Operation、diagnostic、AI Evalの候補Catalog
- 非正本範囲: 親Ownerが所有する安定Architecture原則、実装Task、実装順序、生成済みArtifactまたはQualification結果
- 規範依存: [親Owner](../05-simulation/physics.md)
- 関連文書: [Architecture Governance](../01-governance/architecture-governance.md)
- 根拠区分: project-decision／provisional。実ArtifactがないRegistry、Catalog、Fixtureは候補
- 外部根拠確認日: 2026-07-27

> この補助文書の型、Registry、Catalog、Fixtureは、対応するRepository Artifactが存在しない限り未実装の設計候補である。親Ownerの安定原則や実装済み状態を上書きしない。
## 5. AI semantics

Physics AI surfaceは自然言語を直接Body fieldへ投影せず、`intent -> discovery -> questions／assumptions -> semantic resolution -> planned semantic action -> preview -> validation`を一つのbounded pipelineとして扱う。RuntimeやVendor APIはAI surfaceではない。

### 5.1 Intentとclosed vocabulary

`PhysicsIntentVocabularyEntryV1`は自然言語の単語をMCD Operation identityへ直接bindする辞書ではなく、Game文脈から意味候補を絞るversion付きCatalog entryである。mandatory fieldを次へ固定する。

| Field | 型／規則 |
|---|---|
| `semantic_tag` | closed ID |
| `localized_terms` | locale別の代表語。命令や権限を含めない |
| `positive_examples` | Tagに一致する短いGameplay文 |
| `negative_examples` | 表面語が似ても一致しない文 |
| `candidate_physics_role_refs` | `PhysicsIntentRoleRefV1[]`。owner／version／hash付き |
| `question_triggers` | 意味が分岐する条件 |
| `candidate_capability_ids` | discovery候補。利用可否はManifestで再検査 |
| `forbidden_mappings` | 自動変換してはならないrole／operation |
| `rationale_refs` | Architecture requirement／section参照 |

`localized_terms`の文字列一致だけでResolutionを確定しない。Vocabulary entryはBackend名、exact dependency version、native setting、thread countを含めず、未登録文字列を新enumとして保存しない。

Physics Coreはobject／Genre名のclosed enumを所有せず、次のrole registryだけを所有する。

```text
PhysicsIntentRoleRefV1
  role_id
  role_version: uint32
  role_content_hash: SHA-256

PhysicsIntentRoleRecordV1
  role_id
  role_version: uint32
  owner_ref: exact {owner_id, owner_revision, owner_content_hash}
  status: active | reserved_unsupported | removed
  allowed_motion_authorities[1..5]
  allowed_collision_semantics[1..4]
  allowed_hit_authorities[1..6]
  allowed_shape_strategies[1..7]
  allowed_speed_policies[1..4]
  required_capability_refs[0..32]
  role_content_hash: SHA-256

PhysicsIntentRoleQualificationReceiptRefV1
  qualification_id
  qualification_version: positive uint32
  qualification_subject_hash: SHA-256
  signed_record_hash: SHA-256

PhysicsIntentRoleQualificationSubjectV1
  qualification_id
  qualification_version: positive uint32
  owner_ref: exact {owner_id, owner_revision, owner_content_hash}
  role_ref: PhysicsIntentRoleRefV1
  target_profile_refs[1..16]: exact version/hash-bound Target refs
  fixture_refs[1..64]: exact {fixture_id, fixture_version, fixture_content_hash}
  input_closure_hash
  result: pass | fail
  qualification_subject_hash: SHA-256

PhysicsIntentRoleQualificationReceiptV1
  subject: PhysicsIntentRoleQualificationSubjectV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=physics_intent_role_qualification)

PhysicsIntentRoleActivationBindingRefV1
  activation_binding_id
  activation_binding_version: positive uint32
  activation_binding_hash: SHA-256

PhysicsIntentRoleActivationBindingV1
  activation_binding_id
  activation_binding_version: positive uint32
  role_ref: PhysicsIntentRoleRefV1
  qualification_receipt_refs[1..64]:
    PhysicsIntentRoleQualificationReceiptRefV1
  activation_binding_hash: SHA-256

PhysicsIntentRoleRegistryRefV1
  registry_id
  registry_version
  registry_content_hash

PhysicsIntentRoleRegistryV1
  registry_id: physics.intent_role.registry.active
  registry_version
  registry_content_hash
  records[1..4096]: PhysicsIntentRoleRecordV1

PhysicsIntentRoleSelectionDocumentV1
  common Project Document header
  selection_id: same StableId as Document header
  subject_definition_ref/hash
  selected_role_ref: PhysicsIntentRoleRefV1
  selected_role_activation_binding_ref:
    PhysicsIntentRoleActivationBindingRefV1
  selected_axis_closure_hash
  selection_content_hash
```

Core初期roleは次のbehavior-neutralなexact五Receipt-free base recordだけである。全Refはversion 1、全ownerは`owner.core.physics`のexact revision／content hash、全rowの`status=active`、Capabilityはversion／Contract set root付きで保存する。Physicsが所有しないpresentation-only objectはRoleへ含めない。

| role ID | status | allowed motion | allowed collision | allowed hit | allowed shape | allowed speed | required Capability |
|---|---|---|---|---|---|---|---|
| `role.physics.static_environment` | `active` | `static` | `solid_block; query_only` | `solver_contact; swept_shape_query; overlap_query; none` | `primitive; compound_primitive; convex; static_triangle_mesh; heightfield; tile_chain_2d` | `discrete` | `capability.simulation.collision_response@1` |
| `role.physics.dynamic_body` | `active` | `dynamic_solver` | `solid_block; sensor_notify` | `solver_contact; sensor_event; none` | `primitive; compound_primitive; convex` | `discrete; continuous_body` | `capability.simulation.physics_dynamics@1; capability.simulation.collision_response@1` |
| `role.physics.kinematic_body` | `active` | `kinematic_target` | `solid_block; sensor_notify` | `solver_contact; sensor_event; swept_shape_query; none` | `primitive; compound_primitive; convex` | `discrete; authoritative_sweep; teleport` | `capability.simulation.collision_response@1` |
| `role.physics.sensor` | `active` | `static; kinematic_target` | `sensor_notify` | `sensor_event; overlap_query; none` | `primitive; compound_primitive; convex; tile_chain_2d` | `discrete; teleport` | `capability.simulation.collision_response@1` |
| `role.physics.query_subject` | `active` | `query_driven` | `query_only` | `swept_shape_query; overlap_query; gameplay_rule; none` | `primitive; compound_primitive; convex; tile_chain_2d` | `discrete; authoritative_sweep; teleport` | `capability.simulation.collision_query@1` |

Registry／RoleRef確定後、次のroot外Activation bindingを各Roleへexact一件作る。各receipt refは同じrowのRoleRefをsubjectにする`PhysicsIntentRoleQualificationSubjectV1`のcanonical signed wrapperへ解決する。

| RoleRef | exact Qualification Receipt | exact Activation Binding |
|---|---|---|
| `role.physics.static_environment@1` | `qualification.physics.intent-role.static-environment@1` | `activation.physics.intent-role.static-environment@1` |
| `role.physics.dynamic_body@1` | `qualification.physics.intent-role.dynamic-body@1` | `activation.physics.intent-role.dynamic-body@1` |
| `role.physics.kinematic_body@1` | `qualification.physics.intent-role.kinematic-body@1` | `activation.physics.intent-role.kinematic-body@1` |
| `role.physics.sensor@1` | `qualification.physics.intent-role.sensor@1` | `activation.physics.intent-role.sensor@1` |
| `role.physics.query_subject@1` | `qualification.physics.intent-role.query-subject@1` | `activation.physics.intent-role.query-subject@1` |

recordsはrole ID／version順へstrict sortし、exact duplicate、同一ID／versionの別content hash、同じrole IDのactive record複数、owner namespace偽装、非canonical orderを拒否する。`role_content_hash`はASCII `MIRAKAN_PHYSICS_INTENT_ROLE_RECORD_V1`と、当該hash Fieldだけを除くReceipt-free Record canonical MCD bytesを`uint32_be` length framingしてSHA-256する。Recordはhashを含まないidentity Fieldをpreimageに持ち、外部`PhysicsIntentRoleRefV1`、Qualification Receipt、Activation Bindingをrecord内へ埋め戻さない。Registry hashはASCII `MIRAKAN_PHYSICS_INTENT_ROLE_REGISTRY_V1`、Registry ID／version、record count、全record canonical bytesを各`uint32_be` length framingしてSHA-256し、`registry_content_hash`自身を除外する。`PhysicsIntentRoleRegistryRefV1`は三Fieldすべてを同一active Registryへexact解決し、ID-only、latest version、hash fallbackを許可しない。Registry／RoleRef確定後、Qualification subject hashをASCII `MIRAKAN_PHYSICS_INTENT_ROLE_QUALIFICATION_SUBJECT_V1`、Activation binding hashをASCII `MIRAKAN_PHYSICS_INTENT_ROLE_ACTIVATION_BINDING_V1`と各自己Fieldを除くcanonical bytesから計算する。subjectの`owner_ref`は`role_ref`が解決するReceipt-free Role recordのownerとbyte equality、Activation BindingのRoleRefはsubjectとbyte equalityでなければならず、Receipt refのqualification ID／versionはwrapper subjectと一致する。signed wrapperは完成subjectだけを署名する。生成順は`receipt-free Role → Registry／RoleRef → Qualification subject → signed Receipt → Activation Binding → Selection`である。Production Selection／Compile／Save／ReplayはActivation Bindingが指す署名済みReceiptのsubject／result／freshnessだけを検証し、Fixture bodyを解決しない。Fixture集合は別`PhysicsIntentRoleQualificationSubjectV1`だけが所有する。owner、RoleRef、subject hash、signed hash、Activation Binding RoleRefの各一Fieldを別baseへ差し替えるnegative fixtureを持つ。

Pack／Projectはowner namespace、exact Capability、axis compatibilityを持つReceipt-free Role recordを下向きに登録できる。Registry／RoleRef固定後、同じownerとRoleRefを持つroot外`PhysicsIntentRoleQualificationSubjectV1`だけがobject vocabulary、具体例、default mapping、Fixtureを所有し、canonical signed ReceiptとActivation Bindingを順に生成する。Receipt／BindingをRole recordまたはRegistry hashへ戻さず、Core resolver、Core vocabulary、Core fixture inventoryへcontributor fixtureをコピーしない。role refは候補検索を助ける分類であり、motion／collision／hit／shape／speed各axisの検証を省略または上書きしない。

Project Sourceの選択正本は`PhysicsIntentRoleSelectionDocumentV1`であり、RegistryとResolutionは派生である。ただし選択write surfaceは本Taskで完全Operation登録されていないため、`operation.physics.intent_role.select`は[Executable contracts](../02-foundation/executable-contracts.md#211-既存domain文書から回収した未登録operation候補)の`planning.operation_family.physics_role_selection@1`に属するexact一候補であり、current MCD、Owner Manifest、Trusted Service allowlist、Policy、Validator、Diagnostic、Receipt、Provider／MCP Catalog、generated alias、legacy aliasの各集合をすべて`[]`とする。Capability stateは`not_activated`で、Editor／AIの選択要求は`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`を返してSource不変にする。将来activation work item `activation.physics.intent_role_selection.v1`はcreate/upsertとupdateを別named inputで定義し、MCD全Field、Service allowlist、risk、side effect、idempotency、transaction、pure pre／post policy、closed Diagnostic、Validator closure、rate／timeout、canonical selection set hash／sort／duplicate rule、signed Receipt、private-to-public crash recovery、positive／negative Qualificationを同じContract set transactionで完備する。name-only `select`を先行公開しない。Compile closure、Save、Replayは既存Selection Document ref／hash、RegistryRef、RoleRef、Role Activation Binding ref、axis closure hashを保存し、reload時にSource→Registry→record→Capability→Qualification subject→signed Receipt→Activation Bindingを再検査する。

`PhysicsIntentResolutionV1`のmandatory schemaは次である。fieldの省略、任意propertyの追加、closed value以外の文字列を拒否する。

```text
PhysicsSourceRequestRefV1
  task_id: StableId
  content_id: StableId
  content_revision: positive uint64
  content_hash: SHA-256
  access_policy_ref: McdContractRefV1(kind=policy)

PhysicsCostEstimateRefV1
  estimate_id: StableId
  estimate_version: positive uint32
  estimate_content_hash: SHA-256

PhysicsIntentResolutionV1
  resolution_id: StableId
  resolution_version: positive uint32
  source_request_ref: PhysicsSourceRequestRefV1
  source_request_hash: SHA-256
  contract_set_hash: SHA-256
  project_ref:
    exact {project_id, project_revision, document_set_hash}
  target_profile_refs[1..16]:
    exact {target_profile_id, target_profile_version,
           target_profile_content_hash}
  physics_role_registry_ref: PhysicsIntentRoleRegistryRefV1
  world_space_profile_ref: exact WorldSpaceProfileRefV1
  physics_role_ref: PhysicsIntentRoleRefV1
  motion_authority: PhysicsMotionAuthorityV1
  collision_semantics: PhysicsCollisionSemanticsV1
  hit_authority: PhysicsHitAuthorityV1
  shape_strategy: PhysicsShapeStrategyV1
  speed_policy: PhysicsSpeedPolicyV1
  selected_capability_refs[0..32]: McdContractRefV1(kind=capability)
  selected_operation_refs[0..32]: McdContractRefV1(kind=operation)
  blocking_question_refs[0..32]:
    exact {question_id, question_version, question_content_hash}
  assumptions[0..32]:
    exact {assumption_id, assumption_version, assumption_content_hash}
  rejected_alternatives[0..32]:
    exact {alternative_id, alternative_version, alternative_content_hash}
  diagnostic_refs[0..64]: DiagnosticCodeRefV1
  preview_qualification_receipt_refs[0..64]:
    PhysicsIntentRoleQualificationReceiptRefV1
  cost_estimate_ref: PhysicsCostEstimateRefV1
    | canonical omission
  disposition: ready_to_propose | question_required | capability_unavailable | rejected
  resolution_content_hash: SHA-256
```

`source_request_ref`はAuthoring Task内のaccess-controlled contentを参照し、raw PromptをCatalog、MCD、Receiptへ複製しない。全MCD／Registry／Target／Question／Assumption／Alternative／Diagnostic／Qualification／Cost refは上記named typeのversion／hashを必須にし、bare `ContentRef`、`RevisionId`、`CapabilityId`、`OperationId`、`DiagnosticId`、`FixtureId`をcurrent schemaで受理しない。arrayはlogical ID／version／hash順にstrict sortし、duplicate／same-ID different-hashを拒否する。`physics_role_registry_ref`は候補生成時に読んだactive Registryを固定し、validate／preview／proposal時にcurrent Registry refと三Fieldexact equalityで再検査する。`resolution_content_hash`はASCII `MIRAKAN_PHYSICS_INTENT_RESOLUTION_V1`と自己Fieldを除くlength-framed canonical bytesから計算する。

ResolutionはAuthoring Taskのephemeral derived recordであり、Project Source、Save、Replay、Runtime Packageへ永続化しない。永続化するのは承認済みSelection Documentとそのexact Registry／Role／axis／Project closureだけである。Task終了またはsource／Project／Contract／Registry driftでResolutionをinvalidにし、再計算する。`ready_to_propose`は既存の完全登録OperationでChangeSetを提案できる意味だけを持ち、Commit authorizationを意味しない。本Taskでは選択Operationが`not_activated`のためRole selection要求を`ready_to_propose`にせず`capability_unavailable`へ固定する。`question_required`はblocking ambiguityが残る結果、`capability_unavailable`は要求を満たすactive Capability／Operationがない結果、`rejected`はinvalid／forbiddenな要求である。

role以外のclosed semantic axisを次へ固定する。一つのResolutionは各軸から一つだけを選び、role recordのallowed setと照合する。

| Type | Closed values |
|---|---|
| `PhysicsMotionAuthorityV1` | `static \| kinematic_target \| dynamic_solver \| query_driven \| presentation_only` |
| `PhysicsCollisionSemanticsV1` | `solid_block \| sensor_notify \| query_only \| none` |
| `PhysicsHitAuthorityV1` | `solver_contact \| sensor_event \| swept_shape_query \| overlap_query \| gameplay_rule \| none` |
| `PhysicsShapeStrategyV1` | `primitive \| compound_primitive \| convex \| static_triangle_mesh \| heightfield \| tile_chain_2d \| none` |
| `PhysicsSpeedPolicyV1` | `discrete \| continuous_body \| authoritative_sweep \| teleport` |

`GameplayPhysicsRoleV1` enumのcanonical bytesとそれを使用する旧Project／Documentが実在することは現計画からは証明されておらず、current Source、Editor、AI projection、Compile、deserializerへ登録しない。`operation.physics.intent_role.migrate@1`は[Executable Contracts §8.1.2](../02-foundation/executable-contracts.md#812-conditional-legacy-migration-evidence-gate)のconditional legacy migrationで、current状態は`not_activated`である。この移行に固有のcurrent MCD／Owner Manifest／Service allowlist／Policy／Validator／migration Diagnostic／Operation Receipt／Provider／MCP／alias、migration Qualification subject／Receipt／Binding、Activation Catalog／projection subset、Contribution record／Registry、Migration Manifest集合はすべてexact `[]`である。`service.offline_project_migrator`、`capability.authoring.offline_migration`、`profile.isolation.offline_project_migrator`のcurrent集合もexact `[]`で、移行要求をdispatchしない。

将来Activationするには、実在する旧enum schema／Project／Document bytes、旧MCD local record／common envelope／payload、source `ContractSetSnapshotV2`、Owner Identity Registry、Named Algorithm Registry、`FoundationDefinitionClosureV1`、全retained artifact ref／hash、正負fixtureを推測なしで列挙したsigned `LegacyMigrationInventoryV1`が§8.1.2 gateを満たさなければならない。その同じ承認済みContract set transactionだけが、Operation／Type／Policy／Validator／migration Diagnostic／Receipt、offline Service／Capability／Isolation Profile、Service allowlist、Contribution Registry／records、Qualification Receipt／Binding、Manifest、Provider／MCP projectionを完全closureとして同時にmaterializeできる。owner固有変換をCore switch文へ追加しない。以下のschemaとrecord値はpost-activation destination templateであり、block内の`status=active`、Service ref、Policy ref、Provider exposureをcurrent refまたはcurrent product surfaceとして解釈しない。

```text
PhysicsIntentRoleMigrationContributionRefV1
  contribution_id
  contribution_version
  contribution_content_hash

PhysicsIntentRoleMigrationContributionRecordV1
  contribution_id
  contribution_version
  owner_ref/hash
  retained_source_mcd_ref: McdContractRefV1(
    kind=type, id=type.physics.legacy_gameplay_role,
    version=1, source_contract_set_hash)
  accepted_legacy_values[1..64]
  destination_role_refs[1..64]: PhysicsIntentRoleRefV1
  mapping_policy_ref: McdContractRefV1(kind=policy)
  axis_mapping_policy_ref: McdContractRefV1(kind=policy)
  contribution_content_hash

PhysicsIntentRoleMigrationContributionRegistryRefV1
  registry_id
  registry_version
  registry_content_hash

PhysicsIntentRoleMigrationContributionRegistryV1
  registry_id: physics.intent_role.migration_contribution.registry.active
  registry_version
  records[1..4096]
  registry_content_hash

PhysicsIntentRoleMigrationQualificationSubjectV1
  qualification_id
  qualification_version: positive uint32
  contribution_ref: PhysicsIntentRoleMigrationContributionRefV1
  owner_ref: exact contribution owner ref/hash
  target_profile_refs[1..64]
  fixture_refs[1..64]:
    exact {fixture_id, fixture_version, fixture_content_hash}
  input_closure_hash
  result: pass | fail
  qualification_subject_hash: SHA-256

PhysicsIntentRoleMigrationQualificationReceiptV1
  subject: PhysicsIntentRoleMigrationQualificationSubjectV1
  signed_record:
    exact MirakanSignedRecordV1(
      purpose=physics_intent_role_migration_qualification)

PhysicsIntentRoleMigrationQualificationReceiptRefV1
  qualification_id
  qualification_version: positive uint32
  qualification_subject_hash
  signed_record_hash

PhysicsIntentRoleMigrationQualificationBindingRefV1
  qualification_binding_id
  qualification_binding_version: positive uint32
  qualification_binding_hash

PhysicsIntentRoleMigrationQualificationBindingV1
  qualification_binding_id
  qualification_binding_version: positive uint32
  contribution_ref: PhysicsIntentRoleMigrationContributionRefV1
  qualification_receipt_ref:
    PhysicsIntentRoleMigrationQualificationReceiptRefV1
  qualification_binding_hash

PhysicsIntentRoleMigrationManifestV1
  manifest_id: physics.intent_role.migration.v1_to_registry
  manifest_version: 1
  operation_ref: McdContractRefV1(
    kind=operation, id=operation.physics.intent_role.migrate,
    version=1, contract_set_hash)
  input_type_ref: McdContractRefV1(
    kind=type, id=type.physics.intent_role_migration_input,
    version=1, contract_set_hash)
  output_type_ref: McdContractRefV1(
    kind=type, id=type.physics.intent_role_migration_result,
    version=1, contract_set_hash)
  receipt_type_ref: McdContractRefV1(
    kind=type, id=type.physics.intent_role_migration_receipt,
    version=1, contract_set_hash)
  precondition_policy_ref: McdContractRefV1(
    kind=policy, id=policy.operation.physics.intent_role.migrate.precondition,
    version=1, contract_set_hash)
  postcondition_policy_ref: McdContractRefV1(
    kind=policy, id=policy.operation.physics.intent_role.migrate.postcondition,
    version=1, contract_set_hash)
  rate_limit_policy_ref: McdContractRefV1(
    kind=policy, id=policy.authoring.physics_intent_role_migration.rate_limit,
    version=1, contract_set_hash)
  validator_closure_ref: OperationValidatorClosureRefV1
  contribution_registry_ref: PhysicsIntentRoleMigrationContributionRegistryRefV1
  trusted_service_ref: TrustedServiceRefV1(
    service_id=service.offline_project_migrator, service_version=1,
    service_content_hash)
  trusted_service_allowlist_operation_local_refs[1]:
    ContractSetLocalRefV1(
      kind=operation, id=operation.physics.intent_role.migrate, version=1)
  diagnostic_refs[15]: DiagnosticCodeRefV1
  qualification_binding_refs[1..64]:
    exact PhysicsIntentRoleMigrationQualificationBindingRefV1
  manifest_hash: SHA-256
```

Activation後のdestination Core contribution候補は次の完全三Receipt-free recordだけを持つ。全rowは`contribution_version=1`、`owner_ref={owner.core.physics,destination owner revision,content hash}`、source schema=`{type.physics.legacy_gameplay_role,1,source Contract set hash}`を持ち、表のPolicy／Role refはexact version／hashを保存する。表の三rowはcurrent Registry memberではなく、現時点のcurrent contribution集合はexact `[]`である。

| contribution ID | accepted legacy values | destination role refs | mapping policy | axis mapping policy |
|---|---|---|---|---|
| `physics.intent_role.migration_contribution.core.world_static` | `[world_static]` | `[role.physics.static_environment@1]` | `policy.physics.intent_role.migration.world_static@1` | `policy.physics.intent_role.axis.world_static@1` |
| `physics.intent_role.migration_contribution.core.movable_prop` | `[movable_prop]` | `[role.physics.dynamic_body@1]` | `policy.physics.intent_role.migration.movable_prop@1` | `policy.physics.intent_role.axis.movable_prop@1` |
| `physics.intent_role.migration_contribution.core.sensor_volume` | `[sensor_volume]` | `[role.physics.sensor@1]` | `policy.physics.intent_role.migration.sensor_volume@1` | `policy.physics.intent_role.axis.sensor_volume@1` |

Activation後、object固有legacy値はsigned Inventoryに束縛され、当該Pack／Project contributionがexact一件存在する場合だけ移行する。recordはContribution ID／version順、accepted valueはUTF-8 byte順、destination refはrole ID／version順へstrict sortし、duplicate valueの複数active contribution、stale owner／policy／roleをRegistry全体で拒否する。record hashはASCII `MIRAKAN_PHYSICS_ROLE_MIGRATION_CONTRIBUTION_V1`、Registry hashはASCII `MIRAKAN_PHYSICS_ROLE_MIGRATION_CONTRIBUTION_REGISTRY_V1`のself-excluding Receipt-free length-framed canonical bytesである。生成順は`receipt-free Contribution → Registry／ContributionRef → Migration Qualification subject → signed Receipt → Qualification Binding → Migration Manifest`であり、subject ownerはbase owner、Binding ContributionRefはsubjectとbyte equalityにする。Manifest binding集合と選択Contribution集合はexact set equalityで、Receipt／BindingをContribution／Registry hashへ戻さない。Production migration recordはFixture bodyを解決せず、Bindingが指すsigned Receiptのsubject／result=`pass`／freshnessだけを検証する。三Core rowは同logical suffixの`qualification.physics.intent-role-migration.*@1`とBindingをexact一件ずつ持つ。

```text
operation.physics.intent_role.migrate@1
  MCD common envelope:
    mcd_version=1; kind=operation;
    id=operation.physics.intent_role.migrate;
    version=1; status=active;
    title=Migrate Physics Intent Role;
    description=Atomically migrate one legacy Physics object role through
      an exact owner contribution into a typed role Selection Document;
    owners=[owner.core.physics]; requirement_refs=[];
    rationale_refs=[mirakan.arch.simulation-physics#5-ai-semantics];
    since_contract_set=2; supersedes=[]; tags=[authoring,migration,physics]
  operation_kind: command
  input_type: McdContractRefV1(
    kind=type, id=type.physics.intent_role_migration_input,
    version=1, contract_set_hash)
  output_type: McdContractRefV1(
    kind=type, id=type.physics.intent_role_migration_result,
    version=1, contract_set_hash)
  authority: TrustedServiceRefV1(
    service_id=service.offline_project_migrator, service_version=1,
    service_content_hash)
  risk_class: R3
  side_effects: [authoring]
  idempotency: idempotent_with_key
  transaction: authoring_changeset
  preconditions:
    [McdContractRefV1(
      kind=policy, id=policy.operation.physics.intent_role.migrate.precondition,
      version=1, contract_set_hash)]
  postconditions:
    [McdContractRefV1(
      kind=policy, id=policy.operation.physics.intent_role.migrate.postcondition,
      version=1, contract_set_hash)]
  validator_closure_ref:
    {closure_id=validator_closure.operation.physics.intent_role.migrate,
     closure_version=1, closure_content_hash}
  timeout_ms: 120000
  rate_limit_policy: McdContractRefV1(
    kind=policy, id=policy.authoring.physics_intent_role_migration.rate_limit,
    version=1, contract_set_hash)
  audit_level: full_redacted
  provider_exposure: mcp_proposal
  receipt_type: McdContractRefV1(
    kind=type, id=type.physics.intent_role_migration_receipt,
    version=1, contract_set_hash)
  errors[15]: exact DiagnosticCodeRefV1 records for
    diagnostic.conflict.revision_mismatch
    diagnostic.authorization.denied
    diagnostic.approval.required
    diagnostic.authoring.lock_conflict
    diagnostic.mcd.operation_predicate_invalid
    diagnostic.operation.timeout
    diagnostic.operation.rate_limit_exceeded
    diagnostic.operation.idempotency_key_reuse
    diagnostic.physics.intent_role.source_invalid
    diagnostic.physics.intent_role.registry_invalid
    diagnostic.physics.intent_role.contribution_missing
    diagnostic.physics.intent_role.contribution_ambiguous
    diagnostic.physics.intent_role.capability_unavailable
    diagnostic.physics.intent_role.axis_mapping_invalid
    diagnostic.physics.intent_role.receipt_binding_mismatch

PhysicsIntentRoleMigrationInputV1
  input_type_ref: McdContractRefV1(
    kind=type, id=type.physics.intent_role_migration_input,
    version=1, contract_set_hash)
  operation_ref: McdContractRefV1(
    kind=operation, id=operation.physics.intent_role.migrate,
    version=1, contract_set_hash)
  before_project_ref:
    exact {project_id, expected_project_revision, document_set_hash}
  operation_intent_hash
  request_hash
  idempotency_key
  source_document_ref/hash
  source_foundation_definition_closure_ref:
    FoundationDefinitionClosureRefV1
  retained_source_mcd_ref: McdContractRefV1(
    kind=type, id=type.physics.legacy_gameplay_role,
    version=1, source_contract_set_hash)
  source_legacy_value
  source_axis_closure_hash
  destination_role_registry_ref
  contribution_registry_ref
  selected_contribution_ref
  preview_policy_ref: McdContractRefV1(kind=policy)
  validation_policy_ref: McdContractRefV1(kind=policy)
  mutation_authorization_binding: exact MutationAuthorizationBindingV2(
    risk_class=R3, authority_evidence=approval)

PhysicsIntentRoleMigrationResultV1
  disposition: migrated | capability_unavailable | ambiguous | rejected
  migrated:
    after_project_ref
    selection_document_ref/hash
    selected_role_ref
    axis_closure_hash
    migration_receipt_ref/hash
    public_publication_marker_ref/hash
  non-migrated:
    diagnostics[1..15]

PreparedPhysicsIntentRoleMigrationReceiptPayloadV1
  publication_binding: exact PreparedReceiptPublicationBindingV1
  operation_ref
  operation_intent_hash
  request_hash
  idempotency_key
  source_document_ref/hash
  source_foundation_definition_closure_ref:
    FoundationDefinitionClosureRefV1
  retained_source_mcd_ref: McdContractRefV1(
    kind=type, id=type.physics.legacy_gameplay_role,
    version=1, source_contract_set_hash)
  source_legacy_value
  role_registry_ref
  contribution_registry_ref
  selected_contribution_ref
  before_project_ref
  after_project_ref
  selection_document_ref/hash
  selected_role_ref
  axis_closure_hash
  preview_receipt_payload_ref/hash
  validation_receipt_payload_ref/hash
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1
  diagnostics[0..15]
  prepared_payload_hash

PhysicsIntentRoleMigrationReceiptV1
  published_receipt:
    exact PublishedDomainReceiptV2 whose
    prepared_domain_receipt_payload_ref/hash resolves
    PreparedPhysicsIntentRoleMigrationReceiptPayloadV1
```

Activation後にOperationが参照する三Policyのdestination値は次の完全なactive MCD recordである。表の`status=active`はatomic activation完了後の値だけを表し、current Policy集合はexact `[]`である。表の二列を連結した値が各record全体であり、暗黙既定値、bare ID、説明文からFieldを補完しない。

| Policy MCD共通Envelope exact value | Policy payload exact value |
|---|---|
| `mcd_version=1; kind=policy; id=policy.operation.physics.intent_role.migrate.precondition; version=1; status=active; title=Physics Intent Role Migration Precondition; description=Validate the exact legacy source, Project base, role and contribution registries, selected contribution, axes, authorization, and idempotency snapshot without mutation; owners=[owner.core.physics]; requirement_refs=[]; rationale_refs=[mirakan.arch.simulation-physics#5-ai-semantics]; since_contract_set=2; supersedes=[]; tags=[operation_predicate,physics,pure]` | `evaluation_mode=pure; side_effects=[]; input_type={id=type.operation.precondition_evaluation_input,version=1,contract_set_hash}; result_type={id=type.operation.predicate_result,version=1,contract_set_hash}` |
| `mcd_version=1; kind=policy; id=policy.operation.physics.intent_role.migrate.postcondition; version=1; status=active; title=Physics Intent Role Migration Postcondition; description=Validate the unpublished Selection Document, exact role and axis closure, prepared Receipt payload, and atomic Project revision increment; owners=[owner.core.physics]; requirement_refs=[]; rationale_refs=[mirakan.arch.simulation-physics#5-ai-semantics]; since_contract_set=2; supersedes=[]; tags=[operation_predicate,physics,pure]` | `evaluation_mode=pure; side_effects=[]; input_type={id=type.operation.postcondition_evaluation_input,version=2,contract_set_hash}; result_type={id=type.operation.predicate_result,version=1,contract_set_hash}` |
| `mcd_version=1; kind=policy; id=policy.authoring.physics_intent_role_migration.rate_limit; version=1; status=active; title=Physics Intent Role Migration Rate Limit; description=Bound migration requests per Project without changing role resolution semantics; owners=[owner.core.physics]; requirement_refs=[]; rationale_refs=[mirakan.arch.simulation-physics#5-ai-semantics]; since_contract_set=2; supersedes=[]; tags=[authoring,physics,rate_limit]` | `policy_ref={id=policy.authoring.physics_intent_role_migration.rate_limit,version=1,contract_set_hash}; scope=project; window_ns=60000000000; max_requests=4; burst=1; exceeded_error_ref={diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash}` |

Activation destination Contract set内部では三Policyを`ContractSetLocalRefV1(kind=policy)`へ投影し、self refはlocal identityだけにする。Manifest三Policy ref、Operationのpre／post／rate ref、Physics ownerのPolicy local subsetはexact三件でset equalityである。三recordの共通Envelopeまたはpayloadの実在Fieldを一つだけ変えるfixtureはPolicy member hashとset rootを変更し、旧Manifest／Operation external refを解決不能にする。

Activation後のmigration固有Diagnostic destination値は次の完全な`DiagnosticLocalRecordV2`である。current migration Diagnostic集合はexact `[]`である。全rowは`diagnostic_version=1`、`owner_local_ref={owner_id=owner.core.physics,owner_revision=1,owner_content_hash=SHA-256(MIRAKAN_DIAGNOSTIC_OWNER_LOCAL_IDENTITY_V1, length-framed canonical owner ID／revision)}`、`requirement_local_refs=[]`、`message_key="<diagnostic_id>.message"`、Ownerを含むself-excluding `diagnostic_local_content_hash`を持つ。root確定後だけ同じ三Field Owner ref、`requirement_refs=[]`、別のself-excluding `diagnostic_content_hash`を持つ外部Registry recordへ投影する。共通八件はActivation先Foundationの共通recordを参照し、本書のcurrent Diagnosticへ複写しない。

| Diagnostic ID | code | severity／category／retryability |
|---|---|---|
| `diagnostic.physics.intent_role.source_invalid` | `MIRAKAN-PHYSICS-INTENT-ROLE-SOURCE-INVALID` | blocking／schema／after_input |
| `diagnostic.physics.intent_role.registry_invalid` | `MIRAKAN-PHYSICS-INTENT-ROLE-REGISTRY-INVALID` | blocking／schema／after_change |
| `diagnostic.physics.intent_role.contribution_missing` | `MIRAKAN-PHYSICS-INTENT-ROLE-CONTRIBUTION-MISSING` | blocking／semantic／after_change |
| `diagnostic.physics.intent_role.contribution_ambiguous` | `MIRAKAN-PHYSICS-INTENT-ROLE-CONTRIBUTION-AMBIGUOUS` | blocking／semantic／after_input |
| `diagnostic.physics.intent_role.capability_unavailable` | `MIRAKAN-PHYSICS-INTENT-ROLE-CAPABILITY-UNAVAILABLE` | blocking／semantic／after_change |
| `diagnostic.physics.intent_role.axis_mapping_invalid` | `MIRAKAN-PHYSICS-INTENT-ROLE-AXIS-MAPPING-INVALID` | blocking／semantic／after_input |
| `diagnostic.physics.intent_role.receipt_binding_mismatch` | `MIRAKAN-PHYSICS-INTENT-ROLE-RECEIPT-BINDING-MISMATCH` | blocking／semantic／after_change |

Activation後の`validator_closure.operation.physics.intent_role.migrate@1` destinationは次のexact Validator recordを持つ。current Validator／Validator closure集合はexact `[]`である。各recordはversion 1、実装Artifact ref／hash、表のinput Type ref、表のDiagnostic ref集合、self-excluding content hashを持ち、ID／version順にsortする。

| Validator | input | exact reachable Diagnostic |
|---|---|---|
| `validator.operation.request_envelope` | migration input | idempotency key reuse |
| `validator.operation.authorization` | migration input | authorization denied |
| `validator.operation.approval` | migration input | approval required |
| `validator.operation.revision_and_lock` | migration input | revision mismatch; lock conflict |
| `validator.operation.pure_predicate` | migration input | operation predicate invalid |
| `validator.operation.timeout_and_rate_limit` | migration input | timeout; rate limit exceeded |
| `validator.physics.intent_role.migration_semantics` | migration input | source invalid; registry invalid; contribution missing; contribution ambiguous; capability unavailable; axis mapping invalid |
| `validator.physics.intent_role.migration_postcondition` | postcondition input v2 | receipt binding mismatch |

Activation後のintent／request hashはExecutable Contractsの唯一の`MIRAKAN_OPERATION_INTENT_V2 -> MutationAuthorizationBindingV2 -> MIRAKAN_OPERATION_REQUEST_V2`を使う。Input、選択`PhysicsIntentRoleMigrationContributionRecordV1`、Prepared payloadの`retained_source_mcd_ref`はbyte equalityで、InputとPrepared payloadの`source_foundation_definition_closure_ref`もbyte equalityにする。source Closureはexact `{type.physics.legacy_gameplay_role,1,source_contract_set_hash}`、signed Inventoryが列挙したlegacy Source record／Owner、同時代Named Algorithm Registryを解決する。Operation／input／output／Policy／Validator／Diagnostic、destination Role Registry、Contributionのmapping／axis policy、request Algorithm bindingはdispatch時のdestination Foundation Closureだけへ解決する。両source Fieldをintent semantic inputとPublic Receiptから到達するPrepared payloadへ保持し、missing、旧`source_schema_ref`別名、wrong source root、同じContract Setの別Owner／Algorithm root、Input–Contribution–Receipt差、sourceをdestinationへalias、destination downgradeを一原因ずつ拒否する。

Activation後のPrepared payload Typeはexact `type.physics.prepared_intent_role_migration_receipt_payload@1`、hashはASCII `MIRAKAN_PREPARED_PHYSICS_INTENT_ROLE_MIGRATION_RECEIPT_PAYLOAD_V1`とself-excluding canonical bytesを使い、最終Receipt Typeと相互代用しない。唯一のsigned subject／wrapperは共通`PublishedDomainReceiptPayloadV2`／`PublishedDomainReceiptV2`であり、Domain固有Subject／alternate wrapperを作らない。publicationは[Executable Contracts §8](../02-foundation/executable-contracts.md#8-operation定義)をcanonical reuseし、`private Marker read-back → secret-free PublicCommitClosureV1 candidate → signed wrapper read-back → PublicCommitClosureV1＋PublicPublicationMarkerV1＋after Projectのatomic CAS`の順に固定する。Closureの`domain_commitment.kind`は`owner_typed_state_commit`、`domain_owner_ref`はcurrent Foundation Closureが解決するexact `owner.core.physics` ref、committed artifact集合はPrepared payloadが束縛したreceipt-free Selection Document／関連artifact ref集合とし、Closure Ref／hashの規則は同節から参照して本書で複写しない。Closure bodyまたは同Closureを束縛するsigned wrapperを欠くPublic Marker／after-state current authorityを拒否する。同じidempotency key＋request hashのretryはbyte-identical Result／`PublicCommitClosureV1`／signed Receipt／Public Markerを返し、同じkey＋別requestはidempotency reuse errorでSource不変にする。Validator error union、Operation `errors[]`、Manifest `diagnostic_refs[]`は15 refのset equalityにする。Manifestは上記Operation／Type／Policy／Validator／Registry／Diagnosticのexact version／hashと、選択Contributionごとのexact `PhysicsIntentRoleMigrationQualificationBindingRefV1`を全件持つ。各Bindingは同じContribution refをsubjectにする署名済みReceiptへexact解決し、ManifestがReceipt refまたはFixture bodyを直接保持しない。missing／extra／staleをcompile前に拒否する。ManifestのOperation LocalRef集合と`service.offline_project_migrator`へのallowlist contributionはexact一件でset equalityとし、同じatomic activation transactionでService local recordとset rootを生成する。未導入Capabilityの旧予約値は`capability_unavailable`でSourceを不変にし、Core active roleへ近似変換しない。current serializer／AI projectionは旧enum値を受理しない。

複合objectは複数Resolutionと明示関係で表し、合成enumを追加しない。同じobjectへ複数motion authorityを選ばない。Dynamic Bodyへ`static_triangle_mesh`／`heightfield`を選ばず、Sensorをauthoritative hitへ暗黙昇格せず、`teleport`を経路hitの代用にしない。途中経路がGameplayへ必要なら`authoritative_sweep`を使用する。

### 5.2 Capability discoveryと意味解決

Resolverは[Executable contracts](../02-foundation/executable-contracts.md)のCapability registryから、exact World Space Profile、active maturity、Target、Collision capability、authoring permission、Physics Intent Role Registryを読み、current MCDへ完全登録済みのexact Operationだけを提示する。Backend featureをCapabilityとして直接表示しない。unknown／removed／reserved-unsupported role、required Capability不足は`capability_unavailable`とし、近いCore roleまたは別のactive Operationへsilent downgradeしない。

解決順は次である。

1. source requestのcontent hash、Project revision、Contract set hash、Target Profile、active Physics Intent Role Registry refを取得し、Resolutionへ観測値としてbindする。
2. ユーザー文からowner vocabulary候補と、motion、contact、hit、shape、speedの独立候補を抽出し、dimensionはexact World Space Profileからだけ導く。
3. Scene／Projectの既存World、Body、Collider、Profile、Capabilityをread-only discoveryする。
4. exact owner role refとclosed axisへ候補を割り当て、矛盾と欠落を分類する。
5. gameplay behaviorを変える欠落だけを`blocking_question_ids`へ入れ、`question_required`にする。安全な欠落だけをReference assumptionとして明示する。
6. role ref、motion authority、collision semantics、hit authority、shape strategy、speed policyを独立に確定する。
7. Target、Capability、Profile、field relation、forbidden mappingを同じValidatorで再検査する。対応不能は`capability_unavailable`、invalid／forbiddenは`rejected`にする。
8. 対応するPhysics／Collision authoring familyが完全にatomic Activation済みの場合だけ`ready_to_propose`とし、そのfamilyのexact外側MCD Operationに閉じたtyped change primitiveとPreview fixtureを返す。currentでは対応write familyが空なので`capability_unavailable`を返し、proposal／Preview refをcanonical omissionする。
9. GatewayがProvider出力を同じMCDとValidatorで再計算し、不一致をresolution mismatchとして拒否する。

Resolutionのvalidate、preview、Activation後のoperation proposalの各入口は、現在のsource request hash、Contract set hash、Project revision、Physics Intent Role Registry refを保存済み`source_request_hash`、`contract_set_hash`、`project_revision`、`physics_role_registry_ref`と完全一致で再検査する。一つでも異なるResolutionは`stale`として拒否し、selected MCD Operation ref／change primitive、preview、cost estimateを使用しない。最新source／contract／Project／Registry snapshotでCapability discoveryから再解決し、新しいResolution identityを発行する。stale objectをfield単位で更新、別revisionへrebase、Commitへ継続してはならない。

### 5.3 質問、Assumption、代替案

selected World Space Profile、motion authority、contact／hit authority、shape class、高速移動policy、kinematic support relation、壊れるJoint、Target Profile、概算同時instance数が挙動を変える場合は質問する。World Profileが既にexactに選択されている場合、2D／3D／hybrid gameplay spaceをPhysics側から再質問または再選択しない。質問は「どのsolverを使うか」や特定object名を前提にせず、観測可能な挙動の選択肢、影響、推奨案を示す。

明示情報がない場合もobject名からstatic／dynamic、solid／sensor、solver／query、discrete／continuousを既定化しない。Reference assumptionは登録済みrole recordと独立axisの候補として提示し、source intent、根拠、影響、owner refを持たせてPreviewから変更できるようにする。安全な選択肢が複数ある場合はtyped alternativeを最大限界内で提示し、候補ごとの差分を示す。

Authorization、Risk class、commit可否、credentialは[AI Security／Approval](../01-governance/ai-security-approval.md)だけが決定する。本書はplanned semantic actionとActivation後のexact MCD Operationの意味／validationを定め、approval表を複写しない。

## 6. Operation、preview、diagnostic、AI eval

inspect／discover／validate／preview、World Profile作成／更新、Body dynamics設定、Joint／Constraint作成／更新／削除、Physics Kinematic Motion Provider qualification提案はsemantic action vocabularyであり、それ自体はStable Operation IDまたはcurrent公開Operationではない。Physics ownerのcurrent MCD Operation集合はexact `[]`である。`operation.physics.intent_role.migrate@1`は§5のdestination templateを持つconditional legacy migration exact一件、`planning.operation_family.physics_role_selection@1`のselectは別の未Activation候補 exact一件であり、どちらもcurrent Operationへ数えない。action名から追加ID、Manifest row、Service allowlist、Provider／MCP Toolを生成しない。owner固有proposalのadapter／Provider binding actionは当該PackまたはProject、Collision geometry／filter／query actionは[Collision](../05-simulation/collision.md)へ意味上handoffするだけでOperation権限を暗黙生成しない。将来完全登録されたwriteだけが[Project state](../03-authoring/project-state.md)のChangeSetを作り、live Worldを直接mutateしない。

Activation後のPreviewはbefore／after semantic resolution、affected Entity／Asset、selected assumptions、question state、Capability availability、estimated impact class、diagnostic、rollback boundaryを示す。native setting dumpやVendor object graphをユーザー説明に使わない。Editor手動actionとAI actionは同じDocument、validator、preview、undo／redo、cookを通る。

Diagnosticは少なくとも次を区別する。

- `MIRAKAN-PHYSICS-KINEMATIC-MOTION-INCOMPATIBLE`: intent subset、Profile、Target、exact World Space Profile、Collision query relation不一致
- `MIRAKAN-PHYSICS-KINEMATIC-MOTION-RESOLUTION_FAILED`: bounded resolverがvalid resolved motionを生成できない
- `MIRAKAN-PHYSICS-KINEMATIC-MOTION-STALE_RESULT`: motion subject／intent batch／profile／provider generation不一致
- ambiguous intent／question required
- conflicting role／motion／collision semantics
- Capability unavailable／Target unsupported
- invalid profile／shape／joint／kinematic support relation
- unsafe speed／hit assumption
- stale scene／artifact／generation
- stale source request／Contract set／Project revision
- Provider／Gateway resolution mismatch
- operation scope mismatch
- adapter qualification unavailable

各diagnosticはstable code、対象path、原因、修正候補、blockingか否かを返す。Validation failureを自然言語だけで返さず、unknown intentを最も近い既知roleへ自動確定しない。

AI Evalはvalid intent、question-required intent、conflicting intent、unsupported Capability、adversarial Vendor API要求、stale discovery、preview／commit差分をfixture化する。source request hash、Contract set hash、Project revisionを個別に変更したstale fixtureは旧Resolutionのvalidate／preview／proposal拒否と、fresh discoveryからのnew Resolution発行を検査する。評価は全closed enum、4 disposition、mandatory field、semantic resolutionの正解、必要質問、unsupportedの拒否、operation boundedness、diagnosticの再現性を検査する。Evidence、provenance、trace gradingの形式は[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)だけを消費する。

次を採用しない。

- Vendor API／setting／serializationをpublic Physics contractまたはAI vocabularyにすること
- backend名をユーザーのgameplay intentとして質問すること
- unsupported Capabilityのsilent fallback
- AI、Editor、Project C++からの任意World step／callback登録
- PhysicsによるCollision geometry／event、Runtime phase／capacity、Animation poseの再所有
