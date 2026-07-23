# Miraikanai Engine Physics Contract

- 文書ID: mirakan.arch.simulation-physics
- 状態: review
- 正本範囲: Physics World／Body dynamics、solver profile semantics、command、joint／constraint、generic Kinematic Motion reference Provider、kernel Adapter boundary、Physics save／replay projection、Physics AI intent／discovery／resolution／preview／diagnostic／eval
- 非正本範囲: Collider geometry／filter／query／event、Runtime phase／tick／capacity、Animation pose、Navigation artifact、external dependency version／build pin、AI authorization／evidence envelope。各Owner文書を参照する
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

`PhysicsWorldOwner`だけがnative WorldとEngine handle mappingを所有する。native pointer、lock、callback view、collectorはfunction／job／tick境界を越えて保持しない。callbackはpreallocated local bufferへcopyするだけで、allocation、logging、World mutation、Gameplay dispatchを行わない。全native jobをjoinし、normalizeが完了した後だけRuntimeへ結果を渡す。

## 2. World、Body dynamics、command

| Object | 意味 | persistence |
|---|---|---|
| `PhysicsWorldDocumentV1` | Authoring上のWorldとProfile参照 | Project source |
| `Physics2DWorldProfileV1` | 2D gravity、solver semantics、worker class、Collision profile ref | Project source |
| `Physics3DWorldProfileV1` | 3D gravity、solver semantics、worker class、Collision profile ref | Project source |
| `PhysicsBody2DComponent`／`PhysicsBody3DComponent` | motion kind、mass source、damping、sleep／motion policy、Collider ref | World source |
| `KinematicMotionProfileV1` | actor collider ref、max slope、step height、ground snap距離、slide／step iteration上限、speed上限 | Project source |
| `PhysicsWorldHandle`／`PhysicsBodyHandle` | Engine generation handle | Runtime only |
| `PhysicsStateSnapshotV1` | normalized transform、velocity、sleep、joint／kinematic-executor state | immutable tick snapshot |
| `PhysicsSaveStateV1` | Engine-owned recoverable state | Save stream |

2Dと3Dは別Worldであり、同じEntityへ両dimensionのBodyを付与しない。Body kindは`static | kinematic | dynamic`のclosed enumである。Visual scaleをnative Bodyへ渡さず、Collider geometryは[Collision](collision.md)のCooked Assetに焼き込む。finiteでない値、範囲外のmass／velocity、generation mismatchは明示failureにし、silent clampやnative defaultへのfallbackをしない。

World lifecycleは`uncreated | validating | ready | stepping | stop_requested | draining | destroyed | faulted`のEngine stateで表す。active WorldのProfile、worker class、solver semanticsをlive mutateしない。compatible changeはRuntime boundaryで新generationをactivateし、incompatible changeはPlay停止を要求する。native state名はpublic lifecycleへ露出しない。

### 2.1 Dynamics command

`PhysicsDynamicsCommandV1`はtarget handle／expected generation、consume tick ref、producer metadata、priority、tagged payloadを持つ。payloadは次のclosed command familyである。

- force／torque／impulse
- bounded linear／angular velocity assignment
- kinematic target／teleport
- wake／sleep permission／gravity factor
- joint motor target／limit／break request

同一Bodyへのcommandは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)のcanonical merge keyを消費する。Teleportと通常motion commandの競合、generation不一致、wrong body kind、wrong dimensionはtyped conflictとして全体を拒否する。Force、Impulse、velocityを暗黙変換しない。command arena、queue、overflow値をPhysicsで持たない。

Physics executionはRuntimeのcanonical identifiers `T30_PrePhysics`、`T40_MotionIntent`、`T50_PhysicsStep`、`T60_PhysicsIntegrate`、`T70_PostPhysics`への参照で接続する。正確な順序、writer、tick frequencyは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md#4-60-hz-fixed-tickとphase-identifier)だけが決定し、本書はphase tableを再掲しない。

## 3. Joint、Constraint、optional Physics Kinematic Motion Provider

`PhysicsJointCommonV1`はjoint Stable ID、World ref、Body A／Bまたはtyped World Anchor、enabled、collide-connected、local frame、optional break semanticsを持つ。Joint kindはtagged unionであり、存在しないfieldをproperty bagへ入れない。World Anchorは専用variantで、null Bodyやmagic handleで代用しない。

2D C1 familyはdistance、revolute、prismatic、weld、3D C1 familyはfixed、point、distance、hinge、slider、swing-twistを持つ。各familyはSI単位、normalized axis、orthogonal frame、ordered limit、finite motor targetを検証する。Vendor enum値、constraint pointer、reaction callbackは保存しない。新familyはCapability、schema、Editor、AI vocabulary、fixtureを同時に追加する。

Joint break候補はAdapter結果をSI単位へnormalizeし、Engine Stable IDとtick refを持つ`JointBreakEventV1`へ変換する。配送と次boundaryのcomponent removalはRuntime ownerの順序を消費する。Backend間のreaction値へbitwise一致は要求せず、fixtureで許容されるsemantic rangeを検査する。

### 3.1 Kinematic Motion reference Provider

C1 reference recipeはEngine-owned Kinematic Motion Providerを適格化するが、任意Packのinstall、Path Following、Runtime EntryはこのProviderを要求しない。Backend固有controllerをProject APIへ公開せず、[Collision](collision.md)のoverlap／shape castだけを利用する。

Provider-private `KinematicMoveIntentV1`はactor handle、consume tick ref、planar displacement、vertical proposal、edge-triggered motion flags、up direction、producer metadataを持つ。これはaccepted public intentではなく、Navigation `MovementIntentV1`とowner登録済み`MotionIntentContributionV1`を検証後にPhysics Provider内部で生成するderived inputである。Port、Project Source、Save、Replayへ型参照を公開せず、producer固有proposalを内部Fieldへ複写しない。`PhysicsKinematicResolvedMotionV1`はresolved pose／velocity、state、ground handle／generation／normal／relative point、platform delta、hit summary、diagnostic、input batch hash、generationを持つ。

`capability.motion_executor.physics_kinematic`はPhysics Providerが提供する正式Capabilityであり、次のexact 7-Field descriptorを[Navigation](navigation.md)が所有する`MotionExecutorProviderCatalogV1`へproduction recordとして登録する。Port型、transport batch、contribution registry、Provider Catalogを本書で再定義しない。全MCD参照は表のID、`version=1`、選択Contract set hashを持つ`McdContractRefV1`である。

| `executor_capability_ref.id` | `movement_profile_schema_ref.id` | `accepted_intent_schema_refs[].id` | `transport_message_schema_ref.id` | `resolved_motion_schema_ref.id` | `compatibility_predicate_ref.id` | `failure_diagnostic_refs[]` |
|---|---|---|---|---|---|---|
| `capability.motion_executor.physics_kinematic` | `type.physics.kinematic_motion_profile` | `[type.navigation.movement_intent, type.navigation.motion_intent_contribution]` | `type.navigation.motion_executor_intent_batch` | `type.physics.kinematic_resolved_motion` | `policy.physics.kinematic_motion_intent_profile_target_dimension` | `[MIRAKAN-PHYSICS-KINEMATIC-MOTION-INCOMPATIBLE, MIRAKAN-PHYSICS-KINEMATIC-MOTION-RESOLUTION_FAILED, MIRAKAN-PHYSICS-KINEMATIC-MOTION-STALE_RESULT]` |

production recordは`provider_id=provider.engine.physics.kinematic_motion`、`provider_version=1`、self-excluding content hash、Engine Physics componentのexact owner ref／hash、`usage=production`、implementation System ref／hash、Target Profile集合、Qualification Receipt集合を持つ。Compile／Activation／Batch／Save／Replayが使用するidentityは次のNavigation-owned RecordRefだけである。

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

policyはintent type subset、Profile schema／hash、Target Profile、2D／3D dimension、Collision query availabilityを検証する。owner固有proposalはNavigationのgeneric contribution envelopeとexact adapter recordを介し、Physics recordへowner固有type IDを追加しない。stale Catalog ref、stale provider content hash、fixture RecordRefへの置換をproduction qualificationで個別に拒否する。

T40のgeneric contribution resolverはNavigationとowner登録済みproposalをNavigation-owned `MotionExecutorIntentBatchV1` Port messageとして提出し、選択済みPhysics Kinematic Motion Providerがentriesのaccepted schemaを一度だけ解決する。全proposalはregistryで一意に選んだadapterを経由し、Providerへ直接提出しない。Provider-private `KinematicMoveIntentV1`はこの検証済みbatchからだけderiveし、Portのpublic accepted setへ混ぜない。`MovementIntentV1`の`desired_velocity`は[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)のfixed tick deltaを乗算してplanar displacementへ変換する。同一tickの複数proposalは各adapter policyとcanonical producer順で解決し、不採用のintentをtyped resultとしてproducerへ返す。合成優先順位をProvider内で暗黙に変更しない。

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

SaveはEngine-owned World／Body／Joint／kinematic executor state、Profile identity、Collider Asset identityを保存し、native serializationやpointerを保存しない。Loadはschema、toolchain lock compatibility、Asset identity、finite value、generation relationを検証してstaging Worldを構築し、fixture validation後にcompatible boundaryで置換する。失敗時はactive Worldを維持する。

ReplayはRuntime ownerへnormalized command、accepted async input、Profile／artifact identity、state hash、snapshot projectionを供給する。記録slotは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)のcanonical `T100_ReplayCheckpoint`だけを参照し、Physics固有phaseを設けない。Replay environmentとdebug streamは[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)を消費する。

主要failure classはinvalid profile、unqualified adapter、handle／generation mismatch、command conflict、joint frame invalid、kinematic depenetration failure、native invariant violation、job drain failure、save incompatibilityである。tick publish、fault transition、recovery boundaryはRuntime ownerへ委譲する。共通memory、worker、queue、frame thresholdをここで再定義しない。

Qualificationは全private Backendへ同じWorld lifecycle、stack、sleep／wake、joint、break、C1 reference Providerのslope／stair／kinematic support surface（斜面際のstep、ceilingに接した状態のslide、移動support bodyからの離脱、狭所でのdepenetration発振を含む）、save／load、replay hash、fuzz、fault injectionを与える。Engine contractの結果、ordering、diagnostic、lifetimeが一致することを検査する。Provider fixtureはNavigationのexact `MotionExecutorPortV1`へのbinding、intent／profile／Target compatibility、stale result、generic contribution、provider failure時のlast-valid resolved motion不変を含む。別fixtureはPhysics capability unavailableでも任意Packのinstallが成功し、Physics Provider選択だけがunavailableになることを検証する。Dependency build、exact binary identity、license、target matrixは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、測定とcapacity promotionは[Runtime performance／capacity](../04-runtime/performance-capacity.md)が所有する。

## 5. AI semantics

Physics AI surfaceは自然言語を直接Body fieldへ投影せず、`intent -> discovery -> questions／assumptions -> semantic resolution -> typed operation -> preview -> validation`を一つのbounded pipelineとして扱う。RuntimeやVendor APIはAI surfaceではない。

### 5.1 Intentとclosed vocabulary

`PhysicsIntentVocabularyEntryV1`は自然言語の単語をOperationへ直接bindする辞書ではなく、Game文脈から意味候補を絞るversion付きCatalog entryである。mandatory fieldを次へ固定する。

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
  fixture_refs[1..64]
  role_content_hash: SHA-256

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
  selected_axis_closure_hash
  selection_content_hash
```

Core初期roleは次のbehavior-neutralなexact六recordだけである。全Refはversion 1、全ownerは`owner.core.physics`のexact revision／content hash、Capabilityとfixtureはversion／Contract set rootまたはcontent hash付きで保存する。

| role ID | allowed motion | allowed collision | allowed hit | allowed shape | allowed speed | required Capability | fixture |
|---|---|---|---|---|---|---|---|
| `role.physics.static_environment` | `static` | `solid_block; query_only` | `solver_contact; swept_shape_query; overlap_query; none` | `primitive; compound_primitive; convex; static_triangle_mesh; heightfield; tile_chain_2d` | `discrete` | `capability.simulation.collision` | `fixture.physics.intent-role.static-environment` |
| `role.physics.dynamic_body` | `dynamic_solver` | `solid_block; sensor_notify` | `solver_contact; sensor_event; none` | `primitive; compound_primitive; convex` | `discrete; continuous_body` | `capability.simulation.physics_dynamics` | `fixture.physics.intent-role.dynamic-body` |
| `role.physics.kinematic_body` | `kinematic_target` | `solid_block; sensor_notify` | `solver_contact; sensor_event; swept_shape_query; none` | `primitive; compound_primitive; convex` | `discrete; authoritative_sweep; teleport` | `capability.simulation.collision` | `fixture.physics.intent-role.kinematic-body` |
| `role.physics.sensor` | `static; kinematic_target` | `sensor_notify` | `sensor_event; overlap_query; none` | `primitive; compound_primitive; convex; tile_chain_2d` | `discrete; teleport` | `capability.simulation.collision_query` | `fixture.physics.intent-role.sensor` |
| `role.physics.query_subject` | `query_driven` | `query_only` | `swept_shape_query; overlap_query; gameplay_rule; none` | `primitive; compound_primitive; convex; tile_chain_2d` | `discrete; authoritative_sweep; teleport` | `capability.simulation.collision_query` | `fixture.physics.intent-role.query-subject` |
| `role.physics.presentation_proxy` | `presentation_only` | `none` | `none` | `none` | `discrete; teleport` | `capability.presentation.transform_proxy` | `fixture.physics.intent-role.presentation-proxy` |

recordsはrole ID／version順へstrict sortし、exact duplicate、同一ID／versionの別content hash、同じrole IDのactive record複数、owner namespace偽装、非canonical orderを拒否する。`role_content_hash`はASCII `MIRAKAN_PHYSICS_INTENT_ROLE_RECORD_V1`と、当該hash Fieldだけを除くRecord canonical MCD bytesを`uint32_be` length framingしてSHA-256する。Recordはhashを含まないidentity Fieldをpreimageに持ち、外部`PhysicsIntentRoleRefV1`をrecord内へ埋め戻さない。Registry hashはASCII `MIRAKAN_PHYSICS_INTENT_ROLE_REGISTRY_V1`、Registry ID／version、record count、全record canonical bytesを各`uint32_be` length framingしてSHA-256し、`registry_content_hash`自身を除外する。`PhysicsIntentRoleRegistryRefV1`は三Fieldすべてを同一active Registryへexact解決し、ID-only、latest version、hash fallbackを許可しない。

Pack／Projectはowner namespace、exact Capability、axis compatibility、fixtureを持つrecordを下向きに登録できる。object vocabulary、具体例、default mappingはcontributorが所有し、Core resolver、Core vocabulary、Core fixture inventoryへコピーしない。role refは候補検索を助ける分類であり、motion／collision／hit／shape／speed各axisの検証を省略または上書きしない。

Project Sourceの選択正本は`PhysicsIntentRoleSelectionDocumentV1`であり、RegistryとResolutionは派生である。`operation.physics.intent_role.select@1`だけがexpected Project revision、Selection Document before ref／hash、subject definition、Role Registry ref、selected RoleRef、五axis closure、Preview／Validation、`MutationAuthorizationBindingV2`のR2 ApprovalまたはPredelegationを受け、Prepared Candidate経由で変更する。binding欠落、expired、Scope／request hash不一致はexact `diagnostic.approval.required / MIRAKAN-APPROVAL-REQUIRED`で拒否する。Compile closure、Save、ReplayはSelection Document ref／hash、RegistryRef、RoleRef、axis closure hashを保存し、reload時にSource→Registry→record→Capability／fixtureを再検査する。

`PhysicsIntentResolutionV1`のmandatory schemaは次である。fieldの省略、任意propertyの追加、closed value以外の文字列を拒否する。

```text
PhysicsIntentResolutionV1
  source_request_ref: ContentRef
  source_request_hash: Sha256
  contract_set_hash: Sha256
  project_revision: RevisionId
  target_profile_ids: TargetProfileId[1..16]
  physics_role_registry_ref: PhysicsIntentRoleRegistryRefV1
  scene_dimension: two_d | three_d | hybrid
  hybrid_gameplay_space: optional two_d | three_d
  physics_role_ref: PhysicsIntentRoleRefV1
  motion_authority: PhysicsMotionAuthorityV1
  collision_semantics: PhysicsCollisionSemanticsV1
  hit_authority: PhysicsHitAuthorityV1
  shape_strategy: PhysicsShapeStrategyV1
  speed_policy: PhysicsSpeedPolicyV1
  selected_capability_ids: CapabilityId[]
  selected_operation_ids: OperationId[]
  blocking_question_ids: QuestionId[]
  assumptions: AssumptionRecordV1[]
  rejected_alternatives: RejectedAlternativeV1[]
  diagnostic_ids: DiagnosticId[]
  preview_fixture_ids: FixtureId[]
  cost_estimate_ref: optional CostEstimateRef
  disposition: ready_to_propose | question_required | capability_unavailable | rejected
```

`source_request_ref`はAuthoring Task内のaccess-controlled contentを参照し、raw PromptをCatalog、MCD、Receiptへ複製しない。`physics_role_registry_ref`は候補生成時に読んだactive Registryを固定し、validate／preview／proposal時にcurrent Registry refと三Fieldexact equalityで再検査する。`ready_to_propose`は既存OperationでChangeSetを提案できる意味だけを持ち、Commit authorizationを意味しない。`question_required`はblocking ambiguityが残る結果、`capability_unavailable`は要求を満たすactive Capabilityがない結果、`rejected`はinvalid／forbiddenな要求である。

role以外のclosed semantic axisを次へ固定する。一つのResolutionは各軸から一つだけを選び、role recordのallowed setと照合する。

| Type | Closed values |
|---|---|
| `PhysicsMotionAuthorityV1` | `static \| kinematic_target \| dynamic_solver \| query_driven \| presentation_only` |
| `PhysicsCollisionSemanticsV1` | `solid_block \| sensor_notify \| query_only \| none` |
| `PhysicsHitAuthorityV1` | `solver_contact \| sensor_event \| swept_shape_query \| overlap_query \| gameplay_rule \| none` |
| `PhysicsShapeStrategyV1` | `primitive \| compound_primitive \| convex \| static_triangle_mesh \| heightfield \| tile_chain_2d \| none` |
| `PhysicsSpeedPolicyV1` | `discrete \| continuous_body \| authoritative_sweep \| teleport` |

旧`GameplayPhysicsRoleV1` enumはoffline migration inputだけである。owner固有変換は次のContribution Registryへ登録し、Core switch文へ追加しない。

```text
PhysicsIntentRoleMigrationContributionRefV1
  contribution_id
  contribution_version
  contribution_content_hash

PhysicsIntentRoleMigrationContributionRecordV1
  contribution_id
  contribution_version
  owner_ref/hash
  source_schema_ref: McdContractRefV1(kind=type, version=1)
  accepted_legacy_values[1..64]
  destination_role_refs[1..64]: PhysicsIntentRoleRefV1
  mapping_policy_ref: McdContractRefV1(kind=policy)
  axis_mapping_policy_ref: McdContractRefV1(kind=policy)
  fixture_refs[1..64]
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
  validator_closure_ref: OperationValidatorClosureRefV1
  contribution_registry_ref: PhysicsIntentRoleMigrationContributionRegistryRefV1
  trusted_service_ref: TrustedServiceRefV1(
    service_id=service.offline_project_migrator, service_version=1,
    service_content_hash)
  trusted_service_allowlist_operation_local_refs[1]:
    ContractSetLocalRefV1(
      kind=operation, id=operation.physics.intent_role.migrate, version=1)
  diagnostic_refs[15]: DiagnosticCodeRefV1
  fixture_refs[1..64]: exact {fixture_id, fixture_version, fixture_content_hash}
  manifest_hash: SHA-256
```

Core contributionは`world_static→role.physics.static_environment`、`movable_prop→role.physics.dynamic_body`、`sensor_volume→role.physics.sensor`の三mappingだけを持つ。object固有legacy値は当該Pack／Project contributionがexact一件存在する場合だけ移行する。recordはContribution ID／version順、accepted valueはUTF-8 byte順、destination refはrole ID／version順へstrict sortし、duplicate valueの複数active contribution、stale owner／policy／role／fixture hashをRegistry全体で拒否する。record hashはASCII `MIRAKAN_PHYSICS_ROLE_MIGRATION_CONTRIBUTION_V1`、Registry hashはASCII `MIRAKAN_PHYSICS_ROLE_MIGRATION_CONTRIBUTION_REGISTRY_V1`のself-excluding length-framed canonical bytesである。

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
  operation_ref
  before_project_ref
  request_hash
  idempotency_key
  source_document_ref/hash
  source_schema_ref/hash
  source_legacy_value
  source_axis_closure_hash
  destination_role_registry_ref
  contribution_registry_ref
  selected_contribution_ref
  preview_policy_ref
  validation_policy_ref
  authorization_ref/hash
  mutation_authorization_binding: approval

PhysicsIntentRoleMigrationResultV1
  disposition: migrated | capability_unavailable | ambiguous | rejected
  migrated:
    after_project_ref
    selection_document_ref/hash
    selected_role_ref
    axis_closure_hash
    migration_receipt_ref/hash
    atomic_commit_marker_ref/hash
  non-migrated:
    diagnostics[1..15]

PreparedPhysicsIntentRoleMigrationReceiptPayloadV1
  operation_ref
  request_hash
  idempotency_key
  source_document_ref/hash
  source_schema_ref/hash
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
  diagnostics[0..15]
  prepared_payload_hash

PhysicsIntentRoleMigrationReceiptV1
  prepared_payload_ref/hash:
    PreparedPhysicsIntentRoleMigrationReceiptPayloadV1
  atomic_commit_marker_ref/hash
  signer_identity_ref/hash
  signature
  receipt_hash
```

Domain固有Diagnosticは次のexact Registry recordである。全rowは`diagnostic_version=1`、`message_key="<diagnostic_id>.message"`、self-excluding `diagnostic_content_hash`を持ち、共通八件はExecutable Contractsの同一recordを参照する。

| Diagnostic ID | code | severity／category／retryability |
|---|---|---|
| `diagnostic.physics.intent_role.source_invalid` | `MIRAKAN-PHYSICS-INTENT-ROLE-SOURCE-INVALID` | blocking／schema／after_input |
| `diagnostic.physics.intent_role.registry_invalid` | `MIRAKAN-PHYSICS-INTENT-ROLE-REGISTRY-INVALID` | blocking／schema／after_change |
| `diagnostic.physics.intent_role.contribution_missing` | `MIRAKAN-PHYSICS-INTENT-ROLE-CONTRIBUTION-MISSING` | blocking／semantic／after_change |
| `diagnostic.physics.intent_role.contribution_ambiguous` | `MIRAKAN-PHYSICS-INTENT-ROLE-CONTRIBUTION-AMBIGUOUS` | blocking／semantic／after_input |
| `diagnostic.physics.intent_role.capability_unavailable` | `MIRAKAN-PHYSICS-INTENT-ROLE-CAPABILITY-UNAVAILABLE` | blocking／capability／after_change |
| `diagnostic.physics.intent_role.axis_mapping_invalid` | `MIRAKAN-PHYSICS-INTENT-ROLE-AXIS-MAPPING-INVALID` | blocking／semantic／after_input |
| `diagnostic.physics.intent_role.receipt_binding_mismatch` | `MIRAKAN-PHYSICS-INTENT-ROLE-RECEIPT-BINDING-MISMATCH` | blocking／semantic／after_change |

`validator_closure.operation.physics.intent_role.migrate@1`は次のexact Validator recordを持つ。各recordはversion 1、実装Artifact ref／hash、表のinput Type ref、表のDiagnostic ref集合、self-excluding content hashを持ち、ID／version順にsortする。

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

request hashはExecutable Contractsの唯一のV2式、Prepared payload hashはASCII `MIRAKAN_PREPARED_PHYSICS_INTENT_ROLE_MIGRATION_RECEIPT_PAYLOAD_V1`、外側Receipt hashはASCII `MIRAKAN_PHYSICS_INTENT_ROLE_MIGRATION_RECEIPT_V1`とPrepared payload ref／hash、Atomic Commit Marker ref／hash、signer／signatureを使う。MarkerはPrepared payloadだけをpublish集合へ含め、外側Receiptを含めない。同じidempotency key＋request hashのretryはbyte-identical Result／Receiptを返し、同じkey＋別requestはidempotency reuse errorでSource不変にする。Validator error union、Operation `errors[]`、Manifest `diagnostic_refs[]`は15 refのset equalityにする。Manifestは上記Operation／Type／Policy／Validator／Registry／Diagnostic／fixtureのexact version／hashを全件持ち、missing／extra／staleをcompile前に拒否する。ManifestのOperation LocalRef集合と`service.offline_project_migrator`へのallowlist contributionはexact一件でset equalityとし、同じContract set transactionでService local recordとset rootを再生成する。未導入Capabilityの旧予約値は`capability_unavailable`でSourceを不変にし、Core active roleへ近似変換しない。current serializer／AI projectionは旧enum値を受理しない。

複合objectは複数Resolutionと明示関係で表し、合成enumを追加しない。同じobjectへ複数motion authorityを選ばない。Dynamic Bodyへ`static_triangle_mesh`／`heightfield`を選ばず、Sensorをauthoritative hitへ暗黙昇格せず、`teleport`を経路hitの代用にしない。途中経路がGameplayへ必要なら`authoritative_sweep`を使用する。

### 5.2 Capability discoveryと意味解決

Resolverは[Executable contracts](../02-foundation/executable-contracts.md)のCapability registryから、scene dimension、active maturity、Target、World Profile、Collision capability、authoring permission、Physics Intent Role Registryを読み、利用可能なEngine operationだけを提示する。Backend featureをCapabilityとして直接表示しない。unknown／removed／reserved-unsupported role、required Capability不足は`capability_unavailable`とし、近いCore roleまたは既存operationへsilent downgradeしない。

解決順は次である。

1. source requestのcontent hash、Project revision、Contract set hash、Target Profile、active Physics Intent Role Registry refを取得し、Resolutionへ観測値としてbindする。
2. ユーザー文からowner vocabulary候補と、motion、contact、hit、shape、speed、dimensionの独立候補を抽出する。
3. Scene／Projectの既存World、Body、Collider、Profile、Capabilityをread-only discoveryする。
4. exact owner role refとclosed axisへ候補を割り当て、矛盾と欠落を分類する。
5. gameplay behaviorを変える欠落だけを`blocking_question_ids`へ入れ、`question_required`にする。安全な欠落だけをReference assumptionとして明示する。
6. role ref、motion authority、collision semantics、hit authority、shape strategy、speed policyを独立に確定する。
7. Target、Capability、Profile、field relation、forbidden mappingを同じValidatorで再検査する。対応不能は`capability_unavailable`、invalid／forbiddenは`rejected`にする。
8. `ready_to_propose`の場合だけtyped Physics／Collision write OperationとPreview fixtureを返す。
9. GatewayがProvider出力を同じMCDとValidatorで再計算し、不一致をresolution mismatchとして拒否する。

Resolutionのvalidate、preview、operation proposalの各入口は、現在のsource request hash、Contract set hash、Project revision、Physics Intent Role Registry refを保存済み`source_request_hash`、`contract_set_hash`、`project_revision`、`physics_role_registry_ref`と完全一致で再検査する。一つでも異なるResolutionは`stale`として拒否し、selected operation、preview、cost estimateを使用しない。最新source／contract／Project／Registry snapshotでCapability discoveryから再解決し、新しいResolution identityを発行する。stale objectをfield単位で更新、別revisionへrebase、Commitへ継続してはならない。

### 5.3 質問、Assumption、代替案

2D／3D／hybrid gameplay space、motion authority、contact／hit authority、shape class、高速移動policy、kinematic support relation、壊れるJoint、Target Profile、概算同時instance数が挙動を変える場合は質問する。質問は「どのsolverを使うか」や特定object名を前提にせず、観測可能な挙動の選択肢、影響、推奨案を示す。

明示情報がない場合もobject名からstatic／dynamic、solid／sensor、solver／query、discrete／continuousを既定化しない。Reference assumptionは登録済みrole recordと独立axisの候補として提示し、source intent、根拠、影響、owner refを持たせてPreviewから変更できるようにする。安全な選択肢が複数ある場合はtyped alternativeを最大限界内で提示し、候補ごとの差分を示す。

Authorization、Risk class、commit可否、credentialは[AI Security／Approval](../01-governance/ai-security-approval.md)だけが決定する。本書はoperationの意味とvalidationを定め、approval表を複写しない。

## 6. Operation、preview、diagnostic、AI eval

Physics operation familyはinspect／discover／validate／preview、World Profile作成／更新、Body dynamics設定、Joint／Constraint作成／更新／削除、Physics Kinematic Motion Provider qualification提案を持つ。owner固有proposalのadapter／Provider bindingは当該PackまたはProject operation、Collision geometry／filter／queryのoperationは[Collision](collision.md)へ委譲する。全writeは[Project state](../03-authoring/project-state.md)のChangeSetを作り、live Worldを直接mutateしない。

Previewはbefore／after semantic resolution、affected Entity／Asset、selected assumptions、question state、Capability availability、estimated impact class、diagnostic、rollback boundaryを示す。native setting dumpやVendor object graphをユーザー説明に使わない。Editor手動操作とAI操作は同じDocument、validator、preview、undo／redo、cookを通る。

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
