# Miraikanai Engine Physics Contract

- 文書ID: mirakan.arch.simulation-physics
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Physics World／Body dynamics、solver profile semantics、command、joint／constraint、generic Kinematic Motion reference Provider、kernel Adapter boundary、private Backend optimization eligibility、Physics save／replay projection
- 非正本範囲: Collider geometry／filter／query／event、Runtime phase／Simulation Advance／capacity、ECS storage、Save／Replay header・transport、Animation pose、Navigation artifact、external dependency version／build pin、AI authorization／evidence envelope。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Collision](collision.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)
- 関連文書: [Physics AI Catalog Proposal](../appendices/physics-ai-catalog-proposal.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Project state](../03-authoring/project-state.md)、[Gameplay programming model](../03-authoring/gameplay-programming-model.md)、[Persistence／Save](../04-runtime/persistence-save.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[LOD](../06-rendering/lod.md)、[Virtualized／Continuous Geometry](../06-rendering/virtualized-continuous-geometry.md)、[World](../06-rendering/world.md)、[Collision](collision.md)、[Navigation](navigation.md)、[Animation](animation.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-26

## 1. 結論とPlatform境界

PhysicsはEngine-owned World、Body、Dynamics command、Joint／Constraint、snapshot、diagnosticを公開し、数値kernelをprivate Adapterへ隔離する。Kinematic Motion ExecutorはCore必須契約ではなく、任意のregistered compositionが選択できるEngine-owned C1 reference Providerである。Project C++、GameplayDefinition、AI、EditorへVendorの型、ID、pointer、callback、setting、serializationを公開しない。body／joint generation handle、snapshot lease、kernel scratchとretireの一般規則は[Memory／Pointers](../02-foundation/memory-pointers.md)のbindingを消費し、Physics固有のWorld／Body／substep意味は本書が所有する。採用dependencyとexact version／commit／license／build optionは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)だけが所有する。

[Collision](collision.md)はshape、Collider Asset、material、filter、query、contact／trigger／hit semanticsを所有する。Physicsはそれらを消費してWorldを進めるが再定義しない。[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)はcanonical phase、writer、lease、publishを、[Runtime performance／capacity](../04-runtime/performance-capacity.md)は共通capacity、queue、measurementを所有する。

[LOD](../06-rendering/lod.md#7-simulation-lod境界)へ公開できるPhysics behaviorは、Physics Ownerがexact `SimulationLodCandidateDescriptorV1`、retained state、wake condition、handoff、authoritative equivalence fixtureを定義・Qualificationしたcandidateだけである。現行targetは`full`だけで、reduced integration、遠距離sleep、dormant bodyをLOD candidateとして登録していない。render visibility、Camera distance、Quality、pressureからBody dynamics、solver cadence、accepted command、contact outcomeを変更せず、Backendのnative sleepをSimulation LOD tierとして公開しない。

[Virtualized／Continuous Geometry](../06-rendering/virtualized-continuous-geometry.md)のArtifact、page residency、micro-cluster cut、overflow、fallbackはPresentation inputであり、Body creation／sleep／wake、solver cadence、broadphase、accepted command、contact outcome、Physics Save／Replayへ入力しない。Render boundsとPhysics boundsは各Ownerのexact source relationから導き、micro-cluster boundsまたはresident setからPhysics geometry／World activationを再構成しない。

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

`Physics2DWorldProfileV1`／`Physics3DWorldProfileV1`とBody variantはexecuting semanticsを表す型であり、選択authorityではない。active Physics Worldはexact `WorldSpaceProfileRefV1`から対応variantをderiveし、Physics Profile／Body／Kinematic providerがindependent scene dimensionまたはhybrid gameplay authorityを保存しない。

### 2.1 Physics substep Profile

Physicsはsubstepの意味とpartitionだけを所有し、outer Simulation Advance、Cadence選択、phase、publishは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)が所有する。型を次へ固定する。

```text
PhysicsSubstepProfileRefV1
  profile_id: namespace付きStableId
  profile_version: positive uint32
  profile_content_hash: SHA-256

PhysicsSubstepWorldSpaceEntryV1
  compatible_world_space: WorldSpaceCompatibilityV1
  substep_count: uint8[1..16]

PhysicsSubstepProfileV1
  profile_id: namespace付きStableId
  profile_version: positive uint32
  interval_requirement: non_null_logical_duration
  partition_policy: equal_rational_partition
  world_space_entries[1..2]: PhysicsSubstepWorldSpaceEntryV1
  profile_content_hash: SHA-256

PhysicsSubstepIntervalInputV1
  cadence_profile_ref: SimulationCadenceProfileRefV1
  simulation_advance_interval_hash: SHA-256
  physics_substep_profile_ref: PhysicsSubstepProfileRefV1
  advance_sequence: positive uint64
  world_space_profile_ref: exact WorldSpaceProfileRefV1
  resolved_dimension: d2 | d3
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

`profile_content_hash`はASCII `MIRAKAN_PHYSICS_SUBSTEP_PROFILE_V1`と、同Fieldだけを除くclosed recordのRFC 8785 JCS bytesを`uint32_be` length framingしてSHA-256する。Refは完成base recordのID／version／self-excluding content hashからrecord外でmaterializeし、base record自身へ埋め戻さない。`world_space_entries[]`はcompatibility predicateのcanonical bytes順へstrict sortし、duplicate／overlapを拒否する。active Physics Worldのexact Profileに一致するentryは厳密に一件でなければならず、`resolved_dimension`はそのWorld Profileから再計算するexecution projectionである。ProfileがWorldのstructural dimensionまたはhybrid authorityを独立に保存・選択しない。

Qualification subject hashはASCII `MIRAKAN_PHYSICS_SUBSTEP_QUALIFICATION_SUBJECT_V1`、Activation Binding hashはASCII `MIRAKAN_PHYSICS_SUBSTEP_ACTIVATION_BINDING_V1`と各自己hash Fieldだけを除くclosed canonical bytesから計算する。生成順は`Receipt-free PhysicsSubstepProfile → Profile Ref／Cadence Profile → Target別Qualification subject → signed Receipt → root外Activation Binding／Binding Ref → Runtime Package`である。Binding三RefはReceipt subjectのCadence、Substep、Targetとbyte equalityで、Receiptのsubject hash／signed record hash、`result=pass`、freshness／revocationを検証する。Receipt、Binding、Fixture bodyをSubstep ProfileまたはCadence Profileのcontent hashへ埋め戻さず、Production Runtime PackageはProfile RefとBinding Refを別Fieldで保持する。

T50は一つのsealed `SimulationAdvanceIntervalV1`の正のlogical duration `D={numerator,denominator}`を、exact World Profileに一致するworld-space entryの`substep_count = C`に従いordinal `1..C`の各`D/C`へexact rational partitionする。`g=gcd(numerator,C)`、`substep={numerator/g, denominator*(C/g)}`の順でcross-reduceしてからchecked multiplicationし、既約化後の分子／分母が`ReducedPositiveRationalV1`の各`uint32`表現域へ収まらないProfile／Interval組合せはT50前に`physics_substep_interval_overflow`で拒否する。wrap、saturate、float、別rateへの近似を使わない。全`PhysicsSubstepIntervalInputV1`は同じCadence Profile ref、outer interval hash、advance sequence、Substep Profile ref、exact World Space Profile refへbyte equalityでbindし、ordinal順に実行する。substepは新しいSimulation Advance、Input sample、Timer boundary、event delivery boundaryまたはpublish boundaryを生成せず、T60は全substep完了後にouter advanceにつきexact一回だけ結果を統合する。null duration、World Profile entry不足、Profile／Target Qualification不足、Cadence／outer interval／Substep Profile／World Profile ref不一致ではfallbackせず`cadence_profile_not_qualified`としてT50開始前に拒否する。

current reference `fixed 60/1` Cadenceは`physics_substep_profile_ref=null`かつouter advance当たり一回のPhysics solveだけをProduction Qualification対象とし、current Substep Qualification subject／Receipt／Activation Binding集合はexact `[]`である。non-null Profileは`future.capability.alternate-simulation-cadence-and-substep`がactiveになり、選択する全`{Cadence Profile Ref, PhysicsSubstepProfileRef, Target Profile Ref}`へpassかつfreshな`PhysicsSubstepActivationBindingRefV1`を同じactivation transactionで登録し、Runtime Package、typed Save header、[Persistence／Save](../04-runtime/persistence-save.md)の`RuntimeReplayProjectionV1`／Domain Replay bindingが同じProfile／Binding Refへ閉じるまでplanning-onlyである。Activation前は`cadence_profile_not_qualified`を返し、暗黙のBackend既定substepへ変換しない。

2Dと3DのPhysics authorityは[World](../06-rendering/world.md)のexact `WorldSpaceProfileRefV1`から導く別Worldであり、同じEntityへ両dimensionのBodyを付与しない。`hybrid` Worldも一つの`hybrid_gameplay_space`だけをauthorityにし、二つのPhysics authorityを作らない。Body kindは`static | kinematic | dynamic`のclosed enumである。Visual scaleをnative Bodyへ渡さず、Collider geometryは[Collision](collision.md)のCooked Assetに焼き込む。finiteでない値、範囲外のmass／velocity、generation mismatchは明示failureにし、silent clampやnative defaultへのfallbackをしない。

World lifecycleは`uncreated | validating | ready | stepping | stop_requested | draining | destroyed | faulted`のEngine stateで表す。active WorldのProfile、worker class、solver semanticsをlive mutateしない。compatible changeはRuntime boundaryで新generationをactivateし、incompatible changeはPlay停止を要求する。native state名はpublic lifecycleへ露出しない。

### 2.2 Immutable state snapshot

`PhysicsStateSnapshotV1`は名前だけのprojectionではなく、T60でnormalize済みの一World／一Simulation Advanceを表す次のclosed schemaである。

```text
PhysicsBodySnapshotRefV1
  entity_stable_id: StableId
  body_id: StableId
  body_generation: positive uint64

PhysicsWorldProfileRefV1
  world_space_profile_ref: exact WorldSpaceProfileRefV1
  resolved_dimension: d2 | d3
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
  world_space_profile_ref: exact WorldSpaceProfileRefV1
  resolved_world_dimension: d2 | d3
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
  world_space_profile_ref: exact WorldSpaceProfileRefV1
  world_generation: positive uint64
  physics_world_profile_ref: PhysicsWorldProfileRefV1
  physics_world_profile_activation_generation: positive uint64
  snapshot_content_hash: SHA-256
```

snapshotはT60で全native jobをjoinした後、生成元のcompleted `SimulationAdvanceIntervalV1`からだけ構築する。`cadence_profile_ref`、`simulation_advance_interval_ref.cadence_profile_ref`、生成元Intervalの同Fieldはbyte equality、`advance_sequence`はRefと生成元Intervalの同Fieldへbyte equalityである。Refの`interval_content_hash`は生成元Intervalのself-excluding semantic `interval_content_hash`と一致させ、隣接する`simulation_advance_interval_sha256`はそのhash Fieldを含むcompleted Interval全canonical bytesのSHA-256と一致させる。semantic content hashとcompleted-object SHAを相互代用しない。World／World Space Profileのidentityとactivation generation、Substep Profile／BindingはそのIntervalを実行したactive runtime selectionと一致させ、`physics_world_profile_ref.world_space_profile_ref`、Snapshot／Refの同refはbyte equalityにする。`resolved_dimension`と`resolved_world_dimension`はそのexact World Profileから再計算するruntime projectionであり、Source authorityではない。T50途中のstate、別World、別Profile generation、別advanceの値を混在させない。

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

<a id="physics-capability-records"></a>

#### MCD Capability record closure

Physicsが公開するMCD Capabilityは次のexact二件である。Product `capability.simulation.physics-2d | physics-3d` rowやProvider Catalog rowとは別recordであり、Target別Activation Bindingが同じContract set rootへ接続する。materialized Contract setがない現在は設計候補で、表だけをcurrent refとして解決しない。

共通Envelopeは`mcd_version=1`、`kind=capability`、`version=1`、`status=active`、`owners=[owner.core.physics]`、`requirement_refs=[]`、`since_contract_set=2`、`supersedes=[]`である。Payloadは`maturity=C1`、`supported_targets=[target.android.mobile, target.apple.mobile, target.headless.host, target.windows.desktop, target.windows.editor]`、`conflicts=[]`、`authoring_types=[]`、`operations=[]`、`validators=[]`、`quality_profiles=[]`、`budgets=[]`、`examples=[]`、`ai_guidance=[]`を共通値とする。

| Capability ID | `title` | `description` | `required_capabilities[]` | `rationale_refs[]` | `tags[]` |
|---|---|---|---|---|---|
| `capability.simulation.physics_dynamics` | `Physics dynamics` | Body dynamics、force、constraint、solver advanceを提供する | `[capability.simulation.collision_response@1]` | `[mirakan.arch.simulation-physics#2-worldbody-dynamicscommand]` | `[dynamics, physics]` |
| `capability.motion_executor.physics_kinematic` | `Physics kinematic motion executor` | 検証済みNavigation batchをbounded Physics kinematic resultへ解決する | `[capability.simulation.collision_query@1]` | `[mirakan.arch.simulation-physics#31-kinematic-motion-reference-provider]` | `[kinematic_motion, motion_executor, physics, provider]` |

両recordの`failure_modes`は`[{diagnostic_code=MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED, fallback_id=fallback.capability.unavailable}]`である。`required_capability_refs`、Provider descriptor、Product BindingのID／version／Contract set hashを一致させ、`physics`等のgeneric ID、Backend名、Product maturityからMCD Refを補完しない。

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
| `requirement.physics.kinematic_motion_executor.resolve_batch` v1 | MCD `requirement` | adapted intent、Profile、Target、exact World Space Profile、Collision closureを検証してresolved motionを返す`must` |
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
statement=Each accepted batch is validated against adapted intent Profile Target exact World Space Profile and Collision closure before at most one resolved result is emitted
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

policyはaccepted setがexact `[type.navigation.adapted_motion_intent@1]`、Profile schema／hash、Target Profile、exact World Space Profile、Collision query availabilityであることを検証する。このsingletonへの変更によりreceipt-free `provider.engine.physics.kinematic_motion@1` content hash、Provider Catalog hash、RecordRefを先に更新し、その固定RecordRefと既存System Activation BindingからProvider Qualification subject／input closure／signed Receipt／Provider Activation Binding、Compile／Activation／Save／Replay closureを順に更新する。System／Provider Receipt／Activation BindingをProvider base recordまたはCatalog hashへ戻さない。owner固有proposalはNavigationのgeneric contribution envelopeとexact adapter recordを介し、Physics recordへraw source typeまたはowner固有type IDを追加しない。stale Catalog ref、stale provider content hash、System base／Activation Binding mismatch、fixture RecordRefへの置換をproduction qualificationで個別に拒否する。

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

SaveはEngine-owned World／Body／Joint／kinematic executor stateを§2.2の一つのcompleted `PhysicsStateSnapshotV1`として保存し、native serialization、pointer、Backend substep state、未commitのT50途中state、snapshotと並行する別state arrayを保存しない。`projection_content_hash`はASCII `MIRAKAN_PHYSICS_SAVE_STATE_V1`と同Fieldだけを除くclosed canonical bytesから計算する。完成Projectionは[Persistence／Save](../04-runtime/persistence-save.md#23-timebase-headerとbinding)のroot外`AuthoritativeSaveDomainBindingV1`によってexact completed Headerへ結ばれ、Headerのstate-owner Projection Ref集合に本Projectionのowner／Type／ID／version／content hashがexact一件存在しなければならない。Header／BindingをProjectionへ埋め戻さない。snapshotの`cadence_profile_ref`はHeaderの`simulation_cadence_profile_ref`、`simulation_advance_interval_ref`は`last_committed_simulation_advance_interval_ref`、`simulation_advance_interval_sha256`は`last_committed_simulation_advance_interval_sha256`、`physics_substep_activation_binding_ref`はHeaderの同Fieldとそれぞれbyte equalityにする。snapshotのSubstep Profile RefはBinding subjectのSubstep Refと一致させる。null branchではsnapshotのSubstep Profile／Bindingがともにnullで、outer advance一回solveの契約と一致しなければならない。

`collider_asset_refs[]`は`source_asset_id, source_asset_revision, source_asset_content_hash, artifact_ref`のcanonical byte順へstrict sortし、duplicateを拒否する。各`artifact_ref`はexact completed `CookedColliderAssetV1`へ解決し、解決済みartifactのSource Asset identity三Fieldを隣接三Fieldとbyte equalityにする。集合はsnapshotのWorld generationを復元するためactive World definitionが参照するCooked Collider Asset集合とexact set equalityにし、unused／missing ref、path、`latest`、別revision／content hashへの再解決を拒否する。65,537件目を切り詰めず`physics_save_capacity_exceeded`でSaveを不成立にする。

Loadはschema、toolchain lock compatibility、snapshot content hash、World／Substep Profile、Cadence、last Interval semantic hash／completed SHA、Collider Asset identity、finite value、generation relationを検証してstaging Worldを構築し、fixture validation後にcompatible Simulation Advance boundaryで置換する。Header／Profile／Interval／Binding／Projection／snapshot不一致をcurrent値、Backend default、fixed 60/1へ読み替えず、明示migrationまたは`physics_save_incompatible`で拒否する。失敗時はactive Worldを維持する。

ReplayはRuntime ownerへnormalized command、accepted async input、Profile／artifact identity、state hash、snapshot projectionを供給する。Physicsのreceipt-free Replay Projectionは[Persistence／Save](../04-runtime/persistence-save.md#51-replay-transport-binding)の`RuntimeReplayDomainProjectionRefV1`として一意に参照し、完成`RuntimeReplayProjectionV1`とのroot外`RuntimeReplayDomainBindingV1`を持つ。Persistence OwnerだけがBindingを`RuntimeReplayBundleManifestV1`へmembershipとして閉じ、Header／binding／Bundle refをProjection base recordへ埋め戻さない。Cadence／Substep Binding、sealed Interval、Physics projectionの一致をPersistence validationで検証する。記録slotは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)のcanonical `T100_ReplayCheckpoint`だけを参照し、Physics固有phaseを設けない。

主要failure classはinvalid profile、unqualified adapter、handle／generation mismatch、command conflict、joint frame invalid、kinematic depenetration failure、native invariant violation、job drain failure、save incompatibilityである。Simulation Advance publish、fault transition、recovery boundaryはRuntime ownerへ委譲する。共通memory、worker、queue、frame thresholdをここで再定義しない。

Qualificationは全private Backendへ同じWorld lifecycle、stack、sleep／wake、joint、break、C1 reference Providerのslope／stair／kinematic support surface（斜面際のstep、ceilingに接した状態のslide、移動support bodyからの離脱、狭所でのdepenetration発振を含む）、save／load、replay hash、fuzz、fault injectionを与える。Engine contractの結果、ordering、diagnostic、lifetimeが一致することを検査する。Substep fixtureはnull Profileでouter advance当たりsolve一回、count 2／3／16で全exact rational subintervalの和がouter durationと一致しordinal順、T60／publishがouter advance当たり一回であることを検査する。`denominator*(C/g)`が`uint32`最大値ちょうどに収まる境界と1超過する境界を含め、後者だけを`physics_substep_interval_overflow`で副作用前に拒否する。自己Ref埋込み、dimension欠落／duplicate／逆順、count 0／17、null duration、Cadence／interval hash／Target／Qualification差、substep単位のInput sample／Timer／event delivery／publishを各一原因で拒否する。`fixture.navigation.motion-executor.physics-kinematic@1`はreceipt-free Provider record／Catalogを先に固定し、そのexact RecordRef、implementation System Activation Binding、Target、adapted singleton accepted setを`qualification.navigation.motion-executor.physics-kinematic@1` subjectへbindする。signed Receiptはsubjectだけを署名し、Provider Activation BindingがRecordRef＋Receipt refを保持する。positive batchは`adapted_intent_schema_ref=type.navigation.adapted_motion_intent@1`でProvider call exact一回、raw `type.navigation.movement_intent@1`または`type.navigation.motion_intent_contribution@1`へ一Fieldだけ変えた二fixtureはProvider call前にrejectする。accepted setへのraw型追加、RecordRef／Qualification subject／Receipt／Activation Binding hash staleもrejectし、last-valid resolved motion、Path Follower state、Catalogを不変にする。別fixtureはPhysics capability unavailableでも任意Packのinstallが成功し、Physics Provider選択だけがunavailableになることを検証する。Dependency build、exact binary identity、license、target matrixは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、測定とcapacity promotionは[Runtime performance／capacity](../04-runtime/performance-capacity.md)が所有する。

Snapshot fixtureは同じcompleted Intervalから作ったnormalized body／joint／kinematic stateとself-excluding hashをpositive caseとし、Cadence、Interval Ref semantic hash、completed-object SHA、advance sequence、World generation、World Profile identity／activation generation、Substep Binding、array order、body／ground ref generation、snapshot hashを各一Fieldだけ変えたcaseを`physics_snapshot_stale`またはschema failureでpublish／consume前に拒否する。各arrayは上限ちょうどを受理し一件超過を`physics_snapshot_capacity_exceeded`で拒否し、partial snapshot、`latest`再解決、cross-advance rebaseを0件とする。

Save fixtureは同じsnapshotとCollider Artifact集合をDomain BindingでHeaderへ結ぶcaseを受理し、HeaderのCadence／Interval Ref／completed SHA／Substep Binding、snapshot hash、Collider Source identity／Artifact bytes、Collider集合順／set equalityを各一Fieldだけ変えたcaseを`physics_save_incompatible`でLoad staging前に拒否する。Collider arrayは65,536件ちょうどをschemaとして受理し、65,537件目を`physics_save_capacity_exceeded`でSave生成前に拒否する。

Replay fixtureは[Persistence／Save](../04-runtime/persistence-save.md#51-replay-transport-binding)の`RuntimeReplayDomainProjectionRefV1`、`RuntimeReplayDomainBindingV1`、`RuntimeReplayBundleManifestV1`を受理し、Replay projectionのCadence／Substep Binding／sealed Interval／digest、Bundle membership、Binding hashを各一Fieldだけ変えたcase、Projection base recordへのbinding／Bundle埋戻し、別Replay／Projectionを結ぶbinding、Bundleのmissing／extraをReplay開始前に拒否する。

### 4.1 Private Backend optimization eligibility

次表は[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)が固定するexact Box2D 3.1.1／Jolt 5.6.0へ適用するprivate Adapter候補の適格性条件である。設定値、worker数、allocator容量を全Target共通defaultとして固定せず、Backend release／build option、Target Profile、World／solver／sleep profile、fixture、input traceをcandidate identityへ含める。

| Backend／候補 | 適格性条件 | 禁止／fail-closed条件 |
|---|---|---|
| Box2D task system | `b2WorldDef.workerCount`と`enqueueTask`／`finishTask`をTarget別のclosed worker candidate集合から評価する。single-thread referenceと各候補へ同じWorld traceを与え、全taskをWorld境界前にjoinする | callbackからallocation、logging、Gameplay／AI／Asset呼出し、World mutationを行うこと。CPU名からworker数を推測すること |
| Box2D sleep | World／Body semanticsがsleep／wakeを許すProfileで`enableSleep`の`true`と`false`を別candidateとして測る。resting bodyが多い場合の省力化と、sleep不要Profileで管理costを外す場合の双方を実測する | wake／sleep event、public state、command acceptance、Replay hashを変えること。高速化を理由にProject intentを暗黙変更すること |
| Box2D body creation | BodyはEngine staging commandが確定したfinal transformで作成し、同一activationでoriginに生成してからtransformを移動する手順を候補にしない | Box2Dに存在しないbulk insertion APIを仮定すること、未検証bodyをlive Worldへ部分追加すること |
| Jolt job／temporary memory | Engine-owned `JobSystem` worker候補と`TempAllocator`容量をTarget Profileからexactに選び、job wait、temporary high-water、overflow／OOMを記録する | 未bounded allocator、general heap fallback、job未join、worker callbackから上位層を呼ぶこと |
| Jolt body insertion | 同じactivation transactionで複数Bodyを追加する経路は`BodyInterface::AddBodiesPrepare`／`AddBodiesFinalize`のbatchを使うcandidateを基準とし、single-body commandだけを一件追加経路にする | 複数Bodyを一件ずつ追加してBroadPhase更新を反復すること、Prepare成功後のpartial publish |
| Jolt BroadPhase | 初期BroadPhase layerは少なくともstatic／dynamicを分離し、追加layerはCollision Profileで表現できず実測改善する場合だけ候補にする。`OptimizeBroadPhase`はbulk初期配置後に公式条件上必要な場合だけ評価する | `OptimizeBroadPhase`を毎frame呼ぶこと、適切なbatch insertion後も無条件に呼ぶこと、layerをGameplay categoryの代用にすること |
| Jolt／Box2D sleep共通 | active／sleeping body比率、island／activation、step P50／P95／P99、event normalizationを同時に測る | step平均だけで昇格すること、sleeping stateをSave／Replay外のnative既定へ委ねること |

Collision pairを可能な限り早く拒否する意味とstageは[Collision §3](collision.md#3-materialfiltersensor)が所有し、Physicsはそのresolved filter planをBackendのobject／broadphase／pair filterへ写像するだけである。JoltのContactListenerによるrejectはlateかつexpensiveな経路として計測し、early filter改善に数えない。

全候補はEngine-owned state snapshot、normalized event順、Save／Load、Replay hash、failure／lifetime oracleをreferenceと一致させる。performance metricはstep P50／P95／P99、job wait、active／sleep body数、body insertion throughput、temporary allocator high-water／failure、filter stage別pair数を必須にし、共通比較とpromotionは[Runtime performance／capacity §8.4](../04-runtime/performance-capacity.md#84-algorithm-optimization-candidate-qualification)へ従う。AI／Editorは同節の`OptimizationDecisionProjectionV1`だけをread／explainし、raw World、Vendor setting／object、worker callback、full trace、candidate selectionを操作しない。optimization propose／select Operationは現在登録せず、本書§6のOperation集合を増やさない。

Box2D公式のtask system／sleep／body creation根拠は[Simulation documentation](https://box2d.org/documentation/md_simulation.html)、Jolt公式のbatch insertion／BroadPhase／JobSystem／sleep根拠は[Jolt Physics 5.6.0 documentation](https://jrouwe.github.io/JoltPhysicsDocs/5.6.0/)とする。reference pathをoracleまたは明示的に別Qualificationされたsemantic fallbackとして保持できるが、public Vendor型、custom solver、旧／新経路の暗黙併載、deprecated alias、runtime自動Backend切替は採用しない。

## 5. AI semantics

詳細は[physics-ai-catalog-proposal](../appendices/physics-ai-catalog-proposal.md#5-ai-semantics)へ分離した。本節はnavigationだけを持ち、Catalog／Fixture定義を複写しない。
