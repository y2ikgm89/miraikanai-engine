# Miraikanai Engine Architecture Index

## 1. 目的と非規範性

このIndexはMiraikanai EngineのArchitecture仕様を発見し、読む順序と正本所有者を確認するためのnavigationである。Engine仕様、判断、契約、値、実装順序、合格条件を定義しない。

本文と各正本が矛盾する場合は必ず正本が優先する。このIndexの要約を詳細契約、数値、実装計画、合格条件の代わりに使用してはならない。Decisionは文書体系の変更理由を記録するが、active Engine仕様数には含めない。

## 2. 読む順序

1. [Product](#31-product)で製品意図、scope、Capabilityの扱いを確認する。
2. [Governance](#32-governance)でAIの権限、安全、検証、provenanceを確認する。
3. [Foundation](#33-foundation)で共通architecture、契約、toolchain、命名、言語、math、memoryを確認する。
4. 制作機能は[Authoring](#34-authoring)、実行制御は[Runtime](#35-runtime)を先に読む。
5. 機能実装では[Simulation](#36-simulation)、[Rendering](#37-rendering)、[Platform](#38-platform)から該当ownerを読む。
6. 再利用FeatureとGenre固有compositionは最後に[Packs](#39-packs)を読み、参照するSubsystem ownerへ戻る。

個別変更では最初から全仕様を読む必要はない。正本範囲、非正本範囲、依存linkを辿り、変更する概念のownerと直接依存だけをreviewする。

## 3. 正本一覧

各pathはこのIndex内で一度だけ掲載する。全仕様の状態は再編直後の`review`であり、文書承認状態とCapability成熟度を混同しない。

### 3.1 Product

| # | 仕様 | 状態 | 所有責務 |
|---:|---|---|---|
| 1 | [Product Plan](00-product/product-plan.md) | review | Product intent、scope、Capability portfolio、製品進行と昇格境界 |

### 3.2 Governance

| # | 仕様 | 状態 | 所有責務 |
|---:|---|---|---|
| 2 | [AI Security／Approval](01-governance/ai-security-approval.md) | review | AI authorization、risk、trust boundary、sandbox、credential、人間承認 |
| 3 | [AI Verification／Provenance](01-governance/ai-verification-provenance.md) | review | verification、evaluation、evidence、provenance、trace grading |

### 3.3 Foundation

| # | 仕様 | 状態 | 所有責務 |
|---:|---|---|---|
| 4 | [Core Architecture](02-foundation/core-architecture.md) | review | 基盤layer、host／process、状態変更境界、ownership、repository境界 |
| 5 | [Toolchain／Dependencies](02-foundation/toolchain-dependencies.md) | review | 外部tool、SDK、library、APIの採用、固定、更新根拠 |
| 6 | [Executable Contracts](02-foundation/executable-contracts.md) | review | MCD、operation、state machine、diagnostic、contract projection |
| 7 | [Naming／Project Layout](02-foundation/naming-project-layout.md) | review | 共通語彙、技術識別子、Engine／Project配置、生成物配置 |
| 8 | [C++23 Modules](02-foundation/cpp23-modules.md) | review | C++ language profile、module境界、standard-library移行 |
| 9 | [Math／Core Utilities](02-foundation/math-core.md) | review | math型、座標／単位、数値規則、core utility |
| 10 | [Memory／Pointers](02-foundation/memory-pointers.md) | review | memory ownership、pointer taxonomy、handle、allocation domain |

### 3.4 Authoring

| # | 仕様 | 状態 | 所有責務 |
|---:|---|---|---|
| 11 | [Project State](03-authoring/project-state.md) | review | Project aggregate、revision、ChangeSet、commit、undo／recovery |
| 12 | [Asset Lifecycle](03-authoring/asset-lifecycle.md) | review | Asset import、IR、Derived／Cooked artifact、catalog、package |
| 13 | [Editor UI Framework](03-authoring/editor-ui-framework.md) | review | Editor UI core、widget、layout、event、rendering、accessibility bridge |
| 14 | [Editor Workspace／UX](03-authoring/editor-workspace-ux.md) | review | Workspace、制作journey、AI partner、error／recovery UX |
| 15 | [Gameplay Programming Model](03-authoring/gameplay-programming-model.md) | review | 構造化Gameplay、Game System、Project実装、code generation |
| 16 | [Native Game Module](03-authoring/native-game-module.md) | review | Native module ABI、build、link、package、promotion evidence |

### 3.5 Runtime

| # | 仕様 | 状態 | 所有責務 |
|---:|---|---|---|
| 17 | [Scheduling／Lifetime](04-runtime/scheduling-lifetime.md) | review | tick、execution order、job dependency、message order、lifetime |
| 18 | [Performance／Capacity](04-runtime/performance-capacity.md) | review | 共通capacity、measurement、backpressure、scale、regression |
| 19 | [Debugging／Observability／Replay](04-runtime/debugging-observability-replay.md) | review | debug、telemetry、causality、capture、replay、crash evidence |

### 3.6 Simulation

| # | 仕様 | 状態 | 所有責務 |
|---:|---|---|---|
| 20 | [Collision](05-simulation/collision.md) | review | geometry、collider、filter、query、contact／trigger／hit semantics |
| 21 | [Physics](05-simulation/physics.md) | review | dynamics、body、constraint、character、kernel adapter、AI semantics |
| 22 | [Navigation](05-simulation/navigation.md) | review | 2D grid、3D navmesh、navigation query、artifact lifetime |
| 23 | [Animation](05-simulation/animation.md) | review | animation source／artifact、graph、pose、event、root motion、retarget |

### 3.7 Rendering

| # | 仕様 | 状態 | 所有責務 |
|---:|---|---|---|
| 24 | [Render Graph](06-rendering/render-graph.md) | review | renderer boundary、resource／pass graph、queue、visibility、temporal execution |
| 25 | [Materials](06-rendering/materials.md) | review | material／shader authoring、semantic intent、compile、package |
| 26 | [Project Shader](06-rendering/project-shader.md) | review | bounded HLSL、semantic module、Project Node／Shading Model、Technique、AI理解とShader qualification |
| 27 | [Lighting](06-rendering/lighting.md) | review | light source、photometry、attenuation、shadow intent、lighting semantics |
| 28 | [Post Processing](06-rendering/post-processing.md) | review | post-process source、volume、effect composition、history intent |
| 29 | [VFX Authoring](06-rendering/vfx-authoring.md) | review | VFX source、semantic catalog、typed graph、compiler、authoring operation |
| 30 | [VFX Runtime](06-rendering/vfx-runtime.md) | review | VFX artifact、instance、simulation、render、visual interaction |
| 31 | [Camera](06-rendering/camera.md) | review | Camera profile、rig、director、sequence、runtime／authoring |
| 32 | [Environment／Surfaces](06-rendering/environment-surfaces.md) | review | sky、fog、weather presentation、water、snow／wetness surface response |
| 33 | [LOD](06-rendering/lod.md) | review | representation selection、transition、fallback、LOD semantics |
| 34 | [World／Scene／Level／Cell](06-rendering/world.md) | review | World composition、partition、streaming plan、transition、map resolution |

### 3.8 Platform

| # | 仕様 | 状態 | 所有責務 |
|---:|---|---|---|
| 35 | [Windows](07-platform/windows.md) | review | Windows target、process／window adapter、package、distribution、qualification |
| 36 | [Mobile Common](07-platform/mobile-common.md) | review | Mobile共通target、port、lifecycle、resource policy、device workflow |
| 37 | [Android](07-platform/android.md) | review | Android build／runtime adapter、package、store delivery、device qualification |
| 38 | [Apple](07-platform/apple.md) | review | Apple build／runtime bridge、signing、store、device qualification |
| 39 | [Input](07-platform/input.md) | review | device、action、binding、context、remap、haptics、input replay |
| 40 | [Audio](07-platform/audio.md) | review | Audio asset semantics、cue、voice、mixer、spatial、streaming |
| 41 | [UI／Text／Localization／Accessibility](07-platform/ui-text-localization-accessibility.md) | review | Game UI、text、localization、focus、accessibility、UI authoring |

### 3.9 Packs

| # | 仕様 | 状態 | 所有責務 |
|---:|---|---|---|
| 42 | [Pack Contract](08-packs/pack-contract.md) | review | 4層Pack構造、`PackManifestV1`、dependency、install、update、removal |
| 43 | [Shooter Genre Pack](08-packs/shooter.md) | review | Shooter固有composition、Profile、Game Flow、Action role、fixture |
| 44 | [Gameplay Feature Packs](08-packs/gameplay-features.md) | review | Combat、Ranged Combat、Encounter、Scoring、Pickupのcanonical contract catalog |
| 45 | [Scenario／Stage Feature Pack](08-packs/scenario-stage.md) | review | optional Stage、completion、Scope、transition、Save／Replay |

## 4. ProductからSubsystemへのnavigation

| 調べたいこと | 最初に読む区分 | 次に辿る区分 |
|---|---|---|
| 製品scope、Capability、昇格 | [Product](#31-product) | 該当Subsystem |
| AIによる変更、安全、evidence | [Governance](#32-governance) | [Foundation](#33-foundation)、[Authoring](#34-authoring) |
| 共通型、toolchain、命名、memory | [Foundation](#33-foundation) | 利用するSubsystem |
| Project編集、Asset、Editor、Gameplay実装 | [Authoring](#34-authoring) | [Runtime](#35-runtime) |
| 実行順、capacity、debug／replay | [Runtime](#35-runtime) | [Simulation](#36-simulation)、[Rendering](#37-rendering)、[Platform](#38-platform) |
| 物理的Game挙動 | [Simulation](#36-simulation) | [Runtime](#35-runtime)、[Rendering](#37-rendering) |
| 描画、Camera、World表現 | [Rendering](#37-rendering) | [Runtime](#35-runtime)、[Platform](#38-platform) |
| OS、device、Input、Audio、Game UI | [Platform](#38-platform) | [Foundation](#33-foundation)、[Runtime](#35-runtime) |
| Genre／Feature composition | [Packs](#39-packs) | compositionが参照する全Subsystem |

## 5. Developer toolの分離

Developer tool文書は`docs/developer-tools/`にあり、このIndexのEngine正本一覧には掲載しない。文書体系上の分離理由と扱いは[文書体系再編Decision §5](decisions/2026-07-21-document-system-restructure.md#5-target構造)を参照する。PR #3との履歴成果照合と統合判断は[同Decision §12](decisions/2026-07-21-document-system-restructure.md#12-pr-3履歴成果の正本統合)を参照する。

## 6. 文書変更規則

文書変更規則の正本は[文書体系再編Decision §7](decisions/2026-07-21-document-system-restructure.md#7-正本規則)、Verification Gateの正本は[同Decision §10](decisions/2026-07-21-document-system-restructure.md#10-verification-gate)を参照する。Index更新は発見用metadataの更新にすぎず、このIndexから新しい規則を定義しない。
