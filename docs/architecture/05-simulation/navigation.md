# Miraikanai Engine Navigation Contract

- 文書ID: mirakan.arch.simulation-navigation
- 状態: review
- 正本範囲: 2D Grid Navigation、3D Navmesh source／profile／artifact、Navmesh query request／result／status、Navmesh version／lease
- 非正本範囲: Runtime phase／tick／shared worker／capacity、Physics dynamics、Collision event、Animation、World streaming、external dependency version／build pin、AI authorization。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Collision](collision.md)、[Physics](physics.md)、[World](../06-rendering/world.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論とPlatform境界

Navigationは2D Gridと3D NavmeshのEngine-owned profile、artifact、query、version、leaseを公開し、build／query Backendをprivate Adapterへ隔離する。Project C++、GameplayDefinition、AI、Editor、SaveへVendor config、mesh／query object、polygon reference、status bits、allocator、callback、binary formatを公開しない。dependencyのexact version／commit／license／build optionは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)だけが所有する。

NavigationはWorld Transformを書かず、Physicsをstepせず、Animation poseを選ばない。[Collision](collision.md)のstatic geometry／filterをsourceとしてcookし、[Physics](physics.md)の前snapshotからdynamic obstacle inputを受ける。cross-subsystem order、async acceptance、shared worker、lifetimeは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)を消費する。

ModuleはContracts、Core、Grid2D、private Navmesh Backend、Authoring、Editor Projection、Qualification Toolへ分離する。ContractsはEngine value／Port／request／result／statusだけ、Coreはvalidation／build orchestration／version／lease／status normalizationだけを公開する。Backend名で上位処理を分岐せず、Backend capabilityはprivate conformance inputとして扱う。

## 2. Profile、source、2D Grid

| Object | 意味 | persistence |
|---|---|---|
| `GridNav2DProfileV1` | cell semantics、neighbor policy、clearance class、cost policy | Project source |
| `NavAgentProfileV1` | radius、height、climb、slope、query filter ref | Project source |
| `NavAreaProfileV1` | stable area ID、traversability、relative cost、tags | Project source |
| `NavBuildProfile3DV1` | voxel／tile／region／contour／polygon build semantics | Project source |
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

Dynamic obstacleは前snapshotからbounded update inputを受け、新versionのstaging artifactまたはEngine-owned local avoidance overlayへ反映する。同じRuntime slotのPhysics native Worldを直接queryせず、live Navmeshをcallbackからmutateしない。World streaming固有のcell policyは[World](../06-rendering/world.md)へ委譲し、本書はstreaming phaseや共通capacityを定義しない。

## 4. Query、status、version、lease

`NavQueryRequestV1`はrequest ID、Nav World handle／expected version、query kind、start／end、Agent Profile ref、area filter、hard result bound、deadline／producer metadataを持つ。query kindはpath、nearest navigable point、random reachable point、ray／visibility on nav domain、distance-to-boundaryのclosed setである。

`NavQueryResultV1`はrequest ID、observed `NavMeshVersion`、normalized status、projected endpoints、ordered corridor／points、area／link metadata、diagnosticを持つ。公開結果へnative polygon ref、tile pointer、raw status bitsを含めない。Resultはobserved versionと同じlease中だけ解釈でき、別versionへpolygon identityを再利用しない。

`NavQueryStatusV1`は`success | no_path | invalid_request | version_mismatch | result_bound_exceeded | unavailable | cancelled | backend_failure`のclosed enumである。Result bound超過はpartial successにせず、`no_path`とBackend failureを区別する。Pathのcanonical tie-breakはtotal cost、projected endpoint distance、Engine polygon／cell key、point lexicographicを使い、native traversal順を使わない。

`NavWorldHandle`と`NavQueryHandle`はEngine generation handle、`NavMeshVersion`はactive artifact generationのmonotonic runtime identityであり、ProjectのStable IDやartifact content identityを代用しない。`NavWorldLeaseV1`はWorld handle、exact version、immutable Backend world ref、expiry boundaryを束ねる。lease expiry後のresult解釈、version activation中のlive pointer保持、old polygon refのnew version利用を禁止する。

Async resultのdeadlineとacceptanceは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)のcanonical `T20_AsyncIntegrate`を参照する。Navigationはphase順序を再定義せず、accepted resultだけを次のGameplay evaluationから利用できる。Replayへはrequest、accepted result、version／artifact identityを供給し、記録はRuntime ownerのcanonical `T100_ReplayCheckpoint`に接続する。

## 5. Authoring、AI、diagnostic、recovery

公開OperationはWorld／Profile／source／artifactのinspect、Grid／Navmesh Profile作成・更新、area／modifier／link作成・更新・削除、cook、query preview、validation、qualification reportに限定する。全writeは[Project state](../03-authoring/project-state.md)のChangeSetを生成し、live Backendへ直接writeしない。EditorとAIは同じSource Document、validator、preview、cook、undo／redoを使う。

AIは「どのBackendを使うか」ではなく、2D／3D、移動体の大きさ、登れる段差／傾斜、通行可能area、door／jump link、dynamic obstacle、Targetをゲーム上の言葉で確認する。未指定値はassumptionとしてpreviewに表示する。Risk分類、authorization、commit可否は[AI Security／Approval](../01-governance/ai-security-approval.md)を参照し、Approval ruleを再掲しない。

主要diagnostic classはinvalid Profile relation、source geometry incompatible、tile／grid connectivity failure、area／link reference missing、cook unavailable、artifact incompatible、version mismatch、expired lease、result bound exceeded、Backend invariant violationである。各diagnosticはstable code、source path／Stable ID、原因、remediation候補、active artifactを維持したかを返す。

Cook失敗、Worker crash、invalid staging、incompatible promotionではlast valid artifactを維持する。Query Backend faultでは当該requestをfailureにし、active artifactを破壊しない。Runtime fault／publish policy、shared queue／capacity／backpressureはRuntime ownersへ委譲する。

## 6. Qualificationと採用しないもの

QualificationはGrid rasterization／A*、Profile cross-field validation、canonical triangle conversion、tile seam、area／modifier／off-mesh link、nearest projection、path ordering、no-path、result bound、async stale result、version activation／lease expiry、fault injection、Editor／AI previewを含む。同じfixtureを全private Backendへ与え、Engine result／status／diagnosticが一致することを検査する。

Dependency artifact identityとTarget buildは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、測定／capacity promotionは[Runtime performance／capacity](../04-runtime/performance-capacity.md)、evidence envelopeは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を消費する。

次を採用しない。

- Vendor config、native query object、polygon ref、status bits、tile binaryのpublic API化
- Grid2Dを3D Backend上へ実装すること
- query中のlive artifact mutationやexpired leaseの利用
- partial pathを無印のsuccessとして返すこと
- NavigationからWorld Transform、Physics Body、Animation poseへwriteすること
- Runtime phase、shared worker、共通capacity、World streaming policyの再所有
