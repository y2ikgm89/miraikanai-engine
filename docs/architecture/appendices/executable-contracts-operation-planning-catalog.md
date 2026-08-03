# Executable Contracts Operation／Planning Candidate Catalog

- 文書ID: mirakan.appendix.executable-contracts-operation-planning-catalog
- 文書種別: Owner supplement
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 親Owner: [Executable Contracts](../02-foundation/executable-contracts.md)
- 正本範囲: MCD具体record、Operation catalog、Service／Policy／Diagnostic projection、未Activation planning recordのreview候補詳細
- 非正本範囲: MCD共通意味、Operation成立条件、canonicalization、compiler、projection、current activation。親OwnerとDomain Ownerが決定する
- 規範依存: [親Owner](../02-foundation/executable-contracts.md)
- 関連文書: [Project State](../03-authoring/project-state.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-26

> 本書は分離前Owner文書の大量の具体MCD record、Operation catalog、未Activation planning候補を保持する。親Ownerの共通意味、closed partition、activation条件を上書きせず、RepositoryにSchema、compiler、generated projection、Receiptが存在しない候補をactiveまたはinstalledと扱わない。

Current stateは[Executable Contracts §8.2](../02-foundation/executable-contracts.md#82-target-installed-product-operation-closureとcurrent-empty-state)だけが所有し、Repository-wideの`materialized_operations`、`contract_active_operations`、`active_operations`、`operational_operations`はすべてexact `[]`である。本書§8.1／§8.2の十Operation、六entry Baseline、十entry Composition、MCD／Manifest／Service／Policy／Validator／Diagnostic／Receipt closureは`target_complete_not_materialized`候補であり、current instance、current hash、contract-active record、active Operationまたはdispatch authorityではない。Schema例の`status=active`、ID中の`.active`／`.current`は将来materialization後のrecord vocabularyであってRepository current stateを変更しない。

> 以下の見出し番号は、親Ownerの論点番号との対応を明示するために維持する。欠番は親Ownerが所有する規範であり、本書に補完しない。

## 分離前Owner節から抽出した候補record

### 5.1 Foundation Definition Closure

Owner identityとnamed algorithmはContract set memberではない独立したdefinition rootであり、次のclosureだけが三rootを結合する。`OwnerIdentityRegistryRefV1`の構造正本は[Gameplay programming modelのOwner identity registry](../03-authoring/gameplay-programming-model.md#owner-identity-registry)だけであり、本書はそのexact型をforward importして再定義しない。`ContractSetSnapshotV1.member_kind`の五kindとinitial `snapshot_version=1`を正規値とし、Owner／Algorithm recordまたはそのexternal refをmember preimageへ追加しない。

```text
NamedAlgorithmRegistryRefV1
  registry_id: named_algorithm.registry.active
  registry_version: positive uint32
  registry_content_hash: SHA-256

FoundationDefinitionClosureV1
  closure_version: 1
  contract_set:
    contract_set_id
    snapshot_version: 1
    contract_set_hash: SHA-256
  owner_identity_registry_ref: OwnerIdentityRegistryRefV1
  named_algorithm_registry_ref: NamedAlgorithmRegistryRefV1
  closure_content_hash: SHA-256

FoundationDefinitionClosureRefV1
  closure_version: 1
  closure_content_hash: SHA-256
```

`FoundationDefinitionClosureV1.closure_content_hash = SHA-256(ASCII "MIRAKAN_FOUNDATION_DEFINITION_CLOSURE_V1" || uint32_be(len(RFC 8785 JCS(closed closure object excluding closure_content_hash))) || RFC 8785 JCS(closed closure object excluding closure_content_hash))`とする。全stringはUnicode NFC、SHA-256はlowercase hexadecimal exact 64文字、`closure_version`／`snapshot_version`／Registry versionはJSON safe integer、unknown Fieldは禁止する。Owner rootは[Gameplay programming modelの`OwnerIdentityRegistryV1`](../03-authoring/gameplay-programming-model.md#owner-identity-registry)、Algorithm rootは本節の`NamedAlgorithmRegistryV1`へexact解決し、ID、version、隣接hashを再計算する。生成DAGは`Owner source document→Owner record／Owner Registry root`、`named algorithm definition→Named Algorithm Registry root`、`local MCD／Diagnostic／Trusted Service／Validator／Validator closure→Contract Set root`、`三root→FoundationDefinitionClosureV1`の一方向だけである。Foundation closure、Owner root、Algorithm root、またはそれらのexternal refをContract Set member hash／set rootへ戻さない。三rootのmissing／extra／duplicate、wrong version、隣接hash差、closure self-hash混入をfail closedにする。

Closure verifierは三rootのhash成立だけで成功してはならない。Contract Set内の全active MCD `owners[]`の各bare Owner IDを、Closureが指すOwner Identity Registryのselected `status=active` rowへexact一件解決し、そのrowから`OwnerIdentityLocalRefV1 {owner_id, owner_revision, owner_content_hash}`を投影する。各Owner Manifest contributionも同じ三Fieldと`authority_source`を持つ。set equalityの右辺はRegistry全rowではなく、current Contract Setのactive MCD、Owner Manifest contribution、Diagnostic／Runtime Scope／Game Systemのtyped Owner refから到達するselected-active Owner row subsetを同じ`OwnerIdentityLocalRefV1`へ投影した集合である。MCD bare IDから解決・投影したRef集合、ManifestのRef集合、reachable selected-active rowから投影したRef集合を同型三者でset equalityにする。string ID集合とthree-field Ref集合またはfull row集合を直接比較せず、同じID／stale revision／別content hashを同値にしない。Registryにlifecycle履歴としてselectedされた`deprecated | removed` rowまたはcurrent Contract Setから未参照のactive rowはRegistryに存在できるが、current Contract Set／Manifest／typed recordから参照してはならず、set equalityの右辺へ水増ししない。`authority_source=architecture_document`ではManifestを生成したapproved Architecture Document Registryの`document_id`、`authority_source=project`ではexact Project identityと一致させる。prefix、file path、display name、MCD rationaleからAuthorityを推測しない。

`DiagnosticOwnerLocalRefV1`、`RuntimeScopeOwnerRefV1`、`GameSystemOwnerRefV1`を持つ全local／external recordも同じClosure内のselected active Owner rowへID／revision／identity hashをexact解決し、Game Systemでは`owner_layer`も一致させる。Game System共通Envelopeの`owners[]`は単一`owner_ref.owner_id`とexact equalityにする。missing／multiple／deprecated／removed Owner、同IDのstale revision、同identityで別layer／authority、ManifestとRegistryのOwner差をClosure compile failureにする。retained Contract Setは同時代のretained Owner Registry／Closureで監査できるが、current dispatchへ流用しない。このcross-root resolutionはOwner refをContract Set hash preimageへ戻さず、Closure検証後の受入predicateとして実行する。

Foundation closureは[Product Plan §11](product-execution-registry-proposal.md#11-product-execution-registries)のcurrent `CurrentControlPlaneBaselineBindingV1`が解決するBaseline／Rebaseline Coreへref／hashで束縛し、その後にApproval／Envelope／Transaction、Product current CASへ進む。別系統のTrust authorityは六Registryからexact 15-member `TrustRegistryClosureV1`を生成し、Foundation closureへ混入しない。Control PlaneはFoundation closureとTrust closureの双方を検証するが、互いをhash preimageへ戻さない。

Digest、Derived Artifact ref、共通Manifest envelopeの構造正本を本節だけへ固定する。

```text
Sha256DigestV1
  wire: exact 32 bytes
  JSON/text projection: lowercase hexadecimal exact 64 characters

ArtifactRefV1
  artifact_kind: canonical string
  schema_version: positive uint32
  sha256: Sha256DigestV1

ManifestArtifactEnvelopeV1
  manifest_ref: ArtifactRefV1
  payload_root_sha256: Sha256DigestV1
  entry_count: uint32
  trust_profile_ref: McdContractRefV1(kind=profile)
```

`ArtifactRefV1.sha256`は完成immutable artifact bytesのSHA-256だけであり、path、URI、`latest`、content size、format major／minor、後段signatureを持たない。format metadataとcontent sizeはartifact kind固有のManifest、payload全体のroot／entry count／Trust bindingは`ManifestArtifactEnvelopeV1`へ置く。Manifest自身をpayload entryへ含めず、Envelope自身のref／signatureをEnvelopeへ戻さない。Domain文書はこれらexact型を消費し、`content_sha256`等の別名、追加Fieldを持つ同名V1、digest表示形式ごとのsibling型を再定義しない。

`schema_version`という名のFieldはMCD全域で`uint32`固定とし、本規則を全Domain文書共通の正本とする。SemVer等の互換表現が必要な場合は`schema_version`へ文字列を載せず、`format_version`等の別名Fieldを別途定義する。

`status=deprecated`は新規利用を拒否するが、offline migratorが旧Projectを読むための入力Schemaだけに残せる。Runtime、Editor、Game codeへdeprecated branchを生成しない。`retired`はcurrent Contract setの生成対象外である。

### 8.1 Project Runtime Entry／Runtime Scopeのtarget Operation候補

[Project state](../03-authoring/project-state.md#312-runtime-entryのclosed-operation-catalog)が所有する六Operationだけを、将来のatomic materialization candidateとして次の`McdOperationContractV1`へ閉じる。六recordは全Fieldを明示し、Registry compilerは既定値、別行参照、説明語だけのerror、裸IDを補完しない。対応するMCD／Schema／compiler／artifactは未materializeで、current Operation集合へ登録しない。materialized sourceを持たないRoot Scene／Runtime Scope migration Operationはtarget candidateにも登録しない。

```text
McdOperationContractV1
  MCD common envelope: §5の全Field
  operation_kind: query | command | event | job
  input_type: McdContractRefV1(kind=type)
  output_type: McdContractRefV1(kind=type)
  authority: TrustedServiceRefV1 {service_id, service_version, service_content_hash}
  risk_class: R0 | R1 | R2 | R3 | R4 | R5
  side_effects[0..16]: closed SideEffectKindV1
  idempotency: pure | idempotent_with_key | non_idempotent
  transaction: none | read_snapshot | authoring_changeset | source_promotion
  preconditions[1..16]: McdContractRefV1(kind=policy)
  postconditions[1..16]: McdContractRefV1(kind=policy)
  errors[1..64]: DiagnosticCodeRefV1
  validator_closure_ref: OperationValidatorClosureRefV1
  timeout_ms: uint32
  rate_limit_policy: McdContractRefV1(kind=policy)
  audit_level: metadata | full_redacted | restricted
  provider_exposure: none | mcp_proposal | trusted_internal
  receipt_type: McdContractRefV1(kind=type)

SideEffectKindV1 =
  authoring | source | network | process | release

TrustedServiceRefV1
  service_id
  service_version: uint32
  service_content_hash: SHA-256

TrustedServiceLocalRefV1
  service_id
  service_version: uint32

TrustedServiceLocalRecordV1
  local_ref: TrustedServiceLocalRefV1
  executable_identity_ref/hash
  allowed_operation_local_refs[1..4096]: ContractSetLocalRefV1(kind=operation)
  authority_capability_local_refs[1..64]: ContractSetLocalRefV1(kind=capability)
  isolation_profile_local_ref: ContractSetLocalRefV1(kind=profile)
  service_local_content_hash: SHA-256

TrustedServiceRecordV1
  service_id
  service_version: uint32
  executable_identity_ref/hash
  allowed_operation_refs[1..4096]: McdContractRefV1(kind=operation)
  authority_capability_refs[1..64]: McdContractRefV1(kind=capability)
  isolation_profile_ref: McdContractRefV1(kind=profile)
  service_content_hash: SHA-256

TrustedServiceRegistryV1
  registry_id: trusted_service.registry.active
  registry_version: 1
  registry_content_hash: SHA-256
  records[1..1024]: TrustedServiceLocalRecordV1

ProductionOwnerLayerV1 =
  core | feature_pack | genre_pack | game_project

OperationCompositionRoleV1 =
  generic_core_baseline | installed_core_extension |
  reusable_feature_pack | genre_pack | game_project

OwnerOperationContributionRefV1
  contribution_id
  contribution_version: positive uint32
  contribution_content_hash: SHA-256

OwnerOperationContributionV1
  contribution_id
  contribution_version: positive uint32
  owner_ref: OwnerIdentityLocalRefV1
  production_owner_layer: ProductionOwnerLayerV1
  authority_source:
    architecture_document:
      document_id
    | project:
      project_id
  operation_local_refs[1..4096]:
    ContractSetLocalRefV1(kind=operation)
  contribution_content_hash: SHA-256

OperationCompositionEntryV1
  operation_local_ref: ContractSetLocalRefV1(kind=operation)
  owner_ref: OwnerIdentityLocalRefV1
  production_owner_layer: ProductionOwnerLayerV1
  composition_role: OperationCompositionRoleV1
  contribution_origin_ref: OwnerOperationContributionRefV1

GenericCoreOperationBaselineV1
  artifact_role: production
  baseline_id: operation_baseline.generic_engine_core.bootstrap
  baseline_version: positive uint32
  entry_count: uint32
  entries[1..4096]: OperationCompositionEntryV1
  baseline_content_hash: SHA-256

GenericCoreOperationBaselineRefV1
  baseline_id: operation_baseline.generic_engine_core.bootstrap
  baseline_version: positive uint32
  baseline_content_hash: SHA-256

InstalledProductOperationCompositionV1
  artifact_role: cross_cutting_control_plane
  composition_id: operation_composition.product.current
  composition_version: positive uint32
  generic_core_baseline_ref: GenericCoreOperationBaselineRefV1
  entry_count: uint32
  entries[1..4096]: OperationCompositionEntryV1
  composition_content_hash: SHA-256

InstalledProductOperationCompositionRefV1
  composition_id: operation_composition.product.current
  composition_version: positive uint32
  composition_content_hash: SHA-256

OperationValidatorClosureRefV1
  closure_id
  closure_version: uint32
  closure_content_hash: SHA-256

OperationValidatorClosureLocalRefV1
  closure_id
  closure_version: uint32

ValidatorLocalRefV1
  validator_id
  validator_version: uint32

DiagnosticLocalRefV1
  diagnostic_id
  diagnostic_version: uint32

DiagnosticLocalRecordV1
  diagnostic_local_ref: DiagnosticLocalRefV1
  owner_local_ref: DiagnosticOwnerLocalRefV1
  code: MIRAKAN-<DOMAIN>-<CONDITION>
  severity: info | warning | error | blocking
  category:
    schema | semantic | permission | conflict | build | test |
    performance | security | provider | infrastructure
  message_key
  requirement_local_refs[0..64]:
    ContractSetLocalRefV1(kind=requirement)
  retryability: never | after_input | after_change | transient
  diagnostic_local_content_hash: SHA-256

ValidatorRecordRefV1
  validator_id
  validator_version: uint32
  validator_content_hash: SHA-256

ValidatorLocalRecordV1
  validator_local_ref: ValidatorLocalRefV1
  implementation_artifact_ref/hash
  input_type_local_ref: ContractSetLocalRefV1(kind=type)
  error_local_refs[1..64]: DiagnosticLocalRefV1
  validator_local_content_hash: SHA-256

ValidatorRecordV1
  validator_id
  validator_version: uint32
  materialized_from_contract_set_hash: SHA-256
  implementation_artifact_ref/hash
  input_type_ref: McdContractRefV1(kind=type)
  error_refs[1..64]: DiagnosticCodeRefV1
  validator_content_hash: SHA-256

OperationValidatorClosureLocalRecordV1
  closure_local_ref: OperationValidatorClosureLocalRefV1
  operation_local_ref: ContractSetLocalRefV1(kind=operation)
  validator_local_refs[1..64]: ValidatorLocalRefV1
  reachable_error_local_refs[1..64]: DiagnosticLocalRefV1
  reachable_error_set_hash: SHA-256
  closure_local_record_hash: SHA-256

OperationValidatorClosureV1
  closure_id
  closure_version: uint32
  operation_ref: McdContractRefV1(kind=operation)
  validator_refs[1..64]: exact ValidatorRecordRefV1
  reachable_error_refs[1..64]: DiagnosticCodeRefV1
  reachable_error_set_hash: SHA-256
  closure_content_hash: SHA-256

OperationPreconditionEvaluationInputV1
  operation_ref: McdContractRefV1(kind=operation)
  operation_input_ref/hash
  operation_intent_hash
  request_hash
  before_snapshot_refs[1..64]/hashes
  mutation_authorization_binding:
    exact MutationAuthorizationBindingV1
    | canonical omission: only R0/R1 non-mutation

PreparedCandidateRefV1
  candidate_id
  candidate_schema_ref: McdContractRefV1(kind=type)
  candidate_content_hash: SHA-256

PreparedCandidateV1
  candidate_id
  candidate_schema_ref: McdContractRefV1(kind=type)
  staging_root_hash
  before_state_ref/hash
  proposed_after_state_ref/hash
  prepared_artifact_refs[0..4096]/hashes
  candidate_content_hash: SHA-256

PublishedReceiptPublicationControlPolicyRefV1
  policy_id
  policy_version: positive uint32
  policy_kind:
    verification_retention |
    deterministic_recovery |
    receipt_store_namespace
  policy_content_hash: SHA-256

PublishedReceiptPublicationControlPolicyV1
  policy_id
  policy_version: positive uint32
  policy_kind:
    verification_retention |
    deterministic_recovery |
    receipt_store_namespace
  payload:
    verification_retention:
      historical_retention_rule:
        retain_while_reachable_from_any_retained_root
      retained_closure_kinds[4]: [
        authorization_audit_closure
        receipt_publication_closure
        schema_definition_closure
        trust_verification_closure
      ]
      prepublication_private_retention_rule:
        retain_until_public_marker_or_durable_terminal_abort
      private_key_retention_required: false
      resign_on_key_loss: forbidden
      publication_on_retention_preflight_failure: forbidden
    | deterministic_recovery:
      allowed_transitions[2]: [
        private_marker_to_byte_identical_signed_wrapper
        signed_wrapper_to_expected_predecessor_public_marker
      ]
      signature_identity:
        same_materialization_key_context_subject_and_key
      broker_journal_commit: before_signature_return
      fresh_clock_read: forbidden
      fresh_key_selection: forbidden
      policy_or_trust_substitution: forbidden
      alternate_signature: forbidden
      issued_at_update: forbidden
      post_publication_rollback: forbidden
      missing_or_mismatched_state: fail_closed
    | receipt_store_namespace:
      namespace_id:
        urn:mirakan:receipt-store:namespace:published-domain-receipt:v1
      namespace_version: 1
      key_derivation:
        sha256_completed_published_domain_receipt_wrapper_bytes
      write_semantics: put_if_absent_byte_equality
      overwrite: forbidden
      readback_before_public_marker: required
      deletion_control: verification_retention_policy_only
      backend_binding: qualified_deployment_private_binding
      locator_or_credentials_in_record: forbidden
  policy_content_hash: SHA-256

PublishedReceiptMaterializationPolicyV1
  materialization_policy_version: 1
  operation_ref: McdContractRefV1(kind=operation)
  execution_authority_ref: TrustedServiceRefV1
  signed_record_purpose: operation_domain_receipt
  operation_receipt_signer_policy_ref:
    exact OperationReceiptSignerPolicyRefV1
  trust_registry_closure_head_ref/hash
  governance_policy_config_registry_ref/hash
  trust_registry_closure_ref/hash
  signing_profile_ref/hash: PublishedReceiptSigningProfileRefV1
  signer_subject_ref/hash
  signer_role_ref/hash
  key_id
  public_key_registry_snapshot_ref/hash
  verification_retention_policy_ref:
    exact PublishedReceiptPublicationControlPolicyRefV1(
      policy_id=published_receipt_verification_retention_policy.active,
      policy_kind=verification_retention)
  recovery_policy_ref:
    exact PublishedReceiptPublicationControlPolicyRefV1(
      policy_id=published_receipt_recovery_policy.active,
      policy_kind=deterministic_recovery)
  receipt_canonical_schema_ref: McdContractRefV1(kind=type)
  receipt_store_namespace_policy_ref:
    exact PublishedReceiptPublicationControlPolicyRefV1(
      policy_id=published_receipt_store_namespace_policy.active,
      policy_kind=receipt_store_namespace)
  policy_content_hash: SHA-256

PublishedReceiptMaterializationPolicyRefV1
  operation_ref: McdContractRefV1(kind=operation)
  policy_artifact_ref:
    ArtifactRefV1(
      artifact_kind=receipt.published_receipt_materialization_policy,
      schema_version=1)
  policy_content_hash: SHA-256

PublishedReceiptSigningProfileRefV1
  profile_id
  profile_version: positive uint32
  profile_content_hash: SHA-256

PublishedReceiptSigningProfileV1
  profile_id: signing_profile.operation_receipt.ecdsa_p256_rfc6979
  profile_version: 1
  signature_algorithm: ecdsa-p256-sha256
  signature_format: ieee-p1363-raw
  nonce_derivation: rfc6979-sha256
  low_s_required: true
  canonical_subject: JCS(PublishedDomainReceiptPayloadV1)
  issued_at_source: exact pre-marker materialization context issued_at
  revocation_snapshot_source:
    exact pre-marker materialization context revocation_snapshot_ref/hash
  retry_rule: same subject/key/context produces byte-identical wrapper
  profile_content_hash: SHA-256

OperationAuthorizationAuditBindingV1
  binding_version: 1
  operation_ref: McdContractRefV1(kind=operation)
  input_type_ref: McdContractRefV1(kind=type)
  risk_class: R2 | R3 | R4 | R5
  operation_intent_hash: SHA-256
  request_hash: SHA-256
  request_hash_algorithm_binding:
    exact OperationRequestAlgorithmBindingV1
  request_input_content_ref:
    ArtifactRefV1(
      artifact_kind=operation_request_input,
      schema_version=1)
  authorization_ref: TaskAuthorizationEnvelopeRefV1
  authorization_hash: SHA-256
  authority_evidence:
    approval:
      approval_set_ref: OperationMutationApprovalSetRefV1
      approval_set_hash: SHA-256
    | predelegated:
      predelegation_ref: OperationPredelegationUseRefV1
      predelegation_hash: SHA-256
  binding_content_hash: SHA-256

OperationAuthorizationAuditBindingRefV1
  operation_ref: McdContractRefV1(kind=operation)
  request_hash: SHA-256
  binding_artifact_ref:
    ArtifactRefV1(
      artifact_kind=receipt.operation_authorization_audit_binding,
      schema_version=1)
  binding_content_hash: SHA-256

PublishedReceiptMaterializationContextRefV1
  context_id
  context_content_hash: SHA-256

PublishedReceiptMaterializationContextV1
  context_id
  operation_ref: McdContractRefV1(kind=operation)
  operation_intent_hash
  request_hash
  request_hash_algorithm_binding:
    exact OperationRequestAlgorithmBindingV1
  idempotency_key
  before_state_ref/hash
  staged_after_state_ref/hash
  issued_at
  revocation_snapshot_ref/hash
  operation_receipt_signer_policy_ref:
    exact OperationReceiptSignerPolicyRefV1
  trust_registry_closure_head_ref/hash
  governance_policy_config_registry_ref/hash
  trust_registry_closure_ref/hash
  authorization_audit_binding_ref:
    exact OperationAuthorizationAuditBindingRefV1
  materialization_policy_ref:
    exact PublishedReceiptMaterializationPolicyRefV1
  signing_profile_ref/hash
  signer_subject_ref/hash
  signer_role_ref/hash
  key_id
  public_key_registry_snapshot_ref/hash
  context_content_hash: SHA-256

PublishedReceiptMaterializationKeyPayloadV1
  operation_ref: McdContractRefV1(kind=operation)
  operation_intent_hash
  request_hash
  idempotency_key
  before_state_ref/hash
  staged_after_state_ref/hash
  receipt_type_ref: McdContractRefV1(kind=type)
  prepared_payload_count: 3
  prepared_payloads[3]:
    exact {payload_type_ref, payload_content_ref, payload_content_hash}
  private_commit_marker_hash
  materialization_policy_ref:
    exact PublishedReceiptMaterializationPolicyRefV1
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1

AtomicCommitPlanPayloadV1
  operation_ref: McdContractRefV1(kind=operation)
  operation_intent_hash
  request_hash
  idempotency_key
  prepared_candidate_ref: PreparedCandidateRefV1
  project_source_promotion_authorization_binding_refs[0..1]:
    ProjectSourcePromotionAuthorizationBindingRefV1
  before_state_ref/hash
  proposed_after_state_ref/hash
  prepared_payload_count: 3
  prepared_payloads[3]:
    exact {payload_type_ref, payload_content_ref, payload_content_hash}
  published_receipt_materialization_policy_ref:
    exact PublishedReceiptMaterializationPolicyRefV1
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1

PreparedCommitEnvelopeV1
  operation_ref: McdContractRefV1(kind=operation)
  operation_intent_hash
  request_hash
  idempotency_key
  prepared_candidate_ref: PreparedCandidateRefV1
  project_source_promotion_authorization_binding_refs[0..1]:
    ProjectSourcePromotionAuthorizationBindingRefV1
  before_state_ref/hash
  staged_after_state_ref/hash
  preview_receipt_payload_ref/hash
  validation_receipt_payload_ref/hash
  prepared_domain_receipt_payload_ref/hash
  published_receipt_materialization_policy_ref:
    exact PublishedReceiptMaterializationPolicyRefV1
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1
  atomic_commit_plan_hash
  envelope_hash

PreparedReceiptPublicationBindingV1
  operation_ref: McdContractRefV1(kind=operation)
  operation_intent_hash
  request_hash
  request_hash_algorithm_binding:
    exact OperationRequestAlgorithmBindingV1
  idempotency_key
  project_source_promotion_authorization_binding_refs[0..1]:
    ProjectSourcePromotionAuthorizationBindingRefV1
  before_state_ref/hash
  staged_after_state_ref/hash
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1

PreparedPreviewReceiptPayloadV1
  publication_binding: exact PreparedReceiptPublicationBindingV1
  preview_result_ref/hash
  disposition: preview_ready
  prepared_payload_hash: SHA-256

PreparedValidationReceiptPayloadV1
  publication_binding: exact PreparedReceiptPublicationBindingV1
  validation_result_ref/hash
  disposition: satisfied
  diagnostic_refs: []
  prepared_payload_hash: SHA-256

OperationPostconditionEvaluationInputV1
  operation_ref: McdContractRefV1(kind=operation)
  request_hash
  prepared_operation_output_ref/hash
  before_snapshot_refs[1..64]/hashes
  unpublished_staging_snapshot_refs[1..64]/hashes
  prepared_commit_envelope_ref/hash

StagedPostconditionReceiptV1
  evaluated_input_hash
  prepared_candidate_ref: PreparedCandidateRefV1
  prepared_commit_envelope_ref/hash
  disposition: satisfied | rejected
  predicate_evidence_hash | diagnostics[1..64]
  receipt_payload_hash

PrivateDurableCommitMarkerV1
  marker_id
  operation_ref
  operation_intent_hash
  request_hash
  idempotency_key
  prepared_commit_envelope_ref/hash
  before_state_ref/hash
  staged_after_state_ref/hash
  staged_prepared_receipt_payload_count: 3
  staged_prepared_receipt_payloads[3]:
    exact {payload_type_ref, payload_content_ref, payload_content_hash}
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1
  staged_postcondition_receipt_ref/hash
  visibility: private_internal
  marker_hash

PublicCommitClosureV1
  closure_id
  closure_version: positive uint32
  operation_ref
  operation_intent_hash
  request_hash
  idempotency_key
  prepared_candidate_ref: PreparedCandidateRefV1
  project_source_promotion_authorization_binding_refs[0..1]:
    ProjectSourcePromotionAuthorizationBindingRefV1
  domain_commitment:
    kind: project_change_set_commit
      project_change_set_ref: ProjectChangeSetArtifactRefV1
      candidate_root_sha256: Sha256DigestV1
    | kind: owner_typed_state_commit
      domain_owner_ref: OwnerIdentityLocalRefV1
      committed_artifact_refs[1..4096]: ArtifactRefV1
  before_state_ref/hash
  public_after_state_ref/hash
  prepared_commit_envelope_hash
  private_commit_marker_hash
  closure_content_hash

PublicCommitClosureRefV1
  closure_id
  closure_version: positive uint32
  closure_ref:
    exact ContentAddressedRefV1(type=PublicCommitClosureV1)
  closure_content_hash

PublishedDomainReceiptPayloadV1
  prepared_domain_receipt_payload_ref/hash
  private_commit_marker_hash
  public_commit_closure_ref: PublicCommitClosureRefV1
  public_commit_closure_hash
  operation_ref
  operation_intent_hash
  request_hash
  idempotency_key
  before_state_ref/hash
  after_state_ref/hash
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1
  issued_at
  revocation_snapshot_ref/hash

PublishedDomainReceiptV1
  payload: PublishedDomainReceiptPayloadV1
  signed_record:
    exact MirakanSignedRecordV1

PublicPublicationMarkerV1
  publication_id
  operation_ref
  operation_intent_hash
  request_hash
  idempotency_key
  private_commit_marker_hash
  public_commit_closure_ref: PublicCommitClosureRefV1
  public_commit_closure_hash
  signed_domain_receipt_ref/hash
  before_state_ref/hash
  public_after_state_ref/hash
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1
  expected_previous_publication_ref/hash | null
  publication_sequence: positive uint64
  marker_hash

PublicPublicationMarkerRefV1
  publication_id
  publication_sequence: positive uint64
  marker_ref:
    exact ContentAddressedRefV1(type=PublicPublicationMarkerV1)
  marker_hash: SHA-256

OperationPredicateResultV1
  disposition: satisfied | rejected
  satisfied:
    evaluated_input_hash
    predicate_evidence_hash
  rejected:
    diagnostics[1..64]: DiagnosticCodeRefV1

OperationRateLimitPolicyV1
  policy_ref: McdContractRefV1(kind=policy)
  scope: project | principal_project
  window_ns: uint64
  max_requests: uint32
  burst: uint32
  exceeded_error_ref: DiagnosticCodeRefV1
```

`PublicCommitClosureV1`はprivate Marker readback後にだけ作るsecret-freeな公開commit commitmentであり、private Marker body、Prepared Envelope body、Authorization／Approval body、Staging payload、PII、secret locatorを含まない。`domain_commitment`はclosed tagged unionである。`project_change_set_commit`はProject Stateのimmutable `ProjectChangeSetArtifactRefV1`と、そのCommit／Build／Packageが共通利用する`candidate_root_sha256`を必須にし、Source registration primitiveが0件でlate binding集合が`[]`でもこのbranchと二Fieldを省略しない。`owner_typed_state_commit`はProject ChangeSetを使わないWorld／Pack／Registry／Runtime mutationだけに使い、current Foundation Closureのselected-active `domain_owner_ref`と、private Markerが束縛したreceipt-free staged after stateからpublic CASで到達可能化するexact `committed_artifact_refs[]`をArtifact ref canonical byte順のstrict sorted setとして持つ。`operation.authoring.changeset.commit`のProject ChangeSet Commitをowner-typed branchへdowngradeすること、表示名やafter-state hashからChangeSet／Candidate rootを推測すること、Receipt／Public Marker／Closure自身をcommitted artifact集合へ含めることを拒否する。

semantic hashと完成object hashを分離する。`closure_id`と`closure_content_hash`の両Fieldをcanonical omissionしたclosed recordをMCD canonical encodeしたbytesを`S`とし、`D = SHA-256(ASCII "MIRAKAN_PUBLIC_COMMIT_CLOSURE_V1" || uint32_be(len(S)) || S)`、`closure_content_hash=D`、`closure_id=urn:mirakan:public-commit-closure:sha256:<lowercase-hex(D)>`の順で導出する。両Fieldを挿入した完成`PublicCommitClosureV1`のRFC 8785 JCS bytesを`C`とし、`closure_object_sha256=SHA-256(C)`を別に計算する。`PublicCommitClosureRefV1.closure_content_hash`はsemantic `D`、`closure_ref.sha256`と`PublishedDomainReceiptPayloadV1`／`PublicPublicationMarkerV1`の隣接`public_commit_closure_hash`は完成object SHA `closure_object_sha256`へbyte equalityにする。semantic hashと完成object SHAの等値、`closure_id`のsemantic preimageへの自己混入、completed-object hashを`closure_content_hash`へ複写する実装を拒否する。

`prepared_candidate_ref`とlate binding集合はPrepared Envelope、before／after stateはprivate Marker、`prepared_commit_envelope_hash`と`private_commit_marker_hash`はreadbackした各完成objectへ一致させる。Project branchのChangeSet／Candidate rootはProject StateのCommit inputおよびCore `BuildProjectPublicationBindingV1(kind=committed_revision)`とbyte equalityにし、owner-typed branchのOwner／Artifact集合は選択Domainのprepared payloadとstaged after stateへexact解決する。Closureはprivate objectをAuthorityへ変換せず、Package等の認可済みconsumerが公開Markerと署名済みReceiptから、Commitが束縛したCandidate、domain commitment、late binding、before／after、Envelope commitmentを型付きで再検証するためだけに使う。

Gatewayはprivate Marker readback後、完成`PublicCommitClosureV1`をimmutable content-addressed public-candidate storeへput-if-absentし、同Closure Ref／完成object hashを`PublishedDomainReceiptPayloadV1`へ入れて署名する。signed wrapper readback後のpublic CASでは、同じClosure body、`PublicPublicationMarkerV1`、Receipt-free after stateをatomic publishし、Marker、signed Receipt payload、ClosureのRef／semantic hash／完成object hash、Operation、request、idempotency、private-marker commitment、domain commitment、before／after stateをbyte equalityにする。Closure bodyまたはsigned Receiptを欠くMarker、別Closureへの差替え、Closureだけの先行public authority、private Marker body／Prepared Envelope bodyの公開昇格を拒否する。これにより公開検証DAGは`PublicPublicationMarkerRefV1 -> PublicPublicationMarkerV1 -> signed Domain Receipt + PublicCommitClosureRefV1 -> PublicCommitClosureV1 -> PreparedCandidateRefV1 + domain_commitment`となり、private audit DAGとはhash commitmentだけで一方向に接続する。

`PublicCommitClosureV1`、`PublicCommitClosureRefV1`、その`domain_commitment` unionはtarget Published Domain Receipt／Public Publication Marker root schema内のnested common `$defs`であり、standalone MCD Type、Operation、Policy、Contract Set memberではない。したがってtarget-complete candidateのGeneric Core六件／Installed Product十件に追加せず、current四Operation集合は引き続きexact `[]`である。意味Field追加により将来materialized root schema bytes／content hashを更新する場合も同じDefinition／Control Plane transactionで再発行し、Closure名を理由にOperation、Type、Local Schema Catalog rowまたはOwner Manifest contributionを一件加算しない。

`DiagnosticOwnerLocalRefV1.owner_content_hash`はASCII `MIRAKAN_DIAGNOSTIC_OWNER_LOCAL_IDENTITY_V1`、`uint32_be(len(NFC UTF-8 owner_id)) || owner_id bytes`、`uint32_be(8) || uint64_be(owner_revision)`の順から計算し、Gameplay Programming ModelのOwner identity式とbyte equalityにする。これはContract set rootを含まないimmutable local identityであり、Owner identity自身もContract set hash、Diagnostic ref、Diagnostic Registry hashを含めない。`DiagnosticLocalRecordV1.diagnostic_local_content_hash`はASCII `MIRAKAN_DIAGNOSTIC_LOCAL_RECORD_V1`と、同Fieldだけを除きOwner identityとRequirement edgeをLocalRefのまま保持した完成local recordのlength-framed canonical bytesから計算する。Contract setのDiagnostic `member_hash`はこの完成local recordを別domain `MIRAKAN_CONTRACT_SET_DIAGNOSTIC_MEMBER_V1`でhashする別値である。set root確定後、外部`DiagnosticCodeRecordV1.diagnostic_content_hash`はOwnerをexact external owner ref、Requirement edgeを`contract_set_hash`付き`McdContractRefV1`へ投影した外部recordから§12の規則で計算する第三の値であり、local hashまたはmember hashとの等値を要求しない。外部record／hash／refをlocal payload、member hash、set rootへfeed backしない。

`TrustedServiceLocalRecordV1.service_local_content_hash`はASCII `MIRAKAN_TRUSTED_SERVICE_LOCAL_RECORD_V1`と同Fieldだけを除く完成local recordから計算する。Contract Set `member_hash`は別domain `MIRAKAN_CONTRACT_SET_TRUSTED_SERVICE_MEMBER_V1`と、このlocal hashを含む完成local recordから計算する別値であり、両hashのbyte equalityを要求しない。Local Registry hashはASCII `MIRAKAN_TRUSTED_SERVICE_REGISTRY_V1`、Registry ID／version、record count、`service_id`／version順local record bytesを`uint32_be` length framingして計算する。set root確定後だけ、Operation／Capability／isolation profileのLocalRefをroot付きMCD refへ投影して`TrustedServiceRecordV1`を作り、`service_content_hash`をASCII `MIRAKAN_TRUSTED_SERVICE_RECORD_V1`と自己Fieldを除くexternal recordから計算する。`service_local_content_hash → Service member_hash／Local Registry hash → Contract set root → external service_content_hash／TrustedServiceRefV1`の一方向DAGとし、external record／hash／refをlocal record、member、Registry、rootへ戻さない。duplicate／stale／hash mismatch、OperationのLocalRefがServiceの`allowed_operation_local_refs[]`にない状態を拒否する。Generic Engine Core bootstrap baselineの一Service projectionは次であり、`executable_identity_ref/hash`、`isolation_profile_local_ref`、Capability LocalRef、allowlistを省略しない。

| Service LocalRef | executable identity | exact allowed Operation LocalRefs | authority Capability LocalRefs | isolation profile LocalRef |
|---|---|---|---|---|
| `{service.authoring_command_gateway,1}` | `{artifact.service.authoring_command_gateway,1,artifact_hash}` | `{operation.project.runtime_entry.create,1}; {operation.project.runtime_entry.update,1}; {operation.project.runtime_entry_activation_policy.create,1}; {operation.project.runtime_entry_activation_policy.update,1}; {operation.project.runtime_target_selector.create,1}; {operation.project.runtime_target_selector.update,1}` | `{capability.authoring.command_gateway,1}` | `{profile.isolation.authoring_command_gateway,1}` |

表のOperation集合は本節の初期六Operationを導入するtarget bootstrap seedであり、current Installed Product snapshotではない。他Domain／Pack由来Operationを暗黙追加せず、§8.2の十entry Compositionもtarget candidateとしてだけ使用する。外部`TrustedServiceRefV1 {service_id,service_version,service_content_hash}`はset root確定後にmaterializeするだけでSnapshot preimageへ戻さない。将来DomainまたはPack Operationをactivate／removeするContract set transactionは、Owner ManifestのOperation LocalRef集合と当該OwnerがServiceへ寄与するallowlist LocalRef集合のset equalityを検査し、Service local record、Operation local record、set rootを一方向に再生成する。runtime中のallowlist mutation、旧Service hashと新Operationの混在、prefix／owner名による暗黙許可を禁止する。

`OwnerOperationContributionV1`は既存Owner Manifest contributionの型付き正規Envelopeであり、新しいOwner Authorityまたは追加MCD memberではない。`contribution_content_hash = SHA-256(ASCII "MIRAKAN_OWNER_OPERATION_CONTRIBUTION_V1" || uint32_be(len(RFC 8785 JCS(closed contribution excluding contribution_content_hash))) || RFC 8785 JCS(closed contribution excluding contribution_content_hash))`とする。JCS projectionはkeyをUnicode code point順、全stringをNFC、positive uint32をsafe JSON integer、`owner_revision`をcanonical unsigned decimal string、SHA-256をlowercase hexadecimal exact 64文字にする。`ContractSetLocalRefV1`は`{id,kind,version}`を全Field保存し、本文の二Field短記から`kind=operation`を補完してhash入力を変えない。`operation_local_refs[]`はOperation IDのNFC UTF-8 bytes、version `uint32_be`順へstrict sort／uniqueし、`OwnerOperationContributionRefV1`の三Fieldを解決先recordとbyte equalityにする。`owner_ref`は同じFoundation Closureのselected-active Owner rowへ解決し、`production_owner_layer`はOwner rowの`core | feature_pack | genre_pack | project`をそれぞれ`core | feature_pack | genre_pack | game_project`へexact写像する。`authority_source`も同Owner rowとbyte equalityにし、Owner ID prefix、文書path、Operation ID namespaceからlayerまたはsourceを推測しない。

`OperationCompositionEntryV1`の`owner_ref`、`production_owner_layer`、`operation_local_ref`は`contribution_origin_ref`が解決する同一`OwnerOperationContributionV1`に含まれなければならない。Origin refは同じcompile transactionへ入力されたimmutable Owner Manifest contribution artifactをID／version／hashでexact解決し、global latest、file path、Owner IDだけの探索を禁止する。`composition_role`はProduct構成上の役割であり所有レイヤーとは別軸であるため、`owner.core.world`のようなCore所有の任意追加は`installed_core_extension`になり得るが、`genre_pack`所有を`generic_core_baseline`へ昇格できない。Baseline／CompositionのentriesはOperation ID／version順へstrict sort／uniqueし、`entry_count`を配列長と一致させる。`baseline_content_hash = SHA-256(ASCII "MIRAKAN_GENERIC_CORE_OPERATION_BASELINE_V1" || uint32_be(len(RFC 8785 JCS(closed baseline excluding baseline_content_hash))) || RFC 8785 JCS(closed baseline excluding baseline_content_hash))`、`composition_content_hash = SHA-256(ASCII "MIRAKAN_INSTALLED_PRODUCT_OPERATION_COMPOSITION_V1" || uint32_be(len(RFC 8785 JCS(closed composition excluding composition_content_hash))) || RFC 8785 JCS(closed composition excluding composition_content_hash))`とし、同じJCS projection規則を使う。

Baseline／Composition完成bytesはcompile transactionのimmutable content-addressed evidence namespaceへhash keyで保存し、各Refは同namespaceからID／version／hashを全件照合して解決する。承認済みactivation transactionは使用した二Refとcompleted bytesをretentionし、missing bytes、hash mismatch、bare current、別compile transactionからの同ID substitutionを拒否する。Generic Core baselineの`artifact_role=production`はCore bootstrap自身のproduction contract inventory、Installed Product compositionの`artifact_role=cross_cutting_control_plane`は複数Owner／layerを束ねるProduct／Control Plane projectionを表す。後者をGeneric Core production dependency edgeまたはCore ownershipへ投影しない。これら二recordはOwner Manifest／MCD／Serviceのset equalityを型付きで検査するcompiler evidenceで、Operation、Policy、Type、Contract Set memberまたはdispatch Authorityを追加せず、単独Refの存在から実行権限を生成しない。

次表はGeneric Engine Core operation baseline候補を説明するためのsymbolic hash名である。対応するOwner Registry、Contribution、Baseline Artifact、Contract compilerは未materializeであり、current hashまたは実行可能baselineではない。実Artifactを生成する場合は定義済みpreimageからhashを計算し、symbolic名をwireへ保存しない。

| symbolic name | materialization state |
|---|---|
| `project_state_owner_hash_v1` | `absent` |
| `world_owner_hash_v1` | `absent` |
| `shooter_owner_hash_v1` | `absent` |
| `project_state_contribution_hash_v1` | `absent` |
| `world_contribution_hash_v1` | `absent` |
| `shooter_contribution_hash_v1` | `absent` |
| `generic_core_baseline_hash_v1` | `absent` |
| `installed_product_composition_hash_v1` | `absent` |

六entryはすべて`contribution_origin_ref={contribution.owner.core.project_state.runtime_entry_bootstrap,1,project_state_contribution_hash_v1}`、`owner_ref={owner.core.project_state,1,project_state_owner_hash_v1}`、`production_owner_layer=core`、`composition_role=generic_core_baseline`を持つ。

次のrecordは`GenericCoreOperationBaselineV1`候補Schemaを説明する非materialized例であり、current instanceまたはSchemaの再定義ではない。

```text
GenericCoreOperationBaselineV1
  artifact_role = production
  baseline_id = operation_baseline.generic_engine_core.bootstrap
  baseline_version = 1
  entry_count = 6
  entries = [
    {operation_local_ref={operation.project.runtime_entry.create,1},
     owner_ref={owner.core.project_state,1,project_state_owner_hash_v1},
     production_owner_layer=core,
     composition_role=generic_core_baseline,
     contribution_origin_ref={contribution.owner.core.project_state.runtime_entry_bootstrap,1,project_state_contribution_hash_v1}}
    {operation_local_ref={operation.project.runtime_entry.update,1},
     owner_ref={owner.core.project_state,1,project_state_owner_hash_v1},
     production_owner_layer=core,
     composition_role=generic_core_baseline,
     contribution_origin_ref={contribution.owner.core.project_state.runtime_entry_bootstrap,1,project_state_contribution_hash_v1}}
    {operation_local_ref={operation.project.runtime_entry_activation_policy.create,1},
     owner_ref={owner.core.project_state,1,project_state_owner_hash_v1},
     production_owner_layer=core,
     composition_role=generic_core_baseline,
     contribution_origin_ref={contribution.owner.core.project_state.runtime_entry_bootstrap,1,project_state_contribution_hash_v1}}
    {operation_local_ref={operation.project.runtime_entry_activation_policy.update,1},
     owner_ref={owner.core.project_state,1,project_state_owner_hash_v1},
     production_owner_layer=core,
     composition_role=generic_core_baseline,
     contribution_origin_ref={contribution.owner.core.project_state.runtime_entry_bootstrap,1,project_state_contribution_hash_v1}}
    {operation_local_ref={operation.project.runtime_target_selector.create,1},
     owner_ref={owner.core.project_state,1,project_state_owner_hash_v1},
     production_owner_layer=core,
     composition_role=generic_core_baseline,
     contribution_origin_ref={contribution.owner.core.project_state.runtime_entry_bootstrap,1,project_state_contribution_hash_v1}}
    {operation_local_ref={operation.project.runtime_target_selector.update,1},
     owner_ref={owner.core.project_state,1,project_state_owner_hash_v1},
     production_owner_layer=core,
     composition_role=generic_core_baseline,
     contribution_origin_ref={contribution.owner.core.project_state.runtime_entry_bootstrap,1,project_state_contribution_hash_v1}}
  ]
  baseline_content_hash =
    generic_core_baseline_hash_v1
```

このbaselineのOrigin recordは`contribution_id=contribution.owner.core.project_state.runtime_entry_bootstrap`、`contribution_version=1`、上記六Operation LocalRef、`owner_ref={owner.core.project_state,1,project_state_owner_hash_v1}`、`production_owner_layer=core`、`authority_source.architecture_document.document_id=mirakan.arch.project-state`、`contribution_content_hash=project_state_contribution_hash_v1`をexact値にする。六Operationの追加／欠落／入替え、別Owner／layer／authority source、`composition_role!=generic_core_baseline`、Origin ref hash差をbaseline compile failureにする。

#### 8.1.1 Foundation service dependency MCD records

target Service candidateが参照するdependencyを、将来同じatomic materializationで次のexact四MCD local record候補へ追加する。四record共通のEnvelopeは`mcd_version=1; status=active; requirement_refs=[]; since_contract_set=1; supersedes=[]`であり、この`status`はmaterialized Contract Set内のtarget値であってcurrent Repository stateではない。残る共通Fieldも次表のliteralを省略せず保存する。本文中の`id@version`は可読用表記だけで、MCD `id`へ`@`またはversionを埋め込まない。

| kind | id | version | title | description | owners | rationale_refs | tags（ASCII昇順） |
|---|---|---:|---|---|---|---|---|
| `type` | `type.operation.mutation_authorization_binding` | 1 | `Mutation authorization binding` | `状態変更Operationのintent、risk、Authorization、ApprovalまたはPredelegationを閉じる型` | `[owner.core.security_approval]` | `[mirakan.arch.ai-security-approval#mutation-authorization-binding-v1]` | `[authorization, mutation, security]` |
| `capability` | `capability.runtime.scheduling` | 1 | `Runtime scheduling` | `Runtime scheduling contractの存在とTarget適用範囲` | `[owner.core.runtime]` | `[mirakan.arch.runtime-scheduling-lifetime]` | `[runtime, scheduling]` |
| `capability` | `capability.authoring.command_gateway` | 1 | `Authoring command gateway authority` | `Authoring Command Gatewayが完全登録済みOperationを処理するauthority capability` | `[owner.core.project_state]` | `[mirakan.arch.executable-contracts#81-project-runtime-entryruntime-scopeのtarget-operation候補]` | `[authoring, gateway, service]` |
| `profile` | `profile.isolation.authoring_command_gateway` | 1 | `Authoring command gateway isolation` | `信頼済みAuthoring Command Gatewayの最小権限隔離profile` | `[owner.core.security_approval]` | `[mirakan.arch.ai-security-approval#73-trusted-service-isolation]` | `[authoring, isolation, service]` |

`type.operation.mutation_authorization_binding@1`のpayload schemaは[AI Security／Approvalのcanonical `MutationAuthorizationBindingV1`](../01-governance/ai-security-approval.md#mutation-authorization-binding-v1)だけを正本とする。本書へSchema blockをmirrorせず、Contract projectionは正本Schemaのcontent referenceを入力に生成する。生成器とSchema Artifactが存在しないため、このTypeをcurrent materialized MCDと表現せず、target candidateに限定する。

typed Refとwrapper、署名purpose、Task／Project／typed subject scope grant、intent-bound Approval Set／Predelegation Use、Grant Registry／Consumption Head、current Projector binding exact `[]`の構造正本は[AI Security／ApprovalのMutation authorization binding V1](../01-governance/ai-security-approval.md#mutation-authorization-binding-v1)である。Approval Set隣接hashはRefの`set_content_hash`、Set内各Approval、Grant、Useの隣接hashは各Refの`wrapper_sha256`とbyte equalityにする。R2は`approval | predelegated`のexact一branch、R3～R5は`approval`だけを受理する。裸ref、個別Approval一件、Predelegation Grant単体、別purpose wrapper、scopeだけ一致する別intent、`request_hash_algorithm_binding`欠落を拒否する。本target MCD Type candidateは同節のPayload／Wrapper／Ref、Approval quorum／Set、Project scope、Consumption Head／Scope Projection、`OperationRequestAlgorithmBindingV1`、それらが使うsmall wire shapeを同record schemaのclosed `$defs`として完全投影し、Wrapperの署名Fieldだけを既存`MirakanSignedRecordV1`へexact `$ref`する。CompilerはAI Security正本とのfield ID／type／presence／bound／purpose equalityを検査する。三signed wrapperのstandalone validationはAI Securityが所有するexact Local Schema Catalog root ID／signature slotを使い、nested `$defs`の存在を署名Authorityへ代用しない。これらは追加MCD Type recordまたはArchitecture Governance Ownerの追加Control Plane top-level contractではないため、target Foundation correctionのMCD差分`Type +1`と同Ownerのcontract exact 30を増やさない。initial candidate IDは`type.operation.mutation_authorization_binding@1`、`supersedes=[]`であり、過去draftのversionまたはaliasを存在すると推測しない。

二Capability recordは§10の全Fieldを次のexact値でmaterializeする。Local payload内の`operations[]`と`validators[]`はそれぞれ`ContractSetLocalRefV1`と`ValidatorLocalRefV1`であり、set root確定後だけ外部refへ投影する。

```text
capability.runtime.scheduling@1
  maturity: C0
  supported_targets: [
    target.android.mobile
    target.apple.mobile
    target.headless.host
    target.windows.desktop
    target.windows.editor
  ]
  required_capabilities: []
  conflicts: []
  authoring_types: []
  operations: []
  validators: []
  quality_profiles: []
  budgets: []
  failure_modes: [
    {diagnostic_code: MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED,
     fallback_id: fallback.capability.unavailable}
  ]
  examples: []
  ai_guidance: []

capability.authoring.command_gateway@1
  maturity: C0
  supported_targets: [
    target.headless.host
    target.windows.desktop
    target.windows.editor
  ]
  required_capabilities: []
  conflicts: []
  authoring_types: []
  operations: [
    {operation.project.runtime_entry.create,1}
    {operation.project.runtime_entry.update,1}
    {operation.project.runtime_entry_activation_policy.create,1}
    {operation.project.runtime_entry_activation_policy.update,1}
    {operation.project.runtime_target_selector.create,1}
    {operation.project.runtime_target_selector.update,1}
    {operation.shooter.target_provider_binding.create,1}
    {operation.shooter.target_provider_binding.select,1}
    {operation.shooter.target_provider_binding.update,1}
    {operation.world.allocate_generated_stable_ids,1}
  ]
  validators: [
    {validator.operation.approval,1}
    {validator.operation.authorization,1}
    {validator.operation.pure_predicate,1}
    {validator.operation.request_envelope,1}
    {validator.operation.revision_and_lock,1}
    {validator.operation.timeout_and_rate_limit,1}
  ]
  quality_profiles: []
  budgets: []
  failure_modes: [
    {diagnostic_code: MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED,
     fallback_id: fallback.capability.unavailable}
  ]
  examples: []
  ai_guidance: []

```

`capability.runtime.scheduling@1.required_capabilities=[]`はMCD contract semanticsであり、Product Work Packageの実装順依存を削除しない。二recordの`status=active`は「参照可能なMCD契約」を意味するだけで、Product operational Activation、実装、Qualification、Target shippingを主張しない。

二Capabilityの`failure_modes[]`はFoundation rootからProduct rootへの循環を避けるため、二種類のforeign keyを意図的に保持する。`diagnostic_code=MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`はDiagnostic Registry recordではなく、Contract set dispatch前に使うclosed Policy rejection tokenである。`fallback_id=fallback.capability.unavailable`はMCD refではなく、[Product Planの`FallbackRegistryV1`](product-execution-registry-proposal.md#11-product-execution-registries)へのProduct foreign keyである。Control PlaneのBaseline／Rebaseline verifierはFoundation closureと同時に束縛されたexact `ActiveProductDefinitionBundleV1`からこのfallback rowを一件だけ解決し、`preserves_semantics=false`かつ、その三Field`ProductDiagnosticRefV1`がexact `diagnostic.product.capability-unavailable` version 1のProduct Diagnostic record／hashへ解決することを検査する。missing／multiple／別diagnostic／ID-only ref／hash差／stale Active Product rootではCapabilityをactivateしない。Foundation Contract setへProduct-owned Diagnostic／Fallback refを戻さず、Policy tokenまたは`ProductDiagnosticRefV1`を`DiagnosticCodeRefV1`として偽装しない。

current一Profile recordのpayload schemaは[AI Security／Approvalの`TrustedServiceIsolationProfileV1`](../01-governance/ai-security-approval.md#73-trusted-service-isolation)とし、exact値を次に固定する。

| Field | `profile.isolation.authoring_command_gateway@1` |
|---|---|
| `profile_kind` | `trusted_service_isolation` |
| `base_profile_local_ref` | `null` |
| `execution_mode` | `in_process_trusted_service` |
| `network_access` | `denied` |
| `engine_baseline_access` | `read_only_exact_lock` |
| `project_store_access` | `transactional_authoring_changeset` |
| `staging_access` | `private_transaction_candidate` |
| `project_source_access` | `none` |
| `host_filesystem_paths` | `[]` |
| `input_artifact_access` | `exact_content_refs_only` |
| `output_policy` | `domain_publication_pipeline` |
| `ephemeral_scratch` | `brokered_bounded_private` |
| `process_spawn` | `denied` |
| `environment_access` | `none` |
| `credential_access` | `brokered_non_exportable_operation_purpose_key` |
| `secret_export` | `denied` |

実在consumerのないinitial Contract setへoffline migrator用Service／Capability／Isolation Profile、空allowlistまたはAuthorityを登録しない。最初の公開後にCompatibility Ownerのconsumer inventoryが移行を要求する場合だけ、新しいversioned ChangeSetでexact Operation、Service、Profile、Policy、Fixture、Receiptを完全closureとして設計する。

このFoundation correction candidateのtarget MCD差分はexact `Type +1、Capability +2、Profile +1、合計 +4`である。四recordは未materializeでcurrent MCD差分はexact 0件である。将来のatomic materializationでは一件missing／extra／duplicate、Envelope Field省略、配列順差、Capability Operationと§8.2 Service allowlistのset差、Profile Field差をContract set compile failureにする。

predicate IO/resultのtarget MCD Type candidateは`type.operation.precondition_evaluation_input` version 1、`type.operation.postcondition_evaluation_input` version 1、`type.operation.predicate_result` version 1である。Prepared auxiliary payloadのcandidate Typeは`type.operation.prepared_preview_receipt_payload` version 1と`type.operation.prepared_validation_receipt_payload` version 1、Mutation authorization candidate Typeは`type.operation.mutation_authorization_binding` version 1である。六target Type recordは上記Field、presence、boundを完全投影し、bare snapshot ID、評価中のRegistry query、時計、network、mutable pointer、published Commit Receiptを許可しない。Prepared二Typeの共通Envelopeは将来materialization時にそれぞれ`mcd_version=1; kind=type; id=<上記ID>; version=1; status=active; owners=[owner.core.project_state]; requirement_refs=[]; rationale_refs=[mirakan.arch.executable-contracts#8-operation定義]; since_contract_set=1; supersedes=[]`を持ち、title／description／tagsはID固有、payload schemaは対応するnamed `PreparedPreviewReceiptPayloadV1`／`PreparedValidationReceiptPayloadV1`の全Field、presence、boundをexact投影する。Mutation TypeはAI Security／Approval正本のApproval／Grant／Consumption Head／Predelegation Use／typed Ref／`MutationAuthorizationBindingV1`全wire shapeをclosed `$defs`へ一度だけ投影し、unknown sibling shapeを許可しない。これらは同じMutation Typeのnested definitionでありMCD Type件数へ重複加算しない。三Type LocalRef／local record hash／member hashを将来同じContract Setへ含め、root確定後だけ同root付きMCD Type refへmaterializeする。target rate-limit policy candidateは`policy.authoring.runtime_entry.rate_limit` version 1 exact一件で、`scope=principal_project, window_ns=60000000000, max_requests=120, burst=20`、`exceeded_error_ref={diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash}`を持つ。current Type／Policy／Service集合はすべて`[]`である。legacy migration用rate-limit policyはtarget Policy集合にも含めない。将来materializationではPolicy共通Envelope、payload、Type六record、Service一recordのmissing／wrong-kind／stale version／stale Contract set／content hash mismatchをOperation Registry compile errorにする。

`PreparedCandidateV1.candidate_content_hash`はASCII `MIRAKAN_PREPARED_CANDIDATE_V1`、candidate hash自身を除く全Fieldのcanonical bytesを`uint32_be` length framingして計算する。`PreparedCandidateRefV1`は完成candidateのID、schema ref、content hashの三者をexactにbindし、Record自身へhash付きRefを埋め戻さない。Staging root、before／proposed-after state、prepared Artifact集合の一Fieldでも変われば別Refになり、candidate IDだけ、latest candidate、別Staging rootの同IDへfallbackしない。`PublishedReceiptSigningProfileV1.profile_content_hash`はASCII `MIRAKAN_PUBLISHED_RECEIPT_SIGNING_PROFILE_V1`、`PublishedReceiptMaterializationPolicyV1.policy_content_hash`はASCII `MIRAKAN_PUBLISHED_RECEIPT_MATERIALIZATION_POLICY_V1`と各自己Fieldを除くclosed recordのRFC 8785 JCS bytesを`uint32_be` length framingして計算する。完成Materialization Policy全体の`policy_object_sha256 = SHA-256(RFC 8785 JCS(completed object including policy_content_hash))`を別に計算し、`PublishedReceiptMaterializationPolicyRefV1.policy_artifact_ref.sha256`とbyte equalityにする。semantic `policy_content_hash`と完成object hashの等値を要求せず、RefをPolicy自身へ埋め戻さない。

三`PublishedReceiptPublicationControlPolicyV1`の`policy_content_hash`は、`verification_retention`がASCII `MIRAKAN_PUBLISHED_RECEIPT_VERIFICATION_RETENTION_POLICY_V1`、`deterministic_recovery`が`MIRAKAN_PUBLISHED_RECEIPT_RECOVERY_POLICY_V1`、`receipt_store_namespace`が`MIRAKAN_PUBLISHED_RECEIPT_STORE_NAMESPACE_POLICY_V1`と、同Fieldだけを除くclosed objectのRFC 8785 JCS bytesをlength-frameして計算する。各完成objectの`policy_object_sha256 = SHA-256(RFC 8785 JCS(completed object including policy_content_hash))`はGovernance Policy Config rowの`policy_ref.sha256`／`policy_sha256`とbyte equalityにし、typed Refの四Fieldは解決objectのID／version／kind／semantic hashとexact equalityにする。

Receipt publicationがGovernance Policy Config Registryから解決するrequired subsetは、AI Security Ownerの`operation_receipt_signer_policy.active` exact一rowと、次のExecutable Contracts Owner exact三rowのdisjoint union、すなわちexact四rowである。これはRegistry全体のrow数ではなく、他のControl Plane Policy rowを削除または上書きしない。

| `policy_id` | `policy_kind` | `policy_schema_id` | `policy_ref.artifact_kind` |
|---|---|---|---|
| `published_receipt_verification_retention_policy.active` | `verification_retention` | `urn:mirakan:schema:executable-contracts:published-receipt-publication-control-policy:v1` | `governance.published_receipt_verification_retention_policy` |
| `published_receipt_recovery_policy.active` | `deterministic_recovery` | `urn:mirakan:schema:executable-contracts:published-receipt-publication-control-policy:v1` | `governance.published_receipt_recovery_policy` |
| `published_receipt_store_namespace_policy.active` | `receipt_store_namespace` | `urn:mirakan:schema:executable-contracts:published-receipt-publication-control-policy:v1` | `governance.published_receipt_store_namespace_policy` |

三rowの`policy_ref.schema_version=1`、current objectの`policy_version=1`、`policy_ref.sha256=policy_sha256=policy_object_sha256`とする。共通schema annotationは`x-mirakan-governance-policy-id-kind=fixed_logical`、許可ID集合は表のexact三IDである。意味Fieldを変える場合だけobject versionを`N+1`、row bytesが変わる場合だけRegistry revisionを`N+1`へ進め、旧object、旧Registry、旧Trust closure／Headをretentionする。Materialization Policy自身は発行時Trust／Registry／Keyをpinするper-receipt content objectであり、それ自身がpinするGovernance RegistryにもMCD `kind=policy`にも登録しない。登録すると`Materialization Policy → Governance Registry／Contract Set → Materialization Policy`の循環になるためcompile errorとする。

三closed root schema IDはexact `urn:mirakan:schema:executable-contracts:published-receipt-publication-control-policy:v1`、`urn:mirakan:schema:executable-contracts:published-receipt-materialization-policy:v1`、`urn:mirakan:schema:executable-contracts:operation-authorization-audit-binding:v1`である。Ownerは`mirakan.arch.executable-contracts`、Draft 2020-12、unknown Field禁止、signature slot exact `[]`とする。Control Plane bootstrap候補のpre-ceremony full Local Schema Catalog／Materialization Planは三IDをExecutable Contracts Ownerのcomplement memberとして列挙し、そのcomplementだけが完成schema bytesとsemantic verifierをbyte-exact materializeする。Local Schema Catalog rootはexact `+3`、Architecture Governance Ownerの追加contract exact 30、Content ID Registry exact 32、Authority Binding Source Catalog slot集合は不変である。Catalog rowまたはcomplement setの一件missing／extra、unsigned schemaへのslot追加を拒否する。

`OperationAuthorizationAuditBindingV1.binding_content_hash`はASCII `MIRAKAN_OPERATION_AUTHORIZATION_AUDIT_BINDING_V1`と同Fieldだけを除くclosed recordのRFC 8785 JCS bytesをlength-frameして計算する。完成binding全体の`binding_object_sha256 = SHA-256(RFC 8785 JCS(completed binding including binding_content_hash))`は`OperationAuthorizationAuditBindingRefV1.binding_artifact_ref.sha256`とbyte equalityにし、RefのOperation／request hash／semantic hashを解決bindingへexact一致させる。`request_input_content_ref`は選択input Typeで検証済みの完成final request bytesだけを指し、そのOperation ref、input Type ref、risk、intent、request hash、Algorithm binding、inline `MutationAuthorizationBindingV1`のAuthorization ref／hashとApproval SetまたはPredelegation Use ref／hashをAudit Bindingへbyte equalityで複写する。request input自身へAudit BindingまたはContextを戻さず、固定点を作らない。公開Bindingはrequest body、requester identity、自由文、PIIをinlineせずref／hash commitmentだけを公開し、bodyはimmutable access-controlled CASに保持する。R2はApproval／Predelegationのexact一方、R3～R5はApprovalだけとし、missing／両branch、別Task／intent／scope／request、完成前input、hash-onlyで解決不能なartifactを拒否する。

`PublishedReceiptMaterializationContextV1`は二段階で完成する。まず`context_id`と`context_content_hash`を除いたclosed context recordをRFC 8785 JCSへencodeしたbytesを`B`とし、`D = SHA-256(ASCII "MIRAKAN_PUBLISHED_RECEIPT_MATERIALIZATION_CONTEXT_ID_V1" || uint32_be(len(B)) || B)`、`context_id = "urn:mirakan:published-receipt-materialization-context:sha256:" || lowercase_hex(D)`とする。次にその`context_id`を挿入し`context_content_hash`だけを除いた完成前recordをRFC 8785 JCSへencodeしたbytesを`C`とし、`context_content_hash = SHA-256(ASCII "MIRAKAN_PUBLISHED_RECEIPT_MATERIALIZATION_CONTEXT_V1" || uint32_be(len(C)) || C)`とする。`B`と`C`のFieldは§8のnamed schemaに存在するものだけで、unknown Field、別prefix／domain separator、ID-only lookup、`context_id`または`context_content_hash`の自己混入を拒否する。これにより`context_id`はOperation ref、operation intent hash、request hash、request hash Algorithm binding、idempotency key、before／staged-after state、Authorization Audit Binding、materialization policy、発行時`OperationReceiptSignerPolicyRefV1`、Trust Registry Closure Head、同Headが指すClosureとGovernance Policy Config Registry snapshot、signing profile、Signer／Role／Key、Public key registry snapshot、issued-at、revocation snapshotから機械一意に導出され、caller supplied UUIDや時計の再読取を使わない。

Context作成時は完成`TrustRegistryClosureHeadV1`をref／hashで解決し、そのpayloadが指すClosure pairとContextのTrust closure pairをbyte equalityにする。Closure内のGovernance Policy Config Registry memberからreceipt publication required subset exact四rowをID／schema／ref／hashで解決し、各completed object hashとsemantic hashを再検証する。Signer Policy rowはContextのOperation refをexact一件持ち、execution authority、Signer subject、Role、signed-record purposeが実行ServiceおよびMaterialization Policyと一致しなければならない。三control-policy rowはMaterialization Policyのtyped RefとID／version／kind／semantic hashが一致し、Materialization PolicyのOperation ref／execution authority／purposeはOperation Registry／実行Service／Signer Policy、`receipt_canonical_schema_ref`は同Operation recordの`receipt_type`とbyte equalityにする。ContextのGovernance Registry snapshotはTrust closureの同Registry member、Public key registry snapshotは同Trust closureのPublic Key Registry member、revocation snapshotは同Trust closureが束縛する発行時revocation currentへexact解決する。Contextと全Prepared publication bindingの`request_hash_algorithm_binding`はOperation input内の同bindingとbyte equalityにし、Closure／Registry／Algorithm recordを再解決する。ContextのAuthorization Audit Bindingを解決してfinal request input／`MutationAuthorizationBindingV1`とのequalityを再検証し、ContextのMaterialization Policy、Signer Policy、Trust Head、Governance Registry、Trust closure、profile、Signer／Role／Key／Public key snapshotは解決したMaterialization Policyの同Fieldとbyte equalityでなければならない。

private Marker作成直前と署名Broker呼出し直前にcurrent receipt-publication Policy exact四row、Signer Policy、Trust Head／closure、Key／assignment／revocationをfresh read-backする。Context snapshotから一Fieldでも変わった場合、未署名candidateをterminal abortし、fresh Authorization／Contextから新candidateを作る。既にBroker journalへ完成wrapper bytesがcommit済みのcrash retryだけは同じwrapperを再署名せず復旧できるが、Public Marker CAS前にcurrent Trust Head、Signer Policy、revocation／publication policyを再評価し、失効後の新規公開を許可しない。Verification Retention PolicyはAuthorization audit、Receipt publication、schema definition、Trust verificationの四closureをretained rootから到達する限り保持し、prepublication private stateはPublic Markerまたはdurable terminal abortまで保持する。private key retentionを要求せず、Key消失時に別Keyで再署名しない。Recovery Policyはprivate Marker read-backから同じsemantic hash／完成object SHAのsecret-free `PublicCommitClosureV1` candidateを再materializeまたはreadbackし、同一materialization key／Context／subject／Keyのbyte-exact wrapperを完成する遷移と、保存済みClosure／wrapperから同じClosure body＋Public Marker＋after stateをexpected predecessorへatomic CASする遷移だけを許し、それ以外はfail closedとする。Store Namespace Policyは完成wrapper bytesのSHA-256をkeyとするput-if-absent byte equality、read-back後のClosure＋Public Marker＋after state atomic CAS、Retention Policyだけによる削除制御を強制し、backend locator／credentialをrecordへ保存しない。

Commandの唯一のpublish順は`immutable final request → Authorization Audit Binding → Preview → Validation → per-receipt Materialization Policy／pre-marker Context → receipt-free PreparedCandidate → 必要なprepromotion Evidence／Source Promotion → late source-promotion authorization binding → PreparedCommitEnvelope → staged postcondition → private durable commit marker read-back → secret-free PublicCommitClosure candidate → canonical signed Domain Receipt store → PublicCommitClosure＋public publication marker＋after stateのpublic CAS`である。Source registration primitiveが0件なら`project_source_promotion_authorization_binding_refs=[]`、一件以上ならProject Stateの同じChangeSet／Prepared Candidate／Candidate rootと全source primitiveを束縛するexact一Refだけを許し、Plan、Envelope、三`PreparedReceiptPublicationBindingV1`でbyte-exact set equalityにする。Binding、Promotion Receipt、Build／Test／Review／ApprovalをPrepared Candidateまたはreceipt-free staged after stateのhash preimageへ戻さず、Binding追加後のCandidate／ChangeSet変更を拒否する。

`AtomicCommitPlanPayloadV1.prepared_payload_count`は配列長と一致し、membersはtype ref、content ref、hash順へstrict sortしてduplicate／same-ref different-hashを拒否する。exact三件は`type.operation.prepared_preview_receipt_payload@1`へ解決する`PreparedPreviewReceiptPayloadV1`、`type.operation.prepared_validation_receipt_payload@1`へ解決する`PreparedValidationReceiptPayloadV1`、当該Operationのnamed Domain prepared receipt payloadであり、Plan／Envelope／private Markerでtype ref／content ref／content hashのset equalityを必須にする。三payloadはすべてexact `publication_binding: PreparedReceiptPublicationBindingV1`を持つ。Preview／Validationの`prepared_payload_hash`はそれぞれASCII `MIRAKAN_PREPARED_PREVIEW_RECEIPT_PAYLOAD_V1`／`MIRAKAN_PREPARED_VALIDATION_RECEIPT_PAYLOAD_V1`と自己Fieldを除くcanonical bytesから計算する。Validation rejected、Diagnostic非空、missing preview resultのpayloadはPlanへ入れず、public objectを0件にする。Domain固有の`before_project_ref`／`after_project_ref`等はbindingの`before_state_ref/hash`／`staged_after_state_ref/hash`へ一意に投影し、duplicate Fieldもbyte equalityにする。Envelopeの三payload refsはPlan payload集合とset equality、EnvelopeのCandidate／late binding／policy／ContextはPlanの同Fieldとexact equalityでなければならない。

late source-promotion authorization bindingはimmutable final request／Operation inputのFieldではなく、Prepared Candidate確定後に発行される唯一のlate authorizationである。その集合はPlan、Envelope、三Prepared publication binding、private MarkerからEnvelopeを解決した値の間だけでbyte equalityにし、Operation inputへ逆流させない。Operation ref、intent hash、request hash、request hash Algorithm binding、idempotency key、before state、staged-after state、Authorization Audit Binding、ContextだけをOperation inputを含む全closureで比較する。Candidateのbefore／proposed-after stateも同じbefore／staged-after stateへexact解決する。

`atomic_commit_plan_hash = SHA-256(ASCII "MIRAKAN_ATOMIC_COMMIT_PLAN_V1" || uint32_be(len(MCD canonical plan payload bytes)) || plan payload bytes)`とし、Prepared Envelope自身、staged postcondition Receipt、private Marker、signed Receipt、public Markerをplan payloadへ含めない。`envelope_hash`はASCII `MIRAKAN_PREPARED_COMMIT_ENVELOPE_V1`、`receipt_payload_hash`はASCII `MIRAKAN_STAGED_POSTCONDITION_RECEIPT_V1`、private `marker_hash`はASCII `MIRAKAN_PRIVATE_DURABLE_COMMIT_MARKER_V1`、public `marker_hash`はASCII `MIRAKAN_PUBLIC_PUBLICATION_MARKER_V1`と各自己Fieldを除くcanonical bytesをlength-frameして計算する。postconditionは未発行Staging、Prepared Receipt payload、既に固定されたContextとlate bindingだけを読み、clock／revocation registryを再queryせず、private／public Marker、公開後state、最終Receiptを入力にしない。type refのwrong version／root／kind、preview↔validation type swap、同contentを別typeで再利用するfixtureを各一原因で拒否する。

Gatewayはpostcondition success後、Prepared Envelope、Preview／Validation／Domain semantic payload、Staged Postcondition Receipt、staged after state、`PublishedReceiptMaterializationContextV1`、`PrivateDurableCommitMarkerV1`を外部readerから到達不能なprivate durable transactionへcommitする。この時点でProject／Registry／Runtime current head、Document index、provider-visible Resultは一切変えない。private Marker preimageはPrepared Envelope ref／hash、staged postcondition ref／hash、prepared payload ref／hash、Context ref／hashを束縛するが、Marker自身、`PublicCommitClosureV1`、signed Receipt、public Markerを含めない。

private Markerを完成bytesとしてreadbackし、Prepared Envelope／Candidate／late binding／before／staged-after／Domain prepared payloadとのequalityを再検証した後にだけ、secret-free `PublicCommitClosureV1` candidateを生成してimmutable public-candidate storeへput-if-absentする。Project ChangeSet Commitでは`domain_commitment.kind=project_change_set_commit`とし、Project Stateが固定したexact `project_change_set_ref`／`candidate_root_sha256`を複写する。Source registration primitiveが0件でもこの二Fieldを持ち、late binding集合だけを`[]`にする。その他のmutationは登録済みOwnerとreceipt-free committed Artifact集合を持つ`owner_typed_state_commit`だけを使う。Closureのsemantic hash、完成object SHA、Refを§8の規則で別々に再計算し、Closure candidateだけをpublic current authorityにしない。

その同じClosure Ref／完成object hashを入れて`PublishedDomainReceiptPayloadV1`を完成し、AI Verification／Provenanceが所有する`MirakanSignedRecordV1`をexact `$ref`する`PublishedDomainReceiptV1` wrapperをreceipt storeへput-if-absentする。inline signer／key／algorithm／signature FieldをDomain receiptに定義しない。Published payloadのprepared payload ref／hashはprivate Markerのexact三件集合中のDomain memberへ解決し、Operation ref、intent hash、request hash、idempotency key、before state、after state、Context、Closure ref／完成object hashは解決したDomain publication binding、private Marker、Closure candidateの対応Fieldとbyte equalityにする。Published `after_state_ref/hash`はprivate Markerの`staged_after_state_ref/hash`を名前だけ変えた同一値である。`issued_at`／`revocation_snapshot_ref/hash`は解決したContextの同Fieldとbyte equalityでなければならない。`signed_record.purpose=operation_domain_receipt`、`subject_sha256=SHA-256(JCS(payload))`、signer subject／Role／Keyは完成payloadとContext／Materialization Policyへbyte equalityにする。Published payloadは自己`payload_hash` Fieldを持たず、signed subject hashだけを完成JCS payloadから計算する。

署名済みwrapperの保存／readback後だけ、Gatewayはreadback済みとbyte-identicalな`PublicCommitClosureV1` body、`PublicPublicationMarkerV1`、after stateを一つのpublic CAS transactionでpublishする。Public MarkerのOperation ref、intent hash、request hash、idempotency key、before state、public-after state、Context、private marker hash、Closure Ref／完成object hashはPublished payload、private Marker、Closureの対応Fieldとbyte equalityにし、`signed_domain_receipt_ref/hash`はreadbackしたexact wrapperだけを指す。`public_after_state_ref/hash`はPublished payloadの`after_state_ref/hash`およびprivate Markerの`staged_after_state_ref/hash`と同一である。Closureのsemantic hashはRefの`closure_content_hash`、完成object SHAはRef内`closure_ref.sha256`とMarker／Receiptの隣接`public_commit_closure_hash`へ一致させ、両hashを相互代用しない。

このpublic CASはsecret-free `PublishedReceiptMaterializationContextV1`と、そこから到達するMaterialization Policy、三Publication Control Policy、Signer Policy completed object、Governance Policy Config Registry snapshot、Trust Registry Closure Head／closure／Public Key／revocation snapshot、Signing Profile、Authorization Audit Binding、Public Commit Closureをimmutable content-addressed verification graphへ同時に到達可能化する。Public MarkerのContext ref／hashおよびClosure refからこのpublic部分をoffline解決できない場合はCASを成功させない。Audit Bindingのrequest／Task Authorization／Approval SetまたはPredelegation Useは公開Binding内へbodyをinlineせず、ref／hash commitmentだけを公開する。Public integrity verifierはcommitmentまで、認可済みAudit verifierはaccess-controlled immutable CASからbodyを解決して`MutationAuthorizationBindingV1`とのbyte equalityまで検証する。publication前には両resolverで必要Artifactのdurable存在／hash一致を確認し、匿名公開不能をmissing artifactとして誤判定しない。private Marker、Staging payload、request／AuthorizationのPII body、private key、secret locatorは公開graphへ昇格しない。

このafter stateはPublic Commit Closure、signed Receipt、private／public Marker、publication projectionをFieldにもcontent-hash preimageにも含まないReceipt-free staged stateでなければならず、それらを含むDomain projectionをPublic Markerのafter stateに代用しない。Project／Registry／Runtime current headはPublic Markerが指すこのReceipt-free after stateだけをcurrentとして解決する。Domain Resultまたはroot外publication projectionだけがPublic Marker、Closure、signed wrapperを結合し、unsigned prepared payload、private Marker、Closure candidate store、receipt-store単独存在をstate authorityにしない。これにより`final request → Authorization Audit Binding → Materialization Policy／Context → candidate／Receipt-free staged after state／staged postcondition → private marker read-back → secret-free PublicCommitClosure candidate → signed wrapper → PublicCommitClosure＋public marker＋same after state／verification graphのatomic CAS → root外projection`の一方向DAGとなる。postcondition failureまたはprivate commit失敗はpublic state／Closure／Receipt／Public Markerを0件にし、prepared payloadを権限証拠として外部公開しない。

Receipt publication fixtureは、Materialization Policyの自己Ref埋戻し、同PolicyのMCD／Governance Registry登録、required Policy四rowのmissing／extra／duplicate、三control policyのID／kind／schema／version／Ref swap、semantic hashと完成object hashの混同、retained closure kind欠落、private-key retentionまたは別Key再署名許可、fresh clock／Key／Trustへのrecovery差替え、namespace overwrite／readback省略／locator混入を各一原因で拒否する。Authorization fixtureはAudit Binding欠落、request inputのOperation／Type／intent／request／Algorithm差、inline `MutationAuthorizationBindingV1`とのAuthorization／Evidence差、Approval／Predelegation両branchまたは欠落、PII bodyの公開inline、Context後のBinding差替え、access-controlled bodyのdurable欠落／hash mismatchを各一原因で拒否する。public graph fixtureは三control policyまたはAudit Binding objectの到達不能、private Marker／private key／secret locator／PII bodyの混入、Local Schema Catalog三IDまたはExecutable Contracts complement materializationの一件欠落、unsigned schemaへのAuthority slot追加、Trust closure memberを15から増減する実装を拒否する。

private Markerのdurable commit後、signed wrapper保存前に停止した場合、recoveryはprivate Marker、Prepared Envelope、Prepared payload、materialization Context、staged postcondition、staged after stateをread-backし、上記identity／state／Context equalityを再検証して同じsemantic hash／完成object SHAの`PublicCommitClosureV1`を再materializeまたはbyte-identical candidateとしてreadbackしてから、同じmaterialization key、Contextに固定されたissued-at／revocation／Key、deterministic profileから同じwrapper bytesを作る。Brokerは`{signer subject, purpose, receipt_materialization_key, subject_sha256, key_id}`をkeyとするdurable idempotency journalをsignature返却前にcommitし、同じkey／contextの再開にはcached byte-exact RFC 6979 signature／wrapperだけを返す。別request／Project／Context／subject／Keyによる同じhandleまたはmaterialization keyの再利用はintegrity faultであり、再署名へfallbackしない。clock、current Key選択、発行時revocation snapshotを再取得しない。wrapper保存後かつPublic Marker前に停止した場合、wrapper byte／signature／current revocation、Closure body、private preimageを再検証し、同じClosure＋Marker＋after stateを同じexpected public predecessorへのCASでroll-forwardする。Public Marker後はrollbackせず、同じidempotency key＋request hashのretryへ同じResult／Closure／wrapper／Public Markerを返す。Markerと`PublishedReceiptMaterializationKeyPayloadV1`のpayload countは各配列長と一致させ、両membersはpayload type refのID／version／Contract set root、payload content refのcanonical bytes、content hash順へstrict sortする。両者のOperation ref、intent hash、request hash、idempotency key、before／staged-after state、ContextもPrepared Envelope／Domain publication bindingとbyte equalityにする。duplicate、same-ref different-hash、両集合のmissing／extra、identity／state／Context substitutionを拒否する。`key_payload_bytes`はこのclosed payloadをMCD canonical encodeしたbytesで、`receipt_materialization_key = SHA-256(ASCII "MIRAKAN_PUBLISHED_RECEIPT_MATERIALIZATION_V1" || uint32_be(len(key_payload_bytes)) || key_payload_bytes)`とし、key自身をpayloadへ含めない。別Closure、別wrapper、alternate signature、二重Public Marker、既存bytes overwrite、署名なしpublish、public後rollbackをintegrity faultとして隔離する。private MarkerなしのPrepared artifactは非公開のまま破棄する。

Snapshot preimageでは`OperationValidatorClosureLocalRecordV1`を`member_kind=operation_validator_closure`としてroot化し、Operation、Validator、Diagnosticへの全edgeをLocalRefにする。`closure_local_record_hash`はASCII `MIRAKAN_OPERATION_VALIDATOR_CLOSURE_LOCAL_RECORD_V1`と同Fieldだけを除くlocal recordから計算する。Contract Set `member_hash`は別domain `MIRAKAN_CONTRACT_SET_OPERATION_VALIDATOR_CLOSURE_MEMBER_V1`と、このlocal hashを含む完成local recordから計算する別値であり、両hashのbyte equalityを要求しない。set root確定後にだけLocalRefを外部Operation ref、`ValidatorRecordRefV1`、`DiagnosticCodeRefV1`へ投影し、`OperationValidatorClosureV1`を作る。`closure_content_hash`はASCII `MIRAKAN_OPERATION_VALIDATOR_CLOSURE_V1`、外部closureの`closure_content_hash`自身を除くcanonical bytesから計算し、完成後にだけ`OperationValidatorClosureRefV1`をmaterializeする。この外部hashとRefはSnapshot preimageへ戻さない。外部closureのvalidator refはValidator Registryへexact解決し、各Validator recordが宣言する`error_refs[]`のunionをID／code／version／hash順へcanonicalizeした集合が`reachable_error_refs[]`と一致しなければならない。さらにその集合とOperation `errors[]`をset equalityで比較し、missing errorと到達不能extra errorの双方でOperation Registry全体を拒否する。`reachable_error_set_hash`はLocal形ではASCII `MIRAKAN_OPERATION_REACHABLE_ERROR_LOCAL_SET_V1`とDiagnostic LocalRef集合、外部形ではASCII `MIRAKAN_OPERATION_REACHABLE_ERROR_SET_V1`と各四Fieldrefから別々に計算し、どちらもerror countとcanonical bytesを`uint32_be` length framingして自己Fieldを入力にしない。compilerはLocalと外部のlogical Diagnostic集合が一対一であることも検査する。

Snapshot内部の正本は`ValidatorLocalRecordV1 {validator_local_ref, implementation_artifact_ref/hash, input_type_local_ref, error_local_refs[1..64], validator_local_content_hash}`である。Type／DiagnosticはLocalRefを使い、外部MCD refやset rootをlocal record hashへ含めない。`validator_local_content_hash = SHA-256(ASCII "MIRAKAN_VALIDATOR_LOCAL_RECORD_V1" || uint32_be(len(self-excluding canonical local bytes)) || local bytes)`、Validator Registry hashはASCII `MIRAKAN_VALIDATOR_REGISTRY_V1`、Registry ID／version、record count、validator ID／version順local record bytesから計算する。Contract SetのValidator `member_hash`は別domain `MIRAKAN_CONTRACT_SET_VALIDATOR_MEMBER_V1`と、このlocal hashを含む完成local recordから計算する第三の値であり、local hashとのbyte equalityを要求しない。duplicate、same-ID different-hash、非canonical sort、実装Artifact missing／hash mismatch、input type kind／version mismatch、Diagnostic LocalRef未解決を拒否する。

set root確定後だけ、compilerはLocalRefを同じrootのexact `McdContractRefV1(kind=type)`／`DiagnosticCodeRefV1`へ一対一投影し、`materialized_from_contract_set_hash=contract_set_hash`を持つ完全な`ValidatorRecordV1`を作る。`validator_content_hash = SHA-256(ASCII "MIRAKAN_VALIDATOR_RECORD_V1" || uint32_be(len(external record canonical bytes excluding validator_content_hash)) || external bytes)`であり、完成後だけ`ValidatorRecordRefV1 {validator_id,validator_version,validator_content_hash}`をmaterializeする。外部recordのinput Type refと全Diagnostic refのContract set rootは`materialized_from_contract_set_hash`と一致し、logical ID／version集合はlocal recordとexact equalityでなければならない。外部record／hash／refはlocal record、Validator Registry、member hash、set rootへ戻さない。別rootのValidator ref代用、local/external集合差、stale root、external hash mismatchをContract set compile errorにする。

target初期Diagnostic Registry candidateは次のexact 21 recordである。このうちtarget六Operationの`errors[]`／Validator closureからreachableな集合はexact 17 recordで、残るexact四non-operation recordはruntime target selection用`diagnostic.project.runtime_entry.explicit_target_mismatch`と、target generic Runtime Scope Catalogのmaterialization／activation validator用`diagnostic.runtime_scope.catalog_invalid`、`.owner_unavailable`、`.version_hash_mismatch`だけである。四recordをOperation reachable集合へ暗黙追加せず、Operation reachable集合とRegistry inventoryを混同しない。全rowは`diagnostic_version=1`、`requirement_local_refs=[]`、`message_key="<diagnostic_id>.message"`を持つ完全な`DiagnosticLocalRecordV1`であり、root確定後の外部projectionは`requirement_refs=[]`を持つ。exact Owner local inventoryは既存Owner identityだけを使う`{owner.core.project_state,1,owner_content_hash}`、`{owner.core.security_approval,1,owner_content_hash}`、`{owner.core.gameplay_programming_model,1,owner_content_hash}`の三件で、各hashは`MIRAKAN_DIAGNOSTIC_OWNER_LOCAL_IDENTITY_V1`規則から計算する。`diagnostic.authorization.denied`と`diagnostic.approval.required`は`owner.core.security_approval`、他の共通六件と`diagnostic.project.runtime_entry.*`は`owner.core.project_state`、三`diagnostic.runtime_scope.*`は`owner.core.gameplay_programming_model`のexact `DiagnosticOwnerLocalRefV1`を`owner_local_ref`へ持ち、外部recordも同じ三Field Owner refへ投影する。各rowはOwnerを含む表の全Fieldからself-excluding `diagnostic_local_content_hash`を計算し、root後は別のself-excluding `diagnostic_content_hash`を計算する。表外code、同じcodeの別ID、説明語から合成したcode、owner prefixから補完したrefをOperation errorとして返さない。

| `diagnostic_id` | `code` | severity／category／retryability |
|---|---|---|
| `diagnostic.conflict.revision_mismatch` | `MIRAKAN-CONFLICT-REVISION_MISMATCH` | blocking／conflict／after_change |
| `diagnostic.authorization.denied` | `MIRAKAN-AUTHORIZATION-DENIED` | blocking／permission／never |
| `diagnostic.approval.required` | `MIRAKAN-APPROVAL-REQUIRED` | blocking／permission／after_input |
| `diagnostic.authoring.lock_conflict` | `MIRAKAN-AUTHORING-LOCK_CONFLICT` | blocking／conflict／after_change |
| `diagnostic.mcd.operation_predicate_invalid` | `MIRAKAN-MCD-OPERATION-PREDICATE_INVALID` | blocking／schema／after_change |
| `diagnostic.operation.timeout` | `MIRAKAN-OPERATION-TIMEOUT` | error／infrastructure／transient |
| `diagnostic.operation.rate_limit_exceeded` | `MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED` | error／permission／transient |
| `diagnostic.operation.idempotency_key_reuse` | `MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE` | blocking／conflict／after_input |
| `diagnostic.project.runtime_entry.invalid` | `MIRAKAN-PROJECT-RUNTIME_ENTRY_INVALID` | blocking／semantic／after_input |
| `diagnostic.project.runtime_entry.target_unresolved` | `MIRAKAN-PROJECT-RUNTIME_ENTRY_TARGET_UNRESOLVED` | blocking／semantic／after_change |
| `diagnostic.project.runtime_entry.default_ambiguous` | `MIRAKAN-PROJECT-RUNTIME_ENTRY_DEFAULT_AMBIGUOUS` | blocking／semantic／after_change |
| `diagnostic.project.runtime_entry.branch_field_conflict` | `MIRAKAN-PROJECT-RUNTIME_ENTRY_BRANCH_FIELD_CONFLICT` | blocking／schema／after_input |
| `diagnostic.project.runtime_entry.dangling_reference` | `MIRAKAN-PROJECT-RUNTIME_ENTRY_DANGLING_REFERENCE` | blocking／semantic／after_change |
| `diagnostic.project.runtime_entry.document_hash_mismatch` | `MIRAKAN-PROJECT-RUNTIME_ENTRY_DOCUMENT_HASH_MISMATCH` | blocking／conflict／after_change |
| `diagnostic.project.runtime_entry.semantic_hash_mismatch` | `MIRAKAN-PROJECT-RUNTIME_ENTRY_SEMANTIC_HASH_MISMATCH` | blocking／semantic／after_input |
| `diagnostic.project.runtime_entry.schema_mismatch` | `MIRAKAN-PROJECT-RUNTIME_ENTRY_SCHEMA_MISMATCH` | blocking／schema／after_input |
| `diagnostic.project.runtime_entry.explicit_target_mismatch` | `MIRAKAN-PROJECT-RUNTIME_ENTRY_EXPLICIT_TARGET_MISMATCH` | blocking／semantic／after_input |
| `diagnostic.project.runtime_entry.identity_mismatch` | `MIRAKAN-PROJECT-RUNTIME_ENTRY_IDENTITY_MISMATCH` | blocking／semantic／after_input |
| `diagnostic.runtime_scope.catalog_invalid` | `MIRAKAN-RUNTIME-SCOPE-CATALOG_INVALID` | blocking／schema／after_input |
| `diagnostic.runtime_scope.owner_unavailable` | `MIRAKAN-RUNTIME-SCOPE-OWNER_UNAVAILABLE` | blocking／semantic／after_change |
| `diagnostic.runtime_scope.version_hash_mismatch` | `MIRAKAN-RUNTIME-SCOPE-VERSION_HASH_MISMATCH` | blocking／conflict／after_change |

三Runtime Scope DiagnosticはGameplay Programming Modelのtarget Catalog／Owner／Dependency validator candidateが直接参照する。legacy migration専用Diagnosticはinitial targetへ登録しない。target merged Diagnostic Registry candidate inventoryは§8.1の21にWorld固有6とShooter固有9を加えたexact 36 recordである。これは§8.1の21と§8.2 Domain 23のunionから共有common 8を一度だけ数える同じ集合で、その内訳はtarget-Operation reachable exact 32とnon-operation exact 4のdisjoint unionである。

下表の六rowはset root確定後に生成するtarget external projection candidateである。現Repositoryでは一行もmaterializeしておらず、表中の`status=active`は将来Contract set内の参照可能状態を表すtarget Field値で、current Operation状態ではない。materialized sourceを持たないlegacy migration rowは表へ含めない。表中の`{id,1,contract_set_hash}`は三Fieldを持つexact `McdContractRefV1`、Service refは`{service_id,service_version,service_content_hash}`、Diagnostic refは`{diagnostic_id,code,diagnostic_version,diagnostic_content_hash}`である。各target行の外部MCD共通Envelopeは省略せず表内に全値を持つが、この外部形を`ContractSetMemberLocalRecordV1.canonical_payload`へ直接hashしない。

Snapshot compiler inputへのField projectionを次へ固定する。

| external field | snapshot-local exact field |
|---|---|
| `input_type; output_type; receipt_type; preconditions[]; postconditions[]; rate_limit_policy; requirement_refs[]` | 各`{kind,id,version}`の`ContractSetLocalRefV1`。`contract_set_hash`を除去 |
| `authority: TrustedServiceRefV1` | `{service_id,service_version}`の`TrustedServiceLocalRefV1`。`service_content_hash`を除去 |
| `errors[]: DiagnosticCodeRefV1` | `{diagnostic_id,diagnostic_version}`の`DiagnosticLocalRefV1`。DiagnosticはMCD kindではなく専用Registry recordで、code／content hashは同Registryのlocal record側で検証 |
| `validator_closure_ref` | `{closure_id,closure_version}`の`OperationValidatorClosureLocalRefV1`。closure content hashを除去 |
| closure内`operation_ref`／Service内`allowed_operation_refs[]` | Operationの`ContractSetLocalRefV1(kind=operation)` |
| closure内`validator_refs[]`／`reachable_error_refs[]` | `ValidatorLocalRefV1`／`DiagnosticLocalRefV1`。Validator／Diagnostic content hashを除去 |
| Validator内input／error ref | `ContractSetLocalRefV1(kind=type)`／`DiagnosticLocalRefV1` |

投影はFieldごとのtotal functionで、external ref一件をLocalRef一件へ変換し、配列cardinality／順序／duplicate statusを変えない。表示placeholder、名前prefix、current/latest lookup、hash欠落の補完を禁止する。compiler fixtureは六Operationについてexternal ref集合からhash Fieldだけを除いたlogical identity集合とlocal payload集合、root確定後に再materializeしたexternal集合をField別にset equalityにし、external hashをLocal preimageへ残す、kind変更、Service／Validatorのhash付き相互edge、配列一件欠落／extra／reorderを各一原因で拒否する。

| Operation MCD共通Envelope exact value | Operation固有Field exact value |
|---|---|
| `mcd_version=1; kind=operation; id=operation.project.runtime_entry.create; version=1; status=active; title=Create Runtime Entry; description=Create one identity-consistent Runtime Entry Document; owners=[owner.core.project_state]; requirement_refs=[]; rationale_refs=[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]; since_contract_set=1; supersedes=[]; tags=[authoring,runtime_entry]` | `operation_kind=command; input_type={type.project.runtime_entry.create_input,1,contract_set_hash}; output_type={type.project.runtime_entry.mutation_result,1,contract_set_hash}; authority={service_id=service.authoring_command_gateway,service_version=1,service_content_hash}; risk_class=R2; side_effects=[authoring]; idempotency=idempotent_with_key; transaction=authoring_changeset; preconditions=[{policy.operation.project.runtime_entry.create.precondition,1,contract_set_hash}]; postconditions=[{policy.operation.project.runtime_entry.create.postcondition,1,contract_set_hash}]; errors=[{diagnostic.conflict.revision_mismatch,MIRAKAN-CONFLICT-REVISION_MISMATCH,1,diagnostic_content_hash},{diagnostic.authorization.denied,MIRAKAN-AUTHORIZATION-DENIED,1,diagnostic_content_hash},{diagnostic.approval.required,MIRAKAN-APPROVAL-REQUIRED,1,diagnostic_content_hash},{diagnostic.authoring.lock_conflict,MIRAKAN-AUTHORING-LOCK_CONFLICT,1,diagnostic_content_hash},{diagnostic.mcd.operation_predicate_invalid,MIRAKAN-MCD-OPERATION-PREDICATE_INVALID,1,diagnostic_content_hash},{diagnostic.operation.timeout,MIRAKAN-OPERATION-TIMEOUT,1,diagnostic_content_hash},{diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash},{diagnostic.operation.idempotency_key_reuse,MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.invalid,MIRAKAN-PROJECT-RUNTIME_ENTRY_INVALID,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.target_unresolved,MIRAKAN-PROJECT-RUNTIME_ENTRY_TARGET_UNRESOLVED,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.default_ambiguous,MIRAKAN-PROJECT-RUNTIME_ENTRY_DEFAULT_AMBIGUOUS,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.branch_field_conflict,MIRAKAN-PROJECT-RUNTIME_ENTRY_BRANCH_FIELD_CONFLICT,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.dangling_reference,MIRAKAN-PROJECT-RUNTIME_ENTRY_DANGLING_REFERENCE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.semantic_hash_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SEMANTIC_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.schema_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SCHEMA_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.identity_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_IDENTITY_MISMATCH,1,diagnostic_content_hash}]; validator_closure_ref={validator_closure.operation.project.runtime_entry.create,1,closure_content_hash}; timeout_ms=30000; rate_limit_policy={policy.authoring.runtime_entry.rate_limit,1,contract_set_hash}; audit_level=full_redacted; provider_exposure=mcp_proposal; receipt_type={type.project.runtime_entry.mutation_receipt,1,contract_set_hash}` |
| `mcd_version=1; kind=operation; id=operation.project.runtime_entry.update; version=1; status=active; title=Update Runtime Entry; description=Update one Runtime Entry Document while preserving its identity; owners=[owner.core.project_state]; requirement_refs=[]; rationale_refs=[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]; since_contract_set=1; supersedes=[]; tags=[authoring,runtime_entry]` | `operation_kind=command; input_type={type.project.runtime_entry.update_input,1,contract_set_hash}; output_type={type.project.runtime_entry.mutation_result,1,contract_set_hash}; authority={service_id=service.authoring_command_gateway,service_version=1,service_content_hash}; risk_class=R2; side_effects=[authoring]; idempotency=idempotent_with_key; transaction=authoring_changeset; preconditions=[{policy.operation.project.runtime_entry.update.precondition,1,contract_set_hash}]; postconditions=[{policy.operation.project.runtime_entry.update.postcondition,1,contract_set_hash}]; errors=[{diagnostic.conflict.revision_mismatch,MIRAKAN-CONFLICT-REVISION_MISMATCH,1,diagnostic_content_hash},{diagnostic.authorization.denied,MIRAKAN-AUTHORIZATION-DENIED,1,diagnostic_content_hash},{diagnostic.approval.required,MIRAKAN-APPROVAL-REQUIRED,1,diagnostic_content_hash},{diagnostic.authoring.lock_conflict,MIRAKAN-AUTHORING-LOCK_CONFLICT,1,diagnostic_content_hash},{diagnostic.mcd.operation_predicate_invalid,MIRAKAN-MCD-OPERATION-PREDICATE_INVALID,1,diagnostic_content_hash},{diagnostic.operation.timeout,MIRAKAN-OPERATION-TIMEOUT,1,diagnostic_content_hash},{diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash},{diagnostic.operation.idempotency_key_reuse,MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.invalid,MIRAKAN-PROJECT-RUNTIME_ENTRY_INVALID,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.target_unresolved,MIRAKAN-PROJECT-RUNTIME_ENTRY_TARGET_UNRESOLVED,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.default_ambiguous,MIRAKAN-PROJECT-RUNTIME_ENTRY_DEFAULT_AMBIGUOUS,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.branch_field_conflict,MIRAKAN-PROJECT-RUNTIME_ENTRY_BRANCH_FIELD_CONFLICT,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.dangling_reference,MIRAKAN-PROJECT-RUNTIME_ENTRY_DANGLING_REFERENCE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.document_hash_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_DOCUMENT_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.semantic_hash_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SEMANTIC_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.schema_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SCHEMA_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.identity_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_IDENTITY_MISMATCH,1,diagnostic_content_hash}]; validator_closure_ref={validator_closure.operation.project.runtime_entry.update,1,closure_content_hash}; timeout_ms=30000; rate_limit_policy={policy.authoring.runtime_entry.rate_limit,1,contract_set_hash}; audit_level=full_redacted; provider_exposure=mcp_proposal; receipt_type={type.project.runtime_entry.mutation_receipt,1,contract_set_hash}` |
| `mcd_version=1; kind=operation; id=operation.project.runtime_target_selector.create; version=1; status=active; title=Create Runtime Target Selector; description=Create one identity-consistent Target Selector Document without attaching it to an entry; owners=[owner.core.project_state]; requirement_refs=[]; rationale_refs=[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]; since_contract_set=1; supersedes=[]; tags=[authoring,runtime_entry,target_selector]` | `operation_kind=command; input_type={type.project.runtime_target_selector.create_input,1,contract_set_hash}; output_type={type.project.runtime_entry.mutation_result,1,contract_set_hash}; authority={service_id=service.authoring_command_gateway,service_version=1,service_content_hash}; risk_class=R2; side_effects=[authoring]; idempotency=idempotent_with_key; transaction=authoring_changeset; preconditions=[{policy.operation.project.runtime_target_selector.create.precondition,1,contract_set_hash}]; postconditions=[{policy.operation.project.runtime_target_selector.create.postcondition,1,contract_set_hash}]; errors=[{diagnostic.conflict.revision_mismatch,MIRAKAN-CONFLICT-REVISION_MISMATCH,1,diagnostic_content_hash},{diagnostic.authorization.denied,MIRAKAN-AUTHORIZATION-DENIED,1,diagnostic_content_hash},{diagnostic.approval.required,MIRAKAN-APPROVAL-REQUIRED,1,diagnostic_content_hash},{diagnostic.authoring.lock_conflict,MIRAKAN-AUTHORING-LOCK_CONFLICT,1,diagnostic_content_hash},{diagnostic.mcd.operation_predicate_invalid,MIRAKAN-MCD-OPERATION-PREDICATE_INVALID,1,diagnostic_content_hash},{diagnostic.operation.timeout,MIRAKAN-OPERATION-TIMEOUT,1,diagnostic_content_hash},{diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash},{diagnostic.operation.idempotency_key_reuse,MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.invalid,MIRAKAN-PROJECT-RUNTIME_ENTRY_INVALID,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.target_unresolved,MIRAKAN-PROJECT-RUNTIME_ENTRY_TARGET_UNRESOLVED,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.dangling_reference,MIRAKAN-PROJECT-RUNTIME_ENTRY_DANGLING_REFERENCE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.semantic_hash_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SEMANTIC_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.schema_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SCHEMA_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.identity_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_IDENTITY_MISMATCH,1,diagnostic_content_hash}]; validator_closure_ref={validator_closure.operation.project.runtime_target_selector.create,1,closure_content_hash}; timeout_ms=30000; rate_limit_policy={policy.authoring.runtime_entry.rate_limit,1,contract_set_hash}; audit_level=full_redacted; provider_exposure=mcp_proposal; receipt_type={type.project.runtime_entry.mutation_receipt,1,contract_set_hash}` |
| `mcd_version=1; kind=operation; id=operation.project.runtime_target_selector.update; version=1; status=active; title=Update Runtime Target Selector; description=Update one Target Selector Document and revalidate default coverage; owners=[owner.core.project_state]; requirement_refs=[]; rationale_refs=[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]; since_contract_set=1; supersedes=[]; tags=[authoring,runtime_entry,target_selector]` | `operation_kind=command; input_type={type.project.runtime_target_selector.update_input,1,contract_set_hash}; output_type={type.project.runtime_entry.mutation_result,1,contract_set_hash}; authority={service_id=service.authoring_command_gateway,service_version=1,service_content_hash}; risk_class=R2; side_effects=[authoring]; idempotency=idempotent_with_key; transaction=authoring_changeset; preconditions=[{policy.operation.project.runtime_target_selector.update.precondition,1,contract_set_hash}]; postconditions=[{policy.operation.project.runtime_target_selector.update.postcondition,1,contract_set_hash}]; errors=[{diagnostic.conflict.revision_mismatch,MIRAKAN-CONFLICT-REVISION_MISMATCH,1,diagnostic_content_hash},{diagnostic.authorization.denied,MIRAKAN-AUTHORIZATION-DENIED,1,diagnostic_content_hash},{diagnostic.approval.required,MIRAKAN-APPROVAL-REQUIRED,1,diagnostic_content_hash},{diagnostic.authoring.lock_conflict,MIRAKAN-AUTHORING-LOCK_CONFLICT,1,diagnostic_content_hash},{diagnostic.mcd.operation_predicate_invalid,MIRAKAN-MCD-OPERATION-PREDICATE_INVALID,1,diagnostic_content_hash},{diagnostic.operation.timeout,MIRAKAN-OPERATION-TIMEOUT,1,diagnostic_content_hash},{diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash},{diagnostic.operation.idempotency_key_reuse,MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.invalid,MIRAKAN-PROJECT-RUNTIME_ENTRY_INVALID,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.target_unresolved,MIRAKAN-PROJECT-RUNTIME_ENTRY_TARGET_UNRESOLVED,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.default_ambiguous,MIRAKAN-PROJECT-RUNTIME_ENTRY_DEFAULT_AMBIGUOUS,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.dangling_reference,MIRAKAN-PROJECT-RUNTIME_ENTRY_DANGLING_REFERENCE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.document_hash_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_DOCUMENT_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.semantic_hash_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SEMANTIC_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.schema_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SCHEMA_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.identity_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_IDENTITY_MISMATCH,1,diagnostic_content_hash}]; validator_closure_ref={validator_closure.operation.project.runtime_target_selector.update,1,closure_content_hash}; timeout_ms=30000; rate_limit_policy={policy.authoring.runtime_entry.rate_limit,1,contract_set_hash}; audit_level=full_redacted; provider_exposure=mcp_proposal; receipt_type={type.project.runtime_entry.mutation_receipt,1,contract_set_hash}` |
| `mcd_version=1; kind=operation; id=operation.project.runtime_entry_activation_policy.create; version=1; status=active; title=Create Runtime Entry Activation Policy; description=Create one identity-consistent Runtime Entry Activation Policy Document; owners=[owner.core.project_state]; requirement_refs=[]; rationale_refs=[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]; since_contract_set=1; supersedes=[]; tags=[activation_policy,authoring,runtime_entry]` | `operation_kind=command; input_type={type.project.runtime_entry_activation_policy.create_input,1,contract_set_hash}; output_type={type.project.runtime_entry.mutation_result,1,contract_set_hash}; authority={service_id=service.authoring_command_gateway,service_version=1,service_content_hash}; risk_class=R2; side_effects=[authoring]; idempotency=idempotent_with_key; transaction=authoring_changeset; preconditions=[{policy.operation.project.runtime_entry_activation_policy.create.precondition,1,contract_set_hash}]; postconditions=[{policy.operation.project.runtime_entry_activation_policy.create.postcondition,1,contract_set_hash}]; errors=[{diagnostic.conflict.revision_mismatch,MIRAKAN-CONFLICT-REVISION_MISMATCH,1,diagnostic_content_hash},{diagnostic.authorization.denied,MIRAKAN-AUTHORIZATION-DENIED,1,diagnostic_content_hash},{diagnostic.approval.required,MIRAKAN-APPROVAL-REQUIRED,1,diagnostic_content_hash},{diagnostic.authoring.lock_conflict,MIRAKAN-AUTHORING-LOCK_CONFLICT,1,diagnostic_content_hash},{diagnostic.mcd.operation_predicate_invalid,MIRAKAN-MCD-OPERATION-PREDICATE_INVALID,1,diagnostic_content_hash},{diagnostic.operation.timeout,MIRAKAN-OPERATION-TIMEOUT,1,diagnostic_content_hash},{diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash},{diagnostic.operation.idempotency_key_reuse,MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.invalid,MIRAKAN-PROJECT-RUNTIME_ENTRY_INVALID,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.dangling_reference,MIRAKAN-PROJECT-RUNTIME_ENTRY_DANGLING_REFERENCE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.semantic_hash_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SEMANTIC_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.schema_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SCHEMA_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.identity_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_IDENTITY_MISMATCH,1,diagnostic_content_hash}]; validator_closure_ref={validator_closure.operation.project.runtime_entry_activation_policy.create,1,closure_content_hash}; timeout_ms=30000; rate_limit_policy={policy.authoring.runtime_entry.rate_limit,1,contract_set_hash}; audit_level=full_redacted; provider_exposure=mcp_proposal; receipt_type={type.project.runtime_entry.mutation_receipt,1,contract_set_hash}` |
| `mcd_version=1; kind=operation; id=operation.project.runtime_entry_activation_policy.update; version=1; status=active; title=Update Runtime Entry Activation Policy; description=Update one Runtime Entry Activation Policy Document and invalidate consumers; owners=[owner.core.project_state]; requirement_refs=[]; rationale_refs=[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]; since_contract_set=1; supersedes=[]; tags=[activation_policy,authoring,runtime_entry]` | `operation_kind=command; input_type={type.project.runtime_entry_activation_policy.update_input,1,contract_set_hash}; output_type={type.project.runtime_entry.mutation_result,1,contract_set_hash}; authority={service_id=service.authoring_command_gateway,service_version=1,service_content_hash}; risk_class=R2; side_effects=[authoring]; idempotency=idempotent_with_key; transaction=authoring_changeset; preconditions=[{policy.operation.project.runtime_entry_activation_policy.update.precondition,1,contract_set_hash}]; postconditions=[{policy.operation.project.runtime_entry_activation_policy.update.postcondition,1,contract_set_hash}]; errors=[{diagnostic.conflict.revision_mismatch,MIRAKAN-CONFLICT-REVISION_MISMATCH,1,diagnostic_content_hash},{diagnostic.authorization.denied,MIRAKAN-AUTHORIZATION-DENIED,1,diagnostic_content_hash},{diagnostic.approval.required,MIRAKAN-APPROVAL-REQUIRED,1,diagnostic_content_hash},{diagnostic.authoring.lock_conflict,MIRAKAN-AUTHORING-LOCK_CONFLICT,1,diagnostic_content_hash},{diagnostic.mcd.operation_predicate_invalid,MIRAKAN-MCD-OPERATION-PREDICATE_INVALID,1,diagnostic_content_hash},{diagnostic.operation.timeout,MIRAKAN-OPERATION-TIMEOUT,1,diagnostic_content_hash},{diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash},{diagnostic.operation.idempotency_key_reuse,MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.invalid,MIRAKAN-PROJECT-RUNTIME_ENTRY_INVALID,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.dangling_reference,MIRAKAN-PROJECT-RUNTIME_ENTRY_DANGLING_REFERENCE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.document_hash_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_DOCUMENT_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.semantic_hash_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SEMANTIC_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.schema_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SCHEMA_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.identity_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_IDENTITY_MISMATCH,1,diagnostic_content_hash}]; validator_closure_ref={validator_closure.operation.project.runtime_entry_activation_policy.update,1,closure_content_hash}; timeout_ms=30000; rate_limit_policy={policy.authoring.runtime_entry.rate_limit,1,contract_set_hash}; audit_level=full_redacted; provider_exposure=mcp_proposal; receipt_type={type.project.runtime_entry.mutation_receipt,1,contract_set_hash}` |

pre／postconditionのtarget集合は新kindではなく、target六Operation candidateが参照するexact 12件の`policy` MCD candidateである。target 12 recordは`mcd_version=1`、`kind=policy`、`version=1`、`status=active`、`requirement_refs=[]`、`since_contract_set=1`、`supersedes=[]`、`tags=[operation_predicate,pure]`、`evaluation_mode=pure`、`side_effects=[]`、`result_type={id=type.operation.predicate_result,version=1,contract_set_hash}`を持ち、表が各recordの残り全値を明示する。`status=active`は将来同じContract setへmaterializeされた場合の参照可能状態であり、現Repositoryのcurrent Policy集合は`[]`である。preconditionのinput typeはexact Operation input ref／hashとread-only before snapshot refs／hashesを持つ`type.operation.precondition_evaluation_input` version 1、postconditionはrequest hash、prepared output、before snapshot、未発行Staging snapshot、Prepared Commit Envelopeを持つinitial `type.operation.postcondition_evaluation_input` version 1である。どちらもcanonical immutable valueだけを入力にし、Project／Registry／clock／networkを評価中にqueryしない。postconditionへCommit Markerまたは公開Receipt refを渡さない。

| policy `id` | title／description | owners／rationale／exact input type |
|---|---|---|
| `policy.operation.project.runtime_entry.create.precondition` | Create Runtime Entry Precondition／expected Project revision、draft hash、allocation、selector、policyを検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.precondition_evaluation_input,1,contract_set_hash}` |
| `policy.operation.project.runtime_entry.create.postcondition` | Create Runtime Entry Postcondition／identity三者一致の未発行Document一件とrevision増分候補を検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.postcondition_evaluation_input,1,contract_set_hash}` |
| `policy.operation.project.runtime_entry.update.precondition` | Update Runtime Entry Precondition／current ref、revision、content hash、semantic hashを検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.precondition_evaluation_input,1,contract_set_hash}` |
| `policy.operation.project.runtime_entry.update.postcondition` | Update Runtime Entry Postcondition／同ID新revision候補とconsumer invalidationを検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.postcondition_evaluation_input,1,contract_set_hash}` |
| `policy.operation.project.runtime_target_selector.create.precondition` | Create Target Selector Precondition／draft、Target集合、Project revisionを検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.precondition_evaluation_input,1,contract_set_hash}` |
| `policy.operation.project.runtime_target_selector.create.postcondition` | Create Target Selector Postcondition／未発行selector identity、hash、Project containmentを検証。entry attach／default coverageは変更しない | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.postcondition_evaluation_input,1,contract_set_hash}` |
| `policy.operation.project.runtime_target_selector.update.precondition` | Update Target Selector Precondition／current selector identity、hash、Target集合を検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.precondition_evaluation_input,1,contract_set_hash}` |
| `policy.operation.project.runtime_target_selector.update.postcondition` | Update Target Selector Postcondition／同ID新revision候補と全consumer coverageを検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.postcondition_evaluation_input,1,contract_set_hash}` |
| `policy.operation.project.runtime_entry_activation_policy.create.precondition` | Create Activation Policy Precondition／closed policy draft、hash、Project revisionを検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.precondition_evaluation_input,1,contract_set_hash}` |
| `policy.operation.project.runtime_entry_activation_policy.create.postcondition` | Create Activation Policy Postcondition／未発行policy identity三者一致と新revision候補を検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.postcondition_evaluation_input,1,contract_set_hash}` |
| `policy.operation.project.runtime_entry_activation_policy.update.precondition` | Update Activation Policy Precondition／current policy ref、revision、content／policy hashを検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.precondition_evaluation_input,1,contract_set_hash}` |
| `policy.operation.project.runtime_entry_activation_policy.update.postcondition` | Update Activation Policy Postcondition／同ID新revision候補とconsumer invalidationを検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.postcondition_evaluation_input,1,contract_set_hash}` |

target六`OperationValidatorClosureV1` candidateを次へ固定する。表内の各Validator refは`{validator_id,validator_version=1,validator_content_hash}`、closure refは`{closure_id,closure_version=1,closure_content_hash}`である。`reachable_error_refs`は表が指す当該Operation candidateの完全な`errors[]`四Fieldref集合そのもので、将来Validator Registryの`error_refs[]` unionからmaterializeし、別の共通error categoryやcode prefix展開を行わない。

| closure ref／operation ref | exact `validator_refs[]` | `reachable_error_refs` |
|---|---|---|
| `validator_closure.operation.project.runtime_entry.create`／`operation.project.runtime_entry.create` v1 | `validator.operation.request_envelope; validator.operation.authorization; validator.operation.approval; validator.operation.revision_and_lock; validator.operation.pure_predicate; validator.operation.timeout_and_rate_limit; validator.project.runtime_entry.create_semantics; validator.project.runtime_entry.create_postcondition` | exact `errors[]` set of `operation.project.runtime_entry.create` v1 |
| `validator_closure.operation.project.runtime_entry.update`／`operation.project.runtime_entry.update` v1 | `validator.operation.request_envelope; validator.operation.authorization; validator.operation.approval; validator.operation.revision_and_lock; validator.operation.pure_predicate; validator.operation.timeout_and_rate_limit; validator.project.runtime_entry.update_semantics; validator.project.runtime_entry.update_postcondition` | exact `errors[]` set of `operation.project.runtime_entry.update` v1 |
| `validator_closure.operation.project.runtime_target_selector.create`／`operation.project.runtime_target_selector.create` v1 | `validator.operation.request_envelope; validator.operation.authorization; validator.operation.approval; validator.operation.revision_and_lock; validator.operation.pure_predicate; validator.operation.timeout_and_rate_limit; validator.project.runtime_target_selector.create_semantics; validator.project.runtime_target_selector.create_postcondition` | exact `errors[]` set of `operation.project.runtime_target_selector.create` v1 |
| `validator_closure.operation.project.runtime_target_selector.update`／`operation.project.runtime_target_selector.update` v1 | `validator.operation.request_envelope; validator.operation.authorization; validator.operation.approval; validator.operation.revision_and_lock; validator.operation.pure_predicate; validator.operation.timeout_and_rate_limit; validator.project.runtime_target_selector.update_semantics; validator.project.runtime_target_selector.update_postcondition` | exact `errors[]` set of `operation.project.runtime_target_selector.update` v1 |
| `validator_closure.operation.project.runtime_entry_activation_policy.create`／`operation.project.runtime_entry_activation_policy.create` v1 | `validator.operation.request_envelope; validator.operation.authorization; validator.operation.approval; validator.operation.revision_and_lock; validator.operation.pure_predicate; validator.operation.timeout_and_rate_limit; validator.project.runtime_entry_activation_policy.create_semantics; validator.project.runtime_entry_activation_policy.create_postcondition` | exact `errors[]` set of `operation.project.runtime_entry_activation_policy.create` v1 |
| `validator_closure.operation.project.runtime_entry_activation_policy.update`／`operation.project.runtime_entry_activation_policy.update` v1 | `validator.operation.request_envelope; validator.operation.authorization; validator.operation.approval; validator.operation.revision_and_lock; validator.operation.pure_predicate; validator.operation.timeout_and_rate_limit; validator.project.runtime_entry_activation_policy.update_semantics; validator.project.runtime_entry_activation_policy.update_postcondition` | exact `errors[]` set of `operation.project.runtime_entry_activation_policy.update` v1 |

target六closure candidateから到達する全16 Validator local recordを、将来の同一activation transactionでmaterializeする。target 16 row candidateは`validator_version=1`、`implementation_artifact_ref={artifact.validator.<validator_id suffix>,1,artifact_hash}`、表のinput Type LocalRef、表のDiagnostic LocalRef配列、self-excluding `validator_local_content_hash`を持つ。表中のDiagnosticはすべて`kind=remediation`ではなくDiagnostic Registryのtyped LocalRefで、ID／version 1を保存する。空欄、省略、code prefix展開、Operation error集合への遅延参照を許可しない。set root後の外部16 `ValidatorRecordV1`は同じlogical rowをroot付きType／Diagnostic refsへ投影し、各外部`validator_content_hash`を別計算する。

| Validator ID | input Type LocalRef | exact `error_local_refs[]` Diagnostic ID |
|---|---|---|
| `validator.operation.request_envelope` | `type.operation.precondition_evaluation_input@1` | `diagnostic.operation.idempotency_key_reuse` |
| `validator.operation.authorization` | `type.operation.precondition_evaluation_input@1` | `diagnostic.authorization.denied` |
| `validator.operation.approval` | `type.operation.precondition_evaluation_input@1` | `diagnostic.approval.required` |
| `validator.operation.revision_and_lock` | `type.operation.precondition_evaluation_input@1` | `diagnostic.conflict.revision_mismatch; diagnostic.authoring.lock_conflict` |
| `validator.operation.pure_predicate` | `type.operation.precondition_evaluation_input@1` | `diagnostic.mcd.operation_predicate_invalid` |
| `validator.operation.timeout_and_rate_limit` | `type.operation.precondition_evaluation_input@1` | `diagnostic.operation.timeout; diagnostic.operation.rate_limit_exceeded` |
| `validator.project.runtime_entry.create_semantics` | `type.project.runtime_entry.create_input@1` | `diagnostic.project.runtime_entry.invalid; diagnostic.project.runtime_entry.target_unresolved; diagnostic.project.runtime_entry.default_ambiguous; diagnostic.project.runtime_entry.branch_field_conflict; diagnostic.project.runtime_entry.dangling_reference` |
| `validator.project.runtime_entry.create_postcondition` | `type.operation.postcondition_evaluation_input@1` | `diagnostic.project.runtime_entry.semantic_hash_mismatch; diagnostic.project.runtime_entry.schema_mismatch; diagnostic.project.runtime_entry.identity_mismatch` |
| `validator.project.runtime_entry.update_semantics` | `type.project.runtime_entry.update_input@1` | `diagnostic.project.runtime_entry.invalid; diagnostic.project.runtime_entry.target_unresolved; diagnostic.project.runtime_entry.default_ambiguous; diagnostic.project.runtime_entry.branch_field_conflict; diagnostic.project.runtime_entry.dangling_reference` |
| `validator.project.runtime_entry.update_postcondition` | `type.operation.postcondition_evaluation_input@1` | `diagnostic.project.runtime_entry.document_hash_mismatch; diagnostic.project.runtime_entry.semantic_hash_mismatch; diagnostic.project.runtime_entry.schema_mismatch; diagnostic.project.runtime_entry.identity_mismatch` |
| `validator.project.runtime_target_selector.create_semantics` | `type.project.runtime_target_selector.create_input@1` | `diagnostic.project.runtime_entry.invalid; diagnostic.project.runtime_entry.target_unresolved; diagnostic.project.runtime_entry.dangling_reference` |
| `validator.project.runtime_target_selector.create_postcondition` | `type.operation.postcondition_evaluation_input@1` | `diagnostic.project.runtime_entry.semantic_hash_mismatch; diagnostic.project.runtime_entry.schema_mismatch; diagnostic.project.runtime_entry.identity_mismatch` |
| `validator.project.runtime_target_selector.update_semantics` | `type.project.runtime_target_selector.update_input@1` | `diagnostic.project.runtime_entry.invalid; diagnostic.project.runtime_entry.target_unresolved; diagnostic.project.runtime_entry.default_ambiguous; diagnostic.project.runtime_entry.dangling_reference` |
| `validator.project.runtime_target_selector.update_postcondition` | `type.operation.postcondition_evaluation_input@1` | `diagnostic.project.runtime_entry.document_hash_mismatch; diagnostic.project.runtime_entry.semantic_hash_mismatch; diagnostic.project.runtime_entry.schema_mismatch; diagnostic.project.runtime_entry.identity_mismatch` |
| `validator.project.runtime_entry_activation_policy.create_semantics` | `type.project.runtime_entry_activation_policy.create_input@1` | `diagnostic.project.runtime_entry.invalid; diagnostic.project.runtime_entry.dangling_reference` |
| `validator.project.runtime_entry_activation_policy.create_postcondition` | `type.operation.postcondition_evaluation_input@1` | `diagnostic.project.runtime_entry.semantic_hash_mismatch; diagnostic.project.runtime_entry.schema_mismatch; diagnostic.project.runtime_entry.identity_mismatch` |
| `validator.project.runtime_entry_activation_policy.update_semantics` | `type.project.runtime_entry_activation_policy.update_input@1` | `diagnostic.project.runtime_entry.invalid; diagnostic.project.runtime_entry.dangling_reference` |
| `validator.project.runtime_entry_activation_policy.update_postcondition` | `type.operation.postcondition_evaluation_input@1` | `diagnostic.project.runtime_entry.document_hash_mismatch; diagnostic.project.runtime_entry.semantic_hash_mismatch; diagnostic.project.runtime_entry.schema_mismatch; diagnostic.project.runtime_entry.identity_mismatch` |
将来materializeするRegistryはtarget 16 rowをValidator IDのNFC UTF-8 byte順、version順へstrict sortする。各Artifact refは実行Targetのsigned implementation inventoryへexact一件、input LocalRefは同じContract setのType recordへexact一件、各Diagnostic LocalRefは同じContract setのDiagnostic Registry recordへexact一件解決する。16件以外のrow、missing、extra、duplicate、same-ID別hash、非canonical order、artifact／input／error hash不一致をRegistry全体のcompile errorにする。

各target candidate rowは`operation_local_ref`と`reachable_error_refs`を含むcanonical recordを保存する。fixtureは各Domain Validatorから一codeを削除、到達不能codeをOperationへ追加、ID同じcode違い、code同じID違い、Diagnostic hash stale、Validator hash staleを一原因ずつ注入し、set／ref equality不成立でRegistry全体を拒否する。Runtime Entry create／updateは`MIRAKAN-PROJECT-RUNTIME_ENTRY_BRANCH_FIELD_CONFLICT`、Runtime Entry create／updateとTarget Selector updateは`MIRAKAN-PROJECT-RUNTIME_ENTRY_DEFAULT_AMBIGUOUS`がsemantic Validatorから到達することをfixtureで証明する。Target Selector createは既存entryへattachしないためdefault coverageを変更せず、このDiagnosticをerrors／reachable setへ含めない。

Request hashの解釈を説明文または実装名へ委ねず、次のnamed algorithmと一件Registryへ固定する。これはMCD kindを追加せず、Contract set外の独立definition rootである。

```text
NamedAlgorithmRefV1
  algorithm_id
  algorithm_version: positive uint32
  algorithm_content_hash: SHA-256

OperationRequestHashAlgorithmV1
  algorithm_id: algorithm.operation_request_hash
  algorithm_version: 1
  algorithm_kind: hash
  input_projection: selected_named_input_all_actual_fields
  excluded_fields: [request_hash]
  canonical_encoding_profile:
    exact McdCanonicalBinaryV1(
      profile_id=profile.mcd_canonical_binary,
      profile_version=1,
      profile_content_hash)
  domain_separator_ascii: MIRAKAN_OPERATION_REQUEST_V1
  length_framing: uint32_be_single_payload
  digest: sha256
  output: sha256_bytes
  algorithm_content_hash: SHA-256

NamedAlgorithmRegistryV1
  registry_id: named_algorithm.registry.active
  registry_version: positive uint32
  record_count: uint32
  records[1..1024]: named algorithm records
  registry_content_hash: SHA-256

OperationRequestAlgorithmBindingV1
  foundation_definition_closure_ref:
    FoundationDefinitionClosureRefV1
  request_hash_algorithm_ref:
    NamedAlgorithmRefV1(
      algorithm_id=algorithm.operation_request_hash,
      algorithm_version=positive uint32 selected by the
      referenced Closure's Named Algorithm Registry)
```

`algorithm_content_hash = SHA-256(ASCII "MIRAKAN_NAMED_ALGORITHM_RECORD_V1" || uint32_be(len(RFC 8785 JCS(closed algorithm record excluding algorithm_content_hash))) || RFC 8785 JCS(closed algorithm record excluding algorithm_content_hash))`、`registry_content_hash = SHA-256(ASCII "MIRAKAN_NAMED_ALGORITHM_REGISTRY_V1" || uint32_be(len(RFC 8785 JCS(closed registry excluding registry_content_hash))) || RFC 8785 JCS(closed registry excluding registry_content_hash))`とする。全stringはNFC、digestはlowercase hexadecimal exact 64文字、uint32はJSON safe integer、unknown Fieldは禁止する。Registryは`algorithm_id`のNFC UTF-8 bytes／version順にstrict sortし、`record_count`を配列長と一致させる。target初期Registry candidateは`registry_version=1`、`record_count=1`、exact集合は上記V1一recordだけであり、その`canonical_encoding_profile`は§7.4の完成Profile object／hashとbyte equalityにする。現RepositoryにRegistry instanceは存在しない。実装関数名、Provider、platform、bare `mcd_canonical_binary` token、任意digest aliasからAlgorithmまたはencoding規則を生成しない。missing／extra／duplicate、same ID／version different hash、Profile version／hash mismatch、self-hash混入を拒否する。

将来materializeされた全retained Registry revisionにおいて同じ`{algorithm_id, algorithm_version}`の完成record bytesと`algorithm_content_hash`はbyte-identicalでなければならない。その時点のselected Registryは各`algorithm_id`にexact一つのselected versionだけを持ち、同じAlgorithm IDの複数version併存を拒否する。意味Fieldを一つでも変更する場合は`algorithm_version`をexact `N+1`へ進め、新record、新Named Algorithm Registry root、新Foundation Definition Closureを発行する。同じID／versionで別hash、Registry versionだけ進めたin-place再定義、旧record削除後のversion再利用を拒否する。旧versionはそれを含む旧Registry root／Foundation Closureと共にretentionし、selected Registryへ併存させない。Registryのrow集合が変わるときだけ`registry_version=N+1`とし、同一bytesを別Registry versionとして発行しない。

`OperationRequestAlgorithmBindingV1`はClosure refが解決するNamed Algorithm Registryからexact Algorithm refを一件解決し、隣接Algorithm hashとinline `McdCanonicalBinaryV1` Profile hashを再計算する。Binding schemaはpositive Algorithm versionを持ち、initial target値をexact 1にする。materialized consumer発行後にsemantic変更する場合はAlgorithm recordをV2、Registry rootとFoundation ClosureをN+1へ進めるが、Binding V1のField shapeは変更しない。Closureが指すContract set rootはOperation inputの実行時selected MCD ref rootと一致させる。さらに`MutationAuthorizationBindingV1`が参照する署名済みTask Authorization Envelopeを解決し、bindingのClosure ref、Envelopeの`foundation_definition_closure_ref`、Envelopeへinline束縛した`CurrentControlPlaneBaselineBindingV1`から解決するClosure、dispatch直前のcurrent Product operational headのBaseline bindingから解決するClosureを全てbyte equalityにする。Algorithm refだけをambient current Registryから補完すること、同じContract Setを持つretained／stale別Closure、別Owner rootまたはAlgorithm root、latest lookup、ID／versionだけの一致を拒否する。このbindingは全状態変更inputの`MutationAuthorizationBindingV1.request_hash_algorithm_binding`と、その公開Receipt chainの`PublishedReceiptMaterializationContextV1`／`PreparedReceiptPublicationBindingV1`へbyte-exactに保存する。

唯一のcross-root例外は、Operation input schemaが`retained_source_mcd_ref`と`source_foundation_definition_closure_ref`を両方requiredとして明示したoffline migration branchである。`retained_source_mcd_ref`はexact `McdContractRefV1` Field名であり、Input、Inputがcontent refで選択するmigration contribution、Prepared Domain Receiptに同Fieldが複写される場合は全出現値をbyte equalityにする。別名、schema annotationだけ、bare ID、`source_*`というprefix推測では代用しない。全`retained_source_mcd_ref`のContract set、同refから読むSource recordのOwner ref、source ClosureのNamed Algorithm refはその一つのretained source Closureだけへ解決する。current execution Operation／input／output／Policy／Validator／Diagnostic／destination schema、Contributionのdestination ref／policy、Algorithm bindingの全refはdestination current Closureへ解決する。source Closure自体はrequest semantic inputに含め、全retained source refと同時代rootでなければならない。任意の別root ref、source Closure省略、source refをcurrent rootへalias、destination refをsource rootへdowngrade、複数source Closure、同名Fieldの値差、bare retained latestを拒否する。この例外はmigration Operationがsource bytesをread-onlyで検査するためだけで、source Contractのdispatch authorityまたはcurrent activationを復活させない。

Operation認可とrequest identityの唯一のDAGは本段落である。全Operation inputは選択したnamed input typeから`OperationIntentPayloadV1 {input_type_ref, operation_ref, risk_class, semantic_input_fields}`を作る。`semantic_input_fields`はそのschemaに存在する全意味Fieldをfield ID／presence discriminator込みで持ち、`operation_intent_hash`、`request_hash`、`MutationAuthorizationBindingV1`全体だけを除外する。別置き`authorization_ref`、anonymous approval shape、evidence hashをintentへ残さない。`operation_intent_hash = SHA-256(ASCII "MIRAKAN_OPERATION_INTENT_V1" || uint32_be(len(intent canonical bytes excluding operation_intent_hash)) || intent canonical bytes)`とし、count／array lengthはMCD canonical bytesへ明示、self-exclusionはintent hash自身だけである。Task Authorizationはこのintentを直接署名せずscope coverageを与える。Operation Mutation ApprovalまたはPolicy Service発行のPredelegation Useだけが完成intent hashをpayloadへ含めて署名する。

R2～R5のauthoritative Project／Source／Release変更inputは再計算済みintent hashとexact `MutationAuthorizationBindingV1`を必須にし、bindingのintent hash／risk／Operation／Project Scope、typed Authorization／ApprovalまたはPredelegation Use、Algorithm bindingを照合する。R2はApprovalまたはPredelegation Useのexact一方、R3～R5はApprovalだけを許可する。R0 queryとauthoritative stateを変えないR1はbinding Field自体をcanonical omissionする。将来のR1 Task control等は当該familyの別typed control authorizationを完全登録し、本型へ暗黙昇格しない。binding確定後、全R2～R5 mutation inputについてbinding内`OperationRequestAlgorithmBindingV1`からexact `OperationRequestHashAlgorithmV1`を解決し、`canonical_input_without_request_hash = MCD canonical encode(選択input schemaに存在する全Fieldからrequest_hash Fieldだけを除外)`、`request_hash = SHA-256(ASCII "MIRAKAN_OPERATION_REQUEST_V1" || uint32_be(len(canonical_input_without_request_hash)) || canonical_input_without_request_hash)`で計算する。したがってfinal requestはintent hash、exact authority evidence、Closure-bound Algorithm refを含むが、evidenceはfinal request hashをsubjectにせず固定点を作らない。ambient Registry、実装default、Provider hintからAlgorithmを選ばない。式のbytesは従来定義から変更せず、named recordがその唯一の機械解釈を与える。

Project-bound inputはOperation ref、input type ref、exact Project ref、policy refs、intent hash、named binding、Contract set root、全presence discriminatorを含む。projectless inputはProject refをschemaへ追加せず、後続Ownerが登録するexact workspace／catalog／resource ref等、そのinput schemaに実在する全Fieldを含む。sentinel／null Projectを捏造しない。input Type ref／schema discriminatorが異なるOperationは別canonical bytesになり、Project有無の差だけでhash式をversion-upしない。Domain文書は匿名sibling shapeや別式を定義しない。

`MIRAKAN_OPERATION_REQUEST_V1`、`MIRAKAN_OPERATION_INTENT_V1`、`MutationAuthorizationBindingV1`、`OperationRequestHashAlgorithmV1`は2026-07-29時点でEngine、Project、Tool catalog、Receipt storeへ一度もActivation／materializationされていないinitial design contractである。したがって本節の非循環DAGを最初のcanonical V1として直接定義し、旧draft domain、retained Receipt、offline migration record、compatibility reader、aliasを作らない。過去reviewで使った循環shapeとV2表記はimmutable ADR／review transcriptだけに履歴として残し、current Editor／MCP／CLI／Save／Replay／idempotency storeの入力として受理しない。このDAGとnamed bindingは設計制約であり、本書は実装Taskまたは作業順序を定義しない。

```text
PreparedRuntimeEntryMutationReceiptPayloadV1
  publication_binding: exact PreparedReceiptPublicationBindingV1
  operation_ref: McdContractRefV1(kind=operation)
  operation_intent_hash
  request_hash
  idempotency_key
  before_project_ref: exact {project_id, project_revision, document_set_hash}
  after_project_ref: exact {project_id, project_revision, document_set_hash}
  affected_documents[1]: AffectedDocumentMutationV1
  stable_id_allocation_mappings[0..1]
  preview_receipt_payload_ref/hash
  validation_receipt_payload_ref/hash
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1
  diagnostics[0..64]: DiagnosticCodeRefV1
  prepared_payload_hash: SHA-256

RuntimeEntryMutationReceiptV1
  published_receipt:
    exact PublishedDomainReceiptV1 whose
    prepared_domain_receipt_payload_ref/hash resolves
    PreparedRuntimeEntryMutationReceiptPayloadV1

AffectedDocumentMutationV1
  change_kind: created | updated
  document_kind: world
    before_world_document_ref: exact DocumentRef including content_hash | omitted
    after_world_document_ref: exact DocumentRef including content_hash
  | runtime_entry
    before_runtime_entry_document_ref: exact DocumentRef including content_hash | omitted
    before_runtime_entry_semantic_hash: RuntimeEntryPointSemanticHashV1 | omitted
    after_runtime_entry_document_ref: exact DocumentRef including content_hash
    after_runtime_entry_semantic_hash: RuntimeEntryPointSemanticHashV1
  | runtime_target_selector
    before_selector_document_ref: exact DocumentRef including content_hash | omitted
    before_selector_hash: RuntimeTargetSelectorHashV1 | omitted
    after_selector_document_ref: exact DocumentRef including content_hash
    after_selector_hash: RuntimeTargetSelectorHashV1
  | runtime_entry_activation_policy
    before_activation_policy_document_ref: exact DocumentRef including content_hash | omitted
    before_activation_policy_hash: RuntimeEntryActivationPolicyHashV1 | omitted
    after_activation_policy_document_ref: exact DocumentRef including content_hash
    after_activation_policy_hash: RuntimeEntryActivationPolicyHashV1
```

discriminator外branch Fieldを禁止する。`created`は選択branchの全before Fieldをcanonical omissionし、`updated`は選択branchの全before Fieldを必須にしてbefore／after Stable IDを一致させる。Worldへ普遍的なpayload semantic hashが存在すると仮定せず、Document content hashだけをexact ref内で記録する。entry、selector、activation policyは各Ownerが定義する別semantic hash型を使用し、generic `payload_semantic_hash`へ混同しない。

target六create／update Operation candidateは対象exact一件を持つ。createだけが発行したStable IDのallocation mappingをexact一件、updateはexact 0件を持ち、legacy migration plan、missing／duplicate／extra document kind、null、zero hashを拒否する。`prepared_payload_hash`はASCII `MIRAKAN_PREPARED_RUNTIME_ENTRY_MUTATION_RECEIPT_PAYLOAD_V1`とself-excluding payloadから計算する。唯一のsigned subjectは共通`PublishedDomainReceiptPayloadV1`の完成JCS bytesであり、Domain固有Subject／署名wrapperを作らない。公開手順は本節共通正本の`private Marker read-back → owner_typed_state_commitのsecret-free PublicCommitClosure candidate → signed Receipt → PublicCommitClosure＋PublicPublicationMarkerV1＋after Projectのatomic public CAS`をexact reuseし、別のDomain手順を定義しない。private Marker、prepared payload、Closure candidate storeだけを公開authorityにせず、同じ`idempotency_key`＋`request_hash`のretryは同じResult／Closure／signed Receipt／Public Markerを返し、同じkeyの別request hashは`MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE`で一切変更せず拒否する。

Operation Registry compilerは六Operation ID、12 predicate policy ID、一rate-limit policy ID、三predicate Type、二Prepared auxiliary payload Type、一Mutation authorization Type、全domain input／output／receipt ref、Service一record、SideEffect enum、Diagnostic ref、Validator closureをexact Contract set／各Registryへ解決する。pre／postcondition refがmissing、`kind!=policy`、version／Contract set hash stale、policyが`evaluation_mode!=pure`、side effect非空、IO／result Type不一致、Prepared auxiliary／Mutation authorization Typeまたはschema不一致、Service allowlist不一致、rate-limit payload不一致、Diagnostic四Field不一致、またはerrors／reachable set不一致ならRegistry全体を拒否する。fixtureは六Operationのmeta-schema compileとProject ownerとのset equalityに加え、各参照のwrong-kind、missing、stale version、stale Contract set／content hash、impure policyを一原因ずつ拒否する。

<a id="812-conditional-legacy-migration-evidence-gate"></a>

### 8.1.2 Post-release migration admission rule

Repositoryにmaterialized Project／Schema／reader／writer／Receiptが存在しないinitial designでは、legacy migration Operation／Inventory、offline migrator Service／Capability／Isolation Profile、alias、compatibility shimを登録しない。Root Scene、Runtime Scope、Project Scale Envelope、Physics intent roleはcurrent initial V1 shapeを直接使用し、過去draftの値または名前をcurrent inputとして受理しない。

最初の公開／materialized release後に実在consumerのmigrationが必要になった場合だけ、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)のconsumer inventory、source bytes、reader／writer、rollback、Evidenceを持つ新しいOwner ChangeSetとして設計する。将来の可能性だけを理由にOperation ID、Type、Policy、Validator、Diagnostic、Fixture、SignerまたはServiceを予約しない。

### 8.2 Target Installed Product Operation composition candidate

§8.1のGeneric Engine Core bootstrap六件とWorld一件、Shooter Genre Pack三件のexact十件は、完全closureを同一transactionでmaterializeする場合のtarget Installed Product composition candidateである。現RepositoryのMCD、Contract Set、Owner Manifest、Service allowlist、Policy、Validator、Diagnostic、Receipt、surface projection、Activation Evidenceはすべて未materializeで、current四Operation集合はexact `[]`である。§8.1.2で禁止したlegacy migration、§§20～21のplanning candidate、非活性Scenario vocabulary、§21.2のexample／pending／rejected ID、legacy aliasはtarget十件にも含めない。

target `InstalledProductOperationCompositionV1` candidateは次のexact三Origin contribution／十entryである。表は§8.1のliteral hash短記を使い、将来のmaterialization時に解決先`OwnerOperationContributionV1`および同じFoundation Closureのselected-active Owner identityとbyte equalityにする。

| Origin contribution | owner_ref | production_owner_layer | composition_role | exact target Operation LocalRef集合 | exact target Policy LocalRef数 |
|---|---|---|---|---|---:|
| `{contribution.owner.core.project_state.runtime_entry_bootstrap,1,project_state_contribution_hash_v1}` | `{owner.core.project_state,1,project_state_owner_hash_v1}` | `core` | `generic_core_baseline` | `{operation.project.runtime_entry.create,1}; {operation.project.runtime_entry.update,1}; {operation.project.runtime_entry_activation_policy.create,1}; {operation.project.runtime_entry_activation_policy.update,1}; {operation.project.runtime_target_selector.create,1}; {operation.project.runtime_target_selector.update,1}` | 13 |
| `{contribution.owner.core.world.generated_stable_id_allocation,1,world_contribution_hash_v1}` | `{owner.core.world,1,world_owner_hash_v1}` | `core` | `installed_core_extension` | `{operation.world.allocate_generated_stable_ids,1}` | 3 |
| `{contribution.owner.genre.shooter.target_provider_binding,1,shooter_contribution_hash_v1}` | `{owner.genre.shooter,1,shooter_owner_hash_v1}` | `genre_pack` | `genre_pack` | `{operation.shooter.target_provider_binding.create,1}; {operation.shooter.target_provider_binding.select,1}; {operation.shooter.target_provider_binding.update,1}` | 7 |

Project State Originのauthority sourceは`architecture_document.document_id=mirakan.arch.project-state`、Worldは`mirakan.arch.rendering-world`、Shooterは`mirakan.arch.pack-shooter`である。各Origin recordの`operation_local_refs[]`は表のexact集合、owner／layer／authority sourceは同じOwner Identity Registry rowとbyte equalityにする。target composition candidateを将来の同一activation transactionで次のclosed recordへmaterializeし、十entryの各行は上表のowner／layer／role／Origin refを省略せず持つ。

次のrecordは本書前段のcanonical `InstalledProductOperationCompositionV1` schemaへtarget candidate値を代入した非materialized具体例であり、current instanceまたはschemaの再定義ではない。

```text
InstalledProductOperationCompositionV1
  artifact_role = cross_cutting_control_plane
  composition_id = operation_composition.product.current
  composition_version = 1
  generic_core_baseline_ref = {
    operation_baseline.generic_engine_core.bootstrap,
    1,
    generic_core_baseline_hash_v1
  }
  entry_count = 10
  entries = [
    {operation_local_ref={operation.project.runtime_entry.create,1},
     owner_ref={owner.core.project_state,1,project_state_owner_hash_v1},
     production_owner_layer=core,
     composition_role=generic_core_baseline,
     contribution_origin_ref={contribution.owner.core.project_state.runtime_entry_bootstrap,1,project_state_contribution_hash_v1}}
    {operation_local_ref={operation.project.runtime_entry.update,1},
     owner_ref={owner.core.project_state,1,project_state_owner_hash_v1},
     production_owner_layer=core,
     composition_role=generic_core_baseline,
     contribution_origin_ref={contribution.owner.core.project_state.runtime_entry_bootstrap,1,project_state_contribution_hash_v1}}
    {operation_local_ref={operation.project.runtime_entry_activation_policy.create,1},
     owner_ref={owner.core.project_state,1,project_state_owner_hash_v1},
     production_owner_layer=core,
     composition_role=generic_core_baseline,
     contribution_origin_ref={contribution.owner.core.project_state.runtime_entry_bootstrap,1,project_state_contribution_hash_v1}}
    {operation_local_ref={operation.project.runtime_entry_activation_policy.update,1},
     owner_ref={owner.core.project_state,1,project_state_owner_hash_v1},
     production_owner_layer=core,
     composition_role=generic_core_baseline,
     contribution_origin_ref={contribution.owner.core.project_state.runtime_entry_bootstrap,1,project_state_contribution_hash_v1}}
    {operation_local_ref={operation.project.runtime_target_selector.create,1},
     owner_ref={owner.core.project_state,1,project_state_owner_hash_v1},
     production_owner_layer=core,
     composition_role=generic_core_baseline,
     contribution_origin_ref={contribution.owner.core.project_state.runtime_entry_bootstrap,1,project_state_contribution_hash_v1}}
    {operation_local_ref={operation.project.runtime_target_selector.update,1},
     owner_ref={owner.core.project_state,1,project_state_owner_hash_v1},
     production_owner_layer=core,
     composition_role=generic_core_baseline,
     contribution_origin_ref={contribution.owner.core.project_state.runtime_entry_bootstrap,1,project_state_contribution_hash_v1}}
    {operation_local_ref={operation.shooter.target_provider_binding.create,1},
     owner_ref={owner.genre.shooter,1,shooter_owner_hash_v1},
     production_owner_layer=genre_pack,
     composition_role=genre_pack,
     contribution_origin_ref={contribution.owner.genre.shooter.target_provider_binding,1,shooter_contribution_hash_v1}}
    {operation_local_ref={operation.shooter.target_provider_binding.select,1},
     owner_ref={owner.genre.shooter,1,shooter_owner_hash_v1},
     production_owner_layer=genre_pack,
     composition_role=genre_pack,
     contribution_origin_ref={contribution.owner.genre.shooter.target_provider_binding,1,shooter_contribution_hash_v1}}
    {operation_local_ref={operation.shooter.target_provider_binding.update,1},
     owner_ref={owner.genre.shooter,1,shooter_owner_hash_v1},
     production_owner_layer=genre_pack,
     composition_role=genre_pack,
     contribution_origin_ref={contribution.owner.genre.shooter.target_provider_binding,1,shooter_contribution_hash_v1}}
    {operation_local_ref={operation.world.allocate_generated_stable_ids,1},
     owner_ref={owner.core.world,1,world_owner_hash_v1},
     production_owner_layer=core,
     composition_role=installed_core_extension,
     contribution_origin_ref={contribution.owner.core.world.generated_stable_id_allocation,1,world_contribution_hash_v1}}
  ]
  composition_content_hash =
    installed_product_composition_hash_v1
```

`generic_core_baseline_ref`は§8.1のexact六entry baselineへ解決し、その六`OperationCompositionEntryV1`はtarget composition candidate内の同一bytes entryとset equalityにする。candidateの残り四entryはWorld／Shooter Originのunionとexact equalityにし、将来の同一activation transactionで十entry全体をmaterialized active MCD Operation LocalRef集合、全Owner Manifest contribution union、Authoring Command Gatewayのallowlistへ同型setで比較する。Baseline refのmissing／stale、baseline entryの上書き、同じOperationの別Origin、owner／layer／role差、Origin一件missing／extra、件数だけ十件の別IDをcompile failureにする。

各Ownerについて、`Owner Manifest Operation LocalRefs = active MCD Operation LocalRefs = Trusted Service allowlist owner contribution`をID／versionで比較する。さらに各Operationの`preconditions[] ∪ postconditions[] ∪ {rate_limit_policy}`はOwner文書が列挙するPolicy LocalRef subset、`errors[]`はValidator closureのreachable Diagnostic LocalRef union、`receipt_type`はOwner Manifest Receipt Type、`authority`はallowlistを持つService local recordとexact equalityでなければならない。Worldの`provider_exposure=trusted_internal`ではProvider／MCP projection集合を空、他三件の`provider_exposure=mcp_proposal`では完全登録済みOperation ref集合と生成projection集合をexact equalityにする。prefix、説明文、family名から集合を展開しない。

target `TrustedServiceRegistryV1` candidateの一`TrustedServiceLocalRecordV1`は、将来の同じContract set transactionで`InstalledProductOperationCompositionV1.entries[]`をmergeした次のexact LocalRef配列を持つ。current Registry／Service recordは未materializeである。配列はOperation IDのASCII byte昇順、同IDではversion昇順であり、別順序、件数だけ一致する別ID、prefix展開を拒否する。

```text
service.authoring_command_gateway@1
  allowed_operation_local_refs[10] = [
    {operation.project.runtime_entry.create,1}
    {operation.project.runtime_entry.update,1}
    {operation.project.runtime_entry_activation_policy.create,1}
    {operation.project.runtime_entry_activation_policy.update,1}
    {operation.project.runtime_target_selector.create,1}
    {operation.project.runtime_target_selector.update,1}
    {operation.shooter.target_provider_binding.create,1}
    {operation.shooter.target_provider_binding.select,1}
    {operation.shooter.target_provider_binding.update,1}
    {operation.world.allocate_generated_stable_ids,1}
  ]
  service_local_content_hash =
    SHA-256(MIRAKAN_TRUSTED_SERVICE_LOCAL_RECORD_V1,
      self-excluding canonical current merged TrustedServiceLocalRecordV1)
```

`service_local_content_hash`は上記配列と既存のexecutable identity、Capability、isolation profileを含む完成local recordから再計算する。Service member hashと`TrustedServiceRegistryV1.registry_content_hash`を別domainで生成し、Contract set root確定後にだけroot付きOperation／Capability／profile refを持つ一つの外部`TrustedServiceRecordV1.service_content_hash`と`TrustedServiceRefV1`を生成する。local hash、member hash、Registry hash、external hashの等値を要求せず、後段の値を前段preimageへ戻さない。bootstrap hashの再利用、Domain contributionだけの別record、10件の一件欠落／追加／入替えをcompile failureにする。

`allowed_operation_local_refs[10]`はtarget contract-active allowlist candidateであって、それだけではcurrent stateまたはdispatch authorityにならない。current operational集合はexact `[]`である。将来の承認済みmaterialization後に許可するoperational stage集合は、初期`[]`、`activation.foundation.operation_domain_receipt_pipeline.v1`後のProject State Origin exact六件、`activation.installed_product.operation_composition_extensions.v1`後のProject State＋World＋Shooter三Origin exact十件だけである。Runtime projectorはその時点のcurrent Signer Policyで同Serviceへ割り当てられたOperation集合をOrigin別に逆引きし、選択した各`OwnerOperationContributionV1.operation_local_refs[]`を全件含むこと、そのunionがSigner集合とset equality、full allowlistのsubset、同一Candidateのfresh Qualification集合とset equalityであることを毎dispatch検証する。`[] -> 6 -> 10`以外の集合、Origin内部分集合、allowlistだけのAuthority合成を拒否する。extension時はbaseline六もextension四と同じWave 3 Candidateで再Qualificationするが、baseline Operation／Origin／Signer／Role／assignment／Key bytesは変更せず、World／Shooter四rowだけを追加する。

Shooter Genre Packを初回materialization後に取り外す場合は、承認済みdefinition migrationで`contribution.owner.genre.shooter.target_provider_binding`のexact三entryだけをInstalled Product composition、MCD current集合、Owner Manifest contribution union、Service allowlist、Provider／MCP projection、Signer Policy destination、Trust／Foundation closureから同時に除去し、composition／Service／Contract set／Closureのversionとhashを再発行する。このtransactionは`GenericCoreOperationBaselineV1`、Project State Origin、baseline六entryおよびその`baseline_content_hash`を変更してはならない。Shooter removalをCore baseline migrationとして扱うこと、Shooter三entryを`core`または`generic_core_baseline`へ付替えること、旧三Operationをaliasとしてcurrentに残すことを拒否する。現Repositoryではこの前提となるmaterialized composition自体がなく、current四Operation集合はexact `[]`である。

四OperationのContract set closureは、Operation 4件、参照Policy 10件、下表のdirect reachable Type exact 19件、World Requirement 1件、Service owner contribution、Validator local record、Operation Validator Closure local record、reachable Diagnostic exact 23件をすべて`ContractSetSnapshotV1.members[]`へ実recordとして含める。Type 19件は§8.1と共有するcommon六件を一度だけ数え、Owner固有13件とのdisjoint unionにする。各Typeはcomplete MCD local record、self-excluding local hash、Type member hashを持ち、nested schema refも同じSnapshot内のexact LocalRefへ再帰解決する。外部root付きType ref、bare ID、同名別versionをmember preimageへ入れない。

| Type owner | exact件数 | exact Type LocalRef ID集合（全version 1、全version 1） |
|---|---:|---|
| Common publication／predicate／authorization | 6 | `type.operation.mutation_authorization_binding; type.operation.precondition_evaluation_input; type.operation.postcondition_evaluation_input; type.operation.predicate_result; type.operation.prepared_preview_receipt_payload; type.operation.prepared_validation_receipt_payload` |
| World | 4 | `type.world.stable_id_allocation_input; type.world.stable_id_allocation_result; type.world.stable_id_allocation_receipt; type.world.prepared_stable_id_allocation_receipt_payload` |
| Shooter | 9 | `type.genre.shooter.target_provider_binding_create_input; type.genre.shooter.target_provider_binding_update_input; type.genre.shooter.target_provider_binding_select_input; type.genre.shooter.target_provider_binding_mutation_result; type.genre.shooter.target_provider_binding_selection_result; type.genre.shooter.target_provider_binding_mutation_receipt; type.genre.shooter.target_provider_binding_selection_receipt; type.genre.shooter.prepared_target_provider_binding_mutation_receipt_payload; type.genre.shooter.prepared_target_provider_binding_selection_receipt_payload` |

各Ownerのprepared Domain payload Type recordは対応named `Prepared*ReceiptPayloadV1`の全Fieldをexact投影し、その`publication_binding`を通じてcommon二Prepared auxiliary payloadと同じOperation／intent／request／idempotency／before／after／Contextへbindする。最終Receipt Typeとprepared payload Typeは別LocalRefであり、相互代用しない。19件の一件missing／extra／duplicate、common TypeのOwnerごとの複写、prepared↔final Receipt Type swap、wrong version／kind／schema hashをContract set compile failureにする。

Diagnostic 23件はcommon 8件とowner固有15件のdisjoint unionで、owner固有subsetを次に固定する。

| Owner | exact固有件数 | exact Diagnostic ID集合 |
|---|---:|---|
| World | 6 | `diagnostic.world.generated_candidate_invalid; diagnostic.world.stable_id_project_binding_mismatch; diagnostic.world.stable_id_allocation_count_mismatch; diagnostic.world.stable_id_manifest_invalid; diagnostic.world.stable_id_receipt_signing_failed; diagnostic.world.stable_id_publication_conflict` |
| Shooter | 9 | `diagnostic.genre.shooter.target_provider.identity_mismatch; diagnostic.genre.shooter.target_provider.owner_mismatch; diagnostic.genre.shooter.target_provider.template_mismatch; diagnostic.genre.shooter.target_provider.type_mismatch; diagnostic.genre.shooter.target_provider.hash_mismatch; diagnostic.genre.shooter.target_provider.target_unsupported; diagnostic.genre.shooter.target_provider.fixture_in_production; diagnostic.genre.shooter.target_provider.registry_invalid; diagnostic.genre.shooter.target_provider.receipt_binding_mismatch` |

common exact 8件は`diagnostic.conflict.revision_mismatch; diagnostic.authorization.denied; diagnostic.approval.required; diagnostic.authoring.lock_conflict; diagnostic.mcd.operation_predicate_invalid; diagnostic.operation.timeout; diagnostic.operation.rate_limit_exceeded; diagnostic.operation.idempotency_key_reuse`である。各Owner Manifest Diagnostic subset、各Operation `errors[]`、Validator reachable union、Snapshot Diagnostic local memberのlogical集合をset equalityにし、Shooterのcomposition／profile等、active三Operationの17-ref closureに到達しない別Diagnosticを23件へ混入させない。

fixtureは型ごとに実在するFieldだけを変異させる。10 Policyはcommon envelopeまたはpredicate／rate payload、World Requirementはcommon envelopeまたはRequirement payload、23 Diagnosticの各recordは`owner_local_ref`、`code`、`severity`、`category`、`message_key`、`requirement_local_refs`、`retryability`のいずれか一Fieldだけを変えるcaseを最低一件ずつ生成し、該当local hash、member hash、Contract set root、外部record hashが変わり、旧Manifest／Service／Policy／Validator／Diagnostic／Receipt external refとのset equalityが失敗することを検証する。Owner ref mutationは別の有効Owner identityへ変更した場合も該当Owner Manifest subsetとのequalityを失敗させる。23件の一件missing／extra／重複／cross-owner substitution、bare ID、name-only active record、root確定前のexternal ref、member byte変更後も同じrootを受理する実装を拒否する。

target Installed Product composition candidateの全体summaryは次のexact値である。Operation `10 = Generic Core baseline 6 + installed Core extension 1 + Genre Pack 3`、direct reachable Type `27 = bootstrap 14 + Domain 19 - shared common 6`、target Policy `23 = bootstrap 13 + Domain 10`、target-Operation reachable Diagnostic `32 = bootstrap 17 + Domain 23 - shared common 8`、target-Operation reachable Validator `27 = shared common 6 + Project固有12 + World固有3 + Shooter固有6`、Operation Validator Closure `10`、Trusted Service `1`である。これらはtarget closureの受入値で、current materialized件数ではない。現Repositoryのcurrent MCD／Registry／Contract Set／Manifest／Service／Policy／Validator／Diagnostic／Receipt／Operation件数はすべて0である。将来のcompiler fixtureは各target集合の一件missing／extra、共有commonの二重計上、legacy migration artifactの混入、World publication Validator欠落を一原因ずつ拒否する。

### 8.3 Qualification依存のclosed分類

current文書、schema、plan、specに現れる`qualification_receipt*`／Qualification Receipt参照は、compilerが次の三classのexact一つへ分類する。未分類、複数class、名前だけからの推測を拒否する。

| class | 許可するedge | current exact対象 |
|---|---|---|
| `receipt_free_base` | Receipt／wrapper／Binding／Activation projection／Fixture refへのedgeは0件。self hash、Registry hash、Contract set rootを先に確定する | Game System Spec／auxiliary、current Runtime Scope Catalog／Owner／Dependency、Performance Domain／Intent／Envelope／Dimension、Input Binding Record／Contribution、Physics Role、Navigation Provider（`UsageTaggedImplementationSystemBaseRefV1`だけ）／Binding、Procedural World／Blockout／World Plan／Bundle／generated Delta、Pack Recipe／Profile／Manifest、Project Shader Module／Target Support、Material／Post Process Node／Profile、VFX Extension Manifest |
| `safe_downstream_projection` | 完成base refをsubjectにするQualification subject→signed Receipt→Bindingの後段、またはそのBindingを消費するSelection／Manifest／Compile／Save／Replay／Derived resolution。Receipt subjectから当該projectionへ到達するedgeは0件 | 各current owner-typed Qualification subject／Receipt／Activation Binding、Game System／Fixture System Activation Binding、System base refにそのActivation Bindingを加えた`UsageTaggedImplementationSystemRefV1`、同refを依存として署名するMotion Executor Provider subject、Shooter Target Provider Binding、current Runtime Scope／Performance／Physics／Input activation catalog、Pack Recipe／World／Project Shader／Material Project Node／Post Process／VFX activation projection、Operation Manifest、Project `TargetReadinessV1(state=qualified)`、Renderer／Lighting／LOD resolved plan・evidence、Governance Attestation／Product gate／Toolchain qualified capacity |
| `historical_or_planning` | current Registry、Catalog、Manifest、Service allowlist、Runtime Packageへprojectionされないread-only記録だけ | retained ADRの旧shape、Product Planのplanning-only Future／Local multiplayer profile、§§20–21の未Activation candidate |

`safe_downstream_projection`はDAG stageごとのclosed unionである。Qualification Subject stageは共通`QualificationSubjectValidatorV1`がbase ref／subject ref、owner、Target集合、dependency集合を検証し、signed ReceiptまたはBinding refを入力にしない。signed Receipt stageは`QualificationReceiptValidatorV1`が完成Subject hash、wrapper purpose／subject／署名／freshness／revocationを検証し、Bindingを入力にしない。`TargetReadinessV1`、Technical Qualification、Attestation、Product gate等のdirect Evidence consumerは`DirectEvidenceProjectionValidatorV1`が先に固定したartifact／input closureとexact typed Receipt refのsubject／wrapperを照合し、存在しないDomain Activation Bindingを要求しない。この三stageへ後段Fieldを追加して循環させることをcompile failureにする。

次のvalidation-only normalized viewへtotalに写像する対象は、上表のconcrete Activation／Qualification Bindingと、そのBinding refを直接保持または解決する外部consumer projectionだけである。Subject／Receipt stageとdirect Evidence consumerはこのviewの対象外で、上記stage別validatorを使う。viewは永続MCD、base、Receipt、Binding、ProjectionのFieldではなく、hash preimageや新しい参照edgeを作らない。

```text
QualificationActivationBindingEnvelopeV1
  consumer_base_ref_canonical_hash: SHA-256
  binding_base_ref_canonical_hash: SHA-256
  subject_base_ref_canonical_hash: SHA-256
  receipt_subject_base_ref_canonical_hashes[1..256]: SHA-256
  signed_receipt_ref_set_hash: SHA-256
  owner_ref_canonical_hash: SHA-256
  subject_owner_ref_canonical_hash: SHA-256
  target_ref_set_hash: SHA-256
  receipt_target_ref_set_hash: SHA-256
  dependency_ref_set_hash: SHA-256
  receipt_dependency_ref_set_hash: SHA-256
  binding_generation: {binding_id, binding_version, binding_content_hash}
```

各hashは解決済みDomain typed valueまたはstrict sort／unique済みtyped集合のcount／length-framed canonical bytesから導出し、該当集合が空のDomainもcanonical empty-set hashを必須にする。共通`QualificationActivationBindingValidatorV1`はconsumer／Binding／Subject／全Receipt subjectのbase hashをbyte equality、owner hashをbyte equality、target／dependency set hashをexact equality、Receipt ref集合をpass／fresh／non-revoked／strict sort／unique、generation tupleを解決済みBinding refとexact equalityにする。BindingまたはBinding consumerのDomain固有schemaにconsumer base、owner、target、dependencyのいずれかを導出するFieldがなく、そのDomain契約でもcanonical emptyと宣言されていない場合はadapter未定義としてcompile failureにする。このviewのhash同士をDomain base、Subject、Receiptへ保存しない。

同じBinding validator fixture matrixを上表の全concrete BindingとBinding-consuming外部consumer projectionへ適用する。fixtureはstale base revision／content hash、別の有効base／owner／target／dependency／Bindingへのsubstitution、Receipt subject一件だけの差替え、Receipt missing／extra／duplicate／順序違反、target／dependency missing／extra、generation ID／version／hash不一致を各一原因で変異し、全件をrejectしなければならない。Subject stageはbase／owner／target／dependencyのsubstitution、Receipt stageはsubject／purpose／署名／freshness／revocation、direct Evidence stageはartifact closure／typed Receipt subject／wrapper refのsubstitutionを各専用matrixで拒否する。Domain契約がより強いpairing、cardinality、set-equality規則を持つ場合はそのfixtureを追加し、該当stageの共通matrixを省略しない。

`safe_downstream_projection`に属する、Field名として`qualification_receipt*`を現在保持するexact schemaは、`InferenceDeploymentProfileV1`、`SystemQualificationReceiptV1`、`SystemTechnicalAttestationV1`、`CodeOwnerAssignmentV1`、`GameSystemActivationBindingV1`、`FixtureImplementationSystemActivationBindingV1`、`SystemBundleChangeSetV1`、`TargetReadinessV1`とqualified Runtime Package Target closure、`PerformanceQualificationBindingV1`のmigration以外のsubject instance、`MotionIntentBindingQualificationBindingV1`、`MotionExecutorProviderActivationBindingV1`、`PhysicsIntentRoleActivationBindingV1`、ephemeral `PhysicsIntentResolutionV1`、`PostProcessActivationBindingV1`、`ProjectShaderActivationBindingV1`、`ResolvedShadowPlanV1`、`VfxExtensionActivationBindingV1`、`WorldSourceActivationBindingV1`、`SemanticActionBindingQualificationBindingV1`、`PackRecipeActivationBindingV1`である。各Domainの同名concrete instanceはこのschemaのinstanceであり、別classを作らない。これ以外のcurrent architecture schemaへ同名Fieldを追加する変更は、この分類表を同じContract set transactionでversion-upしない限りcompile failureとする。

`historical_or_planning`のexact残存schemaはProduct Planの`LocalPlaySessionProfileV1`、retained ADRの`RuntimeWorldQualificationBindingV1`および§§20–21の未Activation work itemだけである。これらの文書内にある説明上のQualification／Receipt文字列も同classであり、current schema集合を増やさない。`docs/architecture`のproseに現れるField名以外のQualification／Receipt文字列は、上記safe schemaの検証規則、Receipt wrapper自身、または未Activation work itemを説明する参照であって、新しいedgeをmaterializeしない。

`receipt_free_base`のcurrent schema集合に`qualification_receipt_ref(s)`、`signed_receipt`、`MirakanSignedRecordV1`、Qualification／Activation Binding ref、fixture body refが一件でも存在すればcompile failureであり、current許容件数はexact 0である。compilerはField名検索だけでなく、各Qualification Receipt subjectから全hash／ref edgeを逆向きにwalkし、subjectがbindするbase self hash／Registry root／Contract set rootのpreimageへReceipt、wrapper、Binding、Activation projectionが到達しないことを検証する。許可する唯一の順序は`receipt-free base → base ref/root → Qualification subject → signed Receipt → root外Binding → downstream projection`である。

`safe_downstream_projection`はReceipt refをhashへ含められるが、そのprojection hashを同Receipt subject、base ref、base Registry、Contract set rootへ戻してはならない。Project StateのTechnical Qualification、Renderer／LightingのResolved plan、AI Verification／SecurityのAttestation、Product gate、Toolchain capacityは先に固定したartifact／input closureをsubjectにするEvidence projectionであり、projection自身をsubjectにしない。`historical_or_planning`はActivation時に自動昇格せず、将来current化するtransactionで同じDAGへ書き直す。fixtureは各classの誤分類、baseへのReceipt一Field追加、subjectへのdownstream hash追加、Bindingから別baseへのsubstitutionを一原因ずつrejectする。

### 12.1 `MirakanDiagnosticV1`

Engine、Contract compiler、Provider adapter、MCP、CLIは共通の`MirakanDiagnosticV1`を返す。

Diagnostic codeの正本は次のRegistry recordである。Operation、Validator、Result、Receiptは裸codeを保存せず、四FieldがRegistry recordとexact equalityの`DiagnosticCodeRefV1`だけを使う。

```text
DiagnosticCodeRecordV1
  diagnostic_id: diagnostic.<lower-token-path>
  code: MIRAKAN-<DOMAIN>-<CONDITION>
  diagnostic_version: uint32
  owner_ref:
    exact {owner_id, owner_revision, owner_content_hash}
  severity: info | warning | error | blocking
  category:
    schema | semantic | permission | conflict | build | test |
    performance | security | provider | infrastructure
  message_key
  requirement_refs[0..64]: McdContractRefV1(kind=requirement)
  retryability: never | after_input | after_change | transient
  diagnostic_content_hash: SHA-256

DiagnosticCodeRefV1
  diagnostic_id
  code
  diagnostic_version: uint32
  diagnostic_content_hash: SHA-256

DiagnosticCodeRegistryRefV1
  registry_id
  registry_version: uint32
  registry_content_hash: SHA-256

DiagnosticCodeRegistryV1
  registry_id: diagnostic_code.registry.active
  registry_version: uint32
  registry_content_hash: SHA-256
  records[1..65536]: DiagnosticCodeRecordV1
```

`diagnostic_id`と`code`は一対一であり、同じIDの別code、同じcodeの別ID、同じID／versionの別content hashを拒否する。`owner_ref`はroot前の`DiagnosticOwnerLocalRefV1`と三Fieldexact equalityであり、owner IDのprefixやManifest membershipから補完しない。recordは`diagnostic_content_hash`を除く全Field、Registryは`registry_content_hash`を除きASCII `MIRAKAN_DIAGNOSTIC_CODE_REGISTRY_V1`、Registry ID／version、record count、`diagnostic_id`のNFC UTF-8 byte順にstrict sortしたrecord canonical bytesを、各byte列の`uint32_be` length付きでhashする。duplicate、非canonical order、unknown version、owner mismatch、hash mismatchではRegistry全体をfail closedにする。`DiagnosticCodeRefV1`は四Fieldすべてを同一recordへ解決し、IDだけまたはcodeだけが一致するrefを受理しない。

| Field | 型／規則 |
|---|---|
| `code_ref` | exact `DiagnosticCodeRefV1`。instance内でID／code／versionを再宣言しない |
| `severity` | `code_ref`解決先recordとexact equalityの`info \| warning \| error \| blocking` |
| `category` | `code_ref`解決先recordとexact equalityのclosed category |
| `message_key` | `code_ref`解決先recordとexact equalityのLocalization key |
| `arguments` | primitive map。完成文だけを保存しない |
| `artifact_id`／`revision` | 対象 |
| `location` | JSON Pointerまたはnormalized source location |
| `target_stable_ids` | 実在確認済み対象ID。候補の場合は候補理由を`arguments`へ含める |
| `requirement_ids` | 1件以上。Infrastructureだけ例外 |
| `expected`／`actual` | redacted typed value |
| `remediation_ids` | 機械実行可能または人間向け修正案 |
| `retryability` | `code_ref`解決先recordとexact equalityの`never \| after_input \| after_change \| transient` |
| `cause_chain` | 子`DiagnosticCodeRefV1` array。裸ID／codeを許可しない |
| `trace_id` | Verification trace参照 |

AIへ返すErrorはこの構造を維持する。Provider向け説明文だけへ変換してcode、location、expected、actualを失わない。Source／static analysis結果はこの形式を正本とし、外部Tool連携用にSARIF 2.1.0へexportする。

`MirakanDiagnosticV1`は検証結果と修復入口の正本であり、Debug Event全般の代替ではない。Debugging規約の`DebugEventEnvelopeV1`は必要時に`code_ref`／`cause_chain`／`trace_id`を参照し、severity、location、expected／actualを別Schemaへ重複保存しない。反対に高頻度counter、span、frame marker、domain snapshotを`MirakanDiagnosticV1`として発行してはならない。

### 12.2 `RemediationV1`

`remediation_ids`は自由文ではなく、MCDの`RemediationV1`を参照する。

| Field | 規則 |
|---|---|
| `id`／`version` | `remediation.<domain>.<lower_snake_name>`、意味変更ごとにversion増加 |
| `applicable_codes` | exact `DiagnosticCodeRefV1`のclosed set |
| `required_queries` | exact query Operation ID＋version、field mask、最大件数 |
| `operation_template` | typed Command ID＋固定field／placeholder定義。任意JSON禁止 |
| `preconditions`／`postconditions` | Predicate ID array |
| `risk_class`／`required_approvals` | 元Taskより権限を弱めない |
| `retryable_categories` | `schema \| semantic \| conflict \| build \| test \| performance \| provider \| infrastructure`の許可subset |
| `forbidden_categories` | `permission`、`security`、lock／approval／revision driftを必須化 |
| `max_applications_per_task` | 1または2。0と3以上を禁止 |
| `human_message_key` | Localization key |

RemediationはDiagnosticの解決候補であり、適用権限ではない。Gatewayは現在revision、Envelope、Risk、Approval、preconditionを再検証する。該当しないtarget、未知placeholder、権限追加、Source直接writeを含むtemplateはContract compile errorとする。同じnormalized blocking Diagnostic集合へ同じRemediationを二回適用しても減少しないfixtureはinvalidとし、Providerへ公開しない。

## 20. AI向けDiscovery／Execution候補のplanning record（未Activation）

Activation後の受入条件として、AIへ巨大な全Schemaを一括送信せず、次の二段階planned Discovery semanticsを使う。

1. Capability検索段階がID、title、tag、Target、maturity、短いsummaryを返す。
2. Capability詳細読取段階が選択したCapabilityのType、Operation、Constraint、Budget、Exampleを返す。

Activation後のSearch結果はその時点のContract set hashを含む。AIが古いhashのCapabilityでProposalを送った場合、Gatewayはstaleとして拒否し、差分を返す。AIがSchemaにないFieldや完全登録済みでないOperationを使った場合、fuzzyに推測して補正せず、候補ID付きDiagnosticを返す。planned検索actionのbounded result受入値は、各表で別値を明示しない限り既定50件、最大200件、continuation付きとする。Activation前のcurrent Search／Read Operation集合は空であり、この挙動をcallableとみなさない。

以下の§§20～21.1の候補表に列挙する`operation.*` IDは実行契約ではなく、将来の語彙衝突を防ぐ予約候補である。MCD document、Owner Manifest、Contract set member、Trusted Service allowlist、Provider projection、MCP alias、CLI／Editor commandのいずれにも存在せず、全familyのcurrent集合は空、Capability stateは`not_activated`である。§8.1／§8.2のtarget-complete候補十件もcurrent MCDには存在せず、本planning ledgerのreserved 207件とは別classで保持する。legacy migration IDはplanning ledgerへ登録しない。

```text
PlannedOperationFamilyV1
  planning_record_id
  planning_record_version: 1
  family_id
  reserved_candidate_ids[1..64]
  reserved_candidate_count: exact array count
  capability_state: not_activated
  current_owner_manifest_operation_local_refs: []
  current_mcd_operation_local_refs: []
  current_trusted_service_allowlist_local_refs: []
  current_policy_local_refs: []
  current_validator_local_refs: []
  current_validator_closure_local_refs: []
  current_diagnostic_local_refs: []
  current_receipt_type_local_refs: []
  current_provider_projection_refs: []
  current_mcp_tool_refs: []
  generated_aliases: []
  legacy_aliases: []
  activation_work_item_id
  activation_mode: atomic_family_contract_set_transaction
  unavailable_error_code: MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED
  planning_record_hash: SHA-256
```

`PlannedOperationFamilyV1`はMCD kindではなく、`ContractSetSnapshotV1`、Tool catalog、Package、Save、Replayへ入れないclosed planning recordである。hashはASCII `MIRAKAN_PLANNED_OPERATION_FAMILY_V1`と自己hashを除く全Fieldのlength-framed canonical bytesから計算する。候補要求はGateway dispatch前に上記errorで拒否し、Project／Source／Registry／Taskを変更しない。familyの一部だけ、name-only record、aliasだけ、read-only候補だけを先行activateしない。

| planning_record_id | version | family_id | exact候補数 | atomic activation work item |
|---|---:|---|---:|---|
| `planning.operation_family.authoring_discovery` | 1 | `authoring_discovery` | 4 | `activation.authoring.discovery_operations.v1` |
| `planning.operation_family.build_device_play_debug_task` | 1 | `build_device_play_debug_task` | 14 | `activation.build_gateway.operation_pipeline.v1` |
| `planning.operation_family.game_system_discovery` | 1 | `game_system_discovery` | 4 | `activation.systems.discovery_operations.v1` |
| `planning.operation_family.world_discovery` | 1 | `world_discovery` | 6 | `activation.worlds.discovery_operations.v1` |
| `planning.operation_family.rendering_aa_discovery` | 1 | `rendering_aa_discovery` | 5 | `activation.rendering.aa.discovery_operations.v1` |
| `planning.operation_family.lighting_discovery` | 1 | `lighting_discovery` | 9 | `activation.lighting.discovery_operations.v1` |
| `planning.operation_family.post_process_discovery` | 1 | `post_process_discovery` | 9 | `activation.post_process.discovery_operations.v1` |
| `planning.operation_family.project_shader_discovery` | 1 | `project_shader_discovery` | 16 | `activation.shader.discovery_proposal_operations.v1` |
| `planning.operation_family.math_semantic_authoring` | 1 | `math_semantic_authoring` | 6 | `activation.math.semantic_authoring_operations.v1` |
| `planning.operation_family.camera_authoring` | 1 | `camera_authoring` | 11 | `activation.camera.authoring_operations.v1` |
| `planning.operation_family.material_authoring` | 1 | `material_authoring` | 15 | `activation.material.authoring_operations.v1` |
| `planning.operation_family.vfx_authoring` | 1 | `vfx_authoring` | 24 | `activation.vfx.authoring_operations.v1` |
| `planning.operation_family.environment_authoring` | 1 | `environment_authoring` | 26 | `activation.environment.authoring_operations.v1` |
| `planning.operation_family.lod_authoring` | 1 | `lod_authoring` | 2 | `activation.lod.authoring_operations.v1` |
| `planning.operation_family.input_binding_selection` | 1 | `input_binding_selection` | 1 | `activation.input.semantic_action_binding_selection.v1` |
| `planning.operation_family.navigation_binding_selection` | 1 | `navigation_binding_selection` | 1 | `activation.navigation.motion_intent_binding_selection.v1` |
| `planning.operation_family.physics_role_selection` | 1 | `physics_role_selection` | 1 | `activation.physics.intent_role_selection.v1` |
| `planning.operation_family.feature_authoring` | 1 | `feature_authoring` | 7 | `activation.feature.authoring_operations.v1` |
| `planning.operation_family.authoring_changeset_execution` | 1 | `authoring_changeset_execution` | 4 | `activation.authoring.changeset_execution.v1` |
| `planning.operation_family.native_game_module_source` | 1 | `native_game_module_source` | 5 | `activation.native_game_module.source_operations.v1` |
| `planning.operation_family.project_source_promotion` | 1 | `project_source_promotion` | 1 | `activation.project_source.promotion_operations.v1` |
| `planning.operation_family.build_candidate_test` | 1 | `build_candidate_test` | 6 | `activation.build.candidate_test_operations.v1` |
| `planning.operation_family.gameplay_definition_authoring` | 1 | `gameplay_definition_authoring` | 6 | `activation.gameplay_definition.authoring_operations.v1` |
| `planning.operation_family.asset_authoring` | 1 | `asset_authoring` | 10 | `activation.asset.authoring_operations.v1` |
| `planning.operation_family.game_intent_understanding` | 1 | `game_intent_understanding` | 8 | `activation.game_production.intent_understanding_operations.v1` |
| `planning.operation_family.game_experience_iteration` | 1 | `game_experience_iteration` | 3 | `activation.game_production.experience_iteration_operations.v1` |
| `planning.operation_family.game_production_read` | 1 | `game_production_read` | 3 | `activation.game_production.read_operations.v1` |

上表27行と、この後に文書順で現れる第一列見出しが`reserved candidate ID`の27表を同じindexでzipするclosed expansion ruleを正本とする。各`PlannedOperationFamilyV1` instanceは、ledger列から`planning_record_id`、`planning_record_version`、`family_id`、`reserved_candidate_count`、`activation_work_item_id`を、対応候補表の第一列から行順を保った`reserved_candidate_ids[]`をmaterializeする。残る全Fieldは上のschemaに書かれたliteral、すなわち`capability_state=not_activated`、十のcurrent集合と二つのalias集合すべて`[]`、`activation_mode=atomic_family_contract_set_transaction`、`unavailable_error_code=MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`をField省略なしでmaterializeし、その後にだけ`planning_record_hash`を計算する。これは既定値補完ではない。ledger／候補表が27対27でない、候補数不一致、候補ID重複、別familyへの同一ID混入、literal Fieldの省略／変更、empty集合の省略をplanning record compile failureにする。

各work itemは、そのfamilyの採用exact ID集合、各OperationのMCD共通Envelope全Field、named input／result／Receipt Type、authority Serviceとallowlist、Risk、side effect、idempotency、transaction、pure pre／post Policy、rate／timeout、closed Diagnostic、Validator／closure、Provider exposure、canonical signed Receipt、private-to-public recovery、positive／negative Qualification、Owner Manifest／MCD／Service／Provider／aliasのlike-for-like equalityを同じContract set transactionで完備した場合だけfamily全体をactivateできる。候補IDを削除または分割する場合もplanning recordをversion-upし、実在しないIDをlegacy aliasにしない。以下の表の「予定意味」はactivation後に採用可否を再審査する入力であり、現行動作を記述しない。

Authoring dataの同じDiscovery原則について、次の四つをplanning candidateとして予約する。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.authoring.search` | kind、tag、Component、name token、spatial boundからStableId候補とscore理由を返す |
| `operation.authoring.read` | StableId、field mask、expected revisionからbounded `SceneSliceV1`またはDocument projectionを返す |
| `operation.authoring.dependencies` | inbound／outbound、Requirement、Capability、Decision、lockのbounded closureを返す |
| `operation.authoring.diff` | base／target revisionとStableId scopeからsemantic diff、storage-only diff、continuationを返す |

activation work itemがこのexact四件を採用する場合、全Queryは`project_revision`、`contract_set_hash`、`authoring_index_revision`、`query_hash`、`omitted_ranges`、`continuation_cursor`を返す契約にし、別revisionへのfallback、表示index、曖昧な名前だけのtarget確定、任意JSON断片を禁止する。それまでは四IDのdispatch、追加read、aliasのmaterializeを行わない。

Package／Device／Play／Debug／Task系はBuild Gatewayを入口候補とする次の14 IDをplanning recordへ予約する。これらはcanonical MCDでも公開Operationでもなく、[Core architecture](../02-foundation/core-architecture.md#91-operationtaskv1)のplanned task／Receipt mappingはActivation後の意味候補に留まる。

| reserved candidate ID | 予定Risk／kind／実行Authority | 予定必須identity／hash | 予定Side effect | 予定Idempotency／cancel | 予定成功Receipt／結果 | 予定Failure Diagnostic |
|---|---|---|---|---|---|---|
| `operation.build.request_package` | R2 job／Build Gateway | Project revision、Candidate root、Target Profile、Contract／Toolchain lock、request hash、Authorization | StagingにTarget packageを生成。Commit／sign／installなし | request hash＋idempotency key。publish前までcooperative cancel | `PackageReceiptV1` | `diagnostic.operation.package-input-mismatch` |
| `operation.device.install` | R3 command／Device Bridge | Project revision、Candidate root、Target、Device identity＋generation、exact `PackageReceiptV1` ref／hash＋package artifact ref／hash、request hash、Authorization、明示consent、R3 Approval | 承認済みpackageを一Deviceへinstall | package hash＋Device generation＋idempotency key。Device transaction commit前だけcancel | `DeviceInstallReceiptV1` | `diagnostic.operation.device-install-binding-mismatch` |
| `operation.device.launch` | R1 command／Device Bridge | Project revision、Candidate root、Target、Device identity＋generation、exact `DeviceInstallReceiptV1` ref／hash、package artifact ref／hash、exact `RuntimeEntryDistributionPackageManifestV1` ref／hash、outer `RuntimeEntryPackageRefV1`、exact `RuntimeEntryLaunchSelectionV1` ref／完成record SHA、request hash、固有Authorization | Selectionが一意に束縛するinstall済みouter Runtime Entryのprocess起動 | Selection＋Device generation＋launch request hash＋idempotency key。process spawn前だけcancel | Selectionをread-backする`DeviceLaunchReceiptV1` | `diagnostic.operation.device-launch-binding-mismatch` |
| `operation.device.reset_data` | R3 command／Device Bridge | Project revision、Candidate root、Target、Device identity＋generation、exact `PackageReceiptV1` ref／hash＋package artifact ref／hash、request hash、Authorization、明示consent、R3 Approval | 対象ApplicationのDevice dataを消去 | package hash＋Device generation＋idempotency key。reset commit前だけcancel | `DeviceDataResetReceiptV1` | `diagnostic.operation.device-reset-consent-required` |
| `operation.play.run_smoke` | R1 job／Play Service | Project revision、Candidate root、Target、Device identity＋generation、exact Package／Install／Launch Receipt ref／hash、package artifact ref／hash、fixture ref／hash、request hash、固有Authorization | Candidateに対するbounded smoke sessionを実行 | input hash＋idempotency key。fixture boundaryでcooperative cancel | `SmokeRunReceiptV1` | `diagnostic.operation.smoke-input-mismatch` |
| `operation.debug.aggregate` | R0 query／Debug Query Service | Project revision、Candidate root、Target、Session、exact Build Receipt ref／hash、Store／Index generation、bounded selector hash、request hash、Authorization、remote Device identity＋generation | read-only aggregateを計算 | pure。cancelはquery中断のみ | `DebugAggregateReceiptV1` | `diagnostic.debug.aggregate-input-invalid` |
| `operation.debug.query` | R0 query／Debug Query Service | Project revision、Candidate root、Target、Session、Store／Index generation、remote Device identity＋generation、exact `DebugAggregateReceiptV1` ref／hash、bounded query hash、request hash、新Authorization | read-only record sliceを返す | pure。cancelはquery中断のみ | `DebugQueryReceiptV1` | `diagnostic.debug.query-input-invalid` |
| `operation.debug.read_causality` | R0 query／Debug Query Service | Project revision、Candidate root、Target、Session、Index generation、remote Device identity＋generation、exact `DebugQueryReceiptV1` ref／hash、root Evidence refs、bound hash、request hash、新Authorization | read-only causal subgraphを返す | pure。cancelはquery中断のみ | `DebugCausalityReceiptV1` | `diagnostic.debug.causality-input-invalid` |
| `operation.debug.read_replay_slice` | R0 query／Replay Service | Project revision、Candidate root、Target、Session、remote Device identity＋generation、exact Build Receipt、`DebugQueryReceiptV1`、`DebugCausalityReceiptV1`の各ref／hash、Replay closure／range hash、request hash、新Authorization | immutable Replay Sliceをmaterialize | request hash＋idempotency key。publish前までcooperative cancel | `ReplaySliceReceiptV1` | `diagnostic.debug.replay-slice-input-invalid` |
| `operation.debug.validate_finding` | R0 job／Debug Validation Service | Project revision、Candidate root、Target、Session、remote Device identity＋generation、exact Build Receipt、`DebugQueryReceiptV1`、`DebugCausalityReceiptV1`、`ReplaySliceReceiptV1`の各ref／hash、`DebugFindingV1` hash、Finding closure hash、request hash、新Authorization | append-only validation Evidenceを生成。Source／Project変更なし | finding＋closure hashでidempotent。validation step間でcancel | `DebugFindingValidationReceiptV1` | `diagnostic.debug.finding-evidence-invalid` |
| `operation.debug.support-bundle.generate` | R2 job／Debug Export Service | Project revision、Candidate root、Target、Session、exact Build Receipt ref／hash、source Debug Receipt ref／hash集合、component／redaction／policy hash、request hash、明示consent、Authorization、remote Device identity＋generation | redacted Support BundleをStagingへ生成。uploadなし | manifest input hash＋idempotency key。archive publish前までcooperative cancel | `SupportBundleReceiptV1` | `diagnostic.debug.support-bundle-manifest-mismatch` |
| `operation.task.status` | R0 query／Build Gateway Task Service | control invocation ID、target task／operation ID、target request hash、Project revision、Candidate root、control request hash、Authorization | なし。bounded task snapshotを返す | pure。cancel対象外 | `TaskStatusReceiptV1`＋`OperationTaskV1` snapshot | `diagnostic.operation.task-binding-mismatch` |
| `operation.task.read_receipt` | R0 query／Build Gateway Task Service | control invocation ID、terminal task／operation ID、target request hash、Project revision、Candidate root、exact terminal Receipt ref／hash、control request hash、Authorization | なし。immutable Receiptをread-only取得 | pure。cancel対象外 | `TaskReceiptReadReceiptV1`＋referenced exact Receipt | `diagnostic.operation.task-receipt-mismatch` |
| `operation.task.cancel` | R1 command／Build Gateway Task Service | control invocation ID、target task／operation ID、target request hash、Project revision、Candidate root、cancel request hash、original callerまたは委任済みAuthorization | cancellable taskを`cancel_requested`へ遷移 | task ID＋idempotency key。反復cancelは同じ結果 | `TaskCancellationReceiptV1` | `diagnostic.operation.task-not-cancellable` |

`diagnostic.operation.task-not-cancellable`は上表のplanned `operation.task.cancel`だけに予約した語彙であり、current `DiagnosticLocalRecordV1`、Service allowlist、Validator、Validator closure、Operation `errors[]`のいずれにも存在しない。したがって§8.2のcommon Diagnostic exact 8件およびreachable Diagnostic exact 23件へ加えない。Task control familyのatomic Activation transactionがDiagnostic四Field、Owner、Requirement、Validator、Operation、Receiptを同時にmaterializeするまで、このID単体またはDiagnostic recordだけを先行activateしない。

activation work itemがexact 14件を採用する場合、各成功Receiptは[Core architecture §9.1](../02-foundation/core-architecture.md#91-operationtaskv1)のplanned 14行mappingからexact `signed_record_purpose`、`OperationReceiptEnvelopeV1.operation_id`、型固有payload contract、完成Receipt型を一行で選ぶ完成`MirakanSignedRecordV1`とする。前段Receipt refとhash、purpose、payload contract、Project／Candidate／Target／Device identity＋generation、request hash、Authorizationの一つでも異なれば後段を開始しない。install／resetのconsent／Approval非継承やDevice generation driftも同じactivation transactionのfixtureで閉じる。現在はReceipt、Task、allowlist、remote sessionを一件もmaterializeしない。

将来このfamilyをactivateしても、正規Commit、Approval発行、Promotion、Activation、Signing、Releaseは8節の`trusted_internal`境界を代替しない。AIの実行権限とCaller Profileは[AI Security／Approval](../01-governance/ai-security-approval.md)が所有する。

LOD Discoveryは`lod_class`、semantic role、Target、Qualityで絞り込み、Intent、該当Domain Policy、fallback、選択metric、現在のqualification statusだけを返す。全DomainのLOD Schemaやruntime telemetryを常に一括送信しない。

Game System Discoveryは次の四IDをplanning candidateとして予約する。current MCP／製品Tool名、generated alias、legacy aliasの集合は空である。

| reserved candidate ID | 予定Authority | 予定意味 |
|---|---|---|
| `operation.systems.search` | R0 query | Role、Target、maturity、originでCatalog entryを検索 |
| `operation.systems.read` | R0 query | exact System Contract、constraint、budget、fixtureを取得 |
| `operation.systems.plan` | R1 proposal | `SystemImplementationPlanV1`を提案 |
| `operation.systems.validate_bundle` | R0 query／job | Staging `SystemBundleChangeSetV1`を検証 |

World Discoveryも次の六IDをplanning candidateとして予約する。alias、Input／Output Schema、Provider projectionはcurrent集合に存在しない。

| reserved candidate ID | 予定Authority | 予定意味 |
|---|---|---|
| `operation.worlds.search` | R0 query | kind、role、tag、Target、spatial boundからWorld／Scene／Space／Topology／owner-typed候補、StableId、score理由を返す |
| `operation.worlds.read` | R0 query | exact Stable ref、field mask、Viewport、Targetからbounded `WorldAuthoringContextV1`を返す |
| `operation.worlds.resolve_map_intent` | R0 query／R1 proposal | 6分類候補、Evidence、`resolved \| question_required \| rejected`を返す |
| `operation.worlds.plan_change` | R1 proposal | allowed `WorldSourceChangePrimitiveKindV1` discriminator、precondition、Budget、fixtureを持つ`WorldAuthoringPlanV1`を返す |
| `operation.worlds.validate_bundle` | R0 query／job | Staging BundleのSchema、semantic、reference、ownership、Topology、Budget Diagnosticを返す |
| `operation.worlds.preview_bundle` | R0 query／job | Source／Topology／Space／owner-typed content／Target別Derived差分、activation、performance、fallback比較を返す |

activation work itemが`operation.worlds.read`を採用する場合はAuthoring規約の`AuthoringSelectionContextV1` hashを任意入力として受けられるが、screen coordinate、表示row、Hierarchy path、表示名だけをtargetへ変換しない。出力はProject revision、Contract set hash、Source Document hash、Source／Staging／Derived read-only／Runtime区分、omitted range、continuationを必須とする。Derived Artifactはread-only refだけを返し、Cell、Navmesh、HLOD、Runtime handleのwrite primitive schemaを生成しない。

activation後に`operation.worlds.plan_change`を採用する場合、返すchange primitive discriminatorはWorld規約の`WorldSourceChangePrimitiveKindV1` Catalogに存在し、`ai_mutable` field coverage、expected Document revision、precondition hash、Risk、Approval、inverse availabilityを満たすものだけにする。behavior-neutralな予定語彙は`CreateWorld | UpdateWorldComposition | CreateScene | UpdateSceneComposition | DefineSpace | UpdateTopology | BindOwnerTypedDocument`へ限定し、Scene永続化owner、World composition membership、Space／Topology relation、Derived Cell assignmentを別constraintとしてProvider Schemaとserver-enforced semantic validatorへ投影する。これらは現在公開されていない。

`Level Workspace`はOperation familyまたはwrite authorityではない。将来Activation後、Workspace上の一つのintentがWorldと`StageDefinitionV1`等のpack-owned文書を変更する場合、各Ownerのactive Operationとtyped primitiveへ分解し、同じbase Project revisionの一つの`ProjectChangeSetV1`へ明示列挙する。一つでも未Activation、unauthorizedまたはstaleなら全体をread-only Gapまたはfailed proposalにし、Level固有Operation、自由patch、部分commitへfallbackしない。`Level`、`SetLevelSourceScenes`、`SetLevelEntryExitContract`、`SetLevelGameplayComposition`、`playable_level`はcurrent Core MCD Operation／primitive kind／constraintではなく、legacy Pack-owned migration inputにだけ存在できる。

World CapabilityのMCD `examples`は最低でも、明確な`world_structure`、曖昧で`question_required`、共有Scene変更の影響World列挙、Derived Cell直接write拒否、stale revision拒否を各1件含む。Exampleは説明文だけでなく、exact Input、expected disposition／Activation後のexact MCD Operation IDまたはchange primitive discriminator、expected Diagnostic code、変更されないinvariantを持つ。Pack-owned Stage等のcontent概念をCore World kindまたは暗黙空間前提として登録しない。

`MapIntentResolutionV1.disposition=question_required`をCommit可能Proposalへ自動変換しない。System／WorldのActivation、Source Promotion、Project Commit OperationはProvider projectionへ含めない。

Anti-alias Discoveryは2D／3D機能計画の意味GoalとRenderer規約の実行制約について次の五IDをplanning candidateとして予約する。current MCP aliasやProvider Toolをmaterializeしない。

| reserved candidate ID | 予定Risk／kind | 予定意味 |
|---|---|---|
| `operation.rendering.aa.search` | R0 query | 意味Goal、Target、Renderer、Quality、maturityからIntent／method候補を検索し、候補ID、短い適合理由、制約要約を返す |
| `operation.rendering.aa.read` | R0 query | exact Intent／method／Profileの互換Predicate、sample count、layer scope、cost model、fallback、Diagnostic、必要Qualificationを返す |
| `operation.rendering.aa.resolve_intent` | R0 query | 永続変更なしでIntentをViewFamily単位Planへ決定的に解決し、採用候補、却下候補と理由、`resolved`／`question_required`／`unsupported`を返す |
| `operation.rendering.aa.plan_change` | R1 proposal | expected Project revisionに対するtyped `AntiAliasingChangeSetProposalV1`だけを生成する。Commit、Provider activation、Pipeline rebuildは行わない |
| `operation.rendering.aa.preview_change` | R0 query／job | ProposalをStagingで検証し、resolved Plan差分、Graph／history影響、GPU／memory／bandwidth見積り、visual fixture要求、fallback、必要Receiptを返す |

activation work itemが五件を採用する場合、全Anti-alias Query／Proposal結果は`project_revision`、`contract_set_hash`、`capability_signature_hash`、`renderer_profile_revision`、`qualification_receipt_hashes`、`query_hash`を返し、ViewFamilyへ解決した結果は`view_family_id`と`source_intent_revision`も返す。bounded collection、revision、互換性、typed Diagnosticの規則も同じtransactionで登録する。

`AntiAliasingIntentV1`のclosed enum／tagged union規則はOperation Activationと独立したDomain契約として維持する。将来のProvider／MCP投影を採用する場合も候補五件以外を混ぜず、`ResolvedAntiAliasingPlanV1` write、arbitrary Render Graph write、Provider install／activate、Settings Apply、Source Promotion、Project CommitをTool listへ含めない。

Lighting DiscoveryはLighting規約の意味Role、物理単位、Target／Budget制約について次の九IDをplanning candidateとして予約する。current MCP aliasをmaterializeしない。

| reserved candidate ID | 予定Risk／kind | 予定意味 |
|---|---|---|
| `operation.lighting.search` | R0 query | Light／Profile／role／Target／maturityを検索 |
| `operation.lighting.read` | R0 query | field mask付きSource／Intent／Profile／Planを取得 |
| `operation.lighting.inspect` | R0 query | bounded Scene summary、lock、上限、cost、Diagnosticを取得 |
| `operation.lighting.resolve_intent` | R0 query | 永続変更なしで`ResolvedLightPlanV1`を決定的に生成 |
| `operation.lighting.plan_change` | R1 proposal | expected revisionに対する`LightingChangeSetProposalV1`を生成 |
| `operation.lighting.preview_change` | R0 query／job | before／after、contribution、cluster、Shadow、costを検証 |
| `operation.lighting.explain_plan` | R0 query | Intent→Source field、採用／棄却／fallback理由を取得 |
| `operation.lighting.estimate_cost` | R0 query | Target別CPU／GPU／Memory予測とconfidenceを取得 |
| `operation.lighting.validate_change` | R0 query／job | Schema／semantic／Capability／Budget／lockを検証 |

activation後に九件を採用する場合、結果は`project_revision`、`contract_set_hash`、`lighting_catalog_hash`、`target_capability_hash`、`query_hash`を必須とし、Plan／Previewは追加closure hashを返す。LightのCommit、native GPU resource／cluster buffer書込み、Shadow Technique Source追加、Provider activation、Project HLSL変更をTool listへ含めない。新規TechniqueへのhandoffもProject Shader familyが別途activateされた場合だけ可能である。

Post Process DiscoveryはIntent、Profile、Node Catalog、Volume、AA／Layer互換について次の九IDをplanning candidateとして予約する。current MCP aliasをmaterializeしない。

| reserved candidate ID | 予定Risk／kind | 予定意味 |
|---|---|---|
| `operation.post_process.search` | R0 query | Profile／Volume／Node／Target／maturityを検索 |
| `operation.post_process.read` | R0 query | field mask付きIntent／Profile／Volume／Planを取得 |
| `operation.post_process.inspect` | R0 query | View、active stage、history、layer、cost、Diagnosticを取得 |
| `operation.post_process.resolve_intent` | R0 query | 永続変更なしで`ResolvedPostProcessPlanV1`を決定的に生成 |
| `operation.post_process.plan_change` | R1 proposal | expected revisionに対する`PostProcessChangeSetProposalV1`を生成 |
| `operation.post_process.preview_change` | R0 query／job | before／after、色空間、layer、history、costを検証 |
| `operation.post_process.explain_plan` | R0 query | Intent→Node／parameter、採用／棄却／fallback理由を取得 |
| `operation.post_process.estimate_cost` | R0 query | Target別CPU／GPU／Memory予測とconfidenceを取得 |
| `operation.post_process.validate_change` | R0 query／job | Schema／stage／AA／Layer／Capability／Budgetを検証 |

activation後に九件を採用する場合、結果は`project_revision`、`contract_set_hash`、`post_node_catalog_hash`、`target_capability_hash`、`anti_aliasing_plan_hash`、`query_hash`を必須とし、Plan／Previewは追加closure hashを返す。raw Render pass、native resource、history weight、Node stage並替え、Provider activation、Project Shader Source／Technique mutation、Source Promotion、Project CommitはTool listへ含めない。Project Shaderへのhandoffは同familyのActivationを前提とする。

Lighting／Post Process familyを将来activateする場合、Searchは既定50件、最大200件、continuation付き、Read／Inspectはfield maskとbounded scope、`plan_change`は`expected_project_revision`と`idempotency_key`を必須にし、Provider OutputをInternal validatorで完全再検証する。

## 21. Project Shader Discovery／Proposal候補のplanning record（未Activation）

Project Shaderは[Project Shader](../06-rendering/project-shader.md)の`PublicShaderSdkCatalogV1`、Module、Technique、Fact、Understanding Closureについて次の16 IDをplanning candidateとして予約する。current MCP aliasをmaterializeしない。

| reserved candidate ID | 予定Risk／kind | 予定意味 |
|---|---|---|
| `operation.shader.search` | R0 query | Project／public SDK origin、semantic role、module kind、Stage、Technique Port、Target、Capability、qualificationからStable ID候補を検索 |
| `operation.shader.read_module` | R0 query | exact Module manifest、Source file index／bounded excerpt、public export、typed value／resource interface、Target、fallbackを取得 |
| `operation.shader.read_technique` | R0 query | exact Technique、Pass DAG、logical resource、Port、Target、fallbackを取得 |
| `operation.shader.inspect_symbol` | R0 query | Projectまたはpublic SDKのStable Symbol IDからdeclaration、semantic、type、source span、entry／export、Fact、diagnosticをbounded取得 |
| `operation.shader.find_callers` | R0 query | caller／callee、Pass／Entry到達性、Target／variant scopeを取得 |
| `operation.shader.explain_dataflow` | R0 query | value flow、unit／space／color変換、output寄与とEvidenceを取得 |
| `operation.shader.explain_resource_effects` | R0 query | resource access、side effect、Pass／queue intent、lifetime影響を取得 |
| `operation.shader.compare_targets` | R0 query | interface、Capability、precision、variant、fallback、Diagnostic差を取得 |
| `operation.shader.preview` | R0 query／job | exact Source／artifact／Targetでvisual／analytic fixtureを実行 |
| `operation.shader.parameter_sweep` | R0 query／job | bounded parameter setのcounterfactual、image／invariant／cost差を取得 |
| `operation.shader.estimate_cost` | R0 query | Target別instruction、resource、variant、GPU／memory予測と根拠を取得 |
| `operation.shader.validate_contract` | R0 query／job | Profile、semantic、Fact、reflection、Target、fixture、UnderstandingのDiagnosticを取得 |
| `operation.shader.plan_module` | R1 proposal | Sourceを変更しない`ProjectShaderModulePlanV1`を生成 |
| `operation.shader.plan_technique` | R1 proposal | Sourceを変更しない`ProjectShaderTechniquePlanV1`を生成 |
| `operation.shader.propose_module` | R3 source proposal | expected revisionに対するModule Source／contract ChangeSetをStagingへ生成 |
| `operation.shader.propose_technique` | R3 source proposal | expected revisionに対するTechnique／Module Source ChangeSetをStagingへ生成 |

activation work itemが16件を採用する場合、全結果は`project_revision`、`contract_set_hash`、`bounded_project_shader_profile_hash`、`source_tree_hash`、`query_hash`を必須とし、Fact／Target／Preview／Validate／Proposalごとのclosure hashをnamed Resultへ閉じる。Closure未生成を空hashで表さず、statusを`not_run | stale | failed | passed`として返す。

activation後のSearchは既定50件、最大200件、continuation付き、Read／Inspect／Explainはfield mask、Stable ID、Target／variant scope、最大node／edge／source byteを必須にする。上限超過時は`omitted_ranges`と`continuation_cursor`を返し、GraphまたはSourceを途中で正常完了扱いしない。

activation後のPlan／Proposeは`expected_project_revision`、`idempotency_key`、Profile hash、対象Module／Technique Stable ID、Target集合、Requirement、Budget、fixture、fallback、Riskを必須にする。PlanはR1 Proposal、ProposeはStagingへ実行code候補を生成するためR3 Source Proposalであり、後段Promotionが別であることを理由にRiskを下げない。ProposeはSource ChangeSetだけに限定し、compiler command、Project／Engine filesystem直接write、artifact publish、Commit、Activation、Approval、Policy変更を行わない。現在はPlan／Proposeを含む16件すべてを拒否する。

### 21.1 既存Domain文書から回収した未登録Operation候補

次の十表94件は、既存Domain文書が過去にcurrent／canonical／registeredとして記述していたname-only surfaceを、§20の`PlannedOperationFamilyV1`へ回収したclosed candidate集合である。各表は上のledger後半十行と同じ順で対応する。これらは予約語彙であってMCD Operationではなく、全current集合とalias集合は明示`[]`、Capability stateは`not_activated`である。要求はGateway dispatch前に`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`で拒否し、Source、Project、Registry、Taskを変更しない。

Math semantic authoringは次の六IDだけを予約する。Math文書にある`operation.camera.set_profile_projection`はCamera ownerの次表に属する同一候補への参照であり、Math familyへ複製しない。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.transform.set_world_position` | semantic positionの変更Proposal |
| `operation.transform.set_local_rotation` | semantic local rotationの変更Proposal |
| `operation.transform.set_local_scale` | semantic positive scaleの変更Proposal |
| `operation.physics.set_velocity` | Physics ownerへ渡すsemantic velocity変更Proposal |
| `operation.asset.set_pixels_per_unit` | Asset ownerへ渡すpixel density変更Proposal |
| `operation.ui.set_rect` | UI ownerへ渡すsemantic rectangle変更Proposal |

Camera authoringは次の11 IDだけを予約する。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.camera.resolve_intent` | Camera intentのread-only解決 |
| `operation.camera.create_profile` | Camera Profile作成Proposal |
| `operation.camera.set_profile_projection` | Projection semantic変更Proposal |
| `operation.camera.create_rig` | Rig作成Proposal |
| `operation.camera.add_rig_node` | typed Rig Node追加Proposal |
| `operation.camera.connect_rig_nodes` | typed Port接続Proposal |
| `operation.camera.set_director_rule` | Director rule変更Proposal |
| `operation.camera.set_presentation_profile` | Presentation Profile変更Proposal |
| `operation.camera.create_sequence` | Cinematic Sequence作成Proposal |
| `operation.camera.preview_candidate` | Staging candidateのPreview |
| `operation.camera.analyze_composition` | compositionのread-only解析 |

Material authoringは次の15 IDだけを予約する。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.material.search` | Material Catalog検索 |
| `operation.material.read` | bounded Material読取 |
| `operation.material.inspect` | Material closure検査 |
| `operation.material.preview` | Staging Material Preview |
| `operation.material.explain` | semantic resolution説明 |
| `operation.material.estimate` | Target別cost見積り |
| `operation.material.validate` | Material closure検証 |
| `operation.material.plan` | read-only変更Plan |
| `operation.material.create_instance` | Instance作成Proposal |
| `operation.material.assign_template` | Template割当Proposal |
| `operation.material.set_parameters` | typed parameter変更Proposal |
| `operation.material.create_definition` | Definition作成Proposal |
| `operation.material.edit_graph` | typed Graph変更Proposal |
| `operation.material.create_derived_style` | 派生Style作成Proposal |
| `operation.material.bind_surface_semantics` | Surface semantic binding Proposal |

VFX authoringは次の24 IDだけを予約する。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.vfx.inspect_system` | VFX System検査 |
| `operation.vfx.inspect_semantic_catalog` | semantic Catalog検査 |
| `operation.vfx.validate_changeset` | ChangeSet構造検証 |
| `operation.vfx.validate_semantic_preservation` | Cue semantic保存検証 |
| `operation.vfx.preview_changeset` | Staging Preview |
| `operation.vfx.resolve_effect_intent` | Effect intentのread-only解決 |
| `operation.vfx.set_effect_intent` | Effect intent変更Proposal |
| `operation.vfx.apply_pattern` | Pattern適用Proposal |
| `operation.vfx.create_system` | System作成Proposal |
| `operation.vfx.create_emitter` | Emitter作成Proposal |
| `operation.vfx.update_emitter` | Emitter更新Proposal |
| `operation.vfx.delete_emitter` | Emitter削除Proposal |
| `operation.vfx.add_node` | typed Node追加Proposal |
| `operation.vfx.update_node` | typed Node更新Proposal |
| `operation.vfx.delete_node` | typed Node削除Proposal |
| `operation.vfx.connect_nodes` | typed Port接続Proposal |
| `operation.vfx.disconnect_nodes` | typed Port切断Proposal |
| `operation.vfx.set_curve` | bounded Curve変更Proposal |
| `operation.vfx.set_gradient` | bounded Gradient変更Proposal |
| `operation.vfx.set_output` | Output変更Proposal |
| `operation.vfx.generate_fallback` | explicit fallback生成Proposal |
| `operation.vfx.capture_bounds` | bounded bounds capture |
| `operation.vfx.run_qualification` | Qualification実行Proposal |
| `operation.vfx.propose_extension_operator` | Extension operator Source Proposal |

Environment authoringは次の26 IDだけを予約する。

`operation.environment.explain_effective`を含む26件をinitial version 1のtarget planning definitionとする。これはnonmaterialized vocabularyであり、全current Operation集合は`[]`である。過去draftの25件集合、別version、alias、部分Activationを登録しない。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.environment.inspect_profile` | Environment Profile検査 |
| `operation.environment.explain_effective` | Source／Preset／Style／Weather／Time／fallback合成後の実効値と理由をread-only取得 |
| `operation.environment.list_presets` | bounded Preset列挙 |
| `operation.environment.resolve_intent` | Environment intentのread-only解決 |
| `operation.environment.validate_changeset` | ChangeSet検証 |
| `operation.environment.preview_changeset` | Staging Preview |
| `operation.environment.estimate_cost` | Target別cost見積り |
| `operation.environment.create_profile` | Profile作成Proposal |
| `operation.environment.set_world_binding` | WorldのEnvironment Profile binding変更Proposal |
| `operation.environment.apply_preset` | Preset適用Proposal |
| `operation.environment.set_intent` | Intent変更Proposal |
| `operation.environment.set_sky` | Sky変更Proposal |
| `operation.environment.set_sun_moon_link` | Sun／Moon relation変更Proposal |
| `operation.environment.set_height_distance_fog` | Height／distance fog変更Proposal |
| `operation.environment.set_volumetric_fog` | Volumetric fog変更Proposal |
| `operation.environment.create_local_fog_volume` | Local Fog Volume作成Proposal |
| `operation.environment.update_local_fog_volume` | Local Fog Volume更新Proposal |
| `operation.environment.delete_local_fog_volume` | Local Fog Volume削除Proposal |
| `operation.environment.set_atmosphere_preset` | Atmosphere Preset変更Proposal |
| `operation.environment.set_custom_atmosphere` | Custom Atmosphere変更Proposal |
| `operation.environment.set_cloud_layer` | Cloud Layer変更Proposal |
| `operation.environment.set_lighting` | Environment lighting変更Proposal |
| `operation.environment.bind_weather` | Weather binding Proposal |
| `operation.environment.generate_fallback` | explicit fallback生成Proposal |
| `operation.environment.bake` | offline bake Proposal |
| `operation.environment.run_qualification` | Qualification実行Proposal |

LOD authoringは次の二IDだけを予約する。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.lod.propose_policy` | read-only Policy Proposal |
| `operation.lod.apply_policy` | 承認済みPolicy ChangeSet Proposal |

Input selectionは次の一IDだけを予約する。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.input.semantic_action_binding.select` | Semantic Action Binding Selection作成／更新Proposal |

Navigation selectionは次の一IDだけを予約する。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.navigation.motion_intent_binding.select` | Motion Intent Binding Selection作成／更新Proposal |

Physics role selectionは次の一IDだけを予約する。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.physics.intent_role.select` | Physics Intent Role Selection作成／更新Proposal |

Feature authoringは次の七IDだけを予約する。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.feature.create_definition` | Feature Definition作成Proposal |
| `operation.feature.update_definition` | Feature Definition更新Proposal |
| `operation.feature.configure_system` | Feature System設定Proposal |
| `operation.feature.bind_runtime_port` | Runtime Port binding Proposal |
| `operation.feature.preview_change` | Feature変更Preview |
| `operation.feature.explain_contract` | Feature Contract説明 |
| `operation.feature.validate_contract` | Feature Contract検証 |

AI制作E2Eで`ProjectChangeSetV1`を外側のMCD Operationへ運ぶcarrierは次の四IDだけを予約する。`operation.authoring.apply_changeset`は§21.2の説明例／rejected classのまま維持し、alias、旧名、fallbackとして再利用しない。`commit`はActivation後も`trusted_internal`だけが人間Approvalを再検証して実行し、Provider／MCP／外部CLI Tool projectionをexact `[]`にする。

| reserved candidate ID | 予定Authority／意味 |
|---|---|
| `operation.authoring.changeset.propose` | R1 proposal。bounded primitiveからimmutable `ProjectChangeSetArtifactRefV1`とProposal結果を生成しProjectを変更しない |
| `operation.authoring.changeset.validate` | R0 validation。base revision、Scope、primitive reachability、schema／semantic boundを検証してsigned Validation Receipt候補を返す |
| `operation.authoring.changeset.preview` | R0 preview。canonical ChangeSet hashへ閉じたDiff／impact／invalidated consumerとPreview Receipt候補を返す |
| `operation.authoring.changeset.commit` | trusted-internal R3 commit。fresh Approval、Validation／Preview、same ChangeSet／Candidateを再検証し既存atomic Project publication algorithmだけを呼ぶ |

Project C++ Sourceの外部Client向けquery／proposal surfaceは次の五IDだけを予約する。Source Worker出力は直接Project Sourceへ書かず、Broker再計算Diffを持つ`NativeModulePatchProposalV1`候補までに限定する。

| reserved candidate ID | 予定Authority／意味 |
|---|---|
| `operation.native_module.search` | R0 query。Module、public SDK surface、Contract、Target、qualificationからbounded候補を返す |
| `operation.native_module.read` | R0 query。許可されたModule／path ScopeのSourceBundle ref、hash、omission、continuationを返す |
| `operation.native_module.inspect_contract` | R0 query。public SDK／ABI／Toolchain／Target互換性と禁止dependencyを説明する |
| `operation.native_module.plan` | R1 proposal。typed Source Task、path Scope、test plan、Code Owner assignment requirementを生成する |
| `operation.native_module.propose_patch` | R1 proposal。isolated Worker結果をBroker再検算し、before／after tree hashとtyped Deltaを持つPatch Proposalを返す |

Source revisionのPromotionは次の一IDだけを予約する。Activation後もProvider／MCP／standard external clientへ投影せず、独立Review、Code Owner Approval、Build／Test ReceiptをGatewayが再検証する`trusted_internal` Operationとする。

| reserved candidate ID | 予定Authority／意味 |
|---|---|
| `operation.project_source.promote_revision` | trusted-internal R3。approved Native／Shader source candidateをimmutable Project Source revisionへ昇格しsigned Promotion Receiptを発行する |

Project CandidateのValidate／Cook／Source build／統合build／Testは次の六IDだけを予約する。既存`build_device_play_debug_task` familyのPackage／Install／Launch／Smoke／Debug／Task 14件へ先行し、全結果をsame Project revision、Candidate root、Target、Toolchainへ閉じる。Packageはこの六familyの必要Receiptを欠く場合に成功できない。

| reserved candidate ID | 予定Authority／意味 |
|---|---|
| `operation.build.request_validate` | R2 job request。Project revision／Candidateのschema、reference、policy、target closureを検証する |
| `operation.build.request_cook` | R2 job request。validated CandidateからTarget-bound Cooked closureを生成する |
| `operation.build.request_native_module` | R2 job request。selected Native Source revisionをpublic SDK／ABI／Toolchain lockで隔離Build／Testする |
| `operation.build.request_project_shader` | R2 job request。selected Shader Source revisionをTarget／Renderer／Toolchain lockで隔離Build／Testする |
| `operation.build.request_game_candidate` | R2 job request。Validate／Cook／必要Source Build Receiptを同一Candidateへ統合する |
| `operation.test.request_run` | R2 job request。fixture／test plan／Targetをbounded指定しCandidate Test Receipt候補を返す |

この六IDのdestination execution contractは[Core architecture §9.2](../02-foundation/core-architecture.md#92-build-candidatetestのtyped-execution-closure)だけが所有する。`activation.build.candidate_test_operations.v1`は六件を部分集合化せず、同節のnamed Input exact六型、`BuildCandidateOperationTaskV1`、`OperationReceiptEnvelopeV1` mapping exact六行、success／failure／cancel payload exact六型、完成Receipt alias／exact purpose、`build_gateway | candidate_test_service` allowlist、Operation別Signer Role／singleton-purpose Key、Project revision／Candidate／Target／Toolchain／Source closure、idempotency／cancel fixture、Validate→Cook／Source Build→Game Candidate→Test→Packageのset-equality fixtureを一つのContract set transactionへ含める。Operation Registry、Owner Manifest、Service allowlist、Signer Policy、Input Type、Receipt Type、Validator closure、Diagnostic reachable setの各Operation ID集合はこの六IDとset equalityでなければならない。`build_device_play_debug_task`の14件は別familyとして保持し、Package purpose、Task ControlのActivation、同じexecution Authorityを六件のmaterialization根拠にしない。current familyは`not_activated`で全投影集合exact `[]`、Provider／MCP／CLI／Editor aliasもexact `[]`であり、Schema名、Receipt名、後段参照からdispatchしない。

GameplayDefinitionのauthoring surfaceは次の六IDだけを予約する。外部Clientは検索、計画、Proposal、Validation、Previewまでで、Project変更は上記ChangeSet carrierだけを通す。

| reserved candidate ID | 予定Authority／意味 |
|---|---|
| `operation.gameplay_definition.search` | R0 query。kind、Capability、owner layer、tag、Targetからbounded Definition候補を返す |
| `operation.gameplay_definition.read` | R0 query。exact revision／hash、field mask、dependency、omissionを返す |
| `operation.gameplay_definition.plan` | R1 proposal。typed Definition delta、質問、assumption、validation planを生成する |
| `operation.gameplay_definition.propose` | R1 proposal。bounded Definition primitiveをChangeSet候補へ追加する |
| `operation.gameplay_definition.validate` | R0 validation。schema、owner、Port、state writer、Target互換性を検証する |
| `operation.gameplay_definition.preview` | R0 preview。Definition diff、System Graph impact、Save／Replay／Cook invalidationを返す |

Asset authoring surfaceは次の十IDだけを予約する。Importer実行結果、Derived Asset、Registry、Source publicationを直接変更せず、Project ChangeSetとAsset Ownerのatomic publicationへ渡すProposal／Validationだけを作る。

| reserved candidate ID | 予定Authority／意味 |
|---|---|
| `operation.asset.search` | R0 query。role、kind、tag、license、Target、qualificationからbounded Asset候補を返す |
| `operation.asset.read` | R0 query。Source／Derived identity、revision、dependency、field mask、omissionを返す |
| `operation.asset.analyze_source` | R0 analysis。format、semantic role、license、security、import capabilityを副作用なしで解析する |
| `operation.asset.plan_import` | R1 proposal。Importer、Profile、Target、validation、fallbackをtyped planへ閉じる |
| `operation.asset.propose_import_settings` | R1 proposal。Importer-owned closed settings deltaを生成する |
| `operation.asset.preview_import` | R0 preview。staging import、Diff、diagnostic、budget impactを返しpublishしない |
| `operation.asset.propose_reimport` | R1 proposal。source／profile driftをexact revisionへ束縛した再import候補を作る |
| `operation.asset.resolve_reimport_conflict` | R1 proposal。closed conflict choicesとEvidenceを返し推測mergeしない |
| `operation.asset.plan_bulk_migration` | R1 proposal。bounded item集合、batch、rollback、failure policyを作る |
| `operation.asset.validate_import` | R0 validation。Source、Importer、Derived、license、Target、budget、dependency closureを検証する |

Game production intent理解は次のexact八IDだけを予約する。全件R1 Proposalで、Owner-typed recordまたはChangeSet primitive候補を返すだけでProjectを変更しない。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.game_intent.session.propose_create` | bounded scope、Project／Contract set、Context Capsuleを持つIntent session作成Proposal |
| `operation.game_intent.draft.propose_capture` | Session内Intent Draftのimmutable capture Proposal |
| `operation.game_intent.question.propose_answer` | exact open QuestionへのAnswer record Proposal |
| `operation.game_intent.question.propose_withdraw` | exact open Questionのwithdrawal Proposal |
| `operation.game_intent.assumption.propose_resolution` | Assumptionをacceptまたはreplacementへ閉じるProposal |
| `operation.game_intent.brief.propose_confirmation` | traceable Game Brief confirmation Proposal |
| `operation.game_intent.spec.propose_publication` | Requirement／Decision traceabilityを持つGameSpec publication Proposal |
| `operation.game_intent.understanding.propose_closure` | unresolved Question／Assumption countを再検証するUnderstanding Closure Proposal |

Game experience iterationは次のexact三IDだけを予約する。Observation、Evaluation、Decisionを相互代替せず、Human Gameplay Approvalを発行しない。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.game_experience.playtest_observation.propose_record` | participant、session、build、provenanceへ閉じたObservation record Proposal |
| `operation.game_experience.evaluation.propose` | Experience GoalとObservation集合をsame Candidateで評価するProposal |
| `operation.game_experience.iteration.propose_decision` | Evaluationから`accept_candidate | revise | stop | defer`のexact一branchを選ぶDecision Proposal |

Game production readは次のexact三IDだけを予約する。全件R0 read-onlyでmutation Receipt、Approval、Commit、PromotionまたはActivationを返さない。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.game_production.inspect` | bounded Game Brief／Spec／Question／Assumption／Decision／Playtest projectionを返す |
| `operation.game_production.trace` | Requirement、Decision、Change、Evidence、Observationのbounded traceを返す |
| `operation.game_production.explain` | Closure state、gap、rejection reason、required next decisionを説明する |

三familyは各ledger rowからField省略なし`PlannedOperationFamilyV1`へ展開し、十のcurrent projection集合、`generated_aliases`、`legacy_aliases`をすべてexact `[]`、`capability_state=not_activated`にする。Provider／MCPは将来Activation後もProposal／read subsetだけで、Project Commit、Human Gameplay Approval、Source Promotion、Activation、SigningまたはRelease authorityを持たない。

既存94件、追加32件、Game production 14件についてもfamily単位のatomic activationだけを許可する。Math文書からCamera候補への一回のcross-owner参照を除き、207件全体の候補IDは重複なしである。`67 + 6 + 11 + 15 + 24 + 26 + 2 + 1 + 1 + 1 + 7 + 4 + 5 + 1 + 6 + 6 + 10 + 8 + 3 + 3 = 207`、planning family数は`8 + 10 + 6 + 3 = 27`であり、count、ID union、全empty current集合、work itemをcompiler fixtureでexact比較する。

### 21.2 Architecture内`operation.*` tokenのclosed partition

Contract lintは`docs/architecture/**/*.md`からまず完全なStable ID tokenを切り出し、complete tokenのkind prefixがexact `operation`であるものだけをOperation IDとして分類する。`policy.operation.*`、`validator.operation.*`、`validator_closure.operation.*`、`diagnostic.operation.*`、`type.operation.*`の内部substringは、それぞれ別kindの完全IDであってOperation IDではない。`operations/<id>.mirakan.json`というFile例はpathとsuffixを除いた`<id>`を分類する。`operation.lod.*`、`operation.feature.*`、`operation.packs.*`、`operation.shooter.*`のようなwildcard／不完全prefixはStable IDではなく、MCD ref、alias、dispatch keyとして必ず拒否する。

current architectureで完全なOperation IDとして現れるtokenは、次の互いに素な三classのexact一つへ分類する。

| class | exact集合／件数 | materialization |
|---|---|---|
| `target_complete_not_materialized` | §8.1／§8.2の10件 | target closure定義だけ。current MCD／Manifest／Service／Policy／Validator／Diagnostic／Receipt／surface／Activation Evidence集合は全て`[]` |
| `reserved_not_activated` | §§20～21.1の27 family、207件 | `PlannedOperationFamilyV1`だけ。全current集合／alias集合`[]` |
| `example_pending_or_rejected` | 下表の10件 | current／planning／alias集合すべて`[]`。dispatch拒否 |

| exact non-current ID | reason |
|---|---|
| `operation.authoring.apply_changeset` | MCD filename／ID grammarの説明例だけ |
| `operation.build.package.validate` | Naming grammarの説明例だけ |
| `operation.asset.source.plan_import` | Namingのcorrect-direction例だけ |
| `operation.asset.do` | 明示invalid naming例 |
| `operation.debug.propose` | Debug findingからのgeneric proposalとして明示禁止 |
| `operation.runtime_ecs.search` | [Runtime ECS target review contract](../04-runtime/entity-component-system.md)の提案語彙。current MCD／Operation／Tool集合は`[]` |
| `operation.runtime_ecs.describe_contract` | [Runtime ECS target review contract](../04-runtime/entity-component-system.md)の提案語彙。current MCD／Operation／Tool集合は`[]` |
| `operation.runtime_ecs.inspect_capture` | [Runtime ECS target review contract](../04-runtime/entity-component-system.md)の提案語彙。current MCD／Operation／Tool集合は`[]` |
| `operation.runtime_ecs.explain_access` | [Runtime ECS target review contract](../04-runtime/entity-component-system.md)の提案語彙。current MCD／Operation／Tool集合は`[]` |
| `operation.runtime_ecs.explain_failure` | [Runtime ECS target review contract](../04-runtime/entity-component-system.md)の提案語彙。current MCD／Operation／Tool集合は`[]` |

lintは完全ID token集合について`target_complete_not_materialized ∪ reserved_not_activated ∪ example_pending_or_rejected`とのset equality、三classのpairwise intersectionが空、current時点の未分類件数exact 0を検査する。三classはいずれもcurrent Operationではなく、current materialized／contract-active／active／operational集合とのintersectionをexact emptyにする。説明上のaction名、C++関数、future tokenをOperationへ推測昇格しない。新しい完全`operation.*` tokenを文書へ追加する変更は、同じ変更でtarget complete closure、Field省略なしのplanning family record、または明示的なexample／rejected分類のexact一つへ追加しなければ失敗する。materialized consumerが存在しないlegacy migration IDの予約は拒否する。

さらにprose claim lintは全architecture文書の`registered Operation`、`current Operation`、`public Operation`、`Operationを使う／公開する／登録する`に相当する現在形を検査する。現RepositoryでMCD currentを意味する肯定claimの許容件数はexact 0である。target complete候補はexact完全ID、version、`target_complete_not_materialized` membership、current全投影集合`[]`、`not_activated`を必須とする。ChangeSet内mutationはexact `ProjectChangePrimitiveV1`等の`*Primitive*`／`*JobKind*` discriminator、未登録Domain語彙は`planned semantic action vocabulary`＋current全投影集合`[]`＋`not_activated`＋future atomic work itemを必須にしてMCD Operation claimから除外する。説明対象が将来Activation後の挙動なら`Activation後`を明記する。これらのどれにも分類できない現在形Operation claimの許容件数はexact 0である。
