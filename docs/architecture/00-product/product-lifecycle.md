# Miraikanai Engine Product Lifecycle Contract

- 文書ID: mirakan.arch.product-lifecycle
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: 第三者Developer向けProject bootstrap、Template／Sample／Documentationのrelease binding、GUI／CLI／headless parity、Engine／Project update composition、repair／support window、NOTICE presentation、Product lifecycle E2E acceptance
- 非正本範囲: Product intent／MVP／release Gate、ProjectRevision／ChangeSet、Build／Cook／Package／Signingのdomain semantics、dependency version／SBOM source、migration class、SupportBundle schema、Evidence envelope。各Ownerを参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Product Plan](product-plan.md)、[Project State](../03-authoring/project-state.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)
- 関連文書: [Product Security](../01-governance/product-security.md)、[Core Architecture](../02-foundation/core-architecture.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Editor Workspace／UX](../03-authoring/editor-workspace-ux.md)、[Native Game Module](../03-authoring/native-game-module.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Debugging／Observability／Replay](../04-runtime/debugging-observability-replay.md)、[Windows](../07-platform/windows.md)、[Android](../07-platform/android.md)、[Apple](../07-platform/apple.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec）
- 外部根拠確認日: 2026-07-29

## 1. 結論と所有境界

Product Lifecycleは、第三者DeveloperがEngine releaseを取得し、Projectを作成し、学習し、編集し、build／cook／package／launchし、更新し、問題を診断してsupportを受けるまでの製品横断compositionとacceptanceを所有する。個別domainのSchemaまたは処理を再定義せず、各Ownerのexact artifact／Receiptを同じrelease、Project revision、Candidate、Targetへ束縛する。

Product intent、MVP、Product release／stop／completion Gateは[Product Plan](product-plan.md)、`ProjectRevision`とatomic commitは[Project State](../03-authoring/project-state.md)、Operation authorization／Audit bindingは[Executable Contracts](../02-foundation/executable-contracts.md)と[AI Security／Approval](../01-governance/ai-security-approval.md)、Build／Cook／Package／SigningはCore、Asset、Runtime Package、各Platform、dependency lock／license／SBOM／third-party notice sourceは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、migration class、rollback eligibility、data／public-contract復元意味は[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、`SupportBundleV1`は[Debugging／Observability／Replay](../04-runtime/debugging-observability-replay.md)、Evidence envelope／signature／retention／revocationは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)だけが所有する。本書の`ProductPublicationRecoveryPolicyV1`はCompatibility判断後のCandidate publication、配布済みrelease、不可逆な外部actionの製品横断orchestrationだけを所有し、migration／rollback可能性または復元後data semanticsを決定しない。

本書はProduct lifecycle契約のtarget designである。対応するSchema、Operation、Template、Sample、Documentation bundle、Fixture、ReceiptはRepositoryに存在せず、すべて未materialize／未Activationである。

## 2. 共通規則

- 全objectはclosedであり、未知Field、重複Field、NaN／Inf、範囲外値を拒否する。
- 全Refは参照Ownerが定義するID、positive version／revision、content hashを持つexact Refである。ID-only、表示名、path、`latest`、近いversion、同名別Owner、別Targetへfallbackしない。
- `sorted unique`配列は各要素のcanonical identity byte順にstrict sortし、duplicateを拒否する。
- content hashは自己hash Fieldだけを除くclosed canonical bytesを、型ごとのASCII domain separatorとlength framingしてSHA-256する。
- Editor GUI、CLI、headlessは同じtyped request、Operation、authorization、validation、candidate hash、Receipt／diagnosticを使用する。表示、interactive prompt、progress projectionだけをsurface固有にできる。
- clean-breakは未公開かつ未materializeの内部Schemaにだけ適用する。一度公開したProject、Save、Package、ReleaseはCompatibility Ownerのversioned migrationとlast-known-good規則に従う。

本書が新設するhash型のASCII domain separatorは次である。

| 型 | ASCII domain separator |
|---|---|
| `EngineReleaseBindingV1` | `MIRAKAN_ENGINE_RELEASE_BINDING_V1` |
| `ProjectBootstrapProfileV1` | `MIRAKAN_PROJECT_BOOTSTRAP_PROFILE_V1` |
| `ProjectTemplateManifestV1` | `MIRAKAN_PROJECT_TEMPLATE_MANIFEST_V1` |
| `SampleProjectManifestV1` | `MIRAKAN_SAMPLE_PROJECT_MANIFEST_V1` |
| `DocumentationEntryV1` | `MIRAKAN_DOCUMENTATION_ENTRY_V1` |
| `DocumentationLinkV1` | `MIRAKAN_DOCUMENTATION_LINK_V1` |
| `DocumentationSnippetFixtureV1` | `MIRAKAN_DOCUMENTATION_SNIPPET_FIXTURE_V1` |
| `ProductSurfaceParityReceiptV1` | `MIRAKAN_PRODUCT_SURFACE_PARITY_RECEIPT_V1` |
| `DocumentationQualificationReceiptV1` | `MIRAKAN_DOCUMENTATION_QUALIFICATION_RECEIPT_V1` |
| `DocumentationBundleManifestV1` | `MIRAKAN_DOCUMENTATION_BUNDLE_MANIFEST_V1` |
| `ProductUpdatePlanV1` | `MIRAKAN_PRODUCT_UPDATE_PLAN_V1` |
| `ProductUpdateChannelV1` | `MIRAKAN_PRODUCT_UPDATE_CHANNEL_V1` |
| `ProductPublicationRecoveryPolicyV1` | `MIRAKAN_PRODUCT_PUBLICATION_RECOVERY_POLICY_V1` |
| `ProductNoticePresentationV1` | `MIRAKAN_PRODUCT_NOTICE_PRESENTATION_V1` |
| `ProductNoticePresentationEntryV1` | `MIRAKAN_PRODUCT_NOTICE_PRESENTATION_ENTRY_V1` |
| `ProductLifecycleAcceptanceV1` | `MIRAKAN_PRODUCT_LIFECYCLE_ACCEPTANCE_V1` |
| `ProductSupportWindowV1` | `MIRAKAN_PRODUCT_SUPPORT_WINDOW_V1` |
| `ProductSupportWindowClosureV1` | `MIRAKAN_PRODUCT_SUPPORT_WINDOW_CLOSURE_V1` |
| `ProductSupportChannelV1` | `MIRAKAN_PRODUCT_SUPPORT_CHANNEL_V1` |
| `ProductBootstrapPolicyV1` | `MIRAKAN_PRODUCT_BOOTSTRAP_POLICY_V1` |

本書Ownerのexact Refは次のclosed tupleである。

| Ref | Field |
|---|---|
| `EngineReleaseBindingRefV1` | `{engine_release_id, engine_release_version, release_binding_content_hash}` |
| `ProjectBootstrapProfileRefV1` | `{bootstrap_profile_id, bootstrap_profile_version, bootstrap_profile_content_hash}` |
| `ProjectTemplateManifestRefV1` | `{template_id, template_version, template_content_hash}` |
| `SampleProjectManifestRefV1` | `{sample_id, sample_version, sample_content_hash}` |
| `DocumentationEntryRefV1` | `{documentation_entry_id, documentation_entry_version, documentation_entry_content_hash}` |
| `DocumentationLinkRefV1` | `{documentation_link_id, documentation_link_version, documentation_link_content_hash}` |
| `DocumentationSnippetFixtureRefV1` | `{snippet_fixture_id, snippet_fixture_version, snippet_fixture_content_hash}` |
| `ProductSurfaceParityReceiptRefV1` | `{parity_receipt_id, parity_receipt_version, parity_receipt_content_hash}` |
| `DocumentationQualificationReceiptRefV1` | `{documentation_receipt_id, documentation_receipt_version, documentation_receipt_content_hash}` |
| `DocumentationBundleManifestRefV1` | `{documentation_bundle_id, documentation_bundle_version, bundle_content_hash}` |
| `ProductUpdatePlanRefV1` | `{update_plan_id, update_plan_version, update_plan_content_hash}` |
| `ProductUpdateChannelRefV1` | `{update_channel_id, update_channel_version, update_channel_content_hash}` |
| `ProductPublicationRecoveryPolicyRefV1` | `{recovery_policy_id, recovery_policy_version, recovery_policy_content_hash}` |
| `ProductNoticePresentationRefV1` | `{presentation_id, presentation_version, presentation_content_hash}` |
| `ProductNoticePresentationEntryRefV1` | `{presentation_entry_id, presentation_entry_version, presentation_entry_content_hash}` |
| `ProductSupportWindowRefV1` | `{support_window_id, support_window_version, support_window_content_hash}` |
| `ProductSupportWindowClosureRefV1` | `{support_window_closure_id, support_window_closure_version, support_window_closure_content_hash}` |
| `ProductSupportChannelRefV1` | `{channel_id, channel_version, channel_content_hash}` |
| `ProductBootstrapPolicyRefV1` | `{bootstrap_policy_id, bootstrap_policy_version, bootstrap_policy_content_hash}` |
| `ProductLifecycleAcceptanceRefV1` | `{acceptance_id, acceptance_version, acceptance_content_hash}` |

各RefのFieldは解決先recordとbyte equalityにする。別recordのhash、ID／versionだけ、隣接するrelease、同じpackage pathから補完しない。

## 3. Engine release binding

```text
EngineReleaseBindingV1
  engine_release_id: StableId
  engine_release_version: positive u32
  product_definition_ref: exact ActiveProductDefinitionRefV1
  toolchain_closure_ref: exact BuildToolchainClosureRefV1
  supported_target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1
  public_contract_set_ref: exact PublicContractSetRefV1
  documentation_bundle_ref: exact DocumentationBundleManifestRefV1
  support_window_ref: exact ProductSupportWindowRefV1
  release_binding_content_hash: SHA-256

EngineReleaseBindingRefV1
  engine_release_id: StableId
  engine_release_version: positive u32
  release_binding_content_hash: SHA-256
```

Release bindingはProduct definition、Toolchain、Target集合、public contract、Documentation、support windowを一つのimmutable identityへ閉じる。Release label、Git tag、package version、Store listingのいずれも単独ではrelease bindingにならない。[Product Security](../01-governance/product-security.md)は完成Release refを`ProductSecurityReleaseBindingV1`からSecurity baseline／Threat Registryへ一方向に束縛し、相互hash参照または規範依存cycleを作らない。

`documentation_bundle_ref`と`support_window_ref`が循環しないよう、両者はEngine releaseを含まないreceipt-free base recordである。Engine release bindingがbase refを束縛してrelease固有identityを完成し、DocumentationまたはSupport WindowからRelease bindingを逆参照しない。

## 4. Project bootstrap

```text
ProjectBootstrapProfileV1
  bootstrap_profile_id: StableId
  bootstrap_profile_version: positive u32
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  project_template_ref: exact ProjectTemplateManifestRefV1
  requested_target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1
  foundation_definition_closure_ref: exact FoundationDefinitionClosureRefV1
  bootstrap_policy_ref: exact ProductBootstrapPolicyRefV1
  bootstrap_profile_content_hash: SHA-256

ProductBootstrapPolicyV1
  bootstrap_policy_id: StableId
  bootstrap_policy_version: positive u32
  destination_precondition: must_not_exist
  publication_policy: single_atomic_project_revision
  staging_authority: non_authoritative
  collision_policy: reject
  unknown_template_entry_policy: reject
  maximum_initial_document_count: positive u32
  maximum_source_entry_count: positive u32
  maximum_total_source_bytes: positive u64
  bootstrap_policy_content_hash: SHA-256

ProjectTemplateManifestV1
  template_id: StableId
  template_version: positive u32
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  source_tree_artifact_ref: exact ArtifactRefV1
  initial_document_refs[1..4096]:
    sorted unique exact ProjectDocumentRefV1
  required_capability_refs[0..256]:
    sorted unique exact CapabilityDefinitionRefV1
  supported_target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1
  third_party_notice_source_artifact_refs[1..256]:
    sorted unique exact ArtifactRefV1
  template_content_hash: SHA-256
```

requested Target集合はreleaseとTemplateのsupported Target集合のintersectionのsubsetでなければならない。TemplateのSource tree、initial Document集合、Capability集合、notice集合は同じcontent hashへ閉じる。empty default Project、別release Template、同名Template、近いTargetへ暗黙置換しない。

bootstrap requestは新規Project ID、destinationのProject-scoped root、exact profile ref、Userが明示したProject display metadataだけを入力にする。Authoring Command Gatewayが全Document、Source tree、manifest、Target selectionを一つのprepared candidateとして検証し、成功時だけ最初の`ProjectRevision`をatomic publishする。失敗時はcurrent `ProjectRevision`、openable partial Project、権威を持つdestination manifestを残さない。診断用stagingはProject State Ownerの非authority candidateとしてのみ保持できる。

## 5. Documentation bundle

```text
DocumentationEntryV1
  documentation_entry_id: StableId
  documentation_entry_version: positive u32
  kind:
    getting_started | tutorial | public_api_reference | concept
    | migration | troubleshooting | known_limitation | legal_notice
  locale_profile_ref: exact LocaleProfileRefV1
  audience: beginner | advanced_user | project_cpp_author | operator
  source_artifact_ref: exact ArtifactRefV1
  rendered_artifact_ref: exact ArtifactRefV1
  public_contract_member_refs[0..4096]:
    sorted unique exact PublicContractMemberRefV1
  documentation_entry_content_hash: SHA-256

DocumentationLinkV1
  documentation_link_id: StableId
  documentation_link_version: positive u32
  source_entry_key:
    {
      documentation_entry_id: StableId,
      documentation_entry_version: positive u32,
      expected_entry_content_hash: SHA-256
    }
  destination:
    {
      kind: internal_entry,
      entry_key:
        {documentation_entry_id: StableId, documentation_entry_version: positive u32},
      expected_entry_content_hash: SHA-256
    }
    | {
        kind: public_contract_member,
        public_contract_member_ref: exact PublicContractMemberRefV1
      }
    | {
        kind: approved_external_url,
        normalized_url_artifact_ref: exact ArtifactRefV1
      }
  documentation_link_content_hash: SHA-256

DocumentationSnippetFixtureV1
  snippet_fixture_id: StableId
  snippet_fixture_version: positive u32
  documentation_entry_ref: exact DocumentationEntryRefV1
  public_contract_member_refs[1..256]:
    sorted unique exact PublicContractMemberRefV1
  target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1
  fixture_kind: compile | validate | execute | package
  input_artifact_ref: exact ArtifactRefV1
  expected_result:
    {kind=success, output_contract_ref: exact QualificationOutputContractRefV1}
    | {kind=diagnostic, diagnostic_contract_ref: exact DiagnosticContractRefV1}
  snippet_fixture_content_hash: SHA-256

SampleProjectManifestV1
  sample_id: StableId
  sample_version: positive u32
  source_project_artifact_ref: exact ArtifactRefV1
  expected_project_revision_ref: exact ProjectRevisionRefV1
  supported_target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1
  qualification_scenario_refs[1..256]:
    sorted unique exact QualificationScenarioRefV1
  documentation_entry_refs[1..256]:
    sorted unique exact DocumentationEntryRefV1
  third_party_notice_source_artifact_refs[1..256]:
    sorted unique exact ArtifactRefV1
  sample_content_hash: SHA-256

DocumentationBundleManifestV1
  documentation_bundle_id: StableId
  documentation_bundle_version: positive u32
  public_contract_set_ref: exact PublicContractSetRefV1
  entry_refs[1..65535]:
    sorted unique exact DocumentationEntryRefV1
  link_refs[0..262144]:
    sorted unique exact DocumentationLinkRefV1
  snippet_fixture_refs[1..4096]:
    sorted unique exact DocumentationSnippetFixtureRefV1
  tutorial_scenario_refs[1..4096]:
    sorted unique exact QualificationScenarioRefV1
  sample_project_refs[0..256]:
    sorted unique exact SampleProjectManifestRefV1
  link_graph_content_hash: SHA-256
  bundle_content_hash: SHA-256

```

Documentation graphはEntryからLinkを逆参照しない。Bundleがexact Entry ref集合とexact Link ref集合を一方向に束縛する。`internal_entry`の`entry_key`はRefではなくBundle内node locatorであり、同じBundleの`entry_refs[]`に同じID／versionかつ`expected_entry_content_hash`がbyte equalityのnodeがexactly one件必要である。source keyも同様にexactly one件へ解決し、self edgeは許してもhash参照を作らない。`link_graph_content_hash`は`MIRAKAN_DOCUMENTATION_LINK_GRAPH_V1`、canonical Entry ref集合、canonical Link ref集合、各Linkが解決したsource／destination node keyのclosed bytesから計算し、Entry hashまたはLink hashからBundle hashを逆参照しない。このDAGによりEntry↔LinkまたはEntry A↔Entry Bのcontent-hash cycleを作らず、ID／versionだけのstale destinationも受理しない。

External URLのredirect先、HTTP status、resolved content hashはQualification時のEvidenceであり、URL文字列だけを有効性証明にしない。`normalized_url_artifact_ref`は承認済みnormalized URL bytesをexactに指し、live WebsiteをDocumentation authorityにしない。

Tutorial scenarioは開始Project ref、typed action列、expected Project revision／artifactを持ち、自由shellまたはUI座標を正規stepにしない。Documentation Bundleのentry、snippet、tutorial、Sample集合はpublic contract setのreachable memberをcoverageし、orphan ref、duplicate、別contract setを拒否する。

broken internal／external link、public signatureと異なるsnippet、unrunnable sample、別release向けtutorial、missing locale fallback declarationをProduct Release Gateで失敗させる。Documentation bundleはrelease artifactであり、Websiteのcurrent pageまたはChat回答を正本にしない。

## 6. GUI／CLI／headless parity

```text
Editor GUI ─┐
CLI        ─┼─> same typed request
Headless   ─┘
              -> same registered semantic Operation
              -> same authorization and validation
              -> same Authoring Command Gateway
              -> atomic ProjectRevision or no current mutation
              -> same typed Receipt／diagnostic
```

Surface adapterは入力をtyped requestへ投影し、Operation resultを各presentationへ投影するだけである。次を禁止する。

- GUIだけのhidden defaultまたはsilent repair
- CLIだけのfilesystem mutation
- headlessだけのpartial success
- surface別のauthorization、validation、idempotency、hash規則
- Widget label、command text、exit codeをsemantic Operation identityにすること

```text
ProductSurfaceParityReceiptV1
  parity_receipt_id: StableId
  parity_receipt_version: positive u32
  parity_fixture_id: StableId
  surface_kind: editor_gui | cli | headless
  request_content_hash: SHA-256
  operation_authorization_binding_ref:
    exact OperationAuthorizationAuditBindingRefV1
  prepared_candidate_ref: exact PreparedCandidateRefV1
  operation_receipt_ref: exact OperationReceiptRefV1
  resulting_project_revision_ref: null | exact ProjectRevisionRefV1
  diagnostic_set_content_hash: SHA-256
  evidence_refs[1..256]:
    sorted unique exact EvidenceRefV1
  parity_receipt_content_hash: SHA-256

DocumentationQualificationReceiptV1
  documentation_receipt_id: StableId
  documentation_receipt_version: positive u32
  documentation_bundle_ref: exact DocumentationBundleManifestRefV1
  candidate_ref: exact PreparedCandidateRefV1
  target_profile_ref: exact TargetProfileRefV1
  qualification_scenario_ref: exact QualificationScenarioRefV1
  observed_public_contract_set_ref: exact PublicContractSetRefV1
  result: passed
  evidence_refs[1..4096]:
    sorted unique exact EvidenceRefV1
  documentation_receipt_content_hash: SHA-256
```

同じ`parity_fixture_id`には三surfaceがexactly one件ずつ必要で、request hash、authorization、Candidate、Operationのsemantic result、resulting Project ref、diagnostic setが規定されたset equalityを満たす。presentation bytes equalityは要求しない。Documentation ReceiptはBundleが宣言するrequired Target／scenario closureとexact set equalityでなければAcceptanceへ入れず、失敗、取消済みEvidence、別Candidate、別public contract setを合成しない。

## 7. Product update

```text
ProductUpdateChannelV1
  update_channel_id: StableId
  update_channel_version: positive u32
  kind: offline_installer | managed_layout | platform_store
  target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1
  platform_owner_document_id: ArchitectureDocumentId
  distribution_contract_artifact_ref: exact ArtifactRefV1
  update_channel_content_hash: SHA-256

ProductPublicationRecoveryPolicyV1
  recovery_policy_id: StableId
  recovery_policy_version: positive u32
  last_known_good_required: true
  project_publication: single_atomic_promotion
  prepublication_failure: discard_candidate
  postpublication_failure:
    explicit_release_rollback | forward_repair_required
  user_data_policy: preserve_and_version_validate
  irreversible_external_action_policy:
    retain_evidence_and_require_compensating_decision
  recovery_policy_content_hash: SHA-256

ProductUpdatePlanV1
  update_plan_id: StableId
  update_plan_version: positive u32
  source_engine_release_binding_ref: exact EngineReleaseBindingRefV1
  destination_engine_release_binding_ref: exact EngineReleaseBindingRefV1
  source_project_revision_ref: exact ProjectRevisionRefV1
  candidate_ref: exact PreparedCandidateRefV1
  compatibility_assessment_ref: exact CompatibilityAssessmentRefV1
  migration_plan_ref: exact MigrationPlanRefV1
  update_channel_refs[1..64]:
    sorted unique exact ProductUpdateChannelRefV1
  publication_recovery_policy_ref: exact ProductPublicationRecoveryPolicyRefV1
  target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1
  required_qualification_scenario_refs[1..4096]:
    sorted unique exact QualificationScenarioRefV1
  support_window_ref: exact ProductSupportWindowRefV1
  update_plan_content_hash: SHA-256
```

Updateはlive Projectまたはinstalled releaseをin-place mutationしない。source release／Projectから独立Candidateを作り、次のexact closureを得た後だけCompatibility Ownerがmigration／rollback eligibilityを認め、Product Lifecycleが`ProductPublicationRecoveryPolicyV1`に従う一回のpublicationを行う。Compatibility Ownerはdata／public-contract復元後の意味とlast-known-good eligibility、本書はpublication境界、不可逆外部action、失敗後の製品横断状態だけを所有し、互いのSchemaを複写しない。

1. source inventoryとconsumer inventory
2. compatibility assessmentとversioned migration
3. Project validation
4. Target別cook／build／package
5. install／launch／offline play
6. Save／Load／Replay、必要なrecook／rebuild
7. diagnosis／Support Bundle
8. SBOM／NOTICE／license presentation
9. same-candidate qualificationとrelease acceptance

旧release／旧Projectをlast-known-goodとして維持する。途中成功をcurrentにせず、別Candidate、別Target、別revisionのReceiptを合成しない。不可逆なStore提出、通知または外部公開の失敗をlocal rollback済みと表現しない。

`repair`はcurrent release bindingに対するmanifest／artifact verificationと同一bytesの復元だけを意味する。別version、latest、別channelへのupdateをrepairとして実行しない。

## 8. Support windowとNOTICE presentation

```text
ProductSupportWindowV1
  support_window_id: StableId
  support_window_version: positive u32
  supported_target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1
  support_channel_refs[1..64]:
    sorted unique exact ProductSupportChannelRefV1
  support_start_policy: at_release_publication
  support_end:
    {kind=explicit_date, date: RFC 3339 full-date}
    | {kind=ongoing_until_superseding_decision}
  security_update_support: required
  known_limitation_document_ref: exact DocumentationEntryRefV1
  end_notification:
    minimum_notice_days: positive u32
    channel_refs[1..64]:
      sorted unique exact ProductSupportChannelRefV1
  support_window_content_hash: SHA-256

ProductSupportChannelV1
  channel_id: StableId
  channel_version: positive u32
  kind: offline_documented_process | web_form | email
  endpoint_artifact_ref: exact ArtifactRefV1
  locale_profile_refs[1..64]:
    sorted unique exact LocaleProfileRefV1
  authentication: none | user_account_required
  accepted_attachment_kind:
    none | redacted_support_bundle
  channel_content_hash: SHA-256

ProductNoticePresentationV1
  presentation_id: StableId
  presentation_version: positive u32
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  package_artifact_ref: exact ArtifactRefV1
  sbom_ref: exact SbomRefV1
  notice_bundle_ref: exact ThirdPartyNoticeBundleRefV1
  presentation_entry_refs[1..4096]:
    sorted unique exact ProductNoticePresentationEntryRefV1
  locale_profile_refs[1..64]:
    sorted unique exact LocaleProfileRefV1
  presentation_content_hash: SHA-256

ProductNoticePresentationEntryV1
  presentation_entry_id: StableId
  presentation_entry_version: positive u32
  notice_subject_ref:
    {kind=dependency, dependency_lock_entry_ref: exact DependencyLockEntryRefV1}
    | {
        kind=redistributed_artifact,
        redistributed_artifact_ref: exact RedistributedArtifactRefV1
      }
  license_ref: exact LicenseArtifactRefV1
  notice_text_artifact_ref: exact ArtifactRefV1
  locale_profile_ref: exact LocaleProfileRefV1
  presentation_location:
    editor_about | installed_game_legal | distribution_bundle_document
  package_entry_ref: null | exact PackageEntryRefV1
  presentation_entry_content_hash: SHA-256

ProductSupportWindowClosureV1
  support_window_closure_id: StableId
  support_window_closure_version: positive u32
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  closed_support_window_ref: exact ProductSupportWindowRefV1
  expected_previous_closure_ref: null | exact ProductSupportWindowClosureRefV1
  closure_basis:
    {
      kind: scheduled_explicit_date,
      release_decision_evidence_ref: exact EvidenceRefV1
    }
    | {
        kind: superseding_product_decision,
        signed_product_decision_evidence_ref: exact EvidenceRefV1
      }
  effective_end_at: RFC 3339 timestamp
  notification_receipt_refs[1..4096]:
    sorted unique exact NotificationReceiptRefV1
  notified_channel_refs[1..64]:
    sorted unique exact ProductSupportChannelRefV1
  support_window_closure_content_hash: SHA-256
```

Toolchain Ownerが生成したSBOM／notice sourceとPackage Ownerのartifact entry集合をexact set equalityで照合する。Product Lifecycleはlicense判断またはSBOM生成を所有せず、UserがEditor、installed Gameまたは配布bundleのdocumented locationからnoticeへ到達できることを判定する。別buildのSBOM、空NOTICE、欠落locale、到達不能UIを成功にしない。

Engine release bindingがSupport Window base refを束縛することでrelease、Target、support channel、security update policy、end dateまたは明示的ongoing、known limitation disclosureを閉じる。`end_notification.channel_refs[]`は`support_channel_refs[]`のnon-empty subsetであり、ID、version、hashをbyte equalityにする。

Support終了時は別immutable `ProductSupportWindowClosureV1`を作り、同じEngine Releaseが束縛するexact Support Window、expected previous Closure、closure basis、effective time、全required channelのNotification Receiptを閉じる。`explicit_date` windowは`scheduled_explicit_date`だけを許し、`effective_end_at`をbase dateのend-of-day UTCへ正規化して一致させる。`ongoing_until_superseding_decision`は無期限保証ではなく、`superseding_product_decision`と署名済みProduct decision Evidenceを必須にする。`notified_channel_refs[]`はbase windowの`end_notification.channel_refs[]`とset equalityでなければならない。最初のClosureは`expected_previous_closure_ref=null`かつversion 1、訂正は同じclosure IDの直前versionを参照し、self、branch、merge、cycleを拒否する。base recordまたはEngine ReleaseへClosure refを埋め戻さず、一方向DAGを維持する。`latest releaseだけsupport`、Website text、issue tracker labelからsupport状態を推測しない。`SupportBundleV1`のField、redaction、consent、failureはDebugging Ownerだけが所有する。

## 9. Product lifecycle acceptance

```text
ProductLifecycleAcceptanceV1
  acceptance_id: StableId
  acceptance_version: positive u32
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  candidate_ref: exact PreparedCandidateRefV1
  target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1
  bootstrap_receipt_refs[1..64]:
    sorted unique exact OperationReceiptRefV1
  parity_receipt_refs[3..192]:
    sorted unique exact ProductSurfaceParityReceiptRefV1
  documentation_receipt_refs[1..4096]:
    sorted unique exact DocumentationQualificationReceiptRefV1
  build_cook_package_receipt_refs[1..4096]:
    sorted unique exact OperationReceiptRefV1
  install_launch_offline_receipt_refs[1..4096]:
    sorted unique exact QualificationReceiptRefV1
  update_rollback_repair_receipt_refs[1..4096]:
    sorted unique exact QualificationReceiptRefV1
  support_receipt_refs[1..4096]:
    sorted unique exact QualificationReceiptRefV1
  notice_presentation_ref: exact ProductNoticePresentationRefV1
  acceptance_content_hash: SHA-256
```

AcceptanceはReceipt発行元のdomain meaningを変更せず、same release、same candidate、same Project revision、required Target集合、freshness、non-revocation、required scenario set equalityを検証する。Parityは各required fixtureにGUI／CLI／headlessがexactly one件、DocumentationはBundleが要求するTarget／scenarioがexactly one件、NOTICE EntryはPackage／SBOM／notice sourceとexact set equalityでなければならない。Product PlanのRelease Gateはこのacceptanceを入力にできるが、本書自身がRelease ApprovalまたはCapability Activationを発行しない。

## 10. failureと禁止fallback

- release、public contract、Target、hash、license、qualificationのexact不一致はmutation前に拒否する。
- `latest`、同名、近いversion、別Target、別Sample、既定Templateへ置換しない。
- bootstrap failureではcurrent Projectまたはopenable partial Projectを作らない。
- update failureでは旧release／旧Projectを維持し、新Project revision成功を旧Package成功で代用しない。
- Documentationのbroken link／snippet、stale tutorial、unrunnable Sampleをwarningだけでreleaseしない。
- SBOM／NOTICE mismatch、User到達不能、support window欠落をrelease acceptance成功にしない。
- surface adapter failureを別surfaceの成功で埋めず、GUI／CLI／headlessのどれか一つをsemantic authorityにしない。

## 11. Qualification

最低限、同一release candidateについて次を検証する。

1. clean environmentでProject createから最初のatomic `ProjectRevision`まで完走する。
2. Editor GUI／CLI／headlessが同じrequest meaning、candidate hash、result、typed diagnosticを返す。
3. TemplateとSampleからcook、package、clean install／launch、offline playまで完走する。
4. public API snippet、tutorial、link graph、Sampleがsame public contract setに一致する。
5. clean install、supported update、failed update、rollback、repairを独立fixtureで検証する。
6. diagnosis、`SupportBundleV1` export、support window、data resetを検証する。
7. SBOM、NOTICE、license presentationがexact Packageと一致しUserから到達できる。
8. wrong release／Target／hash、stale docs、partial Project、cross-candidate Receiptをnegative fixtureで拒否する。

Windows Preview iterationは[Native Game Module](../03-authoring/native-game-module.md)のrestart-based contractを使用する。Shipping static link、Preview DLLのprocess起動時一回load、変更時`GameHost` restart、in-process unload／binary replacement／live code patch禁止を維持し、Hot Reloadという別Capabilityを追加しない。

## 12. CMake／Platform projection

[CMake Presets 4.4](https://cmake.org/cmake/help/v4.4/manual/cmake-presets.7.html)はproject／user presetとconfigure／build／test／package／workflow presetを定義する。MiraikanaiはCMake Presetをtyped Build requestのprojectionとして使用できるが、Preset、CMake cacheまたはIDE profileをProject／Build authorityにしない。

Windows package identity、version、differential update、signature、device trustは[Microsoft MSIX package updates](https://learn.microsoft.com/en-us/windows/msix/app-package-updates)と[MSIX signing overview](https://learn.microsoft.com/en-us/windows/msix/package/signing-package-overview)をPlatform OwnerがTarget別に比較する。Product Lifecycleは証明書、Store policy、package schemaを所有せず、それらのfresh Receiptをsame release acceptanceへ束縛する。

Android、Apple、headless distributionも同じProduct lifecycle meaningを使うが、Windows Receiptを流用しない。各Platform Ownerがpackage、signing、install、update、rollback、device qualificationを所有する。

## 13. 明示的非目標

- Product Lifecycleを新しいBuild、Package、Migration、Evidence authorityにすること
- Plugin marketplaceまたは任意native Plugin ecosystem
- 汎用Game Script VM、JIT、Game向けFFI
- Multiplayer、Account、Cloud、広告、課金
- Runtime code generation
- large open worldまたはhigh-end renderingをMVP lifecycle前提にすること
- `build/` root、latest lookup、silent repair、部分migration、互換aliasを再導入すること

## 14. 完了条件

- Engine release、Template、Sample、Documentation、update、support、NOTICEがexact Refで一方向に閉じる。
- 全local RefがID、version／revision、content hashを持ち、Documentation Entry／Link／Bundle、Support Window／Closure、Release／Acceptanceのhash graphがDAGである。
- GUI／CLI／headlessが同じOperationへ収束し、surface固有authorityを持たない。
- bootstrapはatomic、updateはseparate Candidate＋single promotion、failureはlast-known-goodを維持する。
- Product Release GateがDocumentation、Sample、support、NOTICEをBuild成功と別に検証できる。
- Product Lifecycleが各domain Schemaを複写せず、各Ownerのfresh Receiptだけを集約する。
- Schema、Operation、Fixture、Receiptが未materialize／未ActivationであることをArchitecture InventoryとClosure Reviewが保持する。
