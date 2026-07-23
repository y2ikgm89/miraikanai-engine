# Miraikanai Engine Architecture Evolution Control Plane Design

- 作成日: 2026-07-22
- 状態: 設計更新済み（実装、署名ceremony、Artifact qualification、Production activationは未実施）
- 推奨方針: Owner正本化を採用候補とする。実装着手authorityは外部`ControlPlaneConstructionAuthorizationV1`だけが与える
- 対象Baseline: Authorizationの`authorized_base_git_tree_id`で固定するcurrent Git treeと、Task 1が生成しsidecar Receiptへ束縛するsigned inventory
- 後続Subsystem計画: [AI-Readable Direct3D 12 Backend Design](2026-07-22-ai-readable-d3d12-backend-design.md)
- 対象外: Engine実装code、Build script、MCD compiler実装、既存仕様の承認状態変更

## 1. 結論

Miraikanai Engineの各Subsystem仕様は、single writer、typed command／event／snapshot、Source／Derived分離、Target別Qualification、negative fixture、last-valid rollbackを既に強く定義している。一方で、機能追加や契約変更が複数Subsystemへ波及するときに必要な次の横断Ownerが欠けている。

1. Architecture文書のlifecycle、typed relation、Owner、変更影響を決めるOwner。
2. Public Contract、Project、Save、Pack、ABI、Packageの互換性と進化を決めるOwner。
3. System別Save projectionを一つの永続transactionへ集約するOwner。
4. Cook済みRuntimeをload可能なRuntime Packageへ束ねるOwner。
5. Runtime／Content／Shader／Platform artifactをApplication Package、署名、提出へ束ねるOwner。

本設計では次の5正本を追加し、signed Task 1 inventoryに存在する全current正本をそれらへ接続する。文書数を設計時の固定値にしない。

- `docs/architecture/01-governance/architecture-governance.md`
- `docs/architecture/02-foundation/compatibility-evolution.md`
- `docs/architecture/04-runtime/persistence-save.md`
- `docs/architecture/04-runtime/runtime-package.md`
- `docs/architecture/07-platform/application-package-release.md`

保留中のRuntime ECS Decisionは、この5正本と既存Ownerの更新が完了するまでactive specへ昇格しない。新ECS正本は上記横断契約を消費し、Save、Package、Compatibility、Document Governanceを再定義しない。

## 2. 完了条件

計画書更新は、文章を追加しただけでは完了としない。次をすべて満たした状態を完了とする。

1. 全active specが一意な文書ID、文書状態、正本範囲、非正本範囲、`requires`、`integrates_with`を持つ。
2. `requires` graphがcycleのないDAGになり、実装順序と変更影響を決定できる。
3. 全cross-document canonical contractに一意なOwnerがあり、consumerはOwnerへの参照だけを持つ。
4. 永続またはwire Schema typeはすべて`PascalCaseV<major>`で、suffixなしlatest aliasが0件である。
5. Capability ID、Work Package ID、Contract ID、Artifact IDがmaturity、document status、表示名をidentityへ埋め込まない。
6. 文書状態、Capability product tier、Capability activation、Work Package scheduling、Contract registration、Target readiness、Evidence freshness、Dependency adoptionが別dimension／state machineとして定義される。
7. `Authoring -> Cook -> Runtime Package -> Application Package -> Sign -> staging upload/read-back -> Game Candidate -> Human Approval／Activation -> Store submit/publish/read-back`が型付きmanifestとReceiptで閉じる。
8. `Gameplay/System Save Contract -> Runtime checkpoint -> Save aggregate -> Platform atomic storage -> staged load/publish`が型付きcontractで閉じる。
9. Packのminimum Engine contract、Feature dependency DAG、resolved lock、migration、exact Qualificationがmanifestから検証できる。
10. Pre-1.0 clean breakとreleased user data保護、Post-1.0 SemVer、migration、support policyの適用対象がartifact classごとに決まる。
11. Product Phase 0～9、Work Package、Capability、Owner、fixture、Target、fallbackの参照にorphanがない。
12. Index件数、Local link／anchor、文書ID、Owner、DAG、型名、ID文法、Phase参照を機械検査できる。
13. Runtime ECS Decisionが存在しないSave Owner、Package Owner、型versionを前提にしない。
14. 外部Tool／SDK／Libraryのrelease、tag object、peeled commit、artifact hashを区別する。
15. Manifest、Package、Candidate、Receiptのidentity graphに自己参照またはhash cycleがなく、全digestの入力byte列を一意に再構成できる。
16. 人間とAIが同じexact SourceからOwner、依存、phase／lifetimeを説明でき、上限超過、stale revision、根拠欠落を正常結果として扱わない。
17. 5新Owner、Product registry source、Toolchain lock、承認Decisionを一つの承認対象Git treeへ束ねた有効な`ControlPlaneBootstrapApprovalV1`が存在しない限り、Phase 0のWork Packageを`ready`へ遷移させない。

## 3. 非目標

- 全Subsystem algorithmを本変更で再設計しない。
- 既存の正しいRuntime phase、Budget、Physics／Navigation／Rendering algorithmを横断Ownerへ移さない。
- Public Marketplace、任意binary plugin、Runtime AI生成、Online／Multiplayerを暗黙にActivationしない。
- 後方互換shim、旧Schema alias、二重serializer、二重Package入口を追加しない。
- 文書の`review`を自動的に`approved`へ変更しない。
- 実装がない状態でPerformance、Security、Target互換性を合格済みと表現しない。
- MCD compilerやEngine実装の成功を文書整合性だけから推定しない。

### 3.1 D3D12 Companionとの変更境界

[AI-Readable Direct3D 12 Backend Design](2026-07-22-ai-readable-d3d12-backend-design.md)は、本設計が追加する5横断Ownerの6番目ではない。`docs/architecture/06-rendering/d3d12-backend.md`を一意OwnerとするRendering Subsystemの後続設計である。本横断ChangeSetの範囲をD3D12 Backend詳細へ拡張しない。

依存関係と実行順序を次に固定する。

1. 本設計でArchitecture Governanceのmetadata／relation／Owner grammarを確定する。
2. 本横断ChangeSetを完了し、共有するRendering／Windows／UI／Toolchain文書を一つの整合したbaselineにする。
3. D3D12 Companionを別ChangeSetで反映し、Render Graphとnative BackendのOwner分離、AI Profile分離、MCD projection、Qualification Gateを追加する。
4. D3D12詳細設計の承認後にのみ実装計画を作り、Product Phase 2 Windows vertical sliceより前にTarget Qualificationを閉じる。

共有文書を両ChangeSetで同時編集しない。D3D12 Companionは本設計の完了hashをinput baselineとし、完了前の本横断ChangeSetへD3D12固有型または実装を混入させない。

## 4. 設計原則

### 4.1 一つの意味に一つのOwner

Contractの構造、Field意味、state transition、failure policyを二つの文書で共同所有しない。横断文書はDomain意味を再掲せず、exact contract referenceと必須invariantだけを所有する。

### 4.2 Authority、data、executionを分離する

- Governanceは承認、Policy、Evidence freshnessを所有する。
- DomainはSource意味とDomain固有Stateを所有する。
- Runtimeはstaging、publication、lease、transactionを所有する。
- PlatformはOS storage、package format、signing adapterを所有する。
- Build Gatewayは型付きtaskの順序付けを行うが、各artifactの意味を所有しない。

### 4.3 Strict validationとcompatible evolutionを両立する

未知Fieldを黙って無視する汎用tolerant readerは採用しない。互換性は、明示されたSchema revision、version negotiation、migration、extension pointだけで成立させる。

同一major内のadditive changeでは、新readerが旧writerのartifactを読めることを必須とする。旧readerが新writerを読むforward compatibilityは約束しない。Writerはconsumerが合意したexact revisionを出力し、`latest`を出力しない。

### 4.4 Rebuild可能物とUser dataを分離する

Derived cache、BMI、device cache、Runtime Package、Application Packageはexact Sourceから再生成できる。Project Source、Save、User setting、承認済みGame revisionは再生成不能なUser dataとしてmigrationとrollbackを要求する。

### 4.5 ActivationはEvidenceからだけ決める

文書の存在、Schemaの存在、candidate binaryの存在をCapabilityの`qualified`または`production`とみなさない。Target別Receiptとfreshnessを必要とする。

### 4.6 Identityは結果から前提へ逆参照しない

Build入力を表すsubject、生成artifact、検証Receipt、最終Release recordを別identityにする。前段artifactへ後段Candidate hashを埋め込まず、manifest自身のhashをmanifest内へ持たせない。これにより自己参照hash、再署名によるidentity変化、Target間の循環参照を禁止する。

## 5. 目標Architecture

```text
Architecture Governance
  ├─ document lifecycle / typed relation / owner audit
  └─ change impact closure

Compatibility & Evolution
  ├─ public contract / schema / ID / SemVer
  ├─ migration / deprecation / support policy
  └─ artifact-class compatibility

Authoring Source
  -> Cook / Contract compiler / Content build
  -> Runtime Package
       -> Runtime loader
       -> Persistence compatibility binding
  -> Application Package Assembly
       -> Platform validator
       -> Signing service
       -> Store staging upload and remote read-back
       -> Game Candidate / Human Approval / Activation
       -> Store submit, publish, and remote read-back

Gameplay/System `SaveReplayContractV1`
  -> Persistence coordinator
       -> World/ECS/Domain save projectors
       -> SaveCheckpointV1
       -> PlatformStorageTransactionV1
       -> SaveCommitReceiptV1
```

## 6. 新しい正本仕様

| Path | Stable document ID |
|---|---|
| `01-governance/architecture-governance.md` | `mirakan.arch.architecture-governance` |
| `02-foundation/compatibility-evolution.md` | `mirakan.arch.compatibility-evolution` |
| `04-runtime/persistence-save.md` | `mirakan.arch.runtime-persistence-save` |
| `04-runtime/runtime-package.md` | `mirakan.arch.runtime-package` |
| `07-platform/application-package-release.md` | `mirakan.arch.platform-application-package-release` |

5文書は初回作成時にmetadata v2本文Coreと`construction_seed` Lifecycle Recordで`review`とする。本設計の承認をEngine仕様の自動承認へ流用しない。

Product RegistryのWork Packageがこの5文書のいずれかを`owner_document_id`として参照する場合、current `DocumentLifecycleRecordV1` headが`to_state=approved`で、束縛した`ArchitectureDocumentCoreV1`のfile hash／Git blobが現在bytesと一致するまで`declared`から遷移させない。開始要求は`diagnostic.architecture.owner-unapproved`（code `MIRAKAN-ARCHITECTURE-OWNER-UNAPPROVED`）で拒否し、未承認Ownerの責務をconsumer文書、実装Plan、暫定schemaへ複写して迂回しない。

Control Plane自身はまだbaseline、Policy Service、Product state publisherを持たないため、通常`CriticalPathTaskV1`／Work Package lifecycleで実装しない。実装計画Task 0～10Bをpre-baseline construction laneとし、開始にはRepository内Artifactから自己発行できない`OfflineGovernanceRootProfileV1`と、そのRootによる`ControlPlaneConstructionAuthorizationV1`を外部前提とする。Root Profileはoffline quorum public-key fingerprint、trust domain、threshold、secure signer custody、recovery／rotation policyを人間ceremonyでout-of-band照合し、worker、AI、生成対象treeの鍵をrootにしない。Construction Authorizationはexact authorized base Git tree、Control Plane design／implementation plan hash、許可Task `0..10B`、Task別changed-path allowlist、有効期限を署名する。各TaskはReceipt chainでinput／output treeとchanged pathsを閉じ、A→F current CAS→Construction Decision→B→C→D→Eのstaged artifact／sidecar境界を越えない。Construction executorには通常のWP ready／complete、Capability／Gate／Risk変更、Release、Active Product operational signingを許可せず、その`exceptional_operations[]`例外はTask 10Bがfinal Bootstrap Approvalを入力に`policy.product.state.genesis.v1`と`policy.product.wp.bootstrap-control-plane.v1`で行うgenesis／Control Plane completionだけにexact限定する。FはConstruction executor／Task例外ではなく、Product Planの既存Future authorityを持つ独立human R4によるtree外Approvalとplanning-only current CASであり、Active／operational mutationを行わない。

上記trust contractは次のclosed schemaだけを使用する。全objectはunknown Fieldを拒否し、content-derived IDは下記Content ID Registryに列挙したtop-level Fieldだけへ適用する。`*_id`という名前だけから推論せず、nested `key_id`／`task_id`／`schema_id`／logical IDをpayload hash規則へ巻き込まない。hash／content ref、配列sort、safe integerはProduct Plan §11.1と同じ規則を使う。

```text
OfflineGovernanceThresholdSignatureV1
  key_id
  algorithm = ecdsa-p256-sha256
  format = p1363-fixed-64-low-s
  signature_base64url_no_padding

OfflineGovernanceThresholdSigningSubjectV1
  domain_separator = mirakan-offline-governance-threshold-v1
  subject_schema_id
  subject_ref
  subject_sha256
  purpose
  issued_at
  valid_until

OfflineGovernanceThresholdSignedRecordV1
  record_id
  signing_subject: OfflineGovernanceThresholdSigningSubjectV1
  signatures[]: OfflineGovernanceThresholdSignatureV1

OfflineGovernanceAssuranceEvidenceBindingV1
  evidence_kind = governance_key_custody | bootstrap_verifier_reproduction |
    genesis_source
  evidence_ref: content-addressed ref
  evidence_sha256: lowercase hex 64

OfflineGovernanceAssuranceEvidencePayloadV1
  evidence_kind = governance_key_custody | bootstrap_verifier_reproduction |
    genesis_source
  subject_projection_schema_id
  subject_sha256
  challenge_nonce_base64url_32
  issuer_subject_ref
  issuer_public_key_spki_ref, issuer_public_key_spki_sha256
  verification_profile_ref, verification_profile_sha256
  status = passed
  completed_at
  valid_until

OfflineGovernanceAssuranceEvidenceV1
  payload: OfflineGovernanceAssuranceEvidencePayloadV1
  proof_kind = hardware_vendor_chain | independent_verifier_signature
  proof_schema_id
  proof_ref: content-addressed ref
  proof_sha256: lowercase hex 64
  issuance_revocation_evidence_ref: content-addressed ref
  issuance_revocation_evidence_sha256: lowercase hex 64

OfflineGovernanceAssuranceEvaluationJournalRowV1
  evaluation_kind = authority_pass | revocation_detection
  consumer_schema_id, consumer_ref, consumer_sha256
  evidence_ref, evidence_sha256
  evaluation_time
  current_root_head_ref, current_root_head_sha256
  current_global_super_head_ref, current_global_super_head_sha256
  issuance_revocation_evidence_ref, issuance_revocation_evidence_sha256
  current_revocation_evidence_ref, current_revocation_evidence_sha256
  vendor_revocation_observation_ref: null | content-addressed ref
  vendor_revocation_observation_sha256: null | lowercase hex 64
  verifier_profile_ref, verifier_profile_sha256
  result = passed | revoked_detected

OfflineGovernanceVendorRevocationObservationV1
  issuer_public_key_spki_sha256
  certificate_serial_sha256
  source_kind = vendor_crl | vendor_ocsp
  sequence: positive safe integer
  previous_observation_ref: null | content-addressed ref
  previous_observation_sha256: null | lowercase hex 64
  response_ref, response_sha256
  crl_number_base64url_uint: null | canonical unsigned integer, 1..20 octets
  produced_at: null | canonical UTC second
  this_update
  request_nonce_sha256: null | lowercase hex 64
  ocsp_nonce_consumption_ref: null | content-addressed ref
  ocsp_nonce_consumption_sha256: null | lowercase hex 64
  observed_certificate_status = good | revoked
  earliest_revocation_effective_at: null | canonical UTC second
  observed_at

OfflineGovernanceOcspNonceReservationV1
  issuer_public_key_spki_sha256
  responder_public_key_spki_sha256
  certificate_serial_sha256
  request_der_ref, request_der_sha256
  request_nonce_sha256
  requested_at
  expires_at

OfflineGovernanceOcspNonceConsumptionV1
  reservation_ref, reservation_sha256
  response_ref, response_sha256
  received_at

OfflineGovernanceAssuranceIssuerQuarantineObligationV1
  issuer_public_key_spki_sha256
  certificate_serial_sha256
  terminal_observation_ref, terminal_observation_sha256
  revoked_response_ref, revoked_response_sha256
  assurance_failure_effective_at
  created_at
  state = pending

OfflineGovernanceAssuranceIssuerQuarantineFulfillmentV1
  obligation_ref, obligation_sha256
  route_kind = post_overlay_normal_root |
    vendor_source_context_fail_stop | unaffected_recovery_custodian
  global_root_revocation_super_head_ref: null | content-addressed ref
  global_root_revocation_super_head_sha256: null | lowercase hex 64
  recovery_readiness_head_ref, recovery_readiness_head_sha256
  assurance_issuer_fail_stop_ref: null | content-addressed ref
  assurance_issuer_fail_stop_sha256: null | lowercase hex 64
  fulfilled_at

OfflineGovernanceAssuranceEvidencePolicyV1
  assurance_profile = development_bootstrap | production
  kind_bindings[]:
    {evidence_kind, evidence_schema_id, subject_projection_schema_id,
     proof_schema_bindings[]: {proof_kind, proof_schema_id},
     trusted_issuer_manifest_ref,
     trusted_issuer_manifest_sha256,
     accepted_verification_profiles[]:
       {verification_profile_ref, verification_profile_sha256},
     minimum_distinct_issuers,
     max_age_seconds}

OfflineGovernanceRootProfilePayloadV1
  root_profile_id
  trust_domain
  generation: positive safe integer
  assurance_profile = development_bootstrap | production
  root_keys[]:
    {key_id, algorithm, format, public_key_spki_ref, public_key_spki_sha256,
     custodian_subject_ref,
     custody_kind = os_backed_non_exportable | hardware_non_exportable,
     custody_evidence: OfflineGovernanceAssuranceEvidenceBindingV1}
  allowed_purposes[]
  purpose_quorums[]:
    {purpose, exact_key_ids[], distinct_custodian_threshold}
  rotation_policy_ref
  rotation_policy_sha256
  recovery_policy_ref
  recovery_policy_sha256
  assurance_evidence_policy_ref
  assurance_evidence_policy_sha256
  control_plane_governance_recovery_policy_ref
  control_plane_governance_recovery_policy_sha256
  offline_governance_verifier_profile_ref
  offline_governance_verifier_profile_sha256
  offline_governance_schema_catalog_ref
  offline_governance_schema_catalog_sha256
  issued_at
  valid_until

OfflineGovernanceRootProfileV1
  payload: OfflineGovernanceRootProfilePayloadV1
  threshold_signed_record: OfflineGovernanceThresholdSignedRecordV1

OfflineGovernanceEvidenceBindingV1
  evidence_kind
  evidence_ref: content-addressed ref
  evidence_sha256: lowercase hex 64

OfflineGovernanceRotationPolicyV1
  policy_id
  old_root_purpose = offline_governance_root_rotation_old
  new_root_purpose = offline_governance_root_rotation_new
  quorum_source = exact_root_profile_purpose_quorum
  overlap_seconds
  required_ceremony_evidence_kinds[]
  required_purpose_change_risk_evidence_kinds[]
  required_assurance_static_policy_change_risk_evidence_kinds[]
  evidence_kind_schema_bindings[]:
    {evidence_kind, evidence_schema_id, subject_projection_schema_id,
     max_age_seconds: positive safe integer}
  assurance_static_policy_change_invariant:
    {evidence_kind = assurance_static_policy_change_independent_r4,
     evidence_schema_id = urn:mirakan:schema:offline-governance:assurance-static-policy-change-approval:v1,
     subject_projection_schema_id = urn:mirakan:schema:offline-governance:assurance-static-policy-change-subject:v1,
     signed_record_purpose = offline_governance_assurance_static_policy_change_approval,
     signer_role_ref = role.offline-governance-assurance-policy-change-approver.r4,
     required_independence_class = independent_human_r4,
     max_age_seconds: positive safe integer}
  rollback_window_seconds
  ceremony_budget_seconds
  cas_retry_budget_seconds
  minimum_rotation_lead_seconds

OfflineGovernancePurposeSetChangeManifestV1
  manifest_id
  previous_allowed_purposes[]
  next_allowed_purposes[]
  changes[]:
    {purpose, change_kind = added | removed,
     subject_schema_id, schema_owner_document_id, schema_owner_document_sha256,
     architecture_approval_ref, architecture_approval_sha256,
     risk_evidence[]: OfflineGovernanceEvidenceBindingV1}
  generated_at

OfflineGovernanceAssuranceStaticPolicyChangeManifestV1
  previous_assurance_policy_ref, previous_assurance_policy_sha256
  next_assurance_policy_ref, next_assurance_policy_sha256
  previous_trusted_issuer_manifest_pairs[]:
    {evidence_kind, manifest_ref, manifest_sha256}
  next_trusted_issuer_manifest_pairs[]:
    {evidence_kind, manifest_ref, manifest_sha256}
  changes[]:
    {evidence_kind, change_kind = added | removed | modified,
     previous_manifest_ref: null | content-addressed ref,
     previous_manifest_sha256: null | lowercase hex 64,
     next_manifest_ref: null | content-addressed ref,
     next_manifest_sha256: null | lowercase hex 64,
     risk_evidence[]: OfflineGovernanceEvidenceBindingV1}
  generated_at

OfflineGovernanceAssuranceStaticPolicyChangeSubjectV1
  change_manifest_ref, change_manifest_sha256
  source_root_profile_ref, source_root_profile_sha256
  destination_root_profile_ref, destination_root_profile_sha256
  source_trust_registry_head_ref, source_trust_registry_head_sha256
  source_trust_registry_closure_ref, source_trust_registry_closure_sha256
  changed_evidence_kinds[]

OfflineGovernanceAssuranceStaticPolicyChangeApprovalPayloadV1
  evidence_kind = assurance_static_policy_change_independent_r4
  subject_projection_schema_id = urn:mirakan:schema:offline-governance:assurance-static-policy-change-subject:v1
  subject_projection: OfflineGovernanceAssuranceStaticPolicyChangeSubjectV1
  subject_sha256
  status = passed
  issuer_subject_ref
  issuer_role_ref = role.offline-governance-assurance-policy-change-approver.r4
  completed_at
  revocation_snapshot_ref

OfflineGovernanceAssuranceStaticPolicyChangeApprovalV1
  payload: OfflineGovernanceAssuranceStaticPolicyChangeApprovalPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=offline_governance_assurance_static_policy_change_approval)

OfflineGovernanceRecoveryPolicyV1
  policy_id
  triggering_incident_kinds[]
  total_compromise_incident_kinds[]
  uncompromised_root_purpose = offline_governance_root_recovery_uncompromised
  recovery_custodians[]:
    {key_id, subject_ref, public_key_spki_ref, public_key_spki_sha256,
     algorithm = ecdsa-p256-sha256, format = p1363-fixed-64-low-s,
     custody_kind = os_backed_non_exportable | hardware_non_exportable,
     custody_evidence: OfflineGovernanceAssuranceEvidenceBindingV1}
  recovery_custodian_quorum:
    {purpose = offline_governance_root_recovery_custodian,
     exact_key_ids[], distinct_subject_threshold}
  cooling_off_seconds
  total_compromise_cooling_off_seconds
  required_incident_evidence_kinds[]
  required_post_recovery_audit_kinds[]
  required_readiness_drill_evidence_kinds[]
  readiness_incident_evidence_kinds_by_reason[]: {reason_code, evidence_kinds[]}
  assurance_issuer_detection_requirements[]:
    {detection_source_kind = vendor_signed_revocation |
       independent_offline_assessment,
     required_nested_evidence_kinds[]}
  evidence_kind_schema_bindings[]:
    {evidence_kind, evidence_schema_id, subject_projection_schema_id,
     max_age_seconds: positive safe integer}

OfflineGovernanceTrustedIssuerManifestV1
  revision: positive safe integer
  entries[]:
    {issuer_subject_ref, issuer_public_key_spki_ref,
     issuer_public_key_spki_sha256,
     allowed_proof_kinds[],
     accepted_algorithms[]: VendorSignatureAlgorithmV1[] |
       [ecdsa-p256-sha256],
     accepted_formats[] = [p1363-fixed-64-low-s] |
       [vendor-x509-der],
     accepted_verification_profiles[]:
       {verification_profile_ref, verification_profile_sha256},
     accepted_revocation_parser_profiles[]:
       {parser_profile_ref, parser_profile_sha256},
     trust_anchor_ref: null | content-addressed ref,
     trust_anchor_sha256: null | lowercase hex 64,
     chain_policy_ref: null | content-addressed ref,
     chain_policy_sha256: null | lowercase hex 64,
     revocation_mode = vendor_crl | vendor_ocsp |
       offline_root_ledger,
     revocation_responder_spki_sha256: null | lowercase hex 64,
     offline_revocation_genesis_ref: null | content-addressed ref,
     offline_revocation_genesis_sha256: null | lowercase hex 64,
     max_revocation_response_age_seconds: positive safe integer,
     ocsp_nonce_requirement = required | not_applicable,
     valid_from, expires_at}

OfflineGovernanceAssuranceIssuerRevocationSnapshotV1
  sequence: positive safe integer
  previous_snapshot_ref: null | content-addressed ref
  previous_snapshot_sha256: null | lowercase hex 64
  entries[]:
    {issuer_subject_ref, issuer_public_key_spki_sha256,
     effective_at,
     reason_code = key_compromise | issuer_distrust | policy_violation}
  recorded_at

OfflineGovernanceVendorChainPolicyV1
  permitted_root_spki_sha256s[]
  permitted_policy_oids[]
  required_leaf_extended_key_usages[]
  maximum_path_length: positive safe integer
  accepted_certificate_signature_algorithms[]: VendorSignatureAlgorithmV1[]
  accepted_response_signature_algorithms[]: VendorSignatureAlgorithmV1[]
  ocsp_nonce_requirement = not_applicable | required
  ocsp_responder_revocation_method: null | independent_crl |
    nocheck_short_lived
  ocsp_nocheck_max_certificate_lifetime_seconds: null | positive safe integer

VendorSignatureAlgorithmV1 = ecdsa-p256-sha256 | ecdsa-p384-sha384 |
  rsa-pss-sha256 | rsa-pss-sha384

OfflineGovernanceVendorRevocationEvidenceV1
  source_kind = vendor_crl | vendor_ocsp
  signed_response_der_ref, signed_response_der_sha256
  target_certificate_der_ref, target_certificate_der_sha256
  parser_profile_ref, parser_profile_sha256
  issuer_certificate_chain[]:
    {certificate_der_ref, certificate_der_sha256,
     certificate_spki_sha256}
  issuer_subject_ref
  issuer_public_key_spki_sha256
  response_signer_public_key_spki_sha256
  trust_anchor_spki_sha256
  certificate_serial_sha256
  target_certificate_issuer_name_der_sha256
  target_certificate_authority_key_identifier_sha256: null | lowercase hex 64
  response_issuer_name_der_sha256: null | lowercase hex 64
  response_authority_key_identifier_sha256: null | lowercase hex 64
  ocsp_responder_id_kind: null | by_name | by_key
  ocsp_responder_id_sha256: null | lowercase hex 64
  ocsp_responder_revocation_evidence_ref: null | content-addressed ref
  ocsp_responder_revocation_evidence_sha256: null | lowercase hex 64
  ocsp_responder_nocheck_extension_present: null | boolean
  certificate_status = good | revoked
  revoked_at: null | canonical UTC second
  invalidity_at: null | canonical UTC second
  revocation_reason: null | key_compromise | ca_compromise |
    affiliation_changed | superseded | cessation_of_operation |
    certificate_hold | privilege_withdrawn | aa_compromise | unspecified
  request_nonce_sha256: null | lowercase hex 64
  crl_number_base64url_uint: null | canonical unsigned integer, 1..20 octets
  produced_at: null | canonical UTC second
  certificate_crl_distribution_point_der_sha256: null | lowercase hex 64
  crl_issuing_distribution_point_der_sha256: null | lowercase hex 64
  covered_revocation_reasons[]: key_compromise | ca_compromise |
    affiliation_changed | superseded | cessation_of_operation |
    certificate_hold | privilege_withdrawn | aa_compromise
  ocsp_cert_id_hash_algorithm: null | sha256
  ocsp_cert_id_issuer_name_hash: null | lowercase hex 64
  ocsp_cert_id_issuer_key_hash: null | lowercase hex 64
  ocsp_cert_id_serial_sha256: null | lowercase hex 64
  scope_covers_certificate = true
  delta_crl = false
  indirect_crl = false
  this_update
  next_update
  response_signature_algorithm: VendorSignatureAlgorithmV1
  response_encoding = rfc5280-crl-der |
    rfc6960-basic-ocsp-response-der

OfflineGovernanceObservedFingerprintManifestV1
  trust_domain
  root_generation
  root_profile_ref, root_profile_sha256
  witness_subject_ref
  entries[]:
    {key_id, public_key_spki_sha256,
     fingerprint_algorithm = sha256-spki-der,
     displayed_fingerprint_sha256, observed_at}

OfflineGovernanceVerificationChannelManifestV1
  trust_domain
  root_generation
  root_profile_ref, root_profile_sha256
  witness_subject_ref
  observations[]:
    {key_id, channel_kind = in_person_display | printed_fingerprint |
       hardware_token_display,
     channel_endpoint_identity_sha256,
     displayed_fingerprint_sha256,
     observer_subject_ref, observed_at}

OfflineGovernanceWitnessAttestationPayloadV1
  attestation_id
  domain_separator = mirakan-offline-governance-genesis-witness-v1
  root_profile_ref, root_profile_sha256
  root_keyset_sha256
  observed_fingerprint_manifest_ref, observed_fingerprint_manifest_sha256
  verification_channel_manifest_ref, verification_channel_manifest_sha256
  source_evidence[]: OfflineGovernanceAssuranceEvidenceBindingV1
  witness_subject_ref
  witness_public_key_ref, witness_public_key_sha256
  attested_at

OfflineGovernanceWitnessAttestationV1
  payload: OfflineGovernanceWitnessAttestationPayloadV1
  algorithm = ecdsa-p256-sha256
  format = p1363-fixed-64-low-s
  signature_base64url_no_padding

OfflineGovernanceGenesisCeremonyRecordV1
  ceremony_id
  trust_domain
  root_generation = 1
  root_profile_ref, root_profile_sha256
  root_keyset_sha256
  purpose_quorums_sha256
  verification_channel_manifest_ref, verification_channel_manifest_sha256
  witness_attestations[]: {attestation_ref, attestation_sha256}
  completed_at

OfflineGovernanceRootRotationPayloadV1
  rotation_id
  previous_root_profile_ref, previous_root_profile_sha256
  next_root_profile_ref, next_root_profile_sha256
  rotation_policy_ref, rotation_policy_sha256
  purpose_set_change_manifest_ref: null | content-addressed ref
  purpose_set_change_manifest_sha256: null | lowercase hex 64
  assurance_static_policy_change_manifest_ref: null | content-addressed ref
  assurance_static_policy_change_manifest_sha256: null | lowercase hex 64
  verifier_upgrade_authorization_ref: null | content-addressed ref
  verifier_upgrade_authorization_sha256: null | lowercase hex 64
  ceremony_evidence[]: OfflineGovernanceEvidenceBindingV1
  overlap_starts_at
  overlap_ends_at
  rotation_activated_at
  rollback_deadline

OfflineGovernanceRootRotationV1
  payload: OfflineGovernanceRootRotationPayloadV1
  old_root_threshold_signed_record: OfflineGovernanceThresholdSignedRecordV1
  new_root_threshold_signed_record: OfflineGovernanceThresholdSignedRecordV1

OfflineGovernanceRootRecoveryPayloadV1
  recovery_id
  recovery_mode = partial_compromise | total_compromise
  previous_root_head_ref, previous_root_head_sha256, previous_root_generation
  previous_root_profile_ref, previous_root_profile_sha256
  next_root_profile_ref, next_root_profile_sha256
  recovery_policy_ref, recovery_policy_sha256
  source_global_revocation_super_head_ref, source_global_revocation_super_head_sha256
  source_global_revocation_ledger_ref, source_global_revocation_ledger_sha256
  incident_ref, incident_sha256
  incident_kind
  compromise_effective_at
  independent_recovery_custodian_subject_refs[]
  cooling_off_started_at
  recovery_activated_at
  post_recovery_audits[]: OfflineGovernanceEvidenceBindingV1

OfflineGovernanceRootRecoveryV1
  payload: OfflineGovernanceRootRecoveryPayloadV1
  uncompromised_root_threshold_signed_record: null | OfflineGovernanceThresholdSignedRecordV1
  recovery_custodian_threshold_signed_record: OfflineGovernanceThresholdSignedRecordV1
  new_root_threshold_signed_record: OfflineGovernanceThresholdSignedRecordV1

OfflineGovernanceRootHeadPayloadV1
  head_id
  trust_domain
  generation: positive safe integer
  root_profile_ref
  root_profile_sha256
  activation_kind = genesis | rotation | recovery
  activation_record_ref
  activation_record_sha256
  root_revocation_genesis_head_ref
  root_revocation_genesis_head_sha256
  previous_head_ref: null | content-addressed ref
  previous_head_sha256: null | lowercase hex 64
  activated_at

OfflineGovernanceRootHeadV1
  payload: OfflineGovernanceRootHeadPayloadV1
  threshold_signed_record: OfflineGovernanceThresholdSignedRecordV1

OfflineGovernanceRootSecurityIncidentPayloadV1
  incident_id
  status = confirmed
  confirmation_kind = root_quorum | recovery_custodian_quorum
  incident_kind = key_compromise | custody_loss | policy_violation |
    threshold_loss | total_root_compromise
  affected_root_head_ref, affected_root_head_sha256, affected_root_generation
  affected_key_ids[]
  evidence[]: OfflineGovernanceEvidenceBindingV1
  detected_at
  confirmed_at
  compromise_effective_at

OfflineGovernanceRootSecurityIncidentV1
  payload: OfflineGovernanceRootSecurityIncidentPayloadV1
  confirmation_signed_record: OfflineGovernanceThresholdSignedRecordV1

OfflineGovernanceRootRevocationSnapshotPayloadV1
  snapshot_id
  trust_domain
  root_generation: positive safe integer
  root_profile_ref, root_profile_sha256
  sequence: positive safe integer
  previous_snapshot_ref: null | content-addressed ref
  previous_snapshot_sha256: null | lowercase hex 64
  entries[]:
    {key_id, effective_at,
     reason_code = key_compromise | custody_loss | policy_violation,
     incident_ref, incident_sha256}
  recorded_at

OfflineGovernanceRootRevocationSnapshotV1
  payload: OfflineGovernanceRootRevocationSnapshotPayloadV1
  threshold_signed_record: OfflineGovernanceThresholdSignedRecordV1

OfflineGovernanceRootRevocationHeadPayloadV1
  head_id
  trust_domain
  root_generation: positive safe integer
  sequence: positive safe integer
  previous_head_ref: null | content-addressed ref
  previous_head_sha256: null | lowercase hex 64
  snapshot_ref, snapshot_sha256
  recorded_at

OfflineGovernanceRootRevocationHeadV1
  payload: OfflineGovernanceRootRevocationHeadPayloadV1
  threshold_signed_record: OfflineGovernanceThresholdSignedRecordV1

OfflineGovernanceGlobalRootRevocationLedgerPayloadV1
  ledger_id
  sequence: positive safe integer
  previous_ledger_ref: null | content-addressed ref
  previous_ledger_sha256: null | lowercase hex 64
  authorizing_root_head_ref, authorizing_root_head_sha256, authorizing_root_generation
  entries[]:
    {target_kind = root_key,
     target_root_generation, key_id,
     issuer_subject_ref: null, issuer_public_key_spki_sha256: null,
     effective_at, recorded_at,
     reason_code = key_compromise | custody_loss | policy_violation |
       superseded_generation,
     incident_ref, incident_sha256}
    | {target_kind = assurance_issuer,
       target_root_generation: null, key_id: null,
       issuer_subject_ref, issuer_public_key_spki_sha256,
       effective_at, recorded_at,
       reason_code = issuer_compromise | issuer_distrust | policy_violation,
       incident_ref, incident_sha256}
  recorded_at

OfflineGovernanceGlobalRootRevocationLedgerV1
  payload: OfflineGovernanceGlobalRootRevocationLedgerPayloadV1
  threshold_signed_record: OfflineGovernanceThresholdSignedRecordV1

OfflineGovernanceGlobalRootRevocationSuperHeadPayloadV1
  head_id
  sequence: positive safe integer
  previous_head_ref: null | content-addressed ref
  previous_head_sha256: null | lowercase hex 64
  ledger_ref, ledger_sha256
  authorizing_root_head_ref, authorizing_root_head_sha256, authorizing_root_generation
  recorded_at

OfflineGovernanceGlobalRootRevocationSuperHeadV1
  payload: OfflineGovernanceGlobalRootRevocationSuperHeadPayloadV1
  threshold_signed_record: OfflineGovernanceThresholdSignedRecordV1

BootstrapVerifierProfileV1
  verifier_profile_id
  verifier_core_sha256
  source_ref, source_sha256
  binary_ref, binary_sha256
  build_toolchain_ref, build_toolchain_sha256
  supported_offline_governance_schema_catalog_ref
  supported_offline_governance_schema_catalog_sha256
  known_answer_vector_manifest_ref, known_answer_vector_manifest_sha256
  negative_vector_manifest_ref, negative_vector_manifest_sha256
  supplier_subject_ref
  independent_verification_evidence[]: OfflineGovernanceAssuranceEvidenceBindingV1

OfflineGovernanceSchemaCatalogV1
  format_major = 1
  validation_root_schema_ids[]
  policy_bound_schema_ids[]
  members[]:
    {schema_id, schema_ref, schema_sha256,
     signature_slots[]:
       {signature_slot_id, branch_id, wrapper_field_path,
        signed_record_purpose, issued_at_source_kind,
        issued_at_field_path, revocation_context_kind,
        revocation_context_field_path, authority_source_kind}}

OfflineGovernanceDualVerifierComparisonManifestV1
  source_verifier_profile_ref, source_verifier_profile_sha256
  destination_verifier_profile_ref, destination_verifier_profile_sha256
  source_known_answer_vector_manifest_ref
  source_known_answer_vector_manifest_sha256
  source_negative_vector_manifest_ref
  source_negative_vector_manifest_sha256
  destination_known_answer_vector_manifest_ref
  destination_known_answer_vector_manifest_sha256
  destination_negative_vector_manifest_ref
  destination_negative_vector_manifest_sha256
  source_verifier_on_source_suite_result_root_sha256
  destination_verifier_on_source_suite_result_root_sha256
  destination_verifier_on_destination_suite_result_root_sha256
  source_suite_result_equivalence = passed
  destination_suite_result = passed
  executed_at

OfflineGovernanceVerifierUpgradeAuthorizationPayloadV1
  source_root_profile_ref, source_root_profile_sha256
  source_verifier_profile_ref, source_verifier_profile_sha256
  source_schema_catalog_ref, source_schema_catalog_sha256
  destination_root_profile_ref, destination_root_profile_sha256
  destination_verifier_profile_ref, destination_verifier_profile_sha256
  destination_schema_catalog_ref, destination_schema_catalog_sha256
  dual_verifier_comparison_manifest_ref
  dual_verifier_comparison_manifest_sha256
  candidate_reproduction_evidence[]: OfflineGovernanceAssuranceEvidenceBindingV1
  issued_at
  valid_until

OfflineGovernanceVerifierUpgradeAuthorizationV1
  payload: OfflineGovernanceVerifierUpgradeAuthorizationPayloadV1
  old_root_threshold_signed_record: OfflineGovernanceThresholdSignedRecordV1
  new_root_threshold_signed_record: OfflineGovernanceThresholdSignedRecordV1

BootstrapSchemaMaterializationPlanV1
  plan_id
  local_schema_catalog_ref, local_schema_catalog_sha256
  task_schema_sets[]:
    {task_id = task-0 | task-2, schema_ids[]}
  generated_at

AuthorityBindingDerivationPolicyV1
  policy_id
  policy_kind = subject_ref | assignment_scope_sha256
  input_schema_id
  canonical_input_field_paths[]
  algorithm = exact_ref_field_v1 | catalog_fixed_ref_v1 | rfc8785-jcs-sha256-v1
  fixed_output_ref: null | canonical subject ref
  output_urn_prefix: null | canonical URN prefix

AuthorityBindingSourceCatalogV1
  catalog_id
  format_major = 1
  revision: positive safe integer
  members[]:
    {schema_id, schema_sha256, signature_slot_id, branch_id,
     signed_record_purpose,
     issued_at_source_kind = payload_field | threshold_subject_authenticated_clock,
     issued_at_field_path: null | canonical field path,
     revocation_context_kind = payload_field | source_trust_closure |
       destination_trust_closure | atomic_candidate_trust_closure |
       offline_governance_signer_generation | construction_authorization_snapshot,
     revocation_context_field_path: null | canonical field path,
     authority_source_kind = trust_registry | offline_root_profile_purpose_quorum |
       recovery_policy_custodian_quorum | construction_authorization_executor_key,
     role_ref: null | canonical Role ref,
     subject_derivation_policy_ref: null | content-addressed ref,
     subject_derivation_policy_sha256: null | lowercase hex 64,
     scope_derivation_policy_ref: null | content-addressed ref,
     scope_derivation_policy_sha256: null | lowercase hex 64}

AuthorityBindingSourceCatalogHeadPayloadV1
  head_id
  sequence: positive safe integer
  previous_head_ref: null | content-addressed ref
  previous_head_sha256: null | lowercase hex 64
  catalog_ref, catalog_sha256
  authorization_kind = offline_root_genesis | approved_change |
    root_backed_trust_recovery | root_recovery |
    recovery_readiness_remediation
  authorization_ref, authorization_sha256
  recorded_at

AuthorityBindingSourceCatalogHeadV1
  payload: AuthorityBindingSourceCatalogHeadPayloadV1
  root_threshold_signed_record: null | OfflineGovernanceThresholdSignedRecordV1
  source_signed_record: null | MirakanSignedRecordV1(purpose=authority_binding_source_catalog_head)
  destination_signed_record: null | MirakanSignedRecordV1(purpose=authority_binding_source_catalog_head)

OfflineGovernanceRecoveryReadinessPayloadV1
  readiness_id
  root_head_ref, root_head_sha256, root_generation
  recovery_policy_ref, recovery_policy_sha256
  remediates_invalidation_ref: null | content-addressed ref
  remediates_invalidation_sha256: null | lowercase hex 64
  drill_evidence[]: OfflineGovernanceEvidenceBindingV1
  recovered_public_state_hash
  secrets_exposed = false
  issued_at
  valid_until

OfflineGovernanceRecoveryReadinessReceiptV1
  payload: OfflineGovernanceRecoveryReadinessPayloadV1
  threshold_signed_record: OfflineGovernanceThresholdSignedRecordV1

OfflineGovernanceAssuranceIssuerRevocationDetectionSubjectV1
  detection_source_kind = vendor_signed_revocation |
    independent_offline_assessment
  trusted_issuer_manifest_ref, trusted_issuer_manifest_sha256
  affected_issuer_subject_ref, affected_issuer_public_key_spki_sha256
  source_trust_registry_head_ref: null | content-addressed ref
  source_trust_registry_head_sha256: null | lowercase hex 64
  source_trust_registry_closure_ref: null | content-addressed ref
  source_trust_registry_closure_sha256: null | lowercase hex 64
  revoked_vendor_observation_ref: null | content-addressed ref
  revoked_vendor_observation_sha256: null | lowercase hex 64
  revoked_vendor_response_ref: null | content-addressed ref
  revoked_vendor_response_sha256: null | lowercase hex 64
  independent_assessment_evidence[]: OfflineGovernanceEvidenceBindingV1
  global_revocation_reason = issuer_compromise | issuer_distrust |
    policy_violation
  assurance_failure_effective_at
  detected_at

OfflineGovernanceAssuranceIssuerRevocationDetectionPayloadV1
  evidence_kind = assurance_issuer_revocation_detection
  subject_projection_schema_id = urn:mirakan:schema:offline-governance:assurance-issuer-revocation-detection-subject:v1
  subject_projection: OfflineGovernanceAssuranceIssuerRevocationDetectionSubjectV1
  subject_sha256
  status = passed
  finding = revoked
  completed_at
  valid_until

OfflineGovernanceAssuranceIssuerRevocationDetectionV1
  payload: OfflineGovernanceAssuranceIssuerRevocationDetectionPayloadV1

OfflineGovernanceAssuranceIssuerFailStopPayloadV1
  source_root_head_ref, source_root_head_sha256, source_root_generation
  source_root_revocation_head_ref, source_root_revocation_head_sha256
  source_root_revocation_snapshot_ref, source_root_revocation_snapshot_sha256
  source_global_root_revocation_super_head_ref
  source_global_root_revocation_super_head_sha256
  source_recovery_readiness_head_ref, source_recovery_readiness_head_sha256
  trusted_issuer_manifest_ref, trusted_issuer_manifest_sha256
  affected_issuer_subject_ref, affected_issuer_public_key_spki_sha256
  detection_ref, detection_sha256
  readiness_incident_ref, readiness_incident_sha256
  candidate_global_root_revocation_super_head_ref
  candidate_global_root_revocation_super_head_sha256
  candidate_recovery_readiness_head_ref
  candidate_recovery_readiness_head_sha256
  effective_at
  recorded_at

OfflineGovernanceAssuranceIssuerFailStopV1
  payload: OfflineGovernanceAssuranceIssuerFailStopPayloadV1
  source_root_threshold_signed_record: OfflineGovernanceThresholdSignedRecordV1

OfflineGovernanceRecoveryReadinessIncidentPayloadV1
  incident_id
  status = confirmed
  signature_authority = root_profile_quorum |
    recovery_custodian_quorum
  incident_kind = drill_evidence_invalid | secrets_exposure_discovered |
    recovered_state_mismatch | custody_assurance_invalid
  target_readiness_head_ref, target_readiness_head_sha256
  target_readiness_receipt_ref, target_readiness_receipt_sha256
  affected_root_head_ref, affected_root_head_sha256, affected_root_generation
  affected_assurance_issuer_subject_ref: null | canonical subject ref
  affected_assurance_issuer_public_key_spki_sha256: null | lowercase hex 64
  evidence[]: OfflineGovernanceEvidenceBindingV1
  assurance_failure_effective_at
  detected_at
  confirmed_at

OfflineGovernanceRecoveryReadinessIncidentV1
  payload: OfflineGovernanceRecoveryReadinessIncidentPayloadV1
  threshold_signed_record: OfflineGovernanceThresholdSignedRecordV1

OfflineGovernanceRecoveryReadinessInvalidationPayloadV1
  invalidation_id
  invalidation_kind = readiness_assurance | trust_authority |
    root_authority
  signature_authority = root_profile_quorum |
    recovery_custodian_quorum
  target_readiness_head_ref, target_readiness_head_sha256
  target_readiness_receipt_ref, target_readiness_receipt_sha256
  root_head_ref, root_head_sha256, root_generation
  incident_ref, incident_sha256
  global_quarantine_super_head_ref: null | content-addressed ref
  global_quarantine_super_head_sha256: null | lowercase hex 64
  reason_code = drill_evidence_invalid | secrets_exposure_discovered |
    recovered_state_mismatch | custody_assurance_invalid |
    trust_authority_compromise | root_authority_compromise
  effective_at
  recorded_at

OfflineGovernanceRecoveryReadinessInvalidationV1
  payload: OfflineGovernanceRecoveryReadinessInvalidationPayloadV1
  threshold_signed_record: OfflineGovernanceThresholdSignedRecordV1

OfflineGovernanceRecoveryReadinessHeadPayloadV1
  head_id
  sequence: positive safe integer
  previous_head_ref: null | content-addressed ref
  previous_head_sha256: null | lowercase hex 64
  root_head_ref, root_head_sha256, root_generation
  status = valid | quarantined
  signature_authority = root_profile_quorum |
    recovery_custodian_quorum
  readiness_receipt_ref, readiness_receipt_sha256
  invalidation_ref: null | content-addressed ref
  invalidation_sha256: null | lowercase hex 64
  recorded_at

OfflineGovernanceRecoveryReadinessHeadV1
  payload: OfflineGovernanceRecoveryReadinessHeadPayloadV1
  threshold_signed_record: OfflineGovernanceThresholdSignedRecordV1

OfflineGovernanceRecoveryReadinessRemediationAuthorizationPayloadV1
  remediation_mode = same_generation | assurance_rotation
  source_root_head_ref, source_root_head_sha256, source_root_generation
  source_root_revocation_head_ref, source_root_revocation_head_sha256
  source_root_revocation_snapshot_ref, source_root_revocation_snapshot_sha256
  source_global_root_revocation_super_head_ref
  source_global_root_revocation_super_head_sha256
  source_global_root_revocation_ledger_ref
  source_global_root_revocation_ledger_sha256
  source_recovery_readiness_head_ref, source_recovery_readiness_head_sha256
  recovery_readiness_incident_ref, recovery_readiness_incident_sha256
  recovery_readiness_invalidation_ref, recovery_readiness_invalidation_sha256
  source_trust_registry_head_ref, source_trust_registry_head_sha256
  source_trust_registry_closure_ref, source_trust_registry_closure_sha256
  source_authority_catalog_head_ref, source_authority_catalog_head_sha256
  source_authority_catalog_ref, source_authority_catalog_sha256
  root_rotation_ref: null | content-addressed ref
  root_rotation_sha256: null | lowercase hex 64
  destination_root_head_ref, destination_root_head_sha256, destination_root_generation
  destination_root_revocation_head_ref, destination_root_revocation_head_sha256
  destination_root_revocation_snapshot_ref, destination_root_revocation_snapshot_sha256
  destination_global_root_revocation_super_head_ref
  destination_global_root_revocation_super_head_sha256
  destination_global_root_revocation_ledger_ref
  destination_global_root_revocation_ledger_sha256
  destination_recovery_readiness_receipt_ref
  destination_recovery_readiness_receipt_sha256
  destination_recovery_readiness_head_ref
  destination_recovery_readiness_head_sha256
  destination_trust_registry_closure_ref
  destination_trust_registry_closure_sha256
  destination_authority_catalog_ref, destination_authority_catalog_sha256
  issued_at
  valid_until

OfflineGovernanceRecoveryReadinessRemediationAuthorizationV1
  payload: OfflineGovernanceRecoveryReadinessRemediationAuthorizationPayloadV1
  root_threshold_signed_record: OfflineGovernanceThresholdSignedRecordV1

ControlPlaneConstructionAuthorizationPayloadV1
  authorization_id
  root_head_ref
  root_head_sha256
  root_generation
  root_profile_ref
  root_profile_sha256
  bootstrap_verifier_profile_ref
  bootstrap_verifier_profile_sha256
  offline_governance_schema_catalog_ref
  offline_governance_schema_catalog_sha256
  local_schema_catalog_ref
  local_schema_catalog_sha256
  bootstrap_schema_materialization_plan_ref
  bootstrap_schema_materialization_plan_sha256
  authority_binding_source_catalog_ref
  authority_binding_source_catalog_sha256
  authority_binding_source_catalog_head_ref
  authority_binding_source_catalog_head_sha256
  trust_bootstrap_state = expected_empty_genesis | current_closure_reuse
  current_trust_registry_head_ref: null | content-addressed ref
  current_trust_registry_head_sha256: null | lowercase hex 64
  current_trust_registry_closure_ref: null | content-addressed ref
  current_trust_registry_closure_sha256: null | lowercase hex 64
  authorized_base_git_tree_id
  control_plane_design_sha256
  control_plane_implementation_plan_sha256
  allowed_task_ids[] =
    [task-0, task-1, task-2, task-3, task-4, task-5, task-6,
     task-7, task-8, task-9, task-10, task-10a, task-10b]
  task_artifact_scopes[]:
    {task_id, allowed_roots[]}
  exceptional_operations[] =
    [product_operational_genesis, control_plane_bootstrap_completion]
  construction_executor_subject_ref
  construction_executor_public_key_ref
  construction_executor_public_key_sha256
  issued_at
  valid_until
  root_revocation_snapshot_ref
  root_revocation_snapshot_sha256
  root_revocation_head_ref
  root_revocation_head_sha256
  global_root_revocation_super_head_ref
  global_root_revocation_super_head_sha256
  global_root_revocation_ledger_ref
  global_root_revocation_ledger_sha256
  recovery_readiness_head_ref
  recovery_readiness_head_sha256

ControlPlaneConstructionAuthorizationV1
  payload: ControlPlaneConstructionAuthorizationPayloadV1
  threshold_signed_record: OfflineGovernanceThresholdSignedRecordV1

TrustProvisioningReceiptPayloadV1
  receipt_id
  provisioning_kind = construction_genesis | current_closure_reuse
  construction_authorization_ref
  construction_authorization_sha256
  current_trust_registry_head_ref: null | content-addressed ref
  current_trust_registry_head_sha256: null | lowercase hex 64
  current_trust_registry_closure_ref: null | content-addressed ref
  current_trust_registry_closure_sha256: null | lowercase hex 64
  root_head_ref
  root_head_sha256
  root_generation
  root_profile_ref
  root_profile_sha256
  required_authority_bindings[]:
    {schema_id, signature_slot_id, subject_ref, role_ref,
     signed_record_purpose, assignment_scope_sha256}
  identity_registry_ref, identity_registry_sha256
  role_registry_ref, role_registry_sha256
  role_assignment_registry_ref, role_assignment_registry_sha256
  public_key_registry_ref, public_key_registry_sha256
  revocation_genesis_ref, revocation_genesis_sha256
  revocation_current_ref, revocation_current_sha256
  policy_bootstrap_config_ref, policy_bootstrap_config_sha256
  rotation_policy_ref, rotation_policy_sha256
  recovery_readiness_receipt_ref, recovery_readiness_receipt_sha256
  recovery_readiness_head_ref, recovery_readiness_head_sha256
  authority_binding_source_catalog_ref, authority_binding_source_catalog_sha256
  authority_binding_source_catalog_head_ref, authority_binding_source_catalog_head_sha256
  secrets_included = false
  issued_at
  root_revocation_snapshot_ref
  root_revocation_snapshot_sha256
  root_revocation_head_ref
  root_revocation_head_sha256
  global_root_revocation_super_head_ref
  global_root_revocation_super_head_sha256
  global_root_revocation_ledger_ref
  global_root_revocation_ledger_sha256

TrustProvisioningReceiptV1
  payload: TrustProvisioningReceiptPayloadV1
  threshold_signed_record: OfflineGovernanceThresholdSignedRecordV1

TrustRegistryClosureV1
  closure_id
  root_head_ref, root_head_sha256
  root_profile_ref, root_profile_sha256
  root_revocation_head_ref, root_revocation_head_sha256
  root_revocation_snapshot_ref, root_revocation_snapshot_sha256
  global_root_revocation_super_head_ref, global_root_revocation_super_head_sha256
  global_root_revocation_ledger_ref, global_root_revocation_ledger_sha256
  recovery_readiness_head_ref, recovery_readiness_head_sha256
  recovery_readiness_receipt_ref, recovery_readiness_receipt_sha256
  authority_binding_source_catalog_ref, authority_binding_source_catalog_sha256
  identity_registry_ref, identity_registry_sha256
  role_registry_ref, role_registry_sha256
  role_assignment_registry_ref, role_assignment_registry_sha256
  public_key_registry_ref, public_key_registry_sha256
  revocation_current_ref, revocation_current_sha256
  policy_config_ref, policy_config_sha256

IdentityRegistryV1
  registry_id = urn:mirakan:registry:trust:identity:v1, format_major = 1, revision
  entries[]: {subject_ref, subject_kind = human | service, status = active | disabled, created_at}

AuthorityRoleRegistryV1
  registry_id = urn:mirakan:registry:trust:authority-role:v1, format_major = 1, revision
  entries[]: {role_ref, permission_ids[], allowed_signed_record_purposes[], independence_class}

AuthorityRoleAssignmentRegistryV1
  registry_id = urn:mirakan:registry:trust:authority-role-assignment:v1,
  format_major = 1, revision
  entries[]:
    {assignment_id, subject_ref, role_ref, assignment_scope_sha256,
     valid_from, expires_at, revoked_at}

AuthorityPublicKeyRegistryV1
  registry_id = urn:mirakan:registry:trust:authority-public-key:v1,
  format_major = 1, revision
  entries[]:
    {key_id, subject_ref, role_ref, public_key_spki_ref, public_key_spki_sha256,
     algorithm, format, allowed_signed_record_purposes[],
     valid_from, expires_at, revoked_at}

AuthorityRevocationRegistryV1
  registry_id = urn:mirakan:registry:trust:authority-revocation:v1,
  format_major = 1, revision
  entries[]:
    {revocation_id, subject_kind = identity | role | assignment | key | signed_record,
     subject_ref, effective_at, reason_code,
     recovery_record_ref: null | typed content ref,
     recovery_record_sha256: null | lowercase hex 64,
     governance_incident_ref: null | content-addressed ref,
     governance_incident_sha256: null | lowercase hex 64}

GovernancePolicyConfigRegistryV1
  registry_id = urn:mirakan:registry:trust:governance-policy-config:v1,
  format_major = 1, revision
  entries[]: {policy_id, policy_schema_id, policy_ref, policy_sha256}

TrustRegistryChangeEvidenceBindingV1
  evidence_kind
  evidence_ref: content-addressed ref
  evidence_sha256: lowercase hex 64

TrustRegistryChangeEvidencePolicyV1
  policy_id
  effect_requirements[]:
    {privilege_effect = none | reduced | expanded,
     independence_effect = none | strengthened | weakened,
     required_evidence_kinds[], required_risk_evidence_kinds[]}
  evidence_kind_schema_bindings[]:
    {evidence_kind, evidence_schema_id, subject_projection_schema_id,
     max_age_seconds: positive safe integer}
  independent_r4_approval_invariant:
    {evidence_kind = trust_change_independent_r4_approval,
     evidence_schema_id = urn:mirakan:schema:architecture:trust-registry-change-independent-r4-approval-evidence:v1,
     subject_projection_schema_id = urn:mirakan:schema:architecture:trust-registry-change-independent-r4-subject:v1,
     signed_record_purpose = trust_registry_change_independent_r4_approval,
     signer_role_ref = role.trust-registry-change-independent-r4-approver,
     required_independence_class = independent_human_r4}

TrustRegistryChangeIndependentR4SubjectV1
  trust_registry_change_manifest_ref, trust_registry_change_manifest_sha256
  source_trust_registry_closure_ref, source_trust_registry_closure_sha256
  destination_trust_registry_closure_ref, destination_trust_registry_closure_sha256
  source_authority_binding_source_catalog_ref, source_authority_binding_source_catalog_sha256
  destination_authority_binding_source_catalog_ref, destination_authority_binding_source_catalog_sha256
  expanded_or_weakened_change_keys[]:
    {change_domain = registry_entry | authority_binding_member,
     logical_key_sha256, privilege_effect, independence_effect}

TrustRegistryChangeIndependentR4ApprovalEvidencePayloadV1
  evidence_kind = trust_change_independent_r4_approval
  subject_projection: TrustRegistryChangeIndependentR4SubjectV1
  status = passed
  issuer_subject_ref
  issuer_role_ref = role.trust-registry-change-independent-r4-approver
  completed_at
  revocation_snapshot_ref

TrustRegistryChangeIndependentR4ApprovalEvidenceV1
  payload: TrustRegistryChangeIndependentR4ApprovalEvidencePayloadV1
  signed_record: MirakanSignedRecordV1(purpose=trust_registry_change_independent_r4_approval)

TrustRegistryChangeManifestV1
  manifest_id
  source_trust_registry_closure_ref, source_trust_registry_closure_sha256
  destination_trust_registry_closure_ref, destination_trust_registry_closure_sha256
  source_authority_binding_source_catalog_ref, source_authority_binding_source_catalog_sha256
  destination_authority_binding_source_catalog_ref, destination_authority_binding_source_catalog_sha256
  change_evidence_policy_ref, change_evidence_policy_sha256
  closure_member_changes[]:
    {member_kind, source_ref, source_sha256, destination_ref, destination_sha256,
     change_kind = retained | replaced}
  registry_entry_changes[]:
    {registry_kind, logical_id, change_kind = added | removed | retained | modified,
     source_entry_sha256: null | lowercase hex 64,
     destination_entry_sha256: null | lowercase hex 64,
     privilege_effect = none | reduced | expanded,
     independence_effect = none | strengthened | weakened,
     evidence[]: TrustRegistryChangeEvidenceBindingV1,
     risk_evidence[]: TrustRegistryChangeEvidenceBindingV1}
  authority_binding_member_changes[]:
    {schema_id, signature_slot_id, signed_record_purpose,
     change_kind = added | removed | retained | modified,
     source_member_sha256: null | lowercase hex 64,
     destination_member_sha256: null | lowercase hex 64,
     privilege_effect = none | reduced | expanded,
     independence_effect = none | strengthened | weakened,
     evidence[]: TrustRegistryChangeEvidenceBindingV1,
     risk_evidence[]: TrustRegistryChangeEvidenceBindingV1}
  generated_at

TrustRegistryChangeApprovalPayloadV1
  approval_id
  source_trust_registry_head_ref, source_trust_registry_head_sha256
  source_trust_registry_closure_ref, source_trust_registry_closure_sha256
  destination_trust_registry_closure_ref, destination_trust_registry_closure_sha256
  source_authority_binding_source_catalog_head_ref, source_authority_binding_source_catalog_head_sha256
  destination_authority_binding_source_catalog_ref, destination_authority_binding_source_catalog_sha256
  change_manifest_ref, change_manifest_sha256
  change_evidence_policy_ref, change_evidence_policy_sha256
  evidence[]: TrustRegistryChangeEvidenceBindingV1
  approver_subject_ref, approval_authority_ref
  issued_at
  revocation_snapshot_ref

TrustRegistryChangeApprovalV1
  payload: TrustRegistryChangeApprovalPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=trust_registry_change_approval)

TrustRegistryRecoveryAuthorizationPayloadV1
  authorization_id
  source_trust_registry_head_ref, source_trust_registry_head_sha256
  source_trust_registry_closure_ref, source_trust_registry_closure_sha256
  source_status = quarantined
  incident_ref, incident_sha256
  completed_root_recovery_ref, completed_root_recovery_sha256
  destination_root_head_ref, destination_root_head_sha256, destination_root_generation
  destination_trust_registry_closure_ref, destination_trust_registry_closure_sha256
  destination_authority_binding_source_catalog_ref
  destination_authority_binding_source_catalog_sha256
  issued_at
  root_revocation_head_ref, root_revocation_head_sha256
  root_revocation_snapshot_ref, root_revocation_snapshot_sha256
  global_root_revocation_super_head_ref, global_root_revocation_super_head_sha256
  global_root_revocation_ledger_ref, global_root_revocation_ledger_sha256

TrustRegistryRecoveryAuthorizationV1
  payload: TrustRegistryRecoveryAuthorizationPayloadV1
  new_root_threshold_signed_record: OfflineGovernanceThresholdSignedRecordV1

ControlPlaneGovernanceEvidenceBindingV1
  evidence_kind
  evidence_ref: content-addressed ref
  evidence_sha256: lowercase hex 64

ControlPlaneGovernanceRecoveryPolicyV1
  policy_id = policy.control-plane.governance-recovery.v1
  route_bindings[]:
    {incident_schema_id, incident_kind,
     route = healthy_trust_rebaseline |
       root_backed_trust_then_rebaseline |
       offline_root_then_trust_then_rebaseline,
     required_evidence_kinds[]}
  evidence_kind_schema_bindings[]:
    {evidence_kind, evidence_schema_id, subject_projection_schema_id,
     max_age_seconds: positive safe integer}

TrustRegistryQuarantineIncidentPayloadV1
  incident_kind = trust_registry_closure_compromise |
    trust_head_publisher_compromise | authority_catalog_compromise
  status = confirmed
  governance_recovery_policy_ref, governance_recovery_policy_sha256
  root_head_ref, root_head_sha256, root_generation
  root_revocation_head_ref, root_revocation_head_sha256
  root_revocation_snapshot_ref, root_revocation_snapshot_sha256
  global_root_revocation_super_head_ref, global_root_revocation_super_head_sha256
  global_root_revocation_ledger_ref, global_root_revocation_ledger_sha256
  recovery_readiness_head_ref, recovery_readiness_head_sha256
  affected_trust_registry_head_ref, affected_trust_registry_head_sha256
  affected_trust_registry_closure_ref, affected_trust_registry_closure_sha256
  affected_authority_catalog_head_ref, affected_authority_catalog_head_sha256
  affected_authority_catalog_ref, affected_authority_catalog_sha256
  affected_targets[]:
    {target_kind = trust_closure_member,
     member_kind, target_ref, target_sha256}
    | {target_kind = trust_head_publisher_authority,
       subject_ref, role_ref, assignment_id, key_id,
       public_key_entry_sha256}
    | {target_kind = authority_catalog_member,
       schema_id, signature_slot_id, member_sha256}
  evidence[]: ControlPlaneGovernanceEvidenceBindingV1
  detected_at
  confirmed_at
  compromise_effective_at

TrustRegistryQuarantineIncidentV1
  payload: TrustRegistryQuarantineIncidentPayloadV1
  root_threshold_signed_record: OfflineGovernanceThresholdSignedRecordV1

TrustRegistryQuarantineRecoveryAuthorizationPayloadV1
  governance_recovery_policy_ref, governance_recovery_policy_sha256
  source_trust_registry_head_ref, source_trust_registry_head_sha256
  source_trust_registry_closure_ref, source_trust_registry_closure_sha256
  source_authority_catalog_head_ref, source_authority_catalog_head_sha256
  incident_ref, incident_sha256
  destination_trust_registry_closure_ref, destination_trust_registry_closure_sha256
  destination_authority_catalog_ref, destination_authority_catalog_sha256
  root_head_ref, root_head_sha256, root_generation
  root_revocation_head_ref, root_revocation_head_sha256
  root_revocation_snapshot_ref, root_revocation_snapshot_sha256
  global_root_revocation_super_head_ref, global_root_revocation_super_head_sha256
  global_root_revocation_ledger_ref, global_root_revocation_ledger_sha256
  source_recovery_readiness_head_ref, source_recovery_readiness_head_sha256
  destination_recovery_readiness_receipt_ref
  destination_recovery_readiness_receipt_sha256
  destination_recovery_readiness_head_ref
  destination_recovery_readiness_head_sha256
  issued_at

TrustRegistryQuarantineRecoveryAuthorizationV1
  payload: TrustRegistryQuarantineRecoveryAuthorizationPayloadV1
  root_threshold_signed_record: OfflineGovernanceThresholdSignedRecordV1

TrustRegistryClosureHeadPayloadV1
  head_id
  sequence: positive safe integer
  previous_head_ref: null | content-addressed ref
  previous_head_sha256: null | lowercase hex 64
  trust_registry_closure_ref, trust_registry_closure_sha256
  authorization_kind = construction_genesis | approved_change |
    root_backed_trust_recovery | root_recovery |
    recovery_readiness_remediation
  authorization_ref, authorization_sha256
  signer_handoff = source_only | source_and_destination | destination_only
  recorded_at
  source_revocation_snapshot_ref: null | content-addressed ref
  destination_revocation_snapshot_ref: null | content-addressed ref

TrustRegistryClosureHeadV1
  payload: TrustRegistryClosureHeadPayloadV1
  source_signed_record: null | MirakanSignedRecordV1(purpose=trust_registry_closure_head)
  destination_signed_record: null | MirakanSignedRecordV1(purpose=trust_registry_closure_head)

ControlPlaneConstructionTaskReceiptPayloadV1
  receipt_id
  construction_authorization_ref
  construction_authorization_sha256
  task_id
  previous_task_receipt_ref: null | content-addressed ref
  previous_task_receipt_sha256: null | lowercase hex 64
  input_git_tree_id
  output_git_tree_id
  changed_paths_manifest_ref
  changed_paths_manifest_sha256
  input_manifest_ref, input_manifest_sha256
  output_manifest_ref, output_manifest_sha256
  toolchain_lock_sha256
  result = pass | fail
  started_at
  completed_at

ControlPlaneConstructionTaskReceiptV1
  payload: ControlPlaneConstructionTaskReceiptPayloadV1
  executor_signed_record: ConstructionExecutorSignedRecordV1

ConstructionExecutorSignedRecordV1
  signing_subject: ConstructionExecutorSigningSubjectV1
  signature_base64url_no_padding

ConstructionExecutorSigningSubjectV1
  domain_separator = mirakan-control-plane-construction-executor-v1
  subject_schema_id =
    urn:mirakan:schema:architecture:construction-task-receipt-payload:v1 |
    urn:mirakan:schema:architecture:control-plane-legacy-inventory-payload:v1 |
    urn:mirakan:schema:architecture:control-plane-migration-seed-payload:v1 |
    urn:mirakan:schema:architecture:identity-migration-manifest-payload:v1 |
    urn:mirakan:schema:architecture:relation-migration-manifest-payload:v1
  subject_ref
  subject_sha256
  purpose = control_plane_construction_task_receipt |
    control_plane_legacy_inventory | control_plane_migration_seed |
    architecture_identity_migration_manifest | architecture_relation_migration_manifest
  signer_subject_ref
  public_key_ref
  public_key_sha256
  algorithm = ecdsa-p256-sha256
  format = p1363-fixed-64-low-s
  issued_at
```

### 6.0.1 Bootstrap trust content-derived ID registry

本表のscopeは直前code blockのpre-baseline Root／Trust／Construction contractだけである。各suffixは表のprojectionからID Field一件だけを除いたRFC 8785 JCS bytesのSHA-256 lowercase hexである。`payload`は完成payload object、`object`は完成direct object、`wrapper`は署名を含む完成wrapperを意味する。本表にないFieldへbootstrap trust規則を適用せず、後段Control Plane／Architecture／migration型は各Owner節のexact ID規則を使い、同prefixを別型へ再利用しない。

| Type／ID Field | Projection | exact URN prefix |
| --- | --- | --- |
| `OfflineGovernanceThresholdSignedRecordV1.record_id` | wrapper | `urn:mirakan:offline-governance-record:sha256:` |
| `OfflineGovernanceRootProfilePayloadV1.root_profile_id` | payload | `urn:mirakan:offline-governance-root-profile:sha256:` |
| `OfflineGovernanceRotationPolicyV1.policy_id` | object | `urn:mirakan:offline-governance-rotation-policy:sha256:` |
| `OfflineGovernancePurposeSetChangeManifestV1.manifest_id` | object | `urn:mirakan:offline-governance-purpose-change:sha256:` |
| `OfflineGovernanceRecoveryPolicyV1.policy_id` | object | `urn:mirakan:offline-governance-recovery-policy:sha256:` |
| `OfflineGovernanceWitnessAttestationPayloadV1.attestation_id` | payload | `urn:mirakan:offline-governance-witness-attestation:sha256:` |
| `OfflineGovernanceGenesisCeremonyRecordV1.ceremony_id` | object | `urn:mirakan:offline-governance-genesis-ceremony:sha256:` |
| `OfflineGovernanceRootRotationPayloadV1.rotation_id` | payload | `urn:mirakan:offline-governance-root-rotation:sha256:` |
| `OfflineGovernanceRootRecoveryPayloadV1.recovery_id` | payload | `urn:mirakan:offline-governance-root-recovery:sha256:` |
| `OfflineGovernanceRootHeadPayloadV1.head_id` | payload | `urn:mirakan:offline-governance-root-head:sha256:` |
| `OfflineGovernanceRootSecurityIncidentPayloadV1.incident_id` | payload | `urn:mirakan:offline-governance-root-security-incident:sha256:` |
| `OfflineGovernanceRootRevocationSnapshotPayloadV1.snapshot_id` | payload | `urn:mirakan:offline-governance-root-revocation-snapshot:sha256:` |
| `OfflineGovernanceRootRevocationHeadPayloadV1.head_id` | payload | `urn:mirakan:offline-governance-root-revocation-head:sha256:` |
| `OfflineGovernanceGlobalRootRevocationLedgerPayloadV1.ledger_id` | payload | `urn:mirakan:offline-governance-global-root-revocation-ledger:sha256:` |
| `OfflineGovernanceGlobalRootRevocationSuperHeadPayloadV1.head_id` | payload | `urn:mirakan:offline-governance-global-root-revocation-super-head:sha256:` |
| `BootstrapVerifierProfileV1.verifier_profile_id` | object | `urn:mirakan:bootstrap-verifier-profile:sha256:` |
| `BootstrapSchemaMaterializationPlanV1.plan_id` | object | `urn:mirakan:bootstrap-schema-materialization-plan:sha256:` |
| `AuthorityBindingDerivationPolicyV1.policy_id` | object | `urn:mirakan:authority-binding-derivation-policy:sha256:` |
| `AuthorityBindingSourceCatalogV1.catalog_id` | object | `urn:mirakan:authority-binding-source-catalog:sha256:` |
| `AuthorityBindingSourceCatalogHeadPayloadV1.head_id` | payload | `urn:mirakan:authority-binding-source-catalog-head:sha256:` |
| `OfflineGovernanceRecoveryReadinessPayloadV1.readiness_id` | payload | `urn:mirakan:offline-governance-recovery-readiness:sha256:` |
| `OfflineGovernanceRecoveryReadinessIncidentPayloadV1.incident_id` | payload | `urn:mirakan:offline-governance-recovery-readiness-incident:sha256:` |
| `OfflineGovernanceRecoveryReadinessInvalidationPayloadV1.invalidation_id` | payload | `urn:mirakan:offline-governance-recovery-readiness-invalidation:sha256:` |
| `OfflineGovernanceRecoveryReadinessHeadPayloadV1.head_id` | payload | `urn:mirakan:offline-governance-recovery-readiness-head:sha256:` |
| `ControlPlaneConstructionAuthorizationPayloadV1.authorization_id` | payload | `urn:mirakan:control-plane-construction-authorization:sha256:` |
| `TrustProvisioningReceiptPayloadV1.receipt_id` | payload | `urn:mirakan:trust-provisioning-receipt:sha256:` |
| `TrustRegistryClosureV1.closure_id` | object | `urn:mirakan:trust-registry-closure:sha256:` |
| `TrustRegistryChangeManifestV1.manifest_id` | object | `urn:mirakan:trust-registry-change-manifest:sha256:` |
| `TrustRegistryChangeApprovalPayloadV1.approval_id` | payload | `urn:mirakan:trust-registry-change-approval:sha256:` |
| `TrustRegistryRecoveryAuthorizationPayloadV1.authorization_id` | payload | `urn:mirakan:trust-registry-recovery-authorization:sha256:` |
| `TrustRegistryClosureHeadPayloadV1.head_id` | payload | `urn:mirakan:trust-registry-closure-head:sha256:` |
| `ControlPlaneConstructionTaskReceiptPayloadV1.receipt_id` | payload | `urn:mirakan:control-plane-construction-task-receipt:sha256:` |

Root／Recovery custodianの`key_id`は本表scope内の例外的nested deterministic IDで、`urn:mirakan:key:spki-sha256:<SHA-256(SPKI-DER) lowercase-hex>`に固定し、同一SPKIへ別ID、別SPKIへ同一IDを許さない。`task_id`はschemaのclosed enum、`schema_id`はLocal Schema Catalog member ID、Identity／Role／Assignment等のlogical IDは各Registryのstable-ID規則を使う。Bootstrap trust Content ID RegistryのType／ID pair集合は表の32件とset equalityで、対応するLocal Schema Catalog memberだけのsigned schema extension `x-mirakan-content-id={field,projection,prefix}`ともset equalityにする。後段Ownerのannotationをこの集合のextraと数えず、各Owner節で別検証する。表scope内のmissing／extra／prefix差、top-levelとnested key規則の混同をBootstrap known-answer／negative vectorで拒否する。

Task 0より前の唯一のbinary／ProfileはRoot ceremonyがout-of-band供給する`BootstrapVerifierProfileV1`である。source／binary／Build Toolchain／supported offline governance schema catalog／known-answer／一原因negative vectorの全hashをRoot ProfileとAuthorizationへ束縛し、`independent_verification_evidence[]`が同じ完成Profile projectionについて二つ以上のdistinct issuerから同一pass結果を示すことを人間ceremonyで確認する。Stage OでAuthorizationまで検証した後だけ同binaryをStage Cへ遷移させ、Authorization承認済みfull Local Catalogを追加loadする。構築対象Repositoryのcode、生成予定Policy Service、AI出力だけでAuthorizationを自己検証しない。binary／source mismatch、vector不足、issuer重複、単一供給経路、offline catalog hash差、未承認full catalogではTask 0を開始しない。

Root／Recovery-custodian key custody、Bootstrap Verifier独立再現、Genesis sourceのassuranceはbare refでなく`OfflineGovernanceAssuranceEvidenceBindingV1`だけを使う。Root Profileが束縛する`OfflineGovernanceAssuranceEvidencePolicyV1.kind_bindings[]`のkind集合は`governance_key_custody | bootstrap_verifier_reproduction | genesis_source`とset equality、kind順・重複なしで、各bindingはroot schema ID、consumer projection schema、許可proof kind、out-of-band ceremonyで承認したtrusted issuer manifest、accepted verification profile ref／hash exact set、minimum distinct issuer、positive `max_age_seconds`を閉じる。Evidence bindingのrefは`urn:mirakan:offline-governance-assurance-evidence:sha256:<SHA-256(完成wrapper JCS)>`、隣接hashも同じ完成wrapper bytesとし、この型は論理ID Fieldを持たないため§6.0.1の32件へ追加しない。payloadのkind／projection schema、`status=passed`、issuer、verification profile、`completed_at <= consumer_time < valid_until`、`consumer_time - completed_at <= max_age_seconds`をPolicyとexact一致させる。

`proof_kind=hardware_vendor_chain`はproof refが解決するclosed attestationからtrust-anchor→intermediate→leaf chain、leaf issuer、署名algorithm、attested SPKI DER hash、non-exportable state、custody kind、measurement、attestation timeを検証し、Policyのtrusted issuer manifest内anchorだけを許す。attestation challengeは`{evidence_kind,subject_projection_schema_id,subject_sha256,challenge_nonce_base64url_32,verification_profile_ref,verification_profile_sha256}`のJCS SHA-256、attestation timeはpayload `completed_at`、leaf issuer／SPKIはpayload issuer Fieldとexact一致させる。`independent_verifier_signature`はissuer SPKI DERをread-backし、Policy manifestのactive issuer entry、発行時validity／revocation、algorithm／formatを検証して`UTF-8(JCS(payload))`へ一度だけ署名したproofだけを許す。ceremony nonce journalはissuer＋32-byte nonceをexpected-empty CASで消費し、同一nonce、未知challenge、profile差、時刻載せ替えを拒否する。Root／Recovery custodian Keyごとのcustody Evidenceはexactly oneでsubject projectionを`{authority_kind,trust_domain,root_generation,key_id,public_key_spki_sha256,custodian_subject_ref,custody_kind}`とexact一致させ、`authority_kind=root | recovery_custodian`をschema位置から一意に導出する。Productionでは両key setともhardware proofだけを許し、Recovery custodian subject／SPKI／Evidence issuerをRoot custodian、通常worker、AIから独立させる。Bootstrap reproductionはProfileのsource／binary／toolchain／catalog／known-answer／negative-vector／supplier全ref・hash projection、Genesis sourceはWitness payloadのRoot Profile／keyset／observed fingerprint／verification-channel manifest／witness projectionへ一致させる。各consumer collectionはPolicyのrequired kindとminimum distinct issuerをset equalityで満たし、issuerを対象Custodian、Witness、supplier、construction worker、AIから独立させる。unknown issuer、revoked／expired proof、任意blob、subject差、hash-only、同一issuerの別名、missing／extra Evidenceを拒否する。

Assurance Evidenceのconsumer timeとself-referenceを除くprojectionは次表で固定する。Evidence payload／proof／binding自身、content ref、signature、論理IDをprojectionへ含めない。

| evidence kind／consumer | exact subject projection | consumer time |
|---|---|---|
| `governance_key_custody`／Root Key | `authority_kind=root`とRoot Profileのtrust domain、generation、Key ID、SPKI hash、custodian、custody kind | Root Profile payload `issued_at` |
| `governance_key_custody`／Recovery custodian Key | `authority_kind=recovery_custodian`とRoot Profileのtrust domain／generation、Recovery PolicyのKey ID、SPKI hash、subject、custody kind | Recovery Policyを束縛するRoot Profile payload `issued_at` |
| `bootstrap_verifier_reproduction`／Bootstrap Verifier Profile | source、binary、Build Toolchain、supported schema catalog、known-answer vector、negative vectorの全ref／hashとsupplier subject | Construction Authorization payload `issued_at` |
| `genesis_source`／Genesis Witness | domain separator、Root Profile ref／hash、root keyset hash、observed fingerprint manifest ref／hash、verification-channel manifest ref／hash、witness subject／public key ref／hash、`attested_at` | Witness payload `attested_at` |

各rowで完成wrapperの`proof_schema_id`はPolicyの該当`proof_schema_bindings[]`、payload verification profile ref／hashは`accepted_verification_profiles[]`の一件とexact一致させる。BootstrapとGenesisのrequired Evidenceは各consumer arrayそのものをdistinct issuer thresholdとset equality検証し、Key custodyは各Keyのinline bindingを集約してKey集合とset equality検証する。Root／Recovery custody、Genesis sourceに加えRoot Profileがpre-commitするBootstrap Verifier Profileのreproduction Evidenceも、consumer timeを`RootProfile.issued_at`として`RootProfile.valid_until <= Evidence.valid_until`かつ`RootProfile.valid_until - Evidence.completed_at <= max_age_seconds`を満たす。Construction Authorizationは同じBootstrap Verifier Profile／Evidence pairを束縛し、さらに`Authorization.valid_until <= Evidence.valid_until`かつ`Authorization.valid_until - Evidence.completed_at <= max_age_seconds`を満たす。発行時だけでなく各current authority使用／publication時にcurrent Root Profile、Readiness、issuer revocationとともに再検証する。期限を跨ぐProfile／Authorizationを発行せず、Evidence失効時はcurrent operationを停止してfresh Evidenceを束縛したnew Root ProfileをRotation／Recoveryでcurrent化し、その後にnew Authorizationを要求する。ceremony nonce journalへの初回登録はexpected-unused→同一Evidence ref／hashへ一度だけconsumeし、後続historical／current verifierが同じnonce→同じEvidence mappingをread-only再検証することはreplayと数えない。同じnonceの別Evidence、mapping書換え、二度目のissuanceだけを拒否する。projectionに`independent_verification_evidence[]`または`source_evidence[]`を混ぜる自己参照、kind間Evidence流用、consumer timeの呼出元差替えを拒否する。

Verifier起動は同一binary／Profileの二段階に固定する。Stage OはRoot Profileのoffline catalogだけをloadし、current Root／Revocation／Readiness、Construction Authorization wrapper、同Profile reproduction Evidenceを検証する。Stage CはStage O成功後だけAuthorizationが承認したfull Local Catalog／Materialization Planを同binaryへ追加loadし、Task schemaをcompileする。Stage Oでfull Local Catalogをauthorityにすること、Stage CをAuthorization前に起動すること、別binary／Profileへ暗黙切替することを拒否する。Root Profile consumerは`RootProfile.issued_at`とProfile lifetime、Authorization consumerは`Authorization.issued_at`とAuthorization lifetimeをそれぞれ上記同じEvidenceへ独立評価し、どちらか一方の成功を他方へ流用しない。

`OfflineGovernanceTrustedIssuerManifestV1`、Vendor Chain Policy／Revocation Evidence、offline issuer Revocation Snapshotはいずれも論理IDを持たない完成JCS content-addressed objectで、両Schema Catalogには必要なclosureへ含めるが§6.0.1の32件へ追加しない。Issuer entriesはsubject ref順・重複なし、SPKI hash uniqueとする。Manifestはstable policyだけを持ち、live CRL／OCSP ref／hashをpre-commitしない。hardware vendor entryはtrust anchor／`OfflineGovernanceVendorChainPolicyV1`、responder SPKI、`revocation_mode=vendor_crl | vendor_ocsp`、non-empty `accepted_revocation_parser_profiles[]`を必須、offline independent issuerはanchor／chain／responderをnull、parser profile集合をempty、`revocation_mode=offline_root_ledger`、Root Profile ceremonyで承認した`OfflineGovernanceAssuranceIssuerRevocationSnapshotV1` genesis pairをnon-nullとする。`vendor_crl`のChain Policyは`ocsp_nonce_requirement=not_applicable`、responder revocation method／max lifetimeを両null、`vendor_ocsp`はnonce required、method non-nullとし、`independent_crl`ならmax lifetime null、`nocheck_short_lived`ならmax lifetime positiveにする。proof branch別nullability、maximum response age、OCSP policyをexactに検証する。Vendor X.509のalgorithm集合はRoot governanceのP-256固定Policyから分離し、Production hardware-selection spikeで採用TPM／HSMの実certificate／attestation／CRL／OCSP chainを固定parserで検証して、`VendorSignatureAlgorithmV1`のnon-empty subsetをChain Policyへ承認するまでProduction Root Profileを発行しない。ECDSA P-256／P-384とRSA-PSS SHA-256／384だけをV1候補とし、SHA-1、RSA PKCS#1 v1.5、unknown algorithm、algorithm parameter差は拒否する。例外は別schema revision、Architecture／Security Decision、Target実機Qualificationを要し、既存V1 Policyの自由値では許可しない。

Evidence発行時の`issuance_revocation_evidence_ref/hash`はissuance baselineであり、vendor modeなら同issuer／serialの完成`OfflineGovernanceVendorRevocationEvidenceV1(status=good, revoked_at=null, invalidity_at=null)`、offline modeならManifestがpre-commitしたexact `OfflineGovernanceAssuranceIssuerRevocationSnapshotV1(sequence=1, previous pair=null)`へ解決する。offline snapshotはこのgenesis一件だけを許し、unsigned descendantやin-place更新を禁止する。後発offline issuer失効はGlobal Ledgerだけへ記録する。Vendor response検証はtarget certificate DER、full signed CRLまたはBasicOCSPResponse DER、response signer→intermediate→trusted anchorをanchor込みexactly onceで並べたcertificate chain、Manifestで許可した固定parser Profileを全てcontent ref／hashからread-backする。anchor集合、policy OID、leaf EKU、path length、issuer／subject linkage、DER署名対象bytes、responder authorization、serial、status、時刻、algorithm、source kind／encodingを再導出し、payloadとbyte-exact一致させる。CRL signerはcertificate issuerまたはRFC 5280で許可されたdirect CRL issuerとし、Miraikanai V1はCRL issuer certificateをv3だけに限定してRFC 10007どおり`keyUsage` extensionの存在と`cRLSign` bitを両方必須にし、v1／v2 issuer certificateを拒否する。OCSP signerはissuer自身またはtarget certificateと**同じCA signing key**で署名され`id-kp-OCSPSigning` EKUを持つdelegated responderだけとし、issuer Nameだけが同じ別Keyによるdelegationを拒否する。`status=good`は`revoked_at`／`invalidity_at`／`revocation_reason`を全null、`status=revoked`はDERのrevocationDateと任意invalidityDate／reasonを再導出し、両時刻をcanonical UTC secondかつ`<= evaluation_time`にする。`assurance_failure_effective_at`は`min(revoked_at, invalidity_at if non-null)`であり、pass評価にはgoodだけを許す。

`source_kind=vendor_crl`は`response_encoding=rfc5280-crl-der`、`produced_at`／nonce／OCSP CertID／Responder Fieldを全null、`delta_crl=false`、`indirect_crl=false`とする。target certificateのissuer Name DER／AKI、CRL issuer Name DER／AKI、certificate CRL Distribution PointとCRL Issuing Distribution Pointを固定parserで意味比較し、direct issuer、対象certificate scope、全reason coverageを証明する。`covered_revocation_reasons[]`はschemaの8値exact set、scope外、only-user／only-CA差、partial reason、unknown critical extension、delta／indirect CRL、v3 issuerのmissing `keyUsage`／`cRLSign=false`をunknownとして拒否する。CRL NumberはDER INTEGERをnon-negative BigIntへdecodeし、zeroをsingle `0x00`、他をleading-zeroなしminimal unsigned big-endian 1～20 octetへ再encodeしたbase64url no-paddingだけを`crl_number_base64url_uint`に許し、数値比較する。`source_kind=vendor_ocsp`は`response_encoding=rfc6960-basic-ocsp-response-der`、CRL number／CRL issuer／distribution point／covered reasonsをnullまたはempty、DER由来`produced_at`、両CRL flag=falseとする。CertIDはRFC 9919のSHA-256だけをV1で許し、target certificateから再導出したissuerNameHash、issuerKeyHash、serial hashと全Fieldをexact一致させる。BasicOCSPResponseのResponderID `byName | byKey`を実signer certificateへ再導出してexact一致させ、wrong ResponderIDを拒否する。SHA-1を署名またはCertIDに使わず、別algorithmが必要なら別schema revisionと実機Qualificationを要求する。

OCSP delegated responder certificateのrevocation methodはChain Policyで一つだけ固定する。`independent_crl`は`ocsp_nocheck_max_certificate_lifetime_seconds=null`、nocheck extension false、responder revocation Evidence pairを全non-nullにし、targetをresponder certificateとするfresh direct-complete CRL Evidence `status=good`へ解決する。`nocheck_short_lived`はmax lifetimeをpositive、revocation Evidence pairを全null、critical=falseかつDER extnValue=`NULL`の`id-pkix-ocsp-nocheck` extensionをexactly once持つことを固定parserで確認してpresent=trueとし、responder certificate `not_after - not_before <= max`かつevaluation timeでvalidを必須にする。method、extension、pairの組合せ差、nocheckの重複／wrong DER value、未承認local policy、AIAだけへの委譲、自己OCSP再帰、expired responder、同issuer Nameだが別CA signing keyを拒否する。CRLReason／OCSP revocationReason欠落は`unspecified`へ一意に写像する。RFC 5280／6960の広い選択肢からCRLNumber必須、RFC 9919のSHA-256 CertID、RFC 9654の32-octet nonce必須を組み合わせるのがMiraikanai V1 strict subsetであり、RFC 9919 high-volume profile全体を採用する意味ではない。accepted parser Profileはissuance response、current response、static Policy diffで同じManifest entryから解決し、呼出元parserへ差替えない。

rollback防止は`OfflineGovernanceVendorRevocationObservationV1`をissuer SPKI＋certificate serial tupleごとのappend-only chainにし、CRL／OCSP両sourceを同じchainへ集約する。`tuple_sha256=SHA-256(UTF-8(JCS({issuer_public_key_spki_sha256,certificate_serial_sha256})))`を唯一のtuple keyとする。この型は論理ID Fieldを持たず完成JCS SHA-256 content ref／hashで同定する。初回sequence 1／previous pair=null、goodの後続はcurrent完成Observationをpreviousにしてsequence exact `N+1`とし、`offline-governance-vendor-revocation-latest/<tuple_sha256>.ref`をexpected-parent CASする。CRLは最も近いCRL ancestorよりCRL Number BigInt strictly increasingかつ`this_update` non-decreasing、同CRL Numberの別response hashを拒否する。OCSPは最も近いOCSP ancestorから`produced_at`と`this_update`を各non-decreasingとし、同produced-at＋this-update＋request nonce tupleの別response hashを拒否する。Observationのstatus、nonce consumption、`earliest_revocation_effective_at=min(revoked_at, invalidity_at if non-null)`をresponseから再導出する。first revoked Observationはそのtupleのterminal currentであり、その後はgood／revokedのどちらもappendせず、Detectionはこのterminal Observation ref／hashと同Observationが指すresponse ref／hashをbyte-exactに束縛する。oldだが未期限のresponseへのrollback、source切替での失効忘却、sequence gap、tuple差、観測前利用を拒否する。

OCSP request前に完成`OfflineGovernanceOcspNonceReservationV1`を作り、`offline-governance-ocsp-nonce/<request_nonce_sha256>.ref`をexpected-empty→Reservation refへCASする。request DERからCertIDのissuer Name／Key hash、serial、exact 32-octet raw nonceとhashを再導出し、Reservationのissuer SPKIはtarget certificate chain、responder SPKIはTrusted Issuer Manifestのapproved responder／endpoint selectionから事前固定して`requested_at < expires_at`を必須にする。OCSPRequestに存在しないResponderIDまたはresponse signerをrequest DERから推論しない。response受信時は`requested_at <= received_at <= expires_at`、response echo nonce、CertID、issuer／serial、BasicOCSPResponseから再導出したactual signer SPKI／authorizationをReservationと一致させ、完成`OfflineGovernanceOcspNonceConsumptionV1`を作って同pointerをexpected-Reservation→Consumption refへ一度だけCASする。OCSP Observationのconsumption pairはnon-nullでConsumption→Reservation→request DERとresponseを閉じ、CRL Observationではnullにする。同nonceの別request／response、未予約、二度目consume、backdated／期限外Consumption、response nonce欠落、in-place row更新を拒否する。crash retryは同じReservation／Consumption bytesだけをidempotent read-backし、別bytesならterminal conflictとする。`this_update <= evaluation_time < next_update`かつ`evaluation_time - this_update <= max_revocation_response_age_seconds`を必須にし、Root Profile／Evidenceの`valid_until`を一個のresponse `next_update`へ固定しない。

revoked Observationをpublishする同じdurable multi-key CASで、`offline-governance-assurance-quarantine/<tuple_sha256>.ref`をexpected-empty→完成`OfflineGovernanceAssuranceIssuerQuarantineObligationV1(state=pending)`へ進める。片方だけ成功させず、store unavailable／crashではcurrent operationをfail closedにして同じcandidateだけをretryする。pending中は当該issuerを使う`authority_pass` Journal row、通常Root／Trust／Product／AI／build／Release publication、new good Observationを禁止し、`revocation_detection`とN／F／R／Xのcontainment／recovery artifactだけを許す。N fulfillmentはcurrent Global＋quarantined Readiness pairをnon-null、Fail-Stop pair null、Fは三pair全non-null、RはGlobal／Fail-Stop pair nullでquarantined Readiness pairだけnon-nullとする。各current pointerのexpected-parent CASとread-back成功後だけ完成`OfflineGovernanceAssuranceIssuerQuarantineFulfillmentV1`を作り、obligation pointerをexpected-Obligation→Fulfillment refへCASする。Xは旧trust domainのObligationを永久にpending terminalで保持して全operationを禁止し、別trust domainのnew Genesisへ進むため、旧domain内で偽のfulfillmentを作らない。Fulfillmentはcontainment完了だけを表し、Readinessがvalidになるまで通常operationは再開しない。

Evidence発行時だけでなく全current authority使用／CAS直前に、同じstable Manifest／Chain Policyからfresh signed CRL／OCSPを取得し上記手順で検証する。current responseはimmutable Evidence内のissuance baselineと同じbytesである必要はないが、issuer／SPKI／serial／Policy continuityを満たす。評価結果はclosed `OfflineGovernanceAssuranceEvaluationJournalRowV1`としてappend-only external journalへ保存する。`evaluation_kind=authority_pass`は`result=passed`、`evaluation_kind=revocation_detection`は`result=revoked_detected`、vendor modeだけObservation pairを両non-nullにしてcurrent tuple pointerと一致、offline modeは両nullとする。完成row JCS SHA-256 content refをexpected-empty CAS keyとし、publication transaction／Receiptはconsumerごとのrequired Evidence集合とJournal row集合をset equalityでread-backする。terminal revoked Observationまたはpending Obligationがあるtupleへ`authority_pass`を作れず、Detection／Incident／Global Ledger／ReadinessはObservationのeffective timeとreasonをexact継承する。derived Fieldだけの自己申告、unsigned／部分DER、chain順差、unknown critical extension、別serial、nonce欠落、stale／rollback responseを拒否する。

Evaluation Journal row、Vendor Observation、OCSP Nonce Reservation／Consumption、Quarantine Obligation／Fulfillmentは論理ID Fieldを持たないcontent-addressed external assurance journal objectであるため§6.0.1の32件へ追加しない。全schemaをoffline catalog、full Local Catalog、Owner正本集合へtyped content-ref closureとして含め、Task Receipt／publication journal fixtureでmissing／extra／wrong branchを拒否する。

offline issuerの後発失効は既存Global Root Revocation Ledgerの`target_kind=assurance_issuer` branchへissuer subject＋SPKI、effective time、typed Incidentをappendし、全current Evidence評価でcurrent Super Headをoverlayする。vendor responseでrevokedを発見した場合も同じissuer row候補と`custody_assurance_invalid` Readiness Invalidationを完成するが、publication pointer集合は§6のN／F／R／X selectorだけが決める。N／FはGlobal＋Readiness、RはReadinessだけ、Xは旧domain pointerを進めず、いずれもTrust／Catalogをcontainment時のhistorical sourceとして保持する。対応Recovery routeのdestination closureで必要なnew global／Readiness pairへ整合化する。Manifestのanchor／issuer／SPKI／algorithm／profile／revocation mode変更は後述のAssurance Static Policy Change Manifestを持つRoot Rotationだけを許すが、同Policy下のfresh CRL／OCSP取得またはjournal追記はRoot Profile／trust domain resetを要しない。unknown anchor／OID／EKU、untrusted issuer、current global ledgerでrevokedしたissuer、manifest／Policy accepted profile差を拒否する。

`OfflineGovernanceObservedFingerprintManifestV1`と`OfflineGovernanceVerificationChannelManifestV1`も論理IDなしのcontent-addressed schemaで、Root Profile／generation／witnessをWitness payloadとexact一致させる。Fingerprint entriesはRoot `root_keys[]`とkey ID set equality、SPKI hashとdisplayed SHA-256 fingerprintを再計算する。Channel observationsは各Root Keyについて二つ以上のdistinct `channel_kind`とdistinct endpoint identity、同じdisplayed fingerprint、Custodian／worker／AIから独立したobserverを必須にし、observation timeをFingerprint row `observed_at <= channel.observed_at <= Witness.attested_at`へ閉じる。Witness集合もdistinct subject／public keyで、同一人／endpoint／channel別名による水増し、missing Key、fingerprint差、future observation、自由URLを拒否する。

`AuthorityBindingSourceCatalogV1.members[]`はLocal Schema Catalogの全`signature_slots[]`との`{schema_id,signature_slot_id}` set equalityを正本とし、その順で重複なくschema hash、branch、purpose、tagged authority sourceを固定する。`authority_source_kind=trust_registry`だけはRoleとsubject／scope derivation四Fieldを全non-null、他三kindは全nullとする。offline Root slotは当該purposeのRoot Profile quorum、Recovery custodian slotはRecovery Policyのcustodian quorum、construction executor slotはConstruction Authorizationのexact subject／public keyへ解決し、これらを通常Trust Roleへ偽装しない。同purposeを持つsource／destination slot、Rotation old／new、Recovery三slot、Incidentのroot／recovery-custodian branchをslot／branch IDで区別し、inactive branchの署名混入とactive branchの欠落を拒否する。`catalog_id`は同Fieldを除くJCS hashから`urn:mirakan:authority-binding-source-catalog:sha256:<lowercase-hex>`として導出する。初回`AuthorityBindingSourceCatalogHeadV1`はprevious=null／sequence 1／`authorization_kind=offline_root_genesis`で完成Root Headを束縛し、Authorizationはその完成HeadとCatalogをref／hashで承認する。稼働後の変更はcurrent Trust closureとCatalog headをsource、destination Catalogとexact diffをdestinationとして`TrustRegistryChangeManifestV1`へ閉じ、fresh `TrustRegistryChangeApprovalV1`で承認し、Trust closure headとCatalog headを一つのexpected-parent CASで更新する。説明文の「少なくとも」列挙、未注釈schema introspection、Catalogだけの独立更新を正本にしない。

Catalog memberは`{schema_id,signature_slot_id}`のunsigned UTF-8 tuple順・重複なしとし、同一schema IDの複数slotを正規に許す一方、同一tupleを拒否する。Local Schema Catalog annotationからschema hash、branch、purpose、issued-at source／path、revocation context／path、authority source kindをbyte-exact複写し、一FieldでもCatalog側で変更しない。`authority_source_kind=trust_registry`だけがsubject／scope derivation ref／hashを完成`AuthorityBindingDerivationPolicyV1`へexact解決する。`exact_ref_field_v1`はsubject policyだけでinput path一件、fixed output／prefix=nullとし、そのcanonical subject refをbyte-exact pass-throughする。`catalog_fixed_ref_v1`はsubject policyだけでinput paths=[]、fixed output non-null、prefix=nullとする。`rfc8785-jcs-sha256-v1`はfixed output=null、non-empty field-path tupleをpath順のclosed objectへ投影してJCS SHA-256を計算し、subject policyならnon-null URN prefixへsuffix、scope policyならprefix=nullのlowercase hex 64を返す。input schema／path不存在、配列indexや表示名依存、unknown algorithm、nullability差、同入力で複数outputを拒否する。Task 0が生成または再利用確認するIdentity／Role／Assignment／purpose KeyとProvisioning Receiptの`required_authority_bindings[]`は、Catalogのtrust-registry slotだけから導出した`{schema_id,signature_slot_id,subject_ref,role_ref,signed_record_purpose,assignment_scope_sha256}`集合とset equalityにする。offline Root／Recovery custodian／construction executor slotをこの集合へ混入しない。

Architecture Governanceが所有する`MirakanSignedRecordV1` slotは次表のissued-atとrevocation contextを正本とし、Local Schema Catalog／Authority Catalogのslot annotationとset equalityにする。`payload_field`はenvelope `issued_at`／`revocation_snapshot_ref`を指定payload pathへbyte equality、Trust closure contextはenvelope snapshot refを当該closureの完成`revocation_current_ref`へ一致させ、さらに評価時current revocationも適用する。

| wrapper／slot | exact `signed_record.issued_at` | exact revocation context |
|---|---|---|
| Authority Catalog Head source | payload `recorded_at` | authorizationが束縛するsource Trust closure |
| Authority Catalog Head destination | payload `recorded_at` | approved-change destination（必要時）またはRecovery Authorizationのdestination Trust closure。genesis slotは存在しない |
| Trust Registry Change Approval | payload `issued_at` | payload snapshot refかつsource Trust closure |
| Trust Registry Closure Head source | payload `recorded_at` | source Trust closure。`source_revocation_snapshot_ref` non-null |
| Trust Registry Closure Head destination | payload `recorded_at` | destination Trust closure。`destination_revocation_snapshot_ref` non-null |
| Construction Decision | payload `issued_at` | payload `revocation_snapshot_ref` |
| Bootstrap Approval | payload `issued_at` | payload `revocation_snapshot_ref` |
| Control Plane Governance Incident | payload `confirmed_at` | payload `revocation_snapshot_ref`とrouteが指定するhealthy Trust closure |
| Recovery Rebaseline Authorization | payload `issued_at` | payload `revocation_snapshot_ref`とrecovered current Trust closure |
| Rebaseline Approval | payload `issued_at` | payload `revocation_snapshot_ref` |
| Rebaseline Transaction | payload `recorded_at` | payload `revocation_snapshot_ref` |
| Architecture Document Change Manifest | payload `generated_at` | payload `revocation_snapshot_ref` |
| Document Lifecycle Record | payload `recorded_at` | payload `revocation_snapshot_ref` |

Trust／Catalog Headのsource＋destination branchでは各envelopeが別snapshot refを持ち、単一payload snapshotを両signerへ流用しない。Trust Headのgenesis／root-recovery destination-onlyはsource snapshot=null、destination non-null、source-onlyは逆、source-and-destinationは両non-nullとし、対応closureのsnapshotとexact一致させる。slot annotationのissued-at path、revocation kind／path、branch IDのmissing／extra／nullability差、payload時刻の自由入力、source snapshotをdestination signerへ流用することを拒否する。AI／Product Ownerのwrapperは各Ownerの同じslot annotation contractとそのOwner表を正本とし、この表へ再定義しない。

Catalog Headの`head_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:authority-binding-source-catalog-head:sha256:<lowercase-hex>`として導出する。genesis wrapperはRoot purpose `offline_governance_authority_binding_source_catalog_head_genesis`のthreshold signatureだけを持ち、source／destination signatureはnullとする。`approved_change`はsource signed record必須、destination publisher bindingを追加・置換する場合だけdestination signed recordも必須とし、両者は同じ完成payload、purpose `authority_binding_source_catalog_head`へ署名する。`root_recovery`はRoot Recovery Authorizationを束縛し、root signature／source signatureをnull、new destination publisher signatureを必須とする。sequenceはexact `N+1`、previousはcurrent完成Catalog Head、`authority-catalog-current.ref`はTrust current pointerと同一transactionでCASする。Approvalはdestination Catalog bytesまでを承認し、完成前のdestination Headを参照しない。これによりApproval→Catalog bytes→Head→Approvalの一方向とし、Head↔Approval hash cycleを禁止する。

Catalog Headの`root_backed_trust_recovery`はcompleted `TrustRegistryQuarantineRecoveryAuthorizationV1`をauthorizationとして束縛し、root threshold slot／source slotをnull、destination slotを必須にする。previous=current quarantined Catalog Head、sequence exact `N+1`、destination CatalogはAuthorizationとexact一致し、Trust Headおよびremediation valid Readiness Headの同branchと三current pointerを一つのexpected-parent CASで進める。Root／local／global pointerはexpected-currentのまま不変とする。

Catalog Headの`recovery_readiness_remediation`は完成`OfflineGovernanceRecoveryReadinessRemediationAuthorizationV1`をauthorizationとして束縛し、root threshold slot／source slotをnull、destination slotを必須にする。previous=current Catalog Head、sequence exact `N+1`、destination Catalog、recorded time、Authorization pairを同branchのTrust Headとbyte equalityにする。`same_generation`はReadiness／Trust／Catalog三pointer、`assurance_rotation`はRoot／local／global／Readiness／Trust／Catalog六pointerのexpected-parent CASだけでpublishし、root-backed Trust recovery、Root recovery、通常Change Approvalへ読み替えない。

各署名者の入力は`UTF-8(JCS(OfflineGovernanceThresholdSigningSubjectV1))`そのものであり、ECDSA P-256 with SHA-256に一度だけ渡す。`record_id`は`record_id`だけを除き、完成`signing_subject`と全`signatures[]`を含むwrapperのJCS SHA-256から`urn:mirakan:offline-governance-record:sha256:<lowercase-hex>`として導出する。Verifierは解決したsubject bytesのschema ID／ref／SHA-256、purpose、時刻を`signing_subject`とexact一致させ、完成wrapper IDも再計算する。payloadや署名を除いた別projection、二重hash、署名後の`record_id`差替えを拒否する。

Threshold subjectの時刻は次のmappingを唯一の正本とする。`signer Profile expiry`は当該slotのexact key universeを供給するRoot Profileの`valid_until`であり、Recovery custodian slotではRecovery Policyを束縛したprevious Root Profileの`valid_until`を使う。`min(...)`はcanonical UTC秒の最小値である。

| wrapper／signature slot | exact purpose | `signing_subject.issued_at` | `signing_subject.valid_until` |
|---|---|---|---|
| Root Profile | `offline_governance_root_profile` | payload `issued_at` | payload `valid_until` |
| Root Rotation old／new | `offline_governance_root_rotation_old`／`offline_governance_root_rotation_new` | 各wrapper生成時のauthenticated ceremony clock。old／newで別値を許すが`overlap_starts_at <= issued_at <= rotation_activated_at` | 各old／new signer Profile expiry |
| Verifier Upgrade Authorization old／new | `offline_governance_verifier_upgrade_old`／`offline_governance_verifier_upgrade_new` | payload `issued_at` | `min(payload.valid_until, 各source／destination signer Profile expiry)` |
| Root Recovery uncompromised／custodian／new | `offline_governance_root_recovery_uncompromised`／`offline_governance_root_recovery_custodian`／`offline_governance_root_recovery_new` | 各wrapper生成時のauthenticated ceremony clock。`cooling_off_started_at <= issued_at <= recovery_activated_at`、partialのuncompromisedだけnon-null | 各previous／previous／next signer Profile expiry |
| Root Head | `offline_governance_root_head` | payload `activated_at` | signer Profile expiry |
| Root Security Incident root branch／custodian branch | `offline_governance_root_security_incident`／`offline_governance_root_recovery_custodian` | payload `confirmed_at` | affected Root Profile expiry |
| generation-local Revocation Snapshot／Head | `offline_governance_root_revocation_snapshot`／`offline_governance_root_revocation_head` | 各payload `recorded_at` | signer Profile expiry |
| global Revocation Ledger／Super Head | `offline_governance_global_root_revocation_ledger`／`offline_governance_global_root_revocation_super_head` | 各payload `recorded_at` | authorizing signer Profile expiry |
| Recovery Readiness Receipt | `offline_governance_recovery_readiness` | payload `issued_at` | `min(payload.valid_until, signer Profile expiry)`かつpayload `valid_until <= signer Profile expiry` |
| Recovery Readiness Incident root branch | `offline_governance_recovery_readiness_incident` | payload `confirmed_at` | signer Profile expiry |
| Recovery Readiness Incident custodian emergency branch | `offline_governance_root_recovery_custodian` | payload `confirmed_at` | affected previous Root Profile expiry |
| Recovery Readiness Invalidation／Head root branch | `offline_governance_recovery_readiness_invalidation`／`offline_governance_recovery_readiness_head` | 各payload `recorded_at` | signer Profile expiry |
| Recovery Readiness Invalidation／Head custodian emergency branch | `offline_governance_root_recovery_custodian` | 各payload `recorded_at` | affected previous Root Profile expiry |
| Assurance Issuer Fail-Stop | `offline_governance_assurance_issuer_fail_stop` | payload `recorded_at` | source signer Profile expiry |
| Recovery Readiness Remediation Authorization | `offline_governance_recovery_readiness_remediation_authorization` | payload `issued_at` | `min(payload.valid_until, authorization signer Profile expiry)` |
| Authority Catalog genesis Head | `offline_governance_authority_binding_source_catalog_head_genesis` | payload `recorded_at` | signer Profile expiry |
| Trust Quarantine Incident | `trust_registry_quarantine_incident` | payload `confirmed_at` | current healthy Root signer Profile expiry |
| Trust Quarantine Recovery Authorization | `trust_registry_quarantine_recovery_authorization` | payload `issued_at` | current healthy Root signer Profile expiry |
| Construction Authorization | `control_plane_construction_authorization` | payload `issued_at` | payload `valid_until`かつ`valid_until <= signer Profile expiry` |
| Trust Provisioning Receipt | `trust_provisioning_receipt` | payload `issued_at` | `min(signer Profile expiry, Construction Authorization valid_until, Recovery Readiness Receipt valid_until)` |
| Trust Registry Recovery Authorization | `trust_registry_recovery_authorization` | payload `issued_at` | next signer Profile expiry |

全slotで`signer Profile.issued_at <= signing_subject.issued_at < signing_subject.valid_until`、payload event timeとの上表一致、署名時点のlocal／global revocationを必須にする。同一threshold wrapper内の全signatureは同じ完成subject bytesへ署名し、署名追加時にsubject時刻を変更しない。`valid_until`は新しいauthority使用の終端であってimmutable recordの削除時刻ではない。期限後のwrapperは`authentic_but_expired`として履歴検証できるが、新しいAuthorization、Approval、Head、Receipt、CASを認可しない。Rotation／Recoveryのatomic pointer遷移がold global Head等を明示継承した場合だけ、その旧署名済みimmutable dataをcurrent closureの入力として保持できる。この継承は完成遷移recordとcurrent pointer chainから再検証し、期限切れ旧Keyへ新規署名を許す意味にはしない。

generation 1のRoot Profile `allowed_purposes[]`は`control_plane_construction_authorization | trust_provisioning_receipt | offline_governance_root_profile | offline_governance_root_head | offline_governance_root_rotation_old | offline_governance_root_rotation_new | offline_governance_verifier_upgrade_old | offline_governance_verifier_upgrade_new | offline_governance_assurance_issuer_fail_stop | offline_governance_recovery_readiness_remediation_authorization | offline_governance_root_recovery_uncompromised | offline_governance_root_recovery_new | offline_governance_root_security_incident | offline_governance_recovery_readiness | offline_governance_recovery_readiness_head | offline_governance_recovery_readiness_incident | offline_governance_recovery_readiness_invalidation | offline_governance_root_revocation_snapshot | offline_governance_root_revocation_head | offline_governance_global_root_revocation_ledger | offline_governance_global_root_revocation_super_head | offline_governance_authority_binding_source_catalog_head_genesis | trust_registry_quarantine_incident | trust_registry_quarantine_recovery_authorization | trust_registry_recovery_authorization`のclosed setとする。`purpose_quorums[].purpose`はこの集合とset equality、各`exact_key_ids[]`は`root_keys[]`のsubset、同一purpose内で重複なしとする。`offline_governance_root_recovery_custodian`はRoot Profile purpose集合へ入れず、Recovery Policyの別key universeだけが所有する。各wrapper位置はpurposeを一意に固定し、root profile／Root Head／Authorization／Receipt／Root Security Incidentのroot branch／local Revocation Snapshot／Head／global Revocation Ledger／Super Head／Recovery Readiness Receipt／Head／Incident／Invalidation／Assurance Issuer Fail-Stop／Recovery Readiness Remediation Authorization／初回Authority Binding Source Catalog Head／Trust Quarantine Incident／Trust Quarantine Recovery Authorization／Trust Registry Recovery Authorizationはそれぞれ対応Root purpose、RotationとVerifier Upgradeは各old／new、Recoveryはuncompromised／newをRoot quorum、custodian slotだけをRecovery Policy quorumに要求する。通常Product、AI Provider、build、Releaseの署名purposeをRootへ追加しない。

Threshold verifierのnormal branchは「system current Profile」を署名済み履歴へ上書き適用せず、wrapper slotとsubject bytesが束縛するexact signer Root Profile／generationをRoot Head chainから解決する。新規署名ではそのProfileをcurrent完成Root Headと一致させ、履歴検証では当該historical Headのactivation chainを検証する。purposeに対応するそのProfileの`purpose_quorums[].exact_key_ids[]`だけをkey universeとし、Root Profileの`root_keys[]`とsignatureをkey ID byte順にする。各KeyのSPKI DERをread-backしてSHA-256を一致させ、署名`issued_at`時点の同generation local Root Revocation Snapshotと、検証時current Global Root Revocation Ledgerの双方で失効していないdistinct Keyかつdistinct custodianのvalid signature数がthreshold以上であることを検証する。これによりRotation後もold Ledger／Headをhistorical Profileで真正性検証しつつ、retroactive global失効を適用できる。candidate例外は次のclosed ceremony branchだけである。generation 1 Root Profileは自身の`offline_governance_root_profile` purpose self-quorum、全structural／custody条件、out-of-band fingerprint、完成Genesis witness／Ceremonyとのkeyset・quorum一致で検証し、pre-existing local／global revocationを要求しない。続くgeneration 1 local Revocation genesis Snapshot／HeadとRoot Headは完成Genesis Ceremony＋candidate Profileの対応purpose quorum、empty global genesis Ledger／Super Headは完成Root Head＋local genesis＋candidate Profileのglobal purpose quorumで検証する。四pointer publish後にProfileをnormal branchで再検証するまで、candidate Profileまたは中間artifactでAuthorizationを発行しない。generation 2+ next Profileは自身のRoot Profile purpose self-quorumに加え、partial Rotationならsource current local／globalでvalidなold quorum＋next Profileのnew quorum、Recoveryならsource current globalでvalidなRecovery custodian quorum＋next Profileのnew recovery quorumを同一完成Rotation／Recovery payload上で検証する。next local sequence 1 Snapshot／Headとcandidate Root Headはそのactivationとnext Profile purpose quorum、total-compromiseならcandidate next Global Ledgerも検証するが、§6の六pointer CAS成功までcandidateをcurrent authorityにしない。publish後はnext local＋current globalで全candidate wrapperをnormal再検証し、candidate例外の再利用を拒否する。同一custodianの複数Keyは一票、thresholdは1以上かつexact key universe内のdistinct custodian数以下、全KeyはP-256／P1363／low-Sだけを許可する。Recovery custodian purposeだけはRecovery Policyの`recovery_custodian_quorum.exact_key_ids[]`と`recovery_custodians[]`をexact key universeとし、Root purpose_quorumsを参照しない。全Recovery custodianをcurrent Root custodian、worker、AI、通常管理者から独立させる。

Root Profileの`offline_governance_verifier_profile_ref/hash`は完成`BootstrapVerifierProfileV1`、`offline_governance_schema_catalog_ref/hash`は同Profileの`supported_offline_governance_schema_catalog_ref/hash`とbyte equalityの完成`OfflineGovernanceSchemaCatalogV1`へ解決する。このfrozen-minimal catalogのvalidation root schema ID集合は、`urn:mirakan:schema:offline-governance:root-profile:v1`、`urn:mirakan:schema:offline-governance:root-head:v1`、`urn:mirakan:schema:offline-governance:root-rotation:v1`、`urn:mirakan:schema:offline-governance:root-recovery:v1`、`urn:mirakan:schema:offline-governance:root-security-incident:v1`、`urn:mirakan:schema:offline-governance:root-revocation:v1`、`urn:mirakan:schema:offline-governance:global-revocation:v1`、`urn:mirakan:schema:offline-governance:recovery-readiness:v1`、`urn:mirakan:schema:offline-governance:assurance:v1`、`urn:mirakan:schema:offline-governance:bootstrap-verifier-profile:v1`、`urn:mirakan:schema:offline-governance:schema-catalog:v1`、`urn:mirakan:schema:offline-governance:dual-verifier-comparison:v1`、`urn:mirakan:schema:offline-governance:verifier-upgrade-authorization:v1`、`urn:mirakan:schema:offline-governance:trust-quarantine:v1`、`urn:mirakan:schema:offline-governance:construction-authorization:v1`、`urn:mirakan:schema:architecture:local-schema-catalog:v1`のexact setとする。`policy_bound_schema_ids[]`はcandidate Root Profileが束縛するRotation／Recovery／Assurance／Governance Recovery Policy内の全evidence／proof／subject-projection schema ID、Trusted Issuer Manifestが参照するchain／revocation／parser profile schema IDのunionとset equalityにする。membersは全validation root＋policy-bound root自身と、各rootからfragment-only `$ref`またはclosed `x-mirakan-content-ref-target-schema-ids` annotationを辿ったleast fixed-point transitive closureとのset equalityにし、Catalog自身、Dual Verifier Comparison、Trusted Issuer／Vendor／proof／projection／parser、content-refだけで到達する型も省略しない。全content-ref Fieldはtarget schema ID annotationをnon-emptyで持ち、自由refを禁止する。member／signature slotはschema ID＋slot ID順・重複なし、全ref／hash／annotationをread-backし、論理IDを持たない完成JCS content ref／hashで同定する。これによりRoot／Recovery／Readiness／Trust-quarantine／Verifier upgradeとLocal Catalog承認をTrust Registryや生成対象Repositoryに依存せず検証する。

`verifier_core_sha256`は同Field、`verifier_profile_id`、`independent_verification_evidence[]`を除くsource／binary／toolchain／offline catalog／vector／supplier projectionのJCS hashである。genesisはout-of-band verifier／offline catalogを人間ceremonyで照合し、Rotation／Recovery／Trust quarantineはaffected signer Profileが固定したcore／offline catalogだけを使う。Construction AuthorizationのBootstrap Verifier／offline catalog pairはcurrent Root Profileとbyte equalityにする一方、`local_schema_catalog_ref/hash`は別のevolvable full catalogであり、offline catalog全memberを同じschema bytes／hash／signature slotで包含し、さらにControl Plane／Product／AI／Runtime／Platform全schema closureを持つ。AuthorizationのRoot quorumはこのfull catalog、Materialization Plan、Authority Catalogを明示承認する。offline subset差、Local Catalogからのoffline member欠落、Root Profileにfull Local Catalogを固定する実装を拒否する。

通常Rotationでverifier coreとoffline catalogがpreviousからbyte-exactなら、RotationのVerifier Upgrade pairを両nullとし、同coreへfresh reproduction Evidenceを束縛したnew Verifier Profile ref／hashだけを許す。coreまたはoffline catalogを変更する場合は両Fieldをnon-nullにして完成`OfflineGovernanceVerifierUpgradeAuthorizationV1`を参照する。source verifierはsource catalogに既存のUpgrade Authorization schemaでsource Root／Profile／catalog、payload、old-purpose quorumを検証し、destination verifierはdestination catalog／Profile、candidate reproduction Evidence、new-purpose quorum、同じpayloadを検証する。Dual Verifier Comparison Manifestのsource suite pairはsource Profileのknown-answer／negative vector pair、destination suite pairはdestination Profileのpairとbyte equalityにする。old／new両verifierがsource suite全件を実行してresult rootをbyte equalityかつ`source_suite_result_equivalence=passed`、new verifierがdestination suite全件を実行して`destination_suite_result=passed`を必須にし、candidate reproduction Evidence projectionもdestination core／catalog／両destination suite pairとexact一致させる。Authはsource／destination Profile pair、catalog pair、comparison pair、`issued_at < valid_until`を閉じる。Root Rotationは同じAuth pairを署名対象に含め、old＋new Rotation quorumとold＋new Verifier Upgrade quorumを満たして六pointer CASした後だけnew core／catalogをcurrent化する。Auth片側null、別Rotation、source suite差／subset、destination suite未実行、result差、期限後publication、candidate先行利用を拒否する。

old catalogがUpgrade schemaを解釈できない、source verifier／catalog自体のcompromiseが確認された、Root Profile／threshold envelope／Upgrade schemaの互換境界を変更する場合は既存trust domainの継続を禁止しfull trust resetへ進む。Root Recovery中のverifier／offline catalog切替も禁止し、next Profileはaffected Profileと同じcore／offline catalogを使う。対してfull `LocalSchemaCatalogV1`だけの追加／変更はcurrent Rootを変えず、healthy Trust ChangeでAuthority Catalogとclosureを更新し、§6.1.1のsame-definition rebaselineまたはDefinition migrationでcurrent Baselineへ承認できる。Local Catalog変更をRoot Rotation／full trust resetへ誤routeしない。

`assurance_profile=development_bootstrap`は、一人の人間Custodianが管理するOS-backed non-exportable Keyによるpurpose別1-of-1を許すが、このbaselineからProduction release、managed external hostによる実行コード受入れ、Production Credential発行を許可しない。`assurance_profile=production`のProfile構造条件は全Root Keyをhardware non-exportable、各purpose quorumを二人以上の独立Custodian、Recovery custodianをRoot custodianとは別の二人以上、Production Recovery Policyのtotal-compromise branchをpre-commitすることまでとし、まだ存在しないReadiness HeadをProfile FieldまたはProfile検証入力にしない。Production activation／運用開始GateはProfile→Root Head→local／global revocation→Recovery Readiness Receipt／Headの完成後、四current pointerをatomic publishしてstatus=validかつnon-expiredのReadinessをread-backできることを別途必須にする。developmentからproductionへの昇格はRoot Profileの新generation、Production trust closure、Production用Product policyを含むdestination Active Definition、§6.1.1のfull-reset Definition migration Rebaselineを必須とし、same-definition rebindingやProfile Fieldの上書きで代用しない。

generation 1のRoot Headはpayload fingerprintと各Custodian fingerprintを`in_person_display | printed_fingerprint | hardware_token_display`のうち二つ以上の異なるchannelで人間がout-of-band照合したGenesis Ceremony Recordをactivation recordとして必要とし、自己署名だけをtrust anchorにしない。`OfflineGovernanceWitnessAttestationV1`の署名入力は`UTF-8(JCS(payload))`そのもので、domain separator、Root Profile、keyset、観測fingerprint、verification-channel manifest、source Evidence、Witness identityを閉じ、P-256 with SHA-256へ一度だけ渡す。`attestation_id`は同Fieldを除くpayload JCS hash、Genesis Recordのattestation ref／hashは完成wrapper、各channelのfingerprint集合はRoot key集合とset equality、WitnessはCustodianとworker／AIから独立でなければならない。Root Profile自体にactivation record refを持たせず、generation 2+はRoot Headが完成Rotation／Recovery wrapperを一意に束縛する。Root private key、recovery secret、CredentialをRepository、Receipt、logへ入れない。

`OfflineGovernanceRootHeadV1`はimmutableなRoot世代chainの唯一のhead型であり、`head_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:offline-governance-root-head:sha256:<lowercase-hex>`として導出する。wrapperはpayload全bytesをpurpose `offline_governance_root_head` quorumで署名する。genesisはgeneration 1、previous両Field=null、activation record=完成Genesis Ceremony Record、`activated_at=GenesisCeremony.completed_at`とし、Root Profile、ceremony keyset、purpose quorum、trust domainをexact一致させる。また空entries、sequence 1、previous=nullの完成local Root Revocation Snapshot／Headを`root_revocation_genesis_head_ref/hash`へ束縛する。Rotation／Recoveryはcurrent完成Root Head wrapperをprevious ref／hashへ束縛し、generationをexact `N+1`、activation recordを対応する完成signed Rotation／Recovery wrapper、新generationの空local Revocation genesis Headを束縛し、next Root ProfileのRoot Head purpose quorumで署名する。Rotationは`activated_at=rotation_activated_at`かつ`overlap_starts_at <= activated_at < overlap_ends_at`、Recoveryは`activated_at=recovery_activated_at`かつpolicy cooling-off完了後とする。`root-current.ref`、`root-revocation-current.ref`、`root-global-revocation-current.ref`、`root-readiness-current.ref`はそれぞれ完成Head ref／hashだけを保持する。Root generation切替はsingle writerのexpected-parent CASだけで行う。RotationはRoot／local revocation／Readinessを完成candidateとしてstagingし、global Super Headをsource expected-currentのまま継承する。destination Trust closure、exact Change Manifest／Approval、Trust Head、Catalog Headまでをprepublish完成し、Root／local／global／Readiness／Trust／Catalog六current pointerを一つのexpected-parent CASでcurrent化する。RecoveryもRoot／local／Readinessを完成candidateとしてstagingし、partial branchはsource globalを継承、total branchは通常reasonなら全old-generation Root Key、issuer-loss Rならその集合と当該assurance-issuer row一件のexact unionだけをappendしたcandidate next Global Ledger／Super Headを作るが、まだcurrentにしない。完成candidate destination Trust closure、Recovery Authorization、Trust Head、Catalog Headを後段で作り、Root／local／global／Readiness／Trust／Catalogの六current pointerを一つのexpected-parent CASでcurrent化する。missing／extra generation、global reset／rollback、branch、filesystemの`latest`探索を拒否し、全旧headを監査履歴として保持する。

generation-local Root Revocationの`snapshot_id`とHead `head_id`は各ID Fieldを除くpayload JCS hashから`urn:mirakan:offline-governance-root-revocation-snapshot:sha256:`／`urn:mirakan:offline-governance-root-revocation-head:sha256:`をprefixとして導出する。genesis branchはentries=[]、sequence 1、previous ref／hash=nullを必須とし、直前Snapshotを要求しない。generation 1はGenesis CeremonyがattestしたRoot Profileの対応purpose quorum、generation 2+は完成Rotation／Recoveryがactivateしたnext Root Profileの対応purpose quorumでSnapshotとHeadを検証する。non-genesis branchだけが同generationの直前current Snapshot／Headをprevious、sequence exact `N+1`、entries strict superset、`key_id` unique、各Key at most one immutable entry、既存entry byte equalityとし、評価時点にlocal＋global失効していないpurpose quorum署名を必須にする。削除、effective time書換え、unknown／別generation Key、genesisのnon-empty entriesを拒否する。

cross-generation Root KeyとAssurance issuer失効の正本は`OfflineGovernanceGlobalRootRevocationLedgerV1`／`OfflineGovernanceGlobalRootRevocationSuperHeadV1`である。`ledger_id`は同Fieldを除くpayload JCS SHA-256から`urn:mirakan:offline-governance-global-root-revocation-ledger:sha256:<lowercase-hex>`、`head_id`は同Fieldを除くpayload JCS SHA-256から`urn:mirakan:offline-governance-global-root-revocation-super-head:sha256:<lowercase-hex>`として導出する。global genesisはRoot generation 1 Head完成後、entries=[]、sequence 1、previous=null、authorizing Root generation 1の対応purpose quorum署名で作る。以後はcurrent Super Head／Ledgerをprevious、両sequence exact `N+1`、entries strict superset、既存entry byte equalityとし、原則recorded時のcurrent Root Head／Profileのglobal revocation purpose quorumで署名する。例外は、完成total-compromise Recoveryがactivateするcandidate new Root Head／Profile quorumによるprepublication appendと、完成Assurance Issuer Fail-Stopが束縛するsource-context appendだけである。前者はauthorizing Root fieldsをcandidateへ、後者はsource Rootへ一致させ、いずれも対応combined expected-parent CAS成功までcurrentとして扱わない。

`target_kind=root_key`は`{target_root_generation,key_id}`をimmutable Root Head chain上のexact Root Profile Keyへ解決し、nullable issuer両Fieldをnull、unique keyを同tupleとする。`target_kind=assurance_issuer`はRoot Head chain上のいずれかのstable Trusted Issuer Manifest entryへ`{issuer_subject_ref,issuer_public_key_spki_sha256}`をexact解決し、root generation／key IDをnull、unique keyをissuer tupleとする。localはKey IDだけ、globalはbranch別unique keyで各target at most one immutable entryとし、既存row全bytesを後続Ledgerへ保持する。unknown target、duplicate issuer alias、target-kind Field laundering、issuer rowをRoot Key compromise／Key ID永久再利用禁止へ誤適用、entry削除、旧generation-local pointerの再current化を拒否する。

candidate Root Profileの全`root_keys[]`はcurrent Root Headから辿れる全historical Profileとcurrent Global Ledgerを横断検査する。`key_compromise | custody_loss | policy_violation`がeffectiveなhistorical entryの`key_id`または対象Profileの`public_key_spki_sha256`と一致するcandidate Keyは、別generation／別custodian／別key IDへ名前を変えても永久に拒否する。通常Rotation／Recoveryでは全candidate key IDも全historical Profileに対してglobal unique、SPKI hashも未使用を必須にする。唯一の再利用例外はpolicy window内の明示的inverse rollback Rotationで、再利用対象の全historical global entryが`superseded_generation`だけ、serious revocation entryが0、old／new両quorumとrollback deadlineを満たす場合に限る。serious失効Key／SPKIの洗浄、失効前世代へのpointer rollback、同じSPKIのnew key ID再登録をnegative fixtureで拒否する。

Root失効Incidentの正本は`OfflineGovernanceRootSecurityIncidentV1`だけである。`incident_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:offline-governance-root-security-incident:sha256:<lowercase-hex>`として導出し、statusは`confirmed`、Evidenceはnon-empty content-addressed set、affected Keyはaffected Root Profileのexact Key IDだけを許す。`confirmation_kind=root_quorum`はincident kind `key_compromise | custody_loss | policy_violation`だけで、affected Key除外後もIncident／Recoveryに必要なcurrent Root purpose quorumを構成できるpartial branchとし、purpose `offline_governance_root_security_incident`で署名する。`confirmation_kind=recovery_custodian_quorum`はincident kind `threshold_loss | total_root_compromise`だけで、current Root purpose quorumを構成不能なtotal branchとし、current Recovery Policyのpurpose `offline_governance_root_recovery_custodian` exact key universeで署名する。Recovery Policyの`total_compromise_incident_kinds[]`は後者二kindとset equality、`triggering_incident_kinds[] - total_compromise_incident_kinds[]`は前者三kindとset equalityにする。custodian branchでRoot署名を要求／受理せず、root branchでthreshold／total kind、Recovery custodian Keyを受理しない。`detected_at <= confirmed_at`、`compromise_effective_at <= confirmed_at`、recovery-custodian confirmationでは`affected_key_ids[]`をaffected Root Profile全Keyとset equalityにし、そのgeneration全体をuntrustedとして扱う。

localおよびglobal `target_kind=root_key` entryの`incident_ref/hash`はreasonごとのclosed unionである。`key_compromise | custody_loss | policy_violation`は完成confirmed Root Security Incidentへ解決し、affected Root generation／Key、`compromise_effective_at=effective_at`をentryとexact一致させる。reason→Incident kind mappingは`key_compromise -> key_compromise | total_root_compromise`、`custody_loss -> custody_loss | threshold_loss`、`policy_violation -> policy_violation`のclosed relationとする。`superseded_generation`はglobal root-key branchだけで許し、当該old generationからexact `N+1` generationへactivateした完成`OfflineGovernanceRootRotationV1`または`OfflineGovernanceRootRecoveryV1`へ解決し、`effective_at`をそのRoot Head `activated_at`と一致させる。

global `target_kind=assurance_issuer` entryは完成`OfflineGovernanceRecoveryReadinessIncidentV1(incident_kind=custody_assurance_invalid)`へ解決し、Incidentのexactly-one Detectionからissuer tuple、global reason、`assurance_failure_effective_at=effective_at`を継承する。vendor DER mappingまたはoffline assessmentが示す`issuer_compromise | issuer_distrust | policy_violation`以外を許さない。Assurance issuer失効はRoot署名のhistorical cryptographic authenticityやRoot Key SPKI自体を失効させず、当該issuerに依存するcustody／reproduction／Genesis Evidenceの**current usability**だけへoverlayし、Fail-Stop historyはsource-contextで保持する。Issue URL、Markdown、自由Evidence ref、reason／target kind／generation／Key／issuer／時刻差を拒否する。Root Recovery payloadのprevious Root Head ref／hash／generationは開始時current完成Head、Incidentのaffected Root Head ref／hash／generation、previous Root Profile、後続candidate Root Headのprevious pair／`generation-1`とbyte-exact一致させる。incident ref／hash、`incident_kind`、`compromise_effective_at`も完成Root Security Incidentから継承し、`partial_compromise`はroot-quorum確認、`total_compromise`はrecovery-custodian確認だけを許す。

Revocation時刻規則もclosedとする。local non-genesis Snapshotの新規entryは`effective_at <= snapshot.recorded_at`、Snapshotとそれを指すHeadの`recorded_at`はbyte equalityかつ直前Snapshot／Headよりstrict laterとする。global genesis Ledger／Super Headの`recorded_at`はbyte equalityとし、non-genesisで新規追加した全entryの`recorded_at`はdestination Ledger `recorded_at`とbyte equality、`effective_at <= entry.recorded_at`、Ledger／Super Head `recorded_at`はbyte equalityかつ両previous recordよりstrict laterとする。Ledger／Super Headのsequence、previous pair、authorizing Root Head／generationも相互exact一致させ、既存entryの`effective_at`／`recorded_at`／Incidentを含む全bytesは不変とする。retroactive評価は過去の`effective_at`を許すことだけを意味し、未来effective time、Headだけの時刻更新、LedgerとSuper Headの時刻差を許可しない。

Verifierは任意generationの署名時点に対応するlocal Snapshotと、検証時current Global Ledgerの両方を必ず評価する。後日記録でもglobal entryの`effective_at <= historical_evaluation_time`なら当該historical proofをquarantineし、`effective_at > historical_evaluation_time`なら履歴を保持して当該Keyの新規使用だけを停止する。rotation後の旧Key compromiseは旧local chainへ追記せず、current Rootがglobal ledgerへ追記する。global Super HeadはRoot generation切替でresetしない。Rotation／partial recoveryは同じcurrent Headを継承し、total-compromise recoveryだけはsource currentをpreviousとするexact next Headを新generationが署名して§6のatomic transactionでcurrent化する。

Pre-ceremonyはRoot Profile／Genesis Ceremony、local Revocation genesis Snapshot／Head、これを束縛するRoot Head、global Revocation genesis Ledger／Super Head、Root Headを束縛する初回Recovery Readiness Receipt／Headの順に完成し、その後だけRoot／local revocation／global revocation／readinessの四current pointerへatomic publishする。Authorization発行時はRoot Head／Profile、current local Revocation Head／Snapshot、current global Super Head／Ledger、current Recovery Readiness Head／Receiptがexactに解決しなければならない。Task 0のTrust Provisioning ReceiptはこれらをAuthorizationからbyte-exactに継承し、発行時にも四pointerを再検証する。old head／別profile／generation mismatch、stale local/global revocation、quarantined／expired readiness、rotation／recovery activation chain不成立を個別diagnosticで拒否する。

Rotationはold／new Root Profileの対応purpose quorumが同一Rotation payloadを署名し、overlap中のread-back後にだけ新Headを`N+1`へCASする。next Root Profileの`allowed_purposes[]`は既定でpreviousとset equalityとし、変更時だけold／new quorumが同時署名するclosed purpose-change manifest ref／hashをRotation payloadへ必須とする。Recoveryではpurpose set変更を禁止しpreviousとexact set equalityにする。rollbackは新しい逆向きRotationとしてpolicy window内のold＋new purpose quorumを満たす場合だけ許し、pointerの巻戻しは禁止する。期限後はRecoveryだけを許す。Recoveryは後述のclosed `recovery_mode` branch、独立recovery custodian quorum、next Root Profileの`offline_governance_root_recovery_new` purpose quorum、所定cooling-off、post-recovery auditを全て満たす完成Recovery Recordで新generationを発行する。通常worker、AI、単一管理者、失効Keyだけでrotation／recoveryできない。Authorization／Provisioning Receiptは発行時Root Head／generation、validity、local Revocation Head／Snapshot、global Super Head／Ledger、status=validのRecovery Readiness Headを固定する。使用時点の期限切れ、scope外Task／path／operation、wrong tree／design／plan hashを拒否する。

Rotation／Recovery時刻はcanonical UTC秒へ正規化してepoch-second算術し、全duration Fieldをpositive safe integerとする。Rotationは`overlap_ends_at = overlap_starts_at + policy.overlap_seconds`、`overlap_starts_at <= old/new threshold signing_subject.issued_at <= rotation_activated_at < overlap_ends_at`、`rollback_deadline = rotation_activated_at + policy.rollback_window_seconds <= overlap_ends_at`を全て必須にし、Root Head `activated_at=rotation_activated_at`とする。`overlap_seconds >= ceremony_budget_seconds + cas_retry_budget_seconds`、`minimum_rotation_lead_seconds >= RecoveryPolicy.total_compromise_cooling_off_seconds + ceremony_budget_seconds + cas_retry_budget_seconds`とし、Profile current化時に`profile.valid_until - publication_time >= minimum_rotation_lead_seconds`を要求する。Rotation監視は遅くとも`valid_until - minimum_rotation_lead_seconds`に開始し、deadline超過でProduction新規operation／Authorizationをfail closedにする。Recoveryは`cooling_off_started_at = RootSecurityIncident.confirmed_at`、適用秒数をpartialなら`cooling_off_seconds`、totalなら`total_compromise_cooling_off_seconds`とし、`recovery_activated_at >= cooling_off_started_at + applicable_seconds`、全Recovery signature `issued_at <= recovery_activated_at`、candidate next Profile `issued_at <= recovery_activated_at < valid_until`、Root Head `activated_at=recovery_activated_at`を必須にする。

`publication_time`は呼出元時刻または予約済みactivation時刻ではなく、single-writer pointer storeがexpected-parent比較とpointer set置換を不可分に行うCAS linearization pointでauthenticated clockから取得するcanonical UTC秒である。store journalはtransaction digest、expected／next pointer exact set、当該試行でfreshに取得した`publication_time`、commit／reject結果、read-back digestを同じatomic recordとしてimmutable保存する。全four／six-pointer publicationはcandidate Root Profile、current/candidate local・global Revocation Head／Snapshot／Ledger、candidate Readiness Receipt／Headを`publication_time`で再検証し、Readiness `status=valid`、non-expired、effective Invalidationなしを必須にする。initial four-pointerはgeneration 1 Profile、Rotation six-pointerは`rotation_activated_at <= publication_time < overlap_ends_at`かつold／new Profileがvalid、inverse rollbackはさらに`publication_time <= original rollback_deadline`、Recovery six-pointerは`recovery_activated_at <= publication_time < next_profile.valid_until`でなければ副作用0で拒否する。

CAS retryはcandidate canonical bytesとactivation時刻をwindow内で再利用できるが、前回試行の`publication_time`またはjournal commit時刻を再利用せず、各試行でfresh actual time、全期限、local／global revocation、Readiness、expected parentsを再評価する。CAS失敗後に同candidate bytesを再利用できるのは上記window内だけで、window終端以後はjournal transactionをterminal abortにし、新activation時刻、新candidate payload、全必要署名から作り直す。同時刻／同bytesの保存を期限後publish権限にせず、payload時刻差替え、overlap前／終了後publication、rollback deadline後publication、Profile／Readiness期限後Recovery、cooling-off前activationを拒否する。

Offline Governance Evidenceはbare refでなく`OfflineGovernanceEvidenceBindingV1`だけを使う。各Policyの`evidence_kind_schema_bindings[]`はevidence kindを一意keyとしてunsigned UTF-8 byte順・重複なしとする。Rotation Policyのbinding key集合はrequired ceremony集合とrequired purpose-change risk集合のunion、Recovery Policyはrequired incident／post-audit／readiness-drillと全reason rowのunionとset equalityにし、各bindingのrefを解決した完成wrapperのroot schema IDを`evidence_schema_id`へ一致させ、隣接hash、署名／attestation、closed payloadの`evidence_kind`、`status=passed`、`subject_sha256`、`completed_at`を検証する。`subject_sha256`は`subject_projection_schema_id`が定義するself-referenceを含まないcanonical projectionから再計算し、consumer payloadから次表どおり導出したexpected projectionのJCS hashとbyte-exact一致させる。

| Evidence consumer | expected subject projectionのexact Field | consumer time |
|---|---|---|
| Root Rotation ceremony | trust domain、previous／next Root Profile ref／hash、Rotation Policy ref／hash、nullable Purpose Set Change Manifest ref／hash、overlap start／end、activation、rollback deadline | `rotation_activated_at` |
| Root Purpose Set Change risk | previous／next allowed purpose exact set、changed purpose、added／removed kind、subject schema、Owner document ref／hash、Architecture Approval ref／hash | Manifest `generated_at` |
| Root Security Incident | affected Root Head ref／hash／generation、affected Key ID exact set、confirmation kind、incident kind、detected／confirmed／compromise-effective time | `confirmed_at` |
| Root Recovery post-audit | recovery mode、previous Root Head ref／hash／generation、previous／next Root Profile ref／hash、Recovery Policy ref／hash、source Global Head／Ledger ref／hash、Incident ref／hash／kind／effective time、cooling-off start、activation | `recovery_activated_at` |
| Recovery Readiness drill | Root Head ref／hash／generation、Recovery Policy ref／hash、nullable remediation Invalidation ref／hash、recovered public-state hash、`secrets_exposed=false` | Readiness `issued_at` |
| Recovery Readiness Incident | target Readiness Head／Receipt ref／hash、affected Root Head ref／hash／generation、incident kind、detected／confirmed time | Incident `confirmed_at` |

Recovery post-auditはself-referenceを避けるため完成Recovery／candidate Root Head／destination Trust closureをprojectionへ含めない。後段`TrustRegistryRecoveryAuthorizationV1`が完成Recoveryからcandidate Root／local／global／Readiness／destination Trust closureへのexact一方向bindingを所有し、audit単体をcandidate publication authorityへ使わない。Rotation ceremony evidence kind集合は`required_ceremony_evidence_kinds[]`、Root Security Incident evidenceは`required_incident_evidence_kinds[]`、Recovery post-auditは`required_post_recovery_audit_kinds[]`、Readiness Receipt drillは`required_readiness_drill_evidence_kinds[]`とそれぞれset equalityにする。Readiness Incidentのreason row集合は四reason codeとset equality、該当rowのkind集合とIncident evidenceをset equalityにする。各Evidenceは`completed_at <= consumer_time`かつ`consumer_time - completed_at <= max_age_seconds`で、同kind Policy bindingのpositive safe integer値を使う。全Evidence `completed_at`は消費payloadのthreshold signature `issued_at`以下、Rotation／Recoveryではさらにactivation時刻以下とする。unknown kind／schema、kind重複、missing／extra kind、consumer cross-field差、期限切れ、hash-only、自由URL／Markdown、failed／future Evidenceを拒否する。

Recovery Policyの`triggering_incident_kinds[]`はRoot Security Incident enum `key_compromise | custody_loss | policy_violation | threshold_loss | total_root_compromise`の重複なしnon-empty subsetだけを許す。`total_compromise_incident_kinds[]`はそのsubsetかつ`threshold_loss | total_root_compromise`の重複なしsubsetとし、Productionではnon-empty、`total_compromise_cooling_off_seconds >= cooling_off_seconds`を必須にする。unknown incident、空trigger、total集合だけのextraを拒否する。`partial_compromise`はincident kindがtriggering集合かつtotal集合外、uncompromised Root signed recordがnon-nullでold Rootの対応purpose quorumを満たし、通常cooling-offを満たす。さらにRecovery署名前のcurrent Global LedgerがIncidentの`affected_key_ids[]`全件を同じgeneration、reason mapping、Incident ref／hash、`effective_at=compromise_effective_at`でexactly once保持することを必須にし、その完成current Ledger／Super HeadだけをRecovery payloadのsource globalへ束縛する。これによりpartial branchがglobal pointerを継承しても、全historical verifierはcompromise effective timeから対象旧署名をquarantineできる。row不足、local-only失効、別Incident／effective time、Recoveryとglobal appendの同時未承認shortcutを拒否する。

`recovery_custodians[]`はkey ID順、`recovery_custodian_quorum.exact_key_ids[]`はそのKey集合とset equalityで、Key ID／SPKI／subjectを重複させない。Recovery wrapperの`independent_recovery_custodian_subject_refs[]`はcustodian threshold wrapperに含まれる全valid distinct signer KeyをPolicyでlookupして得たsubject ref集合とset equality、subject ref順・重複なし、要素数`>= distinct_subject_threshold`とする。signature側にはKey ID、payload側にはsubject refだけを置き、両者の集合を自由申告で二重管理しない。各signer Keyはexact key universe内、purpose専用、typed current custody Evidence付きで、Root custodian、worker、AI、通常管理者から独立しなければならない。wrong subject、同一subjectの複数Keyによる水増し、payload missing／extra subject、threshold未満、Productionのnon-hardware Key／stale custody Evidenceを拒否する。

V1のbranch totality規則として、上記subset条件に加えて`total_compromise_incident_kinds[]`を`triggering_incident_kinds[]`と`threshold_loss | total_root_compromise`のintersectionとのexact set equalityにする。これにより全triggerはpartialまたはtotalの一方だけへ必ずrouteされ、total集合のmissing／extra、両branch該当、branch無しを拒否する。

`total_compromise`はincident kindがtotal集合、old Root signed record=null、Recovery Policyでpre-committedされた独立recovery custodian quorumとnew Root quorum、total-compromise cooling-off、required incident Evidence、post-recovery auditを必須にし、old Root署名を要求または受理しない。両branchはRecovery payloadのsource global Super Head／Ledgerを開始時currentへexact一致させる。total branchではnew Rootがそのsourceへappendするnext Global Ledger／Super Headにold generationの全Root Keyをexactly once、Incident kindから上記mappingで一意に決まるreason code、同一Root Security Incident ref／hash、`effective_at=compromise_effective_at`で記録する。issuer-loss Rからrouteされた場合だけ、さらにterminal Detectionが指定する当該assurance issuer row一件をReadiness Incident ref／hash、mapped reason、同じfailure effective timeで加え、candidate diffを`old Root Key rows union assurance issuer singleton`とset equalityにする。通常total branchはRoot Key rowsだけとし、別issuer／extra rowを拒否する。candidate Root／local／global／Readinessとdestination Trust closureを束縛するRecovery Authorization、そのAuthorizationを参照するcandidate Trust Head／Catalog Headが全て完成し、Root／local／global／Readiness／Trust／Catalog六current pointerを一つのexpected-parent CASでpublishするまでnew Rootまたはcandidate closureをcurrentにしない。partial branchのold Root signed record欠落、total branchのold Root signature混入、custodian／new quorum不足、partial affected-key global row欠落、全old Key失効rowまたはissuer-loss singleton欠落、source global drift、単一管理者によるcatastrophic re-genesisを拒否する。

Recovery publicationはpartial／totalとも`recovery_activated_at <= publication_time < previous Root Profile.valid_until`かつ`publication_time < next Root Profile.valid_until`を必須にし、previous Profile expiry後に保存済みcustodian署名やcandidate bytesをpublishしない。continuity window内にcooling-off、ceremony、CAS retryを完了できない、またはprevious Profileが既にexpiredの場合は既存trust domainを進めずfull trust resetへrouteする。

Recovery custodian Key／material自体のcompromiseはV1のcontinuity-preserving Recovery対象外としてfail closedにし、既存trust domainのpointerを進めない。clean-roomで新trust domain ID、new Genesis Ceremony、new Root／Recovery custodian set、全Trust／Product authorityの人間による再承認を行う明示的full trust resetだけを許し、旧domainの履歴はquarantined archiveとして保持する。Recovery custodian quorumの自己確認や新Keyへの暗黙置換を拒否する。

`OfflineGovernancePurposeSetChangeManifestV1`は両purpose配列をunsigned UTF-8 byte順・重複なし、`changes[]`をpurpose順にし、previous／nextのsymmetric differenceとchangesをset equalityにする。各changeのtyped `risk_evidence[]` kind集合はRotation Policyの`required_purpose_change_risk_evidence_kinds[]`とset equalityで、上表のrow projection、schema、status、signature、independent issuer、freshnessを検証する。added purposeは完成schema、approved Owner bytes、独立R4 Architecture Approval、non-empty risk Evidenceを必須とし、removed purposeもnon-empty影響Evidenceを必須とする。manifest IDは同Fieldを除くJCS hashから`urn:mirakan:offline-governance-purpose-change:sha256:<lowercase-hex>`として導出する。purpose集合が同じ場合はRotation payloadのmanifest ref／hashを両方null、異なる場合は両方non-nullかつold／new profile、manifestとexact一致させる。片側null、empty change、bare ref、unknown schema／Owner／Evidence kind、risk集合missing／extra、差分漏れを拒否する。

Assurance Policyまたはそのkind bindingが指すTrusted Issuer Manifestのstable bytesが変わる場合、Rotation payloadの`assurance_static_policy_change_manifest_ref/hash`を両non-nullにし、完成`OfflineGovernanceAssuranceStaticPolicyChangeManifestV1`へ解決する。previous／next policy pairはold／new Root Profile、両manifest pair集合は各Policy `kind_bindings[]`から再導出し、`changes[]`をそのsymmetric diffとset equalityにする。各changeの`risk_evidence[]` kind集合はsource Rotation Policyのnon-empty `required_assurance_static_policy_change_risk_evidence_kinds[]`とset equality、Rotation Policyの`evidence_kind_schema_bindings[]` key集合はpurpose-change、static-policy-change、通常ceremonyでrequiredとなる全kind unionとset equalityにする。ceremony Evidenceにはsource Policyの固定`assurance_static_policy_change_invariant`とbyte equalityな完成`OfflineGovernanceAssuranceStaticPolicyChangeApprovalV1`をexactly one含める。

Approval subject projectionは完成Manifest、old／new Root Profile、Manifest change key集合、Rotation開始時current Trust Head／closureを閉じ、source Trust pairを同じ六pointer Rotationのsource Trust／Change Approvalとexact一致させる。signerはそのhealthy source Trustの指定R4 Role／singleton purpose Keyで、Root custodian、Manifest作者、issuer管理者、new custodianから`independent_human_r4`にする。`completed_at <= rotation_activated_at`かつ差がinvariant `max_age_seconds`以下、Approvalのrevocation snapshotがsource closure current Registryと一致することを必須にする。stable bytesが同じならManifest pairを両nullかつ当該Evidence 0、異なるなら両non-nullかつnon-empty change／risk Evidenceとし、Recovery中の変更、別Trust replay、destination Policyだけによる要件弱化、片側null、diff漏れ、自由URLを拒否する。

Root四pointer genesis current化後からTask 0のTrust／Catalog genesis CAS完了前まではhealthy source Trust R4が存在しないため、Assurance Policy／Trusted Issuer Manifest bytesを変えるRotationを禁止する。このwindowで許すRotationはstatic bytes同一かつ両Manifest Field nullの期限／Key rotationだけとし、static変更が不可避なら既存domainを継続せずnew Genesis ceremonyによるfull trust resetへ進む。将来のTrust承認を仮記入したりConstruction executor／Root custodianをR4へ読み替えない。

Construction AuthorizationはTask 0～10Bが同じ完成content IDを意図的に繰り返し参照するscope grantであり、single-use nonceを持たない。同じAuthorization bytesの再投入はidempotent read、同一Task ID／previous Receiptから同一完成Receiptを再投入する場合だけcontent-ID idempotent成功とする。同一branchへ別output、Task ID重複／順序違反、previous Receipt差、既完了Taskの再実行は拒否し、Task Receipt chain、input／output Git tree、changed-path manifest、sidecar expected-parentで実行replayを防ぐ。

各Taskは前Taskのpass Receiptとcurrent Authorizationをread-backし、Task IDとartifact rootがscope内で、`authorization.issued_at <= task.completed_at < authorization.valid_until`の場合だけconstruction executor keyでReceiptを署名する。Task開始／完了時にAuthorizationが固定したRoot Head、local Revocation Head／Snapshot、global Super Head／Ledger、status=validのReadiness Head、Trust Head／closureのnullable branch、Authority Catalog Head／Catalogからなるexact pointer snapshotをfresh read-backし、その時刻以前にeffectiveなRoot Key失効を適用する。一件でもref／hash、generation、sequence、status、expiry、expected-empty状態が変われば当該Taskをfail closed、全staged Git outputを破棄し、既存sidecar Receiptを再利用不能なorphan historyとして保持して、fresh current snapshotを束縛する新Authorization／Task 0から再開する。bootstrap中はsame-generationのdescendant更新であっても継続せず、古いAuthorizationや途中Taskだけの再署名で救済しない。Receipt wrapperはRepository Git tree外のimmutable content-addressed sidecar storeへ保存し、Task output treeは自分自身または後続Receipt bytesを含めない。これにより`output_git_tree_id`の自己参照を禁止する。Task 0の`input_git_tree_id`はAuthorizationの`authorized_base_git_tree_id`、Task 1～10と10Bは前Task pass Receiptの`output_git_tree_id`とexact一致し、outputはinputのGit descendant、unrelated dirty diffは0でなければならない。唯一の例外としてTask 10Aはprevious Receipt=Task 10、input=C、Task 10 output=Bをancestorとし、C\\Bが検証済み完成`ControlPlaneBootstrapApprovalV1`一件だけでなければならない。`changed_paths_manifest`は変更pathをrepository-relative slash形式、UTF-8 byte順、重複なしで列挙し、各pathがAuthorizationの当該Task typed allowlist内であることを検証する。Task 9 output=A、Task 10 output=B、Task 10A output=D、Task 10B output=Eとする。Task 9／10 ReceiptはA／B生成後、Task 10A ReceiptはD生成後、Task 10B ReceiptはE atomic publication後にsidecarへ発行する。`ConstructionExecutorSignedRecordV1.public_key_ref/hash`はIdentity RegistryでなくAuthorizationの`construction_executor_public_key_ref/hash`へ直接解決し、SPKI DER hash、subject、purpose、署名をexact一致させる。Task 10B例外は初回genesisと初回Control Plane completionだけで、通常WP、Capability、Risk、Release、Future claim、後続definition migrationを許可しない。negative fixtureはpurpose quorum threshold-1、wrong key universe、duplicate key／custodian、wrong algorithm／format／high-S、unknown／revoked／期限外Key、wrong purpose／subject、Root hash差、unsigned／wrong-purpose Root Head、Root activation時刻差、candidate Profileのpremature current／exception再利用、old generation、local genesisのnon-empty／previous non-null、stale local／global Revocation、global historical target不明、missing global Super Head、stale／quarantined Readiness Head、Incident／Invalidation target・reason差、remediation nullability／generation branch差、rotation片側quorum、recovery cooling-off未了／branch無し、custodian subject set差、Task missing／extra／duplicate branch、materialization plan missing／overlap／dependency inversion、path traversal、wrong base／input／output tree、non-descendant、dirty／unlisted path、Receipt-in-tree、wrong C diff、pointer snapshot drift、genesis／reuse nullability差、F missing／wrong closure／non-current／sequence branch／pointer rollback、DecisionとF ref／hash差、rejected／unsigned Decision、同じAへのDecision差替え、wrong design／plan、expired Authorization、同一Task branchへの別Receipt、`secrets_included=true`、authority binding missing／extra／duplicateを一原因ずつ検査する。

Executor署名入力は`UTF-8(JCS(ConstructionExecutorSigningSubjectV1))`をECDSA P-256 with SHA-256へ一度だけ渡す。`subject_ref/hash`はschema／purposeが示すID込み完成payload bytes、Receiptでは`issued_at=payload.completed_at`、Inventory／Seed／migration manifestでは`issued_at=payload.generated_at`、Signer subject／public key ref／hashはAuthorizationのexecutor同名Fieldとexact一致する。schemaとpurposeの組合せは上記同じ順の一対一mappingとし、domain／schema／purposeを変えたcross-schema replay、payload部分hash、二重hash、別Artifactへの署名流用を拒否する。

Pre-ceremonyでcurrent Root Profileが固定した完成`OfflineGovernanceSchemaCatalogV1`／全memberを先にoffline verifierでread-backし、そのclosureでConstruction Authorization schemaとRoot quorumを検証する。Authorizationが別途束縛する完成`LocalSchemaCatalogV1`、全member schema bytes、`BootstrapSchemaMaterializationPlanV1`、`AuthorityBindingSourceCatalogV1`／初回HeadはRoot purpose quorumの明示対象とし、Local Catalogのoffline member subsetをRoot Profileのoffline catalogとbyte-exact set containment検証する。Materialization Planのtask ID集合は`task-0 | task-2`とset equality、両`schema_ids[]`は相互disjoint、そのunionはfull Local Schema Catalog member ID集合とset equalityでなければならない。各`$ref`依存はconsumerと同じTaskまたは早いTaskへ割り当て、Task 0 schemaがTask 2 schemaへ依存するplanを拒否する。Task 0は両catalog、plan、schemaを生成・編集せずBootstrap Verifierでexact bytesをread-backし、planのTask 0 subsetだけをRepositoryへbyte-exact materializeする。

`trust_bootstrap_state=expected_empty_genesis`はTrust／Catalog current pointerが共にemptyで、Authorizationのcurrent Trust Head／closure四Fieldを全nullとする初回だけ許す。Task 0は選択assurance profile用identity／Role／assignment、purpose別non-exportable public key、Trust revocation genesis／current snapshot、Policy Service bootstrap configをprovisionし、Root／local／global／Readiness bytesはAuthorizationからread-only継承する。Root quorumは`provisioning_kind=construction_genesis`、current Trust四Field=nullのProvisioning Receiptへ署名する。Task 0はReceiptと同じ15 member bytesのcandidate closureを作り、そのclosure内のdestination head publisher binding／singleton purpose Keyで初回Trust Headを署名し、pre-ceremony完成Catalog genesis Headと共にTrust／Catalog expected-empty pointer CASを一度だけ行う。candidate publisher署名はRoot-signed Receipt、Catalog trust slot、六Registry closureを先にbyte-exact検証した場合だけ候補検証でき、CAS後はcurrent closureのnormal Trust verifierで再検証する。

`trust_bootstrap_state=current_closure_reuse`はAuthorizationのcurrent Trust Head／closure四Fieldを全non-nullとして、current Trust／Catalog pointer、Head、15-member closure、Catalog、六Registryをbyte-exact束縛するrestart専用branchである。Task 0はRegistry／Head／pointerを変更せず既存authority setをread-backし、Root quorumが`provisioning_kind=current_closure_reuse`かつ同じcurrent Head／closureを持つfresh Provisioning Receiptへ署名するだけとする。必要なTrust／Catalog変更はbootstrap外の通常Change Approval＋atomic CASを先に完了し、その新currentを束縛するfresh Authorizationから再開する。reuse branchでexpected-empty genesis、revision再発行、candidate publisher例外、Catalog単独更新を再利用しない。

両branchの`required_authority_bindings[]`はAuthority Catalogの`authority_source_kind=trust_registry` slotだけから導出したexact六tuple集合とset equalityにする。snapshot／transition／WP lifecycle／definition migration／row-migration-manifest／rebaseline transaction各publisher、Product Operational Decision R4、Future portfolio R4、Future claim R4、Future→active manifest publisher、Bootstrap／Rebaseline approverを含み得るが、この例示を固定allowlistにせず全trust-registry slotのmissing／extraをtestする。offline Root、Recovery custodian、construction executor direct slotは除外する。Task 2はPlanのTask 2集合だけをbyte-exact materializeしてAjv compile／semantic verifierを実装し、schema／catalog／plan bytesを変更しない。差が必要なら新Authorization／Task 0から再開する。不要な強権限Role、複数purpose Keyを拒否する。Task 1～10Bは専用sandboxとper-Task construction Receiptだけを使う。synthetic test identity／fixture keyをProductionへpromoteせず、Role、assignment、purpose key、revocation chain、independence、Policy read-backの一件でも欠ければ最終Approvalへ進まない。

pointer drift規則の唯一のpositive transitionは`expected_empty_genesis` Task 0自身によるTrust／Catalog expected-empty→上記完成Head pairのatomic publicationである。Task 0開始時はAuthorizationのpre-empty snapshot、Task 0完了時はRoot／local／global／Readiness四pointerが不変かつTrust／CatalogがTask 0 output manifestのexact pairへ一度だけ遷移したことを検証し、そのpost-publication六pointer snapshotをTask 0 Receiptへ束縛する。Task 1以降はAuthorizationのpre-empty値でなく当該Task 0 Receipt snapshotを固定比較する。`current_closure_reuse` Task 0は六pointer全て不変を要求する。任意Head、片側だけ、別Catalog、Task 1以降のdescendantへ進む変化はdriftとしてrestartし、genesisの正規transitionをdrift扱いするfixtureと任意transitionを正規扱いするfixtureの双方を拒否する。

ここでTask 0のRecovery Readiness「provision」は、pre-ceremonyで既に完成・署名・current化されたReceipt／HeadをTrust closureとProvisioning Receiptへ登録してread-backする意味であり、Authorization署名後に初回Readiness bytesを生成する意味ではない。初回Receipt／Head、四current pointer、Authorizationの順を逆転させず、Task 0はそのbytesを変更しない。

`task_artifact_scopes[]`のtask ID集合は`allowed_task_ids[]`とset equalityで、各`allowed_roots[]`はrepository-relative slash形式、空・絶対path・`..`・symlink escape・case-fold collision・重複を拒否する。Task Receiptのchanged pathsは対応rowのroot配下とset containmentでなければならず、親Task名によるprefix許可や別Task scopeのunionを行わない。

`TrustRegistryClosureV1`はControl Planeが所有するclosed hash closureで、上記15 member ref／hash（Root Head／Profile、generation-local Revocation Head／Snapshot、global Revocation Super Head／Ledger、Recovery Readiness Head／Receipt、Authority Binding Source Catalog、六つのTrust Registry）をexactly once持つ。`closure_id`は同Fieldを除くobjectのJCS SHA-256から`urn:mirakan:trust-registry-closure:sha256:<lowercase-hex>`として導出し、各ref解決、隣接hash、Root Head→Profile、Root generationとlocal Revocation／Readiness generation、global authorizing Root Head chain、Catalog bytes、registry間Identity／Role／assignment／Key参照、Trust revocation current head、Policy purpose集合をset-equality検証する。filesystem列挙、`latest`、部分closureを禁止する。Task 0 genesis branchはProvisioning Receiptと同じmember bytesから初回closureを生成し、reuse branchはReceiptとcurrent完成Trust Headが指す既存closureのmember bytes set equalityをread-backして新closureを生成しない。rebaselineもcurrent完成Trust Headが指すclosureとcurrent Authority Catalog Headが指す同じCatalog bytesだけをread-backする。

Bootstrap後のhealthy local／global Root Revocation Head更新と通常Recovery Readiness renewalは、destination 15-member Trust closureとexact `TrustRegistryChangeManifestV1`／Approvalを先にstagingし、該当Root側current pointer、Trust Head/current pointer、同Catalogを再束縛するCatalog Head/current pointerの三pointerを一つのatomic CASで進める。confirmed Incidentによるemergency Invalidationは§6のreason別containment CASだけを使い、Trust／Catalogをhistorical sourceのまま保持して全通常operationを停止する。その解除は通常Change Approvalでなく完成`OfflineGovernanceRecoveryReadinessRemediationAuthorizationV1`または既存Trust／Root Recovery Authorizationを使う。通常Root Rotationはcandidate Root／local／Readiness、global expected-current、destination closure／Change Approval／Trust・Catalog Headを完成して六pointer CASし、Readiness remediation Rotationは専用Authorization DAG、Root Recoveryは§6のdestination-only Recovery Authorization DAGで六pointer CASする。Root pointerだけ先行したstale closure、closureだけ先行したstale Root pointer、global ledger reset／rollback、Readiness期限延長／status変更のin-place mutationを拒否する。

六つのRegistryは本設計をschema Ownerとするclosed objectで、`entries[]`を主logical IDのunsigned UTF-8 byte順、nested集合もunsigned byte順・重複なしとする。ref整合、Role permission↔purpose、assignment subject／Role、Key subject／Role／purpose、validity、revocation target、Policy schema refを全closureで検証する。初回`TrustRegistryClosureHeadV1`だけprevious=null／sequence 1／`authorization_kind=construction_genesis`で完成`TrustProvisioningReceiptV1(provisioning_kind=construction_genesis)`を束縛し、`signer_handoff=destination_only`、source signature=null、candidate closure内destination head publisher signatureを必須にする。candidate verifierはRoot quorumのReceipt、Receiptとcandidate 15-member closureのexact projection、Catalogのdestination trust slot、publisher Identity／Role／assignment／singleton purpose Keyを先に検証し、Trust／Catalog expected-empty CAS成功後にnormal current verifierで同Headを再検証する。この例外はsequence 1に一度だけで、Change Approvalまたはreuse branchへ流用しない。通常変更はcurrent completed Headをprevious、exact `N+1`、`authorization_kind=approved_change`、fresh `TrustRegistryChangeApprovalV1`を必須にする。quarantineからのclosed回復は三kindだけで、`root_backed_trust_recovery`は完成`TrustRegistryQuarantineRecoveryAuthorizationV1`、current Rootで有効なdestination publisher、Trust／Catalog／Readiness三pointer CAS、`root_recovery`は完成`TrustRegistryRecoveryAuthorizationV1`、new Rootで有効なdestination publisher、六pointer CAS、`recovery_readiness_remediation`は完成`OfflineGovernanceRecoveryReadinessRemediationAuthorizationV1`、同Authorizationが指定するcurrentまたはnew Rootで有効なdestination publisher、mode別三／六pointer CASを必須にする。三kindとも`signer_handoff=destination_only`、source signature=nullとし、branch間Authorization型／Root generation／pointer set流用を拒否する。head publisherは`service.trust-registry-head-publisher`／`role.trust-registry-head-publisher`／singleton purpose key、Trust current pointerとAuthority Catalog current pointerは同じexpected previousを使うatomic CASの一部とし、old closure、branch、gap、unsigned registry、自己承認を拒否する。rebaselineの`current_trust_registry_closure_ref/hash`はcurrent完成Headが指すclosure、current Authority Catalog Headはそのclosure内のCatalog bytesを指すものだけを許し、呼出元申告またはfilesystem `latest`を使わない。

`root_backed_trust_recovery`はRoot Head／local＋global revocationがhealthy currentで、上記Readiness emergency quarantine CASがTrust Incidentをcurrent化した場合だけ許す。IncidentはRoot ProfileのGovernance Recovery Policy ref／hashと当該Root／local／global／source Readiness全pairをpayloadへ束縛し、Root Profile、current pointer、typed Evidence subjectとexact一致させる。`affected_targets[]`はtarget-kind tuple順・重複なし、non-emptyとし、incident kindごとに、closure compromise=source 15-member closureのEvidenceでcompromisedと確認された全member exact set、publisher compromise=current Trust Headの実destination／source slotから解決したsubject・Role・assignment・Key composite exact singleton、Catalog compromise=source CatalogのEvidenceでcompromisedと確認された全`{schema_id,signature_slot_id,member_sha256}` exact setだけを許す。kind外target、source bytesに存在しないtarget、Evidence projectionとのmissing／extraを拒否する。

完成Root-signed Quarantine Recovery AuthorizationはIncidentから同Policy／Root四pairとaffected target集合をbyte-exact継承し、destination 15-member closure／Catalogとfresh remediation Readiness Receipt／valid Headを束縛する。destinationはclosure-member targetを全てreplaced、publisher targetのIdentity／Role／assignment／Keyをdisabled／revoked／removedして独立new publisherへ置換、Catalog targetを全てremovedまたはmodifiedしてapproved schema／derivation bytesへ置換しなければならない。targetから参照整合上必須となるrow／closure member、fresh Readiness、new publisherの最小追加差分だけをclosed remediation closureとして再計算し、それ以外のunaffected Registry／Catalog rowをbyte-exact保持する。追加変更はGovernance Recovery Policyの別required Evidence kindとRoot quorum承認を明示的に持つ場合だけunionし、自由なdestination再構成を許さない。Trust／Catalog Headはprevious=current quarantined source Head、sequence exact `N+1`、source signature=null、destination-only publisher signatureを持つ。candidate destination signerはRoot Authorizationとcandidate closureで検証し、Trust／Catalog／Readiness三pointer expected-parent CAS後にnormal current Trust verifierで再検証する。Root／local／global pointer、Product state、baseline bindingをこのtransactionで変更せず、後続の専用rebaseline authorizationまで通常operationを停止する。compromised target残存、unaffected row差、quarantined Trust Registry内のPolicy差、別Root／Readiness、Incidentからのpair差、Root pointer drift、Readiness単独解除を拒否する。

六Registryの`registry_id`集合はschemaに示した六URNとset equality、初回revisionは1、以後はentry bytesにdiffがある場合だけexact `N+1`とし、同bytes revision bump／同revision別bytesを拒否する。Identityの`subject_ref`とRoleの`role_ref`はAuthority Catalogの完成derivation policyまたはcatalog memberのexact Role値からだけ生成し、表示名／email／配列順から推測しない。`assignment_id`は同Fieldを除くclosed assignment row JCS SHA-256から`urn:mirakan:authority-role-assignment:sha256:<lowercase-hex>`、Public Key `key_id`は§6.0.1のSPKI rule、`revocation_id`は同Fieldを除くclosed revocation row JCS SHA-256から`urn:mirakan:authority-revocation:sha256:<lowercase-hex>`として導出する。Governance Policy `policy_id`は`policy_schema_id`が所有するtagged `content_derived | fixed_logical` ID規則を検証し、前者はprojection hashを再計算、後者はschemaに列挙したexact logical IDへ一致させる。`policy.control-plane.governance-recovery.v1`と`policy.trust-registry-change-evidence.v1`はfixed logical branchである。同じpolicy IDのdestination bytes変更はRegistry revision exact `N+1`、Trust Change Manifestのmodified row、source／destination ref／hash、必要Evidence／Approvalを必須とし、old/new ref混在やin-place bytes置換を拒否する。各Registryの主keyは順にsubject_ref／role_ref／assignment_id／key_id／revocation_id／policy_idでuniqueとし、Catalogから導出したrequired subject／Role／assignment／purpose-key集合とのmissing／extra／duplicateを拒否する。

Trust Headの`head_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:trust-registry-closure-head:sha256:<lowercase-hex>`として導出し、全required signatureはID込み完成payloadをsubjectとする。`authorization_kind`ごとのauthorization型、signature presence、handoff enumは上記五branch以外を許可しない。Catalog HeadとTrust Headはそれぞれ別sequence chainを持つため数値一致を要求しない。`approved_change | root_backed_trust_recovery | root_recovery | recovery_readiness_remediation`では同じatomic publicationのauthorization ref／hash、destination Catalog ref／hash、recorded_atをbyte equalityにする。`construction_genesis`だけはCatalog Head authorization=完成Root Head、Trust Head authorization=完成Provisioning Receiptと型が異なるためref／hash／recorded_at equalityを要求しない。両authorization chainが同じRoot／Readiness transitionとCatalog ref／hashを束縛し、`CatalogHead.recorded_at <= TrustHead.recorded_at <= publication_time`、expected-empty CAS journalのexact next pair／publication time／read-back digestを満たすことを要求する。

Revocation `reason_code`は`administrative_offboarding | key_rotation | credential_compromise | policy_violation | signed_record_invalidation`のclosed enumで、`subject_kind`ごとのrefはそれぞれIdentity、Role、Assignment、Public Key registry entryまたは完成signed wrapperへexact解決する。closed provenance unionは、通常revocation=`recovery_record`／`governance_incident`両pair null、Root Recoveryで置換するKeyの`credential_compromise`=完成`OfflineGovernanceRootRecoveryV1` pairだけnon-null、healthy Control Plane recoveryのKey `credential_compromise`またはsigned record `signed_record_invalidation`=完成`ControlPlaneGovernanceIncidentV1` pairだけnon-null、の三branchだけとする。片側null、両provenance non-null、他reasonとのprovenance、incidentのaffected target／kind／effective timeに一致しないrow、missing targetを拒否する。

Trust change ApprovalとHeadはsource current Trust Head／closure／Authority Catalog Head／revocation snapshot、destination closure／Catalog bytes、完成Change Manifestを署名対象へ含め、recorded時点のsource closureで歴史検証する。Approvalは未生成のdestination Trust Head／Catalog Headを参照せず、完成Approvalを後段の二Headがauthorizationとして参照する。destination revocationは対象recordごとに、そのrecordの実signature slotが解決したKey／signed wrapperとrow targetを一致させ、`effective_at <= record.issued_at／recorded_at`の場合だけ当該recordをretroactively invalid、後なら当該Keyの新規使用だけを停止する。Trust Headをquarantineするのは、当該Trust Head、authorizationであるChange Approval／Recovery Authorization、同atomic publicationのCatalog Head、またはそれらのrequired signer Keyをexact targetとするretroactive rowだけである。Control Plane baseline Approval／Transaction／Product operational snapshotまたはその専用signerだけをtargetとするrowはTrust closureをhealthyのまま保ち、該当baseline bindingだけをquarantineする。destinationがhead publisherのsubject／Role／Keyを失効または置換する場合は`signer_handoff=source_and_destination`とし、同一payloadへsourceとdestinationのpurpose専用Keyが二重署名する。他変更は`source_only`かつdestination signature=null、組合せ不一致を拒否する。

`TrustRegistryChangeManifestV1.manifest_id`は同Fieldを除くobject JCS hashから`urn:mirakan:trust-registry-change-manifest:sha256:<lowercase-hex>`として導出する。`closure_member_changes[]`は15のtyped closure member keyとset equalityで、source／destination exact ref／hashを`retained | replaced`へ一度ずつ分類する。`registry_entry_changes[]`は六Registryのsource／destination entry union、`authority_binding_member_changes[]`は両Catalog member unionを`{schema_id,signature_slot_id}`でset equalityにし、added／removed／retained／modifiedのnullabilityとentry hashを再計算する。

Trust change Evidenceはbare refでなく`TrustRegistryChangeEvidenceBindingV1`だけを使う。Manifestの`change_evidence_policy_ref/hash`はsource current closureのGovernance Policy Registryにあるstable logical ID `policy.trust-registry-change-evidence.v1`の完成`TrustRegistryChangeEvidencePolicyV1` pairとexact一致させ、呼出元またはdestination Policyだけで差替えない。`effect_requirements[]`はprivilege三値×independence三値の九tupleとset equality、各required／risk kindをkind順・重複なし、`evidence_kind_schema_bindings[]`のkey集合を両kind集合のunionとset equalityにする。各Evidence refはPolicy指定schemaのclosed signed wrapperへ解決し、kind、`status=passed`、subject projection hash、issuer／Role／Key purpose、`completed_at`、current revocationを検証する。subject projectionはManifest source／destination closureとCatalog、row kind／logical keyまたはschema＋slot、source／destination entry hash、change kind、privilege／independence effectをexactに含め、Approval Evidenceでは完成Manifest ref／hashとsource／destination closure／Catalogを含める。`completed_at <= Manifest.generated_at`またはApproval `issued_at`、consumer timeとの差が`max_age_seconds`以下でなければならない。

`privilege_effect`／`independence_effect`はManifest作者の申告でなくbootstrap verifierのclosed classifierでsource／destination rowから再計算し、申告値とexact一致させる。Role permission／purpose、assignmentの追加・scope拡張・validity延長、Key追加・purpose拡張・validity延長、Identityのdisabled→active、Governance Policyの権限上限緩和、Catalog signature slot追加・authority source／Role変更・derivation拡張は`expanded`、逆方向だけなら`reduced`、追加と削除が混在または比較不能なら保守的に`expanded`とする。独立subject／Role／class数の減少、作者・対象・Approver集合の交差増加、derivationで同一subjectへ収束、比較不能なindependence変更は`weakened`、逆だけを`strengthened`とし、semantic Fieldがbyte-exactな場合だけ`none`を許す。型別privilege／independence Field集合とlatticeはschema extensionに固定し、Policy自身では変更できない。

各change rowの`evidence[]` kind集合は該当九tupleのrequired集合、`expanded | weakened`を含むrowの`risk_evidence[]`は同tupleの`required_risk_evidence_kinds[]`とset equalityかつnon-empty、他rowはPolicy rowどおりの集合とする。ApprovalはManifestと同じsource Policy ref／hashを継承する。権限拡大またはindependence弱化が一件でもあれば、source Policyの`independent_r4_approval_invariant`をschemaに固定された上記exact定数とbyte equality検証し、完成`TrustRegistryChangeIndependentR4ApprovalEvidenceV1`をrequired risk Evidenceへexactly one含める。そのsubject projectionは完成Manifest、source／destination closure／Catalog、全expanded／weakened change key集合とset equality、signerはsource current `role.trust-registry-change-independent-r4-approver`への`independence_class=independent_human_r4` assignmentとsingleton purpose Keyを持ち、Manifest作者、変更対象subject、final Trust Change approver、head publisherの全subject／independence classから独立させる。このinvariantをPolicyから削除・空化・別schema／purpose／Role／class化できない。Policy row自身を変更する場合はsource requirementsとcandidate destination requirementsのunionに固定invariantを加えて分類・検証し、destination Policyによる自己弱化を防ぐ。自由URL／Issue／Markdown、hash-only、unknown kind／schema、missing／extra／duplicate／stale Evidence、subjectの別row流用、effectの`none`偽装、retained bytes差、差分省略、未列挙entryを拒否する。

quarantined sourceから通常Change Approvalを発行してはならない。`TrustRegistryRecoveryAuthorizationV1`は完成confirmed `OfflineGovernanceRootSecurityIncidentV1`、quarantined source Trust Head／closure、同じincident ref／hashを束縛する完成`OfflineGovernanceRootRecoveryV1`、そのRecoveryがactivateする完成candidate Root Head／generation、新generationのcandidate local Revocation Head／Snapshot、Recovery mode規則どおりsourceから継承またはtotal-compromise appendされたcandidate global Super Head／Ledger、candidate Rootを束縛するstatus=validのcandidate Readiness Head／Receipt、destination closure／Catalog bytesをexact一致させ、new Root Profileのpurpose `trust_registry_recovery_authorization` quorumで署名する。後段Trust HeadとCatalog Headはこの完成Authorizationを`authorization_kind=root_recovery`で参照し、source signature=null、destination publisher signature必須、sequence exact `N+1`、previous=current quarantined Headとする。Recovery wrapper→candidate Root／local／global／Readiness→destination closure／Catalog→Recovery Authorization→Trust Head／Catalog Headの一方向DAGを全てprepublish検証し、Root／local／global／Readiness／Trust／Catalog六current pointerを一つのexpected-parent CASでpublishする。partial handoff、source再署名、別Incident／Recovery、global ledger reset、old Root signature、destination closure差、片側pointer更新を拒否する。

genesisでRotation eventを捏造しない。Provisioning Receiptの`rotation_policy_ref/hash`はRoot Profileと同じapproved policyを指し、`recovery_readiness_receipt_ref/hash`は完成`OfflineGovernanceRecoveryReadinessReceiptV1`、`recovery_readiness_head_ref/hash`はそのReceiptを指す完成Headとする。Readiness ReceiptのRoot bindingはclosed二branchとし、genesis／Rotation／Recoveryのsequence 1は完成candidate Root Head／generation、same-generation renewalはcurrent Root Head／generationへexact一致させる。いずれもRecovery Policy、隔離drill Evidence、復元したpublic state hash、有効期限、`secrets_exposed=false`をRoot Profileのpurpose `offline_governance_recovery_readiness` quorumで署名する。各IDは同ID Fieldだけを除くpayload JCS SHA-256から導出し、prefixをReceipt=`urn:mirakan:offline-governance-recovery-readiness:sha256:`、Head=`urn:mirakan:offline-governance-recovery-readiness-head:sha256:`、Incident=`urn:mirakan:offline-governance-recovery-readiness-incident:sha256:`、Invalidation=`urn:mirakan:offline-governance-recovery-readiness-invalidation:sha256:`へexact固定する。

初回または正常renewal Headは`status=valid`、invalidation ref／hash=nullとする。初回はprevious=null／sequence 1、同generationの通常renewalはprevious=current完成Head／exact `N+1`／issued_at単調増加の新drill Receiptをpurpose `offline_governance_recovery_readiness_head` quorumで署名し、Receiptのremediates-invalidation両Fieldをnullとする。Root generation変更時はprevious=nullのnew-generation sequence 1 valid Headをcandidateとして完成する。initial bootstrapは§6 pre-ceremonyの四pointer CAS、稼働後Rotation／Recoveryはdestination Trust／Catalog Headまで完成した§6の六pointer CASでのみcurrent化する。実secret、実Recovery実行、rotation済みとのclaimを含めず、`valid_until <= evaluation_time`では新しいvalid HeadがcurrentになるまでProduction Root運用と新規authorization／rebaselineをfail closedにする。

Recovery Readiness Invalidationは次の三branchだけを許す。

| invalidation kind | exact incident／reason | signature authority／purpose |
|---|---|---|
| `readiness_assurance` | completed `OfflineGovernanceRecoveryReadinessIncidentV1`。四既存reasonはincident kindとexact一致 | 通常はpost-overlayでvalidなcurrent Root quorum／`offline_governance_recovery_readiness_invalidation`。issuer失効でRoot usability喪失時はvendor fail-stop source Root branch、またはunaffected Recovery custodian quorum／`offline_governance_root_recovery_custodian` |
| `trust_authority` | completed `TrustRegistryQuarantineIncidentV1`／`trust_authority_compromise` | current healthy Root Profile quorum／`offline_governance_recovery_readiness_invalidation` |
| `root_authority` | completed `OfflineGovernanceRootSecurityIncidentV1`／`root_authority_compromise` | incident confirmationがrootならcurrent Root Profile quorum／`offline_governance_recovery_readiness_invalidation`、recovery custodianならRecovery Policy quorum／`offline_governance_root_recovery_custodian` |

issuer失効Detectionは署名authorityを自称するwrapperではなく、nested cryptographic proofを再計算する論理IDなしcontent objectである。vendor branchはterminal revoked Observation pairと、そのObservationが指すresponse pairを全non-null、independent assessment集合empty、Manifestのvendor issuer／parser／DER chainとexact一致、`assurance_failure_effective_at=min(response.revoked_at, response.invalidity_at if non-null)=Observation.earliest_revocation_effective_at`とし、CRL／OCSP reasonを`key_compromise | ca_compromise | aa_compromise -> issuer_compromise`、`affiliation_changed | superseded | cessation_of_operation | certificate_hold | privilege_withdrawn | unspecified -> issuer_distrust`へclosed mappingする。offline branchはObservation／response pairを全null、Recovery Policy `assurance_issuer_detection_requirements[independent_offline_assessment].required_nested_evidence_kinds[]`とassessment kind集合をset equalityかつnon-emptyにし、typed assessmentが署名したeffective time／`issuer_compromise | issuer_distrust | policy_violation`を継承する。vendor required nested集合はemptyとする。Detection subject hashはManifest、issuer tuple、nullable source Trust pair、source Observation／responseまたはassessment exact set、global reason、failure／detected timeのclosed projectionから再計算する。payload `completed_at=subject_projection.detected_at`、`completed_at < valid_until`、Incident `confirmed_at < valid_until`、`confirmed_at - completed_at`がRecovery Policy schema bindingのmax age以下を必須にする。Trust current前はsource Trust四Fieldを全null、current後は四Fieldを全non-nullかつIncident開始時current healthy Trustと一致させ、片側nullや別Trust replayを拒否する。

`custody_assurance_invalid`でaffected issuer pairがnon-nullのReadiness Incidentは`evidence[]`へ完成Detection bindingをexactly one含め、Recovery Policyの同reason required kind集合はこのkindと他required kindのunionに一致させる。Incident projectionはDetection ref／hash、issuer subject／SPKI、`assurance_failure_effective_at`、revoked response ref／hashまたはoffline assessment set、source Root／Trustをbyte-exact継承する。Detection自体の`status=passed`は「失効検知検証がpass」、`finding=revoked`は対象issuerが失効した意味であり、通常Assurance Evidenceの`status=passed`／issuer usableと混同しない。Detectionのauthorityはvendor署名またはPolicy閾値を満たすindependent offline assessmentと後段Root／Recovery threshold Incidentであり、Trust R4はoptional reviewにしか使わない。

全branchでIncidentはcurrent valid Readiness Head／Receipt／Root Headと同じtarget、typed Evidence、`detected_at <= confirmed_at`を持ち、Incident／Invalidation／Headの`signature_authority`をbyte equalityかつ上表と一致させる。Incident root branchはpurpose `offline_governance_recovery_readiness_incident`、issuer-loss custodian emergency branchはPolicy-only purpose `offline_governance_root_recovery_custodian`を使い、Catalogの別signature slotで解決する。`readiness_assurance`の`assurance_failure_effective_at`はreason別typed Evidenceから、drill Evidenceならcurrent ReceiptでrequiredだったEvidenceの失効／revocation effective time、secrets exposureならconfirmed exposure occurrence time、recovered-state mismatchなら最初のsigned mismatch observation time、custody assuranceなら上記Detectionまたはcustody Evidence invalidity effective timeとして固定parserが再導出し、`Invalidation.effective_at`へbyte-exact継承する。`trust_authority | root_authority`は`effective_at=Incident.compromise_effective_at`、全branchで`effective_at <= detected_at <= confirmed_at <= Invalidation.recorded_at`とする。unknown source時刻、将来時刻、検知時刻への丸め、Evidenceより任意に古い時刻を拒否する。

続くHeadはprevious=current、sequence exact `N+1`、`status=quarantined`、同じReceipt、完成Invalidation、同じsignature authority、`Head.recorded_at=Invalidation.recorded_at`を持つ。`status=valid`は常に`signature_authority=root_profile_quorum`、`invalidation_ref/hash=null`とする。`signature_authority=recovery_custodian_quorum`は`status=quarantined`かつ、(a)`invalidation_kind=root_authority`／Incident kind `threshold_loss | total_root_compromise`、または(b)`invalidation_kind=readiness_assurance`／reason `custody_assurance_invalid`／affected issuer non-null／後述branch selectionでnormal Rootとvendor fail-stopの双方が不成立、のどちらかだけを許す。new Root回復後のvalid Headはnew Root Profile quorumだけが署名する。root-profile branchはpurpose `offline_governance_recovery_readiness_head`、custodian emergency branchはPolicy-only purpose `offline_governance_root_recovery_custodian`でHead payloadへ署名する。Incident→Invalidation→quarantined Headを完成して`root-readiness-current.ref`をexpected-parent CASしread-backするまでIncidentをcurrent quarantineとして扱わない。

成功後は通常Product／Trust change／AI／build／Release／新規Construction Authorization／通常Approval／通常rebaseline entrypointがfresh current Head `status=valid`を必須にして即時停止する。quarantine中の例外は、当該Incident／Invalidation／quarantined Headの検証、partial Root compromiseのGlobal Ledger append、該当Trust Quarantine／Root Recovery Authorization、candidate Root／Trust／Catalog／Readiness staging、reasonに対応するTrust・Catalog・Readiness三pointerまたはRoot・local・global・Readiness・Trust・Catalog六pointer recovery CASだけである。`ControlPlaneRecoveryRebaselineAuthorizationV1`はCAS後にcurrent Readinessがvalidとなりrecovered Trustをread-backしてから発行する。これ以外のoperation、quarantineを通常validと扱うshortcut、recoveryに無関係な変更を拒否する。historical signature／Approval／HeadはInvalidation effective time以後をquarantineし、それ以前をimmutable historyとして保持する。Trust／Rootがcompromisedでも旧Trust closureのReadiness refだけを信用しない。wrong incident type、reason／authority差、valid＋custodian authority、custodian KeyをRoot purposeとして検証、Issue URL／Markdown、片側更新を拒否する。

`root_authority`のうちroot-quorum確認されたpartial compromiseでは、Incident affected Key全件を`effective_at=compromise_effective_at`で追記したcandidate Global Ledger／Super Headを先に完成し、Invalidationのglobal quarantine pairをそのSuper Headへnon-null設定する。quarantined Readiness HeadとGlobal Super Headの二current pointerを同じexpected-parent CASでpublishし、Root／local／Trust／Catalogは不変、両read-back成功後だけcurrent quarantineとする。recovery-custodian確認されたtotal compromiseではglobal pairを両null、Readinessだけをemergency CASし、全old Keyのglobal appendはnew Rootによる六pointer Recovery transactionで行う。

`readiness_assurance`のissuer失効branchは、affected issuerからcurrent Root Profile／Head closureへ逆投影し、Root Key custody、Recovery custodian custody、Bootstrap Verifier reproduction、Genesis witness sourceの全Assurance Evidence collectionから当該issuerのEvidenceを除外して各minimum-distinct-issuer条件、Key usability、全purpose quorumを再計算する。単一issuer失効後の状態から次の優先順でexactly one branchを選ぶ。

1. **post-overlay normal Root:** 共通Verifier／Genesis Evidenceと、Global Ledger／Super Head／Readiness Invalidation／Headに必要な全Root purpose quorumがなおvalidなら、通常Root signatureで同issuerの`target_kind=assurance_issuer` rowとquarantined Readinessを完成し、Global＋Readiness二pointer CASする。
2. **vendor source-context fail-stop:** 1が不成立、Detectionがfresh vendor-signed revoked responseを持ち、source pre-CAS Rootの暗号Keyで専用`offline_governance_assurance_issuer_fail_stop` quorumとcandidate Global／Readiness各purpose quorumのsignature自体を検証できる場合だけ使う。candidate Global／Readinessを先に完成し、それら、source Root／local Snapshot・Head／Global／Readiness、Manifest／issuer、完成Detection、Incident、effective／recorded timeを束縛する完成`OfflineGovernanceAssuranceIssuerFailStopV1`を専用purposeで署名する。source-context verifierはこの**失効対象issuer由来Assuranceだけ**を三candidate wrapperの認可判定から除外する一回限りのrestrictive exceptionで、追加可能なGlobal diffを当該issuer row一件、Readiness diffを同Incidentのquarantined Head一件に限定する。completed Fail-StopをCAS journalへ束縛してGlobal＋Readiness二pointer CASし、post-CASはsource→candidate transactionのhistorical authenticityとして検証する。current Root usabilityはfalseで、通常Authorization／Approval／Rotationへ流用しない。
3. **unaffected Recovery custodian:** 1、2が不成立で、issuer除外後もBootstrap Verifier／Genesis共通EvidenceとRecovery Policy custodian quorumがvalidなら、そのquorumがReadiness Incident／Invalidation／quarantined Headだけを署名しReadiness一pointer CASする。global pairはnullのまま、続くrecovery-custodian-confirmed `threshold_loss` Root Security Incidentとtotal Root Recoveryの六pointer transactionでnew Profileのfresh independent Evidenceを検証し、assurance issuer rowをcandidate Global Ledgerへappendする。
4. **full trust reset:** 上記Recovery quorumまたは共通Verifier／Genesis Evidenceも成立しなければ既存domainのpointerを進めずfull trust resetだけを許す。vendor responseにより通常entrypointは即時fail closedするため、quarantine pointerを書けないことを旧authority継続と解釈しない。

Production Profile発行Gateは、使用中の各issuerを一件ずつ除外して上記全consumer closureを再評価し、少なくともnormal Rootまたはindependent Recovery branchの一方が必ず成立し、RootとRecoveryを同時喪失しないissuer／custodian分離を全issuerで証明するTarget Receiptを必須にする。Fail-Stopは可用性Gateの成功扱いにせず防御的停止だけとする。`OfflineGovernanceAssuranceIssuerFailStopV1`のslot、issued-at=`payload.recorded_at`、source Root generation revocation context、purposeを両Schema Catalog／Authority Catalogとset equalityにする。issuer pair片側null、wrong issuer／effective time／reason、assurance issuer rowのないnon-null global pair、例外でRoot Key失効を無視、追加Global row／valid Readiness、partial Rootのglobal row不足、Readiness先行、totalでold Root global署名を要求することを拒否する。他のreadiness reason、issuer pair両nullのcustody failure、`trust_authority`はglobal pairを両nullとしてReadiness一pointer CASだけを行う。

`OfflineGovernanceRecoveryReadinessRemediationAuthorizationV1`は、Trust Registry自体はcompromisedでないがcurrent Readinessがquarantinedのため通常`TrustRegistryChangeApprovalV1`を発行できない場合だけ使う論理IDなしcontent objectである。source Root／local／global、quarantined Readiness Head、Incident／Invalidation、source Trust Head／15-member closure、Authority Catalog Head／Catalogを発行時currentからbyte-exact read-backし、Incident→Invalidation→Headとreason／effective timeを再検証する。source closureが持つpre-containment pairとの差は通常reasonではReadiness pairだけ、issuer Nでは同じemergency transactionがappendした当該assurance-issuer row一件のGlobal pair＋Readiness pairだけを許し、他のsource pair driftを拒否する。payloadは`issued_at < valid_until <= source Root Profile.valid_until`、wrapperはpost-overlayでvalidなsource Root Profileの専用purpose `offline_governance_recovery_readiness_remediation_authorization` quorumだけが署名する。source Root quorumが成立しなければこのAuthorizationを使わずRoot Recoveryまたはfull trust resetへ進む。

`remediation_mode=same_generation`はIncident reasonが`drill_evidence_invalid | recovered_state_mismatch`の一件だけ、Root Rotation pairを両null、destination Root／local／global pairをsourceとbyte equalityにする。原因修正Evidenceとfresh隔離drillを持つReceiptは`remediates_invalidation`をsource Invalidationへexact設定し、destination Readiness Headはprevious=quarantined Head、sequence exact `N+1`、status=validとする。destination closure diffはReadiness memberとその参照整合に必要な最小closure memberだけ、六Trust Registry bytesとAuthority Catalog bytesはsourceからbyte-exact保持する。Authorization→destination Trust／Catalog Headの順で完成し、両Headはprevious=current、sequence exact `N+1`、`authorization_kind=recovery_readiness_remediation`、`signer_handoff=destination_only`、source signature／source revocation snapshot=null、destination signature／snapshot non-nullとする。Root／local／globalをexpected-current read setに固定したままReadiness／Trust／Catalog三current pointerを一つのexpected-parent CASでpublishする。

`remediation_mode=assurance_rotation`はIncident reasonが`secrets_exposure_discovered | custody_assurance_invalid`で、post-overlay normal Root branchが成立する場合だけ許す。Root Rotation pairは完成Rotationへnon-null解決し、destination Root／generationはexact `N+1`、destination local revocationはnew-generation genesis、destination globalはsource currentをbyte-exact継承し、destination Readinessはfresh Evidenceを持つnew-generation sequence 1／status=valid／`remediates_invalidation` exactとする。issuer失効ではN selectorとterminal Observation／Detection／Incident／Global issuer rowを再計算し、F／R／Xをこのmodeへ流用しない。source／destinationのRotation Policy、Recovery Policy、Assurance Policy、Control Plane Governance Recovery Policy、Trusted Issuer Manifest、`allowed_purposes[]`／`purpose_quorums[]`、offline verifier core／catalogをbyte-exactにし、RotationのPurpose Set Change Manifest、Static Policy Change Manifest、Verifier Upgrade Authorizationの全pairをnullとする。差分はRoot Key／custodian／fresh custody・reproduction Evidence／validity／generationと、それに必要なclosed Rotation Fieldだけに限定し、既に承認済みのunaffected alternative issuerによるfresh Evidenceだけを許す。destination closure diffはcandidate Root／local／Readinessと参照整合上必要な最小memberだけ、destination global、六Trust Registry、Authority Catalog bytesはsourceから保持する。candidate Root／local／Readiness→destination closure／Catalog→Authorization→destination-only Trust／Catalog Headsの一方向DAGを完成し、Root／local／global／Readiness／Trust／Catalog六current pointerを一つのexpected-parent CASでpublishする。Policy／purpose／Manifest変更が必要ならRecoveryまたはfull trust resetを選び、隔離中に新しいR4 Approvalを発明しない。

両modeともCAS attemptごとにpublication timeと全expected pointerをfresh readし、`issued_at <= publication_time < valid_until`、Head recorded time、Authorization pair、destination Catalog、read-back digestを一致させる。driftは同candidateのblind retryでなくabortしてA2相当から再stagingする。CAS後にdestination current Root／Readiness／Trust／Catalogで全wrapperを再検証するまで通常operationを再開しない。mode／reason差、余分なRegistry／Policy／Catalog変更、source signature、Readiness単独解除、Head↔Authorization cycle、同世代でRoot差、Rotationでglobal rollback、候補先行利用を拒否する。

Readinessの解除branchはreasonごとにclosedとする。`drill_evidence_invalid | recovered_state_mismatch`は上記`same_generation`三pointer route、post-overlay normal Rootが残る`secrets_exposure_discovered | custody_assurance_invalid`は上記`assurance_rotation`六pointer routeを使う。`trust_authority_compromise`は完成Quarantine Recovery Authorization、destination Trust／Catalog、新Receipt／valid HeadによるTrust／Catalog／Readiness三pointer route、`root_authority_compromise`またはnormal Rootが成立しないcustody failureは完成Root Recovery、新generation Root／local／global／Readiness、destination Trust／Catalogの六pointer routeを使う。Recovery custodian Key／material自体のcompromiseはfull trust resetだけを許す。Invalidationの`effective_at <= bootstrap_transaction_time`はhistorical Bootstrapをquarantineし、後なら履歴を保持してcurrent operationだけを停止する。Invalidation削除、Receipt差替えによる隠蔽、reason不一致、remediation pair片側null、通常Change Approvalへの迂回、quarantined Headからのin-place復帰を拒否する。

### 6.1 Control Plane bootstrap approval

5新Ownerを作成しただけではPhase 0を開始しない。pre-baseline Task 0～10、Production trust-root provisioning、各文書の独立Review、baseline core生成が完了した後にだけ、AI／実装workerとは別のR4承認主体が次のRecordを発行する。

```text
ControlPlaneOwnerDocumentBindingV1
  document_id
  document_sha256
  lifecycle_approval_ref

ControlPlaneConstructionDecisionPayloadV1
  decision_id
  subject_git_tree_id
  construction_authorization_ref, construction_authorization_sha256
  construction_task_receipt_chain_head_ref, construction_task_receipt_chain_head_sha256
  active_product_definition_sha256
  future_portfolio_definition_ref, future_portfolio_definition_sha256
  future_portfolio_approval_ref, future_portfolio_approval_sha256
  toolchain_lock_sha256
  owner_document_hashes[5]: ControlPlaneOwnerDocumentBindingV1
  disposition = approved | rejected
  approver_subject_ref, approval_authority_ref
  issued_at
  valid_until
  revocation_snapshot_ref

ControlPlaneConstructionDecisionV1
  payload: ControlPlaneConstructionDecisionPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=control_plane_construction_decision)

ControlPlaneBootstrapApprovalV1
  payload: ControlPlaneBootstrapApprovalPayloadV1
    approval_id
    approver_subject_ref
    approval_authority_ref
    subject_git_tree_id
    owner_document_hashes[5]: ControlPlaneOwnerDocumentBindingV1
    active_product_definition_sha256
    toolchain_lock_sha256
    decision_ref
    decision_sha256
    offline_governance_root_profile_ref
    offline_governance_root_profile_sha256
    construction_authorization_ref
    construction_authorization_sha256
    trust_provisioning_receipt_ref
    trust_provisioning_receipt_sha256
    construction_task_receipt_chain_head_ref
    construction_task_receipt_chain_head_sha256
    baseline_core_ref
    baseline_core_sha256
    issued_at
    revocation_snapshot_ref
    revoked_at
  signed_record: MirakanSignedRecordV1
```

`ControlPlaneBootstrapApprovalPayloadV1.approval_id`は同Fieldを除くpayload JCS SHA-256から`urn:mirakan:control-plane-bootstrap-approval:sha256:<lowercase-hex>`として導出する。`ControlPlaneBootstrapApprovalPayloadV1`と外側の`ControlPlaneBootstrapApprovalV1`はともにclosed schemaで、全Fieldをrequired、unknown Fieldを禁止する。`signed_record`は[AI Verification／Provenance §7](../architecture/01-governance/ai-verification-provenance.md#7-evidence-envelope)が所有する`urn:mirakan:schema:governance:mirakan-signed-record:v1`へのexact `$ref`であり、Architecture schemaへ署名Fieldを複写しない。`signed_record.subject_sha256`はpayloadだけのRFC 8785 JCS bytesのSHA-256、`signed_record.purpose`はexact `control_plane_bootstrap_approval`とする。`signed_record.signer_subject_ref`は`payload.approver_subject_ref`、`signed_record.signer_role_ref`は`payload.approval_authority_ref`、`signed_record.issued_at`と`signed_record.revocation_snapshot_ref`はpayloadの同名Fieldとそれぞれexact byte equalityでなければならない。

`payload.approval_authority_ref`は自由なAuthority名やPolicy参照ではなく、発行時Trust closureのexact Role IDである。Role entryはR4 Architecture approval、AI／実装workerと5 Ownerの作者／対象Reviewerからのindependence、purpose `control_plane_bootstrap_approval`を許可し、`payload.approver_subject_ref`にそのRoleのactive assignmentが存在しなければならない。KeyはSigner所有かつRoleと同じexact ID、singleton `allowed_signed_record_purposes=[control_plane_bootstrap_approval]`を持つ。unknown Role、assignment missing／発行時expired／発行時revoked、independence不成立、payload／envelope Role mismatchは発行時とTask 10B commit時にfail closedにする。commit後の単なるassignment／Key／Authorization／Decision期限経過は履歴を`authentic_but_expired`として保持し、current baselineを遡及失効させない。

`ControlPlaneOwnerDocumentBindingV1`はclosed、`lifecycle_approval_ref`を完成approved lifecycle wrapperへ解決して同document ID／hashを再計算する。`owner_document_hashes[]`のdocument ID集合は§6の5件とset equalityで、配列はdocument IDのunsigned UTF-8 byte順とし、Construction DecisionとBootstrap Approvalの全rowをbyte-exact一致させる。Rebaseline CoreはA2時点の同じ5 IDについてdestination document hash／current completed approvalを再構成し、missing／extra／duplicate／stale Approvalを拒否する。`active_product_definition_sha256`は[Product Plan §11](../architecture/00-product/product-plan.md#11-product-execution-registries)の`ActiveProductDefinitionBundleV1` closureと一致し、Future portfolio、Activation、Decision／Risk evaluation、lifecycle headを含めない。offline root／construction／provisioningの各ref／hashはpublic trust profileとsigned public Receiptだけを対象とし、private key／secret／recovery secretを含めない。Root fingerprint、assurance profile、purpose quorum、Authorization scope／validity、Task 0で生成したRole／assignment／purpose-key public entry、Root／Trust revocation genesis／current head、rotation／Recovery Readiness policyをread-backする。`revoked_at`は必須nullable Fieldで、有効な新規発行payloadでは`null`だけを許可する。後日の失効は署名済みpayloadを書き換えず、typed revocation chainへ追記する。

`decision_ref`は完成`ControlPlaneConstructionDecisionV1` wrapperのcontent ref、`decision_sha256`は同じbytesのhashである。DecisionはAを確定しTask 9 pass Receiptをsidecar発行し、Fをcurrent化した後、Aの外部sidecar storeへ一度だけ発行する。`decision_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:control-plane-construction-decision:sha256:<lowercase-hex>`として導出し、`role.control-plane-construction-decision.r4`への独立human assignment、singleton purpose key、A tree、Task 9 chain head、Owner／Active Definition／current Future Approval・closure／Toolchainを署名する。Task 10以降は完成署名、current Fとのbyte-exact ref／hash、`disposition=approved`を全て満たすDecisionだけを受理する。`rejected`は当該Aをterminalにし、B／Bootstrap Approvalを禁止する。再開には原因を修正した新A／Task 9 Receipt／新Decisionが必要で、Future closureが同じならvalid current Fをreuse、変わるならFのN+1規則を適用する。同じAへのapproved Decision差替えを拒否する。AへDecisionまたはpointerを含めず、Task 10とBootstrap Approvalがsidecar ref／hashをread-backする。これによりDecision↔Aのhash cycleを作らない。hashだけ、別Decision、自由なIssue URLを受理しない。

自己参照を避ける唯一のstaged ceremonyは次である。

1. **A subject tree:** 5 Ownerのapproved bytes、Active／Future Product Definition、Toolchain lock、全Task 0～9 artifactを確定する。Construction Receiptはtree外sidecar storeに置き、Task 9までのchain headを確定する。baseline core、Decision、Future Approval、Bootstrap Approval、baseline envelope、operational genesis、Receipt bytesをAへ含めない。
2. **F Future inception:** 独立human R4 Future approverがA内のexact `FuturePortfolioDefinitionClosureV1` hashを初回はprevious=null／sequence 1の完成`FuturePortfolioApprovalV1`としてtree外content-addressed sidecarへ署名し、expected-empty `future-portfolio-current.ref`をそのwrapperへCASする。Approval payload hashとA内closure ref／hashを再計算し、Active Definition／Product operational state／A treeを変更しない。currentが既に存在する再試行は下記reuse／renewal branchだけを許す。
3. **Construction Decision:** F current化後、A、Task 9 Receipt head、current F Approval／Future closureへ`ControlPlaneConstructionDecisionV1`をsidecar発行する。DecisionのFuture ref／hashをFとbyte-exact一致させる。
4. **B core tree:** Task 10がcurrent Future pointer→完成Approval→A内Future closureと`disposition=approved`の完成Decisionをread-backし、両Future ref／hashをbyte-exact一致させ、`ControlPlaneBaselineCoreV1.subject_git_tree_id=A`、Future closure ref／hash、Approval ref／hashとしてcoreを生成しBへ格納する。coreはBのtree ID、自身のhash、Bootstrap Approval／envelopeを含めない。AがBのancestorであることをread-backする。
5. **C approval tree:** R4主体がAのexact tree、B内core ref／hash、Task 10までのsidecar Receipt chain head、Active Definition、current Future Approval／closure、Toolchain、Decisionを上記payloadで署名しCへ格納する。AとBがCのancestorで、C\Bが完成Bootstrap Approval artifactだけであることを検査する。
6. **D envelope tree:** Task 10Aがcore ref／hashとApproval ref／hashだけを持つ`ControlPlaneBaselineEnvelopeV1`をDへ格納し、全signature、Role／assignment／purpose key／revocation、ancestor、current Future pointer、current bytesを再検証する。
7. **E operational genesis tree:** Task 10Bがfinal Approvalに閉じたProduct operational genesisを発行し、`policy.product.wp.bootstrap-control-plane.v1`で`wp.architecture.control-plane`の`declared->complete` Recordをatomic appendしたnext snapshotをEへ格納する。Future pointerはFのexact currentをexpected valueとして保持し、別CASや暗黙renewalを行わない。この時点より前に通常Critical Path Taskをmaterializeしない。

FのCAS成功後にB／C／D／Eが失敗してもFuture pointerを巻き戻さない。同じA内closureでFがvalidかつnon-revokedなら完成F wrapperをbyte-exact reuseしてcurrent equalityを確認し、expected-empty CASや再署名を繰り返さない。同じclosureでFがexpired／revokedならProduct Planのsequence exact `N+1` renewalを独立R4が発行・current CASし、Construction Decision以降を新F ref／hashで再開する。Future closure bytesを変える場合はBundle revision、Approval sequenceをそれぞれexact `N+1`とし、新A／Task 9 Receipt／F／Decisionから再開する。F署名済みだがCAS前ならcurrent authorityではなく、CASの成功read-back後だけBへ進む。delete、sequence reset、別branch、old FをCoreへ固定することを拒否する。

Recordは文書ごとのlifecycle Approvalを代替せず、AI／workerの自己承認、placeholder Approval、承認対象tree外のDecision、hash計算前の将来値を拒否する。Task 10B atomic transactionのProduct snapshot `created_at`を唯一の`bootstrap_transaction_time`とし、同transaction内のControl Plane lifecycle `recorded_at`もbyte equalityにする。commit前検証はA→B→C→D→E Git ancestor chainとA→F current CAS→Construction Decision→Bのsidecar dependency、全payload／署名／purpose／subject hash、Signer Role／assignment／R4／independence／Key purpose、5 document hash、Active Definition、current F Approval／Future closure／current pointer、Toolchain、Decision、baseline core、Root assurance／purpose quorum、Construction Authorization scope、Task Receipt chain、Provisioning Receipt、generation-local Revocation Head／Snapshot、global Super Head／Ledger、status=validかつnon-expiredでeffective InvalidationのないRecovery Readiness Head／Receiptを現在bytesから再照合する。全Task Receiptは各`completed_at`がAuthorization interval内、F ApprovalとDecisionは`issued_at <= bootstrap_transaction_time < valid_until`、`valid_until`を持たないBootstrap Approvalは`issued_at <= bootstrap_transaction_time`とし、全てcommit時点以前にeffectiveな失効が0件でなければならない。

commit後は上記をimmutableなhistorical authorizationとして検証する。Authorization、Decision、F Approval、assignment、Key、Readinessの単なる期限経過は`authentic_but_expired`であり、完成baseline、Approval、Receiptを`revoked`へ書き換えない。後から記録された失効でも`effective_at <= bootstrap_transaction_time`ならretroactive compromiseとしてcurrent baselineをquarantineし、`effective_at > bootstrap_transaction_time`なら当該主体による新規署名／Authorizationだけを停止してBootstrap履歴は維持する。hash／scope／ancestor drift、commit時点へ遡る失効、署名不成立だけがhistorical proofをquarantineする。通常運用の継続authorityはcurrent Root Head、current generation-local Revocation Head／Snapshot、current global Super Head／Ledger、status=validかつnon-expiredのRecovery Readiness Head／Receipt、current Trust closure／Catalog Head、current Product baseline bindingを別に評価し、Future planning表示はcurrent Future Approval pointerのvalid／non-revoked状態をさらに別に評価する。初回Authorizationの期限や旧Root generationをcurrent authorityとして再利用しない。Approval Recordの追加や無関係な後続fileだけを理由に失効させず、quarantine時は新規WP／Activation／Releaseを停止し§6.1.1 recovery rebaselineへ進む。共通署名schema、payload schema、発行／失効Receipt、read-back testが揃う前にmetadata migrationを開始しない。

### 6.1.1 稼働後のControl Plane／Active Definition rebaseline

初回Bootstrap用のConstruction Authorization、Provisioning Receipt、`ControlPlaneBootstrapApprovalV1`は稼働後のDefinition変更へ再利用しない。Active Definition revision、Future→Active promotion、Owner／Toolchain／Control Plane contract更新には、current Product stateとcurrent offline rootを始点に次の別DAGを使用する。

```text
ControlPlaneRebaselineCoreV1
  rebaseline_core_id
  publication_kind = definition_migration | same_definition_rebinding
  subject_git_tree_id
  source_control_plane_baseline_binding: CurrentControlPlaneBaselineBindingV1
  source_state_snapshot_ref, source_state_snapshot_sha256
  source_active_product_definition_ref, source_active_product_definition_sha256
  destination_active_product_definition_ref, destination_active_product_definition_sha256
  source_binding_condition:
    {status: valid,
     incident_ref: null, incident_sha256: null,
     recovery_authorization_ref: null, recovery_authorization_sha256: null}
    | {status: quarantined,
       incident_ref, incident_sha256,
       recovery_authorization_ref, recovery_authorization_sha256}
  owner_document_hashes[5]: ControlPlaneOwnerDocumentBindingV1
  architecture_index_sha256
  document_relation_registry_sha256
  identity_migration_registry_sha256
  architecture_explain_schema_sha256
  toolchain_lock_sha256
  architecture_lint_artifact_sha256
  local_schema_catalog_ref, local_schema_catalog_sha256
  current_root_head_ref, current_root_head_sha256, current_root_generation
  current_root_revocation_head_ref, current_root_revocation_head_sha256
  current_root_revocation_snapshot_ref, current_root_revocation_snapshot_sha256
  current_global_root_revocation_super_head_ref, current_global_root_revocation_super_head_sha256
  current_global_root_revocation_ledger_ref, current_global_root_revocation_ledger_sha256
  current_recovery_readiness_head_ref, current_recovery_readiness_head_sha256
  current_trust_registry_closure_ref, current_trust_registry_closure_sha256
  current_authority_binding_source_catalog_head_ref
  current_authority_binding_source_catalog_head_sha256
  current_authority_binding_source_catalog_ref
  current_authority_binding_source_catalog_sha256
  current_revocation_snapshot_ref, current_revocation_snapshot_sha256
  lint_version

ControlPlaneGovernanceIncidentPayloadV1
  incident_kind = baseline_approval_signer_key_compromise |
    baseline_approval_record_invalidation |
    baseline_transaction_signer_key_compromise |
    baseline_transaction_record_invalidation |
    operational_state_publisher_key_compromise |
    operational_state_snapshot_record_invalidation
  status = confirmed
  governance_recovery_policy_ref, governance_recovery_policy_sha256
  affected_control_plane_baseline_binding: CurrentControlPlaneBaselineBindingV1
  affected_operational_state_snapshot_ref
  affected_operational_state_snapshot_sha256
  affected_root_head_ref, affected_root_head_sha256
  affected_trust_registry_closure_ref, affected_trust_registry_closure_sha256
  affected_trust_status = healthy
  affected_subjects[]:
    {subject_kind = key | signed_record,
     subject_ref, subject_sha256: lowercase hex 64}
  evidence[]: ControlPlaneGovernanceEvidenceBindingV1
  detected_at
  confirmed_at
  compromise_effective_at
  incident_authority_subject_ref, incident_authority_role_ref
  revocation_snapshot_ref

ControlPlaneGovernanceIncidentV1
  payload: ControlPlaneGovernanceIncidentPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=control_plane_governance_incident)

ControlPlaneRecoveryRebaselineAuthorizationPayloadV1
  authorization_id
  recovery_route = healthy_trust_rebaseline |
    root_backed_trust_then_rebaseline |
    offline_root_then_trust_then_rebaseline
  source_state_snapshot_ref, source_state_snapshot_sha256
  source_control_plane_baseline_binding: CurrentControlPlaneBaselineBindingV1
  incident_ref, incident_sha256
  completed_root_recovery_ref: null | content-addressed ref
  completed_root_recovery_sha256: null | lowercase hex 64
  completed_trust_recovery_ref: null | content-addressed ref
  completed_trust_recovery_sha256: null | lowercase hex 64
  current_root_head_ref, current_root_head_sha256, current_root_generation
  current_root_revocation_head_ref, current_root_revocation_head_sha256
  current_root_revocation_snapshot_ref, current_root_revocation_snapshot_sha256
  current_global_root_revocation_super_head_ref, current_global_root_revocation_super_head_sha256
  current_global_root_revocation_ledger_ref, current_global_root_revocation_ledger_sha256
  current_recovery_readiness_head_ref, current_recovery_readiness_head_sha256
  current_trust_registry_closure_ref, current_trust_registry_closure_sha256
  current_trust_revocation_registry_ref, current_trust_revocation_registry_sha256
  current_authority_binding_source_catalog_head_ref
  current_authority_binding_source_catalog_head_sha256
  approver_subject_ref, approval_authority_ref
  issued_at
  revocation_snapshot_ref

ControlPlaneRecoveryRebaselineAuthorizationV1
  payload: ControlPlaneRecoveryRebaselineAuthorizationPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=control_plane_recovery_rebaseline_authorization)

ControlPlaneRebaselineApprovalPayloadV1
  approval_id
  publication_kind = definition_migration | same_definition_rebinding
  source_state_snapshot_ref, source_state_snapshot_sha256
  source_active_product_definition_ref, source_active_product_definition_sha256
  destination_active_product_definition_ref, destination_active_product_definition_sha256
  source_binding_condition: exact Core value
  rebaseline_core_ref, rebaseline_core_sha256
  row_migration_manifest_ref: null | content-addressed ref
  row_migration_manifest_sha256: null | lowercase hex 64
  approver_subject_ref
  approval_authority_ref
  current_root_head_ref, current_root_head_sha256, current_root_generation
  issued_at
  revocation_snapshot_ref

ControlPlaneRebaselineApprovalV1
  payload: ControlPlaneRebaselineApprovalPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=control_plane_rebaseline_approval)

ControlPlaneRebaselineEnvelopeV1
  rebaseline_core_ref, rebaseline_core_sha256
  rebaseline_approval_ref, rebaseline_approval_sha256

CurrentControlPlaneBaselineBindingV1
  binding_id
  active_product_definition_sha256
  binding:
    {kind: bootstrap,
     envelope_ref, envelope_sha256,
     approval_ref, approval_sha256}
    | {kind: rebaseline,
       envelope_ref, envelope_sha256,
       approval_ref, approval_sha256,
       transaction_ref, transaction_sha256}

ControlPlaneArtifactQualificationManifestV1
  manifest_id
  publication_kind = definition_migration | same_definition_rebinding
  destination_active_product_definition_ref, destination_active_product_definition_sha256
  candidate_ref, candidate_sha256
  toolchain_lock_sha256
  target_profile_refs[]
  artifacts[]:
    {artifact_id, artifact_ref, artifact_sha256,
     qualification_receipt_ref, qualification_receipt_sha256}
  generated_at
  revocation_snapshot_ref

ControlPlaneRebaselineTransactionPayloadV1
  transaction_id
  state = publication_ready
  publication_kind = definition_migration | same_definition_rebinding
  source_state_snapshot_ref, source_state_snapshot_sha256
  source_active_product_definition_ref, source_active_product_definition_sha256
  destination_active_product_definition_ref, destination_active_product_definition_sha256
  source_binding_condition: exact Core／Approval value
  row_migration_manifest_ref: null | content-addressed ref
  row_migration_manifest_sha256: null | lowercase hex 64
  rebaseline_core_ref, rebaseline_core_sha256
  rebaseline_approval_ref, rebaseline_approval_sha256
  rebaseline_envelope_ref, rebaseline_envelope_sha256
  control_plane_artifact_qualification_manifest_ref
  control_plane_artifact_qualification_manifest_sha256
  requested_by_subject_ref
  recorded_at
  revocation_snapshot_ref

ControlPlaneRebaselineTransactionV1
  payload: ControlPlaneRebaselineTransactionPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=control_plane_rebaseline_transaction)
```

全objectはclosed、ref／隣接hashは同一完成bytes、配列はunsigned UTF-8 byte順・重複なしとする。ID projectionを型別に固定する。flat `ControlPlaneRebaselineCoreV1.rebaseline_core_id`とflat `CurrentControlPlaneBaselineBindingV1.binding_id`は同ID Fieldを除く完成object JCS SHA-256からそれぞれ`urn:mirakan:control-plane-rebaseline-core:sha256:`、`urn:mirakan:control-plane-baseline-binding:sha256:`として導出する。`ControlPlaneRebaselineApprovalPayloadV1.approval_id`と`ControlPlaneRebaselineTransactionPayloadV1.transaction_id`は同ID Fieldを除くpayload JCS SHA-256からそれぞれ`urn:mirakan:control-plane-rebaseline-approval:sha256:`、`urn:mirakan:control-plane-rebaseline-transaction:sha256:`として導出する。`ControlPlaneRecoveryRebaselineAuthorizationPayloadV1.authorization_id`も同Fieldを除くpayload JCS SHA-256から`urn:mirakan:control-plane-recovery-rebaseline-authorization:sha256:<lowercase-hex>`として導出する。Approvalはcurrent `role.control-plane-rebaseline-approver.r4`への独立human assignmentとsingleton-purpose key、Transactionは`service.control-plane-rebaseline-publisher`／`role.control-plane-rebaseline-publisher`のsingleton-purpose keyで署名する。subject hash、Signer／Role、型別時刻、current assignment／Key／Role／revocationを共通署名規則で再検証する。`CurrentControlPlaneBaselineBindingV1`はProduct operational snapshot内のinline immutable valueであり、genesisはbootstrap branch、両publication kindはrebaseline branchを生成し、通常state／lifecycle transitionはbyte-exactに保持する。branchのApproval／Envelope型、active definition hash、Transaction `state=publication_ready`を再検証し、2回目以降も直前current bindingをsource coreへそのまま束縛する。

`publication_kind`、source／destination Definition ref／hashはCore／Approval／Transactionでbyte equality、destination Definition ref／hashはArtifact Qualification Manifestともbyte equalityとし、各refをcontent-addressed closureへ解決して隣接hashを再計算する。`definition_migration`はsourceとdestination Definition ref／hashが異なり、row migration manifest ref／hashを両non-nullにして全Definition row diffへ解決する。`same_definition_rebinding`は両Definition ref／hashを同一current値、manifest ref／hashを両nullにし、Definition全row／closure bytesをbyte-exactに保持する。このbranchで許すsemantic business-state差は`wp.architecture.control-plane` lifecycle Headのexact `N+1` `complete->complete`と`control_plane_baseline_binding`の二つだけであり、全Activation、Decision／Risk evaluation、全non-Control-Plane WP lifecycle Head、その他全business-state rowをsourceとbyte-exactにする。snapshot chain metadataはProduct Planのpublication規則どおりsequence exact `N+1`、`previous_state_snapshot_ref`=source完成wrapper content ref、`applied_change.kind=control_plane_rebaseline`かつrebinding wrapper ref／hash=完成Baseline Rebinding、`created_at`=Rebinding `recorded_at`=atomic publication time、`revocation_snapshot_ref`=Rebindingの同Fieldへだけ更新し、これをbusiness-state変更として数えない。いずれもcurrent local／global Root Revocation、status=validのRecovery Readiness、Trust closure、Authority Catalog Head／CatalogをCoreへ束縛し、staged publication中のどれか一Headの変更でA2から再開する。片側null、same-definitionでmanifest付与、definition migrationで同一ref／hash、publication kind差、許可外operational state差を拒否する。

`ControlPlaneRebaselineApprovalV1`は時刻だけによる定期full-resetを避けるため`valid_until`を持たず、current revocation、source／destination／core drift、Qualification Receipt freshnessのいずれかでfail closedにする。通常rebaselineは`source_binding_condition={status=valid, incident/ref/hash=null, recovery_authorization/ref/hash=null}`だけを許す。quarantined sourceからは次のclosed recovery routeを一つだけ完了した`ControlPlaneRecoveryRebaselineAuthorizationV1`をread-backし、`status=quarantined` branchの全non-null ref／hashをcurrent healthy／recovered Trust closureの独立R4が承認した場合だけrecovery rebaselineを開始できる。Core／Approval／Transactionのconditionはbyte-exactに一致させる。この例外はsource chainを有効と再主張せず、rebaseline／migration以外のWP、Activation、Releaseを許可しない。

`ControlPlaneGovernanceRecoveryPolicyV1`はcurrent Root Profileのref／hashでpre-commitし、healthy TrustのGovernance Policy Registry rowは同じbytesをmirrorするだけとする。route binding集合は、Control Plane Incident schemaの上記六closed kind→`healthy_trust_rebaseline`、Trust Quarantine Incident schemaの三closed kind→`root_backed_trust_then_rebaseline`、Root Security Incident schemaの**affected signer Root Profile**がpre-commitしたOffline Recovery Policy `triggering_incident_kinds[]`→`offline_root_then_trust_then_rebaseline`とのset equalityにする。Root Recoveryで生成したnew ProfileのPolicy、quarantined Trust内Policy row、呼出元route、自由kindでaffected incidentを再解釈しない。Root ProfileとGovernance Recovery Policy ref／hashが変わった場合はRoot Rotation／Recoveryとdestination Trust closure更新を先に完了する。

各routeのincident型とnullabilityは次だけを許す。

| recovery route | exact incident／authority | completed Root Recovery | completed Trust Recovery | Authorization signer context |
|---|---|---|---|---|
| `healthy_trust_rebaseline` | `ControlPlaneGovernanceIncidentV1`。affected Trustはhealthyで、baseline Approval／Transaction／current Product snapshotまたはその実signerだけがcompromised。current Trustの独立R4が署名 | null | null | Incidentを束縛するrevocation change後のhealthy current Trust |
| `root_backed_trust_then_rebaseline` | Root quorumの`TrustRegistryQuarantineIncidentV1`。source Trust／publisher／Catalogがquarantined | null | 完成`TrustRegistryQuarantineRecoveryAuthorizationV1`とTrust／Catalog／Readiness三pointer CAS | recovered destination current Trust |
| `offline_root_then_trust_then_rebaseline` | Root／Recovery-custodian quorumの`OfflineGovernanceRootSecurityIncidentV1` | 完成`OfflineGovernanceRootRecoveryV1` | 完成`TrustRegistryRecoveryAuthorizationV1`と六pointer CAS | recovered Rootが束縛するdestination current Trust |

`ControlPlaneGovernanceIncidentV1`は論理IDを持たず完成signed wrapper content ref／hashだけで同定する。status confirmed、affected current operational snapshot／inline binding／Root／healthy Trust closureをsourceと一致させ、`role.control-plane-governance-incident.r4`への独立human assignmentとsingleton purpose Keyを必須にする。`affected_subjects[]`は次表のexact singleton、`subject_sha256`はKeyならsource Public Key Registry closed row、signed recordなら完成wrapper bytesのSHA-256とする。key compromise branchは`compromise_effective_at <= target record issued_at／recorded_at`を必須にし、単なる将来利用停止をbaseline recoveryへ誤分類しない。

| `incident_kind` | exact target | destination revocation |
|---|---|---|
| `baseline_approval_signer_key_compromise` | bindingの完成Bootstrap／Rebaseline Approval実signature slotが解決したPublic Key row | `subject_kind=key`、`reason_code=credential_compromise` |
| `baseline_approval_record_invalidation` | bindingの完成Approval wrapper | `subject_kind=signed_record`、`reason_code=signed_record_invalidation` |
| `baseline_transaction_signer_key_compromise` | `binding.kind=rebaseline`の完成Transaction実signature slotが解決したPublic Key row | `subject_kind=key`、`reason_code=credential_compromise` |
| `baseline_transaction_record_invalidation` | `binding.kind=rebaseline`の完成Transaction wrapper | `subject_kind=signed_record`、`reason_code=signed_record_invalidation` |
| `operational_state_publisher_key_compromise` | current affected Product operational snapshot実signature slotが解決したPublic Key row | `subject_kind=key`、`reason_code=credential_compromise` |
| `operational_state_snapshot_record_invalidation` | current affected Product operational snapshot wrapper | `subject_kind=signed_record`、`reason_code=signed_record_invalidation` |

healthy routeはIncidentをsource Trustで検証後、destination Authority Revocation Registryへexactly one rowを追加する。rowの`effective_at=Incident.compromise_effective_at`、`governance_incident_ref/hash`=完成Incident、`recovery_record` pair=nullとし、他の全Trust registry／Catalog rowはbyte-exactに保持する。完成Trust Change Manifest／Evidence／Approval、destination closure／Trust Head／Catalog Headを生成し、Trust／Catalog current pointerを同一expected-parent二pointer CASしてread-backするまでIncidentをcurrentにしない。Incident signer／Trust Change approver／head publisherはtargetから独立させる。CAS後はcurrent revocationからaffected bindingを一意にquarantineでき、Trust closure自体はhealthyのまま、同じdestination revocation pairを持つ`ControlPlaneRecoveryRebaselineAuthorizationV1`だけを発行する。P2 destination snapshotと以後のTrust revisionは当該rowを保持し、削除／effective time後退／Incident差替えを拒否する。

Trust Quarantine Incident／Authorizationはquarantined Governance RegistryのRole／Policyを使わず、Root Profileが固定したpurpose quorum、recovery Policy、typed Evidenceだけを使う。各incident `evidence[]` kind集合はRoot Profile固定Governance Recovery Policyの該当route rowとset equalityにし、schema／subject projection／issuer／signature／current revocation、`completed_at <= confirmed_at`、freshnessを検証する。subject projectionはincident schema／kind、affected bindingまたはTrust Head／closure／Catalog、Root Head、typed affected target集合、detected／confirmed timeをexactに含める。

`ControlPlaneRecoveryRebaselineAuthorizationV1`はrouteに応じたincident wrapper、completed Root／Trust Recovery ref／hash nullability、current Root／local／global／Readiness／Trust／Catalogを上表どおりexact一致させる。healthy routeは`current_trust_revocation_registry_ref/hash`がcurrent closure内Registry bytesで、上表のexact target／reason／effective time／Incident pairを持つことを再検証する。root-backed／offline routeはrecovery CAS read-back後のdestination Trustでのみ発行し、source quarantined TrustのR4署名を受理しない。Recovery Authorization／Core／Approval／Transactionは同じ完成incident ref／hashとrouteを継承し、Issue URL、Markdown、自由文字列、別binding incident、wrong wrapper型、recovery ref片側null、route shortcutを拒否する。

`ControlPlaneArtifactQualificationManifestV1.artifacts[]`のrequired ID集合は`artifact.control-plane.schema-bundle | artifact.control-plane.validator | artifact.control-plane.generators | artifact.control-plane.policy-service | artifact.control-plane.product-state-services | artifact.control-plane.architecture-lint`のclosed setとする。各Receiptは完成`TechnicalQualificationReceiptV1`、`freshness_policy_ref=policy.evidence.contract-ci.v1`でなければならない。用途は存在しない自由なEvidence class Fieldでなく、`subject_hash`がcanonical closed closure `{usage=control_plane_rebaseline, publication_kind, candidate_ref/hash, destination_active_product_definition_ref/hash, toolchain_lock_sha256, target_profile_refs, artifact_id/ref/hash, fixture_refs}`のJCS SHA-256であることにより一意にする。Target setは`target.headless.host | target.windows.editor`、fixture setは登録済みControl Plane contract／negative／recovery fixtureのexact setとし、Receiptはfreshかつnon-revokedでなければならない。manifest IDは同Fieldを除くobject JCS hashから`urn:mirakan:control-plane-artifact-qualification:sha256:<lowercase-hex>`として導出し、missing／extra／duplicate artifact、unknown usage/class、publication kind差、別Target／Candidate／definition／toolchain、stale Receiptを拒否する。

許可順序は一つだけである。

1. **A2 subject:** current source snapshot／baselineをread-backし、destination Definition closure、kindに応じたrow migration manifest、5 Owner、Toolchain、lint、current Root／Trust／Authority Catalog closureを確定する。
2. **B2 core:** A2のtree IDを持つ`ControlPlaneRebaselineCoreV1`を生成し、source current head、Root／Revocation／Readiness Head、Trust／Catalog Head、全hashを再検証する。
3. **C2 approval:** 独立R4 Architecture主体だけがB2 core、source current、destination closure、kindに応じたrow manifestを`ControlPlaneRebaselineApprovalV1`として承認する。Product publication判断はこの署名へ混ぜない。
4. **D2 envelope:** coreと完成Approvalだけを束縛する`ControlPlaneRebaselineEnvelopeV1`を生成する。
5. **T2 transaction:** destination closure上でControl Plane artifactを再Qualificationし、fresh Receipt exact closureとD2を持つ`publication_ready` Transactionを発行する。
6. **L2 lifecycle:** `definition_migration`はdestination epochのControl Plane sequence 1 `declared->complete`、`same_definition_rebinding`はcurrent epochのexact `N+1` `complete->complete` wrapperをT2／D2／C2へ束縛する。
7. **P2 product publication:** 前者はmigration approval projectionへのR4 Product Decision、完成`ActiveProductDefinitionMigrationV1`、full-reset destination snapshotをsingle CASでpublishする。後者はbaseline-rebinding approval projectionへの別R4 Product Decision、完成`ControlPlaneBaselineRebindingV1`、Control Plane headだけを更新したnext snapshotをsingle CASでpublishする。

A2→B2→C2→D2→T2→L2→P2以外の順、初回Authorization／Bootstrap Approval参照、source Receipt carry、Approval後のcore差替え、destination publish後の後付けApprovalを拒否する。source current headが途中で変わればstaged artifactを失効させA2から再開する。P2後はkindに対応するProduct migration／rebinding wrapperがcommit Receiptを兼ね、別の「成功」pointerを発明しない。Architecture R4 ApprovalとProduct R4 Decisionを一つの署名や兼任推論で代用しない。将来proof carry-forwardを導入する場合も、Product Planのplanning-only Future項目がactiveへ昇格するまでV1のfull resetを変更しない。

#### 6.1.1.1 Baseline-scoped repository change closure

Architecture document／metadata／relation、Local Schema Catalog、Authority Binding Source Catalog、Control Plane code／validator／generator、Toolchain lockなど、current Baseline Coreの直接Fieldまたはhash closureを変える通常Work Packageは、旧bindingのまま`active->complete`にできない。Active Product Definition hashを変えない変更は次の一方向closureを必須とする。

1. 対象WPを旧binding上で`active`に保ち、隔離stagingへSource／artifactを生成してnegative testまで実行する。この段階のQualification／Build Receiptには`receipt_usage=rebaseline_preliminary`とsource baseline binding ref／hashをsubject closureへ含める。
2. 既存Architecture documentの変更は新Core、`ArchitectureDocumentChangeManifestV1`、`content_revision`でcurrent state→`review`、続く`approve`を完了する。新規Architecture documentはprevious Core／Change Manifestを捏造せず、新Coreへの初回`submit_review`（from=null、to=`review`）→`approve` branchを使う。両branchで人間R4 document approvalをread-backし、未変更Ownerのcurrent approvalもA2で再検証する。
3. same DefinitionのA2→B2→C2→D2→T2→L2→P2を実行する。P2直前／直後で対象WPを含む全non-Control-Plane lifecycle head、全Activation、Decision／Risk evaluation、Definition hashをparentからbyte-exactに保持し、`wp.architecture.control-plane` headとbaseline bindingだけを更新する。
4. new current bindingのread-back後、同じSource revision／Candidate／Toolchain／Target closureへ最終Qualification／Buildを再実行する。最終Receiptは`receipt_usage=work_package_final`とdestination baseline binding ref／hashを持ち、fresh Owner acceptanceとともに通常policyで対象WPを`active->complete`へ進める。

preliminary ReceiptはT2のrebaseline根拠にだけ使用でき、WP final completion、Phase／Release Gate、後続WPのprerequisite、Shipping claimへ使用できない。final Receiptをrebaseline前に発行すること、P2で対象WPをcompleteへ同時遷移すること、rebaseline後に旧binding Receiptを再labelすること、対象WP以外のstateを変更することを拒否する。Definition bytesも変える変更はこのbranchを使わず`definition_migration` full resetへ進む。

`subject_git_tree_id`が指すtree内のSource、generated projection、schema、binary descriptorへcurrent baseline binding、Rebaseline Core／Approval／Envelope／Transaction、Product snapshot ref／hashをinlineしてはならない。tree内artifactは自己完結したcontent hashまでを持ち、artifact hashとcurrent binding ref／hashの結合はtree外append-only preliminary／final Receipt wrapperだけが所有する。これによりtree hash→binding→Core subject tree hashの自己循環を禁止する。bindingを埋めたartifact、Receiptのtree内格納、将来binding値の仮記入をvalidatorで拒否する。

### 6.2 Architecture Governance

`01-governance/architecture-governance.md`は次だけを所有する。

- Architecture document metadataとlifecycle。
- `requires`、`integrates_with`、`supersedes`の意味。
- Canonical contract Owner宣言とconsumer参照規則。
- Change impact closureと更新単位。
- Index生成、lint、review、approval、supersede Gate。
- Decisionとactive specの関係。
- exact metadata／registry／Source revisionから生成するbounded architecture-explain projection。

AI Risk、Release approval、Evidence内容は既存Governance文書が引き続き所有する。

### 6.3 Compatibility／Evolution

`02-foundation/compatibility-evolution.md`は次だけを所有する。

- Engine public surfaceの分類。
- Pre-1.0とPost-1.0のversion policy。
- Schema major／minor／patch、Field identity、enum、extension point規則。
- Project、Save、Replay、Pack、Native ABI、Runtime／Application Packageの互換方向。
- Migration graph、deprecation、support window、revocation。
- Target非依存の`GameCompatibilitySubjectV1`。Project revision、Engine baseline、Contract／Content／Save／Pack closureを一つの互換subjectへ固定する。
- `EngineContractVersionV1`、`EngineCompatibilityRangeV1`、`PackCompatibilityRangeV1`、`CompatibilityDecisionV1`、`MigrationCoverageV1`、`SupportPolicyV1`。
- Change classificationと必須verification closure。

個々のDomain migration algorithmはDomain Ownerが所有し、本書は登録、順序、coverage、failure atomicityを所有する。

### 6.4 Persistence／Save

`04-runtime/persistence-save.md`は次だけを所有する。

- 全System／World／ECSのSave projection集約。
- Save slot、generation、checkpoint、root manifest。
- Capture boundary、canonical order、hash、size bound。
- Migration plan、load staging、atomic publication、failure recovery。
- Replay開始checkpointとの共有境界。
- Platform storage adapterへ渡すtransaction contract。
- Runtime Package／`GameCompatibilitySubjectV1`とのcompatibility binding。

各Systemのsaved／derived Field意味は`SaveReplayContractV1`と各Domain Owner、Runtime Entity／Component再構築はECS Owner、file root／atomic replace／journal／cloud transportはPlatform Ownerが所有する。

### 6.5 Runtime Package

`04-runtime/runtime-package.md`は次だけを所有する。

- `RuntimePackageManifestV1`。
- `RuntimePackageArtifactV1`。Manifest hash、payload root hash、integrity／trust profileをManifest外で束ねる。
- Runtime loaderの検証順、exact dependency closure、load／rollback。
- Engine／Game binary、Contract set、System implementation set、World root、Content Package、Shader artifact、Save migration setのbinding。
- Development loose layoutとShipping packageが通る共通conformance。
- Runtime Packageのrebuild／invalidation規則。

Content Package formatはAsset Lifecycle、Platform executable formatは各Platform、Build task orchestrationはBuild Gatewayが所有する。

### 6.6 Application Package／Release

`07-platform/application-package-release.md`は次だけを所有する。

- `ApplicationPackageAssemblyManifestV1`。
- `UnsignedApplicationPackageV1`の共通envelope。
- `PackageValidationReceiptV1`、`TargetPackagePreparationTransactionV1`、`StoreStagingUploadReceiptV1`、`StoreStagingReadBackReceiptV1`、`TargetPackagePreparationRecordV1`、`ReleaseTransactionV1`、`StorePublicationReceiptV1`。
- Target packageのBuild、Validate、Authorize、Sign、staging upload／read-backと、Candidate承認後のStore submit、publish、public read-backを分けた一方向state machine。
- Windows／Android／Apple固有packageへのmapping。
- Signing／Upload serviceのinput最小化とidentity separation。
- Compatibility Ownerの`GameCompatibilitySubjectV1`参照。
- `ReleaseSigningReceiptV1`、Store staging Receipt、`TargetPackagePreparationRecordV1`へのhash binding。

署名authorizationと最終`GameCandidateManifestV1`はAI Security／Approval、Evidence envelopeはAI Verification／Provenance、AAB／Apple archive／Windows layoutのformatは各Platform Ownerが所有する。Application Package／ReleaseはCandidateを再定義せず、Target別の完了Recordを供給する。

## 7. Architecture文書Governance

### 7.1 Header format

全active specはH1直後の最初のblockに、次のexact JSON metadataを持つ。Lifecycle state／Approval refを本文へ埋め込まず、別content-addressed Artifactへ分離する。Delimiter、key、型を固定し、YAML、Markdown list、別名key、comment、trailing comma、duplicate keyを許可しない。Relationはpathではなくstable document IDを格納し、Index generatorがlinkへ解決する。

```text
<!-- architecture-metadata:v2
{
  "schema": "mirakan.architecture-document-metadata/v2",
  "document_id": "mirakan.arch.compatibility-evolution",
  "owner_scope": ["互換性と進化の規則"],
  "non_owner_scope": ["個別Domainのmigration algorithm"],
  "requires": ["mirakan.arch.architecture-governance"],
  "integrates_with": [
    {
      "document_id": "mirakan.arch.executable-contracts",
      "contract_ids": ["contract.foundation.artifact-ref"]
    }
  ],
  "supersedes": [],
  "external_evidence_verified_at": "2026-07-22"
}
-->
```

```text
ArchitectureMetadataV2
  schema = mirakan.architecture-document-metadata/v2
  document_id
  owner_scope[]
  non_owner_scope[]
  requires[]
  integrates_with[]: {document_id, contract_ids[]}
  supersedes[]
  external_evidence_verified_at: null | YYYY-MM-DD

ArchitectureDocumentCoreV1
  document_core_id
  document_id
  document_path
  document_sha256
  git_blob_id
  metadata_projection_sha256

ArchitectureDocumentChangeManifestPayloadV1
  manifest_id
  document_id
  previous_document_core_ref, previous_document_core_sha256
  next_document_core_ref, next_document_core_sha256
  change_class = editorial | normative
  affected_contract_ids[]
  impacted_document_ids[]
  impact_evidence_refs[]
  requested_by_subject_ref
  generated_at
  revocation_snapshot_ref

ArchitectureDocumentChangeManifestV1
  payload: ArchitectureDocumentChangeManifestPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=architecture_document_change_manifest)

DocumentLifecyclePayloadV1
  lifecycle_record_id
  document_id
  document_core_ref, document_core_sha256
  previous_lifecycle_record_ref: null | content-addressed ref
  previous_lifecycle_record_sha256: null | lowercase hex 64
  lifecycle_sequence: positive safe integer
  from_state: null | draft | review | approved | superseded
  to_state: draft | review | approved | superseded
  transition_kind = construction_seed | submit_review | approve | content_revision | supersede
  change_manifest_ref: null | content-addressed ref
  change_manifest_sha256: null | lowercase hex 64
  requested_by_subject_ref
  recorded_at
  revocation_snapshot_ref

DocumentLifecycleRecordV1
  payload: DocumentLifecyclePayloadV1
  signed_record: MirakanSignedRecordV1(purpose=document_lifecycle_transition)
```

制約は次である。

- `document_id`は`^[a-z][a-z0-9]*(?:\.[a-z][a-z0-9_-]*)+$`に一致し、renameまたは移動でも変えない。
- Arrayは重複なし、`requires`／`supersedes`はdocument ID byte順、`integrates_with`は`document_id` byte順、各`contract_ids`もContract ID byte順とする。`owner_scope`と`non_owner_scope`だけは記述順を意味順として維持する。
- `integrates_with` entryは`document_id`と1件以上のcanonical `contract_ids`だけを持つ。Major／revisionはOwnerのShared canonical contracts表から解決する。Peer側に同じdocument IDと同じContract集合がなければ拒否する。
- `external_evidence_verified_at`は外部根拠を使わない文書だけ`null`を許可し、それ以外はUTC基準の`YYYY-MM-DD`とする。

`ArchitectureDocumentCoreV1`はLifecycle sidecarを含まない完成UTF-8 file bytesとGit blobを束縛し、`document_core_id`を同Fieldを除くJCS hashから`urn:mirakan:architecture-document-core:sha256:<lowercase-hex>`として導出する。`DocumentLifecycleRecordV1`はdocument IDごとのappend-only sidecar chainで、最初だけprevious両Field=null／sequence 1／from=null、以後はcurrent completed wrapperとexact `N+1`を持つ。current pointerはper-document expected-head CAS、Record IDは同Fieldを除くpayload JCS hashから`urn:mirakan:document-lifecycle:sha256:<lowercase-hex>`として導出する。

Task 3の5 Ownerだけは初回`construction_seed`でfrom=null／to=`review`を許す。それ以外の新規文書は完成Coreへの初回`submit_review`でfrom=null／to=`review`とし、既存`draft`文書の`submit_review`はfrom=`draft`／to=`review`とする。新規文書に存在しないprevious Core／Change Manifestを作らず、既存approved文書の変更を初回`submit_review`へ偽装しない。

`approve`はfrom=`review`、to=`approved`、同じDocument Coreを束縛し、独立R4 Document approver Role／singleton purpose keyを必須とする。本文を一byte変えた場合は新Coreと`content_revision` Recordを同一transactionで発行し、from=current state、to=`review`、non-null完成`ArchitectureDocumentChangeManifestV1`を必須にする。Manifestはprevious Coreをcurrent lifecycle headが指すCore、next Coreを当該Recordの`document_core_ref/hash`とexact一致させる。`change_class=normative`ではnon-empty affected Contractまたはimpacted documentとEvidenceを必須、`editorial`ではContract意味不変を示すEvidenceを必須とし、空の「変更なし」主張を許可しない。

Manifest `manifest_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:architecture-document-change-manifest:sha256:<lowercase-hex>`として導出する。`role.architecture-document-change-author`へのcurrent assignmentとsingleton purpose `architecture_document_change_manifest` Keyで署名し、subject hash、`signed_record.issued_at=payload.generated_at`、revocation snapshotを一致させる。transition kind別presenceは`content_revision=manifest ref/hash両non-null`、`construction_seed | submit_review | approve | supersede=両null`である。片側null、別document、old／new Core逆転、currentでないold Core、Lifecycle signerによるManifest自己署名推論を拒否する。本文へApproval refを書き戻さないためhash cycleは生じない。`construction_seed`はTask 3の新5 Ownerを`review`へseedするConstruction Authorization限定、通常の`submit_review`、`supersede`はPolicyが指定するRoleを使う。Indexのstate／approval linkはcurrent signed lifecycle headから導出し、file metadataへ保存しない。missing／fork／gap／別Core approval、予約logical ID、unsigned sidecarを拒否する。

従来の`文書ID`、`状態`、`正本範囲`、`非正本範囲`、無型の`依存`は、metadata v2＋Lifecycle sidecarへ一ChangeSetで置換する。移行中に旧metadata state／approval_refとLifecycle headを併存させない。

### 7.2 Relation semantics

| Relation | 意味 | Cycle | 検査 |
|---|---|---|---|
| `requires` | 本文書のnormative意味を確定する前提Owner | 禁止 | 全体DAG、self edge禁止、存在確認 |
| `integrates_with` | 双方がtyped boundaryで連携するpeer | 許可 | reciprocal edge、境界Contract Owner確認 |
| `supersedes` | 旧文書を置換する一方向関係 | 禁止 | replacement存在、旧文書`superseded` |

External evidence、参考実装、比較対象は本文末のReferenceに置き、document dependencyへ混ぜない。Product PhaseやEvidence Gateは各専用Contract refで結び、`requires`へ代用しない。

`requires`はdirect prerequisiteだけを持ち、別pathで到達できるtransitive edgeを重複登録しない。Architecture lintはtransitive reduction後も同じreachabilityになるedgeをredundantとして拒否する。これにより読む順序と変更影響を過大化させない。

### 7.3 Document lifecycle

```text
draft -> review -> approved -> superseded
          ^          |
          +----------+  normative change
```

| State | 意味 | 許可される行為 |
|---|---|---|
| `draft` | Owner、境界、代替案を作成中 | Prototype計画のみ。Public Contract実装、Promotion禁止 |
| `review` | 内容が完全で検証可能、承認待ち | Fixture／spikeは可能。Production activation禁止 |
| `approved` | Approval refが指すDecisionのread-back済みhashと現在bytesが一致 | Engine機能のimplementation planとActivation candidate作成可能 |
| `superseded` | active正本ではない | `docs/architecture/superseded/`へread-only保持しactive Indexから除外。新規参照禁止 |

承認済みdocumentのfile bytesが一byteでも変われば新Coreと`content_revision` Lifecycle Recordで`approved -> review`へ進め、新しいApprovalを要求する。Typo／Link修正を機械的にnormative変更と同一視するわけではないが、exact hashとの矛盾を避けるため承認状態のbypassは設けない。生成Indexだけの変更はactive spec bytesを変えないため各specのApprovalへ影響しない。

### 7.4 Canonical contract ownership

別文書から参照される型はOwner文書の`Shared canonical contracts`表へ一度だけ登録する。

```text
| Contract type | Contract ID | Major | Current revision | Schema hash | Persistence class | Consumers |
```

Consumerは同名型を再定義せず、Owner linkとexact `McdContractRefV1`を使う。MCD実装後はMCD Registryが機械正本となり、Markdown表は生成projectionへ移行する。それまではArchitecture lintが表の一意性とconsumer参照を検査する。

### 7.5 Indexと件数

`active spec`は`docs/architecture/00-product`～`08-packs`にあり、`decisions/`と`superseded/`の配下ではなく、`ArchitectureMetadataV2`がvalidで、current完成`DocumentLifecycleRecordV1` headのstateが`draft | review | approved`の文書と定義する。metadata内にstate／Approvalを探さない。Lifecycle headが`superseded`の文書とDecisionはactive件数へ含めない。

正本件数をDecision本文の完了条件へ固定しない。Indexはdocument metadataから生成し、並び順はDirectory order、文書ID byte順とする。期待件数は検査結果として表示できるが、Architecture invariantにはしない。

### 7.6 Authority discoveryとarchitecture explain projection

旧worktreeにあった別authority Manifestの要件である主題の一意Owner、正本path／anchor、version／content hash、許可されたprojection、MCD参照、変更Review責任は、`ArchitectureMetadataV2`、document relation registry、current Lifecycle head、Shared canonical contracts、生成Architecture Indexへ統合する。別Manifest Schema、互換alias、第二の正本は追加しない。正規authorityはmetadata、registry、Lifecycle headのexact entryからのみ解決し、説明文またはAI要約をauthorityへ昇格させない。

Architecture Governanceは、指定Project revisionの構造を人間とAIへ同じEvidenceで提示するread-only／Disposableな`ArchitectureExplainProjectionV1`を生成する。

```text
ArchitectureExplainProjectionV1
  project_id
  project_revision
  contract_set_hash
  scope
  game_system_entries[0..256]
  state_owner_entries[0..256]
  dependency_edges[0..1024]
  runtime_phase_entries[0..256]
  world_entries[0..256]
  level_entries[0..256]
  streaming_entries[0..256]
  capability_entries[0..256]
  target_entries[0..256]
  save_replay_entries[0..256]
  evidence_refs[1..1024]
  omitted_ranges[0..128]
  continuation
```

入力はexact `ArchitectureMetadataV2` setとcurrent Lifecycle head、document relation registry、Product registry、MCD Contract registry、Commit済みWorld／Level／Streaming Source、Target Profile、Project revisionに限定する。各entryはcanonical concept ID、正規Owner document／Contract、optional runtime phaseまたはlifetime、Source StableId、Source content hash、Evidence参照を保持する。各dependency edgeはsource／target canonical concept ID、relation Contract ID、Owner document、Source StableId／hash、Evidence参照を保持する。自然言語説明、Editor配置、外部Engine用語、AIの推測からOwner、edge、phaseを生成しない。

各categoryは256 entry、dependencyは1,024 edge、全canonical encodingは2 MiBを上限とする。上限超過は要約で隠さず`omitted_ranges`とhash-bound `continuation`を返す。Continuation payloadは`request_hash`、`source_closure_hash`、`revision`、`scope`、`expires_at`を持ち、token digestを次に固定する。

```text
SHA-256(JCS({request_hash, source_closure_hash, revision, scope, expires_at}))
```

`request_hash`はcontinuation自体を除くrequest、明示`evaluation_time`、field mask、Target Profile ref、category別next offsetをcanonical化して含み、`source_closure_hash`はmetadata、document relation、Product／Contract registry、World／Target Sourceのhash closureを含む。`expires_at`と検証時刻`evaluation_time`はrequestの明示入力としてcanonical UTCで固定し、generator／encoderがwall clockから生成しない。`expires_at <= evaluation_time`、別revision、scope、field mask、Target、offset、Source closureへの再利用、digest不一致、範囲外offset、必要Evidence欠落は`diagnostic.architecture.explain-continuation-invalid`でfail closedにする。

このdigestはrepository-owned secretやauthenticity署名ではなく、入力binding、破損、誤再利用を検出するread-only cursor integrity値である。悪意あるcallerによるdigest再計算を権限証明として扱わず、continuationからCommit、Approval、Owner変更のauthorityを得ない。authorityを要する操作は既存Approval Contractへ委譲する。同一入力、明示`expires_at`、Source closureからは同一canonical bytesを生成する。

このprojectionはauthority発見と説明のための派生物であり、Project正本、ChangeSet、MCD、Approval、Owner登録を変更しない。変更を行うconsumerはprojection entryを直接Commitせず、canonical Stable ID、typed Operation、expected revisionを正規Gatewayへ再指定する。

`ArchitectureComprehensionCaseV1`と`ArchitectureComprehensionFixtureV1`のSchema、Corpus、grader、昇格GateはAI Verification／Provenanceだけが所有する。Control Planeはexact metadata closure、relation／registry hash、deterministic explain schema hashをEvidence inputとして供給し、Eval期待値または合否を所有しない。Comprehension fixtureはControl Plane baseline read-back後にだけ実行し、stale metadata、omitted Evidence、invalid continuation、存在しないOwner／phaseをnegative Caseとして扱う。

## 8. Compatibility／Evolution model

### 8.1 Version dimensions

| Dimension | Format | 用途 |
|---|---|---|
| Engine release | SemVer | Public capabilityとsupport policy |
| Contract type major | Type suffix `V1`, `V2` | incompatible Schema boundary |
| Contract revision | monotonic `uint32 revision`＋exact `schema_sha256` | 同一major内のadditive／fix revision |
| Contract set | SHA-256 | 一つのBuildが使用するexact closure |
| Native ABI profile | major＋exact profile hash | Native binary load可否 |
| Artifact format | format major／minor＋content hash | Package／Save／Content decode |
| Capability activation | closed state | 利用可能性。Versionではない |

`revision`は同じContract type major内で1から単調増加し、欠番は許すが再利用しない。`schema_sha256`はMCD canonical Schema bytesのSHA-256であり、同じrevisionに別hashを割り当てない。Reader／Writer negotiationと永続参照は両方を記録する。

```text
McdContractRefV1 {
  contract_id, type_major, revision
  schema_sha256, contract_set_sha256
}

ArtifactRefV1 {
  artifact_kind_id
  format_major, format_revision, format_schema_sha256
  content_size
  content_sha256: Sha256DigestV1
}
```

従来の`McdContractRefV1.version`と`ArtifactRefV1.schema_version`はこのFieldへclean置換する。Ref内にrange、bare version、path、URI、`latest`を入れない。Logical Source identityが必要な場合は別subject refで保持し、`ArtifactRefV1`はexact immutable bytesだけを指す。

`latest`、version rangeだけの永続参照、maturityを含むID、document statusを含むIDを禁止する。

### 8.2 Schema evolution

同一major内で許可する変更は次に限定する。

- optional Field追加。ただしdefaultまたはabsence semanticsをOwnerが一意に定義する。
- closed enumを変更しない補足metadata追加。
- 既存の受理集合、既定値、diagnostic identityを変えないvalidator実装修正。以前validだったartifactを拒否する修正は同一majorのfixとして扱わない。
- 既存Field ID、wire type、unit、coordinate、meaningを変えない説明改善。

Major更新を必要とする変更は次である。

- required Field追加。
- Field削除、Field ID変更、wire type／unit／coordinate／meaning変更。
- closed enum value追加または削除。ただし最初からversioned extension registryとして定義したFieldを除く。
- cardinality、range、ordering、default、failure semanticsの非互換変更。
- authority、owner、security boundary、persistent identityの変更。

削除済みField IDとenum codeは永久欠番とし、再利用しない。Suffixなしaliasは作らない。

Unknown Field判定は、Envelopeに記録されたexact type major、revision、schema hashを解決した後に行う。そのSchemaにないFieldは拒否する。新readerは同一majorかつsupport window内の旧revision decoderとfixtureを保持する。Saveは同じepoch内の全Shipping revisionを保持する。旧readerへ未来revisionを読ませない。

Live IPCでrevision negotiationが必要な場合、双方が列挙したexact `{revision, schema_sha256}`集合の最大共通要素を選ぶ。Range、`latest`、hashなしrevisionは使わない。永続artifactはOwnerのcurrent revisionを明記して出力し、暗黙downgradeしない。旧revision writerが必要なら専用encoder、fixture、support期限をCompatibility Registryへ登録する。

### 8.3 Artifact class policy

| Artifact class | Compatibility policy |
|---|---|
| Internal cache／BMI／device cache | 互換なし。exact identity mismatchで破棄し再生成 |
| Derived Artifact／Runtime Package／Application Package | exact Sourceからfull rebuild。旧reader／dual schemaを残さない |
| Project Source | committed revisionを失わない。versioned offline migration、Before／After fixture、rollback必須 |
| Save | 同じ`save_compatibility_epoch`で一度Shippingしたversionを後続releaseが読めることを必須化 |
| Replay | exact baseline／contract setを原則とし、明示Replay migrationがある場合だけcross-version再生 |
| User setting／profile | Saveと同じくmigrationまたは明示reset approvalが必要 |
| Pack Source | minimum Engine contract、Feature dependency DAG、migration、exact target qualification |
| Native Game binary | exact ABI profile／contract set。Sourceからrebuildし、暗黙binary shimを作らない |
| Public C++ source API | Post-1.0はSemVer。BMIをToolchain間配布しない |

`save_compatibility_epoch`変更は通常のmajor更新でも自動許可しない。全既存Saveのmigrationを提供するか、User data resetを伴うProduct Decision、明示UI、backup、rollback、Approvalを必要とする。

`SupportPolicyV1`の既定は次とする。

- Pre-1.0の未Shipping内部artifactはclean break可能だが、外部配布済みProject／Save／User settingにはこの例外を適用しない。
- Public C++ source APIとPublic Contractは同じEngine major内の全released minor／patchをsupportする。削除は次majorだけで行い、security emergencyを除き少なくとも一つ前のminorでdeprecation diagnosticとmigration guideを出す。
- Project Sourceはcurrent majorと直前majorからcurrent majorへのdirect offline migrationをsupportする。それ以前は署名・hash固定したarchived migratorをmajor順に適用し、元revisionを上書きしない。
- Saveは同じepochの全Shipping revision、Replayはexact baseline、Packはmanifestが宣言したminimum Engine contractとdependency closureだけをsupportする。
- Security revokeは通常windowを短縮できるが、Product／Security Decision、影響inventory、safe replacement、User通知、rollback可否を必須とする。

### 8.4 Change classification

| Class | 例 | 必須closure |
|---|---|---|
| `internal_rebuild_only` | cache layout、private adapter | clean rebuild、affected fixture、Package再生成 |
| `additive_contract` | optional Field、new independent capability | old fixture、new fixture、new reader／old writer互換 |
| `migration_required` | Project／Save schema変更 | migrator、Before／After、idempotence、rollback、Package binding |
| `breaking_pre1` | 未公開内部契約のclean break | caller、fixture、docs、artifact全同時更新 |
| `breaking_released` | Public major／Save epoch／ABI break | Product Decision、support plan、migrationまたは明示非互換、全Target Qualification |
| `security_revoke` | compromised key／dependency／provider | fail closed、deactivation、affected inventory、safe replacement、incident Evidence |

`ArchitectureChangeSetV1`はchanged document、contract、Field、Capability、Target、Package、Save、Pack、fixture、migration、approvalのclosureを持つ。影響項目が空であることを「影響なし」の根拠にせず、lintとOwner reviewで照合する。

### 8.5 Canonical byte列とhash

- 永続／wire ContractはExecutable Contractsが所有する`McdCanonicalBinaryV1`で一意のbyte列へencodeする。JSON表示、Markdown、filesystem列挙順、pointer値、locale、時刻をidentity入力にしない。
- Digest型は`Sha256DigestV1`へ統一し、lowercase 64桁hexを表示形式、32 bytesをwire形式とする。Algorithm省略や別algorithmの同じField名を禁止する。
- Manifestは自身のdigestまたは後段artifactのdigestをFieldに持たない。`ManifestArtifactEnvelopeV1`がexact `manifest_ref: ArtifactRefV1`、`payload_root_sha256`、`entry_count`、`trust_profile_ref`を外側で束ねる。Manifest digestは`manifest_ref.content_sha256`だけに置き、別Fieldへ複写しない。Signature refはEnvelopeへ入れず、既存`MirakanSignedRecordV1`がEnvelopeの`ArtifactRefV1`を外側からbindする。
- `payload_root_sha256`はpathをUTF-8 NFC、`/`区切り、case-sensitive logical pathへ正規化し、path byte順に並べた`{path, size, sha256, executable_kind}`のcanonical vectorから計算する。ManifestとEnvelope自身はそのvectorへ含めない。
- `*_refs[]`など集合Fieldは要素canonical bytesのbyte順に並べ、duplicateを拒否する。意味上の順序が必要なsequenceは`order_key`またはdependency DAGをSchemaに明記し、serializer入力順を意味にしない。
- Payload entryはregular fileだけを許可し、空path、absolute path、`.`／`..` segment、backslash、NUL、symlink、hardlink、device、socket、alternate data stream、sparse entry、undeclared nested executableを拒否する。Target固有reserved name／path長制約は各Platform Ownerが追加する。
- Project内のintegrity／trust signatureはcanonical Envelope bytesの`ArtifactRefV1`を`MirakanSignedRecordV1`のJCS subjectへbindする。Platform code／package signingは各formatが定めるbytesを対象にし、署名後のReceiptがcanonical Envelope ref、unsigned artifact ref、signed artifact refを別Fieldでbindする。
- Hash mismatch、duplicate normalized path、case-fold collision、Unicode normalization collision、Manifest／Envelopeの自己entryをfail closedで拒否する。

### 8.6 Game compatibility subject

`GameCompatibilitySubjectV1`はSave、Runtime Package、Application Package、最終Candidateを結ぶTarget非依存のimmutable inputであり、V1 revision 1の必須Fieldを次へ固定する。追加は8.2のadditive規則だけで行う。

```text
project_id, project_revision
engine_release_version, engine_baseline_hash
contract_set_hash, gameplay_contract_set_hash
content_semantic_set_hash, project_shader_semantic_set_hash
save_compatibility_epoch, save_schema_set_hash, migration_set_hash
pack_resolved_lock_hash
compatibility_policy_ref, source_provenance_ref
```

Target profile、native binary、Runtime Package、Application Package、signature、Receipt、Candidate hash、wall-clock timestampは含めない。Canonical subject bytesの`ArtifactRefV1`がexact subject refとなる。入力が一つでも変われば新subjectを作り、二つのsubject間の読込可否はhash equalityではなくartifact class別`CompatibilityDecisionV1`とmigration coverageで決める。

### 8.7 新機能／変更の標準フロー

新機能は既存defaultや既存Contractへ暗黙追加せず、次の順序で導入する。

1. Product Planへmaturityなしのstable Capability ID、`target_product_tier`、Owner、fallback、対象外を登録し、Activationは`not_activated`から開始する。
2. 既存Ownerで意味が閉じるかを確認し、閉じなければArchitecture Decisionで新Ownerと非Owner範囲を決める。共有型をconsumer文書へ先に書かない。
3. `ArchitectureChangeSetV1`でartifact class、Schema、Save、Replay、Runtime Package、Application Package、Target、Pack、Security、Performanceへのimpact closureを列挙する。
4. 新しい独立ContractはV1で追加する。既存Contract変更は8.2の分類に従い、additive revisionまたはnew major＋migrationを選ぶ。
5. Feature選択はCapability Registryとtyped composition manifestだけで行う。散在するboolean、directory scan、registration order、Backend存在をActivation条件にしない。
6. Source、Runtime、Save／Replay、Package、Target adapterをtyped Portで接続し、feature absent、unsupported Target、stale Receipt、migration欠落、partial failureのnegative fixtureを先に定義する。
7. `candidate_locked -> qualified -> production`をTarget別Evidenceで進める。未選択Projectと旧Saveの挙動が変わらないことを回帰fixtureで確認する。

既存機能の削除はFeature flagを消すだけで行わない。Public／User data impactを`breaking_released`として扱い、deprecation、Project／Save migration、fallback、Package cleanup、Target Qualification、Approvalを同じChangeSetで閉じる。

## 9. Stable IDと状態モデル

### 9.1 ID grammar

| ID | 規則 |
|---|---|
| Capability | `capability.<domain>.<name>`。C0～C3、candidate、production、`v1`を含めない |
| Work Package | `wp.<domain>.<name>`。Phaseはidentityへ含めずRegistryの`phase_id`で登録 |
| Operation | `operation.<domain>.<verb_noun>`。Schema majorはOperation contract refに置く |
| Schema Contract | `contract.<domain>.<name>`。Type majorはIDでなくSchema type suffixに置く |
| MCD Domain object | Executable Contractsへ登録した`<kind>.<namespace_path>`。例: `game_system.<domain>.<name>`。version／maturityは専用Field |
| Diagnostic | `diagnostic.<domain>.<condition>` |
| Driver／Profile | logical IDにversionとmaturityを含めない。Versionは専用Field、exact artifactはlockで解決 |

document ID、MCD object、Product Registry logical IDは[Naming／Project Layout §3.2](../architecture/02-foundation/naming-project-layout.md#32-stable-idとoperation)のkind別grammarに従う。すべてのkindで数字開始segment、同一segment内の`-`／`_`混在、maturity／version埋込みを拒否する。Capabilityのmaturity、Work PackageのPhase、Driver／Profileのversionは別Fieldで保持し、昇格、再計画、更新でlogical IDを変えない。旧IDは同じChangeSet内のoffline migration tableで新IDへ一度だけ写像し、runtime aliasを残さない。

### 9.2 State axes

| Axis | Closed state | Owner |
|---|---|---|
| Document lifecycle | `draft \| review \| approved \| superseded` | Architecture Governance |
| Capability product tier | `C0 \| C1 \| C2 \| C3`。到達scopeの分類でありstate transitionではない | Product Plan |
| Capability activation | `not_activated \| candidate_locked \| qualified \| production` | Product Plan |
| Work Package scheduling | `declared \| ready \| active \| blocked \| deferred \| complete` | Product Plan |
| MCD registration | `draft \| active \| retired` | Executable Contracts |
| Target readiness | `predicted \| blocked \| qualified` | Project State |
| Target blocked reason | closed `TargetBlockedReasonRegistryV1`。初期値は`optimization_required \| performance_envelope_unqualified`で各entryが`DiagnosticRefV1`を持つ | 各Domainが意味／回復Gateを登録、Project Stateがenvelope／ID一意性を所有 |
| Evidence freshness | `fresh \| expiring \| expired \| revoked` | AI Verification／Provenance |
| Dependency adoption | `proposed \| locked \| qualified \| active \| rejected \| revoked` | Toolchain／Dependencies |

`CapabilityRegistryV1`は`target_product_tier`を持つがactivation scalarを持たない。Capability activationの唯一の保存正本はcurrent `ProductOperationalStateSnapshotV1`内の`CapabilityTargetActivationStateV1` `{capability_id,target_id}` exact rowであり、aggregateはrequired Target rowの最小stateとしてread-only導出する。C2を目標にした未実装Capabilityは`target_product_tier=C2`で、各required Target rowが`state=not_activated`である。Tierやaggregateを実装済み表示、Receipt、保存stateへ使わない。`C2CapabilityCoverageMatrixV1`は対象Activation rowと`owner_work_package_ref`をjoinするread-only projectionとし、freshnessをEvidenceから都度導出する。`optimization_required`をCapability activation stateとして使わず、Target readinessの`blocked_reason_ref`として扱う。Dependencyへ`candidate_locked`を使わない。

## 10. Persistence／Save design

### 10.1 Canonical contracts

| Contract | Owner responsibility |
|---|---|
| `SaveSlotManifestV1` | User-visible slot identity、active generation、timestampsの非権威metadata、conflict state |
| `SaveRootManifestV1` | `GameCompatibilitySubjectV1`、Engine／Contract／Content／Pack closure、Save epoch、checkpoint ref |
| `SaveCheckpointV1` | published tick、World set、System record set、RNG／clock、authoritative digest |
| `SaveDomainRecordSetV1` | owner別bounded record envelope。Domain field意味はOwner Contract参照 |
| `SaveMigrationPlanV1` | source／destination revision、ordered steps、owner、precondition、rollback |
| `SaveLoadPlanV1` | exact package／content／migration／capacity／external preparation closure |
| `SaveStoragePolicyV1` | integrity、confidentiality、compression、stored／decoded size bound、backup generation、key profile要求 |
| `SaveStorageEnvelopeV1` | canonical payload digest、stored byte digest、transform profile、nonce／auth tag ref、key version。Key自体は含めない |
| `PlatformStorageTransactionV1` | temporary write、flush、read-back、atomic replaceまたはjournal要求 |
| `SaveCommitReceiptV1` | written generation、read-back hash、commit marker、previous generation |
| `SaveLoadReceiptV1` | migration、staging、digest、publication result |

`SaveCatalogV1`はUI／Settings文書のlocal projectionであり、payload正本ではない。UIは`SaveSlotManifestV1`のbounded projectionを表示し、Save bytesを解釈しない。

### 10.2 Capture sequence

1. GameplayまたはUIがtyped Save requestを発行する。
2. Scheduling Ownerが許可されたpublished tick boundaryを選ぶ。
3. Persistence coordinatorがactive Root、Section、System、Save contract、Content、Pack lockをfreezeする。
4. 各Owner projectorが`SaveReplayContractV1`に列挙されたFieldだけをcanonical recordへ出力する。
5. ECS OwnerはPersistent Identity、lifecycle、composition、enablement、Field projectionを出し、raw handle／chunk／row／paddingを出さない。
6. Persistence coordinatorが全recordのOwner、schema、bound、ordering、digest、cross-referenceを検証する。
7. Canonical payloadとexact `SaveStoragePolicyV1`を一つの`PlatformStorageTransactionV1`でPlatform adapterへ渡す。
8. Platform adapterが`SaveStorageEnvelopeV1`を作り、temporary write、flush、再読込hash、atomic replaceまたはrecoverable journal、commit markerを順に実行する。
9. Plaintext／stored digestとread-backが一致した後だけ`SaveCommitReceiptV1`を確定しactive generationを切り替える。

失敗時は旧active generationを維持する。部分System Save、last-write-wins、同generation競合の自動選択を禁止する。

### 10.3 Load sequence

1. `SaveStorageEnvelopeV1`をbounded parseし、stored size、stored digest、trust／authentication、key versionを検証する。
2. Policy bound内でdecrypt／decompressし、canonical payload sizeとdigestを検証する。
3. `SaveRootManifestV1`のSave epoch、`GameCompatibilitySubjectV1`、Schema set、checkpoint refを検証する。
4. exact Runtime Package、Content Package、Pack lockとmigration coverageを解決する。
5. `SaveMigrationPlanV1`をsource revisionからdestination revisionまで一意に構成する。0経路または複数経路は拒否する。
6. World／System State／external reservationを非公開stagingへ構築する。
7. Domain owner順ではなく、明示migration dependency DAGのtopological orderでmigrationする。
8. Identity、composition、capacity、Owner、Contract、authoritative digestを検証する。
9. 全検査成功後だけ一つのpublication boundaryで新Worldを公開する。
10. 失敗時はstagingを破棄し、Titleまたは旧Worldを維持する。

Platform cloud conflictはtransportが検出し、Persistenceが二つのgenerationをUserへ提示する。Save payloadをfield単位で自動mergeしない。

### 10.4 Replay relation

`ReplaySliceV1.start_checkpoint`は曖昧なinline objectをやめ、exact `ArtifactRefV1`で`SaveCheckpointV1`またはReplay専用の同一checkpoint envelopeを参照する。Replay event streamはcheckpoint以後のaccepted Command／Event、clock、RNG、content generationを持ち、Save slot metadataへ依存しない。

### 10.5 Storage／security envelope

Saveのcanonical payload digestはcompression／encryption前の`McdCanonicalBinaryV1` bytesから計算する。Platform adapterは`SaveStoragePolicyV1`に従い、bounded compressionとplatform protected encryptionを適用し、stored bytesのdigestとauthentication結果を`SaveStorageEnvelopeV1`へ記録する。Integrity検証は全Profileで必須、`confidentiality_mode=none`はProduct／Platform policyが明示許可したoffline profileだけに限定する。

Loadはstored size bound、stored digest、auth tag、key versionを検証してからdecryptし、decoded size／compression ratio bound内でdecompressし、最後にcanonical payload digestとSchemaを検証する。Key materialをSave、Project、log、Receiptへ書かない。Key rotationまたはstorage transform変更はplaintext digestとSave Schemaを変えず、新generationへのrewrap transactionとして行い、旧generationをread-back完了まで維持する。

## 11. Runtime Package design

`RuntimePackageManifestV1` revision 1の必須Fieldを次へ固定する。追加は8.2のadditive規則だけで行う。

```text
runtime_package_id, format_major, format_minor
engine_release_version, engine_baseline_hash
project_revision, game_compatibility_subject_ref
target_profile_ref, toolchain_lock_hash
contract_set_hash, native_abi_profile_hash
system_implementation_set_ref
runtime_world_root_refs[]
content_package_refs[]
project_shader_artifact_refs[]
save_compatibility_epoch
save_schema_set_hash, migration_set_hash
pack_lock_hash
entrypoint_ref, dependency_entries[]
```

Manifest内で`GameCompatibilitySubjectV1`と重複するEngine、Project、Contract、Save、Pack Fieldはすべてsubjectの値とbyte equalityにし、片側だけの更新を拒否する。重複Fieldはloaderのearly diagnosticとTarget package検索用projectionであり、別Authorityではない。

`RuntimePackageArtifactV1`は`ManifestArtifactEnvelopeV1`を使い、`runtime_package_manifest_ref: ArtifactRefV1`、`payload_root_sha256`、entry count、`runtime_trust_profile_ref`、optional `MirakanSignedRecordV1` refをManifest外で保持する。Development profileでもdigest検証は必須で、Shipping profileはTarget policyが要求する署名またはplatform trust Receiptを必須とする。

LoaderはEnvelope、Manifest hash、payload root、trust／signature、Engine baseline、Compatibility subject、Target、ABI、Contract、System、World、Content、Shader、Save migration、Packの順に検証する。順序はdiagnosticの一意性のため固定し、一つでも不一致なら何もactivateしない。

Development loose layoutも同じmanifestとvalidationを使用する。Path scanやDirectory存在だけでload可否を決めない。Runtime Packageはimmutableであり、hot reloadは新generationをstagingしてsafe boundaryで全closureを切り替える。

## 12. Application Package／Release design

### 12.1 Common assembly manifest

`ApplicationPackageAssemblyManifestV1` revision 1の必須Fieldを次へ固定する。追加は8.2のadditive規則だけで行う。

```text
assembly_id, target_profile_ref, distribution_profile_ref
engine_baseline_hash, project_revision, game_compatibility_subject_ref
runtime_package_ref, content_package_refs[]
platform_binary_refs[], shader_artifact_refs[]
resource_manifest_ref, permission_policy_ref
privacy_policy_ref, entitlement_policy_ref
store_submission_declaration_ref
sbom_ref, provenance_ref
normalized_entries[] { path, size, sha256, executable_kind }
```

`store_submission_declaration_ref`は同じTarget／Distributionへ束縛した`StoreSubmissionDeclarationV1`を参照する。型はApplication Package／Release Ownerが所有し、revision 1を次に固定する。

```text
StoreSubmissionDeclarationV1
  declaration_id
  schema_version: 1
  target_profile_ref
  distribution_profile_ref
  store_family: not_applicable | microsoft_store | google_play | apple_app_store
  content_rating_system_ref?
  content_rating_answers_artifact_ref?
  age_rating_result_ref?
  data_safety_declaration_ref?
  app_privacy_declaration_ref?
  privacy_policy_ref?
  generated_content_policy_ref?
  reviewer_refs[]
  receipt_refs[]
  revision
  content_sha256
```

`store_family=not_applicable`はprivate artifact／non-store distributionだけに許可し、`?`付きStore Fieldを持たない。Store familyではcontent rating system、回答Artifact、age rating結果、privacy policy、generated content policy、1件以上のReviewer／Receiptを必須とする。Google Playはさらにdata safety declaration、Apple App Storeはapp privacy declarationを必須とし、他familyのPlatform固有Fieldを持たない。各Platform OwnerはIARC／age rating、Data Safety／App Privacy等の時点依存requirementを`StorePolicyLock`と公式source refへ投影し、共通schema、Candidate identity、提出時read-back規則を再定義しない。Manifestの`privacy_policy_ref`とDeclaration内の同Fieldはbyte equality、Target／Distributionも完全一致を必須とする。

AssemblyはContent／Shaderを独自選択しない。`engine_baseline_hash`、`project_revision`、`game_compatibility_subject_ref`、Target、Runtime Package refを検証し、`content_package_refs[]`と`shader_artifact_refs[]`はRuntime Package closureとset equalityにする。Platform固有Resourceだけを`resource_manifest_ref`へ追加し、Runtime closureとの差分を一般Contentとして隠さない。

Windows、Android、Appleは共通manifestを参照するTarget固有envelopeを持つ。

- `UnsignedWindowsPackageV1`
- `UnsignedAndroidPackageV1`。既存`UnsignedMobilePackageV1`を明確化して置換する。
- `UnsignedApplePayloadV1`

共通`UnsignedApplicationPackageV1` envelopeは`assembly_manifest_ref: ArtifactRefV1`、`payload_root_sha256`、entry count、Target固有envelope refを持つ。Manifest digestは`assembly_manifest_ref.content_sha256`だけに置く。Manifest、共通envelope、Target固有envelopeはpayload entryへ含めず、8.5の外側hash規則で束ねる。

### 12.2 Target package preparation

```text
planned
  -> built
  -> validated
  -> signing_authorized
  -> signed
  -> signature_verified
  -> staging_uploaded
  -> staging_read_back_verified
  -> prepared
```

`TargetPackagePreparationTransactionV1`は一TargetのCandidate入力を作る。Release Coordinatorだけがstate writerであり、各遷移はprevious state、immutable input hash、Receipt refをcompare-and-swapする。Transaction／attempt identityはUUIDv7 `StableId`、retryは同じtransactionを巻き戻さず、新attempt IDと`parent_attempt_ref`を同じimmutable inputへ結ぶ。Source、manifest、Policy、Targetのhashが変われば新transactionである。任意段階から`failed`へ遷移できる。`staging_kind`は`store_draft | internal_test_track | private_artifact_registry`のclosed enumとする。`staging_uploaded`はいずれも非公開であり、販売、public rolloutを許可しない。

Build identityはSigning／Upload secretを持たない。Signing identityはSource、compiler、arbitrary scriptを受けない。Upload identityはSourceとprivate keyを持たない。Signing Serviceはimmutable unsigned Target artifactと`UnsignedApplicationPackageV1`だけを受け、`assembly_manifest_ref`のcontent digestと`payload_root_sha256`を再計算する。Platform署名は各formatが定めるbytesへ施し、`ReleaseSigningReceiptV1`がcanonical envelope hash、unsigned artifact hash、signed artifact hashを同時にbindする。

`PackageValidationReceiptV1`、`ReleaseSigningReceiptV1`、`StoreStagingUploadReceiptV1`、`StoreStagingReadBackReceiptV1`が同じexact assembly manifest refとunsigned payload rootを参照した場合だけ`TargetPackagePreparationRecordV1`を発行できる。

### 12.3 Candidate identityと循環防止

`GameCompatibilitySubjectV1`はPackage生成前に確定できるTarget非依存入力であり、Runtime Package、Application assembly、Save Rootへ埋め込める。`TargetPackagePreparationRecordV1`は一Targetの署名／staging upload／read-back完了を表す。AI Security／Approval Ownerの`GameCandidateManifestV1`は複数Target recordとwhole-game Attestationを集約する後段Governance recordであり、前段ManifestまたはPackage payloadへ埋め込まない。

```text
GameCompatibilitySubjectV1
  -> RuntimePackageArtifactV1
  -> UnsignedApplicationPackageV1
  -> signed target artifact + Receipts
  -> TargetPackagePreparationRecordV1
  -> AI Security `GameCandidateManifestV1`
```

`TargetPackagePreparationRecordV1`はsubject ref、Target profile ref、assembly manifest ref、unsigned／signed／staged artifact ref、全Receipt ref、release policy refを持つ。AI Security OwnerはCandidateへsubject refとTarget preparation record refsを登録する。Candidate hashは最終Manifestの外側Envelopeで計算するため、Package hashとCandidate hashの循環を作らない。

### 12.4 Store publication transaction

```text
planned
  -> candidate_validated
  -> human_approval_validated
  -> submitted
  -> accepted
  -> rollout_authorized
  -> publication_requested
  -> public_read_back_verified
  -> complete
```

`ReleaseTransactionV1`はexact `GameCandidateManifestV1`、有効な`HumanGameplayApprovalV1`と`GameActivationReceiptV1`、Target preparation record集合、Distribution policy、rollout policyだけを入力にする。全refは同じCandidate hashへ閉じなければならない。Release Coordinatorだけがstate writerで、attempt／retry規則はTarget preparationと同じである。`submitted`以後のStore mutationはUpload／Release identityだけが行い、Source、Build tool、Signing keyを受けない。Store rejectionは`rejected`、技術失敗は`failed`、人間またはPolicyによる公開前中止は`cancelled`とし、いずれもterminalである。Retryは新attempt IDを作る。

`publication_requested`はStore APIがcommandを受理しただけであり公開成功を意味しない。公開channel、version、region、artifact hashをremote read-backし、Candidateと一致した`StorePublicationReceiptV1`発行後だけ`complete`へ進む。段階rolloutの割合変更、停止、rollbackは同じCandidateを指す別`ReleaseRolloutCommandV1`とReceiptで記録し、Build、Package、Candidate identityを変更しない。

## 13. Shared canonical contract Owner整理

### 13.1 新規／変更する横断Contract

| Contract | 唯一のOwner |
|---|---|
| Architecture metadata JSON schema、`ArchitectureChangeSetV1` | Architecture Governance |
| `EngineContractVersionV1`、`EngineCompatibilityRangeV1`、`PackCompatibilityRangeV1`、`CompatibilityDecisionV1`、`MigrationCoverageV1`、`SupportPolicyV1`、`GameCompatibilitySubjectV1` | Compatibility／Evolution |
| `McdContractRefV1`、`ArtifactRefV1`、`Sha256DigestV1`、`ManifestArtifactEnvelopeV1` | Executable Contracts。Compatibility規則を消費するが構造はここだけで定義 |
| `MirakanSignedRecordV1`の共通envelope／hash chain | AI Verification／Provenance。Algorithm、Key用途、authorization policyはAI Security／Approval |
| `TechnicalQualificationReceiptPayloadV1`、`TechnicalQualificationReceiptV1`、Evidence freshness policy／four-state derivation | AI Verification／Provenance。共通`MirakanSignedRecordV1`とAI Security／ApprovalのKey Policyを消費する。Target readiness envelopeはProject State、測定内容は各Technical Owner |
| `ActiveProductDefinitionBundleV1`／Closure配下の`CapabilityRegistryV1`、`CapabilityTargetActivationBindingRegistryV1`、`ProductReleaseGateRegistryV1`、`ProductPhaseRegistryV1`、`PhaseFixtureBindingRegistryV1`、`WorkPackageRegistryV1`、`TargetProfileRegistryV1`、`RequirementRegistryV1`、`FixtureRegistryV1`、`FallbackRegistryV1`、`ProductRiskDefinitionRegistryV1`、`ProductDecisionGateRegistryV1`、`WorkPackageLifecyclePolicyRegistryV1`、`ProductOperationalStatePolicyRegistryV1`、別hash domainの`FuturePortfolioDefinitionBundleV1`／Closure／`FutureCapabilityIncubationRegistryV1`／`FuturePortfolioPolicyRegistryV1`／`ProductClaimDefinitionRegistryV1`／`FuturePortfolioApprovalV1`、別chain／artifactの`ProductOperationalStateSnapshotV1`／transition、`ProductOperationalDecisionV1`、`ProductOperationApprovalSubjectV1`、`WorkPackageLifecycleRecordV1`、`WorkPackageDefinitionSeedBindingV1`、`ArtifactCandidateBindingV1`、`ControlPlaneRebaselineCandidateBindingV1`、`ProductDefinitionRowMigrationManifestV1`、`ActiveProductDefinitionMigrationV1`、`ControlPlaneBaselineRebindingV1`、`FutureToActivePromotionManifestV1`、`FutureProductClaimReleaseV1` | Product Plan |
| `ControlPlaneBootstrapApprovalPayloadV1`、`ControlPlaneBootstrapApprovalV1` | Architecture Governance。AI Verification／Provenanceの`MirakanSignedRecordV1` schemaとAI Security／ApprovalのR4主体・Key Policyを消費するが、Product／Capability Approvalを代替しない |
| `OfflineGovernanceRootProfileV1`、`OfflineGovernanceEvidenceBindingV1`、`OfflineGovernanceAssuranceEvidenceBindingV1`、`OfflineGovernanceAssuranceEvidencePayloadV1`、`OfflineGovernanceAssuranceEvidenceV1`、`OfflineGovernanceAssuranceEvaluationJournalRowV1`、`OfflineGovernanceVendorRevocationObservationV1`、`OfflineGovernanceOcspNonceReservationV1`、`OfflineGovernanceOcspNonceConsumptionV1`、`OfflineGovernanceAssuranceIssuerQuarantineObligationV1`、`OfflineGovernanceAssuranceIssuerQuarantineFulfillmentV1`、`OfflineGovernanceAssuranceEvidencePolicyV1`、`OfflineGovernanceTrustedIssuerManifestV1`、`OfflineGovernanceAssuranceIssuerRevocationSnapshotV1`、`VendorSignatureAlgorithmV1`、`OfflineGovernanceVendorChainPolicyV1`、`OfflineGovernanceVendorRevocationEvidenceV1`、`OfflineGovernanceObservedFingerprintManifestV1`、`OfflineGovernanceVerificationChannelManifestV1`、`OfflineGovernanceAssuranceStaticPolicyChangeManifestV1`、`OfflineGovernanceAssuranceStaticPolicyChangeSubjectV1`、`OfflineGovernanceAssuranceStaticPolicyChangeApprovalPayloadV1`、`OfflineGovernanceAssuranceStaticPolicyChangeApprovalV1`、Rotation／Recovery／Root Head／Root Security Incident、generation-local Root Revocation Snapshot／Head、global Root Revocation Ledger／Super Head、Recovery Readiness Receipt／Head／Incident／Invalidation、`OfflineGovernanceRecoveryReadinessRemediationAuthorizationPayloadV1`、`OfflineGovernanceRecoveryReadinessRemediationAuthorizationV1`、`OfflineGovernanceAssuranceIssuerRevocationDetectionSubjectV1`、`OfflineGovernanceAssuranceIssuerRevocationDetectionPayloadV1`、`OfflineGovernanceAssuranceIssuerRevocationDetectionV1`、`OfflineGovernanceAssuranceIssuerFailStopPayloadV1`、`OfflineGovernanceAssuranceIssuerFailStopV1`、`OfflineGovernanceWitnessAttestationV1`、`BootstrapVerifierProfileV1`、`OfflineGovernanceSchemaCatalogV1`、`OfflineGovernanceDualVerifierComparisonManifestV1`、`OfflineGovernanceVerifierUpgradeAuthorizationPayloadV1`、`OfflineGovernanceVerifierUpgradeAuthorizationV1`、`BootstrapSchemaMaterializationPlanV1`、`AuthorityBindingDerivationPolicyV1`／`AuthorityBindingSourceCatalogV1`／Head、`ControlPlaneConstructionAuthorizationV1`／Task Receipt、六Trust Registry／Closure／Head、`TrustRegistryChangeEvidenceBindingV1`、`TrustRegistryChangeEvidencePolicyV1`、`TrustRegistryChangeIndependentR4SubjectV1`、`TrustRegistryChangeIndependentR4ApprovalEvidencePayloadV1`、`TrustRegistryChangeIndependentR4ApprovalEvidenceV1`、`TrustRegistryChangeManifestV1`／Approval、`TrustRegistryRecoveryAuthorizationV1`、`ControlPlaneGovernanceEvidenceBindingV1`、`ControlPlaneGovernanceRecoveryPolicyV1`、`ControlPlaneGovernanceIncidentPayloadV1`、`ControlPlaneGovernanceIncidentV1`、`TrustRegistryQuarantineIncidentPayloadV1`、`TrustRegistryQuarantineIncidentV1`、`TrustRegistryQuarantineRecoveryAuthorizationPayloadV1`、`TrustRegistryQuarantineRecoveryAuthorizationV1`、`ControlPlaneRebaselineCoreV1`／Approval／Envelope／Transaction、`ControlPlaneRecoveryRebaselineAuthorizationPayloadV1`、`ControlPlaneRecoveryRebaselineAuthorizationV1`、`CurrentControlPlaneBaselineBindingV1`、`ArchitectureMetadataV2`、`ArchitectureDocumentCoreV1`、`ArchitectureDocumentChangeManifestV1`、`DocumentLifecycleRecordV1` | Architecture Governance。表示名はfamily要約で、機械正本は直後のOwner set-equality規則とする。署名envelopeとProduct publication wrapperは各Ownerの正本を消費し再定義しない |
| `SaveSlotManifestV1`、`SaveRootManifestV1`、`SaveCheckpointV1`、`SaveDomainRecordSetV1`、`SaveMigrationPlanV1`、`SaveLoadPlanV1`、`SaveStoragePolicyV1`、`SaveStorageEnvelopeV1`、`PlatformStorageTransactionV1`、Save Receipt群 | Persistence／Save。Key管理／OS crypto実装は各Platform Owner |
| `RuntimePackageManifestV1`、`RuntimePackageArtifactV1` | Runtime Package |
| `ApplicationPackageAssemblyManifestV1`、`StoreSubmissionDeclarationV1`、`UnsignedApplicationPackageV1`、Target mapping rule、`PackageValidationReceiptV1`、`TargetPackagePreparationTransactionV1`、`ReleaseTransactionV1`、`ReleaseSigningReceiptV1`、`StoreStagingUploadReceiptV1`、`StoreStagingReadBackReceiptV1`、`TargetPackagePreparationRecordV1`、`StorePublicationReceiptV1`、`ReleaseRolloutCommandV1` | Application Package／Release |
| `UnsignedWindowsPackageV1`／`UnsignedAndroidPackageV1`／`UnsignedApplePayloadV1`のTarget固有Fieldとformat | Windows／Android／Appleの各Platform Owner |
| `GameCandidateManifestV1`、`GameActivationReceiptV1`、`HumanGameplayApprovalV1` | AI Security／Approval。Target packageの内部構造は再定義せず、`TargetPackagePreparationRecordV1`を参照 |
| `PackManifestV1`、`PackResolvedLockV1` | Pack Contract |

Owner一意性の機械正本は、完成`LocalSchemaCatalogV1.members[]`のうち`owner_document_id=mirakan.arch.architecture-governance`であるschema ID集合である。上表のArchitecture Governance行は可読用family要約であり列挙正本ではない。Validatorはこの集合をArchitecture Governance文書のschema宣言集合とset equality比較し、さらに`OfflineGovernanceSchemaCatalogV1.members[]`の同Owner subsetとfull Local Catalogの同じschema ref／hash／signature slotがbyte-exact包含されることを検証する。missing／extra／duplicate Owner、unknown owner document、family略記からの型推論を拒否し、Bootstrap Materialization PlanのTask 0／2 unionもfull member集合とset equalityにする。

### 13.2 既存の未確定projection

| 現在の型／概念 | 決定するOwner／処置 |
|---|---|
| `TargetProfileRef` | `TargetProfileRefV1`へ改名。構造はExecutable Contracts、値とQualificationは各Platform |
| `TargetCapabilitySnapshotV1` | 共通envelopeはExecutable Contracts、Target固有entryは各Platform |
| `TargetBlockedReasonRegistryV1` | Registry構造と共通値はProject State、Domain固有`DiagnosticRefV1`は各Domain Ownerが一意登録 |
| `LightingBudgetEnvelopeV1` | Performance／Capacity |
| `PostProcessBudgetEnvelopeV1` | Performance／Capacity |
| `EnvironmentLightingSummaryV1` | Environment／Surfaces |
| `MaterialReadabilitySummaryV1` | Materials |
| `CameraPresentationSummaryV1` | Camera |
| `LayerCompositionSummaryV1` | Render Graph |
| `AccessibilityPolicySnapshotV1` | UI／Text／Localization／Accessibility |
| `PostProcessVolumeSummaryV1` | Post Processing |
| `PolicySnapshotV1` | generic名を廃止し`LightingResolutionPolicySnapshotV1`としてLightingが所有 |
| `StableSourcePathV1` | Asset Lifecycleで構造、normalization、case／Unicode規則を定義 |
| `ProviderManifest` | `ProviderManifestV1`としてAI Security／Approvalが所有 |
| `ReleaseTransactionV1` | Application Package／Release |

Consumer文書にはfield一覧を複写せず、Owner link、revision、freshness、read-only、write-back禁止だけを残す。

## 14. Pack evolution

`PackManifestV1`のV1 revision 1は[Pack Contract](../architecture/08-packs/pack-contract.md)が所有する次の必須Fieldへ固定する。ConsumerはFieldを追加、別名化、互換alias化しない。

```text
pack_id, pack_version, pack_kind, content_hash
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
license_ref, provenance_ref
```

`pack_kind`は`feature | genre`のclosed enumとする。Feature間依存だけをDAGとして許可し、Genre Pack間dependencyを拒否する。`pack_version`はSemVer 2.0.0のvalid version、`content_hash`はexact artifact identityであり、同一Pack ID／versionの別hash、dependency cycle、欠落、lock未選択の複数解を拒否する。Resolverは`latest`を自動選択せず、`PackResolvedLockV1`へexact Pack version、content hash、Engine baseline、Contract set、Feature dependency closureを固定し、そのlockに対するTarget別Qualification Receiptを要求する。

Patch／minorでもpersisted Source、Save／Replay、Profile、Recipeが変わる場合はmigrationと再Qualificationを要求する。Majorはoffline migrationと明示承認を必要とする。失敗時は旧Pack version、Project revision、registry head、last-valid artifactを維持し、Runtime shim、旧名alias、synthetic dependencyを作らない。

Shooterは`pack_kind=genre`であり、Profile／Capability IDからmaturityを除去する。C1はRegistryの`target_product_tier`列、現在状態は別軸のActivation stateが持つ。

## 15. Product、Phase、Work Package整合

Product Planの`WorkPackageRegistryV1`を唯一のschema正本として消費する。各entryは次のFieldだけを持つ。

```text
work_package_id
phase_id
owner_document_id
target_refs[]
fallback_ref
provided_fixture_refs[]
required_capability_refs[]
requires_work_package_refs[]
scheduling_state
defer_reason
reconsideration_gate_refs[]
blocked_reason_ref
```

旧`requirement_refs[]`、`capability_refs[]`、`exit_fixture_refs[]`、`schedule_state`、`completion_receipt_refs[]`をcompatibility aliasとして受理しない。Requirement／Phase completionは`PhaseFixtureBindingRegistryV1`だけが束縛し、`provided_fixture_refs[]`は実装寄与を表すだけでWork Package単独の完了Gateではない。Definition内`scheduling_state`はgenesis seedでありapproved revision内ではimmutable、current stateは`ProductOperationalStateSnapshotV1.work_package_lifecycle_heads[]`からだけ導出する。完了を含むstate transitionの監査はDefinition entryへ追記せず、Product Ownerのappend-only `WorkPackageLifecycleRecordV1`へ分離する。

`WorkPackageLifecyclePayloadV1`、`WorkPackageLifecycleRecordV1 {payload, signed_record}`、ID derivation、previous／sequence／CAS、candidate binding、purpose別Service Role、signed wrapper、exact transition policy、Decision／Owner approval nullabilityは[Product Plan §11.1](../architecture/00-product/product-plan.md#111-registry共通規則)を一字違わず消費し、本書でFieldを複写または別schemaを発明しない。最初のRecordはDefinition seedからsequence 1、以後はcurrent signed wrapper headをpreviousとして`N+1`にする。Recordとnext operational snapshotを同一atomic transactionでpublishし、同じheadからのfork、sequence gap／duplicate、ID／payload mismatchを拒否する。`complete`はfresh Owner acceptance、Target closure、artifact candidateを必須とするが、通常Work PackageへPhase exitを要求しない。Phase completionは`PhaseFixtureBindingRegistryV1`が独立評価する。Recordを削除／上書きして巻き戻さず、補正／resetは登録済みPolicyまたはatomic Active Definition migrationだけで行う。

2D coverage Work Packageの正本IDはProduct Plan §11.5の`wp.product.general-coverage-2d`であり、Registryの`phase_id`でPhase 8へ登録する。将来Phaseを移してもIDは変えない。旧表記`WP7a3_2d_product_coverage_c2`がsigned Task 1 inventoryに存在しない場合はTask 4のID migration manifestへrowを作らない。Capability Coverage MatrixはWork Package scheduleとCapability activationを再定義しない。

`requires_work_package_refs`はcycleを禁止し、PrerequisiteのPhase ordinalがConsumerより後なら拒否する。同Phase内はtopological orderを使う。current stateを`ready`へ進めるRecordは全Prerequisite headが`complete`、Ownerがapproved、current Product operational headの`control_plane_baseline_binding`がsnapshotのactive definitionと一致し、tag branchに応じて有効なBootstrapまたはRebaseline Approval／Envelope（rebaselineではTransactionも）をcurrent revocation／freshness込みでread-backできる場合だけ許可する。Definition seedの`deferred`はnon-empty `defer_reason`と1件以上の`reconsideration_gate_refs[]`、`blocked`はnon-null `blocked_reason_ref`を必須とし、Definitionから削除しない。他seed stateでは`defer_reason=null`、`reconsideration_gate_refs=[]`、`blocked_reason_ref=null`を必須とする。

各Work Packageの`required_capability_refs[]`は、各consumer `target_refs[]`について二つの独立条件を満たさなければならない。第一に、参照Capabilityの`CapabilityRegistryV1.target_bindings[]`へexact `target_id` bindingが存在し、その`scope=required`であること。第二に、§26のoperational Activation closure検証が成功しexact `{capability_id,target_id}` rowが一件だけ存在すること。binding missing／`scope=optional | excluded`、Activation rowのmissing／duplicate／extraのいずれでも要求Capability closureとaggregateを拒否する。optional selectionは現schemaに存在せずrequired edgeへ読み替えない。別Targetのauthoring host／build hostが必要な場合は`requires_work_package_refs[]`で順序を表し、consumer Targetに存在しないruntime Capability edgeを作らない。

Phase Gateは二種類を明示する。

- `contract_fixture_gate`: 後続Target実装前でもheadless schema／negative fixtureとして実行できる。
- `product_target_gate`: 実Platform、package、install、device Receiptが必要で、該当Phase前には成功扱いにしない。

Phase 4 `phase.ai-authoring-mvp-a`はDefinition-firstまたは事前Qualification済みPackに限定する。`wp.authoring.prequalified-source-packs`は既存Pack／Variantの選択とprovenanceだけを提供し、このPhaseで新しいNative／Shader Sourceを生成またはactivateしない。`gate.product.phase-4-ai-mvp-a`は`fixture.product.shooter-2d`でAI AuthoringとMVP completionを検証するが、`requirement.product.project-source-activation`を評価しない。

新規Project SourceはPhase 5 `phase.external-agent`で、`wp.authoring.project-native-module`（Owner `mirakan.arch.native-game-module`）と`wp.rendering.project-shader`（Owner `mirakan.arch.rendering-project-shader`）を独立実装し、aggregate `wp.product.project-source-activation`が両方のSource／Diff／Code owner Approval／Target artifact／Qualification Receiptを同一Project revisionへ閉じる。専用`gate.product.phase-5-project-source-activation`だけが`requirement.product.project-source-activation`を評価する。`gate.product.phase-5-external-agent`はProposal-only境界であり、外部Client接続の成功をSource activationの代用にしない。`capability.project.native_module`、`capability.project.shader`、`capability.product.project-source-activation`は`target.windows.editor`と`target.windows.desktop`だけをrequiredとし、Android／Appleは別Qualificationまで`excluded`、Windows Receiptから推論しない。

Scheduling Phase 0のSave／Platform lifecycle項目はcontract fixtureに限定する。Windowsの空Scene save／packageはPhase 2、Android／Apple packageはPhase 7のproduct target gateとする。C++ Modules Phase 0のMobile recipeはcompile／link fixtureであり、Store package合格を意味しない。

## 16. Runtime ECS Decisionとの関係

Runtime ECS設計のarchetype／SoA、closed transition、single writer、access manifest、staged structural transaction、canonical iteration、raw memory非永続化は維持する。ただしactive化前に次を修正する。

昇格先は`docs/architecture/04-runtime/entity-component-system.md`、stable document IDは`mirakan.arch.runtime-entity-component-system`とする。Decision fileをactive specとして数えず、昇格時のDecision refはmetadataでなく完成Lifecycle Approval sidecarへ束縛する。

1. 「既存Save owner」参照を新Persistence／Save正本へ置換する。
2. ECS Save sectionはComponent／Entity projectionだけを所有し、slot／aggregate／migration orchestration／file transactionを所有しない。
3. `DerivedArtifactManifestV2`は、未実装のsuffixなし型から移るなら`DerivedArtifactManifestV1`としてclean定義する。V2を使う場合は実在するreleased V1とmigration履歴を示す。（ECS Decision本文は現行`DerivedArtifactManifestV1`へ統一済みであり、本項のV2→V1改名分は完了している。）
4. Runtime World Root／Section artifactはRuntime Package manifestとContent Group closureへ接続する。
5. `McdCanonicalBinaryV1`をCompatibility／EvolutionのSchema evolution規則へ接続する。
6. `NativeSystemDescriptorV1`変更をNative ABI compatibility classへ登録する。
7. 16 KiB chunkは`ecs_chunk_soa_v1` profile内の実装値であり、Save、Replay、Artifact identityへ露出しないことを維持する。
8. Decision内の固定文書件数を削除し、生成Indexを参照する。
9. ECS Decision本文のrebaseは全体ChangeSet内で行えるが、ECS active specへの昇格は5新正本とECSが直接`requires`する全Ownerの`approved`後に別Approvalを必要とする。

## 17. Toolchain correction

Toolchain／DependenciesのFreeType 2.14.1はannotated tag objectとpeeled commitを分離する。

```text
tag: VER-2-14-1
tag_object: 3bd82b5f543bc84ccf2b1d0cdb63b95218099ee6
peeled_commit: 526ec5c47b9ebccc4754c85ac0c0cdf7c85a5e9b
```

TypeScript 7.0.2はnative compilerでprogrammatic APIを持たないため、既存どおりCLIだけを使用し、正式Artifactでは`--singleThreaded`を固定する。Ajv 8.20.0は§21のDraft 2020-12 validationだけに使い、Engine runtime、MCD semantic validation、Authorizationへ持ち込まない。npm artifactはversion、tarball URL、size、SHA-256、registry integrity、metadata response SHA-256を必須lockとする。`registry_provenance_state`は`verified | not_provided`のclosed stateとし、signature／attestationが存在すれば検証失敗を拒否、存在しなければ`not_provided`を明記してofficial repository tag／commitとartifact digestで補完する。`gitHead`はregistry metadataに存在する場合だけexact値、欠落時は`null`とし推測しない。

OpenAI Model IDはimmutable snapshotが提供される場合はsnapshotを使う。非snapshot IDの場合は同じ出力の再現を主張せず、resolved ID、Provider manifest、request parameters、tool／schema hash、Eval、expiry、Receiptで採用判断を再現する。

## 18. 公式一次資料と検証記録

外部仕様に対しては公式一次資料をconstraintとして使い、Miraikanai固有のOwner分割、Save transaction、Package identityを「外部製品の公式標準」とは表現しない。後者は一次資料とsigned Task 1 inventoryのcurrent仕様を根拠にした本Projectの設計判断である。

| 検証対象 | 公式一次資料 | 本設計への反映 |
|---|---|---|
| Public release version | [Semantic Versioning 2.0.0](https://semver.org/) | Public APIを宣言し、Post-1.0のbreaking／additive／fixをmajor／minor／patchへ対応 |
| Field進化 | [Protocol Buffers best practices](https://protobuf.dev/best-practices/dos-donts/) | Field number／enum code再利用禁止、required追加・default変更・型変更を非互換扱い。MCD固有規則としてさらにstrict化 |
| Canonical hash | [Protocol Buffers serialization is not canonical](https://protobuf.dev/programming-guides/serialization-not-canonical/) | 汎用serializer出力をhash正本にせず、`McdCanonicalBinaryV1`のcanonical bytesだけを使用 |
| Engine／Extension rangeの比較例 | [Godot `.gdextension` compatibility](https://docs.godotengine.org/en/4.4/tutorials/scripting/gdextension/gdextension_file.html) | minimumだけでなく上限も表せるtyped rangeとexact resolved lockを分離。Godot format自体は採用しない |
| TypeScript 7 | [Microsoft TypeScript 7.0 announcement](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/) | 7.0はstable programmatic APIを持たないためCLIのみ。正式lint buildへ`--singleThreaded`を固定 |
| JSON Schema validator | [Ajv JSON Schema versions](https://ajv.js.org/json-schema.html#draft-2020-12)、[npm Ajv 8.20.0 metadata](https://registry.npmjs.org/ajv/8.20.0) | Draft 2020-12専用`ajv/dist/2020`、strict validation、single-error、local `$id` allowlist、offline lock／integrity read-back |
| MCP | [MCP specification 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25) | Protocol versionをexact lockし、Tool schema validation、access control、人間確認を既存Governanceへ接続 |
| X.509 certificate／CRL | [RFC 5280](https://www.rfc-editor.org/info/rfc5280/)、[RFC 10007](https://www.rfc-editor.org/info/rfc10007/) | certificate path、CRL、critical extension、revocation reasonに加え、v3 CRL issuer certificateの`keyUsage`存在＋`cRLSign`を検証。direct complete CRLのみ、delta／indirect拒否、algorithm allowlistはMiraikanaiのstrict subset |
| OCSP／nonce | [RFC 6960](https://www.rfc-editor.org/info/rfc6960/)、[RFC 9654](https://www.rfc-editor.org/info/rfc9654/)、[RFC 9919](https://www.rfc-editor.org/info/rfc9919/) | BasicOCSPResponse、same-CA-key responder authorization、ResponderID、status／time、RFC 9919のSHA-256 CertID、RFC 9654の32-octet requester nonceを検証。RFC 9919 high-volume profile全体は採用せず、nonce必須、response age、fixed parserはProject strict policy |
| Digital signature／Key lifecycle | [NIST FIPS 186-5](https://csrc.nist.gov/pubs/fips/186-5/final)、[NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final) | ECDSA／RSA-PSS候補、Key用途分離、cryptoperiod／compromise recoveryの外部根拠。P-256 governance固定、P1363 low-S、threshold／custodian分離はProject固有strict policy |
| OpenAI model | [GPT-5.6 Sol official model reference](https://developers.openai.com/api/docs/models/gpt-5.6-sol) | 2026-07-22時点で別のdated snapshot IDを確認できないため、non-snapshot扱いでresolved ID、Eval、expiry、Receiptを記録 |
| Android toolchain | [AGP 9.3.0 release notes](https://developer.android.com/build/releases/agp-9-3-0-release-notes) | Gradle 9.5.0、Build Tools 36.0.0、JDK 17、maximum API 37を既存pinと照合。明示NDK pinは別lockで維持 |
| Apple toolchain | [Xcode 26.6 release notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-26_6-release-notes) | Xcode 26.6と26.5 SDK群を既存pinと照合 |
| Store staging／publish | [Google Play tracks／draft／staged rollout](https://developers.google.com/android-publisher/tracks)、[App Store submission](https://developer.apple.com/documentation/appstoreconnectapi/app-store-version-submissions)、[Microsoft Store submission API](https://learn.microsoft.com/en-us/windows/apps/publish/store-submission-api) | 非公開preparationと公開transactionを分離し、Target adapterが外部stateを共通closed stateへ写像。公開後はremote read-back Receiptを要求 |
| FreeType tag identity | [FreeType official repository tag](https://gitlab.freedesktop.org/freetype/freetype/-/tags/VER-2-14-1) | `git ls-remote --tags`でtag object `3bd82...`とpeeled commit `526ec...`を区別 |

Context7の`/microsoft/typescript-go`は照会時点でpreview／main branch資料を返し、stable 7.0.2をversion indexへまだ収録していなかった。この時間差は[Microsoftの7.0 GA資料](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/)と[npm registryの7.0.2 metadata](https://registry.npmjs.org/typescript/7.0.2)で補完し、Context7のpreview記述をGA挙動より優先しない。

Node、npm、CMake、Ninja、LLVM、DXC、vcpkg、Rendering／Physics／Navigation／Text dependencyのexact release、artifact size、digest、licenseは既存[Toolchain／Dependenciesの一次資料表](../architecture/02-foundation/toolchain-dependencies.md#10-context7と公式一次資料)を正本とする。本設計実行時にもremote refとdownloaded artifactを再検証し、設計レビュー日の結果だけを将来Buildの証拠へ流用しない。

## 19. 全active spec影響マトリクス

| Existing spec | 本横断ChangeSetの必須変更／後続boundary |
|---|---|
| `00-product/product-plan.md` | state axes、Work Package Registry、Capability ID、Phase gate分類、C2 Matrix統合 |
| `01-governance/ai-security-approval.md` | `ProviderManifestV1`、Release authorization、`GameCandidateManifestV1`のsubject／Target preparation record参照 |
| `01-governance/ai-verification-provenance.md` | Package／Save／Compatibility Receipt binding、freshness state |
| `02-foundation/core-architecture.md` | Port／Adapter依存方向図、Feature開始Gateの`approved`定義、新Owner参照 |
| `02-foundation/cpp23-modules.md` | compile fixtureとProduct package gate分離、ABI／BMI compatibility |
| `02-foundation/executable-contracts.md` | Target共通ref／snapshot、Contract revision、shared-owner projection |
| `02-foundation/math-core.md` | metadata relation、Compatibility class参照。algorithm変更なし |
| `02-foundation/memory-pointers.md` | metadata relation、ECS固有値の移管前提、ABI／serialization禁止参照 |
| `02-foundation/naming-project-layout.md` | ID grammar、Schema suffix例外0、migration path規則 |
| `02-foundation/toolchain-dependencies.md` | Dependency adoption state、FreeType peeled commit、npm provenance。[D3D12 Companion]後続ChangeSetでAgility SDK／D3D12MAのexact pin、release evidence、runtime inspectionを接続 |
| `03-authoring/asset-lifecycle.md` | suffixless型修正、`StableSourcePathV1`、`DerivedArtifactManifestV1`、Runtime Package handoff |
| `03-authoring/editor-ui-framework.md` | document relation、Package closure、shared contract ref。UI algorithm変更なし。[D3D12 Companion]後続ChangeSetで`ui_d3d12_adapter`を`ui_render_graph_adapter`へ置換し、native ownerの二重化を解消 |
| `03-authoring/editor-workspace-ux.md` | document lifecycle表示、Save／Target readiness表示語彙 |
| `03-authoring/gameplay-programming-model.md` | `SaveReplayContractV1`をPersistenceへ接続、Capability ID、ABI impact |
| `03-authoring/native-game-module.md` | exact ABI compatibility、Package manifest binding、ECS Decision前提 |
| `03-authoring/project-state.md` | Target readiness rename、Project migration、`GameCompatibilitySubjectV1` input、Runtime Package boundary |
| `04-runtime/debugging-observability-replay.md` | Replay start checkpoint、Replay compatibility、SaveとのOwner分離 |
| `04-runtime/performance-capacity.md` | shared Budget Envelope、Qualification／readiness mapping |
| `04-runtime/scheduling-lifetime.md` | Save aggregate Owner移管、capture／load boundary、Phase 0 gate修正 |
| `05-simulation/animation.md` | Save projector／migration、typed relation。algorithm変更なし |
| `05-simulation/collision.md` | Save／Replay projection、typed relation。algorithm変更なし |
| `05-simulation/navigation.md` | Derived artifact／Package／Save relation、Capability ID |
| `05-simulation/physics.md` | Save state projector、Backend adoption state、typed relation |
| `06-rendering/camera.md` | `CameraPresentationSummaryV1` Owner宣言、checkpoint reset relation |
| `06-rendering/environment-surfaces.md` | `EnvironmentLightingSummaryV1` Owner、Capability ID正規化 |
| `06-rendering/lighting.md` | shared projection Owner、generic Policy型改名、Budget ref |
| `06-rendering/lod.md` | Artifact manifest／Runtime Package binding、Capability ID規則 |
| `06-rendering/materials.md` | `MaterialReadabilitySummaryV1` Owner、Artifact compatibility |
| `06-rendering/post-processing.md` | shared projection Owner、Package／history compatibility |
| `06-rendering/project-shader.md` | Runtime／Application Package artifact binding、Frontend compatibility |
| `06-rendering/render-graph.md` | `LayerCompositionSummaryV1` Owner、Target Snapshot、Package validator input。[D3D12 Companion]後続ChangeSetでlogical plan Owner、D3D12 consumer boundary、Enhanced-only conformanceを固定 |
| `06-rendering/vfx-authoring.md` | Capability ID正規化、Source migration、Runtime artifact binding |
| `06-rendering/vfx-runtime.md` | Runtime Package、Save／Replay projector、Capability ref |
| `06-rendering/world.md` | World artifact manifest、Runtime Package、Save checkpoint／migration |
| `07-platform/android.md` | common assembly manifest、`UnsignedAndroidPackageV1`、Play draft／track／rollout mapping、Save adapter |
| `07-platform/apple.md` | common assembly manifest、App Store submission／release mapping、Save adapter |
| `07-platform/audio.md` | Save stable identity、Runtime Package／Application Package entry |
| `07-platform/input.md` | Save／Replay header compatibility、Runtime Package binding |
| `07-platform/mobile-common.md` | `TargetProfileRefV1`、Save adapter boundary、application package common owner、suffixなし`DerivedArtifactManifest`参照（§5.5）の`DerivedArtifactManifestV1`統一 |
| `07-platform/ui-text-localization-accessibility.md` | `AccessibilityPolicySnapshotV1` Owner、Save catalog projection、Package receipt |
| `07-platform/windows.md` | `UnsignedWindowsPackageV1`、common Receipt、Store submission mapping、Save storage adapter。[D3D12 Companion]後続ChangeSetでHWND／OS lifecycleとD3D12 Surface／Device ownerを分離 |
| `08-packs/pack-contract.md` | `PackManifestV1`、4層dependency、resolved lock、migration、last-valid |
| `08-packs/gameplay-features.md` | reusable Gameplay Feature owner、旧Shooter詳細契約のidentity維持と移管 |
| `08-packs/shooter.md` | Shooter Genre identity、Feature Capability mapping、Profile、cross-profile fixture |
| `08-packs/scenario-stage.md` | optional Stage、completion tagged rule、Scope、Save／Replay |

Architecture Indexとsigned inventoryに列挙された全関連Decisionも更新対象とする。Decisionは歴史的判断を改竄せず、staleな固定件数には追記で現在の生成Indexを正本とすることを明示する。Runtime ECS Decisionは未承認設計なので本文を直接修正できる。

## 20. 更新順序

本節は§6.1 ceremonyと同じpre-baseline construction laneを規範順序とする。

1. **Pre-ceremony:** Root Profile／Genesis Ceremony → local Root Revocation genesis Snapshot／Head → Root Head → global Revocation genesis Ledger／Super Head → 初回Recovery Readiness Receipt／Head → 四current pointer → Bootstrap Verifier／Local Schema Catalog／Materialization Plan → Authority Binding Source Catalog／Head → Construction Authorizationの順で外部CASに完成・purpose quorum承認する。
2. **Task 0:** catalog／plan／schema／Readiness bytesを変更せず、planのTask 0 subset、Trust registries／closure／head、purpose別Key、Provisioning Receiptを生成し、pre-ceremony Readinessをclosureへ登録する。
3. **Task 1:** legacy inventory、migration seed、Toolchain／Ajv lockとoffline integrityを固定する。
4. **Task 2:** 外部CASの全schemaをbyte-exact materializeし、Ajv compileとsemantic／signature verifierを実装する。実Approvalは発行しない。
5. **Task 3:** 5新Ownerを`review`で作り、本文core projectionへの独立R4 `DocumentLifecycleRecordV1`でapprovedへ進める。Bootstrap Approvalはまだ発行しない。
6. **Task 4～8:** legacy metadata／ID、parser／diagnostic、DAG／relations、14 Active＋3 Future registryとProduct state／migration／rebinding／rebaseline、Index／CLI／explainを順に実装する。
7. **Task 9:** 全negative、signature、link、anchor、DAG、Owner、type、ID、Phase、E2E trace、deterministic regenerationを通しAを確定する。Task Receiptはsidecarである。
8. **External Future inception F:** A内Future closureへ独立R4 `FuturePortfolioApprovalV1`をsidecar発行し、初回はprevious=null／sequence 1／expected-empty Future current pointer CAS、再試行は§6.1のreuse／renewal規則でcurrent化する。Active／operational stateは変更しない。
9. **External Decision:** A、Task 9 Receipt head、current F Approval／Future closureへ`ControlPlaneConstructionDecisionV1`をsidecar発行する。
10. **Task 10:** A、Decision、current F Approval／Future closureをread-backしてBaseline CoreだけをBへ生成する。
11. **External R4 Approval:** A／F／B／Decision／Task 10 Receipt headへ`ControlPlaneBootstrapApprovalV1`を署名しCへ追加する。
12. **Task 10A:** Cを特例inputとしてApproval、current Fを再検証しBaseline EnvelopeをDへ生成する。
13. **Task 10B:** Dとcurrent F read-back後の単一transaction時刻でgenesis sequence 0、Control Plane lifecycle sequence 1、snapshot sequence 1を生成しEへatomic CASする。成功後にTask 10B Receiptをsidecar発行する。

各Taskは局所lintとsidecar pass Receipt後だけ次へ進む。自動処理はOwner、Construction Decision、Bootstrap Approvalを自己発行せず、drift時はA以前なら該当Task、A以後なら変更scopeに応じF reuse／renewal、新Decision、Bootstrap Approval ceremonyから再開する。D3D12／Runtime ECSの実装はEとcurrent baseline binding read-back後の別Work Packageであり、このconstruction laneへ混在させない。

## 21. Architecture lint design

実装計画では、既に固定されたNode.js／TypeScript toolchainと、JSON Schema Draft 2020-12 validatorとしてAjv `8.20.0`だけをProduction dependencyに持つ`tools/architecture_lint/`を追加する。TypeScript compiler programmatic APIには依存せず、CLIでcompileしたvalidatorをNodeで実行する。AjvはMIT license、npm tarball `https://registry.npmjs.org/ajv/-/ajv-8.20.0.tgz`、npm integrity `sha512-Thbli+OlOj+iMPYFBVBfJ3OmCAnaSyNn4M1vz9T6Gka5Jt9ba/HIR56joy65tY6kx/FCF5VXNB819Y7/GUrBGA==`へ固定し、Toolchain lock、`package.json`、`package-lock.json`、install後package metadataの四者をread-backする。

Validatorはpackage rootの既定draftに依存せず`ajv/dist/2020`のDraft 2020-12 classを使用する。ES module sourceのexact import specifierは実在fileを指す`ajv/dist/2020.js`とし、これを`ajv/dist/2020` entrypointの実装としてlock／read-backする。`strict=true`、`allErrors=false`とし、`loadSchema`を設定せず、`compileAsync`、network／file URL、実行時schema downloadを禁止する。登録可能なroot `$id`は固定8件の手書きlistでなく次のversioned closureを正本にする。

```text
LocalSchemaCatalogV1
  catalog_id
  format_major = 1
  revision: positive safe integer
  members[]:
    {schema_id, schema_ref, schema_sha256, owner_document_id,
     signature_slots[]:
       {signature_slot_id, branch_id, wrapper_field_path,
        signed_record_purpose,
        issued_at_source_kind = payload_field |
          threshold_subject_authenticated_clock,
        issued_at_field_path: null | canonical field path,
        revocation_context_kind = payload_field | source_trust_closure |
          destination_trust_closure | atomic_candidate_trust_closure |
          offline_governance_signer_generation |
          construction_authorization_snapshot,
        revocation_context_field_path: null | canonical field path,
        authority_source_kind = trust_registry |
          offline_root_profile_purpose_quorum |
          recovery_policy_custodian_quorum |
          construction_authorization_executor_key}}
```

`catalog_id`は同Fieldを除くJCS hashから`urn:mirakan:local-schema-catalog:sha256:<lowercase-hex>`として導出する。membersはschema IDのunsigned UTF-8 byte順・重複なしで、current `OfflineGovernanceSchemaCatalogV1`の全memberを同じschema ref／hash／signature slotで包含し、Control Plane、Task Receipt、trust registries／head、Bootstrap／Rebaseline、共通署名／Qualification、14 Active Definition registry、Future portfolio、Product operational／Decision／migration／claimを含む全root schema fileとset equalityにする。各fileのroot `$id`、ref解決bytes hash、Ownerをcatalog rowと一致させる。各`signature_slots[]`は`signature_slot_id`順・重複なしで、schema bytes内のclosed `x-mirakan-signature-slots` annotationとbyte-exact set equalityにする。`branch_id=always`またはschemaのnamed `oneOf` branch ID、`wrapper_field_path`は当該signed wrapper位置を一意に示し、同一purposeのsource／destination署名、Rotation／Verifier Upgradeのold／new、Recoveryのuncompromised／custodian／newを別slotとして列挙する。instanceに存在しないbranch slotはinactive、存在するslotはexactly once activeとし、Field名やnullable状態からauthorityを推測しない。unknown／missing／extra schemaまたはsignature slotを拒否する。full catalog ref／hashはConstruction Authorization、Bootstrap Materialization Plan、Baseline Core、後続Rebaseline Coreへ束縛するが、Bootstrap Verifier Profile／Root Profileはoffline catalogだけを束縛する。稼働後のfull catalog変更はTrust Change Manifest／Approval、Authority Catalog／Trust closure atomic CAS、same-definition rebaselineまたはDefinition migrationを必須とし、Task 1以降またはcurrent Baseline内の無承認変更を拒否する。

`x-mirakan-signature-slots`の各rowはauthority sourceだけでなくissued-at source／field path、revocation context kind／field pathも必須で、`payload_field`の場合だけpathをnon-null、他kindはpath nullまたは当該kindが定義する唯一のtyped relation pathにする。Local Schema Catalog row、Authority Binding Source Catalog row、schema annotationの三者を`{schema_id,signature_slot_id}`でset equality比較し、multi-slot schemaをschema ID重複として拒否しない。annotation missing／extra、slot ID collision、source／destination時刻またはrevocation context差、inactive branch署名を一原因fixtureで拒否する。

compile前walkはfragment-only `$ref`またはcatalog `schema_id`＋fragmentだけを許し、HTTP(S)、file URL、catalog外URN、dynamic network loadを拒否する。全catalog memberをAjvへ登録してから全rootをcompileし、一件でもmissing／duplicate `$id`、hash差、unresolved refがあれば対象schema validationへ進まない。

Schema作成testを先に走らせない。実装順は、Ajv lock／offline install／integrity read-back、`ajv/dist/2020.js` importとembedded minimal Draft 2020-12 schema compile、対象schema未存在test、対象schema作成の順とする。Validator importまたはlock照合が失敗した状態で`ENOENT`を期待結果にしてはならない。

Validatorは次を検査する。

1. active spec inventoryとIndex一致。
2. H1直後のexact `ArchitectureMetadataV2`、document ID、pathと、別current `DocumentLifecycleRecordV1` headのstate／Approval hashの一意性と整合。metadata内のstate／approval Fieldを拒否する。
3. `requires`の存在、self edge、cycle、redundant transitive edge。Directory番号をdependency layerと推測しない。
4. `integrates_with`のreciprocal edgeとContract ID集合一致。
5. Local Markdown linkとGitHub互換heading anchor。
6. Shared canonical contract Ownerの一意性とconsumer参照。
7. 永続／wire Schema typeの`V<major>` suffix。
8. suffixなしalias、同名別定義、unknown Owner。
9. Capability／Work Package／Operation／Diagnostic ID grammarとorphan ref。
10. Product Phase ID、Runtime phase ID、Target Profile IDのclosed registry参照。
11. Save、Runtime Package、Application Package、Candidate／Approval／Activation、Store publication Receiptの必須E2E edge。
12. Decisionの固定件数を正本Gateとして使用していないこと。
13. Manifest／Envelope／Package／Candidate参照graphのcycle、self hash、後段identityの前段埋込みがないこと。
14. `ControlPlaneBootstrapApprovalV1`のclosed payload／exact署名schema ref、purpose、subject hash、Signer、`payload.approval_authority_ref == signed_record.signer_role_ref`のexact Role ID binding、発行時および`bootstrap_transaction_time`時点のRole／assignment／R4／independence／revocation／Key purpose、Git tree、5 Owner hash、Product registry hash、Toolchain lock hash、Decision hash、historical validity、retroactive revocation判定。単なる後日期限切れとcurrent continuous authorityを同一視しない。
15. Product Planが所有する14件のActive Definition Registryとbundle manifestのset equality、Activation Evidence provider binding、Target別Product Release Gate／tagged production policy、typed Product Decision Gate predicate／action、Work Package Field／Phase相互closure、current Activation row、Risk／Decision effective state、previous／sequence／CAS／Definition epoch付きlifecycle、Artifact origin Task、Product Operation Approval SubjectとDecisionの非循環binding、state transition、row migration manifest、Active Definition full-reset migration、same-definition baseline rebinding、初回bootstrap／rebaseline policy、Future exact 3 Registry、Future 23件とdependency DAG／typed claim definition／Future policy／approval、Future→active origin manifest／claim release、Active／Future revision head、DefinitionとOperational stateのhash分離、`predicted | blocked | qualified` Target readiness、Technical Qualification Receiptのclosed payload／wrapper、exact purpose／Role／Key purpose／subject／型別issued-at／revocation binding、参照Evidence由来の決定論的ID／freshness origin／expiry、current revocation。

出力はstable diagnostic ID、document、line、owner、remediationを持ち、同一入力で同一順序とする。CIは生成Indexとの差分とlint errorが一件でもあれば失敗する。

D3D12固有のsymbol／mapping／descriptor／Tool Catalog検査はCompanionの[Architecture lint追加](2026-07-22-ai-readable-d3d12-backend-design.md#26-architecture-lint追加)が所有し、本validatorの15項へ混入させない。本validatorはD3D12正本が追加された後も、文書ID、Ownerの一意性、typed relation、External Evidence refの存在までを共通Gateとして検査する。

## 22. Verification matrix

| Gate | Positive | Negative |
|---|---|---|
| Document graph | 全`requires`をtopological sort | cycle、self edge、missing docを拒否 |
| Owner | 全shared contractが一Owner | 0 Owner、2 Owner、consumer inline再定義を拒否 |
| Schema | V1型とexact revision解決 | suffixなし、unknown major、Field ID再利用を拒否 |
| Compatibility | new readerがold same-major fixtureを読む | migration欠落、multiple path、range外を拒否 |
| Feature evolution | feature未選択の旧Project／Save／Package挙動が不変 | Backend存在、directory scan、散在booleanによる暗黙Activationを拒否 |
| Save | capture→transform→write→read-back→decrypt／load→digest一致 | partial write、auth／hash mismatch、decompression bound超過、orphan Field、cloud conflict自動mergeを拒否 |
| Runtime Package | exact closureをload | ABI、Contract、Content、Shader、migration mismatchで全体拒否 |
| Application Package／Release | subject→manifest→sign→staging read-back→candidate／approval／activation→publish read-back | manifest外file、identity混在、hash差替え、Candidate hash cycle、未承認公開を拒否 |
| Pack | minimum Engine contract＋Feature dependency DAG＋resolved lock＋Qualification | dependency cycle、Genre間dependency、lock未選択の複数解、qualifiedでないTargetを拒否 |
| Product | Work Package→Capability→Receipt closure | orphan WP、maturity入りID、state軸混同、Activation rowのmissing／duplicate／extraをaggregate前に拒否 |
| Technical Qualification | closed payload→exact `MirakanSignedRecordV1`→Evidence-derived freshness | wrong purpose／Role／Key purpose、payload／envelope binding差、non-pass／revoked Evidence、古いEvidenceの新時刻再包装を拒否 |
| Bootstrap Approval | closed payload→exact authority Role binding／current assignment→`MirakanSignedRecordV1`→二段階Git tree→current read-back | missing／invalid署名、wrong purpose／subject、unknown authority Role、payload／envelope Role差、assignment missing／expired／revoked、R4権限またはindependence不成立、Role／Key purpose不一致、revoked signature、自己参照treeを拒否 |
| ECS prerequisite | 新Ownerへのexact ref | 旧Save owner、unversioned manifest、固定件数前提を拒否 |
| D3D12 companion boundary | 後続計画へのexact link、一つのD3D12 Backend Owner | 6番目の横断Owner扱い、Render Graph／Windows／UIでのnative contract再定義を拒否 |

## 23. Riskとmitigation

| Risk | Mitigation |
|---|---|
| 新正本追加で文書数が増える | 固定件数を廃止し、単一責務、1000行目安、生成Indexで管理 |
| 横断型のOwner移動で大量renameが起きる | Pre-implementation clean changeとして一ChangeSetでconsumer、fixture、Decisionを更新 |
| Compatibility規則が開発速度を落とす | Rebuild可能物は互換対象外とし、User data／Public surfaceだけを厳格化 |
| Save仕様がECS詳細へ依存する | Persistenceはaggregate、ECSはprojectionに限定し、raw storageを永続化しない |
| Package仕様がPlatform差を消してしまう | 共通assembly envelopeとTarget固有envelopeを分離する |
| lintがMarkdown表現に過度依存する | exact metadataと明示Registry／Owner／E2E表だけを機械入力にし、本文自然言語を推測しない |
| signed inventoryの全仕様更新中にcurrent ECS成果を失う | ECS Decisionを保持し、最後に新Ownerへrebaseする。既存commitを上書きしない |

## 24. Release／Qualificationでのみ決まる値

本設計を進めるための未決定事項はない。次の値は設計上の未決ではなく、実装またはRelease CandidateでReceiptにより解決するruntime evidenceである。

- 初回Engine release versionとRelease date。
- 実測Performance、Target device、Store validation result。
- 各Capabilityが`qualified`または`production`へ昇格する時期。
- 非snapshot AI Modelの将来resolved ID。

これらを仮値で仕様へ固定せず、exact lock、Candidate、Receipt、Approvalで決定する。

## 25. Review checklist

- 5新正本の責務が重複していない。
- Governance、Compatibility、Persistence、Runtime Package、Application PackageのAuthorityが分離されている。
- User dataとrebuild可能artifactのpolicyが分離されている。
- Pre-1.0、Post-1.0、Project、Save、Replay、Public APIのsupport windowが区別されている。
- Strict unknown rejectionとadditive evolutionの方向が矛盾していない。
- SaveとReplay、Runtime PackageとContent Package、Application PackageとPlatform formatが混同されていない。
- Capability、Work Package、Document、Contract、Target、Evidence、Dependencyのstate axisが分離されている。
- Shared projectionの全Ownerが決まっている。
- Packのminimum Engine contract、exact dependency closure、exact Qualificationがすべて必要になっている。
- Manifest、Package、Candidateのhash graphに自己参照と循環がない。
- Technical QualificationのTTLが署名済み参照Evidenceの時刻だけから導出され、同じ古いEvidenceの再包装で延長できない。
- Work PackageのPhase、Capabilityのmaturity、Profileのversionがlogical IDに入っていない。
- ECS Decisionが未存在Ownerを参照しない移行順になっている。
- signed Task 1 inventoryの全active specが影響manifestにexactly once現れる。
- 実装がないものを合格済みと表現していない。

## 26. Registry closure

Product-owned execution registryは[Product Plan §11](../architecture/00-product/product-plan.md#11-product-execution-registries)を正本とする。Control Planeは構造と参照整合を検査するが、Product tier、Phase outcome、Target scope、fallback選択を再定義しない。

Bootstrap Approvalへ束縛するimmutable Active Product Definitionは次のRegistry closureである。可変stateとFuture portfolioはこのclosureへ含めない。

1. `CapabilityRegistryV1`
2. `CapabilityTargetActivationBindingRegistryV1`
3. `ProductReleaseGateRegistryV1`
4. `ProductPhaseRegistryV1`
5. `PhaseFixtureBindingRegistryV1`
6. `WorkPackageRegistryV1`
7. `TargetProfileRegistryV1`
8. `RequirementRegistryV1`
9. `FixtureRegistryV1`
10. `FallbackRegistryV1`
11. `ProductRiskDefinitionRegistryV1`
12. `ProductDecisionGateRegistryV1`
13. `WorkPackageLifecyclePolicyRegistryV1`
14. `ProductOperationalStatePolicyRegistryV1`

`FutureCapabilityIncubationRegistryV1`、`FuturePortfolioPolicyRegistryV1`、`ProductClaimDefinitionRegistryV1`のexact 3 Registryは別`FuturePortfolioDefinitionBundleV1`／Closure／`FuturePortfolioApprovalV1`へ閉じる。Closureのregistry ID集合はこの3件とset equalityであり、claim定義集合は全Future rowの`excluded_current_product_claims[]` unionとset equalityにする。planning-only membership revisionはactive Bootstrap／stateを変えず、Future→active migrationだけがdestination Active Product Definitionとactive state migrationを更新する。Future origin、promotion、claim releaseは`FutureToActivePromotionManifestV1`と`FutureProductClaimReleaseV1`だけで検証する。

Operational stateは署名済み`ProductOperationalStateSnapshotV1` chainであり、Activation、Decision Gate評価、Risk評価、WP lifecycle headを持つ。Capability activationの正本keyは`{capability_id,target_id}`である。全`required | optional` bindingにexactly one `CapabilityTargetActivationStateV1` row、全`excluded` bindingにrow 0件を必須とする。missing、duplicate、bindingに対応しないextra row、または`excluded` rowが一件でもある場合はActivation closure全体を拒否し、aggregate入力も結果も返さない。freshnessは保存せずcurrent Evidenceから導出する。closure成功後だけ`required` rowの実在stateから`not_activated < candidate_locked < qualified < production`の最小値をread-only導出し、`optional`はTarget support表示だけに使う。`not_activated`をmissingから合成せず、Aggregateを保存しない。

Work Package Definitionは§15の正本Field集合と完全一致させる。`deferred` seedで理由または再検討Gate欠落、`blocked` seedでDiagnostic欠落、旧Fieldの混入をfail closedにする。Phase gateは`PhaseFixtureBindingRegistryV1`から評価し、planning-onlyの再検討条件は`ProductDecisionGateRegistryV1`のexact IDへ閉じ、transition Receiptは`WorkPackageLifecycleRecordV1`だけへ記録する。Risk／Decisionのdefinitionとoperational evaluationを混在させず、`revisit_gate_or_date`がlogical IDなら登録済みGateへ解決する。Phase、Capability tier、profile versionをlogical IDへ埋め込まない。

## 27. Exact migration authority

```text
ControlPlaneLegacyInventoryPayloadV1
  inventory_id
  authorized_base_git_tree_id
  artifact_entries[]:
    {inventory_entry_id,
     entry_kind = architecture_document | decision,
     repository_path, git_blob_id, content_sha256,
     legacy_metadata_sha256: null | lowercase hex 64}
  relation_entries[]:
    {relation_entry_id, source_inventory_entry_id,
     legacy_relation_kind, legacy_target_token, source_content_sha256}
  logical_id_entries[]:
    {logical_id, logical_id_kind, defining_inventory_entry_id, defining_content_sha256}
  generated_at

ControlPlaneLegacyInventoryV1
  payload: ControlPlaneLegacyInventoryPayloadV1
  executor_signed_record: ConstructionExecutorSignedRecordV1(purpose=control_plane_legacy_inventory)

ControlPlaneMigrationSeedPayloadV1
  seed_id
  legacy_inventory_ref, legacy_inventory_sha256
  artifact_classifications[]:
    {inventory_entry_id, disposition = retained | transformed | removed,
     destination_document_id: null | stable document ID,
     evidence_refs[]}
  relation_classifications[]:
    {relation_entry_id,
     disposition = requires | integrates_with | supersedes | external_evidence | removed,
     destination_source_document_id: null | stable document ID,
     destination_target_document_id: null | stable document ID,
     contract_ids[], evidence_refs[]}
  logical_id_classifications[]:
    {logical_id, disposition = retained | renamed | removed,
     destination_logical_id: null | logical ID, evidence_refs[]}
  generated_at

ControlPlaneMigrationSeedV1
  payload: ControlPlaneMigrationSeedPayloadV1
  executor_signed_record: ConstructionExecutorSignedRecordV1(purpose=control_plane_migration_seed)

ControlPlaneTask1OutputManifestV1
  manifest_id
  legacy_inventory_ref, legacy_inventory_sha256
  migration_seed_ref, migration_seed_sha256

ArchitectureIdentityMigrationManifestPayloadV1
  manifest_id
  migration_seed_ref, migration_seed_sha256
  rows[]:
    {old_logical_id, disposition = retained | renamed | removed,
     new_logical_id: null | logical ID,
     old_defining_content_sha256,
     new_document_core_ref: null | content-addressed ref,
     new_document_core_sha256: null | lowercase hex 64,
     evidence_refs[]}
  generated_at

ArchitectureIdentityMigrationManifestV1
  payload: ArchitectureIdentityMigrationManifestPayloadV1
  executor_signed_record: ConstructionExecutorSignedRecordV1(purpose=architecture_identity_migration_manifest)

ArchitectureRelationMigrationManifestPayloadV1
  manifest_id
  migration_seed_ref, migration_seed_sha256
  rows[]:
    {relation_entry_id,
     disposition = requires | integrates_with | supersedes | external_evidence | removed,
     destination_source_document_id: null | stable document ID,
     destination_target_document_id: null | stable document ID,
     contract_ids[], evidence_refs[]}
  generated_at

ArchitectureRelationMigrationManifestV1
  payload: ArchitectureRelationMigrationManifestPayloadV1
  executor_signed_record: ConstructionExecutorSignedRecordV1(purpose=architecture_relation_migration_manifest)
```

固定Appendixや文書件数を移行authorityにしない。Task 1はAuthorizationのexact base treeから`ControlPlaneLegacyInventoryV1`と`ControlPlaneMigrationSeedV1`を決定論生成するが、R4 Bootstrap Approval完成前の両wrapperはstaged migration Evidence／proposalであって承認済みauthorityではない。Inventoryは全current architecture document／Decision／legacy metadata／relation／logical IDをpath、Git blob、content hash付きでexactly once列挙する。Migration Seedは各inventory rowを`retained | transformed | removed`、全legacy relationを`requires | integrates_with | supersedes | external_evidence | removed`へ一度だけ分類し、Task 1 Receiptのoutput manifest ref／hashへ束縛する。unknown／missing／duplicate row、authorized base tree差、説明文からの補完を拒否する。

Task 4は上記完成Seedだけから`ArchitectureIdentityMigrationManifestV1`と`ArchitectureRelationMigrationManifestV1`を生成し、old／new ID、old／new Core hash、relation diff、理由Evidenceを閉じる。Task 6が生成する`document-relations.v1.json`はdestination manifestとset equality、topological orderはそのDAGから導出する。設計本文の影響マトリクスは変更scopeを説明するだけでmigration authorityの代用ではない。Inventory／Seed／ManifestのschemaはLocal Schema CatalogとTask 0／2 Materialization Planへ含め、完成wrapper、Task Receipt、current bytesをread-backできない移行を開始しない。

ID導出は型別に固定する。`ControlPlaneLegacyInventoryPayloadV1.inventory_id = urn:mirakan:control-plane-legacy-inventory:sha256:<lowercase-hex>`、`ControlPlaneMigrationSeedPayloadV1.seed_id = urn:mirakan:control-plane-migration-seed:sha256:<lowercase-hex>`、`ArchitectureIdentityMigrationManifestPayloadV1.manifest_id = urn:mirakan:architecture-identity-migration-manifest:sha256:<lowercase-hex>`、`ArchitectureRelationMigrationManifestPayloadV1.manifest_id = urn:mirakan:architecture-relation-migration-manifest:sha256:<lowercase-hex>`とし、いずれも当該ID Fieldだけを除くpayload JCS SHA-256をsuffixにする。flat `ControlPlaneTask1OutputManifestV1.manifest_id = urn:mirakan:control-plane-task1-output-manifest:sha256:<lowercase-hex>`は同ID Fieldだけを除く完成object JCS SHA-256から導出し、存在しないpayload projectionをhashしない。`inventory_entry_id`は当該artifact rowから同ID Fieldを除いたclosed object `{entry_kind, repository_path, git_blob_id, content_sha256, legacy_metadata_sha256}`のJCS SHA-256を`urn:mirakan:control-plane-inventory-entry:sha256:<lowercase-hex>`へ付けて導出する。`relation_entry_id`は当該relation rowから同ID Fieldを除いたclosed object `{source_inventory_entry_id, legacy_relation_kind, legacy_target_token, source_content_sha256}`のJCS SHA-256を`urn:mirakan:control-plane-relation-entry:sha256:<lowercase-hex>`へ付けて導出する。`repository_path`はrepository-relative slash形式へ正規化し、`.`／`..` segment、absolute path、Unicode normalization差、case-fold collisionを拒否する。logical IDは既存canonical IDを正本とし、defining inventory ID／content hashへのexact解決を必須にする。全配列は主IDのunsigned UTF-8 byte順・重複なしとし、nested row IDを乱数、列挙順、生成時刻、filesystem inodeから作らない。

Seedのartifact／relation／logical ID classification集合はInventoryの対応集合とそれぞれset equalityで、分類行は上記derived IDをbyte-exactに継承する。`removed`だけdestination Fieldを全null、他dispositionは必要destination Fieldを全non-nullにする。relationの`external_evidence | removed`だけdestination targetをnullにでき、`removed`は`contract_ids=[]`、typed relationはnon-empty canonical Contract IDを必須とする。Task 1 Receiptの`output_manifest_ref/hash`は完成`ControlPlaneTask1OutputManifestV1`を指し、同ManifestのInventory／Seed ref／hashは完成signed wrapperとexact一致させる。Task 4両Manifestのrow集合はSeedのlogical ID／relation classification集合とset equalityで、nullabilityとdispositionをbyte-exactに継承する。Inventory↔Seed欠落、derived ID再計算差、unsigned payload、Task Receiptから到達不能なwrapper、別base tree、empty Evidence、片側nullを拒否する。

R4 Bootstrap approverは`ControlPlaneBootstrapApprovalV1`が直接束縛するTask 10 chain headからprevious ReceiptをTask 1まで逆走し、Task 1 Receipt→完成`ControlPlaneTask1OutputManifestV1`→完成signed Inventory／Seedのexact ref／hash、Authorization base tree、derived row ID、分類set equalityを再計算する。さらにTask 4 Receipt／Identity・Relation Manifest、A subject tree内のdestination projectionが同じSeedへ解決し、Task 1からTask 10までchain断絶／差替えが0件であることを確認する。このexact chain、A、B core、Decision、Inventory／Seedを束縛した完成R4 Bootstrap Approvalが存在して初めて、そのSeedは当該A treeのapproved migration authorityになる。Approval前にSeedで正本文書／IDを変更すること、別tree／別chainへの再利用、hash差を説明で受理することを禁止し、一件でも差があればTask 1生成とR4 Approvalを再実行する。Repositoryのoperational current authorityとしての利用はE atomic publication完了後に限る。

Inventory／Seed／Task 4 migration wrapperと`ControlPlaneTask1OutputManifestV1`はGit tree外のimmutable sidecar Evidenceであり、生成Repository bytesへ署名時刻やsignatureを埋め込まない。Task 1／4は開始時に外部execution journalへ一度だけ`task_execution_time`を予約し、当該Taskの全`generated_at`とexecutor signed record `issued_at`をその値へ固定する。crash retryは保存済みpayload、canonical wrapper bytes、signature、output manifestをbyte-exactに再利用し、新時刻／再署名／別IDを生成しない。repository内のdeterministic migration projectionはこれらのref／hashだけを入力に同じbytesを再生成し、sidecar欠落を成功扱いしない。

## 28. Baseline handoff

Control Plane完了時は自己参照を避けるため、Approval前のcoreとApproval後のenvelopeを分ける。

```text
ControlPlaneBaselineCoreV1
  subject_git_tree_id
  architecture_index_sha256
  document_relation_registry_sha256
  active_product_definition_ref, active_product_definition_sha256
  future_portfolio_definition_ref, future_portfolio_definition_sha256
  future_portfolio_approval_ref, future_portfolio_approval_sha256
  identity_migration_registry_sha256
  architecture_explain_schema_sha256
  toolchain_lock_sha256
  architecture_lint_artifact_sha256
  local_schema_catalog_ref, local_schema_catalog_sha256
  authority_binding_source_catalog_ref, authority_binding_source_catalog_sha256
  authority_binding_source_catalog_head_ref, authority_binding_source_catalog_head_sha256
  offline_governance_root_head_ref, offline_governance_root_head_sha256
  offline_governance_root_profile_ref, offline_governance_root_profile_sha256
  root_revocation_head_ref, root_revocation_head_sha256
  root_revocation_snapshot_ref, root_revocation_snapshot_sha256
  global_root_revocation_super_head_ref, global_root_revocation_super_head_sha256
  global_root_revocation_ledger_ref, global_root_revocation_ledger_sha256
  recovery_readiness_head_ref, recovery_readiness_head_sha256
  trust_registry_closure_head_ref, trust_registry_closure_head_sha256
  trust_registry_closure_ref, trust_registry_closure_sha256
  construction_authorization_ref, construction_authorization_sha256
  trust_provisioning_receipt_ref, trust_provisioning_receipt_sha256
  lint_version

ControlPlaneBaselineEnvelopeV1
  baseline_core_ref
  baseline_core_sha256
  control_plane_bootstrap_approval_ref
  control_plane_bootstrap_approval_sha256
```

Task 10は§6.1のA subject tree IDとFでcurrent化したFuture Approval／closureをread-backしてcoreへ記録し、B自身のtree、core自身、Approvalまたはenvelopeをcoreへ含めない。final Bootstrap Approval payloadがAとcore ref／hashを署名した後、Task 10Aがenvelopeを生成し、Task 10Bがoperational genesisとControl Plane completion lifecycleを発行する。Baselineへ文書件数やedge件数を重複保存しない。Readerはhash照合済み`document-relations.v1.json`から件数を導出し、配列間不一致をbaseline mismatchとして拒否する。上記Field集合は実装計画Task 10／10A／10Bおよびbaseline schemaと完全一致させる。

CoreのRoot／local・global Revocation／status=validのReadiness、Trust closure／Head、Authority Catalog／HeadはTask 10時点の各current pointerとexact一致し、Catalog ref／hashはTrust closure内の同じCatalog bytesと一致させる。`active_product_definition_ref/hash`は承認済みActive closure、`future_portfolio_definition_ref/hash`はBootstrap時点のplanning-only Future closureへcontent-addressed exact解決し、それぞれ隣接hashを再計算する。`future_portfolio_approval_ref/hash`はFでexpected-empty pointerへcurrent化したsequence 1完成`FuturePortfolioApprovalV1` wrapper、同payloadのdefinition hashはCoreのFuture closure hashとexact一致させる。Future値は歴史記録として束縛するだけで、後続Future revisionをActive Definition migrationまたはControl Plane rebaselineの暗黙triggerにしない。Futureのcurrent authorityは別`FuturePortfolioApprovalV1` current pointerから解決する。Active／Future refまたはhash domain混同、Headが指さないclosure／Catalog、hash-onlyでref未解決のCoreを拒否する。

Bootstrap後にfull `subject_git_tree_id`のbytesを変えるbaseline-scoped Work Packageは§6.1.1.1を必須経路とする。対象WPを`active`のままsame-definition P2で新Core／bindingへ切り替え、new bindingに対するtree外final ReceiptとOwner acceptance後だけcompleteにできる。旧bindingのpreliminary Receiptまたはtree内へinlineしたbindingをBaseline handoffの成功根拠にしない。

Runtime ECS、D3D12、通常Critical Path Taskはcurrent Product operational head、同headの`CurrentControlPlaneBaselineBindingV1`、current Definition epochで`complete`な`wp.architecture.control-plane` lifecycle headをread-backし、binding tagに応じBootstrap Core／Approval／EnvelopeまたはRebaseline Core／Approval／Envelope／Transactionの全hashと署名を検証できるまで開始しない。bootstrap branchは§6.1のhistorical validityとretroactive revocation、rebaseline branchはcurrent approval／Qualification freshnessを適用し、両branchで別途current generation-local Revocation Head／Snapshot、global Super Head／Ledger、status=validかつnon-expiredでeffective InvalidationのないRecovery Readiness Head／Receipt、Trust closure／Catalog Headをcontinuous authorityとして検証する。初回Authorizationの単なる期限切れをcurrent authority失効へ読み替えず、初回genesis snapshotまたは初回Approvalを永続currentとして固定しない。値は設計時に仮記入せず、clean treeで全lint／test／Index Gateを通したArtifactから生成する。Mismatchは`diagnostic.architecture.baseline-mismatch`、historical proof quarantineまたはcurrent continuous authority不成立は`diagnostic.architecture.current-control-plane-approval-invalid`で停止し、最新値へ暗黙追従しない。

## 29. 実装計画

実装Task、artifact map、test、command、checkpoint、失敗時停止条件は[Architecture Evolution Control Plane Implementation Plan](2026-07-22-architecture-evolution-control-plane-implementation-plan.md)に固定する。文書／Decision／integration件数とID migration rowは同計画の固定本文から推測せず、§27のsigned Inventory／Seed／Manifestから導出する。本文書または実装計画の承認だけで実装完了、署名ceremony完了、Capability activationを宣言しない。
