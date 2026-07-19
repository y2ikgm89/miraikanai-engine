# Miraikanai Engine 独自Physics Platform／Dynamicsアーキテクチャ規約

- 文書版: 1.0
- 作成日: 2026-07-20
- 最終更新日: 2026-07-20
- 対象: 2D／3D Physics World、Dynamics、Joint／Constraint、Character Motor、Backend、Editor、AI Authoring、Save／Replay
- 状態: プロジェクト公式の規範設計レビュー版。Kernelは採用決定済み、Production昇格は実装後のQualification待ち
- 上位文書: [AIネイティブ独自ゲームエンジン 設計計画書](./2026-07-18-ai-native-game-engine-authoring-design.md)
- Collision正本: [Miraikanai Engine Collision／Colliderアーキテクチャ規約](./2026-07-19-collision-collider-architecture-design.md)
- Runtime正本: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- Simulation連携: [Miraikanai Engine Physics／Navigation／Animation連携規約](./2026-07-19-physics-navigation-animation-architecture-design.md)
- 機能範囲: [Miraikanai Engine 2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)
- 基盤規約: [Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)
- Game実装規約: [Miraikanai Engine C++実行コード・構造化ゲームデータ規約](./2026-07-19-cpp-structured-game-data-design.md)
- 実行可能契約: [Miraikanai Engine 実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)

## 1. 結論

Miraikanai EngineのPhysicsは、**独自のPhysics PlatformをC++23で実装し、数値計算kernelだけを非公開Adapterへ隔離する**方式に固定する。

- 2D rigid body／collision solverはBox2D 3.1.1を使用する。
- 3D rigid body／collision solverはJolt Physics 5.6.0を使用する。
- World、Body、Collider、Joint、Character、Command、Event、Profile、Save、Replay、Editor、AI CapabilityはMiraikanai Engineが所有する。
- Project C++、GameplayDefinition、AI、EditorへBox2D／Joltの型、ID、pointer、callback、設定名、serializationを公開しない。
- Gameの実行Codeは従来どおりC++23であり、Luau、Lua、C# Runtime、汎用Script VMを追加しない。
- Solverを最初から自作しない。独自solverはC3 Researchで、既存kernelを同じfixtureで上回る証拠が得られた場合だけ別ADRで検討する。

したがって「C++だけで作る」という製品方針と矛盾しない。Box2Dはportable C17、JoltはC++17のstatic libraryとして同じnative binaryへlinkされるが、Game programming modelとFirst-party Engine／EditorはC++23のままである。

本書でいう「公式推奨」は、外部Libraryの作者がMiraikanai Engine向けに推奨したという意味ではない。一次資料、対象Platform、機能、memory、保守性、AI安全性を比較し、本プロジェクトが公式規約として採用した選択を意味する。

## 2. 決定権と境界

| 主題 | 正本 |
|---|---|
| Physics World、solver profile、Dynamics command、Joint、Character Motor、Backend build、Physics Save／Replay、Physics AI／Editor | 本書 |
| Body／Collider field、shape、material、filter、query、contact／trigger、collision cook | Collision規約 |
| Tick phase、queue、writer、handle lease、global memory、failure publish | Runtime規約 |
| PhysicsとNavigation／Animationのsnapshot／command連携 | Physics／Navigation／Animation連携規約 |
| C1／C2製品機能とReference Scene | 2D／3D機能計画 |
| C++ ownership、pointer、dependency lock、Build Driver、directory上位規則 | 基盤規約 |
| GameplayDefinition／Project C++の選択と権限 | Game実装規約 |
| MCD、Schema、Provider／MCP projection、Codegen | 実行可能契約規約 |

本書はRuntime規約のexactly 60 Hz、Physics 96 MiB、Physics全体P95 2.50 ms、T40～T70のwrite authorityを緩和しない。Collision規約のshape、filter、query、event semanticsを再定義しない。

## 3. 採用判断

### 3.1 比較した三方式

| 方式 | 長所 | 重大な問題 | 判断 |
|---|---|---|---|
| Solverまで全自作 | 数式、data layout、挙動を完全所有 | 2D／3D、CCD、joint、sleep、stack、mobile、toolまでの検証量が過大。最初のPlayableが大幅に遅れる | C1／C2では不採用 |
| Box2D／Jolt APIをGameへ直接公開 | 初期実装が短い | Save、AI、Project C++、Editor、version更新がVendor APIへ固定され、安全な検証境界を失う | 不採用 |
| 独自契約＋private kernel Adapter | 製品の意味とUXを所有しつつ成熟solverを利用できる | Adapter conformanceと二つのBackend保守が必要 | **採用** |

AIが理解しやすいかどうかはsolverの内部実装ではなく、公開する語彙、closed schema、単位、範囲、状態遷移、failure、previewが一意かで決まる。独自solverを作っても無型APIならAIには扱いにくく、Box2D／Joltを使ってもMiraikanaiの型付き契約へ閉じれば安全に扱える。

### 3.2 有名Engineから確認できること

| Engine | 公式資料で確認した構成 | 本計画への意味 |
|---|---|---|
| Unity 6 | Built-in 3DはNVIDIA PhysX integration、Built-in 2DはBox2D integration。DOTSはUnity PhysicsまたはHavok Physics for Unity | 製品API／Editorを所有し、用途別kernelを統合する構成は成立している |
| Unreal Engine 5.8 | EpicのChaos PhysicsをEngineへ統合し、Rigid Body、Constraint、Vehicle、Ragdoll等を提供 | 大規模組織ではfirst-party solverも成立するが、Miraikanai C1の費用対効果とは一致しない |
| Godot 4.6 | 新規3D Projectの既定PhysicsをJoltへ変更し、Engine integrationを所有 | Open source Engineでも外部kernelを深く統合する方式がProductionで成立している |

これらはcoverageとriskの比較材料であり、MiraikanaiのComponent名、Scene形式、Editor、Profile、既定値を模倣する根拠にはしない。

## 4. 成熟度と対象範囲

| Level | Physics到達点 |
|---|---|
| C0 Foundation | MCD、Profile、Validator、Handle、Adapter Port、Diagnostic、Qualification harness |
| C1 First Playable | 2D／3D rigid body、C1 Joint、Kinematic Character Motor、Save／Replay、Editor／AI、Windows Reference合格 |
| C2 Production | Ragdoll、Vehicle、Destructible constraint、SixDOF、large static world、Target別tuning |
| C3 Research | Soft body、cloth／fluid、GPU Physics／hair、double precision large world、network lockstep、独自solver |

C1／C2のSchemaはC3 fieldを受理しない。Jolt 5.6.0にsoft body、vehicle、GPU compute、hair機能が存在しても、Capability昇格前にVendor機能を直接有効化しない。

## 5. 独自Physics Platformの構成

### 5.1 正規object

| Object | 所有者 | 永続化 |
|---|---|---|
| `PhysicsWorldDocumentV1` | Authoring Model | する |
| `Physics2DWorldProfileV1`／`Physics3DWorldProfileV1` | Project Profile | する |
| `PhysicsBody2DComponent`／`PhysicsBody3DComponent` | World Model | する |
| Collider、Material、Filter | Collision | する |
| `Joint2DComponentV1`／`Constraint3DComponentV1` | World Model | する |
| `CharacterMotorProfileV1`／`CharacterMotorComponentV1` | World Model | する |
| `PhysicsWorldHandle`／`PhysicsBodyHandle`／`PhysicsJointHandle` | Runtime Physics | しない |
| Native World／Body／Shape／Joint | Backend Adapter | しない |
| `PhysicsStateSnapshotV1` | Runtime bridge | tick内だけ |
| `PhysicsSaveStateV1`／`PhysicsReplayHeaderV1` | Save／Replay | する |

Runtime handleはすべて`{index32, generation32}`である。Native `b2WorldId`、`b2BodyId`、`b2JointId`、Jolt `PhysicsSystem*`、`BodyID`、`Constraint*`をこれらのbitへ詰め替えず、Adapter private tableで対応付ける。

### 5.2 ModuleとDirectory

```text
engine/physics/
  contracts/
  core/
    world/
    dynamics/
    joints/
    character/
    save_replay/
  collision/
  diagnostics/
  backends/
    box2d/
    jolt/
authoring/physics/
editor/panels/physics/
tools/physics_qualification/
```

- `contracts`はMCD生成value type、Port、Command、Event、Snapshotだけを公開する。
- `core`はBackend非依存のWorld lifecycle、command merge、Joint registry、Character Motor、Save／Replayを所有する。
- `collision`はCollision規約に従うQuery／Event正規化を所有する。
- `backends`だけがVendor headerをincludeし、Vendor libraryへlinkする。
- `authoring/physics`はSource document、Validator、Cost preview、ChangeSet projectionを所有する。
- `editor/panels/physics`は正規documentのProjectionであり、Runtime Worldの正本ではない。
- `tools/physics_qualification`は通常Game Runtimeへlinkせず、Kernel昇格試験と比較Benchmarkを実行する。

Project C++は`mira.runtime`から生成される`PhysicsCommandWriterV1`、`PhysicsQueryPortV1`、`PhysicsEventViewV1`だけを利用する。`mira.physics.backend.*`という公開Moduleは作らない。

### 5.3 所有権とthread

- `PhysicsWorldOwner`だけがNative Worldとmapping tableを所有する。
- Native pointer、Body lock、callback view、query collectorをfunction、tick、job、thread境界へ保持しない。
- `PhysicsBodyHandle`と`PhysicsJointHandle`はT40～T60のPhysics lease内だけresolveできる。
- Joltの複数Body accessはmulti-lock APIへ一括提出し、任意順の個別lockを重ねない。
- Box2D IDは使用直前にvalidityを検査し、destroy後のEvent IDをdereferenceしない。
- Callback中はpreallocated worker-local bufferへのcopyだけを許可し、allocation、logging、World変更、Gameplay、AI、Audio、VFXを禁止する。
- Native jobをすべてjoinしてからT60へ進む。World destroyはjoin、callback drain、lease失効の後だけ行う。

## 6. Kernel lock、Build、Qualification

### 6.1 初期`PhysicsKernelLockV1`

| Dimension | Kernel | Version | exact source commit | License | 初期状態 |
|---|---|---|---|---|---|
| 2D | Box2D | 3.1.1 | `8c661469c9507d3ad6fbd2fea3f1aa71669c2fe3` | MIT | `candidate_locked` |
| 3D | Jolt Physics | 5.6.0 | `e77f175595e64cb44218cc9d9d56fc365ad0e36a` | MIT | `candidate_locked` |

状態は`candidate_locked -> qualified -> active -> deprecated -> removed`だけを許可する。CMake configure成功だけで`qualified`へ進めない。

Phase 0のDependency Workerはtagではなく上表のcommitからSource bundleを作り、archive byte数、SHA-512、license file SHA-256、patch set hash、compiler／flag／Targetを`toolchain.lock.json`、SBOM、`PhysicsKernelQualificationReceiptV1`へ記録する。値を取得できない、tagとcommitが一致しない、署名／hashが再現しない場合はlock activationを失敗させる。文書へ未確定hash placeholderを置かず、Receiptが存在しない状態を`candidate_locked`として機械判定する。

取得とBuildは`third_party/ports/box2d`、`third_party/ports/jolt`のvcpkg overlay portを正規入口とする。Vendor sourceをrepositoryへcopyせず、overlayへcommit、archive hash、patch、option、licenseだけを追跡する。vcpkg builtin portが別version、別optionを解決した場合はbuiltinへ追従せずconfigureを失敗させる。

### 6.2 Box2D product build

| Setting | C1 product値 |
|---|---|
| Language／link | C17、static |
| `BUILD_SHARED_LIBS` | `OFF` |
| `BOX2D_DISABLE_SIMD` | `OFF` |
| `BOX2D_AVX2` | `OFF`。Windows baseline binaryをAVX2必須にしない |
| `BOX2D_SAMPLES` | `OFF` |
| `BOX2D_BENCHMARKS` | `OFF` |
| `BOX2D_DOCS` | `OFF` |
| `BOX2D_PROFILE` | `OFF`。Engine spanを使う |
| `BOX2D_UNIT_TESTS` | Product buildは`OFF`、Qualification buildは`ON` |
| `BOX2D_VALIDATE` | Product buildは`OFF`、Qualification buildは`ON` |
| `BOX2D_SANITIZE` | Product buildはEngine ASan policyに従い`OFF`、Qualification ASan buildだけ`ON` |
| `BOX2D_COMPILE_WARNING_AS_ERROR` | `OFF`。External warning policyへ分離 |

`b2SetAllocator`へthread-safeなPhysics Domain allocatorをProcess内最初のBox2D APIより前に一度だけ登録し、最後の2D World破棄まで差し替えない。Box2D allocation bytesとcountをPhysics telemetryへchargeする。Worker callbackはEngine Worker Bridgeだけを使用し、Box2D用thread poolを別に作らない。

### 6.3 Jolt product build

| Setting | C1 product値 |
|---|---|
| Language／link | C++17、static |
| `JPH_BUILD_SHARED_LIBS` | `OFF` |
| `DOUBLE_PRECISION` | `OFF` |
| `CROSS_PLATFORM_DETERMINISTIC` | `OFF` |
| `JPH_USE_DX12`／`JPH_USE_VK`／`JPH_USE_MTL`／`JPH_USE_CPU_COMPUTE` | すべて`OFF` |
| `CPP_EXCEPTIONS_ENABLED`／`CPP_RTTI_ENABLED` | `OFF` |
| `DISABLE_CUSTOM_ALLOCATOR` | `OFF` |
| `USE_STD_VECTOR` | `OFF` |
| `ENABLE_OBJECT_STREAM` | `OFF` |
| `OBJECT_LAYER_BITS` | `16` |
| `OVERRIDE_CXX_FLAGS` | `OFF` |
| `USE_STATIC_MSVC_RUNTIME_LIBRARY` | `OFF`。Engine Toolchain bindingがCRTを一意に決める |
| `INTERPROCEDURAL_OPTIMIZATION` | Upstream optionは`OFF`。Shipping IPOはEngine build policyがVendor targetを含め一括所有する |
| `ENABLE_ALL_WARNINGS` | `OFF`。External warning policyへ分離 |
| `USE_ASSERTS` | Product buildは`OFF`、Qualification buildは`ON` |
| `GENERATE_DEBUG_SYMBOLS`／`FLOATING_POINT_EXCEPTIONS_ENABLED` | `OFF`。Engine configuration policyが所有 |
| `TRACK_BROADPHASE_STATS`／`TRACK_NARROWPHASE_STATS`／`JPH_TRACK_SIMULATION_STATS` | `OFF` |
| `DEBUG_RENDERER_IN_DEBUG_AND_RELEASE`／`DEBUG_RENDERER_IN_DISTRIBUTION` | `OFF` |
| `PROFILER_IN_DEBUG_AND_RELEASE`／`PROFILER_IN_DISTRIBUTION`／`JPH_USE_EXTERNAL_PROFILE` | `OFF` |
| `TARGET_HELLO_WORLD`／`TARGET_PERFORMANCE_TEST`／`TARGET_SAMPLES`／`TARGET_VIEWER`／`ENABLE_INSTALL` | `OFF` |
| `TARGET_UNIT_TESTS` | Product buildは`OFF`、Qualification buildだけ`ON` |
| `USE_SSE4_1`／`USE_SSE4_2` | Windows x64は`ON` |
| `USE_AVX`／`USE_AVX2`／`USE_AVX512`／`USE_LZCNT`／`USE_TZCNT`／`USE_F16C`／`USE_FMADD` | Windows baselineは`OFF` |

Windows baselineは上表のSSE4.1／SSE4.2だけを必須にする。Android／Apple arm64はJoltのNEON／FP16経路を使う。より上位のSIMD binaryはC2のTarget variantとして別Artifact、別Receipt、別Replay environmentを持つ場合だけ追加する。

Joltのglobal `Allocate`、`Reallocate`、`Free`、`AlignedAllocate`、`AlignedFree`をthread-safeなPhysics Domain allocatorへJolt type登録前に一度だけ接続し、最後の3D WorldとJolt type unregister完了まで差し替えない。`RegisterDefaultAllocator()`をProduction経路で呼ばない。

### 6.4 Targetと昇格条件

| Target | Box2D | Jolt | Production表示条件 |
|---|---|---|---|
| Windows x64 | 必須 | 必須 | Primary MSVC、secondary clang-cl、ASan、Reference性能 |
| Android arm64-v8a | 必須 | 必須 | NDK build、実機baseline／standard／high、thermal／resume |
| iOS／iPadOS arm64 | 必須 | 必須 | Xcode archive、実機、background／resume、endurance |

Box2D upstream READMEが明記する通常互換対象はWindows、Linux、Macである。Android／Apple Productionは「portable C17だから動く」と推測せず、MiraikanaiのTarget別compile、ABI、SIMD、実機fixtureに合格して初めて表示する。JoltはupstreamがAndroid／iOSを対応Platformに含めているが、同じくMiraikanaiの実機Gateを省略しない。

## 7. 座標、単位、数値範囲

| 項目 | 2D | 3D |
|---|---|---|
| 座標 | +X right、+Y up | +X right、+Y up、+Z third axis |
| 長さ／時間 | meter／second | meter／second |
| 角度 | radian、正方向は+Z軸周り | radian、right-handed |
| 力 | N | N |
| Torque | N·m scalar | N·m vector |
| 質量 | kg | kg |
| Gravity Reference | `(0, -9.81)` m/s² | `(0, -9.81, 0)` m/s² |

Collision規約のhard範囲を適用する。

- Body position各軸は`[-10,000, 10,000]` m。
- Full extentは`[0.001, 10,000]` m、推奨は`[0.02, 100]` m。
- Linear speedは`[0, 1,000]` m/s。
- Angular speed magnitudeは`[0, 200]` rad/s。
- NaN、Inf、subnormal依存入力、zero axis、非正規rotationをclampまたは既定値へ置換しない。

Visual scaleをNative Bodyへ渡さない。Collider寸法へCookし、Runtime physics scaleはidentityに固定する。

## 8. World、Solver、Worker Profile

### 8.1 `PhysicsWorkerProfileV1`

| Target class | Physics worker数 |
|---|---:|
| Windows Reference | 4 |
| Mobile baseline | 1 |
| Mobile standard | 2 |
| Mobile high | 4 |

この値を`physics_worker_count`とし、Runtime規約がhardwareから決める共有pool総数`shared_worker_pool_count`とは区別する。Main Runtime threadはどちらにも含めない。Physicsは共有pool内で最大`physics_worker_count` slotだけを同時使用し、Library固有thread／poolを作らない。`shared_worker_pool_count < physics_worker_count`ならworker数1へ無通知fallbackせず`MIRA-PHYSICS-WORKER_PROFILE_UNAVAILABLE`でPlay準備を失敗させるか、ユーザー承認済み下位Target Profileへ変更する。両値はPlay開始時に固定してReplay headerへ保存し、OSのhardware concurrency変化からPlay中に再計算しない。

### 8.2 `Physics2DWorldProfile::ReferenceV1`

| Field | C1値 | 許可範囲 |
|---|---:|---:|
| `fixed_delta_numerator`／`denominator` | `1`／`60` | 固定 |
| `gravity_mps2` | `(0, -9.81)` | 各軸`[-1000, 1000]` |
| `sub_step_count` | `4` | integer 1～8 |
| `restitution_threshold_mps` | `1.0` | `[0, 100]` |
| `hit_event_threshold_mps` | `1.0` | `[0, 1000]` |
| `contact_hertz` | `30.0` | `[1, 240]` |
| `contact_damping_ratio` | `10.0` | `[0, 100]` |
| `max_contact_push_speed_mps` | `3.0` | `[0, 100]` |
| `maximum_linear_speed_mps` | `1000.0` | 固定hard上限 |
| `enable_sleep` | `true` | bool |
| `enable_continuous` | `true` | bool |
| `worker_profile_id` | Target classのProfile | closed ID |

Box2D 3.1.1の`b2DefaultWorldDef()`はgravity -10、maximum linear speed 400を含む。Miraikanaiは上表の-9.81と1000へ明示上書きする。他の値も上表から設定し、Vendor defaultの欠落状態をCooked Profileに残さない。Adapterは`workerCount=physics_worker_count`、`enqueueTask`／`finishTask`をEngine Worker Bridge、`userTaskContext`をWorld lifetime内のBridge contextへ設定する。Friction／restitution callbackは`null`のままとし、Box2D 3.1.1の`sqrt(friction_a*friction_b)`／`max(restitution_a,restitution_b)`をEngineの`geometric_mean`／`max` semanticsとしてconformance testで固定する。Project callbackを登録しない。

### 8.3 `Physics3DWorldProfile::ReferenceV1`

| Field | C1値 |
|---|---:|
| `fixed_delta_numerator`／`denominator` | `1`／`60` |
| `gravity_mps2` | `(0, -9.81, 0)` |
| `collision_steps` | `1`、許可1～2 |
| `temp_allocator_bytes` | `33,554,432` |
| `num_body_mutexes` | `64` |
| `max_bodies` | Target Collision Profileのlive Body上限＋内部World Anchor 1 |
| `max_body_pairs` | Target Collision Profile |
| `max_contact_constraints` | Target Collision Profile |
| `maximum_linear_speed_mps` | `1000.0` |
| `maximum_angular_speed_radps` | `200.0` |
| `worker_profile_id` | Target classのProfile |

`PhysicsSolver3DReferenceV1`はJolt 5.6.0 `PhysicsSettings{}`を次へ全展開する。

| Engine field | C1値 |
|---|---:|
| `max_in_flight_body_pairs` | 16,384 |
| `step_listeners_batch_size` | 8 |
| `step_listener_batches_per_job` | 1 |
| `baumgarte` | 0.2 |
| `speculative_contact_distance_m` | 0.02 |
| `penetration_slop_m` | 0.02 |
| `linear_cast_threshold` | 0.75 |
| `linear_cast_max_penetration` | 0.25 |
| `manifold_tolerance_m` | 0.001 |
| `max_penetration_distance_m` | 0.2 |
| `body_pair_cache_max_delta_position_sq_m2` | 0.000001 |
| `body_pair_cache_cos_max_delta_rotation_div2` | 0.9998476951563912 |
| `contact_normal_cos_max_delta_rotation` | 0.9961946980917455 |
| `contact_point_preserve_lambda_max_dist_sq_m2` | 0.0001 |
| `internal_edge_removal_vertex_tolerance_sq_m2` | 0.00000001 |
| `num_velocity_steps` | 10 |
| `num_position_steps` | 2 |
| `min_velocity_for_restitution_mps` | 1.0 |
| `time_before_sleep_s` | 0.5 |
| `point_velocity_sleep_threshold_mps` | 0.03 |
| `deterministic_simulation` | true |
| `constraint_warm_start` | true |
| `use_body_pair_contact_cache` | true |
| `use_manifold_reduction` | true |
| `use_large_island_splitter` | true |
| `allow_sleeping` | true |
| `check_active_edges` | true |

`PhysicsSystem::Init`へはTarget Profileのcapacity、`num_body_mutexes=64`、Engineが生成したBroadPhase Layer interface、Object-vs-BroadPhase filter、Object pair filterを渡す。三つのfilter objectはPhysics Worldより長生きする。Bodyのmax velocityは作成時とProfile再構築時に全件へ明示設定する。

Profile変更は`PlayStopped`または`PlayPreparing`だけ許可し、Play中のsolver値、`physics_worker_count`、sub-step、collision step、capacity変更を拒否する。

## 9. Lifecycle、固定phase、Dynamics command

### 9.1 World lifecycle

```text
Uncreated
-> ProfileValidated
-> KernelQualified
-> NativeWorldCreated
-> StaticBuilt
-> PlayReady
-> Stepping
-> StopRequested
-> JobsJoined
-> NativeWorldDestroyed
```

`KernelQualified`を通らないTargetでNative Worldを作らない。Play開始前にBody、Shape、Joint、pair、contact、event、query scratchを予約する。2Dと3Dを同一Sceneで使う場合は別Worldとし、Runtime規約の固定`producer_system_id`順、すなわち2D World、join、3D World、joinの順で直列stepする。Stable IDの生成順やthread完了順で順序を変えず、両Worldを直接collisionさせない。

C1の一つのGameHostでactiveにできるWorldは2D最大1、3D最大1である。複数Scene／streaming cellは該当dimensionの同じWorldへ参加する。Editorの比較Previewは別Worker ProcessでWorldを作り、そのProcessのTool budgetへchargeする。二つ目の3D Worldを同じ96 MiB Physics Domainへ作って32 MiB TempAllocatorを暗黙追加しない。Multi-world simulationはC3 Capabilityとする。

### 9.2 Phase

| Phase | Physics処理 |
|---|---|
| `T00_BoundaryApply` | Body／Collider／Joint create・destroy、Profile互換promotion |
| `T30_PrePhysics` | Gameplay command／query requestをlatch |
| `T40_MotionIntent` | Character Motor、root motion解決、query、command validate／canonical merge |
| `T50_PhysicsStep` | Box2D／Joltを60 Hzでstep。AdapterだけがNative World write |
| `T60_PhysicsIntegrate` | Transform、velocity、sleep、contact／trigger／joint eventをcopy、normalize、sort |
| `T70_PostPhysics` | Gameplayへtyped event配送。構造変更は次T00へ提出 |
| `T100_ReplayRecord` | 入力Command、Profile／Build ID、state hashを記録 |

Dynamic BodyのTransform writerはT60だけである。Kinematic targetはT40で確定しT50へ渡す。Animation、Navigation、Rendering、Audio、AIがPhysics Transformへwriteしない。

### 9.3 `PhysicsDynamicsCommandV1`

| Command | 対象 | consume |
|---|---|---|
| `ApplyForce` | dynamic Body | T40 |
| `ApplyTorque` | dynamic Body | T40 |
| `ApplyImpulse` | dynamic Body | T40 |
| `SetLinearVelocity` | Capability許可済みdynamic Body | T40 |
| `SetAngularVelocity` | Capability許可済みdynamic Body | T40 |
| `SetKinematicTarget` | kinematic Body | T40 |
| `TeleportBody` | 全Body kindの明示policy | T40 |
| `WakeBody`／`SetSleepPermission` | dynamic Body | T40 |
| `SetGravityFactor` | Cook済み範囲を持つdynamic Body | T40 |
| `SetJointMotorTarget`／`SetJointLimit` | 対応Joint | T40 |
| `BreakJoint` | Joint | 次T00 |

各Commandは`target_handle`、expected generation、consume tick、producer system ID、producer sequence、priority、payloadを持つ。同一BodyへのCommandは`priority, producer_system_id, producer_sequence`でcanonical mergeする。TeleportはKinematic target、velocity、forceより高く、同tickの低いCommandをtyped conflictとして拒否する。ForceとImpulseを暗黙変換しない。

Runtime規約のSimulation command総数65,536／tick、4 MiB arenaを共有し、Physicsが別の無制限queueを持たない。上限超過はpartial適用せずtickをpublishしない。

## 10. Joint／Constraint

### 10.1 共通契約

```text
PhysicsJointCommonV1
  joint_id: StableId
  world_id: StableId
  body_a/body_b: StableId | WorldAnchor
  enabled: bool
  collide_connected: bool
  local_frame_a/local_frame_b: typed 2D or 3D frame
  break_force_n: optional finite [0.001, 1e9]
  break_torque_nm: optional finite [0.001, 1e9]
```

Joint種別はtagged unionとし、存在しないfieldをproperty bagへ入れない。World anchorは専用variantで表現し、null Body IDで代用しない。各Physics WorldはNative World作成直後に一つの非公開static World Anchor Bodyを原点へ最初に作り、`WorldAnchor`をこのBodyへ接続する。Anchorはshapeを持たず、内部専用layerに置き、contact／trigger／public query／debug selection／Save／Eventから除外し、World破棄までgenerationを変えない。これはProject live Body数へ数えないが、Native `max_bodies`へ1件予約する。Body destroy ChangeSetは接続Jointを同一transactionで削除するかprecondition failureとする。

Joint stressはT60でSI単位へ変換し、magnitudeが設定thresholdを厳密に超えた最初のtickで`JointBreakEventV1`候補にする。EventはStable ID、measured force／torque、threshold、tickを持ち、T70配送、Component削除は次T00である。Cross-backendでreaction値のbitwise一致は要求しない。

### 10.2 2D C1 type

| Type | 必須fieldと範囲 |
|---|---|
| `DistanceJoint2DV1` | `length_m [0.001,10000]`、optional min／max、spring `frequency_hz [0,120]`、`damping_ratio [0,10]`、motor speed `[-1000,1000]` m/s、max force `[0,1e9]` N |
| `RevoluteJoint2DV1` | reference／target angle、limit `[-0.99π,+0.99π]`かつlower<=upper、spring、motor speed `[-200,200]` rad/s、max torque `[0,1e9]` N·m |
| `PrismaticJoint2DV1` | unit local axis、translation limit `[-10000,10000]` mかつlower<=upper、spring、motor speed `[-1000,1000]` m/s、max force `[0,1e9]` N |
| `WeldJoint2DV1` | reference angle、linear／angular frequency `[0,120]` Hz、linear／angular damping ratio `[0,10]` |

Distanceの`min_length_m <= rest_length_m <= max_length_m`を必須とする。Axis長は`[0.9999,1.0001]`だけ受理してCook時に一度normalizeし、zero vectorを既定軸へ置換しない。Revoluteの±0.99πはBox2D 3.1.1のlimit制約に合わせたC1 hard範囲である。

### 10.3 3D C1 type

| Type | 必須fieldと範囲 |
|---|---|
| `FixedConstraint3DV1` | 2 Body frame。自由度0 |
| `PointConstraint3DV1` | 2 anchor position。回転自由 |
| `DistanceConstraint3DV1` | distance `[0.001,10000]` m、optional min／max、spring |
| `HingeConstraint3DV1` | unit hinge axisとnormal、limit `[-π,+π]`、motor |
| `SliderConstraint3DV1` | unit slider axisとnormal、translation `[-10000,10000]` m、motor |
| `SwingTwistConstraint3DV1` | unit twist／plane axis、normal half-cone `[0,π]`、plane half-cone `[0,π]`、twist min／max `[-π,+π]` |

3D frameのaxisは各長`[0.9999,1.0001]`、basisの絶対dotは`<=0.0001`を必須とする。Motor modeは`off | velocity | position_and_velocity`のclosed enumで、velocity、target、force／torque limitを型別に持つ。Jolt 5.6.0のVendor enum値を保存しない。

### 10.4 C2／C3

- C2: 2D wheel／motor、3D SixDOF、ragdoll authoring、vehicle constraint、destructible constraint group。
- C3: pulley／gear／rack、soft-body constraint、network replicated constraint、custom solver callback。

C1の汎用Jointへunknown fieldを足してC2を先取りしない。新しいtypeは新Capability ID、MCD version、Editor、AI vocabulary、fixtureを同じChangeSetで追加する。

## 11. Engine-owned Kinematic Character Motor

### 11.1 採用方式

C1のPlayer／NPC Characterはdynamic rigid bodyへ直接velocityを設定する方式を既定にせず、2D／3Dで同じ意味を持つEngine-owned Kinematic Character Motorを使用する。Jolt `CharacterVirtual`／`Character`やBox2D sample moverをProject APIへ採用しない。BackendはCollision規約のshape cast／overlapを実行するだけで、状態機械と解決順はMiraikanaiが所有する。

### 11.2 Profile

| Field | 2D Reference | 3D Reference | hard範囲 |
|---|---:|---:|---:|
| `capsule_radius_m` | 0.30 | 0.35 | `[0.05,5]` |
| `capsule_half_segment_m` | 0.40 | 0.55 | `[0,5]` |
| `skin_m` | 0.01 | 0.03 | `[0.001,min(0.1,0.25*radius)]` |
| `step_height_m` | 0.25 | 0.35 | `[0,2]` |
| `max_slope_deg` | 50 | 50 | `[0,89]` |
| `ground_snap_m` | 0.10 | 0.15 | `[0,2]` |
| `max_slide_iterations` | 4 | 4 | integer 1～8 |
| `max_overlap_iterations` | 4 | 4 | integer 1～8 |
| `max_depenetration_per_tick_m` | 0.25 | 0.25 | `[0.001,1]` |
| `max_move_speed_mps` | 20 | 20 | `[0.01,100]` |

Capsule full heightは`2 * radius + 2 * half_segment`である。Step heightはfull height未満、ground snapは`max(step_height, 4*skin)`以下、skinはradius未満を必須とする。

### 11.3 Input、State、Output

`CharacterMoveIntentV1`はCharacter handle、consume tick、planar displacement、vertical velocity proposal、jump edge、up direction、root-motion delta、producer metadataを持つ。

Stateは`Disabled | Airborne | Grounded | Sliding | Stepping | CeilingBlocked`のclosed enumである。Outputはresolved pose、resolved velocity、state、ground Body handle／generation、ground normal、ground relative point、applied platform delta、hit summary、Diagnosticを持つ。

### 11.4 T40解決手順

次の順を変更しない。

1. Intent、Character generation、Profile、finite、speed、World versionを検証する。
2. 前tickのground Body generationを再検査し、有効なkinematic／dynamic platformなら前tickからのplatform deltaを先に適用候補へ加える。
3. Current capsule overlapを検査し、penetration depth降順、Stable Body handle昇順、shape slot昇順でProfileの最大回数まで解消する。各回は最深contact normalへ`min(depth + skin, remaining_depenetration_budget)`だけ移動する。合計がProfileの`max_depenetration_per_tick_m`を超える、解消不能、dynamic-only enclosureの場合はfaultにする。
4. 入力のplanar displacementをsweepし、最短fraction、Stable Body handle、shape slotでhitを決める。
5. 残り移動をhit planeへprojectし、Profileの`max_slide_iterations`までslideする。同一normalを繰り返して進捗がskin未満なら終了する。
6. Groundedまたはground snap候補中に非walkable obstacleへ当たった場合だけ、`up -> forward -> down`の三sweepでstep候補を作る。Upは`step_height + skin`、Forwardは未解決planar displacement、Downは`step_height + ground_snap + 2*skin`を上限とする。上方向clear、前進量がskin超、down hitがwalkable、最終overlapなしの全条件を満たす時だけ採用する。
7. Gravity／jumpを含むvertical displacementをsweepする。Jump edgeがあるtickはground snapを禁止する。
8. `dot(hit_normal, up) >= cos(max_slope_deg)`をwalkableとする。それ未満でdownward motionなら`Sliding`へ遷移する。
9. Descending、jumpなし、snap距離内、walkable hitの場合だけground snapする。
10. 最終poseでoverlap、hard range、speedを再検査し、Kinematic targetをT50へ提出する。

Hit tie-breakは`fraction, packed body handle, shape slot, subshape index, point lexicographic`である。Native callback arrival順を使用しない。2Dではsubshape indexが存在しない場合0とする。

### 11.5 Moving platform、Dynamic Body、Animation

- Ground attachmentはStable Body handle、generation、local contact pointを保存し、Native pointerを保存しない。
- Platformがteleport、destroy、generation変更した場合はattachmentを切り、`Airborne`から再判定する。
- Platform deltaが10 m／tickまたは200 rad/s相当を超える場合は追従せずDiagnosticにする。
- Characterがdynamic Bodyを押す効果はT50のkinematic contact responseで生じる。T40 callbackからforceを直接適用しない。
- Root motionはAnimationのproposalであり、Gameplay planar intentとの合成policyをCharacter Profileへ`gameplay_only | root_motion_only | additive_bounded`で明示する。
- Motor resolved poseがauthoritativeであり、AnimationはT80でそのposeを読む。Animation TransformをPhysicsへ二重writeしない。

## 12. AI、GameplayDefinition、Project C++、手動編集

### 12.1 AIへ公開するCapability

```text
capability.physics.world_2d_v1
capability.physics.world_3d_v1
capability.physics.dynamics_v1
capability.physics.joints_2d_v1
capability.physics.constraints_3d_v1
capability.physics.character_motor_v1
capability.physics.save_replay_v1
capability.physics.debug_v1
```

AIは`scene_dimension`／`gameplay_space`から2Dまたは3D Capabilityを選ぶ。Backend名、Kernel version、CMake option、worker thread APIを選ばない。同じWorld Entityへ2Dと3D Bodyを同時付与しない。

### 12.2 AI Authoring Operation

| Operation ID | 動作 | Risk |
|---|---|---|
| `operation.physics.inspect_world` | World、Body、Joint、Character、budgetを読む | R0 |
| `operation.physics.validate_changeset` | Schema、semantic、cost、Profileを検査 | R0 |
| `operation.physics.preview_changeset` | 挙動、memory、fixture差をPreview | R0 |
| `operation.physics.create_world_profile_2d`／`3d` | Referenceを複製し全fieldを保存 | R2 |
| `operation.physics.update_world_profile` | Play停止中Profileを変更 | R2 |
| `operation.physics.configure_body_dynamics` | mass、damping、sleep、motion qualityを変更 | R2 |
| `operation.physics.create_joint_2d`／`constraint_3d` | 型付きJointを作成 | R2 |
| `operation.physics.update_joint`／`delete_joint` | Jointを変更／削除 | R2 |
| `operation.physics.create_character_motor` | ProfileとComponentを作成 | R2 |
| `operation.physics.update_character_motor`／`delete_character_motor` | Profile／Componentを変更または削除 | R2 |
| `operation.physics.generate_fixture` | stack、slope、stair、joint等の検証Scene案 | R2 |
| `operation.physics.run_qualification` | 許可済みToolでread-only reportを作成 | R1 |
| `operation.physics.propose_kernel_upgrade` | Version／Build／behavior変更を提案 | R4 |

全write Operationは`ProjectChangeSet`を作るだけで、Runtime Worldを直接変更しない。Engine Validatorが再検証し、人間または事前委任Policyの承認後だけCommitする。

### 12.3 Level 0の質問と仮定

AIは初心者へ「Box2DかJoltか」「solver iterationはいくつか」と質問しない。次だけをゲーム上の言葉で確認する。

| 不足事項 | 扱い |
|---|---|
| 2D／3D／HybridとHybrid gameplay space | Blocking |
| Player移動が歩行、飛行、車両、物理ragdollのどれか | High Impact |
| 高速Projectileがauthoritative hitを必要とするか | High Impact |
| 押せるobject、壊れるJoint、moving platformの有無 | High Impact |
| Objectの概算最大数、mobile対応、同時敵数 | High Impact |
| 一般propのmass／friction、内部solver field | Reference値を複製してAssumption記録 |

未指定の壁／床はstatic、持ち物や小型propはprimitive／convex dynamic、通常PlayerはKinematic Character Motor、高速Projectileはswept queryをReference仮定とする。AIは仮定、理由、変更影響をGame BriefとDecision Ledgerへ記録し、Userが会話またはInspectorから変更できるようにする。

### 12.4 構造化dataとC++の選択

- World Profile、Body、Collider、Joint、Character Profile、filter、material、Gameplay parameterは構造化dataを使う。
- Force適用条件、Character ability、damage、quest、game-specific state machineはGameplayDefinitionを優先する。
- bounded operationでは表現できないProject固有behavior、またはBenchmarkで必要性が証明されたhot pathはProject C++を使う。
- 新しいsolver、Joint type、contact modification、Backend setting、Physics phaseはProject C++の自由拡張にせずEngine R4変更とする。

AIはLevel 0でも必要なGameplayDefinitionとProject C++を生成できるが、どちらもtyped Physics Portだけを使い、Vendor include、raw pointer、callback登録、任意World stepを生成しない。

### 12.5 手動編集

人間はScene Gizmo、Outliner、Inspector、Profile表、Joint Graph、Character Preview、C++ Editor／外部IDEから編集できる。GUI操作とAI操作は同じAuthoring Document、ChangeSet、Validator、Preview、Undo／Redo、Cookを使う。上級者向け画面だけに存在する非検証write経路を作らない。

## 13. EditorとDebug

`Physics` Panelは次を提供する。

- World Profileと全resolved solver値。Vendor defaultという表示を持たない。
- Body／Joint／CharacterのStable ID、状態、budget、Owner、Sourceへのnavigation。
- Joint anchor／axis／limit／motor／break threshold Gizmo。
- Character capsule、skin、slope cone、step三sweep、ground snap、moving platform attachmentの可視化。
- Body、AABB、center of mass、sleep、velocity、contact、island、broadphase、constraint errorのimmutable debug snapshot。
- tick／sub-step単位のPhysics timeline、Command merge、Event、capacity、allocation、P95。
- Save→Load state差、Replay最初の不一致tick、Kernel／Profile／Build Receipt。
- Source、Preview、Committed、Runtimeの状態を色だけでなくlabel、line pattern、iconで区別。

Panelは独自MiraUIの通常dockとしてresize、dock、floating、multi-monitor、Workspace保存へ対応する。Scene ViewやProfilerからNative Worldを直接参照せず、T60後のbounded `PhysicsDebugSnapshotV1`だけを読む。

AI Partnerは選択中Body／Joint／CharacterのSemantic Snapshot、Diagnostic、budgetを取得できるが、screen座標やpixelから状態を推測しない。AI提案、Engine validation結果、CommitされるDiffを別表示にする。

## 14. Save、Load、Replay、Determinism

### 14.1 Save対象

`PhysicsSaveStateV1`は次だけを保存する。

- Physics World Stable IDとProfile content hash。
- Body Stable ID、kind、Engine-visible pose、linear／angular velocity、sleep permission／sleep state。
- Collider Asset Stable IDと互換generation。
- Joint Stable ID、type、enabled、motor／limitのEngine-visible runtime state、破断済みstate。
- Character pose、velocity、state、ground Stable ID／generation、Profile hash。
- Gameplay-owned physics state。

Native ID／pointer、broadphase node、contact cache、manifold、warm-start impulse、solver island、worker job、Jolt Object Layer番号を保存しない。

### 14.2 Load手順

1. Save schema、Project revision互換、World／Collision／Physics Profile hash、Asset versionを検証する。
2. 現Worldのjobをjoinし、Physics leaseを失効させる。
3. Stable ID昇順でstatic、kinematic、dynamic Bodyを再構築する。
4. Stable Joint ID昇順でJointを再構築する。
5. Pose、velocity、sleep permission、enabled stateを適用する。
6. Character ground attachmentをBody generationと接触再検査に通し、無効ならAirborneにする。
7. Validation snapshotを作り、成功後だけ次の通常60 Hz tickへ公開する。

Load専用の隠れsettle stepを実行しない。互換しないColliderを近似primitiveへ置換せず、Loadを失敗させる。

### 14.3 Replay environment

`PhysicsReplayHeaderV1`は次を必須とする。

- Box2D／Jolt exact version、commit、Source bundle hash。
- Target、CPU architecture、Compiler full version、C++／SIMD／floating-point Build Profile hash。
- Physics Kernel、World、Solver、Collision、Worker Profile hash。
- exactly 60 Hz、sub-step／collision step、`shared_worker_pool_count`、`physics_worker_count`。
- Project revision、Cooked Asset closure hash、initial Save hash。

同一header、同一入力Command列、同一Buildではstate hashとnormalized Event digestを一致させる。Jolt `deterministic_simulation=true`を使うが、Windows／Android／Apple、x64／arm64、Box2D／Jolt間のbitwise一致やnetwork lockstepを保証しない。Joltの`CROSS_PLATFORM_DETERMINISTIC`はC1で無効である。

Replay不一致時は最初のtick、最初のBody／Joint Stable ID、field、expected／actual bit patternをreportする。最終frameだけ比較しない。

## 15. Memory、容量、性能

### 15.1 容量

Body、shape、pair、contact、query上限はCollision規約のTarget Profileを使う。本書で追加する上限は次である。

| Field | Windows C1 | Mobile baseline | Mobile standard | Mobile high |
|---|---:|---:|---:|---:|
| live Joint／Constraint | 8,192 | 4,096 | 8,192 | 16,384 |
| live Character Motor | 2,048 | 1,024 | 2,048 | 4,096 |
| Joint break event／tick | 4,096 | 2,048 | 4,096 | 8,192 |

Projectは低い上限を選べる。高い上限はmemory、2.50 ms P95、10分soak、mobile thermalを再実行したADRなしに許可しない。

### 15.2 Windows Physics Domain 96 MiB

| 用途 | MiB |
|---|---:|
| Normalized Physics event二面buffer | 12 |
| Jolt TempAllocator | 32 |
| Box2D／Jolt World、Body、Shape、Joint、broadphase | 40 |
| Mapping、trigger、query、Character、Joint registry、Replay scratch | 8 |
| non-lendable reserve | 4 |
| **合計** | **96** |

2Dと3Dを同時使用しても96 MiBを二倍にしない。Jolt TempAllocator枯渇、Vendor persistent allocation、registry、event stagingがcapを超えた場合は一般heapやUnassigned headroomへfallbackしない。T50中の一般heap allocation countは0とする。

### 15.3 性能Gate

- Windows ReferenceのPhysics全体P95は2.50 ms以下。
- Collision Query総時間P95 0.50 ms、単一public query P99 0.10 msをPhysics budget内のsoft diagnosticとする。
- Character Motor全体P95 0.40 ms、単一Character P99 0.05 msをsoft diagnosticとする。
- Joint solverだけのhard budgetを別に作らず、Physics全体、Body／Joint count、island、iterationと一緒に測る。
- Warm-up後10分を5 run実行し、各run P95のmedianを判定値にする。
- Mobileは30分thermalと2時間endurance、background／resume後のWorld再構築を追加する。

Kernel version、SIMD、solver field、`physics_worker_count`を変更した比較は同一fixture、同一Build profile、同一power policyでBefore／Afterを測定する。平均値だけ、Vendor sampleだけ、単一desktopだけで採用しない。

## 16. DiagnosticとFailure

| ID | 条件 | 動作 |
|---|---|---|
| `MIRA-PHYSICS-KERNEL_UNQUALIFIED` | Target／BuildのQualification Receiptなし | World作成拒否 |
| `MIRA-PHYSICS-KERNEL_LOCK_MISMATCH` | version、commit、hash、option不一致 | configure／Play拒否 |
| `MIRA-PHYSICS-WORKER_PROFILE_UNAVAILABLE` | 要求workerを提供不能 | Play準備拒否 |
| `MIRA-PHYSICS-PROFILE_INVALID` | solver値、60 Hz、capacity不正 | Commit／Cook拒否 |
| `MIRA-PHYSICS-COMMAND_CONFLICT` | 同一targetの排他的Command | 低priority Command拒否 |
| `MIRA-PHYSICS-JOINT_INVALID` | Body、frame、axis、limit、motor不正 | Commit／Play拒否 |
| `MIRA-PHYSICS-JOINT_CAPACITY_EXCEEDED` | live Joint上限 | tick非publish／Play fault |
| `MIRA-PHYSICS-CHARACTER_PROFILE_INVALID` | capsule、skin、step、slope不正 | Commit拒否 |
| `MIRA-PHYSICS-CHARACTER_DEPENETRATION_EXCEEDED` | overlap解消上限 | tick非publish／Play fault |
| `MIRA-PHYSICS-CALLBACK_REENTRY` | callback中World／Gameplay再入 | Development assert、Release fault |
| `MIRA-PHYSICS-TEMP_MEMORY_EXHAUSTED` | Temp／scratch枯渇 | tick非publish／Play fault |
| `MIRA-PHYSICS-NATIVE_UPDATE_FAILED` | Jolt error、Box2D invariant、non-finite | tick非publish／Play fault |
| `MIRA-PHYSICS-SAVE_INCOMPATIBLE` | Profile／Asset／Schema不一致 | Load失敗、旧World維持 |
| `MIRA-PHYSICS-REPLAY_ENVIRONMENT_MISMATCH` | Build／Profile／worker不一致 | Replay開始拒否 |
| `MIRA-PHYSICS-REPLAY_DIVERGED` | state／Event digest不一致 | 最初の不一致で停止しreport |

Failed tickのTransform、Event、Debug Snapshotを部分publishしない。Play fault後はlast completed tickのread-only snapshotをEditorへ残し、Gameの継続を成功扱いしない。

## 17. Test、Qualification、完了条件

### 17.1 Kernel Qualification

各Kernel、Target、Compiler、SIMD Profileについて次を実行する。

1. Source commit、archive hash、license、patch、Build option、SBOMを照合する。
2. Clean static build、upstream UnitTests、ASan、Miraikanai Adapter conformanceを実行する。
3. Product binaryのexport／import／symbol／include scanでVendor API漏出を0件にする。
4. Allocator、ASan poison／unpoison、red-zone越境／use-after-free、worker、callback、World lifecycle、shutdown、fault injectionを検証する。
5. Reference fixtureでmemory、allocation、P95、Replay 100回を測定する。
6. Android／Appleは実機compile、SIMD、alignment、resume、thermal、enduranceを実行する。
7. `PhysicsKernelQualificationReceiptV1`へ全入力hash、結果、例外0件、Owner、時刻、有効期限を署名する。

一つでも不合格なら当該Targetの状態は`candidate_locked`のままで、Capability ManifestへProductionと表示しない。

### 17.2 Joint fixture

- 2D／3Dの各C1 typeでanchor、limit lower／exact／upper、motor off／velocity／position、spring、breakを検証する。
- zero axis、非直交basis、inverted limit、stale Body、World跨ぎ、BodyとJoint同時destroyをnegative testにする。
- 1000回create／destroy churn、8,192 Joint capacity、large island、sleep／wakeを10分soakする。
- Cross-backendでtrajectoryを一致させず、自由度、limit超過なし、event順、failure semanticsを一致させる。

### 17.3 Character fixture

| Fixture | 合格条件 |
|---|---|
| Flat／slope | 50°以下walkable、境界超はSliding |
| Stair | Reference stepは成功、上限+1 mmは失敗 |
| Narrow passage | skin込みで通過可否がProfileどおり |
| Initial overlap | 4回／0.25 m内で解消、超過はtyped fault |
| Corner／wall | 4 slide以内、振動／速度増幅なし |
| Ceiling／jump | CeilingBlocked、ground snap無効 |
| Moving platform | generation検査、local point追従、teleport時detach |
| Dynamic prop | no callback write、T50 response、hard speed内 |
| Root motion | policyどおり一度だけ合成 |
| Replay | 同一環境100 runでdigest不一致0 |

### 17.4 Save／Load、AI、Editor

- Contact中、sleep中、Joint motor中、Characterがplatform上の各Saveを100回Loadし、最初の通常tickから検証する。
- Native ID、cache、pointerがSave／Project／Provider Schemaに存在しないことをmachine scanする。
- AI promptを2D／3D／Hybrid、Character、Projectile、Joint、moving platform、曖昧、矛盾、unsupportedで各12件以上、各3回実行する。
- AIのBackend直接選択、Vendor field生成、無効Joint、unsafe Commitを0件にし、質問／Assumption／Diffの期待一致を95%以上にする。
- AI作成→Inspector変更→C++ behavior追加→Undo／Redo→再Previewが同じChangeSet／Validator経路で往復する。

### 17.5 C1 Production完了条件

1. 本書のMCD type、operation、capability、profile、diagnosticが正本化される。
2. C++／TypeScript／Cooked descriptor／MCP／Provider projectionが同じMCDから生成される。
3. Box2D／Jolt型がPublic Header、Module interface、Project C++、Save、Event、AI Schemaへ出ない。
4. 対象TargetのKernel Qualification、Adapter、Joint、Character、Save／Replay fixtureが全合格する。
5. Physics 96 MiB、T50 allocation 0、P95 2.50 ms、10分soakを満たす。
6. Mobileを表示するTargetは30分thermal、2時間endurance、resumeを実機で満たす。
7. AI unsafe Commit 0、期待挙動95%以上を満たす。
8. Source、Build、Cook、Test、Qualification、Promotion Receiptがhashで連結される。

2D合格を3D合格、Windows合格をmobile合格として表示しない。

## 18. 更新、互換性、独自solver研究

Kernel更新は通常dependency updateではなくPhysics behavior変更として扱う。

1. 新versionを別`candidate_locked` entryへ追加する。
2. Release note、API diff、default diff、license、security、build optionをEvidenceへ保存する。
3. 旧／新Kernelを同一fixtureでQualificationし、trajectory／contact／save reconstruction差をreportする。
4. Project source migrationが必要ならoffline Migrator、backup、Diff、再検証を作る。
5. Save compatibility、Replay environment、performance baselineを更新する。
6. 人間R4 Review後にだけ`active`を切り替える。

Runtimeへ旧／新Kernel branchを長期共存させない。Pre-1.0では後方互換shimを追加せず、offline migrationまたは明示的な非互換version更新を行う。

独自solver研究はC3で次をすべて満たす場合だけ開始する。

- 現Kernelで解決できない具体的なProduct要件と再現fixture。
- Box2D／Jolt拡張、別kernel、Game-side近似との比較。
- 数値安定性、CCD、sleep、Joint、mobile、determinism、toolingを含む人員／期間見積り。
- 同一APIで既存kernelを上回る性能または機能証拠。
- 失敗時に既存Active Kernelへ戻せるAdapter境界。

「AIが理解しやすそう」という理由だけで独自solverへ進まない。

## 19. 公式資料と採用根拠

調査基準日は2026-07-20である。外部資料はLibrary／Engineの事実確認に使い、Miraikanai固有のSchema、Profile、Editor、AI Policy、数値範囲をコピーしない。

| 公式資料 | 確認事項 | 本書への反映 |
|---|---|---|
| [Unity 6 Physics Manual](https://docs.unity3d.com/ja/current/Manual/PhysicsSection.html) | Built-in 3D PhysX、2D Box2D、DOTS Unity Physics／Havok | 独自Product契約＋用途別kernel統合の比較根拠 |
| [Unreal Engine 5.8 Physics](https://dev.epicgames.com/documentation/en-us/unreal-engine/physics-in-unreal-engine) | Chaos Physicsと機能範囲 | First-party solver案の規模比較 |
| [Godot 4.6 Release](https://godotengine.org/releases/4.6/) | 新規3D ProjectでJoltをdefault化 | Jolt integrationのProduction事例 |
| [Godot 4.6 Jolt Guide](https://docs.godotengine.org/en/4.6/tutorials/physics/using_jolt_physics.html) | JoltとEngine APIの差、互換注意 | Semantic conformanceをtrajectory一致としない |
| [Box2D v3.1.1 release](https://github.com/erincatto/box2d/releases/tag/v3.1.1) | exact stable source | Kernel lock |
| [Box2D v3.1.1 README](https://github.com/erincatto/box2d/blob/v3.1.1/README.md) | C17、data-oriented、multithread、feature、license | 2D候補評価 |
| [Box2D 3.1 Simulation](https://box2d.org/documentation/md_simulation.html) | 60 Hz、sub-step、event、task、ID lifetime | World／step／callback contract |
| [Box2D v3.1.1 `types.c`](https://github.com/erincatto/box2d/blob/v3.1.1/src/types.c) | `b2DefaultWorldDef()`の正確な値 | 2D全値展開とMiraikanai override |
| [Box2D v3.1.1 `types.h`](https://github.com/erincatto/box2d/blob/v3.1.1/include/box2d/types.h) | World／Joint field、worker制約 | Profile／Joint schema |
| [Box2D v3.1.1 `base.h`](https://github.com/erincatto/box2d/blob/v3.1.1/include/box2d/base.h) | Custom allocator API | Physics Domain接続 |
| [Jolt Physics v5.6.0 release](https://github.com/jrouwe/JoltPhysics/releases/tag/v5.6.0) | exact commit、friction変更、GPU／hair WIP、C++26修正 | CPU rigid body限定、version再Qualification |
| [Jolt v5.6.0 README](https://github.com/jrouwe/JoltPhysics/blob/v5.6.0/README.md) | Platform、feature、C++17、MIT、determinism範囲 | 3D候補評価とTarget Gate |
| [Jolt v5.6.0 Architecture](https://github.com/jrouwe/JoltPhysics/blob/v5.6.0/Docs/Architecture.md) | Body lock、Broad／Narrow phase、concurrency | Adapter lifetime／lock |
| [Jolt v5.6.0 `PhysicsSettings.h`](https://github.com/jrouwe/JoltPhysics/blob/v5.6.0/Jolt/Physics/PhysicsSettings.h) | 全solver既定値 | 3D Solver Profile全値展開 |
| [Jolt v5.6.0 Build CMake](https://github.com/jrouwe/JoltPhysics/blob/v5.6.0/Build/CMakeLists.txt) | CPU／GPU、determinism、IPO、CRT、RTTI／exception option | Product build profile |
| [Jolt v5.6.0 ContactListener](https://github.com/jrouwe/JoltPhysics/blob/v5.6.0/Jolt/Physics/Collision/ContactListener.h) | callbackはmultithread、Body lock中、変更不可 | copy-only callback |
| [Jolt deterministic simulation](https://jrouwe.github.io/JoltPhysics/#deterministic-simulation) | same binary determinismとcross-platform optionの制約 | Replay保証範囲 |
