# Miraikanai Engine Required Feature Coverage Closure Design

- 状態: approved
- 判断日: 2026-07-29
- 対象: C++ Game Engine必須機能監査から得たProduct Lifecycle、Product Security、再利用Scene composition、Architecture文書間整合性の設計閉包
- 非対象: Engine／Schema／Editor／CI／Testの実装、実装Task、実装順序、担当、工数、日程、Plugin ecosystem、汎用Script VM／JIT、Multiplayer、既存形式向け互換fallback
- 根拠入力: [ChatGPT 5.6 Pro required-features source](../../reviews/2026-07-29-chatgpt-5-6-pro-required-features-source.md)、[feature coverage audit](../../reviews/2026-07-29-chatgpt-5-6-pro-feature-coverage-audit.md)

## 1. 結論

既存Architectureは、監査対象35分野の大半をOwner文書、明示的非目標、またはより狭く安全な意味置換として保持している。新しい汎用Subsystemを横並びに追加する必要はない。完全閉包には次の四変更だけを採用する。

1. `docs/architecture/00-product/product-lifecycle.md`を新設し、第三者DeveloperがProjectを作成してから更新、診断、support、製品releaseを利用するまでのProduct横断compositionとacceptanceを一意に所有させる。
2. `docs/architecture/01-governance/product-security.md`を新設し、Product全体の脅威責任、脆弱性報告、triage、修正release、開示、security incidentを一意に所有させる。
3. `docs/architecture/06-rendering/world.md`の既存`Scene Source`を、再利用Scene instance、nested composition、typed override、explicit rebaseを持つcleanなPrefab意味置換として閉じる。`Prefab`を正規ID、Schema名またはlegacy aliasにはしない。
4. Mobile Application lifecycleとWindows workspace rootの二つの文書矛盾を修正する。

Windows Previewの高速iterationは既存[Native Game Module](../../architecture/03-authoring/native-game-module.md)が、Shipping static link、Preview DLLのprocess起動時一回load、変更時`GameHost` restart、in-process unload／binary replacement／live patch禁止としてすでに閉じている。Hot Reload不足として新しい仕組みを追加しない。

`ArchitectureInventoryV1`、Explain Projection、Schema、Validator、Fixture、Operation、Receipt等がRepositoryに未materializeである事実は、Architecture Closure `ARCH-C03`のblockerとして残す。設計説明の完成を実装、ActivationまたはProduct利用可能性へ読み替えない。

## 2. 判断原則

### 2.1 完全閉包

完全閉包は、機能名を列挙することではなく、次を一意に説明できる状態である。

- canonical Ownerと非Owner
- typed input／outputとexact identity
- authorityとmutation境界
- state transitionとlast-known-good
- failure時の禁止fallback
- Target別Qualificationとnegative fixture
- Evidenceのproducer、consumer、freshness
- current design、materialization、qualification、activationの区別

外部Engineで一般的な名称があっても、MiraikanaiのOwner境界、AI authority、安全性、Product scopeへ合わなければ追加しない。

### 2.2 clean-break

未公開で未materializeの内部Schemaは、正しいV1を直接定義する。旧称alias、二重read、近似version lookup、暗黙migration、silent repairを追加しない。一方、将来一度公開したProject revision、Save、Package、Releaseは既存[Compatibility／Evolution](../../architecture/02-foundation/compatibility-evolution.md)のversioned migration、last-known-good、rollback規則へ従う。clean-breakをUser data破棄の許可にしない。

### 2.3 scope維持

現在のProduct scopeはWindows desktop C1、後続Android／Apple、C++23＋GameplayDefinition、Generic Core＋Feature Packs＋任意Genre Packs＋Projects、compact 2D command RPG Product Referenceである。Shooterはtechnical qualification consumerでありProduct MVPではない。

次は監査入力に現れても現在の必須要件へ昇格しない。

- Plugin marketplaceまたは任意native Plugin ecosystem
- 汎用Game Script VM、JIT、Game向けFFI
- Multiplayer、Account、Cloud、広告、課金
- large open world、virtualized geometryまたはhigh-end renderingをMVP前提にすること
- Runtime code generation

## 3. Owner境界

### 3.1 Product Lifecycle Owner

`mirakan.arch.product-lifecycle`は次だけを正本化する。

- Project create／bootstrapの製品composition
- Project TemplateとSample Projectの製品公開単位
- public API documentation、tutorial、snippet、sampleのrelease binding
- Editor GUI、CLI、headlessから同じtyped Operationへ到達するparity acceptance
- Engine／Project update candidate、repair、support window、NOTICE presentation
- Product Release Gateで必要なend-to-end acceptance

次は参照するだけで複写しない。

| 関心 | canonical Owner |
|---|---|
| `ProjectRevision`、`ProjectChangeSet`、atomic commit | Project State |
| Build／Cook／Package／Signingのdomain semantics | Core、Asset、Runtime Package、各Platform |
| dependency version、license、SBOM、third-party notice source | Toolchain／Dependencies |
| migration class、compatibility window、rollback rule | Compatibility／Evolution |
| `SupportBundleV1`のField、redaction、生成failure | Debugging／Observability／Replay |
| Evidence envelope、署名、retention、revocation | AI Verification／Provenance |
| Product intent、MVP、release／stop／completion Gate | Product Plan |

Product Lifecycle Ownerはdomain Operationを再定義せず、exact refを一つのProduct candidateへ束縛してE2E acceptanceを判定する。

### 3.2 Product Security Owner

`mirakan.arch.product-security`は次を正本化する。

- Product全体のthreat ownership registry
- security baselineとmemory-safetyを含むrisk treatmentの製品責任
- vulnerability report intake、triage、validation、remediation、release、disclosure、closure
- security update decision、affected／fixed release、notification
- product security incident、containment、recovery、recurrence prevention

次は参照するだけで複写しない。

| 関心 | canonical Owner |
|---|---|
| AI task authorization、approval tier、managed host authority | AI Security／Approval |
| dependency lock、license、SBOM generation | Toolchain／Dependencies |
| signing、Store、OS sandbox、privacy integration | 各Platform Owner |
| input validation、memory／handle／resource contract | 各domain Owner |
| Evidence envelope、signature、redactionの共通規則 | AI Verification／Provenance |
| support bundleの収集／redaction schema | Debugging／Observability／Replay |

報告者の文章、scanner severity、外部score、issue labelは入力Evidenceであって権威判断ではない。Product Security Ownerはdomain Ownerの技術判断を上書きせず、case全体のstateとrelease／disclosure closureを所有する。

### 3.3 World Owner

`mirakan.arch.rendering-world`は既存Scene Sourceの再利用compositionを次まで拡張する。

- exact Scene Source revisionを参照するinstance
- stable instance IDとstable source object ID
- acyclic nested composition
- source path／name／array indexに依存しないtyped override
- source revision更新に対するexplicit rebase
- Authoring lineageのCook／Package／Save／Replay／Debugへの追跡
- Runtime artifactがAuthoring Scene Sourceへ依存しないこと

Gameplay progression、spawn rule、Quest、Stage outcomeは各consumer Ownerのままにする。`Prefab`という第二のSource identity、`PrefabV1`、legacy alias、implicit import形式は作らない。

### 3.4 既存Ownerを維持する項目

- Windows Preview iteration: Native Game Module
- Application lifecycle: Scheduling／Lifetime
- presentation surface availability: Scheduling／Lifetime＋Mobile Common
- workspace root: Naming／Project Layout
- Build projection: Core Architecture＋CMake Presets
- Project update compatibility: Compatibility／Evolution
- Package update／signing: 各Platform Owner

## 4. Product Lifecycle contract

本節のrecordは設計対象であり、実装済みSchemaまたはactive Operationではない。全objectはclosed、未知Fieldと重複Fieldを拒否する。`exact Ref`はOwner定義のID、positive version／revision、content hashをすべて持つ。表示名、path、`latest`、近いversion、別Targetから再解決しない。

### 4.1 Releaseとbootstrap

```text
EngineReleaseBindingV1
  engine_release_ref: exact EngineReleaseRefV1
  product_definition_ref: exact ActiveProductDefinitionRefV1
  toolchain_closure_ref: exact BuildToolchainClosureRefV1
  supported_target_profile_refs[1..64]: sorted unique exact TargetProfileRefV1
  public_contract_set_ref: exact PublicContractSetRefV1
  documentation_bundle_ref: exact DocumentationBundleManifestRefV1
  security_baseline_ref: exact ProductSecurityBaselineRefV1
  support_window_ref: exact ProductSupportWindowRefV1
  release_binding_content_hash: SHA-256

ProjectBootstrapProfileV1
  bootstrap_profile_id: StableId
  bootstrap_profile_version: positive u32
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  project_template_ref: exact ProjectTemplateManifestRefV1
  requested_target_profile_refs[1..64]: sorted unique exact TargetProfileRefV1
  initial_project_contract_ref: exact InitialProjectContractRefV1
  bootstrap_policy_ref: exact ProductBootstrapPolicyRefV1
  bootstrap_profile_content_hash: SHA-256
```

`supported_target_profile_refs[]`と`requested_target_profile_refs[]`はsubset関係を要求する。Template、public contract、documentation、security baseline、support windowは同じEngine release bindingへexactに閉じる。release IDだけ一致する別hash、同名Template、別Target sampleを受理しない。

### 4.2 Template、Sample、Documentation

```text
ProjectTemplateManifestV1
  template_id: StableId
  template_version: positive u32
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  source_tree_artifact_ref: exact ArtifactRefV1
  initial_document_refs[1..4096]: sorted unique exact ProjectDocumentRefV1
  required_capability_refs[0..256]: sorted unique exact CapabilityDefinitionRefV1
  supported_target_profile_refs[1..64]: sorted unique exact TargetProfileRefV1
  license_notice_refs[1..256]: sorted unique exact ProductNoticeRefV1
  template_content_hash: SHA-256

SampleProjectManifestV1
  sample_id: StableId
  sample_version: positive u32
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  source_project_artifact_ref: exact ArtifactRefV1
  expected_project_revision_ref: exact ProjectRevisionRefV1
  supported_target_profile_refs[1..64]: sorted unique exact TargetProfileRefV1
  qualification_scenario_refs[1..256]: sorted unique exact QualificationScenarioRefV1
  documentation_entry_refs[1..256]: sorted unique exact DocumentationEntryRefV1
  license_notice_refs[1..256]: sorted unique exact ProductNoticeRefV1
  sample_content_hash: SHA-256

DocumentationBundleManifestV1
  documentation_bundle_id: StableId
  documentation_bundle_version: positive u32
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  public_contract_set_ref: exact PublicContractSetRefV1
  entry_refs[1..65535]: sorted unique exact DocumentationEntryRefV1
  snippet_fixture_refs[1..4096]: sorted unique exact DocumentationSnippetFixtureRefV1
  tutorial_scenario_refs[1..4096]: sorted unique exact QualificationScenarioRefV1
  sample_project_refs[0..256]: sorted unique exact SampleProjectManifestRefV1
  link_graph_content_hash: SHA-256
  bundle_content_hash: SHA-256
```

Documentationはpublic contractと同じreleaseへ束縛し、snippetのcompile／validate／execute種別を明示する。broken link、stale signature、unrunnable sample、別release向けtutorialはrelease acceptanceを失敗させる。Documentationを補足資料としてrelease Gate外へ逃がさない。

### 4.3 Update、support、NOTICE

```text
ProductUpdatePlanV1
  update_plan_id: StableId
  source_engine_release_binding_ref: exact EngineReleaseBindingRefV1
  destination_engine_release_binding_ref: exact EngineReleaseBindingRefV1
  source_project_revision_ref: exact ProjectRevisionRefV1
  candidate_ref: exact PreparedCandidateRefV1
  compatibility_assessment_ref: exact CompatibilityAssessmentRefV1
  migration_plan_ref: exact MigrationPlanRefV1
  target_profile_refs[1..64]: sorted unique exact TargetProfileRefV1
  required_qualification_scenario_refs[1..4096]: sorted unique exact QualificationScenarioRefV1
  support_window_ref: exact ProductSupportWindowRefV1
  update_plan_content_hash: SHA-256

ProductNoticePresentationV1
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  package_artifact_ref: exact ArtifactRefV1
  sbom_ref: exact SbomRefV1
  notice_bundle_ref: exact ThirdPartyNoticeBundleRefV1
  presentation_entry_refs[1..4096]: sorted unique exact ProductNoticePresentationEntryRefV1
  locale_profile_refs[1..64]: sorted unique exact LocaleProfileRefV1
  presentation_content_hash: SHA-256
```

Updateはlive Projectを直接変更しない。source Project revisionとdestination releaseから独立Candidateを作り、Inventory、migration、validation、cook、package、launch、support、NOTICEのsame-candidate closureを得た後だけ一回のpromotionを許す。失敗時は旧release／旧Projectをlast-known-goodとして維持し、途中状態をcurrentにしない。不可逆な外部操作を「rollback済み」と偽装しない。

NOTICE presentationはToolchain Ownerが生成するSBOM／notice sourceを消費し、package artifactとのset equalityとUser到達可能性だけをProduct Lifecycleが判定する。別buildのSBOM、空NOTICE、UI上の到達不能を成功にしない。

### 4.4 共通Operation flow

Editor GUI、CLI、headlessはそれぞれ独自mutationを持たない。

```text
Editor GUI ─┐
CLI        ─┼─> same typed request
Headless   ─┘
              -> same registered semantic Operation
              -> same authorization／validation
              -> same Authoring Command Gateway
              -> atomic ProjectRevision or no current mutation
              -> same typed Receipt／diagnostic
```

表示形式、interactive prompt、progress projectionはsurfaceごとに異なってよい。request meaning、default、authorization、validation、idempotency、candidate hash、receipt meaningは同一でなければならない。GUIだけのhidden default、CLIだけのrepair、headlessだけのpartial successを禁止する。

## 5. Scene composition contract

```text
SceneCompositionInstanceV1
  instance_id: StableId
  source_scene_ref: exact SceneSourceRevisionRefV1
  parent_instance_id: null | StableId
  attachment_ref: exact WorldAttachmentRefV1
  override_set_ref: null | exact SceneOverrideSetRefV1
  accepted_rebase_change_ref: null | exact SceneRebaseChangeRefV1

SceneOverrideSetV1
  override_set_id: StableId
  override_set_version: positive u32
  source_scene_ref: exact SceneSourceRevisionRefV1
  entries[1..4096]: sorted unique SceneOverrideEntryV1
  override_set_content_hash: SHA-256

SceneOverrideEntryV1
  source_object_id: StableId
  owner_document_ref: exact ProjectDocumentRefV1
  field_contract_ref: exact PublicFieldContractRefV1
  expected_source_value_hash: SHA-256
  replacement_value: exact field-contract value

SceneRebaseChangeV1
  rebase_change_id: StableId
  instance_id: StableId
  before_source_scene_ref: exact SceneSourceRevisionRefV1
  after_source_scene_ref: exact SceneSourceRevisionRefV1
  before_override_set_ref: null | exact SceneOverrideSetRefV1
  result_override_set_ref: null | exact SceneOverrideSetRefV1
  resolutions[0..4096]: sorted unique SceneRebaseResolutionV1
  unresolved_conflicts[0..4096]: sorted unique SceneRebaseConflictV1
  rebase_change_content_hash: SHA-256
```

Nested instance graphはacyclicで、instance、source object、owner document、field contractをstable identityで解決する。同一fieldへのduplicate override、型不一致、削除済みobject、unknown field、別revision値、capacity超過を拒否する。

Source Scene更新は自動追従しない。explicit rebaseがbefore／after exact revision、override delta、conflict resolutionを記録し、未解決conflictが0件の時だけProject ChangeSetへ入る。名前、path、配列index、似たfield、同名objectからrepairしない。

Cookはinstance graphをTarget artifactへ閉じ、runtime stable identityとauthoring lineageの対応をdebug／save／replay用manifestへ保持する。ただしRuntime PackageはAuthoring Scene Source file、Editor objectまたはrebase serviceへ依存しない。

## 6. Product Security contract

### 6.1 Ownershipとbaseline

```text
ThreatOwnershipRegistryV1
  registry_id: StableId
  registry_version: positive u32
  product_definition_ref: exact ActiveProductDefinitionRefV1
  bindings[1..4096]: sorted unique ThreatOwnershipBindingV1
  registry_content_hash: SHA-256

ThreatOwnershipBindingV1
  security_subject_ref: exact SecuritySubjectRefV1
  accountable_owner_document_id: ArchitectureDocumentId
  responsible_owner_document_ids[1..32]: sorted unique ArchitectureDocumentId
  required_baseline_control_refs[1..256]: sorted unique SecurityControlRefV1
  escalation_policy_ref: exact SecurityEscalationPolicyRefV1

ProductSecurityBaselineV1
  baseline_id: StableId
  baseline_version: positive u32
  product_definition_ref: exact ActiveProductDefinitionRefV1
  target_profile_refs[1..64]: sorted unique exact TargetProfileRefV1
  control_refs[1..4096]: sorted unique exact SecurityControlRefV1
  accepted_risk_refs[0..256]: sorted unique exact AcceptedSecurityRiskRefV1
  memory_safety_strategy_ref: exact MemorySafetyStrategyRefV1
  response_policy_ref: exact VulnerabilityResponsePolicyRefV1
  baseline_content_hash: SHA-256
```

`SecuritySubjectRefV1`は`architecture_domain | product_release | target_package | dependency | public_contract | project_source_surface | ai_authority_surface`のclosed unionとする。各branchはowner-specific exact refを持ち、自由文字列またはpathをsubjectにしない。

C++23採用を無条件に安全または不適格とは扱わない。Memory／Pointers、sanitizer、fuzz、static analysis、compiler hardening、dependency isolation、unsafe boundary inventory、incident learningをexact baseline controlへ束縛し、例外はowner、期限、scope、compensating control、revalidationを持つaccepted riskだけに限定する。

### 6.2 Vulnerability lifecycle

```text
VulnerabilityCaseV1
  case_id: StableId
  case_revision: positive u32
  state:
    received | triaged | validating | confirmed | remediation_candidate
    | release_pending | disclosure_pending | monitoring | closed | rejected
  intake_evidence_refs[1..256]: sorted unique exact EvidenceRefV1
  security_subject_refs[1..256]: sorted unique exact SecuritySubjectRefV1
  threat_ownership_binding_refs[1..256]: sorted unique exact ThreatOwnershipBindingRefV1
  affected_release_refs[0..256]: sorted unique exact EngineReleaseBindingRefV1
  unaffected_release_evidence_refs[0..256]: sorted unique exact EvidenceRefV1
  validation_evidence_refs[0..4096]: sorted unique exact EvidenceRefV1
  remediation_candidate_ref: null | exact PreparedCandidateRefV1
  security_update_decision_ref: null | exact SecurityUpdateDecisionRefV1
  disclosure_record_ref: null | exact SecurityDisclosureRecordRefV1
  incident_ref: null | exact ProductSecurityIncidentRefV1
  embargo_ref: null | exact SecurityEmbargoRefV1
  closure_evidence_refs[0..4096]: sorted unique exact EvidenceRefV1
  case_content_hash: SHA-256

SecurityUpdateDecisionV1
  decision_id: StableId
  vulnerability_case_ref: exact VulnerabilityCaseRefV1
  affected_release_refs[1..256]: sorted unique exact EngineReleaseBindingRefV1
  fixed_release_refs[0..256]: sorted unique exact EngineReleaseBindingRefV1
  update_channel_refs[1..64]: sorted unique exact ProductUpdateChannelRefV1
  rollback_policy_ref: exact ProductRollbackPolicyRefV1
  notification_audience_refs[1..256]: sorted unique exact NotificationAudienceRefV1
  decision: prepare_update | release_update | withdraw_update | no_update
  rationale_evidence_refs[1..4096]: sorted unique exact EvidenceRefV1
  decision_content_hash: SHA-256

SecurityDisclosureRecordV1
  disclosure_id: StableId
  vulnerability_case_ref: exact VulnerabilityCaseRefV1
  affected_release_refs[1..256]: sorted unique exact EngineReleaseBindingRefV1
  fixed_release_refs[0..256]: sorted unique exact EngineReleaseBindingRefV1
  publication_state: withheld | scheduled | published | corrected | withdrawn
  embargo_ref: null | exact SecurityEmbargoRefV1
  public_advisory_artifact_ref: null | exact ArtifactRefV1
  notification_receipt_refs[0..4096]: sorted unique exact NotificationReceiptRefV1
  redaction_evidence_ref: exact EvidenceRefV1
  disclosure_content_hash: SHA-256
```

Case stateはreport intake、triage、validation、fix candidate、release、disclosure、closure Evidenceを分離する。`duplicate`は別case refへのexact relationであり、close理由だけではない。scanner名、CVSS、報告者severity、issue priorityを製品severityまたはaffected releaseへ自動変換しない。

次ではcaseをcloseできない。

- duplicate候補だがcanonical caseがexactに解決しない
- validation未完了またはimpact不明
- affected／unaffected releaseのEvidence不足
- embargoが未解決
- fix candidateがreleaseされていない
- required audienceへの通知が未完了
- disclosure artifactまたはredaction Evidenceが不整合
- recurrence preventionまたはmonitoring条件が未完了

古いSBOM、別release inventory、package名一致からunaffectedを推測しない。Disclosure／update／notification失敗時はcaseとEvidenceを保持し、成功Receiptを発行しない。

### 6.3 Incident

```text
ProductSecurityIncidentV1
  incident_id: StableId
  incident_revision: positive u32
  state: declared | containing | recovering | monitoring | closed
  related_vulnerability_case_refs[0..256]: sorted unique exact VulnerabilityCaseRefV1
  affected_release_refs[1..256]: sorted unique exact EngineReleaseBindingRefV1
  containment_evidence_refs[1..4096]: sorted unique exact EvidenceRefV1
  recovery_evidence_refs[0..4096]: sorted unique exact EvidenceRefV1
  notification_receipt_refs[0..4096]: sorted unique exact NotificationReceiptRefV1
  recurrence_prevention_refs[0..256]: sorted unique exact SecurityControlChangeRefV1
  exercise_or_incident: exercise | real_incident
  incident_content_hash: SHA-256
```

Exerciseとreal incidentを同じclosure contractで検証するが相互に代用しない。containment、recovery、notification、recurrence preventionの必須条件をresponse policyがcase／release／Target別に選び、欠落時は`closed`を拒否する。

## 7. failure contract

### 7.1 Product Lifecycle

- release、contract、Target、hash、license、qualificationのexact不一致はmutation前に拒否する。
- `latest`、同名、近いversion、別Target、別Sample、既定Templateへfallbackしない。
- bootstrap failureではcurrent `ProjectRevision`もEditorで開けるpartial Projectも作らない。
- update failureでは旧release／旧Projectをlast-known-goodとして維持する。new Project revision成功を旧Package成功で代用しない。
- public API undocumented、broken link／snippet、stale tutorial、unrunnable Sample、SBOM／NOTICE mismatchはProduct Release Gateを失敗させる。

### 7.2 Scene composition

cycle、exact ref／hash不一致、stable ID欠落、型不一致、削除target、rebase conflict、capacity超過をtyped diagnosticにする。partial commit、name／path／index／similar-field repair、source revisionの暗黙追従を禁止する。

### 7.3 Product Security

untrusted report textからcommand、path、severity、affected release、disclosure dateを自動決定しない。unknown impact、未検証duplicate、未release fix、未完了notification、stale inventoryでcaseをcloseしない。secret、credential、個人情報をadvisoryまたはsupport attachmentへ転記せず、redaction失敗時はpublicationを拒否する。

## 8. Qualification

### 8.1 Product Lifecycle qualification

- clean environmentでProject createから最初のatomic `ProjectRevision`まで
- Editor GUI／CLI／headlessのrequest meaning、result、diagnostic、candidate hash parity
- TemplateおよびSampleからcook、package、install／launch、offline play
- public API snippet、tutorial、link graph、Sampleのsame-release validation
- clean install、supported upgrade、failed upgrade、rollback、repair
- diagnosis、`SupportBundleV1` export、support window、data reset
- SBOM、NOTICE、license presentationのpackage exactnessとUser到達可能性

### 8.2 Scene composition qualification

- simple instance、nested instance、同一Sourceの複数instance
- typed override、source update、explicit rebase、conflict、object delete、field type change
- undo／redo、save／load、replay、cook、package、debug lineage
- crash recovery後のcandidate不変性
- Runtime PackageがAuthoring Scene Sourceなしで起動すること

### 8.3 Product Security qualification

- valid、invalid、malicious、duplicate report
- multi-release impact、dependency vulnerability、embargo、zero-day
- fix candidate、security update、withdraw／rollback、notification
- incident exerciseとreal-incident record
- disclosure correction／withdrawal、redaction failure
- stale SBOM、missing release inventory、unknown impactのnegative fixture

Qualification Receiptの形式、署名、retention、revocationはVerification Owner、domain-specific artifactは各Ownerが所有する。Product Lifecycle／Securityはsame-candidate、same-release、required set、freshnessとacceptanceだけを集約する。

## 9. 文書間整合修正

### 9.1 Mobile lifecycle

[Scheduling／Lifetime](../../architecture/04-runtime/scheduling-lifetime.md)のApplication stateは`Starting | Active | Inactive | Suspended | Terminating`、presentation stateは`absent | active | surface_unavailable`である。[Mobile Common](../../architecture/07-platform/mobile-common.md)からApplication stateの`SurfaceUnavailable`を除き、surface喪失をpresentation stateへ一意に投影する。Application lifecycleとpresentation availabilityを一つのenumに戻さない。

### 9.2 Windows root

[Naming／Project Layout](../../architecture/02-foundation/naming-project-layout.md)はlegacy `build/`を削除し、top-level closed setを`source/ config/ packages/ derived/ intermediate/ staging/ evidence/`としている。[Windows](../../architecture/07-platform/windows.md)の`Build／Cache | Project buildまたはUser cache`をこのroot契約へ合わせ、`build/`を再導入しない。

### 9.3 用語

- `Scene Source`、`SceneCompositionInstanceV1`を正規語にする。
- `Prefab`は外部Engine比較またはUser入力語をScene compositionへ解決する説明でのみ使用し、canonical ID、Schema、aliasにしない。
- `Hot Reload`はUser要求語をNative Game Moduleのrestart-based Preview iterationへ解決する説明に限定する。
- Plugin、Script VM、Multiplayerを現Product requirementへ追加しない。

## 10. Architecture正本へ反映する範囲

### 10.1 新設

- `docs/architecture/00-product/product-lifecycle.md`
- `docs/architecture/01-governance/product-security.md`
- `docs/architecture/decisions/2026-07-29-product-lifecycle-security-ownership.md`

### 10.2 更新

- `docs/architecture/README.md`
- `docs/architecture/00-product/product-plan.md`
- `docs/architecture/01-governance/architecture-governance.md`
- `docs/architecture/01-governance/ai-security-approval.md`
- `docs/architecture/02-foundation/core-architecture.md`
- `docs/architecture/02-foundation/executable-contracts.md`
- `docs/architecture/02-foundation/toolchain-dependencies.md`
- `docs/architecture/02-foundation/compatibility-evolution.md`
- `docs/architecture/03-authoring/project-state.md`
- `docs/architecture/03-authoring/editor-workspace-ux.md`
- `docs/architecture/03-authoring/native-game-module.md`
- `docs/architecture/04-runtime/debugging-observability-replay.md`
- `docs/architecture/06-rendering/world.md`
- `docs/architecture/07-platform/windows.md`
- `docs/architecture/07-platform/mobile-common.md`
- `docs/architecture/appendices/architecture-plan-closure-review.md`
- `docs/architecture/decisions/README.md`

既存Ownerへの更新はowner link、consumer／producer境界、qualification handoff、明白な矛盾修正だけに限定する。Product LifecycleまたはProduct SecurityのSchemaを複写しない。

## 11. 外部公式根拠

| 根拠 | 採用する意味 | 採用しない意味 |
|---|---|---|
| [CMake Presets 4.4](https://cmake.org/cmake/help/v4.4/manual/cmake-presets.7.html) | project／user presetとconfigure／build／test／package／workflow presetをBuild projectionに使える | CMake PresetをMiraikanaiのProject／Build authorityにする |
| [CMake C++ Modules 4.4](https://cmake.org/cmake/help/v4.4/manual/cmake-cxxmodules.7.html) | C++ module依存scanとtoolchain制約をcurrent exact-version設計比較に使う | module対応名だけでcross-target qualification済みとする |
| [Microsoft MSIX signing overview](https://learn.microsoft.com/en-us/windows/msix/package/signing-package-overview) | package identity、valid signature、device trust、bundle署名closure | 署名成功だけでProduct update／support closure済みとする |
| [Microsoft code signing options](https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/code-signing-options) | Windows配布経路別のsigning選択をPlatform Ownerが検証する | Product Lifecycleが証明書／Store policyを所有する |
| [Microsoft package updates](https://learn.microsoft.com/en-us/windows/msix/app-package-updates) | identity、version、differential updateをWindows update qualificationへ入力する | MSIX挙動をcross-platform migration意味へ一般化する |
| [NIST SSDF publications](https://csrc.nist.gov/Projects/ssdf/publications) | SSDF 1.1 Finalをsecure development baseline比較に使う | 1.2 Draftを確定規範として固定する |
| [NIST SP 800-216](https://csrc.nist.gov/pubs/sp/800/216/final) | vulnerability report intake、assessment、management、remediation communicationの比較に使う | 米国連邦機関向け手順を製品policyへそのまま複写する |
| [CISA Product Security Bad Practices](https://www.cisa.gov/sites/default/files/2025-01/joint-guidance-product-security-bad-practices-508c_0.pdf) | Product全体のsecurity ownership、memory-safety roadmap、known exploited issueへの明示責任を設ける | C++を一律禁止する、またはvoluntary guidanceを法的義務とする |

外部根拠はMiraikanai Ownerの代わりではない。Version、確認日、対象範囲をToolchain／Platform／Governanceの各Ownerで保持し、external guidanceの更新だけでcurrent Project contractを暗黙変更しない。

## 12. 完了条件

この設計のArchitecture反映は、次をすべて満たした場合だけ完了とする。

1. 新Owner二文書と所有DecisionがArchitecture Indexから到達できる。
2. Owner文書数、文書ID、規範依存、相対linkが再監査され、重複、未解決、cycleが0件である。
3. Product Lifecycle、Product Security、Scene instance／override／rebaseのOwner、型、data flow、failure、qualificationが一意である。
4. Mobile lifecycleとWindows rootの矛盾が消えている。
5. Native Game Moduleのrestart-based Preview iterationをHot Reload不足として扱っていない。
6. `ARCH-C03`をunmaterialized blockerとして維持し、設計文書を実装済みと表現していない。
7. `ARCH-C21`はProduct Security Owner新設により`closed-in-target-design`へ更新されるが、implementation／operation／qualificationは`absent`のままである。
8. Plugin、汎用Script VM／JIT、Multiplayer、large-world機能をMVPへ追加していない。
9. Required-features監査文書のHot ReloadとScene／Prefab判定を正しいOwner意味へ訂正している。
10. Architecture差分に実装Task、実装順序、担当、工数、日程、互換fallbackが含まれない。

この設計はArchitecture正本の変更境界を承認する文書であり、実装計画ではない。
