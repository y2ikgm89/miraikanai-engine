# Miraikanai Engine Physics Contract

- 文書ID: mirakan.arch.simulation-physics
- 状態: review
- 正本範囲: Physics World／Body dynamics、solver profile semantics、command、joint／constraint、generic Kinematic Motion reference Provider、kernel Adapter boundary、Physics save／replay projection、Physics AI intent／discovery／resolution／preview／diagnostic／eval
- 非正本範囲: Collider geometry／filter／query／event、Runtime phase／Simulation Advance／capacity、Animation pose、Navigation artifact、external dependency version／build pin、AI authorization／evidence envelope。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Project state](../03-authoring/project-state.md)、[Gameplay programming model](../03-authoring/gameplay-programming-model.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[Collision](collision.md)、[Navigation](navigation.md)、[Animation](animation.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論とPlatform境界

PhysicsはEngine-owned World、Body、Dynamics command、Joint／Constraint、snapshot、diagnosticを公開し、数値kernelをprivate Adapterへ隔離する。Kinematic Motion ExecutorはCore必須契約ではなく、任意のregistered compositionが選択できるEngine-owned C1 reference Providerである。Project C++、GameplayDefinition、AI、EditorへVendorの型、ID、pointer、callback、setting、serializationを公開しない。採用dependencyとexact version／commit／license／build optionは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)だけが所有する。

[Collision](collision.md)はshape、Collider Asset、material、filter、query、contact／trigger／hit semanticsを所有する。Physicsはそれらを消費してWorldを進めるが再定義しない。[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)はcanonical phase、writer、lease、publishを、[Runtime performance／capacity](../04-runtime/performance-capacity.md)は共通capacity、queue、measurementを所有する。

Module境界は次の意味へ固定する。

| layer | 所有 | 禁止 |
|---|---|---|
| Physics Contracts | Engine value、Port、command、event view、snapshot | Vendor型、native callback |
| Physics Core | World lifecycle、dynamics merge、joint registry、semantic resolver | Vendor include |
| Physics Kinematic Motion Provider | optional executor state、profile、intent／result adapter | Navigation Port再定義、Pack install必須化 |
| Kernel Adapter | Engine valueとnative objectの変換、private table、conformance | World Model、AI、Editorへの依存 |
| Physics Authoring | Source document、validation、preview、ChangeSet projection | live World直接write |
| Physics Editor | Authoring／snapshotのProjection | 独自の正本state |
| Qualification Tool | Backend fixture、comparison、measurement | Shipping Gameへの常時link |

`PhysicsWorldOwner`だけがnative WorldとEngine handle mappingを所有する。native pointer、lock、callback view、collectorはfunction／job／Simulation Advance境界を越えて保持しない。callbackはpreallocated local bufferへcopyするだけで、allocation、logging、World mutation、Gameplay dispatchを行わない。全native jobをjoinし、normalizeが完了した後だけRuntimeへ結果を渡す。

## 2. World、Body dynamics、command

| Object | 意味 | persistence |
|---|---|---|
| `PhysicsWorldDocumentV1` | Authoring上のWorldとProfile参照 | Project source |
| `Physics2DWorldProfileV1` | 2D gravity、solver semantics、worker class、Collision profile ref | Project source |
| `Physics3DWorldProfileV1` | 3D gravity、solver semantics、worker class、Collision profile ref | Project source |
| `PhysicsBody2DComponent`／`PhysicsBody3DComponent` | motion kind、mass source、damping、sleep／motion policy、Collider ref | World source |
| `KinematicMotionProfileV1` | actor collider ref、max slope、step height、ground snap距離、slide／step iteration上限、speed上限 | Project source |
| `PhysicsWorldHandle`／`PhysicsBodyHandle` | Engine generation handle | Runtime only |
| `PhysicsStateSnapshotV1` | normalized transform、velocity、sleep、joint／kinematic-executor state | immutable Simulation Advance snapshot |
| `PhysicsSaveStateV1` | Engine-owned recoverable state | Save stream |

### 2.1 Physics substep Profile

Physicsはsubstepの意味とpartitionだけを所有し、outer Simulation Advance、Cadence選択、phase、publishは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)が所有する。型を次へ固定する。

```text
PhysicsSubstepProfileRefV1
  profile_id: namespace付きStableId
  profile_version: positive uint32
  profile_content_hash: SHA-256

PhysicsSubstepDimensionEntryV1
  dimension: d2 | d3
  substep_count: uint8[1..16]

PhysicsSubstepProfileV1
  profile_id: namespace付きStableId
  profile_version: positive uint32
  interval_requirement: non_null_logical_duration
  partition_policy: equal_rational_partition
  dimension_entries[1..2]: PhysicsSubstepDimensionEntryV1
  profile_content_hash: SHA-256

PhysicsSubstepIntervalInputV1
  cadence_profile_ref: SimulationCadenceProfileRefV1
  simulation_advance_interval_hash: SHA-256
  physics_substep_profile_ref: PhysicsSubstepProfileRefV1
  advance_sequence: positive uint64
  dimension: d2 | d3
  substep_ordinal: positive uint8
  substep_interval_seconds: ReducedPositiveRationalV1

PhysicsSubstepQualificationReceiptRefV1
  qualification_id: StableId
  qualification_version: positive uint32
  qualification_subject_hash: SHA-256
  signed_record_hash: SHA-256

PhysicsSubstepQualificationSubjectV1
  qualification_id: StableId
  qualification_version: positive uint32
  cadence_profile_ref: SimulationCadenceProfileRefV1
  physics_substep_profile_ref: PhysicsSubstepProfileRefV1
  target_profile_ref: exact version/hash-bound Target ref
  fixture_refs[1..64]: exact {fixture_id, fixture_version, fixture_content_hash}
  input_closure_hash: SHA-256
  result: pass | fail
  qualification_subject_hash: SHA-256

PhysicsSubstepQualificationReceiptV1
  subject: PhysicsSubstepQualificationSubjectV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=physics_substep_qualification)

PhysicsSubstepActivationBindingRefV1
  activation_binding_id: StableId
  activation_binding_version: positive uint32
  activation_binding_hash: SHA-256

PhysicsSubstepActivationBindingV1
  activation_binding_id: StableId
  activation_binding_version: positive uint32
  cadence_profile_ref: SimulationCadenceProfileRefV1
  physics_substep_profile_ref: PhysicsSubstepProfileRefV1
  target_profile_ref: exact version/hash-bound Target ref
  qualification_receipt_ref: PhysicsSubstepQualificationReceiptRefV1
  freshness_policy_ref
  activation_binding_hash: SHA-256
```

`profile_content_hash`はASCII `MIRAKAN_PHYSICS_SUBSTEP_PROFILE_V1`と、同Fieldだけを除くclosed recordのRFC 8785 JCS bytesを`uint32_be` length framingしてSHA-256する。Refは完成base recordのID／version／self-excluding content hashからrecord外でmaterializeし、base record自身へ埋め戻さない。`dimension_entries[]`はactive Physics Worldのdimension集合とexact set equality、`d2`、`d3`の順へstrict sortし、duplicate、逆順、選択されていないdimension、count 0または17以上を拒否する。

Qualification subject hashはASCII `MIRAKAN_PHYSICS_SUBSTEP_QUALIFICATION_SUBJECT_V1`、Activation Binding hashはASCII `MIRAKAN_PHYSICS_SUBSTEP_ACTIVATION_BINDING_V1`と各自己hash Fieldだけを除くclosed canonical bytesから計算する。生成順は`Receipt-free PhysicsSubstepProfile → Profile Ref／Cadence Profile → Target別Qualification subject → signed Receipt → root外Activation Binding／Binding Ref → Runtime Package`である。Binding三RefはReceipt subjectのCadence、Substep、Targetとbyte equalityで、Receiptのsubject hash／signed record hash、`result=pass`、freshness／revocationを検証する。Receipt、Binding、Fixture bodyをSubstep ProfileまたはCadence Profileのcontent hashへ埋め戻さず、Production Runtime PackageはProfile RefとBinding Refを別Fieldで保持する。

T50は一つのsealed `SimulationAdvanceIntervalV1`の正のlogical duration `D={numerator,denominator}`を、選択dimension entryの`substep_count = C`に従いordinal `1..C`の各`D/C`へexact rational partitionする。`g=gcd(numerator,C)`、`substep={numerator/g, denominator*(C/g)}`の順でcross-reduceしてからchecked multiplicationし、既約化後の分子／分母が`ReducedPositiveRationalV1`の各`uint32`表現域へ収まらないProfile／Interval組合せはT50前に`physics_substep_interval_overflow`で拒否する。wrap、saturate、float、別rateへの近似を使わない。全`PhysicsSubstepIntervalInputV1`は同じCadence Profile ref、outer interval hash、advance sequence、Substep Profile refへbyte equalityでbindし、ordinal順に実行する。substepは新しいSimulation Advance、Input sample、Timer boundary、event delivery boundaryまたはpublish boundaryを生成せず、T60は全substep完了後にouter advanceにつきexact一回だけ結果を統合する。null duration、dimension entry不足、Profile／Target Qualification不足、Cadence／outer interval／Substep Profile ref不一致ではfallbackせず`cadence_profile_not_qualified`としてT50開始前に拒否する。

current reference `fixed 60/1` Cadenceは`physics_substep_profile_ref=null`かつouter advance当たり一回のPhysics solveだけをProduction Qualification対象とし、current Substep Qualification subject／Receipt／Activation Binding集合はexact `[]`である。non-null Profileは`future.capability.alternate-simulation-cadence-and-substep`がactiveになり、選択する全`{Cadence Profile Ref, PhysicsSubstepProfileRef, Target Profile Ref}`へpassかつfreshな`PhysicsSubstepActivationBindingRefV1`を同じactivation transactionで登録し、Runtime Package、typed Save header、`AuthoritativeReplayHeaderV1`／Domain Replay closureが同じProfile／Binding Refへ閉じるまでplanning-onlyである。Activation前は`cadence_profile_not_qualified`を返し、暗黙のBackend既定substepへ変換しない。

2Dと3Dは別Worldであり、同じEntityへ両dimensionのBodyを付与しない。Body kindは`static | kinematic | dynamic`のclosed enumである。Visual scaleをnative Bodyへ渡さず、Collider geometryは[Collision](collision.md)のCooked Assetに焼き込む。finiteでない値、範囲外のmass／velocity、generation mismatchは明示failureにし、silent clampやnative defaultへのfallbackをしない。

World lifecycleは`uncreated | validating | ready | stepping | stop_requested | draining | destroyed | faulted`のEngine stateで表す。active WorldのProfile、worker class、solver semanticsをlive mutateしない。compatible changeはRuntime boundaryで新generationをactivateし、incompatible changeはPlay停止を要求する。native state名はpublic lifecycleへ露出しない。

### 2.2 Immutable state snapshot

`PhysicsStateSnapshotV1`は名前だけのprojectionではなく、T60でnormalize済みの一World／一Simulation Advanceを表す次のclosed schemaである。

```text
PhysicsBodySnapshotRefV1
  entity_stable_id: StableId
  body_id: StableId
  body_generation: positive uint64

PhysicsWorldProfileRefV1
  dimension: d2 | d3
  profile_id: StableId
  profile_version: positive uint32
  profile_content_hash: SHA-256

PhysicsBodySnapshotStateV1
  body_ref: PhysicsBodySnapshotRefV1
  motion_kind: static | kinematic | dynamic
  state:
    | kind: d2
      position: WorldPosition2f
      rotation: Radians
      linear_velocity: Velocity2f(world)
      angular_velocity_radians_per_second: finite float32
    | kind: d3
      position: WorldPosition3f
      rotation: NormalizedQuaternion
      linear_velocity: Velocity3f(world)
      angular_velocity: AngularVelocity3f(world)
  sleep_state: not_applicable | awake | sleeping

PhysicsJointCoordinateSnapshotV1
  coordinate_ordinal: uint16
  state:
    | kind: linear
      position_m: finite float32
      velocity_m_per_second: finite float32
    | kind: angular
      position_radians: finite float32
      velocity_radians_per_second: finite float32
  limit_state: inactive | lower | upper
  motor_state: disabled | enabled

PhysicsJointSnapshotStateV1
  joint_id: StableId
  joint_generation: positive uint64
  joint_kind:
    d2_distance | d2_revolute | d2_prismatic | d2_weld |
    d3_fixed | d3_point | d3_distance | d3_hinge |
    d3_slider | d3_swing_twist
  body_a_ref: PhysicsBodySnapshotRefV1
  body_b:
    | kind: body
      body_ref: PhysicsBodySnapshotRefV1
    | kind: world_anchor
  lifecycle_state: active | disabled | broken_pending_removal
  coordinate_states[0..6]: PhysicsJointCoordinateSnapshotV1

PhysicsKinematicExecutorSnapshotStateV1
  actor_body_ref: PhysicsBodySnapshotRefV1
  provider_record_ref: MotionExecutorProviderRecordRefV1
  provider_activation_binding_ref:
    MotionExecutorProviderActivationBindingRefV1
  provider_generation: positive uint64
  motion_profile_ref:
    exact {profile_id, profile_version, profile_content_hash}
  motion_state: grounded | sliding | airborne
  resolved_state:
    | kind: d2
      position: WorldPosition2f
      rotation: Radians
      linear_velocity: Velocity2f(world)
      ground_attachment:
        null |
        {body_ref: PhysicsBodySnapshotRefV1,
         local_contact: LocalPosition2f,
         world_normal: UnitDirection2f,
         platform_displacement: Displacement2f(world)}
    | kind: d3
      position: WorldPosition3f
      rotation: NormalizedQuaternion
      linear_velocity: Velocity3f(world)
      ground_attachment:
        null |
        {body_ref: PhysicsBodySnapshotRefV1,
         local_contact: LocalPosition3f,
         world_normal: UnitDirection3f,
         platform_displacement: Displacement3f(world)}
  source_intent_batch_hash: SHA-256
  resolved_motion_generation: positive uint64

PhysicsStateSnapshotV1
  schema_version: 1
  cadence_profile_ref: SimulationCadenceProfileRefV1
  simulation_advance_interval_ref: SimulationAdvanceIntervalRefV1
  simulation_advance_interval_sha256: SHA-256
  advance_sequence: positive uint64
  physics_substep_profile_ref: null | PhysicsSubstepProfileRefV1
  physics_substep_activation_binding_ref:
    null | PhysicsSubstepActivationBindingRefV1
  world_id: namespace付きStableId
  world_dimension: d2 | d3
  world_generation: positive uint64
  physics_world_profile_ref: PhysicsWorldProfileRefV1
  physics_world_profile_activation_generation: positive uint64
  body_states[0..65536]: PhysicsBodySnapshotStateV1
  joint_states[0..65536]: PhysicsJointSnapshotStateV1
  kinematic_executor_states[0..65536]:
    PhysicsKinematicExecutorSnapshotStateV1
  snapshot_content_hash: SHA-256

PhysicsStateSnapshotRefV1
  cadence_profile_ref: SimulationCadenceProfileRefV1
  simulation_advance_interval_ref: SimulationAdvanceIntervalRefV1
  world_id: namespace付きStableId
  world_generation: positive uint64
  physics_world_profile_ref: PhysicsWorldProfileRefV1
  physics_world_profile_activation_generation: positive uint64
  snapshot_content_hash: SHA-256
```

snapshotはT60で全native jobをjoinした後、生成元のcompleted `SimulationAdvanceIntervalV1`からだけ構築する。`cadence_profile_ref`、`simulation_advance_interval_ref.cadence_profile_ref`、生成元Intervalの同Fieldはbyte equality、`advance_sequence`はRefと生成元Intervalの同Fieldへbyte equalityである。Refの`interval_content_hash`は生成元Intervalのself-excluding semantic `interval_content_hash`と一致させ、隣接する`simulation_advance_interval_sha256`はそのhash Fieldを含むcompleted Interval全canonical bytesのSHA-256と一致させる。semantic content hashとcompleted-object SHAを相互代用しない。World／World Profileのidentityとactivation generation、Substep Profile／BindingはそのIntervalを実行したactive runtime selectionと一致させ、`physics_world_profile_ref.dimension`は`world_dimension`と同じにする。T50途中のstate、別World、別Profile generation、別advanceの値を混在させない。

bodyは`body_ref.entity_stable_id, body_ref.body_id, body_ref.body_generation`、jointは`joint_id, joint_generation`、kinematic executorは`actor_body_ref`のcanonical byte順へstrict sortする。joint coordinateは`coordinate_ordinal`昇順で、ordinal集合は解決したjoint family schemaとexact set equalityにする。duplicate、逆順、wrong-dimension variant、存在しないBody ref、generation不一致、同一Bodyの2D／3D重複を拒否する。各arrayの上限を超える場合はsnapshotを切り詰めず、`physics_snapshot_capacity_exceeded`で当該authoritative advanceのpublishをfaultする。staticはzero velocityかつ`sleep_state=not_applicable`、kinematicも`sleep_state=not_applicable`、dynamicだけが`awake | sleeping`を持つ。全数値は[Math core](../02-foundation/math-core.md)のSI単位、finite、normalization、canonical quantizationを通し、native padding、pointer、solver island、cache、worker順、Backend enumを含めない。

`snapshot_content_hash`はASCII `MIRAKAN_PHYSICS_STATE_SNAPSHOT_V1`と同Fieldだけを除く上記closed recordのMCD canonical bytesを各`uint32_be` length framingしてSHA-256する。Refは完成snapshotからrecord外でmaterializeし、snapshot自身へ埋め戻さない。consumerは要求時にexpected `PhysicsStateSnapshotRefV1`またはその全Fieldと等価なtyped expectationを渡し、Interval Ref、advance sequence、World generation、World Profile ref／activation generation、snapshot hashの一件でも異なるsnapshotを`physics_snapshot_stale`で副作用前に拒否する。次advanceで直前snapshotを読むpipelineは直前のexact Refをexpected値として明示し、`latest`、ID-only lookup、current generationへの再解決、cross-advance rebaseを行わない。

### 2.3 Dynamics command

`PhysicsDynamicsCommandV1`はtarget handle／expected generation、`cadence_profile_ref`、`simulation_advance_interval_hash`、`consume_advance_sequence`、producer metadata、priority、tagged payloadを持つ。payloadは次のclosed command familyである。

- force／torque／impulse
- bounded linear／angular velocity assignment
- kinematic target／teleport
- wake／sleep permission／gravity factor
- joint motor target／limit／break request

同一Bodyへのcommandは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)のcanonical merge keyを消費する。Teleportと通常motion commandの競合、generation不一致、wrong body kind、wrong dimensionはtyped conflictとして全体を拒否する。Force、Impulse、velocityを暗黙変換しない。command arena、queue、overflow値をPhysicsで持たない。

Physics executionはRuntimeのcanonical identifiers `T30_PrePhysics`、`T40_MotionIntent`、`T50_PhysicsStep`、`T60_PhysicsIntegrate`、`T70_PostPhysics`への参照で接続する。正確な順序、writer、Simulation Cadenceは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md#4-simulation-cadenceとphase-identifier)だけが決定し、本書はphase tableを再掲しない。

## 3. Joint、Constraint、optional Physics Kinematic Motion Provider

`PhysicsJointCommonV1`はjoint Stable ID、World ref、Body A／Bまたはtyped World Anchor、enabled、collide-connected、local frame、optional break semanticsを持つ。Joint kindはtagged unionであり、存在しないfieldをproperty bagへ入れない。World Anchorは専用variantで、null Bodyやmagic handleで代用しない。

2D C1 familyはdistance、revolute、prismatic、weld、3D C1 familyはfixed、point、distance、hinge、slider、swing-twistを持つ。各familyはSI単位、normalized axis、orthogonal frame、ordered limit、finite motor targetを検証する。Vendor enum値、constraint pointer、reaction callbackは保存しない。新familyはCapability、schema、Editor、AI vocabulary、fixtureを同時に追加する。

Joint break候補はAdapter結果をSI単位へnormalizeし、Engine Stable ID、`cadence_profile_ref`、`simulation_advance_interval_hash`、`advance_sequence`を持つ`JointBreakEventV1`へ変換する。三Fieldはbreak候補を生成したcanonical `SimulationAdvanceIntervalV1`の同値とbyte equalityにする。配送と次boundaryのcomponent removalはRuntime ownerの順序を消費する。Backend間のreaction値へbitwise一致は要求せず、fixtureで許容されるsemantic rangeを検査する。

### 3.1 Kinematic Motion reference Provider

C1 reference recipeはEngine-owned Kinematic Motion Providerを適格化するが、任意Packのinstall、Path Following、Runtime EntryはこのProviderを要求しない。Backend固有controllerをProject APIへ公開せず、[Collision](collision.md)のoverlap／shape castだけを利用する。

Provider-private `KinematicMoveIntentV1`はactor handle、consume advance sequence、Cadence Profile ref、Simulation Advance interval hash、planar displacement、vertical proposal、edge-triggered motion flags、up direction、producer metadataを持つ。これはaccepted public intentではなく、owner adapterがNavigation `MovementIntentV1`または`MotionIntentContributionV1`を検証してexact `AdaptedMotionIntentV1`へ変換し、generic resolverが封印したbatchをPhysics Providerが受理した後にだけ生成するderived inputである。raw二型はadapter source metadataでありProvider accepted setへ入れない。Port、Project Source、Save、Replayへprivate型参照を公開せず、producer固有proposalを内部Fieldへ複写しない。`PhysicsKinematicResolvedMotionV1`はresolved pose／velocity、state、ground handle／generation／normal／relative point、platform delta、hit summary、diagnostic、input batch hash、generationを持つ。

`capability.motion_executor.physics_kinematic`はPhysics Providerが提供する正式Capabilityであり、次のexact 7-Field descriptorを[Navigation](navigation.md)が所有する`MotionExecutorProviderCatalogV1`へproduction recordとして登録する。Port型、transport batch、contribution registry、Provider Catalogを本書で再定義しない。全MCD参照は表のID、`version=1`、選択Contract set hashを持つ`McdContractRefV1`である。

| `executor_capability_ref.id` | `movement_profile_schema_ref.id` | `accepted_intent_schema_refs[].id` | `transport_message_schema_ref.id` | `resolved_motion_schema_ref.id` | `compatibility_predicate_ref.id` | `failure_diagnostic_refs[]` |
|---|---|---|---|---|---|---|
| `capability.motion_executor.physics_kinematic` | `type.physics.kinematic_motion_profile` | `[type.navigation.adapted_motion_intent]` | `type.navigation.motion_executor_intent_batch` | `type.physics.kinematic_resolved_motion` | `policy.physics.kinematic_motion_intent_profile_target_dimension` | `[MIRAKAN-PHYSICS-KINEMATIC-MOTION-INCOMPATIBLE, MIRAKAN-PHYSICS-KINEMATIC-MOTION-RESOLUTION_FAILED, MIRAKAN-PHYSICS-KINEMATIC-MOTION-STALE_RESULT]` |

#### 3.1.1 Production implementation System contract

production Providerの実装先は未指定の「Physics System」ではなく、exact `game_system.engine.physics.kinematic_motion_executor` v1である。これはNavigation Selection／batch ownershipやPhysics authoritative Stateを持たず、検証済みbatchとProfile／Collision snapshotからresolved motion Portを導出するReceipt-free `GameSystemSpecV2`である。次のrecordは[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)が所有するcanonical schemaのPhysics具体instanceであり、schemaの再定義ではない。全`@1`は同じContract set rootを持つexact MCD refの表示短記であり、ID文字列へ`@1`を含めない。

```text
GameSystemSpecV2
  MCD common envelope: all fields
  id: game_system.engine.physics.kinematic_motion_executor
  version: 1
  status: active
  title: Physics Kinematic Motion Executor
  description:
    resolve a validated Navigation motion-intent batch through the
    bounded Physics kinematic reference implementation
  owners: [owner.core.physics]
  requirement_refs:
    [requirement.physics.kinematic_motion_executor.resolve_batch@1,
     requirement.physics.kinematic_motion_executor.no_navigation_authority@1]
  rationale_refs: [mirakan.arch.simulation-physics#31-kinematic-motion-reference-provider]
  since_contract_set: 2
  supersedes: []
  tags: [kinematic_motion, motion_executor, physics, provider]
  owner_layer: core
  owner_ref:
    {owner_layer=core,owner_id=owner.core.physics,
     owner_revision,owner_content_hash}
  system_origin: engine_standard
  semantic_role_refs:
    [{id=semantic_role.physics.kinematic_motion_executor,
      version=1,content_hash}]
  responsibility_requirement_refs:
    [requirement.physics.kinematic_motion_executor.resolve_batch@1]
  non_responsibility_requirement_refs:
    [requirement.physics.kinematic_motion_executor.no_navigation_authority@1]
  runtime_scope_type_ref:
    {scope_type_id=scope.core.world,scope_type_version=1,scope_type_hash}
  state_class: derived
  owned_state_type_refs: []
  read_snapshot_type_refs:
    [type.physics.kinematic_motion_profile@1]
  accepted_command_type_refs: []
  emitted_event_type_refs: []
  emitted_port_message_type_refs:
    [type.physics.kinematic_resolved_motion@1]
  provided_capability_refs:
    [capability.motion_executor.physics_kinematic@1]
  required_capability_refs:
    [capability.simulation.collision_query@1]
  allowed_phase_ids: [T40_MotionIntent]
  dependency_edge_refs:
    [{id=dependency.physics.kinematic_motion_executor.navigation_batch,
      version=1,content_hash},
     {id=dependency.physics.kinematic_motion_executor.collision_query,
      version=1,content_hash}]
  implementation_policy_ref:
    {id=implementation_policy.physics.kinematic_motion_executor,
     version=1,content_hash}
  save_replay_contract_ref: canonical omission
  behavior_budget_refs:
    [{id=budget.physics.kinematic_motion_executor,
      version=1,content_hash}]
  authoring_surface_ids: []
  fallback_contract:
    no_fallback(reason=invalid or unavailable resolver publishes no new result)
  compatibility_invariant_refs:
    [{id=invariant.physics.kinematic_motion_executor.adapted_input_only,
      version=1,content_hash},
     {id=invariant.physics.kinematic_motion_executor.single_resolution,
      version=1,content_hash},
     {id=invariant.physics.kinematic_motion_executor.no_navigation_authority,
      version=1,content_hash}]
  auxiliary_ref_set_hash:
    SHA-256(MIRAKAN_GAME_SYSTEM_AUXILIARY_REF_SET_V1,
      exact sorted auxiliary refs above,self excluded)
  extension_policy: sealed
```

Specの10件の補助参照は次のactive inventoryへexactly oneで解決する。二RequirementはMCD、残る8件は`id`、`version=1`、`content_hash`付きtyped receipt-free recordである。

| exact record | record type | normative content |
|---|---|---|
| `semantic_role.physics.kinematic_motion_executor` v1 | `SemanticRoleRecordV1` | validated batchを受理し、bounded Physics resolutionをexact一回実行する |
| `requirement.physics.kinematic_motion_executor.resolve_batch` v1 | MCD `requirement` | adapted intent、Profile、Target、dimension、Collision closureを検証してresolved motionを返す`must` |
| `requirement.physics.kinematic_motion_executor.no_navigation_authority` v1 | MCD `requirement` | Selection、binding、canonical batchを所有・変更・再発行しない`must_not` |
| `dependency.physics.kinematic_motion_executor.navigation_batch` v1 | `GameSystemDependencyEdgeV1` | Navigation batch Portのexact source dependency |
| `dependency.physics.kinematic_motion_executor.collision_query` v1 | `GameSystemDependencyEdgeV1` | normalized Collision query capabilityのexact dependency |
| `implementation_policy.physics.kinematic_motion_executor` v1 | `GameSystemImplementationPolicyV1` | sealed Engine Core C++ reference implementation、live switch禁止 |
| `budget.physics.kinematic_motion_executor` v1 | `BehaviorBudgetRecordV1` | active Targetごとのbatch／entry／profile iteration bound |
| `invariant.physics.kinematic_motion_executor.adapted_input_only` v1 | `CompatibilityInvariantRecordV1` | accepted setはadapted intent exact一型 |
| `invariant.physics.kinematic_motion_executor.single_resolution` v1 | `CompatibilityInvariantRecordV1` | actor／generation／advance当たりProvider callとresultは最大一件 |
| `invariant.physics.kinematic_motion_executor.no_navigation_authority` v1 | `CompatibilityInvariantRecordV1` | Selection／binding writeとNavigation batch emissionは0件 |

非MCD recordの共通headerは`{record_id,record_version=1,record_content_hash,owner_ref={owner_layer=core,owner_id=owner.core.physics,owner_revision,owner_content_hash},status=active,introduced_contract_set_local_ref}`である。MCD edgeはroot前に`ContractSetLocalRefV1`、root後のprojectionだけが同root付きexternal refを使う。各hashはASCII `MIRAKAN_PHYSICS_GAME_SYSTEM_AUXILIARY_RECORD_V1`、record type ordinal、自己hashを除くheaderとtyped payloadのMCD canonical bytesをcount／length frameして計算し、System／Provider Qualification Receipt、Activation Binding、Provider Catalog hashを含めない。

```text
SemanticRoleRecordV1.payload
  accepted_port_message_type_local_refs:
    [type.navigation.motion_executor_intent_batch v1]
  read_snapshot_type_local_refs:
    [type.physics.kinematic_motion_profile v1]
  emitted_port_message_type_local_refs:
    [type.physics.kinematic_resolved_motion v1]
  allowed_phase_ids: [T40_MotionIntent]
  authoritative_write_type_local_refs: []

GameSystemDependencyEdgeV1.payload
  source_system_local_ref:
    {kind=game_system,
     id=game_system.engine.physics.kinematic_motion_executor,
     version=1}
  dependency_kind: port_source | capability
  target_contract_local_ref:
    port_source:
      {kind=type,
       id=type.navigation.motion_executor_intent_batch,version=1}
    | capability:
      {kind=capability,
       id=capability.simulation.collision_query,version=1}
  phase_relation: available_at_t40
  access: read_only
  required: true
  fallback: no_inferred_dependency

GameSystemImplementationPolicyV1.payload
  allowed_implementation_kinds: [engine_core_cpp]
  default_implementation_artifact_ref:
    {artifact_kind=engine_core_module,
     logical_id=implementation.engine.physics.kinematic_motion_executor,
     schema_version=1,sha256}
  native_eligibility: true
  replacement_policy: sealed
  live_switch_policy: forbidden
  required_target_refs: exact Provider supported Target set
  configuration_schema_local_ref:
    {kind=type,id=type.game_system.empty_configuration,version=1}
  unavailable_behavior: reject_provider_activation_keep_last_valid

BehaviorBudgetRecordV1.payload
  target_limits[1..64]:
    target_profile_ref: exact Provider supported Target ref/version/hash
    max_batches_per_actor_generation_advance: 1
    max_entries_per_batch: 16
    max_inline_payload_bytes_per_entry: 65536
    max_slide_iterations:
      exact upper bound from type.physics.kinematic_motion_profile
    max_step_iterations:
      exact upper bound from type.physics.kinematic_motion_profile
  overflow_behavior: typed_reject_no_partial_resolution

CompatibilityInvariantRecordV1.payload
  predicate_kind:
    adapted_intent_schema_exact_singleton
    | provider_call_and_result_at_most_once
    | navigation_authority_edge_count_zero
  evaluation_phase: activation | T40_MotionIntent
  input_contract_local_refs[1..8]
  expected: true
  failure_code: MIRAKAN-PHYSICS-KINEMATIC-MOTION-INCOMPATIBLE
  failure_behavior: reject_without_publishing_partial_result
```

二dependency recordはpayload unionの各branchを一件ずつ使う。Role、Policy、Budget、三Invariantは表のIDとpredicate branchを一対一にする。Policyの`configuration_schema_local_ref`はGameplay Programming Model所有のcomplete `type.game_system.empty_configuration` v1 LocalRefであり、同じSnapshot root確定後だけexternal refへ投影する。Budget Target行はProvider base recordの`supported_target_profile_refs[]`とexact set equality、Target ref canonical順、duplicateなしとする。Profileからiteration値を読むがTarget default、Backend default、unbounded sentinelを補完しない。

二Requirementは次の全MCD共通EnvelopeとRequirement payloadを持つ。

```text
mcd_version=1
kind=requirement
id=requirement.physics.kinematic_motion_executor.resolve_batch
version=1
status=active
title=Resolve Validated Kinematic Motion Batch
description=Produce a bounded Physics resolved-motion result from one validated Navigation batch
owners=[owner.core.physics]
requirement_refs=[]
rationale_refs=[mirakan.arch.simulation-physics#31-kinematic-motion-reference-provider]
since_contract_set=2
supersedes=[]
tags=[kinematic_motion,motion_executor,physics]
normative_level=must
priority=blocking
statement=Each accepted batch is validated against adapted intent Profile Target dimension and Collision closure before at most one resolved result is emitted
scope=[game_system.engine.physics.kinematic_motion_executor,T40_MotionIntent]
verification_methods=[gate.physics.kinematic_motion_executor.contract,gate.physics.kinematic_motion_executor.runtime]
acceptance_criteria=[predicate.physics.accepted_schema_exact,predicate.physics.provider_call_at_most_once,predicate.physics.resolved_result_bounded]
failure_code=MIRAKAN-PHYSICS-KINEMATIC-MOTION-INCOMPATIBLE
source_refs=[{ref=mirakan.arch.simulation-physics#31-kinematic-motion-reference-provider,authority=project_normative}]
introduced_by=changeset.architecture.physics.kinematic_motion_executor.v1

mcd_version=1
kind=requirement
id=requirement.physics.kinematic_motion_executor.no_navigation_authority
version=1
status=active
title=Do Not Own Navigation Selection Or Batch Publication
description=Keep the Physics Provider downstream of Navigation selection binding and canonical publication
owners=[owner.core.physics]
requirement_refs=[]
rationale_refs=[mirakan.arch.simulation-physics#31-kinematic-motion-reference-provider]
since_contract_set=2
supersedes=[]
tags=[authority,boundary,motion_executor,physics]
normative_level=must_not
priority=blocking
statement=The Physics executor must not select a Provider mutate Navigation bindings or publish a Navigation batch
scope=[game_system.engine.physics.kinematic_motion_executor,T40_MotionIntent]
verification_methods=[gate.physics.kinematic_motion_executor.contract,gate.physics.kinematic_motion_executor.authority]
acceptance_criteria=[predicate.physics.navigation_selection_write_edge_count_zero,predicate.physics.navigation_batch_emission_count_zero]
failure_code=MIRAKAN-PHYSICS-KINEMATIC-MOTION-INCOMPATIBLE
source_refs=[{ref=mirakan.arch.simulation-physics#31-kinematic-motion-reference-provider,authority=project_normative}]
introduced_by=changeset.architecture.physics.kinematic_motion_executor.v1
```

Contract compilerは二RequirementとSpecを同じ`ContractSetSnapshotV2`のMCD local memberとして含め、全MCD edgeをlocal identityで解決し、8件のtyped auxiliary record hashと10 refのexact `GameSystemAuxiliaryRefSetV1`をSpec local payloadへ含めてからmember hash／set rootを計算する。root確定後にだけ次のSystem ref、Qualification、signed Receipt、Activation Bindingを順にmaterializeする。後三recordは[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)が所有するcanonical `GameSystemQualificationSubjectV1`、`GameSystemQualificationReceiptV1`、`GameSystemActivationBindingV1` schemaのPhysics具体instanceであり、schemaの再定義ではない。

```text
GameSystemContractRefV1
  id: game_system.engine.physics.kinematic_motion_executor
  version: 1
  contract_set_hash: exact compiled Contract set root

GameSystemQualificationSubjectV1
  qualification_id:
    qualification.game_system.physics.kinematic_motion_executor
  qualification_version: 1
  owner_ref:
    exact {owner_layer=core,owner_id=owner.core.physics,
           owner_revision,owner_content_hash}
  system_ref:
    exact game_system.engine.physics.kinematic_motion_executor v1 ref above
  system_contract_hash:
    exact self-excluding resolved GameSystemSpecV2 contract hash
  target_profile_refs:
    exact Provider supported Target Profile set
  fixture_refs[1]:
    {fixture_id=fixture.navigation.motion-executor.physics-kinematic,
     fixture_version=1,fixture_content_hash}
  input_closure_hash:
    SHA-256(MIRAKAN_PHYSICS_KINEMATIC_SYSTEM_QUALIFICATION_INPUT_V1,
      exact System ref/contract hash,auxiliary ref-set hash,
      Provider port descriptor,Target refs,fixture ref;
      count/length framed)
  result: pass
  qualification_subject_hash: SHA-256

GameSystemQualificationReceiptV1
  subject: exact completed subject above
  signed_record:
    exact MirakanSignedRecordV1(
      purpose=game_system_qualification,
      subject_sha256=SHA-256(JCS(completed subject)))

GameSystemActivationBindingV1
  activation_binding_id:
    activation.game_system.physics.kinematic_motion_executor
  activation_binding_version: 1
  system_ref: exact subject.system_ref
  system_contract_hash: exact subject.system_contract_hash
  qualification_receipt_refs[1]:
    {qualification_id=subject.qualification_id,
     qualification_version=subject.qualification_version,
     qualification_subject_hash=subject.qualification_subject_hash,
     signed_record_hash=SHA-256(JCS(completed signed_record))}
  activation_binding_hash:
    SHA-256(MIRAKAN_GAME_SYSTEM_ACTIVATION_BINDING_V1,
      self-excluding canonical binding bytes)
```

production base recordは`provider_id=provider.engine.physics.kinematic_motion`、`provider_version=1`、self-excluding content hash、`owner_identity={owner_layer=core,core_system_owner_ref=上記Specのexact owner_ref,他branch=null}`、`usage=production`、`UsageTaggedImplementationSystemBaseRefV1(usage=production, production_system_ref={id=game_system.engine.physics.kinematic_motion_executor,version=1,contract_set_hash}, production_system_contract_hash=exact resolved System contract hash)`、Target Profile集合を持ち、System／Provider Qualification ReceiptまたはActivation Bindingを含めない。base record／Catalog／RecordRefを固定した後、Provider Qualification subjectだけが同じSystem base refに`production_system_activation_binding_ref={activation_binding_id=activation.game_system.physics.kinematic_motion_executor,activation_binding_version=1,activation_binding_hash}`を加えた`UsageTaggedImplementationSystemRefV1`を持つ。Providerの`owner_identity.core_system_owner_ref`、System refが解決するSpec owner、`ProviderProductionOwnerProjectionV1.game_system_owner_ref`は全Field byte equalityである。Compile／Activation／Batch／Save／Replayが使用するidentityは次のNavigation-owned RecordRef、そのRecordRefをsubjectにするexact Provider Activation Binding ref、Provider subjectが固定したこのSystem Activation Binding refの組である。次のrecordは[Navigation](navigation.md)が所有するcanonical `MotionExecutorProviderRecordRefV1` schemaのPhysics具体instanceであり、schemaの再定義ではない。

```text
MotionExecutorProviderRecordRefV1
  catalog_ref:
    catalog_id=motion_executor.provider_catalog.active
    catalog_version=exact compiled version
    catalog_hash=exact compiled catalog hash
    contract_set_hash=exact compiled Contract set hash
  provider_id=provider.engine.physics.kinematic_motion
  provider_version=1
  provider_content_hash=exact production record content hash
```

policyはaccepted setがexact `[type.navigation.adapted_motion_intent@1]`、Profile schema／hash、Target Profile、2D／3D dimension、Collision query availabilityであることを検証する。このsingletonへの変更によりreceipt-free `provider.engine.physics.kinematic_motion@1` content hash、Provider Catalog hash、RecordRefを先に更新し、その固定RecordRefと既存System Activation BindingからProvider Qualification subject／input closure／signed Receipt／Provider Activation Binding、Compile／Activation／Save／Replay closureを順に更新する。System／Provider Receipt／Activation BindingをProvider base recordまたはCatalog hashへ戻さない。owner固有proposalはNavigationのgeneric contribution envelopeとexact adapter recordを介し、Physics recordへraw source typeまたはowner固有type IDを追加しない。stale Catalog ref、stale provider content hash、System base／Activation Binding mismatch、fixture RecordRefへの置換をproduction qualificationで個別に拒否する。

T40のgeneric contribution resolverはNavigationとowner登録済みproposalをadapterでexact `AdaptedMotionIntentV1`へ変換し、Navigation-owned `MotionExecutorIntentBatchV1` Port messageとして提出する。選択済みPhysics Kinematic Motion Providerは全entryの`adapted_intent_schema_ref`がexact `type.navigation.adapted_motion_intent@1`であることを一度だけ検証する。`source_schema_ref`はprovenance metadataでありaccepted schema判定に使わない。全proposalはregistryで一意に選んだadapterを経由し、Providerへ直接提出しない。Provider-private `KinematicMoveIntentV1`はこの検証済みbatchからだけderiveし、Portのpublic accepted setへ混ぜない。`MovementIntentV1.motion_request.kind=velocity`は同じ[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)の`SimulationAdvanceIntervalV1.interval.logical_duration_seconds`がnon-nullの場合だけ、そのexact rational durationを乗算してplanar displacementへ変換する。`displacement_per_advance`は値をそのまま使い、duration-null turn-based／explicit-stepを0秒または`1/60`秒へ変換しない。Cadence Profile ref、interval hash、advance sequence、Provider Qualificationの一件でも不一致ならT50前に拒否する。同一advanceの複数proposalは各adapter policyとcanonical producer順で解決し、不採用のintentをtyped resultとしてproducerへ返す。合成優先順位をProvider内で暗黙に変更しない。

Executorのmax slope、step height、ground snap距離、iteration上限、speed上限は§2の`KinematicMotionProfileV1`だけが保持し、stage 1が検証するProfileはこのProfileである。`NavAgentProfileV1`のslope／climbとの整合検証は[Navigation](navigation.md)のrequest validationが所有する。

Kinematic resolverは次のsemantic stagesを固定する。

1. Intent、generation、Profile、finite、speed、World versionを検証する。
2. 前snapshotのground attachmentをgeneration付きで再検査する。
3. current overlapをcanonical hit orderで解消する。
4. planar sweepとslideをbounded iterationで解決する。
5. walkable obstacleだけにstep candidateを評価する。
6. vertical motion、jump、slope、ground snapを解決する。
7. final overlapとhard invariantを再検査し、kinematic targetを提出する。

tie-breakは[Collision](collision.md)のnormalized query orderingを使い、native callback順を使わない。Moving platform attachmentはEngine handle、generation、local contact pointだけを保存する。Platform teleport／destroy／generation changeではattachmentを切る。

Root motionは[Animation](animation.md)からgeneric contribution resolverを経由してselected Motion Executorへ届くproposalであり、本Provider選択時はregistered adapter policyと`proposal_only | root_motion_only | additive_bounded`のProvider policyで合成する。Providerのresolved motionがauthoritativeで、Animationはそれを読む。PhysicsとAnimationがTransformへ二重writeしない。

## 4. Save、Replay、failure、qualification

```text
PhysicsColliderArtifactSaveRefV1
  artifact_ref: ArtifactRefV1
  source_asset_id: StableId
  source_asset_revision: uint64
  source_asset_content_hash: SHA-256

PhysicsSaveStateV1
  projection_id: StableId
  projection_version: positive uint32
  schema_version: 1
  physics_state_snapshot: PhysicsStateSnapshotV1
  collider_asset_refs[0..65536]:
    PhysicsColliderArtifactSaveRefV1
  projection_content_hash: SHA-256
```

SaveはEngine-owned World／Body／Joint／kinematic executor stateを§2.2の一つのcompleted `PhysicsStateSnapshotV1`として保存し、native serialization、pointer、Backend substep state、未commitのT50途中state、snapshotと並行する別state arrayを保存しない。`projection_content_hash`はASCII `MIRAKAN_PHYSICS_SAVE_STATE_V1`と同Fieldだけを除くclosed canonical bytesから計算する。完成Projectionは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)のroot外`AuthoritativeSaveDomainBindingV1`によってexact completed Headerへ結ばれ、Headerのstate-owner Projection Ref集合に本Projectionのowner／Type／ID／version／content hashがexact一件存在しなければならない。Header／BindingをProjectionへ埋め戻さない。snapshotの`cadence_profile_ref`はHeaderの`simulation_cadence_profile_ref`、`simulation_advance_interval_ref`は`last_committed_simulation_advance_interval_ref`、`simulation_advance_interval_sha256`は`last_committed_simulation_advance_interval_sha256`、`physics_substep_activation_binding_ref`はHeaderの同Fieldとそれぞれbyte equalityにする。snapshotのSubstep Profile RefはBinding subjectのSubstep Refと一致させる。null branchではsnapshotのSubstep Profile／Bindingがともにnullで、outer advance一回solveの契約と一致しなければならない。

`collider_asset_refs[]`は`source_asset_id, source_asset_revision, source_asset_content_hash, artifact_ref`のcanonical byte順へstrict sortし、duplicateを拒否する。各`artifact_ref`はexact completed `CookedColliderAssetV1`へ解決し、解決済みartifactのSource Asset identity三Fieldを隣接三Fieldとbyte equalityにする。集合はsnapshotのWorld generationを復元するためactive World definitionが参照するCooked Collider Asset集合とexact set equalityにし、unused／missing ref、path、`latest`、別revision／content hashへの再解決を拒否する。65,537件目を切り詰めず`physics_save_capacity_exceeded`でSaveを不成立にする。

Loadはschema、toolchain lock compatibility、snapshot content hash、World／Substep Profile、Cadence、last Interval semantic hash／completed SHA、Collider Asset identity、finite value、generation relationを検証してstaging Worldを構築し、fixture validation後にcompatible Simulation Advance boundaryで置換する。Header／Profile／Interval／Binding／Projection／snapshot不一致をcurrent値、Backend default、fixed 60/1へ読み替えず、明示migrationまたは`physics_save_incompatible`で拒否する。失敗時はactive Worldを維持する。

ReplayはRuntime ownerへnormalized command、accepted async input、Profile／artifact identity、state hash、snapshot projectionを供給する。Physicsのreceipt-free Replay Projectionは[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)の`AuthoritativeReplayDomainProjectionRefV1`としてHeaderのProjection集合へexact一件登録し、Header Ref／Binding RefをProjection base recordへ埋め込まない。Header完成後だけroot外`AuthoritativeReplayDomainBindingV1`がexact `AuthoritativeReplayHeaderRefV1`とPhysics Projection Refを結び、HeaderのCadence／Substep Binding、該当`SimulationAdvanceIntervalRefV1`／completed SHA／sequenceとProjectionをbyte equalityにする。生成順を`receipt-free Physics Projection → Projection set／Header／Ref → Domain Binding`に固定し、Header／BindingをProjection hashへ戻さない。記録slotは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)のcanonical `T100_ReplayCheckpoint`だけを参照し、Physics固有phaseを設けない。

主要failure classはinvalid profile、unqualified adapter、handle／generation mismatch、command conflict、joint frame invalid、kinematic depenetration failure、native invariant violation、job drain failure、save incompatibilityである。Simulation Advance publish、fault transition、recovery boundaryはRuntime ownerへ委譲する。共通memory、worker、queue、frame thresholdをここで再定義しない。

Qualificationは全private Backendへ同じWorld lifecycle、stack、sleep／wake、joint、break、C1 reference Providerのslope／stair／kinematic support surface（斜面際のstep、ceilingに接した状態のslide、移動support bodyからの離脱、狭所でのdepenetration発振を含む）、save／load、replay hash、fuzz、fault injectionを与える。Engine contractの結果、ordering、diagnostic、lifetimeが一致することを検査する。Substep fixtureはnull Profileでouter advance当たりsolve一回、count 2／3／16で全exact rational subintervalの和がouter durationと一致しordinal順、T60／publishがouter advance当たり一回であることを検査する。`denominator*(C/g)`が`uint32`最大値ちょうどに収まる境界と1超過する境界を含め、後者だけを`physics_substep_interval_overflow`で副作用前に拒否する。自己Ref埋込み、dimension欠落／duplicate／逆順、count 0／17、null duration、Cadence／interval hash／Target／Qualification差、substep単位のInput sample／Timer／event delivery／publishを各一原因で拒否する。`fixture.navigation.motion-executor.physics-kinematic@1`はreceipt-free Provider record／Catalogを先に固定し、そのexact RecordRef、implementation System Activation Binding、Target、adapted singleton accepted setを`qualification.navigation.motion-executor.physics-kinematic@1` subjectへbindする。signed Receiptはsubjectだけを署名し、Provider Activation BindingがRecordRef＋Receipt refを保持する。positive batchは`adapted_intent_schema_ref=type.navigation.adapted_motion_intent@1`でProvider call exact一回、raw `type.navigation.movement_intent@1`または`type.navigation.motion_intent_contribution@1`へ一Fieldだけ変えた二fixtureはProvider call前にrejectする。accepted setへのraw型追加、RecordRef／Qualification subject／Receipt／Activation Binding hash staleもrejectし、last-valid resolved motion、Path Follower state、Catalogを不変にする。別fixtureはPhysics capability unavailableでも任意Packのinstallが成功し、Physics Provider選択だけがunavailableになることを検証する。Dependency build、exact binary identity、license、target matrixは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、測定とcapacity promotionは[Runtime performance／capacity](../04-runtime/performance-capacity.md)が所有する。

Snapshot fixtureは同じcompleted Intervalから作ったnormalized body／joint／kinematic stateとself-excluding hashをpositive caseとし、Cadence、Interval Ref semantic hash、completed-object SHA、advance sequence、World generation、World Profile identity／activation generation、Substep Binding、array order、body／ground ref generation、snapshot hashを各一Fieldだけ変えたcaseを`physics_snapshot_stale`またはschema failureでpublish／consume前に拒否する。各arrayは上限ちょうどを受理し一件超過を`physics_snapshot_capacity_exceeded`で拒否し、partial snapshot、`latest`再解決、cross-advance rebaseを0件とする。

Save fixtureは同じsnapshotとCollider Artifact集合をDomain BindingでHeaderへ結ぶcaseを受理し、HeaderのCadence／Interval Ref／completed SHA／Substep Binding、snapshot hash、Collider Source identity／Artifact bytes、Collider集合順／set equalityを各一Fieldだけ変えたcaseを`physics_save_incompatible`でLoad staging前に拒否する。Collider arrayは65,536件ちょうどをschemaとして受理し、65,537件目を`physics_save_capacity_exceeded`でSave生成前に拒否する。

Replay fixtureは生成順`receipt-free Physics Projection → Projection set／Header／Ref → root外 AuthoritativeReplayDomainBindingV1`を受理し、HeaderのCadence／Substep Binding／Interval Ref／completed SHA／sequence、Projection membership、Binding hashを各一Fieldだけ変えたcase、Projection base recordへのHeader／Binding埋戻し、別Header／Projectionを結ぶBindingをReplay開始前に拒否する。

## 5. AI semantics

Physics AI surfaceは自然言語を直接Body fieldへ投影せず、`intent -> discovery -> questions／assumptions -> semantic resolution -> planned semantic action -> preview -> validation`を一つのbounded pipelineとして扱う。RuntimeやVendor APIはAI surfaceではない。

### 5.1 Intentとclosed vocabulary

`PhysicsIntentVocabularyEntryV1`は自然言語の単語をMCD Operation identityへ直接bindする辞書ではなく、Game文脈から意味候補を絞るversion付きCatalog entryである。mandatory fieldを次へ固定する。

| Field | 型／規則 |
|---|---|
| `semantic_tag` | closed ID |
| `localized_terms` | locale別の代表語。命令や権限を含めない |
| `positive_examples` | Tagに一致する短いGameplay文 |
| `negative_examples` | 表面語が似ても一致しない文 |
| `candidate_physics_role_refs` | `PhysicsIntentRoleRefV1[]`。owner／version／hash付き |
| `question_triggers` | 意味が分岐する条件 |
| `candidate_capability_ids` | discovery候補。利用可否はManifestで再検査 |
| `forbidden_mappings` | 自動変換してはならないrole／operation |
| `rationale_refs` | Architecture requirement／section参照 |

`localized_terms`の文字列一致だけでResolutionを確定しない。Vocabulary entryはBackend名、exact dependency version、native setting、thread countを含めず、未登録文字列を新enumとして保存しない。

Physics Coreはobject／Genre名のclosed enumを所有せず、次のrole registryだけを所有する。

```text
PhysicsIntentRoleRefV1
  role_id
  role_version: uint32
  role_content_hash: SHA-256

PhysicsIntentRoleRecordV1
  role_id
  role_version: uint32
  owner_ref: exact {owner_id, owner_revision, owner_content_hash}
  status: active | reserved_unsupported | removed
  allowed_motion_authorities[1..5]
  allowed_collision_semantics[1..4]
  allowed_hit_authorities[1..6]
  allowed_shape_strategies[1..7]
  allowed_speed_policies[1..4]
  required_capability_refs[0..32]
  role_content_hash: SHA-256

PhysicsIntentRoleQualificationReceiptRefV1
  qualification_id
  qualification_version: positive uint32
  qualification_subject_hash: SHA-256
  signed_record_hash: SHA-256

PhysicsIntentRoleQualificationSubjectV1
  qualification_id
  qualification_version: positive uint32
  owner_ref: exact {owner_id, owner_revision, owner_content_hash}
  role_ref: PhysicsIntentRoleRefV1
  target_profile_refs[1..16]: exact version/hash-bound Target refs
  fixture_refs[1..64]: exact {fixture_id, fixture_version, fixture_content_hash}
  input_closure_hash
  result: pass | fail
  qualification_subject_hash: SHA-256

PhysicsIntentRoleQualificationReceiptV1
  subject: PhysicsIntentRoleQualificationSubjectV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=physics_intent_role_qualification)

PhysicsIntentRoleActivationBindingRefV1
  activation_binding_id
  activation_binding_version: positive uint32
  activation_binding_hash: SHA-256

PhysicsIntentRoleActivationBindingV1
  activation_binding_id
  activation_binding_version: positive uint32
  role_ref: PhysicsIntentRoleRefV1
  qualification_receipt_refs[1..64]:
    PhysicsIntentRoleQualificationReceiptRefV1
  activation_binding_hash: SHA-256

PhysicsIntentRoleRegistryRefV1
  registry_id
  registry_version
  registry_content_hash

PhysicsIntentRoleRegistryV1
  registry_id: physics.intent_role.registry.active
  registry_version
  registry_content_hash
  records[1..4096]: PhysicsIntentRoleRecordV1

PhysicsIntentRoleSelectionDocumentV1
  common Project Document header
  selection_id: same StableId as Document header
  subject_definition_ref/hash
  selected_role_ref: PhysicsIntentRoleRefV1
  selected_role_activation_binding_ref:
    PhysicsIntentRoleActivationBindingRefV1
  selected_axis_closure_hash
  selection_content_hash
```

Core初期roleは次のbehavior-neutralなexact六Receipt-free base recordだけである。全Refはversion 1、全ownerは`owner.core.physics`のexact revision／content hash、全rowの`status=active`、Capabilityはversion／Contract set root付きで保存する。

| role ID | status | allowed motion | allowed collision | allowed hit | allowed shape | allowed speed | required Capability |
|---|---|---|---|---|---|---|---|
| `role.physics.static_environment` | `active` | `static` | `solid_block; query_only` | `solver_contact; swept_shape_query; overlap_query; none` | `primitive; compound_primitive; convex; static_triangle_mesh; heightfield; tile_chain_2d` | `discrete` | `capability.simulation.collision` |
| `role.physics.dynamic_body` | `active` | `dynamic_solver` | `solid_block; sensor_notify` | `solver_contact; sensor_event; none` | `primitive; compound_primitive; convex` | `discrete; continuous_body` | `capability.simulation.physics_dynamics` |
| `role.physics.kinematic_body` | `active` | `kinematic_target` | `solid_block; sensor_notify` | `solver_contact; sensor_event; swept_shape_query; none` | `primitive; compound_primitive; convex` | `discrete; authoritative_sweep; teleport` | `capability.simulation.collision` |
| `role.physics.sensor` | `active` | `static; kinematic_target` | `sensor_notify` | `sensor_event; overlap_query; none` | `primitive; compound_primitive; convex; tile_chain_2d` | `discrete; teleport` | `capability.simulation.collision_query` |
| `role.physics.query_subject` | `active` | `query_driven` | `query_only` | `swept_shape_query; overlap_query; gameplay_rule; none` | `primitive; compound_primitive; convex; tile_chain_2d` | `discrete; authoritative_sweep; teleport` | `capability.simulation.collision_query` |
| `role.physics.presentation_proxy` | `active` | `presentation_only` | `none` | `none` | `none` | `discrete; teleport` | `capability.presentation.transform_proxy` |

Registry／RoleRef確定後、次のroot外Activation bindingを各Roleへexact一件作る。各receipt refは同じrowのRoleRefをsubjectにする`PhysicsIntentRoleQualificationSubjectV1`のcanonical signed wrapperへ解決する。

| RoleRef | exact Qualification Receipt | exact Activation Binding |
|---|---|---|
| `role.physics.static_environment@1` | `qualification.physics.intent-role.static-environment@1` | `activation.physics.intent-role.static-environment@1` |
| `role.physics.dynamic_body@1` | `qualification.physics.intent-role.dynamic-body@1` | `activation.physics.intent-role.dynamic-body@1` |
| `role.physics.kinematic_body@1` | `qualification.physics.intent-role.kinematic-body@1` | `activation.physics.intent-role.kinematic-body@1` |
| `role.physics.sensor@1` | `qualification.physics.intent-role.sensor@1` | `activation.physics.intent-role.sensor@1` |
| `role.physics.query_subject@1` | `qualification.physics.intent-role.query-subject@1` | `activation.physics.intent-role.query-subject@1` |
| `role.physics.presentation_proxy@1` | `qualification.physics.intent-role.presentation-proxy@1` | `activation.physics.intent-role.presentation-proxy@1` |

recordsはrole ID／version順へstrict sortし、exact duplicate、同一ID／versionの別content hash、同じrole IDのactive record複数、owner namespace偽装、非canonical orderを拒否する。`role_content_hash`はASCII `MIRAKAN_PHYSICS_INTENT_ROLE_RECORD_V1`と、当該hash Fieldだけを除くReceipt-free Record canonical MCD bytesを`uint32_be` length framingしてSHA-256する。Recordはhashを含まないidentity Fieldをpreimageに持ち、外部`PhysicsIntentRoleRefV1`、Qualification Receipt、Activation Bindingをrecord内へ埋め戻さない。Registry hashはASCII `MIRAKAN_PHYSICS_INTENT_ROLE_REGISTRY_V1`、Registry ID／version、record count、全record canonical bytesを各`uint32_be` length framingしてSHA-256し、`registry_content_hash`自身を除外する。`PhysicsIntentRoleRegistryRefV1`は三Fieldすべてを同一active Registryへexact解決し、ID-only、latest version、hash fallbackを許可しない。Registry／RoleRef確定後、Qualification subject hashをASCII `MIRAKAN_PHYSICS_INTENT_ROLE_QUALIFICATION_SUBJECT_V1`、Activation binding hashをASCII `MIRAKAN_PHYSICS_INTENT_ROLE_ACTIVATION_BINDING_V1`と各自己Fieldを除くcanonical bytesから計算する。subjectの`owner_ref`は`role_ref`が解決するReceipt-free Role recordのownerとbyte equality、Activation BindingのRoleRefはsubjectとbyte equalityでなければならず、Receipt refのqualification ID／versionはwrapper subjectと一致する。signed wrapperは完成subjectだけを署名する。生成順は`receipt-free Role → Registry／RoleRef → Qualification subject → signed Receipt → Activation Binding → Selection`である。Production Selection／Compile／Save／ReplayはActivation Bindingが指す署名済みReceiptのsubject／result／freshnessだけを検証し、Fixture bodyを解決しない。Fixture集合は別`PhysicsIntentRoleQualificationSubjectV1`だけが所有する。owner、RoleRef、subject hash、signed hash、Activation Binding RoleRefの各一Fieldを別baseへ差し替えるnegative fixtureを持つ。

Pack／Projectはowner namespace、exact Capability、axis compatibilityを持つReceipt-free Role recordを下向きに登録できる。Registry／RoleRef固定後、同じownerとRoleRefを持つroot外`PhysicsIntentRoleQualificationSubjectV1`だけがobject vocabulary、具体例、default mapping、Fixtureを所有し、canonical signed ReceiptとActivation Bindingを順に生成する。Receipt／BindingをRole recordまたはRegistry hashへ戻さず、Core resolver、Core vocabulary、Core fixture inventoryへcontributor fixtureをコピーしない。role refは候補検索を助ける分類であり、motion／collision／hit／shape／speed各axisの検証を省略または上書きしない。

Project Sourceの選択正本は`PhysicsIntentRoleSelectionDocumentV1`であり、RegistryとResolutionは派生である。ただし選択write surfaceは本Taskで完全Operation登録されていないため、`operation.physics.intent_role.select`は[Executable contracts](../02-foundation/executable-contracts.md#211-既存domain文書から回収した未登録operation候補)の`planning.operation_family.physics_role_selection@1`に属するexact一候補であり、current MCD、Owner Manifest、Trusted Service allowlist、Policy、Validator、Diagnostic、Receipt、Provider／MCP Catalog、generated alias、legacy aliasの各集合をすべて`[]`とする。Capability stateは`not_activated`で、Editor／AIの選択要求は`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`を返してSource不変にする。将来activation work item `activation.physics.intent_role_selection.v1`はcreate/upsertとupdateを別named inputで定義し、MCD全Field、Service allowlist、risk、side effect、idempotency、transaction、pure pre／post policy、closed Diagnostic、Validator closure、rate／timeout、canonical selection set hash／sort／duplicate rule、signed Receipt、private-to-public crash recovery、positive／negative Qualificationを同じContract set transactionで完備する。name-only `select`を先行公開しない。Compile closure、Save、Replayは既存Selection Document ref／hash、RegistryRef、RoleRef、Role Activation Binding ref、axis closure hashを保存し、reload時にSource→Registry→record→Capability→Qualification subject→signed Receipt→Activation Bindingを再検査する。

`PhysicsIntentResolutionV1`のmandatory schemaは次である。fieldの省略、任意propertyの追加、closed value以外の文字列を拒否する。

```text
PhysicsSourceRequestRefV1
  task_id: StableId
  content_id: StableId
  content_revision: positive uint64
  content_hash: SHA-256
  access_policy_ref: McdContractRefV1(kind=policy)

PhysicsCostEstimateRefV1
  estimate_id: StableId
  estimate_version: positive uint32
  estimate_content_hash: SHA-256

PhysicsIntentResolutionV1
  resolution_id: StableId
  resolution_version: positive uint32
  source_request_ref: PhysicsSourceRequestRefV1
  source_request_hash: SHA-256
  contract_set_hash: SHA-256
  project_ref:
    exact {project_id, project_revision, document_set_hash}
  target_profile_refs[1..16]:
    exact {target_profile_id, target_profile_version,
           target_profile_content_hash}
  physics_role_registry_ref: PhysicsIntentRoleRegistryRefV1
  scene_dimension: two_d | three_d | hybrid
  hybrid_gameplay_space: two_d | three_d
    | canonical omission unless scene_dimension=hybrid
  physics_role_ref: PhysicsIntentRoleRefV1
  motion_authority: PhysicsMotionAuthorityV1
  collision_semantics: PhysicsCollisionSemanticsV1
  hit_authority: PhysicsHitAuthorityV1
  shape_strategy: PhysicsShapeStrategyV1
  speed_policy: PhysicsSpeedPolicyV1
  selected_capability_refs[0..32]: McdContractRefV1(kind=capability)
  selected_operation_refs[0..32]: McdContractRefV1(kind=operation)
  blocking_question_refs[0..32]:
    exact {question_id, question_version, question_content_hash}
  assumptions[0..32]:
    exact {assumption_id, assumption_version, assumption_content_hash}
  rejected_alternatives[0..32]:
    exact {alternative_id, alternative_version, alternative_content_hash}
  diagnostic_refs[0..64]: DiagnosticCodeRefV1
  preview_qualification_receipt_refs[0..64]:
    PhysicsIntentRoleQualificationReceiptRefV1
  cost_estimate_ref: PhysicsCostEstimateRefV1
    | canonical omission
  disposition: ready_to_propose | question_required | capability_unavailable | rejected
  resolution_content_hash: SHA-256
```

`source_request_ref`はAuthoring Task内のaccess-controlled contentを参照し、raw PromptをCatalog、MCD、Receiptへ複製しない。全MCD／Registry／Target／Question／Assumption／Alternative／Diagnostic／Qualification／Cost refは上記named typeのversion／hashを必須にし、bare `ContentRef`、`RevisionId`、`CapabilityId`、`OperationId`、`DiagnosticId`、`FixtureId`をcurrent schemaで受理しない。arrayはlogical ID／version／hash順にstrict sortし、duplicate／same-ID different-hashを拒否する。`physics_role_registry_ref`は候補生成時に読んだactive Registryを固定し、validate／preview／proposal時にcurrent Registry refと三Fieldexact equalityで再検査する。`resolution_content_hash`はASCII `MIRAKAN_PHYSICS_INTENT_RESOLUTION_V1`と自己Fieldを除くlength-framed canonical bytesから計算する。

ResolutionはAuthoring Taskのephemeral derived recordであり、Project Source、Save、Replay、Runtime Packageへ永続化しない。永続化するのは承認済みSelection Documentとそのexact Registry／Role／axis／Project closureだけである。Task終了またはsource／Project／Contract／Registry driftでResolutionをinvalidにし、再計算する。`ready_to_propose`は既存の完全登録OperationでChangeSetを提案できる意味だけを持ち、Commit authorizationを意味しない。本Taskでは選択Operationが`not_activated`のためRole selection要求を`ready_to_propose`にせず`capability_unavailable`へ固定する。`question_required`はblocking ambiguityが残る結果、`capability_unavailable`は要求を満たすactive Capability／Operationがない結果、`rejected`はinvalid／forbiddenな要求である。

role以外のclosed semantic axisを次へ固定する。一つのResolutionは各軸から一つだけを選び、role recordのallowed setと照合する。

| Type | Closed values |
|---|---|
| `PhysicsMotionAuthorityV1` | `static \| kinematic_target \| dynamic_solver \| query_driven \| presentation_only` |
| `PhysicsCollisionSemanticsV1` | `solid_block \| sensor_notify \| query_only \| none` |
| `PhysicsHitAuthorityV1` | `solver_contact \| sensor_event \| swept_shape_query \| overlap_query \| gameplay_rule \| none` |
| `PhysicsShapeStrategyV1` | `primitive \| compound_primitive \| convex \| static_triangle_mesh \| heightfield \| tile_chain_2d \| none` |
| `PhysicsSpeedPolicyV1` | `discrete \| continuous_body \| authoritative_sweep \| teleport` |

`GameplayPhysicsRoleV1` enumのcanonical bytesとそれを使用する旧Project／Documentが実在することは現計画からは証明されておらず、current Source、Editor、AI projection、Compile、deserializerへ登録しない。`operation.physics.intent_role.migrate@1`は[Executable Contracts §8.1.2](../02-foundation/executable-contracts.md#812-conditional-legacy-migration-evidence-gate)のconditional legacy migrationで、current状態は`not_activated`である。この移行に固有のcurrent MCD／Owner Manifest／Service allowlist／Policy／Validator／migration Diagnostic／Operation Receipt／Provider／MCP／alias、migration Qualification subject／Receipt／Binding、Activation Catalog／projection subset、Contribution record／Registry、Migration Manifest集合はすべてexact `[]`である。`service.offline_project_migrator`、`capability.authoring.offline_migration`、`profile.isolation.offline_project_migrator`のcurrent集合もexact `[]`で、移行要求をdispatchしない。

将来Activationするには、実在する旧enum schema／Project／Document bytes、旧MCD local record／common envelope／payload、source `ContractSetSnapshotV2`、Owner Identity Registry、Named Algorithm Registry、`FoundationDefinitionClosureV1`、全retained artifact ref／hash、正負fixtureを推測なしで列挙したsigned `LegacyMigrationInventoryV1`が§8.1.2 gateを満たさなければならない。その同じ承認済みContract set transactionだけが、Operation／Type／Policy／Validator／migration Diagnostic／Receipt、offline Service／Capability／Isolation Profile、Service allowlist、Contribution Registry／records、Qualification Receipt／Binding、Manifest、Provider／MCP projectionを完全closureとして同時にmaterializeできる。owner固有変換をCore switch文へ追加しない。以下のschemaとrecord値はpost-activation destination templateであり、block内の`status=active`、Service ref、Policy ref、Provider exposureをcurrent refまたはcurrent product surfaceとして解釈しない。

```text
PhysicsIntentRoleMigrationContributionRefV1
  contribution_id
  contribution_version
  contribution_content_hash

PhysicsIntentRoleMigrationContributionRecordV1
  contribution_id
  contribution_version
  owner_ref/hash
  retained_source_mcd_ref: McdContractRefV1(
    kind=type, id=type.physics.legacy_gameplay_role,
    version=1, source_contract_set_hash)
  accepted_legacy_values[1..64]
  destination_role_refs[1..64]: PhysicsIntentRoleRefV1
  mapping_policy_ref: McdContractRefV1(kind=policy)
  axis_mapping_policy_ref: McdContractRefV1(kind=policy)
  contribution_content_hash

PhysicsIntentRoleMigrationContributionRegistryRefV1
  registry_id
  registry_version
  registry_content_hash

PhysicsIntentRoleMigrationContributionRegistryV1
  registry_id: physics.intent_role.migration_contribution.registry.active
  registry_version
  records[1..4096]
  registry_content_hash

PhysicsIntentRoleMigrationQualificationSubjectV1
  qualification_id
  qualification_version: positive uint32
  contribution_ref: PhysicsIntentRoleMigrationContributionRefV1
  owner_ref: exact contribution owner ref/hash
  target_profile_refs[1..64]
  fixture_refs[1..64]:
    exact {fixture_id, fixture_version, fixture_content_hash}
  input_closure_hash
  result: pass | fail
  qualification_subject_hash: SHA-256

PhysicsIntentRoleMigrationQualificationReceiptV1
  subject: PhysicsIntentRoleMigrationQualificationSubjectV1
  signed_record:
    exact MirakanSignedRecordV1(
      purpose=physics_intent_role_migration_qualification)

PhysicsIntentRoleMigrationQualificationReceiptRefV1
  qualification_id
  qualification_version: positive uint32
  qualification_subject_hash
  signed_record_hash

PhysicsIntentRoleMigrationQualificationBindingRefV1
  qualification_binding_id
  qualification_binding_version: positive uint32
  qualification_binding_hash

PhysicsIntentRoleMigrationQualificationBindingV1
  qualification_binding_id
  qualification_binding_version: positive uint32
  contribution_ref: PhysicsIntentRoleMigrationContributionRefV1
  qualification_receipt_ref:
    PhysicsIntentRoleMigrationQualificationReceiptRefV1
  qualification_binding_hash

PhysicsIntentRoleMigrationManifestV1
  manifest_id: physics.intent_role.migration.v1_to_registry
  manifest_version: 1
  operation_ref: McdContractRefV1(
    kind=operation, id=operation.physics.intent_role.migrate,
    version=1, contract_set_hash)
  input_type_ref: McdContractRefV1(
    kind=type, id=type.physics.intent_role_migration_input,
    version=1, contract_set_hash)
  output_type_ref: McdContractRefV1(
    kind=type, id=type.physics.intent_role_migration_result,
    version=1, contract_set_hash)
  receipt_type_ref: McdContractRefV1(
    kind=type, id=type.physics.intent_role_migration_receipt,
    version=1, contract_set_hash)
  precondition_policy_ref: McdContractRefV1(
    kind=policy, id=policy.operation.physics.intent_role.migrate.precondition,
    version=1, contract_set_hash)
  postcondition_policy_ref: McdContractRefV1(
    kind=policy, id=policy.operation.physics.intent_role.migrate.postcondition,
    version=1, contract_set_hash)
  rate_limit_policy_ref: McdContractRefV1(
    kind=policy, id=policy.authoring.physics_intent_role_migration.rate_limit,
    version=1, contract_set_hash)
  validator_closure_ref: OperationValidatorClosureRefV1
  contribution_registry_ref: PhysicsIntentRoleMigrationContributionRegistryRefV1
  trusted_service_ref: TrustedServiceRefV1(
    service_id=service.offline_project_migrator, service_version=1,
    service_content_hash)
  trusted_service_allowlist_operation_local_refs[1]:
    ContractSetLocalRefV1(
      kind=operation, id=operation.physics.intent_role.migrate, version=1)
  diagnostic_refs[15]: DiagnosticCodeRefV1
  qualification_binding_refs[1..64]:
    exact PhysicsIntentRoleMigrationQualificationBindingRefV1
  manifest_hash: SHA-256
```

Activation後のdestination Core contribution候補は次の完全三Receipt-free recordだけを持つ。全rowは`contribution_version=1`、`owner_ref={owner.core.physics,destination owner revision,content hash}`、source schema=`{type.physics.legacy_gameplay_role,1,source Contract set hash}`を持ち、表のPolicy／Role refはexact version／hashを保存する。表の三rowはcurrent Registry memberではなく、現時点のcurrent contribution集合はexact `[]`である。

| contribution ID | accepted legacy values | destination role refs | mapping policy | axis mapping policy |
|---|---|---|---|---|
| `physics.intent_role.migration_contribution.core.world_static` | `[world_static]` | `[role.physics.static_environment@1]` | `policy.physics.intent_role.migration.world_static@1` | `policy.physics.intent_role.axis.world_static@1` |
| `physics.intent_role.migration_contribution.core.movable_prop` | `[movable_prop]` | `[role.physics.dynamic_body@1]` | `policy.physics.intent_role.migration.movable_prop@1` | `policy.physics.intent_role.axis.movable_prop@1` |
| `physics.intent_role.migration_contribution.core.sensor_volume` | `[sensor_volume]` | `[role.physics.sensor@1]` | `policy.physics.intent_role.migration.sensor_volume@1` | `policy.physics.intent_role.axis.sensor_volume@1` |

Activation後、object固有legacy値はsigned Inventoryに束縛され、当該Pack／Project contributionがexact一件存在する場合だけ移行する。recordはContribution ID／version順、accepted valueはUTF-8 byte順、destination refはrole ID／version順へstrict sortし、duplicate valueの複数active contribution、stale owner／policy／roleをRegistry全体で拒否する。record hashはASCII `MIRAKAN_PHYSICS_ROLE_MIGRATION_CONTRIBUTION_V1`、Registry hashはASCII `MIRAKAN_PHYSICS_ROLE_MIGRATION_CONTRIBUTION_REGISTRY_V1`のself-excluding Receipt-free length-framed canonical bytesである。生成順は`receipt-free Contribution → Registry／ContributionRef → Migration Qualification subject → signed Receipt → Qualification Binding → Migration Manifest`であり、subject ownerはbase owner、Binding ContributionRefはsubjectとbyte equalityにする。Manifest binding集合と選択Contribution集合はexact set equalityで、Receipt／BindingをContribution／Registry hashへ戻さない。Production migration recordはFixture bodyを解決せず、Bindingが指すsigned Receiptのsubject／result=`pass`／freshnessだけを検証する。三Core rowは同logical suffixの`qualification.physics.intent-role-migration.*@1`とBindingをexact一件ずつ持つ。

```text
operation.physics.intent_role.migrate@1
  MCD common envelope:
    mcd_version=1; kind=operation;
    id=operation.physics.intent_role.migrate;
    version=1; status=active;
    title=Migrate Physics Intent Role;
    description=Atomically migrate one legacy Physics object role through
      an exact owner contribution into a typed role Selection Document;
    owners=[owner.core.physics]; requirement_refs=[];
    rationale_refs=[mirakan.arch.simulation-physics#5-ai-semantics];
    since_contract_set=2; supersedes=[]; tags=[authoring,migration,physics]
  operation_kind: command
  input_type: McdContractRefV1(
    kind=type, id=type.physics.intent_role_migration_input,
    version=1, contract_set_hash)
  output_type: McdContractRefV1(
    kind=type, id=type.physics.intent_role_migration_result,
    version=1, contract_set_hash)
  authority: TrustedServiceRefV1(
    service_id=service.offline_project_migrator, service_version=1,
    service_content_hash)
  risk_class: R3
  side_effects: [authoring]
  idempotency: idempotent_with_key
  transaction: authoring_changeset
  preconditions:
    [McdContractRefV1(
      kind=policy, id=policy.operation.physics.intent_role.migrate.precondition,
      version=1, contract_set_hash)]
  postconditions:
    [McdContractRefV1(
      kind=policy, id=policy.operation.physics.intent_role.migrate.postcondition,
      version=1, contract_set_hash)]
  validator_closure_ref:
    {closure_id=validator_closure.operation.physics.intent_role.migrate,
     closure_version=1, closure_content_hash}
  timeout_ms: 120000
  rate_limit_policy: McdContractRefV1(
    kind=policy, id=policy.authoring.physics_intent_role_migration.rate_limit,
    version=1, contract_set_hash)
  audit_level: full_redacted
  provider_exposure: mcp_proposal
  receipt_type: McdContractRefV1(
    kind=type, id=type.physics.intent_role_migration_receipt,
    version=1, contract_set_hash)
  errors[15]: exact DiagnosticCodeRefV1 records for
    diagnostic.conflict.revision_mismatch
    diagnostic.authorization.denied
    diagnostic.approval.required
    diagnostic.authoring.lock_conflict
    diagnostic.mcd.operation_predicate_invalid
    diagnostic.operation.timeout
    diagnostic.operation.rate_limit_exceeded
    diagnostic.operation.idempotency_key_reuse
    diagnostic.physics.intent_role.source_invalid
    diagnostic.physics.intent_role.registry_invalid
    diagnostic.physics.intent_role.contribution_missing
    diagnostic.physics.intent_role.contribution_ambiguous
    diagnostic.physics.intent_role.capability_unavailable
    diagnostic.physics.intent_role.axis_mapping_invalid
    diagnostic.physics.intent_role.receipt_binding_mismatch

PhysicsIntentRoleMigrationInputV1
  input_type_ref: McdContractRefV1(
    kind=type, id=type.physics.intent_role_migration_input,
    version=1, contract_set_hash)
  operation_ref: McdContractRefV1(
    kind=operation, id=operation.physics.intent_role.migrate,
    version=1, contract_set_hash)
  before_project_ref:
    exact {project_id, expected_project_revision, document_set_hash}
  operation_intent_hash
  request_hash
  idempotency_key
  source_document_ref/hash
  source_foundation_definition_closure_ref:
    FoundationDefinitionClosureRefV1
  retained_source_mcd_ref: McdContractRefV1(
    kind=type, id=type.physics.legacy_gameplay_role,
    version=1, source_contract_set_hash)
  source_legacy_value
  source_axis_closure_hash
  destination_role_registry_ref
  contribution_registry_ref
  selected_contribution_ref
  preview_policy_ref: McdContractRefV1(kind=policy)
  validation_policy_ref: McdContractRefV1(kind=policy)
  mutation_authorization_binding: exact MutationAuthorizationBindingV2(
    risk_class=R3, authority_evidence=approval)

PhysicsIntentRoleMigrationResultV1
  disposition: migrated | capability_unavailable | ambiguous | rejected
  migrated:
    after_project_ref
    selection_document_ref/hash
    selected_role_ref
    axis_closure_hash
    migration_receipt_ref/hash
    public_publication_marker_ref/hash
  non-migrated:
    diagnostics[1..15]

PreparedPhysicsIntentRoleMigrationReceiptPayloadV1
  publication_binding: exact PreparedReceiptPublicationBindingV1
  operation_ref
  operation_intent_hash
  request_hash
  idempotency_key
  source_document_ref/hash
  source_foundation_definition_closure_ref:
    FoundationDefinitionClosureRefV1
  retained_source_mcd_ref: McdContractRefV1(
    kind=type, id=type.physics.legacy_gameplay_role,
    version=1, source_contract_set_hash)
  source_legacy_value
  role_registry_ref
  contribution_registry_ref
  selected_contribution_ref
  before_project_ref
  after_project_ref
  selection_document_ref/hash
  selected_role_ref
  axis_closure_hash
  preview_receipt_payload_ref/hash
  validation_receipt_payload_ref/hash
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1
  diagnostics[0..15]
  prepared_payload_hash

PhysicsIntentRoleMigrationReceiptV1
  published_receipt:
    exact PublishedDomainReceiptV2 whose
    prepared_domain_receipt_payload_ref/hash resolves
    PreparedPhysicsIntentRoleMigrationReceiptPayloadV1
```

Activation後にOperationが参照する三Policyのdestination値は次の完全なactive MCD recordである。表の`status=active`はatomic activation完了後の値だけを表し、current Policy集合はexact `[]`である。表の二列を連結した値が各record全体であり、暗黙既定値、bare ID、説明文からFieldを補完しない。

| Policy MCD共通Envelope exact value | Policy payload exact value |
|---|---|
| `mcd_version=1; kind=policy; id=policy.operation.physics.intent_role.migrate.precondition; version=1; status=active; title=Physics Intent Role Migration Precondition; description=Validate the exact legacy source, Project base, role and contribution registries, selected contribution, axes, authorization, and idempotency snapshot without mutation; owners=[owner.core.physics]; requirement_refs=[]; rationale_refs=[mirakan.arch.simulation-physics#5-ai-semantics]; since_contract_set=2; supersedes=[]; tags=[operation_predicate,physics,pure]` | `evaluation_mode=pure; side_effects=[]; input_type={id=type.operation.precondition_evaluation_input,version=1,contract_set_hash}; result_type={id=type.operation.predicate_result,version=1,contract_set_hash}` |
| `mcd_version=1; kind=policy; id=policy.operation.physics.intent_role.migrate.postcondition; version=1; status=active; title=Physics Intent Role Migration Postcondition; description=Validate the unpublished Selection Document, exact role and axis closure, prepared Receipt payload, and atomic Project revision increment; owners=[owner.core.physics]; requirement_refs=[]; rationale_refs=[mirakan.arch.simulation-physics#5-ai-semantics]; since_contract_set=2; supersedes=[]; tags=[operation_predicate,physics,pure]` | `evaluation_mode=pure; side_effects=[]; input_type={id=type.operation.postcondition_evaluation_input,version=2,contract_set_hash}; result_type={id=type.operation.predicate_result,version=1,contract_set_hash}` |
| `mcd_version=1; kind=policy; id=policy.authoring.physics_intent_role_migration.rate_limit; version=1; status=active; title=Physics Intent Role Migration Rate Limit; description=Bound migration requests per Project without changing role resolution semantics; owners=[owner.core.physics]; requirement_refs=[]; rationale_refs=[mirakan.arch.simulation-physics#5-ai-semantics]; since_contract_set=2; supersedes=[]; tags=[authoring,physics,rate_limit]` | `policy_ref={id=policy.authoring.physics_intent_role_migration.rate_limit,version=1,contract_set_hash}; scope=project; window_ns=60000000000; max_requests=4; burst=1; exceeded_error_ref={diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash}` |

Activation destination Contract set内部では三Policyを`ContractSetLocalRefV1(kind=policy)`へ投影し、self refはlocal identityだけにする。Manifest三Policy ref、Operationのpre／post／rate ref、Physics ownerのPolicy local subsetはexact三件でset equalityである。三recordの共通Envelopeまたはpayloadの実在Fieldを一つだけ変えるfixtureはPolicy member hashとset rootを変更し、旧Manifest／Operation external refを解決不能にする。

Activation後のmigration固有Diagnostic destination値は次の完全な`DiagnosticLocalRecordV2`である。current migration Diagnostic集合はexact `[]`である。全rowは`diagnostic_version=1`、`owner_local_ref={owner_id=owner.core.physics,owner_revision=1,owner_content_hash=SHA-256(MIRAKAN_DIAGNOSTIC_OWNER_LOCAL_IDENTITY_V1, length-framed canonical owner ID／revision)}`、`requirement_local_refs=[]`、`message_key="<diagnostic_id>.message"`、Ownerを含むself-excluding `diagnostic_local_content_hash`を持つ。root確定後だけ同じ三Field Owner ref、`requirement_refs=[]`、別のself-excluding `diagnostic_content_hash`を持つ外部Registry recordへ投影する。共通八件はActivation先Foundationの共通recordを参照し、本書のcurrent Diagnosticへ複写しない。

| Diagnostic ID | code | severity／category／retryability |
|---|---|---|
| `diagnostic.physics.intent_role.source_invalid` | `MIRAKAN-PHYSICS-INTENT-ROLE-SOURCE-INVALID` | blocking／schema／after_input |
| `diagnostic.physics.intent_role.registry_invalid` | `MIRAKAN-PHYSICS-INTENT-ROLE-REGISTRY-INVALID` | blocking／schema／after_change |
| `diagnostic.physics.intent_role.contribution_missing` | `MIRAKAN-PHYSICS-INTENT-ROLE-CONTRIBUTION-MISSING` | blocking／semantic／after_change |
| `diagnostic.physics.intent_role.contribution_ambiguous` | `MIRAKAN-PHYSICS-INTENT-ROLE-CONTRIBUTION-AMBIGUOUS` | blocking／semantic／after_input |
| `diagnostic.physics.intent_role.capability_unavailable` | `MIRAKAN-PHYSICS-INTENT-ROLE-CAPABILITY-UNAVAILABLE` | blocking／semantic／after_change |
| `diagnostic.physics.intent_role.axis_mapping_invalid` | `MIRAKAN-PHYSICS-INTENT-ROLE-AXIS-MAPPING-INVALID` | blocking／semantic／after_input |
| `diagnostic.physics.intent_role.receipt_binding_mismatch` | `MIRAKAN-PHYSICS-INTENT-ROLE-RECEIPT-BINDING-MISMATCH` | blocking／semantic／after_change |

Activation後の`validator_closure.operation.physics.intent_role.migrate@1` destinationは次のexact Validator recordを持つ。current Validator／Validator closure集合はexact `[]`である。各recordはversion 1、実装Artifact ref／hash、表のinput Type ref、表のDiagnostic ref集合、self-excluding content hashを持ち、ID／version順にsortする。

| Validator | input | exact reachable Diagnostic |
|---|---|---|
| `validator.operation.request_envelope` | migration input | idempotency key reuse |
| `validator.operation.authorization` | migration input | authorization denied |
| `validator.operation.approval` | migration input | approval required |
| `validator.operation.revision_and_lock` | migration input | revision mismatch; lock conflict |
| `validator.operation.pure_predicate` | migration input | operation predicate invalid |
| `validator.operation.timeout_and_rate_limit` | migration input | timeout; rate limit exceeded |
| `validator.physics.intent_role.migration_semantics` | migration input | source invalid; registry invalid; contribution missing; contribution ambiguous; capability unavailable; axis mapping invalid |
| `validator.physics.intent_role.migration_postcondition` | postcondition input v2 | receipt binding mismatch |

Activation後のintent／request hashはExecutable Contractsの唯一の`MIRAKAN_OPERATION_INTENT_V2 -> MutationAuthorizationBindingV2 -> MIRAKAN_OPERATION_REQUEST_V2`を使う。Input、選択`PhysicsIntentRoleMigrationContributionRecordV1`、Prepared payloadの`retained_source_mcd_ref`はbyte equalityで、InputとPrepared payloadの`source_foundation_definition_closure_ref`もbyte equalityにする。source Closureはexact `{type.physics.legacy_gameplay_role,1,source_contract_set_hash}`、signed Inventoryが列挙したlegacy Source record／Owner、同時代Named Algorithm Registryを解決する。Operation／input／output／Policy／Validator／Diagnostic、destination Role Registry、Contributionのmapping／axis policy、request Algorithm bindingはdispatch時のdestination Foundation Closureだけへ解決する。両source Fieldをintent semantic inputとPublic Receiptから到達するPrepared payloadへ保持し、missing、旧`source_schema_ref`別名、wrong source root、同じContract Setの別Owner／Algorithm root、Input–Contribution–Receipt差、sourceをdestinationへalias、destination downgradeを一原因ずつ拒否する。

Activation後のPrepared payload Typeはexact `type.physics.prepared_intent_role_migration_receipt_payload@1`、hashはASCII `MIRAKAN_PREPARED_PHYSICS_INTENT_ROLE_MIGRATION_RECEIPT_PAYLOAD_V1`とself-excluding canonical bytesを使い、最終Receipt Typeと相互代用しない。唯一のsigned subject／wrapperは共通`PublishedDomainReceiptPayloadV2`／`PublishedDomainReceiptV2`であり、Domain固有Subject／alternate wrapperを作らない。publicationは[Executable Contracts §8](../02-foundation/executable-contracts.md#8-operation定義)をcanonical reuseし、`private Marker read-back → secret-free PublicCommitClosureV1 candidate → signed wrapper read-back → PublicCommitClosureV1＋PublicPublicationMarkerV1＋after Projectのatomic CAS`の順に固定する。Closureの`domain_commitment.kind`は`owner_typed_state_commit`、`domain_owner_ref`はcurrent Foundation Closureが解決するexact `owner.core.physics` ref、committed artifact集合はPrepared payloadが束縛したreceipt-free Selection Document／関連artifact ref集合とし、Closure Ref／hashの規則は同節から参照して本書で複写しない。Closure bodyまたは同Closureを束縛するsigned wrapperを欠くPublic Marker／after-state current authorityを拒否する。同じidempotency key＋request hashのretryはbyte-identical Result／`PublicCommitClosureV1`／signed Receipt／Public Markerを返し、同じkey＋別requestはidempotency reuse errorでSource不変にする。Validator error union、Operation `errors[]`、Manifest `diagnostic_refs[]`は15 refのset equalityにする。Manifestは上記Operation／Type／Policy／Validator／Registry／Diagnosticのexact version／hashと、選択Contributionごとのexact `PhysicsIntentRoleMigrationQualificationBindingRefV1`を全件持つ。各Bindingは同じContribution refをsubjectにする署名済みReceiptへexact解決し、ManifestがReceipt refまたはFixture bodyを直接保持しない。missing／extra／staleをcompile前に拒否する。ManifestのOperation LocalRef集合と`service.offline_project_migrator`へのallowlist contributionはexact一件でset equalityとし、同じatomic activation transactionでService local recordとset rootを生成する。未導入Capabilityの旧予約値は`capability_unavailable`でSourceを不変にし、Core active roleへ近似変換しない。current serializer／AI projectionは旧enum値を受理しない。

複合objectは複数Resolutionと明示関係で表し、合成enumを追加しない。同じobjectへ複数motion authorityを選ばない。Dynamic Bodyへ`static_triangle_mesh`／`heightfield`を選ばず、Sensorをauthoritative hitへ暗黙昇格せず、`teleport`を経路hitの代用にしない。途中経路がGameplayへ必要なら`authoritative_sweep`を使用する。

### 5.2 Capability discoveryと意味解決

Resolverは[Executable contracts](../02-foundation/executable-contracts.md)のCapability registryから、scene dimension、active maturity、Target、World Profile、Collision capability、authoring permission、Physics Intent Role Registryを読み、current MCDへ完全登録済みのexact Operationだけを提示する。Backend featureをCapabilityとして直接表示しない。unknown／removed／reserved-unsupported role、required Capability不足は`capability_unavailable`とし、近いCore roleまたは別のactive Operationへsilent downgradeしない。

解決順は次である。

1. source requestのcontent hash、Project revision、Contract set hash、Target Profile、active Physics Intent Role Registry refを取得し、Resolutionへ観測値としてbindする。
2. ユーザー文からowner vocabulary候補と、motion、contact、hit、shape、speed、dimensionの独立候補を抽出する。
3. Scene／Projectの既存World、Body、Collider、Profile、Capabilityをread-only discoveryする。
4. exact owner role refとclosed axisへ候補を割り当て、矛盾と欠落を分類する。
5. gameplay behaviorを変える欠落だけを`blocking_question_ids`へ入れ、`question_required`にする。安全な欠落だけをReference assumptionとして明示する。
6. role ref、motion authority、collision semantics、hit authority、shape strategy、speed policyを独立に確定する。
7. Target、Capability、Profile、field relation、forbidden mappingを同じValidatorで再検査する。対応不能は`capability_unavailable`、invalid／forbiddenは`rejected`にする。
8. 対応するPhysics／Collision authoring familyが完全にatomic Activation済みの場合だけ`ready_to_propose`とし、そのfamilyのexact外側MCD Operationに閉じたtyped change primitiveとPreview fixtureを返す。currentでは対応write familyが空なので`capability_unavailable`を返し、proposal／Preview refをcanonical omissionする。
9. GatewayがProvider出力を同じMCDとValidatorで再計算し、不一致をresolution mismatchとして拒否する。

Resolutionのvalidate、preview、Activation後のoperation proposalの各入口は、現在のsource request hash、Contract set hash、Project revision、Physics Intent Role Registry refを保存済み`source_request_hash`、`contract_set_hash`、`project_revision`、`physics_role_registry_ref`と完全一致で再検査する。一つでも異なるResolutionは`stale`として拒否し、selected MCD Operation ref／change primitive、preview、cost estimateを使用しない。最新source／contract／Project／Registry snapshotでCapability discoveryから再解決し、新しいResolution identityを発行する。stale objectをfield単位で更新、別revisionへrebase、Commitへ継続してはならない。

### 5.3 質問、Assumption、代替案

2D／3D／hybrid gameplay space、motion authority、contact／hit authority、shape class、高速移動policy、kinematic support relation、壊れるJoint、Target Profile、概算同時instance数が挙動を変える場合は質問する。質問は「どのsolverを使うか」や特定object名を前提にせず、観測可能な挙動の選択肢、影響、推奨案を示す。

明示情報がない場合もobject名からstatic／dynamic、solid／sensor、solver／query、discrete／continuousを既定化しない。Reference assumptionは登録済みrole recordと独立axisの候補として提示し、source intent、根拠、影響、owner refを持たせてPreviewから変更できるようにする。安全な選択肢が複数ある場合はtyped alternativeを最大限界内で提示し、候補ごとの差分を示す。

Authorization、Risk class、commit可否、credentialは[AI Security／Approval](../01-governance/ai-security-approval.md)だけが決定する。本書はplanned semantic actionとActivation後のexact MCD Operationの意味／validationを定め、approval表を複写しない。

## 6. Operation、preview、diagnostic、AI eval

inspect／discover／validate／preview、World Profile作成／更新、Body dynamics設定、Joint／Constraint作成／更新／削除、Physics Kinematic Motion Provider qualification提案はsemantic action vocabularyであり、それ自体はStable Operation IDまたはcurrent公開Operationではない。Physics ownerのcurrent MCD Operation集合はexact `[]`である。`operation.physics.intent_role.migrate@1`は§5のdestination templateを持つconditional legacy migration exact一件、`planning.operation_family.physics_role_selection@1`のselectは別の未Activation候補 exact一件であり、どちらもcurrent Operationへ数えない。action名から追加ID、Manifest row、Service allowlist、Provider／MCP Toolを生成しない。owner固有proposalのadapter／Provider binding actionは当該PackまたはProject、Collision geometry／filter／query actionは[Collision](collision.md)へ意味上handoffするだけでOperation権限を暗黙生成しない。将来完全登録されたwriteだけが[Project state](../03-authoring/project-state.md)のChangeSetを作り、live Worldを直接mutateしない。

Activation後のPreviewはbefore／after semantic resolution、affected Entity／Asset、selected assumptions、question state、Capability availability、estimated impact class、diagnostic、rollback boundaryを示す。native setting dumpやVendor object graphをユーザー説明に使わない。Editor手動actionとAI actionは同じDocument、validator、preview、undo／redo、cookを通る。

Diagnosticは少なくとも次を区別する。

- `MIRAKAN-PHYSICS-KINEMATIC-MOTION-INCOMPATIBLE`: intent subset、Profile、Target、dimension、Collision query relation不一致
- `MIRAKAN-PHYSICS-KINEMATIC-MOTION-RESOLUTION_FAILED`: bounded resolverがvalid resolved motionを生成できない
- `MIRAKAN-PHYSICS-KINEMATIC-MOTION-STALE_RESULT`: motion subject／intent batch／profile／provider generation不一致
- ambiguous intent／question required
- conflicting role／motion／collision semantics
- Capability unavailable／Target unsupported
- invalid profile／shape／joint／kinematic support relation
- unsafe speed／hit assumption
- stale scene／artifact／generation
- stale source request／Contract set／Project revision
- Provider／Gateway resolution mismatch
- operation scope mismatch
- adapter qualification unavailable

各diagnosticはstable code、対象path、原因、修正候補、blockingか否かを返す。Validation failureを自然言語だけで返さず、unknown intentを最も近い既知roleへ自動確定しない。

AI Evalはvalid intent、question-required intent、conflicting intent、unsupported Capability、adversarial Vendor API要求、stale discovery、preview／commit差分をfixture化する。source request hash、Contract set hash、Project revisionを個別に変更したstale fixtureは旧Resolutionのvalidate／preview／proposal拒否と、fresh discoveryからのnew Resolution発行を検査する。評価は全closed enum、4 disposition、mandatory field、semantic resolutionの正解、必要質問、unsupportedの拒否、operation boundedness、diagnosticの再現性を検査する。Evidence、provenance、trace gradingの形式は[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)だけを消費する。

次を採用しない。

- Vendor API／setting／serializationをpublic Physics contractまたはAI vocabularyにすること
- backend名をユーザーのgameplay intentとして質問すること
- unsupported Capabilityのsilent fallback
- AI、Editor、Project C++からの任意World step／callback登録
- PhysicsによるCollision geometry／event、Runtime phase／capacity、Animation poseの再所有
