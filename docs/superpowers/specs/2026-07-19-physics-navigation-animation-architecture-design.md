# Miraikanai Engine Physics／Navigation／Animation連携規約

- 文書版: 1.2
- 作成日: 2026-07-19
- 最終更新日: 2026-07-20
- 対象: PhysicsとNavigation／Animationの連携、Navigation build／query、2D／3D Animation graph／pose
- 状態: プロジェクト公式の規範設計レビュー版
- Physics正本: [Miraikanai Engine 独自Physics Platform／Dynamicsアーキテクチャ規約](./2026-07-20-physics-engine-architecture-design.md)
- Collision正本: [Miraikanai Engine Collision／Colliderアーキテクチャ規約](./2026-07-19-collision-collider-architecture-design.md)
- Runtime正本: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- 機能範囲: [Miraikanai Engine 2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)
- 基盤規約: [Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)
- C++公開境界: [Miraikanai Engine C++23・Named Modules・`import std`移行規約](./2026-07-20-cpp23-modules-import-std-transition-design.md)
- Asset規約: [Miraikanai Engine Asset Pipeline／Content Package規約](./2026-07-19-asset-pipeline-content-packaging-design.md)

## 1. 結論

Physics、Navigation、Animationは相互にpointerを渡す一体型Subsystemにしない。Runtime Orchestratorの固定phase、typed command／event、immutable snapshot、versioned Derived Assetだけで連携する。

- PhysicsのWorld、Dynamics、Joint／Constraint、Character Motor、Backend、Save／Replay、AI／Editorは独自Physics Platform規約を正本とする。
- Collision shape、filter、query、contactはCollision規約を正本とする。
- Navigation mesh／gridは再生成可能なDerived Assetであり、Worldの正本ではない。
- Animation GraphはMiraikanai独自のtyped graphであり、ozz-animationはoffline／sampling／blend primitiveとしてAdapter内で使う。
- Box2D、Jolt、Recast／Detour、ozzのID、pointer、callback、serializationをProject、AI、Saveへ公開しない。

## 2. 決定権と成熟度

| 主題 | 正本 |
|---|---|
| Shape、Collider、Body field、Filter、Query、Contact | Collision規約 |
| Tick、writer、event順、queue、handle、budget、promotion | Runtime規約 |
| Physics World、Solver、Dynamics command、Joint／Constraint、Character Motor、Backend、Save／Replay | 独自Physics Platform規約 |
| PhysicsとNavigation／Animationのtyped連携 | 本書Physics連携節 |
| 2D grid／3D navmesh、query、avoidance、link | 本書Navigation節 |
| Clip、Graph、state、root motion、pose、event、IK境界 | 本書Animation節 |
| C1／C2 feature listとReference Scene | 2D／3D機能計画 |

C1は2D sliceでBox2D、Grid Nav、Flipbook、3D sliceでJolt、Recast／Detour、ozzをProduction化する。Vehicle、ragdoll、advanced crowd、retarget、motion warpingはC2。Soft body、fluid、GPU Physics、network lockstep、runtime arbitrary navmesh generation、ML motion synthesisはC3である。

# Part I: Physics連携

## 3. Physicsの公開境界

Physics World、Solver Profile、Dynamics Command、Joint／Constraint、Character Motor、Backend build、Save／Replay、memory、failure、Qualificationの詳細は独自Physics Platform規約を唯一の正本とする。本書は次のSubsystem間連携だけを決める。

| Producer | 出力 | Consumer／時点 | 禁止 |
|---|---|---|---|
| Gameplay／NativeGameModule | bounded `PhysicsDynamicsCommandV1`／`CharacterMoveIntentV1` | Physics T30／T40 | Native World call、callback登録 |
| Animation | `RootMotionProposal` | Character Motor T40 | Transform直接write |
| Physics | `PhysicsStateSnapshotV1`、normalized Event | Gameplay T70、Animation T80、Replay T100 | Native ID／pointer |
| Physics | 前tick完了済み`NavObstacleSnapshot`の入力値 | Navigation 次tick T20 | 同tickWorld pointer共有 |
| Navigation／Crowd | desired velocity proposal | Character Motor 次のT40 | authoritative Transform write |
| Rendering／Editor | immutable `PhysicsDebugSnapshotV1` | T60後 | Physics World query |

## 4. 連携時の固定規則

1. 2D／3D Physicsは別Worldであり、相互collisionを行わない。
2. Physicsのdynamic Transform writerはT60だけである。
3. Root motionとGameplay移動はCharacter Profileのclosed policyで一度だけ合成する。
4. Navigationは前tickPhysics snapshotを使い、Physics callback、Body lock、shape pointerを保持しない。
5. AnimationはPhysics resolved poseを読む。Ragdoll C2もtyped pose／constraint commandを介し、SkeletonとNative Bodyを相互所有させない。
6. VFX、Audio、Cameraはnormalized Event／version付きQuery Resultを読み、Physicsのauthoritative結果へwrite backしない。
7. Failed Physics tickはTransform、Event、Nav obstacle、Animation inputを部分publishしない。

# Part II: Navigation

## 5. 正規Navigation model

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

## 6. 2D Navigation

### 6.1 Grid C1

`GridNav2DProfile::ReferenceV1`は2D／3D機能計画の値を正本とし、次を実行契約に固定する。

- 0.25 m cell、8近傍、corner cutting禁止
- `uint8 area_id`＋`uint8 clearance_cells`の2 byte cell
- Q16.16 cost、octile heuristic
- tie-breakは`f, h, canonical cell index`昇順
- query node 65,536、path cell 8,192、Asset 16,777,216 cell／34 MiB
- Agent radiusから`ceil(radius / cell_size)` clearance

A* open set、visited、parentはquery-local bounded memoryである。Budget超過を`NoPath`へ偽装せず`SearchBudgetExceeded`を返す。Start／Goal cellがblockedまたはclearance不足なら`InvalidEndpoint`とする。

### 6.2 Polygon C2

2D polygon navigation、hierarchical graph、flow field、local avoidanceはC2で別Capability IDを持つ。Grid C1 save／requestを同じbinaryへreinterpretしない。

## 7. 3D Navigation

### 7.1 Build

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

### 7.2 Runtime query

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

### 7.3 Dynamic obstacleとlink

- C1 Runtimeは既存Nav上のcost、goal、typed off-mesh link enabled stateだけを変更できる。
- C2 TileCache obstacleは前tick`T60`後の`NavObstacleSnapshot`を次tick`T20`で取込む。
- 同tickPhysics resultをNavへ直接参照しない。
- Door、jump、climb linkは任意callback名でなく`TraversalActionId`を持つ。
- Traversal開始前にlink generation、entry／exit clearance、Capabilityを再検査する。
- AIはpolygon、tile binary、Detour refを生成しない。

### 7.4 Crowd／avoidance

DetourCrowdとlocal avoidanceはC2であり、authoritative path queryと分離する。Crowd outputはdesired velocity proposalで、Character Motor／Physicsが最終transformを確定する。CrowdがWorld Transformへwriteしない。

## 8. Navigation memory、failure、test

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

## 9. 正規Animation model

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

## 10. 2D Animation

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

## 11. 3D Animation

### 11.1 ozz境界

ozz-animation 0.16.0のoffline builder、compression、runtime Skeleton／Animation、SamplingJob、BlendingJob、LocalToModelJobをAdapter内で利用する。

- Raw／offline dataはAsset Workerだけで使用する。
- Runtime Skeleton／Animationはimmutable Asset versionとして共有する。
- `SamplingJob::Context`はAnimation instanceごとに所有しparallel共有しない。
- local pose、blend scratch、model matricesはjobごとのbounded memoryを使う。
- ozz archiveをProject source／Save formatにしない。
- ozzはhigh-level blend treeを提供しないため、Graph、state、transition、event、root motion、IK policyはEngineが所有する。

### 11.2 C1 Graph node

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

## 12. Phaseとownership

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

## 13. Event、IK、Skinning

- Animation EventはClipの`event_track_id`とregistered typed Event IDを持つ。
- frameを飛び越えても`(previous_time, current_time]`内eventを順番どおり検出する。
- loop跨ぎ、reverse、seek policyをClipに明示し、Editor scrubでGameplay eventを発火しない。
- Event orderはevent time、track ID、event IDでcanonicalizeする。
- Foot IK、look-atはC2。Collision Query Snapshotを読み、Physics native pointerを参照しない。
- CPU skinningはC1 fallback、GPU skinningはRendererのregistered Pass Templateで行う。
- Skeleton paletteはAsset／pose generationを持ち、旧Skeletonへ新Poseを適用しない。

## 14. Animation memory、failure、test

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

## 15. AI／Editor操作

AIと人間は次のtyped object／Operationだけを編集する。

- Physics World Profileの公開field、Body／Joint、Character Profile
- Nav Agent／Area／Build Profile、Modifier、Off-mesh Link、reachability test
- Skeleton／Clip import、Animation Graph、parameter、transition、event、root motion mode

AIはvendor setting blob、native ID、solver callback、nav polygon ref、ozz archive byte、arbitrary expressionを生成しない。Profile hard値変更、high-cost nav rebuild、Joint bulk生成、Graph全置換はRiskと予測costを表示する。

EditorはPhysics／Joint、Nav voxel／tile／path、Animation state／blend／root／eventを可視化し、diagnosticからSource fieldへ移動できる。

## 16. Cross-Subsystem Release Gate

1. Root motion proposal→Character Motor→Physics resolved transform→Animation poseが一回だけ適用される。
2. Static Collision Source revision→Nav Derived Asset→query generationが追跡できる。
3. Physics contact／joint break→Gameplay→Animation／Audio／VFXがcallback再入なしで配送される。
4. RagdollはPhysics inputとAnimation final poseのwriterを分離する。
5. Asset hot reloadはPhysics、Nav、Animation dependency closureを部分混在させない。
6. 2D／3D World併用、worker oversubscription、stale async result、queue overflowを合格する。
7. 全vendor型がCX0 Public Header、CX3 Module interface、MCD、Save、Project C++から検出されない。
8. Target別capacity、memory、P95、10分soakを満たす。

## 17. 一次資料

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
