# Miraikanai Engine Camera Contract

- 文書ID: mirakan.arch.rendering-camera
- 状態: review
- 正本範囲: Gameplay／Cinematic CameraのProfile、Rig、Director、Presentation、Sequence、typed authoring、Base Pose runtime、Camera固有budget／diagnostic／qualification
- 非正本範囲: Capability maturity／roadmap、Render View execution／temporal history、Post Process schema、Physics query execution、Runtime phase／shared capacity、AI authorization、Evidence envelope、共通Schema／projection。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan（Recording／Timecode／Genlock／Virtual Productionはnot_activated）](../00-product/product-plan.md#8-future-portfolio)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Math／Core utilities](../02-foundation/math-core.md)、[Project state](../03-authoring/project-state.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Physics](../05-simulation/physics.md)、[Render Graph](render-graph.md)、[Post Processing](post-processing.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論と所有境界

CameraはProjection／Lens、Gameplay Rig、Director／Transition、Presentation、Cinematic SequenceをEngine-owned型とStable IDで接続する。AI、人間、Editor、Project C++は同じtyped operationとChangeSetを使い、GPU View、Physics native query、Editor widget stateをSourceにしない。

Gameplay Cameraはfixed simulation inputからdeterministic Base Poseを作る。Render interpolation、temporal jitter、shake／noiseはPresentationへ分離し、Gameplay aim、Physics、Save、AI perceptionへ返さない。Cinematicは同じRig、Lens、Director、TransitionをSequenceから駆動し、Gameplay authorityを持つ評価とpresentation-only評価をclock domainで分離する。

Product PlanだけがC1／C2のmaturity、Activation、導入順を所有する。本書はactive functional contractとqualification evidenceを定義するが、maturity値を持たない。

## 2. Source、artifact、runtime value

### 2.1 Source documents

```text
CameraProfileDocumentV1
  document_header
  camera_profile_id
  display_name
  scene_dimension
  projection
  lens_profile
  viewport_profile
  post_process_profile_ref
  post_process_override: optional PostProcessCameraOverrideV1
  focus_policy
  culling_far_m
  output_policy
  capability_requirements[]
  locked_fields[]
```

`projection`は`perspective | orthographic | pixel_orthographic | physical_perspective`のtagged unionである。AspectをProfileへ重複保存しない。Post Processは`PostProcessProfileV1` Stable IDとowner-defined `PostProcessCameraOverrideV1`だけを参照し、Exposure fieldをCamera形式へ複写しない。

| Reference profile | 初期値 | Validation |
|---|---|---|
| `PerspectiveGameplayV1` | reversed-Z perspective、vertical FOV 60 degree、near 0.05 m、culling far 10,000 m | FOV 5～170 degree、near 0.01～10 m、far>nearかつ最大1,000,000 m |
| `OrthographicGameplayV1` | reversed-Z orthographic、vertical size 11.25 m、near 0.01 m、far 1,000 m | size 0.001～1,000,000 m、far>near |
| `Pixel2DReferenceV1` | Orthographic、640×360、32 pixels/m、vertical size `360/32=11.25 m` | point sampling、integer scale、pixel-locked layerのtemporal jitter禁止 |

Viewportはnormalized `(x,y,width,height)`、各field `[0,1]`、width／height>0、right／bottom≤1。Aspectは実pixel extentから毎frame計算する。Perspectiveはfinite culling farをVisibility／Fogへ渡し、projectionだけinfinite-farにしない。

```text
CameraRigDocumentV1
  document_header
  camera_rig_id
  display_name
  purpose: gameplay | cinematic | spectator | editor_preview
  graph_version
  nodes[]
  edges[]
  parameters[]
  target_bindings[]
  fallback_rig_id
  comfort_profile_ref
  capability_requirements[]
  locked_fields[]

CameraDirectorDocumentV1
  document_header
  camera_director_id
  default_rig_id
  rig_entries[]
  selection_rules[]
  transitions[]
  shared_transition_id
  viewport_bindings[]

CameraPresentationProfileDocumentV1
  document_header
  camera_presentation_profile_id
  shake_channels[]
  noise_channels[]
  recoil_channels[]
  accessibility_binding
  hard_limits
  locked_fields[]

CameraSequenceDocumentV1
  document_header
  camera_sequence_id
  display_name
  frame_rate: RationalFrameRateV1
  clock_domain: simulation_fixed | cinematic | editor_preview
  playback_range
  pre_roll
  post_roll
  camera_bindings[]
  tracks[]
  markers[]
  subsequences[]
  export_profile_ref
```

Directorは`default_rig_id`を必須とし、一致なしで配列先頭を選ばない。Presentation channelはseed、start condition、duration、translation／rotation amplitude、frequency、priority、combine policyを持ち、Base Rigへ接続しない。

Sequenceの公式TrackはCamera binding、Transform／Rig parameter、Lens／focus／aperture、Director override、cut／blend、Marker／Slate metadata、Subsequence、Presentation Shake／Noiseである。Transform BakeはExport用Derived ArtifactでSourceを上書きしない。Gameplay stateへ影響するTrackは`simulation_fixed`だけ、他clock結果はPhysics、AI、Save、Gameplay event authorityに使わない。

`RationalFrameRateV1`は既約な正のnumerator／denominatorを持ち、allowlistは`24/1, 24000/1001, 25/1, 30/1, 30000/1001, 48/1, 50/1, 60/1, 60000/1001, 120/1`である。Labelからrateを推測しない。

### 2.2 Derived artifactとvalue

`CameraPlanV1`はProfile、Rig、Directorをcanonical compileしたimmutable runtime plan、`CameraSequencePackageV1`はTrack、binding、time transform、cut／blendのCooked packageである。`CameraPreviewReceiptV1`はinput revision、candidate、fixture、metrics、capture／trace hash、`CameraQualificationReceiptV1`はCapability、Target、performance、soak、fault結果を持つ。SourceとDerivedを同じFileへ混在させない。

```text
CameraPoseV1
  position_m: WorldPosition3f
  orientation_xyzw: NormalizedQuaternion
  vertical_fov_rad: nullable<float>
  orthographic_vertical_size_m: nullable<float>
  focus_distance_m: positive_float
  aperture_f_stop: nullable<float>
  generation: u64
```

Perspective系はvertical FOV、Orthographic系はvertical sizeを必須とし同時に持たない。Math ownerのfinite、Quaternion canonicalization、failure contractを使う。

`CameraIntentSnapshotV1`はGameplay用logical origin／direction／target／frustum intent、`CameraBasePoseSnapshotV1`はprevious／current Base Pose、Rig／Director generation、cut flag、`CameraPresentationSnapshotV1`はshake／noise／recoil、`CameraRenderViewV1`はRendering ownerが作るinterpolated View／Projection、Viewport、Post ref、history resetである。Render View／GPU matrixをGameplayへ返さない。

## 3. Rig、Director、transition

Rigはtyped DAGでcycleを拒否する。Nodeはstable ID、type ID、version、parameter、typed portsを持ち、edgeは同型または明示lossless conversionだけを許可する。Topological tie-breakは`node_type_id, node_id`のUTF-8 byte順、state slotもcanonical順であり、worker／allocation順へ依存しない。Rig上限はNode 64、Edge 128、Parameter 64。一tickのheap allocation、mutation、file／network I/O、native device callを禁止する。

公式Node Catalogは次である。

| Domain | Node |
|---|---|
| Input | `TargetBinding, PlayerLookInput, GameplayCameraIntent, SequenceBinding` |
| Position | `FixedPosition, Follow, Orbit, Boom, Rail, Crane` |
| Aim | `FixedAim, LookAt, GroupFraming, ScreenComposer, SequenceAim` |
| Constraint | `WorldLimit, CameraCollision, HorizonLock, MotionComfort, CameraVolume, OccluderFadeProposal` |
| Filter | `Damping, DeadZone, LookAhead` |
| Lens | `PerspectiveLens, OrthographicLens, PixelLens, PhysicalLens` |
| Output | `BasePoseOutput, PreviewPoseOutput` |

Shake／Noise／RecoilはGraph Nodeでなく`CameraPresentationProfileV1`のbounded channelである。CameraCollision Nodeはversioned query requestだけを作り、native Physics objectをqueryしない。`ProjectCameraRigNode`はMCD typed ports／read-write set／phase／memory／determinismを宣言したoffline-linked Native Game Moduleに限り、JIT／dynamic load／script callを禁止する。reference fixtureはCustom Nodeなしで成立させる。

Director selection順はSequence override、typed Gameplay rule、priority降順、specificity降順、Rig ID byte昇順、default Rigである。RuleはGameplayDefinition state／tag、target availability、Camera Volume、Sequence markerだけを参照し、arbitrary callback、Widget state、Renderer visibility、display nameを条件にしない。

Transitionは`cut | smooth_blend | match_pose | match_lens`。default durationは0.35秒、positionはcubic smoothstep `t²(3-2t)`、rotationはshortest-path normalized slerp、vertical FOVは`tan(fov/2)`空間で同じsmoothstepを使う。duration 0は次のCamera evaluation boundaryでcutとなる。Cut、teleport、non-jitter projection、surface／extent変更はRendering ownerのhistory invalidationへ接続する。

## 4. Authoring、runtime、sequence

`CameraAuthoringIntentV1`は`purpose, viewpoint, primary_subjects[], secondary_subjects[], movement_style, composition_constraints[], response_intent, occlusion_policy, motion_comfort, transition_intent, target_profiles[], quality_intent, budget_intent, assumptions[], locked_fields[]`を持つ。

Closed semantic vocabularyはviewpoint `first_person, third_person, top_down, side_view, fixed_view, free_orbit`、movement `follow, orbit, rail, crane, handheld`、composition `center_subject, rule_of_thirds, headroom, look_room, group_framing`、response `dead_zone, look_ahead, soft_follow, responsive_follow`、occlusion `avoid_camera_blocker, reposition, fade_occluder_proposal, cut_on_occlusion`、comfort `comfortable, standard, intense, custom_bounded`、transitionの4値である。自由文の「映画的」「酔いにくい」を保存せずtyped constraints／assumption／questionへ解決する。

Comfortable intentはroll limit 0、horizon stabilization true、angular velocity／acceleration／FOV change rateを`CameraComfortProfile`、shake scaleをAccessibility Profileから読む。数値Presetを人間工学的に安全と表示するにはTarget実測とUser Study evidenceを必要とする。

ResolverはIntentから最大3 `CameraPlanCandidateV1`を返す。候補はProfile／Rig／Director／Sequence、Intent-to-field trace、assumption／question／lock、Target cost、Capability／fallback、fixture、composition／comfort／collision metric、差と理由を持つ。scene dimension、gameplay viewpoint、authority、必須subjectが解決不能な場合だけBlocking questionを返す。

Operationは`ResolveCameraIntent, CreateCameraProfile, CreateCameraRig, AddCameraRigNode, ConnectCameraRigNodes, SetCameraDirectorRule, SetCameraPresentationProfile, CreateCameraSequence, PreviewCameraCandidate, AnalyzeCameraComposition`である。WriteはProject ChangeSetだけを生成し、共通operation envelope、authorization、approvalは各Ownerへ委譲する。

Validatorはgraph／port／output／limit、unit／range／finite／Quaternion／projection variant、Base RigへのPresentation／Editor Node混入、Stable ID／generation／fallback／default、viewport、Physics query domain ceiling、blocker filter／Sensor exclusion、comfort constraint、pixel integer scale／rotation／jitter、Target／Quality cost、lock／revisionを固定順で検査する。失敗時はChangeSet全体をrejectし、clamp、unknown Node無視、default生成をしない。

Runtime evaluationはInput Snapshot固定、前回versioned collision result統合、Director選択、Rig parameter／state更新、Base Pose候補と次query生成、Presentation channel／cut event構築、Base state checkpoint、immutable publishの依存順を持つ。具体phase名とlifetimeはRuntime scheduling ownerが決定する。CameraはWorld TransformのWriterにならない。

Collision初期値はsphere radius 0.20 m、skin 0.05 m、最大補正10 m、Sensor除外、`camera_blocker` roleである。resultなし1～2 tickは前valid、3 tick目は補正なし。owner generation／Physics scene version不一致をdiscardする。Target loss1 tickはlast valid pose、2 tick目はdefault Rig。non-finiteはlast validを保ち、3 tick連続でprecompiled Failsafeへ切り替える。Default／Failsafe欠落Packageは起動前に拒否する。

Renderingはprevious／current Base Poseをalpha補間後にPresentation、Lens、Viewport、jitterを合成する。Shakeはseed、start tick、duration、translation／rotation amplitude、frequencyを持ち、同時16、translation各軸2 m、rotation各軸20 degree、0～60 Hz、0～30秒を上限とする。Accessibility scaleを最終合成だけへ適用する。

Split Viewはnormalized viewport最大4、各viewが独立Director、Base Pose、Presentation、history key、Input owner、Audio listener bindingを持つ。OverlapはEditor overlayまたは明示compositionだけ。shared Rig inputはimmutable、stateはView別である。

Cinematic cut／blendはDirectorと同じcontractを使い、cut frameのpose、Lens、history reset、markerを同じframe IDへ固定する。`OpenTimelineIO`はinterchange adapter候補であってSource正本ではない。schema upgrade／downgrade、round-trip loss、unknown metadata isolationをqualificationし、media byteを埋め込まない。

## 5. Budget、failure、diagnostic

Camera domain hard capはRig当たりNode 64／Edge 128／Parameter 64、Director当たりRig 16／Transition 64、Gameplay Viewport 4、Shake／View 16、Sequence当たりShot 256／Camera binding 64、Subsequence depth 8である。超過ChangeSetを部分commitせず、分割候補とcostを返す。

Gameplay Camera RuntimeはCPU P95 soft 0.25 ms／hard 0.50 ms、memory soft 6 MiB／hard 8 MiB、tick heap allocation 0、collision request／View／tick 1である。Split Viewも合計hard cap内に収める。これらはCamera固有ceilingであり、共通frame／memory envelopeと測定法はRuntime performance ownerへ委譲する。

Camera command queueはactive Gameplay／Cinematic commandだけを対象とし、soft gateはoccupancyが75%未満、hard gateはoverflow countが正確に0である。観測window、warm-up、run集約は[Runtime performance ownerのreference measurement](../04-runtime/performance-capacity.md#8-measurementregressionpromotion)をそのまま使い、本書へ共通測定値を複写しない。Soft pressureではlow-priority Editor Preview commandの生成を止めてBase Poseを維持し、受理済みGameplay commandをdrop／上書き／次tickへ暗黙繰越しない。Hard overflowは`MIRAKAN-CAMERA-BUDGET_EXCEEDED`を発行し、該当runとPlay／Package promotionを失敗させる。

Failureはinvalid graph／profileでlive revision不変、Director no-matchでdefault、target loss／collision stale／non-finiteで前節のfallback、View costまたはcommand queue soft pressureでBase Poseを維持してlow-priority Preview停止とする。

Closed Diagnosticは`MIRAKAN-CAMERA-RIG_CYCLE, MIRAKAN-CAMERA-PORT_TYPE_MISMATCH, MIRAKAN-CAMERA-TARGET_UNAVAILABLE, MIRAKAN-CAMERA-COLLISION_STALE, MIRAKAN-CAMERA-EVALUATION_NON_FINITE, MIRAKAN-CAMERA-FAILSAFE_ACTIVATED, MIRAKAN-CAMERA-BUDGET_EXCEEDED, MIRAKAN-CAMERA-UNAUTHORIZED_OPERATION`である。DiagnosticはRequirement ID、Document／Node／Field path、actual、expected、Capability、Target、fallback、remediationを持つ。

## 6. Editorとqualification

WorkspaceはProfile Inspector、Rig Graph、Director／Transition、Sequence／Shot、2D／3D Preview、safe frame／frustum／subject／composition overlay、collision overlay、Base／interpolated／Presentation pose比較、Active Rig／rule／blend／cut／history debugger、AI candidate trace／assumption／cost／Diagnosticを持つ。Widget／canvasをSourceにせず、Previewは同じcompilerとsandbox runtimeを使う。

Contract／runtime fixtureはSchema round-trip、cycle／port／unknown／64・128 boundary／non-finite／unit fuzz、canonical graph hash、Director全組合せ、position／rotation／FOV blend、cut／history、collision generation／stale、target loss／Failsafe、checkpoint hash、Split View isolationを含む。Camera queue fixtureは同じactive Sceneへbounded command burstとlow-priority Previewを重ね、reference measurement windowの全sampleでoccupancy 75%未満、overflow 0、受理済みGameplay command drop 0、Base Pose／Replay checkpoint hash不変を検証し、overflow fault injectionが`MIRAKAN-CAMERA-BUDGET_EXCEEDED`とpromotion失敗へなることを確認する。

公式Sceneは`camera_pixel_2d_c1_v1, camera_follow_2d_c1_v1, camera_third_person_c1_v1, camera_lock_on_c1_v1, camera_boss_group_c1_v1, camera_motion_comfort_c1_v1, camera_split_view_c2_v1, camera_cinematic_c2_v1`である。fixture名のsuffixはhistorical test IDでありmaturityを定義しない。

Composition gateはprimary subject safe-region 99.9%以上、required visibility missing 0 frame、blocker penetration 0、comfort hard violation 0、pixel non-integer／rotation／temporal jitter 0、cut history-reset漏れ0、non-finite View／Projection 0である。Metric、capture、trace、human rubricを同じPreview Receiptへ結ぶ。

AI corpusはtop-down look-ahead、third-person collision／lock-on、reduced shake、pixel integer scale、boss group framing、multi-shot cut comparison、unsupported Targetを含む。Gateはunknown Node／Field／Operation 0、PresentationのGameplay逆流0、unsafe approximation 0、task success 95%以上、不要Blocking質問5%以下、Capability／assumption未報告0である。Evidenceのrun、holdout、gradingはVerification ownerが決定する。
