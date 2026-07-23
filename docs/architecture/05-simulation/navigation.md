# Miraikanai Engine Navigation Contract

- 文書ID: mirakan.arch.simulation-navigation
- 状態: review
- 正本範囲: 2D Grid Navigation、3D Navmesh source／profile／artifact、Navmesh query request／result／status、Navmesh version／lease、Path Following／Movement Intent contract、`MotionExecutorPortV1`
- 非正本範囲: Runtime phase／tick／shared worker／capacity、Physics dynamics、Collision event、selected Motion ExecutorによるTransform解決、Animation、World streaming、external dependency version／build pin、AI authorization。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Collision](collision.md)、[Physics](physics.md)、[World](../06-rendering/world.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論とPlatform境界

Navigationは2D Gridと3D NavmeshのEngine-owned profile、artifact、query、version、leaseを公開し、build／query Backendをprivate Adapterへ隔離する。Project C++、GameplayDefinition、AI、Editor、SaveへVendor config、mesh／query object、polygon reference、status bits、allocator、callback、binary formatを公開しない。dependencyのexact version／commit／license／build optionは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)だけが所有する。

NavigationはWorld Transformを書かず、Physicsをstepせず、Animation poseを選ばない。[Collision](collision.md)のstatic geometry／filterをsourceとしてcookし、[Physics](physics.md)の前snapshotからdynamic obstacle inputを受ける。cross-subsystem order、async acceptance、shared worker、lifetimeは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)を消費する。

ModuleはContracts、Core、Grid2D、private Navmesh Backend、Authoring、Editor Projection、Qualification Toolへ分離する。ContractsはEngine value／Port／request／result／statusだけ、Coreはvalidation／build orchestration／version／lease／status normalizationだけを公開する。Backend名で上位処理を分岐せず、Backend capabilityはprivate conformance inputとして扱う。

## 2. Profile、source、2D Grid

| Object | 意味 | persistence |
|---|---|---|
| `GridNav2DProfileV1` | cell semantics、neighbor policy、clearance class、cost policy、`max_path_cells` | Project source |
| `NavAgentProfileV1` | radius、height、climb、slope、query filter ref | Project source |
| `NavAreaProfileV1` | stable area ID、traversability、relative cost、tags | Project source |
| `NavBuildProfile3DV1` | voxel／tile／region／contour／polygon build semantics、`max_straight_path_points` | Project source |
| `NavSourceSetV1` | canonical static geometry、area、modifier、link source revisions | Project source |

Profile fieldはSI単位とfinite値を使い、agent寸法、cell／voxel寸法、slope、climb、tile relationをcross-field validationする。Vendor default objectをserializeせず、Engine Profileの全意味を保存する。Backendが表現できないProfileはcook failureにし、近い値へsilent adjustmentしない。

### 2.1 Grid2D

Grid2Dは3D Navmesh Backendへ依存しない。Source boundsをcellへrasterizeし、static blocked、area cost、clearance class、optional directed linkをCooked Grid Artifactへ格納する。neighbor policyはclosed enum、diagonal traversalではcorner cutting policyを明示する。cell coordinateとworld coordinateの変換はartifact origin、cell extent、versionを必須とし、floating positionをidentityとして使わない。

Path queryはprojected start／end、agent clearance、area filter、hard result boundを検証し、canonical A* tie-breakを使う。同scoreではcell coordinate、direction ID、insertion-independent parent keyで決め、heap allocation orderを使わない。Gridのpartial rebuildはstaging artifactを完成させてからversion単位でactivateし、live cell arrayをquery中にmutateしない。

`GridNav2DArtifactV1`はsource/profile identity、origin、extent、dimensions、cell payload、link table、diagnostic summaryを持つimmutable Derived Assetである。Renderer tile mapやPhysics broadphaseをlive backing storeとして参照しない。

## 3. 3D Navmesh cookとartifact

3D cookは次のartifact pipelineをEngine semanticsとして所有する。

1. [Collision](collision.md)のcompatible static geometry revisionをcanonical triangle streamへ変換する。
2. area、modifier、exclusion、off-mesh linkをStable ID順で投影する。
3. validated Agent／Build Profileとsource identityからstaging tilesを生成する。
4. tile seam、connectivity、area、link endpoint、hard boundを検査する。
5. manifest、diagnostic、source mappingを付けたimmutable artifactをpublish候補にする。

`CookedNavWorldV1`はartifact ID、schema ref、source set／Profile identity、coordinate frame、tile directory、area table、link table、Backend-private payload refs、build diagnostic summaryを持つ。Backend-private tile bytesはEngine query contractからopaqueであり、Save、Project C++、AIへ公開しない。

Tile promotionはartifact単位でatomicに行う。部分成功をactive Worldへ混在させない。Source geometry、Profile、area table、Backend lockのいずれかが変われば新しいartifact identityを要求する。Reimport／cook／promotion／rollbackは[Asset lifecycle](../03-authoring/asset-lifecycle.md)を消費する。

`NavModifierV1`はStable ID、shape ref、area override／exclusion、applicabilityを持つ。`NavOffMeshLinkV1`はStable ID、typed endpoints、direction、agent／area filter、traversal tagを持つ。linkのgameplay実行はGameplay ownerの責務で、Navigationはpath candidateとlink metadataだけを返す。

Dynamic obstacleは前snapshotからbounded update inputを受ける。C1の動的変化（door閉鎖等のNavModifier／area／off-mesh link変更を含む）は、差分re-cookによる新versionのstaging artifactとversion切替だけで反映する。差分buildはsource／Profile identityが変わらないtileのpayload再利用を許すが、artifact identityは新規に発行し、Tile promotionのatomic ruleに従う。Engine-owned local avoidance overlayはC2の複数Agent local avoidance専用であり、C1経路では使わない。obstacle input受領からversion activateまでの反映latency boundは[Runtime performance／capacity](../04-runtime/performance-capacity.md)が所有し、本書は値を定義しない。同じRuntime slotのPhysics native Worldを直接queryせず、live Navmeshをcallbackからmutateしない。World streaming固有のcell policyは[World](../06-rendering/world.md)へ委譲し、本書はstreaming phaseや共通capacityを定義しない。

## 4. Query、status、version、lease

`NavQueryRequestV1`はrequest ID、Nav World handle／expected version、query kind、start／end、Agent Profile ref、area filter、hard result bound、deadline／producer metadataを持つ。query kindはpath、nearest navigable point、random reachable point、ray／visibility on nav domain、distance-to-boundaryのclosed setである。

`NavQueryResultV1`はrequest ID、observed `NavMeshVersion`、normalized status、projected endpoints、ordered corridor／points、area／link metadata、diagnosticを持つ。公開結果へnative polygon ref、tile pointer、raw status bitsを含めない。Resultはobserved versionと同じlease中だけ解釈でき、別versionへpolygon identityを再利用しない。

`NavQueryStatusV1`は`success | no_path | invalid_request | version_mismatch | result_bound_exceeded | unavailable | cancelled | backend_failure`のclosed enumである。Result bound超過はpartial successにせず、`no_path`とBackend failureを区別する。Pathのcanonical tie-breakはtotal cost、projected endpoint distance、Engine polygon／cell key、point lexicographicを使い、native traversal順を使わない。

`NavWorldHandle`と`NavQueryHandle`はEngine generation handle、`NavMeshVersion`はactive artifact generationのmonotonic runtime identityであり、ProjectのStable IDやartifact content identityを代用しない。`NavWorldLeaseV1`はWorld handle、exact version、immutable Backend world ref、expiry boundaryを束ねる。lease expiry後のresult解釈、version activation中のlive pointer保持、old polygon refのnew version利用を禁止する。

Async resultのdeadlineとacceptanceは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)のcanonical `T20_AsyncIntegrate`を参照する。Navigationはphase順序を再定義せず、accepted resultだけを次のGameplay evaluationから利用できる。Replayへはrequest、accepted result、version／artifact identityを供給し、記録はRuntime ownerのcanonical `T100_ReplayCheckpoint`に接続する。

### 4.1 C1 Path Following／Movement Intent

`capability.gameplay.path_following`（成熟度C1。maturityはidentityに含めない）は2D／3D共通のgoal、path generation、waypoint進行、replan、stuck判定をNavigation ownerとして所有する。Navigation query resultとselected Motion Executorの最終authoritative motion解決の間を結び、Path FollowingはWorld Transform、Physics body、Nav payloadを直接writeしない。selected executorのwriter authorityを奪わない。

```text
PathFollowRequestV1
  request_id
  actor_ref
  actor_generation
  goal: WorldPosition2f | WorldPosition3f | StableAnchorRef
  nav_agent_profile_ref
  executor_capability_ref
  movement_profile_ref
  arrival_radius_m
  replan_policy_ref
  requested_tick

PathFollowerStateV1
  actor_ref
  request_id
  nav_generation
  path_result_generation
  waypoint_index
  status: awaiting_path | following | arrived | blocked | stuck | stale | cancelled
  last_progress_tick
  replan_count
  generation

MovementIntentV1
  actor_ref
  actor_generation
  source_request_id
  desired_velocity
  facing_intent
  valid_for_tick
  movement_profile_ref
```

`MotionExecutorPortV1`はNavigationが唯一のcanonical ownerとして次の7 Fieldを固定する。全MCD参照は`McdContractRefV1 {id, version, contract_set_hash}`であり、IDは`<kind>.<lower-token-path>`、versionは別Fieldである。

```text
MotionExecutorPortV1
  executor_capability_ref: McdContractRefV1(kind=capability)
  movement_profile_schema_ref: McdContractRefV1(kind=type)
  accepted_intent_schema_refs[1..64]: McdContractRefV1(kind=type)
  transport_message_schema_ref: McdContractRefV1(
    id=type.navigation.motion_executor_intent_batch, version=1, contract_set_hash)
  resolved_motion_schema_ref: McdContractRefV1(kind=type)
  compatibility_predicate_ref: McdContractRefV1(kind=policy)
  failure_diagnostic_refs[1..32]: DiagnosticCodeRefV1
```

`MovementIntentV1`の正規MCD参照は`{id=type.navigation.movement_intent, version=1, contract_set_hash}`、generic contribution envelopeは`{id=type.navigation.motion_intent_contribution, version=1, contract_set_hash}`である。Animation等のproducer固有proposal schemaはCoreのaccepted typeへhard-codeせず、次のregistryにowner付きで登録する。`mirakan.schema.*`、`mirakan.contract.*`、PascalCaseを含むID、ID文字列内の`@1`はcurrent Contract setで無効である。

```text
MotionIntentContributionV1
  contribution_id: StableId
  motion_subject_ref
  motion_subject_generation
  valid_for_tick
  producer_system_ref: GameSystemContractRefV1
  proposal_schema_ref: McdContractRefV1(kind=type)
  proposal_payload:
    kind: referenced
      proposal_ref: TypedValueRefV1
    | kind: inline
      proposal_value: BoundedTypedValueV1
  proposal_hash: SHA-256

MotionIntentContributionBindingRecordV1
  owner_ref: exact {owner_id, owner_revision, owner_content_hash}
  proposal_schema_ref: McdContractRefV1(kind=type)
  adapter_policy_ref: McdContractRefV1(kind=policy)
  compatible_executor_capability_refs[1..64]
  fixture_refs[1..64]
  record_content_hash

MotionIntentContributionBindingRegistryRefV1
  registry_id
  registry_version
  registry_content_hash

MotionIntentContributionBindingRecordRefV1
  registry_ref: MotionIntentContributionBindingRegistryRefV1
  proposal_schema_ref: McdContractRefV1(kind=type)
  owner_ref: exact {owner_id, owner_revision, owner_content_hash}
  record_content_hash

MotionIntentContributionBindingRegistryV1
  registry_id: motion_intent.contribution_binding.registry.active
  registry_version
  registry_content_hash
  records[0..4096]: MotionIntentContributionBindingRecordV1

MotionIntentBindingSelectionDocumentV1
  common Project Document header
  selection_id: same StableId as Document header
  selected_executor_provider_ref: MotionExecutorProviderRecordRefV1
  selected_binding_record_refs[0..64]:
    MotionIntentContributionBindingRecordRefV1
  selection_content_hash: SHA-256
```

Coreはこのgeneric schema、registry、resolverだけを所有する。`proposal_payload`はdiscriminator外Fieldをcanonical omissionし、reference branchはexact schema／payload hash、inline branchはbound／canonical value hashを検証する。`proposal_hash`はASCII `MIRAKAN_MOTION_INTENT_CONTRIBUTION_V1`と自身だけを除く全FieldのMCD canonical bytesを`uint32_be` length framingしてSHA-256する。

recordsは`proposal_schema_ref.id`、version、owner refのcanonical byte順へstrict sortし、duplicate、同じschemaへの複数active adapter、stale owner／policy／fixture hash、Core ownerを偽装するextension recordを拒否する。record hashはASCII `MIRAKAN_MOTION_INTENT_CONTRIBUTION_BINDING_RECORD_V1`とself-excluding canonical bytes、Registry hashはASCII `MIRAKAN_MOTION_INTENT_CONTRIBUTION_BINDING_REGISTRY_V1`、Registry ID／version、record count、全record canonical bytesを各`uint32_be` length framingしてSHA-256し、自己Fieldを除外する。`MotionIntentContributionBindingRegistryRefV1`は三Fieldすべてを同一active Registryへexact解決し、ID-only／latest fallbackを許可しない。`MotionIntentContributionBindingRecordRefV1`はRegistry三Field、schema ref、owner三Field、record hashを一つにbindし、current Registryからlatest recordを引き直さない。

個別Feature／Genreのproposal schema、adapter policy、binding、positive fixtureは当該Pack ownerが登録し、Core binaryまたはCore fixture inventoryへコピーしない。resolverはcurrent Registry ref、producer schema、selected executor capabilityを入力にexact一件を選び、0件または複数なら推測変換しない。Project Sourceの正本は`MotionIntentBindingSelectionDocumentV1`であり、Registry、resolver output、Runtime lookup tableは派生物である。`operation.navigation.motion_intent_binding.select@1`だけがexpected Project revision、Selection Document ref／before hash、Provider RecordRef、RecordRef集合、Preview／Validation、`MutationAuthorizationBindingV2`のR2 ApprovalまたはPredelegationを受け、Prepared Candidate経由でSourceを変更する。binding欠落、expired、Scope／request hash不一致はexact `diagnostic.approval.required / MIRAKAN-APPROVAL-REQUIRED`で拒否する。Registry reloadはSource selectionから決定的に再materializeし、Registryを直接writeするOperationを公開しない。

Compile closureはSelection Document ref／content hash、RegistryRef、全Binding RecordRef、Provider RecordRef、set hashを持つ。Saveは同じselection ref／hashとRecordRef集合、Replay headerはそれらに加えて各tickで実際に使ったRecordRef／proposal hashを記録する。Load／Replay／CompileはSource→Registry→Record→Providerの全ref equalityを再検査し、Registry version／hash、record hash、selection hashがstaleなproposalを再利用しない。`fixture.navigation.motion-intent-binding-roundtrip`はselect→Commit Marker→reload→Compile→Save／Load→Replayを通し、record missing／duplicate／stale Registry／cross-owner／Sourceとderived Registry差を各単独原因でrejectする。

Port transportとgeneric batch publisherをNavigation／Core ownerへ固定する。`game_system.engine.navigation.motion_intent_batch_publisher`はactive Selection DocumentとContribution Registryを読み、全owner contributionをcanonical mergeして、T40でselected Providerへ一回だけ配送する。Character、RTS、board token、Animation等のcontributorは自身のproposalとadapter recordだけを所有し、このpublisher、batch identity、merge orderを所有またはforkしない。これはEventではなくbounded Port messageである。

```text
MotionExecutorIntentBatchV1
  batch_id: StableId
  actor_ref
  actor_generation
  valid_for_tick
  provider_record_ref: MotionExecutorProviderRecordRefV1
  entries[1..16]:
    intent_schema_ref: McdContractRefV1(kind=type)
    payload_kind: referenced | inline_value
    payload_ref: TypedValueRefV1 | null
    payload_value: BoundedTypedValueV1 | null
    payload_value_identity: TypedValueIdentityV1 | null
    payload_hash: SHA-256
    producer_system_ref: GameSystemContractRefV1
    proposal_id: StableId

TypedValueRefV1
  value_id: StableId
  value_revision: uint64
  value_schema_ref: McdContractRefV1(kind=type)
  content_hash: SHA-256

TypedValueIdentityV1
  value_id: StableId
  value_schema_ref: McdContractRefV1(kind=type)
  canonical_value_hash: SHA-256

BoundedTypedValueV1
  value_schema_ref: McdContractRefV1(kind=type)
  canonical_value_bytes: bytes_base64url[1..65536]
  canonical_value_hash: SHA-256
```

`payload_kind=referenced`は`payload_ref`だけをnon-nullにし、value／value identityをnullにする。`inline_value`は`payload_value`と`payload_value_identity`をnon-null、refをnullにする。inline canonical bytesは`intent_schema_ref`でdecode／再encodeしてbyte一致し、value／identity／entryの三hashが一致しなければならない。`payload_ref.value_schema_ref`またはinline value／identityのschema refはentryの`intent_schema_ref`とexact equalityである。entriesは`intent_schema_ref.id`、version、proposal ID、producer System refのcanonical byte順でstrict sortし、duplicate proposal ID、同一payload identityの重複、type spoof、payload hash不一致をrejectする。

selected compositionが生成し得るproposal schema集合は、batch envelopeではなく`entries[].intent_schema_ref`の集合だけを対象に、selected Providerの`accepted_intent_schema_refs[]`のsubsetでなければならない。Cook、Runtime Activation、各batch受理の三箇所でset inclusionを検査する。root motionを生成しないcompositionはroot-motion schemaを集合へ加えず、Animation `mode=animation`を選ぶcompositionだけがexact schemaを要求する。`compatibility_predicate_ref`は各intent schema、`movement_profile_ref`が`movement_profile_schema_ref`へ適合すること、Target Profile、dimension、required Capabilityを検証する。missing provider、同じCapabilityへのactive provider複数、intent不受理、profile schema不一致、Target不適合はtyped failureであり、別ProviderまたはProfileへ推測fallbackしない。

### 4.2 Motion Executor Provider Catalog

NavigationはProvider-neutralな次のCatalog schemaを所有する。Physics、Pack、Project、fixtureは同じCatalogへrecordを登録し、別表やhard-coded whitelistを作らない。

```text
MotionExecutorProviderCatalogV1
  catalog_id: motion_executor.provider_catalog.active
  catalog_version
  catalog_hash
  contract_set_hash
  records[1..1024]: MotionExecutorProviderRecordV1

MotionExecutorProviderCatalogRefV1
  catalog_id
  catalog_version
  catalog_hash
  contract_set_hash

MotionExecutorProviderRecordRefV1
  catalog_ref: MotionExecutorProviderCatalogRefV1
  provider_id
  provider_version: uint32
  provider_content_hash: SHA-256

MotionExecutorProviderRecordV1
  provider_id
  provider_version: uint32
  provider_content_hash: SHA-256
  owner_identity: ProviderOwnerIdentityV1
  usage: production | fixture_only
  port_descriptor: MotionExecutorPortV1
  implementation_system_ref: GameSystemContractRefV1
  supported_target_profile_refs[1..64]
  qualification_receipt_refs[1..64]

ProviderOwnerIdentityV1
  owner_kind: engine | project | fixture
  engine_component_ref: exact ref/hash | null
  project_ref: exact Project ref/revision/hash | null
  fixture_owner_ref: exact fixture ref/hash | null
```

owner unionはdiscriminator外Fieldをnullにする。`usage=production`は`engine | project`だけ、`fixture_only`は`fixture`だけを許可する。recordは`provider_content_hash`を除く全FieldのMCD canonical bytesからhashし、CatalogはASCII `MIRAKAN_MOTION_EXECUTOR_PROVIDER_CATALOG_V1`、catalog ID／version、Contract set hash、provider ID／version順でstrict sortしたrecord全体を入力して自己hashを除外する。`MotionExecutorProviderRecordRefV1`はCatalog identity／version／hash／Contract setとrecord identity／version／content hashを一つにbindし、current Catalogからlatest recordを再解決しない。Batch、Selection State、Save、ReplayはこのRecordRefだけを保存し、`provider_binding_ref`＋別hash、Capability ID、表示名で代用しない。Project ownerは現在compile対象Projectとexact equalityでなければならず、cross-project refやpayload自身が主張するProject IDだけでは所有を証明しない。fixture recordをProduction、Project Save、Runtime Packageへ選択しない。

`arrival_radius_m`はfiniteな0.01～10 mとする。`executor_capability_ref`は選択Provider Capability、`movement_profile_ref`はそのProvider-owned schemaのinstanceを参照する。request validationは`nav_agent_profile_ref`のslope／climb／clearance semanticsとProviderのcompatibility predicateを検査し、Navigation上はtraversableだがexecutorが通行できない組合せをinvalid Profile relationとして拒否する。C1 waypointは`GridNav2DProfileV1`の`max_path_cells`（C1上限8,192 cell）または`NavBuildProfile3DV1`の`max_straight_path_points`（C1上限256 point）に従い、上限値の正本はProfile fieldである。path resultはNav generation、actor generation、request IDがすべて一致した場合だけ統合し、不一致は`stale`としてtypedに扱い、異なるrequestへ推測で転用しない。goal移動、Nav generation変更、path corridor逸脱、`PathFollowerStateV1.status`の`blocked`遷移だけがreplan契機である。`blocked`はProvider stateではなくPath Follower述語であり、Portの`resolved_motion_schema_ref`に適合する結果から得た実進捗だけが、`MovementIntentV1`の要求displacementに対して`replan_policy_ref`の進捗閾値未満であるtickが、同policyのblocked判定tick数だけ連続した場合に遷移する。Physics snapshot、Transform component、Animation pose、Provider-private stateから進捗を迂回判定しない。`replan_policy_ref`は進捗閾値、blocked判定tick数、最短replan間隔、最大replan回数、stuck tick上限を必須とする。上限到達時は`stuck`へ遷移してboundedに停止し、毎tick無制限queryを発行しない。

accepted Navigation resultを`T20_AsyncIntegrate`で統合し、Path Followingは`T30`でactor当たり一つだけの`MovementIntentV1`を生成し、selected Motion Executorが`T40`で解決する。`MovementIntentV1`はproposalであり、actual displacement、grounding、slide、collision responseを所有しない。C1 reference recipeはPhysics Kinematic Motion Providerを選択できるが、board-token／RTS executor stubはPhysicsなしで同じPortへ接続できる。複数Agentのlocal avoidance、flow field、shared corridor optimizationはC2に留める。

ownerが永続化を要求した場合だけSaveへdestination intent、request semantic、`executor_capability_ref`、movement profile identity、replan countを保存する。Scope instance keyとSave identityをruntime generationへ置換しない。waypoint index、Nav path point配列、native query handle、Physics stateは保存しない。Loadは保存済みNav generationやpath進捗を信用せず、同じdestination intentとProfileから再queryする。ReplayはRequest、採用path result hash、Movement Intent、selected executor／profile hash、accepted resolved motion、arrived／stuck／replan Eventを記録する。C1 fixtureは2D moving-anchor follow／interaction-range arrival、3D Navmesh追跡、Physicsなしboard-token、PhysicsなしRTS unitを用い、destination移動、passage閉鎖、stale result、owner scope deactivate、blocked recovery、Save／Load、Replay一致を検証する。

## 5. Authoring、AI、diagnostic、recovery

公開OperationはWorld／Profile／source／artifactのinspect、Grid／Navmesh Profile作成・更新、area／modifier／link作成・更新・削除、cook、query preview、validation、qualification reportに限定する。全writeは[Project state](../03-authoring/project-state.md)のChangeSetを生成し、live Backendへ直接writeしない。EditorとAIは同じSource Document、validator、preview、cook、undo／redoを使う。

AIは「どのBackendを使うか」ではなく、2D／3D、移動体の大きさ、登れる段差／傾斜、通行可能area、door／jump link、dynamic obstacle、Targetをゲーム上の言葉で確認する。未指定値はassumptionとしてpreviewに表示する。Risk分類、authorization、commit可否は[AI Security／Approval](../01-governance/ai-security-approval.md)を参照し、Approval ruleを再掲しない。

主要diagnostic classはinvalid Profile relation、source geometry incompatible、tile／grid connectivity failure、area／link reference missing、cook unavailable、artifact incompatible、version mismatch、expired lease、result bound exceeded、Backend invariant violation、motion executor missing／duplicate／incompatible、stale resolved motion、provider failureである。Motion Executor共通codeは`MIRAKAN-NAV-MOTION-EXECUTOR-INCOMPATIBLE`と`MIRAKAN-NAV-MOTION-EXECUTOR-STALE_RESULT`へ固定する。前者はintent subset／Profile／Target／dimension不一致、後者はactor／request／provider generation不一致を表す。各diagnosticはstable code、source path／Stable ID、原因、remediation候補、active artifact／last-valid resolved motionを維持したかを返す。

Cook失敗、Worker crash、invalid staging、incompatible promotionではlast valid artifactを維持する。Query Backend faultでは当該requestをfailureにし、active artifactを破壊しない。Runtime fault／publish policy、shared queue／capacity／backpressureはRuntime ownersへ委譲する。

## 6. Qualificationと採用しないもの

QualificationはGrid rasterization／A*、Profile cross-field validation、canonical triangle conversion、tile seam、area／modifier／off-mesh link、nearest projection、path ordering、no-path、result bound、async stale result、version activation／lease expiry、fault injection、Editor／AI previewを含む。同じfixtureを全private Backendへ与え、Engine result／status／diagnosticが一致することを検査する。

`fixture.navigation.motion-executor.physics-kinematic`はPhysics reference Providerを検証する。Physics unavailable用の二つのfixture-only Provider recordは`MotionExecutorProviderCatalogV1`へ次の値で登録し、Production選択またはProject Saveへ昇格しない。表中の全MCD IDは`version=1`とfixture Contract set hashを持つ`McdContractRefV1`、Provider／owner／fixtureはversion／content hash付きexact refである。

| Provider ID／usage／owner | `executor_capability_ref.id` | `movement_profile_schema_ref.id` | `accepted_intent_schema_refs[].id` | transport／resolved type ID | compatibility policy ID |
|---|---|---|---|---|---|
| `provider.fixture.motion_executor.board_token` v1／fixture_only／exact `fixture.navigation.motion-executor.board-token-no-physics` owner | `capability.motion_executor.fixture.board_token` | `type.fixture.motion.board_token_movement_profile` | `[type.navigation.movement_intent]` | `type.navigation.motion_executor_intent_batch`／`type.fixture.motion.board_token_resolved_motion` | `policy.fixture.motion.board_token_target_dimension` |
| `provider.fixture.motion_executor.rts_stub` v1／fixture_only／exact `fixture.navigation.motion-executor.rts-stub-no-physics` owner | `capability.motion_executor.fixture.rts_stub` | `type.fixture.motion.rts_stub_movement_profile` | `[type.navigation.movement_intent]` | `type.navigation.motion_executor_intent_batch`／`type.fixture.motion.rts_stub_resolved_motion` | `policy.fixture.motion.rts_stub_target_dimension` |

両policyはfixture Target allowlist、2D／3D dimension、profile schema、intent subsetを全件検証する。`fixture.navigation.motion-executor.board-token-no-physics`と`fixture.navigation.motion-executor.rts-stub-no-physics`は上記7 Field、Provider identity／version／content hash／owner／usage、Target／dimension policy、diagnostic、Physics dependency 0件を検証する。Physics production recordも同じCatalog schemaへ`usage=production`、Engine owner、exact implementation System／Qualification Receiptで登録する。

negative fixtureはmissing／duplicate Provider、Provider record order、composition intent subset不成立、profile／Target／dimension incompatibility、stale result、stale Catalog version／hash、stale provider version／content hash、MCD kind／type spoof、payload refとschema不一致、payload hash mismatch、duplicate proposal ID、noncanonical entry order、fixture Provider RecordRefのProduction選択、cross-project owner、self-asserted owner、contribution adapter 0件／複数を一原因ずつ拒否する。root-motionあり／なしの双方を検査し、proposalは必ずregistered generic contribution resolver経由の`MotionExecutorIntentBatchV1`でselected executorへ届く。Provider failure fixtureはPath progress、last-valid resolved motion、Path Follower stateを部分更新しない。

Dependency artifact identityとTarget buildは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、測定／capacity promotionは[Runtime performance／capacity](../04-runtime/performance-capacity.md)、evidence envelopeは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を消費する。

次を採用しない。

- Vendor config、native query object、polygon ref、status bits、tile binaryのpublic API化
- Grid2Dを3D Backend上へ実装すること
- query中のlive artifact mutationやexpired leaseの利用
- partial pathを無印のsuccessとして返すこと
- NavigationからWorld Transform、Physics Body、Animation poseへwriteすること
- Runtime phase、shared worker、共通capacity、World streaming policyの再所有
