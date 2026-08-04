# P0 Canonical Architecture／Legal-IP Governance Refresh Design

- Date: 2026-08-04
- Status: approved design
- Scope: Architecture Markdown only
- Implementation status: absent and unchanged
- User approval: recommended option approved on 2026-08-04

## 1. 結論

現Repositoryでは、参照ChatGPT conversationが未完成とした領域の多くに既存Ownerが存在する。Product DefinitionはProduct Plan、数値目標はPerformance／Capacity、公開C++はC++23／Native Game Module、共通State MachineはExecutable Contracts、Runtime lifetime／threadingはScheduling／Lifetime、Project failure atomicityはProject State、互換性はCompatibility／Evolution、EvidenceはAI Verification／Provenance、Product surface parityはProduct Lifecycle、AI client parityはAI Production Orchestrationが所有する。

実在するArchitecture gap／整合欠陥は次の三件である。

1. Active Product Definitionに対して、どのSubsystemをP0とし、各SubsystemがArchitecture契約軸を閉じたかを一つの決定論的基準で判定する正本がない。
2. 著作権、商標、特許／freedom-to-operate、第三者契約、AI生成物、製品表示、独立設計EvidenceをRelease法域へ束縛する一意Ownerがない。
3. Product surfaceがSDK、MCP、AI Agentを同じ`AI-MCP` bucketへcollapseし、generic transportの適合を特定Agent／versionへ代用できる。

採用設計は、[Product Plan](../../architecture/00-product/product-plan.md)へP0 membership／closure projectionを追加し、[Product Legal／IP Governance](../../architecture/01-governance/product-legal-ip-governance.md)を一件だけ新設する。さらにProduct surfaceを7種へ分離し、Lifecycle Client ProfileとAI Production Orchestration Agent Host Profileでversion別に対応付ける。Domain semanticsをP0表へ複写せず、各axisをexact Owner fragmentへ接続する。

本設計はC++、Schema file、Registry、Generator、Fixture、Receipt、Conformance executable、Build、CI、測定、法務判断、実装Task、実装順序、工程、工数または担当を作成しない。

## 2. 入力と監査方法

監査入力は次である。

- ChatGPT conversation `6a7099b0-c8f8-83e8-92fa-2d345df1db44`「AIネイティブゲームエンジン機能」。会話はuntrusted gap-discovery inputであり正本ではない。
- 変更前の2026-08-04 Architecture Indexにある65 Owner、Governance、Product Plan、Architecture Plan Closure Review、直近AI Production Orchestration変更。採用設計の反映後はProduct Legal／IP Governanceを加え66 Ownerとなる。
- RepositoryのHeader、文書状態、実装状態、規範依存、Owner scope、相対link、ADR規則。
- 著作権、特許、商標、AI生成物、AI規制、Cybersecurity product regulation、製品表示に関する公式一次資料。

判定語彙はArchitecture Governanceに従い、`closed-in-target-design`、`provisional`、`unmaterialized`、`implementation-absent`を区別する。文書が存在することを実装、測定、Qualification、法的適合またはReleaseへ読み替えない。

## 3. 参照会話の主張とRepository事実

| 会話が未完成とした領域 | Current canonical Owner | Repository判定 |
|---|---|---|
| Engine Product Definition | Product Plan | target designあり、materialized Active Definitionなし |
| 数値目標 | Performance／Capacityと各Platform／Domain Owner | provisional targetあり、測定Receiptなし |
| P0 Subsystem Owner | 各Ownerは存在するがP0 membership正本なし | valid gap |
| Public／Internal API | Core、C++23、Native Game Module、各Domain | target designあり、header／binary／ABI Evidenceなし |
| State Machine | Executable Contracts＋Domain Owner | target designあり、Schema／validatorなし |
| Lifetime／Threading／Invariant | Scheduling／Lifetime、Memory、ECS、各Domain | target designあり、runtime proofなし |
| Failure Atomicity | Project State、Core OperationTask、各Domain | target designあり、fault fixtureなし |
| Version／Migration／互換性 | Compatibility／Evolution＋Domain Owner | target designあり、migration executable／fixtureなし |
| Evidence Contract | AI Verification／Provenance＋Domain payload Owner | target designあり、signed Evidence artifactなし |
| GUI／CLI／SDK／MCP／Agent Conformance | Product Lifecycle＋AI Production Orchestration＋AI Verification | 部分契約はあるがSDK／MCP／Agentが旧`AI-MCP`へcollapseするvalid整合欠陥。materialized suiteなし |
| 実装、性能、制作体験、商用運用 | 該当Owner／Product Release | 未実証。Architecture変更だけでは閉じない |
| 法令／知財／非模倣 | 分散した一部規則のみ | valid gap |

このため、既存Ownerを全面改稿せず、P0 rootとLegal／IP authorityを追加する。

## 4. 比較した方式

### 4.1 既存文書へ横断表だけ追加する

不採用。変更量は小さいが、P0 membershipとLegal／IP判断のcanonical ownerが不在のままで、表が第二の意味定義または非規範checklistになる。

### 4.2 Product PlanへP0 rootを追加し、Legal／IP Ownerを一件追加する

採用。Product Planは既にProduct Definition、Capability tier、required Operation universe、release／completion projectionを所有するため、P0 membershipとArchitecture closure projectionの自然なOwnerである。法令／知財はProduct定義とは異なる判断権限を持つためGovernance Ownerへ分離する。

### 4.3 P0とLegal／IPを二つの新Ownerへ分離する

不採用。P0 membershipをProduct Plan以外へ置くとActive Product Definitionと二重正本になり、Product変更時の不一致を増やす。Legal／IPだけを新Ownerにする。

## 5. P0の意味

`P0`はPriority Zero Architecture Scopeであり、実装Phase、日程、工程、Work Package、担当、重要度ラベルまたはCapability activation stateではない。

P0集合は次のcanonical unionとする。

1. Product Definition、Governance、Foundation、Authoring、Runtime、Releaseを成立させる固定root Owner集合。
2. Active Product Definitionが選ぶCapability、Host、runtime Target、locale、Reference dimension、Pack requirementから決定論的に投影したOwner集合。
3. First Playable DefinitionとMinimum Executable Coreが追加で要求するOwner集合。
4. Product Lifecycle、Security、Privacy、Legal／IP、Evidence、Compatibility、Performance、Conformanceを閉じる横断Owner集合。

Future portfolioだけに属するOnline、large world、advanced light transport、virtualized geometry、Runtime generation等は、Active Product DefinitionへPromotionされるまでP0へ含めない。Owner文書が存在することだけでP0 membershipを得ない。

## 6. P0 Canonical Architecture Specification

Product Planは次のtarget型意味を所有する。

```text
P0CanonicalArchitectureSpecificationV1
  specification_id
  specification_version
  product_definition_ref
  first_playable_definition_ref
  minimum_executable_core_definition_ref
  fixed_root_subsystem_refs[]
  derived_subsystem_refs[]
  subsystem_closure_refs[]
  excluded_future_subject_refs[]
  specification_content_hash

P0SubsystemArchitectureClosureV1
  subsystem_id
  canonical_subsystem_owner_document_ref
  membership_basis
  axis_bindings[16..16]
  closure_state
  unresolved_diagnostic_refs[]
  closure_content_hash

P0ArchitectureAxisBindingV1
  axis_kind
  semantic_owner_document_ref
  canonical_owner_fragment_ref
  shared_contract_refs[]
  applicability
  current_document_state
  current_implementation_state
  current_verification_state
  blocking_evidence_refs[]
```

16 axisはclosed enumである。

1. `canonical_state`
2. `public_api`
3. `internal_api`
4. `state_machine`
5. `lifetime`
6. `threading`
7. `invariant`
8. `failure_atomicity`
9. `versioning`
10. `migration`
11. `compatibility`
12. `security_boundary`
13. `evidence_contract`
14. `acceptance_test`
15. `surface_conformance`
16. `performance_capacity`

各axisはexactly one semantic Ownerと一つのcanonical fragmentを持つ。共通Envelope、encoding、Evidence spine等は`shared_contract_refs[]`へ置き、Ownerを複数化しない。`not_applicable`は空欄ではなく、Owner、理由、適用境界、再評価条件を持つclosed branchとする。

`target_design_closed`は全P0 member、全16 axis、全fragment、規範依存、状態表現が閉じたことだけを示す。Schema、C++、Fixture、Conformance Suite、Receipt、測定または運用Evidenceがない場合は、対応する実装／検証状態を`absent`／`unverified`のまま保持する。Architecture closureをProduct Release Evidenceへ数えない。

## 7. Product Definitionと数値目標

Product Planの既存Active Product Definition、First Playable、2D／3D non-substitution、C0–C3、Future boundaryを維持する。P0は別Product scopeを作らず、同じDefinitionを参照する。

数値はPerformance／Capacityと各Domain／Platform Ownerを一意なOwnerとする。P0 rowはBudget fragmentとEvidence stateだけを参照し、値を複写しない。未計測値は`provisional`であり、測定環境、workload、sample、percentile、warm／cold条件、threshold、Receiptが揃うまで`measured`またはProduct claimにしない。

## 8. API、State、Failure、Evolution

P0 closureは次の分担を強制する。

- Public C++はC++23／Native Game Module、canonical OperationはExecutable Contracts、Project mutationはProject State、Domain APIはDomain Owner。
- Internal APIはCoreのLayer／Host／Gateway境界と各Domain private boundaryへ閉じ、AI、Plugin、Project SDKまたはMCPへ漏らさない。
- 共通State Machine表現はExecutable Contracts、Domain transitionは各Subsystem Owner。
- Runtime lifetime／threadingはScheduling／Lifetime、memory／borrow／handleはMemory、Domain追加制約は各Subsystem Owner。
- Project mutationは全成功または公開変更0件。外部不可逆effectは事前承認とcompensation／recoveryを要求し、偽のundoを定義しない。
- Compatibility class、clean break、reader／writer、migration admissibilityはCompatibility／Evolution、Domain変換意味はDomain Owner。

## 9. Cross-surface Conformance

Product Lifecycleの`editor_gui | cli | headless | native_sdk | external_ide | mcp | ai_automation` journey parityと、AI Production Orchestrationの`editor_builtin | native_sdk | cli | mcp | first_party_agent_host | external_agent_host` parityを別型のまま維持する。Product側の`ai_automation`はAI routeの総称であり、個別のAI Host適合性を隠さない。Lifecycleが所有するversioned cross-surface bindingで、Product surface、AI surface、transport、Agent Host Profileをexactに対応付ける。

P0 conformance rootは両集合を明示mappingし、少なくとも次を同じRequirement、Operation、Project revision、Authorization、Scenario、expected branchへ束縛する。

- Editor GUI manual journey
- CLI interactive journey
- headless deterministic journey
- Native SDK direct client
- MCP protocol Adapter
- built-in AI Console
- first-party Agent Host
- 各qualified external Agent Host profile

vendor名はcanonical enumにしない。Codex、Claude等はversioned external Agent Host Profileとして登録し、同じsuiteを個別に通過した時だけ当該Profileをsupportedと表示する。Agent prose、token、Tool call bytes、latencyの一致は要求せず、semantic request、visible Operation set、authorization、state transition、failure、diagnostic、result hash、commit behaviorの同値性を要求する。

Suite／Fixture／Receiptのgeneric identityとfreshnessはAI Verification、Product journey membershipはProduct Lifecycle、AI route semanticsはAI Production Orchestration、Project hash／CommitはProject Stateが所有する。

## 10. Product Legal／IP Governance

新Ownerは法令解釈そのものをEngine codeへ埋め込まず、Releaseごとの適用範囲とauthorized legal reviewを閉じる。

所有対象は次である。

- Jurisdiction／market／distribution channel／product role／effective-dateを持つLegal Applicability Profile。
- copyright、trademark／trade dress、patent／FTO、trade secret、dependency／content license、AI Provider terms、AI output rights、consumer／advertising claim、privacy、security product obligation、accessibility、Store／Platform contractのRequirement binding。
- Engine source、API、UI、naming、documentation、sample、asset、generated source／contentのIndependent Design／rights review subject。
- required legal Evidence set、unresolved issue、authorized human reviewer、freshness、revocationを持つfail-closed Legal／IP Readiness Decision。

Domain evidence ownershipは移動しない。

| Evidence | Canonical producer |
|---|---|
| dependency version、license text、SBOM、NOTICE source | Toolchain／Dependencies |
| Asset／AI生成Assetのorigin、terms、rights、commercial review | Asset Lifecycle |
| Project source revision、生成差分、Code Owner | Project State／Native Game Module／Project Shader |
| first-party license、redistribution、container別NOTICE | Product Lifecycle |
| data flow、consent、processor、region、retention | Product Privacy |
| vulnerability、secure update、incident | Product Security |
| Product claimとsupport scope | Product Plan／Release／Publication |
| signed Evidence envelope、freshness、revocation | AI Verification／Provenance |

独立作成はcopyright copying riskを下げるEvidenceになり得るが、patent FTO、trademark clearance、contract complianceまたは法的適合の自動証明ではない。特許はcopyingなしでも問題になり得て地域差があるため、対象法域ごとの専門家判断を要求する。AI自身、生成Provider、similarity score、source hash、OSS scannerまたはArchitecture文書はLegal approverになれない。

## 11. Independent-design boundary

外部Engine比較で許可する入力は、公開された公式documentation、公開standard、公開release notes、合法的に取得した製品の通常利用観察に限定する。比較目的はUser journey、failure mode、accessibility、distribution、support gapの発見である。

次を禁止する。

- 外部Engineのsource、decompiled output、leak、NDA／confidential material、取得条件に反する内部Artifactの利用。
- API identifier、type hierarchy、Scene／Project format、UI layout、icon、sample、template、default、workflow、creative expressionの逐語的または一対一の移植。
- 外部Engine互換を初期V1の近道にするalias、reader、importer、migration layer。
- 「一般的な機能」「独立作成」「AI生成」「少し変更した」を権利確認の代用にすること。

採択する設計はMiraikanai側のUser requirement、Owner、state、failure、security、Evidenceから独立に導出し、比較sourceと採択理由を分離して記録する。

## 12. 法域バインド

初回配布地域が未指定であるため、特定国を暗黙defaultにしない。Release候補は対象jurisdiction、market、channel、Product／AI roleをexact profileとして宣言し、そのscopeに対するcurrent legal source snapshotと人間のLegal／IP Decisionを要求する。

Japan、United States、EU／EEA等は異なるRequirement集合になり得る。同じlocale、Store、Host Target、通貨、Company所在地またはWebsite到達性からjurisdictionを推測しない。配布地域追加、法令／Platform terms改訂、Provider role変更、claim変更、権利失効または重大な法的findingではDecisionをstale／revokedにし再審査する。

## 13. 正本反映対象

### 新規

- `docs/architecture/01-governance/product-legal-ip-governance.md`
- `docs/architecture/decisions/2026-08-04-p0-architecture-and-legal-ip-ownership.md`
- 本design spec

### 更新

- Product Plan: P0 meaning、type、membership、16-axis closure、current evidence boundary、Legal／IP link。
- Product Lifecycle: Manifest／Distribution Coverage／license・NOTICE EvidenceのLegal review入力、P0 cross-surface conformance mapping。Legal DecisionはLifecycle Acceptanceへ埋め込まない。
- Product Release Decision: same-scope fresh Legal／IP readinessをrelease subjectへ追加。
- Product Publication／Completion: publication market setとLegal profileのset equality、read-time freshness。
- Architecture Governance: new Owner navigationとP0 state wording。
- AI Verification／Provenance: legal Evidenceのgeneric envelope／freshness consumer boundary。
- Product Privacy／Security、Toolchain、Asset、Naming: Legal Ownerへ渡すdomain evidence boundary。
- AI Production Orchestration: Product conformance mapping consumer boundary。
- Architecture Index、Closure Review、review summary: Owner count、gap IDs、監査結果。

## 14. Current empty state

この刷新後も次は`absent`である。

- `P0CanonicalArchitectureSpecificationV1` Schema、Registry、generator、resolver、closure Artifact。
- `ProductLegalApplicabilityProfileV1`、Requirement Registry、Legal／IP Decision artifact、signing key、legal operations。
- GUI／CLI／SDK／MCP／Agent Conformance Suite、Fixture、runner、Receipt。
- Engine／Editor／SDK／Runtime implementation、Build、CI、benchmark、Reference Project、UX study、support／incident運用Evidence。
- 実際の法務意見、商標search、patent FTO、license approval、Release authorization。

未materializeの型名または文書closureを実装、法令遵守、非侵害、性能達成、制作体験成立、商用運用成立へ昇格しない。

## 15. 一次資料と採用境界

外部資料は法的論点と製品義務が法域／役割／scopeで変わる事実の確認に使う。P0構造、新Owner、型、Gate、独立設計policyはMiraikanaiのproject-decisionであり、外部組織の公式推奨ではない。

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

## 16. Design完了条件

- P0が工程ではなくProduct-derived C0 Architecture scopeとして一意に定義される。
- 全P0 Subsystemがexactly one canonical Ownerを持つ。
- 16 axisにmissing、duplicate Ownerまたは理由なし`not_applicable`がない。
- Cross-surface conformanceがGUI、CLI、headless、Native SDK、MCP、built-in／first-party／external Agentを非代替でcoverする。
- Legal／IP Ownerが法域、rights、independent design、FTO／trademark、人間Decisionを閉じる。
- Lifecycle artifactsからLegal reviewへ一方向に入力し、Release／Publicationが同じProduct、Candidate、market、channel、claim scopeへLegal Decisionを束縛する。
- Domain evidence ownership、Evidence envelope ownership、Product authorityを移動または複製しない。
- Future subjectをP0またはcurrent Capabilityへ暗黙昇格しない。
- 実装、実装計画、法務判断、測定、QualificationまたはReleaseを作成済みと表現しない。
