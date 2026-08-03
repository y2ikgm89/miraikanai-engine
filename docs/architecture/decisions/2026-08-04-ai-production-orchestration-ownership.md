# AI Production Orchestration Ownership

- 文書ID: mirakan.decision.ai-production-orchestration-ownership
- 状態: review
- 正本範囲: AI制作Run／Workflow／Context／route／Agent-loopの一意Owner追加、既存authority分散の維持、既存Ownerへの追記だけまたはAI文書全面再編を採用しない理由
- 非正本範囲: current Schema／Operation／Registry／Fixture／Receipt、Product claim、AI authorization／Approval、Evidence、Project transaction、Game production payload、Editor UI、Host／Process／binary、Provider／Model／SDK version、Runtime package、実装Task／実装順序／工程／工数／担当。current契約は各Owner文書を参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core Architecture](../02-foundation/core-architecture.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Project State](../03-authoring/project-state.md)、[Game Production Loop](../03-authoring/game-production-loop.md)、[Editor Workspace／UX](../03-authoring/editor-workspace-ux.md)、[AI Production Orchestration](../03-authoring/ai-production-orchestration.md)
- 外部根拠検証日: 2026-08-04
- 文書種別: Architecture Decision／owner allocation and control-plane boundary
- Decision owner document: `mirakan.arch.architecture-governance`
- Decision日: 2026-08-04
- Supersedes: none

## Context

Miraikanaiは既に、AI Task Authorization／Provider trustをAI Security、Evidence／ProvenanceをAI Verification、typed OperationをExecutable Contracts、Project transactionをProject State、Game理解／PlaytestをGame Production Loop、AI Partner UIをEditor Workspaceへ分離している。この分離はauthorityの一意性を保つ一方、AI制作を実際に進行させるRun、Workflow、immutable Context、execution route、Agent loop、checkpoint／resume、surface parityのOwnerが存在しなかった。

そのため、既存文書の`AI Orchestrator` actor、Provider caller route、Editor interaction mode、Game production operation laneを組み合わせても、次を一意に答えられなかった。

- 一つのRunを誰が開始・停止・再開し、どのstateで追跡するか。
- first-party Agentとexternal AgentのどちらがAgent loopを所有するか。
- Workflow、Skill、Plugin、MCP、vendor sessionのどれが実行意味を所有するか。
- ContextがProject revision、Authorization、Provider policy、budgetへどう固定されるか。
- Run completionとTask／Commit／Approval／Evidence／Release completionをどう分離するか。
- local／cloud fallback、child Run、retry、cancel、crash recoveryをどのOwnerが閉じるか。

このgapは機能一覧の不足ではなく、control-plane semanticsのOwner欠落である。既存Ownerへ同じ定義を分散追記すると、Task state、Provider route、Editor state、Game loopが相互にshadow definitionを持つ。

## Decision drivers

1. AI Security、Evidence、Project transaction、Game domain、Editor UIの既存authorityを移動または複製しない。
2. built-in AI、CLI、Native SDK、MCP、外部Agentを同じEngine-owned Operationへ収束させる。
3. deterministic Automation、first-party Agent、standard external Agent、managed external Hostのloop ownerをexactly oneにする。
4. WorkflowをSkill／Plugin／Prompt／vendor sessionから分離し、finite boundとtyped resultを持たせる。
5. Run completionをCommit、Approval、Qualification、ReleaseまたはProduct completionへ読み替えない。
6. Authoring AIをRuntime packageおよび将来shipping generative AIから分離する。
7. current implementation／Schema／Registry／Operationが存在しない状態を維持し、initial V1へlegacy aliasを導入しない。

## Considered options

### A. 既存Ownerの関連参照だけを追加する

却下する。Run lifecycleはAI Security、WorkflowはExecutable Contracts、routeはProvider supplement、ContextはProject State、AI PartnerはEditorという複数の候補Ownerに分散したままになり、どのstateがcanonicalかを決められない。変更量は小さいがroot causeを閉じない。

### B. AI SecurityへRun／Workflow／routeを集約する

却下する。Authorization、Risk、Credential、ApprovalというGovernance authorityと、Authoring control flowを同じOwnerへ混ぜる。Workflow completionまたはroute selectionから権限を推測しやすくなり、Security文書の責務が過大になる。

### C. AI関連Owner群を全面再編し、Security／Verification／Editor／Game productionも移管する

却下する。既に閉じているauthority、Evidence、Project transaction、Game domain、UI契約を同時に移動し、規範依存cycle、stale reference、二重Ownerを増やす。control-plane gapを超える変更である。

### D. Authoring層へ`AI Production Orchestration` Ownerを一件追加する

採用する。新OwnerはRun、Workflow、Context、execution route、loop ownership、checkpoint／resume、surface parityだけを所有し、既存Ownerのauthorityをexact refとして消費する。Game Production LoopとEditor Workspaceだけを直接Authoring consumerにし、上位Product／Governance／Foundation Ownerからの逆向き規範依存を作らない。

## Decision

1. `mirakan.arch.ai-production-orchestration`をAuthoring Ownerとして追加する。
2. 新Ownerのdirect normative dependencyはArchitecture Governance、Product Plan、AI Security、AI Verification、Core Architecture、Toolchain、Executable Contracts、Project Stateのexact 8件とする。
3. Game Production LoopとEditor Workspace／UXは新Ownerへdirect normative dependencyを持つ。他の既存Ownerは関連参照とlocal authority boundaryだけを追加する。
4. 新Ownerは`AiProductionRunV1`、Attempt／Outcome、Transition／Head、Result、Checkpoint、Workflow Definition／Registry／Binding、Run Context、Route Selectionのtarget意味を一意所有する。
5. production routeは`deterministic_automation | first_party_agent | standard_external_agent | managed_external_host`のclosed setとし、各Runはexactly one control-loop ownerを持つ。Model routeはexactly one Agent-loop ownerを持つ。
6. AI Securityのcaller route `engine_provider_adapter | standard_external_mcp | managed_external_host`はHost／Transport／Provider security unionとして保持し、production routeとaliasしない。両者はexplicit mappingで接続する。
7. Workflowはimmutable typed definition、Skillはguidance、Pluginはdistribution、MCPとvendor SDK／App ServerはAdapterとする。いずれもOperation、Authorization、Project authorityまたはEvidence passを生成しない。
8. Run `completed`はtyped Result発行だけを意味する。Task completion、Project Commit／Promotion、Approval、Qualification、ReleaseおよびProduct completionは各Ownerの別state／Record／Operationとする。
9. built-in AI Consoleはoptional first-party Clientとし、private writerまたはAI専用Project formatを持たない。AIが利用不能でもmanual Editor／CLI／SDK journeyを維持する。
10. Authoring Orchestrator、Agent Host、Credential、Workflow／Run Store、MCP Server、Compiler、Signer、write Gatewayをshipping Runtime packageへ含めない。将来shipping generative AIは独立Future Owner／Decisionを必要とする。
11. current Repositoryにはmaterialized type、Registry、Operation、Service、Fixture、Receiptまたはimplementationがないため、全current集合をempty／absentのまま保持する。initial V1でlegacy session、alias、dual reader、migration Operationを作らない。

## Consequences

- AI制作のcontrol flowとauthorityの境界を一意に説明できる。
- Security Task、Production Run、Operation Task、Project ChangeSetのstateを混同できない。
- first-partyとexternal Agentのnested loop、silent Provider fallback、conversation resumeによる権限継続を禁止できる。
- Workflow／Context／route／RunのSchema、Registry、Operation、Service、Fixture、Receiptを将来materializeする必要が残る。本Decisionはその実装または実装計画を定めない。
- Game Production LoopとEditorは新Ownerのexact refを消費するため更新が必要になるが、Intent／Playtest payloadとPanel presentationのownershipは維持される。
- Performance、Debugging、Runtime Packageはcapacity、causal projection、shipping exclusionをそれぞれ保持し、新Ownerへ移管しない。
- external AgentのMCP以外のTransport、first-party local inference、managed external Host、shipping generative AIはcurrent Capabilityへ昇格せず、各専用security／toolchain／runtime closureが必要になる。

## Canonical Owner documents

- Run、Workflow、Context、route、loop、checkpoint、surface parity: [AI Production Orchestration](../03-authoring/ai-production-orchestration.md)
- Product claim、manual continuity、MVP／Future: [Product Plan](../00-product/product-plan.md)
- Authorization、Risk、Approval、Credential、Provider／MCP security: [AI Security／Approval](../01-governance/ai-security-approval.md)
- Evidence、Eval、Provenance、freshness: [AI Verification／Provenance](../01-governance/ai-verification-provenance.md)
- MCD、Operation、Diagnostic、projection: [Executable Contracts](../02-foundation/executable-contracts.md)
- Host／Process／IPC: [Core Architecture](../02-foundation/core-architecture.md)
- dependency artifact pin: [Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)
- Project revision／ChangeSet／Commit／Undo: [Project State](../03-authoring/project-state.md)
- Intent／Brief／Playtest／Iteration: [Game Production Loop](../03-authoring/game-production-loop.md)
- AI Partner presentation: [Editor Workspace／UX](../03-authoring/editor-workspace-ux.md)
- capacity／measurement: [Performance／Capacity](../04-runtime/performance-capacity.md)
- causal projection／export: [Debugging／Observability／Replay](../04-runtime/debugging-observability-replay.md)
- shipping package／launch: [Runtime Package](../04-runtime/runtime-package.md)

## Supersedes／Superseded by

- Supersedes: none
- Superseded by: none

## Official or primary sources

外部資料はprotocolとvendor extensionの分類確認に用いる。新Owner方式はMiraikanaiのproject-decisionであり、外部組織の公式推奨ではない。

- [ISO/IEC/IEEE 42010:2022](https://www.iso.org/standard/74393.html)
- [Model Context Protocol 2026-07-28 specification](https://modelcontextprotocol.io/specification/2026-07-28)
- [MCP versioning](https://modelcontextprotocol.io/docs/2026-07-28/learn/versioning)
- [OpenAI Plugin architecture](https://developers.openai.com/plugins/concepts/plugins)
- [OpenAI Build skills](https://developers.openai.com/plugins/build/skills)
- [OpenAI Codex App Server](https://learn.chatgpt.com/docs/app-server)
- [Anthropic Claude Code features overview](https://code.claude.com/docs/en/features-overview)
- [OpenTelemetry specification](https://opentelemetry.io/docs/specs/otel/)

このDecisionはOwner追加と採用理由だけを記録する。current type、state、route、Workflow、Run behavior、Operation、authorityおよびfailureは各Owner文書を正本とし、実装計画を定めない。
