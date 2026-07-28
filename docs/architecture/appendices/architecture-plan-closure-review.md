# Miraikanai Engine Architecture Plan Closure Review

- 文書ID: mirakan.appendix.architecture-plan-closure-review
- 文書種別: proposal appendix
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 親Owner: [Architecture Governance](../01-governance/architecture-governance.md)
- 正本範囲: Architecture計画全体の監査結論、current／target区分、AI可読性、Creative expression境界、Scene／Level authoring意味、Runtime coverage、Editor／Game分離、Target別Build mapping、cross-owner整合性、未解決Closureの追跡
- 非正本範囲: Subsystem semantics、Schema、API、Backend、固定Budget、Product Phase／Work Package、実装Task、実装順序、担当、工数、日程、Capability Activation、承認結果
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Product Plan](../00-product/product-plan.md)
- 関連文書: [Runtime ECS Design Closure Review](runtime-ecs-design-closure-review.md)、[AI-readable Asset／Memory／Async Loading Alignment](../decisions/2026-07-28-ai-asset-memory-async-alignment.md)、[Core Architecture](../02-foundation/core-architecture.md)、[Naming／Project Layout](../02-foundation/naming-project-layout.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Project State](../03-authoring/project-state.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)、[Editor Workspace／UX](../03-authoring/editor-workspace-ux.md)、[Editor UI Framework](../03-authoring/editor-ui-framework.md)、[Native Game Module](../03-authoring/native-game-module.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[Physics](../05-simulation/physics.md)、[World](../06-rendering/world.md)、[Render Graph](../06-rendering/render-graph.md)、[Windows](../07-platform/windows.md)、[Mobile Common](../07-platform/mobile-common.md)、[Android](../07-platform/android.md)、[Apple](../07-platform/apple.md)、[Scenario／Stage](../08-packs/scenario-stage.md)
- 根拠区分: project-review／official-spec comparison
- 外部根拠確認日: 2026-07-28

> 本書は実装計画ではない。表の順序、Closure ID、推奨判断は実装順、Product Phase、Work Package、担当、工数または日程を意味しない。本書は既存Ownerの正本を置き換えず、明白な文書不整合の修正と、未決定事項を未決定のまま一意に追跡する。

## 1. 監査結論

Miraikanai EngineのArchitecture計画は、Owner分離、current／target区分、typed reference、fail-closed、Evidence、Target別Qualificationという設計原則では強い。一方、RepositoryのOwner文書は`review`、実装状態は`absent`であり、Markdown上の型、Registry、Operation、固定値をcurrent実装または利用可能なCapabilityとして扱えない。

| 観点 | target design | current状態 | 結論 |
|---|---|---|---|
| AIによる概念理解 | strong | Markdown reviewのみ | Owner、identity、revision、authority、安全境界は説明可能 |
| AIによる機械解決 | strong contract intent | incomplete | Inventory、Explain Projection、Capsule、Schema、query Toolが未materialize |
| Creative expression | broad inside public Capability | design only | 2D／3D／nonspatial／procedural、Genre非依存Gameplay、Project C++／Shaderを許す一方、任意plugin／private API／JITは意図的に除外 |
| Scene／Level authoring | owner boundary corrected | Projection／Operation absent | `Level Workspace`をEditor presentationへ限定し、World／Scene／Space、Project selection、pack-owned Stageへ正規状態を分離 |
| Runtime描画／物理／Memory | detailed target | implementation absent | 意味、lifetime、backend境界、Qualificationは計画済み |
| Runtime Asset | end-to-end方針はaligned | Owner gap | Source／Cook／Packageは明確だが、汎用Runtime request／residency authorityが未決定 |
| Editor／Game分離 | strong | implementation absent | process、state、dependency、failure isolationが一貫している |
| Target別Build mapping | strong | lock／Receipt absent | Driver、Target、Package、Signing、device Gateは計画済み |
| C++ Build最適化 | policy-level only | unresolved | Configuration名はあるが、Target別compiler／linker最適化closureが未固定 |
| 文書間連携 | structurally sound | manual inventory | Owner参照は概ね一方向だが、生成Inventoryがない |

したがって、現計画を「Architectureの方向と安全境界が閉じつつある」と評価できるが、「AIが機械的に理解・変更できる」「Runtimeが実行できる」「Target別最適化済み」とは評価しない。設計説明、Schema materialization、Runtime実装、Qualification、Product Activationを別の状態として維持する。

## 2. 監査方法と判定語彙

監査はArchitecture Index、全Owner header、Decision、proposal appendix、規範依存、関連文書、相対link、文書ID、current／target記述を対象とした。2026-07-28の監査ではArchitecture Markdown 75件、文書ID 73件、Owner文書50件を確認し、相対Markdown link切れ0件、文書ID重複0件、50 Owner間から抽出した規範依存202 edgeの未解決0件／cycle 0件を確認した。50 Owner文書はすべて`文書状態=review`、`実装状態=absent`、`検証状態=design-reviewed`である。ただし手動IndexとMarkdown解析は、生成済み`ArchitectureInventoryV1`またはSchema validationの代用ではない。

2026-07-29に[Virtualized／Continuous Geometry](../06-rendering/virtualized-continuous-geometry.md)を51件目のOwner文書として追加した。本節の75／73／50／202という数値は2026-07-28監査の履歴Evidenceであり、追加後のcurrent Inventoryまたは再監査結果ではない。current件数は[Architecture Index](../README.md)を参照し、51 Ownerを対象にした同等の再監査Evidenceが作られるまで本節から新文書のclosureを推測しない。

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
| Runtime Package | Runtime Entry、World Root／Section、integrity、staging、dependency、capacity、atomic section publication | generic Texture／Mesh／Audio request、priority、deadline、cancel、residency、evictionのOwnerは未決定 |
| Audio | offline Cook、resident／streamed、decode worker、PCM ring、callback no-allocation／no-lock／no-I/O | backend実装、device fixture、Receipt |
| CPU execution | shared worker pool、declarative access、scalar reference、将来のSSE／AVX／NEON candidate | ISA dispatch、topology／affinity／QoSのcross-platform採用判断とTarget Qualification |
| Performance | memory、queue、worker、frame、GPU、hitch metric、Candidate比較規則 | Reference Hardware、Benchmark executable、absolute threshold、Measurement Receipt。数値は`provisional` |

Rendering、Physics、Memory、Asset Cook、Audioは「何を守るか」が記述されている。最大の横断gapは、Asset LifecycleがRuntime側authorityへ委譲する汎用Runtime Asset request／residency authorityが、一意なOwner文書へ解決しないことである。Runtime Package、Scheduling、Performanceはそれぞれpackage staging、publication boundary、capacityを所有するが、説明文から汎用Asset Manager authorityを合成しない。

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
| Governance ↔ 全Owner | target aligned／Inventory absent | GovernanceはOwner／state／dependency Inventory、各OwnerはDomain fragment |
| AI Security ↔ Project／Engine change | target aligned／Operation absent | Project ChangeSetとEngine Candidate qualificationを分離 |
| Project State ↔ Editor／World context | target aligned／Schema absent | Project Stateは`AuthoringSelectionContextV1`、Worldは`WorldAuthoringContextV1`／`SceneSliceV1`、Editorはattention／Panel binding |
| Editor ↔ World ↔ Scenario／Stage | `corrected-in-review` | Level Workspaceはpresentation、Worldはcomposition／topology、Scenario／StageはEntry／Exit／Objective／finite progression |
| Asset Lifecycle ↔ Runtime | target aligned／Owner gap | AssetはSource／Cook／Catalog、Runtime側の汎用request／residency Ownerは未決定 |
| Runtime Package ↔ Scheduling | aligned | Packageはstaging／dependency、Schedulingはcompletion acceptance／publication boundary |
| Memory ↔ Runtime Resource | aligned | Memoryはgeneration／lease／allocation、Domain Ownerはpayload意味とfallback |
| Editor ↔ GameHost | `corrected-in-review` | PreviewとShippingは同じGameHost role、別binary／dependency／Package |
| Toolchain ↔ Platform Build | mapping aligned／optimization open | Driver matrixはToolchain、Package／deviceはPlatform、performance predicateはPerformance |
| Performance ↔ Target | target aligned／provisional | 同一Target／fixture／Toolchainで比較し、fresh Receiptなしに昇格しない |
| ECS ↔ Gameplay／Package／Save／AI | open-blocker | 詳細はRuntime ECS Design Closure Reviewの`ECS-C01`～`ECS-C22` |

## 8. Architecture closure register

| ID | 論点 | 状態 | Owner／解決条件 |
|---|---|---|---|
| `ARCH-C01` | Product Runtime Owner一覧からRuntime Package／Persistenceが欠落 | `corrected-in-review` | Product Plan §9.2へ既存Owner linkを追加 |
| `ARCH-C02` | 汎用Runtime Asset request／residency authority | `open-decision` | Governance＋Product＋Asset／Runtime Owners。Owner、scope、consumer、state machine、failure、Evidenceを一つのDecisionで選ぶ |
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

## 9. 推奨するArchitecture判断

次は実装順ではなく、未解決Authorityを閉じる際の判断条件である。

### 9.1 Runtime Asset authority

`ARCH-C02`を閉じるDecisionは、専用Ownerを追加する案と既存Runtime Packageを拡張する案を比較し、少なくとも次を一つの正本へ置かなければならない。

- request identity、Target、Catalog／Package generation、priority、deadline、cancel、idempotency。
- dependency closure、range I/O、decode／transcode、CPU／GPU upload、staging。
- activation boundary、generation、lease、resident／streamed、partial mip／LOD。
- eviction、fallback、retire、device loss、memory pressure、last-valid維持。
- queue／worker／memory／hitch charge、metric、Diagnostic、Qualification。
- Texture、Mesh、Audio、Font、World Section等のDomain固有意味を共通Managerへ吸収しないPort境界。

Owner未決定の間、Scheduling diagramの`Asset Runtime` label、Asset Lifecycleの委譲文、Runtime Packageのloaderから汎用ManagerまたはAPIを推測しない。

### 9.2 Build optimization authority

`ARCH-C07`を閉じる判断はToolchainがexact build inputs、PlatformがABI／Package／device constraints、Performanceがmeasurement／promotion predicateを所有する三者分離を維持する。flag表だけ、benchmarkだけ、Package成功だけを最適化closureにしない。PGOは未採用を明記するか、content-addressed training／profile／freshnessを持つ採用Decisionを必要とする。

### 9.3 AI-readable Architecture

`ARCH-C03`と`ARCH-C04`を閉じるまで、AI readinessは`conceptually-readable`と`operationally-readable`を別表示にする。後者は少なくともInventory、Explain Projection、Owner fragment、Task Capsule、read Operation、理解Eval、fresh Receiptが同じArchitecture revision／Inventory hash／Contract Setへ閉じた場合だけ成立する。

### 9.4 Reference Hardwareと最適化

provisional Budgetと候補最適化は、同じTarget、Build、Toolchain、fixture、input trace、warm-up、sample count、aggregation、correctness oracleで比較する。別Target、sanitizer run、推定値、平均値、`latest`、製品deadlineからthresholdまたは採用結果を生成しない。

### 9.5 Scene／Level authoring boundary

Coreの正規SourceはWorld／Scene／Space／Topology relationであり、`Level Workspace`はこれらとowner-typed Gameplay文書を横断表示するEditor presentationである。`LevelSourceV1`、Level Stable ref、Level revision、Level membershipまたはLevel固有Operationを追加しない。

- World OwnerはWorld／Scene composition、Space、Anchor、Topology relation、spatial Transition intent、`WorldAuthoringContextV1`、`SceneSliceV1`を所有する。
- Project State Ownerは同じProject revisionへ閉じた`AuthoringSelectionContextV1`を所有する。
- Editor OwnerはWorkspace、attention、focus、follow／pin、Context表示を所有し、Sourceまたはauthorizationを所有しない。
- `StageDefinitionV1`等のFeature／Game OwnerはEntry／Exit、Objective、finite progressionを所有する。
- 一つのUser intentが複数Ownerを変更する場合はactiveな各Owner primitiveを一つの`ProjectChangeSetV1`へ列挙し、未Activation、staleまたはunauthorizedな一件があれば全体を部分適用しない。

この整理はOwner重複を閉じる設計修正であり、Projection Schema、Operation、Fixture、Work Packageまたは実装を作成したことを意味しない。`ARCH-C03`、`ARCH-C04`、`ARCH-C12`は引き続き未解決である。

## 10. 本Reviewに伴う文書整合

| 文書 | 変更内容 |
|---|---|
| Architecture Index | 本Reviewへのnavigationと変更入口を追加 |
| Product Plan | Runtime／Simulation Owner一覧へRuntime Package、Persistence／Saveを追加し、本Reviewを関連文書へ追加 |
| Product Plan／Naming | Creative expressionとextensionを別axis化し、Level Workspaceを非authorityのEditor presentationへ固定 |
| Project State | `AuthoringSelectionContextV1`のProject lineage、target解決、revision／hash／invalidation Ownerを明記 |
| World | Level／Region／Portal表示語の正規解決、World authoring Projection、cross-owner atomicityを明記 |
| Editor Workspace／Panel／Design System | Level Stable ref／membership／`SetLevel*`を除去し、World Compositionとpack-owned bindingへ分離 |
| Executable Contracts planning catalog | Level WorkspaceがOperation familyでないこと、複数Owner Operationの部分fallback禁止を明記 |
| Scenario／Stage、LOD、Materials、World fixture | `Level Source`をcurrent Core identityとして扱う残存記述をlegacy migrationまたはWorld／Scene／Topology表現へ修正 |
| Asset Lifecycle | `Runtime Owner`を一意Ownerとして扱わず、汎用Runtime Asset authorityがopen decisionであることを明記 |
| Runtime Package | World／Runtime Entry loader scopeと、汎用Asset request／residency非正本範囲を明確化 |
| Scheduling／Lifetime | diagramの`Asset Runtime`が未決定module labelでありOwner／target／Capabilityを生成しないことを明記 |
| AI-readable Asset／Memory／Async Decision | Runtime Asset authority未決定をCost／Owner mapへ明記 |
| Windows | Preview／Shipping executableとGameHost roleの対応を明記 |
| Toolchain／Dependencies | Target別C++ Build optimization closureをKnown unresolved registerへ追加 |

## 11. 非目標

- 本ReviewからC++ source、CMake target、Schema、Generator、Operation、Fixture、Receiptを生成しない。
- Runtime Asset Manager、Build Optimization ProfileまたはAI Toolを仮実装しない。
- Product Phase、Work Package、実装順、担当、工数、日程を追加しない。
- review文書を`normative`、Capabilityを`active | qualified | production`へ昇格しない。
- 外部EngineのAPI、object model、chunk size、Asset identity、path、default、tool permissionをMiraikanaiの正本へ移植しない。
