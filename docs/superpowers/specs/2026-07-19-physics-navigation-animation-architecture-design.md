# Miraikanai Engine Physics／Navigation／Animation連携規約

- 文書版: 1.6
- 作成日: 2026-07-19
- 最終更新日: 2026-07-20
- 対象: Physics、Navigation、Animation間の固定phase連携、2D／3D Animation graph／pose
- 状態: プロジェクト公式の規範設計レビュー版
- Physics正本: [Miraikanai Engine 独自Physics Platform／Dynamicsアーキテクチャ規約](./2026-07-20-physics-engine-architecture-design.md)
- Navigation正本: [Miraikanai Engine 独自Navigation Platformアーキテクチャ規約](./2026-07-20-navigation-platform-architecture-design.md)
- Collision正本: [Miraikanai Engine Collision／Colliderアーキテクチャ規約](./2026-07-19-collision-collider-architecture-design.md)
- Runtime正本: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- 機能範囲: [Miraikanai Engine 2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)
- 基盤規約: [Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)
- C++公開境界: [Miraikanai Engine C++23・Named Modules・`import std`移行規約](./2026-07-20-cpp23-modules-import-std-transition-design.md)
- Asset規約: [Miraikanai Engine Asset Pipeline／Content Package規約](./2026-07-19-asset-pipeline-content-packaging-design.md)
- Asset Import／AI／Editor規約: [Miraikanai Engine Asset Import／AI Authoring／Editor UXアーキテクチャ規約](./2026-07-20-asset-import-ai-authoring-editor-ux-design.md)
- LOD正本: [Miraikanai Engine AI可読LODアーキテクチャ規約](./2026-07-20-ai-readable-lod-architecture-design.md)

## 1. 結論

Physics、Navigation、Animationは相互にpointerを渡す一体型Subsystemにしない。Runtime Orchestratorの固定phase、typed command／event、immutable snapshot、versioned Derived Assetだけで連携する。

- PhysicsのWorld、Dynamics、Joint／Constraint、Character Motor、Backend、Save／Replay、AI／Editorは独自Physics Platform規約を正本とする。
- Collision shape、filter、query、contactはCollision規約を正本とする。
- NavigationのProfile、2D Grid、3D Navmesh、Backend、Artifact、query、AI／Editorは独自Navigation Platform規約を正本とする。本書はPhysics／Animationとのphase連携だけを所有する。
- Animation GraphはMiraikanai独自のtyped graphであり、ozz-animationはoffline／sampling／blend primitiveとしてAdapter内で使う。
- Box2D、Jolt、Recast／Detour、ozzのID、pointer、callback、serializationをProject、AI、Saveへ公開しない。

## 2. 決定権と成熟度

| 主題 | 正本 |
|---|---|
| Shape、Collider、Body field、Filter、Query、Contact | Collision規約 |
| Tick、writer、event順、queue、handle、budget、promotion | Runtime規約 |
| Physics World、Solver、Dynamics command、Joint／Constraint、Character Motor、Backend、Save／Replay | 独自Physics Platform規約 |
| PhysicsとNavigation／Animationのtyped連携 | 本書Physics連携節 |
| 2D grid／3D navmesh、Backend、Artifact、query、avoidance、link、AI／Editor | 独自Navigation Platform規約 |
| Clip、Graph、state、root motion、pose、event、IK境界 | 本書Animation節 |
| Animation presentation LODの共通Intent、選択、fallback、Receipt | LOD正本 |
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

# Part II: Navigation連携

## 5. Navigation公開境界

Navigationの正規object、2D Grid、3D Build、Backend lock、Artifact envelope、query status、capacity、failure、AI／Editor、Qualificationは独自Navigation Platform規約を唯一の正本とする。本書は次のSubsystem間連携だけを決める。

| Producer | 出力 | Consumer／時点 | 禁止 |
|---|---|---|---|
| Static Collision／Asset | `NavSourceSetV1`が参照するsource revision | offline Navigation cook | native shape／mesh pointer共有 |
| Physics | 前tick完了済み`NavObstacleSnapshot`の入力値 | Navigation 次tick`T20` | 同tickWorld／Body pointer共有 |
| Gameplay／NativeGameModule | bounded `NavQueryRequestV1` | Navigation worker | Detour API／polygon ref直接利用 |
| Navigation | version付き`NavQueryResultV1` | 次tick以降のGameplay `T20` | World Transform直接write |
| Navigation／Crowd C2 | desired velocity proposal | Character Motor 次の`T40` | authoritative Transform write |
| Navigation | immutable `NavigationDebugSnapshotV1` | Editor／Rendering | live Backend query |

## 6. 固定phaseとversion

1. Static sourceとProfileから作るGrid／Navmeshは再生成可能なDerived Assetであり、Worldの正本ではない。
2. Build／rebuildはstaging generationで完了させ、影響tile closureのseam、connectivity、memory、reference query合格後だけ`T00`で一括交換する。
3. Runtime requestはdispatch時の`NavMeshVersion`を持ち、workerはそのversionのread leaseだけを使用する。
4. Resultは最短でも次tickの`T20`でrequest ID順に統合し、active version、owner generation、deadlineを再検査する。
5. version不一致、owner失効、deadline超過resultはWorldへ配送しない。
6. C2 dynamic obstacleは前tick`T60`後のPhysics snapshotを次tick`T20`で取込み、同tickPhysics結果を参照しない。
7. Navigation resultとCrowd proposalはCharacter Motorの入力であり、NavigationがTransform writerにならない。

## 7. Cross-Subsystem failure

| Failure | 連携側の処理 |
|---|---|
| Source／Build／Artifact invalid | partial publishなし、旧Nav維持 |
| Query queue full | 新規requestだけを`QueueFull`で拒否 |
| Node／corridor／point上限 | `SearchBudgetExceeded`、Transform変更なし |
| Stale／owner／deadline | T20で破棄、Gameplayへ配送なし |
| Tile promotion memory不足 | promotion延期、旧generation維持 |
| Required pathなし | Level validation error、C1 Scene昇格不可 |

Navigationのclosed statusと優先順位は独自Navigation Platform規約11.3節を正本とし、本書で別statusへ再解釈しない。

## 8. Navigation連携test

Static Collision source revision→Nav Derived Asset→query version、前tickPhysics obstacle snapshot→次tickNavigation、Navigation proposal→Character Motor、stale result→World無変更をconformance testで固定する。2D／3D併用、worker oversubscription、queue overflow、promotion延期、required path failureをCross-Subsystem Release Gateへ含める。

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

Clip名、joint名、function名をRuntime dispatchに使わない。Cookerは一つのexact Cooked Skeleton／Clip Artifact内でcanonical joint pathとClip Source `StableId`を固定順へ並べ、1から`JointRuntimeId uint32`／`ClipRuntimeId uint32`を割り当てる。0はinvalidとし、Runtime IDは対応する`ArtifactRefV1`と組にして使い、Source、Save、別Artifact比較へ使用しない。Semantic bone／event tagはexact `McdContractRefV1`へ解決する。

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

### 11.3 Skeleton／Clip Import

Skeleton／Animation Source ImportはAsset Import／AI Authoring／Editor UX規約の`SceneImportIRV1`、`AnimationImportSettingsV1`、Conversion Report、Preview、Reimport Conflictを使用する。

- C1はglTF 2.0の同一Skeleton exact bindingだけをProduction対応し、stable joint path、parent、bind poseが一致しないClipを結合しない。
- C1 retarget policyは`none`である。Humanoid semantic mapping、reference pose、scale／twist compensation、before／after pose corpusを持つC2 Profileだけがretargetを有効化できる。
- Source hierarchy、Root Transform、unit、negative determinant、shearの処理後にskin bind equationを再検証する。Skinned Meshのnegative determinant、shear、singular bind poseをC1でbakeしない。
- Clip extraction、sample policy、bake rate、interpolation、key reduction、root motion、event metadataは型付きProfile fieldとし、file名やDCC既定値だけで確定しない。
- Animation optimizerはtranslation／rotation／scale errorをProfile閾値で測定し、圧縮前後の最大誤差とloop seamをReceiptへ記録する。
- Importer ID／major version、Hierarchy、rest pose、material／joint orderの変更は自動Reimportせず、consumer closure付きConflict Previewを必須とする。

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

### 12.1 Animation presentation LOD

`AnimationLodProfileV1`はLOD正本の`LodIntentV1`と`LodResolutionPlanV1`からT80のpresentation pose／skinning tierを選ぶ。tierはpose update interval、presentation bone set、skinning mode、shadow poseを変更できるが、T40 root motion、hitbox／weapon socket、foot contact、Gameplay Animation Event、authoritative boundsを低頻度poseから取得しない。

off-screen、occlusion、Renderer visibilityはAnimationのauthoritative clock／state transition／event cursorを停止する入力にしない。Dormancyが必要なEntityはRuntimeの`SimulationLodContractV1`で別に契約し、Animation LODから逆入力しない。tier遷移はenter／exit threshold、`minimum_residency_units`、fallback poseを必須とし、強制tierはEditor previewだけに許可する。

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

Testはforward／reverse／loop／seek、crossfade、layer／mask、transition interruption、root motion単一適用、Character collision、event飛越し、ragdoll blend、Skeleton hot reload、stale Asset、parallel instance、10分soakを含む。Animation LOD testはcamera path／visibility／Target／Qualityを変えてもroot motion、hitbox、Gameplay Event、state hashが一致し、presentation bone fallback、hysteresis、tier traceが再現可能であることを検証する。Import testはmissing／extra／reordered joint、noninvertible bind、step／linear／cubic、key reduction閾値、loop seam、root trajectory、Importer migration Conflict、Editor scrubのEvent非発火を含む。Motion＋Animation P95 1.50 msを維持する。

## 15. AI／Editor操作

AIと人間は次のtyped object／Operationだけを編集する。NavigationのOperation、質問条件、preview、Backend禁止事項は独自Navigation Platform規約を正本とする。

- Physics World Profileの公開field、Body／Joint、Character Profile
- Nav Agent／Area／Build Profile、Modifier、Off-mesh Link、reachability test
- Skeleton／Clip import、Animation Graph、parameter、transition、event、root motion mode
- `AnimationLodProfileV1`のproposal／preview／validation。共通操作IDは`operation.lod.*`
- Skeleton tree、joint axis、bind／reference pose、skin weight、root trajectory、compression error、retarget before／after Preview

AIはvendor setting blob、native ID、solver callback、nav polygon ref／tile／`rcConfig`、ozz archive byte、arbitrary expressionを生成しない。Profile hard値変更、high-cost nav rebuild、Joint bulk生成、Graph全置換はRiskと予測costを表示する。

AIはCatalogにないSkeleton、joint、Clip、retarget Profileを推測しない。Ambiguous root joint、clip range、root motion、retarget mappingは`AssetImportPlanV1.blocking_questions`として返し、未回答のままCookしない。

EditorはPhysics／Joint、Nav voxel／tile／path、Animation state／blend／root／eventを可視化し、diagnosticからSource fieldへ移動できる。

## 16. Cross-Subsystem Release Gate

1. Root motion proposal→Character Motor→Physics resolved transform→Animation poseが一回だけ適用される。
2. Static Collision Source revision→Nav Derived Asset→query generationが追跡できる。
3. Physics contact／joint break→Gameplay→Animation／Audio／VFXがcallback再入なしで配送される。
4. RagdollはPhysics inputとAnimation final poseのwriterを分離する。
5. Asset hot reloadはPhysics、Nav、Animation dependency closureを部分混在させない。
6. 2D／3D World併用、worker oversubscription、stale async result、queue overflowを合格する。
7. 全vendor型がCX0 Public Header、CX3 Module interface、MCD、Save、Project C++から検出されない。
8. Animation presentation LODを切り替えてもroot motion、hitbox、Gameplay Event、authoritative state hashが不変である。
9. Target別capacity、memory、P95、10分soakを満たす。

## 17. 一次資料

- [Box2D 3.1.1 Documentation](https://box2d.org/documentation/)
- [Box2D Simulation and multithreading](https://box2d.org/documentation/md_simulation.html)
- [Jolt Physics 5.6 Documentation](https://jrouwe.github.io/JoltPhysicsDocs/5.6.0/)
- [Jolt Physics Architecture](https://github.com/jrouwe/JoltPhysics/blob/v5.6.0/Docs/Architecture.md)
- [Recast Navigation 1.6.0](https://github.com/recastnavigation/recastnavigation/releases/tag/v1.6.0)
- [Recast 1.6.0 API reference](https://github.com/recastnavigation/recastnavigation/blob/v1.6.0/Docs/Extern/Recast_api.txt)
- [Recast 1.6.0 tiled build sample](https://github.com/recastnavigation/recastnavigation/blob/v1.6.0/RecastDemo/Source/Sample_TileMesh.cpp)
- [ozz-animation Overview](https://guillaumeblanc.github.io/ozz-animation/documentation/)
- [ozz-animation Runtime](https://guillaumeblanc.github.io/ozz-animation/documentation/animation_runtime/)
- [ozz-animation Offline Libraries](https://guillaumeblanc.github.io/ozz-animation/documentation/animation_offline/)

External Libraryはsolver、navmesh、sampling等の検証済みkernelとして使用する。Physicsの製品契約は独自Physics Platform規約、Navigationは独自Navigation Platform規約、AnimationとSubsystem間phaseは本書、Collision／Runtimeの共通規則は各正本が所有する。
