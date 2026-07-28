# Miraikanai Engine VFX Runtime Contract

- 文書ID: mirakan.arch.rendering-vfx-runtime
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: compiled VFX artifact boundary、instance lifecycle、CPU／GPU simulation、VFX render execution、visual collision／event、VFX固有resource ceiling／fallback／diagnostic／qualification
- 非正本範囲: VFX Source／Graph／catalog／compiler authoring、Render Graph共通pass／resource／queue、LOD共通selection、Runtime phase／shared capacity、Physics authority、Asset transaction、Tool／SDK version、AI authorization、Evidence envelope。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Render Graph](render-graph.md)、[VFX Authoring](vfx-authoring.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)
- 関連文書: [Product Plan](../00-product/product-plan.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[Collision](../05-simulation/collision.md)、[Physics](../05-simulation/physics.md)、[Render Graph](render-graph.md)、[Materials](materials.md)、[LOD](lod.md)、[VFX authoring](vfx-authoring.md)、[Environment／surfaces](environment-surfaces.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-21

## 1. 結論と実行境界

VFX Runtimeはoffline compiled artifactだけをleaseし、Source Graph、Node source、shader source、compiler、JITを読まない。CPU／GPU Emitterは同じSystemのparameter、seed、transform、lifecycleを共有できるが、Particle storageを共有せず、実行中stateをbackend間で移送しない。

RuntimeはVFXをPresentationとして実行し、World／Physicsへwrite backしない。Runtime phase、job dependency、publish lifetime、command envelopeは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、共通capacity／measurement／backpressureは[Runtime performance／capacity](../04-runtime/performance-capacity.md)、pass／resource／queue／barrier／surface compositionは[Render Graph](render-graph.md)が所有する。本書はその境界内のVFX Cadence binding、storage、domain resource ceiling、degradationだけを決定する。V1 execution artifactはfixed rational Cadenceだけを表現し、variable／turn-based／explicit-stepを60 Hzへ代入しない。

## 2. Compiled artifact boundary

この節だけがVFX compiler出力とRuntime入力の構造を定義する。

```text
VfxSystemArtifactManifestV1
  manifest_schema_version: 1
  manifest_id: StableId
  manifest_version: positive u32
  system_id
  source_content_hash
  target_profile_ref: TargetProfileRefV1
  quality_profile_ref: VfxQualityProfileRefV1
  budget_profile_ref: VfxBudgetProfileRefV1
  lod_profile_ref: VfxLodProfileRefV1
  simulation_cadence_profile_ref: SimulationCadenceProfileRefV1
  timing_contract: fixed_rational_advance_v1
  emitter_artifact_refs[1..32]:
    exact {artifact_id, artifact_version, artifact_content_hash}
  parameter_layout_hash
  lifecycle_descriptor
  manifest_content_hash: SHA-256

VfxExecutionArtifactV1
  artifact_schema_version: 1
  artifact_id: StableId
  artifact_version: positive u32
  artifact_content_hash: SHA-256
  system_id
  emitter_id
  source_content_hash
  compiler_build_hash
  node_catalog_version
  target_profile_ref: TargetProfileRefV1
  quality_profile_ref: VfxQualityProfileRefV1
  budget_profile_ref: VfxBudgetProfileRefV1
  lod_profile_ref: VfxLodProfileRefV1
  simulation_cadence_profile_ref: SimulationCadenceProfileRefV1
  timing_contract: fixed_rational_advance_v1
  spatial_domain: d2 | d3
  execution_target: cpu | gpu
  parameter_layout_hash
  attribute_layout_hash
  renderer_interface_hash
  cpu_program_hash: optional
  shader_package_refs[]
  render_pass_template_ids[]
  fixed_bounds
  resource_estimate
  capability_requirements[]
  fallback_artifact_ref: optional
```

Manifestは全enabled Emitterの選択Dimension artifactがReadyの場合だけReadyになる。keyはSystem、Emitter、Dimension、execution target、Target、Quality、Budget、LOD、Toolchain、Source closure、完成`SimulationCadenceProfileRefV1`を含み、Source／compiler／Profile／Cadence hash違いを再利用しない。Artifact hashはASCII `MIRAKAN_VFX_EXECUTION_ARTIFACT_V1`、Manifest hashはASCII `MIRAKAN_VFX_SYSTEM_ARTIFACT_MANIFEST_V1`と各自己hashを除くcanonical bytesから計算する。両者はReceipt-free baseで、Verification Receipt／Activation Bindingをpreimageまたはfieldへ埋め戻さない。System Source、Quality binding、Manifest、全Emitter artifact、Runtime Package、Save／Replay closureのTarget／Quality／Budget／LOD／Cadence Profile refはbyte equalityにする。V1は解決先が`cadence.kind=fixed`でTarget別VFX Qualificationがfreshな場合だけloadし、他kindまたは未資格Profileを`cadence_profile_not_qualified`でinstance開始前に拒否する。RuntimeはManifestと[LOD](lod.md)が選んだexact VFX tierをinstance開始／qualified transition boundaryでだけ適用し、frame負荷から別Profile、CPU／GPU、未登録tierを合成しない。

`VfxCpuProgramV1`は事前compile済みclosed C++ kernel IDとimmutable parameter blockのtyped順序であり、bytecode／scriptではない。loop、recursion、file／network／OS API、reflection、allocation instructionを持たず、kernelごとのattribute access、scratch byte、dimension、stageをmanifestへ固定する。unknown kernel、manifest hash、parameter block size不一致をload前に拒否する。

`VfxGraphIrV1`はauthoring compiler内部だけ、`VfxCpuInstanceStateV1`はPlay sessionだけ、`VfxSystemInstance`は複数Emitterのlifecycle／parameter／seedを束ねるRuntime object、`VfxSystemHandle`／`VfxEmitterHandle`は`{index32,generation32}`、GPU buffer／counter／indirect argumentはRenderer submission lifetimeだけに存在する。

## 3. Instance、command、snapshot

Lifecycleは`Unloaded -> Ready -> Prewarming -> Running -> Draining -> Complete`、live stateから`Stopped | Faulted`、`Running <-> Paused`である。Readyはartifact lease、slot、parameter block、buffer容量を確保済み、Prewarmingはpreload対象のhidden instanceだけ、Drainingはspawn停止後alive 0まで更新、Stoppedは次publishでretire、Pausedはage／rate accumulator／stepを進めず、FaultedはWorld不変のまま処理を止める。prewarm 0もzero-work Prewarmingを通る。

CPU Prewarmは通常advance kernel、GPU Prewarmはpublish当たり最大8 `VfxGpuAdvanceRecordV1`のhidden compute batchとし、fence完了後だけ次batchへ進む。surface unavailableではGPU Prewarmを開始せずcompiled CPU fallbackを選ぶかloadを失敗させる。全advanceを一dispatchへ圧縮しない。

Runtime commandは`SpawnVfxSystemV1`（asset、transform source、event payload、parameter block、importance）、`StopVfxSystemV1`（handle、`immediate | drain`）、`PauseVfxSystemV1`、`ResumeVfxSystemV1`、`SetVfxParameterV1`（handle、parameter ID、typed value）、`SetVfxTransformV1`（handle、finite transform）である。seed、drop priority、merge順はEngineが解決し、critical権限をVFX commandへ与えない。

Rate emissionはEmitter stateに`accumulator_q32:u64`と`division_remainder:u64`を持ち、fixed Cadence `rate_hz=n/d`の各advanceに次をchecked `uint128`で順に行う。`0 <= division_remainder < n`を不変条件とし、rateを0へ変更した時だけ両stateを0へresetする。BurstとEvent countをこのspawnへ加えた後に一つのadmissionへ通し、dropを繰り越さない。

```text
numerator_q32 = rate_q32 * d + division_remainder
accumulator_q32 += floor(numerator_q32 / n)
division_remainder = numerator_q32 mod n
spawn = accumulator_q32 >> 32
accumulator_q32 &= 0xffffffff
```

```text
VfxBatchSnapshotV1
  cadence_profile_ref: SimulationCadenceProfileRefV1
  snapshot_simulation_advance_interval_hash: SHA-256
  snapshot_advance_sequence: positive u64
  weather_presentation_snapshot_ref: nullable<WeatherPresentationSnapshotRefV1>
  cpu_draw_batches[]
  gpu_emitter_records[]

VfxGpuEmitterRecordV1
  system_instance_handle
  emitter_id
  artifact_version
  current_transform
  event_payload
  parameter_blocks[1..8]
  last_advance_sequence: u64
  advance_records[0..8]

VfxGpuAdvanceRecordV1
  cadence_profile_ref: SimulationCadenceProfileRefV1
  simulation_advance_interval_hash: SHA-256
  advance_sequence: positive u64
  first_reserved_spawn_id: u64
  external_spawn_count: u32
  sub_emitter_spawn_quota: u32
  emitter_transform
  parameter_block_index: u8
  simulation_step_count: u8 = 1
```

Batch SnapshotのProfile ref／snapshot interval hash／snapshot sequenceは、Snapshotをsealした最新`SimulationAdvanceIntervalV1`の同三値とbyte equalityにする。各`advance_records[]`は自身が表す個別IntervalのProfile ref／interval hash／sequenceとbyte equalityにし、Batchの最新hashを過去Recordへ複写しない。Parameter blocksは直近8 advanceの異なるrevisionをdeduplicateする。GPU Emitterは各advanceに1 recordを生成し、外部spawnと内部Event用quotaを同じProject admissionから予約する。IDは外部spawnを先、Sub-emitterを親Emitter ID／親Spawn ID／Event Node ID順に割り当て、unused IDを再利用しない。

`weather_presentation_snapshot_ref`は[Environment／surfaces](environment-surfaces.md)が所有する同じpublished Environment generationのexact refである。Weather bindingが存在しない時はcanonical nullとし、直前frame、World名、Emitter名または既定天候から補完しない。VFX artifactでtyped Weather presentation inputを宣言したEmitterだけが解決先`WeatherPresentationSnapshotV1`を読み、降水量、温度、風向等をEmitter parameterへ再定義・複製しない。この入力は粒子表現専用であり、Gameplay、Snow accumulation、Water authorityへ逆流させない。

Rendererは`last_consumed_advance_sequence`より新しいrecordだけをadvance sequence昇順に1 record＝1 Simulation Advanceとして処理する。`simulation_step_count`はV1で常に1であり、1以外の値をload時に拒否する。複数advanceの圧縮は将来versionで別途定義する。同じsnapshot再描画でsimulationを重ねず、複数recordを一frameで順次処理する。Paused中はrecordを生成しない。保持範囲を超えるgapはcatch-up dispatchせず、ambient／loopは最新advanceからvisual restart、one-shotは再発火せずDiagnosticを残す。

## 4. CPU simulation

CPU storageは256 Particle、64-byte aligned chunkのSoAである。`spawn_id:u64, age_steps:u32, lifetime_steps:u32, alive:u32`を必須とし、livenessで必要なPosition／previous position／velocity／color／size／rotationだけを配置する。2Dは`vec2f`、3Dは`vec3f`の別layout、Asset参照はEmitter bindingへ集約し、spawn／update／death時のheap allocationとraw pointer captureを禁止する。

Stable update algorithmは次の順である。

1. alive ParticleをSpawn ID昇順のlogical orderで保持する。
2. 256粒子chunkを固定rangeとしてworkerへ提出する。
3. 各jobはprivate dead mask／event bufferへ書く。
4. join後、chunk index順にalive prefix sumを作る。
5. next bufferへstable compactする。
6. admitted particleをSpawn ID順に空slotへage 0で初期化する。
7. 空chunkだけをinstance poolへ返す。

swap-and-pop、completion順append、unordered reductionを使わない。各advanceの`dt`はProfileのfixed `rate_hz=n/d`から得るexact rational `d/n s`で、deterministic numeric policyが定める一回のconversionだけを使う。`v_next=v+a*dt`、`p_next=p+v_next*dt`、linear dragは`v_next*=max(0,1-drag_per_second*dt)`とし、`0 <= drag_per_second <= n/d`をCookで検証する。lifetimeは`clamp(ceil(lifetime_seconds*n/d),1,ceil(3600*n/d))` advanceへspawn時にchecked `uint64`で量子化し、上限が`uint32`へ収まらないProfileを拒否する。Age／Lifetime秒は`steps*d/n`のexact rational projectionであり、60で除算しない。

各stepは「既存aliveのage増加、update、death候補、visual event確定、stable compact、新規spawn」の順であり、新規Particleは生成stepにupdateしない。CPU Target profileはfloat reassociation／implicit FMA contractionを禁止し、scalar referenceとoptimized kernelを同じfixtureで比較する。

Boundsはchunk index順のstable reductionでPosition／Size／Trailを集約し、NaN／InfをFaultとする。GPUはfinite fixed boundsを必須とし、compiler推定boundsをSourceへ自動commitしない。

## 5. GPU simulation、render、collision

GPU variantはcompute shader、read／write storage buffer、32-bit atomic、indirect argument、explicit dependency、validated shader packageを必要とし、wave size、subgroup幅、mesh shader、ray tracing、unbounded bindlessを前提にしない。Portable GPU artifactは`renderer-profile.portable-mobile` shader capability intersection内へ生成する。

Persistent storageは`ParticleAttributesA/B, AliveIndicesA/B, DeadIndices, EmitterParameters, SpawnRequests, EventCandidates, SubEmitterReady, SubEmitterNext, SortKeys, SortIndices, IndirectArguments, Counters`である。Counterはexplicit buffer storageを使い、CPU readbackで通常draw count／event／Gameplay判断を決めない。

GPU advance順は`VfxGpuResetCounters -> VfxGpuUpdateAndCompact -> VfxGpuEventResolve -> VfxGpuAdmitAndSpawn -> SubEmitterReady/SubEmitterNext swap`、全advance後に必要時`VfxGpuSort -> VfxGpuBuildIndirect -> VfxDraw`である。Resetはadvance一時counterだけを初期化し、alive／dead／Readyを保持する。現advance eventはNextへ書き、spawnは前advance Readyだけを`VfxGpuAdvanceRecordV1.sub_emitter_spawn_quota`内で消費する。quota超過をcanonical末尾からdropし、繰り越さない。recordがないframeはsimulationせずIndirect／Drawだけを行う。Pass mergeは意味順を変えない。

GPU sortは`none | spawn_order | view_depth | emitter_only`であり、選択は[VFX authoringの`sort_mode`](vfx-authoring.md#21-systememitterparameter)をSource正本とする。view depthはEmitter 65,536、Project 262,144 Particleまで、depth降順かつ同bucket Spawn ID順とする。超過Sourceはemitter-onlyまたはadditive Materialへ明示変更しない限りCook拒否する。

Render outputは`Sprite2D, Billboard3D, PortableFacingSprite, Flipbook, BasicTrail, Mesh, Ribbon, ParticleLight`である。Portable spriteはDimension別へspecializeする。Blendは`premultiplied_alpha | additive | multiply`。Particle LightはRenderer listだけへ出力し、shadow、GI、Physics、Navigation、AI perceptionを変更せず、Project 32／Emitter 4を上限とする。Pixel-locked outputは明示flag、point sampling、integer scaleを必要とする。

Visual collision sourceはscene depth、versioned global SDF、`VfxCollisionProxy2D | VfxCollisionProxy3D`だけである。

```text
VfxCollisionProxySnapshotV1
  snapshot_advance_sequence: u64
  generation: u64
  proxy2d[0..4096]
  proxy3d[0..2048]
  acceleration_artifact_handle
```

2D shapeはCircle、CapsuleSegment、AABB、8-vertex以下Convex、3DはSphere、Capsule、Box、32-plane以下Convexに閉じる。Collision ownerがStable ID、finite transform、priority、friction／restitution `[0,1]`とimmutable acceleration artifactをpublishする。Mobile上限は2D 1,024／3D 512。dynamic overflowは`priority desc, Stable ID asc`末尾を除外する。

CPU／proxy collisionはprevious→current swept point／sphereで1 step最大1 hit、最小TOI、同値Stable ID昇順を選ぶ。bounce／friction／kill／Eventを明示し、未指定はhit point clamp＋normal velocity zeroとする。Visual resultはGameplayへ返さない。

Event候補は親Emitter Stable ID、親Spawn ID、Route Stable ID順、max超過末尾dropである。CPU／GPU Eventはいずれも同じCadenceの次Simulation Advanceだけへ配送し、そのadvanceのexact Profile ref／Interval hash／sequenceへ束縛する。render frame、presentation step、catch-up dispatch数から配送時点を推測せず、GPU→CPU／CPU→GPU edgeを拒否する。`WaterInteractionEventV1`や`SnowSurfaceInteractionEventV1`等のauthoritative eventを入力にできるが、VFX eventからDamage、Quest、Save、Audio正規eventを生成しない。

## 6. Persistence、resource ceiling、degradation

通常one-shotは保存しない。永続ambientだけ次を保存する。

```text
PersistentVfxDescriptorV1
  system_asset_id
  instance_stable_id
  spatial_domain
  parameter_block
  system_seed
  cadence_profile_ref: SimulationCadenceProfileRefV1
  start_advance_sequence
  paused
```

Particle、GPU buffer、sort、Trail pointは保存しない。Load時はDescriptor、Runtime Package、Artifact、current Profile refのbyte equalityを検証する。Load後prewarmは最大120 advance、超過履歴はloop phaseからvisual restartする。[VFX Authoringが所有する`VfxBakeCacheV1`](vfx-authoring.md#4-compilerplanned-authoring-actionpreview)をoffline代替としてexactに消費し、Fieldを再定義しない。GPU stateはReplay digest／authoritative Save／network stateに含めない。

Domain resource ceilingは次である。共通frame／memory envelope、集約測定、backpressureはRuntime capacity ownerへ委譲する。

| Profile／backend | active Emitter | alive Project／Emitter | spawn/s Project／Emitter | burst/step Project／Emitter | trail Project／Trail | event/step Project／Emitter |
|---|---:|---:|---:|---:|---:|---:|
| desktop CPU | 256 | 65,536／8,192 | 20,000／4,096 | 8,192／2,048 | 16,384／64 | portable 0、advanced 16,384／2,048 |
| desktop GPU | 1,024 | 1,048,576／262,144 | 524,288／131,072 | 131,072／32,768 | 262,144／256 | 65,536／4,096 |
| `mobile_baseline` CPU | 128 | 16,384／2,048 | 5,000／1,024 | 2,048／512 | 4,096／32 | 0 |
| `mobile_baseline` limited GPU | 128 | 65,536／16,384 | 32,768／8,192 | 8,192／2,048 | 32,768／64 | 8,192／1,024 |
| `mobile_standard` CPU | 192 | 32,768／4,096 | 10,000／2,048 | 4,096／1,024 | 8,192／48 | 16,384／2,048 |
| `mobile_standard` GPU | 512 | 262,144／65,536 | 131,072／32,768 | 32,768／8,192 | 65,536／128 | 32,768／2,048 |
| `mobile_high` CPU | 256 | 65,536／8,192 | 20,000／4,096 | 8,192／2,048 | 16,384／64 | 32,768／4,096 |
| `mobile_high` GPU | 768 | 524,288／131,072 | 262,144／65,536 | 65,536／16,384 | 131,072／192 | 65,536／4,096 |

| Profile | CPU memory | GPU persistent／transient | Particle Light Project／Emitter | VFX domain P95 CPU／GPU |
|---|---:|---:|---:|---:|
| desktop | 32 MiB | 128／64 MiB | 32／4 | 0.75／0.75 ms |
| `mobile_baseline` | 12 MiB | limited profile 16／8 MiB | 0 | 0.40／0.40 ms |
| `mobile_standard` | 20 MiB | 48／24 MiB | 8／2 | 0.60／0.60 ms |
| `mobile_high` | 32 MiB | 96／48 MiB | 16／4 | 0.75／0.75 ms |

System slotは`min(4096, CPU active emitter ceiling + GPU active emitter ceiling)`である。

Capacityは`ceil(max_particles/256)*256`へ丸め、各SoA arrayを`align_up(sizeof(attribute)*capacity,64)`で計上する。CPUはcurrent／next、alive index、dead mask、rate、event、Trail、metadata、GPUはpersistent buffers、Trail、transient event／sort／prefix／draw scratchのsimultaneous peakを含む。Eventは`max_events_per_advance * align16(payload size)`、payload最大256 byte。Texture／Mesh／Material／Shader packageを重複計上しない。estimateよりinstrumented peakが大きいArtifactはqualification失敗である。

Overdraw ratioは`particle pixel shader invocations / internal output pixels`、`<=4` pass、`>4` warning、`>8` proposal／Cook rejectとする。Fallbackは[LOD](lod.md)の`VfxLodProfileV1`／`VfxLodTierV1`だけを使い、branch、spawn scale、alive、update interval、render output、execution target、minimum Cueを明示する。Runtimeは未登録variantを合成しない。

突発spawnは`semantic_priority desc, projected_coverage_px_q16 desc, emitter Stable ID asc, Spawn ID asc`末尾をdropし、次advanceへ繰り越さない。`ParticleSpawnDropped{emitter,advance_sequence,reason,count}`へ集計する。critical Cueのshape／timing／minimum visibilityをambientより先に削らず、敵味方を一律に差別化しない。Thermal signalは一段ずつqualified tierを下げられるが、Project dataから無効化しない。

Artifact promotion後の新instanceだけ新generationを使い、live instanceは旧leaseをCompleteまで保持する。Graph hot swapでalive layoutを置換しない。全CPU job、snapshot、GPU serial、lease終了後にretireする。Device faultではGPU submitを止め、ambientだけdescriptorからrestart、one-shotは非再発火、CPU stateはWorld lifetimeが維持される場合だけ保持する。

## 7. Failure、diagnostic、qualification

Runtime closed Diagnosticは`VfxBoundsEscape, VfxCollisionProxyOverflow, VfxInstancePoolFull, VfxSpawnDropped, VfxEventOverflow, VfxGpuCounterOverflow, VfxGpuTimelineGap, VfxGpuFault, VfxDeviceRecoveryRestarted`である。RuntimeはCapability／artifact mismatchでinstanceを開始せず、NaN／Infまたはcounter overflowでinstanceをFaultedへ移し、pool fullではlowest-priority new spawnを拒否する。GPU faultはRenderer recoveryへ渡し、authoritative Worldを変更しない。

CPU qualificationは1／255／256／257／8,192 particle、fixed rational 60/1および別rateでのrate／remainder／burst／loop／pause／drain／parameter／prewarm、lifetime境界、fixed seed 100 run、worker順変更、scalar／optimized一致、allocation 0、bounds、soakを含む。GPU qualificationはresource hazard、zero／capacity／capacity+1、counter wrap拒否、indirect draw、output golden、collision tolerance、Ready／Next非再入、surface／device recovery、LOD hysteresis、snapshot再描画、multi-record、1 render frame内の複数advance catch-upでもCPU／GPU Eventの次advance配送が一致するfixture、9-advance gapを含む。current pass対象は60/1だけで、別rate fixtureはFuture capability promotion時のdestination evidenceである。

Water／Weather／Snow、Audio、Cameraとのintegrationは一つのauthoritative eventを各Presentation ownerへ独立配送する。Weather駆動表現はexact `WeatherPresentationSnapshotV1`を非authoritative入力として消費し、Snow／Water／Gameplayを進めるauthoritative eventまたは状態として扱わない。VFX結果を他Ownerへ逆入力しない。Resource、performance、fault結果は同一Target／Quality／Artifact hashへ結び、Evidence envelopeはAI Verification ownerへ委譲する。
