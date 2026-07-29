# Miraikanai Engine Product Security Contract

- 文書ID: mirakan.arch.product-security
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Product全体のthreat ownership、security baseline、vulnerability report intake／triage／validation／remediation／security release／disclosure／closure、security update decision、Product security incident
- 非正本範囲: AI task authorization、dependency lock／SBOM generation、Platform signing／privacy、domain input／memory／resource contract、SupportBundle schema、Evidence envelope。各Ownerを参照する
- 規範依存: [Architecture Governance](architecture-governance.md)、[Product Plan](../00-product/product-plan.md)、[Product Lifecycle](../00-product/product-lifecycle.md)、[AI Security／Approval](ai-security-approval.md)、[AI Verification／Provenance](ai-verification-provenance.md)
- 関連文書: [Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Native Game Module](../03-authoring/native-game-module.md)、[Project Shader](../06-rendering/project-shader.md)、[Debugging／Observability／Replay](../04-runtime/debugging-observability-replay.md)、[Windows](../07-platform/windows.md)、[Android](../07-platform/android.md)、[Apple](../07-platform/apple.md)
- 根拠区分: project-decision／official-guidance comparison
- 外部根拠確認日: 2026-07-29

## 1. 結論と所有境界

Product Securityは、Product全体のsecurity subjectにaccountable Ownerを割り当て、脆弱性報告を受けてからvalidation、修正Candidate、security update、通知、開示、incident response、再発防止までを一つのcaseとして閉じる。

AI task authorization、approval tier、managed host authorityは[AI Security／Approval](ai-security-approval.md)、dependency version／license／SBOM generationは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、signing／Store／OS sandbox／privacyは各Platform、input validation／memory／handle／resource contractは各domain Owner、Evidence envelope／signature／retention／revocationは[AI Verification／Provenance](ai-verification-provenance.md)、`SupportBundleV1`は[Debugging／Observability／Replay](../04-runtime/debugging-observability-replay.md)だけが所有する。

Product Securityはdomain Ownerの技術判断を上書きしない。report text、scanner result、外部severity score、issue labelをauthorityにせず、case state、affected release closure、security release、disclosure、notification、incident closureを所有する。

本書はtarget designである。対応するRegistry、Schema、Case Store、Operation、Fixture、Receipt、response team運用はRepositoryに存在せず、未materialize／未Activationである。

## 2. 共通規則

- 全objectはclosedであり、未知Field、重複Field、NaN／Inf、範囲外値を拒否する。
- RefはID、positive version／revision、content hashを持つexact Refである。表示名、path、`latest`、同名release、近いversionへfallbackしない。
- 配列はcanonical identity byte順へstrict sortし、duplicateを拒否する。
- content hashは自己hash Fieldだけを除くclosed recordをtyped canonical JSON projectionへ変換してRFC 8785 JCSでencodeし、型ごとのASCII domain separatorと`uint32_be` byte lengthでframeしたbytesをSHA-256する。Semantic `u32`／`u64`／revision／countはJSON numberへ投影せず、それぞれJSON string `"u32:<digits>"`／`"u64:<digits>"`としてencodeする。`<digits>`はASCII base-10、zeroは`0`、positive型は`1`以上、先頭zero、plus sign、whitespace、指数表記を禁止し、Schema rangeをdecode時に検証する。これにより`2^53`を超える`u64`もIEEE 754 roundingを通らない。TimestampはUTCへ正規化したRFC 3339 string、tagged unionはbranch tagとそのbranchで許可されたFieldだけ、nested exact RefはRef tupleの全Fieldをcanonical key順でencodeする。別integer projection、JSON numberへのdowngrade、自己hash混入、未選択branch Field、map key順差、IDだけの短縮表現を拒否する。
- secret、credential、private key、password、personal dataはCase本文、advisory、notificationへ直接格納しない。Evidence Ownerのredacted artifact refだけを使う。
- Reporterの自然言語、添付file名、URL、command、path、severity、affected versionはuntrusted inputである。実行、open、publish、release state変更へ直接使用しない。
- vulnerability state、Product release state、security update state、disclosure state、incident stateを一つのgeneric statusに統合しない。

本書が新設するhash型のASCII domain separatorは次である。

| 型 | ASCII domain separator |
|---|---|
| `ThreatOwnershipRegistryV1` | `MIRAKAN_THREAT_OWNERSHIP_REGISTRY_V1` |
| `ThreatOwnershipBindingV1` | `MIRAKAN_THREAT_OWNERSHIP_BINDING_V1` |
| `ProductSecurityBaselineV1` | `MIRAKAN_PRODUCT_SECURITY_BASELINE_V1` |
| `ProductSecurityReleaseBindingV1` | `MIRAKAN_PRODUCT_SECURITY_RELEASE_BINDING_V1` |
| `SecurityCaseRegistrySnapshotV1` | `MIRAKAN_SECURITY_CASE_REGISTRY_SNAPSHOT_V1` |
| `SecurityControlV1` | `MIRAKAN_SECURITY_CONTROL_V1` |
| `AcceptedSecurityRiskV1` | `MIRAKAN_ACCEPTED_SECURITY_RISK_V1` |
| `MemorySafetyStrategyV1` | `MIRAKAN_MEMORY_SAFETY_STRATEGY_V1` |
| `VulnerabilityResponsePolicyV1` | `MIRAKAN_VULNERABILITY_RESPONSE_POLICY_V1` |
| `VulnerabilityResponseClassV1` | `MIRAKAN_VULNERABILITY_RESPONSE_CLASS_V1` |
| `SecurityEscalationPolicyV1` | `MIRAKAN_SECURITY_ESCALATION_POLICY_V1` |
| `SecurityEmbargoV1` | `MIRAKAN_SECURITY_EMBARGO_V1` |
| `NotificationAudienceV1` | `MIRAKAN_NOTIFICATION_AUDIENCE_V1` |
| `VulnerabilityCaseV1` | `MIRAKAN_VULNERABILITY_CASE_V1` |
| `SecurityUpdateDecisionV1` | `MIRAKAN_SECURITY_UPDATE_DECISION_V1` |
| `SecurityDisclosureRecordV1` | `MIRAKAN_SECURITY_DISCLOSURE_RECORD_V1` |
| `ProductSecurityIncidentV1` | `MIRAKAN_PRODUCT_SECURITY_INCIDENT_V1` |
| `SecurityControlChangeV1` | `MIRAKAN_SECURITY_CONTROL_CHANGE_V1` |

本書Ownerのexact Refは次のclosed tupleである。

| Ref | Field |
|---|---|
| `ThreatOwnershipRegistryRefV1` | `{registry_id, registry_version, registry_content_hash}` |
| `ThreatOwnershipBindingRefV1` | `{binding_id, binding_version, binding_content_hash}` |
| `ProductSecurityBaselineRefV1` | `{baseline_id, baseline_version, baseline_content_hash}` |
| `ProductSecurityReleaseBindingRefV1` | `{security_release_binding_id, security_release_binding_version, security_release_binding_content_hash}` |
| `SecurityCaseRegistrySnapshotRefV1` | `{case_registry_snapshot_id, case_registry_snapshot_version, case_registry_snapshot_content_hash}` |
| `SecurityControlRefV1` | `{control_id, control_version, control_content_hash}` |
| `AcceptedSecurityRiskRefV1` | `{accepted_risk_id, accepted_risk_version, accepted_risk_content_hash}` |
| `MemorySafetyStrategyRefV1` | `{strategy_id, strategy_version, strategy_content_hash}` |
| `VulnerabilityResponsePolicyRefV1` | `{response_policy_id, response_policy_version, response_policy_content_hash}` |
| `SecurityEscalationPolicyRefV1` | `{escalation_policy_id, escalation_policy_version, escalation_policy_content_hash}` |
| `SecurityEmbargoRefV1` | `{embargo_id, embargo_revision, embargo_content_hash}` |
| `NotificationAudienceRefV1` | `{audience_id, audience_version, audience_content_hash}` |
| `VulnerabilityCaseRefV1` | `{case_id, case_revision, case_content_hash}` |
| `SecurityUpdateDecisionRefV1` | `{decision_id, decision_revision, decision_content_hash}` |
| `SecurityDisclosureRecordRefV1` | `{disclosure_id, disclosure_revision, disclosure_content_hash}` |
| `ProductSecurityIncidentRefV1` | `{incident_id, incident_revision, incident_content_hash}` |
| `SecurityControlChangeRefV1` | `{control_change_id, control_change_version, control_change_content_hash}` |

各RefのFieldは解決先recordとbyte equalityにする。Case／Incidentの新revisionは旧recordを上書きせず、新しいcontent hashとpredecessor refを持つ。IDだけ、issue number、CVE、release label、advisory URLからRefを補完しない。

## 3. Threat ownership

```text
ArchitectureContractFragmentRefV1
  owner_document_id: ArchitectureDocumentId
  owner_document_revision: positive u64
  fragment_stable_id: StableId
  fragment_content_hash: SHA-256

SecuritySubjectRefV1
  {
    kind: architecture_domain,
    owner_contract_fragment_ref: exact ArchitectureContractFragmentRefV1
  }
  | {
    kind: product_release,
    release_ref: exact EngineReleaseBindingRefV1
  }
  | {
    kind: target_package,
    artifact_ref: exact ArtifactRefV1,
    target_profile_ref: exact TargetProfileRefV1
  }
  | {
    kind: dependency,
    dependency_lock_entry_ref: exact DependencyLockEntryRefV1
  }
  | {
    kind: public_contract,
    public_contract_member_ref: exact PublicContractMemberRefV1
  }
  | {
    kind: project_source_surface,
    source_surface_ref: exact ProjectSourceSurfaceRefV1
  }
  | {
    kind: ai_authority_surface,
    authority_profile_ref: exact AiAuthorityProfileRefV1
  }

ThreatOwnershipRegistryV1
  registry_id: StableId
  registry_version: positive u32
  product_definition_ref: exact ActiveProductDefinitionRefV1
  bindings[1..4096]:
    sorted unique ThreatOwnershipBindingV1
  registry_content_hash: SHA-256

ThreatOwnershipBindingV1
  binding_id: StableId
  binding_version: positive u32
  security_subject_ref: exact SecuritySubjectRefV1
  accountable_owner_document_id: ArchitectureDocumentId
  responsible_owner_document_ids[1..32]:
    sorted unique ArchitectureDocumentId
  required_baseline_control_refs[1..256]:
    sorted unique exact SecurityControlRefV1
  escalation_policy_ref: exact SecurityEscalationPolicyRefV1
  binding_content_hash: SHA-256
```

`SecuritySubjectRefV1`は独立content recordへのRefではなくclosed tagged subject keyである。各branchはArchitecture Contract fragmentまたは既存Ownerのexact versioned Refを含み、bare document ID、表示名、path、current document headをsubject identityにしない。`ThreatOwnershipBindingRefV1`は`{binding_id, binding_version, binding_content_hash}`とし、解決先Bindingとbyte equalityにする。RegistryはBinding recordをinline保持し、BindingからRegistryを逆参照しない。

一つのsecurity subjectはexactly one accountable Ownerを持つ。Responsible Ownerは複数でもよいが、accountable Owner欠落、複数accountable、unknown document、retired Owner、control set欠落、循環するescalationを拒否する。

Architecture Inventoryに現れる全public surface、Target package、Dependency、AI authority surfaceはRegistryにexactly one bindingを必要とする。Inventory未materializeのcurrent状態では手動文書監査しかできず、Registry完成またはcoverage済みとは主張しない。

## 4. Product security baseline

```text
SecurityControlV1
  control_id: StableId
  control_version: positive u32
  category:
    threat_model | input_validation | memory_safety | build_hardening
    | dynamic_analysis | dependency_provenance | credential_isolation
    | ai_authority | package_update | evidence_redaction
    | vulnerability_response | incident_response
  accountable_owner_document_id: ArchitectureDocumentId
  owner_contract_fragment_ref: exact ArchitectureContractFragmentRefV1
  applicable_subject_kinds[1..7]:
    sorted unique closed SecuritySubjectRefV1.kind
  required_evidence_class_refs[1..256]:
    sorted unique exact EvidenceClassRefV1
  failure_action:
    reject_candidate | block_release | deactivate_subject | declare_incident
  control_content_hash: SHA-256

AcceptedSecurityRiskV1
  accepted_risk_id: StableId
  accepted_risk_version: positive u32
  security_subject_refs[1..256]:
    sorted unique exact SecuritySubjectRefV1
  accountable_owner_document_id: ArchitectureDocumentId
  rationale_evidence_refs[1..256]:
    sorted unique exact EvidenceRefV1
  compensating_control_refs[1..256]:
    sorted unique exact SecurityControlRefV1
  expires_at: RFC 3339 timestamp
  revalidation_triggers[1..32]:
    release_change | target_change | dependency_change | exploit_evidence
    | control_failure | incident | external_guidance_change
  accepted_risk_content_hash: SHA-256

MemorySafetyStrategyV1
  strategy_id: StableId
  strategy_version: positive u32
  memory_contract_fragment_refs[1..256]:
    sorted unique exact ArchitectureContractFragmentRefV1
  unsafe_boundary_inventory_evidence_ref: exact EvidenceRefV1
  required_control_refs[1..256]:
    sorted unique exact SecurityControlRefV1
  sanitizer_profile_refs[1..64]:
    sorted unique exact QualificationProfileRefV1
  static_analysis_profile_refs[1..64]:
    sorted unique exact QualificationProfileRefV1
  fuzz_profile_refs[1..64]:
    sorted unique exact QualificationProfileRefV1
  compiler_hardening_profile_refs[1..64]:
    sorted unique exact QualificationProfileRefV1
  strategy_content_hash: SHA-256

VulnerabilityResponsePolicyV1
  response_policy_id: StableId
  response_policy_version: positive u32
  critical_class: VulnerabilityResponseClassV1
  high_class: VulnerabilityResponseClassV1
  moderate_class: VulnerabilityResponseClassV1
  low_class: VulnerabilityResponseClassV1
  duplicate_requires_canonical_case: true
  unknown_impact_closure: prohibited
  stale_inventory_unaffected_inference: prohibited
  response_policy_content_hash: SHA-256

VulnerabilityResponseClassV1
  product_severity: critical | high | moderate | low
  acknowledgement_within_hours: positive u32
  initial_triage_within_hours: positive u32
  reassessment_interval_hours: positive u32
  fixed_release_required: bool
  disclosure_required: bool
  notification_required: bool
  incident_declaration:
    required | evidence_based | not_required
  monitoring_evidence_class_refs[1..64]:
    sorted unique exact EvidenceClassRefV1
  required_incident_closure_kinds[1..5]:
    sorted unique
      containment | recovery | notification | monitoring
      | recurrence_prevention
  class_content_hash: SHA-256

SecurityEscalationPolicyV1
  escalation_policy_id: StableId
  escalation_policy_version: positive u32
  accountable_owner_document_id: ArchitectureDocumentId
  escalation_owner_document_ids[1..32]:
    sorted unique ArchitectureDocumentId
  triggers[1..32]:
    owner_unavailable | deadline_exceeded | active_exploitation
    | cross_release_impact | disclosure_coordination
    | signing_or_update_failure | evidence_revoked
  maximum_unacknowledged_hours: positive u32
  escalation_policy_content_hash: SHA-256

NotificationAudienceV1
  audience_id: StableId
  audience_version: positive u32
  kind:
    affected_user | affected_developer | distributor | platform_owner
    | dependency_provider | reporter_or_coordinator
  release_scope:
    affected_releases | fixed_releases | both
  locale_profile_refs[1..64]:
    sorted unique exact LocaleProfileRefV1
  delivery_channel_refs[1..64]:
    sorted unique exact ProductSupportChannelRefV1
  audience_content_hash: SHA-256

ProductSecurityBaselineV1
  baseline_id: StableId
  baseline_version: positive u32
  product_definition_ref: exact ActiveProductDefinitionRefV1
  target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1
  control_refs[1..4096]:
    sorted unique exact SecurityControlRefV1
  accepted_risk_refs[0..256]:
    sorted unique exact AcceptedSecurityRiskRefV1
  memory_safety_strategy_ref: exact MemorySafetyStrategyRefV1
  response_policy_ref: exact VulnerabilityResponsePolicyRefV1
  baseline_content_hash: SHA-256

ProductSecurityReleaseBindingV1
  security_release_binding_id: StableId
  security_release_binding_version: positive u32
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  security_baseline_ref: exact ProductSecurityBaselineRefV1
  threat_ownership_registry_ref: exact ThreatOwnershipRegistryRefV1
  support_window_ref: exact ProductSupportWindowRefV1
  case_registry_snapshot_ref: exact SecurityCaseRegistrySnapshotRefV1
  release_security_assessment_evidence_refs[1..4096]:
    sorted unique exact EvidenceRefV1
  security_release_binding_content_hash: SHA-256

SecurityCaseRegistrySnapshotV1
  case_registry_snapshot_id: StableId
  case_registry_snapshot_version: positive u32
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  case_head_refs[0..65535]:
    sorted unique exact VulnerabilityCaseRefV1
  open_critical_case_refs[0..0]:
    exact empty set
  security_update_decision_refs[0..65535]:
    sorted unique exact SecurityUpdateDecisionRefV1
  disclosure_record_refs[0..65535]:
    sorted unique exact SecurityDisclosureRecordRefV1
  embargo_refs[0..65535]:
    sorted unique exact SecurityEmbargoRefV1
  incident_refs[0..65535]:
    sorted unique exact ProductSecurityIncidentRefV1
  required_notification_receipt_refs[0..65535]:
    sorted unique exact NotificationReceiptRefV1
  snapshot_evidence_refs[1..4096]:
    sorted unique exact EvidenceRefV1
  case_registry_snapshot_content_hash: SHA-256
```

Baseline controlは少なくとも次のcategoryをclosed IDでcoverageする。

- threat modelingとsecurity requirement
- public input／file／network／package validation
- memory／handle／lifetime safety
- compiler hardening、sanitizer、static analysis、fuzz
- dependency provenance、SBOM、vulnerability monitoring
- credential、signing material、secret isolation
- AI authority、sandbox、approval
- package signing、update、rollback、revocation
- logging、support bundle、redaction、incident evidence
- vulnerability response、disclosure、notification、exercise

C++23採用を無条件に安全または不適格とは扱わない。Memory／Pointers Ownerのownership／handle contract、sanitizer、fuzz、static analysis、compiler hardening、Dependency隔離、unsafe boundary inventory、incident learningを`MemorySafetyStrategyV1`へ束縛する。

Accepted riskはrisk ID、exact subject refs、accountable Owner、rationale Evidence、compensating control refs、expiration、revalidation triggerを必須にする。permanent exception、Ownerなし、期限なし、自由文だけのacceptance、Target全体へのwildcardを禁止する。

Response Policyは四つのnamed classを持ち、各Fieldの`product_severity`はField名とbyte equalityにする。別severity、重複severity、欠落classを拒否する。各classはIncident closureで必要なcontainment、recovery、notification、monitoring、recurrence-prevention集合をclosed setとして決める。

`ProductSecurityReleaseBindingV1`はProduct Lifecycleの完成Release refを入力にSecurity baseline、Threat Registry、Support Window、case registry snapshot、fresh assessment Evidenceを一方向に束縛する。Engine release bindingからSecurity recordを逆参照せず、Product LifecycleとProduct Securityの規範依存／content hash cycleを作らない。

Releaseの`product_definition_ref`はBaselineとThreat Registryの同Fieldにbyte equality、ReleaseのTarget集合はBaselineのTarget集合とset equalityにする。Required security subject集合は、Releaseの全Target package、public contract member、dependency lock entry、AI authority surface、Architecture Inventoryのapplicable public domainをclosed projection ruleで導出し、Registry Bindingのsubject集合がそのexact superset、各subjectのrequired control集合がBaseline control集合のsubsetでなければならない。Projection input、rule version、結果set hashは`case_registry_snapshot_ref`のEvidenceへ束縛し、名前またはcurrent Inventoryから再推測しない。Support Windowは同じReleaseが束縛するexact refとbyte equalityにする。

`SecurityCaseRegistrySnapshotV1`はReleaseへ影響し得る全Caseのcurrent headをexactly one件ずつ含み、各Caseのdownstream decision／disclosure／embargo／incident headとrequired notification receiptをtyped setで閉じる。`open_critical_case_refs`はSchema上emptyであり、critical caseがunresolvedならSnapshotを作れない。Snapshotは完成Release refを下流から参照し、ReleaseまたはCaseへSnapshot refを埋め戻さない。

## 5. Vulnerability intakeとtriage

```text
VulnerabilityCaseV1
  case_id: StableId
  case_revision: positive u32
  previous_case_ref: null | exact VulnerabilityCaseRefV1
  state:
    received | triaged | validating | confirmed | remediation_candidate
    | release_pending | disclosure_pending | monitoring | closed | rejected
  intake_evidence_refs[1..256]:
    sorted unique exact EvidenceRefV1
  security_subject_refs[1..256]:
    sorted unique exact SecuritySubjectRefV1
  threat_ownership_registry_ref: exact ThreatOwnershipRegistryRefV1
  threat_ownership_binding_refs[1..256]:
    sorted unique exact ThreatOwnershipBindingRefV1
  affected_release_refs[0..256]:
    sorted unique exact EngineReleaseBindingRefV1
  unaffected_release_evidence_refs[0..256]:
    sorted unique exact EvidenceRefV1
  validation_evidence_refs[0..4096]:
    sorted unique exact EvidenceRefV1
  severity_assessment:
    null
    | {
        product_severity: critical | high | moderate | low,
        response_policy_ref: exact VulnerabilityResponsePolicyRefV1,
        response_class_content_hash: SHA-256,
        severity_evidence_refs[1..4096]:
          sorted unique exact EvidenceRefV1
      }
  duplicate_of_case_ref: null | exact VulnerabilityCaseRefV1
  remediation_candidate_ref: null | exact PreparedCandidateRefV1
  security_update_decision_ref: null | exact SecurityUpdateDecisionRefV1
  disclosure_record_ref: null | exact SecurityDisclosureRecordRefV1
  incident_ref: null | exact ProductSecurityIncidentRefV1
  embargo_ref: null | exact SecurityEmbargoRefV1
  closure_evidence_refs[0..4096]:
    sorted unique exact EvidenceRefV1
  case_content_hash: SHA-256
```

Intakeはuntrusted report bytesをimmutable Evidenceとして隔離し、parseしたclaimをauthority-free projectionにする。添付、URL、PoC、commandを自動実行せず、credentialまたはpersonal dataをcase summaryへ転記しない。

Triageはsubject候補、Owner、potential impact、duplicate候補、coordination needを決めるが、confirmed、affected release、severity、fixを確定しない。各`threat_ownership_binding_ref`はCaseのexact RegistryにinlineされたBindingとbyte equalityで、subject集合をexactにcoverageする。Release Snapshotへ含めるCaseのRegistry refはRelease Security BindingのRegistry refとbyte equalityにする。Validationはdomain Ownerがreproduction、source-to-sink、reachability、affected artifact、negative controlをEvidence化する。

外部severity scoreまたはCVSSをEvidence refとして保持できるが、Miraikanaiのrelease decision、notification audience、response deadlineを自動決定しない。Product severityはresponse policyがimpact、exploitability、exposure、affected population、active exploitation Evidenceを入力に判定する。`received | triaged | validating`では`severity_assessment=null`、`confirmed`以降ではnon-nullを必須にする。`response_class_content_hash`は選択Policyのnamed severity classのcanonical bytesとbyte equalityであり、case revisionそのものがseverity判断revisionになる。別Policy、外部score、古いclass、Evidenceなしのseverityをsealしない。

Duplicateは`duplicate_of_case_ref`のexact解決とsubject／root-cause Evidence一致を必須にする。名前、CVE、scanner rule、stack trace類似だけでduplicate closeしない。Canonical caseがclosed、missing、別product、別root causeならduplicateとして閉じない。`duplicate_of_case_ref`はselfを禁止し、参照先のcanonical rootまでの全edgeをstrict predecessor DAGにし、chain cycle、mutual duplicate、duplicateから別duplicateへのunbounded delegationを拒否する。

CaseからSecurity Update、Disclosure、Embargo、IncidentへのRefは、各下流recordが参照する`case_anchor_ref`より後のCase revisionだけへ追加できる。下流recordの`case_anchor_ref`はその作成時点のimmutable Case ancestorであり、同一または後続Case revisionを参照してはならない。このstrict-ancestor ruleを全local reverse edgeへ適用し、Case↔downstream recordのcontent-hash cycleを拒否する。

## 6. Remediationとsecurity update

```text
SecurityUpdateDecisionV1
  decision_id: StableId
  decision_revision: positive u32
  previous_decision_ref: null | exact SecurityUpdateDecisionRefV1
  case_anchor_ref: exact VulnerabilityCaseRefV1
  affected_release_refs[1..256]:
    sorted unique exact EngineReleaseBindingRefV1
  notification_audience_refs[1..256]:
    sorted unique exact NotificationAudienceRefV1
  decision:
    {
      kind: prepare_update,
      remediation_candidate_ref: exact PreparedCandidateRefV1,
      update_channel_refs[1..64]:
        sorted unique exact ProductUpdateChannelRefV1,
      qualification_evidence_refs[1..4096]:
        sorted unique exact EvidenceRefV1
    }
    | {
        kind: release_update,
        fixed_release_refs[1..256]:
          sorted unique exact EngineReleaseBindingRefV1,
        target_package_bindings[1..256]:
          sorted unique {
            target_profile_ref: exact TargetProfileRefV1,
            package_artifact_ref: exact ArtifactRefV1,
            signature_receipt_ref: exact OperationReceiptRefV1
          },
        update_channel_refs[1..64]:
          sorted unique exact ProductUpdateChannelRefV1,
        publication_recovery_policy_ref:
          exact ProductPublicationRecoveryPolicyRefV1,
        disclosure_disposition:
          {kind=embargoed, embargo_ref: exact SecurityEmbargoRefV1}
          | {
              kind=disclosure_ready,
              disclosure_record_ref: exact SecurityDisclosureRecordRefV1
            },
        notification_plan_evidence_refs[1..256]:
          sorted unique exact EvidenceRefV1
      }
    | {
        kind: withdraw_update,
        withdrawn_fixed_release_refs[1..256]:
          sorted unique exact EngineReleaseBindingRefV1,
        distribution_stop_receipt_refs[1..256]:
          sorted unique exact OperationReceiptRefV1,
        notification_receipt_refs[1..4096]:
          sorted unique exact NotificationReceiptRefV1,
        recovery:
          {
            kind: rollback,
            publication_recovery_policy_ref:
              exact ProductPublicationRecoveryPolicyRefV1
          }
          | {
              kind: superseding_release,
              superseding_release_ref: exact EngineReleaseBindingRefV1
            }
      }
    | {
        kind: no_update,
        unaffected_or_unsupported_evidence_refs[1..4096]:
          sorted unique exact EvidenceRefV1,
        mitigation_evidence_refs[1..4096]:
          sorted unique exact EvidenceRefV1
      }
  rationale_evidence_refs[1..4096]:
    sorted unique exact EvidenceRefV1
  decision_content_hash: SHA-256
```

Remediation Candidateは通常Product Candidateと同じ隔離、Build、test、package、signing、qualificationを通り、case refとaffected release closureを追加で束縛する。Fix source、binary patch、package、advisoryを別Candidateから合成しない。

`decision`はclosed tagged unionであり、未選択branchのFieldを禁止する。`prepare_update`は公開またはinstall可能を意味しない。`release_update`はfixed release、Release Target集合とexact set equalityのTarget別Package、各PackageのPlatform-owned signature Receipt、publication recovery、notification plan、disclosure／embargo dispositionが同じcase anchorへ閉じる場合だけ許す。`withdraw_update`は配布停止Receipt、affected audienceへのNotification Receipt、rollbackまたはsuperseding releaseを必須にする。`no_update`はunaffected／unsupported Evidenceとmitigationを明示し、単なるresource不足を理由にしない。

`case_anchor_ref`はDecision作成前のCase revisionである。後続Case revisionだけがそのDecision Refを取り込める。`previous_decision_ref`は同じdecision IDのstrict predecessorだけを許し、self、branch、merge、cycleを拒否する。

Product updateのmigration、last-known-good、repair semanticsは[Product Lifecycle](../00-product/product-lifecycle.md)と[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)へ従う。Security urgencyを理由にpartial Project migration、signature bypass、別Target artifact、old test Receiptへfallbackしない。

## 7. Disclosureとnotification

```text
SecurityEmbargoV1
  embargo_id: StableId
  embargo_revision: positive u32
  previous_embargo_ref: null | exact SecurityEmbargoRefV1
  case_anchor_ref: exact VulnerabilityCaseRefV1
  starts_at: RFC 3339 timestamp
  review_at: RFC 3339 timestamp
  maximum_end_at: RFC 3339 timestamp
  coordinator_identity_evidence_refs[1..64]:
    sorted unique exact EvidenceRefV1
  reporter_contact_evidence_refs[1..64]:
    sorted unique exact EvidenceRefV1
  participant_identity_evidence_refs[0..64]:
    sorted unique exact EvidenceRefV1
  release_condition:
    {
      kind: fixed_release_published,
      fixed_release_ref: exact EngineReleaseBindingRefV1
    }
    | {kind=coordinated_date, coordinated_release_at: RFC 3339 timestamp}
    | {
        kind: active_exploitation,
        exploitation_evidence_ref: exact EvidenceRefV1
      }
    | {
        kind: public_disclosure,
        public_disclosure_evidence_ref: exact EvidenceRefV1
      }
    | {kind=maximum_end_reached}
  exception_evidence_refs[0..64]:
    sorted unique exact EvidenceRefV1
  embargo_content_hash: SHA-256

SecurityDisclosureRecordV1
  disclosure_id: StableId
  disclosure_revision: positive u32
  previous_disclosure_ref: null | exact SecurityDisclosureRecordRefV1
  case_anchor_ref: exact VulnerabilityCaseRefV1
  affected_release_refs[1..256]:
    sorted unique exact EngineReleaseBindingRefV1
  fixed_release_refs[0..256]:
    sorted unique exact EngineReleaseBindingRefV1
  embargo_ref: null | exact SecurityEmbargoRefV1
  publication:
    {
      state: withheld,
      reason_evidence_refs[1..256]:
        sorted unique exact EvidenceRefV1
    }
    | {
        state: scheduled,
        scheduled_at: RFC 3339 timestamp,
        publication_plan_evidence_refs[1..256]:
          sorted unique exact EvidenceRefV1
      }
    | {
        state: published,
        public_advisory_artifact_ref: exact ArtifactRefV1,
        public_location_artifact_ref: exact ArtifactRefV1,
        publish_receipt_ref: exact OperationReceiptRefV1
      }
    | {
        state: corrected,
        superseded_disclosure_ref: exact SecurityDisclosureRecordRefV1,
        reason_evidence_refs[1..256]:
          sorted unique exact EvidenceRefV1,
        public_advisory_artifact_ref: exact ArtifactRefV1,
        public_location_artifact_ref: exact ArtifactRefV1,
        publish_receipt_ref: exact OperationReceiptRefV1
      }
    | {
        state: withdrawn,
        superseded_disclosure_ref: exact SecurityDisclosureRecordRefV1,
        reason_evidence_refs[1..256]:
          sorted unique exact EvidenceRefV1,
        withdrawal_receipt_ref: exact OperationReceiptRefV1
      }
  notification_receipt_refs[0..4096]:
    sorted unique exact NotificationReceiptRefV1
  redaction_evidence_ref: exact EvidenceRefV1
  disclosure_content_hash: SHA-256
```

Advisoryはaffected／fixed release、impact、mitigation、update path、credit／coordination、known limitationをresponse policyに従って含む。PoC detail、secret、personal data、unfixed exploitability detailはembargo／redaction policyへ従う。

`publication`はclosed tagged unionであり、未選択stateのFieldを禁止する。`published`はpublic advisory artifactのimmutable bytes、public location artifact、publish Receipt、redaction Evidence、required notification audience集合が揃う場合だけ許す。Website editor表示、draft URL、送信queue投入をpublishedにしない。Correctionとwithdrawalはstrict ancestorの`superseded_disclosure_ref`、reason Evidence、公開またはwithdrawal Receiptを持ち、旧advisoryを削除しない。

Embargoは開始、対象case anchor、redacted coordinator／reporter contact Evidence、解除条件、coordinated date、review time、maximum duration、exception policyをclosed recordで表す。`coordinated_date` branchだけが`coordinated_release_at`を持ち、他branchで同Fieldを禁止する。日付超過、関係者不明、解除条件不明のembargoを暗黙延長しない。

EmbargoとDisclosureの`case_anchor_ref`は作成前のCase revisionであり、後続Case revisionだけがそれぞれのRefを取り込める。`previous_*_ref`と`superseded_disclosure_ref`は同じrecord familyのstrict ancestorだけを許し、self、branch、merge、cycleを拒否する。

## 8. Incident

```text
ProductSecurityIncidentV1
  incident_id: StableId
  incident_revision: positive u32
  previous_incident_ref: null | exact ProductSecurityIncidentRefV1
  state: declared | containing | recovering | monitoring | closed
  related_case_anchor_refs[0..256]:
    sorted unique exact VulnerabilityCaseRefV1
  affected_release_refs[0..256]:
    sorted unique exact EngineReleaseBindingRefV1
  containment_evidence_refs[0..4096]:
    sorted unique exact EvidenceRefV1
  recovery_evidence_refs[0..4096]:
    sorted unique exact EvidenceRefV1
  notification_receipt_refs[0..4096]:
    sorted unique exact NotificationReceiptRefV1
  recurrence_prevention_refs[0..256]:
    sorted unique exact SecurityControlChangeRefV1
  monitoring_evidence_refs[0..4096]:
    sorted unique exact EvidenceRefV1
  exercise_or_incident: exercise | real_incident
  incident_content_hash: SHA-256

SecurityControlChangeV1
  control_change_id: StableId
  control_change_version: positive u32
  before_baseline_ref: exact ProductSecurityBaselineRefV1
  after_baseline_ref: exact ProductSecurityBaselineRefV1
  candidate_ref: exact PreparedCandidateRefV1
  source_case_anchor_refs[0..256]:
    sorted unique exact VulnerabilityCaseRefV1
  source_incident_anchor_refs[0..256]:
    sorted unique exact ProductSecurityIncidentRefV1
  qualification_evidence_refs[1..4096]:
    sorted unique exact EvidenceRefV1
  control_change_content_hash: SHA-256
```

Incident declarationはcase validation前でも可能である。`state=declared`ではaffected releaseとcontainment Evidenceを0件にできるが、unknownを既知またはunaffectedと表現してはならない。`containing`以降はaffected releaseを1件以上、`recovering`以降はcontainment Evidenceを1件以上必須にする。`closed`では関連Caseのselected response classが要求するcontainment、recovery、notification、monitoring、recurrence-prevention kindをexactに満たし、required kindの配列をnon-emptyにする。複数Caseが異なるclassを持つ場合はrequired kindのcanonical unionを使用する。

`state=closed`では`related_case_anchor_refs[]`を1件以上必須にし、各Case anchorは`severity_assessment`がnon-nullの`confirmed`以降のrevisionでなければならない。CaseなしIncidentは`declared | containing`でEvidenceを保全できるが、閉じる前に少なくとも一つのcanonical Vulnerability Caseへ束縛する。Caseを作れない事象は本Product Security Incident familyではcloseせず、別Ownerのnon-vulnerability event contractへ分類する。空集合に対してresponse class unionを推測しない。

`related_case_anchor_refs[]`と`source_*_anchor_refs[]`はrecord作成前のstrict ancestorだけを指す。後続Case／Incident revisionだけがDecision、Disclosure、Control Change refを取り込める。Incidentの`recurrence_prevention_refs[]`とControl Changeの`source_incident_anchor_refs[]`に同一または後続revisionを許さず、Incident↔Control Changeのhash cycleを拒否する。`previous_incident_ref`は同じincident IDのstrict predecessorだけを許す。

Exerciseとreal incidentは同じclosure contractを検証するが相互に代用しない。Exercise artifactをCustomer notificationまたはreal affected release Evidenceとして使用しない。Real incidentのsecret／personal dataはEvidence Ownerのredactionとretentionへ従う。

## 9. Case transition

```text
received
  -> triaged
  -> validating
  -> confirmed
  -> remediation_candidate
  -> release_pending
  -> disclosure_pending
  -> monitoring
  -> closed
```

`rejected`はinvalid、out-of-scope、non-security、unreproducible-with-bounded-evidenceのclosed reasonを必要とする。`duplicate`はstateではなくcanonical case relationである。Confirmed caseをEvidence削除によりrejectedへ戻さず、新revisionでcorrectionを記録する。

次ではcaseを`closed`にできない。

- canonical duplicate caseがexactに解決しない
- validation未完了またはimpact不明
- affected／unaffected releaseのEvidence不足
- embargo未解決
- required fixが未release
- required audienceへのnotification未完了
- disclosure artifact、publication Receipt、redaction Evidenceが不整合
- monitoringまたはrecurrence prevention条件が未完了

Case transition Operationはexpected case revision、before hash、after candidate hash、Authorization、Evidence setを持ち、concurrent updateはconflictにする。Free-form issue status、mail label、chat reactionをcase stateにしない。

## 10. failureと禁止fallback

- stale SBOM、別release inventory、package名一致からunaffectedを推測しない。
- report textからcommand、path、severity、affected release、disclosure dateを自動決定しない。
- unknown impact、未検証duplicate、未release fix、未通知audienceでcloseしない。
- disclosure／update／notification失敗時はcaseとEvidenceを保持し、success Receiptを発行しない。
- credential、secret、private key、personal dataをadvisory、notification、support attachmentへ転記しない。
- security urgencyを理由にAI approval、code review、signing、Target qualificationを迂回しない。
- Windows ReceiptをAndroid／Apple、Editor ReceiptをGame package、exerciseをreal incidentへ流用しない。

## 11. Qualification

最低限、次をpositive／negative fixtureで検証する。

1. valid、invalid、malicious、duplicate report
2. multi-release impactと明示的unaffected Evidence
3. dependency vulnerabilityとSBOM freshness
4. embargo、coordinated disclosure、zero-day
5. fix Candidate、security update、withdraw／rollback
6. required audience notificationとdelivery failure
7. disclosure publish、correction、withdrawal、redaction failure
8. incident exerciseとreal incident
9. stale inventory、unknown impact、missing Owner、orphan subject
10. concurrent case update、revoked Evidence、cross-candidate artifact

Qualification ReceiptのEnvelope、署名、retention、revocationはAI Verification／Provenance、technical finding validationはdomain Owner、Package／SigningはPlatform Ownerが所有する。Product Securityはsame case、same Candidate、same release、required set、freshnessだけを集約する。

## 12. 外部根拠と適用限界

- [NIST SSDF publications](https://csrc.nist.gov/Projects/ssdf/publications): SSDF 1.1 Finalをsecure development baselineの比較入力にする。1.2は2026-07-29時点でDraftであり、確定規範にしない。
- [NIST SP 800-216](https://csrc.nist.gov/pubs/sp/800/216/final): vulnerability report intake、assessment、management、remediation communicationの比較入力にする。米国連邦機関向けprocessをMiraikanaiへそのまま複写しない。
- [CISA Product Security Bad Practices](https://www.cisa.gov/sites/default/files/2025-01/joint-guidance-product-security-bad-practices-508c_0.pdf): Product全体のsecurity ownership、memory-safety roadmap、known exploited issueへの責任を設ける根拠にする。Voluntary／nonbinding guidanceであり、C++を一律禁止する判断にしない。

External score、framework control ID、advisory formatはEvidenceまたはmappingであり、MiraikanaiのOwner、case state、release decisionを所有しない。外部文書更新だけでcurrent baselineを暗黙変更せず、baseline revisionとQualificationを更新する。

## 13. 明示的非目標

- Product SecurityをAI authorization、SBOM generator、Platform signer、domain validatorにすること
- CVSS、scanner、CVE、issue trackerをProduct severityまたはcase authorityにすること
- C++の一律禁止または安全性の無条件主張
- 未検証報告から自動修正、build、publish、notificationを行うこと
- Plugin ecosystem、汎用Script VM／JIT、Multiplayerをsecurityを理由にMVPへ追加すること
- vulnerability caseを一般support ticketまたはBug trackerの自由statusへ縮退すること

## 14. 完了条件

- Architecture Inventoryの全security subjectを一意のaccountable Ownerへ束縛できる。
- Baseline、accepted risk、memory-safety strategy、response policyがProduct definitionとTargetへexactに閉じる。
- 全local content hashが同じcanonical constructionを使い、CaseとDecision／Embargo／Disclosure／Incident／Control Changeがstrict-ancestor DAGを作る。
- Release Bindingがtyped Case Registry Snapshot、空のopen-critical set、required decision／notification closureを同じEngine Releaseへ束縛する。
- Intake、triage、validation、remediation Candidate、release、disclosure、notification、incident、closureが別state／recordで追跡できる。
- affected／fixed／unaffected releaseをfresh Evidenceなしに推測しない。
- AI、Toolchain、Platform、domain、Evidence、Support Bundleの既存Owner境界を複写しない。
- Registry、Schema、Operation、Fixture、Receiptが未materialize／未ActivationであることをArchitecture InventoryとClosure Reviewが保持する。
