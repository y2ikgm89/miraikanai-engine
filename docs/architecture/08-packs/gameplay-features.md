# Miraikanai Engine Gameplay Feature Packs

- 文書ID: mirakan.arch.pack-gameplay-features
- 状態: review
- 正本範囲: reusable Gameplay Feature Capabilityのcanonical owner catalog、Public Contract、Schema、State owner、Command／Event／Snapshot、Save／Replay、failure、fixture
- 非正本範囲: Genre composition、Genre Profile／Game Flow／Action role、Pack lifecycle、共通Subsystem契約、Product roadmapは各Ownerを参照
- 依存: [Pack Contract](pack-contract.md)、[Scenario／Stage](scenario-stage.md)、[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[Debugging／Replay](../04-runtime/debugging-observability-replay.md)、[Collision](../05-simulation/collision.md)、[Physics](../05-simulation/physics.md)、[Navigation](../05-simulation/navigation.md)、[Input](../07-platform/input.md)
- 外部根拠検証日: 2026-07-23

## 1. 結論

本書はGenre非依存のGameplay契約を所有するcanonical Feature owner catalogである。Platformer、Puzzle、action、simulation、Genre PackなしのGame Projectは同じPublic Contractを直接利用できる。contract identity、Field、closed enum、algorithm、Save／Replay、failure、fixtureはFeature ownerのversioned surfaceとして変更管理する。

| Feature Pack owner | Provided Capability | Canonical contract location |
|---|---|---|
| `feature.combat` | `capability.gameplay.combat` | 本書: Damage、Vital、Faction／Team |
| `feature.ranged_combat` | `capability.gameplay.ranged_combat` | 本書: Weapon、Shot、Projectile、Ammo／Reload |
| `feature.encounter_spawn` | `capability.gameplay.encounter_spawn` | 本書: Encounter、Spawn |
| `feature.scoring` | `capability.gameplay.scoring` | 本書: Score rule／state |
| `feature.pickup_grant` | `capability.gameplay.pickup_grant` | 本書: Pickup、typed Grant |
| `feature.interaction` | `capability.gameplay.interaction` | [Gameplay Programming Model](../03-authoring/gameplay-programming-model.md) |
| `feature.character_locomotion` | `capability.gameplay.character_locomotion` | [Navigation](../05-simulation/navigation.md)のMotion Executor public contract。Physicsは任意reference Provider |
| `feature.path_following` | `capability.gameplay.path_following` | [Navigation](../05-simulation/navigation.md) |
| `feature.scenario_stage` | `capability.gameplay.scenario_stage` | [Scenario／Stage](scenario-stage.md) |

## 2. Ownership、manifest、compatibility

本書のembedded contractは下表のFeature Pack ownerごとに分離して`PackManifestV1`へ登録する。各recordの`pack_version`、`content_hash`、`minimum_engine_contract_ref`、Target、license、provenanceはPack Contractの解決規則に従い、表にない配列Fieldは空配列である。Feature間依存はDAG、Genre／Fixtureへの依存辺は0件とする。

| PackManifestV1 record | required_feature_pack_refs[] | required_capability_refs[] | provided_capability_refs[] | public_contract_refs[]／runtime_port_refs[] | game_system_spec_refs[]／validator_refs[]／test_scenario_refs[] |
|---|---|---|---|---|---|
| `feature.combat@1` (`pack_kind=feature`) | `[]` | `[]` | `[capability.gameplay.combat]` | `[DamageSpecV1, VitalDefinitionV1, VitalStateV1, TeamRelationPolicyV1]`／`[DamageApplicationPortV1]` | `[game_system.engine.combat, game_system.engine.vital]`／`[validator.feature.combat.v1]`／`[fixture.feature.combat.contract]` |
| `feature.ranged_combat@1` (`pack_kind=feature`) | `[feature.combat@1]` | `[capability.gameplay.combat]` | `[capability.gameplay.ranged_combat]` | `[WeaponDefinitionV1, FireModeDefinitionV1, ShotPatternDefinitionV1, ShotDeliveryDefinitionV1, ProjectileDefinitionV1, RangedProjectileStateV1, AmmoPolicyV1, ReloadPolicyV1, WeaponInstanceStateV1, WeaponLoadoutStateV1]`／`[CollisionQueryPortV1]` | `[game_system.engine.weapon, game_system.engine.ranged_projectile]`／`[validator.feature.ranged_combat.v1]`／`[fixture.feature.ranged_combat.contract, fixture.feature.ranged_combat.even_floor, fixture.feature.ranged_combat.explicit_offsets]` |
| `feature.encounter_spawn@1` (`pack_kind=feature`) | `[]` | `[]` | `[capability.gameplay.encounter_spawn]` | `[EncounterDefinitionV1, EncounterPhaseV1, EncounterWaveV1, SpawnGroupV1]`／`[]` | `[game_system.engine.encounter]`／`[validator.feature.encounter_spawn.v1]`／`[fixture.feature.encounter_spawn.contract]` |
| `feature.scoring@1` (`pack_kind=feature`) | `[]` | `[]` | `[capability.gameplay.scoring]` | `[ScoreRuleSetV1, ScoreStateV1]`／`[ScoreAwardPortV1]` | `[game_system.engine.score]`／`[validator.feature.scoring.v1]`／`[fixture.feature.scoring.contract]` |
| `feature.pickup_grant@1` (`pack_kind=feature`) | `[]` | `[]` | `[capability.gameplay.pickup_grant]` | `[PickupDefinitionV1, PickupInstanceStateV1, GrantRequestV1, GrantResultV1]`／`[GrantRequestPortV1]` | `[game_system.engine.pickup]`／`[validator.feature.pickup_grant.v1]`／`[fixture.feature.pickup_grant.provider_neutral]` |
| `feature.interaction@1` (`pack_kind=feature`) | `[]` | `[]` | `[capability.gameplay.interaction]` | `[mirakan.arch.gameplay-programming-model#InteractionDefinitionV1, mirakan.arch.gameplay-programming-model#InteractionRequestV1, mirakan.arch.gameplay-programming-model#InteractionSnapshotV1]`／`[]` | `[mirakan.arch.gameplay-programming-model#Engine-Standard-Interaction-System]`／`[validator.feature.interaction.v1]`／`[fixture.feature.interaction.contract]` |
| `feature.character_locomotion@1` (`pack_kind=feature`) | `[]` | `[]` | `[capability.gameplay.character_locomotion]` | Feature-owned `[type.feature.character_locomotion.gameplay_motion_intent, type.feature.character_locomotion.motion_executor_selection_state]`＋Navigation-owned `[type.navigation.motion_executor_intent_batch, mirakan.arch.simulation-navigation#MotionExecutorPortV1]`／`[mirakan.arch.simulation-navigation#MotionExecutorPortV1]` | `[game_system.engine.character_locomotion.binding]`／`[validator.feature.character_locomotion.v1]`／`[fixture.feature.character_locomotion.motion_executor]` |
| `feature.path_following@1` (`pack_kind=feature`) | `[]` | `[capability.simulation.navigation]` | `[capability.gameplay.path_following]` | `[mirakan.arch.simulation-navigation#PathFollowRequestV1, mirakan.arch.simulation-navigation#PathFollowerStateV1, mirakan.arch.simulation-navigation#MovementIntentV1, mirakan.arch.simulation-navigation#MotionExecutorPortV1]`／`[mirakan.arch.simulation-navigation#MotionExecutorPortV1]` | `[mirakan.arch.simulation-navigation#Path-Following]`／`[validator.feature.path_following.v1]`／`[fixture.feature.path_following.executor_stub]` |
| `feature.scenario_stage@1` (`pack_kind=feature`) | `[]` | `[]` | `[capability.gameplay.scenario_stage]` | `[mirakan.arch.pack-scenario-stage#StageDefinitionV1, mirakan.arch.pack-scenario-stage#CompletionContractV1, mirakan.arch.pack-scenario-stage#StageRuntimeStateV1, mirakan.arch.pack-scenario-stage#StageTransitionDestinationV2, mirakan.arch.pack-scenario-stage#StageTransitionPolicyV1, mirakan.arch.pack-scenario-stage#StageTransitionRequestV2]`／`[mirakan.arch.pack-scenario-stage#StageActivationPortV1, mirakan.arch.pack-scenario-stage#StageTransitionPortV2]` | `[mirakan.arch.pack-scenario-stage#game_system.engine.scenario_stage]`／`[validator.feature.scenario_stage.v1]`／`[fixture.feature.scenario_stage.none, fixture.feature.scenario_stage.explicit_outcomes, fixture.feature.scenario_stage.transition, fixture.feature.scenario_stage.worldless-ui, fixture.feature.scenario_stage.worldless-headless]` |

`feature.interaction`、`feature.path_following`、`feature.scenario_stage`のrequired capabilityとpublic surfaceは、それぞれ[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)、[Navigation](../05-simulation/navigation.md)、[Scenario／Stage](scenario-stage.md)の既存Ownerへexact refで接続する。Character Locomotionは本書が所有するGameplay intent／selection stateの2型を登録し、Navigation-owned `MotionExecutorPortV1`／`MotionExecutorIntentBatchV1`を外部Ownerへのexact refとして接続する。Feature catalogは参照先OwnerのSchemaを複製しない。

上表の`type.*`／`capability.*`は表示上IDだけを示すが、Manifest保存値はすべて`McdContractRefV1 {id, version, contract_set_hash}`である。Pack refは`PackContractRefV1 {pack_id, pack_version, pack_hash}`、Architecture contract anchorはowner文書revision／content hash付き`ArchitectureContractRefV1`で保存し、bare ID／anchorだけを永続参照にしない。

`MotionExecutorPortV1`の唯一のcanonical ownerはNavigationである。本書はexact public contract refだけをmanifestへ登録する。Character Locomotion PackはPhysics unavailableでもinstall／validateでき、Physics Character MotorはC1 reference recipe／qualification Providerを選んだ時だけ依存closureへ入る。consumerはProvider Backend object、native body pointer、render transformへ直接依存せず、Portのtyped intent／resolved resultだけを使う。

## 3. Canonical data model

すべてのSource objectはUUIDv7 `StableId`、MCD Typeはexact `McdContractRefV1`、Derived Artifactは`ArtifactRefV1`を使う。表示名、Asset path、C++型名、配列indexをidentityにしない。

### 3.1 `WeaponDefinitionV1`

```text
WeaponDefinitionV1
  weapon_definition_id: StableId
  technical_name
  semantic_tags[]
  fire_bindings[1..2]:
    semantic_role: primary | secondary
    fire_mode_ref
    shot_delivery_ref
    shot_pattern_ref
    damage_spec_ref
    presentation_cue_ref
  ammo_policy_ref
  reload_policy_ref
  allowed_slot_tags[1..8]
  target_profile_constraints[]
```

`primary` Bindingはexactly one、`secondary`はzero or oneとし、配列indexから役割を推測しない。`WeaponDefinitionV1`は調整dataであり、owner Entity、現在弾数、cooldown、reload進行を持たない。C++ function名、callback、Physics Body pointer、VFX instanceを参照しない。

### 3.2 `FireModeDefinitionV1`

```text
FireModeDefinitionV1
  trigger_mode: single | automatic | burst
  activation_count_per_cycle: uint16
  cycle_ticks: uint16
  cycle_distribution_fixture_ref: fixture.feature.ranged_combat.even_floor | fixture.feature.ranged_combat.explicit_offsets
  activation_offsets_ticks[]: optional
  ammo_cost_per_activation: uint16
  release_policy: stop_unfired | complete_started_cycle
  fire_while_reloading: false
```

範囲は`activation_count_per_cycle=[1,3600]`、`cycle_ticks=[1,3600]`、`activation_count_per_cycle <= cycle_ticks`とする。`burst`だけはcountを`[1,32]`へ制限する。C1は一Weaponにつき一tick最大一Activationである。Patternが一Activationから複数Shotを生成する。この上限はper Weapon instanceの不変条件であり、Bindingごとではない。同tickに`primary`と`secondary`の両Bindingが発火許可となった場合は`primary`のActivationだけを実行し、`secondary`を`FireRejectedCadence`で拒否してそのcadence cycle開始tickを次の有効tickへ繰り延べる。この裁定順は固定であり、同じPublic Contractへ適合するすべての実装とReplayで同一結果とする。

`fixture.feature.ranged_combat.even_floor`はcycle内のActivation `i=[0,count-1]`を`floor(i * cycle_ticks / activation_count_per_cycle)` tickへ配置し、`activation_offsets_ticks`を持たない。`fixture.feature.ranged_combat.explicit_offsets`は`activation_count_per_cycle`と同数のoffsetを`[0,cycle_ticks-1]`へstrictly increasingで持つ。cycle開始tickとDefinition revisionが同じならscheduleは同じである。「毎秒7発」は`7 activation / 60 tick`へexactに解決し、float timerを累積しない。

`single`はcount 1、offset 0で、trigger rising edgeごとに一cycleだけ評価する。`automatic`はtrigger hold中にcycleを反復する。`burst`はrising edgeで一つの有限cycleを開始し、通常は`fixture.feature.ranged_combat.explicit_offsets`で`[0,5,10]`等を表す。release後の再押下なしに次cycleを開始しない。これによりautomatic cadence、burst内間隔、burst後cooldownを同じscheduleで一意に表す。

`release_policy`はcycle内の未発射Activationだけへ適用する。`single`／`automatic`は`stop_unfired`、`burst`はGame Briefで`stop_unfired`または`complete_started_cycle`を選ぶ。Weapon switch、reload開始、owner defeat、operation eligibility policy denialは未発射Activationを常にcancelし、消費済みammoと既発射Shotだけを維持する。

### 3.3 `ShotPatternDefinitionV1`

```text
ShotPatternDefinitionV1
  pattern_kind: single | fixed_offsets | even_fan | radial | deterministic_cone
  shot_count: uint8
  origin_offsets[]
  speed_scale_q16[]
  fixed_direction_offsets_turn_q16[]: optional
  fan_start_turn_q16: optional
  fan_end_turn_q16: optional
  radial_phase_turn_q16: optional
  cone_half_angle_turn_q16: optional
  deterministic_rng_stream_id: optional
```

`shot_count=[1,64]`とし、originとspeed配列は1または`shot_count`要素で、1要素は全Shotへbroadcastする。`shot_count`はhitscan ray数とProjectile spawn数の両方に使い、名前だけでDeliveryを推測しない。角度はturn単位のsigned Q16、倍率はunsigned Q16 `[0.0625,16]`とする。

- `single`: `shot_count=1`で、方向offset 0。5つのoptional angle／RNG Fieldを持たない。
- `fixed_offsets`: `fixed_direction_offsets_turn_q16`を`shot_count`個持ち、他のangle／RNG Fieldを持たない。
- `even_fan`: start／endを必須とし、端点込みで等分する。count 1ではstartとendの一致を要求する。
- `radial`: phaseを必須とし、Profile forwardを0として一周をcount等分する。
- `deterministic_cone`: half angleとauthoritative `DeterministicRngV1`の登録済みstreamを必須とし、他のangle Fieldを持たない。

VFX RNG、render frame、wall clock、Entity pointer、container iteration順をPatternへ使わない。

### 3.4 `ShotDeliveryDefinitionV1`

```text
ShotDeliveryDefinitionV1
  delivery_kind: hitscan | projectile
  max_range_m
  collision_query_profile_ref
  projectile_definition_ref: optional
  max_hits_per_activation
  hit_selection: closest_per_shot
```

hitscanはCollision規約のRay／Shape Castを使う。ProjectileはRanged Projectile Systemのdomain-local recordを使い、C1では各tickの移動segmentをswept Queryとして検証する。Particle collision、depth buffer、render occlusionをHit Evidenceにしない。

`delivery_kind=hitscan`では`projectile_definition_ref`を持たず、Patternの`speed_scale_q16`を全要素1.0、`projectile`ではrefを必須とする。`max_hits_per_activation=[1,64]`かつ`shot_count`以下とし、各Shotは`closest_per_shot`で最大一Hitを生成する。penetration、同一Shotの複数Hit、`ordered_all`は現在のcontractの対象外として拒否する。これらの将来scopeまたはschema判断は[Product Plan](../00-product/product-plan.md)へ委譲する。Projectileは`lifetime_ticks`またはspawn originから`max_range_m`へ到達した早い方でexpireする。

### 3.5 `ProjectileDefinitionV1`

```text
ProjectileDefinitionV1
  projectile_definition_id: StableId
  dimension: two_d | three_d
  speed_mps
  lifetime_ticks
  query_shape_ref
  collision_query_profile_ref
  owner_ignore_ticks
  on_hit_policy: stop
  presentation_cue_ref
```

`speed_mps=(0,10,000]`、`lifetime_ticks=[1,36,000]`、`owner_ignore_ticks=[0,60]`とする。C1 Projectileはscale、arbitrary acceleration、homing target、bounce count、Physics Backend objectを持たない。

Runtime recordは少なくとも次を持つ。

```text
RangedProjectileStateV1
  projectile_spawn_id: uint64
  definition_ref
  delivery_definition_ref
  owner_entity_ref
  owner_team_ref
  position_m
  direction_unit
  speed_mps
  age_ticks
  rng_draw_count
  status: pending | active | hit | expired
```

`projectile_spawn_id`はPlay session内でRanged Projectile Systemが単調増加させ、0を使わない。Pool slot、pointer、Physics handleをSave／Replay identityにしない。

`delivery_definition_ref`はspawn元の`ShotDeliveryDefinitionV1`を指す。`max_range_m`によるexpireは、C1の直線・等速運動を前提に`speed_mps`と`age_ticks`から導出した移動距離と、`delivery_definition_ref`が示す`max_range_m`だけで判定する。spawn originの保存を必要とせず、Save／Load後も同じcanonical recordから同じexpire tickへ再決定できる。

### 3.6 `AmmoPolicyV1`と`ReloadPolicyV1`

```text
AmmoPolicyV1
  ammo_kind: infinite | magazine_reserve
  magazine_capacity
  initial_magazine
  reserve_capacity
  initial_reserve

ReloadPolicyV1
  reload_kind: none | full_magazine
  duration_ticks: optional
  trigger_policy: manual | manual_or_empty, optional
  interruption: not_interruptible | cancel_preserve_ammo, optional
```

capacityは`[0,65,535]`とし、initial値は対応capacity以下とする。`infinite`は4つのcapacity／initial Fieldを0、`ReloadPolicyV1.reload_kind=none`とし、ammo FieldをRuntime State／HUD／Saveへ作らない。`magazine_reserve`は`magazine_capacity>=1`を要求する。`reload_kind=none`では3つのoptional Fieldを持たず、`full_magazine`では`duration_ticks=[1,36,000]`とtrigger／interruptionをすべて必須とする。C1 reloadは完了時に必要量をreserveからmagazineへ一度だけ移し、animation eventやAudio completionをauthoritative完了条件にしない。

### 3.7 `WeaponInstanceStateV1`と`WeaponLoadoutStateV1`

```text
WeaponInstanceStateV1
  weapon_instance_id: StableId
  owner_entity_ref
  definition_ref
  slot_index
  magazine_ammo: optional
  reserve_ammo: optional
  fire_binding_states[1..2]:
    semantic_role: primary | secondary
    cadence_cycle_start_tick
    cadence_activation_index
    trigger_held
  reload_state
  reload_complete_tick
  enabled
```

`magazine_reserve`だけが二つのammo Fieldを持ち、`infinite`では持たない。Binding StateはDefinitionのsemantic roleとexactに一致し、primary／secondaryのcadenceやhold状態を共有しない。Weapon SystemだけがこのTypeを所有する。HUD、Character、Input、VFXは直接writeせずSnapshotまたはEventを使う。

owner Entityごとのslot集合は`WeaponLoadoutStateV1`が所有する。

```text
WeaponLoadoutStateV1
  owner_entity_ref
  slots[1..8]:
    slot_index: uint8
    weapon_instance_ref: optional
  active_slot_index: uint8
```

slot数はProfileが`[1,8]`で宣言し、`slot_index`は`[0,slot数-1]`で重複しない。`active_slot_index`は装備済みslotだけを指す。Weapon SystemだけがこのTypeをwriteする。初期装備はProfile Templateが`slot_index`ごとの`weapon_definition_ref`列として宣言し、Runtimeが暗黙生成しない。

`WeaponGrantRequestAdapterV1`は登録済みWeapon grant payloadを受け、空きslotのうち最小`slot_index`へ割り当てる。既に所持する`WeaponDefinitionV1`を重複取得した場合はWeaponを追加せず、当該instanceのreserve ammoが`reserve_capacity`未満ならpayloadの補充量をcapacityまで加算してacceptし、既にcapacityなら`rejected_no_capacity`を返す(`infinite`では変化なしでacceptする)。全slotが使用中の新規Weapon grantも`rejected_no_capacity`である。このadapterは`feature.ranged_combat`が任意登録し、Grant request producer側の依存辺にはならない。`next_weapon`／`previous_weapon`は装備済みslotだけを対象に`slot_index`の昇順／降順でwrap巡回する。

### 3.8 `DamageSpecV1`

```text
DamageSpecV1
  damage_spec_id: StableId
  damage_type_ref
  base_damage_points
  application_kind: direct | radial
  radial_damage_ref: optional
  shield_interaction
  invulnerability_interaction
  team_policy_ref
  hit_reaction_tag
  defeat_credit_policy
```

全数値はfinite binary32とし、`base_damage_points=[0,1.0e9]`である。`direct`は`radial_damage_ref`を持たず、`radial`は必須とする。negative DamageをHealingとして再解釈しない。Healingは`VitalGrantRequestAdapterV1`が受けるtyped payloadを使い、Damage経路で表現しない。`defeat_credit_policy`のclosed valueは`last_hit`だけとする。assist配分、寄与比例、時間減衰creditは現在のcontractの対象外として拒否し、将来scopeまたはschema判断は[Product Plan](../00-product/product-plan.md)へ委譲する。

### 3.9 `RadialDamageDefinitionV1`

```text
RadialDamageDefinitionV1
  radial_damage_id: StableId
  inner_radius_m
  outer_radius_m
  falloff: step | linear_q16
  max_targets: uint16
  occlusion_policy: none | collision_query
  occlusion_query_profile_ref: optional
```

`0 <= inner_radius_m <= outer_radius_m <= 1,000`、`max_targets=[1,256]`とする。`none`はquery refを持たず、`collision_query`は必須とする。候補はdistance、Target Stable IDの順でcanonicalに選び、`max_targets`超過をpartial Damageで継続せず`CombatQueryCapacityExceeded`としてtickをfaultする。`linear_q16`はinnerで1.0、outerで0.0のQ16倍率とし、Gameplay DamageをVFX radiusから取得しない。

C1のradial originはDeliveryが確定したcanonical Hit位置であり、HitがないActivationではradial Damageを生成しない。timer、remote detonation、persistent area Damageは現在のcontractの対象外であり、将来scopeまたはschema判断は[Product Plan](../00-product/product-plan.md)へ委譲する。

### 3.10 `TeamPolicyV1`

Relationは`self | ally | neutral | hostile`のclosed enumとする。Policyは各Relationへ`ignore | block_without_damage | apply_damage`を一つ指定する。C1既定はself／ally=`ignore`、neutral=`block_without_damage`、hostile=`apply_damage`である。

friendly fireの変更はGame BriefのHigh Impact項目であり、AIが武器単位の見た目や難易度要求から推測して変更しない。

### 3.11 `VitalStateV1`

```text
VitalStateV1
  owner_entity_ref
  health_points
  max_health_points
  shield_points
  max_shield_points
  invulnerable_until_tick
  life_state: alive | defeated
  last_damage_credit
```

Vital SystemだけがHealth、Shield、invulnerability、defeatを書き込む。Combat SystemはHit EvidenceとPolicyを検証して`ApplyVitalDeltaCommandV1`を生成し、Vital Stateを直接変更しない。Presentation Gameplay Cueは`DamageAppliedEventV1`と`DefeatEventV1`を購読する。

Combat Systemが所有するDamage credit ledgerは、targetごとに最新の`DamageCreditRecordV1`を一件保持する。

```text
DamageCreditRecordV1
  target_entity_ref
  credited_entity_ref
  credited_team_ref
  damage_spec_ref
  credit_tick
```

`VitalStateV1.last_damage_credit`は同recordへの参照であり、`defeat_credit_policy=last_hit`の確定、`DefeatEventV1`のcredit、Score加点の根拠に使う。creditはDamage適用が確定したtickだけで更新し、blockされたDamageでは更新しない。

### 3.12 `ScoreRuleDefinitionV1`と`ScoreStateV1`

```text
ScoreRuleDefinitionV1
  rule_id: StableId
  event_kind: defeat | pickup_collected | objective_completed | wave_completed | boss_defeated
  base_points: int64
  combo_increment
  combo_expire_ticks
  multiplier_curve_ref: optional
  target_filter: optional
    hostile_team_filter_ref: optional
    enemy_archetype_refs[0..64]

ScoreMultiplierCurveV1
  curve_id: StableId
  points[1..16]:
    combo_count_threshold: uint32
    multiplier_q16

ScoreStateV1
  participant_ref
  current_score: int64
  combo_count: uint32
  multiplier_q16
  combo_expire_tick
  session_high_score: int64
  persistent_high_score: int64
```

Score SystemだけがScore Stateを所有する。Score加算は`DamageApplied`ではなく、Ruleが`event_kind`で指定した確定Eventだけから行う。`event_kind`は上記closed enumだけを許可し、unknown enumを拒否する。同じDefeatをHit数だけ重複加点しない。

`multiplier_curve_ref`は`ScoreMultiplierCurveV1`だけを参照する。curveの`combo_count_threshold`は先頭0のstrictly increasingとし、現在comboに対する倍率は閾値以下で最大の点をstepとして使い、補間しない。`target_filter`を持つRuleは、filterに合致しない確定Eventへ適用しない。

`int64` overflowをsaturateまたはwrapせず`ScoringOverflow`としてPlay sessionをfaultする。persistent high scoreはSave generationへ含める。

### 3.13 `EncounterPatternDefinitionV1`

```text
EncounterPatternDefinitionV1
  encounter_id: StableId
  phases[1..64]: EncounterPhaseV1
  waves[1..256]: EncounterWaveV1
  spawn_groups[1..1024]: SpawnGroupV1
  completion_condition: all_defeated | goal_reached | time_expired | boss_defeated
  goal_ref: optional
  time_limit_ticks: optional
  boss_phase_refs[0..32]
  difficulty_profile_ref
  rng_stream_ids[]

EncounterPhaseV1
  phase_id: StableId
  wave_refs[1..256]

EncounterWaveV1
  wave_id: StableId
  start_tick
  spawn_group_refs[1..1024]

SpawnGroupV1
  spawn_group_id: StableId
  enemy_archetype_ref
  spawn_transform_ref
  spawn_count: uint16
  spawn_start_offset_ticks
  spawn_interval_ticks
```

各waveは丁度一つのphaseの`wave_refs`に、各spawn groupは丁度一つのwaveの`spawn_group_refs`に現れ、未参照と重複参照を拒否する。`boss_phase_refs`は`phases`のsubsetである。tickはEncounter開始を0とし、`start_tick=[0,36,000]`、`spawn_start_offset_ticks=[0,36,000]`、`spawn_interval_ticks=[0,3600]`、`spawn_count=[1,256]`とする。`spawn_count>=2`は`spawn_interval_ticks>=1`を要求する。

`completion_condition`はclosed enumであり、`goal_reached`だけが`goal_ref`を必須として他では持たない。`time_expired`だけが`time_limit_ticks=[1,36,000]`を必須として他では持たない。`boss_defeated`は`boss_phase_refs`が非空であることを要求する。

Spawn順、Wave遷移、Boss phaseはfixed tick、Stable ID順、登録済みRNG streamで決める。VFX completion、Audio duration、Animationのpresentation-only Notifyを進行条件にしない。

### 3.14 `PickupDefinitionV1`、`GrantRequestV1`、`PickupInstanceStateV1`

```text
PickupDefinitionV1
  pickup_definition_id: StableId
  grant_payload:
    grant_type_ref: McdContractRefV1
    payload_ref: McdContractRefV1
  collection_filter_ref
  presentation_cue_ref

GrantRequestV1
  grant_transaction_id: StableId
  grant_type_ref: McdContractRefV1
  payload_ref: McdContractRefV1
  collector_entity_ref
  source_pickup_instance_ref

GrantResultV1
  grant_transaction_id: StableId
  result: accepted | rejected_no_capacity | rejected_filter | rejected_unsupported_type | failed
  applied_revision_ref: optional
  diagnostic_ref: optional

PickupInstanceStateV1
  pickup_instance_id: StableId
  definition_ref
  state: available | pending_grant | collected
  grant_transaction_id: optional
  pending_collector_ref: optional
  collected_by_ref: optional
  collected_tick: optional
```

`feature.pickup_grant`はGrant providerの型、State、capacityを知らない。Definitionは登録済み`grant_type_ref`とtyped payloadだけを保持し、grant種別からWeapon、Vital、Scoreその他のCapabilityを推論しない。このためmanifestのrequired Feature／Capabilityは空であり、providerが0件でもPackをinstallできる。

```text
GrantRequestPortV1
  port_id
  accepted_grant_type_refs[]
  request_contract_ref: GrantRequestV1
  result_contract_ref: GrantResultV1
  provider_state_owner_ref
  consume_phase_ref
  max_requests_per_tick
```

各provider FeatureまたはProjectは自身のmanifestから`GrantRequestPortV1`実装を登録する。同じ`grant_type_ref`へのactive providerはexactly oneとし、未登録型は`rejected_unsupported_type`、capacity不足は`rejected_no_capacity`を返す。Pickup Systemはprovider Stateを直接writeしない。

同一Pickupへ同tickに複数collectorが接触した場合は`pickup_instance_id, collector StableId`順の先頭だけを`pending_grant`へ遷移する。providerが`accepted`を返した時だけ`collected`へ進み、通常拒否では`available`へ戻す。pending中の重複collectionを禁止し、queue／invariant失敗はpending transactionを保持してSession faultとする。現在のcontractはone-shot collectionだけを扱い、respawn、random loot table、inventory weightは対象外として拒否する。

## 4. Game SystemとState owner

### 4.1 Engine Standard System

| System Contract | Scope | 所有State | 主な責務 |
|---|---|---|---|
| `game_system.engine.weapon` | `scope.core.entity` | `WeaponInstanceStateV1`、`WeaponLoadoutStateV1` | trigger、cadence、ammo、reload、switch、Fire transaction |
| `game_system.engine.ranged_projectile` | `scope.core.world` | `RangedProjectileStateV1` collection | spawn、swept query、hit／expire、capacity |
| `game_system.engine.combat` | `scope.core.world` | Damage credit ledger | Hit Evidence、Team、Damage rule、creditの検証 |
| `game_system.engine.vital` | `scope.core.entity` | `VitalStateV1` | Health、Shield、invulnerability、defeat |
| `game_system.engine.score` | `scope.feature.scoring.instance` | `ScoreStateV1` | 加点、combo、multiplier、high score |
| `game_system.engine.pickup` | `scope.core.world` | `PickupInstanceStateV1` collection | overlap Evidence、provider-neutral typed Grant、one-shot state |
| `game_system.engine.encounter` | `scope.feature.encounter_spawn.instance` | Encounter runtime state | Wave、Spawn、Boss phase、completion |
| `game_system.engine.character_locomotion.binding` | `scope.core.entity` | `MotionExecutorSelectionStateV1` | provider-neutral intent proposalとselected Motion Executor binding |

同じPublic Contractへ適合するProject-defined実装は許可するが、Engine Standardと同時にactiveにしない。WeaponとVitalをCharacter Systemのprivate Fieldへ隠さず、Public State owner tableへ出す。

`feature.scoring`と`feature.encounter_spawn`は`RuntimeScopeTypeCatalogV1`へ次のexact rowを登録する。

| `scope_type_ref` | `instance_key_schema_ref` | `owner_ref` | `lifetime_ref` | `save_replay_policy_ref` | `activation_condition_ref` | `deactivation_condition_ref` |
|---|---|---|---|---|---|---|
| `scope.feature.scoring.instance` | `type.runtime_scope.key.feature_scoring_uuidv7` | `owner.feature.scoring` | `policy.runtime_scope.lifetime.feature_scoring_instance` | `policy.runtime_scope.save_replay.feature_scoring` | `policy.runtime_scope.activation.feature_scoring_request_ready` | `policy.runtime_scope.deactivation.feature_scoring_stop_or_fault` |
| `scope.feature.encounter_spawn.instance` | `type.runtime_scope.key.feature_encounter_spawn_uuidv7` | `owner.feature.encounter_spawn` | `policy.runtime_scope.lifetime.feature_encounter_spawn_instance` | `policy.runtime_scope.save_replay.feature_encounter_spawn` | `policy.runtime_scope.activation.feature_encounter_spawn_request_ready` | `policy.runtime_scope.deactivation.feature_encounter_spawn_stop_or_fault` |

各scope typeは`RuntimeScopeTypeRefV1 {scope_type_id, scope_type_version=1, scope_type_hash}`、type／policy cellは`McdContractRefV1 {id, version=1, contract_set_hash}`、owner cellは`RuntimeScopeOwnerRefV1 {owner_id, owner_revision, owner_content_hash}`として保存し、表の裸IDを永続化しない。全dependency recordをactive Scope Registryへ登録する。旧`play_session`／`encounter_instance`からclean migrationし、aliasを残さない。Save identity、Replay identity、ephemeral runtime generationを別Fieldで保持し、複数instanceのStateをSource IDまたはRuntime handleで合成しない。

Ranged Combat ownerはShooterその他のconsumerが参照するPort／Eventを次のMCD typeとして登録する。

```text
CollisionQueryPortMessageV1
  request_schema_ref: {id=type.feature.ranged_combat.collision_query_request, version=1, contract_set_hash}
  result_schema_ref: {id=type.feature.ranged_combat.collision_query_result, version=1, contract_set_hash}
  consume_phase_ref
  producer_system_ref: GameSystemContractRefV1
  request_id
  payload_hash

ShotHitEventV1
  MCD ref: {id=type.feature.ranged_combat.shot_hit_event, version=1, contract_set_hash}
  shot_ref
  target_ref
  normalized_collision_evidence_ref
  producer_system_ref
  accepted_tick
```

Port descriptorのexact MCD refは`{id=type.feature.ranged_combat.collision_query_port, version=1, contract_set_hash}`であり、request／result schema、Collision owner、consume phase、capacity、failure Diagnosticを閉じる。Genre Pack、Project provider、fixtureはこの三refを再定義せず、version／Contract set hash込みで参照する。旧`mirakan.port.*`、`mirakan.event.*`、PascalCase ID、ID内`@1`をcurrent validatorはrejectする。

`game_system.engine.character_locomotion.binding`が使用するFeature-owned型を次の2型へ固定し、transport batchをFeature ownerへ複製しない。

```text
GameplayMotionIntentV1
  intent_id
  actor_ref
  actor_generation
  desired_linear_motion
  desired_angular_motion
  valid_for_tick
  movement_profile_ref

MotionExecutorSelectionStateV1
  actor_ref
  actor_generation
  provider_binding_ref
  provider_binding_hash
  executor_capability_ref: McdContractRefV1(kind=capability)
  movement_profile_ref
  movement_profile_hash
  binding_generation
```

2型のexact MCD refは`{id=type.feature.character_locomotion.gameplay_motion_intent, version=1, contract_set_hash}`と`{id=type.feature.character_locomotion.motion_executor_selection_state, version=1, contract_set_hash}`である。Navigation intentは`type.navigation.movement_intent`、Animation proposalは`type.animation.root_motion_proposal`、transportはNavigation-owned `type.navigation.motion_executor_intent_batch`を同じ形式で参照する。IDに`mirakan.schema`／`mirakan.contract`、PascalCase、`@1`を含めない。

`game_system.engine.character_locomotion.binding`の全mandatory Fieldは次へ固定する。

```text
GameSystemSpecV1
  mcd_version: 1
  kind: game_system
  id: game_system.engine.character_locomotion.binding
  version: 1
  status: active
  title: Character Locomotion Motion Executor Binding
  description: provider-neutral proposal validation, batching, and selected Provider dispatch
  owners: [owner.feature.character_locomotion]
  requirement_refs:
    [{id=requirement.feature.character_locomotion.bind_selected_executor, version=1, contract_set_hash},
     {id=requirement.non_responsibility.character_locomotion.no_transform_write, version=1, contract_set_hash},
     {id=requirement.non_responsibility.character_locomotion.no_physics_write, version=1, contract_set_hash}]
  rationale_refs: [mirakan.arch.pack-gameplay-features#41-character-locomotion-binding-system]
  since_contract_set: 1
  supersedes: []
  tags: [character_locomotion, motion_executor, provider_neutral]
  system_origin: engine_standard
  semantic_role_ids:
    [{role_id=role.game_system.character_locomotion.binding, role_version=1, role_hash}]
  responsibility_requirement_ids:
    [{id=requirement.feature.character_locomotion.bind_selected_executor, version=1, contract_set_hash}]
  non_responsibility_requirement_ids:
    [{id=requirement.non_responsibility.character_locomotion.no_transform_write, version=1, contract_set_hash},
     {id=requirement.non_responsibility.character_locomotion.no_physics_write, version=1, contract_set_hash}]
  runtime_scope_type_ref:
    {scope_type_id=scope.core.entity, scope_type_version=1, scope_type_hash}
  state_class: authoritative
  owned_state_type_refs:
    [{id=type.feature.character_locomotion.motion_executor_selection_state, version=1, contract_set_hash}]
  read_snapshot_type_refs: []
  accepted_command_type_refs:
    [{id=type.feature.character_locomotion.gameplay_motion_intent, version=1, contract_set_hash},
     {id=type.navigation.movement_intent, version=1, contract_set_hash},
     {id=type.animation.root_motion_proposal, version=1, contract_set_hash}]
  emitted_event_type_refs: []
  emitted_port_message_type_refs:
    [{id=type.navigation.motion_executor_intent_batch, version=1, contract_set_hash}]
  provided_capability_refs:
    [{id=capability.gameplay.character_locomotion, version=1, contract_set_hash}]
  required_capability_refs: []
  allowed_phase_ids: [T40_MotionIntent]
  dependency_edges:
    [{dependency_id=dependency.game_system.character_locomotion.navigation_motion_executor_port,
      dependency_version=1, dependency_hash}]
  implementation_policy:
    {policy_id=implementation_policy.feature.character_locomotion.binding, policy_version=1, policy_hash}
  save_replay_contract_ref:
    {contract_id=save_replay.feature.character_locomotion.binding, contract_version=1, contract_hash}
  behavior_budget_refs:
    [{budget_id=budget.feature.character_locomotion.binding.per_target, budget_version=1, budget_hash}]
  authoring_surface_ids: [form, table, graph]
  fallback_contract: no_fallback(reason=selected motion authority cannot be inferred)
  fixture_ids:
    [{fixture_id=fixture.feature.character_locomotion.motion_executor, fixture_version=1, fixture_hash}]
  compatibility_invariant_ids:
    [{invariant_id=invariant.character_locomotion.accepted_intent_subset, invariant_version=1, invariant_hash},
     {invariant_id=invariant.character_locomotion.single_selected_executor, invariant_version=1, invariant_hash},
     {invariant_id=invariant.character_locomotion.no_transform_or_physics_write, invariant_version=1, invariant_hash}]
  extension_policy: sealed
```

本Systemだけが`MotionExecutorSelectionStateV1`を所有し、Navigation-owned `MotionExecutorIntentBatchV1`の正規producerである。Batch自体とPort schemaのOwnerはNavigationであり、Event inventoryへ入れない。Gameplay、Navigation、Animationの全proposalは本Systemのaccepted Commandを通り、selected Providerへ直接提出しない。dependency edgeはNavigation-owned `MotionExecutorPortV1`へのexact contract dependencyであり、System Graphの別ownerへ直接writeしない。Transform、Physics body／state、Provider-private profile、Animation clockへwriteせず、selected ProviderだけがPortのresolved motionをwriteする。

### 4.1.1 Character Locomotion依存record registry

上記で参照するrole／requirement／dependency／policy／Save Replay／budget／fixture／invariantは、単なる文字列ではなく次のactive recordを厳密に一件ずつ持つ。MCD requirementは`McdContractRefV1`、それ以外は各typed refの`stable ID, version, content_hash`三点を必須とする。

| exact record | record type | 解決内容 |
|---|---|---|
| `role.game_system.character_locomotion.binding` v1 | `SemanticRoleRecordV1` | owner Feature、proposal validation、provider selection、Port batch publish。Transform／Physics writerではない |
| `requirement.feature.character_locomotion.bind_selected_executor` v1 | MCD `requirement` | accepted entry setを検証し、選択bindingへT40で一batchだけ送るmust |
| `requirement.non_responsibility.character_locomotion.no_transform_write` v1 | MCD `requirement` | Transform authoritative Stateをwriteしないmust_not |
| `requirement.non_responsibility.character_locomotion.no_physics_write` v1 | MCD `requirement` | Physics Body／State／native objectをwriteしないmust_not |
| `dependency.game_system.character_locomotion.navigation_motion_executor_port` v1 | `GameSystemDependencyEdgeV1` | target=Navigation Motion Executor Port、kind=`command_target`、transport=`type.navigation.motion_executor_intent_batch` v1、phase=T40、required、no inferred fallback |
| `implementation_policy.feature.character_locomotion.binding` v1 | `GameSystemImplementationPolicyV1` | Engine Definition default、Project replacement禁止、live switch禁止、全Targetで同じschema、unavailable=activation reject |
| `save_replay.feature.character_locomotion.binding` v1 | `SaveReplayContractV1` | selection binding ref／hash、movement profile ref／hash、generation lineageを保存。batch payloadとruntime handleは保存せずReplayへaccepted proposal ID列とbatch hashを記録 |
| `budget.feature.character_locomotion.binding.per_target` v1 | `BehaviorBudgetRecordV1` | actor当たりproposal 16、T40一batch、bounded queue contribution、overflow=typed reject |
| `fixture.feature.character_locomotion.motion_executor` v1 | `FixtureRecordV1` | production Physics＋fixture board／RTS、root motionあり／なし、type spoof／hash／duplicate／order／stale／failure oracle |
| `invariant.character_locomotion.accepted_intent_subset` v1 | `InvariantRecordV1` | entries schema set ⊆ selected Provider accepted set |
| `invariant.character_locomotion.single_selected_executor` v1 | `InvariantRecordV1` | actor／generation／tickごとにexactly one selected binding |
| `invariant.character_locomotion.no_transform_or_physics_write` v1 | `InvariantRecordV1` | binding write setはselection stateだけ |

非MCD recordの共通headerは`record_id`、`record_version=1`、`record_content_hash`、owner ref／hash、`status=active`、introduced Contract set hashを持つ。各record hashはhash Fieldを除くheaderと次のtyped payloadをMCD canonical encodeして計算する。

```text
SemanticRoleRecordV1.payload
  accepted_input_type_refs: exact 3 McdContractRefV1
  emitted_port_message_type_ref: exact Navigation batch McdContractRefV1
  allowed_phase_ids: [T40_MotionIntent]
  authoritative_write_type_refs: [type.feature.character_locomotion.motion_executor_selection_state]
  forbidden_write_owner_refs: [owner.core.transform, owner.engine.physics]

GameSystemDependencyEdgeV1.payload
  target_contract_ref: exact Navigation MotionExecutorPortV1 ref/revision/hash
  edge_kind: command_target
  transport_type_ref: exact type.navigation.motion_executor_intent_batch v1
  phase_relation: same_phase_t40_after_proposal_seal
  delivery: exactly_once_per_actor_generation_tick
  required: true
  fallback: no_fallback

GameSystemImplementationPolicyV1.payload
  allowed_implementation_kinds: [engine_definition]
  default_implementation_ref: exact binding implementation ref/hash
  native_eligibility: false
  replacement_policy: sealed
  live_switch_policy: forbidden
  equivalence_fixture_ref: exact motion executor fixture ref/hash
  required_target_refs: all active Project Targets
  configuration_schema_ref: exact empty configuration type ref
  unavailable_behavior: reject_activation_keep_last_valid

SaveReplayContractV1.payload
  system_ref: exact game_system ref/version/Contract set hash
  owned_state_type_refs: [type.feature.character_locomotion.motion_executor_selection_state v1]
  saved_fields: [provider_binding_ref, provider_binding_hash, executor_capability_ref,
                 movement_profile_ref, movement_profile_hash, binding_generation]
  derived_fields: []
  recorded_commands: exact 3 proposal type refs
  recorded_port_messages: [type.navigation.motion_executor_intent_batch v1]
  checkpoint: T100_ReplayCheckpoint
  unsupported_version_behavior: typed_reject_keep_backup
  state_hash_policy: canonical_owned_fields

BehaviorBudgetRecordV1.payload
  max_proposals_per_actor_tick: 16
  max_batches_per_actor_generation_tick: 1
  max_inline_payload_bytes_per_entry: 65536
  overflow_behavior: typed_reject_no_partial_batch
  target_budget_refs: exact set per active Target
```

三MCD requirementは次の全Fieldを持つ。

| ID | normative／priority | statement | scope／verification／acceptance | failure code |
|---|---|---|---|---|
| `requirement.feature.character_locomotion.bind_selected_executor` | must／blocking | accepted proposalを検証しactor／generation／tickごとに一つのselected bindingへ一batchだけ配送する | binding System／T40、contract＋runtime conformance、exact batch count=1 and accepted set subset | `MIRAKAN-LOCOMOTION-BINDING-INVALID` |
| `requirement.non_responsibility.character_locomotion.no_transform_write` | must_not／blocking | binding SystemはTransform owner Stateを書かない | System write set、static＋runtime conformance、Transform write edge=0 | `MIRAKAN-LOCOMOTION-BINDING-AUTHORITY_VIOLATION` |
| `requirement.non_responsibility.character_locomotion.no_physics_write` | must_not／blocking | binding SystemはPhysics State／native objectを書かない | System write set、static＋runtime conformance、Physics write edge=0 | `MIRAKAN-LOCOMOTION-BINDING-AUTHORITY_VIOLATION` |

各requirementはsource ref=`mirakan.arch.pack-gameplay-features#41-character-locomotion-binding-system`、introduced_by=本Feature contract ChangeSet、owner、version、Contract set hashを持つ。Fixture record payloadはproduction Physics、fixture board／RTS、root motionあり／なし、type spoof、payload ref／value schema、hash、duplicate、order、stale、provider failureのinput／oracle／不変state hashを列挙する。三Invariant recordは上表の述語、評価phase、input refs、expected Boolean、failure Diagnosticを持ち、自由式を保存しない。

RegistryはID、version順にsortし、duplicate、missing owner、version／hash mismatch、deprecated recordをrejectする。`validator.feature.character_locomotion.v1`はMCD Envelope、2 Feature-owned type、Navigation transport、全12 record、Provider Catalog binding、Save／Replay、budget、fixture、invariantのexact解決を検査し、bare persistent ID、未登録System／type、undeclared proposal schema、Transform／Physics write、manifest／System Catalog／fixtureの解決漏れを拒否する。

### 4.2 Runtime data flow

```text
Input Snapshot
  -> Input／AI intent evaluation
  -> RequestFireCommandV1
  -> Weapon System
       -> hitscan: CollisionQueryRequestV1
       -> projectile: SpawnRangedProjectileCommandV1
       -> WeaponFireAcceptedEventV1
  -> Collision query／normalization
  -> CollisionQueryResultV1
  -> ShotHitEventV1
  -> ApplyDamageCommandV1
  -> Combat System
  -> ApplyVitalDeltaCommandV1
  -> Vital System
  -> DamageAppliedEventV1／DefeatEventV1
       -> Score／Encounter／consumer Genre flow
       -> HUD／Audio／VFX／Camera Presentation
  -> Replay checkpoint
  -> immutable Snapshot publish
```

正確な実行phase、message merge、publish時点は[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)を参照する。Projectile spawnはWeaponのFire transactionでcapacityを予約し、Ranged Projectile Systemへ次の有効なactivation境界で渡す。muzzle Audio／VFXはPresentationとして開始できるが、その位置をauthoritative Projectile位置として読み戻さない。

hitscanはWeapon評価でqueryを生成し、Collision ownerの実行とpublishを経てDamageへ接続する。同期Physics objectへ直接raycastするProject callbackを公開しない。

Pickup overlapはCollisionの正規化EvidenceをPickup Systemが消費し、3.14節の順序と`GrantRequestPortV1` transactionに従う。provider Stateへの直接write、grant型名によるowner推論、未登録providerへの暗黙fallbackを許可しない。

### 4.3 Fire transaction

一つの`RequestFireCommandV1`は次の順で原子的に評価する。

1. owner、Weapon instance、Definition revision、active slot、primary／secondary Binding roleを検証する。
2. operation eligibility policy、Vital、Weapon enabled、reload、trigger、cadenceを検証する。
3. ammo cost、Pattern shot count、Projectile spawn数、Collision query数を計算する。
4. Projectile pool、Collision query capacity、authoritative Command／Event queueの必要capacityを事前予約する。
5. すべて成功した場合だけammo、cadence state、reload stateを更新する。
6. hitscan queryまたは次のProjectile activation用Commandを全件生成する。
7. `WeaponFireAcceptedEventV1`を一件生成する。

Pattern 64 Shotのうち32 Shotだけを生成するpartial fire、ammoだけを消費してDeliveryを生成しないfire、capacity不足時にcooldownだけを開始するfireを禁止する。

Presentation cue capacityはFire成立条件へ含めない。`WeaponFireAcceptedEventV1`の購読側がcritical cue priorityに従って別途予約し、不足してもauthoritative Fire結果をrollbackしない。

cooldown、reload中、ammo不足、operation eligibility policy denialは通常の`FireRejectedReasonV1`でありSession faultではない。Schema破損、State owner conflict、予約後のcapacity invariant違反はfaultである。

## 5. Command、Event、Snapshot

### 5.1 Command

```text
RequestFireCommandV1
ReleaseFireCommandV1
RequestReloadCommandV1
RequestWeaponSwitchCommandV1
SpawnRangedProjectileCommandV1
ApplyDamageCommandV1
ApplyVitalDeltaCommandV1
RequestPickupCollectionCommandV1
GrantRequestV1
AdvanceEncounterCommandV1
```

各Commandはproducer、target owner、consume phase、combine policy、conflict key、capacity contributionをMCDへ宣言する。

### 5.2 Event

```text
WeaponFireAcceptedEventV1
WeaponFireRejectedEventV1
WeaponReloadStartedEventV1
WeaponReloadCompletedEventV1
WeaponSwitchedEventV1
GrantResultV1
PickupCollectedEventV1
ShotHitEventV1
ProjectileExpiredEventV1
DamageAppliedEventV1
DamageBlockedEventV1
DefeatEventV1
ScoreChangedEventV1
ComboChangedEventV1
WaveStartedEventV1
WaveCompletedEventV1
BossPhaseChangedEventV1
```

Eventは原因となるCommand ID、tick、producer System、owner／target Stable ID、Definition revision、Evidence refを持つ。Display text、localized string、VFX Asset pathをauthoritative Eventへ入れない。

### 5.3 Snapshot

- `WeaponSnapshotV1`: active slot、ammo、reload、next activation、enabled
- `RangedProjectileSnapshotV1`: count、canonical projectile records、capacity
- `VitalSnapshotV1`: Health、Shield、invulnerability、life state
- `ScoreSnapshotV1`: score、combo、multiplier、high score
- `PickupSnapshotV1`: available／pending／collected、transaction、collector、collection tick
- `EncounterSnapshotV1`: encounter、phase、wave、remaining authoritative actor

HUD、AI behavior、Debuggingは必要なbounded Snapshotだけを読む。Render、Audio、VFXはPresentation projectionを使い、authoritative ownerを直接queryしない。

## 6. Save、Replay、Migration

### 6.1 Save対象

- active Weapon loadout、Weapon instance、slot、ammo、cadence、reload
- active authoritative Projectileのcanonical record
- Vital State
- Pickup available／pending／collected stateとgrant transaction
- Score／Combo／persistent high score
- Encounter phase、Wave、Spawn ordinal、RNG stream state
- active Definition、System Graph、Implementation Set、Profile hash

SaveはProjectileを`projectile_spawn_id`順、WeaponをStable ID順、Entity StateをStable ID順に並べる。Pool slot、Runtime handle、Physics native ID、VFX instance、Audio Voice、Camera shakeを保存しない。

### 6.2 Replay

ReplayはInputSnapshot、accepted external result、Definition／Profile hash、RNG stream state、Command／Event oracle、periodic authoritative state hashを記録する。

最低限次のcausal chainをQueryできる。

```text
Input transition
  -> RequestFire
  -> Fire accepted／rejected
  -> Query／Projectile
  -> Shot hit
  -> Damage applied／blocked
  -> Defeat
  -> Score／Encounter
```

Presentation Event欠落をauthoritative divergenceにしない。Damage、Projectile、Scoreの差異をPresentation-onlyとして無視しない。

### 6.3 Migration

- Field IDをrenameで変更しない。
- enum意味変更は新Type versionを作る。
- cadence algorithm変更は新しい`cycle_distribution_fixture_ref`とReplay migrationを必要とする。既存fixture IDの意味を上書きしない。
- Projectile Stateを保存対象から外す変更はMajor migrationである。
- Weapon Definition更新時、live instanceのammoを無言でrefill／truncateしない。Migration Policyを明示する。

## 7. FailureとDiagnostic

### 7.1 通常結果

```text
FireRejectedDisabled
FireRejectedEligibilityPolicy
FireRejectedDefeated
FireRejectedCadence
FireRejectedReloading
FireRejectedAmmo
ReloadRejectedFull
ReloadRejectedNoReserve
WeaponSwitchRejectedUnavailable
PickupCollectionRejectedUnavailable
PickupCollectionRejectedFilter
GrantRejectedNoCapacity
DamageBlockedTeamPolicy
DamageBlockedInvulnerability
ShotNoHit
ProjectileExpired
```

これらは想定可能なGameplay結果であり、Console errorまたはSession faultにしない。必要なPresentation cueとDebug Eventを持てる。

### 7.2 Diagnostic ID

```text
MIRAKAN-FEATURE-DEFINITION_INVALID
MIRAKAN-FEATURE-CONTRACT_VERSION_MISMATCH
MIRAKAN-FEATURE-CAPABILITY_UNAVAILABLE
MIRAKAN-FEATURE-STATE_OWNER_CONFLICT
MIRAKAN-RANGED-COMBAT-FIRE_TRANSACTION_FAILED
MIRAKAN-RANGED-COMBAT-PROJECTILE_CAPACITY_EXCEEDED
MIRAKAN-RANGED-COMBAT-QUERY_CAPACITY_EXCEEDED
MIRAKAN-FEATURE-AUTHORITATIVE_QUEUE_OVERFLOW
MIRAKAN-COMBAT-DAMAGE_TARGET_INVALID
MIRAKAN-PICKUP-GRANT-FAILED
MIRAKAN-SCORING-OVERFLOW
MIRAKAN-FEATURE-SAVE_CONTRACT_MISMATCH
MIRAKAN-FEATURE-REPLAY_DIVERGENCE
MIRAKAN-FEATURE-PRESENTATION_AUTHORITY_VIOLATION
```

DiagnosticはRequirement ID、Definition／Field path、System、tick／phase、actual、expected、Capability、Target、State owner、Evidence ID、修正候補を持つ。

`MIRAKAN-FEATURE-PRESENTATION_AUTHORITY_VIOLATION`はRender、Audio、VFX、Camera、UI、presentation-only EventまたはfixtureからFeature-owned authoritative State／Command結果へwriteまたは逆入力した時の共通Diagnosticである。Build／conformanceを失敗させ、authoritative State、last-valid Source、Feature receiptを維持する。

### 7.3 Failure policy

| Failure | 結果 |
|---|---|
| Definition／Contract不正 | ChangeSet／Cookを拒否し、last-valid Artifactを維持 |
| State owner重複 | System Bundleをactivateしない |
| Fire capacity不足 | Fire transaction全体を拒否し、authoritative State不変 |
| Query result overflow | partial hitを使わずtick fault |
| Damage target stale | Damageを適用せずtyped stale resultを記録 |
| Pickup Grant通常拒否 | Pickupをavailableへ戻し、authoritative grantを適用しない |
| Pickup Grant queue／invariant失敗 | pending transactionを保持してSession fault。dropしてcollected扱いにしない |
| Score overflow | saturate／wrapせずSession fault |
| Save mismatch | Saveを開かずbackupを維持 |
| Replay divergence | first divergenceで停止しReproduction Bundleを生成可能にする |
| Presentation failure | Gameplay継続。critical cue Gateは別途失敗 |
| AIの未対応Capability要求 | Blocking gapとして質問または対応Profileを提示 |

## 8. TestとQualification

### 8.1 Contract／Schema

- 全Typeのvalid境界、unknown field、duplicate ID、missing ref、cycle
- Stable ID／Contract Ref／Artifact Refの混同拒否
- Fire Mode、Pattern、Delivery、Ammo、Reloadの組合せmatrix
- current Profileへunknown／out-of-contract enumまたはFieldを混入したnegative fixture
- State owner、Command phase、Event delivery、Save closure
- unknown Diagnostic、unbounded collection、NaN／Inf、overflow

### 8.2 Fire／Weapon

- single、automatic、1／2／32 burst
- cadence 1／7／10／60 activation per 60 tickを1,000 cycle実行
- press、hold、release、reload、switch、pause、defeatの境界
- primary／secondary両trigger同時holdの同tick arbitration(primaryのActivation、secondary側`FireRejectedCadence`と繰延)
- ammo 0／1／capacity、reserve 0／1／capacity
- Pattern 1／63／64 Shot、hitscan／Projectileそれぞれのcapacity丁度／capacity+1
- capacity失敗時にammo、cadence、reload、Eventが不変

### 8.3 Shot／Collision

- 2D／3D hitscan `closest_per_shot`、multi-shot時のcanonical ordering
- straight projectile、thin target、owner ignore、lifetime境界
- same fraction hitのcanonical ordering
- team relation matrix、sensor／solid、initial overlap
- Particle／depth／Camera occlusionをHitへ使う依存を拒否
- hitscanとprojectileが同じDamage Specで同じTarget結果を得るfixture

### 8.4 Damage／Vital／Score

- Health、Shield、invulnerability、exact zero、overkill、defeat一回
- self／ally／neutral／hostile matrix
- friendly fire変更のHigh Impact質問
- direct／radial Damageの境界
- ammo／health／shield／score／weapon Pickup、同tick競合、Grant失敗rollback
- Defeat credit、同一Defeat重複加点0
- combo開始／継続／expire、multiplier、high score Save／Load
- Score int64 overflow fault

### 8.5 Encounter Spawn

- Wave 1／256、Spawn group 1／1,024、Boss phase 0／32
- enemy全滅、goal、time、Boss defeatのcompletion evidence
- pause／eligibility policy中のEncounter clockがconsumer policyどおり停止
- Save／Load後に同じWave、RNG、Projectile、Score結果

### 8.6 AI Eval

- 「遠隔攻撃」「弾」「ビーム」「連射」「三方向」「弾幕」「強く」「軽く」のcanonical resolution
- authoritative projectileとPresentation particleの混同0
- hitscan／projectile、ammo、friendly fire、scoreのBlocking不足見逃し0
- Low Impact既定を不要質問する率5%以下
- 対象外Capabilityをcurrent contractの成功として返す件数0
- AI／手動Editor／Project C++ Commandが同じDefinition hashとRuntime結果へ収束
- ExplainがField、理由、Assumption、代替、capacity impact、Testを返す
- Genre catalog、Profile名、product fixture名からFeature依存を推論する件数0

### 8.7 Feature contract fixture

| Fixture | 検証対象 |
|---|---|
| `fixture.feature.combat.contract` | Team／Damage／Vital、owner、Save／Replay、capacity exact／exact+1 |
| `fixture.feature.ranged_combat.contract` | Weapon／Shot／Projectile、Fire transaction、Collision port |
| `fixture.feature.encounter_spawn.contract` | phase／wave／spawn ordering、completion evidence |
| `fixture.feature.scoring.contract` | Score／Combo／overflow／persistent state |
| `fixture.feature.pickup_grant.provider_neutral` | provider 0件、registered provider、unsupported型、rollback |
| `fixture.feature.interaction.contract` | Interaction ownerのexact public port |
| `fixture.feature.character_locomotion.motion_executor` | Navigation-owned PortのPhysicsなし2D／3D stub、optional Physics reference Provider、stale generation |
| `fixture.feature.path_following.executor_stub` | Navigation ownerのpath execution port、board-token／RTS stub、missing／incompatible Provider |
| `fixture.feature.scenario_stage.none` | `completion_mode=none`でcompletion ownerを要求しないStage |

各fixtureはPack単体または宣言済みFeature closureだけでinstall／validate／executeできなければならない。Genre Pack、Genre Profile、product fixture、Genre固有Action roleをtest dependencyへ含めない。manual authoring、AI生成、manual再編集、AI再編集は同じSourceとFeature operationを使い、同じDefinition hash、Receipt、Runtime結果へ収束する。

Character Locomotion fixtureはPhysics capabilityとPhysics Packが不在でもPack installを成功させる。Physics Character Motorを選択したC1 reference caseだけがそのProvider qualification receiptを要求し、Provider failure時にlast-valid resolved motionとFeature Stateが不変であることを検証する。

### 8.8 Performance／Soak

- `fixture.feature.ranged_combat.contract`へ2D／3D Collision stubとscale input exact／exact+1を与える
- `fixture.feature.encounter_spawn.contract`と`fixture.feature.pickup_grant.provider_neutral`を組み合わせた10分soak、spawn／destroy churn、Save／Load
- CPU／memory／queue／pool high-water、P95／P99／P99.9
- authoritative Fire／Projectile／Hit／Damage／Score drop 0
- Replay hash一致、Presentation degradation時もGameplay一致
- Project固有Scaleが組込みfixtureを超える場合の再Qualification

## 9. Feature authoring operation

Feature mutationはFeature ownerのoperationだけが行う。Genre operationはcomposition、Profile、Game Flow、Action roleと統合fixture bindingだけを変更し、Feature Sourceを直接writeしない。

```text
operation.feature.create_definition
operation.feature.update_definition
operation.feature.configure_system
operation.feature.bind_runtime_port
operation.feature.preview_change
operation.feature.explain_contract
operation.feature.validate_contract
```

全mutationは対象`feature.*` pack ID、base revision、changed fields、validator、fixture、rollback tokenを含むChangeSet Receiptを返す。Feature contract qualificationは本節のoperationと8節のFeature fixtureだけが所有する。

## 10. 公式資料

- [Unity Manual: ScriptableObject](https://docs.unity3d.com/Manual/class-ScriptableObject.html)
- [Unity Input System](https://github.com/Unity-Technologies/InputSystem)
- [Unity: Three ways to architect game code with ScriptableObjects](https://unity.com/how-to/architect-game-code-scriptable-objects)
- [Unity Scripting API: ObjectPool](https://docs.unity3d.com/ScriptReference/Pool.ObjectPool_1.html)
- [Unreal Engine Gameplay Framework Quick Reference](https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-framework-quick-reference-in-unreal-engine)
- [Unreal Engine: Abilities in Lyra](https://dev.epicgames.com/documentation/unreal-engine/abilities-in-lyra-in-unreal-engine)
- [Unreal Engine: Lyra Inventory and Equipment](https://dev.epicgames.com/documentation/en-us/unreal-engine/lyra-inventory-and-equipment-in-unreal-engine)
- [Godot: Nodes and Scenes](https://docs.godotengine.org/en/stable/getting_started/step_by_step/nodes_and_scenes.html)
- [Godot: Resources](https://docs.godotengine.org/en/stable/tutorials/scripting/resources.html)
- [Godot: Using signals](https://docs.godotengine.org/en/stable/getting_started/step_by_step/signals.html)

外部資料は責務分離とAuthoring先例の確認に使用する。MiraikanaiのFeature Type、State owner、algorithm、failure、Save、Replayは本規約が決定する。Genre固有AI Operation、Profile、Game Flow、Action role、統合fixtureは各Genre ownerを参照し、本書から特定Genreへ依存しない。共有phase、Risk、budgetは各canonical ownerを参照し、外部EngineのAPI互換性をProduct要件にしない。
