# Miraikanai Engine Authoring Model／Project State規約

- 文書ID: mirakan.arch.project-state
- 状態: review
- 正本範囲: Project aggregate、Authoring Document、ProjectRevision、ProjectChangeSetV1のdomain schema／意味／transaction、Target readiness envelope、Commit、Source／Derived境界、Undo／Redo、外部編集、Recovery
- 非正本範囲: MCD共通Envelope／projection／codegen、命名・Project配置、Asset lifecycle、Editor表示、Gameplay System、Native ABI／Build、Runtime scheduling。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core architecture](../02-foundation/core-architecture.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Naming／Project layout](../02-foundation/naming-project-layout.md)、[Asset lifecycle](asset-lifecycle.md)、[Editor UI Framework](editor-ui-framework.md)、[Editor Workspace UX](editor-workspace-ux.md)、[Gameplay programming model](gameplay-programming-model.md)、[Native game module](native-game-module.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Performance／capacity](../04-runtime/performance-capacity.md)、[World／Level／Map](../06-rendering/world.md)、[UI／Text／Localization／Accessibility](../07-platform/ui-text-localization-accessibility.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論

Miraikanai Engineの正規Project状態は、Editor widget、Scene Tree表示、AI会話、Runtime World、生成済みC++ binaryのいずれでもない。Schema検証可能なAuthoring Document集合と、単調増加する`ProjectRevision`が正本である。

AI、Editor GUI、人間の手動編集、CLI、MCP、外部IDEは同じ`ProjectChangeSetV1`を提案する。状態を確定できるのはC++ `AuthoringCommandGateway`だけであり、全Operationを検証し、一つのrevisionとして原子的にCommitする。部分成功、暗黙補正、Editor内部objectの直接serializeを禁止する。

本書は次を独自に所有する。

- Project aggregateとDocument境界
- field-level ID、reference、revision、lock
- ChangeSet Operationとtransaction
- Source file、snapshot、journal、Undo／Redo、Recovery
- 外部編集とAI編集の競合
- Runtime packageへのcompile入力境界
- Target別`TargetReadinessV1`と`TargetBlockedReasonRegistryV1`。性能測定とEvidence freshnessの意味は各Ownerを参照する

## 2. 決定権と対象外

| 主題 | 正本 |
|---|---|
| Project Document、World Model、`ProjectChangeSetV1`のdomain schema／Operation意味／transaction、Commit、Undo、Recovery | 本書 |
| MCD型、Operation共通Envelope、Error、Schema projection、Codegen | [Executable contracts](../02-foundation/executable-contracts.md) |
| ID、memory、pointer、thread、directory、serialization基礎 | [Core architecture](../02-foundation/core-architecture.md)と各Foundation Owner |
| Runtime World、tick、lease、queue、Asset promotion | [Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md) |
| AI権限、承認、Source sandbox、Promotion | [AI Security／Approval](../01-governance/ai-security-approval.md) |
| Editor panel、workspace、製品操作、人間工学 | [Editor Workspace UX](editor-workspace-ux.md) |
| Editor Widget、Semantic Snapshot、UI eventからtyped Commandへの変換 | [Editor UI Framework](editor-ui-framework.md) |
| Game System Spec、Implementation Set、System Bundle、二段階Activation | [Gameplay programming model](gameplay-programming-model.md) |
| World、Scene、Level、Topology、Partition Intent、Procedural World、Map Presentation | [World／Level／Map](../06-rendering/world.md) |

本書はGitをProject database、Undo system、runtime content storeとして必須化しない。Git連携は任意の外部version-control機能であり、Commitの成否はGit状態へ依存しない。共同リアルタイム編集、CRDT、branch merge UI、networked multi-user sessionはC3であり、C1／C2のChangeSet契約へ含めない。

## 3. Project aggregate

### 3.1 正規Document

| Document | 役割 | ID／revision |
|---|---|---|
| `ProjectManifest` | Project identity、root scene、Target、Package、Capability、Document index | Projectに一つ、`project_id`、`project_revision` |
| `GameSpecDocument` | Genreに依存しない要求、system、content、test、budget、style lock | `game_spec_id`、document revision |
| `WorldDocument` | Scene／Level／Topology参照、global composition、persistent entity、Source Intent root | `world_id`、document revision |
| `SceneDocument` | collaborative edit shard identity、Shard index、global setting、Composition Recipe root。Gameplay LevelまたはStreaming Cellではない | `scene_id`、document revision |
| `SceneEntityShardDocument` | 一つのSceneに属するbounded Entity record集合 | `shard_id`、document revision |
| `WorldTopologyDocument` | Region、Level、Portal、entryの論理Graph | `topology_id`、document revision |
| `LevelDefinitionDocument` | entry／exit、Objective、Spawn、Encounter、Game System、Profileを持つplay可能単位 | `level_id`、document revision |
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

`ProjectManifest`はDocument本文を埋め込まず、`DocumentRef { stable_id, document_kind, relative_path, content_hash, schema_version }`だけを持つ。Authoring Document間の参照はStableIdで行い、相対path、配列index、表示名を意味参照に使用しない。

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

GatewayはChangeSetの影響closureから、`applies_to`、`evidence_refs`、`decision_dependencies`、`validity_predicates`を決定論的に照合する。成立条件が変わる場合、提案ChangeSetに`InvalidateDecision`または新根拠を伴う`ReconfirmDecision`がなければ`MIRAKAN-DECISION-INVALIDATION_REQUIRED`で全体を拒否し、必要OperationとDecision IDを返す。GatewayがEntryを黙って`needs_review`へ変更してはならない。`locked=true`のDecisionへ影響する変更は、別Authorityが承認した`UnlockDecision`を同じtransactionへ含めない限り拒否する。

### 3.4 Target readiness

Project revisionのTarget別実行可否は次の一型だけで表す。

```text
TargetReadinessV1
  target_profile_ref
  project_revision
  input_closure_hash
  state: predicted | blocked | qualified
  blocked_reason_ref: string | null
  technical_qualification_receipt_ref: string | null
```

`input_closure_hash`はSource revision、Scale intent、Representation Plan、Contract set、Toolchain lock、Target Profile、Device generation、Qualification policyのcanonical hash closureである。`predicted`は安全なPlanを生成できるが当該closureの実測Receiptがなく、`blocked_reason_ref=null`、`technical_qualification_receipt_ref=null`とする。`blocked`は現在入力ではPlay／Cook／Shipping promotionを許可できず、登録済みnon-null `blocked_reason_ref`、null Receiptを必須とする。`qualified`は同じ`input_closure_hash`へ束縛されたfresh `TechnicalQualificationReceiptV1`を必須とし、`blocked_reason_ref=null`とする。状態とnullabilityが一致しないRecordを拒否する。

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
- filter／sort中のdrag操作は表示indexでなくStableIdをOperationへ渡す。
- 変更は必ずtyped Operationへ変換し、Projection cacheを先に変更しない。
- Commit成功後に新revisionから再投影する。
- stale read modelからのOperationは`RevisionMismatch`で拒否する。

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
  level_ref optional
  viewport_bounds optional
  field_mask
  target_profile_ref optional
  lock_refs[0..128]
  source_hashes[1..1024]
  omitted_ranges[0..128]
  continuation
```

`AuthoringSelectionContextV1`はCommit済みDocumentと明示的な`EditorUserState`から生成するread-only／DisposableなContextであり、Project正本またはUndo対象ではない。`primary_stable_id`、World／Scene／Level参照は表示名、Hierarchy path、row index、screen coordinateから推測せず、存在確認済みStableIdとrevisionを使う。AIへ渡すContext、Editor command、UI Automation semantic actionは同じContext hashを参照し、操作時には対象StableIdとexpected Document revisionを再指定する。Contextがstale、対象がomitted、lock情報が欠落、またはSource／Derived区分が不明な場合は変更Operationへ昇格しない。

Architecture Governanceが所有する`ArchitectureExplainProjectionV1`はProject Stateの正本ではなく、Commit済みSourceとexact registry closureから生成されるread-only／Disposableなconsumer projectionである。`authoring.explain_architecture`は`scope`、非空`field_mask`、optional `target_profile_ref`、exact `project_revision`を要求し、別revisionへのfallbackを行わない。応答の`omitted_ranges`または署名付き`continuation`が示す未取得範囲をEvidence済みとして扱わず、stale revision、continuation条件不一致、必要Evidence欠落では説明を確定しない。

このqueryまたは自然言語要約からProjectを直接変更してはならない。後続ChangeSetは解決済みcanonical concept、対象StableId、typed Operation、expected Document revisionを正規Gatewayへ再指定する。Projection entry、Owner名、外部用語、表示path、summary textをCommit可能なidentityまたはOperationとして受理しない。

### 4.3 大規模SceneのShard、Index、Slice

`SceneDocument`は論理aggregateであり、Entity本文を直接無制限に埋め込まない。C1から一つ以上の`SceneEntityShardDocument`をStableId参照し、各Shardは4,096 Entity recordまたはcanonical encoded 8 MiBの早い方を上限とする。単一Entityが上限を超える場合はComponent schema違反としてrejectし、巨大blobをShardへ埋め込まない。

- Shardは`scene_id`、`shard_id`、`partition_mode = stable_id_range | spatial_cell`、partition key、Entity StableId範囲、record count、content hashを持つ。
- Entity StableIdはShard移動、Scene rename、spatial cell変更で変えない。親、Recipe、Component参照はShard内indexでなくStableIdを使う。
- `SceneDocument.entity_set_root_hash`はShard配置ではなく、Entity StableId順のcanonical Entity leafからMerkle計算する。Re-shardだけでGameplay上のsemantic diffを作らない。
- Re-shardも正規Source変更なので一つの`ProjectRevision`としてCommitするが、Diffは`storage_only`とEntity意味変更を分離する。
- Shardを跨ぐ親cycle、reference、lock、Decision、Recipe invariantはScene aggregate全体で検証する。

Entityの永続化ownerは、そのrecordを含むShardの`scene_id`で厳密に一つへ決まる。Transform parent、Outliner folder、Level membership、Streaming Cell、Data Layer相当のtagから永続化ownerを推測しない。Scene間移動は`MoveEntityToScene` Domain Operationだけが、移動元／移動先Scene revision、移動rootと全descendant、移動先のoptional parent、参照、lock、Recipe override、boundsを検証してsubtree recordを移す。subtree内部のparent関係とStableIdを維持し、移動rootの新parentは移動先Scene内またはnullに限定する。Level membershipまたはRuntime Cell割当は同Operationの暗黙副作用にせず、必要なSource変更を同じChangeSetへ別のtyped Operationとして明示する。

`AuthoringContextIndexV1`はCommit済みrevisionから生成するDisposableな派生Indexであり、正本ではない。`project_revision`、`contract_set_hash`、Document root hash、index schema version、利用するtokenizer ID／manifest hash集合を固定し、次を索引化する。

- StableIdからDocument／Shard／fieldへの位置。
- inbound／outbound reference、Requirement、Capability、Decision、lockのclosure。
- Sceneのspatial bounds、tag、component type、modified revision。
- field maskごとのcanonical byte数とProvider tokenizer別の実測token数。

Commit後は旧Indexをstaleにし、変更Shardと参照closureだけをcopy-on-write更新してから新revisionとしてpublishする。要求revisionのIndexがReadyでない場合、別revisionの結果を返さず`MIRAKAN-AUTHORING-INDEX_NOT_READY`とretry hintを返す。

AI、Editor、CLIへ返す`SceneSliceV1`は、query ID、Project revision、anchor StableId、選択Shard、field mask、dependency depth、各source hash、選択理由、omitted range、continuation cursorを持つread-only projectionである。任意byte位置で切ったJSON、表示順index、要約だけをChangeSetの根拠にしない。SliceからのOperationもStableIdとexpected Document revisionを必須とし、Gatewayが対象Shardへroutingする。

## 5. ProjectChangeSetV1

### 5.1 Envelope

`ProjectChangeSetV1`のdomain schema、Operation意味、transaction／Commit規則は本節だけが所有する。[Executable contracts](../02-foundation/executable-contracts.md#8-operation定義)のMCD共通Envelopeと生成規則を消費するが、suffixなし最新版aliasまたは別Envelopeは作らない。

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
  operations: ProjectOperation[1..4096]
  preconditions: ProjectPrecondition[0..1024]
  evidence_refs: StableId[0..128]
```

ChangeSet全体のcanonical encoded sizeは8 MiB以下とする。Asset binary、C++ source本文、巨大配列を埋め込まず、許可済みStaging fileのcontent hashとrelative pathを参照する。

### 5.2 Operation

全Operationは`operation_id`、closed `operation_type`、target StableId、typed argument、operation内依存、expected document revision、declared costを持つ。

| Operation群 | C1 Operation |
|---|---|
| Document | `CreateDocument`、`DeleteDocument`、`RenameDocument` |
| Entity | `CreateEntity`、`DeleteEntity`、`ReparentEntity`、`SetSiblingOrder`、`MoveEntityToScene` |
| Component | `AddComponent`、`RemoveComponent`、`SetComponentField`、`ReplaceComponent` |
| Reference | `SetStableReference`、`ClearStableReference` |
| Recipe | `InstantiateRecipe`、`ApplyRecipeUpdate`、`SetRecipeOverride` |
| Gameplay／UI／Style | 各Subsystemが登録するtyped Operation |
| Game System | `RegisterProjectGameSystemSpec`、`SetSystemImplementationVariant`、`ReplaceSystemConfiguration`。`qualified` Contract／Staging hashだけ |
| World／Level | Topology、Level、Partition Intent、Procedural、Map Presentationの各Domain typed Operation |
| Asset | `RegisterAssetSource`、`SetImportField`、`ReplaceAssetSourceRevision` |
| Native C++ | `RegisterNativeModuleRevision`。Source promotion済みhashだけ |
| Target／Decision | `SetTargetProfileField`、`RecordDecision`、`LockDecision`、`UnlockDecision`、`InvalidateDecision`、`ReconfirmDecision` |

自由形式の`SetJsonPointer`、任意path write、任意C++ symbol call、任意console commandをOperation Catalogへ登録しない。複数fieldを不変条件とともに変える操作は一つのDomain Operationとし、細かな`SetField`列へ分解して中間不整合を作らない。

AIへ公開する全Authoring Capabilityは、MCDで`ai_mutable=true`の全fieldが一つ以上のtyped OperationまたはDomain Operationから到達可能であることをContract compilerで証明する。Operation coverageが100%でないCapabilityはAI Tool catalogへ昇格しない。AI TaskのPath Grantへ正規Authoring JSONのwrite権限を含めず、AIがSource fileを直接変更してcoverageを迂回する経路を作らない。

### 5.3 Commit algorithm

`AuthoringCommandGateway`はAuthoring threadで次を順番どおり実行する。

1. Envelope、size、authorization、Project IDを検証する。
2. `base_project_revision == live_project_revision`を検証する。
3. Operation ID一意性とdependency DAGを検証し、canonical topological orderを作る。
4. MCD schema、enum、range、finite、string、StableId、pathを検証する。
5. 全preconditionとDocument revisionを検証する。
6. 参照整合、cycle、Capability、Target intersection、Decision invalidation、Domain invariantを検証する。
7. 変更後aggregateをcopy-on-write stagingへ構築する。
8. Authoring aggregate自体のmemory／schema hard budgetとRisk policyを検証する。Runtime Targetのrender、physics、nav、VFX、package予測costは、安全なRepresentation Planがありestimate内でも未実測なら`state=predicted`、現在のPlanでは未達なら`state=blocked`と登録済み`blocked_reason_ref`を結果revisionへ記録する。`qualified`は予測から生成せず、同じ`input_closure_hash`へ束縛されたfresh統合負荷Receiptを照合できた場合だけ維持する。C1 entity／population envelopeが未校正なら`blocked_reason_ref=performance_envelope_unqualified`とする。
9. Domain dry-runと必要なbackground validation artifactのhashを照合する。schema、safety、boundedness、不変条件の失敗はrejectし、Target performance／capacityだけの未達は`state=blocked`、改善可能なら`blocked_reason_ref=optimization_required`として記録する。
10. 変更Document、inverse Operation、manifest、journal recordを同一temporary transaction directoryへ書く。
11. 全fileをflushし、transaction manifestを最後に原子的renameする。
12. 新`ProjectRevision = old + 1`とDocument indexを一つのcommit pointでpublishする。
13. `AuthoringContextIndexV1`の旧revisionをstaleにし、変更Shardと参照closureの更新Jobを発行する。
14. Projectionへ`ProjectRevisionCommitted` eventを値として配送する。

1～10の失敗はlive stateを変更しない。11以後にProcessが停止した場合、次回起動時にtransaction manifest、file hash、journal recordの三者を検査し、完全なtransactionだけをroll-forwardする。不完全なtemporary directoryは隔離し、勝手に部分復旧しない。

### 5.4 System／World Bundle

`SystemBundleChangeSetV1`のschema、状態遷移、二段階Activation、Source Promotion後のrecoveryは[Gameplay programming model](gameplay-programming-model.md)だけが所有する。本書はBundleが参照する`ProjectChangeSetV1`と最終Project Commitだけを所有する。Gatewayは検証済みexact hashを受け取り、`RegisterNativeModuleRevision`と`SetSystemImplementationVariant`を同じ`ProjectChangeSetV1`でCommitする。Bundle自体をCommitして正規Documentを迂回せず、Project Commit失敗時にSource repositoryをrollbackしない。

World BundleはStaging SourceからTarget別Streaming／Navigation／LOD／Package Artifactを試作し、Topology、playability、budget、failure fixtureを検証してからSource Document群を一つの`ProjectChangeSetV1`へ変換する。Derived Artifactの生成失敗でSource revisionを部分Commitせず、Commit後の非同期再Cook失敗時はSourceを維持して該当Targetを`blocked`へ遷移させ、原因に対応する`blocked_reason_ref`を記録する。

## 6. Source layoutと永続化

Project root全体のPathと命名は[Game Project配置・命名規約](../02-foundation/naming-project-layout.md#5-engine-rootとgame-project-root)を正本とする。Top-level rootは同規約のclosed set（`source/`、`config/`、`packages/`、`derived/`、`intermediate/`、`staging/`、`evidence/`と`mirakan.project.json`）以外を追加しない。旧layout root（`authoring/`、`assets/`、`native/game/`、`.mirakan/`、`build/`）はcanonicalではなく、Gatewayは`LegacyLayoutRoot`としてProject openを拒否する。本書はAuthoring永続化に関係するprojectionを次で固定する。

```text
<project>/
├─ mirakan.project.json
├─ source/
│  ├─ assets/                  # Source Assetとimport設定。AssetMetadataDocumentをAsset IDで併置
│  ├─ worlds/                  # World／Scene／Topology／Level／Partition Intent／Procedural World／Map Presentation
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
- World、Level、Topology、Partition Intent、Procedural World、Map Presentationも`source/worlds/`配下でStable IDから決定論的にpathを導出し、表示名、Region名、Target名をpath identityへ使わない。System Implementation Set、Visual Style、Target Profile、Decision、Test Scenarioも同じ規則で各directoryへ置く。
- `.mirakan.json`は人間Diff用sourceであり、Runtimeは直接読まない。
- journal、snapshot、transaction directoryの配置は本書が所有し、Git追跡・配布対象外の`intermediate/`配下へ置く。canonical stateはCommit済み`source/`のDocumentだけであり、journal／snapshotをcanonical sourceへ昇格しない。
- `intermediate/journal/`はChangeSet、base／result revision、before／after hash、inverse Operation、Receipt参照を持つappend-only recordである。
- 100 Commitまたはjournal 64 MiBの早い方でsnapshotを`intermediate/snapshots/`へ作る。最新2 snapshotと、それ以後のjournalを最低保持する。
- §5.3のtemporary transaction directoryは`intermediate/transactions/<change_set_id>/`であり、roll-forward検査の完了後に削除または隔離する。
- Project source保存成功とAsset／Runtime cook成功を同一transactionにしない。Commit済みsourceからDerived Artifactを非同期に`derived/`へ生成し、失敗時もsource revisionを失わない。
- Auto-saveは未CommitのEditor draftを`intermediate/recovery/<user>/<session>`へ20秒ごと、またはfocus loss時に保存する。正規revisionへ自動Commitしない。

## 7. Undo／Redo、外部編集、競合

Undoは過去fileを上書きする操作ではない。Journalのinverse Operationを現在revisionに対する新ChangeSetとして再検証し、新revisionを作る。依存変更により安全に反転できない場合は対象と衝突を表示し、強制適用しない。Redoも同じ規則である。

Undo可能深度は§6のjournal最低保持範囲（最新2 snapshotとそれ以後のjournal）と一致する。保持範囲を超えて破棄済みのjournalへ到達するUndo要求は`UndoHistoryUnavailable`として拒否し、部分的なinverse適用や推測による復元をしない。

外部Editor／IDEの変更はFile watcherが検出し、次を行う。

1. UTF-8、syntax、duplicate key、schemaをStagingで検証する。
2. 最後に認識したbase hash、外部file、現在Project Documentの三者比較を作る。
3. 差分をtyped Operationへ変換できる場合だけ`ExternalTool` ChangeSetを生成する。
4. 人間がEditor内で確認するか、事前承認Policyに合致した場合だけCommitする。

同じfieldへの異なる変更、delete対edit、Recipe更新対override、StableId再利用は自動mergeしない。異なるDocument／異なるfieldで不変条件を共有しない変更だけ自動merge候補にでき、結局は新しいbase revisionで全Validatorを再実行する。

## 8. AIと手動編集

- AIは現在revision、関係Document、lock、Capability、Target、budgetを含む`AuthoringContextPackV1`を読む。
- Context選択は`AuthoringContextIndexV1`と署名対象の`AuthoringContextPlanV1`から行い、選択理由、field mask、omitted range、continuation、source hashを失わない。
- Editorで選択中のWorld／Scene／Level／Entityを会話Contextへ含める場合は`AuthoringSelectionContextV1`を使い、画面pixel、Hierarchy path、表示名だけを対象識別子にしない。
- AIは存在しないStableIdを推測せず、`Create*` Operationで新IDを要求する。IDはGatewayが生成して結果mapを返す。
- AIは巨大Sceneを全置換せず、目的に必要なOperationだけを提案する。
- AIは正規Authoring JSONへ直接writeせず、`authoring.search`、`authoring.read`、`authoring.dependencies`、`authoring.diff`とtyped Operationだけを使う。
- 人間がlockしたfield、Document、Entity subtreeをAIは変更できない。
- Level 0ではAIが質問と仮定をGame用語で提示し、実装語を初心者へ選ばせない。
- 手動Inspector、Graph、Code連携もAIと同じDiff、Validation、Undoを使う。
- AI説明、AI proposal、Engine validation、Commit済み結果を別stateとして表示する。

`AuthoringContextPackV1`と`AuthoringContextPlanV1`のcanonical schemaは本書が所有する。`AuthoringContextPlanV1`はContext選択の署名対象であり、`plan_id`、`project_id`、exact `project_revision`、`contract_set_hash`、参照した`AuthoringContextIndexV1` revision、選択理由、Documentごとのfield mask、source hash、omitted range、optional continuationをfield setとする。`AuthoringContextPackV1`はPlanから決定論的に生成するread-only／DisposableなAI入力projectionであり、`plan_hash`、現在revision、関係Documentの`SceneSliceV1`、optional `AuthoringSelectionContextV1`、lock、Capability、Target、budgetの各参照をfield setとする。両者はProject正本またはUndo対象ではなく、Commit可能なidentityまたはOperationとして受理しない。

Editor／Workspace UXが生成する`ExternalEngineConceptResolutionV1`は入力解決Evidenceであり、Project DocumentまたはChangeSetへ保存しない。Gatewayは外部用語、resolver object、候補配列、外部Scene path／Hierarchy indexをidentityまたはOperationとして受理しない。後続ChangeSetは`selected_concept`のcanonical concept ID、対象StableId、typed Operation、expected Document revisionを再指定しなければならず、`question_required`または`unsupported`のResolutionからProposalを生成してはならない。

## 9. Runtime compile境界

Runtime compilerはCommit済み`ProjectRevision`だけを入力にし、Editor draft、Projection cache、AI会話を読まない。Compile manifestは次を固定する。

```text
project_id
project_revision
document_set_hash
capability_manifest_hash
target_profile_hash
game_system_dependency_graph_hash
system_implementation_set_hash
world_topology_hash
level_definition_set_hash
world_streaming_plan_hash
native_module_revision_hash
asset_dependency_root_hash
contract_lock_hash
toolchain_lock_hash
scale_intent_hash
representation_plan_hash
technical_qualification_receipt_hash
```

`technical_qualification_receipt_hash`は`state=qualified`のTargetだけ必須で、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md#72-technicalqualificationreceiptv1)の`TechnicalQualificationReceiptV1`完成hashを指す。その`evidence_hashes[]`はPerformance OwnerのIntegrated Scale Receiptを含む。`predicted`／`blocked`では0ではなくfield omissionをcanonical encodingする。PlayはDevelopment Playとqualified promotionを区別する。`predicted`のTargetはDevelopment Playを開始でき、未実測であることをEditor表示とReceiptへ明示する。[Performance／capacityが所有するqualification計測run](../04-runtime/performance-capacity.md#13-integrated-fixtureとqualification)はこのDevelopment Play実行モードで行い、Receipt確定後にだけ`qualified`へ昇格する。`blocked`のTargetはDevelopment Playを含むPlay開始を拒否する。未qualified revisionをqualified扱いのPlay、Cooked Runtime Package promotion、Shippingへ要求した場合、compilerはlast valid Receiptを流用せず`TargetNotQualified`を返す。

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
| Decision成立条件の変化 | `DecisionInvalidationRequired`。必要Decision Operationを返し、暗黙失効しない |
| Journal／disk full | Commit前にreject、dirty draftをRecoveryへ可能な範囲で保持 |
| Crash during Commit | 起動時にhash検証し完全transactionだけroll-forward |
| Derived cook失敗 | Source revision維持、last valid Derived ArtifactをDevelopment previewだけで明示継続 |
| External file破損 | Projectへimportせず隔離、最後の正規Documentを維持 |
| Undo conflict | inverseを適用せず、conflict resolverを表示 |
| journal保持範囲外へのUndo要求 | `UndoHistoryUnavailable`。inverse Operation不在を明示し、部分適用しない |

## 11. TestとRelease Gate

最低限、次を自動化する。

- MCD→C++／TypeScript／JSON／binary round-tripとunknown field拒否
- 4096 Operation、8 MiB境界、dependency cycle、duplicate ID、stale revision
- parent／reference／Recipe cycle、Dimension／Capability／Target不一致
- Commit各stepへのProcess kill fault injectionとroll-forward／隔離
- disk full、permission denial、partial write、rename failure
- Undo／Redo 10,000回後のstate hash一致。§6の最低保持journal範囲内の深度で実行し、保持範囲外Undoの`UndoHistoryUnavailable`拒否もあわせて検証する
- 外部編集三者比較、同field conflict、人間lock保持
- AI proposalと手動GUI操作が同じcanonical ChangeSetになるconformance
- 同じ`AuthoringSelectionContextV1`からmouse、keyboard、UI Automation、AIが同じStableId／revisionを対象にし、sort／filter／rename／re-shard後も表示indexまたはscreen coordinateへ退行しないconformance
- `ai_mutable=true` fieldのtyped Operation coverage 100%、正規Authoring JSONへのAI write権限0
- 100万Entityを複数Shardへ保存し、StableId、親、参照、Decision、lockを跨いで検索／Slice／部分Diffできるfixture
- Re-shard前後で`entity_set_root_hash`とsemantic diffが不変、storage-only Diffだけが生成されるtest
- Index stale／rebuild中に別revisionのSliceを返さないconcurrency test
- Decision dependency変更で`InvalidateDecision`／`ReconfirmDecision`不足をrejectし、locked Decisionを別承認なしで変更できないnegative test
- `predicted -> blocked(optimization_required) -> predicted -> qualified`遷移、Receipt invalidation、`predicted` TargetのDevelopment Play許可と未実測明示、`blocked` TargetのPlay拒否、未qualified TargetのCooked Package promotion／Shipping拒否
- C1 entity／population envelope未校正時の`blocked(performance_envelope_unqualified)`、Target Profile実機fixture＋fresh Receiptによる解除、恣意的な数値defaultの拒否
- Target readinessへの`Predicted`／`Blocked`／`Qualified`／`OptimizationRequired`とCapability専用`not_activated`混入、stateと`blocked_reason_ref`／Receipt nullability不一致を一原因ずつ拒否するnegative fixture
- Game System authoritative State ownerが0件／複数件、stale System Bundle、Source Promotion後Project Commit failureのnegative／recovery test
- World／Scene／Level／Cell identity、Topology reachability、Portal trap、Map intent ambiguity、Cell activation atomicityのfixture
- Source Intentから同じTarget別Streaming Plan hashを再生成し、Derived Planの直接編集を拒否するtest
- `MoveEntityToScene`がsubtreeの永続化owner、Shard、明示したroot parentだけを原子的に変更し、Level membership、subtree内部parent、StableId、Runtime Cellを暗黙変更しないvalid／invalid／Undo test
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
