# AI Production Orchestration Architecture Refresh Design

- 文書状態: proposal
- 実装状態: absent
- 正本性: non-normative design input
- 対象: Miraikanai EngineのAI制作実行、Workflow、Agent／Model経路および既存Architecture Owner群の責務再配置
- 非対象: C++実装、実装計画、工程、工数、担当、MCD Schema file、Registry、Fixture artifact、Receipt、Build、CI、物理process名の固定
- 先行設計: [AI-native Production Loop Architecture Reconstruction Design](2026-08-03-ai-native-production-loop-architecture-reconstruction-design.md)
- 外部根拠確認日: 2026-08-04

## 1. 結論

Miraikanaiは、`AI Orchestrator`、Provider／MCP security、AI Partner UI、Game production loopを既存Ownerへ分散したままにせず、AI制作RunとWorkflowの意味を一意所有する新しいAuthoring Owner `AI Production Orchestration`を追加する。

新Ownerは、決定論的Automation、任意のfirst-party Agent、外部Agent、local／cloud Model経路を同じProject、Operation、Authorization、Evidenceへ収束させる。AI Model、Agent Host、MCP、Skill、Plugin、conversationまたはEditor widgetはProject authorityを持たない。正規変更は引き続きC++ Engine-owned Gateway、typed Operation、Project ChangeSet、Validation、Approval、Commit／Promotionだけを通る。

本刷新はArchitecture target designだけを変更する。RepositoryにAI制作Service、Model runtime、MCP Server、Schema、Registry、Fixture、Receipt、BuildまたはCIが存在しないため、実装、materialization、Qualification、Activation、ReleaseまたはProduct completionを主張しない。

## 2. 監査対象と判定方法

監査inputは次である。

1. ChatGPT conversation `6a7099b0-c8f8-83e8-92fa-2d345df1db44`の全6 messageと、同conversationで提示された40分類のCapability Inventory。
2. 現在のArchitecture Indexに登録された64 Ownerと、その規範依存、関連文書、補助文書、ADR。
3. 2026-08-03のAI-native production loop再構築設計と、その後のArchitecture closure記録。
4. MCP、OpenAI Codex／Plugin／Skill、Anthropic Agent extension、ONNX Runtime GenAI、OpenTelemetryの現行一次資料。

conversationはuntrusted design inputであり、公式仕様、Architecture authorityまたはRepository evidenceではない。各提案は、既存Ownerとの重複、規範依存方向、Trust boundary、現在のmaterialization stateおよび一次資料を個別に照合して採否を決める。conversation transcript自体を正本へ保存しない。

Capability Inventoryの機能分類は、Product、Foundation、Authoring、Runtime、Simulation、Rendering、Platform、Pack、Networkingの既存Ownerでcategory-levelには覆われている。今回のgapは機能一覧の不足ではなく、AI制作を実行するcontrol flow、Agent loop、Workflow、ContextおよびModel routeに一意Ownerがないことである。

## 3. 外部事実とMiraikanai判断

### 3.1 外部事実

- [ISO/IEC/IEEE 42010:2022](https://www.iso.org/standard/74393.html)はArchitecture Descriptionのconceptとrelationshipを扱うが、Miraikanai用の唯一のMarkdown文書様式またはGame Engine内部構成を定めない。
- [MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28)は、LLM applicationと外部data／toolの接続、stateless self-contained request、per-request capability negotiationを定める。[Versioning](https://modelcontextprotocol.io/docs/2026-07-28/learn/versioning)は`2026-07-28`をcurrent revisionとし、Tasks、Skills over MCP、MCP Apps等をopt-in extensionとして分離する。
- [OpenAI Plugin architecture](https://developers.openai.com/plugins/concepts/plugins)はPluginをSkill、MCP Server、optional UIの配布packageとして扱う。[Build skills](https://developers.openai.com/plugins/build/skills)はSkillをrepeatable workflow guidance、MCP Serverをlive data、authentication、authorization、controlled actionの境界として分離する。
- [Codex App Server](https://learn.chatgpt.com/docs/app-server)はCodexのauthentication、conversation history、approval、streamed agent eventを製品へ統合するvendor surfaceであり、汎用Automation contractではない。
- [Claude Code extension model](https://code.claude.com/docs/en/features-overview)はSkill、MCP、subagent、hook、pluginをAgent loopへ接続する異なる拡張点として列挙する。これらをMiraikanaiのinstruction／security boundaryへどう割り当てるかは下記project-decisionである。
- [ONNX Runtime GenAI](https://github.com/microsoft/onnxruntime-genai)はlocal inferenceのC／C++を含む実行候補を提供するが、pre-1.0で継続的に変化している。特定runtimeをEngine正本へ埋め込む根拠にはしない。
- [OpenTelemetry specification](https://opentelemetry.io/docs/specs/otel/)はtrace、metric、log等のobservability signalと共通Contextを提供するが、MiraikanaiのTask、Project、Authorization、EvidenceまたはWorkflow意味を所有しない。

### 3.2 Miraikanaiのproject-decision

1. Engine Automation Contractを正本にし、MCP、Native SDK、CLI、Provider projectionをAdapterとする。
2. Skillはguidance、Pluginはdistribution、PromptはModel inputであり、権限、状態遷移、合否、完了またはWorkflow正本にしない。
3. AI制作RunとWorkflowの意味を新Ownerへ集約し、Security、Project、Editor、Game productionの意味を複製しない。
4. 内蔵AIと外部Agentを同じsemantic Operationへ収束させるが、Agent loopの所有者をrouteごとにexactly oneへ固定する。
5. first-party local inferenceは将来の任意Capabilityのままにし、MVPまたはFirst Playableの必須条件にしない。
6. initial V1はmaterialized legacy consumerがないためclean breakとし、旧candidate名、alias、dual readerまたはmigration Operationを作らない。

これらをOpenAI、Anthropic、Microsoft、MCPまたはISOの公式推奨とは表現しない。

## 4. 比較した刷新方式

### 4.1 採用: 新しいAuthoring Ownerを一件追加する

`docs/architecture/03-authoring/ai-production-orchestration.md`を追加し、AI production Run、Workflow、Context、route、loop ownership、recoveryを一意所有する。既存Ownerは自身のauthority、contract、state、UI、game meaning、dependency pinだけを保持する。

この方式は、欠けている意味だけを補い、閉じている64 Ownerの責務を全面移動しない。新Owner追加後の想定Graphは65 Owner、332規範依存edge、missing target 0、cycle 0である。

### 4.2 不採用: 既存Ownerへの相互参照追加だけで済ませる

変更量は小さいが、AI OrchestratorのRun semanticsがCore、Security、Editor、Provider supplementへ分散したままになり、実装時にAgent loop、retry、context、fallbackのOwnerを一意に決定できない。

### 4.3 不採用: AI関連文書群を全面再編する

Security、Verification、Editor、Game production、Provider candidateを同時再配置すると、既に閉じているauthority、Evidence、Project transactionおよびUI契約まで移動し、正本重複と参照破損を増やす。今回のroot causeを超えるため採用しない。

## 5. 新Ownerのidentityと責務

### 5.1 Header設計

- 文書名: `Miraikanai Engine AI Production Orchestration Contract`
- 文書ID: `mirakan.arch.ai-production-orchestration`
- 配置: `docs/architecture/03-authoring/ai-production-orchestration.md`
- 文書状態: `review`
- 実装状態: `absent`
- 検証状態: `design-reviewed`
- 根拠区分: `project-decision`。外部仕様を引用する箇所だけ`official-spec`
- 外部根拠確認日: `2026-08-04`

### 5.2 正本範囲

- AI production Run、Attempt、Result、Checkpointの意味とlifecycle
- AI Workflow Definition、binding、step semantics、bound、stop／failure規則
- Run Contextのassembly、immutability、lineage、expiry、redaction、Project authority非代替
- deterministic Automation、first-party Agent、standard external Agent、managed external Hostのroute意味
- Agent loopの単一所有、nested loop禁止、delegation／child Run bound
- Provider-neutral inference route selectionと明示fallback要求
- built-in AI Console、CLI、Native SDK、MCP、外部Agentのsemantic parity
- interaction intentとbounded workspace execution profileの分離
- conversation／sessionとProject／Task／Run状態の非代替
- Authoring AIとshipping Runtime AIの非代替境界
- 上記を扱うtarget Operation familyの意味とcurrent `not_activated`境界

### 5.3 非正本範囲

- Product claim、Capability portfolio、MVP、First Playable、Release／Completion: Product Plan／Lifecycle／Release Owners
- Task Authorization、Risk、Approval、Consent、Credential、Trust、Provider／MCP security: AI Security／Approval
- Evidence、Eval、Provenance、freshness、Trace grading: AI Verification／Provenance
- MCD common kind、Operation envelope、Diagnostic、projection: Executable Contracts
- Host／Process／IPC、Worker、Gateway配置: Core Architecture
- Project revision、Document header、ChangeSet、Commit、Undo／Recovery: Project State
- Game Brief／Spec、Question、Playtest、Evaluation、Iteration: Game Production Loop
- Panel、Workspace、visual state、Accessibility: Editor Workspace／UXおよびEditor UI Framework
- external SDK／runtime／model artifactのversion、hash、license、取得元: Toolchain／Dependencies
- CPU／GPU／RAM／VRAMのsystem capacityと測定: Performance／Capacity
- Runtime package内容、launch、shipping eligibility: Runtime Package
- deterministic gameplay AI、NPC behavior、Pathfinding: Gameplay、Navigationおよび各Domain Owner
- 将来のshipping generative Runtime AIのdomain contract: 将来の独立Owner／ADR
- 実装構造、binary名、service数、queue／thread実装、工程、工数、担当

### 5.4 規範依存

新Ownerの規範依存は次のexact 8件とする。

1. Architecture Governance
2. Product Plan
3. AI Security／Approval
4. AI Verification／Provenance
5. Core Architecture
6. Toolchain／Dependencies
7. Executable Contracts
8. Project State

Game Production LoopとEditor Workspace／UXを新Ownerの直接consumerにする。上位Product、Governance、Foundation Ownerから新Ownerへの逆向き規範依存を作らない。Performance、Runtime Package、Debugging／Observabilityは関連文書とし、各Ownerの型または意味を新Ownerへ複写しない。

## 6. 一意所有表

| concern | canonical Owner | 新Ownerが行うこと |
|---|---|---|
| Product AI-native claim | Product Plan | required orchestration Capabilityの参照だけを提供 |
| Task authority、Risk、Approval | AI Security／Approval | exact Authorization refを消費し、拡張しない |
| Evidence、Eval、Provenance | AI Verification／Provenance | required Evidence refとfreshness resultを消費 |
| MCD、Operation、Diagnostic | Executable Contracts | registered memberだけをWorkflow stepへ束縛 |
| Host／Process／Gateway | Core Architecture | logical role requirementを示し、物理配置を決めない |
| Project revision／Commit | Project State | ProposalとRun lineageをexact revisionへ固定 |
| Intent／Brief／Playtest | Game Production Loop | domain Workflowのtyped input／resultを参照 |
| AI Partner UX | Editor Workspace／UX | Run／Workflow／Context／Resultのread projectionを提供 |
| Provider／MCP security profile | AI Security supplement | current governed bindingをroute inputとして消費 |
| dependency pin | Toolchain／Dependencies | selected artifact bindingを消費し、versionを定義しない |
| resource capacity | Performance／Capacity | AI固有request budgetを宣言し、capacity判定を上書きしない |
| Runtime package | Runtime Package | Authoring AI exclusion requirementだけを提供 |

## 7. 論理Architecture

新Ownerは次のlogical roleを区別する。これはbinary、processまたはDirectory名ではない。

```text
Editor / CLI / Native SDK / External Agent
                    |
                    v
          Client Surface Adapter
                    |
                    v
       AI Production Orchestrator
          |         |          |
          |         |          +--> Context Compiler
          |         +-------------> Inference Route Resolver
          +-----------------------> Workflow Executor
                                        |
                                        v
                            Engine Automation Projection
                                        |
                                        v
                       Policy / Validation / Project Gateway
```

### 7.1 Client Surface Adapter

Editor、CLI、Native SDK、MCPまたは外部Host固有inputを、同じWorkflow request、Run Context request、Query、Proposalへ変換する。built-in AI ConsoleはこのAdapterを使うoptional first-party Clientであり、専用writerではない。surface固有hidden default、権限、Project writerまたは別Operationを持たない。

### 7.2 AI Production Orchestrator

RunとAttemptを作成し、Workflow、Context、route、budget、child handle、resultを関連付ける。Project current pointer、Approval、Signer、CredentialまたはRelease authorityを持たない。

### 7.3 Context Compiler

Task、Project revision、Owner ref、Target、selection、contract／policy／tool catalog、sensitivity、redaction、budget、expiryからimmutable Run Contextを作る。conversation全文、Editor widget、native pointer、MCP sessionまたはModel memoryを暗黙入力にしない。

### 7.4 Inference Route Resolver

要求Capability、privacy、region、retention、cost、latency、Model Eval、local resource request、Provider／Deployment／Model bindingから一つのrouteを選ぶ。Model family名をEngine branchにせず、fallbackは別の完全なroute selection、新Context、新Authorization、新Runを要求する。

### 7.5 Workflow Executor

bounded typed stepを実行し、各stepのinput、output、Diagnostic、attempt、child handleをRun lineageへ記録する。任意shell、任意path、任意URL、unregistered OperationまたはPrompt内commandをstepへ変換しない。

### 7.6 Engine Automation Projection

Executable Contractsのcurrent operational Operationだけをsurfaceへ投影する。MCP、Provider、AgentまたはWorkflow名からOperation IDを生成しない。Project write、Commit、Promotion、Activation、Signing、Releaseは各Ownerのauthorityを維持する。

Automation Projection自体はModel、Prompt、conversation、Agent loopまたはInference Route Resolverを要求しない。AI Production Orchestrator、Agent Host、Provider Adapter、local inferenceまたはnetworkがdisabled、absent、crashedもしくはunavailableでも、target design上のEditor起動、Project open、manual authoring、Build、Test、Package、Game launchを阻害しない。これらの製品journeyが実際に成立したというclaimは、将来のmaterializationとQualificationが所有する。

## 8. RouteとAgent loop所有

`AiProductionExecutionRouteKindV1` target candidateを次のclosed 4 branchとする。

| route | control-loop owner | Model利用 | Engine surface | 最大意味 |
|---|---|---|---|---|
| `deterministic_automation` | Miraikanai Workflow Executor | なし | Native Automation projection | 登録済みWorkflowとAuthorization内のoperation |
| `first_party_agent` | Miraikanai first-party Agent Host | localまたはcloud Adapter | Native Automation projection | Query／Proposal。trusted internal operationは別AuthorizationでOrchestratorが依頼 |
| `standard_external_agent` | 外部Host | 外部Hostが所有 | MCP／SDK proposal projection | Query／Plan／Proposal／Validate／Preview |
| `managed_external_host` | conformance済み外部Host | attested deployment | Broker／MCP staging route | 将来Activation後のbounded Source edit／Build output受理 |

各Runはexactly one routeとexactly one control-loop ownerを持つ。Modelを使う三routeはexactly one Agent loop ownerを持ち、`deterministic_automation`はAgent loopを持たない。`standard_external_agent`または`managed_external_host`からのTool callに応じてfirst-party Agent Hostを起動しない。first-party Agentが外部Agentを再帰起動して同じTaskを委譲しない。明示child Runを許す場合も、親と異なるTask／purpose、最大depth、budget、allowed route、join policyをWorkflow Definitionへ固定し、depth超過またはloop-owner重複を開始前に拒否する。

Model Providerのtool-use responseはAgent loop ownershipではない。first-party routeではMiraikanaiがTool resultを検証して次turnを送る。external routeでは外部Hostがloopを継続し、Miraikanaiは各requestを独立に認証・検証する。

## 9. Workflow、Skill、Plugin、MCP

### 9.1 `AiWorkflowDefinitionV1` target candidate

Workflow Definitionは少なくとも次を持つ。

- stable workflow ID、revision、Owner、purpose、applicable route
- typed input／output Ref集合
- required Task kind、required Capability、allowed Operation exact Ref集合
- step graphとclosed step kind
- maximum step、model call、external roundtrip、Operation call、child Run、retry、wall time、cost、route-bound model usage unit、context／output byte budget
- question、stop、cancel、failure、compensation policy
- required Validation、Evidence、Approval class ref
- deterministic semantic hashとversioned compatibility policy

Workflow Definitionはreceipt-free immutable definitionであり、`active`、`approved`または同等のFieldを持ってOperation、Authorization、QualificationもしくはProduct Capabilityを有効化しない。Workflowが列挙するrequired Validation／Evidence／Approvalは下限であり、current Policyが要求する追加条件を削減できない。Modelは実行中にDefinition、allowed Operation、boundまたはstop policyを変更できない。

closed step kindは`query | compile_context | infer | request_input | propose | validate | invoke_authorized_operation | wait_operation | start_child_run | join_child_run | branch_typed_result | emit_result | stop`とする。Loopは明示back-edgeとpositive finite counterを持つ場合だけ許し、再帰、unbounded retry、自然言語だけのterminationを拒否する。

### 9.2 Skill

SkillはWorkflowを選ぶ、入力を集める、Tool順序を案内するguidanceである。Skill本文、description、example、scriptまたは成功宣言からOperation、Authorization、Approval、ResultまたはWorkflow stateを生成しない。SkillがWorkflowを参照する場合はexact Workflow bindingを使い、allowed Operation、budget、stop conditionを拡張できない。

### 9.3 Plugin

PluginはSkill、MCP connection、optional UI等の配布単位である。Engine Pack、Native Game ModuleまたはRuntime package pluginと同一概念にしない。Pluginのinstall、enableまたはvendor署名だけでEngine CapabilityをActivationしない。

### 9.4 MCP

MCPはexternal Agent向けAdapterである。MCP Tool、Prompt、Resource、Task extension、App UIはEngine Workflow、Project stateまたはAuthorityの正本にならない。MCP task handleを使用する場合も、canonical Run／Operation Task refへのtransport projectionに限定し、MCP Task自身を永続authorityにしない。

### 9.5 Codex／Claude固有surface

Codex App Server、Codex SDK、Claude Agent SDK／Claude Code plugin等はClient／Agent Host Adapter候補である。vendor固有thread、session、subagent、permission mode、hookまたはplugin lifecycleをMiraikanai正本型へ流入させない。各surfaceは同じsemantic conformance suiteへ適合した場合だけ`supported`と表示する。

## 10. Run、Attempt、Context、Result

新Ownerは次のtarget candidate familyを一意所有する。

- `AiProductionRunV1／RefV1`
- `AiProductionAttemptV1／RefV1`
- `AiProductionAttemptOutcomeV1／RefV1`
- `AiRunContextBundleV1／RefV1`
- `AiWorkflowDefinitionV1／RefV1`
- `AiWorkflowRegistryV1／RefV1`
- `AiWorkflowBindingV1／RefV1`
- `AiProductionExecutionRouteSelectionV1／RefV1`
- `AiOrchestrationCheckpointV1／RefV1`
- `AiProductionResultV1／RefV1`
- `AiProductionRunTransitionV1／RefV1`と`AiProductionRunHeadV1`

これらは本Design Specのtarget candidateであり、current MCD、Registry、Operation、Provider／MCP ToolまたはRuntime package memberではない。

### 10.1 Run identity

Runはexact Task Specification、Authorization subject、Workflow revision、Run Context hash、route selection、Project subject、Target集合、interaction intent、execution profile、budgetへ固定する。childとfallbackは別Fieldで一方向にlineageを持ち、同時non-nullにしない。Run IDをconversation ID、MCP session ID、Editor tab、process IDまたはdisplay nameから導出しない。

### 10.2 Attempt

retryとresumeは同じRunのAttemptを上書きせず、新しいmonotonic Attemptとして記録する。Attemptは入力Context、route、Model／Host profile binding、start／end、result branch、Diagnostic、child handleを持つ。別routeへのfallbackまたはModel／Host再選択は旧Runをterminalへ閉じ、新しいCaller Context、Authorization、Context、Run、initial Attemptを作る。同じRunのrouteを変更しない。

### 10.3 Run Context

Run Contextは少なくとも次を閉じる。

- Project／bootstrap subjectとexact revision
- Task Specification／Authorization refとhash
- required Owner、Document、selection、read set、typed field mask
- Contract set、Policy set、Tool catalog、Workflow revision
- Target、locale、Capability、generation、environment binding
- input provenance、Sensitivity、redaction、Provider data policy
- Workflowのcontext／output byte、wall-time、model／Operation call、monetary budgetと、routeのexact metering profile／model usage budget、PerformanceのCPU／GPU resource decisionへのbinding
- created time、expiry、issuer、semantic content hash

Contextはimmutable derived inputであり、Project authorityではない。required inputがboundを超える場合は、意味を変えるtruncateまたはAI要約を行わず、scope分割またはQuestionへ戻す。Project revision、Contract、Policy、Tool catalog、Provider policyまたはrequired input hashが変わればstaleとし、暗黙再compileせず新Contextを作る。

### 10.4 Run state

`AiProductionRunStateV1` target candidateは次のclosed enumとする。

```text
prepared | active | waiting_input | waiting_operation |
suspended | completed | failed | cancelled | superseded
```

- `prepared`: Context、Workflow、route、Authorization、budgetを開始前検証済み。
- `active`: 一つのcurrent Attemptがstepを進行中。
- `waiting_input`: Workflowが列挙したtyped User inputだけを待つ。
- `waiting_operation`: exact child Operation Task handleのterminal resultだけを待つ。
- `suspended`: durable Checkpointから同じinput closureでだけ再開可能。
- `completed`: typed Resultを発行した。Commit、Promotion、ApprovalまたはRelease完了を意味しない。
- `failed`: typed Diagnosticを持つterminal failure。
- `cancelled`: cancel closure後のterminal state。既Commit状態をrollbackした意味ではない。
- `superseded`: Project／Contract／Policy等のdriftにより再利用不能なterminal state。

AI SecurityのTask state、Executable ContractsのOperation Task state、Project StateのChangeSet stateをこのenumへ統合またはaliasしない。AI RunはProposal生成で`completed`になり、Approval待ちのまま生存させない。Approval後のCommitは別のtrusted Operationである。

### 10.5 Checkpointとresume

CheckpointはWorkflow step、Attempt、Context hash、completed child handle、idempotency key、remaining budget、secret-free intermediate refを保存する。Model hidden state、raw Credential、native pointer、uncommitted Project writerを保存しない。Checkpointはsuspend前の公開済みRun Transitionを参照し、suspend TransitionがCheckpointを参照する一方向にしてcontent-hash cycleを作らない。同じProject revision、Contract、Policy、Workflow、route binding、input closureがcurrentな場合だけresumeし、差があれば旧Runを`superseded`にして新Runを作る。

## 11. Interaction intentとbounded autonomy

Editorの`Ask | Suggest | Execute Authorized`はUser interaction intentであり、authority classではない。

- `Ask`: read／query／explain Workflowだけ。Proposalを作らない。
- `Suggest`: Proposal／Validationまで。Commit、PromotionまたはActivationを行わない。
- `Execute Authorized`: current Authorizationが許可するregistered operationだけを依頼できる。

conversationで提案された`Autonomous Workspace`と`Release Candidate`は追加authority modeにしない。次のexecution profileとして表現する。

- `bounded_workspace`: isolated workspace、finite Workflow、budget、checkpoint、allowed operation、human review pointを固定する。main／current Projectを直接変更しない。
- `release_candidate_assist`: exact Candidateに対するBuild、Test、Package、Evidence収集の提案またはtrusted internal requestを行う。Signing、Publication、Release Decisionを所有しない。

`Publish`、Store upload、Signing、Release、Product completionは常にProduct Release／Publication Ownerの別authorityである。AI interaction mode、workspace profile、Task成功またはModel confidenceから自動昇格しない。

## 12. Authoring AIとshipping Runtime AI

新OwnerはAuthoring Host／Toolingだけを対象にする。Runtime Packageは、AI Production Orchestrator、Agent Host、Workflow Registry、Run／conversation Store、Model Provider Credential、MCP Server、Source Worker、Compiler、Signer、write-capable Project Gateway、Prompt template、Authoring Tool SDKを含めない。

deterministic NPC／Gameplay AIはGameplay、Navigation、Simulationの既存Ownerに属する。shipping generative AIを将来追加する場合は、Product Future Portfolioから独立Capabilityとして昇格し、Runtime Owner、Threat Model、Privacy／network／cost／content safety、Target package、Save／Replay、offline failure、Service withdrawal、positive／negative Fixtureを一つのArchitecture変更として定義する。Authoring Provider Profile、Model Eval、Consent、Credential、Run ReceiptまたはActivationを流用しない。

## 13. Local／cloud Model route

first-party Agent route内で、cloud APIとfirst-party local inferenceを`InferenceDeploymentProfileV1`等のgoverned bindingにより選ぶ。Model family名、file extension、endpoint到達性またはGPU存在だけからrouteを選ばない。

新OwnerはAI固有のrequested budgetとroute decisionを所有する。model usageはexact metering profileへ固定し、first-party Engine-enforced、managed Host attested、standard external unattestedを別branchにして、異なるtokenizer／Provider unitを比較・換算しない。system CPU／GPU／RAM／VRAM capacity、measurement、reservation、loan、backpressureはPerformance／Capacityが所有し、deployment process、sandbox、network、license、model provenanceはAI Security supplement、artifact pinはToolchainが所有する。

local resource不足、runtime crashまたはtimeoutを理由にcloudへ暗黙送信しない。cloud fallbackはProvider、region、retention、training use、data class、cost、Model snapshot、Tool projectionを再Previewし、新しいroute selection、Context、Caller Context、Authorization、Run、initial Attemptを作る。fallback不成立時は元Projectを不変にして失敗する。

外部Agentがlocal modelを使用できることと、Miraikanaiがfirst-party local inference runtimeを配布・supportすることを別Capabilityにする。一方のConformanceまたは成功を他方へ流用しない。

## 14. Canonical data flow

```text
User intent / registered trigger
  -> Game production record or typed Task Specification
  -> AI Security Authorization
  -> Workflow selection and exact binding
  -> immutable Run Context compilation
  -> one execution route and one loop owner
  -> bounded query / inference / proposal steps
  -> Engine-owned schema and semantic validation
  -> Proposal / typed Result / Evidence refs
  -> Run completed
  -> separate Human Approval when required
  -> separate trusted Commit / Promotion / Build / Package operation
  -> Project revision or immutable artifact publication
```

conversation response、Model memory、Skill output、MCP Task、Agent sessionまたはWorkflow completionからProject revisionを直接進めない。人間のmanual編集もAI編集も同じProject ChangeSet、Validator、Approval、Commitを使う。

## 15. Failure、cancel、retry、recovery

### 15.1 開始前拒否

次のいずれかでRunを開始しない。

- missing／expired／revoked Authorization
- unresolved blocking Question
- unknown Workflow revisionまたはunregistered Operation
- stale Project／Contract／Policy／Tool catalog／Context
- route profile、Conformance、Model snapshot、licenseまたはProvider data policy不成立
- loop owner重複、child depth超過、budget不成立
- required input oversize、Sensitivity／redaction failure

### 15.2 実行中失敗

Provider refusal、malformed structured output、Schema drift、Tool timeout、worker crash、resource exhaustion、network failure、user cancelをtyped Diagnosticへ閉じる。自然言語による成功補完、別Toolへの名称fallback、部分Proposalの自動Commit、別Projectへの再対象化を行わない。

### 15.3 retry

retry policyはWorkflow Definitionに列挙し、同じidempotency class、finite count、retryable Diagnosticだけを許す。mutation operationのtimeout後はOwner Receipt／publication stateをread-backし、結果不明のまま再dispatchしない。Model再試行は新Attemptとし、同じ出力の再現性を主張しない。

### 15.4 cancel

cancelはcurrent Attemptとcancel可能child Taskへ伝播し、bounded deadline後にterminal `cancelled`またはtyped failureへ収束する。既に公開されたProject revision、Source promotion、PackageまたはReceiptを暗黙rollbackしない。rollbackが必要ならOwnerの登録済み別Operationを新Authorizationで実行する。

### 15.5 crash recovery

durable CheckpointとOwner-issued handleだけをread-backする。会話の再送、Modelの「続き」、MCP connection再確立または同じprocess IDをresume evidenceにしない。exact input closureが変わったRunは`superseded`にする。

## 16. Security、Evidence、Projectとの接続

### 16.1 Security

新OwnerはAuthorizationを発行、署名、拡張またはRisk再分類しない。Workflow、Skill、Agent、Model、route、User modeはAI Securityのallowed scopeを拡張できない。Provider CredentialをOrchestrator、Editor、Context、Prompt、LogまたはReceipt本文へ渡さない。

### 16.2 Evidence

RunはEvidenceを生成するOwnerではなく、Owner-issued Evidence refをlineageへ関連付ける。Model output、self-critique、conversation summary、visual impressionまたはWorkflow completionをValidation、Test、Gameplay Approval、QualificationまたはRelease Evidenceへ読み替えない。

### 16.3 Project

Run ContextとCheckpointはProject Sourceではない。Projectへ残すべきGame Decision、Brief、Spec、ChangeSet、Source、Test、Playtest、Receiptは各Ownerの正規recordへ明示promotionする。conversation historyをProject truthにせず、削除してもProjectを再構成できることを要求する。

### 16.4 Observability

Run、Attempt、step、child operationのcausal identityをDebugging／Observabilityへ公開できるが、Prompt、Source、Asset、Tool output本文を既定Telemetryへ保存しない。OpenTelemetryはexport Adapterであり、MiraikanaiのRun／Task／Evidence型を置換しない。

## 17. Surface parityとConformance

Editor、Native SDK、CLI、MCP、first-party Agent、standard external Agentについて、同じsubject、Workflow、Context、Operation、Authorization、Project revisionを与えた場合に次を要求する。

1. visible Operation exact集合がsurface policyとset equalityになる。
2. same semantic requestはsame outer Operation IDとtyped payload meaningへ解決する。
3. unsupported actionは全surfaceで同じstable Diagnostic classになる。
4. Provider／MCP／Plugin固有hidden defaultでTarget、scope、Risk、budgetまたはCommit behaviorを変えない。
5. external proposal-only surfaceにCommit、Promotion、Approval、Activation、Signing、Releaseを公開しない。
6. first-party UIだけが利用できるprivate writerを作らない。

Model文章またはTool call bytesの同一性は要求しない。要求するのはcanonical semantic request、validation、authority、result bindingの同値性である。

## 18. conversation提案の採否

### 18.1 採用する

- AI chatをEditor-onlyのAI Production Console／Partnerとして扱う。
- Engine Runtime、Engine Automation、Agent Runtime、Model Provider、Project Native Codeを分離する。
- MCPをAdapter、Engine Automation Contractを正本とする。
- 内蔵AIと外部Agentを同じcanonical Operationへ収束させる。
- Local／cloud ModelをProvider-neutralに扱い、local runtimeをEditor外へ隔離する。
- Chat historyをProject truthにせず、Decision／Spec／Task／ChangeSet／Test／Receiptへpromotionする。
- WorkflowとSkillを分離し、Skillをsecurity boundaryにしない。
- Project C++を正式surfaceとし、Engine Source変更と区別する。
- Authoring AIとshipping Runtime AIを別Trust Domainにする。

### 18.2 修正して採用する

- `AI Production Console`: UI名はEditor Ownerが決定し、新OwnerはUIでなくRun semanticsを所有する。
- `mira-automationd／mira-agentd／mira-modeld`: logical role分離だけを採用し、binary名とprocess mappingは固定しない。
- `Autonomous Workspace／RC`: authority modeでなくbounded execution profileにする。
- Model Gateway: provider-neutral route selectionだけを新Ownerが所有し、Credential／security profileはAI Security、artifact pinはToolchainに残す。
- Workflow Registry: target Workflow Definitionとbindingを新Ownerが所有するが、materialized Registryを存在すると扱わない。

### 18.3 採用しない

- MCP `2025-11-25`互換、legacy initialize、dual lifecycleまたはalias。
- MCP、Codex App Server、Claude SDK、Provider SDKをEngine内部正本APIにすること。
- generic gameplay Script VM／JITをinitial V1必須にすること。
- Project Runtime C++またはAI生成C++をEditor processへloadすること。
- in-process hot reloadをPreview成立条件にすること。
- conversationで例示されたDirectory構造を現在のNaming／Project Layoutへ上書きすること。
- Model、Skill、Plugin、Prompt、Agent名からCapability supportまたはsecurityを推測すること。
- 「最高レベル」をFeature数、Model名、Chat UIまたはArchitecture文書の存在だけで主張すること。

## 19. 正本反映対象

Design Spec承認後、次の最小coherent closureを一つのArchitecture変更として行う。

### 19.1 新規文書

1. `docs/architecture/03-authoring/ai-production-orchestration.md`: 新Owner。
2. `docs/architecture/decisions/2026-08-04-ai-production-orchestration-ownership.md`: 新Owner追加、既存分散維持、全面再編不採用のDecision。

### 19.2 Owner更新

1. `docs/architecture/00-product/product-plan.md`: AI-native claimへRun／Workflow／surface parity／Authoring-shipping分離を追加し、詳細は新Ownerへ解決する。
2. `docs/architecture/01-governance/ai-security-approval.md`: AI Orchestrator actorを新OwnerのRun semanticsへlinkし、Securityはauthorityだけを保持する。
3. `docs/architecture/01-governance/ai-verification-provenance.md`: Run／Attempt／WorkflowのEvidence consumptionと非代替をlinkする。
4. `docs/architecture/02-foundation/core-architecture.md`: logical roleとphysical Host／Process ownershipを分離し、exact binary名を固定しない。
5. `docs/architecture/02-foundation/toolchain-dependencies.md`: external Agent／local inference artifact pinだけを保持し、Workflow意味を新Ownerへ解決する。
6. `docs/architecture/02-foundation/executable-contracts.md`: Workflowから参照可能なのはcurrent registered Operationだけであること、MCP／Provider projectionがAdapterであることをlinkする。
7. `docs/architecture/03-authoring/project-state.md`: Run Context／CheckpointはProject Sourceでなく、Proposal／Decisionのpromotionだけが正本を変えることをlinkする。
8. `docs/architecture/03-authoring/game-production-loop.md`: 新Ownerを規範依存へ追加し、Game production WorkflowがRun semanticsを複製しないようにする。
9. `docs/architecture/03-authoring/editor-workspace-ux.md`: 新Ownerを規範依存へ追加し、AI PartnerをRun／Workflow／Context／Resultのread projectionへ変更する。
10. `docs/architecture/04-runtime/performance-capacity.md`: AI resource requestとsystem capacity ownershipを分離する関連linkを追加する。
11. `docs/architecture/04-runtime/debugging-observability-replay.md`: Run／Attempt causal projectionと本文redaction、OTel Adapter境界を追加する。
12. `docs/architecture/04-runtime/runtime-package.md`: Authoring Orchestrator、Agent、Credential、MCP、Compiler、write Gateway exclusionを明示する。
13. `docs/architecture/07-platform/windows.md`: 旧single Orchestrator executable前提を除き、logical roleと隔離Tooling Host process groupのPlatform mappingを整合させる。

### 19.3 補助文書、Index、監査記録

1. `docs/architecture/appendices/ai-provider-mcp-security-supplement.md`: security profileを保持し、Workflow、route、Run意味のshadow definitionを新Ownerへlinkする。
2. `docs/architecture/appendices/ai-evidence-envelope-fixture-catalog.md`: security caller route fieldとGeneration Receipt Signer境界を新Owner／Securityへ揃える。
3. `docs/architecture/appendices/architecture-plan-closure-review.md`: 今回gapとclosureを新しい`ARCH-C*`として登録する。
4. `docs/architecture/decisions/README.md`: Ownership DecisionをDecision Logへ追加する。
5. `docs/architecture/README.md`: Owner 65件へ更新し、新OwnerをProject StateとGame Production Loopの間へ追加する。
6. `docs/reviews/README.md`: conversation audit ID、route、input、valid gap、affected Owner、closure、retention dispositionを記録する。

先行Design Spec、normative／rejected ADR body、既存review summaryの履歴を上書きしない。今回のDesignは2026-08-03 Designをsupersedeせず、AI production execution gapを補う後続設計とする。

## 20. Current empty stateとclean break

現Repositoryでは次をexact empty／absentとして扱う。

- `AiProductionRunV1`等のtarget candidate MCD type集合
- AI Workflow Registry／binding
- Orchestrator／Agent Host／Context Compiler／Inference Route Resolver executable
- first-party local inference runtime／loader
- AI Production target Operation familyのmaterialized／active／operational集合
- Provider／MCP／Editor／CLI／SDK projection
- Qualification Fixture／Receipt／Activation

Owner本文にtype、state、routeまたはOperation候補を記載してもcurrent membershipにしない。initial V1でmaterialized predecessorがないため、旧candidate名、conversation例、temporary alias、legacy session、dual readerまたはmigration Operationを受理しない。初回公開後の変更はCompatibility／Evolutionのconsumer Inventoryとversioned migrationを必要とする。

## 21. Target-design検証

正本反映後は少なくとも次を検証する。

1. Owner ID、正本範囲、candidate type ownershipが一意。
2. Architecture Indexが65 Ownerを過不足なく列挙する。
3. 規範依存targetが全て存在し、cycleが0。
4. Product → Governance → Foundation → Authoring／Runtime → Domain → Pack方向を逆転しない。
5. Game Production LoopとEditorだけが新Ownerの直接Authoring consumerとなり、上位Ownerは関連linkに留まる。
6. AI SecurityがAuthorization、Risk、Approval、Credential、Provider／MCP securityを保持する。
7. Executable ContractsがMCD、Operation、Diagnostic、projectionを保持する。
8. Project StateがProject revision、ChangeSet、Commit、Undoを保持する。
9. Skill、Plugin、MCP、Codex／Claude surfaceがWorkflowまたはauthorityの正本にならない。
10. routeごとにcontrol-loop ownerがexactly oneで、Model利用routeではAgent loop ownerもexactly oneになり、nested loopとimplicit fallbackを拒否する。
11. `completed` RunをCommit、Approval、Qualification、ReleaseまたはProduct completionへ読み替えない。
12. Authoring AI artifact／Credential／ServiceがRuntime packageから除外される。
13. local inference、external Host local Model、shipping Runtime AIのCapabilityを相互代用しない。
14. current MCD、Registry、Fixture、Receipt、Operation、Service、Build、CIを存在すると表現しない。
15. 全relative Markdown path／fragmentとADR reciprocal metadataが解決する。
16. external fact、project-decision、provisional、measuredを混同しない。
17. `git diff --check`、`git status --short`、`git diff --stat`と全changed file diff inspectionを行う。

RepositoryにBuild、test runner、Markdown linter、link checker、Inventory generatorまたはCI workflowは存在しないため、それらの実行を主張しない。link、anchor、ownership、DAG、state wordingはmanualまたはread-only scriptで検証する。

## 22. Design完了条件

本Designは次がすべて成立した場合だけ正本反映へ進める。

1. Userが新Owner方式、一意所有、route／loop、Workflow／Skill／Plugin／MCP境界、Run state、Authoring／shipping境界、反映対象を承認する。
2. 未確定項目、仮置き記述、複数解釈可能なauthorityまたはownerがない。
3. 新Ownerのdirect dependency 8件とdirect consumer 2件がcycle 0である。
4. 実装、実装計画、工程、工数、担当またはmaterialized artifactを含まない。
5. conversation提案の採用、修正採用、不採用が明示される。
6. 外部一次資料の事実とMiraikanai project-decisionが分離される。

正本反映後もArchitecture全体は`review`、実装状態は`absent`である。最高レベルのEngine、AI-native support、third-party usability、production readinessまたはProduct completionは、将来の実装、materialization、Qualification、Release Evidenceが同じCandidateへ閉じるまで未達である。
