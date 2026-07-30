# Miraikanai Engine Persistence and Save Contract

- 文書ID: mirakan.arch.persistence-save
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Save Catalog identity／slot membership、Runtime Session Save Bundle、Continue／load resolution、Runtime World Save record、persistent／ephemeral Entity projection、Component lifecycle・enablement projection、authoritative state digest、reconstruction、Replay projection、Save migration・qualification
- 非正本範囲: network connection／participant／session／authority／baseline／replication／prediction buffer／rollback history、ECS storage layout・query・lease、Package binary、generic artifact catalog、debug capture transport、runtime phase／job DAG、Domain field意味、AI authorization。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Runtime ECS](entity-component-system.md)、[Runtime Package](runtime-package.md)、[Scheduling／Lifetime](scheduling-lifetime.md)
- 関連文書: [Architecture Governance](../01-governance/architecture-governance.md)、[Advanced Rendering／Multiplayer Ownership Decision](../decisions/2026-07-29-advanced-rendering-multiplayer-ownership.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Project state](../03-authoring/project-state.md)、[UI／Text／Localization／Accessibility](../07-platform/ui-text-localization-accessibility.md)、[Runtime ECS](entity-component-system.md)、[Runtime Package](runtime-package.md)、[Scheduling／Lifetime](scheduling-lifetime.md)、[Debugging／Observability／Replay](debugging-observability-replay.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[LOD](../06-rendering/lod.md)、[Virtualized／Continuous Geometry](../06-rendering/virtualized-continuous-geometry.md)、[Multiplayer Authority／Replication](../09-networking/multiplayer-authority-replication.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-24

## 1. 状態と結論

Saveのtop-level rootは`RuntimeSessionSaveBundleV1`であり、exact Runtime Entry／branch package、Authoritative Header／Domain projection集合、optional World Saveを一つのimmutable closureへ束縛する。World Saveはlive ECS memoryのdumpではない。seal済みWorld publicationから、Persistent Entity Identity、canonical Component field projection、lifecycle、enablement、sequence、schema／contract provenanceを持つimmutable record setを生成する。reconstructionはPackage／ECS construction contractを再検証して新しいRuntime Entity handleを作るため、Save内のhandle、chunk location、pointer、worker順を再利用しない。[Memory／Pointers](../02-foundation/memory-pointers.md)のbinding上、live pointer、reference、lease、span、writer、allocator objectはSave／Replay projectionに不許可であり、本書はその禁止をpersistent identityとreconstruction fixtureへ具体化する。

ReplayはSaveと同じpayloadではない。Replayはauthoritative input、accepted async result、structural boundary、digest、advance sequenceを再現可能なprojectionとして記録し、capture transportとdebug UIは[Debugging／Observability／Replay](debugging-observability-replay.md)が所有する。

[Multiplayer Authority／Replication](../09-networking/multiplayer-authority-replication.md)はsession／participant／Network Object／authority／baseline、validated command、prediction／rollback、resyncを所有する。本書は同Ownerがexact `PersistentIdentityBindingRefV1`とDomain Save projectionを宣言したauthoritative fieldだけをSaveへ含める。Transport connection／epoch、packet／queue、replication ack、interest／dormancy cache、prediction buffer、rollback snapshotをSave identityまたは復帰shortcutにしない。Load後はnew Runtime Entity generationと必要ならnew Multiplayer session／baselineへ再bindingし、位置、Network Object IDまたは同名participantから旧authorityを復元しない。

本書はinitial V1 review Contractであり、Save reader／writer、Replay reader／writerまたはmigrationのmaterialization／activationを意味しない。Save／Replayの最初のcanonical Schema、identity、reconstruction、failureを本書へ直接定義し、旧Schema、source／target Owner、migration history、aliasまたはdual readerをinitial V1へ作らない。Definition Closure、全Evidence Requirementのpass satisfaction bindingとqualification evidenceが揃うまで利用可能と表示しない。

## 2. Identityとprojection規則

### 2.1 Persistent identity

```text
PersistentEntityIdentityV1
  identity_kind: authoring_entity | world_root_runtime | runtime_spawn
  identity_value: StableId
  identity_revision: positive uint64
  world_root_id: StableId
  owner_ref: OwnerRefV1
  identity_hash: SHA-256

PersistentEntityIdentityRefV1
  identity_kind: authoring_entity | world_root_runtime | runtime_spawn
  identity_value: StableId
  identity_revision: positive uint64
  identity_hash: SHA-256

PersistentEntityIdentityAllocationBindingV1
  allocation_binding_id: StableId
  allocation_binding_version: 1
  persistent_spawn_request_ref:
    exact RuntimePersistentSpawnIdentityRequestRefV1
  allocated_identity_ref: exact PersistentEntityIdentityRefV1(
    identity_kind=runtime_spawn)
  identity_authority_service_ref: exact McdContractRefV1(kind=service)
  allocation_receipt_ref: exact OperationReceiptRefV1
  allocation_binding_content_hash: SHA-256

PersistentEntityIdentityAllocationBindingRefV1
  allocation_binding_id: StableId
  allocation_binding_version: 1
  allocation_binding_content_hash: SHA-256
```

`authoring_entity`はAuthoring stable identityから解決する。`world_root_runtime`はWorld construction authority、`runtime_spawn`はPersistence identity authorityが明示的に発行するpersistent identityであり、Cook時に将来spawnを予約してはならない。raw `RuntimeEntityHandle`、slot index、generation、chunk ID、rowはpersistent identityではない。

runtime spawn allocationはRuntime ECSのexact `RuntimePersistentSpawnIdentityRequestRefV1`だけを入力にする。同じrequest refとidempotency keyは同じAllocation Bindingを返し、別request、World Root、Template、ownerまたはhashへidentityを再利用しない。request一件にAllocation Binding exactly one、allocated identity一件にactive request exactly oneを要求し、duplicate、branch、call-site指定ID、位置／Entity handle／sequenceからのID推測を拒否する。Binding hashは`MIRAKAN_PERSISTENT_ENTITY_IDENTITY_ALLOCATION_BINDING_V1`と自己hashを除くlength-framed canonical bytesでSHA-256し、ECSは有効なBindingとReceiptをstructural batch preflightで検証してもIdentity発行意味を再定義しない。対応Schema、authority service、Operation、Receipt、Binding storeは未materializeである。

Component fieldはDomain Ownerが定めるpersistence policyに従う。persistent fieldだけをcanonical field encodingへ投影し、derived index、presentation value、native object、credential、pointer、runtime-only handleは保存しない。persistent identityを持たないEntityをSaveへ近似せず、required relationがそのEntityを指すならSave validationを失敗させる。

### 2.2 Save record set

```text
RuntimeWorldSaveRecordSetV1
  save_format_version: 1
  save_id: StableId
  source_world_publication_ref: RuntimeWorldPublicationRefV1
  source_project_revision_ref: ProjectRevisionRefV1
  runtime_package_ref: RuntimePackageRefV1
  contract_set_ref: ContractSetRefV1
  save_sequence: positive uint64
  authoritative_save_header_ref: AuthoritativeSaveHeaderRefV1
  authoritative_save_bundle_manifest_ref:
    AuthoritativeSaveBundleManifestRefV1
  authoritative_digest_ref: RuntimeAuthoritativeStateDigestRefV1
  entity_records[0..1048576]: RuntimeEntitySaveRecordV1
  record_set_hash: SHA-256

RuntimeEntitySaveRecordV1
  persistent_identity_ref: PersistentEntityIdentityRefV1
  entity_lifecycle: alive | disabled
  source_section_id: optional StableId
  template_ref: EntityTemplateRefV1
  initializer_spec_ref: InitializerSpecRefV1
  component_records[0..1024]: RuntimeComponentSaveRecordV1
  entity_ordinal: uint64

RuntimeComponentSaveRecordV1
  component_schema_ref: ComponentSchemaRefV1
  component_presence: present
  component_enablement: enabled | disabled | not_applicable
  persisted_field_mask
  canonical_field_value_ref: CanonicalValueRefV1
  component_projection_hash: SHA-256

RuntimeWorldSaveRecordSetRefV1
  save_format_version: positive uint32
  save_id: StableId
  record_set_hash: SHA-256
```

entity recordはpersistent identity canonical bytes順、component recordはComponent schema ref順、fieldはField ID順とする。`entity_ordinal`はsave内のcanonical ordering補助であり、runtime slot、World chunk row、persistent identityの代替ではない。

source World publication `W`に存在するpersistent identityのdistinct集合を`P(W)`とする。`RuntimeWorldSaveRecordSetV1.entity_records[].persistent_identity_ref`のprojectionは`P(W)`とset equality、各identityはexactly one recordでなければならない。`W.runtime_package_ref`が解決するPackageの`RuntimeWorldRootImageV1.capacity_record_ref`からexact `RuntimeWorldCapacityRecordV1.max_persistent_entities`を解決し、World Root、Target、Contract Setをsource publicationとbyte equalityにする。checked arithmeticで`|P(W)| <= max_persistent_entities <= 1048576`を満たす場合だけRecord Setをsealする。ephemeral Entityの省略、duplicate identityのdeduplication、複数Record Setへのshardingまたは`RuntimeSessionSaveBundleV1`外の追加World Save refでcarrier上限を回避しない。

`component_presence = present`だけをSave recordへ書く。remove済みComponent、derived Component、presentation Componentをmissing fieldとして暗黙復元しない。enablementを持たないComponentは`not_applicable`を使い、nullやzero値で代用しない。

### 2.3 Timebase headerとbinding

Schedulerはcadence、advance sequence、completed intervalを一意に発行する。Save payload全体へそれらを束縛するheader、Domain projection binding、bundle rootは本書が所有する。

```text
AuthoritativeSaveStateOwnerProjectionRefV1
  owner_ref: exact {owner_id, owner_revision, owner_content_hash}
  projection_type_ref: McdContractRefV1(kind=type)
  projection_id: StableId
  projection_version: positive uint32
  projection_content_hash: SHA-256

AuthoritativeSaveHeaderRefV1
  header_id: StableId
  header_version: positive uint32
  header_content_hash: SHA-256

AuthoritativeSaveHeaderV1
  header_id: StableId
  header_version: positive uint32
  game_candidate_build_receipt_ref: GameCandidateBuildReceiptRefV1
  game_candidate_build_receipt_sha256: SHA-256
  project_id: UUIDv7
  project_revision: positive uint64
  project_document_set_hash: SHA-256
  contract_set_hash: SHA-256
  target_profile_ref: TargetProfileRefV1
  game_clock_domain_profile_ref: GameClockDomainProfileRefV1
  simulation_cadence_profile_ref: SimulationCadenceProfileRefV1
  physics_substep_activation_binding_ref:
    null | PhysicsSubstepActivationBindingRefV1
  last_committed_simulation_advance_interval_ref:
    SimulationAdvanceIntervalRefV1
  last_committed_simulation_advance_interval_sha256: SHA-256
  state_owner_projection_refs[0..65536]:
    AuthoritativeSaveStateOwnerProjectionRefV1
  state_owner_projection_set_hash: SHA-256
  header_content_hash: SHA-256

AuthoritativeSaveDomainBindingRefV1
  binding_id: StableId
  binding_version: positive uint32
  binding_content_hash: SHA-256

AuthoritativeSaveDomainBindingV1
  binding_id: StableId
  binding_version: positive uint32
  authoritative_save_header_ref: AuthoritativeSaveHeaderRefV1
  state_owner_projection_ref:
    AuthoritativeSaveStateOwnerProjectionRefV1
  binding_content_hash: SHA-256

AuthoritativeSaveBundleManifestV1
  manifest_id: StableId
  manifest_version: positive uint32
  authoritative_save_header_ref: AuthoritativeSaveHeaderRefV1
  domain_binding_refs[0..65536]:
    AuthoritativeSaveDomainBindingRefV1
  manifest_content_hash: SHA-256

AuthoritativeSaveBundleManifestRefV1
  manifest_id: StableId
  manifest_version: positive uint32
  manifest_content_hash: SHA-256

RuntimeSessionSaveBundleV1
  bundle_id: StableId
  bundle_version: positive uint32
  runtime_entry_ref: DocumentRef<RuntimeEntryPointDocumentV1>
  runtime_entry_semantic_hash: RuntimeEntryPointSemanticHashV1
  entry_kind: world | ui | headless
  entry_branch_closure_hash: SHA-256
  runtime_entry_package_ref: RuntimeEntryPackageRefV1
  authoritative_save_header_ref: AuthoritativeSaveHeaderRefV1
  authoritative_save_bundle_manifest_ref:
    AuthoritativeSaveBundleManifestRefV1
  world_save_record_set_ref: RuntimeWorldSaveRecordSetRefV1 | null
  bundle_content_hash: SHA-256

RuntimeSessionSaveBundleRefV1
  bundle_id: StableId
  bundle_version: positive uint32
  bundle_content_hash: SHA-256

SaveContentPackageSetV1
  content_package_set_id: StableId
  content_package_set_version: positive uint32
  package_entries[1..4096]:
    sorted unique {
      package_artifact_ref: exact ArtifactRefV1,
      package_root_hash: SHA-256,
      mount_role: base | patch | dlc | optional_content
    }
  content_package_set_content_hash: SHA-256

SaveContentPackageSetRefV1
  content_package_set_id: StableId
  content_package_set_version: positive uint32
  content_package_set_content_hash: SHA-256

SaveCatalogSlotV1
  slot_id: StableId
  display_metadata_content_hash: SHA-256
  save_schema_version: positive uint32
  runtime_session_save_bundle_ref:
    exact RuntimeSessionSaveBundleRefV1
  content_package_set_ref: exact SaveContentPackageSetRefV1
  transport_checksum: SHA-256
  status: ready | damaged | incompatible | unavailable
  slot_entry_content_hash: SHA-256

SaveCatalogV1
  save_catalog_id: StableId
  catalog_schema_version: 1
  profile_scope_id: StableId
  generation: positive uint64
  content_package_set_ref: exact SaveContentPackageSetRefV1
  slots[0..4096]:
    sorted unique SaveCatalogSlotV1 by slot_id canonical bytes
  slot_entry_set_hash: SHA-256
  save_catalog_content_hash: SHA-256

SaveCatalogRefV1
  save_catalog_id: StableId
  catalog_schema_version: 1
  generation: positive uint64
  save_catalog_content_hash: SHA-256

SaveCatalogActiveRootPreconditionV1
  profile_scope_id: StableId
  expected_commit_generation: positive uint64
  expected_save_catalog_ref: exact SaveCatalogRefV1
  active_root_subject_hash: SHA-256
  precondition_content_hash: SHA-256

SaveCatalogActiveRootPreconditionRefV1
  profile_scope_id: StableId
  expected_commit_generation: positive uint64
  precondition_content_hash: SHA-256

SaveCatalogSlotMembershipV1
  save_catalog_ref: exact SaveCatalogRefV1
  slot_id: StableId
  slot_entry_content_hash: SHA-256
  runtime_session_save_bundle_ref:
    exact RuntimeSessionSaveBundleRefV1
  content_package_set_ref: exact SaveContentPackageSetRefV1
  membership_content_hash: SHA-256

SaveCatalogSlotMembershipRefV1
  save_catalog_ref: exact SaveCatalogRefV1
  slot_id: StableId
  membership_content_hash: SHA-256

RuntimeSessionLoadRequestV1
  request_id: StableId
  idempotency_key: StableId
  play_session_id: StableId
  source_branch_generation: positive uint64
  save_catalog_ref: exact SaveCatalogRefV1
  active_root_precondition_ref:
    exact SaveCatalogActiveRootPreconditionRefV1
  slot_id: StableId
  slot_membership_ref: exact SaveCatalogSlotMembershipRefV1
  runtime_session_save_bundle_ref:
    exact RuntimeSessionSaveBundleRefV1
  content_package_set_ref: exact SaveContentPackageSetRefV1
  requested_apply_advance_sequence: uint64
  request_content_hash: SHA-256

RuntimeSessionLoadRequestRefV1
  request_id: StableId
  request_content_hash: SHA-256

RuntimeSessionLoadActivationPayloadV1
  load_request_ref: exact RuntimeSessionLoadRequestRefV1
  load_request_content_hash: SHA-256
  runtime_session_save_bundle_ref: RuntimeSessionSaveBundleRefV1
  state_owner_projection_set_hash: SHA-256
  world_reconstruction_input_ref: exact typed ref | null
  activation_payload_content_hash: SHA-256

RuntimeSessionLoadActivationPayloadRefV1
  load_request_ref: RuntimeSessionLoadRequestRefV1
  activation_payload_content_hash: SHA-256
```

headerのProject triple、Target、Contract set、Cadence Profile、last committed interval、substep bindingは外側`RuntimeEntryPackageV1`とSchedulerのsealed recordへbyte equalityで解決する。world branchのinner `RuntimePackageV1`はWorld Save recordから別途照合する。state-owner projection refは`owner_ref, projection_type_ref, projection_id, projection_version, projection_content_hash`のcanonical byte順でuniqueにし、header／binding／manifestは自己hashを除くcanonical bytesからhashする。

Domain Projection base recordへHeader refまたはBinding refを埋め戻してhash cycleを作らない。生成順は`receipt-free Domain Projection → Projection Ref集合 → AuthoritativeSaveHeader／Ref → Domain Binding／Ref集合 → Bundle Manifest`とする。bare hash、latest Build、表示Hz、別Target／Project revision、Header外Projectionをload時に補完しない。

`RuntimeSessionSaveBundleV1`はBundle Manifest完成後にだけ生成し、自己hashを除く全FieldをASCII `MIRAKAN_RUNTIME_SESSION_SAVE_BUNDLE_V1`とMCD canonical length framingして`bundle_content_hash`を計算する。Save Content Package Set、Save Catalog、active-root precondition、slot membership、Load Request、activation payloadはそれぞれASCII `MIRAKAN_SAVE_CONTENT_PACKAGE_SET_V1`、`MIRAKAN_SAVE_CATALOG_V1`、`MIRAKAN_SAVE_CATALOG_ACTIVE_ROOT_PRECONDITION_V1`、`MIRAKAN_SAVE_CATALOG_SLOT_MEMBERSHIP_V1`、`MIRAKAN_RUNTIME_SESSION_LOAD_REQUEST_V1`、`MIRAKAN_RUNTIME_SESSION_LOAD_ACTIVATION_PAYLOAD_V1`と自身のhash Fieldだけを除くclosed canonical bytesをlength framingしてhashする。`runtime_entry_ref`／semantic hash／branch closure、外側`RuntimeEntryPackageRefV1`、HeaderのProject triple／Target／Contract set／Cadence、Bundle ManifestのHeader refはbyte equalityでなければならない。`entry_kind=world`だけが`world_save_record_set_ref`を0または1件持て、presentの場合はWorld Save内側`RuntimePackageRefV1`が外側Entry Packageの`world_package_ref`へexact解決する。`ui | headless`はWorld Save refをnullとし、UI／headlessのState owner projectionはBundle Manifestから保存する。UI-only／headless Saveのために偽World publicationまたは空World recordを生成しない。

`SaveContentPackageSetV1`はSaveをLoadするために必要なAsset Content Packageのexact immutable closure projectionであり、各entryのgeneric Artifact ref、package root hash、mount roleを閉じる。Asset Content Packageのformat、assembly、mount semanticsは[Asset Lifecycle](../03-authoring/asset-lifecycle.md)が所有し、本書は複写しない。`SaveCatalogV1`と各Slotはimmutableである。`save_catalog_content_hash`はID、schema、profile scope、generation、Catalog-level content package set、全Slotの全Field、`slot_entry_set_hash`を含み、`slot_entry_set_hash`は各`slot_entry_content_hash`のcanonical sorted setへexact一致する。SlotのBundle ref、content package set、transport checksum、status、display metadata hashの一件でも変われば新Catalog generationと新content hashを発行する。同ID／generation別hash、Slot IDだけ、表示label、timestamp、native path、`latest`、近いcontent setからBundleまたはCatalogを補完しない。

`SaveCatalogSlotMembershipV1`は、解決したexact Catalogの`slots[]`に同じ`slot_id`がexactly one件あり、Slot entry hash、Bundle ref、content package setが全Field byte equalityの場合だけsealする。Membership record自身をCatalogへ埋め戻さず、Catalog全体の再hashとunique membership検証を省略するMerkle proofまたはcaller assertionとして扱わない。

`SaveCatalogActiveRootPreconditionV1`はUI Ownerがcurrent `SettingsCatalogCommitMarkerV1`のProfile scope、commit generation、exact Save Catalog refをclosed projectionして渡す値であり、`active_root_subject_hash`はそのmarker refを含むactive-root subjectのcanonical hashである。PersistenceはUI Profile、marker、file、display stateを検索せず、Load要求に渡されたpreconditionとexact Catalog／membershipだけを検証する。callerはRuntime Entry transition commit直前にも同じactive-root subjectがcurrentであることをCAS検証し、staleならLoadを拒否する。UI Ownerへの逆参照、独立したPersistence側active root、Profile名からのCatalog discoveryを作らない。

`RuntimeSessionLoadActivationPayloadV1.world_reconstruction_input_ref`はbundleの`entry_kind=world`かつWorld Saveがpresentの場合だけexact一件、worldでWorld Save absentまたは`ui | headless`ではnullにする。`state_owner_projection_set_hash`はHeaderの同Fieldとbyte equalityで、Load Request ref／hash、Bundle ref、World reconstruction nullabilityのいずれかが一致しなければpayloadをsealしない。

Stage、Quest、Inventory、Gameplay Timer等のDomain Saveは`AuthoritativeSaveStateOwnerProjectionRefV1`とBindingを通してBundle Manifestへexact membershipを持つ。ContinueはStage表示名や「現在のLevel」を推測せず、保存時に含まれたStage definition／instance／policy projectionをowner validatorへ渡す。Screen Stack、Focus、hover、text draft等のpresentation stateはSession Saveへ含めず、destination Runtime Entryのroot Screenから再構築する。

## 3. Authoritative digest

```text
RuntimeAuthoritativeStateDigestV1
  digest_version: 1
  source_world_publication_ref: RuntimeWorldPublicationRefV1
  world_epoch: positive uint64
  world_publication_generation: positive uint64
  advance_sequence: positive uint64
  contract_set_ref: ContractSetRefV1
  persistent_entity_projection_hashes[]:
    persistent_identity_ref
    entity_projection_hash
  excluded_projection_classes[]:
    ephemeral_entity | derived_index | presentation | native_handle
  digest_sha256: SHA-256

RuntimeAuthoritativeStateDigestRefV1
  digest_version: positive uint32
  source_world_publication_ref: RuntimeWorldPublicationRefV1
  advance_sequence: positive uint64
  digest_sha256: SHA-256
```

digestの`world_epoch`と`world_publication_generation`は`source_world_publication_ref`が解決する同名Fieldとbyte equalityでなければならない。T00のprivate working World、faulted advance、未publish section set、次回向けsealed structural batchからdigestまたはSave projectionを生成しない。

digestはcanonical persistent field projectionから計算する。Componentを`sizeof` bytesでhashしない。Entity relationはpersistent identityへ投影するか、ephemeral／derived relationとして除外根拠を明示する。raw handle、chunk layout、worker index、memory address、wall-clock completion timeをdigest入力にしない。

digestはWorldがsealされた後に一度だけ作り、partial value writeやstaging structural deltaから生成しない。Save record、Replay projection、debug evidenceは同じpublication refを示さなければならない。

## 4. Reconstruction

reconstructionは次を順に実行する。

1. Save format major、record set hash、Package、Contract set、Target compatibilityを検証する。
2. persistent identityのduplicate、missing required relation、template／initializer mismatch、unknown Component schema、unknown persisted fieldをrejectする。
3. Runtime Packageが渡す`RuntimeWorldBuildGatewayV1`とECS construction setをstagingする。
4. entity recordをcanonical orderでtemplateへ展開し、canonical field値とenablementを適用する。
5. relationをpersistent identityでresolveし、derived indexをowner Systemが再構築する。
6. ECS structural transactionが新しいWorld publicationをsealした後、digestをread-backしてexpected valueと比較する。

reconstruction中に旧RuntimeEntityHandleを再利用しない。Source section、World Root、runtime spawnのidentity conflictは明示policyなしにmergeせずrejectする。失敗時はlast-valid Save、Source Project revision、current World publicationを破壊しない。

[LOD](../06-rendering/lod.md#7-simulation-lod境界)のProduction Simulation LODを持つOwnerは、receipt-free `SimulationLodSaveProjectionV1`を`AuthoritativeSaveStateOwnerProjectionRefV1`としてHeaderへ列挙し、root外`AuthoritativeSaveDomainBindingV1`でBundleへ結ぶ。ProjectionのContract ref、subject persistent ref、retained state、queued event、wake condition、handoff generationを各Domain validatorがreconstruction前に検証する。`last_committed_tier_ref`はdiagnostic／handoff検証にだけ使い、Load先のtierを強制しない。View、distance、occlusion、pressure snapshot、resident handleをSaveから復元せず、fullまたは明示last-valid semantic stateをpublishした後にfresh Runtime Contextで再選択する。Projection missing／stale／invalidでfull復元もできない場合はLoad activationを拒否し、tierを推測しない。

[Virtualized／Continuous Geometry](../06-rendering/virtualized-continuous-geometry.md)のpage ID、micro-cluster ID、hierarchy cut、root／page resident set、pool generation、request／feedback queue、View cut history、GPU／native handle、`VirtualGeometryResidencySnapshotV1`、`VirtualGeometryViewCutSummaryV1`はderived Presentation stateであり、Save、authoritative digest、authoritative Replay rootへ含めない。Load／Replayはexact Source／Package／Artifact dependencyを検証した後、current Targetのimmutable Artifactからresidencyをroot-firstで再構築し、fresh Camera／LOD Contextでrepresentationとcutを再選択する。保存時のcut、page availability、pressureを再現または強制しない。

### 4.1 Continue／Runtime Session load

Continueは次の一方向resolutionだけを使う。

1. callerがactive markerから投影したexact `SaveCatalogActiveRootPreconditionRefV1`、`SaveCatalogRefV1`、`slot_id`、`SaveCatalogSlotMembershipRefV1`、Bundle ref、content package setを受け取り、Catalog全体のcontent hash、generation、profile scope、unique Slot membershipと全Field byte equalityを検証する。PersistenceはUI markerを検索しない。
2. Bundle ref／content hash、Catalog Slot／membership、Header、Manifest membership、Project triple、Build、Target、Contract set、Runtime Entry ref／semantic hash、branch closure、`RuntimeEntryPackageRefV1`を検証する。
3. World SaveがpresentならWorld record、inner World Package、persistent identity、digestを検証する。Stageその他のDomain projectionは各Ownerがschema／policy／content hashを検証し、missing projectionを表示metadataから合成しない。
4. 全Owner validation後に`RuntimeSessionLoadActivationPayloadV1`をsealし、[Scheduling／Lifetime](scheduling-lifetime.md)の`RuntimeEntryTransitionRequestV1.trigger_ref`へexact Load Request、activation payload三Fieldへ本payloadのschema／ref／hashを設定する。
5. Runtime Entry transitionがdestination branchをpublishした後にだけ新World／Domain stateをactiveにし、source branchをreverse teardownする。

active-root precondition、Catalog generation／content hash、slot membership、bundle、content package set、entry、package、projectionのいずれかがstale、missing、duplicate、hash不一致ならLoad Requestをrejectし、current Runtime Entry、World／Stage state、Save Catalog、last-valid Saveを変更しない。同じidempotency key＋同じrequest hashは同じ結果を返し、別hashはconflictである。cancelはRuntime Entry transitionのcommit前だけ受理し、commit後に旧branchへ暗黙復帰しない。

本節のRuntime Session Bundle／Load型とContinue resolutionはinitial V1 review definitionである。対応するSave reader／writer、Operation／Service／Provider／MCP inventoryは未materializeであり、Architecture記載から利用可能と解釈しない。表示metadata、World Saveだけ、旧draft shapeからBundleまたはLoad Requestを合成しない。

## 5. Replay projection

```text
RuntimeReplayProjectionV1
  replay_version: 1
  source_world_publication_ref: RuntimeWorldPublicationRefV1
  initial_save_ref: optional RuntimeWorldSaveRecordSetRefV1
  contract_set_ref: ContractSetRefV1
  advance_records[0..1048576]:
    advance_sequence
    accepted_input_refs[]
    accepted_async_result_refs[]
    simulation_lod_domain_projection_refs[]
    structural_delta_batch_hash
    authoritative_digest_ref
    published_or_faulted: published | faulted
  replay_hash: SHA-256

RuntimeReplayProjectionRefV1
  replay_version: positive uint32
  replay_hash: SHA-256
```

`simulation_lod_domain_projection_refs[]`はauthoritative behaviorへ関与するSimulation LOD transitionがそのAdvanceに存在する場合だけ、exact Contract／candidate ref、input context hash、previous／selected tier、transition reason、handoff generationを持つreceipt-free Owner projectionを参照する。Geometry／Material／VFX等のPresentation LOD、virtualized geometryのpage／micro-cluster／cut／residency／feedback、View、pressure snapshotをauthoritative Replay rootへ入れない。各refは§5.1の`RuntimeReplayDomainBindingV1`へexact一件解決し、0件／複数／別rootを拒否する。

Replayはprovider callback順、worker completion時刻、live memory、GPU／Physics native callback pointerを記録しない。async resultはrequest identity、accept advance、validated value projectionを記録し、到着時刻を再現条件にしない。faulted advanceは`published_or_faulted = faulted`を残せるが、World publicationやSave snapshotを生成しない。

Replay recordのcapture・保管・閲覧・redactionはDebug Ownerが所有する。本書はReplayに含めるauthoritative semantic projection、digest、Saveとの参照関係だけを所有する。

### 5.1 Replay transport binding

Debug Ownerがrecordをcapture・store・queryする一方、semantic Replayとcapture recordを結ぶ次の型は本書が所有する。Debug Session identity、store location、debug UI、retention policyのfieldはDebug Ownerを参照し、ここで再定義しない。

```text
RuntimeReplayTransportBindingV1
  binding_version: 1
  replay_projection_ref: RuntimeReplayProjectionRefV1
  debug_session_ref: DebugSessionDescriptorRefV1
  initial_checkpoint_artifact_ref: optional ArtifactRefV1
  input_range_artifact_refs[0..4096]: ArtifactRefV1
  accepted_async_range_artifact_refs[0..4096]: ArtifactRefV1
  rng_range_artifact_refs[0..4096]: ArtifactRefV1
  start_advance_sequence: positive uint64
  end_advance_sequence_inclusive: positive uint64
  required_asset_version_refs[0..65536]
  redaction_manifest_ref
  binding_hash: SHA-256

RuntimeReplayTransportBindingRefV1
  binding_version: positive uint32
  binding_hash: SHA-256

RuntimeReplayDomainProjectionRefV1
  owner_ref: exact {owner_id, owner_revision, owner_content_hash}
  projection_type_ref: McdContractRefV1(kind=type)
  projection_id: StableId
  projection_version: positive uint32
  projection_content_hash: SHA-256

RuntimeReplayDomainBindingV1
  binding_version: 1
  replay_projection_ref: RuntimeReplayProjectionRefV1
  domain_projection_ref: RuntimeReplayDomainProjectionRefV1
  binding_hash: SHA-256

RuntimeReplayDomainBindingRefV1
  binding_version: positive uint32
  binding_hash: SHA-256

RuntimeReplayBundleManifestV1
  manifest_id: StableId
  manifest_version: positive uint32
  replay_projection_ref: RuntimeReplayProjectionRefV1
  replay_transport_binding_ref: RuntimeReplayTransportBindingRefV1
  domain_binding_refs[0..65536]:
    sorted unique RuntimeReplayDomainBindingRefV1
  manifest_content_hash: SHA-256

RuntimeReplayBundleManifestRefV1
  manifest_id: StableId
  manifest_version: positive uint32
  manifest_content_hash: SHA-256

ReplaySliceV1
  slice_id: StableId
  slice_version: positive uint32
  replay_bundle_manifest_ref: RuntimeReplayBundleManifestRefV1
  replay_transport_binding_ref: RuntimeReplayTransportBindingRefV1
  start_runtime_time_ref: RuntimeTimeRefV1(kind=simulation)
  end_runtime_time_ref: RuntimeTimeRefV1(kind=simulation)
  expected_authoritative_digest_ref: RuntimeAuthoritativeStateDigestRefV1
  diagnostic_refs[0..256]
  slice_content_hash: SHA-256
```

range artifactは同じDebug Session、Project／Build／Target、Contract set、advance rangeへ解決し、checkpoint、input、accepted async、RNGのどれが欠けたかをexplicit omissionとして扱う。`ReplaySliceV1`は一つの連続advance rangeだけを参照し、partial capture、Profile推測、Interval欠落、別Session／Project／Buildへのfallbackをportable reproductionとして扱わない。

Save bundleは`AuthoritativeSaveHeaderV1`、domain binding、Save record setを一つのclosureとして結ぶ。Replay bundleは`RuntimeReplayProjectionV1`、transport binding、Domain Replay bindingを同じ方式で結ぶ。いずれもbase projectionへbundle／binding refを埋め戻さず、生成順を`receipt-free Domain Projection → Projection Ref → root Projection／Ref → Binding Ref集合 → Bundle Manifest`とする。Bundleのmissing／extra／別root／別Contract set／別Target／hash差をload、Replay開始、Debug Slice発行前にrejectする。

Save record setの`authoritative_save_header_ref`はBundle ManifestのHeader refと一致し、全Domain Bindingも同じHeader refを持たなければならない。Replay Bundle Manifestのroot Projection refはtransport bindingと全Domain Bindingのroot Projection refに一致し、`ReplaySliceV1`のtransport binding refはBundleのtransport binding refと一致しなければならない。いずれのclosureもlatest検索、外部からのmember補完、hashだけの代用を許可しない。

## 6. Migrationとcompatibility

initial V1のSave／Replay Schemaはmigration chain、old reader、dual schemaまたはhandle aliasを持たない。初回materializationまたは公開後にSave／Replay schema、Component projector、persistent identity policy、digest algorithmを変更する場合だけ、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)のCompatibility ChangeSetを必須とする。`save`／`replay` classを含むConsumer Inventoryがcompleteかつzero verifiedで、scope Requirementのpass fulfillmentがrelease済みSave consumerなしを示した場合に限り、source-preserving fixture再生成とold reader不要を承認できる。

公開済みSaveまたはReplayを読む必要がある場合は、対象format major、old reader期限、new writer開始点、migration failure、release rollback、evidenceを`versioned_reader_migration`として明記する。readerがfield欠落を旧schemaと推測すること、unknown relationをnull objectへ変換すること、old raw handleをpersistent identityへ変換することを禁止する。

## 7. Qualification

initial V1 Persistence／Saveは少なくとも次を証明する。

1. 同じsealed World publicationから同じSave record set、digest、Replay projection hashを二回生成する。
2. stale／duplicate persistent identity、missing required relation、unknown schema／field、hash mismatch、target／contract mismatchをrejectする。
3. Saveにraw handle、chunk row、pointer、native bytes、derived／presentation value、credentialを含めない。
4. reconstructionが新しいRuntimeEntityHandleを発行し、persistent projectionとdigestだけを一致させる。
5. faulted advance、partial structural transaction、partial value writeからSave／digest／published replay snapshotを作らない。
6. Save／Replay bundleのmember setがHeader／root Projection、transport、Domain Bindingへexactに解決し、base projectionへのbinding埋戻しを拒否する。
7. migrationが必要なconsumerをclean breakとして誤分類せず、reader／writer／rollback期限をCompatibility ChangeSetで検証する。
8. initial V1 Save／Replay／Package／ECS／Domain projectionのexact Owner／Definition refと全Evidence Requirementのpass fulfillmentが同じclosureへ解決し、旧Schema、migration chain、alias、dual readerが0件である。
9. world＋optional UI、UI-only、headlessの`RuntimeSessionSaveBundleV1`をround-tripし、UI-only／headlessでWorld Save ref／偽Worldが0件、worldでouter Entry Packageとinner World Packageがexact接続することを検証する。
10. TitleのSave Catalog V1 slotからContinueし、exact Runtime Entry／Package／Stage projection／authoritative digestへ復元する。slot display metadata、timestamp、Level名を変えてもresolutionが不変で、Catalog generation、Bundle hash、Entry hash、Package、Stage policyを一件ずつstaleにしたcaseはsource branch不変でrejectする。

## 8. 非目的

本書はSave implementation、storage backend、cloud sync、debug viewer、Task Plan、migration実行を指示しない。実施はinitial V1 Definition Closure、Qualification要求とProduct Work Package条件の後に開始できる。
