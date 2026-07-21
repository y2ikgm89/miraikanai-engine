# Miraikanai Engine Shooter Reference Pack

- 文書ID: mirakan.arch.domain-pack-shooter
- 状態: review
- 正本範囲: Shooter PackのWeapon／Shot／Projectile／Damage／Vital／Team／Pickup／Score／Encounter／Game Flow契約、Profile composition、Shooter intent／operation、domain algorithm／failure／fixture
- 非正本範囲: Domain Pack lifecycle、共有MCD／Game System envelope、Input／Collision／Physics／Camera／UI／Audio／Asset／VFX schema、Runtime phase／capacity、Product roadmapは各依存先を参照
- 依存: [Product Plan](../00-product/product-plan.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)、[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[Debugging／Replay](../04-runtime/debugging-observability-replay.md)、[Collision](../05-simulation/collision.md)、[Physics](../05-simulation/physics.md)、[Camera](../06-rendering/camera.md)、[VFX Runtime](../06-rendering/vfx-runtime.md)、[Input](../07-platform/input.md)、[Audio](../07-platform/audio.md)、[Game UI](../07-platform/ui-text-localization-accessibility.md)、[Domain Pack Contract](domain-pack-contract.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論

Miraikanai EngineのShooter機能は、巨大な`ShooterManager`、Genre別Class hierarchy、Particleを兼用する弾、任意Script callbackとして実装しない。共通の`mirakan.feature.shooter_core.c1`を、独立したWeapon、Shot Delivery、Projectile、Combat、Vital、Score、Encounter、Game Flowの型付き契約として提供し、2D top-downと3D TPSは同じ契約へProfileを適用する。

C1のreference compositionは、`mirakan.domain.2d_action.c1`へ`shooter.profile.2d_top_down.c1`をcomposeしたsingle-player 2D top-down shooterと、同じShooter Coreへ`shooter.profile.tps_single_player.c1`を適用したsingle-player TPSである。2Dと3DでWeapon、Damage、Team、Score、Save、Replayの意味をforkしない。

人間とAIが編集するのは、version付きDefinition、Semantic Catalog、Requirement、Profileである。実行時はC++23のEngine Standard Game Systemまたは同じPublic Contractへ適合するProject実装が、Runtime ownerのscheduling、bounded queue、単一State ownerに従って評価する。

本規約の中心不変条件は次である。

1. authoritative Projectile、hitscan、Damage、ScoreとPresentation用Particle／Audio／Cameraを分離する。
2. Fireは、発射可否、弾薬、cadence、capacity、出力Commandを一つの原子的処理として解決する。
3. Stateは型ごとに厳密に一つのGame Systemだけが所有し、他SystemはCommand、Event、Snapshotだけで接続する。
4. Target性能に合わせてProjectile数、Damage、敵数、Pattern、Score条件を黙って変更しない。
5. AIは「銃」「弾」「連射」「強くする」等を文字列類似で即決せず、意味候補、Assumption、Capability、capacity impact、Testへ解決する。
6. Save／Replay／DebuggingはWeapon、Projectile、Vital、Score、Encounterのauthoritative状態をStable IDとContract versionで再現する。

## 2. 決定権と境界

| 主題 | 決定権 |
|---|---|
| Shooter語彙、Weapon／Shot／Projectile／Damage／Vital／Score／EncounterのPublic Contract | 本書 |
| Shooter Intent Resolver、AI質問、Operation、説明、Shooter Eval | 本書 |
| Shooter Core reference schema境界、2D／TPS Profile composition、Shooter fixture | 本書 |
| Game System envelope、State owner宣言、dependency graph、System Bundle | [Gameplay Programming Model](../03-authoring/gameplay-programming-model.md) |
| GameplayDefinition／Project C++の選択、Cook、安全制約 | [Gameplay Programming Model](../03-authoring/gameplay-programming-model.md) |
| tick、phase、Command／Event順、queue、memory、scale、fault | [Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)と[Performance／Capacity](../04-runtime/performance-capacity.md) |
| Ray／Shape query、swept projectile、Collision Filter、Hit normalization | [Collision](../05-simulation/collision.md) |
| Device、Action、Binding、Remap、Input Snapshot | [Input](../07-platform/input.md) |
| Camera aim／rig／recoil／shake | [Camera](../06-rendering/camera.md) |
| Character motorとmovement | [Physics](../05-simulation/physics.md) |
| HUD、reticle、score、ammo表示、accessibility | [Game UI](../07-platform/ui-text-localization-accessibility.md) |
| muzzle／trail／impact／explosion表現 | [VFX Runtime](../06-rendering/vfx-runtime.md) |
| shot／impact／reload／music | [Audio](../07-platform/audio.md) |
| Asset import、Cook、package | [Asset Lifecycle](../03-authoring/asset-lifecycle.md) |
| Pack適用、更新、競合、Template lifecycle | [Domain Pack Contract](domain-pack-contract.md) |
| Debug Session、Replay／Rewind、Causality、AI Diagnosis | [Debugging／Replay](../04-runtime/debugging-observability-replay.md) |

本書はRenderer、Physics Backend、Input Device、UI Widget、Audio Voiceを再定義しない。Shooter Game Systemは各SubsystemのPublic Portだけを消費し、Vendor object、Render object、native callback、Platform key codeを所有しない。

## 3. Activation境界

本文書の型、closed value、algorithm、failure、fixtureは`mirakan.feature.shooter_core.c1`のreference contractである。実際のCapability成熟度、製品Phase、将来機能は[Product Plan](../00-product/product-plan.md)だけが所有する。未有効の機能をunknown enum、optional field、placeholder BackendとしてC1へ流し込まない。

single-player Shooterはnetwork replication、prediction、server authority、matchmakingを含まない。2D／TPS Profileは同じWeapon、Damage、Team、Score、Save、Replay意味を使用し、viewまたはTargetの違いでPublic Contractをforkしない。

## 4. 正規用語

| 用語 | 定義 |
|---|---|
| `Shooter Core` | 2D／3Dに共通するWeapon、Shot、Damage、Vital、Score、Encounter契約集合 |
| `Weapon Definition` | 調整可能な発射方式、Delivery、Damage、Ammo、Cueへの不変参照 |
| `Weapon Instance State` | 装備Entityごとのammo、cadence、reload、active slot等のauthoritative状態 |
| `Shot Activation` | 一回のcadence許可で生成されるPattern全体。hitscan rayまたはProjectile一個と同義ではない |
| `Shot Delivery` | `hitscan`または`projectile`としてHit候補を得る方式 |
| `Authoritative Projectile` | Collision、Damage、Save、Replayへ影響するShooter System所有record |
| `Presentation Projectile` | trail、sprite、mesh、particle等の見た目。Gameplayへ逆入力しない |
| `Damage Request` | Hit EvidenceとDamage Specを結ぶCombat Systemへのtyped Command |
| `Vital State` | Health、Shield、invulnerability、defeatを持つauthoritative State |
| `Shot Pattern` | 一つのShot Activationから生成する方向、offset、speed倍率の有限集合 |
| `Encounter Pattern` | Wave、Spawn、Boss phase、completion条件を持つ有限State Machine |
| `Shooter Profile` | Dimension、Camera、Input、Capacity、既定Definition、Fixtureを束ねるProfile |

「Bullet」はauthoritative projectile、hitscan trace、Presentation particleのいずれかを自動では意味しない。「Gun」はWeapon Definitionの表示名であって、Delivery方式を決めない。

## 5. PackとProfile構成

```text
mirakan.feature.shooter_core.c1
  ├─ shooter.profile.2d_top_down.c1
  │    └─ compose mirakan.domain.2d_action.c1
  └─ shooter.profile.tps_single_player.c1
       └─ compose mirakan.domain.tps_single_player.c1
```

Feature PackはPublic Contract、Schema、Reference Definition、Validator、AI vocabulary、Test Fixtureを所有する。Profileは既定値、必須Capability、scale fixture reference、Input template、Scene／UI／Audio／VFX templateだけを所有し、Public Contractをforkしない。

### 5.1 `shooter.profile.2d_top_down.c1`

- orthographic／pixel-safe Camera
- `move` axis2、`aim` axis2／pointer2
- primary fire、secondary fire optional、reload optional、pause
- straight projectileとhitscan
- fixed／fan／radial Pattern
- score、combo、wave、Boss phase
- controlled Character movement／aim-origin契約、chaser／strafer／turretのbounded enemy behavior Template
- keyboard／mouse、controller、touch
- 2D Sprite、Tilemap、2D Collision、Audio、VFX、HUD

### 5.2 `shooter.profile.tps_single_player.c1`

- third-person Camera、camera collision、reticle
- `move` axis2、`look` axis2、primary fire、aim mode、reload、switch weapon、pause
- hitscanとstraight projectile
- deterministic cone spread
- magazine／reserve ammo
- Health、Shield optional、Team、hit reaction
- simple perception／combat behavior、Encounter、Checkpoint、Result
- Character motor／aim-origin契約。AI behaviorは同じ`RequestFireCommandV1`を使いWeapon Stateを直接writeしない
- keyboard／mouse、controller

FPS viewは現在のShooter reference contractの対象外である。将来scope、成熟度、activation、schema／Profile追加の要否は[Product Plan](../00-product/product-plan.md)だけが決定する。

## 6. 正規Data Model

すべてのSource objectはUUIDv7 `StableId`、MCD Typeはexact `McdContractRefV1`、Derived Artifactは`ArtifactRefV1`を使う。表示名、Asset path、C++型名、配列indexをidentityにしない。

### 6.1 `WeaponDefinitionV1`

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

### 6.2 `FireModeDefinitionV1`

```text
FireModeDefinitionV1
  trigger_mode: single | automatic | burst
  activation_count_per_cycle: uint16
  cycle_ticks: uint16
  cycle_distribution: even_floor_v1 | explicit_offsets_v1
  activation_offsets_ticks[]: optional
  ammo_cost_per_activation: uint16
  release_policy: stop_unfired | complete_started_cycle
  fire_while_reloading: false
```

範囲は`activation_count_per_cycle=[1,3600]`、`cycle_ticks=[1,3600]`、`activation_count_per_cycle <= cycle_ticks`とする。`burst`だけはcountを`[1,32]`へ制限する。C1は一Weaponにつき一tick最大一Activationである。Patternが一Activationから複数Shotを生成する。

`even_floor_v1`はcycle内のActivation `i=[0,count-1]`を`floor(i * cycle_ticks / activation_count_per_cycle)` tickへ配置し、`activation_offsets_ticks`を持たない。`explicit_offsets_v1`は`activation_count_per_cycle`と同数のoffsetを`[0,cycle_ticks-1]`へstrictly increasingで持つ。cycle開始tickとDefinition revisionが同じならscheduleは同じである。「毎秒7発」は`7 activation / 60 tick`へexactに解決し、float timerを累積しない。

`single`はcount 1、offset 0で、trigger rising edgeごとに一cycleだけ評価する。`automatic`はtrigger hold中にcycleを反復する。`burst`はrising edgeで一つの有限cycleを開始し、通常は`explicit_offsets_v1`で`[0,5,10]`等を表す。release後の再押下なしに次cycleを開始しない。これによりautomatic cadence、burst内間隔、burst後cooldownを同じscheduleで一意に表す。

`release_policy`はcycle内の未発射Activationだけへ適用する。`single`／`automatic`は`stop_unfired`、`burst`はGame Briefで`stop_unfired`または`complete_started_cycle`を選ぶ。Weapon switch、reload開始、owner defeat、Game Flow停止は未発射Activationを常にcancelし、消費済みammoと既発射Shotだけを維持する。

### 6.3 `ShotPatternDefinitionV1`

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

### 6.4 `ShotDeliveryDefinitionV1`

```text
ShotDeliveryDefinitionV1
  delivery_kind: hitscan | projectile
  max_range_m
  collision_query_profile_ref
  projectile_definition_ref: optional
  max_hits_per_activation
  hit_selection: closest_per_shot
```

hitscanはCollision規約のRay／Shape Castを使う。ProjectileはShooter Projectile Systemのdomain-local recordを使い、C1では各tickの移動segmentをswept Queryとして検証する。Particle collision、depth buffer、render occlusionをHit Evidenceにしない。

`delivery_kind=hitscan`では`projectile_definition_ref`を持たず、Patternの`speed_scale_q16`を全要素1.0、`projectile`ではrefを必須とする。`max_hits_per_activation=[1,64]`かつ`shot_count`以下とし、各Shotは`closest_per_shot`で最大一Hitを生成する。penetration、同一Shotの複数Hit、`ordered_all`は現在のcontractの対象外として拒否する。これらの将来scopeまたはschema判断は[Product Plan](../00-product/product-plan.md)へ委譲する。Projectileは`lifetime_ticks`またはspawn originから`max_range_m`へ到達した早い方でexpireする。

### 6.5 `ProjectileDefinitionV1`

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
ShooterProjectileStateV1
  projectile_spawn_id: uint64
  definition_ref
  owner_entity_ref
  owner_team_ref
  position_m
  direction_unit
  speed_mps
  age_ticks
  rng_draw_count
  status: pending | active | hit | expired
```

`projectile_spawn_id`はPlay session内でShooter Projectile Systemが単調増加させ、0を使わない。Pool slot、pointer、Physics handleをSave／Replay identityにしない。

### 6.6 `AmmoPolicyV1`と`ReloadPolicyV1`

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

### 6.7 `WeaponInstanceStateV1`

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

### 6.8 `DamageSpecV1`

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

全数値はfinite binary32とし、`base_damage_points=[0,1.0e9]`である。`direct`は`radial_damage_ref`を持たず、`radial`は必須とする。negative DamageをHealingとして再解釈しない。Healingは別Command／Specを使う。

### 6.9 `RadialDamageDefinitionV1`

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

`0 <= inner_radius_m <= outer_radius_m <= 1,000`、`max_targets=[1,256]`とする。`none`はquery refを持たず、`collision_query`は必須とする。候補はdistance、Target Stable IDの順でcanonicalに選び、`max_targets`超過をpartial Damageで継続せず`ShooterQueryCapacityExceeded`としてtickをfaultする。`linear_q16`はinnerで1.0、outerで0.0のQ16倍率とし、Gameplay DamageをVFX radiusから取得しない。

C1のradial originはDeliveryが確定したcanonical Hit位置であり、HitがないActivationではradial Damageを生成しない。timer、remote detonation、persistent area Damageは現在のcontractの対象外であり、将来scopeまたはschema判断は[Product Plan](../00-product/product-plan.md)へ委譲する。

### 6.10 `TeamPolicyV1`

Relationは`self | ally | neutral | hostile`のclosed enumとする。Policyは各Relationへ`ignore | block_without_damage | apply_damage`を一つ指定する。C1既定はself／ally=`ignore`、neutral=`block_without_damage`、hostile=`apply_damage`である。

friendly fireの変更はGame BriefのHigh Impact項目であり、AIが武器単位の見た目や難易度要求から推測して変更しない。

### 6.11 `VitalStateV1`

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

### 6.12 `ScoreRuleDefinitionV1`と`ScoreStateV1`

```text
ScoreRuleDefinitionV1
  rule_id: StableId
  event_kind
  base_points: int64
  combo_increment
  combo_expire_ticks
  multiplier_curve_ref
  target_filter

ScoreStateV1
  participant_ref
  current_score: int64
  combo_count: uint32
  multiplier_q16
  combo_expire_tick
  session_high_score: int64
  persistent_high_score: int64
```

Score SystemだけがScore Stateを所有する。Score加算は`DamageApplied`ではなく、Ruleが指定した`Defeat`、pickup、objective、wave completion等の確定Eventから行う。同じDefeatをHit数だけ重複加点しない。

`int64` overflowをsaturateまたはwrapせず`ShooterScoreOverflow`としてPlay sessionをfaultする。persistent high scoreはSave generationへ含める。

### 6.13 `EncounterPatternDefinitionV1`

```text
EncounterPatternDefinitionV1
  encounter_id: StableId
  phases[1..64]
  waves[1..256]
  spawn_groups[1..1024]
  completion_condition
  boss_phase_refs[0..32]
  difficulty_profile_ref
  rng_stream_ids[]
```

Spawn順、Wave遷移、Boss phaseはfixed tick、Stable ID順、登録済みRNG streamで決める。VFX completion、Audio duration、Animationのpresentation-only Notifyを進行条件にしない。

### 6.14 `DifficultyProfileV1`

Difficultyは`easy`、`normal`、`hard`等の表示名だけでは成立しない。Profileは変更できるField ref、基準値、演算、上限、Gameplay fidelity floorを明示する。

C1で変更可能な軸は次に限定する。

- enemy max Health
- enemy base Damage
- Encounter spawn interval／group count
- Shot Pattern／cadenceの選択
- pickup量
- Score multiplier

Difficulty変更でCollision無効化、Hit Event drop、敵やProjectileの無断削除、Replay非決定化を行わない。

### 6.15 `PickupDefinitionV1`と`PickupInstanceStateV1`

```text
PickupDefinitionV1
  pickup_definition_id: StableId
  grant_kind: ammo | health | shield | score | weapon
  grant_amount: uint32 | binary32 | int64, optional by grant_kind
  weapon_definition_ref: optional
  collection_filter_ref
  presentation_cue_ref

PickupInstanceStateV1
  pickup_instance_id: StableId
  definition_ref
  state: available | pending_grant | collected
  grant_transaction_id: optional
  pending_collector_ref: optional
  collected_by_ref: optional
  collected_tick: optional
```

`grant_kind`をdiscriminatorとし、`ammo | health | shield | score`は`grant_amount>0`を必須としてweapon refを持たない。ammoはuint32、health／shieldはfinite binary32のpoints型、scoreはint64とする。`weapon`はweapon refを必須としamountを持たない。現在のcontractはone-shot collectionだけを扱い、respawn、random loot table、inventory weightは対象外として拒否する。これらの将来scopeまたはschema判断は[Product Plan](../00-product/product-plan.md)へ委譲する。Pickup Systemはtyped Grant Commandを各State ownerへ送り、Weapon ammo、Vital、Scoreを直接writeしない。

### 6.16 Shooter Game Flow

Shooter Game Flowのclosed stateは`title | settings | ready | playing | paused | result`とする。`restart`はstateではなく、Resultまたは停止済みsessionから新しいPlay sessionを開始するtyped transition actionである。Profileはstateを追加または省略せず、表示画面がないstateもauthoritative遷移として維持する。

Pause、Result、Restartがcadence、reload、Encounter、Save／Replayへ与える効果はGame Flow SystemがShooter Commandとして宣言し、UI visibilityまたはAudio／Animation completionから推測しない。

## 7. Game SystemとState owner

### 7.1 Engine Standard System

| System Contract | Scope | 所有State | 主な責務 |
|---|---|---|---|
| `game_system.engine.weapon` | `entity_instance` | `WeaponInstanceStateV1` | trigger、cadence、ammo、reload、switch、Fire transaction |
| `game_system.engine.shooter_projectile` | `world_instance` | `ShooterProjectileStateV1` collection | spawn、swept query、hit／expire、capacity |
| `game_system.engine.combat` | `world_instance` | Damage credit ledger | Hit Evidence、Team、Damage rule、creditの検証 |
| `game_system.engine.vital` | `entity_instance` | `VitalStateV1` | Health、Shield、invulnerability、defeat |
| `game_system.engine.score` | `play_session` | `ScoreStateV1` | 加点、combo、multiplier、high score |
| `game_system.engine.pickup` | `world_instance` | `PickupInstanceStateV1` collection | overlap Evidence、collection、typed Grant、one-shot state |
| `game_system.engine.encounter` | `encounter_instance` | Encounter runtime state | Wave、Spawn、Boss phase、completion |
| `game_system.engine.game_flow` | `play_session` | Game flow state | Ready、Playing、Paused、Result、Restart |

同じPublic Contractへ適合するProject-defined実装は許可するが、Engine Standardと同時にactiveにしない。WeaponとVitalをCharacter Systemのprivate Fieldへ隠さず、Public State owner tableへ出す。

### 7.2 Runtime data flow

```text
Input Snapshot
  -> Input／AI intent evaluation
  -> RequestFireCommandV1
  -> Weapon System
       -> hitscan: CollisionQueryRequestV1
       -> projectile: SpawnShooterProjectileCommandV1
       -> WeaponFireAcceptedEventV1
  -> Collision query／normalization
  -> CollisionQueryResultV1
  -> ShotHitEventV1
  -> ApplyDamageCommandV1
  -> Combat System
  -> ApplyVitalDeltaCommandV1
  -> Vital System
  -> DamageAppliedEventV1／DefeatEventV1
       -> Score／Encounter／Game Flow
       -> HUD／Audio／VFX／Camera Presentation
  -> Replay checkpoint
  -> immutable Snapshot publish
```

正確な実行phase、message merge、publish時点は[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)を参照する。Projectile spawnはWeaponのFire transactionでcapacityを予約し、Shooter Projectile Systemへ次の有効なactivation境界で渡す。muzzle Audio／VFXはPresentationとして開始できるが、その位置をauthoritative Projectile位置として読み戻さない。

hitscanはWeapon評価でqueryを生成し、Collision ownerの実行とpublishを経てDamageへ接続する。同期Physics objectへ直接raycastするProject callbackを公開しない。

Pickup overlapはCollisionの正規化EvidenceをPickup Systemが消費し、`pickup_instance_id, collector StableId`順に候補を決める。同じPickupへ同tickに複数collectorが接触した場合はcanonical先頭だけを`pending_grant`へ遷移し、transaction ID付きGrant Commandを対象State ownerへ送る。ownerがgrantを適用して`PickupGrantAcceptedEventV1`を返した時だけ`collected`へ遷移し、full ammo／Health等で`PickupGrantRejectedEventV1`を返した場合は`available`へ戻す。pending中の重複collectionを許可しない。

### 7.3 Fire transaction

一つの`RequestFireCommandV1`は次の順で原子的に評価する。

1. owner、Weapon instance、Definition revision、active slot、primary／secondary Binding roleを検証する。
2. Game Flow、Vital、Weapon enabled、reload、trigger、cadenceを検証する。
3. ammo cost、Pattern shot count、Projectile spawn数、Collision query数を計算する。
4. Projectile pool、Collision query capacity、authoritative Command／Event queueの必要capacityを事前予約する。
5. すべて成功した場合だけammo、cadence state、reload stateを更新する。
6. hitscan queryまたは次のProjectile activation用Commandを全件生成する。
7. `WeaponFireAcceptedEventV1`を一件生成する。

Pattern 64 Shotのうち32 Shotだけを生成するpartial fire、ammoだけを消費してDeliveryを生成しないfire、capacity不足時にcooldownだけを開始するfireを禁止する。

Presentation cue capacityはFire成立条件へ含めない。`WeaponFireAcceptedEventV1`の購読側がcritical cue priorityに従って別途予約し、不足してもauthoritative Fire結果をrollbackしない。

cooldown、reload中、ammo不足、Game Flow停止は通常の`FireRejectedReasonV1`でありSession faultではない。Schema破損、State owner conflict、予約後のcapacity invariant違反はfaultである。

## 8. Command、Event、Snapshot

### 8.1 Command

```text
RequestFireCommandV1
ReleaseFireCommandV1
RequestReloadCommandV1
RequestWeaponSwitchCommandV1
SpawnShooterProjectileCommandV1
ApplyDamageCommandV1
ApplyVitalDeltaCommandV1
RequestPickupCollectionCommandV1
GrantAmmoCommandV1
GrantWeaponCommandV1
AwardScoreCommandV1
AdvanceEncounterCommandV1
SetShooterGameFlowCommandV1
```

各Commandはproducer、target owner、consume phase、combine policy、conflict key、capacity contributionをMCDへ宣言する。

### 8.2 Event

```text
WeaponFireAcceptedEventV1
WeaponFireRejectedEventV1
WeaponReloadStartedEventV1
WeaponReloadCompletedEventV1
WeaponSwitchedEventV1
PickupGrantAcceptedEventV1
PickupGrantRejectedEventV1
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
ShooterResultReachedEventV1
```

Eventは原因となるCommand ID、tick、producer System、owner／target Stable ID、Definition revision、Evidence refを持つ。Display text、localized string、VFX Asset pathをauthoritative Eventへ入れない。

### 8.3 Snapshot

- `WeaponSnapshotV1`: active slot、ammo、reload、next activation、enabled
- `ShooterProjectileSnapshotV1`: count、canonical projectile records、capacity
- `VitalSnapshotV1`: Health、Shield、invulnerability、life state
- `ScoreSnapshotV1`: score、combo、multiplier、high score
- `PickupSnapshotV1`: available／pending／collected、transaction、collector、collection tick
- `EncounterSnapshotV1`: encounter、phase、wave、remaining authoritative actor
- `ShooterGameFlowSnapshotV1`: state、transition reason、result

HUD、AI behavior、Debuggingは必要なbounded Snapshotだけを読む。Render、Audio、VFXはPresentation projectionを使い、authoritative ownerを直接queryしない。

## 9. Input Action Template

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

## 10. PresentationとGameplay分離

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

## 11. AI Semantic Contract

### 11.1 `ShooterIntentResolutionV1`

```text
ShooterIntentResolutionV1
  requirement_refs[]
  source_terms[]
  canonical_concept_refs[]
  selected_feature_pack_ref
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

### 11.2 必須質問

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

### 11.3 禁止する推測

- 「弾」をParticleだけで実装する。
- 「銃」をhitscanと決める。
- 「連射」をautomatic、burst、Pattern同時発射のどれかへ無言で決める。
- 「強くする」をDamage、cadence、range、spread、ammo、feedbackの全部へ適用する。
- 「弾幕」をVFX particle countだけ増やして完成とする。
- 「軽くする」をProjectile、敵、Damage、Hit判定の削減へ変換する。
- 「オンライン風」「対戦風」からnetworkingを有効にする。
- Camera shake、recoil、hit flashをGameplay aim／Damageへ使う。

### 11.4 AI Operation

| Operation | Risk | 結果 |
|---|---:|---|
| `operation.shooter.search_catalog` | R0 | Concept、Capability、Definition、Fixture候補 |
| `operation.shooter.resolve_intent` | R0 | `ShooterIntentResolutionV1` |
| `operation.shooter.explain_resolution` | R0 | Field、理由、Assumption、代替、Evidence |
| `operation.shooter.preview_simulation` | R0 | Headless／visual Preview、Cost、Event trace |
| `operation.shooter.create_weapon` | R1 | `WeaponDefinitionV1` ChangeSet |
| `operation.shooter.set_fire_mode` | R1 | Fire Mode差分 |
| `operation.shooter.set_shot_delivery` | R1 | hitscan／Projectile差分 |
| `operation.shooter.create_shot_pattern` | R1 | Pattern差分とcapacity見積り |
| `operation.shooter.set_ammo_reload` | R1 | ammo／reload差分とHUD／Save closure |
| `operation.shooter.set_damage_policy` | R2 | Damage／Team／Vital差分 |
| `operation.shooter.create_pickup` | R1 | typed Grantとcollection差分 |
| `operation.shooter.create_encounter` | R2 | Wave／Spawn／Boss phase差分 |
| `operation.shooter.set_score_rule` | R1 | Score／Combo差分 |
| `operation.shooter.configure_game_flow` | R1 | Title／Ready／Pause／Result／Restart差分 |
| `operation.shooter.configure_difficulty` | R2 | 許可軸、fidelity floor、capacity差分 |
| `operation.shooter.apply_profile` | R2 | Pack／Input／UI／Asset closure |
| `operation.shooter.propose_native_variant` | R3 | Native Source、Test、capacity evidence、Promotion案 |

AI ProviderへC++ pointer、Runtime handle、unbounded projectile list、raw Physics objectを渡さない。Search／Readは必要なCatalog entryとbounded Snapshotだけを返す。

## 12. Editor／AI UX

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

## 13. Save、Replay、Migration

### 13.1 Save対象

- active Weapon instance、slot、ammo、cadence、reload
- active authoritative Projectileのcanonical record
- Vital State
- Pickup available／pending／collected stateとgrant transaction
- Score／Combo／persistent high score
- Encounter phase、Wave、Spawn ordinal、RNG stream state
- Shooter Game Flow
- active Definition、System Graph、Implementation Set、Profile hash

SaveはProjectileを`projectile_spawn_id`順、WeaponをStable ID順、Entity StateをStable ID順に並べる。Pool slot、Runtime handle、Physics native ID、VFX instance、Audio Voice、Camera shakeを保存しない。

### 13.2 Replay

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
  -> Score／Encounter／Result
```

Presentation Event欠落をauthoritative divergenceにしない。Damage、Projectile、Scoreの差異をPresentation-onlyとして無視しない。

### 13.3 Migration

- Field IDをrenameで変更しない。
- enum意味変更は新Type versionを作る。
- cadence algorithm変更は`cycle_distribution` versionとReplay migrationを必要とする。
- Projectile Stateを保存対象から外す変更はMajor migrationである。
- Weapon Definition更新時、live instanceのammoを無言でrefill／truncateしない。Migration Policyを明示する。

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
| `2d_top_down_c1` | 256 | 2,048 | 256 | 128 | 1080p60、authoritative drop 0 |
| `tps_single_player_c1` | 50 | 256 | 64 | 128 | 1080p60、authoritative drop 0 |

この個数はProduct上限ではない。Project intentが上回る場合はProject固有`IntegratedScaleFixtureV1`を生成する。

### 14.3 Pool

Weapon／Projectile／Damage credit／Pickup transactionのRuntime storageはPlay prepareまたはLoading boundaryでProfile capacityを予約する。Poolはprivate実装であり、SourceのProjectile最大数、敵数、Weapon数、Pickup数を下げる理由にしない。

Projectile pool不足時はPattern全体を`ShooterProjectileCapacityExceeded`で拒否し、ammo／cadenceを変更しない。Authoritative Event queue overflowではEventをdrop、merge、sampleせずSessionをfaultする。

Presentation poolだけが不足した場合はcritical cue priorityに従ってPresentationを縮退できるが、Fire、Hit、Damage、Scoreを変更しない。

## 15. FailureとDiagnostic

### 15.1 通常結果

```text
FireRejectedDisabled
FireRejectedGameFlow
FireRejectedDefeated
FireRejectedCadence
FireRejectedReloading
FireRejectedAmmo
ReloadRejectedFull
ReloadRejectedNoReserve
WeaponSwitchRejectedUnavailable
PickupCollectionRejectedUnavailable
PickupCollectionRejectedFilter
PickupGrantRejectedNoCapacity
DamageBlockedTeamPolicy
DamageBlockedInvulnerability
ShotNoHit
ProjectileExpired
```

これらは想定可能なGameplay結果であり、Console errorまたはSession faultにしない。必要なPresentation cueとDebug Eventを持てる。

### 15.2 Diagnostic ID

```text
MIRAKAN-SHOOTER-DEFINITION_INVALID
MIRAKAN-SHOOTER-CONTRACT_VERSION_MISMATCH
MIRAKAN-SHOOTER-CAPABILITY_UNAVAILABLE
MIRAKAN-SHOOTER-STATE_OWNER_CONFLICT
MIRAKAN-SHOOTER-FIRE_TRANSACTION_FAILED
MIRAKAN-SHOOTER-PROJECTILE_CAPACITY_EXCEEDED
MIRAKAN-SHOOTER-QUERY_CAPACITY_EXCEEDED
MIRAKAN-SHOOTER-AUTHORITATIVE_QUEUE_OVERFLOW
MIRAKAN-SHOOTER-DAMAGE_TARGET_INVALID
MIRAKAN-SHOOTER-PICKUP_GRANT_FAILED
MIRAKAN-SHOOTER-SCORE_OVERFLOW
MIRAKAN-SHOOTER-SAVE_CONTRACT_MISMATCH
MIRAKAN-SHOOTER-REPLAY_DIVERGENCE
MIRAKAN-SHOOTER-PRESENTATION_AUTHORITY_VIOLATION
MIRAKAN-SHOOTER-PROFILE_SCALE_UNDERSPECIFIED
```

DiagnosticはRequirement ID、Definition／Field path、System、tick／phase、actual、expected、Capability、Target、State owner、Evidence ID、修正候補を持つ。

### 15.3 Failure policy

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

## 16. TestとQualification

### 16.1 Contract／Schema

- 全Typeのvalid境界、unknown field、duplicate ID、missing ref、cycle
- Stable ID／Contract Ref／Artifact Refの混同拒否
- Fire Mode、Pattern、Delivery、Ammo、Reloadの組合せmatrix
- current Profileへunknown／out-of-contract enumまたはFieldを混入したnegative fixture
- State owner、Command phase、Event delivery、Save closure
- unknown Diagnostic、unbounded collection、NaN／Inf、overflow

### 16.2 Fire／Weapon

- single、automatic、1／2／32 burst
- cadence 1／7／10／60 activation per 60 tickを1,000 cycle実行
- press、hold、release、reload、switch、pause、defeatの境界
- ammo 0／1／capacity、reserve 0／1／capacity
- Pattern 1／63／64 Shot、hitscan／Projectileそれぞれのcapacity丁度／capacity+1
- capacity失敗時にammo、cadence、reload、Eventが不変

### 16.3 Shot／Collision

- 2D／3D hitscan `closest_per_shot`、multi-shot時のcanonical ordering
- straight projectile、thin target、owner ignore、lifetime境界
- same fraction hitのcanonical ordering
- team relation matrix、sensor／solid、initial overlap
- Particle／depth／Camera occlusionをHitへ使う依存を拒否
- hitscanとprojectileが同じDamage Specで同じTarget結果を得るfixture

### 16.4 Damage／Vital／Score

- Health、Shield、invulnerability、exact zero、overkill、defeat一回
- self／ally／neutral／hostile matrix
- friendly fire変更のHigh Impact質問
- direct／radial Damageの境界
- ammo／health／shield／score／weapon Pickup、同tick競合、Grant失敗rollback
- Defeat credit、同一Defeat重複加点0
- combo開始／継続／expire、multiplier、high score Save／Load
- Score int64 overflow fault

### 16.5 Encounter／Game Flow

- Ready→Playing→Paused→Playing→Result→Restart
- Wave 1／256、Spawn group 1／1,024、Boss phase 0／32
- enemy全滅、goal、time、Boss defeatのcompletion
- pause中cadence／reload／Encounter clockがProfileどおり停止
- Save／Load後に同じWave、RNG、Projectile、Score結果

### 16.6 AI Eval

- 「銃」「弾」「ビーム」「連射」「三方向」「弾幕」「強く」「軽く」のcanonical resolution
- authoritative projectileとPresentation particleの混同0
- hitscan／projectile、ammo、friendly fire、scoreのBlocking不足見逃し0
- Low Impact既定を不要質問する率5%以下
- 対象外Capabilityをcurrent contractの成功として返す件数0
- AI／手動Editor／Project C++ Commandが同じDefinition hashとRuntime結果へ収束
- ExplainがField、理由、Assumption、代替、capacity impact、Testを返す

### 16.7 Performance／Soak

- `2d_shooter_c1_v1`、`2d_crowded_battle_v1`、`tps_shooter_c1_v1`、`3d_crowded_battle_v1`をnamed reference scenarioとして固定する
- 14.2節の2D／TPS Fixtureを各Targetで120秒×5 run
- 10分soak、spawn／destroy churn、pause／restart、Save／Load
- CPU／memory／queue／pool high-water、P95／P99／P99.9
- authoritative Fire／Projectile／Hit／Damage／Score drop 0
- Replay hash一致、Presentation degradation時もGameplay一致
- Project固有Scaleが組込みFixtureを超える場合の再Qualification

## 17. Qualification closure

Shooter reference Packは、本文書の全schema、closed enum、境界値、Fire transaction、canonical ordering、State owner、Save／Replay、failureと16節のfixtureを同じPublic Contractで検証する。Domain Packとしてのinstall／apply／update／remove、Project ChangeSet、qualification receiptは[Domain Pack Contract](domain-pack-contract.md)を使い、Capability成熟度と実装順序は[Product Plan](../00-product/product-plan.md)へ委譲する。

2D top-downとsingle-player TPSは同じWeapon、Projectile、Damage、Vital、Pickup、Score、Save／Replay契約へ適合する。Profile固有のCamera、Input、UI、Audio、VFX、Asset templateは各ownerのschemaを参照し、同じFieldまたは固定値を本Packへ再定義しない。

## 18. 公式資料

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

外部資料は責務分離とAuthoring先例の確認に使用する。MiraikanaiのShooter domain Type、State owner、algorithm、failure、Save、Replay、AI Operationは本規約が決定する。共有phase、Risk、budgetは各canonical ownerを参照し、外部EngineのAPI互換性をProduct要件にしない。
