# Miraikanai Engine Editor／Workspace／UX規約

- 文書ID: mirakan.arch.editor-workspace-ux
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Editor process model、Project Browser／Launcher projection、Shell配置、Panel／Workspaceの共通契約、Editor表示locale／AI返答locale preference、制作journey、外部IDE往復、AI Partner UX、手動編集との往復、Error／Recovery UX、AccessibilityとEditor操作性能
- 非正本範囲: Engine release取得／install／update意味、Widget／Layout実装、Project transaction／VCS semantics、Project test semantics、Asset lifecycle、Gameplay contract、AI authorization／Approval、外部Tool・SDK・Libraryの固定値。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Editor UI Framework](editor-ui-framework.md)、[Project State](project-state.md)、[Game Production Loop](game-production-loop.md)、[Asset Lifecycle](asset-lifecycle.md)
- 関連文書: [Editor Panel／Reference Catalog](../appendices/editor-panel-reference-catalog.md)、[Product Plan](../00-product/product-plan.md)、[Product Lifecycle](../00-product/product-lifecycle.md)、[Product Privacy／Data Governance](../01-governance/product-privacy-data-governance.md)、[Product Security](../01-governance/product-security.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core architecture](../02-foundation/core-architecture.md)、[Naming／Project layout](../02-foundation/naming-project-layout.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[Project state](project-state.md)、[Developer Testing](developer-testing.md)、[Native Game Module](native-game-module.md)、[Asset lifecycle](asset-lifecycle.md)、[Editor UI Framework](editor-ui-framework.md)、[Gameplay programming model](gameplay-programming-model.md)、[World](../06-rendering/world.md)、[Scenario／Stage](../08-packs/scenario-stage.md)、[UI／Text／Localization／Accessibility](../07-platform/ui-text-localization-accessibility.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-26

## 1. 結論

Miraikanai EditorはAI chatを中央に置いた単機能appでも、有名Engineの画面配置を複製したEditorでもない。Scene、Hierarchy／Outliner、Inspector、Asset、Graph、Timeline、Profiler、Build、Source、AI Partnerを同じAuthoring Modelへ投影する、再構成可能なWindows desktop production editorである。

初心者、経験者、Engine開発者は別Editor／別Project formatを使わない。Workspaceと情報密度だけを切り替え、AI編集と手動編集は同じChangeSet、Validator、Diff、Undo、Receiptを通る。

AIは常時表示、tab、floating、非表示をWorkspaceごとに選択できる。既定のProduction WorkspaceではAI Partnerをpinするが、Scene／Hierarchy／Inspector／Assetを置き換えない。

Project create、Template／Sample、update／repair、Documentation、support、NOTICEは[Product Lifecycle](../00-product/product-lifecycle.md)が製品意味を所有する。Editorは同じtyped request／Operation／ReceiptのGUI projectionであり、CLI／headlessと異なるhidden default、silent repair、direct filesystem mutation、partial successを持たない。GUI固有のWizard、progress、confirmation、link presentationは許可するが、request hash、authorization、candidate hash、result、typed Diagnosticを変えない。

## 2. 決定権と対象外

| 主題 | 正本 |
|---|---|
| Shell、panel、layout、workspace、操作、AI Partner、accessibility | 本書 |
| Project Document、ChangeSet、Commit、Undo、Recovery data | [Project state](project-state.md) |
| AI Task、質問、権限、Approval、Provider／MCP／CLI | [AI Security／Approval](../01-governance/ai-security-approval.md) |
| Project bootstrap、Template／Sample／Documentation、surface parity、update／repair／support／NOTICE | [Product Lifecycle](../00-product/product-lifecycle.md) |
| Product data flow、consent、retention、export／delete | [Product Privacy／Data Governance](../01-governance/product-privacy-data-governance.md) |
| Product vulnerability case、security update／disclosure／incident | [Product Security](../01-governance/product-security.md) |
| Project test suite／case／runner／result | [Developer Testing](developer-testing.md) |
| Public C++ API Catalog、Native build／link／package | [Native Game Module](native-game-module.md) |
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
- PanelはDomain Serviceのread projectionとtyped Editor command／Project change primitiveだけを使い、private Runtime objectを保持しない。

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

### 3.2 Project Browser／Launcher

`NoProject`の既定surfaceは空SceneではなくProject Browserである。Project Browserはinstalled Engine release、Editor／SDK health、Template／Sample、recent／pinned Project、repository state、Target compatibility、update／repair／support statusをread-only projectionとして表示し、Product LifecycleとProject Stateのauthorityを置き換えない。

```text
ProjectBrowserEntryV1
  project_identity_ref: exact ProjectIdentityRefV1
  project_manifest_location_ref: exact LocalLocationRefV1
  bound_engine_release_ref: exact EngineReleaseBindingRefV1
  observed_repository_snapshot_ref:
    optional exact ProjectRepositorySnapshotRefV1
  last_opened_project_revision_ref:
    optional exact ProjectRevisionRefV1
  compatibility_assessment_ref:
    exact CompatibilityAssessmentRefV1
  health_state:
    ready | update_required | repair_required | conflicted
    | missing | access_denied | unsupported
  last_observed_at
  browser_entry_content_hash: SHA-256
```

Recent listはUser preferenceでありProject inventoryではない。Project pathを移動、renameまたは削除しても同pathの別Projectへidentityを付け替えない。missing、access denied、corrupt manifest、duplicate Project ID、repository conflict、required Engine release欠落を別stateとして表示し、一覧から自動削除、別releaseで自動open、silent migrationまたはempty Project再作成を行わない。

Project Browserから次のjourneyへ到達できる。

- installed exact Engine releaseでTemplateからProjectをatomic createする。
- existing Project manifestをscanし、変更せずcompatibility／repository／Target readinessをpreviewしてopenする。
- VCS providerが用意したlocal checkoutをscanする。remote clone、credential、branch checkoutはProject openと別Operationにする。
- Sampleをread-onlyで確認するか、new Project identityへ明示copyして開く。
- required Engine release／SDK／Template／Documentationを取得、repairまたは選択する。
- Projectを一覧から忘れる。Source、repository、cache、build outputのdeleteとは分離する。

Engine Launcher surfaceをEditor外processとして設ける場合も、同じ`EngineReleaseBindingV1`、Project Browser entry、install／update／repair Operationを使う。LauncherとEditorのrecent list、health、update stateを別authorityにせず、LauncherからEditorへ渡すのはexact Project identity、manifest location、Engine release refだけとする。arbitrary command line、credential、Project contentをhandoff payloadへ含めない。

Project create／open／clone／update／repair中はprogress、download source、artifact hash、disk impact、license、Privacy network flow、cancel／rollbackを表示する。cancelまたはfailure時はpartial openable Project、stale installed-release成功表示、temporary credentialを残さない。

## 4. Shellと既定画面

### 4.1 常設領域

| 領域 | 内容 |
|---|---|
| Title／Project bar | Project、revision、Target、Play state、dirty draft、connection |
| Menu bar | File、Edit、Create、Scene、Game、Build、AI、Window、Help |
| Main toolbar | Save、Undo／Redo、Transform mode、Play／Pause／Stop、Target、Build |
| Workspace area | dock treeとfloating panel |
| Status bar | validation、background job、AI task、memory／frame、`widget.feedback.notification@1`の一件だけの`status_bar_transient` feedback lane。warning／error／blocking／action付き通知はここへstackせず、owner Panel内の`owner_inline` surfaceへ置く |
| Command palette | 全command、panel、Asset、Entity、Settingへの検索入口 |

Menu／toolbar actionも`EditorCommandId`へ登録し、mouse専用処理を持たない。危険ActionはRisk、対象、Diff、結果を明示し、toolbar一clickでR3以上を確定しない。

### 4.2 `Production`の1920×1080最小既定値

```text
+------------------------------------------------------------------+
| Menu 28 | Toolbar 40                                             |
+-----------+--------------------------------------+---------------+
| Outliner  | Scene / Game                         | Inspector     |
| 300       |                                      | 420           |
|           |                                      +---------------+
|           |                                      | AI Partner    |
+-----------+--------------------------------------+---------------+
|           | Asset / Console / Build / Animation Timeline 260     |
+------------------------------------------------------------------+
| Status 24                                                        |
+------------------------------------------------------------------+
```

数値は100% DPIかつ`editor_ui_scale=1.00`の`ui_lu`初期値であり、最終physical pixelだけをDPIでscaleする。右DockはInspector 55%／AI Partner 45%の縦splitとする。Scene Viewは1920×1080で最低1200×728 `ui_lu`を確保する。16:9 monitorを基準にするが、Editor windowを16:9へ固定しない。

Windowが狭くScene ViewのC1最小`640×360`を維持できない場合、低priority panelをtab化して中央を確保し、勝手にfloating／画面外へ移動しない。

### 4.3 Reference 01: Standard Authoring Workspace

Reference 01は新しいWorkspaceやPanelを追加せず、`Production`の視覚・semantic基準を一枚へ集約する構成である。初期visual reviewは2560×1440 physical display topology、100% DPI、standard density、`editor_ui_scale=1.00`でE00のDarkとE13のpaired Lightの双方へ行い、Window client extentは§6.9.1のlock済みEnvironment Profileを正本とする。左DockはOutliner 320 lu、右Dockは440 lu、下Dockは280 luを初期値とし、中央Sceneは残余領域を使う。右DockはInspector 55%／AI Partner 45%、下DockはProblemsを既定tabとしてAsset Browser／Console／Build／Animation Timelineを同じtab groupから切り替える。1920×1080では§4.2へ縮退し、Scene最小値を侵害する前に下Dock、次にAI Partnerをtab化する。

このReferenceでは一つの選択済み`authoring_target`をOutliner、Scene、Inspectorで共有し、Problemsは同じTarget／fieldへのexact `diagnostic_target` relationと`対象を選択` Actionを示す。ProblemsのEvidence rowをObject selectionとしてselected表示しない。Outlinerは`selection.active` fill、Sceneは同じTargetの`selection.active` outline、Inspectorは同じcanonical field refを使う。Outlinerは[Editor UI Framework §15.6.2](../appendices/editor-ui-design-system-catalog.md#1562-widgetcollectiontree-row1)の`tree-row@1`を使い、expanded parent、`selected + focused + runtime + warning`、`locked`（owner／reason付き）、`proposal(ai) + approval.pending + stale`の四rowを固定表示する。selection、focus、validation rail、Runtime badge、AI proposal dash、stale clockを別layerにし、current hierarchyをproposal後の位置へ先行変更しない。

Inspectorは[Editor UI Framework §15.6.1](../appendices/editor-ui-design-system-catalog.md#1561-widgetpropertyproperty-row1)の`property-row@1`を使い、通常scalar、`runtime_read_only + warning`、`proposal(ai) + approval.pending + stale`の三rowを同時表示する。Runtime中はcontext barと対象へ`runtime`の`Runtime` badgeを出し、AI proposalは別fieldで`ai.proposal` dash、`AI提案`、before／after、accept／rejectを示す。Engine validation error／warningは`validation.error`／`validation.warning`のfield左rail＋icon＋message、stale proposalは`freshness.stale`のclock＋base/current revisionとrebase／discardだけを示す。これらを一つのbadge、色、tooltip、表示rowへ畳み込まない。

Reference 01のacceptanceは、同じStable ID／revisionからkeyboard、UIA、AI typed command、manual editが同じ対象を解決し、panel resize、dock preview、100%↔200% monitor move、Windowsの四つのbuilt-in contrast theme、reduced motionでもsemantic actionと状態の意味が変わらないことである。全interactive Controlとsemantic composite rootは[Editor UI Framework §15.6](../appendices/editor-ui-design-system-catalog.md#156-widget-pattern-contract)のexact `pattern_id`へ一件解決し、同じPattern instanceからvisual anatomy、Semantic Element、UIA pattern、Command Actionを生成する。Screenshotはこの構成の視覚差evidenceにだけ使い、state、authorization、revisionの正本にはしない。

## 5. Panel contract

### 5.1 Panel descriptor

```text
EditorPanelDescriptor
  panel_type_id
  instance_id
  title_key
  icon_token_ref
  singleton_policy
  default_dock
  minimum_size
  preferred_size
  supported_contexts
  reference_surface_roles[]
  command_ids[]
  accessibility_role
```

Panel instance stateはProject gameplay dataと分離し、User Workspaceへ保存する。Panelは`OnProjectRevisionChanged`でread projectionを更新し、Commit前にlocal cacheを正規状態として扱わない。

`icon_token_ref`は[Editor UI FrameworkのIcon semantic token contract](../appendices/editor-ui-design-system-catalog.md#icon-semantic-token-contract)にある`EditorIconTokenContractV1`へexactに解決するpresentation-only fieldである。Panel typeのidentityは`panel_type_id`、localized accessible nameは`title_key`、Panelを開く／閉じるActionは`command_ids[]`とCommand Registryが所有する。Panel iconからidentity、Action、shortcut、authority、AI permissionを推測せず、source icon IDやraw SVGをPanel descriptorへ持ち込まない。

`reference_surface_roles[]`は本書§6.0のclosed roleを1件以上持つarrayであり、排他的な`panel_family` enumではない。例えばImport Inspectorは`property_form`と`diagnostics`、History／Diffは`source_text_diff`と`diagnostics`、AI Partnerは`task_proposal_review`と`diagnostics`を同時に持てる。AI、Command palette、UIAはこのroleを画面座標や表示名の代用にせず、Panel目的のread-only説明にだけ使う。

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

## 6. 標準Panel

詳細は[Editor Panel／Reference Catalog](../appendices/editor-panel-reference-catalog.md#6-標準panel)へ分離した。本節はnavigationだけを持ち、Catalog／Fixture定義を複写しない。

## 7. Workspace

### 7.1 built-in Workspace

| Workspace | 主対象 | 初期構成 |
|---|---|---|
| `AI Creator` | 初心者、高水準指示 | Game Brief、Game、AI Partner、Question／Decision、History／Diff、Problems |
| `Production` | 通常制作 | Scene／Canvas、Hierarchy／Outliner、Inspector、Asset Browser、Console、Build／Package、Animation Timeline、AI Partner |
| `Level Workspace` | World／Scene designer | World Outline、Scene／Canvas、Hierarchy／Outliner、Topology Graph、World Composition Form、Inspector、Streaming Inspector、Navigation、Physics／Collision、Map Presentation Preview、Bundle Review、AI Partner |
| `Gameplay Logic` | Designer／Programmer | Gameplay Definition Graph／Table／Form、Source、Test／Playtest、Console、AI Partner |
| `Rendering` | Technical artist | Scene／Canvas、Material、Visual Style、Environment、Render Graph、Profiler |
| `Animation` | Animator | Scene／Canvas、Animation Timeline、Animation Graph、Asset Browser、Inspector |
| `UI` | UI designer | UI Designer、Hierarchy／Outliner、Inspector、Game |
| `Debug` | Programmer／QA | Game、Session、Problems、Console、Profiler、Debug Timeline、Causality、Breakpoint／Watch、Replay、Reproduction、AI Partner |

初期構成は§6 Panel catalogの正式名だけを使い、catalog外のPanel名を追加しない。play preview相当は`Game` Panel、Diff／validation相当は`History／Diff`と`Problems`、generated API referenceは`Source` Panel内機能、Light調整は`Scene／Canvas` overlayと`Inspector`、Localization previewは`UI Designer`内機能であり、独立Panelにしない。

`AI Creator`は正式に採用する。AIを前面に出すが、Game preview、質問、Diff、validation、戻し方を常に同時表示し、chatだけで状態を隠さない。いつでも`Production`へ切り替えられ、Project変換を行わない。

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

保存先は[Naming／Project layout](../02-foundation/naming-project-layout.md)が定めるProject外のUser data root（OS user config配下、Editor User Profile owner）とし、WorkspaceをProject rootへ保存しない。Project共有は明示export／importだけにする。built-inはimmutable、User Workspaceは複数作成、複製、rename、delete、exportできる。Workspace切替は未Commit Project draftを破棄しない。

### 7.3 Editor表示localeとAI返答locale

Editor User Profile ownerはWorkspaceとは別recordで次を保存する。

```text
EditorLanguagePreferencesV1
  schema_version = 1
  editor_display_locale: system | canonical BCP 47 locale
  ai_reply_locale: follow_editor | canonical BCP 47 locale
```

C1のEditor表示localeと明示AI返答localeはexact `en-US`、`ja-JP`、Editor source localeと最終fallbackは`en-US`である。`system`はOS localeを[UI／Text／Localization／Accessibility](../07-platform/ui-text-localization-accessibility.md#113-localeとfallback)のBCP 47規則でcanonicalizeし、対応localeなら採用、未対応または取得不能なら`en-US`へ解決してtyped non-blocking Diagnosticを出す。明示した未対応localeは近似またはfallbackせずPreference変更をrejectする。初回値は`editor_display_locale=system`、`ai_reply_locale=follow_editor`とする。追加localeは[Product Plan](../00-product/product-plan.md#5-mvp-scope)のlocale set、Catalog qualification、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md#5-ai-eval-lifecycle)の多言語Evalを同時に改訂し、Preference Schemaは変更しない。

PreferenceはOS User単位でProject外へ保存し、Workspace export、Project Source、Project ChangeSet、Undo／Redo、Game Packageへ含めない。invalid tag、unknown enum、future schemaを受理せず、変更失敗時はlast-valid recordを維持する。

表示locale変更はEditor再起動を要求せず、次のcommitted UI generationでlabel、help、message、accessible display textを再解決する。Project revision、Stable target、typed value、Action可否、semantic content hash、Task、in-flight AI requestを変更しない。layout reflowは許可するが、locale変更だけでAI Proposalをstaleにしない。

## 8. AI Partner

### 8.1 常設state

AI Partnerは単なるconversation logでなく、次のstateを分けて表示する。

| State | 表示 |
|---|---|
| Intent | exact `GameIntentSessionRefV1`／`GameIntentDraftRefV1`、production subject、User要求、完了条件 |
| Questions | exact `GameQuestionRecordRefV1`／`GameAssumptionRecordRefV1`／`GameDecisionRecordRefV1`のcurrent headと未解決理由 |
| Plan | exact `GameBriefDocumentRefV1`／`GameSpecDocumentRefV1`／`GameRequirementTraceabilityRefV1`から投影したSystem／Scene／Asset／C++のproposal scopeと依存 |
| Proposal | 未Commit ChangeSet、Native／Asset Source change |
| Validation | schema、semantic、budget、Build、Test、Preview |
| `awaiting_code_owner` | Native／Shader Sourceの対象Scope、exact `role_ref`、Assignment／Approvalの不足または失効、待機／取消／Definition・prequalified Pack fallback |
| Approval | Risk、対象、権限、期限 |
| Result | exact Commit revision、Receipt、`PlaytestObservationSetRefV1`／`GameExperienceEvaluationRefV1`／`GameIterationDecisionRefV1`、Package／Device install結果、rollback |

Panelは現在selection、open Document、Problems、Playtest結果をContext候補として表示し、送信前にUserが除外できる。Project全体を毎回Providerへ送らない。

AI自然言語返答は`ai_reply_locale`をEditor表示localeと独立したPreferenceとして扱う。`follow_editor`は各Turn開始時のeffective Editor localeへ解決し、明示BCP 47値はEditor表示に関係なく使用する。会話ごとのoptional overrideはUser-local conversation metadataにだけ保存し、persistent Preference、Project、過去messageを変更しない。overrideは新しいassistant proseだけに適用し、Tool名、Schema Field、Stable ID、enum、generated code identifierは[Naming／Project layout](../02-foundation/naming-project-layout.md#21-人間言語の境界)のcanonical Englishを維持する。

```text
AiTurnLanguageContextV1
  input_language_tags[]
  requested_reply_locale
  effective_reply_locale
  preference_source: follow_editor | explicit_user_preference | conversation_override
```

各Turnはoriginal User text hashとこのContextをtraceへ保持する。これはRequest／Evidence metadataであり、Project identity、Authorization、ChangeSet targetではない。locale変更中のin-flight requestは開始時に解決した値を維持し、次Turnから新値を使う。既存messageを再翻訳または上書きしない。

Level Workspaceを含むWorld／Scene編集では、画面captureまたはPanel内部objectをAI Contextの正本にせず、Project State所有の`AuthoringSelectionContextV1`、World所有の`WorldAuthoringContextV1`、必要な`SceneSliceV1`をPreviewする。Userは送信前にProject revision、World／Scene／Spaceのexact ref、owner-typed Gameplay ref、Space-bound Viewport query、Target、field mask、omitted rangeを確認できる。`Level`表示label、Workspace instance、Panel layoutをStable refとして送らない。Context生成後にProject revisionまたはselectionが変わった場合、pending promptとProposalをstale表示し、自動で新しい対象へ付け替えない。

AI PartnerはEngine validation、AI proposal、User selection、Runtime stateを一つのstatus表示へ畳み込まない。validationはseverity rail／icon、AI proposalはviolet dashed outline＋`AI提案` badge＋before→after、selectionはblue fill、Runtimeはcyan `Runtime` badge、staleはdotted border＋clock＋base/current revisionで同時に見せる。stale ProposalのCommitはdisabledにし、`Rebase`または`Discard`を明示する。`Ask`／`Suggest`／`Execute Authorized`は常時textで表示し、Proposalの色やTaskの成功だけからmode・approval・Commit可否を推測させない。

#### 8.1.1 Task／Proposal／Review Reference Contract

`task_proposal_review`は、AI Partnerが独自のTask／Approval／Receiptを作る場所ではない。Intent、Question、Assumption、Decision、Brief、Spec、Playtest、Evaluation、Iterationは[Game Production Loop](game-production-loop.md)のexact ref、Task／ApprovalはSecurity、EvidenceはVerification、Project revision／Proposal／CommitはProject Stateのexact refをread-onlyに投影する。Panel独自のGame理解schema、Task state、Approval state、Receipt、Project write権限を持たず、会話要約、表示名、カード、色、UIA、screen coordinate、Pattern IDからActionを発行しない。

表示は`Intent → Question／Assumption／Decision → Brief／Spec → Task → Proposal → Validation → Approval → Commit／Promotion → Playtest Observation → Experience Evaluation → Iteration Decision／Result → Receipt`の一方向とする。括弧内のApproval／PromotionはRisk、scope、Capabilityが要求する場合だけ表示する。各段階はTask ID／attempt ID、owner-issued ref／hash、base／current Project revision、semantic content hash、対象scopeで結び、[AI Security／Approval §3.2](../01-governance/ai-security-approval.md#32-task-state-machine)の15 Task stateをそのまま表示する。Build／Package／Deviceなどの`OperationTaskV1`は実行ledgerであり、Task stateのaliasにせず、Receiptから親Taskへ結果を投影する。

- unresolvedなblocking Questionがある間はProposal作成・実行Actionを公開しない。Questionの回答、Assumptionのaccept、Decisionの変更はOwnerの登録済みCommandだけが行い、会話本文、scroll、`viewed`から自動解決しない。
- ProposalはStaging上の候補であり、current Project valueを置換しない。Proposal Cardはproposal ref、base／current revision、対象scope、before／after Diff、Validation ref、Approval state、Risk、disabled reasonを同時表示する。accept／rejectはchange primitive、Document、fieldの明示単位だけに許し、部分accept後は新しいChangeSetを再構成して全Validatorを再実行する。`Accept all`、自動Commit、カードからの直接writeを禁止する。
- Project revision、semantic content hash、Task input、Contract／Policy／Profile／Tool catalog、Proposal対象、Validation結果、Approval subjectのいずれかが変わればProposal／Approvalをstaleまたはexpiredにする。stale ProposalはCommit不可で、登録済み`Rebase`または`Discard`だけを示す。Rebaseはcurrent contextから候補を再構成し全Validatorを再実行するまでApprovalを再利用しない。DiscardはStaging候補だけを破棄しProject revisionを変えない。Human rejectionはTaskを`Rejected`終端とし、修正は新Taskで始める。
- reviewの`viewed`、Workspace-local pending decision、signed decision、stale／expired Receiptは別stateである。閲覧、Difference navigation、validation pass、AI explanation、Task成功をApprovalへ読み替えない。Approval BarはRisk、exact subject hash、scope、required approver role／independence、issued／expiry、revocation／freshness、signed decisionを示す。
- Receipt Rowはpurpose、issuer、subject ref／hash、result、read-back、timeを表示するだけで、Receiptから新たな権限を導出しない。AI、Reviewer AI、visual review、UIA、pointer、keyboard、PatternはApprovalを代替しない。mutating ActionはCommand Registryがtarget、expected revision、Risk、Authorization、Approvalから投影した場合だけ公開する。

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

`Unity Scene`、`Unreal Level`、`Godot Scene`は文脈によりWorld、Scene Source、Space、`StageDefinitionV1`、`WorldStreamingPlanV1`／Cell、`UiDocument`、Composition RecipeまたはLevel Workspaceのいずれにもなり得る。候補選択がState owner、Save形式、Stage／World spatial transition、Streaming、Target Capability、Project構造を変える場合は`question_required`とし、AIは仮定でChangeSetへ進めない。Level Workspaceへの解決はpresentationだけを求める場合に限定し、Source identityを作らない。表示呼称だけの低影響差はcanonical termへ正規化できるが、未対応機能は`unsupported`として近似実装へ暗黙変換しない。

外部用語とcanonical conceptの永続的な1対1 alias、互換class、外部Scene path、Hierarchy indexをidentityとして作らない。このResolutionは入力理解のEvidenceであり、Project正本、Commit可能なOperation、Owner登録ではない。

### 8.3 Interaction mode

- `Ask`: 説明、質問、比較だけ。状態変更Proposalを作らない。
- `Suggest`: ChangeSet／Source changeを作り、Commitしない。
- `Execute Authorized`: Authorization Envelope内で検証／承認済み操作だけを進める。

Mode表示は常時visibleで、prompt本文によって自己昇格しない。AI outputをEngine validationと同じ色／iconにしない。

### 8.4 初心者workflow

1. 大まかなPromptを`GameIntentDraftRefV1`として捕捉し、production subjectとTask Authorizationを表示する。
2. `GameQuestionRecordRefV1`のBlocking／Highを質問し、Mediumだけを期限・根拠・再検証条件付き`GameAssumptionRecordRefV1`として明示選択させる。
3. User回答と「おまかせ」は`GameDecisionRecordRefV1`へ閉じ、確認済み`GameBriefDocumentRefV1`と`GameSpecDocumentRefV1`をProject ChangeSet候補にする。
4. Requirement traceabilityと`GameUnderstandingClosureRefV1`が`ready_to_stage`の場合だけ、薄い全体と一つの深いplayable loopをDefinition-firstで提案する。
5. Diff、Risk、Asset量、Target影響、AI generation lane、未解決Goalを見せる。工程・工数をArchitecture既定値として作らない。
6. 検証後にCommitし、exact Playtest Sessionへtechnical Resultと人間のObservationを別々に記録する。
7. `GameExperienceEvaluationRefV1`と`GameIterationDecisionRefV1`を表示し、`revise`では新しいTask Authorizationを取得して次iterationを始める。会話要約から旧Authorizationを再利用しない。
8. Playable確認後のTarget選択、Package、Device install、smoke結果提示は、Build／Device／Play familyのatomic Activation後にだけ同じ会話journeyで利用できる。currentでは候補Operation／Task／Receipt／UI commandを公開せず、`capability_unavailable`と対応work itemを表示する。Activation後の各段階は§11の`BackgroundTask`として進み、Receiptと成果物参照をResultへ残す。

初心者へC++／GameplayDefinition、ECS、Render Graph、ABIを選ばせない。Beginner MVPではAIが新規Native／Shader Source laneを選ばず、Definitionまたはprequalified Packで成立しないRequirementを`capability_unavailable`として示す。AdvancedでProject Sourceを明示選択した場合も、生成前の`CodeOwnerAssignmentV1`とexact Diffへの`CodeOwnerApprovalV1`はGameplay Approvalと別である。EditorはAssignmentのclosed 9-Field subject、exact `role_ref`、Scope、qualification、期間、`revoked_at=null`と、信頼済みrevocation registryの署名済みcurrent headをread-backする。Assignment Recordまたはsubject identityのcurrent revocation、snapshot missing／stale／invalid、Role欠落／unknown、RoleとScope kind不一致では`awaiting_code_owner`を表示してSource Workerを起動しない。

`activation.build_gateway.operation_pipeline.v1`完了後のPackage journeyは`operation.build.request_package` → `operation.device.install` → `operation.device.launch` → `operation.play.run_smoke`のexact順で、各段階を別`OperationTaskV1`、Authorization、署名済み`OperationReceiptEnvelopeV1`として表示する。Package成功PanelはPackage artifactとexact `RuntimeEntryDistributionPackageManifestV1` ref／hash、Installは同Package／ManifestとInstall Receipt、Launchは同Manifestの各launch対象outer Runtime Entryに対するexact `RuntimeEntryLaunchSelectionV1` ref／完成record SHA、固有Launch request／Task／Receiptを表示する。SmokeはPackage／Manifest／Install／全required Launch Selection／Launch Receiptとfixtureの全bindingが一致するまで成功表示しない。descriptor hash、entry表示名、同Manifestの別entryまたはInstall ReceiptだけからLaunch Selectionを補完しない。`operation.task.status`、`operation.task.read_receipt`、`operation.task.cancel`は選択Taskだけを対象にする。installと`operation.device.reset_data`ではDevice identity／generation、Package Receipt、削除／install対象、明示consent、R3 Approvalを確認画面に同時表示する。前段のApprovalやconsentをlaunch、smoke、Debugへ引き継いだ表示にしない。

Local inference表示はcurrent `InferenceDeploymentProfileV1.model_snapshot_profile_binding`のrecord／issuance Headからread-backした`ModelSnapshotProfileV1`だけをModel identity正本にする。weight shard closure、native／quantized encoding branch、license、provenanceをDeployment表示値から取得せず、Deployment／Snapshot binding差、`local_process_ipc`からprovider model参照、Snapshot／Conformance失効またはstale Headを`not_activated`とDiagnosticで表示する。

## 9. Manual editingとAIの往復

- GUI、Graph、Inspector、Source、AIの全変更にauthor、base revision、field sourceを記録する。
- Scene View／Level Workspaceのdirty表示はlocal draft、staged Proposal、Commit済みrevision、Derived再Cook待ちを別stateとし、一つの`*`だけで混同しない。Level Workspace自身のrevisionまたはdirty stateは作らない。
- 人間変更を既定でAI lockとせず、AIが変更する場合はDiffで明示する。
- 明示LockはAI、bulk tool、Recipe updateから保護する。
- AI proposal作成中に人間がCommitした場合、proposalをstaleとして自動Commitを禁止する。
- `Accept all`だけでなくchange primitive／Document／field単位のaccept／rejectを提供する。
- 一部accept後は新ChangeSetを再構築し、全Validatorを再実行する。
- Playtest中のruntime tweakをApply Backする場合もtyped Project change primitiveへ変換する。

### 9.1 外部IDE往復

Source Panelはfull source-code editorを必須とせず、Project C++／Shader／test source、generated public binding、diagnostic、build configurationを閲覧し、Userが選んだ外部IDEへexact file、line、column、Project revision、Engine release、Toolchain profileを渡す。外部IDEのworkspace、cache、extension、indexまたはbuild buttonをProject authorityにしない。

外部IDE向けworkspace／compile command／language-server metadataはProduct LifecycleのSDK distributionとBuild Gatewayから生成するDerived artifactである。Engine private include、vendor header、secret、User-specific absolute pathをportable Project Sourceへcommitしない。Toolchain、public API catalogまたはProject revisionが変わればstaleにし、古いindexからbuild successを推測しない。

外部編集の保存は[Project State §7.1](project-state.md#71-version-controlrepository-interoperability)のstable-read、semantic Diff、conflict、atomic ChangeSetへ戻す。Editorは未保存IDE bufferを読むと仮定せず、filesystemへ確定したbytesだけを候補にする。外部IDEのbuild／test起動も同じBuild Gateway／Developer Testing requestへ投影し、独自script、別output root、hidden environmentで成功を作らない。

Compiler／linker／test diagnosticはstable code、public API subject、Project source location、Target、configuration、candidate hashを保持し、EditorとIDEのProblemsへ同じcanonical resultを投影する。IDEがない、未対応、起動失敗の場合もSourceを失わず、exact command／request artifactとmanual setup guidanceを提示する。

## 10. Undo、History、Recovery

- Global Undo／RedoはAuthoring規約のinverse ChangeSetを使う。
- Panel内navigation、selection、camera、foldはProject Undoへ入れない。
- Continuous interactionは一つのUndo unitへcoalesceする。
- History Panelはintent、author、revision、affected objects、validation、Receipt、inverse availabilityを表示する。
- Auto-save cadenceとfocus-loss triggerは[Project stateの永続化正本](project-state.md#6-source-layoutと永続化)をexactに消費し、Workspace UXで値を再定義しない。
- Editor crash後は正規Projectを先に開き、Recovery DiffをUserへ提示する。自動Commitしない。
- GameHost／Worker crashはProblemsとTaskへ表示し、Editor layoutとProject draftを維持する。

## 11. Long-running task

Build、Cook、AI生成、Package、Device install、Testは[Core architecture](../02-foundation/core-architecture.md#91-operationtaskv1)の`OperationTaskV1`を正本とする`BackgroundTask` projectionとして次を持つ。

```text
operation_task_ref
task_id
operation_id
state = queued | running | cancel_requested | succeeded | failed | cancelled
progress_kind
completed_units/total_units
current_stage
cancel_policy
log_stream_id
artifact_refs
receipt_ref?
```

Activation後の`BackgroundTask`はrequest hash、Project revision、Candidate root、Target、Device identity／generation、Authorization、consent、idempotencyを独自保存せず、`operation_task_ref`からread-only表示する。Progress不明なのに擬似percentを表示せず、indeterminateとstageを示す。Cancelは`not_cancelable \| cooperative \| process_terminate`を明示し、UIのCancel押下を成功表示せず`operation.task.cancel`のReceiptまで追跡する。current未Activation時はTask／Cancel controlを表示せず、Modal dialogで擬似task完了までUIをblockしない。

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

- layout test: 1920×1080、2560×1440、100／125／150／175／200／250% DPI
- C1 minimum Editor window: 1280×720 logical px
- toolbar／primary action hit target: 最低32×32 logical px
- focus outline: 最低2 logical px
- Scene／Game view: 最低640×360

| Density | Body size／line | Tree／List row | Field | horizontal padding | group gap | icon |
|---|---:|---:|---:|---:|---:|---:|
| compact | 13／18 lu | 24 lu | 24 lu | 6 lu | 4 lu | 16 lu |
| standard | 14／20 lu | 28 lu | 28 lu | 8 lu | 6 lu | 16 lu |
| comfortable | 15／22 lu | 32 lu | 32 lu | 10 lu | 8 lu | 20 lu |

`effective_dpi / 96`と`editor_ui_scale`は独立axisであり、前者はOS DPI、後者はUserが選ぶ0.75～2.00のEditor UI scaleである。200% UI scaleは`editor_ui_scale=2.00`を意味し、DPI testのいずれか一件で代用しない。Textを画像へ焼き込まず、全densityでUI scale 200%時のclip、重なり、off-screen controlを0件にする。toolbar／primary actionの32 lu下限とfocus outlineの2 lu下限はdensityを下げても緩和しない。

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
| EditorHost＋同時に一つのGameHost aggregate memory | `target.windows.editor` process group soft budget内 |

Editor memory envelopeは[Performance／Capacity](../04-runtime/performance-capacity.md#3-cpu-memory-envelope)の`target.windows.editor`（`profile_version` 1）process group（metric区分、内訳、80／90／100%段階挙動を含む）をexactに消費し、本書で値と段階挙動を再定義しない。GPU memoryは同規約§4の別envelopeである。計測法・Reference環境の正本も同規約とToolchain baselineであり、本書は定義しない。

検索、thumbnail、validation、projection rebuildはcancel可能なbackground jobを使い、結果統合時にProject revisionとPanel generationを再検査する。

## 14. Failure policy

| Failure | 結果 |
|---|---|
| corrupt Workspace | last-validまたはbuilt-inへfallback、Project不変 |
| missing monitor／DPI変化 | windowをwork areaへ回収 |
| Panel exception／invariant | Panel instanceを閉じProblemsへ記録、Editor stateを部分変更しない |
| stale projection | change request reject、最新revisionから再投影 |
| AI disconnect | Manual Editor継続、pending proposalを保持 |
| GameHost crash | Editor継続、crash task／last log／再起動を提示 |
| Debug Store／Index破損 | 完全chunkまで回復し、欠損rangeをgapとして表示。推測で補完しない |
| remote capture切断 | partial captureをread-onlyで確定し、未受信range、handshake、Targetを表示 |
| recording budget超過 | policyに従い低priorityからdropし、件数とrangeを必ずEvent化 |
| Worker timeout | Task failure、cancel／retry、Project revision不変 |
| Package／Device install失敗 | Task failureとして隔離し、原因（署名、容量、device未接続／未承認）とretry／Target変更候補を提示。Project revision不変 |
| Candidate／Device generation／Package Receipt drift | Task failure。新しい対象へ自動付替えせず、before／actual identityと再実行入口を提示 |
| Code owner Role欠落／unknown／Scope不一致／失効 | `awaiting_code_owner`。exact Role／Scope差を表示し、Source生成／Promotionを停止。BeginnerにはDefinition／prequalified Pack fallbackを提示 |
| local inferenceのSnapshot ref／hash／kind／Evidence不一致またはcloud fallback要求 | 推論と送信を停止し、Snapshot差またはProvider／region／privacy／costの新PreviewとAuthorizationを提示。暗黙補正／fallbackはDiagnostic |
| invalid／future／明示未対応Editor language preference | 変更をrejectしlast-valid Preferenceを維持。Project／Workspace不変 |
| unsupported system locale | effective localeを`en-US`へ解決し、typed non-blocking Diagnosticを表示 |
| AI返答locale不一致 | Tool／ChangeSet resultと説明proseを分離し、Turnをlanguage nonconformanceとして記録。Tool resultを翻訳で補正しない |
| UI Automation provider failure | Release gate失敗。accessibilityを無効化してShippingしない |
| Recovery破損 | 隔離し正規Projectだけを開く |

## 15. Testと受入条件

- Panel edge resize、5-zone dock、tab reorder、floating、multi-monitor、pin
- 6種類以上のUser Workspace保存／切替／export／import
- monitor unplug、DPI変更、corrupt layout、missing Panelからの回復
- mouse、keyboard-only、screen readerで2D Project作成／保存／Play
- 200% scale、High Contrast、reduced motion、shortcut remap
- Tree／List、Property／Form、Canvas、Graph／Timeline、Diagnostics、Source／Text／Diff、Task／Proposal／Reviewの各Reference roleと、Shell／Docking／Navigation、Transient／State／Accessibilityが同じPanel catalogから解決される
- 全interactive Controlとsemantic composite rootがclosed Widget Pattern Registryのexact IDへ解決し、未登録variant、Panel固有state axis、Pattern内`ai_actions[]`、表示名／座標由来Actionを0件にする
- normal、hover、pressed、focus、disabled、read-only、error、warning、stale、selected、AI proposal、Runtimeを単独・重畳で区別し、validation・proposal・selection・runtime・approvalを一つの色／badgeへ畳み込まない
- compact／standard／comfortableと100／125／150／175／200／250% DPI、`editor_ui_scale=0.75／1.00／2.00`のmatrixで、primary action 32 lu、focus 2 lu、text・icon・rowの意味、keyboard／UIA targetが維持される
- `editor_display_locale=system | en-US | ja-JP`、`ai_reply_locale=follow_editor | en-US | ja-JP`、会話override、in-flight切替、unsupported system locale、corrupt／future preferenceを検証し、Project／Workspace／過去messageを変更しない
- `en-US`↔`ja-JP`表示切替で同じStable target、typed value、Action、semantic content hashを維持し、layout reflowだけを許可する
- AI CreatorからProductionへ切替えて同じObjectを手動修正し、AI再編集で保持
- Build familyのatomic Activation acceptanceでは、AI CreatorのjourneyだけでTargetを選択し、`operation.build.request_package` → `operation.device.install` → `operation.device.launch` → `operation.play.run_smoke`を別Task／Receiptで完了し、smoke結果がAI PartnerのResultへ提示される
- 同acceptanceではPackage→Install→Launch→Smokeの各前段Receipt ref／hash、Package artifact、typed distribution Manifest、outer Runtime Entry、Launch Selection ref／完成record SHA、request、Authorization、fixture、Device generationを一原因ずつ差し替えて失敗表示し、別entry Receipt、descriptor hashまたはInstall ReceiptだけによるLaunch成功を拒否してProject／Deviceの正規状態が不変。current fixtureでは全候補commandが非表示で、直接dispatchが`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`となる
- BeginnerではDefinition／prequalified PackだけからFirst Playableへ進み、Native／Shader要求は`awaiting_code_owner`または`capability_unavailable`になってSource reviewを要求しない
- Code owner Assignmentのmissing／unknown／wrong-scope `role_ref`、`revoked_at` Field省略、non-null `revoked_at`、unknown extra Field、current snapshotのAssignment／subject revoke、current snapshotのmissing／stale／invalidを一原因ずつ拒否し、`revoked_at=null`だけで`awaiting_code_owner`からSource生成／Promotionへ進めない
- expired Host／Model Profile、Deployment／Snapshot bindingのschema ID／logical ID／record ref／hash／revision／issuance Head差、`model_identity.kind`差、silent cloud fallbackを拒否し、対応状態、送信byte 0、Diagnosticを表示
- stale proposal、部分accept、human lock、Undo／Redo、external IDE conflict
- GameHost／AI／Asset Worker crash中もProjectとlayoutを失わない
- 既知のqueue overflow、stale handle、asset revision drift、simulation divergenceをSession／Debug Timeline／Causality／Replayで識別し、gapを原因確定へ使わない
- pause要求がT110 safe pointでだけ成立し、Simulation Advance／render-frame／GameplayDefinition node stepが混同されず、watch値にrevision／timepoint／validityが表示される
- AI診断のvalidated causeはすべてEvidence IDを持ち、recorded／current混同、presentation→authoritativeの逆因果、再現なし修正成功を0件にする
- Reference 01を2560×1440・100% DPI・standard density・`editor_ui_scale=1.00`のE00 DarkとE13 Lightで描画し、Outliner／Scene／InspectorのAuthoring targetとProblemsのexact Diagnostic relationが一つのStable ID／revisionへ収束する。Evidence selectionをObject selectionへ変換せず、Runtime、AI proposal、validation、staleの各axisを同時表示してもselectionとfocusの意味を失わない
- Reference 01のOutlinerで`tree-row@1`のexpanded parent、`selected + focused + runtime + warning`、`locked`（owner／reason付き）、`proposal(ai) + approval.pending + stale`を同時検証し、current hierarchy、selection、focus、Diagnostic、Action可否を一つのrow／色／badgeへ畳み込まない
- Tree Rowのplain／Ctrl／Shift selection、Left／Right hierarchy、F2 rename、before／into／after move preview、query selection、generation失効、100万Entity／10万Asset virtualizationをmanual／keyboard／UIA／AIで検証し、row index／label／path／座標由来target、silent rename、invalid drop読み替え、全row materializeを0件にする
- Tree Rowをcompact／standard／comfortable、Light／Dark／四High Contrast、`editor_ui_scale=1.00／2.00`、深さ32、長い日本語名、64文字ASCII identifier、Windows pathで検証し、24／28／32 luのrow高、full semantic name、focus ring、status、Stable targetを維持する
- `fixture.asset-browser.collection@1`の同じ六Assetをlist／tile／columnsで描画し、ready、`selected + focused + runtime + warning`、locked、`proposal(ai) + approval.pending + stale`は三View、thumbnail running／failedはtile、six cell value stateとmulti-sort priorityはcolumnsだけで検証する。View差でStable target、selection、Diagnostic、Action可否、list row高を変えない
- Asset Browserのlist↔tile↔columns切替、plain／Ctrl／Shift row selection、query select-all、list keyboard、tile spatial keyboard、columns header／cell keyboard、F2 Name限定rename、Stable ref drag source、10万Asset virtualizationをmanual／keyboard／UIA／AIで検証し、selection／focus target／focus column／scroll anchor drift、全item materialize、implicit replaceを0件にする
- Columnsの11列schema、value／pending／unknown／unavailable／redacted／not-applicable、typed comparator、最大3 sort、hidden sorted column、Stable ID tie-break、pointer／keyboard／UIA resize、Workspace-only保存を検証し、display text parse、silent sort clear、Project write、10万row auto-fit scanを0件にする
- Asset Tileのready／outdated／queued／running／failed／unavailable、last-valid、retry、reduced motionを検証し、thumbnail failureをAsset validation errorへ昇格すること、thumbnail／grid coordinateをidentityにすること、画像をAI Contextへ自動添付することを0件にする
- Asset Viewをcompact／standard／comfortable、Light／Dark／四High Contrast、`editor_ui_scale=1.00／2.00`、長い日本語名、64文字ASCII identifier、Windows path、異なるthumbnail aspectで検証し、Pane／MultipleView、List／ListItem、tile Grid／GridItem、columns DataGrid／Header／DataItem／Grid／Table、Selection、ItemContainer／VirtualizedItemの意味を維持する
- `fixture.diagnostics.reference@1`でProblemsの四severity、Consoleのrepeat／redaction／dropped gap、BuildのTask＋Diagnostic＋log、Profilerのchart-linked canonical Diagnosticを同時検証し、Task progress、metric sample、AI hypothesisのDiagnostic化を0件にする
- Diagnosticsのfilter、selection、F8、details、follow-tail、表示pause／clear、bounded copy／Export Task、virtualization、UIA polite通知をmanual／keyboard／Narrator／NVDA／AIで検証し、row index／message／timestamp由来target、UI second dedup、store削除、本文announce floodを0件にする
- Diagnosticsを三density、Light／Dark／四High Contrast、200% Font、`editor_ui_scale=1.00／2.00`で検証し、severity、selection、focus、recorded／current、Runtime、stale、gap／redactionを一色／一badgeへ畳み込まない
- `fixture.canvas.reference@1`でScene 2D／3D、Game、UI Designer、Map Previewのowner binding、navigation、selection、overlay、gizmo、Game input、safe-area／layout authorityを同時検証し、Panel固有Canvas contractを0件にする
- Canvasのplain／Ctrl／Shift pick、重なりselector、stale generation、2D／UI box select、gizmo begin／preview／commit／cancel、Game input capture／releaseをmanual／keyboard／Narrator／NVDA／UIA／AIで検証し、pixel／coordinate identity、UIA Transform Project write、partial Undoを0件にする
- Canvasを三density、Light／Dark／四High Contrast、200% Font、`editor_ui_scale=1.00／2.00`、640×360で検証し、surface focus、active／multi-selection、XYZ／UI axis、validation、proposal、Runtime、stale、input routing、safe areaを色だけへ畳み込まない
- `fixture.graph-timeline.reference@1`でGameplay／Animation／Topology／Render／Causality GraphとAnimation／Debug Timelineのowner binding、Source／Runtime／recorded、Capability activation、gap／frontierを同時検証し、Panel固有Graph／Timeline contractを0件にする
- Material／VFX／Camera／AudioのOwner Graphは共通`graph-surface@1`／Node／Port／Edge Patternへ束縛するが、初期`fixture.graph-timeline.reference@1`の五Graph、Reference 01の九Panel、166 coverage entryへ混入させない。各DomainのSource schema、read／write authority、Operation activation、Domain fixtureが完成するまでinspect／disabled reasonだけとし、共通Graph表示からCapabilityまたは汎用custom nodeを生成しない
- Graphのplain／Ctrl／Shift／marquee selection、node／edge focus、port connect／reconnect、blank compatible catalog、layout move、LOD／virtual queryをmanual／keyboard／Narrator／NVDA／UIA／AIで検証し、layout／screen座標、port色／名前、UIA Transform、planned action名由来write、implicit conversion、partial edgeを0件にする
- Timelineのtrack／key／span selection、typed time seek、scrub、multi-key move、span retime、snap、same-time stack、preview／Runtime／recorded cursor、gapを同じ入力matrixで検証し、float／display label由来time、Runtime event／root motion／Project scrub write、silent merge／trim、partial Undoを0件にする
- Graph／Timelineを三density、Light／Dark／四High Contrast、200% Font、`editor_ui_scale=1.00／2.00`、640×360で検証し、node／port／edge、track／key／span、selection、focus、validation、proposal、Runtime、stale、recorded、gapを色またはmotionだけへ畳み込まない
- `fixture.cross-panel.attention@1`でObject／Asset／Document element／Evidence／Timeのselection channel、単一Reducer、Panel follow／pin、select／focus／reveal、exact relationをmanual／keyboard／Narrator／NVDA／UIA／AIで検証し、untyped global selection、Panel間callback、受信stateの再送、暗黙related-target選択を0件にする
- Entity A固定InspectorのままEntity B、Graph node、Diagnostic、recorded timepointを順に選び、Outliner／Scene selection、Inspector context、Evidence selection、Temporal context、AI Proposal target、Runtime mappingがownerごとに維持される
- rename、re-shard、sort、filter、virtualization、target delete、permission loss、Project revision drift、Panel close、duplicate／out-of-order intent後もStable refまたは明示missingへ収束し、name／path／nearest item fallbackを0件にする
- UIA Selection／SelectionItemとkeyboard focus event、Reveal、passive `選択対象` ItemStatusを分離し、semantic no-opでeventを発行せず、AI Contextへfocus／hover／Panel boundsまたはselection由来の追加権限を含めない
- 初期六`EditorReferenceFixtureManifestV1`がRegistryへexact一件ずつ解決し、fixture source、Workspace、Panel catalog、Pattern Registry、Comparison Profile Registry、Baseline Registry、Contract set、Environment Profile、coverage matrixのwrong／missing／extra／duplicateを拒否する
- `fixture.editor.reference-01@1`をfixed UUID、revision 42、九Panel、14 Environment Profile、9 scenario、166 coverage entryへ展開し、entry count／tuple／hashまたは七typed expected subjectのwrong／missing／extra／duplicateを拒否する
- Reference 01のToolchain、A／B hardware、client extent、theme snapshot、Command／Operation／Receipt、comparison／performance profile、166 entryのBaseline Registryが一件でも未解決ならManifest／実行Registry rowを発行せず、`not_activated`と完成Manifestの`prohibited`を混同しない
- Comparison Profile Registryをexact七entryへ閉じ、oracle kind／ref／hash／comparator contractのwrong／missing／extra／duplicate、same-version drift、Runner／画像tool／UIA client／Performance harnessの暗黙既定値を拒否する
- Visualはpre-present RGBA8 sRGB、1 LSB、linear global／semantic／critical aggregate、text／non-text／focus contrast、E00..E13別baseline、空dynamic-region set、各環境30-capture repeatabilityを検証し、tolerance／mask自動拡大を0件にする
- UIAはControl View treeのRuntimeId bijectionとtrusted monotonic timestamp値だけをnormalizationし、AutomationId単独identity、event coalescing、Focus／Selection変換、Name／role／state／Action／event順の除外を0件にする
- PerformanceはA／B別のfresh process五run×120秒、nearest-rank P95のmedian、三absolute Threshold Set、5% regression、10分soakを検証し、hardware混合、平均、best run、欠測0補完を0件にする
- 各logical intentをhuman supervised、pointer、keyboard、UIA、AI semantic contractで実行し、sleepではなくtyped barrier後に同じcheckpoint／Stable target／revision／Action結果へ収束する
- `select-b`で全driverのAuthoring selection／Inspector followをBへ収束させ、pointer／keyboardの別focus eventとUIA／AIのfocus不在をdriver別expected subjectへ保持する。focus差のnormalization、focus＝selection、selection由来権限を0件にする
- authoritative、semantic、layout、visual、UIA tree／event、command trace、performanceの一planeを一件ずつ失敗させ、他planeまたはScreenshot、AI説明、Reviewer checkだけでpassにしない
- Light／Dark／四High Contrast、三density、DPI／UI scale／200% Font、reduced motion、日本語／ASCII／code／数値／path、二Windows hardwareのrequired coverage tupleをexact setで検証する
- `fixture.editor.baseline-review-01@1`を三つのexact Change Itemで開き、Ground Truth／Incoming／Difference、typed Difference list、semantic region、全required nonvisual oracleを同時表示する。selection、keyboard focus、viewed、pending decision、signed decision、stale／superseded、guard／publicationを別axisにし、三density、Light／Dark／四High Contrast、200% Font、`editor_ui_scale=2.00`で三subjectを非表示tabへ退避しない
- 初回Baselineはbaseline非依存Execution Definition→166 `captured` observation→Definition Closure→exact 166 initialize Itemへ進め、captureをpass／Verification Receiptへ読み替えず、zero Ground Truth、Manifest↔Baseline／Profile↔Qualificationのhash cycleを0件にする
- baseline変更をatomic Change Itemへ閉じ、initializeはexact 166 Item全件、replaceはcurrent Baseline Head／Registryに閉じた明示approved subsetを要求する。Itemごとのdomain owner＋independent reviewer二`ReviewReceiptV1`、24時間以内のfresh authorization、全guard passがなければpublish requestを許さず、画像click承認、Batch一括Receipt、AI pending decision、failing output、closest baseline、auto tolerance／mask、`Accept all`を0件にする
- Publication Serviceがcandidate Baseline Registry／Manifest／`EditorReferenceBaselineHeadV1`を決定論的にcompileし、全166 entryを再検証してsource Head CAS、`PromotionReceiptV1` before／after／read-back、append-only Change Registry更新を完了するまでold Headを維持する。partial initialize、stale Receipt、自動rebase、max revision／mtime／registry末尾によるcurrent推測を0件にする
- Evidence Bundleと`VerificationReceiptV1`が同じCandidate／Contract／Toolchain／Capability／Environmentへ閉じ、missing／extra coverage、infrastructure error、別profile、wrong purpose／署名をProduct evidenceへ使わない
- Reference 01のInspectorで`property-row@1`の通常scalar、`runtime_read_only + warning`、`proposal(ai) + approval.pending + stale`を同時検証し、manual／keyboard／UIA／AIが同じfield、Diagnostic、Action可否へ収束する
- Property Rowのinline／stacked reflow、mixed multi-select、invalid draft、continuous cancel、Japanese IME、長い日本語label／ASCII identifier／Windows path／large numeric＋unitでclip、silent clamp、代表値生成、partial Commitを0件にする
- Reference 01で人間のWidget操作とAIの`EditorSemanticActionV1`が同じCommandへ収束し、AIがPattern ID、UIA、screen coordinateから追加権限を得ない
- 1920×1080でScene、Outliner、Inspector、Asset、AI Partnerが同時利用可能
- `fixture.world.authoring-cross-view`の64 scenarioで、World Outline、Topology Graph、World Composition Form、Spatial View、AIが同じStableId／revision／`WorldSourceChangePrimitiveKindV1` discriminator／after state hashへ収束
- Scene永続化owner、World composition membership、Cell assignment、pack-owned `StageDefinitionV1` bindingの表示と変更経路を混同せず、共有Scene変更の影響を受けるWorld／owner-typed consumerをCommit前に列挙
- sort／filter／rename／re-shard／DPI変更後も、mouse、keyboard、UI Automation、AIがscreen coordinateまたは表示rowで別Objectを変更しない
- Derived read-only／Runtime対象の編集を全Viewで拒否し、Source Intentへの安全な遷移候補を表示
- 10万Assetのfilter／selection、multi-select共通field edit、dependency／reverse dependency表示
- 3D／Texture／Audio／Font Previewが正式Import Planと同じ変換結果を表示
- Basic／Advanced／AI／headless CLIが同じPlan hash、Diagnostic、Reimport Conflictへ収束
- Importer version、Hierarchy、Skeleton、Material、Clip、channel、coverage変更をconsumer closure付きでblock
- UIA treeのname／role／value／patternとvisual focusの自動test
- 100万Entity、10万Asset、50,000 Graph node、100,000 Timeline item、長時間TaskのPerformance fixture

C1 Editorは、初心者がAI Creatorだけで2D First Playableを作れ、経験者がProduction／Gameplay Logicで同じProjectを手動調整でき、AI panelを常設しても従来の制作PanelとScene領域を損なわない時点で完了する。

## 16. 一次資料

- [Microsoft UI Automation Providers Overview](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-providersoverview)
- [Windows Accessibility Overview](https://learn.microsoft.com/en-us/windows/apps/design/accessibility/accessibility-overview)
- [Windows Keyboard Interactions](https://learn.microsoft.com/en-us/windows/apps/develop/input/keyboard-interactions)
- [Microsoft UI Automation Support for the List Control Type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supportlistcontroltype)
- [Microsoft UI Automation Support for the ListItem Control Type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supportlistitemcontroltype)
- [UI Automation Event Identifiers](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-event-ids)
- [UI Automation Element Property Identifiers](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-automation-element-propids)
- [Microsoft UI Automation Support for the Pane Control Type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supportpanecontroltype)
- [High DPI Desktop Application Development](https://learn.microsoft.com/en-us/windows/win32/hidpi/high-dpi-desktop-application-development-on-windows)
- [Text Services Framework](https://learn.microsoft.com/en-us/windows/win32/api/_tsf/)
- [UI Automation TextPattern Overview](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-ui-automation-textpattern-overview)
- [About Text and TextRange Patterns](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-about-text-and-textrange-patterns)
- [WCAG 2.2](https://www.w3.org/TR/WCAG22/)
- [Unity Console](https://docs.unity3d.com/6000.0/Documentation/Manual/Console.html)
- [Unreal Engine Logging](https://dev.epicgames.com/documentation/unreal-engine/logging-in-unreal-engine?lang=en-US)
- [Unreal Frontend Session Console](https://dev.epicgames.com/documentation/unreal-engine/using-the-unreal-frontend-tool?lang=en-US)
- [Godot Debugger panel](https://docs.godotengine.org/en/stable/tutorials/scripting/debug/debugger_panel.html)
- [Unity Scene view navigation](https://docs.unity3d.com/6000.0/Documentation/Manual/SceneViewNavigation.html)
- [Unity Scene picking](https://docs.unity3d.com/6000.0/Documentation/Manual/ScenePicking.html)
- [Unity Scene visibility](https://docs.unity3d.com/6000.0/Documentation/Manual/SceneVisibility.html)
- [Unity UI Builder](https://docs.unity3d.com/6000.0/Documentation/Manual/UIBuilder.html)
- [Unreal Viewport Controls](https://dev.epicgames.com/documentation/en-us/unreal-engine/viewport-controls-in-unreal-engine)
- [Unreal Viewport Toolbar](https://dev.epicgames.com/documentation/en-us/unreal-engine/viewport-toolbar)
- [Unreal UMG Safe Zones](https://dev.epicgames.com/documentation/unreal-engine/umg-safe-zones-in-unreal-engine)
- [Godot Introduction to 3D](https://docs.godotengine.org/en/stable/tutorials/3d/introduction_to_3d.html)
- [Godot UI](https://docs.godotengine.org/en/stable/tutorials/ui/)
- [Godot Using Containers](https://docs.godotengine.org/en/stable/tutorials/ui/gui_containers.html)
- [Unity Animator window](https://docs.unity3d.com/6000.0/Documentation/Manual/AnimatorWindow.html)
- [Unreal Blueprint Graph Editor](https://dev.epicgames.com/documentation/en-us/unreal-engine/graph-editor-for-the-blueprints-visual-scripting-editor-in-unreal-engine)
- [Unreal Sequencer Editor](https://dev.epicgames.com/documentation/en-us/unreal-engine/sequencer-cinematic-editor-unreal-engine)
- [Godot GraphEdit](https://docs.godotengine.org/en/stable/classes/class_graphedit.html)
- [Godot Introduction to animation features](https://docs.godotengine.org/en/stable/tutorials/animation/introduction.html)
- [UI Automation control type／pattern mapping](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-controlpatternmapping)
- [Working with Virtualized Items](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-workingwithvirtualizeditems)
- [Microsoft Selection and Focus](https://learn.microsoft.com/en-us/windows/win32/winauto/selection-and-focus-properties-and-methods)
- [UI Automation Events Overview](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-eventsoverview)
- [UI Automation Events for Clients](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-eventsforclients)
- [Godot EditorSelection](https://docs.godotengine.org/en/stable/classes/class_editorselection.html)
- [Godot Inspector Dock](https://docs.godotengine.org/en/stable/tutorials/editor/inspector_dock.html)
- [Unreal Engine Outliner](https://dev.epicgames.com/documentation/en-us/unreal-engine/outliner-in-unreal-engine)
- [Unreal Editor Interface](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-editor-interface)
- [Unreal Automation Test Framework](https://dev.epicgames.com/documentation/en-us/unreal-engine/automation-test-framework-in-unreal-engine)
- [Unreal Screenshot Comparison Tool](https://dev.epicgames.com/documentation/en-us/unreal-engine/screenshot-comparison-tool-in-unreal-engine)
- [Unreal Engine Diff Tool](https://dev.epicgames.com/documentation/en-us/unreal-engine/ue-diff-tool-in-unreal-engine)
- [Unreal Engine Source Control](https://dev.epicgames.com/documentation/en-us/unreal-engine/source-control-in-unreal-engine)
- [GitHub Reviewing proposed changes](https://docs.github.com/en/pull-requests/how-tos/review-pull-requests/reviewing-proposed-changes-in-a-pull-request)
- [GitHub Approving a pull request with required reviews](https://docs.github.com/en/pull-requests/how-tos/review-pull-requests/approving-a-pull-request-with-required-reviews?apiVersion=2022-11-28)
- [Using UI Automation for Automated Testing](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-usefortesting)
- [Testing for accessibility](https://learn.microsoft.com/en-us/windows/win32/winauto/accessibility-testingtools)

WCAGはweb conformanceをEditorへそのまま宣言するためでなく、keyboard、focus、drag代替、target size等の人間工学上の最低原則として参照する。Windows desktopの実装合否はUI Automationと実assistive technologyで検証する。
