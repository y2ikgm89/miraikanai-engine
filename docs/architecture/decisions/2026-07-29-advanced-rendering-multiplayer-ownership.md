# Advanced Rendering／Multiplayer Ownership

- 文書ID: mirakan.decision.advanced-rendering-multiplayer-ownership
- 状態: review
- 正本範囲: Advanced Rendering／Multiplayerの将来Owner分割、既存Ownerからのsemantic移管、Product Future原子性、Online Services分離の判断理由
- 非正本範囲: 各OwnerのSchema／Operation／failure／Qualification、Product Activation、実装Task、実装順序、担当、工数、日程、Provider／Protocol／algorithm選定。各Owner文書とProduct Planを参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Product Plan](../00-product/product-plan.md)、[Render Graph](../06-rendering/render-graph.md)、[Advanced Light Transport](../06-rendering/advanced-light-transport.md)、[Terrain／Foliage](../06-rendering/terrain-foliage.md)、[Network Transport／Connection](../09-networking/network-transport-connection.md)、[Multiplayer Authority／Replication](../09-networking/multiplayer-authority-replication.md)、[Architecture Plan Closure Review](../appendices/architecture-plan-closure-review.md)
- 外部根拠検証日: 2026-07-29
- 文書種別: Architecture Decision／cross-owner ownership
- Decision owner document: `mirakan.arch.architecture-governance`
- Decision日: 2026-07-29
- Supersedes: none

## Context

既存ArchitectureはRender Graph、Lighting、Materials、Post Processing、Environment、LOD、Virtual Geometry、World、Runtime Package、Scheduling、ECS等を詳細に分離していた。一方、将来PortfolioはTerrain／Foliage／GIを一行かつEnvironment Ownerへ、Dedicated Server／session transport／replicationを一行かつScheduling Ownerへ束ねていた。

この状態では、次が一意に答えられない。

- GIとReflectionのどのsemantic channelがどのTargetでqualifiedか
- TerrainまたはFoliageだけを独立に昇格／降格できるか
- Render Graphがlight-transport techniqueを選ぶのか、executionだけを行うのか
- connection成功とgameplay session／authority成立の差
- listen co-opがDedicated Serverを必須とするか
- prediction／rollbackが全Multiplayerに必須か
- Lobby／Matchmaking／Relay／HostingがTransportまたはReplicationに含まれるか

関連実装、公開Schema、Fixture、Receiptは`absent`で、Future rowは`planning_only`である。したがって、公開済み互換性を守るaliasより、Ownerとactivation identityが正しいV1をclean-breakで設計できる。

## Decision drivers

1. 一つの意味をexactly one Ownerへ置く。
2. Source、Artifact、Resolved Plan、Runtime Snapshot、ReceiptをAIが一方向に追跡できる。
3. technique、Target、Provider、quality claim、topologyを直交させる。
4. 独立Target support、fallback、Qualification、Product claimを同じFuture IDへ束ねない。
5. 現MVP、active Capability、release claim、実装DAGを拡張しない。
6. 有名Engineの製品名やAPIではなく、責務分離とsupport matrixだけを比較根拠にする。

## Decision

### 1. Ownerは四件だけ追加する

| Owner | 正本 |
|---|---|
| `mirakan.arch.rendering-advanced-light-transport` | diffuse indirect、specular indirect、shadow visibility、reference transport、channel別Technique／Target support／fallback／Qualification |
| `mirakan.arch.rendering-terrain-foliage` | Terrain Source／tile／layer／artifactとFoliage species／placement／identity／artifact。ただし両branchは独立Activation |
| `mirakan.arch.network-transport-connection` | endpoint、handshake、connection epoch、semantic delivery、packet／fragment、backpressure、transport security、private Adapter |
| `mirakan.arch.multiplayer-authority-replication` | topology／role／authority、session、Network Object、replication、typed command、interest、prediction／rollback subprofile、resync／handoff |

既存Ownerは次を維持する。

- Render Graph: pass、resource、queue、barrier、history allocation、submission、`RayTracingPortV1`／`RadianceCachePortV1`／`NeuralRenderModelV1`
- Lighting: Light Source、photometry、attenuation、Shadow Intent／Style
- Post Processing: generic effect composition、exposure、generic temporal history intent
- Environment: sky、atmosphere、fog、cloud、weather presentation、water、snow／wetness
- World: Scene／Space／Cell、partition、activation
- LOD／Virtual Geometry: representation selection／geometry representation
- Runtime Asset Lifecycle: request、generation、residency、lease、eviction
- Runtime Package: world／ui／headless package／launch closure
- Scheduling／ECS: phase／cadence／lifetime、state storage／query／structural transaction
- Product Plan: Future membership、cross-domain AAA／photoreal quality claim

### 2. Render Graphからsemantic selectionだけを移す

GI／Reflection／advanced Shadow／path roleのsemantic Profile ID、Technique selection、representation requirement、channel-specific history／denoise、meaning-equivalence fallback、`ShadowGraphV1`／`ResolvedShadowPlanV1`はAdvanced Light Transportへ移す。

Render Graphはlogical execution port、Pass Template展開、physical history、resource lifetime、Backend Adapterを維持する。Post Processingはgeneric effect／history intentを維持する。これによりALT、Post、Render Graphの三重ownershipを作らない。

### 3. Future Portfolioを31行にする

旧二複合行を廃止する。

- `future.capability.production-terrain-foliage-gi`
- `future.capability.headless-dedicated-server-session-transport-replication`

次の七つへclean-breakする。

| Future ID | Owner | 直接Product claim |
|---|---|---|
| `future.capability.production-terrain` | Terrain／Foliage | `claim.product.feature.production-terrain` |
| `future.capability.production-foliage` | Terrain／Foliage | `claim.product.feature.production-foliage` |
| `future.capability.production-global-illumination` | Advanced Light Transport | `claim.product.feature.production-gi` |
| `future.capability.production-reflections` | Advanced Light Transport | `claim.product.feature.production-reflections` |
| `future.capability.headless-dedicated-server-target` | Runtime Package | `claim.product.network.dedicated-server-support` |
| `future.capability.network-transport-connection` | Network Transport／Connection | なし。技術前提 |
| `future.capability.multiplayer-authority-replication` | Multiplayer Authority／Replication | `claim.product.network.multiplayer` |

計算は`26 - 2 + 7 = 31`である。全行は`planning_only`を維持する。`future.capability.production-global-illumination-reflections`を中間umbrella、alias、fallbackまたはpromotion mappingにしない。

GIとReflectionsは同じOwnerでも、semantic channel、Target support、fallback、Qualification、claimが独立するため別Future IDにする。互いにprerequisiteを持たない。AAA／photorealだけが同一Target上のTerrain、Foliage、GI、Reflections四件を前提にし、特定ray／path／neural／virtual geometry／Providerを一律必須にしない。

### 4. Multiplayer依存を分離する

- Authority／ReplicationはTransportを技術前提にするが、Transport `active`をsession `joined／synchronized／authoritative`と解釈しない。
- small co-opはTransport＋Authority／Replicationを前提にし、listenまたはdedicatedを別Product Decisionで選べる。Dedicated Serverを必須にしない。
- rollback competitiveはTransport＋Authority／Replicationを前提にし、rollback Profileを専用にQualificationする。
- large-session shooterはDedicated Server＋Transport＋Authority／Replicationを前提にするが、rollbackを必須にしない。
- persistence／live-service／moderation operationsはgame Transport、Replication、dedicated game Runtimeから独立させる。
- MMOはDedicated Server＋Transport＋Authority／Replication＋offline large World＋persistence／operationsを前提にする。

### 5. Target-role bundleをProduct正本にする

単一描画Targetのsame-target prerequisiteを、client、authority server、operationsを同時に必要とするProduct capabilityへ適用すると、large-sessionのclient TargetとMMOの全Targetが到達不能になる。candidate Target kindを広げてDedicated Serverやoffline large Worldの意味を壊さず、Future IDをclient／serverごとに再分割もしない。

[Product Plan](../00-product/product-plan.md)の`FutureTargetClosureRegistryV1`で`single_target | target_role_bundle`をtagged unionにする。AAAは`single_target`としてTerrain／Foliage／GI／Reflectionsを同じexact Targetへ要求する。small co-op、rollback、large-session、MMO、persistence、managed external Hostはprofileごとのclient／authority／operations／Host／artifact role、exact Target集合、DAG末端まで再帰展開したactive／common Future／profile固有additional Future前提、role mapping、claimごとのrequired non-empty roleを一つのpromotion／claim release単位へ閉じる。

small co-op／rollbackの`headless_server`はdedicated profileのauthority roleだけに現れ、`future.capability.headless-dedicated-server-target`をprofile固有前提にする。MMOはdesktop clientと同じdistributed clusterにco-locateするauthority／operationsだけを候補にし、未使用のgeneric `headless_server`を候補から除く。Future 31行、Owner 59件、`planning_only`／`absent`は変えない。

### 6. Online Servicesは別将来Decisionにする

Account／platform identity／entitlement、party／lobby／matchmaking／backfill、relay allocation、fleet hosting／region placement／autoscale、cloud persistence／economy／moderationは、このDecisionでOwnerまたはProviderを採択しない。

TransportとMultiplayerは将来Serviceのopaque exact bindingを受けられるが、Lobbyをgameplay session、RelayをReplication、MatchmakerをTransport、HostingをDedicated Server packageと呼び替えない。

## Considered options

### A. 詳細設計を実装直前まで延期する

却下する。既存Future IDが異なるauthority、Target、fallback、claimを束ねたまま残り、consumer文書が各自の仮定を作る。実装前の今なら互換性costなしに正しいV1を固定できる。

### B. Render Graph、Environment、Schedulingへ追記する

却下する。Render Graphがsemantic technique、EnvironmentがTerrain／Foliage／GI、SchedulingがTransport／Authorityを所有すると、execution／domain／timing authorityが混在する。

### C. 四Ownerを追加する

採用する。Scene、LOD、geometry、runtime asset lifecycle等の既存Ownerを再利用しつつ、不在だったsemantic authorityだけを追加できる。

### D. Scene Representation、Reconstruction／Denoise、Prediction／Rollback等を独立Ownerとしてさらに追加する

却下する。Scene／LOD／geometry／asset lifecycleは既存Ownerにあり、generic PostとRender Graphも存在する。Prediction／RollbackはAuthority／Replicationの閉じたsubprofileである。Ownerを増やすとproducer／consumerではなく同じ意味の水平分割になり、AIがauthorityを一意に辿れない。

### E. GI＋Reflectionsを一つのFuture rowにする

反証監査で却下する。同じOwnerであることは同じActivation identityの根拠にならない。一TargetでGIだけqualified、Reflections unsupportedという正当な状態、各channelだけを必要とするconsumer、独立claim／fallback／降格を表現できない。一行から二Active IDをpromotion manifestで出す案も、consumerがどのsubsetを必要とするか識別する子IDが必要で、実質的に分離案と同じである。

### F. large-session shooterへrollbackを必須にする

却下する。large-sessionはsnapshot／delta／interest／interpolation等でも成立し、rollbackはlatency、determinism、state size、securityの別Product判断である。必須化するとGenre consumerの設計自由度を不必要に狭める。

### G. 全Future prerequisiteへsame-target intersectionを強制する

却下する。これはAAA等の単一Target capabilityには正しいが、large-session／MMO等のcross-target Product capabilityではDedicated Server、client、offline World、operationsのcandidate集合を空にする。candidate kindを意味なく広げず、`target_role_bundle`のexact role mappingで検証する。

### H. client／serverごとにFuture IDを増やす

却下する。個別技術CapabilityはすでにDedicated Target、Transport、Authority／Replicationへ原子化されている。Product outcomeをroleごとに再分割するとclaim release時に再結合が必要になり、Future数とProduct identityだけが増える。roleはTarget closureで表す。

## Consequences

- Owner文書は55件から59件になる。全件`review`、実装`absent`のままである。
- Future rowは26件から31件になり、全件`planning_only`のままである。
- Reflection用`claim.product.feature.production-reflections`を追加し、GI claimと一対一に分ける。
- `AAA`／`photoreal`はEnvironment capabilityでなくProduct横断quality claimになる。
- Render Graphの旧semantic profile記述とShadow Plan ownershipをAdvanced Light Transportへ移す。
- current MVP、active Capability、Target support、Phase／Work Package、release claim、Provider選定は変更しない。
- 旧複合IDと中間GI＋Reflection IDへalias、dual read、fallback lookupを残さない。

## Verification

- Architecture Indexが59 Ownerをexactに列挙し、全Header／path／文書IDが一致する。
- Product Future tableが31 unique ID、全`planning_only`、valid Owner ref、acyclic prerequisite、valid `single_target | target_role_bundle` closureを持つ。
- Future Target closureは31行とset equalityで、25行が`single_target`、6行がexact `target_role_bundle` profileを持つ。active binding、common binding、claim requirementは親行とset equality、common／profile固有Future edge unionはacyclicで、再帰展開後のbundle role Activation／mapping／claim releaseがset equalityになる。
- 旧二複合IDと中間GI＋Reflection IDが説明上の廃止記録以外のcurrent Refに残らない。
- Render Graph／Lighting／Post／Environment／World／LOD／Runtime Asset／Runtime Package／Scheduling／ECSの正本／非正本境界が新Ownerと重複しない。
- small co-opにDedicated Server、large-sessionにrollback、persistenceにgame Transportを強制していない。
- Online Serviceを暗黙採択していない。
- link、ID、Owner、claim union、Future DAG、state表現、禁止語、placeholder、diffを機械監査する。

## Official or primary sources

- [Unreal Engine Supported Features by Rendering Path](https://dev.epicgames.com/documentation/en-us/unreal-engine/supported-features-by-rendering-path-for-desktop-with-unreal-engine)
- [Unreal Engine Hardware Ray Tracing](https://dev.epicgames.com/documentation/en-us/unreal-engine/hardware-ray-tracing-in-unreal-engine)
- [Unreal Engine Lumen GI and Reflections](https://dev.epicgames.com/documentation/en-us/unreal-engine/lumen-global-illumination-and-reflections-in-unreal-engine)
- [Unreal Engine Path Tracer](https://dev.epicgames.com/documentation/en-us/unreal-engine/path-tracer-in-unreal-engine)
- [Unreal Engine Landscape Overview](https://dev.epicgames.com/documentation/unreal-engine/landscape-overview?lang=en-US)
- [Unreal Engine Networking Overview](https://dev.epicgames.com/documentation/unreal-engine/networking-overview-for-unreal-engine?lang=en-US)
- [Unreal Engine Iris](https://dev.epicgames.com/documentation/en-us/unreal-engine/introduction-to-iris-in-unreal-engine)
- [Unreal Engine Replication Graph](https://dev.epicgames.com/documentation/en-us/unreal-engine/replication-graph-in-unreal-engine)
- [Unity HDRP](https://docs.unity3d.com/ja/current/Manual/com.unity.render-pipelines.high-definition.html)
- [Unity Netcode for GameObjects](https://docs.unity3d.com/jp/current/Manual/com.unity.netcode.gameobjects.html)
- [Unity Relay integration](https://docs.unity.com/en-us/relay/integration)
- [Godot Renderers](https://docs.godotengine.org/en/stable/tutorials/rendering/renderers.html)
- [Godot Global illumination](https://docs.godotengine.org/en/stable/tutorials/3d/global_illumination/index.html)
- [Godot High-level multiplayer](https://docs.godotengine.org/en/4.7/tutorials/networking/high_level_multiplayer.html)

外部EngineのAPI、algorithm、versioned package、marketing nameをMiraikanaiのcanonical IDまたは実装要件にしない。上記資料は責務分離、support matrix、Target差、fallbackの比較根拠だけである。

このDecisionは将来Owner、責務境界、Future identityだけを決める。実装Task、実装順序、担当、工数、日程、Provider選定またはCapability Activationを決めない。
