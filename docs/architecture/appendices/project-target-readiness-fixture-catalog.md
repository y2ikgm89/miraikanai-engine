# Project Target Readiness／Fixture Candidate Catalog

- 文書ID: mirakan.appendix.project-target-readiness-fixture-catalog
- 文書種別: Owner supplement
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 親Owner: [Project State](../03-authoring/project-state.md)
- 正本範囲: Project Document、Runtime Entry、Target readiness、Change primitive、transaction、fixtureのreview候補詳細
- 非正本範囲: Project aggregate、revision、commit、undo／recoveryの意味、Domain payload、current readiness。親Ownerと各Domain Ownerが決定する
- 規範依存: [親Owner](../03-authoring/project-state.md)
- 関連文書: [Executable Contracts](../02-foundation/executable-contracts.md)、[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)、[Runtime Package](../04-runtime/runtime-package.md)、[World](../06-rendering/world.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-21

> 本書は分離前Owner文書の具体Project Document、Runtime Entry、Target readiness、Change primitive、fixture候補を保持する。親OwnerのProject aggregate、revision、transaction、commit、recovery意味を上書きせず、Repository ArtifactとReceiptがない候補をcurrentまたはrelease-readyとして扱わない。

> 以下の見出し番号は、分離前Ownerからの参照互換性と履歴追跡のために維持する。欠番は省略された親Ownerの安定規範であり、本書に補完しない。

## 分離前Owner節から抽出した候補record

### 3.1 正規Document

| Document | 役割 | ID／revision |
|---|---|---|
| `ProjectManifest` | Project identity、Runtime entry point、Target、Package、Capability、Document index | Projectに一つ、`project_id`、`project_revision` |
| `RuntimeEntryPointDocumentV1` | `RuntimeEntryPointV1` Source、branch、selector／activation policy参照 | `entry_point_id`、document revision |
| `RuntimeTargetSelectorDocumentV1` | Runtime Entryが選択可能なexact Target Profile集合 | `selector_id`、document revision |
| `RuntimeEntryActivationPolicyDocumentV1` | readiness timeout、failure／cancel／deactivation semantics | `policy_id`、document revision |
| `RuntimeEntryPresentationBindingDocumentV1` | world Runtime Entryと初期root UiDocumentのtarget-only exact binding | `binding_id`、document revision |
| `GameSpecDocument` | Genreに依存しない要求、system、content、test、budget、style lock | `game_spec_id`、document revision |
| `WorldDocument` | exact `WorldSpaceProfileRefV1`、Scene／optional spatial topology参照、global composition、persistent entity、Source Intent root | `world_id`、document revision |
| `EnvironmentSurfaceDocumentV1` | Environment Profile、World binding、Weather／Water／Snow surface Source root | `document_id`、`source_revision` |
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

target `ProjectManifest.runtime_entry_presentation_binding_refs`は0～64件のexact `DocumentRef<RuntimeEntryPresentationBindingDocumentV1>`であり、§3.1.1.1のatomic activation前はField自体をcurrent Manifest schemaへ追加せず、current集合をexact `[]`とする。

`EnvironmentSurfaceDocumentV1`は[Environment／Water／Weather／Snow](../06-rendering/environment-surfaces.md)が所有するCore canonical Documentであり、owner-typed Pack Documentとして扱わない。WorldはEnvironment Profile本文または既定値を複製せず、`global_composition_refs[]`にexact `EnvironmentWorldBindingRefV1`を0件または1件だけ保持する。bindingの`world_id`、Environment Profile ref、Document hash、World側refを検証し、bindingの追加・変更・解除はEnvironment DocumentとWorld Documentを同じ`ProjectChangeSetV1`で原子的に更新する。

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

- `world`: `world_ref`を厳密に1件、`ui_document_ref=null`、`startup_game_system_refs[0..128]`。参照WorldはScene 0件、Topology nullでもよいが、exact `WorldSpaceProfileRefV1`を必須にし、そのProfile hashをWorld Document hash closure内で照合する。Runtime EntryはProfileを複写しない。初期HUD等のUIはV1 field意味を変更せず§3.1.1.1のseparate target bindingで表す。
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

#### 3.1.1.1 Target Runtime Entry presentation binding

既存`RuntimeEntryPointV1`のtagged semanticsを変更せず、world branchへ初期HUD等のUI rootを追加するtarget contractを次へ固定する。

```text
RuntimeEntryPresentationBindingV1
  binding_id: StableId
  binding_version: positive uint32
  runtime_entry_ref: DocumentRef<RuntimeEntryPointDocumentV1>
  runtime_entry_semantic_hash: RuntimeEntryPointSemanticHashV1
  root_ui_document_ref: exact UiDocumentRef
  root_ui_document_content_hash: SHA-256
  binding_content_hash: SHA-256

RuntimeEntryPresentationBindingRefV1
  binding_id: StableId
  binding_version: positive uint32
  runtime_entry_semantic_hash: RuntimeEntryPointSemanticHashV1
  binding_content_hash: SHA-256
```

Bindingは同一Project revisionの`entry_kind=world` Runtime Entry一件とUiDocument一件だけを結び、Runtime Entryの`ui_document_ref`は引き続きnullでなければならない。一world entryあたりBindingは0または1件で、同じUiDocumentを複数entryの別Bindingから共有参照することは許可する。`root_ui_document_content_hash`は`root_ui_document_ref.content_hash`とbyte equalityでなければならない。`ui | headless` entryへのBinding、same entryへの二件、entry／Document content hash差、Target selector外entry、display name／path／latestによる解決を`MIRAKAN-PROJECT-RUNTIME_ENTRY_PRESENTATION_BINDING_INVALID`でrejectする。

`binding_content_hash`はASCII `MIRAKAN_RUNTIME_ENTRY_PRESENTATION_BINDING_V1`と自身を除く全FieldのMCD canonical bytesをlength framingして計算する。Bindingは初期root Screenだけを選び、Screen Stack mutation、Focus、HUD state、Pause、Runtime Entry／Stage／World transitionを所有しない。UI ownerはexact UiDocumentとNavigation Policyから`UiScreenDefinitionV1`をCookし、Runtime ownerはBindingをbranch activation closureへ含める。

本Binding、Project Manifest target Field、Document kind、Compile Manifest target Field、validator、reader／writerは一つのCompatibility／Definition Migrationでatomic activationする。current `runtime_entry_presentation_binding_refs`、current authoring Operation、MCD／Service／Provider／MCP inventoryはexact `[]`であり、Markdown記載だけからBindingをcurrent availableにしない。既存V1の`world.ui_document_ref=null` readerを緩和する暫定fallbackを禁止する。

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
| World／Environment／owner-typed content | Topology、Partition Intent、Procedural、Map Presentation、Environment ownerが登録するclosed `EnvironmentChangePrimitiveV1`、登録済みowner namespaceの各Domain typed change primitive。`SetEnvironmentWorldBinding`はEnvironment DocumentとWorld Documentを同じChangeSetで変更 |
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

`SystemBundleChangeSetV1`のschema、状態遷移、二段階Activation、Source Promotion後のrecoveryは[Gameplay programming model](../03-authoring/gameplay-programming-model.md)だけが所有する。本書はBundleが参照する`ProjectChangeSetV1`と最終Project Commitだけを所有する。Gatewayは検証済みexact hashを受け取り、`RegisterNativeModuleRevision`と`SetSystemImplementationVariant`を同じ`ProjectChangeSetV1`でCommitする。Bundle自体をCommitして正規Documentを迂回せず、Project Commit失敗時にSource repositoryをrollbackしない。

World BundleはStaging SourceからTarget別Streaming／Navigation／LOD／非公開trial Package Artifactを試作し、Topology、playability、budget、failure fixtureを検証してからSource Document群を一つの`ProjectChangeSetV1`へ変換する。trial Artifactはfinal Package／Package Receiptではなく`PreparedCandidateV1.prepared_artifact_refs[]`またはRelease入力へ入れない。Derived Artifactの生成失敗でSource revisionを部分Commitせず、Commit後の非同期再Cook失敗時はSourceを維持して該当Targetを`blocked`へ遷移させ、原因に対応する`blocked_reason_ref`を記録する。

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
- target `fixture.project.runtime-entry.world-with-ui-root`: exact Presentation BindingからWorldとUI Documentを同じentry closureへ含め、Binding／World／UI hashをcanonical presentにし、V1 `world.ui_document_ref`をnullのままHUD rootを持つbranchとしてcompileする。atomic activation前はfixture ref集合へ含めない
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
