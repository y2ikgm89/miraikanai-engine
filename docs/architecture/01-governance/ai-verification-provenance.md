# Miraikanai Engine AI Verification／Provenance Contract

- 文書ID: mirakan.arch.ai-verification-provenance
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Verification lifecycle、Requirement coverage、AI Evalの独立性、Qualification Scenario／Evidence Class identity、generic Verification Scope／subject contract／Evidence／Qualification Receipt spine、semantic admissibility predicateとclosed resolver Registry、`MirakanSignedRecordV1`共通署名Envelope／Ref、Evidence envelope意味、Receipt class、Test結果集約・retry・quarantine・waiver、freshness、Provenance、Trace grading、release evidence、保持、failure
- 非正本範囲: Domain固有Evidence／Receipt payload、materialized Registry／Fixture候補、AI authorization、Risk、Approval権限、Sandbox、Credential、MCP security。補助文書または各Ownerを参照する
- 規範依存: [Architecture Governance](architecture-governance.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](ai-security-approval.md)、[Executable Contracts](../02-foundation/executable-contracts.md)
- 関連文書: [AI Evidence Envelope／Fixture Candidate Catalog](../appendices/ai-evidence-envelope-fixture-catalog.md)、[AI Production Orchestration](../03-authoring/ai-production-orchestration.md)、[AI Production Orchestration Ownership Decision](../decisions/2026-08-04-ai-production-orchestration-ownership.md)、[Product Legal／IP Governance](product-legal-ip-governance.md)、[Game Production Loop](../03-authoring/game-production-loop.md)、[Product Plan](../00-product/product-plan.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Project State](../03-authoring/project-state.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-27

## 1. Evidence原則

Legal／IP reviewで使うEvidenceも本書のgeneric Envelope、semantic admissibility、freshness、revocation、retentionへ従うが、法的applicability、privilege、review authorityまたはApproval outcomeは[Product Legal／IP Governance](product-legal-ip-governance.md)だけが所有する。署名、hash、provenanceまたはEvidence class単独を法令遵守、権利処理または非侵害の証明にしない。

AI出力、Schema適合、AIが生成したTest、単一Benchmark、reviewer一名の判断のいずれかだけで変更を採用しない。Riskに応じ、互いに独立したRequirement、検証、Approvalを束縛する。

Evidenceは次を満たす。

1. 対象Candidate、入力、Target、Toolchain、Model、Policy、Validator、期待結果をimmutable refで特定する。
2. Evidence作成主体と変更主体を分離し、同じ生成経路だけの自己承認を認めない。
3. pass／failはclosed predicateから決め、欠損をpassへ補完しない。
4. Secret、credential、live pointer、未認可SourceをPublic Evidenceへ含めない。
5. 文書上の候補SchemaまたはFixture一覧を、発行済みReceiptとして扱わない。

## 2. Verification lifecycle

Candidateは`proposed -> prepared -> evaluated -> reviewed -> qualified | rejected`の順に進む。各遷移は前状態、Candidate identity、Requirement集合、Evidence集合をexactに束縛する。部分集合、latest検索、表示名一致、署名のない外部結果で遷移しない。

`qualified`は対象Capability、Target、version、Contract set、freshness windowに限定され、一般的な実装完了または将来versionの適合を意味しない。失効、入力変更、Toolchain変更、Model／Prompt変更、Target変更時は再評価する。

## 3. Requirement coverageとTest独立性

RequirementはOwner、対象、verification method、acceptance predicate、failure codeを持つ。全mandatory Requirementに少なくとも一つの独立Evidenceが必要であり、同じEvidenceが技術検証と人間Approvalを兼用してはならない。

Testは正例だけでなく、境界値、malformed input、権限不足、stale ref、hash不一致、timeout、partial publication、replay不一致を含む。候補生成時に使った公開Datasetだけで昇格を決めず、holdoutとadversarial setを分離する。

## 4. Formal model

形式モデルは、bounded state、遷移、invariant、failure stateが明示できる契約へ適用する。モデルのbound外を証明済みと表現せず、実装traceとの対応表を必須にする。

## 5. AI Eval lifecycle

AI Evalは固定Suite identity、Dataset partition、grader identity、Model snapshot、Prompt／Tool Schema hash、Target policyを束縛する。平均値だけでなくhard failure、worst-case、variance、abstention、unsafe action、unauthorized disclosureを判定する。

### 5.5 代表Fixture

代表Fixtureの具体recordとDataset候補は[AI Evidence Envelope／Fixture Candidate Catalog](../appendices/ai-evidence-envelope-fixture-catalog.md#55-代表fixture)へ分離する。Owner上の必須分類はpositive、negative、adversarial、authorization denial、stale evidence、partial publication、recoveryである。補助文書の件数やIDはArtifactがmaterializeするまでcandidateであり、Fixture充足を意味しない。

## 6. Performance／reliability Evidence

性能EvidenceはTarget、workload、warmup、sample数、clock、Toolchain、hardware、power／thermal状態を記録する。単一端末または平均値を全Targetの保証へ一般化しない。Reliabilityはtimeout、retry、idempotency、backpressure、crash recovery、partial failureを独立に観測する。

## 7. Evidence envelope

<a id="verification-identity-spine"></a>

Qualification Scenario、Evidence Class、generic Evidence／Qualification Receiptと共通署名EnvelopeのArchitecture上のstable identity spineは本節が一意に所有する。Domain固有payloadとFixture候補は[補助Catalog](../appendices/ai-evidence-envelope-fixture-catalog.md#7-evidence-envelope)へ隔離できるが、次のRef、complete backing record、resolver root、canonical hash規則をappendix、consumer-local tupleまたは表示名から補完しない。

<a id="signed-record-envelope"></a>

### 7.0 `MirakanSignedRecordV1`

全署名Recordは[AI Security／Approval](ai-security-approval.md)のalgorithm／Key／Role／purpose policyを消費し、次の唯一の共通Envelopeを使う。JSON Schema `$id`は`urn:mirakan:schema:governance:mirakan-signed-record:v1`であり、consumer schemaはこのrootをexact `$ref`し、署名Fieldをinline再定義しない。

```text
MirakanSignedRecordV1
  envelope_version: 1
  purpose: registered closed purpose
  subject_sha256: SHA-256
  signer_subject_ref: exact TrustSubjectRefV1
  signer_role_ref: exact TrustRoleRefV1
  key_id: StableId (exact TrustKeyV1 lookup)
  issued_at: UtcTimestamp
  revocation_snapshot_ref: exact RevocationSnapshotRefV1
  signature_algorithm: registered closed algorithm
  signature_format: registered closed format
  signature: bounded bytes

MirakanSignedRecordRefV1
  envelope_schema_id:
    urn:mirakan:schema:governance:mirakan-signed-record:v1
  purpose: registered closed purpose
  subject_sha256: SHA-256
  signed_record_hash: SHA-256
```

全Fieldは必須、unknown Fieldは禁止する。`subject_sha256`は用途別Schemaで閉じたsubject payloadのRFC 8785 JCS bytesをSHA-256したlowercase 64桁hexであり、payloadまたはpayload refをEnvelopeへ複写しない。署名対象は`signature`だけを除くEnvelope FieldのRFC 8785 JCS bytesである。`purpose`、subject hash、Signer、Role、Key、発行時刻、発行時revocation snapshot、algorithm、formatの一つでも変われば署名は成立しない。Refは完成Recordのschema ID、purpose、subject hashとbyte equalityにし、`signed_record_hash`を署名を含む完成Record全体のRFC 8785 JCS SHA-256へ一致させる。別Schema ID、wrong purpose／subject、hash-only ref、inline署名Fieldを拒否する。

VerifierはSchema／canonical encoding、現在のsubject payload bytesから再計算したhash、用途別exact purposeを先に検査する。続いて[AI Security／Approval §6](ai-security-approval.md#trust-identity-spine)のcurrent Trust RegistryでSigner／Role Refと`key_id`を完成recordへexact解決し、Key所有者、許可purpose、algorithm／format、発行時の有効期間を照合する。その後、発行時snapshotとそれ以後のcurrent revocation snapshotの署名／sequenceを検証する。current snapshotがRecord、subject、Signer、Role、Keyまたはpurposeを失効対象に含む場合は拒否する。missing Envelope、invalid signature、unknown／期限外／用途不一致Key、Role不一致、stale／invalid snapshot、revoked対象をfail closedにし、Verification keyをApproval／Promotionへ流用しない。

Domain Qualification subjectが`qualification_subject_hash`を持つ場合は二段階hashを共通規則とする。まずDomain固有ASCII separatorと同Fieldだけを除くclosed canonical subject bytesからcontent identityを計算し、完成Subjectへ格納する。次にそのFieldを含む完成Subject全体のRFC 8785 JCS hashを`MirakanSignedRecordV1.subject_sha256`とする。Qualification Receipt Refのcontent identity、`MirakanSignedRecordRefV1.subject_sha256`、`signed_record_hash`を相互代用せず、それぞれ完成Subject内Field、完成Subject JCS、完成Envelope JCSへexact一致させる。

| 型 | ASCII domain separator |
|---|---|
| `QualificationScenarioV1` | `MIRAKAN_QUALIFICATION_SCENARIO_V1` |
| `QualificationScenarioRegistryV1` | `MIRAKAN_QUALIFICATION_SCENARIO_REGISTRY_V1` |
| `EvidenceClassV1` | `MIRAKAN_EVIDENCE_CLASS_V1` |
| `EvidenceClassRegistryV1` | `MIRAKAN_EVIDENCE_CLASS_REGISTRY_V1` |
| `VerificationRecordTypeRegistryV1` | `MIRAKAN_VERIFICATION_RECORD_TYPE_REGISTRY_V1` |
| `EvidenceRecordV1` | `MIRAKAN_EVIDENCE_RECORD_V1` |
| `QualificationReceiptV1` | `MIRAKAN_QUALIFICATION_RECEIPT_V1` |

```text
QualificationScenarioV1
  qualification_scenario_id: StableId
  qualification_scenario_version: positive u32
  owner_requirement_refs[1..4096]:
    sorted unique exact McdContractRefV1(kind=requirement)
  scenario_purpose_ref:
    exact McdContractRefV1(kind=policy)
  host_applicability:
    {kind=not_applicable}
    | {kind=host_independent}
    | {
        kind=exact_set,
        host_profile_refs[1..64]:
          sorted unique exact TargetProfileRefV1(
            profile_kind=build_host | editor_host)
      }
  target_applicability:
    {kind=not_applicable}
    | {kind=target_independent}
    | {
        kind=exact_set,
        target_profile_refs[1..64]:
          sorted unique exact TargetProfileRefV1(
            profile_kind=runtime_target)
      }
  locale_applicability:
    {kind=not_applicable}
    | {kind=locale_independent}
    | {
        kind=exact_set,
        locale_profile_refs[1..64]:
          sorted unique exact LocaleProfileRefV1
      }
  reference_dimension_applicability:
    {kind=not_applicable}
    | {kind=dimension_independent}
    | {
        kind=exact_set,
        reference_dimensions[1..2]:
          sorted unique two_d | three_d
      }
  expected_result_branches[1..3]:
    sorted unique
      success | expected_policy_rejection | domain_failure_recovery
  immutable_input_contract_refs[1..4096]:
    sorted unique exact McdContractRefV1(kind=type)
  immutable_result_contract_refs[1..4096]:
    sorted unique exact McdContractRefV1(kind=type)
  qualification_scenario_content_hash: SHA-256

QualificationScenarioRegistryV1
  qualification_scenario_registry_id: StableId
  qualification_scenario_registry_version: 1
  qualification_scenario_refs[1..65535]:
    sorted unique exact QualificationScenarioRefV1
  qualification_scenario_registry_content_hash: SHA-256

EvidenceClassV1
  evidence_class_id: StableId
  evidence_class_version: positive u32
  evidence_semantic_kind:
    contract_conformance | technical_qualification | human_review
    | promotion | release_signing | platform_submission
    | publication_readback | operation_result | developer_test
    | security | privacy | license | lifecycle | support | provenance
  allowed_record_kinds[1..2]:
    sorted unique evidence | qualification_receipt
  allowed_purpose_refs[1..256]:
    sorted unique exact McdContractRefV1(kind=policy)
  required_subject_contract_refs[1..4096]:
    sorted unique exact McdContractRefV1(kind=type)
  evidence_class_content_hash: SHA-256

EvidenceClassRegistryV1
  evidence_class_registry_id: StableId
  evidence_class_registry_version: 1
  evidence_class_refs[1..4096]:
    sorted unique exact EvidenceClassRefV1
  evidence_class_registry_content_hash: SHA-256

VerificationScopeVectorV1
  host_scope:
    {kind=not_applicable}
    | {kind=host_independent}
    | {
        kind=host_profile,
        host_profile_ref:
          exact TargetProfileRefV1(
            profile_kind=build_host | editor_host)
      }
  target_scope:
    {kind=not_applicable}
    | {kind=target_independent}
    | {
        kind=target_profile,
        target_profile_ref:
          exact TargetProfileRefV1(profile_kind=runtime_target)
      }
  locale_scope:
    {kind=not_applicable}
    | {kind=locale_independent}
    | {
        kind=locale_profile,
        locale_profile_ref: exact LocaleProfileRefV1
      }
  reference_dimension_scope:
    {kind=not_applicable}
    | {kind=dimension_independent}
    | {
        kind=reference_dimension,
        reference_dimension: two_d | three_d
      }

VerificationRecordTypeRegistryV1
  verification_record_type_registry_id: StableId
  verification_record_type_registry_version: 1
  record_type_entries[1..65535]:
    sorted unique {
      owner_record_type_ref: exact McdContractRefV1(kind=type),
      record_kind: evidence | qualification_receipt,
      evidence_class_refs[1..4096]:
        sorted unique exact EvidenceClassRefV1,
      allowed_purpose_refs[1..256]:
        sorted unique exact McdContractRefV1(kind=policy),
      required_subject_contract_refs[1..4096]:
        sorted unique exact McdContractRefV1(kind=type)
    }
  verification_record_type_registry_content_hash: SHA-256

EvidenceRecordV1
  evidence_id: StableId
  evidence_version: positive u32
  evidence_class_registry_ref: exact EvidenceClassRegistryRefV1
  verification_record_type_registry_ref:
    exact VerificationRecordTypeRegistryRefV1
  evidence_class_ref: exact EvidenceClassRefV1
  evidence_purpose_ref: exact McdContractRefV1(kind=policy)
  owner_evidence_record:
    owner_record_type_ref: exact McdContractRefV1(kind=type)
    owner_record_id: StableId
    owner_record_version: positive u32
    owner_record_content_hash: SHA-256
  verification_scope: exact VerificationScopeVectorV1
  subject_contract_refs[1..4096]:
    sorted unique exact McdContractRefV1(kind=type)
  subject_content_hash: SHA-256
  completed_signed_record_content_hash: SHA-256
  evidence_record_content_hash: SHA-256

QualificationReceiptV1
  qualification_receipt_id: StableId
  qualification_receipt_version: positive u32
  qualification_scenario_registry_ref:
    exact QualificationScenarioRegistryRefV1
  evidence_class_registry_ref: exact EvidenceClassRegistryRefV1
  verification_record_type_registry_ref:
    exact VerificationRecordTypeRegistryRefV1
  qualification_scenario_ref: exact QualificationScenarioRefV1
  evidence_class_ref: exact EvidenceClassRefV1
  qualification_purpose_ref: exact McdContractRefV1(kind=policy)
  owner_qualification_receipt_record:
    owner_record_type_ref: exact McdContractRefV1(kind=type)
    owner_record_id: StableId
    owner_record_version: positive u32
    owner_record_content_hash: SHA-256
  verification_scope: exact VerificationScopeVectorV1
  subject_contract_refs[1..4096]:
    sorted unique exact McdContractRefV1(kind=type)
  subject_content_hash: SHA-256
  observed_result_branch:
    success | expected_policy_rejection | domain_failure_recovery
  completed_signed_record_content_hash: SHA-256
  qualification_receipt_content_hash: SHA-256
```

本書Ownerのexact Refは次のclosed tupleである。

| Ref | Field |
|---|---|
| `QualificationScenarioRefV1` | `{qualification_scenario_id, qualification_scenario_version, qualification_scenario_content_hash}` |
| `QualificationScenarioRegistryRefV1` | `{qualification_scenario_registry_id, qualification_scenario_registry_version=1, qualification_scenario_registry_content_hash}` |
| `EvidenceClassRefV1` | `{evidence_class_id, evidence_class_version, evidence_class_content_hash}` |
| `EvidenceClassRegistryRefV1` | `{evidence_class_registry_id, evidence_class_registry_version=1, evidence_class_registry_content_hash}` |
| `VerificationRecordTypeRegistryRefV1` | `{verification_record_type_registry_id, verification_record_type_registry_version=1, verification_record_type_registry_content_hash}` |
| `EvidenceRefV1` | `{evidence_id, evidence_version, evidence_class_ref, evidence_purpose_ref, owner_record_type_ref, owner_record_id, owner_record_version, owner_record_content_hash, verification_scope, subject_contract_refs, subject_content_hash, completed_signed_record_content_hash, evidence_record_content_hash}` |
| `QualificationReceiptRefV1` | `{qualification_receipt_id, qualification_receipt_version, qualification_scenario_ref, evidence_class_ref, qualification_purpose_ref, owner_record_type_ref, owner_record_id, owner_record_version, owner_record_content_hash, verification_scope, subject_contract_refs, subject_content_hash, observed_result_branch, completed_signed_record_content_hash, qualification_receipt_content_hash}` |

各Registry Refはexactly one complete Registryへ、member RefはそのRegistryからexactly one complete recordへ全Fieldで解決する。`VerificationRecordTypeRegistryV1`の各entryは`owner_record_type_ref`が解決するMCD Type recordの一意Ownerと、許可Evidence class／purpose、required subject contractを閉じる。Evidence／Receiptは同Registryのexact一entryへrecord kind、owner Type、Evidence class、purposeで解決し、owner-specific recordのID／version／content hashをread-backする。wrapperの`subject_contract_refs[]`は、解決したEvidence ClassとType Registry entryの`required_subject_contract_refs[]` canonical unionとexact set equalityにする。Registry validityは、record kindがClassの`allowed_record_kinds[]`に属し、Class refがentryの`evidence_class_refs[]`に属し、両`allowed_purpose_refs[]`のintersectionがnon-emptyな全許可pairについて、このcanonical unionのchecked cardinalityが4,096以下であることを先に要求する。checked overflowまたは4,096超過のpairをRegistryへ登録せず、Evidence／Receipt生成時の表現不能へ遅延させない。owner-specific completed recordからscope vector、subject contract集合、subject content hash、Qualificationのscenario／observed branchを再計算してgeneric wrapper／Refとbyte equalityにし、wrong class、wrong record kind、wrong purpose、wrong owner Type、wrong scenario、wrong subject、hash-only Ref、Technical QualificationによるReview／Promotion／Signing代用を拒否する。

全recordとRegistryのcontent hashは自己hashだけを除く全Fieldを型固有domain separator、algorithm `sha256`、algorithm version 1、schema順、`uint32_be` length framingでcanonical encodeして計算する。Ref内のcompleted signed-record hashは署名wrapperを含む完了recordを束縛し、bodyとsignature wrapperを相互hashしない。ID／versionだけ、同名class／scenario、display label、prefix、`latest`、consumer-local tuple、appendix candidateからRefを生成しない。Target／Locale Refは[Product Planのroot Registry](../00-product/product-plan.md#product-profile-identity)へexact解決し、Scenario、Evidence subjectまたはReceipt間でkind、Host、Target、locale、dimensionを置換しない。

Verification Semantic Admissibility Predicate v1は、recordを任意のRequirement Set、set equality、Acceptance、Release、PublicationまたはCompletionへ参加させる前に次をすべて評価する。構文的Ref解決またはcontent hash一致だけをpredicateの代用にしない。

1. Ref、generic wrapper、owner-specific completed record、signature wrapperをすべてexact解決し、各content hash／signed-record hashを再計算する。
2. owner Type、record kind、Evidence Class、purpose、subject contract集合をType Registry／Evidence Classの許可集合と上記exact union規則で検証する。
3. `VerificationScopeVectorV1`のHost、runtime Target、locale、Reference dimensionをProduct Planのroot Registryへ解決し、owner-specific subjectの同scopeと全Fieldbyte equalityにする。`not_applicable`、independent、scalar Profile／dimension branchを相互変換しない。
4. QualificationではScenarioを解決し、Scenarioの各applicabilityが`not_applicable`なら同scopeだけ、independentなら同independent scopeだけ、`exact_set`ならそのexact memberのscalar scopeだけを許す。`observed_result_branch`はScenarioの`expected_result_branches[]`のexact memberでなければならない。
5. Evidence Class、Scenarioまたはowner Type自身のinvariantに違反するtuple、別Contract set、stale／revoked record、subject hashだけ一致してscope／contract／branchが異なるrecordをfail-closedで拒否する。

Product LifecycleがProduct PlanのJourney projectionをQualification obligationとして消費する境界、Documentation／SDK／Support／Lifecycle transition Requirement Set、Pack lifecycle、各Domain Qualification、Release Decision、Publication／Completionはこのpredicateを各rowのQualification生成時とReceipt read-back時に適用する。Product Planのreceipt-free required universeへVerification Receiptを逆参照させない。無効tupleをrequired集合へ複写してset equalityを成立させること、generic wrapperだけを検証してowner-specific semanticsを省略することを禁止する。現RepositoryにはこれらのSchema、Registry、record、resolverまたはReceiptが存在せず、本節の設計記載を発行済みEvidenceへ数えない。

Evidence envelopeは共通して、上記identity spineに加え、issuer、created time、freshness policy、input closure、result、diagnostic、signature／attestation refを持つ。Envelope classごとのDomain固有field候補は補助Catalogに隔離する。

Ownerが決定する不変条件は次のとおりである。

- Evidence bodyとsignature wrapperを同じobjectとして循環hashしない。
- Public bodyへsecret、raw prompt、credential、private dataset contentを含めない。
- Receiptが参照するRequirement、Candidate、Target、Toolchainを後から差し替えない。
- missing、unknown、expired、revoked、wrong-subject Evidenceをfail-closedで拒否する。
- Review、Technical Qualification、Promotion、Signing、Uploadを一つの万能Receiptへ統合しない。

### 7.1 VerificationReceiptV1

Verification Receiptは一つの検証実行と結果を表し、Product昇格または人間Approvalそのものを表さない。

#### 7.1.1 Evidence Requirementとfulfillment binding

Requirementとfulfillmentは別recordとし、fulfillmentはexact Requirement version、acceptance predicate、Evidence ref集合を束縛する。Requirement欠損、Evidence部分集合、別Targetの代用を拒否する。具体候補は[補助Catalog](../appendices/ai-evidence-envelope-fixture-catalog.md#711-evidence-requirementとfulfillment-binding)を参照する。

#### 7.1.2 外部Registry snapshotとauthority boundary

外部Registry Evidenceはauthority profile、tenant／namespace、query scope、pagination closure、取得時刻、collector identityを必須にする。検索結果や一部pageを完全集合と呼ばない。具体snapshot候補は[補助Catalog](../appendices/ai-evidence-envelope-fixture-catalog.md#712-外部registry-snapshotとauthority-boundary)を参照する。

### 7.2 TechnicalQualificationReceiptV1

Technical Qualificationは定義済みTargetとCandidateに対する技術Evidenceの充足を表す。Architecture Approval、Product decision、release signingを代替しない。stable identityは本節の`QualificationReceiptV1`／Refへ従い、Domain固有payload候補は[補助Catalog](../appendices/ai-evidence-envelope-fixture-catalog.md#72-technicalqualificationreceiptv1)を参照する。

### 7.3 GenerationReceiptV1

Generation Receiptは生成Attempt、入力Closure、生成物候補、Validator結果を束縛する。生成物のpublicationまたは昇格は別境界とする。

### 7.4 ReviewReceiptV1

Review Receiptはreview scope、reviewer authority、finding、decisionを束縛し、技術Qualificationを代用しない。具体候補は[補助Catalog](../appendices/ai-evidence-envelope-fixture-catalog.md#74-reviewreceiptv1)を参照する。

### 7.5 PromotionReceiptV1

Promotion Receiptはqualified Candidateを特定のProduct／Target stateへ昇格する判断を表す。Candidate、Evidence、Approvalの不一致を拒否する。

### 7.6 ReleaseSigningReceiptV1

Signing Receiptは署名入力、signer、certificate／key identityの非秘密ref、出力Artifact hashを束縛する。秘密鍵材料をReceiptへ含めない。

### 7.7 StoreUploadReceiptV1

Store Upload Receiptはupload対象、destination、server response、submission identityを記録し、Store承認または公開完了と同一視しない。

### 7.8 Product release evidence class aggregation

Release Gateは必要なEvidence classのset equalityを検証する。万能Receipt、class alias、missing classの黙示補完を禁止する。

### 7.9 Test結果集約、retry、quarantine

Test aggregationは、各attemptのCandidate、Requirement、suite／fixture、Target、Toolchain、device／runner、input closure、開始／終了、`passed | failed | blocked | infrastructure_error`結果を保持し、定義済みGateが要求するTest集合とのset equalityから判定する。未実行、欠損、unknown、timeout、runner crash、log欠損を`passed`または黙示的`skipped`へ変換しない。

retryは新しいattemptであり、最初の失敗を上書きしない。同じCandidateに対する全attemptとretry reasonを保持し、再実行で通っただけのflaky Testを安定したpassとして集約しない。自動retryはInfrastructure分類とbounded policyに限定し、product defect、security failure、determinism mismatch、data corruptionをInfrastructureへ分類変更して迂回しない。

quarantineはTestの存在と既知問題を追跡する隔離状態であり、Requirement充足ではない。mandatory Testがquarantine中なら対応Gateは`blocked`またはincompleteであり、Test件数から除外してpass率を上げない。解除には原因、修正Candidate、再現fixture、連続pass条件を持つ独立Evidenceを要求する。

waiverはOwner、対象Requirement／Candidate／Target、理由、代替Evidence、発行時刻、expiry、revocationを束縛する明示的なGovernance判断である。別Candidate、別Target、期限後へ継承せず、Security、integrity、credential、署名、artifact identityの必須条件を迂回しない。waiverを`passed`へ書き換えず、Release summaryへ未充足Requirementとともに表示する。

Unit、headless、simulator、screenshot、snapshot、semantic tree、accessibility、performance分布、physical-device sessionは互いの代替ではない。Requirementが指定するclassをexactに満たし、特にphysical-device、install／upgrade、launch／lifecycle、input、thermal／power、accessibility実機、P95／P99 tailをhost／simulator結果から推測しない。

### 7.10 AI Production Run／Workflow lineage

[AI Production Orchestration](../03-authoring/ai-production-orchestration.md)のRun、Attempt、Workflow Binding、Run Context、Route Selection、ResultはEvidenceのcausal subjectまたはinput lineageになれるが、Evidence class、admissibility、freshness、pass、QualificationまたはRelease stateを所有しない。Run `completed`、Attempt `succeeded`、Workflow terminal step、Model self-critique、conversation summary、Agent／MCP session completionをEvidence ReceiptまたはTest passへ変換しない。

Run中に消費する既存EvidenceはRun Contextへexact ref／hashとして固定し、dispatch直前に本書のeffective stateを再評価する。Run／Resultを評価して後から発行するGeneration／Review／Verification Receiptは完成RunまたはResultを一方向参照し、Run／ResultへそのReceiptを埋め戻さない。Runが既存Evidenceを参照し、同じEvidenceがそのRun／Resultをsubjectにする循環、Receipt発行後のResult書換え、Result hashだけからEvidence classを推測する経路を拒否する。

retry／resumeは同Runの別Attempt、explicit fallbackまたはModel／Host／route変更は旧Runをterminalへ閉じた後の別Runとinitial Attemptである。Evidence envelopeは実際に使用したexact Run、Attempt、Context、route、Provider／Host attributionの証明可能範囲を保持する。`standard_external_agent`で上流Provider／Modelがunattestedの場合、そのmetadataをGeneration attributionへ昇格せず、Host／Transport／Schema Conformanceとtyped Proposal lineageだけを記録する。first-party／managed routeもProfile名だけでattributionせず、AI Security所有のcurrent Caller Context／Conformance／Attestationへ解決する。

## 8. Trace gradingとchain

TraceはSource、Candidate、Build、Test、Review、Qualification、Promotion、Release間のexact refを辿れる必要がある。Gradeは到達可能なEvidenceと欠損を表し、品質の主観評価またはApprovalを表さない。

## 9. Supply-chain provenance

Build provenanceはSource、Dependency lock、Toolchain、Builder、command profile、Artifact hashを束縛する。SBOMはexact Artifactへ対応させ、別buildまたはlatest dependency一覧を流用しない。Telemetry exportはSecretと個人情報をredactし、Evidence bodyとの対応を失わない。

## 10. External evidence、保持、freshness

FreshnessはEvidence class、対象、変更trigger、最大期間から決定する。時刻だけでなくSource、Dependency、Toolchain、Model、Policy、Target、Dataset変更でも失効させる。保持期限と削除要件は監査要件とsensitivityを分離して定義する。

### 10.1 Evidence effective state

```text
EvidenceEffectiveStateV1 = fresh | expired | revoked | invalid
```

Evidence consumerは、完成Evidence recordと署名wrapper、評価時のcurrent Trust／revocation snapshot、purpose、subject、required context、freshness policy、`evaluation_time`から一つのeffective stateを導出する。複数条件が同時に成立する場合の唯一の優先順位は`invalid > revoked > expired > fresh`である。

| state | exact condition |
|---|---|
| `invalid` | schema／canonical encoding、署名、purpose、owner Type、subject／content hash、required Evidence集合、Trust chain、Project／Candidate／Contract set／Target／environment contextのいずれかが欠損、不一致または検証不能 |
| `revoked` | record、subject、issuer、Role、Key、Policyまたはtransitive Evidenceの一件が評価時current revocation snapshotで失効し、かつ`invalid`条件がない |
| `expired` | validかつnon-revokedだが`evaluation_time >= expires_at`、またはfreshness policyが固定するsource generation、Toolchain、Model、Dataset、Target／device environmentの許容期間外 |
| `fresh` | valid、non-revoked、期限内で、purpose、subject、required context、current generation／environmentが全てbyte equality |

`stale`を第五stateとして保存しない。Project／Candidate／Contract set／Target／generation／context mismatchは`invalid`、時間またはpolicy window超過は`expired`とし、原因別Diagnosticへ元のstale reasonを保持する。未知state、評価時刻欠損、current revocation snapshot欠損、freshness policy未解決を`fresh`へdefaultしない。Positive Evidence、coverage、Qualification、Approval、Promotion、Activation、ReleaseおよびGame understanding／production loop closureへ数えられるのは`fresh`だけである。

Evidence集合のeffective stateは各memberへ同じ評価contextを適用して導出し、一件でも`fresh`以外なら集合全体をpositive closureとして受理しない。上位Receiptで古いEvidenceを再包装して期限を延長せず、transitive memberの`invalid | revoked | expired`を上位の新しい署名で相殺しない。

## 11. Dependency／toolchain Evidence

Dependency Evidenceはexact version、source、license、integrity、resolved artifactを束縛する。Tool名またはsemver rangeだけでQualificationしない。

## 12. Security negative verification

最低限、権限不足、Approval欠損、credential exposure、path traversal、network denial、prompt injection、tool argument spoofing、stale grant、replay、signature mismatchをnegative fixtureへ含める。

## 13. Failure、Incident、recovery

失敗時はlast-valid stateを維持し、部分Publicationを残さない。Incident Evidenceは原入力の機密値を複製せず、再現に必要なredacted closure、diagnostic、causalityを保存する。

## 14. CI lanes

CI laneは対象Requirement、Candidate、Target、Toolchain、device／runner、isolation、credential class、Artifact retention、freshnessを固定する。Gateはlane名や緑色表示ではなく、§7.9のattempt結果と必要lane集合のset equalityを検証する。具体lane候補は[補助Catalog](../appendices/ai-evidence-envelope-fixture-catalog.md#14-ci-lanes)へ分離し、Runner、Test Artifact、Receiptが存在するまで実行済みと表現しない。

## 15. 完了条件

設計上の完了は、Requirement coverage、独立Evidence、negative verification、freshness、failure、trace、retentionが矛盾なく定義されることである。実装完了またはQualification passはRepository ArtifactとReceiptで別途証明する。

## 16. 一次根拠

- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [SLSA Provenance](https://slsa.dev/spec/v1.2/provenance)
- [in-toto Attestation Framework](https://github.com/in-toto/attestation)
- [OpenSSF Scorecard](https://scorecard.dev/)
