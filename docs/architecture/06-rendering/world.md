# Miraikanai Engine World／Scene／Space／Cell Contract

- 文書ID: mirakan.arch.rendering-world
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: World／Scene／SpaceのSource identity、global composition、再利用Scene instance／nested composition／typed override／explicit rebase、persistent entity、optional spatial topology、Cellのplan-local identity、partition／streaming-plan authoring、spatial transition、Loading presentation、procedural／Tilemap／Blockoutの共通意味、World authoring read projectionの所有境界
- 非正本範囲: 具体World schema、procedural／Tilemap／Blockout catalog、Operation、Fixture、Gameplay progression、Runtime cell phase、ECS schema、Physics／Navigation behavior、Render execution、Save／Replay envelope
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Project State](../03-authoring/project-state.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Collision](../05-simulation/collision.md)
- 関連文書: [Procedural World Catalog／Fixture Candidate](../appendices/procedural-world-catalog-fixture.md)、[Product Lifecycle](../00-product/product-lifecycle.md)、[Editor Workspace／UX](../03-authoring/editor-workspace-ux.md)、[Scenario／Stage](../08-packs/scenario-stage.md)、[Runtime Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Persistence／Save](../04-runtime/persistence-save.md)、[Debugging／Observability／Replay](../04-runtime/debugging-observability-replay.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[Physics](../05-simulation/physics.md)、[Navigation](../05-simulation/navigation.md)、[Render Graph](render-graph.md)、[LOD](lod.md)、[Virtualized／Continuous Geometry](virtualized-continuous-geometry.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-27

## 1. 結論と所有境界

Worldは空間、Scene、global composition、persistent entity、任意のspatial topologyを所有する。Gameplay goal、outcome、spawn、進行単位はconsumer-owned stateであり、World activationへ必須にしない。

World activation、Scene activation、Cell streamingはGameplay goalやResultを要求しない。Scene 0件のprocedural-only World、spatial topologyなしのUI補助World、有限進行を持たないcontinuous simulationをvalidとする。

## 2. 正規用語とidentity

| term | meaning |
|---|---|
| World | global compositionとpersistent spatial／nonspatial sourceのauthority |
| Scene | World内で再利用可能なsource composition |
| Space | coordinate／topology profile |
| Anchor | 一つのWorld／Scene SourceとSpaceへ束縛されたstable attachment point。spawn、Objective、Runtime handleそのものではない |
| Topology relation | source／destinationのAnchorまたはSpaceを結ぶWorld-owned relation。方向、対称性、到達条件をtypedに表す |
| Cell | 一つのpartition／streaming plan内だけで有効なplan-local単位 |
| Map | user intent。World、Scene、Tilemap、Stage等へtypedに解決する |
| Level Workspace | World／Scene／Spaceとowner-typed Gameplay文書を横断するEditor presentation。Source、identity、membership、revisionまたはRuntime activation単位ではない |

World／Scene identityはstable source IDとrevisionであり、display name、path、配列index、Runtime handleを使わない。AnchorとTopology relationはowner World／Scene Sourceのexact ref／revisionへ束縛する。Cell IDをWorld-global identityまたはpersistent entity IDとして保存しない。`Level`または`Region`という表示labelからSource identityを生成しない。

## 3. 「Map」要求の解決規則

`Map`はEngine schema名に固定しない。要求を次へ解決する。

- global spatial composition → World
- reusable authored composition → Scene
- 2D tile source → Tilemap source
- finite gameplay progression → Scenario／Stage
- runtime partition unit → Cell

曖昧な場合は候補を提示して選択を求め、root Scene、Level Workspace、Stage、Worldを相互aliasにしない。

<a id="world-level-workspace-boundary"></a>

### 3.1 Level Workspaceの非authority境界

`Level Workspace`は保存DocumentまたはCore gameplay conceptではない。同じProject revisionにあるWorld composition、Scene Source、Space、Topology relation、必要に応じたowner-typed Gameplay文書を一つのEditor surfaceへ投影する。`level_id`、`level_revision`、`level_membership`、`LevelSourceV1`または`playable_level`をCore Sourceへ追加しない。

表示語は次へ一意に解決する。

| 表示語 | 正規解決 |
|---|---|
| Level | `Level Workspace`の現在Context。永続refとして保存しない |
| Region | coordinate／topology境界なら`Space`またはWorld-owned Topology selection、それ以外は明示されたowner-typed Document／query |
| Portal | World-owned `Topology relation`またはspatial `Transition intent`の表示。別のCore identityを作らない |
| Entry／Exit、Objective、finite progression | [Scenario／Stage](../08-packs/scenario-stage.md)の`StageDefinitionV1`等、選択されたFeature／Game Ownerのtyped ref |
| Scene集合 | World SourceのScene composition。Level membershipという第二の集合を作らない |

Level Workspaceが複数Ownerを同時表示しても、各Fieldのcanonical owner、Document revision、authorization、Operationを維持する。一つのform submitから別Ownerの変更が必要な場合、各Ownerのactiveなtyped primitiveを一つの`ProjectChangeSetV1`へ明示的に列挙してatomic commitする。必要なOperationが一つでも未Activationなら、そのFieldを理由付きread-only Gapにし、Level固有operation、自由patchまたは部分commitへfallbackしない。

## 4. Source Document model

World Sourceはidentity、Space profile、Scene composition、persistent entity source、optional topology、partition plan refs、environment／render refs、activation policyを持つ。

Scene SourceはWorldから独立した再利用identityを持ち、Worldはexact Source revisionを`SceneCompositionInstanceV1`として0件以上compositionする。Scene SourceとinstanceはWorld-global stateまたはGameplay progressionを所有しない。Persistent entityはSource identityとRuntime Entity handleを分離する。

具体Field候補は[Procedural World Catalog](../appendices/procedural-world-catalog-fixture.md#4-source-document-model)を参照する。補助文書のSchemaをmaterialized current Schemaと扱わない。

### 4.1 再利用Scene composition

外部EngineまたはUserの`Prefab`要求は、再利用Source compositionを意味する場合に`Scene Source`＋`SceneCompositionInstanceV1`へ解決する。`Prefab`をcanonical ID、Schema名、保存形式、legacy aliasまたは第二のSource authorityにしない。

```text
SceneSourceRevisionRefV1
  scene_source_id: StableId
  source_revision: positive u64
  source_content_hash: SHA-256

WorldAttachmentRefV1
  {
    kind: composition_root,
    owner_world_document_ref: exact ProjectDocumentRefV1
  }
  | {
      kind: world_anchor,
      owner_world_document_ref: exact ProjectDocumentRefV1,
      anchor_id: StableId,
      anchor_content_hash: SHA-256
    }
  | {
      kind: scene_source_object,
      source_scene_ref: exact SceneSourceRevisionRefV1,
      source_object_id: StableId,
      source_object_content_hash: SHA-256
    }

SceneCompositionInstanceV1
  instance_id: StableId
  source_scene_ref: exact SceneSourceRevisionRefV1
  parent_instance_id: null | StableId
  attachment_ref: exact WorldAttachmentRefV1
  override_set_ref: null | exact SceneOverrideSetRefV1
  accepted_rebase_change_ref: null | exact SceneRebaseChangeRefV1
```

`instance_id`はowner World／Scene composition内でstableかつuniqueである。同じScene Sourceを複数instanceとして使用できるが、instance identity、override、runtime lineageを共有しない。`parent_instance_id`は同じcomposition内だけを指し、instance graphはacyclic、最大depth 32、展開後instance数65,536以下とする。上限はprovisionalであり、Activation前にReference Fixtureの実測で再承認する。

Attachmentはclosed tagged unionでWorld／Scene Ownerが定義するstable Anchor、source objectまたはcomposition rootへexact解決する。`composition_root`と`world_anchor`は同じowner World document revision、`scene_source_object`は同じcompositionから到達可能なexact Scene Source revisionへ解決する。branch外Field、display name、Hierarchy path、array index、Editor selection、Runtime Entity handleをattachmentにしない。

### 4.2 typed override

```text
SceneOverrideSetV1
  override_set_id: StableId
  override_set_version: positive u32
  source_scene_ref: exact SceneSourceRevisionRefV1
  entries[1..4096]:
    sorted unique SceneOverrideEntryV1
  override_set_content_hash: SHA-256

SceneOverrideSetRefV1
  override_set_id: StableId
  override_set_version: positive u32
  override_set_content_hash: SHA-256

SceneOverrideEntryV1
  source_object_id: StableId
  owner_document_ref: exact ProjectDocumentRefV1
  field_contract_ref: exact PublicFieldContractRefV1
  expected_source_value_hash: SHA-256
  replacement_value: exact field-contract value
```

OverrideはSource objectとowner documentとpublic field contractをstable identityで解決し、Source revision上のexpected value hashを必須にする。Field contractが許可する型、range、reference kind、authorization、cross-field invariantをそのまま使用する。World OwnerがGameplay、Physics、Render、UI等のField Schemaを複写しない。

同じ`{source_object_id, owner_document_ref, field_contract_ref}`へのduplicate、型不一致、read-only Field、unknown object、別Source revision、expected value hash不一致、capacity超過を拒否する。名前、path、配列index、似たField、同名object、compatible-looking typeからtargetをrepairしない。Override適用後もconsumer OwnerのDocument validationを満たす必要があり、一部Fieldだけをcommitしない。

### 4.3 explicit rebase

```text
SceneRebaseChangeV1
  rebase_change_id: StableId
  rebase_change_version: positive u32
  instance_id: StableId
  before_source_scene_ref: exact SceneSourceRevisionRefV1
  after_source_scene_ref: exact SceneSourceRevisionRefV1
  before_override_set_ref: null | exact SceneOverrideSetRefV1
  result_override_set_ref: null | exact SceneOverrideSetRefV1
  resolutions[0..4096]:
    sorted unique SceneRebaseResolutionV1
  unresolved_conflicts[0..4096]:
    sorted unique SceneRebaseConflictV1
  rebase_change_content_hash: SHA-256

SceneRebaseChangeRefV1
  rebase_change_id: StableId
  rebase_change_version: positive u32
  rebase_change_content_hash: SHA-256

SceneRebaseResolutionV1
  source_object_id: StableId
  field_contract_ref: exact PublicFieldContractRefV1
  resolution:
    keep_instance_override | accept_source_value | replace_with_typed_value
  replacement_value: absent unless resolution=replace_with_typed_value
  before_value_hash: SHA-256
  after_source_value_hash: null | SHA-256
  result_value_hash: null | SHA-256

SceneRebaseConflictV1
  source_object_id: StableId
  field_contract_ref: exact PublicFieldContractRefV1
  kind:
    object_deleted | field_deleted | field_type_changed
    | source_and_override_changed | target_owner_changed
    | reference_unresolvable | capacity_exceeded
  evidence_hash: SHA-256
```

Scene Source更新は既存instanceへ自動追従しない。Rebaseはbefore／after exact revision、override delta、resolution、conflictをimmutable candidateとして作る。`unresolved_conflicts[]`が0件で、result Override Setとafter Sourceの全Owner validationが成功した場合だけProject ChangeSetへ列挙できる。

`SceneCompositionInstanceV1.accepted_rebase_change_ref`をnon-nullにするProject ChangeSetは、解決したRebaseの`instance_id`とInstanceの`instance_id`、`after_source_scene_ref`とInstanceの`source_scene_ref`、`result_override_set_ref`とInstanceの`override_set_ref`をbyte equalityにする。Instanceの`override_set_ref`がnon-nullなら解決したOverride Setの`source_scene_ref`もInstanceの`source_scene_ref`とbyte equalityにする。Rebaseの`before_source_scene_ref`は同じInstanceの直前Project revisionが持つSource ref、`before_override_set_ref`は同revisionのOverride refとbyte equalityにし、別instance、別Source、別result、飛び越したbefore revisionを拒否する。

Rebaseを持たないInstanceでも、non-null Override Setの`source_scene_ref`はInstanceの`source_scene_ref`とbyte equalityにする。`accepted_rebase_change_ref=null`のままSourceだけを変更する、Rebase resultと異なるOverrideを同じChangeSetで束縛する、受理後にRebase recordを差し替えることを禁止する。

`keep_instance_override`はafter Sourceでも同じField contractと型が存続する場合、`accept_source_value`はresult Overrideからentryを除く場合、`replace_with_typed_value`は新しいtyped valueを明示する場合だけ有効である。Deleted objectを同名object、renamed pathまたは近いIDへ自動mappingしない。

### 4.4 Authoring、Cook、Runtime lineage

AuthoringではScene Source、instance、Override Set、Rebase Changeを同じProject revisionのexact refsへ閉じる。Editor GUI、CLI、headlessは同じOwner-typed Operationと`ProjectChangeSetV1`を使用し、Inspectorだけのhidden overrideまたはSource file直接編集を作らない。

Cookはinstance graphをTarget別Runtime Artifactへ展開し、次のlineageを完成manifestへ保持する。

- source Scene ID／revision／hash
- instance IDとparent instance ID
- source object IDからcooked stable object IDへのmapping
- accepted Override Set ref
- accepted Rebase Change ref
- source owner document／field contract refs

Runtime Package、Save、Replay、Debuggingはこのlineageの必要subsetをexact artifact refで消費する。Runtime ArtifactはAuthoring Scene Source file、Override Set file、Editor object、rebase service、Hierarchy pathへ依存しない。Cook後にSourceを再解決せず、別revisionへ自動追従しない。

Saveはpersistent consumerが宣言するstable identityとstateだけを保存し、Override Set全体またはAuthoring valueをruntime authorityにしない。Replayはaccepted instance／artifact lineageとruntime eventを記録し、Source rebaseを実行しない。

### 4.5 hash、failure、qualification

`SceneOverrideSetV1`と`SceneRebaseChangeV1`のcontent hashはそれぞれASCII `MIRAKAN_SCENE_OVERRIDE_SET_V1`、`MIRAKAN_SCENE_REBASE_CHANGE_V1`と自己hash Fieldだけを除くclosed canonical bytesから計算する。`SceneRebaseChangeRefV1`のID、version、hashは解決先recordとbyte equalityにする。Entries、resolutions、conflictsは`source_object_id`、`field_contract_ref`のcanonical bytes、kind順へstrict sortしduplicateを拒否する。

最低限、cycle、wrong source ref／hash、stable ID欠落、type mismatch、deleted target、expected value mismatch、rebase conflict、capacity超過、runtime Source dependencyを別のtyped Diagnosticにする。Prepublication failureではSource、instance、Override、Rebaseのcurrent Project stateを一byteも変更しない。

Qualificationはsimple instance、nested instance、同一Sourceの複数instance、typed override、source update、explicit rebase、conflict、object delete、field type change、undo／redo、save／load、replay、cook、package、debug lineage、crash recovery、Runtime PackageがAuthoring Sourceなしで起動することを含む。

## 5. Spatial topology

Topologyは2D、3D、nonspatialをclosed profileで表す。座標系、単位、axis、origin policyはMath／World Space Ownerのtyped refへ解決する。WorldがPhysics、Navigation、Renderingの内部payloadを複写しない。

Nonspatial Worldではanchor、Cell、spatial spawn、streaming fieldをcanonical omissionする。Default 3D空間を補完しない。

Topology relationはWorld-owned Source relationであり、Editor上の`Portal`表示を正本にしない。双方向または対になるrelationを要求するprofileでは両側を同じChangeSetで検証し、片側だけを公開しない。Gameplay上の解放条件、Objective、Stage progressionはrelation payloadへ複写せず、owner-typed policy refを使用する。

## 6. Spatial Partitionとstreaming-plan authoring

Partition planはWorld Source revision、Space profile、algorithm profile、Cell集合、dependency、priority、capacity refを束縛する。Cell identityはplan ID＋plan revision＋local cell IDで解決し、別planへ流用しない。

```text
WorldStreamingPlanRefV1
  plan_id: StableId
  plan_revision: positive u64
  plan_artifact_hash: SHA-256

WorldStreamingPlanV1
  schema_version: 1
  plan_id: StableId
  plan_revision: positive u64
  source_world_ref: exact {world_id, source_revision, source_content_hash}
  world_space_profile_ref: WorldSpaceProfileRefV1
  partition_algorithm_profile_ref: exact owner-typed ref
  cells[0..1048576]: WorldStreamingCellDescriptorV1
  representation_slots[0..1048576]:
    WorldStreamingRepresentationSlotV1
  dependency_edges[0..4194304]: WorldStreamingDependencyEdgeV1
  capacity_ref: exact owner-typed ref
  compiler_ref: exact {compiler_id, compiler_version, compiler_hash}
  plan_artifact_hash: SHA-256

WorldStreamingCellDescriptorV1
  cell_id: non-zero u64
  bounds_ref: exact World-space bounds ref
  priority_class: critical | gameplay | presentation | background
  residency_group_ref: exact owner-typed ref
  capacity_charge_ref: exact owner-typed ref

WorldStreamingRepresentationSlotV1
  representation_slot_id: non-zero u64
  cell_ids[1..256]: sorted unique plan-local cell_id
  source_stable_id_set_hash: SHA-256
  allowed_representation_roles[1..8]:
    individual | instanced | spatial_proxy | impostor
  residency_dependency_role_refs[0..64]:
    sorted unique registered owner-qualified role refs

WorldStreamingDependencyEdgeV1
  from_cell_id: non-zero plan-local u64
  to_cell_id: non-zero plan-local u64
  dependency_kind: activation_required | prefetch_hint
```

`plan_artifact_hash`はASCII `MIRAKAN_WORLD_STREAMING_PLAN_V1`と自己hashを除くcanonical bytesから計算する。Cellは`cell_id`、slotは`representation_slot_id`、edgeは`from_cell_id, to_cell_id, dependency_kind`順へstrict sortしduplicate、self edge、`activation_required` cycle、Plan外cell IDを拒否する。`prefetch_hint`はactivation prerequisiteではない。Streaming plan authoringはSourceを直接分割せずReceipt-free Derived plan候補を作る。Runtime phase、I/O scheduling、shared capacityはRuntime Ownerが決定する。

[LOD](lod.md)のHLOD Artifactは完成Plan refとplan-local cell／representation slotを参照するdownstream artifactであり、World PlanへHLOD artifact ref／hashを埋め戻さない。生成順を`World Source -> WorldStreamingPlanV1 -> HlodArtifactV1`に固定し、hash cycleを作らない。Planのresidency dependency roleは必要artifactのOwner-qualified役割だけを表し、HLOD artifact ref、runtime resident／pending stateまたはactivation authorityを保存しない。

[Virtualized／Continuous Geometry](virtualized-continuous-geometry.md)も同じ一方向境界を使う。Worldの`residency_dependency_role_refs[]`は登録済みの`required_root | likely_detail | opportunistic`相当owner-qualified role refを持てるが、virtual page ID、micro-cluster ID、pool slot、resident／pending bit、View cutまたはGPU feedbackを持たない。virtual geometry Artifactは完成World Plan refまたはrepresentation slotをdownstream dependencyとして参照できるが、そのArtifact ref／hashをWorld Plan hashへ埋め戻さない。

HLODのWorld aggregation clusterとvirtual geometryのmicro-clusterは別identity、別owner、別lifetimeである。前者は複数subjectのproxy representation、後者は一つのArtifact family内部のView-local detail nodeであり、World Cell membership、activation、Save identity、Gameplay relevancyをmicro-clusterから生成しない。Capabilityが`planning_only`の現在はWorld role Registryへvirtual geometry roleを登録しない。

## 7. Spatial transition intent

Transition intentはsource context、typed destination、requested anchor、subject refs、fallback policyを持つ。World handle、Scene pointer、Cell addressをPublic Portへ出さない。

Destination activation failure時はsource authorityを維持し、half-activated World、orphan Cell、部分subject transferを残さない。

### 7.1 Loading／prefetch presentation

Loading presentationはtransition stateのUI projectionであり、activation authorityではない。Progressは測定可能なwork closureから算出し、偽の100%、無限spinner、UI closeによるactivation commitを禁止する。

## 8. 参照と依存closure

World／Scene SourceはAsset、Schema、Space、Environment、Render、Collision等のtyped refをversion／hash付きで束縛する。Path、display name、latest assetをpackage時に再解決しない。

## 9. Procedural World source

Procedural Worldはseed、generator profile、input closure、determinism class、budget、stable-ID allocation policyをSourceとして宣言する。Generated resultはDerived candidateであり、World SourceまたはProject revisionへ自動昇格しない。

Stable-ID allocation、generated Delta、Manifest、Receipt、Public Commit Closure、Operation／Policy／Fixtureの具体候補は[Procedural World Catalog](../appendices/procedural-world-catalog-fixture.md#9-procedural-world-source)へ分離する。

Ownerが維持する不変条件は次である。

1. 同じgenerator、version、seed、input closure、Target profileから同じcanonical resultを生成する。
2. Stable IDは一つのimmutable candidate内で決定論的に割り当てる。
3. Prepublication failureではpublic objectを0件にする。
4. 成功時はmapping、Delta、Manifest、Receipt、Marker、after Projectを一つのclosureへ束縛する。
5. Runtime生成結果をAuthoring Sourceへ黙示的に逆書込みしない。

## 10. Navigation、Simulation、Renderingとの境界

Worldはgeometry／topology sourceとtyped attachment pointを提供する。Collisionはcollision representation、Physicsはbody／constraint、Navigationはnav data、Renderingはvisibility／draw executionを所有する。

### 10.1 Tilemap source、cook、publication

Tilemapは2D source grid、layer、tile semantic、asset refを所有するが、Physics collider、Navigation grid、Render meshのruntime payloadを所有しない。Cookは各Subsystem projectionを同じTilemap Source revisionへ束縛する。

具体Tilemap Schema、Cook manifest、Fixture候補は[Procedural World Catalog](../appendices/procedural-world-catalog-fixture.md#101-tilemap-sourcecookpublication)を参照する。

### 10.2 Engine-native 3D Blockout

BlockoutはAuthoring geometry sourceであり、final art、Physics body、Navigation mesh、Render meshを兼用しない。Cooked projectionはSource identity、unit、material semantic、Target profileを束縛する。

## 11. Authoring bundleとAI／Editor UX

World authoring bundleは同一Project revisionのSource closureだけを含む。AI／Editorはtyped ChangeSetを提案し、live Runtime World、Editor selection、partial streaming stateをSource authorityへ直接serializeしない。

### 11.1 World authoring read projection

World Ownerは`WorldAuthoringContextV1`と`SceneSliceV1`の意味、Source境界、freshnessを所有する。Project Stateは`AuthoringSelectionContextV1`を所有し、Editorはattention／focus／Panel bindingだけを所有する。

`WorldAuthoringContextV1`は一つのexact Project revisionとWorld Source ref／revision／hash、選択されたScene／Space／Topology relation／Transition intent ref、Target、`Source | Staging | Derived read-only | Runtime`区分、field mask、任意のSpace-bound viewport query、omitted range／continuation、selection context hashまたは明示`null`、invalidation condition、projection content hashを持つbounded read-only projectionである。`SceneSliceV1`は同じProject／World lineageにある一つのScene Source ref／revision／hash、query／field mask、返却record、omitted range／continuation、invalidation condition、slice content hashを持つ。どちらもProject mutation、AI authorization、完全なWorld closureまたはRuntime handleを意味しない。

Level Workspaceはこの二Projectionとowner-typed Gameplay projectionをContext hashで関連付けるだけで、`Level` refへ統合しない。World／Scene Source revision、Project revision、field mask対象、Space-bound viewport query、selection context hashのいずれかが変われば該当Projectionをstaleにし、新しいContextを要求する。screen coordinate、render texture、Hierarchy path、表示row、表示名からtargetまたは権限を再構成しない。

上記はtarget contractであり、materialized Schema、collection bound、query Tool、Operation、Fixture、Receiptはcurrent Repositoryに存在しない。したがって現在は`conceptually-readable`であり、`operationally-readable`または利用可能と扱わない。

## 12. Save、Replay、Migration境界

Saveはpersistent identityとOwnerが宣言したStateだけを保存する。World Source、Cell plan、Runtime handleを混同しない。Replayはtransition intent、accepted activation、relevant source／artifact identityを記録する。

Migrationはsource／target schema、consumer inventory、stable identity mapping、明示的migration outcome、Evidenceを必要とし、似た名前のWorld／Sceneへ黙示的に置換しない。

## 13. Diagnostic、failure、qualification

最低限、unknown ref、wrong revision／hash、invalid topology、presentation label used as identity、plan-local Cell misuse、capacity overflow、generator nondeterminism、stable-ID conflict、partial publication、Target unsupportedをtyped Diagnosticで区別する。

Qualificationはworldless、Scene 0、2D、3D、large partition、transition failure、procedural determinism、Tilemap／Blockout projection、Level Workspace labelまたは`Prefab`入力語から新しいSource identityを生成しないこと、Scene instance／override／rebase、Save／Replay、crash recoveryを含む。具体Fixture候補は[補助Catalog](../appendices/procedural-world-catalog-fixture.md#13-diagnosticfailurequalification)へ分離する。
