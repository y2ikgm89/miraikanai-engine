# Miraikanai Engine Scenario／Stage Feature Pack

- 文書ID: mirakan.arch.pack-scenario-stage
- 状態: review
- 正本範囲: optional Scenario／Stage Feature、`StageDefinitionV1`、Stage Scope、completion tagged rule、transition、Save／Replay、AI Operation、fixture
- 非正本範囲: World／Scene／Cell、Runtime scheduling、Game Flow、Combat／Encounter、Save storage、Replay transport、Product roadmapは各Ownerを参照
- 依存: [Pack Contract](pack-contract.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Project State](../03-authoring/project-state.md)、[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Debugging／Replay](../04-runtime/debugging-observability-replay.md)、[World](../06-rendering/world.md)
- 外部根拠検証日: 2026-07-23

## 1. 結論

有限Stage、entry／exit、Objective、Completionが必要なProjectだけが`feature.scenario_stage`を使用する。endless、continuous simulation、tool-like、UI-only、headless等のProjectはこのFeature Packを使用しなくてもvalidである。

Scenario／StageはGeneric Engine CoreのWorldまたはRuntimeへLevel契約を追加しない。Worldは空間、Scene、global composition、persistent entity、任意のspatial topologyだけを所有し、Stageは既存Worldを参照してGameplay上の有限区間を構成する。

本Packは`pack_kind=feature`で、`capability.gameplay.scenario_stage`を提供する。

## 2. `StageDefinitionV1`

```text
StageDefinitionV1
  stage_id
  world_ref
  content_source_refs[]
  entry_anchor_refs[]
  exit_anchor_refs[]
  stage_game_system_refs[]
  objective_definition_refs[]
  spawn_definition_refs[]
  transition_policy_refs[]
  completion_mode: none | explicit_outcomes
  completion_contract: CompletionContractV1 | null
  save_replay_policy_ref
  fallback_contract
```

`stage_id`は表示名、World path、配列indexではないstable identityである。`world_ref`は[World](../06-rendering/world.md)の既存Worldを参照し、World schemaを本書で再定義しない。content、anchor、Game System、Objective、Spawn、transitionは各Ownerのexact refであり、Stageが参照先SchemaやState ownerを複写しない。

## 3. Completion tagged rule

`completion_mode`をdiscriminatorとし、次を機械検査する。

| `completion_mode` | `completion_contract` | Objective／Resultの扱い |
|---|---|---|
| `none` | `null`のみ | `objective_definition_refs[]`は0件を許可し、ObjectiveまたはResultを要求しない |
| `explicit_outcomes` | `CompletionContractV1`を必須 | contractがoutcome ID、成立条件、result projectionをexact refで宣言 |

`none`でnon-null contractを持つ、Result routeを必須にする、空Objectiveをvalidation failureにする実装を拒否する。`explicit_outcomes`でcontractがnull、unknown outcome、重複outcome、成立条件がOwner不明の実装を拒否する。

CompletionはWorld activation、Scene activation、Cell streamingの前提ではない。Stage終了をWorld unloadと同一視せず、transition policyが次のStage、別World、UI、headless continuation、session終了のいずれかをtyped destinationとして選ぶ。

## 4. Stage Scope

Feature Packは`scope.feature.scenario_stage.instance`をversioned Runtime Scope Type Catalogへ登録する。Coreは`level_instance`をclosed enumへ追加しない。

Scope entryは次を必須にする。

- instance key schema
- owner
- lifetime
- Save／Replay policy
- activation condition
- deactivation condition

Stage instanceは同じStage definitionから複数生成でき、Stateはinstance keyで分離する。Stage Game System、Objective、Spawn、transitionのState ownerはScope entryと各Game System Specが宣言し、World、UI、Shooter Game Flowが暗黙所有しない。

## 5. Transition

`transition_policy_refs[]`はtyped transition policyへの参照であり、World／Scene／Cellのtransition payloadを再定義しない。policyはsource Stage instance、trigger／outcome、destination kind、destination ref、transfer subject refs、failure fallbackを解決する。

destinationはProjectのRuntime entryとWorld／UI／headless契約へ従う。Player／Party／Characterを固定payloadにせず、transfer対象は参照先のtyped subject contractを使う。遷移準備またはactivationが失敗した場合は`fallback_contract`に従い、source Stageのlast-valid stateを破棄しない。

## 6. Save／Replay

`save_replay_policy_ref`は次を宣言する。

- Stage definition／instance identityとversion
- 保存するStage-owned field
- Objective／Spawn／transition ownerへのprojection ref
- checkpointとresume条件
- Replay input／event／snapshot ref
- migrationと不一致時fallback

Saveは登録済みState ownerとSave／Replay契約が宣言したfieldだけを保存する。World内容、Feature State、Shooter Game FlowをStage所有と推測して複写しない。Load／Replay時にStage definition、World、Feature Pack、policy hashが不一致ならmigrationまたはtyped failureを返し、似たStageへ暗黙fallbackしない。

## 7. AI Operation

MCDへ次のexact Operationを登録する。

- `operation.scenario_stage.create`
- `operation.scenario_stage.read`
- `operation.scenario_stage.plan_update`
- `operation.scenario_stage.preview_update`
- `operation.scenario_stage.validate`
- `operation.scenario_stage.explain_transition`

create／updateは[Project State](../03-authoring/project-state.md)のValidation、Staging、Approval、Receiptを通る。AIは有限Gameという自然言語だけでCompletion outcomeを捏造せず、`completion_mode`、outcome、destination、Save／Replayへの影響が未確定なら質問またはAssumptionとしてPreviewする。

AI contextはselected Stage、World、参照Feature、Scope、transition、Save／Replay policy、Diagnosticだけをsemantic projectionし、全Projectまたは全Schemaを無制限に送信しない。

## 8. Fixture

最低fixtureは次を含む。

- positive: `completion_mode=none`、`completion_contract=null`、Objective 0件、Result routeなしのendless Stageがvalid
- positive: `completion_mode=explicit_outcomes`と有効なoutcome contractを持つfinite Stageがvalid
- negative: `completion_mode=none`でnon-null completion contractを持つStageをreject
- negative: `completion_mode=none`でObjectiveまたはResultを必須とするvalidatorをreject
- negative: `completion_mode=explicit_outcomes`でcompletion contractがnullのStageをreject
- Stage Scopeのinstance分離、activation／deactivation、Save／Load、Replay round-trip
- transition成功、destination activation失敗、fallback、last-valid source維持
- World／Runtimeへ`level_instance`またはCompletion requirementを追加しないdependency検査
- Scenario／Stage Packなしのendless、UI-only、headless Projectがvalid

## 9. Feature manifestとPublic port

| `PackManifestV1` field | Canonical value |
|---|---|
| `pack_id` | `feature.scenario_stage` |
| `pack_kind` | `feature` |
| `required_feature_pack_refs[]` | `[]` |
| `provided_capability_refs[]` | `capability.gameplay.scenario_stage` |
| `public_contract_refs[]` | `StageDefinitionV1; CompletionContractV1; StageRuntimeStateV1` |
| `runtime_port_refs[]` | `StageActivationPortV1; StageTransitionPortV1` |
| `configuration_profile_refs[]` | `StageContentActivationPolicyV1; StageSaveReplayPolicyV1` |
| `test_scenario_refs[]` | `fixture.feature.scenario_stage.none; fixture.feature.scenario_stage.explicit_outcomes; fixture.feature.scenario_stage.transition` |

`StageActivationPortV1`はStage instance key、source Stage ref、World ref、content activation policy、requested entry anchor、requested Feature System refsを入力し、activation generationまたはtyped failureを返す。World／Scene／Cell handleをPublic APIへ出さない。

`StageTransitionPortV1`はsource Stage instance、trigger／outcome、destination、typed transfer subject refs、transition policyを入力とする。Player、Party、Characterを固定型にせず、registered subject contractへ適合する任意のsubjectを扱える。

## 10. Content activationとRuntime state

```text
StageContentActivationPolicyV1
  policy_id
  content_activation_scope: entry_anchor_closure | listed_content_refs
  required_content_refs[]
  prefetch_priority_class_ref
  fallback_contract

StageRuntimeStateV1
  stage_ref
  stage_instance_key
  lifecycle_state: inactive | preparing | ready | activating | active | deactivating | faulted
  active_entry_ref
  activated_game_system_instance_refs[]
  completion_state_ref: null | CompletionStateRef
  activation_generation
```

`listed_content_refs`時だけ`required_content_refs[]`を1件以上持ち、`entry_anchor_closure`時は0件とする。World／Scene／Cellのactivation計画はWorld／Runtime ownerのPublic Portを消費し、Stage側がCell schemaやphase ABIを再定義しない。

`completion_mode=none`では`completion_state_ref=null`を許可し、`active`からProject／Runtime要求で直接`deactivating`へ進める。Objective、completion outcome、Result route、`completing` stateを要求しない。`explicit_outcomes`だけが`CompletionContractV1`に基づくcompletion stateを作る。

同じStage Sourceから複数instanceを作れる。`scope.feature.scenario_stage.instance`のinstance keyでStateを分離し、Source Stable ID、Save identity、ephemeral Runtime generationを相互に置換しない。

## 11. Stage transition contract

```text
StageTransitionPolicyV1
  policy_id
  destination_kind: stage | runtime_entry | world_space | ui | headless | session_end
  destination_ref
  presentation_policy_ref
  persistent_subject_policy_ref
  precondition_ref
  failure_policy
  cancel_policy

StageTransitionRequestV1
  request_id
  source_stage_instance_ref
  trigger_or_outcome_ref
  target_ref
  target_entry_anchor_ref
  requesting_system_ref
  requested_tick
  transfer_subject_refs[]
  precondition_snapshot_hash
  transition_policy_ref
```

`transfer_subject_refs[]`はregistered typed subjectだけを受理し、display roleやowner推測でPlayer／Partyへ変換しない。destinationがWorld Spaceの場合は[World](../06-rendering/world.md)のgeneric spatial transition port、別Stageの場合は`StageActivationPortV1`を使う。

target dependency不足、stale precondition、unknown destination、subject incompatibilityではpartial activationせず、source Stageとlast-valid World generationを維持する。Loading progress／cancel／retryの表示契約はWorld／Runtimeのgeneric Loading projectionを参照し、Stage outcomeをLoading UIから推測しない。

## 12. Save identityとmigration

`StageSaveReplayPolicyV1`はStage Source identity、definition version、Stage save instance identity、Scope key、Stage-owned State fields、Objective／Spawn／transition owner projection、checkpoint／resume、Replay oracle、migration、fallbackを宣言する。

同じSource Stageから複数saved instanceを作る場合も、各instanceへ別のUUIDv7 save identityを発行する。Loadは新しいephemeral runtime generationへone-to-oneでremapし、保存前のhandle／generationを復元しない。`completion_mode=none`ではcompletion fieldやResultをSave recordへ合成しない。

旧World Level契約からのclean migrationは次の一方向とし、World側へaliasを残さない。

| 旧World identity | Scenario／Stage identity |
|---|---|
| `LevelDefinitionV1` | `StageDefinitionV1` |
| `LevelStreamingPolicyV1` | `StageContentActivationPolicyV1` |
| `LevelRuntimeStateV1` | `StageRuntimeStateV1` |
| `LevelTransitionPolicyV1` | `StageTransitionPolicyV1` |
| `LevelTransitionRequestV1` | `StageTransitionRequestV1` |
| `level_gameplay` owner role | owner明示の`stage_game_system_refs[]`とStage Scope |
| `player_or_party_transfer_refs[]` | typed `transfer_subject_refs[]` |
| required `completion_contract` | tagged `completion_mode`＋nullable `completion_contract` |

MigrationはSource ref、Save identity、active entry、Stage-owned State、transition precondition、Replay oracleをPreviewし、Stage ownerのValidationとApproval後にcommitする。旧World fieldをcompatibility aliasとして読まない。

## 13. Diagnosticとqualification

| Diagnostic ID | 条件 | 結果 |
|---|---|---|
| `MIRAKAN-SCENARIO-STAGE-DEFINITION_INVALID` | tagged branch、ref、Scope不正 | Source／Cookを拒否 |
| `MIRAKAN-SCENARIO-STAGE-ACTIVATION_FAILED` | content／Feature closure不成立 | source／last-valid generation維持 |
| `MIRAKAN-SCENARIO-STAGE-TRANSITION_FAILED` | destination／subject／precondition不正 | partial transition禁止 |
| `MIRAKAN-SCENARIO-STAGE-SAVE_CONTRACT_MISMATCH` | Source／Scope／State owner不一致 | Saveを開かずbackup維持 |
| `MIRAKAN-SCENARIO-STAGE-REPLAY_DIVERGENCE` | first Stage-owned divergence | 停止してReproduction Bundle生成 |

Qualificationは`completion_mode=none`と`explicit_outcomes`、複数Stage instance、content activation exact／exact+1、transition success／rollback、typed subject compatibility、Save／Load／Replay、旧Level Source migration、Scenario／Stage Pack removalを含む。Feature PackなしのWorld／endless Projectは引き続きvalidである。
