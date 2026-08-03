# AI-native C++ Product Identity

- 文書ID: mirakan.decision.ai-native-cpp-product-identity
- 状態: review
- 正本範囲: AI-native C++ Game EngineというProduct identity、外部Engine比較の非模倣境界、initial V1 clean-breakの採用理由
- 非正本範囲: current Schema／Operation／Registry／Fixture／Receipt、AI権限、Project transaction、Gameplay type、Native ABI、Editor UI、Asset format、Target、実装Task、実装順序、担当、工数、日程。各Owner文書を参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Product Plan](../00-product/product-plan.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[Core Architecture](../02-foundation/core-architecture.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Project State](../03-authoring/project-state.md)、[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)、[Native Game Module](../03-authoring/native-game-module.md)、[Editor Workspace／UX](../03-authoring/editor-workspace-ux.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)
- 外部根拠検証日: 2026-08-03
- 文書種別: Architecture Decision／product identity and independent design boundary
- Decision owner document: `mirakan.arch.architecture-governance`
- Decision日: 2026-08-03
- Supersedes: none

## Context

Unity、Unreal Engine、Godotの公式資料は、Editor、2D／3D、Asset import、C++／Script、test、build／package、extension等が第三者向けGame Engineの製品surfaceとして存在することを確認する比較材料になる。一方、それらのobject model、Scene、API、Plugin、UIまたはworkflowは各製品の設計であり、Miraikanaiの公式要件でも実装templateでもない。

Miraikanaiには既にcanonical Project state、typed Operation、C++ Engine authority、bounded AI approval、構造化Gameplay data、C++23、Pack、Native Game ModuleのOwnerがある。しかし「既存EngineにAIを付加した製品」と「AIと人間が同じ検証可能なcontractを通るAI-native Engine」の区別、および汎用Engineの基本journeyを保ちながら外部Engineを模倣しない採用原則は、一つのProduct identityとして明示する必要がある。

## Decision drivers

1. AIを特権WriterまたはProvider固有Tool Schemaへ閉じず、C++ Engineの権威判断と分離する。
2. AI不在でも第三者Developerが同じProjectをEditor、CLI、IDEから保守できる。
3. C++23と構造化dataを一つのGameplay modelとして閉じ、汎用Script VM／JITを逃げ道にしない。
4. 汎用Game Engineの製品journeyを満たしつつ、外部Engineの型、API、Scene、Plugin、UI、workflowを複製しない。
5. public materialization前のinitial V1を過去draftや外部Engine互換層で汚さず、public release後の第三者Sourceとconsumerはversioned evolutionで保護する。

## Considered options

### A. Unity／Unreal Engine／Godotとのfeature parityと互換性をProduct identityにする

却下する。機能数、名称、一対一APIまたはProject importerがOwnerとUser requirementを置き換え、外部製品の制約、互換負債、creative expressionをMiraikanaiへ持ち込む。比較対象のrelease差によってProduct identityも不安定になる。

### B. 通常のC++ Game Engineを先に定義し、Chatまたはcode generationを付加する

却下する。AIとmanual surfaceが別state、別authorityまたは別failure semanticsを持ちやすく、AIの出力を後付けautomationとして特別扱いする。AI不在時のcontinuityと、proposalからCommit／Evidenceまでの同一契約をProduct要件として保証できない。

### C. AI Provider／MCP Tool Schemaをcanonical Engine APIにする

却下する。Providerまたはprotocol versionがProject authorityを所有し、manual surfaceとEngine contractがprojectionへ従属する。Provider停止、version非互換、権限誤設定がProject formatまたはEngine semanticsを変え得る。

### D. Provider-neutralなtyped contract coreを中心とするAI-native C++ Engine

採用する。Project state、Operation、Validation、Approval、Commit、EvidenceはEngine-owned canonical contractとし、Editor、CLI、IDE、AI Providerは同じcontractのsurface／projectionにする。汎用Engineの製品surfaceはUser journeyから独立導出し、外部Engine資料はgap discoveryだけに使う。

## Decision

1. `AI-native` Product claimの正本は[Product Plan](../00-product/product-plan.md)とする。AIと人間は同じcanonical Project state、typed proposal、semantic diff、Validation、Approval、atomic Commit、test／build、Evidence／explanation／recovery経路を使い、C++ Engineだけがmutationの権威判断を行う。
2. AI ProviderとMCPはcanonical Engine Operationのversioned projectionであり、Source of Truth、permission source、Project memoryまたはsuccess authorityではない。Provider不在、拒否、timeout、失効または非互換時はfail closedにし、manual Editor／CLI／IDE journeyを維持する。
3. Gameplayは[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)の構造化data、bounded Project C++、hybridを用いる。AI生成C++も通常のSource、Owner、review、build、test、security、license、provenanceへ従い、汎用Script VM、JIT、download code、AI専用Runtimeを導入しない。
4. 汎用Game Engineのminimum surfaceはProduct journeyから導出する。外部Engine公式資料はsurface categoryとfailure modeを発見する入力に限定し、型、API、Scene model、UI layout、名称、Plugin形式、workflow、default、assetまたはcreative expressionをコピーしない。
5. 採用する各subjectはMiraikanaiの一意なOwner、User requirement、public boundary、state、failure、fallback、Target、Evidenceを持たなければならない。外部Engine機能名との対応表、market parityまたはAI生成結果をCapability Evidenceにしない。
6. initial V1は[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)のdirect-definition boundaryを適用し、過去draftまたは外部Engineのalias、dual reader、migration、Project／Scene／Plugin互換層を持たない。初回public materialization後は実在する第三者consumerをInventoryし、clean breakを自動適用しない。
7. Online、large world、Linux／macOS／Web／Console／XR、public Editor extension ecosystem等はProduct PlanのFutureに維持する。本Decisionはそれらをcurrent scope、required Work PackageまたはActivationへ昇格しない。

Product-facing flowは`discover／query -> bounded context -> question／decision（必要時） -> typed proposal -> semantic diff -> validation -> approval -> atomic commit -> test／build -> receipt／explain／recovery`とする。`undo`は対象Ownerがreversibleと定義したOperationだけに許し、不可逆な外部effectには事前Approval、明示的boundary、補償またはrecovery policyを要求する。これは関係するOwnerを接続するProduct意味であって、Operation名、Schema、Registry、Fixture、実装順序またはActivationを本Decisionへ集約しない。

## Consequences

- AI機能の数、Model性能、Prompt例、Chat UIまたはProvider接続だけではAI-native claimを満たさない。
- manual authoringとAI authoringに意味の異なるProject format、private writer、validation bypassまたは結果の推測fallbackを持てない。
- 外部Engineとの比較は継続できるが、採用時にはMiraikanai固有Ownerへ独立導出した理由と境界が必要になる。
- public Editor extension marketplaceや汎用Script runtimeがなくても、bounded Pack／Native Game Moduleと公開C++ SDKのProduct closureを満たせばcurrent汎用Engine claimを評価できる。Future subjectの不在を実装済みと扱わない。
- RepositoryにはEngine implementation、materialized Schema、Operation、Registry、Fixture、Receipt、Build、QualificationまたはReleaseが存在せず、本DecisionはProduct identityのtarget designだけを記録する。

## Canonical Owner documents

- AI-native claim、第三者journey、minimum surface、Future: [Product Plan](../00-product/product-plan.md)
- Decision分類、Owner、文書状態: [Architecture Governance](../01-governance/architecture-governance.md)
- initial V1 clean-break、公開後migration: [Compatibility／Evolution](../02-foundation/compatibility-evolution.md)
- canonical OperationとProvider projection: [Executable Contracts](../02-foundation/executable-contracts.md)
- Project revision、ChangeSet、atomic Commit、recovery: [Project State](../03-authoring/project-state.md)
- AI authority、bounded context、approval、consent: [AI Security／Approval](../01-governance/ai-security-approval.md)
- Evidence、provenance、freshness、understanding Eval: [AI Verification／Provenance](../01-governance/ai-verification-provenance.md)
- gameplay表現選択とAI proposal parity: [Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)
- 公開C++ SDK／ABI／bounded native extension: [Native Game Module](../03-authoring/native-game-module.md)
- Editorのmanual／AI interaction: [Editor Workspace／UX](../03-authoring/editor-workspace-ux.md)
- Asset Source／import／typed IR／Conversion Report: [Asset Lifecycle](../03-authoring/asset-lifecycle.md)

## Supersedes／Superseded by

- Supersedes: none
- Superseded by: none

## Official or primary sources

外部資料は第三者向けEngine surfaceとprotocol／language事実の確認にだけ使用し、Miraikanaiの採用判断を「外部組織の公式推奨」と表現しない。

- [ISO/IEC 14882:2024 publication](https://www.iso.org/standard/83626.html)
- [Model Context Protocol versioning](https://modelcontextprotocol.io/docs/2026-07-28/learn/versioning)
- [Unity Manual](https://docs.unity3d.com/Manual/UnityManual.html)
- [Unity 6 releases and support](https://unity.com/releases/unity-6/support)
- [Unity Scripted Importers](https://docs.unity3d.com/Manual/ScriptedImporters.html)
- [Unity Test Framework](https://docs.unity3d.com/Manual/com.unity.test-framework.html)
- [Unreal Engine Blueprint versus C++](https://dev.epicgames.com/documentation/en-us/unreal-engine/coding-in-unreal-engine-blueprint-vs-cplusplus)
- [Unreal Engine Packaging Projects](https://dev.epicgames.com/documentation/en-us/unreal-engine/packaging-your-project)
- [Unreal Engine Plugins](https://dev.epicgames.com/documentation/en-us/unreal-engine/working-with-plugins-in-unreal-engine)
- [Godot feature list](https://docs.godotengine.org/en/stable/about/list_of_features.html)
- [Godot editor plugins](https://docs.godotengine.org/en/stable/tutorials/plugins/editor/making_plugins.html)
- [Godot GDExtension system](https://docs.godotengine.org/en/stable/engine_details/engine_api/gdextension/index.html)

このDecisionは採用理由だけを記録する。current型、値、Operation、Gate、Authority、Runtime behaviorまたはCompatibility policyは各Owner文書を正本とし、実装計画を定めない。
