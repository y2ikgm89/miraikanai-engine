# Miraikanai Engine AI Verification／Provenance Contract

- 文書ID: mirakan.arch.ai-verification-provenance
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Verification lifecycle、Requirement coverage、AI Evalの独立性、Evidence envelope意味、Receipt class、Test結果集約・retry・quarantine・waiver、freshness、Provenance、Trace grading、release evidence、保持、failure
- 非正本範囲: 具体Envelope／Registry／Fixture候補、AI authorization、Risk、Approval権限、Sandbox、Credential、MCP security。補助文書または各Ownerを参照する
- 規範依存: [Architecture Governance](architecture-governance.md)、[AI Security／Approval](ai-security-approval.md)、[Executable Contracts](../02-foundation/executable-contracts.md)
- 関連文書: [AI Evidence Envelope／Fixture Candidate Catalog](../appendices/ai-evidence-envelope-fixture-catalog.md)、[Product Plan](../00-product/product-plan.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Project State](../03-authoring/project-state.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-27

## 1. Evidence原則

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

Evidence envelopeは共通して、identity、subject、issuer、created time、freshness policy、input closure、result、diagnostic、signature／attestation refを持つ。Envelope classごとの具体field候補は[補助Catalog](../appendices/ai-evidence-envelope-fixture-catalog.md#7-evidence-envelope)に隔離する。

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

Technical Qualificationは定義済みTargetとCandidateに対する技術Evidenceの充足を表す。Architecture Approval、Product decision、release signingを代替しない。具体wire候補は[補助Catalog](../appendices/ai-evidence-envelope-fixture-catalog.md#72-technicalqualificationreceiptv1)を参照する。

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

## 8. Trace gradingとchain

TraceはSource、Candidate、Build、Test、Review、Qualification、Promotion、Release間のexact refを辿れる必要がある。Gradeは到達可能なEvidenceと欠損を表し、品質の主観評価またはApprovalを表さない。

## 9. Supply-chain provenance

Build provenanceはSource、Dependency lock、Toolchain、Builder、command profile、Artifact hashを束縛する。SBOMはexact Artifactへ対応させ、別buildまたはlatest dependency一覧を流用しない。Telemetry exportはSecretと個人情報をredactし、Evidence bodyとの対応を失わない。

## 10. External evidence、保持、freshness

FreshnessはEvidence class、対象、変更trigger、最大期間から決定する。時刻だけでなくSource、Dependency、Toolchain、Model、Policy、Target、Dataset変更でも失効させる。保持期限と削除要件は監査要件とsensitivityを分離して定義する。

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
