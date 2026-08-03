# Miraikanai Engine Architecture Index

## 1. 目的と非規範性

このIndexはArchitecture文書を発見し、読む順序と責務の入口を示すための手動管理navigationである。EngineのSchema、固定値、Gate、実装順序、Capability activationを定義しない。

2026-08-03時点で、Architecture InventoryのGenerator、Schema、生成ArtifactはRepositoryに存在しない。この一覧は現存するOwner文書64件を手作業で列挙したものであり、生成済みprojectionではない。件数、状態、path、依存の機械的な正しさを主張せず、変更時は実ファイルと各Headerを確認する。

Owner文書はすべて`review`であり、対応するEngine実装・Schema・Fixture・ReceiptはRepositoryに存在しない。したがって本文中の型、Registry、固定値、hash、Operationは、明示的な外部Artifactを除き設計候補である。状態と根拠の解釈は[Architecture Governance](01-governance/architecture-governance.md)を正本とする。

以下の「Owner文書」は、各概念のinitial V1 Architecture上の一意な所有先を示す。`review`のままではDefinition、実装、Qualification、CapabilityまたはProduct releaseの証拠にならない。Runtime ECSは[Runtime ECS](04-runtime/entity-component-system.md)、Gameplay authoringは[Gameplay Programming Model](03-authoring/gameplay-programming-model.md)が最初のV1から直接所有し、旧Owner revision、移管元、aliasまたはdual Registryをcurrent Architectureへ定義しない。

## 2. 読む順序

1. [Product](#31-product)で製品意図とCapabilityの扱いを確認する。
2. [Governance](#32-governance)で文書責務、AIの認可、安全、検証を確認する。
3. [Foundation](#33-foundation)で共通architecture、契約、互換性、toolchain、命名、言語、math、memoryを確認する。
4. 制作機能は[Authoring](#34-authoring)、実行制御とWorld dataは[Runtime](#35-runtime)を読む。
5. 機能実装では[Simulation](#36-simulation)、[Rendering](#37-rendering)、[Platform](#38-platform)、[Networking](#310-networking)から該当Ownerを読む。
6. 再利用FeatureとGenre固有compositionは最後に[Packs](#39-packs)を読み、参照するSubsystem Ownerへ戻る。

個別変更では全仕様を通読せず、対象概念の正本範囲、非正本範囲、依存、Owner refを先に確認する。

## 3. Owner文書一覧（全件`review`）

### 3.1 Product

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 1 | [Product Plan](00-product/product-plan.md) | review | Product intent、scope、Capability portfolio、MVP・昇格判断 |
| 2 | [Product Lifecycle](00-product/product-lifecycle.md) | review | Project bootstrap、Template／Sample／Documentation、surface parity、update／repair／support／NOTICE、Product lifecycle acceptance |
| 3 | [Product Release Decision](00-product/product-release-decision.md) | review | Release requirement closure、署名済みauthority／quorum／freshness／revocation、current release authorization |
| 4 | [Product Publication／Completion](00-product/product-publication-completion.md) | review | Platform channel publication集約、published state、support開始binding、署名済みProduct completion |

### 3.2 Governance

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 5 | [AI Security／Approval](01-governance/ai-security-approval.md) | review | AI authorization、risk、trust boundary、sandbox、人間承認、Consent purpose binding |
| 6 | [AI Verification／Provenance](01-governance/ai-verification-provenance.md) | review | verification、test集約／retry／quarantine、evidence、provenance、trace grading |
| 7 | [Architecture Governance](01-governance/architecture-governance.md) | review | 文書／実装／検証等の状態軸、根拠区分、Inventory、一意所有、規範依存、ADR |
| 8 | [Product Security](01-governance/product-security.md) | review | Product threat ownership、security baseline、vulnerability response／update／disclosure／incident |
| 9 | [Product Privacy／Data Governance](01-governance/product-privacy-data-governance.md) | review | Product data inventory、purpose／consent、processor／region、retention、export／delete、privacy release acceptance |

### 3.3 Foundation

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 10 | [Core Architecture](02-foundation/core-architecture.md) | review | 基盤layer、host／process、状態変更境界、ownership、repository境界 |
| 11 | [Toolchain／Dependencies](02-foundation/toolchain-dependencies.md) | review | 外部tool、SDK、library、APIの採用、固定、更新根拠 |
| 12 | [Executable Contracts](02-foundation/executable-contracts.md) | review | MCD、operation、state machine、diagnostic、contract projection |
| 13 | [Compatibility／Evolution](02-foundation/compatibility-evolution.md) | review | clean break、format evolution、reader／writer／alias policy、recook／migration evidence |
| 14 | [Naming／Project Layout](02-foundation/naming-project-layout.md) | review | 共通語彙、技術識別子、Engine／Project配置、生成物配置 |
| 15 | [C++23 Language／Public Surface](02-foundation/cpp23-modules.md) | review | C++ language profile、required feature closure、単一Header-based Shipping surface、Named Modules禁止境界 |
| 16 | [Math／Core Utilities](02-foundation/math-core.md) | review | math型、座標／単位、数値規則、core utility |
| 17 | [Memory／Pointers](02-foundation/memory-pointers.md) | review | memory ownership、pointer taxonomy、handle、allocation domain |

### 3.4 Authoring

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 18 | [Project State](03-authoring/project-state.md) | review | Project aggregate、revision、ChangeSet、commit、undo／recovery |
| 19 | [Game Production Loop](03-authoring/game-production-loop.md) | review | Intent／Brief／GameSpec、理解closure、Playtest評価、Iteration Decision、AI生成lane |
| 20 | [Asset Lifecycle](03-authoring/asset-lifecycle.md) | review | Asset import、Derived Artifact、catalog、package assembly、promotion |
| 21 | [Editor UI Framework](03-authoring/editor-ui-framework.md) | review | Editor UI core、layout、event、rendering、platform／accessibility bridge |
| 22 | [Editor Workspace／UX](03-authoring/editor-workspace-ux.md) | review | Workspace、制作journey、AI partner、error／recovery UX |
| 23 | [Gameplay Programming Model](03-authoring/gameplay-programming-model.md) | review | 構造化Gameplay、Game System、Project実装、code generation |
| 24 | [Native Game Module](03-authoring/native-game-module.md) | review | Native module ABI、build、link、package、promotion evidence |
| 25 | [Developer Testing](03-authoring/developer-testing.md) | review | Project test suite／case／assertion、GUI／CLI／headless runner、isolation、result、public C++ test API |

### 3.5 Runtime

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 26 | [Runtime ECS](04-runtime/entity-component-system.md) | review | Entity／Component、archetype、query、access manifest、structural transaction |
| 27 | [Scheduling／Lifetime](04-runtime/scheduling-lifetime.md) | review | Simulation Advance、execution order、job dependency、message order、lifetime |
| 28 | [Runtime Package](04-runtime/runtime-package.md) | review | Runtime Entry launch closure、world／ui／headless branch、World Root／Section image、loader、publication |
| 29 | [Runtime Asset Lifecycle](04-runtime/runtime-asset-lifecycle.md) | review | Cook済みArtifactのrequest、dependency、generation、residency、lease、eviction、recovery |
| 30 | [Persistence／Save](04-runtime/persistence-save.md) | review | Save Catalog／slot membership、Save、persistent identity projection、digest、reconstruction、Replay projection |
| 31 | [Performance／Capacity](04-runtime/performance-capacity.md) | review | 測定境界、暫定budget／capacity、backpressure、regression |
| 32 | [Debugging／Observability／Replay](04-runtime/debugging-observability-replay.md) | review | debug、telemetry、causality、capture、replay transport、crash evidence |

### 3.6 Simulation

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 33 | [Collision](05-simulation/collision.md) | review | geometry、collider、filter、query、contact／trigger／hit semantics |
| 34 | [Physics](05-simulation/physics.md) | review | dynamics、body、constraint、kinematic motion、kernel adapter |
| 35 | [Navigation](05-simulation/navigation.md) | review | 2D grid、3D navmesh、navigation query、artifact lifetime |
| 36 | [Animation](05-simulation/animation.md) | review | animation source／artifact、graph、pose、event、root motion、retarget |

### 3.7 Rendering

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 37 | [Render Graph](06-rendering/render-graph.md) | review | renderer boundary、resource／pass graph、2D packet／sort／batch、visibility、temporal execution |
| 38 | [Materials](06-rendering/materials.md) | review | material／shader authoring、semantic intent、compile、package |
| 39 | [Project Shader](06-rendering/project-shader.md) | review | bounded HLSL、semantic module、Project Node、Technique、qualification |
| 40 | [Lighting](06-rendering/lighting.md) | review | light source、photometry、attenuation、shadow intent、lighting semantics |
| 41 | [Advanced Light Transport](06-rendering/advanced-light-transport.md) | review | GI／reflection／advanced shadow／reference transport、channel別Technique／Target support／fallback |
| 42 | [Post Processing](06-rendering/post-processing.md) | review | post-process source、volume、effect composition、history intent |
| 43 | [VFX Authoring](06-rendering/vfx-authoring.md) | review | VFX source、semantic catalog、typed graph、compiler、planned authoring action |
| 44 | [VFX Runtime](06-rendering/vfx-runtime.md) | review | VFX artifact、instance、simulation、render、visual interaction |
| 45 | [Camera](06-rendering/camera.md) | review | Camera profile、rig、director、sequence、runtime／authoring |
| 46 | [Environment／Surfaces](06-rendering/environment-surfaces.md) | review | sky、fog、weather presentation、water、snow／wetness surface response |
| 47 | [Terrain／Foliage](06-rendering/terrain-foliage.md) | review | Terrain source／tile／layer／artifact、Foliage species／placement／identity／artifact |
| 48 | [LOD](06-rendering/lod.md) | review | representation selection、transition、fallback、LOD semantics |
| 49 | [Virtualized／Continuous Geometry](06-rendering/virtualized-continuous-geometry.md) | review | virtual geometry表現ファミリ、inner cut、residency／fallback統合、feature qualification |
| 50 | [World／Scene／Space／Cell](06-rendering/world.md) | review | World composition、reusable Scene instance／override／rebase、spatial topology、partition、activation、generic transition |

### 3.8 Platform

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 51 | [Windows](07-platform/windows.md) | review | Windows target、process／window adapter、package、distribution、qualification |
| 52 | [Mobile Common](07-platform/mobile-common.md) | review | Mobile共通target、port、lifecycle、resource policy、device workflow |
| 53 | [Android](07-platform/android.md) | review | Android build／runtime adapter、package、store delivery、device qualification |
| 54 | [Apple](07-platform/apple.md) | review | Apple build／runtime bridge、signing、store、device qualification |
| 55 | [Input](07-platform/input.md) | review | device、action、binding、context、remap、haptics、input replay |
| 56 | [Audio](07-platform/audio.md) | review | Audio asset semantics、cue、voice、mixer、spatial、streaming |
| 57 | [UI／Text／Localization／Accessibility](07-platform/ui-text-localization-accessibility.md) | review | Game UI、text、localization、focus、accessibility、Settings／Save Catalog co-publication、UI authoring |

### 3.9 Packs

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 58 | [Pack Contract](08-packs/pack-contract.md) | review | Pack構造、dependency、install、update、removal |
| 59 | [Shooter Genre Pack](08-packs/shooter.md) | review | Shooter固有composition、Profile、Game Flow、Action role |
| 60 | [Gameplay Feature Packs](08-packs/gameplay-features.md) | review | reusable Featureの共通ownership、RPG Feature family、manifest、Port、State、Save／Replay、failure |
| 61 | [Scenario／Stage Feature Pack](08-packs/scenario-stage.md) | review | optional Stage、completion、Scope、transition、Save／Replay |
| 62 | [RPG Genre Pack](08-packs/rpg.md) | review | RPG Feature composition、Profile、Game Flow、command role、Reference fixture binding |

### 3.10 Networking

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 63 | [Network Transport／Connection](09-networking/network-transport-connection.md) | review | endpoint、handshake、connection epoch、semantic delivery、packet／flow／security binding |
| 64 | [Multiplayer Authority／Replication](09-networking/multiplayer-authority-replication.md) | review | topology／role／authority、session、Network Object、replication、prediction／rollback、resync／handoff |

## 4. 補助文書とProposal

次の文書はOwner本文から分離した詳細CatalogまたはProposalである。`proposal appendix`はcurrent正本ではなく、`Owner supplement`も親Ownerと同じ`review`状態であり、実装済みを意味しない。

| 文書 | 種別 | 内容 |
|---|---|---|
| [Product Execution Registry Proposal](appendices/product-execution-registry-proposal.md) | proposal appendix | Product Registry、Policy、Gate、Work Package候補 |
| [AI Provider／MCP Security Supplement](appendices/ai-provider-mcp-security-supplement.md) | Owner supplement | Provider、MCP、CLI、Pluginのsecurity詳細 |
| [AI Security Assumptions／Questions Guide](appendices/ai-security-assumptions-guide.md) | explanatory supplement | 質問、assumption、Project data境界 |
| [AI Evidence Envelope／Fixture Candidate Catalog](appendices/ai-evidence-envelope-fixture-catalog.md) | Owner supplement | Evidence envelope、Receipt、AI Eval、CI Fixture候補 |
| [Executable Contracts Operation／Planning Candidate Catalog](appendices/executable-contracts-operation-planning-catalog.md) | Owner supplement | 具体MCD record、Operation catalog、未Activation planning候補 |
| [Gameplay Generated Projection／Fixture Candidate Catalog](appendices/gameplay-generated-projection-fixture-catalog.md) | Owner supplement | Gameplay Schema、Registry、generated projection、Fixture候補 |
| [Project Target Readiness／Fixture Candidate Catalog](appendices/project-target-readiness-fixture-catalog.md) | Owner supplement | Project Document、Runtime Entry、Target readiness、Change primitive候補 |
| [Editor UI Design System Catalog](appendices/editor-ui-design-system-catalog.md) | Owner supplement | Widget Pattern、visual／semantic／UIA、Reference Fixture候補 |
| [Editor Panel／Reference Catalog](appendices/editor-panel-reference-catalog.md) | Owner supplement | Panel、Reference Design、environment／coverage候補 |
| [Performance Scale Catalog Proposal](appendices/performance-scale-catalog-proposal.md) | proposal appendix | workload scale、Registry、integrated fixture候補 |
| [Physics AI Catalog Proposal](appendices/physics-ai-catalog-proposal.md) | proposal appendix | Physics intent、role、Operation、AI Eval候補 |
| [Procedural World Catalog／Fixture Candidate](appendices/procedural-world-catalog-fixture.md) | Owner supplement | World Schema、procedural generation、Tilemap、Blockout、Fixture候補 |
| [Gameplay Feature Definition／Fixture Candidate Catalog](appendices/gameplay-feature-definition-fixture-catalog.md) | Owner supplement | Weapon、Damage、Vital、Score、Encounter、Pickup、Locomotion候補 |
| [Shooter Reference Catalog](appendices/shooter-reference-catalog.md) | proposal appendix | AI composition、Fixture、Difficulty、Input template候補 |
| [Governance Migration Proposals](appendices/governance-migration-proposals.md) | proposal appendix | 初回公開後のDefinition／Owner変更に限る未承認binding候補 |
| [Navigation Design Alignment Review](appendices/navigation-design-alignment-review.md) | proposal appendix | NavigationのAI可読性、2D／3D、cross-owner整合性、未解決Closure |
| [Runtime ECS Design Closure Review](appendices/runtime-ecs-design-closure-review.md) | proposal appendix | ECS設計監査、initial V1 Owner境界、cross-owner整合性、未解決Closure |
| [Architecture Plan Closure Review](appendices/architecture-plan-closure-review.md) | proposal appendix | AI可読性、AI-native C++ Product identity／非模倣境界、Runtime coverage、Editor／Game分離、Target別Build、Advanced Rendering／Multiplayer／Future Target closure、cross-owner整合性、未解決Closure |

## 5. Decision Log

[Decision Log](decisions/README.md)はDecisionのrationaleと履歴を所有し、current Schemaまたはruntime behaviorを所有しない。

Selected review Decisions: [AI-readable Asset／Memory／Async Loading Alignment](decisions/2026-07-28-ai-asset-memory-async-alignment.md)、[Product Lifecycle／Product Security Ownership](decisions/2026-07-29-product-lifecycle-security-ownership.md)、[Advanced Rendering／Multiplayer Ownership](decisions/2026-07-29-advanced-rendering-multiplayer-ownership.md)、[Android Compile／Target SDK and Vulkan Profile Baseline](decisions/2026-07-29-android-release-baseline.md)、[Product Release／Publication Authority Ownership](decisions/2026-07-30-product-release-publication-authority.md)、[C++23 Header Shipping／Toolchain Baseline](decisions/2026-07-30-cxx23-header-shipping-toolchain-baseline.md)、[AI-native C++ Product Identity](decisions/2026-08-03-ai-native-cpp-product-identity.md)、[Runtime ECS Static Definition／Entity Reference Boundary](decisions/2026-08-03-runtime-ecs-static-and-entity-reference-boundary.md)、[Initial Morph Capability Boundary](decisions/2026-08-03-initial-morph-capability-boundary.md)、[MCP Current Protocol Baseline](decisions/2026-08-03-mcp-current-protocol-baseline.md)、[Android Adaptive Game Window Baseline](decisions/2026-08-03-android-adaptive-game-window-baseline.md)。これは主要なcross-domain reviewへの抜粋navigationであり、全Decisionのcanonical membershipとstatusは[Decision Log](decisions/README.md#3-decision-log)だけを参照する。これらのLinkは各Owner文書のcurrent Contractを置き換えない。

Current design review: [Runtime ECS Design Closure Review](appendices/runtime-ecs-design-closure-review.md)。このReviewはAI可読性、外部Engine比較、Memory／ECS／Performance／AI／Projectのcross-owner整合性、未解決Closureへのnavigationであり、Runtime ECS semantics、Capability Activationまたは実装計画を所有しない。

Cross-domain design review: [Architecture Plan Closure Review](appendices/architecture-plan-closure-review.md)。このReviewはAI可読性、AI-native C++ Product identity／外部Engine非模倣境界、Runtime、Asset、Editor／Game分離、Target別Build、Advanced Rendering／Multiplayer／Future Target-role closureの監査結論と未解決Authorityへのnavigationであり、Subsystem semantics、Capability Activation、実装Taskまたは実装計画を所有しない。

External consultation record: [ChatGPT Pro Review Summary](../reviews/README.md)。この要約は非規範の監査記録であり、Owner文書、Decision、Repository Evidenceまたは外部一次資料を置き換えない。全文Transcriptは監査収束後に保持しない。

## 6. 変更時の入口

| 調べたいこと | 先に読む文書 | 次に辿るOwner |
|---|---|---|
| 正本追加、統廃合、Owner移管 | [Architecture Governance](01-governance/architecture-governance.md) | [Compatibility／Evolution](02-foundation/compatibility-evolution.md)、対象Domain |
| DecisionのContext、比較案、置換履歴 | [Decision Log](decisions/README.md) | relevant Domain Owner |
| AIによるArchitecture／ECS／最適化の説明 | [Architecture Governance](01-governance/architecture-governance.md) | [Product Plan](00-product/product-plan.md)、[Runtime ECS](04-runtime/entity-component-system.md)、[Memory／Pointers](02-foundation/memory-pointers.md)、[Performance／Capacity](04-runtime/performance-capacity.md)、[Runtime ECS Design Closure Review](appendices/runtime-ecs-design-closure-review.md) |
| AI-native C++ Product identity、manual／AI parity、外部Engine非模倣、initial V1 clean-break | [Product Plan](00-product/product-plan.md#11-ai-native-c-product-identity) | [AI-native C++ Product Identity Decision](decisions/2026-08-03-ai-native-cpp-product-identity.md)、[Gameplay Programming Model](03-authoring/gameplay-programming-model.md#11-ai-native-gameplay-authoring-closure)、[Compatibility／Evolution](02-foundation/compatibility-evolution.md#41-independent-initial-v1-boundary)、[Executable Contracts](02-foundation/executable-contracts.md)、[Project State](03-authoring/project-state.md)、[AI Security／Approval](01-governance/ai-security-approval.md)、[AI Verification／Provenance](01-governance/ai-verification-provenance.md) |
| AIによる提案・変更、安全、evidence | [AI Security／Approval](01-governance/ai-security-approval.md) | [Project State](03-authoring/project-state.md)、[AI Verification／Provenance](01-governance/ai-verification-provenance.md)、対象Owner |
| Project作成、Template／Sample／Documentation、update／repair／support／NOTICE | [Product Lifecycle](00-product/product-lifecycle.md) | [Product Plan](00-product/product-plan.md)、[Project State](03-authoring/project-state.md)、[Compatibility／Evolution](02-foundation/compatibility-evolution.md)、[Toolchain／Dependencies](02-foundation/toolchain-dependencies.md)、各Platform Owner |
| Product threat ownership、vulnerability response、security update／disclosure／incident | [Product Security](01-governance/product-security.md) | [AI Security／Approval](01-governance/ai-security-approval.md)、[Toolchain／Dependencies](02-foundation/toolchain-dependencies.md)、[Debugging／Observability／Replay](04-runtime/debugging-observability-replay.md)、各Platform／domain Owner |
| Product data inventory、telemetry／crash／AI／supportのpurpose／consent／retention／export／delete | [Product Privacy／Data Governance](01-governance/product-privacy-data-governance.md) | [Product Lifecycle](00-product/product-lifecycle.md)、[AI Security／Approval](01-governance/ai-security-approval.md)、[Debugging／Observability／Replay](04-runtime/debugging-observability-replay.md)、各Platform Owner |
| Runtime coverage、Editor／Game分離、Target別Build、未解決Authority | [Architecture Plan Closure Review](appendices/architecture-plan-closure-review.md) | [Runtime Asset Lifecycle](04-runtime/runtime-asset-lifecycle.md)、[Runtime Package](04-runtime/runtime-package.md)、[Scheduling／Lifetime](04-runtime/scheduling-lifetime.md)、[Editor Workspace／UX](03-authoring/editor-workspace-ux.md)、[Toolchain／Dependencies](02-foundation/toolchain-dependencies.md)、各Platform Owner |
| 英語技術語彙、Editor表示locale、AI返答locale、Game source locale | [Naming／Project Layout](02-foundation/naming-project-layout.md) | [Editor Workspace／UX](03-authoring/editor-workspace-ux.md)、[Editor UI Framework](03-authoring/editor-ui-framework.md)、[Project State](03-authoring/project-state.md)、[UI／Text／Localization／Accessibility](07-platform/ui-text-localization-accessibility.md)、[AI Verification／Provenance](01-governance/ai-verification-provenance.md) |
| ECS、World load、Save／Replay | [Runtime ECS](04-runtime/entity-component-system.md) | [Runtime Package](04-runtime/runtime-package.md)、[Persistence／Save](04-runtime/persistence-save.md)、[Scheduling／Lifetime](04-runtime/scheduling-lifetime.md) |
| ECS static phase identity、snapshot-bound／cross-advance Entity Ref | [Runtime ECS](04-runtime/entity-component-system.md) | [Runtime ECS Static Definition／Entity Reference Boundary](decisions/2026-08-03-runtime-ecs-static-and-entity-reference-boundary.md)、[Scheduling／Lifetime](04-runtime/scheduling-lifetime.md)、[Native Game Module](03-authoring/native-game-module.md)、[Persistence／Save](04-runtime/persistence-save.md) |
| Runtime Asset request、generation、residency、lease、eviction | [Runtime Asset Lifecycle](04-runtime/runtime-asset-lifecycle.md) | [Asset Lifecycle](03-authoring/asset-lifecycle.md)、[Runtime Package](04-runtime/runtime-package.md)、対象consumer |
| Morph initial V1除外とFuture end-to-end closure | [Asset Lifecycle](03-authoring/asset-lifecycle.md) | [Initial Morph Capability Boundary](decisions/2026-08-03-initial-morph-capability-boundary.md)、[Animation](05-simulation/animation.md)、[LOD](06-rendering/lod.md)、[Virtualized／Continuous Geometry](06-rendering/virtualized-continuous-geometry.md)、[Persistence／Save](04-runtime/persistence-save.md) |
| MCP protocol baseline、version carrier、materialization gate | [Toolchain／Dependencies](02-foundation/toolchain-dependencies.md) | [MCP Current Protocol Baseline](decisions/2026-08-03-mcp-current-protocol-baseline.md)、[Executable Contracts](02-foundation/executable-contracts.md)、[AI Security／Approval](01-governance/ai-security-approval.md)、[AI Provider／MCP Security Supplement](appendices/ai-provider-mcp-security-supplement.md) |
| Mobile orientation／resizabilityとAndroid adaptive window | [Mobile Common](07-platform/mobile-common.md) | [Android Adaptive Game Window Baseline](decisions/2026-08-03-android-adaptive-game-window-baseline.md)、[Android](07-platform/android.md)、[Product Plan](00-product/product-plan.md) |
| Project編集、Asset、Editor、Gameplay実装 | [Project State](03-authoring/project-state.md) | [Asset Lifecycle](03-authoring/asset-lifecycle.md)、[Gameplay Programming Model](03-authoring/gameplay-programming-model.md) |
| Game Projectのtest suite／runner／assertion／GUI・CLI・headless parity | [Developer Testing](03-authoring/developer-testing.md) | [Native Game Module](03-authoring/native-game-module.md)、[Project State](03-authoring/project-state.md)、[Product Lifecycle](00-product/product-lifecycle.md) |
| 共通型、toolchain、命名、memory | [Core Architecture](02-foundation/core-architecture.md) | 対象Subsystem |
| Pointer／ownership／handle／lease／allocation | [Memory／Pointers](02-foundation/memory-pointers.md) | [Executable Contracts](02-foundation/executable-contracts.md)、[Product Plan](00-product/product-plan.md)、対象Subsystemのconsumer binding |
| 実行順、capacity、debug／replay transport | [Scheduling／Lifetime](04-runtime/scheduling-lifetime.md) | [Performance／Capacity](04-runtime/performance-capacity.md)、[Debugging／Observability／Replay](04-runtime/debugging-observability-replay.md) |
| GI／Reflections／advanced shadow／reference transport、Terrain／Foliage | [Advanced Light Transport](06-rendering/advanced-light-transport.md) | [Terrain／Foliage](06-rendering/terrain-foliage.md)、[Lighting](06-rendering/lighting.md)、[Materials](06-rendering/materials.md)、[World](06-rendering/world.md)、[LOD](06-rendering/lod.md)、[Render Graph](06-rendering/render-graph.md) |
| endpoint／handshake／deliveryとgameplay session／authority／replication | [Network Transport／Connection](09-networking/network-transport-connection.md) | [Multiplayer Authority／Replication](09-networking/multiplayer-authority-replication.md)、[Runtime Package](04-runtime/runtime-package.md)、[Scheduling／Lifetime](04-runtime/scheduling-lifetime.md) |
| Futureの単一Target前提、client／authority／operations／HostのTarget-role bundle、claim release | [Product Plan](00-product/product-plan.md#8-future-portfolio) | [Product Execution Registry Proposal](appendices/product-execution-registry-proposal.md)、該当Future Owner、[Advanced Rendering／Multiplayer Ownership Decision](decisions/2026-07-29-advanced-rendering-multiplayer-ownership.md) |
| RPG Feature、Genre composition、Reference Game境界 | [Gameplay Feature Packs](08-packs/gameplay-features.md) | [RPG Genre Pack](08-packs/rpg.md)、[Product Plan](00-product/product-plan.md) |

## 7. Indexの更新規則

Indexを更新してもArchitecture、Schema、Gate、実装計画、Operation activationは変化しない。文書追加・統廃合・retirementでは、対象OwnerのHeader、正本範囲、規範依存、関連文書、内部linkを先に整合させ、最後にこの手動一覧を更新する。

Inventory Generatorと生成ArtifactがRepositoryへ追加されるまでは、このIndexを`generated`、`materialized`、`exact projection`と表現しない。
