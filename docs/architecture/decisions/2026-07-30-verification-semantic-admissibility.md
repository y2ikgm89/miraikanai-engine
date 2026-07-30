# Verification Semantic Admissibility

- 文書ID: mirakan.decision.verification-semantic-admissibility
- 状態: review
- 正本範囲: generic Evidence／Qualification Receiptへscope、subject contract、branchを保持し、Ref解決後の意味制約をset比較前に評価する判断
- 非正本範囲: current Schema／Ref tuple／predicate、Domain固有Evidence payload、Requirement Set、Acceptance、Release Decision。各Owner文書を参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Product Plan](../00-product/product-plan.md)、[Product Lifecycle](../00-product/product-lifecycle.md)、[Executable Contracts](../02-foundation/executable-contracts.md)
- 外部根拠検証日: none
- 文書種別: Architecture Decision／cross-owner verification identity
- Decision owner document: `mirakan.arch.architecture-governance`
- Decision日: 2026-07-30
- Supersedes: none

## Context

Exact Ref、content hash、set equalityだけでは、同じinvalid tupleをrequired側とsupplied側へ複写した場合に意味的な不整合を検出できない。Qualification ScenarioはHost／Target／locale／Reference dimension applicabilityとexpected branchを、Evidence Classはrecord kind／purpose／subject contractを持つが、generic Receiptがそれらを運ばなければconsumerは制約をread-backできない。

Domainごとに異なるlocal generic Refを追加すると共通Evidence spineが再び分岐する。一方、Target／locale等を持たないEvidenceも存在するため、nullable contextまたは暗黙既定値ではscopeの意味を区別できない。

## Decision drivers

1. `not_applicable`、independent、exact scalar scopeを相互aliasにしない。
2. Evidence Class、Scenario、owner Typeの意味制約をgeneric wrapperから検証可能にする。
3. owner-specific completed recordをauthorityとしてread-backし、hash-only成功を禁止する。
4. 全Domainへ固定payloadを複写せず、共通最小scopeとDomain固有subjectを分離する。

## Considered options

### A. Content hashとowner record hashだけを検証する

却下する。scope、subject contract、Scenario branchがgeneric identityに現れず、cross-target／locale／dimension substitutionまたはinvalid tupleの両辺複写を判定できない。

### B. 各Domainが独自のgeneric Evidence Refを定義する

却下する。Evidence class／purpose／scopeのResolverが分岐し、consumer-local narrow Refと二重authorityを再導入する。

### C. 共通scope vector／subject contract／branchとsemantic predicateをAI Verification Ownerへ置く

採用する。四scopeのclosed branchを共通化し、Domain固有contextはowner Type／subject hashに保持したまま、全consumerが同じadmissibility判定を再実行できる。

## Decision

1. AI Verification Ownerは`VerificationScopeVectorV1`、generic Evidence／Qualification Receiptの`subject_contract_refs[]`、Qualification `observed_result_branch`を所有する。
2. Evidence ClassとVerification Record Type Registryのrequired subject contract canonical unionをgeneric wrapperとexact set equalityにする。
3. QualificationのscopeはScenario applicability、observed branchはScenario expected branchのexact memberでなければならない。
4. generic Ref、wrapper、owner-specific completed record、signed recordを解決し、scope／subject contract／subject hash／branchをread-backしてからRequirement Set、Acceptance、Release、Publication、Completionのset比較へ参加させる。
5. Generic Operation Receiptはsubject contract集合とrequest hashをRef identityへ追加し、Domain固有full contextをowner payload validatorでread-backする。
6. initial V1へ旧Ref alias、scope-less Receipt reader、hash-only fallbackまたはmigrationを設けない。

## Consequences

- 共通Qualification ReceiptはScenario applicabilityを機械的に検証できる。
- Scopeを持たないEvidenceも四軸をexplicit `not_applicable | independent` branchで表し、nullを使わない。
- Domain consumerは意味制約を再定義せず、owner-specific subjectの追加contextだけを検証する。
- 現RepositoryにはSchema、Registry、Predicate実装、Evidence、Receipt、resolverまたはsigned recordはmaterializeしていない。

## Canonical Owner documents

- Verification scope／Evidence／Qualification spine／semantic predicate: [AI Verification／Provenance](../01-governance/ai-verification-provenance.md)
- Generic Operation Receipt: [Executable Contracts](../02-foundation/executable-contracts.md)
- Product Journey requirement universe: [Product Plan](../00-product/product-plan.md)
- Product Qualification consumption／Acceptance: [Product Lifecycle](../00-product/product-lifecycle.md)
- Release authority: [Product Release Decision](../00-product/product-release-decision.md)
- Publication／Completion: [Product Publication／Completion](../00-product/product-publication-completion.md)

## Supersedes／Superseded by

- Supersedes: none
- Superseded by: none

## Verification

- `VerificationScopeVectorV1`、Evidence／Qualification generic Ref、Operation Receipt generic Refに一意Ownerがある。
- Scenario exact-set／independent／not-applicableとReceipt scalar scopeのadmissibilityがclosed predicateで定義される。
- Evidence Class／Type Registry／wrapperのsubject contract集合がexact unionで閉じる。
- Lifecycle、Security、Privacy、Pack、Release、Publication／Completionがset比較前にpredicateまたはowner read-backを要求する。

## Official or primary sources

- none（project-decisionのみ）

このDecisionはtarget Owner／identity／validation境界だけを決める。実装、Schema、Registry、Evidence、Receipt、QualificationまたはReleaseが存在する証拠ではない。
