# Miraikanai Engine Virtualized／Continuous Geometry Contract

- 文書ID: mirakan.arch.rendering-virtualized-continuous-geometry
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: virtualized／continuous geometryの用語、表現ファミリ境界、semantic authoring intent、Target別feature qualification、cluster hierarchy／page artifactの統合契約、outer LODとinner cutの分離、residency／fallback意味、AI read／preview、固有diagnostic／qualification、未決定事項の型付き管理
- 非正本範囲: Terrain／Foliage Source／identity／domain artifact、GI／reflection／shadow Technique、discrete LOD／HLOD policy、Asset transaction／共通Artifact envelope、Material／Animation意味、World partition、Render pass／resource／queue実行、共通memory／I/O budget、Save／Replay envelope、Tool／SDK／Library lock、Platform activation、AI authorization、実装方式／実装工程／日程。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Product Plan](../00-product/product-plan.md)、[LOD](lod.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Render Graph](render-graph.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)
- 関連文書: [Architecture Plan Closure Review](../appendices/architecture-plan-closure-review.md)、[Advanced Rendering／Multiplayer Ownership Decision](../decisions/2026-07-29-advanced-rendering-multiplayer-ownership.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Persistence／Save](../04-runtime/persistence-save.md)、[Debugging／Observability／Replay](../04-runtime/debugging-observability-replay.md)、[Animation](../05-simulation/animation.md)、[Collision](../05-simulation/collision.md)、[Physics](../05-simulation/physics.md)、[Navigation](../05-simulation/navigation.md)、[Camera](camera.md)、[Materials](materials.md)、[Advanced Light Transport](advanced-light-transport.md)、[Terrain／Foliage](terrain-foliage.md)、[World](world.md)
- 根拠区分: project-decision（外部Engineの記述はofficial-spec、未計測の数値・方式選択はprovisional）
- 外部根拠確認日: 2026-07-29

## 1. 結論

Nanite相当のcontinuous／virtualized geometry LODは詳細な独立計画書を持つ。これは通常のMesh LODにfieldを足すだけの機能ではなく、Source／Cook、hierarchy／page artifact、Target feature matrix、runtime residency、Viewごとのcontinuous cut、material／deformation compatibility、capacity failure、非対応Target fallbackまでを横断する統合Capabilityだからである。

採用する構成は次に固定する。

Terrain／Foliage Ownerはdomain Source、tile／cluster、species／instance identity、Cooked domain artifactとrepresentation requestを所有する。本書はそのrequestに対するvirtualized geometry representation family、hierarchy／page Artifact、inner cut compatibilityだけを所有し、Terrain／Foliage capabilityまたはWorld Cellを吸収しない。Advanced Light Transportがacceleration／coverage representationを要求する場合も、本書はgeometry candidateを返すだけでGI／reflection／shadow Technique、channel support、fallbackを選択しない。

1. [LOD](lod.md)のdiscrete Mesh LOD／HLODを現行かつ常設のfallback正本として維持する。
2. outer LOD Resolverは一つのrepresentation descriptorを選び、そのtarget compositionを「subject aggregation」と「geometry family」の直交二軸で解決する。TargetでCapabilityがactiveでなければ`geometry_family=virtualized`を候補集合へ入れない。
3. virtualized representationが選択されたViewだけで、本書のerror contractとresident hierarchyからinner micro-cluster cutを解決する。inner cutはLOD tier、World Cell、HLOD cluster、Asset generation、Save stateではない。
4. unsupported、unqualified、missing、stale、capacity超過、provider faultでは、同じSource semanticsへ束縛されたexact discrete fallbackへ戻す。表示消失、default Material、別generation混在、Gameplay意味変更をfallbackにしない。
5. Geometry detailはPresentationだけを変える。Collision、Physics、Navigation、Animation clock／event／root motion、Entity identity、Damage、Save／Replay authorityを変更しない。

現時点のactive Target集合、active Qualification Binding集合、callable Operation集合はすべてexact `[]`である。本書の型名、候補enum、比較表は設計候補であり、Runtime、Editor、AI ToolまたはShipping supportの存在を意味しない。

### 1.1 current／targetの読み分け

| 観点 | current | target |
|---|---|---|
| Product | Future Portfolioの`planning_only` | Product activation closureに合格したTargetだけactive |
| Geometry path | discrete Mesh LOD／HLODだけが正本 | outer discrete selectionとinner continuous cutを二層化 |
| Artifact | virtual page／hierarchy artifactなし | Source-preservingなimmutable hierarchy／page closure |
| Runtime | pool、feedback、cut executionなし | bounded residency snapshotとView-local cut |
| AI read | 本書の概念と不足条件を説明可能 | bounded Context／Preview／Diagnosticをexact refで取得 |
| AI write | callable Operation `[]` | Governance activation後もsemantic intentだけを同じChangeSetへproposal |
| Target support | qualified Target `[]` | feature軸ごとのReceiptでqualifiedを明示 |

AIは`planning_only`を`experimental support`、`unsupported`を`fallback可能なのでsupported`、`not_evaluated`を`おそらく対応`と読み替えない。

### 1.2 設計内容の評価

| 評価軸 | 現在の判定 | 根拠／限界 |
|---|---|---|
| 概念の網羅性 | `closed-in-target-design` | Source intentからArtifact、outer／inner selection、residency、fallback、fault、AI、qualificationまで本書で接続した |
| AIの概念理解 | strong | closed vocabulary、exact ref、state、Owner、bounded Context、Preview、Diagnosticを定義した |
| AIの操作可能性 | absent | current callable Operation、MCD、Projection Binding、Validator、Receiptがすべて未materialize／`[]` |
| 表現の自由度 | strong at semantic layer | silhouette、attribute、seam、protected region、Material、deformation、owner extensionを作品意味で宣言できる。provider knobの自由入力は意図的に禁止する |
| 外部Engine比較 | current official evidence | Unreal Engine 5.8、Unity 6.5、Godot stableの公式資料を2026-07-29に確認した。Miraikanai自身の性能／対応Targetの証拠ではない |
| 文書間整合 | structurally closed | §3.1と§12でOwnerと禁止逆入力を固定し、各Owner文書へ同じ境界を反映した |
| 実装／性能／製品対応 | absent／not measured／planning only | 実装、provider、Target Receipt、数値budget、Shipping claimは存在しない |

したがって「AIが設計意図を誤読しにくく、人間と表現要件を相談できる計画書」にはなっているが、「AIが現在操作できる」「Nanite相当性能が出る」「特定platformで対応済み」とは評価しない。

## 2. 用語と外部Engine比較

本書で詳細構造を別記しないtarget `*RefV1`は、Owner固有の`stable_id`、positive `version`、SHA-256 `content_hash`を必須にするexact immutable refである。解決先の同三Fieldとbyte equalityにし、bare ID、display name、path、`latest`、名前の近い型で補完しない。Artifact generation、View generation、Target等を追加で持つRefは各節の明示fieldもbyte equalityにする。

| term | canonical meaning |
|---|---|
| virtualized geometry | Geometry hierarchyとpage集合をimmutable Artifact化し、必要部分をbounded poolへresidentにする表現ファミリ |
| continuous geometry selection | Viewとprojected error contractに応じ、離散Asset LOD番号ではなくhierarchy内のvalid cutを選ぶこと |
| micro-cluster | virtualized artifact内部の小さなgeometry node。World／HLOD／Streaming Cellではない |
| hierarchy cut | 親子が重複せず、対象surfaceを一度だけ覆うmicro-cluster集合 |
| root residency set | representationを安全に描画するため常駐必須のcoarsest closure |
| geometry page | 一つのArtifact generation内だけで解釈できるcontent-addressed transfer／residency単位 |
| outer representation | LOD Resolverが選ぶ一つのrepresentation descriptor。subject aggregationとgeometry familyの二軸をexact bindingで持つ |
| inner cut | virtualized family内部でRender Viewごとに解決するmicro-cluster集合 |

「cluster」という単語だけを型名に使わない。World aggregationは`HlodClusterRefV1`、virtual geometry内部は`VirtualGeometryMicroClusterRefV1`とowner-qualifiedに分け、相互代用を拒否する。

### 2.1 有名Engineの現行公式仕様から確認できること

| Engine | 公式仕様で確認した現状 | Miraikanaiへの判断 |
|---|---|---|
| Unreal Engine 5.8 Nanite | fine-grained detailとocclusionを扱うvirtualized geometry、root geometryのminimum residency、残りのstreaming、unsupported platform／feature向けfallback meshを持つ。Static／Skeletal Meshを扱う一方、deformation、foliage、displacement等はfeatureごとに制約／成熟度が異なり、streaming pool thrashやcandidate／visible cluster capacity超過も明示される。[Nanite Technical Details](https://dev.epicgames.com/documentation/en-us/unreal-engine/nanite-technical-details)、[Nanite Virtualized Geometry](https://dev.epicgames.com/documentation/unreal-engine/nanite-in-unreal-engine?lang=en-US)、[Working with Nanite-enabled Content](https://dev.epicgames.com/documentation/unreal-engine/working-with-naniteenabled-content)、[Nanite Foliage](https://dev.epicgames.com/documentation/unreal-engine/nanite-foliage) | root-first residency、feature軸別qualification、exact fallback、overflow／thrash診断を採る。Nanite名、内部format、既定値、hardware条件をMiraikanai Schemaへ複写しない |
| Unity 6.5 | Mesh LODはimport時生成の離散LODをindex bufferへ格納しscreen sizeで選択する。`LODGroup`は離散levelとcross-fadeを持つ。Mesh LODの公式limitationsは一部systemがLOD0を選ぶこと、skinned simplificationがskin weight／blendshape deformationを簡略化評価へ含めないこと等を明示する。GPU Resident DrawerはBatchRendererGroup／GPU instancingによるdraw-call／CPU最適化で、非互換objectは従来経路へfallbackする。[Mesh LOD introduction](https://docs.unity3d.com/Manual/lod/mesh-lod-introduction.html)、[LOD Group](https://docs.unity3d.com/Manual/class-LODGroup.html)、[GPU Resident Drawer](https://docs.unity3d.com/Manual/urp/gpu-resident-drawer.html) | import生成、selection、deformation compatibility、GPU-driven executionを別責務にする。GPU residentまたはindirect drawだけでcontinuous／virtualized geometry対応と判定しない |
| Godot stable | automatic Mesh LODはimport時に生成されscreen-space metricで選択される。Visibility Rangeはmanual LOD／HLOD／impostorとhysteresisを提供しautomatic Mesh LODと併用できる。[Mesh LOD](https://docs.godotengine.org/en/stable/tutorials/3d/mesh_lod.html)、[Visibility ranges](https://docs.godotengine.org/en/stable/tutorials/3d/visibility_ranges.html) | automatic detailとmanual aggregationの共存を採る。Node visibilityをvirtual page residency、Gameplay relevancyまたはcontinuous cutへ読み替えない |

上表の一次資料からの推論として、Unity／Godotの確認対象は離散／import-time LODとGPU-driven／visibility最適化を説明しており、Nanite同様のpage-streamed continuous geometryを標準機能として説明する資料ではない。よって「有名EngineにもLODがある」ことを本Capabilityの成立根拠にしない。

Unreal Engineの現行hardware条件もTarget qualificationの参考にするが、MiraikanaiのTarget可否はUnrealの可否から継承しない。Graphics API、shader model、atomic capability、driver、memory、device classは[Platform](../07-platform/windows.md)と[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)のexact baseline、およびMiraikanai自身のReceiptで決定する。

## 3. 責務と二層selection

### 3.1 Owner matrix

| Owner | 一意に所有するもの | 所有しないもの |
|---|---|---|
| Product Plan | `planning_only -> active`、対象Product claim、導入可否 | cluster／page schema |
| 本書 | 統合語彙、表現ファミリ、feature matrix、error／fallback／residency意味、Owner間invariant | 各Ownerのsource field、pass、budget、tool lock |
| LOD | outer representation候補、semantic floor、hysteresis、exact fallback chain | inner hierarchy traversal／cut |
| Asset lifecycle | Source import、cook intentの格納、hierarchy／page Artifact、generation、Catalog、promotion | View selection、runtime pool pressure |
| Runtime Package | World／Sectionから必要Artifactへのexact dependency closure | page I/O／residency manager |
| 本書のtarget Residency Coordinator | virtual geometry page要求の検証／merge、root pin意味、generation acceptance、non-root eviction適格性、immutable snapshot publication | generic Artifact Store／I/O、request state、phase、lease一般則、capacity値 |
| Runtime Asset Lifecycle | generic Artifact request／read／decode／upload、dependency、generation、residency／lease／eviction、failure atomicity | virtual geometryのroot／page意味、cut、quality fallback |
| Scheduling／Lifetime | request／completionのphase、job dependency、cancel／stale acceptance、device recovery boundary | page意味、quality fallback、capacity値 |
| Memory／Pointers | immutable lease、generation handle、pool allocation／retireの一般則 | page priority、cut、quality意味 |
| World | plan-local Cell、representation slot、prefetch role／priority intent | page ID、resident set、micro-cluster |
| Camera | selected Render View、projection、extent、cut／generation | error threshold、cluster choice |
| Render Graph | selected familyのvisibility、inner cut execution、draw／dispatch、resource lifetime | Source semantics、page I/O policy、shared budget |
| Performance／Capacity | pool／queue／candidate／visible capacity、measurement、pressure snapshot | geometry quality floor |
| Materials | shading／alpha／deformation interfaceのcompatibility | cluster selection |
| Animation | Skeleton／Skin／Morph／deformation envelopeとsemantic invariant | geometry residency |
| Debugging | bounded record／query／overlay transport | authoritative cut selection |
| Persistence | Save／Replayへの包含／除外 | presentation cacheの復元 |
| Toolchain／Platform | provider／builder／codec／SDK／driver exact lockとTarget qualification | Product activation |

同じfieldを複数Ownerへ複写しない。統合Planは各Ownerのexact refを束ねるだけで、Owner payloadの内容を再定義しない。

### 3.2 outer representationとinner cut

outer selectionでは既存`LodRepresentationDescriptorV1.source_identity_ref`がtarget `VirtualGeometryRepresentationFamilyRefV1`へexact解決する候補を一つのrepresentationとして扱える。これは既存Schemaのcurrent member追加を意味せず、本Capability activation時にCompatibility closureで登録するtarget bindingである。

HLODとvirtualized geometryを排他的な機能にしない。outer representationのtarget compositionは次の二軸を別々に解決する。

| subject aggregation | geometry family | 判定 |
|---|---|---|
| `individual`／`instanced` | `discrete` | 現行LODでvalid |
| `individual`／`instanced` | `virtualized` | Target／feature tupleがqualifiedな場合だけvalid |
| `spatial_proxy` | `discrete` | 現行HLOD proxyでvalid |
| `spatial_proxy` | `virtualized` | HLOD proxy Artifact自身がqualified virtual familyとexact fallbackを持つ場合だけvalid |
| `impostor` | `discrete` | 現行明示representationとしてvalid |
| `impostor` | `virtualized` | 意味不一致としてinvalid |
| `hidden_presentation` | `not_applicable` | geometry familyを持たない |

このcomposition bindingはLOD／HLOD Ownerがouter descriptorのdownstream target bindingとして所有し、Source Identity、HLOD eligibility、Virtual Geometry Planをexact refで束ねる。HLOD Source集合をmicro-clusterへ展開する、virtual pageからHLOD membershipを作る、`spatial_proxy`という名前だけでgeometry familyを推測することを禁止する。

```text
ViewLodCandidateSetRefV1
  + ViewLodContextV1
  + LodResolutionPlanRefV1
  -> outer LOD Resolver
     -> selected representation descriptor
        + subject aggregation: individual／instanced／HLOD proxy／impostor
        + geometry family: discrete／virtualized／not_applicable
     -> geometry_family=virtualized
          + VirtualGeometryResolutionPlanRefV1
          + VirtualGeometryResidencySnapshotRefV1
          -> Render Graph inner cut
             -> VirtualGeometryViewCutSummaryV1
             -> draw／visibility execution
```

inner cutを`LodTierRefV1`へ変換しない。前frameのcutはView-local presentation historyとしてのみ使用でき、outer LOD hysteresis、World activation、Simulation relevancy、Save／Replay inputへ返さない。複数Viewは同じArtifact／residency snapshotを共有できるが、cut、projected error、history、fallback結果を共有しない。

## 4. Semantic authoringと表現自由度

### 4.1 `VirtualGeometryAuthoringIntentV1`

target Source intentはproviderのcluster sizeやcodec optionではなく、作品上守るべき意味を記述する。

```text
VirtualGeometryAuthoringIntentV1
  schema_version: 1
  intent_id: StableId
  intent_version: positive u32
  source_asset_ref: exact AssetSourceRefV1
  representation_role:
    primary_surface | supporting_surface | decorative_surface
    | foliage_aggregate | terrain_patch | owner_extension
  surface_preservation_ref:
    exact VirtualGeometrySurfacePreservationContractRefV1
  protected_region_refs[0..1024]: sorted unique exact refs
  silhouette_importance:
    critical | high | normal | low
  attribute_semantics[1..64]:
    semantic_id
    preservation: exact | bounded_error | may_drop
    error_contract_ref: exact owner-typed ref | null
  topology_constraints:
    preserve_boundaries: bool
    preserve_material_seams: bool
    preserve_uv_seams: bool
    preserve_hard_normals: bool
    allow_non_manifold_source: bool
  deformation_envelope_ref:
    exact DeformationEnvelopeRefV1 | null
  material_compatibility_requirement_ref:
    exact MaterialCompatibilityRequirementRefV1
  discrete_fallback_representation_set_ref:
    exact LodRepresentationSetRefV1
  target_policy_ref: exact VirtualGeometryTargetPolicyRefV1
  owner_extensions[0..32]:
    owner_document_id
    extension_type_ref
    extension_content_hash
  intent_content_hash: SHA-256
```

`owner_extension`と`owner_extensions[]`はCore enumを自由文字列化するescape hatchではない。登録済みOwner、versioned Schema、Target validator、Preview renderer、fallback、Qualification fixtureを同時に持つextensionだけを受理する。unknown owner／type／majorは保存時に拒否し、名前一致で近い型へfallbackしない。

表現自由度は「何でも通すこと」ではなく、silhouette、seam、attribute、deformation、material、protected regionを作品ごとにtypedに宣言し、providerを替えても同じ意味を検証できることで確保する。algorithm knobをSourceへ固定せず、Source最高detailとdiscrete fallbackを残すため、後から別builder、別codec、別Targetへ再Cookできる。

`VirtualGeometryAuthoringIntentRefV1`はintent ID、version、`intent_content_hash`のexact refである。

```text
VirtualGeometrySurfacePreservationContractV1
  schema_version: 1
  preservation_contract_id: StableId
  preservation_contract_version: positive u32
  silhouette_measurement_ref: exact owner-typed metric ref
  seam_measurement_refs[1..8]:
    sorted unique owner-typed metric refs
  attribute_measurement_refs[0..64]:
    sorted unique owner-typed metric refs
  protected_region_fixture_refs[0..64]:
    sorted unique exact fixture refs
  failure_policy: reject_candidate
  preservation_contract_hash: SHA-256

VirtualGeometryTargetPolicyV1
  schema_version: 1
  target_policy_id: StableId
  target_policy_version: positive u32
  candidate_target_profile_refs[1..16]:
    sorted unique exact TargetProfileRefV1
  required_render_roles[1..16]:
    sorted unique closed render roles
  unqualified_target_policy:
    require_discrete_fallback
  target_policy_hash: SHA-256
```

両Refは対応ID、version、hashのexact refである。Preservation Contractは「どう測るか」、Authoring Intentは「どこをどの強さで守るか」を所有し、fieldを複写しない。Target Policyは評価候補と必須render roleだけを宣言し、Product activation、provider、budget、driverまたは`qualified`状態をSourceへ書かない。

```text
VirtualGeometryRepresentationFamilyV1
  schema_version: 1
  representation_family_id: StableId
  representation_family_version: positive u32
  source_asset_ref: exact AssetSourceRefV1
  authoring_intent_ref: exact VirtualGeometryAuthoringIntentRefV1
  discrete_fallback_representation_set_ref:
    exact LodRepresentationSetRefV1
  family_content_hash: SHA-256
```

FamilyはSource意味のidentityでありArtifact、Target Activation Binding、residencyまたはView cutを参照しない。`VirtualGeometryRepresentationFamilyRefV1`はfamily ID、version、Source Asset ref、family hashを持つ。Artifact Manifestとouter `LodRepresentationDescriptorV1.source_identity_ref`は同じFamily refへ解決し、path、mesh名、Artifact generationからfamilyを推測しない。

### 4.2 surface／deformationの不変条件

- protected regionはinteractive cue、顔、手、文字形状、hard-surface edge、terrain boundary等のowner-typed regionを参照できる。
- `preservation=exact`は該当attributeの変更を許さない。実現不能ならCookを失敗させ、`bounded_error`へ黙って緩和しない。
- `may_drop`はauthorが明示したattributeだけに許し、material interface、skin binding、collision、navigation、persistent identityへ伝播しない。
- deformation envelopeが必要なsourceでnullなら対応Feature Requirementを満たせず、評価時Bindingは`qualified`になれない。SourceをstaticとしてCookせず、未評価なら`not_evaluated`、評価済みなら`unsupported`を明示する。
- runtime displacement／world-position deformationがboundsを越える場合はcandidateを描画せず、qualified discrete fallbackへ戻す。

## 5. Feature supportを一つのboolにしない

### 5.1 feature requirementとQualification Binding

```text
VirtualGeometryFeatureRequirementV1
  schema_version: 1
  requirement_id: StableId
  requirement_version: positive u32
  target_profile_ref: exact TargetProfileRefV1
  toolchain_lock_ref: exact ToolchainLockRefV1
  provider_profile_ref: exact VirtualGeometryProviderProfileRefV1
  source_geometry_kind:
    static_mesh | skeletal_mesh | foliage_aggregate
    | terrain_patch | owner_extension
  deformation_kind:
    none | skinning | morph_target | world_position_deformation
    | spline_deformation | displacement | owner_extension
  material_surface_kind:
    opaque | masked | translucent | owner_extension
  render_role:
    main_view | shadow | depth_only | reflection
    | ray_query_fallback | owner_extension
  capability_requirement_refs[0..32]: sorted unique exact refs
  constraint_refs[0..64]: sorted unique exact refs
  discrete_fallback_contract_ref: exact VirtualGeometryFallbackContractRefV1
  requirement_content_hash: SHA-256

VirtualGeometryFeatureQualificationBindingV1
  schema_version: 1
  binding_id: StableId
  binding_version: positive u32
  feature_requirement_ref:
    exact VirtualGeometryFeatureRequirementRefV1
  support_state:
    unsupported | not_evaluated | provisional | qualified
  qualification_receipt_ref:
    exact VirtualGeometryQualificationReceiptRefV1 | null
  binding_content_hash: SHA-256
```

Feature RequirementはTargetで必要な組合せをreceipt-freeに宣言し、Qualification Bindingが現在のEvidence状態を重ねる。`qualified`だけがcandidateをactive Targetへ登録でき、Receiptを必須とする。`unsupported | not_evaluated | provisional`ではReceiptをnullにしてvirtualized candidateを除外し、Requirementのexact discrete fallbackを使う。特定meshでstatic／opaque／main viewがqualifiedでも、skeletal、masked、shadow、displacementまたは別Targetへsupportを横展開しない。

`VirtualGeometryFeatureRequirementRefV1`はrequirement ID／version、Target Profile ref、四つのfeature discriminator、`requirement_content_hash`を持つ。Requirement hashはASCII `MIRAKAN_VIRTUAL_GEOMETRY_FEATURE_REQUIREMENT_V1`と自己hashを除く全Fieldから計算する。Binding hashはASCII `MIRAKAN_VIRTUAL_GEOMETRY_FEATURE_QUALIFICATION_BINDING_V1`と自己hashを除く全Fieldから計算する。Receipt subjectはRequirement ref、fixture、Device／Driver、measured resultを束縛し、Bindingを参照しない。生成順を`Requirement -> Qualification Receipt -> Qualification Binding -> staged Target Activation Binding -> atomic Product Activation／Binding publication`に固定し、Receipt／BindingをRequirementまたはResolution Plan hashへ戻さない。

`VirtualGeometryFeatureQualificationBindingRefV1`はbinding ID、version、exact Feature Requirement ref、`binding_content_hash`を持つ。同じRequirementへ複数のcurrent Bindingを許さず、requalificationは新version／hashとしてActivation Bindingを明示更新する。

current Requirement／Bindingは未materialize、active Binding集合は`[]`である。したがってstatic meshを含む全組合せがcurrentでは`not_activated`であり、対応済みとは主張しない。

## 6. Asset、hierarchy、page統合契約

### 6.1 Artifact closure

[Asset lifecycle](../03-authoring/asset-lifecycle.md)がtargetとして所有する`VirtualGeometryArtifactManifestV1`は次のexact refを束ねる。詳細Schema、Cook transaction、Catalog key、promotionはAsset Ownerを正本とする。

- exact Source Asset ref／revision／content hash
- exact `VirtualGeometryAuthoringIntentV1` ref／hash
- exact `VirtualGeometryRepresentationFamilyRefV1`
- Target Profile、Toolchain lock、Provider profile
- sorted unique exact `VirtualGeometryFeatureRequirementRefV1`集合
- deterministic hierarchy descriptor artifact
- content-addressed page set artifact
- root residency set
- bounds／error／attribute／material／deformation metadata
- discrete fallback representation set
- generation、dependency closure、Artifact content hash

Artifact内部の`page_id`と`micro_cluster_id`は`{artifact_manifest_ref, generation}`内だけで有効なplan-local IDである。World、Save、Project Source、AI authoring output、Material、Animationへ保存しない。Artifact generationを跨ぐID再利用や、旧hierarchyと新page setの混在を拒否する。

### 6.2 error contract

```text
VirtualGeometryErrorContractV1
  schema_version: 1
  error_contract_id: StableId
  error_contract_version: positive u32
  object_error_metric:
    maximum_geometric_deviation_m
  projection_algorithm_ref:
    exact algorithm.lod.projected_metric.v1 ref
  threshold_profile_ref:
    exact Target／Quality-qualified threshold profile ref
  silhouette_contract_ref:
    exact VirtualGeometrySurfacePreservationContractRefV1
  attribute_error_contract_refs[0..64]:
    sorted unique exact refs
  deformation_error_contract_ref:
    exact owner-typed ref | null
  unresolved_error_policy:
    use_discrete_fallback
  error_contract_hash: SHA-256
```

View metricは[LOD](lod.md#4-共通選択契約)の`ViewLodContextV1`と同じprojection algorithm、単位、量子化、境界包含を使う。virtual pathだけ別FOV、別dynamic-resolution解釈、backend floating comparisonを持たない。thresholdの数値はTarget measurement前に固定せず、`not_measured`を0または外部Engine既定値で埋めない。

`VirtualGeometryErrorContractRefV1`は`error_contract_id`、version、`error_contract_hash`のexact refである。

### 6.3 hierarchy cut invariant

valid cutは次をすべて満たす。

1. rootから到達可能で、同一Artifact generationに属する。
2. 同じsurface domainでancestorとdescendantを同時選択しない。
3. covered surfaceにgap／duplicateがない。
4. resident pageだけを参照する。
5. 各nodeのobject errorを`VirtualGeometryErrorContractV1`でViewへ投影し、qualified threshold以内にする。
6. material／deformation／render roleの各Feature Requirementに同じTargetの`qualified` Bindingがある。
7. candidate／visible capacity、resource lifetime、bounds invariantに合格する。

より細かいnodeがnonresidentなら、同generationのresident ancestorがerror contractを満たす範囲でcoarser cutを使える。満たせなければouter discrete fallbackへ戻す。missing pageをhole、前generation node、default mesh、無制限同期loadで補わない。

## 7. Resolution Plan、residency、runtime境界

### 7.1 receipt-free Plan

```text
VirtualGeometryResolutionPlanV1
  schema_version: 1
  plan_id: StableId
  plan_version: positive u32
  source_asset_ref: exact AssetSourceRefV1
  artifact_manifest_ref:
    exact VirtualGeometryArtifactManifestRefV1 | null
  target_profile_ref: exact TargetProfileRefV1
  quality_profile_ref: exact QualityProfileRefV1
  outer_representation_ref: exact LodRepresentationRefV1 | null
  lod_resolution_plan_ref: exact LodResolutionPlanRefV1
  error_contract_ref: exact VirtualGeometryErrorContractRefV1 | null
  feature_requirement_refs[0..256]:
    sorted unique exact refs
  material_compatibility_ref:
    exact VirtualGeometryMaterialCompatibilityRefV1 | null
  deformation_compatibility_ref:
    exact VirtualGeometryDeformationCompatibilityRefV1 | null
  residency_contract_ref:
    exact VirtualGeometryResidencyContractRefV1 | null
  discrete_fallback_contract_ref:
    exact VirtualGeometryFallbackContractRefV1 | null
  status:
    requirements_closed | fallback_only | blocked
  unresolved_requirements[0..256]:
    owner_document_id
    requirement_ref
    reason:
      missing | stale | unsupported | incompatible
  plan_hash: SHA-256
```

`requirements_closed`はunresolved requirement 0件、全nullable virtual field non-null、Feature Requirement 1件以上、exact discrete fallbackを必須にするが、Targetでqualifiedまたはactiveであることを意味しない。Activation ResolverはPlan外のexact Qualification Binding集合を検証し、全`feature_requirement_refs[]`へ一対一の`qualified` Bindingがある場合だけTarget Activation Bindingを作る。outer LODは同Bindingがある場合だけvirtual candidateを登録する。

`fallback_only`はexact discrete fallbackを必須、unresolved requirement 1件以上とし、解決できたvirtual fieldだけをnon-nullにする。null fieldごとに対応する一件のtyped unresolved requirementを要求し、virtual candidateを登録しない。`blocked`はfallback refをnull、unresolved requirement 1件以上とし、Package／promotionを拒否する。`requirements_closed`でnull、`fallback_only`でfallback null／unexplained null、`blocked`でfallback non-null、statusと矛盾するunresolved reasonを拒否する。Plan hashはASCII `MIRAKAN_VIRTUAL_GEOMETRY_RESOLUTION_PLAN_V1`と自己hashを除くcanonical bytesから計算し、Qualification Receipt／Binding、Target Activation Binding、runtime residency、View cut、GPU feedback、timestampを含めない。

hash依存順は`Source／semantic intent -> Representation Family -> fallback／Material／deformation／Feature Requirement -> Toolchain／Provider lock -> hierarchy／page Artifact -> outer LOD Plan -> Virtual Geometry Resolution Plan -> Qualification Receipt／Binding -> staged Target Activation Binding -> atomic Product Activation／Binding publication`とする。FamilyはArtifactを参照せず、Fallback ContractはFamilyだけを参照する。後段refを前段preimageへ戻さずcycleを拒否する。

`VirtualGeometryResolutionPlanRefV1`は`plan_id`、`plan_version`、`plan_hash`のexact refである。

`requirements_closed` Planの`feature_requirement_refs[]`はArtifact Manifestに束縛されたRequirement集合、およびMaterial／deformation／covered render roleから到達するRequirement集合とset equalityにする。`fallback_only`はnon-null Artifact／compatibility refから到達可能なRequirementだけを列挙し、未解決集合をtyped unresolved requirementで表す。RuntimeまたはActivation ResolverがRequirementを追加／削除せず、Binding欠損を別tupleのqualified結果で補わない。

```text
VirtualGeometryTargetActivationBindingV1
  schema_version: 1
  activation_binding_id: StableId
  activation_binding_version: positive u32
  future_capability_ref:
    exact future.capability.virtualized-continuous-geometry-lod ref
  target_profile_ref: exact TargetProfileRefV1
  resolution_plan_ref:
    exact VirtualGeometryResolutionPlanRefV1
  feature_qualification_binding_refs[1..256]:
    sorted unique exact VirtualGeometryFeatureQualificationBindingRefV1
  runtime_asset_authority_ref:
    exact approved／applied owner／port ref
  renderer_capability_signature_ref: exact owner-typed ref
  package_dependency_closure_ref: exact owner-typed ref
  discrete_fallback_contract_ref:
    exact VirtualGeometryFallbackContractRefV1
  activation_receipt_ref: exact owner-typed ref
  activation_binding_hash: SHA-256
```

Activation Bindingは`status=requirements_closed`のPlanだけを参照し、全Feature Requirementへexact一件の`qualified` Bindingを持ち、extra／missing／duplicate tupleを拒否する。`runtime_asset_authority_ref`のOwner Decisionが`applied`、Renderer signatureとPackage closureが同じTarget、fallbackがqualifiedかつreadyである場合にだけnon-dispatchableなstaged candidateを作れる。Product Capabilityの`active`移行とTarget Activation Bindingのpublicationは同一Governance transactionでatomicに成立させ、Product activeをcandidate作成の事前条件にするcycleも、Binding publicationをProduct activeより先にする隙間も作らない。BindingはPlan hashへ戻さず、revoke後は新規View／Graph generationでvirtual candidateを登録しない。`VirtualGeometryTargetActivationBindingRefV1`はactivation binding ID、version、Target ref、hashを持つ。current staged／published Binding集合はともに`[]`である。

### 7.2 Residency CoordinatorとRuntime Asset authority

virtual geometry固有のtarget `VirtualGeometryResidencyContractV1`は本書が所有し、次を固定する。

```text
VirtualGeometryResidencyContractV1
  schema_version: 1
  residency_contract_id: StableId
  residency_contract_version: positive u32
  runtime_asset_authority_ref:
    exact approved owner／port ref
  request_input_kinds:
    recovery_root | world_prefetch_hint | render_detail_hint
  root_policy:
    pinned_while_outer_representation_active
  request_merge_key:
    artifact_manifest_ref + artifact_generation + page_id
  acceptance_contract_ref:
    exact Scheduling／Lifetime generation acceptance ref
  lease_contract_ref:
    exact Memory／Pointers immutable generation lease ref
  capacity_contract_ref:
    exact Performance／Capacity profile ref
  non_root_eviction_eligibility:
    no_active_cut_lease_and_not_pending_acceptance
  failure_policy:
    resident_ancestor_then_discrete_fallback
  snapshot_publication_contract_ref:
    exact immutable publication boundary ref
  residency_contract_hash: SHA-256
```

Coordinatorは三入力をowner-qualified priority intentとして受け、同じArtifact generation内でcanonical mergeする。`recovery_root`はroot-first再構築、`world_prefetch_hint`はactivation prerequisiteでない予測、`render_detail_hint`はView cutから得たPresentation hintである。priority、completion時刻またはGPU feedbackをGameplay順へ変換しない。root setはouter virtual representationがactiveな間eviction不適格、non-root pageはactive cut leaseとpending acceptanceがともにない場合だけ候補になり、実際のcapacity判定はPerformance Ownerを消費する。

`VirtualGeometryResidencyContractRefV1`は`residency_contract_id`、version、`residency_contract_hash`のexact refである。

generic Artifact request／read／decode／upload／residencyのinitial V1 Ownerは[Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md)へ一意化する。本書はその汎用authorityをvirtual geometry用に横取りせず、`runtime_asset_authority_ref`が同Ownerのexact Definition refへ解決し、Runtime Asset Capabilityが対象TargetでactiveになるまでPlanを`fallback_only`、active Targetを`[]`にする。Architecture上のOwner closureは実装、Definition materialization、Capability activationを意味しないため、discrete LOD／HLOD fallbackは維持する。

### 7.3 runtime residency snapshot

```text
VirtualGeometryResidencySnapshotV1
  schema_version: 1
  snapshot_generation: positive u64
  artifact_manifest_ref: exact VirtualGeometryArtifactManifestRefV1
  artifact_generation: positive u64
  root_state: ready | pending | failed | stale
  resident_page_set_digest: SHA-256
  pending_page_set_digest: SHA-256
  failed_page_set_digest: SHA-256
  pool_generation: positive u64
  pressure_snapshot_ref:
    exact LodBudgetPressureSnapshotRefV1
  completeness:
    complete | bounded_partial | unavailable
  gap_reason:
    none | telemetry_delayed | provider_fault | device_fault
  snapshot_hash: SHA-256
```

これはimmutable read projectionでありraw GPU address、descriptor index、native handle、page payload pointer、request queue pointerを公開しない。page集合のfull listはruntime selection inputの内部に留め、Debug／AIにはdigest、count、bounded sample、gapだけを出す。Snapshotがstale、Target／generation不一致、root非readyならvirtual pathを選択しない。

WorldはCell／representation slotへ`prefetch_role: required_root | likely_detail | opportunistic`とowner-qualified priority intentを関連付けられるが、page ID、pool slot、resident bitをWorld Planへ持たない。Render Graphはpage request hintを発行できるがI/O順、eviction、shared budgetを決定しない。

`VirtualGeometryResidencySnapshotRefV1`は`snapshot_generation`、exact Artifact Manifest ref、`artifact_generation`、`snapshot_hash`を持つ。同じgeneration番号でもArtifact ref／hashが異なるSnapshotを相互代用しない。

### 7.4 View cut summary

```text
VirtualGeometryViewCutSummaryV1
  schema_version: 1
  render_view_id: StableId
  view_generation: positive u64
  view_lod_context_hash: SHA-256
  resolution_plan_ref: exact VirtualGeometryResolutionPlanRefV1
  residency_snapshot_ref: exact VirtualGeometryResidencySnapshotRefV1
  selected_micro_cluster_count: u32
  maximum_projected_error_q16: u32 | null
  fallback_state:
    none | resident_ancestor | discrete_representation
  fallback_reason:
    none | feature_unqualified | root_unavailable | page_unavailable
    | error_bound_exceeded | capacity_exceeded | material_incompatible
    | deformation_incompatible | stale_generation | provider_fault
  request_hint_digest: SHA-256 | null
  completeness:
    complete | bounded_partial | unavailable
  summary_hash: SHA-256
```

SummaryはDebug／qualification用のbounded projectionでありdraw commandまたはSave stateではない。`maximum_projected_error_q16=null`は`fallback_state=discrete_representation`または`completeness=unavailable`だけに許し、0で不明を表さない。

`VirtualGeometryViewCutSummaryRefV1`は`render_view_id`、`view_generation`、`view_lod_context_hash`、`resolution_plan_ref`、`summary_hash`を持つ。別View、camera cut前generationまたは別Context hashのSummaryをhistoryとして再利用しない。

## 8. fallback、fault、device recovery

```text
VirtualGeometryFallbackContractV1
  schema_version: 1
  fallback_contract_id: StableId
  fallback_contract_version: positive u32
  source_asset_ref: exact AssetSourceRefV1
  virtual_representation_family_ref:
    exact VirtualGeometryRepresentationFamilyRefV1
  discrete_representation_set_ref:
    exact LodRepresentationSetRefV1
  required_semantic_cue_refs[0..64]:
    sorted unique exact owner-typed refs
  allowed_visual_difference_refs[0..64]:
    sorted unique exact owner-typed refs
  covered_render_roles[1..16]:
    sorted unique closed render roles
  target_profile_ref: exact TargetProfileRefV1
  qualification_fixture_ref: exact owner-typed ref
  fallback_contract_hash: SHA-256
```

`VirtualGeometryFallbackContractRefV1`はfallback contract ID、version、hashのexact refである。fallback Setは同じSource Asset semanticsへ解決し、covered render roleをすべて持つ。role欠損をdefault Material、hidden presentationまたは別Sourceで補わない。

| Condition | 必須結果 |
|---|---|
| Capability `planning_only`／Target entryなし | virtual candidateを登録せずdiscrete chainを使う |
| Source featureのBindingが`unsupported`／`not_evaluated`／`provisional`または欠損 | Planを変更せずvirtual candidateを非登録、exact discrete fallbackと理由をPreview／Diagnosticへ出す |
| discrete fallback欠損／stale／別Source | Cook／promotion拒否。virtual-only packageを作らない |
| root residency setがpending／failed／stale | virtual path非選択、discrete fallback |
| non-root page missing | error内ならresident ancestor、超えるならdiscrete fallback |
| hierarchy／page generation mismatch | 全virtual candidate拒否、generationを混在させない |
| Material／deformation interface mismatch | 該当passをdefaultへ置換せずdiscrete fallback |
| candidate／visible capacity超過 | current Graph generationを不成立にし、partial draw／same-frame fallbackを挿入しない。approved discrete fallbackを次Graph generationで選びDiagnosticを出す |
| pool thrash／I/O backpressure | semantic floor内でcoarser resident cut、不能ならdiscrete fallback |
| provider／decode／integrity fault | failed pageを隔離しlast-valid generationまたはdiscrete fallback |
| device loss | runtime residency／feedbackを破棄し、immutable Artifactからroot-firstで再構築。Gameplay stateは変更しない |

fallbackでMaterial、lighting、shadow、deformationの完全なpixel equalityを常に保証するとは主張しない。`VirtualGeometryFallbackContractV1`は許容されるvisual差、必ず保持するsemantic cue、対象pass、Target、fixtureをexactに宣言し、Content Quality Receiptで判定する。

## 9. AI／Editorの理解境界

### 9.1 bounded read model

target `VirtualGeometryAuthoringContextV1`は次を最大256件ずつのbounded collectionで返す。

- Capability／Target／Feature Requirement／Qualification Bindingのexact state
- Source Asset、Authoring Intent、Artifact generation
- outer representationとdiscrete fallback chain
- Material／deformation compatibility
- root／page residencyのaggregate、pressure class、last Diagnostic
- View別cut summaryのbounded sample
- unresolved requirementとOwner
- relevant Source／Plan／Receipt／Diagnostic ref
- omitted count、truncated flag、stale／gap
- current callable Operation refs

AIは本文全文、GPU buffer dump、全page list、全View historyを既定contextにしない。Contextにないfeature、Target、threshold、provider、fallbackを補完せず、`unsupported | not_evaluated | provisional | qualified`と`unknown | omitted | stale | not_activated`を区別して説明する。

### 9.2 Preview

target `VirtualGeometryPlanPreviewV1`は少なくとも次を表示する。

- 何を守るか: silhouette、seam、attribute、material、deformation、critical cue
- 何が変わるか: outer representation、Target candidate、Artifact closure
- 何が変わらないか: Source最高detail、Collision、Physics、Navigation、Entity identity、Save
- RequirementとQualification Bindingを重ねたqualified／unqualified feature matrix
- exact fallback pathとvisual差契約
- pool／I/O／candidate／visible capacityの測定状態
- unresolved Decision、必要Evidence、Owner
- Source変更、derived-only再Cook、Project revision変更の区別

「NaniteをON」「自動最適化」「高品質」といった一つの曖昧bool／commandへ畳み込まない。AIは「巨大な彫刻を近接でも輪郭を保つ」「葉の抜けを維持する」「顔と手の誤差を厳しくする」等のsemantic intentへ解決し、provider／page size／cluster builderを作品上の選択肢として露出しない。

### 9.3 current operation closure

planned semantic action vocabularyは`inspect intent／feature matrix／plan／artifact／residency／cut summary／diagnostic`、`propose semantic intent`、`preview fallback／quality／capacity impact`、`validate source／plan／artifact／Target`である。これらはStable Operation IDではない。

current MCD／Owner Manifest／Service allowlist／Policy／Validator／Receipt／Provider／MCP／alias Operation集合はすべてexact `[]`、Capabilityは`planning_only`である。将来のactivation ChangeSetが採用するexact Operation集合と完全closureを一transactionで登録するまでdispatchせず、action名、Editor command、外部Engine機能名からIDを生成しない。

## 10. Diagnosticとqualification

### 10.1 closed Diagnostic ID

固有Diagnostic IDを次に固定する。

```text
MIRAKAN-VG-CAPABILITY_NOT_ACTIVATED
MIRAKAN-VG-SCHEMA_UNKNOWN
MIRAKAN-VG-TARGET_UNQUALIFIED
MIRAKAN-VG-FEATURE_NOT_QUALIFIED
MIRAKAN-VG-FALLBACK_MISSING
MIRAKAN-VG-ROOT_NOT_RESIDENT
MIRAKAN-VG-PAGE_UNAVAILABLE
MIRAKAN-VG-PAGE_INTEGRITY
MIRAKAN-VG-GENERATION_MIX
MIRAKAN-VG-ERROR_BOUND_EXCEEDED
MIRAKAN-VG-MATERIAL_INCOMPATIBLE
MIRAKAN-VG-DEFORMATION_INCOMPATIBLE
MIRAKAN-VG-RESIDENCY_THRASH
MIRAKAN-VG-CANDIDATE_CAPACITY_EXCEEDED
MIRAKAN-VG-VISIBLE_CAPACITY_EXCEEDED
MIRAKAN-VG-PROVIDER_FAULT
MIRAKAN-VG-DEBUG_PROJECTION_INCOMPLETE
```

各DiagnosticはSource／Artifact／Plan／Target／View ref、feature tuple、expected／observed generation、fallback結果、last-valid維持、remediation、completeness／gapを持つ。Backend result codeを公開IDへ昇格せずprivate cause detailとして束ねる。

### 10.2 qualification fixture

Target別qualificationは少なくとも次を含む。

- 同じSource／Intent／Toolchain／Targetからhierarchy、page set、root set、Manifest hashが二回一致する。
- hierarchy cutのcoverage、ancestor／descendant排他、bounds、error monotonicity、page dependency、generationをpositive／negative fixtureで検証する。
- silhouette、UV／material seam、normal／tangent、vertex color、attribute、protected regionのerrorをcontent rubricで検証する。
- static、skeletal、morph、world-position deformation、spline、displacement、opaque、masked、translucent、shadow、reflectionをfeature tupleごとに独立判定する。
- Camera FOV、orthographic／perspective、dynamic resolution、near-plane、camera cut、teleport、split view、Editor／reflection／shadow Viewでprojected metricとView-local historyを検証する。
- root missing、non-root page loss、corruption、stale generation、promotion同時発生、rapid movement、pool pressure、I/O delay、candidate／visible overflow、provider fault、device lossを注入する。
- HLOD World aggregationとmicro-cluster hierarchyが別identity／ownerで、Cell／page／cluster IDを相互代用しない。
- virtual path on／off、fallback、device recoveryでCollision、Physics、Navigation、Damage、Animation event／root motion、Entity identity、Save／Replay authoritative digestが一致する。
- unsupported TargetのPackageがvirtual-onlyにならず、exact discrete fallbackから表示できる。
- bounded AI Contextがunsupported、not-evaluated、provisional、qualified、omitted、stale、gapを正しく区別し、Operation `[]`をdispatch可能と誤読しない。

performance指標はpool resident bytes、root bytes、page request／read／decode／upload量、request miss、resident ancestor fallback、discrete fallback、thrash、candidate／visible count、cut traversal／raster時間、hitchを区別する。数値threshold、page size、cluster size、compression ratio、supported scene scaleはすべて`not_measured`であり、[Performance／Capacity](../04-runtime/performance-capacity.md)のTarget fixtureとReceiptなしに固定値または達成値を記載しない。

## 11. 未決定事項を曖昧なままにしない

未決定事項は自由文の`TBD`／`TODO`で残さず、target `VirtualGeometryDecisionV1`へ登録する。

```text
VirtualGeometryDecisionV1
  schema_version: 1
  decision_id: StableId
  decision_version: positive u32
  owner_document_id
  state: not_evaluated | rejected | approved
  candidate_refs[1..32]: sorted unique exact refs
  required_evidence_refs[1..64]: sorted unique exact refs
  fallback_contract_ref: exact ref
  selected_candidate_ref: exact ref | null
  rationale_ref: exact Evidence-backed ref | null
  decision_content_hash: SHA-256
```

`state=not_evaluated`はselected／rationaleをnull、`approved | rejected`はDecision Receiptを必須とする。candidateが一つでもEvidenceなしに自動承認しない。

| Decision | current state | candidate scope | mandatory fallback |
|---|---|---|---|
| provider／private Adapter | `not_evaluated` | in-house／qualified third-party | discrete LOD／HLOD |
| generic Runtime Asset request／residency authority | `owner_selected_target`（`ARCH-C02 closed-in-target-design`、Capability `not_activated`） | [Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md)のDefinition／Port materializationとTarget qualification | virtual candidateを登録せずdiscrete LOD／HLOD |
| hierarchy builder／simplification algorithm | `not_evaluated` | Toolchain lockへ登録する候補だけ | Source保存＋discrete chain |
| micro-cluster granularity | `not_evaluated` | Target measurementで比較するbounded profile | provider-neutral intent |
| page granularity／layout | `not_evaluated` | I/O、memory、integrity fixtureで比較 | root＋discrete fallback |
| compression／decode | `not_evaluated` | exact version／license／security evidence付き候補 | uncompressed test artifactまたはdiscrete path |
| static／skeletal／foliage／terrain feature | `not_evaluated` | feature tupleごとの独立entry | featureごとのdiscrete representation |
| Windows／mobile／console activation | `not_evaluated` | exact Target Profileごと | Targetのdiscrete renderer |
| pool／queue／candidate／visible capacity値 | `not_evaluated` | Target measurement profile | hard bounded fallback |

これは「詳細が未定」という曖昧さではなく、現在選ばれていない対象、必要Evidence、失敗時の安全経路を明示した状態である。`approved`になるまでSource Schema、Product claim、Target support matrix、Shipping packageへ候補名を有効値として露出しない。

## 12. 他計画書との整合条件

1. Product PlanのFuture Ownerは本書とし、activation triggerはLOD、Asset、Runtime Package、Scheduling、Memory、World、Camera、Render Graph、Performance、Materials、Animation、Persistence、Debug、Toolchain／Platformのexact closureと`ARCH-C02`のapplied Owner Decisionを要求する。
2. LODはouter representationとfallbackだけを所有し、inner cutをtier、hysteresis、Save projectionへ追加しない。
3. Asset LifecycleはArtifactを所有するが、View metric、pool pressure、Render passを持たない。
4. Runtime PackageはArtifact dependencyを列挙するが、page resident stateをPackageへ焼き込まない。
5. Worldはprefetch intentを宣言できるが、page ID、micro-cluster、pool slotを持たない。
6. Render Graphの`RenderViewV1`と、そこからLOD Ownerがbyte-exactに投影した`ViewLodContextV1`だけをView入力とし、virtual path用Cameraを作らない。
7. Materials／Animationはfeature compatibilityとsemantic invariantを所有し、Geometry側がshader／deformation意味を推測しない。
8. Render Graphはresident snapshotからcut／drawを実行するが、Asset generation、I/O／eviction policy、shared budgetを所有しない。
9. Performanceはcapacityとpressureを所有するが、critical cue、silhouette、fallback semantic floorを変更しない。
10. Persistenceはvirtual page、resident set、cut、GPU feedback、Presentation historyをSave／authoritative Replayから除外する。
11. Debuggingはbounded projectionだけを保持し、raw page／GPU dumpをAI既定contextまたはauthoritative replayへ入れない。
12. Collision／Physics／Navigationはrender virtual geometryをsource、fallback、reduced simulation candidateへ使わない。

この条件と矛盾するfield追加は本書だけで承認せず、該当OwnerのCompatibility ChangeとConsumer Inventoryを必要とする。

## 13. 非目的

本書はvirtualized geometryの実装、prototype、provider選定、algorithm選定、Schema materialization、Operation activation、Task Plan、Work Package、実装順序、工数、日程、担当、依存導入またはmigration実行を指示しない。

また、次を採用しない。

- Nanite互換format／API／名称を公開Contractにする。
- `supports_virtual_geometry: bool`一つでTarget、Material、deformation、passの可否を代表する。
- virtual-only Assetを作り、fallbackをruntimeのguessに委ねる。
- page ID／micro-cluster ID／GPU addressをWorld、Save、AI Source、Project C++へ公開する。
- inner cutをLOD tier、Entity state、network state、authoritative Replayへ保存する。
- GPU-driven、meshlet、indirect draw、bindless、resident resourceの存在だけから本Capabilityを成立と推測する。
- pressure時にcritical cue、Collision、Navigation、Damage、Animation eventを削る。
- 外部Engineの既定値、performance claim、対応platformをMiraikanaiのqualified結果として流用する。
