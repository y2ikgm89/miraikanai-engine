# Miraikanai Engine Runtime Package Contract

- 文書ID: mirakan.arch.runtime-package
- 状態: review
- 正本範囲: Runtime World Root／Section image、World capacity record、section entity record set、Runtime Package directory・binary integrity、loader staging、section publication／retirement、World artifactとgeneric artifact envelopeの接続
- 非正本範囲: generic Derived Artifact manifest／catalog、ECS storage・query・lease、Save／Replay record、runtime phase／job DAG、Domain World source意味、debug transport、AI認可。各Owner文書を参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Runtime ECS](entity-component-system.md)、[Scheduling／Lifetime](scheduling-lifetime.md)、[Persistence／Save](persistence-save.md)、[Performance／Capacity](performance-capacity.md)、[World](../06-rendering/world.md)
- 外部根拠検証日: 2026-07-24

## 1. 状態と結論

Runtime Packageは、承認済みProject revisionからCookされたimmutable World Root／Section imageを、検証済みの一単位としてRuntime World build gatewayへ渡す。World imageはAuthoring object、Editor object、live ECS chunk、native objectを含まない。[Memory／Pointers](../02-foundation/memory-pointers.md)のbindingにより、Packageが渡すのはimmutable bytes、stable／typed reference、bounded readだけであり、live pointer、lease、allocator、runtime slotをdirectory・binary・handoffへ投影しない。

generic artifact identity、promotion、catalogは[Asset lifecycle](../03-authoring/asset-lifecycle.md)が所有する。Runtime Packageは`DerivedArtifactManifestV1`を再定義せず、World RootまたはWorld Sectionというtagged artifact subjectとartifact roleを解決してpayloadを読む。

本書はtarget review Contractであり、Package reader、binary format、loader、section streamingがcurrentにactiveであることを意味しない。current化にはRuntime ECS正本化ChangeSet、Package Ownerのdefinition migration、complete／zero-verified Consumer Inventory、Compatibility Change、Owner reference migration manifest、source／target Foundation Definition Closure、全Evidence Requirementのpass satisfaction binding、qualification evidenceが同一closureで必要である。

## 2. World artifactの境界

World artifactのsubjectはAsset Lifecycle Ownerが定める`ArtifactSubjectRefV1`の次のvariantだけを使う。

| subject kind | artifact role | Runtime Packageの責務 |
|---|---|---|
| `world_root` | `runtime_world_root` | Runtime entry、section directory、capacity、global construction input |
| `world_section` | `runtime_world_section` | section entity record、section-local dependency、load／unload input |

World Root／Sectionをasset-only manifestへ偽装するsynthetic Asset ID、`asset://` URI、Domain固有Catalogを作らない。artifact keyは`{artifact_subject_ref canonical bytes, artifact_role_id}`で一意にし、Catalog valueの共通`ArtifactRefV1`をAsset Lifecycleから受け取る。

## 3. Capacityとsection entity records

### 3.1 World capacity record

```text
RuntimeWorldCapacityRecordV1
  record_version: 1
  world_root_id: StableId
  target_profile_ref: TargetProfileRefV1
  max_loaded_sections: positive uint32
  max_runtime_entities: positive uint32
  max_archetypes: positive uint32
  max_chunks: positive uint32
  max_structural_deltas_per_boundary: positive uint32
  max_query_work_units_per_advance: positive uint32
  section_reservation_records[0..4096]:
    world_section_id
    entity_reservation
    chunk_reservation
    memory_budget_ref: ProjectScaleEnvelopeRefV2
  capacity_envelope_ref: ProjectScaleEnvelopeRefV2
  record_hash: SHA-256
```

capacity recordはWorld build／section publicationに必要な上限とreservationを表す。global capacity envelope、測定方法、backpressureは[Performance／Capacity](performance-capacity.md)が所有し、本recordはそのOwnerが承認したrefを消費する。loaderは予約を超えるSection、Entity、chunk、delta、work unitをstagingへ入れない。

### 3.2 Data-oriented construction closure

Package Candidateは次のexact transitive closureをcontent-addressed refで束縛する。

```text
Package Candidate
  -> exact RuntimeWorldCapacityRecordV1 ref
  -> exact RuntimeEntityConstructionSetV1 root
  -> RuntimeArchetypeLayoutPlanV1 refs
  -> RuntimeComponentLayoutPolicyV1 refs
  -> exact Contract Set root
  -> MemoryContractV1 and PointerMemoryConsumerBindingV1 refs
```

Package assemblyとloaderは、すべてのrefが同じTarget Profile、Contract Set、Toolchain lock、Component schema version、Candidate rootへ解決することを検証する。Runtime Packageはlayout policyやmemory contractのfield listを複製せず、各Ownerのexact refとroot hashだけを保持する。

### 3.3 Section entity record set

```text
RuntimeWorldSectionEntityRecordSetV1
  record_set_version: 1
  world_root_id: StableId
  world_section_id: StableId
  section_revision: positive uint64
  section_dependency_refs[0..4096]: sorted unique ArtifactRefV1
  entity_records[0..1048576]:
    entity_record_id: StableId
    entity_template_ref: EntityTemplateRefV1
    initializer_spec_ref: InitializerSpecRefV1
    persistent_identity_ref: optional PersistentEntityIdentityRefV1
    initial_component_value_refs[0..1024]
    lifetime_policy_ref
  canonical_entity_order_hash: SHA-256
  record_set_hash: SHA-256
```

entity recordはRuntime Entity handle、chunk ID、row、native pointer、Editor pathを持たない。`entity_record_id`はSection payload内のimmutable record identityであり、runtime slotやpersistent identityの代替ではない。persistent identityが必要なrecordだけがexplicit refを持ち、identity collision、template／initializer mismatch、section dependency missingをloader preflightで拒否する。

record orderは`entity_record_id`のcanonical byte順とし、payload file offset、Cook worker順、OS filesystem順を意味にしない。section entity record setはECSの`RuntimeEntityConstructionSetV1`へ入力として渡されるが、archetype row placementやlive location tableを所有しない。

## 4. World Root／Section image

```text
RuntimeWorldRootImageV1
  image_version: 1
  world_root_id: StableId
  source_project_revision_ref: ProjectRevisionRefV1
  runtime_entry_ref: DocumentRef<RuntimeEntryPointDocumentV1>
  capacity_record_ref: RuntimeWorldCapacityRecordRefV1
  construction_set_ref: RuntimeEntityConstructionSetRefV1
  section_directory_refs[0..4096]: RuntimeWorldSectionImageRefV1
  world_root_dependency_refs[0..4096]: ArtifactRefV1
  contract_set_ref: ContractSetRefV1
  target_profile_ref: TargetProfileRefV1
  image_hash: SHA-256

RuntimeWorldSectionImageV1
  image_version: 1
  world_root_id: StableId
  world_section_id: StableId
  section_revision: positive uint64
  entity_record_set_ref: RuntimeWorldSectionEntityRecordSetRefV1
  section_dependency_refs[0..4096]: ArtifactRefV1
  required_capacity_reservation_ref
  unload_policy_ref
  image_hash: SHA-256
```

Root imageはWorld全体のentry、capacity、construction input、section directoryを持つ。Section imageはsection-local entity recordとdependency、reservation、unload policyだけを持つ。World topology、cell、streaming priority、visual representationの意味は[World](../06-rendering/world.md)が所有し、Runtime Packageはそのtyped refとCooked projectionを受け取る。

Root／Section imageは同一Project revision、Contract set、Target profile、Catalog generationへ閉じる。RootとSectionを異なるrevisionまたはtargetから組み合わせる、欠落Sectionを空Sectionとして補完する、hash一致だけでschema／capacity／dependencyを省略することを禁止する。

## 5. Runtime Package binary

### 5.1 Package descriptorとdirectory

```text
RuntimePackageV1
  package_version: 1
  package_id: StableId
  package_revision: positive uint64
  target_profile_ref: TargetProfileRefV1
  contract_set_ref: ContractSetRefV1
  catalog_ref: ArtifactCatalogRefV1
  world_root_image_ref: RuntimeWorldRootImageRefV1
  directory_ref: RuntimePackageDirectoryRefV1
  package_root_hash: SHA-256

RuntimePackageDirectoryEntryV1
  entry_kind: world_root_image | world_section_image | dependency_blob
  subject_ref: ArtifactSubjectRefV1
  artifact_role_id: ClosedArtifactRoleId
  payload_offset: uint64
  payload_length: positive uint64
  payload_hash: SHA-256
  alignment: positive uint32

RuntimePackageDirectoryV1
  directory_version: 1
  entries[]: RuntimePackageDirectoryEntryV1 sorted by
    subject_ref canonical bytes, artifact_role_id
  directory_hash: SHA-256
```

### 5.2 Immutable record reference forms

Runtime Packageが公開するRefは次のclosed shapeだけを使う。各Refは同じrecordのversion、identity field、自己hashをbyte equalityで解決し、`path`、`offset`、live handleをreference identityに使わない。

```text
RuntimeWorldCapacityRecordRefV1
  record_version: positive uint32
  world_root_id: StableId
  target_profile_ref: TargetProfileRefV1
  record_hash: SHA-256

RuntimeWorldSectionEntityRecordSetRefV1
  record_set_version: positive uint32
  world_root_id: StableId
  world_section_id: StableId
  section_revision: positive uint64
  record_set_hash: SHA-256

RuntimeWorldRootImageRefV1
  image_version: positive uint32
  world_root_id: StableId
  image_hash: SHA-256

RuntimeWorldSectionImageRefV1
  image_version: positive uint32
  world_root_id: StableId
  world_section_id: StableId
  section_revision: positive uint64
  image_hash: SHA-256

RuntimePackageDirectoryRefV1
  directory_version: positive uint32
  directory_hash: SHA-256

RuntimePackageRefV1
  package_version: positive uint32
  package_id: StableId
  package_revision: positive uint64
  package_root_hash: SHA-256
```

binary headerはformat major／minor、Package ID／revision、Target Profile hash、Contract set hash、directory location、root hashを持つ。loaderはbounds、overlap、integer overflow、duplicate key、unknown major、truncation、trailing bytes、payload hash mismatch、directory noncanonical orderを拒否する。

directoryはbounded range readを可能にするが、Package全体のmemory mapやlive object pointerを前提にしない。binaryはcanonical MCD field encodingを保持し、C++ layout、padding、endianness推測、pointer fixupをformatに使わない。

### 5.3 Artifact linkage

Package assemblyはAsset LifecycleのCatalogとpromotion closureを消費する。Runtime Packageが独自のCook、artifact hash、Catalog substitution、content mount policyを定義してはならない。Section payloadが必要とするAsset／Domain artifactは`section_dependency_refs[]`へ明示し、外部dependencyはgeneric manifestとCatalog launch setだけが所有する。

## 6. Loaderとpublication

### 6.1 Load staging

Loaderは次を順に検査する。

1. Package root hash、directory、target、Contract set、Catalog generationを検証する。
2. Root imageと選択Section imageのschema、revision、dependency、capacity reservationを検証する。
3. `RuntimeWorldBuildGatewayV1`を作り、ECS construction setをstagingする。
4. persistent identity、template、initializer、State store binding、capacityを検証する。
5. 全Sectionとdependencyがreadyである場合だけRuntime Worldへpublishを要求する。

World construction前に、missing／extra／duplicateなlayout policyまたはarchetype layout ref、layout policyと異なるComponent schema hash、alignment／payload計算後のrow capacity 0、unbounded archetype permutationまたはstructural delta capacity、capacity recordに覆われないquery／command／output reservationを拒否する。old AoS、sparse-set、object-graphのPackage section、old generated signature、pointer-backed inline payload、persisted row selection、およびglobal `new`、default PMR、第二のShipping storage backendへのfallbackも拒否する。

load中のimage、decoded record、reservationはstagingだけに存在する。failure、cancel、stale Project revision、missing dependency、capacity不足、identity collisionではlast-valid World publicationを維持し、partial Worldをpublishしない。

上記のいずれかに失敗した場合は、partial Worldも自動修復したPackageもpublishしない。

### 6.2 Section publication

```text
RuntimeWorldSectionPublicationSetV1
  publication_set_version: 1
  world_instance_handle: RuntimeWorldInstanceHandle
  expected_world_publication_generation: positive uint64
  section_add_refs[0..4096]: RuntimeWorldSectionImageRefV1
  section_remove_ids[0..4096]: StableId
  capacity_record_ref: RuntimeWorldCapacityRecordRefV1
  structural_boundary_ref: RuntimeStructuralCommitBoundaryRefV1
  publication_reason: initial_load | approved_activation | section_streaming
  publication_hash: SHA-256
```

section add／removeはRuntime ECS structural transactionと同じpublication boundaryへ参加する。addはentity record、dependency、reservation、identityを先にvalidateし、removeはunload policy、outstanding lease、persistent handoff、external consumer snapshotをvalidateする。全件が成功した時だけsection membershipとWorld publication generationを更新する。

Package loaderはvalue writeを観測せず、ECSはdirectory offsetやartifact storeを観測しない。section publicationが失敗した場合、旧section setと旧publication generationを保持する。

### 6.3 Persistent handoff

section間またはSectionからWorld Rootへのpersistent entity handoffは、Persistence Ownerのidentity・reconstruction policyとECS structural transactionの双方に合格する場合だけ許可する。raw RuntimeEntityHandleを保持して移送するのではなく、persistent identity、source／destination projection hash、template／initializer、lifetime owner、handoff boundaryを検証する。Save recordとreplay projectionのfieldは[Persistence／Save](persistence-save.md)が所有する。

## 7. Compatibilityとretirement

World Root／Section image、directory、Package binaryの変更は[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)のCompatibility ChangeSetを必要とする。`runtime_package` classを含むConsumer Inventoryがcompleteかつzero verifiedで、scope Requirementがpass fulfillment済みである場合だけ、target ECS clean breakで旧Package reader、dual directory、synthetic Asset ID、old manifest aliasを残さず、committed Sourceからfull recookする。

cache、old Package、old World imageを物理的にいつ消去するかは本書の対象外である。ただしcurrent candidateがold bytesをinputとして受理しないこと、last-valid release rollbackが必要ならexplicit versioned reader migrationを別途承認することは必須である。

## 8. Qualification

target Runtime Packageは少なくとも次を証明する。

loader／capacity qualification receiptは、同じCandidate、Target、Contract Set、Toolchain、fixture、input traceを持つexact `RuntimeDataOrientedQualificationCampaignV1`を参照し、そのhard-predicate resultを消費する。sampling、metric定義、promotion判定は[Performance／Capacity](performance-capacity.md)が所有し、Runtime Packageは閾値や集計方法を再定義しない。

1. 同じProject revision、Catalog、Target、Contract setから同じRoot／Section image、directory、Package root hashを二回生成する。
2. bounds、overlap、duplicate key、hash mismatch、target mismatch、Contract mismatch、unknown major、truncation、trailing bytesをrejectする。
3. missing dependency、capacity overrun、identity collision、template mismatch、stale section revisionでpartial publicationしない。
4. section add／removeがECS boundary外のlive storageを直接変更せず、失敗時にold publicationを保持する。
5. Package、ECS、Persistence、Asset Lifecycle間にraw pointer、live handle、synthetic Asset ID、old aliasを渡さない。
6. source-preserving recookが旧Package bytesに依存せず、Catalog／dependency／qualification closureをread-backできる。
7. Consumer InventoryのPackage／distribution scopeと全Evidence Requirementのpass fulfillment、Compatibility Change、Owner reference migration manifest、source／target Definition Closure、Definition Migration bindingが同じclosureへexact解決する。

## 9. 非目的

本書はWorld streaming implementation、Package writer、loader implementation、Task Plan、cache削除を実行しない。各実施は承認済みdefinition migrationとProduct Work Packageの条件が満たされた後に開始できる。
