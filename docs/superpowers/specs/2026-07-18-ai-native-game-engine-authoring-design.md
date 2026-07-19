# AIネイティブ独自ゲームエンジン 設計計画書

- 文書版: 0.4
- 作成日: 2026-07-18
- 最終更新日: 2026-07-19
- 対象: 独自C++ゲームエンジン、独自Editor、AI制作基盤
- 状態: 基本構想、基盤、Runtime連携、2D／3D機能範囲を統合した設計レビュー版

## 1. エグゼクティブサマリー

本プロジェクトは、既存ゲームエンジンへAI機能を追加するものではない。人間とAIが同じ安全な編集経路を利用し、C++エンジンがすべての変更を検証・試行・確定する、独自のAIネイティブゲームエンジンとEditorを開発する。

中心となる制作体験は次のとおりである。

1. ユーザーが大まかな自然言語プロンプトを入力する。
2. AIがゲーム制作に重要な不足要件だけを質問する。
3. AIが理解したゲーム概要と補完事項をGame Briefとして提示する。
4. AIがゲーム全体のGameSpecと実装計画を作成する。
5. AIが承認済みVisual StyleとEngine Capabilityに従い、構造化編集、Script、C++、またはその組み合わせをシステム単位で選択する。
6. C++エンジンが生成結果を検証し、短縮されたゲーム全体と代表的な完成部分を持つFirst Playableを生成する。
7. ユーザーはAIとの会話、Scene／Graph／Inspector、Script、C++のいずれからでも調整できる。
8. すべての変更はChangeSetとして記録され、Diff、Dry-run、Test、Undo、監査を経由する。

AIはC++エンジン内部を直接操作しない。AI、Editor GUI、外部ツールはすべて変更提案の作成者であり、最終的な状態変更権限はC++ Command Gatewayだけが持つ。

## 2. プロダクトビジョン

### 2.1 目標

ゲーム開発経験のないユーザーから上級者まで、同じプロジェクト形式と同じEditorを利用できる制作環境を実現する。

- 初心者は自然言語だけで制作を開始できる。
- AIは不足要件を適応的に質問し、設計と実装を補助する。
- AIはScene、UI、ゲームルール、Asset設定、Script、C++、Testを生成・修正できる。
- 経験者はビジュアルEditor、Script、C++を直接編集できる。
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

「独自」は、OS、Compiler、Direct3D 12、標準Library、検証済みのPhysics／Navigation／Script kernelまで再発明する意味ではない。製品の正規data model、公開Capability、編集protocol、validation、lifecycle、serialization、Editor UXを本プロジェクトが所有し、外部LibraryはEngine-owned Adapter内へ隔離する。採用条件と初期Dependencyは基盤アーキテクチャ規約で固定する。

## 4. 対象ユーザーとAuthoring Mode

### 4.1 制作レベル

同じプロジェクトへ段階的にアクセスできるProgressive Disclosureを採用する。

| レベル | 主な操作 | 対象 |
|---|---|---|
| Level 0 | 自然言語、参考資料、選択肢への回答 | 初心者を含む全ユーザー |
| Level 1 | Scene、Inspector、Graph、UI、Data、Timeline | 制作経験者、ノーコード利用者 |
| Level 2 | 安全なゲームScriptの生成・直接編集 | 中級者、Designer、Mod制作者 |
| Level 3 | Project C++、Engine Extensionの生成・直接編集 | 上級者、Engine開発者 |

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

回答操作は影響度で制限する。`Blocking` は、人間が回答するか、その判断だけを対象にした有効な `allow_ai_select` 委任によって解消するまで制作Commitへ進めない。`High Impact` の「おまかせ」は有効な委任とCapability／BudgetのHard gate通過を必須とし、条件を満たさなければ確認へ戻す。「後で決める」は `Medium Impact` 以下に限り、期限、暫定値、影響範囲をDecision Logへ記録する。「仮実装して試す」は、Schema、保存データ、Public API、Gameplay spaceを固定しない可逆な実験だけに許可する。

初心者へScript／C++、ECS、RPCなどの実装方式を質問しない。AIは「同時に何体表示したいか」「オンラインが必要か」など、ゲーム上の要件を確認し、実装方式を内部で選択する。

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
- Script Editorまたは外部IDE
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

Editorが保持する正規のゲーム状態。Scene TreeやInspector表示そのものを正規状態にしない。

同じWorld ModelをScene、Graph、表、UI、Timeline、Simulation MonitorなどへProjectionする。

### 6.3 ChangeSet

現在状態に対する不変な変更提案。

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
| GameChangeSet | World、Scene、UI、Rule、Asset設定 |
| StyleChangeSet | VisualStyleProfile、Material Template、Art Asset、Animation presentation、Lighting、Camera、Post、VFX、UI Style、関連Import設定 |
| ScriptChangeSet | Script、公開Parameter、Binding、Test |
| NativeCodeChangeSet | C++ source、header、Build定義、Test |
| AssetChangeSet | 画像、音声、3D、Animation、Import設定 |

各種別は共通してRevision、Diff、承認、履歴を持つが、検証・適用方法は分離する。

### 6.5 Behavior Contract

ScriptとC++実装を交換可能にするため、GameSpecは具体的な言語実装ではなくBehavior ContractまたはCapabilityを参照する。

Script版がPerformance要件を満たさない場合、同じContractを実装するC++版へ置き換えられる。呼び出し側のGameSpecと他システムを全面変更しない。

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
- 最低スペック
- 目標FPS
- CPU／GPU／Memory budget
- Render Quality Tierと必須D3D12 feature
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

## 7. システムアーキテクチャ

本章はProduct全体の論理構成を定義する。C++ module依存、所有権の一般則、Build、directoryは[Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)、Runtime phase、Subsystem連携、borrow無効化、Asset version、memory／performance budget、障害復旧は[Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)、2D／3D Subsystemの機能範囲は[Miraikanai Engine 2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)を詳細基準とする。

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
       +--------+---------+
       |        |         |
 Structured   Script     C++
       |        |         |
       +--------+---------+
                |
                v
       Normalized ChangeSet
                |
                v
        C++ Command Gateway
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

- World Model
- Stable ID
- Revision
- Command Gateway
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

システム単位で構造化編集、Script、C++、Hybridを選択する。LLMの推測だけで決めず、Engine Policyと実測を組み合わせる。

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

Host固有Pluginは薄い配布層とする。

- Engine固有skill
- GameSpec／ChangeSet説明
- Coding／Asset規約
- Build／Test手順
- MCP起動設定
- 検証・修復workflow

権威ロジックやEngine状態は保持しない。

### 7.9 CLI／Desktop

Codex／Claude CLI・DesktopはEngine開発、試作、CI、Code生成、Build、Testに利用する。出荷ゲームのRuntime依存にはしない。

## 8. Script＋C++実装戦略

### 8.1 基本方針

ユーザーはScriptかC++かを毎回選ばない。AIがシステムごとに最適な方式を選び、理由と測定結果を記録する。

```text
構造化編集で表現可能
  → 構造化編集

ゲーム固有ロジックで頻繁に変更
  → Script

高頻度、大量Entity、低遅延、Engine Capability
  → C++

高性能基盤と調整可能なルールが両方必要
  → C++＋Script＋構造化データ
```

### 8.2 判断材料

- ジャンル
- 2D／3D
- Real-time／Turn-based
- Single／Multiplayer
- Open World
- 同時Entity数
- Mapサイズ
- 目標FPS
- Platform
- 更新頻度
- Hot Reload要件
- 決定論
- Network Authority
- Mod公開範囲
- Security境界
- 既存Capability
- 過去のBenchmark

### 8.3 Scriptの責務

- ゲーム固有Behavior
- Ability
- Enemy AI
- Event
- Quest condition
- UI behavior
- Stage gimmick
- 頻繁に調整するRule

Scriptは許可されたCapability APIだけを利用する。任意メモリ、任意File、任意Network、OS commandを許可しない。

### 8.4 C++の責務

- 大量Entity処理
- 毎frameのHot path
- Physics／Navigation／Rendering
- Network Authority
- Deterministic simulation
- Asset importer
- 新しいCapability
- Scriptへ公開する安全な基盤

### 8.5 C++変更範囲

| 範囲 | AIの標準権限 | 必須検証 |
|---|---|---|
| Game Module | 生成・修正可能 | Compile、Unit、Integration |
| Project Native Module | 生成・修正可能 | Dependency、Capability、Sandbox |
| Engine Extension | 明示的な高リスク変更 | 全体Test、current API conformance、承認 |
| Engine Core | 通常は提案のみ | 特権承認、完全Review、回帰Test |

初心者の自然言語指示からC++が必要になった場合も、まず隔離WorkspaceへNativeCodeChangeSetを生成し、CompileとTestを通す。稼働中Engineへ未検証コードを直接注入しない。

### 8.6 実測による再評価

AIの予測だけでPerformanceを確定しない。

```text
候補実装
  → Benchmark
  → Budgetとの比較
  → Hot path特定
  → Script／C++境界の変更
  → 再測定
  → 採用
```

Script版をC++へ昇格する場合も、Behavior Contractを維持する。

## 9. AI編集と手動編集の統合

### 9.1 同じ正規状態を利用する

AI、Scene、Graph、Inspectorからの状態変更はGameChangeSetへ統合する。ScriptとC++の正規情報はSource fileとし、File hash、Base revision、Build結果を管理する。

任意のC++をGameSpecへ逆変換しようとしない。C++ Moduleは提供するCapabilityとContractをEngineへ登録する。

### 9.2 外部IDE

ScriptやC++が外部IDEで変更された場合、File watcherが変更を検出する。

1. File hash更新
2. Parse／Index
3. Compile
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
- 「ScriptになっているHot pathをC++化して」

AIは指示の対象、範囲、影響度を解決し、ChangeSetとTestを作成する。

## 10. 検証・安全・Transaction

### 10.1 唯一のCommit権限

API、Agent SDK、MCP、Plugin、CLI、Editor GUIのいずれも安全境界とはしない。C++ Command Gatewayだけが状態を確定できる。

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

### 10.6 Code Sandbox

Script／C++生成は隔離されたBuild環境を利用する。

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
- Script／Native Capability
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

AIはインストール済みCapabilityだけを利用できる。不足する場合は、構造化composition、Script、Project Native Module、Engine Extensionの順に選択し、Engine Extensionから人間の重要操作承認を必須とする。

## 12. Editor制作型とRuntime生成型

### 12.1 Editor制作型

最初の主対象とする。

- GameSpec生成
- Scene／Rule／UI／Asset設定
- Script／C++生成
- Diff
- Dry-run
- Playtest
- Build
- Export

### 12.2 Runtime生成型

Editorで検証基盤が成熟した後、同じIRとValidatorを異なるPolicy Profileで再利用する。

Runtimeでは次を原則禁止する。

- 任意C++生成・実行
- 任意Script実行
- 任意Asset download
- Arbitrary World mutation
- Physics／Network stateの直接確定
- Client内へのProvider API key埋め込み

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
- Agent loop: MVPでは独自Orchestrator。Agent SDKはPhase 7以降にEvalとADRを通した場合だけ採用
- 外部AI接続: MCP Adapter
- Host固有配布: 薄いCodex／Claude Plugin
- 開発・CI: CLI／Desktop
- 権威境界: C++ Command Gateway

### 13.2 Provider依存の隔離

GameSpec、ChangeSet、Behavior ContractをProvider固有形式へ依存させない。内部Provider adapterを設け、一社から開始して同一Evalで追加Providerを比較する。

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

- 本四文書の承認
- Windows 11 25H2以降 x64／Direct3D 12／C++20 Toolchain
- VS Build Tools 18.8.0／MSVC 14.51／Windows SDK 10.0.26100.8249／CMake 4.4.0／Ninja 1.13.2の`toolchain.lock.json`
- vcpkg manifest／固定Dependency
- Module dependency DAGとComposition Root
- `mira_runtime_contracts`、`mira_runtime_package`、`RuntimeOrchestrator`、Domain Port／Runtime／Adapter境界、`ComponentAccessManifest`
- RAII、所有権、generation handle
- 12段階fixed tick、8段階render frame、typed command／event、bounded queue
- CPU memory domain、GPU allocator、residency、fence-deferred release
- Result／Error、thread affinity、borrow epoch、Asset version／atomic promotion
- Naming、format、static analysis、sanitizer
- 2D／3Dの座標、単位、色、時間規約
- Scene dimension、Art Direction、Composition、Shading Modelの正規四軸

完了条件は、空のEditorHost／GameHost／WorkerHostが固定toolchainでBuildでき、Foundation contract、phase順序、handle／borrow、bounded queue、memory failure、Asset atomic promotionのtest、ASan、format、static analysisがCIで成功することである。

### Phase 1: Headless Authoring Core

- GameSpec、World Model、ChangeSet
- Stable ID／Project Revision
- Capability、Command Gateway
- Schema／Semantic／Budget validation
- Dry-run、Diff、Atomic Commit
- Transaction、Undo／Redo、Journal
- Save／Load、Replay
- Versioned schemaとoffline Project Migrator
- VisualStyleProfile、StyleChangeSet、StyleCapabilityManifestのschema

完了条件は、手書きChangeSetをheadlessでvalidate→stage→commit→save→load→replayし、同じstate hashを得られることである。

### Phase 2: Editor Shellと共通Runtime

- Dock／resize／floating／multi-workspace Editor shell
- Scene／Canvas、Outliner、Inspector、Asset、Diff／History、AI Partner panel
- Windows window／GameInput／XAudio2／DirectWrite
- D3D12 device、Render Graph、D3D12MA、Debug Layer／DRED／PIX marker
- Asset staging、content-addressed cache、sandboxed importer、cook
- Luau hostとCapability boundary
- Material IR、Domain別Root Signature、Material／Style Validator、代表Material preview

完了条件は、AIを使わずEditor操作がChangeSetを経由し、空Sceneをplay、save、packageできることである。

### Phase 3: 2D Manual First Playable

- CanvasRenderer、sprite、tilemap、2D light、camera
- Box2D Adapter、2D grid navigation
- UI、Audio、2D animation、CPU particle
- Luauでgame ruleを実装
- `pixel_2d`と`illustrated_2d` Profile
- `pixel_2d` 2D top-down action縦切り

完了条件は、AIなしでTitleからResultまでplayでき、640×360 logical pixelを1920×1080へ3倍integer scaleし、Reference stress sceneが1080p60とmemory budgetを満たすことである。

### Phase 4: AI Authoring MVP-A

- Node.js／TypeScript AI Orchestratorとnamed-pipe IPC
- OpenAI Responses API、Structured Outputs、strict function calling
- Requirement Resolver、Game Brief、GameSpec生成
- VisualStyleResolver、2D Style候補Preview、Decision Ledger
- Structured／Script／C++ strategy planner
- ScriptChangeSet、NativeCodeChangeSet、isolated build／test
- Engine-generated Diff、Approval、手動変更との競合処理
- Playtest feedbackと自動修復

完了条件は、大まかなprompt→必要質問→人間選択または一件限定の`おまかせ`委任による2D Visual Style確定→First Playable生成→AI修正→手動修正→AI再編集を一つのProject revision historyで完走し、Style Resolver EvalとStyle Validator gateを満たすことである。

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
- Third-person compact action arena

完了条件は、`realistic_basic` 3D縦切りを自然言語と手動編集の両方で作成でき、Khronos core／Unlit／Emissive Strength／Texture Transform material fixture、Reference stress scene、1080p60、memory budgetを満たすことである。

### Phase 7: Production CapabilityとDomain Pack

- Hybrid deferred path
- `realistic_advanced` Material、Skin／Hair／Eye／Cloth template
- `toon_basic`、`toon_character`、Art Asset／Animation presentation、inverted-hull／screen-space outline
- `pixel_diorama`のhigh-resolution 3D＋crisp sprite modeとunified low-resolution mode
- Multiple light、physically based atmosphere、volumetric fog／cloud
- Baked lightmap、irradiance／reflection probe、C3 dynamic GI研究
- GPU VFX、terrain、foliage、water、streaming
- 2D Action、FPS／TPS、RPG／Action RPG、Quest、Simulation、Strategy Pack
- 画像／音声／3D生成Provider adapter
- 自動Playtestとperformance regression

各CapabilityはAuthoring schema、Validator、Editor、AI command、Runtime compiler、Diagnostics、Test、fallback、VisualStyleProfile integrationの完了定義を満たしてからProduction扱いにする。ToonとPixel Dioramaは同時実装せず、Realistic advanced→Toon→Pixel Dioramaの順に個別vertical prototypeとperformance gateを通す。

### Phase 8: 制限付きRuntime生成

- Runtime専用SchemaとThreat Model
- Server-authoritative Gateway
- Allowlist Command
- Quota／Timeout／Fallback
- Dialogue、Quest outline、Encounter、NPC intent、bounded World構成
- Multiplayer revision／tick検証はNetwork設計承認後

任意C++、任意Script、raw Physics／Rendering state、client-side Commitは許可しない。

## 15. MVPベースライン

MVPはAI Authoringの安全な往復を証明する製品縦切りであり、Engine機能を網羅する版ではない。開発上の最小成立点を**MVP-A**、2D／3D両対応の基盤成立点を**MVP-B**として混同しない。

- MVP-A: Phase 4完了。2D top-down actionでAI Authoring全loopを証明する。
- MVP-B: Phase 6完了。3D compact action arenaを追加し、共通基盤が2D専用設計でないことを証明する。
- 最初のTechnology Preview配布条件はMVP-B完了とする。

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
7. 一種類の安全なScript生成
8. 一種類のProject C++ Capability生成
9. AIによるStructured／Script／C++選択
10. 隔離Compile／Test
11. Engine-generated Diff
12. Approval／Commit
13. Undo／Replay
14. AI会話による再編集
15. 手動Scene／Script／C++編集の検出
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
- AI生成Scriptがゲーム挙動へ使われる
- AI生成C++ Capabilityが一つ使われる
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
- Script生成とC++ Capability生成が隔離環境で検証される。
- AI変更をGame／Script／C++ごとにDiff表示できる。
- 手動変更後のAI再編集で、人間の変更が無条件に消えない。
- ChangeSetをUndoおよびReplayできる。
- 無効なAI出力がWorld ModelへCommitされない。

## 16. 確定事項と実装計画前の残作業

### 16.1 確定事項

| 項目 | 決定 |
|---|---|
| 最初の縦切り | 2D top-down action |
| 第二の縦切り | 3D single-player third-person compact action arena |
| Platform | Windows 11 25H2以降 x64（OS build 26200以上） |
| Graphics | Direct3D 12、C1はForward+、C2でHybrid renderer |
| Language | C++20 |
| Script | Luau 0.730 strict mode＋Engine Capability API |
| 2D Physics | Box2D 3.1.1 Adapter |
| 3D Physics | Jolt Physics 5.6.0 Adapter |
| 3D Navigation | Recast／Detour 1.6.0 Adapter |
| GPU memory | D3D12MA 3.2.0をEngine-owned wrapper内で利用 |
| Build | VS Build Tools 18.8.0／MSVC 14.51、CMake 4.4.0 Presets＋Ninja Multi-Config 1.13.2＋vcpkg manifest、hashとversion固定 |
| Performance | 1080p60、Ryzen 5 5600、16 GiB、RTX 3060 12 GB／RX 6600 8 GB、runtime CPU 2 GiB soft budget |
| Visual Style model | Scene dimension、Art Direction、Composition、Shading Modelを分離 |
| Material | 型付きMaterial IR、複数Shading Model、Engine-owned Root Signature |
| 最初の2D Style | `pixel_2d`、640×360、32 PPU、integer scale |
| 最初の3D Style | `realistic_basic` |
| Production Style順 | Realistic advanced→Toon→独自`pixel_diorama` |
| Style決定 | 明示要件優先、Genre単独決定禁止、High Impactは人間選択または一件限定`allow_ai_select`委任 |
| AI経路 | 内蔵はModel API、外部HostはMCP、配布単位はPlugin |
| AI Orchestrator | Node.js 24.18.0 LTS／TypeScript 7.0.2 strict、別Process |
| 初期Provider | OpenAI Responses API、`gpt-5.6-sol`、reasoning `medium` |
| IPC | ACL付きWindows named pipe、length-prefixed JSON-RPC 2.0 |
| Editor | Dock／resize／floating、保存可能な複数Workspace、AI Partnerをpin可能 |
| Compatibility | Pre-1.0 API／ABI互換なし。永続Projectだけoffline migratorで一方向移行 |
| Runtime連携 | Domain間の直接呼出し禁止。`RuntimeOrchestrator`が固定phaseでtyped command／event／snapshotを統合 |
| Runtime storage | 独自16 KiB archetype chunk＋SoA列。構造変更はtick boundaryだけ |
| Lifetime | generation handle、phase／epoch付きlease、Asset Registry単一所有、queue別GPU fence retire |
| Performance target | CPU／GPU P95 14.00 ms soft、16.67 ms hard。Subsystem別budgetと10分soak |

### 16.2 実装計画書で分解する事項

次は設計上の選択肢ではなく、承認済み設計を実装taskへ分解する作業である。

1. World Model、ChangeSet、Capability、VisualStyleProfile schemaのfield-level定義
2. 承認済みtarget DAGをfile／public header／CMake targetへ割り当てる実装単位
3. Authoring ServiceとAI Orchestrator間のJSON-RPC method／message schema
4. Luau Capability contractのfield-level定義
5. D3D12 feature check、descriptor field layout、Render Graph resource recordのfield-level schema
6. Reference sceneのfixtureとBenchmark測定手順
7. Editor shellのaccessibility bridge検証
8. Approval Policyのoperation別初期値
9. Asset placeholderのlicense、生成、差替えworkflow
10. Material IR node、Domain output、Shading Model interface、Root Signature layout
11. VisualStyleResolverのrule engine、prompt fixture、60件×3回Eval harness
12. typed command／event、snapshot、queue payloadのfield-level schemaとcode generation
13. Reference profileのmemory／frame budgetを計測するtelemetry fixture

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
| Scriptが遅い | Behavior Contractを維持してHot pathをC++化 |
| AIが手動変更を消す | Base revision、File hash、三者比較、Lock |
| Schema-validだが意味的に不正 | C++ Semantic validator、Budget、Dry-run |
| 生成C++が危険 | 隔離Build、Network制限、固定Toolchain、Test |
| Provider依存 | Canonical IR、Provider adapter、Eval |
| チャットが長くなり決定を失う | Decision Ledger、GameSpec、Project memory |
| 初心者に技術質問をする | ゲーム上の要件へ翻訳、内部方式はAIが判断 |
| 自由度を上げると安全性が下がる | Structured／Script／C++で権限と検証を分離 |
| 既存Engineの模倣になる | 独自原則を先に固定し、比較調査は検証だけに使う |
| 所有権が曖昧でleak／use-after-freeが起きる | RAII、unique ownership、borrow規則、generation handle、ASan |
| D3D12 resourceをGPU使用中に解放する | queue別fenceを持つdeferred release |
| Allocatorを自作して逆に遅くなる | `pmr`境界、domain telemetry、実測したhot pathだけ専用化 |
| GPU memoryのthrashing | OS budget監視、residency priority、streaming、明示的な失敗 |
| Clean実装の名目でProject dataを失う | Runtime互換分岐は持たず、backup付きoffline migratorだけを提供 |
| Module境界が崩れVendorへ固定される | CMake dependency DAG、Ports／Adapters、public header検査 |
| Subsystemが相互に直接変更して順序依存になる | RuntimeOrchestrator、固定phase、typed command／event、非再入配送 |
| Hot reloadで新旧Assetが混在する | dependency closure全体をstagingし、boundaryでatomic promotion |
| 非同期jobが破棄済みobjectへ書く | handle＋versionを開始時と統合時に再検査し、stale resultを破棄 |

## 18. 調査から得た位置付け

調査上、API、MCP、Plugin、CLIは代替関係ではなく異なる層である。

- Model APIは推論を提供する。
- Agent SDKは高度なAgent loopを管理する。
- MCPはAI HostとEngine Toolを接続する。
- Codex／Claude PluginはworkflowとMCP設定の配布単位である。
- CLI／DesktopはEngine開発と試作に適する。
- どの層もEngineのSemantic validationとTransactionを代替しない。

Unity、Unreal Engine、Godotからは、Editor拡張、Undo、Tool registry、Main thread反映などの一般的教訓だけを得る。プロダクト固有の内部モデルやEditor UXは採用しない。

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
- Node.js 24.18.0 LTS: https://nodejs.org/en/blog/release/v24.18.0
- TypeScript 7.0: https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/
- Direct3D 12 Programming Guide: https://learn.microsoft.com/en-us/windows/win32/direct3d12/directx-12-programming-guide
- Direct3D 12 Memory Management Strategies: https://learn.microsoft.com/en-us/windows/win32/direct3d12/memory-management-strategies
- D3D12 Memory Allocator: https://github.com/GPUOpen-LibrariesAndSDKs/D3D12MemoryAllocator
- vcpkg Manifest Mode: https://learn.microsoft.com/en-us/vcpkg/concepts/manifest-mode
- Box2D: https://box2d.org/documentation/
- Jolt Physics: https://jrouwe.github.io/JoltPhysics/
- Recast Navigation: https://github.com/recastnavigation/recastnavigation
- Luau C API／Sandbox: https://luau.org/api/
- glTF 2.0 Specification: https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html

## 20. 次のアクション

本書、基盤アーキテクチャ規約、Runtime連携・寿命・性能規約、2D／3D機能計画の四文書を一つの設計としてReviewする。矛盾、未定義の責務、根拠のない技術選択を解消して承認した後、実装タスク、依存関係、Test、Milestone、性能Gate、完了条件を含む実装計画書を別文書として作成する。

実装計画はPhase 0 Foundationから開始し、2D First Playable、3D First Playableの順に分解する。承認前にEngine実装へ着手しない。
