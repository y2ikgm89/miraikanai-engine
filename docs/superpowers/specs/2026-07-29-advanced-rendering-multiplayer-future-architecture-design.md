# Miraikanai Engine Advanced Rendering／Multiplayer Future Architecture Design

- 状態: approved
- 判断日: 2026-07-29
- 対象: Advanced RenderingとMultiplayerの将来Architectureを、現Product scopeを維持したままOwner、型、data flow、failure、Qualificationまで設計閉包する判断
- 非対象: Engine／Schema／Editor／Server／Service／CI／Testの実装、実装Task、実装順序、担当、工数、日程、Provider選定、Account／Lobby／Matchmaking／Hosting／Cloud運用の採択、現在CapabilityへのActivation
- 現行状態: 関連Owner文書は`review`、実装は`absent`、将来Portfolioは`planning_only`
- 反証監査訂正: 初稿のGI＋Reflections一行／全30行は独立ActivationとTarget別claimを閉じないため撤回し、GIとReflectionsを別IDにした全31行を採用する。Owner数は四件のまま
- 最終反証監査: [Research Adjudication](../../reviews/2026-07-29-advanced-rendering-multiplayer-research-adjudication.md)に全応答と採否を保存し、current bytesの最終確認はBlocker／Major／Minor 0、到達不能／過大claim／authority重複・空白／実装scope混入 0、`公式推奨: 可`。外部監査出力をQualification／承認Receiptにはしない

## 1. 結論

Advanced RenderingとMultiplayerは、実装直前まで名前だけを残すのではなく、詳細な将来Architectureを今作成する。ただし、現行MVP、active Capability、release claim、実装DAG、Work Packageへは一切追加しない。

採用するOwner分割は次の四文書である。

| 新Owner | 正本化する意味 |
|---|---|
| `mirakan.arch.rendering-advanced-light-transport` | GI、reflection、advanced shadow、ray／path-traced light transport、scene representation requirementのsemantic declarationとavailability／completeness validation、denoise／reference比較、Target別technique／fallback |
| `mirakan.arch.rendering-terrain-foliage` | Terrain source／tile／layer／cook／streamingとFoliage species／scatter／instance／wind interaction／qualification |
| `mirakan.arch.network-transport-connection` | endpoint、protocol handshake、connection、delivery class、packetization、security binding、congestion／timeout、provider adapter、fault injection |
| `mirakan.arch.multiplayer-authority-replication` | multiplayer role／authority、network object identity、replication、RPC／command、relevancy／priority、prediction／reconciliation／rollback、join／leave／resync |

既存Ownerは次の境界を維持する。

- Render Graphはresource、pass、queue、barrier、history、submissionを実行するが、light transport方式を選ばない。
- LightingはLight Sourceの物理意味とshadow intentを所有するが、GI／reflection solverを選ばない。
- Environment／Surfacesはsky、fog、cloud、weather presentation、water、snow／wetnessを所有するが、Terrain、Foliage、GIを所有しない。
- WorldはScene／Cell／partition／activationを所有するが、Terrain／Foliage domain artifactやnetwork replicationを所有しない。
- Runtime Packageはworld／ui／headless package closureを所有し、Dedicated Server targetのpackage／launch closureを保持する。
- Scheduling／LifetimeはSimulation Advance、phase、cadence、lifetimeを所有し、network protocol、authority、replicationを所有しない。
- Product PlanはProduct intent、future incubation、cross-domain quality claimを所有するが、networkingやrenderingのdomain schemaを所有しない。

Online ServicesはMultiplayer Architectureへ暗黙統合しない。Account、identity provider、party、lobby、matchmaking、relay allocation、fleet hosting、region placement、commerce、moderation、cloud persistenceは、将来別Owner Decisionが成立するまで非正本範囲である。Transport OwnerはProvider-neutralな接続Portを、Multiplayer Ownerはexact external session bindingを受けられるが、外部Serviceの意味を再定義しない。

## 2. なぜ今設計するか

### 2.1 現状の閉包不足

Advanced RenderingにはRender Graph、Lighting、Materials、Post Processing、Environment、LOD、Virtualized Geometryが存在し、ray／path profile名もある。しかし`future.capability.production-terrain-foliage-gi`がTerrain、Foliage、GIという異なるsource、artifact、runtime、fallbackを一つのEnvironment Ownerへ束ね、`future.capability.aaa-photoreal-rendering`もEnvironmentを単独Ownerにしている。機能名は存在するが、変更authorityと個別Activationが一意でない。

Multiplayerには将来Capability行があるが専用Ownerがない。Dedicated Server、session transport、replicationがScheduling／Lifetimeへ、co-opとrollbackがProduct Planへ、large-session shooterがShooter Packへ分散している。接続、delivery、authority、network identity、state sync、prediction、join／leave、securityの型とfailure authorityを一意に辿れない。

この状態で実装を待つと、AIまたは人間は既存文書の近い節へ局所Schemaを追加し、次を起こしやすい。

- execution techniqueをauthoring semanticsとして公開する。
- EnvironmentまたはSchedulingを無関係なdomainのmega-ownerにする。
- Transportのdelivery guaranteeとReplicationのgameplay guaranteeを混同する。
- Dedicated Server、listen server、rollback、large sessionを一つのboolean capabilityにする。
- 一つのTargetで成功した高度機能を全TargetのProduct claimへ昇格する。

### 2.2 今作らないもの

詳細Architectureを作成しても、次は作成しない。

- C++ header、module、service、protocol wire format、shader、render pass
- implementation-ready backlog、task graph、milestone、優先順位、見積り
- Vendor API、Cloud、relay、hosting、graphics backendの採択
- 数値budgetの確定値
- active Capability、Product release label、MVP acceptanceへのedge
- 旧形式alias、compatibility reader、silent migration

設計は将来の選択空間を狭める実装指示ではなく、選択が置かれるOwnerと比較・Activation・fallbackの共通文法を固定する。

## 3. 比較した案

| 案 | 内容 | 利点 | 欠点 | 判断 |
|---|---|---|---|---|
| A. 既存文書へ追記 | Environment、Render Graph、Scheduling、Product Planへ詳細節を足す | 文書数が増えない | Owner重複、巨大文書、個別Activation不能、AIが近接語から誤Ownerを選ぶ | 不採用 |
| B. 薄い将来境界 | Owner名、非目標、数個の不変条件だけを追加する | 初期変更が小さい | 型、failure、cross-owner handoffが実装時まで未決定で、将来AIが再設計する | 不採用 |
| C. scope維持の詳細Future Owner | 四専用Ownerを`review`／`absent`で追加し、Portfolioだけを`planning_only`に保つ | Owner一意性、独立Activation、Target別fallback、AI可読性、既存scope維持 | 文書間linkと整合検証が増える | 採用 |

案Cは「すべてを今実装する」判断ではない。意味の閉包を先に作り、materialization、qualification、activationを別状態として残す判断である。

## 4. 状態分離とAI可読性

### 4.1 四状態を混ぜない

各新Ownerとconsumerは次を独立に表現する。

| 軸 | このChangeでの状態 |
|---|---|
| Architecture design | `review` |
| implementation／Schema materialization | `absent` |
| Product incubation | `planning_only` |
| Target qualification／Product activation | `absent` |

文書量、型候補、公式Engine比較、ChatGPT監査はimplementationまたはqualification Evidenceではない。

### 4.2 共通記述規則

新Ownerは次の順序と語彙を共通にする。

1. Headerで正本範囲、非正本範囲、規範依存、関連Owner、実装状態を示す。
2. Owner conclusionで「所有する意味」と「受け取る解決済み入力」を分ける。
3. Source／Profile、Derived Artifact、Runtime Snapshot／State、Receiptを別の型として表す。
4. closed tagged union、bounded cardinality、exact version／hash Refを使い、自由文字列、暗黙default、`latest` lookupをauthorityにしない。
5. capability、target、technique／topology、providerを直交軸として扱う。
6. producerとconsumerを一方向Refで結び、相互hash cycleを作らない。
7. unsupported、degraded、fallback、faultを別状態にし、silent quality／authority changeを許さない。
8. positive、negative、fault、scale、Target comparisonのQualificationを定義する。
9. external Engine名、Vendor機能名、marketing tierをcanonical IDにしない。
10. 同じ概念を複数Ownerで再定義せず、exact section refで参照する。

### 4.3 clean-break

関連実装と公開Schemaは`absent`であるため、正しいV1を直接設計する。既存文書内の候補名が新Owner境界と衝突する場合はOwnerごと移し、alias、dual read、近似変換を残さない。将来公開済みartifactが生じた後の変更だけがCompatibility／Evolutionのmigration対象である。

## 5. Advanced Rendering Owner境界

### 5.1 Advanced Light Transport

Advanced Light Transport Ownerは、照明結果を得る「方式の意味」とTarget別selectionを所有する。Light Source、Material BSDF、Environment radiance source、World geometry snapshotを入力として、GI、reflection、advanced shadow、path referenceの解決済み要求をRender Graphへ渡す。

正本対象は次である。

- diffuse indirect、specular indirect、visibility／occlusion、shadow transportのsemantic channel
- baked、probe、screen-space、software-ray、hardware-ray、hybrid、path-traced reference／preview／runtimeのclosed technique family
- techniqueごとのscene representation requirement宣言、必要generationのexact request、Runtime Asset／Render Graphから返却されたgenerationのcompleteness検証。ALTはgeneration identityを発行しない
- static／dynamic source、receiver、emissive、translucent／volumetric participation
- view／region／quality profileとTarget capability resolution
- temporal accumulation、denoise、history validityに必要なsemantic input
- exact fallback ladderとmeaning-equivalence class
- reference path comparison、artifact／runtime／Target qualification

非正本対象は次である。

- Light shape、unit、color、attenuation: Lighting
- Material response、shader authoring: Materials／Project Shader
- sky、fog、cloud、water、weather: Environment／Surfaces
- pass、resource、queue、barrier、acceleration build command、submission: Render Graph
- generic temporal effect composition／exposure: Post Processing
- geometry representation／LOD／residency: LOD、Virtualized Geometry、Runtime Asset Lifecycle
- device API／extension: Render Graph private Backend Adapter、各Platform

`ray tracing`は単独のProduct意味ではなくtechnique familyである。例えばreflectionだけhardware rayを使い、GIはsoftware ray、shadowはrasterを使うhybrid profileを正しく表現できなければならない。`path tracing`はreference、preview、runtimeを別profileにし、reference成功をruntime supportと表示しない。

### 5.2 Terrain／Foliage

Terrain／Foliage Ownerは、World内の大規模surfaceとvegetation distributionを一つのdomain closureとして扱うが、TerrainとFoliageを別Source、Artifact、Runtime branchとして独立にActivationできるようにする。

Terrain正本対象は次である。

- terrain root、tile／region、coordinate／bounds、height／mesh／hole representation family
- sculpt／stamp／layer／paint source operation
- surface layer binding、collision／navigation／water／snow receiver projection
- cook、seam、neighbor、streaming、edit／rebuild invalidation
- LOD／virtualized geometryへのrepresentation request
- Target budget、fallback、artifact／runtime qualification

Foliage正本対象は次である。

- species、variant、placement rule、density／mask／exclusion、seedとdeterministic scatter
- authored instanceとgenerated instanceのidentity分離
- cell／terrain／surface binding、streaming、cull／LOD request
- wind、season、wetness／snow、interaction／damageのtyped handoff
- collision／navigation／lighting participationとfallback
- Target budget、scale、artifact／runtime qualification

Worldはcell、partition、activation setを渡し、Terrain／Foliageはdomain artifactとruntime snapshotを返す。Environmentはweatherやsurface conditionを渡す。Terrain／Foliage側がWorld partition、Environment weather、Physics body、Navigation mesh、Material shaderを再定義しない。

### 5.3 AAA／photoreal quality claim

`AAA`と`photoreal`はdomain capabilityではなく、content、lighting、material、environment、VFX、animation、camera、performance、Targetの横断Product claimである。したがって`future.capability.aaa-photoreal-rendering`のOwnerはEnvironmentではなくProduct Planへ移す。

Product Planは品質rubric、Target set、blind comparison、required Owner Receiptのset equalityだけを集約する。見た目の達成手段をadvanced ray tracing、virtualized geometry、Terrain、特定Providerへ固定しない。

## 6. Multiplayer Owner境界

### 6.1 Network Transport／Connection

Network Transport／Connection Ownerは、bytesを接続相手へ運ぶ契約だけを所有する。

正本対象は次である。

- endpoint／listener／dial、connection identity、handshake、protocol revision negotiation
- connection state、keepalive、timeout、disconnect／reconnect reason
- semantic delivery class、ordering、reliability、duplication、loss、fragmentation／reassembly
- message envelope上限、packetization、MTU policy、flow／congestion／backpressure
- integrity、confidentiality、peer authentication binding、key／credential Refの消費
- native socket、platform network、relay／tunnel等のprivate Provider Adapter
- network fault injection、trace、transport telemetry、Target qualification

GameplayまたはReplicationはraw channel number、socket、packet、IP address、Provider objectを公開Contractへ保持しない。`unreliable_latest`、`unreliable_sequenced`、`reliable_ordered`、`reliable_unordered`等のsemantic delivery requirementを要求し、Transport ProfileがTarget／Providerへ写像する。

Transport成功はgameplay state convergenceを保証しない。Transport encryptionもAccount ownership、player entitlement、anti-cheat、server authorityを保証しない。

### 6.2 Multiplayer Authority／Replication

Multiplayer Authority／Replication Ownerは、「誰が何を決定し、どの状態をどの相手へいつ投影するか」を所有する。

正本対象は次である。

- dedicated server、listen server、peer authority、sharded／distributed authorityのclosed topology branch
- server、owning client、simulated client、spectator、handoff participantのrole／authority
- exact network object identityとRuntime ECS／World object binding
- spawn／despawn、property／component state、event、typed command／RPCのreplication semantics
- snapshot／delta／baseline／acknowledgement、relevancy、interest、dormancy、priority、bandwidth budget request
- authoritative tick／network tick／presentation interpolationのtime mapping
- client prediction、input buffering、reconciliation、rollback、resimulation、correction visibility
- join、leave、late join、resync、reconnect、host migration、authority handoff
- Save／Replay／Debug／Securityへのtyped handoff
- deterministic、network fault、scale、adversarial、Target qualification

wire packet、encryption、socket retryはTransportへ、Simulation Advanceとrollback可能なstate boundaryはScheduling／ECS／Persistenceへ、player input semanticsはInputへ、anti-cheat／incidentはProduct Securityへ委譲する。

RPCは任意native function callではない。versioned typed command／event、sender role、authority requirement、rate／size bound、validation rule、idempotency／ordering classを持つ。remote入力、position、inventory、combat resultを受信しただけでauthoritative stateにしない。

### 6.3 Online Servicesとの境界

Multiplayer runtimeとOnline Servicesは別問題である。

| 関心 | この設計での扱い |
|---|---|
| in-process／dedicated game simulation session | Multiplayer Authority／Replication |
| byte transport、direct／relay tunnel adapter | Network Transport／Connection |
| Account、platform identity、entitlement | 将来Online Identity Owner |
| party、lobby、matchmaking、backfill | 将来Session Discovery Owner |
| fleet hosting、region placement、autoscale | 将来Hosting／Operations Owner |
| cloud persistence、economy、moderation | 既存future portfolio＋将来専用Owner Decision |

将来Serviceを採択するまで、Transport／Multiplayer Profileはopaqueなexternal binding Refを任意入力として受けるだけにする。Lobbyをgame session authority、Relayをreplication、Matchmakerをtransportと呼び替えない。

## 7. Product Future portfolioの原子化

二つの複合Entryを七つの原子的Entryへclean-breakし、26行を31行にする。すべて`planning_only`を維持する。

### 7.1 Rendering

`future.capability.production-terrain-foliage-gi`を廃止し、次へclean-breakする。

| future capability | Owner |
|---|---|
| `future.capability.production-terrain` | `mirakan.arch.rendering-terrain-foliage` |
| `future.capability.production-foliage` | `mirakan.arch.rendering-terrain-foliage` |
| `future.capability.production-global-illumination` | `mirakan.arch.rendering-advanced-light-transport` |
| `future.capability.production-reflections` | `mirakan.arch.rendering-advanced-light-transport` |

`future.capability.aaa-photoreal-rendering`はOwnerを`mirakan.arch.product-plan`へ移し、上記四Entryを同一Target上のFuture prerequisiteにする。GIとReflectionsは同じOwnerだが相互に依存せず、Target support、fallback、Qualification、claimを独立に昇格／降格する。`claim.product.feature.production-reflections`を追加し、GI行は`claim.product.feature.production-gi`だけ、Reflections行は新claimだけを所有する。Virtualized Geometry、ray tracing、path tracing、neural reconstructionは品質目標を満たす候補であり、一律の必須手段にはしない。

### 7.2 Multiplayer

`future.capability.headless-dedicated-server-session-transport-replication`を廃止し、次へclean-breakする。

| future capability | Owner |
|---|---|
| `future.capability.headless-dedicated-server-target` | `mirakan.arch.runtime-package` |
| `future.capability.network-transport-connection` | `mirakan.arch.network-transport-connection` |
| `future.capability.multiplayer-authority-replication` | `mirakan.arch.multiplayer-authority-replication` |

Authority／Replication行自身がTransportをFuture prerequisiteにする。small co-opとrollback competitiveはTransportとAuthority／Replicationを明示前提にし、small co-opはlistenまたはdedicated topologyをProduct Decisionで選べるようにする。large-session shooterはDedicated Server、Transport、Authority／Replicationを前提にするが、rollbackを必須にしない。MMOはこの三件にoffline large Worldとpersistence／operationsを加える。persistence／operationsはOnline Service境界であり、game Transport、Replication、dedicated game Runtimeを暗黙前提にしない。Dedicated Serverが存在するだけでmultiplayer、co-op、rollback、large sessionを主張しない。

単一Target向けsame-target prerequisiteをcross-target Product consumerへ一律適用しない。[Product Plan](../../architecture/00-product/product-plan.md)の`FutureTargetClosureRegistryV1`は全31 Futureとset equalityで、25行を`single_target`、persistence、small co-op、rollback、large-session、MMO、managed external Hostの六行を`target_role_bundle`にする。bundleはexact profile一件、role→Target Profile集合、role間relation、親行とset equalityなactive／common Future前提とclaim requirement、profile固有additional Future前提、bundle前提のrole mapping、DAG末端までの再帰Activation closure、claimごとのrequired non-empty roleを一つのpromotion／claim release単位へ閉じる。

small co-op／rollbackのlisten／peer profileはDedicated Serverを要求せず、dedicated profileだけが`future.capability.headless-dedicated-server-target`を追加前提にする。large-sessionはdedicatedまたはdistributed authority profile、MMOはdesktop client＋同じdistributed cluster上のauthority／operations profileを持つ。AAAは`single_target`のまま四つのRendering前提を同じexact Targetへ要求する。Target kind集合の交差、暗黙cross product、一roleの成功からbundle claimを推測しない。

旧Future IDは公開済みCapabilityではなく`planning_only`かつ未materializeであるためaliasを残さない。

## 8. Cross-owner data flow

### 8.1 Advanced Rendering

```text
Lighting Source ─┐
Materials ───────┼─> Advanced Light Transport Profile Resolver
Environment ─────┤         │
World／Terrain ──┘         ├─> semantic transport requirements
                            ├─> scene representation requests
Target capabilities ───────┘
                                  │
                                  v
LOD／Virtual Geometry／Runtime Asset Lifecycle
                                  │ resolved representation
                                  v
Render Graph ──> Backend Adapter ──> GPU
      │
      └─> measurements／diagnostics ─> Performance／Debug／Qualification
```

Advanced Light TransportはRender Graphのpassをsource documentへ保存せず、Render GraphはGI／reflectionのauthoring意味を推測しない。

### 8.2 Multiplayer

```text
Product topology/profile
        │
        v
Multiplayer Authority／Replication
   │ commands / snapshots / deltas / acks
   v
Network Transport／Connection
   │ semantic delivery classes
   v
private socket／platform／relay adapter

Scheduling／ECS／Input／Persistence／World
   <── exact typed snapshots and commands ──>
Multiplayer Authority／Replication
```

Transportはreplicated objectを知らず、Replicationはsocket／packetを知らない。Schedulingはremote peerをphase ownerとして登録せず、Multiplayer Ownerが検証済みcommandまたはsnapshotを既存phase boundaryへ投入する。

## 9. Famous engineから採用する構造上の教訓

外部EngineのAPI名や実装を複写せず、責務分割とsupport matrixだけを比較根拠にする。

| 公式根拠 | 観察 | Miraikanaiで採用する意味 |
|---|---|---|
| [Unreal Engine: Supported Features by Rendering Path](https://dev.epicgames.com/documentation/en-us/unreal-engine/supported-features-by-rendering-path-for-desktop-with-unreal-engine) | Lumen、Nanite、VSM、TSR、Path Tracerはrendering pathごとにsupportが異なる | `advanced_rendering=true`を作らず、Target／path／technique別support matrixとfallbackを持つ |
| [Unreal Engine: Hardware Ray Tracing](https://dev.epicgames.com/documentation/en-us/unreal-engine/hardware-ray-tracing-in-unreal-engine) | standalone ray-traced shadowとLumen統合機能が併存し、software fallbackやscene costがある | ray tracingを一枚岩のfeatureにせず、semantic channelごとのtechnique選択にする |
| [Unreal Engine: Lumen GI and Reflections](https://dev.epicgames.com/documentation/en-us/unreal-engine/lumen-global-illumination-and-reflections-in-unreal-engine) | GIとreflectionsは統合される一方、geometry、world、shadow、materialと連携する | Advanced Light Transportを専用Ownerにし、各domainを消費するが所有しない |
| [Unreal Engine: Path Tracer](https://dev.epicgames.com/documentation/en-us/unreal-engine/path-tracer-in-unreal-engine) | real-time ray tracingとcodeを共有してもsupport、fallback mesh、出力用途が別である | reference／preview／runtime profileとrepresentation fallbackを分離する |
| [Unreal Engine: Landscape](https://dev.epicgames.com/documentation/unreal-engine/landscape-overview?lang=en-US) | Terrainは専用authoring systemを持ちWorld Partitionと統合する | TerrainをEnvironmentの設定節にせず、Worldと接続するdomain Ownerにする |
| [Unity: High Definition Render Pipeline](https://docs.unity3d.com/ja/current/Manual/com.unity.render-pipelines.high-definition.html) | high-fidelity pipelineはTarget hardwareとlighting architectureを明示する | quality claimとTarget capability／render profileを分離する |
| [Unity: Multiplayer overview](https://docs.unity3d.com/kr/current/Manual/multiplayer-overview.html) | GameObject系とEntities系のnetcode、client prediction、Dedicated Server、Servicesが別構成である | replication model、server target、online servicesを一つのOwnerへ押し込まない |
| [Unity: Relay integrations](https://docs.unity.com/en-us/relay/integration) | Transportは接続、信頼性、順序、fragmentationを提供し、RelayやNetcodeは別layerである | semantic Transport ContractとReplication Contractを分ける |
| [Godot: renderer overview](https://docs.godotengine.org/en/stable/tutorials/rendering/renderers.html) | Forward+、Mobile、Compatibilityでfeature setとfallback結果が異なる | Target profileごとのunsupported／degradedを明示し、起動成功を意味同等としない |
| [Godot: Global illumination](https://docs.godotengine.org/en/stable/tutorials/3d/global_illumination/index.html) | LightmapGI、VoxelGI、SDFGI等を用途と制約で選ぶ | GIを一手法へ固定せずclosed technique familyとqualificationで選ぶ |
| [Unreal Engine: Networking Overview](https://dev.epicgames.com/documentation/unreal-engine/networking-overview-for-unreal-engine?lang=en-US) | server authorityと複数replication systemを分け、connection別filter／priorityを扱う | authority、replication method、relevancy、priorityを明示profileにする |
| [Unreal Engine: Iris](https://dev.epicgames.com/documentation/en-us/unreal-engine/introduction-to-iris-in-unreal-engine) | gameplay bridge、replication state、filter、priority、serializer、data streamを分ける | gameplay stateとtransport serializationの間にtyped replication boundaryを置く |
| [Godot: High-level multiplayer](https://docs.godotengine.org/en/stable/tutorials/networking/high_level_multiplayer.html) | peer transport、RPC authority／delivery、dedicated export、authentication／server authorityが別関心である | delivery class、role authority、headless target、securityを別Owner参照で閉じる |

有名Engineが複数方式を提供している事実を、Miraikanaiが同じ方式をすべて実装する要件にはしない。採用するのは、profile／support matrix、domain separation、fallback、qualificationを明示する構造である。

## 10. Failureとfallbackの共通原則

### 10.1 Advanced Rendering

- unsupported technique、missing representation、stale artifact、Target mismatch、history invalid、budget overflowを区別する。
- hardware rayからsoftware ray、screen-space、probe、baked、disabledへの遷移はProfileに列挙し、意味差とquality classを表示する。
- path-traced referenceが成功してもruntime profileをqualifiedにしない。
- Terrain tile、Foliage scatter、GI representationの一部だけをpublishしない。
- seam、cell、representation、lighting generationのexact mismatchを近いrevisionで補修しない。
- fallbackでGameplay collision、navigation、visibility、save identityを変えない。

### 10.2 Multiplayer

- connect済み、authenticated transport、joined session、spawned、synchronized、authoritative participationを別状態にする。
- protocol mismatch、auth failure、timeout、loss、baseline欠落、desync、rollback window超過、authority lossを別diagnosticにする。
- reliable deliveryを無制限queueまたはeventual gameplay successと解釈しない。
- stale snapshot、unknown object、wrong owner、out-of-window input、rate／size超過をauthoritative stateへ適用しない。
- resyncできない場合はsilent continueせず、profileに従いspectate、disconnect、session fault等へ遷移する。
- listen server failureを自動的に別peer authorityへ移さず、qualified host migration profileがある場合だけhandoffする。

## 11. Qualification設計

### 11.1 Advanced Light Transport

- semantic sourceからresolved profile、scene representation request／返却generation completeness、Render Graph要求までのcontract fixture
- raster、screen、software-ray、hardware-ray、hybrid、path referenceのTarget support matrix
- camera cut、dynamic geometry、emissive、translucent、volumetric、large instance count、streaming境界
- history invalidation、device loss、representation欠落、budget overflow、fallback ladder
- reference outputに対するblind visual、numeric、temporal stability比較
- same Material／Lighting／Environment intentがprofile間で許容meaning classを維持すること

### 11.2 Terrain／Foliage

- tile create／edit／undo／redo／cook／load／unload／rebuild、neighbor seam
- height／mesh／hole、surface layer、water／snow／collision／navigation projection
- authored／generated foliage identity、deterministic scatter、exclusion、streaming、wind／season
- large-world scale、cell activation、LOD／virtual representation、memory pressure
- partial／stale artifact、invalid seed／mask、seam mismatch、Target fallback
- Save／Replay／Debug lineageとEditorなしのRuntime Package起動

### 11.3 Transport／Connection

- handshake、revision negotiation、connect／disconnect／reconnect、timeout
- loss、duplication、reorder、latency、jitter、fragmentation、MTU、congestion、backpressure
- delivery classごとのpositive／negative guarantee
- malformed、oversized、replayed、unauthenticated messageとcredential失効
- direct、platform、relay adapter候補のsame semantic conformance
- desktop、mobile、headless、将来web／consoleのTarget差

### 11.4 Authority／Replication

- dedicated、listen、peer、将来distributed topologyのrole／authority
- spawn／despawn、late join、relevancy、priority、dormancy、baseline／delta
- prediction、correction、rollback、resimulation、interpolation
- disconnect／reconnect／resync、qualified host migration、authority handoff
- Save／Load、Replay、World streaming、Runtime Entry transition
- malicious／out-of-order／stale command、rate abuse、desync、network partition
- small co-op、rollback fixture、large-session scaleを別Qualificationとして保持すること

共通Evidence envelope、署名、freshness、retention、revocationはAI Verification／Provenance、共通capacity measurementはPerformance／Capacity、security caseとincidentはProduct Securityだけが所有する。

## 12. Architecture正本へ反映する範囲

### 12.1 新設

- `docs/architecture/06-rendering/advanced-light-transport.md`
- `docs/architecture/06-rendering/terrain-foliage.md`
- `docs/architecture/09-networking/network-transport-connection.md`
- `docs/architecture/09-networking/multiplayer-authority-replication.md`
- `docs/architecture/decisions/2026-07-29-advanced-rendering-multiplayer-ownership.md`

`09-networking`は新しい責務domainであり、既存`08-packs`をrenameしない。番号は実装順序またはProduct優先順位を意味しない。

### 12.2 更新

- `docs/architecture/README.md`
- `docs/architecture/00-product/product-plan.md`
- `docs/architecture/01-governance/product-security.md`
- `docs/architecture/03-authoring/asset-lifecycle.md`
- `docs/architecture/04-runtime/runtime-package.md`
- `docs/architecture/04-runtime/entity-component-system.md`
- `docs/architecture/04-runtime/scheduling-lifetime.md`
- `docs/architecture/04-runtime/persistence-save.md`
- `docs/architecture/04-runtime/debugging-observability-replay.md`
- `docs/architecture/04-runtime/performance-capacity.md`
- `docs/architecture/06-rendering/render-graph.md`
- `docs/architecture/06-rendering/lighting.md`
- `docs/architecture/06-rendering/materials.md`
- `docs/architecture/06-rendering/post-processing.md`
- `docs/architecture/06-rendering/environment-surfaces.md`
- `docs/architecture/06-rendering/lod.md`
- `docs/architecture/06-rendering/virtualized-continuous-geometry.md`
- `docs/architecture/06-rendering/world.md`
- `docs/architecture/07-platform/input.md`
- `docs/architecture/08-packs/shooter.md`
- `docs/architecture/appendices/architecture-plan-closure-review.md`
- `docs/architecture/decisions/README.md`

更新はOwner link、正本／非正本境界、exact handoff、Future prerequisite、矛盾修正に限定する。新OwnerのSchemaをconsumerへ複写しない。

## 13. 完了条件

Architecture反映は次をすべて満たした場合だけ完了とする。

1. 新Owner四文書とDecisionがIndexから到達でき、Owner文書数が59件として整合する。
2. Advanced Light Transport、Terrain／Foliage、Transport／Connection、Authority／Replicationの正本範囲が重複しない。
3. Render Graph、Lighting、Environment、World、Runtime Package、Scheduling、Product Planから新Ownerへのproducer／consumer境界が一方向かつexactである。
4. 二つの複合Future IDが七つの原子的IDへclean-breakされ、全31行が`planning_only`である。
5. AAA／photoreal claimはProduct Planが集約し、特定technique／Providerを暗黙必須にしない。
6. small co-opはlisten／dedicatedの選択自由を持ち、Dedicated ServerだけでMultiplayer対応を主張しない。
7. Online ServicesをTransportまたはReplicationの一部として暗黙採択していない。
8. closed tagged union、exact Ref、bounded cardinality、failure、fallback、negative fixtureが各Ownerにある。
9. 公式Engine比較とChatGPT 5.6 Pro監査の指摘がOwner正本へ反映または根拠付きで不採用になっている。
10. 文書link、文書ID、Future ID／Owner ref、関連Owner、禁止語、状態表現の機械監査がpassする。
11. 全31 Futureにexact `single_target | target_role_bundle` closureが一件あり、active prerequisite binding、common Future binding、claim requirementが親行とset equality、common／profile固有Future edge unionがacyclic、DAG末端まで再帰展開したrole Target／bundle prerequisite／claim required role／claim releaseがset equalityである。
12. 実装、実装計画、Task、順序、担当、工数、日程、Provider選定、Product Activationを追加していない。
13. 現行MVPとcurrent Product claimsが変更されず、Multiplayerとadvanced high-end renderingはcompletion dependencyのままではない。

この文書は将来Architectureの所有判断を承認する設計であり、実装計画ではない。
