# Shooter Reference Catalog

- 文書ID: mirakan.appendix.shooter-reference-catalog
- 文書種別: proposal appendix
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 親Owner: [Shooter Genre Pack](../08-packs/shooter.md)
- 正本範囲: Shooter AI composition、Fixture、Difficulty、Input template、UX、Qualificationの候補Catalog
- 非正本範囲: 親Ownerが所有する安定Architecture原則、実装Task、実装順序、生成済みArtifactまたはQualification結果
- 規範依存: [親Owner](../08-packs/shooter.md)
- 関連文書: [Architecture Governance](../01-governance/architecture-governance.md)
- 根拠区分: project-decision／provisional。実ArtifactがないRegistry、Catalog、Fixtureは候補
- 外部根拠確認日: 2026-07-27

> この補助文書の型、Registry、Catalog、Fixtureは、対応するRepository Artifactが存在しない限り未実装の設計候補である。親Ownerの安定原則や実装済み状態を上書きしない。
## 7. AI composition

AIは[Pack Contract](../08-packs/pack-contract.md)のplanned Pack actionを使い、要求をGenre identity、Profile、Feature Capability closure、Target、fixtureへ解決する。Pack authoringのcurrent MCD Operation集合は空であり、Activationまではmutation要求を`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`で拒否する。2D／3D、finite／endless、input modality、Camera、required Featureが未確定でGame構造が変わる場合は`question_required`とし、類似表示名からGenreやProfileを推測しない。

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

`genre.shooter.top_down_2d`、`profile.shooter.top_down_2d`、Feature Capability closureを使い、ReadyからPlaying、Paused／resume、Result、restartを検証する。Damage／Vital／Faction、Ranged Combat、Encounter／Spawn、Scoring、Pickup／Grant、Interaction、Character Locomotion、Path Following、Scenario／Stageが各Feature OwnerのPublic Contractだけを使用することを確認する。Perception境界の出入り、`search_last_known`／`return_to_route`、target消失後のfire-intent停止advanceをfixed inputで検証する。

### 8.2 `fixture.product.shooter-arena-3d`

`genre.shooter.third_person_3d`、`profile.shooter.third_person_3d`、同じFeature Capability semanticsを使い、third-person Camera／Audio／LOD binding、3D locomotion、path following、Save／Load、Replay、Resultまでを検証する。target取得／lost-target／fire-intentのEvent sequenceとReplay hashが2Dと同じowner contractに適合することを確認し、Feature schemaをforkしない。

### 8.3 `fixture.genre.shooter.endless-top-down-2d`

`recipe.shooter.top_down_2d.endless`を選択し、初期Shooter baselineのEncounter、Scoring、Pickup、Interaction、Locomotion、Path、Perception closureを維持しつつ、effective closureとclosure hashに`feature.scenario_stage@1`が含まれないこと、World／Scene activation後にObjective／Completion／Result routeなしでPlayingを継続できること、Pack registry上にScenario／Stage FeatureがなくてもRecipe apply／Save／Load／Replayが成功することを検証する。

### 8.4 `fixture.genre.shooter.target-practice-minimal`

`recipe.shooter.target_practice.minimal`、`profile.shooter.target_practice`の`genre_ref=genre.shooter.top_down_2d`、Pack-owned `template.genre.shooter.target_provider.stationary`、Fixture Registryのfixture-only Binding recordを選択し、effective closureが`feature.ranged_combat@1`と推移依存`feature.combat@1`だけであることを検証する。fixture-only Collision Query Port implementationがWorld／Physics／Perceptionなしにdeterministic Hit Evidenceを返し、Fire／Hit／Damage、Qualification sandbox内のSave／Load／Replayが成功する。ただしfixture bindingをProduction Source／Registry／Save／Packageへ選択するnegative caseは必ずrejectする。

`fixture.genre.shooter.target-practice-minimal-no-perception`はPerception Validator／Profile、full 2D／TPS fixture、Scenario／Stage、Scoring、Character Locomotion、Path Following、Encounter、Pickup、Interactionを未installにしたregistryからminimal Recipeだけをapply／qualifyするregression fixtureである。Manifest inventoryに`validator.genre.shooter.perception_binding`やfull fixtureが存在しても選択Recipe gateへ追加されないことを検証する。

`fixture.genre.shooter.target-practice-minimal-project-provider`はProduction modeでProject revision NのOperation inputからBinding Documentをcreateし、`private Marker read-back→owner-typed PublicCommitClosure candidate→canonical signed wrapper→Closure＋Public Marker＋Projectのatomic CAS`をreadbackしてN+1のrevision／document set hashを確認した後、N+1をreload→select→cook→Play→Save／Load→Replayする。各crash window、Closure欠落／差替え、semantic hashと完成object SHAの混同、署名前public、alternate signature、二重publicationをrejectする。続けてBindingと無関係なDocumentだけを編集したN+2を作り、binding semantic hashがN+1と同一のまま、N+2 Registry membershipとCompile closureのrevision／document set hashだけを更新して再compileできることを検証する。payloadへN+1／N+2 revisionを戻さずfixed pointを作らない。Project-defined logical target implementationはWorld、Physics、Perceptionを一切installせず、Ranged Combat ownerのexact Collision Query Port／Shot Hit Event type、Project-owned System、target dataだけでdeterministic Hit Evidenceを返す。Compile Manifestとentry closureが`selected_provider_binding_set_hash`を持ち、binding／template／System／Save Replay／target data hashのいずれかを変更するとpackageとReplay closureがinvalidateされることを検証する。

### 8.5 Negative fixture

- Shooter Packから別Genre Packへのdependencyを拒否する。
- Feature CapabilityのSchema／State ownerをShooter側で再宣言したrecipeを拒否する。
- missing／incompatible `PerceptionProfileV1`、unknown lost-target value、unbounded fire-intent継続を拒否する。
- Shooter role文字列をInput Action identityまたはSave identityにしたProjectを拒否する。
- Shooter PackなしのFeature-only Projectをvalidとする。
- Shooter Pack removal後もCore／Editor／AI／Project source／Build／Packageを成功させる。
- minimal Recipeへ未宣言のPerception／Stage／Score／Locomotionをclosure resolverが追加する実装を拒否する。
- fixture-only bindingのProduction選択、cross-project Document index containment、payload project ID self-assert spoof、current Registry membership欠落、Operation inputのstale base revision／document set hash、Binding identity三者不一致、payloadへのProject revision／document set hash混入、template／implementation／Save Replay hash mismatchに加え、implementation両branch／両方欠落、usageとbranch不一致、fixture owner mismatch、stale Fixture Registry／record／Qualification subject／signed Receipt／Activation Binding hash、Fixture System refのProduction Source／Registry／Save／Replay／Package混入を各一原因で拒否する。
- 既存2D／TPS Recipeから初期Shooter baselineのFeature closureまたはPerception bindingが欠落した場合、qualificationを拒否する。

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

`axis_field_ref`は下記C1軸のFieldだけを参照でき、それ以外をvalidatorが拒否する。`multiply_q16`の倍率は[Gameplay Features §3.3](../08-packs/gameplay-features.md)と同域のunsigned Q16 `[0.0625,16]`とする。適用結果は`clamp_min`／`clamp_max`で有限に固定し、NaN／Infを生成しない。`gameplay_fidelity_floor_ref`はDifficultyが下回れないGameplay fidelity(Collision有効、Hit Event非drop、authoritative objectの非削除、Replay決定性)の宣言を指す。

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

### 12.4 Future AI authoring vocabulary（not activated）

この節の九actionは要求分類用のfuture vocabularyであり、MCD `operation` identityではない。従来のname-only `operation.shooter.*`表記は一度もactivateされておらず、current MCD、Pack Manifest、Service allowlist、Provider／MCP Catalogから除外し、legacy aliasとしても読まない。§4のtarget-provider Binding三Operationもtarget-complete candidateであって未materializeであり、current Shooter Operation setはexact `[]`である。

| Future vocabulary token | 想定Risk | 想定結果 |
|---|---:|---|
| `future.shooter.action.search_catalog` | R0 | Concept、Capability、Definition、Qualification候補 |
| `future.shooter.action.resolve_intent` | R0 | `ShooterIntentResolutionV1` |
| `future.shooter.action.explain_resolution` | R0 | Field、理由、Assumption、代替、Evidence |
| `future.shooter.action.preview_simulation` | R0 | Headless／visual Preview、Cost、Event trace |
| `future.shooter.action.compose_recipe` | R2 | Feature ref、Profile、Action role、統合Qualification binding |
| `future.shooter.action.configure_game_flow` | R1 | Ready／Pause／Result／Restart差分 |
| `future.shooter.action.configure_difficulty` | R2 | 許可軸、fidelity floor、capacity差分 |
| `future.shooter.action.apply_profile` | R2 | Pack／Input／UI／Asset closure |
| `future.shooter.action.propose_native_variant` | R3 | Native Source、Test、capacity evidence、Promotion案 |

Capability stateは九actionすべて`not_activated`で、要求は`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`としてSource不変で拒否する。future work item `activation.shooter.ai_authoring.v1`は採用するexact Operation set、MCD全Field、named input／result、Service／Policy／Validator／Diagnostic／Receipt、Risk、authorization intent DAG、private-to-public recovery、Qualificationを同じContract set transactionで完全登録するまでactivateしない。Weapon、Fire Mode、Shot、Damage、Vital、Pickup、Encounter、Scoreその他のFeature Source mutationも、Feature authoring Capabilityが別途完全登録されるまで非活性であり、ShooterがFeature contractのFieldを直接writeしない。

AI ProviderへC++ pointer、Runtime handle、unbounded projectile list、raw Physics objectを渡さない。Search／Readは必要なCatalog entryとbounded Snapshotだけを返す。

## 13. Editor／AI UX

Shooter Workspaceは専用別Editorではなく、既存Scene、Outliner、Inspector、Graph、Table、AI Partnerへ次のProjectionを追加する。

- Weapon Inspector: cadence、ammo、reload、Delivery、Damage、Cue
- Shot Pattern Preview: origin、direction、Simulation Advance、capacity、Hit candidate
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
peak_fire_request_per_simulation_advance
peak_shot_activation_per_simulation_advance
peak_projectile_spawn_per_simulation_advance
peak_live_authoritative_projectile
peak_hitscan_query_per_simulation_advance
peak_shot_hit_per_simulation_advance
peak_damage_event_per_simulation_advance
peak_score_event_per_simulation_advance
peak_live_pickup
peak_pickup_collection_per_simulation_advance
peak_enemy_and_ally
peak_simultaneous_presentation_cue
```

値はGame BriefのScale intentと一致させる。Runtime Compilerが合格しやすい値へ丸めない。

### 14.2 C1組込み最低Fixture

| Profile | active combat actor | live projectile | projectile spawn／Simulation Advance | hitscan query／Simulation Advance | 目標 |
|---|---:|---:|---:|---:|---|
| `profile.shooter.top_down_2d` | 256 | 2,048 | 256 | 128 | 1080p60、authoritative drop 0 |
| `profile.shooter.third_person_3d` | 50 | 256 | 64 | 128 | 1080p60、authoritative drop 0 |

この個数はProduct上限ではない。Project intentが上回る場合は[Performance Scale Catalog Proposalが定義する`IntegratedScaleFixtureV1`候補](performance-scale-catalog-proposal.md#13-integrated-fixtureとqualification)をProject固有Envelopeから生成する。

Shooter Profileは上表の規模値とRecipe bindingだけを所有する。Weapon／Projectile／Damage／Pickup／Event queueのreservation、pool、capacity failure、rollback、Diagnosticは[Gameplay Feature Packs §4～8](../08-packs/gameplay-features.md#4-game-systemとstate-owner)と[Performance／Capacity](../04-runtime/performance-capacity.md)が所有し、本書は挙動または失敗型を再定義しない。

### 14.3 Genre integrated-scale fixture

| Destination fixture | Profile scale input | 正式な検証 |
|---|---|---|
| `fixture.genre.shooter.integrated-scale.top-down-2d` | `profile.shooter.top_down_2d`の§14.1／14.2 exact値 | Genre composition、Profile、`scope.genre.shooter.game_flow.instance`、Perception bindingをFeature owner receiptと統合 |
| `fixture.genre.shooter.integrated-scale.third-person-3d` | `profile.shooter.third_person_3d`の§14.1／14.2 exact値 | Genre composition、Profile、`scope.genre.shooter.game_flow.instance`、Perception bindingをFeature owner receiptと統合 |

両fixtureは[Performance Scale Catalog Proposalが定義する`IntegratedScaleFixtureV1`候補](performance-scale-catalog-proposal.md#13-integrated-fixtureとqualification)へ上表の規模値を入力し、Feature failure semanticsを複製しない。宣言値不足、非finite、負値、required axis欠落はGenre-owned `MIRAKAN-GENRE-SHOOTER-PROFILE-SCALE-UNDERSPECIFIED`でProfile applyを拒否し、last-valid Profile／Recipe／fixture receiptを維持する。

### 14.4 Genre-owned diagnostic

| Diagnostic ID | code | Owner | 条件 | 結果 |
|---|---|---|---|---|
| `diagnostic.genre.shooter.profile.scale_underspecified` | `MIRAKAN-GENRE-SHOOTER-PROFILE-SCALE-UNDERSPECIFIED` | `genre.shooter` | §14.1 required axis欠落、非finite／負値、§14.2 Qualification input未解決 | Profile／Recipe applyを拒否しlast-valid Genre receiptを維持 |

このrowは`diagnostic_version=1`とself-excluding `diagnostic_content_hash`を持つexact `DiagnosticCodeRefV1`である。Feature Definition、State owner、Fire transaction、Projectile／query／queue capacity、Damage、Pickup、Score、Feature Save／Replay、Feature presentation authorityのDiagnosticは本表へ追加せず、[Gameplay Feature Packs §7](../08-packs/gameplay-features.md#7-failureとdiagnostic)を参照する。

## 15. Qualification closure

Shooter Genre PackのQualification owner範囲はcomposition recipe、Profile、Game Flow、Action role、`ShooterPerceptionBindingV1`、Shooter統合fixtureに限定する。Feature schema、closed enum、境界値、Fire transaction、canonical ordering、State owner、Save／Replay、failure、Feature contract fixtureの合否は各Feature ownerが発行したQualification Receiptをconsumeし、Shooter側で再所有または再判定しない。

Shooter側は2D top-down／third-person 3Dの両fixtureで、required receipt closure、Perception target selection、lost-target behavior、fire-intent policy、Game Flow、Action role、Profile bindingが接続できることだけを検証する。Packとしてのinstall／apply／update／remove、Project ChangeSet、qualification receiptは[Pack Contract](../08-packs/pack-contract.md)を使い、Capability成熟度と実装順序は[Product Plan](../00-product/product-plan.md)へ委譲する。Profile固有のCamera、Input、UI、Audio、VFX、Asset templateは各ownerのschemaを参照し、同じFieldまたは固定値を本Packへ再定義しない。

## 16. Feature identity migration

旧Genre由来のFeature identityはaliasとして残さず、Feature ownerのmigration stepで次のexact identityへclean renameする。旧Save／Replay／fixture参照はmigration receiptへ記録し、AI catalogは旧名からGenre dependencyを推論しない。

| Legacy source identity | Exact destination identity | Migration owner |
|---|---|---|
| `ShooterProjectileStateV1` | `RangedProjectileStateV1` | `feature.ranged_combat` |
| `SpawnShooterProjectileCommandV1` | `SpawnRangedProjectileCommandV1` | `feature.ranged_combat` |
| `ShooterProjectileSnapshotV1` | `RangedProjectileSnapshotV1` | `feature.ranged_combat` |
| `fixture.shooter.even-floor` | `fixture.feature.ranged_combat.even_floor` | `feature.ranged_combat` |
| `fixture.shooter.explicit-offsets` | `fixture.feature.ranged_combat.explicit_offsets` | `feature.ranged_combat` |
| `MIRAKAN-SHOOTER-DEFINITION_INVALID` | `MIRAKAN-FEATURE-DEFINITION_INVALID` | referenced Feature owner |
| `MIRAKAN-SHOOTER-CONTRACT_VERSION_MISMATCH` | `MIRAKAN-FEATURE-CONTRACT_VERSION_MISMATCH` | referenced Feature owner |
| `MIRAKAN-SHOOTER-CAPABILITY_UNAVAILABLE` | `MIRAKAN-FEATURE-CAPABILITY_UNAVAILABLE` | referenced Feature owner |
| `MIRAKAN-SHOOTER-STATE_OWNER_CONFLICT` | `MIRAKAN-FEATURE-STATE_OWNER_CONFLICT` | referenced Feature owner |
| `MIRAKAN-SHOOTER-FIRE_TRANSACTION_FAILED` | `MIRAKAN-RANGED-COMBAT-FIRE_TRANSACTION_FAILED` | `feature.ranged_combat` |
| `MIRAKAN-SHOOTER-PROJECTILE_CAPACITY_EXCEEDED` | `MIRAKAN-RANGED-COMBAT-PROJECTILE_CAPACITY_EXCEEDED` | `feature.ranged_combat` |
| `MIRAKAN-SHOOTER-QUERY_CAPACITY_EXCEEDED` | `MIRAKAN-RANGED-COMBAT-QUERY_CAPACITY_EXCEEDED` | `feature.ranged_combat` |
| `MIRAKAN-SHOOTER-AUTHORITATIVE_QUEUE_OVERFLOW` | `MIRAKAN-FEATURE-AUTHORITATIVE_QUEUE_OVERFLOW` | referenced Feature owner |
| `MIRAKAN-SHOOTER-DAMAGE_TARGET_INVALID` | `MIRAKAN-COMBAT-DAMAGE_TARGET_INVALID` | `feature.combat` |
| `MIRAKAN-SHOOTER-PICKUP_GRANT_FAILED` | `MIRAKAN-PICKUP-GRANT-FAILED` | `feature.pickup_grant` |
| `MIRAKAN-SHOOTER-SCORE_OVERFLOW` | `MIRAKAN-SCORING-OVERFLOW` | `feature.scoring` |
| `MIRAKAN-SHOOTER-SAVE_CONTRACT_MISMATCH` | `MIRAKAN-FEATURE-SAVE_CONTRACT_MISMATCH` | referenced Feature owner |
| `MIRAKAN-SHOOTER-REPLAY_DIVERGENCE` | `MIRAKAN-FEATURE-REPLAY_DIVERGENCE` | referenced Feature owner |
| `MIRAKAN-SHOOTER-PRESENTATION_AUTHORITY_VIOLATION` | `MIRAKAN-FEATURE-PRESENTATION_AUTHORITY_VIOLATION` | referenced Feature owner |
| `MIRAKAN-SHOOTER-PROFILE_SCALE_UNDERSPECIFIED` | `MIRAKAN-GENRE-SHOOTER-PROFILE-SCALE-UNDERSPECIFIED` | `genre.shooter` |
| `fixture.shooter.crowded-battle-2d` | `fixture.genre.shooter.integrated-scale.top-down-2d` | `genre.shooter` |
| `fixture.shooter.crowded-battle-3d` | `fixture.genre.shooter.integrated-scale.third-person-3d` | `genre.shooter` |

上記source identityはこのmigration表にexactly one rowだけ存在し、alias／wildcard／prefix fallbackで照合しない。migration stepはsource identityとdestination identityの双方を完全一致で記録し、unknown legacy identityを最も近いdestinationへ推測変換しない。
