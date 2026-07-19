# Miraikanai Engine Authoring Model／Project State規約

- 文書版: 1.0
- 作成日: 2026-07-19
- 対象: Project source、World Model、Scene、ChangeSet、保存、Undo／Redo、外部編集、Recovery
- 状態: プロジェクト公式の規範設計レビュー版
- 上位文書: [AIネイティブ独自ゲームエンジン 設計計画書](./2026-07-18-ai-native-game-engine-authoring-design.md)
- 基盤規約: [Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)
- Runtime規約: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- 契約規約: [Miraikanai Engine 実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)

## 1. 結論

Miraikanai Engineの正規Project状態は、Editor widget、Scene Tree表示、AI会話、Runtime World、生成済みC++ binaryのいずれでもない。Schema検証可能なAuthoring Document集合と、単調増加する`ProjectRevision`が正本である。

AI、Editor GUI、人間の手動編集、CLI、MCP、外部IDEは同じ`ProjectChangeSet`を提案する。状態を確定できるのはC++ `AuthoringCommandGateway`だけであり、全Operationを検証し、一つのrevisionとして原子的にCommitする。部分成功、暗黙補正、Editor内部objectの直接serializeを禁止する。

本書は次を独自に所有する。

- Project aggregateとDocument境界
- field-level ID、reference、revision、lock
- ChangeSet Operationとtransaction
- Source file、snapshot、journal、Undo／Redo、Recovery
- 外部編集とAI編集の競合
- Runtime packageへのcompile入力境界

## 2. 決定権と対象外

| 主題 | 正本 |
|---|---|
| Project Document、World Model、ChangeSet、Commit、Undo、Recovery | 本書 |
| MCD型、Operation、Error、Schema projection、Codegen | 実行可能契約規約 |
| ID、memory、pointer、thread、directory、serialization基礎 | 基盤規約 |
| Runtime World、tick、lease、queue、Asset promotion | Runtime規約 |
| AI権限、承認、Source sandbox、Promotion | AI実装・保守ガバナンス規約 |
| Editor panel、workspace、操作、accessibility | Editor規約 |

本書はGitをProject database、Undo system、runtime content storeとして必須化しない。Git連携は任意の外部version-control機能であり、Commitの成否はGit状態へ依存しない。共同リアルタイム編集、CRDT、branch merge UI、networked multi-user sessionはC3であり、C1／C2のChangeSet契約へ含めない。

## 3. Project aggregate

### 3.1 正規Document

| Document | 役割 | ID／revision |
|---|---|---|
| `ProjectManifest` | Project identity、root scene、Target、Package、Capability、Document index | Projectに一つ、`project_id`、`project_revision` |
| `GameSpecDocument` | Genreに依存しない要求、system、content、test、budget、style lock | `game_spec_id`、document revision |
| `WorldDocument` | Scene参照、global composition、persistent entity、streaming boundary | `world_id`、document revision |
| `SceneDocument` | Entity、component、parent、local transform、Composition Recipe instance | `scene_id`、document revision |
| `UiDocument` | UI tree、style、binding、navigation、localization key | `ui_document_id`、document revision |
| `GameplayDefinitionDocument` | Rule、state、task、typed Capability参照 | Definition StableId、document revision |
| `AssetMetadataDocument` | Source identity、import settings、license、provenance、tag | Asset StableId、document revision |
| `VisualStyleProfileDocument` | 表現四軸、Material、Lighting、Camera、Post、UI style | Profile StableId、document revision |
| `NativeGameModuleManifest` | Project C++ source root、Capability、access、build contract | Module StableId、manifest revision |
| `TargetProfileDocument` | Platform、quality、distribution、content delivery、budget | Profile StableId、document revision |
| `DecisionLedgerDocument` | 判断値、由来、理由、approval、lock、依存 | Entry StableId、document revision |
| `TestScenarioDocument` | Preconditions、input、oracle、budget、Target | Scenario StableId、document revision |

`ProjectManifest`はDocument本文を埋め込まず、`DocumentRef { stable_id, document_kind, relative_path, content_hash, schema_version }`だけを持つ。Authoring Document間の参照はStableIdで行い、相対path、配列index、表示名を意味参照に使用しない。

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

## 4. World Model

### 4.1 EntityとComposition

`SceneDocument`のEntity recordを次で固定する。

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

## 5. ProjectChangeSet

### 5.1 Envelope

```text
ProjectChangeSet
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
| Entity | `CreateEntity`、`DeleteEntity`、`ReparentEntity`、`SetSiblingOrder` |
| Component | `AddComponent`、`RemoveComponent`、`SetComponentField`、`ReplaceComponent` |
| Reference | `SetStableReference`、`ClearStableReference` |
| Recipe | `InstantiateRecipe`、`ApplyRecipeUpdate`、`SetRecipeOverride` |
| Gameplay／UI／Style | 各Subsystemが登録するtyped Operation |
| Asset | `RegisterAssetSource`、`SetImportField`、`ReplaceAssetSourceRevision` |
| Native C++ | `RegisterNativeModuleRevision`。Source promotion済みhashだけ |
| Target／Decision | `SetTargetProfileField`、`RecordDecision`、`LockDecision` |

自由形式の`SetJsonPointer`、任意path write、任意C++ symbol call、任意console commandをOperation Catalogへ登録しない。複数fieldを不変条件とともに変える操作は一つのDomain Operationとし、細かな`SetField`列へ分解して中間不整合を作らない。

### 5.3 Commit algorithm

`AuthoringCommandGateway`はAuthoring threadで次を順番どおり実行する。

1. Envelope、size、authorization、Project IDを検証する。
2. `base_project_revision == live_project_revision`を検証する。
3. Operation ID一意性とdependency DAGを検証し、canonical topological orderを作る。
4. MCD schema、enum、range、finite、string、StableId、pathを検証する。
5. 全preconditionとDocument revisionを検証する。
6. 参照整合、cycle、Capability、Target intersection、Domain invariantを検証する。
7. 変更後aggregateをcopy-on-write stagingへ構築する。
8. memory、Asset cook、render、physics、nav、package等の予測costとRisk policyを検証する。
9. Domain dry-runと必要なbackground validation artifactのhashを照合する。
10. 変更Document、inverse Operation、manifest、journal recordを同一temporary transaction directoryへ書く。
11. 全fileをflushし、transaction manifestを最後に原子的renameする。
12. 新`ProjectRevision = old + 1`とDocument indexを一つのcommit pointでpublishする。
13. Projectionへ`ProjectRevisionCommitted` eventを値として配送する。

1～10の失敗はlive stateを変更しない。11以後にProcessが停止した場合、次回起動時にtransaction manifest、file hash、journal recordの三者を検査し、完全なtransactionだけをroll-forwardする。不完全なtemporary directoryは隔離し、勝手に部分復旧しない。

## 6. Source layoutと永続化

Project rootの正規layoutを次で固定する。

```text
<project>/
├─ mira.project.json
├─ authoring/
│  ├─ game_spec/
│  ├─ worlds/
│  ├─ scenes/
│  ├─ ui/
│  ├─ gameplay/
│  ├─ visual_styles/
│  ├─ targets/
│  ├─ decisions/
│  └─ tests/
├─ assets/
│  ├─ source/
│  └─ metadata/
├─ native/
│  └─ game/
├─ .mira/
│  ├─ journal/
│  ├─ snapshots/
│  ├─ recovery/
│  └─ user/
└─ build/                 # 生成物。source control対象外
```

- `mira.project.json`とAuthoring JSONはUTF-8 without BOM、LF、重複key禁止、comments禁止、trailing comma禁止とする。
- JSONは人間Diff用sourceであり、Runtimeは直接読まない。
- `.mira/journal`はChangeSet、base／result revision、before／after hash、inverse Operation、Receipt参照を持つappend-only recordである。
- 100 Commitまたはjournal 64 MiBの早い方でsnapshotを作る。最新2 snapshotと、それ以後のjournalを最低保持する。
- Project source保存成功とAsset／Runtime cook成功を同一transactionにしない。Commit済みsourceからDerived Artifactを非同期生成し、失敗時もsource revisionを失わない。
- Auto-saveは未CommitのEditor draftを`.mira/recovery/<user>/<session>`へ20秒ごと、またはfocus loss時に保存する。正規revisionへ自動Commitしない。

## 7. Undo／Redo、外部編集、競合

Undoは過去fileを上書きする操作ではない。Journalのinverse Operationを現在revisionに対する新ChangeSetとして再検証し、新revisionを作る。依存変更により安全に反転できない場合は対象と衝突を表示し、強制適用しない。Redoも同じ規則である。

外部Editor／IDEの変更はFile watcherが検出し、次を行う。

1. UTF-8、syntax、duplicate key、schemaをStagingで検証する。
2. 最後に認識したbase hash、外部file、現在Project Documentの三者比較を作る。
3. 差分をtyped Operationへ変換できる場合だけ`ExternalTool` ChangeSetを生成する。
4. 人間がEditor内で確認するか、事前承認Policyに合致した場合だけCommitする。

同じfieldへの異なる変更、delete対edit、Recipe更新対override、StableId再利用は自動mergeしない。異なるDocument／異なるfieldで不変条件を共有しない変更だけ自動merge候補にでき、結局は新しいbase revisionで全Validatorを再実行する。

## 8. AIと手動編集

- AIは現在revision、関係Document、lock、Capability、Target、budgetを含む`AuthoringContextPack`を読む。
- AIは存在しないStableIdを推測せず、`Create*` Operationで新IDを要求する。IDはGatewayが生成して結果mapを返す。
- AIは巨大Sceneを全置換せず、目的に必要なOperationだけを提案する。
- 人間がlockしたfield、Document、Entity subtreeをAIは変更できない。
- Level 0ではAIが質問と仮定をGame用語で提示し、実装語を初心者へ選ばせない。
- 手動Inspector、Graph、Code連携もAIと同じDiff、Validation、Undoを使う。
- AI説明、AI proposal、Engine validation、Commit済み結果を別stateとして表示する。

## 9. Runtime compile境界

Runtime compilerはCommit済み`ProjectRevision`だけを入力にし、Editor draft、Projection cache、AI会話を読まない。Compile manifestは次を固定する。

```text
project_id
project_revision
document_set_hash
capability_manifest_hash
target_profile_hash
native_module_revision_hash
asset_dependency_root_hash
contract_lock_hash
toolchain_lock_hash
```

Source revisionと全dependency closureが同じであれば、Cooked Runtime Packageはbyte-for-byte同一でなければならない。Build日時、machine path、user、random IDはartifact本文へ含めずReceiptへ分離する。

## 10. Failure policy

| Failure | 結果 |
|---|---|
| Schema／semantic／budget不合格 | ChangeSet全体reject、live revision不変 |
| Stale base revision | `RevisionMismatch`、最新Diff summaryを返す |
| Document／StableId不足 | `MissingReference`、placeholderへ黙って置換しない |
| Journal／disk full | Commit前にreject、dirty draftをRecoveryへ可能な範囲で保持 |
| Crash during Commit | 起動時にhash検証し完全transactionだけroll-forward |
| Derived cook失敗 | Source revision維持、last valid Derived ArtifactをDevelopment previewだけで明示継続 |
| External file破損 | Projectへimportせず隔離、最後の正規Documentを維持 |
| Undo conflict | inverseを適用せず、conflict resolverを表示 |

## 11. TestとRelease Gate

最低限、次を自動化する。

- MCD→C++／TypeScript／JSON／binary round-tripとunknown field拒否
- 4096 Operation、8 MiB境界、dependency cycle、duplicate ID、stale revision
- parent／reference／Recipe cycle、Dimension／Capability／Target不一致
- Commit各stepへのProcess kill fault injectionとroll-forward／隔離
- disk full、permission denial、partial write、rename failure
- Undo／Redo 10,000回後のstate hash一致
- 外部編集三者比較、同field conflict、人間lock保持
- AI proposalと手動GUI操作が同じcanonical ChangeSetになるconformance
- 同一Project revisionを二回compileしたArtifact hash一致
- 100万Entityのread projectionを変更せず、影響Documentだけを再投影する性能fixture

本SubsystemのC1完了条件は、AIと手動GUIの両方で2D縦切りProjectを作成・保存・再起動・Undo・外部編集・Cookでき、無効ChangeSet、kill fault、stale revisionで正規状態を失わないことである。

## 12. 一次資料

- [RFC 9562 UUID](https://www.rfc-editor.org/rfc/rfc9562.html)
- [RFC 8785 JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785.html)
- [O3DE Asset Pipeline](https://docs.o3de.org/docs/user-guide/assets/pipeline/)
- [O3DE Product Assets and deterministic generation](https://docs.o3de.org/docs/user-guide/assets/pipeline/product-assets/)

外部EngineのProject formatやPrefab実装は採用しない。SourceとDerivedの分離、安定ID、決定論的生成という検証済み原則を参照し、MiraikanaiのDocument、ChangeSet、Commit、Projectionは本書で独自に定義する。
