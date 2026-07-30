# Gameplay Feature Definition／Fixture Candidate Catalog

- 文書ID: mirakan.appendix.gameplay-feature-definition-fixture-catalog
- 文書種別: Owner supplement
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 親Owner: [Gameplay Feature Packs](../08-packs/gameplay-features.md)
- 正本範囲: Weapon、Damage、Vital、Score、Encounter、Pickup、Locomotion等のSchema、Registry、Operation、Fixtureのreview候補詳細
- 非正本範囲: Feature共通ownership、manifest、Port、Save／Replay、failure、current activation、Genre composition。親Ownerと各Subsystem Ownerが決定する
- 規範依存: [親Owner](../08-packs/gameplay-features.md)
- 関連文書: [Pack Contract](../08-packs/pack-contract.md)、[Scenario／Stage](../08-packs/scenario-stage.md)、[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)、[Collision](../05-simulation/collision.md)、[Physics](../05-simulation/physics.md)、[Navigation](../05-simulation/navigation.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-23

> 本書は分離前Owner文書のWeapon、Damage、Vital、Score、Encounter、Pickup、Character Locomotion等の具体Schema、Registry、Fixture候補を保持する。親OwnerのFeature共通境界、ownership、Port、Save／Replay、failure意味を上書きせず、ArtifactとQualificationがない候補をactive Packとして扱わない。

> 以下の見出し番号は、親Ownerの論点番号との対応を明示するために維持する。欠番は親Ownerが所有する規範であり、本書に補完しない。

## 3. Canonical data model

すべてのSource objectはUUIDv7 `StableId`、MCD Typeはexact `McdContractRefV1`、Derived Artifactは`ArtifactRefV1`を使う。表示名、Asset path、C++型名、配列indexをidentityにしない。

時間またはSimulation Advanceを使うFeatureは、Compile時にroot外`FeatureRuntimeTimebaseBindingV1`へ`Project revision／document-set hash、PackContractRefV1、Composition Recipe ref／hash、SimulationCadenceProfileRefV1、GameClockDomainProfileRefV1、timebase consumer Type ref集合、Target別Qualification Binding集合、binding hash`を固定する。両Profile Refは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md#41-clock-domainpausegameplay-timer)の完成base recordからrecord外でmaterializeされたexact ID／version／self-excluding content hashであり、Binding内のCadence Profile RefはGame Clock Domain Profileが選択する同Refとbyte equalityにする。Definition／Pack ManifestへProject固有ProfileやReceiptを埋め戻さず、Runtime Package、Save、Replayは同じBinding ref／hashを保存する。以下の`*_advances`は常に選択Cadenceにおける`SimulationAdvanceIntervalV1.advance_sequence`差であり、秒、render frameまたは固定60 Hzを意味しない。速度等のSI時間積分は各Advance recordのnon-null exact rational durationを使い、duration-null Profileでは対応するadvance-driven branchとQualificationがなければ`cadence_profile_not_qualified`でActivation前に拒否する。current C1／C2でこのFeature集合がqualifiedなのはreference `fixed 60/1` Profileだけであり、別rate／variable／turn-based／explicit-step対応は`future.capability.alternate-simulation-cadence-and-substep`のconsumer migrationとTarget別fixture完了後に限る。

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
  cadence:
    kind: rate_per_second
      activation_rate_hz: ReducedPositiveRationalV1
      distribution_policy_ref:
        policy.feature.ranged_combat.cadence.even_floor
    | kind: advance_cycle
      activation_count_per_cycle: uint16
      cycle_advances: uint16
      distribution_policy_ref:
        policy.feature.ranged_combat.cadence.even_floor
        | policy.feature.ranged_combat.cadence.explicit_offsets
      activation_offsets_advances[]: optional
  ammo_cost_per_activation: uint16
  release_policy: stop_unfired | complete_started_cycle
  fire_while_reloading: false
```

`rate_per_second.activation_rate_hz`は既約なpositive rationalとする。Source rateを`p/q Hz`、Compile先のfixed Cadenceを`n/d Hz`とした場合、`raw_count=p×d`、`raw_cycle=q×n`、`g=gcd(raw_count,raw_cycle)`、Cooked schedule=`activation_count_per_cycle=raw_count/g`、`cycle_advances=raw_cycle/g`としてchecked `uint64`でexact導出する。導出後count／cycleが各`[1,3600]`かつcount<=cycleでなければTarget／Profile Qualificationを拒否する。`advance_cycle`も同じ範囲と不等式をSourceで要求する。`burst`だけはcountを`[1,32]`へ制限し、current `automatic`は`rate_per_second`、`single | burst`は`advance_cycle`を使う。C1は一Weaponにつき一advance最大一Activationである。Patternが一Activationから複数Shotを生成する。この上限はper Weapon instanceの不変条件であり、Bindingごとではない。同一advanceに`primary`と`secondary`の両Bindingが発火許可となった場合は`primary`のActivationだけを実行し、`secondary`を`FireRejectedCadence`で拒否してそのcadence cycle開始advanceを次の有効advanceへ繰り延べる。この裁定順は固定であり、同じPublic Contractへ適合するすべての実装とReplayで同一結果とする。

`policy.feature.ranged_combat.cadence.even_floor`はCooked cycle内のActivation `i=[0,count-1]`を`floor(i × cycle_advances / activation_count_per_cycle)` advanceへ配置し、`activation_offsets_advances`を持たない。`policy.feature.ranged_combat.cadence.explicit_offsets`は`activation_count_per_cycle`と同数のoffsetを`[0,cycle_advances-1]`へstrictly increasingで持つ。両方ともexact `McdContractRefV1(kind=policy)`であり、対応Fixtureはowner-typed Qualification subjectだけが解決する。Production DefinitionはReceipt-freeで固定し、root外Activation bindingが指すsigned Qualification Receiptだけを消費する。cycle開始advance、Cadence Profile ref／hash、Definition revisionが同じならscheduleは同じである。「毎秒7発」はSourceで`activation_rate_hz=7/1`とし、fixed `60/1 Hz`では上式によりexact `7 activation / 60 advances`、fixed `120/1 Hz`では`7 / 120 advances`へderiveする。Profileを変えずに60 literalを埋め込む、float timerを累積する、variable／duration-null Profileへ同scheduleを暗黙適用することを禁止する。

`single`はcount 1、offset 0で、trigger rising edgeごとに一cycleだけ評価する。`automatic`はtrigger hold中にcycleを反復する。`burst`はrising edgeで一つの有限cycleを開始し、通常は`policy.feature.ranged_combat.cadence.explicit_offsets`で`[0,5,10]`等を表す。release後の再押下なしに次cycleを開始しない。これによりautomatic cadence、burst内間隔、burst後cooldownを同じscheduleで一意に表す。

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

hitscanはCollision規約のRay／Shape Castを使う。ProjectileはRanged Projectile Systemのdomain-local recordを使い、C1では各advanceの移動segmentをswept Queryとして検証する。Particle collision、depth buffer、render occlusionをHit Evidenceにしない。

`delivery_kind=hitscan`では`projectile_definition_ref`を持たず、Patternの`speed_scale_q16`を全要素1.0、`projectile`ではrefを必須とする。`max_hits_per_activation=[1,64]`かつ`shot_count`以下とし、各Shotは`closest_per_shot`で最大一Hitを生成する。penetration、同一Shotの複数Hit、`ordered_all`は現在のcontractの対象外として拒否する。これらの将来scopeまたはschema判断は[Product Plan](../00-product/product-plan.md)へ委譲する。Projectileは`lifetime_advances`またはspawn時からの積算距離が`max_range_m`へ到達した早い方でexpireする。

### 3.5 `ProjectileDefinitionV1`

```text
ProjectileDefinitionV1
  projectile_definition_id: StableId
  dimension: two_d | three_d
  speed_mps
  lifetime_advances
  query_shape_ref
  collision_query_profile_ref
  owner_ignore_advances
  on_hit_policy: stop
  presentation_cue_ref
```

`speed_mps=(0,10,000]`、`lifetime_advances=[1,36,000]`、`owner_ignore_advances=[0,60]`とする。C1 Projectileはscale、arbitrary acceleration、homing target、bounce count、Physics Backend objectを持たない。

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
  age_advances
  traveled_distance_m
  rng_draw_count
  status: pending | active | hit | expired
```

`projectile_spawn_id`はPlay session内でRanged Projectile Systemが単調増加させ、0を使わない。Pool slot、pointer、Physics handleをSave／Replay identityにしない。

`delivery_definition_ref`はspawn元の`ShotDeliveryDefinitionV1`を指す。各advanceの移動距離は`speed_mps × SimulationAdvanceIntervalV1.interval.logical_duration_seconds`をchecked deterministic numeric policyで積算し、`traveled_distance_m`をauthoritative State／Save／Replayへ保存する。durationがnullなら対応advance-driven Projectile policyなしに実行しない。`max_range_m`によるexpireは、この積算距離と`delivery_definition_ref`が示す`max_range_m`だけで判定する。spawn originの保存を必要とせず、Save／Load後も同じInterval record列から同じexpire advanceへ再決定できる。

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
  duration_advances: optional
  trigger_policy: manual | manual_or_empty, optional
  interruption: not_interruptible | cancel_preserve_ammo, optional
```

capacityは`[0,65,535]`とし、initial値は対応capacity以下とする。`infinite`は4つのcapacity／initial Fieldを0、`ReloadPolicyV1.reload_kind=none`とし、ammo FieldをRuntime State／HUD／Saveへ作らない。`magazine_reserve`は`magazine_capacity>=1`を要求する。`reload_kind=none`では3つのoptional Fieldを持たず、`full_magazine`では`duration_advances=[1,36,000]`とtrigger／interruptionをすべて必須とする。C1 reloadは完了時に必要量をreserveからmagazineへ一度だけ移し、animation eventやAudio completionをauthoritative完了条件にしない。

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
    cadence_cycle_start_advance
    cadence_activation_index
    trigger_held
  reload_state
  reload_complete_advance
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

`0 <= inner_radius_m <= outer_radius_m <= 1,000`、`max_targets=[1,256]`とする。`none`はquery refを持たず、`collision_query`は必須とする。候補はdistance、Target Stable IDの順でcanonicalに選び、`max_targets`超過をpartial Damageで継続せず`CombatQueryCapacityExceeded`として当該advanceをfaultする。`linear_q16`はinnerで1.0、outerで0.0のQ16倍率とし、Gameplay DamageをVFX radiusから取得しない。

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
  invulnerable_until_advance
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
  credit_advance
```

`VitalStateV1.last_damage_credit`は同recordへの参照であり、`defeat_credit_policy=last_hit`の確定、`DefeatEventV1`のcredit、Score加点の根拠に使う。creditはDamage適用が確定したadvanceだけで更新し、blockされたDamageでは更新しない。

### 3.12 `ScoreRuleDefinitionV1`と`ScoreStateV1`

```text
ScoreRuleDefinitionV1
  rule_id: StableId
  event_kind: defeat | pickup_collected | objective_completed | wave_completed | boss_defeated
  base_points: int64
  combo_increment
  combo_expire_advances
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
  combo_expire_advance
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
  time_limit_advances: optional
  boss_phase_refs[0..32]
  difficulty_profile_ref
  rng_stream_ids[]

EncounterPhaseV1
  phase_id: StableId
  wave_refs[1..256]

EncounterWaveV1
  wave_id: StableId
  start_advance
  spawn_group_refs[1..1024]

SpawnGroupV1
  spawn_group_id: StableId
  enemy_archetype_ref
  spawn_transform_ref
  spawn_count: uint16
  spawn_start_offset_advances
  spawn_interval_advances
```

各waveは丁度一つのphaseの`wave_refs`に、各spawn groupは丁度一つのwaveの`spawn_group_refs`に現れ、未参照と重複参照を拒否する。`boss_phase_refs`は`phases`のsubsetである。advance offsetはEncounter開始を0とし、`start_advance=[0,36,000]`、`spawn_start_offset_advances=[0,36,000]`、`spawn_interval_advances=[0,3600]`、`spawn_count=[1,256]`とする。`spawn_count>=2`は`spawn_interval_advances>=1`を要求する。

`completion_condition`はclosed enumであり、`goal_reached`だけが`goal_ref`を必須として他では持たない。`time_expired`だけが`time_limit_advances=[1,36,000]`を必須として他では持たない。`boss_defeated`は`boss_phase_refs`が非空であることを要求する。

Spawn順、Wave遷移、Boss phaseは選択Cadenceのadvance sequence、Stable ID順、登録済みRNG streamで決める。Definitionは`FeatureRuntimeTimebaseBindingV1`と異なるCadence Profileへ再生せず、VFX completion、Audio duration、Animationのpresentation-only Notifyを進行条件にしない。

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
  collected_advance: optional
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
  max_requests_per_advance
```

各provider FeatureまたはProjectは自身のmanifestから`GrantRequestPortV1`実装を登録する。同じ`grant_type_ref`へのactive providerはexactly oneとし、未登録型は`rejected_unsupported_type`、capacity不足は`rejected_no_capacity`を返す。Pickup Systemはprovider Stateを直接writeしない。

同一Pickupへ同一advanceに複数collectorが接触した場合は`pickup_instance_id, collector StableId`順の先頭だけを`pending_grant`へ遷移する。providerが`accepted`を返した時だけ`collected`へ進み、通常拒否では`available`へ戻す。pending中の重複collectionを禁止し、queue／invariant失敗はpending transactionを保持してSession faultとする。現在のcontractはone-shot collectionだけを扱い、respawn、random loot table、inventory weightは対象外として拒否する。

## 4. Game SystemとState owner

### 4.1 Feature-owned System

| System Contract | Scope | 所有State | 主な責務 |
|---|---|---|---|
| `game_system.extension.feature.weapon` | `scope.core.entity` | `WeaponInstanceStateV1`、`WeaponLoadoutStateV1` | trigger、cadence、ammo、reload、switch、Fire transaction |
| `game_system.extension.feature.ranged_projectile` | `scope.core.world` | `RangedProjectileStateV1` collection | spawn、swept query、hit／expire、capacity |
| `game_system.extension.feature.combat` | `scope.core.world` | Damage credit ledger | Hit Evidence、Team、Damage rule、creditの検証 |
| `game_system.extension.feature.vital` | `scope.core.entity` | `VitalStateV1` | Health、Shield、invulnerability、defeat |
| `game_system.extension.feature.score` | `scope.feature.scoring.instance` | `ScoreStateV1` | 加点、combo、multiplier、high score |
| `game_system.extension.feature.pickup` | `scope.core.world` | `PickupInstanceStateV1` collection | overlap Evidence、provider-neutral typed Grant、one-shot state |
| `game_system.extension.feature.encounter` | `scope.feature.encounter_spawn.instance` | Encounter runtime state | Wave、Spawn、Boss phase、completion |
| `game_system.extension.feature.character_locomotion.contribution` | `scope.core.entity` | なし（derived） | Character Gameplay intentをgeneric motion contributionへ写像 |

全Systemは`owner_layer=feature_pack`、exact Feature `owner_ref`、`system_origin=owner_package`を持つ。同じPublic Contractへ適合するProject-supplied実装VariantをPolicyが許可してもSpecのowner layerは変えず、default Variantと同時にactiveにしない。WeaponとVitalをCharacter Systemのprivate Fieldへ隠さず、Public State owner tableへ出す。

`feature.scoring`と`feature.encounter_spawn`は`RuntimeScopeTypeCatalogV1`へ次のexact rowを登録する。

| `scope_type_ref` | `instance_key_schema_ref` | `owner_ref` | `lifetime_ref` | `save_replay_policy_ref` | `activation_condition_ref` | `deactivation_condition_ref` |
|---|---|---|---|---|---|---|
| `scope.feature.scoring.instance` | `type.runtime_scope.key.feature_scoring_uuidv7` | `owner.feature.scoring` | `policy.runtime_scope.lifetime.feature_scoring_instance` | `policy.runtime_scope.save_replay.feature_scoring` | `policy.runtime_scope.activation.feature_scoring_request_ready` | `policy.runtime_scope.deactivation.feature_scoring_stop_or_fault` |
| `scope.feature.encounter_spawn.instance` | `type.runtime_scope.key.feature_encounter_spawn_uuidv7` | `owner.feature.encounter_spawn` | `policy.runtime_scope.lifetime.feature_encounter_spawn_instance` | `policy.runtime_scope.save_replay.feature_encounter_spawn` | `policy.runtime_scope.activation.feature_encounter_spawn_request_ready` | `policy.runtime_scope.deactivation.feature_encounter_spawn_stop_or_fault` |

各scope typeは`RuntimeScopeTypeRefV1 {scope_type_id, scope_type_version=1, scope_type_hash}`、type／policy cellは`McdContractRefV1 {id, version=1, contract_set_hash}`、owner cellは`RuntimeScopeOwnerRefV1 {owner_id, owner_revision, owner_content_hash}`として保存し、表の裸IDを永続化しない。全dependency recordをactive Scope Registryへ登録する。Save identity、Replay identity、ephemeral runtime generationを別Fieldで保持し、複数instanceのStateをSource IDまたはRuntime handleで合成しない。

Feature SystemのScopeと`GameSystemSpecV1.runtime_scope_type_ref`はinitial V1から上表のtyped refを使用する。対応する旧System bytesまたはreader／writerはmaterializeされていないため、`play_session`／`encounter_instance`／`entity_instance`からのRuntime Scope migration contribution、offline Operation、alias、legacy fixtureをcurrent Packへ定義しない。過去draftの裸scope値をowner名またはSystem名からcurrent refへ推測変換しない。

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
  accepted_advance
```

Port descriptorのexact MCD refは`{id=type.feature.ranged_combat.collision_query_port, version=1, contract_set_hash}`であり、request／result schema、Collision owner、consume phase、capacity、failure Diagnosticを閉じる。Genre Pack、Project provider、fixtureはこの三refを再定義せず、version／Contract set hash込みで参照する。旧`mirakan.port.*`、`mirakan.event.*`、PascalCase ID、ID内`@1`をcurrent validatorはrejectする。

`game_system.extension.feature.character_locomotion.contribution`が受理するFeature-owned型を次の一型へ固定し、generic contribution、selection、transport batchをFeature ownerへ複製しない。

```text
GameplayMotionIntentV1
  intent_id
  actor_ref
  actor_generation
  desired_linear_motion
  desired_angular_motion
  cadence_profile_ref: SimulationCadenceProfileRefV1
  simulation_advance_interval_hash: SHA-256
  valid_for_advance_sequence
  movement_profile_ref
```

exact MCD refは`{id=type.feature.character_locomotion.gameplay_motion_intent, version=1, contract_set_hash}`である。出力はNavigation-owned `{id=type.navigation.motion_intent_contribution, version=1, contract_set_hash}`、最終transportはCore publisherだけが発行する`type.navigation.motion_executor_intent_batch`である。旧案`type.feature.character_locomotion.motion_executor_selection_state`はactivateされておらず、current MCD／Source／Save／Replayに登録もaliasも持たない。selected Provider identityはNavigationのSelection Documentとbatch closureだけが所有する。IDに`mirakan.schema`／`mirakan.contract`、PascalCase、`@1`を含めない。

<a id="41-character-locomotion-binding-system"></a>

`game_system.extension.feature.character_locomotion.contribution`の全mandatory Fieldは次へ固定する。次のrecordは[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)が所有するcanonical `GameSystemSpecV1` schemaのCharacter Locomotion具体instanceであり、schemaの再定義ではない。

```text
GameSystemSpecV1
  MCD common envelope: all fields
  id: game_system.extension.feature.character_locomotion.contribution
  version: 1
  status: active
  title: Character Locomotion Motion Intent Contribution
  description: validate Character gameplay intent and emit one generic contribution
  owners: [owner.feature.character_locomotion]
  owner_layer: feature_pack
  owner_ref:
    {owner_layer=feature_pack,
     owner_id=owner.feature.character_locomotion,
     owner_revision, owner_content_hash}
  requirement_refs:
    [{id=requirement.feature.character_locomotion.emit_motion_intent_contribution, version=1, contract_set_hash},
     {id=requirement.non_responsibility.character_locomotion.no_batch_publish, version=1, contract_set_hash},
     {id=requirement.non_responsibility.character_locomotion.no_transform_write, version=1, contract_set_hash},
     {id=requirement.non_responsibility.character_locomotion.no_physics_write, version=1, contract_set_hash}]
  rationale_refs: [mirakan.arch.pack-gameplay-features#41-character-locomotion-binding-system]
  since_contract_set: 1
  supersedes: []
  tags: [character_locomotion, motion_intent_contribution, provider_neutral]
  system_origin: owner_package
  semantic_role_refs:
    [{id=role.game_system.character_locomotion.contribution, version=1, content_hash}]
  responsibility_requirement_refs:
    [{id=requirement.feature.character_locomotion.emit_motion_intent_contribution, version=1, contract_set_hash}]
  non_responsibility_requirement_refs:
    [{id=requirement.non_responsibility.character_locomotion.no_batch_publish, version=1, contract_set_hash},
     {id=requirement.non_responsibility.character_locomotion.no_transform_write, version=1, contract_set_hash},
     {id=requirement.non_responsibility.character_locomotion.no_physics_write, version=1, contract_set_hash}]
  runtime_scope_type_ref:
    {scope_type_id=scope.core.entity, scope_type_version=1, scope_type_hash}
  state_class: derived
  owned_state_type_refs: []
  read_snapshot_type_refs: []
  accepted_command_type_refs:
    [{id=type.feature.character_locomotion.gameplay_motion_intent, version=1, contract_set_hash}]
  emitted_event_type_refs: []
  emitted_port_message_type_refs:
    [{id=type.navigation.motion_intent_contribution, version=1, contract_set_hash}]
  provided_capability_refs:
    [{id=capability.gameplay.character_locomotion, version=1, contract_set_hash}]
  required_capability_refs:
    [{id=capability.navigation.motion_intent_binding_resolve, version=1, contract_set_hash}]
  allowed_phase_ids: [T40_MotionIntent]
  dependency_edge_refs:
    [{id=dependency.game_system.character_locomotion.navigation_contribution_port,
      version=1, content_hash}]
  implementation_policy_ref:
    {id=implementation_policy.feature.character_locomotion.contribution, version=1, content_hash}
  save_replay_contract_ref: canonical omission
  behavior_budget_refs:
    [{id=budget.feature.character_locomotion.contribution.per_target, version=1, content_hash}]
  authoring_surface_ids: [form, table, graph]
  fallback_contract: no_fallback(reason=invalid Character intent produces no contribution)
  compatibility_invariant_refs:
    [{id=invariant.character_locomotion.generic_contribution_only, version=1, content_hash},
     {id=invariant.character_locomotion.owner_binding_exact, version=1, content_hash},
     {id=invariant.character_locomotion.no_authoritative_write, version=1, content_hash}]
  auxiliary_ref_set_hash:
    SHA-256(MIRAKAN_GAME_SYSTEM_AUXILIARY_REF_SET_V1,
      exact sorted auxiliary refs above, self excluded)
  extension_policy: sealed
```

本Systemはauthoritative stateもselection stateも所有せず、Feature固有Gameplay intentからgeneric contributionを最大一件生成する。Navigation-owned `game_system.engine.navigation.motion_intent_batch_publisher`が全owner contributionをcanonical mergeして`MotionExecutorIntentBatchV1`を一度だけ発行するため、本SystemをBatch producer、selection owner、generic publisherにしない。dependency edgeはNavigation-owned contribution inputへのexact Port dependencyであり、selected Providerへ直接提出しない。Transform、Physics body／state、Provider-private profile、Animation clockへwriteしない。`type.feature.character_locomotion.gameplay_motion_intent`からgeneric `type.navigation.motion_intent_contribution`へのadapter policy、Character-owned binding contribution、Physics Kinematic Motion Providerとのcompatibility record、Qualification record／FixtureはこのFeature ownerが`MotionIntentContributionBindingRegistryV1`へ登録する。Navigation／Physics CoreへFeature type IDやFixtureを追加しない。

### 4.1.1 Character Locomotion依存record registry

上記Specが参照するrole／requirement／dependency／policy／budget／invariantと、Spec固定後にroot外Activation bindingが参照するQualification Receiptは、単なる文字列ではなく次のactive recordを厳密に一件ずつ持つ。MCD requirementは`McdContractRefV1`、Receipt-free補助recordは各typed refの`id, version, content_hash`三点を必須とする。Qualification ReceiptはSpec／補助hashへ含めない。

| exact record | record type | 解決内容 |
|---|---|---|
| `role.game_system.character_locomotion.contribution` v1 | `SemanticRoleRecordV1` | owner Feature、Gameplay intent validation、generic contribution emission。selection／batch／Transform／Physics ownerではない |
| `requirement.feature.character_locomotion.emit_motion_intent_contribution` v1 | MCD `requirement` | valid Character Gameplay intentをT40でgeneric contribution一件へ写像するmust |
| `requirement.non_responsibility.character_locomotion.no_batch_publish` v1 | MCD `requirement` | selection解決とcanonical batch publishを行わないmust_not |
| `requirement.non_responsibility.character_locomotion.no_transform_write` v1 | MCD `requirement` | Transform authoritative Stateをwriteしないmust_not |
| `requirement.non_responsibility.character_locomotion.no_physics_write` v1 | MCD `requirement` | Physics Body／State／native objectをwriteしないmust_not |
| `dependency.game_system.character_locomotion.navigation_contribution_port` v1 | `GameSystemDependencyEdgeV1` | target=Navigation generic contribution input、kind=`port_target`、transport=`type.navigation.motion_intent_contribution` v1、T40 publisher-before relation、required、no inferred fallback |
| `implementation_policy.feature.character_locomotion.contribution` v1 | `GameSystemImplementationPolicyV1` | Feature Definition default、Project replacement禁止、live switch禁止、全Targetで同じschema、unavailable=activation reject |
| `budget.feature.character_locomotion.contribution.per_target` v1 | `BehaviorBudgetRecordV1` | actor当たりCharacter contribution一件、bounded payload／queue、overflow=typed reject |
| `qualification.feature.character_locomotion.contribution` v1 | `GameSystemQualificationReceiptV1` | separate Qualification recordのsubject／Target／fixture-set resultを署名し、Fixture bodyを含まない |
| `invariant.character_locomotion.generic_contribution_only` v1 | `CompatibilityInvariantRecordV1` | emitted Port type setはgeneric contribution exact一件、Navigation batchは0件 |
| `invariant.character_locomotion.owner_binding_exact` v1 | `CompatibilityInvariantRecordV1` | Spec、adapter、binding contributionのFeature owner ref／hashがexact equality |
| `invariant.character_locomotion.no_authoritative_write` v1 | `CompatibilityInvariantRecordV1` | owned authoritative typeとTransform／Physics write edgeは0件 |

非MCD recordの共通headerは`record_id`、`record_version=1`、`record_content_hash`、owner ref／hash、`status=active`、introduced Contract set hashを持つ。各record hashはhash Fieldを除くheaderと次のtyped payloadをMCD canonical encodeして計算する。

```text
SemanticRoleRecordV1.payload
  accepted_input_type_refs:
    [exact type.feature.character_locomotion.gameplay_motion_intent ref]
  emitted_port_message_type_ref:
    exact type.navigation.motion_intent_contribution ref
  allowed_phase_ids: [T40_MotionIntent]
  authoritative_write_type_refs: []
  forbidden_write_requirement_refs:
    [exact requirement.non_responsibility.character_locomotion.no_transform_write ref,
     exact requirement.non_responsibility.character_locomotion.no_physics_write ref]

GameSystemDependencyEdgeV1.payload
  target_contract_ref: exact Navigation generic contribution input ref/revision/hash
  edge_kind: port_target
  transport_type_ref: exact type.navigation.motion_intent_contribution v1
  phase_relation: same_phase_t40_before_core_batch_publisher
  delivery: at_most_once_per_actor_generation_advance
  required: true
  fallback: no_fallback

GameSystemImplementationPolicyV1.payload
  allowed_implementation_kinds: [owner_definition]
  default_implementation_ref: exact Feature contribution implementation ref/hash
  native_eligibility: false
  replacement_policy: sealed
  live_switch_policy: forbidden
  required_target_refs: all active Project Targets
  configuration_schema_ref:
    exact {id=type.game_system.empty_configuration,
           version=1,contract_set_hash}
  unavailable_behavior: reject_activation_keep_last_valid

BehaviorBudgetRecordV1.payload
  max_contributions_per_actor_generation_advance: 1
  max_inline_payload_bytes_per_entry: 65536
  overflow_behavior: typed_reject_no_partial_contribution
  target_budget_refs: exact set per active Target

CompatibilityInvariantRecordV1.payload
  predicate_kind:
    emitted_generic_contribution_only
    | feature_owner_binding_exact
    | authoritative_write_edge_count_zero
  evaluation_phase: activation | T40_MotionIntent
  input_contract_refs[1..8]: exact version/hash-bound refs
  expected: true
  failure_code:
    MIRAKAN-LOCOMOTION-CONTRIBUTION-INVALID
    | MIRAKAN-LOCOMOTION-AUTHORITY_VIOLATION
  failure_behavior: reject_without_partial_contribution

```

次の三recordは[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)が所有するcanonical `GameSystemQualificationSubjectV1`、`GameSystemQualificationReceiptV1`、`GameSystemActivationBindingV1` schemaのCharacter Locomotion具体instanceであり、schemaの再定義ではない。

```text
GameSystemQualificationSubjectV1
  qualification_id: qualification.feature.character_locomotion.contribution
  qualification_version: 1
  owner_ref:
    exact {owner_layer=feature_pack,
           owner_id=owner.feature.character_locomotion,
           owner_revision, owner_content_hash}
  system_ref:
    exact game_system.extension.feature.character_locomotion.contribution ref
  system_contract_hash: exact system_ref resolved contract hash
  target_profile_refs[1..64]
  fixture_refs[1..64]:
    exact fixture.feature.character_locomotion.* ref/version/content_hash
  input_closure_hash:
    SHA-256(MIRAKAN_CHARACTER_LOCOMOTION_QUALIFICATION_INPUT_V1,
      exact binding contribution ref, adapter policy ref,
      invariant refs, Target refs, fixture refs; count/length framed)
  result: pass | fail
  qualification_subject_hash: SHA-256

GameSystemQualificationReceiptV1
  subject: exact completed GameSystemQualificationSubjectV1 above
  signed_record:
    exact MirakanSignedRecordV1(
      purpose=game_system_qualification,
      subject_sha256=SHA-256(JCS(subject)))

GameSystemActivationBindingV1
  activation_binding_id:
    activation.game_system.feature.character_locomotion.contribution
  activation_binding_version: 1
  system_ref: exact subject.system_ref
  system_contract_hash: exact subject.system_contract_hash
  qualification_receipt_refs[1]:
    {qualification_id=qualification.feature.character_locomotion.contribution,
     qualification_version=1,
     qualification_subject_hash=subject.qualification_subject_hash,
     signed_record_hash=SHA-256(JCS(receipt.signed_record completed bytes))}
  activation_binding_hash: SHA-256
```

Character Locomotionの生成順は`Receipt-free Spec／補助record hash → Contract set root／system_ref → Qualification subject → signed Receipt → Activation binding → GameSystem Catalog projection`だけである。subject owner／System／contract hashは参照先Specの`owner_ref`／external System ref／resolved contract hashとbyte equality、Activation bindingのSystem pairはsubjectとbyte equalityでなければならない。Spec、Implementation Policy、auxiliary set hash、Contract set rootへReceipt／Bindingを戻さない。owner、System、contract hash、binding contribution、Target、fixture、result、subject hash、signed record hashの一Fieldだけをstale／substituteするnegative fixtureを各一件持つ。

四MCD requirementは次の全Fieldを持つ。

| ID | normative／priority | statement | scope／verification／acceptance | failure code |
|---|---|---|---|---|
| `requirement.feature.character_locomotion.emit_motion_intent_contribution` | must／blocking | valid Character Gameplay intentをgeneric contribution exact一件へ写像する | contribution System／T40、contract＋runtime conformance、output count=1 and adapter hash valid | `MIRAKAN-LOCOMOTION-CONTRIBUTION-INVALID` |
| `requirement.non_responsibility.character_locomotion.no_batch_publish` | must_not／blocking | selectionを解決せずNavigation batchを発行しない | System emitted type／dependency set、batch output count=0 | `MIRAKAN-LOCOMOTION-AUTHORITY_VIOLATION` |
| `requirement.non_responsibility.character_locomotion.no_transform_write` | must_not／blocking | contribution SystemはTransform owner Stateを書かない | System write set、static＋runtime conformance、Transform write edge=0 | `MIRAKAN-LOCOMOTION-AUTHORITY_VIOLATION` |
| `requirement.non_responsibility.character_locomotion.no_physics_write` | must_not／blocking | contribution SystemはPhysics State／native objectを書かない | System write set、static＋runtime conformance、Physics write edge=0 | `MIRAKAN-LOCOMOTION-AUTHORITY_VIOLATION` |

各requirementはsource ref=`mirakan.arch.pack-gameplay-features#41-character-locomotion-binding-system`、introduced_by=本Feature contract ChangeSet、owner、version、Contract set hashを持つ。禁止対象は上記registered `must_not` Requirement refで表し、未登録owner文字列をSemantic Role payloadへ置かない。Physics ownerを別のtyped owner relationで照合する場合のcanonical identityは`owner.core.physics`だけであり、別namespaceのaliasを作らない。Fixture bodyは上記`GameSystemQualificationSubjectV1`だけが解決し、production Physics、fixture board／RTS、root motionあり／なし、type spoof／hash／duplicate／order／stale／failureのinput、oracle、不変state hashを列挙する。三`CompatibilityInvariantRecordV1`は上表順に三predicate branchを一対一で使い、共通headerの`record_content_hash`とSpecの`CompatibilityInvariantRecordRefV1 {id,version,content_hash}`をbyte equalityで解決する。評価phase、input refs、expected Boolean、failure codeをhashへ含め、旧`InvariantRecordV1`、自由式、bare invariant ID、wrong predicate／owner／hash、三件のmissing／extraを拒否する。

RegistryはID、version順にsortし、duplicate、missing owner、version／hash mismatch、deprecated recordをrejectする。`validator.feature.character_locomotion.v1`はMCD Envelope、owner layer/ref、一Feature-owned input型、Navigation generic contribution型、全Receipt-free typed auxiliary ref、budget、invariantをSpecからexact解決し、Qualification Receiptはroot外Activation bindingからだけ解決する。Production SpecのQualification／Fixture ref、Save／Replay ref、selection state、Navigation batch output、Core namespace、`system_origin=engine_standard`、旧`semantic_role_ids`、inline dependency／policy／budget、bare invariant ID、Transform／Physics writeを拒否する。旧Spec v1がselection stateまたはbatch ownershipを主張する場合、current Specへ自動昇格せずoffline migration conflictとしてlast-valid current Specを維持する。

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

次のblockは既に本書で定義したCommand schemaを列挙するsymbol inventoryであり、`GrantRequestV1`を含め型を再定義しない。

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

次のblockは既に本書で定義したEvent／result schemaを列挙するsymbol inventoryであり、`GrantResultV1`および`ShotHitEventV1`を含め型を再定義しない。

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

Eventは原因となるCommand ID、Cadence Profile ref、Simulation Advance interval hash／sequence、producer System、owner／target Stable ID、Definition revision、Evidence refを持つ。Display text、localized string、VFX Asset pathをauthoritative Eventへ入れない。

### 5.3 Snapshot

- `WeaponSnapshotV1`: active slot、ammo、reload、next activation、enabled
- `RangedProjectileSnapshotV1`: count、canonical projectile records、capacity
- `VitalSnapshotV1`: Health、Shield、invulnerability、life state
- `ScoreSnapshotV1`: score、combo、multiplier、high score
- `PickupSnapshotV1`: available／pending／collected、transaction、collector、collection advance
- `EncounterSnapshotV1`: encounter、phase、wave、remaining authoritative actor

HUD、AI behavior、Debuggingは必要なbounded Snapshotだけを読む。Render、Audio、VFXはPresentation projectionを使い、authoritative ownerを直接queryしない。

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

DiagnosticはRequirement ID、Definition／Field path、System、Cadence Profile／advance sequence／phase、actual、expected、Capability、Target、State owner、Evidence ID、修正候補を持つ。

`MIRAKAN-FEATURE-PRESENTATION_AUTHORITY_VIOLATION`はRender、Audio、VFX、Camera、UI、presentation-only EventまたはfixtureからFeature-owned authoritative State／Command結果へwriteまたは逆入力した時の共通Diagnosticである。Build／conformanceを失敗させ、authoritative State、last-valid Source、Feature receiptを維持する。

### 7.3 Failure policy

| Failure | 結果 |
|---|---|
| Definition／Contract不正 | ChangeSet／Cookを拒否し、last-valid Artifactを維持 |
| State owner重複 | System Bundleをactivateしない |
| Fire capacity不足 | Fire transaction全体を拒否し、authoritative State不変 |
| Query result overflow | partial hitを使わずadvance fault |
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
- fixed 60/1と120/1 ProfileでSource rate 1／7／10／60 Hzをexact Cooked cycleへ変換し1,000 cycle実行
- press、hold、release、reload、switch、pause、defeatの境界
- primary／secondary両trigger同時holdの同一advance arbitration（primaryのActivation、secondary側`FireRejectedCadence`と繰延）
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
- ammo／health／shield／score／weapon Pickup、同一advance競合、Grant失敗rollback
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
| `fixture.feature.interaction.contract` | Interaction ownerのexact public contract、初期`spatial`／`logical`／`ui`三Contribution、排他的typed binding、World／Collisionなしlogical／ui、branch mismatch／空間Field偽装拒否 |
| `fixture.feature.character_locomotion.motion_executor` | Navigation-owned PortのPhysicsなし2D／3D stub、optional Physics reference Provider、stale generation |
| `fixture.feature.path_following.executor_stub` | Navigation ownerのpath execution port、board-token／RTS stub、missing／incompatible Provider |
| `fixture.feature.scenario_stage.none` | `completion_mode=none`でcompletion ownerを要求しないStage |
| `fixture.feature.scenario_stage.aggregate-manifest-set-equality` | Scenario Stage owner／aggregate／Contract Manifestのexact ref set equality |

各fixtureはPack単体または宣言済みFeature closureだけでinstall／validate／executeできなければならない。Genre Pack、Genre Profile、product fixture、Genre固有Action roleをtest dependencyへ含めない。manual authoring、AI生成、manual再編集、AI再編集は同じSourceとFeature operationを使い、同じDefinition hash、Receipt、Runtime結果へ収束する。

Character Locomotion fixtureはPhysics capabilityとPhysics Packが不在でもPack installを成功させる。Physics Kinematic Motion Providerを選択したC1 reference caseだけがそのProvider qualification receiptを要求し、Provider failure時にlast-valid resolved motionとFeature Stateが不変であることを検証する。

### 8.8 Performance／Soak

- `fixture.feature.ranged_combat.contract`へ2D／3D Collision stubとscale input exact／exact+1を与える
- `fixture.feature.encounter_spawn.contract`と`fixture.feature.pickup_grant.provider_neutral`を組み合わせた10分soak、spawn／destroy churn、Save／Load
- CPU／memory／queue／pool high-water、P95／P99／P99.9
- authoritative Fire／Projectile／Hit／Damage／Score drop 0
- Replay hash一致、Presentation degradation時もGameplay一致
- Project固有Scaleが組込みfixtureを超える場合の再Qualification

## 9. Feature authoring operation

current Feature authoring write surfaceは非活性である。Feature Source mutation要求は`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`で拒否し、Project／Pack Source、revision、last-valid artifactを変更しない。Genre planned actionも完全登録とActivationまではcallableでなく、composition、Profile、Game Flow、Action role、統合Qualification bindingの境界を越えてFeature Sourceを直接writeしない。

```text
Current Feature authoring Operation set = {}

Previously proposed, never activated logical IDs:
  operation.feature.create_definition
  operation.feature.update_definition
  operation.feature.configure_system
  operation.feature.bind_runtime_port
  operation.feature.preview_change
  operation.feature.explain_contract
  operation.feature.validate_contract
```

七IDは`planning.operation_family.feature_authoring@1`の予約候補以外に存在せず、current MCD、Core Authoring Gateway Manifest、Service allowlist、Provider／MCP Catalog、各Feature Manifestに存在せず、legacy aliasとしても読まない。Capability stateは`not_activated`で、要求は`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`としてSource不変で拒否する。future work item `activation.feature.authoring_operations.v1`は、採用するexact Operation setを一つに固定し、initial create/upsertとupdate、named input／result、semantic intent hash、`MutationAuthorizationBindingV1`、Policy／Validator／Diagnostic closure、canonical signed Receipt、Qualificationを同じContract set transactionで完全登録するまでactivateしない。state-changing Operationのpublication／crash recoveryは[Executable Contracts §8](../02-foundation/executable-contracts.md#8-operation定義)をcanonical reuseし、`private Marker read-back → secret-free PublicCommitClosureV1 candidate → signed wrapper read-back → PublicCommitClosureV1＋PublicPublicationMarkerV1＋after stateのatomic CAS`へ固定する。Closureは`domain_commitment.kind=owner_typed_state_commit`、exact selected Feature owner、Prepared payloadが束縛したreceipt-free committed artifact ref集合を持ち、Ref／hash規則を本書へ複写しない。Closure bodyまたは同Closureを束縛するsigned wrapperを欠くPublic Marker／after-state current authorityを拒否する。read-only preview／explain／validateを将来別Operationとして採用する場合も、name-only entryを先行公開せず、state-changing publicationを発生させない。
