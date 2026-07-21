# Miraikanai Engine Physics Contract

- 文書ID: mirakan.arch.simulation-physics
- 状態: review
- 正本範囲: Physics World／Body dynamics、solver profile semantics、command、joint／constraint、kinematic character motor、kernel Adapter boundary、Physics save／replay projection、Physics AI intent／discovery／resolution／preview／diagnostic／eval
- 非正本範囲: Collider geometry／filter／query／event、Runtime phase／tick／capacity、Animation pose、Navigation artifact、external dependency version／build pin、AI authorization／evidence envelope。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Project state](../03-authoring/project-state.md)、[Gameplay programming model](../03-authoring/gameplay-programming-model.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Collision](collision.md)、[Navigation](navigation.md)、[Animation](animation.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論とPlatform境界

PhysicsはEngine-owned World、Body、Dynamics command、Joint／Constraint、Character Motor、snapshot、diagnosticを公開し、数値kernelをprivate Adapterへ隔離する。Project C++、GameplayDefinition、AI、EditorへVendorの型、ID、pointer、callback、setting、serializationを公開しない。採用dependencyとexact version／commit／license／build optionは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)だけが所有する。

[Collision](collision.md)はshape、Collider Asset、material、filter、query、contact／trigger／hit semanticsを所有する。Physicsはそれらを消費してWorldを進めるが再定義しない。[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)はcanonical phase、writer、lease、publishを、[Runtime performance／capacity](../04-runtime/performance-capacity.md)は共通capacity、queue、measurementを所有する。

Module境界は次の意味へ固定する。

| layer | 所有 | 禁止 |
|---|---|---|
| Physics Contracts | Engine value、Port、command、event view、snapshot | Vendor型、native callback |
| Physics Core | World lifecycle、dynamics merge、joint registry、character state、semantic resolver | Vendor include |
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

## 3. Joint、Constraint、Character Motor

`PhysicsJointCommonV1`はjoint Stable ID、World ref、Body A／Bまたはtyped World Anchor、enabled、collide-connected、local frame、optional break semanticsを持つ。Joint kindはtagged unionであり、存在しないfieldをproperty bagへ入れない。World Anchorは専用variantで、null Bodyやmagic handleで代用しない。

2D C1 familyはdistance、revolute、prismatic、weld、3D C1 familyはfixed、point、distance、hinge、slider、swing-twistを持つ。各familyはSI単位、normalized axis、orthogonal frame、ordered limit、finite motor targetを検証する。Vendor enum値、constraint pointer、reaction callbackは保存しない。新familyはCapability、schema、Editor、AI vocabulary、fixtureを同時に追加する。

Joint break候補はAdapter結果をSI単位へnormalizeし、Engine Stable IDとtick refを持つ`JointBreakEventV1`へ変換する。配送と次boundaryのcomponent removalはRuntime ownerの順序を消費する。Backend間のreaction値へbitwise一致は要求せず、fixtureで許容されるsemantic rangeを検査する。

### 3.1 Kinematic Character Motor

C1 CharacterはEngine-owned Kinematic Character Motorを使う。Backend固有character controllerをProject APIへ公開せず、[Collision](collision.md)のoverlap／shape castだけを利用する。

`CharacterMoveIntentV1`はCharacter handle、consume tick ref、planar displacement、vertical proposal、jump edge、up direction、root-motion proposal、producer metadataを持つ。Stateは`disabled | airborne | grounded | sliding | stepping | ceiling_blocked`、Outputはresolved pose／velocity、state、ground handle／generation／normal／relative point、platform delta、hit summary、diagnosticを持つ。

Motor resolverは次のsemantic stagesを固定する。

1. Intent、generation、Profile、finite、speed、World versionを検証する。
2. 前snapshotのground attachmentをgeneration付きで再検査する。
3. current overlapをcanonical hit orderで解消する。
4. planar sweepとslideをbounded iterationで解決する。
5. walkable obstacleだけにstep candidateを評価する。
6. vertical motion、jump、slope、ground snapを解決する。
7. final overlapとhard invariantを再検査し、kinematic targetを提出する。

tie-breakは[Collision](collision.md)のnormalized query orderingを使い、native callback順を使わない。Moving platform attachmentはEngine handle、generation、local contact pointだけを保存する。Platform teleport／destroy／generation changeではattachmentを切る。

Root motionは[Animation](animation.md)のproposalであり、`gameplay_only | root_motion_only | additive_bounded`のCharacter policyで合成する。Motorのresolved poseがauthoritativeで、Animationはそれを読む。PhysicsとAnimationがTransformへ二重writeしない。

## 4. Save、Replay、failure、qualification

SaveはEngine-owned World／Body／Joint／Character state、Profile identity、Collider Asset identityを保存し、native serializationやpointerを保存しない。Loadはschema、toolchain lock compatibility、Asset identity、finite value、generation relationを検証してstaging Worldを構築し、fixture validation後にcompatible boundaryで置換する。失敗時はactive Worldを維持する。

ReplayはRuntime ownerへnormalized command、accepted async input、Profile／artifact identity、state hash、snapshot projectionを供給する。記録slotは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)のcanonical `T100_ReplayCheckpoint`だけを参照し、Physics固有phaseを設けない。Replay environmentとdebug streamは[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)を消費する。

主要failure classはinvalid profile、unqualified adapter、handle／generation mismatch、command conflict、joint frame invalid、character depenetration failure、native invariant violation、job drain failure、save incompatibilityである。tick publish、fault transition、recovery boundaryはRuntime ownerへ委譲する。共通memory、worker、queue、frame thresholdをここで再定義しない。

Qualificationは全private Backendへ同じWorld lifecycle、stack、sleep／wake、joint、break、character slope／stair／platform、save／load、replay hash、fuzz、fault injectionを与える。Engine contractの結果、ordering、diagnostic、lifetimeが一致することを検査する。Dependency build、exact binary identity、license、target matrixは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、測定とcapacity promotionは[Runtime performance／capacity](../04-runtime/performance-capacity.md)が所有する。

## 5. AI semantics

Physics AI surfaceは自然言語を直接Body fieldへ投影せず、`intent -> discovery -> questions／assumptions -> semantic resolution -> typed operation -> preview -> validation`を一つのbounded pipelineとして扱う。RuntimeやVendor APIはAI surfaceではない。

### 5.1 Intentとclosed vocabulary

`PhysicsIntentVocabularyEntryV1`はstable intent ID、gameplay-facing phrase class、required context、resolved role／authority／collision／shape／speed policies、required Capability、question rule、explanation templateを持つ。`PhysicsIntentResolutionV1`はinput intent refs、observed scene facts、selected Capability、resolved semantic fields、questions、assumptions、alternatives、diagnostic、proposed operationsを持つ。

closed vocabularyは次の軸を別々に解決する。

| axis | 例 | 禁止する混同 |
|---|---|---|
| Gameplay role | environment、prop、character、projectile、trigger、vehicle、ragdoll | roleからmassを推測して確定 |
| Motion authority | static、kinematic motor、dynamic、animation proposal、gameplay-driven | visible motionとwriter authorityの混同 |
| Collision semantics | ignore、overlap、block、query-only | event subscriptionとの混同 |
| Hit authority | contact event、swept query、overlap、gameplay trace | sensorをauthoritative hitへ暗黙昇格 |
| Shape strategy | primitive、compound、convex、static mesh、height field | render meshのlive利用 |
| Speed policy | ordinary、fast-moving、teleporting | speed不明時のCCD断定 |

Vocabulary entryはBackend名、version、native setting、thread countを含めない。同義語はlocale-aware phrase mapからstable intentへ解決し、未登録文字列を新enumとして保存しない。

### 5.2 Capability discoveryと意味解決

Resolverは[Executable contracts](../02-foundation/executable-contracts.md)のCapability registryから、scene dimension、active maturity、Target、World Profile、Collision capability、authoring permissionを読み、利用可能なEngine operationだけを提示する。Backend featureをCapabilityとして直接表示しない。unsupportedなvehicle／ragdoll等は`capability_unavailable`とし、近い既存operationへsilent downgradeしない。

解決順は次である。

1. ユーザー文からgameplay object、motion、contact、hit、speed、dimensionの候補を抽出する。
2. Scene／Projectの既存World、Body、Collider、Profile、Capabilityをread-only discoveryする。
3. closed vocabularyへ候補を割り当て、矛盾と欠落を分類する。
4. gameplay behaviorを変える欠落だけを質問し、安全な欠落はReference assumptionとして明示する。
5. role、motion authority、collision、hit、shape、speedを独立に確定する。
6. typed Physics／Collision operationへ投影し、ChangeSet、preview、diagnosticを生成する。
7. validatorを再実行し、質問、assumption、unavailable Capabilityを結果へ残す。

### 5.3 質問、Assumption、代替案

2D／3D／hybrid gameplay space、Player motion class、authoritative hit方式、高速object、moving platform、壊れるJoint、mobile target、概算同時object数が挙動を変える場合は質問する。質問は「どのsolverを使うか」ではなく、ゲーム上の選択肢、影響、推奨案を示す。

明示情報がない通常の床／壁、一般prop、Player、projectileにはReference assumption候補を提示できるが、確定値として隠さない。各assumptionはsource intent、理由、影響を持ち、previewから変更できる。安全な選択肢が複数ある場合はtyped alternativeを最大限界内で提示し、候補ごとの差分を示す。

Authorization、Risk class、commit可否、credentialは[AI Security／Approval](../01-governance/ai-security-approval.md)だけが決定する。本書はoperationの意味とvalidationを定め、approval表を複写しない。

## 6. Operation、preview、diagnostic、AI eval

Physics operation familyはinspect／discover／validate／preview、World Profile作成／更新、Body dynamics設定、Joint／Constraint作成／更新／削除、Character Motor作成／更新／削除、qualification提案を持つ。Collision geometry／filter／queryのoperationは[Collision](collision.md)へ委譲する。全writeは[Project state](../03-authoring/project-state.md)のChangeSetを作り、live Worldを直接mutateしない。

Previewはbefore／after semantic resolution、affected Entity／Asset、selected assumptions、question state、Capability availability、estimated impact class、diagnostic、rollback boundaryを示す。native setting dumpやVendor object graphをユーザー説明に使わない。Editor手動操作とAI操作は同じDocument、validator、preview、undo／redo、cookを通る。

Diagnosticは少なくとも次を区別する。

- ambiguous intent／question required
- conflicting role／motion／collision semantics
- Capability unavailable／Target unsupported
- invalid profile／shape／joint／character relation
- unsafe speed／hit assumption
- stale scene／artifact／generation
- operation scope mismatch
- adapter qualification unavailable

各diagnosticはstable code、対象path、原因、修正候補、blockingか否かを返す。Validation failureを自然言語だけで返さず、unknown intentを最も近い既知roleへ自動確定しない。

AI Evalはvalid intent、question-required intent、conflicting intent、unsupported Capability、adversarial Vendor API要求、stale discovery、preview／commit差分をfixture化する。評価はsemantic resolutionの正解、必要質問、unsupportedの拒否、operation boundedness、diagnosticの再現性を検査する。Evidence、provenance、trace gradingの形式は[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)だけを消費する。

次を採用しない。

- Vendor API／setting／serializationをpublic Physics contractまたはAI vocabularyにすること
- backend名をユーザーのgameplay intentとして質問すること
- unsupported Capabilityのsilent fallback
- AI、Editor、Project C++からの任意World step／callback登録
- PhysicsによるCollision geometry／event、Runtime phase／capacity、Animation poseの再所有
