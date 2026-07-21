# Miraikanai Engine Domain Pack Contract

- 文書ID: mirakan.arch.domain-pack-contract
- 状態: review
- 正本範囲: Domain Packのidentity／manifest、Capability宣言、依存解決、install／activation、Project適用、AI語彙、qualification、update、removal、Domain固有failure
- 非正本範囲: Product roadmap／Capability成熟度、共有MCD／ChangeSet構造、Subsystem契約／数値、AI承認、Asset／Native build、Runtime scheduling／capacityは各依存先を参照
- 依存: [Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Project State](../03-authoring/project-state.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)、[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)、[Native Game Module](../03-authoring/native-game-module.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論と境界

Genre固有の語彙、Template、Rule、Validator、reference scenario、AI制作手順をEngine Coreの継承階層へ組み込まず、versioned `DomainPack`として構成する。Domain PackはEngine forkまたは任意binary pluginではなく、MCDで検証可能なdata packageである。Project C++ source templateを含められるが、生成物は通常のNative Game Module build／review／promotionを通る。

本文書は次だけを決定する。

- Pack identity、manifest、dependencyとCapability要求
- install、Project適用、activation、update、removalの原子的状態遷移
- Domain固有Template、semantic vocabulary、validator、reference scenario
- Pack固有qualification closureと失敗結果

Product horizon、Capability成熟度と昇格順序は[Product Plan](../00-product/product-plan.md)だけが所有する。Subsystemの型、固定値、phase、budget、Backendは各Subsystem正本だけが所有する。Domain Packは上限を緩和せず、未有効Capabilityを利用可能と宣言しない。

## 2. Pack identityとartifact

Pack identityはUUIDv7 `StableId`、`pack_version`はSemVer、artifact内容はSHA-256で固定する。`pack_id`はrename、version更新、配布元変更で変えず、別の互換性境界を作る時だけ新規発行する。表示名、path、registry順、package filenameをidentityにしない。

Pack artifactはcanonical manifest、MCD modules、Template、Validator、Fixture、license／provenanceを含むcontent-addressed packageである。Runtimeがnetwork registryから暗黙更新したり、manifestからnative binary、Script、post-install executableをloadしたりしてはならない。

Pack registry entryは少なくとも`pack_id`、`pack_version`、content hash、signature／provenance reference、Engine contract range、install stateを結ぶ。同じidentity／versionで異なるhashを受理せず、registry mutationはProject stateと分離した原子的操作にする。

## 3. `DomainPackManifest`

`DomainPackManifest`は次のcanonical fieldを持つ。

```text
DomainPackManifest
  pack_id: StableId
  pack_version: SemVer
  minimum_engine_contract: McdContractRefV1
  supported_target_profile_refs[]
  required_capability_refs[]
  optional_capability_refs[]
  schema_module_refs[]
  definition_template_refs[]
  scene_ui_asset_template_refs[]
  native_source_template_refs[]
  validator_refs[]
  test_scenario_refs[]
  ai_vocabulary_refs[]
  ai_planning_recipe_refs[]
  performance_profile_refs[]
  migration_step_refs[]
  license_ref
  provenance_ref
  content_sha256
```

全参照はexact versionまたはcontent identityへ固定する。required／optional、Source／Derived、data／native sourceを曖昧なtagで兼用しない。arrayは参照identityのcanonical orderでserializeし、missing ref、duplicate ref、自己依存、dependency cycle、同じCapabilityへの矛盾したversion要求を拒否する。

Manifestは次を所有できる。

- Genre用GameSpec section、typed data、Gameplay Definition composition
- Scene、Game UI、Input Action、Asset RequirementのTemplate
- Capability parameter profile、semantic tag、AI vocabulary
- Domain validator、reachability／playability test、reference scenario
- Tutorial、onboarding、sample、bounded Native source template

ManifestはWorld／Entity／Asset／Saveの共有identity、native Renderer／Physics／Navigation／Animation Backend、Platform SDK／Store、AI権限、Gateway bypass、Engine private header、vendor object、無制限callbackを所有しない。

## 4. Capability宣言と依存解決

`required_capability_refs`は適用、Cook、Packageに必須であり、`optional_capability_refs`は不在時の明示fallbackまたは機能非表示をManifest内で参照する。文字列一致、似た表示名、上位Tierから暗黙補完しない。解決結果はPack、Engine contract、Target Profile、Capability Manifestのexact revisionへ固定する。

一Projectは複数Packを使用できる。ただしPack間共有機能はversioned Core Capabilityまたは独立したreusable Feature Packへ抽出し、Genre Pack同士を直接linkしない。resolverは全Packのdependency closureを先に構築し、cycle、version rangeの空intersection、Target不適合、Capability未有効、Input Action／Save field／UI route／semantic tag／Asset pathのconflictを一括報告する。

Core Capabilityの追加が必要なら、Product Planのactivation手順と該当Subsystem正本の改訂を先に完了する。Pack内のplaceholder、optional ref、validator suppressionで不足Capabilityを成功扱いにしない。

## 5. Install、Project適用、activation

Installは次の順で原子的に評価する。

1. artifact bounds、canonical manifest、license、signature／hash、Engine contract rangeを検証する。
2. dependency closure、required Capability、Target intersectionを解決する。
3. schema modules、Template、Validator、Fixtureをloadせずに検証する。
4. 全検証成功時だけregistry revisionをcommitする。

Project適用は`ApplyDomainPackChangeSet`として[Project State](../03-authoring/project-state.md)のPreview／Commit／Undoに従う。Previewは生成または変更するDocument、Asset、Definition、C++ sourceと、既存ID／name／Input／Save／UI／Style conflictを分けて表示する。部分適用、直接Source write、validator bypassを禁止する。

ProjectはPackのlive mutable objectを参照せず、適用済みTemplate instanceと次の`PackApplicationRecord`を保持する。

```text
PackApplicationRecord
  application_id: StableId
  pack_id: StableId
  pack_version: SemVer
  pack_content_sha256
  applied_project_revision
  target_profile_refs[]
  resolved_capability_refs[]
  instantiated_source_refs[]
  human_override_refs[]
  qualification_receipt_refs[]
```

Activationはinstall済みであること、Project apply commit、Cook／Build成功、Pack validatorとrequired test scenarioの合格、必要な承認とevidenceが揃うことを要求する。Native source templateは別の`NativeCodeChangeSet`へ隔離し、Pack適用と同時に昇格させない。

## 6. AI vocabularyとplanning recipe

`AiDomainVocabulary`は次を対応付ける。

```text
AiDomainVocabulary
  user_term
  canonical_concept_id
  required_questions[]
  candidate_capability_refs[]
  default_profile_ref
  validation_scenario_refs[]
  forbidden_assumptions[]
```

`AiPlanningRecipe`はPrompt本文ではなく、RequirementからDocument、Capability、Testへのtyped mappingである。Providerが変わっても同じGateway、Validator、Testを使う。AIはPackを検索、提案、選択できるが、未install Packまたは未有効Capabilityを存在するものとして生成しない。

候補間でGameplay、Security、Cost、Target対応が変わる項目は質問し、safe defaultで解決できる項目はAssumptionとしてPreviewへ出す。Pack追加、major update、Native source適用はrisk、dependency closure、Diffを人間へ表示し、[AI Security／Approval](../01-governance/ai-security-approval.md)の承認を迂回しない。

## 7. Qualification

Packのqualificationは「Templateを生成できる」ことではなく、reference scenarioを同じPublic Contract、Project format、AI／manual workflowでinstall、apply、edit、Cook、Package、play、update、removeできることを証明する。

各Packは少なくとも次を宣言する。

- required／optional CapabilityとTarget Profile
- reference scenario、playability／reachability invariant、domain acceptance test
- Domain固有scale intentと[Performance／Capacity](../04-runtime/performance-capacity.md)へのprofile reference
- Save／Load、Replay、migration closure
- manual editとAI editの往復、failure／recovery fixture
- qualification evidenceと再Qualification trigger

Subsystemの共有budgetまたは測定値をManifestへ複写せず、ownerのprofile／receiptを参照する。Pack固有の入力規模と期待Gameplayは宣言してよいが、性能不足時に敵、authoritative object、Damage、Ruleを黙って削らない。

## 8. Updateとmigration

Pack updateはinstalled base、Projectで適用済みのinstance、人間／AIによる現在のProject変更を使う三者比較Diffで行う。Templateの再展開でhuman override、AI変更、Save field、Stable IDを上書きしない。

Patch／minor updateはmanifestが宣言した互換範囲とmigration stepを検証する。Major updateはoffline migrator、Preview、明示承認、Save／Replay／Packageの再Qualificationを必須とし、Runtime互換shimまたは旧名aliasを生成しない。field renameでfield identityを変えず、削除済みfield IDを再利用しない。

update transactionが失敗した場合は新revisionをcommitせず、旧Pack version、registry、Project Document、last-valid artifactを維持する。Package済みGameへnetworkからPackを暗黙更新しない。data-only Runtime content更新はAsset packageの署名、Target、rollback規則へ従い、Native codeを含めない。

## 9. Removal

Removal前にProject source、applied instance、Save field、Asset closure、Pack dependency、Package、migration requirementを検査する。live referenceまたはSave dependencyがあるPackを強制削除せず、依存closureと解消操作をPreviewする。

RemovalはProject変更、Cook／Package再生成、registry mutationを別々のcommit境界として扱い、中間失敗時に既存Projectを開ける状態を維持する。Pack removalで共有Capability、別PackのTemplate instance、人間作成Assetを所有推測により削除しない。

## 10. Failureとdiagnostic

| Failure | 結果 |
|---|---|
| Manifest、signature、hash不正 | Installを拒否し、既存registryを維持 |
| dependency cycle／version intersectionなし | closure全体を拒否し、cycleまたは競合rangeを列挙 |
| Engine contract／Target不適合 | `IncompatiblePackVersion`として拒否し、shimを生成しない |
| required Capabilityなし | Apply／Cookを拒否し、CapabilityとTargetを列挙 |
| Project identity／Input／Save／UI／Asset conflict | Conflict Diffを提示し、部分適用しない |
| ChangeSet precondition不一致 | `ApplyDomainPackChangeSet`全体を拒否し、最新revisionから再生成 |
| Native sourceを含む | Native build／reviewへ隔離し、data applyだけでactivationしない |
| update migration失敗 | 新revisionをcommitせず、旧versionとProjectを維持 |
| removal dependencyあり | Removalを拒否し、dependency closureを列挙 |
| 未有効Capability要求 | blocking gapまたは有効な代替として返し、成功placeholderを作らない |

Diagnosticはpack／application identity、version／hash、Project revision、Target、Capability、dependency path、conflict path、failed stage、evidence、remediationを含む。秘密、credential、署名秘密鍵、unbounded Source本文を含めない。

## 11. Fixtures

Domain Pack contractのfixtureは次を最低限含む。

- canonical manifest round-trip、hash一致、unknown／duplicate／missing ref、invalid identity／SemVer
- dependency DAG、cycle、diamond、version range conflict、required／optional Capability、Target intersection
- install／apply／activationの成功と各stageの原子的rollback
- 複数Packのidentity、Input、Save、UI、semantic tag、Asset path conflict
- AI vocabularyからの質問、Requirement、Assumption、Capability gap、Test生成
- manual edit→AI edit→update三者比較でoverride、Stable ID、Save fieldを維持
- patch／minor／major migration、offline migrator失敗、last-valid維持
- live Project／Save／Asset dependencyがあるremoval拒否と、closure解消後の成功
- Native source templateが通常Project applyから分離されること
- 未有効CapabilityをAI、Cook、Packageが選択または成功扱いにしないこと

具体的なreference compositionとして[Shooter reference Pack](shooter.md)を使う。Shooter固有のWeapon、Projectile、Damage、Vital、Pickup、Score、Encounter、Save／Replay契約は同文書だけが所有する。

## 12. 外部根拠と採用判断

- [Unreal Engine Game Features and Modular Gameplay](https://dev.epicgames.com/documentation/unreal-engine/game-features-and-modular-gameplay-in-unreal-engine)
- [Unity ScriptableObject](https://docs.unity3d.com/6000.5/Documentation/Manual/class-ScriptableObject.html)
- [Godot Nodes and Scenes](https://docs.godotengine.org/en/stable/getting_started/step_by_step/nodes_and_scenes.html)
- [O3DE Gems](https://docs.o3de.org/docs/user-guide/gems/)

外部資料は機能、data、Scene、Editor拡張を選択可能な単位へ分離する先例の確認に使う。Miraikanaiの`DomainPack`は外部Plugin形式との互換を目標にせず、任意binaryをloadしないMCD package、Project ChangeSet、Capability Gate、AI vocabulary、reference scenarioを一体化した独自契約である。
