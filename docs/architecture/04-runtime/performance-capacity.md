# Miraikanai Engine Runtime Performance／Capacity Contract

- 文書ID: mirakan.arch.runtime-performance-capacity
- 状態: review
- 正本範囲: 共通CPU／GPU／memory／queue budget、capacity、reservation／loan、backpressure、worker capacity、測定法、regression、Owner横断algorithm optimization candidate qualification、`OptimizationDecisionProjectionV1`、`ProjectScaleEnvelopeV2`、owner-typed workload resolution、非破壊遷移、Qualification
- 非正本範囲: Runtime phase／Simulation Advance／lifetime、ECS storage／query／digest field、Runtime Package binary、Save／Replay record、World cell／coordinate field、LOD policy field、Authoring Document／ChangeSet field、Domain固有budget、外部Tool／SDK／driverの固定値、AI承認、Evidence envelope。各Owner文書を参照する
- 依存: [Runtime ECS](entity-component-system.md)、[Runtime Package](runtime-package.md)、[Persistence／Save](persistence-save.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core architecture](../02-foundation/core-architecture.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Math／Core utilities](../02-foundation/math-core.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Project state](../03-authoring/project-state.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Gameplay programming model](../03-authoring/gameplay-programming-model.md)、[Scheduling／lifetime](scheduling-lifetime.md)、[Debugging／observability／replay](debugging-observability-replay.md)、[World](../06-rendering/world.md)、[LOD](../06-rendering/lod.md)、[Mobile common](../07-platform/mobile-common.md)
- 外部根拠検証日: 2026-07-26

## 1. 結論とauthority

共通budget、capacity envelope、reservation、loan、backpressure、測定法、regression threshold、Scale qualificationは本書だけが決定する。Subsystem Ownerは本書が割り当てたparent budget内の固有配分とquality fallbackを所有し、共通上限を再定義しない。Runtimeのphase／Simulation Advance／lifetimeは[Scheduling／lifetime](scheduling-lifetime.md)だけが所有する。

性能は平均fpsや推定costではなく、同一Source revision、Target Profile、Quality、Toolchain lock、fixture、input trace、process条件で計測する。correctness、Replay、visual／audio tolerance、fault、memory、hitchのいずれかを悪化させて性能合格を作らない。budget不足時はSource意味を黙って削らず、bounded planまたはProject Stateの`state=blocked`と登録済み`blocked_reason_ref`を返す。改善可能な未達は`blocked_reason_ref=optimization_required`とする。

本書は外部Tool、SDK、OS、driver、Libraryのversion、hash、license、取得元を固定しない。Benchmark environmentは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)のexact baseline refを使い、値を本書へ複写しない。

## 2. Budget modelとmeasurement boundary

全allocation、queue reservation、resident resource、worker reservationは`BudgetScopeRefV1`へchargeする。scopeは`process_group | process | parent_domain | child_domain | frame_slot | job | artifact_build`のclosed kind、Target Profile、configuration、owner、soft／hard limit、emergency classを持つ。未登録allocationをUnassignedへ黙って課金して継続しない。

memoryは次を区別する。

- Engine allocator current／peak／allocation count。
- vendor Adapter allocationとmetadata。
- process private working set／commit／resident。
- GPU committed／resident／Platform-reported budget。
- frame／job transient high-waterとupstream fallback。
- borrowed bytes、owner、deadline、return result。

MiBは`2^20` bytesとする。P50／P95／P99.9はwarm-upを除く全sampleを昇順にし、nearest-rank `ceil(p × N)`番目を採る。欠測は0で補わずAvailabilityとBlocking理由を記録する。各runの値、run集合の選択規則、Target／environment ref、instrumentation tierをEvidenceへ含める。

CPU critical pathはSimulation Advanceの`T00_BoundaryApply`開始から、そのadvanceのstateを含む最初のrender submission呼出しがreturnするまでとする。catch-upで中間advanceのsnapshotが単独submitされない場合は、そのstateを包含する後続snapshotの最初のsubmissionで測定する。R00～R70を実行しないheadless Targetと、`SurfaceUnavailable`／`Inactive`／`Suspended`区間は本測定の対象外とする。対象runで当該stateを含むsubmissionが一度も発生しなければhard failureである。GPU frameは当該snapshotの最初のGPU timestampから最終composite timestampまでとし、display sync待機を含めない。real frameとgenerated／displayed frameを分離する。

## 3. CPU memory envelope

`target.windows.desktop`（`profile_version` 1）のactive GameHost共通soft budgetは2,048 MiBとする。Target IDは[Product Plan](../00-product/product-plan.md)のTarget Profile registryを正本とし、IDへ`v1`等のversionを埋め込まない。

| Parent domain | Child envelope | MiB |
|---|---|---:|
| Core World／Save | ECS・World 128、snapshot／bridge 32、Save／Replay 64、stack／system 32 | 256 |
| Rendering CPU／upload | render extract 48、VFX CPU 32、shader／material metadata 32、descriptor metadata 16、upload staging 96、reserve 32 | 256 |
| Physics／Navigation／Animation | Physics 96、Navigation 64、Animation 64、reserve 32 | 256 |
| Unassigned headroom | 通常allocation、cache、loanへ使用禁止 | 64 |
| Debug diagnostics | ingress、Store ring、Index、fault capture専用 | 64 |
| Audio | decoded／stream ring 96、voice／control 16、reserve 16 | 128 |
| Asset streaming | compressed cache 256、decode／transcode 256、hot cache 192、dependency metadata 32、reserve 32 | 768 |
| Frame／Job transient | Frame 32、Render frame slots最大48、Job scratch 40、reserve 8 | 128 |
| Emergency | diagnostic、journal、controlled shutdown専用 | 128 |
| **合計** |  | **2,048** |

`target.windows.editor`（`profile_version` 1）process groupの共通soft budgetは4,096 MiBで、active Play Runtime 2,048、Authoring World／undo／revision 512、Asset import／cache client 512、UI／preview／thumbnail 384、AI bridge／schema／diagnostics 256、Editor reserve 384 MiBに分ける。Playしていない間もPlay partitionを長寿命Authoring cacheへ転用しない。Play準備前に必ずevictできる一時bufferだけ最大512 MiBのmode-exclusive loanを利用でき、返却不能ならPlay開始を拒否する。

Runtime Emergency 128 MiBを除く1,920 MiBを通常global scopeとする。80／90／100%閾値の分母は1,920 MiBである。Unassigned headroom 64 MiBはvendor超過・計測誤差吸収域、Debug diagnostics 64 MiBは§5.1のDebug専用reservationであり、どちらも一般Domainへ配分しない。したがって一般Domainへ配分可能な上限は従来どおり1,792 MiBである。

- 80%: eviction後に戻すtarget。
- 90%: warning、nonessential cacheとPresentation qualityの縮退を開始。
- 100%: Domain cap、nonessential allocation拒否。
- Emergency、Unassigned headroom、未使用Debug reservationを通常処理、cache、quality維持へ貸さない。
- 未使用Parent budgetのloanは一load jobまたは最大120 render frameの先着期限までとする。
- 借用bytesを貸出元、借用先、global totalへ同時記録し、global scopeを超えない。
- deadline超過loanはconfiguration defectとしてqualificationを失敗させる。
- 必須allocationはeviction後に一度だけretryし、再失敗したauthoritative transactionをpublishしない。

`target.windows.desktop`のframes-in-flightは3とし、Render frame slotsはslotあたり16 MiBの3面で最大48 MiBを消費する。他Targetのframes-in-flightは各Platform Ownerが定め（mobileは[Mobile common](../07-platform/mobile-common.md)を参照）、本書のmeasurement interfaceへ投影する。Frame、render-frame、job scratchはscope完了後に一括resetする。GPU consumerを持つframe slotは全last-use submission完了までresetしない。hot pathのupstream fallbackを一般heapへ流さず、Development／Profileでは発生frameをfailureとして記録する。[Memory／Pointers](../02-foundation/memory-pointers.md)がallocator／pointer semanticsを所有する。

## 4. GPU memory envelope

`target.windows.desktop`のGPU project budgetは`min(5,632 MiB, 0.80 × Platform reported budget)`とする。他TargetはPlatform Ownerが定めるaggregate working-set capを本書のmeasurement interfaceへ投影する。

| Domain | MiB |
|---|---:|
| Texture | 3,072 |
| Geometry | 1,024 |
| Render target／transient | 1,024 |
| Shader／descriptor | 256 |
| Emergency reserve | 256 |
| **合計** | **5,632** |

Platform budgetが小さい場合はcritical resourceとEmergencyを先に確保し、Presentation-only texture mip、streaming distance、shadow、transient resolutionの順にDomain Ownerの承認済みfallbackを適用する。owner-typed authoritative state／event outcome、registered collision evidence、simulation timingをGPU pressureから変更しない。

resource作成はcommitted／resident bytes、metadata、upload、main／render stallへchargeする。nonessential allocationはPlatform budget内でだけ許可する。全allocation走査は明示capture時だけ行い、毎frameはaggregate counterを使う。resource作成完了を即時live化せず、[Scheduling／lifetime](scheduling-lifetime.md)のactivation boundaryへ提出する。

aliasはRender Graphがlifetime非重複、heap compatibility、barrier、完全初期化を証明した場合だけ許可する。defragmentationはEditorまたはloading boundaryに限定し、一pass最大64 MiBまたは64 allocationの先着上限とする。copy、submission completion、handle swap、old allocation retireを一transactionで行い、失敗時はsource allocationを維持する。

device loss時のcapture／recovery順はScheduling、Renderer、Platform、Debugging Ownerへ委譲する。未完了submissionが進むと仮定した無期限waitをbudget回復手段にしない。

## 5. Queue capacityとbackpressure

次は共通C1 queue／buffer capacity profileのhard reservationであり、Targetを理由に暗黙縮小しない。Projectが変更する場合はmemory envelope、stress、Replay、Domain qualificationを再承認する。Runtime contract固有のdeterministic上限（[Scheduling／lifetime](scheduling-lifetime.md) §4.1のGameplay Timer active／fire上限等）は各Owner文書が所有し、本表へ複写しない。その変更も本節と同じ再承認を必要とする。本表はqueue storage上限であり、同時resident／visible Entity、owner-typed authoritative／presentation instance、lifecycle burstの製品capacityを表さず、それらの未校正値を本表から逆算しない。

| Queue／buffer | faces | Entry capacity | Payload arena | max payload／entry | charge | critical reserve |
|---|---:|---:|---:|---:|---|---:|
| Structural command | 2 | 16,384 / simulation step | 2 MiB | 16 KiB | ECS／World | 0 |
| Simulation command total | 2 | 65,536 / simulation step | 4 MiB | 16 KiB | ECS／World | 0 |
| Gameplay event total | 2 | 32,768 / simulation step | 4 MiB | 4 KiB | ECS／World | 0 |
| Normalized Physics event | 2 | 65,536 / simulation step | 4 MiB | 256 B | Physics | 0 |
| Navigation request／result | 1 | each 4,096 / simulation step | each 8 MiB | 64 KiB | Navigation | 0 |
| Presentation event | 2 | 32,768 / simulation step | 4 MiB | 8 KiB | snapshot／bridge | 1,024 entries |
| Audio command | 1 | 8,192 entries | 1 MiB | 4 KiB | Audio | 512 entries |
| Audio completion | 1 | 4,096 entries | 256 KiB | 64 B | Audio | 256 entries |
| Async completion | 1 | 8,192 entries | 512 KiB | 256 B | 所属Domain | 256 entries |
| Asset activation | 2 | 1,024 / boundary | 1 MiB | 4 KiB | dependency metadata | 64 entries |

entry headerは32 bytes／entryとする。起動時commitは`Σ faces × (Entry capacity × 32 B + Payload arena)`で導出し、header 13.9375 MiBとarena 55.75 MiBの合計69.6875 MiBを所属Domainへchargeする。`faces = 2`はcurrent／nextの二面buffer、`faces = 1`は単面またはbounded ringであり、Navigationはrequest／resultの二queueを各一面持つ。Gameplay event totalは、active owner schema registryに登録されたtyped authoritative Game Eventの配送queueである。[Scheduling／lifetime](scheduling-lifetime.md) §4.1のtimer deadline fire（1 Simulation Advance最大4,096件）も登録済みeventとしてこの内数に含める。Async completionは`IoCompletion`／`AssetWorker` latch sourceのcompletionを運ぶ。entry数、個別payload、arena bytesのいずれかが先に上限へ達した時点でoverflowとする。

Navigationのobstacle input受領からNavigation artifact version activationまでの反映latency bound（Simulation Advance上限）は本書所有のcapacity項目である。[Navigation](../05-simulation/navigation.md) §3は値の所有を本書へ委譲しており、初期boundは未固定とし、§8のmeasurement／promotion手続きで確定するまで当該boundを前提とするqualificationを合格にしない。

critical bitとpriorityはregistered schema／Capability manifestだけが設定し、Project payload、AI、GameplayDefinitionから昇格できない。criticalはcontrolled shutdown、resource release／retire、generation rollback等のEngine-owned operationに限定する。critical reserveをnoncritical producerへ貸さない。

| class | overflow／pressure behavior |
|---|---|
| Structural／Simulation／Gameplay event／Physics authoritative | current transactionをpublishせずsession fault |
| Navigation request | new requestを`QueueFull`で拒否しaccepted resultを失わない |
| Async completion | new requestの発行を`QueueFull`で拒否しaccepted completionを失わない |
| Presentation | lowest priorityからdropしgap／countを記録 |
| Audio | critical stop／releaseを維持しlow-priority playをdrop |
| Asset activation | next boundaryへ延期しclosureを部分activateしない |
| Debug telemetry | [Debugging／observability／replay](debugging-observability-replay.md)のgap／capture stop semanticsを使いGameplayへ影響させない |

Presentation／Audioはpriority昇順、同priorityはScheduling Ownerのcanonical message keyで後発からdropする。Asset generationは古いready generationを優先する。Developmentでは80%超をwarning、95%超をcapacity gate failureとする。Shippingでauthoritative recordを黙ってdropしない。

### 5.1 Debug capacity request

`DebugCapacityRequestV1`は`request_id`、Target Profile ref、instrumentation tier、channel set hash、expected duration、ingress entry／arena bytes、Store ring bytes、disk retention bytes、maximum write throughput、critical-path P95 overhead、process CPU overhead、reservation source、backpressure policy refを持つ。本表はMiraikanai C1のProject policyであり、外部SDKの既定値ではない。

| tier | ingress entries | payload arena | Store ring | disk retention / session | max write | critical-path P95増分 | process CPU増分 |
|---|---:|---:|---:|---:|---:|---:|---:|
| `fault_minimal` | 2,048 | 0.5 MiB | 8 MiB | 64 MiB | 4 MiB/s | 0.10 ms | 1% |
| `baseline` | 8,192 | 2 MiB | 32 MiB | 512 MiB | 16 MiB/s | 0.25 ms | 2% |
| `interactive` | 32,768 | 8 MiB | 64 MiB | 2,048 MiB | 64 MiB/s | 0.50 ms | 5% |
| `capture` | 65,536 | 16 MiB | 64 MiB | 8,192 MiB | 256 MiB/s | 1.00 ms | 10% |

ingress headerは32 bytes／entryで、arena、Store ring、Index working setとともにDebug diagnosticsへchargeする。`fault_minimal`と`baseline`は64 MiB reservation内で起動し、`interactive`と`capture`は開始前に未使用のnoncritical Parent envelopeからSession期限付きmode-exclusive loanを明示予約する。loanを確保できなければtier開始を拒否し、EmergencyまたはUnassigned headroomを使わない。CPU増分は同一fixture・同一Buildのtier-off比較で測り、process CPUはlogical processor数で正規化したhost process使用率差とする。entry、arena、ring、disk、throughput、CPUのいずれかが先に上限へ達した場合も、[Debugging／observability／replay](debugging-observability-replay.md)のgap／capture stop semanticsを使い、authoritative Runtimeを遅延させない。

Backpressure actionは`reject | defer | drop_presentation | degrade_presentation | stop_capture | fault_transaction`のclosed setとし、owner、trigger、hysteresis、maximum delay、fallback、counter、recovery predicateを持つ。Source meaningやfidelity floorを変更するactionを自動選択しない。

## 6. Worker、I/O、job capacity

GameHostはlatency-sensitive roleと一つのshared worker poolを持ち、Libraryごとの独立poolを作らない。Target Adapterはavailable performance processor countを`P`として投影し、desktopは`clamp(P - 4, 1, 12)`、mobileはonline processor count `L`から`clamp(L - 4, 1, 8)`をshared worker capacityの初期値とする。hardware topology取得不能時のfallbackとaffinity／QoSはPlatform Ownerが決定し、Gameplay結果を変えない。

Physics worker countはshared pool内の最大同時slotであり別thread数ではない。required slotを提供できなければPlay準備またはReplayを拒否する。worker profileはPlay開始時に固定し、Replay environmentへ記録する。

job priorityは`critical_simulation | critical_render | streaming | background_tool`とする。長いstreaming／tool jobは最大0.50 msのcooperative sliceまたは明示yield pointを持つ。file／network待機をworkerでblockせずI/O completionへ渡し、completion roleで0.25 msを超えるCPU処理を行わない。thermal／memory pressure時は新しいnoncritical jobを止め、active critical worker countをauthoritative run途中で変えない。

Tool child processのexact executable／version／hashは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、Build work item／cancellation／artifact semanticsは[Core architecture](../02-foundation/core-architecture.md)と[Asset lifecycle](../03-authoring/asset-lifecycle.md)が所有する。本書はtool process groupのaggregate hard commit 4,096 MiB、heavy job同時予約合計がその値を超える開始拒否、tree termination時のstaging破棄だけを共通capacityとして所有する。

## 7. Frame、latency、Subsystem budget

| Target cadence | CPU／GPU P95 soft | hard deadline | endurance deadline rule |
|---|---:|---:|---|
| 60 fps | 14.00 ms | 16.67 ms | 10分で50.00 ms超0件 |
| mobile 30 fps | 28.00 ms | 33.33 ms | deadline miss 1%以下 |
| optional 120 fps | 7.00 ms | 8.33 ms | generated frameをsampleへ含めない |

simulation cadence自体は[Scheduling／lifetime](scheduling-lifetime.md)を参照し、本書で再定義しない。60 fpsはhard deadlineを常用targetにせず、2.67 ms以上のheadroomを確保する。

| CPU critical-path group | P95 soft cap |
|---|---:|
| boundary＋input latch | 0.50 ms |
| async integration | 0.25 ms |
| Gameplay Logic | 1.50 ms |
| Motion＋Animation | 1.50 ms |
| Physics | 2.50 ms |
| Navigation result integration | 0.25 ms |
| PostPhysics＋Presentation | 0.75 ms |
| Snapshot＋Replay | 0.75 ms |
| Simulation thread subtotal | 8.00 ms |
| Render extract＋Graph＋submit | 4.00 ms |
| scheduling／sync／OS jitter headroom | 2.00 ms |
| **Critical-path total** | **14.00 ms** |

上のCPU critical-path group表は`target.windows.desktop`の60 fpsかつreference `fixed 60/1` Simulation Cadenceだけを対象とする。mobile 30 fpsの同reference測定（1 render frameに最大2 Simulation Advances）のgroup内訳は各Platform Ownerが定め、本書のmeasurement interfaceへ投影する。この比率をCore schemaまたは別Cadence kindの規則へ一般化しない。

| GPU pass group | P95 soft cap |
|---|---:|
| Shadow | 2.00 ms |
| Visibility／Depth | 1.50 ms |
| Opaque／Material／Lighting | 3.00 ms |
| Transparent／VFX | 1.50 ms |
| Atmosphere／Environment | 2.00 ms |
| Post／Exposure | 1.00 ms |
| UI／Composite | 0.50 ms |
| Copy／queue sync | 0.50 ms |
| Headroom | 2.00 ms |
| **Total** | **14.00 ms** |

Subsystem固有pass／Domain budgetは各Ownerが上表の内数として割り当てる。未使用pass budgetを無関係な機能へ無制限に転用しない。

`PostProcessBudgetEnvelopeV1`は本書がOwnerとして公開するread-only／revisioned projectionであり、最低fieldとしてrevision、Target Profile ref、Post／Exposure pass groupの内数としてのGPU P95 cap、Post Process用persistent／transient byte上限を持つ。[Post Processing](../06-rendering/post-processing.md) §7のresolver入力はこのprojectionを消費し、field一覧を複写せず書き戻さない。Temporal reconstruction、frame generation、ray tracing等の追加経路はbase real frame、headroom、memory、visual、fault Gateを満たすTarget限定profileとし、generated frameをreal fpsへ加算して合格を作らない。

`RendererBudgetEnvelopeV1`と`LightingBudgetEnvelopeV1`も本書がOwnerとして公開するread-only／revisioned projectionである。

```text
RendererBudgetEnvelopeV1
  revision: positive uint64
  target_profile_ref:
    exact {target_profile_id, target_profile_version, target_profile_content_hash}
  quality_profile_ref:
    exact {quality_profile_id, quality_profile_version, quality_profile_content_hash}
  view_family_id: StableId
  renderer_profile_ref:
    exact {profile_id, profile_version, profile_content_hash}
  gpu_p95_cap_us: finite nonnegative microseconds
  gpu_p95_reserved_us: finite nonnegative microseconds
  gpu_p95_remaining_us: derived cap - reserved
  transient_byte_cap: uint64
  transient_byte_reserved: uint64
  transient_byte_remaining: derived cap - reserved
  persistent_byte_cap: uint64
  persistent_byte_reserved: uint64
  persistent_byte_remaining: derived cap - reserved
  content_hash: SHA-256

LightingBudgetEnvelopeV1
  revision: positive uint64
  target_profile_ref:
    exact {target_profile_id, target_profile_version, target_profile_content_hash}
  world_level_scope_hash: SHA-256
  view_family_id: StableId
  lighting_gpu_p95_cap_us: finite nonnegative microseconds
  lighting_gpu_p95_reserved_us: finite nonnegative microseconds
  lighting_gpu_p95_remaining_us: derived cap - reserved
  shadow_gpu_p95_cap_us: finite nonnegative microseconds
  shadow_gpu_p95_reserved_us: finite nonnegative microseconds
  shadow_gpu_p95_remaining_us: derived cap - reserved
  transient_byte_cap: uint64
  transient_byte_reserved: uint64
  transient_byte_remaining: derived cap - reserved
  persistent_byte_cap: uint64
  persistent_byte_reserved: uint64
  persistent_byte_remaining: derived cap - reserved
  content_hash: SHA-256
```

全remaining値は同じrevisionの`cap - reserved`から再計算し、負値、別Target／scope／View Familyの混在、stale reservationを拒否する。`content_hash`はそれぞれASCII `MIRAKAN_RENDERER_BUDGET_ENVELOPE_V1`、`MIRAKAN_LIGHTING_BUDGET_ENVELOPE_V1`と自己Fieldを除くlength-framed canonical bytesをSHA-256する。[Render Graph](../06-rendering/render-graph.md)のOutline resolverと[Lighting](../06-rendering/lighting.md)のIntent Resolverはこのprojectionをread-only inputとして消費し、field一覧を複写または書き戻さない。

共通operation budgetはAudio callback P99 0.25 ms以下かつhard 1.00 ms未満、main／render thread activation slice soft 0.50 msかつhard 1.00 ms以下、warm-cache package start P95 soft 5.00 sかつhard 8.00 s、Scene reload P95 soft 2.00 sかつhard 3.00 sとする。

## 8. Measurement、regression、promotion

測定loopは次を必須とする。

```text
Contract / Budget
  -> Reference implementation
  -> Deterministic fixture
  -> Target trace
  -> Bottleneck selection
  -> Candidate optimization
  -> Before / After + correctness / visual / fault comparison
  -> Governance Evidence
  -> promote | blocked(optimization_required) | reject
```

最低metric familyはframe／latency、hitch、Runtime CPU、memory、loading、GPU／streaming、queue／backpressure、correctnessである。Hitchはdeadline、2倍deadline、50 ms超を数え、shader／pipeline、Asset I/O、allocation／page fault、job／queue wait、driver／device、unknownへ分類する。unknownを除外しない。

reference measurementはdeterministic warm-up後、同じ入力traceの120秒runを5回実行し、各run P95のmedianを採り、同じbuildで10分soakを追加する。Scale qualificationは10分runを3回、Production enduranceは2時間runを追加する。Mobile／Platform固有Targetは実機baselineを使い、Emulator／Simulatorで代用しない。

Windows reference environmentのC0 Reference hardware構成は次の2構成とし、本表が正本である。A／BのGPU構成差だけを比較できるよう、CPU、memory、storage、firmware、power planはA／Bで同一SKU・同一設定にする。消費文書（[Editor UI Framework](../03-authoring/editor-ui-framework.md)、[Windows Platform](../07-platform/windows.md)等）は構成値を複写しない。

| 項目 | 構成A | 構成B |
|---|---|---|
| GPU | NVIDIA GeForce RTX 3060 | AMD Radeon RX 6600 |
| CPU世代／コア数 | Intel Core i5-12400、6C／12T | Intel Core i5-12400、6C／12T |
| RAM容量 | 32 GiB DDR4-3200、2×16 GiB dual channel | 32 GiB DDR4-3200、2×16 GiB dual channel |
| Storage | 1 TB NVMe PCIe 3.0 ×4 | 1 TB NVMe PCIe 3.0 ×4 |
| Display test topology | 2560×1440・100% single、3840×2160・200% single、両者を跨ぐmulti-monitor move／hot-unplug | 同左 |
| Editor visual baseline | Dark、standard density、`editor_ui_scale=1.00`、Production Workspace／Reference 01 | 同左 |
| GPU driver固定方針 | official WHQL packageをclean installし、package version／SHA-256／Authenticode publisher／install mode／power planをToolchain lockへ記録。自動更新を無効化し、変更時はreference profile versionを更新して再計測 | 同左 |

```text
ReferenceHardwareProfileV1
  profile_id
  profile_version
  qualification_state = characterization_only | locked
  cpu_sku_stepping_ref
  motherboard_bios_ref
  memory_configuration_ref
  storage_firmware_ref
  monitor_topology_edid_ref
  os_image_ref
  gpu_device_ref
  gpu_driver_package_ref
  power_plan_ref
  toolchain_profile_ref
  profile_content_hash
```

`profile_content_hash`はASCII `MIRAKAN_REFERENCE_HARDWARE_PROFILE_V1`と、自己hash Fieldだけを除く全Fieldを宣言順、presence／count／length framing付きcanonical bytesへ直列化してSHA-256する。Refはprofile ID、version、content hashの三つを必須とし、logical IDだけ、GPU名、table row、latest versionを受理しない。

構成A／Bのlogical identityはそれぞれ`profile.performance.windows-reference-a@1`、`profile.performance.windows-reference-b@1`とする。これは表示名やGPU family aliasではなく、CPU stepping、motherboard／BIOS、memory part／timing、storage firmware、monitor EDID、OS image、GPU／driver package、power plan、Toolchain profile refを持つ完成`ReferenceHardwareProfileV1`のID／versionである。現在のC0表から未確定Fieldを推測してcontent hashを作らず、全Fieldとhashがlockされるまでは両Profileを`characterization_only`とし、Reference Fixture materializationをblockして関連Capabilityを`not_activated`に保つ。完成Profileが存在するがCapability対象外の場合のManifest applicability `prohibited`と、Profile未完成を混同しない。UI／Runtime consumerは構成値を複写せず、この二refと完成hashだけを使う。

Reference Fixtureのouter `EditorReferenceEnvironmentProfileV1.reference_hardware_profile_ref`はA／Bの完成Profile一件へ解決し、同Profileの`os_image_ref`は[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)がHost UI fontのresolved fileを得たCI OS imageとexact一致しなければならない。GPU driver package、monitor topology／EDID、power planはこのPerformance profileだけが所有し、Toolchainの`style_font_generation`またはfont／icon artifact hashへ複写しない。outer Environment Profile、Toolchain OS image、Performance profileの三者が一Fieldでも異なる場合は`infrastructure_error`としてcapture／comparisonを開始しない。これによりfont assetの失効とhardware／display driftを別generationとして保持し、相互hash cycleを作らない。

### 8.1 Editor Reference 01 performance profile

[Editor UI Framework §15.8.5](../03-authoring/editor-ui-framework.md#1585-comparison-profile-registry)の`comparison.editor.performance.absolute-and-regression@1`が消費するPerformance Owner artifactを次で閉じる。これはReference 01専用の計測定義であり、一般Game Runtime、Build、Importのworkloadへ流用しない。

```text
EditorReferencePerformanceWorkloadProfileV1
  workload_profile_id
  workload_profile_version
  fixture_manifest_ref
  fixture_manifest_hash
  hardware_profile_refs[2]
  hardware_profile_hashes[2]
  setup_barrier_ref
  setup_barrier_hash
  operation_schedule_ref
  operation_schedule_hash
  auxiliary_corpus_ref optional
  auxiliary_corpus_hash optional
  expected_sample_family_set_ref
  expected_sample_family_set_hash
  workload_profile_content_hash

EditorReferencePerformanceSamplePolicyV1
  sample_policy_id
  sample_policy_version
  fresh_process_run_count
  warmup_cycle_count
  measured_duration_ns_per_run
  scheduled_cycle_period_ns
  percentile_method
  required_soak_duration_ns
  sample_policy_content_hash

EditorReferencePerformanceThresholdSetV1
  threshold_set_id
  threshold_set_version
  workload_profile_ref
  workload_profile_hash
  thresholds[1..32]:
    metric_ref
    statistic = p95 | peak | count
    relation = less_than_or_equal
    threshold_value
    unit_ref
  relative_regression_ppm_max
  threshold_set_content_hash

EditorReferencePerformanceAggregationV1
  aggregation_id
  aggregation_version
  run_percentile = p95_nearest_rank
  across_run_aggregation = median_odd_exact
  hardware_aggregation = independent_all_pass
  missing_sample_disposition = infrastructure_error
  aggregation_content_hash

EditorReferencePerformanceHarnessQualificationV1
  qualification_id = qualification.editor.performance-harness
  qualification_version
  comparator_parameter_ref
  comparator_parameter_hash
  hardware_profile_refs[2]
  hardware_profile_hashes[2]
  workload_profile_refs[3]
  workload_profile_hashes[3]
  sample_policy_ref
  sample_policy_hash
  aggregation_ref
  aggregation_hash
  absolute_threshold_set_refs[3]
  absolute_threshold_set_hashes[3]
  soak_profile_ref
  soak_profile_hash
  characterization_result_refs[6]
  characterization_result_hashes[6]
  characterization_soak_result_refs[6]
  characterization_soak_result_hashes[6]
  verification_receipt_ref
  verification_receipt_hash
  result = pass | fail | infrastructure_error
  qualification_content_hash

EditorReferencePerformanceQualificationV1
  qualification_id
  qualification_version
  comparison_profile_ref
  comparison_profile_hash
  hardware_profile_refs[2]
  hardware_profile_hashes[2]
  workload_result_refs[6]
  workload_result_hashes[6]
  soak_result_refs[6]
  soak_result_hashes[6]
  approved_baseline_refs[6]
  approved_baseline_hashes[6]
  verification_receipt_ref
  verification_receipt_hash
  result = pass | fail | infrastructure_error
  qualification_content_hash
```

`sample.editor.ref01.five-by-120s@1`はfresh process五run、各runのtyped setup barrier後にwarm-up十cycleを捨て、120秒を測定し、cycle periodをreal monotonic clockのexact 2秒とする。operationが次periodまでにterminal barrierへ到達しない場合はsampleを欠測へせず、そのoperation latencyを記録した上でrunをfailする。Performance計測はvirtual clockから値を作らず、virtual clockはVisual／Motion stateを固定するためだけに使う。各fresh processは同じCandidate、Contract set、Toolchain lock、hardware profile、OS imageから開始し、run間cacheまたはprocessを流用しない。

#### Windows high-resolution measurement clock

`target.windows.editor`のPerformance Harnessはinterval sourceをWin32 `QueryPerformanceCounter`（QPC）へ固定し、process初期化時に`QueryPerformanceFrequency`を一回だけ取得・cacheする。各measured operationは同じdesignated Harness threadでbarrierの受信後にstart／endを採取し、Evidenceに`clock_source=win32_qpc`、`qpc_frequency_hz`、`start_qpc_tick`、`end_qpc_tick`、checked `delta_qpc_tick`を必須保存する。durationは`{delta_qpc_tick, qpc_frequency_hz}`のexact rationalとして保持し、threshold／aggregate比較はchecked integer演算で行う。float、丸め済みms、wall clock、message timestamp、animation／virtual clockをthreshold入力へ使わない。raw tick値はsystem bootをまたいで比較せず、同じratioへ正規化したdurationだけを比較する。

barrierの順序はproducer threadがHarness threadへsignalし、Harnessが自身のQPC採取点で確定する。異threadで採ったQPC値の±1 tick以内の前後関係からbarrier順序を推論しない。`QueryPerformanceCounter`／`QueryPerformanceFrequency`の失敗、frequency=0、`end_qpc_tick <= start_qpc_tick`、checked subtraction／rational comparisonのoverflow、clock evidence mismatchは`infrastructure_error`であり、0、直前sample、別clockへ置換しない。`RDTSC`／`RDTSCP`、thread-affinity pinning、他machineのtimestampをこの計測または順序決定に使わない。

この規則はMicrosoftの[QueryPerformanceCounter](https://learn.microsoft.com/windows/win32/api/profileapi/nf-profileapi-queryperformancecounter)、[QueryPerformanceFrequency](https://learn.microsoft.com/windows/win32/api/profileapi/nf-profileapi-queryperformancefrequency)、[Acquiring high-resolution time stamps](https://learn.microsoft.com/windows/win32/sysinfo/acquiring-high-resolution-time-stamps)に従う。QPC frequencyがboot中固定であること、変換を遅らせること、異thread timestampの微小な順序不確実性を、このEvidence contractで明示的に吸収する。

`aggregate.editor.ref01.median-of-five-p95@1`は各runのwarm-up後sampleを昇順にし、P95をnearest-rank `ceil(0.95 × N)`で選ぶ。五runのP95を昇順にした三番目を最終値とする。A／Bのsampleは別集合で同じ処理を行い、平均、best hardware、closest baselineで統合しない。五run未満、required metric欠測、NaN、infinite、negative duration、counter overflow、hardware／environment driftは`infrastructure_error`であり、0または直前値で補わない。

`qualification.editor.performance-harness@1`はComparison Profileをback-referenceせず、`parameter.editor.performance.absolute-and-regression@1`、A／B hardware、三Workload、sample、aggregation、三absolute Threshold Set、soakへ直接閉じる。A／B×三workloadのexact六characterization runと六soakがabsolute threshold、sample completeness、deterministic aggregation、run間repeatabilityをpassする場合だけ`result=pass`とする。relative regressionは未承認Baselineが存在しないためこのQualificationでは評価せず、0%差またはpassとして捏造しない。このbaseline非依存Qualificationが完成した後にだけComparison Profile RegistryとInitial Baseline Execution Definitionをmaterializeできる。

Exact workloadは次の三件である。

| workload profile | setup後の120秒schedule | auxiliary corpus |
|---|---|---|
| `workload.editor.ref01.open@1` | 下Dockのactive tabをProblems↔Consoleへ2秒ごとに切替、60 measured cycle。各Panel activationからterminal layout／present barrierまでをsampleにし、60回後はProblems activeへ戻す。Asset load時間を含めない | なし |
| `workload.editor.ref01.steady-state@1` | B選択→A選択、Inspector position.x preview `12.500→13.000 m`→cancel、Dock preview→cancelを2秒ごとに一cycle、60 measured cycle。各cycle末にrevision 42、root、selection／focus A、Workspace不変へ戻す | なし |
| `workload.editor.ref01.capacity@1` | 偶数cycleは100万Outliner corpusのfilter clear→exact query→initial result、奇数cycleは10万Asset corpusのsearch clear→exact query→initial result。各30 measured sample | `corpus.editor.ref01.capacity@1` |

`corpus.editor.ref01.capacity@1`はfixture seed `2026072501`から事前materializeするimmutable artifactで、exact 1,000,000 Outliner projection recordと100,000 Asset index recordを持つ。record ID、parent、display name、type、tag、search token、expected match setをartifactに保存し、run時の乱数生成、User Project、filesystem scan、network、thumbnail decodeを含めない。Outliner queryはexact 10,000 match、Asset queryはexact 1,000 matchを返し、初期result barrierは最初のvirtualized viewport＋total count＋continuation tokenが同じgenerationでpublishされた時点とする。corpus ref/hash、query ref/hash、expected match-set ref/hashの一件でも未解決ならcapacity workloadを開始しない。

Absolute threshold setは既存UI budgetを次のexact profileへ束縛する。

| threshold set | required metric／final statistic | threshold |
|---|---|---:|
| `threshold.editor.ref01.open@1` | Workspace／Panel open latency P95 | `<= 200.00 ms` |
| 同上 | Full Window layout＋packet CPU P95 | `<= 8.00 ms` |
| 同上 | UI GPU pass P95 | `<= 1.00 ms` |
| 同上 | UI thread blocking task `> 50.00 ms` count | `0` |
| `threshold.editor.ref01.steady-state@1` | Idle UI frame P95 | `<= 16.67 ms` |
| 同上 | Input→visual response P95 | `<= 50.00 ms` |
| 同上 | Inspector continuous preview frame P95 | `<= 16.67 ms` |
| 同上 | dirty subtree event＋style＋layout＋packet CPU P95 | `<= 4.00 ms` |
| 同上 | UI GPU pass P95 | `<= 1.00 ms` |
| 同上 | UI thread blocking task `> 50.00 ms` count | `0` |
| `threshold.editor.ref01.capacity@1` | 100万Entity filtered Outliner initial result P95 | `<= 500.00 ms` |
| 同上 | 10万Asset search initial result P95 | `<= 200.00 ms` |
| 同上 | EditorHost＋同時に一つのGameHost aggregate memory peak | `<= target.windows.editor process-group soft budget` |
| 同上 | UI thread blocking task `> 50.00 ms` count | `0` |

各Threshold Setの`relative_regression_ppm_max=50,000`とし、同じhardware／workload／metricのapproved baselineに対してlatency、memory peak、allocation countのいずれも5%超悪化を許さない。絶対値とrelative guardはANDであり、baselineが遅いことを絶対値違反の免除にせず、絶対値内であっても5%超regressionをpassにしない。count threshold `0`はrelative計算をせず一件でfailする。

五runとは別に、A／Bごとにfresh process一件でwarm-up十cycle後に開始する`soak.editor.ref01.ten-minute@1`は同じworkload scheduleを10分継続し、連続する各120秒windowへ同じabsolute thresholdを適用する。process crash、device／provider generation loss、required metric gap、process-group soft budget超過のいずれか一件でfailし、soak sampleを五runのP95へ混ぜない。Performance passはperformance oracleだけの結果であり、Project root、Semantic、Visual、UIA、Commandのcorrectness passを内包または代用しない。

`qualification.editor.performance-reference@1`はA／B×三workloadのexact六workload result、exact六soak result、exact六approved baselineを持ち、全absolute thresholdとrelative regressionがpass、同じCandidate／Contract／Toolchain、current `VerificationReceiptV1`に閉じる場合だけpassにする。六`approved_baseline_refs/hashes`は[Editor Workspace UX §6.9.1](../03-authoring/editor-workspace-ux.md#691-fixtureeditorreference-011-concrete-manifest)のG00 open A／B、G01 steady-state A／B、G08 capacity A／Bの六Performance coverage entryと一対一に対応し、[Editor UI Framework §15.8.6](../03-authoring/editor-ui-framework.md#1586-baseline-changeevidence-bundle署名)のcurrent Baseline Registry Publicationにあるexpected subject ref/hashだけを受理する。手入力の数値、直前run、threshold file、`latest` artifactをapproved baselineへ読み替えない。三Workload Profile、capacity corpus、setup／terminal barrier、sample、aggregation、三Threshold Set、soak、A／B approved baselineの全ref/hashが同じContract setで解決するまで通常Performance resultをpassにしない。現時点でA／B hardware profileが`characterization_only`である間はHarness Qualification、Comparison Profile Registry、Initial Baseline Execution Definitionをmaterializeせず、数値表を実測Receiptへ読み替えず、関連Capabilityを`not_activated`に保つ。

上表はC0の構成選択であり、C1 performance qualificationではない。CPU stepping、motherboard／BIOS、memory part／timing、storage firmware、monitor EDID、driver package version／hash、OS imageを[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)の`target.windows.editor` profileへexactに束縛するまで、結果はcharacterizationだけとし比較・regression判定を承認しない。GPU差をCPU、memory、display、driver driftで説明したり補正したりしない。

新しい高速経路は同一fixtureでP95を5%以上かつ0.20 ms以上改善し、memory peak／allocation countを5%超悪化させず、correctness、visual／audio、fault、startup／hitchのhard Gateを悪化させない場合だけ既定候補へ昇格する。baseline比のframe P95またはmemory peak／allocation countが5%超悪化した変更をregressionとする。

Baseline緩和は最適化と別Reviewとし、過去run分布、旧／新値、原因、quality差、下流Capability影響、[AI Security／Approval](../01-governance/ai-security-approval.md)の人間承認を必要とする。Evidence／Receipt構造、Provenance、保持、freshnessは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)だけが決定する。

### 8.2 Pointer／Memory qualification profile

[Memory／Pointers](../02-foundation/memory-pointers.md)は型、所有権、allocation policy、telemetry fieldを所有し、本節はそれを性能Evidenceへ束縛する。`requirement.foundation.memory-pointer-contract`のbaselineは、同じCandidate、Contract Set、Toolchain lock、Target Profile、input traceで次を個別metricとして採取する。

- handle resolveのP50／P95／P99、success／stale／invalid count、retire backlog。
- lease validation latencyとexpired／range／thread-affinity rejection count。
- hot path allocation count、一般upstream fallback試行数、arena high-water／reset、pool reuse／contention／internal fragmentation。
- domainごとのcurrent／peak／budget、allocation／free count、OOMとfailure injectionの結果。

hot pathの一般heap fallbackはcount 0をhard predicateとし、unsupported Targetのhardware counter、sanitizer実行、別hardwareの測定値で代用しない。cache miss、branch miss、memory bandwidth等のhardware counterは、Target Profileが対応し同一fixtureで再現可能な場合だけ診断用の補助Evidenceにする。sanitizer laneはmemory safetyの証拠であり、通常performance baselineまたはShipping throughputの測定値として混在させない。

絶対latency値はReference Hardware Profileがlockedになるまで推測で固定しない。Phase 0では同一Targetのfresh baselineとの差、hot path fallback 0、metric completeness、correctness／negative fixtureの全てを満たすことを要求し、後続のTarget qualificationでProfile別のabsolute／regression thresholdへ束縛する。Memory／PointersにないECS layout、GPU residency、Audio callback固有budgetは各Ownerが本書のmetric familyを参照して決定し、ここで再定義しない。

### 8.3 Runtime ECS data-oriented qualification

本節だけがRuntime ECS data-oriented qualificationのprofile、campaign、mandatory metric、sampling、promotion predicateを所有する。

```text
RuntimeDataOrientedQualificationProfileV1
  profile_version: 1
  target_profile_ref:
    exact {target_profile_id, target_profile_version,
           target_profile_content_hash}
  contract_set_ref: ContractSetRefV1
  toolchain_lock_sha256: SHA-256
  fixture_id: fixture.runtime.ecs-data-oriented-core
  sample_policy: runtime_ecs_warmup_5x120s_median_p95_10m_soak_v1
  chunk_payload_candidates_bytes: [8192, 16384, 32768]
  scenario_ids:
    sequential_motion
    position_only_projection
    lifetime_only_scan
    structural_burst
    archetype_fragmentation
    query_cache_invalidation
  metric_set: runtime_ecs_data_oriented_metrics_v1
  correctness_oracle:
    runtime_ecs_semantic_publication_failure_atomicity_v1
  profile_hash: SHA-256

RuntimeDataOrientedQualificationCampaignV1
  campaign_version: 1
  profile_ref:
    exact {profile_version, target_profile_ref, contract_set_ref,
           toolchain_lock_sha256, profile_hash}
  artifact_candidate_binding_ref: content-addressed ref
  artifact_candidate_binding_sha256: SHA-256
  input_trace_ref: content-addressed ref
  input_trace_sha256: SHA-256
  sample_artifact_ref: content-addressed ref
  sample_artifact_sha256: SHA-256
  correctness_artifact_ref: content-addressed ref
  correctness_artifact_sha256: SHA-256
  result: pass | fail | infrastructure_error
  campaign_hash: SHA-256
```

profileはCandidateを含まない。campaignの`ArtifactCandidateBindingV1` Target member、Contract Set、Toolchainはprofile、sample artifact、correctness artifactとbyte-equalでなければならない。Contract Set rootを先にfinalizeし、それを参照するprofile／campaign instanceをContract Set preimageへ循環挿入しない。hashはMCD canonical encodingを使い、Product signed wrapperのRFC 8785 JCSで代用しない。

```text
profile_hash =
  SHA-256(
    ASCII "MIRAKAN_RUNTIME_DATA_ORIENTED_QUALIFICATION_PROFILE_V1"
    || uint32_be(len(canonical profile bytes excluding profile_hash))
    || canonical profile bytes excluding profile_hash
  )

campaign_hash =
  SHA-256(
    ASCII "MIRAKAN_RUNTIME_DATA_ORIENTED_QUALIFICATION_CAMPAIGN_V1"
    || uint32_be(len(canonical campaign bytes excluding campaign_hash))
    || canonical campaign bytes excluding campaign_hash
  )
```

`runtime_ecs_data_oriented_metrics_v1`は次の35 IDからなるclosed mandatory setである。

| Metric ID | Value kind |
|---|---|
| `metric.runtime.ecs.callback-general-heap-allocation-count` | `uint64_count` |
| `metric.runtime.ecs.callback-upstream-fallback-count` | `uint64_count` |
| `metric.runtime.ecs.memory-reserved-bytes` | `uint64_bytes` |
| `metric.runtime.ecs.memory-committed-bytes` | `uint64_bytes` |
| `metric.runtime.ecs.memory-live-bytes` | `uint64_bytes` |
| `metric.runtime.ecs.memory-peak-bytes` | `uint64_bytes` |
| `metric.runtime.ecs.chunk-count` | `uint64_count` |
| `metric.runtime.ecs.chunk-row-capacity` | `uint64_count` |
| `metric.runtime.ecs.chunk-occupied-rows` | `uint64_count` |
| `metric.runtime.ecs.chunk-unused-payload-bytes` | `uint64_bytes` |
| `metric.runtime.ecs.archetype-count` | `uint64_count` |
| `metric.runtime.ecs.archetype-fragmentation` | `ratio_rational` |
| `metric.runtime.ecs.selected-row-count` | `uint64_count` |
| `metric.runtime.ecs.contiguous-work-unit-count` | `uint64_count` |
| `metric.runtime.ecs.chunk-transition-count` | `uint64_count` |
| `metric.runtime.ecs.exposed-column-bytes` | `uint64_bytes` |
| `metric.runtime.ecs.useful-selected-payload-bytes` | `uint64_bytes` |
| `metric.runtime.ecs.query-cache-hit-count` | `uint64_count` |
| `metric.runtime.ecs.query-cache-miss-count` | `uint64_count` |
| `metric.runtime.ecs.query-cache-rebuild-count` | `uint64_count` |
| `metric.runtime.ecs.query-cache-invalidation-count` | `uint64_count` |
| `metric.runtime.ecs.structural-moved-row-count` | `uint64_count` |
| `metric.runtime.ecs.structural-copy-bytes` | `uint64_bytes` |
| `metric.runtime.ecs.handle-resolve-p50-ns` | `uint64_duration_ns` |
| `metric.runtime.ecs.handle-resolve-p95-ns` | `uint64_duration_ns` |
| `metric.runtime.ecs.handle-resolve-p99-ns` | `uint64_duration_ns` |
| `metric.runtime.ecs.lease-validation-p50-ns` | `uint64_duration_ns` |
| `metric.runtime.ecs.lease-validation-p95-ns` | `uint64_duration_ns` |
| `metric.runtime.ecs.lease-validation-p99-ns` | `uint64_duration_ns` |
| `metric.runtime.ecs.scenario-cpu-p50-ns` | `uint64_duration_ns` |
| `metric.runtime.ecs.scenario-cpu-p95-ns` | `uint64_duration_ns` |
| `metric.runtime.ecs.scenario-cpu-p99-ns` | `uint64_duration_ns` |
| `metric.runtime.ecs.semantic-result-hash` | `sha256` |
| `metric.runtime.ecs.publication-hash` | `sha256` |
| `metric.runtime.ecs.failure-atomicity` | `bool` |

`ratio_rational`はchecked `{numerator:uint64, denominator:uint64}`であり、denominator 0は`infrastructure_error`とする。hardware cache miss、branch miss、bandwidthはsupplementalであり、mandatory IDの代用にしない。

| Scenario | Required observation |
|---|---|
| `sequential_motion` | Position＋Velocity columns traverse contiguous row ranges |
| `position_only_projection` | Velocity、Lifetime、cold metadata are not exposed |
| `lifetime_only_scan` | Lifetime traversal does not require Position／Velocity payload access |
| `structural_burst` | bounded create／destroy／add／remove cost and atomic failure |
| `archetype_fragmentation` | archetype count、chunk occupancy、unused bytes、chunk transitions |
| `query_cache_invalidation` | hit、miss、rebuild、invalidation after structural commit |

Position、Velocity、Lifetime、cold metadataはsynthetic bounded test schemaであり、Gameplay Component authorityではない。

`runtime_ecs_warmup_5x120s_median_p95_10m_soak_v1`は、3 payload × 6 scenarioについて各cellをfresh process 5 runで測る。各runはdeterministic 10 cycleを捨ててexact 120秒を測定し、nearest rank `ceil(0.95 × N)`でrun P95を選び、5値の昇順3番目を採る。さらにpayloadごとにfresh processで10 cycleを捨てた600秒composite soakを一件行い、scenarioを宣言順と固定inputで反復する。したがって90 measured runs、3 soaks、合計12,600秒である。run／soak欠落、process reuse、scenario reorder、sample substitution、NaN／infinite、overflow、identity mismatch、environment driftは`infrastructure_error`とする。

`runtime_ecs_semantic_publication_failure_atomicity_v1`は、callback general-heap allocation count、callback upstream fallback count、chunk boundaryを跨ぐcallback work unit、unselected／undeclared column access、semantic／publication／failure oracle mismatch、stale handle／expired lease／accepted direct structural mutation、required metric missingがすべて0であることを要求する。全predicateを90 runと3 soakのすべてで満たす。initial selected layoutは16,384 bytesであり、8,192／32,768 bytesの結果が良くてもRuntimeを自動切替しない。

Initial qualification invents neither a zero baseline nor an improvement percentage. It passes only when the selected 16 KiB layout completes the full matrix and every hard predicate.

After an initial qualified layout exists, promotion compares baseline and candidate campaigns with byte-equal profile ref、Target、Contract Set、Toolchain、fixture、input trace、sample policy, and correctness oracle. Candidate refs are intentionally different.

promotionは、integrated P95が5%以上かつ0.20 ms以上改善し、peak memoryとallocation countの悪化が5%以内である場合、またはpeak memoryが15%以上改善し、latency、allocation、correctness、fault、load、presentationが既存Gate内である場合だけ許可する。それ以外はcurrent qualified layoutを維持し、Runtimeでauto-switchしない。

```text
diagnostic.performance.ecs-required-metric-missing
MIRAKAN-PERFORMANCE-ECS-REQUIRED-METRIC-MISSING
arguments = campaign_hash, scenario_id, payload_bytes, metric_id
result = qualification failure
```

### 8.4 Algorithm optimization candidate qualification

algorithm／layout／Backend settingの変更は、Ownerが定義するsemantic oracleを一切緩和せず、同じCandidate、Contract Set、Toolchain lock、Target Profile、fixture、input trace、warm-up、sample count、aggregationでbaselineと比較する。predicted／estimated値、別Target、異なるfixture、sanitizer run、直前の`latest` runをpromotion Evidenceへ混在させない。一般高速経路の昇格は§8.1の「P95を5%以上かつ0.20 ms以上改善し、memory peak／allocation countを5%超悪化させず、correctness／fault／startup／hitch hard Gateを悪化させない」を適用し、Owner固有のより厳しいGateは各Owner文書が追加する。

`OptimizationDecisionProjectionV1`はsealed qualification recordから導出するPerformance-owned read-only Projectionであり、AI、Editor、Review toolが候補の状態と根拠を同じ意味で読むためのものとする。Projection自体はSource、selection authority、Receipt、Runtime dispatch tableではない。

```text
OptimizationCandidateRefV1
  artifact_candidate_id:
    exact ArtifactCandidateBindingV1.artifact_candidate_id
  artifact_candidate_binding_sha256:
    SHA-256(JCS(completed ArtifactCandidateBindingV1))

OptimizationEvidenceRefV1
  evidence_role:
    metric_receipt | semantic_receipt | selection_reason
  signed_record_ref: MirakanSignedRecordRefV1

OptimizationDecisionProjectionV1
  schema_version: 1
  projection_id: StableId
  projection_revision: positive uint64
  owner_document_id: StableId
  source_revision_ref:
    exact ArtifactCandidateBindingV1.source_revision_ref
  contract_set_sha256:
    exact ArtifactCandidateBindingV1.contract_set_sha256
  target_profile_ref:
    exact one member of ArtifactCandidateBindingV1.target_profile_refs[]
  toolchain_lock_sha256: SHA-256
  fixture_set_ref: ArtifactRefV1
  input_trace_ref: ArtifactRefV1
  semantic_oracle_ref: ArtifactRefV1
  baseline_candidate_ref: OptimizationCandidateRefV1 | null
  candidates[1..32]:
    candidate_ref: OptimizationCandidateRefV1
    algorithm_ref: NamedAlgorithmRefV1
    implementation_variant_ref: ArtifactRefV1
    eligibility_predicate_refs[1..32]:
      McdContractRefV1(kind=policy)
    metric_receipt_refs[0..64]:
      OptimizationEvidenceRefV1(evidence_role=metric_receipt)
    semantic_receipt_refs[0..64]:
      OptimizationEvidenceRefV1(evidence_role=semantic_receipt)
    disposition: not_evaluated | rejected | qualified_not_selected | selected
    blocking_diagnostic_ref: DiagnosticCodeRefV1 | null
  selected_candidate_ref: OptimizationCandidateRefV1 | null
  selected_reason_evidence_refs[0..64]:
    OptimizationEvidenceRefV1(evidence_role=selection_reason)
  invalidation_condition_refs[1..32]:
    McdContractRefV1(kind=policy)
  exposure: read_only
  projection_content_hash: SHA-256

OptimizationDecisionProjectionRefV1
  projection_id: StableId
  projection_revision: positive uint64
  owner_document_id: StableId
  source_revision_ref:
    exact ArtifactCandidateBindingV1.source_revision_ref
  projection_content_hash: SHA-256
```

`OptimizationEvidenceRefV1.signed_record_ref`は[AI Verification／Provenance §7](../01-governance/ai-verification-provenance.md#7-evidence-envelope)の完成`MirakanSignedRecordV1`へexact解決し、`purpose`はOwnerが登録したmetric／semantic／selection Receipt purpose、`subject_sha256`は当該Candidate closureを含む完成subject、`signed_record_hash`は署名を含む完成Recordとbyte equalityにする。裸のArtifact ref、hash-only値、別roleのReceipt、AI生成説明を代用しない。

同じ`candidate_ref`は完成`ArtifactCandidateBindingV1`のSource revision、Candidate root、Contract、Toolchain、Target、artifact集合を一意に固定する。algorithm revision、implementation variant、fixture、input traceはcandidate行とProjection rootにexactに閉じ、一Fieldでも変われば別Projection／Candidateである。`baseline_candidate_ref != null`は`candidates[]`内のexact一件へ解決し、`qualified_not_selected | selected`かつ完全Receipt closureでなければならない。初回characterizationだけはnullを許し、改善率または過去Shipping baselineを主張しない。

`selected_candidate_ref != null`なら同じrefの`disposition=selected`がexact一件、`selected_reason_evidence_refs[]`が1件以上あり、nullならselected行とselection reasonは双方0件でなければならない。候補行は次のexact matrixを満たす。

| `disposition` | metric／semantic Receipt | `blocking_diagnostic_ref` | selection reason |
|---|---|---|---|
| `not_evaluated` | 双方exact 0件 | exact `null` | 禁止 |
| `rejected` | 検証済み取得分だけ0～64件。missing分を捏造しない | exact一件。失敗した`eligibility_predicate_refs[]`のうちcanonical ref順で最小のpredicateに登録されたDiagnosticを指す | 禁止 |
| `qualified_not_selected` | Owner-required完全closure、双方1件以上 | exact `null` | root selection reasonの対象外 |
| `selected` | Owner-required完全closure、双方1件以上 | exact `null` | rootに1～64件必須 |

全semantic／performance hard predicateは`eligibility_predicate_refs[]`へ一件ずつ登録し、実行完了順ではなくcanonical ref順でrejection Diagnosticを決める。baselineとcandidateの測定環境が一致しない、Receiptがmissing／stale／revoked、oracle不一致、選択が複数、未登録候補、unknown enum、hash不一致ならProjection生成と昇格をfail closedにする。`OptimizationDecisionProjectionRefV1`は完成Projectionのidentity／revision／Owner／source revision／content hashとbyte equalityにし、ID-only、revision-only、`latest`、同revision別hashを拒否する。同じ`projection_id`はrevision 1から開始し、Candidate集合、policy、Receipt、disposition、baseline、selection、invalidation条件の完成bytesが変わるたびexact `N+1`へ進める。read時はsource revision、Target Profile、Toolchain lock、policy ref、全Receiptのfreshness／revocationを再検査し、drift時はstale Projectionを返さず新revisionを要求する。

全objectはclosedで全Field必須、unknown Field禁止とする。`candidates[]`は`artifact_candidate_id, artifact_candidate_binding_sha256`のunsigned byte順、各Evidence arrayは`evidence_role, signed_record_ref.purpose, signed_record_ref.subject_sha256, signed_record_ref.signed_record_hash`順、policy ref配列はMCD canonical ref順へstrict sortし、duplicate／same ID different hashを拒否する。`projection_content_hash`はASCII `MIRAKAN_OPTIMIZATION_DECISION_PROJECTION_V1`と同Fieldだけを除くclosed canonical MCD bytesを`uint32_be` length framingしてSHA-256する。Projection生成時に読んだ全Candidate binding、Receipt、policy、artifactのfreshness／revocationを再検査し、omitted rangeまたは未解決refを許可しない。

Owner別の最低metric familyは次とする。値、absolute threshold、fixture、Backend固有fieldは各Ownerが決定し、本書は共通比較形式だけを所有する。

| Owner領域 | 最低metric family |
|---|---|
| ECS／Memory | iteration P50／P95／P99、allocation／一般fallback count、bytes per row、chunk occupancy／fragmentation、query-cache hit／miss／invalidation、structural change cost |
| Physics／Collision | step P50／P95／P99、job wait、active／sleep body数、body insertion throughput、temporary allocator high-water／failure、candidate pair数とfilter stage別reject数 |
| Navigation | query P50／P95／P99、node expansion、heap operation、allocation／fallback、cache hit／miss／stale、out-of-nodes、sliced iteration／completion |
| Rendering | CPU／GPU P50／P95／P99、command／indirect argument数、visible-set equalityとfalse occlusion、transient physical peak、alias／barrier／validation error |

AIはProjectionをread／explainできるが、raw full trace、全Project dump、native object、pointer、address、Backend objectを受け取らず、threshold、eligibility、baseline、selected ref、Receiptを変更または補完しない。将来の提案／選択Operationは[AI Security／Approval](../01-governance/ai-security-approval.md)に従って別途Activationするまで存在せず、current Performance MCD Operation集合は§11のexact状態を維持する。reference implementationはoracle、またはOwnerが別途適格化した明示的semantic fallbackになり得るが、旧／新経路の暗黙併載、deprecated alias、silent fallback、runtime auto-tuningを許可しない。benchmark candidateは非dispatchableである。

AIがこのProjectionを正しく理解することは[AI Verification／Provenance §5.5](../01-governance/ai-verification-provenance.md#55-代表fixture)の`OptimizationDecisionExplanationFixtureV1`で検証する。fixture logical ID、Case、Suite、Receiptが未materializeの間は、Provider／Model／Hostの一般Eval成功をoptimization explanationのEvidenceへ流用せず、AI向けoptimization bindingを利用可能と表示しない。

## 9. Owner-typed workload scale modelと`ProjectScaleEnvelopeV2`

ScaleはWorldやGameplayの存在を前提にせず、ownerが登録したworkload domainと数値dimensionの集合で表す。UI-only、strict headless、Editor tool、resource service、content-only Projectは、存在しないWorld／Entity／authoritative gameplay用の偽axisやfidelity floorを作らない。World／spatialは選択domainが明示要求する場合だけ追加する。

本節の全`owner_ref`は[Gameplay programming model §Owner identity registry](../03-authoring/gameplay-programming-model.md#owner-identity-registry)が所有する`OwnerIdentityLocalRefV1`をexact importする。`owner_id`だけ、`latest`／`current revision`、文書revision／文書content hash、未解決の`owner_content_hash` placeholderをwire値として許可しない。次の短記は保存するIDではなく、右列の三Fieldをそのままmaterializeする本節内の規範的な値短記である。hashは同節のASCII `MIRAKAN_DIAGNOSTIC_OWNER_LOCAL_IDENTITY_V1`式で再計算した32 byte値をlowercase hex表示したものであり、Owner Identity Registryの同一active rowとbyte equalityを必須にする。

| 規範的な値短記 | exact `OwnerIdentityLocalRefV1` |
|---|---|
| `performance_owner_ref_v1` | `{owner_id=owner.core.performance, owner_revision=1, owner_content_hash=0bfc3cea41e2ef23d63ba9bc6550e83d839047dac3e56ce21f004cc1e24c64bd}` |
| `world_owner_ref_v1` | `{owner_id=owner.core.world, owner_revision=1, owner_content_hash=02e558a2c554d54711631421e22d449b92c4339c74d1159157620b5e37dc6625}` |
| `asset_lifecycle_owner_ref_v1` | `{owner_id=owner.core.asset_lifecycle, owner_revision=1, owner_content_hash=fbff9d16233120f9a61ffdf578a631003bdfdabd0460bd5b6e030bff13303f6b}` |
| `project_state_owner_ref_v1` | `{owner_id=owner.core.project_state, owner_revision=1, owner_content_hash=8629bba70acf6c53968cdbe97a9ccc4043adacb8eef29c9e4303df091f6ee428}` |
| `security_approval_owner_ref_v1` | `{owner_id=owner.core.security_approval, owner_revision=1, owner_content_hash=670f9d8831a330c6b0b58e500009af1f5bcafbc1f47d88896a9911854f6e0f8d}` |

```text
WorkloadDomainTypeRefV1
  domain_type_id
  domain_type_version: positive uint32
  domain_type_content_hash: SHA-256

WorkloadIntentKindRefV1
  intent_kind_id
  intent_kind_version: positive uint32
  intent_kind_content_hash: SHA-256

WorkloadIntentKindRecordV1
  intent_kind_id
  intent_kind_version: positive uint32
  owner_ref: OwnerIdentityLocalRefV1
  required_owner_intent_branch:
    content | authoring | authority
  allowed_authority_classes[1..5]:
    authoritative_simulation | presentation | ui | tooling | resource_service
  allowed_semantic_requirement_modes[1..5]:
    authoritative_equivalence | presentation_fidelity |
    functional_contract | resource_slo | none
  intent_kind_content_hash: SHA-256

WorkloadIntentKindRegistryRefV1
  registry_id
  registry_version: positive uint32
  registry_content_hash: SHA-256

WorkloadIntentKindRegistryV1
  registry_id: performance.workload_intent_kind.registry.active
  registry_version: positive uint32
  records[1..256]: WorkloadIntentKindRecordV1
  registry_content_hash: SHA-256

WorkloadOwnerDefinitionRefV1
  registry_ref: WorkloadOwnerDefinitionRegistryRefV1
  definition_id
  definition_version: positive uint32
  definition_content_hash: SHA-256
  definition_kind: workload_domain_owner | owner_scale_intent_owner
  owner_ref: OwnerIdentityLocalRefV1

WorkloadOwnerDefinitionRecordV1
  definition_id
  definition_version: positive uint32
  definition_kind: workload_domain_owner | owner_scale_intent_owner
  owner_ref: OwnerIdentityLocalRefV1
  schema_kind:
    workload_domain_intent_v1 |
    world_scale_intent_v1 | content_scale_intent_v1 |
    authoring_scale_intent_v1 | authority_scale_intent_v1
  owner_intent_branch:
    world | content | authoring | authority |
    canonical omission when definition_kind=workload_domain_owner
  allowed_domain_type_refs[0..256]: WorkloadDomainTypeRefV1
  definition_content_hash: SHA-256

WorkloadOwnerDefinitionRegistryRefV1
  registry_id
  registry_version: positive uint32
  registry_content_hash: SHA-256

WorkloadOwnerDefinitionRegistryV1
  registry_id: performance.workload_owner_definition.registry.active
  registry_version: positive uint32
  records[1..4096]: WorkloadOwnerDefinitionRecordV1
  registry_content_hash: SHA-256

PerformanceTargetProfileRefV1
  target_profile_id
  target_profile_version: positive uint32
  target_profile_content_hash: SHA-256

PerformanceQualificationReceiptRefV1
  qualification_id
  qualification_version: positive uint32
  qualification_subject_hash: SHA-256
  signed_record_hash: SHA-256

PerformanceQualificationBindingRefV1
  qualification_binding_id
  qualification_binding_version: positive uint32
  qualification_binding_hash: SHA-256

PerformanceDecisionRecordRefV1
  decision_id
  decision_version: positive uint32
  decision_content_hash: SHA-256

WorkloadDomainTypeRecordV1
  domain_type_id
  domain_type_version: positive uint32
  owner_ref: OwnerIdentityLocalRefV1
  authority_class:
    authoritative_simulation | presentation | ui | tooling | resource_service
  spatial_requirement: forbidden | optional | required
  required_intent_kind_refs[0..32]: WorkloadIntentKindRefV1
  allowed_scale_dimension_refs[1..256]: RuntimeScaleIntentDimensionRefV1
  semantic_requirement_mode:
    authoritative_equivalence | presentation_fidelity |
    functional_contract | resource_slo | none
  domain_type_content_hash: SHA-256

WorkloadDomainTypeRegistryRefV1
  registry_id
  registry_version
  registry_content_hash

WorkloadDomainTypeRegistryV1
  registry_id: performance.workload_domain_type.registry.active
  registry_version
  intent_kind_registry_ref: WorkloadIntentKindRegistryRefV1
  records[1..4096]: WorkloadDomainTypeRecordV1
  registry_content_hash

WorkloadDomainIntentV1
  intent_id: StableId
  intent_version: positive uint32
  domain_type_ref: WorkloadDomainTypeRefV1
  owner_definition_ref: WorkloadOwnerDefinitionRefV1
  dimension_values[1..64]: RuntimeScaleIntentDimensionValueV1
  requirement_refs[0..64]: McdContractRefV1(kind=requirement)
  equivalence_or_fidelity_policy_refs[0..64]: McdContractRefV1(kind=policy)
  intent_content_hash

WorkloadDomainIntentRefV1
  intent_id: StableId
  intent_version: positive uint32
  intent_content_hash: SHA-256

SelectedWorkloadDomainRecordBindingV1
  domain_type_ref: WorkloadDomainTypeRefV1
  registry_ref: WorkloadDomainTypeRegistryRefV1
  selected_record_hash: exact domain_type_ref.domain_type_content_hash

OwnerScaleIntentRefV1
  intent_kind:
    world | content | authoring | authority
  intent_ref:
    world: exact {intent_id, intent_version, intent_content_hash,
                  owner_ref: OwnerIdentityLocalRefV1,
                  owner_definition_ref: WorkloadOwnerDefinitionRefV1}
    | content: exact {intent_id, intent_version, intent_content_hash,
                     owner_ref: OwnerIdentityLocalRefV1,
                     owner_definition_ref: WorkloadOwnerDefinitionRefV1}
    | authoring: exact {intent_id, intent_version, intent_content_hash,
                       owner_ref: OwnerIdentityLocalRefV1,
                       owner_definition_ref: WorkloadOwnerDefinitionRefV1}
    | authority: exact {intent_id, intent_version, intent_content_hash,
                       owner_ref: OwnerIdentityLocalRefV1,
                       owner_definition_ref: WorkloadOwnerDefinitionRefV1}

ProjectScaleEnvelopeV2
  scale_envelope_id: StableId
  project_id: StableId
  schema_version: 2
  source_project_ref:
    exact {project_id, project_revision, document_set_hash}
  target_profile_refs[1..16]: PerformanceTargetProfileRefV1
  workload_domain_registry_ref: WorkloadDomainTypeRegistryRefV1
  workload_intent_kind_registry_ref: WorkloadIntentKindRegistryRefV1
  workload_owner_definition_registry_ref:
    WorkloadOwnerDefinitionRegistryRefV1
  scale_dimension_registry_ref: RuntimeScaleIntentDimensionRegistryRefV1
  workload_domain_intents[1..64]: WorkloadDomainIntentV1
  selected_domain_record_bindings[1..64]:
    SelectedWorkloadDomainRecordBindingV1
  owner_intent_refs[0..4]: OwnerScaleIntentRefV1
  decision_refs[0..64]: PerformanceDecisionRecordRefV1
  envelope_hash: SHA-256

ProjectScaleEnvelopeRefV2
  scale_envelope_id: StableId
  schema_version: 2
  envelope_hash: SHA-256

PerformanceQualificationSubjectRefV1
  subject_kind:
    workload_domain | workload_intent | integrated_envelope |
    scale_dimension | migration
  subject_ref:
    workload_domain: WorkloadDomainTypeRefV1
    | workload_intent: WorkloadDomainIntentRefV1
    | integrated_envelope: ProjectScaleEnvelopeRefV2
    | scale_dimension: RuntimeScaleIntentDimensionRefV1
    | migration: ProjectScaleDomainMappingRefV1

PerformanceQualificationSubjectV1
  qualification_id
  qualification_version: positive uint32
  owner_ref: OwnerIdentityLocalRefV1
  subject: PerformanceQualificationSubjectRefV1
  target_profile_refs[1..16]: PerformanceTargetProfileRefV1
  fixture_refs[1..64]: exact {fixture_id, fixture_version, fixture_content_hash}
  input_closure_hash
  result: pass | fail
  qualification_subject_hash: SHA-256

PerformanceQualificationReceiptV1
  subject: PerformanceQualificationSubjectV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=performance_qualification)

PerformanceQualificationBindingV1
  qualification_binding_id
  qualification_binding_version: positive uint32
  subject: exact PerformanceQualificationSubjectRefV1
  qualification_receipt_ref: PerformanceQualificationReceiptRefV1
  qualification_binding_hash: SHA-256

ProjectScaleActivationProjectionV1
  projection_id
  projection_version: positive uint32
  scale_envelope_ref/hash: exact receipt-free ProjectScaleEnvelopeV2
  workload_domain_qualification_binding_refs[1..64]:
    PerformanceQualificationBindingRefV1
  workload_intent_qualification_binding_refs[1..64]:
    PerformanceQualificationBindingRefV1
  integrated_envelope_qualification_binding_ref:
    PerformanceQualificationBindingRefV1
  scale_dimension_qualification_binding_refs[1..256]:
    PerformanceQualificationBindingRefV1
  projection_hash: SHA-256
```

`WorkloadDomainTypeRefV1`、`WorkloadIntentKindRefV1`、`WorkloadOwnerDefinitionRefV1`、`WorkloadDomainIntentRefV1`、`ProjectScaleEnvelopeRefV2`は各Receipt-free Record外の参照形であり、Recordのlogical ID／version／self-excluding content hashからmaterializeする。Domain record hashはASCII `MIRAKAN_WORKLOAD_DOMAIN_TYPE_RECORD_V1`、Intent Kind record hashはASCII `MIRAKAN_WORKLOAD_INTENT_KIND_RECORD_V1`、Owner Definition record hashはASCII `MIRAKAN_WORKLOAD_OWNER_DEFINITION_RECORD_V1`と当該hash Fieldだけを除くReceipt-free canonical bytesから計算し、Record自身へhash付きRef、Qualification Receipt／Bindingを埋め戻さない。Intent Kind Registry hashはASCII `MIRAKAN_WORKLOAD_INTENT_KIND_REGISTRY_V1`、Owner Definition Registry hashはASCII `MIRAKAN_WORKLOAD_OWNER_DEFINITION_REGISTRY_V1`、各Registry ID／version、record count、logical IDのNFC UTF-8 bytes／version順へstrict sortした完成record bytesを各`uint32_be` length framingして計算し、各`registry_content_hash`だけを除外する。Intent／Envelope／Dimensionも同様に各自己hashだけを除くReceipt-free preimageを持つ。`migration` branchとMapping baseはwire designを予約するが、§9のsigned evidence gate成立前はsubject／base／Binding／Registryをmaterializeしない。

`WorkloadOwnerDefinitionRefV1`の全Fieldは同じRegistry rowとbyte equalityでなければならない。`definition_kind=workload_domain_owner`は`schema_kind=workload_domain_intent_v1`、`owner_intent_branch` canonical omission、`allowed_domain_type_refs`非空である。`owner_scale_intent_owner`はbranchと同名の`*_scale_intent_v1` schema、allowed Domain集合`[]`である。discriminator外schema／branch、Owner ID prefixからのschema推論、RegistryなしのDefinition refを拒否する。各`WorkloadDomainIntentV1.owner_definition_ref`はEnvelopeの`workload_owner_definition_registry_ref`と同Registryを指す`workload_domain_owner`で、そのallowed Domain集合へ当該`domain_type_ref`を含み、Intent Qualification subject ownerはDefinition rowの`owner_ref`とbyte equalityでなければならない。各`OwnerScaleIntentRefV1.intent_ref.owner_ref`はbranchごとに`world_owner_ref_v1 | asset_lifecycle_owner_ref_v1 | project_state_owner_ref_v1 | security_approval_owner_ref_v1`の対応するexact三Field、`owner_definition_ref`は`owner_scale_intent_owner`、branch／schema／同じexact Owner refがenclosing union branchとbyte equalityでなければならない。

`PerformanceQualificationSubjectRefV1.subject_kind`のwire schemaは上記五branchを予約するが、current materialization可能集合は`workload_domain | workload_intent | integrated_envelope | scale_dimension`のexact四branchである。`migration` branchのcurrent Subject／Receipt／Binding／Mapping集合はexact `[]`で、signed evidence gate成立後のatomic activationだけが五番目を有効化できる。discriminator外branch、ID／version／content hash欠落、別kind ref、activation前の`migration`を拒否する。Subject `owner_ref`はkind別に、`workload_domain`ではRefが解決する`WorkloadDomainTypeRecordV1.owner_ref`、`workload_intent`ではIntentの`owner_definition_ref`が解決する上記Registry rowの`owner_ref`、`integrated_envelope`では`performance_owner_ref_v1`、`scale_dimension`ではRefが解決する`RuntimeScaleIntentDimensionRecordV1.owner_ref`とbyte equalityにする。Activation後の`migration`はRefが解決する`ProjectScaleDomainMappingRecordV1.owner_ref`とbyte equalityにする。表示名、ID prefix、signer自己申告からownerを補完しない。Intent Kind registryは`intent_kind_id`／version順、Owner Definition Registryは`definition_id`／version順、Domain registry recordは`domain_type_id`／version順、Envelopeのintentとselected bindingはdomain type ID／intent ID／intent version順、owner intentは上記kind順へstrict sortし、duplicate、same-ID／version different-hash、owner偽装を拒否する。Domain Registryの`intent_kind_registry_ref`、Envelopeの`workload_intent_kind_registry_ref`、Envelopeが参照するDomain Registryの同Refはbyte equalityでなければならない。Envelope内の全Definition refのRegistryはEnvelopeの`workload_owner_definition_registry_ref`とbyte equalityである。Envelopeのintent集合、`selected_domain_record_bindings[]`、Registryから選択したrecord集合はdomain type ref／record hashでset equalityを必須にし、各bindingのRegistry refはEnvelopeの一件とbyte equalityでなければならない。これにより選択Domain、Intent Kind、Owner Definition rowの一Field変更はRegistryとEnvelope preimageを必ず変更する。

Qualification生成順はcurrent四subject kind、およびgate成立後の`migration`で`receipt-free base → base ref → PerformanceQualificationSubjectV1 → signed Receipt → PerformanceQualificationBindingV1 → root外Activation projection`である。`qualification_subject_hash`はASCII `MIRAKAN_PERFORMANCE_QUALIFICATION_SUBJECT_V1`、binding hashはASCII `MIRAKAN_PERFORMANCE_QUALIFICATION_BINDING_V1`、projection hashはASCII `MIRAKAN_PROJECT_SCALE_ACTIVATION_PROJECTION_V1`と各自己Fieldを除くcount／length-framed canonical bytesから計算する。Receipt refのID／version／subject hash／signed hashはwrapper内subject／signed recordとexact equalityで、BindingのsubjectはReceipt subjectのbase refとbyte equalityにする。Receipt／Binding／Projectionをbase、Registry、Envelope hashへ戻さない。

Projectionのdomain Binding subject集合はEnvelope `selected_domain_record_bindings[]`が解決するDomain ref集合、intent Binding subject集合はEnvelope `workload_domain_intents[]`からmaterializeした`WorkloadDomainIntentRefV1`集合、Dimension Binding subject集合は同Intent群の`dimension_values[].dimension_ref`のunionと、それぞれexact set equalityにする。`integrated_envelope_qualification_binding_ref`はexact一件で、そのBinding／Receipt subjectはProjectionの`scale_envelope_ref`とbyte equalityにする。四Binding fieldはsubject kindごとに分離し、各配列をsubject logical ID／version／content hash、Binding ID／version順へstrict sortし、duplicate、別kind、missing、extraを拒否する。全Bindingが同じProjectionのEnvelope closureから導出されたTarget集合を持つことを検証し、別Envelopeで有効なDomain／Intent／Dimension Bindingを混在させない。Production consumerはProjectionが指すsigned Receiptのsubject／result=`pass`／freshness／revocationだけを検証し、Fixture bodyを解決しない。current四branchそれぞれで正しいbase refのままSubject ownerだけを別の有効Ownerへ差し替えるfixture、Binding subjectだけを別baseへ差し替えるfixtureに加え、Domain／Intent／Dimension各集合のmissing／extra、cross-envelope Binding、integrated Bindingだけ別Envelope、canonical順序違反を各一原因で拒否する。`migration` branchの同等fixtureは§9のActivation Qualificationにだけ属する。

`spatial_requirement=required`のactive domainが一件以上なら`owner_intent_refs[]`にworld branchをexact一件必須、全active domainが`forbidden`ならworld branchを禁止する。`optional`が一件以上かつ`required`が0件ならworld branch有無の両方を許すが、存在時はexact World owner intentへ解決し、当該optional domainのspatial dimension closureへ含めなければならない。`forbidden` domainへspatial dimensionまたはWorld refを結び付けない。content／authoring／authority branchは選択Domain群の`required_intent_kind_refs[]`をEnvelope固定のIntent Kind Registryへ全件exact解決し、そのrecord集合が要求する`required_owner_intent_branch`ごとにexact一件を必須とする。同じbranchを複数kindが要求してもOwner intent refは一件だけで、どのkindも要求しないbranchは禁止する。各Domain recordの`authority_class`と`semantic_requirement_mode`は参照する全Intent Kind recordの各allowed集合へ含まれなければならない。discriminator外branch、同kind／branch重複、owner不一致、unknown／stale kind ref、bare ref、Kind Registry差し替えを拒否する。

`semantic_requirement_mode`が`authoritative_equivalence`ならequivalence policy、`presentation_fidelity`ならfidelity policy、`functional_contract`なら機能Requirement、`resource_slo`ならSLO Requirementを1件以上必須にする。`none`は`authority_class=tooling | resource_service`かつDomain recordが明示許可する場合だけ使用し、他branchのpolicy／requirementをcanonical omissionする。全Project共通のGameplay fidelity floorや最低Entity数を置かない。

初期Core registryは次の完全五Receipt-free recordだけを持つ。全rowは`domain_type_version=1`、exact `owner_ref=performance_owner_ref_v1`、表のtyped dimension ref、self-excluding content hashを持つ。`none` branchはtooling／resource serviceだけに許可する。

| domain type ID | authority／spatial | required intent kind refs | allowed dimension refs | semantic mode |
|---|---|---|---|---|
| `workload.core.authoritative_simulation` | `authoritative_simulation`／`optional` | `intent_kind.performance.authoritative_state@1` | `scale.dimension.instance.peak_active_authoritative@1; scale.dimension.lifecycle.peak_create_per_simulation_step@1; scale.dimension.lifecycle.peak_retire_per_simulation_step@1; scale.dimension.event.peak_authoritative_per_simulation_step@1` | `authoritative_equivalence` |
| `workload.core.presentation` | `presentation`／`forbidden` | `intent_kind.performance.presentation_fidelity@1` | `scale.dimension.instance.peak_visible@1; scale.dimension.presentation.peak_active@1` | `presentation_fidelity` |
| `workload.core.ui` | `ui`／`forbidden` | `intent_kind.performance.functional_contract@1` | `scale.dimension.instance.peak_live@1; scale.dimension.presentation.peak_active@1` | `functional_contract` |
| `workload.core.tooling` | `tooling`／`forbidden` | `[]` | `scale.dimension.instance.total_authored@1; scale.dimension.instance.peak_live@1` | `none` |
| `workload.core.resource_service` | `resource_service`／`forbidden` | `intent_kind.performance.resource_slo@1` | `scale.dimension.instance.total_authored@1; scale.dimension.instance.peak_live@1` | `resource_slo` |

初期Intent Kind Registryは次の完全四Receipt-free recordだけを持つ。全rowは`intent_kind_version=1`、exact `owner_ref=performance_owner_ref_v1`、表のbranch／allowed集合、self-excluding `intent_kind_content_hash`を持つ。Domain表の`@1` refはこの四recordのID／version／content hashとbyte equalityで、ID文字列に`@1`を含めない。

| intent kind ID | required owner-intent branch | allowed authority classes | allowed semantic modes |
|---|---|---|---|
| `intent_kind.performance.authoritative_state` | `authority` | `[authoritative_simulation]` | `[authoritative_equivalence]` |
| `intent_kind.performance.presentation_fidelity` | `content` | `[presentation]` | `[presentation_fidelity]` |
| `intent_kind.performance.functional_contract` | `authoring` | `[ui]` | `[functional_contract]` |
| `intent_kind.performance.resource_slo` | `content` | `[resource_service]` | `[resource_slo]` |

四record以外をCore Registryへ暗黙追加せず、extension ownerの追加recordは自身のexact owner ref、closed branch、非空allowed集合を持つ。Registry ref／record refのID、version、content hashの一Fieldmutation、四Core recordのmissing／extra／duplicate／noncanonical order、owner substitution、Domainが許可外authorityまたはsemantic modeを使うcase、必要owner-intent branchのmissing／extra／duplicate、Domain RegistryとEnvelopeのKind Registry ref差を一原因ずつrejectし、last-valid Envelope／Activation projectionを不変にする。

初期Owner Definition Registryのbuilt-in inventoryは次の完全九recordである。全rowは`definition_version=1`、表のexact owner ref／kind／schema／branch／allowed Domain集合、self-excluding content hashを持つ。最初の五recordはCore Domain用`workload_domain_owner`、後半四recordはcanonical owner intent用`owner_scale_intent_owner`である。

| definition ID | exact owner | kind／schema／branch | allowed Domain refs |
|---|---|---|---|
| `definition.performance.workload.authoritative_simulation` | `performance_owner_ref_v1` | `workload_domain_owner`／`workload_domain_intent_v1`／omitted | `[workload.core.authoritative_simulation@1]` |
| `definition.performance.workload.presentation` | `performance_owner_ref_v1` | `workload_domain_owner`／`workload_domain_intent_v1`／omitted | `[workload.core.presentation@1]` |
| `definition.performance.workload.ui` | `performance_owner_ref_v1` | `workload_domain_owner`／`workload_domain_intent_v1`／omitted | `[workload.core.ui@1]` |
| `definition.performance.workload.tooling` | `performance_owner_ref_v1` | `workload_domain_owner`／`workload_domain_intent_v1`／omitted | `[workload.core.tooling@1]` |
| `definition.performance.workload.resource_service` | `performance_owner_ref_v1` | `workload_domain_owner`／`workload_domain_intent_v1`／omitted | `[workload.core.resource_service@1]` |
| `definition.performance.owner_intent.world` | `world_owner_ref_v1` | `owner_scale_intent_owner`／`world_scale_intent_v1`／`world` | `[]` |
| `definition.performance.owner_intent.content` | `asset_lifecycle_owner_ref_v1` | `owner_scale_intent_owner`／`content_scale_intent_v1`／`content` | `[]` |
| `definition.performance.owner_intent.authoring` | `project_state_owner_ref_v1` | `owner_scale_intent_owner`／`authoring_scale_intent_v1`／`authoring` | `[]` |
| `definition.performance.owner_intent.authority` | `security_approval_owner_ref_v1` | `owner_scale_intent_owner`／`authority_scale_intent_v1`／`authority` | `[]` |

Extension Domainは同じRegistryへowner自身の`workload_domain_owner` recordを寄与し、allowed Domain ref集合をexactに宣言する。built-in九recordの一件missing／extra／duplicate、schema／branch／owner／allowed Domainの一Fieldmutation、Definition RefのRegistry／kind／ownerだけを別valid rowへ差し替えるcase、EnvelopeとIntentのRegistry ref不一致を一原因ずつrejectする。Registry／Definition refはReceipt-freeであり、Qualification Receipt／Binding／Activation projectionをhash preimageへ戻さない。

各row固定後に同logical suffixの`qualification.performance.workload_domain.core.*@1` subject／Receiptと`qualification_binding.performance.workload_domain.core.*@1`をroot外で一件ずつ作る。Activation projectionのdomain binding集合はEnvelopeが選択したDomain record集合とexact set equalityであり、§9のProjection四集合規則を同じvalidatorで検証する。

表の`@1`はexact version／content hashまたはContract set root付きrefで保存する。World ownerは必要なProjectに`workload.world.spatial`を寄与し、Feature／Genre／Project ownerは自身のdomainを寄与する。CoreはShooter、enemy、vehicle、level、quest、terrain等の語彙を登録しない。Network authority variantsは専用仕様、Threat Model、Product activation前は`not_activated`である。

`envelope_hash`はASCII `MIRAKAN_PROJECT_SCALE_ENVELOPE_V2`と自己Fieldを除く全Receipt-free Fieldのlength-framed canonical bytesから計算する。selected Registry row hash、exact Project triple、Target、owner intent branch、Decisionの一Fieldでも変われば別Envelopeである。Qualification Receipt／Binding／Activation projectionはEnvelope hash入力ではない。Production Envelope／Domain／Intent／Dimension recordはFixture bodyを解決せず、root外Activation projectionから署名済みQualification Receiptのsubject／result／freshness／revocationだけを検証する。表示用`scale_class`をSourceへ保存せず、Projectionはdomain closureとEnvelope hashから`compact_reference | medium_candidate | large_local_candidate | distributed_candidate`を決定的に導出する。Target readinessは[Project State §3.4](../03-authoring/project-state.md#34-target-readiness)の`TargetReadinessV1`をread-only投影し、`state`は`predicted | blocked | qualified`だけ、性能未達の理由は`blocked_reason_ref`だけに置く。

`ProjectScaleEnvelopeV1`のcanonical bytesとそれを使用する旧Projectが実在することは現計画からは証明されておらず、current Source、Editor、AI projection、Compile Manifest、deserializerへ登録しない。`operation.performance.migrate_project_scale_envelope@1`は[Executable Contracts §8.1.2](../02-foundation/executable-contracts.md#812-conditional-legacy-migration-evidence-gate)のconditional legacy migrationで、current状態は`not_activated`である。この移行に固有のcurrent MCD／Owner Manifest／Service allowlist／Policy／Validator／migration Diagnostic／Operation Receipt／Provider／MCP／alias、migration Qualification subject／Receipt／Binding、Activation Catalog／projection subset、`ProjectScaleDomainMappingRecordV1`／Registry、Migration Manifestはすべてexact `[]`である。`service.offline_project_migrator`、`capability.authoring.offline_migration`、`profile.isolation.offline_project_migrator`のcurrent集合もexact `[]`で、移行要求をdispatchしない。

将来Activationするには、実在する旧Envelope／Project bytes、旧MCD local record／common envelope／payload、source `ContractSetSnapshotV2`、Owner Identity Registry、Named Algorithm Registry、`FoundationDefinitionClosureV1`、全retained artifact ref／hash、正負fixtureを推測なしで列挙したsigned `LegacyMigrationInventoryV1`が§8.1.2 gateを満たさなければならない。その同じ承認済みContract set transactionだけが、Operation／Type／Policy／Validator／migration Diagnostic／Receipt、offline Service／Capability／Isolation Profile、Service allowlist、Mapping Registry／records、Qualification Receipt／Binding、Manifest、Provider／MCP projectionを完全closureとして同時にmaterializeできる。次のschemaとrecord値はpost-activation destination templateであり、block内の`status=active`、Service ref、Policy ref、Provider exposureをcurrent refまたはcurrent product surfaceとして解釈しない。

```text
ProjectScaleEnvelopeMigrationManifestV1
  manifest_id: performance.project_scale_envelope.migration.v1_to_v2
  manifest_version: 1
  operation_ref: McdContractRefV1(
    kind=operation,
    id=operation.performance.migrate_project_scale_envelope,
    version=1, contract_set_hash)
  input_type_ref: McdContractRefV1(
    kind=type, id=type.performance.project_scale_envelope_migration_input,
    version=1, contract_set_hash)
  output_type_ref: McdContractRefV1(
    kind=type, id=type.performance.project_scale_envelope_migration_result,
    version=1, contract_set_hash)
  receipt_type_ref: McdContractRefV1(
    kind=type, id=type.performance.project_scale_envelope_migration_receipt,
    version=1, contract_set_hash)
  precondition_policy_ref: McdContractRefV1(
    kind=policy, id=policy.operation.performance.scale_migration.precondition,
    version=1, contract_set_hash)
  postcondition_policy_ref: McdContractRefV1(
    kind=policy, id=policy.operation.performance.scale_migration.postcondition,
    version=1, contract_set_hash)
  rate_limit_policy_ref: McdContractRefV1(
    kind=policy, id=policy.authoring.performance_scale_migration.rate_limit,
    version=1, contract_set_hash)
  validator_closure_ref: OperationValidatorClosureRefV1
  trusted_service_ref: TrustedServiceRefV1(
    service_id=service.offline_project_migrator, service_version=1,
    service_content_hash)
  trusted_service_allowlist_operation_local_refs[1]:
    ContractSetLocalRefV1(
      kind=operation,
      id=operation.performance.migrate_project_scale_envelope,
      version=1)
  diagnostic_refs[12]: DiagnosticCodeRefV1
  qualification_binding_refs[1..64]:
    exact PerformanceQualificationBindingRefV1(
      subject_kind=migration)
  manifest_hash: SHA-256

operation.performance.migrate_project_scale_envelope@1
  MCD common envelope:
    mcd_version=1; kind=operation;
    id=operation.performance.migrate_project_scale_envelope;
    version=1; status=active;
    title=Migrate Project Scale Envelope V1 to V2;
    description=Atomically migrate one legacy five-axis scale envelope
      through exact owner-typed workload-domain mappings;
    owners=[owner.core.performance]; requirement_refs=[];
    rationale_refs=[mirakan.arch.runtime-performance-capacity#9-owner-typed-workload-scale-modelとprojectscaleenvelopev2];
    since_contract_set=2; supersedes=[];
    tags=[authoring,migration,performance]
  operation_kind: command
  input_type: McdContractRefV1(
    kind=type, id=type.performance.project_scale_envelope_migration_input,
    version=1, contract_set_hash)
  output_type: McdContractRefV1(
    kind=type, id=type.performance.project_scale_envelope_migration_result,
    version=1, contract_set_hash)
  authority: TrustedServiceRefV1(
    service_id=service.offline_project_migrator, service_version=1,
    service_content_hash)
  risk_class: R3
  side_effects: [authoring]
  transaction: authoring_changeset
  idempotency: idempotent_with_key
  preconditions:
    [McdContractRefV1(
      kind=policy,
      id=policy.operation.performance.scale_migration.precondition,
      version=1, contract_set_hash)]
  postconditions:
    [McdContractRefV1(
      kind=policy,
      id=policy.operation.performance.scale_migration.postcondition,
      version=1, contract_set_hash)]
  validator_closure_ref:
    {closure_id=validator_closure.operation.performance.scale_migration,
     closure_version=1, closure_content_hash}
  timeout_ms: 120000
  rate_limit_policy: McdContractRefV1(
    kind=policy,
    id=policy.authoring.performance_scale_migration.rate_limit,
    version=1, contract_set_hash)
  audit_level: full_redacted
  provider_exposure: mcp_proposal
  receipt_type: McdContractRefV1(
    kind=type, id=type.performance.project_scale_envelope_migration_receipt,
    version=1, contract_set_hash)
  errors[12]: exact DiagnosticCodeRefV1 records for
    diagnostic.conflict.revision_mismatch
    diagnostic.authorization.denied
    diagnostic.approval.required
    diagnostic.authoring.lock_conflict
    diagnostic.mcd.operation_predicate_invalid
    diagnostic.operation.timeout
    diagnostic.operation.rate_limit_exceeded
    diagnostic.operation.idempotency_key_reuse
    diagnostic.performance.scale_v1_source_invalid
    diagnostic.performance.workload_domain_unresolved
    diagnostic.performance.scale_migration_ambiguous
    diagnostic.performance.scale_receipt_binding_mismatch

ProjectScaleEnvelopeMigrationInputV1
  input_type_ref: McdContractRefV1(
    kind=type, id=type.performance.project_scale_envelope_migration_input,
    version=1, contract_set_hash)
  operation_ref: McdContractRefV1(
    kind=operation, id=operation.performance.migrate_project_scale_envelope,
    version=1, contract_set_hash)
  before_project_ref:
    exact {project_id, expected_project_revision, document_set_hash}
  operation_intent_hash
  request_hash
  idempotency_key
  source_foundation_definition_closure_ref:
    FoundationDefinitionClosureRefV1
  retained_source_mcd_ref: McdContractRefV1(
    kind=type, id=type.performance.project_scale_envelope,
    version=1, source_contract_set_hash)
  source_envelope_v1_ref/hash
  source_axis_and_intent_closure_hash
  destination_domain_registry_ref: WorkloadDomainTypeRegistryRefV1
  destination_owner_definition_registry_ref:
    WorkloadOwnerDefinitionRegistryRefV1
  destination_dimension_registry_ref: RuntimeScaleIntentDimensionRegistryRefV1
  domain_mapping_registry_ref: ProjectScaleDomainMappingRegistryRefV1
  selected_domain_mapping_refs[1..64]: ProjectScaleDomainMappingRefV1
  preview_policy_ref: McdContractRefV1(kind=policy)
  validation_policy_ref: McdContractRefV1(kind=policy)
  mutation_authorization_binding: exact MutationAuthorizationBindingV2(
    risk_class=R3, authority_evidence=approval)

ProjectScaleEnvelopeMigrationResultV1
  disposition: migrated | rejected
  migrated:
    before_project_ref
    after_project_ref
    destination_envelope_v2_ref/hash
    destination_domain_registry_ref/hash
    destination_owner_definition_registry_ref/hash
    destination_dimension_registry_ref/hash
    preview_receipt_ref/hash
    validation_receipt_ref/hash
    public_publication_marker_ref/hash
    migration_receipt_ref/hash
  rejected:
    diagnostics[1..12]: DiagnosticCodeRefV1

PreparedProjectScaleEnvelopeMigrationReceiptPayloadV1
  publication_binding: exact PreparedReceiptPublicationBindingV1
  operation_ref
  operation_intent_hash
  request_hash
  idempotency_key
  before_project_ref
  after_project_ref
  source_foundation_definition_closure_ref:
    FoundationDefinitionClosureRefV1
  retained_source_mcd_ref: McdContractRefV1(
    kind=type, id=type.performance.project_scale_envelope,
    version=1, source_contract_set_hash)
  source_envelope_v1_ref/hash
  destination_envelope_v2_ref/hash
  destination_domain_registry_ref/hash
  destination_owner_definition_registry_ref/hash
  destination_dimension_registry_ref/hash
  domain_mapping_registry_ref/hash
  selected_domain_mapping_refs[1..64]: ProjectScaleDomainMappingRefV1
  omitted_legacy_axis_refs[0..5]
  preview_receipt_payload_ref/hash
  validation_receipt_payload_ref/hash
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1
  diagnostics[0..12]: DiagnosticCodeRefV1
  prepared_payload_hash

ProjectScaleEnvelopeMigrationReceiptV1
  published_receipt:
    exact PublishedDomainReceiptV2 whose
    prepared_domain_receipt_payload_ref/hash resolves
    PreparedProjectScaleEnvelopeMigrationReceiptPayloadV1

ProjectScaleDomainMappingRefV1
  mapping_id
  mapping_version: positive uint32
  mapping_content_hash: SHA-256

ProjectScaleDomainMappingRecordV1
  mapping_id
  mapping_version: positive uint32
  owner_ref: OwnerIdentityLocalRefV1
  source_axis_ref/hash
  source_intent_predicate_ref: McdContractRefV1(kind=policy)
  destination_domain_type_ref: WorkloadDomainTypeRefV1
  destination_owner_definition_ref: WorkloadOwnerDefinitionRefV1
  destination_intent_schema_ref: McdContractRefV1(kind=type)
  semantic_field_mapping_policy_ref: McdContractRefV1(kind=policy)
  mapping_content_hash: SHA-256

ProjectScaleDomainMappingRegistryRefV1
  registry_id
  registry_version: positive uint32
  registry_content_hash: SHA-256

ProjectScaleDomainMappingRegistryV1
  registry_id: performance.project_scale_domain_mapping.registry.active
  registry_version: 1
  records[1..4096]: ProjectScaleDomainMappingRecordV1
  registry_content_hash: SHA-256
```

Activation後の`ProjectScaleDomainMappingRecordV1.mapping_content_hash`はASCII `MIRAKAN_PROJECT_SCALE_DOMAIN_MAPPING_RECORD_V1`と自己Fieldを除くReceipt-free length-framed canonical bytes、Registry hashはASCII `MIRAKAN_PROJECT_SCALE_DOMAIN_MAPPING_REGISTRY_V1`、Registry ID／version、record count、mapping ID／version順の全Receipt-free record bytesから計算して自己Fieldを除く。Ref三FieldはRegistry内のexact一件へ解決し、selected ref集合はRegistry member subset、source axis closure、destination Domain intent集合とset equalityでなければならない。各Mappingの`destination_owner_definition_ref`は`definition_kind=workload_domain_owner`、fixed destination Owner Definition Registry、allowed Domain集合へ同recordの`destination_domain_type_ref`を含み、Definition rowの`owner_ref`はMappingの`owner_ref`とbyte equalityである。生成する`WorkloadDomainIntentV1.owner_definition_ref`は選択Mappingの同Refをそのまま保存し、同じDomainを許可する別DefinitionやID prefixから再選択しない。Migration inputの`destination_owner_definition_registry_ref`は生成するEnvelopeの`workload_owner_definition_registry_ref`、Result／Prepared payloadの同Registry ref／hashとbyte equalityであり、全destination Intent／Owner intentのDefinition refはこの固定Registryだけへ解決する。destination Intent Kind Registryはinputのexact `destination_domain_registry_ref`が解決する`WorkloadDomainTypeRegistryV1.intent_kind_registry_ref`から決定し、生成Envelopeの`workload_intent_kind_registry_ref`とbyte equalityにする。独立したdestination-head lookupを行わない。Mapping Registry確定後、各Mapping refを`PerformanceQualificationSubjectV1(subject_kind=migration)`で署名し、root外`PerformanceQualificationBindingV1`を作る。Migration Manifestのbinding集合と選択Mapping集合はexact set equalityで、Receipt／BindingをMapping／Registry hashへ戻さない。0件／複数match、same source predicateへの複数active mapping、noncanonical sort、owner／policy／Domain／Definition／Type／Qualification BindingまたはReceipt hash mismatchを全migration rejectにする。

Activation後のlogical Operation ID候補はversion-neutral `operation.performance.migrate_project_scale_envelope`だけであり、同じatomic transactionのMCD／Manifest／Service allowlistへ登録する。現時点では三集合ともexact `[]`である。レビュー対象の旧綴り`operation.performance.migrate_project_scale_envelope_v1_to_v2`は一度もActivation／materializationされていない計画上の名前であり、alias、redirect、current refを作らない。将来、signed Inventoryが別の実在source spellingを証明した場合だけ、Tool catalog外のoffline alias migration recordとしてそのexact spellingを同じactivation transactionへ保存できる。

Activation後にOperationが参照する三Policyのdestination値は次の完全なactive MCD recordである。表の`status=active`はatomic activation完了後の値だけを表し、current Policy集合はexact `[]`である。表の共通Envelope列とpayload列を連結した値がrecord全体であり、別段落の既定値、bare ID、説明からFieldを補完しない。

| Policy MCD共通Envelope exact value | Policy payload exact value |
|---|---|
| `mcd_version=1; kind=policy; id=policy.operation.performance.scale_migration.precondition; version=1; status=active; title=Performance Scale Migration Precondition; description=Validate the exact V1 source envelope, Project base, workload registries, selected owner mappings, authorization, and idempotency snapshot without mutation; owners=[owner.core.performance]; requirement_refs=[]; rationale_refs=[mirakan.arch.runtime-performance-capacity#9-owner-typed-workload-scale-modelとprojectscaleenvelopev2]; since_contract_set=2; supersedes=[]; tags=[operation_predicate,performance,pure]` | `evaluation_mode=pure; side_effects=[]; input_type={id=type.operation.precondition_evaluation_input,version=1,contract_set_hash}; result_type={id=type.operation.predicate_result,version=1,contract_set_hash}` |
| `mcd_version=1; kind=policy; id=policy.operation.performance.scale_migration.postcondition; version=1; status=active; title=Performance Scale Migration Postcondition; description=Validate the unpublished V2 envelope, owner mapping closure, prepared Receipt payload, and atomic Project revision increment; owners=[owner.core.performance]; requirement_refs=[]; rationale_refs=[mirakan.arch.runtime-performance-capacity#9-owner-typed-workload-scale-modelとprojectscaleenvelopev2]; since_contract_set=2; supersedes=[]; tags=[operation_predicate,performance,pure]` | `evaluation_mode=pure; side_effects=[]; input_type={id=type.operation.postcondition_evaluation_input,version=2,contract_set_hash}; result_type={id=type.operation.predicate_result,version=1,contract_set_hash}` |
| `mcd_version=1; kind=policy; id=policy.authoring.performance_scale_migration.rate_limit; version=1; status=active; title=Performance Scale Migration Rate Limit; description=Bound migration requests per Project without changing migration semantics; owners=[owner.core.performance]; requirement_refs=[]; rationale_refs=[mirakan.arch.runtime-performance-capacity#9-owner-typed-workload-scale-modelとprojectscaleenvelopev2]; since_contract_set=2; supersedes=[]; tags=[authoring,performance,rate_limit]` | `policy_ref={id=policy.authoring.performance_scale_migration.rate_limit,version=1,contract_set_hash}; scope=project; window_ns=60000000000; max_requests=4; burst=1; exceeded_error_ref={diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash}` |

Activation destination Contract set内部では三Policyを`ContractSetLocalRefV1(kind=policy)`へ投影し、self refはlocal identityだけにする。Manifest `precondition_policy_ref`／`postcondition_policy_ref`／`rate_limit_policy_ref`、Operation三ref、Performance ownerのPolicy local subsetはexact三件でset equalityである。三recordの共通Envelopeまたはpayloadの実在Fieldを一つだけ変えるfixtureはPolicy member hashとset rootを変更し、旧Manifest／Operation external refを解決不能にする。

Activation後のmigration固有Diagnostic destination値は次の完全な`DiagnosticLocalRecordV2`である。current migration Diagnostic集合はexact `[]`である。全rowは`diagnostic_version=1`、`owner_local_ref=performance_owner_ref_v1`、`requirement_local_refs=[]`、`message_key="<diagnostic_id>.message"`、Ownerを含むself-excluding `diagnostic_local_content_hash`を持つ。root確定後だけ同じ三Field Owner ref、`requirement_refs=[]`、別のself-excluding `diagnostic_content_hash`を持つ外部Registry recordへ投影する。共通八件はActivation先Foundationの共通recordを参照し、本書のcurrent Diagnosticへ複写しない。

| Diagnostic ID | code | severity／category／retryability |
|---|---|---|
| `diagnostic.performance.scale_v1_source_invalid` | `MIRAKAN-PERFORMANCE-SCALE-V1-SOURCE-INVALID` | blocking／schema／after_input |
| `diagnostic.performance.workload_domain_unresolved` | `MIRAKAN-PERFORMANCE-WORKLOAD-DOMAIN-UNRESOLVED` | blocking／semantic／after_change |
| `diagnostic.performance.scale_migration_ambiguous` | `MIRAKAN-PERFORMANCE-SCALE-MIGRATION-AMBIGUOUS` | blocking／semantic／after_input |
| `diagnostic.performance.scale_receipt_binding_mismatch` | `MIRAKAN-PERFORMANCE-SCALE-RECEIPT-BINDING-MISMATCH` | blocking／semantic／after_change |

Activation後の`validator_closure.operation.performance.scale_migration@1` destinationは次のexact Validator recordで閉じる。current Validator／Validator closure集合はexact `[]`である。各recordはversion 1、実装Artifact ref／hash、表のinput Type LocalRef、表のDiagnostic LocalRef、self-excluding content hashを持つ。

| Validator | input | exact reachable Diagnostic |
|---|---|---|
| `validator.operation.request_envelope` | migration input | idempotency key reuse |
| `validator.operation.authorization` | migration input | authorization denied |
| `validator.operation.approval` | migration input | approval required |
| `validator.operation.revision_and_lock` | migration input | revision mismatch; lock conflict |
| `validator.operation.pure_predicate` | migration input | operation predicate invalid |
| `validator.operation.timeout_and_rate_limit` | migration input | timeout; rate limit exceeded |
| `validator.performance.scale_migration_semantics` | migration input | V1 source invalid; workload domain unresolved; migration ambiguous |
| `validator.performance.scale_migration_postcondition` | postcondition input v2 | Receipt binding mismatch |

wrong-kind、stale version／Contract set／content hash、impure policy、rate payload mismatch、Validator Artifact／input／error mismatchをManifest compile前に拒否する。

Activation Qualificationでは旧五axisを名前だけで自動変換しない。World axisはWorld Document／intentが実在する場合だけ`workload.world.spatial`、populationは対応owner mapping recordが一意な場合だけそのdomain、content／authoring／authorityは各Owner intentへ解決する。UI-only、headless tool、resource-only fixtureはWorld／Gameplay domain 0件のV2へ変換できなければならない。共通四fixture bodyは各選択Mappingをsubjectにする別`PerformanceQualificationSubjectV1(subject_kind=migration).fixture_refs[]`のexact四件としてだけ解決する。Activation後のProduction Manifestの`qualification_binding_refs[1..64]`は選択Mapping ref集合とexact set equalityで、各Bindingは一つのMapping refをsubjectにする署名済みReceiptへexact解決する。ManifestはReceipt refを直接保持せず、Fixture bodyを解決しない。Mapping数とfixture数を等置せず、1件または5～64件のvalid Mapping closureも同じ四fixtureを各subjectで検証できなければならない。

Activation後のInputの`source_foundation_definition_closure_ref`は`retained_source_mcd_ref={type.performance.project_scale_envelope,1,source_contract_set_hash}`、signed Inventoryが列挙したV1 Envelope record、そのOwner ref、同時代Named Algorithm Registryをexact source Closureへ解決する。InputとPrepared payloadのsource Closure refおよび`retained_source_mcd_ref`はbyte equalityで、Operation／input／output／Policy／Validator／Diagnostic、全destination Registry／Mapping policy／V2 schema、request Algorithm bindingはdispatch時のdestination Foundation Closureだけへ解決する。両Fieldはoperation intentのsemantic inputとPublic Receiptから到達するPrepared payloadへ保持し、missing、別名、wrong source root、同じContract Setの別Owner／Algorithm root、input–Receipt差、sourceをdestinationへalias、destination refのsource downgradeを一原因ずつrejectしてSourceと既存idempotency resultを不変にする。

Activation Qualificationは[Executable Contracts §8](../02-foundation/executable-contracts.md#8-operation定義)のcanonical publicationを再利用し、`Source → Preview → Validation → Prepared payload → private Marker read-back → secret-free PublicCommitClosureV1 candidate → signed wrapper read-back → PublicCommitClosureV1＋PublicPublicationMarkerV1＋after Projectのatomic CAS → reload → Compile`を検証する。Closureの`domain_commitment.kind`は`owner_typed_state_commit`、`domain_owner_ref`はexact `performance_owner_ref_v1`、committed artifact集合はPrepared payloadが束縛したreceipt-free artifact ref集合とし、Closure Ref／hashの生成・比較は同節をそのまま使って本書で別式を定義しない。0件／複数mapping、偽World生成、Gameplay floor捏造、partial migrationをrejectする。destination Owner Definition Registry ref／version／hashの一Fieldだけをinput、Envelope、Result、Prepared payloadのいずれかで差し替えるcase、Domain Registry内Intent Kind Registry refとEnvelopeのrefをずらすcase、retry時にRegistry driftから別Envelopeを生成するcaseを一原因ずつrejectし、Sourceと既存idempotency resultを不変にする。Operation `errors[]`、Validator reachable errors、Manifest `diagnostic_refs[]`は上記12件のID／code／version／content hashでset equalityにする。ManifestのOperation LocalRef集合と`service.offline_project_migrator`へのallowlist contributionはexact一件でset equalityとし、同じatomic activation transactionでService local recordとset rootを生成する。Prepared payload Typeはexact `type.performance.prepared_project_scale_envelope_migration_receipt_payload@1`、hashはASCII `MIRAKAN_PREPARED_PROJECT_SCALE_ENVELOPE_MIGRATION_RECEIPT_PAYLOAD_V1`とself-excluding canonical bytesから計算する。最終Receipt Typeと相互代用しない。唯一のsigned subject／wrapperはExecutable Contractsの`PublishedDomainReceiptPayloadV2`／`PublishedDomainReceiptV2`とする。Domain固有Subject／alternate wrapperを作らず、Closure bodyまたは同Closureを束縛するsigned wrapperを欠くPublic Marker／after-state current authorityを拒否する。同じidempotency key＋request hashのretryはbyte-identical Result／`PublicCommitClosureV1`／signed Receipt／Public Markerを返し、同じkey＋別requestはidempotency reuse errorでSourceを変更しない。

Project固有の同時workload製品Envelopeは現時点で未校正であり、数値を仮定しない。この項目のOwnerは本書、readiness envelopeのOwnerはProject Stateである。Target Profileごとに次の入力が揃うまでは`state=blocked`、`blocked_reason_ref=performance_envelope_unqualified`を返す。

1. CPU世代／core、RAM、storage、GPU／driver、OS、Device generationを固定した実機Target Profile。
2. `ProjectScaleEnvelopeV2`が参照する全`WorkloadDomainIntentV1`について、選択domainが登録したinstance、lifecycle、event、spatial、presentation、UI、tool、resource dimensionのboundを数値化したProject Requirement。
3. その数値を丸めず同時発生させる`IntegratedScaleFixtureV1`とcanonical input trace。
4. Source、Contract、Toolchain、Target、Device、Quality、Representation Planを束ねた同一`input_closure_hash`。
5. §8／§13のcorrectness、Replay、memory、hitch、fault、10分×3 run、2時間enduranceを通過したfresh `policy.evidence.target-device.v1` Technical Qualification Receipt。

上記Receiptが同じclosureでfreshな場合だけ`qualified`へ遷移できる。安全なRepresentation Planは作れるが製品Envelopeとは別の小規模入力を測定しただけなら、その小規模入力closureに限り`predicted`または`qualified`を判定し、製品workload closure全体へ外挿しない。Mobile commonが所有するbaseline pixel／render budget表は変更せずTarget Profile入力として保持するが、それ単独で他domainのreadinessを解除しない。

Scale dimensionは次の型付きregistryで所有する。CoreはGenre／object role／event名を列挙せず、Feature Pack、Genre Pack、Projectが自身の語彙を同じcontractへ寄与する。

```text
RuntimeScaleIntentDimensionRefV1 {
  dimension_id,
  dimension_version,
  dimension_content_hash
}

PerformanceScaleUnitSemanticRefV1 {
  unit_id,
  unit_version,
  unit_content_hash
}

PerformanceScaleUnitSemanticRecordV1 {
  unit_id,
  unit_version,
  owner_ref: OwnerIdentityLocalRefV1,
  quantity_kind: count | distance,
  canonical_scalar_encoding: uint64_be | ieee754_binary64_be,
  canonical_base_unit: one_count | si.meter,
  canonical_symbol: count | m,
  conversion_to_canonical: exact rational scale and offset,
  value_constraint: nonnegative_integer | nonnegative_finite,
  unit_content_hash
}

PerformanceScaleUnitSemanticRegistryRefV1 {
  registry_id,
  registry_version,
  registry_content_hash
}

PerformanceScaleUnitSemanticRegistryV1 {
  registry_id,
  registry_version,
  record_count,
  records[2..256],
  registry_content_hash
}

RuntimeScaleIntentDimensionRecordV1 {
  dimension_ref,
  owner_ref: OwnerIdentityLocalRefV1,
  measurement_schema_ref: McdContractRefV1(kind=type),
  unit_ref: PerformanceScaleUnitSemanticRefV1,
  authority_class,
  fidelity_contract_ref?: McdContractRefV1(kind=policy),
  semantic_equivalence_contract_ref?: McdContractRefV1(kind=policy),
}

RuntimeScaleIntentDimensionRegistryRefV1 {
  registry_id,
  registry_version,
  registry_content_hash
}

RuntimeScaleIntentDimensionRegistryV1 {
  registry_id,
  registry_version,
  unit_registry_ref: PerformanceScaleUnitSemanticRegistryRefV1,
  records[1..4096],
  registry_content_hash
}
```

Performance Scale Unit Semantic RegistryはRuntime scale envelopeで使う量のwire意味だけを所有し、Math型、Editor表示単位またはPack固有の換算表を所有しない。logical IDは`registry.performance.scale_unit_semantic`、current `registry_version=1`、`record_count=2`、`registry_content_hash=1337af0489705db3c743029e23b9ba7a0f31d02eb4aeb4bd81a33b5f07de1bde`である。初期recordは次のexact二件で、両方の`owner_ref`は`performance_owner_ref_v1`、`conversion_to_canonical={scale_numerator="1", scale_denominator="1", offset_numerator="0", offset_denominator="1"}`とする。

| `unit_id` | version | quantity／wire／base／symbol | value constraint | `unit_content_hash` |
|---|---:|---|---|---|
| `unit.count` | 1 | `count`／`uint64_be`／`one_count`／`count` | `nonnegative_integer` | `f1cf9acc828548cf9a1b8eac19f5994f390944b86d3868e72c4ea6ead8b2b514` |
| `unit.meter` | 1 | `distance`／`ieee754_binary64_be`／`si.meter`／`m` | `nonnegative_finite` | `e35e5f3740d7b69384965fde3173ad82c88ba09b7c3d6794cdf6aa7d09057227` |

`unit_content_hash`はASCII `MIRAKAN_PERFORMANCE_SCALE_UNIT_SEMANTIC_RECORD_V1`と、当該hash Fieldだけを除くclosed recordのRFC 8785 JCS UTF-8 bytesを`uint32_be` length framingしてSHA-256する。JCS projectionではkeyをUnicode code point順、全stringをNFC、`unit_version`をsafe JSON integer、`owner_revision`とrational四値をcanonical unsigned decimal string、SHA-256をlowercase hexadecimal exact 64文字にする。`registry_content_hash`はASCII `MIRAKAN_PERFORMANCE_SCALE_UNIT_SEMANTIC_REGISTRY_V1`と、当該hash Fieldだけを除き`records[]`を`unit_id`のNFC UTF-8 bytes／version／content hash順へstrict sortしたclosed Registryの同じJCS framingから計算する。上記literal hashはこのpreimageから再計算できなければならず、ID-only、表示symbol、latest versionまたは別Registry rootによる補完を禁止する。将来単位を追加する場合は新Registry versionとexact record hashを発行し、Project／Pack固有表示変換は別Owner contributionとしてQualificationする。

`dimension_id`はowner namespaceを含むversion非依存logical ID、`dimension_version`は正の`uint32`とする。`dimension_content_hash`はASCII `MIRAKAN_RUNTIME_SCALE_INTENT_DIMENSION_RECORD_V1`と、当該hash Fieldだけを除くReceipt-free Record canonical MCD bytesを`uint32_be` length framingしてSHA-256する。`authority_class`は`authoritative_state | authoritative_event | presentation_only | resource_only`のclosed enumとする。`records`は`dimension_id`のUTF-8 byte昇順、同一IDまたは同一Ref重複を拒否し、RefはRegistry内でちょうど一件へ解決する。`authoritative_state | authoritative_event`は`semantic_equivalence_contract_ref`必須、`presentation_only`は`fidelity_contract_ref`必須、該当しないoptionalはcanonical omissionする。Dimension Registry確定後に各Dimension refを`PerformanceQualificationSubjectV1(subject_kind=scale_dimension)`へbindし、signed Receiptとroot外Qualification Bindingを作る。Receipt／BindingをDimension record／Registryへ戻さない。

Registryのlogical IDは`registry.performance.runtime_scale_intent_dimension`、initial `registry_version=1`とし、`unit_registry_ref`はexact `{registry.performance.scale_unit_semantic, 1, 1337af0489705db3c743029e23b9ba7a0f31d02eb4aeb4bd81a33b5f07de1bde}`とする。`registry_content_hash`はASCII `MIRAKAN_RUNTIME_SCALE_INTENT_DIMENSION_REGISTRY_V1`、Registry ID／version、unit Registry ref、record count、strict sort済み全Record canonical bytesを各`uint32_be` length framingしてSHA-256し、自己Fieldを除外する。`RuntimeScaleIntentDimensionRegistryRefV1`は三Fieldすべてを同一active Registryへexact解決し、ID-only、latest version、hash fallbackを許可しない。

Core-owned初期Recordは次のexact九件だけとする。表の`count-bound`は`measurement_schema_ref={type.performance.bounded_count, version=1, Contract set hash}`／`unit_ref={unit.count, 1, f1cf9acc828548cf9a1b8eac19f5994f390944b86d3868e72c4ea6ead8b2b514}`、`distance-bound`は`{type.performance.bounded_distance, version=1, Contract set hash}`／`unit_ref={unit.meter, 1, e35e5f3740d7b69384965fde3173ad82c88ba09b7c3d6794cdf6aa7d09057227}`を表す。全Receipt-free Recordの`owner_ref`はexact `performance_owner_ref_v1`であり、文書revision／文書content hashまたはactive Ownerの自動追従値へ置換しない。

| `dimension_id` | schema | `authority_class` | required contract |
|---|---|---|---|
| `scale.dimension.instance.total_authored` | count-bound | `resource_only` | optional refs omitted |
| `scale.dimension.instance.peak_live` | count-bound | `resource_only` | optional refs omitted |
| `scale.dimension.instance.peak_active_authoritative` | count-bound | `authoritative_state` | `policy.performance.authoritative-state-equivalence` |
| `scale.dimension.instance.peak_visible` | count-bound | `presentation_only` | `policy.performance.presentation-fidelity` |
| `scale.dimension.lifecycle.peak_create_per_simulation_step` | count-bound | `authoritative_state` | `policy.performance.authoritative-state-equivalence` |
| `scale.dimension.lifecycle.peak_retire_per_simulation_step` | count-bound | `authoritative_state` | `policy.performance.authoritative-state-equivalence` |
| `scale.dimension.event.peak_authoritative_per_simulation_step` | count-bound | `authoritative_event` | `policy.performance.authoritative-event-equivalence` |
| `scale.dimension.spatial.maximum_interaction_radius` | distance-bound | `authoritative_state` | `policy.performance.authoritative-state-equivalence` |
| `scale.dimension.presentation.peak_active` | count-bound | `presentation_only` | `policy.performance.presentation-fidelity` |

九Dimension refに対応するroot外Qualification Bindingは、同じ意味を共有するReceiptを許可してもentryごとにexact subject refを持つ。Activation projectionの`scale_dimension_qualification_binding_refs[]`が解決するsubject集合と、Envelope全Intentの`dimension_values[].dimension_ref` unionをexact set equalityにし、subject ref／Binding refのcanonical順とduplicate不在を検証する。count-bound等のReceipt IDだけから複数baseへ暗黙展開せず、Dimension missing／extra／別Envelope substitutionを一原因fixtureで拒否する。

三policy refは本書のexact revision／content hashを持ち、`authoritative-state-equivalence`は§12のstate／random-stream gate、`authoritative-event-equivalence`はowner schemaごとのevent ID／apply step／canonical payload hash／ordering equality、`presentation-fidelity`はowner fixtureのvisual／audio tolerance、critical cue、timing floorを要求する。これらを説明名だけのpolicy、別revisionの同ID、Genre固有の暗黙比較へ置換しない。

Project固有のunit群、役割別instance数、イベント別peak、車両／群衆／弾体等のobject分類はCore enumへ追加せず、所有Pack／Projectが新しいRecord、measurement schema、fidelity／equivalence、別Qualification record／signed Receiptを一緒に登録する。Production recordはFixture bodyへ依存しない。登録されていないdimension、owner不一致、schema不一致、hash不一致は`ambiguous requirement`として拒否する。

```text
BoundedScaleQuantityV1 {
  unit_ref,
  minimum_required,
  target_value,
  maximum_expected
}

RuntimeScaleIntentDimensionValueV1 {
  dimension_ref: RuntimeScaleIntentDimensionRefV1,
  measurement_schema_ref: McdContractRefV1(kind=type),
  quantity: BoundedScaleQuantityV1
}

LegacyRuntimeScaleIntentV1 {
  intent_id: UUIDv7 StableId,
  schema_version: uint32,
  owner_definition_ref:
    exact {definition_id, definition_version, definition_content_hash},
  dimension_values[1..64],
  target_profile_refs[1..16]: exact Target Profile refs,
  fidelity_floor_refs[1..64]: McdContractRefV1(kind=requirement),
  semantic_equivalence_refs[0..64]: McdContractRefV1(kind=policy),
  reference_fixture_refs[1..64]:
    exact {fixture_id, fixture_version, fixture_content_hash},
  intent_content_hash: SHA-256
}
```

`LegacyRuntimeScaleIntentV1`は§9のpost-activation source schema templateで、current retained artifact／Source／Editor／AI／Compile／Save／Replay／Registry集合はexact `[]`である。signed `LegacyMigrationInventoryV1`が旧`ProjectScaleEnvelopeV1`とIntent bytesを束縛するまでoffline migration inputとして受理しない。旧logical type `RuntimeScaleIntentV1`をcurrent aliasとしてdeserializeせず、Activation後も`ProjectScaleDomainMappingRecordV1`がexact一件対応する場合だけcanonical `WorkloadDomainIntentV1` candidateへ変換する。新しいauthorityはdomain-discriminated `WorkloadDomainIntentV1`だけである。

current `WorkloadDomainIntentV1.dimension_values[]`はEnvelopeのexact Dimension Registryへ`dimension_ref`を解決する。Activation後のlegacy `dimension_values`はsigned Inventoryが束縛したsource Envelopeのexact Dimension Registryだけへ解決する。各Valueの`measurement_schema_ref`は解決先`RuntimeScaleIntentDimensionRecordV1.measurement_schema_ref`、`quantity.unit_ref`は同recordの`unit_ref`と全Field byte equalityで、さらにDimension RegistryがpinしたPerformance Scale Unit Semantic Registry内のexact一件へ解決しなければならない。三quantity値はそのmeasurement schemaでcanonical decode後に再encodeして入力canonical bytesと一致し、同じsemantic typeのfiniteかつ非負値、`minimum_required <= target_value <= maximum_expected`であることを検証する。schemaがcountならinteger count、distanceならfinite SI meterというrecord固有制約も同じdecoderが適用し、unit表示名やdimension IDから変換・補完しない。

配列は`dimension_id`のNFC UTF-8 byte／version／content hash順でstrict sortし、同一dimensionを拒否する。正しいDimension refのままcount schemaをdistance schemaへ、meter unitをcount unitへ、quantityの一値を非canonical encodingへ、Registryだけを別valid versionへ差し替えるfixtureをcurrent Envelopeで一原因ずつrejectする。legacy側の同等fixtureはActivation Qualificationにだけ属し、migration Sourceとlast-valid Resultを不変にする。Activation後のlegacy `intent_content_hash`はASCII `MIRAKAN_RUNTIME_SCALE_INTENT_V1`と、自身だけを除いた全Fieldのcanonical MCD bytesを`uint32_be` length framingして検証するが、新規生成しない。Target／fidelity／Fixture refはmigration Qualificationだけが読み、destination Production intentへFixture bodyをコピーせずsigned Receiptへ置換する。unknownを0、最大値、空optional、無制限へ補正しない。

World extent／coordinate／cell／streaming fieldは[World](../06-rendering/world.md)、LOD strategy／predicate／transition fieldは[LOD](../06-rendering/lod.md)、Authoring writer／Document／ChangeSet fieldは[Project state](../03-authoring/project-state.md)、content／build／cook fieldは[Asset lifecycle](../03-authoring/asset-lifecycle.md)と[Core architecture](../02-foundation/core-architecture.md)が所有する。本書はそれらをEnvelopeへexact refで束ねるだけで、field listを複写しない。

Envelope変更は通常の`ProjectChangeSetV1`であり、Before／After axisとdimension値、Target、Capability、Artifact、fixture、Decision closure、fidelity差分、再Cook／再Qualification、Save／Replay互換性、last-valid rollback refを必要とする。owner-typed authoritative instance／event bound、semantic-equivalence requirement、registered collision evidence、World範囲等を下げる変更は性能最適化ではなくGame behavior changeとして人間承認を必要とする。

## 10. Canonical sourceとDomain resolver

Scaleの四層を分離する。

| layer | content | mutation |
|---|---|---|
| Canonical Source | Requirement、World、Entity、Asset metadata、GameplayDefinition、Scale Intent、Decision | ChangeSetだけ |
| Derived Plan | streaming、representation、residency、LOD、cook work | direct edit禁止 |
| Runtime State | generation、queue、active／resident set、chunk | Runtime ownerだけ |
| Evidence | Trace、Benchmark、Diff、Explanation、Qualification | append-only、Governance envelope参照 |

Runtime StateまたはEvidenceからSourceへ値を自動write-backしない。Stable IDはrename、repartition、recook、HLOD、instance化、Simulation LODで変えない。Runtime handle、vendor ID、cell-local／plan-local IDをSource／Saveへ保存しない。Save／Replayはexact Source、Contract、Envelope、Plan hashを使う。

万能な`ScaleManager`を作らない。各Domainは同じEnvelopeと自身のIntentを読み、自身のDerived Planを所有する。`ScalePlanSetV1`はplan本文を埋め込まず、exact Artifact ref、Source revision、Target Profile、Capability signature、dependency edge、`TargetReadinessV1` refを束ねるmanifestである。required planのSource／Contract／Target／Capability hashが一件でもstale、missing、unqualifiedなら新setをpublishせずlast-valid playable setを維持する。

Population resolverはFull Entity、pool、archetype／SoA、instanced Presentation、reduced-frequency simulation、dormant record、aggregate simulation、HLOD、presentation-effect Artifactからclosed strategyを選ぶ。各strategyはentry／exit predicate、owner、state／Save mapping、recovery、fallback、budgetを持つ。distance／visibilityだけでauthoritative instanceを削らない。

## 11. AI scale action候補とbounded explanation

`search | read_envelope | dependencies | resolve_preview | explain_plan | propose_envelope_change | validate_transition`はStable IDでないplanned semantic action vocabularyであり、registered MCD Operationではない。Performance ownerのcurrent MCD Operation集合はexact `[]`である。`operation.performance.migrate_project_scale_envelope@1`は§9のdestination templateを持つconditional legacy migration exact一件、Scale AI actionは別の未Activation候補であり、どちらもcurrent Operationへ数えない。Scale AI actionのcurrent MCD／Owner Manifest／Service allowlist／Policy／Validator／Diagnostic／Receipt／Provider／MCP／alias集合はすべて`[]`、Capability stateは`not_activated`とし、future work item `activation.performance.scale_ai_operations.v1`が採用するexact ID集合と完全closureを一transactionで登録するまでdispatchしない。将来ActivationしたQuery／read／explainは[AI Security／Approval](../01-governance/ai-security-approval.md)が許可するread-only範囲、preview／changeは同OwnerのRisk／Approvalを消費する。本書はRisk値を再定義しない。

ProviderへProject Commit、Plan write、Capability activation、baseline緩和、Source直接write、server authority移動を公開しない。Queryはrevision、Envelope hash、Target、Capability signature、index revision、query hash、selected items、omitted ranges、cursor、Governance Evidence refを返す。全World／全Project dumpを行わない。

`ScaleExplanationReceiptV1`はSource revision、Envelope hash、Target、Plan set hash、selected／rejected closed strategy、fidelity proof ref、cost measurement ref、fallback chain、`TargetReadinessV1` ref、invalidation conditionをDomain evidenceとして生成する。共通Receipt envelope、Provenance、署名、保持は[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を参照し、本書で再掲しない。

algorithm optimizationの説明は§8.4の`OptimizationDecisionProjectionV1`だけを消費し、raw benchmark logや設計文書の全量をAIへ投入して不足Fieldを推測させない。Project Source／文書抜粋はread-only contextであり、Projectionまたはregistered Operationの代用ではない。AIの説明、Tool visibility、Provider schema適合はselection、promotion、Commit authorityを付与しない。

最低Diagnosticはmissing envelope、ambiguous requirement、unqualified capability、stale plan、fidelity violation、invalid reference closure、budget exceeded、partial activation rejected、distributed authority not activatedを区別する。unknownを近いenum、0、最大値、current Target defaultへ補正しない。

## 12. MediumからLargeへの非破壊遷移

許可する変更は、Partition Intent／Target追加、同じSourceからのinstance／batch／HLOD／Simulation LOD生成、Asset closure分割、Authoring re-shard、Build work item分割、Target別residency／fallback追加である。Source Stable ID、Save schema、Gameplay authorityを変えず、Derived Planだけを置換する。

禁止する変更は、Large専用owner typeへのSource一括変換、Medium／Large別Save fork、cell／shard／build／server IDの混同、HLOD／GPU instanceのSave authoritative record化、unqualified planのProduction表示、Medium fallback削除、性能のための無承認authoritative semantics変更である。

同じSource revisionとinput traceに対するMedium／Large planは、Save field／Stable ID、Input→Command→Event順序、registered runtime-entry／transition outcomeを一致させる。authoritative stateの同値Gateは二層とする。(a) 両planでSimulation LODを適用しないfull fidelity対象entityは、[Persistence／Save](persistence-save.md)の`RuntimeAuthoritativeStateDigestV1`が定めるsealed World publicationで採取したentity projection hashと、当該entityへ帰属するdeterministic random stream消費を同一`advance_sequence`で一致させる。(b) いずれかのplanでSimulation LODを適用するentityは、[LOD](../06-rendering/lod.md)の`authoritative_equivalence_contract`と`reference_fixture_id`により、active owner schema registryが定めるauthoritative state／event outcome、registered collision／navigation evidence、wake後のstate収束をsemantic同値として判定する。full fidelity対象集合は両planのSimulation LOD適用集合の補集合として決定的に導出し、runごとに変えない。Presentation bitwise一致は不要でも、visual／audio tolerance、critical cue、event timing、fallback Gateを満たす。

Large World coordinate、continuous streaming、partition-owned multi-writer、distributed build、distributed simulation／authorityは専用Owner仕様がactivationされるまで`not_activated`である。現在のbounded Sourceへ空Manager、server field、RPC、global double座標を先回り追加しない。要求された場合は明示Diagnosticでfail closedする。

## 13. Integrated fixtureとqualification

共有Contract `IntegratedScaleFixtureV1`は本節だけが所有し、各ProjectのEnvelopeとactive `RuntimeScaleIntentDimensionRegistryV1`から、全owner-typed instance dimension、create／retire lifecycle dimension、authoritative event dimension、Physics／Navigation／Animation、Game System、presentation effect、Audio、view、streaming、LOD、Asset activationを、実際に同時発生し得る一つのdeterministic integrated fixtureへ生成する。Subsystem最大値を別runへ分離して同時性を隠さず、Compilerは宣言された`maximum_expected`を丸めない。未登録dimensionまたはfixture recipe欠落はqualification開始前に拒否する。

fixtureは次を全て満たす。

1. frame、Subsystem、memory、queue、GPU resource、streamingのhard Gate。
2. registered authoritative create／retire／state／event record drop 0。
3. 各workload ownerが登録したauthoritative state／event、functional result、resource SLO、Replay oracleがreferenceと一致。
4. Presentation degradationがpriority、style、critical cue floorを満たす。
5. registered lifecycle burst、streaming boundary、presentation-effect burstのP99.9がdeadlineを満たす。

`medium_candidate / qualified`にはProject固有Envelope、exact runtime-entryと選択workload owner closure、必要な場合だけWorld／spatial closure、Save／Replay／Package、content totalとactive working setの分離、incremental Import／Cook、Target budget、2時間endurance、migration／load、bounded AI edit、last-valid recoveryを必要とする。

`large_local_candidate / qualified`にはMedium Gateに加え、利用するLarge Capabilityの専用仕様／Receipt、Project固有traversal／population trace、partition boundary／reference closure／load deadline／memory pressure／recovery、same-source Medium fallback、repartition後のStable ID／Save／Replay、bounded context、incremental／partial Cook同値、10分×3 run、2時間endurance、failure injectionを必要とする。

本節のqualification計測run（§8のScale qualification 10分×3 run、Production endurance 2時間runを含む）は、`predicted` TargetのDevelopment Play実行モードで実行できる（[Project state](../03-authoring/project-state.md#9-runtime-compile境界)）。計測run自体の開始に`qualified`を要求せず、同じ`input_closure_hash`へ束縛されたfresh Receipt確定後にだけ`qualified`へ昇格する。`blocked_reason_ref=performance_envelope_unqualified`の製品Envelopeは、§9の5入力を揃えた専用qualification harnessだけを開始でき、通常Development Playを許可しない。

Distributed qualification Gateは本書でactivationしない。専用Authority仕様、Threat Model、server実機、loss／latency／abuse／recovery fixture、人間承認が揃うまでCatalogへactive Gateを掲載しない。

## 14. Failure、CI、completion

| failure | behavior |
|---|---|
| Envelope不足／曖昧 | Blocking Diagnostic。値を推測しない |
| Plan compile／migration失敗 | new plan非publish、Sourceとlast valid維持 |
| activation dependency不足 | authoritative closure全体をinactiveにする |
| stale Source／Target／Contract | result破棄、current revisionで再計画 |
| budget超過 | `state=blocked`、改善可能なら`blocked_reason_ref=optimization_required`。fidelityを自動緩和しない |
| Project workload Envelope未校正 | `state=blocked`、`blocked_reason_ref=performance_envelope_unqualified`。Target実機fixtureとfresh Receiptまで数値を発明しない |
| Presentation Artifact不足 | approved visual fallback、Gameplay Source維持 |
| Simulation LOD restore失敗 | Full／last valid fallback、不可能ならactivation拒否 |
| partial Cook／Package失敗 | last valid package維持 |
| unactivated Authority | fail closed、意味同等single-process alternativeだけ提示 |

CIはbudget hard limit、loan deadline、queue pressure、§5のqueue表から導出したcommit合計と記載値の不一致、missing metric、SourceへのRuntime／Derived ID、stale plan／Receipt、owner requirement低下、partial activation、Presentation→authoritative owner逆入力、Medium fallback欠落、unactivated Authority公開を拒否する。加えて、workload未校正なのに`performance_envelope_unqualified`以外を返すこと、PascalCase readiness、`optimization_required`／`not_activated`のstate混入、`blocked`でreason欠落、fresh Target-device Receiptなしの`qualified`を一原因ずつnegative fixtureで拒否する。

本書のcompletionには、共通budget／capacity／backpressureが一意、`ProjectScaleEnvelopeV2`とowner-typed workload registryが一意、Worldなし／UI-only／headless／tool／resource-only fixtureがvalid、Domain fieldのowner委譲が明示、same-source transition fixture、bounded explanation、Target qualification、last-valid recoveryが実行可能であることを必要とする。Product Phase順序、Capability maturity、Governance authorization、Evidence envelopeを本書で再定義しない。

## 15. 明示的に採用しないもの

- average fps、predicted cost、単一microbenchmarkだけによるProduction合格。
- external version／driver値のPerformance文書への複写。
- budget超過時の一般heap fallback、Emergency／headroomの通常利用、無期限loan。
- unbounded queue、authoritative drop、priorityのpayload由来昇格。
- frame generationをreal frameとして数えること、quality／Gameplay低下による見かけの改善。
- `large=true`、総file数、GenreだけによるScale判定。
- 万能Scale owner、SourceとDerived Plan、Runtime State、Evidenceの混在。
- World／LOD／Authoring固有fieldの本書での再定義。
- Medium／Large別Source／Save、repartitionによるStable ID変更。
- 未Activated distributed Authority／World／Authoring／Buildの空実装。
- raw benchmark／log／設計文書／全Project dumpをAIへ渡してcandidate、threshold、Receipt、選択理由を補完させること。
- AIの自己評価によるpromotion、runtime auto-tuning、全Target／Profileへ一つの高速candidateを既定化すること。
