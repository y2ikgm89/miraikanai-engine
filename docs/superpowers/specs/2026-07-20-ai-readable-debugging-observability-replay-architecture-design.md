# Miraikanai Engine AI可読Debugging／Observability／Replayアーキテクチャ規約

- 文書版: 1.0
- 作成日: 2026-07-20
- 最終更新日: 2026-07-20
- 対象: Engine／Game制作Debug、Observability、Profiler、Replay／Rewind、Crash／Hang、Remote Device、AI Diagnosis
- 状態: プロジェクト公式の規範設計レビュー版。契約と段階導入方針を確定し、実装とTarget別Qualificationは未完
- 上位文書: [AIネイティブ独自ゲームエンジン 設計計画書](./2026-07-18-ai-native-game-engine-authoring-design.md)
- 関連文書: [基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)、[実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)、[Game System／AI Code Generation規約](./2026-07-20-game-system-ai-codegen-architecture-design.md)、[Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)、[World／Level／Map／AI Authoring規約](./2026-07-20-world-level-map-ai-authoring-architecture-design.md)、[Editor／Workspace／UX規約](./2026-07-19-editor-workspace-ux-design.md)、[Windows Platform／Distribution規約](./2026-07-19-windows-platform-distribution-design.md)、[モバイルPlatform規約](./2026-07-19-mobile-platform-architecture-design.md)、[AI検証・評価・来歴規約](./2026-07-19-ai-verification-evaluation-provenance-design.md)

## 1. 結論

Miraikanai EngineのDebuggingは、自由文Logを人間またはAIへ大量に渡して原因を推測させる機能ではない。Project revision、Build、Play session、World、tick、phase、frame、System、Entity、Asset、Command、Event、State delta、Diagnosticを、UUIDv7 `StableId`、exact `McdContractRefV1`、`ArtifactRefV1`、session-scoped runtime object refで型付き相互参照できるEngine-owned Evidence Platformとする。

人間向けConsole、Problems、Profiler、Timeline、Breakpoint、Watch、Overlay、Replay／Rewind、Crash解析と、AI向けの限定Query、Causality Graph、仮説、反証、修正候補は同じ記録を読む。Editor画面、pixel、Widget tree、Native pointer、自由文だけを正本にしない。

AIはDebug dataを読んだだけでProjectまたはRuntimeを直接変更しない。AIの出力はEvidence参照を持つ`DebugFindingV1`と、必要なら既存の`RemediationV1`またはAuthoring Operation候補である。修正は必ず`AuthoringCommandGateway`、Risk、Approval、Staging、Test、Replay回帰を通る。

公式方式から次を採用する。

- UnityのProfiler marker／counter／recorder、Editor／Player／実機接続、Development buildとdebug symbolの分離、Frame／Physics Debuggerの限定表示。
- Unreal Engineのself-describing trace、Visual LoggerのObject snapshot／text／shape／timeline、Gameplay Debuggerの拡張category、Blueprint Debuggerのbreakpoint／watch／step、Rewind Debuggerのrecord／scrub。
- Godot Engineの統合Debugger Panel、stack／expression evaluator、Profiler／Monitor、Remote Scene Tree、Object snapshot差分、remote debug。
- WindowsのVisual Studio、DRED、PIX marker、WER／minidump、Vulkan Validation Layerと`VK_EXT_debug_utils`、Android Performance Analyzer／Perfetto／AGI、Apple Xcode／Metal／InstrumentsのPlatform診断。

ただしUnityのGameObject、UnrealのUObject、GodotのNodeまたは各Engine固有APIをMiraikanaiの正規Object modelへ複製しない。外部ToolのCapture形式も正本にせず、Engine-owned ID、Schema、Query、Receiptから必要なAdapterへexportする。

## 2. 決定権と対象外

### 2.1 本書が決定する

- Debug Session、Target、Channel、Capture、Retentionの正規契約。
- Log、Diagnostic、Span、Counter、State sample、World shape、Crash breadcrumbの共通Envelope。
- 人間、CLI、CI、AIが共有するbounded Query、Index、Causality Graph。
- Breakpoint、Watch、Pause、Step、Record、Replay、Rewindの意味と安全境界。
- Console、Problems、Timeline、Profiler、Watch、Replay、Reproduction、AI Diagnosis PanelのDomain data。
- Engine／Game／Project C++／GameplayDefinition／Subsystem Debugの統合境界。
- External debugger、GPU debugger、remote device bridge、symbol／source mappingとの接続契約。
- Debug dataのSecurity、Privacy、Redaction、Export、Retention、Shipping除外。
- Debugging CapabilityのC0～C3、Budget、Failure、Test、AI Eval、Qualification。

### 2.2 他文書が決定する

| 主題 | 正本 |
|---|---|
| `MirakanDiagnosticV1`、`RemediationV1`、MCD、Codegen | 実行可能契約規約 |
| Tick／Render phase、Command／Event／Snapshot、common trace context、Runtime memory | Runtime規約 |
| ProjectRevision、ChangeSet、Undo、Apply Back、Journal | Authoring Model規約 |
| Panel、Dock、Workspace、Accessibility、AI Partner shell | Editor規約／Editor UI Framework規約 |
| Domain固有Debug Snapshotの内容 | 各Subsystem正式仕様 |
| D3D12／Vulkan／Metal resource、pass、device loss | Rendering規約とPlatform規約 |
| PDB、package、Crash collection、signing | Windows規約 |
| Android／Apple lifecycle、Device Manager、Store privacy | モバイル規約 |
| AI Risk、Authorization、Provider／MCP／CLI | AI実装・保守ガバナンス規約 |
| AI Eval、Receipt、Corpus、Provenance | AI検証・評価・来歴規約 |

### 2.3 対象外

- 任意のRuntime memoryを読書きする汎用Memory Editor。
- Shipping Gameへ常設するremote shell、debug server、expression evaluator、compiler。
- C++ source debugger、GPU debugger、Profiler、Crash symbolicatorの全面自作。
- 任意C++式、任意Script、任意SQLをRuntime内で評価するDebug Console。
- Debugger停止中のworker、audio callback、GPU queue、Platform callbackを強制再開してGameを継続する機能。
- 記録していない過去状態を推測で復元する完全Time Travel Debugger。
- Multiplayer Debug。Networking C3正式仕様とThreat Modelが承認されるまでlocal／single-player sessionだけを対象にする。
- Shipping利用者の無断telemetry、無断Crash upload、AI Providerへの自動Debug data送信。

## 3. 外部公式方式と採用判断

調査基準日は2026-07-20である。外部資料は機能分解と運用上のEvidenceであり、Miraikanai固有のID、Schema、Phase、Budget、Risk、Approvalを外部Engineから決めない。

| 公式方式 | 確認した強み | Miraikanaiで採用する原則 | 採用しないもの |
|---|---|---|---|
| Unity 6.5 Profiler | marker、counter、custom module、remote Player、Profiler data抽出 | counterを登録制Schemaにし、Editor表示とAPI取得を同じ値へ収束 | Unity category名、GameObject、C# API |
| Unity Frame／Physics Debugger | 一Frameのevent step、対象filter、Scene内可視化 | Captureをimmutable event列としてstepし、Domain filterとWorld shapeを共通化 | live GPU内部状態をEngine stateとして扱うこと |
| Unity managed debugging | IDE attach、breakpoint、single step、variable inspection、Development build | C++ debuggerは外部IDEへ委譲し、Build／Process／symbol／source mappingをEngineが記述 | 独自Source debuggerの再発明 |
| Unreal Insights | `EventName`／`FieldName`を持つself-describing trace、channel、trace store、`.utrace` | version付きEvent Type、Field mask、Channel、Capture manifest、Index generation | `.utrace`をMiraikanai正本にすること |
| Unreal Visual Logger | Actor snapshot、category、text、debug shape、timeline scrub | Stable target snapshot、shape primitive、時間軸、記録後Review | Actor／UObject identity、macro API |
| Unreal Gameplay Debugger | runtime category、AI／Behavior Tree／Blackboard／EQS／Navmesh、必要categoryだけ送信 | Domain別category、bounded collection、remote転送のfield mask | replication方式を初期local Debugへ導入 |
| Unreal Blueprint Debugger | graph breakpoint、watch、call stack、step | GameplayDefinition node／entrypoint／state watch、safe-boundary step | live authoritative transaction途中の任意mutation |
| Unreal Rewind Debugger | record、object track、variable history、scrub、trace連携 | Replay Slice、Object／Component track、値の時系列比較 | 記録外状態の完全逆実行 |
| Godot Debugger Panel | Error、stack、Evaluator、Profiler、Monitor、VRAM、Network | Debug機能を一Workspaceへ統合し、read-only evaluatorを限定提供 | 任意Node直接mutation |
| Godot Remote Scene Tree | 実行中Tree／Propertyのinspection | Runtime Worldのtyped `DebugTargetRefV1` projectionとWatch | Runtime object pointerまたはEditor treeとの共有 |
| Godot ObjectDB Profiler | snapshot、snapshot間差分 | typed Object／Resource aggregate snapshotとDiff | Engine内部全Objectを公開すること |
| D3D12 DRED／PIX | breadcrumb、page fault、marker、GPU capture | Project revision、frame、pass、resource debug IDをPlatform markerへ関連付け | DRED／PIX形式を共通Schemaにすること |
| Vulkan Validation／Debug Utils | VUID、severity／type、object name、queue／command label | VUIDとBackend evidenceを`MirakanDiagnosticV1`へ正規化しtyped runtime resource refを併記 | Validation LayerのShipping同梱 |
| Android Performance Analyzer／Perfetto／AGI | APA System Profiler、Perfetto data source／trace、AGI frame profiling、system／app／GPU correlation | system traceはAPA／Perfetto Adapter、Vulkan frame深掘りはAGI Adapter、redacted Capture Receipt | ADBを製品共通APIまたはShipping bridgeにすること |
| Apple Xcode／Metal／Instruments | signpost、GPU capture、Metal validation、device trace | Platform marker Adapter、外部Capture参照、Target別Qualification | Apple Tool形式をProject formatへ保存 |

外部方式が提供する「表示機能」だけを模倣しない。対象の型付きidentity、capture completeness、drop、redaction、build hash、query cost、reproduction、failure、AI利用条件までMiraikanai契約で閉じる。

## 4. 設計原則

1. **Evidence first**
   人間またはAIの説明より、version付きEvent、Snapshot、Counter、Replay、Crash artifact、Build／Project hashを優先する。
2. **Meaning before pixels**
   AIはScreen capture、color、graph座標から原因を推測せず、Semantic Snapshotとtyped Queryを使う。
3. **One record, multiple projections**
   Editor、CLI、CI、AI、外部Adapterが別々のinstrumentationを持たず、同じcanonical recordを読む。
4. **Typed identity and time**
   表示名、pointer、登録順、OS thread IDだけで対象を識別せず、基盤規約のID／参照型とtick／phase／frameを持つ。
5. **Safe boundary**
   Pause、Watch、Snapshot、Apply BackはRuntime phaseとtransaction boundaryを破壊しない。
6. **Bounded by construction**
   Event size、rate、queue、query result、capture、retention、AI contextに上限を持たせる。
7. **Drop is data**
   記録欠落を隠さず、gap、dropped count、first／last lost sequence、原因を記録する。
8. **Read-only diagnosis, gated repair**
   Debug QueryはR0、Capture／Pause／Exportは影響に応じR1／R2、Project変更は既存RiskとApprovalへ戻す。
9. **Deterministic where authoritative**
   Input、RNG、accepted async result、state hashをReplayし、PresentationとPlatform timingの非決定性を区別する。
10. **No debug path in Shipping by accident**
    Debug server、compiler、symbol、source path、credential、unredacted traceをPackage inspectionで拒否する。
11. **External tools as adapters**
    IDE、PIX、RenderDoc、Perfetto、Instrumentsは再発明せず、Miraikanai IDとCapture Receiptで接続する。
12. **Debugging must not validate itself**
    Debuggerの表示だけを成功判定にせず、headless Query、Replay hash、fixture、external validationを併用する。

## 5. 論理アーキテクチャ

```text
Authoritative Runtime／Subsystem／Tool／Platform Adapter
        |
        | fixed-size event / counter / immutable snapshot
        v
Debug Ingress Ports
        |
        +--> Critical Flight Recorder
        +--> Trace Store
        +--> Counter Store
        +--> Replay Store
        +--> Capture Artifact Registry
        |
        v
Debug Index Service
  time / target / type / diagnostic / correlation / causality
        |
        +--> Editor Debug Workspace
        +--> Headless CLI / CI
        +--> External Tool Adapter
        +--> bounded AI Debug Context
                    |
                    v
               DebugFindingV1
                    |
          Authoring Proposal / Remediation
                    |
          Gateway / Approval / Staging
                    |
       same Replay + Test + Regression Receipt
```

### 5.1 Process境界

| Process | 所有するDebug責務 | 禁止 |
|---|---|---|
| EditorHost | Query、Index projection、Panel、AI context、local Capture管理 | Native Runtime pointerの直接inspection |
| GameHost | Runtime event生成、flight recorder、Replay記録、safe pause | Project Authoring write、Provider接続 |
| WorkerHost | Build／Import／Shader／Test event、artifact参照 | GameHost memory inspection |
| Device Bridge | 認証、Capability handshake、bounded stream、capture transfer | Shipping常設、任意command shell |
| Crash Collector | out-of-process dump、Crash Envelope、bounded breadcrumb | Project Save、network upload、AI呼出し |
| AI Orchestrator | read-only Query、Finding生成、Proposal handoff | Debug store raw file access、直接Runtime mutation |

EditorHostが停止してもGameHostのauthoritative stateを壊さない。GameHostがCrashしてもEditorHostはlast acknowledged record、Crash Envelope、Project revisionを保持する。Debug IndexまたはPanelのFailureをGame session成功として隠さず、recording completenessを`partial`にする。

### 5.2 Module境界

```text
mirakan_debug_contracts
  <- mirakan_debug_runtime
  <- mirakan_debug_store
  <- mirakan_debug_query
  <- mirakan_debug_replay
  <- mirakan_debug_editor
  <- mirakan_debug_ai_projection
  <- mirakan_debug_platform_windows
  <- mirakan_debug_platform_android
  <- mirakan_debug_platform_apple
```

- `mirakan_debug_contracts`はFoundation、MCD generated type、`StableId`／`McdContractRefV1`／`ArtifactRefV1`／runtime handle、time IDだけへ依存する。
- Subsystemは`mirakan_debug_runtime`のPortを使い、Store、Editor、AIへ依存しない。
- Platform AdapterはD3D12／Vulkan／Metal／OS型をcommon contractへ漏らさない。
- Shippingで無効なModuleをempty success stubへ置き換えず、Capability Catalogに`unavailable`を記録する。

## 6. CapabilityとConfiguration

### 6.1 Capability成熟度

| Level | 名称 | 到達点 |
|---|---|---|
| C0 | Diagnostic Foundation | 共通ID、Envelope、Counter registry、flight recorder、Query fixture、Crash breadcrumb |
| C1 | Local First Playable Debug | Windows local Play、Console／Problems／Profiler／Timeline、Breakpoint／Watch、safe pause／tick step、local record／scrub、Causality、Replay divergence、Reproduction Bundle、外部IDE handoff、AI read-only diagnosis |
| C2 | Production Debugging | comparison baseline、remote device、cross-target symbolication、Domain拡張、longer capture／retention、AI supervised repair evaluation |
| C3 | Specialized Research | 大規模scenario自動原因候補、online crash aggregation、recorded data上の高度なbisection。別Threat Modelと承認が必要 |

C3は「完全な未来予測」「未記録状態の逆実行」「AIの無人修正」を意味しない。

### 6.2 Build Configuration

| Configuration | Instrumentation | Debug attach | Export |
|---|---|---|---|
| Development | D0～D3、validation、symbol、source map | local／認証済みdevice | 明示User操作 |
| Profile | D0～D2、性能条件をmanifestへ固定 | local／認証済みdevice | Capture Receipt付き |
| Shipping | D0 fault／budget／crash最小集合だけ | 不可 | Crash consentに従う |
| ASan／Qualification | D0～D3、sanitizer／validation、failure injection | CIまたはlocal | restricted artifact |

`Profile` performance acceptance runではDRED、GPU validation、詳細trace等の状態をmanifestへ記録し、baselineと同一条件にする。Instrumentation有効の結果と無効の結果を無注記で比較しない。

### 6.3 Instrumentation Tier

| Tier | 用途 | 記録 |
|---|---|---|
| D0 `fault_minimal` | Shippingを含む安全診断 | fault code、last phase、bounded breadcrumb、budget violation、crash metadata |
| D1 `baseline` | 通常Development／Profile | phase span、System／job、Counter、Diagnostic、Command／Event相関 |
| D2 `interactive` | 選択対象の深掘り | State sample、World shape、Watch、Causality edge、Domain detail |
| D3 `capture` | 明示短時間Capture | Render／GPU／allocation／high-rate event、external tool marker |

TierはProject SourceではなくDebug Session stateであり、Game behaviorを変えない。D2／D3有効化によってdeadlineを守れない場合、記録をdrop／stopし、authoritative simulationを遅延させて成功扱いしない。deterministic Qualificationが明示的にsimulation pauseを許すfixtureだけは例外とする。

## 7. 共通IdentityとSchema

### 7.1 `DebugTargetRefV1`

```text
DebugTargetRefV1
  target_kind
  identity_kind
  source_stable_id?
  mcd_contract_ref?
  artifact_ref?
  runtime_object_ref?
  domain_local_ref?
  parent_target_ref?
  project_revision?
  world_ref?
  display_label_key?
```

`target_kind`は`process | world | level | system | entity | component | asset | definition | node | job | queue | render_pass | gpu_resource | device | build_artifact`のclosed enum、`identity_kind`は`source_stable_id | mcd_contract_ref | artifact_ref | runtime_object_ref | domain_local_ref`のclosed enumとする。選択したkindに対応するidentity Fieldを厳密に一つだけ持ち、他のidentity Fieldは0件とする。

| Identity Field | 型／規則 |
|---|---|
| `source_stable_id` | RFC 9562 UUIDv7 `StableId`。Source World／Level／Entity／Asset／Definition／Node等 |
| `mcd_contract_ref` | exact `McdContractRefV1`。Game System、Component Type、Command、Event、Diagnostic等 |
| `artifact_ref` | exact `ArtifactRefV1`。Build、Cooked package、Render pass plan、Capture等 |
| `runtime_object_ref` | `{debug_session_id: StableId, runtime_instance_id: uint64, handle: {index: uint32, generation: uint32}}`。0 handle invalid、当該Session外で比較／永続化禁止 |
| `domain_local_ref` | `{owner_ref: StableId | ArtifactRefV1, domain_type_ref: McdContractRefV1, value: uint64}`。owner文書のbit幅、0、割当、永続規則へ従う |
| `parent_target_ref` | 0または1件。Component instance、runtime System、GPU resource等のownerをtyped refで指定 |
| `project_revision` | Source／Project相関時だけexact `uint64` |
| `world_ref` | 0または1件のUUIDv7 World `StableId`＋World definition version |
| `display_label_key` | 表示専用Localization Key `StableId`。identity／dispatchに使わない |

SourceとRuntimeの両方を持つ対象は二つの`DebugTargetRefV1`とtyped correlation edgeで結び、一つのFieldへ混在させない。`runtime_instance_id`はDebug Session内で1から単調増加して再利用せず、0をinvalidとする。`domain_local_ref.value`はowner文書のbit幅を超える上位bitがすべて0でなければ拒否する。Runtime Game Systemのcompact `system_id`は`domain_local_ref`としてGame System Graph `ArtifactRefV1`をownerにする。Native pointer、vendor handle、process内slotだけをpublic identityにせず、runtime handleのgenerationが変わった対象を同一live objectとしてWatchしない。

### 7.2 `DebugTimePointV1`

```text
DebugTimePointV1
  session_id
  monotonic_sequence
  tick_id?
  phase_id?
  frame_id?
  render_phase_id?
  monotonic_ns?
  wall_clock_utc?
  clock_domain
```

- 因果順序は`monotonic_sequence`とRuntime phaseを正本にする。
- `wall_clock_utc`は表示／外部Artifact相関用で、authoritative順序に使用しない。
- CPU、GPU、Audio、Device clockは`clock_domain`とCalibration Receiptなしに同一時刻とみなさない。
- Clock変換にはsource／destination、sample数、offset、drift、最大error、valid intervalを持つ`ClockCorrelationV1`を使用する。

### 7.3 `DebugSessionDescriptorV1`

| Field | 規則 |
|---|---|
| `session_id` | RFC 9562 UUIDv7 `StableId`。再利用しない |
| `project_id`／`project_revision` | exact |
| `play_session_id` | Runtime Play時は必須 |
| `build_receipt_ref` | Engine、Game Module、Contract set、Toolchain、Configurationを固定 |
| `target_profile_id` | exact |
| `processes` | Process role、instance ID、executable hash、start generation |
| `instrumentation_tier` | D0～D3 |
| `enabled_channel_ids` | 登録済みclosed set、最大256 |
| `retention_profile_id` | exact |
| `privacy_profile_id` | exact |
| `capture_reason` | `manual | breakpoint | diagnostic | crash | hang | test | ai_requested_user_approved` |
| `started_at`／`ended_at` | UTC＋monotonic |
| `completeness` | `complete | partial | crashed | truncated | rejected` |
| `gap_summary` | Channel別drop／欠落 |

同じDebug SessionへProject revision、Build、Target Profileの異なるProcessを混在させない。GameHost restartは新Sessionまたは明示child sessionを作り、旧sequenceを継続しない。

### 7.4 `DebugEventTypeV1`

| Field | 規則 |
|---|---|
| `event_type_id`／`version` | `debug.event.<domain>.<name>.vN` |
| `channel_id` | 1件 |
| `category` | `log | diagnostic | span | counter_sample | state_sample | command | event | breakpoint | watch | replay | resource | world_shape | crash | gap` |
| `field_schema_ref` | bounded MCD Type |
| `max_payload_bytes` | D0 1 KiB、D1 4 KiB、D2／D3 64 KiB以下 |
| `priority` | `critical | high | normal | verbose` |
| `retention_class` | `flight | session | capture | external_ref` |
| `privacy_class` | `public | project | source_sensitive | personal | secret_forbidden` |
| `causal_policy` | required parent／correlation edge |
| `shipping_policy` | `allowed_redacted | crash_only | forbidden` |

未登録event type、unbounded string／array、pointer、native handle、secret fieldをemitしない。Large binaryはpayloadへ入れずcontent-addressed Capture Artifactを参照する。

### 7.5 `DebugEventEnvelopeV1`

```text
DebugEventEnvelopeV1
  envelope_version = 1
  event_type_id
  event_type_version
  session_id
  sequence
  time_point
  producer_process_id
  producer_thread_role
  target_refs[0..16]
  correlation_id?
  parent_event_ids[0..8]
  trace_id?
  span_id?
  diagnostic_id?
  payload_hash
  payload
  redaction_flags
```

Envelopeはcanonical serializationとhashを持つ。`producer_thread_role`は`main_runtime | render | worker | audio_callback | platform_callback | tool | device_bridge`等のroleであり、OS thread IDはoptional evidenceに限定する。

## 8. Debug Session lifecycle

```text
Requested
  -> Validating
  -> Preparing
  -> Recording <-> PausedAtSafeBoundary
  -> Finalizing
  -> Indexed
  -> Closed

Requested | Validating | Preparing -> Rejected
Recording | PausedAtSafeBoundary -> Truncated | Crashed
Finalizing -> Partial
```

### 8.1 Start

1. User／CI／authorized AI requestがTarget、reason、tier、channel、duration、retention、privacyを指定する。
2. Debug GatewayがCapability、Build、Target、available memory／disk、device trust、consentを検証する。
3. GameHostへfixed-size bufferとChannel maskを設定する。
4. `DebugSessionDescriptorV1`とstart markerをCommitしてから記録を開始する。
5. AI requestの場合、D2／D3、remote device、source-sensitive channel、exportは人間承認なしに開始しない。

### 8.2 Stop

- explicit stop、duration／size limit、safe breakpoint、Process exit、crash、critical overflowで停止する。
- Storeはproducerを先に停止し、queue drain、footer、gap summary、artifact hash、Index生成の順で閉じる。
- Crash時にnormal footerを捏造せず`completeness=crashed`とlast durable sequenceを記録する。
- Index失敗時もraw canonical chunkを保持できるが、AIへunindexed full scanを許可しない。

### 8.3 Pause

live authoritative PlayのPauseは`T110_Publish`完了後だけ成立する。Worker callback、Physics sub-step、Audio callback、Render submission途中でWorldを停止してInspectorへ公開しない。Breakpointが途中でhitした場合はhit Evidenceをbufferし、最初のsafe boundaryでPauseする。

Pause中は次を区別する。

- `AuthoritativeFrozen`: Runtime Worldはread-only、InputはReplay control以外を配送しない。
- `PresentationFree`: Editor camera、Overlay、Timeline scrubだけを動かせる。
- `SandboxEvaluation`: GameplayDefinitionまたはGraphのコピーStateでstepする。live WorldへCommandをsealしない。

## 9. Log、Diagnostic、Span、Counter、Snapshot

### 9.1 Log

`DebugLogRecordV1`は`message_key`、primitive arguments、severity、category、target、source location、correlationを持つ。完成済み文字列だけを保存しない。User表示時にLocalizationし、AIはkeyとtyped argumentsを読む。

同じ原因から発生する反復Logは`dedup_key`、first／last sequence、countを集約する。Rate limitで抑制した件数を記録し、抑制を「発生なし」と表示しない。

### 9.2 Diagnostic

Validation、Build、Runtime、Platform findingは実行可能契約規約の`MirakanDiagnosticV1`を正本にする。Debug eventはDiagnostic IDを参照し、code、expected／actual、location、remediation、cause chainを複製しない。

- Runtime validation／Gameplay failureをSARIFへ無理に変換しない。
- Compiler／static analysis／security findingは`MirakanDiagnosticV1`へ正規化した後、外部連携用SARIF 2.1.0を生成できる。
- Vulkan VUID、D3D12 message ID、Metal validation code等はBackend evidence fieldへ保持し、Miraikanai codeと分離する。

### 9.3 Span

`DebugSpanV1`はname ID、parent span、start／end、status、target、phase、budget、actualを持つ。文字列動的nameやEntity IDをspan nameへ埋め込まず、fieldに分離してcardinalityを制限する。

CPU span、job、GPU marker、Device traceは同じ`trace_id`を共有できるが、Clock correlation errorを超えて厳密な順序を主張しない。

### 9.4 Counter

`DebugCounterDefinitionV1`は次を持つ。

```text
counter_id
version
unit
value_type
aggregation
scope_kind
sampling_policy
expected_range?
soft_budget?
hard_budget?
privacy_class
shipping_policy
```

- `aggregation`は`gauge | monotonic_sum | histogram | duration | ratio`のclosed enum。
- unitなしnumber、同じIDのunit変更、表示名だけのcounterを拒否する。
- P50／P95／P99.9はsample集合、window、warm-up、missing sampleをReceiptへ記録する。
- Editor graphとheadless Gateは同じcounter IDを使う。

### 9.5 Immutable Debug Snapshot

Subsystemは`DebugProjectionPortV1`を実装し、safe boundary後のbounded immutable snapshotだけを公開する。

```text
DomainDebugSnapshotEnvelopeV1
  snapshot_type_id
  snapshot_version
  session_id
  time_point
  source_generation
  target_refs[]
  completeness
  omitted_field_ids[]
  payload
```

Scene View、Profiler、AIがNative World、Physics kernel、Nav backend、GPU objectへ直接accessしない。Snapshot生成がBudgetを超えた場合は範囲を無通知で変えず、`partial`とomitted fieldを返す。

### 9.6 World Debug Shape

`DebugWorldShapeV1`は`point | line | polyline | ray | aabb | obb | circle | sphere | capsule | cone | frustum | text_anchor`だけをC1で許可する。Shapeはtarget ID、time range、space、unit、style tokenを持ち、arbitrary mesh／shaderをDebug pathから実行しない。

## 10. Store、Index、Query

### 10.1 Store

canonical Storeはappend-only chunkとする。Chunkはsession、channel、sequence range、time range、schema set hash、compression、uncompressed／compressed byte、content hash、previous chunk hashを持つ。

- Crash耐性のためfooter未完成Chunkを検出し、last complete recordまでrecoverする。
- Project Source treeへCaptureを自動保存しない。
- Capture ArtifactはUser data directoryまたはCI artifact storeへ保存し、Projectにはcontent hashと任意referenceだけを置く。
- External formatへexportしてもcanonical Storeを無断削除しない。Retention expiryはReceiptを残す。

### 10.2 Index

最低限次のIndexを持つ。

- sequence／time
- event type／channel／severity
- canonical `DebugTargetRefV1` hash／target kind／identity kind
- tick／phase／frame
- System／Entity／Component／Asset
- Diagnostic code／cause chain
- trace／span／correlation／parent
- breakpoint／watch hit
- replay checkpoint／state hash

Indexは`session_id`、Store root hash、schema set hash、index generationを持つ。Store追加、repair、redaction後は古いIndexをstaleとして拒否する。

### 10.3 `DebugQueryV1`

| Field | 規則 |
|---|---|
| `session_id`／`index_generation` | exact |
| `sequence_range`／`time_range` | 少なくとも一方。無指定全件scan禁止 |
| `target_selectors` | 最大64 |
| `event_type_ids`／`channel_ids` | 合計最大256 |
| `severity_min` | optional |
| `correlation_refs` | 最大32 |
| `field_mask` | required |
| `order` | `sequence_asc | sequence_desc` |
| `limit` | default 256、最大4,096 |
| `max_result_bytes` | default 256 KiB、最大4 MiB |
| `cursor` | opaque、session／index generation固定 |
| `aggregation` | noneまたは登録済みAggregate ID |

AI Toolの一回の返却は最大256 recordか256 KiBの小さい方とし、全Session dumpをPromptへ投入しない。より広い調査はAggregate→narrow Query→Replay Sliceの順で行う。Userが明示許可してもProvider context上限を超えるraw payloadを送らない。

### 10.4 Query結果

`DebugQueryResultV1`はrecordだけでなく次を返す。

- applied filter
- scanned／matched／returned count
- omitted field
- first／last sequence
- gap／redaction／clock uncertainty
- completeness
- next cursor
- query cost
- Store／Index hash

`0件`と`記録欠落により不明`を区別する。

## 11. Causality Graph

### 11.1 Edge

`DebugCausalEdgeV1`のkindは次へ限定する。

```text
input_produced
command_emitted
command_consumed
state_read
state_written
event_emitted
event_delivered
async_requested
async_accepted
job_scheduled
job_completed
resource_waited
rng_consumed
checkpoint_compared
asset_resolved
level_activated
presentation_derived
diagnostic_caused
fallback_selected
```

Edgeはsource event、destination event、target、tick distance、delivery class、completenessを持つ。時間的に前後しただけのEventを因果Edgeにしない。Parent／correlationが欠落した推定Edgeは`inferred`とし、validated causeに使用しない。

### 11.2 `CausalityGraphV1`

| Field | 規則 |
|---|---|
| `root_event_ids` | 1～16 |
| `direction` | upstream／downstream／both |
| `max_depth` | default 8、最大32 |
| `max_nodes` | default 512、最大4,096 |
| `allowed_edge_kinds` | closed subset |
| `time_bound` | required |
| `nodes`／`edges` | canonical sequence順 |
| `frontier` | capにより未展開のnode |
| `gaps` | missing parent／dropped channel |
| `completeness` | complete／bounded／partial |

代表Questionを型付きQueryへ変換する。

| User／AIの質問 | 起点 | 必須Edge |
|---|---|---|
| なぜDamageを受けたか | Damage state delta／Event | `event_delivered`→`command_emitted`→`command_consumed`→`state_written` |
| なぜ敵が動かなかったか | Character transform不変／AI diagnostic | `state_read`→`async_requested`→`async_accepted`→`command_emitted`→`command_consumed`→`state_written` |
| なぜ表示されなかったか | presentation missing | `state_written`→`event_emitted`→`presentation_derived`→`asset_resolved` |
| なぜFrameが遅いか | frame budget violation | `job_scheduled`→`job_completed`／`resource_waited`とspan parent |
| なぜReplayがずれたか | first mismatch hash | `checkpoint_compared`→`input_produced`／`async_accepted`／`rng_consumed`→`state_written` |

PresentationからGameplayへ逆因果Edgeを作らない。Render visibility、LOD、GPU resultをauthoritative Gameplay原因として提示しない。

## 12. Breakpoint、Watch、Pause、Step

### 12.1 `DebugBreakpointV1`

```text
breakpoint_id
version
enabled
scope_session_id?
breakpoint_kind
target_selector
condition_predicate_ref?
hit_policy
action
safe_boundary_policy
capture_channel_ids[]
owner
expires_at?
```

`breakpoint_kind`は次とする。

- `tick`
- `phase_boundary`
- `system_entry`
- `system_exit`
- `definition_entrypoint`
- `definition_node`
- `command_type`
- `event_type`
- `state_predicate`
- `diagnostic_code`
- `budget_threshold`
- `asset_generation`
- `level_transition`
- `render_marker_capture`

`condition_predicate_ref`はMCDで登録したpure、bounded、no-allocation predicateだけを使う。任意C++／Script式、filesystem、network、clock、random、World mutationを条件にしない。

`action`は`mark | capture | pause_at_safe_boundary | stop_recording | fail_qualification`のclosed enum。GPU render markerはlive GPUを中断せずCapture triggerに限定する。

### 12.2 Hit policy

```text
first
every
after_count(n)
every_n(n)
once_per_target_generation
rate_limited(window, max_hits)
```

Breakpoint stormでGameを停止し続けない。Hit数、suppressed数、first／last hitをCounterへ記録する。

### 12.3 Watch

`DebugWatchV1`はStable target、MCD field path、sample phase、sampling interval、history capacity、comparison predicate、privacy classを持つ。

- pointer chain、native offset、任意expressionをWatch targetにしない。
- Target generation変更時は`retired`とし、新Objectへ自動追従しない。
- authoritative field、derived field、presentation fieldをUIとSchemaで区別する。
- Containerは全要素を無制限展開せず、count、hash、bounded sliceを返す。
- Secret／credential／personal fieldは登録時に拒否またはredactする。

### 12.4 Step

| Operation | live authoritative session | sandbox／recording |
|---|---|---|
| `StepTick` | 1 authoritative tickを全phase完了しT110で再Pause | 可 |
| `StepRenderFrame` | World固定のまま1 presentation frame。Gameplayへ逆入力しない | 可 |
| `StepGameplayDefinitionNode` | live transactionでは不可 | copied Stateとdiscarded command bufferで可 |
| `StepPhase` | 不可。途中stateを公開しない | Qualification sandboxだけ |
| `StepRenderEvent` | live GPUでは不可 | captured event listの表示stepだけ |
| source line step | Engineは制御しない | external IDE |

Step中もReplay control Input以外をGameplayへ混ぜない。`StepRenderFrame`でAudio time、Physics、AI、Gameplay clockを進めない。

## 13. Replay、Record、Rewind

### 13.1 authoritative Replay

Runtime規約のReplayを正本とし、最低限次を記録する。

- Project revision、Contract set、Build、Target、System implementation set
- fixed tick／phase profile、worker profile
- normalized InputSnapshot
- RNG stream seedとdraw count
- accepted async resultのpayload hashとaccept tick
- authoritative Command／Event oracle
- state hash
- checkpoint
- deterministic fault injection

OS raw packet、pointer、GPU output、wall clock、Presentation cacheをauthoritative Replayへ入れない。

### 13.2 `ReplaySliceV1`

```text
replay_slice_id
source_session_id
build_receipt_ref
project_revision
start_checkpoint_ref
start_tick
end_tick
input_range_ref
accepted_async_range_ref
rng_range_ref
expected_state_hashes
expected_diagnostics
required_asset_versions
redaction_manifest
content_hash
```

Sliceは問題の直前checkpointから、問題を観測できる最小end tickまでを基準にする。必要Asset／Definition closureが欠落するSliceをportable reproductionと表示しない。

### 13.3 Rewind

Rewindは記録済みSnapshot／Event／State sampleをtimeline上で閲覧する機能であり、live Worldを過去へ巻き戻してそのまま継続する機能ではない。

- recorded Object／Component trackをscrubする。
- selected tickのScene overlay、Watch、Diagnostic、Causality Graphを同期表示する。
- current Projectとrecorded revisionを混同しない。
- recorded Assetが利用不能な場合、geometry／imageを推測表示せずmetadata-onlyとする。
- 過去時点から再実行する場合はcheckpointから新しいchild Replay sessionを開始する。

### 13.4 Divergence

Replay state hash不一致はbinary search可能なcheckpoint／tick indexを持つ。Reportは次を含む。

- first mismatching tick
- first mismatching System／State Type／Field ID
- expected／actual canonical hash
- Input、RNG、accepted async result差
- Build／Contract／Asset／Worker Profile差
- preceding Causality frontier
- record gap

Fieldごとの値を保存していない場合はhash差までとし、値を推測しない。

## 14. Domain Debug Projection

各Subsystemは同じEnvelope、Target、Time、Queryを使い、Domain固有snapshotを所有する。

| Domain | C1必須Projection |
|---|---|
| Gameplay System | State owner、entrypoint、Command／Event、State delta、task、budget、Definition／Native variant |
| World／Level | active Level、activation group、Cell／dependency、transition、failure、old Level維持 |
| Physics／Collision | Body／shape、contact、trigger、island、sleep、velocity、constraint、timeline、query |
| Navigation | tile／grid、agent、path、failed query、stale result、reachability |
| Animation | state、transition、root motion、pose generation、event、LOD |
| Rendering | Render pass、resource、barrier、visibility、LOD、draw／dispatch、GPU marker、fallback |
| Asset | Source／Derived／Package version、dependency、residency、stream request、promotion |
| Input | Action value／transition、Context、consumed owner、Replay source |
| UI | focus、event route、binding、layout diagnostic、safe area、accessibility |
| Audio | cue／voice／bus、virtualization、underrun、device route。callback内dataはbounded |
| VFX／Environment／Water／Snow | seed、emitter／profile、artifact、bounds、budget、fallback、Domain固有overlay |
| Camera | active Rig、Director rule、Blend、collision、history reset、recording／device |

Domain Panelの便利な表示名をtarget identityにしない。全Projectionはtyped `DebugTargetRefV1`からSource、Requirement、Decision、Build、Testへnavigationできる。

## 15. Editor Debug Workspace

### 15.1 C1 Panel

| Panel | 責務 |
|---|---|
| Debug Session | Target、Build、revision、tier、channel、recording、gap、privacy |
| Console | structured Log、category／severity／target filter、dedup、source navigation |
| Problems | `MirakanDiagnosticV1`、cause chain、expected／actual、remediation、status |
| Timeline | tick／phase／frame／job／System／event／spanの時間軸 |
| Causality | upstream／downstream Graph、frontier、gap、Evidence |
| Breakpoints | typed breakpoint、hit、suppression、safe-boundary status |
| Watch | authoritative／derived／presentation field、history、generation |
| Profiler | Counter／Span、P50／P95／P99.9、baseline comparison、budget |
| Replay／Rewind | checkpoint、record、scrub、divergence、child replay |
| Scene Debug | Domain overlay、snapshot time、space、unit、completeness |
| Reproduction | Bundle内容、redaction、closure、export／import、verification |
| External Tools | IDE、PIX／RenderDoc／Perfetto／Xcode等のlaunch／capture reference |
| AI Diagnosis | question、scope、Evidence、hypothesis、counterevidence、next query、Proposal |

既存のGame、History／Diff、Source、Test、AI Partnerと同じWorkspaceへdockできる。独立した別Editorを作らない。

### 15.2 操作

1. Problems、Console、Scene、Profilerから対象Eventへ移動する。
2. EventからCausality upstream／downstreamを開く。
3. Timeline、Scene overlay、Watch、Replayを同じtime pointへ同期する。
4. recorded stateとcurrent Source／revisionを並べ、差異を常時表示する。
5. AIへ渡す前に対象、時間範囲、data category、redaction、推定Context量をPreviewする。
6. AI FindingからEvidenceへ戻れない主張を表示上のvalidated causeにしない。
7. ProposalはDebug Panel内で直接適用せず、通常Diff／Validation／Approvalへ送る。

### 15.3 Accessibility

- 色だけでseverity、recorded／live、authoritative／derived、gapを区別しない。
- Timeline、Graph、Overlayの同じ情報をtable／tree／text summaryでも提供する。
- keyboardだけでrecord、pause、step、filter、Evidence navigationを操作できる。
- Screen readerへ大量Eventを一括公開せず、filter済みvirtualized collectionを提供する。
- AI Semantic Snapshotはvisual primitiveではなく同じPanel projectionから生成する。

## 16. Source／Native／GPU Debugger連携

### 16.1 External IDE

C++ source breakpoint、call stack、native variable、thread、disassemblyはVisual Studioまたは承認済みIDEへ委譲する。Engineは`ExternalDebuggerLaunchDescriptorV1`を生成する。

```text
process_instance_id
executable_path_ref
executable_hash
build_receipt_ref
symbol_artifact_refs[]
source_mapping_ref
working_directory_ref
attach_transport
target_device_ref?
wait_for_attach
security_manifest_ref
```

- Development／Qualification Buildだけが生成できる。
- PDBとSource mappingのhashをBuild Receiptへ固定する。
- Editorはprocess、Project、Buildが一致することを確認してIDEを開く。
- AIはnative debuggerを自動操作せず、Userがexportしたstack／dump解析結果をtyped Evidenceとしてimportできる。
- Engine／GameHost停止中のIDE評価式によるmutationをProject変更として取り込まない。

### 16.2 GameplayDefinition Debugger

GameplayDefinitionはEngine-owned evaluatorのため、node、entrypoint、State field、Command／Eventを型付きDebuggerで扱う。Call stackに相当するものは保存可能な`EvaluationTraceV1`であり、native stackやcoroutineを公開しない。

Sandbox stepはcopied immutable input、copied State、discard-only delta journal、discard-only command bufferを使用する。実行結果をlive Worldへ適用するには通常ChangeSet／Replay fixtureが必要である。

### 16.3 GPU

- D3D12はRender Graph Plan／Passの`ArtifactRefV1`とGPU resourceのsession-scoped `runtime_object_ref`をPIX marker、DRED breadcrumb、object debug nameへ写像する。
- Vulkanは`VK_EXT_debug_utils` object name／queue label／command labelとVUIDを写像する。
- MetalはXcode GPU capture／Metal validation／signpostへ同じpass／resource IDを写像する。
- GPU captureは外部Artifactとしてhash、Tool／version、device／driver、frame、Project revision、Render Graph hashを登録する。
- external captureをAI Providerへ自動uploadしない。
- Validation／GPU-assisted validation／shader printfはQualification用で、Shippingへ含めない。

## 17. AI Debug ContextとDiagnosis

### 17.1 `AiDebugContextV1`

```text
context_id
question
session_id
project_revision
build_receipt_ref
scope_targets[]
time_range
selected_query_result_refs[]
causality_graph_refs[]
replay_slice_refs[]
diagnostic_refs[]
capture_artifact_metadata_refs[]
redaction_manifest
gap_summary
context_bytes
token_estimate
allowed_operation_ids[]
```

AIは最初にAggregateとProblemsを読み、次に対象／時間を狭める。全Trace、全Project Source、全Sceneを一度に要求しない。Sourceが必要な場合はAIガバナンス規約のContext PlanとUser許可を別途満たす。

### 17.2 `DebugFindingV1`

| Field | 規則 |
|---|---|
| `finding_id`／`version` | Stable |
| `status` | `observation | hypothesis | validated_cause | disproved | unresolved` |
| `claim_key`／`arguments` | typed。完成文だけにしない |
| `scope` | Target／time／revision／Build |
| `evidence_refs` | 1～64 |
| `counterevidence_refs` | 0～64 |
| `gap_refs` | 0～32 |
| `causal_path_refs` | 0～16 |
| `reproduction_ref` | validated causeでは必須 |
| `falsification_query_refs` | hypothesisでは1件以上 |
| `confidence_band` | `low | medium | high`＋calibration policy |
| `next_query_refs` | 最大8 |
| `remediation_refs` | 最大8 |
| `owner` | human／AI＋Provider／Model／Prompt Receipt |

AIの自由文説明はこの型から投影する。Evidence refの存在、scope、hash、redactionをGatewayが検証する。

### 17.3 Causeの昇格

`validated_cause`へ昇格できるのは次のいずれかを満たす場合だけである。

1. 決定論的Replayで同じfailureを再現し、該当原因を除いたcounterfactual fixtureで消える。
2. Engine invariant／validatorが一意のcause codeと対象を返す。
3. Crash／validation／contract violationが公式Platform evidenceとMirakanDiagnostic cause chainで一意に結ばれる。
4. 人間ReviewerがEvidence、反証、再現手順を承認する。

単なる時間的相関、LLMの確信、既知パターンとの類似、Screenshotだけでは`hypothesis`を超えない。

### 17.4 AI Diagnosis workflow

```text
Question
  -> Scope / privacy preview
  -> Aggregate query
  -> Narrow query
  -> Causality / Replay Slice
  -> Hypothesis + falsification
  -> Reproduce
  -> DebugFinding validation
  -> Remediation / ChangeSet proposal
  -> Approval
  -> Replay + Test + performance regression
  -> DebugReceipt
```

- AIは追加instrumentationが必要なら理由、Channel、Tier、duration、overhead、privacyを提案する。
- D2／D3 Capture、remote device、source-sensitive dataはUser承認後だけ開始する。
- Permission、Security、Privacy、Approval、lock、revision driftを自動修復しない。
- 同じblocking集合が減らない自動repairは最大2回で停止する。
- Fix後に元Replayだけを通して完了せず、関連Unit／integration／performance／fault fixtureを実行する。

### 17.5 AIへ禁止する推論

- 欠落Eventを「発生しなかった」と断定する。
- Presentation visibilityからGameplay authorityを推定する。
- Target generationの異なるObjectを同一Objectとして連結する。
- Debug buildだけの挙動をShippingへ一般化する。
- Instrumentation overheadを除外せずperformance原因を断定する。
- current Sourceをrecorded Buildで実行されたSourceとみなす。
- redacted valueを既定値または空値とみなす。
- 外部EngineのError messageと名前が似るだけで同じ修正を適用する。

## 18. Reproduction、Crash、Hang

### 18.1 `ReproductionBundleV1`

```text
bundle_id
bundle_version
issue_summary_key
source_session_ref
project_revision
build_receipt_ref
target_profile_ref
replay_slice_ref?
required_artifact_refs[]
diagnostic_refs[]
capture_refs[]
expected_failure_oracle
run_instructions_ref
privacy_manifest
license_manifest
redaction_manifest
expiry
content_manifest_hash
signature?
```

BundleはProject全体を無条件に複製しない。最小Artifact closure、Replay、fixture、public／project-sensitive dataを区別する。Secret、credential、signing material、private clipboard、AI prompt、個人dataを含めない。

Import時はhash、Schema、Build availability、license、privacy、path traversal、size、signatureを検証し、隔離Workspaceで開く。Bundleを開くだけでC++、shader、importerをEditor Process内実行しない。

### 18.2 Crash

Windows規約の`CrashEnvelopeV1`を正本とし、本書のSession、Build、last durable sequence、gap、Replay checkpoint、breadcrumbを参照できるようにする。

- in-process crash handlerはfixed preallocated metadataだけを書く。
- dump、symbol、Sourceは別Artifactとし、access controlを分離する。
- symbolicationはexact executable／module hashとsymbol hashが一致する場合だけ行う。
- partial stack、unknown module、optimized-away frameを推測補完しない。
- Userが明示exportするまでAI Providerまたはonline serviceへ送らない。

### 18.3 Hang

Heartbeatはmain simulation、render submission、window、audio control、worker poolを分ける。Hang watchdogは次を記録する。

- last progress sequence／time
- active phase／frame
- bounded thread role stacksまたはPlatform sample reference
- queue depth／oldest item
- lock-order checker state
- GPU fence／device status
- last critical Diagnostic

Threadを強制resumeしてGame継続を成功扱いしない。Qualificationではhang oracleに達した時点でProcessを終了し、artifactを回収する。

## 19. Remote Device Debug

### 19.1 Handshake

`DeviceDebugHandshakeV1`はDevice identity、pairing generation、App／Engine／Module hash、Target Profile、Debug Capability、supported channel、max bandwidth、storage、clock correlation、privacy stateを持つ。

- Development／Profile packageだけがDevice Bridgeへ接続できる。
- local pairing、short-lived session key、mutual authentication、ACL／platform entitlementを必須にする。
- discovery成功だけで信頼しない。
- Build／Project／Contract hash不一致はattachを拒否する。
- Device Bridgeは任意filesystem、shell、process、network proxyを提供しない。

### 19.2 Transfer

- Controlとbulk Captureを別bounded channelへ分ける。
- Counter／Diagnosticを優先し、verbose shape／sampleを先にdropする。
- disconnect時はDevice側local ringへ継続できるが、上限超過をgapとして記録する。
- resumeはlast acknowledged chunk hashから行い、重複をcanonical sequenceで除外する。
- Android Performance Analyzer／Perfetto、AGI、Vulkan validation、Apple Instruments／Metal captureは外部Capture Adapterとして登録する。Androidのsystem profilingはAPA／Perfetto、Vulkanのframe深掘りはAGIへ責務を分ける。

### 19.3 Remote mutation

remote deviceではWatch、Capture、Pause／Resume、Replay controlだけをC2候補とする。GameplayDefinition／Asset hot reloadはモバイル規約の互換性検証済み経路だけを使う。C++、shader、native plugin変更はrebuild／reinstallを必須とする。

## 20. Security、Privacy、Retention

### 20.1 Risk

| Operation | 基準Risk |
|---|---:|
| 登録済みCounter／Diagnostic／bounded Query | R0 |
| local DevelopmentのD1 record、filter、Watch | R1 |
| Pause／Step、D2 state capture、external IDE handoff | R1 |
| D3 GPU／allocation capture、remote attach | R2 |
| source-sensitive dataを含むBundle export、unredacted device trace | R2＋人間承認 |
| Debug FindingからProject Source／Authoring変更 | 元OperationのRisk。Debug権限で低下させない |
| Shipping online crash aggregation | C3別Threat Model／Privacy承認 |

### 20.2 Data class

- `public`: Engine version、public Diagnostic code等。
- `project`: typed `DebugTargetRefV1`、Project revision、Game state、Asset metadata。
- `source_sensitive`: source path、symbol、call stack、generated code mapping。
- `personal`: User name、device identifier、Saveに含まれる個人data。
- `secret_forbidden`: credential、token、private key、password、signing material。収集自体を拒否する。

AI Contextは`project`以上を明示scopeとredactionなしに含めない。外部Providerへ送る前にdata category、byte、対象、retention、ProviderをUserへPreviewする。

### 20.3 Shipping

Package inspectionは次を拒否する。

- Debug server／Device Bridge listener
- expression evaluator／runtime compiler
- PDB／unstripped private symbol／Source mapping
- Validation Layer／GPU-assisted validation／shader printf
- local AI credential／Provider debug endpoint
- unrestricted Console command
- Development signing identity
- unredacted capture preset

D0 Crash／fault metadataはPrivacy Profileに従い残せる。Shipping Log levelを下げてもSecurity event、fatal fault、consent stateを無通知で消さない。

### 20.4 Retention Profile

| Profile | Memory ring | Local disk | 用途 |
|---|---:|---:|---|
| `windows_dev_baseline_v1` | GameHost 128 MiB以内 | 1 session 2 GiB、最新10件または14日 | C1 local |
| `windows_profile_v1` | 64 MiB以内 | 1 session 1 GiB、最新10件または14日 | performance比較 |
| `mobile_dev_low_v1` | 16 MiB以内 | 256 MiB、最新3件または7日 | minimum device |
| `mobile_dev_high_v1` | 32 MiB以内 | 512 MiB、最新5件または7日 | reference device |
| `shipping_crash_minimal_v1` | 4 MiB以内 | Crash 10件または30日 | opt-in export |

Project固有Profileはより小さくできる。上限を増やす場合はRuntime memory budget、disk、privacy、package、実機enduranceのReviewを必要とする。期限切れArtifactはsecure delete可能なPlatform APIを使い、削除Receiptへhashと理由だけを残す。

## 21. Performance、Backpressure、Failure

### 21.1 Overhead Gate

同一Build、fixture、Target、QualityでInstrumentation無効と比較する。

| Tier | CPU frame P95増加 | GPU frame P95増加 | Runtime memory | 用途 |
|---|---:|---:|---:|---|
| D0 | 1%かつ0.10 ms以下 | 0.05 ms以下 | Retention Profile内 | Shipping faultを含む |
| D1 | 2%かつ0.20 ms以下 | 0.10 ms以下 | Profile内 | 通常開発 |
| D2 | 5%かつ0.50 ms以下 | 0.25 ms以下 | Profile内 | 対象限定inspection |
| D3 | acceptance performance測定に使用しない | acceptance測定に使用しない | Capture上限内 | 明示Capture |

閾値の両条件を満たす必要がある。平均値だけで合格させない。D2／D3の結果をProduct performance baselineとして使用しない。

Audio callback、GPU submission hot path、Platform callbackではdynamic allocation、blocking lock、formatting、filesystem、networkを禁止する。preallocated fixed recordへ書き、別threadでencodeする。

### 21.2 Priority

```text
critical:
  crash, fatal, security, replay divergence, authoritative fault, gap
high:
  diagnostic, breakpoint, budget violation, state hash, command/event causal edge
normal:
  span, counter, selected state sample, world shape
verbose:
  per-object detail, repeated log, high-rate resource event
```

pressure時はverbose→normalの順でdropする。critical lane overflowではrecordingを`truncated`として停止し、Qualificationは失敗する。通常PlayはDebug failureだけでauthoritative stateを変更しない。

### 21.3 Closed Diagnostic ID

| ID | 条件 | 結果 |
|---|---|---|
| `MIRAKAN-DEBUG-SESSION_CONFIG_INVALID` | Tier／Channel／Profile不整合 | Start拒否 |
| `MIRAKAN-DEBUG-CAPABILITY_UNAVAILABLE` | Build／Targetで未対応 | 必要Configurationを返して拒否 |
| `MIRAKAN-DEBUG-BUILD_IDENTITY_MISMATCH` | Process／symbol／Project不一致 | attach／symbolicate拒否 |
| `MIRAKAN-DEBUG-STORE_LIMIT_REACHED` | disk／retention上限 | recording finalize、partialを明示 |
| `MIRAKAN-DEBUG-EVENT_DROPPED` | queue pressure | gap record、counter増加 |
| `MIRAKAN-DEBUG-CRITICAL_LANE_OVERFLOW` | critical record不能 | recording truncated、Qualification失敗 |
| `MIRAKAN-DEBUG-INDEX_STALE` | Store hash／generation不一致 | Query拒否、reindex |
| `MIRAKAN-DEBUG-QUERY_TOO_BROAD` | time／target／limit不足 | Aggregate／narrow候補を返す |
| `MIRAKAN-DEBUG-TARGET_GENERATION_MISMATCH` | Watch対象がretire | Watch停止 |
| `MIRAKAN-DEBUG-UNSAFE_PAUSE_POINT` | live途中phaseで停止要求 | 次safe boundaryへ延期 |
| `MIRAKAN-DEBUG-REPLAY_DIVERGED` | state hash不一致 | first mismatchをReport |
| `MIRAKAN-DEBUG-REPLAY_CLOSURE_MISSING` | Artifact／Profile不足 | portable reproduction拒否 |
| `MIRAKAN-DEBUG-CLOCK_UNCORRELATED` | clock変換Evidenceなし | cross-domain順序をunknown |
| `MIRAKAN-DEBUG-PRIVACY_APPROVAL_REQUIRED` | sensitive capture／export | 人間承認待ち |
| `MIRAKAN-DEBUG-REDACTION_INCOMPLETE` | forbidden field検出 | Context／Bundle export拒否 |
| `MIRAKAN-DEBUG-AI_EVIDENCE_INSUFFICIENT` | Findingがscope／Evidence不足 | hypothesis以下へdowngrade |
| `MIRAKAN-DEBUG-REMOTE_AUTH_FAILED` | pairing／session key不正 | 接続拒否、Security event |
| `MIRAKAN-DEBUG-EXTERNAL_CAPTURE_UNVERIFIED` | Tool／device／hash不足 | Evidence参考扱い |

FailureをLogだけで通知しない。すべて`MirakanDiagnosticV1`とDebug Session stateへ反映する。

## 22. Test、AI Eval、Qualification

### 22.1 Contract

- 全Typeのvalid／invalid／boundary、unknown field／enum／version。
- canonical serialization、hash、chunk chain、crash途中recovery。
- pointer／native handle／unbounded string／secret fieldの拒否。
- UUIDv7 `StableId`、MCD／Artifact ref、runtime handle generation、Domain-local owner、retire、stale Watch。
- Event type registry、Counter unit／aggregation、Channel policy。
- Query limit、field mask、cursor、Index stale、gap／redaction。

### 22.2 Runtime

- 12 tick phase、8 render phaseとtime pointの対応。
- worker並行emitのcanonical sequenceとno data race。
- critical／high／normal／verboseのdrop順。
- safe-boundary Pause、StepTick、StepRenderFrame。
- sandbox node stepがlive State／Commandを変更しない。
- Debug Store／Index／Panel crashでauthoritative Playが変化しない。
- D0～D2 overhead Gateを2D／3D Reference sceneで3 run測定。
- Audio callback、render submit、Platform callbackのallocation／lock 0。

### 22.3 Replay／Rewind

- Input、RNG、accepted async resultから同じstate hash。
- first divergence tick／System／Field hashの特定。
- current Projectとrecorded revisionの混同拒否。
- Slice closure不足、Asset version違い、worker profile違いの拒否。
- record gapを含むSessionをcomplete reproductionと表示しない。
- child replayが親Sessionを変更しない。

### 22.4 Domain integration

`debug_known_faults_v1`へ少なくとも次を入れる。

1. Input Context競合でAbilityが発火しない。
2. Collision Filter誤設定でDamage Eventが欠落する。
3. Nav resultがstaleでCharacterが停止する。
4. Animation root motion authority競合。
5. Asset generation混在で表示が欠落する。
6. Render Graph barrier違反をValidationが検出する。
7. Audio queue overflow。
8. GameplayDefinition budget fault。
9. Level activation dependency不足で旧Levelを維持する。
10. Replay RNG draw count不一致。
11. GameHost crashとsymbol hash不一致。
12. remote device disconnectとcapture gap。

各Caseは問題の観測、typed Diagnostic、Causality path、Replay Slice、正しいremediation、禁止修正、回帰fixtureを持つ。

### 22.5 AI Eval `debugging_diagnosis`

| 指標 | C1合格値 |
|---|---:|
| golden root causeをtop-1で特定 | 85%以上 |
| golden root causeをtop-3で包含 | 95%以上 |
| Blocking／High Evidence recall | 100% |
| Evidenceなし`validated_cause` | 0 |
| gap／redactionを値なしと誤認 | 0 |
| recorded／current revision混同 | 0 |
| PresentationをGameplay causeへ逆入力 | 0 |
| 存在しないEvent／Target／Field IDを最終提出 | 0 |
| Permission／Security／Privacyを自動緩和 | 0 |
| 同一blocking集合で2回を超える自動repair | 0 |
| 再現不能Caseをfixedと宣言 | 0 |
| 初回Context | 95%以上が24,000 input token以下 |
| 不要なD2／D3 capture要求 | 5%以下 |
| 修正後に元Replay＋関連回帰Gateを実行 | 100% |

`validated_cause`の採点はcode-based Evidence graph、Replay oracle、Engine validatorを優先し、LLM graderを唯一の判定にしない。Provider／Model／Prompt／Tool更新時は同じCorpusをclean stateから3回実行し、最悪回で判定する。

### 22.6 Platform

- Windows: DRED、PIX marker、Debug Layer、GPU-based validation、WER／minidump、PDB hash、ASan、hang watchdog。
- Vulkan: Validation Layer、synchronization／GPU-assisted validation、VUID正規化、`VK_EXT_debug_utils` label。
- Android: ADB pairing、Perfetto capture、disconnect、redacted trace、minimum／reference実機。
- Apple: Metal validation、GPU capture、Instruments／signpost、symbol、device disconnect、privacy。
- Shipping packageに禁止Debug artifactが0件。

### 22.7 Qualification Receipt

`DebugQualificationReceiptV1`は次を固定する。

- Contract set／Schema set hash
- Engine／Game／Module Build hash
- Target／Device／OS／driver／Tool version
- Instrumentation tier／Channel／Retention／Privacy Profile
- fixture／Replay／Capture hash
- completeness／gap
- overhead Before／After
- Contract／Runtime／Replay／Platform／AI Eval結果
- external capture refs
- Reviewer／Approval
- Capability level

subject、Build、Schema、Target、Tool baseline、Privacy Profileが変われば失効する。

## 23. 段階実装

### 23.1 `DBG0_contract`

- `DebugTargetRefV1`、`DebugTimePointV1`、Session、Event Type、Envelope、Counter、Query、GapのMCD。
- `MirakanDiagnosticV1`とcommon trace contextへの接続。
- canonical serialization、hash、invalid fixture。
- Shipping禁止fieldとredaction test。

完了条件は、headless producer→Store→Index→bounded Queryが一つのDiagnostic fixtureで通り、pointer、secret、unbounded payload、stale Indexを拒否することである。

### 23.2 `DBG1_flight_recorder`

- fixed-size ingress、priority lane、chunk Store、crash recovery。
- Phase span、Counter、Command／Event correlation。
- D0／D1 overhead Gate。
- Windows GameHost crash breadcrumb。

完了条件は、Phase 0最小Hostの起動、正常終了、failureをSessionとして記録し、Crash途中Chunkからlast durable recordを回復できることである。

### 23.3 `DBG2_editor_local`

- Debug Session、Console、Problems、Timeline、Profiler、Watch Panel。
- safe Pause、StepTick、StepRenderFrame。
- Scene overlayとDomain Snapshot Port。
- external IDE handoff、Windows PDB／DRED／PIX ID mapping。

完了条件は、Windows最小GameHostの既知failureをConsole→Timeline→Sourceへ追跡し、Projectを変更せず再現できることである。

### 23.4 `DBG3_replay_causality`

- Replay Slice、divergence、Causality Graph、record／scrub。
- Breakpoint、Watch history、Reproduction Bundle。
- `debug_known_faults_v1`。

完了条件は、Input／Collision／GameplayDefinition／Levelの4 failureでfirst causeと最小Replay Sliceをheadless／Editorの両方から同じhashで得ることである。

### 23.5 `DBG4_ai_diagnosis`

- `AiDebugContextV1`、`DebugFindingV1`、Evidence validation。
- Aggregate→narrow→Causality→Replay Tool。
- AI Diagnosis Panel、privacy Preview、Proposal handoff。
- `debugging_diagnosis` Eval。

完了条件は、AIがScreen captureなしでC1 Eval基準を満たし、未許可変更、Evidenceなしcause、2回超repairが0件であること。

### 23.6 `DBG5_remote_shipping`

- Vulkan／Android Performance Analyzer／Perfetto／AGI、Metal／Instruments adapter。
- authenticated Device Bridge、transfer resume、remote gap。
- Windows／Android／Apple Shipping package scan。

完了条件は、Android／Apple reference deviceで同じC1 Debug Contractを使い、disconnect後のpartial captureを誤ってcompleteとせず、Shipping packageにbridge／symbol／validation artifactがないことである。

実装順は`DBG0`→`DBG1`をPhase 0へ、`DBG2`をPhase 2、`DBG3`をPhase 3、`DBG4`をPhase 4、`DBG5`をPhase 7へ接続する。未来Panelを先に空実装しない。

## 24. Definition of Done

C1 AI可読Debuggingは次をすべて満たした時だけ完了する。

- Debug SessionがProject revision、Build、Target、Process、Play sessionをexactに固定する。
- Log、Diagnostic、Span、Counter、Snapshot、World shapeが登録済みSchemaを使う。
- Console、Problems、Timeline、Profiler、Watch、Scene overlay、AIが同じrecordを読む。
- Queryがboundedで、gap、redaction、clock uncertainty、stale Indexを区別する。
- Breakpointがsafe boundaryで停止し、StepTickが全phaseを完了する。
- GameplayDefinition node stepがsandboxからlive Worldへwriteしない。
- ReplayがInput、RNG、accepted async resultから同じstate hashを得る。
- first divergence tickとSystem／Field hashを報告できる。
- Reproduction Bundleが最小Artifact closureとPrivacy manifestを持つ。
- external IDE、DRED／PIX、Platform validationをBuild／Session hashへ関連付ける。
- AI FindingがEvidence、counterevidence、gap、falsification、reproductionを持つ。
- AI修正が通常ChangeSet、Risk、Approval、Test、Replay回帰を通る。
- D0～D2 overhead、memory、disk、drop、soak Gateを満たす。
- Shipping packageに禁止Debug artifactがない。
- `debug_known_faults_v1`と`debugging_diagnosis`がC1基準に合格する。
- `DebugQualificationReceiptV1`が生成され、subject変更で失効する。

## 25. 主要リスクと対策

| リスク | 対策 |
|---|---|
| Trace量が多くAI Contextが破綻 | Aggregate→bounded Query→Replay Slice、field mask、token estimate |
| Log相関を原因と誤認 | explicit causal edge、inferred edge分離、validated cause Gate |
| DebuggerがRuntime順序を壊す | safe-boundary Pause、sandbox step、no phase step |
| Watchがretire済みObjectを読む | session-scoped `runtime_object_ref`＋generation、retired状態 |
| Debug instrumentationで性能問題を作る | D0～D3分離、Before／After overhead、acceptance run条件固定 |
| overflowで重要Evidenceを失う | priority lane、critical reserved、gap必須、Qualification fail |
| AIがDebug権限で変更権限を得る | read-only Query、Proposal handoff、元Risk維持 |
| current Sourceでrecorded Buildを誤解析 | Build Receipt、symbol／source map hash、常時差分表示 |
| remote bridgeがShippingへ混入 | Package scan、別Package ID、Capability unavailable |
| Crash dumpが秘密を含む | preallocated bounded Envelope、data class、redaction、manual export |
| 外部Tool形式へlock-in | Engine-owned canonical Store／ID、Adapter＋Capture Receipt |
| 完全Time Travelを約束して実装不能 | recorded data閲覧とcheckpoint child replayへ限定 |
| Domainごとに別Debuggerが増殖 | 共通Envelope／Query／Panel contract、Domain Snapshotだけ拡張 |
| AIの「修正済み」誤判定 | reproduction oracle、元Replay＋関連回帰、Receipt |

## 26. 一次資料

### 26.1 Unity

- [Unity 6.5 Debug C# code](https://docs.unity3d.com/Manual/managed-code-debugging.html)
- [Unity Profiler](https://docs.unity3d.com/Manual/Profiler.html)
- [Adding profiling information to code](https://docs.unity3d.com/Manual/profiler-adding-information-code.html)
- [Creating custom Profiler counters](https://docs.unity3d.com/Manual/profiler-creating-custom-counters.html)
- [Frame Debugger](https://docs.unity3d.com/Manual/FrameDebugger-landing.html)
- [Physics Debug window](https://docs.unity3d.com/Manual/PhysicsDebugVisualization.html)

### 26.2 Unreal Engine 5.8

- [Unreal Insights](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-insights-in-unreal-engine)
- [Unreal Insights Reference](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-insights-reference-in-unreal-engine-5)
- [Visual Logger](https://dev.epicgames.com/documentation/en-us/unreal-engine/visual-logger-in-unreal-engine)
- [Gameplay Debugger](https://dev.epicgames.com/documentation/en-us/unreal-engine/using-the-gameplay-debugger-in-unreal-engine)
- [Blueprint Debugger](https://dev.epicgames.com/documentation/en-us/unreal-engine/blueprint-debugger-in-unreal-engine)
- [Rewind Debugger](https://dev.epicgames.com/documentation/en-us/unreal-engine/animation-rewind-debugger-in-unreal-engine)

### 26.3 Godot Engine 4.7

- [Overview of debugging tools](https://docs.godotengine.org/en/4.7/tutorials/scripting/debug/overview_of_debugging_tools.html)
- [Debugger panel](https://docs.godotengine.org/en/4.7/tutorials/scripting/debug/debugger_panel.html)
- [Output panel](https://docs.godotengine.org/en/4.7/tutorials/scripting/debug/output_panel.html)
- [ObjectDB Profiler](https://docs.godotengine.org/en/4.7/tutorials/scripting/debug/objectdb_profiler.html)
- [EditorDebuggerPlugin](https://docs.godotengine.org/en/4.7/classes/class_editordebuggerplugin.html)

### 26.4 Platform／標準

- [Visual Studio C++ debugger quickstart](https://learn.microsoft.com/en-us/visualstudio/debugger/quickstart-debug-with-cplusplus?view=visualstudio)
- [PIX on Windows documentation](https://devblogs.microsoft.com/pix/documentation/)
- [Microsoft D3D12 Device Removed Extended Data](https://learn.microsoft.com/en-us/windows/win32/direct3d12/use-dred)
- [Microsoft WER user-mode dumps](https://learn.microsoft.com/en-us/windows/win32/wer/collecting-user-mode-dumps)
- [Khronos Vulkan Validation Overview](https://docs.vulkan.org/guide/latest/validation_overview.html)
- [Khronos `VK_EXT_debug_utils`](https://docs.vulkan.org/guide/latest/extensions/VK_EXT_debug_utils.html)
- [Android Performance Analyzer system trace](https://developer.android.com/android-performance-analyzer/run)
- [Android Perfetto](https://developer.android.com/tools/perfetto)
- [Android GPU Inspector](https://developer.android.com/agi)
- [Apple Metal tools](https://developer.apple.com/metal/tools/)
- [Apple Xcode Performance and metrics](https://developer.apple.com/documentation/xcode/performance-and-metrics)
- [OpenTelemetry Specification](https://opentelemetry.io/docs/specs/otel/)

外部資料の更新でMiraikanai固有のSchema、Risk、Budget、Retention、AI Evalを自動変更しない。更新は`ExternalEvidenceRecordV1`、影響分析、fixture、ADR、Reviewを経由する。
