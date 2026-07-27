# Miraikanai Engine Pack Contract

- 文書ID: mirakan.arch.pack-contract
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Packの4層境界、`PackManifestV1`、Feature／Genre依存、Profile ownership、install／update／removal、migration、last-valid規則
- 非正本範囲: Product roadmap／Capability成熟度、各FeatureのPublic Contract、Genre固有composition、共有MCD／ChangeSet、Subsystem契約は各Ownerを参照
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Project State](../03-authoring/project-state.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)、[Executable Contracts](../02-foundation/executable-contracts.md)
- 関連文書: [Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Project State](../03-authoring/project-state.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)、[Native Game Module](../03-authoring/native-game-module.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-23

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

Game ProjectはGenre Packを使用せず、Feature Packを直接構成できる。[Product Plan](../00-product/product-plan.md)が採用するRPG-firstは最初のProduct Reference選択であり、Core、Editor、AI、Project C++、Project Shader、Test、Build、Package、Network、ReleaseへRPG Feature／Genre Pack依存を追加しない。current Installed ProductのShooter compositionはsource baselineおよびtechnical qualification consumerとして保持できるが、将来Product identityまたはCore baselineへ読み替えない。ProductのRPG、Shooter、Platformer、Puzzle-Dialogue等のGateはbundled Reference Game coverageを資格化するnonblocking trackであり、Generic EngineのRelease Gate、CX3 shipping、production-release bindingから参照しない。Reference選択の変更はPack IDのrenameやReceipt流用ではなく、owner designとatomic Product Definition Migrationで行う。

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
  authoring_operation_refs[]:
    McdContractRefV1(kind=operation)
  runtime_port_refs[]
  configuration_profile_refs[]
  composition_recipe_refs[]
  source_template_refs[]
  validator_refs[]:
    exact {validator_id, validator_version, validator_content_hash}
  test_scenario_refs[]:
    exact {fixture_id, fixture_version, fixture_content_hash}
  example_refs[]
  counterexample_refs[]
  ai_vocabulary_refs[]
  ai_planning_recipe_refs[]
  performance_profile_refs[]
  migration_step_refs[]:
    exact {migration_id, migration_version, migration_content_hash}
  migration_contribution_refs[]:
    exact {registry_id, contribution_id, contribution_version,
           contribution_content_hash}
  license_ref
  provenance_ref

PackLocalIdentityV1
  pack_id
  pack_version

PackContractRefV1
  pack_id
  pack_version
  content_hash
```

`pack_id`はPackの論理identity、`pack_version`はSemVer、`content_hash`は自己Fieldを除くcanonical manifestと同梱payloadのcontent identityである。同じ`pack_id`と`pack_version`に異なる`content_hash`を受理しない。全参照は上記version／hashを含むexact identityへ解決し、arrayは参照identityのcanonical orderでserializeする。unknown、missing、duplicate、曖昧な表示名、`latest`の暗黙選択を拒否する。Pack artifactに含まれるOperation、migration step、migration contribution、Validator、test scenarioは一件残らず対応inventoryに列挙し、payload探索や命名prefixで補完しない。

`minimum_engine_contract_ref`は利用可能性の証拠ではない。active TargetでEngine contract、required Capability、Feature Pack closure、Validator、Test、root外Recipe Activation Bindingが指すQualification Receiptがすべて解決して初めてPackを適用できる。

`validator_refs[]`と`test_scenario_refs[]`はPack artifact内のinventory、identity resolution、owner／hash検査の一覧であり、列挙した全Validator／Fixtureを全Recipe共通の実行gateにしない。Pack installは全Receipt-free recordのschema、owner、version、hashを検証するが、Project apply／Cookの実行gateは選択Recipeの`validator_refs[]`とroot外`PackRecipeActivationBindingV1`が指すexact signed Qualification Receiptだけである。Fixture bodyは別owner-typed Qualification subjectだけが解決し、Production Recipe／Runtime Packageは解決しない。Manifest inventoryに存在しても未選択Recipe専用Validator／Fixture、その条件Capability、依存Packをactive closureへ追加しない。

Pack activation transactionは`PackManifestV1.authoring_operation_refs[]`、active MCD Contract set内でownerが当該PackのOperation LocalRef集合、`TrustedServiceLocalRecordV2.allowed_operation_local_refs[]`へ当該Packが寄与する集合の三者をID／versionでset equalityにする。missing／extra／duplicate／wrong kind／stale version／hash、Service allowlistだけのOperation、ManifestだけのOperationを一件でも検出したらset rootを発行しない。Pack removalも同じtransactionで三集合からexact subsetを除去し、別Pack／Coreのallowlistを変更しない。

Validator gateは異種集合を混ぜず、次の二equalitiesを独立に検査する。

1. `PackManifestV1.validator_refs[] = Validator Registryの当該Pack owner record subset`（Validator ID／version／content hash）。
2. 各Operationについて`OperationValidatorClosureV1.validator_refs[]が宣言するValidator error_refs[] union = closure.reachable_error_refs[] = McdOperationContractV1.errors[]`（Diagnostic ID／code／version／content hash）。

Manifest Validator inventoryをDiagnostic集合、closure reachable error集合をValidator inventoryと比較しない。一方だけの成功を他方へ読み替えず、missing／extra／duplicate／stale refを各gateで別Diagnosticにする。

### 3.1 `CompositionRecipeV1`

Pack全体で常時必要なFeature dependencyと、選択したcompositionだけが必要なFeature dependencyを分離する。`PackManifestV1.required_feature_pack_refs[]`は全Recipe共通のunconditional edge、`CompositionRecipeV1.required_feature_pack_refs[]`は当該RecipeをProjectへ適用する時だけ有効なconditional edgeである。

```text
CompositionRecipeV1
  recipe_id
  recipe_version
  recipe_hash
  owner_pack_local_identity: exact PackLocalIdentityV1
  required_capability_refs[]
  required_feature_pack_refs[]
  configuration_profile_refs[]
  game_spec_template_refs[]
  action_role_set_refs[]
  source_template_refs[]
  validator_refs[]
  fallback_recipe_ref: CompositionRecipeRef | null

PackRecipeQualificationSubjectV1
  qualification_id
  qualification_version: positive uint32
  owner_pack_ref: exact PackContractRefV1
  recipe_ref/hash: exact CompositionRecipeV1
  target_profile_refs[1..64]
  fixture_refs[1..256]:
    exact {fixture_id, fixture_version, fixture_content_hash}
  input_closure_hash
  result: pass | fail
  qualification_subject_hash: SHA-256

PackRecipeQualificationReceiptV1
  subject: PackRecipeQualificationSubjectV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=pack_recipe_qualification)

PackRecipeQualificationReceiptRefV1
  qualification_id
  qualification_version: positive uint32
  qualification_subject_hash: SHA-256
  signed_record_hash: SHA-256

PackRecipeActivationBindingRefV1
  activation_binding_id
  activation_binding_version: positive uint32
  activation_binding_hash: SHA-256

PackRecipeActivationBindingV1
  activation_binding_id
  activation_binding_version: positive uint32
  recipe_ref/hash: exact receipt-free CompositionRecipeV1
  qualification_receipt_ref: PackRecipeQualificationReceiptRefV1
  activation_binding_hash: SHA-256

PackRecipeActivationProjectionV1
  projection_id
  projection_version: positive uint32
  selected_recipe_ref/hash: exact receipt-free CompositionRecipeV1
  recipe_activation_binding_ref: PackRecipeActivationBindingRefV1
  dependency_closure_ref/hash: exact RecipeDependencyClosureV1
  projection_hash: SHA-256
```

`recipe_hash`は自己Fieldを除くReceipt-free canonical recordのSHA-256であり、所有Packの`content_hash`へ含める。Recipeの`owner_pack_local_identity`は所有Manifestの`{pack_id,pack_version}`とbyte equalityにし、Pack `content_hash`、`PackContractRefV1`、Qualification Receipt／Bindingを含めない。Recipe、Profile、Template、Pack Manifestのhash preimageへQualification Receipt／Binding／Fixtureを含めない。全arrayはexact identityのcanonical orderとし、unknown、duplicate、self dependency、Genre Pack ref、Project／FixtureへのProduction dependency、version／hash conflictを拒否する。`validator_refs[]`はこのRecipe選択時だけapply gateになり、別RecipeのPerception、Stage、full-profile Fixture等を要求しない。Productionはroot外Activation BindingからReceiptのsubject／owner／Recipe／Target／result／freshnessだけを検証し、`PackRecipeQualificationSubjectV1.fixture_refs[]`を解決しない。`fallback_recipe_ref`は同じowner Pack内のRecipeだけを指し、fallback cycleを拒否する。fallbackは元RecipeのGameplay意味を暗黙変更せず、Projectが明示選択した時だけ別のdependency closureを解決する。

生成順は`Pack local identity → receipt-free CompositionRecipeV1／recipe hash → Pack Manifest content hash → PackContractRefV1／Recipe ref → PackRecipeQualificationSubjectV1 → signed Receipt → PackRecipeActivationBindingV1 → Project-owned Activation projection`である。subject hashはASCII `MIRAKAN_PACK_RECIPE_QUALIFICATION_SUBJECT_V1`、binding hashはASCII `MIRAKAN_PACK_RECIPE_ACTIVATION_BINDING_V1`、projection hashはASCII `MIRAKAN_PACK_RECIPE_ACTIVATION_PROJECTION_V1`と各自己Fieldを除くcount／length-framed canonical bytesから計算する。Subject `owner_pack_ref`のpack ID／versionはRecipe `owner_pack_local_identity`とbyte equality、content hashはそのRecipeをinventoryに含む完成Manifestの`content_hash`とbyte equalityにする。Binding Recipe pairはsubjectとbyte equalityでなければならない。PackContractRefをRecipe hash preimageへ戻さず、Receipt／Binding／ProjectionをRecipe、Profile、Manifest、Pack content hashへ戻さない。owner local ID/version、Pack hash、Recipe、Target、fixture、subject hash、signed hash、binding hashのstaleまたはsubstitutionを各一原因でrejectする。

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
- Validator、Runtime Port、planned authoring action vocabularyまたは完全登録済みMCD Operation ref
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

AIは自然言語からPack IDを推測してcommitしない。本節で従来name-onlyに列挙した八actionは完全なMCD登録を持たないため、current Operation setを空にしてCapability stateを`not_activated`とする。current MCD、Pack Manifest、Trusted Service allowlist、Provider／MCP Catalogから除外し、旧`operation.packs.*`名をlegacy aliasとして読まない。

```text
Current generic Pack AI Operation set = {}
Future vocabulary:
  future.packs.action.search
  future.packs.action.read
  future.packs.action.resolve_dependencies
  future.packs.action.explain_composition
  future.packs.action.plan_apply
  future.packs.action.preview_apply
  future.packs.action.validate
  future.packs.action.plan_remove
```

要求は`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`でProject／Pack registry不変として拒否する。future work item `activation.pack.ai_operations.v1`は採用するexact Operation set、MCD全Field、named input／result、Service／Policy／Validator／Diagnostic／Receipt、Risk、authorization intent DAG、private-to-public recovery、Qualificationを同じContract set transactionで完全登録するまでactivateしない。将来の依存、影響、migration、fallbackの説明は`pack_id`、`pack_version`、`content_hash`、Capability ID、Feature Pack edge、Project revisionを含め、Strict Tool Callingに適合しないProviderをCommit Operationへ推測変換しない。

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
