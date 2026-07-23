# Miraikanai Engine Pack Contract

- 文書ID: mirakan.arch.pack-contract
- 状態: review
- 正本範囲: Packの4層境界、`PackManifestV1`、Feature／Genre依存、Profile ownership、install／update／removal、migration、last-valid規則
- 非正本範囲: Product roadmap／Capability成熟度、各FeatureのPublic Contract、Genre固有composition、共有MCD／ChangeSet、Subsystem契約は各Ownerを参照
- 依存: [Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Project State](../03-authoring/project-state.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)、[Native Game Module](../03-authoring/native-game-module.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)
- 外部根拠検証日: 2026-07-23

## 1. 結論

Miraikanai Engineの製品構造は次の4層である。

1. `Generic Engine Core`
2. `Reusable Feature Packs`
3. `Genre Packs`（任意）
4. `Game Projects`

`Reference Games`は第5のProduction Runtime層ではない。通常のGame Projectとして選定された検証対象であり、Fixtureと同様にProduction artifactから参照されない。

依存方向は利用側から提供側への一方向とする。

```text
Game Project
  -> optional Genre Pack
  -> Feature Pack
  -> Generic Engine Core
```

Game ProjectはGenre Packを使用せず、Feature Packを直接構成できる。Shooter-firstは実装順序にすぎず、Core、Editor、AI、Project C++、Project Shader、Test、Build、Package、Network、Releaseの成立条件にShooter Genre Packを置かない。

## 2. 層と依存規則

| 利用側 | 許可する依存先 | 拒否する依存先 |
|---|---|---|
| Generic Engine Core | Core内の下位契約 | Feature Pack、Genre Pack、Game Project、Reference Game、Fixture |
| Feature Pack | Generic Engine Core、別Feature Pack | Genre Pack、Game Project、Reference Game、Fixture |
| Genre Pack | Generic Engine Core、Feature Pack | 別Genre Pack、Game Project、Reference Game、Fixture |
| Game Project | Generic Engine Core、Feature Pack、任意のGenre Pack | Reference Game、Fixture |
| Reference Game／Fixture | 検証対象のCore、Pack、Project | Production artifactへの逆向き依存 |

Feature Pack間依存だけを有向非巡回グラフ（DAG）として許可する。resolverは自己依存、重複edge、cycle、欠落Pack、互換versionの空intersection、同一Pack identity／versionの異なるhashを拒否する。Genre Pack同士の直接依存は、cycleの有無にかかわらず拒否する。複数Genreの合成はGame Projectが明示的に行い、Genre Pack間の隠れた推移依存にしない。

Profileは独立Packではない。所有Pack artifact内の構成単位であり、独立したmanifest、install state、registry entryを持たず、所有Packの`pack_version`と`content_hash`に含まれる。

## 3. `PackManifestV1`

`PackManifestV1`は次のcanonical fieldを一つずつ持つ。`pack_kind`は`feature | genre`のclosed enumである。

```text
PackManifestV1
  pack_id
  pack_version
  pack_kind: feature | genre
  content_hash
  minimum_engine_contract_ref
  supported_target_profile_refs[]
  required_capability_refs[]
  required_feature_pack_refs[]
  provided_capability_refs[]
  public_contract_refs[]
  component_schema_refs[]
  game_system_spec_refs[]
  authoring_operation_refs[]
  runtime_port_refs[]
  configuration_profile_refs[]
  composition_recipe_refs[]
  source_template_refs[]
  validator_refs[]
  test_scenario_refs[]
  example_refs[]
  counterexample_refs[]
  ai_vocabulary_refs[]
  ai_planning_recipe_refs[]
  performance_profile_refs[]
  migration_step_refs[]
  license_ref
  provenance_ref
```

`pack_id`はPackの論理identity、`pack_version`はSemVer、`content_hash`は自己Fieldを除くcanonical manifestと同梱payloadのcontent identityである。同じ`pack_id`と`pack_version`に異なる`content_hash`を受理しない。全参照はexact identityへ解決し、arrayは参照identityのcanonical orderでserializeする。unknown、missing、duplicate、曖昧な表示名、`latest`の暗黙選択を拒否する。

`minimum_engine_contract_ref`は利用可能性の証拠ではない。active TargetでEngine contract、required Capability、Feature Pack closure、Validator、Test、Qualification Receiptがすべて解決して初めてPackを適用できる。

`validator_refs[]`と`test_scenario_refs[]`はPack artifact内のinventory、identity resolution、owner／hash検査の一覧であり、列挙した全Validator／Fixtureを全Recipe共通の実行gateにしない。Pack installは全recordのschema、owner、version、hashを検証するが、Project apply／Cook／Qualificationの実行gateは選択Recipeの`validator_refs[]`と`qualification_fixture_refs[]`だけである。Manifest inventoryに存在しても未選択Recipe専用Validator／Fixture、その条件Capability、依存Packをactive closureへ追加しない。

### 3.1 `CompositionRecipeV1`

Pack全体で常時必要なFeature dependencyと、選択したcompositionだけが必要なFeature dependencyを分離する。`PackManifestV1.required_feature_pack_refs[]`は全Recipe共通のunconditional edge、`CompositionRecipeV1.required_feature_pack_refs[]`は当該RecipeをProjectへ適用する時だけ有効なconditional edgeである。

```text
CompositionRecipeV1
  recipe_id
  recipe_version
  recipe_hash
  owner_pack_ref
  required_capability_refs[]
  required_feature_pack_refs[]
  configuration_profile_refs[]
  game_spec_template_refs[]
  action_role_set_refs[]
  source_template_refs[]
  validator_refs[]
  qualification_fixture_refs[]
  fallback_recipe_ref: CompositionRecipeRef | null
```

`recipe_hash`は自己Fieldを除くcanonical recordのSHA-256であり、所有Packの`content_hash`へ含める。全arrayはexact identityのcanonical orderとし、unknown、duplicate、self dependency、Genre Pack ref、Project／FixtureへのProduction dependency、version／hash conflictを拒否する。`validator_refs[]`／`qualification_fixture_refs[]`はこのRecipe選択時だけapply／qualification gateになり、別RecipeのPerception、Stage、full-profile fixture等を要求しない。`fallback_recipe_ref`は同じowner Pack内のRecipeだけを指し、fallback cycleを拒否する。fallbackは元RecipeのGameplay意味を暗黙変更せず、Projectが明示選択した時だけ別のdependency closureを解決する。

選択Recipe `R`のeffective Feature closureは、所有Manifestの`required_feature_pack_refs[]`、`R.required_feature_pack_refs[]`、両集合から到達するFeature Pack DAGの和集合である。resolverは次を生成する。

```text
RecipeDependencyClosureV1
  selected_recipe_ref
  owner_pack_content_hash
  manifest_required_feature_pack_refs[]
  recipe_required_feature_pack_refs[]
  transitive_feature_pack_refs[]
  resolved_pack_version_and_hash_refs[]
  closure_hash
```

`closure_hash`は上記Fieldのcanonical serializationから計算し、Preview、Project ChangeSet、Cook、Qualification Receipt、Save／Replay headerへ同じ値を伝播する。Pack install時は全Recipe recordのschema、hash、参照kind、所有関係を検証するが、未選択Recipeのconditional dependencyをinstalled closureへ暗黙追加しない。Project apply／Cook時は選択Recipeのeffective closureを原子的に解決し、一件でもmissing、incompatible、unqualified、Target不適合なら`MIRAKAN-PACK-RECIPE-DEPENDENCY_UNRESOLVED`で拒否する。Project revision、active Recipe、registry head、installed closure、Cooked Artifactは変更せず、直前のlast-valid Recipe activationとclosure hashを維持する。partial Recipe applyとmissing Featureのplaceholderを禁止する。

### 3.2 Feature Pack

Feature Packは複数Genre／Projectで再利用する次の要素を提供する。

- Public Contract、Component Schema、Game System Spec
- Validator、Runtime Port、Authoring Operation
- AI vocabulary、planning recipe
- Engine-ownedまたはProject-ownedのreference implementation
- contract fixture、example、counterexample、performance profile

Feature Packの`required_feature_pack_refs[]`は別Feature Packだけを指す。Feature PackはGenre固有Profile、Genre vocabulary、Game Project、Fixtureの内容を参照しない。

### 3.3 Genre Pack

Genre PackはFeature Packを組み合わせる次の要素だけを提供する。

- composition recipe
- Genre vocabularyとGameSpec template
- Genre固有configuration Profile
- reference scenarioとGenre fixture binding

Genre Packは新しい汎用Core契約を作らず、Feature CapabilityのPublic Contract、Schema、State owner、Runtime Portを複写しない。ManifestとRecipeの`required_feature_pack_refs[]`はFeature Packだけを指し、別Genre Packを指せない。条件依存をManifestのunconditional edgeへ昇格させず、使用するRecipe recordへ記録する。

## 4. Installとdependency resolution

Installは次の順で原子的に評価する。

1. artifact bounds、canonical manifest、`content_hash`、license、provenance、signatureを検証する。
2. `minimum_engine_contract_ref`、Target intersection、required Capabilityを検証する。
3. Manifest共通Feature Pack DAGを解決し、Genre間dependency、cycle、version／hash conflictを拒否する。
4. Public Contract、Schema、Profile、`CompositionRecipeV1`、Template、Validator、Fixtureの全参照kind／hash／ownerをload前に検証する。未選択Recipeのconditional dependencyはinstalled closureへ追加しない。
5. 全検証成功時だけPack registry revisionをcommitする。

失敗時はregistry、Project、installed Pack closureを変更せず、直前のlast-valid registry headとartifactを維持する。部分install、依存の暗黙追加、Genre間のsynthetic edge、未有効Capabilityのplaceholderを禁止する。

Projectへの適用は[Project State](../03-authoring/project-state.md)のPreview／Commit／Undoを使う。PreviewはPack／version／hash、dependency closure、Target、生成または変更するDocument／Asset／Source、migration、conflict、fallback、Qualificationへの影響をexact IDで表示する。Native source templateは[Native Game Module](../03-authoring/native-game-module.md)の別ChangeSet、review、build、promotionへ隔離する。

## 5. Updateとmigration

Updateはinstalled base、Projectへ適用済みのinstance、現在の人間／AI変更を使う三者比較Diffで行う。`migration_step_refs[]`を順序付きで解決し、各stepの入力version／hash、出力version／hash、対象Source／Save／Replay、precondition、rollbackまたはfallbackをPreviewする。

Patch／minor updateでもPublic Contract、persisted Source、Save／Replay、Profile、Recipeが変わる場合は対応migrationと再Qualificationを要求する。Major updateはoffline migration、明示承認、Save／Replay／Build／Packageの再Qualificationを必須とする。Runtime互換shim、削除済みidentityの再利用、旧名alias、Template再展開によるhuman override上書きを禁止する。

update transactionのいずれかが失敗した場合は新revisionをcommitしない。旧Pack version、旧registry head、Project revision、Save／Replay、last-valid Build／Package artifactを維持し、失敗したstepとremediationを返す。Package済みGameへnetworkからPackを暗黙更新しない。

## 6. Removal

Removal planは次をexact IDで列挙する。

- 依存するFeature Pack、Genre Pack、Game Project
- 適用済みTemplate instance、Source、Asset、Save／Replay field
- 削除または置換が必要なProfile、Recipe、Capability、Package
- 必要なmigration、fallback、再Qualification

live dependencyが一件でも残る場合はremovalを拒否する。Genre Packの削除は共有Feature Packを所有推測で削除せず、別Project／Genreが参照するFeature Packを維持する。Project変更、Cook／Package再生成、registry mutationを別々のcommit境界にし、中間失敗時もlast-valid Projectとartifactを開ける状態を維持する。

Shooter Genre Packを未installまたは削除した状態でも、Core、Editor、AI、Project C++、Project Shader、Test、Build、Packageが成功しなければならない。

## 7. AI discoveryとoperation

AIは自然言語からPack IDを推測してcommitしない。MCDへ次のexact Operationを登録し、同じValidation、Staging、Approval、ReceiptをEditor内AI、local inference、cloud Provider、外部MCP Client、CLIへ適用する。

- `operation.packs.search`
- `operation.packs.read`
- `operation.packs.resolve_dependencies`
- `operation.packs.explain_composition`
- `operation.packs.plan_apply`
- `operation.packs.preview_apply`
- `operation.packs.validate`
- `operation.packs.plan_remove`

依存、影響、migration、fallbackの説明は`pack_id`、`pack_version`、`content_hash`、Capability ID、Feature Pack edge、Project revisionを含む。Strict Tool Callingに適合しないProviderはread-onlyまたはproposal-onlyとし、自然言語をCommit Operationへ推測変換しない。

## 8. Diagnosticとfixture

| Failure | Diagnostic ID | 結果 |
|---|---|---|
| Manifest／hash／license／provenance不正 | `MIRAKAN-PACK-MANIFEST_INVALID` | Installを拒否しlast-valid registryを維持 |
| Feature dependency cycle／version conflict | `MIRAKAN-PACK-DEPENDENCY_UNRESOLVED` | closureを拒否しcycleまたは競合rangeを列挙 |
| 選択Recipe dependencyのmissing／incompatible／unqualified | `MIRAKAN-PACK-RECIPE-DEPENDENCY_UNRESOLVED` | Recipe applyを拒否しlast-valid Recipe closureを維持 |
| Genre Pack間dependency | `MIRAKAN-PACK-GENRE_DEPENDENCY_FORBIDDEN` | edgeを拒否しGame Project compositionを案内 |
| Engine contract／Target不適合 | `MIRAKAN-PACK-VERSION_INCOMPATIBLE` | Applyを拒否しshimを生成しない |
| required Capabilityなし | `MIRAKAN-PACK-CAPABILITY_UNAVAILABLE` | Apply／Cookを拒否しTargetとCapabilityを列挙 |
| migration失敗 | `MIRAKAN-PACK-MIGRATION_FAILED` | 新revisionをcommitせずlast-validを維持 |
| live dependencyあり | `MIRAKAN-PACK-REMOVAL_BLOCKED` | Removalを拒否しdependency closureを列挙 |

Pack contract fixtureは最低限、manifest／`CompositionRecipeV1` round-trip、全Field presence、feature DAG／cycle／diamond、Manifest共通closureと選択Recipe conditional closureのhash、未選択Recipe dependency非install、Recipe missing／version conflict／Target不適合時のlast-valid維持、fallback cycle、Genre間dependency拒否、Genre PackなしのFeature-only Project、install／update／removal rollback、migration失敗時のlast-valid維持、Shooter未install時のCore／Editor／AI／Build／Package成功を含む。Reference Game／FixtureがProduction artifactへ逆依存しないこともdependency graphで検査する。
