# Miraikanai Engine Authoring Model／Project State規約

- 文書ID: mirakan.arch.project-state
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Project aggregate、Authoring Document、ProjectRevision、ProjectChangeSetV1の意味とtransaction、Target readiness意味、Commit、Source／Derived境界、Project Source closure／canonical transport artifactとexact wire grammar、Undo／Redo、外部編集、Recovery、Version Control／repository interoperability、authoring target selection projectionの所有境界
- 非正本範囲: 具体Document／Operation／Change primitive／readiness／fixture候補、VCS provider UI／credential／remote hosting、MCD共通Envelope、命名・Project配置、Asset lifecycle、Editor表示、Gameplay System、Native ABI、Runtime package
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[Core Architecture](../02-foundation/core-architecture.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Naming／Project Layout](../02-foundation/naming-project-layout.md)
- 関連文書: [Target Readiness／Fixture Candidate Catalog](../appendices/project-target-readiness-fixture-catalog.md)、[Product Lifecycle](../00-product/product-lifecycle.md)、[Product Security](../01-governance/product-security.md)、[Product Privacy／Data Governance](../01-governance/product-privacy-data-governance.md)、[Asset Lifecycle](asset-lifecycle.md)、[Editor Workspace UX](editor-workspace-ux.md)、[Developer Testing](developer-testing.md)、[Gameplay Programming Model](gameplay-programming-model.md)、[Runtime Package](../04-runtime/runtime-package.md)、[World](../06-rendering/world.md)
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

Runtime Entryのcreate／update／selector／activation policyに関するtarget Operation候補は[Executable Contracts](../02-foundation/executable-contracts.md#81-project-runtime-entryruntime-scopeのtarget-operation候補)の共通境界を使用する。Project StateはDocument identity、expected revision、transaction、postconditionを所有する。現RepositoryのOperation集合は空であり、initial V1にlegacy migration Operationを含めない。

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
  request_id: UUIDv7
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

`request_id`は一回のAuthoring Command Gateway試行を相関するidentityであり、`changeset_id`または`idempotency_key`の代替ではない。全producerは`request_id`を必須で発行し、Editor commandから生成する場合は`EditorCommandRequestV1.command_request_id`とbyte equalityにする。ChangeSet ID、intent、idempotency keyの再利用時に別payloadを受理しない。Authorizationはsubjectとscopeをexactに束縛し、Approvalをtechnical validationで代用しない。

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

### 7.1 Version Control／repository interoperability

Project Sourceは一般的なVersion Controlで追跡できるstable file集合として投影する。VCS commit、branch、index、working tree、remote、lockはProject authorityではなく、`ProjectRevision`とSource closureの外部transport／collaboration boundaryである。Git object ID、branch名、path、filesystem timestampをDocument identityまたはProject revisionにしない。

```text
ProjectSourceEntryV1
  source_entry_id: StableId
  source_entry_version: 1
  project_ref: exact ProjectRefV1
  project_revision_ref: exact ProjectRevisionRefV1
  source_role:
    authoring_document
    | project_configuration_source
    | asset_source
    | native_source
    | shader_source
    | project_test_source
    | dependency_lock
    | pack_lock
    | migration_input
  canonical_project_relative_path: normalized UTF-8
  source_byte_length: uint64
  source_content_hash: SHA-256
  source_entry_content_hash: SHA-256

ProjectSourceEntryRefV1
  source_entry_id: StableId
  source_entry_version: 1
  source_entry_content_hash: SHA-256

ProjectSourceClosureV1
  source_closure_id: StableId
  source_closure_version: 1
  project_ref: exact ProjectRefV1
  project_revision_ref: exact ProjectRevisionRefV1
  source_entry_refs[1..1048576]:
    sorted unique exact ProjectSourceEntryRefV1
  canonical_transport_artifact_ref:
    exact ArtifactRefV1(
      artifact_kind=project_source_transport,
      schema_version=1)
  source_closure_content_hash: SHA-256

ProjectSourceClosureRefV1 =
  exact ArtifactRefV1(
    artifact_kind=project_source_closure,
    schema_version=1)

ProjectRepositorySnapshotV1
  project_ref: exact ProjectRefV1
  project_revision_ref: exact ProjectRevisionRefV1
  source_closure_ref: exact ProjectSourceClosureRefV1
  source_closure_hash: SHA-256
  repository_provider: git | filesystem_snapshot | external
  repository_root_identity_ref: exact RepositoryRootIdentityRefV1
  observed_worktree_state:
    clean | modified | conflicted | unavailable
  provider_revision_ref: optional exact RepositoryProviderRevisionRefV1
  conflict_entry_refs[0..4096]:
    sorted unique exact RepositoryConflictEntryRefV1
  ignored_source_entry_refs[0..4096]:
    sorted unique exact RepositoryPathRefV1
  snapshot_content_hash: SHA-256

ProjectRepositorySnapshotRefV1
  project_ref: exact ProjectRefV1
  project_revision_ref: exact ProjectRevisionRefV1
  source_closure_hash: SHA-256
  snapshot_content_hash: SHA-256
```

`ProjectSourceEntryV1`はPathをidentityにせずstable `source_entry_id`を使う。`canonical_project_relative_path`はNaming／Project Layoutのseparator、Unicode、case collision、reserved name、root escape、encoding、line-ending policyへ従う。同じclosure内では`source_entry_id` projectionと`canonical_project_relative_path` projectionをそれぞれuniqueにし、同一stable IDへ異なるpath／role／bytesを割り当てる、または同一pathへ異なるstable IDを割り当てることを拒否する。各Entry refは同じProject／revisionの完成Entryへexact解決し、`source_content_hash`はそのSource bytes、`source_entry_content_hash`はASCII `MIRAKAN_PROJECT_SOURCE_ENTRY_V1`と自身を除く全FieldのMCD canonical bytesを各`uint32_be` length framingした列のSHA-256にする。

`ProjectSourceClosureV1.source_entry_refs[]`はそのProject revisionへ参加する全canonical Project Sourceの完全なsorted setである。canonical comparatorは`project_source_entry_ref_mcd_bytes_lexicographic_v1`だけとし、各`ProjectSourceEntryRefV1`のExecutable Contracts Owner準拠MCD canonical bytes全体をunsigned byte列として先頭から辞書式比較する。最初の異なるbyteが小さいRefを先、片方が他方の完全prefixなら短いRefを先とし、全byteが同じ場合だけduplicateとする。Stable ID、path、role、content hash、locale／case変換、producer列挙順またはそれらの一部を別sort key／tie-breakへ使わない。このcomparatorをClosure record、Closure content hashおよび`ProjectSourceTransportWireV1`の反復順へ同一適用する。

Source role、stable entry identity、canonical path、byte length、content hashのいずれかが異なるEntryを同一視せず、Derived、cache、BMI、build、package、test result、Evidence、credential、User preferenceまたはlocal Workspaceを混入させない。`source_closure_content_hash`はASCII `MIRAKAN_PROJECT_SOURCE_CLOSURE_V1`と、自身を除く全FieldのMCD canonical bytesを各`uint32_be` length framingした列のSHA-256である。`ProjectSourceClosureRefV1.sha256`は`source_closure_content_hash`を含む完成record bytesのSHA-256へ一致し、bare closure hash、Project revision、path集合またはVCS tree IDをRefとして受理しない。

`canonical_transport_artifact_ref`が指すbytesは次のexact wire grammarだけで生成する。`uint32_be`／`uint64_be`はunsigned network byte order、`bytes[n]`は直前のlengthとexactに同じbyte数であり、alignment、padding、NUL終端、delimiterまたは暗黙lengthを持たない。

```text
ProjectSourceTransportWireV1
  bytes[35]: exact ASCII "MIRAKAN_PROJECT_SOURCE_TRANSPORT_V1"
  uint32_be: entry_count
  repeat entry_count times in
    project_source_entry_ref_mcd_bytes_lexicographic_v1 order:
    uint32_be: entry_ref_byte_length
    bytes[entry_ref_byte_length]:
      exact ProjectSourceEntryRefV1 MCD canonical bytes
    uint32_be: canonical_path_byte_length
    bytes[canonical_path_byte_length]:
      exact normalized canonical_project_relative_path UTF-8 bytes
    uint32_be: source_role_byte_length
    bytes[source_role_byte_length]:
      exact closed source_role token ASCII bytes
    uint64_be: source_byte_length
    bytes[source_byte_length]: exact raw Source bytes
  end_of_input
```

`entry_count`は`1..1048576`かつClosureの実`source_entry_refs[]`件数と一致し、各反復は同じcomparator indexの完成Entry recordへ解決する。三つのlengthと全offset加算はoverflowを起こさないchecked arithmeticで検証し、`entry_ref_byte_length`、`canonical_path_byte_length`、`source_role_byte_length`はそれぞれ`1..4294967295`、`source_byte_length`は`0..18446744073709551615`の範囲内で実bytes数と一致させる。Source bytesのhash／lengthはEntry recordとbyte equalityにし、最後のSource byte直後だけを`end_of_input`としてtrailing byteを拒否する。countまたはlengthの`uint64_be`／varint／ASCII decimal化、little-endian、delimiter framing、Field省略、別MCD encoding、別Entry順序、別comparator、missing／extra Entry、改変bytes、compression、archive timestamp、owner、permission、filesystem inode、symlinkまたはprovider metadataの混入を禁止する。

Qualification用wire-vector候補は少なくとも一Entry、zero-length Source、non-ASCII canonical path、count／各lengthの境界、role差、Entry順序差、trailing byte、count mismatch、big-endianからlittle-endianへの置換を各一原因でcoverし、同じEntry record／Source bytesから生成した全conforming writerのtransport bytesとArtifact Refがbyte equalityであることを要求する。これはtarget contractであり、Fixture、writer、ArtifactまたはReceiptが現在materializeしていることを意味しない。このexact一件のtransport ArtifactだけがSnapshot Source closureのcanonical transport artifactであり、generic Artifactのhash、path、表示名またはReceipt上のref併記からmembershipを推論しない。

`ProjectRepositorySnapshotV1.source_closure_ref`は完成`ProjectSourceClosureV1`へ解決し、Project／revisionをSnapshotとbyte equality、`source_closure_hash`を解決先の`source_closure_content_hash`とbyte equalityにする。`ProjectRepositorySnapshotRefV1`の四Fieldは解決先Snapshotとbyte equalityにし、provider revision、branch名、path、timestampまたは表示中のworktree labelから補完しない。同じProject revisionでもsource closureまたはSnapshot bytesが異なるRefを代用せず、Release、Editor Workspace、Support projectionはexact Refを保存する。

EditorでProjectを開く時は、Project manifest、Source closure、repository root、provider revision、working-tree差、未解決conflictをread-only scanし、どのbytesをProject authorityとしてparseするかを表示する。untracked／modified Sourceを黙って破棄、stash、commit、resetまたはcheckoutしない。repository操作はpreview、対象path、provider command semantics、credential／remote影響、expected provider head、Project revision影響を示した明示Operationにする。

外部IDEまたはVCS clientによる変更は、filesystem watcher eventを直接commitせず、stable read後にbase Project revisionへ対するsemantic Diff候補としてparseする。複数fileの一時保存、rename、generated output、conflict marker、partial mergeを一つずつcurrent Projectへ反映しない。変更集合がschema／owner／dependency validationに合格した時だけ一つの`ProjectChangeSetV1`としてcommitする。

Mergeはtext merge結果の受理ではなく、全Authoring Documentのidentity、schema、semantic dependency、Asset source、Native／Shader source、Pack lock、Target selectionを再検証する。conflict marker残存、同一Document IDの異なる意味、delete／modify、Pack lock conflict、case-only path collision、line-ending／encoding violationをtyped conflictとして止める。binary AssetはOwnerが定めるsource-level mergeまたは一方選択だけを許し、bytesを自動結合しない。

Repositoryへ含めるものはcanonical Project Source、Project test source、Pack／dependency lock、必要なmigration inputである。Derived、cache、BMI、build、package、test result、crash dump、telemetry queue、credential、User preference、local Workspace、provider tokenをcommit対象にしない。ignore ruleがcanonical Sourceを除外する、またはgenerated outputをSourceとして含める場合はProject open／release readinessをfail closedにする。

Project作成、clone／open、branch switch／provider revision change、merge、external edit後は同じreconciliation経路を使う。VCSが利用不能でもlocal Projectを開けるが、remote同期済み、cleanまたはrelease-readyと表示しない。VCS provider integrationはGitを最初のprojectionにできるが、Project formatとpublic authoring contractをGit専用にしない。

## 8. AIと手動編集

AIと人間は同じproposal、validation、commit境界を使う。AI生成Source、手動編集、Editor operationを別の弱い経路へ分けない。説明またはpreviewはState mutation権限を与えない。

## 9. Runtime compile境界

RuntimeはProject Sourceを直接読むのではなく、qualified Cooked／Package artifactを読む。Compile／CookはProject revision、Document closure、Contract set、Toolchain、Targetを束縛し、stale artifactを黙示利用しない。

## 10. Failure policy

Schema failure、dangling ref、revision conflict、authorization denial、validator failure、publication conflictはtyped Diagnosticを返す。失敗時はProject revision、Source、Public Markerを変更せず、partial Derived outputをcurrentにしない。

## 11. TestとRelease Gate

Project transactionはpositive／negative、conflict、crash recovery、undo／redo、external edit、repository clean／modified／conflicted／unavailable、multi-file atomic save、branch／provider revision drift、ignore-rule violation、Target readiness、Cook boundaryを検証する。Source closureはEntryのmissing／extra／duplicate stable ID／duplicate path／content改変／path collision／wrong Project／wrong revision、同revision別closure／Snapshot、別順序／別metadata／別compressionのtransport repack、count／length／endianness／trailing byte違反、Snapshot外Artifact、hash-only／provider revision／path推測を各一原因でrejectする。具体Fixture候補は[Target Readiness／Fixture Catalog](../appendices/project-target-readiness-fixture-catalog.md#11-testとrelease-gate)へ分離する。

## 12. 一次資料

- [Git object model](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects)
- [SQLite Atomic Commit](https://www.sqlite.org/atomiccommit.html)
- [JSON Schema Draft 2020-12](https://json-schema.org/draft/2020-12)
