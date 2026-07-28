# Miraikanai Engine Runtime Performance／Capacity Contract

- 文書ID: mirakan.arch.runtime-performance-capacity
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: 共通CPU／GPU／memory／queueの暫定budget、capacity、reservation／loan、backpressure、worker capacity、測定法、regression、Owner横断algorithm optimization candidate qualification
- 非正本範囲: Runtime phase／Simulation Advance／lifetime、ECS storage／query／digest field、Runtime Package binary、Save／Replay record、World cell／coordinate field、LOD policy field、Authoring Document／ChangeSet field、Domain固有budget、外部Tool／SDK／driverの固定値、AI承認、Evidence envelope。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Scheduling／Lifetime](scheduling-lifetime.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)
- 関連文書: [Performance Scale Catalog Proposal](../appendices/performance-scale-catalog-proposal.md)、[Runtime ECS](entity-component-system.md)、[Runtime Package](runtime-package.md)、[Persistence／Save](persistence-save.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core architecture](../02-foundation/core-architecture.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Math／Core utilities](../02-foundation/math-core.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Project state](../03-authoring/project-state.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Gameplay programming model](../03-authoring/gameplay-programming-model.md)、[Scheduling／lifetime](scheduling-lifetime.md)、[Debugging／observability／replay](debugging-observability-replay.md)、[World](../06-rendering/world.md)、[LOD](../06-rendering/lod.md)、[Mobile common](../07-platform/mobile-common.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-26

## 1. 結論とauthority

測定法、Budget Scope、backpressure意味、将来の共通budgetを決定する手続きは本書だけが所有する。Subsystem Ownerは、採用済みparent budgetが存在する場合にその範囲内の固有配分とquality fallbackを所有する。Runtimeのphase／Simulation Advance／lifetimeは[Scheduling／lifetime](scheduling-lifetime.md)だけが所有する。

根拠: provisional — 2026-07-27時点でEngine実装、Target fixture、Benchmark executable、Measurement ReceiptはRepositoryに存在しない。本文のmemory、GPU、queue、worker、frame、latency、件数、時間、percentageは初期測定profileを設計するための候補値であり、製品が達成済みのBudget、公式推奨、Shipping上限ではない。

候補値を`project-decision`へ昇格するには、Target、hardware、OS、Toolchain、Source revision、fixture、Quality、sample選択、計測結果を固定したEvidenceと、同じ条件での再現結果を必要とする。計測前に値を`current baseline`、`qualified`、`supported scale`と表現しない。

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

本節の全MiB値、percentage、loan期限、frames-in-flightは`provisional`である。表は初回計測のcharge漏れを検出するための候補分解であり、実測済み配分ではない。

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

本節の全MiB値と割合は`provisional`である。Platform-reported budget、実際のresource構成、driver挙動を測定するまでTarget capabilityとして公開しない。

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

次は共通C1 queue／buffer capacityの`provisional` profileである。初回Prototypeではこの値をstress inputとして使用できるが、hard reservationや製品capacityとして公開しない。採用後にProjectが変更する場合はmemory envelope、stress、Replay、Domain qualificationを再確認する。Runtime contract固有のdeterministic上限（[Scheduling／lifetime](scheduling-lifetime.md) §4.1のGameplay Timer active／fire上限等）は各Owner文書が所有し、本表へ複写しない。本表から同時resident／visible Entity、owner-typed authoritative／presentation instance、lifecycle burstの製品capacityを逆算しない。

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

本節のfps、millisecond、percentageは`provisional`であり、Target別Measurement Receiptがない状態では合否判定、対応表明、Release Gateに使用しない。

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

[Editor UI Framework §15.8.5](../appendices/editor-ui-design-system-catalog.md#1585-comparison-profile-registry)の`comparison.editor.performance.absolute-and-regression@1`が消費するPerformance Owner artifactを次で閉じる。これはReference 01専用の計測定義であり、一般Game Runtime、Build、Importのworkloadへ流用しない。

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

`qualification.editor.performance-reference@1`はA／B×三workloadのexact六workload result、exact六soak result、exact六approved baselineを持ち、全absolute thresholdとrelative regressionがpass、同じCandidate／Contract／Toolchain、current `VerificationReceiptV1`に閉じる場合だけpassにする。六`approved_baseline_refs/hashes`は[Editor Workspace UX §6.9.1](../appendices/editor-panel-reference-catalog.md#691-fixtureeditorreference-011-concrete-manifest)のG00 open A／B、G01 steady-state A／B、G08 capacity A／Bの六Performance coverage entryと一対一に対応し、[Editor UI Framework §15.8.6](../appendices/editor-ui-design-system-catalog.md#1586-baseline-changeevidence-bundle署名)のcurrent Baseline Registry Publicationにあるexpected subject ref/hashだけを受理する。手入力の数値、直前run、threshold file、`latest` artifactをapproved baselineへ読み替えない。三Workload Profile、capacity corpus、setup／terminal barrier、sample、aggregation、三Threshold Set、soak、A／B approved baselineの全ref/hashが同じContract setで解決するまで通常Performance resultをpassにしない。現時点でA／B hardware profileが`characterization_only`である間はHarness Qualification、Comparison Profile Registry、Initial Baseline Execution Definitionをmaterializeせず、数値表を実測Receiptへ読み替えず、関連Capabilityを`not_activated`に保つ。

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
RuntimeEcsInitialAcceptanceThresholdSetV1
  threshold_set_version: 1
  target_profile_ref:
    exact {target_profile_id, target_profile_version,
           target_profile_content_hash}
  selected_chunk_payload_bytes: 16384
  scenario_limits[6]:
    sorted unique by scenario_id
    scenario_id:
      sequential_motion | position_only_projection | lifetime_only_scan
      | structural_burst | archetype_fragmentation
      | query_cache_invalidation
    scenario_cpu_p95_ns_max: positive uint64
    scenario_memory_peak_bytes_max: positive uint64
  handle_resolve_p95_ns_max: positive uint64
  lease_validation_p95_ns_max: positive uint64
  threshold_set_hash: SHA-256

RuntimeEcsInitialAcceptanceThresholdSetRefV1
  threshold_set_version: positive uint32
  target_profile_ref:
    exact {target_profile_id, target_profile_version,
           target_profile_content_hash}
  threshold_set_hash: SHA-256

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
  initial_acceptance_threshold_set_ref:
    exact RuntimeEcsInitialAcceptanceThresholdSetRefV1
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

initial threshold setは初回selected 16 KiB layoutの絶対memory／latency acceptanceを所有する。値はlocked Reference Hardware Profile上のcharacterizationと独立Reviewから承認し、Markdownの推測値、別Target、別device、相対差、0 baseline、Product deadlineから生成しない。Target Profileが未完成、scenarioがexact 6件でない、上限が0／欠落、またはthreshold setが未承認の場合は`RuntimeDataOrientedQualificationProfileV1`をmaterializeせず、Phase GateとCapabilityをpassにしない。8／32 KiBはinitial campaignでcharacterizationするが、このthreshold setを満たしたことをselected layoutのpassまたは自動切替へ使わない。

profileはCandidateを含まない。campaignの`ArtifactCandidateBindingV1` Target member、Contract Set、Toolchainはprofile、initial threshold set、sample artifact、correctness artifactとbyte-equalでなければならない。Contract Set rootを先にfinalizeし、それを参照するprofile／campaign instanceをContract Set preimageへ循環挿入しない。hashはMCD canonical encodingを使い、Product signed wrapperのRFC 8785 JCSで代用しない。

```text
threshold_set_hash =
  SHA-256(
    ASCII "MIRAKAN_RUNTIME_ECS_INITIAL_ACCEPTANCE_THRESHOLD_SET_V1"
    || uint32_be(len(canonical threshold set bytes
                     excluding threshold_set_hash))
    || canonical threshold set bytes excluding threshold_set_hash
  )

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

Initial qualification invents neither a zero baseline nor an improvement percentage. It passes only when the selected 16 KiB layout completes the full matrix、every hard predicate、exact six `scenario_cpu_p95_ns_max`、exact six `scenario_memory_peak_bytes_max`、`handle_resolve_p95_ns_max`、`lease_validation_p95_ns_max`を全て満たす。absolute threshold failureを8／32 KiBの良好値、相対改善、平均値、別TargetのReceiptで免除しない。

After an initial qualified layout exists, promotion compares baseline and candidate campaigns with byte-equal profile ref、Target、Contract Set、Toolchain、fixture、input trace、sample policy, and correctness oracle. Candidate refs are intentionally different.

promotionは、integrated P95が5%以上かつ0.20 ms以上改善し、peak memoryとallocation countの悪化が5%以内である場合、またはpeak memoryが15%以上改善し、latency、allocation、correctness、fault、load、presentationが既存Gate内である場合だけ許可する。それ以外はcurrent qualified layoutを維持し、Runtimeでauto-switchしない。

```text
diagnostic.performance.ecs-required-metric-missing
MIRAKAN-PERFORMANCE-ECS-REQUIRED-METRIC-MISSING
arguments = campaign_hash, scenario_id, payload_bytes, metric_id
result = qualification failure

diagnostic.performance.ecs-initial-threshold-invalid
MIRAKAN-PERFORMANCE-ECS-INITIAL-THRESHOLD-INVALID
arguments = threshold_set_ref, target_profile_ref, invalid_field_set_hash
result = Profile materialization and initial qualification prohibited
```

### 8.4 Algorithm optimization candidate qualification

algorithm／layout／Backend settingの変更は、Ownerが定義するsemantic oracleを一切緩和せず、同じCandidate、Contract Set、Toolchain lock、Target Profile、fixture、input trace、warm-up、sample count、aggregationでbaselineと比較する。predicted／estimated値、別Target、異なるfixture、sanitizer run、直前の`latest` runをpromotion Evidenceへ混在させない。一般高速経路の昇格は§8.1の「P95を5%以上かつ0.20 ms以上改善し、memory peak／allocation countを5%超悪化させず、correctness／fault／startup／hitch hard Gateを悪化させない」を適用し、Owner固有のより厳しいGateは各Owner文書が追加する。

`OptimizationDecisionProjectionV1`はsealed qualification recordから導出するPerformance-owned read-only Projectionであり、AI、Editor、Review toolが候補の状態と根拠を同じ意味で読むためのものとする。Projection自体はSource、selection authority、Receipt、Runtime dispatch tableではない。

ArchitectureのOwner／文書状態／current／targetは`ArchitectureExplainProjectionV1`、ECSのstorage／query／structural semanticsは`RuntimeEcsContractGraphV1`、pointer／allocation規則はMemory Contract fragment、Taskごとの可視範囲とauthorizationは`AiTaskContextCapsuleV1`が所有する。本Projectionはそれらを複写せず、同じProject lineage、source revision、Target、Contract Set、Toolchain、fixtureへexactに束縛する。ArchitectureまたはECS graphが可読でも、本Projectionの完全なmetric／semantic／selection Receipt closureがなければCandidateを`qualified_not_selected | selected`として説明しない。

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

<a id="9-owner-typed-workload-scale-model"></a>

## 9. Owner-typed workload scale modelと`ProjectScaleEnvelopeV2`

詳細は[performance-scale-catalog-proposal](../appendices/performance-scale-catalog-proposal.md#9-owner-typed-workload-scale-modelとprojectscaleenvelopev2)へ分離した。本節はnavigationだけを持ち、Catalog／Fixture定義を複写しない。

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
