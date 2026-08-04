# Miraikanai Engine Product Legal／IP Governance Contract

- 文書ID: mirakan.arch.product-legal-ip-governance
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Product release／publicationに適用するjurisdiction・market・distribution channel・Product／AI roleのidentity、Legal Requirement categoryとapplicability binding、独立設計・第三者権利・copyright・trademark／trade dress・patent／freedom-to-operate・trade secret・contributor chain-of-title・dependency／content／AI Provider terms・Product claimの横断review subject、署名済みLegal／IP Readiness Decision、freshness／revocation／failure
- 非正本範囲: 法律解釈、法律相談、法務意見本文、司法判断、Product intent／claim requirement、Release Content Manifest、Platform／Store submission、dependency license／SBOM生成、Asset provenance／rights payload、Project source／Code Owner、Privacy／Security domain意味、Evidence共通Envelope、署名primitive、実装方式／工程／工数／担当。各authorityまたはOwnerを参照する
- 規範依存: [Architecture Governance](architecture-governance.md)、[Product Plan](../00-product/product-plan.md)、[Product Lifecycle](../00-product/product-lifecycle.md)、[AI Verification／Provenance](ai-verification-provenance.md)、[Executable Contracts](../02-foundation/executable-contracts.md)
- 関連文書: [Product Release Decision](../00-product/product-release-decision.md)、[Product Publication／Completion](../00-product/product-publication-completion.md)、[Product Privacy／Data Governance](product-privacy-data-governance.md)、[Product Security](product-security.md)、[AI Security／Approval](ai-security-approval.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Naming／Project Layout](../02-foundation/naming-project-layout.md)、[Project State](../03-authoring/project-state.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)、[Native Game Module](../03-authoring/native-game-module.md)、[Project Shader](../06-rendering/project-shader.md)、[Editor Workspace／UX](../03-authoring/editor-workspace-ux.md)、[Windows](../07-platform/windows.md)、[Android](../07-platform/android.md)、[Apple](../07-platform/apple.md)、[P0 Architecture／Legal-IP Ownership Decision](../decisions/2026-08-04-p0-architecture-and-legal-ip-ownership.md)
- 根拠区分: project-decision（法令、Regulator、Store、Platform、Providerまたはlicenseの内容を引用する箇所だけofficial-spec／official-guidance。個別法域への適用、非侵害、適法性または法的十分性はauthorized legal reviewを必要とする）
- 外部根拠確認日: 2026-08-04

## 1. 結論と所有境界

Miraikanai EngineはArchitecture、Scanner、AI、類似度、独立作成宣言またはlicense名だけから「法令遵守」「権利処理済み」「非侵害」を推論しない。本書は、対象Product、Candidate、claim、jurisdiction、market、distribution channel、Product／AI role、effective-date、法的Requirement、Domain Evidenceおよびauthorized human reviewを一つのfail-closed判断subjectへ束縛する。

本書が所有するのは法的結論を導くEngine algorithmではなく、次のGovernance contractである。

1. 何を、どの法域／市場／配布経路／役割についてreviewしたか。
2. どのRequirement categoryと公式source snapshotを適用対象にしたか。
3. どのOwnerが発行したEvidenceをreview inputにしたか。
4. 未解決issueが0件であることを誰が、いつ、どのscopeへ承認したか。
5. law／terms／scope／Evidence変更、期限切れまたはrevocation後にいつ承認を無効化するか。

法律、規制、契約条件の意味は各authorityとauthorized counselが判断する。本書のenum、Schema候補、validatorまたはDecision recordは法律相談、法的保証、court judgment、patent opinionまたはtrademark clearance reportの代替ではない。

## 2. 外部事実とMiraikanai判断

外部一次資料から確認する事実とProject判断を分離する。

- WIPOはcomputer programがcopyright保護対象になり得る一方、copyrightはidea、procedure、method of operationまたはmathematical conceptそれ自体を保護しないと説明する。
- WIPOはpatent rightがterritorialで、国／地域ごとに制度と判断が異なると説明する。
- 文化庁は生成AIと著作権に関する判例・裁判例の蓄積がない現状を明記し、risk低減のchecklist／guidanceを提供する。
- U.S. Copyright OfficeはAI outputのcopyrightabilityをhuman authorshipと個別のcreative contributionに結び付けている。
- EU AI ActはUnion marketへの提供、provider／deployer等のroleおよび一部GPAI obligationをscopeとして持つ。
- EU Cyber Resilience Actはmarketへ提供するproducts with digital elementsとvulnerability handlingを対象にする。
- U.S. FTCはobjective product claimに事前substantiationを要求する。

これらからMiraikanaiが採用するのは、特定の法律解釈ではなく、法域・role・subject・時点を固定し、専門家reviewとEvidenceなしにRelease claimを許可しないArchitectureである。

<a id="product-legal-ip-identity"></a>

## 3. Identityとcanonicalization

本書の型は[Executable Contracts](../02-foundation/executable-contracts.md)のMCD canonicalization、Ref解決、unknown Field rejection、size bound、Diagnostic規則と[AI Verification／Provenance](ai-verification-provenance.md)の署名Envelope／freshness／revocationを利用する。

| 型 | ASCII domain separator |
|---|---|
| `LegalAuthoritySourceSnapshotV1` | `MIRAKAN_LEGAL_AUTHORITY_SOURCE_SNAPSHOT_V1` |
| `LegalJurisdictionProfileV1` | `MIRAKAN_LEGAL_JURISDICTION_PROFILE_V1` |
| `ProductLegalDistributionBindingV1` | `MIRAKAN_PRODUCT_LEGAL_DISTRIBUTION_BINDING_V1` |
| `ProductLegalApplicabilityProfileV1` | `MIRAKAN_PRODUCT_LEGAL_APPLICABILITY_PROFILE_V1` |
| `ProductLegalRequirementBindingV1` | `MIRAKAN_PRODUCT_LEGAL_REQUIREMENT_BINDING_V1` |
| `IndependentDesignReviewSubjectV1` | `MIRAKAN_INDEPENDENT_DESIGN_REVIEW_SUBJECT_V1` |
| `ProductLegalIpReviewSubjectV1` | `MIRAKAN_PRODUCT_LEGAL_IP_REVIEW_SUBJECT_V1` |
| `ProductLegalIpDecisionV1` | `MIRAKAN_PRODUCT_LEGAL_IP_DECISION_V1` |
| `ProductLegalIpDecisionHeadV1` | `MIRAKAN_PRODUCT_LEGAL_IP_DECISION_HEAD_V1` |

全IDは意味を名前から推測しないStable ID、全RefはID、version、content hashのcomplete tupleである。`latest`、URLだけ、Company名、locale、currency、Store表示名、Target名またはcountry文字列をauthority、jurisdiction、marketまたはDecision identityにしない。

### 3.1 `LegalAuthoritySourceSnapshotV1`

```text
LegalAuthoritySourceSnapshotV1
  legal_source_snapshot_id: StableId
  legal_source_snapshot_version: positive u32
  authority_kind:
    legislation | regulation | regulator_guidance
    | court_or_tribunal_record | treaty
    | platform_contract | provider_terms | license_text
  issuing_authority_name: normalized UTF-8
  canonical_source_uri: normalized absolute URI
  source_identifier: normalized UTF-8
  source_version_or_revision: normalized UTF-8
  published_at: null | RFC 3339 UTC timestamp
  effective_from: null | RFC 3339 UTC timestamp
  effective_until: null | RFC 3339 UTC timestamp
  retrieved_at: RFC 3339 UTC timestamp
  captured_artifact_ref: null | exact ArtifactRefV1
  captured_content_hash: null | SHA-256
  language_tag: canonical BCP 47 language tag
  supersedes_source_snapshot_ref:
    null | exact LegalAuthoritySourceSnapshotRefV1
  legal_source_snapshot_content_hash: SHA-256
```

`captured_artifact_ref`と`captured_content_hash`はboth-nullまたはboth-non-nullで、後者はArtifactのcontent hashとbyte equalityにする。合法的に保持できるimmutable copyが存在する場合だけnon-nullにする。URL、検索snippetまたはretrieval時刻だけから本文bytesを推測しない。Source snapshotは公式資料の存在と取得時点を示すだけで、適用判断または法務承認ではない。

### 3.2 `LegalJurisdictionProfileV1`

```text
LegalJurisdictionProfileV1
  jurisdiction_profile_id: StableId
  jurisdiction_profile_version: positive u32
  jurisdiction_kind:
    country | subdivision | supranational | treaty_area
  authority_identifier_uri: normalized absolute URI
  territory_member_identifiers[1..512]:
    sorted unique normalized UTF-8
  applicable_source_snapshot_refs[1..4096]:
    sorted unique exact LegalAuthoritySourceSnapshotRefV1
  effective_from: RFC 3339 UTC timestamp
  effective_until: null | RFC 3339 UTC timestamp
  jurisdiction_profile_content_hash: SHA-256
```

`supranational`とmember country、countryとsubdivisionを相互代用しない。EU／EEA、United States、Japan等をlocale、currency、Store、IP address、Company所在地またはTarget Profileから推測しない。`territory_member_identifiers[]`の意味とauthorityはProfile sourceにより閉じ、自由なcountry aliasを許可しない。

### 3.3 `ProductLegalApplicabilityProfileV1`

```text
ProductLegalDistributionBindingV1
  jurisdiction_profile_ref: exact LegalJurisdictionProfileRefV1
  market_id: StableId
  publication_requirement_ref:
    exact McdContractRefV1(kind=requirement)
  platform_kind: windows | android | apple
  channel_kind: direct_distribution | managed_store
  distribution_scope_kind:
    ProductCompletionDistributionScopeKindV1
  distribution_subject_ref:
    exact ProductDistributionSubjectRefV1
  product_role:
    engine_provider | sdk_provider | editor_provider
    | runtime_redistributor | ai_system_provider
    | ai_system_deployer | content_provider
  commercial_mode:
    no_charge | one_time | subscription | enterprise_contract
  intended_user_class:
    professional_developer | organization_admin | end_user
  legal_distribution_binding_content_hash: SHA-256

ProductLegalApplicabilityProfileV1
  applicability_profile_id: StableId
  applicability_profile_version: positive u32
  product_definition_ref: exact ActiveProductDefinitionRefV1
  claim_scope_ref: exact ProductClaimScopeRefV1
  candidate_ref: exact PreparedCandidateRefV1
  release_content_manifest_ref:
    exact ProductReleaseContentManifestRefV1
  public_contract_set_ref: exact PublicContractSetRefV1
  engine_source_snapshot_ref: exact EngineSourceSnapshotRefV1
  distribution_bindings[1..65535]:
    sorted unique ProductLegalDistributionBindingV1
  legal_requirement_binding_refs[1..65535]:
    sorted unique exact ProductLegalRequirementBindingRefV1
  effective_evaluation_at: RFC 3339 UTC timestamp
  applicability_profile_content_hash: SHA-256
```

`distribution_bindings[]`はProduct LifecycleのRelease Content ManifestとDistribution Coverage Projectionが持つ全required publication routeへ、`{publication requirement, platform, channel, distribution scope, distribution subject}`でexact joinする。Legal OwnerはManifest、routeまたはDistribution Subjectのdomain schemaを再定義しない。同じbinaryでもmarket、channel、roleまたはcommercial modeが異なれば別bindingである。Profile作成者が失敗するmarketだけを黙って除外することを禁止する。

`legal_requirement_binding_refs[]`は、Active Product Definitionで`requirement_category=legal_ip`のProduct Requirement集合、`distribution_bindings[]`、16 categoryのchecked Cartesian expansionをexactly one rowずつcoverする。展開数が65,535を超える、checked arithmeticがoverflowする、missing／extra／duplicate rowがある、または一つのRequirement bindingで別Product Requirement／distribution binding／categoryをcollapseする場合はProfileを生成しない。
各`applicable_distribution_binding_hash`は同Profileのexactly one `ProductLegalDistributionBindingV1`へ解決し、Requirement bindingの`jurisdiction_profile_ref`とDistribution bindingの同Refをbyte equalityにする。Profile外hash、zero／multiple resolutionまたは別jurisdictionへの付替えを拒否する。

### 3.4 Legal Requirement category

`ProductLegalRequirementCategoryV1`は次のclosed setである。

```text
copyright
trademark_trade_dress
patent_freedom_to_operate
trade_secret_confidential_information
contributor_chain_of_title
dependency_open_source_license
asset_content_rights
ai_provider_model_output_terms
product_license_eula_redistribution
advertising_product_claim
privacy_data_protection
cybersecurity_product_obligation
accessibility_nondiscrimination
platform_store_distribution_terms
export_control_sanctions
consumer_commerce_tax
```

V1にないcategoryを`other`、free text、nearest categoryまたは`not_applicable`へ押し込まない。新しいmaterialなcategoryはSchema versionとconsumer migration判断を伴うsuccessor contractで追加する。

```text
ProductLegalRequirementBindingV1
  legal_requirement_binding_id: StableId
  legal_requirement_binding_version: positive u32
  product_requirement_ref:
    exact McdContractRefV1(kind=requirement)
  category: ProductLegalRequirementCategoryV1
  jurisdiction_profile_ref: exact LegalJurisdictionProfileRefV1
  legal_source_snapshot_refs[1..4096]:
    sorted unique exact LegalAuthoritySourceSnapshotRefV1
  applicable_distribution_binding_hash: SHA-256
  reviewed_subject_refs[1..65535]:
    sorted unique exact ArtifactRefV1
  domain_evidence_class_refs[1..4096]:
    sorted unique exact EvidenceClassRefV1
  applicability:
    {kind=required}
    | {
        kind=not_applicable,
        authorized_analysis_artifact_ref: exact ArtifactRefV1,
        reconsideration_condition_refs[1..256]:
          sorted unique exact ArtifactRefV1
      }
  legal_requirement_binding_content_hash: SHA-256
```

`not_applicable`はreview省略ではない。authorized analysis、対象scope、source snapshot、reconsideration conditionを必須にし、blank、`unknown`、`probably not applicable`、他市場DecisionまたはAI回答を受理しない。

### 3.5 `IndependentDesignReviewSubjectV1`

```text
IndependentDesignReviewSubjectV1
  independent_design_subject_id: StableId
  independent_design_subject_version: positive u32
  product_definition_ref: exact ActiveProductDefinitionRefV1
  engine_source_snapshot_ref: exact EngineSourceSnapshotRefV1
  public_contract_set_ref: exact PublicContractSetRefV1
  editor_presentation_artifact_refs[0..4096]:
    sorted unique exact ArtifactRefV1
  sample_template_documentation_refs[0..65535]:
    sorted unique exact ArtifactRefV1
  internal_requirement_refs[1..65535]:
    sorted unique exact McdContractRefV1(kind=requirement)
  architecture_decision_refs[1..4096]:
    sorted unique exact ArtifactRefV1
  permitted_external_fact_source_refs[0..4096]:
    sorted unique exact LegalAuthoritySourceSnapshotRefV1
  prohibited_material_declaration_ref: exact ArtifactRefV1
  third_party_identifier_review_ref: exact ArtifactRefV1
  api_format_ui_expression_review_ref: exact ArtifactRefV1
  reviewer_independence_declaration_ref: exact ArtifactRefV1
  unresolved_similarity_finding_refs[0..4096]:
    sorted unique exact ArtifactRefV1
  independent_design_subject_content_hash: SHA-256
```

このSubjectは「似ていない」ことを自動証明しない。公開一次資料から得た外部factと、MiraikanaiのUser requirement／Decision／contract導出を分離し、prohibited materialを利用していないというreview scopeを固定する。`unresolved_similarity_finding_refs[]`がnon-emptyのSubjectはLegal／IP approvalへ進めない。

### 3.6 Review subjectとDecision

```text
ProductLegalIpReviewSubjectV1
  legal_review_subject_id: StableId
  legal_review_subject_version: positive u32
  applicability_profile_ref:
    exact ProductLegalApplicabilityProfileRefV1
  independent_design_subject_ref:
    exact IndependentDesignReviewSubjectRefV1
  required_requirement_binding_refs[1..65535]:
    sorted unique exact ProductLegalRequirementBindingRefV1
  supplied_domain_evidence_bindings[1..65535]:
    sorted unique {
      requirement_binding_ref:
        exact ProductLegalRequirementBindingRefV1,
      evidence_class_ref: exact EvidenceClassRefV1,
      evidence_ref: exact EvidenceRefV1
    }
  privileged_review_artifact_refs[0..4096]:
    sorted unique exact ArtifactRefV1
  public_disclosure_artifact_refs[0..4096]:
    sorted unique exact ArtifactRefV1
  unresolved_issue_refs[0..4096]:
    sorted unique exact ArtifactRefV1
  legal_review_subject_content_hash: SHA-256

ProductLegalIpDecisionV1
  legal_ip_decision_id: StableId
  legal_ip_decision_version: positive u32
  legal_review_subject_ref: exact ProductLegalIpReviewSubjectRefV1
  decision_sequence: positive u64
  previous_decision_ref: null | exact ProductLegalIpDecisionRefV1
  outcome: approved | rejected
  authorized_human_reviewer_refs[1..16]:
    sorted unique exact ArtifactRefV1
  decision_reason_artifact_ref: exact ArtifactRefV1
  issued_at: RFC 3339 UTC timestamp
  valid_until: RFC 3339 UTC timestamp
  signed_record_ref: exact MirakanSignedRecordRefV1(
    purpose=product_legal_ip_decision)
  legal_ip_decision_content_hash: SHA-256

ProductLegalIpDecisionHeadV1
  legal_ip_decision_head_id: StableId
  legal_ip_decision_head_version: positive u32
  applicability_profile_ref:
    exact ProductLegalApplicabilityProfileRefV1
  current_decision_ref: exact ProductLegalIpDecisionRefV1
  expected_previous_head_ref:
    null | exact ProductLegalIpDecisionHeadRefV1
  head_sequence: positive u64
  published_at: RFC 3339 UTC timestamp
  signed_record_ref: exact MirakanSignedRecordRefV1(
    purpose=product_legal_ip_decision_head)
  legal_ip_decision_head_content_hash: SHA-256
```

`approved`は各required bindingの`domain_evidence_class_refs[]`を展開した`{requirement binding,evidence class}`集合とsupplied bindingの同projectionをset equalityにし、各rowのEvidenceについてbindingの`product_requirement_ref`、category、distribution scope、semantic admissibility、freshness、non-revocationをread-backする。さらに`unresolved_issue_refs=[]`、Independent Designのunresolved finding `[]`を要求する。条件付き承認を保存しない。条件がある場合はRequirement／issueとして残し、すべて閉じた後にのみ`approved`を発行する。

`issued_at < valid_until`を必須とし、期限内でもsource、scope、Evidenceまたはreviewer authorityがstale／revokedなら有効にしない。`rejected`は一件以上のunresolved issueまたはreasonを必須にする。AI、Provider、author、Candidate作成者、Marketing claim作成者またはrelease automationだけでauthorized reviewer集合を満たさない。署名Role／Trust／revocationの共通意味はAI Verificationを参照する。

Decisionの署名subject projectionは`{legal_ip_decision_id,legal_ip_decision_version,legal_review_subject_ref,decision_sequence,previous_decision_ref,outcome,authorized_human_reviewer_refs[],decision_reason_artifact_ref,issued_at,valid_until}`のclosed canonical payloadである。Headの署名subject projectionも`signed_record_ref`と自己content hashだけを除くclosed payloadである。各`MirakanSignedRecordRefV1.subject_sha256`をこのprojectionのRFC 8785 JCS SHA-256へ一致させた後、署名Refを含む完成Decision／Headの型固有content hashを計算する。署名Envelope、Decision／Head content hash、subject projection hashを相互代用せず、self-referenceまたはhash cycleを作らない。

HeadはApplicability Profileごとのlinear current pointerである。初回はversion／sequence 1かつprevious `null`、更新は同じHead IDの直前versionと`previous.sequence + 1`へCASする。Head、Decision、Decisionが解決するReview Subject／Applicability Profileを全Field read-backし、dual genesis、branch、merge、skip、別Profile、unsigned、expiredまたはrevoked reviewer authorityを拒否する。DecisionとHeadの同一transaction publicationに失敗した場合は新Decisionをcurrentにせず、last-valid Headを維持する。

本書のexact Refは次のclosed tupleである。

| Ref | Field |
|---|---|
| `LegalAuthoritySourceSnapshotRefV1` | `{legal_source_snapshot_id, legal_source_snapshot_version, legal_source_snapshot_content_hash}` |
| `LegalJurisdictionProfileRefV1` | `{jurisdiction_profile_id, jurisdiction_profile_version, jurisdiction_profile_content_hash}` |
| `ProductLegalApplicabilityProfileRefV1` | `{applicability_profile_id, applicability_profile_version, applicability_profile_content_hash}` |
| `ProductLegalRequirementBindingRefV1` | `{legal_requirement_binding_id, legal_requirement_binding_version, legal_requirement_binding_content_hash}` |
| `IndependentDesignReviewSubjectRefV1` | `{independent_design_subject_id, independent_design_subject_version, independent_design_subject_content_hash}` |
| `ProductLegalIpReviewSubjectRefV1` | `{legal_review_subject_id, legal_review_subject_version, legal_review_subject_content_hash}` |
| `ProductLegalIpDecisionRefV1` | `{legal_ip_decision_id, legal_ip_decision_version, legal_ip_decision_content_hash}` |
| `ProductLegalIpDecisionHeadRefV1` | `{legal_ip_decision_head_id, legal_ip_decision_head_version, legal_ip_decision_head_content_hash}` |

各Refは解決先recordとbyte equalityにする。Head ID、最大version、時刻、一覧順、Decision outcomeまたはApplicability Profile名からcurrent Headを推測しない。

## 4. Applicability Closure Named Algorithm v1

1. exact Active Product Definition、Claim Scope、Candidate、Release Content Manifest、Public Contract Set、Engine Source Snapshotを解決し、ProfileのManifest、Candidate、Public Contract、Engine SourceをManifestの同Fieldとbyte equalityにする。
2. Product LifecycleのRelease Content Manifest／Distribution Coverage Projectionが選ぶ全required publication routeを、publication requirement、platform、channel、distribution scope、distribution subject、jurisdiction、market、Product role、commercial mode、user classへ完全展開する。
3. 各distribution bindingについて、承認済みJurisdiction Profileとeffective Legal Source Snapshotを解決する。
4. Active Product Definitionで`requirement_category=legal_ip`の全Product Requirementと、Profile、role、subjectから必要なLegal Requirement bindingをauthorized legal analysisにより選び、各`{Product Requirement,distribution binding,category}`を`required`または証拠付き`not_applicable`へexactly one total mappingする。
5. required bindingごとにDomain Ownerが要求するEvidence classとexact Evidenceを解決し、semantic admissibility、Candidate／subject／scope、freshness、revocationを検証する。
6. Independent Design Subjectが同じSource、Public Contract、UI／Documentation／Sample集合をcoverし、unresolved findingが0件であることを検証する。
7. required／supplied pairのset equalityとunresolved issue exact emptyを確認し、authorized human reviewerだけがimmutable Decisionへ署名する。
8. Product Release／Publication consumerは保存Outcomeだけでなく、current Decision Head、validity、source supersession、Evidence revocation、distribution scope equalityをread-time再検証する。

categoryの自動推測、legal sourceのAI要約だけによるapplicability決定、最も緩いjurisdictionへのfallback、market欠落、Decision後のscope追加、hash-only Evidenceまたは別Candidate Evidenceを拒否する。

## 5. Domain Evidence routing

| Legal subject | Domain Evidence Owner | Legal Ownerが行わないこと |
|---|---|---|
| dependency、SDK、tool、font、icon、model artifact | [Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md) | version、license、SBOM、NOTICE sourceを再定義しない |
| Asset／AI生成content | [Asset Lifecycle](../03-authoring/asset-lifecycle.md) | origin、terms、rights、commercial review payloadを再定義しない |
| Native／Shader source | [Project State](../03-authoring/project-state.md)、[Native Game Module](../03-authoring/native-game-module.md)、[Project Shader](../06-rendering/project-shader.md) | revision、Diff、Code Owner、build／test意味を再定義しない |
| first-party grant、redistribution、NOTICE presentation | [Product Lifecycle](../00-product/product-lifecycle.md) | license text、container、presentationを再定義しない |
| data flow、consent、processor、region、retention | [Product Privacy／Data Governance](product-privacy-data-governance.md) | Privacy applicability／acceptanceを代替しない |
| vulnerability、secure update、incident | [Product Security](product-security.md) | Threat、Case、Security Releaseを代替しない |
| AI authority、Provider connection、credential | [AI Security／Approval](ai-security-approval.md) | Task approvalをLegal Decisionへ読み替えない |
| Product claim、support scope、publication state | [Product Plan](../00-product/product-plan.md)、Release／Publication Owner | Product requirement、Release authority、公開結果を所有しない |
| Evidence envelope、signature、freshness、revocation | [AI Verification／Provenance](ai-verification-provenance.md) | domain evidenceまたは法務判断を自己発行しない |

Legal Ownerは上表のEvidenceをsame Subjectへ束縛し、欠落／scope差／staleをfail closedにする。Scanner success、SBOM生成、Provider commercial-use label、copyright notice、C2PA credentialまたは署名だけをrights approvalへ昇格しない。

## 6. Independent designと外部Engine非模倣

外部Engine、SDK、Tool、StoreまたはPlatformはgap discoveryの比較対象であり、MiraikanaiのSchema、API、type hierarchy、Scene／Project format、Editor layout、icon、sample、template、workflow、default、permission modelまたはcreative expressionの正本ではない。

許可する外部入力は次に限定する。

- 公開された公式documentation、公開standard、public release note。
- 正当に取得した製品を通常のlicense範囲で利用して観察したUser journeyとfailure mode。
- 公開APIのinteroperability requirementをauthorized legal reviewが許可した範囲。

次を禁止する。

- leaked、stolen、NDA／confidential、trade-secret material。
- licenseまたは法令に反するdecompilation、circumvention、scraping、extraction。
- external source、generated binding、Asset、sample、documentation、iconまたはUI expressionのcopy／translation／thin rewrite。
- 外部API、type、Scene、Plugin、format、command、defaultへの一対一aliasまたはinitial V1 compatibility layer。
- competitor名やmarketing表現をProduct identifier、feature nameまたはclaimへ流用すること。

共通概念を採択する場合も、MiraikanaiのUser requirement、Owner、state、invariant、failure、security、Evidenceから独立に導出し、採択理由と外部factを別recordにする。外部Engineの実装、人気または市場占有率を設計正当化やProduct Evidenceにしない。

## 7. Copyright、AI output、creative expression

Source code、documentation、sample、UI graphic、icon、font、music、audio、image、model、animation、video、game contentはそれぞれrights subjectとして扱う。アイデア／methodとexpressionの区別をArchitectureが一律判定せず、具体的なcopying、derivation、license、human authorship、territoryをreviewする。

AI生成物は、AIを使ったという理由だけでpublic domain、Miraikanai所有、copyrightable、commercially usableまたはredistributableと扱わない。Provider／Model／terms revision、入力Asset／Promptの権利、生成日時、human contribution、modification chain、output hash、territory／commercial restriction、attribution、expiryをDomain provenanceへ固定する。

AI生成SourceもProject Sourceと同じlicense、Code Owner、independent review、build／test、provenance条件を通す。Prompt、model responseまたはProvider dashboard表示をsource ownershipの唯一Evidenceにしない。

## 8. Trademark、trade dress、naming

Product名、logo、icon family、feature name、package ID、domain、sample character／world名、Editor visual identityはtarget marketごとにclearance subjectへ含める。[Naming／Project Layout](../02-foundation/naming-project-layout.md)はtechnical token／path規則を所有し、本書は第三者identifierとの権利reviewを所有する。

同一でない綴り、翻訳、接頭辞追加、色変更またはAI生成を非侵害Evidenceにしない。confusion、visual identityまたはmarket-specific issueが未解決なら該当marketのDecisionを承認しない。

## 9. Patent／freedom-to-operate

独立作成、copyright非侵害、公開standard準拠、OSS license適合または外部Source未閲覧はpatent FTOを証明しない。Patentはterritorialであり、対象jurisdiction、claim scope、Product feature、distribution／use role、評価日を固定したauthorized specialist reviewを別Requirementとして扱う。

自動keyword search、AI patent summary、件数、同一名称なし、expiredと思われる表示または競合製品の実装をFTO Decisionへ昇格しない。public disclosureがMiraikanai自身のpatent filing可能性へ影響し得る事項は、公開前の別IP protection reviewへ送る。

## 10. License、contract、contributor chain

Dependency／Asset／Provider／Platform／Store／font／icon／sample／documentation／contributionはexact terms revisionとsubjectへ束縛する。`MIT`、`commercial use allowed`等のlabelだけでnotice、source offer、copyleft、patent grant、trademark、redistribution、attribution、seat／usage、territoryまたはtermination条件を推測しない。

Contributor sourceはauthor identity、employer／contract authority、contribution terms、sign-off／assignment／license、source provenanceをchain-of-title Evidenceへ含める。anonymous paste、AI answer、forum snippet、unverified gistまたはlicense不明SourceをRepository sourceへ採用しない。

## 11. Product claimとcommercial representation

`AI-native`、`2D／3D`、`production-ready`、`secure`、`private`、`offline`、`compatible`、`faster`、`lower cost`等のexpress／implied claimは[Product Plan](../00-product/product-plan.md)のexact Claim Scopeとsame Candidate Evidenceへ束縛する。Marketing text、Screenshot、comparison table、benchmark chart、AI生成copyまたはdisclaimerは不足Evidenceを補わない。

Legal／IP Decisionはclaimの法的reviewを記録するが、technical truthを自己証明しない。Product ReleaseはOwner-issued technical Evidenceを先に満たし、Legal reviewは同じEvidence scopeと表示文言を審査する。

## 12. Lifecycleとeffective state

Decision recordはimmutableで、同一`{Product Definition, Claim Scope, Candidate, distribution binding set}`に一つのlinear sequenceを持つ。初回は`decision_sequence=1`かつprevious `null`、後続はcurrent completed Decisionをexact `N+1`で参照し、branch、gap、同sequence別bytesを拒否する。

Effective stateは保存Outcomeだけでなく次から投影する。

```text
not_reviewed | in_review | approved | rejected
| expired | revoked | superseded | scope_changed
```

`approved`はcurrent head、valid time、source snapshot、Evidence、reviewer authority、Product／Candidate／distribution scopeがすべてcurrentの場合だけ有効である。law／regulation／terms／license改訂、market／channel／role／claim／artifact変更、Evidence expiry／revocation、security incident、rights claimまたはcourt／regulator actionでは`scope_changed`または`revoked`へfail closedにし、新しいDecisionなしに復帰しない。

## 13. Failure atomicity、partial publication、recovery

Legal review中またはDecision発行失敗時にProduct Source、Candidate、Release Content Manifestまたは既存Decisionを変更しない。新Decisionとcurrent head publicationは一つのCASで行い、crash、signature failureまたはconflict時はlast-valid headを維持する。

複数marketを含むApplicability Profileの一部だけがreadyな場合、失敗marketを隠してProfile-wide `approved`を発行しない。承認済みsubsetだけを公開するには、そのsubsetをexactly列挙する別Applicability Profile、Claim／Release scope、Legal DecisionおよびRelease Decisionを発行する。同一Release Publicationの`partially_published`は承認済みbindingに対する外部routeの失敗／pendingを表すだけで、未承認marketをPublication Projectionへ混ぜる仕組みではない。未承認marketへのupload／download／claimを止め、別market Evidenceで代用しない。

既に公開済みSubjectのDecisionが失効／revokedになった場合、Recordを削除せずProduct Release／Publication Ownerへwithdrawal、update、notice、support、preservationのtyped actionを要求する。法的に不可逆な外部actionへ偽のrollbackを表示しない。

## 14. Version、Migration、Compatibility

current Repositoryにmaterialized Legal Schema／Decisionがないためinitial V1はdirect definitionであり、legacy alias、dual readerまたはmigration Operationを持たない。

公開後は次を区別する。

- Schema evolution: 本書と[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)のclassに従う。
- Legal source supersession: old Snapshotをimmutable auditとして保持し、新Snapshot／Requirement／Decisionを発行する。
- Scope migration: market、channel、role、claimまたはProduct artifact変更を新Applicability Profileへ移し、旧Approvalを流用しない。
- Provider／license terms change: affected subjectをstaleにし、new termsへの同意またはreviewをMigration成功と混同しない。
- Product Project migration: Legal DecisionはProject data migrationの正本ではなく、Compatibility／Domain OwnerのEvidenceを消費する。

法改正をSchema migrationだけで「対応済み」にせず、effective dateとauthorized applicability reviewを必須にする。

## 15. Security、Privacy、confidentiality

法務意見、patent analysis、claim strategy、契約、個人情報、未公開製品情報はprivileged／confidentialになり得る。一般Project Source、AI Prompt、MCP Resource、Support Bundle、telemetry、public Evidenceまたはrelease packageへ本文を含めない。

Review subjectは最小限のopaque Artifact Ref、hash、access policy、retention／legal hold、authorized reviewer identityを持ち、公開可能なsummaryを別Artifactにする。AI／external AgentはTask Authorizationとdata policyが明示的に許可したredacted projectionだけを参照できる。secret、credential、legal advice本文またはrestricted sourceをContextへ自動投入しない。

Privacy violationをLegal Approvalでwaiveせず、Security accepted riskを法令遵守へ読み替えない。各OwnerのGateをすべて独立に満たす。

## 16. Public／Internal APIとsurface boundary

本ContractにProduct Developer向けmutable public APIはない。将来公開できるのは、対象Releaseのpublic disclosure policyが許可するread-only `legal readiness summary`、license／NOTICE、known restriction、support contactのprojectionだけである。

Internal Operation候補はsource snapshot登録、Applicability Profile proposal、Evidence attachment、review request、Decision signing、status query、revoke／supersedeに限定する。AI、CLI、SDK、MCPまたはEditorからDecision signing、reviewer assignment、category削除、source改変、privileged artifact readを直接許可しない。各surfaceは同じAuthorization、validation、Audit、failureを使う。

## 17. Evidence、retention、audit

全Decisionは[AI Verification／Provenance](ai-verification-provenance.md)のsigned record、semantic admissibility、freshness、revocation、retentionを通る。Legal Ownerはgeneric Evidence Envelopeを複製しない。

Legal source、review subject、Decision、withdrawal reasonは監査期間中immutableに保持する一方、privileged content、personal data、Provider confidential termsはaccess control、retention、legal holdとdelete restrictionを別管理する。hashだけを保持する場合も、hashから法的内容またはapprovalを再構成できると主張しない。

## 18. Acceptanceとnegative conformance

Target acceptanceは少なくとも次をcoverする。

### Positive

- exact Product、Candidate、Claim Scope、Manifest、Public Contract、Source Snapshot、全distribution bindingを一つのReview Subjectへ閉じる。
- 全`{legal_ip Product Requirement,distribution binding}`で16 Legal Requirement categoryが`required`またはauthorized `not_applicable`へtotal mappingされる。
- required Evidence pairがset equality、fresh、non-revokedで、unresolved issue／similarity findingが0件である。
- authorized human reviewerだけがDecisionへ署名し、current headとread-time effective stateが`approved`になる。
- Product Release／Publicationが同じmarket／channel／subject集合をread-backする。

### Negative

- locale、currency、IP addressまたはStore名からjurisdictionを推測する。
- 一市場のApprovalを別市場、別channel、別role、別Candidateへ流用する。
- `AI-generated`、`commercial use` label、OSS scanner、SBOM、C2PA、signatureまたはsimilarity scoreだけでrightsを承認する。
- independent creationからpatent FTOまたはtrademark clearanceを推論する。
- competitor API／UI／Scene／Asset／sampleの一対一copy、thin rewriteまたはlegacy aliasを受理する。
- leaked／confidential／license違反materialをcomparison inputへ含める。
- stale source／terms、expired／revoked Evidence、missing category、conditional issueありで`approved`を発行する。
- privileged review本文をAI Context、Support Bundleまたはpublic packageへ漏らす。
- 部分market approvalをProduct-wide approvalとして表示する。

これらのSchema、Fixture、Suite、Runner、Decision、Evidence、legal operationは現Repositoryにmaterializeしていない。Markdown reviewをAcceptance実行済みと表現しない。

## 19. Diagnostic class

Target diagnostic familyは次を含む。Exact code RegistryはExecutable Contractsがmaterializeするまでemptyである。

- `diagnostic.legal.applicability-profile-missing`
- `diagnostic.legal.jurisdiction-unresolved`
- `diagnostic.legal.requirement-category-incomplete`
- `diagnostic.legal.source-stale-or-superseded`
- `diagnostic.legal.domain-evidence-missing-or-invalid`
- `diagnostic.legal.independent-design-finding-open`
- `diagnostic.legal.patent-review-missing`
- `diagnostic.legal.trademark-review-missing`
- `diagnostic.legal.rights-or-license-unresolved`
- `diagnostic.legal.unauthorized-reviewer`
- `diagnostic.legal.decision-expired-or-revoked`
- `diagnostic.legal.distribution-scope-mismatch`
- `diagnostic.legal.privileged-data-exposure`

自由文だけ、warning downgrade、User checkbox、AI confidenceまたはMarketing approvalでDiagnosticを消さない。

## 20. Current stateと完了条件

Current Repositoryには本書のSchema、Registry、Source Snapshot artifact、Applicability Profile、Requirement binding、Independent Design Subject、legal review artifact、Decision、Head、Operation、Fixture、Suite、Receipt、signing keyまたはlegal reviewer assignmentが存在しない。実装状態は`absent`、法的review状態は`not_reviewed`である。

Target-design完了条件は次である。

- 法域／market／channel／role／時点を暗黙推測せずexact Profileへ閉じる。
- 全Legal categoryがrequiredまたは証拠付きnot-applicableへtotal mappingされる。
- Domain Evidence OwnerとLegal Decision Ownerが重複しない。
- Independent design、copyright、trademark、patent FTO、license／contract、AI output、claim reviewを相互代用しない。
- Human legal authority、signature、freshness、revocation、partial publication、withdrawalが閉じる。
- privileged／confidential contentをpublic／AI／support dataへ漏らさない。
- Product Release／Publication consumerが同じscopeとDecision effective stateをread-backする。
- Architecture文書を法令遵守、非侵害、権利処理またはReleaseのEvidenceにしない。

## 21. 一次資料

- [WIPO Copyright Protection of Computer Software](https://www.wipo.int/en/web/copyright/activities/software)
- [WIPO Copyright protection](https://www.wipo.int/en/web/copyright/protection)
- [WIPO Copyright Treaty](https://www.wipo.int/en/web/treaties/ip/wct/index)
- [WIPO Patent law](https://www.wipo.int/en/web/patents/law)
- [WIPO Patent protection](https://www.wipo.int/en/web/patents/protection)
- [WIPO Trademark protection](https://www.wipo.int/en/web/trademarks/protection)
- [文化庁 AIと著作権について](https://www.bunka.go.jp/seisaku/chosakuken/aiandcopyright.html)
- [U.S. Copyright Office Copyright and Artificial Intelligence](https://www.copyright.gov/ai/)
- [Regulation (EU) 2024/1689, Artificial Intelligence Act](https://eur-lex.europa.eu/eli/reg/2024/1689/oj?locale=en)
- [Regulation (EU) 2024/2847, Cyber Resilience Act](https://eur-lex.europa.eu/eli/reg/2024/2847/oj?locale=en)
- [U.S. FTC Advertising and Marketing](https://www.ftc.gov/business-guidance/advertising-marketing)

上記は2026-08-04時点で確認したscope／authority factである。Miraikanaiへの具体的適用、解釈、legal sufficiencyおよびRelease判断は本書のauthorized human reviewを必要とする。
