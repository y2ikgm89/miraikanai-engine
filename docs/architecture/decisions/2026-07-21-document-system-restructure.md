# Miraikanai Engine 設計文書体系再編

- 文書種別: Architecture Decision／再編設計
- 状態: ユーザー承認済み
- 承認日: 2026-07-21
- 対象: `docs/superpowers/specs/`、`docs/superpowers/plans/`
- 実装方針: 後方互換性を設けない一括移行

## 1. 結論

Miraikanai Engineの現行Engine仕様47件を、責務別ディレクトリに配置した42件の正本仕様へ再編する。日付付きの旧Filename、旧Directory、redirect、互換stub、旧Indexは残さない。旧文書はGit履歴だけを移行記録とする。

Engine外の個人用Codex設定資料は`docs/developer-tools/codex/configuration.md`へ分離する。完了済み実装計画2件はactive planではないため削除し、Git履歴だけを記録とする。

新しい`docs/architecture/README.md`はIndex、読順、正本一覧、状態一覧だけを所有する。IndexはArchitectureの判断、固定値、契約、Gateを再定義しない。

## 2. 目的と完了条件

本再編の目的は次である。

1. 一つの概念、型、識別子、固定値、Gateに正本を一つだけ割り当てる。
2. Product計画、Architecture、Authoring、Runtime、Platform、Domain Packの責務を分離する。
3. 大き過ぎる複合仕様を変更理由が同じ単位へ分割する。
4. 常に同時変更される小規模仕様と、同じ状態を二重所有する仕様を統合する。
5. 外部Library、SDK、API、Toolchainの事実を公式一次資料とContext7で検証し、固定値を一か所で所有する。
6. 未決定事項をNormativeな文章へ残さず、将来機能は`not_activated`として閉じる。
7. 旧Path互換を維持するための重複文書や転送文書を作らない。

再編は、42件のEngine正本、1件のEngine Index、分離したCodex設定資料、再編Decisionだけが残り、後述する検証Gateへ全合格した時に完了する。

## 3. 現状調査

### 3.1 Inventory

調査時点の文書構成は次である。

| 区分 | 件数 | 状態 |
|---|---:|---|
| Engine仕様 | 47 | activeなReview set |
| Engine仕様Index | 1 | 47仕様の要約と正本表を重複保持 |
| 個人用Codex設定仕様 | 1 | Engine Review setから明示除外済み |
| 実装計画 | 2 | 全Task完了済み、GitへCommit済み |

Engine仕様だけで約34,000行、約2.5 MiBある。1,000行を超える仕様はMaster、2D／3D計画、Runtime、Particle／VFX、Debugging、Foundation、Mobile、Collision、Camera、Capability Portfolio、Shooterである。

### 3.2 重複と参照構造

文字4-gramによる構造比較で、特に次の組が高い類似度を示した。

| 文書組 | 類似度 | 判断 |
|---|---:|---|
| Master／README | 0.860 | IndexがArchitecture判断とRoadmapを再定義している |
| 2D・3D計画／README | 0.829 | Capability概要、Phase、Gateが重複している |
| 2D・3D計画／Rendering | 0.820 | AA、Render経路、Visual Effect詳細が重複している |
| Master／2D・3D計画 | 0.812 | Product scopeとSubsystem要約が重複している |
| AI Governance／Verification | 0.805 | Approval、Evidence、Audit境界が交差している |
| 2D・3D計画／Runtime | 0.802 | Tick、Budget、実行順、Qualificationが重複している |
| Foundation／Runtime | 0.781 | Phase 0、Memory、Toolchain、Lifecycle Gateが重複している |
| Asset Pipeline／Asset Import | 0.753 | Import ownershipが二重化している |
| Scale／World | 0.749 | Cell、Streaming、Budget envelopeが重複している |
| Domain Pack／Capability Portfolio | 0.738 | 将来RoadmapとActivationが重複している |
| Physics Semantic Catalog／Physics | 0.727 | Physics CapabilityとAI意味契約が分離し過ぎている |

参照GraphではRuntime、Executable Contracts、Master、2D／3D計画、Foundationが多くの文書から参照される一方、ScaleとCapability PortfolioのIncoming linkは各1件であった。後から追加された横断文書が既存の正本Graphへ統合されず、同等の規範文書として並立している。

### 3.3 根本原因

1. Master、Index、Capability計画が、一覧ではなくSubsystem本文を再記載している。
2. 文書の状態とCapabilityの実装成熟度が同じ「状態」Fieldへ混在している。
3. 固定Toolchain、Budget、Phase、Gateが参照ではなく複写されている。
4. 「横断的である」ことを理由に、既存のOwnerへ配分すべき内容が新しい正本として追加されている。
5. 日付付きFilenameが恒久識別子として使われ、文書責務より作成順を優先している。
6. 完了済みPlanがactive plan Directoryへ残り、現在作業と履歴を区別できない。

## 4. 比較した案

| 案 | 内容 | 採否 |
|---|---|---|
| A. 最小整理 | 47仕様を維持し、重複記述とLinkだけを修正する | 不採用。多重正本と巨大文書が残る |
| B. 階層型正本 | 単一責務の42仕様へ統廃合・分割し、責務Directoryへ配置する | 採用。正本一意性、局所変更、読順を両立する |
| C. 少数の大冊 | 8～12冊へ統合する | 不採用。部分Review、変更競合、AIのbounded retrievalが悪化する |

## 5. Target構造

ActiveなEngine仕様を次へ固定する。

```text
docs/architecture/
├── README.md
├── decisions/
│   └── 2026-07-21-document-system-restructure.md
├── 00-product/
│   └── product-plan.md
├── 01-governance/
│   ├── ai-security-approval.md
│   └── ai-verification-provenance.md
├── 02-foundation/
│   ├── core-architecture.md
│   ├── toolchain-dependencies.md
│   ├── executable-contracts.md
│   ├── naming-project-layout.md
│   ├── cpp23-modules.md
│   ├── math-core.md
│   └── memory-pointers.md
├── 03-authoring/
│   ├── project-state.md
│   ├── asset-lifecycle.md
│   ├── editor-ui-framework.md
│   ├── editor-workspace-ux.md
│   ├── gameplay-programming-model.md
│   └── native-game-module.md
├── 04-runtime/
│   ├── scheduling-lifetime.md
│   ├── performance-capacity.md
│   └── debugging-observability-replay.md
├── 05-simulation/
│   ├── collision.md
│   ├── physics.md
│   ├── navigation.md
│   └── animation.md
├── 06-rendering/
│   ├── render-graph.md
│   ├── materials.md
│   ├── lighting.md
│   ├── post-processing.md
│   ├── vfx-authoring.md
│   ├── vfx-runtime.md
│   ├── camera.md
│   ├── environment-surfaces.md
│   ├── lod.md
│   └── world.md
├── 07-platform/
│   ├── windows.md
│   ├── mobile-common.md
│   ├── android.md
│   ├── apple.md
│   ├── input.md
│   ├── audio.md
│   └── ui-text-localization-accessibility.md
└── 08-domain-packs/
    ├── domain-pack-contract.md
    └── shooter.md
```

`docs/developer-tools/codex/configuration.md`はEngine Architecture外に置く。

## 6. 完全移行表

### 6.1 ProductとGovernance

| 旧文書 | 新しい正本 | 移行規則 |
|---|---|---|
| `2026-07-18-ai-native-game-engine-authoring-design.md` | `00-product/product-plan.md` | Vision、Product原則、Scope、MVPだけを残す。Subsystem詳細は各Ownerへ移す |
| `2026-07-19-2d-3d-capability-plan.md` | `00-product/product-plan.md`＋各Subsystem Owner | Capability maturityとPhaseだけをProductへ置き、座標、Budget、Schema、Backend詳細はOwnerへ移す |
| `2026-07-20-ai-readable-capability-portfolio-productization-roadmap-design.md` | `00-product/product-plan.md` | Portfolio、Activation、製品化Roadmapを統合する |
| `2026-07-19-domain-pack-future-capability-roadmap.md` | `08-domain-packs/domain-pack-contract.md`＋`00-product/product-plan.md` | Pack形式とActivation契約を前者、将来Capability一覧を後者へ置く |
| `2026-07-19-ai-engine-development-governance-design.md` | `01-governance/ai-security-approval.md` | Trust、Risk、Authorization、Sandbox、Provider／MCP境界を所有する |
| `2026-07-21-immutable-engine-beginner-ai-approval-design.md` | `01-governance/ai-security-approval.md` | 不変Engine境界、初心者向け質問、Approval、Activationを統合する |
| `2026-07-19-ai-verification-evaluation-provenance-design.md` | `01-governance/ai-verification-provenance.md` | Eval、Evidence、Provenance、Trace gradingを所有する |

### 6.2 Foundation

| 旧文書 | 新しい正本 | 移行規則 |
|---|---|---|
| `2026-07-19-engine-foundation-architecture-design.md` | `02-foundation/core-architecture.md`＋`02-foundation/toolchain-dependencies.md` | Layer、Host、Build Gateway、Repository boundaryと、Version／Hash／License／Upgrade Gateを分離する |
| `2026-07-19-executable-contract-schema-codegen-design.md` | `02-foundation/executable-contracts.md` | MCD、Schema、Codegen、Provider projectionを所有する |
| `2026-07-20-ai-readable-engine-naming-convention-design.md` | `02-foundation/naming-project-layout.md` | 技術識別子規則を統合する |
| `2026-07-20-ai-readable-game-project-layout-naming-design.md` | `02-foundation/naming-project-layout.md` | Engine／Project Path、Filename、生成物配置を統合する |
| `2026-07-20-cpp23-modules-import-std-transition-design.md` | `02-foundation/cpp23-modules.md` | C++23、Named Modules、`import std` Cutoverだけを所有する |
| `2026-07-20-ai-readable-math-core-utilities-architecture-design.md` | `02-foundation/math-core.md` | 一対一移行 |
| `2026-07-20-ai-readable-memory-pointer-architecture-design.md` | `02-foundation/memory-pointers.md` | 一対一移行 |

### 6.3 Authoring

| 旧文書 | 新しい正本 | 移行規則 |
|---|---|---|
| `2026-07-19-authoring-model-project-state-design.md` | `03-authoring/project-state.md` | Revision、ChangeSet、Commit、Undo、Source／Derivedを所有する |
| `2026-07-19-asset-pipeline-content-packaging-design.md` | `03-authoring/asset-lifecycle.md` | Cook、Package、Content addressingを統合する |
| `2026-07-20-asset-import-ai-authoring-editor-ux-design.md` | `03-authoring/asset-lifecycle.md` | Import、Reimport、Preview、Import Operationを統合する |
| `2026-07-20-editor-ui-framework-architecture-design.md` | `03-authoring/editor-ui-framework.md` | Widget、Layout、Shell、Rendering、Accessibility bridgeを所有する |
| `2026-07-19-editor-workspace-ux-design.md` | `03-authoring/editor-workspace-ux.md` | Workspace、Journey、Error UX、Beginner／Expert projectionを所有する |
| `2026-07-19-cpp-structured-game-data-design.md` | `03-authoring/gameplay-programming-model.md` | `GameplayDefinition`とC++選択境界を統合する |
| `2026-07-20-game-system-ai-codegen-architecture-design.md` | `03-authoring/gameplay-programming-model.md` | `GameSystemSpecV1`、AI Codegen、生成Bundleを統合する |
| `2026-07-19-native-game-module-architecture-design.md` | `03-authoring/native-game-module.md` | ABI、Build、Promotion、Packagingだけを所有する |

### 6.4 RuntimeとSimulation

| 旧文書 | 新しい正本 | 移行規則 |
|---|---|---|
| `2026-07-19-runtime-integration-lifetime-performance-design.md` | `04-runtime/scheduling-lifetime.md`＋`04-runtime/performance-capacity.md` | Tick／DAG／LifetimeとBudget／Measurement／Capacityを分離する |
| `2026-07-20-scale-resilient-canonical-architecture-design.md` | `04-runtime/performance-capacity.md`＋各Owner | Envelope、Backpressure、Scale GateをPerformanceへ置き、World／LOD固有値はOwnerへ戻す |
| `2026-07-20-ai-readable-debugging-observability-replay-architecture-design.md` | `04-runtime/debugging-observability-replay.md` | Debug surface、Telemetry、Replay、Crash evidenceを所有する。共通Evidence形式はGovernanceを参照する |
| `2026-07-19-collision-collider-architecture-design.md` | `05-simulation/collision.md` | Geometry、Collider Asset、Query、Filter、Eventを所有する |
| `2026-07-20-physics-engine-architecture-design.md` | `05-simulation/physics.md` | Dynamics、World、Body、Constraint、Character、Kernel Adapterを所有する |
| `2026-07-20-physics-ai-semantic-capability-catalog-design.md` | `05-simulation/physics.md` | PhysicsのAI Intent、Capability discovery、Semantic validationを統合する |
| `2026-07-20-navigation-platform-architecture-design.md` | `05-simulation/navigation.md` | 一対一移行 |
| `2026-07-19-physics-navigation-animation-architecture-design.md` | `05-simulation/animation.md`＋`04-runtime/scheduling-lifetime.md` | Animation Asset／Graph／Pose／Retargetを独立し、Physics／Navigationとの実行順だけをRuntimeへ移す |

### 6.5 Rendering

| 旧文書 | 新しい正本 | 移行規則 |
|---|---|---|
| `2026-07-19-rendering-render-graph-architecture-design.md` | `06-rendering/render-graph.md` | Renderer、Render Graph、Resource、AA実行経路を所有する |
| `2026-07-20-material-visual-style-ai-authoring-architecture-design.md` | `06-rendering/materials.md` | 一対一移行 |
| `2026-07-20-lighting-ai-authoring-architecture-design.md` | `06-rendering/lighting.md` | 一対一移行 |
| `2026-07-20-post-process-ai-authoring-architecture-design.md` | `06-rendering/post-processing.md` | 一対一移行 |
| `2026-07-20-particle-vfx-architecture-design.md` | `06-rendering/vfx-authoring.md`＋`06-rendering/vfx-runtime.md` | Source／Graph／Compiler／AI OperationとSimulation／Render／Budget／Qualificationを分離する |
| `2026-07-20-camera-platform-ai-authoring-virtual-production-architecture-design.md` | `06-rendering/camera.md`＋`00-product/product-plan.md` | C1 Camera Runtime／Authoringを残す。Virtual Productionは`not_activated` Portfolio entryだけに戻す |
| `2026-07-20-environment-platform-ai-authoring-architecture-design.md` | `06-rendering/environment-surfaces.md` | Environment Source、Runtime compilation、AI Operationを統合する |
| `2026-07-20-water-surface-platform-architecture-design.md` | `06-rendering/environment-surfaces.md` | Water profile、surface interactionを統合する |
| `2026-07-20-weather-snow-surface-architecture-design.md` | `06-rendering/environment-surfaces.md` | Weather、snow、surface stateを統合する |
| `2026-07-20-ai-readable-lod-architecture-design.md` | `06-rendering/lod.md` | 一対一移行 |
| `2026-07-20-world-level-map-ai-authoring-architecture-design.md` | `06-rendering/world.md` | 一対一移行。Streaming capacityはRuntimeを参照する |

### 6.6 PlatformとDomain Pack

| 旧文書 | 新しい正本 | 移行規則 |
|---|---|---|
| `2026-07-19-windows-platform-distribution-design.md` | `07-platform/windows.md` | 一対一移行。Tool versionはFoundationを参照する |
| `2026-07-19-mobile-platform-architecture-design.md` | `07-platform/mobile-common.md`＋`07-platform/android.md`＋`07-platform/apple.md` | 共通Lifecycle／Budget、Android Build／Runtime／Store、Apple Build／Signing／Runtimeへ分割する |
| `2026-07-19-input-action-device-architecture-design.md` | `07-platform/input.md` | 一対一移行 |
| `2026-07-19-audio-mixer-spatial-architecture-design.md` | `07-platform/audio.md` | 一対一移行 |
| `2026-07-19-ui-text-localization-accessibility-design.md` | `07-platform/ui-text-localization-accessibility.md` | 一対一移行 |
| `2026-07-20-ai-readable-shooter-gameplay-architecture-design.md` | `08-domain-packs/shooter.md` | Engine正本ではなくDomain Pack referenceとして配置する |

### 6.7 Index、Engine外資料、完了Plan

| 旧文書 | 処理 |
|---|---|
| `docs/superpowers/specs/README.md` | `docs/architecture/README.md`へ置換する。旧本文は移植せず、正本一覧と読順を再生成する |
| `2026-07-18-codex-config-optimization-design.md` | `docs/developer-tools/codex/configuration.md`へ分離する。Engine文書から参照しない |
| `docs/superpowers/plans/2026-07-18-codex-config-optimization.md` | 削除する。完了記録はGit履歴だけとする |
| `docs/superpowers/plans/2026-07-20-existing-documentation-naming-directory-unification.md` | 削除する。完了記録はGit履歴だけとする |

## 7. 正本規則

### 7.1 Header

全Active仕様は冒頭に次のFieldを同じ順序で持つ。

```text
- 文書ID: 一意で不変なASCII ID
- 状態: review | normative
- 正本範囲: 本文書だけが決定する事項
- 非正本範囲: 参照先と、本文書が決定してはならない事項
- 依存: 相対Linkの一覧
- 外部根拠検証日: YYYY-MM-DD
```

再編直後は、再編により本文が変わるため全仕様を`review`とする。文書の承認状態とCapability maturityを混同しない。Capability maturityは各仕様内の`not_activated | candidate_locked | qualified | production`で表し、文書Headerへ書かない。

### 7.2 一意所有

1. 型、Schema、Operation、Diagnostic、Stable IDの構造定義は`executable-contracts.md`または明示されたDomain Ownerだけが所有する。
2. Tool、SDK、LibraryのVersion、commit、artifact size、hash、license、取得元は`toolchain-dependencies.md`だけが所有する。
3. Product scope、Capability maturityの意味、Phase順序、Roadmapは`product-plan.md`だけが所有する。
4. Runtime phase、tick、job dependency、lifetimeは`scheduling-lifetime.md`だけが所有する。
5. 共通Budgetの定義、測定法、capacity envelope、backpressureは`performance-capacity.md`だけが所有する。Subsystem固有BudgetはSubsystem Ownerが所有する。
6. AI authorization、Approval、Sandbox、Credential、MCP securityは`ai-security-approval.md`だけが所有する。
7. Evidence envelope、Eval、Provenance、Trace gradingは`ai-verification-provenance.md`だけが所有する。
8. Index、Product Plan、Domain PackはSubsystemのField、数値、APIを複写しない。

他文書は正本名と相対Linkを示し、同じ表、Field一覧、既定値、Gateを再掲しない。読者向けの一文要約は許可するが、Normativeな値を含めない。

### 7.3 外部事実とプロジェクト判断

外部事実には次を必須とする。

- exact Versionまたは参照したRelease。
- 公式Project、公式Vendor、標準化団体、原論文の一次資料。
- APIやLibrary挙動はContext7で対象Library IDとVersionを先に確認する。
- Context7に対象またはVersionがない場合だけ公式一次資料へFallbackし、その事実をReference noteへ記録する。
- 時点依存の「推奨」「最新」はArchitecture定数にしない。検証日と、Toolchain更新手順を記録する。
- 公式の事実とMiraikanaiの採用判断を別文で表す。

比較対象EngineのDocumentationは設計比較のEvidenceに使えるが、MiraikanaiのContract正本にはしない。Blog、Forum、二次記事は、公式資料や原論文で確認できないNormative判断の根拠にしない。

## 8. 公式資料・Context7監査結果

再編設計時点で、次の主要前提を確認した。

| 対象 | 確認結果 | 根拠 |
|---|---|---|
| CMake 4.4.0 | Release tagとWindows artifact SHA-256が一致。C++ Module scanはNinja、Ninja Multi-Config、VS 2022／2026を扱う。`import std`はExperimental gateが必要で、Ninja系だけが対応 | [CMake 4.4.0](https://github.com/Kitware/CMake/releases/tag/v4.4.0)、Context7 `/kitware/cmake` |
| MSVC 14.51 | Stable GA。C++23未完項目はP2564R3とP0533R9で、正式`/std:c++23`は14.52以降のStableを待つ方針が妥当 | [MSVC 14.51 GA](https://devblogs.microsoft.com/cppblog/msvc-version-1451-available/)、[C++23 status](https://devblogs.microsoft.com/cppblog/c23-support-in-msvc-build-tools-14-51/) |
| Box2D 3.1.1 | Releaseとcommit `8c661469...`が一致。C17、ID handle、固定step／substep、multithread Adapter前提が妥当 | [Box2D 3.1.1](https://github.com/erincatto/box2d/releases/tag/v3.1.1)、Context7 `/erincatto/box2d` |
| Jolt 5.6.0 | Releaseとcommit `e77f175...`が一致。GPU computeとhairは追加機能で、`JPH_USE_DX12`等をOFFにできる | [Jolt 5.6.0](https://github.com/jrouwe/JoltPhysics/releases/tag/v5.6.0)、Context7 `/jrouwe/joltphysics` |
| Recast 1.6.0 | Annotated tagがcommit `6dc1667...`へ解決する。Recast build／Detour queryのprivate Backend化が妥当 | [Recast 1.6.0](https://github.com/recastnavigation/recastnavigation/releases/tag/v1.6.0)、Context7 `/recastnavigation/recastnavigation` |
| Node.js／TypeScript | Node.js 24.18.0 LTSとTypeScript 7.0.2が正式Release。TypeScript 7.0はProgrammatic APIを出していないためCLI限定方針が妥当 | [Node.js 24.18.0](https://nodejs.org/en/blog/release/v24.18.0)、[TypeScript 7.0](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/) |
| OpenAI | 新規ProjectはResponses APIが公式推奨。Function Callingは`strict: true`が推奨で、全Property required、各Objectの`additionalProperties: false`が必要 | [Responses API migration](https://developers.openai.com/api/docs/guides/migrate-to-responses)、[Strict mode](https://developers.openai.com/api/docs/guides/function-calling#strict-mode) |
| Android | AGP 9.3.0の最小／既定Gradleは9.5.0、Build Toolsは36.0.0、JDKは17。Context7に公式AGP 9.3資料がなかったためAndroid公式へFallbackした | [AGP 9.3.0](https://developer.android.com/build/releases/agp-9-3-0-release-notes)、[NDK history](https://developer.android.com/ndk/downloads/revision_history) |
| Apple | Xcode 26.6はmacOS Tahoe 26.2以降を要求し、iOS／iPadOS 26.5 SDKを含む。Xcode Cloudはephemeral環境を使用する | [Xcode 26.6](https://developer.apple.com/documentation/Xcode-Release-Notes/xcode-26_6-release-notes)、[Xcode Cloud security](https://developer.apple.com/xcode-cloud/security/) |

全676外部URLを新文書へ機械的に移植しない。各OwnerがNormative判断を支える最小の一次資料だけを残し、同じSourceを複数文書で説明しない。

## 9. 実装順序

本Decisionのユーザーレビュー後、別のImplementation Planを作成して次の順に実行する。

1. Target Directoryと共通Headerを作る。
2. Product／Governanceを統合し、PhaseとApprovalの正本を一意化する。
3. Foundation／Authoringを移行し、Toolchainと外部Versionを一か所へ集約する。
4. Runtime／Simulationを移行し、ScaleとAnimationの責務を再配分する。
5. Renderingを移行し、VFXを分割、Environment／Water／Weatherを統合する。
6. Platformを移行し、MobileをCommon／Android／Appleへ分割する。
7. Domain PackとShooter referenceを移行する。
8. Engine Indexを新規作成し、Codex設定をEngine外へ移す。
9. 旧仕様、旧Index、完了Planを削除する。
10. 全Gateを実行し、失敗を修正する。

中間状態では旧文書と新文書が同時に存在し得るが、同じCommit seriesの最終状態で旧文書を全削除する。新文書から旧Pathを参照しない。

## 10. Verification Gate

### 10.1 InventoryとPath

- `docs/architecture/`にEngine正本42件とIndexが存在する。
- Active仕様のFilenameに作成日を含めない。
- `docs/superpowers/specs/`と`docs/superpowers/plans/`にactive文書を残さない。
- `docs/developer-tools/codex/configuration.md`がEngine IndexまたはEngine正本から参照されない。
- 旧Filename、旧Directory、redirect、compatibility stubへの参照が0件である。

### 10.2 MarkdownとGraph

- 全相対LinkのTargetが存在し、Anchorが存在する。
- 文書IDが一意である。
- 全仕様が必須Header Fieldを持つ。
- Indexから42正本へ到達でき、孤立した正本が0件である。
- 正本間の循環参照は許可するが、Ownershipの循環は許可しない。
- H1は各Fileに1件だけで、Heading levelを飛ばさない。

### 10.3 正本一意性

- Toolchain Version／Hashを`toolchain-dependencies.md`以外で再定義しない。
- Product PhaseとCapability maturity定義を`product-plan.md`以外で再定義しない。
- Runtime phase／tickを`scheduling-lifetime.md`以外で再定義しない。
- 共通Budgetを`performance-capacity.md`以外で再定義しない。
- Approval／Authorizationを`ai-security-approval.md`以外で再定義しない。
- Evidence／Provenance envelopeを`ai-verification-provenance.md`以外で再定義しない。
- `StableId`、`ProjectChangeSetV1`、`GameSystemSpecV1`等の共有Contractは定義箇所が1件で、他はLink参照だけである。

### 10.4 重複と大きさ

- 120文字以上の同一Paragraph重複が0件である。共通Header labelは除外する。
- 文字4-gram類似度0.70以上の文書組は手動Reviewし、責務上必要な類似だけを記録する。
- 1,000行を超えるActive仕様が0件である。
- IndexはSubsystemの数値表、Schema、Phase task、DoDを複写しない。

### 10.5 曖昧さと外部根拠

- Normativeな未解決marker、`要検討`、Ownerなしの`未定`が0件である。
- 将来機能は`not_activated`、Activation条件、禁止される現在動作を明記する。
- Project固有の値を外部Vendorの推奨として表現しない。
- 外部Version、API挙動、Platform要件は公式一次資料またはContext7の対象Versionで検証できる。
- 到達不能URL、検索結果URL、非公式MirrorをNormative根拠に使わない。
- Context7に対象VersionがないFallbackはReference noteに記録する。

### 10.6 Gitと最終Review

- 作業開始前から存在したUser変更を変更または削除していない。
- Final diffに文書再編以外の変更がない。
- `git diff --check`が成功する。
- Link、Inventory、Heading、Duplicate、Authority、Placeholder、External Referenceの各検証結果を保存または最終報告へ記載する。

## 11. リスクと対策

| リスク | 対策 |
|---|---|
| 統合時に固有要件を落とす | 全旧文書を本Decisionの移行表へ対応させ、旧H2単位の移行先Coverageを検証する |
| 分割後に同じ値を再複写する | 正本一意性Gateと共有Contract定義箇所検査を必須にする |
| 大規模RenameでLinkが壊れる | 旧文書削除前と削除後に全Markdown Link／Anchorを検査する |
| 公式資料が更新される | exact Versionと検証日を固定し、floatingな「最新」を削除する |
| 将来Capabilityの詳細が現行契約化する | `not_activated` entryだけをProduct Planへ残し、Activation前の実装Schemaを置かない |
| Codex個人設定がEngine判断へ混入する | Engine外Directoryへ分離し、Engine GraphからLinkしない |

## 12. 未解決事項

なし。Target構造、全旧文書の移行先、削除対象、正本規則、後方互換性を設けない方針、検証Gateは本Decisionで確定している。
