# Navigation Design Alignment Review

- 文書ID: mirakan.appendix.navigation-design-alignment-review
- 文書種別: proposal appendix
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 親Owner: [Navigation](../05-simulation/navigation.md)
- 正本範囲: Navigation設計監査の結論、AI可読性、主要Engine比較、cross-owner整合性、意図的な未確定事項の追跡
- 非正本範囲: Navigation runtime semantics、MCD Schema、Product Registry、実装Task、実装順序、担当、工数、生成済みArtifact、Qualification結果、Capability activation
- 規範依存: [Navigation](../05-simulation/navigation.md)、[Product Plan](../00-product/product-plan.md)、[Runtime Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)
- 関連文書: [Product Execution Registry Proposal](product-execution-registry-proposal.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Persistence／Save](../04-runtime/persistence-save.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[Gameplay Feature Packs](../08-packs/gameplay-features.md)、[Collision](../05-simulation/collision.md)、[Physics](../05-simulation/physics.md)、[World](../06-rendering/world.md)
- 根拠区分: project-review／official-spec comparison
- 外部根拠確認日: 2026-07-28

> 本書は実装計画ではない。列挙順は実装順序、Product Phase、担当または日程を意味しない。`closed-in-target-design`はtarget文書内の責務と意味が閉じていることだけを表し、Schema、Artifact、Runtime、Receipt、CapabilityまたはOperationが存在・承認・適用済みであることを意味しない。

## 1. 監査結論

Miraikanai Navigationは、2D Gridと3D Navmeshを同格のEngine-owned contractとして扱い、Backendをprivate Adapterへ隔離し、immutable artifact、atomic publication、version／lease、normalized status、canonical orderingでpublic semanticsを閉じる方向を維持する。この方向は主要Engineよりauthoring即応性では未成熟だが、Backend非依存性、決定性、stale拒否、AI安全境界では明確である。

AI可読性は「誤操作を防ぐ能力」と「現在利用可能な操作を発見して実行する能力」を分けて評価する。

| 観点 | 判定 | 根拠 |
|---|---|---|
| 概念理解 | strong | Profile、Source、Artifact、Query、Result、Version、Lease、Movement Intent、Executorの責務が型名とOwnerで分離される |
| current／target判別 | explicit after alignment | Navigation §1.1がtarget contract、planned vocabulary、conditional research、deferred capabilityを分離する |
| AI安全境界 | strong target design | Vendor object、native ref、raw status、live World、mutable lease、Backend traceをAIへ公開しない |
| 文書検索性 | usable but dense | Navmesh coreとPath Following／Motion Executor closureが一つの正本に同居し、後者のrecord定義が長い |
| 機械的解決 | incomplete | MCD Schema、Contract compiler、generated projection、active Contract setがRepositoryに存在しない |
| 実行可能性 | absent | current Navigation MCD Operation集合は空で、authoring actionはplanned vocabularyに留まる |

したがって「AIが設計を安全に説明する」targetは強いが、「AIがcurrent capabilityを機械発見してNavigationを編集・cook・previewする」状態ではない。自然言語上の型名、action、fixture、work itemをactive inventoryと解釈しない。

## 2. AI向け読解モデル

AI、Editor、Project C++、計画書readerは次の順序でNavigationを読む。

1. [Navigation §1.1](../05-simulation/navigation.md#navigation-current-target-reading)でcurrent／target／planned／deferredを判別する。
2. Worldのexact `WorldSpaceProfileRefV1`から2D Gridまたは3D Navmesh authorityを一意に決める。Navigation側でdimensionを再選択しない。
3. Source、Profile、Artifact、active versionを分離し、Editor selection、display name、path、latest artifactからidentityを推測しない。
4. Queryはexpected version、Profile、Agent、filter、result boundを完全指定し、Resultはobserved versionとlease中だけ解釈する。
5. Path FollowingはGameplay Feature Capabilityであり、Navigationはquery／versionとMovement Intent境界だけを所有する。
6. AIが説明できることをOperation activation、Backend採用、Qualification passまたはProduct availabilityのEvidenceにしない。

AI向けの要約はArchitecture本文の代替正本を作らず、本文のexact ref、status、Owner、hashへ戻れるread-only projectionにする。主要EngineのUI用語をMiraikanaiのStable ID、MCD ID、Operation IDとして流用しない。

## 3. 主要Engineとの比較

比較はMiraikanaiのpublic contractを主要Engineへ合わせるためではなく、authoring／debug projectionの不足と、維持すべきEngine-owned境界を識別するために使う。

| Engine | 公式public model | Miraikanaiへの示唆 | 採らないもの |
|---|---|---|---|
| Unreal Engine | Collision geometryからtiled Navigation Meshを生成し、Static／Dynamic／Dynamic Modifiers Only、Navigation Modifier、Link、Navigation Invoker、RVO／Detour Crowdを公開する | Bounds、area、link、runtime generation、agent別結果をEditor上で発見・可視化するprojectionは有用 | `ARecastNavMesh`、`NavNodeRef`、Recast setting、Crowd objectをEngine public identityまたはSave formatにする |
| Unity AI Navigation | `NavMeshSurface`、Modifier／Volume、Link、Agent、Obstacle等の高水準Componentでedit time／runtime build、dynamic obstacle、link traversalを扱う | 少数の高水準authoring conceptとScene上のpreviewはAI／初心者の説明負荷を下げる | Component load順、Scene object、implicit defaultをartifact identity、activation boundary、writer authorityにする |
| Godot Navigation | `NavigationServer2D/3D`、map、region、agent、RID、path query parameter／result、physics-frame synchronization、map iterationを公開する | 2D／3D parity、query object再利用、同期後iteration確認は明示的なlifecycle説明として有用 | raw RID、次frame待ち、iteration IDだけをlease、artifact identity、stale判定の完全な代替にする |
| Miraikanai | Engine-owned Profile／Artifact／Query／Status／Version／Lease、private Grid／Navmesh Backend、atomic generation publication、selected Motion Executor Port | 決定性、Backend交換、Save／Replay、AI安全性を維持しつつ、同じ正本からEditor／AI projectionを生成する | Vendor object公開、live mutation、partial success、latest推測、runtime自動algorithm切替 |

外部比較から採用するのは、authoring conceptの少なさ、可視化、同期状態の明示、診断可能性である。Miraikanai固有のversion／lease、atomic publication、exact World Profile、owner分離を弱めて互換UIへ近似しない。

## 4. Cross-owner整合性

statusは次の三値で読む。

- `closed-in-target-design`: Ownerとconsumer境界、failure semantics、identityがtarget文書で一致する。
- `intentional-open`: 測定、materialization、Qualificationまたは別Capability Decisionを待つ値であり、別文書から補完しない。
- `absent-evidence`: target意味は記述されるが、RepositoryにSchema、Artifact、Runtime、Receiptまたは合格Evidenceがない。

| 領域 | 正本／連携 | status | 監査結果 |
|---|---|---|---|
| Product ownership | Product Plan／Product Execution Registry／Gameplay Feature Packs | `closed-in-target-design` | `wp.navigation.core`はquery／artifact／provider portと`capability.simulation.navigation`、`wp.gameplay.reusable-features-c1`は`capability.gameplay.path_following`を所有する。Navigationは後者のNavigation-facing contract正本である |
| Spatial authority | World／Navigation | `closed-in-target-design` | exact `WorldSpaceProfileRefV1`だけが2D／3D authorityを選び、hybrid presentationから第二Navigation authorityを作らない |
| Geometry input | Collision／Asset Lifecycle／Navigation | `closed-in-target-design` | Collisionのcanonical static triangle streamとSource revisionをNavigation cookが消費し、Visual Meshまたはlive Physics objectを直接読むことを禁止する |
| Artifact publication | Asset Lifecycle／Scheduling／Navigation | `closed-in-target-design` | immutable staging、hard dependency all-ready、Simulation boundary、atomic Navmesh version publication、last-valid維持が一致する |
| World streaming | World／Asset Lifecycle／Scheduling／Navigation | `closed-in-target-design` | WorldがCell membership、Assetがclosure、Schedulingが`T00_BoundaryApply`、NavigationがNav World binding／version／leaseを所有する。部分Cell／部分Navmeshを公開しない |
| Async query | Scheduling／Navigation | `closed-in-target-design` | `T20_AsyncIntegrate`でdeadline、expected version、owner generationを検査し、accepted resultだけを次のGameplay evaluationへ渡す |
| Motion authority | Navigation／Gameplay Feature Packs／Physics／Animation／Scheduling | `closed-in-target-design` | Path Followingはintentを提出し、selected Motion Executorだけがresolved transformを書き、Animationはposeを所有する |
| Save／Load／Replay | Gameplay Feature Packs／Persistence／Navigation | `closed-in-target-design` | owner要求時だけdestination intent等を保存し、waypoint、path point、query handle、Physics stateを保存しない。Loadは再queryし、Replayはaccepted resultとversion／hashを記録する |
| Capacity／latency | Performance／Navigation | `intentional-open` | queue bound、integration budget、metric familyは定義済み。obstacle inputからversion activationまでのAdvance上限は測定前のため未固定で、値を前提にQualificationしない |
| Toolchain | Toolchain／Dependencies／Navigation | `closed-in-target-design` | Recast／Detour exact version、commit、license、build boundaryはToolchainだけが所有し、Navigation public identityへVendor versionを露出しない |
| MCD／Operation | Executable Contracts／Navigation／AI Security | `intentional-open` | authoring actionはplanned vocabulary、selection familyは`not_activated`、current Navigation Operation集合は空である |
| Runtime／Qualification Evidence | Navigation／AI Verification／Product | `absent-evidence` | canonical result、allocation 0、sliced lifecycle、tile publish、cross-target hash等のfixture要求はあるが、実装ArtifactとReceiptは存在しない |

## 5. 本Reviewで確定する整理

1. Navigationの中核設計は、Grid／Navmesh Backendの統一ではなく、Engine-owned public contractとBackend-private implementationの統一である。Grid2Dを3D Navmesh上へ実装しない。
2. Path FollowingはProduct上のGameplay Feature Capabilityである。Navigationはquery／version integration、Navigation-facing request／state／intent semantics、Motion Executor Port境界を所有するが、Feature PackのProduct ownerを置換しない。
3. World streamingでは、World membership、Asset closure、Scheduling boundary、Navigation version／leaseを一Ownerへ集約しない。四Ownerのsealed activation setが一致した場合だけ公開する。
4. Save／Replay境界は既にtarget design内で閉じている。Nav path、waypoint、native handleを保存せず、永続化されたdestination intentからload後に再queryする。
5. obstacle反映latency、MCD materialization、Operation activation、Backend Qualificationは意図的な未確定事項である。Architecture readerまたはAIが値、ID、pass結果を補完しない。
6. JPS／ALT／D* Lite／HPA*／flow field／local avoidanceのdispositionを変更しない。主要Engineが提供することをMiraikanaiでのC1採用根拠にしない。

## 6. 文書改善の推奨

推奨はArchitectureの可読性と正本境界に限定し、実装Taskまたは実装順序を定義しない。

- Navigation §1.1のcurrent／target tableをAI／Editorの最初の読解入口として維持する。
- public contractとProduct Capability ownerを別Field／別文で表し、「Navigation owner」の一語で両者を兼用しない。
- Editor／AI向けprojectionは主要Engineの高水準conceptを参考にできるが、必ずMiraikanaiのSource ref、artifact identity、active version、validation status、assumptionを表示する。
- Motion Executor closureの長さが検索精度を下げる場合は、将来の文書再編Decisionでrecord catalogをAppendixへ分離できる。ただしcurrent正本を重複せず、stable anchorとConsumer Inventoryを伴う一対一移行だけを許可する。
- `closed-in-target-design`と実装済みを区別し、Review、Product Registry、AI説明からCapability activationを導出しない。
- 外部Engine比較は本Reviewのinformational根拠に留め、Navigation contractへVendor固有名または設定を追加しない。

## 7. 文書Qualification

本ReviewとNavigation正本の変更は次を満たす。

- Navigationから参照する全local Markdown targetが存在する。
- `capability.gameplay.path_following`のProduct ownerとNavigation-facing contract ownerを区別する。
- current Navigation MCD Operation集合を空、selection候補を`not_activated`のまま維持する。
- Product Capability、Work Package、Target、Phase、Dependency pin、budget、algorithm dispositionを変更しない。
- Save／Replay既存意味を縮小・拡張せず、整合済みとして記録する。
- World／Cell／Asset／Navigation publicationのfailure時にlast-valid groupとversionを維持する。
- 実装Task、順序、担当、工数、deadline、生成済みSchema／Artifact／Receiptを追加しない。

## 8. 外部一次資料

- Unreal Engine: [Navigation System](https://dev.epicgames.com/documentation/en-us/unreal-engine/navigation-system-in-unreal-engine)、[Using Navigation Invokers](https://dev.epicgames.com/documentation/en-us/unreal-engine/using-navigation-invokers-in-unreal-engine)、[Navmesh Runtime API](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/Navmesh)
- Unity: [AI Navigation package](https://docs.unity3d.com/Manual/com.unity.ai.navigation.html)
- Godot: [Using NavigationServer](https://docs.godotengine.org/en/stable/tutorials/navigation/navigation_using_navigationservers.html)、[Using NavigationPathQueryObjects](https://docs.godotengine.org/en/stable/tutorials/navigation/navigation_using_navigationpathqueryobjects.html)
- Recast／Detour: [`dtNavMeshQuery`](https://recastnav.com/classdtNavMeshQuery.html)、[`dtTileCache`](https://recastnav.com/classdtTileCache.html)
