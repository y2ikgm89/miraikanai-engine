# Miraikanai Engine Editor／Workspace／UX規約

- 文書版: 1.1
- 作成日: 2026-07-19
- 対象: Windows Editor shell、panel、docking、workspace、AI Partner、手動編集、accessibility、recovery
- 状態: プロジェクト公式の規範設計レビュー版
- Product設計: [AIネイティブ独自ゲームエンジン 設計計画書](./2026-07-18-ai-native-game-engine-authoring-design.md)
- Authoring規約: [Miraikanai Engine Authoring Model／Project State規約](./2026-07-19-authoring-model-project-state-design.md)
- AI規約: [Miraikanai Engine AI実装・保守ガバナンス規約](./2026-07-19-ai-engine-development-governance-design.md)
- 機能範囲: [Miraikanai Engine 2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)
- UI規約: [Miraikanai Engine UI／Text／Localization／Accessibility規約](./2026-07-19-ui-text-localization-accessibility-design.md)
- Editor UI Framework規約: [Miraikanai Engine 独自Editor UI Framework／Shellアーキテクチャ規約](./2026-07-20-editor-ui-framework-architecture-design.md)
- Windows規約: [Miraikanai Engine Windows Platform／Distribution規約](./2026-07-19-windows-platform-distribution-design.md)

## 1. 結論

Miraikanai EditorはAI chatを中央に置いた単機能appでも、有名Engineの画面配置を複製したEditorでもない。Scene、Hierarchy／Outliner、Inspector、Asset、Graph、Timeline、Profiler、Build、Source、AI Partnerを同じAuthoring Modelへ投影する、再構成可能なWindows desktop production editorである。

初心者、経験者、Engine開発者は別Editor／別Project formatを使わない。Workspaceと情報密度だけを切り替え、AI編集と手動編集は同じChangeSet、Validator、Diff、Undo、Receiptを通る。

AIは常時表示、tab、floating、非表示をWorkspaceごとに選択できる。既定のProduction WorkspaceではAI Partnerをpinするが、Scene／Hierarchy／Inspector／Assetを置き換えない。

## 2. 決定権と対象外

| 主題 | 正本 |
|---|---|
| Shell、panel、layout、workspace、操作、AI Partner、accessibility | 本書 |
| Project Document、ChangeSet、Commit、Undo、Recovery data | Authoring規約 |
| AI Task、質問、権限、Approval、Provider／MCP／CLI | AIガバナンス規約 |
| Runtime／Play、Renderer、Asset、Input／UI／Audio等のDomain意味 | 各Subsystem規約 |
| MiraUI Core、MiraEditor Shell、Widget、Rendering、Platform Adapter、禁止GUI dependency | Editor UI Framework規約 |

C1ではMobile上でEditorを動かさず、web editor、VR editor、共同リアルタイムmulti-user、Editor extension marketplace、任意binary pluginを実装しない。Editor shellはC++23の独自`MiraUI Core`と`MiraEditor Shell`で構築し、Dear ImGui、Qt、WinUI、WPF、Windows Forms、GTK、wxWidgets、Electron、CEFをGUI Frameworkとして使用しない。Win32、D3D12、DirectWrite、TSF、UI Automation、OLEはEditor UI Framework規約のPlatform Adapter境界でだけ利用する。

## 3. Process model

```text
EditorHost
  ├─ Authoring Service／Command Gateway
  ├─ Panel Projection／Workspace Service
  ├─ MiraUI Core／MiraEditor Shell
  ├─ Platform UI／Text／Accessibility Adapter
  └─ IPC clients
       ├─ GameHost
       ├─ AI Orchestrator
       ├─ Asset／Shader／Source Worker
       └─ Device／Package Service
```

- EditorHostは正規Authoring stateを所有する。
- GameHost crash、device loss、Project C++ faultでEditorHostを終了しない。
- AI Orchestratorは別Processで、Editor widget IDやnative handleを受け取らない。
- 長時間Build／Cook／GenerateをUI threadで実行しない。
- PanelはDomain Serviceのread projectionとtyped Operationだけを使い、private Runtime objectを保持しない。

### 3.1 Editor session lifecycle

Editor session stateを次のclosed enumへ固定する。

```text
Booting        -> NoProject | EditorClosing
NoProject      -> ProjectOpening | EditorClosing
ProjectOpening -> Authoring | NoProject | EditorClosing
Authoring      -> PlayPreparing | ProjectClosing | EditorClosing
PlayPreparing  -> Playing | Authoring | EditorClosing
Playing        -> PlayStopping | EditorClosing
PlayStopping   -> Authoring | EditorClosing
ProjectClosing -> NoProject | Authoring | EditorClosing
EditorClosing  -> terminal
```

- `Booting`はToolchain／Dependency lock、built-in Workspace、Accessibility Adapterを検証し、Project Documentを開かない。
- `ProjectOpening`はmanifest、schema、journal、snapshot、lockをAuthoring規約どおり回復し、成功した一つの`ProjectRevision`からPanelを投影する。途中状態を表示可能Projectにしない。
- `PlayPreparing`はCommit済みrevisionをcompile／cookし、専用GameHostを起動する。staged draftやAI proposalをRuntime packageへ混入させない。
- `Playing`のRuntime tweakは一時Session stateで、明示`Apply Back`が新`ProjectChangeSet`としてCommitされるまでProject正本ではない。
- `PlayStopping`はInput、Audio、GameHost、GPU／Asset leaseをRuntime規約の順序で停止し、GameHost終了確認後だけ`Authoring`へ戻る。
- `ProjectClosing`はCommit済みjournal／snapshotとEditorUserStateをflushし、Projectを参照するtaskへcancel／detach／blockの宣言済みpolicyを適用する。未Commit draftはRecovery Diffとして保存し、正規revisionへ自動Commitしない。
- `EditorClosing`は新taskとChangeSet受付を閉じ、Worker／AI／GameHostを停止し、timeout後はchild Processを終了してRecovery receiptを残す。

許可されない遷移、二重Play、closing中のCommit、旧Project generationのtask resultはtyped errorとして拒否する。GameHost／Worker crashはEditor sessionを`Faulted`へ遷移させず、Task failureとして隔離し、Authoringを継続する。

## 4. Shellと既定画面

### 4.1 常設領域

| 領域 | 内容 |
|---|---|
| Title／Project bar | Project、revision、Target、Play state、dirty draft、connection |
| Menu bar | File、Edit、Create、Scene、Game、Build、AI、Window、Help |
| Main toolbar | Save、Undo／Redo、Transform mode、Play／Pause／Stop、Target、Build |
| Workspace area | dock treeとfloating panel |
| Status bar | validation、background job、AI task、memory／frame、notification |
| Command palette | 全command、panel、Asset、Entity、Settingへの検索入口 |

Menu／toolbar actionも`EditorCommandId`へ登録し、mouse専用処理を持たない。危険ActionはRisk、対象、Diff、結果を明示し、toolbar一clickでR3以上を確定しない。

### 4.2 `Production`の1920×1080既定値

```text
+------------------------------------------------------------------+
| Menu 28 | Toolbar 40                                             |
+-----------+--------------------------------------+---------------+
| Outliner  | Scene / Game                         | Inspector     |
| 300       |                                      | 420           |
|           |                                      +---------------+
|           |                                      | AI Partner    |
+-----------+--------------------------------------+---------------+
|           | Asset / Console / Build / Timeline 260               |
+------------------------------------------------------------------+
| Status 24                                                        |
+------------------------------------------------------------------+
```

数値は100% DPIのlogical pixel初期値であり、DPIでscaleする。右DockはInspector 55%／AI Partner 45%の縦splitとする。Scene Viewは1920×1080で最低1200×728 logical pxを確保する。16:9 monitorを基準にするが、Editor windowを16:9へ固定しない。

Windowが狭くScene ViewのC1最小`640×360`を維持できない場合、低priority panelをtab化して中央を確保し、勝手にfloating／画面外へ移動しない。

## 5. Panel contract

### 5.1 Panel descriptor

```text
EditorPanelDescriptor
  panel_type_id
  instance_id
  title_key
  icon_id
  singleton_policy
  default_dock
  minimum_size
  preferred_size
  supported_contexts
  command_ids[]
  accessibility_role
```

Panel instance stateはProject gameplay dataと分離し、User Workspaceへ保存する。Panelは`OnProjectRevisionChanged`でread projectionを更新し、Commit前にlocal cacheを正規状態として扱わない。

### 5.2 Dock、resize、floating

- Panelの上下左右edgeをdragしてsize変更できる。
- resize hit targetは6 logical px、visual separatorは1～2 logical pxとする。
- dock previewはleft／right／top／bottom／center-tabの5 zoneを表示する。
- tabをdragしてdock入替え、split作成、tab group移動、floating化できる。
- floating panelはOS windowとして複数monitorへ移動できる。
- keyboardでもpanel移動、dock先選択、resize、tab切替へ到達できる。
- docking中はpreview outlineと最終sizeを表示し、drop前にlayoutをCommitしない。
- Panelごとに最小sizeを持ち、下回るdropを拒否して理由を表示する。
- AI Partnerを含む全主要Panelをpinできる。

Panelが消失した場合、`Window > Find Panel`、Command palette、`Reset Current Workspace`の三経路で回復できる。

### 5.3 Layout validity

Workspace load時にmonitor topology、work area、DPI、Panel type、minimum sizeを検証する。

- 存在しないmonitor上のfloating windowはprimary monitorの安全領域へ戻す。
- 20%以上が全monitor work area外なら回収する。
- 欠落Panel typeはplaceholder tabで理由を示し、layout全体を破棄しない。
- corrupt layoutはlast-valid layoutを使い、built-in `Production`へfallbackしたことを通知する。
- UserがSaveしていない一時layoutは次回起動へ自動上書きしない。

## 6. 公式Panel

| Group | C1 Panel |
|---|---|
| World | Scene／Canvas、Game、Hierarchy／Outliner、Inspector |
| Content | Asset Browser、Import、Visual Style、Material |
| Logic | Gameplay Definition Graph／Table／Form、UI Designer、Source |
| Motion | Timeline、Animation Graph |
| Simulation | Physics／Collision、Navigation、Simulation Monitor |
| Production | Build／Package、Target／Device、Test／Playtest |
| Diagnostics | Console、Problems、Profiler、Render Graph、History／Diff |
| AI | AI Partner、Question／Decision、Task／Receipt |

同じ機能を初心者用と上級者用に別実装しない。初心者Workspaceではadvanced Panelを初期非表示にし、AIが必要時に理由とともに開く提案をする。

### 6.1 Hierarchy／Outliner

- World／Scene／EntityをWorld Modelのparent関係で表示する。
- search、type／tag／validation filter、multi-select、rename、reparentを持つ。
- drag reparentはcycle、Scene boundary、persistent policyをdrop前にvalidateする。
- 表示row indexでなくStableIdをselectionへ使う。
- 100万Entity fixtureではvirtualized treeとincremental projectionを使う。

### 6.2 Inspector

- MCD field metadataからlabel、unit、range、enum、reference picker、help、live-edit policyを生成する。
- multi-selectは共通fieldとmixed valueを表示する。
- continuous dragはpreview valueをlocal draftへ書き、pointer release／Enterで一ChangeSetとしてCommitする。
- Escapeでdraftを破棄する。
- invalid fieldをclampして成功させず、範囲と修正候補を示す。
- AI source、human lock、default、override、runtime-onlyを視覚的に区別する。

### 6.3 Scene／Canvas

- Select、translate、rotate、scale、frame、measure、grid／snapを持つ。
- 2D／3D coordinate、meter／radian、pixel／PPU表示を明示する。
- Physics、Collider、Navigation、Light、Camera、VFX、Audio、UI safe areaをoverlayできる。
- Gizmo操作はtyped Transform Operationを生成し、native transform pointerへwriteしない。
- Play中の編集はfieldの`live_edit_policy`を表示し、restart要求を隠さない。

### 6.4 Source

C1 Source WorkspaceはProject C++／HLSL／MCD JSONのtree、UTF-8 editor、syntax highlight、diagnostic、Diff、Build／Test、symbolへのgenerated API referenceを持つ。大規模refactor、debugger、completionが必要な場合は設定済み外部IDEへProject／file／lineを開ける。

外部IDE変更はAuthoring規約の三者比較と`NativeCodeChangeSet`／外部Document ChangeSetへ取り込む。Editor内text buffer保存だけでSource Promotionまたはbinary loadを行わない。

## 7. Workspace

### 7.1 built-in Workspace

| Workspace | 主対象 | 初期構成 |
|---|---|---|
| `AI Creator` | 初心者、高水準指示 | Game Brief、Preview、AI Partner、Question、Diff／Validation |
| `Production` | 通常制作 | Scene、Outliner、Inspector、Asset、AI Partner |
| `Level` | Level designer | Scene、Outliner、Inspector、Nav／Physics、AI Partner |
| `Gameplay Logic` | Designer／Programmer | Definition、Source、API、Test、Console、AI Partner |
| `Rendering` | Technical artist | Scene、Material、Style、Light、Render Graph、GPU Profiler |
| `Animation` | Animator | Scene、Timeline、Animation Graph、Asset、Inspector |
| `UI` | UI designer | UI Designer、Hierarchy、Inspector、Localization、Preview |
| `Debug` | Programmer／QA | Game、Problems、Console、Profiler、History、AI Partner |

`AI Creator`は正式に採用する。AIを前面に出すが、Preview、質問、Diff、validation、戻し方を常に同時表示し、chatだけで状態を隠さない。いつでも`Production`へ切り替えられ、Project変換を行わない。

### 7.2 保存形式

`EditorWorkspaceV1`は次を持つ。

```text
workspace_id
display_name
base_workspace_id
layout_tree
floating_windows[]
panel_instances[]
visibility
toolbars
shortcut_profile_id
density_profile
last_focused_panel
monitor_signature
schema_version
```

保存先は`.mira/user/<user-id>/workspaces`またはOS user configであり、Project共有は明示export／importだけにする。built-inはimmutable、User Workspaceは複数作成、複製、rename、delete、exportできる。Workspace切替は未Commit Project draftを破棄しない。

## 8. AI Partner

### 8.1 常設state

AI Partnerは単なるconversation logでなく、次のstateを分けて表示する。

| State | 表示 |
|---|---|
| Intent | User要求、対象、完了条件 |
| Questions | High／Medium Impactの不足要件、回答、AI仮定 |
| Plan | System／Scene／Asset／C++の作業単位と依存 |
| Proposal | 未Commit ChangeSet、Native／Asset Source change |
| Validation | schema、semantic、budget、Build、Test、Preview |
| Approval | Risk、対象、権限、期限 |
| Result | Commit revision、Receipt、Playtest、rollback |

Panelは現在selection、open Document、Problems、Playtest結果をContext候補として表示し、送信前にUserが除外できる。Project全体を毎回Providerへ送らない。

### 8.2 Interaction mode

- `Ask`: 説明、質問、比較だけ。状態変更Proposalを作らない。
- `Suggest`: ChangeSet／Source changeを作り、Commitしない。
- `Execute Authorized`: Authorization Envelope内で検証／承認済み操作だけを進める。

Mode表示は常時visibleで、prompt本文によって自己昇格しない。AI outputをEngine validationと同じ色／iconにしない。

### 8.3 初心者workflow

1. 大まかなPromptからGame Briefを抽出する。
2. High Impact不足だけをGame用語で質問する。
3. Userが「おまかせ」を選んだ項目はAI仮定と理由をDecision Ledgerへ記録する。
4. 薄い全体と一つの深いplayable loopを提案する。
5. Diff、Risk、予測時間／Asset量、Target影響を見せる。
6. 検証後にCommitし、Playtest結果を自然言語と計測で返す。
7. 会話で修正し、手動編集があればbase revisionから再読込する。

初心者へC++／GameplayDefinition、ECS、Render Graph、ABIを選ばせない。

## 9. Manual editingとAIの往復

- GUI、Graph、Inspector、Source、AIの全変更にauthor、base revision、field sourceを記録する。
- 人間変更を既定でAI lockとせず、AIが変更する場合はDiffで明示する。
- 明示LockはAI、bulk tool、Recipe updateから保護する。
- AI proposal作成中に人間がCommitした場合、proposalをstaleとして自動Commitを禁止する。
- `Accept all`だけでなくOperation／Document／field単位のaccept／rejectを提供する。
- 一部accept後は新ChangeSetを再構築し、全Validatorを再実行する。
- Playtest中のruntime tweakをApply Backする場合もtyped Authoring Operationへ変換する。

## 10. Undo、History、Recovery

- Global Undo／RedoはAuthoring規約のinverse ChangeSetを使う。
- Panel内navigation、selection、camera、foldはProject Undoへ入れない。
- Continuous interactionは一つのUndo unitへcoalesceする。
- History Panelはintent、author、revision、affected objects、validation、Receipt、inverse availabilityを表示する。
- Auto-saveは20秒またはfocus lossでdraft Recoveryを更新する。
- Editor crash後は正規Projectを先に開き、Recovery DiffをUserへ提示する。自動Commitしない。
- GameHost／Worker crashはProblemsとTaskへ表示し、Editor layoutとProject draftを維持する。

## 11. Long-running task

Build、Cook、AI生成、Package、Device install、Testは`BackgroundTask`として次を持つ。

```text
task_id
task_kind
state
progress_kind
completed_units/total_units
current_stage
cancel_policy
log_stream_id
artifact_refs
result
```

Progress不明なのに擬似percentを表示せず、indeterminateとstageを示す。Cancelは`not_cancelable \| cooperative \| process_terminate`を明示する。Modal dialogでtask完了までUIをblockしない。

## 12. Accessibilityと人間工学

### 12.1 必須要件

- 全主要機能をkeyboardで操作でき、keyboard trapを作らない。
- focus order、focus indicator、accessible name／role／value／stateを持つ。
- colorだけでerror、approval、selection、AI／Engine stateを区別しない。
- text、icon、focus、graph edgeにHigh Contrast対応を持つ。
- shortcutを表示、検索、remap、resetできる。
- drag-only操作にはkeyboard／menuによる代替を持つ。
- animationを減らす設定、font／UI scale、comfortable densityを持つ。
- Problemsから対象Panel／Object／fieldへfocus移動できる。

`EditorViewDescriptor`とcommitted Layoutから`EditorSemanticSnapshotV1`を構築し、Windows UI Automation providerへprojectする。Draw primitive、pixel、hit-test結果からSemantic Treeを逆算しない。Custom controlは適用可能な標準UIA control patternを実装し、Narrator、NVDA、Accessibility Insightsで検証する。

### 12.2 SizeとDPI

- layout test: 1920×1080、2560×1440、100／125／150／200% DPI
- C1 minimum Editor window: 1280×720 logical px
- toolbar／primary action hit target: 最低32×32 logical px
- dense row: 24 logical px、comfortable mode: 32 logical px
- focus outline: 最低2 logical px
- Scene／Game view: 最低640×360

Textを画像へ焼き込まず、UI scale 200%でclip、重なり、off-screen controlを0件にする。

## 13. Performance

| Metric | C1 acceptance |
|---|---:|
| Idle UI frame P95 | 16.67 ms以下 |
| Input→visual response P95 | 50 ms以下 |
| Panel open／workspace switch P95 | 200 ms以下。Asset load除外 |
| Inspector continuous drag P95 | 16.67 ms以下 |
| 100万Entity filtered Outliner initial result | 500 ms以下 |
| 10万Asset search initial result | 200 ms以下 |
| UI thread blocking task | 50 ms超を0件 |
| EditorHost＋同時に一つのGameHost aggregate memory | 4 GiB hard cap内 |

検索、thumbnail、validation、projection rebuildはcancel可能なbackground jobを使い、結果統合時にProject revisionとPanel generationを再検査する。

## 14. Failure policy

| Failure | 結果 |
|---|---|
| corrupt Workspace | last-validまたはbuilt-inへfallback、Project不変 |
| missing monitor／DPI変化 | windowをwork areaへ回収 |
| Panel exception／invariant | Panel instanceを閉じProblemsへ記録、Editor stateを部分変更しない |
| stale projection | Operation reject、最新revisionから再投影 |
| AI disconnect | Manual Editor継続、pending proposalを保持 |
| GameHost crash | Editor継続、crash task／last log／再起動を提示 |
| Worker timeout | Task failure、cancel／retry、Project revision不変 |
| UI Automation provider failure | Release gate失敗。accessibilityを無効化してShippingしない |
| Recovery破損 | 隔離し正規Projectだけを開く |

## 15. TestとDefinition of Done

- Panel edge resize、5-zone dock、tab reorder、floating、multi-monitor、pin
- 6種類以上のUser Workspace保存／切替／export／import
- monitor unplug、DPI変更、corrupt layout、missing Panelからの回復
- mouse、keyboard-only、screen readerで2D Project作成／保存／Play
- 200% scale、High Contrast、reduced motion、shortcut remap
- AI CreatorからProductionへ切替えて同じObjectを手動修正し、AI再編集で保持
- stale proposal、部分accept、human lock、Undo／Redo、external IDE conflict
- GameHost／AI／Asset Worker crash中もProjectとlayoutを失わない
- 1920×1080でScene、Outliner、Inspector、Asset、AI Partnerが同時利用可能
- UIA treeのname／role／value／patternとvisual focusの自動test
- 100万Entity、10万Asset、長時間TaskのPerformance fixture

C1 Editorは、初心者がAI Creatorだけで2D First Playableを作れ、経験者がProduction／Gameplay Logicで同じProjectを手動調整でき、AI panelを常設しても従来の制作PanelとScene領域を損なわない時点で完了する。

## 16. 一次資料

- [Microsoft UI Automation Providers Overview](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-providersoverview)
- [Windows Accessibility Overview](https://learn.microsoft.com/en-us/windows/apps/design/accessibility/accessibility-overview)
- [Windows Keyboard Interactions](https://learn.microsoft.com/en-us/windows/apps/develop/input/keyboard-interactions)
- [High DPI Desktop Application Development](https://learn.microsoft.com/en-us/windows/win32/hidpi/high-dpi-desktop-application-development-on-windows)
- [Text Services Framework](https://learn.microsoft.com/en-us/windows/win32/api/_tsf/)
- [WCAG 2.2](https://www.w3.org/TR/WCAG22/)

WCAGはweb conformanceをEditorへそのまま宣言するためでなく、keyboard、focus、drag代替、target size等の人間工学上の最低原則として参照する。Windows desktopの実装合否はUI Automationと実assistive technologyで検証する。
