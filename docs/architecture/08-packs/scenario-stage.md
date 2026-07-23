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
