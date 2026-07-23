# Miraikanai Engine Shooter Genre Pack

- 文書ID: mirakan.arch.pack-shooter
- 状態: review
- 正本範囲: Shooter固有composition recipe、Genre identity、Profile、Game Flow、Action role、Shooter fixture
- 非正本範囲: Feature CapabilityのPublic Contract／Schema／State owner、Pack lifecycle、Input／Collision／Physics／Navigation／Camera／UI／Audio／LOD、World／Stage、Runtime scheduling、Product roadmapは各Ownerを参照
- 依存: [Product Plan](../00-product/product-plan.md)、[Pack Contract](pack-contract.md)、[Gameplay Feature Packs](gameplay-features.md)、[Scenario／Stage](scenario-stage.md)、[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Collision](../05-simulation/collision.md)、[Physics](../05-simulation/physics.md)、[Navigation](../05-simulation/navigation.md)、[Camera](../06-rendering/camera.md)、[LOD](../06-rendering/lod.md)、[Input](../07-platform/input.md)、[Audio](../07-platform/audio.md)、[Game UI](../07-platform/ui-text-localization-accessibility.md)
- 外部根拠検証日: 2026-07-23

## 1. 結論

Shooterは`pack_kind=genre`のGenre Packであり、Generic Engine Coreまたは単一の共通Gameplay基盤ではない。Shooterは再利用可能なFeature Packをcomposition recipeで組み合わせ、Shooter固有のProfile、Game Flow、Action role、reference scenarioだけを所有する。

正規Genre identityは次の二つである。

- `genre.shooter.top_down_2d`
- `genre.shooter.third_person_3d`

2D／3Dは同じFeature Capabilityを使い、Damage、Vital、Faction、Weapon、Shot、Projectile、Encounter、Scoring、Save／Replayの意味をforkしない。Shooter Packを未installまたは削除した状態でCore、Editor、AI、Project C++、Project Shader、Test、Build、Packageが成功しなければならない。

## 2. Feature Capability mapping

Shooter Genre Packは次のFeature Packを参照する。各Public Contract、Schema、State owner、Validator、Runtime Port、failureはFeature Packが唯一のOwnerであり、本書は再定義しない。

| Feature Pack | Provided Feature Capability | Shooterでの利用 |
|---|---|---|
| `feature.combat` | `capability.gameplay.combat` | Damage、Vital、Faction |
| `feature.ranged_combat` | `capability.gameplay.ranged_combat` | Weapon、Shot、Projectile |
| `feature.encounter_spawn` | `capability.gameplay.encounter_spawn` | Encounter、Spawn |
| `feature.scoring` | `capability.gameplay.scoring` | score、combo、result projection |
| `feature.pickup_grant` | `capability.gameplay.pickup_grant` | Pickup、typed Grant |
| `feature.interaction` | `capability.gameplay.interaction` | interact eligibilityとgeneric `policy_denied` |
| `feature.character_locomotion` | `capability.gameplay.character_locomotion` | controlled actorの移動intentとMotion Executor binding |
| `feature.path_following` | `capability.gameplay.path_following` | enemy／agent route。Character Motorへ固定しない |
| `feature.scenario_stage` | `capability.gameplay.scenario_stage` | 有限Stage、Objective、Completion、transition |

Feature Capabilityは必要なcomposition recipeだけが選ぶ。たとえばendless Shooter recipeはScenario／Stageを省略でき、tool-like weapon editorはCombat、Encounter、Worldを要求しない。現在の二つのreference fixtureは開始からResultまでの有限Gameを検証するため、表のFeature Pack closureを使用する。

Shooterのenemy targetingは`capability.gameplay.perception`をrequired Capabilityとして参照し、Public ContractとState ownerは[Gameplay Programming Model §2.4](../03-authoring/gameplay-programming-model.md#24-c1-perceptioninteraction)が所有する。Shooterは`PerceptionProfileV1`、`PerceptionSnapshotV1`、Perception memoryを再定義またはwriteしない。

Genre Packは別Genre Packへ依存しない。複数Genreとの合成はGame Projectが行う。

## 3. Genre composition

```text
genre.shooter.top_down_2d
  -> profile.shooter.top_down_2d
  -> Feature Capability closure
  -> Generic Engine Core

genre.shooter.third_person_3d
  -> profile.shooter.third_person_3d
  -> Feature Capability closure
  -> Generic Engine Core
```

Shooter Packのcanonical manifest recordは次である。表外の`PackManifestV1` Fieldは[Pack Contract](pack-contract.md)どおりexact identityで解決し、Feature契約をpayloadとして内包しない。

```text
PackManifestV1
  pack_id: genre.shooter
  pack_kind: genre
  required_feature_pack_refs:
    [feature.combat@1, feature.ranged_combat@1, feature.encounter_spawn@1,
     feature.scoring@1, feature.pickup_grant@1, feature.interaction@1,
     feature.character_locomotion@1, feature.path_following@1,
     feature.scenario_stage@1]
  required_capability_refs:
    [capability.gameplay.perception]
  composition_recipe_refs:
    [genre.shooter.top_down_2d, genre.shooter.third_person_3d]
  configuration_profile_refs:
    [profile.shooter.top_down_2d, profile.shooter.third_person_3d]
  public_contract_refs:
    [ShooterPerceptionBindingV1, ShooterGameFlowV1, ShooterActionRoleSetV1]
  validator_refs:
    [validator.genre.shooter.composition, validator.genre.shooter.perception_binding]
  test_scenario_refs:
    [fixture.product.shooter-2d, fixture.product.shooter-arena-3d]
```

Profileは独立PackではなくShooter Packのversion／hashに含まれる。recipeがScenario／Stageを省略する場合は`feature.scenario_stage@1`もそのrecipe closureから省略できるが、上記reference fixture closureは有限Gameとして全9 Featureを要求する。

composition recipeはFeature Capability、Profile、GameSpec template、Action role、reference scenarioをexact IDで結ぶ。Feature schema、World、Stage、Runtime Scope、Input Action identityをrecipe内で再定義しない。

## 4. Shooter Profile

### 4.1 `profile.shooter.top_down_2d`

- `genre_ref=genre.shooter.top_down_2d`
- orthographic／pixel-safe Camera binding
- `move`、`aim`、primary fire、任意のsecondary fire／reload／pause Action role
- top-down向けCamera、Audio、LOD configuration
- 2D collision／animation／VFX／UIへのbinding
- `fixture.product.shooter-2d`

### 4.2 `profile.shooter.third_person_3d`

- `genre_ref=genre.shooter.third_person_3d`
- third-person Camera、camera collision、reticle binding
- `move`、`look`、primary fire、aim mode、reload、weapon switch、pause Action role
- third-person向けCamera、Audio、LOD configuration
- 3D collision／animation／VFX／UIへのbinding
- `fixture.product.shooter-arena-3d`

両Profileのenemy archetype bindingは次のGenre-owned recordを使う。

```text
ShooterPerceptionBindingV1
  binding_id
  enemy_archetype_ref
  perception_profile_ref
  hostile_team_filter_ref
  lost_target_behavior: search_last_known | return_to_route
  fire_intent_policy_ref
```

`perception_profile_ref`はPerception ownerのactive `PerceptionProfileV1`へexactに解決し、target selection policyと`hostile_team_filter_ref`の互換性を検証する。`lost_target_behavior`は上記closed enumだけを許可し、`search_last_known`はbounded Perception memory、`return_to_route`はPath Following portのtyped requestだけを読む。`fire_intent_policy_ref`はvisibleまたは記憶済みtarget evidenceから`RequestFireCommandV1`を生成するpolicyへ解決し、target消失後のfire継続条件と最大tick数を明示しなければならない。Render visibility、Camera frustum、Audio mixer、VFX、native Physics objectをtarget／fire authorityにしたbindingを拒否する。

Camera、Audio、LOD Profileのparameter schemaと実行規則は各Subsystem Ownerが所有する。Shooter Profileは参照とGenre固有の組合せだけを所有し、Camera rig、Audio voice、representation selectionをGameplay authorityにしない。

## 5. Shooter Game Flow

Shooter Genre Packが所有するclosed stateは`Ready | Playing | Paused | Result`である。

| From | To | 条件 |
|---|---|---|
| `Ready` | `Playing` | selected entry／Stageのactivationが成功 |
| `Playing` | `Paused` | Pause Capabilityが有効でpolicyが許可 |
| `Paused` | `Playing` | 同じsessionをresume |
| `Playing` | `Result` | Scenario／StageまたはProject-owned policyがtyped outcomeを提出 |
| `Result` | `Ready` | restartが新しいplay attemptを作成 |

表外の遷移とdefault transitionを拒否する。Title、Settings、Menu、LoadingはUI／Runtime entryの状態でありShooter Game Flow stateへ追加しない。Pause非対応Projectでは`Playing -> Paused`を適用せず、Runtime ownerのtyped failureを返す。

Game FlowはFeature Stateを直接writeしない。開始、pause、resume、completion、restartを各Feature ownerへのtyped CommandとSnapshotで接続し、UI visibility、Audio completion、Animation Eventからauthorityを推測しない。Save／Replayは登録済みState ownerと各FeatureのSave／Replay契約が宣言したfieldだけを記録する。

## 6. Shooter Action role

Shooter Packは次のsemantic roleを提供する。

- `shooter.move`
- `shooter.aim`
- `shooter.look`
- `shooter.fire_primary`
- `shooter.fire_secondary`
- `shooter.aim_mode`
- `shooter.reload`
- `shooter.next_weapon`
- `shooter.previous_weapon`
- `shooter.pause`

Action roleはGenre vocabularyでありRuntime identityではない。[Input](../07-platform/input.md)がProject適用時にActionのStable ID、value schema、binding、remap、snapshotを所有する。Profileは必要roleとInput templateを宣言できるが、Platform key code、device、dead zone、gestureをShooter契約へ固定しない。

## 7. AI composition

AIは[Pack Contract](pack-contract.md)のPack Operationを使い、要求をGenre identity、Profile、Feature Capability closure、Target、fixtureへ解決する。2D／3D、finite／endless、input modality、Camera、required Featureが未確定でGame構造が変わる場合は`question_required`とし、類似表示名からGenreやProfileを推測しない。

Previewは少なくとも次をexact IDで示す。

- selected Genre identityとProfile
- Feature Pack dependency closure
- Camera／Audio／LOD binding
- Action roleから生成するInput Action
- Scenario／Stageの有無とcompletion policy
- Target support、budget、fixture、fallback

AIはFeature schemaをShooter schemaとして複写せず、Feature OwnerのCatalog entry、Operation、Validatorを使用する。

## 8. Fixtureとconformance

### 8.1 `fixture.product.shooter-2d`

`genre.shooter.top_down_2d`、`profile.shooter.top_down_2d`、Feature Capability closureを使い、ReadyからPlaying、Paused／resume、Result、restartを検証する。Damage／Vital／Faction、Ranged Combat、Encounter／Spawn、Scoring、Pickup／Grant、Interaction、Character Locomotion、Path Following、Scenario／Stageが各Feature OwnerのPublic Contractだけを使用することを確認する。Perception境界の出入り、`search_last_known`／`return_to_route`、target消失後のfire-intent停止tickをfixed inputで検証する。

### 8.2 `fixture.product.shooter-arena-3d`

`genre.shooter.third_person_3d`、`profile.shooter.third_person_3d`、同じFeature Capability semanticsを使い、third-person Camera／Audio／LOD binding、3D locomotion、path following、Save／Load、Replay、Resultまでを検証する。target取得／lost-target／fire-intentのEvent sequenceとReplay hashが2Dと同じowner contractに適合することを確認し、Feature schemaをforkしない。

### 8.3 Negative fixture

- Shooter Packから別Genre Packへのdependencyを拒否する。
- Feature CapabilityのSchema／State ownerをShooter側で再宣言したrecipeを拒否する。
- missing／incompatible `PerceptionProfileV1`、unknown lost-target value、unbounded fire-intent継続を拒否する。
- Shooter role文字列をInput Action identityまたはSave identityにしたProjectを拒否する。
- Shooter PackなしのFeature-only Projectをvalidとする。
- Shooter Pack removal後もCore／Editor／AI／Project source／Build／Packageを成功させる。

Capability成熟度、Phase、Target別Activation、Product claimは[Product Plan](../00-product/product-plan.md)だけが所有する。

## 9. Shooter Difficulty Profile detail

Difficultyは`easy`、`normal`、`hard`等の表示名だけでは成立しない。Profileは変更できるField ref、基準値、演算、上限、Gameplay fidelity floorを次のcanonical schemaで明示する。

```text
DifficultyProfileV1
  difficulty_profile_id: StableId
  axes[1..16]:
    axis_field_ref
    base_value
    operation: add | multiply_q16 | replace
    clamp_min
    clamp_max
  gameplay_fidelity_floor_ref
```

`axis_field_ref`は下記C1軸のFieldだけを参照でき、それ以外をvalidatorが拒否する。`multiply_q16`の倍率は[Gameplay Features §3.3](gameplay-features.md)と同域のunsigned Q16 `[0.0625,16]`とする。適用結果は`clamp_min`／`clamp_max`で有限に固定し、NaN／Infを生成しない。`gameplay_fidelity_floor_ref`はDifficultyが下回れないGameplay fidelity(Collision有効、Hit Event非drop、authoritative objectの非削除、Replay決定性)の宣言を指す。

C1で変更可能な軸は次に限定する。

- enemy max Health
- enemy base Damage
- Encounter spawn interval／group count
- Shot Pattern／cadenceの選択
- pickup量
- Score multiplier

Difficulty変更でCollision無効化、Hit Event drop、敵やProjectileの無断削除、Replay非決定化を行わない。

## 10. Input Action Template

Shooter PackはActionの表示名ではなくdomain-specific Semantic roleを提供し、Action identity、Value schema、Binding、Remap、Snapshotは[Input](../07-platform/input.md)へ委譲する。Project適用時はInput ownerを通じてAction `StableId`を生成する。

| Semantic role | Value | 2D | TPS | Required |
|---|---|---:|---:|---:|
| `shooter.move` | axis2 | yes | yes | yes |
| `shooter.aim` | axis2／pointer2 | yes | no | yes for 2D |
| `shooter.look` | axis2 | no | yes | yes for TPS |
| `shooter.fire_primary` | digital | yes | yes | yes |
| `shooter.fire_secondary` | digital | optional | optional | no |
| `shooter.aim_mode` | digital | no | yes | yes for TPS |
| `shooter.reload` | digital | optional | yes | Profile |
| `shooter.next_weapon` | digital | optional | yes | Profile |
| `shooter.previous_weapon` | digital | optional | yes | Profile |
| `shooter.pause` | digital | yes | yes | yes |

Actionから直接Weapon Stateを書き換えず、Input SnapshotをControl／Weapon intent evaluatorがtyped Commandへ変換する。AI、Replay、Controller、Keyboard／Mouse、Touchは同じsemantic Actionを使う。

User Remap、toggle／hold、sensitivity、dead zone、left-handed layoutはInput規約が所有する。Difficulty ProfileがUser Remapを変更しない。

## 11. PresentationとGameplay分離

一つのauthoritative Eventから次を独立配送する。

| Event | UI | Audio | VFX | Camera |
|---|---|---|---|---|
| Fire accepted | ammo／reticle | shot／mechanical | muzzle／shell／trail | recoil channel |
| Shot hit | hit marker | impact | impact／decal | bounded shake |
| Damage applied | Health／damage number | damage cue | hit reaction | bounded shake |
| Defeat | score／result | defeat cue | defeat effect | director request |
| Reload | ammo progress | reload cue | optional prop | none |

Presentation cueの失敗、Voice不足、VFX drop、Camera unavailableでFire、Damage、Score結果を変更しない。critical cueがownerのcapacity内で出せない場合はPresentation Qualificationを失敗させるが、Gameplay Eventをdropしない。

Camera recoilは現在のcontractではPresentation-onlyであり、Gameplay aim、Shot direction、Collision query、Save transformへ戻さない。authoritative recoilは対象外であり、将来scope、activation、schema／Profile判断は[Product Plan](../00-product/product-plan.md)だけが所有する。

## 12. AI Semantic Contract

### 12.1 `ShooterIntentResolutionV1`

```text
ShooterIntentResolutionV1
  requirement_refs[]
  source_terms[]
  canonical_concept_refs[]
  selected_genre_pack_ref
  selected_profile_ref
  weapon_intents[]
  shot_delivery_intents[]
  fire_mode_intents[]
  ammo_reload_intents[]
  damage_intents[]
  team_vital_intents[]
  pickup_intents[]
  encounter_intents[]
  score_intents[]
  game_flow_intents[]
  input_intents[]
  presentation_intents[]
  target_profiles[]
  scale_intent_ref
  assumptions[]
  blocking_questions[]
  high_impact_questions[]
  capability_gaps[]
  forbidden_approximations[]
  validation_scenarios[]
```

ResolverはSource termごとにEvidence Requirement IDとconfidenceを返す。confidenceの平均値でBlocking不足を隠さない。

### 12.2 必須質問

次が未指定で、候補間にGameplay上の差がある場合だけ質問する。

- 2D top-down、TPS、その他のview／movementか
- hitscan、projectile、両方か
- infinite ammoかmagazine／reloadか
- friendly fireを許可するか
- score／combo／high scoreが目的か
- Boss、Wave、stage completionの有無
- 最大同時敵、Projectile、spawn burst
- Windows、Mobile、Controller、Touchの対象
- visual violence、flash、shake、color依存の制約

既定Profileで安全に解決できるLow Impact項目はAssumptionとしてPreviewへ出し、質問しない。

### 12.3 禁止する推測

- 「弾」をParticleだけで実装する。
- 「銃」をhitscanと決める。
- 「連射」をautomatic、burst、Pattern同時発射のどれかへ無言で決める。
- 「強くする」をDamage、cadence、range、spread、ammo、feedbackの全部へ適用する。
- 「弾幕」をVFX particle countだけ増やして完成とする。
- 「軽くする」をProjectile、敵、Damage、Hit判定の削減へ変換する。
- 「オンライン風」「対戦風」からnetworkingを有効にする。
- Camera shake、recoil、hit flashをGameplay aim／Damageへ使う。

### 12.4 AI Operation

| Operation | Risk | 結果 |
|---|---:|---|
| `operation.shooter.search_catalog` | R0 | Concept、Capability、Definition、Fixture候補 |
| `operation.shooter.resolve_intent` | R0 | `ShooterIntentResolutionV1` |
| `operation.shooter.explain_resolution` | R0 | Field、理由、Assumption、代替、Evidence |
| `operation.shooter.preview_simulation` | R0 | Headless／visual Preview、Cost、Event trace |
| `operation.shooter.compose_recipe` | R2 | Feature ref、Profile、Action role、統合fixture binding |
| `operation.shooter.configure_game_flow` | R1 | Ready／Pause／Result／Restart差分 |
| `operation.shooter.configure_difficulty` | R2 | 許可軸、fidelity floor、capacity差分 |
| `operation.shooter.apply_profile` | R2 | Pack／Input／UI／Asset closure |
| `operation.shooter.propose_native_variant` | R3 | Native Source、Test、capacity evidence、Promotion案 |

Weapon、Fire Mode、Shot、Damage、Vital、Pickup、Encounter、Scoreその他のFeature Source mutationは[Gameplay Feature Packs §9](gameplay-features.md#9-feature-authoring-operation)のFeature owner operationへ委譲する。Shooter operationはFeature contractのFieldを直接writeせず、Feature ChangeSet Receiptをcomposition recipeへbindする。

AI ProviderへC++ pointer、Runtime handle、unbounded projectile list、raw Physics objectを渡さない。Search／Readは必要なCatalog entryとbounded Snapshotだけを返す。

## 13. Editor／AI UX

Shooter Workspaceは専用別Editorではなく、既存Scene、Outliner、Inspector、Graph、Table、AI Partnerへ次のProjectionを追加する。

- Weapon Inspector: cadence、ammo、reload、Delivery、Damage、Cue
- Shot Pattern Preview: origin、direction、tick、capacity、Hit candidate
- Encounter Timeline: Phase、Wave、Spawn、Boss transition
- Damage／Team Matrix: Relation、block、apply、credit
- Score Rule Table: Event、Filter、point、combo、multiplier
- Shooter HUD Preview: keyboard／controller／touch、safe area、locale、color／flash
- Shooter Profiler: Fire request、accepted／rejected、Projectile、query、hit、Damage、queue、pool
- Replay Timeline: Input→Fire→Hit→Damage→Defeat→Scoreのcausal edge

PreviewはSource Definitionと同じC++ evaluator、Collision query normalization、RNG、Target Profileを使う。Editor専用の簡易弾道式を正解系にしない。

AI変更はRequirement、Before／After、Gameplay差分、Presentation差分、capacity impact、Testを分けて表示する。「Damage +10」と「muzzle flashを強くする」を同じ変更としてまとめない。

## 14. Capacity、Memory、Performance

### 14.1 Profileが必須宣言するScale

```text
peak_live_weapon_instance
peak_fire_request_per_tick
peak_shot_activation_per_tick
peak_projectile_spawn_per_tick
peak_live_authoritative_projectile
peak_hitscan_query_per_tick
peak_shot_hit_per_tick
peak_damage_event_per_tick
peak_score_event_per_tick
peak_live_pickup
peak_pickup_collection_per_tick
peak_enemy_and_ally
peak_simultaneous_presentation_cue
```

値はGame BriefのScale intentと一致させる。Runtime Compilerが合格しやすい値へ丸めない。

### 14.2 C1組込み最低Fixture

| Profile | active combat actor | live projectile | projectile spawn／tick | hitscan query／tick | 目標 |
|---|---:|---:|---:|---:|---|
| `profile.shooter.top_down_2d` | 256 | 2,048 | 256 | 128 | 1080p60、authoritative drop 0 |
| `profile.shooter.third_person_3d` | 50 | 256 | 64 | 128 | 1080p60、authoritative drop 0 |

この個数はProduct上限ではない。Project intentが上回る場合は[Performance／capacityが所有する`IntegratedScaleFixtureV1`](../04-runtime/performance-capacity.md#13-integrated-fixtureとqualification)をProject固有Envelopeから生成する。

### 14.3 Pool

Weapon／Projectile／Damage credit／Pickup transactionのRuntime storageはPlay prepareまたはLoading boundaryでProfile capacityを予約する。Poolはprivate実装であり、SourceのProjectile最大数、敵数、Weapon数、Pickup数を下げる理由にしない。

Projectile pool不足時はPattern全体を`ShooterProjectileCapacityExceeded`で拒否し、ammo／cadenceを変更しない。Authoritative Event queue overflowではEventをdrop、merge、sampleせずSessionをfaultする。

Presentation poolだけが不足した場合はcritical cue priorityに従ってPresentationを縮退できるが、Fire、Hit、Damage、Scoreを変更しない。

## 15. Qualification closure

Shooter Genre PackのQualification owner範囲はcomposition recipe、Profile、Game Flow、Action role、`ShooterPerceptionBindingV1`、Shooter統合fixtureに限定する。Feature schema、closed enum、境界値、Fire transaction、canonical ordering、State owner、Save／Replay、failure、Feature contract fixtureの合否は各Feature ownerが発行したQualification Receiptをconsumeし、Shooter側で再所有または再判定しない。

Shooter側は2D top-down／third-person 3Dの両fixtureで、required receipt closure、Perception target selection、lost-target behavior、fire-intent policy、Game Flow、Action role、Profile bindingが接続できることだけを検証する。Packとしてのinstall／apply／update／remove、Project ChangeSet、qualification receiptは[Pack Contract](pack-contract.md)を使い、Capability成熟度と実装順序は[Product Plan](../00-product/product-plan.md)へ委譲する。Profile固有のCamera、Input、UI、Audio、VFX、Asset templateは各ownerのschemaを参照し、同じFieldまたは固定値を本Packへ再定義しない。

## 16. Feature identity migration

旧Genre由来のFeature identityはaliasとして残さず、Feature ownerのmigration stepで次のexact identityへclean renameする。旧Save／Replay／fixture参照はmigration receiptへ記録し、AI catalogは旧名からGenre dependencyを推論しない。

| Old identity | New Feature-owned identity |
|---|---|
| `ShooterProjectileStateV1` | `RangedProjectileStateV1` |
| `SpawnShooterProjectileCommandV1` | `SpawnRangedProjectileCommandV1` |
| `ShooterProjectileSnapshotV1` | `RangedProjectileSnapshotV1` |
| `fixture.shooter.even-floor` | `fixture.feature.ranged_combat.even_floor` |
| `fixture.shooter.explicit-offsets` | `fixture.feature.ranged_combat.explicit_offsets` |
| `MIRAKAN-SHOOTER-{FEATURE_FAILURE}` | owner別`MIRAKAN-FEATURE-*`／`MIRAKAN-RANGED-COMBAT-*`／`MIRAKAN-COMBAT-*`／`MIRAKAN-PICKUP-GRANT-*`／`MIRAKAN-SCORING-*` |
