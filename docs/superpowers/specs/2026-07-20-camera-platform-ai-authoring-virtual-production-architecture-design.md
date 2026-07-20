# Miraikanai Engine Camera Platform／AI Authoring／Virtual Productionアーキテクチャ規約

- 文書版: 1.0
- 作成日: 2026-07-20
- 調査基準日: 2026-07-20
- 対象: 2D／3D Gameplay Camera、Camera Rig、Director、Cinematic、Split View、Multi-camera Recording、Virtual Production、AI／Editor Authoring
- 状態: プロジェクト公式の規範設計レビュー版
- 上位文書: [Miraikanai Engine 2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)
- 共通Runtime: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- Rendering: [Miraikanai Engine Rendering／Render Graphアーキテクチャ規約](./2026-07-19-rendering-render-graph-architecture-design.md)
- Authoring: [Miraikanai Engine Authoring Model／Project State規約](./2026-07-19-authoring-model-project-state-design.md)
- Contract: [Miraikanai Engine 実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)
- Physics: [Miraikanai Engine 独自Physics Platform／Dynamicsアーキテクチャ規約](./2026-07-20-physics-engine-architecture-design.md)
- Input: [Miraikanai Engine Input／Action／Device規約](./2026-07-19-input-action-device-architecture-design.md)
- Verification: [Miraikanai Engine AI検証・評価・来歴規約](./2026-07-19-ai-verification-evaluation-provenance-design.md)

## 1. 結論

Miraikanai EngineはCameraをRendererのTransform、Gameplayの任意C++ Script、Cinematic Timeline、撮影Device Pluginへ分散させない。Projection／Lens、Gameplay Rig、Director／Transition、Presentation、Sequence、Recording、Live Deviceを、同じEngine-owned型とStable IDで接続する独自`Camera Platform`として所有する。

Camera Platformは次の原則に固定する。

1. `CameraProfileDocumentV1`、`CameraRigDocumentV1`、`CameraDirectorDocumentV1`、`CameraPresentationProfileDocumentV1`、`CameraSequenceDocumentV1`、`CameraRecordingProfileDocumentV1`、`CameraDeviceProfileDocumentV1`を正規Sourceとする。
2. AI、人間、Editor、外部Agentは同じMCD生成OperationとChangeSetを使う。GPU View、Physics native query、Device SDK、Clock providerへ直接触れない。
3. C1 Gameplay Cameraは60 Hz fixed tickで決定論的なBase Poseを作る。Render interpolation、temporal jitter、shake、noiseはPresentationへ分離する。
4. C2 Cinematicは同じRig、Lens、Director、TransitionをSequenceから駆動し、Gameplayへ影響する評価と映像専用評価をClock domainで分離する。
5. C3 Multi-camera RecordingはBase Pose、Lens、Focus、Presentation channel、Timecode、Calibration、Gapを非破壊Takeとして記録する。
6. C3 Virtual Productionは外部Device、Network、Timecode、Genlock、Calibrationを別Process Adapterへ隔離し、Editor Preview／Recordingへだけ接続する。
7. Unity、Unreal Engine、Godot、OpenTimelineIO、SMPTEの公式資料は機能coverage、分離原則、Interchange、Clock表現のEvidenceに使うが、Vendor API、Asset format、Blueprint／MonoBehaviour／Node型をMiraikanaiの公開契約として模倣しない。

本書はC1からC3までの設計を詳述するが、実装とProduction昇格は一括で行わない。C0 Contract、C1 2D、C1 3D、C2 Cinematic、C3 Recording、C3 Virtual Production、C3 Advanced Adapterを個別Capability Gateで判定する。

## 2. 決定権と対象外

### 2.1 本書が決定するもの

- Camera Source Document、Cooked Artifact、Runtime Snapshot、Recording Artifact。
- Camera Rig Graphの公式Node、Port、評価順、state、上限。
- Director、Priority、Transition、Blend、Cut。
- Camera向けAI Semantic Catalog、Resolver、Operation、Preview、Diagnostic、Eval。
- 2D／3D、Pixel、Physical Lens、Split View、Cinematic Sequence。
- Multi-camera Pose／Lens recording、Take、Slate、Clock、Gap、Recovery。
- Virtual Production Device Gateway、Calibration、Timecode、Genlock、安全境界。
- Camera固有のCPU／Memory／Graph／Recording budgetとQualification。

### 2.2 他文書が決定するもの

| 主題 | 正本 |
|---|---|
| 座標、Quaternion、clip／depth、Projection初期値 | 2D／3D機能計画 |
| fixed tick、phase、command、snapshot、replay、memory | Runtime連携規約 |
| `RenderView`、jitter、history、Render Graph、GPU resource | Rendering規約 |
| Physics World、shape cast、query result、scene version | Physics規約 |
| Device sample、player input、action、replay input | Input規約 |
| ProjectRevision、ChangeSet、Commit、Undo、Recovery | Authoring Model規約 |
| MCD、Operation、Capability、Provider projection | 実行可能契約規約 |
| AI権限、Review、Receipt、Eval、Provenance | AIガバナンス／検証規約 |
| Media codec、audio/video container、hardware encoder | 将来のMedia／Capture Adapter規約 |

### 2.3 対象外

- C1／C2のmultiplayer spectator／broadcast authority。
- XR HMD poseとstereo projection。C3 Research前にOpenXR固有契約を別設計する。
- LED wall、nDisplay相当のcluster rendering、in-camera VFX color pipeline。
- Camera Tracking hardware、Lens encoder、Genlock generatorの自作。
- Runtimeでの任意Script Camera Node、JIT、Device SDK直接load。
- AIによる無承認Device接続、Recording開始、Take削除、外部Export。
- Camera画像からGameplay hit、visibility、AI perceptionを逆算する設計。

## 3. Capability成熟度

| Capability | C0 | C1 | C2 | C3 |
|---|---|---|---|---|
| Profile／Projection | MCD、Validator、Preview | Perspective／Orthographic／Pixel | Physical Lens、品質Profile | Live Lens metadata／calibration |
| Rig | Node Catalog、fake evaluator | Follow、Orbit、LookAt、Framing、Damping、Limit、Collision | Rail、Crane、Volume、advanced transition | Device Pose、recording overlay |
| Director | Default、Priority、Blend contract | Gameplay state選択 | Sequence／State driven、Split View | Multi-operator／recording selection |
| Presentation | channel分離 | Shake、Recoil、Noise | Cinematic handheld | live operator channel |
| Sequence | Schemaだけ | 対象外 | Shot、Track、Cut、Subsequence | Take conform／interchange |
| Recording | State／Artifact schema | 対象外 | 対象外 | Multi-camera Pose／Lens、Recovery |
| Virtual Production | fake Adapter contract | 対象外 | 対象外 | Live Device、Timecode、Genlock、Calibration |
| AI Authoring | Semantic Catalog、Operation schema | 2D／3D Gameplay intent | Cinematic shot intent | Recording／VP setup提案 |

C3の`recording`、`device_input`、`timecode`、`genlock`、`remote_preview`、各Hardware Adapterは別Capability IDを持つ。一つの成功をC3全体へ一般化しない。

## 4. Module Architecture

### 4.1 First-party target

| Target | 責務 | 禁止依存 |
|---|---|---|
| `mira_camera_contract` | MCD生成type、ID、enum、Operation、Diagnostic | Renderer／Physics／Device native header |
| `mira_camera_authoring` | Intent Resolver、Validator、cost、Cook | live Runtime World、GPU、Device SDK |
| `mira_camera_runtime` | C1 Rig、Director、Blend、Collision統合、Snapshot | Editor、Recording、Vendor API |
| `mira_camera_sequence` | C2 Shot、Track、Cut、Cinematic evaluator | Device SDK、GPU native API |
| `mira_camera_recording` | C3 Session、Take、spool、recovery、playback source | Network／Device API、Renderer native handle |
| `mira_camera_device_port` | Device／Clock／CalibrationのEngine-owned Port | Vendor型の公開 |
| `mira_camera_device_worker` | 別Process Adapter、packet decode、normalization | Project Commit、Gameplay World |
| `mira_camera_render_bridge` | Base Poseから`RenderView`入力を生成 | Physics query、Authoring write |
| `mira_camera_editor` | Inspector、Rig Graph、Preview、Debugger、AI projection | Project file直接write |

### 4.2 依存DAG

```text
mira_camera_contract
  ├─ mira_camera_authoring
  ├─ mira_camera_runtime
  │    ├─ mira_camera_sequence
  │    └─ mira_camera_render_bridge
  ├─ mira_camera_recording
  ├─ mira_camera_device_port
  │    └─ mira_camera_device_worker
  └─ mira_camera_editor

mira_camera_sequence  -> mira_camera_runtime public Portだけ
mira_camera_recording -> contract／sequence interchangeだけ
mira_camera_device_worker -> device_port IPCだけ
```

`mira_camera_runtime`はRecordingまたはDevice Workerをlinkしない。C1 GameHostとShipping packageへVirtual Production dependency、Network discovery、Device SDKを含めない。EditorHostもVendor SDKをin-process loadせず、Device Workerとbounded IPCで通信する。

### 4.3 Owner

| Data | 唯一のWriter |
|---|---|
| Camera Source Document | AuthoringCommandGatewayのCommit |
| Cooked `CameraPlanV1` | Camera Compiler |
| Base Rig state／Director state | `mira_camera_runtime`、`T30` |
| Collision result | Physics Adapter、Cameraへの統合は次tick `T20` |
| Shake／Noise channel | Presentation subsystem |
| Render interpolation／jitter／history | Rendering |
| Recording Session | `mira_camera_recording` |
| Device sample normalization | `mira_camera_device_worker` |
| Calibration approval | 人間承認済みAuthoring Operation |
| Take Artifact promotion | Recording Finalizer＋Authoring Gateway |

## 5. 正規DocumentとArtifact

### 5.1 Source Document

#### `CameraProfileDocumentV1`

```text
document_header
camera_profile_id
display_name
scene_dimension
projection
lens_profile
viewport_profile
exposure_profile_ref
post_process_profile_ref
focus_policy
culling_far_m
output_policy
capability_requirements[]
locked_fields[]
```

`projection`は`perspective`、`orthographic`、`pixel_orthographic`、`physical_perspective`のtagged unionである。C1初期値と範囲は2D／3D機能計画4.1節を正本とし、AspectをProfileへ重複保存しない。

#### `CameraRigDocumentV1`

```text
document_header
camera_rig_id
display_name
purpose
graph_version
nodes[]
edges[]
parameters[]
target_bindings[]
fallback_rig_id
comfort_profile_ref
capability_requirements[]
locked_fields[]
```

`purpose`は`gameplay`、`cinematic`、`spectator`、`recording`、`editor_preview`である。C1 Shippingは`gameplay`だけを必須保証し、`recording`と`editor_preview`をGameplay authorityへ使わない。

#### `CameraDirectorDocumentV1`

```text
document_header
camera_director_id
default_rig_id
rig_entries[]
selection_rules[]
transitions[]
shared_transition_id
viewport_bindings[]
```

すべてのDirectorは`default_rig_id`を必須とする。どのRuleにも一致しない場合、暗黙の最初のRigではなくDefault Rigを選ぶ。

#### `CameraPresentationProfileDocumentV1`

```text
document_header
camera_presentation_profile_id
shake_channels[]
noise_channels[]
recoil_channels[]
accessibility_binding
recording_channel_policy
hard_limits
locked_fields[]
```

Presentation ProfileはBase Rigへ接続しない。各channelはseed、開始条件、duration、translation／rotation amplitude、frequency、priority、combine policyを持ち、最終View合成と独立Recording channelにだけ使用する。

#### `CameraSequenceDocumentV1`

```text
document_header
camera_sequence_id
display_name
frame_rate
clock_domain
playback_range
pre_roll
post_roll
camera_bindings[]
tracks[]
markers[]
subsequences[]
export_profile_ref
```

`clock_domain`は`simulation_fixed`、`cinematic`、`editor_preview`、`external_timecode`である。Gameplay stateへ影響するTrackを`simulation_fixed`以外へ置くことを拒否する。

#### `CameraRecordingProfileDocumentV1`

```text
document_header
recording_profile_id
source_camera_ids[]
clock_profile
capture_channels[]
media_output_policy
spool_policy
storage_policy
gap_policy
retention_class
capability_requirements[]
```

#### `CameraDeviceProfileDocumentV1`

```text
document_header
camera_device_profile_id
adapter_capability_id
device_role
coordinate_mapping
clock_mapping
lens_mapping
calibration_ref
input_allowlist[]
output_allowlist[]
network_policy_ref
```

Device identifier、credential、live endpointはProject Sourceへ保存しない。Sourceへ保存できるのは論理Profile、Adapter Capability、承認済み接続Policy、Calibration Artifact参照だけである。

### 5.2 Derived Artifact

| Artifact | 内容 |
|---|---|
| `CameraPlanV1` | Profile、Rig Graph、DirectorをRuntime向けにcanonical compileしたimmutable plan |
| `CameraSequencePackageV1` | Track、binding、time transform、cut／blendをCookしたPackage |
| `CameraCalibrationArtifactV1` | coordinate transform、lens model、nodal offset、誤差、device／tool hash |
| `CameraTakeArtifactV1` | Take manifest、chunk hash、Gap、Clock、Calibration、media参照 |
| `CameraPreviewReceiptV1` | 入力revision、candidate、fixture、metrics、画像／trace hash |
| `CameraQualificationReceiptV1` | Capability、Target、hardware、performance、soak、fault結果 |

Source DocumentとDerived Artifactを同じFileへ混在させない。Takeの編集可能SourceはSequenceへのImport Operationで作り、Recording spoolを直接Sequenceとして扱わない。

## 6. 共通Runtime型

### 6.1 `CameraPoseV1`

```text
position_m: Vec3f
orientation_xyzw: NormalizedQuaternion
vertical_fov_rad: nullable<float>
orthographic_vertical_size_m: nullable<float>
focus_distance_m: positive_float
aperture_f_stop: nullable<float>
generation: uint64
```

Perspective／Physical Perspectiveでは`vertical_fov_rad`を必須、Orthographicでは`orthographic_vertical_size_m`を必須とし、両方を同時に持たない。Quaternion canonicalization、NaN／Inf拒否、右手系、+Y up、Camera view forward −Zは2D／3D機能計画に従う。

### 6.2 Snapshot

| Type | 内容 | Authority |
|---|---|---|
| `CameraIntentSnapshotV1` | Gameplay aim／visibilityへ必要な論理origin、direction、target、frustum intent | World／Gameplay入力 |
| `CameraBasePoseSnapshotV1` | 前／現在tickのBase Pose、Rig／Director generation、cut flag | Camera Runtime |
| `CameraPresentationSnapshotV1` | shake、noise、recoil channel | Presentation only |
| `CameraRenderViewV1` | 補間済みView／Projection入力、Viewport、exposure／post参照、history reset | Rendering only |
| `CameraRecordingSampleV1` | Base Pose、Lens、Focus、Presentation channel参照、Clock | Recording |

`CameraRenderViewV1`またはGPU matrixをGameplay aim、Physics、Save、AI perceptionへ返さない。GameplayがCamera方向を必要とする場合は`CameraIntentSnapshotV1`を使う。

### 6.3 Rational frame rate

`RationalFrameRateV1`は既約な正の`numerator／denominator`を持つ。公式初期allowlistは次である。

```text
24/1
24000/1001
25/1
30/1
30000/1001
48/1
50/1
60/1
60000/1001
120/1
```

Custom rateはC3 CapabilityとADRを必要とする。Drop-frame timecodeは`30000/1001`と`60000/1001`だけで許可する。Timecode labelからFrame rateを推測しない。

## 7. Camera Rig Graph

### 7.1 Graph規則

- Graphはtyped DAGとし、cycleを拒否する。
- Nodeはstable `node_id`、`node_type_id`、version、parameter、input port、output portを持つ。
- Edgeは同じPort型または明示されたlossless conversionだけを許可する。
- Topological orderが複数ある場合は`node_type_id、node_id`のUTF-8 byte順で固定する。
- stateful Nodeのstate slotはcanonical orderで割り当て、worker indexやallocation順へ依存しない。
- RigごとのNode上限64、Parameter上限64、Edge上限128とする。
- 一tick中のHeap allocation、Graph mutation、Device call、file／network I/Oを禁止する。
- C1／C2のBase Rigはdeterministic Nodeだけで構成する。

### 7.2 公式Node Catalog

| Domain | C1 Node | C2／C3 Node |
|---|---|---|
| Input | `TargetBinding`、`PlayerLookInput`、`GameplayCameraIntent` | `SequenceBinding`、`RecordedPoseInput`、`DevicePoseInput` |
| Position | `FixedPosition`、`Follow`、`Orbit`、`Boom` | `Rail`、`Crane`、`RecordedPosition` |
| Aim | `FixedAim`、`LookAt`、`GroupFraming`、`ScreenComposer` | `SequenceAim`、`OperatorAim` |
| Constraint | `WorldLimit`、`CameraCollision`、`HorizonLock`、`MotionComfort` | `CameraVolume`、`OccluderFadeProposal` |
| Filter | `Damping`、`DeadZone`、`LookAhead` | `RecordedFilter`、`OperatorSmoothing` |
| Lens | `PerspectiveLens`、`OrthographicLens`、`PixelLens` | `PhysicalLens`、`LiveLensInput` |
| Output | `BasePoseOutput` | `RecordingPoseOutput`、`PreviewPoseOutput` |

`Shake`、`Noise`、`Recoil`はBase Rig Nodeにしない。`CameraPresentationProfileV1`のbounded channelとしてBase Pose後へ合成する。Camera CollisionはPhysics query requestを作るだけで、Render threadまたはNode evaluatorからPhysics native objectを直接queryしない。

### 7.3 Custom Node

L3 `ProjectCameraRigNode`は次をすべて満たす場合だけ許可する。

- C++23 NativeGameModuleの承認済みSource。
- MCD生成Manifest、typed Port、read／write set、phase、memory、determinism宣言。
- R3以上のReview、static analysis、unit、replay、performance test。
- Shippingでoffline static linkされ、runtime dynamic load、JIT、script callを行わない。
- AIがNode sourceを自動Promotionしない。

Custom NodeがなくてもC1／C2 reference fixtureを実現できなければならない。

## 8. Director、Transition、Blend

### 8.1 Director

DirectorはActive Rigを次のcanonical順で選ぶ。

1. 明示Sequence override。
2. 有効なGameplay state rule。
3. priority降順。
4. specificity降順。
5. `rig_id` UTF-8 byte順。
6. 一致なしなら`default_rig_id`。

RuleはGameplayDefinitionのtyped state、tag、target availability、Camera Volume、Sequence markerを参照できる。任意C++ callback、Widget state、Renderer visibility、Device name文字列を条件にしない。

### 8.2 Transition

Transitionは`cut`、`smooth_blend`、`match_pose`、`match_lens`をC1／C2公式値とする。既定Blendは2D／3D機能計画どおり0.35秒、positionはcubic smoothstep、rotationはshortest-path normalized slerp、vertical FOVは`tan(fov/2)`空間で補間する。

`duration=0`は次の`T30`でcutとして適用し、同tickの`T90`がhistory reset eventを作る。Cut、teleport、non-jitter projection変更、surface／extent変更はRendering規約のhistory invalidationへ接続する。

## 9. AI Semantic／Authoring

### 9.1 Authoring Level

| Level | 表現 | 主な利用者 |
|---|---|---|
| L0 | 自然言語`CameraAuthoringIntentV1` | 初心者、AI Creator |
| L1 | 用途別Preset＋意味Parameter | Designer、AI |
| L2 | 型付きCamera Rig Graph／Director／Sequence | Advanced Designer、AI |
| L3 | 承認済みProject C++ Custom Node | Programmer |

AIはL0からL2を標準経路とする。L3を「柔軟性のため」という理由だけで選択せず、公式Nodeで表現不能なRequirement ID、代替、Risk、Testを提示する。

### 9.2 `CameraAuthoringIntentV1`

```text
purpose
viewpoint
primary_subjects[]
secondary_subjects[]
movement_style
composition_constraints[]
response_intent
occlusion_policy
motion_comfort
transition_intent
target_profiles[]
quality_intent
budget_intent
assumptions[]
locked_fields[]
```

Semantic Catalogは少なくとも次を正規語彙として持つ。

- `first_person`、`third_person`、`top_down`、`side_view`、`fixed_view`、`free_orbit`。
- `follow`、`orbit`、`rail`、`crane`、`handheld`、`recorded`。
- `center_subject`、`rule_of_thirds`、`headroom`、`look_room`、`group_framing`。
- `dead_zone`、`look_ahead`、`soft_follow`、`responsive_follow`。
- `avoid_camera_blocker`、`reposition`、`fade_occluder_proposal`、`cut_on_occlusion`。
- `comfortable`、`standard`、`intense`、`custom_bounded`。
- `cut`、`smooth_blend`、`match_pose`、`match_lens`。

自由文字列の「映画的」「酔いにくい」「いい感じ」を正規状態として保存しない。Resolverが型付きConstraint、Assumption、質問、候補へ展開する。

`motion_comfort=comfortable`の初期Resolutionは次とする。

```text
roll_limit_rad = 0
horizon_stabilization = true
angular_velocity_limit = CameraComfortProfileのcomfortable値
angular_acceleration_limit = CameraComfortProfileのcomfortable値
fov_change_rate_limit = CameraComfortProfileのcomfortable値
shake_scale = Accessibility Profile参照
```

Comfortの数値ProfileはTarget実測とUser Studyなしに「人間工学的に安全」と表示しない。C0では上限のSchemaと違反検出を実装し、C1 UX QualificationでPreset値を昇格する。

### 9.3 Resolver

```text
Natural language／Game Brief
  -> CameraAuthoringIntentV1
  -> CameraIntentResolver
  -> 最大3件のCameraPlanCandidateV1
  -> structural／semantic／capability／budget validation
  -> representative Preview
  -> CameraPreviewReceiptV1
  -> Camera ChangeSet
  -> AuthoringCommandGateway
```

候補は次を必須で持つ。

- 使用Profile、Rig、Director、Sequence。
- Intentから各Node／Parameterへのtrace。
- Assumption、未解決質問、locked field。
- Target別予測CPU／Memory／View cost。
- Capability maturityとfallback。
- Preview fixture、composition／comfort／collision metrics。
- 他候補との差と推奨理由。

Blockingはscene dimension、gameplay viewpoint、authority、必須subjectが解決不能な場合に限る。Pixel Diorama projection、motion comfortの大幅変更、Device／Network／RecordingはHigh ImpactとしてPreviewまたは明示承認を要求する。

### 9.4 公式Operation

| Operation | Risk | 結果 |
|---|---:|---|
| `ResolveCameraIntent` | R1 | 最大3 candidate。Project変更なし |
| `CreateCameraProfile` | R2 | Profile ChangeSet |
| `CreateCameraRig` | R2 | Rig ChangeSet |
| `AddCameraRigNode` | R2 | typed Node追加 |
| `ConnectCameraRigNodes` | R2 | typed Edge追加 |
| `SetCameraDirectorRule` | R2 | Rule／Transition変更 |
| `SetCameraPresentationProfile` | R2 | Shake／Noise／Recoil Profile変更 |
| `CreateCameraSequence` | R2 | Sequence ChangeSet |
| `PreviewCameraCandidate` | R1 | Preview Receipt |
| `AnalyzeCameraComposition` | R1 | metric／Diagnostic |
| `ProposeCameraDeviceProfile` | R2 | 未承認Profile提案 |
| `ProposeRecordingSession` | R2 | Session案。Armしない |
| `ImportCameraTake` | R2／R3 | TakeからSequence ChangeSet |

AI Providerへ公開しない`trusted_internal` Operationは次である。

- Device discovery／connect／disconnect。
- Calibration capture／approval。
- Recording `Arm`／`Start`／`Stop`。
- Take delete／retention変更。
- Network endpoint承認。
- 外部Media／Timeline export。

### 9.5 Validator

Camera ChangeSetは次を決定論的に検査する。

- Graph cycle、Port型、必須Output、Node／Edge／Parameter上限。
- 単位、range、finite、Quaternion、Projection／Lens variant。
- Gameplay Base RigへのPresentation／Device／Editor node混入。
- C1 RuntimeへのC2／C3 Capability混入。
- Stable ID、Target generation、fallback rig、Default Director。
- Viewport normalized bounds、split view overlap policy、Aspect重複保存。
- Physics query budget、camera blocker filter、Sensor除外。
- Comfort Profile hard constraint。
- Pixel cameraのinteger scale、rotation、jitter禁止。
- Target／Quality Capability、CPU／Memory／View cost。
- locked field、base revision、permission、approval。

Validation failure時はChangeSet全体をrejectし、live revisionを変更しない。近い値へのclamp、未知Nodeの無視、Default Rigの暗黙生成を行わない。

## 10. C1 Runtime

### 10.1 Tick接続

```text
T10_InputLatch
  Player Camera ActionをInputSnapshotへ固定

T20_AsyncIntegrate
  前tickのCamera Collision結果をowner generation／Physics scene version検査後に統合

T30_PrePhysics
  Director選択
  Rig parameter解決
  Base Rig state更新
  CameraBasePoseSnapshotV1候補生成
  version付きCamera Collision request生成

T40_MotionIntent
  Physics Adapterが直前完了Worldへtyped sphere cast

T60_PhysicsIntegrate
  query resultをEngine型へnormalize

T90_PresentationBuild
  Shake／Recoil／Noise／Cut history reset eventを構築

T100_ReplayCheckpoint
  Director、Rig state、Blend、受理async result、Base Poseを記録

T110_Publish
  前／現在Base Poseをimmutable RenderSnapshotへpublish
```

Camera RuntimeはWorld TransformのWriterにならず、Camera-owned stateだけを更新する。Gameplay aimはCamera Base Poseを逆参照せず、同じInput／Targetから生成された`CameraIntentSnapshotV1`を使う。

### 10.2 Collision

初期値はsphere radius 0.20 m、skin 0.05 m、最大補正距離10 mとする。Sensorを除外し、`camera_blocker` semantic roleを対象にする。

- 結果なしの1～2 tickは前回valid resultを保持する。
- 3 tick目はcollision補正なしへ切り替え、`CameraCollisionStale`を記録する。
- owner generationまたはPhysics scene version不一致をdiscardする。
- Render thread、Editor Preview draw、AI ToolからPhysics native queryを行わない。

### 10.3 Target lossとFailsafe

- Targetが一tickだけ欠落した場合は最終valid Base Poseを保持する。
- 2 tick目の`T30`でDirectorのDefault Rigへ遷移する。
- Runtime Rig評価がnon-finiteまたは契約違反なら最終valid Base Poseを保持する。
- 3 tick連続失敗でprecompiled Failsafe Rigへ切り替え、`CameraFailsafeActivated`を記録する。
- Default／Failsafe RigがCookされていないPackageは起動前に拒否する。

### 10.4 Render接続

Renderingは`R00_AcquireSnapshot`後、前／現在Base Poseをrender alphaで補間する。その後だけPresentation Shake／Noise、Lens、Viewport、jitterを合成して`CameraRenderViewV1`を作る。

Cut、teleport、non-jitter projection変更、extent／surface generation変更は同Frameの`history_reset=true`へ変換する。Temporal history、occlusion result、exposure resultをCamera RuntimeまたはGameplayへ書き戻さない。

### 10.5 Shake

Shakeはseed、開始tick、duration、translation／rotation amplitude、frequencyを持つbounded Presentation commandである。既定上限は同時16、translation各軸2 m、rotation各軸20°、frequency 0～60 Hz、duration 0～30 sとする。

Accessibility Profileのshake scaleを最終合成時に適用し、Gameplay aim、Physics、Save transform、Recording Base Poseへ戻さない。Recording時はBase PoseとShake channelを別々に保存する。

### 10.6 Split View

- Viewportはnormalized rectで最大4。
- 各Viewportは独立したDirector、Base Pose、Presentation channel、history keyを持つ。
- Input ownerとAudio listener bindingを明示し、最初のViewportを暗黙ownerにしない。
- Viewport overlapはEditor overlayまたは明示composition modeだけで許可する。
- 同じRig stateを複数Viewportから同時writeせず、共有Rigはimmutable evaluation inputからView別stateを持つ。

## 11. C2 Cinematic／Sequence

### 11.1 Sequence構造

公式Trackは次である。

- Camera binding。
- Transform／Rig parameter。
- Lens／focus／aperture。
- Director override。
- Camera cut／blend。
- Marker／Slate metadata。
- Subsequence。
- Presentation Shake／Noise。

Sequenceは可能な限りRig parameter、Target binding、Lens parameterを保持する。Transform BakeはExport用Derived Artifactであり、Source Sequenceへ不可逆上書きしない。

### 11.2 Clock

- Gameplayへ影響するSequenceは`simulation_fixed`で`T30`評価する。
- 映像専用Sequenceは`cinematic`または`editor_preview`で評価できる。
- `cinematic`／`editor_preview`結果をPhysics、AI、Save、Gameplay event authorityへ使わない。
- External Timecode追従はC3 Capabilityであり、C2 Shippingの必須条件にしない。

### 11.3 Cut／Blend

CutとBlendはDirectorの同じTransition contractを使う。Sequence専用の別補間実装を作らない。Cut frameはCamera pose、Lens、history reset、Sequence markerを同じframe IDへ固定する。

### 11.4 Interchange

OpenTimelineIOはTimeline、Track、Clip、Gap、Transition、Marker、外部Media参照のInterchange Adapter候補として扱う。Miraikanaiの正規Sourceは`CameraSequenceDocumentV1`であり、OTIOをRuntime正本またはProject全体の保存形式にしない。

OTIO import／export Adapterはversion、license、schema upgrade／downgrade、round-trip loss report、未知metadata隔離をQualificationした後だけ有効にする。Media byteはSequenceへ埋め込まず、Asset／Media ArtifactをStable IDで参照する。

## 12. C3 Recording

### 12.1 Session state

```text
Idle
  -> Configuring
  -> Armed
  -> Recording
  -> Finalizing
  -> Ready

Recording／Finalizing
  -> Recovering
  -> Recovered | Failed
```

不正transitionはtyped errorとし、暗黙Start、二重Stop、Failed Takeへの追記を禁止する。Recording開始はinteractive user presenceまたは承認済みAutomation identityを必要とし、AI ProviderへOperationを公開しない。

### 12.2 `CameraTakeManifestV1`

```text
take_id
session_id
slate
take_number
project_revision
contract_set_hash
camera_streams[]
frame_rate
clock_source
sync_quality
calibration_artifacts[]
start_frame
end_frame
chunks[]
gaps[]
markers[]
media_references[]
operator_notes_ref
provenance
```

各Camera StreamはBase Pose、Lens、focus、iris、Presentation channel、Clock sampleを別channelとして持つ。Shake／NoiseをBakeするかはExport Profileで選択し、Recording時にBase Poseへ不可逆合成しない。

### 12.3 Multi-camera policy

- `ReferenceRecordingProfileV1`はPose／Lens Stream最大16、各60 Hz。
- Low-resolution Previewは最大4、各960×540。
- Full-resolution Recording Viewは1、1920×1080、60 fps。
- Pose／Lens記録可能数とFull render可能数を同じ上限にしない。
- Preview budget超過時はpriority昇順でPreviewだけを停止し、Pose／Lens記録を維持する。
- 複数Full-resolution View、120 Hz、HDR、raw outputはC3 Advanced ProfileとHardware別Qualificationを必要とする。

Camera PlatformはPose、Lens、Timing、Media参照を所有する。Image sequence／video encodeはMedia Adapterへ渡し、codec、container、hardware encoderをCamera Runtimeへ実装しない。

### 12.4 Append-only spool

- metadata spool chunkは1秒または8 MiBの早い方で閉じる。
- 各Chunkはsequence、start／end frame、sample count、CRC32C、SHA-256、previous chunk hashを持つ。
- Chunk close時にfile flushし、manifest journalへ追記する。
- 起動時Recoveryは最後の完全Chunkまで検証し、不完全末尾を隔離する。
- 完全Chunkを黙って削除せず、欠損範囲を`CameraRecordingGapV1`へ記録する。
- Media Adapter outputは自身のatomic segment receiptを返し、Take manifestがhash参照する。

RecordingをArmする前に予定容量＋10%を予約する。予定時間がない場合はProfileの最長Take 30分で見積もる。Recording中はFinalization用に現在bitrateの10秒分を常時reserveし、下回る前に新規sample受付を停止して安全にFinalizationする。

## 13. C3 Timecode／Genlock

### 13.1 分離

- TimecodeはSampleへframe addressを付与する。
- Genlock／Custom TimestepはEngineまたはDeviceのframe cadenceを同期する。
- Monotonic Clockは順序、timeout、latency、drift測定に使う。
- Wall Clockは表示と来歴だけに使う。

Timecodeが存在するだけでGenlock済みと判定しない。Frame rate、drop-frame、Clock source、sync qualityを別Fieldで保存する。

### 13.2 `CameraClockSampleV1`

```text
source_id
frame_rate
frame_number
drop_frame
timecode_label
monotonic_timestamp_ns
sync_quality
discontinuity_generation
```

`sync_quality`は`locked`、`tracking`、`free_run`、`unqualified`、`lost`である。LTC、VITC、ATC、PTP、Platform API、Device SDKの固有値はAdapter内で正規型へ変換する。

### 13.3 Clock failure

次をClock discontinuityとする。

- Frame number逆行。
- Profile範囲を超えるFrame jump。
- rate／drop-frame変更。
- source generation変更。
- Genlock loss。
- monotonic timestamp逆行。

Failure時はProfileの`stop_take`、`record_gap_and_continue`、`unqualified_free_run`のいずれかを適用する。無言でEngine Timeへ切り替えず、Take manifestへdiscontinuity generationと範囲を記録する。

## 14. C3 Virtual Production

### 14.1 Device Gateway

Device Workerは次を正規化する。

- 6DoF tracking pose。
- focal length、sensor／filmback、zoom、focus、iris。
- lens distortion、nodal offset。
- operator input。
- preview output request。
- timecode／genlock status。
- device health、packet loss、latency。

Vendor type、pointer、socket、SDK enumをMCD、Project Source、Game C++へ公開しない。Gateway IPC payloadはbounded、version付き、little-endian canonical binary descriptorとし、native struct memory copyをWire formatにしない。

### 14.2 Calibration

`CameraCalibrationArtifactV1`は次を持つ。

- Device／Lens Profile ID。
- coordinate basis、scale、handedness conversion。
- tracking origin transform。
- nodal offset。
- distortion modelとparameter。
- focus／zoom sample range。
- calibration tool／version／hash。
- capture時刻、operator、error metric。
- Target／Capability。

Take開始時にCalibration generationを固定する。Recording中のCalibration、coordinate mapping、Lens Profile変更は同一Takeへ混在させず、現TakeをFinalizationして新Takeを作る。

### 14.3 Previewとauthority

Live Device PoseはEditor Preview CameraまたはRecording Cameraを駆動できる。Shipping Gameplay Camera、Gameplay aim、Physics、Save、Project Source、Approval stateへ直接writeできない。

Recorded PoseをGameplay SequenceへImportする場合は、別の`ImportCameraTake` ChangeSetでTarget、Clock、comfort、collision、Capabilityを再検証する。

## 15. Security／Privacy

### 15.1 Process／Network

- C1／C2のNetworkは既定無効。
- C3 Device Workerだけが承認済みNetwork Capabilityを取得できる。
- Adapter manifestのID、version、binary hash、signature／trust class、license、protocol、endpoint classを検証する。
- Userが承認したAdapter、network interface、subnet／endpoint、input／output roleだけを許可する。
- Packet size、rate、collection count、string byte、sample frequencyへhard capを持つ。
- malformed、NaN／Inf、重複sequence、Clock逆行、異常座標をProject／Runtime到達前に拒否する。
- CredentialはOS Secret Storeへ保存し、Project、Log、Receipt、AI Contextへ複製しない。
- Device Worker crashまたはcompromiseでAuthoring Commit、Gameplay memory、signing keyへ到達できないProcess権限にする。

### 15.2 Path／Metadata

- Recording出力は承認済みStaging Root配下へ限定する。
- Slate、Take、Device name、operator noteをPathとして連結せず、Stable IDからEngineがFile名を生成する。
- `..`、absolute path、separator、reserved name、alternate data streamを拒否する。
- Device名、Slate、Take note、OTIO metadataをPrompt／Tool instructionとして解釈しない。
- Raw Device Streamは`restricted`、正規化Pose／Lensは`project_private`、公開Exportは明示Policyを必要とする。
- 既存Takeを上書きせず、新しいTake IDを発行する。Deleteはtombstone＋Retention Serviceの承認操作とする。

### 15.3 AI権限

AIはDevice Profile、Recording Profile、Sequence、Rig、Previewを提案できる。Device discovery、接続、Calibration確定、Arm、Start、Stop、Delete、外部Export、Network許可は実行できない。

Prompt injectionを受けてもProviderへ`trusted_internal` OperationのSchemaを公開しない。Camera Tool結果はProject dataとuntrusted metadataを区別し、Device／Take由来文字列をinstruction channelへ昇格しない。

## 16. Budget

### 16.1 Graph／Authoring hard cap

| 項目 | 上限 |
|---|---:|
| Node／Rig | 64 |
| Edge／Rig | 128 |
| Parameter／Rig | 64 |
| Rig／Director | 16 |
| Transition／Director | 64 |
| Gameplay Viewport | 4 |
| Shake／View | 16 |
| Shot／Sequence | 256 |
| Camera binding／Sequence | 64 |
| Subsequence深さ | 8 |

上限を超えるSource ChangeSetを部分Commitせず、分割候補とcostを返す。C3 Advanced Profileで上限を増やす場合もMCD type version、memory、AI context、Editor virtualization、fuzz fixtureを同時更新する。

### 16.2 Runtime

Windows Reference条件はRuntime規約14.1節を使う。

| Metric | Soft | Hard |
|---|---:|---:|
| C1 Camera Runtime CPU P95 | 0.25 ms | 0.50 ms |
| Camera Runtime memory | 6 MiB | 8 MiB |
| tick中Heap allocation | 0 | 0 |
| Collision request／View／tick | 1 | 1 |
| Camera command queue | 75%未満 | overflow 0 |

Camera CPUはRuntime規約のGameplay Logic／T20／Presentationの内数であり、追加Frame budgetではない。Split Viewも合計0.50 ms hardを免除せず、超過ProfileはPlay／Package promotionを拒否する。

### 16.3 Recording

| 項目 | `ReferenceRecordingProfileV1` |
|---|---:|
| Pose／Lens Stream | 16 × 60 Hz |
| Low-resolution Preview | 4 × 最大960×540 |
| Full-resolution View | 1 × 1920×1080 60 fps |
| metadata chunk | 1秒または8 MiB |
| planned capacity reserve | 見積り＋10% |
| finalization reserve | 現在bitrate 10秒分 |
| continuous soak | 2時間 |

Media encodeのCPU／GPU／disk budgetはMedia Adapter Receiptへ分離し、Camera CPU値へ隠さない。

## 17. Failure／Diagnostic

| Failure | 状態変更 | Recovery／結果 |
|---|---|---|
| Graph／Profile不正 | live revision変更なし | ChangeSet reject、field Diagnostic |
| Director matchなし | Default Rig | Default欠落はCook reject |
| Target消失 | 1 tick保持 | 2 tick目にDefault Rig |
| Collision stale | 2 tick前回結果 | 3 tick目補正なし |
| Rig non-finite | 最終valid pose | 3 tick連続でFailsafe |
| View budget超過 | Base Pose維持 | 低priority Preview停止 |
| Device loss | 他Stream継続 | 対象Gap、PolicyでTake継続／停止 |
| Clock discontinuity | generation更新 | stop／gap／unqualified |
| Calibration mismatch | 現Take停止 | 新Takeへ分割 |
| Storage reserve不足 | Armしない／安全停止 | Finalization |
| Spool corrupt | live Takeへ追記停止 | 最終完全ChunkまでRecovery |
| Adapter crash | Camera Runtime影響なし | Worker再起動、Gap、再承認 |

公式Diagnostic IDは次とする。

```text
MIRA-CAMERA-RIG_CYCLE
MIRA-CAMERA-PORT_TYPE_MISMATCH
MIRA-CAMERA-TARGET_UNAVAILABLE
MIRA-CAMERA-COLLISION_STALE
MIRA-CAMERA-EVALUATION_NON_FINITE
MIRA-CAMERA-FAILSAFE_ACTIVATED
MIRA-CAMERA-BUDGET_EXCEEDED
MIRA-CAMERA-CLOCK_DISCONTINUITY
MIRA-CAMERA-GENLOCK_LOST
MIRA-CAMERA-DEVICE_LOST
MIRA-CAMERA-CALIBRATION_MISMATCH
MIRA-CAMERA-RECORDING_GAP
MIRA-CAMERA-SPOOL_CORRUPT
MIRA-CAMERA-STORAGE_RESERVE_INSUFFICIENT
MIRA-CAMERA-UNAUTHORIZED_OPERATION
```

DiagnosticはRequirement ID、Document／Node／Field path、actual、expected、Capability、Target、fallback、修正候補を持つ。AI向け説明だけでなくC++ enumとgenerated referenceを同じMCDから作る。

## 18. Editor／Debugger

Camera Workspaceは次を持つ。

- Camera Profile Inspector。
- Rig Graph Editor。
- Director／Transition table。
- Sequence／Shot／Take Browser。
- 2D／3D Scene Preview。
- Safe frame、frustum、subject bounds、rule-of-thirds、headroom、look-room overlay。
- Collision sphere／hit／stale result overlay。
- Base Pose、interpolated pose、Presentation poseの比較。
- Active Rig、Director rule、Blend、cut、history reset debugger。
- Clock、Timecode、Genlock、Device latency、packet loss、Calibration generation。
- Recording state、storage reserve、Chunk、Gap、Recovery status。
- AI candidate比較、Intent trace、Assumption、Cost、Diagnostic。

Editor ViewはCommitted Documentまたは明示DraftのProjectionであり、Widget／Graph canvasを正本にしない。Previewは同じCamera Compilerとfake／sandbox Runtimeを使い、Editor専用の別Rig評価を実装しない。

## 19. Test／AI Eval

### 19.1 Contract／Runtime

- MCD→C++／TypeScript／Cooked descriptor／MCP／Provider projection round-trip。
- Graph cycle、Port mismatch、unknown Node、64／128境界、non-finite、unit fuzz。
- canonical topological order、state slot、compile hash決定性。
- Director priority、specificity、Default、Transition全組合せ。
- Blend position／rotation／FOV、cut、history reset。
- Collision generation／scene version、Sensor除外、2 tick stale、3 tick fallback。
- target destroy、Rig evaluation fault、Failsafe。
- Replay Base Camera state hash一致。
- Split Viewのstate、Viewport、Input owner、history分離。

### 19.2 公式Scene

| Fixture | 証明するもの |
|---|---|
| `camera_pixel_2d_c1_v1` | integer scale、scroll、resize、rotation／jitter拒否 |
| `camera_follow_2d_c1_v1` | dead zone、look-ahead、World limit、target loss |
| `camera_third_person_c1_v1` | orbit、boom、framing、collision、狭い通路 |
| `camera_lock_on_c1_v1` | Player／Target同時framing、target switch |
| `camera_boss_group_c1_v1` | 複数subject、Director、fallback |
| `camera_motion_comfort_c1_v1` | roll、角速度／加速度、FOV、shake constraint |
| `camera_split_view_c2_v1` | 4 View、Input、history、budget |
| `camera_cinematic_c2_v1` | Shot、Cut、Blend、Lens、Focus、Subsequence |
| `camera_multistream_c3_v1` | 16 Pose／Lens、Clock、Gap、Preview縮退 |
| `camera_virtual_production_c3_v1` | Device、Calibration、Timecode、Genlock loss |
| `camera_recording_recovery_c3_v1` | write／flush／manifest／finalize各Process kill |

### 19.3 Composition／Comfort metric

- primary subjectが宣言safe region内にあるFrame 99.9%以上。
- hard subject visibilityが必要なFixtureで欠落0 frame。
- Camera blocker貫通0 frame。
- Comfort hard constraint違反0 frame。
- Pixel fixtureの非integer sample、rotation、temporal jitter 0 frame。
- Cutと同Frameのhistory reset漏れ0件。
- non-finite View／Projection 0件。

Metricだけで映像品質を完全判定しない。Preview image、trace、metric、human rubricを`CameraPreviewReceiptV1`へまとめる。

### 19.4 Recording／Security fault

- packet drop、duplicate、reorder、oversize、rate flood、NaN／Inf。
- Timecode逆行、rate変更、drop-frame不一致、Genlock loss。
- Device disconnect／reconnect、Worker crash、Calibration変更。
- disk full、permission denial、Chunk partial write、manifest partial write。
- Device名、Slate、Take note、OTIO metadataのPrompt injection／Path traversal。
- unauthorized connect／arm／record／delete／export。
- 2時間soak、完全Chunk消失0、意図しないGap 0。

### 19.5 AI Eval

Public、holdout、adversarial Corpusに少なくとも次を入れる。

- 「トップダウンで少し先を見る」。
- 「三人称で壁にめり込まず、敵をロックオン」。
- 「酔いにくく、揺れを減らす」。
- 「Pixel artを整数倍表示」。
- 「BossとPlayerを両方画面内」。
- 「3 CameraのShot listを作り、Cutを比較」。
- 「16 Cameraを同期記録」。
- 「Deviceが切れても欠損を隠さない」。
- Capability未対応Targetへの要求。
- Metadata内の命令を無視するadversarial case。

Release基準は次である。

| Metric | 合格 |
|---|---:|
| SchemaにないNode／Field／Operation | 0 |
| Unauthorized Device／Recording操作 | 0 |
| GameplayへのShake／Device Pose逆流 | 0 |
| unsafe approximation | 0 |
| Camera Task success | 95%以上 |
| 不要なBlocking質問 | 5%以下 |
| Capability／Assumption未報告 | 0 |

## 20. 段階導入

| Work package | 内容 | Gate |
|---|---|---|
| `CA0` | MCD、Semantic Catalog、Node／Operation／Diagnostic、fake evaluator | schema、round-trip、fuzz、AI public Eval |
| `CA1` | Compiler、Base Runtime、Director、Blend、Replay、fake Physics | deterministic、phase、memory、fault |
| `CA2` | C1 2D Orthographic／Pixel／Follow | Phase 3 fixture、1080p60 |
| `CA3` | C1 3D Perspective／Orbit／Framing／Collision | Phase 6 fixture、1080p60 |
| `CA4` | Camera Workspace、Preview、AI Resolver | candidate trace、manual／AI canonical ChangeSet |
| `CA5` | C2 Sequence、Shot、Cut、Split View、OTIO Adapter候補 | cinematic／split fixture、loss report |
| `CA6` | C3 Recording、Take、spool、recovery | multistream、Process kill、2時間soak |
| `CA7` | C3 Device Gateway、Calibration、Timecode、Genlock | security／clock／device qualification |
| `CA8` | C3 Advanced multi-render／120 Hz／hardware Adapter | hardware別Receipt |

Phase 0へ含めるのは`CA0`のMCD、fake Port、Validator、fixture定義までであり、Camera Runtime、Editor、Device SDKを同時実装しない。Phase 3で`CA1`／`CA2`、Phase 4で2D向け`CA4`、Phase 6で`CA3`、Phase 8以後に`CA5`～`CA8`を個別計画へ分解する。

## 21. Definition of Done

Camera Platformの設計完了条件は次である。

1. Camera Profile、Rig、Director、Presentation、Sequence、Recording、Deviceの正本とOwnerが一意である。
2. Gameplay Base Pose、Presentation、Render View、Recording Sampleのauthorityが分離されている。
3. Rig Node、Port、Graph上限、評価順、Custom Node条件が機械可読に定義されている。
4. AI Intent、Resolver、Operation、質問、Assumption、Preview、Diagnostic、Evalが定義されている。
5. C1 tick／Physics／Replay／Rendering接続が既存phaseと矛盾しない。
6. C2 SequenceとC3 Recordingが同じRig／Lens／Transitionを再利用し、別実装へ分裂していない。
7. TimecodeとGenlock、Clock failure、Calibration generation、Take recoveryが明示されている。
8. Device／Network／Path／Metadata／AI権限境界とnegative fixtureがある。
9. CPU、Memory、Graph、View、Recording hard capとQualification Gateがある。
10. C0～C3が個別Work packageとReceiptへ分解され、一括Production表示を防いでいる。

## 22. 公式資料と採用判断

### 22.1 Camera／Rig

- [Unity Cinemachine 3.1 manual](https://docs.unity3d.com/Packages/com.unity.cinemachine%403.1/manual/index.html): tracking、composition、blend、cutを手続き型Moduleへ分ける構造をcoverage Evidenceに使う。Unity Component／MonoBehaviour／Asset formatは採用しない。
- [Unreal Engine Gameplay Camera System Overview](https://dev.epicgames.com/documentation/unreal-engine/gameplay-camera-system-overview): Camera Asset、Rig、Transition、Director、Variable、Debuggerの分離を参照する。調査時点でExperimentalであるため、MiraikanaiのProduction根拠またはAPI baselineにはしない。
- [Godot Camera2D](https://docs.godotengine.org/en/4.5/classes/class_camera2d.html): smoothing、drag margin、limit、current cameraの単純なtyped property coverageを参照する。
- [Godot Camera3D](https://docs.godotengine.org/en/4.5/classes/class_camera3d.html): Perspective／Orthogonal／Frustum、CameraAttributes、Cull Mask、Viewport projectionのcoverageを参照する。
- [Godot SubViewport](https://docs.godotengine.org/en/stable/classes/class_subviewport.html): 独立Viewport、update policy、stereo view countのcoverageを参照する。

### 22.2 Recording／Virtual Production

- [Unreal Engine Take Recorder](https://dev.epicgames.com/documentation/en-us/unreal-engine/take-recorder-in-unreal-engine): Camera、Live Link、multiple takes、metadata、non-destructive Sequencer workflowを参照する。
- [Unreal Engine Live Link](https://dev.epicgames.com/documentation/en-us/unreal-engine/live-link-in-unreal-engine): 外部Sourceを共通InterfaceとAdapterで取り込む分離を参照する。
- [Unreal Engine Virtual Camera／Live Link](https://dev.epicgames.com/documentation/unreal-engine/controlling-a-virtual-camera-actor-using-live-link-in-unreal-engine): physical device、Lens／Focus／Exposure、Take連携のcoverageを参照する。
- [Unreal Engine Timecode and Genlock](https://dev.epicgames.com/documentation/unreal-engine/timecode-and-genlock-in-unreal-engine): Timecode providerとGenlock／custom timestepを別設定として扱う構造を参照する。

### 22.3 Interchange／Clock

- [OpenTimelineIO documentation](https://opentimelineio.readthedocs.io/en/latest/index.html): Timeline、Track、Clip、Transition、Marker、Gap、外部Media参照をInterchange Adapter候補にする。OTIOはmedia byteを内包しないため、Camera TakeまたはProject正本の代替にしない。
- [SMPTE ST 12-1](https://pub.smpte.org/pub/st12-1/st0012-1-2014.pdf): Timecode address、LTC／VITC、nominal frame rateの外部標準として参照する。標準のWire transportをCamera Coreへ直接実装せずAdapterで正規化する。

## 23. 明示的に採用しないもの

- Unity Cinemachine、Unreal Gameplay Camera／VCam、Godot Camera NodeのAPI／Asset互換。
- 一つの万能Camera GraphへGameplay、Presentation、Recording、Deviceを混在させる設計。
- Gameplay Cameraを任意C++ Scriptの`Update`／`LateUpdate`相当だけで実装する設計。
- Render後Camera transform、shake、occlusion、exposureをauthoritative gameplayへ戻す設計。
- Timecode存在をGenlock成立とみなす設計。
- Device SDKをEditorHostまたはGameHostへin-process loadする設計。
- OTIO、Vendor Take、Media containerをMiraikanai Projectの正本にする設計。
- AIへDevice接続、Recording開始、Take削除、Network許可を委任する設計。
- Clock loss、Device loss、Gap、Calibration変更を補間して無かったことにする設計。

本書にはC0～C3の実装計画作成を止める未確定選択肢を残さない。Hardware Adapter、Media codec、Custom rate、Comfort presetのProduction値は、Owner、初期安全状態、Qualification Gate、不合格時の非表示／拒否動作を持つ未実証項目として管理する。
