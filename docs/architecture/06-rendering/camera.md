# Miraikanai Engine Camera Contract

- 文書ID: mirakan.arch.rendering-camera
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Gameplay／Cinematic CameraのProfile、Rig、Director、Presentation、Sequence、typed authoring、Base Pose runtime、Camera固有budget／diagnostic／qualification
- 非正本範囲: Capability maturity／roadmap、Render View execution／temporal history、Post Process schema、Physics query execution、Runtime phase／shared capacity、AI authorization、Evidence envelope、共通Schema／projection。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Render Graph](render-graph.md)、[Math／Core Utilities](../02-foundation/math-core.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)
- 関連文書: [Product Plan（Recording／Timecode／Genlock／Virtual Productionはnot_activated）](../00-product/product-plan.md#8-future-portfolio)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Math／Core utilities](../02-foundation/math-core.md)、[Project state](../03-authoring/project-state.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Physics](../05-simulation/physics.md)、[World](world.md)、[Render Graph](render-graph.md)、[LOD](lod.md)、[Virtualized／Continuous Geometry](virtualized-continuous-geometry.md)、[Post Processing](post-processing.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-21

## 1. 結論と所有境界

CameraはProjection／Lens、Gameplay Rig、Director／Transition、Presentation、Cinematic SequenceをEngine-owned型とStable IDで接続する。AI、人間、Editor、Project C++は同じtyped change primitiveとChangeSetを使い、GPU View、Physics native query、Editor widget stateをSourceにしない。

Gameplay Cameraは選択済みSimulation Cadenceのimmutable inputからdeterministic Base Poseを作る。Render interpolation、temporal jitter、view impulse／view noiseはPresentationへ分離し、Gameplay intent、Physics、Save、AI perceptionへ返さない。Cinematicは同じRig、Lens、Director、TransitionをSequenceから駆動し、Gameplay authorityを持つ評価とpresentation-only評価をclock domainで分離する。

Product PlanだけがC1／C2のmaturity、Activation、導入順を所有する。本書はactive functional contractとqualification evidenceを定義するが、maturity値を持たない。

## 2. Source、artifact、runtime value

### 2.1 Source documents

```text
CameraProfileDocumentV1
  document_header
  camera_profile_id
  display_name
  compatible_world_space: WorldSpaceCompatibilityV1
  projection
  lens_profile
  viewport_profile
  post_process_profile_ref
  post_process_override: optional PostProcessCameraOverrideV1
  focus_policy
  culling_far_m
  output_policy: CameraOutputPolicyV1
  capability_requirements[]
  locked_fields[]
```

`compatible_world_space`は[World](world.md)が所有するreusable compatibility constraintであり、Cameraがscene dimensionやhybrid gameplay authorityを所有・選択するfieldではない。Worldを参照するCamera解決では、入力`WorldSpaceProfileRefV1`がそのconstraintを満たすことを検証し、選択済みexact refを`CameraPlanCandidateV1`とCooked `CameraPlanV1`へ記録する。compatibleでないProfile、stale ref、WorldなしのGameplay／Cinematic Cameraを拒否し、2D／3DをCamera既定値から推測しない。UI-only／Editor preview CameraはWorldを要さないが、Worldを読むPreviewは同じ検証を行う。

`projection`は`perspective | orthographic | pixel_orthographic | physical_perspective`のtagged unionである。AspectをProfileへ重複保存しない。Post Processは`PostProcessProfileV1` Stable IDとowner-defined `PostProcessCameraOverrideV1`だけを参照し、Exposure fieldをCamera形式へ複写しない。`focus_policy`は`manual | target_distance | subject_group`に閉じる。`manual`はlens profileのfocus distance、`target_distance`はprimary target bindingへの距離、`subject_group`は[Post Processing](post-processing.md)の`DepthOfFieldProfileV1`のsubject group focusへ接続し、DOF parameterをCamera形式へ複写しない。

`CameraOutputPolicyV1`は`kind: primary_surface | shared_surface_viewport | offscreen_render_target | editor_preview`と、kind別のexact bindingだけを持つtagged unionである。`primary_surface`はPlay Sessionのprimary ViewFamilyを一件だけ選び追加bindingを禁止する。`shared_surface_viewport`は`view_family_ref`と`viewport_profile`、`offscreen_render_target`は`render_target_profile_ref`、`editor_preview`はEditor-owned `preview_session_ref`を必須とし、他kindのbinding fieldを拒否する。`editor_preview`はShipping PackageへCookせず、`offscreen_render_target`をpresent surfaceとして扱わない。未登録kind、暗黙のprimary選択、同一Cameraから複数kindへのfan-outを拒否する。

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
  view_impulse_channels[]
  view_noise_channels[]
  accessibility_binding
  hard_limits
  locked_fields[]

CameraSequenceDocumentV1
  document_header
  camera_sequence_id
  display_name
  frame_rate: RationalFrameRateV1
  clock_binding:
    kind: runtime_domain | editor_preview
    runtime_domain_ref: optional exact ClockDomainRefV1
    editor_preview_clock_ref: optional exact Editor-owned preview clock ref
  playback_range
  pre_roll
  post_roll
  camera_bindings[]
  tracks[]
  markers[]
  subsequences[]
  export_profile_ref
```

Directorは`default_rig_id`を必須とし、一致なしで配列先頭を選ばない。`view_impulse_channels[]`はboundedな一過性translation／rotation impulse、`view_noise_channels[]`はseed付きbounded noiseとして、start condition、duration、amplitude、frequency、priority、combine policyを持つ。意味はowner-qualified cue bindingが宣言し、Core CameraはWeapon、Damage、Explosionその他のGenre原因をchannel IDから推測しない。どちらもBase Rigへ接続しない。

Sequenceの標準Track候補はCamera binding、Transform／Rig parameter、Lens／focus／aperture、Director override、cut／blend、Marker／Slate metadata、Subsequence、Presentation View Impulse／View Noiseである。Transform BakeはExport用Derived ArtifactでSourceを上書きしない。`kind=runtime_domain`は[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md#41-clock-domainpausegameplay-timer)のactive Clock Domain Registry／Profileへexact解決し、Entryが`sequence` consumerを許可しなければならない。Gameplay stateへ影響するTrackは`clock.domain.core.gameplay@1`だけ、他clock結果はPhysics、AI、Save、Gameplay event authorityに使わない。`kind=editor_preview`は`runtime_domain_ref`を禁止してShipping PackageへCookしない。

`RationalFrameRateV1`は既約な正のnumerator／denominatorを持ち、allowlistは`24/1, 24000/1001, 25/1, 30/1, 30000/1001, 48/1, 50/1, 60/1, 60000/1001, 120/1`である。Labelからrateを推測しない。

`runtime_domain_ref=clock.domain.core.gameplay@1`かつ選択Cadenceが`fixed`の場合、0開始の`frame_index`から1開始のSimulation Advanceへの写像は整数演算`advance_sequence = 1 + floor(frame_index × cadence_rate_numerator_hz × frame_rate_denominator / (cadence_rate_denominator × frame_rate_numerator))`に固定する。rateは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)のexact `SimulationCadenceProfileV1`から読み、Camera schemaやABIへ60を複写しない。中間丸めと逆写像を使わず、Gameplayへ影響するTrack評価とcut frame固定はこの写像後の`advance_sequence`で行う。frame rateを整除しないCadenceでは複数frameが同一advanceへ写像されうる。`variable | turn_based | explicit_step` Cadenceはkind別のSequence mapping policyとQualificationがactiveになるまでGameplay-authoritative Trackを`sequence_cadence_mapping_unavailable`で拒否し、Presentation-only Trackへ権限を拡張しない。Cinematic defaultは`clock.domain.camera.cinematic@1`へexact解決し、unknown／unqualified Domain、Profile非選択、`sequence` consumer不許可をSequence compile前に拒否する。

```text
CameraComfortProfileV1
  document_header
  comfort_profile_id
  roll_limit_rad
  max_angular_velocity_rad_s
  max_angular_acceleration_rad_s2
  max_fov_change_rate_per_s
  locked_fields[]
```

`comfort_profile_ref`と§4のcomfort constraint検査は`CameraComfortProfileV1`だけを参照する。全fieldはfinite、`roll_limit_rad`は`[0, π]` radian、velocity／accelerationは0より大きいradian/s・radian/s²、FOV change rateは`tan(fov/2)`空間の0より大きい1/second変化率とする。本schemaは数値Presetを持たず、安全表示の条件は§4に従う。

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

`CameraIntentSnapshotV1`はGameplay用logical origin／direction／target／frustum intent、`CameraBasePoseSnapshotV1`はprevious／current Base Pose、Rig／Director generation、cut flag、`CameraPresentationSnapshotV1`はview impulse／view noise channelを持つ。Rendering ownerはこれらからcanonical `RenderViewV1`を作り、interpolated View／Projection、Viewport、Post ref、cut／history resetを束縛する。Camera Ownerは別名のRender View recordを所有せず、Render View／GPU matrixをGameplayへ返さない。

`CameraPresentationSummaryV1`は本書がOwnerとして公開するread-only／revisioned projectionであり、最低fieldとして`camera_id`、source revision、projection variant、vertical FOVまたはorthographic vertical size、focus distance、aperture、cut／history reset flag、pixel-locked policyを持つ。[Post Processing](post-processing.md)等のConsumerはfield一覧を複写せず、write backしない。

[LOD](lod.md#4-共通選択契約)は選択済み`RenderViewV1`からだけ`ViewLodContextV1`を構築する。対応は`perspective | physical_perspective -> perspective{vertical_fov_rad,near_m}`、`orthographic | pixel_orthographic -> orthographic{vertical_span_m,near_m}`で、render extent、view transform、view purpose、view identity／generation、cut／history reset、Target／Qualityもbyte-exactに投影する。LODはCamera Source、Lens、display aspect、Post Processからprojectionを再解決せず、Cameraはprojected error、LOD threshold、tier、pressureを所有しない。cut、projection variant、render extent generationの変化はLODへ明示し、古いView historyを別View／generationへ流用しない。

[Virtualized／Continuous Geometry](virtualized-continuous-geometry.md)のinner cutも同じ`RenderViewV1`／`ViewLodContextV1`だけをView入力とし、virtual path専用Camera、FOV補正、resolution scale、camera cut判定を作らない。main、shadow、reflection、Editor、thumbnail、split ViewはそれぞれのView ID／generationで独立cutを持ち、別Viewのhistory、error、fallbackを共有しない。Cameraはmicro-cluster、page request、residencyまたはcapacityを所有しない。

## 3. Rig、Director、transition

Rigはtyped DAGでcycleを拒否する。Nodeはstable ID、type ID、version、parameter、typed portsを持ち、edgeは同型または明示lossless conversionだけを許可する。Topological tie-breakは`node_type_id, node_id`のUTF-8 byte順、state slotもcanonical順であり、worker／allocation順へ依存しない。Rig上限はNode 64、Edge 128、Parameter 64。一Simulation Advanceのheap allocation、mutation、file／network I/O、native device callを禁止する。

標準Node Catalog候補は次である。

| Domain | Node |
|---|---|
| Input | `TargetBinding, ViewControlInput, GameplayCameraIntent, SequenceBinding` |
| Position | `FixedPosition, Follow, Orbit, Boom, Rail, Crane` |
| Aim | `FixedAim, LookAt, GroupFraming, ScreenComposer, SequenceAim` |
| Constraint | `WorldLimit, CameraCollision, HorizonLock, MotionComfort, CameraVolume, OccluderFadeProposal` |
| Filter | `Damping, DeadZone, LookAhead` |
| Lens | `PerspectiveLens, OrthographicLens, PixelLens, PhysicalLens` |
| Output | `BasePoseOutput, PreviewPoseOutput` |

View Impulse／View NoiseはGraph Nodeでなく`CameraPresentationProfileDocumentV1`のbounded channelである。CameraCollision Nodeはversioned query requestだけを作り、native Physics objectをqueryしない。`ProjectCameraRigNode`はMCD typed ports／read-write set／phase／memory／determinismを宣言したoffline-linked Native Game Moduleに限り、JIT／dynamic load／script callを禁止する。reference fixtureはCustom Nodeなしで成立させる。

Director selection順はSequence override、typed Gameplay rule、priority降順、specificity降順、Rig ID byte昇順、default Rigである。RuleはGameplayDefinition state／tag、target availability、Camera Volume、Sequence markerだけを参照し、arbitrary callback、Widget state、Renderer visibility、display nameを条件にしない。

Transitionは`cut | smooth_blend | match_pose | match_lens`。default durationは0.35秒、positionはcubic smoothstep `t²(3-2t)`、rotationはshortest-path normalized slerp、vertical FOVは`tan(fov/2)`空間で同じsmoothstepを使う。`match_pose`は遷移開始advanceの現在Base Poseへ遷移先RigのFilter／Damping系Node stateを整合初期化してposition／orientationを連続に保ち、Lens fieldだけを上記補間でblendする。整合不能またはstate初期化がnon-finiteの場合は`smooth_blend`へfallbackする。`match_lens`はposition／orientationをcutし、vertical FOV（`tan(fov/2)`空間）、orthographic vertical size、focus distance、apertureだけを上記補間で連続化し、cutと同じhistory invalidationを接続する。durationは4種共通で、duration 0は次のCamera evaluation boundaryでcutとなる。ただし`match_pose`のduration 0は整合初期化だけを行いblendしない。Cut、teleport、non-jitter projection、surface／extent変更はRendering ownerのhistory invalidationへ接続する。

## 4. Authoring、runtime、sequence

Camera authoringの意味語彙はCore enumへ固定せず、Owner-qualified contributionを次のRegistryへ合成する。CoreはRegistry schemaと初期defaultだけを所有し、Feature Pack、Genre Pack、ProjectのIDを列挙または参照しない。

```text
CameraSemanticRefV1
  axis_id: viewpoint | movement | composition | response | occlusion | comfort | transition
  semantic_id: namespace付きStableId
  semantic_version: positive uint32
  semantic_content_hash: SHA-256

CameraSemanticContributionV1
  semantic_ref: CameraSemanticRefV1
  owner_ref: exact {owner_id, owner_revision, owner_content_hash}
  contribution_layer: core | feature_pack | genre_pack | project
  display_name
  semantic_description
  search_synonyms[]
  constraint_schema_ref
  executor_binding_refs[]
  required_capability_refs[]
  target_support_refs[]

CameraSemanticRegistryV1
  registry_id: registry.camera.authoring_semantic
  registry_version
  contributions[1..1024]
  registry_content_hash: SHA-256

CameraSemanticActivationProjectionV1
  projection_id
  projection_version: positive uint32
  registry_ref: exact {registry_id, registry_version, registry_content_hash}
  selected_semantic_refs[1..1024]: CameraSemanticRefV1
  qualification_binding_refs[1..1024]:
    exact {binding_id, binding_version, binding_content_hash}
  projection_content_hash: SHA-256
```

`semantic_content_hash`はASCII `MIRAKAN_CAMERA_SEMANTIC_CONTRIBUTION_V1`と自己hashを除くReceipt-free Contribution canonical bytes、Registry hashはASCII `MIRAKAN_CAMERA_SEMANTIC_REGISTRY_V1`、Registry ID／version、axis／semantic ID／version順の全Contribution canonical bytesから計算する。selected refsはaxis／semantic ID／version／hash順、Binding refsは解決したsubject refの同じ順にstrict sortし、duplicateを拒否する。Activation Projectionのselected ref集合、Qualification Bindingが解決する合格かつfreshなsubject集合はexact set equalityとし、Receipt／BindingをContribution／Registry hashへ戻さない。Feature／Genre contributionは所有Pack identity、Project contributionはProject owner identityへexact解決する。各Ownerは自己namespaceだけへ追加でき、Core entryの上書き、同一logical IDの別hash、unknown／stale owner、未Qualification、Target／Capability不成立をfail closedにする。Registry materializationはProject／Pack dependency closureを解決したCompilerが行い、Generic Engine CoreからPackへのdependency edgeを生成しない。

initial V1のCore defaultはexact 32 entryである。viewpointは`first_person | third_person | top_down | side_view | fixed_view | free_orbit`、movementは`follow | orbit | rail | crane | handheld`、compositionは`center_subject | rule_of_thirds | headroom | look_room | group_framing`、responseは`dead_zone | look_ahead | soft_follow | responsive_follow`、occlusionは`avoid_camera_blocker | reposition | fade_occluder_proposal | cut_on_occlusion`、comfortは`comfortable | standard | intense | custom_bounded`、transitionは`cut | smooth_blend | match_pose | match_lens`を、それぞれ`camera.semantic.<axis>.<value>@1`として登録する。この集合はinitial canonical fixtureを構成する開始値であってclosed上限ではない。Feature／Genre／Projectは同じaxisへqualified entryを追加できるが、新しいexecutor primitive、Physics query、Presentation side effectをsemantic名だけで作れず、登録済み`executor_binding_refs[]`へ解決する。

initial V1 Sourceは上記の完全な`CameraSemanticRefV1`だけを受理し、表記上のvalue、suffix、synonym、display nameをsemantic refまたはaliasとして受理しない。`CameraSequenceDocumentV1`は`clock_binding`だけを持ち、裸のClock Domain scalarをSchemaへ定義しない。

`CameraAuthoringIntentV1`はexact `{registry_id, registry_version, registry_content_hash}`、exact `{projection_id, projection_version, projection_content_hash}`、`purpose, viewpoint_ref, primary_subjects[], secondary_subjects[], movement_semantic_ref, composition_semantic_refs[], response_semantic_ref, occlusion_semantic_ref, comfort_semantic_ref, transition_semantic_ref, target_profiles[], quality_intent, budget_intent, assumptions[], locked_fields[]`を持つ。各semantic refは同じactive Registry／Activation Projectionへexact解決する。自由文の「映画的」「酔いにくい」、display name、synonymをSourceへ保存せず、qualified ref、typed constraints、assumption、questionへ解決する。0件または複数の意味同等候補、unknown／unqualified ref、axis mismatchはBlocking questionまたはtyped rejectにし、近い名前、Genre既定、配列先頭へ黙ってfallbackしない。

上記Registry、Projection、Core default／extension fixtureはtarget contractであり、実装済み、active Gateway surface、Production Qualification済みという主張ではない。

Comfortable intentはroll limit 0、horizon stabilization true、angular velocity／acceleration／FOV change rateを`CameraComfortProfileV1`（§2.1）、view impulse／view noise scaleをAccessibility Profileから読む。数値Presetを人間工学的に安全と表示するにはTarget実測とUser Study evidenceを必要とする。

ResolverはIntentから最大3 `CameraPlanCandidateV1`を返す。候補はProfile／Rig／Director／Sequence、exact `WorldSpaceProfileRefV1`またはWorld不要の明示、Intent-to-field trace、assumption／question／lock、Target cost、Capability／fallback、fixture、composition／comfort／collision metric、差と理由を持つ。World compatibility、gameplay viewpoint、authority、必須subjectが解決不能な場合だけBlocking questionを返す。

予約候補IDは`operation.camera.resolve_intent, operation.camera.create_profile, operation.camera.set_profile_projection, operation.camera.create_rig, operation.camera.add_rig_node, operation.camera.connect_rig_nodes, operation.camera.set_director_rule, operation.camera.set_presentation_profile, operation.camera.create_sequence, operation.camera.preview_candidate, operation.camera.analyze_composition`のexact 11件に閉じる。これらは[Executable contracts](../02-foundation/executable-contracts.md#211-既存domain文書から回収した未登録operation候補)の`planning.operation_family.camera_authoring@1`だけに属する未Activation vocabularyであり、current MCD／Owner Manifest／Service allowlist／Policy／Validator／Diagnostic／Receipt／Provider／MCP／generated alias／legacy alias集合はすべて`[]`、Capability stateは`not_activated`である。`operation.camera.resolve_intent`はActivation後にだけRegistry selectionを行う予定候補であり、現在のselection権限ではない。Registry contribution自体のauthoring／activationは別計画語彙`planning.camera.semantic_registry_contribution_authoring@1`とし、reserved Operation ID集合`[]`、current MCD／Owner Manifest／Service allowlist／Policy／Validator／Diagnostic／Receipt／Provider／MCP／CLI／Editor／generated alias／legacy alias集合もexact `[]`、Capability stateも`not_activated`である。Foundationが専用Operationをatomic登録するまで、既存11候補からContribution登録権限を推測しない。`activation.camera.authoring_operations.v1`が11件を同じContract set transactionで完全登録するまでGatewayはdispatchせず、要求を`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`でSource不変として拒否する。表中のWrite／Preview／解析の意味はActivation審査用の予定意味であり、現在のProject ChangeSet生成、authorization、approvalを与えない。

Validatorはgraph／port／output／limit、unit／range／finite／Quaternion／projection variant、Base RigへのPresentation／Editor Node混入、Stable ID／generation／fallback／default、Camera semantic Registry／Projection／Qualification、Sequence Clock Domain Registry／Profile／consumer kind、viewport、Physics query domain ceiling、blocker filter／Sensor exclusion、comfort constraint、pixel integer scale／rotation／jitter、Target／Quality cost、lock／revisionを固定順で検査する。失敗時はChangeSet全体をrejectし、clamp、unknown Node無視、default生成をしない。

Runtime evaluationはInput Snapshot固定、前回versioned collision result統合、Director選択、Rig parameter／state更新、Base Pose候補と次query生成、Presentation channel／cut event構築、Base state checkpoint、immutable publishの依存順を持つ。具体phase名とlifetimeはRuntime scheduling ownerが決定する。CameraはWorld TransformのWriterにならない。

Collision初期値はsphere radius 0.20 m、skin 0.05 m、最大補正10 m、Sensor除外、`camera_blocker` roleである。resultなし1～2 advancesは前valid、3 advance目は補正なし。owner generation／Physics scene version不一致をdiscardする。

precompiled FailsafeはEngine同梱の固定Rig（`FixedPosition`＋`FixedAim`＋`PerspectiveLens`構成）からCook時に生成するimmutable `CameraPlanV1`であり、`CameraSequencePackageV1`を含む全Cook出力Packageへ必須含有し、Project dataから置換・無効化しない。Default／Failsafe欠落Packageは起動前に拒否する。Rig fallbackは次の一表に固定する。

| 段階 | Trigger | 遷移先 |
|---|---|---|
| 1 | target loss 1 advance、またはnon-finite評価 | last valid poseを維持 |
| 2 | target loss 2 advances継続 | 当該Rigの`fallback_rig_id`、未設定ならDirectorの`default_rig_id` |
| 3 | fallback／default Rigも評価不能、またはnon-finite 3 advances連続 | precompiled Failsafe |

Renderingはprevious／current Base Poseをalpha補間後にPresentation、Lens、Viewport、jitterを合成する。View Impulse／View Noise channelはseed、start step、duration、translation／rotation amplitude、frequencyを持ち、View当たり同時16、translation各軸2 m、rotation各軸20 degree、0～60 Hz、0～30秒を上限とする。Accessibility scaleを最終合成だけへ適用する。

Split Viewはnormalized viewport最大4、各viewが独立Director、Base Pose、Presentation、history key、Input owner、Audio listener bindingを持つ。OverlapはEditor overlayまたは明示compositionだけ。shared Rig inputはimmutable、stateはView別である。

Cinematic cut／blendはDirectorと同じcontractを使い、cut frameのpose、Lens、history reset、markerを同じframe IDへ固定する。`OpenTimelineIO`はinterchange adapter候補であってSource正本ではない。schema upgrade／downgrade、round-trip loss、unknown metadata isolationをqualificationし、media byteを埋め込まない。

## 5. Budget、failure、diagnostic

Camera domain hard capはRig当たりNode 64／Edge 128／Parameter 64、Director当たりRig 16／Transition 64、Gameplay Viewport 4、View Impulse／Noise channel／View 16、Sequence当たりShot 256／Camera binding 64、Subsequence depth 8である。超過ChangeSetを部分commitせず、分割候補とcostを返す。

Gameplay Camera RuntimeはCPU P95 soft 0.25 ms／hard 0.50 ms、memory soft 6 MiB／hard 8 MiB、Simulation Advance中のheap allocation 0、collision request／View／advance 1である。Split Viewも合計hard cap内に収める。これらはCamera固有ceilingであり、共通frame／memory envelopeと測定法はRuntime performance ownerへ委譲する。

Camera command queueはactive Gameplay／Cinematic commandだけを対象とし、soft gateはoccupancyが75%未満、hard gateはoverflow countが正確に0である。観測window、warm-up、run集約は[Runtime performance ownerのreference measurement](../04-runtime/performance-capacity.md#8-measurementregressionpromotion)をそのまま使い、本書へ共通測定値を複写しない。Soft pressureではlow-priority Editor Preview commandの生成を止めてBase Poseを維持し、受理済みGameplay commandをdrop／上書き／次advanceへ暗黙繰越しない。Hard overflowは`MIRAKAN-CAMERA-BUDGET_EXCEEDED`を発行し、該当runとPlay／Package promotionを失敗させる。

Failureはinvalid graph／profileでlive revision不変、Director no-matchでdefault、target loss／collision stale／non-finiteで前節のfallback、View costまたはcommand queue soft pressureでBase Poseを維持してlow-priority Preview停止とする。

Closed Diagnosticは`MIRAKAN-CAMERA-RIG_CYCLE, MIRAKAN-CAMERA-PORT_TYPE_MISMATCH, MIRAKAN-CAMERA-TARGET_UNAVAILABLE, MIRAKAN-CAMERA-COLLISION_STALE, MIRAKAN-CAMERA-EVALUATION_NON_FINITE, MIRAKAN-CAMERA-FAILSAFE_ACTIVATED, MIRAKAN-CAMERA-BUDGET_EXCEEDED, MIRAKAN-CAMERA-UNAUTHORIZED_OPERATION`である。DiagnosticはRequirement ID、Document／Node／Field path、actual、expected、Capability、Target、fallback、remediationを持つ。

## 6. Editorとqualification

WorkspaceはProfile Inspector、Rig Graph、Director／Transition、Sequence／Shot、2D／3D Preview、safe frame／frustum／subject／composition overlay、collision overlay、Base／interpolated／Presentation pose比較、Active Rig／rule／blend／cut／history debugger、AI candidate trace／assumption／cost／Diagnosticを持つ。Widget／canvasをSourceにせず、Previewは同じcompilerとsandbox runtimeを使う。

Contract／runtime fixtureはSchema round-trip、cycle／port／unknown／64・128 boundary／non-finite／unit fuzz、canonical graph hash、Director全組合せ、position／rotation／FOV blend、cut／history、collision generation／stale、target loss／Failsafe、checkpoint hash、Split View isolationを含む。Camera semantic fixtureはinitial Core 32 entryから一意なcanonical Plan hashへ解決するgolden、Genre Pack 0件のneutral fixed-view Project、qualified `project.board_game.overview@1` contributionのpositiveを持つ。unknown ref、unqualified contribution、axis違い、owner／hash stale、Core ID上書き、同一ID別hash、Pack未選択、synonymだけの自動選択を各一原因negativeとしてSource不変で拒否する。Sequence Clock fixtureはGameplay／Cinematicのinitial frame mappingを検証し、unknown／unqualified Domain、Profile非選択、`sequence` consumer不許可、editor preview clockのShipping混入を各一原因で拒否する。Camera queue fixtureは同じactive Sceneへbounded command burstとlow-priority Previewを重ね、reference measurement windowの全sampleでoccupancy 75%未満、overflow 0、受理済みGameplay command drop 0、Base Pose／Replay checkpoint hash不変を検証し、overflow fault injectionが`MIRAKAN-CAMERA-BUDGET_EXCEEDED`とpromotion失敗へなることを確認する。

標準Scene候補は`camera_pixel_2d_c1_v1, camera_follow_2d_c1_v1, camera_third_person_c1_v1, camera_lock_on_c1_v1, camera_boss_group_c1_v1, camera_motion_comfort_c1_v1, camera_split_view_c2_v1, camera_cinematic_c2_v1`である。fixture名のsuffixはhistorical test IDでありmaturityを定義しない。

Composition gateはprimary subject safe-region 99.9%以上、required visibility missing 0 frame、blocker penetration 0、comfort hard violation 0、pixel non-integer／rotation／temporal jitter 0、cut history-reset漏れ0、non-finite View／Projection 0である。Metric、capture、trace、human rubricを同じPreview Receiptへ結ぶ。

AI corpusはtop-down look-ahead、third-person collision／lock-on、reduced view impulse、pixel integer scale、boss group framing、multi-shot cut comparison、unsupported Targetを含む。Gateはunknown Node／Field／planned semantic action token 0、PresentationのGameplay逆流0、unsafe approximation 0、task success 95%以上、不要Blocking質問5%以下、Capability／assumption未報告0である。Evidenceのrun、holdout、gradingはVerification ownerが決定する。
