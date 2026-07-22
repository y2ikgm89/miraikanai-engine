# Miraikanai Engine Editor／Workspace／UX規約

- 文書ID: mirakan.arch.editor-workspace-ux
- 状態: review
- 正本範囲: Editor process model、Shell配置、Panel／Workspace、制作journey、AI Partner UX、手動編集との往復、Error／Recovery UX、初心者／上級者projection、AccessibilityとEditor操作性能
- 非正本範囲: Widget／Layout実装、Project transaction、Asset lifecycle、Gameplay contract、AI authorization／Approval、外部Tool・SDK・Libraryの固定値。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core architecture](../02-foundation/core-architecture.md)、[Naming／Project layout](../02-foundation/naming-project-layout.md)、[Project state](project-state.md)、[Asset lifecycle](asset-lifecycle.md)、[Editor UI Framework](editor-ui-framework.md)、[Gameplay programming model](gameplay-programming-model.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論

Miraikanai EditorはAI chatを中央に置いた単機能appでも、有名Engineの画面配置を複製したEditorでもない。Scene、Hierarchy／Outliner、Inspector、Asset、Graph、Timeline、Profiler、Build、Source、AI Partnerを同じAuthoring Modelへ投影する、再構成可能なWindows desktop production editorである。

初心者、経験者、Engine開発者は別Editor／別Project formatを使わない。Workspaceと情報密度だけを切り替え、AI編集と手動編集は同じChangeSet、Validator、Diff、Undo、Receiptを通る。

AIは常時表示、tab、floating、非表示をWorkspaceごとに選択できる。既定のProduction WorkspaceではAI Partnerをpinするが、Scene／Hierarchy／Inspector／Assetを置き換えない。

## 2. 決定権と対象外

| 主題 | 正本 |
|---|---|
| Shell、panel、layout、workspace、操作、AI Partner、accessibility | 本書 |
| Project Document、ChangeSet、Commit、Undo、Recovery data | [Project state](project-state.md) |
| AI Task、質問、権限、Approval、Provider／MCP／CLI | [AI Security／Approval](../01-governance/ai-security-approval.md) |
| Debug Session／Event／Counter／Query、pause／step、Replay／Causality、AI診断意味 | Debugging規約 |
| Runtime／Play、Renderer、Asset、Input／UI／Audio等のDomain意味 | 各Subsystem規約 |
| Asset Browser、Import Inspector、Preview、Conversion Report、Reimport ConflictのDomain data | [Asset lifecycle](asset-lifecycle.md) |
| MirakanUi Core、MirakanEditor Shell、Widget、Rendering、Platform Adapter、禁止GUI dependency | [Editor UI Framework](editor-ui-framework.md) |

C1ではMobile上でEditorを動かさず、web editor、VR editor、共同リアルタイムmulti-user、Editor extension marketplace、任意binary pluginを実装しない。Editor shellはC++23の独自`MirakanUi Core`と`MirakanEditor Shell`で構築し、Dear ImGui、Qt、WinUI、WPF、Windows Forms、GTK、wxWidgets、Electron、CEFをGUI Frameworkとして使用しない。Win32、D3D12、DirectWrite、TSF、UI Automation、OLEはEditor UI Framework規約のPlatform Adapter境界でだけ利用する。

## 3. Process model

```text
EditorHost
  ├─ Authoring Service／Command Gateway
  ├─ Panel Projection／Workspace Service
  ├─ MirakanUi Core／MirakanEditor Shell
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
- `Playing`のRuntime tweakは一時Session stateで、明示`Apply Back`が新`ProjectChangeSetV1`としてCommitされるまでProject正本ではない。
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
| World | Scene／Canvas、Game、World Outline、Hierarchy／Outliner、Inspector、Topology Graph、Level Form、Streaming Inspector、Map Presentation Preview、Bundle Review |
| Content | Asset Browser、Import、Visual Style、Material |
| Logic | Gameplay Definition Graph／Table／Form、UI Designer、Source |
| Motion | Timeline、Animation Graph |
| Simulation | Physics／Collision、Navigation、Simulation Monitor |
| Production | Build／Package、Target／Device、Test／Playtest |
| Diagnostics | Session、Console、Problems、Profiler、Timeline、Causality、Breakpoint／Watch、Replay、Reproduction、Render Graph、History／Diff、External Tools |
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

### 6.3.1 Level Authoring View

Level WorkspaceはWorld／Level／Map規約の同じSourceを次のProjectionで編集する。

| View | 編集対象 | 主Operation／制約 |
|---|---|---|
| World Outline | World、Region、Level、Scene | StableId selection。Scene永続化ownerとLevel membershipを別columnで表示 |
| Topology Graph | Level、Portal、entry／exit Anchor | `CreatePortal`／`UpdatePortalContract`／`DeletePortal`。片側edgeだけを保存しない |
| Level Form | Source Scene集合、entry／exit、System、Objective、Profile、Budget | `SetLevelSourceScenes`、`SetLevelEntryExitContract`、`SetLevelGameplayComposition` |
| Spatial View | Entity、Anchor、bounds、Scene owner | Transform Operation、`MoveEntityToScene`。Level membershipとCellを暗黙変更しない |
| Streaming Inspector | Cell、residency、dependency、memory／IO | Target別Derived Planのread-only projection。Source編集欄を持たない |
| Navigation Overlay | walkability、cost、query、Source／Artifact差 | Source Intentだけをtyped Operationで変更し、Navmesh／tileへwriteしない |
| Map Presentation Preview | minimap、world map、marker、fog | Presentation Sourceだけを変更し、Quest／Objective／Navigation authorityへwriteしない |
| Bundle Review | Requirement、Source Diff、Topology、Target、Budget、Test、Risk | Staging Bundleのaccept／reject。Commit権限を持たない |

各ViewのContext barはProject revision、World／Scene／Level Stable ref、Target、lock、`Source | Staging | Derived read-only | Runtime`を常時表示し、Authoring規約の`AuthoringSelectionContextV1`とWorld規約の`WorldAuthoringContextV1`を同じContext hashで結ぶ。Scene／Outliner／Graph／Form／Inspector間のselection同期はStableIdを使い、screen coordinate、表示row、同名Object、Hierarchy pathを対象identityにしない。

共有Sceneを複数Levelが参照する場合、編集の影響を受ける全LevelとTargetを操作前に表示する。Scene間Entity移動では永続化owner変更、参照closure、lock、Recipe overrideをPreviewし、Level membership変更は別Operationとして明示する。Derived read-only対象へのdrag／property edit／pasteは拒否し、対応するSource Intent Viewへの遷移候補を示す。

### 6.4 Source

C1 Source WorkspaceはProject C++／HLSL／MCD JSONのtree、UTF-8 editor、syntax highlight、diagnostic、Diff、Build／Test、symbolへのgenerated API referenceを持つ。大規模refactor、debugger、completionが必要な場合は設定済み外部IDEへProject／file／lineを開ける。

外部IDE変更はAuthoring規約の三者比較と`NativeCodeChangeSet`／外部Document ChangeSetへ取り込む。Editor内text buffer保存だけでSource Promotionまたはbinary loadを行わない。

### 6.5 Asset Browser／Import Inspector

Asset BrowserとImport Inspectorは[Asset lifecycle](asset-lifecycle.md)が所有するStable ID selection、`AssetSourceAnalysisV1`、`AssetImportProfileV1`、`AssetConversionReportV1`、`AssetReimportConflictV1`を投影する。

- Asset Browserはtype、semantic role、tag、license、Production readiness、diagnostic、dependencyでfilterできる。
- thumbnail、waveform、font sample、3D turntableは選択補助であり、Operation targetにはStable IDを使う。
- Import Inspectorは`Source`、`Analysis`、`Profile`、`Preview`、`Conversion`、`Dependencies`、`Diagnostics`、`History`を持つ。
- Basic viewはProfile候補とHigh Impact質問、Advanced viewは同じDocumentの全型付きfieldとevidenceを表示する。別設定を持たない。
- 3D PreviewはSource／Engine軸、Root、Pivot、bounds、ground、Hierarchy、Skeleton、Animation rootを表示する。
- Texture Previewはsource／scene-linear／Target compressed、channel、alpha、normal、mip、Sprite pivot／PPUを表示する。
- Audio Previewはwaveform、loop、loudness、true peak、channel、codec A／B、stream costを表示する。
- Font Previewはrequired script、fallback、missing glyph、variation、Target raster差を表示する。
- Reimportはbefore／after、consumer closure、Importer／Profile version、lossをDiffし、破壊的Conflictを自動promotionしない。
- Import／Preview／Cook／bulk migrationはcancel可能なlong-running taskであり、partial outputを公開しない。

### 6.6 Debugging

Debug WorkspaceはDebugging Ownerのtyped Storeを投影し、Panelごとに独自log parser、別timestamp、別Object identityを持たない。選択したSession、Project revision、Build、Target、tick／frame、recorded／current stateを上部Context barで固定表示する。

- Sessionは接続、recording tier、retention、gap、redaction、remote trust、crash／hang状態を表示する。
- Console／ProblemsはDebugging Ownerが所有する`DebugEventEnvelopeV1`と`MirakanDiagnosticV1`をseverity／domain／phase／Stable IDでfilterし、元Event、Snapshot、source map、Replay pointへ移動できる。
- Profiler／Timeline／Causalityはcounter／span／event／causal edgeを同じtimepointへ整列し、presentation結果をauthoritative causeとして逆向きに結ばない。
- Breakpoint／Watchはtarget、condition、scope、hit count、suspend policyを型付きで表示する。Runtime pauseは要求時点で即時停止せずT110 safe pointで成立させ、tick step／render-frame step／GameplayDefinition node stepを区別する。
- Replayはrecord→scrub→inspect、first divergence、recorded／current revision差分、欠損rangeを表示する。gapまたはredactionを値なしの正常状態として扱わない。
- Reproductionは選択Evidenceからbounded BundleをPreviewし、含有／除外file、secret／PII scan、hash、retention、export先を承認前に示す。
- External ToolsはIDE、PIX、RenderDoc、Perfetto、Instruments等を`ExternalDebuggerLaunchDescriptorV1`から起動し、Session／Process／Build／capture IDを戻す。外部Toolの表示だけをEngineの正本にしない。
- AI Partnerへは画面pixelや無制限raw traceではなく`AiDebugContextV1`を渡す。AI回答はEvidence ID、仮説、反証、不足データ、confidence、次のbounded Queryを示し、修正提案は既存Diff／Approval／Authoring Gatewayへ送る。

## 7. Workspace

### 7.1 built-in Workspace

| Workspace | 主対象 | 初期構成 |
|---|---|---|
| `AI Creator` | 初心者、高水準指示 | Game Brief、Preview、AI Partner、Question、Diff／Validation |
| `Production` | 通常制作 | Scene、Outliner、Inspector、Asset、AI Partner |
| `Level` | Level designer | World Outline、Scene、Outliner、Topology Graph、Level Form、Inspector、Streaming Inspector、Navigation／Physics、Map Preview、Bundle Review、AI Partner |
| `Gameplay Logic` | Designer／Programmer | Definition、Source、API、Test、Console、AI Partner |
| `Rendering` | Technical artist | Scene、Material、Style、Light、Render Graph、GPU Profiler |
| `Animation` | Animator | Scene、Timeline、Animation Graph、Asset、Inspector |
| `UI` | UI designer | UI Designer、Hierarchy、Inspector、Localization、Preview |
| `Debug` | Programmer／QA | Game、Session、Problems、Console、Profiler、Timeline、Causality、Breakpoint／Watch、Replay、Reproduction、AI Partner |

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

Project内の保存先はGame Project配置・命名規約に従う`.mirakan/user/<user_id>/workspaces/`とし、代替はOS user configとする。Project共有は明示export／importだけにする。built-inはimmutable、User Workspaceは複数作成、複製、rename、delete、exportできる。Workspace切替は未Commit Project draftを破棄しない。

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

World／Scene／Level編集では、画面captureまたはPanel内部objectをAI Contextの正本にせず、`AuthoringSelectionContextV1`、`WorldAuthoringContextV1`、必要な`SceneSliceV1`をPreviewする。Userは送信前にWorld／Scene／Level Stable ref、Viewport bounds、Target、field mask、omitted rangeを確認できる。Context生成後にProject revisionまたはselectionが変わった場合、pending promptとProposalをstale表示し、自動で新しい対象へ付け替えない。

### 8.2 外部Engine用語の入力解決

AI PartnerはUnity、Unreal Engine、Godotなどの用語を入力として受け取れるが、外部製品のObject modelまたは名称をMiraikanaiの正規modelへ永続化しない。Requirement Resolverは要求文脈、Project Capability、Target Profile、canonical Owner registryから次のread-only／DisposableなEvidenceを返す。

```text
ExternalEngineConceptResolutionV1
  request_id
  source_engine_family
  source_term
  context_summary_hash
  candidate_canonical_concepts[]
  selected_concept?
  resolution_status = resolved | question_required | unsupported
  evidence_refs[]
```

`resolved`は候補が一つで、要求、Project Capability、Target、正規Ownerと矛盾せず、`evidence_refs`がcanonical concept IDと決定根拠を閉じる場合だけ許可する。`selected_concept`は表示名でなくMCDまたは正規仕様のstable concept IDである。

`Unity Scene`、`Unreal Level`、`Godot Scene`は文脈により`SceneDocument`、`LevelDefinition`、`WorldStreamingPlanV1`／Cell、`UiDocument`、Composition Recipeのいずれにもなり得る。候補選択がState owner、Save形式、Level遷移、Streaming、Target Capability、Project構造を変える場合は`question_required`とし、AIは仮定でChangeSetへ進めない。表示呼称だけの低影響差はcanonical termへ正規化できるが、未対応機能は`unsupported`として近似実装へ暗黙変換しない。

外部用語とcanonical conceptの永続的な1対1 alias、互換class、外部Scene path、Hierarchy indexをidentityとして作らない。このResolutionは入力理解のEvidenceであり、Project正本、Commit可能なOperation、Owner登録ではない。

### 8.3 Interaction mode

- `Ask`: 説明、質問、比較だけ。状態変更Proposalを作らない。
- `Suggest`: ChangeSet／Source changeを作り、Commitしない。
- `Execute Authorized`: Authorization Envelope内で検証／承認済み操作だけを進める。

Mode表示は常時visibleで、prompt本文によって自己昇格しない。AI outputをEngine validationと同じ色／iconにしない。

### 8.4 初心者workflow

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
- Scene／Level Viewのdirty表示はlocal draft、staged Proposal、Commit済みrevision、Derived再Cook待ちを別stateとし、一つの`*`だけで混同しない。
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
- Auto-save cadenceとfocus-loss triggerは[Project stateの永続化正本](project-state.md#6-source-layoutと永続化)をexactに消費し、Workspace UXで値を再定義しない。
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
| Debug Store／Index破損 | 完全chunkまで回復し、欠損rangeをgapとして表示。推測で補完しない |
| remote capture切断 | partial captureをread-onlyで確定し、未受信range、handshake、Targetを表示 |
| recording budget超過 | policyに従い低priorityからdropし、件数とrangeを必ずEvent化 |
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
- 既知のqueue overflow、stale handle、asset revision drift、simulation divergenceをSession／Timeline／Causality／Replayで識別し、gapを原因確定へ使わない
- pause要求がT110 safe pointでだけ成立し、tick／render-frame／GameplayDefinition node stepが混同されず、watch値にrevision／timepoint／validityが表示される
- AI診断のvalidated causeはすべてEvidence IDを持ち、recorded／current混同、presentation→authoritativeの逆因果、再現なし修正成功を0件にする
- 1920×1080でScene、Outliner、Inspector、Asset、AI Partnerが同時利用可能
- `world_authoring_cross_view_v1`の64 scenarioで、World Outline、Topology Graph、Level Form、Spatial View、AIが同じStableId／revision／Domain Operation／after state hashへ収束
- Scene永続化owner、Level membership、Cell assignmentの表示と変更経路を混同せず、共有Scene変更の影響LevelをCommit前に列挙
- sort／filter／rename／re-shard／DPI変更後も、mouse、keyboard、UI Automation、AIがscreen coordinateまたは表示rowで別Objectを変更しない
- Derived read-only／Runtime対象の編集を全Viewで拒否し、Source Intentへの安全な遷移候補を表示
- 10万Assetのfilter／selection、multi-select共通field edit、dependency／reverse dependency表示
- 3D／Texture／Audio／Font Previewが正式Import Planと同じ変換結果を表示
- Basic／Advanced／AI／headless CLIが同じPlan hash、Diagnostic、Reimport Conflictへ収束
- Importer version、Hierarchy、Skeleton、Material、Clip、channel、coverage変更をconsumer closure付きでblock
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
