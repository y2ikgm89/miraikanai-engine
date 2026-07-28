# Miraikanai Engine Persistence and Save Contract

- 文書ID: mirakan.arch.persistence-save
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Runtime Session Save Bundle、Continue／load resolution、Runtime World Save record、persistent／ephemeral Entity projection、Component lifecycle・enablement projection、authoritative state digest、reconstruction、Replay projection、Save migration・qualification
- 非正本範囲: ECS storage layout・query・lease、Package binary、generic artifact catalog、debug capture transport、runtime phase／job DAG、Domain field意味、AI authorization。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Runtime ECS](entity-component-system.md)、[Runtime Package](runtime-package.md)、[Scheduling／Lifetime](scheduling-lifetime.md)
- 関連文書: [Architecture Governance](../01-governance/architecture-governance.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Project state](../03-authoring/project-state.md)、[Runtime ECS](entity-component-system.md)、[Runtime Package](runtime-package.md)、[Scheduling／Lifetime](scheduling-lifetime.md)、[Debugging／Observability／Replay](debugging-observability-replay.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[LOD](../06-rendering/lod.md)、[Virtualized／Continuous Geometry](../06-rendering/virtualized-continuous-geometry.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-24

## 1. 状態と結論

Saveのtop-level rootは`RuntimeSessionSaveBundleV1`であり、exact Runtime Entry／branch package、Authoritative Header／Domain projection集合、optional World Saveを一つのimmutable closureへ束縛する。World Saveはlive ECS memoryのdumpではない。seal済みWorld publicationから、Persistent Entity Identity、canonical Component field projection、lifecycle、enablement、sequence、schema／contract provenanceを持つimmutable record setを生成する。reconstructionはPackage／ECS construction contractを再検証して新しいRuntime Entity handleを作るため、Save内のhandle、chunk location、pointer、worker順を再利用しない。[Memory／Pointers](../02-foundation/memory-pointers.md)のbinding上、live pointer、reference、lease、span、writer、allocator objectはSave／Replay projectionに不許可であり、本書はその禁止をpersistent identityとreconstruction fixtureへ具体化する。

ReplayはSaveと同じpayloadではない。Replayはauthoritative input、accepted async result、structural boundary、digest、advance sequenceを再現可能なprojectionとして記録し、capture transportとdebug UIは[Debugging／Observability／Replay](debugging-observability-replay.md)が所有する。

本書はtarget review Contractであり、Save reader／writer、Replay reader／writer、migrationのcurrent activationを意味しない。current化にはcomplete／zero-verified Consumer Inventory、approved Compatibility Change、Owner reference migration manifest、source／target Foundation Definition Closure、Definition Migration binding、全Evidence Requirementのpass satisfaction binding、qualification evidenceが同一closureで必要である。

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
```

`authoring_entity`はAuthoring stable identityから解決する。`world_root_runtime`と`runtime_spawn`はRuntime Ownerが明示的に発行するpersistent identityであり、Cook時に将来spawnを予約してはならない。raw `RuntimeEntityHandle`、slot index、generation、chunk ID、rowはpersistent identityではない。

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
  migration_chain_refs[0..64]
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

RuntimeSessionLoadRequestV1
  request_id: StableId
  idempotency_key: StableId
  play_session_id: StableId
  source_branch_generation: positive uint64
  profile_id: StableId
  save_catalog_id: StableId
  save_catalog_schema_version: uint32 = 2
  expected_save_catalog_generation: positive uint64
  slot_id: StableId
  runtime_session_save_bundle_ref: RuntimeSessionSaveBundleRefV1
  requested_apply_advance_sequence: uint64
  precondition_snapshot_hash: SHA-256
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

`RuntimeSessionSaveBundleV1`はBundle Manifest完成後にだけ生成し、自己hashを除く全FieldをASCII `MIRAKAN_RUNTIME_SESSION_SAVE_BUNDLE_V1`とMCD canonical length framingして`bundle_content_hash`を計算する。Load Requestとactivation payloadもそれぞれASCII `MIRAKAN_RUNTIME_SESSION_LOAD_REQUEST_V1`、`MIRAKAN_RUNTIME_SESSION_LOAD_ACTIVATION_PAYLOAD_V1`と自身のhash Fieldを除くMCD canonical bytesをlength framingしてhashする。`runtime_entry_ref`／semantic hash／branch closure、外側`RuntimeEntryPackageRefV1`、HeaderのProject triple／Target／Contract set／Cadence、Bundle ManifestのHeader refはbyte equalityでなければならない。`entry_kind=world`だけが`world_save_record_set_ref`を0または1件持て、presentの場合はWorld Save内側`RuntimePackageRefV1`が外側Entry Packageの`world_package_ref`へexact解決する。`ui | headless`はWorld Save refをnullとし、UI／headlessのState owner projectionはBundle Manifestから保存する。UI-only／headless Saveのために偽World publicationまたは空World recordを生成しない。

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

1. Save format major、record set hash、Package、Contract set、Target compatibility、migration chainを検証する。
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

1. UI Profile Ownerのactive `SettingsCatalogCommitMarkerV1`からexact target `SaveCatalogV2` generationを解決し、`slot_id`に一致する一件の`runtime_session_save_bundle_ref`を読む。current V1はBundle refを持たないためContinue requestを生成しない。
2. Bundle ref／content hash、Header、Manifest membership、Project triple、Build、Target、Contract set、Runtime Entry ref／semantic hash、branch closure、`RuntimeEntryPackageRefV1`を検証する。
3. World SaveがpresentならWorld record、inner World Package、persistent identity、digest、migrationを検証する。Stageその他のDomain projectionは各Ownerがschema／policy／content hashを検証し、missing projectionを表示metadataから合成しない。
4. 全Owner validation後に`RuntimeSessionLoadActivationPayloadV1`をsealし、[Scheduling／Lifetime](scheduling-lifetime.md)の`RuntimeEntryTransitionRequestV1.trigger_ref`へexact Load Request、activation payload三Fieldへ本payloadのschema／ref／hashを設定する。
5. Runtime Entry transitionがdestination branchをpublishした後にだけ新World／Domain stateをactiveにし、source branchをreverse teardownする。

Catalog generation／slot／bundle／entry／package／projection／migrationのいずれかがstale、missing、duplicate、hash不一致ならLoad Requestをrejectし、current Runtime Entry、World／Stage state、Save Catalog、last-valid Saveを変更しない。同じidempotency key＋同じrequest hashは同じ結果を返し、別hashはconflictである。cancelはRuntime Entry transitionのcommit前だけ受理し、commit後に旧branchへ暗黙復帰しない。

本節のRuntime Session Bundle／Load型とContinue resolutionはtarget review definitionであり、UI ownerのSave Catalog V2、Runtime Entry Package／transition、Project Presentation Binding、Domain projection migrationと同じatomic Definition Migration前はcurrent Save reader／writer、Operation／Service／Provider／MCP inventoryへ追加しない。V1 Catalog、World Saveだけ、表示metadataからtarget BundleまたはLoad Requestを合成しない。

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

Save／Replay schema、Component projector、persistent identity policy、digest algorithmを変更する場合は[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)のCompatibility ChangeSetを必須とする。`save`／`replay` classを含むConsumer Inventoryがcompleteかつzero verifiedで、scope Requirementのpass fulfillmentがrelease済みSave consumerなしを示したtarget ECS clean breakだけが、`source_preserving_recook`の範囲でfixtureを再生成し、old Save／Replay reader、dual schema、handle aliasを残さない。

公開済みSaveまたはReplayを読む必要がある場合は、対象format major、old reader期限、new writer開始点、migration failure、release rollback、evidenceを`versioned_reader_migration`として明記する。readerがfield欠落を旧schemaと推測すること、unknown relationをnull objectへ変換すること、old raw handleをpersistent identityへ変換することを禁止する。

## 7. Qualification

target Persistence／Saveは少なくとも次を証明する。

1. 同じsealed World publicationから同じSave record set、digest、Replay projection hashを二回生成する。
2. stale／duplicate persistent identity、missing required relation、unknown schema／field、hash mismatch、target／contract mismatchをrejectする。
3. Saveにraw handle、chunk row、pointer、native bytes、derived／presentation value、credentialを含めない。
4. reconstructionが新しいRuntimeEntityHandleを発行し、persistent projectionとdigestだけを一致させる。
5. faulted advance、partial structural transaction、partial value writeからSave／digest／published replay snapshotを作らない。
6. Save／Replay bundleのmember setがHeader／root Projection、transport、Domain Bindingへexactに解決し、base projectionへのbinding埋戻しを拒否する。
7. migrationが必要なconsumerをclean breakとして誤分類せず、reader／writer／rollback期限をCompatibility ChangeSetで検証する。
8. Consumer InventoryのSave／Replay scopeと全Evidence Requirementのpass fulfillment、Compatibility Change、Owner reference migration manifest、source／target Definition Closure、Definition Migration bindingが同じclosureへexact解決する。
9. world＋optional UI、UI-only、headlessの`RuntimeSessionSaveBundleV1`をround-tripし、UI-only／headlessでWorld Save ref／偽Worldが0件、worldでouter Entry Packageとinner World Packageがexact接続することを検証する。
10. Titleのtarget V2 Save slotからContinueし、exact Runtime Entry／Package／Stage projection／authoritative digestへ復元する。V1ではContinue unavailable、V2ではslot display metadata、timestamp、Level名を変えてもresolutionが不変で、Catalog generation、Bundle hash、Entry hash、Package、Stage policyを一件ずつstaleにしたcaseはsource branch不変でrejectする。

## 8. 非目的

本書はSave implementation、storage backend、cloud sync、debug viewer、Task Plan、migration実行を指示しない。実施は対象Ownerの承認済みdefinition migrationとProduct Work Package条件の後に開始できる。
