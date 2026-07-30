# Product Release／Publication Authority Ownership

- 文書ID: mirakan.decision.product-release-publication-authority
- 状態: review
- 正本範囲: Release requirement projection、署名済みRelease authorization、Platform publication、Product publication／completionのOwner分離と一方向DAG
- 非正本範囲: 各OwnerのSchema／固定値／Operation／Receipt、Platform固有signing／submission、実装Task、実装順序、担当、工数、日程、Capability activation。各Owner文書を参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Product Plan](../00-product/product-plan.md)、[Product Lifecycle](../00-product/product-lifecycle.md)、[Product Security](../01-governance/product-security.md)、[Product Release Decision](../00-product/product-release-decision.md)、[Product Publication／Completion](../00-product/product-publication-completion.md)
- 外部根拠検証日: none
- 文書種別: Architecture Decision／cross-owner authority
- Decision owner document: `mirakan.arch.architecture-governance`
- Decision日: 2026-07-30
- Supersedes: none

## Context

Product Plan、Lifecycle Acceptance、Security Release Bindingが存在しても、それらは「誰が、どのrequired universeに対して、いつまで有効なRelease authorizationを発行したか」を表さない。またRelease authorization、Platform signing／submission、Store approval、public read-back、support開始、Product completionは異なる時点と失敗境界を持つ。

これらをProduct PlanのbooleanやLifecycle Acceptanceへ統合すると、required集合をDecision作成者が選べる、unsigned状態をPlatform authorityとして使える、Release Decisionが後段Platform Receiptをhash入力にして循環する、upload成功をpublicationと誤認する、partial publicationからsupport／completionを開始する問題が生じる。

## Decision drivers

1. Required universeをreceipt-free Product Definitionと決定論的Projectionだけから導出する。
2. Release authorizationへqualified authority、quorum、signature purpose、freshness、revocation、current stateを要求する。
3. Platform signing／submissionはauthorizationを消費するが、DecisionはPlatform Receiptを逆参照しない。
4. 実公開、partial failure、withdrawal、supersession、support開始、CompletionをRelease approvalと分離する。
5. 後段refを前段hash preimageへ埋め戻さない一方向DAGを維持する。

## Considered options

### A. Product PlanがRelease／Completion booleanを所有する

却下する。Product Planはintent、claim scope、requirement projection、pure predicateのOwnerであり、署名、authority state、Platform read-backまたはpublication stateを所有すると定義と実行結果が混在する。

### B. Product LifecycleへDecisionとPublicationを統合する

却下する。Lifecycle Acceptance自身をRelease authorizationの入力にしながらDecision、Platform Receipt、actual publicationまで同じrecordへ入れると、receipt-free Manifest／Acceptance、authority、外部副作用の境界とhash DAGが閉じない。

### C. Release authorization、Platform publication、Completionを一つの署名済みrecordにする

却下する。Decision時点ではPlatform public read-backが存在せず、Completion時点のEvidenceを前段へ含めると循環する。partial publication、withdrawal、supersessionも一つのstatusでは正しく表せない。

### D. Requirement、Decision、Platform Receipt、Publication、Completionを分離する

採用する。Product Planがreceipt-free required universe、Product Release Decisionがpre-publication authorization、各Platform Ownerがsigning／submission／read-back Receipt、Product Publication／Completionが後段集約とauthoritative Completionを所有する。

## Decision

1. `mirakan.arch.product-release-decision`を追加し、Release Decision Subject、qualified authority／quorum、purpose-separated signatures、freshness／revocation、current／superseded／revoked stateを所有させる。
2. `mirakan.arch.product-publication-completion`を追加し、required channel setの結果集約、`published | partially_published | withdrawn | superseded`、effective publication time、support開始binding、署名済みProduct Completionを所有させる。
3. Product PlanはActive Product Definition、Claim Scope、Release／Completion Requirement Projectionとpure predicateだけを所有する。required集合をDecisionまたはPublication作成者のinline入力にしない。
4. Platform Ownerはcurrent signed Product Release Decision Recordだけをsigning／upload／submission authorizationとして受理し、unsigned Subject、label、issue state、bare hashを拒否する。
5. Canonical順序を`Product Definition／Requirement Projection → Manifest／Engine Release／Acceptance → signed Release Decision → Platform Receipt → Product Publication → signed Completion`とする。
6. Publication RequirementはRelease Decision前のEvidence setへ混ぜず、Release Requirement Projection内の独立したrequired channel universeとして固定する。

## Consequences

- Architecture Owner文書は61件から63件になる。
- 新しい二Ownerの文書状態は`review`、実装状態は`absent`であり、Decision store、Authority、Key、Platform submission、Publication store、ReceiptまたはCompletionが実在する証拠ではない。
- Product Release Decision後でも全required channelのpublic read-backが揃うまではProduct-wide `published`、support開始、Completionを記録できない。
- Release approval、signing、upload、Store approval、public availability、Completionは別identity／別failureとして残る。
- PlatformからPublication／Completionへ逆依存せず、Publication OwnerがPlatform Receiptを下流集約する。

## Canonical Owner documents

- Required universe／pure predicate: [Product Plan](../00-product/product-plan.md)
- Release content／Lifecycle acceptance: [Product Lifecycle](../00-product/product-lifecycle.md)
- Security release closure: [Product Security](../01-governance/product-security.md)
- Signed release authorization: [Product Release Decision](../00-product/product-release-decision.md)
- Platform signing／submission／read-back: [Windows](../07-platform/windows.md)、[Android](../07-platform/android.md)、[Apple](../07-platform/apple.md)
- Product publication／support start／Completion: [Product Publication／Completion](../00-product/product-publication-completion.md)

## Supersedes／Superseded by

- Supersedes: none
- Superseded by: none

## Verification

- Owner header、相対link、Document ID、Architecture Index、Decision Logを照合する。
- 規範依存graphがDAGであり、後段RefがManifest、Acceptance、Decisionへ戻らないことを確認する。
- Required universeがProjection以外のinline入力を持たず、Release Decision前のEvidenceとpublication後のEvidenceが混在しないことを確認する。
- Windows、Android、Appleがexact signed Decision Record／current Stateを要求し、unsigned Subjectを拒否することを確認する。
- `published`、support開始、Completionが全required channelのpublic read-backを要求することを確認する。

## Official or primary sources

- none（Platform／Store外部仕様はCanonical Owner documentsへ委譲する）

このDecisionはOwnerとauthority境界だけを決める。実装、Authority assignment、Key provisioning、Platform submission、Publication、CompletionまたはCapability Activationを記録しない。
