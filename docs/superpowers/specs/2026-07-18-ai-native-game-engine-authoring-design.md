# AIネイティブ独自ゲームエンジン 設計計画書

- 文書版: 1.6
- 作成日: 2026-07-18
- 最終更新日: 2026-07-20
- 対象: 独自C++ゲームエンジン、独自Editor、AI制作基盤
- 状態: 基本構想とSubsystem別正式仕様29文書を統合した内部整合レビュー版。ユーザー承認待ち
- 設計文書Index: [Miraikanai Engine 設計文書Index](./README.md)
- Game実装規約: [Miraikanai Engine C++実行コード・構造化ゲームデータ規約](./2026-07-19-cpp-structured-game-data-design.md)
- C++言語・Modules規約: [Miraikanai Engine C++23・Named Modules・`import std`移行規約](./2026-07-20-cpp23-modules-import-std-transition-design.md)
- Authoring状態規約: [Miraikanai Engine Authoring Model／Project State規約](./2026-07-19-authoring-model-project-state-design.md)
- Native Game規約: [Miraikanai Engine NativeGameModuleアーキテクチャ規約](./2026-07-19-native-game-module-architecture-design.md)
- Renderer規約: [Miraikanai Engine Rendering／Render Graphアーキテクチャ規約](./2026-07-19-rendering-render-graph-architecture-design.md)
- Particle／VFX規約: [Miraikanai Engine 独自Particle／VFX Platformアーキテクチャ規約](./2026-07-20-particle-vfx-architecture-design.md)
- Asset規約: [Miraikanai Engine Asset Pipeline／Content Package規約](./2026-07-19-asset-pipeline-content-packaging-design.md)
- Editor規約: [Miraikanai Engine Editor／Workspace／UX規約](./2026-07-19-editor-workspace-ux-design.md)
- Editor UI Framework規約: [Miraikanai Engine 独自Editor UI Framework／Shellアーキテクチャ規約](./2026-07-20-editor-ui-framework-architecture-design.md)
- Player I/O規約: [Input](./2026-07-19-input-action-device-architecture-design.md)／[UI・Text](./2026-07-19-ui-text-localization-accessibility-design.md)／[Audio](./2026-07-19-audio-mixer-spatial-architecture-design.md)
- Physics Engine規約: [Miraikanai Engine 独自Physics Platform／Dynamicsアーキテクチャ規約](./2026-07-20-physics-engine-architecture-design.md)
- Navigation規約: [Miraikanai Engine 独自Navigation Platformアーキテクチャ規約](./2026-07-20-navigation-platform-architecture-design.md)
- Simulation連携規約: [Physics／Navigation／Animation連携](./2026-07-19-physics-navigation-animation-architecture-design.md)
- Platform規約: [Windows](./2026-07-19-windows-platform-distribution-design.md)／[Mobile](./2026-07-19-mobile-platform-architecture-design.md)
- 将来Capability規約: [Miraikanai Engine Domain Pack／将来Capability規約](./2026-07-19-domain-pack-future-capability-roadmap.md)
- Collision詳細規約: [Miraikanai Engine Collision／Colliderアーキテクチャ規約](./2026-07-19-collision-collider-architecture-design.md)
- AI実装・保守規約: [Miraikanai Engine AI実装・保守ガバナンス規約](./2026-07-19-ai-engine-development-governance-design.md)
- 実行可能契約規約: [Miraikanai Engine 実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)
- AI検証規約: [Miraikanai Engine AI検証・評価・来歴規約](./2026-07-19-ai-verification-evaluation-provenance-design.md)

## 0. 統合レビュー結果

本書は、これまで個別に確定した要求を一つのProduct計画として束ねるマスター計画書である。詳細契約はSubsystem別正式仕様へ分離し、本書と各仕様を一つのReview setとして扱う。29文書を単一巨大文書へ統合せず、次の三層を維持する。

1. 本書がProduct vision、制作体験、Milestone、MVP、全体の依存順を決定する。
2. [設計文書Index](./README.md)が読む順序、決定権、要求トレーサビリティ、Review状態を決定する。
3. Subsystem別正式仕様が型、phase、lifetime、budget、failure、test、Definition of Doneを決定する。

2026-07-20の内部整合レビューでは、会話で確定した要求が次の閉じた設計へ収束していることを確認した。

| 領域 | 統合判断 |
|---|---|
| AIとEngineの境界 | AIはGame Brief、GameSpec、GameplayDefinition、C++、Test、ProjectChangeSetを提案できるが、Engine object、pointer、GPU resource、ProjectRevisionを直接変更できない |
| Authoring | 自然言語、Editor GUI、外部IDE、MCPは同じAuthoring DocumentとProjectChangeSetへ収束し、C++ GatewayだけがCommitする |
| Game実装 | C++23実行Code＋構造化GameplayDefinition。Luauを含む汎用Game Script VMは持たない |
| Editor | C++23の独自`MiraUI Core`＋`MiraEditor Shell`で実装するProjection Editor。Retained UIと限定typed Immediate Canvasを併用し、禁止GUI toolkit、screen-coordinate AI操作、Projectへの直接writeを持たない |
| AI統合 | 内蔵はProvider API、外部Hostはlocal MCP、Pluginは任意UX。Source変更は隔離WorkerとPromotionを必須にする |
| Engine機能 | 2D／3D、Renderer、Asset、Collision、Physics、Navigation、Animation、Input、UI／Text、Audio、VFX、環境表現をSubsystem契約として分離する |
| Physics | 独自World／Body／Joint／Character／Command／Save／AI契約をC++23で所有し、Box2D 3.1.1／Jolt 5.6.0をprivate kernel候補としてTarget別Qualification後にProduction昇格する |
| Particle／VFX | 単一の型付きVFX Asset／Graph IRを2D／3D・CPU／GPU専用Artifactへoffline compileする。初心者Stack、上級者Graph、AI編集は同じSourceのProjectionとし、Particle結果をauthoritative Gameplayへ逆入力しない |
| Visual表現 | Realistic、Toon、2D、独自Pixel DioramaをVisual Style、Material、Shader、Quality Profileの組合せとして解決する |
| Platform | Windows／D3D12を先行し、Android／Vulkan、Apple／Metalを同じPortへ接続する |
| Build | CMakeがC++ Build定義、NinjaがC++ Build executor、Build Gatewayが製品入口。Android packageはGradle、Apple app／archive／署名はXcodeが所有する |
| C++進化 | C++23をShipping基準とし、Named Modules＋`import std`へ一方向移行する。C++26はreadiness CIで先行検証する |
| 実装順 | Foundation→Headless Authoring→Editor／Runtime→2D Manual→AI MVP-A→外部Agent→3D MVP-B→Mobile→Production→制限付きRuntime生成 |

このレビューで新たな製品機能は追加していない。Ninjaを含むBuild層の責務、EditorからのCMake File API利用、増分Buildの正当性／性能Gateを明示し、「Build Toolの採用」と「製品Build architecture」を混同しないようにした。

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
| Level 3 | NativeGameModule（Project C++）、Engine Extensionの生成・直接編集 | 上級者、Engine開発者 |

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

Authoring Serviceが保持するAuthoring Document集合のうち、`WorldDocument`、`SceneDocument`、Entity／Component／CompositionをWorld Modelと総称する。Editorが独自に所有する一枚のobject graphではなく、Scene Tree、Inspector、AI要約、Runtime Worldを正規状態にしない。

Commit済み`ProjectRevision`の同じDocument集合をScene、Graph、表、UI、Timeline、Simulation MonitorなどへProjectionする。Document種類、header、ID、参照、保存、Recoveryの詳細はAuthoring Model／Project State規約を正本とする。

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

各種別は共通してRevision、Diff、承認、履歴を持つが、検証・適用方法は分離する。

### 6.5 Gameplay Capability Contract

GameSpecとGameplayDefinitionは具体的なC++ symbolではなく、version付きCapability ID、typed command／event／snapshotを参照する。

既存Capabilityの組合せで表現できない機能、または同一fixtureの計測で構造化実装がPerformance要件を満たさない機能は、同じ公開Contractを維持した`NativeGameModule`へ実装できる。呼び出し側のGameSpec、Save field、他Systemを全面変更しない。

### 6.6 Decision Ledger

設計判断の出所と理由を記録する。

```text
value
source: human | ai_recommendation | ai_assumption | engine_default
reason
confidence
approved
locked
dependencies
created_revision
```

チャット履歴そのものではなく、確定要件、仮定、却下案、未解決事項、ロック、承認を構造化して保存する。

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
- 最大Entity数
- Network要件
- Offline要件
- Mod対応
- Buildサイズ
- Input方式
- Accessibility
- Privacy／Retention policy
- Mobile orientation／resize／safe-area／touch fallback policy
- Permission、Content Delivery、Content Safety policy

## 7. システムアーキテクチャ

本章はProduct全体の論理構成を定義し、field、phase、lifetime、budget、failure、testの詳細は各正式仕様へ委譲する。

- C++ module依存、所有権、Build、directory、Dependencyは[基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)を基準とする。
- 正規Project状態、ProjectRevision、ChangeSet、journal、recoveryは[Authoring Model／Project State規約](./2026-07-19-authoring-model-project-state-design.md)を基準とする。
- Game実行コードと構造化ゲームデータの境界は[C++実行コード・構造化ゲームデータ規約](./2026-07-19-cpp-structured-game-data-design.md)、Native ABI、Target別link、Build／Promotionは[NativeGameModule規約](./2026-07-19-native-game-module-architecture-design.md)を基準とする。
- Runtime phase、Subsystem連携、borrow無効化、Asset version、memory／performance budget、障害復旧は[Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)を基準とする。
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

正規四軸、VisualStyleProfile、Material／Shader、Toon、Realistic、2D、Pixel Diorama、Preview、Validator、Evalの詳細は、[2D／3D機能計画 §8](./2026-07-19-2d-3d-capability-plan.md#8-visual-styleshadermaterial)を唯一の基準とする。

### 7.6 Implementation Strategy Planner

システム単位でGameplayDefinition、C++、または型付きCapability境界での併用を選択する。LLMの推測だけで決めず、Engine Policyと実測を組み合わせる。汎用Game scripting runtimeは候補へ含めない。

### 7.7 MCP Adapter

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

### 7.8 Codex／Claude Plugin

Host固有Pluginは任意の薄い補助UXとする。PluginがなくてもProvider APIとMCPで全公式機能を利用できなければならない。

- Engine固有skill
- GameSpec／ChangeSet説明
- Coding／Asset規約
- Build／Test手順
- MCP起動設定
- 検証・修復workflow

権威ロジックやEngine状態は保持しない。

### 7.9 CLI／Desktop

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

未提供Gameplay Capability／新規Algorithm
  → NativeGameModule

複数Projectで再利用するEngine-wide Capability
  → Engine ExtensionとしてR4 Review

Platform／Native SDK統合
  → Engine Adapter変更としてR4 Review

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
| NativeGameModule | 隔離Workspaceで生成・修正可能 | Primary／secondary Compile、Static analysis、Unit、Integration、Capability conformance |
| Engine／Platform Adapter | 明示的なR4変更 | Dependency、Security、Platform／Store、全体Test、Domain owner、独立Review |
| Engine Extension | 明示的なR4変更 | 全体Test、current API conformance、Domain owner、独立Review |
| Engine Core | R4 Proposalと隔離Patch | Threat／lifetime analysis、完全Review、回帰・fault・soak |

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
2. Strict Schema parse
3. Canonicalization
4. Stable ID／Reference
5. Semantic invariant
6. Capability／Authorization
7. Resource budget
8. Base revision／Conflict
9. Staging／Dry-run
10. Engine-generated Diff
11. Risk policy／Approval
12. Atomic Commit
13. Undo／Journal／Receipt

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
- Character／Combat共通
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

AIはインストール済みCapabilityだけを利用できる。不足する場合は、GameplayDefinition composition、NativeGameModule、Engine Extensionの順に選択し、NativeGameModuleはR3、Engine ExtensionはR4の承認を必須とする。

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

### Phase 0: Foundation契約とToolchain

- 設計文書Indexに列挙した公式Review set 29文書の承認
- C++23共通Runtime Contract、`CxxFrontendProfileV1`、`CppDependencySetV1`、`BuildDriverProfileV1`と`windows_desktop_v1`、`android_mobile_v1`、`apple_mobile_v1`のTarget Profile schema
- Windows 11 25H2以降 x64／Direct3D 12を最初に実装し、Android Vulkan／Apple Metalを同じGraphics Portへ接続する境界
- Windows、Android、Apple toolchainをprofile別に固定する`toolchain.lock.json`と、Store要件を14日以内かつSubmission 7日前以内に再確認する`store_policy.lock.json`
- vcpkg manifest／固定Dependency
- Build GatewayをEditor／AI／CLI／CIの唯一の製品Build入口とし、CMakeをC++ Build定義、Ninjaを生成DAG実行器、Gradle／XcodeをPlatform package ownerへ固定
- CMake Presets／File API、Engine-owned Build Receipt、生成Ninja file非公開、Target／Configuration／Toolchain別Build tree分離
- `VerificationReceiptV1` gate `mira.build.ninja_adoption.v1`によるclean／no-op／leaf変更／Module変更／generated Header変更／中断復旧／memory／artifact一致の実測
- Module dependency DAGとComposition Root
- `mira_runtime_contracts`、`mira_runtime_package`、`RuntimeOrchestrator`、Domain Port／Runtime／Adapter境界、`ComponentAccessManifest`
- RAII、所有権、generation handle
- 12段階fixed tick、8段階render frame、typed command／event、bounded queue
- CPU memory domain、GPU allocator、residency、submission-deferred release
- Result／Error、thread affinity、borrow epoch、Asset version／atomic promotion
- Naming、format、static analysis、sanitizer
- 2D／3Dの座標、単位、色、時間規約
- Scene dimension、Art Direction、Composition、Shading Modelの正規四軸
- Lifecycle、Display、Graphics、Input、Audio、Text、Content Delivery、Thermal／MemoryのPlatform Port
- `/schemas/mira/`のMCD meta-schema、Contract compiler、C++／TS／Cooked binary descriptor／MCP／Provider projection
- `TaskSpecification`、署名済み`TaskAuthorizationEnvelope`、`ContextPack`、Provider Manifest、R0–R5
- Source Worker／Path broker／Network／PromotionのMCD契約、`NotActivated`拒否stub、security fixture。実WorkerはA1で解放
- TLA+／TLC v1.7.4で検査する5 State machineのmapping。Phase 0ではA0とFoundation実装に該当するModel、残りは各Activation前に実行
- Requirement Coverage、AI Eval、Verification／Generation ReceiptのA0実装と、Review／Promotion Receipt、SBOM／provenanceのSchema境界

完了条件は、空のWindows EditorHost／Windows GameHost／WorkerHostが固定toolchainでBuildでき、Android／Appleの未実装Adapterが偽の成功ではなく`UnsupportedTarget`を返し、Foundation contract、Target Profile、phase順序、handle／borrow、bounded queue、memory failure、Asset atomic promotionのtest、ASan、format、static analysisがCIで成功することである。さらに、同一MCDからのA0用Projectionが決定論的に生成され、AIが変更不能なAuthorization Envelope、未解放Source Workerの`MIRA-POLICY-CAPABILITY_NOT_ACTIVATED`拒否、Provider再検証、該当TLA+ fast model、AI security negative fixture、Verification／Generation Receipt hash chainがheadless CIで合格しなければPhase 1へ進まない。

Phase 0で全Risk classの契約と拒否動作を定義するが、製品機能として解放するのはAI実装・保守ガバナンス規約のA0（R0–R2）だけである。A1のProject Source、A2のEngine保守、A3のReleaseは各Activation Gate完成後にTool catalogへ追加し、未完成機能を弱い検証で先行開放しない。

### Phase 1: Headless Authoring Core

- GameSpec、World Model、ChangeSet
- Stable ID／Project Revision
- Capability、`AuthoringCommandGateway`
- Schema／Semantic／Budget validation
- Dry-run、Diff、Atomic Commit
- Transaction、Undo／Redo、Journal
- Save／Load、Replay
- Versioned schemaとoffline Project Migrator
- VisualStyleProfile、StyleChangeSet、StyleCapabilityManifestのschema

完了条件は、手書きChangeSetをheadlessでvalidate→stage→commit→save→load→replayし、同じstate hashを得られることである。

### Phase 2: Editor Shellと共通Runtime

- 独自MiraUI Core、Retained tree、Layout、Event、Focus、Semantic、D3D12 UI pass
- Dock／resize／floating／multi-workspace Editor shell
- Scene／Canvas、Outliner、Inspector、Asset、Diff／History、AI Partner panel
- Windows window／GameInput／XAudio2 Adapter、Editor shell用DirectWrite／TSF／UI Automation／OLE
- Human、keyboard、assistive technology、AIを同じtyped Editor Commandへ収束
- D3D12 device、Render Graph、D3D12MA、Debug Layer／DRED／PIX marker
- Asset staging、content-addressed cache、sandboxed importer、cook
- GameplayDefinition schema／Validator／Cooker／C++ evaluatorとNativeGameModule boundary
- Material IR、Engine-owned Target Binding Layout、Material／Style Validator、代表Material preview

本Phaseのconcrete Adapterとpackageは`windows_desktop_v1`を対象とする。完了条件は、AIを使わずEditor操作がChangeSetを経由し、空SceneをWindowsでplay、save、packageできることである。

### Phase 3: 2D Manual First Playable

- CanvasRenderer、sprite、tilemap、2D light、camera
- Box2D Adapter、2D grid navigation
- UI、Audio、2D animation、CPU particle
- GameplayDefinitionでgame ruleを実装し、C++ evaluatorで実行
- `pixel_2d`と`illustrated_2d` Profile
- `pixel_2d` 2D top-down action縦切り

完了条件は、AIなしでTitleからResultまでplayでき、640×360 logical pixelを1920×1080へ3倍integer scaleし、Reference stress sceneが1080p60とmemory budgetを満たすことである。

### Phase 4: AI Authoring MVP-A

- Node.js／TypeScript AI Orchestratorとnamed-pipe IPC
- OpenAI Responses API、Structured Outputs、strict function calling
- Requirement Resolver、Game Brief、GameSpec生成
- VisualStyleResolver、2D Style候補Preview、Decision Ledger
- GameplayDefinition／C++ strategy planner
- GameplayDefinitionChangeSet、NativeCodeChangeSet、isolated validation／cook／build／test
- Engine-generated Diff、Approval、手動変更との競合処理
- Playtest feedbackと自動修復
- NativeGameModule Capabilityを公開する前にA1を完成し、Source Worker、Promotion、Review ReceiptをTool catalogへ解放

完了条件は、大まかなprompt→必要質問→人間選択または一件限定の`おまかせ`委任による2D Visual Style確定→First Playable生成→AI修正→手動修正→AI再編集を一つのProject revision historyで完走し、Style Resolver EvalとStyle Validator gateを満たすことである。MVP能力に含むNativeGameModule生成はA1 Gate合格後だけ成功とし、A1未完ならMVP-A完了を宣言しない。

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
- Third-person compact action arena

完了条件は、`realistic_basic` 3D縦切りを自然言語と手動編集の両方で作成でき、Khronos core／Unlit／Emissive Strength／Texture Transform material fixture、Reference stress scene、1080p60、memory budgetを満たすことである。

### Phase 7: Mobile Platform

- `android_mobile_v1`: GameActivity、Vulkan 1.1／AVP 2022、VMA、GameActivity input／text、Oboe、Swappy、AAB／PAD
- `apple_mobile_v1`: UIScene／MTKView、Metal／MTLHeap、UIKit touch／text、GameController、AVAudioSession／AudioUnit、archive／TestFlight
- Portable HLSL→SPIR-V→MSLのoffline shader pipeline、ASTC／ETC2／BCnのTarget別cook
- Device Manager、Apple Unsigned Build Worker／Signing Service／Upload Service、safe-area／cutout／orientation preview、touch simulation
- Mobile memory class、dynamic resolution、thermal governor、physical device matrix、16 KiB／privacy／Store package gate
- Phase 3のWindows 2D C1を入力にAndroid 2D→Apple 2D、Phase 6のWindows 3D C1を入力にmobile 3D品質の順で成立させる

完了条件は、同一Project revisionの2D First PlayableがAndroid minimum／reference実機とA12 iPhone／iPadでplay、save、suspend復帰でき、Build／Signing／Uploadの秘密分離を証明した署名済みinternal track／TestFlight package、memory／frame／thermal／privacy gateを満たすことである。

署名済みinternal track／TestFlight packageの作成前にA3 Releaseを完成する。A2 Engine MaintenanceはGame制作MVP-A／MVP-Bの必須条件ではないが、Engine coreをAI保守対象として公開する前に別途完成し、未完成時はR4 Toolを公開しない。

### Phase 8: Production CapabilityとDomain Pack

- Hybrid deferred path
- `realistic_advanced` Material、Skin／Hair／Eye／Cloth template
- `toon_basic`、`toon_character`、Art Asset／Animation presentation、inverted-hull／screen-space outline
- `pixel_diorama`のhigh-resolution 3D＋crisp sprite modeとunified low-resolution mode
- Multiple light、physically based atmosphere、volumetric fog／cloud
- Baked lightmap、irradiance／reflection probe、C3 dynamic GI研究
- GPU VFX、C2 Water Body／Query／Underwater、dynamic snow field、terrain、foliage、streaming
- 2D Action、FPS／TPS、RPG／Action RPG、Quest、Simulation、Strategy Pack
- 画像／音声／3D生成Provider adapter
- 自動Playtestとperformance regression

各CapabilityはAuthoring schema、Validator、Editor、AI command、Runtime compiler、Diagnostics、Test、fallback、VisualStyleProfile integrationの完了定義を満たしてからProduction扱いにする。ToonとPixel Dioramaは同時実装せず、Realistic advanced→Toon→Pixel Dioramaの順に個別vertical prototypeとperformance gateを通す。

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

- MVP-A: Phase 4完了。2D top-down actionでAI Authoring全loopを証明する。
- MVP-B: Phase 6完了。3D compact action arenaを追加し、共通基盤が2D専用設計でないことを証明する。
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
- ChangeSetをUndoおよびReplayできる。
- 無効なAI出力がWorld ModelへCommitされない。

## 16. 確定事項と実装計画への引渡し

### 16.1 確定事項

| 項目 | 決定 |
|---|---|
| 最初の縦切り | 2D top-down action |
| 第二の縦切り | 3D single-player third-person compact action arena |
| Editor Host | Windows 11 25H2以降 x64（OS build 26200以上） |
| Game Target | `windows_desktop_v1`、`android_mobile_v1`、`apple_mobile_v1`。Mobile Editor、TV／Wear／XR、tvOS／visionOSは初期対象外 |
| Graphics | WindowsはD3D12、AndroidはVulkan 1.1＋AVP 2022、AppleはMetal。C1はForward+、desktop C2でHybridを選択可 |
| Game executable language | C++23。CX0は非Shipping Header bootstrap、最終方式はNamed Modules＋`import std`、C++26はreadiness CIだけ |
| Game implementation | C++実行コード＋GameplayDefinition。Authoring dataはoffline Cookし、C++ evaluatorが実行 |
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
| AI契約 | `/schemas/mira/`のMCDを正本とし、C++／TypeScript／Cooked binary descriptor／MCP／Provider Schemaを生成 |
| AI検証 | Contract、semantic、policy、build、test、budget、選択的TLA+、Eval、Review、ProvenanceをRisk別に適用 |
| IPC | ACL付きWindows named pipe、length-prefixed JSON-RPC 2.0 |
| Editor | Dock／resize／floating、保存可能な複数Workspace、AI Partnerをpin可能 |
| Compatibility | Pre-1.0 API／ABI互換なし。永続Projectだけoffline migratorで一方向移行 |
| Runtime連携 | Domain間の直接呼出し禁止。`RuntimeOrchestrator`が固定phaseでtyped command／event／snapshotを統合 |
| Runtime storage | 独自16 KiB archetype chunk＋SoA列。構造変更はtick boundaryだけ |
| Lifetime | generation handle、phase／epoch付きlease、Asset Registry単一所有、queue別`GpuSubmissionSerial` retire |
| Performance target | CPU／GPU P95 14.00 ms soft、16.67 ms hard。Subsystem別budgetと10分soak |
| Authoring source of truth | Authoring Document集合＋単調増加`ProjectRevision`。UI、AI会話、Runtime Worldは正本ではない |
| NativeGameModule | Windows Developmentは別ProcessのGameHostが単一DLLを起動時に一度だけload、Shipping／Android／Appleは静的link。ABI越しにSTL／exception／allocator／native objectを渡さない |
| Asset model | Source／Import／Derived／Packageの四層、隔離Importer、content-addressed Derived Data、Asset Catalog／VFS、`.mirapack` |
| Renderer model | Immutable `RenderSnapshot`をextractし、Engine-owned Render GraphをcompileしてD3D12／Vulkan／Metal Adapterへ投入 |
| Water／Snow model | Water Surface／CPU Query、降雪VFX、Snow Surface、Gameplay Surface Stateを分離し、GPU presentation結果をPhysics／Gameplayへ逆入力しない |
| Editor model | Production Editorと初心者用`AI Creator` Workspaceを同一Document／ChangeSet上で提供。Panelはdock／resize／floating／multi-monitor／保存可能 |
| Player I/O | InputはAction→`InputSnapshot`、UIはretained typed document、AudioはEngine-owned mixer。Platform native APIはAdapter内だけ |
| Game text | UTF-8、HarfBuzz 14.2.1、FreeType 2.14.1、ICU4C 78.3。Editor shellのDirectWrite／TSFとは責務を分離 |
| Domain Pack | CoreはGenre非依存。2D Action／TPS／RPG／Simulationを初期Packとし、multiplayer等は別GateまでCoreへ混入させない |

### 16.2 実装計画書で分解する事項

次は設計上の選択肢ではなく、29文書で確定した契約をfile、target、生成物、fixture、testへ割り当てる実装計画作成作業である。fieldとpolicyを実装担当者が再決定してはならない。

1. Phase 0のrepository bootstrap、Build Gateway、CMake target DAG、`BuildDriverProfileV1`、Windows Ninja Multi-Config／Android Gradle→Single-Config Ninja／Apple Ninja–Xcode Preset、CMake File API query、Build Receipt、C++23 Header bootstrap、`mira_add_cpp_component()`、Named Module／`import std` Probe、`mira.build.ninja_adoption.v1` Gate、Miraikanai Contract Definition配置をtaskへ分解する。
2. 固定Toolchain／Dependency artifactの取得、hash lock、SBOM、offline CI image、更新Gateをtaskへ分解する。
3. Contract compiler、C++／TypeScript／binary descriptor／MCP／Provider projection、round-trip／transition conformance testをtaskへ分解する。
4. Authoring Document Store、ProjectRevision、ChangeSet transaction、journal／snapshot／crash recovery、headless fixtureをtaskへ分解する。
5. RuntimeOrchestrator、fixed phase、typed command／event／snapshot、queue、handle／lease、memory telemetryをtaskへ分解する。
6. Render extraction、Render Graph compiler、D3D12 Adapter、Vulkan／Metal interface、Shader／Material pipeline、reference captureを段階taskへ分解する。
7. Asset Import Worker、Derived Data、Catalog／VFS、Cook／`.mirapack`、AI Asset staging／provenanceをtaskへ分解する。
8. Editor document shell、dock／workspace、Scene／Outliner／Inspector／Asset／AI Partner、UI Automation bridge、recovery testをtaskへ分解する。
9. NativeGameModule C ABI、Target別static／dynamic link、isolated Build、GameHost restart、Promotion Gateをtaskへ分解する。
10. Input、UI／Text／Localization／Accessibility、AudioのC0契約と2D First Playable用C1 vertical sliceをtaskへ分解する。
11. Collision、独自Physics PlatformのWorld／Dynamics／Joint／Character／Save／Replay、Box2D／Jolt Kernel Qualification、独自Navigation PlatformのGrid2D／Backend Port／Artifact／status／Recast・Detour Qualification、ozz、Engine-owned Animation Graphをconformanceとvertical sliceへ分解する。
12. Water Source／Compiler／Render／CPU Query／Volume／Underwaterと、Weather Snapshot／降雪VFX／static・dynamic Snow Surface／stampをC1とC2の独立task、Gameplay分離fixture、Target別Gateへ分解する。
13. Windows MSIX／folder package、Android AAB／PAD／16 KiB、Apple archive／signing／uploadをPlatform別taskと実機Gateへ分解する。
14. AI Orchestrator、Requirement／Visual Style Resolver、OpenAI Provider、local MCP、外部Client Security ProfileをRisk別taskへ分解する。
15. Source Worker、Path Broker、Promotion Service、Receipt署名、TLA+ model、AI Eval corpus／holdout、Provider migration harnessを検証taskへ分解する。
16. 2D／3D reference scene、Water／Snow fixture、frame／memory／thermal／soak fixture、performance baseline、Milestone判定をtaskへ分解する。

将来の多ジャンル対応を理由に最初の縦切りを過剰に汎用化しない。各taskは「AI編集と手動編集の安全な往復」「Engine側検証」「playable result」のいずれかへ直接寄与しなければならない。

## 17. リスクと対策

| リスク | 対策 |
|---|---|
| 一括生成が巨大化する | 薄い全体＋深い代表部分、段階的ChangeSet |
| 質問が多すぎる | 影響度分類、「おまかせ」、段階別質問 |
| AIが誤った実装方式を選ぶ | Policy、Project Profile、Benchmark、Decision Record |
| AIが画風をGenreだけで決める | VisualStyleResolverの優先順位、Engine hard gate、High Impact承認 |
| 画風変更でMaterialだけが変わり不整合になる | StyleChangeSetでLighting、Camera、Post、VFX、UI、Importを一括検証 |
| 既存作品の映像表現を模倣する | 一般属性へ分解し、固有名を正規Profile／Shader／Assetへ保存しない |
| 構造化Gameplay Logicが遅い | Cook／index／layoutを先に最適化し、同一Contractとfixtureで有意な場合だけHot pathをC++化 |
| AIが手動変更を消す | Base revision、File hash、三者比較、Lock |
| Schema-validだが意味的に不正 | C++ Semantic validator、Budget、Dry-run |
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
| 増分Buildが速くても依存漏れで古いartifactを使う | generated Input／Output／Depfileを完全宣言し、clean／incremental hash一致、mutation、interrupt recoveryを`mira.build.ninja_adoption.v1` Gateで検証する |
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

Unity、Unreal Engine、Godot、O3DEからは、Retained tree、独自C++ Framework、Engine UI共有、third-party toolkit方式の利点と制約を比較するための一般的教訓だけを得る。Widget API、宣言構文、Node model、画面配置、serialization、プロダクト固有のEditor UXは採用しない。MiraUIの具体的な独自性と一次資料は独自Editor UI Framework規約を正本とする。

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

設計文書Indexに列挙した29文書の内部整合レビューは完了した。次はユーザーが[統合計画サマリー](./README.md#0-統合計画サマリー)と本書16章の確定事項を入口にReviewし、修正点または承認を返す。承認後、実装task、依存関係、Test、Milestone、性能Gate、完了条件を含むPhase 0実装計画書を別文書として作成する。

実装計画はPhase 0 Foundationから開始し、Windows 2D First Playable、Windows 3D First Playable、Android／Appleの順序付きmobile vertical sliceへ分解する。承認前にEngine実装へ着手しない。
