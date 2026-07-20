# AIネイティブ独自ゲームエンジン 設計計画書

- 文書版: 2.8
- 作成日: 2026-07-18
- 最終更新日: 2026-07-21
- 対象: 独自C++ゲームエンジン、独自Editor、AI制作基盤
- 状態: 基本構想とSubsystem別正式仕様47文書を統合した内部整合レビュー版。ユーザー承認待ち
- 設計文書Index: [Miraikanai Engine 設計文書Index](./README.md)
- Game実装規約: [Miraikanai Engine C++実行コード・構造化ゲームデータ規約](./2026-07-19-cpp-structured-game-data-design.md)
- C++言語・Modules規約: [Miraikanai Engine C++23・Named Modules・`import std`移行規約](./2026-07-20-cpp23-modules-import-std-transition-design.md)
- Math／Core Utilities規約: [Miraikanai Engine AI可読Math／Core Utilitiesアーキテクチャ規約](./2026-07-20-ai-readable-math-core-utilities-architecture-design.md)
- Authoring状態規約: [Miraikanai Engine Authoring Model／Project State規約](./2026-07-19-authoring-model-project-state-design.md)
- Game System規約: [Miraikanai Engine Game System／AI Code Generationアーキテクチャ規約](./2026-07-20-game-system-ai-codegen-architecture-design.md)
- Game制作時のEngine不変境界・初心者承認: [Miraikanai Engine 不変Engine境界・初心者向けAI技術承認規約](./2026-07-21-immutable-engine-beginner-ai-approval-design.md)
- World／Level／Map規約: [Miraikanai Engine World／Level／Map／AI Authoringアーキテクチャ規約](./2026-07-20-world-level-map-ai-authoring-architecture-design.md)
- Native Game規約: [Miraikanai Engine NativeGameModuleアーキテクチャ規約](./2026-07-19-native-game-module-architecture-design.md)
- Renderer規約: [Miraikanai Engine Rendering／Render Graphアーキテクチャ規約](./2026-07-19-rendering-render-graph-architecture-design.md)
- LOD規約: [Miraikanai Engine AI可読LODアーキテクチャ規約](./2026-07-20-ai-readable-lod-architecture-design.md)
- Particle／VFX規約: [Miraikanai Engine 独自Particle／VFX Platformアーキテクチャ規約](./2026-07-20-particle-vfx-architecture-design.md)
- Asset規約: [Miraikanai Engine Asset Pipeline／Content Package規約](./2026-07-19-asset-pipeline-content-packaging-design.md)
- Asset Import／AI／Editor規約: [Miraikanai Engine Asset Import／AI Authoring／Editor UXアーキテクチャ規約](./2026-07-20-asset-import-ai-authoring-editor-ux-design.md)
- Debugging規約: [Miraikanai Engine AI可読Debugging／Observability／Replayアーキテクチャ規約](./2026-07-20-ai-readable-debugging-observability-replay-architecture-design.md)
- Editor規約: [Miraikanai Engine Editor／Workspace／UX規約](./2026-07-19-editor-workspace-ux-design.md)
- Editor UI Framework規約: [Miraikanai Engine 独自Editor UI Framework／Shellアーキテクチャ規約](./2026-07-20-editor-ui-framework-architecture-design.md)
- Player I/O規約: [Input](./2026-07-19-input-action-device-architecture-design.md)／[UI・Text](./2026-07-19-ui-text-localization-accessibility-design.md)／[Audio](./2026-07-19-audio-mixer-spatial-architecture-design.md)
- Physics Engine規約: [Miraikanai Engine 独自Physics Platform／Dynamicsアーキテクチャ規約](./2026-07-20-physics-engine-architecture-design.md)
- Navigation規約: [Miraikanai Engine 独自Navigation Platformアーキテクチャ規約](./2026-07-20-navigation-platform-architecture-design.md)
- Simulation連携規約: [Physics／Navigation／Animation連携](./2026-07-19-physics-navigation-animation-architecture-design.md)
- Platform規約: [Windows](./2026-07-19-windows-platform-distribution-design.md)／[Mobile](./2026-07-19-mobile-platform-architecture-design.md)
- 将来Capability規約: [Miraikanai Engine Domain Pack／将来Capability規約](./2026-07-19-domain-pack-future-capability-roadmap.md)
- Shooter Gameplay規約: [Miraikanai Engine AI可読Shooter Gameplay／Weapon／Projectileアーキテクチャ規約](./2026-07-20-ai-readable-shooter-gameplay-architecture-design.md)
- Collision詳細規約: [Miraikanai Engine Collision／Colliderアーキテクチャ規約](./2026-07-19-collision-collider-architecture-design.md)
- AI実装・保守規約: [Miraikanai Engine AI実装・保守ガバナンス規約](./2026-07-19-ai-engine-development-governance-design.md)
- 実行可能契約規約: [Miraikanai Engine 実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)
- AI検証規約: [Miraikanai Engine AI検証・評価・来歴規約](./2026-07-19-ai-verification-evaluation-provenance-design.md)

## 0. 統合レビュー結果

本書は、これまで個別に確定した要求を一つのProduct計画として束ねるマスター計画書である。詳細契約はSubsystem別正式仕様へ分離し、本書と各仕様を一つのReview setとして扱う。47文書を単一巨大文書へ統合せず、次の三層を維持する。

1. 本書がProduct vision、制作体験、Milestone、MVP、全体の依存順を決定する。
2. [設計文書Index](./README.md)が読む順序、決定権、要求トレーサビリティ、Review状態を決定する。
3. Subsystem別正式仕様が型、phase、lifetime、budget、failure、test、Definition of Doneを決定する。

このReview setにはEngine製品自体の将来開発も含まれるが、Game制作は`GameAuthoringProfileV1`で実行する。Game制作中のEngine、Editor、公開SDK、Validator、Policyは署名済みread-only baselineであり、Engine製品開発のRepository、Tool、A2／R4権限をGame制作Taskへ公開または継承しない。詳細と矛盾時の優先規則は[不変Engine境界・初心者向けAI技術承認規約](./2026-07-21-immutable-engine-beginner-ai-approval-design.md)を正本とする。

2026-07-20の内部整合レビューでは、会話で確定した要求が次の閉じた設計へ収束していることを確認した。

| 領域 | 統合判断 |
|---|---|
| AIとEngineの境界 | Game制作AIはGame Brief、GameSpec、GameplayDefinition、bounded Project C++、Test、ProjectChangeSetを提案できるが、署名済みEngine baseline、Engine object、pointer、GPU resource、ProjectRevisionを直接変更できない。公開SDKで実現不能なら`capability_unavailable`で停止する |
| Authoring | 自然言語、Editor GUI、外部IDE、MCPは同じAuthoring DocumentとProjectChangeSetへ収束し、C++ GatewayだけがCommitする |
| Game実装 | C++23実行Code＋構造化GameplayDefinition。Luauを含む汎用Game Script VMは持たない |
| Game System | 契約固定・実装開放型。Engine標準、Project定義、署名済みBaselineに含まれるread-only Extensionを同じSystem Contract、State owner、Catalog、Bundle、Testへ収束させる。Game制作AIがExtensionを生成・変更する経路は持たない |
| World／Level／Map | `Map`をWorld structure、playable Level、Streaming、Procedural layout、Navigation、Map presentationへ解決する。Scene edit shard、Gameplay Level、Target別Streaming Cellを別identityとする |
| Editor | C++23の独自`MirakanUi Core`＋`MirakanEditor Shell`で実装するProjection Editor。Retained UIと限定typed Immediate Canvasを併用し、禁止GUI toolkit、screen-coordinate AI操作、Projectへの直接writeを持たない |
| Game UI／AI UI | HUDと画面UIは独立した`UiDocument`として所有し、手動操作とAI提案を同じ型付きChangeSetへ収束させる。C1は標準Widget＋`UiCompositeDefinition`、C2は`UiEffectGraph`＋承認済み`UiNativeWidget`をGameHostで扱い、AIによるWidget／生成Assetの直接writeを許可しない |
| AI統合 | 内蔵はProvider API、外部Hostはlocal MCP、Pluginは任意UX。Source変更は隔離WorkerとPromotionを必須にする |
| Engine機能 | 2D／3D、Renderer、Asset、Collision、Physics、Navigation、Animation、Input、UI／Text、Audio、VFX、環境表現をSubsystem契約として分離する |
| Physics | 独自World／Body／Joint／Character／Command／Save／AI契約をC++23で所有し、Box2D 3.1.1／Jolt 5.6.0をprivate kernel候補としてTarget別Qualification後にProduction昇格する |
| Particle／VFX | 単一の型付きVFX Asset／Graph IRを2D／3D・CPU／GPU専用Artifactへoffline compileする。初心者Stack、上級者Graph、AI編集は同じSourceのProjectionとし、Particle結果をauthoritative Gameplayへ逆入力しない |
| LOD | 共通`LodIntentV1`、Domain別Policy、Target別`LodResolutionPlanV1`、実測Receiptへ分離する。Mesh／Sprite、HLOD、Simulation、Animation、Material、VFX、Terrain／Foliage／Water／Snow Surface、Geometry residencyを同じ万能indexへ混ぜず、AIは意味、Engineは決定論的解決とfidelity Gateを所有する |
| 大量制作／自動最適化 | AIが大量配置、敵味方spawn、同時VFXをScale intentとGameplay fidelity floorへ分解し、Target別Representation PlanへCookする。固定個数だけで制作意図を捨てず、Gameplayを黙って削らず、統合負荷fixtureの実測で合否を決める |
| Visual表現 | Realistic、Toon、2D、独自Pixel DioramaをVisual Style、Material、Shader、Quality Profileの組合せとして解決する |
| Platform | Windows／D3D12を先行し、Android／Vulkan、Apple／Metalを同じPortへ接続する |
| Build | CMakeがC++ Build定義、NinjaがC++ Build executor、Build Gatewayが製品入口。Android packageはGradle、Apple app／archive／署名はXcodeが所有する |
| C++進化 | C++23をShipping基準とし、Named Modules＋`import std`へ一方向移行する。C++26はreadiness CIで先行検証する |
| 実装順 | Foundation→Headless Authoring→Editor／Runtime→2D Manual→AI MVP-A→外部Agent→3D MVP-B→Mobile→Production→制限付きRuntime生成。各段階は契約→最小縦切り→実測→ボトルネック最適化→回帰Gate→Capability昇格の順で閉じる |

本レビューでは、Ninjaを含むBuild層の責務に加え、大量制作を固定capだけで拒否しないAI Scale Planningを正式化した。Source intentの保持とTarget RuntimeのQualificationを分離し、Presentation-only最適化は自動化できるが、敵味方数、Damage、collision、goal、spawn timingの変更は人間承認を必須にした。

## 1. エグゼクティブサマリー

本プロジェクトは、既存ゲームエンジンへAI機能を追加するものではない。人間とAIが同じ安全な編集経路を利用し、C++エンジンがすべての変更を検証・試行・確定する、独自のAIネイティブゲームエンジンとEditorを開発する。

中心となる制作体験は次のとおりである。

1. ユーザーが大まかな自然言語プロンプトを入力する。
2. AIがゲーム制作に重要な不足要件だけを質問する。
3. AIが理解したゲーム概要と補完事項をGame Briefとして提示する。
4. AIがゲーム全体のGameSpecと実装計画を作成する。
5. AIが承認済みVisual Style、Engine Capability、Behavior Budgetに従い、GameplayDefinitionまたはC++をシステム単位で選択し、必要なら型付き境界で併用する。
6. C++エンジンが生成結果を検証し、短縮されたゲーム全体と代表的な完成部分を持つFirst Playableを生成する。
7. ユーザーはAIとの会話、Scene／Graph／Inspector／Definition Table、C++のいずれからでも調整できる。
8. すべての変更はChangeSetとして記録され、Diff、Dry-run、Test、Undo、監査を経由する。

AIはC++エンジン内部を直接操作しない。AI、Editor GUI、外部ツールはすべて変更提案の作成者であり、最終的なProject状態変更権限はC++ `AuthoringCommandGateway`だけが持つ。

## 2. プロダクトビジョン

### 2.1 目標

ゲーム開発経験のないユーザーから上級者まで、同じプロジェクト形式と同じEditorを利用できる制作環境を実現する。

- 初心者は自然言語だけで制作を開始できる。
- AIは不足要件を適応的に質問し、設計と実装を補助する。
- AIはScene、UI、ゲームルール、Asset設定、GameplayDefinition、C++、Testを生成・修正できる。
- 経験者はビジュアルEditor、GameplayDefinition、C++を直接編集できる。
- AIの編集と人間の編集を双方向に往復できる。
- 2D、3D、FPS、TPS、RPG、Action RPG、SimulationなどへDomain Pack方式で拡張できる。
- Editor制作型を先に完成させ、成熟した検証基盤を制限付きRuntime生成へ再利用する。

### 2.2 中核となる価値

本エンジンが提供する価値は、単なるコード生成ではない。

- 自然言語からゲーム全体の構造へ展開できること
- ゲームの設計判断と実装判断を分離できること
- AIが行った仮定と変更理由を追跡できること
- 生成されたゲームを人間が自由に引き継げること
- 人間の手動変更をAIが尊重できること
- 生成物を再生成するのではなく、承認済み変更を決定論的に再生できること
- 初心者向けの簡単さと、C++まで到達できる自由度を同時に提供すること

### 2.3 対象外とする設計

以下は本プロジェクトの中核設計に採用しない。

- AIがC++オブジェクト、ポインタ、メモリを直接変更する経路
- AIが任意のEngine関数をreflectionで呼び出す経路
- AIへ任意shell、任意path、任意URL、任意console commandを公開する設計
- AIが出したJSONをSchema検証だけで実行可能と判断する設計
- チャット履歴だけをプロジェクトの正規記憶とする設計
- 一行のプロンプトから無検証で完成品を一括生成する設計
- 初心者向けと経験者向けで互換性のない別プロジェクト形式を作ること
- Unity、Unreal Engine、Godotの型階層、Scene形式、Editor操作、名称を模倣すること

## 3. 独自性に関する設計原則

有名ゲームエンジンは安全性、拡張性、Editor自動化の比較対象としてのみ調査する。内部アーキテクチャやUXの設計図として利用しない。

以下の既存製品固有モデルを出発点にしない。

- UnityのGameObject／MonoBehaviourモデル
- Unreal EngineのUObject／Actor／Component階層
- GodotのNode／Scene Treeモデル

本エンジンは、次の独自原則からデータモデルを設計する。

1. ゲーム状態の正規表現と表示方法を分離する。
2. 人間とAIが同じ変更プロトコルを利用する。
3. Editorは特権的な状態変更者ではなく、World ModelのProjection兼ChangeSet作成者とする。
4. ゲーム全体の宣言的設計と、実行時に最適化された状態を分離する。
5. AIの推論結果とEngineの権威判断を分離する。
6. ゲームジャンル固有機能は巨大な共通基底へ集約せず、Domain Packとして追加する。

一般的な数学、Rendering、Physics、Audio、Input、Asset管理、Undoなどはゲームエンジンに必要な共通概念であるが、公開API、ライフサイクル、シリアライズ、Editor表現は本プロジェクトの要件から独自に定義する。

「独自」は、OS、Compiler、Direct3D 12、標準Library、検証済みのPhysics／Navigation kernelまで再発明する意味ではない。製品の正規data model、公開Capability、編集protocol、validation、lifecycle、serialization、Editor UXを本プロジェクトが所有し、外部LibraryはEngine-owned Adapter内へ隔離する。Game programming modelはC++23とGameplayDefinitionに固定し、汎用Game scripting runtimeは採用しない。First-party C++公開境界はHeader準備期からNamed Modules＋`import std`へ一方向移行する。採用条件と初期Dependencyは基盤アーキテクチャ規約で固定する。

## 4. 対象ユーザーとAuthoring Mode

### 4.1 制作レベル

同じプロジェクトへ段階的にアクセスできるProgressive Disclosureを採用する。

| レベル | 主な操作 | 対象 |
|---|---|---|
| Level 0 | 自然言語、参考資料、選択肢への回答 | 初心者を含む全ユーザー |
| Level 1 | Scene、Inspector、Graph、UI、Data、Timeline | 制作経験者、ノーコード利用者 |
| Level 2 | GameplayDefinitionのGraph／Table／Form生成・直接編集 | 中級者、Designer、Mod制作者 |
| Level 3 | 公開SDK内のNativeGameModule（Project C++）の生成・直接編集 | 上級者。Engine製品開発は別Repository／Profile |

レベルは権限階層であり、別製品や別ファイル形式ではない。初心者がLevel 0で作ったプロジェクトを、経験者がLevel 1〜3で直接編集できる。

### 4.2 設計解像度

次の三つをAuthoring Modeとして提供する。

#### 詳細設計型

人間がゲームルール、数値、状態遷移、コンテンツを詳しく指定し、AIが忠実に実装する。重要な未指定事項をAIが黙って補完しない。

#### 共同設計型

人間が方向性と一部仕様を示し、AIが複数案、推奨案、影響を提示する。高影響事項は人間が決め、軽微な実装事項はAIが決める。

#### 高水準指示型

人間がコンセプト、体験、テーマ、規模を示し、AIが企画、詳細設計、実装、Testまで展開する。不可逆、高コスト、プロジェクト構造を大きく変える判断は確認する。

### 4.3 サブシステム単位の混在

Authoring Modeはプロジェクト全体へ固定しない。

```text
ゲーム全体       共同設計型
戦闘             詳細設計型
Quest            共同設計型
背景環境         高水準指示型
UI               高水準指示型
経済バランス     詳細設計型
Test             自動
```

ユーザーは途中でモードを変更できる。ロックされた人間の決定は、AIが変更提案なしに上書きできない。

## 5. 推奨制作ライフサイクル

本プロジェクトは「会話主導・段階生成型」を採用する。

```text
大まかな初回プロンプト
  → 要件抽出
  → 適応型の追加質問
  → Game Brief確認
  → ゲーム全体のGameSpec
  → Scale intent／Gameplay fidelity floor
  → Target別Representation Plan／統合負荷fixture
  → 実装方式の選択
  → 薄いゲーム全体＋深い代表部分
  → First Playable
  → AI会話と手動編集による反復
  → Alpha
  → Content Complete
  → Beta
  → Release Candidate
```

### 5.1 初回プロンプト

ユーザーは専門用語を使わずに入力できる。

```text
中世ファンタジーの3D Action RPGを作りたい。
剣と魔法を使い、街でQuestを受けてDungeonを攻略する。
一人用で、初心者でも遊びやすくしたい。
```

AIはこの入力からジャンル、中心体験、Core loop、視点、進行、世界、主要システム、規模、Platform、Performance要件に加え、`scene_dimension`、Art Direction、Camera／Composition、参考表現の候補を抽出する。2D／3DとRealistic／Toon／Pixelを同じ分類として扱わない。

### 5.2 適応型の追加質問

不足事項を次のように分類する。

| 分類 | 例 | 対応 |
|---|---|---|
| Blocking | 2D／3D／Hybrid、Hybridのgameplay space、Single／Multiplayerなど根本が不明 | 回答を得るまで実装判断を保留 |
| High Impact | Art Direction、Pixel DioramaのCamera、規模、Platform、目標FPS、Open World、PvP | 選択肢、低cost Preview、推奨理由を提示 |
| Medium Impact | Inventory枠、Checkpoint間隔、UI初期配置 | 仮設定し、報告または確認 |
| Low Impact | 内部ID、変数名、Test fixture | AIが決定して記録 |

回答操作は影響度で制限する。`Blocking` は、人間が回答するか、その判断だけを対象にした有効な `allow_ai_select` 委任によって解消するまで制作Commitへ進めない。`High Impact` の「おまかせ」は有効な委任とCapability／BudgetのHard gate通過を必須とし、条件を満たさなければ確認へ戻す。「期限付きで保留」は `Medium Impact` 以下に限り、期限、期限内に使用する明示値、影響範囲をDecision Logへ記録する。「仮実装して試す」は、Schema、保存データ、Public API、Gameplay spaceを固定しない可逆な実験だけに許可する。

初心者へGameplayDefinition／C++、ECS、RPCなどの実装方式を質問しない。AIは「同時に何体表示したいか」「オンラインが必要か」など、ゲーム上の要件を確認し、実装方式を内部で選択する。

### 5.3 Game Brief

質問後、AIは長大な企画書ではなく、確認しやすいGame Briefを提示する。

- ゲームの中心体験
- 対象プレイヤー
- Core loop
- 主要システム
- Scene dimension、Art Direction、Composition、Camera方針
- 規模
- 総配置、最大同時存在／可視、spawn burst、interaction範囲、敵味方VFX同時性、Target別Scale intent
- 敵味方数、Damage、collision、goal、timing等のGameplay fidelity floor
- First Playableの範囲
- AIが補完した事項
- 人間の確認が必要な事項
- 技術的・制作上の主要リスク

ユーザーが承認または自然言語で修正した後、全体設計へ進む。

### 5.4 薄いゲーム全体と深い代表部分

First Playableは二種類の完成度を組み合わせる。

#### 薄いゲーム全体

開始から目標達成まで短縮された一連の流れを動作させる。

- Title
- ゲーム開始
- Player操作
- Tutorial
- Core loop
- 短いQuestまたは目標
- ResultまたはEnding
- Save／Load

#### 深い代表部分

ゲームの魅力と技術的成立性を評価できる一部分を、他より高い完成度で実装する。

- 代表的な戦闘
- 代表的な探索エリア
- 主要Character
- 代表的なQuest
- 中核となるUI
- 主要なPerformance負荷

この組み合わせによって、一部分だけ豪華で全体へ拡張できない状態と、システムだけ揃って面白さを評価できない状態を避ける。

### 5.5 継続編集

First Playable以降、次の編集経路を同時に利用できる。

- AIへの自然言語指示
- Scene／Inspector
- Rule Graph
- UI Editor
- Data Table
- GameplayDefinition Graph／Table／Form
- C++ Editorまたは外部IDE

小変更は即時Preview、中変更は短い実装計画、大変更は影響分析と承認を経由する。

## 6. 正規データモデル

本章の名称を正規概念名とする。C++ symbol、file、namespaceの表記は基盤アーキテクチャ規約へ従う。

### 6.1 GameSpec

「どのようなゲームを作るか」を表す宣言的な設計図。

- ゲームの柱
- Core loop
- Scene／World構成
- ゲームルール
- Character／Enemy
- Item／Ability
- Quest／Dialogue
- UI
- Input
- Save
- Visual Style Contract
  - `scene_dimension`
  - `art_direction`
  - `composition`
  - `composition_variant`
  - `gameplay_space`
  - `visual_style_profile_id`
  - Style-critical fieldとlock状態
- Asset要求
- Test条件
- Performance budget

### 6.2 World Model

Authoring Serviceが保持するAuthoring Document集合のうち、`WorldDocument`、論理`SceneDocument`、`SceneEntityShardDocument`、Entity／Component／CompositionをWorld Modelと総称する。Editorが独自に所有する一枚のobject graphではなく、Scene Tree、Inspector、AI要約、Runtime Worldを正規状態にしない。

Commit済み`ProjectRevision`の同じDocument集合をScene、Graph、表、UI、Timeline、Simulation MonitorなどへProjectionする。Document種類、header、ID、参照、保存、Recoveryの詳細はAuthoring Model／Project State規約を正本とする。

World Modelの`SceneDocument`は編集Shardであり、Gameplay上のLevelまたはRuntime Streaming Cellではない。World、Topology、Level、Spatial Partition Intent、Target別Streaming Plan、Procedural Definition、Navigation、Map Presentationの正規境界は[World／Level／Map規約](./2026-07-20-world-level-map-ai-authoring-architecture-design.md)を正本とする。自然言語の「マップ」を型名またはSystem名へ直接採用せず、同規約の6分類へ解決する。

### 6.3 ChangeSet

`ProjectChangeSet`の略称。現在の`ProjectRevision`に対する不変な変更提案であり、Editor、AI、CLI、MCP、外部IDEのすべてが同じ形式を使う。

最低限、次を持つ。

```text
schema_version
request_id
base_project_revision
operations[]
  operation_id
  command_type
  target_stable_id
  typed_arguments
  preconditions
  dependencies
  declared_cost
```

### 6.4 変更種別

| 種別 | 対象 |
|---|---|
| GameChangeSet | World、Scene、Entity／Composition Recipe、UI layout／style binding、Asset参照／設定 |
| StyleChangeSet | VisualStyleProfile、Material Template、Art Asset、Animation presentation、Lighting、Camera、Post、VFX、UI Style、関連Import設定 |
| GameplayDefinitionChangeSet | Rule、FSM、Behavior Tree、Quest、Dialogue、Ability、UI Flow、公開Parameter、Test |
| NativeCodeChangeSet | C++ source、header、Build定義、Test |
| AssetChangeSet | 画像、音声、3D、Animation、Import設定 |
| SystemBundleChangeSetV1 | System Contract、Definition、Native Source、Asset、Migration、Testをexact hashで結ぶcoordination envelope |
| WorldAuthoringBundleV1 | World／Topology／Level／Streaming Intent／Procedural／Navigation／Map Presentation変更を結ぶcoordination envelope |

下二つのBundleは既存ChangeSetを置き換えず、複数AuthorityのStaging artifactを結ぶ。各種別は共通してRevision、Diff、承認、履歴を持つが、検証・適用方法は分離する。

### 6.5 Gameplay Capability Contract

GameSpecとGameplayDefinitionは具体的なC++ symbolではなく、version付きCapability ID、typed command／event／snapshotを参照する。

既存Capabilityの組合せで表現できない機能、または同一fixtureの計測で構造化実装がPerformance要件を満たさない機能は、同じ公開Contractを維持した`NativeGameModule`へ実装できる。呼び出し側のGameSpec、Save field、他Systemを全面変更しない。

ゲームシステム、レベルシステム、戦闘、Character、Ability、Encounter等は`GameSystemSpecV1`で責務、authoritative State owner、Command、Event、Snapshot、Runtime scope／phase、Save／Replay、Budget、Target fallback、Testを固定する。Engine Standard System Catalogは開始点でありWhitelistではない。Projectは検証済み`game_system.project.<project_namespace>.*`を第一級Systemとして追加できる。AIは`mirakan.systems.search／read`で必要な契約だけを取得し、`SystemImplementationPlanV1`と`SystemBundleChangeSetV1`を提案する。詳細は[Game System／AI Code Generation規約](./2026-07-20-game-system-ai-codegen-architecture-design.md)を正本とする。

### 6.6 Decision Ledger

設計判断の出所と理由を記録する。

```text
decision_id
value
source: human | ai_recommendation | ai_assumption | engine_default
reason
confidence
approved
locked
status: active | needs_review | superseded | rejected
applies_to
evidence_refs
decision_dependencies
validity_predicates
created_revision
confirmed_revision
superseded_by
```

チャット履歴そのものではなく、確定要件、仮定、却下案、未解決事項、ロック、承認を構造化して保存する。依存Document、Target、Capability、根拠hashが変わるChangeSetは、影響するDecisionの`InvalidateDecision`または`ReconfirmDecision`を明示的に含める。Gatewayは失効を黙って補完せず、lock済みDecisionは別AuthorityのUnlock承認なしに変更しない。

### 6.7 Project Profile

AIが実装方式を判断するための制約を保持する。

- 対象Platform
- Target Profileとrevision
- Distribution Profile
- 最低Capability Signatureとmemory class
- 目標render FPS。SimulationはC1／C2で60 Hz固定
- CPU／GPU／Memory budget
- Render Quality TierとTarget別必須Graphics capability
- 対象出力解像度／HDR
- Asset制作量、Texture／Mesh／Sprite animation budget
- 必須／無効化するVisual Style Capability。実際の利用可否はEngine生成`StyleCapabilityManifest`
- Scale intent。総配置、peak live／visible、spawn burst、interaction／visibility範囲、同時VFX envelope、Gameplay fidelity floor
- Target別qualification statusとIntegrated Scale Fixture Receipt
- Network要件
- Offline要件
- Mod対応
- Buildサイズ
- Input方式
- Accessibility
- Privacy／Retention policy
- Mobile orientation／resize／safe-area／touch fallback policy
- Permission、Content Delivery、Content Safety policy

### 6.8 AI可読Authoring契約

AIへProject全体を無条件に送らず、Commit済みrevisionから生成する`AuthoringContextIndexV1`、Task固有`AuthoringContextPlanV1`、実送信Manifest `ContextPackV2`の三層でContextを構成する。

`AuthoringContextIndexV1`はStableId、Document／Scene Shard位置、inbound／outbound reference、Requirement、Capability、Decision、lock、spatial bound、field byte／token costを索引化するDisposable artifactである。正本はAuthoring Document集合のままとし、Indexがstaleまたは未完成なら別revisionへfallbackしない。

`AuthoringContextPlanV1`はTask goal、anchor StableId、field mask、依存closure、必須Requirement／Decision、byte／token上限、追加read上限を固定する。`ContextPackV2`は実際に送ったItem、順序、source hash、選択理由、省略範囲、continuation、tokenizerと実測tokenを記録する。Blocking／High要件、lock済みDecisionを要約だけに置換せず、収まらなければTaskを分割する。

大規模Sceneは論理`SceneDocument`とbounded `SceneEntityShardDocument`へ分け、AIには任意に切断したJSONでなく`SceneSliceV1`を返す。`authoring.search`、`authoring.read`、`authoring.dependencies`、`authoring.diff`はrevision付きR0 query、編集はStableId対象のtyped Operationだけとする。AIへ正規Authoring JSONのwrite権限を与えず、`ai_mutable` fieldのtyped Operation coverage 100%をCapability昇格条件にする。

Validator failureはMCD `RemediationV1`から必要read、typed Operation template、precondition、Risk、Approvalを返す。初回Proposal後の修復は最大2回、同じblocking Diagnostic集合が減らない場合は停止する。Permission、Security、lock、Approval、revision driftを自動修復しない。詳細型、状態遷移、性能fixture、Release GateはAuthoring、AIガバナンス、実行可能契約、AI検証の各規約を正本とする。

## 7. システムアーキテクチャ

本章はProduct全体の論理構成を定義し、field、phase、lifetime、budget、failure、testの詳細は各正式仕様へ委譲する。

- C++ module依存、所有権、Build、directory、Dependencyは[基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)を基準とする。
- 正規Project状態、ProjectRevision、ChangeSet、journal、recoveryは[Authoring Model／Project State規約](./2026-07-19-authoring-model-project-state-design.md)を基準とする。
- Game実行コードと構造化ゲームデータの境界は[C++実行コード・構造化ゲームデータ規約](./2026-07-19-cpp-structured-game-data-design.md)、Native ABI、Target別link、Build／Promotionは[NativeGameModule規約](./2026-07-19-native-game-module-architecture-design.md)を基準とする。
- Runtime phase、Subsystem連携、borrow無効化、Asset version、memory／performance budget、障害復旧は[Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)を基準とする。
- Debug Session、構造化Event／Counter、Query、Breakpoint／Watch、Causality、Replay／Rewind、Reproduction Bundle、AI診断境界は[AI可読Debugging／Observability／Replay規約](./2026-07-20-ai-readable-debugging-observability-replay-architecture-design.md)を基準とする。
- Renderer、Asset、Collision、Physics／Navigation／Animation、Input、UI／Text、Audio、Editorは、それぞれのSubsystem正式仕様を基準とする。全リンクと決定権は[設計文書Index](./README.md)を正本とする。
- 2D／3Dの製品機能範囲と成熟度は[2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)、Genre別機能と将来Capabilityは[Domain Pack／将来Capability規約](./2026-07-19-domain-pack-future-capability-roadmap.md)を基準とする。
- Windows Target／Distributionは[Windows Platform／Distribution規約](./2026-07-19-windows-platform-distribution-design.md)、Android／AppleのTarget、Adapter、実機budget、package、Store gateは[モバイルPlatform規約](./2026-07-19-mobile-platform-architecture-design.md)を基準とする。
- AI Task、権限、Source隔離、API／MCP／CLI／Pluginの役割は[AI実装・保守ガバナンス規約](./2026-07-19-ai-engine-development-governance-design.md)、Requirement、Schema、Codegen、Provider projectionは[実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)、Test、形式モデル、Eval、Receipt、Provenance、Evidence更新は[AI検証・評価・来歴規約](./2026-07-19-ai-verification-evaluation-provenance-design.md)を基準とする。

```text
Human Prompt / Editor / External Agent
                |
                v
      Requirement Resolver
                |
                v
           Game Brief
                |
                v
      Visual Style Resolver
                |
                v
             GameSpec
                |
                v
       UI Authoring Planner
                |
                v
 Implementation Strategy Planner
                |
       +------------------+
       |                  |
 GameplayDefinition     C++
       |                  |
       +------------------+
                |
                v
       Normalized ChangeSet
                |
                v
    C++ AuthoringCommandGateway
   Schema → Semantic → Capability
   Policy → Budget → Revision
                |
                v
        Staging / Dry-run
                |
                v
    Engine-generated Diff / Tests
                |
                v
       Approval / Atomic Commit
                |
                v
    World Model / Runtime Package
```

### 7.1 C++ Engine Core

最終的な権威処理を担当する。

- Authoring Document Store／World Model
- Stable ID
- Revision
- `AuthoringCommandGateway`
- Validator registry
- Transaction
- Undo／Redo
- Journal
- Compiler
- Runtime package
- Capability registry
- Test harness

### 7.2 Projection Editor

同じWorld Modelを複数の編集表現へ投影する。

- Scene View
- Inspector
- Graph
- UI
- Data Table
- Timeline
- Code View
- Diff／History
- AI Conversation
- Build／Playtest結果

Editor GUIの操作も直接状態を書き換えず、ChangeSetへ変換する。

### 7.3 AI Orchestrator

Engine外のNode.js 24.18.0 LTS／TypeScript 7.0.2 strict別Processとして配置し、Model Providerの更新、障害、CredentialをEngineから隔離する。EngineとはACL付きWindows named pipe上のversioned JSON-RPCで接続する。

- Provider adapter
- Prompt／Schema version
- Model routing
- Session
- Job queue
- Retry
- Rate limit
- Cost budget
- Audit
- Background／Batch
- Asset generation provider

最終検証とCommit権限は持たない。

初期ProviderはOpenAI Responses API、公式TypeScript SDK、評価Modelは`gpt-5.6-sol`のreasoning effort `medium`に固定する。ModelとPromptはEval後にだけProvider manifestを更新し、Engine codeやProject schemaへModel IDを埋め込まない。Anthropicは第二Providerとして同じconformance suite通過後に追加する。

### 7.4 Requirement Resolver

自然言語をGoal、Constraint、Acceptance Criteria、Unknown、Conflictへ分解する。不足要件を影響度で分類し、質問、提案、自動補完を選択する。Art DirectionはHigh Impact、2D／3D／HybridとHybrid gameplay spaceはBlockingとして扱う。

### 7.5 Visual Style Resolver

Game Brief、参考Asset、既存Asset、Project Profile、`StyleCapabilityManifest`から利用可能なVisual Style候補を作る。EngineがCapabilityとbudgetをhard gateし、AIは残った候補の制作cost、runtime cost、一貫性、理由を説明する。GenreだけでStyleを決めず、曖昧なHigh Impact項目を無確認でlockしない。

正規四軸、VisualStyleProfile、Material／Shader、Toon、Realistic、2D、Pixel Diorama、Semantic Catalog、AI Operation、Preview、Explain、Validator、Evalの詳細は、[Material／Visual Style／AI Authoring規約](./2026-07-20-material-visual-style-ai-authoring-architecture-design.md)を唯一の基準とする。2D／3D機能計画§8は製品到達点とPhase配置だけを所有する。

### 7.6 UI Authoring Planner

`GameSpec`、画面遷移、Target、safe area、入力方式、locale、accessibility、Visual Style lock、Asset／frame budgetを読み、HUD、Title、Settings、Pause、Result等の`UiDocument`、`UiCompositeDefinition`、style、binding、`AssetRequirement`を提案する。出力はStable IDを対象にした型付きOperationを含む`ProjectChangeSet`であり、Editor tree、Runtime Widget、画像Asset、Project fileを直接変更しない。

提案はSchema、semantic、Capability、binding型、layout cycle、locale、focus、semantics、contrast、Target別budgetを検証する。承認前Previewは解像度、aspect ratio、safe area、代表locale、keyboard／mouse、gamepad、touch、accessibility設定の組合せを対象にし、構造化検証と操作testを合格条件とする。Screenshot差分は補助証拠であり、UIの正本または唯一の成功判定にはしない。

Widget選択はBuiltin、`UiCompositeDefinition`、`UiEffectGraph`、`UiNativeWidget`の順に最小能力を選ぶ。C1はBuiltinとCompositeまで、Effect GraphとNative WidgetはC2で個別にPromotionする。Native WidgetのSource生成はA1 Gate、R3 Review／Promotion、型付きCapability、budget、fallbackを必須とし、Editor processへProject C++や外部binaryをloadせず、実Previewは別ProcessのGameHostで行う。

画像生成を含むAsset要求は、Providerへ直接Project pathを書かせず、Stagingへ候補、Prompt／Model／Provider、license、provenance、safety結果を保存する。Asset Pipelineのimport、cook、Target／budget検証を通過した候補だけをChangeSetで参照し、未承認候補、文字を焼き込んだUI画像、秘密情報を含む生成物をCommitしない。詳細契約は[UI／Text／Localization／Accessibility規約](./2026-07-19-ui-text-localization-accessibility-design.md)を正本とする。

### 7.7 Implementation Strategy Planner

システム単位でGameplayDefinition、C++、または型付きCapability境界での併用を選択する。LLMの推測だけで決めず、Engine Policyと実測を組み合わせる。汎用Game scripting runtimeは候補へ含めない。

Plannerは`GameSystemCatalogV1`から既存composition、Project GameplayDefinition、Project-defined System、bounded NativeGameModuleの順に比較し、`SystemImplementationPlanV1`へRequirement、Contract hash、State owner、Target、Budget、同値fixture、Save／Migration、fallback、却下案を記録する。いずれでも意味同等に実現できなければ`capability_unavailable`で停止する。AIがC++を選ぶ場合も`.cpp`だけを生成せず、generated binding、CMake、manifest、Definition参照、Test、Benchmark、System Bundleを同じdependency closureへ含める。

大量制作では実装言語を選ぶ前に、Runtime規約のFull Entity／simulation LOD／休眠state／pool、Renderer規約のindividual／instanced／spatial／presentation、VFX規約のCPU／GPU／aggregate emitterをTarget別Representation Planへ解決する。AIは`Predicted | OptimizationRequired | Qualified`を区別し、実測Receiptなしに「快適」「最適化済み」と説明しない。Presentation-only変更で合格できない場合は、Gameplay変更またはTarget除外を承認付き別ChangeSetとして提示する。

### 7.8 MCP Adapter

外部のCodex、Claude Code、その他MCP対応HostへEngine機能を公開する、EditorHostから独立して有効化できる北向きAdapter。

初期MCP公開範囲は次に固定する。

- describe_capabilities
- get_world_summary
- query_scene
- query_asset_catalog
- propose_changeset
- validate_changeset
- preview_changeset
- run_playtest
- get_build_report
- capture_viewport

初期段階では、MCPからの直接Commitを公開しない。CommitはEditorで承認する。

### 7.9 Codex／Claude Plugin

Host固有Pluginは任意の薄い補助UXとする。PluginがなくてもProvider APIとMCPで全公式機能を利用できなければならない。

- Engine固有skill
- GameSpec／ChangeSet説明
- Coding／Asset規約
- Build／Test手順
- MCP起動設定
- 検証・修復workflow

権威ロジックやEngine状態は保持しない。

### 7.10 CLI／Desktop

Codex／Claude CLI・DesktopはEngine開発、試作、CI、Code生成、Build、Testに利用する。出荷ゲームのRuntime依存にはしない。

## 8. C++実行コード＋構造化ゲームデータ戦略

### 8.1 基本方針

Engine、Editor、GameHost、Project固有の実行コードはC++23とする。頻繁に調整するゲーム内容は`GameplayDefinition`として宣言し、offline Cookした`CookedGameplayPackage`をEngineのC++ evaluatorが実行する。汎用Game scripting runtime、bytecode interpreter、JIT、Game向けFFIは導入しない。

ユーザーは実装方式を毎回選ばない。AIは次の順で判定し、理由、前提、Contract、Budget、Benchmark ReceiptをDecision Ledgerへ記録する。

```text
既存CapabilityとGameplayDefinitionで表現可能
  → GameplayDefinition

参照・index・layout・Cook最適化でBudget内
  → GameplayDefinitionを維持

公開SDK内の新規Game Algorithm／計測済みhot path
  → NativeGameModule

Engine private API、Platform／Native SDK、新しいEngine-wide Capabilityが必要
  → capability_unavailable（Game制作Task内でEngine変更へfallbackしない）

同一fixtureで構造化実装がBudget超過
  → C++候補と比較し、有意な改善時だけC++へ昇格

調整可能なRuleと高性能Kernelが両方必要
  → GameplayDefinition＋typed Capability境界＋C++
```

### 8.2 判断材料

- 既存Capabilityで完全に表現できるか
- データ量、同時Entity数、呼出頻度、Latency
- CPU／MemoryのBehavior Budget
- Offline Cookで参照解決、index作成、SoA化、事前計算が可能か
- 決定論、Save／Replay、Network Authority
- Platform／Store／Security境界
- 新しいNative SDKまたは低Level APIが必要か
- 人間が頻繁に調整する値・Ruleか
- 対象Target ProfileのBenchmarkと過去Receipt

Genre名、2D／3D、Project規模だけを理由にC++へ昇格しない。

### 8.3 GameplayDefinitionの責務

- Rule、typed State Machine、bounded Behavior Tree
- Ability、Quest、Dialogue、Encounter
- Enemy intentと既存Capabilityのcomposition
- UI Flow、Stage gimmick、Event reaction
- 数値、Curve、Tag、Profile、参照
- AI生成と人間によるGraph／Table／Form編集

Definitionは許可済みCapability ID、型付き引数、Stable ID、状態IDだけを保持する。任意関数名、pointer、native handle、File、Network、OS command、loop、recursion、callback登録を保持しない。State Machineは一つのinstance／phaseにつきauthoritative transitionを最大1件、Behavior Treeは`max_node_visits_per_tick`を必須とする。

### 8.4 C++の責務

- Engine／Editor／Platform／Adapterの実行基盤
- Physics、Navigation、Rendering、Audio、Input等のKernel
- Serializer、Allocator、Job system、Security境界
- 大量Entity処理または低Latency hot path
- 新しいAlgorithm、Native SDK、Platform API
- GameplayDefinitionへ公開するversion付きCapability

Project固有C++は`NativeGameModule`として、Engine-ownedのtyped command／event／immutable snapshotだけを利用する。Engine／vendor pointer、allocator、World内部container、GPU／Physics native objectを公開しない。

### 8.5 C++変更範囲

| 範囲 | AIの標準権限 | 必須検証 |
|---|---|---|
| bounded NativeGameModule | Game Projectの隔離Workspaceで生成・修正可能 | 不変Engine規約のG0～G7、`SystemTechnicalAttestationV1` |
| Engine／Platform Adapter | Game制作Profileでは権限なし | `capability_unavailable`。別のEngine製品開発工程だけで扱う |
| Engine Extension | Game制作Profileでは権限なし | `capability_unavailable`。別のEngine製品開発工程だけで扱う |
| Engine Core | Game制作Profileでは権限なし | `capability_unavailable`。別のEngine製品開発工程だけで扱う |

初心者の自然言語指示からC++が必要になった場合も、まず隔離Workspaceへ`NativeCodeChangeSet`を生成し、CompileとTestを通す。稼働中Engineへ未検証codeを注入しない。Desktop Previewは別ProcessのGameHostをBuild後に再起動し、C++ DLLのin-process hot unloadを1.0では行わない。Android／AppleのC++変更は再Build、再署名、再installを必要とする。

### 8.6 実測による再評価

Systemごとに`BehaviorBudget`がなければ、性能を理由に実装方式を確定せず`MissingImplementationBudget`をBlocking診断にする。同じReference fixture、入力、品質、Target Profile、clean processで10分間を3回測定し、最悪回のP95、peak memory、deadline miss、allocation countを比較する。

- Budgetの80%以下、deadline miss 0、決定論／安全性Gate合格: GameplayDefinitionを維持する。
- 80%超100%以下: DefinitionのCook、index、layoutを最適化し、C++候補と同一fixtureで比較する。
- 100%超、deadline miss発生、またはNative APIが不可欠: C++候補を作る。
- C++の改善が5%未満または測定Noise内: GameplayDefinitionを維持する。

80%閾値と5%差は初期Project Policyであり、Task内のAI判断では変更しない。C++へ昇格する場合もCapability ID、command／event、Save field、GameplayDefinition参照を維持する。

## 9. AI編集と手動編集の統合

### 9.1 同じ正規状態を利用する

AI、Scene、Graph、Inspectorからの状態変更はGameChangeSetまたはGameplayDefinitionChangeSetへ統合する。GameplayDefinitionの正規情報はAuthoring source、C++の正規情報はSource fileとし、File hash、Base revision、Definition Cook結果またはBuild結果を管理する。

任意のC++をGameSpecへ逆変換しようとしない。C++ Moduleは提供するCapabilityとContractをEngineへ登録する。

### 9.2 外部IDE

GameplayDefinition sourceまたはC++が外部Editor／IDEで変更された場合、File watcherが変更を検出する。

1. File hash更新
2. Parse／Index
3. Schema／semantic validationとCook、またはC++ Compile
4. Capability影響分析
5. GameSpec参照の検証
6. 履歴登録
7. Editor表示更新

### 9.3 AIによる再編集

AIは現在のSource、GameSpec、World Model、Decision Ledgerを再読込してから変更する。

- 人間が変更した箇所を無条件に再生成しない。
- 古いBase revisionを持つ提案は拒否または再生成する。
- 衝突時は三者比較を提示する。
- ユーザーがロックした設計とSource領域は、変更提案なしに編集しない。

### 9.4 自然言語による編集

ユーザーは次のように指示できる。

- 「この敵だけ弱くして」
- 「手動変更を維持して空中攻撃を追加して」
- 「昨日追加したQuestだけ元に戻して」
- 「今の見た目を変えずに処理を軽くして」
- 「AIが仮に決めた項目を表示して」
- 「構造化ルールのHot pathを計測して、必要ならC++化して」

AIは指示の対象、範囲、影響度を解決し、ChangeSetとTestを作成する。

## 10. 検証・安全・Transaction

### 10.1 唯一のCommit権限

API、Agent SDK、MCP、Plugin、CLI、Editor GUIのいずれも安全境界とはしない。C++ `AuthoringCommandGateway`だけがAuthoring Document集合と`ProjectRevision`を確定できる。

### 10.2 検証パイプライン

1. Provider response状態
2. Context Plan、Project／Index revision、Blocking／High evidence completeness
3. Strict Schema parse
4. Canonicalization
5. Stable ID／Reference
6. Semantic invariant、Decision invalidation
7. Capability／Authorization、typed Operation coverage
8. Resource budget
9. Base revision／Conflict
10. Staging／Dry-run
11. Engine-generated Diff
12. Risk policy／Approval
13. Atomic Commit
14. Undo／Journal／Context／Generation Receipt

### 10.3 AIへ書かせない信頼情報

- 実ユーザーID
- Tenant
- 権限
- 承認済みフラグ
- API Credential
- Provider request metadata
- 最終Commit Capability

これらは信頼済みHostまたはBackendがAI出力とは別のEnvelopeへ付与する。

### 10.4 Approval

承認はAIの説明文ではなく、EngineがCanonicalizeしたChangeSet hash、対象World、Base revision、有効期限、一回限りのNonceへ紐付ける。AIが`approved: true`を出力しても権限として扱わない。

### 10.5 Undoと再現性

逆操作をAIへ生成させない。EngineがBefore image、Event log、Transaction log、CheckpointからUndoを構成する。

再現性は次の三つへ分ける。

1. 生成条件の記録: Model、Prompt、Schema、Context hash
2. 監査の再現: Raw response、Normalized plan、Validation、Retry
3. Engine replay: 承認済みChangeSetをAIなしで再生

最重要なのはEngine replayである。

### 10.6 Source Sandbox

C++／Shader source生成は隔離されたBuild環境を利用する。GameplayDefinitionは同じStaging revisionへ置くが、Source実行は行わず、Schema、semantic、Capability、budget、Cook、simulation gateを通す。

- Read-only input
- Ephemeral workspace
- Network denyまたはAllowlist
- 固定Toolchain
- CPU／Memory／Time／Output上限
- 静的解析
- Compile
- Unit／Integration Test
- Artifact検査
- Capability登録検査

失敗したCodeChangeSetをProjectの有効状態へ反映しない。

## 11. 多ジャンル対応

巨大な単一Schemaへ全ジャンルを詰め込まず、CoreとDomain Packへ分離する。

### 11.1 Core

- Stable identity
- 2D／3D Transform／Spatial
- World
- Asset reference
- Data table
- Graph
- 2D Canvas／3D Rendering
- 2D／3D Physics
- 2D grid／polygon Navigation、3D Navmesh
- Animation／VFX
- Input／Audio／UI
- GameplayDefinition／Native Capability
- Serialization
- Command／Transaction
- Test

### 11.2 計画済みDomain Pack

- 2D Action
- reusable `mirakan.feature.shooter_core.c1`
- 2D top-down shooter Profile
- Character／Combat／Vital共通
- FPS／TPS
- RPG／Action RPG
- Quest／Narrative
- Simulation／Economy
- Strategy
- MultiplayerはC3のNetwork設計承認後

各Packは次を持つ。

- Schema fragment
- Semantic validator
- Capability manifest
- Preview
- Runtime compiler
- Migration
- Test suite
- AI向け説明

AIは署名済みEngine baselineへインストール済みのCapabilityだけを利用できる。不足する場合は、GameplayDefinition composition、bounded NativeGameModuleの順に検討し、それでも実現不能なら`capability_unavailable`で停止する。Game制作TaskからEngine Extensionを生成または変更しない。

## 12. Editor制作型とRuntime生成型

### 12.1 Editor制作型

最初の主対象とする。

- GameSpec生成
- Scene／Rule／UI／Asset設定
- GameplayDefinition／C++生成
- Diff
- Dry-run
- Playtest
- Build
- Export

### 12.2 Runtime生成型

Editorで検証基盤が成熟した後、同じIRとValidatorを異なるPolicy Profileで再利用する。

Runtimeでは次を原則禁止する。

- 任意C++生成・実行
- 任意の汎用Game bytecode生成・実行
- 任意Asset download
- Arbitrary World mutation
- Physics／Network stateの直接確定
- Client内へのProvider API key埋め込み

Android／iOS／iPadOSのShipping Runtimeでは、この禁止をさらに具体化し、C++、native library、DEX、JavaScript、Python、汎用Game bytecode、shader source／binaryの生成、post-install remote download、JIT、動的loadを認めない。Store審査対象のbase packageへoffline compile済みshaderを同梱する通常Buildは除く。許可するのは、署名済みbinaryが既に実装するSchemaとCapabilityの範囲内で、検証された構造化dataを変更することだけである。Asset deliveryへ実行codeを混入しない。詳細なStore、content safety、package scanの規約は[モバイルPlatformアーキテクチャ規約](./2026-07-19-mobile-platform-architecture-design.md)に従う。

最初のRuntime Policyで提案を許可する範囲は次に限定する。実装開始前にRuntime専用Threat Modelとserver authority設計を別途承認する。

- Dialogue
- Quest outline
- 既存TemplateからのEncounter
- 制限されたNPC intent
- 既存Assetを使ったWorld構成
- 事前定義されたSimulation parameter

MultiplayerではAuthoritative serverだけがCommitし、LLMは提案者に限定する。

## 13. AI統合技術の位置付け

### 13.1 推奨構成

- 中核推論: OpenAI／Anthropicなどの直接Model API
- Agent loop: MVPでは独自Orchestrator。Agent SDKはProduction Capability段階以降にEvalとADRを通した場合だけ採用
- 外部AI接続: MCP Adapter
- Host固有配布: 薄いCodex／Claude Plugin
- 開発・CI: CLI／Desktop
- 権威境界: C++ `AuthoringCommandGateway`

### 13.2 Provider依存の隔離

GameSpec、ChangeSet、Gameplay Capability ContractをProvider固有形式へ依存させない。内部Provider adapterを設け、一社から開始して同一Evalで追加Providerを比較する。

MCDを正本とし、MCP、OpenAI strict、Anthropic Toolへ別々のProvider projectionを生成する。Provider subsetで表現できないConstraintをManifestへ列挙し、C++ `AuthoringCommandGateway`がInternal JSON Schema 2020-12とsemantic／permission／budget validatorで完全再検証する。Provider Schema適合だけをCommit条件にしない。

評価指標は次のとおり。

- Schema適合率
- Semantic validation合格率
- 修復ターン数
- Build／Playtest成功率
- 採用ChangeSet当たりの費用
- P95 latency
- 禁止Tool提案率
- 手動変更保持率

## 14. 段階計画

### 全Phase共通: 計測先行・縦切り実装・性能Gate付きCapability昇格

Miraikanai Engineは、高度機能を先に並列実装して最後に一括最適化する方式も、全Subsystemの汎用基盤を完成するまでPlayableを作らない方式も採用しない。各Phaseと各Capabilityは、次の一方向loopで実装する。

1. MCD Requirement、Capability、Target Profile、Budget、failure、fallbackを固定する。
2. 最小のreference経路と、成功／失敗を再現する決定論的fixtureを作る。
3. PlayableまたはHeadless vertical sliceへ接続し、Target実機または規定Reference環境でtraceを取得する。
4. CPU critical path、GPU pass、memory、allocation、queue latency、Asset／pipeline準備、起動／reloadの実測から上位ボトルネックだけを選ぶ。
5. 同一Source、input trace、Target Profile、Quality、Toolchain、driver条件でBefore／Afterを比較し、意味、画質、Replay、fault recoveryを回帰検証する。
6. Runtime規約のhard budget、Subsystem固有Gate、統合密度fixtureをすべて満たした場合だけCapabilityを昇格する。
7. 不合格時はSource intentとlast valid baselineを維持し、`OptimizationRequired`または当該Capability非表示として次機能へ進まない。

独立した「最適化最終Phase」は設けない。Phase 0は測定、trace、Receipt、基準fixtureを実装可能にし、Phase 2以降は各Playableへ同じ測定経路を接続する。C1はPortableな基準経路を先に完成させ、C2は高機能を一Capabilityずつ予算内へ追加し、C3は別仕様、別prototype、意味同等fallback、個別Qualificationを持つ場合だけ着手する。

Authoring Source、Material／Visual Style、Gameplay fidelity floor、LOD IntentはTargetごとに分裂させない。Runtime Compilerが同じ正本からPortable、Mobile、AdvancedのTarget別Artifact closureを生成し、Packageは選択Targetに必要な検証済みArtifactだけを含める。高性能Targetの成功を最低Targetへ一般化せず、最低Targetの制約を理由に高性能TargetのSource表現を削除しない。

平均fps、推定cost、単一GPU、生成Frame込みのdisplayed fps、短いmicrobenchmarkだけで完了を宣言しない。性能値と測定方法の正本はRuntime規約14節、Renderer固有の画質／Provider／GPU GateはRenderer規約19～21節、機能範囲と順序は2D／3D機能計画13節を適用する。

### Phase 0: Foundation契約とToolchain

- 設計文書Indexに列挙した公式Review set 47文書の承認
- C++23共通Runtime Contract、`CxxFrontendProfileV1`、`CppDependencySetV1`、`BuildDriverProfileV1`と`windows_desktop_v1`、`android_mobile_v1`、`apple_mobile_v1`のTarget Profile schema
- Windows 11 25H2以降 x64／Direct3D 12を最初に実装し、Android Vulkan／Apple Metalを同じGraphics Portへ接続する境界
- Windows、Android、Apple toolchainをprofile別に固定する`toolchain.lock.json`と、Store要件を14日以内かつSubmission 7日前以内に再確認する`store_policy.lock.json`
- vcpkg manifest／固定Dependency
- Build GatewayをEditor／AI／CLI／CIの唯一の製品Build入口とし、CMakeをC++ Build定義、Ninjaを生成DAG実行器、Gradle／XcodeをPlatform package ownerへ固定
- CMake Presets／File API、Engine-owned Build Receipt、生成Ninja file非公開、Target／Configuration／Toolchain別Build tree分離
- `VerificationReceiptV1` gate `mirakan.build.ninja_adoption.v1`によるclean／no-op／leaf変更／Module変更／generated Header変更／中断復旧／memory／artifact一致の実測
- Module dependency DAGとComposition Root
- `mirakan_runtime_contracts`、`mirakan_runtime_package`、`RuntimeOrchestrator`、Domain Port／Runtime／Adapter境界、`ComponentAccessManifest`
- RAII、所有権、generation handle
- 12段階fixed tick、8段階render frame、typed command／event、bounded queue
- `RuntimeScaleIntentV1`、Gameplay fidelity floor、Target別Representation Plan、`IntegratedScaleFixtureV1`、qualification statusのMCD契約
- CPU memory domain、GPU allocator、residency、submission-deferred release
- Result／Error、thread affinity、borrow epoch、Asset version／atomic promotion
- `mirakan_foundation`と`mirakan_math`の一方向依存、portable scalar reference、storage／semantic math type、finite／normalization／failure contract
- Naming、format、static analysis、sanitizer
- 2D／3Dの座標、単位、色、時間規約と、Position／Direction／Velocity／Scale／Quaternion／TransformのMCD意味型
- Scene dimension、Art Direction、Composition、Shading Modelの正規四軸
- Lifecycle、Display、Graphics、Input、Audio、Text、Content Delivery、Thermal／MemoryのPlatform Port
- `/schemas/mirakan/`のMCD meta-schema、Contract compiler、C++／TS／Cooked binary descriptor／MCP／Provider projection
- `game_system` meta-schema、最小`GameSystemSpecV1`／Catalog／State owner／dependency graphのvalid・invalid fixture。Game実装または空System classは作らない
- World／Topology／Level／Partition Intent／Streaming Plan／Map Intentの最小Schemaとnegative fixture。large-world Runtimeまたは空`MapManager`は作らない
- `TaskSpecification`、署名済み`TaskAuthorizationEnvelope`、`AuthoringContextPlanV1`、`ContextPackV2`、`RemediationV1`、Provider Manifest、R0–R5
- Source Worker／Path broker／Network／PromotionのMCD契約、`NotActivated`拒否stub、security fixture。実WorkerはA1で解放
- TLA+／TLC v1.7.4で検査する5 State machineのmapping。Phase 0ではA0とFoundation実装に該当するModel、残りは各Activation前に実行
- Requirement Coverage、AI Eval、Verification／Generation ReceiptのA0実装と、Review／Promotion Receipt、SBOM／provenanceのSchema境界
- `DBG0_contract`／`DBG1_flight_recorder`: `DebugSessionDescriptorV1`、`DebugEventEnvelopeV1`、Counter定義、priority、gap、bounded retention、crash-safe chunk、D0／D1計測、`DebugQueryV1`のMCD契約とheadless reference実装

完了条件は、空のWindows EditorHost／Windows GameHost／WorkerHostが固定toolchainでBuildでき、Android／Appleの未実装Adapterが偽の成功ではなく`UnsupportedTarget`を返し、Foundation contract、Target Profile、phase順序、handle／borrow、bounded queue、memory failure、Asset atomic promotionのtest、ASan、format、static analysisがCIで成功することである。さらに、同一MCDからのA0用Projectionが決定論的に生成され、AIが変更不能なAuthorization Envelope、未解放Source Workerの`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`拒否、Provider再検証、該当TLA+ fast model、AI security negative fixture、Verification／Generation Receipt hash chainがheadless CIで合格しなければPhase 1へ進まない。最小GameHostでSession開始、D0／D1 Event／Counter記録、priority別dropとgapの可視化、bounded Query、異常終了後の完全chunk回収を再現し、生pointer／credential／無制限Query／stale indexを拒否できなければ`DBG1_flight_recorder`を完了扱いにしない。

Phase 0で全Risk classの契約と拒否動作を定義するが、製品機能として解放するのはAI実装・保守ガバナンス規約のA0（R0–R2）だけである。A1のProject Source、A2のEngine保守、A3のReleaseは各Activation Gate完成後にTool catalogへ追加し、未完成機能を弱い検証で先行開放しない。

### Phase 1: Headless Authoring Core

- GameSpec、World Model、ChangeSet
- `WorldDocument`／`SceneDocument`／`LevelDefinitionV1`／`WorldTopologyDefinitionV1`のSource境界とcompact Level lifecycle
- `SystemImplementationSetV1`、Project-defined System registration、System／World BundleのStagingと二段階Activation境界
- Stable ID／Project Revision
- Capability、`AuthoringCommandGateway`
- Schema／Semantic／Budget validation
- Authoring hard budgetとRuntime Target budgetの分離、`Predicted | OptimizationRequired | Qualified`、Receipt invalidation
- Dry-run、Diff、Atomic Commit
- Transaction、Undo／Redo、Journal
- Save／Load、Replay
- Versioned schemaとoffline Project Migrator
- VisualStyleProfile、StyleChangeSet、StyleCapabilityManifestのschema
- 論理`SceneDocument`、bounded `SceneEntityShardDocument`、`entity_set_root_hash`
- `AuthoringContextIndexV1`、`SceneSliceV1`、revision付きsearch／read／dependencies／diff
- Decision validity／invalidation、lock、`InvalidateDecision`／`ReconfirmDecision`
- `ai_mutable` fieldのtyped Operation coverage 100%検査

完了条件は、手書きChangeSetをheadlessでvalidate→stage→commit→save→load→replayし、同じstate hashを得られることに加え、100万Entity fixtureをShardへ保存してStableId検索、限定Slice、部分Diff、re-shard、Decision／lock closure、stale Index拒否を実行できることである。

### Phase 2: Editor Shellと共通Runtime

- 独自MirakanUi Core、Retained tree、Layout、Event、Focus、Semantic、D3D12 UI pass
- Dock／resize／floating／multi-workspace Editor shell
- Scene／Canvas、Outliner、Inspector、Asset、Diff／History、AI Partner panel
- Windows window／GameInput／XAudio2 Adapter、Editor shell用DirectWrite／TSF／UI Automation／OLE
- Human、keyboard、assistive technology、AIを同じtyped Editor Commandへ収束
- D3D12 device、Render Graph、D3D12MA、Debug Layer／DRED／PIX marker
- individual／instanced／spatial／presentation Render Representation、CPU cull／LOD／instancing、Scale diagnostic
- Asset staging、content-addressed cache、sandboxed importer、cook
- GameplayDefinition schema／Validator／Cooker／C++ evaluatorとNativeGameModule boundary
- Material IR、Engine-owned Target Binding Layout、Material／Style Validator、代表Material preview
- `DBG2_editor_local`: Session／Console／Problems／Profiler／Timeline／WatchのDebug Workspace、T110 safe pause、tick／render-frame step、GameplayDefinition node step、外部IDE／GPU debugger launch

本Phaseのconcrete Adapterとpackageは`windows_desktop_v1`を対象とする。完了条件は、AIを使わずEditor操作がChangeSetを経由し、空SceneをWindowsでplay、save、packageできることに加え、同じtyped Debug StoreをEditor panel、headless Query、外部debugger launchが参照し、pause要求がT110以外でsimulation stateを露出せず、step結果がSession／tick／frame／revisionへ再結合できることである。

### Phase 3: 2D Manual First Playable

- CanvasRenderer、sprite、tilemap、2D light、camera
- Box2D Adapter、2D grid navigation
- compact 2D Level、Portal transition、Level activation失敗時の旧Level維持
- Game Flow、Level Gameplay、Character、Weapon、Shooter Projectile、Combat、Vital、Score、Ability、Encounterの最小Engine Standard System Contract
- 標準Widget＋`UiCompositeDefinition`によるTitle、Settings、HUD、Pause、Result、Audio、2D animation、CPU particle
- GameplayDefinitionでgame ruleを実装し、C++ evaluatorで実行
- `pixel_2d`と`illustrated_2d` Profile
- `mirakan.feature.shooter_core.c1`＋`shooter.profile.2d_top_down.c1`による`pixel_2d` 2D top-down shooter縦切り
- `2d_shooter_c1_v1`と`2d_crowded_battle_v1`でWeapon、Projectile、Hit、Damage、Score、大量配置、敵味方spawn、Physics、VFX、Audio／Cameraを同時測定
- `DBG3_replay_causality`: input／seed／authoritative command／snapshotを記録し、record→scrub→inspect、first divergence、causal edge、recorded／current revision差分、最小Reproduction Bundleを成立

完了条件は、AIなしでTitleからResultまで5分playでき、typed Weapon、hitscan／Projectile、ammo／reload Profile、Health／Damage／Team、one-shot Pickup、Wave／Boss phase、Score／Combo、Save／Loadを含み、640×360 logical pixelを1920×1080へ3倍integer scaleすることである。Reference stress scene、`2d_shooter_c1_v1`、`2d_crowded_battle_v1`は1080p60、memory／queue／VFX budget、authoritative drop 0、Replay一致を満たす。既知のsimulation divergence、queue overflow、stale handle、asset revision driftをTimeline／Causality／Replayから誤因果なしで特定し、recorded data欠損時は原因確定ではなくgapとして停止できなければならない。

### Phase 4: AI Authoring MVP-A

- Node.js／TypeScript AI Orchestratorとnamed-pipe IPC
- OpenAI Responses API、Structured Outputs、strict function calling
- Requirement Resolver、Game Brief、GameSpec生成
- VisualStyleResolver、2D Style候補Preview、Decision Ledger
- UI Authoring Planner、自然言語からのHUD／画面UI生成、`AssetRequirement`と生成Asset Staging
- 解像度／aspect ratio／safe area／locale／入力方式／accessibilityを跨ぐUI Previewと構造化検証
- GameplayDefinition／C++ strategy planner
- `mirakan.systems.search／read／plan`、System Bundle生成、Definition／Native同値fixture
- Map intent resolver、World Authoring Bundle、compact 2D Level生成／Preview／手動変更保持／再編集
- AI Scale Planner、Gameplay fidelity floor、Representation Plan、Project固有Integrated Scale Fixture、Optimization Receipt
- GameplayDefinitionChangeSet、NativeCodeChangeSet、isolated validation／cook／build／test
- Engine-generated Diff、Approval、手動変更との競合処理
- Playtest feedbackと自動修復
- `AuthoringContextPlanV1`からのbounded retrieval、24,000 token初回Context Gate、追加Resource read
- `RemediationV1`による最大2回の修復、同一blocking反復停止、Permission／Security／lock非修復
- Decision Ledgerの成立条件追跡、失効Preview、再確認
- NativeGameModule Capabilityを公開する前にA1を完成し、Source Worker、Promotion、Review ReceiptをTool catalogへ解放
- `DBG4_ai_diagnosis`: `AiDebugContextV1`、`DebugFindingV1`、Evidence ID付き仮説／反証／不足データ／最小追加計測、Reproduction Bundleからのsupervised修正提案、既存Authoring Gateway／Risk／Approval／Replay回帰への接続

完了条件は、大まかなprompt→必要質問→人間選択または一件限定の`おまかせ`委任による2D Visual Style確定→First Playable生成→AI修正→手動修正→AI再編集を一つのProject revision historyで完走し、Style Resolver EvalとStyle Validator gateを満たすことである。Title、Settings、HUD、Pause、ResultはAIと手動編集の往復、代表解像度／locale／入力方式／accessibility Preview、生成Assetの来歴確認を合格しなければならない。さらに、AI Authoring通常Caseの95%以上で初回Contextを24,000 input token以下、Blocking／High evidence recall 100%、存在しないStableIdの最終提出0、stale DecisionによるCommit成功0、repairable Caseの2回以内修復90%以上、禁止Categoryの自動修復0を3 run最悪値で満たす。Debugging Evalでは、根拠なしのvalidated cause、gap／redactionの誤解釈、recorded／current revision混同、presentation結果からauthoritative causeへの逆因果、再現不能な修正成功をすべて0件とし、承認済み修正のReplay＋回帰成功100%を満たす。MVP能力に含むNativeGameModule生成はA1 Gate合格後だけ成功とし、A1未完ならMVP-A完了を宣言しない。

### Phase 5: 外部Agent接続

- Read／Propose中心のMCP Adapter
- Codex Plugin
- Claude Plugin
- Headless validation／build／playtest
- CI統合

完了条件は、Codex／Claude CLIまたはDesktopから直接Commit権限なしで同じ2D Projectを安全に編集できることである。

### Phase 6: 3D First Playable MVP-B

- glTF core／Unlit／Emissive Strength／Texture Transform import、mesh、`realistic_basic` PBR、Forward+、shadow、sky／IBL／height fog
- Jolt Physics Adapter、Recast／Detour Adapter
- ozz-based sampling＋独自Animation Graph
- 3D UI、Audio、particle、camera
- C1 bounded Water、flat Water Volume、CPU降雪VFX、static snow mask
- 同じShooter Coreを使うsingle-player third-person compact shooter arena
- `3d_crowded_battle_v1`で大量instance、敵味方spawn、Physics、Navigation、Animation、VFX、Audio／Cameraを同時測定

完了条件は、`realistic_basic` 3D縦切りを自然言語と手動編集の両方で作成でき、Khronos core／Unlit／Emissive Strength／Texture Transform material fixture、Reference stress scene、`3d_crowded_battle_v1`、1080p60、memory／queue／VFX budget、authoritative drop 0、Replay一致を満たすことである。

### Phase 7: Mobile Platform

- `android_mobile_v1`: GameActivity、Vulkan 1.1／AVP 2022、VMA、GameActivity input／text、Oboe、Swappy、AAB／PAD
- `apple_mobile_v1`: UIScene／MTKView、Metal／MTLHeap、UIKit touch／text、GameController、AVAudioSession／AudioUnit、archive／TestFlight
- Portable HLSL→SPIR-V→MSLのoffline shader pipeline、ASTC／ETC2／BCnのTarget別cook
- Device Manager、Apple Unsigned Build Worker／Signing Service／Upload Service、safe-area／cutout／orientation preview、touch simulation
- Mobile memory class、dynamic resolution、thermal governor、physical device matrix、16 KiB／privacy／Store package gate
- `DBG5_remote_shipping`: 認証済みDevice Debug handshake、bounded remote capture、Android Performance Analyzer／Perfetto／AGI／Vulkan Validation、Apple Metal validation／Instruments連携、offline bundle回収、Shipping instrumentation／secret混入scan
- Phase 3のWindows 2D C1を入力にAndroid 2D→Apple 2D、Phase 6のWindows 3D C1を入力にmobile 3D品質の順で成立させる

完了条件は、同一Project revisionの2D First PlayableがAndroid minimum／reference実機とA12 iPhone／iPadでplay、save、suspend復帰でき、Build／Signing／Uploadの秘密分離を証明した署名済みinternal track／TestFlight package、memory／frame／thermal／privacy gateを満たすことである。さらに、失敗時のpartial captureをgap付きで回収し、同じSession／Build／Project revisionへ再結合でき、Shipping packageからvalidation layer、IDE attach、raw trace、source path、credentialを除外できなければならない。

署名済みinternal track／TestFlight packageの作成前にA3 Releaseを完成する。A2 Engine Maintenanceは別Repository／別AuthorizationのEngine製品開発Profileにだけ存在し、Game制作MVP-A／MVP-BへTool、Path、Approvalを公開しない。

### Phase 8: Production CapabilityとDomain Pack

- Hybrid deferred path
- `realistic_advanced` Material、Skin／Hair／Eye／Cloth template
- `toon_basic`、`toon_character`、Art Asset／Animation presentation、inverted-hull／screen-space outline
- `pixel_diorama`のhigh-resolution 3D＋crisp sprite modeとunified low-resolution mode
- Multiple light、physically based atmosphere、volumetric fog／cloud
- L2 `ShadowGraphV1`、static／dynamic cache、PCSS／contact-hardening、Capability Gate後のWindows High Virtual Shadow
- Baked lightmap、irradiance／reflection probe、C3 dynamic GI研究
- GPU VFX、C2 Water Body／Query／Underwater、dynamic snow field、terrain、foliage、streaming
- Shooter C2 Feature、2D Action、FPS／TPS、RPG／Action RPG、Quest、Simulation、Strategy Pack
- `UiEffectGraph`と承認済みProject C++ `UiNativeWidget`
- 画像／音声／3D生成Provider adapter
- 自動Playtestとperformance regression

各CapabilityはAuthoring schema、Validator、Editor、AI command、Runtime compiler、Diagnostics、Test、fallback、VisualStyleProfile integrationの完了定義を満たしてからProduction扱いにする。ShadowはL0 Intent、L1 Profile、L2型付きGraph、L3 Project Techniqueへ段階化し、L0／L1でAIにBackend設定を推測させず、ResolverがTarget、Style、budgetから説明可能なPlanを生成する。L2は閉じたNode catalogとoffline compile、L3とHardware Ray Traced ShadowはStable Shadow Extension SDK、A1、R3 Promotion、全Targetまたは明示fallback、専用C3 Gate前に公開しない。`UiEffectGraph`は閉じたNode catalogとoffline compile、`UiNativeWidget`はA1、R3 Promotion、GameHost隔離、callback budget、accessibility semantics、fallbackを満たしたものだけをC2へ昇格する。第三者binary Widget、Marketplace、Editor processへのProject／外部Widget loadは別Threat Modelを持つC3まで対象外とする。ToonとPixel Dioramaは同時実装せず、Realistic advanced→Toon→Pixel Dioramaの順に個別vertical prototypeとperformance gateを通す。

### Phase 9: 制限付きRuntime生成

- Runtime専用SchemaとThreat Model
- Server-authoritative Gateway
- Allowlist Command
- Quota／Timeout／Fallback
- Dialogue、Quest outline、Encounter、NPC intent、bounded World構成
- Multiplayer revision／tick検証はNetwork設計承認後

任意C++、汎用Game bytecode、raw Physics／Rendering state、client-side Commitは許可しない。

## 15. MVPベースライン

MVPはAI Authoringの安全な往復を証明する製品縦切りであり、Engine機能を網羅する版ではない。開発上の最小成立点を**MVP-A**、2D／3D両対応の基盤成立点を**MVP-B**として混同しない。

- MVP-A: Phase 4完了。共通Shooter Coreを使う2D top-down shooterでAI Authoring全loopを証明する。
- MVP-B: Phase 6完了。同じShooter Coreを使う3D compact third-person shooter arenaを追加し、共通基盤が2D専用設計でないことを証明する。
- 最初のTechnology Preview配布条件はMVP-BとA3 Releaseの両方の完了とする。

詳細なSubsystem範囲と性能条件は2D／3D機能計画で固定する。

### 15.1 MVPの目的

「大まかなプロンプトから質問を経て、短いが最初から最後まで遊べるゲームを生成し、AI会話と手動編集の両方で安全に修正できる」ことを実証する。

### 15.2 MVPに含める能力

1. 自由な初回プロンプト
2. Blocking／High Impact要件の質問
3. Game Brief
4. 最小GameSpec
5. GameChangeSet
6. 構造化Scene／Rule編集
7. 一種類以上のGameplayDefinition生成
8. 一種類のNativeGameModule Capability生成
9. AIによるGameplayDefinition／C++選択
10. 隔離Validation／Cook／Compile／Test
11. Engine-generated Diff
12. Approval／Commit
13. Undo／Replay
14. AI会話による再編集
15. 手動Scene／GameplayDefinition／C++編集の検出
16. Base revisionによる競合防止
17. 最小First Playableの実行

### 15.3 MVP First Playableの形

2Dと3Dの各First Playableは、次の共通成立条件を満たす。

- Titleから開始できる
- Playerが操作できる
- 一つのCore loopが成立する
- 一種類以上の敵、課題、またはSimulation対象が存在する
- 一つの目標を達成できる
- Resultまたは終了状態へ到達できる
- Save／LoadまたはCheckpointが動作する
- AI生成GameplayDefinitionがゲーム挙動へ使われる
- AI生成NativeGameModule Capabilityが一つ使われる
- 人間が一箇所を手動修正し、AIがその変更を保持して追加編集できる

### 15.4 MVPから除外する範囲

- 商用品質の全Asset生成
- 複数ジャンルの同時対応
- Multiplayer
- RuntimeでのAI生成
- Engine CoreのAI自動変更
- Plugin Marketplace
- 複数Model Providerの本番Routing
- 大規模Open World
- Production品質のvolumetric cloud／GI／terrain／foliage
- Ray tracing必須化
- 完全自動リリース
- AIによる無承認Commit

### 15.5 MVP成功条件

- 初回プロンプトからGame Briefを生成できる。
- 重要な不足を質問し、低影響事項を記録付きで補完できる。
- 承認済みGame BriefからFirst Playableを生成できる。
- 生成Projectが固定ToolchainでCompileできる。
- First Playableを開始から終了まで操作できる。
- GameplayDefinition生成とC++ Capability生成が隔離環境で検証される。
- AI変更をGame／GameplayDefinition／C++ごとにDiff表示できる。
- 手動変更後のAI再編集で、人間の変更が無条件に消えない。
- 大量制作要求をScale intentとGameplay fidelity floorへ分解し、Target別Representation Planと統合負荷fixtureを生成できる。
- 大量配置、spawn、敵味方VFXの同時runで、実測Receiptなしに最適化済みと表示せず、Gameplay変更を無承認で性能調整に使わない。
- ChangeSetをUndoおよびReplayできる。
- 無効なAI出力がWorld ModelへCommitされない。
- Blocking／High要件とlock済みDecisionを欠落させず、通常Caseの95%以上を24,000 input token以下の初回Contextで処理できる。
- AIが正規Authoring JSONを直接変更せず、全`ai_mutable` fieldをtyped Operationで編集できる。
- repairable failureを初回後2回以内で90%以上修復し、同一blocking反復、Permission、Security、lock、revision driftでは自動停止する。
- Decision成立条件の変更をPreviewし、失効または再確認なしにstale Decisionを根拠としたCommitを成功させない。

## 16. 確定事項と実装計画への引渡し

### 16.1 確定事項

| 項目 | 決定 |
|---|---|
| 最初の縦切り | `mirakan.feature.shooter_core.c1`を使う2D top-down shooter |
| 第二の縦切り | 同じShooter Coreを使う3D single-player third-person compact shooter arena |
| Editor Host | Windows 11 25H2以降 x64（OS build 26200以上） |
| Game Target | `windows_desktop_v1`、`android_mobile_v1`、`apple_mobile_v1`。Mobile Editor、TV／Wear／XR、tvOS／visionOSは初期対象外 |
| Graphics | WindowsはD3D12、AndroidはVulkan 1.1＋AVP 2022、AppleはMetal。C1はForward+、desktop C2でHybridを選択可 |
| Game executable language | C++23。CX0は非Shipping Header bootstrap、最終方式はNamed Modules＋`import std`、C++26はreadiness CIだけ |
| Game implementation | C++実行コード＋GameplayDefinition。Authoring dataはoffline Cookし、C++ evaluatorが実行 |
| Game System model | 契約固定・実装開放型。Engine標準、Project定義、Extensionを`GameSystemSpecV1`、一意State owner、typed Port、Catalog、Implementation Plan、System Bundle、Qualificationへ収束 |
| World／Level／Map model | World／Scene／Level／Cellを分離し、Source IntentからTarget別Streaming PlanをCookする。「Map」は6分類へ解決し、曖昧または高影響なら質問する |
| General Game Script VM | 不採用。汎用bytecode interpreter、JIT、Game向けFFIを設けない |
| 2D Physics | 独自Physics Platform＋Box2D 3.1.1 private Adapter。KernelはTarget別Qualification後にProduction昇格 |
| 3D Physics | 独自Physics Platform＋Jolt Physics 5.6.0 private Adapter。C1はCPU rigid bodyだけ、Target別Qualification後にProduction昇格 |
| Collision contract | Body／Collider分離、32 Channel、typed Query／Contact／Trigger、immutable Cooked Asset、Collider Editing Mode、AI Typed Operation |
| 3D Navigation | 独自Navigation契約＋交換可能Backend。C1基準BackendはRecast／Detour 1.6.0、標準32-bit ref、1,024 tile×4,096 polygon |
| GPU memory | D3D12MA 3.2.0、VMA 3.3.0、Metal `MTLHeap`を各Adapter内で利用 |
| Build | Build GatewayをEditor／AI／CLI／CIの唯一の製品入口、CMakeをFirst-party C++ Build定義の正本、NinjaをCMake生成DAGの実行器とする。Makefiles系を公式経路にしない。WindowsはNinja Multi-Config、AndroidはGradle `externalNativeBuild`からABI／Variant別Single-Config Ninja、Apple CX0はXcode、CX1以降のportable C++ Module graphはNinja Multi-Config、App shell／最終link／archiveはXcode 26.6。EditorはCMake File APIとBuild Receiptを使用し、生成Ninja fileを公開APIにしない |
| Performance | Desktopは1080p60、Ryzen 5 5600、16 GiB、RTX 3060 12 GB／RX 6600 8 GB、runtime CPU 2 GiB。Mobileは30／60 fps、Baseline process 1,024 MiB／GPU 384 MiBから実機class別 |
| Visual Style model | Scene dimension、Art Direction、Composition、Shading Modelを分離 |
| Material | 型付きMaterial IR、複数Shading Model、Engine-owned Target Binding Layout |
| 最初の2D Style | `pixel_2d`、640×360、32 PPU、integer scale |
| 最初の3D Style | `realistic_basic` |
| Production Style順 | Realistic advanced→Toon→独自`pixel_diorama` |
| Style決定 | 明示要件優先、Genre単独決定禁止、High Impactは人間選択または一件限定`allow_ai_select`委任 |
| AI経路 | 内蔵はModel API、外部Hostはlocal MCP。Source保守はProvider APIまたは`ExternalClientSecurityProfile`合格済みManaged CLI Agent Host＋隔離Worker。非合格ClientはMCP Proposal専用。Pluginは任意の補助UX |
| AI Orchestrator | Node.js 24.18.0 LTS／TypeScript 7.0.2 strict、別Process |
| 初期Provider | OpenAI Responses API、`gpt-5.6-sol`、reasoning `medium` |
| AI権限 | AI可視Taskと署名済みAuthorization Envelopeを分離。R0–R5、Proposal／Approval／Promotionを別Authorityにする |
| AI契約 | `/schemas/mirakan/`のMCDを正本とし、C++／TypeScript／Cooked binary descriptor／MCP／Provider Schemaを生成 |
| AI可読Authoring | `AuthoringContextIndexV1`→`AuthoringContextPlanV1`→`ContextPackV2`、bounded Scene Shard／Slice、revision付きR0 query、typed Operation coverage 100%、Decision invalidation、MCD `RemediationV1` |
| AI検証 | Contract、semantic、policy、build、test、budget、選択的TLA+、Eval、Review、ProvenanceをRisk別に適用 |
| IPC | ACL付きWindows named pipe、length-prefixed JSON-RPC 2.0 |
| Editor | Dock／resize／floating、保存可能な複数Workspace、AI Partnerをpin可能 |
| Compatibility | Pre-1.0 API／ABI互換なし。永続Projectだけoffline migratorで一方向移行 |
| Runtime連携 | Domain間の直接呼出し禁止。`RuntimeOrchestrator`が固定phaseでtyped command／event／snapshotを統合 |
| Runtime storage | 独自16 KiB archetype chunk＋SoA列。構造変更はtick boundaryだけ |
| Lifetime | generation handle、phase／epoch付きlease、Asset Registry単一所有、queue別`GpuSubmissionSerial` retire |
| Debugging／AI Diagnosis | Engine-owned Session／Event／Counter／Query／Snapshot／Causality／Replayを正本とし、IDE／GPU tool／OS traceはAdapterとして関連IDで接続する。AIはbounded evidenceから仮説と不足データを提示し、修正は既存Risk／Approval／Authoring Gateway／Replay回帰を必ず通す |
| Math／Core Utilities | `mirakan::foundation`と`mirakan::math`を分離。内部はcompact storage type、AI／Editor／Authoring／Save／Subsystem公開境界はunit／space／normalizationを持つsemantic type。portable scalar referenceを正解系にし、SIMD／Platform backendは実測とconformance後にprivate昇格 |
| Performance target | CPU／GPU P95 14.00 ms soft、16.67 ms hard。Subsystem別budgetと10分soak |
| Scale／optimization | 固定個数超過だけでSource intentを拒否しない。`RuntimeScaleIntentV1`、Gameplay fidelity floor、Target別Representation Plan、`OptimizationRequired`、Project固有Integrated Scale Fixtureで管理 |
| Authoring source of truth | Authoring Document集合＋単調増加`ProjectRevision`。UI、AI会話、Runtime Worldは正本ではない |
| NativeGameModule | Windows Developmentは別ProcessのGameHostが単一DLLを起動時に一度だけload、Shipping／Android／Appleは静的link。ABI越しにSTL／exception／allocator／native objectを渡さない |
| Asset model | Source／Import／Derived／Packageの四層、隔離Importer、content-addressed Derived Data、Asset Catalog／VFS、`.mirakanpack` |
| Renderer model | Immutable `RenderSnapshot`をextractし、Engine-owned Render GraphをcompileしてD3D12／Vulkan／Metal Adapterへ投入 |
| Water／Snow model | Water Surface／CPU Query、降雪VFX、Snow Surface、Gameplay Surface Stateを分離し、GPU presentation結果をPhysics／Gameplayへ逆入力しない |
| Editor model | Production Editorと初心者用`AI Creator` Workspaceを同一Document／ChangeSet上で提供。Panelはdock／resize／floating／multi-monitor／保存可能 |
| Player I/O | InputはAction→`InputSnapshot`、UIはretained typed document、AudioはEngine-owned mixer。Platform native APIはAdapter内だけ |
| Game text | UTF-8、HarfBuzz 14.2.1、FreeType 2.14.1、ICU4C 78.3。Editor shellのDirectWrite／TSFとは責務を分離 |
| Domain Pack | CoreはGenre非依存。2D Action／TPS／RPG／Simulationを初期Packとし、Shooter共通機能は`mirakan.feature.shooter_core.c1`＋2D／TPS Profileとしてcomposeする。multiplayer等は別GateまでCoreへ混入させない |

### 16.2 実装計画書で分解する事項

次は設計上の選択肢ではなく、47文書で確定した契約をfile、target、生成物、fixture、testへ割り当てる実装計画作成作業である。fieldとpolicyを実装担当者が再決定してはならない。

各実装taskは、対象file／target／public contract／依存taskに加え、基準fixture、Target Profile、測定metric、soft／hard Gate、Before／After比較、fallback、rollback、生成する`VerificationReceiptV1`、完了時に昇格できるCapability状態を明記する。実測不能なtaskは、機能実装taskへ混ぜずinstrumentationを先行taskにする。性能改善とbaseline緩和を同じ判断にせず、baseline変更は旧値、新値、理由、Reference環境、品質差、全下流Gate影響を持つ別Review対象にする。

1. Phase 0のrepository bootstrap、Build Gateway、CMake target DAG、`BuildDriverProfileV1`、Windows Ninja Multi-Config／Android Gradle→Single-Config Ninja／Apple Ninja–Xcode Preset、CMake File API query、Build Receipt、C++23 Header bootstrap、`mirakan_add_cpp_component()`、Named Module／`import std` Probe、`mirakan.build.ninja_adoption.v1` Gate、Miraikanai Contract Definition配置をtaskへ分解する。
2. 固定Toolchain／Dependency artifactの取得、hash lock、SBOM、offline CI image、更新Gateをtaskへ分解する。
3. `MATH0_contract`、`FOUNDATION0_core`、`MATH1_scalar_reference`から`MATH5_baseline`までを分け、`mirakan_foundation`／`mirakan_math`、storage／semantic type、Transform／Quaternion／projection、failure、C++／C ABI／Editor／TypeScript／MCP projection、unit／property／golden／CPU・HLSL conformance、portable baselineをtaskへ分解する。
4. Contract compiler、C++／TypeScript／binary descriptor／MCP／Provider projection、`game_system` meta-schema、System Catalog／Graph projection、`RemediationV1`、typed Operation coverage、Authoring query projection、round-trip／transition conformance testをtaskへ分解する。
5. Authoring Document Store、ProjectRevision、World／Scene／Level／Topology、Scene Shard／Slice、Context Index／Plan／Pack、Decision invalidation、ChangeSet transaction、System／World Bundle、journal／snapshot／crash recovery、100万Entity headless fixtureをtaskへ分解する。
6. RuntimeOrchestrator、fixed phase、typed command／event／snapshot、Game System一意State owner、Level／Cell lifecycle、queue、handle／lease、memory telemetry、Scale intent／Representation Plan、qualification status、Integrated Scale Fixtureをtaskへ分解する。
7. `DBG0_contract`から`DBG5_remote_shipping`までを分け、Debug Session／Event／Counter／Store／Query、priority／gap／retention、safe pause／step、Breakpoint／Watch、Replay／Rewind、Causality、Reproduction Bundle、Crash／Hang、外部IDE／GPU／OS Adapter、AI evidence diagnosis、remote device、security／privacy／overhead Gateをtaskへ分解する。
8. `LodIntentV1`、Domain別LOD Policy、CPU reference metric、Mesh／Sprite／Simulation／Animation／Material／VFX／Surface／Residency Plan、Render extraction、individual／instanced／spatial／presentation分類、CPU cull／LOD／instancing、Render Graph compiler、D3D12 Adapter、Vulkan／Metal interface、Shader／Material pipeline、reference captureを段階taskへ分解する。
9. Asset Import Worker、Derived Data、Catalog／VFS、Cook／`.mirakanpack`、AI Asset staging／provenanceをtaskへ分解する。
10. Editor document shell、dock／workspace、Scene／Outliner／Inspector／Asset／AI Partner／Debug Workspace、UI Automation bridge、recovery testをtaskへ分解する。
11. NativeGameModule C ABI、`GameSystemSpecV1` conformance、Target別Implementation Variant、static／dynamic link、isolated Build、GameHost restart、Promotion GateとC2 `UiNativeWidget` Capabilityをtaskへ分解する。
12. Input、UI／Text／Localization／Accessibility、AudioのC0契約、標準Widget＋`UiCompositeDefinition`による2D First Playable用C1 vertical slice、C2 `UiEffectGraph`を独立taskへ分解する。
13. Collision、独自Physics PlatformのWorld／Dynamics／Joint／Character／Save／Replay、Box2D／Jolt Kernel Qualification、独自Navigation PlatformのGrid2D／Backend Port／Artifact／status／Recast・Detour Qualification、ozz、Engine-owned Animation Graphをconformanceとvertical sliceへ分解する。
14. Water Source／Compiler／Render／CPU Query／Volume／Underwaterと、Weather Snapshot／降雪VFX／static・dynamic Snow Surface／stampをC1とC2の独立task、Gameplay分離fixture、Target別Gateへ分解する。
15. Windows MSIX／managed Directory package、Android AAB／PAD／16 KiB、Apple archive／signing／uploadをPlatform別taskと実機Gateへ分解する。
16. AI Orchestrator、bounded Context retrieval、最大2回repair loop、Requirement／Game System／Map Intent／World／Visual Style／UI Authoring／Scale Planner、Gameplay fidelity floor、Optimization Receipt、生成Asset Staging、Preview matrix、OpenAI Provider、local MCP、外部Client Security ProfileをRisk別taskへ分解する。
17. Source Worker、Path Broker、Promotion Service、Receipt署名、TLA+ model、16 AI Eval suite、`AiReadableAuthoringFixtureV1`、`VisualEffectRoutingFixtureV1` 96 Case、`VfxAiAuthoringFixtureV1` 360 Case、debugging diagnosis corpus／holdout、Provider／retrieval migration harnessを検証taskへ分解する。
18. 2D／3D compact Level、Portal／Streaming failure、System semantic equivalence、procedural connectivity、`2d_crowded_battle_v1`／`3d_crowded_battle_v1`、Project固有Integrated Scale Fixture、Water／Snow fixture、frame／memory／thermal／soak fixture、performance baseline、Milestone判定をtaskへ分解する。

将来の多ジャンル対応を理由に最初の縦切りを過剰に汎用化しない。各taskは「AI編集と手動編集の安全な往復」「Engine側検証」「playable result」のいずれかへ直接寄与しなければならない。

## 17. リスクと対策

| リスク | 対策 |
|---|---|
| 一括生成が巨大化する | 薄い全体＋深い代表部分、段階的ChangeSet |
| Contextが巨大または必要根拠を欠く | Context Index／Plan／Packを分離し、field mask、省略Manifest、追加read、24,000 token Gate、Blocking／High recall 100%を強制 |
| 巨大SceneをAIが全読込・全置換する | bounded Scene Shard、StableId検索、SceneSlice、部分Diff、100万Entity性能fixture |
| 質問が多すぎる | 影響度分類、「おまかせ」、段階別質問 |
| AIが誤った実装方式を選ぶ | Policy、Project Profile、Benchmark、Decision Record |
| AIが画風をGenreだけで決める | VisualStyleResolverの優先順位、Engine hard gate、High Impact承認 |
| 画風変更でMaterialだけが変わり不整合になる | StyleChangeSetでLighting、Camera、Post、VFX、UI、Importを一括検証 |
| 既存作品の映像表現を模倣する | 一般属性へ分解し、固有名を正規Profile／Shader／Assetへ保存しない |
| 構造化Gameplay Logicが遅い | Cook／index／layoutを先に最適化し、同一Contractとfixtureで有意な場合だけHot pathをC++化 |
| AIが手動変更を消す | Base revision、File hash、三者比較、Lock |
| AIがScreenshotだけでUI完成と誤判定する | typed Operation、layout／binding／focus／semantic／budget検証、複数Target Preview matrixを正規Gateにする |
| 独自WidgetがEngine境界を迂回する | Composite、Effect、Nativeの段階Capability、R3 Promotion、GameHost隔離、bounded callback、fallbackで閉じる |
| AI生成画像が未検証のままUIへ入る | Staging、provenance／license／safety、Target別import／cook、人間承認、missing fallbackを必須にする |
| Schema-validだが意味的に不正 | C++ Semantic validator、Budget、Dry-run |
| Diagnosticを読めず同じ修正を反復する | MCD Remediation、最大2回、同一blocking集合の減少確認、全Attempt Receipt |
| 古い設計判断をAIが正しい前提として使う | Decision成立条件、根拠hash、依存closure、明示Invalidate／Reconfirm、lock承認 |
| 生成C++が危険 | 隔離Build、Network制限、固定Toolchain、Test |
| Provider依存 | Canonical IR、Provider adapter、Eval |
| ProviderごとのSchema方言が異なる | MCDを正本にし、Provider別subsetと未表現Constraint Manifestを生成。Gatewayが完全再検証 |
| MCP外でCLIがFile／Shellを操作する | full-access起動を公式境界外と明示。Managed modeはClient conformance、Credential分離、OS sandbox、隔離Worktree、Network deny、Path解決、差分Promotionを必須化 |
| AIがTask本文で自己昇格する | Task内容とAIが変更不能な署名済みAuthorization Envelopeを分離 |
| 形式モデルが実装保証に見える | Model–transition mappingと生成Conformance Testを併用し、C++全体の証明とは表現しない |
| 最新Model／SDKで無通知回帰する | exact Provider Manifest、3回Eval、canary、明示昇格、rollback |
| チャットが長くなり決定を失う | Decision Ledger、GameSpec、Project memory |
| 初心者に技術質問をする | ゲーム上の要件へ翻訳、内部方式はAIが判断 |
| 自由度を上げると安全性が下がる | GameplayDefinitionとC++で権限、表現力、検証、Build境界を分離 |
| 既存Engineの模倣になる | 独自原則を先に固定し、比較調査は検証だけに使う |
| 所有権が曖昧でleak／use-after-freeが起きる | RAII、unique ownership、borrow規則、generation handle、ASan |
| D3D12 resourceをGPU使用中に解放する | queue別fenceを持つdeferred release |
| Allocatorを自作して逆に遅くなる | `pmr`境界、domain telemetry、実測したhot pathだけ専用化 |
| GPU memoryのthrashing | OS budget監視、residency priority、streaming、明示的な失敗 |
| Clean実装の名目でProject dataを失う | Runtime互換分岐は持たず、backup付きoffline migratorだけを提供 |
| Module境界が崩れVendorへ固定される | CMake dependency DAG、Ports／Adapters、CX0 Public Header／CX3 Module interfaceのVendor型・依存scan |
| Make／Ninja二重対応でBuild結果が分岐する | `BuildDriverProfileV1`をclosed set化し、First-party Makefiles／`ndk-build`をPromotion対象外にする |
| Ninjaだけで製品Build全体を表現してAsset／Package／署名が混線する | Build Gateway、CMake、Ninja、Content Build、Gradle／Xcode／Signingの責務を分離し、Ninja成功だけではPromotionを許可しない |
| 増分Buildが速くても依存漏れで古いartifactを使う | generated Input／Output／Depfileを完全宣言し、clean／incremental hash一致、mutation、interrupt recoveryを`mirakan.build.ninja_adoption.v1` Gateで検証する |
| Subsystemが相互に直接変更して順序依存になる | RuntimeOrchestrator、固定phase、typed command／event、非再入配送 |
| Hot reloadで新旧Assetが混在する | dependency closure全体をstagingし、boundaryでatomic promotion |
| 非同期jobが破棄済みobjectへ書く | handle＋versionを開始時と統合時に再検査し、stale resultを破棄 |

## 18. 調査から得た位置付け

調査上、API、MCP、Plugin、CLIは代替関係ではなく異なる層である。

- Model APIは推論を提供する。
- Agent SDKは高度なAgent loopを管理する。
- MCPはAI HostとEngine Toolを接続する。
- Codex／Claude Pluginは任意のworkflow／MCP設定／UX配布層であり、必須契約や権限の正本ではない。
- CLI／DesktopはEngine開発と試作に適する。
- どの層もEngineのSemantic validationとTransactionを代替しない。

Unity、Unreal Engine、Godot、O3DEからは、Retained tree、独自C++ Framework、Engine UI共有、third-party toolkit方式の利点と制約を比較するための一般的教訓だけを得る。Widget API、宣言構文、Node model、画面配置、serialization、プロダクト固有のEditor UXは採用しない。MirakanUiの具体的な独自性と一次資料は独自Editor UI Framework規約を正本とする。

## 19. 参考一次資料

- OpenAI Plugins: https://learn.chatgpt.com/docs/build-plugins
- OpenAI Responses／Agents: https://developers.openai.com/api/docs/guides/agents
- OpenAI Structured Outputs: https://developers.openai.com/api/docs/guides/structured-outputs
- OpenAI SDKs and CLI: https://developers.openai.com/api/docs/libraries
- OpenAI Using Tools: https://developers.openai.com/api/docs/guides/tools
- OpenAI GPT-5.6 Sol: https://developers.openai.com/api/docs/models/gpt-5.6-sol
- Claude Code Plugins: https://code.claude.com/docs/en/plugins
- Claude Agent SDK: https://code.claude.com/docs/en/agent-sdk/overview
- Claude Structured Outputs: https://platform.claude.com/docs/en/build-with-claude/structured-outputs
- MCP Architecture: https://modelcontextprotocol.io/specification/2025-11-25/architecture
- MCP Tools: https://modelcontextprotocol.io/specification/2025-11-25/server/tools
- Unity AI: https://unity.com/blog/unity-ai-how-to-get-started
- Unreal MCP: https://dev.epicgames.com/documentation/unreal-engine/unreal-mcp-in-unreal-editor
- Godot EditorPlugin: https://docs.godotengine.org/en/stable/tutorials/plugins/editor/making_plugins.html
- C++ Core Guidelines: https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines
- Windows 11 Release Information: https://learn.microsoft.com/en-us/windows/release-health/windows11-release-information
- Visual Studio 2026 Release History: https://learn.microsoft.com/en-us/visualstudio/releases/2026/release-history
- MSVC Build Tools 14.51 GA: https://devblogs.microsoft.com/cppblog/msvc-version-1451-available/
- MSVC C++ language standard: https://learn.microsoft.com/en-us/cpp/build/reference/std-specify-language-standard-version
- CMake 4.4: https://cmake.org/cmake/help/v4.4/release/4.4.html
- CMake C++ Modules: https://cmake.org/cmake/help/v4.4/manual/cmake-cxxmodules.7.html
- CMake Ninja Multi-Config: https://cmake.org/cmake/help/v4.4/generator/Ninja%20Multi-Config.html
- CMake IDE Integration／File API: https://cmake.org/cmake/help/v4.4/guide/ide-integration/index.html
- Ninja Manual: https://ninja-build.org/manual.html
- Node.js 24.18.0 LTS: https://nodejs.org/en/blog/release/v24.18.0
- TypeScript 7.0: https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/
- Direct3D 12 Programming Guide: https://learn.microsoft.com/en-us/windows/win32/direct3d12/directx-12-programming-guide
- Direct3D 12 Memory Management Strategies: https://learn.microsoft.com/en-us/windows/win32/direct3d12/memory-management-strategies
- D3D12 Memory Allocator: https://github.com/GPUOpen-LibrariesAndSDKs/D3D12MemoryAllocator
- vcpkg Manifest Mode: https://learn.microsoft.com/en-us/vcpkg/concepts/manifest-mode
- Box2D: https://box2d.org/documentation/
- Jolt Physics: https://jrouwe.github.io/JoltPhysics/
- Recast Navigation: https://github.com/recastnavigation/recastnavigation
- glTF 2.0 Specification: https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html

## 20. 次のアクション

設計文書Indexに列挙した47文書の内部整合レビューは完了した。次はユーザーが[統合計画サマリー](./README.md#0-統合計画サマリー)、[不変Engine境界・初心者向けAI技術承認規約](./2026-07-21-immutable-engine-beginner-ai-approval-design.md)、本書16章の確定事項を入口にReviewし、修正点または承認を返す。承認後、実装task、依存関係、Test、Milestone、性能Gate、完了条件を含むPhase 0実装計画書を別文書として作成する。

実装計画はPhase 0 Foundationから開始し、Windows 2D First Playable、Windows 3D First Playable、Android／Appleの順序付きmobile vertical sliceへ分解する。承認前にEngine実装へ着手しない。
