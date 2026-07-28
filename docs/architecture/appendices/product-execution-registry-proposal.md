# Product Execution Registry Proposal

- 文書ID: mirakan.appendix.product-execution-registry-proposal
- 文書種別: proposal appendix
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 親Owner: [Product Plan](../00-product/product-plan.md)
- 正本範囲: Product execution Registry、Policy、Gate、Work Packageの候補Catalog
- 非正本範囲: 親Ownerが所有する安定Architecture原則、実装Task、実装順序、生成済みArtifactまたはQualification結果
- 規範依存: [親Owner](../00-product/product-plan.md)
- 関連文書: [Architecture Governance](../01-governance/architecture-governance.md)
- 根拠区分: project-decision／provisional。実ArtifactがないRegistry、Catalog、Fixtureは候補
- 外部根拠確認日: 2026-07-27

> この補助文書の型、Registry、Catalog、Fixtureは、対応するRepository Artifactが存在しない限り未実装の設計候補である。親Ownerの安定原則や実装済み状態を上書きしない。

本文中の`Task 0`～`Task 10B`と`task-*`は、既存Control Plane bootstrap protocolから引用した状態・Receipt labelであり、本書が新しい実装Task、担当、日程または作業順序を計画するものではない。
## 11. Product execution registries

本節はControl Plane移行時にMCDへ移すProduct-owned機械正本である。それまではMarkdown表を入力とし、表外のID、暗黙行、前方一致、別名を拒否する。

本節のRegistryはcurrent source Product Definitionを表す。§5.1.1のRPG-first target directionを理由に既存行を逐次改変せず、focused owner designとatomic Product Definition migrationが完了するまでは、Shooter fixture、Work Package、Capability参照をsource baselineとして保持する。RPG向けの未定義ID、Availability、Receiptを本節から推測または生成しない。

### 11.1 Registry共通規則

全Registryは`registry_id`、`format_major=1`、`revision`、`entries[]`を持つ。現行初版の`revision`は1であり、以後は同じ`registry_id`／`format_major`内で1ずつ増えるpositive safe integerとする。Active Definition変更時に変更のないRegistryは同じrevision／bytesを再利用し、変更するRegistryだけを`N+1`へ進める。revisionの巻戻し、gap、同revision別bytes、内容変更なしのrevision増加を拒否する。`ActiveProductDefinitionBundleV1.revision`と`FuturePortfolioDefinitionBundleV1.revision`も同じ規則を持ち、いずれかのmember ref／hashまたはbundle Fieldが変わる場合だけexact `N+1`、全bytes不変なら同revisionを再利用する。Active headはsigned operational current pointerとinline `CurrentControlPlaneBaselineBindingV1`で一件に固定し、genesisではBootstrap branch、後続Rebaselineではtagged rebaseline branchを検証する。Future headは`FuturePortfolioApprovalV1`のCASで一件に固定し、同revision別closureを拒否する。`entries[]`はlogical IDのUTF-8 byte順、重複禁止である。Markdown表は人間の可読性のためPhase順などで提示してよいが、generatorは表をID付き集合として読み、serialized `entries[]`をlogical IDのUTF-8 byte順へ正規化する。validatorは正規化前後のID集合が完全一致することを検査し、表の表示順を保存状態やidentityとして扱わない。参照はexact logical IDだけを受理し、display name、path、配列index、maturity、Phase番号、schema versionをidentityとして使わない。

本節の`CapabilityRegistryV1`はProduct Phase、Target、fallback、Product labelのactivation判定に使うProduct-owned選抜registryであり、全Subsystem内部Capability catalogの複製ではない。Collision、Physics、Animation、Input、Audio等のSubsystem-local C0／C1 contractとdiscovery entryは各Ownerが所有する。ただしProduct Phase、Work Package、fixture、C2 matrix、またはTarget GateからCapability IDを参照する場合は、同じChangeSetで本節へexact行、owner Work Package、Target scope、fallbackを追加しなければならない。Subsystem catalog entryだけをProduct activationの代用にせず、本節未登録IDをProduct-qualifiedとして扱わない。

Foundation MCDとのbridgeを次に固定する。`capability.runtime.scheduling@1`はExecutable Contractsに完全なactive MCD recordを持ち、その`supported_targets`は`{target.android.mobile, target.apple.mobile, target.headless.host, target.windows.desktop, target.windows.editor}`である。本節の同ID行はこの五Targetすべてを`required`で持ち、set equalityを必須にする。MCD `status=active`は契約がContract setで参照可能であることだけを意味し、`ProductOperationalStateSnapshotV1`のActivation rowを変更しない。Candidate／Qualification／Production Receiptがないrowは既定どおり`not_activated`であり、MCD recordをReceiptとして扱わない。

`capability.authoring.command_gateway@1`はTrusted Serviceのauthority境界を記述するcurrent C0 Capabilityで、Product選択、Phase label、Target shippingを表すCapabilityではない。このためProduct `CapabilityRegistryV1`に行を持たないことが正規であり、欠落補完、暗黙Product Activation、利用可能表示を行わない。Serviceの起動可否は[AI Security／Approval §7.3](../01-governance/ai-security-approval.md#73-trusted-service-isolation)のservice-authority branch、すなわちcurrent Signer Policy operation集合、Service allowlist、Trust Head／closure、executable／Profile／Contract set／Targetのfresh Technical Qualificationをset equalityにして判定し、Product row不存在を起動不能またはProduct support claimへ読み替えない。`capability.authoring.offline_migration@1`、`service.offline_project_migrator@1`、`profile.isolation.offline_project_migrator@1`のcurrent集合はexact `[]`で、Executable Contracts §8.1.2のlegacy evidence gateまたは将来の汎用schema migration familyを満たすatomic activationだけが初めて三recordを追加できる。将来Product Phase／WP／fixture／Target Gateがいずれかを直接参照する場合だけ、本節の通常ChangeSet規則でProduct-owned行を追加する。

MCD `capability.runtime.scheduling@1.required_capabilities=[]`はCapability契約そのもののsemantic dependencyである。一方、`wp.runtime.scheduling-core`の`requires_work_package_refs[]`と他WPから同WPへのedgeは実装／検証のbuild orderであり、Product Planが所有する。両者を同じ配列として比較せず、MCDが空であることを理由にWP依存を削除したり、WP依存からMCD Capability dependencyを暗黙生成したりしない。

```text
CapabilityRegistryV1
  entries[]: capability_id, target_product_tier, owner_work_package_ref,
             target_bindings[], fallback_ref
  target_bindings[]: target_id, scope(required | optional | excluded)

CapabilityTargetActivationStateV1 // operational state only
  capability_id, target_id, state, candidate_ref, receipt_refs[]

PlannedOperationFamilyRefV1
  planning_record_id
  planning_record_version
  planning_record_hash

ExecutionSurfaceBindingV1
  planned_family_ref: PlannedOperationFamilyRefV1
  activation_work_item_id
  required_operation_ids[1..64]
  required_operation_set_sha256: lowercase hex 64
  execution_host_target_refs[1..5]
  artifact_target_id: null | target_id
  required_execution_state = operational
  provider_projection_requirement =
    none
    | all_activation_declared_external_safe
    | proposal_subset_excluding_trusted_internal

AiCapabilityActivationMatrixV1
  matrix_id = matrix.product.ai-capability-activation.v1
  matrix_version = 1
  execution_surface_required_capability_refs[10]
  deny_only_capability_refs[1]
  rows[10]:
    capability_id
    owner_work_package_ref
    requirement_refs[]
    phase_refs[]
    fixture_refs[]
    risk_refs[]
    state_transition_policy_refs[]:
      ProductOperationalStatePolicyRegistryV1.state_policy_id
    candidate_binding_policy_literal:
      ProductCandidateBindingPolicyLiteralV1
    planning_family_refs[]
    execution_host_rule =
      windows_editor_for_artifact_target
      | headless_proposal_only
    provider_projection_requirement
  matrix_sha256

CapabilityTargetActivationBindingRegistryV1 // immutable promotion definition
  ai_capability_activation_matrix: AiCapabilityActivationMatrixV1
  entries[]:
    capability_id, target_id, owner_work_package_ref
    candidate_lock_policy = owner_wp_task_technical_qualification
    execution_surface_bindings[]: ExecutionSurfaceBindingV1
    qualification_policy = all_evidence_bindings | unavailable_no_gate
    qualification_evidence_bindings[]:
      {provider_work_package_ref, gate_ref, fixture_ref,
       requirement_refs[], freshness_policy_ref}
    qualification_gate_refs[]
    qualification_requirement_refs[]
    production_policy =
      {kind: disabled_current_definition, release_gate_refs: []}
      | {kind: all_release_gates, release_gate_refs[]}
    candidate_binding_policy_ref
    freshness_policy_refs[]

ProductReleaseGateRegistryV1
  entries[]:
    release_gate_id, target_id, fixture_id,
    evaluated_requirement_refs[], required_phase_gate_refs[],
    required_work_package_states[]: {work_package_id, required_state},
    candidate_binding_policy_ref = policy.product.same-candidate.v1,
    freshness_policy_ref,
    critical_risk_predicate = all_impact_critical_effective_mitigated_or_closed

ProductPhaseRegistryV1
  entries[]: phase_id, order_index, outcome_requirement_refs[], work_package_refs[], exit_gate_refs[]

ProductCandidateBindingPolicyLiteralV1 =
  policy.product.same-candidate.v1

WorkPackageRegistryV1
  entries[]: work_package_id, phase_id, owner_document_id, target_refs[], fallback_ref,
             provided_fixture_refs[]: ProvidedFixtureRefV1,
             required_capability_refs[], requires_work_package_refs[],
             scheduling_state, defer_reason, reconsideration_gate_refs[], blocked_reason_ref

ProvidedFixtureRefV1 =
  {kind: product_fixture,
   fixture_id: FixtureRegistryV1.fixture_id}
  | {kind: component_qualification_fixture,
     owner_document_id,
     owner_pack_ref: PackContractRefV1,
     owner_pack_manifest_ref: ArtifactRefV1(
       artifact_kind=pack_manifest, schema_version=1),
     owner_pack_manifest_sha256,
     fixture_id, fixture_version, fixture_content_sha256}

PhaseFixtureBindingRegistryV1
  entries[]: gate_id, phase_id, fixture_id, evaluated_requirement_refs[], target_refs[],
             candidate_binding_policy_ref = policy.product.same-candidate.v1,
             freshness_policy_ref

TargetProfileRegistryV1
  entries[]:
    target_id, owner_document_id, profile_version,
    target_kind =
      headless_server | desktop | mobile | console |
      web | xr | distributed_cluster,
    surface_role = execution_host | artifact_runtime | both,
    qualification_requirement_ref

RequirementRegistryV1
  entries[]: requirement_id, owner_document_id, verification_kind, failure_diagnostic_id

FixtureRegistryV1
  entries[]: fixture_id, owner_document_id, requirement_refs[], target_refs[], minimum_duration_seconds
```

次の二recordはRegistry一覧に必要なentry payloadの投影であり、型の再定義ではない。`FallbackRegistryV1`のclosed canonical schemaは§11.3、`FutureCapabilityIncubationRegistryV1`の唯一のcanonical schemaは§8が所有し、本一覧はそれらとField名・意味を一致させる。

```text
FallbackRegistryV1
  entries[]: ProductFallbackRecordV1 {
    fallback_id, owner_document_id, preserves_semantics,
    failure_diagnostic_ref: ProductDiagnosticRefV1}
  diagnostic_records[]: ProductDiagnosticRecordV1

FutureCapabilityIncubationRegistryV1
  entries[]: future_capability_id, owner_document_id, planning_state,
             prerequisite_capability_refs[], prerequisite_future_capability_refs[], required_decision_kinds[], candidate_target_kinds[],
             qualification_fixture_kinds[], activation_trigger, excluded_current_product_claims[]

ProductClaimDefinitionRegistryV1
  entries[]: claim_id, canonical_label, claim_scope

ProductRiskDefinitionRegistryV1
  entries[]: risk_id, owner_document_id, affected_work_package_refs[], trigger,
             likelihood, impact, mitigation, contingency, monitor_gate_refs[], revisit_gate_or_date,
             genesis_state = open
  revisit_gate_or_date:
    { kind: phase_gate, ref: PhaseFixtureBindingRegistryV1.gate_id }
    | { kind: decision_gate, ref: ProductDecisionGateRegistryV1.gate_id }
    | { kind: date, value: YYYY-MM-DD }

ProductDecisionGateRegistryV1
  evidence_class_definitions[]:
    {class_id, owner_document_id, wrapper_schema_id, signed_record_purpose,
     freshness_policy_ref, subject_policy_ref, required_target_refs[]}
  definition_change_class_definitions[]:
    {class_id, owner_document_id, wrapper_schema_id, signed_record_purpose,
     subject_policy_ref}
  entries[]:
    gate_id, owner_document_id
    genesis_state = blocked
    predicate:
      evaluator_policy = all_of
      required_phase_gate_refs[]
      required_work_package_states[]: {work_package_id, required_state}
      required_evidence_classes[]
      required_definition_change_classes[]
    on_satisfied_action:
      {kind: permit_work_package_transition,
       work_package_id, from_state, to_state, transition_policy_ref}

WorkPackageLifecyclePolicyRegistryV1
  entries[]: transition_policy_id, allowed_edges[], subject_binding_policy,
             prerequisite_policy, receipt_policy, authorization_requirements[],
             bootstrap_only_work_package_ref
  authorization_requirements[]: {edge, authorization_schema, authorization_kind}

ProductOperationalStatePolicyRegistryV1
  entries[]: state_policy_id, change_kind, allowed_edges[], authority_role_ref,
             signed_record_purpose, evidence_policy_refs[], decision_requirements[]
  decision_requirements[]: {edge, requirement}
```

`ProductFallbackRecordV1`、`ProductDiagnosticRefV1`、`ProductDiagnosticRecordV1`のexact Field、hash、set-equality規則は§11.3のclosed shapeをforward-referenceする。この概要からID-only `failure_diagnostic_id`、anonymous Diagnostic、別Registryを生成しない。`RequirementRegistryV1.failure_diagnostic_id`はRequirement評価器の別closed tokenであり、Fallbackのtyped Product Diagnostic refへ機械置換しない。

Work Package表の`provided_fixture_refs[]`列は`ProvidedFixtureRefV1`の可読source notationである。`fixture.product.*`は`kind=product_fixture`として同じActive Definitionの`FixtureRegistryV1`へexact解決する。現行表で許可する`kind=component_qualification_fixture`は、`fixture.feature.combat.contract`、`fixture.feature.ranged_combat.contract`、`fixture.feature.encounter_spawn.contract`、`fixture.feature.scoring.contract`、`fixture.feature.pickup_grant.provider_neutral`、`fixture.feature.interaction.contract`、`fixture.feature.character_locomotion.motion_executor`、`fixture.feature.path_following.executor_stub`、`fixture.feature.scenario_stage.none`のexact九件だけである。先頭八件は`owner_document_id=mirakan.arch.pack-gameplay-features`かつ各対応`feature.combat@1 | feature.ranged_combat@1 | feature.encounter_spawn@1 | feature.scoring@1 | feature.pickup_grant@1 | feature.interaction@1 | feature.character_locomotion@1 | feature.path_following@1`の`PackContractRefV1`、Scenario一件は`owner_document_id=mirakan.arch.pack-scenario-stage`かつ`feature.scenario_stage@1`の`PackContractRefV1`へ解決する。generatorは各current approved owner Pack Manifestから`owner_pack_ref`、完成manifest ref／hash、fixture version／content hashを解決してtagged objectを生成し、Gameplay Featuresのaggregate表示をScenario ownerへ偽装せず、裸IDを保存しない。missing、owner／Pack不一致、version／hash不一致、上記以外のprefixまたはfeature fixtureをDefinition construction errorにする。component fixtureは当該WP実装のQualification入力であってProduct Phase／Release Gateのevidenceではない。Product Gateへ使うには`FixtureRegistryV1`への明示rowと`kind=product_fixture` bindingをActive Definition revisionで追加する。

`policy.product.same-candidate.v1`はRegistry lookup IDではなく、`ProductCandidateBindingPolicyLiteralV1`が許す唯一の固定literalである。そのclosed predicateはProject revision、Candidate root hash、Contract set hash、Toolchain lock hash、Target Profile ref／hash集合のbyte equalityである。`PhaseFixtureBindingRegistryV1`、`ProductReleaseGateRegistryV1`、`CapabilityTargetActivationBindingRegistryV1`の`candidate_binding_policy_ref`はこのliteralだけを受理し、別ID、未登録の説明語、部分一致を拒否する。これに対し`policy.product.activation.promotion.v1`は`ProductOperationalStatePolicyRegistryV1`の登録rowであり、両者を同じRegistry refとして解釈しない。

上記のうち`CapabilityTargetActivationStateV1`だけは定義Registryではない。Product definitionと実行中の観測状態を次の二層へ分離し、Bootstrap Approvalが可変stateをhash対象に含めて自己失効することを禁止する。

```text
ActiveProductDefinitionBundleV1 // immutable within one approved active definition revision
  registry_id, format_major, revision
  capability_registry_ref
  capability_target_activation_binding_registry_ref
  product_release_gate_registry_ref
  product_phase_registry_ref
  work_package_registry_ref
  phase_fixture_binding_registry_ref
  target_profile_registry_ref
  requirement_registry_ref
  fixture_registry_ref
  fallback_registry_ref
  product_risk_definition_registry_ref
  product_decision_gate_registry_ref
  work_package_lifecycle_policy_registry_ref
  product_operational_state_policy_registry_ref

ActiveProductDefinitionClosureV1
  bundle: ActiveProductDefinitionBundleV1
  registry_manifest[]:
    {registry_id, format_major, revision, registry_ref, registry_sha256}

FuturePortfolioDefinitionBundleV1 // independently approved planning-only hash domain
  portfolio_id, format_major, revision
  future_capability_registry_ref
  product_claim_definition_registry_ref
  future_portfolio_policy_registry_ref
  membership_revision_policy_ref
  claim_transition_policy_ref

FuturePortfolioPolicyRegistryV1
  entries[]:
    {policy_id, policy_kind, authority_role_ref,
     signed_record_purpose, required_decision_kind: null | ProductOperationalDecisionV1.decision_kind}

FuturePortfolioDefinitionClosureV1
  bundle: FuturePortfolioDefinitionBundleV1
  registry_manifest[]: {registry_id, format_major, revision, registry_ref, registry_sha256}

FuturePortfolioApprovalPayloadV1
  approval_id
  approval_sequence: positive safe integer
  future_portfolio_definition_sha256
  previous_approval_ref: null | content-addressed ref
  previous_approval_sha256: null | lowercase hex 64
  approver_subject_ref
  approval_authority_ref
  issued_at
  valid_until
  revocation_snapshot_ref

FuturePortfolioApprovalV1
  payload: FuturePortfolioApprovalPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=future_portfolio_definition_approval)

FutureProductClaimReleasePayloadV1
  claim_release_id
  claim_release_sequence: positive safe integer
  previous_release_ref: null | content-addressed ref
  previous_release_sha256: null | lowercase hex 64
  future_capability_id
  destination_active_product_definition_sha256
  promotion_manifest_ref
  promotion_manifest_sha256
  operational_state_snapshot_ref
  operational_state_snapshot_sha256
  activation_keys[]: {capability_id, target_id}
  production_receipt_refs[]
  released_claims[]
  authorization_decision_ref
  issued_at
  valid_until
  revocation_snapshot_ref

FutureProductClaimReleaseV1
  payload: FutureProductClaimReleasePayloadV1
  signed_record: MirakanSignedRecordV1(purpose=future_product_claim_release)

FutureToActivePromotionManifestPayloadV1
  promotion_manifest_id
  source_future_portfolio_definition_sha256
  source_future_portfolio_approval_ref
  source_future_portfolio_approval_sha256
  future_capability_id
  destination_active_product_definition_sha256
  promoted_active_ids[]: {registry_id, row_kind, logical_id, migration_kind}
  active_definition_migration_ref
  active_definition_migration_sha256
  generated_at
  revocation_snapshot_ref

FutureToActivePromotionManifestV1
  payload: FutureToActivePromotionManifestPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=future_to_active_promotion_manifest)

ProductOperationalStateSnapshotPayloadV1
  state_snapshot_id
  active_product_definition_sha256
  control_plane_baseline_binding: CurrentControlPlaneBaselineBindingV1
  sequence
  previous_state_snapshot_ref: null | content-addressed ref
  applied_change:
    {kind: genesis,
     bootstrap_approval_ref, bootstrap_approval_sha256,
     baseline_envelope_ref, baseline_envelope_sha256,
     construction_decision_ref, trust_provisioning_receipt_ref}
    | {kind: operational_transition, transition_wrapper_ref, transition_wrapper_sha256}
    | {kind: wp_lifecycle, lifecycle_wrapper_ref, lifecycle_wrapper_sha256}
    | {kind: active_definition_migration, migration_wrapper_ref, migration_wrapper_sha256}
    | {kind: control_plane_rebaseline, rebinding_wrapper_ref, rebinding_wrapper_sha256}
  capability_target_activation_rows[]: CapabilityTargetActivationStateV1
  decision_gate_evaluations[]: {gate_id, state, evidence_refs[]}
  risk_evaluations[]: {risk_id, state, evidence_refs[]}
  work_package_lifecycle_heads[]:
    {work_package_id, lifecycle_record_ref: null | content-addressed ref, lifecycle_sequence}
  created_at
  revocation_snapshot_ref

ProductOperationalStateSnapshotV1
  payload: ProductOperationalStateSnapshotPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=product_operational_state_snapshot)
```

`active_product_definition_sha256`は`ActiveProductDefinitionBundleV1`と全参照active definition Registryのcanonical bytes closureから決定論的に算出する。`ControlPlaneBootstrapApprovalV1`、baseline、Critical Path Taskはこのhashを`product_registry_sha256`という旧名でなくexact `active_product_definition_sha256`として束縛する。Activation、Decision Gate評価、Risk評価、Work Package lifecycle headの通常更新はdefinition bytesを変更しない。

`control_plane_baseline_binding`はControl Plane設計が所有するclosed `CurrentControlPlaneBaselineBindingV1`をinlineで保持する。初回genesisは`kind=bootstrap`、Active Definition migrationは同payloadのRebaseline Approval／Envelope／Transactionから`kind=rebaseline`を生成し、通常state transition／WP lifecycleはparent値をbyte-exactに保持する。bindingの`active_product_definition_sha256`はsnapshot同Fieldと一致しなければならず、初回Approvalを後続definitionへ流用、別definitionのEnvelope、missing Transaction、任意の`latest` lookupを拒否する。通常Taskは「初回Bootstrapが永続的にcurrent」と仮定せず、このcurrent bindingだけをentry conditionに使う。

`ActiveProductDefinitionClosureV1`は上記2 Fieldだけのclosed objectで、`registry_manifest[]`をregistry IDのunsigned UTF-8 byte順にする。`active_product_definition_sha256 = SHA-256(JCS(ActiveProductDefinitionClosureV1))`とし、文字列連結、filesystem順、別framingを使わない。manifestはBundleが要求する全registryとset equalityで、missing／extra registry、ref先hash差、unknown Fieldを拒否する。全`*_sha256` Fieldはprefixなしlowercase hex 64文字、content-addressed `*_ref`は`sha256:<lowercase-hex-64>`で完成artifactまたは完成signed wrapperを指す。refと隣接hash Fieldは同じbytesへ解決する。Operational snapshotのActivation rowsは`{capability_id,target_id}`、Decisionは`gate_id`、Riskは`risk_id`、WP headは`work_package_id`のunsigned UTF-8 byte順で、全配列を重複なしとする。Receipt／Evidence／Target／policy ref配列もunsigned UTF-8 byte順で、duplicate、unknown、未正規順を拒否する。Markdown表内のsemicolon列は集合の可読表示であってserialization順ではない。generatorはtop-level `entries[]`だけでなく全nested set配列をschema指定keyのunsigned byte順へ正規化し、ID集合が入力表と一致することを確認してからJSONを生成する。validatorは生成済みJSONの未正規順を拒否し、表の表示順をhashへ使わない。

JSONへ保存する全Product `sequence`はinteger `0..9007199254740991`（ECMAScript safe integer）に固定する。genesisだけ0、通常nextはexact `N+1`、上限でのincrementはoverflowとして拒否し、float、負値、指数表記文字列、丸めたuint64を受理しない。Revocation sequenceも同じcanonical表現を使う。

`FuturePortfolioDefinitionClosureV1`はActive closureと同じJCS、manifest set equality、ref／hash、sort規則を別hash domainで適用し、manifestは`FutureCapabilityIncubationRegistryV1`、`ProductClaimDefinitionRegistryV1`、`FuturePortfolioPolicyRegistryV1`のexact 3件とする。`future_portfolio_definition_sha256=SHA-256(JCS(closure))`とし、claim definitionを別fileやMarketing表へ逃がさない。`FuturePortfolioApprovalV1`は`role.future-portfolio-approver.r4`へのactive human assignmentとsingleton purpose keyで署名し、payload／signed recordのsubject、Signer、Role、issued_at、revocation snapshotをexact一致させる。`approval_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:future-portfolio-approval:sha256:<lowercase-hex>`として導出する。初版だけprevious両Field=nullかつ`approval_sequence=1`、後続はcurrent completed approval wrapper ref／hashを持ち、同じapproval chain内で`approval_sequence`をexact `N+1`、current head CASで一件だけ進める。Future Definition内容を変更する場合だけBundle `revision`をexact `N+1`として新hashを承認し、内容不変のexpiry／revocation後renewalは同じdefinition hash／revisionを保持してapproval sequenceだけを進める。内容不変revision bump、内容変更を伴う同revision、sequence gap／branchを拒否する。expired／revoked approvalのportfolioはplanning表示も`unapproved`とし、active Product対応を意味しない。

`FuturePortfolioPolicyRegistryV1`の初期entryは次の2件だけである。Bundleの二つのpolicy refはこのexact IDへ解決し、policy RegistryをFuture closure manifestへ含める。

| policy_id | policy_kind | authority_role_ref | signed_record_purpose | required_decision_kind |
|---|---|---|---|---|
| `policy.future.membership.revision.v1` | `membership_revision` | `role.future-portfolio-approver.r4` | `future_portfolio_definition_approval` | `null` |
| `policy.future.claim.release.v1` | `claim_release` | `role.future-claim-release.r4` | `future_product_claim_release` | `future_claim_release_product` |

planning-only entryの追加、説明修正、claim vocabulary更新、Future dependency変更はFuture membership revisionだけを更新し、current active Product Definition、WP lifecycle、Activation、Phase Evidenceを失効させない。Future entryをactiveへ移すChangeSetだけがdestination Active Product Definitionを変更し、destination `ControlPlaneRebaselineApprovalV1`／Envelope／Transaction、Product migration Decision、full-reset state migrationを必要とする。初回Bootstrap Approvalを再発行または後続Definitionへ流用しない。`prerequisite_future_capability_refs[]`は同じapproved Future closure内のexact IDだけを参照し、自己参照、missing、cycleを拒否する。前提Futureはそれぞれactive definitionへ移行し必要Targetでproductionになるまでconsumer entryをactive migration候補にしない。

Future起源のactive migrationでは`future_promotion_inputs[]`にsource Future closure hash、その時点のcurrent完成`FuturePortfolioApprovalV1` ref／hash、Future ID、追加／保持するactive Registry IDを列挙し、migration authorization Decision、row migration manifestと一緒に承認する。Approvalのpayloadが同じsource Future closure hashを承認し、current approval headで有効、非失効でなければならない。同一definition hashのrenewalでもapproval ref／hash／sequenceを省略しない。配列はFuture ID、内側IDは`{registry_id,row_kind,logical_id}`のunsigned byte tuple順で、duplicate、source entry不存在、row manifestにないID、`removed` IDを拒否する。migration完成後、専用`service.future-to-active-promotion-manifest-publisher`／`role.future-to-active-promotion-manifest-publisher`／singleton purpose `future_to_active_promotion_manifest`が、migration inputのsource Approval ref／hashをbyte-exactに複写した同じmappingと完成migration wrapper ref／hashを`FutureToActivePromotionManifestV1`として署名する。`promotion_manifest_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:future-promotion:sha256:<lowercase-hex>`として導出し、`signed_record.issued_at=payload.generated_at`、revocation snapshotをexact一致させる。これをFuture IDとactive IDsの唯一のorigin mappingとし、名前一致や説明文から由来を推測しない。

`excluded_current_product_claims[]`はplanning中の禁止claimであり、active migrationだけで自動解除しない。解除は、valid `FutureToActivePromotionManifestV1`が列挙するactive Capability、必要Target全Activation `production`、fresh Release Receipt、effective open Critical Risk 0、kind `future_claim_release_product`／subject kind `future_product_claim_release`のfresh R4 `ProductOperationalDecisionV1`を閉じた`FutureProductClaimReleaseV1`を必要とする。Claim payloadのoperational snapshot ref／hashは発行時監査とcommit条件であり、validatorはそのinline rowを`activation_keys[]`でlookupしてproduction、same Candidate、freshness、promotion manifest内Capability／Target closureを再計算する。独立しない`activation_row_refs[]`を発明しない。`claim_release_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:future-claim-release:sha256:<lowercase-hex>`として導出し、`role.future-claim-release.r4`とsingleton purpose keyで署名する。`signed_record.issued_at=payload.issued_at`、revocation snapshot、purpose、Roleをexact一致させる。

Claim release chainは`future_capability_id`ごとに一つで、初回だけprevious両Field=nullかつ`claim_release_sequence=1`、後続はcurrent完成Release wrapper ref／hashとexact `N+1`を持ち、per-Future current-head CASで一件だけ進める。branch、sequence gap、別Futureのprevious、filesystemのlatest探索を拒否する。read-time `effective_release`は発行時snapshotをcurrentと見なさず、Product operational current headから同じ`activation_keys[]`を毎回key lookupし、current headのactive definition hashがpayloadのdestination hashと一致し、全行がproductionかつeffective availability、same Candidate、fresh non-revoked Release Evidence、Risk 0、current Decision／Signer validityを満たす場合だけ列挙claimを解除する。無関係transitionで条件が不変ならReleaseは有効を維持するが、Definition migration、Target降格、Candidate変更、Evidence expiry／revocation、Decision／Release期限切れで即時effective unreleasedへ戻す。Marketing文言やCapability存在だけをrelease Receiptにしない。

`released_claims[]`はsource Future rowの`excluded_current_product_claims[]`に含まれ、同じsource Future closureの`ProductClaimDefinitionRegistryV1`へ解決するnon-empty `claim_id` subsetだけを許し、全解除を主張する場合はset equalityを必須とする。unknown／duplicate／別Future claim、canonical label、Marketing自由文字列を拒否する。`activation_keys[]`はPromotion Manifestが列挙した全promoted Capabilityのdestination `CapabilityRegistryV1.target_bindings[]`のうち`required | optional`全keyから決定論的に導くexact setであり、呼出元がTargetを省略できない。`production_receipt_refs[]`はそのcurrent Activation rows、`CapabilityTargetActivationBindingRegistryV1`、該当`ProductReleaseGateRegistryV1`をproductionで支えるfresh Receiptのexact closureで、missing／extra／別Candidate／別Targetを拒否する。部分claimでもActivation／Production Evidence closureを縮小しない。

`CapabilityTargetActivationBindingRegistryV1`はCapabilityの「どのEvidence provider／Gateと実行surfaceで昇格できるか」を一意にする。`CapabilityRegistryV1`の全`required | optional` bindingとentriesの`{capability_id,target_id}` set equalityを必須とし、`excluded` row、missing、extra、duplicateを拒否する。各rowは次の決定論的規則でmaterializeし、手入力でGateまたはOperationを増減しない。

1. `owner_work_package_ref`はCapability rowとexact一致する。Evidence providerは既定でOwner WP一件、下表のexact keyだけは表のWP一件へ置換する。複数候補、暗黙の依存WP、display name一致を拒否する。
2. providerごとに、`PhaseFixtureBindingRegistryV1`のうちTargetがrowの`target_id`を含み、Fixtureがprovider WPの`provided_fixture_refs[]`内の`kind=product_fixture`に含まれ、Gate Phase orderがOwner WPとprovider WPの両Phase order以上であるGateをcandidate setにする。`kind=component_qualification_fixture`はWP自体のQualificationへだけ使い、このProduct Gate候補へ混ぜない。non-emptyなら最小Phase orderを求め、そのorderにある全Gateを一件ずつ`qualification_evidence_bindings[]`へ展開する。bindingのprovider／gate／fixture／requirements／freshnessは参照先のexact値で、同じ最小Phase／Targetの全Gateを要求する。
3. `qualification_gate_refs[]`は全evidence bindingのGate exact union、`qualification_requirement_refs[]`はRequirement exact union、`freshness_policy_refs[]`はfreshness exact unionである。各unionとbinding間のmissing／extraを拒否する。
4. `AiCapabilityActivationMatrixV1.execution_surface_required_capability_refs[]`は下表のpositive十Capability exact集合、`deny_only_capability_refs[]`は`capability.product.runtime-generation-boundary`一件で、両集合の積は空とする。positive Capabilityの全Target rowでは、行が列挙する各`planning_family_ref`を[Executable Contracts](../02-foundation/executable-contracts.md#20-ai向けdiscoveryexecution候補のplanning-record未activation)の完成`PlannedOperationFamilyV1`へID／version／hashでexact解決し、familyの`reserved_candidate_ids[]`全件を`required_operation_ids[]`へ複写する。`required_operation_ids[]`はNFC UTF-8 Operation ID byte順、duplicate禁止である。`required_operation_set_sha256`はASCII `MIRAKAN_REQUIRED_OPERATION_SET_V1`、Operation count、各Operation IDのNFC UTF-8 bytesをこの順に`uint32_be` length framingしたbytesのSHA-256 lowercase hexであり、family ID、Markdown表示順、delimiter連結、prefix matchをhash入力にしない。familyの一部だけ、別version、別work item、missing／extra Operation、set hash不一致を拒否する。該当しないCapabilityの`execution_surface_bindings[]`はexact `[]`、deny-only Capabilityもpositive bindingを持たない。
5. `required_execution_state=operational`は保存されたActivation stateではなくqualification時のcurrent derived predicateである。各familyについてcurrent Contract Set、Owner Manifest、Trusted Service allowlist、Policy、Validator／closure、Diagnostic、Receipt、Signer Policy、Trust、Service Technical Qualificationの完全集合を[Executable Contracts](../02-foundation/executable-contracts.md)と[AI Security／Approval](../01-governance/ai-security-approval.md)のlike-for-like規則で再計算し、`required_operation_ids[]`全件が同じContract setでoperationalの場合だけ満たす。planning recordの`capability_state=not_activated`、一部Operationだけ、Signer rowだけ、Provider aliasだけ、stale Contract set、Provider投影過多／不足はfalseである。`provider_projection_requirement=none`はProvider／MCP projection exact `[]`、`all_activation_declared_external_safe`はActivation recordで`exposure=external_safe`と確定した全Operationのexact集合、`proposal_subset_excluding_trusted_internal`は同external-safe集合のうちread／query／plan／propose／validate／preview／requestだけのexact集合である。後二者とも`trusted_internal` Commit／Promotion／Approval／Activation／Signing／ReleaseをProviderまたはMCPへ含めず、Operation名やHTTP routeからexposureを推測しない。
6. `execution_host_rule=windows_editor_for_artifact_target`は`execution_host_target_refs=[target.windows.editor]`、`artifact_target_id`を当該Capability rowの`target_id`へする。Android／Apple artifactをWindows Editorで生成しても、Build／Cook／Package／Test ReceiptのTargetはAndroid／AppleのままでWindows Receiptを流用しない。`headless_proposal_only`は`execution_host_target_refs=[target.headless.host]`、`artifact_target_id=null`で、Commit／Promotion／Activationを外部Toolへ投影しない。
7. Evidence binding集合がnon-emptyで、かつ全`execution_surface_bindings[]`がcurrent `operational`なら`qualification_policy=all_evidence_bindings`で全bindingのfresh success、Owner WPと全provider WPのcurrent state=`complete`、same Candidateを必須にする。Evidenceがemptyなら`unavailable_no_gate`、必要実行surfaceが一件でも非operationalなら`not_activated`からの昇格を拒否する。WP完了、fixture pass、説明文のOperation名を実行surfaceの代用にしない。
8. `candidate_locked`はOwner WPのcurrent `CriticalPathTaskV1`と同じCandidateへ閉じたfresh `TechnicalQualificationReceiptV1`だけで許可する。
9. 現DefinitionはCX3 Release／Product Release Gateが未成立のため全row `production_policy={kind=disabled_current_definition, release_gate_refs=[]}`とし、`production`遷移を拒否する。`wp.product.production-release-binding`のsource `production_release_migration_authoring`が生成したpreliminary destination候補をR4 Product migration Decisionで承認した場合だけ、destination Active Definitionの各rowを、そのTargetと一致する`ProductReleaseGateRegistryV1`一件を持つ`{kind=all_release_gates, release_gate_refs=[...]}`へ変更できる。migration V1のfull reset後、同WPをdestination `production_release_binding_qualification`として再実行し、そのWPがcomplete、かつrelease gateがsame Candidateでfresh successの場合だけ`qualified->production`を許可する。Target不一致、empty／複数gate、Release WP未完了、旧definition Receiptのcarryを拒否する。

`wp.product.production-release-binding`のexecution modeはcurrent `CapabilityTargetActivationBindingRegistryV1`全293行から決定論的に導出し、Task入力として署名する。293行すべてが`disabled_current_definition`かつgate集合emptyなら`production_release_migration_authoring`、293行すべてが各Targetのexact Release Gate一件を持つ`all_release_gates`なら`production_release_binding_qualification`である。mixed kind、292以下／294以上、Target違い、empty／複数gateはinvalid Active Definitionであり、同WPを開始しない。

`production_release_migration_authoring`はGateで既にcurrent validatedな`evidence.class.product-release-policy-ready`／`evidence.class.product-release-artifact-plan-valid`の完成Receipt ref／hashをread-backし、内容を新規作成または自己発行しない。その入力をdestination Active 14 Registry closure、全293 policy rowを変更するsigned row migration入力、Target別plan projection、Control Plane Rebaseline入力へbyte-exactに組み込み、preliminary Staging outputとして生成するだけである。source lifecycleは`declared->ready->active`までを許可し、`active->complete`、`ArtifactCandidateBindingV1`、Activation／Release Evidenceへの利用を禁止する。R4 DecisionとA2→B2→C2→D2→T2→L2→P2 migrationがsource active epochを置換し、destinationへsource WP state／Receiptをcarryしない。

`production_release_binding_qualification`はActive Definition、Registry、row migration manifest、Rebaseline artifactを一byteも変更しない。同じdestination Definition／Candidateへfresh Target artifact、package、support、rollback、signing、SBOM／provenance、device-lab、Release Receipt policy Evidenceを閉じ、独立Owner acceptance付き`ArtifactCandidateBindingV1`でのみ`active->complete`へ進む。このcompletionはRelease Gateを自動満足またはActivation／human Release Approvalを発行せず、後続Gateがsame Candidate、freshness、critical Riskを別途再評価する。

| capability_id | target_id | exact evidence provider WP |
|---|---|---|
| `capability.platform.input-core` | `target.windows.editor` | `wp.platform.input-windows` |
| `capability.platform.input-core` | `target.windows.desktop` | `wp.platform.input-windows` |
| `capability.platform.input-core` | `target.android.mobile` | `wp.platform.mobile-io-ui-android` |
| `capability.platform.input-core` | `target.apple.mobile` | `wp.platform.mobile-io-ui-apple` |
| `capability.platform.audio-core` | `target.windows.editor` | `wp.platform.audio-windows` |
| `capability.platform.audio-core` | `target.windows.desktop` | `wp.platform.audio-windows` |
| `capability.platform.audio-core` | `target.android.mobile` | `wp.platform.mobile-io-ui-android` |
| `capability.platform.audio-core` | `target.apple.mobile` | `wp.platform.mobile-io-ui-apple` |
| `capability.platform.ui-core` | `target.windows.editor` | `wp.platform.ui-windows` |
| `capability.platform.ui-core` | `target.windows.desktop` | `wp.platform.ui-windows` |
| `capability.platform.ui-core` | `target.android.mobile` | `wp.platform.mobile-io-ui-android` |
| `capability.platform.ui-core` | `target.apple.mobile` | `wp.platform.mobile-io-ui-apple` |

この12行は`CapabilityTargetActivationBindingRegistryV1` row内のprovider Fieldへmaterializeする正本例外であり、別のoperational stateではない。各keyは対応Capability Target bindingとset inclusion、provider WP Targetと一致し、その他のkeyはOwner WP既定を使う。これにより後段のI/O／Audio／UI Target adapter Receipt要件と同一schemaで判定できる。

`AiCapabilityActivationMatrixV1.rows[]`のcurrent exact十行は次である。`planning family refs`は表示時のbare planning IDで、materialization時は[Executable Contracts §20](../02-foundation/executable-contracts.md#20-ai向けdiscoveryexecution候補のplanning-record未activation)の`{planning_record_id, planning_record_version, planning_record_hash}`へ展開する。配列はcanonical ID順、重複禁止であり、略称、family count、Operation prefix、`latest`参照を保存しない。

| capability_id | owner WP | Requirement refs | Phase | Fixture refs | planning family refs | host rule | Provider projection | Risk refs | Policy binding `{state transition refs; candidate binding literal}` |
|---|---|---|---|---|---|---|---|---|---|
| `capability.authoring.ai-core` | `wp.authoring.ai-core` | `requirement.product.ai-authoring-mvp-a; requirement.product.ai-genre-neutral-authoring` | `phase.ai-authoring-mvp-a` | `fixture.product.genreless-ai-project; fixture.product.shooter-2d` | `planning.operation_family.authoring_discovery; planning.operation_family.game_system_discovery; planning.operation_family.world_discovery; planning.operation_family.gameplay_definition_authoring; planning.operation_family.asset_authoring; planning.operation_family.authoring_changeset_execution` | `windows_editor_for_artifact_target` | `all_activation_declared_external_safe` | `risk.product.ai-tool-safety-code-owner` | `policy.product.activation.promotion.v1; policy.product.same-candidate.v1` |
| `capability.runtime.debug-replay-support` | `wp.runtime.debug-replay-support` | `requirement.product.ai-authoring-mvp-a; requirement.product.ai-genre-neutral-authoring` | `phase.ai-authoring-mvp-a` | `fixture.product.genreless-ai-project; fixture.product.shooter-2d` | `planning.operation_family.build_device_play_debug_task` | `windows_editor_for_artifact_target` | `all_activation_declared_external_safe` | `risk.product.ai-tool-safety-code-owner` | `policy.product.activation.promotion.v1; policy.product.same-candidate.v1` |
| `capability.runtime.ecs-e6-debug-ai` | `wp.runtime.ecs-e6-debug-ai` | `requirement.product.ai-authoring-mvp-a; requirement.product.ai-genre-neutral-authoring` | `phase.ai-authoring-mvp-a` | `fixture.product.genreless-ai-project; fixture.product.shooter-2d` | `planning.operation_family.authoring_discovery; planning.operation_family.game_system_discovery; planning.operation_family.world_discovery; planning.operation_family.gameplay_definition_authoring; planning.operation_family.asset_authoring; planning.operation_family.authoring_changeset_execution; planning.operation_family.build_device_play_debug_task` | `windows_editor_for_artifact_target` | `all_activation_declared_external_safe` | `risk.product.ai-tool-safety-code-owner` | `policy.product.activation.promotion.v1; policy.product.same-candidate.v1` |
| `capability.product.ai-authoring-mvp-a` | `wp.product.ai-authoring-mvp-a` | `requirement.product.ai-authoring-mvp-a; requirement.product.ai-genre-neutral-authoring; requirement.product.authoring-roundtrip; requirement.product.mvp-completion` | `phase.ai-authoring-mvp-a` | `fixture.product.genreless-ai-project; fixture.product.shooter-2d` | `planning.operation_family.authoring_discovery; planning.operation_family.game_system_discovery; planning.operation_family.world_discovery; planning.operation_family.gameplay_definition_authoring; planning.operation_family.asset_authoring; planning.operation_family.authoring_changeset_execution; planning.operation_family.build_candidate_test; planning.operation_family.build_device_play_debug_task` | `windows_editor_for_artifact_target` | `all_activation_declared_external_safe` | `risk.product.ai-tool-safety-code-owner` | `policy.product.activation.promotion.v1; policy.product.same-candidate.v1` |
| `capability.product.external-agent` | `wp.product.external-agent` | `requirement.product.external-agent-boundary` | `phase.external-agent` | `fixture.product.external-agent-proposal` | `planning.operation_family.authoring_discovery; planning.operation_family.game_system_discovery; planning.operation_family.world_discovery; planning.operation_family.gameplay_definition_authoring; planning.operation_family.asset_authoring; planning.operation_family.authoring_changeset_execution; planning.operation_family.build_candidate_test; planning.operation_family.build_device_play_debug_task` | `headless_proposal_only` | `proposal_subset_excluding_trusted_internal` | `risk.product.ai-tool-safety-code-owner` | `policy.product.activation.promotion.v1; policy.product.same-candidate.v1` |
| `capability.project.native_module` | `wp.authoring.project-native-module` | `requirement.product.core-pack-independence; requirement.product.project-source-activation` | `phase.external-agent` | `fixture.product.genreless-ai-project` | `planning.operation_family.native_game_module_source; planning.operation_family.project_source_promotion; planning.operation_family.build_candidate_test; planning.operation_family.authoring_changeset_execution` | `windows_editor_for_artifact_target` | `all_activation_declared_external_safe` | `risk.product.ai-tool-safety-code-owner` | `policy.product.activation.promotion.v1; policy.product.same-candidate.v1` |
| `capability.project.shader` | `wp.rendering.project-shader` | `requirement.product.core-pack-independence; requirement.product.project-source-activation` | `phase.external-agent` | `fixture.product.genreless-ai-project` | `planning.operation_family.project_shader_discovery; planning.operation_family.project_source_promotion; planning.operation_family.build_candidate_test; planning.operation_family.authoring_changeset_execution` | `windows_editor_for_artifact_target` | `all_activation_declared_external_safe` | `risk.product.ai-tool-safety-code-owner` | `policy.product.activation.promotion.v1; policy.product.same-candidate.v1` |
| `capability.product.project-source-activation` | `wp.product.project-source-activation` | `requirement.product.core-pack-independence; requirement.product.project-source-activation` | `phase.external-agent` | `fixture.product.genreless-ai-project` | `planning.operation_family.native_game_module_source; planning.operation_family.project_shader_discovery; planning.operation_family.project_source_promotion; planning.operation_family.build_candidate_test; planning.operation_family.build_device_play_debug_task; planning.operation_family.authoring_changeset_execution` | `windows_editor_for_artifact_target` | `all_activation_declared_external_safe` | `risk.product.ai-tool-safety-code-owner` | `policy.product.activation.promotion.v1; policy.product.same-candidate.v1` |
| `capability.product.general_production_2d` | `wp.product.general-coverage-2d` | `requirement.product.core-pack-independence; requirement.product.c2-2d-coverage` | `phase.production-capability` | `fixture.product.genreless-core-2d; fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d` | `planning.operation_family.authoring_discovery; planning.operation_family.game_system_discovery; planning.operation_family.world_discovery; planning.operation_family.gameplay_definition_authoring; planning.operation_family.asset_authoring; planning.operation_family.authoring_changeset_execution; planning.operation_family.feature_authoring; planning.operation_family.camera_authoring; planning.operation_family.material_authoring; planning.operation_family.vfx_authoring; planning.operation_family.environment_authoring; planning.operation_family.input_binding_selection; planning.operation_family.navigation_binding_selection; planning.operation_family.physics_role_selection; planning.operation_family.build_candidate_test; planning.operation_family.build_device_play_debug_task` | `windows_editor_for_artifact_target` | `all_activation_declared_external_safe` | `risk.product.ai-tool-safety-code-owner` | `policy.product.activation.promotion.v1; policy.product.same-candidate.v1` |
| `capability.product.general_production_3d` | `wp.product.general-coverage-3d` | `requirement.product.first-playable-3d` | `phase.production-capability` | `fixture.product.shooter-arena-3d` | `planning.operation_family.authoring_discovery; planning.operation_family.game_system_discovery; planning.operation_family.world_discovery; planning.operation_family.gameplay_definition_authoring; planning.operation_family.asset_authoring; planning.operation_family.authoring_changeset_execution; planning.operation_family.feature_authoring; planning.operation_family.camera_authoring; planning.operation_family.material_authoring; planning.operation_family.vfx_authoring; planning.operation_family.environment_authoring; planning.operation_family.input_binding_selection; planning.operation_family.navigation_binding_selection; planning.operation_family.physics_role_selection; planning.operation_family.rendering_aa_discovery; planning.operation_family.lighting_discovery; planning.operation_family.post_process_discovery; planning.operation_family.math_semantic_authoring; planning.operation_family.lod_authoring; planning.operation_family.build_candidate_test; planning.operation_family.build_device_play_debug_task` | `windows_editor_for_artifact_target` | `all_activation_declared_external_safe` | `risk.product.ai-tool-safety-code-owner` | `policy.product.activation.promotion.v1; policy.product.same-candidate.v1` |

最終列は配列ではなく、`AiCapabilityActivationMatrixV1`の二Fieldを`{state_transition_policy_refs[]; candidate_binding_policy_literal}`の順に表示したsource notationである。前半は`ProductOperationalStatePolicyRegistryV1`へ解決するexact一件のRegistry ref、後半はRegistry lookupを行わない`ProductCandidateBindingPolicyLiteralV1`の固定literalである。compilerは二Fieldへ別々にmaterializeし、順序交換、片方欠落、二値を同じRegistryへ解決する実装、任意の三値目を拒否する。

`capability.product.general_production_3d`の行は必要実行surfaceを先に固定するが、`wp.product.general-coverage-3d`が`deferred`で第二の非Shooter 3D Fixtureを持たないためcurrent qualificationへ使えない。`gate.product.reconsider-c2-3d`が要求するActive Definition migrationで第二fixture、Requirement、WP、Target bindingを同時追加するまで、Shooter fixture一件を汎用3DのEvidenceとして代用しない。

24 planning familyはpositive十行のunionへexact全件接続する。`rendering_aa_discovery`、`lighting_discovery`、`post_process_discovery`、`math_semantic_authoring`、`lod_authoring`のexact五件はGeneral Production 3D行だけが追加要求し、同rowがdeferredの間はProduct bindingが存在してもActivation／Provider公開／Capability昇格を許可しない。これらをMVP-AまたはGeneral 2Dへ暗黙追加せず、逆にC2 3D移行時に説明名だけで省略しない。matrix compilerは24 family集合と全positive rowのfamily unionをset equalityにし、unboundまたはunknown familyをActive Definition compile errorにする。

`deny_only_capability_refs[]`のcurrent exact一件は`capability.product.runtime-generation-boundary`である。同Capabilityと`wp.product.runtime-generation`のpositive `execution_surface_bindings[]`はexact `[]`とし、`fixture.product.runtime-generation-denial`でRuntime packageにOrchestrator、Provider credential、MCP server、Source Worker、Compiler、Signer、write-capable Gatewayが0件であることだけをQualificationする。将来Runtime generationを追加する場合は本行をsilent変更せず、Future Portfolio→Active Definition migration、Threat Model、Target別Operation family、Signer／Service／Receipt closure、positive／negative fixtureを一つの変更として承認する。

`matrix_sha256`はASCII `MIRAKAN_AI_CAPABILITY_ACTIVATION_MATRIX_V1`、matrix ID／version、positive十Capability集合、deny-only一Capability集合、全十rowのself-excluding canonical bytesを`uint32_be` length framingしてSHA-256する。rowは`capability_id`、内部ref集合はNFC UTF-8 byte順でstrict sortする。matrix row集合とpositive集合、row owner WPとCapability Registry owner WP、Requirement／Phase／Fixtureと各Registry、planning family ID／version／hash／work item／candidate ID setをexact比較する。missing／extra row、同じfamilyの部分subset、Targetをhostとartifactの一Fieldへ圧縮、Runtime generationへのpositive binding、unknown FieldをActive Definition compile errorにする。

このRegistryによりTier、WP existence、provided fixtureだけからActivationを推測しない。optional rowも同じEvidence／execution mappingを持つがProduct aggregateには含めない。current 24 planning family／192候補は全て`not_activated`であり、上表のpositive十CapabilityはWP状態にかかわらず現時点で`qualified`になれない。

Snapshot chainは`sequence=0`、`previous_state_snapshot_ref=null`、`applied_change.kind=genesis`から始める。genesisはfinal Bootstrap Approval、D baseline envelope、construction Decision、Task 0 trust provisioning Receiptをexact参照し、A→B→C→D ancestor／hashをread-backする。全`required | optional` Capability bindingのrowを`not_activated, candidate_ref=null, receipt_refs=[]`で生成し、Decision Gate／Risk表に示す初期state、全WPのnull head／sequence 0を持つ。

ただしControl Plane bootstrap ceremonyの外部公開current headはsequence 0ではない。Task 10Bはfinal D envelopeをread-backし、Approval／Signer／Role／Keyのvalidityとcurrent revocationを再検証した後、candidate生成開始時に一度だけ`bootstrap_transaction_time`とcandidate `revocation_snapshot_ref`を採番する。同一atomic transaction内で、(a)その時刻を`created_at`に持つsigned genesis snapshot sequence 0、(b)それをparentとし同時刻を`recorded_at`に持つ`wp.architecture.control-plane`専用lifecycle sequence 1 `declared->complete`、(c) lifecycle wrapperを`applied_change.kind=wp_lifecycle`として適用し同時刻を`created_at`に持つsnapshot sequence 1を順に生成し、全wrapperのsigned-record issued_atとrevocation snapshotを型規則どおり一致させ、current pointerを(c)へ一度だけCASする。sequence 1ではControl Plane headだけがlifecycle sequence 1、他の全WP headはnull／0である。(a)と(b)は監査ancestorとして保存するが単独currentとして公開せず、いずれかの生成／署名／検証に失敗した場合はcurrent pointerを一切作らない。同じcandidateのcrash／idempotent retryは保存済み`bootstrap_transaction_time`、candidate revocation snapshot、canonical bytes、signatureをbyte-exact再利用し、別event timeで再署名しない。一方、各CAS attemptはjournalのactual `publication_time`をfresh取得し、直前にcurrent Root／local／global／Readiness／Trust／Catalog／Future Approval、Product signerのIdentity／Role／assignment／purpose Key／revocation、expected-empty operational parentをread-backする。candidateが束縛したauthority／validity／publication windowと一件でもdriftした場合はcandidateを変更せずterminal abortし、該当Control Plane規則のfresh Authorization／Task 0またはF renewalからnew candidateを生成する。parentがnon-emptyなら同じcandidateの既存publicationをread-backしexact一致時だけ成功回復、別candidateなら競合quarantineとする。通常Critical Pathはsequence 1のread-back後にsequence 2から開始する。sequence 2以降はstate transition／lifecycle／definition migration／same-definition baseline rebindingのいずれでも、next snapshotの`created_at`を適用payloadの`recorded_at`、`revocation_snapshot_ref`を同payload値に固定する。`applied_change`のref／hashは完成した同一wrapperとexact一致し、同じ適用wrapperまたは同じretryから別event time、別candidate revocation snapshot、別canonical bytes、複数のnext snapshotを生成しない。

以後のnext snapshotは`applied_change`でexactly oneの完成signed transition／lifecycle／definition migration／baseline rebinding wrapperをref／hash束縛し、wrapperのparent、subject、from／toまたはmigration／rebinding projectionとsnapshot diffを一対一で再計算する。複数change、missing wrapper、別parent、指定外row差を拒否する。current snapshot wrapper hashへのcompare-and-swapでexactly one次snapshotだけをcurrentにする。同じprevious wrapper hashから複数candidateが生じた場合は一件だけをacceptし、他を`diagnostic.product.operational-state-fork`で拒否する。同じcanonical payloadの再送は再署名せず既にpublish済みのexact wrapperを返し、同payloadを別signature wrapperとして分岐させない。Snapshot logical IDは`state_snapshot_id`を除く`ProductOperationalStateSnapshotPayloadV1`のRFC 8785 JCS SHA-256から`urn:mirakan:product-state:sha256:<lowercase-hex>`として導出する。`signed_record.subject_sha256`はIDを含む完成payloadのJCS hashであり、wrapper、signature、signed recordをID derivationへ含めない。`previous_state_snapshot_ref`、current head、CAS refはすべて署名検証済み完成wrapperの`sha256:<JCS wrapper lowercase-hex>` content refであり、payload logical IDではない。ref解決時にwrapper hash、signature、payload ID derivationをすべて再計算する。

通常のnext snapshotはexactly oneの署名済みtransitionを適用して生成する。Lifecycle transitionだけは`WorkPackageLifecycleRecordV1`自身をtransitionとして用い、別の重複Recordを作らない。

```text
ProductOperationalStateTransitionPayloadV1
  transition_id
  active_product_definition_sha256
  parent_state_snapshot_ref
  parent_state_snapshot_sha256
  change:
    {kind: activation,
     capability_id, target_id, from_state, to_state,
     prior_candidate_ref: null | exact candidate ref,
     next_candidate_ref: null | exact candidate ref,
     prior_receipt_refs[], next_receipt_refs[]}
    | {kind: decision_gate_evaluation,
       gate_id, from_state, to_state,
       prior_evidence_refs[], next_evidence_refs[]}
    | {kind: risk_evaluation,
       risk_id, from_state, to_state,
       prior_evidence_refs[], next_evidence_refs[]}
  state_policy_ref
  authorization_decision_ref: approved ProductOperationalDecisionV1 ref
  requested_by_subject_ref
  recorded_at
  revocation_snapshot_ref

ProductOperationalStateTransitionV1
  payload: ProductOperationalStateTransitionPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=product_operational_state_transition)

ProductOperationalDecisionPayloadV1
  decision_id
  decision_kind
  definition_binding_kind = current_parent | destination
  active_product_definition_sha256
  owner_document_id
  owner_document_sha256
  subject_kind
  subject_ref
  subject_sha256
  evidence_refs[]
  disposition = approved | rejected
  approver_subject_ref
  approval_authority_ref
  issued_at
  valid_until
  revocation_snapshot_ref

ProductOperationalDecisionV1
  payload: ProductOperationalDecisionPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=product_operational_decision)

ProductOperationApprovalSubjectV1
  schema_id = urn:mirakan:schema:product:operation-approval-subject:v1
  payload_kind = product_state_transition | work_package_lifecycle | active_definition_migration | control_plane_baseline_rebinding | future_product_claim_release
  approval_projection

ProductDefinitionRowMigrationManifestPayloadV1
  manifest_id
  source_active_product_definition_sha256
  destination_active_product_definition_sha256
  rows[]:
    {registry_id,
     row_kind = standard | evidence_class | definition_change_class | decision_gate,
     logical_id, migration_kind,
     source_row_sha256, destination_row_sha256}
  generated_at
  revocation_snapshot_ref

ProductDefinitionRowMigrationManifestV1
  payload: ProductDefinitionRowMigrationManifestPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=active_product_definition_row_migration_manifest)

ActiveProductDefinitionMigrationPayloadV1
  migration_id
  source_active_product_definition_sha256
  destination_active_product_definition_sha256
  source_state_snapshot_ref
  source_state_snapshot_sha256
  destination_rebaseline_approval_ref
  destination_rebaseline_approval_sha256
  destination_rebaseline_envelope_ref
  destination_rebaseline_envelope_sha256
  control_plane_rebaseline_transaction_ref
  control_plane_rebaseline_transaction_sha256
  architecture_definition_migration_binding_ref: optional ArchitectureDefinitionMigrationBindingRefV1
  architecture_definition_migration_binding_sha256: optional SHA-256
  authorization_decision_ref: approved ProductOperationalDecisionV1 ref
  row_migration_manifest_ref
  row_migration_manifest_sha256
  future_promotion_inputs[]:
    {source_future_portfolio_definition_sha256,
     source_future_portfolio_approval_ref, source_future_portfolio_approval_sha256,
     future_capability_id,
     promoted_active_ids[]: {registry_id, row_kind, logical_id, migration_kind}}
  state_policy_ref = policy.product.state.definition-migration.v1
  destination_capability_target_rows[]
  destination_decision_gate_evaluations[]
  destination_risk_evaluations[]
  destination_work_package_heads[]
  control_plane_rebaseline_lifecycle_wrapper_ref
  control_plane_rebaseline_lifecycle_wrapper_sha256
  requested_by_subject_ref
  recorded_at
  revocation_snapshot_ref

ActiveProductDefinitionMigrationV1
  payload: ActiveProductDefinitionMigrationPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=active_product_definition_migration)

ControlPlaneBaselineRebindingPayloadV1
  rebinding_id
  source_state_snapshot_ref, source_state_snapshot_sha256
  active_product_definition_sha256
  source_control_plane_baseline_binding: CurrentControlPlaneBaselineBindingV1
  destination_control_plane_baseline_binding: CurrentControlPlaneBaselineBindingV1(kind=rebaseline)
  rebaseline_approval_ref, rebaseline_approval_sha256
  rebaseline_envelope_ref, rebaseline_envelope_sha256
  rebaseline_transaction_ref, rebaseline_transaction_sha256
  control_plane_lifecycle_wrapper_ref, control_plane_lifecycle_wrapper_sha256
  state_policy_ref = policy.product.state.control-plane-rebaseline.v1
  authorization_decision_ref: approved ProductOperationalDecisionV1 ref
  requested_by_subject_ref
  recorded_at
  revocation_snapshot_ref

ControlPlaneBaselineRebindingV1
  payload: ControlPlaneBaselineRebindingPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=control_plane_baseline_rebinding)
```

`transition_id`は同Fieldを除くpayloadのJCS SHA-256から`urn:mirakan:product-state-transition:sha256:<lowercase-hex>`として導出し、wrapper／signatureを含めない。transitionのparentはcurrent signed snapshot wrapperとexact一致し、next snapshotはtagged `change`が指定する一行以外の全row／headをbyte-exactに保持する。activationのprior candidate／Receiptはparent row、next値はnext rowとexact一致する。Decision／Riskのprior／next Evidenceは対応evaluation rowとexact一致し、Receipt content refの重複なしunsigned UTF-8 byte順集合である。logical Gate IDをEvidence refへ入れない。`requested_by_subject_ref`は変更要求主体であり、`authorization_decision_ref`は当該edgeのPolicyが指定するkindのfresh approved `ProductOperationalDecisionV1`である。要求主体やpublisher署名を承認へ読み替えない。

`ProductOperationalDecisionV1`だけを通常Product state／通常WP lifecycle／baseline rebinding／claim releaseの承認Decision型として受理する。`decision_kind`は`activation_owner_approval | deactivation_r4 | decision_gate_owner_approval | risk_monitoring_owner_approval | risk_acceptance_r4 | risk_mitigation_owner_approval | risk_closure_owner_approval | work_package_owner_transition | work_package_defer_release_product | work_package_owner_acceptance | active_definition_migration_architecture_product | control_plane_baseline_rebinding_product | future_claim_release_product`、`subject_kind`は`product_state_transition | work_package_lifecycle | active_definition_migration | control_plane_baseline_rebinding | future_product_claim_release`のclosed enumである。definition migration／same-definition rebinding内のControl Plane lifecycle Recordは、Control Plane Ownerの`ControlPlaneRebaselineApprovalV1`を専用authorization型として使い、Product Decisionに偽装しない。Product snapshotのbinding切替そのものは対応するProduct Decisionを別途必須にする。

Decisionと実行payloadのhash循環を避けるため、`subject_ref`／`subject_sha256`は実行payload wrapperではなく、closed `ProductOperationApprovalSubjectV1`の完成JCS bytesを指す。`approval_projection`は対象payloadから次のFieldだけを除いたobjectとする。state transitionは`transition_id, authorization_decision_ref, recorded_at, revocation_snapshot_ref`、WP lifecycleは`lifecycle_record_id, authorization_record_ref, recorded_at, revocation_snapshot_ref`、definition migrationは`migration_id, authorization_decision_ref, recorded_at, revocation_snapshot_ref`、baseline rebindingは`rebinding_id, authorization_decision_ref, recorded_at, revocation_snapshot_ref`、Future claim releaseは`claim_release_id, authorization_decision_ref, issued_at, valid_until, revocation_snapshot_ref`を除く。それ以外のFieldは階層、null、配列順を含めbyte-equivalentに保持し、追加Field、説明文、将来のnext snapshot、publisher metadataを入れない。手順は(1) unsigned実行payload候補からprojectionを生成、(2) `ProductOperationApprovalSubjectV1`をcontent-address、(3) Decisionを署名、(4) authorization refを実行payloadへ挿入、(5)実行payload IDとpublisher署名を生成、の一方向だけである。Decision検証時は完成実行payloadからprojectionを再生成してsubject ref／hashと一致させる。

`decision_id`は同Fieldを除くpayloadのJCS SHA-256から`urn:mirakan:product-decision:sha256:<lowercase-hex>`として導出する。commit authorizationでは`disposition=approved`、`decision.issued_at <= execution_payload.recorded_at < decision.valid_until`（claim releaseは`execution_payload.issued_at`）、execution payloadが束縛するrevocation snapshot時点でDecision／Signer／Role／assignment／Keyが非revoked、active definition hash、Owner document bytes hash、approval subject、必要Evidenceが一致するときだけ受理する。`rejected`、commit時期限切れ、別subject、別Owner、別definition、commit時stale Evidenceを流用しない。

Definition bindingはkind別に固定する。通常state transition、通常／defer WP lifecycle、baseline rebinding、Future claim releaseは`definition_binding_kind=current_parent`で、execution parent/current operational snapshotのdefinition hashと一致する。baseline rebindingはsource／destination binding内のdefinition hashも同じcurrent hashでなければならない。`active_definition_migration_architecture_product`だけは`definition_binding_kind=destination`で、まだcurrentでなくてもmigration payload／approval projectionが束縛するdestination definition hashと一致し、同projection内のsource snapshot hashがcurrent sourceを指すことを検証する。他kindでdestination、migration kindでsource hashを入れることを拒否する。これによりdestinationをcurrentと偽装せず、かつ移行前にdestination承認を発行できる。

Architecture Owner transferを含む`ActiveProductDefinitionMigrationPayloadV1`では、`architecture_definition_migration_binding_ref`／hashを対で必須にし、hashはresolved Bindingの`binding_content_hash`とexact一致させる。[Governance Migration Proposals](governance-migration-proposals.md#1-definition-migration-binding-candidate)の候補Bindingへexact解決する。Product-only Definition migrationでは両Fieldをcanonical omissionする。Binding→Subjectが持つsource／target active Product Definition hashはpayloadの同名Fieldと、source／target Architecture Inventory、Owner Registry、Contract Set、Foundation Definition Closure、Owner reference migration manifest、Compatibility Change、Consumer Inventory、全Evidence Requirementのpass satisfaction bindingはそれぞれ同じmigration closureと一致しなければならない。Product payloadはBindingを参照するだけでBindingへ自分自身を戻さず、Architecture ChangeSetもProduct wrapperをhash preimageへ戻さない。これによりProduct current pointer切替とArchitecture Owner transferを同じclosureで検証しつつhash cycleを作らない。

後日の通常expiryは、当時正当にcommit済みのimmutable transition／lifecycle／migration chainと署名の履歴整合性を壊さない。Verifierは現在時刻ではなくrecorded／issued時点の上記条件を再現する。ただしrevocationの`effective_at`がcommit時刻以前なら鍵侵害等のretroactive invalidationとしてchainをquarantineし、登録済みrecovery policy以外の新規遷移を停止する。commit後を`effective_at`とするrevocationは過去Recordを監査履歴として維持するが、そのDecisionを新規実行へ再利用できない。Risk acceptance、Future claim release、Capability effective availabilityなどPolicyが継続的権限を要求するprojectionだけはcurrent time／current revocation／current Evidenceで再評価し、失効時にeffective state／claimをfail closedへ戻す。

Decision signerはcurrent `role.product-operational-decision.r4`へのactive assignmentを持つ人間主体でなければならず、assignmentのclosed `decision_scopes[] {owner_document_id, allowed_decision_kinds[]}`にpayloadのOwner／kindがexactに含まれ、Role permissionが`R4_PRODUCT_OPERATIONAL_DECISION`でなければならない。AI、requester、対象実装worker、publisher service、対象Evidence発行者からのindependenceをcurrent Identity／Role／assignment registryで検証する。purpose専用Keyの`allowed_signed_record_purposes`はsingleton `[product_operational_decision]`とし、payloadのapprover／authority／issued_at／revocation snapshotと外側signed recordをexact一致させる。Owner文書IDはcurrent generated Architecture Document Registryへ解決し、`owner_document_sha256`はそのapproved bytesと一致させる。自由なApproval URL、Issue、Markdown文字列、別Decision schemaを代用しない。

Decisionのexpected Ownerはsubjectから次のclosed mappingで導出し、`payload.owner_document_id`／hashとexact一致させる。Activationは`CapabilityTargetActivationBindingRegistryV1`のCapability→owner WP→`WorkPackageRegistryV1.owner_document_id`、WP lifecycleは対象WP rowのOwner、Decision GateはGate rowのOwner、RiskはRisk definition rowのOwnerを使う。Active Definition migrationとsame-definition baseline rebindingのProduct判断は`mirakan.arch.product-plan`を使い、別途`ControlPlaneRebaselineApprovalV1`がArchitecture側を承認する。Future claim releaseはPromotion Manifestが束縛するsource Future closure内の対象Future rowのOwnerを使う。subject kindとDecision kindがこのmappingにない組合せ、caller申告Owner、現在hashだけが一致する別Document、複数Owner候補を拒否する。

Publisherはleast-privilege Service Roleをpurpose別に分離する。Snapshotは`service.product-state.snapshot-publisher`＋`role.product-state.snapshot-publisher`＋purpose `product_operational_state_snapshot`、state transitionは`service.product-state.transition-publisher`＋`role.product-state.transition-publisher`＋purpose `product_operational_state_transition`、WP lifecycleは`service.product-state.wp-lifecycle-publisher`＋`role.product-state.wp-lifecycle-publisher`＋purpose `work_package_lifecycle_transition`、same-definition baseline rebindingは`service.product-state.control-plane-rebinder`＋`role.product-state.control-plane-rebinder`＋purpose `control_plane_baseline_rebinding`を使う。各Role assignment、current revocation snapshot、singleton-purpose non-exportable KeyをServer側で検証し、subject／Role／Key／purpose流用を拒否する。全wrapperの`signed_record.subject_sha256`はIDを含む完成payload JCS hash、`signed_record.revocation_snapshot_ref`はpayloadの同Fieldとexact一致させる。時刻Fieldは型別に、snapshot=`signed_record.issued_at == payload.created_at`、state transition／WP lifecycle／definition migration／baseline rebinding=`issued_at == recorded_at`、Product Decision=`issued_at == payload.issued_at`、row migration manifest=`issued_at == generated_at`とする。存在しないgeneric `payload.issued_at`を推測せず、時刻不一致を拒否する。R4 Owner／Risk／Architecture判断は上記Decision wrapperとして別検証し、publisher ServiceへR4判断権を与えない。requesterまたはOwnerをpublisher signerとして推測しない。

`ProductDefinitionRowMigrationManifestV1`の`migration_kind`は`added | removed | retained`だけである。Registry rowはinlineであり独立content refを発明しない。`rows[]`は`{registry_id,row_kind,logical_id}`のunsigned UTF-8 byte tuple順、重複なしとし、source／destination closureの全row unionとset equalityでなければならない。通常の単一`entries[]` Registryは`row_kind=standard`、`ProductDecisionGateRegistryV1`だけは上記三つの専用kindを使う。validatorは各closureのregistryを`registry_id`と`row_kind`で解決し、rowをlogical IDでexact lookupして`SHA-256(JCS(row))`を再計算する。`added`はsource hash=null／destination hash=non-null、`removed`は逆、`retained`は両方non-nullを必須とし、retainedのbytesが変わる場合も同じlogical IDの旧新hashを明示する。`manifest_id`は同Fieldを除くpayload JCS SHA-256から`urn:mirakan:product-definition-row-migration:sha256:<lowercase-hex>`として導出する。`service.product-state.definition-migrator`／`role.product-state.definition-migrator`のpurpose専用Keyで署名し、source／destination closure、row lookup／hash、sort、ID、署名、current revocationを再計算する。migration payloadのmanifest ref／hashはこの完成signed wrapperを指し、未署名JSON、部分集合、説明文diffを拒否する。

row manifestはProduct Registry rowの完全性を示すだけで、Architecture Owner transfer、Compatibility Consumer Inventory、Owner reference migration manifestを代用しない。Architecture Owner transferを含む場合だけ、それらは`architecture_definition_migration_binding_ref`から別にexact解決する。

Active Product Definition変更は通常transitionを複数回適用してはならず、上記migration wrapper一件をcurrent snapshotへCASする。manifestはsource／destination全Definition rowのadded／removed／retainedをexactly once分類する。現行Migration V1は安全側のfull resetだけを許し、destination Activationは全required／optional bindingを`not_activated, candidate_ref=null, receipt_refs=[]`、Decision／Riskはdefinitionのgenesis evaluation、全通常WP headは`null, sequence=0`へ全面resetする。`wp.architecture.control-plane`だけは完成destination `ControlPlaneRebaselineApprovalV1`、完成destination baseline envelope、完成`ControlPlaneRebaselineTransactionV1`へ束縛し、`policy.product.wp.definition-migration-control-plane-rebaseline.v1`で新しいdestination epoch lifecycle wrapper（previous=null、sequence 1、`declared->complete`）を同じmigrationへ含める。初回Construction Authorization、`ControlPlaneBootstrapApprovalV1`または初回bootstrap policyを再利用しない。source lifecycle／Activation／Gate／Risk Evidence／Receiptはimmutable historyとして保持するがdestination currentへ一件もcarry-forwardせず、Control Plane artifactの再利用可否もrebaseline transaction内でdestination closureへ再検証する。destination snapshotの`previous_state_snapshot_ref`はsource signed wrapper、`sequence=source+1`、`applied_change.kind=active_definition_migration`とし、Definition hashと全state row/headを一つのatomic publicationで切替える。部分carry、複数migration wrapper、Approval前publishを拒否する。

Architecture Owner transferを含む場合は、同じatomic publicationでBindingが参照するArchitecture ChangeSetだけを`applied`へ進め、source Owner revisionをcurrentへ残さない。BindingなしのArchitecture current化、複数Binding、Product migrationと異なるsource／target definition hash、Consumer Inventoryがcomplete／zero verifiedでないCompatibility Change、Evidence Requirementのpass satisfaction binding不足を拒否する。Architecture Owner transferを含まないProduct-only migrationはBindingを持たず、Architecture ChangeSetを進めず、Architecture Consumer InventoryやOwner reference migration manifestを要求しない。

Migration wrapperは`service.product-state.definition-migrator`＋`role.product-state.definition-migrator`＋singleton purpose `active_product_definition_migration`で署名し、`authorization_decision_ref`はkind `active_definition_migration_architecture_product`、subject_kind `active_definition_migration`、同じapproval projectionを持つfresh approved `ProductOperationalDecisionV1`でなければならない。`migration_id`は同Fieldを除くpayloadのJCS SHA-256から`urn:mirakan:active-product-migration:sha256:<lowercase-hex>`として導出する。`signed_record.subject_sha256`はIDを含む完成payload JCS hash、current CAS refは完成signed wrapper JCS hashとし、同payload retryを再署名しない。

`architecture_definition_migration_binding_ref`を持つpayloadは`binding_state=approved`の一件だけを指す。CAS後のread-backで同Binding、Consumer Inventory、Compatibility Change、全Evidence Requirementのpass satisfaction binding、source／target Foundation Definition Closure、Owner reference migration manifestの全refがpayloadと一致し、対応Architecture ChangeSetを`applied`と評価できなければPublicationをrollbackする。Binding record自身の状態を`applied`へ書き換えず、Product-only migrationへBindingを補完しない。

Owner document bytes、Toolchain lock、Local Schema Catalog、Authority Binding Source Catalog、Control Plane artifactの変更がActive Product Definition bytesを変えない場合は、Definition migrationを捏造せず`ControlPlaneBaselineRebindingV1`だけを使う。source snapshotはCAS時のcurrent完成wrapper、payloadとsource／destination bindingの`active_product_definition_sha256`は同一current値、destination bindingは完成Rebaseline Approval／Envelope／Transactionから生成した`kind=rebaseline`でなければならない。Control Planeのcurrent lifecycle headだけを`policy.product.wp.control-plane-rebaseline.v1`の`complete->complete`、same Definition epoch、exact `N+1` Recordへ更新し、Activation全行、Decision／Risk全行、`wp.architecture.control-plane`以外の全WP head、definition hashをparentからbyte-exactに保持する。next snapshotはsource wrapperをprevious、sequence exact `N+1`、`applied_change.kind=control_plane_rebaseline`としてRebinding wrapperを指し、Rebinding wrapperとnext snapshotを一回のexpected-parent CASでpublishする。source drift、Definition hash差、他state変更、CP head未更新／複数更新、初回Bootstrap artifact再利用を拒否する。

Rebindingの`authorization_decision_ref`はkind `control_plane_baseline_rebinding_product`、subject kind `control_plane_baseline_rebinding`、current-parent definition bindingのfresh approved `ProductOperationalDecisionV1`だけを許す。`rebinding_id`は同Fieldを除くpayload JCS SHA-256から`urn:mirakan:control-plane-baseline-rebinding:sha256:<lowercase-hex>`として導出し、上記専用publisherで署名する。Control Plane Rebaseline ApprovalはArchitecture判断、Product Decisionはcurrent Product pointer切替判断、publisherは実行主体であり相互代用しない。

`ProductOperationalStatePolicyRegistryV1`の初期entryは次の7件である。各列はschema Fieldそのものであり、自由記述から値を推測しない。

| state_policy_id | change_kind | allowed_edges[] | authority_role_ref | signed_record_purpose | evidence_policy_refs[] | decision requirements by edge |
|---|---|---|---|---|---|---|
| `policy.product.activation.promotion.v1` | `activation` | `not_activated->candidate_locked; candidate_locked->qualified; qualified->production` | `role.product-state.transition-publisher` | `product_operational_state_transition` | `policy.evidence.contract-ci.v1; policy.evidence.target-device.v1; policy.evidence.release.v1` | 全edge=`activation_owner_approval` |
| `policy.product.activation.deactivation.v1` | `activation` | `production->qualified; qualified->candidate_locked; candidate_locked->not_activated` | `role.product-state.transition-publisher` | `product_operational_state_transition` | `policy.evidence.contract-ci.v1; policy.evidence.target-device.v1; policy.evidence.release.v1` | 全edge=`deactivation_r4` |
| `policy.product.decision-gate.evaluate.v1` | `decision_gate_evaluation` | `blocked->open; open->blocked; open->satisfied; satisfied->open; satisfied->retired` | `role.product-state.transition-publisher` | `product_operational_state_transition` | `policy.evidence.contract-ci.v1; policy.evidence.target-device.v1; policy.evidence.release.v1` | 全edge=`decision_gate_owner_approval` |
| `policy.product.risk.evaluate.v1` | `risk_evaluation` | `open->monitoring; monitoring->open; monitoring->mitigated; open->accepted; monitoring->accepted; mitigated->monitoring; mitigated->closed; accepted->monitoring; accepted->closed` | `role.product-state.transition-publisher` | `product_operational_state_transition` | `policy.evidence.contract-ci.v1; policy.evidence.target-device.v1; policy.evidence.release.v1` | `open->monitoring=risk_monitoring_owner_approval; monitoring->open=risk_monitoring_owner_approval; monitoring->mitigated=risk_mitigation_owner_approval; open->accepted=risk_acceptance_r4; monitoring->accepted=risk_acceptance_r4; mitigated->monitoring=risk_monitoring_owner_approval; mitigated->closed=risk_closure_owner_approval; accepted->monitoring=risk_acceptance_r4; accepted->closed=risk_acceptance_r4` |
| `policy.product.state.genesis.v1` | `genesis` | `genesis_only` | `role.product-state.snapshot-publisher` | `product_operational_state_snapshot` | `[]` | `genesis_only=construction_decision_required` |
| `policy.product.state.control-plane-rebaseline.v1` | `control_plane_rebaseline` | `same_definition_current_binding_to_approved_rebaseline` | `role.product-state.control-plane-rebinder` | `control_plane_baseline_rebinding` | `[]` | `same_definition_current_binding_to_approved_rebaseline=control_plane_baseline_rebinding_product` |
| `policy.product.state.definition-migration.v1` | `active_definition_migration` | `definition_revision_to_approved_revision` | `role.product-state.definition-migrator` | `active_product_definition_migration` | `[]` | `definition_revision_to_approved_revision=active_definition_migration_architecture_product` |

`decision_requirements[]`は`allowed_edges[]`とedge set equalityで、`requirement`は上表のDecision kindまたは`construction_decision_required`だけを受理する。Promotionは`CapabilityTargetActivationBindingRegistryV1`のsame Target／Candidate、Owner WP、exact Gate／Requirement、stateに応じたEvidence policy、fresh Target closure、no revoked evidenceを必須にし、同Registryでdisabledなtransitionを拒否する。Deactivationはsecurity／regression／license／provider／Target／freshness原因、impact、rollback／migrationをDecisionへ閉じ、自動fallbackを禁止する。Decision Gate `satisfied`はtyped definition predicateと全prerequisiteをfresh approved Evidenceで再評価しactionを自動実行しない。Risk `monitoring`は観測計画、`accepted`はR4 Risk Decision、`mitigated`はfresh mitigation Receipt、`closed`はtrigger消滅のfresh Evidenceを必須にする。GenesisはControl Plane §6.1 A／B／C／D、final Bootstrap、construction Decision、全row set equalityを一度だけ検査する。Same-definition Control Plane rebaselineは完成Rebaseline closure、R4 Product Decision、CP lifecycleの一行更新、他state byte equality、single CASを必須にする。Definition migrationはapproved source／destination、destination Rebaseline Approval／Envelope／Transaction、R4 Product Change Decision、全row manifest、全面state reset、Control Plane completion、single CASを必須にする。Architecture Owner transferを含むbranchだけは、さらにapproved Architecture Definition Migration binding、complete Consumer Inventory、approved Compatibility Change、全Evidence Requirementのpass satisfaction binding、Owner reference migration manifestを必須にする。

段階飛越、Target／Candidate差、predicate未成立、unknown state、stale／revoked Evidence、Owner／Role不一致、自由なpolicy ID、Snapshot rowの直接差替えを拒否する。genesisだけはfinal Bootstrap Approvalとoffline governance construction Decisionに閉じた専用`policy.product.state.genesis.v1`で全row set equalityを一度に生成し、通常更新に流用しない。

Activation rowの更新は、`not_activated->candidate_locked`だけが`candidate_ref=null`からnon-null exact Candidateを設定でき、current row `receipt_refs[]`をCandidate lockの現在有効なqualifying closureとして開始する。`candidate_locked->qualified->production`は同じCandidate refをbyte-exactに維持するが、current rowのReceiptは「現在stateを支えるfresh exact closure」へatomic replaceする。旧Receiptを永久unionせず、再Qualificationではexpired／revoked refを新fresh Receiptへ置換できる。transition payloadの`prior_receipt_refs[]`はparent rowと、`next_receipt_refs[]`はnext rowとexact一致し、immutable transition chainが旧新両集合を監査する。隣接降格はlower stateを支えるcurrent closureへ再計算し、`candidate_locked->not_activated`で`candidate_ref=null, receipt_refs=[]`へclearする。別Candidateへの切替は旧Candidateを隣接降格で`not_activated`へ閉じた後、新しいlock transitionとして開始する。`effective_availability`はcurrent rowの`receipt_refs[]`だけを評価し、historical transition Receiptをcurrent合否へ混ぜない。

Risk／Decision Gateの表は可読性のためdefinition列とgenesis評価列を併記するが、generatorは`state`／`evidence refs`をdefinition projectionへ含めずgenesis operational snapshotへ分離する。state／evidence更新は署名済みtransitionと新snapshotで行い、Product DefinitionまたはBootstrap Approvalを更新しない。Definition自体のmembership、predicate、action、Target binding、Phase／WP関係を変える場合だけ新definition revision、二段階Approval、state migrationを必要とする。

Riskの保存stateもread-timeで再評価する。`mitigated`／`closed`は対応するfresh non-revoked mitigation／trigger消滅Evidence、`accepted`はfresh non-revoked kind `risk_acceptance_r4` Decisionがcurrent definition／subjectへ一致する場合だけeffectiveに維持する。根拠がexpired／revoked／input driftなら、current fresh monitoring plan Evidenceがあれば`effective_state=monitoring`、なければ`effective_state=open`とする。保存snapshotを黙って書き換えないが、Release、Activation、WP遷移、Decision Gate、claim releaseは必ずeffective stateを消費し、保存上の`mitigated | accepted | closed`を根拠失効後に許可へ使わない。Risk Ownerはその後、対応する署名済みtransitionで保存stateを追随させる。

Work Packageのcurrent stateは次のsigned append-only chainだけが所有する。

```text
WorkPackageLifecyclePayloadV1
  lifecycle_record_id
  previous_lifecycle_record_ref: null | content-addressed ref
  lifecycle_sequence: integer 1..9007199254740991
  work_package_id
  from_scheduling_state
  to_scheduling_state
  active_product_definition_sha256
  parent_operational_state_snapshot_sha256
  candidate_binding_kind = task_plan | artifact_candidate | baseline_candidate | control_plane_rebaseline_candidate | definition_seed
  candidate_binding_ref
  candidate_binding_hash
  transition_policy_ref
  receipt_refs[]
  blocked_reason_ref: null | exact Diagnostic ref
  authorization_record_ref: approved authorization record ref
  requested_by_subject_ref
  recorded_at
  revocation_snapshot_ref

WorkPackageLifecycleRecordV1
  payload: WorkPackageLifecyclePayloadV1
  signed_record: MirakanSignedRecordV1(purpose=work_package_lifecycle_transition)

ArtifactCandidateBindingV1
  artifact_candidate_id
  origin_task_plan_ref
  origin_task_plan_sha256
  source_revision_ref
  candidate_root_sha256
  contract_set_sha256
  toolchain_lock_sha256
  target_profile_refs[]
  artifact_refs[]

WorkPackageDefinitionSeedBindingV1
  definition_seed_binding_id
  active_product_definition_sha256
  work_package_registry_ref, work_package_registry_sha256
  work_package_id
  work_package_row_sha256

ControlPlaneRebaselineCandidateBindingV1
  candidate_binding_id
  active_product_definition_sha256
  rebaseline_core_ref, rebaseline_core_sha256
  rebaseline_approval_ref, rebaseline_approval_sha256
  rebaseline_envelope_ref, rebaseline_envelope_sha256
  rebaseline_transaction_ref, rebaseline_transaction_sha256
```

`lifecycle_record_id`は同Fieldを除く`WorkPackageLifecyclePayloadV1`のRFC 8785 JCS SHA-256から`urn:mirakan:wp-lifecycle:sha256:<lowercase-hex>`として導出する。`signed_record.subject_sha256`はIDを含む完成payloadのJCS hashであり、wrapper／signatureをIDへ含めない。Signer Role、purpose、`signed_record.issued_at=payload.recorded_at`、revocation snapshotは上記Product state署名規則と一致させる。Lifecycle chain identityは`{active_product_definition_sha256, work_package_id}`のepochである。`lifecycle_sequence`とsnapshot `sequence`は上記safe-integer contractを使い、各Definition epochの最初のLifecycle Recordは1、以後exact `N+1`としoverflowを拒否する。Active Definition migrationではdestination epochを新規開始し、通常WPはnull head／0、Control Planeだけprevious=null／sequence 1のbootstrap Recordとする。別Definition epoch間でprevious refを接続せず、同じsequence値を使ってもdefinition hashが異なるため別chainである。最初のRecordは`previous_lifecycle_record_ref=null`、`from_scheduling_state`をDefinition seedと一致させる。headがnullならcurrent WP stateはDefinition seed、headがnon-nullならverified head payloadの`to_scheduling_state`である。以後はcurrent signed lifecycle wrapper ref、sequence `N`、`to_scheduling_state`をそれぞれprevious ref、`N+1`、次Recordのfromへbyte-exactに引き継ぐ。`previous_lifecycle_record_ref`とsnapshotのlifecycle headは完成signed wrapperのcontent ref、`parent_operational_state_snapshot_sha256`はCAS対象current signed snapshot wrapperのJCS hashであり、payload ID／hashではない。new snapshotの対応headはnew signed lifecycle wrapper ref／sequenceでなければならない。Recordとnext snapshotは同一atomic transactionでpublishする。同じparent／headからの二分岐、sequence gap／duplicate、ID／payload不一致、同じWPに複数current headを拒否する。同一payload retryは再署名せず既存wrapperを返す。`to_scheduling_state=blocked`だけ`blocked_reason_ref`をnon-null registered Diagnosticにし、他stateは`null`とする。

`candidate_binding_kind=task_plan`は`declared -> ready`、`ready -> active`、通常の`ready | active -> blocked`／`blocked -> ready`でcurrent `CriticalPathTaskV1` ref／hashを束縛する。同じWPの`ready->active`、`ready|active->blocked`、`blocked->ready`は直前headと同じtask plan ref／hashをbyte-exactに維持し、別Taskへ切り替える場合は登録済みcancel／supersede policyを追加したActive Definition revisionまで拒否する。`artifact_candidate`は`active -> complete`だけで上記closed `ArtifactCandidateBindingV1`の完成bytesをref／hash束縛し、`origin_task_plan_ref`／hashが直前active headのtask binding、Source revision、Candidate root、Contract set、Toolchain、Target ProfileがTaskとexact一致しなければならない。`artifact_candidate_id`は同Fieldを除くpayloadのJCS SHA-256から`urn:mirakan:artifact-candidate:sha256:<lowercase-hex>`として導出する。`baseline_candidate`はControl Plane専用bootstrap完了Recordだけ、`control_plane_rebaseline_candidate`は同じDefinition内でのControl Plane rebaselineだけに完成`ControlPlaneRebaselineCandidateBindingV1`を束縛する。`definition_seed`は初回`deferred->declared`で完成`WorkPackageDefinitionSeedBindingV1`をref／hash束縛する。同artifactのRegistry ref／hashはcurrent Active Definition closure内のexact `WorkPackageRegistryV1`、WP IDは遷移対象、`work_package_row_sha256=SHA-256(JCS(exact inline row))`でなければならない。`definition_seed_binding_id`は同Fieldを除くJCS hashから`urn:mirakan:work-package-definition-seed:sha256:<lowercase-hex>`、rebaseline candidate IDも同様に`urn:mirakan:control-plane-rebaseline-candidate:sha256:<lowercase-hex>`から導出する。inline rowへの架空content ref、空ref、自由文字列、別Candidateへの置換、origin Taskを持たないartifactを許さない。Active Definition変更時のresetはLifecycle edgeでなく`ActiveProductDefinitionMigrationV1`がatomicに所有する。

`WorkPackageLifecyclePolicyRegistryV1`の初期exact entryは次の5件である。Field値は表から直接生成し、説明文から補完しない。

| transition_policy_id | allowed_edges[] | subject_binding_policy | prerequisite_policy | receipt_policy | authorization requirements by edge | bootstrap_only_work_package_ref |
|---|---|---|---|---|---|---|
| `policy.product.wp.normal.v1` | `declared->ready; ready->active; ready->blocked; active->blocked; blocked->ready; active->complete` | `task_or_artifact` | `normal` | `normal` | `declared->ready=ProductOperationalDecisionV1/work_package_owner_transition; ready->active=ProductOperationalDecisionV1/work_package_owner_transition; ready->blocked=ProductOperationalDecisionV1/work_package_owner_transition; active->blocked=ProductOperationalDecisionV1/work_package_owner_transition; blocked->ready=ProductOperationalDecisionV1/work_package_owner_transition; active->complete=ProductOperationalDecisionV1/work_package_owner_acceptance` | `null` |
| `policy.product.wp.defer-release.v1` | `deferred->declared` | `definition_seed` | `decision_gates` | `no_carry` | `deferred->declared=ProductOperationalDecisionV1/work_package_defer_release_product` | `null` |
| `policy.product.wp.bootstrap-control-plane.v1` | `declared->complete` | `baseline` | `bootstrap` | `bootstrap` | `declared->complete=ControlPlaneConstructionAuthorizationV1/control_plane_construction_authorization` | `wp.architecture.control-plane` |
| `policy.product.wp.control-plane-rebaseline.v1` | `complete->complete` | `baseline` | `same_definition_rebaseline` | `rebaseline` | `complete->complete=ControlPlaneRebaselineApprovalV1/control_plane_rebaseline_approval` | `wp.architecture.control-plane` |
| `policy.product.wp.definition-migration-control-plane-rebaseline.v1` | `declared->complete` | `baseline` | `definition_migration_rebaseline` | `rebaseline` | `declared->complete=ControlPlaneRebaselineApprovalV1/control_plane_rebaseline_approval` | `wp.architecture.control-plane` |

`subject_binding_policy`は`task_or_artifact | definition_seed | baseline`、`prerequisite_policy`は`normal | decision_gates | bootstrap | same_definition_rebaseline | definition_migration_rebaseline`、`receipt_policy`は`normal | no_carry | bootstrap | rebaseline`のclosed enumであり、logical refではない。`authorization_requirements[]`はallowed edgeとset equalityで、各edgeの`authorization_schema`／`authorization_kind`を上表から直接生成する。通常／defer行の`authorization_record_ref`は同じapproval projectionを承認する`ProductOperationalDecisionV1`、初回bootstrap行だけはTask 0～10Bと初回Control Plane completionをscope許可した`ControlPlaneConstructionAuthorizationV1`、二つのrebaseline行だけはsource currentとCore closureを承認した`ControlPlaneRebaselineApprovalV1`である。初回Authorizationは個別lifecycle内容を承認するDecisionではなく、final Bootstrap Approval、baseline candidate、construction Receiptとのclosureでだけ有効なoffline scope authorizationである。`normal` readyは全prerequisite current head=`complete`、approved Owner、current Control Plane approval、qualified task plan、completeはfresh Owner acceptance、Target closure、artifact candidateを必須にする。`defer-release`は全reconsideration gateのread-time effective evaluation=`satisfied`、完成`WorkPackageDefinitionSeedBindingV1`、`receipt_refs=[]`、approved Product Decisionを必須にする。初回`bootstrap`はfinal baseline envelope、Bootstrap Approval、`task-0..task-10a`のpass Construction Receipt exact set、offline `ControlPlaneConstructionAuthorizationV1`へ閉じる。Task 10Bはgenesis／lifecycle／current CASを生成する当事Taskなので、そのReceiptを入力条件へ含めない。atomic publication後にTask 10B pass Receiptを監査記録として発行し、失敗Receiptでも既存currentを巻き戻さずquarantine手順へ進む。`same_definition_rebaseline`はcurrent Control Plane headが`complete`、source／destination Definition hashが同一、完成Rebaseline Core／Approval／Envelope／Transactionと`ControlPlaneRebaselineCandidateBindingV1`が一致する場合だけ`complete->complete`をexact `N+1`で許可する。`definition_migration_rebaseline`は同じ完成artifactをdestination closureへ閉じ、新Definition epochのsequence 1として初回Authorization／Bootstrap Approvalを参照せずsource epoch Receiptをcarryしない。上表外のedge／enum、未登録policy、wrong authorization schema／kind／subject、authorization欠落、completeでtask-plan hashのままを拒否する。`ready`／`complete`等のcurrent値をDefinition表へ書き戻さない。

`scheduling_state`のclosed値は`declared | ready | active | blocked | deferred | complete`である。Definition内の同Fieldはgenesis seedであり、approved definition revision内ではimmutableである。現在stateはcurrent `work_package_lifecycle_heads[]`から導出し、Definition行へwrite-backしない。`defer_reason`と`blocked_reason_ref`は常にFieldを持ち、非該当時は`null`とする。`reconsideration_gate_refs[]`は常に存在し、非deferred時は空配列である。genesis `scheduling_state=deferred`ではnon-empty `defer_reason`と1件以上の`reconsideration_gate_refs[]`、`blocked`ではnon-null `blocked_reason_ref`を必須とする。他stateでこれらを設定した行を拒否する。`reconsideration_gate_refs[]`は`PhaseFixtureBindingRegistryV1.gate_id`または`ProductDecisionGateRegistryV1.gate_id`のexact IDだけを受理する。

`ProductPhaseRegistryV1`の`work_package_refs[]`は、`phase_id`が当該Phaseに一致する全Work Packageの全量列挙である。Phase→Work PackageとWork Package→Phaseの相互参照は双方向で一致させ、片側にのみ現れる参照を拒否する。`WorkPackageRegistryV1`の`provided_fixture_refs[]`は当該Work Packageが実装へ寄与するtyped `ProvidedFixtureRefV1`の列挙であり、Work Package単独の完了gateではない。`kind=product_fixture`だけがPhase／Release evidence候補になり、`kind=component_qualification_fixture`はowner manifestへversion／hash付きで閉じたWP内部Qualification入力である。RequirementとPhase completionをWork Packageへ複写せず、完了Receiptはappend-onlyな`WorkPackageLifecycleRecordV1`だけが所有する。

`PhaseFixtureBindingRegistryV1`の`evaluated_requirement_refs[]`と`target_refs[]`は参照先Fixtureの各集合のsubsetでなければならず、範囲外参照を拒否する。`candidate_binding_policy_ref=policy.product.same-candidate.v1`はProject revision、Candidate root hash、Contract set hash、Toolchain lock、Target Profileを全Receiptで一致させる。`freshness_policy_ref`はEvidence Ownerの`policy.evidence.contract-ci.v1`または`policy.evidence.target-device.v1`をexact参照し、失効、revoked、入力hash不一致のReceiptをPhase exitへ使用しない。物理Device上のplay、cook／package、install／launch、performanceまたはSource artifact使用を含むGateは`policy.evidence.target-device.v1`だけを使う。current OS／driver／device／firmware identityまたはpackage artifact hash／signature／install stateがReceipt発行時からdriftした場合は即時`expired`とし、同じTargetで再実行する。`policy.evidence.contract-ci.v1`のReceiptや別TargetのReceiptを代用しない。

Activationの唯一の保存状態正本はcurrent `ProductOperationalStateSnapshotV1.capability_target_activation_rows[]`の`CapabilityTargetActivationStateV1`である。`CapabilityRegistryV1`の各`target_bindings[]`について、scopeが`required`または`optional`の`{capability_id, target_id}`行をgenesisで生成し、`state=not_activated`、`candidate_ref=null`、`receipt_refs=[]`とする。scopeが`excluded`のTargetにはActivation行を生成せず、選択を拒否する。freshnessをrowへ保存せずEvidence Ownerのcurrent receipt／revocation／expiry／input hashから都度導出する。Capability全体の表示状態はrequired Target行の最小stateから毎回導出するread-only projectionであり、Definition、C2 matrix、Capability表、Product labelへ保存またはwrite-backしない。

`capability.platform.input-core`、`capability.platform.audio-core`、`capability.platform.ui-core`のTarget Activationはportable contractのOwner Receiptだけでは昇格しない。Windows Editor／Desktop行は対応する`wp.platform.input-windows`、`wp.platform.audio-windows`、`wp.platform.ui-windows`の`fixture.product.windows-empty-scene` fresh Receiptを必須とする。Android行は`wp.platform.mobile-io-ui-android`、Apple行は`wp.platform.mobile-io-ui-apple`が同Targetで発行する`fixture.product.mobile-lifecycle`、`fixture.product.shooter-2d`、`fixture.product.shooter-arena-3d`のPhase 7 fresh Receiptを必須とする。Phase 8の`fixture.product.platformer-2d`または`fixture.product.puzzle-dialogue-2d`をPhase 7 Activationの前提にせず、Windows Receiptまたは反対側Mobile TargetのReceiptによる代用を拒否する。

`capability.domain.shooter-2d`と`capability.domain.shooter-3d`はWindows／Android／Appleに別々の`CapabilityTargetActivationStateV1`行を持つ。Phase 3の`gate.product.phase-3-manual-2d`はShooter 2DのWindows行だけ、Phase 6の`gate.product.phase-6-first-playable-3d`はShooter 3DのWindows行だけを昇格候補にする。Phase 7の四つの`*-runtime-2d`／`*-runtime-3d` Gateは対応dimensionとAndroidまたはAppleの一行だけを昇格候補にし、Windows、反対側Mobile Target、別dimensionのReceiptを流用しない。Phase 8 genre GateはC2 coverageを評価するだけでC1 Shooter行を遡及昇格しない。

`decision_gate_evaluations[].state`のclosed値は`blocked | open | satisfied | retired`である。Decision GateはPhase exit Receiptではなく、未登録logical IDを作らずにplanning-only条件を追跡する。`state=satisfied`でも`on_satisfied_action`のChangeSetを自動適用せず、Owner approvalと全Definition validationを別途必須とする。評価state／evidenceはcurrent operational snapshotだけが所有し、`ProductDecisionGateRegistryV1`へ保存しない。

### 11.2 Target Profile registry

| target_id | Owner | profile_version | target_kind | surface_role | qualification_requirement_ref |
|---|---|---:|---|---|---|
| `target.headless.host` | `mirakan.arch.core-architecture` | 1 | `headless_server` | `both` | `requirement.target.headless-determinism` |
| `target.windows.editor` | `mirakan.arch.platform-windows` | 1 | `desktop` | `execution_host` | `requirement.target.windows-editor` |
| `target.windows.desktop` | `mirakan.arch.platform-windows` | 1 | `desktop` | `artifact_runtime` | `requirement.target.windows-package` |
| `target.android.mobile` | `mirakan.arch.platform-android` | 1 | `mobile` | `artifact_runtime` | `requirement.target.android-package` |
| `target.apple.mobile` | `mirakan.arch.platform-apple` | 1 | `mobile` | `artifact_runtime` | `requirement.target.apple-package` |

Profile versionは更新可能なFieldであり、Target IDへ`v1`を埋め込まない。`target_kind`と`surface_role`はID文字列から推測せずRegistry rowのclosed Fieldだけを使う。AI Conformanceのexecution／artifact Target、Product `ExecutionSurfaceBindingV1`はcurrent rowを解決し、execution refは`execution_host | both`、artifact refは`artifact_runtime | both`だけを許す。将来kindまたはroleを変える場合はTarget Profile Registry revisionと全consumer Evidenceを更新し、同ID／versionの意味をin-place変更しない。

Future Dossierはcurrent rowを参照する`current_profile`と、authority-free `proposed_destination_profile`をtagged unionで分離する。後者はFuture Inceptionだけが所有する未登録Target候補であり、本Registryの件数、Target support、Conformance、Build、PackageまたはShippingを変更しない。Future→Active migration時は全proposed候補をdestination Registry rowへ一対一投影し、row migration manifestへ列挙してから全Activationを`not_activated`で開始する。proposal refをcurrent `TargetProfileRefV1`として受理する、現行5 rowへ暗黙追加する、Target kindだけ一致する別Profileへ置換する実装を拒否する。

### 11.3 Requirement、Fixture、Fallback registry

| requirement_id | Owner | verification_kind | failure diagnostic |
|---|---|---|---|
| `requirement.target.headless-determinism` | `mirakan.arch.core-architecture` | `determinism` | `diagnostic.product.headless-determinism-failed` |
| `requirement.foundation.memory-pointer-contract` | `mirakan.arch.memory-pointers` | `pointer_memory_contract_qualification` | `diagnostic.product.memory-pointer-contract-failed` |
| `requirement.target.windows-editor` | `mirakan.arch.platform-windows` | `package_and_launch` | `diagnostic.product.windows-editor-gate-failed` |
| `requirement.target.windows-package` | `mirakan.arch.platform-windows` | `clean_install_offline_run` | `diagnostic.product.windows-package-gate-failed` |
| `requirement.target.android-package` | `mirakan.arch.platform-android` | `store_package_device_run` | `diagnostic.product.android-package-gate-failed` |
| `requirement.target.apple-package` | `mirakan.arch.platform-apple` | `store_package_device_run` | `diagnostic.product.apple-package-gate-failed` |
| `requirement.target.mobile-runtime-2d` | `mirakan.arch.product-plan` | `target_device_playable_e2e` | `diagnostic.product.mobile-runtime-2d-gate-failed` |
| `requirement.target.mobile-runtime-3d` | `mirakan.arch.product-plan` | `target_device_playable_e2e` | `diagnostic.product.mobile-runtime-3d-gate-failed` |
| `requirement.product.authoring-roundtrip-manual` | `mirakan.arch.product-plan` | `manual_e2e` | `diagnostic.product.authoring-roundtrip-manual-failed` |
| `requirement.product.editor-reference-design` | `mirakan.arch.product-plan` | `visual_semantic_accessibility_matrix` | `diagnostic.product.editor-reference-design-incomplete` |
| `requirement.product.authoring-roundtrip` | `mirakan.arch.product-plan` | `ai_e2e` | `diagnostic.product.authoring-roundtrip-failed` |
| `requirement.product.core-pack-independence` | `mirakan.arch.product-plan` | `dependency_and_holdout_matrix` | `diagnostic.product.core-pack-independence-failed` |
| `requirement.product.manual-first-playable-2d` | `mirakan.arch.product-plan` | `manual_playable_e2e` | `diagnostic.product.manual-first-playable-2d-incomplete` |
| `requirement.product.ai-authoring-mvp-a` | `mirakan.arch.product-plan` | `ai_mvp_e2e` | `diagnostic.product.ai-authoring-mvp-a-incomplete` |
| `requirement.product.ai-genre-neutral-authoring` | `mirakan.arch.product-plan` | `ai_holdout_e2e` | `diagnostic.product.ai-genre-neutral-authoring-incomplete` |
| `requirement.product.first-playable-3d` | `mirakan.arch.product-plan` | `playable_e2e` | `diagnostic.product.first-playable-3d-incomplete` |
| `requirement.product.mvp-completion` | `mirakan.arch.product-plan` | `mvp_completion_e2e` | `diagnostic.product.mvp-completion-incomplete` |
| `requirement.product.project-source-activation` | `mirakan.arch.product-plan` | `source_activation_e2e` | `diagnostic.product.project-source-activation-incomplete` |
| `requirement.product.title-to-result` | `mirakan.arch.product-plan` | `playable_e2e` | `diagnostic.product.title-to-result-failed` |
| `requirement.product.save-load-replay` | `mirakan.arch.product-plan` | `state_roundtrip` | `diagnostic.product.save-load-replay-failed` |
| `requirement.product.c2-2d-coverage` | `mirakan.arch.product-plan` | `cross_genre_matrix` | `diagnostic.product.c2-2d-coverage-incomplete` |
| `requirement.product.c2-3d-coverage` | `mirakan.arch.product-plan` | `cross_genre_matrix` | `diagnostic.product.c2-3d-coverage-incomplete` |
| `requirement.product.cpp23-cx2-cross-target` | `mirakan.arch.cpp23-modules` | `cross_target_module_cutover` | `diagnostic.product.cpp23-cx2-cross-target-incomplete` |
| `requirement.product.cpp23-cx3-cross-target` | `mirakan.arch.cpp23-modules` | `cross_target_shipping_language_mode` | `diagnostic.product.cpp23-cx3-cross-target-incomplete` |
| `requirement.product.external-agent-boundary` | `mirakan.arch.ai-security-approval` | `authorization_conformance` | `diagnostic.product.external-agent-boundary-failed` |
| `requirement.product.runtime-generation-boundary` | `mirakan.arch.ai-security-approval` | `threat_model_conformance` | `diagnostic.product.runtime-generation-boundary-failed` |
| `requirement.product.release-headless` | `mirakan.arch.product-plan` | `release_contract_support_rollback` | `diagnostic.product.release-headless-incomplete` |
| `requirement.product.release-windows-editor` | `mirakan.arch.product-plan` | `clean_install_launch_support_rollback_release` | `diagnostic.product.release-windows-editor-incomplete` |
| `requirement.product.release-windows-desktop` | `mirakan.arch.product-plan` | `clean_install_offline_support_rollback_release` | `diagnostic.product.release-windows-desktop-incomplete` |
| `requirement.product.release-android` | `mirakan.arch.product-plan` | `signed_package_device_offline_support_rollback_release` | `diagnostic.product.release-android-incomplete` |
| `requirement.product.release-apple` | `mirakan.arch.product-plan` | `signed_package_device_offline_support_rollback_release` | `diagnostic.product.release-apple-incomplete` |

`requirement.foundation.memory-pointer-contract`は、[Memory／Pointers](../02-foundation/memory-pointers.md)の`PointerContractV1`、`MemoryContractV1`、`PointerMemoryConsumerBindingV1`のdefinition closure、consumer Matrixの正逆参照、live pointer／lease／span／allocator objectの保存・job capture禁止、supported sanitizer lane、hot path fallback 0のbaselineを同一Candidateで検査する。Targetがsanitizerを実行できない場合は、対応するstatic／negative Evidenceとsupported Target laneを別Evidenceとして記録し、未実行laneをpassへ読み替えない。これはPhase 0の計画上のGateであり、MCD、生成物、実装、Activationが既に存在するという主張ではない。

`requirement.product.editor-reference-design`は§5.2の七surface roleと二つの横断Referenceを、排他的なPanel分類や機能Activationにせず、typed style／semantic／Workspace definitionとして閉じる。全interactive Controlとsemantic composite rootはclosed Widget Pattern Registryへ解決し、Patternはvisual、Semantic、UIA、Command投影を同じinstanceへ束縛するが、AI操作権限またはProject writeを所有しない。`fixture.product.windows-empty-scene`では実装済みsurfaceについてLight／Dark／High Contrast、`editor_ui_scale=2.00`、density、state overlay、keyboard／UIA／AI typed commandのmatrixを同一Stable ID／revisionで検証し、未Activation Panelを成功例へ補完しない。同Product fixtureは[Editor UI Design System Catalog §15.8](editor-ui-design-system-catalog.md#158-reference-fixture-manifestevidence-contract)の`EditorReferenceFixtureRegistryV1`候補をcurrent Capability snapshotへ解決したrequired Manifest集合と、各Manifestのpass `VerificationReceiptV1`／Evidence Bundleをexact set equalityで含む。最初のcomponent Manifest `fixture.editor.reference-01@1`は[Editor Panel／Reference Catalog §6.9.1](editor-panel-reference-catalog.md#691-fixtureeditorreference-011-concrete-manifest)の14 Environment Profile、9 scenario、166 coverage entry、七typed expected subjectとbyte equalityでなければならず、同ManifestのComparison Profile Registryは[Editor UI Design System Catalog §15.8.5](editor-ui-design-system-catalog.md#1585-comparison-profile-registry)のexact七entryとbyte equalityでなければならない。同ManifestのBaseline RegistryはCoverage Matrixとexact 166 entryで一致し、各expected subjectはatomic Change Item、domain owner＋independent reviewer二`ReviewReceiptV1`、candidate全件検証、source-head CAS、`PromotionReceiptV1` read-backを通ったcurrent Publicationだけから解決する。未materialized logical ID、Markdown表、characterization Profile、Runner既定値、pending decision、AI説明をReceiptへ読み替えない。component Manifestを`FixtureRegistryV1`の新しいProduct fixture rowへ偽装せず、prohibited Manifest、未Activation surfaceのmissing Evidence、別Candidate／environment／contract setのReceiptをpassへ補完しない。Font／icon lock、combined allocation model、TSF／OLE／UIA virtualization、Widget Pattern、AI semanticとVisual evidence、Source／Text／Diff Reference Contract、Task／Proposal／Review Reference Contract、Cross-Panel、Reference Fixture Manifest、Comparison Profile Registry、Baseline Review／Publication Contractの各blockerは§5.2のOwner closureがなければ失敗とし、screenshotだけ、色だけ、Panel名だけ、表示rowだけをEvidenceにしない。`requirement.product.manual-first-playable-2d`は手動AuthoringだけでTitle→Result、Save／Load／Replay、Windows package／clean install／offline playを完走する。`requirement.product.core-pack-independence`はShooterを含む全Genre Packと全optional Feature Packをuninstallした状態でCore／Editor／AI query／Project SDK／Build／Packageが成立し、Core→Feature／Genre required edgeが0件、Genreなし／zero-character／noncombat／UI-only／headlessのholdoutをschema変更なしで表現できることをdependency graphと実行fixtureの両方で検証する。`gate.product.phase-3-genreless-core`は同じCandidate、Active Product Definition、Foundation Closure、Generic Core baseline、Installed Product compositionを束縛し、`result.kind=pass`のfresh `ArchitectureDependencyConformanceReportV1`と`fixture.product.genreless-core-2d`実行Receiptを両方持つまでpassにしない。Report Refだけ、別Candidate、stale Registry、count-only自己申告をEvidenceへ補完しない。`requirement.product.ai-authoring-mvp-a`はDefinition-firstまたは事前Qualification済みNative／Shader Packを使い、AIと手動編集の往復、typed ChangeSet、Diff、Approval、Commit、Undo、Replay、§5のMVP completion chainを一つのProject historyと同一Candidateで完走する。sub-requirementとして`requirement.product.authoring-roundtrip`と`requirement.product.mvp-completion`をEvidenceへ記録するが、`requirement.product.project-source-activation`をPhase 4 gateへ含めない。`requirement.product.ai-genre-neutral-authoring`はGenre／Character／Combat／Objective／Worldを全てoptionalにしたzero-character UI／logic Projectを、同じAI Operation／Validation／Debug／Build DAGで生成、質問解決、Commit、Replay、Packageできることを要求し、Shooter fixtureの成功を代用しない。

`requirement.product.mvp-completion`は§5の一方向chain全体、すなわちclean environment相当でのcook、package、install／launch、first-run settings、offline play、checkpoint／resume、diagnosis、`SupportBundleV1`取得、data resetを同一Candidate hashで検証する。`requirement.product.project-source-activation`は`capability.project.native_module`と`capability.project.shader`の両方を実際のFirst Playable挙動から使用し、Source、生成Diff、Approval、Target artifact、Qualification Receiptが同一Project revisionへ閉じる場合だけ成功する。`requirement.target.mobile-runtime-2d`と`requirement.target.mobile-runtime-3d`はPhase 7のC1実機Requirementであり、同一CandidateのShooter fixtureをAndroidまたはAppleのpackageからclean install／offline起動し、Target別I/O／UI、E7 integration、Save／Load／Replay、Title→Resultまでを実機で完走する。C2 genre coverageや別TargetのReceiptを前提または代用にしない。

五つの`requirement.product.release-*`は単なるbuild成功ではなく、同一Candidate／Toolchain／Target Profileでのclean build、署名済みpackageまたはheadless distribution、clean install／launch、offline run（適用Target）、Support Bundle、rollback rehearsal、license／SBOM／provenance、Release Approvalを閉じる。Headlessは物理Deviceを要求しないが、exact build host profileとdistributable artifactのreproducibilityを要求する。Windows Editor／Desktop、Android、AppleのReceiptは相互流用せず、`policy.evidence.release.v1`でcurrent signer／package／device／support windowを再評価する。

| fixture_id | Owner | requirement_refs | targets | minimum duration seconds |
|---|---|---|---|---:|
| `fixture.product.headless-contract-smoke` | `mirakan.arch.product-plan` | `requirement.target.headless-determinism` | `target.headless.host` | 60 |
| `fixture.foundation.memory-pointer-contract` | `mirakan.arch.memory-pointers` | `requirement.foundation.memory-pointer-contract` | `target.headless.host` | 300 |
| `fixture.product.authoring-transaction` | `mirakan.arch.project-state` | `requirement.product.authoring-roundtrip-manual` | `target.headless.host` | 300 |
| `fixture.product.windows-empty-scene` | `mirakan.arch.product-plan` | `requirement.target.windows-editor; requirement.target.windows-package; requirement.foundation.memory-pointer-contract; requirement.product.editor-reference-design` | `target.windows.editor; target.windows.desktop` | 300 |
| `fixture.product.genreless-core-2d` | `mirakan.arch.product-plan` | `requirement.product.core-pack-independence` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | 300 |
| `fixture.product.genreless-ai-project` | `mirakan.arch.product-plan` | `requirement.product.core-pack-independence; requirement.product.ai-authoring-mvp-a; requirement.product.ai-genre-neutral-authoring; requirement.product.authoring-roundtrip; requirement.product.mvp-completion; requirement.product.project-source-activation; requirement.product.save-load-replay` | `target.windows.editor; target.windows.desktop` | 300 |
| `fixture.product.shooter-2d` | `mirakan.arch.pack-shooter` | `requirement.target.mobile-runtime-2d; requirement.product.manual-first-playable-2d; requirement.product.ai-authoring-mvp-a; requirement.product.authoring-roundtrip; requirement.product.title-to-result; requirement.product.save-load-replay; requirement.product.mvp-completion; requirement.product.project-source-activation; requirement.product.c2-2d-coverage` | `target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | 300 |
| `fixture.product.platformer-2d` | `mirakan.arch.product-plan` | `requirement.product.c2-2d-coverage; requirement.product.save-load-replay` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | 300 |
| `fixture.product.puzzle-dialogue-2d` | `mirakan.arch.product-plan` | `requirement.product.c2-2d-coverage; requirement.product.save-load-replay` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | 300 |
| `fixture.product.shooter-arena-3d` | `mirakan.arch.pack-shooter` | `requirement.target.mobile-runtime-3d; requirement.product.first-playable-3d; requirement.product.title-to-result; requirement.product.save-load-replay; requirement.product.c2-3d-coverage` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | 300 |
| `fixture.product.external-agent-proposal` | `mirakan.arch.ai-security-approval` | `requirement.product.external-agent-boundary` | `target.headless.host` | 120 |
| `fixture.product.mobile-lifecycle` | `mirakan.arch.platform-mobile-common` | `requirement.target.android-package; requirement.target.apple-package; requirement.foundation.memory-pointer-contract` | `target.android.mobile; target.apple.mobile` | 900 |
| `fixture.product.runtime-generation-denial` | `mirakan.arch.ai-security-approval` | `requirement.product.runtime-generation-boundary` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | 300 |
| `fixture.product.release-closure` | `mirakan.arch.product-plan` | `requirement.product.release-headless; requirement.product.release-windows-editor; requirement.product.release-windows-desktop; requirement.product.release-android; requirement.product.release-apple` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | 3600 |
| `fixture.product.cpp23-cx2-cross-target` | `mirakan.arch.cpp23-modules` | `requirement.product.cpp23-cx2-cross-target` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | 1800 |
| `fixture.product.cpp23-cx3-cross-target` | `mirakan.arch.cpp23-modules` | `requirement.product.cpp23-cx3-cross-target` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | 3600 |

`fixture.product.genreless-core-2d`はGenre／Feature Pack ref exact `[]`、Character／Combat／Objective／Completion exact `[]`のCore-only Projectで、明示Runtime Entry、2D World／Camera、deterministic timer、optional Physics／Animation Port、Save／Replayを検証する。`fixture.product.genreless-ai-project`はGenre Pack ref exact `[]`、zero-character／noncombat、WorldなしUI／logic Runtime Entryを既定branchとし、Phase 4ではprequalified Sourceだけ、Phase 5ではCode-owner付きProject C++ moduleとProject Shaderを追加する別Candidate branchを使う。両fixtureはShooter ID／Profile／Game Flow／Assetを入力、fallback、expected outputへ含めず、Shooter install時とuninstall時のContract set／Product dependency graph差を検査する。Pack dependency lint、UI-only、headless noncombatは`core-pack-independence` fixture matrixの別caseであり、一つの偽World／Characterへ近似変換しない。small-model Pack Discoveryはcurrent Operation setが空のためPhase Gateへ成功扱いで登録せず、`activation.pack.ai_operations.v1`のfuture acceptance fixtureとして`not_activated`を維持する。

`FallbackRegistryV1`はfallback entryと、そのentryが参照するProduct-owned Diagnostic recordを同じProduct root内で閉じる。別の15番目のActive Definition Registryを作らず、次のclosed shapeを本Registryのschemaに含める。

```text
ProductDiagnosticRefV1
  diagnostic_id
  diagnostic_version: positive uint32
  diagnostic_content_hash: SHA-256

ProductDiagnosticRecordV1
  diagnostic_id
  diagnostic_version: positive uint32
  owner_document_id
  severity: blocking | error | warning | info
  category
  message_key
  diagnostic_content_hash: SHA-256

ProductFallbackRecordV1
  fallback_id
  owner_document_id
  preserves_semantics: bool
  failure_diagnostic_ref: ProductDiagnosticRefV1

FallbackRegistryV1
  registry_id
  format_major: 1
  revision: positive safe integer
  entries[1..4096]: ProductFallbackRecordV1
  diagnostic_records[1..4096]: ProductDiagnosticRecordV1
```

`diagnostic_content_hash = SHA-256(ASCII "MIRAKAN_PRODUCT_DIAGNOSTIC_RECORD_V1" || uint32_be(len(RFC 8785 JCS(closed record excluding diagnostic_content_hash))) || RFC 8785 JCS(closed record excluding diagnostic_content_hash))`とする。entryはfallback ID、Diagnosticはdiagnostic ID／version順へstrict sortし、`failure_diagnostic_ref`集合と`diagnostic_records[]`をset equalityにする。同じID／version別hash、ID-only lookup、Owner prefix推測、Contract set `DiagnosticCodeRefV1`への型置換を拒否する。下表は可読表示としてDiagnostic IDだけを示すが、保存値は三Fieldの`ProductDiagnosticRefV1`である。

| fallback_id | Owner | preserves_semantics | failure diagnostic |
|---|---|---:|---|
| `fallback.capability.unavailable` | `mirakan.arch.product-plan` | false | `diagnostic.product.capability-unavailable` |
| `fallback.rendering.environment-core` | `mirakan.arch.rendering-environment-surfaces` | true | `diagnostic.rendering.environment-fallback-selected` |
| `fallback.rendering.ibl-baked` | `mirakan.arch.rendering-environment-surfaces` | true | `diagnostic.rendering.ibl-baked-selected` |
| `fallback.rendering.vfx-cpu` | `mirakan.arch.rendering-vfx-runtime` | true | `diagnostic.rendering.vfx-cpu-selected` |
| `fallback.rendering.vfx-core` | `mirakan.arch.rendering-vfx-runtime` | true | `diagnostic.rendering.vfx-core-selected` |

`diagnostic_records[]`のcurrent exact五recordは次である。全rowは`diagnostic_version=1`で、`diagnostic_content_hash`は上記自己除外式から再計算し、表にhash placeholderを保存しない。

| diagnostic_id | owner_document_id | severity | category | message_key |
|---|---|---|---|---|
| `diagnostic.product.capability-unavailable` | `mirakan.arch.product-plan` | `blocking` | `capability` | `diagnostic.product.capability-unavailable.message` |
| `diagnostic.rendering.environment-fallback-selected` | `mirakan.arch.rendering-environment-surfaces` | `warning` | `rendering.environment` | `diagnostic.rendering.environment-fallback-selected.message` |
| `diagnostic.rendering.ibl-baked-selected` | `mirakan.arch.rendering-environment-surfaces` | `warning` | `rendering.environment` | `diagnostic.rendering.ibl-baked-selected.message` |
| `diagnostic.rendering.vfx-cpu-selected` | `mirakan.arch.rendering-vfx-runtime` | `warning` | `rendering.vfx` | `diagnostic.rendering.vfx-cpu-selected.message` |
| `diagnostic.rendering.vfx-core-selected` | `mirakan.arch.rendering-vfx-runtime` | `warning` | `rendering.vfx` | `diagnostic.rendering.vfx-core-selected.message` |

各Fallback rowのOwnerは解決先Diagnostic recordの`owner_document_id`とbyte equalityにする。上表五ID／version集合、Fallbackの`failure_diagnostic_ref`集合、`diagnostic_records[]`集合はexact set equalityで、missing／extra／duplicate、severity／category／message key／Owner差をActive Product Definition compile failureにする。

`preserves_semantics=false`は代替実装ではなくfail-closedを表す。成功Receiptを発行せず、Capabilityを選択不能として説明する。

[Materials](../06-rendering/materials.md)はrealistic／toon／unlit間の汎用かつ意味保存のdefault fallbackを定義していないため、Material Capabilityは現在`fallback.capability.unavailable`へfail-closedする。将来Material fallbackを追加する場合は、Materials所有のexact fallback record、Diagnostic、意味差、Target範囲、Qualification fixtureと全参照元を同じProduct Definition revisionでatomicに登録し、既存Styleをdefault名だけで置換しない。

Executable ContractsのCapability `failure_modes[]`にある`fallback_id`はFoundation Contract setから本RegistryへのProduct foreign keyであり、MCD refではない。Baseline／Rebaseline verifierは同時に束縛されたexact Foundation Definition ClosureとActive Product Definition Bundleについて、全Capability foreign keyを本Registryへexact一件解決する。`fallback.capability.unavailable`は`preserves_semantics=false`かつ、exact `ProductDiagnosticRefV1`が`ProductDiagnosticRecordV1 {diagnostic_id=diagnostic.product.capability-unavailable, diagnostic_version=1, owner_document_id=mirakan.arch.product-plan, severity=blocking, category=capability, message_key=diagnostic.product.capability-unavailable.message}`のclosed recordへ一件解決し、その`diagnostic_content_hash`が直前の`MIRAKAN_PRODUCT_DIAGNOSTIC_RECORD_V1`式で再計算した32-byte SHA-256 digestと一定時間比較で一致することを必須にする。missing／multiple／別diagnostic／hash不一致／stale Product rootでは該当Capabilityをactivateしない。Foundation rootをProduct rootへ戻すhash edge、Product-owned DiagnosticをContract set Diagnosticへ偽装するedgeを作らない。

### 11.4 Product Phase registry

| order | phase_id | outcome requirements | work packages | exit gates |
|---:|---|---|---|---|
| 0 | `phase.foundation` | `requirement.target.headless-determinism; requirement.foundation.memory-pointer-contract` | `wp.architecture.control-plane; wp.foundation.cpp23-cx0; wp.foundation.math-core; wp.foundation.memory-pointers; wp.runtime.scheduling-core; wp.runtime.ecs-e0; wp.runtime.ecs-e1-storage; wp.runtime.ecs-e2-query-mutation` | `gate.product.phase-0-headless-contract; gate.product.phase-0-memory-pointer-contract` |
| 1 | `phase.headless-authoring` | `requirement.product.authoring-roundtrip-manual` | `wp.runtime.ecs-e3-cook-load; wp.authoring.project-state-headless; wp.authoring.asset-save-headless; wp.authoring.headless-core` | `gate.product.phase-1-authoring-transaction` |
| 2 | `phase.editor-runtime` | `requirement.target.windows-editor; requirement.target.windows-package; requirement.foundation.memory-pointer-contract; requirement.product.editor-reference-design` | `wp.runtime.ecs-e4-game-system; wp.rendering.render-graph-core; wp.rendering.d3d12-backend; wp.platform.input-core; wp.platform.audio-core; wp.platform.ui-core; wp.platform.input-windows; wp.platform.audio-windows; wp.platform.ui-windows; wp.platform.windows-package; wp.product.editor-runtime-windows` | `gate.product.phase-2-windows-empty-scene` |
| 3 | `phase.manual-2d` | `requirement.product.manual-first-playable-2d; requirement.product.core-pack-independence` | `wp.gameplay.core-c1; wp.runtime.timer; wp.rendering.world-2d; wp.rendering.camera-2d; wp.simulation.collision-2d; wp.simulation.physics-2d; wp.simulation.animation-2d; wp.navigation.core; wp.runtime.ecs-e5-2d-integration; wp.gameplay.reusable-features-c1; wp.domain.shooter-2d` | `gate.product.phase-3-manual-2d; gate.product.phase-3-genreless-core` |
| 4 | `phase.ai-authoring-mvp-a` | `requirement.product.ai-authoring-mvp-a; requirement.product.ai-genre-neutral-authoring` | `wp.authoring.ai-core; wp.runtime.debug-replay-support; wp.runtime.ecs-e6-debug-ai; wp.authoring.prequalified-source-packs; wp.runtime.ecs-e7-windows-2d; wp.product.ai-authoring-mvp-a` | `gate.product.phase-4-ai-mvp-a; gate.product.phase-4-ai-genreless` |
| 5 | `phase.external-agent` | `requirement.product.external-agent-boundary; requirement.product.project-source-activation` | `wp.product.external-agent; wp.authoring.project-native-module; wp.rendering.project-shader; wp.product.project-source-activation` | `gate.product.phase-5-external-agent; gate.product.phase-5-project-source-activation` |
| 6 | `phase.manual-3d-mvp-b` | `requirement.product.first-playable-3d` | `wp.rendering.world-3d; wp.rendering.camera-3d; wp.simulation.collision-3d; wp.simulation.physics-3d; wp.simulation.animation-3d; wp.runtime.ecs-e5-3d-integration; wp.runtime.ecs-e7-windows-3d; wp.domain.shooter-3d` | `gate.product.phase-6-first-playable-3d` |
| 7 | `phase.mobile` | `requirement.target.android-package; requirement.target.apple-package; requirement.foundation.memory-pointer-contract; requirement.target.mobile-runtime-2d; requirement.target.mobile-runtime-3d` | `wp.rendering.vulkan-backend; wp.rendering.metal-backend; wp.platform.mobile-offline; wp.platform.mobile-io-ui-android; wp.platform.mobile-io-ui-apple; wp.runtime.ecs-e7-android-2d; wp.runtime.ecs-e7-apple-2d; wp.runtime.ecs-e7-android-3d; wp.runtime.ecs-e7-apple-3d; wp.platform.android-package; wp.platform.apple-package` | `gate.product.phase-7-android-lifecycle; gate.product.phase-7-apple-lifecycle; gate.product.phase-7-android-runtime-2d; gate.product.phase-7-apple-runtime-2d; gate.product.phase-7-android-runtime-3d; gate.product.phase-7-apple-runtime-3d` |
| 8 | `phase.production-capability` | `requirement.product.c2-2d-coverage` | `wp.foundation.cpp23-cx2-cutover; wp.foundation.cpp23-cx3-shipping; wp.domain.platformer; wp.domain.puzzle-dialogue; wp.rendering.environment-c2; wp.rendering.vfx-c2; wp.rendering.material-realistic; wp.rendering.material-toon; wp.ui.native-widget; wp.product.general-coverage-2d; wp.product.general-coverage-3d; wp.product.production-release-binding` | `gate.product.phase-8-c2-shooter-2d; gate.product.phase-8-c2-platformer-2d; gate.product.phase-8-c2-puzzle-dialogue-2d` |
| 9 | `phase.runtime-generation` | `requirement.product.runtime-generation-boundary` | `wp.product.runtime-generation` | `gate.product.phase-9-runtime-generation-denial` |

Phase 5とPhase 9はこのRegistry行を唯一のscheduling identityとし、本文上の見出しや番号だけで存在を表現しない。Phase 5の`gate.product.phase-5-external-agent`はProposal-only authorizationだけを評価し、Native／Shader Sourceを作成またはactivateしない。`gate.product.phase-5-project-source-activation`はCode owner付き新規Source laneだけを評価し、external Client接続の成功を代用しない。Phase 8の三Genre gateは`capability.product.general_production_2d`とbundled Reference Game coverageだけを評価する独立trackで、Generic Engine Release、CX3 shipping、production-release bindingをblockしない。Phase 9はdeny-only C0 boundaryであり、Phase 8 coverageの完了またはC2 Product Capabilityへ依存せず、許可されたpositive Runtime structured-data generationは`future.capability.runtime-structured-data-generation`の`planning_only`境界に留める。

Phase exitのRequirement、Target、Candidate binding、Freshnessは次のbinding表だけが決定する。fixtureの全Requirement／Targetを暗黙評価せず、同じfixtureを別Phaseで使っても別gateのReceiptを代用しない。

| gate_id | phase_id | fixture_id | evaluated requirement refs | target refs | candidate binding policy | freshness policy |
|---|---|---|---|---|---|---|
| `gate.product.phase-0-headless-contract` | `phase.foundation` | `fixture.product.headless-contract-smoke` | `requirement.target.headless-determinism` | `target.headless.host` | `policy.product.same-candidate.v1` | `policy.evidence.contract-ci.v1` |
| `gate.product.phase-0-memory-pointer-contract` | `phase.foundation` | `fixture.foundation.memory-pointer-contract` | `requirement.foundation.memory-pointer-contract` | `target.headless.host` | `policy.product.same-candidate.v1` | `policy.evidence.contract-ci.v1` |
| `gate.product.phase-1-authoring-transaction` | `phase.headless-authoring` | `fixture.product.authoring-transaction` | `requirement.product.authoring-roundtrip-manual` | `target.headless.host` | `policy.product.same-candidate.v1` | `policy.evidence.contract-ci.v1` |
| `gate.product.phase-2-windows-empty-scene` | `phase.editor-runtime` | `fixture.product.windows-empty-scene` | `requirement.target.windows-editor; requirement.target.windows-package; requirement.foundation.memory-pointer-contract; requirement.product.editor-reference-design` | `target.windows.editor; target.windows.desktop` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-3-manual-2d` | `phase.manual-2d` | `fixture.product.shooter-2d` | `requirement.product.manual-first-playable-2d; requirement.product.title-to-result; requirement.product.save-load-replay` | `target.windows.desktop` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-3-genreless-core` | `phase.manual-2d` | `fixture.product.genreless-core-2d` | `requirement.product.core-pack-independence` | `target.windows.desktop` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-4-ai-mvp-a` | `phase.ai-authoring-mvp-a` | `fixture.product.shooter-2d` | `requirement.product.ai-authoring-mvp-a; requirement.product.authoring-roundtrip; requirement.product.mvp-completion; requirement.product.title-to-result; requirement.product.save-load-replay` | `target.windows.editor; target.windows.desktop` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-4-ai-genreless` | `phase.ai-authoring-mvp-a` | `fixture.product.genreless-ai-project` | `requirement.product.core-pack-independence; requirement.product.ai-authoring-mvp-a; requirement.product.ai-genre-neutral-authoring; requirement.product.authoring-roundtrip; requirement.product.mvp-completion; requirement.product.save-load-replay` | `target.windows.editor; target.windows.desktop` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-5-external-agent` | `phase.external-agent` | `fixture.product.external-agent-proposal` | `requirement.product.external-agent-boundary` | `target.headless.host` | `policy.product.same-candidate.v1` | `policy.evidence.contract-ci.v1` |
| `gate.product.phase-5-project-source-activation` | `phase.external-agent` | `fixture.product.genreless-ai-project` | `requirement.product.core-pack-independence; requirement.product.project-source-activation` | `target.windows.editor; target.windows.desktop` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-6-first-playable-3d` | `phase.manual-3d-mvp-b` | `fixture.product.shooter-arena-3d` | `requirement.product.first-playable-3d; requirement.product.title-to-result; requirement.product.save-load-replay` | `target.windows.desktop` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-7-android-lifecycle` | `phase.mobile` | `fixture.product.mobile-lifecycle` | `requirement.target.android-package; requirement.foundation.memory-pointer-contract` | `target.android.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-7-apple-lifecycle` | `phase.mobile` | `fixture.product.mobile-lifecycle` | `requirement.target.apple-package; requirement.foundation.memory-pointer-contract` | `target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-7-android-runtime-2d` | `phase.mobile` | `fixture.product.shooter-2d` | `requirement.target.mobile-runtime-2d` | `target.android.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-7-apple-runtime-2d` | `phase.mobile` | `fixture.product.shooter-2d` | `requirement.target.mobile-runtime-2d` | `target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-7-android-runtime-3d` | `phase.mobile` | `fixture.product.shooter-arena-3d` | `requirement.target.mobile-runtime-3d` | `target.android.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-7-apple-runtime-3d` | `phase.mobile` | `fixture.product.shooter-arena-3d` | `requirement.target.mobile-runtime-3d` | `target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-8-c2-shooter-2d` | `phase.production-capability` | `fixture.product.shooter-2d` | `requirement.product.c2-2d-coverage` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-8-c2-platformer-2d` | `phase.production-capability` | `fixture.product.platformer-2d` | `requirement.product.c2-2d-coverage` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-8-c2-puzzle-dialogue-2d` | `phase.production-capability` | `fixture.product.puzzle-dialogue-2d` | `requirement.product.c2-2d-coverage` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-8-cpp23-cx2-cross-target` | `phase.production-capability` | `fixture.product.cpp23-cx2-cross-target` | `requirement.product.cpp23-cx2-cross-target` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-8-cpp23-cx3-cross-target` | `phase.production-capability` | `fixture.product.cpp23-cx3-cross-target` | `requirement.product.cpp23-cx3-cross-target` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-9-runtime-generation-denial` | `phase.runtime-generation` | `fixture.product.runtime-generation-denial` | `requirement.product.runtime-generation-boundary` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |

`gate.product.phase-8-cpp23-cx2-cross-target`と`gate.product.phase-8-cpp23-cx3-cross-target`はCapability qualification専用bindingであり、`ProductPhaseRegistryV1.exit_gate_refs[]`には含めない。Phase bindingがPhase exitから未参照でも、`CapabilityTargetActivationBindingRegistryV1.qualification_gate_refs[]`からexactly one以上参照される上記2 IDだけはorphanではない。その他の未参照binding、またはPhase exitとCapability qualificationのどちらからも参照されないbindingを拒否する。この二GateによりCX2／CX3の5 Target計10行は`unavailable_no_gate`にならず、対応WP完了後に同一Candidate・全Target fresh Receiptを得た場合だけ`qualified`へ進める。C2 3Dの3行は§11.9の承認済み第二fixtureがまだないため意図的に`unavailable_no_gate`を維持する。

`ProductReleaseGateRegistryV1`はPhase exitと別であり、`ProductPhaseRegistryV1.exit_gate_refs[]`へ含めない。現行definitionにも次の5 definition rowを登録するが、Activation bindingのproduction policyはdisabledなので評価結果をproduction遷移へ使えない。Release-binding migration後だけTarget一致rowからexact一件を参照する。

| release_gate_id | target_id | fixture_id | evaluated requirement refs | required phase gates | required WP states | candidate binding | freshness | critical risk policy |
|---|---|---|---|---|---|---|---|---|
| `gate.product.release-headless` | `target.headless.host` | `fixture.product.release-closure` | `requirement.product.release-headless` | `gate.product.phase-0-headless-contract; gate.product.phase-1-authoring-transaction; gate.product.phase-5-external-agent; gate.product.phase-9-runtime-generation-denial` | `wp.foundation.cpp23-cx3-shipping=complete; wp.product.production-release-binding=complete` | `policy.product.same-candidate.v1` | `policy.evidence.release.v1` | `all_impact_critical_effective_mitigated_or_closed` |
| `gate.product.release-windows-editor` | `target.windows.editor` | `fixture.product.release-closure` | `requirement.product.release-windows-editor` | `gate.product.phase-2-windows-empty-scene; gate.product.phase-4-ai-genreless; gate.product.phase-5-external-agent; gate.product.phase-5-project-source-activation; gate.product.phase-9-runtime-generation-denial` | `wp.foundation.cpp23-cx3-shipping=complete; wp.platform.windows-package=complete; wp.product.production-release-binding=complete` | `policy.product.same-candidate.v1` | `policy.evidence.release.v1` | `all_impact_critical_effective_mitigated_or_closed` |
| `gate.product.release-windows-desktop` | `target.windows.desktop` | `fixture.product.release-closure` | `requirement.product.release-windows-desktop` | `gate.product.phase-2-windows-empty-scene; gate.product.phase-3-genreless-core; gate.product.phase-4-ai-genreless; gate.product.phase-5-project-source-activation; gate.product.phase-9-runtime-generation-denial` | `wp.foundation.cpp23-cx3-shipping=complete; wp.platform.windows-package=complete; wp.product.production-release-binding=complete` | `policy.product.same-candidate.v1` | `policy.evidence.release.v1` | `all_impact_critical_effective_mitigated_or_closed` |
| `gate.product.release-android` | `target.android.mobile` | `fixture.product.release-closure` | `requirement.product.release-android` | `gate.product.phase-7-android-lifecycle; gate.product.phase-9-runtime-generation-denial` | `wp.foundation.cpp23-cx3-shipping=complete; wp.platform.android-package=complete; wp.product.production-release-binding=complete` | `policy.product.same-candidate.v1` | `policy.evidence.release.v1` | `all_impact_critical_effective_mitigated_or_closed` |
| `gate.product.release-apple` | `target.apple.mobile` | `fixture.product.release-closure` | `requirement.product.release-apple` | `gate.product.phase-7-apple-lifecycle; gate.product.phase-9-runtime-generation-denial` | `wp.foundation.cpp23-cx3-shipping=complete; wp.platform.apple-package=complete; wp.product.production-release-binding=complete` | `policy.product.same-candidate.v1` | `policy.evidence.release.v1` | `all_impact_critical_effective_mitigated_or_closed` |

各Release Gateは参照Phase Gateのread-time fresh success、WP current state、fixture requirement、Target、same Candidate、release freshness、critical Risk closureを`all_of`で毎回再評価する。Generic Engine Releaseが参照できるPhase Gateはheadless／empty-scene、Genre／Feature Pack未導入の`phase-3-genreless-core`／`phase-4-ai-genreless`、external／Source境界、Target lifecycle、deny-only Phase 9のsubsetだけである。`phase-3-manual-2d`、`phase-4-ai-mvp-a`、`phase-6-first-playable-3d`、Phase 7のShooter fixtureによるruntime gate、Phase 8の三Genre coverage gate、および`wp.product.general-coverage-2d`をRelease predicateへ追加するDefinitionを拒否する。これらはbundled Reference Game／genre coverageの非blocking consumerだけが参照できる。closed predicate `all_impact_critical_effective_mitigated_or_closed`はcurrent `ProductRiskDefinitionRegistryV1`で`impact=critical`の全risk ID集合と評価対象をset equalityにし、各current effective stateが`mitigated | closed`の場合だけ成功する。`open | monitoring | accepted`、missing／extra Risk、expired／revoked Evidence、別Candidateならeffective failであり、保存ActivationがproductionでもRelease／claimを停止する。Phase 9のdeny-only security Gateは全5 Release Gateのglobal prerequisiteであり、未実行・失敗時にTarget別Gateを通さない。Release Gate自身はstateを保存せずEvidenceから投影する。

Planning Decision GateはPhase exit bindingと別Registryであり、`ProductPhaseRegistryV1.exit_gate_refs[]`へ含めない。

| gate_id | Owner | evaluator | required phase gates | required WP states | required evidence classes | required definition-change classes | genesis state | typed on-satisfied action |
|---|---|---|---|---|---|---|---|---|
| `gate.product.reconsider-cpp23-cx2-cutover` | `mirakan.arch.cpp23-modules` | `all_of` | `gate.product.phase-0-headless-contract; gate.product.phase-2-windows-empty-scene; gate.product.phase-7-android-lifecycle; gate.product.phase-7-apple-lifecycle` | `[]` | `evidence.class.cpp23-nonexperimental-import-std; evidence.class.cpp23-cross-target-cutover; evidence.class.cpp23-module-dag-valid; evidence.class.toolchain-lock-exact` | `[]` | `blocked` | `{kind=permit_work_package_transition, work_package_id=wp.foundation.cpp23-cx2-cutover, from_state=deferred, to_state=declared, transition_policy_ref=policy.product.wp.defer-release.v1}` |
| `gate.product.reconsider-cpp23-cx3-shipping` | `mirakan.arch.cpp23-modules` | `all_of` | `gate.product.phase-0-headless-contract; gate.product.phase-2-windows-empty-scene; gate.product.phase-3-genreless-core; gate.product.phase-4-ai-genreless; gate.product.phase-7-android-lifecycle; gate.product.phase-7-apple-lifecycle` | `{work_package_id=wp.foundation.cpp23-cx2-cutover, required_state=complete}` | `evidence.class.cpp23-formal-language-mode; evidence.class.cpp23-nonexperimental-import-std; evidence.class.cpp23-cross-target-release-readiness; evidence.class.package-install-offline-rollback-qualification` | `[]` | `blocked` | `{kind=permit_work_package_transition, work_package_id=wp.foundation.cpp23-cx3-shipping, from_state=deferred, to_state=declared, transition_policy_ref=policy.product.wp.defer-release.v1}` |
| `gate.product.reconsider-c2-3d` | `mirakan.arch.product-plan` | `all_of` | `gate.product.phase-6-first-playable-3d` | `[]` | `evidence.class.active-definition-validation-zero-error` | `definition.change.c2-3d-second-fixture-closure` | `blocked` | `{kind=permit_work_package_transition, work_package_id=wp.product.general-coverage-3d, from_state=deferred, to_state=declared, transition_policy_ref=policy.product.wp.defer-release.v1}` |
| `gate.product.reconsider-production-release-binding` | `mirakan.arch.product-plan` | `all_of` | `gate.product.phase-0-headless-contract; gate.product.phase-2-windows-empty-scene; gate.product.phase-3-genreless-core; gate.product.phase-4-ai-genreless; gate.product.phase-7-android-lifecycle; gate.product.phase-7-apple-lifecycle` | `{work_package_id=wp.foundation.cpp23-cx3-shipping, required_state=complete}` | `evidence.class.product-release-policy-ready; evidence.class.product-release-artifact-plan-valid` | `[]` | `blocked` | `{kind=permit_work_package_transition, work_package_id=wp.product.production-release-binding, from_state=deferred, to_state=declared, transition_policy_ref=policy.product.wp.defer-release.v1}` |

`evidence_class_definitions[]`の初期exact rowsは次の10件である。全行のwrapperは`TechnicalQualificationReceiptV1`、purposeは`technical_qualification_receipt`で、`subject_policy_ref`は同名classのprefixを`evidence.class.`から`policy.subject.`へ置換したclosed IDとする。subject policyはclass ID、Owner、same Candidate、current active definition、current Toolchain、下表Target exact set、class固有入力を持つcanonical closureのJCS hashを要求する。

| class_id | Owner | freshness policy | required Targets | class固有subject入力 |
|---|---|---|---|---|
| `evidence.class.cpp23-nonexperimental-import-std` | `mirakan.arch.cpp23-modules` | `policy.evidence.contract-ci.v1` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | formal compiler mode、`import std` probe、compiler artifact |
| `evidence.class.cpp23-cross-target-cutover` | `mirakan.arch.cpp23-modules` | `policy.evidence.contract-ci.v1` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | module compile／link／load fixture set |
| `evidence.class.cpp23-module-dag-valid` | `mirakan.arch.cpp23-modules` | `policy.evidence.contract-ci.v1` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | generated module DAG、cycle diagnostics |
| `evidence.class.toolchain-lock-exact` | `mirakan.arch.toolchain-dependencies` | `policy.evidence.contract-ci.v1` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | five-profile lock closure、artifact read-back |
| `evidence.class.cpp23-formal-language-mode` | `mirakan.arch.cpp23-modules` | `policy.evidence.contract-ci.v1` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | non-preview language flag／frontend identity |
| `evidence.class.cpp23-cross-target-release-readiness` | `mirakan.arch.cpp23-modules` | `policy.evidence.contract-ci.v1` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | release configuration compile／link plan and probes |
| `evidence.class.package-install-offline-rollback-qualification` | `mirakan.arch.ai-verification-provenance` | `policy.evidence.target-device.v1` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | package、install、offline launch、rollback fixtures |
| `evidence.class.active-definition-validation-zero-error` | `mirakan.arch.product-plan` | `policy.evidence.contract-ci.v1` | `target.headless.host` | 14-registry closure validator report |
| `evidence.class.product-release-policy-ready` | `mirakan.arch.product-plan` | `policy.evidence.contract-ci.v1` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | approved support／rollback／signing／SBOM policy |
| `evidence.class.product-release-artifact-plan-valid` | `mirakan.arch.ai-verification-provenance` | `policy.evidence.contract-ci.v1` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | artifact／Target lab／store staging plan |

`definition_change_class_definitions[]`は一件だけで、`definition.change.c2-3d-second-fixture-closure`をOwner `mirakan.arch.product-plan`、wrapper `ActiveProductDefinitionMigrationV1`、purpose `active_product_definition_migration`、subject policy `policy.subject.c2-3d-second-fixture-closure`へ束縛する。subject policyは下記第二Fixture closure、row manifest、Rebaseline closure、destination snapshot CASをexact検証する。definitionsのclass ID集合は全Gate `required_*_classes[]`のunionとset equalityにし、missing／extra／duplicate class、wrong wrapper／purpose／Owner／subject policy／freshness／Targetを拒否する。

`ProductDecisionGateRegistryV1`の三配列はrow migration時に一つのtagged logical-row unionとして扱う。`evidence_class_definitions[]`は`row_kind=evidence_class`かつ`logical_id`が`evidence.class.` prefix、`definition_change_class_definitions[]`は`row_kind=definition_change_class`かつ`definition.change.` prefix、`entries[]`は`row_kind=decision_gate`かつ`gate.product.` prefixでなければならない。三集合のlogical IDはRegistry全体でglobal unique、相互disjointであり、`ProductDefinitionRowMigrationManifestV1`は各行を`{registry_id,row_kind,logical_id}`でexactly once列挙する。prefixだけからrow内容を合成せず、wrong kind、二配列重複、manifestのkind欠落を拒否する。

`evaluator_policy`のclosed値は現在`all_of`だけである。`required_work_package_states[].required_state`は`complete`だけ、`required_evidence_classes[]`は上表の`evidence.class.*` 10値、`required_definition_change_classes[]`は`definition.change.c2-3d-second-fixture-closure`だけを受理する。Evidence classは自由文ではなく、classごとにOwnerが発行した完成signed Evidence content refの集合へ解決し、全refが同一Candidate、必要Target closure、current Toolchain lock、current active definition hashへ閉じ、freshかつ非revokedでなければならない。`cpp23-cross-target-release-readiness`と`package-install-offline-rollback-qualification`はCX3 WPを開始できるToolchain／candidate package前提であり、CX3 WP自身が後で発行するRelease Receiptを要求しない。`product-release-policy-ready`はProduct Plan Ownerがcompleted CX3、Genre／Feature Pack未導入のCore／AI holdout Gate、Target package Ownerのcurrent Receiptを入力に独立R4承認したsupport window、rollback、signing／SBOM／provenance policy Evidence、`product-release-artifact-plan-valid`はAI Verification／Provenance Release Evidence Ownerが同じprerequisite closureを入力に承認したartifact／Target lab／store staging plan Evidenceである。同OwnerはWindows、Android、Apple各Ownerのfresh Target-specific Receiptをexact set equalityで集約してEvidence classを発行するだけで、Target package schema、signing、upload、rollback、Store policyを所有または上書きしない。`wp.product.production-release-binding`、そのTask、またはCandidateはこのEvidenceを自己発行できない。General 2D／3D、Shooter／Platformer／Puzzle-Dialogue等のReference Game Receiptを両Evidenceの必須入力にしない。両Evidenceは`wp.product.production-release-binding`のdeferred→declaredより前に完成し、同WPまたはそのTask／candidateをproducer／subjectにせず、同WPはref／hashをread-backしてdestination projectionへ組み込むだけである。不足時はGateを`blocked`のまま保つ。実Release Receiptはdestination WP実行後の`fixture.product.release-closure`が発行する。`definition.change.c2-3d-second-fixture-closure`は承認済み`ActiveProductDefinitionMigrationV1`のrow migration manifestが、第二の非Shooter 3D Fixture、`requirement.product.c2-3d-coverage` binding、Genre／Rendering／UI provider WP、Owner、Windows／Android／Apple Target bindingを同一destination closureへ追加し、definition validatorがmissing／extra／duplicate／orphan／cycleを0件とした場合だけ成立する。

Decision評価時は、参照Phase Gateのcurrent fresh success、WP lifecycle headのexact state、Evidence class、definition-change classを`all_of`で毎回再評価する。保存stateが`satisfied`でも、いずれかのEvidenceがexpired／revoked／input hash不一致、Phase Gateが失効、WP stateが不一致になった時点でread-time `effective_state=blocked`とし、defer-releaseへ使わない。`on_satisfied_action`は許可候補を表すだけで自動遷移ではない。exact action一致、current `effective_state=satisfied`、`policy.product.wp.defer-release.v1`、独立Ownerのfresh `ProductOperationalDecisionV1`を揃えた`WorkPackageLifecycleRecordV1`だけが遷移を行い、Activation行の作成または昇格は一切行わない。自然文のpredicateまたはactionを実行入力に使うことを禁止する。

### 11.5 Work Package registry

表の`scheduling_state`は`wp.foundation.cpp23-cx2-cutover`、`wp.foundation.cpp23-cx3-shipping`、`wp.product.general-coverage-3d`、`wp.product.production-release-binding`の4件を`deferred`とし、それ以外を`declared`とする。計画書の存在を`ready`、`active`、`complete`へ読み替えず、deferred WPはPhase exitの完了条件へ含めない。

| work_package_id | phase_id | Owner | targets | fallback | provided fixtures | required capabilities | requires WP | scheduling state | defer_reason | reconsideration gates | blocked reason |
|---|---|---|---|---|---|---|---|---|---|---|---|
| `wp.architecture.control-plane` | `phase.foundation` | `mirakan.arch.core-architecture` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.headless-contract-smoke; fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `[]` | `[]` | `declared` | `null` | `[]` | `null` |
| `wp.foundation.cpp23-cx0` | `phase.foundation` | `mirakan.arch.toolchain-dependencies` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.headless-contract-smoke; fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.foundation.control-plane` | `wp.architecture.control-plane` | `declared` | `null` | `[]` | `null` |
| `wp.foundation.math-core` | `phase.foundation` | `mirakan.arch.math-core` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.headless-contract-smoke; fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.foundation.cpp23-cx0` | `wp.foundation.cpp23-cx0` | `declared` | `null` | `[]` | `null` |
| `wp.foundation.memory-pointers` | `phase.foundation` | `mirakan.arch.memory-pointers` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.foundation.memory-pointer-contract; fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.foundation.math-core` | `wp.foundation.math-core` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.scheduling-core` | `phase.foundation` | `mirakan.arch.runtime-scheduling-lifetime` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.headless-contract-smoke; fixture.foundation.memory-pointer-contract; fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.foundation.memory-pointers` | `wp.foundation.memory-pointers` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e0` | `phase.foundation` | `mirakan.arch.runtime-scheduling-lifetime` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.headless-contract-smoke; fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.runtime.scheduling` | `wp.runtime.scheduling-core` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e1-storage` | `phase.foundation` | `mirakan.arch.runtime-scheduling-lifetime` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.headless-contract-smoke; fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.runtime.ecs-e0-contract` | `wp.runtime.ecs-e0` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e2-query-mutation` | `phase.foundation` | `mirakan.arch.runtime-scheduling-lifetime` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.headless-contract-smoke; fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.runtime.ecs-e1-storage` | `wp.runtime.ecs-e1-storage` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e3-cook-load` | `phase.headless-authoring` | `mirakan.arch.runtime-scheduling-lifetime` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.authoring-transaction; fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.runtime.ecs-e2-query-mutation` | `wp.runtime.ecs-e2-query-mutation` | `declared` | `null` | `[]` | `null` |
| `wp.authoring.project-state-headless` | `phase.headless-authoring` | `mirakan.arch.project-state` | `target.headless.host; target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.authoring-transaction; fixture.product.windows-empty-scene` | `capability.runtime.ecs-e3-cook-load` | `wp.runtime.ecs-e3-cook-load` | `declared` | `null` | `[]` | `null` |
| `wp.authoring.asset-save-headless` | `phase.headless-authoring` | `mirakan.arch.asset-lifecycle` | `target.headless.host; target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.authoring-transaction; fixture.product.windows-empty-scene` | `capability.authoring.project-state-headless` | `wp.authoring.project-state-headless` | `declared` | `null` | `[]` | `null` |
| `wp.authoring.headless-core` | `phase.headless-authoring` | `mirakan.arch.project-state` | `target.headless.host; target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.authoring-transaction; fixture.product.windows-empty-scene` | `capability.authoring.asset-save-headless` | `wp.authoring.asset-save-headless` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e4-game-system` | `phase.editor-runtime` | `mirakan.arch.runtime-scheduling-lifetime` | `target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.runtime.ecs-e3-cook-load` | `wp.runtime.ecs-e3-cook-load; wp.authoring.headless-core` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.render-graph-core` | `phase.editor-runtime` | `mirakan.arch.rendering-render-graph` | `target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.foundation.cpp23-cx0; capability.runtime.ecs-e0-contract` | `wp.foundation.cpp23-cx0; wp.runtime.ecs-e0` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.d3d12-backend` | `phase.editor-runtime` | `mirakan.arch.rendering-render-graph` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene` | `capability.rendering.render-graph-core` | `wp.rendering.render-graph-core` | `declared` | `null` | `[]` | `null` |
| `wp.platform.input-core` | `phase.editor-runtime` | `mirakan.arch.platform-input` | `target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.runtime.ecs-e4-game-system` | `wp.runtime.ecs-e4-game-system` | `declared` | `null` | `[]` | `null` |
| `wp.platform.audio-core` | `phase.editor-runtime` | `mirakan.arch.platform-audio` | `target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.runtime.ecs-e4-game-system` | `wp.runtime.ecs-e4-game-system` | `declared` | `null` | `[]` | `null` |
| `wp.platform.ui-core` | `phase.editor-runtime` | `mirakan.arch.platform-ui-text-localization-accessibility` | `target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.platform.input-core; capability.platform.audio-core` | `wp.platform.input-core; wp.platform.audio-core` | `declared` | `null` | `[]` | `null` |
| `wp.platform.input-windows` | `phase.editor-runtime` | `mirakan.arch.platform-input` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene` | `capability.platform.input-core` | `wp.platform.input-core` | `declared` | `null` | `[]` | `null` |
| `wp.platform.audio-windows` | `phase.editor-runtime` | `mirakan.arch.platform-audio` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene` | `capability.platform.audio-core` | `wp.platform.audio-core` | `declared` | `null` | `[]` | `null` |
| `wp.platform.ui-windows` | `phase.editor-runtime` | `mirakan.arch.platform-ui-text-localization-accessibility` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene` | `capability.platform.ui-core` | `wp.platform.ui-core` | `declared` | `null` | `[]` | `null` |
| `wp.platform.windows-package` | `phase.editor-runtime` | `mirakan.arch.platform-windows` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene` | `capability.rendering.d3d12-cx0` | `wp.rendering.d3d12-backend; wp.authoring.headless-core` | `declared` | `null` | `[]` | `null` |
| `wp.product.editor-runtime-windows` | `phase.editor-runtime` | `mirakan.arch.product-plan` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene` | `capability.runtime.ecs-e4-game-system; capability.rendering.d3d12-cx0; capability.platform.input-core; capability.platform.audio-core; capability.platform.ui-core; capability.platform.windows-package` | `wp.runtime.ecs-e4-game-system; wp.rendering.d3d12-backend; wp.platform.input-windows; wp.platform.audio-windows; wp.platform.ui-windows; wp.platform.windows-package` | `declared` | `null` | `[]` | `null` |
| `wp.gameplay.core-c1` | `phase.manual-2d` | `mirakan.arch.gameplay-programming-model` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.genreless-core-2d; fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d; fixture.product.shooter-arena-3d` | `capability.runtime.ecs-e4-game-system` | `wp.runtime.ecs-e4-game-system` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.timer` | `phase.manual-2d` | `mirakan.arch.runtime-scheduling-lifetime` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.genreless-core-2d; fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d` | `capability.runtime.scheduling` | `wp.runtime.scheduling-core` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.world-2d` | `phase.manual-2d` | `mirakan.arch.rendering-world` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.genreless-core-2d; fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d` | `capability.rendering.render-graph-core` | `wp.rendering.render-graph-core` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.camera-2d` | `phase.manual-2d` | `mirakan.arch.rendering-camera` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.genreless-core-2d; fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d` | `capability.world.2d` | `wp.rendering.world-2d` | `declared` | `null` | `[]` | `null` |
| `wp.simulation.collision-2d` | `phase.manual-2d` | `mirakan.arch.simulation-collision` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.genreless-core-2d; fixture.product.shooter-2d; fixture.product.platformer-2d` | `capability.runtime.ecs-e4-game-system` | `wp.runtime.ecs-e4-game-system` | `declared` | `null` | `[]` | `null` |
| `wp.simulation.physics-2d` | `phase.manual-2d` | `mirakan.arch.simulation-physics` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.genreless-core-2d; fixture.product.shooter-2d; fixture.product.platformer-2d` | `capability.simulation.collision-2d` | `wp.simulation.collision-2d` | `declared` | `null` | `[]` | `null` |
| `wp.simulation.animation-2d` | `phase.manual-2d` | `mirakan.arch.simulation-animation` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.genreless-core-2d; fixture.product.shooter-2d; fixture.product.platformer-2d` | `capability.runtime.timer` | `wp.runtime.timer` | `declared` | `null` | `[]` | `null` |
| `wp.navigation.core` | `phase.manual-2d` | `mirakan.arch.simulation-navigation` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.genreless-core-2d; fixture.feature.path_following.executor_stub` | `capability.runtime.ecs-e4-game-system` | `wp.runtime.ecs-e4-game-system` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e5-2d-integration` | `phase.manual-2d` | `mirakan.arch.runtime-scheduling-lifetime` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.genreless-core-2d; fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d; fixture.product.mobile-lifecycle` | `capability.world.2d; capability.camera.2d; capability.simulation.physics-2d; capability.simulation.animation-2d` | `wp.rendering.world-2d; wp.rendering.camera-2d; wp.simulation.physics-2d; wp.simulation.animation-2d` | `declared` | `null` | `[]` | `null` |
| `wp.gameplay.reusable-features-c1` | `phase.manual-2d` | `mirakan.arch.pack-gameplay-features` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.feature.combat.contract; fixture.feature.ranged_combat.contract; fixture.feature.encounter_spawn.contract; fixture.feature.scoring.contract; fixture.feature.pickup_grant.provider_neutral; fixture.feature.interaction.contract; fixture.feature.character_locomotion.motion_executor; fixture.feature.path_following.executor_stub; fixture.feature.scenario_stage.none` | `capability.runtime.timer; capability.simulation.navigation` | `wp.gameplay.core-c1; wp.runtime.timer; wp.navigation.core` | `declared` | `null` | `[]` | `null` |
| `wp.domain.shooter-2d` | `phase.manual-2d` | `mirakan.arch.pack-shooter` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-2d` | `capability.gameplay.combat; capability.gameplay.ranged_combat; capability.gameplay.encounter_spawn; capability.gameplay.scoring; capability.gameplay.pickup_grant; capability.gameplay.interaction; capability.gameplay.character_locomotion; capability.gameplay.path_following; capability.gameplay.scenario_stage; capability.gameplay.perception; capability.runtime.ecs-e5-2d-integration; capability.camera.2d` | `wp.gameplay.reusable-features-c1; wp.runtime.ecs-e5-2d-integration; wp.rendering.camera-2d` | `declared` | `null` | `[]` | `null` |
| `wp.authoring.ai-core` | `phase.ai-authoring-mvp-a` | `mirakan.arch.ai-security-approval` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.genreless-ai-project; fixture.product.shooter-2d` | `capability.authoring.manual-roundtrip` | `wp.authoring.headless-core` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.debug-replay-support` | `phase.ai-authoring-mvp-a` | `mirakan.arch.runtime-debugging-observability-replay` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.genreless-ai-project; fixture.product.shooter-2d` | `capability.runtime.scheduling; capability.authoring.manual-roundtrip` | `wp.runtime.scheduling-core; wp.authoring.headless-core` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e6-debug-ai` | `phase.ai-authoring-mvp-a` | `mirakan.arch.runtime-scheduling-lifetime` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.genreless-ai-project; fixture.product.shooter-2d` | `capability.runtime.ecs-e4-game-system; capability.authoring.ai-core; capability.runtime.debug-replay-support` | `wp.runtime.ecs-e4-game-system; wp.authoring.ai-core; wp.runtime.debug-replay-support` | `declared` | `null` | `[]` | `null` |
| `wp.authoring.prequalified-source-packs` | `phase.ai-authoring-mvp-a` | `mirakan.arch.native-game-module` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.genreless-ai-project; fixture.product.shooter-2d` | `capability.authoring.ai-core; capability.runtime.ecs-e6-debug-ai` | `wp.authoring.ai-core; wp.runtime.ecs-e6-debug-ai` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e7-windows-2d` | `phase.ai-authoring-mvp-a` | `mirakan.arch.runtime-scheduling-lifetime` | `target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.genreless-core-2d; fixture.product.shooter-2d` | `capability.runtime.ecs-e5-2d-integration; capability.runtime.ecs-e6-debug-ai; capability.authoring.prequalified-source-packs` | `wp.runtime.ecs-e5-2d-integration; wp.runtime.ecs-e6-debug-ai; wp.authoring.prequalified-source-packs` | `declared` | `null` | `[]` | `null` |
| `wp.product.ai-authoring-mvp-a` | `phase.ai-authoring-mvp-a` | `mirakan.arch.product-plan` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.genreless-ai-project; fixture.product.shooter-2d` | `capability.authoring.ai-core; capability.runtime.debug-replay-support; capability.authoring.prequalified-source-packs; capability.platform.windows-package` | `wp.authoring.ai-core; wp.runtime.debug-replay-support; wp.authoring.prequalified-source-packs; wp.platform.windows-package` | `declared` | `null` | `[]` | `null` |
| `wp.product.external-agent` | `phase.external-agent` | `mirakan.arch.ai-security-approval` | `target.headless.host` | `fallback.capability.unavailable` | `fixture.product.external-agent-proposal` | `capability.foundation.control-plane` | `wp.product.ai-authoring-mvp-a` | `declared` | `null` | `[]` | `null` |
| `wp.authoring.project-native-module` | `phase.external-agent` | `mirakan.arch.native-game-module` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.genreless-ai-project` | `capability.authoring.ai-core; capability.runtime.ecs-e6-debug-ai` | `wp.authoring.ai-core; wp.runtime.ecs-e6-debug-ai` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.project-shader` | `phase.external-agent` | `mirakan.arch.rendering-project-shader` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.genreless-ai-project` | `capability.authoring.ai-core; capability.runtime.ecs-e6-debug-ai; capability.rendering.render-graph-core` | `wp.authoring.ai-core; wp.runtime.ecs-e6-debug-ai; wp.rendering.render-graph-core` | `declared` | `null` | `[]` | `null` |
| `wp.product.project-source-activation` | `phase.external-agent` | `mirakan.arch.product-plan` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.genreless-ai-project` | `capability.product.ai-authoring-mvp-a; capability.project.native_module; capability.project.shader` | `wp.product.ai-authoring-mvp-a; wp.authoring.project-native-module; wp.rendering.project-shader` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.world-3d` | `phase.manual-3d-mvp-b` | `mirakan.arch.rendering-world` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.rendering.render-graph-core` | `wp.rendering.render-graph-core` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.camera-3d` | `phase.manual-3d-mvp-b` | `mirakan.arch.rendering-camera` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.world.3d` | `wp.rendering.world-3d` | `declared` | `null` | `[]` | `null` |
| `wp.simulation.collision-3d` | `phase.manual-3d-mvp-b` | `mirakan.arch.simulation-collision` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.runtime.ecs-e4-game-system` | `wp.runtime.ecs-e4-game-system` | `declared` | `null` | `[]` | `null` |
| `wp.simulation.physics-3d` | `phase.manual-3d-mvp-b` | `mirakan.arch.simulation-physics` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.simulation.collision-3d` | `wp.simulation.collision-3d` | `declared` | `null` | `[]` | `null` |
| `wp.simulation.animation-3d` | `phase.manual-3d-mvp-b` | `mirakan.arch.simulation-animation` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.simulation.physics-3d; capability.runtime.timer` | `wp.simulation.physics-3d; wp.runtime.timer` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e5-3d-integration` | `phase.manual-3d-mvp-b` | `mirakan.arch.runtime-scheduling-lifetime` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d; fixture.product.mobile-lifecycle` | `capability.world.3d; capability.camera.3d; capability.simulation.physics-3d; capability.simulation.animation-3d` | `wp.rendering.world-3d; wp.rendering.camera-3d; wp.simulation.physics-3d; wp.simulation.animation-3d` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e7-windows-3d` | `phase.manual-3d-mvp-b` | `mirakan.arch.runtime-scheduling-lifetime` | `target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.runtime.ecs-e5-3d-integration; capability.runtime.ecs-e6-debug-ai` | `wp.runtime.ecs-e5-3d-integration; wp.runtime.ecs-e6-debug-ai` | `declared` | `null` | `[]` | `null` |
| `wp.domain.shooter-3d` | `phase.manual-3d-mvp-b` | `mirakan.arch.pack-shooter` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.gameplay.combat; capability.gameplay.ranged_combat; capability.gameplay.encounter_spawn; capability.gameplay.scoring; capability.gameplay.pickup_grant; capability.gameplay.interaction; capability.gameplay.character_locomotion; capability.gameplay.path_following; capability.gameplay.scenario_stage; capability.gameplay.perception; capability.runtime.ecs-e5-3d-integration; capability.camera.3d` | `wp.gameplay.reusable-features-c1; wp.runtime.ecs-e5-3d-integration; wp.rendering.camera-3d` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.vulkan-backend` | `phase.mobile` | `mirakan.arch.rendering-render-graph` | `target.android.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle` | `capability.rendering.render-graph-core` | `wp.rendering.render-graph-core` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.metal-backend` | `phase.mobile` | `mirakan.arch.rendering-render-graph` | `target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle` | `capability.rendering.render-graph-core` | `wp.rendering.render-graph-core` | `declared` | `null` | `[]` | `null` |
| `wp.platform.mobile-offline` | `phase.mobile` | `mirakan.arch.platform-mobile-common` | `target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle` | `capability.runtime.ecs-e4-game-system` | `wp.runtime.ecs-e4-game-system` | `declared` | `null` | `[]` | `null` |
| `wp.platform.mobile-io-ui-android` | `phase.mobile` | `mirakan.arch.platform-mobile-common` | `target.android.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle; fixture.product.shooter-2d; fixture.product.shooter-arena-3d` | `capability.platform.input-core; capability.platform.audio-core; capability.platform.ui-core` | `wp.platform.input-core; wp.platform.audio-core; wp.platform.ui-core` | `declared` | `null` | `[]` | `null` |
| `wp.platform.mobile-io-ui-apple` | `phase.mobile` | `mirakan.arch.platform-mobile-common` | `target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle; fixture.product.shooter-2d; fixture.product.shooter-arena-3d` | `capability.platform.input-core; capability.platform.audio-core; capability.platform.ui-core` | `wp.platform.input-core; wp.platform.audio-core; wp.platform.ui-core` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e7-android-2d` | `phase.mobile` | `mirakan.arch.runtime-scheduling-lifetime` | `target.android.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle; fixture.product.genreless-core-2d; fixture.product.shooter-2d` | `capability.runtime.ecs-e5-2d-integration; capability.rendering.vulkan-backend; capability.platform.mobile-io-ui-android` | `wp.runtime.ecs-e5-2d-integration; wp.rendering.vulkan-backend; wp.platform.mobile-io-ui-android` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e7-apple-2d` | `phase.mobile` | `mirakan.arch.runtime-scheduling-lifetime` | `target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle; fixture.product.genreless-core-2d; fixture.product.shooter-2d` | `capability.runtime.ecs-e5-2d-integration; capability.rendering.metal-backend; capability.platform.mobile-io-ui-apple` | `wp.runtime.ecs-e5-2d-integration; wp.rendering.metal-backend; wp.platform.mobile-io-ui-apple` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e7-android-3d` | `phase.mobile` | `mirakan.arch.runtime-scheduling-lifetime` | `target.android.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle; fixture.product.shooter-arena-3d` | `capability.runtime.ecs-e5-3d-integration; capability.rendering.vulkan-backend; capability.platform.mobile-lifecycle; capability.platform.mobile-io-ui-android` | `wp.runtime.ecs-e5-3d-integration; wp.rendering.vulkan-backend; wp.platform.mobile-offline; wp.platform.mobile-io-ui-android` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e7-apple-3d` | `phase.mobile` | `mirakan.arch.runtime-scheduling-lifetime` | `target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle; fixture.product.shooter-arena-3d` | `capability.runtime.ecs-e5-3d-integration; capability.rendering.metal-backend; capability.platform.mobile-lifecycle; capability.platform.mobile-io-ui-apple` | `wp.runtime.ecs-e5-3d-integration; wp.rendering.metal-backend; wp.platform.mobile-offline; wp.platform.mobile-io-ui-apple` | `declared` | `null` | `[]` | `null` |
| `wp.platform.android-package` | `phase.mobile` | `mirakan.arch.platform-android` | `target.android.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle; fixture.product.shooter-2d; fixture.product.shooter-arena-3d` | `capability.rendering.vulkan-backend; capability.runtime.ecs-e7-android-2d; capability.runtime.ecs-e7-android-3d; capability.platform.mobile-lifecycle` | `wp.rendering.vulkan-backend; wp.runtime.ecs-e7-android-2d; wp.runtime.ecs-e7-android-3d; wp.platform.mobile-offline` | `declared` | `null` | `[]` | `null` |
| `wp.platform.apple-package` | `phase.mobile` | `mirakan.arch.platform-apple` | `target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle; fixture.product.shooter-2d; fixture.product.shooter-arena-3d` | `capability.rendering.metal-backend; capability.runtime.ecs-e7-apple-2d; capability.runtime.ecs-e7-apple-3d; capability.platform.mobile-lifecycle` | `wp.rendering.metal-backend; wp.runtime.ecs-e7-apple-2d; wp.runtime.ecs-e7-apple-3d; wp.platform.mobile-offline` | `declared` | `null` | `[]` | `null` |
| `wp.foundation.cpp23-cx2-cutover` | `phase.production-capability` | `mirakan.arch.cpp23-modules` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.cpp23-cx2-cross-target; fixture.product.headless-contract-smoke; fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.foundation.cpp23-cx0` | `wp.foundation.cpp23-cx0; wp.platform.windows-package; wp.platform.android-package; wp.platform.apple-package` | `deferred` | `CMake 4.4のimport stdがExperimental opt-inかつNinja系限定で、全Target Cutover／Tooling／ABI Receiptが未成立のためCX0 Headerを維持する` | `gate.product.reconsider-cpp23-cx2-cutover` | `null` |
| `wp.foundation.cpp23-cx3-shipping` | `phase.production-capability` | `mirakan.arch.cpp23-modules` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.cpp23-cx3-cross-target; fixture.product.headless-contract-smoke; fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.foundation.cpp23-cx2-cutover` | `wp.foundation.cpp23-cx2-cutover` | `deferred` | `MSVC 14.51はpreviewのC++23だけであり、正式C++23 language mode、非Experimental CMake、全Target Package／Release Receiptが未成立のためRelease Activation不可` | `gate.product.reconsider-cpp23-cx3-shipping` | `null` |
| `wp.domain.platformer` | `phase.production-capability` | `mirakan.arch.gameplay-programming-model` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.platformer-2d` | `capability.gameplay.interaction; capability.runtime.timer; capability.gameplay.path_following` | `wp.gameplay.reusable-features-c1; wp.runtime.timer; wp.runtime.ecs-e7-windows-2d; wp.runtime.ecs-e7-android-2d; wp.runtime.ecs-e7-apple-2d` | `declared` | `null` | `[]` | `null` |
| `wp.domain.puzzle-dialogue` | `phase.production-capability` | `mirakan.arch.gameplay-programming-model` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.puzzle-dialogue-2d` | `capability.gameplay.interaction; capability.runtime.timer; capability.platform.ui-core` | `wp.gameplay.reusable-features-c1; wp.runtime.timer; wp.platform.ui-windows; wp.runtime.ecs-e7-windows-2d; wp.runtime.ecs-e7-android-2d; wp.runtime.ecs-e7-apple-2d` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.environment-c2` | `phase.production-capability` | `mirakan.arch.rendering-environment-surfaces` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.rendering.environment-core` | `fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d; fixture.product.shooter-arena-3d` | `capability.rendering.render-graph-core` | `wp.rendering.render-graph-core; wp.platform.android-package; wp.platform.apple-package` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.vfx-c2` | `phase.production-capability` | `mirakan.arch.rendering-vfx-runtime` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.rendering.vfx-core` | `fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d; fixture.product.shooter-arena-3d` | `capability.rendering.render-graph-core` | `wp.rendering.render-graph-core; wp.platform.android-package; wp.platform.apple-package` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.material-realistic` | `phase.production-capability` | `mirakan.arch.rendering-materials` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d; fixture.product.shooter-arena-3d` | `capability.rendering.render-graph-core` | `wp.rendering.render-graph-core; wp.platform.android-package; wp.platform.apple-package` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.material-toon` | `phase.production-capability` | `mirakan.arch.rendering-materials` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d; fixture.product.shooter-arena-3d` | `capability.rendering.render-graph-core` | `wp.rendering.render-graph-core; wp.platform.android-package; wp.platform.apple-package` | `declared` | `null` | `[]` | `null` |
| `wp.ui.native-widget` | `phase.production-capability` | `mirakan.arch.platform-ui-text-localization-accessibility` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d; fixture.product.shooter-arena-3d` | `capability.platform.ui-core` | `wp.platform.ui-windows; wp.platform.mobile-io-ui-android; wp.platform.mobile-io-ui-apple` | `declared` | `null` | `[]` | `null` |
| `wp.product.general-coverage-2d` | `phase.production-capability` | `mirakan.arch.product-plan` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.genreless-core-2d; fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d` | `capability.domain.platformer; capability.domain.puzzle-dialogue; capability.environment.core; capability.vfx.system; capability.render.material.realistic_advanced; capability.render.material.toon; capability.ui.native_widget; capability.gameplay.interaction; capability.runtime.timer; capability.gameplay.path_following` | `wp.domain.platformer; wp.domain.puzzle-dialogue; wp.rendering.environment-c2; wp.rendering.vfx-c2; wp.rendering.material-realistic; wp.rendering.material-toon; wp.ui.native-widget; wp.gameplay.reusable-features-c1; wp.runtime.timer` | `declared` | `null` | `[]` | `null` |
| `wp.product.general-coverage-3d` | `phase.production-capability` | `mirakan.arch.product-plan` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.environment.core; capability.vfx.system; capability.render.material.realistic_advanced; capability.render.material.toon; capability.ui.native_widget; capability.gameplay.interaction; capability.runtime.timer; capability.gameplay.path_following` | `wp.rendering.environment-c2; wp.rendering.vfx-c2; wp.rendering.material-realistic; wp.rendering.material-toon; wp.ui.native-widget; wp.gameplay.reusable-features-c1; wp.runtime.timer` | `deferred` | `第二の非Shooter 3D Fixture、C2 Requirement binding、provider WP、Owner、三Target bindingを一つのProduct Decisionでactive Registryへ登録しvalidationを通すまでC2 3Dを評価不能` | `gate.product.reconsider-c2-3d` | `null` |
| `wp.product.production-release-binding` | `phase.production-capability` | `mirakan.arch.product-plan` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.release-closure` | `capability.foundation.cpp23-cx3-shipping` | `wp.foundation.cpp23-cx3-shipping; wp.runtime.ecs-e5-2d-integration; wp.platform.windows-package; wp.platform.android-package; wp.platform.apple-package` | `deferred` | `current Activation Binding全293行からmodeを導出する。全disabledのsourceではproduction_release_migration_authoringとしてpreliminary destinationだけを作りcomplete禁止、全all_release_gatesのdestinationではproduction_release_binding_qualificationとしてDefinition不変のfresh Target closure＋Owner acceptanceでcompleteするまでproduction遷移を禁止する。Genre／Feature Pack未導入のCore holdoutは`wp.runtime.ecs-e5-2d-integration`の`fixture.product.genreless-core-2d`で検証し、bundled Reference Game coverageは別nonblocking trackとする` | `gate.product.reconsider-production-release-binding` | `null` |
| `wp.product.runtime-generation` | `phase.runtime-generation` | `mirakan.arch.ai-security-approval` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.runtime-generation-denial` | `capability.foundation.control-plane` | `wp.architecture.control-plane` | `declared` | `null` | `[]` | `null` |

`wp.rendering.environment-c2`は`capability.rendering.render-graph-core`が提供するtyped `LightSnapshotV1` runtime入力を利用するが、これをLighting discovery／authoringのProduct activationまたは新しいLighting WPとして数えない。Time-of-Day intentがsun／moon Source変更を必要とする場合だけ、[Lighting](../06-rendering/lighting.md)のexact `ResolvedLightPlanRefV1`と`LightingChangeSetProposalRefV1`を同じbase Project revision／World／Target／expiryへ解決する。対象Light集合はbindingのnon-null sun／moon集合とexact set equalityにし、Environment／Lighting／World binding primitiveを一つの`ProjectChangeSetV1`へmaterializeしてatomicにPreview／Approval／Commitする。`planning.operation_family.lighting_discovery`が未Activation、Plan／Proposalがmissing／stale、scope／対象Light集合が不一致の時はEnvironment側だけの部分変更を成功させず、質問またはCapability未Activationとして停止する。

同Work Packageの`fallback.rendering.environment-core`はC2 advanced capabilityをC1 Core構成へ落とすTarget内substitutionであり、Work PackageまたはCore provider自体の失敗回避ではない。Coreはprecomputed sky、height fog、2D cloud、baked diffuse／specular IBLの全件を要求し、`capability.environment.core`、`height_fog`、`ibl_baked`のいずれかが不成立なら`fallback.capability.unavailable`でfail closedにする。Water／Weather／SnowのSource／Runtime schemaは同じOwner文書に存在するが、現行`wp.rendering.environment-c2`、`planning.operation_family.environment_authoring`、`capability.environment.*`のmutation／Product closureに含めない。別familyとTarget Qualificationが登録されるまでread-only／not activatedである。

#### ECS Work Packageのtarget owner mapping

Work Package表の`Owner`列はcurrent generated Architecture Document Registryへ解決する`owner_document_id`であり、文書刷新だけで先行変更しない。したがって`wp.runtime.ecs-e0`から`wp.runtime.ecs-e7-*`の現行Owner列は、`RuntimeEcsCanonicalizationChangeSetV1`が`applied`になるまで旧current authorityを指す。この扱いはECS実装またはOwner移管を許可するものではない。

次表はProductの計画上のtarget責務を示す閲覧用mappingであり、Work Package lifecycle、Capability activation、current Owner Registryを変更しない。

| Work Package群 | target Architecture Owner | 境界 |
|---|---|---|
| `ecs-e0`、`ecs-e1-storage`、`ecs-e2-query-mutation` | [Runtime ECS](../04-runtime/entity-component-system.md) | Entity／Component、archetype、query、access manifest、structural transaction |
| `ecs-e3-cook-load` | [Runtime Package](../04-runtime/runtime-package.md)＋[Asset Lifecycle](../03-authoring/asset-lifecycle.md)＋[Persistence／Save](../04-runtime/persistence-save.md) | World image、artifact catalog、load／reconstruction boundary |
| `ecs-e4-game-system`、`ecs-e5-*-integration` | [Runtime ECS](../04-runtime/entity-component-system.md)＋[Gameplay programming model](../03-authoring/gameplay-programming-model.md) | System binding、Domain integration、current／target Owner transfer |
| `ecs-e6-debug-ai` | [Runtime ECS](../04-runtime/entity-component-system.md)＋[Debugging／Observability／Replay](../04-runtime/debugging-observability-replay.md)＋[AI Security／Approval](../01-governance/ai-security-approval.md) | sealed projection、debug transport、authorization |
| `ecs-e7-*-*` | [Runtime ECS](../04-runtime/entity-component-system.md)＋各Platform Owner | Target qualification。Platform-specific runtimeをECS正本へ複写しない |

### Destination projection: data-oriented ECS core

This subsection is a closed projection of an unmaterialized, unapproved, and unapplied destination migration candidate associated with `RuntimeEcsCanonicalizationChangeSetV1` and its Owner-reference migration. The current Governance profile remains `state=review`, `contract_activation_effect=none`, with `definition_migration_binding_ref` absent. This subsection is not part of the current source Definition and does not change the operational snapshot. Only if a complete Product Definition Migration is separately materialized and approved do the destination projection, Change Set, and Owner-reference migration apply atomically; the source remains byte-equal until that atomic application.

| Registry | Exact addition or replacement |
|---|---|
| `RequirementRegistryV1` | `{requirement_id=requirement.runtime.ecs-data-oriented-core, owner_document_id=mirakan.arch.runtime-entity-component-system, verification_kind=runtime_ecs_data_oriented_qualification, failure_diagnostic_id=diagnostic.product.ecs-data-oriented-core-failed}` |
| `FixtureRegistryV1` | `{fixture_id=fixture.runtime.ecs-data-oriented-core, owner_document_id=mirakan.arch.runtime-entity-component-system, requirement_refs=[requirement.runtime.ecs-data-oriented-core], target_refs=[target.headless.host,target.windows.desktop,target.android.mobile,target.apple.mobile], minimum_duration_seconds=12600}` |
| `PhaseFixtureBindingRegistryV1` | `{gate_id=gate.product.phase-0-ecs-data-oriented-core, phase_id=phase.foundation, fixture_id=fixture.runtime.ecs-data-oriented-core, evaluated_requirement_refs=[requirement.runtime.ecs-data-oriented-core], target_refs=[target.headless.host], candidate_binding_policy_ref=policy.product.same-candidate.v1, freshness_policy_ref=policy.evidence.contract-ci.v1}` |
| `ProductPhaseRegistryV1` `phase.foundation` | append the new Requirement to `outcome_requirement_refs[]` and the new Gate to `exit_gate_refs[]`; `work_package_refs[]` is unchanged |
| `WorkPackageRegistryV1` | append `{kind=product_fixture, fixture_id=fixture.runtime.ecs-data-oriented-core}` to `provided_fixture_refs[]` of `wp.foundation.memory-pointers`, `wp.runtime.scheduling-core`, `wp.runtime.ecs-e0`, `wp.runtime.ecs-e1-storage`, and `wp.runtime.ecs-e2-query-mutation`; all other Fields remain unchanged except the coordinated Owner replacements |

Headless Phase 0 GateはWindows、Android、Apple qualificationを代用しない。各Targetはfull 12,600-second fixtureを`policy.evidence.target-device.v1`で再実行する。

各E7 rerunはartifact Targetだけでなく、exact Target Profile content hash、device identity／hardware profile、OS image、driver／runtime package、Toolchain lock、Contract Set、fixture、input trace、initial absolute threshold set、campaign hashを一つのEvidence subjectへ束縛する。Editor execution Host、別Target、同family device、characterization result、Shooter ReceiptをTarget runtime Receiptへ読み替えない。いずれかが未materializeまたはstaleなら該当E7 Capabilityを`not_activated`に保つ。

destinationにおける`requirement.foundation.memory-pointer-contract`のdefinition closureはexactly次の4 Typeである。

```text
[PointerContractV1, MemoryContractV1, PointerMemoryConsumerBindingV1, CppValueTransferPolicyV1]
```

同Requirementは、同じfour-Type Contract Set内のexact `MemoryContractV1` Type member ref／schema hashと、retained-Field、single-`capacity_source`、six-layout／access-Field invariantを証明するfresh `fixture.foundation.memory-pointer-contract` Receiptへ束縛する。Memory field listは本書へ複写しない。

destinationだけで`risk.product.memory-pointer-contract-drift.mitigation`を次のexact valueへ置換し、他Fieldはsource rowとbyte-equalに保つ。

```text
the exact four-Type definition closure [PointerContractV1, MemoryContractV1, PointerMemoryConsumerBindingV1, CppValueTransferPolicyV1], bidirectional consumer Matrix, static／negative fixture, supported sanitizer lane, and hot path fallback 0 are bound to the same Phase 0 Candidate Gate
```

Owner-reference migrationは`wp.runtime.ecs-e0`、`wp.runtime.ecs-e1-storage`、`wp.runtime.ecs-e2-query-mutation`の`owner_document_id`だけを`mirakan.arch.runtime-scheduling-lifetime`から`mirakan.arch.runtime-entity-component-system`へ置換する。`wp.foundation.memory-pointers`、`wp.runtime.scheduling-core`、依存chainは変更しない。

destination Requirementはfixture Componentのaccepted `RuntimeComponentLayoutPolicyV1` record、`ecs_chunk_soa_v1`を使う一つの`RuntimeArchetypeLayoutPlanV1`、query／lease／structural contract、全35 mandatory metric ID、全hard predicate、Target別`RuntimeEcsInitialAcceptanceThresholdSetV1`、同一Candidateに束縛した8192／16384／32768-byte characterization、Shipping AoS／sparse-set／object graph／general-heap fallback 0を要求する。

| Work Package | Added completion responsibility |
|---|---|
| `wp.foundation.memory-pointers` | value-transfer policy、container layout Fields、static and negative Gates |
| `wp.runtime.ecs-e0` | type、owner、diagnostic, and Contract closure |
| `wp.runtime.ecs-e1-storage` | chunk SoA、layout policy、capacity、handle、fragmentation metrics |
| `wp.runtime.ecs-e2-query-mutation` | cached query、contiguous dispatch、allocation-free callback、deferred structural transaction |
| later `wp.runtime.ecs-e7-*` | rerun the qualified profile on the exact Target and device Evidence policy |

Phase Fixture GateはRequirement、fixture、Target、Candidate、freshnessだけを評価する。Phase exitは別に、E1／E2を含む全non-deferred Work Packageのcurrent lifecycle headが`complete`であることを要求する。

destination `ProductRiskRegistryV1`には次の一行を追加する。risk rowの唯一OwnerはProduct Planであり、他文書はtrigger、severity、mitigation、containmentを複写しない。

| risk_id | owner_document_id | affected_work_package_refs[] | trigger | likelihood | impact | mitigation | contingency | monitor_gate_refs[] | genesis_state | revisit_gate_or_date |
|---|---|---|---|---|---|---|---|---|---|---|
| `risk.product.ecs-data-oriented-regression` | `mirakan.arch.product-plan` | `wp.foundation.memory-pointers; wp.runtime.scheduling-core; wp.runtime.ecs-e0; wp.runtime.ecs-e1-storage; wp.runtime.ecs-e2-query-mutation` | missing layout policy、dual Shipping layout、hot callback allocation／fallback、unbounded archetype growth、missing campaign cell／metric, or wrong-Target Receipt substitution | `high` | `critical` | require the destination Phase 0 Gate、complete campaign、hard predicates, and fresh Target-specific reruns | reject the affected ECS Work Package transition and dependent Runtime activation; retain the last qualified layout without an alternate Shipping fallback | `gate.product.phase-0-ecs-data-oriented-core` | `open` | `{kind=phase_gate, ref=gate.product.phase-0-ecs-data-oriented-core}` |

```text
diagnostic.product.ecs-target-receipt-mismatch
MIRAKAN-PRODUCT-ECS-TARGET-RECEIPT-MISMATCH
arguments = campaign_hash, expected_target_ref, actual_target_ref
result = Product Gate failure
```

aggregate failureはRuntime ECS所有の`diagnostic.product.ecs-data-oriented-core-failed`を参照し、本書でschemaを複製しない。

`wp.authoring.prequalified-source-packs`は初心者向けDefinition-first経路と、事前Qualification済みNative／Shader Packの選択だけを提供する。Phase 5の`wp.authoring.project-native-module`と`wp.rendering.project-shader`で新規AI生成Sourceを採用する場合は、各Ownerの独立したCode owner approval GateとSource／artifact／Receipt hash closureを必須とし、事前PackのReceiptを代用しない。`wp.product.project-source-activation`は両Source WPを集約するが、`wp.product.external-agent`とは依存もRequirementも共有せず、Proposal-only境界をSource生成の成功扱いにしない。

Production owner layerがGeneric Engine CoreであるWPは、`required_capability_refs[]`または`requires_work_package_refs[]`からFeature Pack、Genre Pack、Reference Gameへ到達してはならない。Reference Gameを`provided_fixture_refs[]`の`kind=product_fixture`として置くことはQualification inputの宣言でありproduction dependencyではないが、各Core Capabilityは同じTargetで`fixture.product.genreless-core-2d`または`fixture.product.genreless-ai-project`を使うPack-uninstalled holdout ReceiptなしにActivationできない。`wp.navigation.core`はNavigation query／artifact／provider portと`capability.simulation.navigation`だけを所有し、`capability.gameplay.path_following`は`wp.gameplay.reusable-features-c1`が所有する。InteractionもPublic schemaの正本所在と実装Capability ownerを分け、`capability.gameplay.interaction`のProduct owner WPはFeature Packに固定する。

`wp.runtime.ecs-e7-windows-2d`と`capability.runtime.ecs-e7-windows-2d`のartifact／runtime Targetは`target.windows.desktop`だけである。AI／EditorがそのBuild、Test、Previewを起動するexecution Hostは§10.6の`execution_host_rule=windows_editor_for_artifact_target`で`target.windows.editor`へ別に束縛し、WP／Capability TargetへEditorを重複登録しない。これによりDesktop-only `capability.runtime.ecs-e5-2d-integration`をEditor artifact Capabilityへ偽装せず、Editor Host ReceiptをDesktop runtime Receiptとして流用しない。

`wp.platform.input-core`、`wp.platform.audio-core`、`wp.platform.ui-core`はportable contractを所有し、対応する`*-windows` WPはWindows qualificationだけを所有する。`mobile-io-ui-*`からportable core WPへの`requires_work_package_refs[]`はcontract build-order edgeであり、Windows qualification WPまたはWindows Receiptへの依存ではない。`wp.domain.shooter-2d`と`wp.domain.shooter-3d`はWindows／Android／Apple共通のportable Genre implementationを所有し、Generic Runtime、AI、Debug、Source、Shader、Target backend、E7、Packageのいずれからもrequired edgeを受けない。Android／Appleの実機I/O／UIは同Targetの`wp.platform.mobile-io-ui-android`／`wp.platform.mobile-io-ui-apple`、2D／3D ECS統合はTarget別E7 WPがCore subsystem closureだけをrequired edgeに持ってfresh Phase 7 Receiptを提供する。Shooter fixtureはTarget別E7のQualification caseとして利用できるが、Shooter Capability、Shooter WP、Shooter ReceiptをE7またはPackageの成立条件にしない。Phase 8のPlatformer／Puzzle ReceiptもPhase 7 Activation前提にしない。既存V1 IDの`wp.domain.*`／`capability.domain.*`にある`domain` tokenは互換用opaque segmentであり製品層を表さない。authoritative layer／artifact roleはArchitecture Dependency分類とGeneric Architectureの四層規則だけから解決し、AI／Compiler／Release GateはID substringからGenre、Core、required dependencyを推測しない。

### 11.6 Capability–Target–Fallback registry

本表はCapability identity、Product tier、owner WP、Target scope、fallbackだけを所有し、Activation stateを保存しない。Target scopeの`Headless`、`Windows`、`Windows Editor`、`Android`、`Apple`はそれぞれ`target.headless.host`、`target.windows.desktop`、`target.windows.editor`、`target.android.mobile`、`target.apple.mobile`の略記であり、Registry生成時は略記を保存せずexact IDへ展開する。明記しないTargetは暗黙requiredにせず`excluded`として生成する。

| capability_id | tier | owner WP | Target scope | fallback |
|---|---|---|---|---|
| `capability.foundation.control-plane` | C0 | `wp.architecture.control-plane` | Headless required; Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.foundation.cpp23-cx0` | C0 | `wp.foundation.cpp23-cx0` | Headless required; Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.foundation.cpp23-cx2-cutover` | C0 | `wp.foundation.cpp23-cx2-cutover` | Headless required; Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.foundation.cpp23-cx3-shipping` | C0 | `wp.foundation.cpp23-cx3-shipping` | Headless required; Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.foundation.math-core` | C0 | `wp.foundation.math-core` | Headless required; Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.foundation.memory-pointers` | C0 | `wp.foundation.memory-pointers` | Headless required; Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.runtime.scheduling` | C0 | `wp.runtime.scheduling-core` | Headless required; Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e0-contract` | C0 | `wp.runtime.ecs-e0` | Headless required; Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e1-storage` | C0 | `wp.runtime.ecs-e1-storage` | Headless required; Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e2-query-mutation` | C0 | `wp.runtime.ecs-e2-query-mutation` | Headless required; Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e3-cook-load` | C0 | `wp.runtime.ecs-e3-cook-load` | Headless required; Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.authoring.project-state-headless` | C0 | `wp.authoring.project-state-headless` | Headless required; Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.authoring.asset-save-headless` | C0 | `wp.authoring.asset-save-headless` | Headless required; Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.authoring.manual-roundtrip` | C0 | `wp.authoring.headless-core` | Headless required; Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e4-game-system` | C0 | `wp.runtime.ecs-e4-game-system` | Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.rendering.render-graph-core` | C0 | `wp.rendering.render-graph-core` | Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.rendering.d3d12-cx0` | C0 | `wp.rendering.d3d12-backend` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.platform.input-core` | C0 | `wp.platform.input-core` | Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.platform.audio-core` | C0 | `wp.platform.audio-core` | Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.platform.ui-core` | C0 | `wp.platform.ui-core` | Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.platform.windows-package` | C0 | `wp.platform.windows-package` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.product.editor-runtime-windows` | C0 | `wp.product.editor-runtime-windows` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.world.2d` | C1 | `wp.rendering.world-2d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.camera.2d` | C1 | `wp.rendering.camera-2d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.simulation.collision-2d` | C1 | `wp.simulation.collision-2d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.simulation.physics-2d` | C1 | `wp.simulation.physics-2d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.simulation.animation-2d` | C1 | `wp.simulation.animation-2d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.simulation.navigation` | C1 | `wp.navigation.core` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e5-2d-integration` | C1 | `wp.runtime.ecs-e5-2d-integration` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.domain.shooter-2d` | C1 | `wp.domain.shooter-2d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.authoring.ai-core` | C1 | `wp.authoring.ai-core` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.runtime.debug-replay-support` | C1 | `wp.runtime.debug-replay-support` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e6-debug-ai` | C1 | `wp.runtime.ecs-e6-debug-ai` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.authoring.prequalified-source-packs` | C1 | `wp.authoring.prequalified-source-packs` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e7-windows-2d` | C1 | `wp.runtime.ecs-e7-windows-2d` | Windows Editor excluded; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.product.ai-authoring-mvp-a` | C1 | `wp.product.ai-authoring-mvp-a` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.product.external-agent` | C1 | `wp.product.external-agent` | Headless required; Windows Editor excluded; Windows excluded; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.product.project-source-activation` | C1 | `wp.product.project-source-activation` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.world.3d` | C1 | `wp.rendering.world-3d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.camera.3d` | C1 | `wp.rendering.camera-3d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.simulation.collision-3d` | C1 | `wp.simulation.collision-3d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.simulation.physics-3d` | C1 | `wp.simulation.physics-3d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.simulation.animation-3d` | C1 | `wp.simulation.animation-3d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e5-3d-integration` | C1 | `wp.runtime.ecs-e5-3d-integration` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e7-windows-3d` | C1 | `wp.runtime.ecs-e7-windows-3d` | Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e7-android-2d` | C1 | `wp.runtime.ecs-e7-android-2d` | Android required; Windows Editor excluded; Windows excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e7-apple-2d` | C1 | `wp.runtime.ecs-e7-apple-2d` | Apple required; Windows Editor excluded; Windows excluded; Android excluded | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e7-android-3d` | C1 | `wp.runtime.ecs-e7-android-3d` | Android required; Windows Editor excluded; Windows excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e7-apple-3d` | C1 | `wp.runtime.ecs-e7-apple-3d` | Apple required; Windows Editor excluded; Windows excluded; Android excluded | `fallback.capability.unavailable` |
| `capability.domain.shooter-3d` | C1 | `wp.domain.shooter-3d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.rendering.vulkan-backend` | C1 | `wp.rendering.vulkan-backend` | Android required; Windows Editor excluded; Windows excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.rendering.metal-backend` | C1 | `wp.rendering.metal-backend` | Apple required; Windows Editor excluded; Windows excluded; Android excluded | `fallback.capability.unavailable` |
| `capability.platform.android-package` | C1 | `wp.platform.android-package` | Android required; Windows Editor excluded; Windows excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.platform.apple-package` | C1 | `wp.platform.apple-package` | Apple required; Windows Editor excluded; Windows excluded; Android excluded | `fallback.capability.unavailable` |
| `capability.platform.mobile-lifecycle` | C1 | `wp.platform.mobile-offline` | Android required; Apple required; Windows Editor excluded; Windows excluded | `fallback.capability.unavailable` |
| `capability.platform.mobile-io-ui-android` | C1 | `wp.platform.mobile-io-ui-android` | Android required; Windows Editor excluded; Windows excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.platform.mobile-io-ui-apple` | C1 | `wp.platform.mobile-io-ui-apple` | Apple required; Windows Editor excluded; Windows excluded; Android excluded | `fallback.capability.unavailable` |
| `capability.domain.platformer` | C2 | `wp.domain.platformer` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.domain.puzzle-dialogue` | C2 | `wp.domain.puzzle-dialogue` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.environment.aerial_perspective` | C2 | `wp.rendering.environment-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.environment-core` |
| `capability.environment.atmosphere_lut` | C2 | `wp.rendering.environment-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.environment-core` |
| `capability.environment.cloud_shadow` | C2 | `wp.rendering.environment-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.environment-core` |
| `capability.environment.core` | C1 | `wp.rendering.environment-c2` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.environment.dynamic_ibl` | C2 | `wp.rendering.environment-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.ibl-baked` |
| `capability.environment.height_fog` | C1 | `wp.rendering.environment-c2` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.environment.ibl_baked` | C1 | `wp.rendering.environment-c2` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.environment.intent_resolver` | C1 | `wp.rendering.environment-c2` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.environment.local_fog_volume` | C2 | `wp.rendering.environment-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.environment-core` |
| `capability.environment.sky_hdri` | C1 | `wp.rendering.environment-c2` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.environment.volumetric_cloud` | C2 | `wp.rendering.environment-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.environment-core` |
| `capability.environment.volumetric_fog` | C2 | `wp.rendering.environment-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.environment-core` |
| `capability.gameplay.interaction` | C1 | `wp.gameplay.reusable-features-c1` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.gameplay.path_following` | C1 | `wp.gameplay.reusable-features-c1` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.gameplay.perception` | C1 | `wp.gameplay.core-c1` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.gameplay.combat` | C1 | `wp.gameplay.reusable-features-c1` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.gameplay.ranged_combat` | C1 | `wp.gameplay.reusable-features-c1` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.gameplay.encounter_spawn` | C1 | `wp.gameplay.reusable-features-c1` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.gameplay.scoring` | C1 | `wp.gameplay.reusable-features-c1` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.gameplay.pickup_grant` | C1 | `wp.gameplay.reusable-features-c1` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.gameplay.character_locomotion` | C1 | `wp.gameplay.reusable-features-c1` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.gameplay.scenario_stage` | C1 | `wp.gameplay.reusable-features-c1` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.runtime.timer` | C1 | `wp.runtime.timer` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.project.native_module` | C1 | `wp.authoring.project-native-module` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.project.shader` | C1 | `wp.rendering.project-shader` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.product.general_production_2d` | C2 | `wp.product.general-coverage-2d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.product.general_production_3d` | C2 | `wp.product.general-coverage-3d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.product.runtime-generation-boundary` | C0 | `wp.product.runtime-generation` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.render.material.realistic_advanced` | C2 | `wp.rendering.material-realistic` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.render.material.toon` | C2 | `wp.rendering.material-toon` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.ui.native_widget` | C2 | `wp.ui.native-widget` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.vfx.bake_cache` | C2 | `wp.rendering.vfx-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-core` |
| `capability.vfx.billboard_3d` | C1 | `wp.rendering.vfx-c2` | Windows required; Android required; Apple required | `fallback.rendering.vfx-cpu` |
| `capability.vfx.extension_operator` | C2 | `wp.rendering.vfx-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-core` |
| `capability.vfx.mesh_ribbon` | C2 | `wp.rendering.vfx-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-cpu` |
| `capability.vfx.particle_cpu` | C1 | `wp.rendering.vfx-c2` | Windows required; Android required; Apple required | `fallback.rendering.vfx-core` |
| `capability.vfx.particle_gpu` | C2 | `wp.rendering.vfx-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-cpu` |
| `capability.vfx.particle_light` | C2 | `wp.rendering.vfx-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-core` |
| `capability.vfx.pattern_catalog` | C1 | `wp.rendering.vfx-c2` | Windows required; Android required; Apple required | `fallback.rendering.vfx-core` |
| `capability.vfx.semantic_intent` | C1 | `wp.rendering.vfx-c2` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.vfx.sprite_2d` | C1 | `wp.rendering.vfx-c2` | Windows required; Android required; Apple required | `fallback.rendering.vfx-cpu` |
| `capability.vfx.system` | C1 | `wp.rendering.vfx-c2` | Windows required; Android required; Apple required | `fallback.rendering.vfx-core` |
| `capability.vfx.trail` | C1 | `wp.rendering.vfx-c2` | Windows required; Android required; Apple required | `fallback.rendering.vfx-cpu` |
| `capability.vfx.visual_collision` | C2 | `wp.rendering.vfx-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-core` |

`required_capability_refs[]`はconsumer WPの各`target_refs[]`で実行時に必須なCapabilityだけを列挙する。各edgeについてCapability表の同Target scopeが`required`でなければ拒否し、`optional`または`excluded`をrequired edgeに使わない。optional Capabilityを選べるWPはrequired edgeへ混在させず、Owner contractの明示selection policyとfallback refで表し、選択時だけcurrent Target state／fresh fallbackを検査する。headless authoring、build host、Code owner、Phase順序などcross-targetのorderingだけを表す関係は`requires_work_package_refs[]`へ置き、Target availabilityの証拠にしない。`defer_reason`、依存、`reconsideration_gate_refs[]`はWork Packageだけが所有し、Capability表または`CapabilityTargetActivationStateV1`へ複写しない。

現行`WorkPackageRegistryV1`にはoptional Capability選択Fieldがなく、全`required_capability_refs[]` edgeは`required` scopeだけで閉じる。将来WPがoptional selection自体をProduct正本へ持つ場合は、selection policy、fallback、selected／unselected fixtureを含むDefinition schema revisionを先に承認し、自由記述や既存required edgeの読み替えで追加しない。

### 11.7 Product risk registry

`likelihood`のclosed値は`low | medium | high | unknown`、`impact`は`moderate | major | critical`、operational `state`は`open | monitoring | mitigated | accepted | closed`である。`revisit_gate_or_date`は`kind=phase_gate`なら登録済みPhase Fixture Gateへの`ref`、`kind=decision_gate`なら登録済みProduct Decision Gateへの`ref`、`kind=date`なら根拠のあるISO 8601 `YYYY-MM-DD`の`value`だけを受理し、kind不一致Field、unknown kind、裸の文字列を拒否する。Markdown表は厳密な略記`phase_gate:<exact-id>`、`decision_gate:<exact-id>`、`date:<YYYY-MM-DD>`だけを許し、Registry生成時に上記discriminated unionへ展開する。本Revisionは未確定日程を作らず、全行をexact gateで再評価する。表の`monitor gate refs`はDefinitionのlogical gate refsであり、Evidence content refではない。genesis `risk_evaluations[].state=open`、`evidence_refs=[]`とし、後続の署名済みEvaluation transitionだけが実Receipt refを追加する。

| risk_id | Owner | affected work packages | trigger | likelihood | impact | mitigation | contingency | monitor gate refs | genesis state | revisit gate or date |
|---|---|---|---|---|---|---|---|---|---|---|
| `risk.product.control-plane-schema-drift` | `mirakan.arch.core-architecture` | `wp.architecture.control-plane` | Product schema consumerがcanonical Field名、closed値、またはrevisionと不一致になる | `high` | `critical` | Product Registryを唯一の正本とし、bootstrap前にschema hashとexact Field集合を検査する | scheduling startを拒否し、last-known-good schemaへ固定する | `gate.product.phase-0-headless-contract` | `open` | `phase_gate:gate.product.phase-0-headless-contract` |
| `risk.product.cpp23-modules-cmake-constraint` | `mirakan.arch.cpp23-modules` | `wp.foundation.cpp23-cx0; wp.foundation.cpp23-cx2-cutover; wp.foundation.cpp23-cx3-shipping` | CX0 productionへCX1 fixture外の`.ixx`／`.cppm`が混入する、またはCX2／CX3でCMake import std、BMI順序、Tooling、正式C++23、全Target Receiptのいずれかが成立しない | `high` | `critical` | CX0はself-contained Headerへ固定し、CX1 fixtureを隔離し、CX2 cutoverとCX3 Shippingを別deferred WP／Decision Gateで判定する | CX0を維持してCX2／CX3 schedulingとRelease Activationを拒否し、別Generatorまたはpreview Toolchainへfallbackしない | `gate.product.phase-0-headless-contract; gate.product.reconsider-cpp23-cx2-cutover; gate.product.reconsider-cpp23-cx3-shipping` | `open` | `decision_gate:gate.product.reconsider-cpp23-cx3-shipping` |
| `risk.product.memory-pointer-contract-drift` | `mirakan.arch.memory-pointers` | `wp.foundation.memory-pointers; wp.runtime.scheduling-core; wp.runtime.ecs-e0; wp.runtime.ecs-e1-storage; wp.runtime.ecs-e2-query-mutation` | consumer binding、owner、保存／job capture制約、またはsupported sanitizer Evidenceのいずれかが欠落し、local wrapperや暗黙fallbackで迂回される | `high` | `critical` | MCDの三Contract、正逆consumer Matrix、static／negative fixture、supported sanitizer lane、hot path fallback 0をPhase 0の同一Candidate Gateへ束縛する | 当該WPのschedulingと後続Runtime activationを拒否し、欠落Evidenceを別Target／旧ID／aliasで代用しない | `gate.product.phase-0-memory-pointer-contract` | `open` | `phase_gate:gate.product.phase-0-memory-pointer-contract` |
| `risk.product.ai-tool-safety-code-owner` | `mirakan.arch.ai-security-approval` | `wp.authoring.ai-core; wp.authoring.project-native-module; wp.rendering.project-shader; wp.product.project-source-activation` | AI toolが権限外Operationを要求する、または新規Native／`bounded_hlsl`にCode Ownerが割り当てられない、または`typed_ir`のcoverage／Target gateが欠落する | `high` | `critical` | MVP-AをDefinition-first／prequalified Packへ限定し、Native／`bounded_hlsl`を独立GateとCode Owner approvalへ、`typed_ir`をIR・coverage・Target gateへ送る | 新規Source／IR laneをfail closedにし、MVP-AとProposal-only external agentを維持する | `gate.product.phase-4-ai-mvp-a; gate.product.phase-5-project-source-activation` | `open` | `phase_gate:gate.product.phase-5-project-source-activation` |
| `risk.product.target-device-signing` | `mirakan.arch.product-plan` | `wp.platform.windows-package; wp.platform.android-package; wp.platform.apple-package` | 実機、署名資格情報、store package、device lab、またはoffline launch Receiptが揃わない | `high` | `critical` | Target別package WP、同一Candidate binding、target-device freshnessで検査する | 未合格TargetをProduct labelとShippingから除外する | `gate.product.phase-2-windows-empty-scene; gate.product.phase-7-android-lifecycle; gate.product.phase-7-apple-lifecycle; gate.product.phase-7-android-runtime-2d; gate.product.phase-7-apple-runtime-2d; gate.product.phase-7-android-runtime-3d; gate.product.phase-7-apple-runtime-3d` | `open` | `phase_gate:gate.product.phase-7-apple-runtime-3d` |
| `risk.product.asset-license-provenance` | `mirakan.arch.asset-lifecycle` | `wp.authoring.asset-save-headless; wp.product.ai-authoring-mvp-a` | Asset source、license、生成由来、またはSBOM provenanceが同一Project revisionへ閉じない | `medium` | `critical` | Asset import／save ReceiptとAI provenanceをCandidate hashへ束縛する | 問題Assetを隔離し、packageとProduct labelを拒否する | `gate.product.phase-1-authoring-transaction; gate.product.phase-4-ai-mvp-a` | `open` | `phase_gate:gate.product.phase-4-ai-mvp-a` |
| `risk.product.c1-performance-calibration` | `mirakan.arch.runtime-performance-capacity` | `wp.gameplay.core-c1; wp.domain.shooter-2d; wp.domain.shooter-3d` | C1 fixtureのframe、memory、load、soak基準が実測Target profileでcalibrateされていない | `unknown` | `major` | Phase 3／6／7 Candidateで同一入力のTarget別baselineを測定し、推測値をReceiptへ置換する | C1表示を対象Targetで保留し、機能削減を勝手に行わない | `gate.product.phase-3-manual-2d; gate.product.phase-6-first-playable-3d; gate.product.phase-7-android-runtime-2d; gate.product.phase-7-apple-runtime-2d; gate.product.phase-7-android-runtime-3d; gate.product.phase-7-apple-runtime-3d` | `open` | `phase_gate:gate.product.phase-7-apple-runtime-3d` |
| `risk.product.mobile-backend-closure` | `mirakan.arch.rendering-render-graph` | `wp.rendering.vulkan-backend; wp.rendering.metal-backend; wp.platform.mobile-io-ui-android; wp.platform.mobile-io-ui-apple; wp.runtime.ecs-e7-android-2d; wp.runtime.ecs-e7-apple-2d; wp.runtime.ecs-e7-android-3d; wp.runtime.ecs-e7-apple-3d; wp.platform.android-package; wp.platform.apple-package` | Mobile backend、I/O／UI、2D／3D ECS qualificationのいずれかが対象Targetで未合格またはWindows Receiptを流用する | `high` | `critical` | Android／Apple別WP、C1 Shooter実機Gate、CapabilityTargetActivation行を使い、Windows Receipt流用を0件にする | 該当Mobile TargetのpackageとShipping labelを拒否する | `gate.product.phase-7-android-lifecycle; gate.product.phase-7-apple-lifecycle; gate.product.phase-7-android-runtime-2d; gate.product.phase-7-apple-runtime-2d; gate.product.phase-7-android-runtime-3d; gate.product.phase-7-apple-runtime-3d` | `open` | `phase_gate:gate.product.phase-7-apple-runtime-3d` |
| `risk.product.c2-3d-second-fixture` | `mirakan.arch.product-plan` | `wp.product.general-coverage-3d` | 第二の非Shooter 3D Fixtureとprovider closureがactive Registryに存在しない | `high` | `major` | aggregate WPをexact理由付き`deferred`にし、closed Decision Gateだけで再検討する | Product labelを`3D First Playable`に限定し、C2 3D Receiptを発行しない | `gate.product.phase-6-first-playable-3d; gate.product.reconsider-c2-3d` | `open` | `decision_gate:gate.product.reconsider-c2-3d` |
| `risk.product.online-large-world-incubation` | `mirakan.arch.product-plan` | `wp.product.general-coverage-3d; wp.product.runtime-generation` | Online、Large World、またはpositive Runtime generationをplanning-only Decisionなしでactive DAGへ入れようとする | `medium` | `critical` | 対応FutureCapability entryを`planning_only`に保ち、Owner、authority、Threat Model、Target、fallback、Qualificationを一括Decision化する | activationを拒否し、offline／deny-only Product boundaryを維持する | `gate.product.phase-8-c2-shooter-2d; gate.product.phase-9-runtime-generation-denial` | `open` | `phase_gate:gate.product.phase-9-runtime-generation-denial` |

### 11.8 C++23 Cutover／Shipping gate

CX0／CX1はDevelopment、Test、candidate Package、internal Technology Previewだけに使用し、Release Activationへ入力しない。CX0／CX1を使う`capability.rendering.d3d12-cx0`は内部`candidate_locked`を上限とし、`qualified`／`production`への遷移を拒否する。Shipping Configurationのcandidate Package成功は公開Releaseの証拠ではない。

`wp.foundation.cpp23-cx2-cutover`と`capability.foundation.cpp23-cx2-cutover`は全TargetのModule Cutover候補、`wp.foundation.cpp23-cx3-shipping`と`capability.foundation.cpp23-cx3-shipping`は全TargetのRelease有効化を所有する。両Work Packageは初期`deferred`、両CapabilityのTarget行は初期`not_activated`であり、Planning Decision Gateが`satisfied`でもWork Packageの再schedulingだけを許可してActivationを自動昇格しない。

CMakeの非Experimental `import std`、Ninja／Ninja Multi-Config経路、Microsoft公式資料で文書化された非PreviewのC++23適合mode、全TargetのBuild／Tooling／ABI／Package／Release Receiptのいずれかが欠ける場合はCX0 Headerを維持する。Visual Studio Generator由来BMI、MSVC 14.51の`/std:c++23preview`、CX1 fixture ReceiptをCX2／CX3の代用にしない。

### 11.9 C2 3D gate

`fixture.product.shooter-arena-3d`一件だけではProduct C2 3Dを評価しない。本RevisionのPhase 8 outcomeと`PhaseFixtureBindingRegistryV1`には`requirement.product.c2-3d-coverage`のexit gateが存在せず、`wp.product.general-coverage-3d`は`gate.product.reconsider-c2-3d`が`satisfied`になるまで`deferred`である。第二の非Shooter 3D Fixtureを未登録IDで先取りせず、同Decision Gateのpredicateが要求するFixture、Requirement binding、provider WP、Owner、Windows／Android／Apple Target closureを一つの承認済みChangeSetで登録し、validationが0 errorになった後だけ再schedulingできる。

`capability.product.general_production_3d`のTarget別保存状態はcurrent `CapabilityTargetActivationStateV1`だけが所有する。Decision Gateの`satisfied`、Work Packageの再scheduling、Shooter 3D ReceiptのいずれもTarget行を自動昇格せず、第二fixtureを含む新しいPhase bindingのfresh Receiptが揃うまでProduct labelを`3D First Playable`に限定する。
