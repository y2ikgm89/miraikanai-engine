# Miraikanai Engine Physics Contract

- 文書ID: mirakan.arch.simulation-physics
- 状態: review
- 正本範囲: Physics World／Body dynamics、solver profile semantics、command、joint／constraint、Character Locomotion向けPhysics reference Provider、kernel Adapter boundary、Physics save／replay projection、Physics AI intent／discovery／resolution／preview／diagnostic／eval
- 非正本範囲: Collider geometry／filter／query／event、Runtime phase／tick／capacity、Animation pose、Navigation artifact、external dependency version／build pin、AI authorization／evidence envelope。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Project state](../03-authoring/project-state.md)、[Gameplay programming model](../03-authoring/gameplay-programming-model.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[Collision](collision.md)、[Navigation](navigation.md)、[Animation](animation.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論とPlatform境界

PhysicsはEngine-owned World、Body、Dynamics command、Joint／Constraint、snapshot、diagnosticを公開し、数値kernelをprivate Adapterへ隔離する。Character MotorはCore必須契約ではなく、Character Locomotion Featureが選択できるEngine-owned C1 reference Providerである。Project C++、GameplayDefinition、AI、EditorへVendorの型、ID、pointer、callback、setting、serializationを公開しない。採用dependencyとexact version／commit／license／build optionは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)だけが所有する。

[Collision](collision.md)はshape、Collider Asset、material、filter、query、contact／trigger／hit semanticsを所有する。Physicsはそれらを消費してWorldを進めるが再定義しない。[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)はcanonical phase、writer、lease、publishを、[Runtime performance／capacity](../04-runtime/performance-capacity.md)は共通capacity、queue、measurementを所有する。

Module境界は次の意味へ固定する。

| layer | 所有 | 禁止 |
|---|---|---|
| Physics Contracts | Engine value、Port、command、event view、snapshot | Vendor型、native callback |
| Physics Core | World lifecycle、dynamics merge、joint registry、semantic resolver | Vendor include |
| Physics Character Motor Provider | optional Locomotion provider state、profile、intent／result adapter | Navigation Port再定義、Pack install必須化 |
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
| `CharacterMotorProfileV1` | Character collider ref、max slope、step height、ground snap距離、slide／step iteration上限、speed上限 | Project source |
| `PhysicsWorldHandle`／`PhysicsBodyHandle` | Engine generation handle | Runtime only |
| `PhysicsStateSnapshotV1` | normalized transform、velocity、sleep、joint／character state | immutable tick snapshot |
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

## 3. Joint、Constraint、optional Physics Character Motor Provider

`PhysicsJointCommonV1`はjoint Stable ID、World ref、Body A／Bまたはtyped World Anchor、enabled、collide-connected、local frame、optional break semanticsを持つ。Joint kindはtagged unionであり、存在しないfieldをproperty bagへ入れない。World Anchorは専用variantで、null Bodyやmagic handleで代用しない。

2D C1 familyはdistance、revolute、prismatic、weld、3D C1 familyはfixed、point、distance、hinge、slider、swing-twistを持つ。各familyはSI単位、normalized axis、orthogonal frame、ordered limit、finite motor targetを検証する。Vendor enum値、constraint pointer、reaction callbackは保存しない。新familyはCapability、schema、Editor、AI vocabulary、fixtureを同時に追加する。

Joint break候補はAdapter結果をSI単位へnormalizeし、Engine Stable IDとtick refを持つ`JointBreakEventV1`へ変換する。配送と次boundaryのcomponent removalはRuntime ownerの順序を消費する。Backend間のreaction値へbitwise一致は要求せず、fixtureで許容されるsemantic rangeを検査する。

### 3.1 Kinematic Character Motor reference Provider

C1 reference recipeはEngine-owned Kinematic Character Motor Providerを適格化するが、Character Locomotion Featureのinstall、Path Following、Runtime EntryはこのProviderを要求しない。Backend固有character controllerをProject APIへ公開せず、[Collision](collision.md)のoverlap／shape castだけを利用する。

Provider-private `CharacterMoveIntentV1`はCharacter handle、consume tick ref、planar displacement、vertical proposal、jump edge、up direction、producer metadataを持つ。これはaccepted public intentではなく、Feature-owned `GameplayMotionIntentV1`、Navigation `MovementIntentV1`、Animation `RootMotionProposalV1`を検証後にPhysics Provider内部で生成するderived inputである。Port、Project Source、Save、Replayへ型参照を公開せず、`RootMotionProposalV1`を内部Fieldへ複写しない。`PhysicsCharacterResolvedMotionV1`はresolved pose／velocity、state、ground handle／generation／normal／relative point、platform delta、hit summary、diagnostic、input batch hash、generationを持つ。

`capability.motion_executor.physics_character_motor`はPhysics Providerが提供する正式Capabilityであり、次のexact 7-Field descriptorを[Navigation](navigation.md)が所有する`MotionExecutorProviderCatalogV1`へproduction recordとして登録する。Port型、transport batch、Provider Catalogを本書で再定義しない。全MCD参照は表のID、`version=1`、選択Contract set hashを持つ`McdContractRefV1`である。

| `executor_capability_ref.id` | `movement_profile_schema_ref.id` | `accepted_intent_schema_refs[].id` | `transport_message_schema_ref.id` | `resolved_motion_schema_ref.id` | `compatibility_predicate_ref.id` | `failure_diagnostic_refs[]` |
|---|---|---|---|---|---|---|
| `capability.motion_executor.physics_character_motor` | `type.physics.character_motor_profile` | `[type.feature.character_locomotion.gameplay_motion_intent, type.navigation.movement_intent, type.animation.root_motion_proposal]` | `type.navigation.motion_executor_intent_batch` | `type.physics.character_resolved_motion` | `policy.physics.character_motor_intent_profile_target_dimension` | `[MIRAKAN-PHYSICS-CHARACTER-MOTOR-INCOMPATIBLE, MIRAKAN-PHYSICS-CHARACTER-MOTOR-RESOLUTION_FAILED, MIRAKAN-PHYSICS-CHARACTER-MOTOR-STALE_RESULT]` |

production recordは`provider_id=provider.engine.physics.character_motor`、`provider_version=1`、self-excluding content hash、Engine Physics componentのexact owner ref／hash、`usage=production`、implementation System ref／hash、Target Profile集合、Qualification Receipt集合を持つ。policyはintent type subset、Profile schema／hash、Target Profile、2D／3D dimension、Collision query availabilityを検証する。root-motion modeが`animation`なのにexact `type.animation.root_motion_proposal`を受理できないrecordはActivation前に拒否する。

T40のFeature-owned binding SystemはGameplay、Navigation、Animation proposalをNavigation-owned `MotionExecutorIntentBatchV1` Port messageとして提出し、選択済みPhysics Character Motor Providerがentriesのaccepted schemaを一度だけ解決する。Animationを含む全proposalはbinding Systemを経由し、Providerへ直接提出しない。Provider-private `CharacterMoveIntentV1`はこの検証済みbatchからだけderiveし、Portのpublic accepted setへ混ぜない。`MovementIntentV1`の`desired_velocity`は[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)のfixed tick deltaを乗算してplanar displacementへ変換する。同一tickにGameplay移動入力と`MovementIntentV1`が競合した場合はGameplay入力を採用し、不採用のintentをtyped resultとしてPath Followerへ返す。root-motionの合成は後述のProvider policyに従い、優先順位を暗黙に変更しない。

Motorのmax slope、step height、ground snap距離、iteration上限、speed上限は§2の`CharacterMotorProfileV1`だけが保持し、stage 1が検証するProfileはこのProfileである。`NavAgentProfileV1`のslope／climbとの整合検証は[Navigation](navigation.md)のrequest validationが所有する。

Motor resolverは次のsemantic stagesを固定する。

1. Intent、generation、Profile、finite、speed、World versionを検証する。
2. 前snapshotのground attachmentをgeneration付きで再検査する。
3. current overlapをcanonical hit orderで解消する。
4. planar sweepとslideをbounded iterationで解決する。
5. walkable obstacleだけにstep candidateを評価する。
6. vertical motion、jump、slope、ground snapを解決する。
7. final overlapとhard invariantを再検査し、kinematic targetを提出する。

tie-breakは[Collision](collision.md)のnormalized query orderingを使い、native callback順を使わない。Moving platform attachmentはEngine handle、generation、local contact pointだけを保存する。Platform teleport／destroy／generation changeではattachmentを切る。

Root motionは[Animation](animation.md)からCharacter Locomotion bindingを経由してselected Motion Executorへ届くproposalであり、本Provider選択時は`gameplay_only | root_motion_only | additive_bounded`のProvider policyで合成する。Providerのresolved motionがauthoritativeで、Animationはそれを読む。PhysicsとAnimationがTransformへ二重writeしない。

## 4. Save、Replay、failure、qualification

SaveはEngine-owned World／Body／Joint／Character state、Profile identity、Collider Asset identityを保存し、native serializationやpointerを保存しない。Loadはschema、toolchain lock compatibility、Asset identity、finite value、generation relationを検証してstaging Worldを構築し、fixture validation後にcompatible boundaryで置換する。失敗時はactive Worldを維持する。

ReplayはRuntime ownerへnormalized command、accepted async input、Profile／artifact identity、state hash、snapshot projectionを供給する。記録slotは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)のcanonical `T100_ReplayCheckpoint`だけを参照し、Physics固有phaseを設けない。Replay environmentとdebug streamは[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)を消費する。

主要failure classはinvalid profile、unqualified adapter、handle／generation mismatch、command conflict、joint frame invalid、character depenetration failure、native invariant violation、job drain failure、save incompatibilityである。tick publish、fault transition、recovery boundaryはRuntime ownerへ委譲する。共通memory、worker、queue、frame thresholdをここで再定義しない。

Qualificationは全private Backendへ同じWorld lifecycle、stack、sleep／wake、joint、break、C1 reference Providerのcharacter slope／stair／platform（斜面際のstep、ceilingに接した状態のslide、moving platformからの降車、狭所でのdepenetration発振を含む）、save／load、replay hash、fuzz、fault injectionを与える。Engine contractの結果、ordering、diagnostic、lifetimeが一致することを検査する。Provider fixtureはNavigationのexact `MotionExecutorPortV1`へのbinding、intent／profile／Target compatibility、stale result、root-motion proposal、provider failure時のlast-valid resolved motion不変を含む。別fixtureはPhysics capability unavailableでもCharacter Locomotion Packのinstallが成功し、Physics Provider選択だけがunavailableになることを検証する。Dependency build、exact binary identity、license、target matrixは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、測定とcapacity promotionは[Runtime performance／capacity](../04-runtime/performance-capacity.md)が所有する。

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
| `candidate_gameplay_roles` | `GameplayPhysicsRoleV1[]` |
| `question_triggers` | 意味が分岐する条件 |
| `candidate_capability_ids` | discovery候補。利用可否はManifestで再検査 |
| `forbidden_mappings` | 自動変換してはならないrole／operation |
| `rationale_refs` | Architecture requirement／section参照 |

`localized_terms`の文字列一致だけでResolutionを確定しない。Vocabulary entryはBackend名、exact dependency version、native setting、thread countを含めず、未登録文字列を新enumとして保存しない。

`PhysicsIntentResolutionV1`のmandatory schemaは次である。fieldの省略、任意propertyの追加、closed value以外の文字列を拒否する。

```text
PhysicsIntentResolutionV1
  source_request_ref: ContentRef
  source_request_hash: Sha256
  contract_set_hash: Sha256
  project_revision: RevisionId
  target_profile_ids: TargetProfileId[1..16]
  scene_dimension: two_d | three_d | hybrid
  hybrid_gameplay_space: optional two_d | three_d
  gameplay_role: GameplayPhysicsRoleV1
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

`source_request_ref`はAuthoring Task内のaccess-controlled contentを参照し、raw PromptをCatalog、MCD、Receiptへ複製しない。`ready_to_propose`は既存OperationでChangeSetを提案できる意味だけを持ち、Commit authorizationを意味しない。`question_required`はblocking ambiguityが残る結果、`capability_unavailable`は要求を満たすactive Capabilityがない結果、`rejected`はinvalid／forbiddenな要求である。

closed semantic valuesを次へ固定する。一つのResolutionは各軸から一つだけを選ぶ。

| Type | Closed values |
|---|---|
| `GameplayPhysicsRoleV1` | `world_static \| movable_prop \| moving_platform \| character \| projectile \| sensor_volume \| camera_blocker \| ragdoll \| vehicle \| destructible \| soft_deformable \| cloth_fluid_hair` |
| `PhysicsMotionAuthorityV1` | `static \| kinematic_target \| dynamic_solver \| query_driven \| presentation_only` |
| `PhysicsCollisionSemanticsV1` | `solid_block \| sensor_notify \| query_only \| none` |
| `PhysicsHitAuthorityV1` | `solver_contact \| sensor_event \| swept_shape_query \| overlap_query \| gameplay_rule \| none` |
| `PhysicsShapeStrategyV1` | `primitive \| compound_primitive \| convex \| static_triangle_mesh \| heightfield \| tile_chain_2d \| none` |
| `PhysicsSpeedPolicyV1` | `discrete \| continuous_body \| authoritative_sweep \| teleport` |

`ragdoll`、`vehicle`は将来拡張との語彙衝突を防ぐ予約値であり、現行C1／C2のactive Capability、operation、positive fixtureではない。`future.capability.vehicle-ragdoll-crowd-motion-warping`がapproved Future-to-Active Promotion Manifest、Control Plane Rebaseline、Active Definition migrationで機能別active Capabilityへ分割され、Owner contract、Target binding、Authority、Save／Replay、Qualificationが成立するまで、これらのroleを選んだResolutionは必ず`capability_unavailable`とする。予約値がenumに存在することをBackend supportまたはProduct supportの根拠にしてはならない。

複合objectは複数Resolutionと明示関係で表し、合成enumを追加しない。同じobjectへ複数motion authorityを選ばない。Dynamic Bodyへ`static_triangle_mesh`／`heightfield`を選ばず、Sensorをauthoritative hitへ暗黙昇格せず、`teleport`を経路hitの代用にしない。途中経路がGameplayへ必要なら`authoritative_sweep`を使用する。

### 5.2 Capability discoveryと意味解決

Resolverは[Executable contracts](../02-foundation/executable-contracts.md)のCapability registryから、scene dimension、active maturity、Target、World Profile、Collision capability、authoring permissionを読み、利用可能なEngine operationだけを提示する。Backend featureをCapabilityとして直接表示しない。unsupportedなvehicle／ragdoll等は`capability_unavailable`とし、近い既存operationへsilent downgradeしない。

解決順は次である。

1. source requestのcontent hash、Project revision、Contract set hash、Target Profileを取得し、Resolutionへ観測値としてbindする。
2. ユーザー文からgameplay object、motion、contact、hit、speed、dimensionの候補を抽出する。
3. Scene／Projectの既存World、Body、Collider、Profile、Capabilityをread-only discoveryする。
4. closed vocabularyへ候補を割り当て、矛盾と欠落を分類する。
5. gameplay behaviorを変える欠落だけを`blocking_question_ids`へ入れ、`question_required`にする。安全な欠落だけをReference assumptionとして明示する。
6. role、motion authority、collision semantics、hit authority、shape strategy、speed policyを独立に確定する。
7. Target、Capability、Profile、field relation、forbidden mappingを同じValidatorで再検査する。対応不能は`capability_unavailable`、invalid／forbiddenは`rejected`にする。
8. `ready_to_propose`の場合だけtyped Physics／Collision write OperationとPreview fixtureを返す。
9. GatewayがProvider出力を同じMCDとValidatorで再計算し、不一致をresolution mismatchとして拒否する。

Resolutionのvalidate、preview、operation proposalの各入口は、現在のsource request hash、Contract set hash、Project revisionを保存済み`source_request_hash`、`contract_set_hash`、`project_revision`と完全一致で再検査する。一つでも異なるResolutionは`stale`として拒否し、selected operation、preview、cost estimateを使用しない。最新source／contract／Project snapshotでCapability discoveryから再解決し、新しいResolution identityを発行する。stale objectをfield単位で更新、別revisionへrebase、Commitへ継続してはならない。

### 5.3 質問、Assumption、代替案

2D／3D／hybrid gameplay space、Player motion class、authoritative hit方式、高速object、moving platform、壊れるJoint、mobile target、概算同時object数が挙動を変える場合は質問する。質問は「どのsolverを使うか」ではなく、ゲーム上の選択肢、影響、推奨案を示す。

明示情報がない通常の床／壁、一般prop、Player、projectileにはReference assumption候補を提示できるが、確定値として隠さない。各assumptionはsource intent、理由、影響を持ち、previewから変更できる。安全な選択肢が複数ある場合はtyped alternativeを最大限界内で提示し、候補ごとの差分を示す。

Authorization、Risk class、commit可否、credentialは[AI Security／Approval](../01-governance/ai-security-approval.md)だけが決定する。本書はoperationの意味とvalidationを定め、approval表を複写しない。

## 6. Operation、preview、diagnostic、AI eval

Physics operation familyはinspect／discover／validate／preview、World Profile作成／更新、Body dynamics設定、Joint／Constraint作成／更新／削除、Physics Character Motor Provider qualification提案を持つ。Character LocomotionのProvider選択／bindingはFeature operation、Collision geometry／filter／queryのoperationは[Collision](collision.md)へ委譲する。全writeは[Project state](../03-authoring/project-state.md)のChangeSetを作り、live Worldを直接mutateしない。

Previewはbefore／after semantic resolution、affected Entity／Asset、selected assumptions、question state、Capability availability、estimated impact class、diagnostic、rollback boundaryを示す。native setting dumpやVendor object graphをユーザー説明に使わない。Editor手動操作とAI操作は同じDocument、validator、preview、undo／redo、cookを通る。

Diagnosticは少なくとも次を区別する。

- `MIRAKAN-PHYSICS-CHARACTER-MOTOR-INCOMPATIBLE`: intent subset、Profile、Target、dimension、Collision query relation不一致
- `MIRAKAN-PHYSICS-CHARACTER-MOTOR-RESOLUTION_FAILED`: bounded resolverがvalid resolved motionを生成できない
- `MIRAKAN-PHYSICS-CHARACTER-MOTOR-STALE_RESULT`: actor／intent batch／profile／provider generation不一致
- ambiguous intent／question required
- conflicting role／motion／collision semantics
- Capability unavailable／Target unsupported
- invalid profile／shape／joint／character relation
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
