# P0 Canonical Architecture／Product Legal-IP Ownership

- 文書ID: mirakan.decision.p0-architecture-and-legal-ip-ownership
- 状態: review
- 正本範囲: P0 Architecture membership／closureをProduct Planへ置く判断、Product Legal／IP Governance Owner追加、既存Owner分散またはP0専用Ownerを採用しない理由
- 非正本範囲: current P0 Registry／Schema／closure Artifact、個別Subsystem semantics、法律解釈／法務意見、Legal Decision、Conformance Suite、実装、測定、Qualification、実装Task／順序／工程／工数／担当。current契約は各Ownerを参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Product Plan](../00-product/product-plan.md)、[Product Legal／IP Governance](../01-governance/product-legal-ip-governance.md)、[Product Lifecycle](../00-product/product-lifecycle.md)、[Product Release Decision](../00-product/product-release-decision.md)、[Product Publication／Completion](../00-product/product-publication-completion.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Project State](../03-authoring/project-state.md)、[AI Production Orchestration](../03-authoring/ai-production-orchestration.md)、[Developer Testing](../03-authoring/developer-testing.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)
- 外部根拠検証日: 2026-08-04
- 文書種別: Architecture Decision／product-scope closure and governance ownership
- Decision owner document: `mirakan.arch.architecture-governance`
- Decision日: 2026-08-04
- Supersedes: none

## Context

参照ChatGPT conversationは、最高水準のEngineへ進む前にP0 SubsystemごとのOwner、正本状態、Public／Internal API、State Machine、Lifetime、Threading、Invariant、Failure Atomicity、Version、Migration、Security Boundary、Evidence、Acceptance Test、cross-surface Conformanceを閉じる必要があると指摘した。

Current Repositoryには各論点のOwnerが広く存在するが、次の二点は閉じていなかった。

1. Active Product Definitionに必要なSubsystem集合をP0として一意に選び、共通axisのmissing／duplicate Ownerを判定するcanonical root。
2. copyright、trademark、patent FTO、trade secret、third-party／AI terms、independent design、Product claimをRelease法域へ束縛するcanonical Legal／IP authority。

前者を非規範checklistへ置くとProduct変更に追随せず、後者をToolchain、Asset、Privacy、Security、Lifecycleへ分散すると誰が最終Legal Decisionを所有するか不明になる。

## Decision drivers

1. P0を実装Phase、日程、Work PackageまたはActivation stateにしない。
2. Active Product Definitionと別のSubsystem membership正本を作らない。
3. 各Subsystem／axisの意味を既存OwnerからP0表へ複写しない。
4. Product Definition、数値Budget、API、state、failure、migration、Evidence、Conformanceの既存Ownerを維持する。
5. 法的適用をlocale、Target、Store、ProviderまたはAIから推測しない。
6. Domain Evidenceとauthorized Legal／IP Decisionを分離する。
7. Independent designをcopyright、trademark、patent FTO、licenseまたは法令適合の万能証明にしない。
8. 文書変更から実装、測定、UX、商用運用または法令遵守を主張しない。

## Considered options

### A. 既存OwnerとClosure Reviewへ横断表だけ追加する

却下する。P0 membershipが非規範の手動表になり、Legal／IPの最終authorityは分散したままになる。表の更新漏れをProduct Definitionから検出できない。

### B. P0とLegal／IPの専用Ownerを二件追加する

却下する。P0専用OwnerがActive Product DefinitionとSubsystem membershipを二重所有する。Product scope変更時にProduct PlanとP0 Ownerのどちらが正本か曖昧になる。

### C. Product PlanへP0 rootを追加し、Legal／IP Ownerだけを新設する

採用する。Product PlanはProduct scope、Capability tier、required Operation／journey、Release／Completion requirement projectionを既に所有するため、P0 membershipとclosure projectionを同じProduct Definitionから導出できる。Legal／IPはProduct scopeとは異なるhuman authorityと法域bound reviewを持つためGovernance Ownerへ分離する。

### D. Legal／IPをProduct SecurityまたはProduct Lifecycleへ統合する

却下する。Security threat／vulnerability authorityまたはdistribution／support compositionへcopyright、trademark、patent、contract、independent designの判断を混ぜ、Domain EvidenceとLegal Decisionの責務が過大になる。

## Decision

1. [Product Plan](../00-product/product-plan.md)を`P0CanonicalArchitectureSpecificationV1`、`P0SubsystemArchitectureClosureV1`、P0 membership／closure projectionの一意Ownerとする。
2. P0はPriority Zero Architecture Scopeであり、Active Product Definition、First Playable Definition、Minimum Executable Coreおよび横断Product requirementから決定論的に導出する。実装順、工程、工数または担当を表さない。
3. 各P0 Subsystemはexactly one canonical Ownerを持ち、`canonical_state | public_api | internal_api | state_machine | lifetime | threading | invariant | failure_atomicity | versioning | migration | compatibility | security_boundary | evidence_contract | acceptance_test | surface_conformance | performance_capacity`の16 axisをexactly one semantic Owner fragmentへ束縛する。
4. shared contractはaxisの`shared_contract_refs[]`として参照し、semantic Ownerを増やさない。`not_applicable`はOwner、理由、境界、再評価条件を必須にする。
5. P0 `target_design_closed`はArchitectureの参照／意味が閉じたことだけを示す。実装、Schema、Fixture、Suite、Receipt、測定、Qualification、Activation、ReleaseまたはProduct completionを意味しない。
6. `mirakan.arch.product-legal-ip-governance`をGovernance Ownerとして追加する。
7. 新OwnerはLegal Authority Source Snapshot、Jurisdiction Profile、Product Legal Applicability Profile、Legal Requirement Binding、Independent Design Review Subject、Legal／IP Review Subject、signed Legal／IP Decisionとeffective stateを所有する。
8. Legal categoryはcopyright、trademark／trade dress、patent FTO、trade secret、contributor chain-of-title、dependency／content license、AI Provider／output terms、Product license、advertising claim、privacy、cybersecurity product obligation、accessibility、Platform／Store terms、export／sanctions、consumer／commerce／taxを非代替に扱う。
9. Toolchain、Asset、Project Source、Privacy、Security、Lifecycle、Product claimおよびEvidence OwnerはDomain payloadを保持する。Legal Ownerはsame Product／Candidate／market／channel／subjectへ集約し、authorized human reviewを発行する。
10. 初回配布地域を暗黙固定せず、各Releaseがjurisdiction、market、channel、Product／AI roleをexact Profileへ束縛する。
11. `approved`は全required Evidence、全category mapping、unresolved issue exact empty、human reviewer署名を必要とする。条件付き承認を保存せず、未解決条件はRequirementとして残す。
12. external Engine比較は公開一次資料と正当な通常利用によるgap discoveryに限定し、API／type／Scene／Project／UI／Asset／sample／workflow／default／creative expressionのcopy、thin rewrite、一対一alias、license違反／confidential materialを禁止する。
13. 独立作成、AI生成、copyright review、OSS scan、SBOM、similarity scoreまたは署名からpatent FTO、trademark clearance、rights approvalまたは法令適合を推論しない。
14. Product LifecycleはManifest／Distribution Coverage／license・NOTICE EvidenceをLegal reviewへ一方向に供給する。Release DecisionがLifecycle Acceptanceとsame-scope fresh Legal／IP Decisionを合流させ、Publication／Completionが同Decisionを再検証する。partial market approvalをProduct-wide approvalにしない。
15. current RepositoryではP0／Legal Schema、Registry、Generator、Decision、Suite、Fixture、Receipt、implementationおよびlegal opinionを`absent`のまま保持する。

## Consequences

- Product scope変更からP0 Owner集合と16-axis closureを一方向に再計算できる。
- 既存Ownerを維持しながら、missing axis、duplicate Owner、理由なしN/A、Future混入を一つのrootで検出できる。
- GUI、CLI、headless、Native SDK、MCP、built-in／first-party／external AgentのConformanceを同じProduct requirementへ結べる。
- 法的reviewの対象scope、時点、法域、source、Evidence、reviewer、Decision expiry／revocationが明示される。
- 特定国の法律をArchitectureへ固定せず、Releaseごとの法域差に対応できる。
- 新しいP0／Legal types、Registry、resolver、operations、suite、evidence store、review processを将来materializeする必要が残る。本Decisionはその実装または実装計画を定めない。
- Legal／IP Decisionは法的保証ではなく、authorized reviewのscope-bound recordである。必要な専門家判断と運用体制はRepository外にも残る。

## Canonical Owner documents

- Product scope、P0 membership／closure: [Product Plan](../00-product/product-plan.md)
- Legal applicability、Independent Design、Legal／IP Decision: [Product Legal／IP Governance](../01-governance/product-legal-ip-governance.md)
- Architecture state、Owner uniqueness、dependency／ADR: [Architecture Governance](../01-governance/architecture-governance.md)
- Product journey／surface parity／distribution: [Product Lifecycle](../00-product/product-lifecycle.md)
- Release authority: [Product Release Decision](../00-product/product-release-decision.md)
- actual publication／completion: [Product Publication／Completion](../00-product/product-publication-completion.md)
- Evidence envelope／freshness／revocation: [AI Verification／Provenance](../01-governance/ai-verification-provenance.md)
- compatibility／migration class: [Compatibility／Evolution](../02-foundation/compatibility-evolution.md)
- Operation／State Machine／Diagnostic projection: [Executable Contracts](../02-foundation/executable-contracts.md)
- Project atomicity: [Project State](../03-authoring/project-state.md)
- AI client／Agent surface parity: [AI Production Orchestration](../03-authoring/ai-production-orchestration.md)
- Project test semantics: [Developer Testing](../03-authoring/developer-testing.md)
- Runtime lifetime／threading: [Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)
- budgets／measurement: [Performance／Capacity](../04-runtime/performance-capacity.md)

## Supersedes／Superseded by

- Supersedes: none
- Superseded by: none

## Official or primary sources

外部資料は法的subjectが法域、role、scope、timeで変わる事実の確認に使う。本DecisionのOwner配置、P0 axis、fail-closed Gate、independent-design policyはMiraikanaiのproject-decisionであり、外部組織の公式推奨ではない。

- [WIPO Copyright Protection of Computer Software](https://www.wipo.int/en/web/copyright/activities/software)
- [WIPO Patent law](https://www.wipo.int/en/web/patents/law)
- [WIPO Trademark protection](https://www.wipo.int/en/web/trademarks/protection)
- [文化庁 AIと著作権について](https://www.bunka.go.jp/seisaku/chosakuken/aiandcopyright.html)
- [U.S. Copyright Office Copyright and Artificial Intelligence](https://www.copyright.gov/ai/)
- [Regulation (EU) 2024/1689, Artificial Intelligence Act](https://eur-lex.europa.eu/eli/reg/2024/1689/oj?locale=en)
- [Regulation (EU) 2024/2847, Cyber Resilience Act](https://eur-lex.europa.eu/eli/reg/2024/2847/oj?locale=en)
- [U.S. FTC Advertising and Marketing](https://www.ftc.gov/business-guidance/advertising-marketing)

このDecisionはOwner配置と採用理由だけを記録する。current Product、P0、Legal／IP、Release、Evidence、Conformanceおよびfailure semanticsは各Owner文書を正本とし、実装計画または法務意見を定めない。
