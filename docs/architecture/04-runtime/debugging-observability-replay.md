# Miraikanai Engine Debugging／Observability／Replay Contract

- 文書ID: mirakan.arch.runtime-debugging-observability-replay
- 状態: review
- 正本範囲: Debug Session、typed event／counter／snapshot、bounded Store／Index／Query、causality、breakpoint／watch／safe pause、deterministic capture／replay／rewind、crash／hang evidence、support bundle（構成artifact、redaction manifest、consent、生成operation）、remote device bridge、Editor Debug UX、AI diagnosis、Debug qualification
- 非正本範囲: Runtime phase／tick／lifetime、共通memory／performance／queue budget、AI Risk／authorization／approval、Evidence／Provenance envelope、Project transaction、Subsystem固有state schema、外部Tool／SDK version。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core architecture](../02-foundation/core-architecture.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Naming／Project layout](../02-foundation/naming-project-layout.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Project state](../03-authoring/project-state.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Editor UI Framework](../03-authoring/editor-ui-framework.md)、[Editor Workspace UX](../03-authoring/editor-workspace-ux.md)、[Gameplay programming model](../03-authoring/gameplay-programming-model.md)、[Scheduling／lifetime](scheduling-lifetime.md)、[Performance／capacity](performance-capacity.md)、[Physics](../05-simulation/physics.md)、[Collision](../05-simulation/collision.md)、[Navigation](../05-simulation/navigation.md)、[Animation](../05-simulation/animation.md)、[Render Graph](../06-rendering/render-graph.md)、[World](../06-rendering/world.md)、[VFX runtime](../06-rendering/vfx-runtime.md)、[Environment／surfaces](../06-rendering/environment-surfaces.md)、[Camera](../06-rendering/camera.md)、[Input](../07-platform/input.md)、[UI／Text／Localization／Accessibility](../07-platform/ui-text-localization-accessibility.md)、[Audio](../07-platform/audio.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論とauthority

Debuggingは自由文Logを大量に収集して原因を推測する機能ではない。Project revision、Build、Session、World、runtime time point、System、Entity、Asset、Command、Event、State delta、Diagnosticをtyped identityで相互参照するEngine-owned Evidence surfaceである。Console、Problems、Profiler、Timeline、Breakpoint、Watch、Overlay、Replay、Crash解析、AI diagnosisは同じcanonical recordを読む。

本書はDebug dataの生成／格納／検索／再現／投影を所有する。Runtime phase、tick、pause可能boundary、lifetimeは[Scheduling／lifetime](scheduling-lifetime.md)を消費する。runtime memory、frame、queue、instrumentation overhead、backpressureの合否は[Performance／capacity](performance-capacity.md)を消費し、本書で共通budgetを再定義しない。AI操作のRisk、authorization、approvalは[AI Security／Approval](../01-governance/ai-security-approval.md)、Evidence／Provenance／Receipt envelopeは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)だけが定義する。

Debug Queryはread-onlyである。AI Findingから修正候補を作る場合も、[Project state](../03-authoring/project-state.md)のChangeSet、Governance authorization、Staging、Test、Replay regressionを迂回しない。

## 2. 原則、Capability、構成

1. typed Event、Snapshot、Counter、Replay、Crash artifact、Build／Project hashを説明より優先する。
2. Screen pixel、Widget tree、pointer、表示名をidentityまたはcauseの正本にしない。
3. Editor、CLI、CI、AI、external adapterは同じrecordを投影する。
4. Pause、Watch、SnapshotはScheduling Ownerのsafe boundaryを破壊しない。
5. Event、rate、queue、query、capture、retention、AI contextをboundedにする。
6. drop、redaction、clock uncertainty、missing rangeをgapとして記録し、0件と区別する。
7. authoritative ReplayとPresentation／Platform timingの非決定性を区別する。
8. Shippingへdebug server、compiler、symbol、source mapping、credentialを偶発的に含めない。
9. IDE、GPU capture、system profilerはAdapterとして接続し、外部formatをcanonical Storeにしない。
10. Debugger自身の表示を唯一のsuccess oracleにせず、headless Query、Replay hash、fixture、Platform validationを併用する。

Capabilityは`diagnostic_foundation | local_first_playable | production_debugging | specialized_research`の順でProduct maturityへ関連付けるが、Product maturity自体は[Product Plan](../00-product/product-plan.md)を参照する。`specialized_research`は未記録状態の完全逆実行、AI無人修正、Multiplayer Debugを意味せず、それらは専用仕様とactivation前は`not_activated`である。

Instrumentation tierは`fault_minimal | baseline | interactive | capture`のclosed setとする。tierはDebug Session stateで、Project SourceまたはGameplay behaviorを変えない。high-rate captureがcapacityを満たせない場合はrecordをdrop／stopし、authoritative simulationを遅延させて成功扱いしない。

Build Configurationごとの利用可能channel、symbol、attach、exportはBuild ReceiptとCapability Catalogで宣言する。Shippingはfault／crashの最小registered recordだけを許可し、attach、expression evaluation、source-sensitive exportを提供しない。exact Tool／SDK／Platform versionは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)だけが固定する。

## 3. 論理アーキテクチャとprocess境界

```text
Runtime / Subsystem / Tool / Platform Adapter
  -> fixed-size Debug Ingress
  -> Flight Recorder / Trace Store / Counter Store / Replay Store
  -> Debug Index Service
  -> Editor / CLI / CI / External Adapter / bounded AI Context
  -> DebugFindingV1
  -> governed Authoring proposal
```

| process | responsibility | prohibited |
|---|---|---|
| EditorHost | Query、Index projection、Panel、AI context、Capture管理 | live Runtime pointer inspection |
| GameHost | record生成、flight recorder、Replay、safe pause | Project write、Provider接続 |
| WorkerHost | Build／Import／Shader／Test record、artifact ref | GameHost memory inspection |
| Device Bridge | authenticated handshake、bounded stream、capture transfer | Shipping常設、任意shell |
| Crash Collector | out-of-process dump ref、crash metadata、breadcrumb | Save mutation、AI呼出し |
| AI Orchestrator | bounded read-only Query、Finding、proposal handoff | raw Store access、Runtime mutation |

EditorHost failureでGameHost stateを壊さない。GameHost crash後もEditorHostはlast acknowledged record、Project revision、crash metadataを保持する。Store／Index／Panel failureを隠さずSession completenessをpartialまたはcrashedにする。

`mirakan_debug_contracts`はFoundation、generated MCD type、Stable／Contract／Artifact ref、session-scoped runtime refだけへ依存する。SubsystemはDebug ingress portを使い、Store、Editor、AIへ依存しない。Platform Adapterはvendor型をcommon contractへ漏らさない。Buildで無効なmoduleをempty-success stubへ置き換えずCapability unavailableを返す。

## 4. Identity、time、Session、event schema

`DebugTargetRefV1`は次を持つ。

| Field | Contract |
|---|---|
| `target_kind` | registered closed kind |
| `identity_kind` | `source \| contract \| artifact \| runtime_object \| domain_local` |
| selected identity | kindに対応するexact refを厳密に一つ |
| `parent_target_ref` | optional typed owner ref |
| `project_revision` | Source相関時のexact revision |
| `world_ref` | optional exact World ref |
| `display_label_key` | presentation only、identityへ使用禁止 |

runtime object refは`{debug_session_id, runtime_instance_id, handle{index,generation}}`で、当該Session外で比較／永続化しない。domain-local refはowner Artifact／Stable ref、Domain Contract ref、bounded valueを併記する。SourceとRuntimeの相関は二つのtarget refとtyped edgeで表し、一つのidentityへ混在させない。pointer、vendor handle、registration順をpublic identityにしない。

`DebugTimePointV1`はsession、monotonic sequence、Scheduling Ownerのruntime time ref、optional monotonic／wall time、clock domainを持つ。authoritative順序はmonotonic sequenceとRuntime orderを使い、wall timeを使わない。CPU、GPU、Audio、Device clockはoffset、drift、maximum error、valid intervalを持つ`ClockCorrelationV1`なしに同時刻とみなさない。

`DebugSessionDescriptorV1`はsession ID、Project／revision、Play session、Build Receipt ref、Target Profile、process set、tier、channel set、retention／privacy profile ref、capture reason、start／end、completeness、gap summaryを持つ。同一Sessionへ異なるProject revision、Build、Targetを混在させない。GameHost restartは新Sessionまたは明示child Sessionとする。

`DebugEventTypeV1`はtype ID／version、channel、category、bounded payload schema ref、maximum payload、priority、retention class、privacy class、causal policy、Shipping policyを登録する。未登録型、unbounded string／array、pointer、native handle、secret fieldをemitしない。large binaryはcontent-addressed Capture Artifactを参照する。

`DebugEventEnvelopeV1`はevent type ref、session、sequence、time point、producer process／thread role、target refs、correlation、bounded parent refs、trace／span／diagnostic ref、payload hash、payload、redaction flagsを持ち、canonical encodingとhashを使う。共通Evidence envelope fieldを埋め込まず、Governance Evidenceから本record hashを参照する。

## 5. Debug Session lifecycle

```text
Requested -> Validating -> Preparing -> Recording -> Finalizing -> Indexed -> Closed
Requested | Validating | Preparing -> Rejected
Recording -> PausedAtSafeBoundary -> Recording
Recording | PausedAtSafeBoundary -> Truncated | Crashed
Finalizing -> Partial
```

StartはrequestされたTarget、reason、tier、channel、duration、retention／privacy profileをGovernance authorization、Build Capability、Target、Performance capacity、device trust、consentへ照合し、fixed ingressを準備してdescriptorとstart markerをcommitしてからrecordを受け付ける。本書はRisk値、承認者、authorization envelopeを再定義しない。

Stopはexplicit request、profile limit、safe breakpoint、process exit、crash、critical recorder failureで開始し、producer停止、queue drain、footer、gap summary、artifact hash、Indexの順にfinalizeする。Crash時にnormal footerを捏造せずlast durable sequenceを記録する。Index失敗時にraw chunkを保持できても、AIへunindexed full scanを許可しない。

live pauseは[Scheduling／lifetime](scheduling-lifetime.md)のpublish safe boundary後だけ成立する。worker callback、Physics step、Audio callback、Render submission途中のWorldをInspectorへ公開しない。Pause中はauthoritative World read-only、Presentation camera／timelineは独立、GameplayDefinition node stepはcopied stateとdiscard-only outputを使う。

## 6. Log、Diagnostic、Span、Counter、Snapshot

`DebugLogRecordV1`はmessage key、typed primitive arguments、severity、category、target、source location ref、correlationを持つ。完成済み文字列だけを保存しない。repeated logはdedup key、first／last sequence、count、suppressed countへ集約する。

Diagnosticは[Executable contracts](../02-foundation/executable-contracts.md)の`MirakanDiagnosticV1`を参照し、code、expected／actual、location、remediation、cause chainをDebug eventへ複写しない。Platform validator IDはBackend evidenceとして保持し、Miraikanai Diagnostic codeと分離する。

`DebugSpanV1`はregistered name ID、parent、start／end、status、target、Runtime time ref、budget ref、actualを持つ。動的文字列やEntity IDをspan nameへ埋め込まない。異なるclock domainのspanはcorrelation errorを超えて厳密な順序を主張しない。

`DebugCounterDefinitionV1`はcounter ID／version、unit、value type、aggregation、scope kind、sampling policy、optional expected range、Performance budget ref、privacy、Shipping policyを持つ。aggregationは`gauge | monotonic_sum | histogram | duration | ratio`である。共通soft／hard budget値をcounter定義へ複写しない。Editor graphとheadless Gateは同じcounter IDと[Performance／capacity](performance-capacity.md)のbudget refを使う。

Subsystemは`DebugProjectionPortV1`からsafe-boundary後のbounded immutable `DomainDebugSnapshotEnvelopeV1`を公開する。envelopeはtype／version、session、time、source generation、targets、completeness、omitted field、payloadを持つ。Budgetを超えた場合は範囲を無通知で変えずpartialとomitted fieldを返す。

World debug shapeは`point | line | polyline | ray | aabb | obb | circle | sphere | capsule | cone | frustum | text_anchor`のregistered primitiveだけを許可する。target、time range、space、unit、style tokenを持ち、arbitrary mesh／shaderをDebug pathから実行しない。

## 7. Store、Index、bounded Query

canonical Storeはappend-only chunkである。Chunkはsession、channel、sequence／time range、schema set hash、compression ref、byte count、content hash、previous chunk hashを持つ。footer未完成chunkはlast complete recordまでrecoverする。CaptureをProject Sourceへ自動保存せず、User dataまたはCI artifact storeへ置く。external formatへexportしてもcanonical recordのretention stateを追跡する。

Indexはsequence／time、type／channel／severity、target hash／kind、Runtime time ref、System／Entity／Component／Asset、Diagnostic、trace／span／correlation／parent、breakpoint／watch、Replay checkpoint／state hashを持つ。session、Store root hash、schema set hash、index generationが一致しないIndexをstaleとして拒否する。

`DebugQueryV1`はexact session／index generation、bounded sequence／time range、最大64 target selector、最大256 type／channel selector、最大32 correlation ref、required field mask、canonical order、result count／byte limit、opaque cursor、registered aggregateを持つ。defaultは256 record／256 KiB、hard上限は4,096 record／4 MiBとする。AI Tool一回の返却は256 recordまたは256 KiBの小さい方で、広い調査はAggregate→narrow Query→Replay Sliceの順にする。

`DebugQueryResultV1`はrecordに加え、applied filter、scanned／matched／returned count、omitted field、first／last sequence、gap／redaction／clock uncertainty、completeness、next cursor、query cost、Store／Index hashを返す。0件、redacted、not recorded、dropped、not indexedを区別する。

Store ring、disk retention、ingress queue、capture throughput、instrumentation overheadはTarget別`DebugCapacityRequestV1`として[Performance／capacity](performance-capacity.md#51-debug-capacity-request)へ提出し、そのownerのtier別reservation／backpressure／measurementを消費する。本書は共通Runtime memory tableやframe thresholdを持たない。

## 8. Causality Graph

`DebugCausalEdgeV1.kind`は`input_produced | command_emitted | command_consumed | state_read | state_written | event_emitted | event_delivered | async_requested | async_accepted | job_scheduled | job_completed | resource_waited | rng_consumed | checkpoint_compared | asset_resolved | level_activated | presentation_derived | diagnostic_caused | fallback_selected`のclosed setとする。

edgeはsource／destination event、target、Runtime distance、delivery class、completenessを持つ。時系列上近いだけのeventをcausal edgeにしない。Parent／correlationが欠けた推定edgeは`inferred`とし、validated causeに使用しない。Presentationからauthoritative Gameplayへの逆edgeを作らない。

`CausalityGraphV1`は1～16 root event、direction、maximum depth／node、allowed edge kind、required time bound、canonical nodes／edges、unexpanded frontier、gap、completenessを持つ。default depth 8／node 512、hard上限depth 32／node 4,096とする。frontierを隠してcompleteと表示しない。

## 9. Breakpoint、Watch、Pause、Step

`DebugBreakpointV1`はID／version、enabled、optional Session scope、kind、target selector、registered pure predicate ref、hit policy、action、safe-boundary policy、capture channel、owner、expiryを持つ。kindはRuntime time、System／Definition entry／exit、command／event、state predicate、Diagnostic、budget threshold、Asset generation、Level transition、render capture triggerをregistered IDで表す。任意C++／Script式、filesystem、network、clock、random、World mutationをpredicateにしない。

hit policyは`first | every | after_count | every_n | once_per_target_generation | rate_limited`とし、hit／suppressed countとfirst／last hitを記録する。actionは`mark | capture | pause_at_safe_boundary | stop_recording | fail_qualification`である。

`DebugWatchV1`はStable target、MCD field path、sample boundary ref、interval、history capacity、comparison predicate、privacy classを持つ。pointer chain、native offset、任意expressionを使わない。target generation変更でretiredとし、新objectへ自動追従しない。containerはcount、hash、bounded sliceだけを返す。

`StepTick`はScheduling Ownerのcomplete fixed tickを最後まで実行してsafe boundaryへ戻る。`StepRenderFrame`はWorldを固定しPresentationだけを進める。GameplayDefinition node stepはcopied state／discard-only journalでだけ許可する。live phase途中step、live GPU event step、Engineによるsource-line stepを提供しない。

## 10. Deterministic capture、Replay、Rewind

authoritative Replayは[Scheduling／lifetime](scheduling-lifetime.md)が定めるProject／Build／Contract／System set、Input、RNG、accepted async resultとaccept time、command／event oracle、state hash、checkpoint、deterministic faultを記録する。OS raw packet、pointer、GPU output、wall time、Presentation cacheをauthoritative inputにしない。

`ReplaySliceV1`はslice ID、source Session、Build Receipt ref、Project revision、start checkpoint、start／end Runtime time ref、Input／async／RNG range ref、expected state hash／Diagnostic、required Asset version、redaction manifest、content hashを持つ。問題直前のcheckpointから観測可能な最小rangeを選ぶ。required closure不足をportable reproductionと表示しない。

Rewindはrecord済みSnapshot／Event／State sampleをtimelineで閲覧する機能で、live Worldを過去へ戻して継続する機能ではない。recorded object trackをscrubし、Overlay、Watch、Diagnostic、Causalityを同じtime pointへ同期する。current Projectとrecorded revisionを常時区別する。過去から再実行する場合はcheckpointからchild Replay Sessionを開始する。

divergence reportはfirst mismatch Runtime time、System／State Type／Field ID、expected／actual canonical hash、Input／RNG／accepted async差、Build／Contract／Asset／worker profile差、preceding causal frontier、record gapを含む。field valueを記録していない場合はhash差までとし、値を推測しない。

## 11. Domain Debug Projection

各Domainは共通Envelope、Target、Time、Queryを使い、自身のbounded snapshot schemaを所有する。

| Domain | required projection family |
|---|---|
| [Gameplay](../03-authoring/gameplay-programming-model.md) | state owner、entrypoint、command／event、delta、task、variant、budget ref |
| [World／Level](../06-rendering/world.md) | active Level、activation closure、cell ref、transition、last-valid state |
| [Physics](../05-simulation/physics.md)／[Collision](../05-simulation/collision.md) | body／shape ref、contact、sleep、constraint、query evidence |
| [Navigation](../05-simulation/navigation.md) | artifact／tile ref、agent、path、stale／failed result |
| [Animation](../05-simulation/animation.md) | graph state、transition、root motion、pose generation、event、LOD ref |
| [Rendering／Render Graph](../06-rendering/render-graph.md) | pass／resource ref、barrier、visibility、draw／dispatch、fallback |
| [Asset](../03-authoring/asset-lifecycle.md) | Source／Derived／Package version、dependency、residency、activation |
| [Input](../07-platform/input.md)／[UI／Text／Localization／Accessibility](../07-platform/ui-text-localization-accessibility.md)／[Audio](../07-platform/audio.md) | action／focus／route／voice／device、bounded callback evidence |
| [VFX runtime](../06-rendering/vfx-runtime.md)／[Environment／surfaces](../06-rendering/environment-surfaces.md)／[Camera](../06-rendering/camera.md) | artifact、seed、bounds、profile、fallback、owner-specific overlay |

Panel表示名をidentityにせず、typed targetからSource、Requirement、Decision、Build、Testへnavigationできるようにする。Domain固有field、Backend value、Qualificationは上表で直接Linkした各canonical Ownerへ委譲する。

## 12. Editor Debug Workspaceとexternal tool

Editor WorkspaceはSession、Console、Problems、Timeline、Causality、Breakpoints、Watch、Profiler、Replay／Rewind、Scene Debug、Reproduction、External Tools、AI Diagnosis panelを同じdockable workspaceへ置く。各Panelが独自log parser、clock、object identity、Storeを持たない。

操作はrecordからtarget／causeへ移動し、Causalityを展開し、Timeline／Overlay／Watch／Replayを同じtimeへ同期し、recorded stateとcurrent Sourceを並べる。AIへ送る前に対象、range、data category、redaction、推定context量をPreviewする。Evidenceへ戻れないAI claimをvalidated causeとして表示せず、proposalをPanelから直接applyしない。

severity、recorded／live、authoritative／derived、gapを色だけで区別しない。Timeline／Graph／Overlayと同じ情報をtable／tree／text summaryでも提供し、keyboard操作とscreen reader向けvirtualized collectionを持つ。

C++ source breakpoint、native stack／variable／thread／disassemblyはapproved IDEへ委譲する。Engineはprocess、executable hash、Build Receipt、symbol／source mapping Artifact ref、attach transport、Target device、security manifestを持つlaunch descriptorを生成し、一致を検証する。exact IDE／symbol Tool versionはToolchain Ownerを参照する。

GPU／system capture AdapterはEngine pass／resource targetをexternal markerへ写像し、capture hash、Toolchain ref、device／driver baseline ref、frame、Project revision、plan hashを登録する。external captureをProviderへ自動uploadしない。validation、shader print、private symbolをShippingへ含めない。

## 13. AI Debug ContextとDiagnosis

本節のDebug六ID、Task三ID、Build／Device／Play IDへのReceipt chainは、[Executable contracts](../02-foundation/executable-contracts.md)の`planning.operation_family.build_device_play_debug_task`に属する未Activation候補である。current MCD／Manifest／Service allowlist／Task／Receipt／Provider／MCP Tool集合は空で、候補IDをdispatchしない。以下のworkflow、payload、Support Bundle、fixtureは`activation.build_gateway.operation_pipeline.v1`が14件をatomic activateする場合の受入条件であり、現在利用可能なDebug workflowを主張しない。

`AiDebugContextV1`はquestion、Session、revision、Build Receipt、scope target、time range、selected Query／Causality／Replay／Diagnostic ref、capture metadata ref、redaction manifest、gap summary、context bytes／token estimate、allowed operation IDsを持つ。Activation後もAIはAggregateとProblemsから始め、target／timeを狭める。全Trace、全Project、全Sceneを一度に要求しない。

`DebugFindingV1`はstable ID／version、`observation | hypothesis | validated_cause | disproved | unresolved`、typed claim、scope、Evidence／counterevidence／gap／causal path ref、reproduction ref、falsification query ref、confidence band、next query、remediation ref、author refを持つ。validated causeにはreproduction refを必須とする。共通Evidence／Provenance fieldはGovernance envelopeから参照し、本型へ複写しない。

validated causeへ昇格できるのは、deterministic Replayとcounterfactual、Engine invariant／validator、exact Platform fault evidenceとDiagnostic cause chain、または人間ReviewerのEvidence承認のいずれかである。時間的相関、LLM confidence、pattern類似、Screenshotだけではhypothesisを超えない。

Activation後のDiagnosis workflowはscope／privacy Preview、Aggregate、narrow Query、Causality／Replay Slice、hypothesis／falsification、reproduce、Finding validation、governed proposal、関連Replay／test／performance regressionの順である。各段階を次のexact MCD Operationへ束縛し、表示だけの段階名やProvider独自Toolへ置換しない。

| 順序 | atomic activation時のreserved candidate | 必須入力binding | Result／次段 |
|---:|---|---|---|
| 1 | `operation.debug.aggregate` | Project revision、Candidate root、Target、Session、Build Receipt、Store／Index generation、bounded selector、Authorization | `DebugAggregateReceiptV1`からtarget／time／type候補を絞る |
| 2 | `operation.debug.query` | 同じidentity、exact Aggregate Receipt ref／hash、bounded `DebugQueryV1` hash、新しいAuthorization | `DebugQueryReceiptV1`とrecord／gap／redactionを得る |
| 3a | `operation.debug.read_causality` | 同じidentity、exact Query Receipt ref／hash、root Evidence refs、depth／node bound、新しいAuthorization | `DebugCausalityReceiptV1`とtyped causal subgraphを得る |
| 3b | `operation.debug.read_replay_slice` | 同じidentity、exact Build／Query／Causality Receipt ref／hash、Replay closure／range hash、新しいAuthorization | `ReplaySliceReceiptV1`とreproduction／divergence evidenceを得る |
| 4 | `operation.debug.validate_finding` | 同じidentity、exact Build／Query／Causality／Replay Receipt ref／hash、`DebugFindingV1` hash、Finding closure hash、新しいAuthorization | `DebugFindingValidationReceiptV1`がvalidityとexact proposal Operation refを返す |
| 5 | Receiptの`proposal_operation_ref` | validation Receipt、同じrevision／Candidate、Caller allowlist、R1以上の新Authorization | 対応familyも独立にatomic Activation済みの場合だけ、例としてGame Systemは`operation.systems.plan`、Worldは`operation.worlds.plan_change`でProposalを生成 |

Activation時のDebug Operation型固有payloadは次だけを持つ。共通identity、request、Authorization、result、Diagnostic、署名は[Core architecture §9.1](../02-foundation/core-architecture.md#91-operationtaskv1)のplanned `OperationReceiptEnvelopeV1`が所有する。

Build familyのActivationはSystems／World familyを暗黙activateしない。Step 4が妥当なFindingを得ても、対応する`planning.operation_family.game_system_discovery`または`planning.operation_family.world_discovery`が未Activationなら`proposal_operation_ref`を省略し、`decision=insufficient`と`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`を返してProjectを不変にする。文字列ID、別familyのread-only候補、Provider aliasへfallbackしない。

```text
DebugAggregateReceiptPayloadV1
  build_receipt_ref
  build_receipt_sha256
  session_ref
  store_generation
  index_generation
  selector_sha256
  aggregate_result_sha256?

DebugQueryReceiptPayloadV1
  debug_aggregate_receipt_ref
  debug_aggregate_receipt_sha256
  session_ref
  store_generation
  index_generation
  query_sha256
  record_slice_sha256?
  gap_summary_sha256?
  redaction_manifest_sha256?

DebugCausalityReceiptPayloadV1
  debug_query_receipt_ref
  debug_query_receipt_sha256
  session_ref
  index_generation
  root_evidence_refs[]
  bounds_sha256
  causal_graph_sha256?

ReplaySliceReceiptPayloadV1
  build_receipt_ref
  build_receipt_sha256
  debug_query_receipt_ref
  debug_query_receipt_sha256
  debug_causality_receipt_ref
  debug_causality_receipt_sha256
  session_ref
  replay_closure_sha256
  range_sha256
  replay_slice_artifact_ref?
  replay_slice_sha256?

DebugFindingValidationReceiptPayloadV1
  build_receipt_ref
  build_receipt_sha256
  debug_query_receipt_ref
  debug_query_receipt_sha256
  debug_causality_receipt_ref
  debug_causality_receipt_sha256
  replay_slice_receipt_ref
  replay_slice_receipt_sha256
  session_ref
  finding_sha256
  finding_closure_sha256
  decision? = valid | invalid | insufficient
  proposal_operation_ref?

SupportBundleReceiptPayloadV1
  build_receipt_ref
  build_receipt_sha256
  session_ref
  source_debug_receipts[] { receipt_ref, receipt_sha256 }
  policy_ref
  consent_record_ref
  redaction_manifest_ref
  support_bundle_ref?
  content_manifest_sha256?
  archive_artifact_ref?
  archive_sha256?
```

全Fieldは`?`を除き必須、`root_evidence_refs[]`と`source_debug_receipts[]`は1件以上、重複なしunsigned byte順で、unknown Fieldは禁止する。Envelopeの`result=succeeded`ではAggregate result、Queryのrecord／gap／redaction、Causality graph、Replay artifact／slice、Finding decision、Support Bundle／content manifest／archiveの各success output groupを全て必須にする。`result=failed | cancelled`では対応groupを全て省略し、`diagnostic_refs[]`を1件以上必須にする。`proposal_operation_ref`は後述のvalid条件だけで許可する。

`source_debug_receipts[]`は`DebugAggregateReceiptV1 | DebugQueryReceiptV1 | DebugCausalityReceiptV1 | ReplaySliceReceiptV1 | DebugFindingValidationReceiptV1`の`result=succeeded`完成Recordだけを受理する。各前段refは署名を含む完成`OperationReceiptEnvelopeV1` Record、その`*_sha256`は同じRecord hashでなければならない。全段のProject revision、Candidate root、Target、Session、Build Receipt、remote Device identity／generationをexact一致させる。Aggregate→Query→Causality→Replay Slice→Finding validationの順を短絡せず、前段のmissing、非success、hash／署名／operation ID／payload contract差、revocation、Store／Index generation差を後段で拒否する。

Finding validationにはCausalityとReplay Sliceの両Receiptを必須にする。Replayまたは必要Evidenceを生成できない場合は前段Diagnosticから新しいvalidation Taskを作り、`decision=insufficient`以外を返さず、`proposal_operation_ref`を省略する。`proposal_operation_ref`は`decision=valid`かつMCD登録済みのR1 proposal Operationが一意に解決した場合だけ許可し、汎用`operation.debug.propose`、Source write、Commitを生成しない。Callerの`AiDebugContextV1.allowed_operation_ids`、新しいAuthorization Envelope、全Receipt ref／hashが一致しなければStep 5へ進まない。

Activation後、追加instrumentationはchannel、tier、duration、capacity／privacy影響を提示し、Governance authorizationを得て開始する。同じblocking集合が減らない自動repairは2回で停止する。各Operationは[Core architecture](../02-foundation/core-architecture.md#91-operationtaskv1)の別`OperationTaskV1`であり、前段のread権限、Device binding、consentを後段へ継承しない。状態確認、Receipt取得、cancelは同時Activationした`operation.task.status`、`operation.task.read_receipt`、`operation.task.cancel`だけを使う。

AIはmissing eventをnon-occurrenceと断定せず、PresentationからGameplay causeを推定せず、異なるgenerationを同一objectとして連結せず、Debug BuildをShippingへ一般化せず、redacted valueをdefault／emptyとみなさず、recorded Buildとcurrent Sourceを混同しない。

## 14. Reproduction、Crash、Hang、remote device

`ReproductionBundleV1`はbundle ID／version、issue key、source Session、Project revision、Build／Target ref、optional Replay Slice、required Artifact／Diagnostic／Capture ref、expected oracle、run instruction ref、privacy／license／redaction manifest、expiry、content manifest hash、optional signatureを持つ。Project全体を無条件に複製せず、secret、credential、signing material、private clipboard、prompt、personal dataを含めない。Import時はhash、schema、Build availability、license、privacy、path、size、signatureを検証し隔離Workspaceで開く。

support bundleのschema、redaction、size bound、生成operation、failureは本書だけが所有する。`SupportBundleV1`は[Product Plan](../00-product/product-plan.md)のdiagnosis→support製品E2E終端を成すUser提出用bundleであり、開発内再現用の`ReproductionBundleV1`とは別概念で相互に代用しない。

```text
SupportBundleV1
  bundle_id: StableId
  schema_version: uint32 = 1
  session_ref
  project_revision
  build_receipt_ref
  target_profile_ref
  component_artifact_refs: SupportBundleComponentRefV1[1..64]
  redaction_manifest_ref
  consent_record_ref
  policy_ref: exact SupportBundlePolicyV1 ref
  uncompressed_size_bytes: uint64
  archive_size_bytes: uint64
  content_manifest_sha256
  generated_by_operation_id = operation.debug.support-bundle.generate
  signature_ref: optional

SupportBundleComponentRefV1
  component_kind: crash_evidence | hang_evidence | diagnostic_slice | log_slice | capability_summary | environment_summary
  artifact_ref
  content_sha256
  uncompressed_size_bytes: uint64
  data_class_refs[]

SupportBundlePolicyV1
  policy_id
  max_input_bytes: uint64
  max_archive_bytes: uint64
  max_file_count: uint32
  allowed_data_class_refs[]
  retention_policy_ref
```

`SupportBundleRedactionManifestV1`は`policy_ref`、入力component hash集合、Field／recordごとの`included | removed | transformed`、data class、rule ID、出力hash、omitted count、gap summaryを持つ。credential、token、private key、password、signing materialは変換せず収集段階で拒否する。redaction後bytesからcomponent／manifest hashとsizeを再計算し、入力hash、出力hash、bundle manifestが一致しなければexportしない。

`SupportBundleReceiptV1`は`OperationReceiptEnvelopeV1<SupportBundleReceiptPayloadV1>`の完成署名Recordである。BundleとReceiptのCandidate、Target、Session、Build Receipt、source Debug Receipt、consent、redaction manifest、content manifest、archive hashが一致しなければ成功にしない。

Atomic activation後の生成は`operation.debug.support-bundle.generate`だけが行い、対象Session、component Preview、data class、概算／上限bytes、redaction policy、提出先を表示して明示consentを得る。これは上記と同じOperation Registry、`OperationTaskV1`、task status／Receipt／cancel経路を使うexport branchであり、独自Task APIまたは自由形式Toolにしない。Aggregate／Query Receiptを入力component選択に使えるが、Support Bundle生成をFinding validationまたはProposal成功として扱わない。`max_input_bytes`、`max_archive_bytes`、`max_file_count`のいずれかを超える場合は切り詰めて成功扱いせず、対象rangeを狭める新Proposalを返す。最低failureは`diagnostic.debug.support-bundle-consent-required`、`diagnostic.debug.support-bundle-redaction-incomplete`、`diagnostic.debug.support-bundle-size-limit-exceeded`、`diagnostic.debug.support-bundle-artifact-unavailable`、`diagnostic.debug.support-bundle-manifest-mismatch`をclosed IDとして区別する。

Target別の生成UX、保存先、提出transportは各Platform Owner（[Windows](../07-platform/windows.md)、[Mobile Common](../07-platform/mobile-common.md)）が本schemaを投影する。Platform文書は収集可能componentとnative share UIだけを定義し、独自Support Bundle schema、緩いredaction、別size cap、silent uploadを作らない。

Crash recordはPlatform OwnerのCrash envelopeを参照し、Session、Build、last durable sequence、gap、Replay checkpoint、breadcrumbを関連付ける。in-process handlerはpreallocated metadataだけを書き、dump／symbol／Sourceを別Artifactにする。exact binary／module／symbol hashが一致する場合だけsymbolicateし、partial stackを推測補完しない。User actionとGovernance authorizationなしにonline service／Providerへ送らない。

Hang watchdogはsimulation、render submission、window、audio control、worker poolのheartbeatを分け、last progress、active Runtime time ref、bounded role stack ref、queue depth／oldest item、lock-order state、GPU submission status、last critical Diagnosticを記録する。threadを強制resumeして継続を成功扱いしない。

`HangDetectionPolicyV1`は`policy_id`、schema version、Target Profile ref、role entries、Application／debug execution state条件、evidence profile refを持つ。role entryは`role`、expected cadence ref、minimum missed interval count、minimum no-progress duration、active predicate、exempt state setを必須とする。C1の値は次で固定する。

| role | hang成立条件 | 明示除外 |
|---|---|---|
| simulation | Activeかつrunning中、進行なしが`max(120 × expected simulation interval, 2 s)` | gameplay pauseではなくdebug `paused_at_t110`、Inactive、Suspended、Terminating |
| render submission | 有効surfaceで進行なしが`max(120 × expected render interval, 2 s)` | headless、SurfaceUnavailable、Inactive、Suspended、Terminating |
| window | visible windowでmessage dispatch／present acknowledgement進行なし5 s | hidden／minimized、Inactive、Suspended、Terminating |
| audio control | active audio sessionでcontrol sequence進行なし2 s | audio session停止、Suspended、Terminating |
| worker pool | runnable itemがあり、oldest runnable age 10 s以上かつrole progressなし10 s | queue empty、全itemが外部I/O待機として登録済み、Suspended、Terminating |

watchdogは判定に用いたexpected interval、threshold、ApplicationState、debug execution mode、last progress sequenceをhang evidenceへ値で記録する。該当除外に入っただけで直前のsuspected hangを成功へ変えず、`cleared_by_progress | terminated | evidence_partial`の終端を記録する。Platform watchdogがより短い期限を課す場合は早期capture triggerとして併記するが、本Policyのrole判定をsilentに置換しない。

Remote handshakeはDevice identity、pairing generation、App／Engine／Module hash、Target、Debug Capability、channel、bandwidth／storage capacity ref、clock correlation、privacy stateを持つ。Development／Profile Buildだけがshort-lived mutual-authenticated Sessionで接続できる。Device Bridgeはfilesystem、shell、process、network proxyを提供しない。

transferはcontrolとbulk captureを分け、Counter／Diagnosticを優先し、disconnect時のlocal ring overflowをgapにする。resumeはlast acknowledged chunk hashから行い、sequenceでdeduplicateする。remote mutationのWatch、Capture、Pause／Resume、Replay controlは§13のDebug familyがatomic Activationされ、current MCD／Owner Manifest／Service allowlistへ同時登録された後だけ許可する。現在のremote-mutation Operation集合はexact 0件で、候補名やcontrol名によるdispatchを拒否する。C++／shader／native plugin変更はActivation後もrebuild／reinstallを必要とする。

## 15. Security、privacy、retention、failure

OperationごとのRisk、authorization、human approval、Provider／MCP／CLI boundaryは[AI Security／Approval](../01-governance/ai-security-approval.md)をそのまま消費する。Debug read権限をSource write、Runtime mutation、export、remote attach、Capability activationへ昇格させない。

data classはGovernance／Project privacy profileのregistered classを参照し、credential、token、private key、password、signing materialは収集自体を拒否する。Providerへ送る前にcategory、bytes、target、retention、ProviderをPreviewする。retentionはTarget別Profile refとPerformance capacity reservationを持ち、expiry後はPlatform deleteを実行してGovernance deletion Evidence refだけを残す。

Shipping package scanはDebug listener、expression evaluator、runtime compiler、private symbol／source map、validation layer、shader print、AI credential、unrestricted console、development signing identity、unredacted capture presetを拒否する。fault／crash最小recordもconsentとPrivacy Profileへ従う。

pressure時は`critical | high | normal | verbose`のregistered priorityを使い、verbose、normalの順でdropする。critical recordを保存できない場合はcaptureをtruncatedとして停止し、Qualificationを失敗させる。Debug pressureだけでauthoritative stateを変更しない。gapにはchannel、dropped count、first／last lost sequence、reasonを記録する。

最低Diagnostic familyはinvalid session config、capability unavailable、Build mismatch、store limit、event drop、critical recorder failure、stale Index、broad Query、target generation mismatch、unsafe pause、Replay divergence／closure missing、uncorrelated clock、privacy approval required、redaction incomplete、AI Evidence insufficient、remote auth failure、external capture unverifiedをclosed codeで区別する。Operation workflowは少なくとも`diagnostic.debug.aggregate-input-invalid`、`diagnostic.debug.query-input-invalid`、`diagnostic.debug.causality-input-invalid`、`diagnostic.debug.replay-slice-input-invalid`、`diagnostic.debug.finding-evidence-invalid`をclosed codeにする。Logだけで通知しない。

## 16. Test、AI Eval、qualification

Contract fixtureは全Typeのvalid／invalid／boundary、canonical encoding／hash／crash recovery、pointer／native handle／unbounded field／secret拒否、target generation、event registry、counter unit、Query／cursor／Index stale、gap／redactionを検証する。

Runtime fixtureは[Scheduling／lifetime](scheduling-lifetime.md)の全Runtime orderへのtime ref対応、parallel emitのcanonical sequence、priority drop、safe pause／complete tick step／render step、sandbox node step、Store／Panel crash非干渉、callback hot path allocation／block 0を検証する。instrumentation overhead、memory、disk、queue、soakは[Performance／capacity](performance-capacity.md)のcurrent Gateで測定する。

Replay fixtureはInput、RNG、accepted async resultから同じstate hash、first divergence、recorded／current revision分離、closure／Asset／worker mismatch拒否、gapを含むSessionのpartial表示、child Session isolationを検証する。

Atomic activation acceptance fixtureはAggregate→Query→Causality→Replay→Finding validation→exact domain Proposalのtask／Receipt chainを検証する。QueryのAggregate Receipt、CausalityのQuery Receipt、ReplayのBuild／Query／Causality Receipt、Finding validationのBuild／Query／Causality／Replay Receiptについて、missing、ref hash差、署名差、別operation payload、revocationを一原因ずつ注入して後段を停止する。stale Candidate、別Session／Build／Store／Index generation、remote Device交換、request hash／Authorization差でも拒否する。Evidence ref不在、別revision、gap／redaction隠蔽、時間相関だけ、reproductionなしの`validated_cause`を含む偽Findingは`operation.debug.validate_finding`で`diagnostic.debug.finding-evidence-invalid`となり、proposal Operation refを返さずProject stateを不変にする。Support Bundle branchは同じ署名Envelope／Task APIを使い、source Debug Receipt差、consentなし、redaction不完全、manifest mismatchでexport byteを公開しない。current fixtureは14 candidateすべてについてdispatch前の`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`とProject／Task／export byte不変を検査する。

`fixture.debug.known-faults`は少なくともInput context conflict、Collision filter、stale Nav result、root-motion authority conflict、Asset generation mismatch、Render barrier diagnostic、Audio pressure、Gameplay bounded-execution fault、Level closure不足、RNG divergence、GameHost crash／symbol mismatch、remote disconnect／gapを含む。各caseはobservation、typed Diagnostic、causal path、Replay Slice、correct remediation、forbidden remediation、regression fixtureを持つ。

AI Eval `debugging_diagnosis`はroot cause top-1 85%以上、top-3 95%以上、Blocking／High Evidence recall 100%、Evidenceなしvalidated cause 0、gap／redaction／revision／Presentation authority誤認0、unknown ID提出0、permission緩和0、2回超repair 0、unreproduced fixed claim 0、関連regression実行100%をC1 targetとする。Corpus／grader／3-run／Receipt構造は[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)だけが決定する。

Platform fixtureはCPU／GPU validation、crash／hang artifact、symbol hash、remote pairing／disconnect、redacted capture、minimum／reference device、Shipping package scanを各Platform Ownerのbaselineで実行する。外部captureだけをsuccess oracleにしない。

Debug qualificationはContract／Build／Target／instrumentation／retention／privacy／fixture／captureのexact refs、completeness／gap、Performance measurement ref、Contract／Runtime／Replay／Platform／AI Eval resultをGovernance Evidenceへ提出する。共通Receipt field、reviewer、approval、signature、freshnessは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を参照する。

## 17. Implementation sequenceとcompletion

実装artifactは`DBG0_contract -> DBG1_flight_recorder -> DBG2_editor_local -> DBG3_replay_causality -> DBG4_ai_diagnosis -> DBG5_remote_shipping`の依存順で進める。Product Phaseへの配置は[Product Plan](../00-product/product-plan.md)が所有し、本書はPhase番号を再定義しない。

- `DBG0_contract`: target／time／Session／event／counter／Query／gap schema、headless producer→Store→Index→Query。
- `DBG1_flight_recorder`: fixed ingress、priority、crash recovery、Runtime correlation、Performance capacity接続。
- `DBG2_editor_local`: local Panel、safe pause／step、Overlay、external IDE／GPU evidence mapping。
- `DBG3_replay_causality`: Replay Slice、divergence、Causality、Watch history、Reproduction Bundle。
- `DBG4_ai_diagnosis`: bounded Context、Finding validation、privacy Preview、governed proposal、AI Eval。
- `DBG5_remote_shipping`: authenticated Device Bridge、resume／gap、Platform Adapter、Shipping scan。

C1 completionにはexact Session identity、registered record、one-record multiple projection、bounded Query、gap／redaction／clock uncertainty、safe pause／step、deterministic Replay、first divergence、minimal Reproduction Bundle、external evidence mapping、Evidence-backed AI Finding、governed repair、Performance capacity合格、Shipping artifact scan、known-fault／AI Eval合格が必要である。

## 18. 明示的に採用しないもの

- 汎用memory editor、arbitrary expression／SQL／script console、Runtime mutation shell。
- Shipping常設debug server、compiler、symbol、source map、unredacted telemetry。
- Source／GPU debugger、Profiler、symbolicatorの全面自作と外部formatの正本化。
- native pointer、vendor handle、display name、pixel、Widget pathをidentityにすること。
- unsafe phase pause、live phase step、recordしていない過去stateの推測復元。
- record gap、redaction、clock uncertainty、stale Indexを隠したvalidated cause。
- Debug権限によるauthorization、approval、privacy、Source transactionの迂回。
- 共通Runtime budget、Risk table、Evidence／Provenance envelopeの本書での複写。
- Multiplayer Debug、online crash aggregation、AI無人修正のactivation前実装。
