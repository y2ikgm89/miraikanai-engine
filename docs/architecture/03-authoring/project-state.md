# Miraikanai Engine Authoring Model／Project State規約

- 文書ID: mirakan.arch.project-state
- 状態: review
- 正本範囲: Project aggregate、Authoring Document、ProjectRevision、ProjectChangeSetV1のdomain schema／意味／transaction、Target readiness envelope、Commit、Source／Derived境界、Undo／Redo、外部編集、Recovery
- 非正本範囲: MCD共通Envelope／projection／codegen、命名・Project配置、Asset lifecycle、Editor表示、Gameplay System、Native ABI／Build、Runtime ECS storage／identity、Runtime Package binary、Save／Replay、Runtime scheduling。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Architecture Governance](../01-governance/architecture-governance.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core architecture](../02-foundation/core-architecture.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Naming／Project layout](../02-foundation/naming-project-layout.md)、[Asset lifecycle](asset-lifecycle.md)、[Editor UI Framework](editor-ui-framework.md)、[Editor Workspace UX](editor-workspace-ux.md)、[Gameplay programming model](gameplay-programming-model.md)、[Native game module](native-game-module.md)、[Runtime ECS](../04-runtime/entity-component-system.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Persistence／Save](../04-runtime/persistence-save.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Performance／capacity](../04-runtime/performance-capacity.md)、[World／Scene／Space／Cell](../06-rendering/world.md)、[UI／Text／Localization／Accessibility](../07-platform/ui-text-localization-accessibility.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論

Miraikanai Engineの正規Project状態は、Editor widget、Scene Tree表示、AI会話、Runtime World、生成済みC++ binaryのいずれでもない。Schema検証可能なAuthoring Document集合と、単調増加する`ProjectRevision`が正本である。

AI、Editor GUI、人間の手動編集、CLI、MCP、外部IDEは同じ`ProjectChangeSetV1`を提案する。状態を確定できるのはC++ `AuthoringCommandGateway`だけであり、外側のactive MCD Operationと全change primitiveを検証し、一つのrevisionとして原子的にCommitする。部分成功、暗黙補正、Editor内部objectの直接serializeを禁止する。

本書は次を独自に所有する。

- Project aggregateとDocument境界
- field-level ID、reference、revision、lock
- ChangeSet change primitiveとtransaction
- Source file、snapshot、journal、Undo／Redo、Recovery
- 外部編集とAI編集の競合
- Runtime packageへのcompile入力境界
- Target別`TargetReadinessV1`と`TargetBlockedReasonRegistryV1`。性能測定とEvidence freshnessの意味は各Ownerを参照する

## 2. 決定権と対象外

| 主題 | 正本 |
|---|---|
| Project Document、World Model、`ProjectChangeSetV1`のdomain schema／change primitive意味／transaction、Commit、Undo、Recovery | 本書 |
| MCD型、Operation共通Envelope、Error、Schema projection、Codegen | [Executable contracts](../02-foundation/executable-contracts.md) |
| ID、thread、directory、serialization基礎 | [Core architecture](../02-foundation/core-architecture.md)と各Foundation Owner |
| pointer／lease／allocatorの公開・保存・capture規則 | [Memory／Pointers](../02-foundation/memory-pointers.md)の`PointerMemoryConsumerBindingV1` |
| Runtime ECS storage／Entity identity／query／lease／structural transaction | [Runtime ECS](../04-runtime/entity-component-system.md) |
| Runtime World Root／Section image、Package binary、loader | [Runtime Package](../04-runtime/runtime-package.md) |
| Save／Replay record、persistent identity projection、reconstruction | [Persistence／Save](../04-runtime/persistence-save.md) |
| Simulation Advance、phase、queue、Asset activation boundary | [Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md) |
| AI権限、承認、Source sandbox、Promotion | [AI Security／Approval](../01-governance/ai-security-approval.md) |
| Editor panel、workspace、製品操作、人間工学 | [Editor Workspace UX](editor-workspace-ux.md) |
| Editor Widget、Semantic Snapshot、UI eventからtyped Commandへの変換 | [Editor UI Framework](editor-ui-framework.md) |
| Localization Catalog schema、BCP 47、source locale／fallback、Editor／Game Catalog境界 | [UI／Text／Localization／Accessibility](../07-platform/ui-text-localization-accessibility.md) |
| Game System Spec、Implementation Set、System Bundle、二段階Activation | [Gameplay programming model](gameplay-programming-model.md) |
| World、Scene、Space、Topology、Partition Intent、Procedural World、Map Presentation | [World／Scene／Space／Cell](../06-rendering/world.md) |

本書はGitをProject database、Undo system、runtime content storeとして必須化しない。Git連携は任意の外部version-control機能であり、Commitの成否はGit状態へ依存しない。共同リアルタイム編集、CRDT、branch merge UI、networked multi-user sessionはC3であり、C1／C2のChangeSet契約へ含めない。

## 3. Project aggregate

### 3.1 正規Document

| Document | 役割 | ID／revision |
|---|---|---|
| `ProjectManifest` | Project identity、Runtime entry point、Target、Package、Capability、Document index | Projectに一つ、`project_id`、`project_revision` |
| `RuntimeEntryPointDocumentV1` | `RuntimeEntryPointV1` Source、branch、selector／activation policy参照 | `entry_point_id`、document revision |
| `RuntimeTargetSelectorDocumentV1` | Runtime Entryが選択可能なexact Target Profile集合 | `selector_id`、document revision |
| `RuntimeEntryActivationPolicyDocumentV1` | readiness timeout、failure／cancel／deactivation semantics | `policy_id`、document revision |
| `GameSpecDocument` | Genreに依存しない要求、system、content、test、budget、style lock | `game_spec_id`、document revision |
| `WorldDocument` | exact `WorldSpaceProfileRefV1`、Scene／optional spatial topology参照、global composition、persistent entity、Source Intent root | `world_id`、document revision |
| `SceneDocument` | collaborative edit shard identity、Shard index、global setting、Composition Recipe root。Gameplay LevelまたはStreaming Cellではない | `scene_id`、document revision |
| `SceneEntityShardDocument` | 一つのSceneに属するbounded Entity record集合 | `shard_id`、document revision |
| `WorldTopologyDocument` | Space、transition edge、optional activation entryの論理Graph | `topology_id`、document revision |
| `SpatialPartitionIntentDocument` | Target非依存のresidency／grouping／priority Intent | `partition_intent_id`、document revision |
| `ProceduralWorldDefinitionDocument` | generator、seed policy、constraint、bound、fallback | `procedural_world_id`、document revision |
| `MapPresentationDocument` | minimap／world map／marker／fogの非authoritative UI Source | `map_presentation_id`、document revision |
| `SystemImplementationSetDocument` | active Game System ref、Implementation Variant、Target selection、configuration | `system_implementation_set_id`、document revision |
| `UiDocument` | UI tree、style、binding、navigation、localization key | `ui_document_id`、document revision |
| `LocalizationCatalogDocument` | Localization namespace、key、entry、messageのsource。schema正本は[UI／Text／Localization／Accessibility](../07-platform/ui-text-localization-accessibility.md#11-localization)の`LocalizationCatalog` | `localization_catalog_id`、document revision |
| `GameplayDefinitionDocument` | Rule、state、task、typed Capability参照 | Definition StableId、document revision |
| `AssetMetadataDocument` | Source identity、import settings、license、provenance、tag | Asset StableId、document revision |
| `VisualStyleProfileDocument` | 表現四軸、Material、Lighting、Camera、Post、UI style | Profile StableId、document revision |
| `NativeGameModuleManifest` | Project C++ source root、Capability、access、build contract | Module StableId、manifest revision |
| `TargetProfileDocument` | Platform、quality、distribution、content delivery、budget | Profile StableId、document revision |
| `DecisionLedgerDocument` | 判断値、由来、理由、approval、lock、依存 | Entry StableId、document revision |
| `TestScenarioDocument` | Preconditions、input、oracle、budget、Target | Scenario StableId、document revision |

`LocalizationCatalogDocument.source_locale`は各Catalogが宣言するGame／Project contentの原文localeであり、Editor表示locale、AI返答locale、OS localeから推測または同期しない。Projectは日本語その他のlocaleをsourceにでき、Editor Preferenceを変更してもCatalog DocumentまたはProject revisionを変更しない。Editor自身のCatalogはProject Document indexへ登録せず、Game Packageへ混入させない。

共通headerの`display_name`、Asset／Entity名、台詞、Localization source message、User comment、Promptその他のUser／Project原文は入力されたNFC UTF-8と宣言localeを保持する。AIまたはEditorは英語の正規技術語彙を理由に自動翻訳、transliteration、canonical Englishへの置換を行わない。翻訳は対象Localization entryまたはUserが明示した別Fieldへのtyped ChangeSetとしてだけ提案できる。

`ProjectManifest`はDocument本文を埋め込まず、`DocumentRef { stable_id, document_kind, relative_path, content_hash, schema_version }`だけを持つ。Authoring Document間の参照はStableIdで行い、相対path、配列index、表示名を意味参照に使用しない。

`ProjectManifest.runtime_entry_point_refs`は1～64件のexact `DocumentRef<RuntimeEntryPointDocumentV1>`を必須とし、各refの`stable_id`、`document_kind`、`schema_version`、`content_hash`をDocument indexと照合する。`RuntimeEntryPointV1.target_selector_ref`はexact `DocumentRef<RuntimeTargetSelectorDocumentV1>`、`activation_policy_ref`はexact `DocumentRef<RuntimeEntryActivationPolicyDocumentV1>`へ解決し、IDだけ、表示名、path、latest revisionを参照にしない。World Space Profileは[World](../06-rendering/world.md)の`WorldDocumentV1.world_space_profile_ref`にだけ保持し、Project ManifestまたはRuntime Entryへglobal／copied dimension fieldを追加しない。World、Scene、Topology、Stageを全Project共通の必須Documentにしない。owner-typed Pack DocumentはCoreのclosed `document_kind`へ追加せず、登録済みowner namespaceを持つ`DocumentRef`としてDocument indexへ投影する。

### 3.1.1 `RuntimeEntryPointV1`

```text
RuntimeEntryPointV1
  entry_point_id
  entry_kind: world | ui | headless
  target_selector_ref
  default_for_selected_targets: bool
  world_ref: WorldDocumentRef | null
  ui_document_ref: UiDocumentRef | null
  startup_game_system_refs[]
  activation_policy_ref
```

`RuntimeEntryPointDocumentV1`は共通Document headerと上記`RuntimeEntryPointV1`を一件だけ持つ。selectorとactivation policyも同様に共通headerを持つ正規Project Sourceであり、Project Manifestへ本文を埋め込まない。

```text
RuntimeTargetSelectorV1
  selector_id
  selector_version
  selector_hash: RuntimeTargetSelectorHashV1
  target_profile_refs[1..64]

RuntimeTargetSelectorHashPayloadV1
  selector_id
  selector_version
  target_profile_ref_count: uint32
  target_profile_refs[1..64]:
    exact DocumentRef<TargetProfileDocument>

RuntimeEntryActivationPolicyV1
  policy_id
  policy_version
  policy_hash
  readiness_timeout_ms: uint32[1..120000]
  failure_semantics: reject_activation_keep_last_valid | fault_session_reverse_teardown
  cancel_semantics: preparing_before_first_publish | not_cancelable
  explicit_deactivation_semantics: graceful_reverse_teardown | immediate_reverse_teardown
```

`target_profile_refs[]`はexact `DocumentRef<TargetProfileDocument>`をStable ID byte順、同IDならschema version／content hash byte順でcanonicalizeし、duplicate Stable ID、同ID別hash、missing／removed Target、schema／hash不一致を拒否する。wildcard、tag、表示名、platform名、active Targetの現在値を使うlookupはselector schemaに存在しない。

`RuntimeTargetSelectorHashPayloadV1`はSource selectorから`selector_hash`を投影しない別MCD typeであり、`target_profile_ref_count == target_profile_refs.length`を必須にする。`RuntimeTargetSelectorHashV1 = SHA-256(ASCII "MIRAKAN_RUNTIME_TARGET_SELECTOR_V1" || uint32_be(length(MCD canonical bytes of RuntimeTargetSelectorHashPayloadV1)) || MCD canonical bytes)`とする。domain、payload byte length、count、各DocumentRefの型付きField境界により連結曖昧性を作らず、hash payload自身にhash Fieldがないためself-exclusionを暗黙Ruleにしない。selector ID、version、count、Target refのID／kind／path／content hash／schema versionのどれか一つだけを変えた場合も別hashにし、旧hash再利用を拒否する。

各Runtime Entry系Documentのidentityは三重に複製して別々に解決せず、保存時に次のequalityを必須とする。

```text
DocumentRef.stable_id
  == common Document header.document_id
  == payload.entry_point_id | payload.selector_id | payload.policy_id
```

三値は同じUUIDv7 byte列でなければならない。createではGatewayが一つのIDを発行して三箇所へ同時投影し、update／migrationでは既存IDを維持する。cross-document ID、payloadだけのID差替え、self-assertした別Project ID、表示名／pathからの再解決を`MIRAKAN-PROJECT-RUNTIME_ENTRY_IDENTITY_MISMATCH`でChangeSet全体rejectする。

hashの意味を次へ一意に固定する。

- `RuntimeEntryPointSemanticHashV1 = SHA-256(ASCII "MIRAKAN_RUNTIME_ENTRY_POINT_SEMANTIC_V1" || uint32_be(length(MCD canonical bytes of RuntimeEntryPointV1)) || MCD canonical bytes)`。payloadはhash Fieldを持たず、Field／array countはcanonical bytesへ明示する。
- `RuntimeTargetSelectorV1.selector_hash`は直前の`RuntimeTargetSelectorHashPayloadV1`式だけを参照し、別の短縮式、Document hash、Target集合だけのhashを定義しない。
- `RuntimeEntryActivationPolicyV1.policy_hash = SHA-256(ASCII "MIRAKAN_RUNTIME_ENTRY_ACTIVATION_POLICY_V1" || uint32_be(length(MCD canonical bytes of policy payload excluding policy_hash)) || MCD canonical bytes)`。唯一のself-exclusionは`policy_hash`自身である。
- 共通headerの`content_hash`は§3.2どおり、`content_hash`だけを除外したheaderとpayload全体のDocument hashであり、上記semantic hashと同一視しない。
- Compile Manifestの`selected_runtime_entry_point_hash`は厳密に`RuntimeEntryPointSemanticHashV1`である。選択Documentのexact content hashは`selected_runtime_entry_point_ref.content_hash`に保持し、どちらも照合する。

activation policyはreadiness期限、失敗後のsession処理、graceful／immediateな明示deactivation要求だけを所有する。session stop／faultではpolicy値に関係なくbranch activation setのreverse dependency teardownを常時必須にする。System Graphのdependency、activation順、reverse teardown順を再宣言せず、Scheduling ownerが確定したbranch activation setへclosed semanticsを適用する。

tagged validationは次へ固定する。

- `world`: `world_ref`を厳密に1件、`ui_document_ref=null`、`startup_game_system_refs[0..128]`。参照WorldはScene 0件、Topology nullでもよいが、exact `WorldSpaceProfileRefV1`を必須にし、そのProfile hashをWorld Document hash closure内で照合する。Runtime EntryはProfileを複写しない。
- `ui`: `ui_document_ref`を厳密に1件、`world_ref=null`、`startup_game_system_refs[0..128]`。World／World Space Profileを要求しない。
- `headless`: `startup_game_system_refs[1..128]`、`world_ref=null`、`ui_document_ref=null`。surfaceとWorld／World Space Profileを要求しない。

Commit対象`project_revision`のactive `TargetProfileDocument`集合と、`default_for_selected_targets=true` entryが参照するselector集合の和集合はset equalityで完全一致し、各Targetはexactly once被覆されなければならない。default 0件、default 2件以上、uncovered／inactive extra Target、`world_ref`／`ui_document_ref`／surface・spatial fieldのbranch外混在を拒否し、優先順位や登録順で補正しない。`default_for_selected_targets=false`のentryは同じTarget selectorで重複でき、benchmark、menu、game、server等の明示選択肢を共存させる。明示entry選択でもselected Targetが当該selectorのexact memberでなければならない。startup systemは全branchで許可し、branch conflictにしない。

selector documentのrevision／hashまたはTarget Profile集合が変わると、そのselectorを参照する全entry、default coverage result、Compile Manifest、Cooked Runtime Packageをinvalidateする。旧entry hashや別Target membershipを流用せず、同じProject revisionから再materializeする。

旧単数root sceneを検出したProjectは`MIRAKAN-PROJECT-RUNTIME_ENTRY_MIGRATION_REQUIRED`でcurrent load／compileをfail closedにし、正規Projectへ暗黙変換しない。`operation.project.runtime_entry.migrate_root_scene@1`は[Executable contracts §8.1.2](../02-foundation/executable-contracts.md#812-conditional-legacy-migration-evidence-gate)のconditional legacy migrationで、current MCD／Manifest／Service／Provider／MCP／alias集合はexact `[]`である。実在する旧Project Manifest／Root Sceneのschema bytes、source Contract Set／Owner／Named Algorithm／Foundation Closure、immutable fixtureを束縛したsigned inventoryが成立した将来のatomic activationだけが、参照Sceneを含む明示World、`entry_kind=world`、Targetごとのexact default entryへ変換できる。Runtimeの暗黙default、`Level` alias、`ui`／`headless`への近似変換はactivation後も禁止する。

| Diagnostic ID | 条件 |
|---|---|
| `MIRAKAN-PROJECT-RUNTIME_ENTRY_MIGRATION_REQUIRED` | 旧root sceneが残り、明示migrationが未Commit |
| `MIRAKAN-PROJECT-RUNTIME_ENTRY_INVALID` | schema、ref、count、activation policy不正 |
| `MIRAKAN-PROJECT-RUNTIME_ENTRY_TARGET_UNRESOLVED` | selectorがunknown／inactive Targetへしか解決しない |
| `MIRAKAN-PROJECT-RUNTIME_ENTRY_DEFAULT_AMBIGUOUS` | default 0件または2件以上 |
| `MIRAKAN-PROJECT-RUNTIME_ENTRY_BRANCH_FIELD_CONFLICT` | entry kindとbranch fieldが不一致 |
| `MIRAKAN-PROJECT-RUNTIME_ENTRY_DANGLING_REFERENCE` | entry／selector／policy／Targetのexact DocumentRefがmissing／removed |
| `MIRAKAN-PROJECT-RUNTIME_ENTRY_DOCUMENT_HASH_MISMATCH` | DocumentRefのcontent hashが現在revisionと不一致 |
| `MIRAKAN-PROJECT-RUNTIME_ENTRY_SEMANTIC_HASH_MISMATCH` | entry／selector／policyのOwner固有semantic hashがcanonical payloadと不一致 |
| `MIRAKAN-PROJECT-RUNTIME_ENTRY_SCHEMA_MISMATCH` | entry／selector／policyのschema versionまたはref kind不一致 |
| `MIRAKAN-PROJECT-RUNTIME_ENTRY_EXPLICIT_TARGET_MISMATCH` | 明示選択Targetがentry selector集合のmemberでない |
| `MIRAKAN-PROJECT-RUNTIME_ENTRY_IDENTITY_MISMATCH` | DocumentRef、header、payloadのUUIDv7 identityが一致しない |

### 3.1.2 Runtime Entryのclosed Operation Catalog

Project ownerがcurrentで許可するOperation集合は次のexact六refだけである。各要素は`McdContractRefV1 {id, version=1, contract_set_hash}`で、MCD共通Envelope、input／output、pure pre／postcondition policy、authority、risk、side effect、idempotency、transaction、closed Diagnostic set、timeout、rate limit、audit、Provider exposure、Receiptの正本は[Executable contracts §8.1](../02-foundation/executable-contracts.md#81-project-runtime-entryruntime-scopeの正規operation登録)だけが所有する。Project ownerはこれらのFieldを再宣言または補完しない。

```text
RuntimeEntryOperationCatalogRefV1
  operation_refs[6]:
    {id=operation.project.runtime_entry.create, version=1, contract_set_hash}
    {id=operation.project.runtime_entry.update, version=1, contract_set_hash}
    {id=operation.project.runtime_target_selector.create, version=1, contract_set_hash}
    {id=operation.project.runtime_target_selector.update, version=1, contract_set_hash}
    {id=operation.project.runtime_entry_activation_policy.create, version=1, contract_set_hash}
    {id=operation.project.runtime_entry_activation_policy.update, version=1, contract_set_hash}
```

Operation RegistryのProject owner集合と上記集合はID／version／Contract set hashのset equalityを必須とする。Root Scene migration候補を含むmissing／extra／duplicate、wrong kind、stale version／hash、pre／post policyのwrong kind／missing／staleはCatalog materializationを全rejectする。suffixなしalias、自由JSON write、selector／policyをentry本文へ埋め込むOperationを登録しない。

current六input typeのexact fieldを次へ固定する。`common`を展開したGenerated schemaは下記のFieldとpresence ruleへ閉じ、`additionalProperties=false`であり、継承やuntyped extensionとして扱わない。状態を変更するR2は署名済みApprovalまたは同Scopeの署名済みPredelegationを厳密に一つ持つ。どちらもない入力をcanonical omissionとして受理せず`MIRAKAN-APPROVAL-REQUIRED`で拒否する。コードブロック末尾のRoot Scene migration schemaはpost-activation destination templateであり、current Type LocalRef／external ref／Generated schema集合はexact `[]`である。

```text
RuntimeEntryMutationCommonInputV1
  input_type_ref: McdContractRefV1(kind=type)
  operation_ref: McdContractRefV1(kind=operation)
  project_ref: exact {project_id, expected_project_revision, document_set_hash}
  contract_set_hash
  operation_intent_hash
  request_hash
  idempotency_key
  preview_policy_ref: McdContractRefV1(kind=policy)
  validation_policy_ref: McdContractRefV1(kind=policy)
  mutation_authorization_binding: exact MutationAuthorizationBindingV2

type.project.runtime_entry.create_input
  common
  allocation_scope
  relative_path
  payload_draft_without_entry_point_id
  payload_draft_hash
  target_selector_ref
  activation_policy_ref

type.project.runtime_entry.update_input
  common
  current_document_ref
  expected_document_revision
  before_content_hash
  before_semantic_hash
  after_payload_with_same_entry_point_id
  after_semantic_hash

type.project.runtime_target_selector.create_input
  common
  allocation_scope
  relative_path
  payload_draft_without_selector_id
  payload_draft_hash
  canonical_target_profile_refs[1..64]

type.project.runtime_target_selector.update_input
  common
  current_document_ref
  expected_document_revision
  before_content_hash
  before_selector_hash
  after_payload_with_same_selector_id
  after_selector_hash

type.project.runtime_entry_activation_policy.create_input
  common
  allocation_scope
  relative_path
  payload_draft_without_policy_id
  payload_draft_hash

type.project.runtime_entry_activation_policy.update_input
  common
  current_document_ref
  expected_document_revision
  before_content_hash
  before_policy_hash
  after_payload_with_same_policy_id
  after_policy_hash

type.project.runtime_entry.migrate_root_scene_input
  common
  migration_plan_ref: RootSceneMigrationPlanRefV1

RootSceneMigrationPlanRefV1
  plan_id: StableId
  plan_version: positive uint32
  plan_hash: SHA-256

RootSceneMigrationPlanV1
  plan_id: StableId
  plan_version: positive uint32
  project_ref: exact {project_id, expected_project_revision, document_set_hash}
  legacy_inventory_ref/hash: exact LegacyMigrationInventoryRefV1
  source_foundation_definition_closure_ref:
    exact FoundationDefinitionClosureRefV1
  retained_source_project_manifest_mcd_ref:
    exact McdContractRefV1(kind=type, contract_set_hash=source root)
  retained_source_root_scene_mcd_ref:
    exact McdContractRefV1(kind=type, contract_set_hash=source root)
  legacy_project_manifest_artifact_ref/hash
  legacy_root_scene_artifact_ref/hash
  active_target_profile_refs[1..64]
  document_mutations[4]:
    exact one RootSceneDocumentMutationV1 for each
      world | runtime_target_selector |
      runtime_entry_activation_policy | runtime_entry
  create_branch_count: uint32[0..4]
  default_entry_bindings[1..64]:
    exact {target_profile_ref,
           runtime_entry_plan_local_ref: RootScenePlanLocalRefV1(kind=runtime_entry),
           is_default=true}
  plan_hash: SHA-256

RootScenePlanLocalRefV1
  plan_id: same StableId as enclosing plan
  document_kind:
    world | runtime_target_selector |
    runtime_entry_activation_policy | runtime_entry
  local_ordinal: uint32[1..4]

RootSceneDocumentMutationV1
  plan_local_ref: RootScenePlanLocalRefV1
  mutation:
    create:
      allocation_intent:
        exact {document_kind, allocation_scope, relative_path,
               requested_identity=null}
      payload_draft_without_stable_id_ref/hash
    | update:
      current_document_ref: exact DocumentRef including content_hash
      before_payload_semantic_hash:
        Worldではcanonical omission |
        Owner固有Runtime Entry／Selector／Policy hash
      preserved_identity: exact StableId equal to current_document_ref.stable_id
      after_payload_with_preserved_identity_ref/hash
```

current全inputは選択したnamed typeだけを使用し、anonymous sibling shapeを許可しない。`operation_intent_hash`と`request_hash`は[Executable contracts §8.1](../02-foundation/executable-contracts.md#81-project-runtime-entryruntime-scopeの正規operation登録)が所有する`MIRAKAN_OPERATION_INTENT_V2 -> MutationAuthorizationBindingV2 -> MIRAKAN_OPERATION_REQUEST_V2`の唯一のDAGをそのまま使い、本書では別式を定義しない。Operation、Project、Preview／Validation policy、idempotency、全Domain fieldがintentへ入り、binding確定後のfinal requestはintent hashとexact authority evidenceを含む。create inputは`project_id`、expected Project revision、idempotency key、identity Fieldを持たないpayload draft、draft hash、allocation scope、relative path、selector／policy exact refsを持つ。GatewayがIDを発行し完成payload semantic hashを出力する。update inputはexact current `DocumentRef`、expected Document revision、before content／semantic hashとidentity固定済みafter payloadを持つ。selector create／updateはcanonical Target ref集合、policy create／updateは全closed semanticsを持つ。

Root Scene migration候補をactivateする場合、`RootSceneMigrationPlanV1`は上記signed Inventoryとsource Foundation ClosureをDomain preimageへ必須にする。二retained source MCD ref、二legacy Artifact、Inventory、Closureのsource Contract set rootはbyte equalityで、Closureから同時代Owner Registry／Named Algorithm Registry／schema bytes／decoder Artifactへexact解決しなければならない。bare `legacy_source_closure_hash`、current schemaへの別名解決、未知旧bytesの推測decodeを拒否する。`plan_hash = SHA-256(ASCII "MIRAKAN_ROOT_SCENE_MIGRATION_PLAN_V1" || uint32_be(len(plan bytes excluding plan_hash)) || plan bytes)`であり、`RootSceneMigrationPlanRefV1`は完成recordの`plan_id`／`plan_version`／`plan_hash`から外部materializeする。Record自身へhash付きPlanRefを埋め戻さない。四`plan_local_ref`は同じplan ID、上記kind順のordinal 1～4で重複なく、各document kindをexact一件持つ。`create_branch_count`はcreate discriminator件数と一致し、Stable-ID allocation intent／mappingもcreate branchだけから同順でexact同数0～4件を導出する。update branchはcurrent ref／before hash／preserved identityを必須にし、allocation intent／mappingを持たない。Gatewayは別ID、別path、五件目、欠落、update identity差替え、Targetごとの暗黙entry追加を行わない。各active Targetは`default_entry_bindings[]`に厳密に一件あり、そのtyped plan-local refは唯一のruntime entry mutationを指す。四Documentの完成ref、create mapping集合、Target→default mappingをPreview、Prepared Candidate、postcondition、Executable Contracts §8の`PublicCommitClosureV1.domain_commitment`、signed Receipt、Public Publication Markerで同一にする。公開順序は同正本の`private Marker read-back → secret-free PublicCommitClosure candidate → signed Receipt → PublicCommitClosure＋Public Marker＋四Documentのatomic CAS`だけを使う。Inventory、source Closure、legacy Artifact、Plan hash、branch discriminator、create allocationのどれかが変われば別intent／requestとなり、部分migrationを公開できない。この段落は§8.1.2 gate成立後のacceptance criteriaで、current callable behaviorではない。

`type.project.runtime_entry.mutation_result`は`disposition=committed | rejected`のtagged unionである。committed branchだけが`PublicPublicationMarkerV1` readback後のbefore／after exact Project ref、exact signed `mutation_receipt_ref/hash`、Preview／Validation／Public Marker ref／hashを持ち、Public MarkerとReceiptの同一`PublicCommitClosureRefV1`を完成`PublicCommitClosureV1`へ解決する。`RuntimeEntryMutationReceiptV1`は[Executable contracts §8.1](../02-foundation/executable-contracts.md#81-project-runtime-entryruntime-scopeの正規operation登録)のpayload＋canonical `MirakanSignedRecordV1` wrapperをexact reuseし、Domain signer／key／algorithm／signature Fieldをinline定義しない。current六Operationの`affected_documents[]`はexact一件である。WorldはDocument content hashだけ、entry／selector／policyはcontent hashと各Owner固有semantic hashを記録する。Root Scene候補が将来activateされた場合だけWorld／selector／policy／entryのexact四branchとPlan create branch数に等しいallocation mappingを許し、update branchは0 mappingとする。rejected branchだけが、選択active Operation recordの`errors[]`とValidator reachable error setの双方に存在する四Field `DiagnosticCodeRefV1`を1～64件持つ。Registry外Diagnostic、ID／code／version／hashの一部一致、string error、current Operationの複数Document、candidate未activation時のPlan refを拒否する。失敗時はProject revision、Document index、default coverage、Compile Manifest、last-valid Runtime Packageを一切変更しない。

current positive fixtureはentry／selector／policy create→save→reload→update→compileの三identity／二hash照合を検査する。negative fixtureは三箇所のidentity差を各一件、selector ID／version／count／Target exact refの一Fieldだけを変更して旧`selector_hash`を再利用するmutation、payload semantic hash mismatch、Document content hash mismatch、Target count／array length mismatch、self-hash循環を作るpayload、stale revision、selector／policy cross-kind ref、Operation pre／post policyのwrong kind／missing／stale ref、Diagnostic ID／code／version／hash mismatch、current Operationへのmigration Plan混入、署名Receipt前public stateをそれぞれ単独原因で拒否し、全経路でrevision不変を検査する。Root Scene候補のfixture current ref集合はexact `[]`であり、signed Inventory成立後のatomic activation fixtureだけが実legacy bytes、source wrong-root／Owner／Algorithm／schema／decoder、create 0～4件の四Document tagged union、allocation count equality、部分migration拒否、signed Receipt後のpublic resultを検査する。

`WorldStreamingPlanV1`、Navigation Artifact、HLOD、Cooked Gameplay Package、generated System Catalog／Dependency GraphはDerived Artifactであり、正規Document種別へ追加しない。CreatorまたはAIがDerived Artifactを直接編集した変更をGatewayは拒否する。

### 3.2 共通Document header

全Documentは次を必須とする。

| Field | 型／規則 |
|---|---|
| `schema_id` | MCDの固定ID |
| `schema_version` | `uint32`。現在版だけをEditorが保存 |
| `document_id` | RFC 9562 UUIDv7 |
| `document_kind` | closed enum |
| `document_revision` | `uint64`、Document変更ごとに+1 |
| `project_id` | 親Project UUIDv7 |
| `created_revision` | 最初に追加された`ProjectRevision` |
| `modified_revision` | 最後に変更された`ProjectRevision` |
| `display_name` | NFC UTF-8、1～128 grapheme、参照には使わない |
| `relative_path` | Project root相対の正規path。absolute、`..`、case-fold衝突を拒否。`DocumentRef.relative_path`と同じ値 |
| `content_hash` | 自己Fieldを除く共通header＋Document payloadのMCD canonical SHA-256 |
| `source_provenance_ref` | Provenance record StableId |
| `extension_fields` | 登録済みnamespaceだけ。未知namespaceは保存時に拒否 |

浮動小数点はfiniteだけを受理する。map keyはMCDが定めるcanonical順、setはStableId byte順、配列は意味的順序を持つfieldだけに使う。Document hashはJCSへ変換した意味modelではなく、実行可能契約規約のMCD canonical binary encodingからSHA-256で計算する。

### 3.3 Decision Ledgerの有効性

`DecisionLedgerDocument`の各Entryは、説明文だけでなく次を必須とする。

| Field | 型／規則 |
|---|---|
| `decision_id` | StableId |
| `status` | `active \| needs_review \| superseded \| rejected` |
| `value`／`reason` | typed valueとNFC UTF-8 |
| `source` | `human \| ai_recommendation \| ai_assumption \| engine_default` |
| `approved`／`locked` | 信頼済みHostが付与するbool。AI出力から昇格しない |
| `applies_to` | Document、Entity、field、Requirement、CapabilityのStable reference集合 |
| `evidence_refs` | 根拠Artifactと、そのrevision／hash |
| `decision_dependencies` | 他Decision StableId |
| `validity_predicates` | MCDで登録済みPredicate ID＋typed argument |
| `created_revision`／`confirmed_revision` | `ProjectRevision` |
| `superseded_by` | optional Decision StableId |

GatewayはChangeSetの影響closureから、`applies_to`、`evidence_refs`、`decision_dependencies`、`validity_predicates`を決定論的に照合する。成立条件が変わる場合、提案ChangeSetに`InvalidateDecision`または新根拠を伴う`ReconfirmDecision`がなければ`MIRAKAN-DECISION-INVALIDATION_REQUIRED`で全体を拒否し、必要change primitiveとDecision IDを返す。GatewayがEntryを黙って`needs_review`へ変更してはならない。`locked=true`のDecisionへ影響する変更は、別Authorityが承認した`UnlockDecision`を同じtransactionへ含めない限り拒否する。

### 3.4 Target readiness

Project revisionのTarget別実行可否は次の一型だけで表す。

```text
TargetReadinessV1
  target_profile_ref
  project_revision
  input_closure_hash
  state: predicted | blocked | qualified
  blocked_reason_ref: string | null
  technical_qualification_receipt_ref:
    TechnicalQualificationReceiptRefV1
    | canonical omission
```

`input_closure_hash`はSource revision、Scale intent、Representation Plan、Contract set、Toolchain lock、Target Profile、Device generation、Qualification policyのcanonical hash closureである。Target Readiness record／hash、Qualification Receipt／refをこのpreimageへ含めない。`predicted`は安全なPlanを生成できるが当該closureの実測Receiptがなく、`blocked_reason_ref=null`、`technical_qualification_receipt_ref`をcanonical omissionする。`blocked`は現在入力ではPlay／Cook／Shipping promotionを許可できず、登録済みnon-null `blocked_reason_ref`とReceipt ref omissionを必須とする。`qualified`は先に固定した同じ`input_closure_hash`へ`subject_hash`がbyte equalityで束縛されたfresh `TechnicalQualificationReceiptV1`のexact `TechnicalQualificationReceiptRefV1`を検証後にmaterializeするdownstream projectionで、`blocked_reason_ref=null`、Receipt ref presentを必須とする。Refのreceipt ID／subject hash／signed wrapper refはAI Verification ownerのwrapperへexact解決し、hash-only値、bare ID、別closureのfresh Receiptを拒否する。Receipt subjectへTarget Readiness record／hashを戻さない。状態とField presenceが一致しないRecordを拒否する。

`TargetBlockedReasonRegistryV1`は`reason_id`、`diagnostic_ref`、`owner_document_id`、`blocking_scope`、`recovery_gate_ref`を持ち、本書がenvelopeとID一意性、各Domain Ownerが意味と回復条件を所有する。初期共通entryは次の2件である。

| reason_id | diagnostic_ref | Owner | recovery gate |
|---|---|---|---|
| `optimization_required` | `diagnostic.performance.optimization-required` | `mirakan.arch.runtime-performance-capacity` | 同じGameplay fidelity floorを維持するRepresentation Planを再生成し、Target実機fixtureを再測定する |
| `performance_envelope_unqualified` | `diagnostic.performance.envelope-unqualified` | `mirakan.arch.runtime-performance-capacity` | C1 entity／population envelopeをTarget Profile実機の`IntegratedScaleFixtureV1`で校正し、fresh Target-device Receiptを発行する |

特にC1 entity／population数値が未校正のTargetを推測値で`predicted`または`qualified`にせず、`state=blocked`、`blocked_reason_ref=performance_envelope_unqualified`とする。Performance OwnerがTarget Profile、fixture、input trace、Device generation、測定Receiptを揃えて値を確定するまで解除しない。Mobileのpixel budget／render baselineは別のTarget Profile入力であり、本readiness stateへ合成または置換しない。

wire値はlower snake caseだけを受理する。`Predicted`、`Blocked`、`Qualified`、`OptimizationRequired`等のPascalCaseと、Capability activation専用`not_activated`をTarget readinessへ混入させない。

## 4. World Model

### 4.1 EntityとComposition

`SceneEntityShardDocument`のEntity recordを次で固定する。

| Field | 型／規則 |
|---|---|
| `entity_id` | StableId。Sceneを越えて一意 |
| `parent_entity_id` | optional StableId。同一Scene内だけ |
| `sibling_order_key` | `uint64`。表示と明示順序だけに使用 |
| `name` | NFC UTF-8、同名可 |
| `enabled` | bool |
| `lifecycle` | `scene_owned \| persistent \| streamed` |
| `tags` | 登録済みTag StableId、最大64 |
| `components` | Component Type IDごとに最大一つ |
| `recipe_instance` | optional Recipe ID＋parameter block |
| `editor_metadata` | fold、color、note等。Runtime compile対象外 |

ComponentはMCDの`authoring_types`で定義し、`component_type_id`、`component_schema_version`、typed field mapを持つ。Component間pointer、Editor object address、native handle、vendor型を保存しない。親cycle、存在しない参照、Dimension不一致、同時付与禁止Component、Capability不足はsemantic validation errorである。

Composition RecipeはEntity subtreeの再利用sourceである。InstanceごとにRecipe内部Local IDからProject StableIdへの対応表を保持し、Recipe更新時は三者比較Diffを生成する。Instance固有overrideはMCDで`overridable=true`のfieldだけ許可し、未承認のoverride消失を禁止する。

### 4.2 Projection

Hierarchy、Outliner、Inspector、Graph、Table、Timeline、UI Designer、AI要約は同じDocument集合から生成するread modelである。Projection固有stateは`EditorUserState`へ保存し、Project gameplay stateへ混入させない。

Projectionは次を保証する。

- StableIdを非表示にしても内部selection keyとして維持する。
- filter／sort中のdrag操作は表示indexでなくStableIdをtyped change primitiveへ渡す。
- 変更は必ずtyped change primitiveへ変換し、Projection cacheを先に変更しない。
- Commit成功後に新revisionから再投影する。
- stale read modelからのchange primitiveは`RevisionMismatch`で拒否する。

人間、AI、keyboard automation、assistive technologyが同じ対象を指せるよう、選択状態の機械可読projectionを`AuthoringSelectionContextV1`へ固定する。

```text
AuthoringSelectionContextV1
  project_id
  project_revision
  contract_set_hash
  view_kind
  primary_stable_id optional
  selected_stable_ids[0..1024]
  world_ref optional
  scene_ref optional
  owner_typed_feature_selection_refs[0..128]
  viewport_bounds optional
  field_mask
  target_profile_ref optional
  lock_refs[0..128]
  source_hashes[1..1024]
  omitted_ranges[0..128]
  continuation
```

`AuthoringSelectionContextV1`はCommit済みDocumentと明示的な`EditorUserState`から生成するread-only／DisposableなContextであり、Project正本またはUndo対象ではない。`primary_stable_id`、World／Scene参照とowner-typed Feature selectionは表示名、Hierarchy path、row index、screen coordinateから推測せず、存在確認済みStableId、owner、revisionを使う。Stage selection等はFeature Packが登録したoptional projectionであり、Coreのclosed `document_kind`へ追加しない。AIへ渡すContext、Editor command、UI Automation semantic actionは同じContext hashを参照し、操作時には対象StableIdとexpected Document revisionを再指定する。Contextがstale、対象がomitted、lock情報が欠落、またはSource／Derived区分が不明な場合はchange primitiveへ昇格しない。

Canonical formでは`selected_stable_ids`と`owner_typed_feature_selection_refs`をduplicateなしのStable ID canonical orderにし、同じ集合から一つのContext hashだけを得る。`selected_stable_ids`が空なら`primary_stable_id`はnull、非空なら`primary_stable_id`はその集合に含まれるexact一件でなければならない。削除後にprimaryだけが失効した場合はcanonical orderの先頭を新primaryにし、集合が空ならnullにする。keyboard focus、hover、range anchor、focus column、scroll anchor、inline draft、Panel pinはView／Panelの`EditorUserState`であり、本Contextへ混入させない。UI Frameworkが複数種類の選択を束ねる場合も、本ContextのAuthoring target集合をAsset、Graph element、Diagnostic、Runtime objectへ拡張してはならない。

Architecture Governanceが所有する`ArchitectureExplainProjectionV1`はProject Stateの正本ではなく、Commit済みSourceとexact registry closureから生成されるread-only／Disposableなconsumer projectionである。`authoring.explain_architecture`は`scope`、非空`field_mask`、optional `target_profile_ref`、exact `project_revision`を要求し、別revisionへのfallbackを行わない。応答の`omitted_ranges`または署名付き`continuation`が示す未取得範囲をEvidence済みとして扱わず、stale revision、continuation条件不一致、必要Evidence欠落では説明を確定しない。

このqueryまたは自然言語要約からProjectを直接変更してはならない。後続ChangeSetは解決済みcanonical concept、対象StableId、typed change primitive、expected Document revisionを正規Gatewayへ再指定する。Projection entry、Owner名、外部用語、表示path、summary textをCommit可能なidentityまたはchange primitiveとして受理しない。

### 4.3 大規模SceneのShard、Index、Slice

`SceneDocument`は論理aggregateであり、Entity本文を直接無制限に埋め込まない。C1から一つ以上の`SceneEntityShardDocument`をStableId参照し、各Shardは4,096 Entity recordまたはcanonical encoded 8 MiBの早い方を上限とする。単一Entityが上限を超える場合はComponent schema違反としてrejectし、巨大blobをShardへ埋め込まない。

- Shardは`scene_id`、`shard_id`、`partition_mode = stable_id_range | spatial_cell`、partition key、Entity StableId範囲、record count、content hashを持つ。
- Entity StableIdはShard移動、Scene rename、spatial cell変更で変えない。親、Recipe、Component参照はShard内indexでなくStableIdを使う。
- `SceneDocument.entity_set_root_hash`はShard配置ではなく、Entity StableId順のcanonical Entity leafからMerkle計算する。Re-shardだけでGameplay上のsemantic diffを作らない。
- Re-shardも正規Source変更なので一つの`ProjectRevision`としてCommitするが、Diffは`storage_only`とEntity意味変更を分離する。
- Shardを跨ぐ親cycle、reference、lock、Decision、Recipe invariantはScene aggregate全体で検証する。

Entityの永続化ownerは、そのrecordを含むShardの`scene_id`で厳密に一つへ決まる。Transform parent、Outliner folder、owner-typed gameplay membership、Streaming Cell、Data Layer相当のtagから永続化ownerを推測しない。Scene間移動は`MoveEntityToScene` Domain change primitiveだけが、移動元／移動先Scene revision、移動rootと全descendant、移動先のoptional parent、参照、lock、Recipe override、boundsを検証してsubtree recordを移す。subtree内部のparent関係とStableIdを維持し、移動rootの新parentは移動先Scene内またはnullに限定する。owner-typed gameplay membershipまたはRuntime Cell割当は同primitiveの暗黙副作用にせず、必要なSource変更を同じChangeSetへ別のtyped change primitiveとして明示する。

`AuthoringContextIndexV1`はCommit済みrevisionから生成するDisposableな派生Indexであり、正本ではない。`project_revision`、`contract_set_hash`、Document root hash、index schema version、利用するtokenizer ID／manifest hash集合を固定し、次を索引化する。

- StableIdからDocument／Shard／fieldへの位置。
- inbound／outbound reference、Requirement、Capability、Decision、lockのclosure。
- Sceneのspatial bounds、tag、component type、modified revision。
- field maskごとのcanonical byte数とProvider tokenizer別の実測token数。

Commit後は旧Indexをstaleにし、変更Shardと参照closureだけをcopy-on-write更新してから新revisionとしてpublishする。要求revisionのIndexがReadyでない場合、別revisionの結果を返さず`MIRAKAN-AUTHORING-INDEX_NOT_READY`とretry hintを返す。

AI、Editor、CLIへ返す`SceneSliceV1`は、query ID、Project revision、anchor StableId、選択Shard、field mask、dependency depth、各source hash、選択理由、omitted range、continuation cursorを持つread-only projectionである。任意byte位置で切ったJSON、表示順index、要約だけをChangeSetの根拠にしない。Sliceからのchange primitiveもStableIdとexpected Document revisionを必須とし、Gatewayが対象Shardへroutingする。

## 5. ProjectChangeSetV1

### 5.1 Envelope

`ProjectChangeSetV1`のdomain schema、change primitive意味、transaction／Commit規則は本節だけが所有する。[Executable contracts](../02-foundation/executable-contracts.md#8-operation定義)の外側MCD Operation共通Envelopeと生成規則を消費するが、suffixなし最新版aliasまたは別Envelopeは作らない。

```text
ProjectChangeSetV1
  schema_version: uint32
  change_set_id: UUIDv7
  request_id: UUIDv7
  project_id: UUIDv7
  base_project_revision: uint64
  author_kind: human | ai | editor_tool | external_tool | migrator
  authorization_envelope_hash: sha256
  intent_summary: string
  declared_impact: ImpactSummary
  change_primitives: ProjectChangePrimitiveV1[1..4096]
  preconditions: ProjectPrecondition[0..1024]
  evidence_refs: StableId[0..128]
```

ChangeSet全体のcanonical encoded sizeは8 MiB以下とする。Asset binary、C++ source本文、巨大配列を埋め込まず、許可済みStaging fileのcontent hashとrelative pathを参照する。

AI／Editor／CLI／MCPが同じChangeSet bytesをProposal、Validation、Preview、Commitへ運ぶためのtarget carrierを次へ固定する。これは[Executable Contracts §21.1](../02-foundation/executable-contracts.md#211-既存domain文書から回収した未登録operation候補)末尾の`planning.operation_family.authoring_changeset_execution`を将来atomic ActivationするためのDomain schemaであり、current MCD Operation、Manifest、Service、Policy、Validator、Diagnostic、Receipt、Signer、Provider／MCP投影集合はexact `[]`、Capability stateは`not_activated`である。

```text
ProjectRevisionBindingV1
  project_id: UUIDv7
  project_revision: uint64
  document_set_sha256: Sha256DigestV1

ProjectChangeSetArtifactRefV1
  change_set_schema_ref: McdContractRefV1(
    id=type.project.change_set, version=1, contract_set_hash)
  change_set_id: UUIDv7
  project_id: UUIDv7
  base_project_revision: uint64
  artifact_ref: ArtifactRefV1(
    artifact_kind=project_change_set, schema_version=1)
  canonical_change_set_sha256: Sha256DigestV1
  primitive_set_sha256: Sha256DigestV1

ProjectChangeSetProposalResultV1
  disposition:
    kind: proposed
      change_set_ref: ProjectChangeSetArtifactRefV1
      question_record_refs[0..128]
      assumption_record_refs[0..128]
      diagnostic_refs[0..64]
    | kind: question_required
      question_record_refs[1..128]
      diagnostic_refs[0..64]
    | kind: rejected
      diagnostic_refs[1..64]

ProjectChangeSetValidationSubjectV1
  validation_id:
    urn:mirakan:project-changeset-validation:sha256:<lowercase-hex-64>
  change_set_ref: ProjectChangeSetArtifactRefV1
  project_revision_binding: ProjectRevisionBindingV1
  contract_set_hash: Sha256DigestV1
  target_profile_refs[0..64]
  reachable_primitive_type_refs[1..4096]
  operation_to_primitive_coverage_sha256: Sha256DigestV1
  input_closure_sha256: Sha256DigestV1
  result: pass | fail
  diagnostic_refs[0..64]
  validated_at

ProjectChangeSetValidationReceiptV1
  subject: ProjectChangeSetValidationSubjectV1
  signed_record: MirakanSignedRecordV1(
    purpose=project_changeset_validation)

ProjectChangeSetPreviewSubjectV1
  preview_id:
    urn:mirakan:project-changeset-preview:sha256:<lowercase-hex-64>
  change_set_ref: ProjectChangeSetArtifactRefV1
  project_revision_binding: ProjectRevisionBindingV1
  diff_artifact_ref: ArtifactRefV1(
    artifact_kind=project_changeset_diff, schema_version=1)
  impact_summary_ref: ArtifactRefV1(
    artifact_kind=project_changeset_impact_summary, schema_version=1)
  invalidated_consumer_manifest_ref: ArtifactRefV1(
    artifact_kind=project_invalidated_consumer_manifest, schema_version=1)
  input_closure_sha256: Sha256DigestV1
  disposition: previewable | blocked
  diagnostic_refs[0..64]
  previewed_at

ProjectChangeSetPreviewReceiptV1
  subject: ProjectChangeSetPreviewSubjectV1
  signed_record: MirakanSignedRecordV1(
    purpose=project_changeset_preview)

ProjectTargetProfileRefV1
  target_profile_id: StableId
  target_profile_version: uint32
  target_profile_content_hash: Sha256DigestV1

ProjectSourceChangedPathRecordV1
  source_kind: native_module | project_shader
  project_id: UUIDv7
  project_relative_path: NfcUtf8StringV1(1..1024 bytes)
  change_kind: create | modify | delete
  before_blob_sha256: null | Sha256DigestV1
  after_blob_sha256: null | Sha256DigestV1
  path_record_content_hash: Sha256DigestV1

ProjectSourceChangedPathRefV1
  exact ArtifactRefV1(
    artifact_kind=project_source_changed_path, schema_version=1)

BrokerRecomputedSourceDiffV1
  source_kind: native_module | project_shader
  project_id: UUIDv7
  before_source_tree_sha256: Sha256DigestV1
  after_source_tree_sha256: Sha256DigestV1
  changed_path_refs[1..4096]: ProjectSourceChangedPathRefV1
  diff_content_hash: Sha256DigestV1

NativeModuleSourceTaskArtifactRefV1
  exact ArtifactRefV1(
    artifact_kind=project_native_source_task, schema_version=1)

NativeModulePatchProposalArtifactRefV1
  exact ArtifactRefV1(
    artifact_kind=project_native_patch_proposal, schema_version=1)

NativeModuleSourceRevisionArtifactRefV1
  exact ArtifactRefV1(
    artifact_kind=project_native_source_revision, schema_version=1)

ProjectShaderSourceTaskArtifactRefV1
  exact ArtifactRefV1(
    artifact_kind=project_shader_source_task, schema_version=1)

ProjectShaderPatchProposalArtifactRefV1
  exact ArtifactRefV1(
    artifact_kind=project_shader_patch_proposal, schema_version=1)

ProjectShaderSourceRevisionArtifactRefV1
  exact ArtifactRefV1(
    artifact_kind=project_shader_source_revision, schema_version=1)

ProjectSourceRegistrationIntentV1
  registration_intent_id:
    urn:mirakan:project-source-registration-intent:sha256:<lowercase-hex-64>
  project_id: UUIDv7
  base_project_revision: uint64
  primitive_id: UUIDv7
  source:
    {source_kind: native_module,
     source_task_ref: NativeModuleSourceTaskArtifactRefV1,
     patch_proposal_ref: NativeModulePatchProposalArtifactRefV1,
     candidate_source_revision_ref:
       NativeModuleSourceRevisionArtifactRefV1}
    | {source_kind: project_shader,
       source_task_ref: ProjectShaderSourceTaskArtifactRefV1,
       patch_proposal_ref: ProjectShaderPatchProposalArtifactRefV1,
       candidate_source_revision_ref:
         ProjectShaderSourceRevisionArtifactRefV1}
  revision_transition_kind: create | update
  before_source_revision_ref:
    null | NativeModuleSourceRevisionArtifactRefV1
         | ProjectShaderSourceRevisionArtifactRefV1
  broker_recomputed_diff_ref:
    exact ArtifactRefV1(
      artifact_kind=broker_recomputed_source_diff, schema_version=1)
  before_source_tree_sha256: Sha256DigestV1
  candidate_source_tree_sha256: Sha256DigestV1
  target_profile_refs[1..64]: ProjectTargetProfileRefV1
  toolchain_lock_sha256: Sha256DigestV1
  registration_intent_content_hash: Sha256DigestV1

ProjectSourceRegistrationIntentRefV1
  exact ArtifactRefV1(
    artifact_kind=project_source_registration_intent,
    schema_version=1)

ProjectSourcePromotionSubjectV1
  promotion_id:
    urn:mirakan:project-source-promotion:sha256:<lowercase-hex-64>
  project_id: UUIDv7
  base_project_revision: uint64
  project_change_set_ref: ProjectChangeSetArtifactRefV1
  prepared_candidate_ref: PreparedCandidateRefV1
  registration_intent_ref: ProjectSourceRegistrationIntentRefV1
  source:
    source_kind: native_module
      source_task_ref: NativeModuleSourceTaskArtifactRefV1
      patch_proposal_ref: NativeModulePatchProposalArtifactRefV1
      revision_transition:
        kind: create
          promoted_source_revision_ref:
            NativeModuleSourceRevisionArtifactRefV1
        | kind: update
          before_source_revision_ref:
            NativeModuleSourceRevisionArtifactRefV1
          promoted_source_revision_ref:
            NativeModuleSourceRevisionArtifactRefV1
      build_receipt_refs[1..64]:
        NativeModuleBuildReceiptRefV1
    | source_kind: project_shader
      source_task_ref: ProjectShaderSourceTaskArtifactRefV1
      patch_proposal_ref: ProjectShaderPatchProposalArtifactRefV1
      revision_transition:
        kind: create
          promoted_source_revision_ref:
            ProjectShaderSourceRevisionArtifactRefV1
        | kind: update
          before_source_revision_ref:
            ProjectShaderSourceRevisionArtifactRefV1
          promoted_source_revision_ref:
            ProjectShaderSourceRevisionArtifactRefV1
      build_receipt_refs[1..64]:
        ProjectShaderBuildReceiptRefV1
  broker_recomputed_diff_ref:
    exact ArtifactRefV1(
      artifact_kind=broker_recomputed_source_diff, schema_version=1)
  before_source_tree_sha256: Sha256DigestV1
  promoted_source_tree_sha256: Sha256DigestV1
  test_receipt_refs[1..64]: CandidateTestReceiptRefV1
  independent_review_receipt_ref:
    IndependentSourceReviewReceiptRefV1
  code_owner_assignment_ref: CodeOwnerAssignmentRecordRefV1
  code_owner_approval_ref: CodeOwnerApprovalRecordRefV1
  candidate_root_sha256: Sha256DigestV1
  target_profile_refs[1..64]: ProjectTargetProfileRefV1
  toolchain_lock_sha256: Sha256DigestV1
  promoted_at

ProjectSourcePromotionReceiptV1
  subject: ProjectSourcePromotionSubjectV1
  signed_record: MirakanSignedRecordV1(
    purpose=project_source_promotion)

ProjectSourcePromotionReceiptRefV1
  exact MirakanSignedRecordRefV1(
    purpose=project_source_promotion)

ProjectSourcePromotionAuthorizationEntryV1
  primitive_id: UUIDv7
  registration_intent_ref: ProjectSourceRegistrationIntentRefV1
  promotion_receipt_ref: ProjectSourcePromotionReceiptRefV1

ProjectSourcePromotionAuthorizationBindingV1
  binding_id:
    urn:mirakan:project-source-promotion-authorization:sha256:<lowercase-hex-64>
  binding_version: positive uint32
  project_id: UUIDv7
  base_project_revision: uint64
  project_change_set_ref: ProjectChangeSetArtifactRefV1
  prepared_candidate_ref: PreparedCandidateRefV1
  candidate_root_sha256: Sha256DigestV1
  entry_count: positive uint32
  entries[1..256]: ProjectSourcePromotionAuthorizationEntryV1
  binding_content_hash: Sha256DigestV1

ProjectSourcePromotionAuthorizationBindingRefV1
  exact ArtifactRefV1(
    artifact_kind=project_source_promotion_authorization_binding,
    schema_version=1)
```

`artifact_ref.sha256`は完成`ProjectChangeSetV1` canonical bytesへ解決し、`canonical_change_set_sha256`はASCII `MIRAKAN_PROJECT_CHANGE_SET_V1`、全Fieldを宣言順にMCD canonical encodeしたbytesを`uint32_be` length framingしてSHA-256する。両digestはbyte equalityでなければならない。`primitive_set_sha256`はprimitiveをdependency DAGのcanonical topological order、同順位を`primitive_id` UUID bytes順に並べ、各完成primitive canonical bytesを同じframingでhashする。filesystem順、入力配列順、表示順、JSON text、intent summaryだけをidentityにしない。RefのID／Project／revision／各hashの一Fieldでもartifactと不一致なら全routeを拒否する。

Proposalのtagged branch外Fieldはcanonical omissionする。Question／Assumption／Diagnostic／Target／primitive type ref集合は各typed refのcanonical tuple順、重複禁止である。Validation IDはASCII `MIRAKAN_PROJECT_CHANGESET_VALIDATION_SUBJECT_V1`、Preview IDは`MIRAKAN_PROJECT_CHANGESET_PREVIEW_SUBJECT_V1`と各自己IDを除く完成subject canonical bytesからSHA-256を計算して上記URNへ投影する。任意UUID、時刻、署名wrapperをID derivationへ使わない。Activation work itemはDiff、Impact Summary、Invalidated Consumer Manifestのclosed entry schema、exact ID derivation、Local Schema Catalog、purpose、Signer、validity、revocation、Receipt ref型を同じContract set transactionで固定する。`operation.authoring.changeset.commit`の成功結果は新しいCommit Receipt型を作らず、§5.3の完成`PublicCommitClosureV1`、`PublishedDomainReceiptV2`、`PublicPublicationMarkerV1`、before／after Project refを再利用する。三型はExecutable Contracts §8のnested common schemaであり、Project Operation／MCD Type／Contract Set member件数を増やさない。

Promotion IDはASCII `MIRAKAN_PROJECT_SOURCE_PROMOTION_SUBJECT_V1`と自己IDを除く完成subject canonical bytesから導出する。`project_change_set_ref`はPrepared Candidateが束縛するreceipt-free ChangeSet、`prepared_candidate_ref`と`candidate_root_sha256`はprepromotion Build／Testのexact Candidate、`registration_intent_ref`は対象source registration primitiveのtyped argumentとbyte equalityにする。`source`は`source_kind`をdiscriminatorとするclosed tagged unionで、Native／ShaderのTask、Proposal、Source revision、Build Receiptを上記のexact kind／schema version／purpose以外へ差し替えられない。Native Source revision refは[Native game module §9.4](native-game-module.md#94-ai-source-taskpatch-proposalpromotion-carrier)の`NativeModuleSourceRevisionV1`、Shader Source revision refは[Project Shader §8](../06-rendering/project-shader.md#8-aieditor-operationとcontext)の`ProjectShaderSourceRevisionV1`へ解決し、Artifact refのSHA-256を完成record bytesへ照合する。IntentのProject／base revision、Task／Proposal、Candidate Source revision、Diff、tree、Target、ToolchainとPromotion subjectの対応Fieldをbyte equalityにし、Promotion時にChangeSet、Prepared Candidate、Intentを再生成しない。

`ProjectSourceChangedPathRecordV1`と`BrokerRecomputedSourceDiffV1`はclosed objectである。pathはNFC正規化したProject-relative `/` 区切りとし、absolute path、drive／UNC prefix、空segment、`.`／`..`、NUL、symlink escapeを拒否する。`create`は`before_blob_sha256=null`かつafter non-null、`modify`は両方non-nullかつ非同値、`delete`はbefore non-nullかつafter nullだけを許す。`path_record_content_hash = SHA-256(ASCII "MIRAKAN_PROJECT_SOURCE_CHANGED_PATH_V1" || uint32_be(len(closed MCD canonical bytes excluding path_record_content_hash)) || closed bytes)`とし、`ProjectSourceChangedPathRefV1.sha256`は同Fieldを含む完成record bytesのSHA-256へ一致させる。

Diffの`changed_path_refs[]`は各refを完成Path recordへ解決し、`{project_relative_pathのNFC UTF-8 bytes, change_kind enum order=create|modify|delete, ref.sha256 bytes}`のtupleでstrict昇順にする。同じProject-relative pathのduplicateはref／change kindが異なっても拒否する。全Path recordの`source_kind`／`project_id`はDiffと一致し、Diffのbefore／after treeはPath集合をcanonical path順へ適用した再計算結果と一致しなければならない。`diff_content_hash = SHA-256(ASCII "MIRAKAN_BROKER_RECOMPUTED_SOURCE_DIFF_V1" || uint32_be(len(closed MCD canonical bytes excluding diff_content_hash)) || closed bytes)`とし、`ArtifactRefV1(artifact_kind=broker_recomputed_source_diff,schema_version=1).sha256`は同Fieldを含む完成Diff bytesのSHA-256へ一致させる。Proposalのtyped ref集合、source kind、Project、before／after treeはこのDiff recordとbyte equalityにし、Worker申告path、表示path、scopeだけ、missing／extra／duplicate／未正規順pathを受理しない。

`ProjectSourceRegistrationIntentV1`はPromotion Receiptを含まないimmutable semantic inputである。`registration_intent_content_hash`はASCII `MIRAKAN_PROJECT_SOURCE_REGISTRATION_INTENT_V1`と自己ID／hash Fieldを除くclosed recordのMCD canonical bytesを`uint32_be` length framingして計算し、`registration_intent_id`は同digestから上記URNへ投影する。Artifact refのSHA-256はID／content hashを含む完成record bytesと一致させる。`revision_transition_kind=create`では`before_source_revision_ref=null`かつcandidate revision 1、`update`ではsource kindに一致するbefore refをnon-null、同Stable IDのcandidate revisionをexact `N+1`にする。IntentのProject／base revision、Task／Proposal、Candidate Source revision、Broker Diff、before／candidate tree、Target、Toolchainは外側Task、ChangeSet、Prepared Candidate、prepromotion Build／Testとbyte equalityにし、Receipt、Approval、Promotion、Commit Markerをhash preimageへ戻さない。

`revision_transition.kind=create`では`before_source_revision_ref`をcanonical omissionし、`base_project_revision`時点に同じSource Stable IDが存在しないこと、promoted recordの`source_revision=1`を必須にする。`kind=update`ではbefore refを必須にし、そのresolved Project／Source Stable ID／revision／tree hashがTaskのbaseと一致し、promoted refは同じStable IDの厳密な`source_revision=N+1`でなければならない。createでbefore ref、updateでbefore ref省略、空ref／zero revision／latest aliasによる補完、branch外Field、異種refを拒否する。

`NativeEmptySourceTreeHashV1 = SHA-256(ASCII "MIRAKAN_EMPTY_NATIVE_SOURCE_TREE_V1" || uint32_be(0))`、`ProjectShaderEmptySourceTreeHashV1 = SHA-256(ASCII "MIRAKAN_EMPTY_PROJECT_SHADER_SOURCE_TREE_V1" || uint32_be(0))`とする。createの`before_source_tree_sha256`はsource kindに対応するこのexact値、updateはbefore refが解決するtree hashとbyte equalityにし、Field自体は両branchで必須とする。空directoryのfilesystem列挙結果、空文字列、zero digest、別domainのempty hashを代用しない。

`promoted_source_revision_ref`はprepromotion Build／Testが検査したimmutable Candidate Source revisionと同じArtifact refであり、Promotion時にSource bytes、record、hash、revisionを再生成しない。Promotion成功は同refをProject Source Registryへ到達可能化する状態遷移であり、別hashのcopy、Build後の修正、同revision別treeを拒否する。

`target_profile_refs[]`はTarget Profile ID／version／content hash順、`build_receipt_refs[]`はresolved Target tuple／Source kind／Source revision ref／signed record hash順、`test_receipt_refs[]`はresolved Target tuple／Test Plan ref／signed record hash順にcanonicalizeし、同じTarget Profile ref、同じ完成wrapper、同じTest Plan／Target pairのduplicateをそれぞれ拒否する。Build／Test refは[Core architecture](../02-foundation/core-architecture.md)のcanonical aliasを再利用し、Native Buildはexact `purpose=operation_receipt:operation.build.request_native_module`、Shader Buildは`purpose=operation_receipt:operation.build.request_project_shader`、Candidate Testは`purpose=operation_receipt:operation.test.request_run`以外を拒否する。Promotionが受理する全Build Receiptは`source_authorization.kind=prepromotion_candidate`、全Candidate Test Receiptが参照するGame Candidate Build Receiptは`source_build_closure.kind=prepromotion_project_source`でなければならない。各Receipt subjectが束縛するTask、Proposal、Broker Diff、Assignment、Project、`prepared_candidate_ref`／Candidate root、Target、Toolchain、Candidate Source revision、Registration IntentをPromotion subjectとbyte equalityにする。Promotion Receiptを要求するBuild／Game Candidate branchを前提Evidenceにせず、`receipt-free Intent／ChangeSet -> Prepared Candidate -> prepromotion Build／Candidate Test -> independent review／Code Owner Approval -> Promotion -> late authorization binding -> Prepared Commit Envelope`のDAGを循環なしで固定する。`independent_review_receipt_ref`はSource Worker／Builder／Code Ownerと分離されたreviewerの同じBroker Diff、`code_owner_assignment_ref`と`code_owner_approval_ref`は[AI Security／Approval §9.4](../01-governance/ai-security-approval.md#94-code-owner-assignmentとapproval)の完成subjectと署名済みwrapperへ解決し、ApprovalのAssignment、Diff、Source revision、Build／Review集合を本subjectとexact一致させる。hashだけ、purposeなし、別purposeの有効署名、裸subject refを受理しない。

Promotion signerはAI／Worker／Providerではなく、[AI Security／Approval](../01-governance/ai-security-approval.md)が所有するpurpose専用trusted internal Serviceである。current Code Owner Assignment、同じBroker Diffへのapproved Code Owner Approval、distinct independent review、全required TargetのBuild／Test Receipt、Project／Candidate／Toolchain／before-after tree equalityを副作用前とpublication直前に再検証する。成功Receipt後だけ対応Source登録primitiveを許可し、Provider／MCPへPromotionを投影しない。

`ProjectSourcePromotionAuthorizationBindingV1`はPromotion後、Project Commit前にだけ生成するcontent-addressed late authorization artifactである。`entries[]`を`primitive_id` UUID bytes順へstrict sortし、ChangeSet内のNative／Shader registration primitive集合とexact set equalityにする。各entryのIntentはprimitiveのtyped argument、Promotion ReceiptのChangeSet／Prepared Candidate／Candidate root／Intent／Source revisionはBindingとbyte equalityで、Receipt resultと署名、Code Owner／Review／Build／Test closure、expiry／revocationを再検証する。`binding_content_hash`はASCII `MIRAKAN_PROJECT_SOURCE_PROMOTION_AUTHORIZATION_BINDING_V1`と自己ID／hash Fieldを除くclosed recordのMCD canonical bytesを`uint32_be` length framingして計算し、`binding_id`を同digestから上記URNへ投影する。Artifact refのSHA-256はID／content hashを含む完成record bytesへ一致させる。source registration primitiveが0件ならBinding Ref集合はexact `[]`、一件以上なら同じChangeSet／Prepared Candidateに対するBinding Ref exact一件だけを`AtomicCommitPlanPayloadV1`、`PreparedCommitEnvelopeV1`、`PreparedReceiptPublicationBindingV1`へ保存する。ReceiptをPrepared Candidateへ後付けする、Candidate hashを再計算する、複数Bindingへ分割する、別Candidate／ChangeSetのReceiptを混ぜる実装を拒否する。

### 5.2 `ProjectChangePrimitiveV1`

`ProjectChangePrimitiveV1`はChangeSet内部だけのtagged mutation unionであり、MCD kind `operation`、`operation.*` logical ID、Owner Manifest row、Service allowlist、Provider／MCP Toolではない。全primitiveは`primitive_id`、closed `primitive_type`、target StableId、typed argument、primitive内依存、expected document revision、declared costを持つ。primitive名からMCD Operation IDを生成せず、外側の完全登録済みMCD OperationだけがChangeSetをauthorization／publication境界へ運べる。

| primitive群 | C1 `primitive_type` |
|---|---|
| Document | `CreateDocument`、`DeleteDocument`、`RenameDocument` |
| Entity | `CreateEntity`、`DeleteEntity`、`ReparentEntity`、`SetSiblingOrder`、`MoveEntityToScene` |
| Component | `AddComponent`、`RemoveComponent`、`SetComponentField`、`ReplaceComponent` |
| Reference | `SetStableReference`、`ClearStableReference` |
| Recipe | `InstantiateRecipe`、`ApplyRecipeUpdate`、`SetRecipeOverride` |
| Gameplay／UI／Style | 各Subsystemが登録するtyped change primitive |
| Game System | `RegisterProjectGameSystemSpec`、`SetSystemImplementationVariant`、`ReplaceSystemConfiguration`。`qualified` Contract／Staging hashだけ |
| World／owner-typed content | Topology、Partition Intent、Procedural、Map Presentationと登録済みowner namespaceの各Domain typed change primitive |
| Asset | `RegisterAssetSource`、`SetImportField`、`ReplaceAssetSourceRevision` |
| Native C++ | `RegisterNativeModuleRevision`。receipt-free `ProjectSourceRegistrationIntentRefV1`とCandidate Source revisionだけ。Promotionはlate authorization bindingで検証 |
| Project Shader | `RegisterProjectShaderModuleRevision`、`RegisterProjectShaderTechniqueRevision`。receipt-free Registration Intent、Candidate Source revision、Technique／Port compatibility closureだけ。Promotion／Target別Buildはlate bindingで検証 |
| Target／Decision | `SetTargetProfileField`、`RecordDecision`、`LockDecision`、`UnlockDecision`、`InvalidateDecision`、`ReconfirmDecision` |

自由形式の`SetJsonPointer`、任意path write、任意C++ symbol call、任意console commandをprimitive unionへ登録しない。複数fieldを不変条件とともに変える変更は一つのDomain primitiveとし、細かな`SetField`列へ分解して中間不整合を作らない。

AIへ公開する全Authoring Capabilityは、MCDで`ai_mutable=true`の全fieldが一つ以上の完全登録済み外側MCD Operationから到達し、そのnamed inputが一つ以上のtyped `ProjectChangePrimitiveV1` branchへ閉じることをContract compilerで証明する。MCD Operation→primitive coverageが100%でないCapabilityはAI Tool catalogへ昇格しない。`RegisterNativeModuleRevision`／`RegisterProjectShaderModuleRevision`／`RegisterProjectShaderTechniqueRevision`のprimitiveはBroker検証済みreceipt-free `ProjectSourceRegistrationIntentRefV1`とCandidate Source revisionを持ち、Step 1～9／`PreparedCandidateV1`のhash preimageへPromotion Receipt、Build／Test Receipt、Code Owner Approvalを含めない。これらの後発EvidenceはStep 10直前に`ProjectSourcePromotionAuthorizationBindingV1`から全source registration primitiveへexact一件ずつ対応付けて検証する。Source本文、Worker自己申告Diff、未昇格artifact、missing／extra／duplicate bindingを受理しない。AI TaskのPath Grantへ正規Authoring JSONのwrite権限を含めず、AIがSource fileを直接変更してcoverageを迂回する経路を作らない。

### 5.3 Commit algorithm

`AuthoringCommandGateway`はAuthoring threadで次を順番どおり実行する。

1. Envelope、size、authorization、Project IDを検証する。
2. `base_project_revision == live_project_revision`を検証する。
3. primitive ID一意性とdependency DAGを検証し、canonical topological orderを作る。
4. MCD schema、enum、range、finite、string、StableId、pathを検証する。
5. 全preconditionとDocument revisionを検証する。
6. 参照整合、cycle、Capability、Target intersection、Decision invalidation、Domain invariantを検証する。
7. 変更後aggregateをcopy-on-write stagingへ構築し、`PreparedCandidateRefV1`を確定する。
8. Authoring aggregate自体のmemory／schema hard budgetとRisk policyを検証する。Runtime Targetのrender、physics、nav、VFX、package予測costは、安全なRepresentation Planがありestimate内でも未実測なら`state=predicted`、現在のPlanでは未達なら`state=blocked`と登録済み`blocked_reason_ref`を候補revisionへ記録する。`qualified`は予測から生成せず、同じ`input_closure_hash`へ束縛されたfresh統合負荷Receiptを照合できた場合だけ維持する。未校正workload envelopeは`blocked_reason_ref=performance_envelope_unqualified`とする。
9. Domain dry-runと必要なbackground validation artifactのhashを照合し、Preview／Validation／Domain Receiptの未発行payloadを作る。schema、safety、boundedness、不変条件の失敗はrejectし、Target performance／capacityだけの未達は`state=blocked`、改善可能なら`blocked_reason_ref=optimization_required`として候補へ記録する。
9a. Source registration primitiveがある場合だけ、ここでCommit algorithmを一時停止し、不変`PreparedCandidateRefV1`に対するprepromotion Build／Test、独立Review、Code Owner Approval、Promotionを完了して`ProjectSourcePromotionAuthorizationBindingRefV1` exact一件を生成する。ない場合はlate authorization Ref集合をexact `[]`にする。
10. `PreparedCandidateRefV1`、late authorization binding Ref集合、未発行Receipt payload、予定after stateを束ねた`PreparedCommitEnvelopeV1`を作り、その不変bytesだけへpostcondition v2を評価して`StagedPostconditionReceiptV1`を得る。BindingはEnvelopeのauthorization closureであってPrepared Candidateまたはstaged after stateを変更しない。
11. 変更Document、inverse change primitive、manifest、journal record、全Prepared Receipt payload、staged after state、`PrivateDurableCommitMarkerV1` payloadを同一temporary transaction directoryへ書き、全fileをflushする。
12. transaction manifestを外部readerから到達不能なprivate durable namespaceへ最後に原子的renameし、private Markerをcommit decisionとしてreadbackする。この時点でlive Project head、Document index、公開Receipt、provider-visible Resultを変更しない。
13. private Marker、Prepared Envelope、全Prepared Receipt payload、request hash、staged postcondition Receipt hashのexact equalityを確認する。次に[Executable Contracts §8](../02-foundation/executable-contracts.md#8-operation定義)のnested common schemaからsecret-free `PublicCommitClosureV1` candidateを作り、`domain_commitment.kind=project_change_set_commit`、exact `project_change_set_ref`、`candidate_root_sha256`、Prepared Candidate、late binding集合、before／after Project、Envelope／private Marker commitmentを固定する。Source registration primitiveが0件でもProject branchとChangeSet／Candidate rootを必須にし、late binding集合だけを`[]`にする。Closureのsemantic hashと完成object SHAを分離してcandidate storeへput-if-absentした後、そのClosure Ref／完成object hashを含むcanonical `PublishedDomainReceiptV2`／`MirakanSignedRecordV1` wrapperを同節の固定materialization key／issued-at／revocation snapshot／key context／deterministic signing profileでreceipt storeへput-if-absentする。
14. signed wrapperをreadbackした後だけ、同じ`PublicCommitClosureV1` body、`PublicPublicationMarkerV1`、after Project head、Document indexを同じexpected predecessorに対する一つのpublic CASでpublishし、Closure、Public Marker、signed Receiptの同一Ref／semantic hash／完成object hashを検証してdomain Resultで返す。Closureだけ、Markerだけ、Project headだけを先行公開しない。
15. `AuthoringContextIndexV1`の旧revisionをstaleにし、変更Shardと参照closureの更新Jobを発行する。
16. Projectionへ`ProjectRevisionCommitted` eventを値として配送する。

1～10の失敗はlive stateと公開Closure／Receiptを変更しない。11～12でProcessが停止した場合、次回起動時にprivate transaction manifest、file hash、journal record、private Markerの四者を検査し、完全なtransactionだけをroll-forwardする。不完全なtemporary directoryまたはMarkerなしpayloadは隔離後に非公開廃棄し、部分復旧しない。private Markerがdurableだがsigned wrapperが未保存の場合は同じimmutable preimageからbyte-identical `PublicCommitClosureV1` candidateとwrapperをexact一度だけmaterializeし、wrapper保存後かつPublic Marker前のcrashは同じClosure＋Marker＋after Projectを同じexpected predecessorへのpublic CASでroll-forwardする。既存Closure／wrapperはbyte equalityで再利用し、別Closure、alternate signature、二重publication、overwrite、public後rollbackを禁止する。postconditionはClosure、private／public Marker、signed Receipt、公開後stateを入力にせず、private Marker、unsigned payload、Closure candidate store、receipt-store単独存在をpublic authorityにしない。

Source変更のprepromotion Build／Candidate Testは、receipt-free Registration Intentを含むlive base Project revision `N`の不変`ProjectChangeSetArtifactRefV1`／`PreparedCandidateRefV1`、Candidate root、未昇格Source revisionに対してStep 9aで実行する。これはSource Promotion専用Evidenceであり最終Packageではない。`BuildProjectRevisionRefV1.project_revision=N`を予定after Project revision `N+1`と同一視せず、prepromotion Receipt、Promotion Receipt、late authorization Binding、Package artifactを`PreparedCandidateV1`またはその`prepared_artifact_refs[]`へ戻さない。Promotion後はBindingをStep 10のEnvelopeへ追加するだけでCandidate／ChangeSet／staged after stateを一byteも変更せず、Step 10～14のpublic CASがexact promoted Source ref、Candidate root、Project ChangeSet、Bindingを`PublicCommitClosureV1.domain_commitment.project_change_set_commit`へ束縛して`N -> N+1`を完成する。Source primitiveが0件でも同Project branch、ChangeSet、Candidate rootは残り、late binding集合だけが空である。そのClosure＋Public Marker発行後だけ、committed Project revision `N+1`を入力に最終Validate／Cook／Game Candidate Build／Candidate Test／Packageを実行する。Package成功はpost-commit Projectを変更せず、失敗時はProject `N+1`とSource promotion headを維持して対象Targetを`blocked`にし、last-valid Packageを新revisionの成功結果へ流用しない。

### 5.4 System／World Bundle

`SystemBundleChangeSetV1`のschema、状態遷移、二段階Activation、Source Promotion後のrecoveryは[Gameplay programming model](gameplay-programming-model.md)だけが所有する。本書はBundleが参照する`ProjectChangeSetV1`と最終Project Commitだけを所有する。Gatewayは検証済みexact hashを受け取り、`RegisterNativeModuleRevision`と`SetSystemImplementationVariant`を同じ`ProjectChangeSetV1`でCommitする。Bundle自体をCommitして正規Documentを迂回せず、Project Commit失敗時にSource repositoryをrollbackしない。

World BundleはStaging SourceからTarget別Streaming／Navigation／LOD／非公開trial Package Artifactを試作し、Topology、playability、budget、failure fixtureを検証してからSource Document群を一つの`ProjectChangeSetV1`へ変換する。trial Artifactはfinal Package／Package Receiptではなく`PreparedCandidateV1.prepared_artifact_refs[]`またはRelease入力へ入れない。Derived Artifactの生成失敗でSource revisionを部分Commitせず、Commit後の非同期再Cook失敗時はSourceを維持して該当Targetを`blocked`へ遷移させ、原因に対応する`blocked_reason_ref`を記録する。

## 6. Source layoutと永続化

Project root全体のPathと命名は[Game Project配置・命名規約](../02-foundation/naming-project-layout.md#5-engine-rootとgame-project-root)を正本とする。Top-level rootは同規約のclosed set（`source/`、`config/`、`packages/`、`derived/`、`intermediate/`、`staging/`、`evidence/`と`mirakan.project.json`）以外を追加しない。旧layout root（`authoring/`、`assets/`、`native/game/`、`.mirakan/`、`build/`）はcanonicalではなく、Gatewayは`LegacyLayoutRoot`としてProject openを拒否する。本書はAuthoring永続化に関係するprojectionを次で固定する。

```text
<project>/
├─ mirakan.project.json
├─ source/
│  ├─ assets/                  # Source Assetとimport設定。AssetMetadataDocumentをAsset IDで併置
│  ├─ worlds/                  # World／Scene／Topology／Partition Intent／Procedural World／Map Presentation
│  ├─ gameplay/
│  ├─ ui/
│  ├─ localization/
│  ├─ native/                  # Native Game Module root
│  ├─ game_spec/
│  ├─ system_implementations/
│  ├─ visual_styles/
│  ├─ targets/
│  ├─ decisions/
│  └─ tests/
├─ config/
├─ packages/
├─ derived/
│  └─ index/                   # AuthoringContextIndex等の派生Index。source control対象外
├─ intermediate/
│  ├─ journal/                 # append-only Commit journal。source control対象外
│  ├─ snapshots/
│  ├─ transactions/            # §5.3のtemporary transaction directory
│  └─ recovery/
├─ staging/                    # Import／AI候補。source control対象外
└─ evidence/
```

- `mirakan.project.json`とAuthoring MCDはUTF-8 without BOM、LF、重複key禁止、comments禁止、trailing comma禁止とする。
- Scene sourceは`source/worlds/scenes/<scene_id>/scene.mirakan.json`と`source/worlds/scenes/<scene_id>/shards/<shard_id>.mirakan.json`へ置き、IDから決定論的にpathを導出する。表示名、cell名、Entity名をpathへ使わない。
- World、Topology、Partition Intent、Procedural World、Map Presentationも`source/worlds/`配下でStable IDから決定論的にpathを導出し、表示名、Region名、Target名をpath identityへ使わない。owner-typed Feature Documentは登録済みowner pathへ置き、Core World pathへ偽装しない。System Implementation Set、Visual Style、Target Profile、Decision、Test Scenarioも同じ規則で各directoryへ置く。
- `.mirakan.json`は人間Diff用sourceであり、Runtimeは直接読まない。
- journal、snapshot、transaction directoryの配置は本書が所有し、Git追跡・配布対象外の`intermediate/`配下へ置く。canonical stateはCommit済み`source/`のDocumentだけであり、journal／snapshotをcanonical sourceへ昇格しない。
- `intermediate/journal/`はChangeSet、base／result revision、before／after hash、inverse change primitive、Receipt参照を持つappend-only recordである。
- 100 Commitまたはjournal 64 MiBの早い方でsnapshotを`intermediate/snapshots/`へ作る。最新2 snapshotと、それ以後のjournalを最低保持する。
- §5.3のtemporary transaction directoryは`intermediate/transactions/<change_set_id>/`であり、roll-forward検査の完了後に削除または隔離する。
- Project source保存成功とAsset／Runtime cook成功を同一transactionにしない。Commit済みsourceからDerived Artifactを非同期に`derived/`へ生成し、失敗時もsource revisionを失わない。
- Auto-saveは未CommitのEditor draftを`intermediate/recovery/<user>/<session>`へ20秒ごと、またはfocus loss時に保存する。正規revisionへ自動Commitしない。

## 7. Undo／Redo、外部編集、競合

Undoは過去fileを上書きする操作ではない。Journalのinverse change primitiveを現在revisionに対する新ChangeSetとして再検証し、新revisionを作る。依存変更により安全に反転できない場合は対象と衝突を表示し、強制適用しない。Redoも同じ規則である。

Undo可能深度は§6のjournal最低保持範囲（最新2 snapshotとそれ以後のjournal）と一致する。保持範囲を超えて破棄済みのjournalへ到達するUndo要求は`UndoHistoryUnavailable`として拒否し、部分的なinverse適用や推測による復元をしない。

外部Editor／IDEの変更はFile watcherが検出し、次を行う。

1. UTF-8、syntax、duplicate key、schemaをStagingで検証する。
2. 最後に認識したbase hash、外部file、現在Project Documentの三者比較を作る。
3. 差分をtyped change primitiveへ変換できる場合だけ`ExternalTool` ChangeSetを生成する。
4. 人間がEditor内で確認するか、事前承認Policyに合致した場合だけCommitする。

同じfieldへの異なる変更、delete対edit、Recipe更新対override、StableId再利用は自動mergeしない。異なるDocument／異なるfieldで不変条件を共有しない変更だけ自動merge候補にでき、結局は新しいbase revisionで全Validatorを再実行する。

## 8. AIと手動編集

- AIは現在revision、関係Document、lock、Capability、Target、budgetを含む`AuthoringContextPackV1`を読む。
- Context選択は`AuthoringContextIndexV1`と署名対象の`AuthoringContextPlanV1`から行い、選択理由、field mask、omitted range、continuation、source hashを失わない。
- Editorで選択中のWorld／Scene／owner-typed content／Entityを会話Contextへ含める場合は`AuthoringSelectionContextV1`を使い、画面pixel、Hierarchy path、表示名だけを対象識別子にしない。
- AIは存在しないStableIdを推測せず、`Create*` change primitiveで新IDを要求する。IDはGatewayが生成して結果mapを返す。
- AIは巨大Sceneを全置換せず、目的に必要なchange primitiveだけを提案する。
- `authoring.search`、`authoring.read`、`authoring.dependencies`、`authoring.diff`は将来のread-only authoring actionを表すplanning tokenであり、Stable ID、MCD Operation ID、current callableではない。対応familyのatomic Activationまではread action集合をexact 0件とし、AIは正規Authoring JSONへ直接writeしない。Activation後もread actionと、完全登録済み外側MCD Operationのnamed inputへ閉じたtyped change primitiveだけを使う。
- 人間がlockしたfield、Document、Entity subtreeをAIは変更できない。
- Editor表示localeまたはAI返答localeの変更をProject変更として扱わず、User／Project原文を暗黙翻訳しない。翻訳Proposalは元Fieldと翻訳先Field、source／target locale、expected Document revisionをtyped change primitiveで明示する。
- Level 0ではAIが質問と仮定をGame用語で提示し、実装語を初心者へ選ばせない。
- 手動Inspector、Graph、Code連携もAIと同じDiff、Validation、Undoを使う。
- AI説明、AI proposal、Engine validation、Commit済み結果を別stateとして表示する。

`AuthoringContextPackV1`と`AuthoringContextPlanV1`のcanonical schemaは本書が所有する。`AuthoringContextPlanV1`はContext選択の署名対象であり、`plan_id`、`project_id`、exact `project_revision`、`contract_set_hash`、参照した`AuthoringContextIndexV1` revision、選択理由、Documentごとのfield mask、source hash、omitted range、optional continuationをfield setとする。`AuthoringContextPackV1`はPlanから決定論的に生成するread-only／DisposableなAI入力projectionであり、`plan_hash`、現在revision、関係Documentの`SceneSliceV1`、optional `AuthoringSelectionContextV1`、lock、Capability、Target、budgetの各参照をfield setとする。両者はProject正本またはUndo対象ではなく、Commit可能なidentityまたはchange primitiveとして受理しない。

Editor／Workspace UXが生成する`ExternalEngineConceptResolutionV1`は入力解決Evidenceであり、Project DocumentまたはChangeSetへ保存しない。Gatewayは外部用語、resolver object、候補配列、外部Scene path／Hierarchy indexをidentityまたはchange primitiveとして受理しない。後続ChangeSetは`selected_concept`のcanonical concept ID、対象StableId、typed change primitive、expected Document revisionを再指定しなければならず、`question_required`または`unsupported`のResolutionからProposalを生成してはならない。

## 9. Runtime compile境界

Runtime compilerはCommit済み`ProjectRevision`だけを入力にし、Editor draft、Projection cache、AI会話を読まない。Compile manifestは次を固定する。

```text
project_id
project_revision
document_set_hash
capability_manifest_hash
target_profile_hash
selected_runtime_entry_point_ref
selected_runtime_entry_point_hash
target_selector_hash
activation_policy_hash
entry_branch_closure_hash
game_system_dependency_graph_hash
system_implementation_set_hash
selected_provider_binding_set_hash: optional
world_document_hash: optional
spatial_topology_hash: optional
world_streaming_plan_hash: optional
ui_document_hash: optional
startup_system_closure_hash: optional
native_module_revision_hash
asset_dependency_root_hash
contract_lock_hash
toolchain_lock_hash
scale_intent_hash
representation_plan_hash
technical_qualification_receipt_ref:
  TechnicalQualificationReceiptRefV1 | canonical omission
```

`selected_runtime_entry_point_ref`、`selected_runtime_entry_point_hash`、`target_selector_hash`、`activation_policy_hash`、`entry_branch_closure_hash`は全branchで共通に必須である。`selected_runtime_entry_point_hash`は§3.1.1のpayload semantic hashだけを意味し、Document content hashと置換しない。選択Runtime Entry closureがowner-typed Provider Binding Documentを一件以上選択する時だけ`selected_provider_binding_set_hash`をcanonical presentとし、0件では省略する。これはGenre固有型をCoreへ埋め込まず、選択Runtime Entryのexact DocumentRef／semantic hash、current Project refのpost-commit revision／document set hash、owner-typed Provider Binding Registry ref／hash、各bindingのexact DocumentRef／content hash、revision非依存binding semantic hash、stable owner identity、implementation System ref、Save／Replay contract refをDocument Stable ID byte順にcanonicalizeした集合hashである。Binding payloadへProject revision／document set hashを埋めず、Projectのunrelated revisionではRegistry／Compile closureだけをrematerializeする。`world` branchでは`world_document_hash`を必須、Topology／Streaming Sourceが存在する時だけ対応hashをcanonical presentとし、`ui_document_hash`を省略する。`ui` branchは`ui_document_hash`を必須にし、World／Topology／streaming hashをcanonical omissionする。`headless` branchはWorld／UI／Topology／streaming hashをcanonical omissionする。

`startup_game_system_refs[]`が1件以上なら`startup_system_closure_hash`はworld／ui／headlessの全branchでcanonical present、0件ならworld／uiだけcanonical omissionとする。headlessはstartup system 1～128件を必須にするため常にpresentである。startup closureは各startup Systemの全transitive System dependency、Implementation Variant ref／hash、State owner relation、Target compatibility resultをcanonical System Graph順で含む。null、0 hash、空配列をomissionの代用にせず、branch外field混在を`MIRAKAN-PROJECT-RUNTIME_ENTRY_BRANCH_FIELD_CONFLICT`で拒否する。

`entry_branch_closure_hash`は次の順序付きcanonical inputだけから計算する。

```text
entry_kind
selected_runtime_entry_point_hash
target_selector_hash
activation_policy_hash
target_profile_hash
game_system_dependency_graph_hash
system_implementation_set_hash
selected_provider_binding_set_hash?
world_document_hash?
ui_document_hash?
spatial_topology_hash?
world_streaming_plan_hash?
startup_system_closure_hash?
```

optional hashは上記tagged branch／count規則に従って存在または省略し、0値をserializeしない。Runtime Packageはこのclosureを格納する外側Artifactであり、Package自身のhashをclosure inputへ含めない。selector hash、activation policy hash、Target Profile hash、System Graph、Implementation Set、selected Provider Binding集合、startup dependency／Target compatibilityの変更はCompile Manifestとbranch closureをinvalidateし、last-valid packageを現在Sourceの合格結果として再利用しない。

`technical_qualification_receipt_ref`は`state=qualified`のTargetだけ必須で、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md#72-technicalqualificationreceiptv1)のexact `TechnicalQualificationReceiptRefV1`を格納する。Refの`subject_hash`は同PackageのTarget readiness `input_closure_hash`とbyte equalityで、そのReceiptの`evidence_hashes[]`はPerformance OwnerのIntegrated Scale Receiptを含む。`predicted`／`blocked`ではzero hash／nullではなくField omissionをcanonical encodingする。PlayはDevelopment Playとqualified promotionを区別する。`predicted`のTargetはDevelopment Playを開始でき、未実測であることをEditor表示とReceiptへ明示する。[Performance／capacityが所有するqualification計測run](../04-runtime/performance-capacity.md#13-integrated-fixtureとqualification)はこのDevelopment Play実行モードで行い、Receipt確定後にだけ`qualified`へ昇格する。`blocked`のTargetはDevelopment Playを含むPlay開始を拒否する。未qualified revisionをqualified扱いのPlay、Cooked Runtime Package promotion、Shippingへ要求した場合、compilerはlast valid Receiptを流用せず`TargetNotQualified`を返す。

Source revisionと全dependency closureが同じであれば、Cooked Runtime Packageはbyte-for-byte同一でなければならない。Build日時、machine path、user、random IDはartifact本文へ含めずReceiptへ分離する。

大量配置や大量生成のScale intentは、Authoring Documentを無制限なEntity列挙にしてよいという意味ではない。procedural descriptor、Recipe、spatial partition等のbounded schemaを使い、Authoring aggregate自体のhard budgetは常に満たす。一方、Target Runtimeの予測budget未達だけを理由に有効な制作意図を破棄しない。`state=blocked`のrevisionもSourceとしてCommit、Diff、Undo、AI再提案できるが、対象TargetのPlay開始、Cooked Runtime Package promotion、Shippingには使えない。改善可能なbudget未達は`blocked_reason_ref=optimization_required`を保持する。

`target_readiness`はTargetごとに§3.4の`predicted | blocked | qualified`だけを持つ。`qualified`には[Performance／capacityが所有する`IntegratedScaleFixtureV1`](../04-runtime/performance-capacity.md#13-integrated-fixtureとqualification)のfresh Receipt hashが必要であり、Source、Scale intent、Representation Plan、Contract set、Toolchain lock、Target Profile、Device generation、Qualification policyのいずれかが変われば、新closureを評価して`predicted`または登録済み理由を持つ`blocked`へ戻す。last valid Derived ArtifactをDevelopment previewで使う場合、現在Sourceの合格結果に見せない。

## 10. Failure policy

| Failure | 結果 |
|---|---|
| Schema／semantic／Authoring hard budget不合格 | ChangeSet全体reject、live revision不変 |
| Runtime Target予測budget未達 | Source revisionを`state=blocked`、`blocked_reason_ref=optimization_required`でCommit可能。対象TargetのPlay／Cook／Shipping promotionを拒否し、制作意図と最適化候補を維持 |
| C1 entity／population製品Envelope未校正 | Source revisionを`state=blocked`、`blocked_reason_ref=performance_envelope_unqualified`でCommit可能。専用qualification harness以外のPlay／Cook／Shipping promotionを拒否 |
| Stale base revision | `RevisionMismatch`、最新Diff summaryを返す |
| Document／StableId不足 | `MissingReference`、placeholderへ黙って置換しない |
| 要求revisionのContext Index未完成 | `IndexNotReady`。別revisionへfallbackせず、bounded retryまたはTask分割 |
| Decision成立条件の変化 | `DecisionInvalidationRequired`。必要Decision change primitiveを返し、暗黙失効しない |
| Journal／disk full | Commit前にreject、dirty draftをRecoveryへ可能な範囲で保持 |
| Crash during Commit | 起動時にhash検証し完全transactionだけroll-forward |
| Derived cook失敗 | Source revision維持、last valid Derived ArtifactをDevelopment previewだけで明示継続 |
| External file破損 | Projectへimportせず隔離、最後の正規Documentを維持 |
| Undo conflict | inverseを適用せず、conflict resolverを表示 |
| journal保持範囲外へのUndo要求 | `UndoHistoryUnavailable`。inverse change primitive不在を明示し、部分適用しない |

## 11. TestとRelease Gate

最低限、次を自動化する。

- MCD→C++／TypeScript／JSON／binary round-tripとunknown field拒否
- 4096 change primitive、8 MiB境界、dependency cycle、duplicate ID、stale revision
- parent／reference／Recipe cycle、Dimension／Capability／Target不一致
- Commit各stepへのProcess kill fault injectionとroll-forward／隔離
- disk full、permission denial、partial write、rename failure
- Undo／Redo 10,000回後のstate hash一致。§6の最低保持journal範囲内の深度で実行し、保持範囲外Undoの`UndoHistoryUnavailable`拒否もあわせて検証する
- 外部編集三者比較、同field conflict、人間lock保持
- AI proposalと手動GUI操作が同じcanonical ChangeSetになるconformance
- 同じ`AuthoringSelectionContextV1`からmouse、keyboard、UI Automation、AIが同じStableId／revisionを対象にし、sort／filter／rename／re-shard後も表示indexまたはscreen coordinateへ退行しないconformance
- `ai_mutable=true` fieldの外側MCD Operation→typed change primitive coverage 100%、正規Authoring JSONへのAI write権限0
- 100万Entityを複数Shardへ保存し、StableId、親、参照、Decision、lockを跨いで検索／Slice／部分Diffできるfixture
- Re-shard前後で`entity_set_root_hash`とsemantic diffが不変、storage-only Diffだけが生成されるtest
- Index stale／rebuild中に別revisionのSliceを返さないconcurrency test
- Decision dependency変更で`InvalidateDecision`／`ReconfirmDecision`不足をrejectし、locked Decisionを別承認なしで変更できないnegative test
- `predicted -> blocked(optimization_required) -> predicted -> qualified`遷移、Receipt invalidation、`predicted` TargetのDevelopment Play許可と未実測明示、`blocked` TargetのPlay拒否、未qualified TargetのCooked Package promotion／Shipping拒否
- C1 entity／population envelope未校正時の`blocked(performance_envelope_unqualified)`、Target Profile実機fixture＋fresh Receiptによる解除、恣意的な数値defaultの拒否
- Target readinessへの`Predicted`／`Blocked`／`Qualified`／`OptimizationRequired`とCapability専用`not_activated`混入、stateと`blocked_reason_ref`／Receipt nullability不一致を一原因ずつ拒否するnegative fixture
- Game System authoritative State ownerが0件／複数件、stale System Bundle、Source Promotion後Project Commit failureのnegative／recovery test
- World／Scene／Space／Cell identity、Topology reachability、Portal trap、Map intent ambiguity、Cell activation atomicityのfixture
- Source Intentから同じTarget別Streaming Plan hashを再生成し、Derived Planの直接編集を拒否するtest
- `MoveEntityToScene`がsubtreeの永続化owner、Shard、明示したroot parentだけを原子的に変更し、owner-typed gameplay membership、subtree内部parent、StableId、Runtime Cellを暗黙変更しないvalid／invalid／Undo test
- `fixture.project.runtime-entry.world-empty`: Scene 0件／Topology nullかつexact World Space Profileを持つWorld entryがvalidでbranch hashだけを出力する
- `fixture.project.runtime-entry.world-space-profile`: World refからexact Profile ID／version／hashを解決し、missing／stale／copied dimensionを各一原因で拒否する
- `fixture.project.runtime-entry.ui-only`: World／TopologyなしのUI entryがvalidでWorld系hashをcanonical omissionする
- world／ui entryがstartup system 0件と128件の両境界でvalidになり、startup systemをbranch conflictにしないfixture
- `fixture.project.runtime-entry.headless`: startup system 1件以上、UI／World／surfaceなしでvalid
- 同一Targetにdefault 1件とnon-default 2件を登録し、non-default selector overlapを許可して明示選択できるpositive fixture
- `fixture.project.runtime_entry.document_identity`と`fixture.project.runtime_entry.selector_policy_resolution`が3種のProject-owned Document、identity三者一致、payload semantic／Document hash、selector set equality、activation policyを独立検証する。Runtime ownerの二fixtureとはSchedulingの`RuntimeEntryOwnerIntegrationManifestV1`だけでmappingする
- `fixture.integration.project-runtime-entry.owner-resolution`: 上記Project二系統とRuntime branch activation／reverse teardown二系統のexact fixture ref／hash、Compile Manifest、expected branch closure、owner Receipt mappingを照合する
- world／ui／headlessのstartup system 1件以上で同じtransitive closure hash入力を検査し、world／uiの0件だけomissionするfixture
- branch field混在、headless startup system 0件、default 0件／2件、unknown／inactive／duplicate Target selector、dangling ref、payload／Document hash、identity三者、schema、explicit Target membership mismatchを各Diagnosticへ一原因ずつ対応させるnegative fixture
- Root Scene conditional migrationのcurrent fixture ref集合がexact `[]`であること、および将来activation fixtureが実legacy inventoryからTargetごとに明示World entry一件を生成し、暗黙default、`Level` alias、UI／headless近似を行わないこと
- 大量Scale intentをbounded Recipe／partitionでCommitでき、Runtime budget未達時もSource、Diff、Undo、Gameplay fidelity floorを失わない
- 同一Project revisionを二回compileしたArtifact hash一致
- 100万Entityのread projectionを変更せず、影響Documentだけを再投影する性能fixture

本SubsystemのC1完了条件は、AIと手動GUIの両方で2D縦切りProjectを作成・保存・再起動・Undo・外部編集・Cookでき、無効ChangeSet、kill fault、stale revisionで正規状態を失わないことである。

## 12. 一次資料

- [RFC 9562 UUID](https://www.rfc-editor.org/rfc/rfc9562.html)
- [RFC 8785 JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785.html)
- [O3DE Asset Pipeline](https://docs.o3de.org/docs/user-guide/assets/pipeline/)
- [O3DE Product Assets and deterministic generation](https://docs.o3de.org/docs/user-guide/assets/pipeline/product-assets/)
- [Unity Undo API](https://docs.unity3d.com/ScriptReference/Undo.html)
- [Unreal Engine Transactions](https://dev.epicgames.com/documentation/en-us/unreal-engine/BlueprintAPI/Transactions/BeginTransaction)
- [Godot Running code in the editor](https://docs.godotengine.org/en/stable/tutorials/plugins/running_code_in_the_editor.html)

外部EngineのProject formatやPrefab実装は採用しない。UnityのEditor変更をUndoへ登録する原則、UnrealのEditor transaction、Godotの永続化owner／unsaved／Undoを明示する原則をEvidenceとし、Miraikanaiでは全経路を`ProjectChangeSetV1`、`AuthoringSelectionContextV1`、Scene永続化ownerへ統合する。SourceとDerivedの分離、安定ID、決定論的生成を含め、Document、ChangeSet、Commit、Projectionは本書で独自に定義する。
