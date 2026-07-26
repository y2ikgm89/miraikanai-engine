# Miraikanai Engine Architecture Index

## 1. 目的と非規範性

このIndexはArchitecture文書を発見し、読む順序と責務の入口を示すためのnavigationである。EngineのSchema、固定値、Gate、実装順序、Capability activationを定義しない。

current文書件数、文書ID、状態、path、依存は各Headerから生成する[Architecture Governance](01-governance/architecture-governance.md)の`ArchitectureInventoryV1`が正本である。この一覧はそのInventoryと一致する閲覧用projectionであり、一覧だけを根拠にOwnerやcurrent Contractを推測してはならない。

## 2. 読む順序

1. [Product](#31-product)で製品意図とCapabilityの扱いを確認する。
2. [Governance](#32-governance)で文書責務、AIの認可、安全、検証を確認する。
3. [Foundation](#33-foundation)で共通architecture、契約、互換性、toolchain、命名、言語、math、memoryを確認する。
4. 制作機能は[Authoring](#34-authoring)、実行制御とWorld dataは[Runtime](#35-runtime)を読む。
5. 機能実装では[Simulation](#36-simulation)、[Rendering](#37-rendering)、[Platform](#38-platform)から該当Ownerを読む。
6. 再利用FeatureとGenre固有compositionは最後に[Packs](#39-packs)を読み、参照するSubsystem Ownerへ戻る。

個別変更では全仕様を通読せず、対象概念の正本範囲、非正本範囲、依存、Owner refを先に確認する。

## 3. 正本一覧

### 3.1 Product

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 1 | [Product Plan](00-product/product-plan.md) | review | Product intent、scope、Capability portfolio、Work Package宣言 |

### 3.2 Governance

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 2 | [AI Security／Approval](01-governance/ai-security-approval.md) | review | AI authorization、risk、trust boundary、sandbox、credential、人間承認 |
| 3 | [AI Verification／Provenance](01-governance/ai-verification-provenance.md) | review | verification、evaluation、evidence、provenance、trace grading |
| 4 | [Architecture Governance](01-governance/architecture-governance.md) | review | 文書Inventory、正本責務移管、Architecture ChangeSet、AI explain projection |

### 3.3 Foundation

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 5 | [Core Architecture](02-foundation/core-architecture.md) | review | 基盤layer、host／process、状態変更境界、ownership、repository境界 |
| 6 | [Toolchain／Dependencies](02-foundation/toolchain-dependencies.md) | review | 外部tool、SDK、library、APIの採用、固定、更新根拠 |
| 7 | [Executable Contracts](02-foundation/executable-contracts.md) | review | MCD、operation、state machine、diagnostic、contract projection |
| 8 | [Compatibility／Evolution](02-foundation/compatibility-evolution.md) | review | clean break、format evolution、reader／writer／alias policy、recook／migration evidence |
| 9 | [Naming／Project Layout](02-foundation/naming-project-layout.md) | review | 共通語彙、技術識別子、Engine／Project配置、生成物配置 |
| 10 | [C++23 Modules](02-foundation/cpp23-modules.md) | review | C++ language profile、module境界、standard-library移行 |
| 11 | [Math／Core Utilities](02-foundation/math-core.md) | review | math型、座標／単位、数値規則、core utility |
| 12 | [Memory／Pointers](02-foundation/memory-pointers.md) | review | memory ownership、pointer taxonomy、handle、allocation domain |

### 3.4 Authoring

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 13 | [Project State](03-authoring/project-state.md) | review | Project aggregate、revision、ChangeSet、commit、undo／recovery |
| 14 | [Asset Lifecycle](03-authoring/asset-lifecycle.md) | review | Asset import、Derived Artifact、catalog、package assembly、promotion |
| 15 | [Editor UI Framework](03-authoring/editor-ui-framework.md) | review | Editor UI core、widget、layout、event、rendering、accessibility bridge |
| 16 | [Editor Workspace／UX](03-authoring/editor-workspace-ux.md) | review | Workspace、制作journey、AI partner、error／recovery UX |
| 17 | [Gameplay Programming Model](03-authoring/gameplay-programming-model.md) | review | 構造化Gameplay、Game System、Project実装、code generation |
| 18 | [Native Game Module](03-authoring/native-game-module.md) | review | Native module ABI、build、link、package、promotion evidence |

### 3.5 Runtime

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 19 | [Runtime ECS](04-runtime/entity-component-system.md) | review | Entity／Component、archetype、query、access manifest、structural transaction |
| 20 | [Scheduling／Lifetime](04-runtime/scheduling-lifetime.md) | review | Simulation Advance、execution order、job dependency、message order、lifetime |
| 21 | [Runtime Package](04-runtime/runtime-package.md) | review | World Root／Section image、package directory、loader、section publication |
| 22 | [Persistence／Save](04-runtime/persistence-save.md) | review | Save、persistent identity projection、digest、reconstruction、Replay projection |
| 23 | [Performance／Capacity](04-runtime/performance-capacity.md) | review | 共通capacity、measurement、backpressure、scale、regression |
| 24 | [Debugging／Observability／Replay](04-runtime/debugging-observability-replay.md) | review | debug、telemetry、causality、capture、replay transport、crash evidence |

### 3.6 Simulation

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 25 | [Collision](05-simulation/collision.md) | review | geometry、collider、filter、query、contact／trigger／hit semantics |
| 26 | [Physics](05-simulation/physics.md) | review | dynamics、body、constraint、character、kernel adapter、AI semantics |
| 27 | [Navigation](05-simulation/navigation.md) | review | 2D grid、3D navmesh、navigation query、artifact lifetime |
| 28 | [Animation](05-simulation/animation.md) | review | animation source／artifact、graph、pose、event、root motion、retarget |

### 3.7 Rendering

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 29 | [Render Graph](06-rendering/render-graph.md) | review | renderer boundary、resource／pass graph、queue、visibility、temporal execution |
| 30 | [Materials](06-rendering/materials.md) | review | material／shader authoring、semantic intent、compile、package |
| 31 | [Project Shader](06-rendering/project-shader.md) | review | bounded HLSL、semantic module、Project Node、Technique、qualification |
| 32 | [Lighting](06-rendering/lighting.md) | review | light source、photometry、attenuation、shadow intent、lighting semantics |
| 33 | [Post Processing](06-rendering/post-processing.md) | review | post-process source、volume、effect composition、history intent |
| 34 | [VFX Authoring](06-rendering/vfx-authoring.md) | review | VFX source、semantic catalog、typed graph、compiler、planned authoring action |
| 35 | [VFX Runtime](06-rendering/vfx-runtime.md) | review | VFX artifact、instance、simulation、render、visual interaction |
| 36 | [Camera](06-rendering/camera.md) | review | Camera profile、rig、director、sequence、runtime／authoring |
| 37 | [Environment／Surfaces](06-rendering/environment-surfaces.md) | review | sky、fog、weather presentation、water、snow／wetness surface response |
| 38 | [LOD](06-rendering/lod.md) | review | representation selection、transition、fallback、LOD semantics |
| 39 | [World／Scene／Space／Cell](06-rendering/world.md) | review | World composition、spatial topology、partition、activation、generic transition |

### 3.8 Platform

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 40 | [Windows](07-platform/windows.md) | review | Windows target、process／window adapter、package、distribution、qualification |
| 41 | [Mobile Common](07-platform/mobile-common.md) | review | Mobile共通target、port、lifecycle、resource policy、device workflow |
| 42 | [Android](07-platform/android.md) | review | Android build／runtime adapter、package、store delivery、device qualification |
| 43 | [Apple](07-platform/apple.md) | review | Apple build／runtime bridge、signing、store、device qualification |
| 44 | [Input](07-platform/input.md) | review | device、action、binding、context、remap、haptics、input replay |
| 45 | [Audio](07-platform/audio.md) | review | Audio asset semantics、cue、voice、mixer、spatial、streaming |
| 46 | [UI／Text／Localization／Accessibility](07-platform/ui-text-localization-accessibility.md) | review | Game UI、text、localization、focus、accessibility、UI authoring |

### 3.9 Packs

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 47 | [Pack Contract](08-packs/pack-contract.md) | review | Pack構造、dependency、install、update、removal |
| 48 | [Shooter Genre Pack](08-packs/shooter.md) | review | Shooter固有composition、Profile、Game Flow、Action role、fixture |
| 49 | [Gameplay Feature Packs](08-packs/gameplay-features.md) | review | Combat、Ranged Combat、Encounter、Scoring、Pickupのcontract catalog |
| 50 | [Scenario／Stage Feature Pack](08-packs/scenario-stage.md) | review | optional Stage、completion、Scope、transition、Save／Replay |

### 3.10 Decisions

| # | 文書 | 状態 | 閲覧上の責務 |
|---:|---|---|---|
| 51 | [Architecture Document System Restructure](decisions/2026-07-21-document-system-restructure.md) | normative | 文書体系の再編原則、旧Path互換を残さない移行方針 |
| 52 | [Runtime ECS Contract Decision](decisions/2026-07-22-runtime-ecs-contract.md) | review | Engine-owned ECSの採用判断、責務分割、current化の承認境界 |

## 4. 変更時の入口

| 調べたいこと | 先に読む文書 | 次に辿るOwner |
|---|---|---|
| 正本追加、統廃合、Owner移管 | [Architecture Governance](01-governance/architecture-governance.md) | [Compatibility／Evolution](02-foundation/compatibility-evolution.md)、対象Domain |
| AIによる説明・変更、安全、evidence | [AI Security／Approval](01-governance/ai-security-approval.md) | [AI Verification／Provenance](01-governance/ai-verification-provenance.md)、対象Owner |
| 英語技術語彙、Editor表示locale、AI返答locale、Game source locale | [Naming／Project Layout](02-foundation/naming-project-layout.md) | [Editor Workspace／UX](03-authoring/editor-workspace-ux.md)、[Editor UI Framework](03-authoring/editor-ui-framework.md)、[Project State](03-authoring/project-state.md)、[UI／Text／Localization／Accessibility](07-platform/ui-text-localization-accessibility.md)、[AI Verification／Provenance](01-governance/ai-verification-provenance.md) |
| ECS、World load、Save／Replay | [Runtime ECS](04-runtime/entity-component-system.md) | [Runtime Package](04-runtime/runtime-package.md)、[Persistence／Save](04-runtime/persistence-save.md)、[Scheduling／Lifetime](04-runtime/scheduling-lifetime.md) |
| Project編集、Asset、Editor、Gameplay実装 | [Project State](03-authoring/project-state.md) | [Asset Lifecycle](03-authoring/asset-lifecycle.md)、[Gameplay Programming Model](03-authoring/gameplay-programming-model.md) |
| 共通型、toolchain、命名、memory | [Core Architecture](02-foundation/core-architecture.md) | 対象Subsystem |
| Pointer／ownership／handle／lease／allocation | [Memory／Pointers](02-foundation/memory-pointers.md) | [Executable Contracts](02-foundation/executable-contracts.md)、[Product Plan](00-product/product-plan.md)、対象Subsystemのconsumer binding |
| 実行順、capacity、debug／replay transport | [Scheduling／Lifetime](04-runtime/scheduling-lifetime.md) | [Performance／Capacity](04-runtime/performance-capacity.md)、[Debugging／Observability／Replay](04-runtime/debugging-observability-replay.md) |

## 5. Indexの更新規則

Indexを更新してもArchitecture、Schema、Gate、実装計画、Operation activationは変化しない。文書追加・統廃合・retirementでは、先に[Architecture Governance](01-governance/architecture-governance.md)のInventoryとChangeSetを整合させ、次にこの表示projectionを更新する。
