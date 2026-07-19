# Miraikanai Engine Physics Dynamics／Navigation／Animation規約

- 文書版: 1.0
- 作成日: 2026-07-19
- 対象: 2D／3D Physics step、Joint／Constraint、Navigation build／query、2D／3D Animation graph／pose
- 状態: プロジェクト公式の規範設計レビュー版
- Collision正本: [Miraikanai Engine Collision／Colliderアーキテクチャ規約](./2026-07-19-collision-collider-architecture-design.md)
- Runtime正本: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- 機能範囲: [Miraikanai Engine 2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)
- 基盤規約: [Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)
- Asset規約: [Miraikanai Engine Asset Pipeline／Content Package規約](./2026-07-19-asset-pipeline-content-packaging-design.md)

## 1. 結論

Physics、Navigation、Animationは相互にpointerを渡す一体型Subsystemにしない。Runtime Orchestratorの固定phase、typed command／event、immutable snapshot、versioned Derived Assetだけで連携する。

- Physicsの正規authoritative stateはEngine-owned Body／Joint ComponentとPhysics Adapter内のsession stateである。
- Collision shape、filter、query、contact、character motorはCollision規約を正本とする。
- Navigation mesh／gridは再生成可能なDerived Assetであり、Worldの正本ではない。
- Animation GraphはMiraikanai独自のtyped graphであり、ozz-animationはoffline／sampling／blend primitiveとしてAdapter内で使う。
- Box2D、Jolt、Recast／Detour、ozzのID、pointer、callback、serializationをProject、AI、Saveへ公開しない。

## 2. 決定権と成熟度

| 主題 | 正本 |
|---|---|
| Shape、Collider、Body field、Filter、Query、Contact、Character Motor | Collision規約 |
| Tick、writer、event順、queue、handle、budget、promotion | Runtime規約 |
| World step、Joint／Constraint、Dynamics command | 本書Physics節 |
| 2D grid／3D navmesh、query、avoidance、link | 本書Navigation節 |
| Clip、Graph、state、root motion、pose、event、IK境界 | 本書Animation節 |
| C1／C2 feature listとReference Scene | 2D／3D機能計画 |

C1は2D sliceでBox2D、Grid Nav、Flipbook、3D sliceでJolt、Recast／Detour、ozzをProduction化する。Vehicle、ragdoll、advanced crowd、retarget、motion warpingはC2。Soft body、fluid、GPU Physics、network lockstep、runtime arbitrary navmesh generation、ML motion synthesisはC3である。

# Part I: Physics Dynamics

## 3. Physics World

### 3.1 2D Profile

`Physics2DWorldProfile::ReferenceV1`を次で固定する。

| Field | 値／範囲 |
|---|---|
| `fixed_delta_s` | exactly `1/60` |
| `gravity_mps2` | `(0, -9.81)`、各軸finite `[-1000, 1000]` |
| `sub_step_count` | 既定4、integer 1～8 |
| `enable_sleep` | true |
| `enable_continuous` | true |
| `restitution_threshold_mps` | 1.0、`[0, 100]` |
| `hit_event_threshold_mps` | 1.0、`[0, 1000]` |
| `worker_count` | Engine Worker Bridgeが起動時に決定しReplay headerへ保存 |

Box2Dの`b2DefaultWorldDef()`はbaselineとしてAdapter testに固定するが、Engineが使用する全fieldをCooked Profileへ明記する。vendor version更新でdefaultが変わっても自動採用しない。

### 3.2 3D Profile

`Physics3DWorldProfile::ReferenceV1`を次で固定する。

| Field | 値／範囲 |
|---|---|
| `fixed_delta_s` | exactly `1/60` |
| `gravity_mps2` | `(0, -9.81, 0)`、各軸finite `[-1000, 1000]` |
| `collision_steps` | 既定1、integer 1～2 |
| `allow_sleep` | true |
| `deterministic_simulation` | 同一Build／Target／worker設定でtrue |
| `temp_allocator_bytes` | exactly 32 MiB／World |
| `max_bodies`／pair／contact | Collision規約のTarget Profile |
| `worker_bridge` | Engine共通Worker pool |

Jolt `PhysicsSettings{}`と`PhysicsSystem::Init`に渡す値はCooked Profileから明示変換する。BroadPhase Layer／Object Layer filter instanceはPhysics Worldより長生きし、Adapterが所有する。

### 3.3 Lifecycle

```text
Uncreated -> ProfileValidated -> NativeWorldCreated -> StaticBuilt
-> PlayReady -> Stepping -> StopRequested -> JobsJoined
-> NativeWorldDestroyed
```

Play開始前にbody、shape、joint、pair、contact、temporary memory、event queueを予約する。`T50`中にWorld設定、body／constraint collectionを構造変更しない。全native jobがjoinした後だけ`T60`へ進む。

2Dと3Dを同一Sceneで使う場合も別Worldとし、`producer_system_id`昇順にstepする。2D／3D Worldを同時stepせず、shape同士を直接collisionさせない。

## 4. Dynamics command

Gameplay／NativeGameModuleは次のtyped commandだけを`T30`／`T40`へ提出する。

| Command | 対象／規則 |
|---|---|
| `ApplyForce` | dynamic body、finite、max magnitudeをProfileで検証 |
| `ApplyTorque` | dynamic body |
| `ApplyImpulse` | dynamic body、instantaneous |
| `SetLinearVelocity` | 許可Capabilityだけ。hard speed内 |
| `SetAngularVelocity` | 許可Capabilityだけ。hard speed内 |
| `SetKinematicTarget` | kinematic body、`T40` |
| `TeleportBody` | Collision規約のevent意味 |
| `WakeBody`／`AllowSleep` | dynamic body |
| `SetGravityFactor` | 3D dynamic、Cooked field範囲内 |
| `SetJointMotorTarget` | motor対応Joint／Constraint |
| `SetJointLimit` | schema範囲内、Play policyが許可時 |
| `BreakJoint` | 次`T00`の構造変更 |

CommandはBody／Joint handle、expected generation、producer、sequence、consume tickを持つ。ForceとImpulseの単位を混同せず、2D torqueはN·m、3D torque vectorもN·mとする。NaN、Inf、hard speed超過、static対象、stale handleをclampせずrejectする。

同一Bodyへのcommandは`command priority, producer_system_id, sequence`でcanonical mergeする。Teleport、kinematic target、velocity、forceを暗黙加算せず、競合表をMCDで固定する。Teleportとkinematic targetの同tick競合はTeleportだけを採用し、warningではなくtyped conflictとして低priority commandを拒否する。

## 5. Joint／Constraint

### 5.1 共通Component

| Field | 規則 |
|---|---|
| `joint_id` | StableId |
| `body_a`／`body_b` | Body Component StableId。World anchorは明示enum |
| `enabled` | bool |
| `collide_connected` | bool。Collision pair overrideへ変換 |
| `local_frame_a/b` | 2D poseまたは3D transform、finite |
| `limit` | Type別typed object |
| `motor` | Type別typed object |
| `break_force_n` | optional positive finite |
| `break_torque_nm` | optional positive finite |
| `solver_priority` | 0～7。一般Authoringでは既定0 |

BodyをdestroyするChangeSetは接続Jointを同じtransactionでdeleteするか、precondition failureになる。Jolt bodyがConstraintを所有しないというvendor挙動をProject modelへ漏らさず、Engine `JointRegistry`が接続を追跡する。

### 5.2 C1 Type

| 2D | 3D |
|---|---|
| distance、revolute、prismatic、weld | fixed、point、distance、hinge、slider、swing-twist |

Joint Typeごとにanchor、axis、limit、motor force／torque、frequency／damping等の必要fieldをMCDで別structにする。存在しないfieldを汎用property bagへ入れない。AxisはCook時にnormalizeし、zero vectorを既定軸へ置換しない。

Joint breakはnative callbackからGame処理を呼ばず、candidateをcopyする。`T60`でStableId、force／torque、threshold、tickをnormalizeし、`T70`でGameplay eventを配送、実際のComponent削除は次`T00`とする。

## 6. Physics stepと結果

| Phase | 処理 |
|---|---|
| `T40` | Character Motor、root motion、kinematic target、command validation |
| `T50` | Box2D／Jolt step。Adapter以外World write禁止 |
| `T60` | transform、sleep、velocity、contact／trigger／joint eventをcopy／normalize |
| `T70` | Gameplayへtyped event配送、次tick command生成 |

Dynamic BodyのTransform writerは`T60`だけである。Animation／Gameplayが同じTransformへwriteしない。Physics resultのcanonical順、event上限、capacity、handle、hard spatial rangeはCollision／Runtime規約を優先する。

同一Build／Target／worker設定ではReplay state hashとevent順を一致させる。異なるCPU architecture／Platformのbitwise lockstepを保証しない。

## 7. Physics memory、failureとtest

Windows Physics Domain 96 MiBのうち、Runtime message queue予約12 MiBを先に差し引き、native World、Body、Joint、pair、contact、event staging、temporary memoryの合計を84 MiB以下とする。3D `temp_allocator_bytes=32 MiB`はこの84 MiBに含み、残る52 MiBをJolt persistent allocation、Box2D World、Engine mappingへ使用する。2Dと3Dを同時利用してもDomain capを二倍にしない。Collision Profileのcapacity予測が84 MiBを超える場合はCook／Play prepareを拒否し、Runtimeで一般heapやUnassigned headroomへfallbackしない。

| Failure | 結果 |
|---|---|
| Temp allocator枯渇 | heap fallbackせずtick非publish、Play fault |
| body／pair／contact上限 | tick非publish、Play fault |
| native assert／non-finite | Worldを継続せずPlay fault |
| Joint invalid body／axis | Play prepareまたはCommand reject |
| Event overflow | tick非publish |
| Callback中の再入 | Development hard assertion、Release fault |

TestはJointごとのlimit／motor／break、sleep／wake、continuous motion、stack、high-speed、Character、capacity、invalid ID、multi-thread callback、10分soakを2D／3D Reference Profileで実行する。Physics P95 2.50 msを維持する。

# Part II: Navigation

## 8. 正規Navigation model

| Object | 役割 |
|---|---|
| `NavAgentProfile` | radius、height、climb、slope、query／avoidance設定 |
| `NavSourceSet` | Static geometry、Collision source revision、area marker |
| `NavAreaProfile` | area ID、Q16.16 cost、flags、semantic tag |
| `NavModifier` | blocked／area override、bounded shape |
| `NavOffMeshLink` | start／end、direction、radius、typed traversal action |
| `NavBuildProfile` | grid／voxel、tile、region、simplification、hard cap |
| `CookedNavWorld` | Target／Agent別immutable Derived Asset |
| `NavObstacleSnapshot` | 前tick Physics transformから作るdynamic obstacle values |
| `NavQueryRequest／Result` | async typed message |

Navmesh polygon ref、`dtPolyRef`、Recast／Detour object pointer、worker query objectをWorld、Save、AIへ保存しない。

## 9. 2D Navigation

### 9.1 Grid C1

`GridNav2DProfile::ReferenceV1`は2D／3D機能計画の値を正本とし、次を実行契約に固定する。

- 0.25 m cell、8近傍、corner cutting禁止
- `uint8 area_id`＋`uint8 clearance_cells`の2 byte cell
- Q16.16 cost、octile heuristic
- tie-breakは`f, h, canonical cell index`昇順
- query node 65,536、path cell 8,192、Asset 16,777,216 cell／34 MiB
- Agent radiusから`ceil(radius / cell_size)` clearance

A* open set、visited、parentはquery-local bounded memoryである。Budget超過を`NoPath`へ偽装せず`SearchBudgetExceeded`を返す。Start／Goal cellがblockedまたはclearance不足なら`InvalidEndpoint`とする。

### 9.2 Polygon C2

2D polygon navigation、hierarchical graph、flow field、local avoidanceはC2で別Capability IDを持つ。Grid C1 save／requestを同じbinaryへreinterpretしない。

## 10. 3D Navigation

### 10.1 Build

Recast 1.6.0をoffline builderとして使い、次のstageをArtifact Receiptへ記録する。

```text
Source collect
-> Rasterize heightfield
-> Walkable filter
-> Compact heightfield
-> Erode by agent radius
-> Distance／region partition
-> Contour
-> Polygon mesh
-> Detail mesh
-> Detour tile data
-> Validation query
```

`NavAgentProfile::HumanReferenceV1`、Recast `rcConfig`値、tile／polygon／memory／query上限は2D／3D機能計画6.7節を正本とする。Source geometry、static transform、Collision source revision、Agent／Build Profile、builder version、Toolchain hashからArtifactKeyを作る。

Build中のpartial tileをlive NavWorldへpublishしない。影響tileと境界隣接tileのclosureをStagingし、seam、connectivity、memory、reference queryを合格後、`T00`で`CookedNavWorld` generationを一度に交換する。

### 10.2 Runtime query

```text
NavQueryRequest
  request_id
  nav_world_version
  agent_profile_id
  start/end
  projection_half_extents
  area_mask/cost_table
  allow_partial
  max_nodes/max_corridor/max_points
  deadline_tick
  owner_generation
```

Resultは`Success | NoPath | PartialPath | SearchBudgetExceeded | StaleNavMesh | InvalidEndpoint | QueueFull | Cancelled`のclosed status、projected endpoints、corridor Stable representation、straight path point、cost、visited countを持つ。

Detour query objectとnode poolはworker／jobごとに分離する。同一`dtNavMeshQuery`をparallel jobで共有しない。ResultはRuntime規約どおり最短でも次tickの`T20`でrequest ID順に統合し、version／owner／deadlineを再検査する。

### 10.3 Dynamic obstacleとlink

- C1 Runtimeは既存Nav上のcost、goal、typed off-mesh link enabled stateだけを変更できる。
- C2 TileCache obstacleは前tick`T60`後の`NavObstacleSnapshot`を次tick`T20`で取込む。
- 同tickPhysics resultをNavへ直接参照しない。
- Door、jump、climb linkは任意callback名でなく`TraversalActionId`を持つ。
- Traversal開始前にlink generation、entry／exit clearance、Capabilityを再検査する。
- AIはpolygon、tile binary、Detour refを生成しない。

### 10.4 Crowd／avoidance

DetourCrowdとlocal avoidanceはC2であり、authoritative path queryと分離する。Crowd outputはdesired velocity proposalで、Character Motor／Physicsが最終transformを確定する。CrowdがWorld Transformへwriteしない。

## 11. Navigation memory、failure、test

Navigation Domain 64 MiBの内訳、request／result各4096、live 2D／3D payload合計36 MiBはRuntime規約を正本とする。

| Failure | 結果 |
|---|---|
| Source／Build invalid | Artifact publishなし、旧Nav維持 |
| Query queue full | 新規requestを`QueueFull`で拒否 |
| Stale／deadline | Result破棄、World変更なし |
| Node／corridor／point上限 | partial成功にせずtyped status |
| Tile promotion memory不足 | promotion延期、旧generation維持 |
| Required pathなし | Level validation error |

Testは2D corner、cost、clearance、deterministic tie-break、3D seam、slope、step、off-mesh link、partial policy、dynamic obstacle遅延、stale result、tile swap、spawn→goal reachability、memory、query latencyを含む。

# Part III: Animation

## 12. 正規Animation model

| Object | 役割 |
|---|---|
| `SkeletonSourceAsset` | joint Stable path、parent、bind pose、semantic bone |
| `AnimationClipSourceAsset` | typed track、event、root track、compression setting |
| `CookedSkeleton／Clip` | Target非依存のimmutable ozz-backed payload |
| `AnimationGraphDefinition` | Engine-owned typed state／blend graph |
| `AnimationControllerComponent` | Graph、parameter set、root motion mode |
| `AnimationInstanceState` | current state、transition、clock、loop、event cursor |
| `AnimationParameterSnapshot` | T30でlatchしたtyped parameter |
| `RootMotionProposal` | T40へ渡すdeltaとinterval |
| `SkeletonPose` | T80で確定したlocal／model poseとbounds |
| `AnimationEvent` | typed Event ID、normalized time、source state |

Clip名、joint名、function名をRuntime dispatchに使わない。Authoring名はCook時にStable numeric ID／semantic tagへ解決する。

## 13. 2D Animation

C1 Graph nodeを次で固定する。

- Flipbook sample
- Property curve
- Tween
- State
- Transition
- 1D parameter blend
- Layer／mask
- Typed event track

Sprite frameはTexture Asset version＋sprite roleを参照し、atlas indexをSourceへ保存しない。Onion skin、scrub、curve editorはAuthoring projectionであり、Runtime stateへ混入させない。

Cutout skeleton、2D IK、2D retarget、2D blend spaceはC2である。

## 14. 3D Animation

### 14.1 ozz境界

ozz-animation 0.16.0のoffline builder、compression、runtime Skeleton／Animation、SamplingJob、BlendingJob、LocalToModelJobをAdapter内で利用する。

- Raw／offline dataはAsset Workerだけで使用する。
- Runtime Skeleton／Animationはimmutable Asset versionとして共有する。
- `SamplingJob::Context`はAnimation instanceごとに所有しparallel共有しない。
- local pose、blend scratch、model matricesはjobごとのbounded memoryを使う。
- ozz archiveをProject source／Save formatにしない。
- ozzはhigh-level blend treeを提供しないため、Graph、state、transition、event、root motion、IK policyはEngineが所有する。

### 14.2 C1 Graph node

| Node | 規則 |
|---|---|
| Clip | Clip StableId、speed、loop、start policy |
| State Machine | Stable State ID、entry、一意active state |
| Transition | typed condition、priority、duration、interrupt policy |
| Blend1D | sorted threshold、最大8 sample |
| CrossFade | source／target pose、normalized curve |
| Layer | 最大8、mask、weight |
| Additive | C2までCook不可 |
| Event Track | registered Event ID |
| Root Motion Extract | Controller policyに従う |
| Output | Graphに一つ |

Graphはcycleを原則rejectし、State Machineの遷移cycleは有限stateとして許可する。Pose nodeの再帰、任意loop、function callback、Script expressionを禁止する。Conditionはtyped parameter／event／state timeのbounded expressionである。

## 15. Phaseとownership

| Phase | Animation処理 |
|---|---|
| `T30` | Gameplay parameter／state requestをlatchし、State／Transitionを一度評価 |
| `T40` | clock interval`[begin,end]`を確定し、root deltaだけsampleしてMotorへproposal |
| `T60` | resolved Character／Ragdoll inputを受ける |
| `T80` | 同じintervalからclip sample、blend、IK、local-to-model、bounds、eventを確定 |
| `T90` | presentation event、VFX／Audio requestを生成 |
| `T110` | pose／skin packetをRenderSnapshotへcopy |

ClockをT40とT80で二度進めない。Root motion modeを次のclosed enumとする。

- `animation`: Animation deltaをCharacter Motorがcollision解決する。
- `gameplay_motor`: Animation root deltaは0、resolved velocityからin-place poseを選ぶ。
- `disabled`: root trackをpresentationにも使わない。

Mode変更は`T00`だけで適用する。Animation deltaとGameplay deltaを加算しない。

RagdollはPhysics `T60`の`RagdollPoseInput`をT80でweight blendする。Physicsが`SkeletonPose`へ直接writeしない。

## 16. Event、IK、Skinning

- Animation EventはClipの`event_track_id`とregistered typed Event IDを持つ。
- frameを飛び越えても`(previous_time, current_time]`内eventを順番どおり検出する。
- loop跨ぎ、reverse、seek policyをClipに明示し、Editor scrubでGameplay eventを発火しない。
- Event orderはevent time、track ID、event IDでcanonicalizeする。
- Foot IK、look-atはC2。Collision Query Snapshotを読み、Physics native pointerを参照しない。
- CPU skinningはC1 fallback、GPU skinningはRendererのregistered Pass Templateで行う。
- Skeleton paletteはAsset／pose generationを持ち、旧Skeletonへ新Poseを適用しない。

## 17. Animation memory、failure、test

Animation Domain 64 MiBをRuntime規約どおり守り、Skeleton／Clip payload、instance state、sampling context、pose／scratchを別counterへchargeする。

| Failure | 結果 |
|---|---|
| Skeleton／Clip mismatch | Asset generation promotion拒否 |
| Graph cycle／missing state／invalid condition | Cook reject |
| Runtime invalid handle | instance fault、authoritative root proposal 0、診断 |
| Pose scratch不足 | heap fallbackせずPlay fault |
| Missing optional presentation clip | 明示fallback Clipがある場合だけ使用 |
| Required locomotion clip不足 | Play prepare失敗 |
| Event overflow | presentationはpriority policy、authoritative eventはtick fault |

Testはforward／reverse／loop／seek、crossfade、layer／mask、transition interruption、root motion単一適用、Character collision、event飛越し、ragdoll blend、Skeleton hot reload、stale Asset、parallel instance、10分soakを含む。Motion＋Animation P95 1.50 msを維持する。

## 18. AI／Editor操作

AIと人間は次のtyped object／Operationだけを編集する。

- Physics World Profileの公開field、Body／Joint、Character Profile
- Nav Agent／Area／Build Profile、Modifier、Off-mesh Link、reachability test
- Skeleton／Clip import、Animation Graph、parameter、transition、event、root motion mode

AIはvendor setting blob、native ID、solver callback、nav polygon ref、ozz archive byte、arbitrary expressionを生成しない。Profile hard値変更、high-cost nav rebuild、Joint bulk生成、Graph全置換はRiskと予測costを表示する。

EditorはPhysics／Joint、Nav voxel／tile／path、Animation state／blend／root／eventを可視化し、diagnosticからSource fieldへ移動できる。

## 19. Cross-Subsystem Release Gate

1. Root motion proposal→Character Motor→Physics resolved transform→Animation poseが一回だけ適用される。
2. Static Collision Source revision→Nav Derived Asset→query generationが追跡できる。
3. Physics contact／joint break→Gameplay→Animation／Audio／VFXがcallback再入なしで配送される。
4. RagdollはPhysics inputとAnimation final poseのwriterを分離する。
5. Asset hot reloadはPhysics、Nav、Animation dependency closureを部分混在させない。
6. 2D／3D World併用、worker oversubscription、stale async result、queue overflowを合格する。
7. 全vendor型がpublic header、MCD、Save、Project C++から検出されない。
8. Target別capacity、memory、P95、10分soakを満たす。

## 20. 一次資料

- [Box2D 3.1.1 Documentation](https://box2d.org/documentation/)
- [Box2D Simulation and multithreading](https://box2d.org/documentation/md_simulation.html)
- [Jolt Physics 5.6 Documentation](https://jrouwe.github.io/JoltPhysicsDocs/5.6.0/)
- [Jolt Physics Architecture](https://github.com/jrouwe/JoltPhysics/blob/v5.6.0/Docs/Architecture.md)
- [Recast Navigation 1.6.0](https://github.com/recastnavigation/recastnavigation/releases/tag/v1.6.0)
- [Recast Introduction and build process](https://github.com/recastnavigation/recastnavigation/blob/v1.6.0/Docs/_1_Introduction.md)
- [ozz-animation Overview](https://guillaumeblanc.github.io/ozz-animation/documentation/)
- [ozz-animation Runtime](https://guillaumeblanc.github.io/ozz-animation/documentation/animation_runtime/)
- [ozz-animation Offline Libraries](https://guillaumeblanc.github.io/ozz-animation/documentation/animation_offline/)

External Libraryはsolver、navmesh、sampling等の検証済みkernelとして使用する。Miraikanaiの公開data、phase、event、budget、failure、Editor、AI operationは本書とCollision／Runtime規約が所有する。
