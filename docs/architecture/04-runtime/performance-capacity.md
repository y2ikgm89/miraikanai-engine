# Miraikanai Engine Runtime Performance／Capacity Contract

- 文書ID: mirakan.arch.runtime-performance-capacity
- 状態: review
- 正本範囲: 共通CPU／GPU／memory／queue budget、capacity、reservation／loan、backpressure、worker capacity、測定法、regression、`ProjectScaleEnvelopeV1`、Target別Scale resolution、非破壊遷移、Qualification
- 非正本範囲: Runtime phase／tick／lifetime、World cell／coordinate field、LOD policy field、Authoring Document／ChangeSet field、Domain固有budget、外部Tool／SDK／driverの固定値、AI承認、Evidence envelope。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core architecture](../02-foundation/core-architecture.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Math／Core utilities](../02-foundation/math-core.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Project state](../03-authoring/project-state.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Gameplay programming model](../03-authoring/gameplay-programming-model.md)、[Scheduling／lifetime](scheduling-lifetime.md)、[Debugging／observability／replay](debugging-observability-replay.md)、[World](../06-rendering/world.md)、[LOD](../06-rendering/lod.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論とauthority

共通budget、capacity envelope、reservation、loan、backpressure、測定法、regression threshold、Scale qualificationは本書だけが決定する。Subsystem Ownerは本書が割り当てたparent budget内の固有配分とquality fallbackを所有し、共通上限を再定義しない。Runtimeのphase／tick／lifetimeは[Scheduling／lifetime](scheduling-lifetime.md)だけが所有する。

性能は平均fpsや推定costではなく、同一Source revision、Target Profile、Quality、Toolchain lock、fixture、input trace、process条件で計測する。correctness、Replay、visual／audio tolerance、fault、memory、hitchのいずれかを悪化させて性能合格を作らない。budget不足時はSource意味を黙って削らず、bounded planまたは`optimization_required`を返す。

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

CPU critical pathはInputを固定した時点から、そのstateを含む最初のrender submission呼出しがreturnするまでとする。該当snapshotがrun中にsubmitされなければhard failureである。GPU frameは当該snapshotの最初のGPU timestampから最終composite timestampまでとし、display sync待機を含めない。real frameとgenerated／displayed frameを分離する。

## 3. CPU memory envelope

`windows_desktop_v1`のactive GameHost共通soft budgetは2,048 MiBとする。

| Parent domain | Child envelope | MiB |
|---|---|---:|
| Core World／Save | ECS・World 128、snapshot／bridge 32、Save／Replay 64、stack／system 32 | 256 |
| Rendering CPU／upload | render extract 48、VFX CPU 32、shader／material metadata 32、descriptor metadata 16、upload staging 96、reserve 32 | 256 |
| Physics／Navigation／Animation | Physics 96、Navigation 64、Animation 64、reserve 32 | 256 |
| Unassigned headroom | 通常allocation、cache、loanへ使用禁止 | 128 |
| Audio | decoded／stream ring 96、voice／control 16、reserve 16 | 128 |
| Asset streaming | compressed cache 256、decode／transcode 256、hot cache 192、dependency metadata 32、reserve 32 | 768 |
| Frame／Job transient | Frame 32、Render frame slots最大48、Job scratch 40、reserve 8 | 128 |
| Emergency | diagnostic、journal、controlled shutdown専用 | 128 |
| **合計** |  | **2,048** |

`windows_editor_v1` process groupの共通soft budgetは4,096 MiBで、active Play Runtime 2,048、Authoring World／undo／revision 512、Asset import／cache client 512、UI／preview／thumbnail 384、AI bridge／schema／diagnostics 256、Editor reserve 384 MiBに分ける。Playしていない間もPlay partitionを長寿命Authoring cacheへ転用しない。Play準備前に必ずevictできる一時bufferだけ最大512 MiBのmode-exclusive loanを利用でき、返却不能ならPlay開始を拒否する。

Runtime Emergency 128 MiBを除く1,920 MiBを通常global scopeとする。

- 80%: eviction後に戻すtarget。
- 90%: warning、nonessential cacheとPresentation qualityの縮退を開始。
- 100%: Domain cap、nonessential allocation拒否。
- EmergencyとUnassigned headroomを通常処理、cache、quality維持へ貸さない。
- 未使用Parent budgetのloanは一load jobまたは最大120 render frameの先着期限までとする。
- 借用bytesを貸出元、借用先、global totalへ同時記録し、global scopeを超えない。
- deadline超過loanはconfiguration defectとしてqualificationを失敗させる。
- 必須allocationはeviction後に一度だけretryし、再失敗したauthoritative transactionをpublishしない。

Frame、render-frame、job scratchはscope完了後に一括resetする。GPU consumerを持つframe slotは全last-use submission完了までresetしない。hot pathのupstream fallbackを一般heapへ流さず、Development／Profileでは発生frameをfailureとして記録する。[Memory／Pointers](../02-foundation/memory-pointers.md)がallocator／pointer semanticsを所有する。

## 4. GPU memory envelope

`windows_desktop_v1`のGPU project budgetは`min(5,632 MiB, 0.80 × Platform reported budget)`とする。他TargetはPlatform Ownerが定めるaggregate working-set capを本書のmeasurement interfaceへ投影する。

| Domain | MiB |
|---|---:|
| Texture | 3,072 |
| Geometry | 1,024 |
| Render target／transient | 1,024 |
| Shader／descriptor | 256 |
| Emergency reserve | 256 |
| **合計** | **5,632** |

Platform budgetが小さい場合はcritical resourceとEmergencyを先に確保し、Presentation-only texture mip、streaming distance、shadow、transient resolutionの順にDomain Ownerの承認済みfallbackを適用する。Gameplay state、collision、goal、timingをGPU pressureから変更しない。

resource作成はcommitted／resident bytes、metadata、upload、main／render stallへchargeする。nonessential allocationはPlatform budget内でだけ許可する。全allocation走査は明示capture時だけ行い、毎frameはaggregate counterを使う。resource作成完了を即時live化せず、[Scheduling／lifetime](scheduling-lifetime.md)のactivation boundaryへ提出する。

aliasはRender Graphがlifetime非重複、heap compatibility、barrier、完全初期化を証明した場合だけ許可する。defragmentationはEditorまたはloading boundaryに限定し、一pass最大64 MiBまたは64 allocationの先着上限とする。copy、submission completion、handle swap、old allocation retireを一transactionで行い、失敗時はsource allocationを維持する。

device loss時のcapture／recovery順はScheduling、Renderer、Platform、Debugging Ownerへ委譲する。未完了submissionが進むと仮定した無期限waitをbudget回復手段にしない。

## 5. Queue capacityとbackpressure

次は共通C1 capacity profileのhard reservationであり、Targetを理由に暗黙縮小しない。Projectが変更する場合はmemory envelope、stress、Replay、Domain qualificationを再承認する。

| Queue／buffer | Entry capacity | Payload arena | max payload／entry | charge | critical reserve |
|---|---:|---:|---:|---|---:|
| Structural command | 16,384 / simulation step | 2 MiB | 16 KiB | ECS／World | 0 |
| Simulation command total | 65,536 / simulation step | 4 MiB | 16 KiB | ECS／World | 0 |
| Normalized Physics event | 65,536 / simulation step | 4 MiB | 256 B | Physics | 0 |
| Navigation request／result | each 4,096 / simulation step | each 8 MiB | 64 KiB | Navigation | 0 |
| Presentation event | 32,768 / simulation step | 4 MiB | 8 KiB | snapshot／bridge | 1,024 entries |
| Audio command | 8,192 entries | 1 MiB | 4 KiB | Audio | 512 entries |
| Audio completion | 4,096 entries | 256 KiB | 64 B | Audio | 256 entries |
| Asset activation | 1,024 / boundary | 1 MiB | 4 KiB | dependency metadata | 64 entries |

headerとarenaを含む起動時commitは合計58.9375 MiBで、所属Domainへchargeする。simulation／boundary bufferはcurrent／nextの二面、Navigationはrequest／resultを各一面、Audioは一つのbounded ringとする。entry数、個別payload、arena bytesのいずれかが先に上限へ達した時点でoverflowとする。

critical bitとpriorityはregistered schema／Capability manifestだけが設定し、Project payload、AI、GameplayDefinitionから昇格できない。criticalはcontrolled shutdown、resource release／retire、generation rollback等のEngine-owned operationに限定する。critical reserveをnoncritical producerへ貸さない。

| class | overflow／pressure behavior |
|---|---|
| Structural／Simulation／Physics authoritative | current transactionをpublishせずsession fault |
| Navigation request | new requestを`QueueFull`で拒否しaccepted resultを失わない |
| Presentation | lowest priorityからdropしgap／countを記録 |
| Audio | critical stop／releaseを維持しlow-priority playをdrop |
| Asset activation | next boundaryへ延期しclosureを部分activateしない |
| Debug telemetry | [Debugging／observability／replay](debugging-observability-replay.md)のgap／capture stop semanticsを使いGameplayへ影響させない |

Presentation／Audioはpriority昇順、同priorityはScheduling Ownerのcanonical message keyで後発からdropする。Asset generationは古いready generationを優先する。Developmentでは80%超をwarning、95%超をcapacity gate failureとする。Shippingでauthoritative recordを黙ってdropしない。

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

Subsystem固有pass／Domain budgetは各Ownerが上表の内数として割り当てる。未使用pass budgetを無関係な機能へ無制限に転用しない。Temporal reconstruction、frame generation、ray tracing等の追加経路はbase real frame、headroom、memory、visual、fault Gateを満たすTarget限定profileとし、generated frameをreal fpsへ加算して合格を作らない。

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
  -> promote | optimization_required | reject
```

最低metric familyはframe／latency、hitch、Runtime CPU、memory、loading、GPU／streaming、queue／backpressure、correctnessである。Hitchはdeadline、2倍deadline、50 ms超を数え、shader／pipeline、Asset I/O、allocation／page fault、job／queue wait、driver／device、unknownへ分類する。unknownを除外しない。

reference measurementはdeterministic warm-up後、同じ入力traceの120秒runを5回実行し、各run P95のmedianを採り、同じbuildで10分soakを追加する。Scale qualificationは10分runを3回、Production enduranceは2時間runを追加する。Mobile／Platform固有Targetは実機baselineを使い、Emulator／Simulatorで代用しない。

新しい高速経路は同一fixtureでP95を5%以上かつ0.20 ms以上改善し、memory peak／allocation countを5%超悪化させず、correctness、visual／audio、fault、startup／hitchのhard Gateを悪化させない場合だけ既定候補へ昇格する。baseline比のframe P95またはmemory peak／allocation countが5%超悪化した変更をregressionとする。

Baseline緩和は最適化と別Reviewとし、過去run分布、旧／新値、原因、quality差、下流Capability影響、[AI Security／Approval](../01-governance/ai-security-approval.md)の人間承認を必要とする。Evidence／Receipt構造、Provenance、保持、freshnessは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)だけが決定する。

## 9. Scale modelと`ProjectScaleEnvelopeV1`

Scaleは一つのboolやclassではなく、`world | population | content | authoring | authority`の五axisと数値Envelopeで表す。各axisは次のclosed orderを持つが、Source wire値は文字列enumとしordinal integerを保存しない。

| axis | closed values |
|---|---|
| world | `bounded_level | explicit_level_graph | continuous_partitioned | planetary_or_space` |
| population | `full_entity | pooled_or_batched | simulation_lod | distributed_simulation` |
| content | `single_working_set | incremental_partitioned | partial_cook_streamed | distributed_content_service` |
| authoring | `single_writer | optimistic_multi_writer | partition_owned_multi_writer | federated_repository` |
| authority | `single_process | client_server | sharded_server` |

Network authority variantsは専用仕様、Threat Model、Product activation前は`not_activated`である。local worker、render process、Asset workerはGameplay authorityを持たず`single_process`のままである。

`ProjectScaleEnvelopeV1`は次のroot contractを持つ。本型の構造定義は本節だけが正本である。

| Field | Contract |
|---|---|
| `scale_envelope_id` | UUIDv7 `StableId` |
| `project_id` | parent Project UUIDv7 |
| `schema_version` | current `uint32` |
| `source_revision` | exact `ProjectRevision` |
| `target_profile_refs` | exact StableId＋revision／hash、1～16 |
| `world_axis` | closed world axis |
| `population_axis` | closed population axis |
| `content_axis` | closed content axis |
| `authoring_axis` | closed authoring axis |
| `authority_axis` | closed authority axis |
| `world_intent_ref` | World Ownerのexact intent ref |
| `population_intent_refs` | `RuntimeScaleIntentV1` exact ref set、1件以上 |
| `content_intent_ref` | Asset lifecycle Ownerのexact intent ref |
| `authoring_intent_ref` | Project state Ownerのexact intent ref |
| `authority_intent_ref` | authority intent exact ref |
| `gameplay_fidelity_floor_refs` | Requirement／Game System exact ref、1件以上 |
| `integrated_fixture_refs` | Test Scenario exact ref、1件以上 |
| `decision_refs` | Scale判断を所有するDecision ref set |

表示用`scale_class`をSourceへ保存しない。Projectionは五axisとEnvelope hashから`compact_reference | medium_candidate | large_local_candidate | distributed_candidate`を決定的に導出し、`predicted | optimization_required | qualified | not_activated`と別表示する。

`RuntimeScaleIntentV1`はexperience role、total authored、peak live、peak active authoritative、peak spawn per simulation step、peak visible、interaction radius、simultaneous VFX、fidelity floor、Target setを必須とする。unknownを0、最大値、空optional、無制限で表さない。単位は[Math／Core utilities](../02-foundation/math-core.md)のsemantic typeを使い、non-finite、負数、range逆転、Targetなし、fidelity floorなしを拒否する。

World extent／coordinate／cell／streaming fieldは[World](../06-rendering/world.md)、LOD strategy／predicate／transition fieldは[LOD](../06-rendering/lod.md)、Authoring writer／Document／ChangeSet fieldは[Project state](../03-authoring/project-state.md)、content／build／cook fieldは[Asset lifecycle](../03-authoring/asset-lifecycle.md)と[Core architecture](../02-foundation/core-architecture.md)が所有する。本書はそれらをEnvelopeへexact refで束ねるだけで、field listを複写しない。

Envelope変更は通常の`ProjectChangeSet`であり、Before／After axisと数値、Target、Capability、Artifact、fixture、Decision closure、fidelity差分、再Cook／再Qualification、Save／Replay互換性、last-valid rollback refを必要とする。敵数、Damage、collision、goal、World範囲等を下げる変更は性能最適化ではなくGameplay changeとして人間承認を必要とする。

## 10. Canonical sourceとDomain resolver

Scaleの四層を分離する。

| layer | content | mutation |
|---|---|---|
| Canonical Source | Requirement、World、Entity、Asset metadata、GameplayDefinition、Scale Intent、Decision | ChangeSetだけ |
| Derived Plan | streaming、representation、residency、LOD、cook work | direct edit禁止 |
| Runtime State | generation、queue、active／resident set、chunk | Runtime ownerだけ |
| Evidence | Trace、Benchmark、Diff、Explanation、Qualification | append-only、Governance envelope参照 |

Runtime StateまたはEvidenceからSourceへ値を自動write-backしない。Stable IDはrename、repartition、recook、HLOD、instance化、Simulation LODで変えない。Runtime handle、vendor ID、cell-local／plan-local IDをSource／Saveへ保存しない。Save／Replayはexact Source、Contract、Envelope、Plan hashを使う。

万能な`ScaleManager`を作らない。各Domainは同じEnvelopeと自身のIntentを読み、自身のDerived Planを所有する。`ScalePlanSetV1`はplan本文を埋め込まず、exact Artifact ref、Source revision、Target Profile、Capability signature、dependency edge、qualification statusを束ねるmanifestである。required planのSource／Contract／Target／Capability hashが一件でもstale、missing、unqualifiedなら新setをpublishせずlast-valid playable setを維持する。

Population resolverはFull Entity、pool、archetype／SoA、instanced Presentation、reduced-frequency simulation、dormant record、aggregate simulation、HLOD、VFX Artifactからclosed strategyを選ぶ。各strategyはentry／exit predicate、owner、state／Save mapping、recovery、fallback、budgetを持つ。distance／visibilityだけでauthoritative actorを削らない。

## 11. AI scale operationとbounded explanation

Scale operationは`search | read_envelope | dependencies | resolve_preview | explain_plan | propose_envelope_change | validate_transition`のregistered MCD operationだけとする。Query／read／explainは[AI Security／Approval](../01-governance/ai-security-approval.md)が許可するread-only範囲、preview／changeは同OwnerのRisk／Approvalを消費する。本書はRisk値を再定義しない。

ProviderへProject Commit、Plan write、Capability activation、baseline緩和、Source直接write、server authority移動を公開しない。Queryはrevision、Envelope hash、Target、Capability signature、index revision、query hash、selected items、omitted ranges、cursor、Governance Evidence refを返す。全World／全Project dumpを行わない。

`ScaleExplanationReceiptV1`はSource revision、Envelope hash、Target、Plan set hash、selected／rejected closed strategy、fidelity proof ref、cost measurement ref、fallback chain、qualification status、invalidation conditionをDomain evidenceとして生成する。共通Receipt envelope、Provenance、署名、保持は[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を参照し、本書で再掲しない。

最低Diagnosticはmissing envelope、ambiguous requirement、unqualified capability、stale plan、fidelity violation、invalid reference closure、budget exceeded、partial activation rejected、distributed authority not activatedを区別する。unknownを近いenum、0、最大値、current Target defaultへ補正しない。

## 12. MediumからLargeへの非破壊遷移

許可する変更は、Partition Intent／Target追加、同じSourceからのinstance／batch／HLOD／Simulation LOD生成、Asset closure分割、Authoring re-shard、Build work item分割、Target別residency／fallback追加である。Source Stable ID、Save schema、Gameplay authorityを変えず、Derived Planだけを置換する。

禁止する変更は、Large専用Entity typeへのSource一括変換、Medium／Large別Save fork、cell／shard／build／server IDの混同、HLOD／GPU instanceのSave entity化、unqualified planのProduction表示、Medium fallback削除、性能のための無承認Gameplay変更である。

同じSource revisionとinput traceに対するMedium／Large planはauthoritative System state digest、Save field／Stable ID、Input→Command→Event順序、goal／damage／collision／navigation result、deterministic random stream、Level transition outcomeを一致させる。Presentation bitwise一致は不要でも、visual／audio tolerance、critical cue、event timing、fallback Gateを満たす。

Large World coordinate、continuous streaming、partition-owned multi-writer、distributed build、distributed simulation／authorityは専用Owner仕様がactivationされるまで`not_activated`である。現在のbounded Sourceへ空Manager、server field、RPC、global double座標を先回り追加しない。要求された場合は明示Diagnosticでfail closedする。

## 13. Integrated fixtureとqualification

各ProjectはEnvelopeから、実際に同時発生し得るresident／visible object、authoritative actor／projectile／interactive prop、spawn／destroy burst、Physics contact、Navigation request、Animation、Gameplay、VFX、Audio、camera、streaming、LOD、Asset activationを一つのdeterministic integrated fixtureへ生成する。Subsystem最大値を別runへ分離して同時性を隠さない。Compilerが個数を丸めて合格しやすくしない。

fixtureは次を全て満たす。

1. frame、Subsystem、memory、queue、GPU resource、streamingのhard Gate。
2. authoritative spawn、damage、collision、goal event drop 0。
3. Gameplay state、Replay hash、最終count、outcomeがreferenceと一致。
4. Presentation degradationがpriority、style、critical cue floorを満たす。
5. spawn、streaming boundary、VFX burstのP99.9がdeadlineを満たす。

`medium_candidate / qualified`にはProject固有Envelope、explicit Level graph、Save／Replay／Package、content totalとactive working setの分離、incremental Import／Cook、Target budget、2時間endurance、migration／load、bounded AI edit、last-valid recoveryを必要とする。

`large_local_candidate / qualified`にはMedium Gateに加え、利用するLarge Capabilityの専用仕様／Receipt、Project固有traversal／population trace、partition boundary／reference closure／load deadline／memory pressure／recovery、same-source Medium fallback、repartition後のStable ID／Save／Replay、bounded context、incremental／partial Cook同値、10分×3 run、2時間endurance、failure injectionを必要とする。

Distributed qualification Gateは本書でactivationしない。専用Authority仕様、Threat Model、server実機、loss／latency／abuse／recovery fixture、人間承認が揃うまでCatalogへactive Gateを掲載しない。

## 14. Failure、CI、completion

| failure | behavior |
|---|---|
| Envelope不足／曖昧 | Blocking Diagnostic。値を推測しない |
| Plan compile／migration失敗 | new plan非publish、Sourceとlast valid維持 |
| activation dependency不足 | authoritative closure全体をinactiveにする |
| stale Source／Target／Contract | result破棄、current revisionで再計画 |
| budget超過 | `optimization_required`、fidelityを自動緩和しない |
| Presentation Artifact不足 | approved visual fallback、Gameplay Source維持 |
| Simulation LOD restore失敗 | Full／last valid fallback、不可能ならactivation拒否 |
| partial Cook／Package失敗 | last valid package維持 |
| unactivated Authority | fail closed、意味同等single-process alternativeだけ提示 |

CIはbudget hard limit、loan deadline、queue pressure、missing metric、SourceへのRuntime／Derived ID、stale plan／Receipt、fidelity floor低下、partial activation、Presentation→Gameplay逆入力、Medium fallback欠落、unactivated Authority公開を拒否する。

本書のcompletionには、共通budget／capacity／backpressureが一意、Envelopeの五axisとroot contractが一意、Domain fieldのowner委譲が明示、same-source transition fixture、bounded explanation、Target qualification、last-valid recoveryが実行可能であることを必要とする。Product Phase順序、Capability maturity、Governance authorization、Evidence envelopeを本書で再定義しない。

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
