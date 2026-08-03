# Miraikanai Engine Architecture Plan Closure Review

- 文書ID: mirakan.appendix.architecture-plan-closure-review
- 文書種別: proposal appendix
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 親Owner: [Architecture Governance](../01-governance/architecture-governance.md)
- 正本範囲: Architecture計画全体の監査結論、current／target区分、AI可読性、AI-native C++ Product identity／外部Engine非模倣境界、Game production／Playtest iteration／AI generation claim／First Playable、Minimum Executable Core、Creative expression境界、Product Lifecycle／Privacy／Security／Release Decision／Publication、Scene instance／override／rebase／Level authoring意味、Advanced Rendering／Multiplayer ownership、Future Target-role closure、Runtime coverage、Editor／Game分離、Target別Build／Release evidence mapping、Consent／RPG／QA／2D／Save Catalog／reference integrityのcross-owner整合性、未解決Closureの追跡
- 非正本範囲: Subsystem semantics、Schema、API、Backend、固定Budget、Product Phase／Work Package、実装Task、実装順序、担当、工数、日程、Capability Activation、承認結果
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Product Plan](../00-product/product-plan.md)
- 関連文書: [Runtime ECS Design Closure Review](runtime-ecs-design-closure-review.md)、[Runtime ECS Static Definition／Entity Reference Boundary](../decisions/2026-08-03-runtime-ecs-static-and-entity-reference-boundary.md)、[Initial Morph Capability Boundary](../decisions/2026-08-03-initial-morph-capability-boundary.md)、[glTF Import Dependency Baseline](../decisions/2026-08-03-gltf-import-dependency-baseline.md)、[MCP Current Protocol Baseline](../decisions/2026-08-03-mcp-current-protocol-baseline.md)、[Android Adaptive Game Window Baseline](../decisions/2026-08-03-android-adaptive-game-window-baseline.md)、[AI-readable Asset／Memory／Async Loading Alignment](../decisions/2026-07-28-ai-asset-memory-async-alignment.md)、[AI-native C++ Product Identity](../decisions/2026-08-03-ai-native-cpp-product-identity.md)、[Product Lifecycle／Product Security Ownership](../decisions/2026-07-29-product-lifecycle-security-ownership.md)、[Product Release／Publication Authority Ownership](../decisions/2026-07-30-product-release-publication-authority.md)、[Advanced Rendering／Multiplayer Ownership](../decisions/2026-07-29-advanced-rendering-multiplayer-ownership.md)、[Product Lifecycle](../00-product/product-lifecycle.md)、[Product Release Decision](../00-product/product-release-decision.md)、[Product Publication／Completion](../00-product/product-publication-completion.md)、[Product Privacy／Data Governance](../01-governance/product-privacy-data-governance.md)、[Product Security](../01-governance/product-security.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core Architecture](../02-foundation/core-architecture.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Naming／Project Layout](../02-foundation/naming-project-layout.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Project State](../03-authoring/project-state.md)、[Game Production Loop](../03-authoring/game-production-loop.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)、[Developer Testing](../03-authoring/developer-testing.md)、[Editor Workspace／UX](../03-authoring/editor-workspace-ux.md)、[Editor UI Framework](../03-authoring/editor-ui-framework.md)、[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)、[Native Game Module](../03-authoring/native-game-module.md)、[Persistence／Save](../04-runtime/persistence-save.md)、[Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[Physics](../05-simulation/physics.md)、[World](../06-rendering/world.md)、[Render Graph](../06-rendering/render-graph.md)、[Materials](../06-rendering/materials.md)、[Advanced Light Transport](../06-rendering/advanced-light-transport.md)、[Terrain／Foliage](../06-rendering/terrain-foliage.md)、[Network Transport／Connection](../09-networking/network-transport-connection.md)、[Multiplayer Authority／Replication](../09-networking/multiplayer-authority-replication.md)、[Windows](../07-platform/windows.md)、[Mobile Common](../07-platform/mobile-common.md)、[Android](../07-platform/android.md)、[Apple](../07-platform/apple.md)、[UI／Text](../07-platform/ui-text-localization-accessibility.md)、[Gameplay Feature Packs](../08-packs/gameplay-features.md)、[RPG Genre Pack](../08-packs/rpg.md)、[Scenario／Stage](../08-packs/scenario-stage.md)
- 根拠区分: project-review／official-spec comparison
- 外部根拠確認日: 2026-08-03

> 本書は実装計画ではない。表の順序、Closure ID、推奨判断は実装順、Product Phase、Work Package、担当、工数または日程を意味しない。本書は既存Ownerの正本を置き換えず、明白な文書不整合の修正と、未決定事項を未決定のまま一意に追跡する。

## 1. 監査結論

Miraikanai EngineのArchitecture計画は、Owner分離、current／target区分、typed reference、fail-closed、Evidence、Target別Qualificationという設計原則では強い。一方、RepositoryのOwner文書は`review`、実装状態は`absent`であり、Markdown上の型、Registry、Operation、固定値をcurrent実装または利用可能なCapabilityとして扱えない。

| 観点 | target design | current状態 | 結論 |
|---|---|---|---|
| AIによる概念理解 | strong | Markdown reviewのみ | Owner、identity、revision、authority、安全境界は説明可能 |
| AIによる機械解決 | strong contract intent | incomplete | Inventory、Explain Projection、Capsule、Schema、query Toolが未materialize |
| AI-native Game production loop | `closed-in-target-design` | Owner／Operation／Fixture／Receipt／Project absent | Intent→理解closure→staging→test／playtest→evaluation→iteration→acceptanceを同じCandidate／Project lineageへ閉じ、manual／AI continuityを分離せず接続 |
| First Playable | `closed-in-target-design` | compact RPG execution projection／Evidence absent | receipt-free exact 2D command RPG Definition、AI claim lane、manual／AI journey、Playtest／Approvalを定義し、Shooter／3D／別Genreを代用しない |
| Minimum Executable Core | `closed-in-target-design` | C++／Build／Package／Qualification absent | 30 exact roleと九scenarioでworldless headless boot、determinism、cancel／fault／shutdown、sanitizer／negative inputを同一Candidateへ閉じる |
| Creative expression | broad inside public Capability | design only | 2D／3D／nonspatial／procedural、Genre非依存Gameplay、Project C++／Shaderを許す一方、任意plugin／private API／JITは意図的に除外 |
| Product lifecycle | `closed-in-target-design` | Schema／Operation／Template／Sample／Documentation／Receipt absent | bootstrap、surface parity、update、repair、support、NOTICE、release acceptanceを専用Ownerへ一意化 |
| Product security | `closed-in-target-design` | Registry／Case Store／Operation／Fixture／Receipt absent | threat ownership、baseline、vulnerability response、security update／disclosure／incidentを専用Ownerへ一意化 |
| Scene／Level authoring | `closed-in-target-design` | Schema／Projection／Operation absent | `Level Workspace`をpresentation、Scene Sourceを再利用instance／nested composition／typed override／explicit rebaseのauthorityへ分離 |
| Native iteration | `closed-in-target-design` | Native build／Preview executable／Receipt absent | Shipping static link、Preview DLLのstartup一回load、変更時GameHost restart。in-process Hot Reloadは採用しない |
| Runtime描画／物理／Memory | detailed target | implementation absent | 意味、lifetime、backend境界、Qualificationは計画済み |
| Advanced Rendering | `closed-in-target-design` | Owner／Schema review、implementation absent | Light transport semantic channelとchannel-local Technique binding、Terrain／Foliage branch別fallback outcome、Render Graph execution、World／LOD／Asset境界を分離し、GI／Reflectionsを独立Futureにした |
| Multiplayer | `closed-in-target-design` in Future scope | Owner／Schema review、implementation absent | Transport／Connection、Provider generation／peer exchange／pair identity、gameplay session／authority／replication、Dedicated Runtime Target、Online Servicesを分離。current C1／C2ではNetworkingをactivateしない |
| Future Target closure | `closed-in-target-design` | Registry／Promotion／Receipt absent | 25 single Targetと6 Target-role bundleをtagged union化し、client／authority／operationsの到達不能とoptional role由来の過大claimを除去した |
| Runtime Asset | `closed-in-target-design` | implementation／Definition absent | 専用Ownerへrequest／dependency／generation／residency／lease／eviction／recoveryを一意化 |
| Editor／Game分離 | strong | implementation absent | process、state、dependency、failure isolationが一貫している |
| Target別Build mapping | strong | lock／Receipt absent | Driver、Target、Package、Signing、device Gateは計画済み |
| C++ Build最適化 | `closed-in-target-design` | policy materialization／measurement absent | Toolchain Ownerが全Target×Configurationのexact compiler／linker／LTO／symbol／hardening／ISA policyを閉じる。選択済みpolicyを最適化済み実測へ読み替えない |
| 文書間連携 | structurally sound | manual inventory | Owner参照は概ね一方向だが、生成Inventoryがない |

したがって、現計画を「Architectureの方向と安全境界が閉じつつある」と評価できるが、「AIが機械的に理解・変更できる」「Runtimeが実行できる」「Target別最適化済み」とは評価しない。設計説明、Schema materialization、Runtime実装、Qualification、Product Activationを別の状態として維持する。

## 2. 監査方法と判定語彙

監査はArchitecture Index、全Owner header、Decision、proposal appendix、規範依存、関連文書、相対link、文書ID、current／target記述を対象とした。2026-07-28の監査ではArchitecture Markdown 75件、文書ID 73件、Owner文書50件を確認し、相対Markdown link切れ0件、文書ID重複0件、50 Owner間から抽出した規範依存202 edgeの未解決0件／cycle 0件を確認した。50 Owner文書はすべて`文書状態=review`、`実装状態=absent`、`検証状態=design-reviewed`である。ただし手動IndexとMarkdown解析は、生成済み`ArchitectureInventoryV1`またはSchema validationの代用ではない。

2026-07-29に[Virtualized／Continuous Geometry](../06-rendering/virtualized-continuous-geometry.md)、[Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md)、[RPG Genre Pack](../08-packs/rpg.md)を追加した。本節の75／73／50／202という数値は2026-07-28監査の履歴Evidenceであり、追加後のcurrent Inventoryまたは再監査結果ではない。同日の最初の再確認ではArchitecture Markdown 78件、文書ID 76件、Owner文書53件、相対Markdown path 2,483件の未解決0件、文書ID重複0件、Owner Header不一致0件、53 Owner間の規範依存225 edgeの未解決0件／cycle 0件を確認した。

required-features closureでは[Product Lifecycle](../00-product/product-lifecycle.md)と[Product Security](../01-governance/product-security.md)を追加した。追加後のcurrent Repository bytesではArchitecture Markdown 81件、Header内文書ID 79件、Owner文書55件、相対Markdown path 2,611件の未解決0件、文書ID重複0件、Owner Header不一致0件、55 Owner間の規範依存234 edgeの未解決0件／cycle 0件を確認した。この手動検証も生成済み`ArchitectureInventoryV1`またはSchema validationの代用にはしない。

Advanced Rendering／Multiplayer closure追加後の2026-07-29 current Repository bytesでは、Architecture Markdown 86件、Header内文書ID 84件、Owner文書59件、相対Markdown path 2,885件の未解決0件、文書ID重複0件、Owner Header不一致0件、59 Owner間の規範依存252 edgeの未解決0件／cycle 0件を確認した。Futureは31 unique行すべて`planning_only`、claim Registry 60件とFuture claim unionがset equality、Target closureは25 single／6 bundle・10 profile・21 role、common／profile固有edgeを含むFuture DAG 19 edgeはmissing／self／cycle 0件である。Profileごとのactive／common prerequisiteとclaim requirementは親Future行、candidate Target kind unionは親candidate集合とset equalityである。これらもMarkdownからの機械抽出であり、生成済みRegistry、Schema validator、Profile Artifact、Promotion Manifest、Receiptの実在証拠ではない。

Privacy、Developer Testing、Release Decision／Publication Owner、R10～R15 closure、公式current AGP再確認の反映後となる2026-07-30 current Repository bytesでは、Architecture Markdown 93件、文書ID91件／unique 91件（IDを持たないNavigation用README 2件を除く）、Owner文書63件、相対Markdown link 2,964件の未解決0件、fragment link 290件の未解決0件、Owner Header不一致0件、規範依存317 edgeのOwner外参照0件／cycle 0件、unbalanced code fence 0件を確認した。この手動抽出はSchema validator、generated Inventory、materialized Requirement／Journey Projection、Decision、Platform Receipt、PublicationまたはCompletionの代用ではない。

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
| Runtime Asset | Cooked Artifact request identity、dependency closure、priority／deadline／cancel、staging／activation、generation、residency、lease、eviction、retirement、device-loss recovery | Definition／Service／Port／Target Receiptは未materialize |
| Runtime Package | Runtime Entry、World Root／Section、integrity、staging、dependency、capacity、atomic section publication | generic Runtime Asset実行はRuntime Asset Lifecycleへ委譲。Package実装／Receiptは未materialize |
| Audio | offline Cook、resident／streamed、decode worker、PCM ring、callback no-allocation／no-lock／no-I/O | backend実装、device fixture、Receipt |
| CPU execution | shared worker pool、declarative access、scalar reference、将来のSSE／AVX／NEON candidate | ISA dispatch、topology／affinity／QoSのcross-platform採用判断とTarget Qualification |
| Performance | memory、queue、worker、frame、GPU、hitch metric、Candidate比較規則 | Reference Hardware、Benchmark executable、absolute threshold、Measurement Receipt。数値は`provisional` |

Rendering、Physics、Memory、Asset Cook、Runtime Asset、Audioは「何を守るか」が記述されている。汎用Runtime Asset request／residency authorityはinitial V1の専用Ownerへ一意化したが、文書は`review`、実装は`absent`である。Runtime Package、Scheduling、PerformanceはそれぞれPackage staging、phase／publication boundary、capacityを維持し、Runtime Asset Ownerへ意味を吸収されない。remaining blockerはRuntime Asset／Runtime ECSのmachine-readable Definition Closure、Fixture、Qualificationであり、design closureをcurrent availabilityへ読み替えない。

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
6. Shipping dependency closureに`mirakan.editor.*`、Authoring、Workspace、Editor UIA providerを含めない。CMake graph、public／private include graph、link map、SBOMで検査する。
7. UIは`MirakanUi Core`のalgorithmとcontractだけを共有し、Widget instance、state、font cache、focus、GPU resourceをProcess間共有しない。

WindowsではEditor childのPreview executableを`mirakan_game_host.exe`、standalone Shipping executableを`mirakan_game.exe`とする。両者はScheduling OwnerのGameHost role／outer-loop contractを共有するが、binary identity、dependency closure、load方式、Package Profileを共有しない。

## 6. Target別Buildと最適化

### 6.1 閉じているBuild mapping

| Target | 正規Driver | Shipping artifact／route | Target固有closure |
|---|---|---|---|
| Windows | checked-in CMake Preset＋Ninja Multi-Config | static-linked Game、MSIXまたはmanaged layout、独立Signing | x86_64、D3D12／DXIL、GameInput／XAudio2、Package inspection |
| Android | fixed Gradle Wrapper＋CMake／Ninja | arm64 AAB、独立Signing／Upload | ABI、16 KiB page、Vulkan、ASTC primary＋ETC2 fallback、device／thermal Gate |
| Apple | Ninja C++ archive＋Xcode App shell、またはXcode Cloud | arm64 App archive／TestFlight route | C ABI／Objective-C++ bridge、Metal library、ASTC、Signing／Upload分離 |

Target、C++ Frontend、Configuration、Driver、Generator、Toolchain lockが異なるBuild tree、object、PCH、compiler cache、archive、log、Receiptを共有しない。Named Module、Header Unit、BMIは生成または配布しない。Development、Profile、Shipping、ASanを近いbuild typeへ暗黙変換せず、Preview artifactをShipping directoryへcopyして昇格しない。Apple final Metal library、arm64 link、archiveはApple Workerだけが生成する。

Mobileの描画／Asset最適化は、device profile、actual Qualification、offline shader／texture Cook、明示fallbackを使う。実機不合格時はPresentation品質だけを`High → Standard → Baseline`へ下げ、Gameplay authorityを変更しない。Runtimeで汎用Texture transcodeまたはShader compilerをShipping packageへ残さない。

### 6.2 C++ Build最適化closure

`Development | Test | Profile | Shipping`というConfiguration名だけは、compiler／linker flag、LTO、symbol、hardening、ISAまたは性能の証拠ではない。[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)の`TargetConfigurationBuildPolicyV1`が全Target×Configurationのexact compile／link argument、exception／RTTI、visibility、LTO、sanitizer、symbol、hardening、ISA baseline／dispatch、PGO不採用をinitial V1 targetとして所有する。Schema、Build、Receipt、性能測定は未materializeであり、文書上のtarget closureを実測済み最適化へ読み替えない。

次はTarget designとして固定済みの事項と、materialization／measurementで初めて証明できる事項を分離する。

- compiler／linkerのexact flag、full version、CRT／STL、exception／RTTI、visibilityはToolchain OwnerのProfileとPolicyを正本にする。
- LTO、symbol、strip、crash symbol retention、security hardening、CPU ISA baseline／optional dispatch、PGO不採用は同Policyを正本にする。
- binary／Package size、cold／warm startup、load、frame、memory、compile／link時間のmetricと承認済みthreshold。
- clean／incremental／cancel recovery、reproducible output、link map、SBOM、package inspection、performance comparisonのReceipt。

C++23 initial V1はHeader-based Shipping public surfaceであり、Named Modules、header units、BMI、`import std`はrequired universe外である。将来のNamed Modules比較はsuccessor ADRなしにcurrent optimization Evidenceへ混ぜず、Header buildの成功もShipping Runtime最適化完了または全Target Build throughput保証へ読み替えない。

## 7. Cross-owner closure

| 接続 | 判定 | 一意な境界 |
|---|---|---|
| Product ↔ Runtime Owners | `corrected-in-review` | Product Owner一覧へRuntime PackageとPersistence／Saveを含める |
| Product Lifecycle ↔ Project／Build／Docs／Platform | `closed-in-target-design`／materialization absent | Lifecycleはbootstrap／update／support／NOTICE acceptance、各domainはProject／Build／Package／Documentation artifactの意味 |
| Product Security ↔ AI／Toolchain／Platform／Domain | `closed-in-target-design`／materialization absent | Securityはthreat ownership／case／update／disclosure／incident、各domainはauthorization／SBOM／signing／technical validation |
| Product Definition ↔ Requirement Projection ↔ Release Decision | `closed-in-target-design`／materialization absent | Planはreceipt-free required universe、Release Decisionはqualified authority／quorum／signature／freshness／revocation／current state |
| Release Decision ↔ Platform Receipt ↔ Publication／Completion | `closed-in-target-design`／materialization absent | Platformはcurrent signed Decisionを消費し、Publicationがpublic read-backを下流集約してsupport開始／Completionを閉じる |
| UI Settings marker ↔ Persistence Save Catalog／Load | `closed-in-target-design`／materialization absent | UIはSettingsとのatomic co-publication、PersistenceはCatalog content identity／Slot membership／active-root precondition／Load Request |
| Governance ↔ 全Owner | target aligned／Inventory absent | GovernanceはOwner／state／dependency Inventory、各OwnerはDomain fragment |
| AI Security ↔ Project／Engine change | target aligned／Operation absent | Project ChangeSetとEngine Candidate qualificationを分離 |
| Project State ↔ Editor／World context | target aligned／Schema absent | Project Stateは`AuthoringSelectionContextV1`、Worldは`WorldAuthoringContextV1`／`SceneSliceV1`、Editorはattention／Panel binding |
| Editor ↔ World ↔ Scenario／Stage | `corrected-in-review` | Level Workspaceはpresentation、Worldはcomposition／topology、Scenario／StageはEntry／Exit／Objective／finite progression |
| Scene Source ↔ instance／override／rebase ↔ Runtime | `closed-in-target-design`／materialization absent | Worldはexact Source instance／typed override／explicit rebase、Cookはlineage、RuntimeはAuthoring Source非依存 |
| Asset Lifecycle ↔ Runtime Asset ↔ Runtime Package | `closed-in-target-design`／implementation absent | AssetはSource／Cook／Catalog、Runtime Assetはrequest／generation／residency、Runtime PackageはEntry／World package closure |
| Runtime Package ↔ Scheduling | aligned | Packageはstaging／dependency、Schedulingはcompletion acceptance／publication boundary |
| Memory ↔ Runtime Resource | aligned | Memoryはgeneration／lease／allocation、Domain Ownerはpayload意味とfallback |
| Editor ↔ GameHost | `corrected-in-review` | PreviewとShippingは同じGameHost role、別binary／dependency／Package |
| Toolchain ↔ Platform Build | `closed-in-target-design`／materialization・measurement absent | Language／Driver／Target×ConfigurationとMCP singleton baselineはToolchain、Package／deviceはPlatform、performance predicateはPerformance。KTX／FLAC component evidenceとMCP implementation／Conformanceは未materialize |
| Performance ↔ Target | target aligned／provisional | 同一Target／fixture／Toolchainで比較し、fresh Receiptなしに昇格しない |
| ECS ↔ Gameplay／Package／Save／AI | `closed-in-target-design`／materialization absent | static Manifestは`TickPhaseId`、runtime leaseはexact Time Ref、Entity Refはsnapshot-bound／live delivery、durable identityはPersistenceへ分離 |
| Consent ↔ UI／Operation／Support Bundle | `closed-in-target-design` | AI Securityがsubject／purpose／grant／deny／revoke／freshness、UIが提示、Domainが収集／redactionを所有 |
| RPG Product ↔ Feature／Genre／Project | Owner境界`closed-in-target-design`／Execution projection absent | Featureは再利用State、Genreはcomposition、Reference Gameは通常Project、Productはoutcome／acceptance。Execution ProposalはShooterをRPG Evidenceへ投影せず、RPG Fixture／WP／Capability／Gateは実装計画段階まで未登録 |
| QA ↔ CI／Release | `closed-in-target-design` | Verificationがattempt集約、retry、quarantine、waiver、非代替を所有 |
| 2D Asset／World／Animation ↔ Renderer | `closed-in-target-design` | 各Source／selection／residencyをOwnerに残し、Rendererがpacket／sort／batchを所有 |
| Lighting／Materials／World ↔ ALT ↔ Render Graph／Post | `closed-in-target-design` | Source意味は既存Owner、ALTはchannel別Technique／fallback／transport plan、Render Graphはexecution、Postはgeneric effect／history intent |
| World／LOD／Runtime Asset ↔ Terrain／Foliage | `closed-in-target-design` | Terrain／Foliageはdomain Source／Artifact、Worldはpartition、LODはselection、Runtime Assetはgeneration／residency |
| Runtime Package ↔ Transport ↔ Multiplayer | `closed-in-target-design` in Future scope／implementation absent | PackageはDedicated Target、TransportはConnection／deliveryとGateway-owned Provider generation、Multiplayerはsession／authority／replicationへ分離 |
| Asset Lifecycle ↔ Animation ↔ Rendering Morph | `closed-in-target-design`／Future unmaterialized | Morphをinitial V1／C1／C2から除外し、Source検出をfail closedにする。Future採択はSource／track／runtime／Rendering／LOD／fallback／Save／Replay／Target Qualificationの新version closureを必須にする |
| Product Future ↔ Target roles | `closed-in-target-design`／materialization absent | Product Planがsingle Targetまたはclient／authority／operations bundle、各Ownerがrole別Qualificationを所有 |
| AI-native Product identity ↔ Operation／Project／Gameplay／Editor／Evidence／Compatibility | `closed-in-target-design`／materialization absent | Productはclaimとminimum surface、Executable ContractsはOperation、Project Stateはtransaction、Gameplayはstructured／C++選択、Editorはinteraction、AI Securityはauthority、VerificationはEvidence、Compatibilityはinitial V1／公開後evolutionを一意に所有 |

## 8. Architecture closure register

| ID | 論点 | 状態 | Owner／解決条件 |
|---|---|---|---|
| `ARCH-C01` | Product Runtime Owner一覧からRuntime Package／Persistenceが欠落 | `corrected-in-review` | Product Plan §9.2へ既存Owner linkを追加 |
| `ARCH-C02` | 汎用Runtime Asset request／residency authority | `closed-in-target-design` | initial V1の専用[Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md)へ一意化。Definition／Port、exact consumer ref、Qualificationは未materialize |
| `ARCH-C03` | Architecture Inventory／Explain Projection | `unmaterialized` | Architecture Governance。Schema、Generator、immutable Artifact、bounded query、negative fixtureを同じInventory hashへ閉じる |
| `ARCH-C04` | AI Task Capsule／read／explain／propose Operation／理解Eval | `closed-in-target-design`／materialization absent | Product Planのclaim-derived Required Operation Universe、AI SecurityのTask Capsule、各Owner Projection、Verificationの理解Eval／fresh Receiptをexact intersectionへ束縛する |
| `ARCH-C05` | Runtime ECS initial V1 Owner uniquenessとECS closure | `closed-in-target-design`／materialization absent | Runtime ECSの一意Owner、type／Ref inventory、layout、query、structural transaction、Component evolution、E6／E7 Binding、static Manifest／runtime Time Ref、snapshot／live Entity Ref境界を定義。Schema／runtime／Fixture／Receiptは未materialize |
| `ARCH-C06` | Preview／Shipping GameHost executable-role mapping | `corrected-in-review` | Windows Ownerで`mirakan_game_host.exe`と`mirakan_game.exe`を同じrole／別compositionとして明示 |
| `ARCH-C07` | Target別C++ Build optimization closure | `closed-in-target-design`／materialization・measurement absent | Toolchain `TargetConfigurationBuildPolicyV1`が全Target×Configurationのexact flag／LTO／symbol／hardening／ISA／PGO不採用、Performanceがsize／startup／load／frame／memory／build metricとthresholdを所有。Configuration名だけから最適化を推測しない |
| `ARCH-C08` | CPU ISA dispatch／topology／affinity／QoS | `partially-closed` | ToolchainでTarget ISA baseline／Windows optional dispatchはclosed。topology／affinity／QoSと全Target runtime selection／QualificationはMath＋Performance＋Platformで未materialize |
| `ARCH-C09` | Reference HardwareとRuntime budget | `provisional` | Performance＋Platform。Hardware Profile、Benchmark executable、absolute threshold、Measurement Receiptを固定 |
| `ARCH-C10` | 必須C++ dependency、minimum OS、CI／device pool | `partially-closed` | Release-critical first-party componentとWindows minimum OSはtarget design選定済み。Repository lock／Schema／Fixture／Receipt、CI runner／GPU・macOS host／mobile device poolのowner・capacityは未materializeまたは未選定で、候補名だけではGateを開かない |
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
| `ARCH-C27` | Dedicated Target、Transport Connection、gameplay session／authority／replicationの複合ownership | `closed-in-target-design` in Future scope／implementation absent | Runtime Package、[Network Transport／Connection](../09-networking/network-transport-connection.md)、[Multiplayer Authority／Replication](../09-networking/multiplayer-authority-replication.md)へ三分し、Gateway-owned Provider generation／peer exchange／pair identityをTransportへ閉じ、Online Servicesを別Future Decisionに維持 |
| `ARCH-C28` | same-target規則でlarge-session clientとMMO全Targetが到達不能 | `closed-in-target-design` | Product Planの`FutureTargetClosureRegistryV1`で25 `single_target`／6 `target_role_bundle`、active／common／profile固有Future binding、role mapping、claimごとのrequired non-empty role、bundle claim releaseをexact化 |
| `ARCH-C29` | Product Definition／Release・Completion required universeがDecision作成者入力になり得る | `closed-in-target-design` | Product Planのreceipt-free `ActiveProductDefinitionV1`とNamed AlgorithmによるRelease／Completion Requirement Projectionへ一意化 |
| `ARCH-C30` | Release authorization、Platform upload、actual publication、support開始、Completionのauthority混同 | `closed-in-target-design` | [Product Release Decision](../00-product/product-release-decision.md)と[Product Publication／Completion](../00-product/product-publication-completion.md)へ分離し、signed Decision→Platform Receipt→Publication→Completionの一方向DAGを固定 |
| `ARCH-C31` | Engine source／Host Editor・Tool・Installer／Target／SDK／Template／Sampleのrelease identityとNOTICE／Security coverage不足 | `closed-in-target-design` | Product Lifecycleのreceipt-free Release Content Manifest、Host Distribution、Target Packageと全source／container set equalityをLifecycle／Security acceptanceへ要求 |
| `ARCH-C32` | Save Catalog Schemaとload resolutionがUI Ownerへ混在 | `closed-in-target-design` | PersistenceがCatalog／Slot／Bundle・content package membership／active-root precondition／Load Request、UIがSettingsとのatomic co-publicationだけを所有 |
| `ARCH-C33` | 公式current AGPとMiraikanai selected pinの混同 | `closed-in-target-design`／qualification absent | Toolchainは2026-07-30公式API referenceのCurrent Release exact 9.3.1とselected coordinate `com.android.tools.build:gradle:9.3.1`を一致させ、9.3 family Compatibility、Preview 9.4.0-alpha07、Miraikanai-selected NDK r29を分離する。resolved artifact hash／provenanceと同一baseline Qualificationを別途要求する |
| `ARCH-C34` | document-only Operation候補とcurrent materialized／active／operational状態の混同 | `corrected-in-review`／materialization absent | [Executable Contracts](../02-foundation/executable-contracts.md) §8.2をcurrent状態の唯一Ownerとし、四集合をexact `[]`へ固定。Planning Appendix、AI Security、Core、Domainはtarget candidateまたは将来activation後の条件だけを記述する |
| `ARCH-C35` | Product必須Operation familyの省略・縮退 | `closed-in-target-design`／materialization absent | Product PlanのRequired Product Operation Universeへinstall／update／repair／uninstall、bootstrap／open、human author／preview／validate／commit、build／test／cook／package／launch、diagnostics／support、Pack acquire／install／apply／update／remove、AI read／explain／propose／validate／approve／commit、security、publication／withdrawalをnon-collapsed familyとして閉じる |
| `ARCH-C36` | PackがProduct distribution subject／Manifest／publication controlから脱落 | `closed-in-target-design`／materialization absent | Product Lifecycleへ`pack_distribution` branch、Pack requirement→Contract→Subject coverage、Manifest／Engine Release `pack_contract_refs[]`、全control universe、publication route applicability、Lifecycle acceptanceのexact set equalityを追加する |
| `ARCH-C37` | Completion Evidenceを別requirement pairのTarget／locale／dimension／distribution scopeから借用可能 | `closed-in-target-design`／materialization absent | Product Planの各required completion bindingへpair固有scopeを保存し、Publication／Completionで各pairのEvidence unionをそのexact scopeとset equalityにする。global unionは補助投影に限定する |
| `ARCH-C38` | PublicationのSubject×Artifactごとのchannel／Target／locale applicabilityが未閉鎖 | `closed-in-target-design`／materialization absent | Lifecycleが各`{distribution_subject_ref, artifact_ref}`のrequired／forbidden routeを全route universeの完全分割として所有し、Publication Projectionはrequired relationだけをexact flattenする |
| `ARCH-C39` | Runtime ECSの文書全体type／Ref canonicalization、structural boundary／permission branch不足 | `closed-in-target-design`／materialization absent | Runtime ECS §8へ全top-level type／Ref inventory、persistent hash domain、process-local persistence禁止、external Owner解決、`RuntimeStructuralCommitBoundaryV1`、kind別required／empty Field matrixとnegative Fixtureを追加する |
| `ARCH-C40` | Windows CFGがlinker flagだけでcompiler instrumentationを閉じていない | `closed-in-target-design`／qualification absent | ToolchainとWindowsで全first-party object-producing Shipping compileへ`/guard:cf`、linkへ`/GUARD:CF /DYNAMICBASE`を要求し、最終binaryのGuard／CF Instrumented／FID tableをinspection Evidenceにする |
| `ARCH-C41` | Owner supplementのcomplete `status=active` candidateがcurrent Operation membershipを肯定 | `corrected-in-review`／materialization absent | Procedural World supplementをtarget-complete candidateへ限定し、current四集合の唯一OwnerをExecutable Contracts §8.2、全集合exact `[]`へ維持する。candidate／fixture／Manifestからcurrent membershipを導出しない |
| `ARCH-C42` | 34 required Operation familyのActivationは閉じるが全journey acceptanceが未閉包 | `closed-in-target-design`／materialization absent | Product PlanがClaim Scope×family×Operation×surface×Host／Target×scenario×branch×Evidence classのrequired tupleとforbidden surfaceを完全分割し、Lifecycleが全tupleとtyped Journey Evidenceをexact set equalityにする。external IDE／AI-MCP、expected rejection、failure recoveryを別surface／familyで代用しない |
| `ARCH-C43` | Pack lifecycle aggregateがaction別before／after lineageとReceipt identityを失う | `closed-in-target-design`／materialization absent | Pack Contractがacquire／install／apply／update／removeごとのtyped signed Receipt binding、Project revision、Pack Registry、installed closure、dependency closure、source／destination Contract、Compatibility Change、last-validを保持し、continuous E2Eとindependent fixtureを分離する |
| `ARCH-C44` | Product Authority Headがbusiness scope単位のcanonical stream identityを持たずdual genesis可能 | `closed-in-target-design`／materialization absent | Release Decisionが`{Authority Service,State Owner,State Type,Authority Scope}`のstream key、hash-derived State／Head Stable ID、stream単位CASを所有し、Release／Publication／Completionが各typed scopeを投影する。dual genesis、wrong stream／scope、Service-global collapseを拒否する |
| `ARCH-C45` | Completion rejected branchをcurrent completionへ昇格可能 | `closed-in-target-design`／materialization absent | Publication／Completionがapproved／rejected branch matrix、gapとreason kindの関係、approved-only `establish_current`を所有し、Product Plan completion gateもapproved Subject、empty gap／reason、current scoped Headを要求する |
| `ARCH-C46` | claim-derived 0／1／2 Reference DimensionとLifecycle carrierの両dimension固定下限が矛盾 | `closed-in-target-design`／materialization absent | Product LifecycleがReference Package／Sample／Coverage／Qualification bindingの各subsetをRelease Requirement Projectionのexact Target×dimension×requirement集合へ閉じ、`none`、2D-only、3D-only、各Targetの2D＋3Dを一意に表現する。workflow Sample、wrong Target／dimension／Candidate／Projectによるcardinality充足を拒否する |
| `ARCH-C47` | Reference SampleとTarget Packageを別Project snapshot／source closureからstitch可能 | `closed-in-target-design`／materialization absent | Project Stateがcanonical Source Entry集合、完成Entry Ref MCD bytesのunsigned lexicographic comparator、domain-separated Closure hash、exact canonical transport artifactを持つ解決可能な`ProjectSourceClosureV1`と、count／全可変長Field／endianness／end-of-inputを固定したtransport wire grammarを所有する。Product LifecycleとCore Package ReceiptはSample／全Target Package／Acceptance binding／Receiptのsnapshot四Field、Closure ref／content hash、transport artifact、Project revisionをbyte equalityにし、同revision別closure／snapshot、missing／extra Source entry、alternate comparator／encoding／repackまたはref併記だけの充足を拒否する |
| `ARCH-C48` | Manifest／SDK／Documentationへ分散するSampleのrelease universeが未定義 | `closed-in-target-design`／materialization absent | Product Lifecycleの`ReleaseSampleSetV1` Named Algorithmが三carrierのexact Refをcanonical unionし、Sample Distribution Subject、Reference Coverage、Reference Qualification bindingのdistinct Sample projectionをexact集合へ閉じる。direct Manifestを空にしたnested Reference除外、orphan／hidden Sample、同名別identity collapseを拒否する |
| `ARCH-C49` | Target Packageが外側Runtime Entry launch rootでなく内側World Package単数だけを束縛 | `closed-in-target-design`／materialization absent | Runtime Package OwnerがPackage artifactの全memberとlaunchable outer entryをread-backするdetached typed Manifest、およびManifestのexact一行を選ぶreceipt-free `RuntimeEntryLaunchSelectionV1`を所有する。Core Package／install／launch ReceiptとProduct Lifecycleは同じManifest ref／hash、Selection、artifact、Source provenance、Candidate、Target、Contract Set、outer entry集合をset equalityにし、worldのouter→inner edge、UI／headlessのnull、bare Manifest／descriptor／entry-set hashや別entry launchのstitchを拒否する |
| `ARCH-C50` | Release集合へ属するWorkflow SampleをTarget×scenario単位の実行Evidenceなしで受理可能 | `closed-in-target-design`／materialization absent | Product Lifecycleが各Sampleのtyped Target applicability／scenario／expected branch／Operation family／exact Operation Ref／execution branch、`WorkflowSampleQualificationRequirementSetV1`およびoperation-specific evidence unionを所有し、全Workflow Sampleのnested Acceptance bindingをexact set equalityにする。Package non-successは完成成果物を持たず、Installはpackage-level、LaunchはSelection単位で証明し、direct Manifest-only Sample、missing／extra Target・scenario・Operation、carrierだけの充足、Reference Receipt流用またはOwner Receipt意味の拡張を拒否する |
| `ARCH-C51` | Launch ReceiptがManifest内の実選択outer Runtime Entryを証明せず、descriptor共有entryをstitch可能 | `closed-in-target-design`／materialization absent | Runtime Package Ownerのreceipt-free Launch SelectionがManifest ref、outer entry ref、entry artifact ref、descriptor artifact refをexactly one rowへ閉じる。CoreのLaunch request／Task／Authorization／ReceiptとProduct LifecycleのTarget／Reference／Workflow bindingは同Selection ref／hashをbyte equalityにし、E1実行でE2 claim、descriptor SHAだけ、別Manifest、outer／inner Ref代用を拒否する |
| `ARCH-C52` | WorkflowのPackage／Install／Launch共通evidence rowがOwner Receiptのoutput presence／granularityと矛盾 | `closed-in-target-design`／materialization absent | Product Lifecycleのclosed unionがauthoring／build／test、Package success、Package non-success、Install、Launchを分離する。Package failureはsuccess output groupを省略しattempt request／Diagnostic／before-after stateを、Installはpackage-level subjectを、Launchはentry別Selection／Owner Receiptを保持し、stale完成Package貼付やQualification wrapperによるOwner意味拡張を拒否する |
| `ARCH-C53` | Reference Sampleのdistinct membershipだけで一Sampleのsupported Target／requirement欠落を隠せ、Coverage上限も不足 | `closed-in-target-design`／materialization absent | Product Lifecycleの`ReferenceSampleCoverageSetV1(M)`と`ReferenceSampleQualificationRequirementSetV1(M)`が全Reference Sample×supported Target×scenarioのfull identityをCoverage／Acceptanceへset equalityにする。Coverage上限を`576 × 64 = 36864`、flat requirement上限を`576 × 16384 = 9437184`から導出し、別Sample代用、Target欠落、duplicate展開、4,096件打切りを拒否する |
| `ARCH-C54` | canonical Project Source transportのcount／可変長Field encodingが未固定で複数byte列を適合扱い可能 | `closed-in-target-design`／materialization absent | Project Stateが35-byte marker、`uint32_be` count、Ref／path／roleの`uint32_be` length、`uint64_be` Source length、exact bytes、checked bounds、trailing-byte禁止、完成Entry Ref MCD bytesのunsigned lexicographic反復順をfield-by-fieldで固定する。varint／ASCII／u64 count、little-endian、delimiter、別comparator／順序／metadata混入を拒否し、required future wire-vector caseを列挙する |
| `ARCH-C55` | 非Owner proposalがReview Appendixの規範依存targetとなりauthority解釈が分岐 | `corrected-in-review` | Navigation Design Alignment ReviewからProduct Execution Registry Proposalを規範依存から関連文書へ移し、全規範依存targetをOwnerへ限定する |
| `ARCH-C56` | Source Closure／wire反復のcanonical Entry comparatorが未定義 | `closed-in-target-design`／materialization absent | Project Stateが`project_source_entry_ref_mcd_bytes_lexicographic_v1`を唯一のcomparatorとし、unsigned byte比較、prefix、duplicate、禁止sort keyをClosure hashとwireへ共用する |
| `ARCH-C57` | Capability×Target直積がActivation carrierを超過 | `closed-in-target-design`／materialization absent | Product PlanのProjection Capacity Validity Algorithmがchecked直積4,096件以下をClaim Scope validityへ要求し、truncate／aggregateを拒否する |
| `ARCH-C58` | Requirement×Evidence class pairがRelease／Completion carrierを超過 | `closed-in-target-design`／materialization absent | Product Requirement typed inputのprepublication／Completion classを分離し、distinct pairを各4,096／8,192件以下へchecked制約する |
| `ARCH-C59` | publication distribution subject source集合がcarrierを超過 | `closed-in-target-design`／materialization absent | typed Requirement inputの各selector rowを256件以下、distribution scope／artifact role／Host・runtime Target・scope-independent／locale branchへ完全展開したdistinct subjectを65,535件以下へchecked制約しRequirement identityのcollapseを拒否する |
| `ARCH-C60` | Required JourneyがRequirement identityを失いAcceptanceがsemantic groupを縮約 | `closed-in-target-design`／materialization absent | Product PlanとLifecycleのJourney／Parity full identityへRequirement Refとsemantic groupを保持し、両Fieldを含むexact set equalityへ統一する |
| `ARCH-C61` | Required Journey tupleが65,535件carrierを超過 | `closed-in-target-design`／materialization absent | Projection Capacity Validity Algorithmがrequired／forbidden full tupleを各65,535件以下へchecked制約する |
| `ARCH-C62` | required／bundled Pack unionが1,024件coverageを超過 | `closed-in-target-design`／materialization absent | Active Product Definition validityがcanonical Pack requirement unionを1,024件以下へ制約する |
| `ARCH-C63` | Template×supported Target coverageのexact universeと容量が不足 | `closed-in-target-design`／materialization absent | Product Lifecycleの`TemplateTargetCoverageSetV1(M)`が全Template×supported Targetを最大16,384件へ投影しCoverageとset equalityにする |
| `ARCH-C64` | Bootstrap Receipt 64件では65 Templateをcover不能 | `closed-in-target-design`／materialization absent | `TemplateBootstrapRequirementSetV1(M)`が各Templateと全requested Target setを一rowへ閉じ、Acceptance carrierを最大Template数256件へ整合する |
| `ARCH-C65` | SDK Manifestの同Target複数Artifactを64件Coverageが表現不能 | `closed-in-target-design`／materialization absent | SDK Target entryをTarget projection uniqueかつ最大64件にし、Manifest／Coverageのfull `{Target,Subject,Artifact}`をset equalityにする |
| `ARCH-C66` | Documentation Qualificationのrequired universe／full identityがない | `closed-in-target-design`／materialization absent | detached Documentation Requirement Setとaggregate ReceiptがEntry、Link、Snippet×Target、Tutorial×Target／branchを各exact set equalityにする |
| `ARCH-C67` | 単一Distribution Subjectの再帰Artifact集合が65,535件を超過 | `closed-in-target-design`／materialization absent | Distribution Projection Capacity Validity Algorithmが各Subjectの全再帰distinct Artifactを65,535件以下へchecked制約し、Documentation artifact省略を拒否する |
| `ARCH-C68` | 全Subject×Artifact applicability rowが65,535件を超過 | `closed-in-target-design`／materialization absent | 同Algorithmがproduct-wide `{Subject,Artifact,artifact role,execution scope,locale scope}` full binding総数を65,535件以下へ制約する |
| `ARCH-C69` | 一Artifact bindingのrequired／forbidden route完全分割がcarrierを超過 | `closed-in-target-design`／materialization absent | full binding×Product publication subject完全partitionを8,388,608件以下、required flattenを65,535件以下、forbidden flattenを8,388,608件以下へchecked制約する |
| `ARCH-C70` | Publication flatten後のchannel keyが65,535件を超過 | `closed-in-target-design`／materialization absent | LifecycleがSubject／Artifact identity付きrequired route総数を65,535件以下へ制約し、Publication Ownerも入力validityを再検証する |
| `ARCH-C71` | Workflow coverage key導出とnull semanticsが未定義 | `closed-in-target-design`／materialization absent | typed Requirement workflow applicabilityからHost／Target independentとexact expansionを分離する`WorkflowCoverageKeySetV1(M)`を定義し、最大49,152 keyと各keyのRequirement／Subject exact unionを保持する |
| `ARCH-C72` | Security required Subject closed unionがDistribution Subject／artifactを表現不能 | `closed-in-target-design`／materialization absent | Product Securityへexact Distribution Subject branchと親Subject付きdistributed Artifact branchを追加し、required projectionを262,144件以下へ閉じつつRegistryを524,288件へ分離してrequired外subjectの容量も保持する |
| `ARCH-C73` | Privacy Acceptanceの自己申告flow集合にreceipt-free canonical rootがなくflow単位Evidence identityも失う | `closed-in-target-design`／materialization absent | Active Product DefinitionとHost／runtime Target／locale／flow集合がset equalityなcandidate-bound `ProductDataFlowInventoryV1`を新設し、flow別execution scopeと法的keyのdetached Required Scope Projection、nested typed resultへ閉じてcross-flow／cross-scope stitchを拒否する |
| `ARCH-C74` | Product release Project test required universeとResult full coverageがない | `closed-in-target-design`／materialization absent | Developer TestingのCatalog／nested Requirement Set／Selection／Attempt／Run ResultとLifecycleの全Template＋Release Sample外側Set／nested Result bindingがProject×Suite×Case×Parameter×Targetをset equalityにする |
| `ARCH-C75` | Product Requirementから各Projectionを生成するclosed typed入力がない | `closed-in-target-design`／materialization absent | Product Planの`ProductRequirementProjectionInputV1`がcategory、Evidence class、Target／locale／dimension、route、journey、surface、Completion scope、workflowのclosed applicabilityを一意所有し、全Product projectorが同Refだけを消費する |
| `ARCH-C76` | Workflow Sampleがexecution branchだけを持ち同branchの別Operation Receiptをstitch可能 | `closed-in-target-design`／materialization absent | Sample requirement、Reference／Workflow Requirement Set、Acceptance bindingへOperation familyとexact Operation Refを追加し、branch-family matrix、Required Operation Universe membership、Owner ReceiptのOperationをbyte equalityにする |
| `ARCH-C77` | Editor／planning catalogのLaunch journeyがtyped Manifest／Launch Selectionを表示せずInstall Receiptだけで成功表示可能 | `corrected-in-review`／materialization absent | Editor WorkspaceとOperation Planning CatalogがPackage Manifest、outer Runtime Entry、Launch Selection Ref／完成record SHA、固有request／Task／Receiptを表示・比較し、descriptor hash／別entry／Install Receiptだけの成功を拒否する |
| `ARCH-C78` | Product rootがHostとlocale universeを持たずruntime Targetへ混同可能 | `closed-in-target-design`／materialization absent | Active Product Definition、Claim Minimum／Scope、Release Projection、Manifest、Engine Release、Lifecycle Acceptance、Security baselineをBuild／Editor Host、runtime Target、localeのdisjoint集合へ分け、Toolchain tagged profile projectionとset equalityにする。third-party minimumはbuild Host、Editor Host、`en-US`、`ja-JP`を欠かせない |
| `ARCH-C79` | pre-publication Requirement acceptanceがpair固有Host／Target／locale／dimension scopeを失う | `closed-in-target-design`／materialization absent | Release ProjectionとRelease Decision supplemental Evidenceが同じRequirement×Evidence class pairのclosed scope branchを保持し、pair内canonical unionとexact equalityを行う。not-applicable、independent、exact set、別pair unionを相互代用しない |
| `ARCH-C80` | Publication routeのrequired／forbiddenがLifecycle自己申告でartifact role／execution scopeを持たない | `closed-in-target-design`／materialization absent | Product typed inputが中立distribution scope／artifact role／execution／locale selectorを所有し、Lifecycleの各Subject artifact full bindingへ決定論的にjoinしてrequired／forbiddenを完全分割する。Publication keyはHost／runtime Target／scope-independentとlocale-independentを保持する |
| `ARCH-C81` | Required Operation Universeのfamily／OperationがJourney 0件でもset equalityを通過可能 | `closed-in-target-design`／materialization absent | Required Journeyのdistinct family／Operation projectionをUniverseとset equalityにし、各pairへ少なくとも一つのsuccess rowとfresh Journey Evidenceを要求する |
| `ARCH-C82` | forbidden surface tupleがsemantic equivalence groupを失い別groupと衝突判定できない | `closed-in-target-design`／materialization absent | forbidden row、容量計算、Lifecycle intersectionへsemantic groupを保持し、同Requirement内でも別groupを相互代用しない |
| `ARCH-C83` | Privacyがruntime Target軸だけでHost／scope-independent flowと法的applicability universeを閉じない | `closed-in-target-design`／materialization absent | Flow Definitionがexecution／locale／route applicabilityを持ち、receipt-free Required Scope Projectionがflow×execution×route×locale×jurisdiction×region×processor keyを生成する。Acceptanceは各keyのPlatform／Consent／retention／Legal結果をpassed／approvedまたはEvidence付きnot-applicableでexact equalityにする |
| `ARCH-C84` | Developer TestingのRequest／Selection／Attempt／Result Refとretry aggregateが未閉包 | `closed-in-target-design`／materialization absent | 全local Ref／domain separator、ID／versionを定義し、Catalog full case universeをSelection dispositionへ完全分割、Attemptを連続ordinal／retry lineageへ閉じ、Run terminal outcomeとflakyを決定論的集約する。Release required caseはselectedかつ初回passだけを受理する |
| `ARCH-C85` | installed Product全体のinstall／update／repair／uninstall transition closureがない | `closed-in-target-design`／materialization absent | receipt-free Installed State、Installation Channel、detached Transition Requirement Setとnested Resultを新設し、Host／channel／layout／Subject／artifact／owned・retained data、before／after／failure／cancel／recoveryを閉じる。Update branchはexact ProductUpdatePlanを必須にしTarget Package uninstallと分離する |
| `ARCH-C86` | SDK／Support qualificationとfirst-party licenseがflat自己選択Receipt pool | `closed-in-target-design`／materialization absent | SDKをHost×Target×action／Snippet、Supportをrelease×window×channel×locale×Target scope×auth×attachment×scenario×branchのdetached Requirement Setとnested resultへ置換する。licenseはexact Grant、legal Evidence、全required SubjectのNotice Presentation／controlから導出し独立flat poolを除去する |
| `ARCH-C87` | Required Journeyがlocale／2D・3D dimension identityを失いEvidenceをcross-scope再利用可能 | `closed-in-target-design`／materialization absent | Requirement typed journey applicability、Required／forbidden Journey tuple、Parity Receipt、Release Decision、Pack Action bindingへlocaleとReference dimensionのscalar closed scopeを保持し、specific／independent／not-applicableを分離してfull key set equalityとcapacityへ含める |
| `ARCH-C88` | localized Tutorial Entryと実行scenarioが結合せずgeneric成功を複数Entryへ流用可能 | `closed-in-target-design`／materialization absent | Bundle binding、detached Documentation Requirement、Receiptへexact Entry Ref／content hash、Entry locale、source／rendered artifact、scenario、Target、branchを保持し、全Tutorial Entry↔bindingの双方向closureとfull identity set equalityを要求する |
| `ARCH-C89` | Support Requirement SetがWindowのsupported runtime Target集合を投影しない | `closed-in-target-design`／materialization absent | Support Window scenarioへall／exact／independent Target applicability、bindingへscalar Target scopeを追加し、Window／Manifest／Engine Release Target set equality、Target込みchecked expansion、Receipt read-backを要求する |
| `ARCH-C90` | Product-wide Target／Locale／Scenario／Evidence／Qualification／Operation Receipt Refにcomplete backing recordとclosed resolverがない | `closed-in-target-design`／materialization absent | Product PlanがTarget／Locale Profile recordとroot Registry、AI VerificationがScenario／Evidence Class／generic Evidence／Qualification Receipt spine、Executable Contractsがgeneric Operation Receipt／Type Registryを一意所有する。全Refはkind／class／purpose／owner-specific backing record／subject／signed recordを保持し、consumer-local narrow tupleを削除する |
| `ARCH-C91` | claim-facing RequirementがEvidence class義務三集合emptyでもProjectionから自己省略可能 | `closed-in-target-design`／materialization absent | typed Inputごとにpre-publication／Completion／Journey Evidence class unionをnon-emptyにし、publication／Completion obligationはCompletion classも必須にする。Release DecisionがRequirement単位で再検証し、global class unionまたは別Requirementで補完しない |
| `ARCH-C92` | required publication routeのmissing条件が否定反転し完全routeを誤拒否し得る | `corrected-in-review`／materialization absent | Product Lifecycleの条件を「全rowで一度もrequiredへ投影されない場合」に修正し、required／forbidden完全分割、required flatten set equality、missing／extra rejectionと同じ論理へ統一する |
| `ARCH-C93` | Claim MinimumのRequirement categoryとtyped Input categoryに一意joinがなくcategory override可能 | `closed-in-target-design`／materialization absent | Product PlanのRequirement→typed Inputを唯一のcategory total mappingにし、全Minimum／Projectionが同Ref・categoryをbyte equalityにする。third-party minimumのdistinct category projectionはclosed enum全体とexact set equalityにする |
| `ARCH-C94` | Ref解決後のScenario applicability／Evidence Class subject contract／scope／branch invariantをset比較前に消費しない | `closed-in-target-design`／materialization absent | AI VerificationがVerification Scope Vector、subject contract集合、observed branch、Type Registry constraintとSemantic Admissibility Predicateを所有する。全Requirement Set／Acceptance／Decisionがowner-specific recordをread-backし、invalid tupleの両辺複写を拒否する |
| `ARCH-C95` | generic Operation Receiptがsubject contract集合とrequestをRef identityへ保持せずhash-only誤用を十分に拒否できない | `closed-in-target-design`／materialization absent | Operation Receipt／Refへsubject contract集合とrequest hashを含め、owner-specific completed subjectからfull contextを再計算する。Pack bindingも同tupleをread-backし、generic wrapperだけからDomain successを推測しない |
| `ARCH-C96` | Product Plan／Privacyが参照するMCD `kind=data_flow`がExecutable Contractsのclosed kind集合に存在しない | `closed-in-target-design`／materialization absent | Executable Contractsへ共通`data_flow` Envelope kindを追加し、Privacyがcomplete payloadだけを所有する。全Data Flow参照を同Contract setの`McdContractRefV1(kind=data_flow)`へ統一し、Privacy-local narrow Refを除去する |
| `ARCH-C97` | Completion／pre-publication pair数は上限内でもscalar Evidence cover、さらにCompletionではDistribution Subject coverを含む総数が65,535件carrierを超過可能 | `closed-in-target-design`／materialization absent | Product Planは四scalar軸のsub-boundを`h+t+l+d-3`とし、Product Publication／Completionはexact Subject join後の最終上限を`h+t+l+d+max(1,s)-4`として全pairのchecked sumを65,535件以下へ制約する。pre-publicationは四軸上限を維持し、scope／Subject aggregate、別pair collapseまたはCartesian productへ変更しない |
| `ARCH-C98` | third-party product release Minimumの必須workflowが自由語彙で、ライフサイクル機能の欠落を集合比較できない | `closed-in-target-design`／materialization absent | Product Planが`ProductLifecycleWorkflowKindV1`をexact 12 kindで所有し、Minimumのdistinct workflow projectionを全12 kind集合とset equalityにする |
| `ARCH-C99` | required Evidence Class／Record Type pairは個別配列上限内でもcanonical unionが4,096件carrierを超過可能 | `closed-in-target-design`／materialization absent | AI Verificationが全allowed pairのrequired canonical unionをchecked構築し、distinct union件数を4,096以下に制約する。分割、重複collapseまたは別Requirementへの移送を代用にしない |
| `ARCH-C100` | Operation必須quorum ruleとEngine R5 baselineのrule／witness集合を固定carrierへ収容できない | `closed-in-target-design`／materialization absent | AI Security Approvalの`applied_quorum_rules`を`[1..11]`へ拡張し、canonical rule unionを11件以下、`minimum_distinct_subjects`のchecked sumを`approvals[1..16]`へ収まる16以下にする |
| `ARCH-C101` | Product publication selectorのplatform値がHost／runtime Target execution Profileと結合せずcross-platform memberを選択可能 | `closed-in-target-design`／materialization absent | publication selectorの`platform_kind`をexact Host／Target Profileの`product_platform_kind`とbyte equalityにし、`all` selectorも同Platform memberだけの完全partitionにする |
| `ARCH-C102` | Lifecycle transitionがchannel Host scope外Hostを処理でき、required Host coverageとUpdate Plan対象がずれる | `closed-in-target-design`／materialization absent | transition algorithmを`host_profile_scope_bindings`へexact解決するHostだけに閉じ、required Host coverage、Update Plan intersection、runtime-target-only除外を同じ集合で検証する |
| `ARCH-C103` | Windows signing subjectがTarget Packageだけに閉じ、Host Distribution publicationを署名できない | `closed-in-target-design`／materialization absent | Windows signingを`host_distribution | target_package`のclosed tagged unionにし、各branchのexact subject tuple、artifact、publication scopeをbyte equalityで検証する |
| `ARCH-C104` | withdrawn completionに成功Publication channel key全件のwithdrawal結果がなく、部分撤回をcompleteと判定可能 | `closed-in-target-design`／materialization absent | Completionへbounded `withdrawal_results`を追加し、withdrawn branchでは成功Publication channel key集合とのset equalityとEvidence read-backを必須にする |
| `ARCH-C105` | Multiplayerのsession-scoped record／Refが同IDの新lifecycleへ旧Participant／Object／Baseline等を再束縛可能 | `closed-in-target-design`／materialization absent | Participant、Transport Binding、Network Object、Authority Lease、Baseline／Ack、Recipient Projection、Network Time、Handoffへ`session_epoch`を追加し、Network Object chain、Projection内全object Ref、Envelope→Projectionを含む全ref／state／deduplicationを`{session_id, session_epoch}`へbyte equalityにする |
| `ARCH-C106` | Camera-local Render View identityとframe slotが混同され、Render Graph／LOD／virtual geometry間でstable identityを共有できない | `closed-in-target-design`／materialization absent | Render Graphがcanonical `RenderViewV1／RefV1`、`ViewFamilyV1／RefV1`、Requirementを所有し、StableId／generationと`frame_view_slot`を分離して全consumerを同Refへ統一する |
| `ARCH-C107` | Advanced Light Transportの入力Refにbacking producer recordがなく、Project／World／Target／Quality／generation joinを評価不能 | `closed-in-target-design`／materialization absent | Lighting、Materials、Environment、World、Terrain／Foliage、Render Graphが各versioned summary／snapshot recordとexact Refを所有し、ALTが全join keyをchannel評価前にbyte equalityで検証する |
| `ARCH-C108` | LOD Previewが未定義`DiagnosticRefV1`を保持し、canonical Diagnostic Registry recordへ解決できない | `closed-in-target-design`／materialization absent | `blocking_diagnostic_refs`をexact四Fieldの`DiagnosticCodeRefV1`へ統一し、旧tokenをArchitecture全文から除去する |
| `ARCH-C109` | `ProjectChangeSetV1.request_id`をEditor consumerが要求する一方、Project State EnvelopeにFieldが存在しない | `closed-in-target-design`／materialization absent | Project Stateへ必須`request_id: UUIDv7`を追加し、Editor由来では`command_request_id`とbyte equality、ChangeSet identity／idempotencyとは別のattempt correlationにする |
| `ARCH-C110` | Public API subject上限内でもcallable parameter／return binding総数が`CppValueTransferPolicyV1.bindings[1..65536]`を超過可能 | `closed-in-target-design`／materialization absent | exact Contract Set／Target／Toolchainごとに`sum(p(f)+r(f)) <= 65536`をCatalog publication前にchecked検証し、binding集合とのset equality、sharding／subset合成禁止を要求する |
| `ARCH-C111` | Developer Testing resultが未定義`DiagnosticRefV1`を使い、assertion failureをDiagnostic 0件で表現可能 | `closed-in-target-design`／materialization absent | Attempt／Run Resultを`DiagnosticCodeRefV1`へ統一し、`attempt_status=assertion_failed`では1件以上と四Field Registry解決を必須にする |
| `ARCH-C112` | Query dispatch Planは各65,536 work unit以下でも、一Simulation Advanceの複数Plan aggregateがcapacity carrierを超過可能 | `closed-in-target-design`／materialization absent | `max_query_work_units_per_advance`を`[1,65536]`へ閉じ、全Planの`sum(|work_units|)`をchecked preflightしてcallback開始前に拒否する |
| `ARCH-C113` | SectionごとのEntity上限内でもWorld全体のpersistent identity数がSave `entity_records[0..1048576]`を超過可能 | `closed-in-target-design`／materialization absent | Runtime World capacityへ`max_persistent_entities <= 1048576`を追加し、Package load集合、ECS commit後集合、Save projection集合を同上限へ接続してSave recordとのset equalityとsharding禁止を要求する |
| `ARCH-C114` | `RenderSnapshot.view_families[1..64]`に対して`light_snapshot`が単数で、二つ以上のFamilyを各Family固有のLight／ALT入力へ閉じられない | `closed-in-target-design`／materialization absent | `light_snapshots[1..64]`をFamily Ref集合とexact set equalityにし、Family Ref順、duplicate／missing／extra拒否、Light 0件Familyのempty Snapshot、ALTへのexact一件選択を要求する |
| `ARCH-C115` | Product Planのcompact 2D command RPG First PlayableをExecution ProposalがShooter Fixtureで代用していた | `corrected-in-review`／execution projection absent | ProposalはShooterのlegacy Phase／GateをRPG、C1またはProduct-facing Phase exitへ投影しない。RPG Fixture／WP／Capability／Gate／Target bindingは本作業で発明せず、将来の承認済みActive Definition変更まで未materializeとして保持する |
| `ARCH-C116` | persisted `RuntimeComponentAccessManifestV1`がadvance固有`RuntimeTimeRefV1`を保持するがDefinition／runtime artifact境界が不明 | `closed-in-target-design`／materialization absent | Runtime ECSのManifest／structural permissionはstatic `TickPhaseId`、lease／token／deltaはruntime `RuntimeTimeRefV1`へ分離し、advance identityをcontent-addressed Definitionへ保存しない |
| `ARCH-C117` | exact publication一致のEntity Refをcross-advance carrierが保持すると次advanceでstaleになり得る | `closed-in-target-design`／materialization absent | Runtime ECSが`RuntimeEntitySnapshotRefV1`と`RuntimeEntityLiveRefV1`を分離。liveはdelivery時にWorld／Entity generationとaliveを検査し、durable identityはPersistenceへ委譲 |
| `ARCH-C118` | Network pair identityがprovider generationの定義、生成authority、peer exchangeへ閉じていない | `closed-in-target-design` in Future scope／implementation absent | Network TransportがGateway-owned `TransportProviderGenerationV1／RefV1`、nonce／generation lifecycle、Attempt／Offer exchange、Pairの二generation set equality、旧generation拒否を定義 |
| `ARCH-C119` | Product state／WP approvalのindependenceが単独の複数role運用を制約する | `closed-in-target-design`／operational enforcement absent | Product PlanはProject authoringとEngine production／release self-certificationを分離。Product operational Decisionの独立性を維持し、同一humanの複数Role／Key、AI、publisherを別主体に数えない |
| `ARCH-C120` | Morphのimport、animation track、runtime evaluation、fallbackのOwner間contractが部分的 | `closed-in-target-design`／Future unmaterialized | Morphをinitial V1／C1／C2から除外し、Source検出をfail closedにする。Future採択はAsset／Animation／Rendering／LOD／Persistence／Target Qualificationのend-to-end新versionを要求 |
| `ARCH-C121` | KTX／FLAC source archiveのcomponent licenseと採用build／distribution scopeのclosureが未materialize | `closed-in-target-design`／materialization absent | Toolchainのclassification、required file-level closure、fail-closed gateは確定。exact archive、BOM／license、compile／link／distribution manifest、除外または承認、SBOM／NOTICEが実在するまで採用・Qualificationを主張しない |
| `ARCH-C122` | MCP公式currentとMiraikanai baselineのlifecycle／version negotiation判断 | `closed-in-target-design`／materialization absent | supported-version setをexact singleton `[2026-07-28]`にし、per-request version／HTTP header／`server/discover`を同Revisionへ閉じる。`2025-11-25` initialize、alias、fallback、dual lifecycleをinitial V1へ持たない |
| `ARCH-C123` | Android orientation／resizabilityの公式条件とgame category例外のApplication package投影 | `closed-in-target-design`／materialization absent | Mobile Commonはadaptive-only、Androidは`appCategory=game`、resizable true、orientation／aspect lockなし。game例外へ依存せずphone／tablet／foldable／resizeを同packageでQualificationする |
| `ARCH-C124` | AI-native C++ Product identity、汎用Engine minimum surface、外部Engine非模倣、initial V1 clean-breakが複数Ownerへ暗黙分散 | `closed-in-target-design`／materialization absent | [AI-native C++ Product Identity Decision](../decisions/2026-08-03-ai-native-cpp-product-identity.md)で採用理由を記録し、Product Plan、Gameplay Programming Model、Compatibility／Evolutionへ正本責務を分離。Operation／Schema／Fixture／Receipt／Engine実装は生成せず、`ARCH-C115`～`ARCH-C123`の独立closureまたは残るmaterializationを代用しない |
| `ARCH-C125` | Windows GameInput／VCLibs／WARPを同じapp-local配布前提で扱い、公式再配布条件とProduction routeが不整合 | `closed-in-target-design`／distribution materialization absent | GameInputをnormal-install prerequisite、MSIX `/MD`をVCLibs framework dependencyへ分離し、Store／sideload／bootstrapperごとのclean-machine read-backを必須化。WARP packageはDevelopment／internal conformance限定としShipping／third-party distributionから除外する |
| `ARCH-C126` | AI task context bindingがundefined projection enum／自由なGame Understanding候補へ依存 | `corrected-in-review`／Game Understanding excluded／materialization absent | AI Securityが`AiTaskContextCapsuleV1`とMCD Type ref集合によるProjection Bindingを一意所有し、Project／source revision、Field mask、completeness、freshnessを閉じる。Game Understanding候補はOwner／Ref／bound／hash closure採択までbinding不能とする |
| `ARCH-C127` | appendicesが共通signed record envelopeを複写し、署名／hash／Trust／revocation正本が二重化またはbacking record不在 | `closed-in-target-design`／materialization absent | AI Verificationが`MirakanSignedRecordV1／RefV1`、AI SecurityがTrust Subject／Role／Key／Revocation Snapshotを一意所有し、appendixはdomain payload候補だけを説明する。署名record、Trust Registry、revocation、receiptは未materialize |
| `ARCH-C128` | `ProjectRevisionRefV1`、changed-path、Source Promotion Subject／Receiptのbacking recordがなくNative／Shader consumerが解決不能 | `closed-in-target-design`／materialization absent | Project StateがProject／revision三Field identity、changed-path exact ref、Native／Shader tagged Promotion Subject、Code Owner／Evidence closure、signed Promotion Receiptを所有し、PromotionとProject Commitを分離する |
| `ARCH-C129` | Rendererのfar plane、World packet、Target Capability、exposure family、motion producerが未定義または重複 | `closed-in-target-design`／materialization absent | Render View／LODへfarをbyte-exact投影し、undefined `WorldRenderPacket`を除去、Executable ContractsへTarget Capability Snapshot共通Envelope、View Familyへexposure history key、Render Graphへmotion／exposure／history execution carrierとproducer／reset規則を配置する |
| `ARCH-C130` | First-party例外無効方針とthrowing constructor変換Diagnosticが矛盾 | `closed-in-target-design`／materialization absent | `MirakanMakePersistent<T>`を`noexcept` construction／destructionだけへ限定し、constructor exception branch／Diagnosticを除去。fallible post-construction initializationは非公開ownerのexplicit `Result`とrollbackへ分離する |
| `ARCH-C131` | Pack source／transport／index／signature／license／permission refにbacking typeがない | `closed-in-target-design`／materialization absent | Pack ContractがPack固有Origin、Transport Policy、Index Snapshot、Signing Identity、Signature／License／Permission Reviewとexact Refを所有し、integrity、trust、license、permission、qualificationの相互推論を禁止する |
| `ARCH-C132` | Asset `user_provided` originがundefined Project source record／ingest receipt／Source boundaryを参照 | `closed-in-target-design`／materialization absent | exact `ProjectSourceEntryRefV1`、Asset-owned `AssetSourceIngestReceiptV1／RefV1`、Broker read-backの`project_source_boundary_snapshot` Artifactへ統一し、path／bytes／media type／Safety Evidence／decisionをAsset descriptorへbyte equalityで束縛する |
| `ARCH-C133` | Privacy Data FlowのSecurity Control refとConsent subject／Decision chainがuntyped | `closed-in-target-design`／materialization absent | `SecurityControlRefV1`へ統一し、opaque `ProductConsentSubjectRefV1`、versioned Decision／Ref、expected-previous CAS chain、withdraw／regrant規則をPrivacy Ownerへ配置する |
| `ARCH-C134` | MobileのTarget ref正本重複、root分類にAndroid／Apple directory／backup／file protection写像がなく、Android Backとconfirmed Settingsがkeyboard／controllerへ偏る | `closed-in-target-design`／materialization absent | Product Planの`TargetProfileRefV1`を再利用してMobile固有Configuration／Distribution／Project Specを別型化。Androidはfiles／no-backup／cache、AppleはApplication Support／Cachesとbackup／Data Protectionを公式区分へ写像し、System navigationとAccessibility-safe Settings commandを閉じる |
| `ARCH-C135` | glTF parser／MikkTSpace generatorを要求する一方、Production dependencyが未選定・未lock | `closed-in-target-design`／dependency materialization absent | [glTF Import Dependency Baseline](../decisions/2026-08-03-gltf-import-dependency-baseline.md)でcgltf single parser、原典MikkTSpace、Khronos Validatorの三役をexact commitへ選定。Toolchainがversion／license／lock Gate、Asset LifecycleがWorker／IR／validation、Materialsがtangent意味を所有し、artifact／hash／SBOM／Adapter／Fixture／Receipt／Qualificationまで全glTF importをfail closedにする |
| `ARCH-C136` | Compatibilityが参照する`ArchitectureApprovalRefV1`、Architecture Change、Reviewer identityにbacking recordがない | `closed-in-target-design`／materialization absent | Architecture GovernanceがDocument Version、ChangeSet、Decision Ref、Reviewer Identity Ref、Approval／Refを所有し、Architecture approval、ADR state、Compatibility approval、implementation、Qualificationを分離する |
| `ARCH-C137` | AIがGame intentを収集してもBrief／Spec／Question／Assumption／Decision／Requirement traceabilityとstaging readinessへ閉じる一意Ownerがない | `closed-in-target-design`／materialization absent | [Game Production Loop](../03-authoring/game-production-loop.md)がGame production subject、Intent session／draft、Brief／Spec／Decision Ledger、Understanding Closureを一意所有し、会話logやAI自己申告をProject authorityまたは理解完了にしない |
| `ARCH-C138` | technical test、human Playtest Observation、Experience Evaluation、Iteration Decision、Human Gameplay Approvalが相互代替可能 | `closed-in-target-design`／materialization absent | Game Production LoopがObservation→Evaluation→Decisionを同一Candidate／Project lineageで分離し、Developer TestingとHuman Approvalを別Ownerへ保持する。新Iterationは新Authorizationを要求し、Closure発行をCommit／Releaseへ昇格しない |
| `ARCH-C139` | AI game generationがGame composition、external content生成、Project Source生成を一つの曖昧なclaimとして扱える | `closed-in-target-design`／claim Evidence absent | [Product Plan](../00-product/product-plan.md)が`AiGameGenerationLaneV1`三laneとnon-substituting claim scopeを所有する。initial First Playableはexact `{ai_composed_game}`だけを要求し、Asset／Source生成supportを主張しない |
| `ARCH-C140` | Product C1がcompact RPG、Shooter、3D、manual-only、AI-onlyのどれで成立するか一意でない | `closed-in-target-design`／execution projection・materialization absent | Product Planのreceipt-free `FirstPlayableDefinitionV1`がcompact 2D command RPG、en-US／ja-JP、keyboard／controller、manual／AI continuity、clean install／offline、Game Production Closureをexact化する。Product Lifecycleが同じCandidate／Project revisionのEvidenceを集約し、Shooter／3D／別Genreを拒否する |
| `ARCH-C141` | Evidence freshnessがassumption shadow shape、catalog参照、Verification stateの間で重複・参照切れ | `corrected-in-review`／Evidence materialization absent | [AI Verification／Provenance](../01-governance/ai-verification-provenance.md)が`EvidenceEffectiveStateV1 = fresh | expired | revoked | invalid`とpriorityを一意所有し、AI Security／assumption／catalog consumerはexact Refと同stateへ統一する |
| `ARCH-C142` | Gameplay System dependency、実装選択、authoritative State ownerをRuntime Package／Scheduling／Fixtureが名前や登録順から再構成可能 | `closed-in-target-design`／runtime materialization absent | [Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)が`GameSystemDependencyGraphV1`、`SystemImplementationSetV1`、`GameStateOwnerProjectionV1`を一意所有し、Runtime Package／Schedulingはexact三Refを同じProject revision／Contract set／Targetへ束縛する |
| `ARCH-C143` | Engine Coreが何を満たせば実行可能かがProduct First Playableや全Engine完成と混同される | `closed-in-target-design`／Qualification absent | [Core Architecture §13](../02-foundation/core-architecture.md#13-minimum-executable-core-target-closure)が30 exact member roleと九Qualification scenarioを所有する。worldless headless bootをRuntime Package／Schedulingへ束縛し、Rendering、Genre、First Playable、Releaseを代用しない |
| `ARCH-C144` | stale節参照、Index件数、Closure Review末尾番号、planning family count、外部tool current表現がreference integrityを破る | `corrected-in-review`／generated Inventory・validator absent | Architecture Indexを64 Ownerへ更新し、Memory→Toolchain §2.6、Performance→Executable Contracts §8.2、Review末尾§12–§13へ修正、planning ledgerを実行行数どおり27 family／207 unique候補へ統一した。CMake 4.4.1をselected pinとして維持し、2026-07-31公開の4.4.2を「自動current pin」にしない |

`ARCH-C03`はArchitecture Inventory／Explain Projectionというsubjectのtarget semantics欠落ではなくmaterialization未実施を表す。ただしArchitecture全体で唯一のmaterialization blockerではない。`ARCH-C04`のOperation／Capsule／Eval、`ARCH-C05`のECS Schema／Binding／Fixture、`ARCH-C10`のCI／device pool、`ARCH-C12`、`ARCH-C115`のexecution projection、`ARCH-C116`～`ARCH-C123`に残るSchema／runtime／dependency／Conformance／device Evidence materialization、`ARCH-C124`～`ARCH-C134`／`ARCH-C136`の未materialized contract／Evidence／distribution、`ARCH-C135`の選定済みだが未materializeのProduction Asset dependency closure、`ARCH-C137`～`ARCH-C143`のGame production／claim／First Playable／freshness／Gameplay graph／Minimum Coreに残るSchema／Operation／Project／Fixture／Receipt／runtime／Qualification、`ARCH-C144`のgenerated Inventory／validator、ならびにLifecycle／Security Ownerが明記する未materialized Artifactを独立に保持し、一件の状態を他件の代表にしない。

## 9. 推奨するArchitecture判断

次は実装順ではなく、未解決Authorityを閉じる際の判断条件である。

### 9.1 Runtime Asset initial V1 authority

`ARCH-C02`の比較では専用Owner案を選び、[Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md)へ次をinitial V1の一つの正本として配置した。

- request identity、Target、Catalog／Package generation、priority、deadline、cancel、idempotency。
- dependency closure、range I/O、decode／transcode、CPU／GPU upload、staging。
- activation boundary、generation、lease、resident／streamed、partial mip／LOD。
- eviction、fallback、retire、device loss、memory pressure、last-valid維持。
- queue／worker／memory／hitch charge、metric、Diagnostic、Qualification。
- Texture、Mesh、Audio、Font、World Section等のDomain固有意味を共通Managerへ吸収しないPort境界。

initial V1 Owner選択後も、Scheduling diagramの`Asset Runtime` label、Asset Lifecycleの委譲文、Runtime Packageのloader、Runtime Asset Owner文書からManager、Service、API、Schema、CapabilityまたはOperationのmaterializationを推測しない。利用可能表示にはDefinition Closure、exact consumer binding、Target Qualificationを要する。

### 9.2 Build optimization authority

`ARCH-C07`のtarget design closureはToolchainがexact build inputs、PlatformがABI／Package／device constraints、Performanceがmeasurement／promotion predicateを所有する三者分離を維持する。flag表だけ、benchmarkだけ、Package成功だけを最適化済みEvidenceにしない。initial V1はPGO不採用であり、将来採用はcontent-addressed training／profile／freshnessを持つ別Decisionを必要とする。

### 9.3 AI-readable Architecture

`ARCH-C03`と`ARCH-C04`のtarget semanticsは閉じたがmaterializationはないため、AI readinessは`conceptually-readable`と`operationally-readable`を別表示にする。後者は少なくともInventory、Explain Projection、Owner fragment、Task Capsule、read／explain Operation、理解Eval、fresh Receiptが同じArchitecture revision／Inventory hash／Contract Setへ閉じた場合だけ成立する。

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

`ARCH-C27`はDedicated game Runtime TargetをRuntime Package、endpoint／handshake／semantic delivery／Connection epoch／Gateway-owned Provider generation／peer exchange／pair identity／whole-envelope duplicate／Packet Plan reassembly identityを[Network Transport／Connection](../09-networking/network-transport-connection.md)、gameplay session／role／authority／Network Object／replication／prediction／rollback／session lifecycleとauthority epochに閉じたCommand deduplicationを[Multiplayer Authority／Replication](../09-networking/multiplayer-authority-replication.md)へ分離し、Future target semanticsのownership境界を`closed-in-target-design`とした。Provider、Schema、socket、runtime、Fixture、Receipt、Target QualificationおよびActivationは引き続きabsentである。

Transport `connected`をsession `joined | synchronized | authoritative`へ読み替えず、Dedicated TargetだけでMultiplayer claimを解放しない。Account／entitlement、Lobby／Matchmaking、Relay allocation、Hosting fleet、cloud persistence／economy／moderationは別将来Decisionであり、opaque binding以外を二Ownerへ吸収しない。

### 9.15 Future Target-role closure

`ARCH-C28`は単一Target prerequisiteとcross-target Product compositionを`single_target | target_role_bundle`へ分けて`closed-in-target-design`とした。AAA等25行はsame exact Target、persistence、small co-op、rollback、large-session、MMO、managed external Hostの六行はexact profile、role Target集合、全role pair relation、DAG末端まで再帰展開するactive／common Future／profile固有additional Future前提、bundle前提role mapping、claimごとのrequired non-empty roleを一つのpromotion／claim release単位にする。

small co-op／rollbackのheadless Targetはdedicated profileだけが選べ、Dedicated Target Futureをprofile固有前提にする。large-sessionのclient Targetとauthority Target、MMOのdesktop clientとdistributed authority／operationsをflat intersectionへ潰さない。一roleのReceipt、Target kindだけの一致、別profile間の混成、implicit cross product、optional artifact roleが空のmanaged Source buildからProduct claimを解放しない。Registry、Profile Artifact、Promotion Manifest、Activation、Receiptは未materializeである。

### 9.16 Product release authorityとactual publication

`ARCH-C29`～`ARCH-C31`は[Product Release／Publication Authority Ownership](../decisions/2026-07-30-product-release-publication-authority.md)の四段分離を採用した。Product Planはreceipt-free Definitionとrequired universe、Lifecycle／Securityはsame-release Manifest／Acceptance、Product Release Decisionはqualified authorityの署名済みpre-publication authorization、各Platformはsigning／submission／public read-back Receipt、Product Publication／Completionはrequired channel set equality、effective publication time、support開始、withdrawal／supersession、authoritative Completionを所有する。

Release Decision前のrequired Evidenceへpublic read-backを混ぜず、Release Requirement Projection内の独立したrequired publication channel universeとして固定する。Platformはunsigned Subject、approval label、issue stateまたはbare hashを受理せず、Publicationはupload／Store approvalだけを`published`へ昇格しない。全型、Authority、Key、Receipt、Publication storeは未materializeである。

### 9.17 AI-native C++ Product identityとindependent design

`ARCH-C124`はprovider-neutralなtyped contract core案を採用した。Product PlanがAI-native claimと第三者向けminimum surface、Executable Contractsがcanonical Operation、Project Stateがtransaction、AI Securityがauthorization／approval、AI VerificationがEvidence、Gameplay Programming Modelがstructured data／bounded C++の選択、Editorがmanual／AI interaction、Compatibilityがinitial V1 direct definitionと公開後evolutionを所有する。

Unity、Unreal Engine、Godotの公式資料は製品surfaceとfailure modeのgap discoveryに限定し、型、API、Scene model、Plugin形式、UI layout、名称、workflow、defaultまたはcreative expressionをMiraikanaiへ移植しない。initial V1に過去draft／外部Engine互換alias、dual reader、Project importerまたはmigration layerを作らず、初回公開後は実consumer Inventoryなしにclean breakを適用しない。これはMiraikanaiのproject-decisionであり、外部組織の公式推奨ではない。

### 9.18 PLAN-AUDIT-20260803の設計判断closure

`ARCH-C115`～`ARCH-C123`は、一つの包括Decisionへ吸収せず、subjectごとのOwnerとDecisionへ分離して判断した。`ARCH-C115`はShooter EvidenceによるRPG代用だけを修正し、RPG execution projectionは未作成のまま残す。`ARCH-C116`／`ARCH-C117`は[Runtime ECS Static Definition／Entity Reference Boundary](../decisions/2026-08-03-runtime-ecs-static-and-entity-reference-boundary.md)、`ARCH-C120`は[Initial Morph Capability Boundary](../decisions/2026-08-03-initial-morph-capability-boundary.md)、`ARCH-C122`は[MCP Current Protocol Baseline](../decisions/2026-08-03-mcp-current-protocol-baseline.md)、`ARCH-C123`は[Android Adaptive Game Window Baseline](../decisions/2026-08-03-android-adaptive-game-window-baseline.md)へ採用理由を記録し、current contractは各Ownerだけに置く。

`ARCH-C118`のProvider generationはFuture [Network Transport／Connection](../09-networking/network-transport-connection.md)、`ARCH-C119`の独立承認境界は[Product Plan](../00-product/product-plan.md)、`ARCH-C121`のKTX／FLAC license／build／distribution evidence境界は[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)が所有する。これらはtarget designまたはclassificationを閉じたものであり、Schema、Registry、Fixture、Receipt、dependency archive、Build input、runtime、device／Conformance Evidence、QualificationまたはActivationの実在を意味しない。

### 9.19 glTF Import dependency baseline

`ARCH-C135`は[glTF Import Dependency Baseline](../decisions/2026-08-03-gltf-import-dependency-baseline.md)により、Production Asset parserをcgltf single path、tangent generatorを未変更の原典MikkTSpace、仕様適合性oracleをDevelopment／Qualification専用Khronos Validatorへ分けた。Toolchainがexact version／commit／license／lock Gate、Asset Lifecycleがbounded Source／Worker／typed IR／validation／Receipt、Materialsがprovided／generated tangent、normal texture UV、handednessの意味を一意に所有する。

parserが理解するfeatureをEngine対応へ自動昇格せず、外部Scene object、URI resolver、C++型、他EngineのImporter workflowをMiraikanaiの正本にしない。single parserを別Backendへfallbackせず、MikkTSpace per-corner出力を既存indexへ平均しない。これはArchitecture上の選定だけを閉じる。dependency archive／hash、license bundle／NOTICES、Toolchain lock、SBOM、Adapter、Schema、Fixture、Receipt、Build、Khronos conformance、cross-host determinism、QualificationおよびActivationは未materializeであり、全glTF importは引き続きfail closedである。

## 10. 本Reviewに伴う文書整合

| 文書 | 変更内容 |
|---|---|
| Architecture Index | Owner文書を63件へ更新し、Product Release Decision／Publication・Completion、Privacy、Developer Testing、Advanced Light Transport／Terrain・Foliage／Network Transport／Multiplayer AuthorityとOwnership Decisionへのnavigationを追加 |
| Product Lifecycle | Project bootstrap、Template／Sample／Documentation、surface parity、update／repair／support／NOTICE、E2E acceptanceの専用Ownerを追加 |
| Product Security | threat ownership、baseline、vulnerability case、security update／disclosure／notification／incidentの専用Ownerを追加 |
| Product Release Decision／Publication・Completion | deterministic requirement projection、signed authority、Platform publication集約、support開始、Completionの専用Ownerを追加 |
| Persistence／UI | Save Catalog／Slot membership／Load preconditionをPersistenceへ一意化し、UIをSettingsとのatomic co-publicationへ限定 |
| Product Plan | Product completionへLifecycle／Security acceptanceを追加し、各専用Ownerとの境界を明記 |
| Product Plan／Naming | Creative expressionとextensionを別axis化し、Level Workspaceを非authorityのEditor presentationへ固定 |
| Project State | `AuthoringSelectionContextV1`のProject lineage、target解決、revision／hash／invalidation Ownerを明記 |
| World | Level／Region／Portal表示語の正規解決、World authoring Projection、Scene instance／nested composition／typed override／explicit rebase、cross-owner atomicityを明記 |
| Editor Workspace／Panel／Design System | Level Stable ref／membership／`SetLevel*`を除去し、World Compositionとpack-owned bindingへ分離 |
| Executable Contracts planning catalog | Level WorkspaceがOperation familyでないこと、複数Owner Operationの部分fallback禁止を明記 |
| Scenario／Stage、LOD、Materials、World fixture | `Level Source`をcurrent Core identityとして扱う残存記述をlegacy migrationまたはWorld／Scene／Topology表現へ修正 |
| Runtime Asset Lifecycle | 汎用request、dependency、generation、residency、lease、eviction、recoveryのinitial V1 Ownerを追加 |
| Asset Lifecycle／Runtime Package／Scheduling | Source／Cook、Runtime Entry staging、phase／publicationとRuntime Asset authorityの境界を更新 |
| AI Security／UI／Debug | Consent Record、purpose binding、提示、Support Bundle消費境界を統合 |
| AI Verification／Core Architecture | Test集約／retry／quarantine／waiverとcross-target Build／Release Evidence closureを追加 |
| Architecture Governance／Naming | 直交状態軸、subject-qualified状態語彙、Package／Pack／Bundleのqualified名称を追加 |
| Gameplay Programming Model | Perception→Decision→Action、interrupt、failure、Save／Replay、causal debug接続を追加 |
| Gameplay Feature Packs／RPG Genre Pack／Product Plan | Reusable RPG Feature、Genre composition、通常Reference Game、Product acceptanceを四層化 |
| Runtime ECS | initial V1の直接Owner、Gameplay authoringとの境界、Definition Closure条件を追加 |
| Render Graph／World／Animation／Runtime Asset | Sprite／Tile／frame／residency／packet／stable sort／batchの2D authority chainを追加 |
| Native Game Module | restart-based Windows PreviewをHot Reloadのcleanな意味置換として明記し、in-process unload／replacement／patch禁止を維持 |
| Windows／Mobile Common | legacy `build/` rootを除去し、Application lifecycleとpresentation surface availabilityを別stateへ修正 |
| Toolchain／Dependencies | Target別C++ Build optimization closureをKnown unresolved registerへ追加し、Owner本文のexact Target×Configuration policyでtarget designを閉鎖 |
| Toolchain／Asset Lifecycle／Materials／Decision Log | cgltf／原典MikkTSpace／Khronos Validatorをsingle dependency closureへ選定し、version所有、Worker／typed IR、tangent意味、materialization Gateを分離。選定だけからImport利用可能性を推論しない |
| Advanced Light Transport／Render Graph／Lighting／Materials／Post／Environment | Source意味、channel別Technique／fallback、generic effect、physical executionのauthorityを分離 |
| Terrain／Foliage／World／LOD／Runtime Asset | TerrainとFoliageのSource／Artifact branchを独立させ、partition／selection／residencyを既存Ownerへ維持 |
| Runtime Package／Network Transport／Multiplayer | Dedicated Target、Connection／delivery、gameplay session／authority／replicationを三分し、Online Servicesを非正本化 |
| Product Plan／Execution Registry proposal／Shooter | Future 31行、Reflection claim、Target-role bundle、active／common／profile固有Future prerequisite、claim required role、promotion／claim release closureを追加 |
| Product Plan／Release Decision／Product Lifecycle | Requirement単位Evidence non-empty、category total mapping、publication route missing条件、semantic admissibility read-backを追加 |
| AI Verification／Executable Contracts／Privacy／Pack | Verification scope／subject contract／branch、Operation Receipt subject contract／request、MCD `data_flow` Envelope／Refを一意化し、hash-only／invalid tuple／local narrow Refを拒否 |
| Product Plan／Gameplay Programming Model／Compatibility／Decision Log | AI-native claim、汎用Engine minimum surface、manual continuity、structured data／bounded C++、外部Engine非模倣、initial V1 direct definitionと公開後consumer保護を一意なOwnerへ接続 |
| Runtime ECS／Native Game Module／Scheduling／Decision Log | static phase identityとsnapshot-bound／cross-advance Entity Refを分離し、process-local／durable identity境界を接続 |
| Asset Lifecycle／Animation／LOD／Virtual Geometry／Decision Log | Morphをinitial V1／C1／C2からfail-closedで除外し、Future end-to-end Capabilityへ分離 |
| Toolchain／Executable Contracts／AI Security／MCP Supplement／Decision Log | MCP current protocol singletonとlegacy lifecycle非採用を接続し、Server／Schema／Conformanceの未materialize境界を維持 |
| Mobile Common／Android／Decision Log | adaptive-only Mobile policyをAndroid manifest／window lifecycle／device Qualification境界へ接続 |
| Product Plan／Execution Registry Proposal／Network Transport／Toolchain | RPG Evidence代用、approval independence、Future provider generation、dependency evidence closureをsubject別Ownerへ接続 |

## 11. 非目標

- 本ReviewからC++ source、CMake target、Schema、Generator、Operation、Fixture、Receiptを生成しない。
- Runtime Asset Manager、Consent Registry、RPG Pack Registry、QA aggregator、Build Optimization ProfileまたはAI Toolを仮実装しない。
- Product Lifecycle／Privacy／Security／Release Decision／Publication・CompletionのOwner追加からSchema、Operation、Template、Sample、Case Store、Authority、Key、Fixture、Receipt、Publication storeまたは運用体制を生成済みとみなさない。
- Product Phase、Work Package、実装順、担当、工数、日程を追加しない。
- review文書を`normative`、Capabilityを`active | qualified | production`へ昇格しない。
- 外部EngineのAPI、object model、chunk size、Asset identity、path、default、tool permissionをMiraikanaiの正本へ移植しない。
- Plugin ecosystem、汎用Script VM／JIT、Multiplayer、large-world機能をrequired-featuresの名称coverageだけでMVPへ追加しない。

## 12. Missing-contract and platform-distribution follow-up

| Field | Value |
|---|---|
| Date | 2026-08-03 |
| Scope | Windows配布、AI task context、署名Envelope、Project revision／promotion、renderer temporal input、例外禁止、Pack trust、Asset ingest、Privacy consent、Mobile system navigation／storage、glTF／tangent dependency、Architecture approvalについて、実装または実装計画を作らずOwner境界を閉じる |
| Canonical tracking | `ARCH-C125`～`ARCH-C136`の12件。10件を`closed-in-target-design`、`ARCH-C126`を`corrected-in-review`（provisional Game Understanding projectionをcurrent capsuleから除外）、`ARCH-C135`を`open-blocker`／`not_adopted`として保持 |
| External fact route | Microsoft Learn／Microsoft NuGet package license、Android Developers、Apple Developerの一次資料を外部事実に限定して確認した。採用条件、fail-closed policy、clean-break境界はMiraikanaiのproject-decisionである |
| Canonical Owners | Toolchain／Dependencies、Windows、Executable Contracts、AI Security／Approval、AI Verification／Provenance、Project State、Architecture Governance、Render Graph、Memory／Pointers、Pack Contract、Asset Lifecycle、Product Privacy／Data Governance、Mobile Common、Android、Apple、Materials |
| Clean-break result | absentな過去Schema、Registry、Fixture、Receipt、外部Engine互換alias、dual reader、migration layerを初期V1へ導入していない |
| Open blocker retained | glTF parserおよびMikkTSpace相当tangent生成器はProduction dependency未採用。依存ChangeSetとライセンス／provenance／build／distribution evidenceが承認されるまでglTF importおよびtangent-required primitiveをfail-closedで拒否する |
| Materialization boundary | 記載した型、Ref、policy、Receipt shapeはいずれもtarget semanticsであり、Schema file、Registry、Fixture、Receipt、C++、Build、CI、Qualification、Activation、release artifactは`absent` |
| Completeness boundary | 原監査の「168 issues／66 unresolved」に安定したissue ID対照表がないため、全件との一対一照合は立証しない。本Reviewの正規追跡集合についてのみ閉鎖状態を主張する |
| Exact terminal response／marker | `missing_contracts_corrected_dependency_adoption_pending`／`[[MIRAIKANAI-PLAN-AUDIT-20260803-MISSING-CONTRACT-CLOSURE-RECORDED]]` |
| Retention disposition | repository外の原監査Artifactは削除せず、Owner／consumer closureと未採用dependencyの決定が完了するまでtransient扱いを維持する |

## 13. ARCH-C135 glTF dependency decision follow-up

| Field | Value |
|---|---|
| Date | 2026-08-03 |
| Supersedes current-state interpretation | §12の`ARCH-C135 open-blocker／not_adopted`は同follow-up時点の記録として保持し、current Architecture状態は本節と§8のClosure Registerで上書きする |
| Decision | [glTF Import Dependency Baseline](../decisions/2026-08-03-gltf-import-dependency-baseline.md)でProduction Asset parser=`cgltf` single path、tangent generator=原典MikkTSpace、spec oracle=Development／Qualification専用Khronos Validatorをexact commitへ選定 |
| Owner closure | [Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md#gltf-tangent-dependency-state)がversion／commit／license／materialization Gate、[Asset Lifecycle](../03-authoring/asset-lifecycle.md#gltf-import-adapter-boundary)がBroker／Worker／typed IR／validation／Receipt、[Materials](../06-rendering/materials.md#41-canonical-pbrとgltf-mapping)がtangent／normal textureの意味を所有 |
| Canonical tracking | `ARCH-C125`～`ARCH-C136`の12件は11件`closed-in-target-design`、`ARCH-C126`は`corrected-in-review`。Architecture選定に関する`open-blocker`は0件だが、全件のmaterialization／Evidence blockerがなくなったことを意味しない |
| External fact boundary | Khronos glTF 2.0.1のtangent規則と公式Validator、各upstreamのsource／licenseはofficial／primary fact。cgltf選定、single-parser、Worker隔離、atomic materialization、fail-closedはMiraikanaiのproject-decisionであり、Khronos公式推奨とは表現しない |
| Clean-break／independent-design result | dual parser、legacy alias、fallback importer、外部Scene object公開、Runtime Source import、他Engine API／Asset workflowの模倣をinitial V1へ導入しない |
| Materialization boundary | source archive／file hash、license bundle／NOTICES、Toolchain lock、SBOM、Adapter、Schema、Registry、Fixture、Receipt、C++、Build、CI、Conformance、cross-host Qualification、Activationは`absent`。全glTF importを`dependency_materialization_absent`で拒否する |
| Non-goals retained | 実装、実装計画、C++方式、Work Package、順序、工程、工数、担当または実行可能Artifactを追加しない |
| Exact terminal response／marker | `gltf_dependency_target_selected_materialization_pending`／`[[MIRAIKANAI-PLAN-AUDIT-20260803-ARCH-C135-DECISION-RECORDED]]` |
| Retention disposition | repository外の原監査Artifactは削除せず、accepted findingのOwner／consumer closure、materialization evidenceとterminal audit完了までtransient扱いを維持する |
