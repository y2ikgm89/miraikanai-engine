# Miraikanai Engine Debugging／Observability／Replay Contract

- 文書ID: mirakan.arch.runtime-debugging-observability-replay
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Debug Session、typed event／counter／snapshot、Runtime Entry／Stage／World spatial／UI Screen transition event、bounded Store／Index／Query、causality、breakpoint／watch／safe pause、deterministic capture／replay／rewind、crash／hang evidence、support bundle（構成artifact、redaction manifest、consent、生成operation）、remote device bridge、Editor Debug UX、AI diagnosis、Debug qualification
- 非正本範囲: Runtime phase／Simulation Advance／lifetime、Runtime ECS storage／live query・lease、Save／Replay semantic record・digest・reconstruction、Runtime Package binary、共通memory／performance／queue budget、AI Risk／authorization／approval、Evidence／Provenance envelope、Project transaction、Subsystem固有state schema、外部Tool／SDK version。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Scheduling／Lifetime](scheduling-lifetime.md)、[Performance／Capacity](performance-capacity.md)
- 関連文書: [Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[Core architecture](../02-foundation/core-architecture.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Naming／Project layout](../02-foundation/naming-project-layout.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Project state](../03-authoring/project-state.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Editor UI Framework](../03-authoring/editor-ui-framework.md)、[Editor Workspace UX](../03-authoring/editor-workspace-ux.md)、[Gameplay programming model](../03-authoring/gameplay-programming-model.md)、[Runtime ECS](entity-component-system.md)、[Runtime Package](runtime-package.md)、[Persistence／Save](persistence-save.md)、[Scheduling／lifetime](scheduling-lifetime.md)、[Performance／capacity](performance-capacity.md)、[Physics](../05-simulation/physics.md)、[Collision](../05-simulation/collision.md)、[Navigation](../05-simulation/navigation.md)、[Animation](../05-simulation/animation.md)、[Render Graph](../06-rendering/render-graph.md)、[LOD](../06-rendering/lod.md)、[Virtualized／Continuous Geometry](../06-rendering/virtualized-continuous-geometry.md)、[World](../06-rendering/world.md)、[VFX runtime](../06-rendering/vfx-runtime.md)、[Environment／surfaces](../06-rendering/environment-surfaces.md)、[Camera](../06-rendering/camera.md)、[Input](../07-platform/input.md)、[UI／Text／Localization／Accessibility](../07-platform/ui-text-localization-accessibility.md)、[Audio](../07-platform/audio.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-21

## 1. 結論とauthority

Debuggingは自由文Logを大量に収集して原因を推測する機能ではない。Project revision、Build、Session、World、runtime time point、System、Entity、Asset、Command、Event、State delta、Diagnosticをtyped identityで相互参照するEngine-owned Evidence surfaceである。Console、Problems、Profiler、Timeline、Breakpoint、Watch、Overlay、Replay、Crash解析、AI diagnosisは同じcanonical recordを読む。

本書はDebug dataの生成／格納／検索／capture transport／表示を所有する。Runtime phase、Simulation Advance、pause可能boundary、lifetimeは[Scheduling／lifetime](scheduling-lifetime.md)、Entity／Component identityとsealed ECS projectionは[Runtime ECS](entity-component-system.md)、Save／Replay semantic record・digest・reconstructionは[Persistence／Save](persistence-save.md)を消費する。runtime memory、frame、queue、instrumentation overhead、backpressureの合否は[Performance／capacity](performance-capacity.md)を消費し、本書で共通budgetを再定義しない。AI操作のRisk、authorization、approvalは[AI Security／Approval](../01-governance/ai-security-approval.md)、Evidence／Provenance／Receipt envelopeは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)だけが定義する。

Debug Queryはread-onlyである。AI Findingから修正候補を作る場合も、[Project state](../03-authoring/project-state.md)のChangeSet、Governance authorization、Staging、Test、Replay regressionを迂回しない。

Runtime ECS正本化ではDebug transportとAI captureを`observability_projection` consumerとして[Compatibility／Evolution](../02-foundation/compatibility-evolution.md#42-ecs-consumer-inventory-boundary)へ列挙する。これは現行reader、公開capture、retention windowの存在を主張しない。対象formatを読む／保持するconsumerがあるか、redacted projectionをsourceから再生成できるか、external captureがあるかをcomplete Consumer Inventoryとscope Requirementのpass fulfillmentで確定するまで、clean breakを承認しない。

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

`DebugTimePointV1`はsession、monotonic sequence、Scheduling Ownerのexact `RuntimeTimeRefV1`、optional monotonic／wall time、clock domainを持つ。authoritative順序はmonotonic sequenceとRuntime orderを使い、wall timeを使わない。simulation branchはCadence Profile Ref、Interval Ref／completed SHA、advance sequence、optional `TickPhaseId`、presentation branchはrender frame ID、optional `RenderPhaseId`だけを使い、両branch Fieldを混在させない。CPU、GPU、Audio、Device clockはoffset、drift、maximum error、valid intervalを持つ`ClockCorrelationV1`なしに同時刻とみなさない。

```text
DebugSessionDescriptorRefV1
  session_id: StableId
  descriptor_version: positive uint32
  descriptor_content_hash: SHA-256

DebugSessionStartPointV1
  start:
    kind: before_first_runtime_time
      start_record_sequence: uint64 = 0
    | kind: correlated_runtime_time
      start_record_sequence: uint64 = 0
      runtime_time_ref: RuntimeTimeRefV1

DebugSessionDescriptorV1
  session_id: StableId
  descriptor_version: positive uint32
  project_id: UUIDv7
  project_revision: positive uint64
  project_document_set_hash: SHA-256
  play_session_id: StableId
  game_candidate_build_receipt_ref: GameCandidateBuildReceiptRefV1
  game_candidate_build_receipt_sha256: SHA-256
  target_profile_ref: TargetProfileRefV1
  initial_process_set_hash: SHA-256
  instrumentation_tier:
    fault_minimal | baseline | interactive | capture
  channel_set_hash: SHA-256
  retention_profile_ref
  privacy_profile_ref
  capture_reason_ref
  start_point: DebugSessionStartPointV1
  descriptor_content_hash: SHA-256

DebugSessionClosureRefV1
  session_id: StableId
  closure_version: positive uint32
  closure_content_hash: SHA-256

DebugSessionClosureV1
  session_id: StableId
  closure_version: uint32 = 1
  debug_session_ref: DebugSessionDescriptorRefV1
  end_record_sequence: uint64
  end_runtime_time_ref: null | RuntimeTimeRefV1
  completeness: complete | partial | truncated | crashed
  gap_summary_hash: SHA-256
  closure_content_hash: SHA-256
```

`DebugSessionDescriptorV1`は開始時に一度だけcommitするimmutable identity recordであり、停止、gap、crash、Index完成によってversionまたはhashを更新しない。`start_point.kind=before_first_runtime_time`はGameHost／headless Sessionが最初のSimulation AdvanceまたはPresentation frameより前に開始するbranchで、Runtime timeを捏造しない。`correlated_runtime_time`だけが開始時に既にcompletedなexact `RuntimeTimeRefV1`を持てる。両branchの`start_record_sequence`は0であり、最初のDebug recordは1から始める。`descriptor_content_hash`はASCII `MIRAKAN_DEBUG_SESSION_DESCRIPTOR_V1`と同Fieldだけを除くclosed DescriptorのMCD canonical bytesを`uint32_be` length framingしてSHA-256し、Refは完成DescriptorのID／version／self-excluding hashからrecord外でmaterializeする。Build Receipt Refはcompleted signed recordへ解決し、隣接SHAは同じcompleted record全bytesのSHA-256である。同一Sessionへ異なるProject triple、Build、Targetを混在させない。GameHost restartは新Descriptorまたは明示child Sessionとする。

`DebugSessionClosureV1`はproducer停止とdurable record境界確定後に同一Sessionへexact一件だけ作るimmutable終了Recordである。`session_id`はDescriptorとbyte equality、`end_record_sequence`は0以上のlast durable Debug record sequence、`end_runtime_time_ref`は終了時にcompleted Runtime timeが存在する場合だけnon-nullにする。最初のRuntime time前の拒否／crash、strict headlessのtime-less lifecycle終端へ偽のsimulation／presentation branchを作らない。`closure_content_hash`はASCII `MIRAKAN_DEBUG_SESSION_CLOSURE_V1`と同Fieldだけを除くclosed Closureのcanonical bytesから計算し、Closure Refは完成Recordからrecord外でmaterializeする。Closure未存在は`recording | finalizing`を意味し得るだけで、`complete`と推測しない。Replay Bundle、Range Artifact、Slice、Debug Receiptは不変Descriptor Refを束縛し、終了後の完全性／gapを判定するconsumerだけが対応Closure Refを別入力として要求する。Closure RefをDescriptorまたはReplay Bundle hashへ埋め戻さない。

`DebugEventTypeV1`はtype ID／version、channel、category、bounded payload schema ref、maximum payload、priority、retention class、privacy class、causal policy、Shipping policyを登録する。未登録型、unbounded string／array、pointer、native handle、secret fieldをemitしない。large binaryはcontent-addressed Capture Artifactを参照する。

`DebugEventEnvelopeV1`はevent type ref、session、sequence、time point、producer process／thread role、target refs、correlation、bounded parent refs、trace／span／diagnostic ref、payload hash、payload、redaction flagsを持ち、canonical encodingとhashを使う。共通Evidence envelope fieldを埋め込まず、Governance Evidenceから本record hashを参照する。

画面／進行切替の観測は曖昧な`Level transition` eventを使わず、次のtyped payloadを登録する。

```text
DebugTransitionEventV1
  transition_kind:
    runtime_entry_transition
    | stage_transition
    | world_space_transition
    | ui_screen_navigation
  transition_phase:
    requested | preparing | ready | committed | rejected | cancelled_before_commit
  request_ref: exact typed transition | navigation request ref
  source_target_ref: DebugTargetRefV1
  destination_target_ref: DebugTargetRefV1 | null
  source_generation: positive uint64
  destination_generation: positive uint64 | null
  boundary_runtime_time_ref: RuntimeTimeRefV1 | null
  final_result_ref: exact typed owner result | receipt ref | null
  diagnostic_refs[0..64]
  transition_event_content_hash: SHA-256
```

`requested | preparing | ready`はdestination generation／boundary time／final resultをnull、`committed`は三Fieldをpresent、`rejected | cancelled_before_commit`はdestination generation／boundary timeをnullとしてexact final resultをpresentにする。final resultはOwnerが定めるReceipt、boundary result、published Stack／Stage／World generation refのclosed variantであり、Debug ownerが疑似Receiptを作らない。`transition_event_content_hash`はASCII `MIRAKAN_DEBUG_TRANSITION_EVENT_V1`と自身を除く全FieldのMCD canonical bytesをlength framingして計算する。source／destination refは各OwnerのStable／typed identityへ解決し、display screen name、Scene path、Level名、pixel、Widget pathをtargetにしない。同じrequestのphase eventはrequest refとRuntime monotonic sequenceで相関し、Loading表示や時間的近接からcommitを推測しない。

`DebugTransitionEventV1`はtarget review event typeであり、current Event Type Registry、Breakpoint Registry、MCD Contract Set、capture／query Operation inventoryを本節だけで変更しない。四Ownerのsource event／result schema、Consumer Inventory、Definition Migration、redaction／capacity／qualificationが同じclosureで揃うまで、current V1の`Level transition`をtarget Event／Breakpoint／Evidenceへ流用して暫定availableにしない。

## 5. Debug Session lifecycle

```text
Requested -> Validating -> Preparing -> Recording -> Finalizing -> Indexed -> Closed
Requested | Validating | Preparing -> Rejected
Recording -> PausedAtSafeBoundary -> Recording
Recording | PausedAtSafeBoundary -> Truncated | Crashed
Finalizing -> Partial
```

StartはrequestされたTarget、reason、tier、channel、duration、retention／privacy profileをGovernance authorization、Build Capability、Target、Performance capacity、device trust、consentへ照合し、fixed ingressを準備してimmutable Descriptorとstart markerをcommitしてからrecordを受け付ける。本書はRisk値、承認者、authorization envelopeを再定義しない。

Stopはexplicit request、profile limit、safe breakpoint、process exit、crash、critical recorder failureで開始し、producer停止、queue drain、footer、gap summary、artifact hash、Index、immutable Closureの順にfinalizeする。Crash時にnormal footerを捏造せずlast durable sequenceをClosureへ記録する。Index失敗時にraw chunkを保持できても、AIへunindexed full scanを許可しない。finalization retryは同じDescriptor Ref、end sequence、optional end Runtime time、completeness、gap hashから同じClosure bytesを再現し、別Closure version、latest探索、Descriptor更新を行わない。

live pauseは[Scheduling／lifetime](scheduling-lifetime.md)のpublish safe boundary後だけ成立する。worker callback、Physics step、Audio callback、Render submission途中のWorldをInspectorへ公開しない。Pause中はauthoritative World read-only、Presentation camera／timelineは独立、GameplayDefinition node stepはcopied stateとdiscard-only outputを使う。

## 6. Log、Diagnostic、Span、Counter、Snapshot

`DebugLogRecordV1`はmessage key、typed primitive arguments、severity、category、target、source location ref、correlationを持つ。完成済み文字列だけを保存しない。repeated logはdedup key、first／last sequence、count、suppressed countへ集約する。

Diagnosticは[Executable contracts](../02-foundation/executable-contracts.md)の`MirakanDiagnosticV1`を参照し、code、expected／actual、location、remediation、cause chainをDebug eventへ複写しない。Platform validator IDはBackend evidenceとして保持し、Miraikanai Diagnostic codeと分離する。

`DebugSpanV1`はregistered name ID、parent、start／end、status、target、exact `RuntimeTimeRefV1`、budget ref、actualを持つ。動的文字列やEntity IDをspan nameへ埋め込まない。異なるclock domainのspanはcorrelation errorを超えて厳密な順序を主張しない。

`DebugCounterDefinitionV1`はcounter ID／version、unit、value type、aggregation、scope kind、sampling policy、optional expected range、Performance budget ref、privacy、Shipping policyを持つ。aggregationは`gauge | monotonic_sum | histogram | duration | ratio`である。共通soft／hard budget値をcounter定義へ複写しない。Editor graphとheadless Gateは同じcounter IDと[Performance／capacity](performance-capacity.md)のbudget refを使う。

### 6.1 Runtime ECS data-oriented evidence projection

Debuggingは[Performance／capacity](performance-capacity.md)所有の`runtime_ecs_data_oriented_metrics_v1`に登録されたexact metric IDを使い、次のmetric familyを欠落なく投影する。

- callback general-heap allocation and upstream fallback
- reserved, committed, live, and peak bytes
- chunk count, row capacity, occupied rows, and unused payload bytes
- archetype count and archetype fragmentation
- selected rows, contiguous work units, and chunk transitions
- exposed column bytes and useful selected payload bytes
- query-cache hit, miss, rebuild, and invalidation
- structural moved rows and structural copy bytes
- handle resolve and lease-validation P50／P95／P99
- scenario CPU P50／P95／P99
- semantic result hash, publication hash, and failure atomicity

local alias、sampling duration、percentile algorithm、pass thresholdは定義しない。各sampleはexact campaign、scenario、payload candidate、Candidate、Target、Contract Set、Toolchain、fixture、input traceのidentityをすべて持ち、欠けたmemberを0へ正規化しない。

Replay、Debug panel、AI summaryはsealed snapshotだけを読む。counter、panel、Replay event、AI summaryからlive Component layout、query membership、structural capacity、authoritative stateへのauthority back-edgeを作らない。

Subsystemは`DebugProjectionPortV1`からsafe-boundary後のbounded immutable `DomainDebugSnapshotEnvelopeV1`を公開する。envelopeはtype／version、session、time、source generation、targets、completeness、omitted field、payloadを持つ。Budgetを超えた場合は範囲を無通知で変えずpartialとomitted fieldを返す。

World debug shapeは`point | line | polyline | ray | aabb | obb | circle | sphere | capsule | cone | frustum | text_anchor`のregistered primitiveだけを許可する。target、time range、space、unit、style tokenを持ち、arbitrary mesh／shaderをDebug pathから実行しない。

## 7. Store、Index、bounded Query

canonical Storeはappend-only chunkである。Chunkはsession、channel、sequence／time range、schema set hash、compression ref、byte count、content hash、previous chunk hashを持つ。footer未完成chunkはlast complete recordまでrecoverする。CaptureをProject Sourceへ自動保存せず、User dataまたはCI artifact storeへ置く。external formatへexportしてもcanonical recordのretention stateを追跡する。

Indexはsequence／time、type／channel／severity、target hash／kind、exact `RuntimeTimeRefV1`、System／Entity／Component／Asset、Diagnostic、trace／span／correlation／parent、breakpoint／watch、Replay checkpoint／state hashを持つ。session、Store root hash、schema set hash、index generationが一致しないIndexをstaleとして拒否する。

`DebugQueryV1`はexact session／index generation、bounded sequence／time range、最大64 target selector、最大256 type／channel selector、最大32 correlation ref、required field mask、canonical order、result count／byte limit、opaque cursor、registered aggregateを持つ。defaultは256 record／256 KiB、hard上限は4,096 record／4 MiBとする。AI Tool一回の返却は256 recordまたは256 KiBの小さい方で、広い調査はAggregate→narrow Query→Replay Sliceの順にする。

`DebugQueryResultV1`はrecordに加え、applied filter、scanned／matched／returned count、omitted field、first／last sequence、gap／redaction／clock uncertainty、completeness、next cursor、query cost、Store／Index hashを返す。0件、redacted、not recorded、dropped、not indexedを区別する。

Store ring、disk retention、ingress queue、capture throughput、instrumentation overheadはTarget別`DebugCapacityRequestV1`として[Performance／capacity](performance-capacity.md#51-debug-capacity-request)へ提出し、そのownerのtier別reservation／backpressure／measurementを消費する。本書は共通Runtime memory tableやframe thresholdを持たない。

## 8. Causality Graph

current `DebugCausalEdgeV1.kind`は`input_produced | command_emitted | command_consumed | state_read | state_written | event_emitted | event_delivered | async_requested | async_accepted | job_scheduled | job_completed | resource_waited | rng_consumed | checkpoint_compared | asset_resolved | level_activated | presentation_derived | diagnostic_caused | fallback_selected`のclosed setを維持し、`level_activated`をRuntime Entry／Stage／World spatial／UI navigationのいずれかへ読み替えない。target `DebugCausalEdgeV2.kind`はV1の`level_activated`一件を削除し、`runtime_entry_transitioned | stage_transitioned | world_space_transitioned | ui_screen_navigated`の四件を追加したclosed setとする。他のV1 kindは同名で維持する。V1→V2はEvent Type Registry、Query／Index、Replay／Causality reader、Breakpoint、fixtureを同時移行し、V2に`level_activated` aliasを残さない。

edgeはsource／destination event、target、Runtime distance、delivery class、completenessを持つ。時系列上近いだけのeventをcausal edgeにしない。Parent／correlationが欠けた推定edgeは`inferred`とし、validated causeに使用しない。Presentationからauthoritative Gameplayへの逆edgeを作らない。

`CausalityGraphV1`は1～16 root event、direction、maximum depth／node、allowed edge kind、required time bound、canonical nodes／edges、unexpanded frontier、gap、completenessを持つ。default depth 8／node 512、hard上限depth 32／node 4,096とする。frontierを隠してcompleteと表示しない。

## 9. Breakpoint、Watch、Pause、Step

current `DebugBreakpointV1`はID／version、enabled、optional Session scope、kind、target selector、registered pure predicate ref、hit policy、action、safe-boundary policy、capture channel、owner、expiryを持つ。kindはRuntime time、System／Definition entry／exit、command／event、state predicate、Diagnostic、budget threshold、Asset generation、Level transition、render capture triggerのclosed registered IDである。ただし`Level transition`は§4の四typed transitionを区別できないため、新しいtransitionのQualification、commit Evidence、AI validated causeに使用しない。target `DebugBreakpointV2`は`Level transition`を削除してtyped transition kindを持ち、selectorは`runtime_entry_transition | stage_transition | world_space_transition | ui_screen_navigation`とphaseをexact指定する。Scene名、screen labelをaliasとして受理しない。V1／V2とも任意C++／Script式、filesystem、network、clock、random、World mutationをpredicateにしない。

hit policyは`first | every | after_count | every_n | once_per_target_generation | rate_limited`とし、hit／suppressed countとfirst／last hitを記録する。actionは`mark | capture | pause_at_safe_boundary | stop_recording | fail_qualification`である。

`DebugWatchV1`はStable target、MCD field path、sample boundary ref、interval、history capacity、comparison predicate、privacy classを持つ。pointer chain、native offset、任意expressionを使わない。target generation変更でretiredとし、新objectへ自動追従しない。containerはcount、hash、bounded sliceだけを返す。

```text
DebugSimulationAdvanceRequestV1
  request_id: StableId
  request_version: 1
  debug_session_ref: DebugSessionDescriptorRefV1
  cadence_profile_ref: SimulationCadenceProfileRefV1
  expected_next_advance_sequence: positive uint64
  control:
    kind: fixed
    | kind: variable
      requested_logical_duration_seconds: ReducedPositiveRationalV1
    | kind: turn_based
      advance_command_ref: SimulationAdvanceControlRefV1
      advance_command_sha256: SHA-256
    | kind: explicit_step
      step_request_ref: SimulationAdvanceControlRefV1
      step_request_sha256: SHA-256
      requested_step_ordinal: positive uint16
  request_content_hash: SHA-256
```

`request_content_hash`はASCII `MIRAKAN_DEBUG_SIMULATION_ADVANCE_REQUEST_V1`と同Fieldだけを除くclosed requestのMCD canonical bytesを`uint32_be` length framingして計算する。Debuggerはこのtyped requestだけを提出し、`SimulationAdvanceIntervalV1`、そのhash、accepted Command／request Field、advance sequenceを生成しない。Scheduling Ownerだけがcurrent Debug Session authorization／safe boundary、selected Profile ref、expected next sequence、branch equalityを検証する。fixedはProfileからdurationを導出し、variableはrequested durationがProfileのmin／max内かつ当該Profile／Targetのdebug-step Qualificationがfreshな場合だけ採用する。turn-basedはCommand ref／hashの`control_type_ref`がProfileの`advance_command_type_ref`へ解決され未消費であること、explicit-stepはrequest ref／hashの`control_type_ref`がProfileの`step_request_type_ref`へ解決され、requested count、ordinal、未消費状態がvalidであることを検証する。成功時だけScheduling Ownerが次sequenceの`SimulationAdvanceIntervalV1`を生成・sealし、turn／explicit branchのRef／hash／ordinalをrequestからbyte equalityで投影してDebuggerへ返す。失敗時はintervalとWorld stateを生成しない。

`StepSimulationAdvance`はScheduling Ownerが返したsealed `SimulationAdvanceIntervalV1`一件だけを入力に、選択CadenceでT00～T110のcomplete advanceを最後まで実行してsafe boundaryへ戻る。fixed／variableはrecord内のexact rational durationを使い、turn-based／explicit-stepは登録済みCommand／requestがScheduling Ownerに受理済みの場合だけ進め、待機中にDebug側が疑似advanceまたは`1/60`秒を生成しない。`StepRenderFrame`はWorldを固定しPresentationだけを進める。GameplayDefinition node stepはcopied state／discard-only journalでだけ許可する。live phase途中step、live GPU event step、Engineによるsource-line stepを提供しない。

## 10. Deterministic capture、Replay、Rewind

authoritative Replayのsemantic projection、Saveとの関係、digest、transport／Domain bundle、`ReplaySliceV1`は[Persistence／Save](persistence-save.md#5-replay-projection)が所有する。Debuggingはそのimmutable refをcapture・store・query・redact・表示するtransport ownerであり、Save header、Replay record、Domain projection、bundle、range artifact closureのSchemaを再定義しない。

captureは同じDebug Session、Project／Build／Target、Contract set、sealed World publicationへ束縛する。`RuntimeReplayTransportBindingV1`が欠落するcheckpoint、input、accepted async result、RNG、asset version、redactionをexplicitに示す。OS raw packet、pointer、GPU output、wall time、Presentation cacheをauthoritative inputにしない。

Rewindはrecord済みSnapshot／Event／State sampleをtimelineで閲覧する機能であり、live Worldを過去へ戻して継続する機能ではない。過去から再実行する場合はPersistence Ownerが検証したcheckpointからchild Replay Sessionを開始する。

divergence reportはfirst mismatch Runtime time、System／State Type／Field ID、expected／actual canonical digest、Input／RNG／accepted async差、Build／Contract／Asset差、preceding causal frontier、record gapを含む。field valueを記録していない場合はhash差までとし、値を推測しない。
## 11. Domain Debug Projection

各Domainは共通Envelope、Target、Time、Queryを使い、自身のbounded snapshot schemaを所有する。

| Domain | required projection family |
|---|---|
| [Gameplay](../03-authoring/gameplay-programming-model.md) | state owner、entrypoint、command／event、delta、task、variant、budget ref |
| [World／Scene／Space](../06-rendering/world.md) | active World／Scene／Space ref、activation closure、Cell ref、Topology／transition、last-valid state |
| [Physics](../05-simulation/physics.md)／[Collision](../05-simulation/collision.md) | body／shape ref、contact、sleep、constraint、query evidence |
| [Navigation](../05-simulation/navigation.md) | artifact／tile ref、agent、path、stale／failed result |
| [Animation](../05-simulation/animation.md) | graph state、transition、root motion、pose generation、event、LOD ref |
| [Rendering／Render Graph](../06-rendering/render-graph.md) | pass／resource ref、barrier、visibility、draw／dispatch、fallback |
| [LOD](../06-rendering/lod.md) | Plan／Context hash、candidate／selected tier、integer metric／threshold、pressure class、transition、fallback、completeness／gap |
| [Virtualized／Continuous Geometry](../06-rendering/virtualized-continuous-geometry.md) | Plan／Artifact generation、feature tuple、root state、resident／pending／failed aggregate、pool generation、View cut count／maximum error、fallback、capacity／thrash、completeness／gap |
| [Asset](../03-authoring/asset-lifecycle.md) | Source／Derived／Package version、dependency、residency、activation |
| [Input](../07-platform/input.md)／[UI／Text／Localization／Accessibility](../07-platform/ui-text-localization-accessibility.md)／[Audio](../07-platform/audio.md) | action／focus／route／voice／device、bounded callback evidence |
| [VFX runtime](../06-rendering/vfx-runtime.md)／[Environment／surfaces](../06-rendering/environment-surfaces.md)／[Camera](../06-rendering/camera.md) | artifact、seed、bounds、profile、fallback、owner-specific overlay |

Panel表示名をidentityにせず、typed targetからSource、Requirement、Decision、Build、Testへnavigationできるようにする。Domain固有field、Backend value、Qualificationは上表で直接Linkした各canonical Ownerへ委譲する。

Virtual geometry projectionは`VirtualGeometryResidencySnapshotV1`と`VirtualGeometryViewCutSummaryV1`のbounded envelope、aggregate count／digest、最大256件のsample、omitted count、truncated、stale／gapだけを受理する。raw page payload、全page list、GPU address、descriptor index、native command、unbounded cluster dumpをStoreまたはAI既定contextへ入れない。ProjectionはPresentation evidenceであり、Save／authoritative Replay input、World activation、LOD selectionまたはpage requestへのwrite-backを禁止する。Capabilityが`planning_only`のcurrent Binding集合は`[]`である。

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
| 3b | `operation.debug.read_replay_slice` | 同じidentity、exact Build／Query／Causality Receipt ref／hash、Persistence Ownerの`RuntimeReplayBundleManifestRefV1`（transport／root Projection／Domain Binding closure）、連続range、新しいAuthorization | `ReplaySliceReceiptV1`とreproduction／divergence evidenceを得る |
| 4 | `operation.debug.validate_finding` | 同じidentity、exact Build／Query／Causality／Replay Receipt ref／hash、`DebugFindingV1` hash、Finding closure hash、新しいAuthorization | `DebugFindingValidationReceiptV1`がvalidityとexact proposal Operation refを返す |
| 5 | Receiptの`proposal_operation_ref` | validation Receipt、同じrevision／Candidate、Caller allowlist、R1以上の新Authorization | 対応familyも独立にatomic Activation済みの場合だけ、例としてGame Systemは`operation.systems.plan`、Worldは`operation.worlds.plan_change`でProposalを生成 |

Activation時のDebug Operation型固有payloadは次だけを持つ。request、Authorization、result、Diagnostic、署名を含む共通Envelope fieldは[Core architecture §9.1](../02-foundation/core-architecture.md#91-operationtaskv1)のplanned `OperationReceiptEnvelopeV1`が所有する。以下のDebug evidence identity fieldはEnvelopeの共通identityを置換するものではなく、解決したSession／Build／Project／Targetと後段artifactをbyte equalityで束縛する型付きpayload projectionである。

Build familyのActivationはSystems／World familyを暗黙activateしない。Step 4が妥当なFindingを得ても、対応する`planning.operation_family.game_system_discovery`または`planning.operation_family.world_discovery`が未Activationなら`outcome={kind:succeeded, decision:{kind:insufficient}}`と`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`を返してProjectを不変にする。`proposal_operation_ref` Field、文字列ID、別familyのread-only候補、Provider aliasへfallbackしない。

```text
DebugOperationReceiptRefV1
  receipt_kind:
    aggregate | query | causality | replay_slice |
    finding_validation | support_bundle
  operation_ref: McdContractRefV1(kind=operation)
  task_id: UUIDv7
  signed_record_ref:
    MirakanSignedRecordRefV1(purpose=debug_operation_receipt)

DebugEvidenceRefV1
  debug_session_ref: DebugSessionDescriptorRefV1
  record_type_ref: McdContractRefV1(kind=type)
  store_generation: positive uint64
  record_sequence: positive uint64
  record_content_hash: SHA-256
  evidence_ref_content_hash: SHA-256

DebugAggregateReceiptPayloadV1
  debug_session_ref: DebugSessionDescriptorRefV1
  game_candidate_build_receipt_ref: GameCandidateBuildReceiptRefV1
  game_candidate_build_receipt_sha256: SHA-256
  project_id: UUIDv7
  project_revision: positive uint64
  project_document_set_hash: SHA-256
  target_profile_ref: TargetProfileRefV1
  store_generation: positive uint64
  index_generation: positive uint64
  selector_sha256: SHA-256
  outcome:
    kind: succeeded
      aggregate_result_ref:
        ArtifactRefV1(artifact_kind=debug_aggregate_result,
                      schema_version=1)
      aggregate_result_sha256: SHA-256
    | kind: failed
      diagnostic_refs[1..64]: DiagnosticCodeRefV1
    | kind: cancelled
      diagnostic_refs[1..64]: DiagnosticCodeRefV1

DebugQueryReceiptPayloadV1
  debug_session_ref: DebugSessionDescriptorRefV1
  game_candidate_build_receipt_ref: GameCandidateBuildReceiptRefV1
  game_candidate_build_receipt_sha256: SHA-256
  project_id: UUIDv7
  project_revision: positive uint64
  project_document_set_hash: SHA-256
  target_profile_ref: TargetProfileRefV1
  debug_aggregate_receipt_ref:
    DebugOperationReceiptRefV1(receipt_kind=aggregate)
  debug_aggregate_receipt_sha256: SHA-256
  store_generation: positive uint64
  index_generation: positive uint64
  query_sha256: SHA-256
  outcome:
    kind: succeeded
      record_slice_ref:
        ArtifactRefV1(artifact_kind=debug_record_slice,
                      schema_version=1)
      record_slice_sha256: SHA-256
      gap_summary_ref:
        ArtifactRefV1(artifact_kind=debug_gap_summary,
                      schema_version=1)
      gap_summary_sha256: SHA-256
      redaction_manifest_ref:
        ArtifactRefV1(artifact_kind=debug_redaction_manifest,
                      schema_version=1)
      redaction_manifest_sha256: SHA-256
    | kind: failed
      diagnostic_refs[1..64]: DiagnosticCodeRefV1
    | kind: cancelled
      diagnostic_refs[1..64]: DiagnosticCodeRefV1

DebugCausalityReceiptPayloadV1
  debug_session_ref: DebugSessionDescriptorRefV1
  game_candidate_build_receipt_ref: GameCandidateBuildReceiptRefV1
  game_candidate_build_receipt_sha256: SHA-256
  project_id: UUIDv7
  project_revision: positive uint64
  project_document_set_hash: SHA-256
  target_profile_ref: TargetProfileRefV1
  debug_query_receipt_ref:
    DebugOperationReceiptRefV1(receipt_kind=query)
  debug_query_receipt_sha256: SHA-256
  index_generation: positive uint64
  root_evidence_refs[1..16]: DebugEvidenceRefV1
  bounds_sha256: SHA-256
  outcome:
    kind: succeeded
      causal_graph_ref:
        ArtifactRefV1(artifact_kind=debug_causality_graph,
                      schema_version=1)
      causal_graph_sha256: SHA-256
    | kind: failed
      diagnostic_refs[1..64]: DiagnosticCodeRefV1
    | kind: cancelled
      diagnostic_refs[1..64]: DiagnosticCodeRefV1

ReplaySliceReceiptPayloadV1
  debug_session_ref: DebugSessionDescriptorRefV1
  game_candidate_build_receipt_ref: GameCandidateBuildReceiptRefV1
  game_candidate_build_receipt_sha256: SHA-256
  project_id: UUIDv7
  project_revision: positive uint64
  project_document_set_hash: SHA-256
  target_profile_ref: TargetProfileRefV1
  debug_query_receipt_ref:
    DebugOperationReceiptRefV1(receipt_kind=query)
  debug_query_receipt_sha256: SHA-256
  debug_causality_receipt_ref:
    DebugOperationReceiptRefV1(receipt_kind=causality)
  debug_causality_receipt_sha256: SHA-256
  replay_bundle_manifest_ref: RuntimeReplayBundleManifestRefV1
  replay_bundle_manifest_sha256: SHA-256
  replay_transport_binding_ref: RuntimeReplayTransportBindingRefV1
  replay_transport_binding_sha256: SHA-256
  replay_projection_ref: RuntimeReplayProjectionRefV1
  replay_projection_sha256: SHA-256
  start_advance_sequence: positive uint64
  end_advance_sequence_inclusive: positive uint64
  slice_content_hash: SHA-256
  outcome:
    kind: succeeded
      replay_slice_artifact_ref:
        ArtifactRefV1(artifact_kind=replay_slice, schema_version=1)
      replay_slice_sha256: SHA-256
    | kind: failed
      diagnostic_refs[1..64]: DiagnosticCodeRefV1
    | kind: cancelled
      diagnostic_refs[1..64]: DiagnosticCodeRefV1

DebugFindingValidationReceiptPayloadV1
  debug_session_ref: DebugSessionDescriptorRefV1
  game_candidate_build_receipt_ref: GameCandidateBuildReceiptRefV1
  game_candidate_build_receipt_sha256: SHA-256
  project_id: UUIDv7
  project_revision: positive uint64
  project_document_set_hash: SHA-256
  target_profile_ref: TargetProfileRefV1
  debug_query_receipt_ref:
    DebugOperationReceiptRefV1(receipt_kind=query)
  debug_query_receipt_sha256: SHA-256
  debug_causality_receipt_ref:
    DebugOperationReceiptRefV1(receipt_kind=causality)
  debug_causality_receipt_sha256: SHA-256
  replay_slice_receipt_ref:
    DebugOperationReceiptRefV1(receipt_kind=replay_slice)
  replay_slice_receipt_sha256: SHA-256
  finding_ref:
    ArtifactRefV1(artifact_kind=debug_finding, schema_version=1)
  finding_sha256: SHA-256
  finding_closure_ref:
    ArtifactRefV1(artifact_kind=debug_finding_closure,
                  schema_version=1)
  finding_closure_sha256: SHA-256
  outcome:
    kind: succeeded
      decision:
        kind: valid
          proposal_operation_ref:
            McdContractRefV1(kind=operation,status=active)
        | kind: invalid
        | kind: insufficient
    | kind: failed
      diagnostic_refs[1..64]: DiagnosticCodeRefV1
    | kind: cancelled
      diagnostic_refs[1..64]: DiagnosticCodeRefV1

SupportBundleReceiptPayloadV1
  debug_session_ref: DebugSessionDescriptorRefV1
  debug_session_closure_ref: DebugSessionClosureRefV1
  debug_session_closure_sha256: SHA-256
  game_candidate_build_receipt_ref: GameCandidateBuildReceiptRefV1
  game_candidate_build_receipt_sha256: SHA-256
  project_id: UUIDv7
  project_revision: positive uint64
  project_document_set_hash: SHA-256
  target_profile_ref: TargetProfileRefV1
  source_debug_receipts[1..64]:
    receipt_ref: DebugOperationReceiptRefV1
    receipt_sha256: SHA-256
  policy_ref: SupportBundlePolicyRefV1
  consent_record_ref:
    MirakanSignedRecordRefV1(purpose=support_bundle_consent)
  redaction_manifest_ref:
    ArtifactRefV1(artifact_kind=support_bundle_redaction_manifest,
                  schema_version=1)
  redaction_manifest_sha256: SHA-256
  outcome:
    kind: succeeded
      support_bundle_ref:
        ArtifactRefV1(artifact_kind=support_bundle, schema_version=1)
      support_bundle_sha256: SHA-256
      content_manifest_ref:
        ArtifactRefV1(artifact_kind=support_bundle_content_manifest,
                      schema_version=1)
      content_manifest_sha256: SHA-256
      archive_artifact_ref:
        ArtifactRefV1(artifact_kind=support_bundle_archive,
                      schema_version=1)
      archive_sha256: SHA-256
    | kind: failed
      diagnostic_refs[1..64]: DiagnosticCodeRefV1
    | kind: cancelled
      diagnostic_refs[1..64]: DiagnosticCodeRefV1
```

全objectはclosed、全Fieldは必須、unknown Fieldは禁止する。`DebugOperationReceiptRefV1.receipt_kind`は`aggregate→operation.debug.aggregate`、`query→operation.debug.query`、`causality→operation.debug.read_causality`、`replay_slice→operation.debug.read_replay_slice`、`finding_validation→operation.debug.validate_finding`、`support_bundle→operation.debug.support-bundle.generate`へ一意に対応し、`signed_record_ref`は同Operation、Task、Payload Typeのcompleted signed `OperationReceiptEnvelopeV1`へ解決する。各隣接`*_receipt_sha256`は`MirakanSignedRecordRefV1.signed_record_hash`および同じ完成Record全bytesのSHA-256とbyte equalityにする。`DebugEvidenceRefV1.evidence_ref_content_hash`はASCII `MIRAKAN_DEBUG_EVIDENCE_REF_V1`と同Fieldだけを除くclosed Ref canonical bytesから計算し、Session／Store generation／record sequence／Type／content hashの一Fieldでも異なるrecordへfallbackしない。

全PayloadのDebug Session Ref、Game Candidate Build Receipt Ref／completed signed SHA、exact Project triple `{project_id, project_revision, project_document_set_hash}`、Target Profile RefはDescriptor、Envelope、前段Receiptおよび生成Artifactとbyte equalityでなければならない。`store_generation`と`index_generation`は1開始で、同じReceipt chain内で減少させず、Query／Causalityが参照する前段のexact generationと一致させる。`root_evidence_refs[]`は1～16件、`source_debug_receipts[]`は1～64件で、各型のcanonical ref bytes順、duplicateなしとする。全Artifact Refの隣接SHAは`ArtifactRefV1.sha256`および解決したcompleted artifact bytesと一致させる。

Payload `outcome.kind`はEnvelopeの`result`とbyte equalityである。`succeeded` branchだけが当該success output groupを全Field必須で持ち、`failed | cancelled` branchはsuccess Fieldを一つも持たず、Envelopeと同一の`DiagnosticCodeRefV1[1..64]`を持つ。他branch Field混在、`?`によるpartial success、empty Diagnosticを拒否する。Replay successではPayloadのReplay Bundle ref／hash、transport binding ref／hash、Replay projection ref／hash、start／end、Slice content hashが解決した[Persistence／Save](persistence-save.md#51-replay-transport-binding)の`RuntimeReplayBundleManifestV1`と`ReplaySliceV1`へbyte equalityでなければならない。Findingの`proposal_operation_ref`は`succeeded.valid`だけで必須、`invalid | insufficient | failed | cancelled`ではField自体を禁止する。

`source_debug_receipts[]`は`DebugAggregateReceiptV1 | DebugQueryReceiptV1 | DebugCausalityReceiptV1 | ReplaySliceReceiptV1 | DebugFindingValidationReceiptV1`の`result=succeeded`完成Recordだけを受理する。各前段refは署名を含む完成`OperationReceiptEnvelopeV1` Record、その`*_sha256`は同じRecord hashでなければならない。全段のexact Project triple、Candidate root、typed Target、typed Debug Session、typed Game Candidate Build Receipt／completed signed SHA、remote Device identity／generationを一致させる。Aggregate→Query→Causality→Replay Slice→Finding validationの順を短絡せず、前段のmissing、非success、hash／署名／operation ID／payload contract差、revocation、Store／Index generation差を後段で拒否する。

Finding validationにはCausalityとReplay Sliceの両Receiptを必須にする。Replayまたは必要Evidenceを生成できない場合は前段Diagnosticから新しいvalidation Taskを作り、`outcome={kind:succeeded, decision:{kind:insufficient}}`以外を返さない。`proposal_operation_ref`は`decision.kind=valid`かつMCD登録済みのR1 proposal Operationが一意に解決した場合だけ必須とし、汎用`operation.debug.propose`、Source write、Commitを生成しない。Callerの`AiDebugContextV1.allowed_operation_ids`、新しいAuthorization Envelope、全Receipt ref／hashが一致しなければStep 5へ進まない。

Activation後、追加instrumentationはchannel、tier、duration、capacity／privacy影響を提示し、Governance authorizationを得て開始する。同じblocking集合が減らない自動repairは2回で停止する。各Operationは[Core architecture](../02-foundation/core-architecture.md#91-operationtaskv1)の別`OperationTaskV1`であり、前段のread権限、Device binding、consentを後段へ継承しない。状態確認、Receipt取得、cancelは同時Activationした`operation.task.status`、`operation.task.read_receipt`、`operation.task.cancel`だけを使う。

AIはmissing eventをnon-occurrenceと断定せず、PresentationからGameplay causeを推定せず、異なるgenerationを同一objectとして連結せず、Debug BuildをShippingへ一般化せず、redacted valueをdefault／emptyとみなさず、recorded Buildとcurrent Sourceを混同しない。

## 14. Reproduction、Crash、Hang、remote device

`ReproductionBundleV1`はbundle ID／version、issue key、source Session、Project revision、Build／Target ref、optional Replay Slice、required Artifact／Diagnostic／Capture ref、expected oracle、run instruction ref、privacy／license／redaction manifest、expiry、content manifest hash、optional signatureを持つ。Project全体を無条件に複製せず、secret、credential、signing material、private clipboard、prompt、personal dataを含めない。Import時はhash、schema、Build availability、license、privacy、path、size、signatureを検証し隔離Workspaceで開く。

support bundleのschema、redaction、size bound、生成operation、failureは本書だけが所有する。`SupportBundleV1`は[Product Plan](../00-product/product-plan.md)のdiagnosis→support製品E2E終端を成すUser提出用bundleであり、開発内再現用の`ReproductionBundleV1`とは別概念で相互に代用しない。

```text
SupportBundlePolicyRefV1
  policy_id: StableId
  policy_version: positive uint32
  policy_content_hash: SHA-256

SupportBundleDataClassRefV1
  privacy_profile_ref:
    ArtifactRefV1(artifact_kind=privacy_profile, schema_version=1)
  data_class_id: StableId
  data_class_version: positive uint32
  data_class_content_hash: SHA-256

SupportBundleV1
  bundle_id: StableId
  schema_version: uint32 = 1
  debug_session_ref: DebugSessionDescriptorRefV1
  debug_session_closure_ref: DebugSessionClosureRefV1
  debug_session_closure_sha256: SHA-256
  game_candidate_build_receipt_ref: GameCandidateBuildReceiptRefV1
  game_candidate_build_receipt_sha256: SHA-256
  project_id: UUIDv7
  project_revision: positive uint64
  project_document_set_hash: SHA-256
  target_profile_ref: TargetProfileRefV1
  component_artifact_refs[1..64]: SupportBundleComponentRefV1
  redaction_manifest_ref:
    ArtifactRefV1(artifact_kind=support_bundle_redaction_manifest,
                  schema_version=1)
  redaction_manifest_sha256: SHA-256
  consent_record_ref:
    MirakanSignedRecordRefV1(purpose=support_bundle_consent)
  policy_ref: SupportBundlePolicyRefV1
  uncompressed_size_bytes: uint64
  archive_size_bytes: uint64
  content_manifest_ref:
    ArtifactRefV1(artifact_kind=support_bundle_content_manifest,
                  schema_version=1)
  content_manifest_sha256: SHA-256
  archive_artifact_ref:
    ArtifactRefV1(artifact_kind=support_bundle_archive,
                  schema_version=1)
  archive_sha256: SHA-256
  generated_by_operation_ref:
    McdContractRefV1(
      kind=operation,
      operation_id=operation.debug.support-bundle.generate)
  bundle_content_hash: SHA-256

SupportBundleComponentRefV1
  component_kind:
    crash_evidence | hang_evidence | diagnostic_slice | log_slice |
    capability_summary | environment_summary
  artifact_ref:
    ArtifactRefV1(artifact_kind=support_bundle_component,
                  schema_version=1)
  content_sha256: SHA-256
  uncompressed_size_bytes: uint64
  data_class_refs[1..32]: SupportBundleDataClassRefV1
  component_ref_content_hash: SHA-256

SupportBundlePolicyV1
  policy_id: StableId
  policy_version: positive uint32
  privacy_profile_ref:
    ArtifactRefV1(artifact_kind=privacy_profile, schema_version=1)
  max_input_bytes: uint64[1..18446744073709551615]
  max_archive_bytes: uint64[1..18446744073709551615]
  max_file_count: uint32[1..65536]
  allowed_data_class_refs[1..64]: SupportBundleDataClassRefV1
  retention_policy_ref: McdContractRefV1(kind=policy)
  policy_content_hash: SHA-256

SupportBundleRedactionManifestV1
  policy_ref: SupportBundlePolicyRefV1
  input_component_refs[1..64]: SupportBundleComponentRefV1
  redaction_entries[1..4096]:
    component_ref_content_hash: SHA-256
    field_path: CanonicalJsonPointer
    action: included | removed | transformed
    data_class_ref: SupportBundleDataClassRefV1
    rule_ref: McdContractRefV1(kind=policy)
    output_value_sha256: null | SHA-256
  output_component_refs[1..64]: SupportBundleComponentRefV1
  omitted_count: uint64
  gap_summary_ref:
    ArtifactRefV1(artifact_kind=debug_gap_summary, schema_version=1)
  gap_summary_sha256: SHA-256
  manifest_content_hash: SHA-256
```

各`SupportBundle*RefV1`のcontent hashは型名に対応するASCII domain `MIRAKAN_SUPPORT_BUNDLE_POLICY_V1`、`MIRAKAN_SUPPORT_BUNDLE_COMPONENT_REF_V1`、`MIRAKAN_SUPPORT_BUNDLE_V1`と各自己hash Fieldだけを除くclosed canonical bytesから計算する。Policy Refは完成Policyから、Component Refは完成component referenceからrecord外でmaterializeする。Data Class RefはPolicyの同じPrivacy Profile memberへexact解決し、allowed集合外、別Profile、ID-only、latestへfallbackしない。Component／Policy／Data Class配列は各Ref canonical bytes順へstrict sortし、duplicateを拒否する。全Artifact Refの`sha256`と隣接SHAは同じcompleted bytesに一致し、BundleのComponent集合、Redaction Manifestのoutput集合、Content Manifest、Archiveのentry集合はexact set equalityである。`bundle_content_hash`は外部Operation Receipt／署名を入力にせず、completed receipt-free Bundleから計算する。

`SupportBundleRedactionManifestV1`は入力／出力component、Field／recordごとの`included | removed | transformed`、data class、rule、出力hash、omitted count、gap summaryを上記bound内で閉じる。`included | transformed`は`output_value_sha256`を必須、`removed`はnullにし、他branch Field混在を拒否する。credential、token、private key、password、signing materialは変換せず収集段階で拒否する。redaction後bytesからcomponent／manifest hashとsizeを再計算し、入力hash、出力hash、bundle manifestが一致しなければexportしない。`manifest_content_hash`はASCII `MIRAKAN_SUPPORT_BUNDLE_REDACTION_MANIFEST_V1`と同Fieldだけを除くclosed bytesから計算する。

`SupportBundleReceiptV1`は`OperationReceiptEnvelopeV1<SupportBundleReceiptPayloadV1>`の完成署名Recordである。BundleとReceiptのCandidate、typed Target、typed Debug Session Descriptor／Closure、typed Game Candidate Build Receipt／completed signed SHA、exact Project triple、source Debug Receipt、Policy、consent、redaction manifest、content manifest、archive hashが一致しなければ成功にしない。ClosureはDescriptorと同一Sessionのcompleted hash-valid Recordで、Bundleのcompleteness／gap表示はClosureからだけ取得する。

Atomic activation後の生成は`operation.debug.support-bundle.generate`だけが行い、対象Session、component Preview、data class、概算／上限bytes、redaction policy、提出先を表示し、[AI Security／Approval §3.3](../01-governance/ai-security-approval.md#33-consent-recordとpurpose-binding)のexact `support_bundle_consent` purposeへ明示consentを得る。別のcrash upload、telemetry、AI Provider、network consent、Settings boolを継承しない。これは上記と同じOperation Registry、`OperationTaskV1`、task status／Receipt／cancel経路を使うexport branchであり、独自Task APIまたは自由形式Toolにしない。Aggregate／Query Receiptを入力component選択に使えるが、Support Bundle生成をFinding validationまたはProposal成功として扱わない。`max_input_bytes`、`max_archive_bytes`、`max_file_count`のいずれかを超える場合は切り詰めて成功扱いせず、対象rangeを狭める新Proposalを返す。最低failureは`diagnostic.debug.support-bundle-consent-required`、`diagnostic.debug.support-bundle-redaction-incomplete`、`diagnostic.debug.support-bundle-size-limit-exceeded`、`diagnostic.debug.support-bundle-artifact-unavailable`、`diagnostic.debug.support-bundle-manifest-mismatch`をclosed IDとして区別する。

Target別の生成UX、保存先、提出transportは各Platform Owner（[Windows](../07-platform/windows.md)、[Mobile Common](../07-platform/mobile-common.md)）が本schemaを投影する。Platform文書は収集可能componentとnative share UIだけを定義し、独自Support Bundle schema、緩いredaction、別size cap、silent uploadを作らない。

Crash recordはPlatform OwnerのCrash envelopeを参照し、Session、Build、last durable sequence、gap、Replay checkpoint、breadcrumbを関連付ける。in-process handlerはpreallocated metadataだけを書き、dump／symbol／Sourceを別Artifactにする。exact binary／module／symbol hashが一致する場合だけsymbolicateし、partial stackを推測補完しない。User actionとGovernance authorizationなしにonline service／Providerへ送らない。

Hang watchdogはsimulation、render submission、window、audio control、worker poolのheartbeatを分け、last progress、active exact `RuntimeTimeRefV1`、bounded role stack ref、queue depth／oldest item、lock-order state、GPU submission status、last critical Diagnosticを記録する。threadを強制resumeして継続を成功扱いしない。

`HangDetectionPolicyV1`は`policy_id`、schema version、Target Profile ref、role entries、Application／debug execution state条件、evidence profile refを持つ。role entryは`role`、次のclosed `HangSimulationProgressExpectationV1`または非simulation role固有expectation、minimum no-progress duration、active predicate、exempt state setを必須とする。

```text
HangSimulationProgressExpectationV1
  cadence_profile_ref: SimulationCadenceProfileRefV1
  expectation:
    kind: fixed_periodic
      expected_logical_duration_seconds: ReducedPositiveRationalV1
      minimum_missed_advance_count: positive uint32
    | kind: variable_periodic
      minimum_logical_duration_seconds: ReducedPositiveRationalV1
      maximum_logical_duration_seconds: ReducedPositiveRationalV1
      minimum_missed_advance_count: positive uint32
    | kind: turn_command_driven
      accepted_advance_command_ref: SimulationAdvanceControlRefV1
      accepted_advance_command_sha256: SHA-256
      simulation_advance_interval_hash: SHA-256
    | kind: explicit_request_driven
      accepted_step_request_ref: SimulationAdvanceControlRefV1
      accepted_step_request_sha256: SHA-256
      request_step_ordinal: positive uint16
      simulation_advance_interval_hash: SHA-256
```

simulation roleは選択したProfileとbyte equalityのbranchだけを使う。fixed expectationはProfileの`rate_hz.denominator / rate_hz.numerator`を既約化した値、variableのmin／maxはProfileの両Fieldとbyte equalityにし、表示Hz、前回sample、wall-time観測から補完しない。turn／explicitはScheduling OwnerがsealしたIntervalをhashで解決し、そのInterval内の`SimulationAdvanceControlRefV1`、completed control SHA、explicit ordinalとbyte equalityにする。accepted command sequence ref、accepted request sequence ref、bare control ID、直前requestからのordinal推測を禁止する。Command／request未受理の待機stateはactive predicate=falseであり、周期intervalを捏造してhang判定しない。C1の値は次で固定する。

| role | hang成立条件 | 明示除外 |
|---|---|---|
| simulation fixed | Activeかつrunning中、進行なしが`max(120 × expected_logical_duration_seconds, 2 s)` | gameplay pauseではなくdebug `paused_at_t110`、Inactive、Suspended、Terminating |
| simulation variable | Activeかつrunning中、進行なしが`max(120 × maximum_logical_duration_seconds, 2 s)` | gameplay pauseではなくdebug `paused_at_t110`、Inactive、Suspended、Terminating |
| simulation turn-based | accepted advance CommandがpendingなのにT110 publish／fault acknowledgementなし2 s | advance Command待機中、debug `paused_at_t110`、Inactive、Suspended、Terminating |
| simulation explicit-step | accepted requestの未完step ordinalがあるのにT110 publish／fault acknowledgementなし2 s | step request待機中、debug `paused_at_t110`、Inactive、Suspended、Terminating |
| render submission | 有効surfaceで進行なしが`max(120 × expected render interval, 2 s)` | headless、SurfaceUnavailable、Inactive、Suspended、Terminating |
| window | visible windowでmessage dispatch／present acknowledgement進行なし5 s | hidden／minimized、Inactive、Suspended、Terminating |
| audio control | active audio sessionでcontrol sequence進行なし2 s | audio session停止、Suspended、Terminating |
| worker pool | runnable itemがあり、oldest runnable age 10 s以上かつrole progressなし10 s | queue empty、全itemが外部I/O待機として登録済み、Suspended、Terminating |

watchdogは判定に用いたCadence Profile Ref、exact expectation branch、fixed durationまたはvariable min／max、turn／explicitではControl Ref＋completed SHA＋sealed Interval hash、explicit ordinal、threshold、ApplicationState、debug execution mode、last progress sequenceをhang evidenceへ値で記録する。Profile content hashだけを別Fieldへ複写せずRef全体を保存し、accepted command／request sequence refを生成しない。該当除外に入っただけで直前のsuspected hangを成功へ変えず、`cleared_by_progress | terminated | evidence_partial`の終端を記録する。Platform watchdogがより短い期限を課す場合は早期capture triggerとして併記するが、本Policyのrole判定をsilentに置換しない。

Remote handshakeはDevice identity、pairing generation、App／Engine／Module hash、Target、Debug Capability、channel、bandwidth／storage capacity ref、clock correlation、privacy stateを持つ。Development／Profile Buildだけがshort-lived mutual-authenticated Sessionで接続できる。Device Bridgeはfilesystem、shell、process、network proxyを提供しない。

transferはcontrolとbulk captureを分け、Counter／Diagnosticを優先し、disconnect時のlocal ring overflowをgapにする。resumeはlast acknowledged chunk hashから行い、sequenceでdeduplicateする。remote mutationのWatch、Capture、Pause／Resume、Replay controlは§13のDebug familyがatomic Activationされ、current MCD／Owner Manifest／Service allowlistへ同時登録された後だけ許可する。現在のremote-mutation Operation集合はexact 0件で、候補名やcontrol名によるdispatchを拒否する。C++／shader／native plugin変更はActivation後もrebuild／reinstallを必要とする。

## 15. Security、privacy、retention、failure

OperationごとのRisk、authorization、human approval、Provider／MCP／CLI boundaryは[AI Security／Approval](../01-governance/ai-security-approval.md)をそのまま消費する。Debug read権限をSource write、Runtime mutation、export、remote attach、Capability activationへ昇格させない。

data classはGovernance／Project privacy profileのregistered classを参照し、credential、token、private key、password、signing materialは収集自体を拒否する。Providerへ送る前にcategory、bytes、target、retention、ProviderをPreviewする。retentionはTarget別Profile refとPerformance capacity reservationを持ち、expiry後はPlatform deleteを実行してGovernance deletion Evidence refだけを残す。

Shipping package scanはDebug listener、expression evaluator、runtime compiler、private symbol／source map、validation layer、shader print、AI credential、unrestricted console、development signing identity、unredacted capture presetを拒否する。fault／crash最小recordもconsentとPrivacy Profileへ従う。

pressure時は`critical | high | normal | verbose`のregistered priorityを使い、verbose、normalの順でdropする。critical recordを保存できない場合はcaptureをtruncatedとして停止し、Qualificationを失敗させる。Debug pressureだけでauthoritative stateを変更しない。gapにはchannel、dropped count、first／last lost sequence、reasonを記録する。

最低Diagnostic familyはinvalid session config、capability unavailable、Build mismatch、store limit、event drop、critical recorder failure、stale Index、broad Query、target generation mismatch、unsafe pause、Replay divergence／closure missing、uncorrelated clock、privacy approval required、redaction incomplete、AI Evidence insufficient、remote auth failure、external capture unverifiedをclosed codeで区別する。Operation workflowは少なくとも`diagnostic.debug.aggregate-input-invalid`、`diagnostic.debug.query-input-invalid`、`diagnostic.debug.causality-input-invalid`、`diagnostic.debug.replay-slice-input-invalid`、`diagnostic.debug.finding-evidence-invalid`をclosed codeにする。Logだけで通知しない。

## 16. Test、AI Eval、qualification

Contract fixtureは全Typeのvalid／invalid／boundary、canonical encoding／hash／crash recovery、pointer／native handle／unbounded field／secret拒否、target generation、event registry、counter unit、Query／cursor／Index stale、gap／redactionを検証する。Debug Sessionは最初のRuntime time前に`before_first_runtime_time` Descriptorを作るcase、completed Runtime timeへ相関した開始case、終了後に同じDescriptor Refを維持して外部Closureだけを一件作るcaseを受理し、開始前の偽Runtime time、Descriptorへのend／completeness／gap埋込み、終了時Descriptor更新、Closure二重発行、Descriptor／Closure Session差を拒否する。

Runtime fixtureは[Scheduling／lifetime](scheduling-lifetime.md)の全Runtime orderへのtime ref対応、parallel emitのcanonical sequence、priority drop、safe pause／complete Simulation Advance step／render step、sandbox node step、Store／Panel crash非干渉、callback hot path allocation／block 0を検証する。Debug stepでは四branchのRequest→Scheduling検証→sealed Interval投影を検査し、Debugger生成Interval、Profile／expected sequence差、variable範囲外、wrong Command／request type、Ref／hash差、消費済みcontrol、explicit ordinal 0／上限超過を各一原因で拒否してInterval／World state 0件変更を確認する。Hang fixtureはfixed duration、variable min／max、turn Control Ref＋completed SHA＋Interval hash、explicit request Ref＋completed SHA＋ordinal＋Interval hashを検証し、旧sequence ref、bare ID、Profile／branch差、待機中の偽periodic expectationを拒否する。instrumentation overhead、memory、disk、queue、soakは[Performance／capacity](performance-capacity.md)のcurrent Gateで測定する。

Transition observability fixtureはRuntime Entry、Stage、World spatial、UI Screen navigationの四kindについてrequested→preparing／ready→committedとrequested→rejected／cancelledを検証し、request ref、source／destination typed target、generation、T00 Runtime time、exact owner final resultをread-backする。`Level transition` event／breakpoint、display name／path target、commit前のdestination generation、Loading表示から合成したcommit、別request final result、source generation差を各一原因でrejectし、instrumentation gapがある場合はtransition失敗または成功を推測せず`partial`として表示する。

Runtime ECS qualification fixtureはmandatory metric欠落、NaN／infinite value、counter overflow、wrong Target、wrong Candidate、process reuse、missing campaign cell、Debug-to-Runtime authority back-edgeを一原因ずつ拒否する。mandatory metric欠落は`diagnostic.performance.ecs-required-metric-missing`へexact mappingし、他のinvalid sampleを0またはpassへ補正しない。

Replay fixtureはInput、RNG、accepted async resultから同じstate hashとfirst divergenceを得ること、recorded／current revision分離、closure／Asset／worker mismatch拒否、gapを含むSessionのpartial表示、child Session isolationを検証する。`RuntimeReplayProjectionV1`、`RuntimeReplayTransportBindingV1`、`RuntimeReplayBundleManifestV1`、`ReplaySliceV1`はtyped Debug Session、Build、Project triple、Target、Contract set、連続advance rangeへbyte equalityで解決する。transport bindingのcheckpoint／input／accepted async／RNG／asset version／redaction artifact集合はcanonical order、duplicateなし、range完全性を検証し、missing、extra、別Session／Build／Target／Contract set、hash差、range gap／overlap、redaction隠蔽、base projectionへのbinding埋戻し、別rootを束ねるBundleをReplay開始前に拒否する。Domain Replayはreceipt-free Projectionからroot外Bindingを生成し、Persistence OwnerだけがBundle Manifestへmembershipを閉じる。

Atomic activation acceptance fixtureはAggregate→Query→Causality→Replay→Finding validation→exact domain Proposalのtask／Receipt chainを検証する。QueryのAggregate Receipt、CausalityのQuery Receipt、ReplayのBuild／Query／Causality Receipt、Finding validationのBuild／Query／Causality／Replay Receiptについて、missing、typed receipt kind／Operation／Task差、completed SHA差、署名差、別operation payload、revocationを一原因ずつ注入して後段を停止する。全Receipt payloadのtyped Session／Build Ref／completed SHA／Project triple／Targetを前段、Descriptor、Persistence OwnerのReplay Bundle／Sliceから一Fieldずつ差し替えるfixtureを持つ。各Payload outcomeとEnvelope resultの差、success output欠落／他branch混入、failed／cancelledのempty／65件目Diagnostic、generation 0、root Evidence 0／17件、source Receipt 0／65件も拒否する。Replay ReceiptのReplay Bundle Ref／completed SHA、transport binding Ref／completed SHA、Replay projection Ref／completed SHA、range start／end、Slice content hash差も後段を停止する。stale Candidate、別Session／Build／Project document set／Target／Store／Index generation、remote Device交換、request hash／Authorization差でも拒否する。Evidence ref不在、別revision、gap／redaction隠蔽、時間相関だけ、reproductionなしの`validated_cause`を含む偽Findingは`operation.debug.validate_finding`で`diagnostic.debug.finding-evidence-invalid`となり、proposal Operation refを返さずProject stateを不変にする。Support Bundle branchは同じ署名Envelope／Task APIを使い、Descriptor／Closure差またはClosure欠落、source Debug Receipt差、consentなし、Policy／Data Class不一致、redaction action branch混在、component／manifest／archive set差、size／file／entry bound超過でexport byteを公開しない。current fixtureは14 candidateすべてについてdispatch前の`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`とProject／Task／export byte不変を検査する。

`fixture.debug.known-faults`は少なくともInput context conflict、Collision filter、stale Nav result、root-motion authority conflict、Asset generation mismatch、Render barrier diagnostic、Audio pressure、Gameplay bounded-execution fault、Runtime Entry／Stage closure不足、RNG divergence、GameHost crash／symbol mismatch、remote disconnect／gapを含む。各caseはobservation、typed Diagnostic、causal path、Replay Slice、correct remediation、forbidden remediation、regression fixtureを持つ。

AI Eval `debugging_diagnosis`はroot cause top-1 85%以上、top-3 95%以上、Blocking／High Evidence recall 100%、Evidenceなしvalidated cause 0、gap／redaction／revision／Presentation authority誤認0、unknown ID提出0、permission緩和0、2回超repair 0、unreproduced fixed claim 0、関連regression実行100%をC1 targetとする。Corpus／grader／3-run／Receipt構造は[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)だけが決定する。

Platform fixtureはCPU／GPU validation、crash／hang artifact、symbol hash、remote pairing／disconnect、redacted capture、minimum／reference device、Shipping package scanを各Platform Ownerのbaselineで実行する。外部captureだけをsuccess oracleにしない。

Debug qualificationはContract／Build／Target／instrumentation／retention／privacy／fixture／captureのexact refs、completeness／gap、Performance measurement ref、Contract／Runtime／Replay／Platform／AI Eval resultをGovernance Evidenceへ提出する。共通Receipt field、reviewer、approval、signature、freshnessは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を参照する。

ECS format migrationを伴うqualificationでは、observability Consumer Inventory record、全Evidence Requirementのpass satisfaction binding、Compatibility Change、Owner reference migration manifest、source／target Definition Closure、Definition Migration bindingのrefを同じevidence closureへ束縛する。old capture reader、unredacted export、別Contract setのReplay transportをsilent fallbackにしない。

## 17. Capability分解と受入closure

本節は実装順序、Task、担当またはProduct Phaseを定義しない。Debugging capabilityを独立に審査できるcontract subjectへ分解する。

| Subject | 正本範囲 | 受入条件 |
|---|---|---|
| Debug contract | target／time／Session／event／counter／Query／gap schema | exact identity、bounded Query、unknown／invalid input拒否 |
| Flight recorder | fixed ingress、priority、crash recovery、Runtime correlation | gap・lossの明示、capacity Receipt、failure recovery |
| Local authoring surface | safe pause／step、Overlay、external IDE／GPU evidence mapping | Runtime authorityを変更しないread-only projection |
| Replay／causality | Replay Slice、divergence、Causality、Watch history、Reproduction Bundle | deterministic replay、first divergence、minimal reproduction |
| AI diagnosis | bounded Context、Finding validation、privacy Preview、governed proposal | Evidence-backed Finding、permission不変、AI Eval合格 |
| Remote／Shipping boundary | authenticated Device Bridge、resume／gap、Platform Adapter、Shipping scan | Target binding、redaction、disconnect recovery、artifact scan |

統合受入にはexact Session identity、one-record multiple projection、gap／redaction／clock uncertainty、safe pause／step、deterministic Replay、minimal Reproduction Bundle、external evidence mapping、Evidence-backed AI Finding、governed repair、Performance capacity、Shipping artifact scan、known-fault fixtureを同じCandidate closureへ束縛する。各subjectの実装時期は本書の責務外である。

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
