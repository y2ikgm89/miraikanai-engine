# Miraikanai Engine 独自Navigation Platformアーキテクチャ規約

- 文書版: 1.0
- 作成日: 2026-07-20
- 最終更新日: 2026-07-20
- 対象: 2D Grid Navigation、3D Navmesh build／query、Backend、Derived Asset、Editor、AI Authoring、Qualification
- 状態: プロジェクト公式の規範設計レビュー版。C1 Backendは採用決定済み、Production昇格は実装後のQualification待ち
- 上位文書: [AIネイティブ独自ゲームエンジン 設計計画書](./2026-07-18-ai-native-game-engine-authoring-design.md)
- Runtime正本: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- Simulation連携: [Miraikanai Engine Physics／Navigation／Animation連携規約](./2026-07-19-physics-navigation-animation-architecture-design.md)
- 機能範囲: [Miraikanai Engine 2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)
- Asset正本: [Miraikanai Engine Asset Pipeline／Content Package規約](./2026-07-19-asset-pipeline-content-packaging-design.md)
- 基盤規約: [Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)
- 実行可能契約: [Miraikanai Engine 実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)

## 1. 結論

Miraikanai EngineのNavigationは、**公開契約、正規data、build／query orchestration、lifetime、failure、Editor、AI CapabilityをC++23で独自実装し、3D Navmeshのpolygon生成と経路探索kernelだけを交換可能な非公開Backendへ隔離する**方式に固定する。

- 2D C1はEngine-owned Grid NavigationとA*を実装する。
- 3D C1の基準BackendはRecast Navigation／Detour 1.6.0、commit `6dc1667f580357e8a2154c28b7867bea7e8ad3a7`である。
- RecastはEditor／cook時のNavmesh生成、DetourはRuntimeのpoint projection、polygon corridor、straight path計算に使用する。
- `NavAgentProfile`、`NavAreaProfile`、`NavBuildProfile`、`CookedNavWorld`、request／result、status、version、budget、Editor／AI OperationはMiraikanai Engineが所有する。
- Project C++、GameplayDefinition、AI、Editor、Saveへ`rcConfig`、`dtNavMesh`、`dtNavMeshQuery`、`dtPolyRef`、`dtStatus`、tile binary、Vendor allocator／callbackを公開しない。
- C1では`DT_POLYREF64`を定義せず、標準32-bit `dtPolyRef`をprivate Backend内だけで使用する。
- 完全自作3D BackendはC3 Researchであり、同じ公開契約とQualification fixtureを使って基準Backendを上回る証拠が得られた場合だけ別ADRで昇格を検討する。
- Recast／Detourを永久に公開契約へ固定しない。Backendを交換してもProject data、AI Tool、Gameplay API、Save形式を変更しない。

本書でいう「独自Navigation Platform」はNavmesh polygon生成アルゴリズムまで全自作する意味ではない。Engineが製品上の意味、制約、状態、操作、検証、互換性を所有し、外部kernelを交換可能な実装詳細にすることを意味する。

## 2. 決定権と境界

| 主題 | 正本 |
|---|---|
| 2D Grid、3D Navmesh、Profile、Backend Port、Artifact、Query、Status、Editor／AI、Qualification | 本書 |
| Tick phase、queue、lease、promotion、global memory、message header | Runtime規約 |
| Static geometry、Collider source revision、Physics obstacle snapshot | Collision／Physics規約とSimulation連携規約 |
| Source／Derived Asset、ArtifactKey、Package、SBOM、atomic promotion | Asset規約 |
| C1／C2製品機能、Reference Scene、成熟度 | 2D／3D機能計画 |
| C++ ownership、pointer、dependency lock、Build Driver、directory上位規則 | 基盤規約 |
| MCD、Schema、Provider／MCP projection、Codegen | 実行可能契約規約 |

本書はRuntime規約のNavigation Domain 64 MiB、request／result各4,096件／tick、T00 promotion、T20 result integrationを緩和しない。Large World、floating origin、network lockstep、GPU pathfinding、任意Runtime Navmesh生成はC1／C2に暗黙包含しない。

## 3. 採用判断

### 3.1 比較した三方式

| 方式 | 長所 | 重大な問題 | 判断 |
|---|---|---|---|
| Navmesh生成／queryまで完全自作 | data layout、特殊移動、決定性、専用最適化を完全所有 | C1だけで18～30人月、Production相当は累計50～90人月を見込む。geometry耐性、tile seam、ID寿命、部分再生成、toolの検証が最初の3D Playableを遅らせる | C1／C2では不採用 |
| Recast／Detour APIをGame／AIへ直接公開 | 初期Adapterが短い | Project data、Save、AI、Editor、version更新がVendor型へ固定され、bounded failureと安全な再検証境界を失う | 不採用 |
| 独自契約＋交換可能なprivate Backend | 製品の意味とAI向け語彙を所有しつつ成熟kernelを利用でき、将来自作Backendも同じ契約で比較できる | Adapter conformance、Artifact envelope、Backend Qualificationが必要 | **採用** |

AIの理解しやすさはpolygon生成の所有者ではなく、公開語彙、単位、範囲、closed status、状態遷移、preview、failureが一意かで決まる。AIはMiraikanaiの正規契約だけを扱い、Vendor object graphやstatus bitを推論しない。

### 3.2 公式資料から確認した実例

| Engine | 公式資料で確認した構成 | 本計画への意味 |
|---|---|---|
| Unreal Engine 5.8 | Engine-owned Navigation System内に`ARecastNavMesh`、tile、area cost、dynamic generation、Detour Crowdを統合 | 独自Product APIとRecast系kernelの統合は実運用されている |
| Godot 4 | Scene geometryを第三者Recastへ渡して3D Navmeshをbakeし、EngineのNavigation APIで利用 | Open source Engineでもbake kernelとProduct APIを分離している |
| Open 3D Engine | Recast Navigation Gemがbuild、path query、可視化を提供 | Recastを機能単位のBackendとして隔離できる |
| CRYENGINE 5 | 独自Multi-layer Navigation Meshをvoxel生成、A*、Runtime再生成まで所有 | 完全自作は可能だがNavigation専用製品投資を必要とする |

これらは採用方式の規模比較であり、他EngineのComponent名、Scene形式、既定値、Editor UXをMiraikanaiへコピーする根拠にしない。

### 3.3 開発工数と暦期間

見積りはNavigation経験を持つC++ Engine Engineerが専任し、1人月を実働20日として、設計、実装、code review、test、CI、文書化を含めた2026-07-20時点のplanning rangeである。採用済みRenderer、Asset、Collision、Runtime、Editor、MCD基盤が利用可能という前提であり、それらの新規開発工数は含めない。採用後N2で実測し、見積り中央値から30%以上外れた場合はDecision Ledgerを更新して再承認する。

| 3D Navigation作業 | 完全自作C1 人月 | 独自契約＋Recast／Detour C1 人月 |
|---|---:|---:|
| geometry canonicalize、voxel／walkable build | 4～6 | 1～2 |
| region、contour、polygon、detail、tile seam | 5～8 | 1～2 |
| polygon ref、A*、filter、projection、straight path、link | 3～5 | 1～2 |
| Artifact、version、lease、threading、failure正規化 | 2～4 | 2～3 |
| Editor debug、diagnostic、AI Operation | 2～3 | 1～2 |
| adversarial fixture、benchmark、Target Qualification | 2～4 | 1～2 |
| **C1合計** | **18～30** | **7～13** |

完全自作Production相当は、C1にdynamic obstacle／partial rebuild 8～14人月、streaming／hierarchical query 6～10人月、crowd／local avoidance 5～10人月、Editor／diagnostic深化 4～8人月、multi-Target robustnessと長期fuzz／性能改善 9～18人月を加え、**累計50～90人月**とする。推奨方式のC2累計はBackend Adapterと同じ製品機能を含めて**22～40人月**をplanning rangeとする。

| 専任人数 | 完全自作C1の暦期間 | 完全自作Production累計 | 推奨方式C1の暦期間 |
|---:|---:|---:|---:|
| 1人 | 18～30か月 | 50～90か月 | 7～13か月 |
| 2人 | 10～17か月 | 28～50か月 | 4～8か月 |
| 3人 | 7～12か月 | 19～35か月 | 3～6か月 |
| 5人 | 非推奨。C1範囲では調整costが支配 | 13～25か月 | 非推奨。C1範囲では調整costが支配 |

暦期間は単純な人月割りではなく、geometry→mesh→queryの直列依存、fixture作成、統合待ちを含む。ここでいう完全自作は第三者のpolygon生成／pathfinding kernelを利用しない場合であり、一般的なcontainer、math、test frameworkまで再実装する意味ではない。未知のgeometry不具合、Target差、Editor品質を見積り下限で計画せず、staffingとRelease計画には上限を使う。

## 4. 成熟度と対象範囲

| Level | Navigation到達点 |
|---|---|
| C0 Foundation | MCD、Profile、Validator、Backend Port、Artifact envelope、Diagnostic、Qualification harness |
| C1 First Playable | 2D Grid A*、3D offline tiled Navmesh、projection、path／straight path、area cost、typed off-mesh link、async query、Editor／AI、Windows Reference合格 |
| C2 Production | DetourTileCache相当のdynamic obstacle、partial rebuild、streaming tile、hierarchical query、crowd／local avoidance、Target別tuning |
| C3 Research | 完全自作3D Backend、Large World、GPU／massive-agent pathfinding、network lockstep、飛行／壁面／volume navigation、任意Runtime生成 |

C1／C2 SchemaはC3 fieldを受理しない。Recast／Detourに機能が存在しても、Capability、budget、failure、fixtureが本書へ追加されるまで公開しない。

## 5. 独自Navigation Platformの構成

### 5.1 Layer

```text
AI／Editor／GameplayDefinition／Project C++
  -> Miraikanai Navigation Contract
  -> Navigation Core
       build orchestration
       query validation／normalization
       version／lease／budget
       artifact／diagnostic
  -> NavBuildBackendV1／NavQueryBackendV1
  -> Grid2D Backend | Recast／Detour Backend | future qualified Backend
```

上位層はBackend名で処理を分岐しない。Backend能力差は`NavigationBackendCapabilityV1`へ閉じ、Projectが要求するCapabilityを満たさないBackendではcookを失敗させる。AIはBackendを直接選択しない。

### 5.2 正規object

| Object | 所有者 | 永続化 |
|---|---|---|
| `NavigationWorldDocumentV1` | Authoring Model | する |
| `GridNav2DProfileV1`／`NavAgentProfileV1`／`NavBuildProfile3DV1` | Project Profile | する |
| `NavSourceSetV1`／`NavAreaProfileV1`／`NavModifierV1`／`NavOffMeshLinkV1` | Authoring Model | する |
| `CookedNavWorldV1` | Asset Pipeline | Derived Assetとしてする |
| `NavWorldHandle`／`NavQueryHandle` | Runtime Navigation | しない |
| `NavMeshVersion` | Runtime Navigation | Runtime generationだけ |
| `NavQueryRequestV1`／`NavQueryResultV1` | Runtime message | tick／deadline内だけ |
| Grid cell payload／Recast／Detour object／polygon ref | Backend | 公開・Saveしない |
| `NavigationBuildReceiptV1`／`NavigationQualificationReceiptV1` | Verification | する |

Runtime handleは`{index32, generation32}`である。`dtPolyRef`をこのbitへ詰め替えず、query-local corridorからEngine-owned path point／stable link actionへ変換後に破棄する。

`NavMeshVersion`は`{nav_world_id: StableId, generation: uint64, artifact_sha256: 32 byte}`である。`StableId`は基盤規約どおりRFC 9562 UUIDv7の128 bit値とする。generationは最初のpublishを1とし、同じ`nav_world_id`内でT00 promotionごとに1増加して再利用しない。2D Gridと3D Navmeshで同じversion型を使うが、`nav_world_id`とdimensionが異なるAssetを比較可能扱いにしない。

### 5.3 ModuleとDirectory

```text
engine/navigation/
  contracts/
  core/
    build/
    query/
    runtime/
    artifacts/
  grid2d/
  diagnostics/
  backends/
    recast/
authoring/navigation/
editor/panels/navigation/
tools/navigation_qualification/
```

- `contracts`はMCD生成value type、Port、request／result、statusだけを公開する。
- `core`はBackend非依存のvalidation、build orchestration、version、lease、budget、status normalizationを所有する。
- `grid2d`はC1のGrid cookとA*を所有し、Recastへ依存しない。
- `backends/recast`だけがRecast／Detour headerをincludeし、Vendor static libraryへlinkする。
- `authoring/navigation`はSource document、semantic／budget validator、cost preview、ChangeSet projectionを所有する。
- `editor/panels/navigation`は正規documentとimmutable debug snapshotのProjectionであり、Navmeshの正本ではない。
- `tools/navigation_qualification`はShipping Gameへlinkせず、Backend比較、fixture、benchmarkを実行する。

Project C++は生成された`NavigationQueryPortV1`、`NavigationResultViewV1`、`NavigationCommandWriterV1`だけを利用する。`mira.navigation.backend.recast`という公開Moduleを作らない。

## 6. Backend lock、Build、交換条件

### 6.1 `NavigationBackendLockV1`

C1のlock entryを次に固定する。

| Field | 値 |
|---|---|
| `backend_id` | `mira.nav.recast_detour` |
| `source_version` | `1.6.0` |
| `source_commit` | `6dc1667f580357e8a2154c28b7867bea7e8ad3a7` |
| `license` | zlib |
| `poly_ref_bits` | 32 |
| `dt_navmesh_version` | 7 |
| `max_vertices_per_polygon` | 6 |
| C1 modules | Recast、Detour |
| C2候補 modules | DetourTileCache、DetourCrowd |
| 非Product modules | RecastDemo、DebugUtilsのsample renderer、Tests executable |

Source archive SHA-512、patch hash、compiler full version、compile definition、CRT、allocator hook、license hashを`toolchain.lock.json`、SBOM、Build Receiptへ記録する。`DT_POLYREF64`は未定義に固定する。32-bitと64-bitでbuildしたtileは互換とみなさない。

Recast／Detour sourceを変更する場合は、変更file、理由、upstream issue／commit、behavior差、fixtureをpatch manifestへ記録する。警告抑制やnamespace変更も無記録patchにしない。

### 6.2 Product build

- Recast／Detourはprivate static libraryとしてBuildし、Public Header／Module interfaceへ伝播させない。
- Engine allocatorはprocess初期化時に一度だけ登録し、Navigation job実行中に差し替えない。
- allocationはTool cook中はAsset build、Runtime query objectはNavigation Domainへchargeする。
- exception、RTTI、assert、floating-point、sanitizer、IPO、CRTはTarget Build Profileへ明記し、Vendor defaultへ委ねない。
- Vendor logはbounded `NavigationDiagnosticSink`へ変換し、geometry内容、path、Project secretを通常telemetryへ送らない。
- C1 ShippingはRecastDemo、sample input、sample UI、sample serializerを含めない。
- zlib noticeをSource distributionとThird-party noticesへ保持する。

### 6.3 Backend Port

| Port | 入力 | 出力 | 禁止 |
|---|---|---|---|
| `NavBuildBackendV1` | canonical triangle stream、area、link、validated Profile、bounded allocator | private tile payload、build metrics、diagnostic | Authoring object pointer保持、partial live publish |
| `NavQueryBackendV1` | immutable Backend world、projected endpoint候補、filter、hard cap | private corridor、point、raw backend status、metrics | Engine Entity／Physics pointer参照 |
| `NavDynamicUpdateBackendV1` | 前tick obstacle snapshot、bounded region、version | staging update | 同tickPhysics参照、World Transform write |

BackendはEngine statusを直接決めない。Coreがraw result、request policy、version、deadline、ownerを検査して正規statusへ変換する。

## 7. 座標、単位、数値規則

- 3Dは右手系、+Y up、meter、radianである。Recast／Detour Adapterでも軸交換、単位scale、handedness変換を行わない。
- Position各軸はfinite binary32かつ`[-10,000, 10,000]` mである。C1の10 km hard spatial rangeを超えるSourceはLarge World Capabilityなしに拒否する。
- Source transformはcook前にworld meterへbakeする。non-uniform scale、negative determinant、mirrored windingはcanonical triangleへ変換し、変換後のwindingとnormalを再検証する。
- 3D triangleは3つの異なるindex、finite position、面積`>= 1e-10 m²`を必要とする。違反triangleを黙って除外せず、Source Asset、primitive、triangle index付きcook errorにする。
- `NavAgentProfile`のheight、radius、climbはmeter、slopeはdegreeで保存する。Runtimeで別単位へ暗黙変換しない。
- world値からvoxel値への変換は`walkableHeight=ceil(height/ch)`、`walkableRadius=ceil(radius/cs)`、`walkableClimb=floor(climb/ch)`である。
- `width`、`height`、`tileSize`、`borderSize`、`walkable*`、`maxEdgeLen`はvoxel、`cs`、`ch`、bounds、detail sample値はworld meterとして`rcConfig`へ渡す。
- NaN、Inf、negative zeroをcanonical dataへ保存しない。`-0.0`は`+0.0`へ正規化する。
- Recast／Detourのfloat演算をcross-platform network lockstepの根拠にしない。同一source、target、backend lock、toolchain、Build Profileでのcook再現性だけを検証する。

## 8. 正規ProfileとSchema

### 8.1 `NavAgentProfileV1`

| Field | `HumanReferenceV1` | Schema範囲／規則 |
|---|---:|---|
| `height_m` | 1.80 | finite `[0.10, 20.0]` |
| `radius_m` | 0.40 | finite `[0.01, 8.0]` |
| `max_climb_m` | 0.40 | finite `[0, height_m]` |
| `max_slope_deg` | 50.0 | finite `[0, 89.0]` |
| `projection_half_extents_m` | `(2.0, 4.0, 2.0)` | 各軸finite `[0.01, 100.0]` |
| `default_area_mask` | area 1～63 enabled | area 0は常にdisabled |
| `query_node_cap` | 4,096 | integer `[64, 65,535]`、Project C1既定は変更不可 |
| `corridor_cap` | 2,048 | integer `[2, 2,048]` |
| `straight_path_cap` | 256 | integer `[2, 256]` |

Profile IDは内容hashではなくStable IDで、内容hashを別に持つ。Profile field変更は同じIDでも全tileを再cookし、`NavMeshVersion`を更新する。

### 8.2 `NavAreaProfileV1`

| Field | 規則 |
|---|---|
| `area_id` | `uint8` 0～63。0=`blocked`、1=`walkable_default` |
| `semantic_tag` | MCD Catalogへ登録済みのTag `StableId`。自由文字列をRuntime比較しない |
| `traversal_multiplier` | sourceはfinite `[0.0625, 64.0]`、既定1.0。cook後はQ16.16 |
| `flags` | `uint16`。Project定義bitはCapability Catalogへ登録 |
| `debug_color_rgba8` | Editor表示だけ。query semanticsへ使用しない |

Q16.16化は`round_ties_to_even(multiplier*65,536)`で、許容integerは4,096～4,194,304である。Detour filterへ渡すときだけ`q16_16/65,536.0f`へ変換し、result costはfinite検査後に同じ丸めでQ16.16へ戻す。overflowは`CostOverflow`である。0または負costをblockedの別表現にしない。blockedはarea 0またはfilter除外で表す。Detourの64 areaと同値変換するが、Detour area IDを正本にしない。

### 8.3 `NavBuildProfile3DV1::HumanReferenceV1`

| Agent／Recast field | C1公式値 |
|---|---:|
| `cs`／`ch` | 0.20 m／0.10 m |
| `walkableHeight`／`walkableRadius`／`walkableClimb` | 18／2／4 voxel |
| `tileSize`／world tile width | 64 voxel／12.80 m |
| `borderSize` | `walkableRadius + 3 = 5` voxel |
| build heightfield width／height | `tileSize + 2*borderSize = 74` voxel |
| partition | Watershed |
| `minRegionArea`／`mergeRegionArea` | 64／400 voxel² |
| `maxEdgeLen`／`maxSimplificationError` | 16 voxel／1.3 voxel |
| `maxVertsPerPoly` | 6 |
| `detailSampleDist`／`detailSampleMaxError` | 1.20 m／0.10 m |
| filter | low-hanging obstacle、ledge span、low-height spanをすべて有効 |
| max total loaded Detour tile slots | 1,024 |
| max polygon／tile | 4,096 |
| max vertex／tile | 65,534。到達前でも4,096 polygon capを優先 |
| max source triangle／build tile | 262,144 |
| max source vertex／build tile | 262,144 |
| max typed off-mesh link／tile | 256 |
| max C2 tile layer／XZ coordinate | 8 |
| max cooked／resident live payload | 2D／3D合計36 MiB |

`max total loaded Detour tile slots`はXZ座標数ではなく全layerを含む`dtNavMesh`のslot総数である。C1はlayer 0だけを生成する。C2 TileCacheを有効化した場合、同一XZ座標の各layerが1 slotを消費する。

標準32-bit `dtPolyRef`は`tileBits=ceil_log2(maxTiles)`、`polyBits=ceil_log2(maxPolys)`、`saltBits=32-tileBits-polyBits`である。1,024 tileと4,096 polygonでは10／12／10 bitとなり、Detour 1.6.0の`min saltBits=10`を満たす。旧値8,192 polygonではsaltが9 bitとなり`DT_FAILURE | DT_INVALID_PARAM`になるため採用しない。

上限超過時は値を切り捨てない。Source geometryをWorld cell／tileへ分割するかProfileを人間が変更し、再cookする。

### 8.4 Custom AgentからのC1 Build値導出

C1で人間またはAIが編集できるのは`NavAgentProfileV1`のworld値だけで、`rcConfig` fieldを直接編集できない。Custom AgentのC1 Build値は次で一意に導出する。

```text
cs = clamp(radius_m / 2, 0.05, 0.50)
ch = cs / 2
walkableHeight = ceil(height_m / ch)
walkableRadius = ceil(radius_m / cs)
walkableClimb = floor(max_climb_m / ch)
tileSize = 64
borderSize = walkableRadius + 3
detailSampleDist = 6 * cs
detailSampleMaxError = ch
```

`walkableHeight`は3～255、`walkableRadius`と`walkableClimb`は0～255、`tileSize+2*borderSize`は1～2,048でなければならない。範囲外は`NAV_PROFILE_VOXEL_RANGE`で拒否し、clampして成功させない。partition、region、simplification、max polygon／tile、ref bitは`HumanReferenceV1`と同じ固定値である。これらを変更するBuild ProfileはC2以降の別Profile ID、fixture、Approvalを必要とする。

## 9. 2D Grid Navigation

`GridNav2DProfile::ReferenceV1`を次に固定する。

| 項目 | C1公式値 |
|---|---:|
| cell | 0.25 m正方形 |
| 接続 | 8近傍。斜めは両隣の直交cellがwalkableの場合だけ許可 |
| cost | 直交65,536、斜め92,682のQ16.16 |
| heuristic | admissible octile distance |
| tie-break | `f`, `h`, canonical cell indexの昇順 |
| query node | 65,536 |
| path cell | 8,192 |
| asset | 最大16,777,216 cell、cell payload 32 MiB、metadata込み34 MiB |
| agent radius | 0.40 m、schema `[0,8]` m、clearance=`ceil(radius/cell_size)` |

cellはrow-major `cell_index=y*width+x`、exactly 2 byteの`uint8 area_id`＋`uint8 clearance_cells`である。2D／3D共通のarea 0はblocked、1～63は64-entry Q16.16 cost tableを参照し、64～255はinvalidとしてcook／loadを拒否する。累積costは`uint64`で、加算前overflowを検査する。

正規statusは3Dと共有するが、2Dの`CostOverflow`は`SearchBudgetExceeded`へ偽装せず`CostOverflow`として返す。TilemapとNavigation cellが一致しない場合は、重なるcollision／blocked sourceを保守的ORで集約する。

## 10. 3D Build、Artifact、Load

### 10.1 Build pipeline

```text
Source collect／canonicalize
-> finite／range／triangle／area／link validation
-> affected tile＋border geometry collection
-> Rasterize heightfield
-> Walkable filters
-> Compact heightfield
-> Erode by agent radius
-> Distance field／Watershed regions
-> Contour
-> Polygon mesh
-> Detail mesh
-> Detour tile data
-> tile seam／capacity／connectivity validation
-> reference query validation
-> CookedNavWorldV1 envelope
-> Artifact Receipt
```

各stageは入力数、出力数、CPU time、peak committed bytes、warning／error countをReceiptへ記録する。中間object pointerやraw geometryをReceiptへ保存しない。

Source geometry、static transform、Collision source revision、Agent Profile hash、Build Profile hash、Backend lock hash、Target、Toolchain hashからArtifactKeyを作る。どれか一つでも変われば再cookする。

### 10.2 `CookedNavWorldV1` envelope

Artifactはraw Detour tile列ではなくEngine-owned envelopeである。

| Field | 規則 |
|---|---|
| magic | ASCII `MNAV` |
| `schema_version` | 1 |
| `endianness` | C1はlittle endian |
| `dimension` | `grid2d \| navmesh3d` |
| `backend_id／version／commit` | lockとexact match |
| `poly_ref_bits` | Recast Backendは32 |
| `backend_payload_version` | Detour C1は`DT_NAVMESH_VERSION=7` |
| `target_profile_id` | cook Targetとexact match |
| `source／agent／build／toolchain_hash` | ArtifactKey入力と一致 |
| `max_tiles／max_polys` | 1,024／4,096 |
| `tile_count` | `[0,1,024]` |
| tile record | coordinate、layer、AABB、payload offset／length、SHA-256 |
| `artifact_sha256` | headerのhash fieldを0扱いにした全Artifact hash |

BinaryはC++ structのmemory dumpにしない。MCD生成descriptorでfield順、width、endianness、alignmentを固定する。Detour tile payloadはenvelope内部のBackend-private sectionとして保持できるが、Project C++、Save、AI、Editor documentへ露出しない。

Loadはenvelope、length／offset、hash、Target、Backend lock、payload version、capacityを全件検査してから`dtNavMesh`を構築する。1 tileでも失敗したArtifactを部分publishせず、旧`NavMeshVersion`を維持する。Backend更新時は旧payloadをRuntime変換せずoffline recookする。

### 10.3 Tile promotion

build／rebuildはstaging generationで行う。影響tileと境界隣接tileのclosureをbuildし、seam、connectivity、memory、reference queryが全合格した後だけ、Runtime規約に従い`T00`で`CookedNavWorld` generationを一度に交換する。

旧generationはlease countが0になるまでretireしない。retire待ちは1 generationだけで、120 frame以内に解放されなければ新しいpromotionを停止してCI／Developmentを失敗させる。

## 11. Runtime query契約

### 11.1 `NavQueryRequestV1`

| Field | 型／規則 |
|---|---|
| `request_id` | 単調増加`uint64` |
| `request_tick／deadline_tick` | `uint64`、`request_tick < deadline_tick` |
| `owner_handle` | generation付きEngine handle |
| `nav_world_version` | dispatch時のexact version |
| `agent_profile_id` | 登録済みStable ID |
| `start_m／goal_m` | finite float3、各軸`[-10000,10000]` |
| `projection_half_extents_m` | Profile既定または各軸`[0.01,100]` |
| `area_mask` | 64-bit。bit 0は常に0 |
| `cost_table` | 64-entry Q16.16または登録済みtable ID |
| `allow_partial` | bool |
| `max_nodes` | C1は4,096以下 |
| `max_corridor` | C1は2,048以下 |
| `max_points` | C1は256以下 |

未登録Profile、非finite、範囲外、0以下のcap、deadline逆転、area 0許可、cost範囲外を`InvalidRequest`でdispatch前に拒否する。

### 11.2 実行順

1. request schema、owner、deadline、queue capacityを検査する。
2. `NavMeshVersion`のread leaseを取得する。
3. start／goalを指定half extentsとarea filterでprojectする。
4. endpointのどちらかをprojectできなければ`InvalidEndpoint`とする。
5. worker-local `NavQueryLease`でcorridorを検索する。
6. corridor statusとcapを検査する。
7. corridorが有効な場合だけstraight pathを生成する。
8. Backend refをEngine path point、typed traversal action、metricへ変換する。
9. private corridorとleaseを破棄する。
10. 最短でも次tickの`T20`でrequest ID順に統合し、version、owner、deadlineを再検査する。

同じ`dtNavMeshQuery`をparallel requestで共有しない。Detour 1.6.0の`findPath`は`const` memberだが、内部node poolとopen listをclear／更新するため、Miraikanaiは1 concurrent jobにつき1 query objectを割り当てる。sliced query stateもleaseを跨いで共有しない。

### 11.3 `NavQueryStatusV1`

| Status | 正確な条件 | Path利用 |
|---|---|---|
| `Success` | endpoint到達、全cap内、version／owner／deadline有効 | 利用可 |
| `NoPath` | endpointは有効だが、best corridorがstart polygonから前進しない | 不可 |
| `PartialPath` | endpoint未到達だが2 polygon以上のbest corridorがあり、budget超過なし | `allow_partial=true`の場合だけ利用可 |
| `InvalidRequest` | schema、Profile、range、mask、cost、cap、deadlineが無効 | 不可 |
| `InvalidEndpoint` | startまたはgoalをprojection box内のpolygonへprojectできない | 不可 |
| `SearchBudgetExceeded` | `DT_OUT_OF_NODES`、corridor buffer不足、straight path buffer不足、Engine iteration cap到達 | 不可。diagnostic pathだけ添付可 |
| `CostOverflow` | 2D Q16.16累積costまたはBackend正規化costが`uint64`をoverflow | 不可 |
| `StaleNavMesh` | dispatch versionとT20時点のactive versionが不一致 | 不可、Worldへ配送しない |
| `QueueFull` | request queue 4,096件／tickまたはarena 8 MiBを超える | 不可 |
| `Cancelled` | owner失効、明示cancel、deadline超過 | 不可 |
| `BackendFailure` | validated requestに対する予期しないVendor failure、corrupt internal state | 不可、Diagnostic必須 |

正規化の優先順位は、dispatch前が`InvalidRequest > QueueFull`、worker内が`InvalidEndpoint > BackendFailure > SearchBudgetExceeded > CostOverflow > NoPath／PartialPath／Success`である。T20ではowner失効、明示cancel、deadline超過なら`Cancelled`を最優先し、それらがなくactive versionだけが不一致なら`StaleNavMesh`へ置換する。`Cancelled`と`StaleNavMesh`はGameplayへ配送しない。

Detourの`DT_SUCCESS`はEngineの`Success`と同義ではない。`DT_PARTIAL_RESULT`、`DT_OUT_OF_NODES`、`DT_BUFFER_TOO_SMALL`をdetail bitとして必ず検査する。`DT_OUT_OF_NODES`または`DT_BUFFER_TOO_SMALL`を`PartialPath`へ格下げしない。

### 11.4 Result

`NavQueryResultV1`はrequest ID、status、request／completion tick、requested／active Navmesh version、projected endpoints、Engine-owned path point、typed traversal action、Q16.16 total cost、visited node、corridor／point count、worker CPU time、Diagnostic IDを持つ。

`dtPolyRef`、tile index、salt、Vendor flagを含めない。Path pointはfinite float3で、startからgoal方向の順序を持つ。off-mesh traversal点には`TraversalActionId`とlink generationを付け、実行直前に再検査する。

## 12. Area、Modifier、Link、Dynamic obstacle

- C1 Runtimeが変更できるのは既存Nav上のcost table、goal、typed off-mesh linkのenabled stateである。
- C1はAI、Gameplay、Project C++からpolygon、tile、voxel、raw link callbackを生成しない。
- `NavModifierV1`はbounded box、cylinder、convex prismと`blocked | area_override`だけを持つ。arbitrary executable predicateを持たない。
- `NavOffMeshLinkV1`はStable ID、start／end、direction、radius、area、`TraversalActionId`、generationを持つ。
- Door、jump、climbは任意callback名でなくCapability Catalog登録済み`TraversalActionId`を使う。
- C2 dynamic obstacleは前tick`T60`後の`NavObstacleSnapshot`を次tick`T20`で取り込む。同tickPhysics Worldを参照しない。
- Recast 1.6.0のDetourTileCacheは1 instance当たりrequest／update内部固定配列各64件である。C2 Adapterはこの上限をMiraikanai queueへ写さず、Engine queueから1 updateずつ投入し、`upToDate`までbounded stepする。Vendor queue fullを成功扱いにしない。
- DetourCrowdとlocal avoidanceはC2の非authoritative velocity proposalである。World Transformを書かず、Character Motor／Physicsが最終移動を決める。

## 13. AI、Editor、Gameplay操作

### 13.1 AIへ公開するOperation

| Operation | 入力 | 検証／出力 |
|---|---|---|
| `CreateNavAgentProfile` | semantic role、寸法、移動制約 | voxel換算、予測memory／cook cost、clearance preview |
| `AssignNavigationSource` | Scene／Collider Stable ID、area | source revision、static性、triangle／range |
| `SetNavAreaCost` | area Stable ID、multiplier | range、影響Agent、path diff preview |
| `CreateNavModifier` | bounded shape、mode、area | overlap、required path影響 |
| `CreateNavOffMeshLink` | endpoints、direction、action | projection、clearance、Capability、generation |
| `RequestNavigationBuild` | source set、Profile、Target | affected tile、peak memory、estimated duration、Risk |
| `RunReachabilityValidation` | spawn／goal／required interaction set | version付きPass／Fail／Diagnostic |

AIはBackend ID、`rcConfig` field、voxel値、tile coordinate、polygon ref、Detour flag、binary payload、allocator、thread数を直接指定しない。`HumanReferenceV1`から外れるCustom Profileは正規Agent値を入力し、Engineがvoxel値を計算する。

### 13.2 質問とApproval

次はHigh ImpactとしてCommit前に人間へ質問する。

- 通路幅に対してAgent radius／heightが未指定で、既定値により到達可能性が変わる。
- 複数体格Agentが同じNavmeshを共有するか、Profile別にcookするかでmemoryが変わる。
- required spawn→goal、door、jump、climbのどれを必須経路とするか不明である。
- Custom Profile、全tile rebuild、256 tile超の影響、live payload 75%以上を消費する変更である。
- C2 dynamic obstacle、crowd、streamingを要求しているがTarget CapabilityがC1だけである。

質問不要な低Impact変更は、既存登録Profileの選択、範囲内area cost、既存typed linkのenable／disable、debug表示である。Assumptionを使った場合はProject既定Profile IDと影響をChangeSetへ記録する。

### 13.3 Editor

EditorはSource geometry、walkable voxel、region、contour、polygon、tile／layer、area、link、projected endpoint、corridor、straight path、failed query、version、memory、build stage timeを切替表示する。

DiagnosticからSource Asset、primitive、triangle、Profile field、Modifier、Linkへ移動できる。Editorはraw Backend pointerを保持せず、bounded `NavigationDebugSnapshotV1`だけを描画する。

Gameplayはquery requestとtyped resultだけを扱う。NPC移動、path following、Character MotorはNavigation resultを入力として使うが、Navmesh polygonへ直接writeしない。

## 14. Memory、容量、性能

### 14.1 Runtime capacity

| 項目 | C1 hard cap |
|---|---:|
| Navigation Domain | 64 MiB |
| request／result queue | 各4,096件／tick、各8 MiB arena |
| live 2D／3D payload合計 | 36 MiB |
| query lease pool | 4 MiB |
| TileCache／dynamic obstacle reserve | 4 MiB |
| metadata | 3.75 MiB |
| active Navmesh | 1 generation |
| retire待ち | 1 generation、最大120 frame |
| Detour tile slot | 1,024 |
| polygon／tile | 4,096 |
| node／query | 4,096 |
| corridor／query | 2,048 |
| point／query | 256 |

cap超過はallocation fallback、silent truncation、old result再利用を行わず、closed statusとDiagnosticを返す。

### 14.2 Tool cook

- Editor-launched child tree 4 GiB hard commit capを超えない。
- 1 Nav build jobのtemporary commitは256 MiB hard cap、同時build jobは`min(max(logical_processors-4,1),6)`以下で、合計1.5 GiBを超えない。
- 1 tileのsource triangle／vertex capは各262,144である。
- Windows Referenceで1,024 tileのclean cookを60秒以内、1 tile incremental recookをP95 100 ms以内とする。
- build時間、peak commit、tile bytes、polygon、vertex、span、region、contour数をReceiptへ保存する。

### 14.3 Runtime performance Gate

Windows Referenceで、256 m×256 m C1 arena、Human Profile、path length 5～200 mの固定10,000 query traceを5回測定する。

| Metric | C1 Gate |
|---|---:|
| worker query CPU time | P95 0.50 ms、P99 1.50 ms／query |
| T20 result integration | P95 0.20 ms／tick |
| 64 concurrent request batch | P95 4.00 ms wall time |
| Runtime query allocation | lease初期化後0 allocation |
| 10分soak | leak 0、stale publish 0、cap超過成功扱い0 |

4096件／tickはqueue safety capであり、毎tickの性能保証件数ではない。Reference workloadは通常64件、stressは4,096件を一度投入し、QueueFull、deadline、翌tick回復を検証する。

## 15. Failure、Diagnostic、Recovery

| Failure | 正規処理 |
|---|---|
| Source invalid／非finite／縮退 | cook停止、Source位置付きDiagnostic、旧Artifact維持 |
| Profile invalid／capacity矛盾 | Commitまたはcook拒否、計算式と違反値を表示 |
| Recast build失敗／OOM | Artifact publishなし、stage／tile／peak memoryを記録 |
| Detour magic／version／hash不一致 | Artifactを`IncompatibleArtifact`として拒否、offline recook要求 |
| tile seam／connectivity不合格 | generation全体をpublishしない |
| query queue／arena満杯 | 新規requestを`QueueFull`、受付済みresultは保持 |
| node／corridor／point cap | `SearchBudgetExceeded`、成功扱いしない |
| stale version／owner／deadline | T20で破棄しWorldを変更しない |
| promotion memory不足 | promotion延期、旧generation維持 |
| Backend invariant違反 | `BackendFailure`、Development assert＋Shipping bounded failure |
| required pathなし | Level validation error、C1 promotion不可 |

Diagnostic IDは`NAV_SOURCE_*`、`NAV_PROFILE_*`、`NAV_BUILD_*`、`NAV_ARTIFACT_*`、`NAV_QUERY_*`、`NAV_PROMOTION_*`のclosed familyにする。Vendor log文字列を公開error contractにしない。

## 16. Test、Qualification、完了条件

### 16.1 Backend Qualification

Recast／Detour Backendは次をTargetごとに合格して`candidate_locked`から`active`へ昇格する。

- exact tag／commit／archive hash、zlib notice、patch manifest、SBOM。
- MSVC、clang-cl、Android NDK clang、Apple clangのprivate static build。
- ASan／UBSan相当、Windows Application Verifier、portable TSan fixture。
- upstream Recast／Detour TestsとMiraikanai Adapter conformance。
- 32-bit ref、1,024 tile、4,096 polygonでinit成功し、8,192 polygon設定をProfile Validatorが事前拒否する。
- allocator failureを全build stage、query object init、tile loadへ注入し、partial publish 0。
- corrupt magic、version、offset、length、hash、tile coordinate、duplicate tileを全件拒否。

### 16.2 Geometry／build fixture

- flat、slope 49.9／50.0／50.1°、step 0.39／0.40／0.41 m、clearance 1.79／1.80／1.81 m。
- narrow corridor、spiral stair、bridge over road、洞窟、8層stack、island、hole、thin ledge。
- tile四辺と四隅を横断するpath、border geometry、negative coordinate。
- mirrored transform、non-uniform scale、duplicate triangle、zero-area、NaN／Inf、index範囲外、巨大triangle。
- 4,096 polygon直前、4,097 polygon、65,534 vertex、65,535 vertex、1,024 tile、1,025 tile。
- 同一input 100 cookのArtifact hash一致。Target／toolchain差は別ArtifactKeyで混同しない。

### 16.3 Query fixture

- same polygon、直線、曲折、cost迂回、area mask、off-mesh one-way／two-way。
- projection box境界、endpointなし、start blocked、goal blocked。
- disconnectedで前進なし=`NoPath`、前進あり=`PartialPath`。
- node 4,095／4,096／4,097相当、corridor 2,047／2,048／2,049、point 255／256／257。
- `allow_partial` true／false、cancel、owner destroy、deadline、tile swap、stale result。
- 1 query objectの同時利用をconcurrency testで禁止し、worker別leaseでrace 0。
- query結果へVendor ref、pointer、tile indexが含まれないことをschema／binary scanする。

### 16.4 AI／Editor

- AI promptを2D／3D／Hybrid、Agent、area cost、blocked、link、required path、Custom Profile、全rebuild、曖昧、矛盾、unsupportedで各12件以上、各3回実行する。
- AIのBackend直接選択、Vendor field／polygon／tile生成、hard cap回避、unsafe Commitを0件にする。
- 質問／Assumption／Diff／cost previewの期待一致を95%以上にする。
- AI作成→Inspector変更→Project C++ query→Undo／Redo→再cookが同じChangeSet／Validator経路で往復する。
- debug overlayの表示対象とSource Diagnostic navigationをgolden UI fixtureで検証する。

### 16.5 C1 Production完了条件

1. 本書のMCD type、operation、capability、profile、diagnosticが正本化される。
2. C++／TypeScript／Cooked descriptor／MCP／Provider projectionが同じMCDから生成される。
3. Recast／Detour型がPublic Header、Module interface、Project C++、Save、Event、AI Schemaへ出ない。
4. 2D Grid、3D Backend、Artifact、query、Editor、AI fixtureが全合格する。
5. Navigation 64 MiB、runtime allocation 0、P95／P99、cook時間、10分soakを満たす。
6. required spawn→goal／interaction reachabilityが3D C1 Reference Sceneで全合格する。
7. Source、Build、Cook、Test、Qualification、Promotion Receiptがhashで連結される。

2D合格を3D合格、Windows合格をmobile合格として表示しない。

## 17. 実装順序とGate

| 順序 | 成果物 | Exit Gate |
|---:|---|---|
| N0 | MCD Navigation type／status／Profile、Backend Port、fake fixture Backend | schema projection、dependency negative、status totality |
| N1 | Grid2D cook、A*、clearance、cost、debug | 2D fixture、34 MiB cap、deterministic tie-break |
| N2 | Recast／Detour exact lock、private Build、allocator、Qualification harness | source／license／Build／sanitizer receipt |
| N3 | 3D canonical geometry、Profile mapping、tiled build、Artifact envelope | geometry、seam、capacity、corrupt Artifact fixture |
| N4 | immutable load、query lease、projection、corridor、straight path、status normalization | query、concurrency、stale、budget fixture |
| N5 | Authoring Operation、Editor overlay、AI projection、reachability validator | AI unsafe 0、Editor golden、required path |
| N6 | 3D compact action arena、performance、memory、soak | C1 Production完了条件 |
| N7 | C2 TileCache、partial rebuild、streaming、hierarchical、crowd | 個別ADR、Target benchmark、C2 Gate |

N2未合格でN3をProduction実装へ進めない。N6合格前にC2 CapabilityをProduction表示しない。N0～N6のfile、target、test command、commit単位は本Review set承認後の3D C1実装計画書へ分解する。

## 18. Backend更新と完全自作研究

Backend更新は通常dependency updateではなくNavigation behavior／Artifact compatibility変更として扱う。

1. 新versionを別`candidate_locked` entryへ追加する。
2. release note、source diff、magic／version、ref bit、allocator、license、Build optionをEvidenceへ保存する。
3. 旧／新Backendを同一geometry、query、performance fixtureでQualificationする。
4. 旧Artifactはoffline recookし、Runtime互換shimを追加しない。
5. reachability、path status、endpoint、cost、memory、cook／query性能差をreportする。
6. 人間R4 Review後にだけ`active`を切り替える。

完全自作3D Backend研究はC3で次をすべて満たす場合だけ開始する。

- 現Backendで解決できない具体的Product要件と再現fixture。
- Recast拡張、別kernel、Game-side近似との比較。
- C1相当18～30人月、Production累計50～90人月を基準にした人員、期間、保守見積り。
- geometry耐性、tile seam、query、dynamic update、Editor、AI、全Targetを含む実装範囲。
- 同一公開契約と全conformance testを変更せず通せるBackend Port。
- 基準Backend比で全必須metricの回帰5%以内を満たし、未達Product要件を解決するか、選定した主要metricを20%以上改善する5回測定。
- 失敗時にactive Recast／Detour Backendへ戻せるArtifact／Project境界。

「AIが理解しやすそう」「外部Libraryを使いたくない」だけを開始理由にしない。

## 19. 主要リスクと確定対策

| リスク | 対策 |
|---|---|
| Vendor APIが公開契約へ漏れる | Backend-only include／link、schema／binary／module scan |
| 32-bit ref capacityが成立しない | 1,024 tile×4,096 polygon、10 salt bitをValidator／init testで固定 |
| Detour detail bitを成功扱いする | status正規化の優先順位、budget fixture |
| query objectを並列共有する | 1 concurrent job＝1 lease、TSan／race fixture |
| NavmeshがWorldの正本になる | Source＋Profileから再生成するDerived Asset |
| raw tile更新で新旧が混在する | staging closure、T00 atomic generation swap |
| AIがvoxel／polygonを直接編集する | semantic Operation、Profile換算、Vendor field denylist |
| Custom Profileでcook costが爆発する | source／tile／memory cap、preview、High Impact Approval |
| dynamic obstacleがPhysicsと競合する | 前tick snapshot、C2 bounded update、Transform write禁止 |
| Backend更新で旧Artifactを誤読する | Engine envelope、exact lock、hash、offline recook |
| 完全自作が目的化する | C3 evidence／benchmark／rollback Gate |

## 20. 公式資料と採用根拠

調査基準日は2026-07-20である。外部資料はLibrary／Engineの事実確認に使い、Miraikanai固有のSchema、Profile、status、budget、AI Policyをコピーしない。

| 公式資料 | 確認事項 | 本書への反映 |
|---|---|---|
| [Recast Navigation 1.6.0 release](https://github.com/recastnavigation/recastnavigation/releases/tag/v1.6.0) | exact release | Backend lock |
| [Recast 1.6.0 API reference](https://github.com/recastnavigation/recastnavigation/blob/v1.6.0/Docs/Extern/Recast_api.txt) | rasterize、filter、compact field、region、contour、poly、detail API | Build pipeline |
| [Recast 1.6.0 tiled build sample](https://github.com/recastnavigation/recastnavigation/blob/v1.6.0/RecastDemo/Source/Sample_TileMesh.cpp) | tile bounds、border、stage呼出順、Detour tile生成 | C1 tiled build mapping |
| [Recast `rcConfig`](https://github.com/recastnavigation/recastnavigation/blob/v1.6.0/Recast/Include/Recast.h) | field単位、範囲、`maxVertsPerPoly` | Profile mappingとvalidator |
| [Detour status](https://github.com/recastnavigation/recastnavigation/blob/v1.6.0/Detour/Include/DetourStatus.h) | success／failureとwrong version、OOM、invalid、buffer、nodes、partial detail bit | Engine status正規化 |
| [Detour NavMesh](https://github.com/recastnavigation/recastnavigation/blob/v1.6.0/Detour/Include/DetourNavMesh.h) | 32／64-bit ref非互換、6 vertices、version 7、64 area | Build lock、Artifact envelope |
| [Detour NavMesh init](https://github.com/recastnavigation/recastnavigation/blob/v1.6.0/Detour/Source/DetourNavMesh.cpp) | tile／poly／salt bit算出、salt 10 bit未満を拒否 | 1,024×4,096 capacity |
| [Detour NavMesh builder](https://github.com/recastnavigation/recastnavigation/blob/v1.6.0/Detour/Source/DetourNavMeshBuilder.cpp) | `nvp<=6`、vertex `<65535`、magic／version | tile hard capとload validation |
| [Detour query header](https://github.com/recastnavigation/recastnavigation/blob/v1.6.0/Detour/Include/DetourNavMeshQuery.h) | max node 65,535、path／straight path buffer | query schema |
| [Detour query implementation](https://github.com/recastnavigation/recastnavigation/blob/v1.6.0/Detour/Source/DetourNavMeshQuery.cpp) | node pool／open list更新、partial／out-of-nodes／buffer behavior | worker別lease、status優先順位 |
| [Detour TileCache](https://github.com/recastnavigation/recastnavigation/blob/v1.6.0/DetourTileCache/Include/DetourTileCache.h) | obstacle、bounded update、内部request／update各64 | C2 queue Adapter |
| [Recast zlib License](https://github.com/recastnavigation/recastnavigation/blob/v1.6.0/License.txt) | 使用、改変、再配布条件 | Third-party notice、patch表示 |
| [Unreal Navigation System](https://dev.epicgames.com/documentation/en-us/unreal-engine/navigation-system-in-unreal-engine) | tile、polygon cost、dynamic mode、avoidance | Product API＋Recast系統合の比較 |
| [Unreal `ARecastNavMesh`](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/NavigationSystem/ARecastNavMesh) | Engine classとしてRecast Navmeshを統合 | private Backend方式の比較 |
| [Godot NavigationMesh](https://docs.godotengine.org/en/4.0/tutorials/navigation/navigation_using_navigationmeshes.html) | 3D bakeで第三者Recastを使用 | bake kernel分離の比較 |
| [O3DE Recast Navigation Gem](https://docs.o3de.org/docs/user-guide/gems/reference/ai/recast/recast-navigation/) | build、path、可視化 | Backend機能分離の比較 |
| [CRYENGINE MNM](https://www.cryengine.com/docs/static/engines/cryengine-5/categories/23756816/pages/25534274) | 独自Multi-layer Navmesh | 完全自作案の規模比較 |
