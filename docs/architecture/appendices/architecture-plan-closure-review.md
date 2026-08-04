# Miraikanai Engine Architecture Plan Closure Review

- 文書ID: mirakan.appendix.architecture-plan-closure-review
- 文書種別: proposal appendix
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 親Owner: [Architecture Governance](../01-governance/architecture-governance.md)
- 正本範囲: Architecture計画全体の監査結論、current／target区分、P0 Product-derived membership／16-axis closure、Product Legal／IP Governance、AI可読性、AI-native C++ Product identity／外部Engine非模倣境界、AI production Run／Workflow／Context／execution route／loop owner、versioned GUI／CLI／SDK／MCP／Agent cross-surface conformance、Game production／Playtest iteration／AI generation claim／First Playable、C2／Mobile post-C1 Product boundary、Minimum Executable Core、Creative expression境界、Product Lifecycle／Privacy／Security／Release Decision／Publication、Scene instance／override／rebase／Level authoring意味、Advanced Rendering／Multiplayer ownership、Future Target-role closure、Runtime coverage、Editor／Game分離、Target別Build／Release evidence mapping、Consent／RPG／QA／2D／Save Catalog／reference integrityのcross-owner整合性、未解決Closureの追跡
- 非正本範囲: Subsystem semantics、Schema、API、Backend、固定Budget、Product Phase／Work Package、実装Task、実装順序、担当、工数、日程、Capability Activation、承認結果
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Product Plan](../00-product/product-plan.md)
- 関連文書: [P0 Canonical Architecture／Product Legal-IP Ownership](../decisions/2026-08-04-p0-architecture-and-legal-ip-ownership.md)、[Product Legal／IP Governance](../01-governance/product-legal-ip-governance.md)、[Runtime ECS Design Closure Review](runtime-ecs-design-closure-review.md)、[Runtime ECS Static Definition／Entity Reference Boundary](../decisions/2026-08-03-runtime-ecs-static-and-entity-reference-boundary.md)、[Initial Morph Capability Boundary](../decisions/2026-08-03-initial-morph-capability-boundary.md)、[glTF Import Dependency Baseline](../decisions/2026-08-03-gltf-import-dependency-baseline.md)、[MCP Current Protocol Baseline](../decisions/2026-08-03-mcp-current-protocol-baseline.md)、[AI Production Orchestration Ownership](../decisions/2026-08-04-ai-production-orchestration-ownership.md)、[Android Adaptive Game Window Baseline](../decisions/2026-08-03-android-adaptive-game-window-baseline.md)、[AI-readable Asset／Memory／Async Loading Alignment](../decisions/2026-07-28-ai-asset-memory-async-alignment.md)、[AI-native C++ Product Identity](../decisions/2026-08-03-ai-native-cpp-product-identity.md)、[Product Lifecycle／Product Security Ownership](../decisions/2026-07-29-product-lifecycle-security-ownership.md)、[Product Release／Publication Authority Ownership](../decisions/2026-07-30-product-release-publication-authority.md)、[Advanced Rendering／Multiplayer Ownership](../decisions/2026-07-29-advanced-rendering-multiplayer-ownership.md)、[Product Lifecycle](../00-product/product-lifecycle.md)、[Product Release Decision](../00-product/product-release-decision.md)、[Product Publication／Completion](../00-product/product-publication-completion.md)、[Product Privacy／Data Governance](../01-governance/product-privacy-data-governance.md)、[Product Security](../01-governance/product-security.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core Architecture](../02-foundation/core-architecture.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Naming／Project Layout](../02-foundation/naming-project-layout.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Project State](../03-authoring/project-state.md)、[AI Production Orchestration](../03-authoring/ai-production-orchestration.md)、[Game Production Loop](../03-authoring/game-production-loop.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)、[Developer Testing](../03-authoring/developer-testing.md)、[Editor Workspace／UX](../03-authoring/editor-workspace-ux.md)、[Editor UI Framework](../03-authoring/editor-ui-framework.md)、[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)、[Native Game Module](../03-authoring/native-game-module.md)、[Persistence／Save](../04-runtime/persistence-save.md)、[Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[Physics](../05-simulation/physics.md)、[World](../06-rendering/world.md)、[Render Graph](../06-rendering/render-graph.md)、[Materials](../06-rendering/materials.md)、[Advanced Light Transport](../06-rendering/advanced-light-transport.md)、[Terrain／Foliage](../06-rendering/terrain-foliage.md)、[Network Transport／Connection](../09-networking/network-transport-connection.md)、[Multiplayer Authority／Replication](../09-networking/multiplayer-authority-replication.md)、[Windows](../07-platform/windows.md)、[Mobile Common](../07-platform/mobile-common.md)、[Android](../07-platform/android.md)、[Apple](../07-platform/apple.md)、[UI／Text](../07-platform/ui-text-localization-accessibility.md)、[Gameplay Feature Packs](../08-packs/gameplay-features.md)、[RPG Genre Pack](../08-packs/rpg.md)、[Scenario／Stage](../08-packs/scenario-stage.md)
- 根拠区分: project-review／official-spec comparison
- 外部根拠確認日: 2026-08-04

> 本書は実装計画ではない。表の順序、Closure ID、推奨判断は実装順、Product Phase、Work Package、担当、工数または日程を意味しない。本書は既存Ownerの正本を置き換えず、明白な文書不整合の修正と、未決定事項を未決定のまま一意に追跡する。

## 1. 監査結論

Miraikanai EngineのArchitecture計画は、Owner分離、current／target区分、typed reference、fail-closed、Evidence、Target別Qualificationという設計原則では強い。一方、RepositoryのOwner文書は`review`、実装状態は`absent`であり、Markdown上の型、Registry、Operation、固定値をcurrent実装または利用可能なCapabilityとして扱えない。

| 観点 | target design | current状態 | 結論 |
|---|---|---|---|
| AIによる概念理解 | strong | Markdown reviewのみ | Owner、identity、revision、authority、安全境界は説明可能 |
| AIによる機械解決 | strong contract intent | incomplete | Inventory、Explain Projection、Capsule、Schema、query Toolが未materialize |
| AI-native Game production loop | `closed-in-target-design` | Owner／Operation／Fixture／Receipt／Project absent | Intent→理解closure→staging→test／playtest→evaluation→iteration→acceptanceを同じCandidate／Project lineageへ閉じ、manual／AI continuityを分離せず接続 |
| AI production orchestration | `closed-in-target-design` | Owner review、Run／Workflow／Registry／Store／Operation／Host absent | Run／Attempt／Result／Checkpoint、Workflow／Context、production route／loop owner、surface parityを一意化し、Security Task、OperationTask、Project commit、Evidence、domain completionと別状態にする |
| P0 canonical Architecture | `closed-in-target-design` | Inventory／Specification／Closure artifact absent | Product PlanがActive Product Definition、First Playable、Minimum Executable CoreからP0 membershipを導出し、各Subsystemを一意Ownerと16 axisへ束縛する。Architecture closureを実装またはReleaseへ数えない |
| Product Legal／IP governance | `closed-in-target-design` | Legal source snapshot／Profile／Decision／human review absent | jurisdiction／market／channel／role、rights、Independent Design、authorized human Decisionを一意Ownerへ閉じ、Domain EvidenceとLegal Approvalを分離する |
| Product cross-surface conformance | `closed-in-target-design` | Client／Agent Profile、Suite、Fixture、Receipt absent | GUI／CLI／headless／Native SDK／external IDE／MCP／AI automationを別surfaceとし、built-in／first-party／各external Agent versionを個別Profileで検証する |
| First Playable | `closed-in-target-design` | exact Architecture coverage projectionはOwner反映済み、Profile／MCD／RPG contract／Project／Package／Evidence absent | Windows desktop／MSIX、compact 2D command RPGのplayable path、exact五Feature family、required coverage、explicit exclusion、manual／AI journey、Playtest／Approvalを定義し、Shooter／3D／Mobile／別Genreを代用しない |
| C2／Mobile Product boundary | `closed-in-target-design` | Product Definition／Profile／2D・3D Reference／Mobile runtime／Package／Store Evidence absent | Windows／Android／Appleの各2D＋3D、Project C++／Shader、en-US／ja-JP、Microsoft Store／Google Play／App Storeをsame Candidateへ閉じ、Product C1とMobileを混同しない |
| Minimum Executable Core | `closed-in-target-design` | C++／Build／Package／Qualification absent | 30 exact roleと九scenarioでworldless headless boot、determinism、cancel／fault／shutdown、sanitizer／negative inputを同一Candidateへ閉じる |
| Creative expression | broad inside public Capability | design only | 2D／3D／nonspatial／procedural、Genre非依存Gameplay、Project C++／Shaderを許す一方、任意plugin／private API／JITは意図的に除外 |
| Product lifecycle | `closed-in-target-design` | Schema／Operation／Template／Sample／Documentation／Receipt absent | bootstrap、surface parity、update、repair、support、NOTICE、release acceptanceを専用Ownerへ一意化 |
| Product security | `closed-in-target-design` | Registry／Case Store／Operation／Fixture／Receipt absent | threat ownership、baseline、vulnerability response、security update／disclosure／incidentを専用Ownerへ一意化 |
| Scene／Level authoring | `closed-in-target-design` | Schema／Projection／Operation absent | `Level Workspace`をpresentation、Scene Sourceを再利用instance／nested composition／typed override／explicit rebaseのauthorityへ分離 |
| Native iteration | `closed-in-target-design` | Native build／Preview executable／Receipt absent | Shipping static link、Preview DLLのstartup一回load、変更時GameHost restart。in-process Hot Reloadは採用しない |
| Runtime描画／物理／Memory | detailed target | implementation absent | 意味、lifetime、backend境界、Qualificationは計画済み |
| Advanced Rendering | `closed-in-target-design` | Owner／Schema review、implementation absent | Light transport semantic channelとchannel-local Technique binding、Terrain／Foliage branch別fallback outcome、Render Graph execution、World／LOD／Asset境界を分離し、GI／Reflectionsを独立Futureにした |
| Multiplayer | `closed-in-target-design` in Future scope | Owner／Schema review、implementation absent | Transport／Connection、Provider generation／peer exchange／pair identity、gameplay session／authority／replication、Dedicated Runtime Target、Online Servicesを分離。current C1／C2ではNetworkingをactivateしない |
| Future Target closure | `closed-in-target-design` | Product inventoryはOwner反映済み、execution Registry／Promotion／Receipt absent | 30 Futureを24 single Targetと6 Target-role bundle、構造上directな29 candidateと1 decomposition-requiredへ分類し、Owner scope逸脱、client／authority／operationsの到達不能、C2 Mobile sourceとの重複、optional role由来の過大claimを除去した |
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

`ARCH-C145`反映後の2026-08-04 current candidate bytesでは、Architecture Markdown 104件、文書ID102件／unique 102件（Navigation用README 2件を除く）、Owner文書65件、Owner Header欠落／順序／state不一致0件、相対Markdown path 3,415件の未解決0件、fragment link 360件の未解決0件、65 Owner間の規範依存332 edgeのOwner外参照0件／cycle 0件、unbalanced code fence 0件を確認した。新Ownerはdirect normative dependency 8件、direct consumer 2件で、13 target type familyのcanonical definitionは新Owner一件だけである。production／security route分離、fallback新Run、suspend Outcome、Checkpoint非循環、Run completion非代替、model usage／monetary route branch、Generation Receipt signer payload境界、Shipping exclusion集合、旧Security caller-route fieldのSchema用法0件、旧single AI-Orchestrator executable前提0件も検査し、error 0である。これはlocal read-only Markdown解析であり、committed Generator／Schema validator／CI、materialized Registry／Operation／Service、実装またはQualificationの代用ではない。

`ARCH-C146`～`ARCH-C148`反映後の2026-08-04 current candidate bytesでは、Architecture Markdown 106件、文書ID104件／unique 104件（Navigation用README 2件を除く）、Owner文書66件、Owner Header欠落／順序／state不一致0件、相対Markdown path 3,556件の未解決0件、fragment link 356件の未解決0件、66 Owner間の規範依存340 edgeのOwner外参照0件／cycle 0件、全docs Markdown 111件のunbalanced code fence 0件を確認した。21 target type familyのcanonical definitionは各exact一Owner、P0 fixed rootは34件／missing Owner 0件、Architecture axis 16件、Product surface 7件、AI surface 6件、Agent Host subset 3件、Legal category 16件で、各集合にduplicate 0件である。これはlocal read-only Markdown解析であり、committed Generator／Inventory／Schema validator／CI、materialized P0／Legal／Conformance Artifact、実装、法務判断またはQualificationの代用ではない。

`ARCH-C149`～`ARCH-C150`反映後の2026-08-04 current candidate bytesでは、Architecture Markdown 106件、文書ID104件／unique 104件（Navigation用README 2件を除く）、Owner文書66件、Owner Header欠落／順序／state不一致0件、相対Markdown link 3,620件の未解決0件、fragment link 405件の未解決0件、66 Owner間の規範依存340 edgeのOwner外参照0件／cycle 0件、全docs Markdown 113件のunbalanced code fence 0件を確認した。Futureは30 unique行、24 `single_target`／6 `target_role_bundle`、10 profile、21 role、12 role relation、direct prerequisite 19 edgeで、missing Owner／prerequisite、self edge、cycle、parent-profile／candidate-kind／profile-applicable-prerequisite union差は0件である。履歴31件との差はC2へ移したMobile Project Source一件のclean removal、Owner差はscope是正した四件、前提差はsmall co-op／rollbackのdedicated-profile edgeを親unionへ明記した二件だけである。これはlocal read-only Markdown解析であり、committed Generator／Inventory／Schema validator／CI、materialized Product Definition／Target Profile／Registry／Promotion carrier／Fixture／Receipt、実装またはQualificationの代用ではない。

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
     AI Production Tooling Host process group
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
| Product Plan P0 root ↔ 全Subsystem Owner | `closed-in-target-design`／Inventory・Closure artifact absent | Product PlanはP0 membership／16-axis projectionだけを所有し、各Subsystemとshared Contract Ownerがaxisの意味を保持する |
| Product Legal／IP ↔ Toolchain／Asset／Privacy／Security／Lifecycle／Evidence | `closed-in-target-design`／legal review・materialization absent | Domain Ownerはterms／provenance／data／security／distribution payload、Legal Ownerはsame-scope applicability、Independent Design、authorized human Decision／Headを所有する |
| Product Lifecycle ↔ AI Production Orchestration ↔ Verification | `closed-in-target-design`／Profile・Suite・Receipt absent | LifecycleはProduct surface／required client profile、OrchestrationはAI surface／Agent Host Profile、VerificationはSuite／Fixture／Receipt identityとfreshnessを所有する |
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
| AI Production Orchestration ↔ Security／Verification／Project／Operation／Game Production／Editor | `closed-in-target-design`／materialization absent | OrchestrationはRun／Workflow／immutable Context／production route／loop owner、SecurityはTask／authorization／caller route、VerificationはEvidence、ProjectはChangeSet／commit、Executable ContractsはOperation、Game Productionはdomain closure、Editorはpresentationを一意に所有 |

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
| `ARCH-C28` | same-target規則でlarge-session clientとMMO全Targetが到達不能 | `corrected-in-review`／target design closed、materialization absent | Product Plan §8のreceipt-free inventoryで24 `single_target`／6 `target_role_bundle`、exact bundle profile、role cardinality／relation、direct Future prerequisite role mapping、required non-empty roleを閉じる。旧execution RegistryをProduct正本または実在Artifactにしない |
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
| `ARCH-C42` | 34 required Operation familyのActivationは閉じるが全journey acceptanceが未閉包 | `closed-in-target-design`／materialization absent | Product PlanがClaim Scope×family×Operation×surface×Host／Target×scenario×branch×Evidence classのrequired tupleとforbidden surfaceを完全分割し、Lifecycleが全tupleとtyped Journey Evidenceをexact set equalityにする。Native SDK／external IDE／MCP／AI automation、expected rejection、failure recoveryを別surface／familyで代用しない |
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
| `ARCH-C115` | Product Planのcompact 2D command RPG First PlayableをExecution ProposalがShooter Fixtureで代用していた | `corrected-in-review`／execution projection intentionally absent | ProposalはShooterのlegacy Phase／GateをRPG、C1またはProduct-facing Phase exitへ投影しない。Product／RPG OwnerのArchitecture requirement projectionは`ARCH-C149`で閉じるが、RPG Fixture／WP／Capability／Gate／Target execution bindingは実装計画に相当するため本作業で作成せず、未materializedとして保持する |
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
| `ARCH-C137` | AIがGame intentを収集してもBrief／Spec／Question／Assumption／Decision／Requirement traceabilityとstaging readinessへ閉じる一意Ownerがない | `closed-in-target-design`／materialization absent | [Game Production Loop](../03-authoring/game-production-loop.md)がGame production subject、Intent session／draft、Brief／Spec／Decision Ledger、Understanding Closureを一意所有する。bootstrap prepublicationをprivate stagingに限定し、revision 1のatomic publication後にtraceability／Graph／Understanding Closureを完成させ、会話logやAI自己申告をProject authorityまたは理解完了にしない |
| `ARCH-C138` | technical test、human Playtest Observation、Experience Evaluation、Iteration Decision、Human Gameplay Approvalが相互代替可能 | `closed-in-target-design`／materialization absent | Game Production LoopがObservation→Evaluation→Decisionを同一Candidate／Project lineageで分離し、Developer TestingとHuman Approvalを別Ownerへ保持する。新Iterationは新Authorizationを要求し、Closure発行をCommit／Releaseへ昇格しない |
| `ARCH-C139` | AI game generationがGame composition、external content生成、Project Source生成を一つの曖昧なclaimとして扱える | `closed-in-target-design`／claim Evidence absent | [Product Plan](../00-product/product-plan.md)が`AiGameGenerationLaneV1`三laneとnon-substituting claim scopeを所有する。initial First Playableはexact `{ai_composed_game}`だけを要求し、Asset／Source生成supportを主張しない |
| `ARCH-C140` | Product C1がcompact RPG、Shooter、3D、manual-only、AI-onlyのどれで成立するか一意でない | `closed-in-target-design`／Architecture coverage closed、execution・materialization absent | Product Planのreceipt-free `FirstPlayableDefinitionV1`がcompact 2D command RPG、en-US／ja-JP、keyboard／controller、manual／AI continuity、clean install／offline、Game Production Closureをexact化する。`ARCH-C149`がWindows Target、MSIX、playable path、五Feature family、required coverage、explicit exclusionまで補完し、Product Lifecycleが同じCandidate／Project revisionのEvidenceを集約する |
| `ARCH-C141` | Evidence freshnessがassumption shadow shape、catalog参照、Verification stateの間で重複・参照切れ | `corrected-in-review`／Evidence materialization absent | [AI Verification／Provenance](../01-governance/ai-verification-provenance.md)が`EvidenceEffectiveStateV1 = fresh | expired | revoked | invalid`とpriorityを一意所有し、AI Security／assumption／catalog consumerはexact Refと同stateへ統一する |
| `ARCH-C142` | Gameplay System dependency、実装選択、authoritative State ownerをRuntime Package／Scheduling／Fixtureが名前や登録順から再構成可能 | `closed-in-target-design`／runtime materialization absent | [Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)が`GameSystemDependencyGraphV1`、`SystemImplementationSetV1`、`GameStateOwnerProjectionV1`を一意所有し、Runtime Package／Schedulingはexact三Refを同じProject revision／Contract set／Targetへ束縛する。System／authoritative State 0件のUI-only／worldless headlessも完成empty closureで表現し、required System集合がある場合はset equalityでempty退化を拒否する |
| `ARCH-C143` | Engine Coreが何を満たせば実行可能かがProduct First Playableや全Engine完成と混同される | `closed-in-target-design`／Qualification absent | [Core Architecture §13](../02-foundation/core-architecture.md#13-minimum-executable-core-target-closure)が30 exact member roleと九Qualification scenarioを所有する。worldless headless bootをRuntime Package／Schedulingへ束縛し、Rendering、Genre、First Playable、Releaseを代用しない |
| `ARCH-C144` | stale節参照、Index件数、Closure Review末尾番号、planning family count、外部tool current表現がreference integrityを破る | `corrected-in-review`／generated Inventory・validator absent | Architecture Indexを64 Ownerへ更新し、Memory→Toolchain §2.6、Performance→Executable Contracts §8.2、Game Production→Project State §3.2、Review末尾§12–§13へ修正、planning ledgerを27 family／207 unique候補へ実行行数どおり統一した。候補partitionは先行67件＋旧Domain回収十表94件＋追加32件＋Game production 14件である。CMake 4.4.1をselected pinとして維持し、2026-07-31公開の4.4.2を「自動current pin」にしない |
| `ARCH-C145` | AI-native制作のRun／Attempt／Result／Checkpoint、Workflow／Context、production route／loop owner、built-in／external surface parityを一意に所有するOwnerがなく、Security Task、OperationTask、Project commit、Evidence、Game Production completionと状態が混同可能 | `closed-in-target-design`／materialization absent | [AI Production Orchestration](../03-authoring/ai-production-orchestration.md)が制作実行の一意Ownerとなり、[AI Production Orchestration Ownership](../decisions/2026-08-04-ai-production-orchestration-ownership.md)が既存Ownerへの分散、Game Production吸収、Security吸収を退ける。Game Production LoopとEditorだけが直接規範依存し、Security／Verification／Project／Executable Contracts／Core／Toolchain／Performance／Debug／Runtime Packageは既存正本を維持する |
| `ARCH-C146` | Active Product Definitionに対するP0 Subsystem membershipと、各Subsystemの共通Architecture軸を判定するcanonical rootがない | `closed-in-target-design`／Inventory・Specification・Closure artifact absent | [Product Plan §3.3](../00-product/product-plan.md#p0-canonical-architecture)が固定34 rootとProduct-derived Owner集合、16-axis exactly-once binding、理由付きN/A、target-design closure predicateを所有する。Subsystem意味は各Ownerに残す |
| `ARCH-C147` | copyright、trademark、patent FTO、trade secret、third-party／AI terms、Independent Design、Product claimをRelease法域へ束縛する一意Legal／IP authorityがない | `closed-in-target-design`／legal materialization・authorized human review absent | [Product Legal／IP Governance](../01-governance/product-legal-ip-governance.md)がSource Snapshot、Jurisdiction／Applicability Profile、16 category binding、Independent Design Subject、Review Subject、signed Decision／Headを所有する。Lifecycleはdomain inputを供給し、Release／Publicationがsame-scope current Approvalをread-backする |
| `ARCH-C148` | Product surfaceがSDK、MCP、AI Agentを`AI-MCP`へcollapseし、generic transport成功を個別Agent／versionへ代用できる | `corrected-in-review`／Client・Agent Profile、Suite、Fixture、Receipt absent | Product Plan／Lifecycleを7 Product surfaceへ分離し、Lifecycleのversioned Client ProfileとOrchestrationのAgent Host Profileをmappingする。generic MCP、Native SDK、built-in、first-party、各external Agent versionへ個別Receiptを要求する |
| `ARCH-C149` | initial First PlayableのHost／runtime Target／Distribution、playable path、RPG Feature exact subset、minimum gameplay coverage、explicit exclusionがOwner再編時に要約され、Inputのcross-target C1／Windowsのcross-distribution C1とProduct C1を混同できた | `corrected-in-review`／exact Architecture boundary closed、materialization absent | Product Plan §5.1がWindows build／Editor Host、`target.windows.desktop@1`、signed internal MSIX、Title→Town→Field→Dungeon→Boss→Result、exact五RPG Feature family、required gameplay coverageとexclusionを一意化する。RPG／Gameplay Featureがcompositionとmeaning、Input／Windowsがdomain C1とProduct subset、LifecycleがEvidence read-backを分離する。実装、実装計画、WP、Fixture、Registry、Receiptは作成しない |
| `ARCH-C150` | C1後続対象だったC2／Mobile／Futureにexact Product Target／publication不在、Product C1とMobile C1語彙の混同、deny-only Runtime generationと必須Content Safety Refの矛盾、current C2 Mobile source requirementと旧Future一件の重複、Future closure正本消失、AI Securityへのscope外Owner集中、未materialize Promotion Manifest参照が残った | `corrected-in-review`／exact Architecture boundary closed、materialization absent | Product Plan §5.3がC2のHost、Windows／Android／Apple、en-US／ja-JP、各Target 2D＋3D、Project C++／Shader、三managed Store publicationを一意化する。Mobile／Platform／Project Source OwnerがC1／C2、deny-only、package／qualificationを整合する。Product Plan §8は30 unique Future、24 `single_target`／6 `target_role_bundle`をexecution Registryなしで正本化し、Authoring AI route／Security／Toolchain／domain Owner未採択を分離する |

`ARCH-C03`はArchitecture Inventory／Explain Projectionというsubjectのtarget semantics欠落ではなくmaterialization未実施を表す。ただしArchitecture全体で唯一のmaterialization blockerではない。`ARCH-C04`のOperation／Capsule／Eval、`ARCH-C05`のECS Schema／Binding／Fixture、`ARCH-C10`のCI／device pool、`ARCH-C12`、`ARCH-C115`のexecution projection、`ARCH-C116`～`ARCH-C123`に残るSchema／runtime／dependency／Conformance／device Evidence materialization、`ARCH-C124`～`ARCH-C134`／`ARCH-C136`の未materialized contract／Evidence／distribution、`ARCH-C135`の選定済みだが未materializeのProduction Asset dependency closure、`ARCH-C137`～`ARCH-C143`のGame production／claim／First Playable／freshness／Gameplay graph／Minimum Coreに残るSchema／Operation／Project／Fixture／Receipt／runtime／Qualification、`ARCH-C144`のgenerated Inventory／validator、`ARCH-C145`のRun／Workflow／Registry／Store／Operation／Host／Receipt、`ARCH-C146`のP0 Inventory／Specification／Closure、`ARCH-C147`のLegal source／Profile／Decision／human review、`ARCH-C148`のClient／Agent Profile／Conformance Suite／Receipt、`ARCH-C149`のTarget／Locale Profile record、MCD、RPG／Feature contract、Project、Package、Fixture、Receipt、`ARCH-C150`のC2 Product Definition、Android／Apple Project Source contract／artifact、三Target×2D／3D Reference、Mobile runtime／device、三Store publication／support Evidence、Future materialization、ならびにLifecycle／Security Ownerが明記する未materialized Artifactを独立に保持し、一件の状態を他件の代表にしない。

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

`ARCH-C28`は単一Target prerequisiteとcross-target Product compositionを`single_target | target_role_bundle`へ分け、`ARCH-C150`のpost-C1再監査でcurrent Product Planへ正本を戻した。direct candidateのsingle 23行はsame exact Target、persistence、small co-op、rollback、large-session、MMO、managed external Hostの六行はexact profile、role Target集合、全role relation、DAG末端まで再帰展開するdirect Future前提、bundle前提role mapping、required non-empty roleを一つのpromotion／claim release単位にする。24番目のsingle closureであるunrestricted scriptingはcandidate kind上限だけを保持する`decomposition_required` subjectで、直接promotionしない。旧`FutureTargetClosureRegistryV1`は非規範execution候補であり、target-design closureまたはmaterialized Registryの証拠にしない。

small co-op／rollbackのheadless Targetはdedicated profileだけが選べ、Dedicated Target Futureをprofile固有前提にする。large-sessionのclient Targetとauthority Target、MMOのdesktop clientとdistributed authority／operationsをflat intersectionへ潰さない。一roleのReceipt、Target kindだけの一致、別profile間の混成、implicit cross product、optional artifact roleが空のmanaged Source buildからProduct claimを解放しない。Registry、Profile Artifact、Promotion Manifest、Activation、Receiptは未materializeである。

### 9.16 Product release authorityとactual publication

`ARCH-C29`～`ARCH-C31`は[Product Release／Publication Authority Ownership](../decisions/2026-07-30-product-release-publication-authority.md)の四段分離を採用した。Product Planはreceipt-free Definitionとrequired universe、Lifecycle／Securityはsame-release Manifest／Acceptance、Product Release Decisionはqualified authorityの署名済みpre-publication authorization、各Platformはsigning／submission／public read-back Receipt、Product Publication／Completionはrequired channel set equality、effective publication time、support開始、withdrawal／supersession、authoritative Completionを所有する。

Release Decision前のrequired Evidenceへpublic read-backを混ぜず、Release Requirement Projection内の独立したrequired publication channel universeとして固定する。Platformはunsigned Subject、approval label、issue stateまたはbare hashを受理せず、Publicationはupload／Store approvalだけを`published`へ昇格しない。全型、Authority、Key、Receipt、Publication storeは未materializeである。

### 9.17 AI-native C++ Product identityとindependent design

`ARCH-C124`はprovider-neutralなtyped contract core案を採用した。Product PlanがAI-native claimと第三者向けminimum surface、Executable Contractsがcanonical Operation、Project Stateがtransaction、AI Securityがauthorization／approval、AI VerificationがEvidence、Gameplay Programming Modelがstructured data／bounded C++の選択、Editorがmanual／AI interaction、Compatibilityがinitial V1 direct definitionと公開後evolutionを所有する。

Unity、Unreal Engine、Godotの公式資料は製品surfaceとfailure modeのgap discoveryに限定し、型、API、Scene model、Plugin形式、UI layout、名称、workflow、defaultまたはcreative expressionをMiraikanaiへ移植しない。initial V1に過去draft／外部Engine互換alias、dual reader、Project importerまたはmigration layerを作らず、初回公開後は実consumer Inventoryなしにclean breakを適用しない。これはMiraikanaiのproject-decisionであり、外部組織の公式推奨ではない。

### 9.18 PLAN-AUDIT-20260803の設計判断closure

`ARCH-C115`～`ARCH-C123`は、一つの包括Decisionへ吸収せず、subjectごとのOwnerとDecisionへ分離して判断した。`ARCH-C115`はShooter EvidenceによるRPG代用を修正した。後続`ARCH-C149`はRPG execution projectionを作らず、承認済みRPG-first Product方向からOwner再編時に欠落したexact Architecture boundaryだけをProduct／RPG／Feature／Input／Windows／Lifecycle Ownerへ戻した。Fixture、WP、Capability、Gate、Target execution binding、順序または工数は引き続き未作成である。`ARCH-C116`／`ARCH-C117`は[Runtime ECS Static Definition／Entity Reference Boundary](../decisions/2026-08-03-runtime-ecs-static-and-entity-reference-boundary.md)、`ARCH-C120`は[Initial Morph Capability Boundary](../decisions/2026-08-03-initial-morph-capability-boundary.md)、`ARCH-C122`は[MCP Current Protocol Baseline](../decisions/2026-08-03-mcp-current-protocol-baseline.md)、`ARCH-C123`は[Android Adaptive Game Window Baseline](../decisions/2026-08-03-android-adaptive-game-window-baseline.md)へ採用理由を記録し、current contractは各Ownerだけに置く。

`ARCH-C118`のProvider generationはFuture [Network Transport／Connection](../09-networking/network-transport-connection.md)、`ARCH-C119`の独立承認境界は[Product Plan](../00-product/product-plan.md)、`ARCH-C121`のKTX／FLAC license／build／distribution evidence境界は[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)が所有する。これらはtarget designまたはclassificationを閉じたものであり、Schema、Registry、Fixture、Receipt、dependency archive、Build input、runtime、device／Conformance Evidence、QualificationまたはActivationの実在を意味しない。

### 9.19 glTF Import dependency baseline

`ARCH-C135`は[glTF Import Dependency Baseline](../decisions/2026-08-03-gltf-import-dependency-baseline.md)により、Production Asset parserをcgltf single path、tangent generatorを未変更の原典MikkTSpace、仕様適合性oracleをDevelopment／Qualification専用Khronos Validatorへ分けた。Toolchainがexact version／commit／license／lock Gate、Asset Lifecycleがbounded Source／Worker／typed IR／validation／Receipt、Materialsがprovided／generated tangent、normal texture UV、handednessの意味を一意に所有する。

parserが理解するfeatureをEngine対応へ自動昇格せず、外部Scene object、URI resolver、C++型、他EngineのImporter workflowをMiraikanaiの正本にしない。single parserを別Backendへfallbackせず、MikkTSpace per-corner出力を既存indexへ平均しない。これはArchitecture上の選定だけを閉じる。dependency archive／hash、license bundle／NOTICES、Toolchain lock、SBOM、Adapter、Schema、Fixture、Receipt、Build、Khronos conformance、cross-host determinism、QualificationおよびActivationは未materializeであり、全glTF importは引き続きfail closedである。

### 9.20 AI Production Orchestration ownership

`ARCH-C145`は[AI Production Orchestration Ownership](../decisions/2026-08-04-ai-production-orchestration-ownership.md)により、Authoring層の専用[AI Production Orchestration](../03-authoring/ai-production-orchestration.md) Ownerを採用した。Run／Attempt／Outcome／Result／Checkpoint、Workflow Definition／Registry／Binding、immutable Run Context、production route、loop owner、fallback／child run、conversation非正本性、interaction profile、surface parity、Authoring／shipping境界を同じ制作実行subjectへ閉じる。

Security Task／Authorization／Approval／Caller Context、Verification Evidence、Project revision／ChangeSet／Commit、canonical Operation／OperationTask、Game ProductionのBrief／Spec／Playtest／Iteration、Editor presentation、system capacity／telemetry／runtime packageは既存Ownerに残す。Runの`completed`はtyped Resultの確定だけを表し、Security Task完了、Project commit、Evidence acceptance、domain acceptance、Qualification、ReleaseまたはProduct completionへ昇格しない。production routeは`deterministic_automation | first_party_agent | standard_external_agent | managed_external_host`、security caller routeは`engine_provider_adapter | standard_external_mcp | managed_external_host`の別型であり、明示mapping以外から補完しない。

Built-in AI制作Console、CLI／SDK、standard external Agentは同じcanonical OperationとRun semanticsを消費し、built-inだけのprivate writerを持たない。MCPは外部Transport AdapterであってEngine操作正本、Workflow正本またはProject状態正本ではない。Local／Cloud／external hostはdeployment形態であり、権限またはEvidence強度を表さない。AI tooling不在または失敗時にもmanual Editor／CLI／SDK journeyを閉じず、Authoring Orchestrator、Agent、Workflow／Run Store、credential、compiler／signer／write gatewayをShipping packageへ含めない。これらはtarget Architecture判断であり、Schema、Registry、Store、Service、Host、Operation projection、Fixture、Receipt、実装、QualificationまたはActivationは未materializeである。

### 9.21 P0 Canonical Architecture／Product Legal-IP ownership

`ARCH-C146`～`ARCH-C148`は[P0 Canonical Architecture／Product Legal-IP Ownership](../decisions/2026-08-04-p0-architecture-and-legal-ip-ownership.md)により閉じる。P0を実装Phaseへせず、[Product Plan §3.3](../00-product/product-plan.md#p0-canonical-architecture)がActive Product Definition、First Playable、Minimum Executable Coreからmembershipを導出し、各Subsystemを一意Ownerと16 Architecture axisへ束縛する。Axis意味、数値、API、state、failure、migration、Evidence、Conformanceは既存Ownerから移動または複写しない。

[Product Legal／IP Governance](../01-governance/product-legal-ip-governance.md)はjurisdiction／market／channel／Product・AI role、official source snapshot、16 non-substituting Legal category、Independent Design、Domain Evidence、authorized human Decision／Headを一意所有する。Toolchain、Asset、Project、Privacy、Security、Lifecycle、Verificationは各domain payloadを保持し、Legal Ownerがそれらを法的に自己生成または上書きしない。Architecture、AI、scanner、SBOM、provenance、独立作成またはrelease authority署名を法令遵守、権利処理、非侵害、FTOまたはtrademark clearanceへ読み替えない。

Product surfaceは`editor_gui | cli | headless | native_sdk | external_ide | mcp | ai_automation`、AI surfaceは`editor_builtin | native_sdk | cli | mcp | first_party_agent_host | external_agent_host`として別型を維持する。LifecycleのClient ProfileとAI OrchestrationのAgent Host Profileで明示mappingし、generic MCP、Native SDK、built-in、first-party、各external Agent／versionを個別にqualifyする。これはtarget semanticsであり、P0／Legal／Client Profile Schema、Inventory、Generator、Decision、Suite、Fixture、Receipt、C++、Build、CI、測定、法務意見またはReleaseは未materializeである。

## 10. 本Reviewに伴う文書整合

| 文書 | 変更内容 |
|---|---|
| Architecture Index | Owner文書を66件へ更新し、Product Legal／IP Governance、AI Production Orchestration、Product Release Decision／Publication・Completion、Privacy、Developer Testing、Advanced Light Transport／Terrain・Foliage／Network Transport／Multiplayer AuthorityとOwnership Decisionへのnavigationを追加 |
| Product Plan／Product Legal-IP／Lifecycle／Release／Publication／AI Orchestration | P0 membership／16-axis closure、Legal applicability／Decision、7 Product surfaceとversioned Client／Agent Host Profile別Conformanceを一意Ownerへ接続。materialization、法的承認、Qualificationまたは実装は行わない |
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
| Gameplay Feature Packs／RPG Genre Pack／Product Plan／Lifecycle／Input／Windows | Reusable RPG Feature、Genre composition、通常Reference Game、Product acceptanceを四層化し、initial First Playableのexact Windows／MSIX boundary、playable path、五Feature family、required coverage、explicit exclusion、domain C1との非代替を閉じる |
| Runtime ECS | initial V1の直接Owner、Gameplay authoringとの境界、Definition Closure条件を追加 |
| Render Graph／World／Animation／Runtime Asset | Sprite／Tile／frame／residency／packet／stable sort／batchの2D authority chainを追加 |
| Native Game Module | restart-based Windows PreviewをHot Reloadのcleanな意味置換として明記し、in-process unload／replacement／patch禁止を維持 |
| Windows／Mobile Common | legacy `build/` rootを除去し、Application lifecycleとpresentation surface availabilityを別stateへ修正 |
| Toolchain／Dependencies | Target別C++ Build optimization closureをKnown unresolved registerへ追加し、Owner本文のexact Target×Configuration policyでtarget designを閉鎖 |
| Toolchain／Asset Lifecycle／Materials／Decision Log | cgltf／原典MikkTSpace／Khronos Validatorをsingle dependency closureへ選定し、version所有、Worker／typed IR、tangent意味、materialization Gateを分離。選定だけからImport利用可能性を推論しない |
| Advanced Light Transport／Render Graph／Lighting／Materials／Post／Environment | Source意味、channel別Technique／fallback、generic effect、physical executionのauthorityを分離 |
| Terrain／Foliage／World／LOD／Runtime Asset | TerrainとFoliageのSource／Artifact branchを独立させ、partition／selection／residencyを既存Ownerへ維持 |
| Runtime Package／Network Transport／Multiplayer | Dedicated Target、Connection／delivery、gameplay session／authority／replicationを三分し、Online Servicesを非正本化 |
| Product Plan／Execution Registry proposal／Shooter | Future 30行、Reflection claim、24 single-target／6 Target-role bundle、構造上directな29 candidate／1 decomposition-required、direct Future prerequisite、required role、promotion／claim release boundaryを正本化し、execution Registry候補を非規範へ戻す |
| Product Plan／Release Decision／Product Lifecycle | Requirement単位Evidence non-empty、category total mapping、publication route missing条件、semantic admissibility read-backを追加 |
| AI Verification／Executable Contracts／Privacy／Pack | Verification scope／subject contract／branch、Operation Receipt subject contract／request、MCD `data_flow` Envelope／Refを一意化し、hash-only／invalid tuple／local narrow Refを拒否 |
| Product Plan／Gameplay Programming Model／Compatibility／Decision Log | AI-native claim、汎用Engine minimum surface、manual continuity、structured data／bounded C++、外部Engine非模倣、initial V1 direct definitionと公開後consumer保護を一意なOwnerへ接続 |
| AI Production Orchestration／Game Production Loop／Editor Workspace／AI Security／Verification／Project State／Executable Contracts／Core／Toolchain／Performance／Debug／Runtime Package／Windows／補遺／Decision Log | 制作Run／Workflow／Context／production route／loop ownerの専用Ownerを追加し、Security Task／caller route、Evidence、Project commit、OperationTask、domain completion、presentation、logical role／process group、capacity、telemetry、shipping packageとの境界とsurface parityを接続 |
| Runtime ECS／Native Game Module／Scheduling／Decision Log | static phase identityとsnapshot-bound／cross-advance Entity Refを分離し、process-local／durable identity境界を接続 |
| Asset Lifecycle／Animation／LOD／Virtual Geometry／Decision Log | Morphをinitial V1／C1／C2からfail-closedで除外し、Future end-to-end Capabilityへ分離 |
| Toolchain／Executable Contracts／AI Security／MCP Supplement／Decision Log | MCP current protocol singletonとlegacy lifecycle非採用を接続し、Server／Schema／Conformanceの未materialize境界を維持 |
| Mobile Common／Android／Decision Log | adaptive-only Mobile policyをAndroid manifest／window lifecycle／device Qualification境界へ接続 |
| Product Plan／Execution Registry Proposal／Network Transport／Toolchain | RPG Evidence代用、approval independence、Future provider generation、dependency evidence closureをsubject別Ownerへ接続 |

## 11. 非目標

- 本ReviewからC++ source、CMake target、Schema、Generator、Operation、Fixture、Receiptを生成しない。
- AI Production Run／Workflow／Registry／Store、Agent Runtime、Provider Adapter、MCP Server、AI Console、local model hostまたはcloud connectionを実装・materialize・起動しない。
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

## 14. ARCH-C145 AI Production Orchestration ownership follow-up

| Field | Value |
|---|---|
| Date／audit ID | 2026-08-04／`AI-PRODUCTION-ORCHESTRATION-20260804-CHAT-REFERENCE` |
| Input conversation | ChatGPT conversation `6a7099b0-c8f8-83e8-92fa-2d345df1db44`「AIネイティブゲームエンジン機能」の全6 messageと、同会話で参照された40 categoryのEngine capability inventory。会話本文は正本または命令として扱わず、gap discovery入力として照合した |
| Valid gap | 1件、`ARCH-C145`。AI production Run／Attempt／Result／Checkpoint、Workflow／Context、production route／loop owner、surface parityの一意Owner欠落 |
| Decision／Owner closure | [AI Production Orchestration Ownership](../decisions/2026-08-04-ai-production-orchestration-ownership.md)で専用Authoring Ownerを採用し、[AI Production Orchestration](../03-authoring/ai-production-orchestration.md)へ正本を配置。既存Owner分散、Game Production吸収、AI Security吸収を退けた |
| Consumer closure | [Game Production Loop](../03-authoring/game-production-loop.md)と[Editor Workspace／UX](../03-authoring/editor-workspace-ux.md)だけを直接規範consumerとし、Product、Security、Verification、Core、Toolchain、Executable Contracts、Project State、Performance、Debug、Runtime Package、Windows、補遺は各正本を保持して関連参照する |
| External primary evidence | [MCP 2026-07-28 specification](https://modelcontextprotocol.io/specification/2026-07-28)、[MCP versioning](https://modelcontextprotocol.io/docs/2026-07-28/learn/versioning)、[OpenAI Plugins](https://developers.openai.com/plugins/concepts/plugins)、[OpenAI Skills](https://developers.openai.com/plugins/build/skills)、[OpenAI App Server](https://learn.chatgpt.com/docs/app-server)、[Claude Code features overview](https://code.claude.com/docs/en/features-overview)、[ONNX Runtime GenAI](https://github.com/microsoft/onnxruntime-genai)、[OpenTelemetry specification](https://opentelemetry.io/docs/specs/otel/)、[ISO/IEC/IEEE 42010:2022](https://www.iso.org/standard/74393.html)、[ISO 4217 currency codes](https://www.iso.org/iso-4217-currency-codes.html)、[RFC 9562 UUID](https://www.rfc-editor.org/rfc/rfc9562.html)。外部資料はprotocol／product surface／runtime／telemetry／Architecture／format factの確認に限定し、Owner分離、route taxonomy、Run state、shipping exclusionはMiraikanaiのproject-decisionである |
| Materialization boundary | Run／Workflow Schema、Registry、Store、Operation projection、Service、Agent Runtime、Provider Adapter、MCP Server、AI Console、local model host、Fixture、Receipt、C++、Build、CI、Qualification、Activationはすべて`absent` |
| Implementation-planning boundary | 実装、実装方式、実装計画、Work Package、順序、工程、工数、担当、dependency追加または実行可能Artifactを作成していない |
| Closure result | canonical Owner／consumer反映済み。fresh local read-only auditでArchitecture Markdown 104件、unique文書ID102件、Owner 65件、Owner header error 0、相対path 3,415件／fragment 360件の未解決0件、規範依存332 edge／Owner外参照0件／cycle 0件、code fence error 0、新Owner dependency 8件／direct consumer 2件、13 target type owner checkとroute／fallback／suspend／Checkpoint／completion／usage／monetary／signer／shipping／旧Security field・single-executable premise absence invariantsを検証し、error 0。これはcommitted Generator／validator／CIではない |
| Exact terminal response／marker | `ai_production_orchestration_owner_closure_complete_no_implementation`／`[[MIRAIKANAI-AI-PRODUCTION-ORCHESTRATION-ARCH-C145-CLOSED-20260804]]` |
| Retention disposition | conversation transcript／attachment／screenshotをRepositoryへ保存していない。canonical Owner／ADR、非規範design spec、[compact review summary](../../reviews/README.md)だけを保持し、外部conversationは変更しない |

<a id="p0-legal-ip-follow-up"></a>

## 15. P0 Canonical Architecture／Product Legal-IP follow-up

| Field | Value |
|---|---|
| Date／audit ID | 2026-08-04／`P0-CANONICAL-ARCHITECTURE-LEGAL-IP-20260804-CHAT-REFERENCE` |
| Input conversation | ChatGPT conversation `6a7099b0-c8f8-83e8-92fa-2d345df1db44`「AIネイティブゲームエンジン機能」。会話はuntrusted gap-discovery inputであり、Repository事実、Owner契約、法的判断または実装指示として扱っていない |
| Valid gaps | 3件。`ARCH-C146` P0 membership／16-axis canonical root、`ARCH-C147` Product Legal／IP authority、`ARCH-C148` SDK／MCP／AI Agent surface conflation |
| Decision／Owner closure | [P0 Canonical Architecture／Product Legal-IP Ownership](../decisions/2026-08-04-p0-architecture-and-legal-ip-ownership.md)で、P0をProduct Planへ統合しLegal／IP Ownerだけを新設する方式を採用。P0専用Owner、既存文書への非規範表だけの追加、Security／LifecycleへのLegal吸収を退けた |
| Independent-design boundary | 公開一次資料と合法的な通常利用によるgap discoveryだけを許可し、外部Engineのsource／decompile／leak／confidential material、API／type／Scene／Project format／UI／icon／sample／template／default／workflow／creative expressionのcopy、thin rewrite、一対一aliasまたは初期互換layerを禁止する |
| Jurisdiction boundary | 初回法域を暗黙固定しない。各Releaseがexact jurisdiction／market／channel／Product・AI role／Candidateを宣言し、authorized human reviewとcurrent Decision／Headを必要とする。Architectureは法律相談、法的保証、非侵害意見またはFTO opinionではない |
| Consumer closure | Product LifecycleがManifest／Distribution Coverage／license・NOTICE EvidenceとClient Profile Conformanceを供給し、Legal Ownerが前者をreview inputへ、Release DecisionがLifecycle Acceptanceとsame-scope Legal Decisionをauthority subjectへ、Publication／CompletionがLegal distribution bindingとread-time effective stateを公開scopeへ束縛する。AI Verification、Privacy、Security、Toolchain、Asset、Namingは各正本を維持する |
| Materialization boundary | P0／Legal／Client・Agent Profile Schema、Architecture Inventory／Generator／Specification／Closure、Legal source snapshot／Decision／Head、Conformance Suite／Fixture／Receipt、C++、Build、CI、benchmark、Reference Project、UX study、法務意見、trademark search、patent FTO、Release artifactはすべて`absent`または未実施 |
| Implementation-planning boundary | 実装、実装方式、実装計画、Work Package、順序、工程、工数、担当、production dependencyまたは実行可能Artifactを作成していない |
| Closure result | canonical Owner／consumer反映済み。fresh local read-only auditでArchitecture Markdown 106件、unique文書ID104件、Owner 66件、Owner header error 0、相対path 3,556件／fragment 356件の未解決0件、規範依存340 edge／Owner外参照0件／cycle 0件、全docs code fence error 0、21 canonical type owner check error 0、P0 fixed root 34件／missing Owner 0件、16 axis、7 Product surface、6 AI surface、3 Agent Host subset、16 Legal categoryのcardinality／duplicate error 0を確認した。これはcommitted Generator／Inventory／Schema validator／CI、実装、法務判断またはQualificationではない |
| Exact terminal response／marker | `p0_legal_ip_architecture_refresh_complete_no_implementation`／`[[MIRAIKANAI-P0-LEGAL-IP-ARCHITECTURE-C146-C148-CLOSED-20260804]]` |
| Retention disposition | conversation transcript／attachment／screenshotをRepositoryへ保存しない。canonical Owner／ADR、approved design spec、[compact review summary](../../reviews/README.md)だけを保持する |

### 15.1 要求領域別のcanonical判定

| 要求領域 | Canonical Owner／closure | target design | current Evidence state |
|---|---|---|---|
| 1. Engine Product Definition | [Product Plan](../00-product/product-plan.md)のActive Product Definition／First Playable／Minimum Core | `closed-in-target-design` | Definition Schema／record／generator absent |
| 2. 数値目標／性能・容量 | [Performance／Capacity](../04-runtime/performance-capacity.md)＋各Domain／Platform Owner | Owner／測定条件はtarget定義済み | provisional。Benchmark、Target measurement、Receipt absent |
| 3. P0 Subsystem／一意Owner | [Product Plan §3.3](../00-product/product-plan.md#p0-canonical-architecture) | fixed 34 root＋Product-derived集合＋16-axis predicateを`closed-in-target-design` | Inventory／Specification／Closure instance absent |
| 4. Public／Internal API | C++23、Native Game Module、Executable Contracts、Core、各Domain Owner | public／private／Operation boundaryはtarget定義済み | Header／library／ABI／API catalog／Conformance absent |
| 5. State／Lifetime／Threading／Invariant | Executable Contracts、Scheduling／Lifetime、Memory、Runtime ECS、各Domain Owner | generic表現とDomain意味を分離済み | Schema／runtime／race・lifetime Fixture absent |
| 6. Failure Atomicity | Project State、Core OperationTask、各Domain Owner | atomic publication、last-valid、不可逆effect recoveryをtarget定義済み | fault-injection Fixture／Receipt absent |
| 7. Version／Migration／Compatibility | Compatibility／Evolution＋各Domain Owner | clean break、consumer protection、migration admissibilityをtarget定義済み | migration executable／legacy consumer inventory／Fixture absent |
| 8. Evidence／Acceptance | AI Verification／Provenance＋Domain payload Owner＋Product gates | identity、semantic admissibility、freshness、non-substitutionをtarget定義済み | signed Evidence／Suite／Receipt／Release authority absent |
| 9. GUI／CLI／SDK／MCP／Agent Conformance | Product Plan、Product Lifecycle、AI Production Orchestration、AI Verification | 7 Product surface、6 AI surface、versioned Client／Agent Profile mappingを`closed-in-target-design` | Profile Registry／Suite／Fixture／Receipt absent |
| 10. Legal／IP／Independent Design | [Product Legal／IP Governance](../01-governance/product-legal-ip-governance.md) | jurisdiction、16 category、Domain Evidence、human Decision／Headを`closed-in-target-design` | legal materialization、authorized review、clearance／FTO／opinion absent |

この表の`closed-in-target-design`は意味とOwner境界だけの判定である。Engine実装、性能達成、制作体験、商用品質、法令遵守、権利処理、非侵害、Qualification、Activation、PublicationまたはProduct completionを証明しない。

<a id="first-playable-boundary-follow-up"></a>

## 16. Initial First Playable boundary follow-up

| Field | Value |
|---|---|
| Date／closure ID | 2026-08-04／`ARCH-C149` |
| Scope | 実装または実装計画を作らず、現計画で到達させるGame、required feature、explicit exclusion、current evidence gapを一意化する |
| Prior decision basis | [Document System Restructure §14.4](../decisions/2026-07-21-document-system-restructure.md#144-rpg-first-product-mvp設計の節disposition)がRPG-first Product MVPのplayable flow、gameplay coverage、Runtime cadence、explicit exclusionをProduct PlanへMergedすると記録していたが、current Ownerには要約だけが残っていた |
| Canonical closure | [Product Plan §5.1](../00-product/product-plan.md#51-first-playable-outcome)がProduct boundary、[RPG Genre Pack §4.1](../08-packs/rpg.md#41-initial-first-playable-composition)がcomposition、[Gameplay Feature Packs §4.4](../08-packs/gameplay-features.md#44-reusable-rpg-feature-family)がFeature meaning、Lifecycle／Input／WindowsがEvidence／domain C1／package境界を所有する |
| Clean-break result | public materialization前のinitial V1としてpredecessor、v0、Shooter rename、legacy alias、dual reader、migration、managed／portable fallbackを追加していない |
| Planning boundary | Product Phase、Work Package、実装Task、順序、工程、工数、担当、RPG execution Registry／Fixture／Gateを追加していない |

### 16.1 現計画で到達させるGameとcurrent実在性

| 判定対象 | 結論 | 根拠 |
|---|---|---|
| current Repositoryで今すぐ作成・起動できるGame | 0件 | 全Ownerが`review`、Engine implementation／executable Schema／Registry／Fixture／Receipt／Build systemはabsent |
| Architectureが一意に目標化した最初のGame | original、offline、Windows desktop向けcompact 2D command RPG | Product Planのexact Host／Target／MSIX、Title→Town→Field→Dungeon→Boss→Result、五RPG Feature family、locale／Input／manual・AI closure |
| そのGameから主張できる範囲 | 限定した2D First Playableだけ | 3D、Mobile、全RPG、production-ready Engine、third-party release、Store publicationまたは商用品質を代用して主張しない |

### 16.2 Required closure and missing evidence

次表は機能の実装順または優先順位ではない。各行はFirst Playableを成立させるために欠けてはならないnon-substituting closureと、current Repository evidenceの有無を示す。

| Required closure | Architecture target | Current evidence |
|---|---|---|
| Product identity／scope | Windows／MSIX、compact 2D command RPG、exact inclusion／exclusionをOwner反映済み | `FirstPlayableDefinitionV1` instance、Target／Locale Profile Registry absent |
| Generic Core holdout | 全Genre／optional Feature Packをuninstallし、RPG、Core、Shooter stressを相互代用しないProduct C1 conjunctionをOwner反映済み | genre-neutral Project／Runtime／Package、Pack removal、Qualification Receipt absent |
| RPG gameplay | Command Battle、Actor Progression、Inventory／Equipment、Dialogue／Quest、Currency／Shopのexact五family | concrete Feature contract、MCD、Operation、Registry、Runtime、Qualification absent |
| Traversal／flow | 2D World／Camera／Collision／Navigation／Interaction／Scenarioでfinite Title→Result path | RPG Project、World／Scenario Source、cooked artifact、Runtime package absent |
| Presentation／I/O | Sprite／Tile、UI／Text、`en-US`／`ja-JP`、Accessibility、Audio、keyboard／controller remap | Asset、locale catalog、Input profile、adapter runtime、device Receipt absent |
| Persistence | checkpoint＋manual Save／Load、corrupt／wrong-version fail-closed、cross-feature atomicity | Save Schema、bundle、migration executable、positive／negative Receipt absent |
| Authoring continuity | GUI／公開C++／structured data、manualと`ai_composed_game`が同じProject stateへ収束 | Editor、SDK、Gateway、AI Operation、Project revision／Receipt absent |
| Test／playtest | automated project test、debug／performance capture、Human Gameplay Approval、same-candidate regression | runner、Fixture、measurement、Evaluation／Approval／Evidence absent |
| Build／distribution | clean Build／Cook／Package、signed internal MSIX、GameInput／VCLibs prerequisite、clean install／offline launch／uninstall | Build system、Toolchain lock、package、signing、clean-machine Receipt absent |
| Legal／release support | original content、license／NOTICE／privacy／support boundaryをsame candidateへ束縛 | content、SBOM／NOTICE、Legal Decision、Lifecycle Acceptance、Release authority absent |

Architecture上の意味不足として閉じたのは`ARCH-C149`のscope projectionだけである。上表のcurrent evidence不足は文書追記では解消せず、実在しないartifactを作成済み、qualifiedまたはavailableと表現しない。

### 16.3 Explicit non-requirements

job／class、party recruitment／reorder、branching ending、crafting、random loot、procedural dungeon、open world、complex economy、action battle、3D、advanced VFX、voice／cinematic、unrestricted scripting、runtime generation、Online／Account／Cloud／広告／課金、bulk Asset generation、Android／Appleその他Targetはinitial First Playableの不足機能ではない。Futureで採択されるまでnon-goalまたはplanning-onlyであり、これらの追加はrequired closure欠落の補修にならない。

### 16.4 Official／primary-source boundary

2026-08-04に次の一次資料を再確認した。外部資料はPlatform／accessibility／format／一般的なbuild・package surfaceの事実とgap discoveryにだけ使い、compact RPGの内容、五Feature family、Windows-only C1、MSIX選択またはclean-breakを外部組織の公式推奨とは表現しない。

- [Microsoft GameInput overview](https://learn.microsoft.com/en-us/gaming/gdk/docs/features/common/input/overviews/input-overview)：new codeでGameInputを使用し、PCではMicrosoft.GameInput NuGet packageを使う公式guidance。
- [Microsoft Windows package and deployment overview](https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/)／[distribution path guidance](https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/choose-distribution-path)：MSIX、install／update／servicing、distribution routeの公式区分。
- [Microsoft Windows accessibility overview](https://learn.microsoft.com/en-us/windows/apps/design/accessibility/accessibility-overview)／[WCAG 2.2](https://www.w3.org/TR/WCAG22/)：keyboard、focus、screen reader、text scaling、contrast等の一次要件／guidance。
- [RFC 5646／BCP 47](https://www.rfc-editor.org/rfc/rfc5646.html)：language tagのcanonical external basis。
- [CMake 4.4 Presets](https://cmake.org/cmake/help/v4.4/manual/cmake-presets.7.html)：project-wide presetとlocal user presetの分離。Toolchain baselineのcurrent pinはToolchain Ownerだけが所有し、本follow-upでは変更しない。
- [Unreal Engine packaging](https://dev.epicgames.com/documentation/en-us/unreal-engine/packaging-your-project)／[Godot project export](https://docs.godotengine.org/en/stable/tutorials/export/exporting_projects.html)：build／cook／package／playable exportのgap discovery。Miraikanaiの型、workflow、UI、defaultまたは互換性の正本にはしない。

<a id="c2-mobile-future-follow-up"></a>

## 17. C2／Mobile／Future post-C1 follow-up

| Field | Value |
|---|---|
| Date／closure ID | 2026-08-04／`ARCH-C150` |
| Scope | C1 closure後の別単位として、C2 Product boundary、Mobile tier／deny-only／distribution、Future membership／Target closureを監査する |
| Canonical closure | [Product Plan §5.3](../00-product/product-plan.md#53-c2-production-exact-product-boundary)がC2 selection、[§8](../00-product/product-plan.md#8-future-portfolio)がFuture inventory、LifecycleがEvidence aggregation、Mobile Common／Android／Apple／WindowsがPlatform境界、Native Game Module／Project ShaderがProject Source Target qualificationを所有する |
| Clean-break result | Mobile source qualificationをC2へ一意化し、旧Future ID、alias、supersession row、dual binding、migrationまたはhidden optional branchを追加していない |
| Planning boundary | Product Phase、Work Package、実装Task、順序、工程、工数、担当、execution Registry／Fixture／Gateを追加していない。既存Execution Proposalのstale canonical claimだけを非規範へ訂正した |

### 17.1 C2 Product boundary

| Axis | Exact target design | Current evidence |
|---|---|---|
| Host | `{target.headless.host@1, target.windows.editor@1}` | Target Profile record／Registry、Host distribution、Editor／SDK binary absent |
| Runtime Target | `{target.windows.desktop@1, target.android.mobile@1, target.apple.mobile@1}` | Target Profile record、runtime、Platform Adapter、device qualification absent |
| Reference | 各Targetの`two_d`＋`three_d`、same release／candidate、non-substituting | 六つのTarget×dimension pairに対応するReference Project、Asset、Package、Receipt absent |
| Locale | `{en-US, ja-JP}` | Locale Profile record／Registry、Catalog、font／layout／semantic invariance Evidence absent |
| Project Source | structured data、bounded Project C++、Project Shaderを全required Targetで個別Qualification | compiler output、static link、SPIR-V／Metal library、Project Shader／Native Receipt absent |
| Windows publication | Microsoft Store＋`package-profile.windows.msix` | Store identity、submission、certification、public read-back、update／withdrawal Receipt absent |
| Android publication | Google Play＋`package-profile.android.play` AAB | signed AAB、Play track／review／public read-back、device／16 KiB／coverage Receipt absent |
| Apple publication | App Store＋`package-profile.apple.bundle` | signed archive、TestFlight／review／public read-back、iPhone／iPad Receipt absent |
| Runtime content | offline completionに必要な全2D／3D contentをinstall-timeで完結 | Target別install payload、delivery manifest、offline completion Evidence absent |
| Product closure | Privacy、Legal／IP、License、SBOM／NOTICE、Documentation、support、update／repair／withdrawalをsame scopeへ束縛 | Active Product Definition、Lifecycle／Security／Privacy／Legal Acceptance、Release／Publication／Completion Decision absent |

Architecture targetとしてC2で作れるGameのclassは、Windows desktop、Android phone／tablet／foldable、Apple iPhone／iPadでoffline完結する2D Gameと3D Gameである。ただしcurrent Repositoryで実際に作成、起動、packageまたはStore公開できるGameは引き続き0件である。C1 First PlayableのArchitecture boundary、C2 target designまたはPlatform公式資料をEngine implementation、Reference content、Qualification、Publicationまたは商用品質のEvidenceへ数えない。

### 17.2 Mobile correction

Product C1はWindows-onlyで、Mobileが最初にProduct claimへ参加するtierはC2である。Mobile Commonの旧`C1／C2 Mobile`、`2D C1`、`3D C1 scalable subset`はProduct tierとdomain maturityを混同したため除去した。C2の2D／3DはBaseline presentationでcomplete Gameplay meaning、Save、Input、UI、Audio、offline completionを維持し、Target差はOwner-declared meaning-preserving fallbackだけに限定する。

`ProjectMobileSpecV1`は`runtime_generation_policy=deny_all`、`content_safety_profile_ref=null`をexact current V1とする。Content Safety Profileを存在しない必須Ref、未設定、default allowまたはhidden future branchにしない。positive structured-data generationは`future.capability.runtime-structured-data-generation`を採択する新Product Definition／Mobile spec revision、Authority／Threat Model、Content Safety Profile、Target qualificationが揃う場合だけ別subjectとして扱う。

### 17.3 Future inventory correction

| Classification | Exact count／members | State |
|---|---|---|
| Future identity | Product Plan §8の30 unique ID | 全件`planning_only`／`not_activated` |
| `single_target` | 24件（direct candidate 23＋decomposition-required 1） | direct candidateはpromotion時にexactly one Targetと再帰direct prerequisiteをsame-target closure。unrestricted scripting rowは分解候補kindの上限だけを保持 |
| `target_role_bundle` | persistence、small co-op、rollback、large-session、MMO、managed external Hostの6件 | 親Futureごとにexactly one bundle profile、role cardinality／relation／prerequisite mappingを要求 |
| Direct promotion eligibility | 29件direct candidate＋1件`decomposition_required` | unrestricted scripting umbrellaはstructured content MOD／sandboxed executable MOD／signed AOT desktop native extension／developer-only executable codeへclean replacementするまで直接昇格禁止 |
| Owner routing | local inference／managed Host=`mirakan.arch.ai-production-orchestration`、persistence operations／Runtime structured generation=`mirakan.arch.product-plan` | AI SecurityはAuthorization／Trust／Credential、Toolchainはartifactを所有。後二件は専用domain Owner未採択のcomposite planning identityで、promotion revisionが専用Ownerを採択するまでdomain contract absent |
| Removed from Future | `future.capability.mobile-project-native-shader-source-qualification` | Product C2へ再分類。unknown IDとして拒否し、alias／migrationなし |
| Rejected umbrella IDs | terrain＋foliage＋GI複合、dedicated＋transport＋replication複合、GI＋reflections中間ID | current Ref、fallback、alias、promotion mappingとして不受理 |

collaborative multi-user authoring、UGC、public Editor extension ecosystem、video／media、virtual production、recording／timecode／genlock、platform account／identity／achievement／leaderboard、cloud save、push、camera／microphone／location、background gameplay、Account、Cloud、commerce／in-app purchase、advertisingはcurrent 30件に含まれず、名称やPlatform APIだけでFutureとして採択済みとしない。platform account連携のcloud save／汎用Cloud serviceと、Product-level `persistence-live-service-moderation-operations` compositionを同一視せず、後者にもOnline Service／Provider／domain contractはまだない。Owner、public boundary、Target closure、Security／Privacy／Legal、offline／failure、support、claimを持つProduct revisionまではunadopted candidateまたはnon-goalである。

### 17.4 Official／primary-source boundary

2026-08-04に次の一次資料を再確認した。外部資料はPlatform factの根拠であり、C2の三Target、locale、Store組合せ、Reference dimension、AVP baseline、minimum OS、install-time contentまたはFuture選択を外部組織の公式推奨とは表現しない。

- [Microsoft distribution path guidance](https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/choose-distribution-path)：多くのDeveloperにMicrosoft Storeを推奨し、MSIX submissionのStore signing／hosting／updateを説明する。
- [Google Play target API requirement](https://developer.android.com/google/play/requirements/target-sdk)：2026-08-31から新規app／updateへAndroid 16／API 36以上を要求する。
- [Android 16 KiB page support](https://developer.android.com/guide/practices/page-sizes)／[adaptive design](https://developer.android.com/develop/adaptive-apps/guides/adaptive-dos-and-donts)／[Android Vulkan Profiles](https://developer.android.com/ndk/guides/graphics/android-vulkan-profile)／[texture compression targeting](https://developer.android.com/guide/playcore/asset-delivery/texture-compression)：native alignment、adaptive window、Profile選択、ASTC／ETC2 deliveryの公式境界。
- [Apple App Store submission requirements](https://developer.apple.com/app-store/submitting/)／[Xcode SDK／system requirements](https://developer.apple.com/xcode/system-requirements)：current upload SDK要件とXcode 26.6のSDK／deployment support範囲。
- [Apple-hosted asset packs](https://developer.apple.com/help/app-store-connect/manage-asset-packs/overview-of-apple-hosted-asset-packs/)：iOS／iPadOS 26以上のManaged Background AssetsとTestFlight／App Store route。

29件の`direct candidate`はFuture IDの事前分解を要求しないというPortfolio構造分類であり、promotion-readyではない。全件が新しいActive Product Definition revision、Owner contract、Target closure、Security／Privacy／Legal、support、fallback、same-scope Evidenceを必要とし、persistence operationsとRuntime structured generationは専用domain Owner採択も欠く。

`ARCH-C150`が閉じたのはC2／Mobile／Futureのmeaning、Owner、exact Product selectionとclassificationである。上表のmaterialization不足は文書追記では解消せず、実在しないProduct Definition、Target／Locale Profile、Project、Package、Registry、Fixture、Receipt、Qualification、Activation、PublicationまたはCompletionを作成済みと表現しない。
