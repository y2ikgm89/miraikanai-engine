# Miraikanai Engine Physics AI Semantic Capability Catalog規約

- 文書版: 1.0
- 作成日: 2026-07-20
- 最終更新日: 2026-07-20
- 対象: Physics／Collision自然言語Intent、Capability discovery、意味解決、質問、Assumption、Preview、Diagnostic、AI Eval
- 状態: プロジェクト公式の規範設計レビュー版。MCD、Validator、Fixture、Provider projectionはPhase 0実装待ち
- 上位文書: [AIネイティブ独自ゲームエンジン 設計計画書](./2026-07-18-ai-native-game-engine-authoring-design.md)
- Physics正本: [Miraikanai Engine 独自Physics Platform／Dynamicsアーキテクチャ規約](./2026-07-20-physics-engine-architecture-design.md)
- Collision正本: [Miraikanai Engine Collision／Colliderアーキテクチャ規約](./2026-07-19-collision-collider-architecture-design.md)
- 実行可能契約: [Miraikanai Engine 実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)
- AI実装・保守規約: [Miraikanai Engine AI実装・保守ガバナンス規約](./2026-07-19-ai-engine-development-governance-design.md)
- AI検証規約: [Miraikanai Engine AI検証・評価・来歴規約](./2026-07-19-ai-verification-evaluation-provenance-design.md)
- 機能範囲: [Miraikanai Engine 2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)

## 1. 結論

Miraikanai Engineは、AIが「Rigidbodyを追加する」「Joltを使う」のような実装語を直接Projectへ書く方式を採用しない。AIまたは人間がゲーム上の意図を提案し、Engine-ownedの`PhysicsIntentResolutionV1`が次を型付きに確定した後だけ、既存のPhysics／Collision Operationで`ProjectChangeSet`を作る。

- 何が動き、何が動かすか。
- 押し返す、通知する、問い合わせるのどれか。
- Contact、Trigger、Shape Query、Gameplay Ruleのどれをauthoritative hitにするか。
- 2D／3D、Target、成熟度、Budgetで利用可能なCapabilityは何か。
- 不足情報を質問するか、安全なReference値をAssumptionとして記録するか。
- 未対応Intentを拒否するか、明示的な代替案を提示するか。

AIが扱いやすいかどうかはsolverを自作したかではなく、語彙、closed enum、単位、範囲、質問条件、失敗、Preview、検証結果が一意かで決まる。本書はそのAI向け意味契約を所有する。Box2D／Joltの型、設定名、callback、Backend選択はCatalogへ含めない。

## 2. 決定権と境界

| 主題 | 正本 |
|---|---|
| Physics自然言語Intent、AI向けcanonical role、意味解決順、質問、Assumption、説明、Semantic Eval | 本書 |
| World、Dynamics、Joint／Constraint、Character Motor、Save／Replay、Kernel Qualification | Physics規約 |
| Body／Collider、shape、material、filter、sensor、query、contact／trigger | Collision規約 |
| Capability／Type／Operationの共通Envelope、MCD、Schema projection、Codegen | 実行可能契約規約 |
| ProjectRevision、ChangeSet、Preview、Commit、Undo／Redo | Authoring Model規約 |
| Risk、Approval、Context、Provider、権限 | AI実装・保守規約 |
| Eval corpus、grader、holdout、Receipt、Promotion | AI検証規約 |
| C0～C3の製品機能、Target、Reference Scene | 2D／3D機能計画 |

本書はPhysics／Collisionのfield、数値範囲、Budget、phase、Capability成熟度を再定義しない。同じ概念の値が本書と所有文書で矛盾する場合、所有文書を優先し、Catalog、MCD fixture、AI Evalを同じChangeSetで修正する。

## 3. 比較の目的と採用判断

### 3.1 有名Engineの構成

調査基準日は2026-07-20である。次の比較は機能範囲と統合方式を確認するためのEvidenceであり、他EngineのAPI、Scene、Component名、既定値をMiraikanaiへ転用する根拠ではない。

| Engine | 公式資料で確認した構成 | AI向け設計への示唆 |
|---|---|---|
| Unity 6 | Built-in 3DはNVIDIA PhysX、Built-in 2DはBox2Dを統合し、DOTSではUnity PhysicsまたはHavok Physics for Unityを選択する。Ragdoll WizardはCollider、Rigidbody、Jointを生成する | 製品APIとEditorを所有し、用途別kernelや複数Componentを一つの制作体験へ統合できる |
| Unreal Engine 5.8 | EpicのChaos Physicsを統合し、Rigid Body、Constraint、Destruction、Networked Physics、Ragdoll、Vehicle、Cloth、Fluid、Hair、Fleshを製品機能として展開する | 広い機能範囲にはsolverだけでなく専用Asset、Editor、Debug、Network、Qualificationが必要 |
| Godot 4.7 | 2D／3DのEngine APIを所有し、新規3D ProjectはJoltを既定利用する。RigidBody、CharacterBody、Ragdoll、SoftBody、Vehicle等をEngineのNode／Editor経路へ統合する | 外部kernelを使ってもEngine-ownedの制作語彙とProject形式を維持できる |
| Miraikanai | 2DはBox2D 3.1.1、3DはJolt 5.6.0をprivate kernel候補とし、World、Body、Collider、Joint、Character、Command、Save、Editor、AI契約を独自所有する | C1の完成を優先し、機能ごとのMCD、Editor、AI、Fixture、Target Qualificationが揃った時だけC2／C3から昇格する |

### 3.2 Miraikanaiの到達点

| Capability領域 | C1 First Playable | C2 Production | C3 Research | AIの未昇格時動作 |
|---|---|---|---|---|
| 2D／3D rigid body | Static／kinematic／dynamic、material、sleep、CCD profile | Target別tuning、追加shape／batch | extreme scale | 利用可能なDimension／Targetだけ提案 |
| Query／Event | Ray／Cast／Overlap、Contact／Trigger／Hit | distance／batch、advanced surface event | rewind query | 未対応Queryへ推測変換しない |
| Joint／Constraint | C1 fixed set | SixDOF、wheel／motor、destructible group | custom／network constraint | 対応typeを列挙し、unknown fieldを生成しない |
| Character | Engine-owned Kinematic Character Motor | advanced crowd／vehicle連携 | specialized motor | 通常Playerをdynamic bodyへ置換しない |
| Ragdoll | 非対応 | Authoring、Animation接続、Save／Replay | specialized deformable連携 | `capability_unavailable`とC2要件を返す |
| Vehicle | 非対応 | Vehicle constraintと専用Authoring | advanced／network vehicle | 一般dynamic bodyで車両完成を装わない |
| Destruction | Joint breakまで | Destructible constraint setup | fracture／voxel destruction | Meshを自動分割して成功扱いしない |
| Soft body | 非対応 | 非対応 | CPU soft body研究 | Rigid bodyへの近似を自動Commitしない |
| Cloth／Fluid／Hair | 非対応 | 非対応 | 専用Capability研究 | Physics Core機能として表示しない |
| Large world | 10 km以内のReference範囲 | large static world／streaming | double precision／floating origin | Target CapabilityなしではWorld範囲を縮小しない |
| Networked physics | 非対応 | 非対応 | server rewind、prediction、lockstep研究 | local replayをnetwork determinismと説明しない |
| GPU physics | 非対応 | 非対応 | GPU rigid／soft／hair研究 | Joltの実験機能やRenderer computeを暗黙有効化しない |
| 独自solver | 非対応 | 非対応 | 同一契約で比較研究 | 「AIに理解しやすい」を自作理由にしない |

Unreal相当の機能名が表に存在するだけではCapability完成ではない。MCD、実装、Editor、AI Operation、Budget、Save／Replay、Diagnostic、Fixture、Target Qualification、Promotion Receiptが揃うまで`research_only`または`unavailable`とする。

## 4. 正規Semantic object

### 4.1 `PhysicsIntentVocabularyEntryV1`

`PhysicsIntentVocabularyEntryV1`は、自然言語の単語を直接Operationへbindする辞書ではない。Game文脈から意味候補を絞るためのversion付きCatalog entryである。

| Field | 型／規則 |
|---|---|
| `semantic_tag` | closed ID。例`physics.intent.pushable_prop` |
| `localized_terms` | locale別の代表語。命令や権限を含めない |
| `positive_examples` | このTagに一致する短いGameplay文 |
| `negative_examples` | 表面語が似ていても一致しない文 |
| `candidate_gameplay_roles` | 5.1節のclosed enum |
| `question_triggers` | 意味が分岐する条件 |
| `candidate_capability_ids` | discovery候補。利用可否はManifestで再検査 |
| `forbidden_mappings` | 自動変換してはならないrole／operation |
| `rationale_refs` | 本Review setのRequirement／Section参照 |

`localized_terms`の文字列一致だけで解決を確定しない。「動く床」はkinematic platformがReference候補だが、崩落床ならdynamicまたはdestructibleの質問が必要である。

### 4.2 `PhysicsIntentResolutionV1`

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

`source_request_ref`はAuthoring Task内のAccess-controlled contentを参照し、raw PromptをCatalog、MCD、Receiptへ複製しない。`contract_set_hash`が現在値と異なるResolutionはstaleとして拒否し、再解決する。

`ready_to_propose`はCommit許可ではない。既存OperationでChangeSetを作成できる状態を示すだけで、Validator、Preview、Risk、Approval、base revision検査を省略しない。

## 5. Closed semantic vocabulary

### 5.1 `GameplayPhysicsRoleV1`

```text
world_static
movable_prop
moving_platform
character
projectile
sensor_volume
camera_blocker
ragdoll
vehicle
destructible
soft_deformable
cloth_fluid_hair
```

一つのResolutionは一つのprimary roleを持つ。複合objectは複数Resolutionと明示的な関係で表現し、`character_vehicle_ragdoll`のような合成enumを追加しない。

### 5.2 `PhysicsMotionAuthorityV1`

| Value | 意味 |
|---|---|
| `static` | Simulation中にposeを変更しない |
| `kinematic_target` | Gameplay／Animation由来のTargetをT40で確定する |
| `dynamic_solver` | Force、Impulse、Velocityとsolver結果をT60で統合する |
| `character_motor` | Engine-owned Character Motorがposeを解決する |
| `query_driven` | authoritative hitをQueryで決め、Body trajectoryを正本にしない |
| `presentation_only` | Gameplay stateへ影響しない表示専用 |

AIは同じobjectの同じphaseへ複数authorityを選ばない。Dynamic Bodyへ毎frame Transform writeを生成せず、Characterへdynamic velocity直書きをC1既定にしない。

### 5.3 `PhysicsCollisionSemanticsV1`

| Value | 意味 |
|---|---|
| `solid_block` | Filterを通るnon-sensor shape間でsolver responseを生成 |
| `sensor_notify` | ResponseなしでEnter／Exit等を生成 |
| `query_only` | Simulation pairへ参加せず型付きQueryで検出 |
| `none` | Physics collisionを持たない |

「当たる」だけでは`solid_block`と`sensor_notify`を決めない。押し返しがGameplayに必要かを質問する。

### 5.4 `PhysicsHitAuthorityV1`

```text
solver_contact
sensor_event
swept_shape_query
overlap_query
gameplay_rule
none
```

高速Projectileは`swept_shape_query`をReferenceとする。表示用Rigid BodyのContactだけでauthoritative hitを保証しない。

### 5.5 `PhysicsShapeStrategyV1`

```text
primitive
compound_primitive
convex
static_triangle_mesh
heightfield
tile_chain_2d
none
```

DimensionとBody kindに存在しないstrategyを拒否する。Dynamic Bodyへ`static_triangle_mesh`または`heightfield`を選ばない。Render Meshの複雑さだけを理由にCollider精度を上げない。

### 5.6 `PhysicsSpeedPolicyV1`

| Value | 意味 |
|---|---|
| `discrete` | 通常速度のsimulation |
| `continuous_body` | 2D bullet／3D linear castを明示 |
| `authoritative_sweep` | Query結果をGameplay hitの正本にする |
| `teleport` | 経路hitを生成しない明示`TeleportBody` |

`teleport`を高速移動の代用品にしない。途中経路が必要なら`authoritative_sweep`を使う。

## 6. 意味解決手順

`operation.physics.resolve_intent`はread-only R0 Operationとし、次の順序を変更しない。

1. Task、Project revision、Contract set、Target Profile、権限Envelopeを検証する。
2. Sceneを`two_d | three_d | hybrid`へ分類し、Hybridでは対象Intentのgameplay spaceを一つ選ぶ。
3. Capability Manifestから現在Targetで利用可能なPhysics／Collision Capabilityだけを取得する。
4. `PhysicsIntentVocabularyEntryV1`からprimary gameplay role候補を作る。
5. motion authority、collision semantics、hit authority、shape strategy、speed policyの順に候補を狭める。
6. 結果を変える不足だけをBlocking／High Impact質問へ変換する。
7. 安全なReference値で確定できる不足はAssumption、理由、変更影響を記録する。
8. Target、maturity、Budget、field組合せ、禁止mappingをValidatorで検査する。
9. 対応不能なら`capability_unavailable`とし、Source要求を削減せず、利用可能な代替案と必要な昇格Gateを返す。
10. `ready_to_propose`の場合だけ既存write Operation候補とPreview fixtureを返す。

AI Providerの出力だけで`disposition`を確定しない。Gatewayが同じMCDとValidatorで再計算し、不一致なら`MIRAKAN-PHYSICS-AI-RESOLUTION_MISMATCH`で提案を拒否する。

## 7. Canonical intent mapping

| ユーザーIntent例 | Reference解決 | 質問または制約 | 禁止する近似 |
|---|---|---|---|
| 「動かない床／壁」 | `world_static`、`static`、`solid_block`、primitiveまたはstatic mesh | Render精度とprimitive性能で結果が変わる時だけ質問 | Kinematic／dynamic Bodyにしない |
| 「押せる木箱」 | `movable_prop`、`dynamic_solver`、`solid_block`、primitive／convex、mass from shape | 極端な重さ、浮遊、破壊が必要なら質問 | Transform animationで押せるように見せない |
| 「決まった経路の動く床」 | `moving_platform`、`kinematic_target`、`solid_block` | 崩落または外力で軌道が変わるなら質問 | Dynamic Bodyへ毎frame Transform writeしない |
| 「プレイヤーを歩かせる」 | `character`、`character_motor`、capsule query | 歩行、飛行、車両、ragdollをHigh Impact質問 | Dynamic Body直書きをC1既定にしない |
| 「高速弾を当てる」 | `projectile`、`query_driven`、`authoritative_sweep` | 貫通、複数hit、反射、表示Bodyの有無 | Discrete contactだけに依存しない |
| 「入ったらイベントを出す区域」 | `sensor_volume`、`sensor_notify` | 移動する区域か、対象Filterが不明なら質問 | Solidへ変更してEventを得ない |
| 「カメラを壁にめり込ませない」 | `camera_blocker`、`query_only`、shape cast | Sensorを除外するReferenceを記録 | Cameraをdynamic Bodyにしない |
| 「敵が倒れたら全身物理」 | `ragdoll`、C2 Capability | C1では`capability_unavailable` | 一つのcapsule Dynamic Bodyでragdoll完成を装わない |
| 「運転できる車」 | `vehicle`、C2 Capability | arcade／simulation、wheel数、network要否 | 一般propへ推進力だけ付けてVehicle Capabilityと表示しない |
| 「壊れる橋」 | `destructible`、Joint breakまたはC2 group | 破断単位、再現性、Save、破片数 | Runtimeで任意Meshを自動fractureしない |
| 「柔らかく変形するスライム」 | `soft_deformable`、C3 Research | Production要求なら別ADR | Rigid Bodyのscale変更でsoft bodyを模倣しない |
| 「服／液体／髪を物理で動かす」 | `cloth_fluid_hair`、C3 Research | 表示専用VFXでよい場合だけ別Capabilityを提案 | Physics CoreのRigid Body機能として成功扱いしない |
| 「瞬間移動する」 | 対応role＋`teleport` | 途中のTrigger／hitが必要か質問 | Teleportでsweep済みと説明しない |

この表は例文の完全一致表ではない。否定、条件、対象、前後文、GameSpecを含めて解決し、曖昧なら質問する。

## 8. 質問、Assumption、代替案

### 8.1 Blocking／High Impact

次は結果を大きく変えるため、無回答のまま`ready_to_propose`にしない。

- 2D／3D／Hybridと、Hybrid内のauthoritative gameplay space。
- 「当たる」が押し返すsolidか、通知だけのsensorか。
- Objectを経路どおり動かすkinematicか、外力で動くdynamicか。
- Characterが歩行、飛行、車両、ragdollのどれか。
- 高速Projectileのauthoritative hit、貫通、複数hit、反射。
- Render Mesh精度とprimitive／convex性能のどちらを優先するか。
- Project全体のCollision channel matrix変更。
- C2／C3 CapabilityをProduct要件として昇格するか、代替表現でよいか。

### 8.2 Safe Reference Assumption

次はGameSpecと矛盾しない場合だけAssumptionへ記録できる。

- 壁、床、地形はstatic solid。
- 小型propはprimitive／convex dynamic。
- 通常Player／NPCはKinematic Character Motor。
- 高速Projectile hitはswept Shape Query。
- Camera collisionはSensorを除外。
- mass、friction、dampingは承認済みReference Profileから複製する。

### 8.3 代替案

代替案は要求を黙って弱めず、差を明示する。

```text
RejectedAlternativeV1
  alternative_id: StableId
  description: localized string ref
  rejection_reason: DiagnosticId
  behavior_difference: typed summary
  capability_difference: CapabilityId[]
  target_difference: TargetProfileId[]
  cost_difference_ref: optional CostEstimateRef
  user_approval_required: bool
```

「Soft Bodyは未対応なのでRigid Bodyに置換」「Network Physicsは未対応なので敵数を減らす」のような代替は自動Commitしない。

## 9. AI OperationとCapability

Phase 0で次をMCDへ追加する。

| ID | 動作 | Risk |
|---|---|---|
| `capability.physics.semantic_authoring_v1` | Physics Intent vocabulary、resolution、explanationを利用可能にする | Capability |
| `operation.physics.resolve_intent` | 自然言語Intentを`PhysicsIntentResolutionV1`へread-only解決する | R0 |
| `operation.physics.explain_intent_resolution` | 選択、質問、Assumption、拒否理由、代替、Costを人間向けに説明する | R0 |

これらはwrite Operationではない。`ready_to_propose`後の変更は既存の`operation.collision.create_body_*`、`operation.collision.*`、`operation.physics.configure_body_dynamics`、`operation.physics.create_joint_*`等を使う。

AIは最初に`capabilities.search`でTarget、maturity、tagを絞り、選択したCapabilityだけを`capabilities.read`で取得する。全Physics Schema、全Backend資料、C3 fieldを一度にPromptへ投入しない。

## 10. Diagnosticとremediation

| ID | 条件 | 動作／remediation |
|---|---|---|
| `MIRAKAN-PHYSICS-AI-INTENT_AMBIGUOUS` | role、solid／sensor、authority等を一意にできない | 一つのBlocking questionを返す |
| `MIRAKAN-PHYSICS-AI-CAPABILITY_UNAVAILABLE` | Target／maturityで要求Capabilityが利用不能 | 要求を維持し、利用可能な代替と昇格Gateを提示 |
| `MIRAKAN-PHYSICS-AI-AUTHORITY_CONFLICT` | 同じobject／phaseへ複数writerを選択 | canonical authorityを質問または明示選択 |
| `MIRAKAN-PHYSICS-AI-UNSAFE_APPROXIMATION` | 未対応機能を別機能で成功扱い | ChangeSet生成を拒否 |
| `MIRAKAN-PHYSICS-AI-BUDGET_UNQUALIFIED` | CostまたはTarget fixtureが未検証 | Preview／Commitを停止しfixtureを要求 |
| `MIRAKAN-PHYSICS-AI-STALE_CATALOG` | Contract set、Catalog、Project revisionが古い | 最新hashで再解決 |
| `MIRAKAN-PHYSICS-AI-RESOLUTION_MISMATCH` | Provider提案とGateway再計算が不一致 | Provider結果を採用せずDiffを返す |

DiagnosticはSource location、Requirement ID、expected／actual、remediation IDを持つ。AIはDiagnosticを理由にCollider削除、Sensor化、Filter全許可、Budget引上げ、Target除外を自動実行しない。

## 11. Previewと説明

`operation.physics.explain_intent_resolution`は、少なくとも次を返す。

- Game上の理解を一文で示す。
- 選択したrole、authority、collision、hit、shape、speed。
- 選択したCapabilityのmaturityとTarget状態。
- 採用したAssumptionと変更時の影響。
- 拒否した代替案と理由。
- 追加／変更されるComponent、Asset、Profile、Ruleの概要。
- CPU、memory、body／shape／query／event capacityへの概算影響。
- 実行するPreview fixtureと未実証項目。
- Commitに必要なRiskとApproval。

Backend名、Native setting、solver iterationを初心者向け説明の主語にしない。上級者がEvidenceを要求した場合はKernel lockとQualification Receiptを別のread-only参照として提示できる。

Previewは少なくともCollider形状、motion authority、collision pair、Query経路、expected Event、Budget差を示す。C2／C3未昇格機能はPreview成功をProduction Capabilityの証拠にしない。

## 12. Canonical fixture

### 12.1 Valid

| Fixture ID | Input | 必須結果 |
|---|---|---|
| `physics_semantic.pushable_crate_3d_v1` | 「3D倉庫に押せる木箱を置く」 | `movable_prop`、`dynamic_solver`、`solid_block`、primitive／convex、R2 write候補 |
| `physics_semantic.trigger_area_2d_v1` | 「2D出口に入ったらクリア」 | `sensor_volume`、`sensor_notify`、2D filter／event候補 |
| `physics_semantic.fast_projectile_3d_v1` | 「高速弾を薄い標的へ確実に当てる」 | `projectile`、`authoritative_sweep`、contact単独非採用 |
| `physics_semantic.moving_platform_v1` | 「一定経路を往復する足場」 | `moving_platform`、`kinematic_target` |

### 12.2 Question required

| Fixture ID | Input | 必須質問 |
|---|---|---|
| `physics_semantic.hit_ambiguous_v1` | 「触れたら当たるようにする」 | 押し返すsolidか通知sensorか |
| `physics_semantic.moving_object_ambiguous_v1` | 「この岩を動かす」 | 経路制御か外力simulationか |
| `physics_semantic.hybrid_space_v1` | 「2D UI上の駒と3D盤面をぶつける」 | authoritative gameplay spaceとevent bridge |

### 12.3 Capability unavailable／invalid

| Fixture ID | Input | 必須結果 |
|---|---|---|
| `physics_semantic.soft_body_c1_v1` | 「C1で柔らかいスライムを完全物理化」 | `capability_unavailable`、Rigid Body自動近似なし |
| `physics_semantic.network_lockstep_v1` | 「全端末でbitwise一致する対戦Physics」 | C3 Research、local Replayを代替にしない |
| `physics_semantic.dynamic_mesh_v1` | 「動くRender MeshをそのままColliderにする」 | dynamic triangle mesh拒否、convex候補は承認付き代替 |
| `physics_semantic.teleport_sweep_v1` | 「瞬間移動中も全部にhitする」 | Teleportとsweepの矛盾を質問、成功扱いしない |

## 13. AI Eval

Physics Semantic Evalは既存Physics／Collision Evalを置き換えず、その前段の意味解決を評価する。

| Suite | Coverage | Release invariant |
|---|---|---|
| `physics_semantic_role` | 12 role×2D／3D／Hybrid×locale | primary role期待一致95%以上 |
| `physics_semantic_authority` | static／kinematic／dynamic／character／query／presentation | authority conflict 0件 |
| `physics_semantic_ambiguity` | solid／sensor、moving、projectile、hybrid | Blocking question期待一致95%以上 |
| `physics_semantic_unavailable` | C2／C3、Target非対応、Budget未実証 | unsafe approximation 0件 |
| `physics_semantic_adversarial` | Vendor field要求、Budget引上げ、承認回避、Prompt injection | unauthorized／unsafe Commit 0件 |
| `physics_semantic_roundtrip` | AI作成→Inspector変更→再解決→Undo／Redo | 同じMCD／ChangeSet／Validator経路100% |
| `physics_semantic_provider` | MCP／OpenAI／Anthropic projection | Schema conformance 100%、意味invariant一致 |

各Caseは3 runし、Model／Prompt／Tool Schema／Context retrievalの変更は一変数ずつ比較する。LLM graderをSchema、権限、Capability、Budget、Commit可否の唯一の判定にしない。

## 14. Phase 0実装Artifact

実装計画は次の正本Artifactを対象にする。

```text
/schemas/mirakan/types/type.physics.intent_vocabulary_entry_v1.mirakan.json
/schemas/mirakan/types/type.physics.intent_resolution_v1.mirakan.json
/schemas/mirakan/operations/operation.physics.resolve_intent.mirakan.json
/schemas/mirakan/operations/operation.physics.explain_intent_resolution.mirakan.json
/schemas/mirakan/capabilities/capability.physics.semantic_authoring_v1.mirakan.json
/tests/contracts/fixtures/valid/physics_semantic/
/tests/contracts/fixtures/invalid/physics_semantic/
/tests/contracts/golden/physics_semantic/
/evals/public/physics_semantic/
/evals/adversarial/physics_semantic/
```

Phase 0では正規MCD、Contract compiler／lint、C++／TypeScript descriptor、MCP／Provider projection、semantic validator、fixture、public／adversarial Evalを実装する。Box2D／Jolt Adapter、Runtime World、Ragdoll、Vehicle、Soft BodyはこのPhase 0作業へ含めない。

## 15. 更新規則

次の変更はCatalogだけで完結しない。

| 変更 | 必須ChangeSet |
|---|---|
| 語彙例、説明、localized termだけ | Catalog、MCD example、該当Eval |
| role、authority、semantics enum追加 | Catalog、MCD type version、Validator、Provider projection、Migration、Eval |
| CapabilityのC1／C2／C3昇格 | 所有Subsystem仕様、2D／3D計画、MCD Capability、実装、Editor、AI、Fixture、Qualification、Receipt |
| Backend／Kernel更新 | Physics規約のKernel update手順、Adapter conformance、CatalogのEvidence参照 |
| Budget／Target変更 | 所有Profile、Capability Manifest、Preview、Target fixture、Eval |

外部Engineが新機能を追加してもMiraikanaiのCapabilityを自動昇格しない。比較Evidenceを更新し、Product要件、同一契約、Budget、Target、保守費用をReviewする。

## 16. 完了条件

本CatalogのC0／Phase 0は次をすべて満たした時だけ完了とする。

1. `PhysicsIntentVocabularyEntryV1`と`PhysicsIntentResolutionV1`がMCDとして正本化される。
2. C++／TypeScript／Cooked descriptor／MCP／Provider projectionが同じContract setから生成される。
3. `capabilities.search`／`read`がTarget、maturity、tag、Budget、failure、example、`ai_guidance`を返す。
4. `resolve_intent`と`explain_intent_resolution`がread-only R0として実装される。
5. GatewayがProvider出力を再検証し、stale、authority conflict、unsafe approximationを拒否する。
6. Valid、question required、capability unavailable、invalid fixtureが全合格する。
7. Role／質問／Assumption期待一致95%以上、Schema conformance 100%、unsafe approximation／unsafe Commit 0件を3 runで満たす。
8. AI／Inspector／Undo／Redoが同じChangeSet／Validator／Preview経路を使う。
9. Raw Prompt、Vendor API、Native ID、pointer、callback、秘密情報がMCD、Project、Receiptへ混入しない。
10. Source、Contract set、generated artifact、Eval、Review、Promotion Receiptがhashで連結される。

この完了はPhysics Runtime C1、Box2D／Jolt Qualification、2D／3D First Playableの完了を意味しない。

## 17. 公式資料

| 公式資料 | 確認事項 |
|---|---|
| [Unity Physics integrations](https://docs.unity3d.com/Manual/physics-integrations.html) | Built-in 3D PhysX、2D Box2D、DOTS Unity Physics／Havok |
| [Unity 6 Ragdoll Wizard](https://docs.unity3d.com/jp/current/Manual/wizard-RagdollWizard.html) | Collider、Rigidbody、JointをEditorで生成 |
| [Unreal Engine 5.8 Physics](https://dev.epicgames.com/documentation/en-us/unreal-engine/physics-in-unreal-engine) | ChaosのRigid Body、Destruction、Network、Ragdoll、Vehicle、Cloth、Fluid等 |
| [Godot 4.7 Using Jolt Physics](https://docs.godotengine.org/en/4.7/tutorials/physics/using_jolt_physics.html) | Joltの3D既定利用とGodot Physicsとの差 |
| [Godot 4.7 Physics](https://docs.godotengine.org/en/4.7/tutorials/physics/index.html) | RigidBody、CharacterBody、Ragdoll、SoftBody等のEngine統合 |
| [Godot 4.7 VehicleBody3D](https://docs.godotengine.org/en/4.7/classes/class_vehiclebody3d.html) | Vehicle機能と既知の制約 |
| [Box2D v3.1.1](https://github.com/erincatto/box2d/releases/tag/v3.1.1) | 2D private kernel候補 |
| [Jolt Physics v5.6.0](https://github.com/jrouwe/JoltPhysics/releases/tag/v5.6.0) | 3D private kernel候補 |

外部資料はcoverageとintegration strategyのEvidenceである。Miraikanai固有のrole、Resolution、Diagnostic、Risk、Budget、Question、Operation、成熟度は本Review setが所有する。
