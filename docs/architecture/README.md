# Miraikanai Engine Architecture Index

## 1. 目的と非規範性

このIndexはArchitecture文書を発見し、読む順序と責務の入口を示すための手動管理navigationである。EngineのSchema、固定値、Gate、実装順序、Capability activationを定義しない。

2026-07-29時点で、Architecture InventoryのGenerator、Schema、生成ArtifactはRepositoryに存在しない。この一覧は現存するOwner文書55件を手作業で列挙したものであり、生成済みprojectionではない。件数、状態、path、依存の機械的な正しさを主張せず、変更時は実ファイルと各Headerを確認する。

Owner文書はすべて`review`であり、対応するEngine実装・Schema・Fixture・ReceiptはRepositoryに存在しない。したがって本文中の型、Registry、固定値、hash、Operationは、明示的な外部Artifactを除き設計候補である。状態と根拠の解釈は[Architecture Governance](01-governance/architecture-governance.md)を正本とする。

以下の「Owner文書」は、各概念の目標上の所有先を示す。`review`のままでは採択済み正本または現行実装の証拠にならない。とくにRuntime ECSは移管先候補であり、[Governance Migration Proposals](appendices/governance-migration-proposals.md#2-runtime-ecs-canonicalization-candidate)の候補が完成ChangeSetとして承認され`applied`となるまでは、[Gameplay Programming Model](03-authoring/gameplay-programming-model.md) revision 1に残る現行authorityを置き換えない。

## 2. 読む順序

1. [Product](#31-product)で製品意図とCapabilityの扱いを確認する。
2. [Governance](#32-governance)で文書責務、AIの認可、安全、検証を確認する。
3. [Foundation](#33-foundation)で共通architecture、契約、互換性、toolchain、命名、言語、math、memoryを確認する。
4. 制作機能は[Authoring](#34-authoring)、実行制御とWorld dataは[Runtime](#35-runtime)を読む。
5. 機能実装では[Simulation](#36-simulation)、[Rendering](#37-rendering)、[Platform](#38-platform)から該当Ownerを読む。
6. 再利用FeatureとGenre固有compositionは最後に[Packs](#39-packs)を読み、参照するSubsystem Ownerへ戻る。

個別変更では全仕様を通読せず、対象概念の正本範囲、非正本範囲、依存、Owner refを先に確認する。

## 3. Owner文書一覧（全件`review`）

### 3.1 Product

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 1 | [Product Plan](00-product/product-plan.md) | review | Product intent、scope、Capability portfolio、MVP・昇格判断 |
| 2 | [Product Lifecycle](00-product/product-lifecycle.md) | review | Project bootstrap、Template／Sample／Documentation、surface parity、update／repair／support／NOTICE、Product lifecycle acceptance |

### 3.2 Governance

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 3 | [AI Security／Approval](01-governance/ai-security-approval.md) | review | AI authorization、risk、trust boundary、sandbox、人間承認、Consent purpose binding |
| 4 | [AI Verification／Provenance](01-governance/ai-verification-provenance.md) | review | verification、test集約／retry／quarantine、evidence、provenance、trace grading |
| 5 | [Architecture Governance](01-governance/architecture-governance.md) | review | 文書／実装／検証等の状態軸、根拠区分、Inventory、一意所有、規範依存、ADR |
| 6 | [Product Security](01-governance/product-security.md) | review | Product threat ownership、security baseline、vulnerability response／update／disclosure／incident |

### 3.3 Foundation

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 7 | [Core Architecture](02-foundation/core-architecture.md) | review | 基盤layer、host／process、状態変更境界、ownership、repository境界 |
| 8 | [Toolchain／Dependencies](02-foundation/toolchain-dependencies.md) | review | 外部tool、SDK、library、APIの採用、固定、更新根拠 |
| 9 | [Executable Contracts](02-foundation/executable-contracts.md) | review | MCD、operation、state machine、diagnostic、contract projection |
| 10 | [Compatibility／Evolution](02-foundation/compatibility-evolution.md) | review | clean break、format evolution、reader／writer／alias policy、recook／migration evidence |
| 11 | [Naming／Project Layout](02-foundation/naming-project-layout.md) | review | 共通語彙、技術識別子、Engine／Project配置、生成物配置 |
| 12 | [C++23 Modules](02-foundation/cpp23-modules.md) | review | C++ language profile、module境界、standard-library移行 |
| 13 | [Math／Core Utilities](02-foundation/math-core.md) | review | math型、座標／単位、数値規則、core utility |
| 14 | [Memory／Pointers](02-foundation/memory-pointers.md) | review | memory ownership、pointer taxonomy、handle、allocation domain |

### 3.4 Authoring

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 15 | [Project State](03-authoring/project-state.md) | review | Project aggregate、revision、ChangeSet、commit、undo／recovery |
| 16 | [Asset Lifecycle](03-authoring/asset-lifecycle.md) | review | Asset import、Derived Artifact、catalog、package assembly、promotion |
| 17 | [Editor UI Framework](03-authoring/editor-ui-framework.md) | review | Editor UI core、layout、event、rendering、platform／accessibility bridge |
| 18 | [Editor Workspace／UX](03-authoring/editor-workspace-ux.md) | review | Workspace、制作journey、AI partner、error／recovery UX |
| 19 | [Gameplay Programming Model](03-authoring/gameplay-programming-model.md) | review | 構造化Gameplay、Game System、Project実装、code generation |
| 20 | [Native Game Module](03-authoring/native-game-module.md) | review | Native module ABI、build、link、package、promotion evidence |

### 3.5 Runtime

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 21 | [Runtime ECS](04-runtime/entity-component-system.md) | review（target Owner） | Entity／Component、archetype、query、access manifest、structural transaction |
| 22 | [Scheduling／Lifetime](04-runtime/scheduling-lifetime.md) | review | Simulation Advance、execution order、job dependency、message order、lifetime |
| 23 | [Runtime Package](04-runtime/runtime-package.md) | review | Runtime Entry launch closure、world／ui／headless branch、World Root／Section image、loader、publication |
| 24 | [Runtime Asset Lifecycle](04-runtime/runtime-asset-lifecycle.md) | review（target Owner） | Cook済みArtifactのrequest、dependency、generation、residency、lease、eviction、recovery |
| 25 | [Persistence／Save](04-runtime/persistence-save.md) | review | Save、persistent identity projection、digest、reconstruction、Replay projection |
| 26 | [Performance／Capacity](04-runtime/performance-capacity.md) | review | 測定境界、暫定budget／capacity、backpressure、regression |
| 27 | [Debugging／Observability／Replay](04-runtime/debugging-observability-replay.md) | review | debug、telemetry、causality、capture、replay transport、crash evidence |

### 3.6 Simulation

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 28 | [Collision](05-simulation/collision.md) | review | geometry、collider、filter、query、contact／trigger／hit semantics |
| 29 | [Physics](05-simulation/physics.md) | review | dynamics、body、constraint、kinematic motion、kernel adapter |
| 30 | [Navigation](05-simulation/navigation.md) | review | 2D grid、3D navmesh、navigation query、artifact lifetime |
| 31 | [Animation](05-simulation/animation.md) | review | animation source／artifact、graph、pose、event、root motion、retarget |

### 3.7 Rendering

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 32 | [Render Graph](06-rendering/render-graph.md) | review | renderer boundary、resource／pass graph、2D packet／sort／batch、visibility、temporal execution |
| 33 | [Materials](06-rendering/materials.md) | review | material／shader authoring、semantic intent、compile、package |
| 34 | [Project Shader](06-rendering/project-shader.md) | review | bounded HLSL、semantic module、Project Node、Technique、qualification |
| 35 | [Lighting](06-rendering/lighting.md) | review | light source、photometry、attenuation、shadow intent、lighting semantics |
| 36 | [Post Processing](06-rendering/post-processing.md) | review | post-process source、volume、effect composition、history intent |
| 37 | [VFX Authoring](06-rendering/vfx-authoring.md) | review | VFX source、semantic catalog、typed graph、compiler、planned authoring action |
| 38 | [VFX Runtime](06-rendering/vfx-runtime.md) | review | VFX artifact、instance、simulation、render、visual interaction |
| 39 | [Camera](06-rendering/camera.md) | review | Camera profile、rig、director、sequence、runtime／authoring |
| 40 | [Environment／Surfaces](06-rendering/environment-surfaces.md) | review | sky、fog、weather presentation、water、snow／wetness surface response |
| 41 | [LOD](06-rendering/lod.md) | review | representation selection、transition、fallback、LOD semantics |
| 42 | [Virtualized／Continuous Geometry](06-rendering/virtualized-continuous-geometry.md) | review | virtual geometry表現ファミリ、inner cut、residency／fallback統合、feature qualification |
| 43 | [World／Scene／Space／Cell](06-rendering/world.md) | review | World composition、reusable Scene instance／override／rebase、spatial topology、partition、activation、generic transition |

### 3.8 Platform

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 44 | [Windows](07-platform/windows.md) | review | Windows target、process／window adapter、package、distribution、qualification |
| 45 | [Mobile Common](07-platform/mobile-common.md) | review | Mobile共通target、port、lifecycle、resource policy、device workflow |
| 46 | [Android](07-platform/android.md) | review | Android build／runtime adapter、package、store delivery、device qualification |
| 47 | [Apple](07-platform/apple.md) | review | Apple build／runtime bridge、signing、store、device qualification |
| 48 | [Input](07-platform/input.md) | review | device、action、binding、context、remap、haptics、input replay |
| 49 | [Audio](07-platform/audio.md) | review | Audio asset semantics、cue、voice、mixer、spatial、streaming |
| 50 | [UI／Text／Localization／Accessibility](07-platform/ui-text-localization-accessibility.md) | review | Game UI、text、localization、focus、accessibility、UI authoring |

### 3.9 Packs

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 51 | [Pack Contract](08-packs/pack-contract.md) | review | Pack構造、dependency、install、update、removal |
| 52 | [Shooter Genre Pack](08-packs/shooter.md) | review | Shooter固有composition、Profile、Game Flow、Action role |
| 53 | [Gameplay Feature Packs](08-packs/gameplay-features.md) | review | reusable Featureの共通ownership、RPG Feature family、manifest、Port、State、Save／Replay、failure |
| 54 | [Scenario／Stage Feature Pack](08-packs/scenario-stage.md) | review | optional Stage、completion、Scope、transition、Save／Replay |
| 55 | [RPG Genre Pack](08-packs/rpg.md) | review | RPG Feature composition、Profile、Game Flow、command role、Reference fixture binding |

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
| [Governance Migration Proposals](appendices/governance-migration-proposals.md) | proposal appendix | Definition移管、Runtime ECS Owner移管の未承認候補 |
| [Runtime ECS Design Closure Review](appendices/runtime-ecs-design-closure-review.md) | proposal appendix | ECS設計監査、current／target区分、cross-owner整合性、未解決Closure |
| [Architecture Plan Closure Review](appendices/architecture-plan-closure-review.md) | proposal appendix | AI可読性、Runtime coverage、Editor／Game分離、Target別Build、cross-owner整合性、未解決Closure |

## 5. Decision Log

[Decision Log](decisions/README.md)はDecisionのrationaleと履歴を所有し、current Schemaまたはruntime behaviorを所有しない。

Current review Decisions: [AI-readable Asset／Memory／Async Loading Alignment](decisions/2026-07-28-ai-asset-memory-async-alignment.md)、[Product Lifecycle／Product Security Ownership](decisions/2026-07-29-product-lifecycle-security-ownership.md)。これらのLinkは判断理由へのnavigationであり、各Owner文書のcurrent Contractを置き換えない。

Current design review: [Runtime ECS Design Closure Review](appendices/runtime-ecs-design-closure-review.md)。このReviewはAI可読性、外部Engine比較、Memory／ECS／Performance／AI／Projectのcross-owner整合性、未解決Closureへのnavigationであり、Runtime ECS semantics、Capability Activationまたは実装計画を所有しない。

Cross-domain design review: [Architecture Plan Closure Review](appendices/architecture-plan-closure-review.md)。このReviewはAI可読性、Runtime、Asset、Editor／Game分離、Target別Buildの監査結論と未解決Authorityへのnavigationであり、Subsystem semantics、Capability Activation、実装Taskまたは実装計画を所有しない。

## 6. 変更時の入口

| 調べたいこと | 先に読む文書 | 次に辿るOwner |
|---|---|---|
| 正本追加、統廃合、Owner移管 | [Architecture Governance](01-governance/architecture-governance.md) | [Compatibility／Evolution](02-foundation/compatibility-evolution.md)、対象Domain |
| DecisionのContext、比較案、置換履歴 | [Decision Log](decisions/README.md) | relevant Domain Owner |
| AIによるArchitecture／ECS／最適化の説明 | [Architecture Governance](01-governance/architecture-governance.md) | [Product Plan](00-product/product-plan.md)、[Runtime ECS](04-runtime/entity-component-system.md)、[Memory／Pointers](02-foundation/memory-pointers.md)、[Performance／Capacity](04-runtime/performance-capacity.md)、[Runtime ECS Design Closure Review](appendices/runtime-ecs-design-closure-review.md) |
| AIによる提案・変更、安全、evidence | [AI Security／Approval](01-governance/ai-security-approval.md) | [Project State](03-authoring/project-state.md)、[AI Verification／Provenance](01-governance/ai-verification-provenance.md)、対象Owner |
| Project作成、Template／Sample／Documentation、update／repair／support／NOTICE | [Product Lifecycle](00-product/product-lifecycle.md) | [Product Plan](00-product/product-plan.md)、[Project State](03-authoring/project-state.md)、[Compatibility／Evolution](02-foundation/compatibility-evolution.md)、[Toolchain／Dependencies](02-foundation/toolchain-dependencies.md)、各Platform Owner |
| Product threat ownership、vulnerability response、security update／disclosure／incident | [Product Security](01-governance/product-security.md) | [AI Security／Approval](01-governance/ai-security-approval.md)、[Toolchain／Dependencies](02-foundation/toolchain-dependencies.md)、[Debugging／Observability／Replay](04-runtime/debugging-observability-replay.md)、各Platform／domain Owner |
| Runtime coverage、Editor／Game分離、Target別Build、未解決Authority | [Architecture Plan Closure Review](appendices/architecture-plan-closure-review.md) | [Runtime Asset Lifecycle](04-runtime/runtime-asset-lifecycle.md)、[Runtime Package](04-runtime/runtime-package.md)、[Scheduling／Lifetime](04-runtime/scheduling-lifetime.md)、[Editor Workspace／UX](03-authoring/editor-workspace-ux.md)、[Toolchain／Dependencies](02-foundation/toolchain-dependencies.md)、各Platform Owner |
| 英語技術語彙、Editor表示locale、AI返答locale、Game source locale | [Naming／Project Layout](02-foundation/naming-project-layout.md) | [Editor Workspace／UX](03-authoring/editor-workspace-ux.md)、[Editor UI Framework](03-authoring/editor-ui-framework.md)、[Project State](03-authoring/project-state.md)、[UI／Text／Localization／Accessibility](07-platform/ui-text-localization-accessibility.md)、[AI Verification／Provenance](01-governance/ai-verification-provenance.md) |
| ECS、World load、Save／Replay | [Runtime ECS](04-runtime/entity-component-system.md) | [Runtime Package](04-runtime/runtime-package.md)、[Persistence／Save](04-runtime/persistence-save.md)、[Scheduling／Lifetime](04-runtime/scheduling-lifetime.md) |
| Runtime Asset request、generation、residency、lease、eviction | [Runtime Asset Lifecycle](04-runtime/runtime-asset-lifecycle.md) | [Asset Lifecycle](03-authoring/asset-lifecycle.md)、[Runtime Package](04-runtime/runtime-package.md)、対象consumer |
| Project編集、Asset、Editor、Gameplay実装 | [Project State](03-authoring/project-state.md) | [Asset Lifecycle](03-authoring/asset-lifecycle.md)、[Gameplay Programming Model](03-authoring/gameplay-programming-model.md) |
| 共通型、toolchain、命名、memory | [Core Architecture](02-foundation/core-architecture.md) | 対象Subsystem |
| Pointer／ownership／handle／lease／allocation | [Memory／Pointers](02-foundation/memory-pointers.md) | [Executable Contracts](02-foundation/executable-contracts.md)、[Product Plan](00-product/product-plan.md)、対象Subsystemのconsumer binding |
| 実行順、capacity、debug／replay transport | [Scheduling／Lifetime](04-runtime/scheduling-lifetime.md) | [Performance／Capacity](04-runtime/performance-capacity.md)、[Debugging／Observability／Replay](04-runtime/debugging-observability-replay.md) |
| RPG Feature、Genre composition、Reference Game境界 | [Gameplay Feature Packs](08-packs/gameplay-features.md) | [RPG Genre Pack](08-packs/rpg.md)、[Product Plan](00-product/product-plan.md) |

## 7. Indexの更新規則

Indexを更新してもArchitecture、Schema、Gate、実装計画、Operation activationは変化しない。文書追加・統廃合・retirementでは、対象OwnerのHeader、正本範囲、規範依存、関連文書、内部linkを先に整合させ、最後にこの手動一覧を更新する。

Inventory Generatorと生成ArtifactがRepositoryへ追加されるまでは、このIndexを`generated`、`materialized`、`exact projection`と表現しない。
