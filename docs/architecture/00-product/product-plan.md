# Miraikanai Engine Product Plan

- 文書ID: mirakan.arch.product-plan
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Product intent、AI-native C++ Product claim、AI game generation claim lane、第三者向け汎用Engine minimum surface、非交渉原則、Capability成熟度、P0 Architecture scope／fixed root membership／Product-derived membership／16-axis closure projection、2D／3D Product portfolio、MVP outcome、FirstPlayableDefinitionV1、Product Host／runtime Target／locale Profile identityとroot Registry、Active Product Definition、claim-kind minimum、製品claim、release／completion requirement projection、claim-derived required Operation universe、required Operation journeyとsurface applicability projection、昇格・停止・pure release／completion predicate
- 非正本範囲: P0 Subsystem各axisのdomain semantics、最終Release authority record、Publication／Completion authority record、実装順序、工程、工数、担当、Fixture件数、実行／materialization Registry、Target technical Toolchain binding、Localization Catalog／fallback、Subsystemの型・Field・API・Backend・既定値・Budget、AI権限、各domain Evidence／Receiptの形式と意味、法令解釈／Legal Decision。各Owner文書または非規範proposalを参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)
- 関連文書: [Product Lifecycle](product-lifecycle.md)、[Product Release Decision](product-release-decision.md)、[Product Publication／Completion](product-publication-completion.md)、[Product Legal／IP Governance](../01-governance/product-legal-ip-governance.md)、[AI Production Orchestration](../03-authoring/ai-production-orchestration.md)、[Game Production Loop](../03-authoring/game-production-loop.md)、[Product Privacy／Data Governance](../01-governance/product-privacy-data-governance.md)、[Product Security](../01-governance/product-security.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[AI-native C++ Product Identity Decision](../decisions/2026-08-03-ai-native-cpp-product-identity.md)、[AI Production Orchestration Ownership Decision](../decisions/2026-08-04-ai-production-orchestration-ownership.md)、[P0 Architecture／Legal-IP Ownership Decision](../decisions/2026-08-04-p0-architecture-and-legal-ip-ownership.md)、[Android Adaptive Game Window Baseline Decision](../decisions/2026-08-03-android-adaptive-game-window-baseline.md)、[Product Execution Registry Proposal](../appendices/product-execution-registry-proposal.md)、[Architecture Plan Closure Review](../appendices/architecture-plan-closure-review.md)、[Advanced Rendering／Multiplayer Ownership Decision](../decisions/2026-07-29-advanced-rendering-multiplayer-ownership.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[Developer Testing](../03-authoring/developer-testing.md)、[Runtime Performance／Capacity](../04-runtime/performance-capacity.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-08-03

## 1. Product intent

Miraikanai Engineは、2Dと3Dを同格のfirst-class capabilityとして扱い、独自Editor、公開C++ SDK、build／package経路、診断、更新、文書、supportを一つの製品releaseへ閉じる、第三者Developer向けGame Engineである。既存EngineへChat機能を付加する製品ではない。人間とAIが同じ型付きAuthoring経路を使い、C++ Engineの信頼済みGatewayだけが検証済み変更をProjectへ確定する。

対象Userは初心者からC++を扱う上級者までである。両者は互換性のない別Project形式を使わず、同じ正規Project stateを異なるWorkspaceとsurfaceから扱う。Editor GUI、CLI、headless runner、Native SDK、external IDE、MCP、AI automationはそれぞれ別authorityを持たず、同じProject revision、public contract、policy、diagnostic、evidenceへ収束する。

第三者Developerが完結できる標準journeyは次である。各段階の意味と型は該当Ownerが所有し、本書はProduct outcomeだけを所有する。

1. 対応HostへEngine、Editor、SDK、Template、Documentationを取得し、license／support条件を確認する。
2. Project Browserから作成または既存Projectを開き、Version Control管理下の正規Sourceを壊さず復旧できる。
3. 2Dまたは3DのAsset、World、Gameplay、UI、Input、AudioをEditorまたは公開APIで制作する。
4. Project C++、Project Shader、構造化dataを公開契約の範囲で編集し、外部IDEからbuild／diagnosticへ到達する。
5. Project固有testをGUI、CLIまたはheadlessで実行し、debug、profile、crash evidenceを同じrevisionへ関連付ける。
6. Targetごとにbuild、cook、package、clean install、launch、offline playを検証する。
7. Packとdependencyのsource、publisher、license、trustを確認して導入、更新、削除する。
8. Privacy disclosure、telemetry／crash consent、NOTICE、redistribution条件を確認して製品を配布する。
9. Engine／Projectを更新、repairまたはrollbackし、既知制限、security update、support終了へ追随する。

上記の一段でもrelease固有Evidenceがない場合、Architecture記載、Demo動画、Editor表示またはbuild成功だけから「第三者へ提供可能」と主張しない。

標準journeyを第三者Developerが完結できることは、Engine自身のProduct operational state、Architecture baseline、R4 Engine core／security変更、R5 release／publicationを一人の主体が自己認証できることを意味しない。Project authoring、候補生成、local build／testは[AI Security／Approval](../01-governance/ai-security-approval.md)のRisk別規則に従い、独立性を要求しないR2／R3 edgeへ架空の第二人員を要求しない。一方、Ownerが独立承認を要求するProduct Decision、Engine R4、release／publicationは独立主体または分離Pipelineが成立するまで未承認に留め、同一人物の複数Role、複数Key、AI判断またはpublisher署名を別主体として数えない。単独作業者は候補とEvidenceを作成できるが、独立性が必要なEngine production／release claimを自己認証できることはProduct requirementではない。

根拠: official-spec／project-decision — [NIST SP 800-53 AC-5 Separation of Duties](https://doi.org/10.6028/NIST.SP.800-53r5)は権限濫用riskを下げるため分離対象職務とaccess authorizationの明確化を求める。どのMiraikanai edgeをR2／R3／R4／R5へ分類するか、独立性を要求する範囲およびProduct journeyとの分離は本プロジェクトの判断である。

### 1.1 AI-native C++ Product identity

`AI-native`は外部標準名ではなくMiraikanaiのproject-decisionであり、「Chat画面がある」「AI Providerへ接続できる」「AIがC++ textを生成できる」の同義語ではない。MiraikanaiがAI-native C++ Game Engineを名乗れるのは、次の条件が同じProject revisionと公開契約へ閉じる場合だけである。

1. 人間、AI、Editor、CLI、IDEは同じcanonical Project stateを読み、同じtyped Operation／ChangeSet境界へproposalを作る。AI専用のhidden writer、private Engine APIまたはProvider Schema由来authorityを持たない。
2. AIが保持する文脈はTask authorization、Project revision、対象Owner、read set、policy、redaction、期限を明示したbounded contextであり、Chat履歴、Model memory、Editor selectionまたはRuntime live stateを暗黙の正本にしない。
3. `read／query／explain`と`propose／validate／approve／commit`を分離する。mutationはsemantic diff、Validation、必要Approval、atomic Commitを経由し、C++ Engineだけが現在revision、Capability、policy、contract、preconditionを権威判定する。
4. AIが生成したC++、Shaderまたは構造化dataも通常のProject Sourceである。同じOwner、format、review、build、test、diagnostic、provenance、license、security、rollback条件を適用し、生成元を理由にGateを迂回しない。
5. proposal、Validation、Commit、test／build、結果、拒否理由は同じrequest／revision／content identityへ関連付け、説明とundo／recoveryに必要なbounded Receiptを残す。ただしReceipt Schemaと意味は各Ownerが所有する。
6. AI Provider、MCP endpointまたはそれらへのnetwork pathが不在、失効、拒否、timeout、非互換であっても、install済みのsupported local環境ではEditor／CLI／IDEによるmanual authoring、Validation、build、test、recoveryを維持する。AI経路の停止をProject破損、権限拡張またはEngine機能の偽装成功へfallbackしない。

Product-facing標準flowは`discover／query -> bounded context -> question／decision（必要時） -> typed proposal -> semantic diff -> validation -> approval -> atomic commit -> test／build -> receipt／explain／recovery`である。これはProduct journeyの順序関係であり、Operation名、Schema、実装順序またはActivationを定義しない。`undo`は対象Ownerがreversibleと定義したOperationだけに許し、不可逆な外部effectには事前Approval、明示的boundary、補償またはrecovery policyを要求して偽のundoを提示しない。正確なOperation membershipとprojectionは[Executable Contracts](../02-foundation/executable-contracts.md)、権限と承認は[AI Security／Approval](../01-governance/ai-security-approval.md)、transactionは[Project State](../03-authoring/project-state.md)、Editor interactionは[Editor Workspace／UX](../03-authoring/editor-workspace-ux.md)、Evidenceは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を一意なOwnerとする。

このflowを実行するRun、Workflow、immutable Context、deterministic／first-party／external route、Agent-loop単一所有、checkpoint／resume、surface semantic parityは[AI Production Orchestration](../03-authoring/ai-production-orchestration.md)が一意に所有する。AI-native claimでは、Run `completed`をTask completion、Commit、Approval、Evidence pass、ReleaseまたはProduct completionへ読み替えず、built-in AI、CLI、Native SDK、MCP、外部Agentが同じcanonical Operationとfailure境界へ収束しなければならない。Authoring Orchestrator、Agent、Credential、Workflow／Run Store、MCP Server、Compiler、Signerまたはwrite Gatewayをshipping Runtimeへ含めず、将来のshipping generative AIをAuthoring AIの実績から主張しない。

#### 1.1.1 AI game generation claim lane

```text
AiGameGenerationLaneV1 =
  ai_composed_game |
  ai_generated_external_content |
  ai_generated_project_source

AiGameGenerationClaimScopeV1
  ai_generation_claim_scope_id: StableId
  ai_generation_claim_scope_version: 1
  claimed_lane_set[1..3]: sorted unique AiGameGenerationLaneV1
  required_requirement_refs[1..256]:
    sorted unique exact McdContractRefV1(kind=requirement)
  required_journey_refs[1..256]: sorted unique exact ArtifactRefV1
  required_evidence_class_refs[1..256]:
    sorted unique exact EvidenceClassRefV1
  excluded_lane_set[0..2]: sorted unique AiGameGenerationLaneV1
  ai_generation_claim_scope_content_hash: SHA-256

AiGameGenerationClaimScopeRefV1
  ai_generation_claim_scope_id: StableId
  ai_generation_claim_scope_version: 1
  ai_generation_claim_scope_content_hash: SHA-256
```

各public claimは`claimed_lane_set[]`を明示し、次のnon-substituting意味を使う。`ai_composed_game`はGame Production Loopのtyped Document／Gameplay Definition／prequalified Pack／許可Assetを組成する。`ai_generated_external_content`はqualified Providerによる画像、音声、3D等をAsset provenance／rights／safety／human reviewへ閉じる。`ai_generated_project_source`はNative C++／Project Shader SourceをCode Owner、independent review、Build／Test、Promotionへ閉じる。

一laneのpass Receipt、CapabilityまたはDemoを他laneのsupport claimへ数えない。`ai_composed_game`だけの製品をoriginal Asset生成済みまたはNative Source生成対応と表現せず、外部content生成だけからGame完成またはProject Source supportを主張しない。Game Production Loopはこのclosed enumを消費してauthoring semanticsを所有し、Product Planへ別enum、aliasまたは第四laneを返さない。

### 1.2 第三者向け汎用Engineとしてのminimum surface

Miraikanaiの独自性は汎用Game Engineの基本journeyを省略する根拠ではない。第三者向けC2 claimは、競合製品の機能名や個数ではなく、少なくとも次のnon-substituting surface familyが同一releaseへ閉じることを要求する。

- Project作成／open／version control／recoveryと、Editor・CLI・外部IDE・AIからの同一canonical state利用。
- 2D／3D World、Gameplay、UI、Input、Audioと、構造化data／bounded C++の明示的なauthoring選択。
- Asset import／reimport／Cook／runtime delivery、依存関係、診断、source preservation、Target別artifact。
- Runtime、Simulation、Rendering、Save／Load／Replayの意味、failure、fallback、debug／profile。
- Project test、headless execution、Build／Cook／Package／clean install／launchとTarget qualification。
- 署名・由来付きPackとbounded Native Game Moduleによる公開拡張、dependency／license／update／removal。
- Documentation、Sample、Template、privacy、security、license、update／repair／rollback、supportとknown limitation。

各familyの型、API、Backend、UI layout、数値Budget、Target support、Activationは該当Ownerが所有する。public Editor extension marketplace、汎用Script runtime、Online、large world、Linux／macOS／Web／Console／XRは§8のFutureまたは明示的非目標であり、第三者向け汎用性から暗黙にcurrent scopeへ昇格しない。

## 2. 非交渉原則と明示的な非目標

### 2.1 非交渉原則

1. Game状態の正規表現とEditor／AI／RuntimeのProjectionを分離する。
2. 人間とAIは同じ変更Protocolを使い、AIの推論とEngineの権威判断を分離する。
3. Editorは特権的Writerではなく、正規状態のProjection兼ChangeSet作成者である。
4. 宣言的Source intentとTarget別Runtime artifactを分離する。
5. 製品構造を`Generic Engine Core`、`Reusable Feature Packs`、任意の`Genre Packs`、`Game Projects`の4層とし、Genre固有機能をCoreへ混入させない。
6. Engineが所有するのは公開Capability、正規data model、編集Protocol、validation、lifecycle、serialization、fallback、Editor UXである。OS、Compiler、Graphics API、検証済みKernelまで再発明しない。
7. 外部LibraryはEngine-owned Adapter内へ隔離し、Project／AI APIへVendor型を漏らさない。
8. Game programming modelはC++23と構造化Gameplay dataであり、汎用Game Script VM、JIT、Game向けFFIを持たない。
9. Game制作中のEngine、Editor、公開SDK、Validator、Policyは署名済みread-only baselineである。
10. 設計名、Tier、比較対象Engineの存在を「利用可能」の証拠にしない。利用可能性はTarget別Activation Evidenceだけで決める。
11. 契約を固定し、代表workloadを実測し、意味同等性と回帰Gateを通した候補だけを昇格する。Product PlanはAlgorithm、実装順、工数を固定しない。
12. Source intent、Gameplay fidelity floor、Visual StyleをTargetごとに分裂させない。Target差は各OwnerのCompilerとProfileが解決する。
13. 英語の正規技術語彙、localized presentation、User／Project原文を分離する。
14. 2Dと3Dの一方を他方の特殊値として定義せず、共有可能な基盤と固有の表現／Simulation contractを明示的に分ける。
15. Engine releaseのclaimは、同一release identityへ束縛されたEditor、SDK、Testing、Privacy、License、Support、Target qualificationのEvidence集合より強くしない。

### 2.2 非目標

- AIがC++ object、pointer、memory、GPU resource、Physics native handleを直接操作する経路。
- AIが任意Engine関数、shell、path、URL、console commandを呼ぶ経路。
- Provider Schemaに適合しただけで実行またはCommitを許すこと。
- Chat履歴、Editor view、Runtime WorldをProjectの正規記憶にすること。
- 一行Promptから無検証で完成品を一括生成すること。
- 他Engineの型階層、Scene形式、Editor操作、名称、creative expressionを模倣すること。
- 外部EngineのProject、Scene、Script、Plugin、Editor APIと一対一互換な別名、import pathまたはmigration layerをinitial V1へ持つこと。
- AI、Chat、MCPまたはcode generationの存在だけをAI-native、第三者利用可能、production-readyのEvidenceにすること。
- C1へOnline Multiplayer、Account、Cloud、広告、課金、Runtime code generationを持ち込むこと。
- 将来Capabilityを理由に空Module、空Directory、placeholder APIを先行作成すること。
- 未選定Provider、Backend、Store、Hardware、数値Budget、開発日程または人員をProduct正本で仮固定すること。
- full source-code editor、public Editor extension marketplace、AAA表現、video／media systemを第三者製品claimの暗黙必須条件にすること。採択する場合は独立Capabilityとして定義する。

## 3. Capability成熟度

### 3.1 C0–C3

Capability tierはProduct上の到達点であり、実装状態、日程または工程番号ではない。

| Tier | Product上の意味 | 最低Evidence class |
|---|---|---|
| C0 Foundation | identity、Owner、公開／禁止境界、validation、diagnostic、positive／negative contract scenarioが閉じる | contract／conformance Evidence |
| C1 First Playable | 一つの限定した2Dまたは3D Reference Projectを開始から結果、Save／Load、packageまで利用できる | same-candidate playable／Target smoke／regression Evidence |
| C2 Production | 2Dと3D、Authoring、SDK、Project testing、profiling、fallback、privacy、license、supportを第三者releaseとして閉じる | same-release 2D／3D／product-lifecycle Evidence |
| C3 Advanced | 大規模World、Online、特殊Device、特殊Genre等を独立した汎用Capabilityとして閉じる | 専用仕様、Threat Model、Benchmark、Pack／Target Evidence |

C3は「C2より良い実装」の総称ではない。各Capabilityが別Owner、別公開境界、意味同等fallback、個別Qualificationを持つ場合だけ採択する。

### 3.2 Activation state

Activation stateはTarget別Artifact Evidenceに基づく運用状態であり、tier、文書状態、実装状態、release claimと分離する。

| State | 意味 | Productで許される扱い |
|---|---|---|
| `not_activated` | 契約未完成、計画だけ、または利用可能なCandidateがない | Gapとして説明できるがProject利用、Package収録、成功表示は禁止 |
| `candidate_locked` | Source、Contract、Toolchain、Target、Policy、Evidence入力が固定された | 隔離Prototypeだけ。Production Projectへ提供しない |
| `qualified` | exact Target／Profileの必須Gateに合格した | allowlistされたProjectで制約付き利用可。別Targetへ一般化しない |
| `production` | Release closureと継続Support Gateまで合格した | Production Manifestの範囲で通常選択、Package、Release可 |

通常遷移は`not_activated -> candidate_locked -> qualified -> production`とする。Security incident、重大回帰、license失効、Evidence期限切れ、Provider停止では即時availabilityを停止し、署名済み降格Recordを追随させる。保存stateだけをavailabilityとしない。

必須Targetのrow欠落を`not_activated`で補完せずclosureを拒否する。optional Targetはaggregate claimへ混ぜない。Architecture文書、Feature名、OS／GPU brand、近いTarget、fallbackの存在からstateを推測しない。

<a id="p0-canonical-architecture"></a>

### 3.3 P0 Canonical Architecture Specification

`P0`は`Priority Zero Architecture Scope`であり、実装Phase、日程、Work Package、担当、工程順、Capability tierまたはActivation stateではない。Active Product Definition、First Playable、Minimum Executable CoreおよびProduct横断Gateを実現するため、C++、Schema、Registry、Fixture、Buildまたは実装計画を作る前に意味上閉じていなければならないOwner集合を表す。

P0は機能名の一覧ではない。各Subsystemの意味を本文書へ複写せず、exact Owner文書とcanonical fragmentへ束縛する。Owner文書が存在してもProduct Definitionから到達不能なFuture subjectはP0にせず、逆にProduct Definitionから到達するsubjectを手動判断で除外しない。

#### 3.3.1 型と16-axis closure

| 型 | ASCII domain separator |
|---|---|
| `P0SubsystemRefV1` | `MIRAKAN_P0_SUBSYSTEM_REF_V1` |
| `P0ArchitectureAxisBindingV1` | `MIRAKAN_P0_ARCHITECTURE_AXIS_BINDING_V1` |
| `P0SubsystemArchitectureClosureV1` | `MIRAKAN_P0_SUBSYSTEM_ARCHITECTURE_CLOSURE_V1` |
| `P0CanonicalArchitectureSpecificationV1` | `MIRAKAN_P0_CANONICAL_ARCHITECTURE_SPECIFICATION_V1` |

```text
P0SubsystemRefV1
  subsystem_id: StableId
  canonical_subsystem_owner_document_id:
    canonical ASCII Architecture document ID
  canonical_subsystem_owner_source_content_hash: SHA-256

P0ArchitectureAxisKindV1 =
  canonical_state
  | public_api
  | internal_api
  | state_machine
  | lifetime
  | threading
  | invariant
  | failure_atomicity
  | versioning
  | migration
  | compatibility
  | security_boundary
  | evidence_contract
  | acceptance_test
  | surface_conformance
  | performance_capacity

P0ArchitectureAxisBindingV1
  subsystem_ref: exact P0SubsystemRefV1
  axis_kind: P0ArchitectureAxisKindV1
  semantic_owner_document_id:
    canonical ASCII Architecture document ID
  semantic_owner_source_content_hash: SHA-256
  canonical_owner_fragment_ref:
    mirakan.arch.<document-id>#<fragment>
  shared_contract_fragment_refs[0..32]:
    sorted unique mirakan.arch.<document-id>#<fragment>
  applicability:
    {kind=required}
    | {
        kind=not_applicable,
        reason_artifact_ref: exact ArtifactRefV1,
        boundary_owner_document_id:
          canonical ASCII Architecture document ID,
        reconsideration_condition_refs[1..32]:
          sorted unique exact ArtifactRefV1
      }
  observed_document_status: draft | review | accepted | deprecated
  observed_implementation_status: absent | partial | implemented
  observed_verification_status:
    unreviewed | design-reviewed
    | prototype-verified | measurement-verified
  blocking_evidence_refs[0..256]:
    sorted unique exact EvidenceRefV1
  axis_binding_content_hash: SHA-256

P0SubsystemArchitectureClosureV1
  subsystem_closure_id: StableId
  subsystem_closure_version: positive u32
  source_inventory_ref:
    {source_repository_revision, inventory_content_hash}
  subsystem_ref: exact P0SubsystemRefV1
  membership_basis_refs[1..4096]:
    sorted unique exact ArtifactRefV1
  axis_bindings[16..16]:
    sorted unique P0ArchitectureAxisBindingV1
  architecture_closure_result: open | target_design_closed
  unresolved_diagnostic_refs[0..256]:
    sorted unique DiagnosticCodeRefV1
  subsystem_closure_content_hash: SHA-256

P0CanonicalArchitectureSpecificationV1
  p0_specification_id: StableId
  p0_specification_version: positive u32
  source_inventory_ref:
    {source_repository_revision, inventory_content_hash}
  product_definition_ref: exact ActiveProductDefinitionRefV1
  first_playable_definition_ref: exact FirstPlayableDefinitionRefV1
  minimum_executable_core_definition_ref:
    exact MinimumExecutableCoreDefinitionRefV1
  fixed_root_subsystem_refs[34..34]:
    sorted unique exact P0SubsystemRefV1
  derived_subsystem_refs[0..512]:
    sorted unique exact P0SubsystemRefV1
  subsystem_closure_refs[34..546]:
    sorted unique exact P0SubsystemArchitectureClosureRefV1
  excluded_future_subject_refs[0..4096]:
    sorted unique exact ArtifactRefV1
  p0_specification_content_hash: SHA-256
```

本節で使用するexact Refは次のclosed tupleである。

| Ref | Field |
|---|---|
| `P0SubsystemArchitectureClosureRefV1` | `{subsystem_closure_id, subsystem_closure_version, subsystem_closure_content_hash}` |
| `P0CanonicalArchitectureSpecificationRefV1` | `{p0_specification_id, p0_specification_version, p0_specification_content_hash}` |

各Refは解決先recordと全Field byte equalityでなければならない。Subsystem ID、Owner文書ID、最大version、Index順または近いsource hashからClosure／Specification Refを補完しない。

`axis_bindings[]`は16 enumをexactly onceずつ持つ。各axisの`semantic_owner_document_id`はexact一件であり、共通Envelope、encoding、generic state-machine表現、Evidence spineまたはcompatibility classを`shared_contract_fragment_refs[]`へ置いて複数Owner化しない。`not_applicable`は空欄、`none`、推測またはSubsystem名から生成せず、同じInventory上のboundary Owner、理由Artifact、再評価条件を必須にする。

`canonical_owner_fragment_ref`と全shared fragmentは同じ`source_inventory_ref`が束縛するOwner文書へ解決しなければならない。Source content hash、Header state、fragment、Owner relation、規範依存のいずれかが変わったClosureはstaleである。Index、表示見出し、path、行番号、自然言語説明、AI回答または`latest`からOwner／fragmentを補完しない。

#### 3.3.2 固定root Owner集合

どのActive Product Definitionでも、Product scope、Governance、Foundation、Authoring control plane、Runtime coreを解釈するため次の34 OwnerをP0固定rootとする。IDは集合membershipであり、表の表示順をserialization順または依存順にしない。

| group | exact canonical document IDs | cardinality |
|---|---|---:|
| Product | `mirakan.arch.product-plan`; `mirakan.arch.product-lifecycle`; `mirakan.arch.product-release-decision`; `mirakan.arch.product-publication-completion` | 4 |
| Governance | `mirakan.arch.architecture-governance`; `mirakan.arch.product-legal-ip-governance`; `mirakan.arch.product-security`; `mirakan.arch.product-privacy-data-governance`; `mirakan.arch.ai-security-approval`; `mirakan.arch.ai-verification-provenance` | 6 |
| Foundation | `mirakan.arch.core-architecture`; `mirakan.arch.toolchain-dependencies`; `mirakan.arch.executable-contracts`; `mirakan.arch.compatibility-evolution`; `mirakan.arch.naming-project-layout`; `mirakan.arch.cpp23-modules`; `mirakan.arch.math-core`; `mirakan.arch.memory-pointers` | 8 |
| Authoring | `mirakan.arch.project-state`; `mirakan.arch.ai-production-orchestration`; `mirakan.arch.game-production-loop`; `mirakan.arch.asset-lifecycle`; `mirakan.arch.editor-ui-framework`; `mirakan.arch.editor-workspace-ux`; `mirakan.arch.gameplay-programming-model`; `mirakan.arch.native-game-module`; `mirakan.arch.developer-testing` | 9 |
| Runtime core | `mirakan.arch.runtime-entity-component-system`; `mirakan.arch.runtime-scheduling-lifetime`; `mirakan.arch.runtime-asset-lifecycle`; `mirakan.arch.runtime-package`; `mirakan.arch.persistence-save`; `mirakan.arch.runtime-debugging-observability-replay`; `mirakan.arch.runtime-performance-capacity` | 7 |

固定rootが34件でない、ID重複、Owner文書不存在、Header ID不一致、source hash欠落、`review`を`approved`へ補完または実装`absent`を無視する場合はSpecification生成を拒否する。

#### 3.3.3 Product-derived Owner集合 Named Algorithm v1

Derived集合は次の順で決定論的に生成する。

1. `ActiveProductDefinitionV1`のCapability、Host、runtime Target、locale、required／bundled Pack requirement、Target×2D／3D Reference requirement、claim-facing requirement、Product data-flow、Operation family bindingを完全に展開する。
2. `FirstPlayableDefinitionV1`のCapability、Pack、Game flow、Input、Accessibility、Target、Runtime Entry、Scenario、Requirement、Operation family、Evidence classを追加する。
3. `MinimumExecutableCoreDefinitionV1`のrequired role、Scenario、Subsystem bindingを追加する。
4. 各subjectを同じ`ArchitectureInventoryV1`のexact owner relationへjoinし、canonical Owner documentを一件だけ解決する。zero／multiple Ownerはerrorである。
5. 解決したOwnerから同Inventoryの規範依存edgeをDAG末端まで展開し、Owner文書だけをcanonical set unionする。
6. 固定rootとの差集合を`derived_subsystem_refs[]`、union全体を`subsystem_closure_refs[]`のSubsystem projectionとset equalityにする。
7. Future Portfolioだけから到達するsubjectは`excluded_future_subject_refs[]`へ記録し、Active／First Playable／Minimum Coreからも到達するsubjectだけを除外集合から取り除く。

Owner文書の存在、Architecture Index掲載、Feature名、競合Engine比較、ProposalのWork Package、Future Capability、optional Fixtureまたは近いSubsystemからDerived membershipを追加しない。反対に、Active Product DefinitionのTarget／Pack／Reference／Requirementから到達するOwnerを「後で実装する」「optionalに見える」等の説明で削除しない。

#### 3.3.4 16-axisの所有規則

| Axis | primary routing rule | 不正な代用 |
|---|---|---|
| `canonical_state` | Subsystem Ownerのauthoritative stateと[Architecture Governance](../01-governance/architecture-governance.md)の文書／実装／検証状態 | UI表示、Feature名、保存Boolean |
| `public_api` | Subsystem Owner。C++共通surfaceは[C++23](../02-foundation/cpp23-modules.md)／[Native Game Module](../03-authoring/native-game-module.md)、Operationは[Executable Contracts](../02-foundation/executable-contracts.md)をshared refにする | internal header、Widget、MCP Tool名 |
| `internal_api` | Subsystem Ownerと[Core Architecture](../02-foundation/core-architecture.md)のLayer／Host／Gateway境界 | public SDKまたはAI projectionへのprivate型漏出 |
| `state_machine` | Domain transitionはSubsystem Owner、generic representationはExecutable Contracts | conversation、session、UI state |
| `lifetime` | Domain lifetimeはSubsystem Owner、Runtime共通order／scopeは[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md) | pointer lifetime、frame表示からの推測 |
| `threading` | Subsystem access／writer ruleはSubsystem Owner、共通Job／phaseはScheduling／Lifetime | callback実行threadの暗黙default |
| `invariant` | Subsystem Ownerのclosed invariantとvalidator boundary | example、happy-path testだけ |
| `failure_atomicity` | Subsystem Owner。Project mutationは[Project State](../03-authoring/project-state.md)、共通Task publicationはCoreをshared refにする | partial success、silent repair、偽undo |
| `versioning` | Domain versionはSubsystem Owner、class／consumer protectionは[Compatibility／Evolution](../02-foundation/compatibility-evolution.md) | file名、latest、release label |
| `migration` | Domain変換意味はSubsystem Owner、admission／clean breakはCompatibility | dual reader、silent coercion |
| `compatibility` | Compatibility classはCompatibility Owner、対象consumerはSubsystem Owner | ABIだけによる全互換主張 |
| `security_boundary` | Domain trust／input boundaryはSubsystem Owner、AI authorityはAI Security、Product threatはProduct Security | Provider permission、UI confirmationだけ |
| `evidence_contract` | Domain合否payloadはSubsystem Owner、generic envelope／freshnessは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md) | log、screenshot、hash-only |
| `acceptance_test` | Subsystem Ownerのpositive／negative／fault requirement。Project test表現はDeveloper Testing | test名または件数だけ |
| `surface_conformance` | Product journeyは[Product Lifecycle](product-lifecycle.md)、AI client routeは[AI Production Orchestration](../03-authoring/ai-production-orchestration.md)、Domain Operation意味はSubsystem Owner | GUI成功によるSDK／MCP／Agent代用 |
| `performance_capacity` | [Performance／Capacity](../04-runtime/performance-capacity.md)またはDomain／Platform Ownerのexact budget／measurement fragment | provisional値、別Target、平均値だけ |

Subsystemが外部surfaceを持たない場合も`public_api`または`surface_conformance`を空にせず、internal-only boundaryと再評価条件を持つ`not_applicable`へ閉じる。数値Budgetが意味を持たないGovernance recordも同様に、処理量／carrier bound／latencyがどのOwnerに属するか、またはなぜProduct性能Gate対象外かを明示する。

#### 3.3.5 Closure predicateとcurrent state

Subsystem `target_design_closed`は次のpure predicateをすべて満たす時だけ成立する。

1. Subsystem refがfixed rootまたはProduct-derived集合へexactly once存在する。
2. canonical Subsystem Ownerがexactly oneで、InventoryとHeaderが一致する。
3. 16 axisがexactly once存在し、全Owner／fragment／shared refが同Inventoryへ解決する。
4. `not_applicable`が理由、boundary Owner、再評価条件を持つ。
5. 規範依存がDAGで、下位文書からP0 rootへの意味逆流または第二Ownerがない。
6. `unresolved_diagnostic_refs[]=[]`である。

このpredicateは`observed_implementation_status=implemented`、`observed_verification_status=prototype-verified | measurement-verified`またはEvidence non-emptyを要求してArchitecture意味を実装と混同しない。一方、各axisはcurrent stateをread-backするため、`absent`、`unreviewed`、`provisional`を`closed`へ隠せない。Product release、Capability activation、性能達成、制作体験または商用運用は各Ownerの別Gateであり、P0 Architecture closureをEvidenceへ数えない。

Current Repositoryでは`ArchitectureInventoryV1`、Active Product Definition record、P0 Schema／Generator／Registry／Specification／Closure Artifactが未materializeである。したがってP0 target semanticsと固定root集合はclosed-in-target-design候補だが、P0 Specification instance、全Derived membership、16-axis machine validation、Conformance SuiteまたはAcceptance Receiptは`absent`である。手動Indexまたは本節の表をmaterialized Specificationとして扱わない。

## 4. 2D／3D Capability portfolio

2Dと3Dは同格のfirst-class Runtimeである。Asset、Input、Audio、UI、Gameplay、AI Authoring、Build、Save、Diagnosticsの共有契約を利用できるが、Rendering、Camera、Physics、Navigation、Animation、World Spaceの固有意味を相互代用しない。

| Product surface | 2D closure | 3D closure | 共通closure |
|---|---|---|---|
| World／Scene | Tile／Layer／2D spatial authoring | Mesh／Scene／3D spatial authoring | Project state、streaming、identity、transition |
| Rendering | Sprite／2D material／ordering／2D camera | Mesh／material／lighting／3D camera | Render Graph、resource lifecycle、diagnostic |
| Simulation | 2D collision／physics／grid navigation／2D animation | 3D collision／physics／navmesh／skeletal animation | scheduling、events、Save／Replay semantics |
| Authoring | 2D Reference Project journey | 3D Reference Project journey | Editor、Asset、C++ SDK、Shader、Testing、Debugging |
| Product delivery | 2D clean build／package／play evidence | 3D clean build／package／play evidence | release、docs、privacy、license、support |

First Playableは学習とrisk低減のため2Dまたは3Dのどちらか一方で成立できる。ただし一方のEvidenceは他方を代用しない。`2D・3D対応Game Engine`、`2D／3D production-ready`または同等のProduct claimは、同一Engine release、同一release candidate、同一required Target集合へ束縛された次のnon-substituting Evidenceをすべて必要とする。

1. 完成した2D Reference Projectのauthoring、test、Save／Load、build、package、clean install／launch。
2. 完成した3D Reference Projectのauthoring、test、Save／Load、build、package、clean install／launch。
3. 共通Editor、SDK、Project format、diagnostic、Documentation、update、privacy、license、supportのE2E acceptance。
4. 2D／3D固有Ownerが定める意味、performance、fault、fallback、Target qualification。

2D RPG Referenceはdata-rich Gameplay、UI／Text／Localization、Save／Load、AI AuthoringのProduct referenceとする。3D technical ReferenceはWorld、Camera、Mesh、Lighting、Animation、Physics、Navigation、3D packageのProduct referenceとする。Shooter fixtureを3D referenceに利用できるが、Product MVPやGeneric CoreのGenre依存を生じさせない。両Referenceは独自contentとし、第三者の名称、人物、story、map、music、art、UI、商標その他のcreative expressionを再利用しない。

## 5. MVP scope

MVPは「全Engine完成」ではなく、第三者Developer journeyを最小のrelease candidateで閉じるProduct outcomeである。工程、順序、工数、担当、Fixture件数は本書で定義しない。

### 5.1 First Playable outcome

最初のProduct-facing First Playableはcompact 2D command RPGとする。最低Outcomeは次である。

- signed Engine／Editor／SDK候補からProjectを作成し、TitleからResultまでofflineで完走できる。
- Map探索、会話、command battle、inventory／progress、Save／Load、Settings、localization、input remapを一つのProject revision系列で扱える。
- GUIと公開C++／structured-data経路が同じ正規Project state、validation、diagnosticへ収束する。
- Project test、debug capture、performance capture、build、cook、package、clean install／launchをsame candidateへ関連付けられる。
- Template、Sample、Documentation、NOTICE、privacy disclosure、support boundaryが同じEngine releaseへ束縛される。
- 失敗時にpartial Project、壊れたSave、silent fallback、誤った成功表示を残さない。

Exact scene数、dialogue数、Asset数、test数、対応環境数はQualification Ownerまたは非規範Fixture catalogが、対象契約を過不足なく証明するために決める。件数自体をProduct完成条件にしない。

```text
FirstPlayableGameFlowRoleV1 =
  title |
  map_exploration |
  conversation |
  command_battle |
  inventory_progress |
  save_load |
  settings |
  result

FirstPlayableInputDeviceClassV1 = keyboard | controller

FirstPlayableDefinitionV1
  first_playable_definition_id:
    first_playable.compact_2d_command_rpg
  first_playable_definition_version: 1
  product_definition_ref: exact ActiveProductDefinitionRefV1
  reference_dimension: two_d
  genre_pack_requirement_ref:
    exact McdContractRefV1(kind=requirement)
  required_feature_pack_requirement_refs[1..64]:
    sorted unique exact McdContractRefV1(kind=requirement)
  required_game_flow_roles[8..8]:
    sorted unique FirstPlayableGameFlowRoleV1
  required_locale_profile_refs[2..2]: sorted unique exact LocaleProfileRefV1
  required_input_action_refs[1..256]: sorted unique exact ArtifactRefV1
  required_input_device_classes[2..2]:
    sorted unique FirstPlayableInputDeviceClassV1
  input_remap_required: true
  required_accessibility_requirement_refs[1..256]:
    sorted unique exact McdContractRefV1(kind=requirement)
  required_capability_refs[1..1024]: sorted unique exact ArtifactRefV1
  required_host_profile_refs[1..16]:
    sorted unique exact TargetProfileRefV1(
      profile_kind=build_host | editor_host)
  required_runtime_target_profile_refs[1..16]:
    sorted unique exact TargetProfileRefV1(profile_kind=runtime_target)
  required_runtime_entry_role_refs[1..64]: sorted unique exact ArtifactRefV1
  required_qualification_scenario_refs[1..256]:
    sorted unique exact QualificationScenarioRefV1
  required_requirement_refs[1..4096]:
    sorted unique exact McdContractRefV1(kind=requirement)
  required_operation_family_refs[1..256]: sorted unique exact ArtifactRefV1
  required_evidence_class_refs[1..256]:
    sorted unique exact EvidenceClassRefV1
  required_ai_generation_claim_scope_ref:
    exact AiGameGenerationClaimScopeRefV1
  required_game_production_loop_contract_ref:
    exact McdContractRefV1(kind=type)
  manual_authoring_continuity_required: true
  clean_install_required: true
  offline_completion_required: true
  first_playable_definition_content_hash: SHA-256

FirstPlayableDefinitionRefV1
  first_playable_definition_id: StableId
  first_playable_definition_version: 1
  first_playable_definition_content_hash: SHA-256
```

initial exact Definitionは`required_locale_profile_refs={en-US, ja-JP}`、`required_input_device_classes={keyboard, controller}`、`required_ai_generation_claim_scope_ref.claimed_lane_set={ai_composed_game}`である。`ai_generated_external_content`と`ai_generated_project_source`はexcluded laneで、C1 First Playable成立条件にしない。required Capabilityは2D rendering、UI／Text／Localization／Accessibility、Audio、Input、Save／Load、Gameplay Definition、Project Testing、Build／Cook／Package／Launchを過不足なく含み、required PackはRPG GenreとDefinitionが選ぶreusable RPG Featureだけを含む。

Acceptanceは同じsigned Engine／Editor／SDK Candidateからbootstrapした一つのProject revision lineageで、author、validate、test、package、clean install、launch、Title→Result、offline completion、checkpoint Save／Load、Settings、keyboard／controller remap、`en-US`／`ja-JP` semantic invarianceを閉じる。さらに[Game Production Loop](../03-authoring/game-production-loop.md)の`GameProductionLoopClosureV1=accepted`を同じCandidate／Project revisionへ要求する。Loop Closureから`GameUnderstandingClosureV1=ready_to_stage`、Experience Goal、Playtest Observation／Evaluation、`GameIterationDecisionV1=accept_candidate`、有効Human Gameplay Approvalをread-backし、Candidateを持たないUnderstanding ClosureへCandidate Fieldを補完しない。

Shooter Fixture、3D technical Reference、別Genre、別Project、manual-only journey、AI-only journey、wrong Target、別Candidateまたは`fresh`でないEvidenceをこのDefinitionへ使わない。Definition自身はReceipt-freeなtarget requirementであり、対応MCD、Registry、RPG Project、Package、Fixture、ReceiptがmaterializeするまでFirst Playable完成を主張しない。

### 5.2 3D Product reference outcome

3D ReferenceはC2の2D／3D製品claimより前に、同じEngine release candidateで次を閉じる。

- 3D World／Scene、Mesh／Material／Lighting／Camera、skeletal Animation、3D Collision／Physics／NavigationをEditorからauthoringできる。
- Project C++とProject Shaderの公開契約を使い、外部IDEからbuild／diagnosticへ往復できる。
- deterministicでないKernel結果を含む場合も、Ownerが定めるSave／Replay／debug evidence boundaryを破らない。
- required Targetでperformance、fallback、device loss／resource pressure、package、clean launchを検証する。
- 2D Referenceと同じProject lifecycle、Testing、Privacy、License、Support contractを使用する。

### 5.3 Localeとaccessibility floor

C1のEditor表示locale、明示AI返答locale、Game Reference localeは`en-US`と`ja-JP`を最低集合とする。追加localeはUI／Text Ownerのlocale profile、font／line-break／input、Documentation、AI Eval、Reference contentを同じreleaseでQualificationする。locale数だけを国際化完了の証拠にしない。

EditorとGame UIはkeyboard-only、focus、screen-reader／platform accessibility bridge、text scaling、contrast、reduced-motionまたはOwnerが定める代替をProduct Gateへ含める。自動checkだけで利用者journeyの成立を主張しない。

## 6. Product evidence hierarchy

本書はProduct Profile identityのreceipt-free root Registry contractだけを所有し、実行計画、materialized execution Registryまたは生成Schemaを所有しない。Product claimは次の順でのみ強くできる。

1. Owner文書が対象Capabilityの意味、禁止境界、必要Evidenceを定義する。
2. materialized Contract／Artifact／Toolがexact revisionとCandidateへ束縛される。
3. Developer TestingとDomain qualificationがpositive／negative／fault scenarioを実行する。
4. Product LifecycleがEditor、SDK、Documentation、Platform package、Privacy、License、Supportをsame releaseへ集約する。
5. Release decisionがclaimとEvidence scopeの一致を確認する。

下位段階が欠けた状態で上位段階を記録しない。`review`文書、proposal、mock、Prototype、別candidate Receipt、別Target成功、手動デモ、未再現計測を代用しない。

Product executionの候補Registry、Work Package、phase、担当、見積りは[Product Execution Registry Proposal](../appendices/product-execution-registry-proposal.md)に非規範proposalとして隔離する。承認されてもProduct intent、Capability意味またはRelease Gateの正本にはならない。

### 6.1 Active Product Definitionとrequirement projection

Product releaseとProduct completionのrequired universeはDecision作成者のinline入力ではなく、receipt-freeなActive Product Definitionと決定論的requirement projectionからだけ導出する。本節はProduct定義、claim scope、要求導出、pure predicateだけを所有し、最終authority recordは[Product Release Decision](product-release-decision.md)、実公開とauthoritative Completionは[Product Publication／Completion](product-publication-completion.md)が所有する。

本節のASCII domain separatorは次である。

| 型 | ASCII domain separator |
|---|---|
| `TargetProfileV1` | `MIRAKAN_TARGET_PROFILE_V1` |
| `TargetProfileRegistryV1` | `MIRAKAN_TARGET_PROFILE_REGISTRY_V1` |
| `LocaleProfileV1` | `MIRAKAN_LOCALE_PROFILE_V1` |
| `LocaleProfileRegistryV1` | `MIRAKAN_LOCALE_PROFILE_REGISTRY_V1` |
| `ActiveProductDefinitionV1` | `MIRAKAN_ACTIVE_PRODUCT_DEFINITION_V1` |
| `ProductClaimKindMinimumV1` | `MIRAKAN_PRODUCT_CLAIM_KIND_MINIMUM_V1` |
| `ProductClaimScopeV1` | `MIRAKAN_PRODUCT_CLAIM_SCOPE_V1` |
| `ProductRequirementProjectionInputV1` | `MIRAKAN_PRODUCT_REQUIREMENT_PROJECTION_INPUT_V1` |
| `ProductReleaseRequirementProjectionV1` | `MIRAKAN_PRODUCT_RELEASE_REQUIREMENT_PROJECTION_V1` |
| `ProductCompletionRequirementProjectionV1` | `MIRAKAN_PRODUCT_COMPLETION_REQUIREMENT_PROJECTION_V1` |
| `RequiredProductOperationUniverseV1` | `MIRAKAN_REQUIRED_PRODUCT_OPERATION_UNIVERSE_V1` |
| `RequiredProductOperationJourneyProjectionV1` | `MIRAKAN_REQUIRED_PRODUCT_OPERATION_JOURNEY_PROJECTION_V1` |
| `ProductOperationActivationClosureV1` | `MIRAKAN_PRODUCT_OPERATION_ACTIVATION_CLOSURE_V1` |
| `ProductProductionActivationSetV1` | `MIRAKAN_PRODUCT_PRODUCTION_ACTIVATION_SET_V1` |

```text
ProductSurfaceKindV1 =
  editor_gui | cli | headless | native_sdk
  | external_ide | mcp | ai_automation

TargetProfileV1
  target_profile_id: StableId
  target_profile_version: positive u32
  profile_kind: build_host | editor_host | runtime_target
  product_platform_kind: windows | android | apple
  product_scope_role:
    build_test_host | editor_authoring_host
    | desktop_game_runtime | mobile_game_runtime
  canonical_owner_document_id: mirakan.arch.product-plan
  canonical_owner_fragment: product-profile-identity
  target_profile_content_hash: SHA-256

TargetProfileRegistryV1
  target_profile_registry_id: StableId
  target_profile_registry_version: 1
  target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1
  target_profile_registry_content_hash: SHA-256

LocaleProfileV1
  locale_profile_id: StableId
  locale_profile_version: positive u32
  canonical_language_tag: canonical BCP 47 language tag
  locale_role: source_and_presentation | presentation
  canonical_owner_document_id: mirakan.arch.product-plan
  canonical_owner_fragment: product-profile-identity
  locale_profile_content_hash: SHA-256

LocaleProfileRegistryV1
  locale_profile_registry_id: StableId
  locale_profile_registry_version: 1
  locale_profile_refs[1..64]:
    sorted unique exact LocaleProfileRefV1
  locale_profile_registry_content_hash: SHA-256

ActiveProductDefinitionV1
  active_product_definition_id: StableId
  active_product_definition_schema_version: 1
  active_product_definition_revision: positive u64
  target_profile_registry_ref: exact TargetProfileRegistryRefV1
  locale_profile_registry_ref: exact LocaleProfileRegistryRefV1
  capability_refs[1..4096]:
    sorted unique exact McdContractRefV1(kind=capability)
  host_profile_refs[2..64]:
    sorted unique exact TargetProfileRefV1(
      profile_kind=build_host | editor_host)
  runtime_target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1(profile_kind=runtime_target)
  locale_profile_refs[2..64]:
    sorted unique exact LocaleProfileRefV1
  required_pack_requirement_refs[0..1024]:
    sorted unique exact McdContractRefV1(kind=requirement)
  bundled_pack_requirement_refs[0..1024]:
    sorted unique exact McdContractRefV1(kind=requirement)
  reference_requirement_bindings[2..4096]:
    sorted unique {
      target_profile_ref:
        exact TargetProfileRefV1(profile_kind=runtime_target),
      reference_dimension: two_d | three_d,
      requirement_ref: exact McdContractRefV1(kind=requirement)
    }
  claim_facing_requirement_refs[1..4096]:
    sorted unique exact McdContractRefV1(kind=requirement)
  requirement_projection_input_bindings[1..4096]:
    sorted unique {
      requirement_ref: exact McdContractRefV1(kind=requirement),
      projection_input_ref:
        exact ProductRequirementProjectionInputRefV1
    }
  required_product_data_flow_refs[0..1024]:
    sorted unique exact McdContractRefV1(kind=data_flow)
  claim_kind_minimum_refs[5..5]:
    sorted unique exact ProductClaimKindMinimumRefV1
  operation_family_bindings[1..512]:
    sorted unique {
      operation_family_kind:
        product_install | product_update | product_repair
        | product_uninstall | project_bootstrap | project_open
        | authoring_author | authoring_preview
        | authoring_validate | authoring_commit
        | project_build | project_test | project_cook
        | project_package | project_launch
        | diagnostics | support
        | pack_acquire | pack_install | pack_apply
        | pack_update | pack_remove
        | ai_read | ai_explain | ai_propose
        | ai_validate | ai_approve | ai_commit
        | security_case_transition
        | security_update | disclosure | incident_response
        | publication | withdrawal,
      operation_ref: exact McdContractRefV1(kind=operation),
      governed_requirement_categories[1..32]:
        sorted unique ProductRequirementCategoryV1,
      allowed_surface_kinds[1..7]:
        sorted unique ProductSurfaceKindV1
    }
  active_product_definition_content_hash: SHA-256

ProductClaimKindMinimumV1
  claim_kind_minimum_id: StableId
  claim_kind_minimum_version: 1
  claim_kind:
    technology_preview | first_playable_2d | first_playable_3d
    | engine_2d_3d | third_party_product_release
  required_host_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1(
      profile_kind=build_host | editor_host)
  required_runtime_target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1(profile_kind=runtime_target)
  required_locale_profile_refs[1..64]:
    sorted unique exact LocaleProfileRefV1
  required_capability_refs[1..4096]:
    sorted unique exact McdContractRefV1(kind=capability)
  required_requirement_bindings[1..4096]:
    sorted unique {
      requirement_category: ProductRequirementCategoryV1,
      requirement_ref: exact McdContractRefV1(kind=requirement)
    }
  required_target_dimension_bindings[0..128]:
    sorted unique {
      target_profile_ref:
        exact TargetProfileRefV1(profile_kind=runtime_target),
      reference_dimension: two_d | three_d,
      reference_requirement_ref:
        exact McdContractRefV1(kind=requirement)
    }
  required_operation_family_kinds[1..64]:
    sorted unique operation_family_kind
  claim_kind_minimum_content_hash: SHA-256

ProductClaimScopeV1
  claim_scope_id: StableId
  claim_scope_version: positive u32
  claim_kind:
    technology_preview | first_playable_2d | first_playable_3d
    | engine_2d_3d | third_party_product_release
  host_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1(
      profile_kind=build_host | editor_host)
  runtime_target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1(profile_kind=runtime_target)
  capability_refs[1..4096]:
    sorted unique exact McdContractRefV1(kind=capability)
  required_reference_dimensions:
    none | two_d_only | three_d_only | two_d_and_three_d
  locale_profile_refs[1..64]:
    sorted unique exact LocaleProfileRefV1
  known_limitation_refs[0..4096]:
    sorted unique exact DocumentationEntryRefV1
  additional_requirement_refs[0..4096]:
    sorted unique exact McdContractRefV1(kind=requirement)
  additional_operation_family_kinds[0..64]:
    sorted unique operation_family_kind
  claim_scope_content_hash: SHA-256

ProductRequirementProjectionInputV1
  requirement_projection_input_id: StableId
  requirement_projection_input_version: 1
  requirement_ref: exact McdContractRefV1(kind=requirement)
  requirement_category: ProductRequirementCategoryV1
  prepublication_evidence_class_refs[0..32]:
    sorted unique exact EvidenceClassRefV1
  completion_evidence_class_refs[0..32]:
    sorted unique exact EvidenceClassRefV1
  host_applicability:
    {kind=all_claim_scope_hosts}
    | {kind=host_independent}
    | {kind=not_applicable}
    | {
        kind=exact_set,
        host_profile_refs[1..64]:
          sorted unique exact TargetProfileRefV1(
            profile_kind=build_host | editor_host)
      }
  target_applicability:
    {kind=all_claim_scope_targets}
    | {kind=target_independent}
    | {kind=not_applicable}
    | {
        kind=exact_set,
        target_profile_refs[1..64]:
          sorted unique exact TargetProfileRefV1(
            profile_kind=runtime_target)
      }
  locale_applicability:
    {kind=all_claim_scope_locales}
    | {kind=locale_independent}
    | {kind=not_applicable}
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
  completion_distribution_scope_applicability:
    {kind=not_applicable}
    | {
        kind=exact_set,
        distribution_scope_kinds[1..10]:
          sorted unique ProductCompletionDistributionScopeKindV1
      }
  publication_distribution_applicabilities[0..256]:
    sorted unique {
      platform_kind: windows | android | apple,
      channel_kind: direct_distribution | managed_store,
      distribution_scope_applicability:
        {kind=all_requirement_distribution_scopes}
        | {
            kind=exact_set,
            distribution_scope_kinds[1..10]:
              sorted unique ProductCompletionDistributionScopeKindV1
          },
      artifact_role_applicability:
        {kind=all_distribution_artifact_roles}
        | {
            kind=exact_set,
            artifact_role_kinds[1..20]:
              sorted unique ProductDistributionArtifactRoleKindV1
          },
      execution_scope_applicability:
        {kind=all_requirement_hosts}
        | {kind=all_requirement_targets}
        | {kind=scope_independent}
        | {
            kind=exact_host_set,
            host_profile_refs[1..64]:
              sorted unique exact TargetProfileRefV1(
                profile_kind=build_host | editor_host)
          }
        | {
            kind=exact_runtime_target_set,
            runtime_target_profile_refs[1..64]:
              sorted unique exact TargetProfileRefV1(
                profile_kind=runtime_target)
          },
      locale_applicability:
        {kind=all_requirement_locales}
        | {kind=locale_independent}
        | {
            kind=exact_set,
            locale_profile_refs[1..64]:
              sorted unique exact LocaleProfileRefV1
          }
    }
  journey_applicabilities[0..4096]:
    sorted unique {
      operation_family_kind: operation_family_kind,
      host_applicability:
        {kind=host_independent}
        | {
            kind=exact_set,
            host_profile_refs[1..64]:
              sorted unique exact TargetProfileRefV1(
                profile_kind=build_host | editor_host)
          },
      target_applicability:
        {kind=target_independent}
        | {kind=all_requirement_targets}
        | {
            kind=exact_set,
            target_profile_refs[1..64]:
              sorted unique exact TargetProfileRefV1(
                profile_kind=runtime_target)
          },
      locale_applicability:
        {kind=all_requirement_locales}
        | {kind=locale_independent}
        | {kind=not_applicable}
        | {
            kind=exact_set,
            locale_profile_refs[1..64]:
              sorted unique exact LocaleProfileRefV1
          },
      reference_dimension_applicability:
        {kind=all_requirement_reference_dimensions}
        | {kind=dimension_independent}
        | {kind=not_applicable}
        | {
            kind=exact_set,
            reference_dimensions[1..2]:
              sorted unique two_d | three_d
          },
      qualification_scenario_ref: exact QualificationScenarioRefV1,
      semantic_equivalence_group_id: StableId,
      expected_result_branches[1..3]:
        sorted unique
          success | expected_policy_rejection | domain_failure_recovery,
      evidence_class_refs[1..32]:
        sorted unique exact EvidenceClassRefV1,
      required_surface_kinds[1..7]:
        sorted unique ProductSurfaceKindV1,
      forbidden_surface_kinds[0..7]:
        sorted unique ProductSurfaceKindV1
    }
  workflow_applicabilities[0..12]:
    sorted unique {
      workflow_kind:
        install | project_bootstrap | author | build | test | package
        | launch | update | repair | diagnose | support | uninstall,
      host_applicability:
        {kind=host_independent}
        | {
            kind=exact_set,
            host_profile_refs[1..64]:
              sorted unique exact TargetProfileRefV1(
                profile_kind=build_host | editor_host)
          },
      target_applicability:
        {kind=target_independent}
        | {kind=all_requirement_targets}
        | {
            kind=exact_set,
            target_profile_refs[1..64]:
              sorted unique exact TargetProfileRefV1(
                profile_kind=runtime_target)
          },
      required_distribution_scope_kinds[1..10]:
        sorted unique ProductCompletionDistributionScopeKindV1
    }
  requirement_projection_input_content_hash: SHA-256

ProductReleaseRequirementProjectionV1
  release_requirement_projection_id: StableId
  release_requirement_projection_version: 1
  active_product_definition_ref: exact ActiveProductDefinitionRefV1
  claim_scope_ref: exact ProductClaimScopeRefV1
  claim_kind_minimum_ref: exact ProductClaimKindMinimumRefV1
  host_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1(
      profile_kind=build_host | editor_host)
  target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1(profile_kind=runtime_target)
  locale_profile_refs[1..64]:
    sorted unique exact LocaleProfileRefV1
  required_activation_subjects[1..4096]:
    sorted unique {
      capability_ref: exact McdContractRefV1(kind=capability),
      target_profile_ref:
        exact TargetProfileRefV1(profile_kind=runtime_target)
    }
  required_evidence_class_refs[1..4096]:
    sorted unique exact EvidenceClassRefV1
  required_acceptance_subjects[1..4096]:
    sorted unique {
      requirement_ref: exact McdContractRefV1(kind=requirement),
      requirement_category: ProductRequirementCategoryV1,
      evidence_class_ref: exact EvidenceClassRefV1,
      required_host_scope:
        {kind=not_applicable}
        | {kind=host_independent}
        | {
            kind=exact_set,
            host_profile_refs[1..64]:
              sorted unique exact TargetProfileRefV1(
                profile_kind=build_host | editor_host)
          },
      required_target_scope:
        {kind=not_applicable}
        | {kind=target_independent}
        | {
            kind=exact_set,
            target_profile_refs[1..64]:
              sorted unique exact TargetProfileRefV1(
                profile_kind=runtime_target)
          },
      required_locale_scope:
        {kind=not_applicable}
        | {kind=locale_independent}
        | {
            kind=exact_set,
            locale_profile_refs[1..64]:
              sorted unique exact LocaleProfileRefV1
          },
      required_reference_dimension_scope:
        {kind=not_applicable}
        | {kind=dimension_independent}
        | {
            kind=exact_set,
            reference_dimensions[1..2]:
              sorted unique two_d | three_d
          }
    }
  required_target_dimension_bindings[0..128]:
    sorted unique {
      target_profile_ref:
        exact TargetProfileRefV1(profile_kind=runtime_target),
      reference_dimension: two_d | three_d,
      reference_requirement_ref:
        exact McdContractRefV1(kind=requirement)
    }
  required_publication_distribution_subjects[1..65535]:
    sorted unique {
      publication_requirement_ref:
        exact McdContractRefV1(kind=requirement),
      platform_kind: windows | android | apple,
      channel_kind: direct_distribution | managed_store,
      distribution_scope_kind:
        ProductCompletionDistributionScopeKindV1,
      artifact_role_kind: ProductDistributionArtifactRoleKindV1,
      execution_scope:
        {
          kind=host_profile,
          host_profile_ref: exact TargetProfileRefV1(
            profile_kind=build_host | editor_host)
        }
        | {
            kind=runtime_target_profile,
            runtime_target_profile_ref: exact TargetProfileRefV1(
              profile_kind=runtime_target)
          }
        | {kind=scope_independent},
      locale_scope:
        {
          kind=locale_profile,
          locale_profile_ref: exact LocaleProfileRefV1
        }
        | {kind=locale_independent}
    }
  projection_algorithm_id:
    product_release_requirement_projection
  projection_algorithm_version: 1
  projection_algorithm_content_hash: SHA-256
  release_requirement_projection_content_hash: SHA-256

ProductCompletionRequirementProjectionV1
  completion_requirement_projection_id: StableId
  completion_requirement_projection_version: 1
  active_product_definition_ref: exact ActiveProductDefinitionRefV1
  release_requirement_projection_ref:
    exact ProductReleaseRequirementProjectionRefV1
  claim_scope_ref: exact ProductClaimScopeRefV1
  required_completion_bindings[1..8192]:
    sorted unique {
      requirement_ref: exact McdContractRefV1(kind=requirement),
      evidence_class_ref: exact EvidenceClassRefV1,
      required_host_scope:
        {kind=not_applicable}
        | {kind=host_independent}
        | {
            kind=exact_set,
            host_profile_refs[1..64]:
              sorted unique exact TargetProfileRefV1(
                profile_kind=build_host | editor_host)
          },
      required_target_scope:
        {kind=not_applicable}
        | {kind=target_independent}
        | {
            kind=exact_set,
            target_profile_refs[1..64]:
              sorted unique exact TargetProfileRefV1(
                profile_kind=runtime_target)
          },
      required_locale_scope:
        {kind=not_applicable}
        | {kind=locale_independent}
        | {
            kind=exact_set,
            locale_profile_refs[1..64]:
              sorted unique exact LocaleProfileRefV1
          },
      required_reference_dimension_scope:
        {kind=not_applicable}
        | {kind=dimension_independent}
        | {
            kind=exact_set,
            reference_dimensions[1..2]:
              sorted unique two_d | three_d
          },
      required_distribution_scope:
        {kind=not_applicable}
        | {
            kind=exact_set,
            distribution_scope_kinds[1..10]:
              sorted unique ProductCompletionDistributionScopeKindV1
          }
    }
  projection_algorithm_id:
    product_completion_requirement_projection
  projection_algorithm_version: 1
  projection_algorithm_content_hash: SHA-256
  completion_requirement_projection_content_hash: SHA-256

RequiredProductOperationUniverseV1
  required_operation_universe_id: StableId
  required_operation_universe_version: 1
  active_product_definition_ref: exact ActiveProductDefinitionRefV1
  claim_scope_ref: exact ProductClaimScopeRefV1
  release_requirement_projection_ref:
    exact ProductReleaseRequirementProjectionRefV1
  required_operation_entries[1..512]:
    sorted unique {
      operation_family_kind:
        product_install | product_update | product_repair
        | product_uninstall | project_bootstrap | project_open
        | authoring_author | authoring_preview
        | authoring_validate | authoring_commit
        | project_build | project_test | project_cook
        | project_package | project_launch
        | diagnostics | support
        | pack_acquire | pack_install | pack_apply
        | pack_update | pack_remove
        | ai_read | ai_explain | ai_propose
        | ai_validate | ai_approve | ai_commit
        | security_case_transition
        | security_update | disclosure | incident_response
        | publication | withdrawal,
      operation_ref: exact McdContractRefV1(kind=operation),
      governed_requirement_refs[1..4096]:
        sorted unique exact McdContractRefV1(kind=requirement),
      allowed_surface_kinds[1..7]:
        sorted unique ProductSurfaceKindV1
    }
  projection_algorithm_id: required_product_operation_universe
  projection_algorithm_version: 1
  projection_algorithm_content_hash: SHA-256
  required_operation_universe_content_hash: SHA-256

RequiredProductOperationJourneyProjectionV1
  required_operation_journey_projection_id: StableId
  required_operation_journey_projection_version: 1
  active_product_definition_ref: exact ActiveProductDefinitionRefV1
  claim_scope_ref: exact ProductClaimScopeRefV1
  release_requirement_projection_ref:
    exact ProductReleaseRequirementProjectionRefV1
  required_operation_universe_ref:
    exact RequiredProductOperationUniverseRefV1
  required_journey_entries[1..65535]:
    sorted unique {
      claim_scope_ref: exact ProductClaimScopeRefV1,
      requirement_ref: exact McdContractRefV1(kind=requirement),
      operation_family_kind: operation_family_kind,
      operation_ref: exact McdContractRefV1(kind=operation),
      surface_kind: ProductSurfaceKindV1,
      host_scope:
        {kind=host_independent}
        | {
            kind=host_profile,
            host_profile_ref:
              exact TargetProfileRefV1(
                profile_kind=build_host | editor_host)
          },
      target_scope:
        {kind=target_independent}
        | {
            kind=target_profile,
            target_profile_ref:
              exact TargetProfileRefV1(profile_kind=runtime_target)
          },
      locale_scope:
        {kind=not_applicable}
        | {kind=locale_independent}
        | {
            kind=locale_profile,
            locale_profile_ref: exact LocaleProfileRefV1
          },
      reference_dimension_scope:
        {kind=not_applicable}
        | {kind=dimension_independent}
        | {
            kind=reference_dimension,
            reference_dimension: two_d | three_d
          },
      qualification_scenario_ref: exact QualificationScenarioRefV1,
      semantic_equivalence_group_id: StableId,
      expected_result_branch:
        success | expected_policy_rejection | domain_failure_recovery,
      evidence_class_ref: exact EvidenceClassRefV1
    }
  forbidden_surface_entries[0..65535]:
    sorted unique {
      claim_scope_ref: exact ProductClaimScopeRefV1,
      requirement_ref: exact McdContractRefV1(kind=requirement),
      operation_family_kind: operation_family_kind,
      operation_ref: exact McdContractRefV1(kind=operation),
      surface_kind: ProductSurfaceKindV1,
      host_scope:
        {kind=host_independent}
        | {
            kind=host_profile,
            host_profile_ref:
              exact TargetProfileRefV1(
                profile_kind=build_host | editor_host)
          },
      target_scope:
        {kind=target_independent}
        | {
            kind=target_profile,
            target_profile_ref:
              exact TargetProfileRefV1(profile_kind=runtime_target)
          },
      locale_scope:
        {kind=not_applicable}
        | {kind=locale_independent}
        | {
            kind=locale_profile,
            locale_profile_ref: exact LocaleProfileRefV1
          },
      reference_dimension_scope:
        {kind=not_applicable}
        | {kind=dimension_independent}
        | {
            kind=reference_dimension,
            reference_dimension: two_d | three_d
          },
      qualification_scenario_ref: exact QualificationScenarioRefV1,
      semantic_equivalence_group_id: StableId
    }
  projection_algorithm_id:
    required_product_operation_journey_projection
  projection_algorithm_version: 1
  projection_algorithm_content_hash: SHA-256
  required_operation_journey_projection_content_hash: SHA-256

ProductOperationActivationClosureV1
  operation_activation_closure_id: StableId
  operation_activation_closure_version: 1
  required_operation_universe_ref:
    exact RequiredProductOperationUniverseRefV1
  contract_set_ref: exact ContractSetRefV1
  activated_operation_entries[1..512]:
    sorted unique {
      operation_family_kind: operation_family_kind,
      operation_ref: exact McdContractRefV1(kind=operation),
      owner_manifest_artifact_ref: exact ArtifactRefV1,
      authority_service_ref: exact McdContractRefV1(kind=service),
      service_allowlist_policy_ref: exact McdContractRefV1(kind=policy),
      validator_closure_ref: exact McdContractRefV1(kind=policy),
      diagnostic_refs[1..256]:
        sorted unique exact McdContractRefV1(kind=diagnostic),
      receipt_type_ref: exact McdContractRefV1(kind=type),
      surface_projection_artifact_refs[1..7]:
        sorted unique {
          surface_kind: ProductSurfaceKindV1,
          projection_artifact_ref: exact ArtifactRefV1
        },
      activation_evidence_refs[1..4096]:
        sorted unique exact EvidenceRefV1
    }
  operation_activation_closure_content_hash: SHA-256

ProductProductionActivationSetV1
  production_activation_set_id: StableId
  production_activation_set_version: positive u32
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  candidate_ref: exact PreparedCandidateRefV1
  release_requirement_projection_ref:
    exact ProductReleaseRequirementProjectionRefV1
  supplied_activation_entries[1..4096]:
    sorted unique {
      capability_ref: exact McdContractRefV1(kind=capability),
      target_profile_ref: exact TargetProfileRefV1,
      owner_activation_binding_type_ref: exact McdContractRefV1(kind=type),
      owner_activation_binding_artifact_ref: exact ArtifactRefV1,
      owner_activation_subject_hash: SHA-256
    }
  production_activation_set_content_hash: SHA-256
```

本書Ownerのlocal Refは次のclosed tupleである。

| Ref | Field |
|---|---|
| `TargetProfileRefV1` | `{target_profile_id, target_profile_version, profile_kind, target_profile_content_hash}` |
| `TargetProfileRegistryRefV1` | `{target_profile_registry_id, target_profile_registry_version=1, target_profile_registry_content_hash}` |
| `LocaleProfileRefV1` | `{locale_profile_id, locale_profile_version, locale_profile_content_hash}` |
| `LocaleProfileRegistryRefV1` | `{locale_profile_registry_id, locale_profile_registry_version=1, locale_profile_registry_content_hash}` |
| `ActiveProductDefinitionRefV1` | `{active_product_definition_id, active_product_definition_schema_version=1, active_product_definition_revision, active_product_definition_content_hash}` |
| `ProductClaimKindMinimumRefV1` | `{claim_kind_minimum_id, claim_kind_minimum_version=1, claim_kind_minimum_content_hash}` |
| `ProductClaimScopeRefV1` | `{claim_scope_id, claim_scope_version, claim_scope_content_hash}` |
| `ProductRequirementProjectionInputRefV1` | `{requirement_projection_input_id, requirement_projection_input_version=1, requirement_projection_input_content_hash}` |
| `ProductReleaseRequirementProjectionRefV1` | `{release_requirement_projection_id, release_requirement_projection_version=1, release_requirement_projection_content_hash}` |
| `ProductCompletionRequirementProjectionRefV1` | `{completion_requirement_projection_id, completion_requirement_projection_version=1, completion_requirement_projection_content_hash}` |
| `RequiredProductOperationUniverseRefV1` | `{required_operation_universe_id, required_operation_universe_version=1, required_operation_universe_content_hash}` |
| `RequiredProductOperationJourneyProjectionRefV1` | `{required_operation_journey_projection_id, required_operation_journey_projection_version=1, required_operation_journey_projection_content_hash}` |
| `ProductOperationActivationClosureRefV1` | `{operation_activation_closure_id, operation_activation_closure_version=1, operation_activation_closure_content_hash}` |
| `ProductProductionActivationSetRefV1` | `{production_activation_set_id, production_activation_set_version, production_activation_set_content_hash}` |

<a id="product-profile-identity"></a>

`ProductSurfaceKindV1`は`editor_gui | cli | headless | native_sdk | external_ide | mcp | ai_automation`のclosed Product surface enumである。SDK、MCP、AI automationを一つのAI／tooling surfaceへcollapseせず、個別Agent／versionはProduct surface enumへ追加せずLifecycleのClient ProfileとAI Production OrchestrationのAgent Host Profileで束縛する。

`TargetProfileV1`と`LocaleProfileV1`はProduct-level identityだけのreceipt-free complete recordであり、Toolchain technical fields、OS locale、Localization fallback、Artifact、EvidenceまたはReceiptを含めない。各Refは対応root RegistryからID／version／kind（Targetだけ）／content hashの全Fieldでexactly one recordへ解決し、同ID／version別hash、同ID／hash別kind、language tagだけの一致、表示名、prefix、OS localeまたは`latest`を拒否する。各recordのcontent hashは自己hashだけを除く全Fieldを上表の型固有domain separator、algorithm `sha256`、algorithm version 1、`uint32_be` length framing、schema順でcanonical encodeして計算する。Registryはmember Refをunsigned UTF-8 tuple bytes順にsortし、Registry自己hashだけを除く全Fieldを同じ規則でhashする。現Repositoryにrecord、Registry、resolverまたは生成Artifactは存在せず、本節はtarget contractだけを定義する。

`ProductRequirementCategoryV1`は`capability_activation | lifecycle | security | privacy | legal_ip | license | support | sdk | template | sample | reference | documentation | workflow | pack | developer_testing | packaging | publication | completion`のclosed enumである。`legal_ip`は[Product Legal／IP Governance](../01-governance/product-legal-ip-governance.md)のsame-scope Applicability Profile、全category binding、Domain Evidence、Independent Design Subject、fresh signed Decisionを要求し、`license`、`privacy`または`security`へcollapseしない。`ProductCompletionDistributionScopeKindV1`は`engine_source | host_distribution | target_package | sdk_distribution | project_template | sample_project | documentation_bundle | product_license | support_material | pack_distribution`のclosed enumであり、Product requirementがCompletionで要求する配布classだけを表す。`ProductDistributionArtifactRoleKindV1`は`engine_source | editor_executable | engine_host_runtime | project_launcher | cli_headless_runner | build_tool | installer_layout | target_package | public_header | public_library | sdk_tool | debug_symbol | project_template | sample_project | pack_payload | documentation_bundle | license_text | notice_or_sbom | support_material | publication_metadata`のclosed enumであり、公開route selectorが要求するartifactの配布上の主roleだけを表す。`ProductLifecycleWorkflowKindV1`は`install | project_bootstrap | author | build | test | package | launch | update | repair | diagnose | support | uninstall`のclosed enumであり、Lifecycleのdocumented workflow coverageと同じtoken集合を使う。Lifecycleのexact Distribution Subject identityまたはartifact identityをProduct Planへ逆依存させない。`ActiveProductDefinitionV1`はinitial V1からProductのCapability、Build／Editor Host、runtime Target、locale、required／bundled Pack requirement、Target×2D／3D Reference requirement、claim-facing requirement membership、typed Requirement projection input、Product data-flow membership、claim-kind minimum、required Operation family bindingを閉じるreceipt-free rootである。`host_profile_refs[] ∪ runtime_target_profile_refs[]`は`target_profile_registry_ref`のmembership subset、`locale_profile_refs[]`は`locale_profile_registry_ref`のmembership subsetであり、全member Refを解決してkindとhashをread-backする。三membershipは型でdisjointで、Hostをruntime Targetへ、runtime TargetをHostへ、BCP 47 localeをいずれかのProfileへaliasしない。Activation Receipt、Release Content Manifest、Engine Release、Security／Privacy／Legal／Lifecycle Acceptance、Release Decision、PublicationまたはCompletionを含めない。`active_product_definition_content_hash`は自己hashだけを除く全Fieldを上表のdomain separatorと`uint32_be` length framingでhashする。appendixのBundle／ClosureからRef shapeを補完せず、同ID／revision別hash、同名Definitionまたは`latest`へfallbackしない。

`requirement_projection_input_bindings[]`のRequirement projectionは`claim_facing_requirement_refs[]`とset equalityで、各Requirementへexactly oneの完成`ProductRequirementProjectionInputV1`を束縛する。解決先の`requirement_ref`はbindingとbyte equalityである。この一対一bindingが各Requirementの唯一のProduct category割当であり、同じRequirement Refへ異なる`requirement_category`を持つ第二Input、category alias、Minimum／Projection／consumerによるcategory overrideを禁止する。各`ProductClaimKindMinimumV1.required_requirement_bindings[]`は同じRequirementのInputを解決し、`requirement_ref`と`requirement_category`をともにbyte equalityにしなければならない。

各claim-facing Requirementについて、`prepublication_evidence_class_refs[]`、`completion_evidence_class_refs[]`、全`journey_applicabilities[].evidence_class_refs[]`のcanonical unionはnon-emptyでなければならない。三集合がすべてemptyの「検証不能Requirement」、別RequirementのEvidence classによる補完、global `required_evidence_class_refs[]`だけのnon-emptyを拒否する。`publication_distribution_applicabilities[]`またはnon-`not_applicable`な`completion_distribution_scope_applicability`を持つRequirementは`completion_evidence_class_refs[]`もnon-emptyにする。Field省略、null、空配列、自由文statement、Owner固有payload名からglobal、not applicable、independent、Host／runtime Target／locale applicability、Evidence class、surface、route、workflowまたはCompletion scopeを推測しない。`kind=all_*`、`kind=not_applicable`、`kind=*_independent`、`kind=scope_independent`およびexact branchは相互排他的なclosed branchであり、empty exact setをglobal、independentまたはnot applicableのaliasにしない。Publication selectorのdistribution scope、artifact role、Host／runtime Target／scope-independent execution scope、locale／locale-independent scopeも全Field必須で、platform／channelだけから補完しない。Release、Completion、Required Journey、PublicationおよびLifecycle Workflowの全Named Algorithmはこのtyped inputだけをProduct Requirement意味の入力にする。

Publication selectorの`all_requirement_distribution_scopes`はCompletion distribution scopeが`exact_set`、`all_requirement_hosts`はHost applicabilityがall／exact、`all_requirement_targets`はTarget applicabilityがall／exact、`all_requirement_locales`はlocale applicabilityがall／exactの場合だけvalidである。independent selectorは対応するindependent applicability、exact Host／runtime Target／locale selectorは対応するProduct root membershipのsubsetでなければならない。execution scopeがexact Hostまたはexact runtime Target `p`なら、selectorの`platform_kind`は解決した`TargetProfileV1.product_platform_kind`とbyte equalityでなければならない。`all_requirement_hosts | all_requirement_targets`はapplicable Profile集合を`product_platform_kind`でpartitionし、selectorのPlatformと一致するmemberだけをscalar tupleへ展開する。一致memberが0件のselector、cross-platform Profile、Host／runtime Target kind差を拒否する。`scope_independent`だけはProfileを持たず、このPlatform equalityの対象外である。not-applicable scopeをallへ、independent scopeを一件の既定Profile／localeへ、Hostをruntime Targetへ展開しない。artifact roleは中立closed enumであり、Product Planがexact artifact、path、containerまたはDistribution Subjectを選ばない。

Active Definitionは上記5 `claim_kind`についてexactly oneのMinimumを持つ。各MinimumのHost、runtime Target、locale、Capability、requirement binding、Target×dimension、Operation familyはDefinition内membershipのsubsetでなければならない。各Minimumの`required_requirement_bindings[]`はRequirement Refでstrict uniqueなtotal mappingで、対応する`ProductRequirementProjectionInputV1`のcategoryとbyte equalityにする。同じRequirementを複数categoryへ置くこと、同じcategoryを満たす目的でRequirementのcanonical categoryをMinimum内だけ変更すること、表示labelまたはsemantic groupからcategoryを推測することを拒否する。`engine_2d_3d`と`third_party_product_release`は各required runtime Targetについて`two_d`と`three_d`をexactly oneずつ要求し、異なるTargetまたはdimensionのReferenceで代用しない。`third_party_product_release`はHost minimumへ`target.headless.host`のbuild Hostと`target.windows.editor`のEditor Hostをexactly一件ずつ、locale minimumへcanonical language tag `en-US`と`ja-JP`のexact Locale Profileを含め、`required_requirement_bindings[].requirement_category`のdistinct projectionを全`ProductRequirementCategoryV1`のclosed setとexact set equalityにし、`product_install`から`withdrawal`までの全Operation familyをminimumへ含める。さらに、そのMinimumが選ぶ全Requirementの`workflow_applicabilities[].workflow_kind` distinct projectionを全`ProductLifecycleWorkflowKindV1`とexact set equalityにする。配列下限またはHost×Target展開row数をdistinct workflow kind coverageの代用にせず、12 kindのいずれかがmissing／extra／aliasならMinimumを拒否する。空集合、別claim Minimum、表示上のclaim名、known limitationによるminimum削減を拒否する。

Active Definitionの`required_pack_requirement_refs[]`と`bundled_pack_requirement_refs[]`のcanonical unionは1,024件以下でなければならない。このunionおよび`reference_requirement_bindings[].requirement_ref`のprojectionは`claim_facing_requirement_refs[]`のsubsetであり、Packまたは2D／3D Reference requirementだけをtyped projection inputの外へ隠せない。Requirement projection inputが参照するexact Host／runtime Target／localeはDefinition membership内、Evidence class／scenario／surface／workflow／distribution scope／artifact role enumはclosed branch内でなければならず、required／forbidden surface集合はdisjointとする。`required_product_data_flow_refs[]`はProductが所有またはbundleする全first-party／third-party data flowのreceipt-free membershipであり、optional flowも省略しない。空集合はProductがremote／persisted Product data flowを一件も持たない明示的判断だけに使い、Privacy Acceptance作成者が縮小できない。

Release ProjectionのNamed Algorithm v1は、Claim Scopeの`claim_kind`に一致するexact Minimumを選び、Claim ScopeのHost、runtime Target、locale、Capability、Reference dimension、additional requirement、additional Operation familyがDefinition membership内であることを先に検証する。次にClaim Scopeの各集合がMinimumの対応集合をsupersetとして含むことを要求し、Minimumと許可された追加集合のcanonical unionを生成する。Minimumの各Requirement bindingは同Requirementの唯一のtyped Inputとcategoryを含めてbyte equalityにし、追加Requirementも同じInputのcanonical categoryだけを使う。Evidence class三集合のunionがempty、publication／Completion obligationに対するCompletion Evidence classがempty、category mismatch／alias／overrideまたは`third_party_product_release`のcategory projection不完全ならProjection生成を拒否する。Host、runtime Targetまたはlocaleをintersectionで縮めず、Minimumを欠くscopeをrejectする。required `{Capability, runtime Target}`直積、requirement identity付きacceptance subject、Target×dimension binding、publication distribution subjectをこのunionと各Requirementのexact `ProductRequirementProjectionInputV1`から一意投影する。各acceptance subjectは同じ`{Requirement,Evidence class}` pair固有のHost、Target、locale、Reference dimension applicabilityをclosed branchのまま保持する。配布routeを要求するRequirementはpre-publication acceptanceへ混ぜず、typed inputが所有するPlatform、channel kind、distribution scope、artifact role、Host／runtime Target／scope-independent execution scope、locale／locale-independent scopeをscalar tupleへ完全展開して`required_publication_distribution_subjects[]`へ投影する。`required_evidence_class_refs[]`はtyped inputの`prepublication_evidence_class_refs[]`だけをcanonical unionし、publication／Completion Evidence classはCompletion Projectionへ入れ、Decision入力から追加・削除しない。

Completion Projectionは同じDefinition、Claim Scope、Release Projectionおよび各Requirementのexact typed inputだけから、各completion requirement identityと`completion_evidence_class_refs[]`のordered pairに加え、pair固有のHost、Target、locale、2D／3D Reference Dimension、配布scope kindを`required_completion_bindings[]`へ投影する。各scopeはtyped inputのcategory、closed Host／Target／locale／dimension applicability、Completion distribution scopeとProduct Claim Minimumから決定し、global Claim Scopeを全pairへ無条件複写せず、not-applicable／independent scopeも明示する。同じclassを満たす別requirement、同じrequirementの別class、classだけのset equalityまたは全pairを跨ぐscope unionを代用しない。Release／Completion Decision、Lifecycleのexact Distribution Subject、PublicationまたはEvidence Receiptをhash preimageへ含めない。両Algorithm content hashはAlgorithm ID、version、canonical projection rule document refを束縛し、別Algorithm、versionまたは手入力集合を受理しない。

Projection Capacity Validity Algorithm v1はProjection作成前にchecked arithmeticで次の導出件数を計算し、全条件を同時に満たすClaim Scopeだけをvalidとする。

1. canonical Capability集合件数×runtime Target集合件数は4,096以下で、Host、runtime Target、localeの各rootはそれぞれ64件以下である。
2. `{Requirement, requirement category, prepublication Evidence class, Host scope, Target scope, locale scope, Reference dimension scope}`のdistinct scoped pairは4,096以下。
3. `{Requirement, completion Evidence class}`のdistinct pairは8,192以下。
4. 各Completion pair `p`について、Host、runtime Target、locale、Reference dimensionの各scope cardinality `h(p), t(p), l(p), d(p)`は`exact_set`ならmember件数、independentまたは`not_applicable`なら1件とする。四軸のaxis-wise canonical unionを満たす任意のsatisfiable scalar Evidence集合から、一つのbase rowと各軸の未cover memberごとに高々一rowを選べるため、四scalar軸だけのsound sub-boundを`completion_scalar_evidence_cover_upper_bound(p) = h(p) + t(p) + l(p) + d(p) - 3`とする。全`required_completion_bindings[]`についてこの値をchecked sumした総数は65,535以下でなければならない。これはCartesian productではなく、実Evidence tuple集合の最小coverを`max(h,t,l,d)`と仮定するものでもない。別pairのEvidenceで補完しない。Distribution Subjectは本Projectionが所有するscalar軸ではなく、exact Subject集合がLifecycle join後に確定するため、本値だけを`completion_evidence_bindings[]`の最終carrier上限としない。[Product Publication／Completion](product-publication-completion.md)はexact Subject join後、四scalar軸とSubject unionを同時にcoverする五軸上限をchecked sumし、Completion Decision Subject生成前に65,535以下へ閉じなければならない。
5. fully expanded distinct publication distribution subjectは65,535以下で、各typed inputのapplicability rowは256件以下である。
6. RequirementごとのJourney applicabilityをHost、runtime Target、locale、Reference dimensionのscalar scopeへ完全展開した後、Required Journeyのfull `{Claim Scope,Requirement,semantic group,family,Operation,surface,Host scope,Target scope,locale scope,Reference dimension scope,scenario,branch,Evidence class}` tupleは65,535以下、forbidden full `{Claim Scope,Requirement,semantic group,family,Operation,surface,Host scope,Target scope,locale scope,Reference dimension scope,scenario}` tupleも65,535以下。
7. 各pre-publication acceptance pair `p`について、Host、runtime Target、locale、Reference dimensionの各scope cardinality `h(p), t(p), l(p), d(p)`は`exact_set`ならmember件数、independentまたは`not_applicable`なら1件とする。四軸のaxis-wise canonical unionを満たす任意のsatisfiable scalar Evidence集合から、一つのbase rowと各軸の未cover memberごとに高々一rowを選べるため、carrierへ収容すべきsound cover上限を`prepublication_evidence_cover_upper_bound(p) = h(p) + t(p) + l(p) + d(p) - 3`とする。全`required_acceptance_subjects[]`についてこの値をchecked sumした総数は`ProductReleaseDecisionSubjectV1.supplemental_evidence_bindings[]`の65,535以下でなければならない。Owner aggregateへ解決するpairも同じ上限以内のsatisfying scalar Evidence subsetをread-backし、aggregate名だけで1件へ縮約しない。checked overflow、上限超過、別pair collapseまたはcross-pair unionはProjection生成失敗とする。

overflow、上限超過、truncate、Completion scope aggregate、Capability／Target／locale／Reference dimension／Requirement／Evidence class／surfaceのaggregateまたは別RequirementへのcollapseはProjection生成失敗であり、小さいcarrierへ黙って収めない。これらはProduct DefinitionまたはRequirement個別上限を置換せず、組合せをrelease Claim Scopeとして受理できるかを決めるderived invariantである。

Required Operation UniverseのNamed Algorithm v1は、選択MinimumとClaim Scope追加familyのcanonical unionを取り、Active Definitionの`operation_family_bindings[]`へexactly oneでjoinする。各entryの`governed_requirement_refs[]`はRelease Projectionに現れる同category requirementの完全なsetであり、family、Operation Ref、surfaceまたはrequirementを追加・削除しない。closed family enumはProduct install／update／repair／uninstall、Project bootstrap／open、human author／preview／validate／commit、build／test／cook／package／launch、diagnostics、support、Pack acquire／install／apply／update／remove、Security、Publication、およびAI read／explain／propose／validate／approve／commitをそれぞれ独立identityとして持つ。bootstrapとopen、diagnosticsとsupport、human authoringとAI phase、Packの異なるlifecycle actionを一familyまたは一Operationへcollapseしない。AIまたはMCP clientがcommit、approvalまたはGatewayを迂回せず、GUI、CLI、headless、Native SDK、external IDE、MCP、AI automationは許可されたOperation Refのsurface projectionであって別authorityではない。

Required Operation Journey ProjectionのNamed Algorithm v1は、Required Universeの各non-collapsed `{family,operation}`、同entryのgoverned Requirement、Claim Scope、そのRequirementのtyped `journey_applicabilities[]`から、required journey tupleとforbidden surface tupleを決定論的に生成する。applicability rowのfamilyはUniverse entryとbyte equalityでなければならない。`all_requirement_locales`と`all_requirement_reference_dimensions`は同Requirementのtop-level applicabilityが要求するexact locale／dimension集合へ、exact branchは各memberへ完全展開する。independentとnot-applicableは異なるscalar branchとして一件だけ生成し、空集合、既定locale、既定dimension、Claim Scope aggregate Refへ変換しない。required rowは`{claim scope,requirement,semantic group,family,operation,surface,host scope,target scope,locale scope,reference dimension scope,scenario,expected result branch,evidence class}`を、forbidden rowもbranchとEvidence classを除く同じfull scopeを最後まで保持し、Activation Evidence、workflow表示名、別Requirement、別semantic group、別family、別Operation、別surface、別Host／Target／locale／dimensionまたは別scenarioへcollapseしない。各base `{requirement,semantic group,family,operation,host scope,target scope,locale scope,reference dimension scope,scenario}`について7 surfaceはrequiredとforbiddenのdisjoint unionへ完全分割し、Required Operation Universeの`allowed_surface_kinds[]`外は必ずforbidden、内側もtyped inputが許可しないsurfaceはforbiddenとする。全familyへ7 surfaceを無条件要求せず、`not_applicable`をmissing Receiptとして扱わない一方、forbiddenにない許可surfaceを暗黙省略しない。

Required Journeyのdistinct `{operation_family_kind,operation_ref}` projectionはRequired Operation Universeの同projectionとset equalityである。各Universe pairは少なくとも一つのrequired `success` rowを持ち、governed Requirementのtyped inputが要求するpolicy拒否またはdomain failure／recoveryをそれぞれ`expected_policy_rejection`、`domain_failure_recovery`の独立rowへ投影する。これによりUniverseに存在するがJourneyが0件のfamily／Operationを許可しない。Native SDK、external IDE、MCP、AI automationもapplicableなOperationではrequired rowになり、GUI／CLI／headless成功で代用しない。`semantic_equivalence_group_id`は同じRequirement意味を要求するsurface rowだけを束縛し、同groupではrequest meaning、authorization、Candidate、before／after Project revision、semantic result、typed diagnosticの同値をProduct Lifecycleが検証する。Requirement、semantic group、family、Operation、surface、Host scope、Target scope、locale scope、Reference dimension scope、scenario、branch、Evidence classのmissing、extraまたは一Field substitutionを拒否する。Projectionはreceipt-freeであり、Operation、surface adapter、Fixture、Receipt、Evidence、ActivationまたはQualificationの存在を示さない。

Operation Activation ClosureはRequired Universeの`{family,operation}` projectionとactivated entry projectionをset equalityにし、各entryのsurface projectionをUniverseのallowed surface集合とset equalityにする。Operation MCD、Owner Manifest、Service allowlist、Policy、Validator、Diagnostic、Receipt type、全surface projection、fresh non-revoked Activation Evidenceは同じContract setとOperation hashへ解決する。authoring commandは同じAuthoring Command Gateway、expected Project revision、prepared candidate、Approval、atomic Commit Closureを使い、read／explainはmutation Receiptを発行しない。これらがmaterializeしClosureが成立するまでProduct releaseは不可である。本書の型またはfamily名だけをOperation登録、Activationまたは実装の証拠にしない。

Activation SetはRelease Requirement Projectionの`required_activation_subjects[]`と`supplied_activation_entries[]`の同二Field projectionをset equalityにする。Release／Completion authority OwnerはProjectionのrequired集合とsupplied Evidence／acceptance集合を比較するだけで、inline required集合を持たない。`remaining required`は入力Fieldではなく、canonical `required − satisfied`差分として決定論的に再計算し、approval時にemptyでなければならない。Architecture文書、Decision自身、署名wrapperまたはCompletion recordをrequired Evidenceへ数えない。

`QualificationScenarioRefV1`、`EvidenceClassRefV1`、`EvidenceRefV1`および`QualificationReceiptRefV1`は[AI Verification／Provenance §7](../01-governance/ai-verification-provenance.md#verification-identity-spine)、`OperationReceiptRefV1`は[Executable Contracts §8.0](../02-foundation/executable-contracts.md#operation-receipt-identity)のexact wire tupleとしてopaqueに束縛する。Product Lifecycle、Product Security、Capability Ownerの他Refも解決先Ownerのexact tupleへ従う。本書から下流payloadの意味、class、purpose、owner-specific backing recordを複写または補完せず、Product Planから下流Ownerへの規範依存cycleを作らない。生成順は`Active Product Definition／Claim Scope → Requirement Projection／Required Operation Universe／Required Journey Projection → Release Content Manifest → Engine Release → Lifecycle Journey Evidence／Security／Activation closure → signed Release Decision → Platform publication Receipt → Product Publication → signed Completion`であり、後段Refを前段hash preimageへ埋め戻さない。

## 7. Promotion、deactivation、Product completion

### 7.1 Promotion gate

Capabilityを上位stateへ昇格するには、対象Ownerが定めるContract、Artifact、Target、Toolchain、Policy、Security／Privacy／Legal-IP、license、positive／negative／fault Evidence、known limitation、fallbackまたは明示的non-supportをsame candidateへ閉じる。Legal／IPは対象jurisdiction、market、channel、Product／AI roleへ束縛されたfresh Decisionを要求し、別市場、同じlocaleまたはlicense scannerの結果で代用しない。Evidence requirementの欠落、stale、revoked、scope差は失敗である。

### 7.2 Deactivation gate

重大なcorrectness／security／privacy regression、licenseまたはProvider失効、Target非対応、Evidence期限切れ、support不能では、新規利用、package、claimを即時停止する。既存Projectへの影響、export／migration／rollback可否、User通知、再昇格条件はOwnerとProduct Lifecycleが閉じる。

### 7.3 Release claim gate

Release claimは証明済みscopeを越えず、exact `ProductClaimScopeRefV1`、`ProductReleaseRequirementProjectionRefV1`、`RequiredProductOperationUniverseRefV1`、`RequiredProductOperationJourneyProjectionRefV1`、`ProductOperationActivationClosureRefV1`、[Product Release Decision](product-release-decision.md)の有効な署名済み`ProductReleaseDecisionRecordRefV1`とcurrent authority head、[Product Publication／Completion](product-publication-completion.md)の`state=published`なexact Publication、state authorization、current headを持つ場合だけ公開できる。

| Claim | 必須closure |
|---|---|
| Technology Preview | exact preview limitation、対象Target、期限または終了条件、support boundary |
| 2D First Playable | 2D Reference outcomeと共通Product lifecycle |
| 3D First Playable | 3D Reference outcomeと共通Product lifecycle |
| 2D・3D Engine | 同一releaseにおける2Dと3Dのnon-substituting closure |
| Third-party product release | 公開SDK、Project testing、Documentation、Privacy、License、Security、update／repair／supportを含むE2E acceptance |

Generic Engine releaseを一つのGenre Pack成功だけで主張しない。2Dと3Dの一方、Editorとheadlessの一方、Windowsと別Target、internal testとProject test、SecurityとPrivacyを相互代用しない。Release label、Lifecycle Acceptance、Security Binding、Build成功、unsigned Decision subject、upload／submission Receiptまたはpartial publicationの単独存在はProduct-wide publicationではなく、`decision_state=rejected`または`partially_published`をProduct claimへ読み替えない。

### 7.4 Product completion gate

Product completionはArchitecture文書の完了ではない。required Engine releaseについて次のすべてが成立し、[Product Publication／Completion](product-publication-completion.md)の`completion_state=approved`なSubjectを指す有効な署名済み`ProductCompletionDecisionRecordRefV1`、approved-only `ProductCompletionAuthorityStateRefV1`、State Authorization、exact authority stream keyのcurrent署名済みHeadが揃った時だけ記録できる。

- required CapabilityとTargetのproduction activation。
- 2D／3D Reference、Editor、SDK、Testing、Build／Packageのsame-release E2E acceptance。
- Security、Privacy、same-scope Legal／IP Readiness Decision、SBOM／NOTICE、first-party license、third-party redistribution条件のclosure。
- Documentation、Sample、known limitation、support channel、support window、update／repair／rollbackのclosure。
- required performance、stability、fault、accessibility、locale、clean-machine／device evidence。
- release claimがEvidence scopeを越えず、未対応Futureを明示している。

Completion Requirement Projectionは上記各項目をrequired requirement／Evidence classへ決定論的に投影する。Completion authority recordは`completion_state=approved`、Projectionのrequired集合とsupplied集合のset equality、canonical gap exact empty、rejection reason exact empty、required scopeのProduct Publication `state=published`を要求する。`completion_state=rejected`なRecordは理由付きimmutable audit Decisionであって、canonical gapがemptyでもProduct completion gate、support済みcompletion claim、current Completion Authority Stateまたはcompleted Product stateを満たさない。Release DecisionとCompletion Decisionを同一record、同一hashまたは相互参照にせず、Release authorization、Platform publication、Product Publication、Completionの一方向closureにする。

## 8. Future portfolio

次はProduct requirementではなく、独立したOwner、Threat Model、public boundary、Target、fallback、Evidence、support costを承認して初めてPortfolioへ追加できるFuture subjectである。Architecture上の名称や既存Ownerはactivationを意味しない。

- Online／dedicated server／replication／platform service。
- large world、virtualized geometry、advanced light transportのproduction profile。
- Linux、macOS、Web、Console、XR、specialized device。
- collaborative multi-user authoring、UGC、public Editor extension ecosystem。
- video／media、virtual production、recording／timecode／genlock。
- Account、Cloud、commerce、advertising、live operations。
- Runtime structured-data generationまたはruntime code generation。

現時点でFuture subjectはC1または第三者製品releaseの暗黙要件ではなく、`not_activated`として扱う。Networking Ownerの存在はOnline supportのclaimにならない。Future追加時はProduct claim、Security、Privacy、support、license、Target costへの影響を先に審査する。

## 9. Owner coverageとprimary Product evidence

| Product concern | Primary Owner | Productが要求するclosure |
|---|---|---|
| P0 Architecture membership／16-axis closure | 本[Product Plan §3.3](#p0-canonical-architecture) | fixed root 34件、Product-derived Owner集合、全16 axis、unique Owner／fragment、current state read-back、Future非混入 |
| AI-native Product claim／manual continuity | 本[Product Plan](#11-ai-native-c-product-identity) | shared canonical state、typed proposal／commit flow、C++ Engine authority、Provider不在時のmanual journey |
| AI production Run／Workflow／Context／route | [AI Production Orchestration](../03-authoring/ai-production-orchestration.md) | finite Workflow、immutable Context、single loop owner、surface parity、Run／Task／Commit非代替、Authoring／shipping分離 |
| AI Operation semantics／Provider projection | [Executable Contracts](../02-foundation/executable-contracts.md) | canonical OperationとProvider-neutral projection、current membership、validation／diagnostic |
| AI authority／approval | [AI Security／Approval](../01-governance/ai-security-approval.md) | Task authorization、bounded context、consent、approval、deny／revoke／expiry |
| Product lifecycle／distribution | [Product Lifecycle](product-lifecycle.md) | install、bootstrap、docs、update、repair、support、license presentation |
| Privacy／data | [Product Privacy／Data Governance](../01-governance/product-privacy-data-governance.md) | data inventory、purpose、consent、retention、export／delete、processor／region |
| Security | [Product Security](../01-governance/product-security.md) | threat、secure update、vulnerability response、incident |
| Legal／IP／independent design | [Product Legal／IP Governance](../01-governance/product-legal-ip-governance.md) | jurisdiction／market／channel／role、copyright／trademark／patent FTO／license／AI terms、Independent Design、human signed Decision |
| Project transaction／VCS | [Project State](../03-authoring/project-state.md) | canonical source、revision、merge／recovery、repository interop |
| Editor journey | [Editor Workspace／UX](../03-authoring/editor-workspace-ux.md) | Project Browser、authoring、build／test／debug／profile導線 |
| Gameplay expression | [Gameplay Programming Model](../03-authoring/gameplay-programming-model.md) | structured data／bounded C++／hybridの選択、State／Port、manual／AI proposal parity |
| Asset pipeline | [Asset Lifecycle](../03-authoring/asset-lifecycle.md) | Source、import／reimport、Cook、typed IR、diagnostic、source preservation |
| Public C++ SDK | [Native Game Module](../03-authoring/native-game-module.md) | public API catalog、subject stability、build／link／package evidence |
| Project testing | [Developer Testing](../03-authoring/developer-testing.md) | public suite／case／assertion／runner／result contract |
| Runtime／Simulation | Runtime、Simulation各Owner | 2D／3D meaning、fault、Save／Replay、qualification |
| Rendering | Rendering各Owner | 2D／3D presentation、resource、fallback、qualification |
| Platform | Platform各Owner | package、device、store、permission、accessibility |
| Pack | [Pack Contract](../08-packs/pack-contract.md) | source／publisher trust、dependency、install／update／remove |
| Evidence | [AI Verification／Provenance](../01-governance/ai-verification-provenance.md) | provenance、freshness、revocation、aggregation |

Owner文書にexact SchemaやGateが存在しても、Repository実装またはmaterialized Evidenceがない限りProduct capabilityは利用不能である。本書は各Domainの意味を再定義せず、release claimに必要なclosureを列挙する。

## 10. 外部比較の使用範囲

他Engine、SDK、Store、Platformの比較は、必須journey、failure mode、accessibility、distribution、supportの見落としを発見するために使う。名称、UI配置、API形状、object／Scene model、Plugin形式、workflow、既定値または機能数を模倣せず、公式一次資料で確認した外部事実とMiraikanaiの採用判断を分離する。

外部Engine由来の候補は、Miraikanai側のUser requirement、canonical Owner、public boundary、state／failure／fallback、Target、Evidenceへ独立に導出できた時だけ採択できる。外部Engineの型、API、Scene、Plugin、Editor commandとの一対一alias、同名互換層またはproject importerをinitial V1の近道にしない。共通content formatを採用する場合も、そのformat自身の公式仕様と[Asset Lifecycle](../03-authoring/asset-lifecycle.md)のtyped IR／Conversion Reportへ束縛し、外部EngineのProject object model互換と主張しない。

比較表、market positioning、競合機能一覧はProduct claimのEvidenceにならない。採択しない機能はGapとして無期限に残さず、明示的non-goal、Future subjectまたはOwner付きrequirementのいずれかへ分類する。

## 11. 完了条件

- Product Planが工程、工数、担当、Fixture件数、実行Registry Schemaを所有していない。
- 2Dと3Dの同格性、non-substitution、same-release claim Gateが明示されている。
- 第三者Developer journeyが取得から配布、更新、supportまで閉じている。
- Editor、公開SDK、Project testing、Privacy、License、SupportがProduct release evidenceへ含まれている。
- First Playable、2D／3D Engine、第三者製品releaseのclaimが区別されている。
- ReleaseとCompletionがtyped Decision、required／supplied Evidence set equality、Claim Scope、Production Activation集合へ閉じ、文章だけで自己充足しない。
- Future subjectがC1または現行supportとして誤読されない。
- AI-native claimがshared canonical state、C++ Engine authority、typed proposal／validation／approval／atomic Commit、Evidence、manual continuityへ閉じ、AI機能の存在だけで自己充足しない。
- P0が実装PhaseではなくActive Product Definition由来のOwner集合として閉じ、固定root 34件、Product-derived membership、16 axis、unique Owner／fragment、current stateを一つのSpecificationへ束縛している。
- Legal／IPをSecurity、Privacy、LicenseまたはAI生成Provenanceへcollapseせず、Release法域、market、channel、role、Independent Designとauthorized human Decisionへ閉じている。
- 汎用Engineのminimum surfaceがOwnerへ一意に接続され、外部Engineの型／API／Scene／Plugin／workflowを模倣または互換層として持ち込まない。
- Architectureの存在を実装、Activation、QualificationまたはProduct completionと表現していない。
