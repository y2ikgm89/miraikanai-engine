# Miraikanai Engine 設計文書Index

- 最終更新日: 2026-07-21
- 状態: Subsystem別正式仕様を統合した設計レビュー用Index。2026-07-21 Engine不変境界・初心者承認の内部整合レビュー済み、ユーザー最終レビュー待ち

## 0. 統合計画サマリー

Miraikanai Engineは、AIがEngine内部objectを直接操作するゲームエンジンではない。人間またはAIが「作りたいゲーム状態」と「編集Command」を型付き構造化データとして提案し、C++ `AuthoringCommandGateway`がSchema、semantic、Capability、Policy、budget、base revisionを検証した後だけ、一つの`ProjectRevision`として原子的にCommitする独自AIネイティブゲームエンジンである。

| 主題 | 確定した計画 |
|---|---|
| Product形態 | Editor制作型を先に完成させる。制限付きRuntime生成はPhase 9で別Threat ModelとServer-authoritative Gatewayを設けた後だけ扱う |
| 制作レベル | Level 0自然言語、Level 1 Scene／Inspector／Graph、Level 2 GameplayDefinition、Level 3 Project C++を同一Project形式で往復できる |
| 初回制作 | 大まかなPromptをRequirementへ分解し、Blocking／High Impact不足だけを質問し、Game Brief承認後に薄いゲーム全体と深い代表部分を作る |
| Game実装 | 実行CodeはC++23、頻繁に調整する内容は型付き`GameplayDefinition`。offline CookしたPackageをC++ evaluatorが実行する |
| Script | Luau、Lua、C# Game runtime、汎用Script VM、bytecode interpreter、JIT、Game向けFFIは採用しない |
| 実装方式選択 | AIはSystem単位でGameplayDefinition、C++、または型付きCapability境界での併用を選ぶ。選択理由、budget、benchmarkをDecision Ledgerへ記録する |
| Engine不変境界／初心者承認 | Game制作では署名済みEngine baselineをread-only固定し、AIはProject構造化data、Asset、Test、公開SDK内のbounded C++、Engine-owned binding内のbounded Shaderだけを変更する。AIは自己承認せず、Engine-owned G0～G7とPolicy Serviceの署名済みAttestationが技術適格化し、人間はGame意図・Capability・試遊・最終Activationを承認する |
| Game System／コード生成 | 契約固定・実装開放型を公式方式とする。Engine標準、Project定義、署名済みBaseline内のread-only Extensionを同じ`GameSystemSpecV1`、State owner、Command／Event／Snapshot、System Catalog、Bundle、Testで扱う。Game制作AIはExtensionを生成・変更せず、`.cpp`だけを単独生成しない |
| Shooter Gameplay | `mirakan.feature.shooter_core.c1`をWeapon、Fire／Shot Delivery、Projectile、Combat、Vital、Pickup、Score、Encounter、Game Flowへ分割し、2D top-downと3D TPSは同じPublic ContractへProfileを適用する。Fireは原子的transaction、Projectileはauthoritative object、ParticleはPresentation専用とする |
| Engine／Editor | Engine、EditorHost、GameHost、Tool、NativeGameModuleのFirst-party CPU codeはC++23。EditorはC++ Projection Editorで、状態の正本ではない |
| C++移行 | CX0のHeader bootstrapから、Named Modules＋`import std`を使うCX3へ一方向移行する。C++26はShippingに使わずreadiness CIだけを持つ |
| Build | CMakeがFirst-party C++ Build定義の正本、Ninjaが高速なC++実行器、独自Build GatewayがEditor／AI／CIの唯一の入口。`build.ninja`は生成物であり公開APIにしない |
| Platform Build | WindowsはNinja Multi-Config、AndroidはGradle→CMake→Single-Config Ninja、Appleはportable C++ archiveをNinja、App shell／resource／最終link／archive／署名をXcodeが所有する |
| 実装・最適化 | 契約→最小縦切り→Target実測→上位ボトルネック最適化→回帰Gate→Capability昇格を全Phaseで反復する。最終一括最適化、平均fpsだけの合格、未計測の高機能並列実装を行わない |
| AI接続 | 内蔵AIはProvider API、外部Codex／Claude等はlocal MCP、Host Pluginは任意の補助UX。どの接続も検証・Commit権限を持たない |
| Debugging／AI Diagnosis | Project revision、Build、Play session、tick／phase／frame、System／Entity／Asset、Command／Event／StateをStable ID付きEvidenceとして記録する。Console、Profiler、Breakpoint／Watch、Replay／Rewind、Crash、remote device、人間とAIの原因分析を同じbounded Queryへ収束させ、AI修正は通常ChangeSet／Approval／回帰Gateへ戻す |
| Editor UI／UX | C++23の独自`MirakanUi Core`＋`MirakanEditor Shell`。Retained UIと限定typed Immediate Canvasを併用し、Scene／Canvas、World Outline、Topology Graph、Level Form、Streaming Inspector、Hierarchy／Outliner、Inspector、Asset、Source、Console、AI Partnerをdock、resize、floating、入替、multi-monitor、複数Workspace保存できる。mouse、keyboard、UI Automation、AIは`AuthoringSelectionContextV1`とStableIdを共有する |
| Game UI／AI UI | HUD／画面UIは型付き`UiDocument`を正本とし、手動操作とAI提案を同じChangeSetへ収束させる。C1はBuiltin＋`UiCompositeDefinition`、C2はoffline compileする`UiEffectGraph`＋A1／R3承認済み`UiNativeWidget`。生成画像はStaging、来歴、license、安全性、import／cook、Preview、承認を経由する |
| 初心者UX | `AI Creator` Workspaceを同じEditor内に用意し、AI Partnerを常設可能にする。Production／Debug／Art／Level Design等のWorkspaceへ切替可能 |
| Graphics | Windows D3D12、Android Vulkan、Apple Metal。Portable Rasterを基準に、GPU indirect／HZB／HLOD、Hybrid Deferred、Temporal Reconstruction、DLSS／XeSS／FSR／MetalFX、Frame Generation、RT Shadow／Reflection、RTGI、Path Tracing、Neural RenderingをEngine-owned契約＋交換可能Adapter＋個別Qualificationで段階導入する |
| Anti-alias／Upscale | 人間とAIは`balanced`、低cost、低blur、低ghosting、temporal安定、pixel-crisp、VR低latency等の`AntiAliasingIntentV1`を編集し、EngineがViewFamily単位`ResolvedAntiAliasingPlanV1`へ決定的に解決する。C1はFXAA、Mirakan TAA、Forward+ MSAA 2x／4x、C2はSMAA 1x、MSAA 8x、TAAU／Qualified Provider。MSAA×temporal、Hybrid Deferred MSAA、pixel-locked UI適用を禁止し、Target別visual／cost／thermal Receiptで昇格する |
| LOD | AIと人間が共通`LodIntentV1`とDomain別Policyを編集し、Runtime CompilerがMesh／Sprite、Representation／HLOD、Simulation、Animation、Material、VFX、Terrain／Foliage／Water／Snow Surface、Geometry residencyのTarget別`LodResolutionPlanV1`を生成する。PresentationからGameplayへ逆入力せず、screen-space metric、hysteresis、fallback、実測Receiptを必須にする |
| World／Level／Map | `Map`を単一Systemにせず、World structure、playable Level、Streaming、Procedural layout、Navigation、Map presentationへ解決する。Sceneは編集Shardと永続化owner、LevelはGameplay単位とScene membership、CellはTarget別Derived residency単位とし、三関係を混同しない。AIはbounded `WorldAuthoringContextV1`から型付きDomain OperationとWorld Bundleだけを提案し、Source IntentとCook済みPlanを分離する |
| Camera | Engine-owned Profile、Rig Graph、Director、Presentation、Sequence、Recording、Device Gatewayを共通契約で接続する。C1は決定論的Gameplay Camera、C2はCinematic／Split View、C3はMulti-camera Recording／Virtual Productionとし、AI Semantic、Timecode／Genlock、Calibration、Device安全境界を専用規約で所有する |
| Light／Shadow | 個別光源は物理単位付き`LightSourceV1`、意味要求は`LightIntentV1`、Target別結果は`ResolvedLightPlanV1`へ分離する。AIはRole、Mood、Coverage、重要度を編集し、ResolverがSource、2D selection／3D cluster、budget、fallbackへ決定的に解決する。ShadowはL0 Intent～L3承認済みTechniqueを別責務として段階導入する |
| Post Process | `PostProcessIntentV1`、`PostProcessProfileV1`、`PostProcessNodeCatalogV1`、`ResolvedPostProcessPlanV1`を正本にし、固定stage、AA互換、色空間、Volume、history reset、UI／pixel-locked分離をEngineが検証する。AIは見た目Goalを編集し、任意Pass順、native resource、history内部値を操作しない |
| Material／Visual Style | 型付きMaterial IR、複数Shading Model、version付きVisualStyleProfileを正本とする。人間とAIはIntent→Template→Instance→Graph→承認済みProject HLSLの順で最小権限を選び、Semantic Catalog、Preview、Explain、Validator、Target別offline compile、Material専用Evalを必須にする |
| 表現 | 2D、3D、`realistic_basic`、Realistic advanced、Toon、独自`pixel_diorama`を段階実装し、AI Visual Style ResolverがCapabilityとbudget内で選択する |
| Particle／VFX | 単一の型付きVFX Asset／Graph IRから2D／3D・CPU／GPU専用Artifactをoffline生成する。初心者Stack、上級者Graph、AIは同じSourceを編集し、VFX結果をGameplayへ逆入力しない |
| Water | C1 bounded surfaceからC2 lake／river／oceanへ段階導入し、Surface／Wave／Flow／Depth／UnderwaterとCPU Queryを専用Platformが所有する。GPU水面を浮力へ逆入力しない |
| Weather／Snow | Weather Snapshot、降雪VFX、Snow Surfaceを分離する。C1静的mask、C2 dynamic fieldを提供し、摩擦等はauthoritative Gameplay Surface Stateが所有する |
| Environment | Sky／Atmosphere／Fog／Cloud／IBLを`EnvironmentProfileV1`へ統合する。AIは意味IntentとPresetを提案し、Engineが物理値、Capability、Target、budget、lockを検証してDerived ArtifactへCookする |
| 必須Subsystem | Renderer、Camera、Asset、Collision、Physics、Navigation、Animation、Input、UI／Text／Localization／Accessibility、Audio、Particle／VFX、Lighting、Post Process、Sky／Atmosphere／Fog／Cloudを正式仕様で所有する |
| 外部Library | Engineの正規data model、Capability、validation、lifecycle、serialization、Editor UXは独自所有する。OS、Graphics API、Box2D、Jolt、Recast、GPU allocator、DirectSR、DLSS／Streamline、XeSS、FSR、MetalFX等はversion／hash／署名／license／実機Gate後にAdapter内へ隔離し、再発明しない |
| Physics Platform | 独自World／Body／Joint／Character／Command／Save／AI契約をC++23で所有し、2DはBox2D 3.1.1、3DはJolt 5.6.0をprivate kernel候補としてQualification後にProduction昇格する。自然言語Intentは専用Semantic Catalogから型付きResolutionへ解決し、Game／AIへVendor APIを公開しない |
| Navigation Platform | 2D GridはEngine-owned、3Dは独自契約＋交換可能Backendとし、Recast／Detour 1.6.0をC1 private基準Backendにする。Profile、Artifact envelope、status、version／lease、AI／Editorを独自所有し、Vendor ref／binaryを公開しない |
| Memory／Pointer | RAII、明示所有権、generation handle、phase／epoch lease、memory domain、GPU deferred destruction、Target別budgetを正規規約にする |
| ID／参照 | SourceはUUIDv7 `StableId`、契約はexact `McdContractRefV1`、Derived ArtifactはSHA-256 `ArtifactRefV1`、Domain-local numeric IDはowner scope付き、Runtime handleはgeneration付き非永続へ分離する |
| Math／Core Utilities | `mirakan::foundation`と`mirakan::math`を分離し、内部storage mathと公開semantic typeの二層にする。位置／方向／単位／座標空間／正規化をMCDへ持たせ、portable scalar referenceを正解系、SIMD／DirectXMath等を実測後のprivate backend候補にする |
| 大量制作／性能 | 大量配置、敵味方spawn、同時VFXを固定個数だけで制作拒否しない。AIがScale intentをFull Entity、simulation LOD、pool、instance、HLOD、streaming、CPU／GPU VFXへTarget別Cookし、Gameplayを黙って削らず統合負荷fixtureで実測する |
| 対象Platform | Windows Editor／Gameを先行し、Android、iOS／iPadOSを順に追加する。Mobile Editor、Linux製品Target、multiplayer、XRは初期対象外 |
| MVP | MVP-AはAIで作る2D top-down shooter、MVP-Bは同じShooter Coreを使う3D compact third-person shooter arena。Android／Apple vertical sliceはその後に行う |
| Capability Portfolio | MVP製品化と将来の汎用Engine化を`CapabilityPortfolioV1`で分離し、tierとactivation state、Owner、依存、Entry Gate、Evidenceを固定する。未Qualified CapabilityをAI／Editor／Buildが利用可能と表示しない |
| AIによるEngine保守 | Game制作とは別Repository／別Profile／別AuthorizationのEngine製品開発だけを対象にする。Game制作TaskへEngine source、A2／R4 Tool、Path、Approvalを公開または継承せず、新Engineは別の署名済みBaselineとして明示Migrationする |

### 0.1 要求から正式仕様への対応

| これまでに確認した要求 | 決定権を持つ正式仕様 |
|---|---|
| AI設計図、追加質問、共同／詳細／高水準設計、Editor制作型 | AIネイティブ設計計画書 |
| C++のみの実行層、構造化ゲームデータ、Luau不採用 | C++実行コード・構造化ゲームデータ規約 |
| Memory、Pointer、Architecture、Directory、Dependency、Build | 基盤アーキテクチャ規約 |
| Engine C++、C ABI、Module、CMake、Schema、Diagnostic、Test、Generated code、File／Directoryの命名と技術識別子 | AI可読命名・技術識別子規約 |
| Game ProjectのDirectory、File、Asset、Authoring Document、Native Game Code、AI staging／promotionの命名と配置 | AI可読Game Project配置・命名規約 |
| C++23、C++26 readiness、Modules、`import std`、Ninja | C++23・Named Modules移行規約 |
| AI／手動編集の共通状態、Undo、Recovery、Transaction | Authoring Model／Project State規約 |
| AIが理解できる型、Operation、Capability、Schema、Codegen | 実行可能契約規約。Physics／Collisionの自然言語Intent、canonical role、質問、Assumption、未対応Capability説明はPhysics AI Semantic Capability Catalog規約 |
| Game制作AIにEngineを変更させない、初心者がC++を読まずに安全性を判定する、AIがGame全体をStaging生成してSystem／Feature／Game単位で承認する | 不変Engine境界・初心者向けAI技術承認規約を正本とし、Game System、NativeGameModule、AI実装・保守ガバナンス規約へ接続する |
| Engine／ゲーム制作Debug、Profiler、Breakpoint／Watch、Replay／Rewind、Crash、remote device、AI原因分析 | AI可読Debugging／Observability／Replay規約を正本とし、Runtime、Editor、Platform、各SubsystemのDebug Snapshotへ接続する |
| Game system、Level system、戦闘、Character、AI C++コード生成、実装自由度 | Game System／AI Code Generation規約を正本とし、C++／GameplayDefinition境界はC++実行コード規約、Native実装はNativeGameModule規約へ接続する |
| Single-player Shooter、Weapon、Projectile、Damage、Vital、Pickup、Score、Encounter、2D top-down／3D TPS Profile | AI可読Shooter Gameplay／Weapon／Projectile規約をShooter意味契約の正本とし、Game System、Runtime、Input、Collision、UI、VFX、Audio各正本へ接続する |
| World、Scene、Level、Map、Streaming、Procedural generation、minimap／world map | World／Level／Map／AI Authoring規約を正本とし、Navigation、LOD、Asset、Runtime規約へtyped contractで接続する |
| 中規模／大規模制作、Scale axis、Canonical Source、Derived Plan、大量Entity、Large World、Large Project Authoring | Scale-Resilient Canonical Architecture規約を横断不変条件の正本とし、具体的な型、phase、budget、backendは各Subsystem正本へ接続する |
| Offline Single-player MVP製品化、Capability不足一覧、将来の汎用Engine化、Extension／Collaboration／Terrain／Multiplayer／XR等の着手Gate | AI可読Capability Portfolio／MVP製品化・将来Roadmap規約 |
| Renderer／Asset／Editor／Input／UI／Audio／2D／3D | 各Subsystem正式仕様、独自Editor UI Framework規約、2D／3D機能計画。Game UIのAI生成、画像設定、Composite／Effect／Native WidgetはUI規約とNativeGameModule規約 |
| 2D／3D Camera、Rig、Director、Cinematic、Multi-camera Recording、Virtual Production、AI Camera編集 | Camera Platform／AI Authoring／Virtual Production規約を正本とし、Runtime、Rendering、Physics、Input、Authoring規約とtyped Portで接続する |
| Collision、Physics Engine、Joint、Character、Navmesh、Animation | Collision規約、独自Physics Platform規約、独自Navigation Platform規約、Physics／Navigation／Animation連携規約 |
| Light、Lighting Intent／Profile／Resolver、Light selection／cluster | Lighting／AI Authoring規約をLightの意味・制作・AI操作の正本、Rendering規約をGPU実行の正本、Environment規約をSun／Moon／Sky／IBLの正本、2D／3D機能計画をShadow成熟度の正本とする |
| HDR／SDR Post Process、Exposure、Tone Mapping、Color、Bloom、DOF、Motion Blur、SSAO／SSR、Volume、AI設定 | Post Process／AI Authoring規約を意味・Profile・Resolver・Node Catalogの正本、Rendering規約をRender Graph実行の正本とする |
| Shader、Material、Toon／Realistic／Pixel Diorama | Material／Visual Style／AI Authoring規約を意味・制作・AI操作の正本、Rendering規約をRuntime実行の正本、2D／3D機能計画を成熟度・Phase配置の正本とする |
| Anti-alias、FXAA、SMAA、MSAA、TAA／TAAU、AI自動選択 | 意味Goalと成熟度は2D／3D機能計画、実行Plan／Pass／排他／history／visual GateはRendering規約、MCD Operation／MCP projectionは実行可能契約規約、Android／Apple実機Gateはモバイル規約 |
| Mesh／Sprite LOD、HLOD、simulation／animation／material／VFX／surface LOD、AI最適化 | AI可読LOD規約を共通正本とし、Runtime、Rendering、Asset、Asset Import、Animation、VFX、Water、Weather／Snow、Domain Pack規約がDomain実行を所有する |
| Sky、Atmosphere、Fog、Cloud、Environment Lighting、AI環境設定 | Environment Platform規約、Rendering規約、2D／3D機能計画 |
| 2D／3D Particle、CPU／GPU VFX、VFX Graph、AI／Editor編集 | 独自Particle／VFX Platform規約と2D／3D機能計画 |
| 大量配置、大量spawn、敵味方同時VFX、AI自動最適化 | Runtime連携・寿命・性能規約、Rendering規約、Particle／VFX規約、2D／3D機能計画、Authoring Model規約 |
| 水面、川、湖、海、波、流れ、水深、水中、浮力Query | Water Surface Platform規約と2D／3D機能計画 |
| 雨、降雪、積雪、融雪、圧雪、足跡表示 | Weather／Snow Surface規約、Particle／VFX規約、2D／3D機能計画 |
| Android、iOS／iPadOS、Build、Store、実機budget | モバイルPlatform規約 |
| Codex／Claude API、MCP、CLI／Desktop、Plugin、Engine保守AI | AI実装・保守ガバナンス規約 |
| Test、Eval、Provenance、SBOM、最新情報更新 | AI検証・評価・来歴規約 |

### 0.2 2026-07-20内部整合レビュー

- `Miraikanai Engine`の名称へ統一され、旧綴りは公式Review setに残っていない。
- Engine所有の技術stemはAI可読命名・技術識別子規約で`mirakan`／`Mirakan`／`MIRAKAN`へ固定し、公式Review setの規範例をcutoverした。`mira`表記は明示されたMigration Table、negative example、historical fixture、外部引用の入力に限り、新規実装または互換aliasの根拠にしない。
- C++23、GameplayDefinition、Script VM不採用、Target、Build Driver、Editor、AI権限、Runtime phaseの主要判断に矛盾する公式経路は見つかっていない。
- 未記入placeholder、実装者任せの保留、暫定Defaultを正式要件として残していない。外部Artifact取得後に確定するhash等は、Owner、取得手順、失敗条件を持つPhase 0作業として区別している。
- NinjaはBuild architecture全体ではなくC++ Build executorであること、Editor統合はCMake File APIとEngine-owned Receiptを使うこと、Phase 0で増分正当性と性能を実測することを今回の見直しで明文化した。
- Editor UIはDear ImGui等の置換前提prototypeを持たず、独自`MirakanUi Core`、`MirakanEditor Shell`、DirectWrite／TSF／UIA／OLE Adapter、AI Semantic Interfaceを正式経路にした。
- Physicsは「全Solver自作」でも「Vendor API直接公開」でもなく、独自の型付きPlatform契約へBox2D／Joltを隔離する方式に確定した。World／Solver全値、Joint、Character Motor、AI、Save／Replay、Qualificationの決定権を専用仕様へ分離し、Navigation／Animation連携文書との重複を解消した。
- Physics／CollisionのAI理解は追加Promptへ依存させず、version付きSemantic Vocabulary、`PhysicsIntentResolutionV1`、Capability maturity、質問、Assumption、禁止近似、Diagnostic、Golden fixtureへ固定した。Unity／Unreal／Godot比較はcoverage Evidenceに限定し、MiraikanaiのAPI模倣には使用しない。
- Debuggingは自由文Log、Screen capture、Native pointerをAIへ渡す方式ではなく、version付きSession、Event、Counter、Snapshot、Causality、Replay、Finding、Reproduction Bundleへ固定した。UnityのProfiler／remote attach、UnrealのInsights／Visual Logger／Gameplay／Blueprint／Rewind Debugger、GodotのDebugger／remote inspectionから有効な原則だけを採用し、AIはread-only Evidenceから開始して修正を通常Gatewayへ戻す。
- Navigationは「Navmesh生成／探索まで全自作」でも「Recast API直接公開」でもなく、独自契約＋交換可能Backendに確定した。2D GridはEngine-owned、3D C1はRecast／Detour 1.6.0の標準32-bit refをprivate基準Backendとし、1,024 tile×4,096 polygonで最低10 salt bitを保証する。Profile、Artifact、status、AI／Editor、Qualificationの決定権を専用仕様へ分離した。
- Particle／VFXは4つの別Authoring Systemや万能Runtime Graphではなく、単一Source／型付きIRから2D／3D・CPU／GPU専用Artifactへspecializeする方式に確定した。C1 CPU、C2 GPU、Gameplay分離、SoA、counter RNG、Render Graph、AI／Editor、Budget、Qualificationの決定権を専用仕様へ分離した。
- Waterは一般Transparent Material、Particle、Physicsへ暗黙分散させず、独自Water SourceからRender／CPU Query／Volume ArtifactをCookする方式に確定した。C1 bounded surface、C2 Body／wave／flow／depth／underwater、C3 fluid researchを分離した。
- Weather／Snowは降雪Particle、地表積雪、Gameplay Surface Stateを分離し、version付きWeather SnapshotとPresentation Eventで接続する方式に確定した。C1 static mask、C2 paged dynamic field、C3 deformable snowを分離した。
- EnvironmentはSky、Atmosphere、Fog、Cloud、IBLを一つのSource Platformへ統合した。自然言語Intentをversion付きPresetとtyped Overrideへ決定論的に解決し、AIへLUT、froxel、history、GPU resourceを公開しない。Weather、Water、VFX、Visual Style、Gameplayとの所有境界、Preview Receipt、Risk、Budget、AI Evalを専用仕様へ分離した。
- 大量制作は通常objectを無制限に並べる方式でも、固定cap超過で制作意図を捨てる方式でもなく、Scale intent、Gameplay fidelity floor、Target別Representation Plan、`OptimizationRequired` Source状態、Project固有Integrated Scale Fixtureで管理する。Presentation-only最適化はAIが自動提案できるが、敵味方数、Damage、collision、goal、spawn timingの変更は人間承認を必須にした。
- LODは万能な距離値または共通`lod_index`へ統合せず、共通Intent、Domain別Policy、Target別Plan、Qualification Receiptへ分離した。Meshはscreen-space error、HLODはstatic decorativeだけ、Simulationはauthoritative relevanceだけを使い、Presentation visibilityをGameplay判断へ逆入力しない。
- Game Systemは固定Class hierarchyまたは標準SystemのWhitelistへ閉じず、契約固定・実装開放型に確定した。Project固有SystemもMCD、State owner、依存Graph、Save／Replay、Budget、Target、Testを満たせば第一級Catalog entryになり、AIはDefinition、C++、Asset、Migration、Testを一つのSystem Bundleとして提案する。
- Shooterは2D／3D別の重複実装や巨大Managerへ集約せず、共通Shooter Core＋Profileへ確定した。Weapon／Fire／Shot Delivery／Projectile／Damage／Vital／Pickup／Score／Encounterを型付きDefinitionと一意State ownerへ分割し、Fire／Pickup transaction、Causality、Save／Replay、AI Intent、Target別capacityを専用Fixtureで検証する。
- World／Level／Mapは巨大`MapManager`へ集約せず、World Document、Scene edit shard、Topology、Level Definition、Spatial Intent、Target別Streaming Plan、Procedural Definition、Navigation、Map Presentationへ分離した。自然言語の「マップ」は6分類へ解決し、曖昧または高影響なら質問して推測Commitしない。Unityの構造化Editor操作／Undo、Unreal EngineのWorld Partition／One File Per Actor／stable GUID検証、GodotのScene composition／永続化ownerから有効な原則だけを採用し、`AuthoringSelectionContextV1`、`WorldAuthoringContextV1`、Scene永続化owner、Level membership、Cell assignment、Domain Operation、cross-view／holdout Evalへ統合した。外部EngineのObject model、Scene format、API名は複製しない。
- CameraはRenderer Transformや任意Scriptへ分散させず、Profile、Rig、Director、Presentation、Sequence、Recording、Deviceを同じEngine-owned契約へ統合した。AIは自然言語Intentを型付きRig候補へ解決し、外部Device、Timecode／Genlock、Calibration、Recording操作を別Processと承認境界へ隔離する。
- RendererはPortable Rasterを最低保証に維持し、GPU-driven、Temporal Reconstruction、Frame Generation、RT／Path／NeuralをEngine-owned frame／ray／model契約へ統合した。Vendor SDKは一つのProviderがSR／FG／Latency ownershipを持ち、real frame性能、署名、license、visual、latency、fault、実機Gate合格前にProduction表示しない。
- LightingはLightの値を散在させず、物理単位付きSource、意味Intent、Style Profile、決定論Resolver、Target別Plan、immutable Snapshotへ分離した。AIはbounded OperationとPreview／Explainを使い、GPU binding、cluster buffer、native format、Shadow内部技法を直接編集しない。
- Post ProcessはEffect一覧や任意Graphではなく、Intent、Profile、Volume、閉じたNode Catalog、固定execution stage、AA／Layer互換、history reset、Target別Planへ分離した。AIは見た目Goalから提案し、UI／Text／pixel-locked layer、HDR色空間、BudgetをValidatorがfail-closedに保護する。
- Anti-aliasは単なる機能一覧から、AI可読Intent、ViewFamily単位Plan、方式互換表、sample／resolve、Pass順、history reset、layer分離、typed Diagnostic、visual／performance Receiptへ具体化した。Unreal Engine 5.8、Unity 6.3 URP、Godot stableの公式資料から方式・Scope・Renderer／Platform制約を確認し、外部Engine固有設定を模倣せずMiraikanaiのMCD OperationとTarget Qualificationへ統合した。
- Math／Core Utilitiesは、汎用`Vector3`と雑多な`utils`を公開する方式ではなく、内部storage typeと公開semantic typeを分離した。AI／Editor／Save境界は意味付き型とDomain Operationを使い、portable scalar referenceからTarget別backendまで同じMCDとconformance fixtureで検証する。
- IDは曖昧な「stable numeric ID」へ統合せず、Source UUID、MCD ContractRef、Artifact hash、owner-scoped Domain ID、Runtime handleへ分離した。Game SystemのMCD ID、World／LevelのUUID、Cook内`system_id`／Cell ID、UI／Input／Audio等のruntime compact IDの永続可否と比較範囲を明記した。
- 実装はUnreal型の機能別budget／fallback、Unity型のhot path限定最適化、Godot型のPortable／Mobile／Advanced経路分離から有効な原則を採用する。ただしAuthoring SourceをTarget別に分裂させず、同じMCD／Material／Style／LOD IntentからTarget別ArtifactをCookし、C1→C2→C3を個別Receiptで昇格する。
- 本レビューは文書内部の整合確認であり、Engine実装完了または技術成立の実測証明ではない。実装開始には本Review setのユーザー承認と、別のPhase 0実装計画書の承認が必要である。

### 0.3 見直し結論と未実証項目

設計の粒度は、Phase 0実装計画をfile、target、contract、fixture、testへ分解できる段階に達している。一方、次は設計上の未決定ではなく、実artifactで合否を確定する未実証項目である。

| 未実証項目 | 証明するPhase／Gate | 不合格時 |
|---|---|---|
| C++23 Modules／`import std`の全Target Shipping成立 | CX1 Probe、CX2 Candidate、CX3 Cutover Gate | Header bootstrapまたはCX2に留まり、Shippingへ昇格しない |
| Ninjaの増分正当性、no-op性能、memory、中断復旧 | Phase 0 `VerificationReceiptV1` gate `mirakan.build.ninja_adoption.v1` | Makeへfallbackせず、DAG／dependency／poolを修正して再測定 |
| AppleのNinja C++ archive＋Xcode App shell境界 | Phase 0 C ABI fixture、Mobile Phaseの実archive | Apple CX0または非Promotion Probeに留める |
| AuthoringCommandGatewayとContract compilerの安全性 | Phase 0／1 conformance、negative、crash recovery | Editor／AIのwrite機能を有効化しない |
| `GameSystemSpecV1`、Project-defined System、State owner／dependency graph、Definition／Native同値性、System Bundle二段階Activation | Phase 0 contract fixture、Phase 3 C1 System、Phase 4 AI生成、Target別Qualification | 該当SystemをCatalogのactive／Productionへ昇格しない |
| Shooter Core、Fire transaction、Projectile／hitscan、Damage／Vital／Pickup／Score／Encounter、2D／TPS Profile、AI Intent、Save／Replay | Phase 3 `2d_shooter_c1_v1`、Phase 4 Shooter Semantic Eval、Phase 6 `tps_shooter_c1_v1`、Target別Profile Scale Gate | 該当Shooter Capability／ProfileをProduction表示せず、AIに選択させない |
| World／Scene／Level境界、Scene永続化owner／Level membership／Cell assignment、cross-view selection、Topology、Cell activation、Target別Streaming Plan、Map intent resolver、Level生成／再編集 | Phase 0／1 `world_authoring_semantics_v1`、Phase 2 `world_authoring_cross_view_v1`、Phase 3 compact 2D、Phase 4 `world_authoring_intent_v1`、Phase 6 compact 3D、Phase 8 large-world個別Gate | 旧LevelとSource intentを維持し、Scene／Level／CellまたはStableIdを誤変更するWorld CapabilityをAI／Editor／Productionへ表示しない |
| 独自MirakanUi Core／MirakanEditor Shell、D3D12 UI pass、Docking、DirectWrite／TSF／UIA | Phase 2 dependency negative、UI Automation、DPI、IME、keyboard、screen reader、device-loss fixture | Technology Preview配布へ進めない |
| 共通Debug Session、bounded Query、safe Pause／Step、Causality、Replay／Rewind、AI Diagnosis、remote device、Shipping除外 | `DBG0`～`DBG5`、`debug_known_faults_v1`、`debugging_diagnosis`、Windows／Android／Apple実機Gate | 該当Debug Capabilityを昇格せず、AI原因断定／修正とremote attachを公開しない |
| Box2D 3.1.1／Jolt 5.6.0のTarget別Kernel Qualification、Adapter、Joint、Character、Save／Replay | Phase 0 source／build lock、Phase 3の2D C1、Phase 6の3D C1、Mobile Phase実機Gate | 当該Dimension／TargetのPhysics CapabilityをProduction表示しない |
| Grid2D／Recast・Detour 1.6.0 Backend、Artifact、32-bit ref capacity、query status、AI／Editor | Phase 0 MCD／Backend Port、Phase 3 Grid C1、Phase 6 3D Qualification／Reference Scene、Mobile Phase実機Gate | 当該Dimension／TargetのNavigation CapabilityをProduction表示しない |
| VFX Graph Compiler、2D／3D CPU specialization、D3D12／Vulkan／Metal GPU Artifact、indirect draw、device recovery | C0 Compiler fixture、Phase 3／6 CPU Reference Effect、Phase 8 GPU／Mobile実機Gate | 該当CPU／GPU VFX CapabilityをProduction表示しない |
| Water Body／Surface Compiler、CPU Query、D3D12／Vulkan／Metal Water pass、buoyancy境界 | C0 schema／flat fixture、Phase 6 C1 bounded water、C2 Water Qualification、Mobile実機Gate | 該当Water CapabilityをProduction表示しない |
| Weather Snapshot、降雪VFX、static／dynamic Snow Surface、Gameplay分離 | Phase 6 C1 snow fixture、C2 dynamic field qualification、Mobile thermal Gate | 該当Snow CapabilityをProduction表示しない |
| Environment Intent Resolver、Preset、Atmosphere LUT、Volumetric Fog／Cloud、AI Preview、Target fallback | C0 resolver／contract fixture、Phase 6 C1 Environment、Production Environment Qualification、Mobile thermal Gate | 該当Environment CapabilityをManifestへ掲載せず、AIに選択させない |
| 2D／3Dのframe、memory、visual、physics／navigation連携 | Phase 3／6 reference scene、10分soak、golden capture | 対応CapabilityをC1へ昇格しない |
| FXAA／SMAA／MSAA／Mirakan TAA／TAAUの画質、sample／resolve、history、AI Intent Resolver、D3D12／Vulkan／Metal互換性 | Renderer R0／R1／R3、4x linear SSAA reference、AA visual receipt、Mobile実機30分thermal／2時間endurance | 非Qualified方式をCatalogから隠してFXAAへ留め、FXAAも不成立ならTargetを未対応にする |
| Camera Profile／Rig／Director、AI Intent、Cinematic、Multi-camera Recording、Device／Clock／Calibrationの決定論、安全性、Recovery | Camera C0 contract fixture、Phase 3の2D C1、Phase 6の3D C1、Phase 8以後のC2／C3個別Gate、2時間Recording soak | 該当Camera Capabilityだけを非表示にし、C1 Failsafe RigまたはDeviceなしGameplayへ留める |
| Mesh／Sprite、HLOD、Simulation、Animation、Material、VFX、Surface、Residency LODの決定論、visual／Gameplay fidelity、CPU／GPU一致 | LOD C0 contract fixture、Phase 3の2D C1、Phase 6の3D C1、Phase 8のC2、Project固有Integrated Scale Fixture | 該当DomainをProduction表示せず、LOD0／Full Simulation／CPU direct等の意味同等fallbackへ留める |
| 大量配置＋spawn burst＋Physics／Nav／Animation＋敵味方VFXの同時成立 | Phase 3 `2d_crowded_battle_v1`、Phase 6 `3d_crowded_battle_v1`、Project固有Integrated Scale Fixture、Target実機soak | Sourceを維持して`OptimizationRequired`、Play／Package promotion停止。Gameplay変更とTarget除外は人間承認 |
| GPU indirect／HZB／HLOD／meshlet、DLSS／XeSS／FSR／MetalFX、Frame Generation、RTGI／Path／Neural | Renderer R2～R9、Vendor別hardware、visual／latency／signature／license／fault／bridge baseline | Portable Rasterへfallbackし、該当Provider／CapabilityをProduction表示しない |
| Android／AppleのStore、実機、thermal、package | Phase 7 physical device／package／privacy Gate | 対象Platformを配布Targetへ昇格しない |
| AI生成品質、安全性、Provider更新耐性 | MVP-A以降のTask Eval、holdout、adversarial、Promotion Gate | 対象Risk classまたはProviderを有効化しない |

したがって次の正規作業はEngine全体の一括実装ではなく、Phase 0だけを対象とする実装計画書の作成である。2D、3D、Mobile、Production機能は各Milestone Gateの入力が揃ってから別実装計画へ分解する。

## 1. 読む順序

Miraikanai Engineの公式Review setは次の47文書である。上位のProduct判断からGame制作の不変境界、共通契約、Game／World model、Subsystem固有契約、検証規約の順に読む。

1. [AIネイティブ独自ゲームエンジン 設計計画書](./2026-07-18-ai-native-game-engine-authoring-design.md)
2. [Miraikanai Engine 不変Engine境界・初心者向けAI技術承認規約](./2026-07-21-immutable-engine-beginner-ai-approval-design.md)
3. [Miraikanai Engine C++実行コード・構造化ゲームデータ規約](./2026-07-19-cpp-structured-game-data-design.md)
4. [Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)
5. [Miraikanai Engine AI可読命名・技術識別子規約](./2026-07-20-ai-readable-engine-naming-convention-design.md)
6. [Miraikanai Engine AI可読Game Project配置・命名規約](./2026-07-20-ai-readable-game-project-layout-naming-design.md)
7. [Miraikanai Engine AI可読Math／Core Utilitiesアーキテクチャ規約](./2026-07-20-ai-readable-math-core-utilities-architecture-design.md)
8. [Miraikanai Engine AI可読Memory／Pointerアーキテクチャ規約](./2026-07-20-ai-readable-memory-pointer-architecture-design.md)
9. [Miraikanai Engine C++23・Named Modules・`import std`移行規約](./2026-07-20-cpp23-modules-import-std-transition-design.md)
10. [Miraikanai Engine Authoring Model／Project State規約](./2026-07-19-authoring-model-project-state-design.md)
11. [Miraikanai Engine 実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)
12. [Miraikanai Engine Game System／AI Code Generationアーキテクチャ規約](./2026-07-20-game-system-ai-codegen-architecture-design.md)
13. [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
14. [Miraikanai Engine AI可読Debugging／Observability／Replayアーキテクチャ規約](./2026-07-20-ai-readable-debugging-observability-replay-architecture-design.md)
15. [Miraikanai Engine World／Level／Map／AI Authoringアーキテクチャ規約](./2026-07-20-world-level-map-ai-authoring-architecture-design.md)
16. [Miraikanai Engine 2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)
17. [Miraikanai Engine AI可読Capability Portfolio／MVP製品化・将来Roadmap規約](./2026-07-20-ai-readable-capability-portfolio-productization-roadmap-design.md)
18. [Miraikanai Engine Camera Platform／AI Authoring／Virtual Productionアーキテクチャ規約](./2026-07-20-camera-platform-ai-authoring-virtual-production-architecture-design.md)
19. [Miraikanai Engine NativeGameModuleアーキテクチャ規約](./2026-07-19-native-game-module-architecture-design.md)
20. [Miraikanai Engine Asset Pipeline／Content Package規約](./2026-07-19-asset-pipeline-content-packaging-design.md)
21. [Miraikanai Engine Asset Import／AI Authoring／Editor UXアーキテクチャ規約](./2026-07-20-asset-import-ai-authoring-editor-ux-design.md)
22. [Miraikanai Engine Rendering／Render Graphアーキテクチャ規約](./2026-07-19-rendering-render-graph-architecture-design.md)
23. [Miraikanai Engine Material／Visual Style／AI Authoringアーキテクチャ規約](./2026-07-20-material-visual-style-ai-authoring-architecture-design.md)
24. [Miraikanai Engine Lighting／AI Authoringアーキテクチャ規約](./2026-07-20-lighting-ai-authoring-architecture-design.md)
25. [Miraikanai Engine Post Process／AI Authoringアーキテクチャ規約](./2026-07-20-post-process-ai-authoring-architecture-design.md)
26. [Miraikanai Engine AI可読LODアーキテクチャ規約](./2026-07-20-ai-readable-lod-architecture-design.md)
27. [Miraikanai Engine Scale-Resilient Canonical Architecture規約](./2026-07-20-scale-resilient-canonical-architecture-design.md)
28. [Miraikanai Engine Environment Platform／AI Authoringアーキテクチャ規約](./2026-07-20-environment-platform-ai-authoring-architecture-design.md)
29. [Miraikanai Engine 独自Particle／VFX Platformアーキテクチャ規約](./2026-07-20-particle-vfx-architecture-design.md)
30. [Miraikanai Engine Water Surface Platformアーキテクチャ規約](./2026-07-20-water-surface-platform-architecture-design.md)
31. [Miraikanai Engine Weather／Snow Surfaceアーキテクチャ規約](./2026-07-20-weather-snow-surface-architecture-design.md)
32. [Miraikanai Engine Collision／Colliderアーキテクチャ規約](./2026-07-19-collision-collider-architecture-design.md)
33. [Miraikanai Engine 独自Physics Platform／Dynamicsアーキテクチャ規約](./2026-07-20-physics-engine-architecture-design.md)
34. [Miraikanai Engine Physics AI Semantic Capability Catalog規約](./2026-07-20-physics-ai-semantic-capability-catalog-design.md)
35. [Miraikanai Engine 独自Navigation Platformアーキテクチャ規約](./2026-07-20-navigation-platform-architecture-design.md)
36. [Miraikanai Engine Physics／Navigation／Animation連携規約](./2026-07-19-physics-navigation-animation-architecture-design.md)
37. [Miraikanai Engine Input／Action／Device規約](./2026-07-19-input-action-device-architecture-design.md)
38. [Miraikanai Engine UI／Text／Localization／Accessibility規約](./2026-07-19-ui-text-localization-accessibility-design.md)
39. [Miraikanai Engine 独自Editor UI Framework／Shellアーキテクチャ規約](./2026-07-20-editor-ui-framework-architecture-design.md)
40. [Miraikanai Engine Audio／Mixer／Spatial規約](./2026-07-19-audio-mixer-spatial-architecture-design.md)
41. [Miraikanai Engine Editor／Workspace／UX規約](./2026-07-19-editor-workspace-ux-design.md)
42. [Miraikanai Engine Windows Platform／Distribution規約](./2026-07-19-windows-platform-distribution-design.md)
43. [Miraikanai Engine モバイルPlatformアーキテクチャ規約](./2026-07-19-mobile-platform-architecture-design.md)
44. [Miraikanai Engine Domain Pack／将来Capability規約](./2026-07-19-domain-pack-future-capability-roadmap.md)
45. [Miraikanai Engine AI可読Shooter Gameplay／Weapon／Projectileアーキテクチャ規約](./2026-07-20-ai-readable-shooter-gameplay-architecture-design.md)
46. [Miraikanai Engine AI実装・保守ガバナンス規約](./2026-07-19-ai-engine-development-governance-design.md)
47. [Miraikanai Engine AI検証・評価・来歴規約](./2026-07-19-ai-verification-evaluation-provenance-design.md)

`2026-07-18-codex-config-optimization-design.md`は開発者個人のCodex設定資料であり、Engineの製品・Runtime・Editor・AI契約に対する決定権を持たず、公式Review setへ含めない。

## 2. 文書ごとの決定権

| 文書 | 決定する範囲 |
|---|---|
| AIネイティブ設計計画書 | Product vision、制作体験、AI／人間の関係、Phase、MVP、Review set全体の上位目的 |
| 不変Engine境界・初心者向けAI技術承認規約 | `ImmutableEngineBaselineV1`、Game制作Tool／Path境界、bounded Project C++／Shader、G0～G7、AIとPolicy Serviceの権限分離、System／Feature／Game Candidate承認、初心者UI、atomic Activation／Rollback |
| C++実行コード・構造化ゲームデータ規約 | C++／data境界、GameplayDefinition、NativeGameModule、AI実装選択、Script VM不採用 |
| 基盤アーキテクチャ規約 | C++、Memory、Pointer、Module、Dependency、`BuildDriverProfileV1`、Ninja／Gradle／Xcode、Directory |
| AI可読命名・技術識別子規約 | Product technical stem、Repository slug、C++、C ABI、Module、CMake、HLSL、Tool、Schema、Contract ID、Diagnostic、Test、Generated code、File／Directory、Naming Policy、lint、移行Gate |
| AI可読Game Project配置・命名規約 | Game Projectのroot、Authoring、Asset、Native code、Import、`.mirakan/` staging、AI提案、promotion、subject-first Asset名、Path衝突、legacy入力 |
| AI可読Math／Core Utilities規約 | Foundation／Math target、storage／semantic type、unit／space／normalization、matrix／Quaternion／Transform、floating-point、failure、AI projection、portable reference／backend Qualification |
| AI可読Memory／Pointer規約 | Pointer taxonomy、safe／unsafe境界、generation handle、lease、Memory Resource、AI contract、static rule、failure、qualification |
| C++23・Named Modules移行規約 | C++23 Profile、Named Module名、`import std`、CMake、Ninja Generator matrix、Make非対応、BMI、Header例外、AI依存表現、Apple Build分離、CX0→CX3 Cutover Gate |
| Authoring Model／Project State規約 | Authoring Document、ProjectRevision、ChangeSet、transaction、journal、recovery、唯一の状態変更経路 |
| 実行可能契約規約 | Requirement、Type、Operation、State、Capability、Schema projection、Codegen。Anti-alias、Lighting、Post Processのclosed契約と`operation.rendering.aa.*`／`operation.lighting.*`／`operation.post_process.*`、MCP／Provider projection |
| Game System／AI Code Generation規約 | `GameSystemSpecV1`、System Catalog、State owner、依存Graph、Implementation Plan、System Bundle、Definition／Native選択、Project-defined System |
| Runtime連携規約 | Tick、Writer、Command／Event／Snapshot、寿命、Asset version、Budget、Scale intent／Representation Plan、統合密度Gate、Failure |
| AI可読Debugging／Observability／Replay規約 | Debug Session、Event／Counter／Snapshot、bounded Query、Causality、Breakpoint／Watch／Step、Replay／Rewind、Crash／Hang、remote device、AI Diagnosis、Reproduction、Debug Qualification |
| World／Level／Map／AI Authoring規約 | World／Scene／Level／Topology、Map intent、Spatial Partition、Streaming Plan、Procedural World、Map Presentation、Level transition |
| 2D／3D機能計画 | 2D／3D Capability範囲、成熟度、Visual Styleの製品到達点とPhase配置、各Subsystemの製品上の到達点。Anti-aliasの人間／AI向け意味Goal、Intent、Light／Post／ShadowのC1／C2製品範囲 |
| AI可読Capability Portfolio／MVP製品化・将来Roadmap規約 | `CapabilityPortfolioV1`、MVP release readiness、capability tier／activation state、不足Capability、Product horizon、Extension／Collaboration／Timeline／Terrain／Multiplayer／Platform ExpansionのEntry Gate |
| Camera Platform／AI Authoring／Virtual Production規約 | Camera Profile／Rig／Director／Presentation、AI Semantic／Operation、Runtime、Sequence、Recording、Device Gateway、Timecode／Genlock、Calibration、Security、Budget、Qualification |
| NativeGameModule規約 | 公開ABI、Target別link方式、Build／Promotion、GameHost restart、信頼境界 |
| Asset Pipeline／Content Package規約 | Source／Import／Derived／Package、Importer隔離、Catalog／VFS、Cook、Patch／DLC、AI Asset来歴 |
| Asset Import／AI Authoring／Editor UX規約 | Import Profile、Source解析、Texture／3D／Animation／Audio／Font IR、Conversion／Loss Report、Preview、Reimport Conflict、AI Operation、Asset Browser／Import Inspector |
| Rendering／Render Graph規約 | RenderSnapshot、extract、individual／instanced／spatial／presentation表現、Render Graph、resource／pass／access、Backend、GPU visibility、`ResolvedAntiAliasingPlanV1`、sample／resolve、方式排他、history、Temporal Reconstruction／Frame Generation Provider、RT／Path／Neural、同期、device loss、Renderer Qualification |
| Material／Visual Style／AI Authoring規約 | Material Semantic／Node Catalog、Definition／Instance／Template、Domain／Shading Model、VisualStyleProfile、AI Operation、Preview／Explain／Validator、offline compiler、budget、fallback、Material専用Eval |
| Lighting／AI Authoring規約 | `LightSourceV1`、`LightIntentV1`、`LightingStyleProfileV1`、Resolver、2D selection／3D cluster、Runtime Snapshot、AI Operation、Budget、Fallback、Lighting Qualification |
| Post Process／AI Authoring規約 | `PostProcessIntentV1`、Profile、Volume、Node Catalog、Resolver、Execution Stage、AA／UI／HDR互換、History、AI Operation、Budget、Visual Qualification |
| AI可読LOD規約 | LOD共通語彙、`LodIntentV1`、Domain別Policy envelope、metric、transition、AI Operation、Target別Plan、fallback、Qualification Receipt、Presentation／Gameplay分離 |
| Scale-Resilient Canonical Architecture規約 | Scale axis、`ProjectScaleEnvelopeV1`、Canonical Source→Derived Plan→Runtime State→Evidence、identity／Save／Replay不変条件、Large World／Massive Simulation／Large Project Authoring／Distributed AuthorityのActivation |
| Environment Platform／AI Authoring規約 | Sky／Atmosphere／Fog／Cloud／Environment LightingのSource、Intent、Preset、AI／Editor Operation、Validator、Preview、Compiler、Artifact、Budget、Diagnostic、Qualification |
| 独自Particle／VFX Platform規約 | VFX Asset／Emitter／Graph／IR、2D／3D・CPU／GPU specialization、Runtime、Renderer binding、AI／Editor、Budget、Diagnostic、Qualification |
| Water Surface Platform規約 | Water Body／Surface／Volume、Wave／Flow／Depth、CPU Query、Underwater、VFX／Physics境界、Budget、Qualification |
| Weather／Snow Surface規約 | Weather Snapshot、降雪VFX接続、static／dynamic積雪、融雪／圧雪／stamp、Gameplay分離、Budget、Qualification |
| Collision／Collider規約 | Body／Collider／Shape／Material／Filter、Query、Contact／Trigger、Cook、Editor／AI操作 |
| 独自Physics Platform／Dynamics規約 | Physics World、Solver Profile、Kernel Qualification、Dynamics、Joint／Constraint、Character Motor、Physics Save／Replay、Physics Editor／AI |
| Physics AI Semantic Capability Catalog規約 | Physics／Collision自然言語Intent、canonical role、`PhysicsIntentResolutionV1`、質問、Assumption、禁止近似、Capability比較、Semantic Eval |
| 独自Navigation Platform規約 | Grid2D、3D Navmesh、Backend lock／Port、Profile、Artifact、query status、capacity、AI／Editor、Qualification、完全自作研究Gate |
| Physics／Navigation／Animation連携規約 | Physics／Navigation／Animationのsnapshot／command境界、Animation Graph、root motion、固定phase連携 |
| Input／Action／Device規約 | Device sample、semantic action、InputSnapshot、remap、replay、text入力分離、haptics |
| UI／Text／Localization／Accessibility規約 | `UiDocument`、Retained UI、layout、event／focus、IME、shaping／raster、localization、Game accessibility、AI UI Authoring、`UiCompositeDefinition`／`UiEffectGraph`／`UiNativeWidget`契約 |
| 独自Editor UI Framework／Shell規約 | MirakanUi C++ target、Widget、Retained／Immediate境界、D3D12 UI pass、DirectWrite／TSF／UIA／OLE Adapter、Docking、AI Semantic Interface、禁止GUI dependency |
| Audio／Mixer／Spatial規約 | Audio cue／voice／bus、mixer、streaming、spatial audio、callback、device adapter |
| Editor／Workspace／UX規約 | Window／Panel／Dock、Scene／Outliner／Inspector、AI Partner、AI Creator、workspace、Editor accessibility |
| Windows Platform／Distribution規約 | Windows Target、MSIX／managed Directory distribution、signing、crash privacy、package layout |
| モバイル規約 | Android／iOS／iPadOS Target、Adapter、Lifecycle、Build／Signing／Upload分離、実機、Store。AAのtile memory／bandwidth、sample／store／resolve、thermal、Target別Qualification |
| Domain Pack／将来Capability規約 | Genre Pack境界、初期Pack、将来Capabilityの非対象範囲と設計着手Gate |
| AI可読Shooter Gameplay／Weapon／Projectile規約 | Shooter Core、Weapon／Fire／Shot Delivery／Projectile／Damage／Vital／Pickup／Score／Encounter、2D top-down／3D TPS Profile、AI Intent、Save／Replay、Qualification |
| AI実装・保守ガバナンス | AI Task、権限、Risk、質問、Source sandbox、API／MCP／CLI／Plugin |
| AI検証・評価・来歴規約 | Gate、Test、形式モデル、Eval、Receipt、SBOM、Provenance、最新情報 |

同じ主題で矛盾する場合は、その主題の決定権を持つ正式仕様を優先する。上位のProduct目標を下位文書が変更してはならず、下位の具体的な安全規約を上位文書の概略表現で緩和してはならない。複数Subsystemに跨る変更では、各所有文書を同じChangeSetで更新し、Runtime phase、Schema、Memory、Target、Testの整合を確認する。

## 3. 外部規格とプロジェクト規約

外部公式資料はAPI、Platform、標準、Libraryの事実と推奨を提供する。Miraikanai Engine固有のphase、Budget、Risk、Approval、Directory、選択閾値は、本Review setが一次資料を検討して確定したプロジェクト公式規約である。

文書内の「最新」はfloating versionを意味しない。調査基準日に確認したStable／LTS／approved artifactをexact versionとhashで固定し、更新はEvidence、Eval、ADR、Reviewを経由する。

## 4. 設計から実装計画へ進むGate

実装計画書の作成へ進む前に、次をすべて満たす。

- ユーザーが公式Review set 47文書の方向性をReviewし、修正点または承認を返す。
- 文書間Link、Heading anchor、Table、Code fence、規範語の機械検査が通る。
- C0／C1に必要な選択が暗黙Defaultになっておらず、外部artifactから取得する値には取得手順、Owner、Block条件がある。
- Authoring、Runtime、Renderer、Asset、Editor、Input、UI、Audio、Physics、Navigation、Animation、Platform、AIの境界にOwner、入力、出力、phase、budget、failure、testが定義されている。
- Phase 0から2D First Playableまでを、依存関係と検証可能な完了条件を持つ実装taskへ分解できる。

このGateの通過は、**実装計画書を作成してよいことだけ**を意味する。Engine本体の実装開始を承認するものではない。

## 5. 実装開始とPhase完了Gate

Engine本体の実装は、公式Review set承認後に別文書として実装計画書を作成し、その計画書をユーザーがReviewして承認した後に開始する。

- 実装計画書はtaskごとに対象file、public contract、依存task、test、performance／memory gate、rollback、完了条件を記載する。
- 各taskは基準fixture、Target Profile、測定metric、Before／After、fallback、生成するReceipt、昇格可能なCapability状態を記載し、instrumentationがない機能は先に実装しない。
- Phase 0で生成するMiraikanai Contract Definition、Toolchain lock、Contract compiler、generated code、Gate、Fixtureは、設計Reviewの事前成果物ではなくPhase 0の実装成果物である。
- Phase完了は、該当文書のDefinition of Done、target matrix、conformance／integration／soak／package testが実artifactで合格した時だけ宣言する。
- 2D First Playable、3D First Playable、Android／Apple vertical slice、Production Capabilityは、それぞれ独立したMilestone Gateとして順に判定する。
- 仕様と実装が不一致になった場合、無断で仕様を実装へ合わせず、Evidenceと影響範囲を伴うADR／仕様ChangeSetを先にReviewする。

### 5.1 Phase 0実装計画の設計

Phase 0は、全Subsystemの空Directoryや将来APIを先に作るRepository scaffoldではなく、公式Toolchainから最小Hostの起動・終了と検証Receiptまでを一方向に通す最小vertical sliceとする。実装計画は次の順序で分解し、各段階が独立した失敗条件、test、rollback、完了Evidenceを持つ。

1. CMake 4.4.0、Ninja 1.11以上、固定MSVC／Clang、vcpkg manifest baselineを取得し、hash、license、Target対応を`toolchain.lock.json`へ固定する。
2. Root `CMakeLists.txt`、schema 12の`CMakePresets.json`、Toolchain file、Compiler policy target、CTest入口を作り、source tree外のTarget別Build treeだけを許可する。
3. CX0 `cxx23_headers_bootstrap`をPhase 0の実装Profileとし、CX1 `cxx23_modules_probe`は限定fixtureの非Promotion CIへ隔離する。`import std`がCMakeのExperimental gateを必要とする間はCX2へ進まない。
4. 最小MCD、Math Type／Operation／Diagnostic、Contract Compiler、generated Header／C ABI／Receiptを一つのfixtureで接続し、入力hash、生成物hash、再生成、round-trip、negative caseを検証する。
5. `mirakan_foundation`、`mirakan_math`のportable scalar reference、最小Hostを作り、生成契約を介した起動、正常終了、失敗診断、semantic type検証、process exit code、共通trace context、typed metric、process／memory／queue計測をIntegration testで固定する。
6. Mathのunit／property／golden／cross-language／CPU・HLSL conformanceと、clean build、incremental rebuild、no-op build、並列Build、入力変更、中断復旧、ASan、warning-as-error、memory／時間budgetを決定論的fixtureで測定し、`VerificationReceiptV1`へ記録する。
7. Phase 0 Gateに失敗した場合はEngine機能を追加せず、失敗したToolchain、DAG、契約、Fixtureだけを修正して再測定する。Make、未固定Toolchain、Experimental artifact、空実装によるfallbackを作らない。

Phase 0では2D／3D Runtime、Renderer機能、Editor UI、Physics／Navigation Backend、Mobile package、Production Capabilityを実装しない。それらに対応するTop-level DirectoryとCMake targetは、最初の実装Taskが承認されるまで作成しない。Phase 0完了は、最小vertical sliceの実artifactと全Gate Receiptが揃った場合に限る。

### 5.2 Milestone共通の実装loop

Phase 1以降も、機能実装と最適化を別の後処理にしない。各Milestoneは、MCD／Budget固定、reference経路、決定論的fixture、Playable接続、Target trace、最大寄与ボトルネックの修正、Before／After、回帰／fault／soak、Receipt、昇格の順で閉じる。

実装順序とEntry／Promotion Gateの正本は[2D／3D機能計画13.1節](./2026-07-19-2d-3d-capability-plan.md#131-delivery-work-package)、計測手順と必須metricの正本は[Runtime性能規約14.1.2節](./2026-07-19-runtime-integration-lifetime-performance-design.md#1412-継続性能工学と測定loop)とする。

- C1はPortableな意味基準とfallbackを先に完成させる。
- C2は高機能を一Capabilityずつ追加し、無効化可能性と基準経路非退行を証明する。
- C3は別正式仕様、prototype、人間承認、個別Security／Performance Gateなしに着手しない。
- Gate不合格ではSource intentとlast valid playableを維持し、`OptimizationRequired`またはCapability非表示として後段へ進まない。
- Baseline緩和と実装最適化を同じChangeSetで暗黙処理しない。
