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
  target_profile_refs[1..16]: sorted unique exact TargetProfileRefV1
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
    sorted unique exact TargetProfileRefV1
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

`GameplayPhysicsRoleV1` enumとそれを使用する旧Project／Document、reader／writer、Registry、Operation、ReceiptはRepositoryまたは配布releaseにmaterializeされていない。Physics intent roleはinitial V1から`PhysicsIntentRoleRegistryV1`とclosed semantic axisを正規shapeとして直接定義し、legacy migration Operation、移行Contribution／Manifest／Qualification、offline migrator、alias、compatibility shimをcurrent planへ登録しない。current serializer、Editor、AI projection、Compileは過去draftのenum値を受理または近似変換しない。


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

inspect／discover／validate／preview、World Profile作成／更新、Body dynamics設定、Joint／Constraint作成／更新／削除、Physics Kinematic Motion Provider qualification提案はsemantic action vocabularyであり、それ自体はStable Operation IDまたはcurrent公開Operationではない。Physics ownerのcurrent MCD Operation集合はexact `[]`である。`planning.operation_family.physics_role_selection@1`のselectは未Activation候補 exact一件であり、current Operationへ数えない。action名から追加ID、Manifest row、Service allowlist、Provider／MCP Toolを生成しない。owner固有proposalのadapter／Provider binding actionは当該PackまたはProject、Collision geometry／filter／query actionは[Collision](../05-simulation/collision.md)へ意味上handoffするだけでOperation権限を暗黙生成しない。将来完全登録されたwriteだけが[Project state](../03-authoring/project-state.md)のChangeSetを作り、live Worldを直接mutateしない。

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
