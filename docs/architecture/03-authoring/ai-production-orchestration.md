# Miraikanai Engine AI Production Orchestration Contract

- 文書ID: mirakan.arch.ai-production-orchestration
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: AI Production Run／Attempt／Result／Checkpoint、AI Workflow Definition／Registry／binding／step semantics／bound、immutable Run Context assembly、deterministic Automation／first-party Agent／standard external Agent／managed external Hostのproduction route、Runごとのcontrol-loop／Agent-loop単一所有、明示fallback／child Run、conversation／session非authority、interaction intent／bounded execution profile、Authoring AIとshipping Runtime AIの非代替、AI surface semantic parity、versioned Agent Host Conformance Profile、およびこれらを扱うtarget Operation familyの意味
- 非正本範囲: Product claim／MVP／First Playable／Release／Completion、Product journeyのrequired surface／client集合とrelease acceptance、Task Authorization／Risk／Approval／Consent／Credential／Provider・MCP security、Evidence／Eval／Provenance／freshness、MCD／Operation共通Envelope／Diagnostic／projection、Host／Process／IPC配置、Project revision／Document／ChangeSet／Commit／Undo、Game Brief／Spec／Question／Playtest／Iteration、Panel／Workspace／Accessibility、外部SDK／runtime／model artifactのversion・hash・license、system CPU／GPU／memory capacity、Runtime package／shipping eligibility、NPC／Gameplay AI／Navigation、shipping generative Runtime AI、binary名／service数／queue・thread実装、実装順序／工程／工数／担当
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core Architecture](../02-foundation/core-architecture.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Project State](project-state.md)
- 関連文書: [AI Production Orchestration Ownership Decision](../decisions/2026-08-04-ai-production-orchestration-ownership.md)、[Product Lifecycle](../00-product/product-lifecycle.md)、[Game Production Loop](game-production-loop.md)、[Editor Workspace／UX](editor-workspace-ux.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[Debugging／Observability／Replay](../04-runtime/debugging-observability-replay.md)、[Runtime Package](../04-runtime/runtime-package.md)、[AI Provider／MCP Security Supplement](../appendices/ai-provider-mcp-security-supplement.md)、[AI-native C++ Product Identity Decision](../decisions/2026-08-03-ai-native-cpp-product-identity.md)、[MCP Current Protocol Baseline Decision](../decisions/2026-08-03-mcp-current-protocol-baseline.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未materializeの型・Registry・Operation・Fixture・Receiptおよび未計測boundはtarget candidate／provisional）
- 外部根拠確認日: 2026-08-04

## 1. 結論とcurrent状態

MiraikanaiのAI制作は、ModelまたはChat UIへProject write権限を与える機能ではない。User intentまたは登録済みtriggerを、署名済みTask Authorization、immutable Run Context、exact Workflow、単一execution route、bounded step、Engine-owned typed Operationへ接続し、Proposalまたはtyped Resultまでを追跡するAuthoring control planeである。Projectを変更するのは引き続き[Project State](project-state.md)のValidation、Approval、atomic Commit／Promotionを通ったtrusted Operationだけである。

本書は、これまでAI Security、Provider supplement、Editor、Game Production Loopへ分散していたRun、Workflow、Context、route、Agent loopの意味を一意所有する。Securityは権限、VerificationはEvidence、Executable ContractsはOperation、Coreは物理Host／Process、Project StateはProject authorityを保持する。Model、Agent Host、MCP、Skill、Plugin、Prompt、conversation、Editor PanelまたはWorkflow completionは、これらのauthorityを新設または代替しない。

現Repositoryに本書のMCD Schema、Workflow Registry、Run Store、Operation、Service、Provider／MCP projection、Fixture、ReceiptまたはC++実装は存在しない。以下の型、state、route、algorithmはtarget Architecture candidateであり、`materialized`、`active`、`operational`、`qualified`、`supported`、`released`またはProduct completionを意味しない。

## 2. Authorityと一意所有

| concern | canonical Owner | 本書の使用 |
|---|---|---|
| AI-native Product claim、Capability、manual continuity、MVP／Future | [Product Plan](../00-product/product-plan.md) | orchestration requirementを受ける。Product stateを決めない |
| Task Specification、Authorization、Risk、Consent、Approval、Credential、Caller／Provider trust | [AI Security／Approval](../01-governance/ai-security-approval.md) | exact signed refとcurrent security decisionを消費し、scopeを拡張しない |
| Evidence、Eval、Provenance、freshness、Trace grade、Receipt | [AI Verification／Provenance](../01-governance/ai-verification-provenance.md) | Owner-issued refをlineageへ結び、pass／freshnessを再定義しない |
| MCD kind、Operation、Task／Receipt共通Envelope、Diagnostic、projection | [Executable Contracts](../02-foundation/executable-contracts.md) | current operational OperationだけをWorkflowへ束縛する |
| Host、Process、IPC、Gateway、Worker、failure isolation | [Core Architecture](../02-foundation/core-architecture.md) | logical role要件だけを示し、binary名またはprocess数を決めない |
| external Tool／SDK／runtime／Model artifact pin | [Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md) | exact selected bindingを消費し、version／licenseを定義しない |
| Project／revision、Document、ChangeSet、Commit、Undo／Recovery | [Project State](project-state.md) | exact subject／revisionへProposal lineageを固定する |
| Intent、Brief、Spec、Question、Understanding、Playtest、Iteration | [Game Production Loop](game-production-loop.md) | domain Workflowのtyped input／outputとして参照する |
| Panel、Workspace、visual／accessible presentation | [Editor Workspace／UX](editor-workspace-ux.md)とEditor UI Framework | read projectionを供給し、Widget stateをauthorityにしない |
| CPU／GPU／RAM／VRAM capacity、reservation、measurement | [Performance／Capacity](../04-runtime/performance-capacity.md) | orchestration budgetとは別のcapacity判定として消費する |
| Run causal event、Store／Query／export、redaction | [Debugging／Observability／Replay](../04-runtime/debugging-observability-replay.md) | typed causal projectionだけを公開する |
| shipping package、launch、Runtime inclusion／exclusion | [Runtime Package](../04-runtime/runtime-package.md) | Authoring AI artifact exclusionを要求する |
| Product journey surface、required client profile集合、release acceptance | [Product Lifecycle](../00-product/product-lifecycle.md) | AI Profile identityを供給し、Product parity／release passを決めない |

本書の`AiProductionRunStateV1`はAI SecurityのGovernance Task state、Coreの`OperationTaskV1.state`、Project StateのChangeSet stateまたはProvider session stateと別の直交stateである。同じ表示語があってもalias、implicit transitionまたは完了推論を作らない。

## 3. 共通identity、canonicalization、lineage

本書が所有する全top-level target recordはclosed object、`schema_version=1`、型固有Stable ID、自己hashを除く全Fieldのcanonical bytesから導出する型固有`content_hash: SHA-256`を持つ。共通encoding、hash、`ArtifactRefV1`、`McdContractRefV1`のshapeはExecutable Contractsを参照し、本書へ複写しない。

各`XxxRefV1`は型固有ID、schema version、完成record content hashのexact tupleで、解決した完成recordとbyte equalityにする。裸ID、display name、path、array index、`latest`、conversation ID、MCP session ID、Editor tab、process ID、Model-generated aliasまたは同ID別hashを受理しない。配列は別記がなければcanonical ref bytes順のstrict sorted unique setで、duplicate、同logical ID別hash、欠損memberの自然言語補完を拒否する。

```text
AiProductionSurfaceKindV1 =
  editor_builtin | native_sdk | cli | mcp
  | first_party_agent_host | external_agent_host
```

Run lineageは次の一方向だけを許す。

```text
Workflow Definition / Registry
  -> Workflow Binding
  -> Route Selection
  -> Run Context Bundle
  -> Run
  -> Attempt start
  -> Step / child Operation / child Run result
  -> Attempt Outcome
  -> Run Transition
  -> typed Result or terminal Diagnostic
  -> optional downstream Evidence / Approval / Commit
```

後段Recordを前段のsemantic hashへ埋め戻さない。ResultをsubjectにするEvidenceはResultへ逆参照し、Result側へその後発Evidenceを追加しない。Run中に消費した既存EvidenceはContext inputとして参照できるが、後発Receiptとのhash cycleを作らない。

## 4. 論理roleと物理配置の分離

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

| logical role | responsibility | prohibited |
|---|---|---|
| Client Surface Adapter | surface固有requestを同じtyped Workflow／Query／Proposal requestへ変換 | private writer、hidden permission、surface固有Project format |
| AI Production Orchestrator | Run／Attempt／Transition／Checkpoint／Resultのlineage調整 | Project current pointer、Approval、Credential、Signer key／authority、Release authority |
| Context Compiler | authorized projectionからimmutable Run Contextを構築 | conversation全文、native pointer、Widget object、Model memoryの暗黙取込 |
| Inference Route Resolver | requirement、privacy、Profile、Eval、budgetからexact一routeを選択 | Model名branch、到達性だけのfallback、Authorization拡張 |
| Workflow Executor | bounded typed stepとchild handleを進行 | arbitrary shell／path／URL、未登録Operation、Prompt command実行 |
| Engine Automation Projection | current operational Operationをsurfaceへ投影 | Operation生成、Validation bypass、Providerをauthority化 |

これらはlogical roleであり、binary、process、thread、queue、Directoryまたはdeployment countではない。物理Host／Process／IPCはCore Architecture、artifact pinはToolchainが所有する。built-in AI Consoleはoptional first-party Clientであり、OrchestratorまたはProject Gatewayそのものではない。

Engine Automation ProjectionはModel、Agent、Prompt、conversationまたはnetworkを要求しない。Orchestrator、Agent Host、Provider Adapter、local inferenceまたはMCPがdisabled、absent、crashedもしくはunavailableでも、target design上のEditor起動、Project open、manual authoring、Build、Test、Package、Game launchをprivate AI dependencyでblockしない。実際のjourney成立は将来の実装とQualificationが所有する。

## 5. Workflow Definition、Registry、binding

### 5.1 Workflow Definition

```text
AiWorkflowBoundsV1
  max_step_transitions: positive uint32
  max_model_calls: uint32
  max_external_host_roundtrips: uint32
  max_operation_calls: uint32
  max_child_runs: uint32
  max_child_depth: uint8
  max_parallel_children: uint16
  max_retries: uint16
  max_context_bytes: positive uint64
  max_output_bytes: positive uint64
  max_wall_time_ms: positive uint64
  monetary_budget:
    {kind: no_engine_billed_model}
    | {kind: engine_enforced,
       currency_code: canonical ISO 4217 ASCII,
       maximum_microunits: positive uint64}
    | {kind: external_host_owned_unattested}

AiWorkflowStepV1
  step_id: StableId
  step_kind:
    query | compile_context | infer | request_input | propose |
    validate | invoke_authorized_operation | wait_operation |
    start_child_run | join_child_run | branch_typed_result |
    emit_result | stop
  input_contract_refs[]: McdContractRefV1(kind=type)
  output_contract_refs[]: McdContractRefV1(kind=type)
  operation_ref: McdContractRefV1(kind=operation) | null
  instruction_artifact_ref: ArtifactRefV1 | null
  child_workflow_ref: AiWorkflowDefinitionRefV1 | null
  child_task_specification_contract_ref: McdContractRefV1(kind=type) | null
  child_bounds: AiWorkflowBoundsV1 | null
  child_allowed_route_kinds[]: AiProductionExecutionRouteKindV1
  child_join_policy_ref: McdContractRefV1(kind=policy) | null
  idempotency_class_ref: McdContractRefV1(kind=policy) | null
  step_policy_refs[]: McdContractRefV1(kind=policy)

AiWorkflowEdgeV1
  from_step_id: StableId
  to_step_id: StableId
  edge_kind: success | typed_branch | failure | bounded_back_edge
  branch_contract_ref: McdContractRefV1(kind=type) | null
  branch_value_hash: SHA-256 | null
  back_edge_counter_id: StableId | null
  maximum_traversals: positive uint32 | null

AiWorkflowDefinitionV1
  schema_version: 1
  workflow_id: StableId
  workflow_revision: positive uint32
  owner_document_id: ArchitectureDocumentId
  purpose_id: StableId
  applicable_route_kinds[1..4]: AiProductionExecutionRouteKindV1
  required_task_kind_refs[]: McdContractRefV1(kind=type)
  required_capability_refs[]: McdContractRefV1(kind=capability)
  input_contract_refs[]: McdContractRefV1(kind=type)
  output_contract_refs[]: McdContractRefV1(kind=type)
  allowed_operation_refs[]: McdContractRefV1(kind=operation)
  entry_step_id: StableId
  terminal_step_ids[1..]: StableId
  steps[1..]: AiWorkflowStepV1
  edges[]: AiWorkflowEdgeV1
  bounds: AiWorkflowBoundsV1
  question_policy_refs[]: McdContractRefV1(kind=policy)
  cancel_policy_refs[]: McdContractRefV1(kind=policy)
  failure_policy_refs[]: McdContractRefV1(kind=policy)
  compensation_operation_refs[]: McdContractRefV1(kind=operation)
  required_validation_contract_refs[]: McdContractRefV1(kind=type)
  required_evidence_class_refs[]: McdContractRefV1(kind=type)
  required_approval_class_refs[]: McdContractRefV1(kind=type)
  workflow_content_hash: SHA-256

AiWorkflowDefinitionRefV1
  workflow_id: StableId
  schema_version: 1
  workflow_revision: positive uint32
  workflow_content_hash: SHA-256
```

Workflow Definitionはreceipt-free immutable definitionであり、`active`、`approved`、`qualified`または同等Fieldを持たない。列挙したValidation／Evidence／Approvalは下限であり、current PolicyまたはOperation固有要件を削減しない。Model、Skill、Plugin、User modeまたは実行中stepはDefinition、allowed Operation、bound、stop／failure policyを変更できない。

通常edgeからbounded back edgeを除いたgraphはacyclicで、全stepがentryから到達し、全terminal pathが`emit_result | stop`へ有限到達しなければならない。`bounded_back_edge`だけが明示counterとpositive maximumを持てる。再帰的Prompt、自然言語だけのtermination、unbounded retry、counter reset、別Attemptによるloop budget resetを禁止する。

`infer`は`first_party_agent`だけで有効である。external routeでは外部HostがAgent loopを所有し、Miraikanaiは外部Hostから受けたtyped payloadを`query | propose | validate`の入口で検証するため、Workflow Executorが外部Provider turnを代行しない。`deterministic_automation`は`infer`を含めない。

`start_child_run`と`join_child_run`はpairであり、exact child Workflow、Task Specification contract、予約したsub-budget、allowed route、join policyをDefinitionに固定する。child開始時にはそのcontractに適合する別Task Specificationと別Authorizationを発行し、同じTask ID、同じpurpose、ancestor Workflow chain内の再帰、depth／count／parallel bound超過または親Credential／Context／Authorization継承を拒否する。各childは独立Runとexact一loop ownerを持ち、親Run completionを子のCommit／Approvalへ読み替えない。child以外のstepは全`child_*` Fieldをnull／emptyにし、`start_child_run`だけがWorkflow／Task contract／bounds／allowed routeをnon-null、対応`join_child_run`だけがjoin policyとexact child handle inputを持つ。

### 5.2 Workflow Registry

```text
AiWorkflowRegistryEntryV1
  workflow_ref: AiWorkflowDefinitionRefV1
  owner_document_id: ArchitectureDocumentId
  definition_artifact_ref: ArtifactRefV1
  definition_artifact_sha256: SHA-256

AiWorkflowRegistryV1
  schema_version: 1
  registry_id: StableId
  registry_revision: positive uint32
  entries[]: AiWorkflowRegistryEntryV1
  registry_content_hash: SHA-256

AiWorkflowRegistryRefV1
  registry_id: StableId
  schema_version: 1
  registry_revision: positive uint32
  registry_content_hash: SHA-256
```

Registryはimmutable definition inventoryであり、Operation、Authority、Capability、Qualificationまたはexecution availabilityを有効化しない。Entryが存在しても、全参照型／Operationがcurrent Contract setにmaterializeし、required Operationがoperational、Product CapabilityとTargetが利用可能、Task AuthorizationとCaller／Provider ConformanceがcurrentでなければWorkflowを開始しない。

### 5.3 Workflow Binding

```text
AiWorkflowBindingV1
  schema_version: 1
  binding_id: StableId
  workflow_registry_ref: AiWorkflowRegistryRefV1
  workflow_ref: AiWorkflowDefinitionRefV1
  task_authorization_ref: TaskAuthorizationEnvelopeRefV1
  task_authorization_wrapper_sha256: SHA-256
  foundation_definition_closure_ref: FoundationDefinitionClosureRefV1
  contract_set_hash: SHA-256
  policy_set_hash: SHA-256
  tool_catalog_hash: SHA-256
  surface_kind: AiProductionSurfaceKindV1
  surface_policy_hash: SHA-256
  resolved_operation_refs[]: McdContractRefV1(kind=operation)
  applicable_route_kind: AiProductionExecutionRouteKindV1
  resolved_validation_contract_refs[]: McdContractRefV1(kind=type)
  resolved_evidence_class_refs[]: McdContractRefV1(kind=type)
  resolved_approval_class_refs[]: McdContractRefV1(kind=type)
  binding_content_hash: SHA-256

AiWorkflowBindingRefV1
  binding_id: StableId
  schema_version: 1
  binding_content_hash: SHA-256
```

`surface_kind`と`surface_policy_hash`は開始surfaceで利用可能なsemantic projectionを固定する。`resolved_operation_refs[]`はDefinitionのallowed集合、Authorizationのallowed集合、current operational Operation集合、surface policy集合のintersectionであり、Workflowが実際に参照する全Operationとset equalityにする。別surfaceからstatus／cancel／resultを扱ってもoriginal Bindingを変更せず、そのsurface自身のcurrent policyが同じActionを公開する場合だけexact Run refへ接続する。empty intersectionを名称、prefix、Provider aliasまたは「近いTool」で補完しない。Bindingは実行時inputを固定するが、Operation ActivationまたはAuthorizationを発行しない。

## 6. Run Context

```text
AiRunContextInputBindingV1
  input_contract_ref: McdContractRefV1(kind=type)
  input_artifact_ref: ArtifactRefV1
  input_content_sha256: SHA-256
  owner_document_id: ArchitectureDocumentId
  subject_revision_binding_hash: SHA-256
  requested_field_path_set_hash: SHA-256
  returned_field_path_set_hash: SHA-256
  completeness_kind: complete | bounded_query
  omitted_range_set_hash: SHA-256 | null
  sensitivity_class_ref: McdContractRefV1(kind=policy)
  redaction_result_ref: ArtifactRefV1 | null
  input_binding_content_hash: SHA-256

AiRunContextBundleV1
  schema_version: 1
  context_id: StableId
  task_authorization_ref: TaskAuthorizationEnvelopeRefV1
  task_authorization_wrapper_sha256: SHA-256
  task_context_capsule_contract_ref: McdContractRefV1(kind=type)
  task_context_capsule_artifact_ref: ArtifactRefV1
  task_context_capsule_content_sha256: SHA-256
  workflow_binding_ref: AiWorkflowBindingRefV1
  route_selection_ref: AiProductionExecutionRouteSelectionRefV1
  context_input_set_hash: SHA-256
  input_bindings[1..]: AiRunContextInputBindingV1
  target_profile_refs[]: TargetProfileRefV1
  locale_profile_refs[]: McdContractRefV1(kind=profile)
  environment_profile_refs[]: McdContractRefV1(kind=profile)
  contract_set_hash: SHA-256
  policy_set_hash: SHA-256
  tool_catalog_hash: SHA-256
  context_byte_count: positive uint64
  created_at: UtcTimestamp
  expires_at: UtcTimestamp
  issuer_subject_ref
  context_content_hash: SHA-256

AiRunContextBundleRefV1
  context_id: StableId
  schema_version: 1
  context_content_hash: SHA-256
```

`task_context_capsule_*`はAI Security所有の完成`AiTaskContextCapsuleV1`へexact解決し、そのTask Authorization、subject revision、active Operation、projection binding、expiryをbyte equalityで消費する。本書はCapsule shape、Sensitivity、redaction、AuthorizationまたはProject revisionの意味を複製しない。

Context CompilerはWorkflow required inputとCapsule projectionをset equalityで照合し、`context_input_set_hash`を全input bindingのcanonical bytesから計算する。conversation全文、User-local message history、Model memory、MCP Resource listing、Editor selection object、screen capture、filesystem current stateまたはWeb resultを未登録inputとして追加しない。必要なら各Ownerのregistered bounded Projectionとして明示取得する。

required inputがbudgetを超える場合、意味を変えるtruncate、silent omissionまたはModel要約を行わない。Operationが`bounded_query`を許す場合だけomitted rangeとcontinuationを保持し、それ以外はscopeを分割するかtyped Questionへ戻す。Project revision、Contract、Policy、Tool Catalog、Workflow、route、Provider data policy、required input hashまたはexpiryが変わればContextをstaleとし、in-place rebaseせず新Contextと新Runを作る。

Run Contextはimmutable derived inputであり、Project Source、Decision Ledger、Authorization、Approval、Evidence、CredentialまたはProvider memoryではない。Contextを削除しても、Projectの正規Source、Decision、Commit、ReceiptからProject stateを再構成できなければならない。

## 7. execution routeとloop owner

`AiProductionExecutionRouteKindV1`は次のclosed setである。

| production route | control-loop owner | Agent loop | AI Security caller routeとの対応 | 最大surface意味 |
|---|---|---|---|---|
| `deterministic_automation` | Miraikanai Workflow Executor | なし | caller routeなし。Task Authorizationは必須 | registered query／authorized Operation |
| `first_party_agent` | Miraikanai first-party Agent Host | exactly one | `engine_provider_adapter` | Query／Proposal。trusted Operationは別dispatch |
| `standard_external_agent` | external Host | exactly one external | current target projectionは`standard_external_mcp` | Query／Plan／Proposal／Validate／Preview |
| `managed_external_host` | conformance済みexternal Host | exactly one external | `managed_external_host` | Future Activation後のbounded managed output staging |

production routeはRun control flowの意味、AI Security caller routeはHost／Transport／Provider／Credential／Conformanceのsecurity unionであり、別型である。`standard_external_agent`をMCP以外の外部SDKへ投影するには、AI Security OwnerがそのTransport／Caller Context branchとConformanceを別途定義・materializeしなければならない。SDK名または接続成功だけで`standard_external_mcp`を代用しない。

```text
AiModelUsageBudgetV1 =
  {kind: no_model}
  | {kind: engine_enforced,
     metering_profile_binding_ref: ArtifactRefV1,
     maximum_input_units: uint64,
     maximum_output_units: uint64,
     maximum_total_units: positive uint64}
  | {kind: managed_host_enforced_attested,
     metering_profile_binding_ref: ArtifactRefV1,
     maximum_input_units: uint64,
     maximum_output_units: uint64,
     maximum_total_units: positive uint64}
  | {kind: external_host_owned_unattested}

AiProductionExecutionRouteSelectionV1
  schema_version: 1
  route_selection_id: StableId
  task_authorization_ref: TaskAuthorizationEnvelopeRefV1
  task_authorization_wrapper_sha256: SHA-256
  workflow_binding_ref: AiWorkflowBindingRefV1
  context_input_set_hash: SHA-256
  route_kind: AiProductionExecutionRouteKindV1
  control_loop_owner_kind:
    workflow_executor | first_party_agent_host | external_agent_host
  agent_loop_owner_binding_ref: ArtifactRefV1 | null
  security_caller_context_ref: ArtifactRefV1 | null
  security_caller_context_sha256: SHA-256 | null
  inference_deployment_binding_ref: ArtifactRefV1 | null
  model_snapshot_binding_ref: ArtifactRefV1 | null
  model_usage_budget: AiModelUsageBudgetV1
  external_host_profile_binding_ref: ArtifactRefV1 | null
  operation_projection_set_hash: SHA-256
  capacity_decision_binding_ref: ArtifactRefV1 | null
  fallback_of_route_selection_ref:
    AiProductionExecutionRouteSelectionRefV1 | null
  explicit_user_confirmation_ref: ArtifactRefV1 | null
  selected_at: UtcTimestamp
  expires_at: UtcTimestamp
  route_selection_content_hash: SHA-256

AiProductionExecutionRouteSelectionRefV1
  route_selection_id: StableId
  schema_version: 1
  route_selection_content_hash: SHA-256
```

branch invariantは次である。

- `deterministic_automation`: control owner=`workflow_executor`、Agent／Caller Context／Inference／Model／external Hostは全null、usage budget=`no_model`。
- `first_party_agent`: control owner=`first_party_agent_host`、Agent loop binding、Security `engine_provider_adapter` Caller Context、Inference Deployment、Model Snapshotが全non-nullで同じProfile tupleへ解決し、external Hostはnull、usage budget=`engine_enforced`。
- `standard_external_agent`: control owner=`external_agent_host`、external HostとSecurity `standard_external_mcp` Caller Contextがnon-null、Miraikanai側Inference／Model bindingはnull／unattested、usage budget=`external_host_owned_unattested`である。
- `managed_external_host`: control owner=`external_agent_host`、external Host、Security `managed_external_host` Caller Context、managed deployment／Model bindingがcurrent、usage budget=`managed_host_enforced_attested`で、専用Future CapabilityがActivation済みの場合だけ選択できる。

一Runはexactly one routeとexactly one control-loop ownerを持つ。Modelを使うrouteはexactly one Agent-loop ownerを持ち、Providerのtool-use responseを別Agent loopに数えない。external AgentのTool callに応じてfirst-party Agent Hostを起動せず、first-party Agentが同じTaskをexternal Agentへ再帰委譲しない。child Runは別Task／Authorization／budget／routeを持つ独立Runであり、一Run内のloop-owner重複を正当化しない。

Route SelectionはauthorityまたはConformance Receiptではない。参照したProfile／Receipt／Caller Contextのcurrent head、expiry、revocation、Target、Tool projection、Model Eval、capacity decisionを開始前と各外部call前にread-backし、一件でもdriftすれば継続しない。

`metering_profile_binding_ref`はusage unitの名称、Model／tokenizerまたはProvider meter snapshot、計算法、rounding、report sourceをexactに固定する。同じ`token`表示、整数値またはModel familyでもprofileが異なるunitを比較、加算または換算しない。Editorはexact unit labelと`engine-enforced | managed-host-attested | unknown/external-host-owned`を表示し、unattested usageを0、無料、上限内またはMiraikanai計測済みとしない。Context／output byteとmodel-call boundはusage meterに関係なくWorkflowのhard boundとして残す。

monetary branchはdeploymentへ明示束縛する。`deterministic_automation`とfirst-party local inferenceは`no_engine_billed_model`、Engineが請求meterとhard capをcurrent検証できるfirst-party cloudは`engine_enforced`、`standard_external_agent`は`external_host_owned_unattested`とする。`managed_external_host`で`engine_enforced`を選べるのは、registered Brokerがexact ISO 4217 currency、microunit計算法、上限の事前強制と事後cost attestationを同じRoute Selectionへ束縛する場合だけで、それ以外は`external_host_owned_unattested`としてcost-bound Workflowを拒否する。`no_engine_billed_model`はCPU／GPU／電力／license costが0、無料またはPerformance capacity内であることを意味しない。

## 8. Run、Attempt、Transition、Result

### 8.1 Run identity

```text
AiProductionRunV1
  schema_version: 1
  run_id: UUIDv7
  parent_run_ref: AiProductionRunRefV1 | null
  fallback_of_run_ref: AiProductionRunRefV1 | null
  task_specification_ref: ArtifactRefV1
  task_specification_sha256: SHA-256
  task_authorization_ref: TaskAuthorizationEnvelopeRefV1
  task_authorization_wrapper_sha256: SHA-256
  workflow_binding_ref: AiWorkflowBindingRefV1
  route_selection_ref: AiProductionExecutionRouteSelectionRefV1
  context_ref: AiRunContextBundleRefV1
  project_or_bootstrap_subject_ref: ArtifactRefV1
  target_profile_refs[]: TargetProfileRefV1
  interaction_intent: AiProductionInteractionIntentV1
  interaction_intent_binding_ref: ArtifactRefV1
  execution_profile_kind: AiProductionExecutionProfileKindV1
  execution_profile_policy_ref: McdContractRefV1(kind=policy)
  run_budget: AiWorkflowBoundsV1
  initial_state: prepared
  prepared_at: UtcTimestamp
  run_content_hash: SHA-256

AiProductionRunRefV1
  run_id: UUIDv7
  schema_version: 1
  run_content_hash: SHA-256
```

RunはTask、Authorization、Workflow、route、Context、subject、Target、interaction intent、execution profile、budgetへ固定する。Run IDをconversation、MCP session、thread、Panel、process、Model responseまたはdisplay nameから導出しない。`parent_run_ref`はexplicit child relation、`fallback_of_run_ref`はexplicit route fallbackだけに使い、rootでは両方null、child／fallbackではexactly oneだけをnon-nullにする。childはsame Task／purpose、ancestor cycle、depth／budget違反を拒否する。fallbackは旧Runを変更せず、旧Runをterminal `superseded | failed`へ閉じてから新Task Authorization、Context、Route Selection、Runを作る。

### 8.2 Attempt startとoutcome

```text
AiProductionAttemptV1
  schema_version: 1
  attempt_id: UUIDv7
  run_ref: AiProductionRunRefV1
  attempt_ordinal: positive uint32
  attempt_kind: initial | retry | resume
  previous_attempt_outcome_ref: AiProductionAttemptOutcomeRefV1 | null
  route_selection_ref: AiProductionExecutionRouteSelectionRefV1
  context_ref: AiRunContextBundleRefV1
  workflow_binding_ref: AiWorkflowBindingRefV1
  security_attempt_reservation_ref: ArtifactRefV1 | null
  input_closure_hash: SHA-256
  reserved_budget_hash: SHA-256
  started_at: UtcTimestamp
  attempt_content_hash: SHA-256

AiProductionAttemptRefV1
  attempt_id: UUIDv7
  schema_version: 1
  attempt_content_hash: SHA-256

AiProductionAttemptOutcomeV1
  schema_version: 1
  outcome_id: StableId
  attempt_ref: AiProductionAttemptRefV1
  terminal_kind: succeeded | suspended | failed | cancelled | superseded
  completed_step_result_refs[]: ArtifactRefV1
  child_operation_task_refs[]: ArtifactRefV1
  child_run_refs[]: AiProductionRunRefV1
  model_or_host_output_artifact_ref: ArtifactRefV1 | null
  checkpoint_ref: AiOrchestrationCheckpointRefV1 | null
  diagnostic_refs[]: DiagnosticCodeRefV1
  ended_at: UtcTimestamp
  outcome_content_hash: SHA-256

AiProductionAttemptOutcomeRefV1
  outcome_id: StableId
  schema_version: 1
  outcome_content_hash: SHA-256
```

initial attemptはordinal 1かつprevious null、後続はprevious outcomeとordinal `N+1`を必須にする。`retry`はsame Workflow／Context／routeのretryable `failed` Outcome、`resume`はsame Workflow／Context／routeの`suspended` Outcomeとexact Checkpointだけをpreviousにできる。route、Context、Task Authorizationまたはsubject revisionが変わる場合はretry／resumeでなく新Runにする。AI Securityがsigned attempt reservation／repair headを要求するrouteまたはProposal acceptanceではexact reservationを参照し、Attempt変更、Host／Model／route変更、restartまたはresumeでTask-scoped repair countをresetしない。

`succeeded`はWorkflow stepがtyped outputを生成したことだけを表し、Run completed、Validation pass、Evidence admissible、Task Completed、Project CommitまたはReleaseを単独で意味しない。`suspended`はexact Checkpointだけをnon-nullにして同Attemptを閉じ、resume時は新Attemptを作る。`failed | cancelled | superseded`はsuccess outputを持たず、原因をtyped Diagnosticまたはsupersession reasonへ閉じる。`suspended`以外は`checkpoint_ref=null`とし、raw Credential、Model hidden state、unbounded transcriptまたはnative pointerをOutcomeへ保存しない。

### 8.3 Result

```text
AiProductionResultV1
  schema_version: 1
  result_id: StableId
  run_ref: AiProductionRunRefV1
  final_attempt_outcome_ref: AiProductionAttemptOutcomeRefV1
  result_kind:
    answer | proposal | validation_report | workflow_output
  output_contract_ref: McdContractRefV1(kind=type)
  output_artifact_refs[1..]: ArtifactRefV1
  project_or_bootstrap_subject_ref: ArtifactRefV1
  target_profile_refs[]: TargetProfileRefV1
  input_closure_hash: SHA-256
  emitted_at: UtcTimestamp
  result_content_hash: SHA-256

AiProductionResultRefV1
  result_id: StableId
  schema_version: 1
  result_content_hash: SHA-256
```

Resultはreceipt-free typed output carrierで、Evidenceまたはauthorityではない。`proposal`は未Commit ChangeSet、Source deltaまたはDomain Proposalのexact refだけを持ち、current Project revisionを進めない。`answer`はread-onlyでProject decisionにならず、`validation_report`はOwner-issued Validator resultを参照するがQualification Receiptを代替しない。後段Evidence、Approval、Commit、Promotion、BuildまたはPackageはResultを入力にできるが、別Ownerの別Record／Operationである。

### 8.4 append-only Run transition

```text
AiProductionRunStateV1 =
  prepared | active | waiting_input | waiting_operation |
  suspended | completed | failed | cancelled | superseded

AiProductionRunTransitionV1
  schema_version: 1
  transition_id: StableId
  run_ref: AiProductionRunRefV1
  sequence: positive uint64
  previous_transition_ref: AiProductionRunTransitionRefV1 | null
  from_state: AiProductionRunStateV1 | null
  to_state: AiProductionRunStateV1
  current_attempt_ref: AiProductionAttemptRefV1 | null
  attempt_outcome_ref: AiProductionAttemptOutcomeRefV1 | null
  waiting_input_contract_ref: McdContractRefV1(kind=type) | null
  waiting_operation_task_ref: ArtifactRefV1 | null
  checkpoint_ref: AiOrchestrationCheckpointRefV1 | null
  result_ref: AiProductionResultRefV1 | null
  diagnostic_refs[]: DiagnosticCodeRefV1
  transition_reason_ref: ArtifactRefV1
  transitioned_at: UtcTimestamp
  transition_content_hash: SHA-256

AiProductionRunTransitionRefV1
  transition_id: StableId
  schema_version: 1
  transition_content_hash: SHA-256

AiProductionRunHeadV1
  schema_version: 1
  head_id: StableId
  run_ref: AiProductionRunRefV1
  head_sequence: positive uint64
  head_transition_ref: AiProductionRunTransitionRefV1
  head_content_hash: SHA-256
```

Run Storeはexpected previous HeadへのCASでTransitionとHeadをatomic publicationするtarget semanticsを持つ。同じsequenceの別Transition、gap、fork、previous mismatch、terminal後Transitionを拒否する。これはlogical state contractであり、physical Store／database／serviceの選定ではない。

## 9. Run lifecycleと親Task境界

許可遷移は次だけである。

```text
none -> prepared
prepared -> active | failed | cancelled | superseded
active -> active | waiting_input | waiting_operation | suspended |
          completed | failed | cancelled | superseded
waiting_input -> active | failed | cancelled | superseded
waiting_operation -> active | failed | cancelled | superseded
suspended -> active | failed | cancelled | superseded
completed | failed | cancelled | superseded -> no transition
```

| state | 必須条件 | 意味しないこと |
|---|---|---|
| `prepared` | Workflow、Context、route、Authorization、budgetの開始前検証済み | Service起動、Capability Activation |
| `active` | exactly one current Attemptがstepを進行中 | Project write lock、Approval |
| `waiting_input` | Workflowで宣言したtyped User inputだけを待つ | Chat継続による権限維持 |
| `waiting_operation` | exact child `OperationTaskV1` terminal read-backだけを待つ | Operation成功の推測 |
| `suspended` | durable Checkpointとsame input closureでresume可能 | process memory保存、indefinite pause |
| `completed` | exactly one typed Resultを発行した | Task Completed、Commit、Promotion、Approval、Qualification、Release |
| `failed` | terminal failure Diagnosticが一件以上 | retry不能の一般化、Project変更 |
| `cancelled` | cancel closureがterminalへ収束 | 既Commit revisionのrollback |
| `superseded` | Project／Contract／Policy／Workflow／route等のdriftで再利用不能 | old result削除、migration完了 |

`completed`はResult ref、`failed`は一件以上のDiagnostic、`suspended`はCheckpointと`terminal_kind=suspended`のAttempt Outcome、`waiting_input`はinput contract、`waiting_operation`はOperation Task refを必須にし、他branch Fieldをnullにする。`active -> active`はretryable Outcomeからfinite retry policyに従って新Attemptへ交代する場合だけ許し、old Outcomeとnew current Attemptを同Transitionへexactに束縛する。`suspended -> active`もCheckpoint-bound resume Attemptを新規作成し、同じAttemptをprocess memoryから再開しない。RunはProposal生成で`completed`になり、Human Approval待ちのために生存させない。Approval後のCommitは別trusted Operationである。

親Governance Task stateはAI Securityが所有する。Run `completed`は親Taskを通常`Running`または`Validating`へ進めるinputにできるだけで、Task `Completed`を直接発行しない。read-only Task、proposal-only Task、mutation Taskのterminal output／read-back条件はSecurity Ownerが区別し、Model response、Run stateまたはEditor badgeからTask completionを推測しない。

## 10. Checkpoint、resume、retry、cancel

```text
AiOrchestrationCheckpointV1
  schema_version: 1
  checkpoint_id: StableId
  run_ref: AiProductionRunRefV1
  prior_run_transition_ref: AiProductionRunTransitionRefV1
  current_attempt_ref: AiProductionAttemptRefV1 | null
  workflow_binding_ref: AiWorkflowBindingRefV1
  context_ref: AiRunContextBundleRefV1
  route_selection_ref: AiProductionExecutionRouteSelectionRefV1
  current_step_id: StableId
  completed_step_result_refs[]: ArtifactRefV1
  completed_child_operation_task_refs[]: ArtifactRefV1
  completed_child_run_refs[]: AiProductionRunRefV1
  idempotency_record_refs[]: ArtifactRefV1
  secret_free_intermediate_refs[]: ArtifactRefV1
  remaining_budget_hash: SHA-256
  input_closure_hash: SHA-256
  created_at: UtcTimestamp
  expires_at: UtcTimestamp
  checkpoint_content_hash: SHA-256

AiOrchestrationCheckpointRefV1
  checkpoint_id: StableId
  schema_version: 1
  checkpoint_content_hash: SHA-256
```

CheckpointはModel hidden state、raw Prompt全文、Credential、native pointer、live Project writer、uncommitted filesystem handle、process IDまたはMCP connection stateを保存しない。Project revision、Contract、Policy、Workflow、Context input、route Profile、Target、Authorization、required Evidence freshnessがbyte-exactかつcurrentの場合だけ同じRunをresumeする。一件でも差があれば旧Runを`superseded`にし、新Context／route／Runを作る。

Checkpointはsuspend Transitionより前に公開済みの`prior_run_transition_ref`だけを参照し、その後のsuspend TransitionがCheckpointを参照する。Checkpointから同じTransitionを逆参照させず、Transition／Checkpointのcontent-hash cycleを作らない。

retryはWorkflow Definitionのfinite policyとretryable Diagnosticに限定し、新Attemptを作る。mutation Operationのtimeout後はOwner-issued Task／Receipt／publication stateをread-backし、結果不明のまま再dispatchしない。Model再試行は同じ出力を保証せず、response byte equalityまたは決定性を主張しない。

cancelはcurrent Attempt、cancel可能child Operation Task、明示child Runへ伝播し、bounded deadline後に`cancelled`またはtyped failureへ収束する。既に公開済みのProject revision、Source、Artifact、PackageまたはReceiptを暗黙rollbackしない。compensationまたはrollbackが必要なら、Owner登録済み別Operation、新Authorization、必要Approvalを用いる。

conversation再送、Modelの「続き」、同じHost／process、MCP reconnect、Editor tab復元または同名Workflowをresume Evidenceにしない。

## 11. Interaction intentとbounded execution profile

`AiProductionInteractionIntentV1`は`ask | suggest | execute_authorized`のclosed setである。これはUserが求めるinteractionの上限であり、Riskまたはauthority classではない。

Runの`interaction_intent`はAI Security所有のindividual intent bindingへexact解決し、`interaction_intent_binding_ref`、Task、subject、scope、expiryをbyte equalityにする。`execution_profile_kind`はcurrent `execution_profile_policy_ref`が許す一branchへ固定し、surface label、conversationまたはWorkflow名から補完しない。

| intent | 最大結果 | 禁止 |
|---|---|---|
| `ask` | bounded Query、explanation、typed answer | Proposal、mutation Operation |
| `suggest` | Proposal、Validation、Preview | Commit、Promotion、Activation |
| `execute_authorized` | current Authorization、Approval、Operation Policyが許すregistered Operation request | scope拡張、private writer、Release authority |

Prompt、SkillまたはModelはintentを自己昇格できない。`execute_authorized`を選んでも、Task Authorization、individual intent binding、Consent、Approval、Operation ActivationまたはSigner policyの欠落を埋めない。

`AiProductionExecutionProfileKindV1`は`interactive | bounded_workspace | release_candidate_assist`のclosed setである。

- `interactive`: User inputと一Runずつ進む通常経路。
- `bounded_workspace`: isolated workspace、finite Workflow、budget、checkpoint、allowed Operation、review pointを固定し、current／main Projectを直接変更しない。
- `release_candidate_assist`: exact Candidateに対するBuild／Test／Package／Evidence収集のProposalまたはtrusted requestを扱う。Signing、Publication、Release DecisionまたはCompletionを所有しない。

`Autonomous`、`full access`、`agent mode`、`release mode`等のsurface labelを追加authorityにしない。Publish、Store upload、Signing、ReleaseおよびProduct completionは常にProduct Release／Publication Ownerの別authorityである。

## 12. Workflow、Skill、Plugin、MCP、vendor surface

| concept | 正規位置付け | authorityにならないもの |
|---|---|---|
| Workflow | 本書のimmutable typed execution definition | Operation Activation、Authorization、Evidence pass |
| Skill | Workflow選択、入力収集、Tool利用を案内するguidance | allowed Operation、budget、stop condition、Result |
| Plugin | Skill、MCP connection、optional UI等の配布package | Engine Pack、Native Module、Capability Activation |
| MCP | external Agent向けOperation／Resource／Task transport Adapter | Engine API正本、Project state、permission source |
| Codex／Claude等のSDK／App Server | ClientまたはAgent Host Adapter候補 | Miraikanai Run／Task／Approval／session正本 |

Skill本文、description、example、scriptまたは成功宣言からWorkflow、Operation、Authorization、Approval、Evidence、Resultまたはstateを生成しない。SkillがWorkflowを参照する場合はexact Workflow Bindingを使い、Definitionを拡張しない。

Plugin install／enable／vendor署名だけでEngine Capabilityをactivateしない。AI PluginとEngine Plugin／Pack／Native Game Module／Runtime packageを同一kindにしない。Pluginなしでもmanual Editor／CLI／Native SDKのcanonical journeyを維持する。

MCP Tool、Prompt、Resource、Task extension、App UIはcanonical OperationまたはRunのtransport projectionである。MCP task handleを使う場合も、exact `AiProductionRunRefV1`または`OperationTaskV1`へのprojectionに限定し、MCP session／Taskをdurable authorityにしない。MCP protocol適合はSchema、Authorization、Host Conformance、Operation Activationまたは成功を代替しない。

`editor_builtin`、Native SDK、CLIおよびfirst-party Agent HostはEngine-owned typed local IPC／Native SDK projectionを使い、MCP Serverの存在を起動・authoring・Build／Test／Packageの前提にしない。`mcp`はstandard external Agent向けAdapterであり、built-in Console専用writer、内部Service discovery、Workflow storeまたはEngine内通信の正本にしない。ただし両projectionのouter Operation ref、payload meaning、Diagnostic、Run state、AuthorizationおよびResultは同一canonical contractから生成する。

vendor thread、session、subagent、permission mode、hook、plugin lifecycle、Model family名をMiraikanai正本型へ流入させない。Adapterを`supported`と表示できるのは、同じsurface policy、Workflow、Context、Operation、Diagnostic、Resultを評価するcurrent Conformance Receiptがある場合だけである。

<a id="ai-surface-parity-conformance"></a>

## 13. surface parityとConformance

`AiProductionSurfaceKindV1`は§3のclosed target setである。`AiAgentHostSurfaceKindV1`はそのうちAgent Host Profileを持つ`editor_builtin | first_party_agent_host | external_agent_host`のclosed subsetである。各surfaceは同じcanonical requestへ投影し、surface固有の表示、transport、streamingまたはauthenticationをsemantic Operationへ混ぜない。

```text
AiAgentHostSurfaceKindV1 =
  editor_builtin | first_party_agent_host | external_agent_host

AiAgentHostConformanceProfileV1
  schema_version: 1
  agent_host_profile_id: StableId
  agent_host_profile_version: positive u32
  ai_surface_kind: AiAgentHostSurfaceKindV1
  agent_product_identifier: StableId
  agent_product_version_identity: non-empty normalized UTF-8
  adapter_artifact_ref: exact ArtifactRefV1
  host_binding_ref: exact ArtifactRefV1
  transport_projection_kind:
    editor_projection | cli | native_sdk | mcp
  workflow_registry_ref: exact AiWorkflowRegistryRefV1
  allowed_workflow_refs[1..4096]:
    sorted unique exact AiWorkflowDefinitionRefV1
  allowed_operation_refs[1..65535]:
    sorted unique exact McdContractRefV1(kind=operation)
  surface_policy_content_hash: SHA-256
  conformance_requirement_artifact_ref: exact ArtifactRefV1
  agent_host_profile_content_hash: SHA-256

AiAgentHostConformanceProfileRefV1
  agent_host_profile_id: StableId
  agent_host_profile_version: positive u32
  agent_host_profile_content_hash: SHA-256
```

`AiAgentHostConformanceProfileV1.agent_host_profile_content_hash`はASCII domain separator `MIRAKAN_AI_AGENT_HOST_CONFORMANCE_PROFILE_V1`と§3のcanonical framingで、自己hashだけを除く全Fieldから計算する。Refは解決先のID、version、content hashとbyte equalityである。

Profileのclosed matrixは`editor_builtin→editor_projection`、`first_party_agent_host→cli | native_sdk`、`external_agent_host→mcp | cli | native_sdk`である。同じ製品名でもAgent version、adapter artifact、Host binding、transport、Workflow Registry、surface policyまたはOperation集合が異なれば別Profileとする。`latest`、vendor account、thread、session、display label、Model familyまたはtransportだけからProfileを補完しない。Codex、Claudeその他の製品名は`agent_product_identifier`のdataであり、canonical enum、authorityまたは互換保証ではない。

Profileは適合対象を固定するだけで、supported claimまたはConformance passではない。[Product Lifecycle](../00-product/product-lifecycle.md)がReleaseごとのrequired Client Profile集合とProduct surface `ai_automation`へのmappingを所有し、AI VerificationがProfileをsubjectとするcurrent Fixture／Suite／Receiptのidentity、freshness、revocationを所有する。generic `mcp` transport Fixture、Native SDK Fixture、同vendor別version、別Agentまたは別surfaceのReceiptをProfile passへ代用しない。現RepositoryにProfile Schema、Registry、Fixture、SuiteまたはReceiptは存在しない。

同じsubject、Workflow Binding、Run Context input、route class、Authorization、Project revision、surface policyを与えた場合、Conformanceは少なくとも次を検証する。

1. visible Operation集合がsurface policyとのset equalityである。
2. same semantic requestがsame outer Operation refとtyped payload meaningへ解決する。
3. unsupported、unauthorized、stale、not activatedが全surfaceで同じstable Diagnostic classになる。
4. Provider／MCP／Plugin固有hidden defaultがTarget、scope、Risk、budget、Project writeまたはCommit behaviorを変更しない。
5. proposal-only external surfaceへCommit、Promotion、Approval、Activation、Signing、Releaseを公開しない。
6. first-party Editorだけが利用できるprivate writerを作らない。
7. cancel、timeout、retry、partial result、disconnect後のstateとread-backが同じOwner ruleへ収束する。
8. locale、streaming、message orderまたはModel prose差をtyped result／authority差へ変換しない。

Model文章、token列、Tool call bytesまたはlatencyの同一性は要求しない。要求するのはcanonical semantic request、validation、authority、failure、result bindingの同値性である。Fixture／Suite／Receiptの具体shapeとpass判定はAI Verificationが所有し、現Repositoryには存在しない。

## 14. canonical data flowとOperation境界

```text
User intent / registered trigger
  -> Game production record or typed Task Specification
  -> AI Security Authorization
  -> Workflow selection and exact Binding
  -> one route and one loop owner
  -> immutable Run Context compilation
  -> bounded query / inference / proposal / child steps
  -> Engine-owned schema and semantic validation
  -> typed Proposal / Result
  -> Run completed
  -> separate Human Approval when required
  -> separate trusted Commit / Promotion / Build / Package Operation
  -> Project revision or immutable artifact publication
```

Workflowから呼べるのはExecutable Contractsのcurrent `operational_operations`に存在し、Workflow Binding、Task Authorization、surface policy、Owner policyの全intersectionへ含まれるexact Operationだけである。Operation IDをWorkflow名、Tool名、Provider annotation、Prompt、自然言語actionまたはMCP Resourceから生成しない。

外部Agent routeはQuery、Plan、Proposal、Validate、Previewまでを上限にする。外部HostがBuild／Test／Debugを希望する場合はtyped Proposalとして記録し、人間Approval後にEngine-owned trusted callerが別AuthorizationでOperationを依頼する。managed external Hostの将来output acceptanceはHost自身のmanaged job結果をStagingへ受理する別authorityであり、Engine Build Operationの外部dispatch権限ではない。

conversation response、Model memory、Skill output、MCP Task、Agent session、Run completedまたはWorkflow completedからProject revisionを進めない。人間のmanual編集とAI Proposalは同じProject ChangeSet、Validator、Approval、Commitを使う。

## 15. Security、Evidence、Project、Observability

### 15.1 Security

本書はTask Authorization、Caller Context、Approval、Consent、Credential、Risk、Role、Key、Signer PolicyまたはConformance Receiptを発行、署名、拡張、再分類しない。Workflow、route、Skill、Plugin、Model、User modeはSecurity scopeを拡張できない。AI Securityが所有する専用Generation Receipt Signerは、完成Orchestration recordをsubjectにして別途構成・Policy検証されたexact `GenerationReceiptPayloadV1` bytesだけを用途限定で署名し、Workflow Executor、Agent HostまたはOrchestrator control roleへKey handle、payload生成・変更、署名判断または他purpose authorityを渡さない。Provider CredentialをOrchestrator、Editor、Run Context、Prompt、Log、Checkpoint、ResultまたはReceipt本文へ渡さない。

### 15.2 Evidence

Run／Attempt／Workflow／ContextはEvidenceのcausal subjectになれるが、Evidence class、admissibility、freshness、revocationまたはpassを決めない。Model output、self-critique、conversation summary、visual impression、Run／Workflow completionをValidation、Test、Gameplay Approval、QualificationまたはRelease Evidenceへ読み替えない。

Run中に消費した既存EvidenceはContext inputとして固定できる。Run／Resultを評価して後から発行するGeneration／Review／Verification ReceiptはRunまたはResultを一方向参照し、Run／Resultへ埋め戻さない。これによりhash cycleと、後発Evidenceによる過去Resultの意味変更を禁止する。

### 15.3 Project

Run Context、Checkpoint、conversation、Model output cacheまたはsurface sessionはProject Sourceではない。Projectへ残すべきDecision、Brief、Spec、ChangeSet、Source、Test、Playtest、Receiptは各Ownerの正規recordへ明示promotionする。conversation履歴を削除してもProjectのcurrent stateと理由をProject Source／Decision Ledger／Receiptから再構成できなければならない。

### 15.4 Observability

Run、Attempt、Transition、step、child Operation、child Runのcausal identityをDebugging／Observabilityへtyped projectionできる。既定TelemetryへPrompt、Source、Asset、Tool output、Provider responseまたはconversation本文を保存せず、ID／hash／classification／state／duration／bounded countだけを送る。OpenTelemetryはexport Adapter候補であり、MiraikanaiのRun／Task／Evidence／Project型またはretention policyを置換しない。

## 16. local／cloud route、fallback、resource境界

first-party Agent routeはAI Security supplementの`InferenceDeploymentProfileV1`等のgoverned bindingによりcloud APIとfirst-party local inferenceを区別する。Model family名、file extension、endpoint到達性、GPU存在またはUser labelからrouteを選ばない。

本書はstep、model call、external roundtrip、Operation call、retry、child Run、Context byte、output byte、wall time、route-bound model usage unitおよびEngineが課金を観測できるrouteのmonetary budgetを所有する。CPU／GPU／RAM／VRAM、worker／queue capacity、reservation、loan、backpressure、measurementはPerformance Ownerが所有し、Route Selectionはそのcurrent decisionをopaque exact bindingとして消費する。resource requestまたはcapacity decisionが未materializeならfirst-party local routeを開始しない。

`external_host_owned_unattested` monetary／usage branchはstandard external Agentが外部Provider費用またはmodel usageをMiraikanaiへ証明しないことを明示する。Editorは両方を`unknown/external-host-owned`と表示し、0、free、budget-compliantまたはMiraikanai計測済みと表示しない。費用またはmodel usage上限をProduct requirementにするWorkflowはこのbranchを選べない。

local resource不足、runtime crash、Provider refusal、timeoutまたはnetwork failureを理由にcloud／別Provider／外部Hostへ暗黙fallbackしない。fallbackは別の完全なRoute Selectionであり、Provider、region、retention、training use、data class、cost、Model snapshot、Tool projection、capacityを再Previewし、新Caller Context、Task Authorization、Context、Run、initial Attempt、必要なUser confirmationを作る。旧Runをin-place変更またはresumeせず、`fallback_of_run_ref`と`fallback_of_route_selection_ref`を一方向に束縛する。fallback失敗時はProjectを不変にしてterminal failureへ閉じる。

外部Agentがlocal Modelを使用できることと、Miraikanaiがfirst-party local inference runtimeを配布・supportすることは別Capabilityである。一方のHost Conformance、Model Eval、license、successまたはActivationを他方へ流用しない。first-party local inferenceはProduct Futureのままで、MVPまたはFirst Playableの必須条件にしない。

## 17. Authoring AIとshipping Runtime AI

本OwnerはAuthoring Host／Toolingだけを対象にする。Runtime packageへAI Production Orchestrator、Agent Host、Workflow Registry、Run／conversation Store、Provider／Model Credential、MCP Server、Source Worker、Compiler、Signer、write-capable Project Gateway、Prompt templateまたはAuthoring Tool SDKを含めない。Build／Packageのinput closureとinspectionはこれらが0件であることを検証し、未参照だから安全と推測しない。

deterministic NPC／Gameplay AI、Navigation、behavior selectionはGameplay、Navigation、Simulationの既存Ownerに属する。将来shipping generative AIを追加する場合は、Product Future Portfolioから独立Capabilityとして昇格し、Runtime Owner、Threat Model、Privacy／network／cost／content safety、Target package、Save／Replay、offline failure、Service withdrawal、positive／negative Fixtureを一つのArchitecture変更で定義する。Authoring Provider Profile、Model Eval、Consent、Credential、Run Receipt、WorkflowまたはActivationを流用しない。

## 18. failure、drift、Diagnostic class

開始前に次を一件でも満たすRunを作らず、Projectとcurrent Run Registryを不変にする。

- missing／expired／revoked Authorization、Caller Context、Profile、ConformanceまたはConsent
- unresolved blocking Question、unknown Workflow revision、Registry差、unregistered／non-operational Operation
- stale Project、Contract、Policy、Tool Catalog、Context、EvidenceまたはTarget
- route branch mismatch、loop owner重複、ancestor recursion、child depth／count、budget不成立
- required input oversize、Sensitivity／redaction failure、unsupported bounded projection
- local Model license／provenance／capacity不成立、managed Host Future未Activation

実行中のProvider refusal、malformed structured output、Schema drift、Tool timeout、worker crash、resource exhaustion、network failure、disconnectまたはUser cancelはtyped Diagnosticへ閉じる。自然言語による成功補完、別Toolへの名称fallback、部分Proposalの自動Commit、別Project／Targetへの再対象化、missing outputのdefault化を行わない。

最低stable Diagnostic classは次を区別する。具体IDと共通EnvelopeはExecutable Contracts Ownerがmaterializeするまでcandidateである。

| class | condition |
|---|---|
| `workflow_unknown_or_stale` | Registry／Definition／Binding mismatch |
| `context_incomplete_or_stale` | input／revision／policy／expiry mismatch |
| `route_unavailable_or_mismatched` | route branch／Profile／Conformance mismatch |
| `loop_owner_conflict` | Agent loop重複、再帰、child bound違反 |
| `budget_exhausted` | finite bound超過またはcapacity denial |
| `structured_output_invalid` | output Schema／contract不適合 |
| `operation_not_available` | unregistered／non-operational／surface外Operation |
| `operation_result_unknown` | timeout後read-back未確定。再dispatch禁止 |
| `silent_fallback_forbidden` | explicit route／Authorizationなしのfallback |
| `resume_input_drift` | Checkpoint closureとcurrent input差 |
| `authoring_runtime_boundary_violation` | shipping packageへのAuthoring AI artifact混入 |

## 19. target Operation familyとcurrent empty state

本書のtarget semantic familyは次の四つである。これはOperation ID、planning work item、Registry memberまたはActivationを作らない。

| semantic family | target action | authority上限 |
|---|---|---|
| `ai_workflow_discovery` | Workflow／surface／route availabilityのbounded query | read-only。Registry membershipをactivateしない |
| `ai_run_control` | prepare／start／typed input／suspend／resume／cancel／status | Run stateだけ。Project mutationを直接行わない |
| `ai_context_projection` | authorized input assembly／staleness query | read-only derived Context。Credential／Project Sourceを返さない |
| `ai_result_read` | Result、Attempt、Transition、child handleのbounded read | Approval、Commit、Evidence passを発行しない |

将来Operationをmaterializeする場合は、Executable ContractsのMCD、Owner Manifest、Service allowlist、Security Policy、Validator、Diagnostic、Receipt、Signer、surface projection、positive／negative Fixtureをfamily単位でatomicに閉じる。本書から具体Operation ID、Service executable、Role、Key、Registry rowまたは実装順序を生成しない。

current stateは次のexact empty／absentである。

- 本書所有target MCD type／Ref family: materialized 0件
- Workflow Registry／Definition／Binding instance: 0件
- Run／Attempt／Transition／Checkpoint／Result instance: 0件
- AI Production Operationの`materialized | contract_active | active | operational`集合: 全て`[]`
- Orchestrator／Context Compiler／Route Resolver／Workflow Executor executable: absent
- Editor／CLI／SDK／MCP／Provider projection: 0件
- Conformance Fixture／Suite／Receipt: 0件
- first-party local inference runtime／loader: 0件／`not_activated`
- managed external Host execution Capability: `planning_only`／`not_activated`

initial V1でmaterialized predecessorがないため、旧candidate名、conversation例、temporary alias、legacy session、dual readerまたはmigration Operationを受理しない。初回public materialization後の変更はCompatibility／EvolutionのConsumer Inventory、versioned replacement、migration Evidenceを必要とする。

## 20. target-design検証と完了条件

Architecture target closureは少なくとも次を満たす。

1. 本Ownerの正本範囲、文書ID、全target type familyのOwnerがexact一件である。
2. 規範依存はHeaderのexact 8件で、Game Production LoopとEditor Workspace／UXだけが直接Authoring consumerになり、Owner graph cycleが0である。
3. Product claim、Security authority、Evidence、Operation、Process、Project transaction、Game domain、Editor UI、Performance、Runtime packageの意味を複写しない。
4. routeごとにcontrol-loop ownerがexactly one、Model routeではAgent-loop ownerもexactly oneである。
5. Workflow graph、retry、child Run、budget、cancel、resumeがfiniteで、Promptまたはconversation terminationに依存しない。
6. `completed` RunをTask Completed、Commit、Approval、Qualification、ReleaseまたはProduct completionへ読み替えない。
7. Resultと後発Evidenceの参照が一方向でhash cycleを持たない。
8. Editor、CLI、SDK、MCP、first-party／external Agentにprivate writerまたはhidden permissionがない。
9. local→cloud、Provider、Hostまたはroute fallbackが新Preview／Context／Authorization／Run／initial Attemptを要求する。
10. Authoring AI artifact／Credential／ServiceがRuntime packageから除外される。
11. local external Host Model、first-party local inference、shipping Runtime AIを相互代用しない。
12. current MCD、Registry、Operation、Service、Fixture、Receipt、Build、CI、Qualificationを存在すると表現しない。

本書のDesign review完了は、最高レベルのEngine、AI-native support、third-party usability、production readinessまたはProduct completionの証拠ではない。これらは将来のRepository artifact、実装、materialization、Target別Qualification、Release Evidenceが同じCandidateへ閉じた場合だけ各Ownerが判定する。

## 21. 一次資料と採用境界

外部資料はprotocol／product extension／observability／runtime候補の事実確認に用い、MiraikanaiのOwner配置、route、Workflow、stateまたはsecurityを外部組織の公式推奨とは表現しない。

- [ISO/IEC/IEEE 42010:2022](https://www.iso.org/standard/74393.html)
- [ISO 4217 currency codes](https://www.iso.org/iso-4217-currency-codes.html)
- [RFC 9562 — Universally Unique IDentifiers](https://www.rfc-editor.org/rfc/rfc9562.html)
- [Model Context Protocol 2026-07-28 specification](https://modelcontextprotocol.io/specification/2026-07-28)
- [MCP versioning](https://modelcontextprotocol.io/docs/2026-07-28/learn/versioning)
- [OpenAI Plugin architecture](https://developers.openai.com/plugins/concepts/plugins)
- [OpenAI Build skills](https://developers.openai.com/plugins/build/skills)
- [OpenAI Codex App Server](https://learn.chatgpt.com/docs/app-server)
- [Anthropic Claude Code features overview](https://code.claude.com/docs/en/features-overview)
- [ONNX Runtime GenAI](https://github.com/microsoft/onnxruntime-genai)
- [OpenTelemetry specification](https://opentelemetry.io/docs/specs/otel/)

ISO 4217とRFC 9562はcurrency codeとUUIDv7の外部format根拠、MCPはexternal Tool／Resource／Task transport、OpenAI／AnthropicのSkill／Plugin／Agent surfaceはvendor extension model、OpenTelemetryはobservability export、ONNX Runtime GenAIはevolving local inference候補である。いずれもMiraikanaiのProject authority、Workflow、Run state、Security Policy、EvidenceまたはProduct completionを所有しない。
