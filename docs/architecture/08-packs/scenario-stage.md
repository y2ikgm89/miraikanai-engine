# Miraikanai Engine Scenario／Stage Feature Pack

- 文書ID: mirakan.arch.pack-scenario-stage
- 状態: review
- 正本範囲: optional Scenario／Stage Feature、`StageDefinitionV1`、Stage Scope、completion tagged rule、transition、Save／Replay、AI Operation、fixture
- 非正本範囲: World／Scene／Cell、Runtime scheduling、Game Flow、Combat／Encounter、Save storage、Replay transport、Product roadmapは各Ownerを参照
- 依存: [Pack Contract](pack-contract.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Project State](../03-authoring/project-state.md)、[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Debugging／Replay](../04-runtime/debugging-observability-replay.md)、[World](../06-rendering/world.md)
- 外部根拠検証日: 2026-07-23

## 1. 結論

有限Stage、entry／exit、Objective、Completionが必要なProjectだけが`feature.scenario_stage`を使用する。endless、continuous simulation、tool-like、UI-only、headless等のProjectはこのFeature Packを使用しなくてもvalidである。

Scenario／StageはGeneric Engine CoreのWorldまたはRuntimeへLevel契約を追加しない。Worldは空間、Scene、global composition、persistent entity、任意のspatial topologyだけを所有する。Stageは必要な場合だけ既存Worldを参照し、UI-only／headless Stageは偽Worldなしに有限workflowを構成する。

本Packは`pack_kind=feature`で、`capability.gameplay.scenario_stage`を提供する。

## 2. `StageDefinitionV1`

```text
StageDefinitionV1
  stage_id
  world_ref: WorldDocumentRef | null
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

`stage_id`は表示名、World path、配列indexではないstable identityである。`world_ref`はrequired nullable fieldであり、[World](../06-rendering/world.md)の既存Worldを参照するかexact `null`を持つ。World schemaを本書で再定義しない。`content_source_refs[]`はowner-typed exact refで、content、anchor、Game System、Objective、Spawn、transitionは各OwnerのSchemaやState ownerを複写しない。`spawn_definition_refs[]`はWorld空間へ配置するspatial spawn definitionだけを指し、非空間workflowのtask／content生成を兼用しない。

branch validationは次へ固定する。

- world Runtime Entryへ結ぶStageはnon-null exact `world_ref`を必須とし、entry／exit anchorとspatial spawn fieldを参照Worldのkind／revisionへ照合する。
- UI-only／headless Stageは`world_ref=null`、`entry_anchor_refs=[]`、`exit_anchor_refs=[]`、`spawn_definition_refs=[]`とする。非空間workflowはowner-typed `content_source_refs[]`と`stage_game_system_refs[]`、Objective／transition／Save／Replayを利用する。
- null Worldをdefault Worldへ補完する、anchorやspatial spawnを表示名で再解決する、UI contentを偽Sceneへ変換する実装を拒否する。

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

Stage lifecycleを所有するSystem IDは`game_system.extension.feature.scenario_stage`であり、`GameSystemSpecV2.owner_layer=feature_pack`、exact `owner.feature.scenario_stage` ref／revision／content hash、`system_origin=owner_package`を必須にする。Core namespaceまたはGenre／Project ownerを自己申告するSpecを拒否する。

Scope entryは`RuntimeScopeTypeCatalogV1`へ次のexact 7-Field rowで登録する。

| `scope_type_ref` | `instance_key_schema_ref` | `owner_ref` | `lifetime_ref` | `save_replay_policy_ref` | `activation_condition_ref` | `deactivation_condition_ref` |
|---|---|---|---|---|---|---|
| `scope.feature.scenario_stage.instance` | `type.runtime_scope.key.feature_scenario_stage_uuidv7` | `owner.feature.scenario_stage` | `policy.runtime_scope.lifetime.feature_scenario_stage_instance` | `policy.runtime_scope.save_replay.feature_scenario_stage` | `policy.runtime_scope.activation.feature_scenario_stage_request_ready` | `policy.runtime_scope.deactivation.feature_scenario_stage_transition_stop_or_fault` |

保存値は`RuntimeScopeTypeRefV1`、`McdContractRefV1`、`RuntimeScopeOwnerRefV1`のversion／hash付きtyped refであり、表のIDだけを永続化しない。全dependencyをactive Runtime Scope Registryへ実体recordとして登録する。Stage instanceは同じStage definitionから複数生成でき、Stateはinstance keyで分離する。Stage Game System、Objective、Spawn、transitionのState ownerはScope entryと各Game System Specが宣言し、World、UI、Shooter Game Flowが暗黙所有しない。

旧Level Systemのscope migrationは本Packが`RuntimeScopeMigrationContributionRegistryV1`へ`runtime_scope.migration_contribution.feature.scenario_stage`として登録する。Production contributionは`owner.feature.scenario_stage`、exact legacy Level System ref／hash、source `type.game_system.spec` version 1、destination version 2、legacy `level_instance`、destination `scope.feature.scenario_stage.instance`、Stage-owned auxiliary／identity migration policyを持つReceipt-free recordである。Registry／ContributionRef固定後に`qualification.runtime_scope_migration.feature.scenario_stage@1` subject／signed Receiptと`qualification_binding.runtime_scope_migration.feature.scenario_stage@1`をroot外で作る。Fixture bodyはsubjectだけが`fixture.feature.scenario_stage.runtime_scope_migration`を解決する。Core migratorはgeneric record／Binding解決だけを行い、Level／Stage ID、Qualification subject／Fixture、adapterをCoreへhard-codeしない。

## 5. Transition

`transition_policy_refs[]`は§11のexact `StageTransitionPolicyV1` ref／hashであり、World／Scene／Cellのtransition payloadを再定義しない。Requestはpolicy ref／hashだけを指定し、destinationを複製しない。destinationのtagged union、Runtime Entry経由、World-owned spatial type、typed subject、failure rollbackは§11だけを正本とする。Player／Party／Characterを固定payloadにしない。

## 6. Save／Replay

`save_replay_policy_ref`は次を宣言する。

- Stage definition／instance identityとversion
- 保存するStage-owned field
- Objective／Spawn／transition ownerへのprojection ref
- checkpointとresume条件
- Replay input／event／snapshot ref
- migrationと不一致時fallback

Saveは登録済みState ownerとSave／Replay契約が宣言したfieldだけを保存する。World内容、Feature State、Shooter Game FlowをStage所有と推測して複写しない。Load／Replay時にStage definition、Feature Pack、policy hashと、non-nullの場合だけWorld hashが不一致ならmigrationまたはtyped failureを返し、似たStageや偽Worldへ暗黙fallbackしない。

## 7. AI Operation

Scenario／Stage authoring surfaceは本Taskで完全登録されていないため、current Operation setを空に固定する。

```text
Current Scenario/Stage authoring Operation set = {}
Capability state = not_activated
```

従来記載された六件の候補setとManifestの七件setはどちらも一度もactivateされておらず、current MCD、Pack Manifest、Service allowlist、Provider／MCP Catalog、Policy／Validator／Diagnostic／Receipt inventoryから全件除外する。legacy aliasとしても読まない。authoring要求は`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`で拒否し、Stage／Project Source、revision、last-valid artifactを変更しない。

future work item `activation.scenario_stage.authoring.v1`は採用するexact Operation setを一つに固定し、read-onlyとmutationを含む各OperationのMCD全Field、initial create/upsertとupdate、named input／result、semantic intent hash、`MutationAuthorizationBindingV2`、Service allowlist、Risk／side effect／idempotency／transaction、pure pre／post Policy、closed Diagnostic、Validator closure、rate／timeout、canonical signed Receipt、private commit→signed wrapper→public publication recovery、positive／negative Qualificationを同じContract set transactionで完全登録するまでactivateしない。将来AI contextを有効化する場合もselected Stage、World、参照Feature、Scope、transition、Save／Replay policy、Diagnosticだけをsemantic projectionし、全Projectまたは全Schemaを無制限に送信しない。

## 8. Fixture

最低fixtureは次を含む。

- positive: `completion_mode=none`、`completion_contract=null`、Objective 0件、Result routeなしのendless Stageがvalid
- positive: `completion_mode=explicit_outcomes`と有効なoutcome contractを持つfinite Stageがvalid
- positive: `world_ref=null`のfinite Dialogue／Visual Novel／UI workflowがowner-typed UI content、Stage transition、Save／Load／Replayを利用し、World／Scene／Topology／spatial anchorを生成しない
- positive: `world_ref=null`のheadless workflowがstartup Stage systemsとtyped transitionだけでvalid
- positive: `fixture.feature.scenario_stage.worldless-ui`が`ui` destination、anchor null、World／spatial spawn 0件でround-tripする
- positive: `fixture.feature.scenario_stage.worldless-headless`が`content_activation_scope=none`、system-only、headless destinationでround-tripする
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
| `public_contract_refs[]` | version／hash付き`StageDefinitionV1; CompletionContractV1; StageRuntimeStateV1; StageTransitionDestinationV2; StageTransitionPolicyV1; StageTransitionRequestV2; StageTransitionContractRefSetV1` |
| `runtime_port_refs[]` | version／hash付き`StageActivationPortV1; StageTransitionPortV2` |
| `configuration_profile_refs[]` | `StageContentActivationPolicyV1; StageSaveReplayPolicyV1` |
| `authoring_operation_refs[]` | `[]`。Scenario／Stage authoring Capabilityは`not_activated` |
| `validator_refs[]` | version／content hash付き`validator.feature.scenario_stage.v1` |
| `migration_contribution_refs[]` | version／content hash付き`runtime_scope.migration_contribution.feature.scenario_stage` |
| `migration_step_refs[]` | version／content hash付き`migration.feature.scenario_stage.level_contract_to_stage_v1` |
| `test_scenario_refs[]` | version／content hash付き`fixture.feature.scenario_stage.none; fixture.feature.scenario_stage.explicit_outcomes; fixture.feature.scenario_stage.transition; fixture.feature.scenario_stage.worldless-ui; fixture.feature.scenario_stage.worldless-headless; fixture.feature.scenario_stage.aggregate-manifest-set-equality; fixture.feature.scenario_stage.runtime_scope_migration` |

`StageActivationPortV1`はStage instance key、source Stage ref、required nullable World ref、content activation policy、optional requested entry anchor、requested Feature System refsを入力し、activation generationまたはtyped failureを返す。World／Scene／Cell handleをPublic APIへ出さない。

`StageTransitionPortV2`はsource Stage instance、trigger／outcome、exact transition policy ref／hash、typed transfer subject refsを入力とする。destinationはPolicyから解決し、Port messageへ複製しない。Player、Party、Characterを固定型にせず、registered subject contractへ適合する任意のsubjectを扱える。旧V1のinline destinationはoffline migration inputだけで、current Port inventoryへ登録しない。

```text
StageTransitionContractRefSetV1
  ref_set_version: 1
  ref_set_hash
  destination_type_ref:
    McdContractRefV1(id=type.feature.scenario_stage.transition_destination, version=2, contract_set_hash)
  request_type_ref:
    McdContractRefV1(id=type.feature.scenario_stage.transition_request, version=2, contract_set_hash)
  policy_type_ref:
    McdContractRefV1(id=type.feature.scenario_stage.transition_policy, version=1, contract_set_hash)
  spatial_destination_type_ref:
    McdContractRefV1(id=type.world.spatial_transition_destination, version=1, contract_set_hash)
  transition_port_type_ref:
    McdContractRefV1(id=type.feature.scenario_stage.transition_port_message, version=2, contract_set_hash)
```

`StageTransitionContractRefSetV1`は上記exact 5 Fieldだけを持ち、generic `refs[]`、optional ref、六件目を許可しない。`ref_set_hash`はASCII `MIRAKAN_STAGE_TRANSITION_CONTRACT_REF_SET_V1`、`ref_set_version`、exact 5 refを上記Field順に`uint32_be` length framingしたMCD canonical bytesから計算し、自己Fieldを除外する。duplicate ID、wrong kind／version／Contract set hash、同じIDの別hash、非canonical Fieldを拒否する。

左辺の到達性を単一ownerの名前検索にしない。まずReceiptを一切含まないcandidateを各Owner rootから決定的にmaterializeし、そのimmutable subjectへReceiptを発行してからfinal closureを構成する。

```text
StageTransitionCrossOwnerCandidateV1
  candidate_version: 1
  contract_set_hash: SHA-256
  pack_ref: exact PackContractRefV1
  stage_owner_root_ref: exact {owner_id, owner_revision, owner_content_hash}
  world_owner_root_ref: exact {owner_id, owner_revision, owner_content_hash}
  runtime_owner_root_ref: exact {owner_id, owner_revision, owner_content_hash}
  owner_subsets:
    stage:
      destination_type_ref
      request_type_ref
      policy_type_ref
      transition_port_type_ref
    world:
      spatial_destination_type_ref
    runtime:
      boundary_delivery_contract_ref: exact BoundaryDeliveryContractRefV1
  exact_transition_ref_set: StageTransitionContractRefSetV1
  candidate_hash: SHA-256

StageTransitionOwnerValidationReceiptPayloadV1
  owner_kind: stage | world | runtime
  candidate_hash: SHA-256
  contract_set_hash: exact candidate.contract_set_hash
  validated_subset_hash: SHA-256
  result: pass

StageTransitionOwnerValidationReceiptV1
  payload: StageTransitionOwnerValidationReceiptPayloadV1
  signed_record:
    exact MirakanSignedRecordV1(
      purpose=scenario_stage_owner_validation,
      subject_sha256=SHA-256(JCS(payload)))

StageTransitionOwnerValidationReceiptRefV1
  owner_kind: stage | world | runtime
  candidate_hash: SHA-256
  result: pass
  payload_subject_sha256: exact SHA-256(JCS(resolved payload))
  signed_record_ref:
    exact MirakanSignedRecordRefV1(
      purpose=scenario_stage_owner_validation,
      subject_sha256=payload_subject_sha256)

StageTransitionPublicContractInventoryGatePayloadV1
  gate_id: gate.scenario_stage.public_contract_inventory
  candidate_hash: SHA-256
  manifest_public_contract_set_hash: SHA-256
  stage_owner_public_contract_set_hash: SHA-256
  result: pass

StageTransitionRuntimePortInventoryGatePayloadV1
  gate_id: gate.scenario_stage.runtime_port_inventory
  candidate_hash: SHA-256
  manifest_runtime_port_set_hash: SHA-256
  stage_owner_runtime_port_set_hash: SHA-256
  result: pass

StageTransitionMcdRefSetGatePayloadV1
  gate_id: gate.scenario_stage.transition_mcd_ref_set
  candidate_hash: SHA-256
  candidate_transition_ref_set_hash: SHA-256
  canonical_transition_ref_set_hash: SHA-256
  result: pass

StageTransitionPortClosureGatePayloadV1
  gate_id: gate.scenario_stage.transition_port_closure
  candidate_hash: SHA-256
  reachable_cross_owner_ref_set_hash: SHA-256
  canonical_transition_ref_set_hash: SHA-256
  result: pass

StageTransitionAggregateProjectionGatePayloadV1
  gate_id: gate.scenario_stage.aggregate_projection
  candidate_hash: SHA-256
  aggregate_public_and_runtime_inventory_hash: SHA-256
  manifest_public_and_runtime_inventory_hash: SHA-256
  result: pass

StageTransitionGateReceiptPayloadV1 =
  public_contract_inventory:
    StageTransitionPublicContractInventoryGatePayloadV1
  | runtime_port_inventory:
    StageTransitionRuntimePortInventoryGatePayloadV1
  | transition_mcd_ref_set:
    StageTransitionMcdRefSetGatePayloadV1
  | transition_port_closure:
    StageTransitionPortClosureGatePayloadV1
  | aggregate_projection:
    StageTransitionAggregateProjectionGatePayloadV1

StageTransitionGateReceiptV1
  payload: StageTransitionGateReceiptPayloadV1
  signed_record:
    exact MirakanSignedRecordV1(
      purpose=scenario_stage_gate_validation,
      subject_sha256=SHA-256(JCS(payload)))

StageTransitionGateReceiptRefV1
  gate_id: exact discriminator-selected closed gate ID
  candidate_hash: SHA-256
  result: pass
  payload_subject_sha256: exact SHA-256(JCS(resolved payload))
  signed_record_ref:
    exact MirakanSignedRecordRefV1(
      purpose=scenario_stage_gate_validation,
      subject_sha256=payload_subject_sha256)

StageTransitionCrossOwnerReachableClosureV1
  candidate: StageTransitionCrossOwnerCandidateV1
  candidate_hash: exact candidate.candidate_hash
  owner_validation_receipt_refs[3]:
    StageTransitionOwnerValidationReceiptRefV1
  gate_receipt_refs[5]: StageTransitionGateReceiptRefV1
  final_closure_hash: SHA-256
```

`candidate_hash`はASCII `MIRAKAN_STAGE_TRANSITION_CROSS_OWNER_CANDIDATE_V1`、candidate version、Contract set hash、Pack ref、三owner root、三owner subset、exact transition ref setをField順にMCD canonical encodeし、各segmentを`uint32_be` length framingしてSHA-256する。自己Fieldだけを除外し、Receipt ref、signature、final closure hashを入力にしない。三`validated_subset_hash`はowner kindから一意に選び、`stage = SHA-256(ASCII "MIRAKAN_STAGE_TRANSITION_OWNER_SUBSET_STAGE_V1" || uint32_be(len(MCD canonical candidate.owner_subsets.stage bytes)) || bytes)`、`world`と`runtime`も同じ式でdomainだけを各`MIRAKAN_STAGE_TRANSITION_OWNER_SUBSET_WORLD_V1`／`MIRAKAN_STAGE_TRANSITION_OWNER_SUBSET_RUNTIME_V1`、bytesを対応subsetへ置換した値とexact equalityにする。別owner subset、全candidate hash、表示上のref集合、任意hashを代用しない。

Runtime subsetはgeneric `BoundaryDeliveryContractV1`が`T00_BoundaryApply`で配送することを検証するが、exact五ref集合へ六件目として入れない。Stage validatorは四Stage refのkind／owner／version、World validatorはspatial refのWorld owner／type semantics、Runtime validatorはPort delivery edgeがexact Boundary Delivery refへ到達し、spatial／Locomotion前提を持たないことを検証する。三owner Receipt payloadはcandidate hash、Contract set hash、owner kind、validated subset hash、`result=pass`を閉じ、五gateは上記五exact payload型のどれか一つだけを使う。Payload自身はderived payload hash Fieldを持たず、完成payloadのRFC 8785 JCS bytesをSHA-256した値だけを`MirakanSignedRecordV1.subject_sha256`とReceipt Refの`payload_subject_sha256`に同じ値で保存する。canonical wrapper purposeはowner=`scenario_stage_owner_validation`、gate=`scenario_stage_gate_validation`である。Receipt refはpayloadのowner／gate discriminator、candidate hash、result、payload subject hashと完成wrapperのcanonical `MirakanSignedRecordRefV1`を持ち、payload／wrapper／refの同Fieldをbyte equalityで検査する。discriminator外payload、generic left/right pair、unknown Field、wrong purpose、subject substitution、payloadへderived self hashを再導入するcaseを拒否する。いずれも`final_closure_hash`をpayload、subject、signature preimageへ含めない。

三owner Receiptは`stage, world, runtime`のclosed順、五gate Receiptは下表の順でexact countを要求し、全八件の`result=pass`をfinal closureの必須条件にする。duplicate、missing、extra、fail／unknown result、同じcandidate hashへの別subject、payload subject hash／generic signed ref不一致、stale key／revocation、署名不成立を拒否する。`final_closure_hash`はASCII `MIRAKAN_STAGE_TRANSITION_CROSS_OWNER_FINAL_CLOSURE_V1`、candidate hash、三owner Receipt Refの全Field、五gate Receipt Refの全Fieldをこの順に各`uint32_be` length framingしてSHA-256し、自身だけを除外する。Receipt発行後にcandidateを変更せず、final closureを再署名対象へ戻すcycleを作らない。

異種inventoryを一つのset equalityへ混ぜず、次のlike-for-like gateを独立に実行する。

| Gate | 左辺 | 右辺 | equality |
|---|---|---|---|
| `gate.scenario_stage.public_contract_inventory` | Pack Manifest `public_contract_refs[]` | Stage owner public contract inventory exact 7件 | ID／version／Contract set hashのset equality |
| `gate.scenario_stage.runtime_port_inventory` | Pack Manifest `runtime_port_refs[]` | Stage owner runtime port inventory exact 2件 | ID／version／Contract set hashのset equality |
| `gate.scenario_stage.transition_mcd_ref_set` | `StageTransitionCrossOwnerCandidateV1.exact_transition_ref_set` | `StageTransitionContractRefSetV1` exact 5件 | ID／kind／version／Contract set hashのset equality |
| `gate.scenario_stage.transition_port_closure` | candidateのStage四ref＋World一ref | `StageTransitionContractRefSetV1` exact 5件 | cross-owner reachable set equality |
| `gate.scenario_stage.aggregate_projection` | Gameplay Features aggregateのScenario Stage public／runtime refs | Pack Manifestの対応するpublic／runtime refs | 各inventoryを別々にset equality |

各gateは同じcandidate hashをsubjectに固有Receiptを発行し、三owner Receipt、五gate Receipt、final closure hashが上記DAGを閉じる場合だけPack apply／Runtime Activationを許可する。`fixture.feature.scenario_stage.aggregate-manifest-set-equality`は各gateについてmissing／extra／duplicate／version／Contract set hash／Architecture owner revision／content hash mismatchに加え、candidate hash mismatch、owner kindを維持した`validated_subset_hash`の別subset／任意hash差替え、Receiptがfinal closure hashを含むcycle、Receipt order／count不正を各一原因で拒否し、一gateの成功を別inventoryの成功へ読み替えない。

## 10. Content activationとRuntime state

```text
StageContentActivationPolicyV1
  policy_id
  content_activation_scope: none | entry_anchor_closure | listed_content_refs
  required_content_refs[]
  prefetch_priority_class_ref
  fallback_contract

StageRuntimeStateV1
  stage_ref
  stage_instance_key
  lifecycle_state: inactive | preparing | ready | activating | active | deactivating | faulted
  active_entry_ref: AnchorRef | null
  activated_game_system_instance_refs[]
  completion_state_ref: null | CompletionStateRef
  activation_generation
```

`listed_content_refs`時だけ`required_content_refs[]`を1件以上持つ。`entry_anchor_closure`は`required_content_refs=[]`かつnon-null World／exact entry anchorを必須にする。`none`は`required_content_refs=[]`、entry／exit anchor 0件、content activation 0件とし、`stage_game_system_refs[]`だけで動くsystem-only headless Stageをvalidにする。`world_ref=null`では`listed_content_refs | none`を許可し、entry anchor closureを拒否する。non-null World branchのWorld／Scene／Cell activation計画はWorld／Runtime ownerのPublic Portを消費し、Stage側がCell schemaやphase ABIを再定義しない。

`world_ref=null`では`active_entry_ref=null`を必須にし、non-null World branchだけが参照WorldのAnchorを持てる。`completion_mode=none`では`completion_state_ref=null`を許可し、`active`からProject／Runtime要求で直接`deactivating`へ進める。Objective、completion outcome、Result route、`completing` stateを要求しない。`explicit_outcomes`だけが`CompletionContractV1`に基づくcompletion stateを作る。

同じStage Sourceから複数instanceを作れる。`scope.feature.scenario_stage.instance`のinstance keyでStateを分離し、Source Stable ID、Save identity、ephemeral Runtime generationを相互に置換しない。

## 11. Stage transition contract

```text
StageTransitionDestinationV2
  destination_kind: stage | runtime_entry | world_space | ui | headless | session_end
  stage_ref: StageDefinitionRef | null
  stage_hash: SHA-256 | null
  runtime_entry_ref: RuntimeEntryPointDocumentRef | null
  runtime_entry_hash: RuntimeEntryPointSemanticHashV1 | null
  spatial_destination_type_ref: McdContractRefV1 | null
  spatial_destination_ref: SpatialTransitionDestinationRefV1 | null

StageTransitionPolicyV1
  policy_id
  policy_version
  policy_hash
  destination: StageTransitionDestinationV2
  presentation_policy_ref
  persistent_subject_policy_ref
  precondition_ref
  failure_policy
  cancel_policy

StageTransitionRequestV2
  request_id
  source_stage_instance_ref
  trigger_or_outcome_ref
  requesting_system_ref
  requested_tick
  transfer_subject_refs[]
  precondition_snapshot_hash
  transition_policy_ref
  transition_policy_hash
```

`StageTransitionDestinationV2`、`StageTransitionPolicyV1`、`StageTransitionRequestV2`はそれぞれ`StageTransitionContractRefSetV1`に示した`McdContractRefV1`で参照する。`destination_kind`は全destination fieldのdiscriminatorであり、Requestはdestinationを一切持たずexact Policy ref／hashだけを指定する。Policy hashはhash Field自身を除く全Policy Field、destination中の各semantic hash／Document content hashを含むMCD canonical bytesから計算する。

| kind | non-null／non-empty Field | null／empty Field |
|---|---|---|
| `stage` | exact `stage_ref`＋`stage_hash` | Runtime Entry／spatial三Fieldはnull。Stage ownerがStage content closureを解決 |
| `runtime_entry` | exact `runtime_entry_ref`＋payload semantic `runtime_entry_hash` | Stage／spatial三Fieldはnull。解決entry kindはworldだけで、特定spatial destinationを指定しない |
| `world_space` | exact `runtime_entry_ref`＋hash、`spatial_destination_type_ref={id=type.world.spatial_transition_destination, version=1, contract_set_hash}`、exact `spatial_destination_ref` | Stageはnull。entry kindはworldだけ |
| `ui` | exact `runtime_entry_ref`＋hash | Stage／spatial三Fieldはnull。entry kindはuiだけ |
| `headless` | exact `runtime_entry_ref`＋hash | Stage／spatial三Fieldはnull。entry kindはheadlessだけ |
| `session_end` | なし | Stage／Runtime Entry／spatial三Fieldを全件null |

UI Document、headless startup systems、World ref、AnchorをStage destinationへ直接持たせず、すべてRuntime EntryまたはWorld-owned `SpatialTransitionDestinationV1`のcompile済みclosureを通す。`runtime_entry`はworld entry全体への非spatial遷移、`world_space`は同じworld entryに加えてexact spatial destinationを持つ遷移、`ui`と`headless`は各entry kind専用であり、六branchの受理集合は相互排他的である。`world_space`はruntime entryが参照するWorldとspatial destinationのWorldがexact equalityであること、entry selectorが現在Targetを含むこと、World／Topology／Edge／Space／Anchor version／hashが一致することをactivation前に検証する。`ui`／`headless`もentry kindを検証し、Stageがstartup closureを再構成しない。

`transfer_subject_refs[]`はregistered typed subjectだけを受理し、display roleやowner推測でPlayer／Partyへ変換しない。destinationがWorld Spaceの場合は[World](../06-rendering/world.md)のgeneric spatial transition port、別Stageの場合は`StageActivationPortV1`を使う。いずれのbranchもsealed transition payloadを[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)のregistered `BoundaryDeliveryContractV1`へ渡し、次のeligible `T00_BoundaryApply`で適用する。Stage側にT40／Motion Executor／Locomotion dependencyを追加しない。

target dependency不足、stale precondition、unknown destination、subject incompatibilityではpartial activationせず、source Stageとlast-valid World generationを維持する。Loading progress／cancel／retryの表示契約はWorld／Runtimeのgeneric Loading projectionを参照し、Stage outcomeをLoading UIから推測しない。

qualificationは六kindを各一件round-tripし、`runtime_entry`はworld／spatialなしだけを検証する。negative fixtureはruntime_entryへのui／headless entryまたはspatial ref、discriminator外Field、Requestへのinline destination、Policy hash mismatch、Runtime Entry payload／Document hash mismatch、ui／headless entry kind mismatch、world_spaceのnon-world entry、spatial type ref kind／version／Contract set mismatch、required anchor欠落、stale World／Topology／Space／Edge／Anchor、spatial destination hash mismatch、session_end payloadを各単独原因で拒否し、source Stage／World／Project revisionを不変にする。

## 12. Save identityとmigration

`StageSaveReplayPolicyV1`はStage Source identity、definition version、Stage save instance identity、Scope key、Stage-owned State fields、Objective／Spawn／transition owner projection、checkpoint／resume、Replay oracle、migration、fallbackを宣言する。

同じSource Stageから複数saved instanceを作る場合も、各instanceへ別のUUIDv7 save identityを発行する。Loadは新しいephemeral runtime generationへone-to-oneでremapし、保存前のhandle／generationを復元しない。`completion_mode=none`ではcompletion fieldやResultをSave recordへ合成しない。

旧World Level契約からのclean migrationは次の一方向とし、World側へaliasを残さない。

| 旧World identity | Scenario／Stage identity |
|---|---|
| `LevelDefinitionV1` | `StageDefinitionV1` |
| `LevelStreamingPolicyV1` | `StageContentActivationPolicyV1` |
| `LevelRuntimeStateV1` | `StageRuntimeStateV1` |
| `LevelTransitionPolicyV1` | `StageTransitionPolicyV1`＋`StageTransitionDestinationV2` |
| `LevelTransitionRequestV1` | `StageTransitionRequestV2`（inline destinationをPolicy ref／hashへ分離） |
| `level_gameplay` owner role | owner明示の`stage_game_system_refs[]`とStage Scope |
| `player_or_party_transfer_refs[]` | typed `transfer_subject_refs[]` |
| required `completion_contract` | tagged `completion_mode`＋nullable `completion_contract` |

MigrationはSource ref、Save identity、active entry、Stage-owned State、transition precondition、Replay oracleをPreviewし、Stage ownerのValidationとApproval後にcommitする。旧World fieldをcompatibility aliasとして読まない。

## 13. Diagnosticとqualification

| Diagnostic ID | 条件 | 結果 |
|---|---|---|
| `MIRAKAN-SCENARIO-STAGE-DEFINITION_INVALID` | world nullable branch、anchor／spatial spawn、ref、Scope不正 | Source／Cookを拒否 |
| `MIRAKAN-SCENARIO-STAGE-ACTIVATION_FAILED` | content／Feature closure不成立 | source／last-valid generation維持 |
| `MIRAKAN-SCENARIO-STAGE-TRANSITION_FAILED` | destination／subject／precondition不正 | partial transition禁止 |
| `MIRAKAN-SCENARIO-STAGE-SAVE_CONTRACT_MISMATCH` | Source／Scope／State owner不一致 | Saveを開かずbackup維持 |
| `MIRAKAN-SCENARIO-STAGE-REPLAY_DIVERGENCE` | first Stage-owned divergence | 停止してReproduction Bundle生成 |

Qualificationは`completion_mode=none`と`explicit_outcomes`、world branch、`fixture.feature.scenario_stage.worldless-ui`、`fixture.feature.scenario_stage.worldless-headless`、WorldなしDialogue／Visual Novel workflow、複数Stage instance、content activation exact／exact+1、`none` system-only、全六destination kindのtagged transition positive／negative、transition success／rollback、typed subject compatibility、Save／Load／Replay、旧Level Source migration、Scenario／Stage Pack removalを含む。UI／headless branchのanchor／spatial spawn混在、session_endのdestination payload、world_space required anchor欠落、Request inline destination、Policy ref／hash不一致をrejectし、Feature PackなしのWorld／endless Projectは引き続きvalidである。
