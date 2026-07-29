# Miraikanai Engine Authoring Model／Project State規約

- 文書ID: mirakan.arch.project-state
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Project aggregate、Authoring Document、ProjectRevision、ProjectChangeSetV1の意味とtransaction、Target readiness意味、Commit、Source／Derived境界、Undo／Redo、外部編集、Recovery、authoring target selection projectionの所有境界
- 非正本範囲: 具体Document／Operation／Change primitive／readiness／fixture候補、MCD共通Envelope、命名・Project配置、Asset lifecycle、Editor表示、Gameplay System、Native ABI、Runtime package
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[Core Architecture](../02-foundation/core-architecture.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)
- 関連文書: [Target Readiness／Fixture Candidate Catalog](../appendices/project-target-readiness-fixture-catalog.md)、[Product Lifecycle](../00-product/product-lifecycle.md)、[Product Security](../01-governance/product-security.md)、[Asset Lifecycle](asset-lifecycle.md)、[Editor Workspace UX](editor-workspace-ux.md)、[Gameplay Programming Model](gameplay-programming-model.md)、[Runtime Package](../04-runtime/runtime-package.md)、[World](../06-rendering/world.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-27

## 1. 結論

Projectの正規状態はEditor widget、Scene Tree、AI会話、Runtime World、生成binaryではない。Schema検証可能なAuthoring Document集合と単調増加する`ProjectRevision`がauthorityである。

AI、Editor、CLI、MCP、外部IDEは同じ`ProjectChangeSetV1`を提案する。Stateを確定できるのはAuthoring Command Gatewayだけであり、expected revision、全primitive、Policy、Validator、authorizationを検証して一つのrevisionとしてatomic commitする。

[Product Lifecycle](../00-product/product-lifecycle.md)はTemplate、Target selection、Documentation、Engine releaseを一つのbootstrap profileへcompositionするが、最初のProject作成も本書のAuthoring Command Gatewayとatomic publicationを使用する。全initial DocumentとSource treeを一つのprepared candidateとして検証し、成功時だけ最初の`ProjectRevision`を発行する。失敗時はcurrent revision、openable partial Projectまたは権威を持つdestination manifestを残さない。Product Lifecycle、Editor、CLI、headlessにProject stateの別write pathを設けない。

## 2. 決定権と対象外

Project State OwnerはDocument identity、Project containment、revision、transaction、commit、undo／recoveryを決定する。各Document payloadのDomain意味、Asset import、Runtime package、Save、Gameplay System、World spatial semanticsは各Ownerが決定する。

UI表示、AI suggestion、filesystem watcher、generated projectionはproposalまたはviewであり、Project authorityへ直接書き込まない。

## 3. Project aggregate

Project aggregateはProject identity、revision、Document set、Active Definition ref、Target readiness summary、Decision Ledger、Source／Derived rootを束縛する。Documentの部分集合だけをProject snapshotと呼ばず、異なるrevisionのDocumentを混在させない。

### 3.1 正規Document

全Documentはstable document ID、kind、schema ref、owner ref、revision、content hash、semantic hash、dependency refsを持つ。Path、表示名、Editor object identityをDocument identityとして保存しない。

具体Document候補は[Target Readiness／Fixture Catalog](../appendices/project-target-readiness-fixture-catalog.md#31-正規document)へ分離する。補助Catalogの種類やFieldをmaterialized Schemaまたはcurrent Registryと解釈しない。

### 3.1.1 `RuntimeEntryPointV1`

Runtime EntryはTarget条件と起動対象のtyped selectorを結び、World、UI-only、headless等のbranchをclosed tagged unionで表す。Root Scene名、配列先頭、Editor選択中objectを暗黙defaultにしない。

#### 3.1.1.1 Target Runtime Entry presentation binding

Presentation bindingはTarget／configurationとRuntime Entryの表示・選択projectionを結ぶ。UI表示順はRuntime activation priorityまたはdefault authorityを変更しない。具体候補は[補助Catalog](../appendices/project-target-readiness-fixture-catalog.md#3111-target-runtime-entry-presentation-binding)を参照する。

<a id="312-runtime-entry"></a>

### 3.1.2 Runtime Entryのclosed Operation Catalog

Runtime Entryのcreate／update／selector／activation policy／legacy migrationは[Executable Contracts](../02-foundation/executable-contracts.md#81-project-runtime-entryruntime-scopeの正規operation登録)の共通Operation境界を使用する。Project StateはDocument identity、expected revision、transaction、postconditionを所有する。

具体Operation input／result／Policy／Diagnostic候補は[補助Catalog](../appendices/project-target-readiness-fixture-catalog.md#312-runtime-entryのclosed-operation-catalog)へ分離する。候補表の存在だけでOperationをactiveにしない。

### 3.2 共通Document header

Document headerはidentity、kind、schema、owner、revision、hash、dependencyを共通化する。Payload、Domain invariant、lifecycleを共通headerへ押し込まず、unknown sibling Fieldを拒否する。

### 3.3 Decision Ledgerの有効性

Decision LedgerはProject固有判断、subject、choice、reason、supersedesをappend-onlyに記録する。Architecture ADR、Approval Receipt、runtime event logを兼用しない。Current choiceは同一subjectの非superseded recordが一つである場合だけ解決する。

### 3.4 Target readiness

Target readinessはTarget、configuration、Runtime Entry、Toolchain lock、required Capability、package profile、Evidence freshnessのclosureである。単一Boolean、Editor上の緑表示、Build成功をrelease readinessと同一視しない。

具体Envelope、predicate、fixture候補は[Target Readiness／Fixture Catalog](../appendices/project-target-readiness-fixture-catalog.md#34-target-readiness)へ分離する。

## 4. World Model

Project StateはWorld／Scene／EntityをAuthoring Documentとしてcontainできるが、spatial topology、streaming、rendering、Gameplay進行を所有しない。Projectionは同一revisionのSource closureから生成し、Editor viewをSourceへ逆serializeしない。

Large SceneのShard、Index、Sliceは同じWorld Source identityとrevisionへ束縛し、部分sliceを完全Worldとしてcommitしない。

### 4.1 `AuthoringSelectionContextV1`

Project State Ownerは`AuthoringSelectionContextV1`のProject lineage、target解決、revision binding、content hashとinvalidation semanticsを所有する。Editor Workspaceはpointer／keyboard／UIA／AIから届くselection intent、attention channel、focus、follow／pinを所有するが、同名Schemaを再定義しない。

`AuthoringSelectionContextV1`はexact Project ref／revision、owner-typed target refのbounded集合、明示primary targetまたは`null`、target set hash、query selectionを使う場合のquery ref／omitted range／continuation、invalidation condition、context content hashを持つread-only disposable projectionである。空集合ではprimary targetを`null`にし、非空集合のprimary targetは集合内exact一件でなければならない。World／Scene／Entity、Assetまたはowner-typed Sourceのpayloadを複写せず、各refを同じProject revisionのDocument／Indexへ解決する。

このContextはselectionの説明とbounded read queryを結ぶためのものであり、Project mutation、lock、authorization、Approval、AI scopeまたはRuntime object selectionを意味しない。display name、path、Hierarchy row、screen coordinate、pointer hit、Editor object addressをtarget refに変換して保存しない。Project revision、target existence／owner／revision、query result generationのいずれかが変わればstaleにし、別targetへの自動rebaseを禁止する。

上記はtarget contractであり、materialized Schema、collection bound、query Tool、Fixture、Receiptはcurrent Repositoryに存在しない。AIまたはEditorが同名の自由JSONを生成して不足を補わない。

## 5. `ProjectChangeSetV1`

ChangeSetは一つのbase Project、expected revision、semantic intent、authorization、ordered primitive集合、precondition、postconditionを束縛する。Primitiveを個別commitせず、全成功または変更0件にする。

### 5.1 Envelope

```text
ProjectChangeSetV1
  changeset_id
  project_ref
  expected_project_revision
  semantic_intent_hash
  authorization_binding_ref
  primitives[]
  precondition_refs[]
  postcondition_refs[]
  requested_by
  idempotency_key
```

ChangeSet ID、intent、idempotency keyの再利用時に別payloadを受理しない。Authorizationはsubjectとscopeをexactに束縛し、Approvalをtechnical validationで代用しない。

### 5.2 `ProjectChangePrimitiveV1`

Primitiveはcreate、replace、delete、rename／move metadata、set membership、typed migration等のclosed tagged unionである。Domain payloadの自由patch、JSON Pointerによるprivate Field変更、unknown operationを許可しない。具体union候補は[補助Catalog](../appendices/project-target-readiness-fixture-catalog.md#52-projectchangeprimitivev1)を参照する。

### 5.3 Commit algorithm

1. Authorization、Project identity、expected revisionを検証する。
2. 全refをbase revisionから解決し、primitiveをprivate candidateへ適用する。
3. Schema、Domain Validator、dependency、Owner、postconditionを検証する。
4. secret-free Public Commit Closure候補とsigned wrapperをread-backする。
5. Closure、Public Marker、after Projectを一つのCASでpublicationする。
6. Conflictまたはcrash時はlast-valid revisionを維持し、partial public objectを0件にする。

## 6. Source layoutと永続化

Source、Derived、cache、temporary、Evidence、package outputを分離する。Source DocumentだけがProject revisionへ参加し、Derived artifactはsource ref、generator、Toolchain、Target、hashから再生成可能にする。

Path変更はDocument identity変更ではない。外部編集はparse、schema、expected revision、semantic diffを通し、filesystem timestampまたはlast writer winsでcommitしない。

## 7. Undo／Redo、外部編集、競合

Undo／Redoは過去bytesの上書きではなく、新しいrevisionとしてinverse ChangeSetを適用する。既に他変更が入った場合はpreconditionを再検証し、競合を黙示mergeしない。

## 8. AIと手動編集

AIと人間は同じproposal、validation、commit境界を使う。AI生成Source、手動編集、Editor operationを別の弱い経路へ分けない。説明またはpreviewはState mutation権限を与えない。

## 9. Runtime compile境界

RuntimeはProject Sourceを直接読むのではなく、qualified Cooked／Package artifactを読む。Compile／CookはProject revision、Document closure、Contract set、Toolchain、Targetを束縛し、stale artifactを黙示利用しない。

## 10. Failure policy

Schema failure、dangling ref、revision conflict、authorization denial、validator failure、publication conflictはtyped Diagnosticを返す。失敗時はProject revision、Source、Public Markerを変更せず、partial Derived outputをcurrentにしない。

## 11. TestとRelease Gate

Project transactionはpositive／negative、conflict、crash recovery、undo／redo、external edit、Target readiness、Cook boundaryを検証する。具体Fixture候補は[Target Readiness／Fixture Catalog](../appendices/project-target-readiness-fixture-catalog.md#11-testとrelease-gate)へ分離する。

## 12. 一次資料

- [Git object model](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects)
- [SQLite Atomic Commit](https://www.sqlite.org/atomiccommit.html)
- [JSON Schema Draft 2020-12](https://json-schema.org/draft/2020-12)
