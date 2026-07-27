# Miraikanai Engine Navigation Contract

- 文書ID: mirakan.arch.simulation-navigation
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: 2D Grid Navigation、canonical A*／高度探索のalgorithm eligibility、3D Navmesh source／profile／artifact、Detour sliced query／version付き結果cache、Navmesh query request／result／status、Navmesh version／lease、Path Following／Movement Intent contract、`MotionExecutorPortV1`
- 非正本範囲: Runtime phase／Simulation Advance／shared worker／capacity、Physics dynamics、Collision event、selected Motion ExecutorによるTransform解決、Animation、World streaming、external dependency version／build pin、AI authorization。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Collision](collision.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)
- 関連文書: [AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Collision](collision.md)、[Physics](physics.md)、[World](../06-rendering/world.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-26

## 1. 結論とPlatform境界

Navigationは[World](../06-rendering/world.md)のexact `WorldSpaceProfileRefV1`から選ばれる2D Gridまたは3D NavmeshのEngine-owned profile、artifact、query、version、leaseを公開し、build／query Backendをprivate Adapterへ隔離する。Project C++、GameplayDefinition、AI、Editor、SaveへVendor config、mesh／query object、polygon reference、status bits、allocator、callback、binary formatを公開しない。query lease、job scratch、artifact versionの一般Pointer／Memory Contractは[Memory／Pointers](../02-foundation/memory-pointers.md)のbindingを消費し、Navmesh固有のversion／stale判定は本書だけが所有する。dependencyのexact version／commit／license／build optionは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)だけが所有する。

NavigationはWorld Transformを書かず、Physicsをstepせず、Animation poseを選ばない。[Collision](collision.md)のstatic geometry／filterをsourceとしてcookし、[Physics](physics.md)の前snapshotからdynamic obstacle inputを受ける。cross-subsystem order、async acceptance、shared worker、lifetimeは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)を消費する。

ModuleはContracts、Core、Grid2D、private Navmesh Backend、Authoring、Editor Projection、Qualification Toolへ分離する。ContractsはEngine value／Port／request／result／statusだけ、Coreはvalidation／build orchestration／version／lease／status normalizationだけを公開する。Backend名で上位処理を分岐せず、Backend capabilityはprivate conformance inputとして扱う。

## 2. Profile、source、2D Grid

| Object | 意味 | persistence |
|---|---|---|
| `GridNav2DProfileV1` | cell semantics、neighbor policy、clearance class、cost policy、`max_path_cells` | Project source |
| `NavAgentProfileV1` | radius、height、climb、slope、query filter ref | Project source |
| `NavAreaProfileV1` | stable area ID、traversability、relative cost、tags | Project source |
| `NavBuildProfile3DV1` | voxel／tile／region／contour／polygon build semantics、`max_straight_path_points` | Project source |
| `NavSourceSetV1` | exact World Document ref／`WorldSpaceProfileRefV1`、canonical static geometry、area、modifier、link source revisions | Project source |

Profile fieldはSI単位とfinite値を使い、agent寸法、cell／voxel寸法、slope、climb、tile relationをcross-field validationする。`NavSourceSetV1.world_space_profile_ref`はsource Worldのexact Profile refとbyte equalityにし、World Profileが導く2D／3D authorityだけがGrid／Navmesh buildを選ぶ。`GridNav2DProfileV1`と`NavBuildProfile3DV1`はそれぞれのWorld Profileに対するcompatibility／build semanticsを表すが、独立したscene dimensionまたはhybrid gameplay authorityを保存・推測しない。Vendor default objectをserializeせず、Engine Profileの全意味を保存する。Backendが表現できないProfileはcook failureにし、近い値へsilent adjustmentしない。

### 2.1 Grid2D

Grid2Dは3D Navmesh Backendへ依存しない。Source boundsをcellへrasterizeし、static blocked、area cost、clearance class、optional directed linkをCooked Grid Artifactへ格納する。neighbor policyはclosed enum、diagonal traversalではcorner cutting policyを明示する。cell coordinateとworld coordinateの変換はartifact origin、cell extent、versionを必須とし、floating positionをidentityとして使わない。

Path queryはprojected start／end、agent clearance、area filter、hard result boundを検証し、canonical A* tie-breakを使う。同scoreではcell coordinate、direction ID、insertion-independent parent keyで決め、heap allocation orderを使わない。Gridのpartial rebuildはstaging artifactを完成させてからversion単位でactivateし、live cell arrayをquery中にmutateしない。

canonical A*のproduction candidateは、worker／in-flight queryごとにProfile上限へ予約したnode poolとbinary heapを再利用し、generation tagによるlazy resetを許す。artifactがdense cell indexを提供する場合はそのbounded indexでnode stateを引く。一般heap allocation／fallback、query間のmutable node共有、全node配列の無条件clearは個別metricにし、hot queryの一般heap allocation／fallbackは0をhard predicateにする。memory layoutを変えてもtotal cost、projected endpoint、parent、canonical tie-break、status、ordered path hashを変えない。

path結果cacheはread-throughのDerived optimizationで、exact keyを次へ固定する。

```text
NavPathCacheKeyV1
  algorithm_ref
  algorithm_profile_revision
  world_space_profile_ref
  artifact_id／artifact_content_hash／NavMeshVersion
  agent_profile_ref
  area_filter_ref／cost_policy_ref／link_policy_ref
  projected_start／projected_end
  result_bound
  key_hash
```

cache valueはnormalized `NavQueryStatusV1`、projected endpoints、ordered corridor／points、area／link metadata、result hashだけを持ち、native ref／query objectを保持しない。keyの一Field、artifact activation generation、Profile／policy、algorithm revisionが変わればmiss／staleとして破棄し、`nearest`／`latest`／位置許容差で別queryへ流用しない。failure／cancelled／backend_failureはcacheしない。cache障害はquery semanticsを変えず、cache更新は成功結果のpublication後にatomicに行う。

`GridNav2DArtifactV1`はexact `world_space_profile_ref`、source/profile identity、origin、extent、dimensions、cell payload、link table、diagnostic summaryを持つimmutable Derived Assetである。Grid dimensionはそのProfileからのDerived projectionであり、Renderer tile mapやPhysics broadphaseをlive backing storeとして参照しない。

## 3. 3D Navmesh cookとartifact

3D cookは次のartifact pipelineをEngine semanticsとして所有する。

1. [Collision](collision.md)のcompatible static geometry revisionをcanonical triangle streamへ変換する。
2. area、modifier、exclusion、off-mesh linkをStable ID順で投影する。
3. validated Agent／Build Profileとsource identityからstaging tilesを生成する。
4. tile seam、connectivity、area、link endpoint、hard boundを検査する。
5. manifest、diagnostic、source mappingを付けたimmutable artifactをpublish候補にする。

`CookedNavWorldV1`はartifact ID、schema ref、exact `world_space_profile_ref`、source set／Profile identity、coordinate frame、tile directory、area table、link table、Backend-private payload refs、build diagnostic summaryを持つ。Navmesh dimensionはそのProfileからのDerived projectionであり、Backend-private tile bytesはEngine query contractからopaqueで、Save、Project C++、AIへ公開しない。

Tile promotionはartifact単位でatomicに行う。部分成功をactive Worldへ混在させない。Source geometry、Profile、area table、Backend lockのいずれかが変われば新しいartifact identityを要求する。Reimport／cook／promotion／rollbackは[Asset lifecycle](../03-authoring/asset-lifecycle.md)を消費する。

`NavModifierV1`はStable ID、shape ref、area override／exclusion、applicabilityを持つ。`NavOffMeshLinkV1`はStable ID、typed endpoints、direction、agent／area filter、traversal tagを持つ。linkのgameplay実行はGameplay ownerの責務で、Navigationはpath candidateとlink metadataだけを返す。

Dynamic obstacleは前snapshotからbounded update inputを受ける。C1の動的変化（door閉鎖等のNavModifier／area／off-mesh link変更を含む）は、差分re-cookによる新versionのstaging artifactとversion切替だけで反映する。差分buildはsource／Profile identityが変わらないtileのpayload再利用を許すが、artifact identityは新規に発行し、Tile promotionのatomic ruleに従う。Engine-owned local avoidance overlayはC2の複数Agent local avoidance専用であり、C1経路では使わない。obstacle input受領からversion activateまでの反映latency boundは[Runtime performance／capacity](../04-runtime/performance-capacity.md)が所有し、本書は値を定義しない。同じRuntime slotのPhysics native Worldを直接queryせず、live Navmeshをcallbackからmutateしない。World streaming固有のcell policyは[World](../06-rendering/world.md)へ委譲し、本書はstreaming phaseや共通capacityを定義しない。

Detour private Adapterのsliced path candidateは、in-flight query／workerごとに専用`dtNavMeshQuery`を持つ。`initSlicedFindPath -> updateSlicedFindPath -> finalizeSlicedFindPath`の開始からfinalizeまで、同じquery objectへ別のsliced queryまたはnon-sliced queryを呼ばない。`dtNavMeshQuery::init(nav,maxNodes)`の`maxNodes`はTarget／query execution Profileが固定する1～65,535のexact値で、pool枯渇はtyped `backend_failure`と一原因diagnosticにし、partial pathをsuccessへ変換しない。`maxIter`は同Profileのadvance／deadline budgetへ固定し、sliced executionは完了時点だけを分割してpath semantics、tie-break、result boundを変更しない。`finalizeSlicedFindPathPartial`はcallerが明示した別query kind／contractがActivationされるまで通常path successに使わない。

`dtTileCache::update`がtouchしたtileはstaging `dtNavMesh`だけへ反映し、`upToDate=true`、全tile／seam／area／link validation、artifact hash完成後にだけ新`NavMeshVersion`としてatomic publishする。途中のstaging navmesh、old／new tile混在、Backend poly refをactive queryへ公開しない。公式制約は[Detour `dtNavMeshQuery`](https://recastnav.com/classdtNavMeshQuery.html)と[`dtTileCache`](https://recastnav.com/classdtTileCache.html)を根拠とする。

## 4. Query、status、version、lease

`NavQueryRequestV1`はrequest ID、Nav World handle／expected version、exact `WorldSpaceProfileRefV1`、query kind、start／end、Agent Profile ref、area filter、hard result bound、deadline／producer metadataを持つ。requestのProfile refはactive `GridNav2DArtifactV1`または`CookedNavWorldV1`の同refとbyte equalityにし、query kindはpath、nearest navigable point、random reachable point、ray／visibility on nav domain、distance-to-boundaryのclosed setである。

`NavQueryResultV1`はrequest ID、observed `NavMeshVersion`、normalized status、projected endpoints、ordered corridor／points、area／link metadata、diagnosticを持つ。公開結果へnative polygon ref、tile pointer、raw status bitsを含めない。Resultはobserved versionと同じlease中だけ解釈でき、別versionへpolygon identityを再利用しない。

`NavQueryStatusV1`は`success | no_path | invalid_request | version_mismatch | result_bound_exceeded | unavailable | cancelled | backend_failure`のclosed enumである。Result bound超過はpartial successにせず、`no_path`とBackend failureを区別する。Pathのcanonical tie-breakはtotal cost、projected endpoint distance、Engine polygon／cell key、point lexicographicを使い、native traversal順を使わない。

`NavWorldHandle`と`NavQueryHandle`はEngine generation handle、`NavMeshVersion`はactive artifact generationのmonotonic runtime identityであり、ProjectのStable IDやartifact content identityを代用しない。`NavWorldLeaseV1`はWorld handle、exact version、exact `WorldSpaceProfileRefV1`、immutable Backend world ref、expiry boundaryを束ねる。lease expiry後のresult解釈、version activation中のlive pointer保持、old polygon refのnew version利用を禁止する。

Async resultのdeadlineとacceptanceは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)のcanonical `T20_AsyncIntegrate`を参照する。Navigationはphase順序を再定義せず、accepted resultだけを次のGameplay evaluationから利用できる。Replayへはrequest、accepted result、version／artifact identityを供給し、記録はRuntime ownerのcanonical `T100_ReplayCheckpoint`に接続する。

### 4.1 C1 Path Following／Movement Intent

<a id="PathFollowRequestV1"></a>
<a id="PathFollowerStateV1"></a>
<a id="MovementIntentV1"></a>
<a id="MotionExecutorPortV1"></a>
<a id="Path-Following"></a>
<a id="41-generic-motion-executor-port"></a>

`capability.gameplay.path_following`（成熟度C1。maturityはidentityに含めない）は2D／3D共通のgoal、path generation、waypoint進行、replan、stuck判定をNavigation ownerとして所有する。Navigation query resultとselected Motion Executorの最終authoritative motion解決の間を結び、Path FollowingはWorld Transform、Physics body、Nav payloadを直接writeしない。selected executorのwriter authorityを奪わない。

```text
PathFollowRequestV1
  request_id
  actor_ref
  actor_generation
  world_space_profile_ref: exact WorldSpaceProfileRefV1
  goal: WorldPosition2f | WorldPosition3f | StableAnchorRef
  nav_agent_profile_ref
  executor_capability_ref
  movement_profile_ref
  arrival_radius_m
  replan_policy_ref
  requested_advance_sequence

PathFollowerStateV1
  actor_ref
  request_id
  nav_generation
  path_result_generation
  waypoint_index
  status: awaiting_path | following | arrived | blocked | stuck | stale | cancelled
  last_progress_advance_sequence
  replan_count
  generation

MovementIntentV1
  actor_ref
  actor_generation
  source_request_id
  cadence_profile_ref: SimulationCadenceProfileRefV1
  simulation_advance_interval_hash: SHA-256
  motion_request:
    kind: velocity
      desired_velocity
    | kind: displacement_per_advance
      desired_displacement
  facing_intent
  valid_for_advance_sequence
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
  cadence_profile_ref: SimulationCadenceProfileRefV1
  simulation_advance_interval_hash: SHA-256
  valid_for_advance_sequence
  producer_system_ref: GameSystemContractRefV1
  proposal_schema_ref: McdContractRefV1(kind=type)
  proposal_payload:
    kind: referenced
      proposal_ref: TypedValueRefV1
    | kind: inline
      proposal_value: BoundedTypedValueV1
  proposal_hash: SHA-256

MotionIntentContributionBindingRecordV1
  owner_layer: core | feature_pack | genre_pack | project
  owner_ref: exact GameSystemOwnerRefV1 with matching owner_layer
  proposal_schema_ref: McdContractRefV1(kind=type)
  adapter_policy_ref: McdContractRefV1(kind=policy)
  adapter_output_schema_ref: McdContractRefV1(
    id=type.navigation.adapted_motion_intent, version=1, contract_set_hash)
  compatible_executor_capability_refs[1..64]
  record_content_hash

MotionIntentContributionBindingRegistryRefV1
  registry_id
  registry_version
  registry_content_hash

MotionIntentContributionBindingRecordRefV1
  registry_ref: MotionIntentContributionBindingRegistryRefV1
  proposal_schema_ref: McdContractRefV1(kind=type)
  adapter_output_schema_ref: McdContractRefV1(kind=type)
  owner_layer: core | feature_pack | genre_pack | project
  owner_ref: exact GameSystemOwnerRefV1
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
  selected_executor_provider_activation_binding_ref:
    MotionExecutorProviderActivationBindingRefV1
  selected_binding_record_refs[0..64]:
    MotionIntentContributionBindingRecordRefV1
  selected_binding_qualification_binding_refs[0..64]:
    MotionIntentBindingQualificationBindingRefV1
  selection_content_hash: SHA-256

AdaptedMotionIntentV1
  adapted_intent_id: StableId
  motion_subject_ref
  motion_subject_generation
  cadence_profile_ref: SimulationCadenceProfileRefV1
  simulation_advance_interval_hash: SHA-256
  valid_for_advance_sequence
  source_proposal_schema_ref: McdContractRefV1(kind=type)
  source_proposal_hash: SHA-256
  binding_record_ref: MotionIntentContributionBindingRecordRefV1
  adapter_policy_ref: McdContractRefV1(kind=policy)
  adapted_value: BoundedTypedValueV1(
    value_schema_ref=type.navigation.adapted_motion_intent@1)
  adapted_intent_hash: SHA-256

MotionIntentBindingQualificationSubjectV1
  qualification_id
  qualification_version: positive uint32
  binding_record_ref: MotionIntentContributionBindingRecordRefV1
  owner_ref: GameSystemOwnerRefV1
  target_profile_refs[1..64]
  fixture_refs[1..64]: exact {fixture_id, fixture_version, fixture_content_hash}
  input_closure_hash
  result: pass | fail
  qualification_subject_hash: SHA-256

MotionIntentBindingQualificationReceiptV1
  subject: MotionIntentBindingQualificationSubjectV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=motion_intent_binding_qualification)

MotionIntentBindingQualificationReceiptRefV1
  qualification_id
  qualification_version
  qualification_subject_hash
  signed_record_hash

MotionIntentBindingQualificationBindingRefV1
  qualification_binding_id
  qualification_binding_version
  qualification_binding_hash

MotionIntentBindingQualificationBindingV1
  qualification_binding_id
  qualification_binding_version
  binding_record_ref: MotionIntentContributionBindingRecordRefV1
  qualification_receipt_ref: MotionIntentBindingQualificationReceiptRefV1
  qualification_binding_hash: SHA-256
```

Coreはこのgeneric schema、registry、resolverだけを所有する。`proposal_payload`はdiscriminator外Fieldをcanonical omissionし、reference branchはexact schema／payload hash、inline branchはbound／canonical value hashを検証する。`proposal_hash`はASCII `MIRAKAN_MOTION_INTENT_CONTRIBUTION_V1`と自身だけを除く全FieldのMCD canonical bytesを`uint32_be` length framingしてSHA-256する。

recordsは`proposal_schema_ref.id`、version、owner layer／owner refのcanonical byte順へstrict sortし、duplicate、同じschemaへの複数active adapter、stale owner／policy／adapter output schema、Core ownerを偽装するPack recordを拒否する。record hashはASCII `MIRAKAN_MOTION_INTENT_CONTRIBUTION_BINDING_RECORD_V1`とReceipt-free self-excluding canonical bytes、Registry hashはASCII `MIRAKAN_MOTION_INTENT_CONTRIBUTION_BINDING_REGISTRY_V1`、Registry ID／version、record count、全record canonical bytesを各`uint32_be` length framingしてSHA-256し、自己Fieldを除外する。`MotionIntentContributionBindingRegistryRefV1`は三Fieldすべてを同一active Registryへexact解決し、ID-only／latest fallbackを許可しない。`MotionIntentContributionBindingRecordRefV1`はRegistry三Field、source／output schema ref、owner layer／ref、record hashを一つにbindし、current Registryからlatest recordを引き直さない。

Registry／RecordRef固定後、Qualification subject hashはASCII `MIRAKAN_MOTION_INTENT_BINDING_QUALIFICATION_SUBJECT_V1`、Qualification binding hashはASCII `MIRAKAN_MOTION_INTENT_BINDING_QUALIFICATION_BINDING_V1`と各自己Fieldを除くcanonical bytesから計算する。subjectの`owner_ref`は`binding_record_ref`が解決するReceipt-free base recordの`owner_layer／owner_ref` projectionとbyte equality、Qualification BindingのRecordRefはsubjectとbyte equalityでなければならない。signed wrapperは完成subjectだけを署名し、subject／record／Registry hash preimageへwrapper、Receipt ref、Qualification bindingを含めない。生成順は`receipt-free Binding record → Registry／RecordRef → Qualification subject → signed Receipt → Qualification binding → Selection`である。Selectionのrecord ref集合とQualification bindingが解決するrecord ref集合はexact set equalityであり、Production Selection／Runtime Packageはbindingが指すsigned Receiptのsubject／result／freshnessだけを検証し、Fixture bodyを解決しない。subject owner、RecordRef、Receipt subject、signed hash、Binding RecordRefの各一Fieldを別baseへ差し替えるnegative fixtureを持つ。

個別Feature／Genreのproposal schema、adapter policy、binding、Qualification record／Fixtureは当該Pack ownerが登録し、Core binaryまたはCore fixture inventoryへコピーしない。resolverはcurrent Registry ref、producer schema、selected executor capabilityを入力にexact一件を選び、adapter policyをpure evaluationしてexact `AdaptedMotionIntentV1`を生成する。0件／複数binding、output schema mismatch、adapter hash mismatchなら推測変換しない。Project Sourceの正本は`MotionIntentBindingSelectionDocumentV1`であり、Registry、resolver output、Runtime lookup tableは派生物である。ただしwrite surfaceは本Taskでは完全登録されていないため、`operation.navigation.motion_intent_binding.select`は[Executable contracts](../02-foundation/executable-contracts.md#211-既存domain文書から回収した未登録operation候補)の`planning.operation_family.navigation_binding_selection@1`に属するexact一候補であり、current MCD、Owner Manifest、Service allowlist、Policy、Validator、Diagnostic、Receipt、Provider／MCP Catalog、generated alias、legacy aliasの各集合をすべて`[]`、capability stateを`not_activated`にする。選択要求は`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`でSource不変にする。future work item `activation.navigation.motion_intent_binding_selection.v1`はinitial create/upsertとupdate、selection semantic hash、sort／duplicate、MCD全Field、Policy／Validator／Diagnostic／Receipt、private-to-public recoveryを同じContract set transactionで完全登録するまでactivateしない。Registry reloadは既存Source selectionから決定的に再materializeし、Registryを直接writeするOperationを公開しない。

Compile closureはSelection Document ref／content hash、RegistryRef、全Binding RecordRef、全Qualification Binding ref、Provider RecordRef、Provider Activation Binding ref、exact `ProviderProductionOwnerProjectionRefV1`、exact `ProviderSelectionCompileBindingRefV1`、set hashを持つ。Compile Binding Refは完成BindingのID／version／self-excluding content hashとbyte equalityで、Bindingはcurrent Project containment、Selection、Provider Record、Provider Activation Binding、Owner Projectionの全ref equalityを検証してからだけmaterializeする。Runtime PackageはこのCompile closureとCompile Binding Refをexact inputとして持ち、Project ID／revision／document-set hash、Selection hash、Provider／Activation／Projection hashの一つでもdriftした場合はpackage compileを拒否する。Saveは同じselection ref／hash、RecordRef集合、Binding／Provider Activation Binding集合、Replay headerはそれらに加えて各Simulation Advanceで実際に使ったCadence Profile ref、Interval hash、advance sequence、RecordRef／source proposal hash／adapted intent hashを記録する。Movement／Contribution／BatchのProfile ref、Interval hash、advance sequenceは同じcanonical `SimulationAdvanceIntervalV1`の三値とbyte equalityにする。Load／Replay／CompileはSource→Registry→Record→Qualification subject／signed Receipt／Binding→Provider Record→Owner Projection→Provider subject／signed Receipt／Activation Binding→Project Compile Bindingの全ref equalityを再検査し、Registry／Catalog version／hash、record／subject／Receipt／binding／Projection／Compile Binding hash、selection hashがstaleなproposalを再利用しない。`fixture.navigation.motion-intent-binding-roundtrip`は別Qualification subject内で既存Selection reload→Compile→Save／Load→Replayを通し、record missing／duplicate／stale Registry／cross-owner／source-output schema spoof／Receipt substitution／Sourceとderived Registry差／Compile BindingまたはOwner Projection欠落を各単独原因でrejectする。

Port transportとgeneric batch publisherをNavigation／Core ownerへ固定する。`game_system.engine.navigation.motion_intent_batch_publisher`はactive Selection DocumentとContribution Registryを読み、全owner contributionをcanonical mergeして、T40でselected Providerへ一回だけ配送する。Character、RTS、board token、Animation等のcontributorは自身のproposalとadapter recordだけを所有し、このpublisher、batch identity、merge orderを所有またはforkしない。これはEventではなくbounded Port messageである。

このpublisherはcurrent Core System Catalogへ次の一件だけを完全な`GameSystemSpecV2`として登録する。省略したFieldをdefault扱いせず、空集合もcanonical count 0としてhashへ含める。

```text
GameSystemSpecV2 game_system.engine.navigation.motion_intent_batch_publisher
  MCD common envelope: all fields
  id: game_system.engine.navigation.motion_intent_batch_publisher
  version: 1
  status: active
  title: Navigation Motion Intent Batch Publisher
  description:
    canonically merge registered owner contributions and publish one
    selected-provider batch without owning authoritative motion state
  owners: [owner.core.navigation]
  requirement_refs:
    [requirement.navigation.publish_motion_intent_batch@1,
     requirement.navigation.not_authoritative_motion_writer@1]
  rationale_refs: [mirakan.arch.simulation-navigation#41-generic-motion-executor-port]
  since_contract_set: 2
  supersedes: []
  tags: [core, motion_intent, navigation, port_publisher]
  owner_layer: core
  owner_ref:
    {owner_layer=core, owner_id=owner.core.navigation,
     owner_revision, owner_content_hash}
  system_origin: engine_standard
  semantic_role_refs:
    [exact semantic_role.navigation.motion_intent_batch_publisher ref/version/hash]
  responsibility_requirement_refs:
    [exact requirement.navigation.publish_motion_intent_batch ref]
  non_responsibility_requirement_refs:
    [exact requirement.navigation.not_authoritative_motion_writer ref]
  runtime_scope_type_ref: exact scope.core.world ref/version/hash
  state_class: derived
  owned_state_type_refs: []
  read_snapshot_type_refs:
    [type.navigation.motion_intent_binding_selection@1 exact MCD ref,
     type.navigation.motion_intent_contribution_binding_registry@1 exact MCD ref,
     type.navigation.motion_intent_contribution@1 exact MCD ref,
     type.navigation.motion_executor_provider_catalog@1 exact MCD ref]
  accepted_command_type_refs: []
  emitted_event_type_refs: []
  emitted_port_message_type_refs:
    [type.navigation.motion_executor_intent_batch@1 exact MCD ref]
  provided_capability_refs:
    [capability.navigation.motion_intent_batch_publish@1 exact MCD ref]
  required_capability_refs:
    [capability.navigation.motion_intent_binding_resolve@1 exact MCD ref]
  allowed_phase_ids: [T40_MotionIntent]
  dependency_edge_refs:
    [exact dependency.navigation.publisher.selection ref/version/hash,
     exact dependency.navigation.publisher.binding_registry ref/version/hash,
     exact dependency.navigation.publisher.provider_catalog ref/version/hash,
     exact dependency.navigation.publisher.contributor_port ref/version/hash]
  implementation_policy_ref:
    exact implementation_policy.navigation.motion_intent_batch_publisher ref/version/hash
  save_replay_contract_ref: canonical omission
  behavior_budget_refs:
    [exact budget.navigation.motion_intent_batch_publisher per Target Profile ref/version/hash]
  authoring_surface_ids: []
  fallback_contract: no_fallback; invalid closure leaves last-valid batch unpublished
  compatibility_invariant_refs:
    [exact invariant.navigation.single_batch_publisher ref/version/hash,
     exact invariant.navigation.canonical_contribution_merge ref/version/hash,
     exact invariant.navigation.selected_provider_delivery_once ref/version/hash]
  auxiliary_ref_set_hash: exact domain-separated auxiliary set hash
  extension_policy: sealed
```

表記上の`@1`は`McdContractRefV1 {id, version=1, contract_set_hash}`の短記であり、ID文字列に`@1`を含めない。依存edgeはpublisherがSelection、Binding Registry、Provider Catalog、全contributorのgeneric Portだけを読むことを許可し、Feature／Genre固有proposal schemaへのCore dependencyを許可しない。Core validatorはcurrent System CatalogにこのReceipt-free Spec IDがexact一件、`state_class=derived`、owned state 0件、Save／Replay ref canonical omission、batch emitter exact一件であることを検証する。Spec／Contract set root確定後のexact `GameSystemActivationBindingRefV1`だけがpublisher System contract／Target setのsigned Qualification Receiptを持ち、Catalog entryはSpec refとActivation Binding refを別Fieldで保持する。

<a id="navigation-capability-records"></a>

#### MCD Capability record closure

上記Systemとresolverが参照するNavigation-owned MCD Capabilityは次のexact二件である。Product `capability.simulation.navigation` rowとは別recordであり、Target別Activation Bindingが同じContract set rootへ接続する。materialized Contract setがない現在は設計候補で、Game System本文にIDが現れるだけでは定義済みまたはactiveと扱わない。

共通Envelopeは`mcd_version=1`、`kind=capability`、`version=1`、`status=active`、`owners=[owner.core.navigation]`、`requirement_refs=[]`、`since_contract_set=2`、`supersedes=[]`である。Payloadは`maturity=C1`、`supported_targets=[target.android.mobile, target.apple.mobile, target.headless.host, target.windows.desktop, target.windows.editor]`、`conflicts=[]`、`authoring_types=[]`、`operations=[]`、`validators=[]`、`quality_profiles=[]`、`budgets=[]`、`examples=[]`、`ai_guidance=[]`を共通値とする。

| Capability ID | `title` | `description` | `required_capabilities[]` | `rationale_refs[]` | `tags[]` |
|---|---|---|---|---|---|
| `capability.navigation.motion_intent_binding_resolve` | `Motion intent binding resolver` | owner登録済みproposalをexact一つのadapterで`AdaptedMotionIntentV1`へ解決する | `[]` | `[mirakan.arch.simulation-navigation#41-generic-motion-executor-port]` | `[motion_intent, navigation, resolver]` |
| `capability.navigation.motion_intent_batch_publish` | `Motion intent batch publisher` | 解決済みcontributionをcanonical mergeし、選択Providerへ一batchだけpublishする | `[capability.navigation.motion_intent_binding_resolve@1]` | `[mirakan.arch.simulation-navigation#41-generic-motion-executor-port]` | `[motion_intent, navigation, publisher]` |

両recordの`failure_modes`は`[{diagnostic_code=MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED, fallback_id=fallback.capability.unavailable}]`である。Product Capability、Game System、Selection、Provider Catalogは明示Bindingで接続し、`navigation`等のgeneric IDやProduct maturityからMCD Refを合成しない。

#### 4.1.1 Core publisher補助record closure

上記Specが参照する12件は文字列予約ではなく、次のReceipt-free record inventoryへexactly oneで解決する。二RequirementだけがMCD member、残る10件は`id`、`version=1`、`content_hash`を持つNavigation-owned typed auxiliary recordである。

| exact record | record type | normative content |
|---|---|---|
| `semantic_role.navigation.motion_intent_batch_publisher` v1 | `SemanticRoleRecordV1` | registered contributionだけをcanonical mergeし、selected Provider向けbatchを一件発行する |
| `requirement.navigation.publish_motion_intent_batch` v1 | MCD `requirement` | actor／generation／Simulation Advanceごとにcanonical batchを最大一件publishする`must` |
| `requirement.navigation.not_authoritative_motion_writer` v1 | MCD `requirement` | Transform、Physics、Gameplay authoritative Stateをwriteしない`must_not` |
| `dependency.navigation.publisher.selection` v1 | `GameSystemDependencyEdgeV1` | active Selection Documentをread-onlyで読む |
| `dependency.navigation.publisher.binding_registry` v1 | `GameSystemDependencyEdgeV1` | current contribution Binding Registryをread-onlyで読む |
| `dependency.navigation.publisher.provider_catalog` v1 | `GameSystemDependencyEdgeV1` | selected Providerをcurrent Provider Catalogへexact解決する |
| `dependency.navigation.publisher.contributor_port` v1 | `GameSystemDependencyEdgeV1` | generic contribution PortだけをT40で受理する |
| `implementation_policy.navigation.motion_intent_batch_publisher` v1 | `GameSystemImplementationPolicyV1` | sealed Core C++ implementation、live switch禁止、意味fallbackなし |
| `budget.navigation.motion_intent_batch_publisher` v1 | `BehaviorBudgetRecordV1` | active Targetごとに一行、actor／generation／Simulation Advance当たりbatch 1件、entry 16件 |
| `invariant.navigation.single_batch_publisher` v1 | `CompatibilityInvariantRecordV1` | batch publisherはcurrent Catalogにexact一件 |
| `invariant.navigation.canonical_contribution_merge` v1 | `CompatibilityInvariantRecordV1` | 全entryはregistered adapter出力でcanonical順 |
| `invariant.navigation.selected_provider_delivery_once` v1 | `CompatibilityInvariantRecordV1` | selected Providerへの配送はactor／generation／Simulation Advance当たりexact一回以下 |

全非MCD recordは共通header
`{record_id, record_version=1, record_content_hash, owner_ref={owner_layer=core,owner_id=owner.core.navigation,owner_revision,owner_content_hash}, status=active, introduced_contract_set_local_ref}`を持つ。`introduced_contract_set_local_ref`はrootを含まないlocal identityである。payload内のMCD edgeはroot確定前には`ContractSetLocalRefV1 {kind,id,version}`、root確定後の外部projectionだけが同じrootの`McdContractRefV1`を使う。各`record_content_hash`はASCII `MIRAKAN_NAVIGATION_GAME_SYSTEM_AUXILIARY_RECORD_V1`、record type ordinal、自己hashだけを除くheaderとtyped payloadのMCD canonical bytesをcount／length frameして計算し、System ref、Contract set root、Qualification Receipt、Activation Bindingをpreimageへ含めない。

```text
SemanticRoleRecordV1.payload
  accepted_input_type_local_refs:
    [type.navigation.motion_intent_contribution v1]
  emitted_port_message_type_local_refs:
    [type.navigation.motion_executor_intent_batch v1]
  allowed_phase_ids: [T40_MotionIntent]
  responsibility_requirement_local_refs:
    [requirement.navigation.publish_motion_intent_batch v1]
  non_responsibility_requirement_local_refs:
    [requirement.navigation.not_authoritative_motion_writer v1]
  authoritative_write_type_local_refs: []

GameSystemDependencyEdgeV1.payload
  source_system_local_ref:
    {kind=game_system,
     id=game_system.engine.navigation.motion_intent_batch_publisher,
     version=1}
  target_contract_local_ref:
    selection:
      {kind=type,
       id=type.navigation.motion_intent_binding_selection,version=1}
    | binding_registry:
      {kind=type,
       id=type.navigation.motion_intent_contribution_binding_registry,version=1}
    | provider_catalog:
      {kind=type,
       id=type.navigation.motion_executor_provider_catalog,version=1}
    | contributor_port:
      {kind=type,id=type.navigation.motion_intent_contribution,version=1}
  edge_kind: snapshot_read | port_ingress
  phase_relation: available_before_t40 | accepted_during_t40
  access: read_only
  required: true
  fallback: no_inferred_target

GameSystemImplementationPolicyV1.payload
  allowed_implementation_kinds: [engine_core_cpp]
  default_implementation_artifact_ref:
    {artifact_kind=engine_core_module,
     logical_id=implementation.engine.navigation.motion_intent_batch_publisher,
     schema_version=1,sha256}
  native_eligibility: true
  replacement_policy: sealed
  live_switch_policy: forbidden
  required_target_refs: exact active Project Target set
  configuration_schema_local_ref:
    {kind=type,id=type.game_system.empty_configuration,version=1}
  unavailable_behavior: reject_activation_keep_last_valid

BehaviorBudgetRecordV1.payload
  target_limits[1..64]:
    target_profile_ref: exact active Target Profile ref/version/hash
    max_batches_per_actor_generation_advance: 1
    max_entries_per_batch: 16
    max_inline_payload_bytes_per_entry: 65536
    max_inline_payload_bytes_per_batch: 1048576
  overflow_behavior: typed_reject_no_partial_publish

CompatibilityInvariantRecordV1.payload
  predicate_kind:
    catalog_singleton_by_system_id
    | canonical_registered_contribution_merge
    | selected_provider_at_most_once_delivery
  evaluation_phase: activation | T40_MotionIntent
  input_contract_local_refs[1..8]
  expected: true
  failure_code: MIRAKAN-NAV-MOTION-EXECUTOR-INCOMPATIBLE
  failure_behavior: reject_without_publishing_partial_batch
```

四dependency recordは上記unionの各branchをexact一件ずつ使い、`selection`／`binding_registry`／`provider_catalog`は`edge_kind=snapshot_read, phase_relation=available_before_t40`、`contributor_port`だけは`edge_kind=port_ingress, phase_relation=accepted_during_t40`とする。Role、Policy、Budget、Invariantは表のIDとpayload branchを一対一にする。sealed built-inで構成Fieldを持たない本PolicyはGameplay Programming Model所有のcomplete `type.game_system.empty_configuration` v1 LocalRefを使い、root後は同じContract set hash付きexternal refへ投影する。`null`、zero ref、ownerごとのempty Type複写で代用しない。Budgetの`target_limits[]`はactive Project Target Profile集合とexact set equalityで、Target ref canonical順、duplicateなしとし、Target非依存のdefault行やlatest Target lookupを許可しない。

二Requirementは次の全MCD共通EnvelopeとRequirement payloadを持つ。

```text
mcd_version=1
kind=requirement
id=requirement.navigation.publish_motion_intent_batch
version=1
status=active
title=Publish Canonical Motion Intent Batch
description=Publish at most one canonical selected-provider batch per actor generation and Simulation Advance
owners=[owner.core.navigation]
requirement_refs=[]
rationale_refs=[mirakan.arch.simulation-navigation#41-generic-motion-executor-port]
since_contract_set=2
supersedes=[]
tags=[motion_intent,navigation,publisher]
normative_level=must
priority=blocking
statement=Validated registered contributions map to at most one canonically ordered selected-provider batch for each actor generation and Simulation Advance
scope=[game_system.engine.navigation.motion_intent_batch_publisher,T40_MotionIntent]
verification_methods=[gate.navigation.motion_intent_batch_publisher.contract,gate.navigation.motion_intent_batch_publisher.runtime]
acceptance_criteria=[predicate.navigation.batch_count_at_most_one,predicate.navigation.batch_entries_registered_and_canonical]
failure_code=MIRAKAN-NAV-MOTION-EXECUTOR-INCOMPATIBLE
source_refs=[{ref=mirakan.arch.simulation-navigation#41-generic-motion-executor-port,authority=project_normative}]
introduced_by=changeset.architecture.navigation.motion_intent_batch_publisher.v1

mcd_version=1
kind=requirement
id=requirement.navigation.not_authoritative_motion_writer
version=1
status=active
title=Do Not Write Authoritative Motion State
description=Keep the generic publisher outside Transform Physics and Gameplay state ownership
owners=[owner.core.navigation]
requirement_refs=[]
rationale_refs=[mirakan.arch.simulation-navigation#41-generic-motion-executor-port]
since_contract_set=2
supersedes=[]
tags=[authority,boundary,motion_intent,navigation]
normative_level=must_not
priority=blocking
statement=The publisher must not write Transform Physics or Gameplay authoritative state
scope=[game_system.engine.navigation.motion_intent_batch_publisher,T40_MotionIntent]
verification_methods=[gate.navigation.motion_intent_batch_publisher.contract,gate.navigation.motion_intent_batch_publisher.authority]
acceptance_criteria=[predicate.navigation.publisher_authoritative_write_edge_count_zero]
failure_code=MIRAKAN-NAV-MOTION-EXECUTOR-INCOMPATIBLE
source_refs=[{ref=mirakan.arch.simulation-navigation#41-generic-motion-executor-port,authority=project_normative}]
introduced_by=changeset.architecture.navigation.motion_intent_batch_publisher.v1
```

Contract compilerは二Requirementを`ContractSetMemberLocalRecordV2(member_kind=mcd)`、publisher Specを同じkindの別local memberとして同一`ContractSetSnapshotV2`へ含める。Spec内のRequirement／Type／Capability edgeはlocal identityで解決し、10件の非MCD auxiliary record hashと上記12 refのexact `GameSystemAuxiliaryRefSetV1`をSpec local payloadへ含めてからmember hashとset rootを計算する。root確定後だけ二RequirementとSpecのexternal refをmaterializeする。12件の一件missing／extra／duplicate、wrong owner／type／version／hash、Target budget一行missing／extra、各payload一Field mutation、noncanonical ref順でauxiliary hashまたはContract set rootが変わらないfixtureをrejectし、last-valid Catalogを不変にする。

```text
MotionExecutorIntentBatchV1
  batch_id: StableId
  actor_ref
  actor_generation
  cadence_profile_ref: SimulationCadenceProfileRefV1
  simulation_advance_interval_hash: SHA-256
  valid_for_advance_sequence
  provider_record_ref: MotionExecutorProviderRecordRefV1
  entries[1..16]:
    adapted_intent_schema_ref: McdContractRefV1(
      id=type.navigation.adapted_motion_intent, version=1, contract_set_hash)
    source_proposal_schema_ref: McdContractRefV1(kind=type)
    binding_record_ref: MotionIntentContributionBindingRecordRefV1
    payload_kind: referenced | inline_value
    payload_ref: TypedValueRefV1 | null
    payload_value: BoundedTypedValueV1 | null
    payload_value_identity: TypedValueIdentityV1 | null
    payload_hash: SHA-256
    source_proposal_hash: SHA-256
    adapted_intent_hash: SHA-256
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

`payload_kind=referenced`は`payload_ref`だけをnon-nullにし、value／value identityをnullにする。`inline_value`は`payload_value`と`payload_value_identity`をnon-null、refをnullにする。inline canonical bytesは`adapted_intent_schema_ref`でdecode／再encodeしてbyte一致し、value／identity／entryの三hashが一致しなければならない。`payload_ref.value_schema_ref`またはinline value／identityのschema refはentryの`adapted_intent_schema_ref`とexact equalityである。`source_proposal_schema_ref`、source hash、binding refはadapter inputとexact equality、payloadは同bindingが出力した`AdaptedMotionIntentV1.adapted_value/hash`とexact equalityでなければならない。entriesはadapted schema ID／version、source proposal schema ID／version、proposal ID、producer System refのcanonical byte順でstrict sortし、duplicate proposal ID、同一payload identityの重複、source／output type spoof、adapter substitution、payload hash不一致をrejectする。

selected compositionが生成するsource proposal schema集合はBinding Registryにexact一件ずつ解決し、batchが運ぶ`entries[].adapted_intent_schema_ref`集合だけをselected Providerの`accepted_intent_schema_refs[]`とsubset比較する。元のAnimation／Feature proposal schemaをProvider accepted setへ直接比較しない。Cook、Runtime Activation、各batch受理の三箇所でsource→binding→adapted output→Provider accepted setの全edgeを検査する。root motionを生成しないcompositionはroot-motion source bindingを集合へ加えず、Animation `mode=animation`を選ぶcompositionだけがexact source bindingを要求する。`compatibility_predicate_ref`はadapted intent schema、`movement_profile_ref`が`movement_profile_schema_ref`へ適合すること、Target Profile、exact World Space Profile、required Capabilityを検証する。missing provider、同じCapabilityへのactive provider複数、source bindingなし、adapted intent不受理、profile schema不一致、Target不適合はtyped failureであり、別ProviderまたはProfileへ推測fallbackしない。

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
  implementation_system_base_ref:
    UsageTaggedImplementationSystemBaseRefV1
  supported_target_profile_refs[1..64]

MotionExecutorProviderQualificationSubjectV1
  qualification_id
  qualification_version: positive uint32
  provider_record_ref: MotionExecutorProviderRecordRefV1
  owner_identity: exact ProviderOwnerIdentityV1
  production_owner_projection_ref:
    ProviderProductionOwnerProjectionRefV1
    | canonical omission when usage=fixture_only
  implementation_system_ref:
    exact UsageTaggedImplementationSystemRefV1
  target_profile_refs[1..64]
  accepted_intent_schema_refs[1..64]
  input_closure_hash
  result: pass | fail
  qualification_subject_hash: SHA-256

MotionExecutorProviderQualificationReceiptV1
  subject: MotionExecutorProviderQualificationSubjectV1
  signed_record:
    exact MirakanSignedRecordV1(
      purpose=motion_executor_provider_qualification)

MotionExecutorProviderQualificationReceiptRefV1
  qualification_id
  qualification_version: positive uint32
  qualification_subject_hash: SHA-256
  signed_record_hash: SHA-256

MotionExecutorProviderActivationBindingRefV1
  activation_binding_id
  activation_binding_version: positive uint32
  activation_binding_hash: SHA-256

MotionExecutorProviderActivationBindingV1
  activation_binding_id
  activation_binding_version: positive uint32
  provider_record_ref: MotionExecutorProviderRecordRefV1
  qualification_receipt_refs[1..64]:
    MotionExecutorProviderQualificationReceiptRefV1
  activation_binding_hash: SHA-256

ProviderOwnerIdentityV1
  owner_layer: core | feature_pack | genre_pack | project | fixture
  core_system_owner_ref: exact GameSystemOwnerRefV1 | null
  pack_contract_ref: exact PackContractRefV1 | null
  project_identity_ref: exact {project_id} | null
  fixture_owner_ref: exact fixture ref/hash | null

ProviderProductionOwnerProjectionRefV1
  projection_id
  projection_version: positive uint32
  projection_hash: SHA-256

ProviderProductionOwnerProjectionV1
  projection_id
  projection_version: positive uint32
  provider_record_ref: exact MotionExecutorProviderRecordRefV1
  provider_owner_identity:
    exact ProviderOwnerIdentityV1(owner_layer != fixture)
  game_system_owner_ref: exact GameSystemOwnerRefV1
  projection_hash: SHA-256

ProviderSelectionCompileBindingV1
  compile_binding_id
  compile_binding_version: positive uint32
  project_containment_ref:
    exact {project_id, project_revision, project_document_set_hash}
  selection_document_ref/hash:
    exact MotionIntentBindingSelectionDocumentV1
  provider_record_ref: exact MotionExecutorProviderRecordRefV1
  provider_activation_binding_ref:
    exact MotionExecutorProviderActivationBindingRefV1
  production_owner_projection_ref:
    exact ProviderProductionOwnerProjectionRefV1
  compile_binding_hash: SHA-256

ProviderSelectionCompileBindingRefV1
  compile_binding_id
  compile_binding_version: positive uint32
  compile_binding_hash: SHA-256
```

owner unionはdiscriminator外Fieldをnullにする。`owner_layer=core`は`core_system_owner_ref`だけをnon-nullにし、その`GameSystemOwnerRefV1`は同Provider recordのproduction implementation baseが指すReceipt-free `GameSystemSpecV2.owner_ref`と全Field byte equalityでなければならない。これによりSystem contractとは別のEngine Component identity／Registryを作らない。`owner_layer=feature_pack | genre_pack`は`pack_contract_ref`だけをnon-nullにする。そのRefは[Pack Contract](../08-packs/pack-contract.md)のexact `PackManifestV1 {pack_id,pack_version,content_hash}`へ一件だけ解決し、Manifestの`pack_kind`がそれぞれ`feature | genre`、三FieldがCatalog owner inventoryとbyte equalityでなければならない。未定義の中間`PackContractV1`、bare Pack ID、latest Manifestを介さない。`usage=production`は`core | feature_pack | genre_pack | project`だけ、`fixture_only`は`fixture`だけを許可する。base recordの`implementation_system_base_ref.usage`とQualification subjectの`implementation_system_ref.base_ref.usage`はenclosing usageとexact equalityである。

Production ownerの照合は異型union同士の直接比較を行わず、root外のexact `ProviderProductionOwnerProjectionV1`を介す。CoreはProvider recordのexact `core_system_owner_ref`、Feature／Genreはexact Pack owner inventory、Projectはstable `project_identity_ref`とtrusted Project owner inventoryがそれぞれ登録した一件の`GameSystemOwnerRefV1`へだけ投影できる。Provider `core | feature_pack | genre_pack | project`はGame System `core | feature_pack | genre_pack | project`の同ordinalへ写像し、projectionの`provider_record_ref`は対象Record、provider identityは同Recordの`owner_identity`、`game_system_owner_ref`は参照先`GameSystemSpecV2.owner_ref`とbyte equalityでなければならない。Coreではこの三者に加えてProvider `owner_identity.core_system_owner_ref`も同じOwner refとbyte equalityにする。projection hashはASCII `MIRAKAN_PROVIDER_PRODUCTION_OWNER_PROJECTION_V1`と自己Fieldを除くcanonical bytesから計算する。Provider recordはProjection ref、Project revision、Document-set hashを持たず、production Qualification subjectだけがexact Projection refを持つ。production base branchは`production_system_ref: GameSystemContractRefV1`と`production_system_contract_hash`だけ、fixture-only base branchはProjection Fieldをcanonical omissionして`fixture_system_ref: FixtureImplementationSystemRefV1`だけを持つ。Provider Qualification subjectの`implementation_system_ref.base_ref`はbase recordの`implementation_system_base_ref`とbyte equalityにし、その後段でproductionならexact `production_system_activation_binding_ref`、fixtureならexact `fixture_system_activation_binding_ref`を加える。fixture Registry record／System Qualification subjectの`fixture_owner_ref`はProvider ownerの同型fixture refとexact equalityにする。Core namespace、Pack kind、fixture名の自己申告だけでは所有を証明しない。

recordは`provider_content_hash`を除くReceipt-free全FieldのMCD canonical bytesからhashし、CatalogはASCII `MIRAKAN_MOTION_EXECUTOR_PROVIDER_CATALOG_V1`、catalog ID／version、Contract set hash、provider ID／version順でstrict sortしたrecord全体を入力して自己hashを除外する。`MotionExecutorProviderRecordRefV1`はCatalog identity／version／hash／Contract setとrecord identity／version／content hashを一つにbindし、current Catalogからlatest recordを再解決しない。Catalog／RecordRef固定後、Production Owner Projectionを作り、既に完成したSystem Activation Bindingをbase refへ付加した`UsageTaggedImplementationSystemRefV1`とProjection refをProvider Qualification subjectへだけ入れる。Qualification subject hashはASCII `MIRAKAN_MOTION_EXECUTOR_PROVIDER_QUALIFICATION_SUBJECT_V1`、Activation binding hashはASCII `MIRAKAN_MOTION_EXECUTOR_PROVIDER_ACTIVATION_BINDING_V1`と各自己Fieldを除くcanonical bytesから計算する。subjectの`owner_identity`はbase recordの同Field、Production Projectionは同RecordRef／owner／System owner、`implementation_system_ref.base_ref`はbase recordの`implementation_system_base_ref`とbyte equality、Activation BindingのRecordRefはsubjectとbyte equalityでなければならない。signed wrapperは完成subjectだけを署名し、System／Provider record、Catalog hashへReceipt、Projection、wrapper、System／Provider Activation Bindingを戻さない。生成順は`receipt-free System base → System subject／signed Receipt／Activation Binding; receipt-free Provider record(System base ref only) → Catalog／RecordRef → root外Owner Projection → Provider subject(System Activation Binding＋Owner Projection) → signed Provider Receipt → Provider Activation Binding → Selection Document／Project document-set hash → root外ProviderSelectionCompileBindingV1`である。

Compile Binding hashはASCII `MIRAKAN_PROVIDER_SELECTION_COMPILE_BINDING_V1`と自己Fieldを除くcanonical bytesから計算する。`project_containment_ref`は全production owner branchで現在compileするSelection DocumentのProject tripleとbyte equalityにするが、Provider ownerとの照合はtagged ruleにする。`owner_layer=project`だけがProvider recordのstable `project_identity_ref.project_id`とcontainmentの`project_id`をexact equalityにする。`core`は`project_identity_ref=null`のままProvider core System owner ref→Owner Projection→同じCore Game System ownerを、`feature_pack | genre_pack`は同FieldをnullのままPack Manifest inventory→Owner Projection→同ordinal Pack Game System ownerを検証し、compile対象ProjectはそのProviderを選択するcontainment contextとしてだけ保持する。Core／Pack Providerへ架空のProject IDを補完せず、Project containmentとOwner identityを異型比較しない。全branchでSelection内Provider RecordRef／Activation BindingはCompile Bindingの同Field、Owner ProjectionはReceipt subjectの同Refとbyte equalityである。Compile BindingをSelection、Provider、Catalog、Receipt、Activation Binding、Project document-set hashへ戻さない。Batch、Selection State、Save、Replayはproduction RecordRef、そのRecordRefをexact subjectにするProvider Activation Binding ref、subjectが固定したSystem Activation Binding refを保存し、`provider_binding_ref`＋別hash、Capability ID、表示名で代用しない。fixture record、fixture Activation Binding、`FixtureImplementationSystemRefV1`をProduction Catalog、Project Source／Save／Replay、Compile Manifest、Runtime Packageへ選択しない。全branch共通fixtureはcurrent Project containment、subject owner、System base ref、System Activation Binding、owner projection、Provider RecordRef、signed Receipt、Provider Activation Bindingの各一Field差し替えをrejectする。Project branchはさらにstable Project IDのcross-project substitutionを、Core branchはcore System owner refだけを別valid Core ownerへ差し替えるcaseを、Core／Pack branchは`project_identity_ref`をnon-nullにするdiscriminator外FieldとProject containmentからownerを推論するcaseを一原因ずつrejectする。

`arrival_radius_m`はfiniteな0.01～10 mとする。`executor_capability_ref`は選択Provider Capability、`movement_profile_ref`はそのProvider-owned schemaのinstanceを参照する。request validationは`nav_agent_profile_ref`のslope／climb／clearance semanticsとProviderのcompatibility predicateを検査し、Navigation上はtraversableだがexecutorが通行できない組合せをinvalid Profile relationとして拒否する。C1 waypointは`GridNav2DProfileV1`の`max_path_cells`（C1上限8,192 cell）または`NavBuildProfile3DV1`の`max_straight_path_points`（C1上限256 point）に従い、上限値の正本はProfile fieldである。path resultはNav generation、actor generation、request IDがすべて一致した場合だけ統合し、不一致は`stale`としてtypedに扱い、異なるrequestへ推測で転用しない。goal移動、Nav generation変更、path corridor逸脱、`PathFollowerStateV1.status`の`blocked`遷移だけがreplan契機である。`blocked`はProvider stateではなくPath Follower述語であり、Portの`resolved_motion_schema_ref`に適合する結果から得た実進捗だけが、`MovementIntentV1`の要求displacementに対して`replan_policy_ref`の進捗閾値未満であるadvanceが、同policyのblocked判定advance数だけ連続した場合に遷移する。Physics snapshot、Transform component、Animation pose、Provider-private stateから進捗を迂回判定しない。`replan_policy_ref`は進捗閾値、blocked判定advance数、最短replan間隔、最大replan回数、stuck advance上限を必須とする。上限到達時は`stuck`へ遷移してboundedに停止し、各advanceで無制限queryを発行しない。

accepted Navigation resultを`T20_AsyncIntegrate`で統合し、Path Followingは`T30`でactor当たり一つだけの`MovementIntentV1`を生成し、selected Motion Executorが`T40`で解決する。`MovementIntentV1`はproposalであり、actual displacement、grounding、slide、collision responseを所有しない。`velocity` branchは同じ`SimulationAdvanceIntervalV1`のnon-null exact rational durationを使えるqualified Providerだけが受理し、`displacement_per_advance` branchはduration-nullのturn-based／explicit-stepでも使用できる。Profile ref、interval hash、advance sequenceの一件でもBatchと不一致なら副作用前にstaleとして拒否し、fixed 60 Hzへ変換しない。C1 reference recipeはPhysics Kinematic Motion Providerを選択できるが、board-token／RTS executor stubはPhysicsなしで同じPortへ接続できる。複数Agentのlocal avoidance、flow field、shared corridor optimizationはC2に留める。

ownerが永続化を要求した場合だけSaveへdestination intent、request semantic、`executor_capability_ref`、movement profile identity、replan countを保存する。Scope instance keyとSave identityをruntime generationへ置換しない。waypoint index、Nav path point配列、native query handle、Physics stateは保存しない。Loadは保存済みNav generationやpath進捗を信用せず、同じdestination intentとProfileから再queryする。ReplayはRequest、採用path result hash、Movement Intent、selected executor／profile hash、accepted resolved motion、arrived／stuck／replan Eventを記録する。C1 fixtureは2D moving-anchor follow／interaction-range arrival、3D Navmesh追跡、Physicsなしboard-token、PhysicsなしRTS unitを用い、destination移動、passage閉鎖、stale result、owner scope deactivate、blocked recovery、Save／Load、Replay一致を検証する。

## 5. Authoring planned actions、AI、diagnostic、recovery

World／Profile／source／artifactのinspect、Grid／Navmesh Profile作成・更新、area／modifier／link作成・更新・削除、cook、query preview、validation、qualification reportはStable IDでないplanned semantic action vocabularyであり、current公開Operationではない。Navigationのcurrent MCD Operation集合は空、exact一件のselection候補は`planning.operation_family.navigation_binding_selection@1`で`not_activated`である。future work item `activation.navigation.authoring_operations.v1`が採用するexact ID集合と完全なMCD／Service／Policy／Validator／Diagnostic／Receipt／publication closureを一transactionで登録するまでaction名からIDを生成せずdispatchしない。Activation後のwriteだけが[Project state](../03-authoring/project-state.md)のChangeSetを生成し、live Backendへ直接writeしない。EditorとAIは同じSource Document、validator、preview、cook、undo／redoを使う。

AIは「どのBackendを使うか」ではなく、選択済みWorld Space Profile、移動体の大きさ、登れる段差／傾斜、通行可能area、door／jump link、dynamic obstacle、Targetをゲーム上の言葉で確認する。World Profileがexactに選択済みなら2D／3DをNavigation側で再選択しない。未指定値はassumptionとしてpreviewに表示する。Risk分類、authorization、commit可否は[AI Security／Approval](../01-governance/ai-security-approval.md)を参照し、Approval ruleを再掲しない。

algorithm optimizationの説明は[Runtime performance／capacity §8.4](../04-runtime/performance-capacity.md#84-algorithm-optimization-candidate-qualification)の`OptimizationDecisionProjectionV1`をread-onlyで消費する。AIはA*／Detour／JPS／ALTの選択、cache key、node／iteration bound、artifact version、Receiptを変更せず、raw Backend query object、poly ref、node pool、full traceを受け取らない。説明可能であることをOperation Activationまたはproduction selectionの証拠にしない。

主要diagnostic classはinvalid Profile relation、World Space Profile mismatch、source geometry incompatible、tile／grid connectivity failure、area／link reference missing、cook unavailable、artifact incompatible、version mismatch、expired lease、result bound exceeded、Backend invariant violation、motion executor missing／duplicate／incompatible、stale resolved motion、provider failureである。Motion Executor共通codeは`MIRAKAN-NAV-MOTION-EXECUTOR-INCOMPATIBLE`と`MIRAKAN-NAV-MOTION-EXECUTOR-STALE_RESULT`へ固定する。前者はintent subset／exact World Profile／Target不一致、後者はactor／request／provider generation不一致を表す。各diagnosticはstable code、source path／Stable ID、原因、remediation候補、active artifact／last-valid resolved motionを維持したかを返す。

Cook失敗、Worker crash、invalid staging、incompatible promotionではlast valid artifactを維持する。Query Backend faultでは当該requestをfailureにし、active artifactを破壊しない。Runtime fault／publish policy、shared queue／capacity／backpressureはRuntime ownersへ委譲する。

## 6. Qualificationと採用しないもの

高度探索候補のeligibilityを次へ固定する。これらはProduct Priorityであってcurrent Operation、production selection、C1要件ではない。

| algorithm | eligible input／必要Evidence | disposition |
|---|---|---|
| JPS | uniform-cost Grid、compatibleなsymmetric movement／corner policy、static traversal graph。canonical A*とcost、status、ordered path／tie-breakが一致する場合だけ透明なcandidateにする | `conditional_research` |
| ALT | immutable weighted／directed graph artifact、landmark selection／preprocessing artifact hash、admissible lower-bound proofを持ち、canonical A*とcost、status、ordered path／tie-breakが一致する場合だけcandidateにする | `conditional_research` |
| D* Lite | unknown／changing graphに対するrepeated replanning、incremental state lifetime、update ordering、Save／Replay契約を先に定義する必要がある | `deferred` |
| HPA* | hierarchy／portal／abstract pathのbuild、suboptimalityまたはexact-path contract、dynamic updateを先に定義する必要がある | `deferred` |
| flow field／local avoidance | 多数Agentのfield共有、crowd motion authority、Physics／Animationとの合成契約が必要でC2以降の別Capabilityである | `deferred` |

JPS／ALTがcanonical resultを保てない場合はtransparent optimizationにせず、新しいalgorithm／profile contract version、専用fixture、Projectの明示選択を要求する。sourceの自動変換、旧A*との暗黙dual shipping、runtime自動切替を行わない。研究根拠は[JPS原論文](https://ojs.aaai.org/index.php/AAAI/article/view/7994)、[ALT原論文](https://www.microsoft.com/en-us/research/publication/computing-the-shortest-path-a-search-meets-graph-theory/)、[D* Lite原論文](https://www.cs.cmu.edu/~maxim/files/dlite_icra02.pdf)とし、Libraryの公式推奨と誤記しない。

QualificationはGrid rasterization／canonical A*、node／heap memory再利用、Profile cross-field validation、exact World Space ProfileからのGrid／Navmesh selection、canonical triangle conversion、tile seam、area／modifier／off-mesh link、nearest projection、path ordering、no-path、result bound、version付きcache hit／miss／stale、Detour sliced budget／out-of-nodes／finalize、tile-cache staging／atomic publish、async stale result、version activation／lease expiry、fault injection、Editor／AI previewを含む。同じfixtureを全private Backendへ与え、Engine result／status／diagnosticが一致することを検査する。missing／stale World Profile、Grid／Navmesh profile mismatch、hybrid Worldのnon-authoritative gameplay branch選択、sliced query object再利用、partial finalizeの通常success化、cache key一Field差、`upToDate=false` publishを各一原因negative fixtureにする。A*／cache metricはP50／P95／P99、node expansion、heap operation、一般allocation／fallback、cache hit／miss／stale、result hashを、Detourはout-of-nodes、iteration、completion advance、tile update／publish latencyを必須にする。

`fixture.navigation.motion-executor.physics-kinematic`はPhysics reference Providerを検証する。Physics unavailable用の二つのfixture-only Provider recordは`MotionExecutorProviderCatalogV1`へ次の値で登録し、Production選択またはProject Saveへ昇格しない。表中の全MCD IDは`version=1`とfixture Contract set hashを持つ`McdContractRefV1`、Provider／owner／fixtureはversion／content hash付きexact refである。

| Provider ID／usage／owner | `executor_capability_ref.id` | `movement_profile_schema_ref.id` | `accepted_intent_schema_refs[].id` | transport／resolved type ID | compatibility policy ID |
|---|---|---|---|---|---|
| `provider.fixture.motion_executor.board_token` v1／fixture_only／exact `fixture.navigation.motion-executor.board-token-no-physics` owner | `capability.motion_executor.fixture.board_token` | `type.fixture.motion.board_token_movement_profile` | `[type.navigation.adapted_motion_intent]` | `type.navigation.motion_executor_intent_batch`／`type.fixture.motion.board_token_resolved_motion` | `policy.fixture.motion.board_token_target_dimension` |
| `provider.fixture.motion_executor.rts_stub` v1／fixture_only／exact `fixture.navigation.motion-executor.rts-stub-no-physics` owner | `capability.motion_executor.fixture.rts_stub` | `type.fixture.motion.rts_stub_movement_profile` | `[type.navigation.adapted_motion_intent]` | `type.navigation.motion_executor_intent_batch`／`type.fixture.motion.rts_stub_resolved_motion` | `policy.fixture.motion.rts_stub_target_dimension` |

二つの`executor_capability_ref`は未定義の文字列ではなく、対応Fixture専用Contract setだけに含む次のMCD Capability recordである。両recordは`mcd_version=1`、`kind=capability`、`version=1`、`status=active`、`owners=[owner.core.navigation]`、`requirement_refs=[]`、`since_contract_set=2`、`supersedes=[]`、`maturity=C1`、`supported_targets=[target.headless.host]`、`required_capabilities=[]`を持つ。Payloadの共通値は`conflicts=[]`、`authoring_types=[]`、`operations=[]`、`validators=[]`、`quality_profiles=[]`、`budgets=[]`、`failure_modes=[{diagnostic_code=MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED, fallback_id=fallback.capability.unavailable}]`、`examples=[]`、`ai_guidance=[]`である。Target Profile ref／hashは§6のFixture Qualification subjectが`target.headless.host@1`へ別に束縛する。Production Contract set、Product `CapabilityRegistryV1`、通常Provider discoveryにはrecord自体を含めない。

| Capability ID | `title` | `description` | `rationale_refs[]` | `tags[]` | usage boundary |
|---|---|---|---|---|---|
| `capability.motion_executor.fixture.board_token` | `Board-token fixture motion executor` | Physicsなしのboard-token movementを検証するfixture executor | `[mirakan.arch.simulation-navigation#6-qualificationと採用しないもの]` | `[fixture, motion_executor, navigation]` | `fixture.navigation.motion-executor.board-token-no-physics`だけ |
| `capability.motion_executor.fixture.rts_stub` | `RTS-stub fixture motion executor` | PhysicsなしのRTS movementを検証するfixture executor | `[mirakan.arch.simulation-navigation#6-qualificationと採用しないもの]` | `[fixture, motion_executor, navigation]` | `fixture.navigation.motion-executor.rts-stub-no-physics`だけ |

両fixture-only implementation base recordを次の全Fieldで固定する。これら二件は[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)が所有するcanonical `FixtureImplementationSystemRecordV1` schemaの具体instanceであり、schemaの再定義ではない。全`@1` MCD／Target／Fixture refはversionとcontent hashまたはContract set rootを持つexact refであり、空配列もrecord hashへcount 0で含める。

```text
FixtureImplementationSystemRecordV1
  fixture_system_id: fixture_system.navigation.board_token_motion_executor
  fixture_system_version: 1
  fixture_owner_ref:
    {fixture.navigation.motion-executor.board-token-no-physics,1,
     fixture_content_hash}
  implementation_artifact_ref:
    {artifact_kind=fixture_system.navigation.board_token_motion_executor,
     schema_version=1,sha256}
  read_type_refs:
    [type.navigation.adapted_motion_intent@1,
     type.fixture.motion.board_token_movement_profile@1]
  accepted_command_type_refs: []
  accepted_port_message_type_refs:
    [type.navigation.motion_executor_intent_batch@1]
  emitted_event_type_refs: []
  emitted_port_message_type_refs:
    [type.fixture.motion.board_token_resolved_motion@1]
  supported_target_profile_refs: [target.headless.host@1]
  fixture_system_content_hash:
    SHA-256(MIRAKAN_FIXTURE_IMPLEMENTATION_SYSTEM_RECORD_V1,
      self-excluding canonical fields)

FixtureImplementationSystemRecordV1
  fixture_system_id: fixture_system.navigation.rts_stub_motion_executor
  fixture_system_version: 1
  fixture_owner_ref:
    {fixture.navigation.motion-executor.rts-stub-no-physics,1,
     fixture_content_hash}
  implementation_artifact_ref:
    {artifact_kind=fixture_system.navigation.rts_stub_motion_executor,
     schema_version=1,sha256}
  read_type_refs:
    [type.navigation.adapted_motion_intent@1,
     type.fixture.motion.rts_stub_movement_profile@1]
  accepted_command_type_refs: []
  accepted_port_message_type_refs:
    [type.navigation.motion_executor_intent_batch@1]
  emitted_event_type_refs: []
  emitted_port_message_type_refs:
    [type.fixture.motion.rts_stub_resolved_motion@1]
  supported_target_profile_refs: [target.headless.host@1]
  fixture_system_content_hash:
    SHA-256(MIRAKAN_FIXTURE_IMPLEMENTATION_SYSTEM_RECORD_V1,
      self-excluding canonical fields)
```

Registry／System ref確定後、`qualification.fixture_system.navigation.board_token_motion_executor@1`／`qualification.fixture_system.navigation.rts_stub_motion_executor@1`のsubjectは対応System ref、上表owner、`target.headless.host@1`、各fixture一件、input closure、`result=pass`を全Fieldで持つ。canonical signed wrapperを発行後、`activation.fixture_system.navigation.board_token_motion_executor@1`／`activation.fixture_system.navigation.rts_stub_motion_executor@1`が対応System ref＋四FieldQualification Receipt refを持つ。ID／ownerだけからArtifact、Type、Target、Receiptを補完しない。

両policyはfixture Target allowlist、World Profileから導く2D／3D execution、profile schema、intent subsetを全件検証する。`fixture.navigation.motion-executor.board-token-no-physics`と`fixture.navigation.motion-executor.rts-stub-no-physics`は上記7 Field、Provider identity／version／content hash／owner／usage、Target／World Profile compatibility policy、diagnostic、Physics dependency 0件を検証する。Physics production base recordも同じCatalog schemaへ`usage=production`、Engine owner、exact Receipt-free implementation System base refだけで登録する。Provider Qualification Receipt／Activation BindingとSystem Activation Binding dependencyはCatalog hash確定後のsubject／root外closureだけが保持する。

両fixture Provider base recordの`implementation_system_base_ref`は`usage=fixture_only`、Registry ref=`fixture_implementation_system.registry.active`、各exact `fixture_system.navigation.board_token_motion_executor@1`／`fixture_system.navigation.rts_stub_motion_executor@1` ref／content hashだけを持つ。Catalog／RecordRef確定後、Provider Qualification subjectの`implementation_system_ref`だけが同じbase refと対応するexact Fixture System Activation Binding refを持ち、対応Fixture ownerと両段のQualification subjectがexact equalityである。各Provider自身のReceiptとProvider Activation BindingもProvider base record／Catalog確定後に作る。Physics production base recordは`usage=production`とexact Physics `production_system_ref: GameSystemContractRefV1`＋`production_system_contract_hash`だけを持ち、`production_system_activation_binding_ref`は同じbase refを持つProvider Qualification subjectのdownstream `UsageTaggedImplementationSystemRefV1`だけに存在する。

negative fixtureはmissing／duplicate Provider、Provider record order、composition intent subset不成立、profile／Target／dimension incompatibility、stale result、stale Catalog version／hash、stale provider version／content hash、MCD kind／type spoof、payload refとschema不一致、payload hash mismatch、duplicate proposal ID、noncanonical entry order、fixture Provider RecordRefのProduction選択、cross-project owner、self-asserted owner、contribution adapter 0件／複数に加え、両implementation branchの同時存在／両方欠落、usageとbranch不一致、fixture owner不一致、stale Fixture Registry／record／Qualification subject／signed Receipt／Fixture System Activation Binding hash、stale Provider Qualification subject／signed Receipt／Provider Activation Binding hash、Fixture refのProduction Catalog／Source／Save／Replay／Package混入を一原因ずつ拒否する。root-motionあり／なしの双方を検査し、proposalは必ずregistered generic contribution resolver経由の`MotionExecutorIntentBatchV1`でselected executorへ届く。Provider failure fixtureはPath progress、last-valid resolved motion、Path Follower stateを部分更新しない。

Dependency artifact identityとTarget buildは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、測定／capacity promotionは[Runtime performance／capacity](../04-runtime/performance-capacity.md)、evidence envelopeは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を消費する。

次を採用しない。

- Vendor config、native query object、polygon ref、status bits、tile binaryのpublic API化
- Grid2Dを3D Backend上へ実装すること
- query中のlive artifact mutationやexpired leaseの利用
- partial pathを無印のsuccessとして返すこと
- NavigationからWorld Transform、Physics Body、Animation poseへwriteすること
- Runtime phase、shared worker、共通capacity、World streaming policyの再所有
