# Miraikanai Engine Architecture Plan Closure Review

- 文書ID: mirakan.appendix.architecture-plan-closure-review
- 文書種別: proposal appendix
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 親Owner: [Architecture Governance](../01-governance/architecture-governance.md)
- 正本範囲: Architecture計画全体の監査結論、current／target区分、AI可読性、Creative expression境界、Product Lifecycle／Product Security、Scene instance／override／rebase／Level authoring意味、Advanced Rendering／Multiplayer ownership、Future Target-role closure、Runtime coverage、Editor／Game分離、Target別Build／Release evidence mapping、Consent／RPG／QA／2Dのcross-owner整合性、未解決Closureの追跡
- 非正本範囲: Subsystem semantics、Schema、API、Backend、固定Budget、Product Phase／Work Package、実装Task、実装順序、担当、工数、日程、Capability Activation、承認結果
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Product Plan](../00-product/product-plan.md)
- 関連文書: [Runtime ECS Design Closure Review](runtime-ecs-design-closure-review.md)、[AI-readable Asset／Memory／Async Loading Alignment](../decisions/2026-07-28-ai-asset-memory-async-alignment.md)、[Product Lifecycle／Product Security Ownership](../decisions/2026-07-29-product-lifecycle-security-ownership.md)、[Advanced Rendering／Multiplayer Ownership](../decisions/2026-07-29-advanced-rendering-multiplayer-ownership.md)、[Product Lifecycle](../00-product/product-lifecycle.md)、[Product Security](../01-governance/product-security.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core Architecture](../02-foundation/core-architecture.md)、[Naming／Project Layout](../02-foundation/naming-project-layout.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Project State](../03-authoring/project-state.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)、[Editor Workspace／UX](../03-authoring/editor-workspace-ux.md)、[Editor UI Framework](../03-authoring/editor-ui-framework.md)、[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)、[Native Game Module](../03-authoring/native-game-module.md)、[Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[Physics](../05-simulation/physics.md)、[World](../06-rendering/world.md)、[Render Graph](../06-rendering/render-graph.md)、[Advanced Light Transport](../06-rendering/advanced-light-transport.md)、[Terrain／Foliage](../06-rendering/terrain-foliage.md)、[Network Transport／Connection](../09-networking/network-transport-connection.md)、[Multiplayer Authority／Replication](../09-networking/multiplayer-authority-replication.md)、[Windows](../07-platform/windows.md)、[Mobile Common](../07-platform/mobile-common.md)、[Android](../07-platform/android.md)、[Apple](../07-platform/apple.md)、[Gameplay Feature Packs](../08-packs/gameplay-features.md)、[RPG Genre Pack](../08-packs/rpg.md)、[Scenario／Stage](../08-packs/scenario-stage.md)
- 根拠区分: project-review／official-spec comparison
- 外部根拠確認日: 2026-07-29

> 本書は実装計画ではない。表の順序、Closure ID、推奨判断は実装順、Product Phase、Work Package、担当、工数または日程を意味しない。本書は既存Ownerの正本を置き換えず、明白な文書不整合の修正と、未決定事項を未決定のまま一意に追跡する。

## 1. 監査結論

Miraikanai EngineのArchitecture計画は、Owner分離、current／target区分、typed reference、fail-closed、Evidence、Target別Qualificationという設計原則では強い。一方、RepositoryのOwner文書は`review`、実装状態は`absent`であり、Markdown上の型、Registry、Operation、固定値をcurrent実装または利用可能なCapabilityとして扱えない。

| 観点 | target design | current状態 | 結論 |
|---|---|---|---|
| AIによる概念理解 | strong | Markdown reviewのみ | Owner、identity、revision、authority、安全境界は説明可能 |
| AIによる機械解決 | strong contract intent | incomplete | Inventory、Explain Projection、Capsule、Schema、query Toolが未materialize |
| Creative expression | broad inside public Capability | design only | 2D／3D／nonspatial／procedural、Genre非依存Gameplay、Project C++／Shaderを許す一方、任意plugin／private API／JITは意図的に除外 |
| Product lifecycle | `closed-in-target-design` | Schema／Operation／Template／Sample／Documentation／Receipt absent | bootstrap、surface parity、update、repair、support、NOTICE、release acceptanceを専用Ownerへ一意化 |
| Product security | `closed-in-target-design` | Registry／Case Store／Operation／Fixture／Receipt absent | threat ownership、baseline、vulnerability response、security update／disclosure／incidentを専用Ownerへ一意化 |
| Scene／Level authoring | `closed-in-target-design` | Schema／Projection／Operation absent | `Level Workspace`をpresentation、Scene Sourceを再利用instance／nested composition／typed override／explicit rebaseのauthorityへ分離 |
| Native iteration | `closed-in-target-design` | Native build／Preview executable／Receipt absent | Shipping static link、Preview DLLのstartup一回load、変更時GameHost restart。in-process Hot Reloadは採用しない |
| Runtime描画／物理／Memory | detailed target | implementation absent | 意味、lifetime、backend境界、Qualificationは計画済み |
| Advanced Rendering | `closed-in-target-design` | Owner／Schema review、implementation absent | Light transport semantic channelとchannel-local Technique binding、Terrain／Foliage branch別fallback outcome、Render Graph execution、World／LOD／Asset境界を分離し、GI／Reflectionsを独立Futureにした |
| Multiplayer | `closed-in-target-design` | Owner／Schema review、implementation absent | Transport／Connection、gameplay session／authority／replication、Dedicated Runtime Target、Online Servicesを分離し、Envelope whole-content identityとsession／authority epoch付きCommand deduplication domainを閉じた |
| Future Target closure | `closed-in-target-design` | Registry／Promotion／Receipt absent | 25 single Targetと6 Target-role bundleをtagged union化し、client／authority／operationsの到達不能とoptional role由来の過大claimを除去した |
| Runtime Asset | `closed-in-target-design` | implementation／Definition absent | 専用Ownerへrequest／dependency／generation／residency／lease／eviction／recoveryを一意化 |
| Editor／Game分離 | strong | implementation absent | process、state、dependency、failure isolationが一貫している |
| Target別Build mapping | strong | lock／Receipt absent | Driver、Target、Package、Signing、device Gateは計画済み |
| C++ Build最適化 | policy-level only | unresolved | Configuration名はあるが、Target別compiler／linker最適化closureが未固定 |
| 文書間連携 | structurally sound | manual inventory | Owner参照は概ね一方向だが、生成Inventoryがない |

したがって、現計画を「Architectureの方向と安全境界が閉じつつある」と評価できるが、「AIが機械的に理解・変更できる」「Runtimeが実行できる」「Target別最適化済み」とは評価しない。設計説明、Schema materialization、Runtime実装、Qualification、Product Activationを別の状態として維持する。

## 2. 監査方法と判定語彙

監査はArchitecture Index、全Owner header、Decision、proposal appendix、規範依存、関連文書、相対link、文書ID、current／target記述を対象とした。2026-07-28の監査ではArchitecture Markdown 75件、文書ID 73件、Owner文書50件を確認し、相対Markdown link切れ0件、文書ID重複0件、50 Owner間から抽出した規範依存202 edgeの未解決0件／cycle 0件を確認した。50 Owner文書はすべて`文書状態=review`、`実装状態=absent`、`検証状態=design-reviewed`である。ただし手動IndexとMarkdown解析は、生成済み`ArchitectureInventoryV1`またはSchema validationの代用ではない。

2026-07-29に[Virtualized／Continuous Geometry](../06-rendering/virtualized-continuous-geometry.md)、[Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md)、[RPG Genre Pack](../08-packs/rpg.md)を追加した。本節の75／73／50／202という数値は2026-07-28監査の履歴Evidenceであり、追加後のcurrent Inventoryまたは再監査結果ではない。同日の最初の再確認ではArchitecture Markdown 78件、文書ID 76件、Owner文書53件、相対Markdown path 2,483件の未解決0件、文書ID重複0件、Owner Header不一致0件、53 Owner間の規範依存225 edgeの未解決0件／cycle 0件を確認した。

required-features closureでは[Product Lifecycle](../00-product/product-lifecycle.md)と[Product Security](../01-governance/product-security.md)を追加した。追加後のcurrent Repository bytesではArchitecture Markdown 81件、Header内文書ID 79件、Owner文書55件、相対Markdown path 2,611件の未解決0件、文書ID重複0件、Owner Header不一致0件、55 Owner間の規範依存234 edgeの未解決0件／cycle 0件を確認した。この手動検証も生成済み`ArchitectureInventoryV1`またはSchema validationの代用にはしない。

Advanced Rendering／Multiplayer closure追加後の2026-07-29 current Repository bytesでは、Architecture Markdown 86件、Header内文書ID 84件、Owner文書59件、相対Markdown path 2,885件の未解決0件、文書ID重複0件、Owner Header不一致0件、59 Owner間の規範依存252 edgeの未解決0件／cycle 0件を確認した。Futureは31 unique行すべて`planning_only`、claim Registry 60件とFuture claim unionがset equality、Target closureは25 single／6 bundle・10 profile・21 role、common／profile固有edgeを含むFuture DAG 19 edgeはmissing／self／cycle 0件である。Profileごとのactive／common prerequisiteとclaim requirementは親Future行、candidate Target kind unionは親candidate集合とset equalityである。これらもMarkdownからの機械抽出であり、生成済みRegistry、Schema validator、Profile Artifact、Promotion Manifest、Receiptの実在証拠ではない。

本書では次の語を分ける。

| 語 | 意味 |
|---|---|
| `closed-in-target-design` | Owner文書内で意味と責務が矛盾なく定義されている |
| `corrected-in-review` | 明白な文書索引、scopeまたはmapping不整合を同じreview状態の文書で修正した |
| `open-decision` | 複数の妥当案があり、Ownerまたは採用判断をまだ固定していない |
| `open-blocker` | current化、QualificationまたはActivation前に解決が必要 |
| `unmaterialized` | target Schema／Artifact／Toolは記述されるがRepositoryに実体がない |
| `provisional` | 初回測定または判断の入力候補で、合否や製品表示に使用できない |
| `implementation-absent` | 実行binary、C++ source、generated Schema、fixture ArtifactまたはReceiptが存在しない |

文書をAIまたは人間が説明できることを、`materialized`、`qualified`、`operational`または`production`へ読み替えない。

## 3. AI可読性と外部Engine比較

### 3.1 Miraikanaiのtarget経路

AIがArchitectureおよびGame制作を理解するtarget経路は次である。

```text
Canonical Architecture／Project／Contract／Evidence
  -> immutable Inventory
  -> bounded typed Projection
  -> AiTaskContextCapsule
  -> AI read／explain／propose
  -> registered semantic Operation
  -> ChangeSetまたはEngine Candidate qualification
  -> validation／approval／publication
  -> read-back／Receipt／new revision
```

raw file、Editor widget、native object、runtime handle、pointer、allocator、GPU／OS handle、credential、任意property write、shellまたはevalをAI authorityにしない。Project mutationとEngine algorithm／layout Candidateを別branchに保ち、Project Commit ReceiptとOptimization Receiptを相互流用しない。

currentでは`ArchitectureInventoryV1`、`ArchitectureExplainProjectionV1`、`AiTaskContextCapsuleV1`、Owner別説明Projection、理解Eval、read／explain／propose Operationの全部または一部が未materialize／未Activationである。AIがMarkdownを読めることはこの不足を補わない。

### 3.2 外部Engineから採る原則

外部Engineは比較根拠であり、MiraikanaiのSchema、API、authority、既定値または互換layerではない。

| 比較対象 | 公式資料で確認した実働構造 | Miraikanaiへの示唆 | そのまま採らないもの |
|---|---|---|---|
| Unreal Engine 5.8 | Level Editor、World Partition／Data Layers、Reflection／Asset metadata、Mass、RDG、Asset Manager、Toolset Registry経由のexperimental MCP Tool | 実働するWorld authoring、typed metadata、declared resource access、async Asset handle、tool registryはAI／Runtime可読性に有効 | UObject／Actor／Mass API、reflection可視性をwrite authorityにすること、実験中MCPのAPI／data format |
| Unity 6.3 LTS／Entities 1.4／Addressables 1.21 | Scene／multi-Scene editing、GameObject／Component、`SerializedObject`、Editor extension、AI Ask／Plan／Agent、archetype chunk、async Asset handle | 実働するScene composition、Editor-owned mutation、bounded project context、data-oriented storageとruntime Asset managerの分離 | Unity object／serialization／Addressables identity、UI permissionだけをauthorityにすること、既定chunk size |
| Godot 4.7.1 | Node／Scene／Resource、text `.tscn`、EditorPlugin／`@tool`、ResourceLoader cache／threaded load、RenderingDevice | 再利用Sceneと透明なSource、軽量なEditor拡張、runtime loaderとlow-level graphics handleの分離 | pathを永続authorityにすること、Node treeをECS layoutにすること、Editor内任意codeまたはRIDをAI authorityにすること |

公開一次資料で確認できる範囲では、MiraikanaiのtargetはOwner、revision、exact hash、authorization、Evidenceの接続をより厳格にする。一方、外部Engineには実働するreflection、Asset loader、profiler、editor tool surfaceがある。Miraikanaiはtarget contractの厳格さを、未実装機能の存在証明に使わない。

一次資料:

- [Unreal Engine 5.8 Unreal MCP](https://dev.epicgames.com/documentation/unreal-engine/unreal-mcp-in-unreal-editor)
- [Unreal Engine Level Editor](https://dev.epicgames.com/documentation/en-us/unreal-engine/level-editor-in-unreal-engine)
- [Unreal Engine World Partition](https://dev.epicgames.com/documentation/unreal-engine/world-partition-in-unreal-engine?lang=en-US)
- [Unreal Engine 5.8 Render Dependency Graph](https://dev.epicgames.com/documentation/en-us/unreal-engine/render-dependency-graph-in-unreal-engine)
- [Unreal Engine 5.8 Asset Management](https://dev.epicgames.com/documentation/en-us/unreal-engine/asset-management-in-unreal-engine)
- [Unreal Engine 5.8 MassEntity overview](https://dev.epicgames.com/documentation/unreal-engine/overview-of-mass-entity-in-unreal-engine?lang=en-US)
- [Unity Entities 1.4 archetype concepts](https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/concepts-archetypes.html)
- [Unity Entities structural change optimization](https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/optimize-structural-changes.html)
- [Unity Addressables 1.21 loading](https://docs.unity3d.com/Packages/com.unity.addressables@1.21/manual/LoadingAddressableAssets.html)
- [Unity 6.3 LTS support](https://unity.com/releases/unity-6/support)
- [Unity multi-Scene editing](https://docs.unity3d.com/6000.0/Documentation/Manual/MultiSceneEditing.html)
- [Unity `SerializedObject`](https://docs.unity3d.com/6000.0/Documentation/ScriptReference/SerializedObject.html)
- [Godot release archive](https://godotengine.org/download/archive/)
- [Godot 4.7 ResourceLoader](https://docs.godotengine.org/en/4.7/classes/class_resourceloader.html)
- [Godot Editor plugins](https://docs.godotengine.org/en/stable/tutorials/plugins/editor/making_plugins.html)
- [Godot `@tool`](https://docs.godotengine.org/en/stable/tutorials/plugins/running_code_in_the_editor.html)
- [Godot 4.7 internal rendering architecture](https://docs.godotengine.org/en/4.7/engine_details/architecture/internal_rendering_architecture.html)

## 4. Runtime計画のcoverage

| Domain | closed-in-target-design | 未解決または未materialize |
|---|---|---|
| Render | private D3D12／Vulkan／Metal Adapter、Render Graph、resource state、queue、barrier、alias、generation handle、backend Qualification | backend実装、Target fixture、driver／device Receipt |
| Physics | Engine-owned World／Body、private Box2D／Jolt kernel、fixed step、substep binding、event order、worker／allocator Qualification | currentはreference `fixed 60/1`のみ。追加Substep、worker profile、Target Receiptは未Activation |
| Memory | System→Tracking→Budget→Arena／Pool、domain、lease、generation、hot-path no-fallback、telemetry | allocator実装、consumer manifest、stress／soak Receipt |
| Asset authoring | Source、typed Import、IR、deterministic Cook、immutable Artifact、Catalog、VFS、Package、atomic promotion | current採用format、Importer、Schema、Operation、Artifactが未materialize |
| Runtime Asset | Cooked Artifact request identity、dependency closure、priority／deadline／cancel、staging／activation、generation、residency、lease、eviction、retirement、device-loss recovery | Definition／Service／Port／consumer migration／Target Receiptは未materialize |
| Runtime Package | Runtime Entry、World Root／Section、integrity、staging、dependency、capacity、atomic section publication | generic Runtime Asset実行はRuntime Asset Lifecycleへ委譲。Package実装／Receiptは未materialize |
| Audio | offline Cook、resident／streamed、decode worker、PCM ring、callback no-allocation／no-lock／no-I/O | backend実装、device fixture、Receipt |
| CPU execution | shared worker pool、declarative access、scalar reference、将来のSSE／AVX／NEON candidate | ISA dispatch、topology／affinity／QoSのcross-platform採用判断とTarget Qualification |
| Performance | memory、queue、worker、frame、GPU、hitch metric、Candidate比較規則 | Reference Hardware、Benchmark executable、absolute threshold、Measurement Receipt。数値は`provisional` |

Rendering、Physics、Memory、Asset Cook、Runtime Asset、Audioは「何を守るか」が記述されている。汎用Runtime Asset request／residency authorityは専用Ownerへ一意化したが、文書は`review`、実装は`absent`である。Runtime Package、Scheduling、PerformanceはそれぞれPackage staging、phase／publication boundary、capacityを維持し、Runtime Asset Ownerへ意味を吸収されない。remaining critical blockerはRuntime Asset Definition／consumer migrationとRuntime ECS authority migrationであり、target design closureをcurrent availabilityへ読み替えない。

## 5. EditorとGame Runtimeの分離

target process modelは次を分離する。

```text
EditorHost
  -> Authoring state／Workspace／Command Gateway
  -> IPC:
     GameHost
     AI Orchestrator
     Asset／Shader／Source Worker
     Device／Package Service

Shipping Game
  -> standalone GameHost role
  -> Cook済みimmutable package＋ephemeral runtime state
  -> Editor／Authoring dependencyなし
```

分離不変条件:

1. EditorHostはrevision付きAuthoring正本を所有し、GameHostはCommit済みrevisionから生成したCook済みRuntimeだけを使用する。
2. staged draft、AI proposal、Editor object、Widget ID、native handleをRuntime packageへ混入させない。
3. Play中のtweakはtemporary Session stateであり、明示`Apply Back`から新しいProject ChangeSetをCommitするまでAuthoring正本ではない。
4. Preview DLLをEditor Processへloadしない。
5. GameHost、Project C++、Renderer、Workerまたはdevice faultでEditorHostとProject draftを終了させない。
6. Shipping dependency closureに`mirakan.editor.*`、Authoring、Workspace、Editor UIA providerを含めない。CMake graph、Named Module graph、link map、SBOMで検査する。
7. UIは`MirakanUi Core`のalgorithmとcontractだけを共有し、Widget instance、state、font cache、focus、GPU resourceをProcess間共有しない。

WindowsではEditor childのPreview executableを`mirakan_game_host.exe`、standalone Shipping executableを`mirakan_game.exe`とする。両者はScheduling OwnerのGameHost role／outer-loop contractを共有するが、binary identity、dependency closure、load方式、Package Profileを共有しない。

## 6. Target別Buildと最適化

### 6.1 閉じているBuild mapping

| Target | 正規Driver | Shipping artifact／route | Target固有closure |
|---|---|---|---|
| Windows | checked-in CMake Preset＋Ninja Multi-Config | static-linked Game、MSIXまたはmanaged layout、独立Signing | x86_64、D3D12／DXIL、GameInput／XAudio2、Package inspection |
| Android | fixed Gradle Wrapper＋CMake／Ninja | arm64 AAB、独立Signing／Upload | ABI、16 KiB page、Vulkan、ASTC primary＋ETC2 fallback、device／thermal Gate |
| Apple | Ninja C++ archive＋Xcode App shell、またはXcode Cloud | arm64 App archive／TestFlight route | C ABI／Objective-C++ bridge、Metal library、ASTC、Signing／Upload分離 |

Target、C++ Frontend、Configuration、Driver、Generator、Toolchain lockが異なるBuild tree、object、BMI、archive、log、Receiptを共有しない。Development、Profile、Shipping、ASanを近いbuild typeへ暗黙変換せず、Preview artifactをShipping directoryへcopyして昇格しない。Apple final Metal library、arm64 link、archiveはApple Workerだけが生成する。

Mobileの描画／Asset最適化は、device profile、actual Qualification、offline shader／texture Cook、明示fallbackを使う。実機不合格時はPresentation品質だけを`High → Standard → Baseline`へ下げ、Gameplay authorityを変更しない。Runtimeで汎用Texture transcodeまたはShader compilerをShipping packageへ残さない。

### 6.2 未解決のC++ Build最適化closure

`Development | Profile | Shipping | ASan`というConfiguration名は、compiler／linker flag、LTO、symbol、hardening、ISAまたは性能の証拠ではない。現在の文書にはWindows Native Game ModuleのShipping static link／LTO／symbol split方針と一部CRT／floating-point規則はあるが、全Targetを覆うexactなConfiguration optimization closureはない。

少なくとも次を一つのTarget／Configuration／Toolchain bindingとして決定するまで、「Target別最適化済みBuild」と表示しない。

- compiler／linkerのexact flag、full version、CRT／STL、exception／RTTI、visibility。
- LTOの`disabled | selected`、selected時のmode、linker、cross-module compatibility、fallback。
- symbol生成、split、strip、crash symbol retentionとPackage除外。
- security hardeningとperformance optionの組合せ。
- CPU ISA baseline、optional dispatch、unsupported hardware failure／fallback。
- PGOを採らない明示判断、または採る場合のtraining corpus、profile artifact hash、freshness、Target binding、再現性。
- binary／Package size、cold／warm startup、load、frame、memory、compile／link時間のmetricと承認済みthreshold。
- clean／incremental／cancel recovery、reproducible output、link map、SBOM、package inspection、performance comparisonのReceipt。

C++ ModulesのBuild性能比較はCX0とCX2の採否Evidenceであり、Shipping Runtimeの最適化完了または全TargetのBuild throughput保証ではない。

## 7. Cross-owner closure

| 接続 | 判定 | 一意な境界 |
|---|---|---|
| Product ↔ Runtime Owners | `corrected-in-review` | Product Owner一覧へRuntime PackageとPersistence／Saveを含める |
| Product Lifecycle ↔ Project／Build／Docs／Platform | `closed-in-target-design`／materialization absent | Lifecycleはbootstrap／update／support／NOTICE acceptance、各domainはProject／Build／Package／Documentation artifactの意味 |
| Product Security ↔ AI／Toolchain／Platform／Domain | `closed-in-target-design`／materialization absent | Securityはthreat ownership／case／update／disclosure／incident、各domainはauthorization／SBOM／signing／technical validation |
| Governance ↔ 全Owner | target aligned／Inventory absent | GovernanceはOwner／state／dependency Inventory、各OwnerはDomain fragment |
| AI Security ↔ Project／Engine change | target aligned／Operation absent | Project ChangeSetとEngine Candidate qualificationを分離 |
| Project State ↔ Editor／World context | target aligned／Schema absent | Project Stateは`AuthoringSelectionContextV1`、Worldは`WorldAuthoringContextV1`／`SceneSliceV1`、Editorはattention／Panel binding |
| Editor ↔ World ↔ Scenario／Stage | `corrected-in-review` | Level Workspaceはpresentation、Worldはcomposition／topology、Scenario／StageはEntry／Exit／Objective／finite progression |
| Scene Source ↔ instance／override／rebase ↔ Runtime | `closed-in-target-design`／materialization absent | Worldはexact Source instance／typed override／explicit rebase、Cookはlineage、RuntimeはAuthoring Source非依存 |
| Asset Lifecycle ↔ Runtime Asset ↔ Runtime Package | `closed-in-target-design`／implementation absent | AssetはSource／Cook／Catalog、Runtime Assetはrequest／generation／residency、Runtime PackageはEntry／World package closure |
| Runtime Package ↔ Scheduling | aligned | Packageはstaging／dependency、Schedulingはcompletion acceptance／publication boundary |
| Memory ↔ Runtime Resource | aligned | Memoryはgeneration／lease／allocation、Domain Ownerはpayload意味とfallback |
| Editor ↔ GameHost | `corrected-in-review` | PreviewとShippingは同じGameHost role、別binary／dependency／Package |
| Toolchain ↔ Platform Build | mapping aligned／optimization open | Driver matrixはToolchain、Package／deviceはPlatform、performance predicateはPerformance |
| Performance ↔ Target | target aligned／provisional | 同一Target／fixture／Toolchainで比較し、fresh Receiptなしに昇格しない |
| ECS ↔ Gameplay／Package／Save／AI | open-blocker | 詳細はRuntime ECS Design Closure Reviewの`ECS-C01`～`ECS-C22` |
| Consent ↔ UI／Operation／Support Bundle | `closed-in-target-design` | AI Securityがsubject／purpose／grant／deny／revoke／freshness、UIが提示、Domainが収集／redactionを所有 |
| RPG Product ↔ Feature／Genre／Project | `closed-in-target-design` | Featureは再利用State、Genreはcomposition、Reference Gameは通常Project、Productはoutcome／acceptance |
| QA ↔ CI／Release | `closed-in-target-design` | Verificationがattempt集約、retry、quarantine、waiver、非代替を所有 |
| 2D Asset／World／Animation ↔ Renderer | `closed-in-target-design` | 各Source／selection／residencyをOwnerに残し、Rendererがpacket／sort／batchを所有 |
| Lighting／Materials／World ↔ ALT ↔ Render Graph／Post | `closed-in-target-design` | Source意味は既存Owner、ALTはchannel別Technique／fallback／transport plan、Render Graphはexecution、Postはgeneric effect／history intent |
| World／LOD／Runtime Asset ↔ Terrain／Foliage | `closed-in-target-design` | Terrain／Foliageはdomain Source／Artifact、Worldはpartition、LODはselection、Runtime Assetはgeneration／residency |
| Runtime Package ↔ Transport ↔ Multiplayer | `closed-in-target-design` | PackageはDedicated Target、TransportはConnection／delivery、Multiplayerはsession／authority／replication |
| Product Future ↔ Target roles | `closed-in-target-design`／materialization absent | Product Planがsingle Targetまたはclient／authority／operations bundle、各Ownerがrole別Qualificationを所有 |

## 8. Architecture closure register

| ID | 論点 | 状態 | Owner／解決条件 |
|---|---|---|---|
| `ARCH-C01` | Product Runtime Owner一覧からRuntime Package／Persistenceが欠落 | `corrected-in-review` | Product Plan §9.2へ既存Owner linkを追加 |
| `ARCH-C02` | 汎用Runtime Asset request／residency authority | `closed-in-target-design` | 専用[Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md)へ一意化。current化にはDecision適用、Definition／Port、全consumer migration、Qualificationが必要 |
| `ARCH-C03` | Architecture Inventory／Explain Projection | `unmaterialized` | Architecture Governance。Schema、Generator、immutable Artifact、bounded query、negative fixtureを同じInventory hashへ閉じる |
| `ARCH-C04` | AI Task Capsule／read／explain／propose Operation／理解Eval | `open-blocker` | AI Security＋Verification＋各Owner。active Operationとfresh Projection／Receiptのintersectionを要求 |
| `ARCH-C05` | Runtime ECS current authority移管とECS closure | `open-blocker` | Runtime ECS Design Closure Review `ECS-C01`～`ECS-C22`を参照 |
| `ARCH-C06` | Preview／Shipping GameHost executable-role mapping | `corrected-in-review` | Windows Ownerで`mirakan_game_host.exe`と`mirakan_game.exe`を同じrole／別compositionとして明示 |
| `ARCH-C07` | Target別C++ Build optimization closure | `open-decision` | Toolchain＋Platform＋Performance。Configuration名からflags／LTO／PGO／size／performanceを推測しない |
| `ARCH-C08` | CPU ISA dispatch／topology／affinity／QoS | `open-decision` | Math＋Performance＋Platform。scalar referenceを維持し、Target別CandidateとQualificationなしに選択しない |
| `ARCH-C09` | Reference HardwareとRuntime budget | `provisional` | Performance＋Platform。Hardware Profile、Benchmark executable、absolute threshold、Measurement Receiptを固定 |
| `ARCH-C10` | 必須C++ dependency、minimum OS、CI／device pool | `open-blocker` | Toolchain Known unresolved register。候補名だけでGateを開かない |
| `ARCH-C11` | 手動Indexと長大Owner／Catalog | `design-risk` | Governance。生成Inventoryまでは手動Indexをcurrent truthと主張しない。分割はOwner／ID／anchor migrationを伴う別文書変更 |
| `ARCH-C12` | Runtime、Build、AIの実装可能性 | `implementation-absent` | 本Reviewでは解消しない。実装、実装計画、Work Package、日程を生成しない |
| `ARCH-C13` | Level Source、Region、Portal、Entry／Exit、Authoring ContextのOwner重複 | `corrected-in-review` | `Level Workspace`をpresentation限定、Region／PortalをSpace／Topologyへ解決、Entry／Exitを`StageDefinitionV1`等へ分離し、Project State／WorldのContext Ownerを明記 |
| `ARCH-C14` | ConsentがSettings／Approval／Platform declarationと混同可能 | `closed-in-target-design` | AI Security §3.3へsubject、purpose、scope、grant／deny／revoke、freshness、irreversible boundaryを一意化 |
| `ARCH-C15` | RPG Feature、Genre、Reference Game、Product outcomeのOwner境界 | `closed-in-target-design` | Gameplay Feature family、RPG Genre Pack、通常Game Project、Product acceptanceの四層へ分離 |
| `ARCH-C16` | Runtime Entry state／staging／acceptance authorityの分散 | `corrected-in-review` | Schedulingを遷移正本、Runtime Packageをstaging、Productをacceptance、Project／Persistence／UI／Stage／Worldを各projectionへ固定 |
| `ARCH-C17` | Gameplay AIのPerception→Decision→Action接続 | `closed-in-target-design` | Gameplay Programming Modelへ一方向chain、interrupt／failure、Save／Replay、causal debug境界を追加 |
| `ARCH-C18` | Test結果集約、retry、quarantine、waiver、Test class非代替 | `closed-in-target-design` | AI Verification §7.9へ集約意味とfail-closed Gateを一意化 |
| `ARCH-C19` | 共通BuildとTarget固有Package／Signing／Device／Store Evidenceの合成 | `closed-in-target-design` | Core Architecture §9.3＋Verification §7.8–7.9＋各Platform Owner |
| `ARCH-C20` | 文書、実装、検証、authority、activation、promotion、qualification、release状態の混同 | `closed-in-target-design` | Architecture Governance §2.3で直交軸とsubject-qualified語彙を固定 |
| `ARCH-C21` | AI task security以外の製品横断Security／Vulnerability governance | `closed-in-target-design` | 専用[Product Security](../01-governance/product-security.md)へthreat ownership、baseline、case、security update、disclosure、incidentを一意化。canonical hash、typed Case Registry Snapshot、strict-ancestor record DAG、severity／publication／incident branchをclosed contract化。Registry／Operation／Fixture／Receiptは未materialize |
| `ARCH-C22` | 2D Sprite／Tile／Animation／Residency／Render sort authority chain | `closed-in-target-design` | Render Graph §12.1のOwner matrix、stable sorting、batch非並替え、negative qualificationで閉じる |
| `ARCH-C23` | Project bootstrap、Template／Sample／Documentation、surface parity、update／repair／support／NOTICEの製品横断closure | `closed-in-target-design` | 専用[Product Lifecycle](../00-product/product-lifecycle.md)がversioned local Ref、acyclic Documentation Link Graph、typed parity／documentation Receipt、Support Closure、publication recovery境界を各Ownerのexact artifact／Receiptとsame release／Project／Candidateへ束縛。Schema／Operation／Artifact／Receiptは未materialize |
| `ARCH-C24` | 再利用Scene instance、nested composition、typed override、explicit rebase | `closed-in-target-design` | [World](../06-rendering/world.md)のScene Sourceをcleanな意味置換として拡張。closed Attachment Ref、versioned Rebase Ref、Instance／Source／Override／before-after byte equalityを要求し、`Prefab` canonical aliasなし、RuntimeはAuthoring Source非依存 |
| `ARCH-C25` | Mobile Application stateとsurface availability、Windows workspace rootの文書矛盾 | `corrected-in-review` | Mobile CommonをSchedulingのApplication／presentation二軸へ一致させ、Windowsからlegacy `build/` rootを除去 |
| `ARCH-C26` | GI／Reflection／advanced Shadow／Terrain／Foliageと既存Renderer／World authority | `closed-in-target-design` | [Advanced Light Transport](../06-rendering/advanced-light-transport.md)と[Terrain／Foliage](../06-rendering/terrain-foliage.md)を追加し、Source／semantic selection／execution／representation／residencyを一意化 |
| `ARCH-C27` | Dedicated Target、Transport Connection、gameplay session／authority／replicationの複合ownership | `closed-in-target-design` | Runtime Package、[Network Transport／Connection](../09-networking/network-transport-connection.md)、[Multiplayer Authority／Replication](../09-networking/multiplayer-authority-replication.md)へ三分し、Online Servicesを別Future Decisionに維持 |
| `ARCH-C28` | same-target規則でlarge-session clientとMMO全Targetが到達不能 | `closed-in-target-design` | Product Planの`FutureTargetClosureRegistryV1`で25 `single_target`／6 `target_role_bundle`、active／common／profile固有Future binding、role mapping、claimごとのrequired non-empty role、bundle claim releaseをexact化 |

`ARCH-C03`はArchitecture Inventory／Explain Projectionというsubjectのtarget semantics欠落ではなくmaterialization未実施を表す。ただしArchitecture全体で唯一のmaterialization blockerではない。`ARCH-C04`、`ARCH-C05`、`ARCH-C10`、`ARCH-C12`、ならびにLifecycle／Security Ownerが明記する未materialized Schema／Operation／Fixture／Receiptを独立に保持し、一件の状態を他件の代表にしない。

## 9. 推奨するArchitecture判断

次は実装順ではなく、未解決Authorityを閉じる際の判断条件である。

### 9.1 Runtime Asset authority

`ARCH-C02`の比較では専用Owner案を選び、[Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md)へ次を一つのtarget正本として配置した。

- request identity、Target、Catalog／Package generation、priority、deadline、cancel、idempotency。
- dependency closure、range I/O、decode／transcode、CPU／GPU upload、staging。
- activation boundary、generation、lease、resident／streamed、partial mip／LOD。
- eviction、fallback、retire、device loss、memory pressure、last-valid維持。
- queue／worker／memory／hitch charge、metric、Diagnostic、Qualification。
- Texture、Mesh、Audio、Font、World Section等のDomain固有意味を共通Managerへ吸収しないPort境界。

target Owner選択後も、Scheduling diagramの`Asset Runtime` label、Asset Lifecycleの委譲文、Runtime Packageのloader、Runtime Asset Owner文書からcurrent Manager、Service、API、Schema、CapabilityまたはOperationを推測しない。current化にはGovernance Decision、Definition Closure、consumer migration manifest、Target qualificationを要する。

### 9.2 Build optimization authority

`ARCH-C07`を閉じる判断はToolchainがexact build inputs、PlatformがABI／Package／device constraints、Performanceがmeasurement／promotion predicateを所有する三者分離を維持する。flag表だけ、benchmarkだけ、Package成功だけを最適化closureにしない。PGOは未採用を明記するか、content-addressed training／profile／freshnessを持つ採用Decisionを必要とする。

### 9.3 AI-readable Architecture

`ARCH-C03`と`ARCH-C04`を閉じるまで、AI readinessは`conceptually-readable`と`operationally-readable`を別表示にする。後者は少なくともInventory、Explain Projection、Owner fragment、Task Capsule、read Operation、理解Eval、fresh Receiptが同じArchitecture revision／Inventory hash／Contract Setへ閉じた場合だけ成立する。

### 9.4 Reference Hardwareと最適化

provisional Budgetと候補最適化は、同じTarget、Build、Toolchain、fixture、input trace、warm-up、sample count、aggregation、correctness oracleで比較する。別Target、sanitizer run、推定値、平均値、`latest`、製品deadlineからthresholdまたは採用結果を生成しない。

### 9.5 Scene／Level authoring boundary

Coreの正規SourceはWorld／Scene／Space／Topology relationであり、`Level Workspace`はこれらとowner-typed Gameplay文書を横断表示するEditor presentationである。`LevelSourceV1`、Level Stable ref、Level revision、Level membershipまたはLevel固有Operationを追加しない。

- World OwnerはWorld／Scene composition、exact Scene Source instance、acyclic nested composition、typed override、explicit rebase、Space、Anchor、Topology relation、spatial Transition intent、`WorldAuthoringContextV1`、`SceneSliceV1`を所有する。
- Project State Ownerは同じProject revisionへ閉じた`AuthoringSelectionContextV1`を所有する。
- Editor OwnerはWorkspace、attention、focus、follow／pin、Context表示を所有し、Sourceまたはauthorizationを所有しない。
- `StageDefinitionV1`等のFeature／Game OwnerはEntry／Exit、Objective、finite progressionを所有する。
- 一つのUser intentが複数Ownerを変更する場合はactiveな各Owner primitiveを一つの`ProjectChangeSetV1`へ列挙し、未Activation、staleまたはunauthorizedな一件があれば全体を部分適用しない。

外部要求の`Prefab`は`Scene Source`＋`SceneCompositionInstanceV1`へ解決し、第二のSource identity、Schema名、legacy aliasを作らない。Source更新は暗黙追従せず、before／after exact revision、override delta、typed resolution、unresolved conflictを持つ`SceneRebaseChangeV1`でのみ受理する。Cookはsource／instance／override／rebase lineageを保持するが、Runtime PackageはAuthoring Scene Sourceまたはrebase serviceへ依存しない。

この整理はOwner重複を閉じる設計修正であり、Projection Schema、Operation、Fixture、Work Packageまたは実装を作成したことを意味しない。`ARCH-C03`、`ARCH-C04`、`ARCH-C12`は引き続き未解決である。

### 9.6 Consent purpose authority

同意は[AI Security／Approval §3.3](../01-governance/ai-security-approval.md#33-consent-recordとpurpose-binding)の署名Recordだけが正本である。Settings bool、同意画面の表示、Platform privacy declaration、Task Authorization、Risk Approval、別purposeのgrantを代用しない。UIは提示と入力、各Domainは収集対象／redaction、VerificationはEvidence保持を所有し、grant／deny／revoke／freshnessの意味を複写しない。

### 9.7 RPG Product definition

RPG-first Productは、Genre非依存Core、[Reusable RPG Feature family](../08-packs/gameplay-features.md#44-reusable-rpg-feature-family)、[RPG Genre Pack](../08-packs/rpg.md)、通常のRPG Reference Game Projectへ分離する。Feature StateをGenreへ、Genre compositionをCoreへ、Reference content／balanceをProduction Packへ、Product acceptanceをRuntime schemaへ移さない。MVP outcomeとCore holdoutはProduct Planが所有し、RPG文書追加だけでCapabilityをactiveにしない。

### 9.8 QAとcross-target Release Evidence

[AI Verification §7.9](../01-governance/ai-verification-provenance.md#79-test結果集約retryquarantine)はattemptを保持し、retryで初回失敗を消さず、mandatory quarantine／Infrastructure failureをpassにしない。[Core Architecture §9.3](../02-foundation/core-architecture.md#93-cross-target-buildrelease-evidence-closure)は共通Candidate closureとTarget固有Package／Signing／Install／Launch／Physical Device／Submission Evidenceを分離する。Signing、Upload、Simulator、Headless Testのいずれか一つからRelease完了を推測しない。

### 9.9 状態軸とqualified語彙

[Architecture Governance §2.3](../01-governance/architecture-governance.md#23-状態軸とsubject-qualified語彙)の文書、実装、検証、authority、Capability activation、Artifact promotion、Target qualification、Release stateを直交させる。Closure Registerの`closed-in-target-design`はtarget Owner境界だけを表し、`accepted`、`implemented`、`active`、`qualified`、`released`を意味しない。

### 9.10 製品横断Security判断

`ARCH-C21`は専用[Product Security](../01-governance/product-security.md)を採用して`closed-in-target-design`とした。Product Securityはthreat ownership、baseline、vulnerability intake／triage／validation、remediation Candidate、security update、disclosure／notification、incident／recurrence preventionをcase全体として所有する。AI SecurityはAI authorization、Toolchainはdependency lock／SBOM、Platformはsigning／privacy、各Domainはtechnical validation、VerificationはEvidence envelope、DebuggingはSupport Bundleを維持する。

報告者の文章、scanner severity、CVSS、issue label、stale SBOMをauthorityにせず、affected／fixed／unaffected release、embargo、fix release、notification、redaction、monitoringが閉じるまでcaseをcloseしない。C++23採用を一律禁止または無条件に安全とせず、Memory／Pointers、compiler hardening、sanitizer、static analysis、fuzz、dependency isolation、unsafe boundary inventory、accepted riskをProduct Security baselineへ束縛する。Registry、Case Store、Operation、Fixture、Receiptは未materializeであり、Owner追加をoperationalなvulnerability programと表現しない。

### 9.11 Product lifecycle判断

`ARCH-C23`は専用[Product Lifecycle](../00-product/product-lifecycle.md)を採用して`closed-in-target-design`とした。Product LifecycleはEngine release binding、Project bootstrap、Template／Sample／Documentation、Editor GUI／CLI／headless parity、update／repair／support／NOTICE、E2E release acceptanceを所有し、Project State、Build／Cook／Package、Toolchain、Compatibility、Support Bundle、Evidenceのdomain meaningを複写しない。

Bootstrapは全initial DocumentとSourceを一つのprepared candidateとして検証し、成功時だけ最初のProject revisionをatomic publishする。Updateは旧release／Projectをlast-known-goodに保ったseparate CandidateでInventory、migration、validation、cook、package、launch、support、NOTICEを閉じた後だけ一回promotionする。`latest`、同名、近いversion、別Target、別Sample、partial migrationへ置換しない。Schema、Operation、Template、Sample、Documentation bundle、Fixture、Receiptは未materializeである。

### 9.12 2D runtime authority

2DはAsset import／Atlas、World Tile Source／chunk、Animation frame selection、Runtime Asset generation／lease、Material compatibility、Camera pixel policy、Renderer packet／stable sort／batch、Collision／Navigation projectionを一つのOwnerへ統合しない。Render batchingはcanonical sort orderを変えず、Tile representationの更新はSource revisionとRuntime Asset generationの一致を要求する。

### 9.13 Advanced Rendering authority

`ARCH-C26`は[Advanced Light Transport](../06-rendering/advanced-light-transport.md)と[Terrain／Foliage](../06-rendering/terrain-foliage.md)を追加して`closed-in-target-design`とした。Lighting／Materials／Environment／WorldはSource意味、ALTはdiffuse indirect／specular indirect／shadow visibility／reference transportのchannel別Technique requirement／Target support／fallbackと同channel binding validation、Terrain／Foliageは独立domain Source／Artifactとdomain完全列挙fallback outcome、LOD／Virtual Geometryはrepresentation generation、Runtime Assetはgeneration／residency、Render Graphはpass／resource／history executionを所有する。

GIとReflectionsは同じALT OwnerでもTarget support、fallback、Qualification、claimが独立するため別Future IDにする。AAA／photorealはProduct Planが同じexact Target上のTerrain／Foliage／GI／Reflections Receiptを集約するだけで、ray／path／neural／virtual geometryまたはProviderを必須手段にしない。

### 9.14 Multiplayer authority

`ARCH-C27`はDedicated game Runtime TargetをRuntime Package、endpoint／handshake／semantic delivery／Connection epoch／whole-envelope duplicate／Packet Plan reassembly identityを[Network Transport／Connection](../09-networking/network-transport-connection.md)、gameplay session／role／authority／Network Object／replication／prediction／rollback／session lifecycleとauthority epochに閉じたCommand deduplicationを[Multiplayer Authority／Replication](../09-networking/multiplayer-authority-replication.md)へ分離して`closed-in-target-design`とした。

Transport `connected`をsession `joined | synchronized | authoritative`へ読み替えず、Dedicated TargetだけでMultiplayer claimを解放しない。Account／entitlement、Lobby／Matchmaking、Relay allocation、Hosting fleet、cloud persistence／economy／moderationは別将来Decisionであり、opaque binding以外を二Ownerへ吸収しない。

### 9.15 Future Target-role closure

`ARCH-C28`は単一Target prerequisiteとcross-target Product compositionを`single_target | target_role_bundle`へ分けて`closed-in-target-design`とした。AAA等25行はsame exact Target、persistence、small co-op、rollback、large-session、MMO、managed external Hostの六行はexact profile、role Target集合、全role pair relation、DAG末端まで再帰展開するactive／common Future／profile固有additional Future前提、bundle前提role mapping、claimごとのrequired non-empty roleを一つのpromotion／claim release単位にする。

small co-op／rollbackのheadless Targetはdedicated profileだけが選べ、Dedicated Target Futureをprofile固有前提にする。large-sessionのclient Targetとauthority Target、MMOのdesktop clientとdistributed authority／operationsをflat intersectionへ潰さない。一roleのReceipt、Target kindだけの一致、別profile間の混成、implicit cross product、optional artifact roleが空のmanaged Source buildからProduct claimを解放しない。Registry、Profile Artifact、Promotion Manifest、Activation、Receiptは未materializeである。

## 10. 本Reviewに伴う文書整合

| 文書 | 変更内容 |
|---|---|
| Architecture Index | Owner文書を59件へ更新し、Advanced Light Transport／Terrain・Foliage／Network Transport／Multiplayer AuthorityとOwnership Decisionへのnavigationを追加 |
| Product Lifecycle | Project bootstrap、Template／Sample／Documentation、surface parity、update／repair／support／NOTICE、E2E acceptanceの専用Ownerを追加 |
| Product Security | threat ownership、baseline、vulnerability case、security update／disclosure／notification／incidentの専用Ownerを追加 |
| Product Plan | Product completionへLifecycle／Security acceptanceを追加し、各専用Ownerとの境界を明記 |
| Product Plan／Naming | Creative expressionとextensionを別axis化し、Level Workspaceを非authorityのEditor presentationへ固定 |
| Project State | `AuthoringSelectionContextV1`のProject lineage、target解決、revision／hash／invalidation Ownerを明記 |
| World | Level／Region／Portal表示語の正規解決、World authoring Projection、Scene instance／nested composition／typed override／explicit rebase、cross-owner atomicityを明記 |
| Editor Workspace／Panel／Design System | Level Stable ref／membership／`SetLevel*`を除去し、World Compositionとpack-owned bindingへ分離 |
| Executable Contracts planning catalog | Level WorkspaceがOperation familyでないこと、複数Owner Operationの部分fallback禁止を明記 |
| Scenario／Stage、LOD、Materials、World fixture | `Level Source`をcurrent Core identityとして扱う残存記述をlegacy migrationまたはWorld／Scene／Topology表現へ修正 |
| Runtime Asset Lifecycle | 汎用request、dependency、generation、residency、lease、eviction、recoveryのtarget Ownerを追加 |
| Asset Lifecycle／Runtime Package／Scheduling | Source／Cook、Runtime Entry staging、phase／publicationとRuntime Asset authorityの境界を更新 |
| AI Security／UI／Debug | Consent Record、purpose binding、提示、Support Bundle消費境界を統合 |
| AI Verification／Core Architecture | Test集約／retry／quarantine／waiverとcross-target Build／Release Evidence closureを追加 |
| Architecture Governance／Naming | 直交状態軸、subject-qualified状態語彙、Package／Pack／Bundleのqualified名称を追加 |
| Gameplay Programming Model | Perception→Decision→Action、interrupt、failure、Save／Replay、causal debug接続を追加 |
| Gameplay Feature Packs／RPG Genre Pack／Product Plan | Reusable RPG Feature、Genre composition、通常Reference Game、Product acceptanceを四層化 |
| Runtime ECS | all-or-nothing authority migration、Consumer Inventory、旧authority retirement条件を追加 |
| Render Graph／World／Animation／Runtime Asset | Sprite／Tile／frame／residency／packet／stable sort／batchの2D authority chainを追加 |
| Native Game Module | restart-based Windows PreviewをHot Reloadのcleanな意味置換として明記し、in-process unload／replacement／patch禁止を維持 |
| Windows／Mobile Common | legacy `build/` rootを除去し、Application lifecycleとpresentation surface availabilityを別stateへ修正 |
| Toolchain／Dependencies | Target別C++ Build optimization closureをKnown unresolved registerへ追加 |
| Advanced Light Transport／Render Graph／Lighting／Materials／Post／Environment | Source意味、channel別Technique／fallback、generic effect、physical executionのauthorityを分離 |
| Terrain／Foliage／World／LOD／Runtime Asset | TerrainとFoliageのSource／Artifact branchを独立させ、partition／selection／residencyを既存Ownerへ維持 |
| Runtime Package／Network Transport／Multiplayer | Dedicated Target、Connection／delivery、gameplay session／authority／replicationを三分し、Online Servicesを非正本化 |
| Product Plan／Execution Registry proposal／Shooter | Future 31行、Reflection claim、Target-role bundle、active／common／profile固有Future prerequisite、claim required role、promotion／claim release closureを追加 |

## 11. 非目標

- 本ReviewからC++ source、CMake target、Schema、Generator、Operation、Fixture、Receiptを生成しない。
- Runtime Asset Manager、Consent Registry、RPG Pack Registry、QA aggregator、Build Optimization ProfileまたはAI Toolを仮実装しない。
- Product Lifecycle／Product SecurityのOwner追加からSchema、Operation、Template、Sample、Case Store、Fixture、Receiptまたは運用体制を生成済みとみなさない。
- Product Phase、Work Package、実装順、担当、工数、日程を追加しない。
- review文書を`normative`、Capabilityを`active | qualified | production`へ昇格しない。
- 外部EngineのAPI、object model、chunk size、Asset identity、path、default、tool permissionをMiraikanaiの正本へ移植しない。
- Plugin ecosystem、汎用Script VM／JIT、Multiplayer、large-world機能をrequired-featuresの名称coverageだけでMVPへ追加しない。
