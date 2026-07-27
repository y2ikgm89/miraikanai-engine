# Editor Panel／Reference Catalog

- 文書ID: mirakan.appendix.editor-panel-reference-catalog
- 文書種別: Owner supplement
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 親Owner: [Editor Workspace／UX](../03-authoring/editor-workspace-ux.md)
- 正本範囲: Panel catalog、Reference Design、environment・coverage・baseline候補
- 非正本範囲: 親Ownerが所有する安定Architecture原則、実装Task、実装順序、生成済みArtifactまたはQualification結果
- 規範依存: [親Owner](../03-authoring/editor-workspace-ux.md)
- 関連文書: [Architecture Governance](../01-governance/architecture-governance.md)
- 根拠区分: project-decision／provisional。実ArtifactがないRegistry、Catalog、Fixtureは候補
- 外部根拠確認日: 2026-07-27

> この補助文書の型、Registry、Catalog、Fixtureは、対応するRepository Artifactが存在しない限り未実装の設計候補である。親Ownerの安定原則や実装済み状態を上書きしない。
## 6. 標準Panel

| Group | C1 Panel |
|---|---|
| World | Scene／Canvas、Game、World Outline、Hierarchy／Outliner、Inspector、Topology Graph、Level Form、Streaming Inspector、Map Presentation Preview、Environment、Bundle Review |
| Content | Asset Browser、Import、Visual Style、Material |
| Logic | Gameplay Definition Graph／Table／Form、UI Designer、Source |
| Motion | Animation Timeline、Animation Graph |
| Simulation | Physics／Collision、Navigation、Simulation Monitor |
| Production | Build／Package、Target／Device、Test／Playtest |
| Diagnostics | Session、Console、Problems、Profiler、Debug Timeline、Causality、Breakpoint／Watch、Replay、Reproduction、Render Graph、History／Diff、External Tools |
| AI | AI Partner、Game Brief、Question／Decision、Task／Receipt |

本表がC1 Panel catalogの唯一の正本である。Workspace初期構成、Command palette、`Window > Find Panel`、AIの対象解決は本表の正式名だけを使う。Panel type IDは`panel.<group>.<lower-kebab-name>`に閉じる。`Animation Timeline`（Motion）は`panel.motion.animation-timeline`、`Debug Timeline`（Diagnostics）は`panel.diagnostics.debug-timeline`であり、単一の`panel.timeline`または表示名だけの「Timeline」Panelを定義しない。

同じ機能を初心者用と上級者用に別実装しない。初心者Workspaceではadvanced Panelを初期非表示にし、AIが必要時に理由とともに開く提案をする。

### 6.0 Reference Design

次表はC1 Panel catalogを実装順や外部Engineの画面名から切り離し、人間の画面意図とAIへ渡すsemantic contextを一致させるReference Designである。§4.3のReference 01を最初の統合visual／semantic fixtureとし、色、font、icon、state overlay、density、motionのexact tokenとWidget Patternは[Editor UI Design System Catalog §15](editor-ui-design-system-catalog.md#15-design-systemと人間工学)が所有する。ここではPanelごとの情報構造、Patternの構成、見分け方だけを定め、Patternの寸法、state、Semantic、Action規則を再定義しない。

| role | 代表Panel | 基本構成と見分け方 |
|---|---|---|
| `tree_list` | Hierarchy／Outliner、Asset Browser、World Outline | hierarchyは`tree-view／tree-row`、Asset Browserのflat contentは`asset-view`の下でlist=`list-row`、tile=`asset-tile`、columns=`table-view／column-header／table-row`を使う。columnsはDataGrid conformanceがclosedになるまでCapabilityをactivateしない。いずれもrow／grid coordinateではなくStable IDを選択し、view／filter／sort後もselectionを維持する。validation rail、proposal dash、runtime badgeはselected fillの上へ重ねる |
| `property_form` | Inspector、Import Inspector、Level Form、Environment | label・value/control・unit・field statusを同じbaselineに置く。mixedはem dash、unsetは明示`未設定`、read-onlyはlock＋reason、error／warningはfield左railとinline messageで表す。before／afterはfield単位のDiffへ遷移する |
| `canvas` | Scene、Game、UI Designer、Map Presentation Preview | `surface.canvas`を背景にし、screen/world座標、camera、Target、`Source／Staging／Derived／Runtime`をcontext barへ固定表示する。selectionはblue outline、runtimeはcyan badge、gizmo／overlayはSceneの正規状態を直接書き換えない |
| `graph_timeline` | Gameplay Definition Graph、Animation Graph、Animation Timeline、Debug Timeline | graphはnode・port・edge、timelineはtrack・time ruler・keyframeを別semantic roleにする。viewport外はLOD／queryで省略し、50k nodeを一枚のsemantic treeに展開しない。selection、playhead、recorded/current、gapを形状・labelでも区別する |
| `diagnostics` | Problems、Console、Profiler、Build、Replay、Causality | severity rail＋icon＋message、対象Stable ref、recorded/current revision、timepoint、gap／redactionを一行で追跡できる。errorをAI proposalやselectionと同じviolet／blueにしない |
| `source_text_diff` | Source、History／Diff、Review | code face、line／hunk、before／after、conflict、Source revision、read-only／generatedを分離する。C1はread／select／copy／reviewを先に閉じ、IMEを伴う編集・適用・PromotionはSource Capability activation後だけにする。日本語説明とcode identifierを同じfont roleへ混在させない |
| `task_proposal_review` | AI Partner、Question／Decision、Task／Receipt、Approval | intent→question／assumption／decision→Task→proposal→validation→approval→promotion／result→receiptを一方向に表示する。AI proposalはviolet dash＋`AI提案`＋Diff、staleはclock＋base/current revision、Taskはexact stateと実測progressまたはstageを別に出す |
| `shell_docking_navigation` | Menu、Toolbar、Command palette、Status、Dock、Workspace | 主要command、panel discovery、five-zone preview、focus scope、workspace recoveryを提供する。Dock previewは最終layoutをCommitする前の一時表示であり、Panelのsemantic identityを変えない |
| `transient_state_accessibility` | menu、tooltip、overlay、notification、全Panel共通 | focus、keyboard／screen reader、High Contrast、200% UI scale、reduced motion、read-only／disabled／staleを横断して扱う。notificationはin-windowだけとし、actionなしinformationまたはowner recordを持つcompletedだけを利用者のmessage-durationでauto-dismissする。actionなし短文はStatus bar一件の`status_bar_transient`、warning／error／blocking／action付きはmanual-only＋同じowner Panel内の`owner_inline`を必須にし、floating stackを作らない。required reasonやactionをtooltipだけに置かない |

同じPanelは複数roleを持てるが、roleの数だけ別Panelや別Widget treeを作らない。Surface roleと`pattern_id`はAIの操作許可、Capability activation、Project identity、Panel instance IDを代用せず、それぞれの正本を参照する。AIは画面を画像認識、mouse macro、UIA、screen coordinateで操作せず、`EditorContextSnapshotV1`から対象、状態、許可された`EditorSemanticActionV1`を読み、状態変更はtyped Command／ChangeSetへ収束させる。Panel open、focus、reveal等の表示操作も登録済みCommandまたはfocus requestで行い、Widget callbackを直接呼ばない。

### 6.1 Hierarchy／Outliner

- 全hierarchy rowは`widget.collection.tree-row@1`を使う。World OutlineとAsset Browserのhierarchical viewも同じselection、focus、expand、rename、move preview、virtualization、Semantic／UIA契約を消費し、Panel固有row variantを作らない。
- World／Scene／EntityをWorld Modelのparent関係で表示する。
- search、type／tag／validation filter、multi-select、rename、reparentを持つ。
- selectionとfocusは別stateとし、StableId集合、active item、range anchor、collection／filter／sort revisionを保持する。hidden selectionは件数を示し、表示row index、同名Object、Hierarchy pathから選択を再構成しない。
- renameはF2＋local draft＋Enter Commitとし、Escape、selection／revision drift、Panel closeでsilent Commitしない。sort／filter後もStableIdでfocusを再解決する。
- drag reparentはbefore／into／afterを区別し、cycle、Scene boundary、persistent owner、lock、capability、permission、expected revisionをdrop前にvalidateする。hover auto-expandは行わず、invalid zoneを別zoneへ読み替えない。
- drag代替としてkeyboard／context menuのMove Commandを持ち、pointer、UIA、keyboardを同じpreview validationとtyped Move ChangeSetへ収束させる。
- 100万Entity fixtureではvisible range＋前後2 viewportだけのvirtualized treeとincremental projectionを使い、AI／UIA queryのために全rowをmaterializeしない。

### 6.2 Inspector

- 全field rowは`widget.property.property-row@1`、section headerは`widget.property.property-group-header@1`を使う。Panel固有のlabel列、mixed、validation、proposal、read-only、Runtime表現を作らない。
- MCD field metadataからlabel、unit、range、enum、reference picker、help、live-edit policyを生成する。
- multi-selectは共通fieldとmixed valueを表示する。
- continuous dragはpreview valueをlocal draftへ書き、pointer release／Enterで一ChangeSetとしてCommitする。
- Escapeでdraftを破棄する。
- invalid fieldをclampして成功させず、範囲と修正候補を示す。
- AI source、human lock、default、override、runtime-onlyを視覚的に区別する。
- inline／stacked reflow、focus order、standard UIA child control、Semantic Action投影はUI Framework §15.6.1を消費する。InspectorはPattern ID、表示label、screen coordinateからAI Actionまたは権限を生成しない。

### 6.3 Scene／Canvas

Canvas型の候補正本は[Editor UI Design System Catalog §15.6.15～§15.6.19](editor-ui-design-system-catalog.md#15615-widgetcanvascanvas-surface1)とし、Scene、Game、UI Designer、Map Presentation Previewは同じ`canvas-surface@1`、`context-bar@1`、`selection-overlay@1`、`gizmo@1`をowner別variantとして合成する。Panel固有のpick、selection outline、axis色、Game input capture、UI layout dragを追加しない。

- Sceneは`scene_2d | scene_3d`としてSelect、translate、rotate、scale、frame、measure、grid／snapを持つ。2D／3D coordinate、meter／radian、pixel／PPU、world／local／parent、pivot、snapをcontext barへ明示し、editor cameraのpan／orbit／zoomをSource Camera editへ変換しない。
- Gameは`game_preview`としてPlay Session、Build、Target、Runtime Camera、resolution／aspect、Simulation Advance／render frame、`ゲーム入力中 | エディタ入力`を固定表示する。Project selection／gizmoを持たず、明示captureしたfocused surface一件だけへGame inputをrouteし、Escape、focus／generation変更、GameHost crashでreleaseする。
- UI Designerは`ui_designer`としてexact UiDocument revision、C++ layout／text／semantic result、design extent、scale policy、device／orientation、safe area／cutout、locale、input profileを表示する。absolute nodeだけanchor／offsetをfree dragでき、stack／flex／grid childはregistered reorder／layout field editへ送る。Editor独自layout、Screenshot edge、device名からのsafe-area推測を使わない。
- Map Presentation PreviewはPresentation SourceまたはDerived Artifactを`map_preview`でread-only表示し、Gameplay Objective／Navigation authorityへwriteしない。編集は対応するSource Intent Viewの登録済みActionへ遷移する。
- Physics、Collider、Navigation、Light、Camera、VFX、Audio、safe areaはowner-issued overlay descriptorとして重ね、passive／pickable／editing toolを分離する。overlay visibility、Scene visibility、pickability、Source enabled、Runtime visibilityを一状態へ畳み込まない。
- Human pointerは同じcontent／view generationのbounded Pick SnapshotからStable targetを選ぶ。AI、UIA、automationはPick Snapshot、render texture、screen coordinateを使わず、`AuthoringSelectionContextV1`、`WorldAuthoringContextV1`、`SceneSliceV1`、UiDocument projectionと登録済みActionを使う。
- Gizmo操作はpointer downでbase revision／target／value／space／pivot／snapをfreezeし、drag中はlocal preview、releaseで一ChangeSetとする。Escape、capture／focus／revision／lock／device driftで全previewを破棄し、native transform pointer、Runtime object、moveごとのProject writeを行わない。
- Play中の編集はfieldの`live_edit_policy`を表示し、`allowed | preview_only | restart_required | runtime_read_only`を区別する。Runtime表示が選択色またはAI proposalへ吸収されないよう、context barと対象へplay glyph＋`Runtime` textを残す。

### 6.3.1 Level Authoring View

Level WorkspaceはWorld／Level／Map規約の同じSourceを次のProjectionで編集する。

| View | 編集対象 | planned change action／制約 |
|---|---|---|
| World Outline | World、Region、Level、Scene | StableId selection。Scene永続化ownerとLevel membershipを別columnで表示 |
| Topology Graph | Level、Portal、entry／exit Anchor | `CreatePortal`／`UpdatePortalContract`／`DeletePortal`。片側edgeだけを保存しない |
| Level Form | Source Scene集合、entry／exit、System、Objective、Profile、Budget | `SetLevelSourceScenes`、`SetLevelEntryExitContract`、`SetLevelGameplayComposition` |
| Spatial View | Entity、Anchor、bounds、Scene owner | typed Transform change primitive、`MoveEntityToScene` primitive。Level membershipとCellを暗黙変更しない |
| Streaming Inspector | Cell、residency、dependency、memory／IO | Target別Derived Planのread-only projection。Source編集欄を持たない |
| Navigation Overlay | walkability、cost、query、Source／Artifact差 | Source Intentだけをtyped change primitiveで変更し、Navmesh／tileへwriteしない |
| Map Presentation Preview | minimap、world map、marker、fog | Presentation Sourceだけを変更し、Quest／Objective／Navigation authorityへwriteしない |
| Bundle Review | Requirement、Source Diff、Topology、Target、Budget、Test、Risk | Staging Bundleのaccept／reject。Commit権限を持たない |

各ViewのContext barはProject revision、World／Scene／Level Stable ref、Target、lock、`Source | Staging | Derived read-only | Runtime`を常時表示し、Authoring規約の`AuthoringSelectionContextV1`とWorld規約の`WorldAuthoringContextV1`を同じContext hashで結ぶ。Scene／Outlinerは同じ`authoring_target`をStableIdで共有し、Graph／Form／Inspectorは§6.8のtyped channel、Panel binding、exact relationで関連付ける。一つのselectionへ統合せず、screen coordinate、表示row、同名Object、Hierarchy pathをtarget identityにしない。

共有Sceneを複数Levelが参照する場合、編集の影響を受ける全LevelとTargetを操作前に表示する。Scene間Entity移動では永続化owner変更、参照closure、lock、Recipe overrideをPreviewし、Level membership変更は別change primitiveとして明示する。Derived read-only対象へのdrag／property edit／pasteは拒否し、対応するSource Intent Viewへの遷移候補を示す。

### 6.3.2 Environment Panel

Environment Panelは[Environment／Water／Weather／Snow](../06-rendering/environment-surfaces.md)の`EnvironmentWorldBindingRefV1`と`EnvironmentProfileRefV1`を選択Worldへexact投影する。上部Context barはProject revision、World ref、binding ref、Profile ref、Target、quality tier、Source／Derived／Runtime状態を常時表示し、配列先頭、表示名、直前のEditor選択からProfileを推測しない。

- 基本表示はIntent／Preset、World binding、visibility、time、cloud、cost、fallback、Diagnosticである。詳細表示は同じcanonical field refのAtmosphere、Height／Volumetric／Local Fog、IBLを追加するだけで、別の簡易設定や既定値を持たない。
- fieldは`property-row@1`、Preset候補はStable ref付きbounded collection、before／afterはHistory／Diff、視覚結果はScene／Canvasの同じTarget／Profile revisionへ投影する。Panel内に第二のPreview state、Renderer settingまたはLight Sourceコピーを作らない。
- Environment semantic actionは完全登録済みOperationへ解決できるActivation後だけenabledにする。Water、Weather、SnowのSource projectionは表示できるが、対応Owner familyが未登録の間は`capability_not_activated`理由付きread-only Gapであり、labelやcontrolからfuture Operationを生成しない。
- Time-of-Dayがsun／moon変更を伴う場合はexact companion `ResolvedLightPlanRefV1`と同じrevision／World／Targetを表示する。Planがmissing／staleならEnvironmentだけのApplyをdisabledにし、質問またはLighting discoveryのActivation不足を示す。

Environment PanelはC1 Panel catalog上の正式な情報設計だが、`fixture.editor.reference-01@1`のexact九Panel instanceへ追加せず、同fixtureの166 coverage entry、九Panel countまたはBaselineを変更しない。Capability activationと専用Reference fixtureはProduct／UI Ownerの別登録が成立するまで未materializedとする。

### 6.4 Source／Text／Diff

`source_text_diff`は、Source Capabilityを有効化するための根拠ではなく、既存Ownerが発行したSource／revision／Diffを人間、UIA、AIへ矛盾なく投影するReference Contractである。C1はcommitted Source、staging candidate、generated／historical projection、外部三者比較結果を`read_only`で表示し、人間／UIAのread、search、select、copy、focus、登録済みrevealだけを提供する。これは現在exact `[]`のAI Authoring read／action集合をcallableにしない。Insert／Delete／Paste／Save／Apply／Commit／Promotion、AI Semantic mutation、UIA mutationは公開しない。`UTF-8 editor`、Build／Test、Source worker、外部IDEからの編集取り込みを実Actionとして出すのは、対応するSource operation family、Policy、Validator、Receipt、sandboxが同一Activationで有効になった後だけとする。

一つのSource lineまたはDiff hunkは、owner-issued document／source subject ref、displayed revision、comparison base revision（該当時）、current revision、range／hunk projection refへ束縛する。path、表示行／列、screen位置、検索結果順、external IDE linkはpresentationであり、identity、Authority、write targetを代用しない。`generated`、`derived`、`runtime`、`policy_locked`、`capability_not_activated`のread-only reasonと、`stale`、`superseded`、three-way conflictは別に表示し、read-only、disabled、locked、stale、conflictを一つのdisabled表示へ畳み込まない。

Source／History／Diff／Reviewは[Editor UI Design System Catalog §15.6.31](editor-ui-design-system-catalog.md#15631-sourcetextdiff-pattern-contract)の`multi-line-text-editor@1`、`source-line@1`、`diff-hunk@1`を使う。read-only textはUIA Text／TextRangeとselection／copyを維持する一方、TSF Document Managerへの接続とIME compositionはeditable Controlだけに限定する。AIがboundedなSource／Document／Diff projectionと登録済みtyped Actionを使えるのは対応familyのActivation後だけであり、C1はSource AI Actionをexact `[]`に保つ。いずれの場合もAIはUIA provider、text buffer、screen coordinate、line numberからmutationや権限を得ない。

外部IDE連携はActivation後にProject／file／lineを開く登録済みActionとしてのみ提供する。外部変更は[Project State §7](../03-authoring/project-state.md#7-undoredo外部編集競合)のUTF-8・schema検証、base／external／currentの三者比較、typed `ExternalTool` ChangeSet、人間確認または事前承認Policyだけを使う。自動merge、implicit save、片側採用、Editor text bufferからのSource Promotion／binary loadを行わない。conflictはbase／ours／theirsと未解決状態を明示し、将来のeditable stateでも明示解決、新baseでのProposal再構成、全Validator再実行なしにApply／Commitしない。

### 6.5 Asset Browser／Import Inspector

Asset BrowserとImport Inspectorは[Asset lifecycle](../03-authoring/asset-lifecycle.md)が所有するStable ID selection、`AssetSourceAnalysisV1`、`AssetImportProfileV1`、`AssetConversionReportV1`、`AssetReimportConflictV1`を投影する。

- Asset Browserはtype、semantic role、tag、license、Production readiness、diagnostic、dependencyでfilterできる。
- Sources／folder hierarchyは`widget.collection.tree-view@1`＋`tree-row@1`、flat contentは`widget.collection.asset-view@1`をrootとしてlist modeで`list-row@1`、tile modeで`asset-tile@1`、columns modeで`table-view@1`＋`column-header@1`＋`table-row@1`を使う。columnsは[Editor UI Design System Catalog §15.6.9](editor-ui-design-system-catalog.md#1569-required-collection-conformance-scenarios)を満たす実装だけactivateし、それまでは`capability_unavailable`としてlist rowへfallbackしない。
- view切替はProjectを変更せず、同じStable ID selection、active／focus target、columnsのfocus column、range anchor、filter／sort revision、scroll anchorを維持する。thumbnail、path、row index、tile row／column、visible column indexをidentityにしない。
- list rowはname、kind、logical directory、readiness、Diagnostic要約を一行にし、Source／Import revision、Target residency、dependency／reverse dependencyの詳細は同じStable targetのInspectorへ投影する。見た目だけ列揃えしてTable／DataGridへ偽装しない。
- tileは三densityの固定size、aspect-preserving contain、current name、type icon、readiness／runtime／validation statusを使う。real-time thumbnail、tile内orbit／scrub／playbackは行わず、Preview Panelへ遷移する。
- columnsはName／Kind／Readiness／Diagnostics／Target residencyを既定表示し、logical directory、Source／Import revision、active generation、license、dependency countを任意列にする。列schema、欠損値、最大3 key sort、pinned Name、resize、cell keyboard、DataGrid／Header／Grid／Table semanticsはUI Framework §15.6.6～§15.6.8をそのまま消費し、Panel固有のcolumn ID、formatter、null表示、shortcutを作らない。
- thumbnailはrevision付き非正本projectionである。ready／outdated／queued／running／failed／unavailableをAsset validationと分離し、outdatedはlast-valid＋`Preview outdated`、failedはtype icon＋`Preview failed`＋retry／Problems Actionを表示する。
- list／tile／columnsは同じquery selectionを使い、`Ctrl+A`でcurrent filter全件を10万item列挙せず選ぶ。tileのspatial keyboardはcurrent committed grid layout、columnsのcell focusとrow selectionは別state、range selectionはcanonical sort orderを使い、resize／DPI／column visibilityが変わっても既存selectionを変更しない。
- list row／tileはStable ref drag sourceになれるがAsset上のdropをreplace／reimportへ推測しない。folder moveはTree Row、external importはcontainer drop surface、明示Operationはregistered Actionへ分離する。
- thumbnail、waveform、font sample、3D turntableは選択補助であり、change primitive targetにはStable IDを使う。
- Import Inspectorは`Source`、`Analysis`、`Profile`、`Preview`、`Conversion`、`Dependencies`、`Diagnostics`、`History`を持つ。
- Import Inspectorのtyped fieldも`property-row@1`を使い、Basic／Advancedで同じcanonical field ref、draft、validation、proposalを保持する。Basic非表示fieldを別既定値へ補完しない。
- Basic viewはProfile候補とHigh Impact質問、Advanced viewは同じDocumentの全型付きfieldとevidenceを表示する。別設定を持たない。
- 3D PreviewはSource／Engine軸、Root、Pivot、bounds、ground、Hierarchy、Skeleton、Animation rootを表示する。
- Texture Previewはsource／scene-linear／Target compressed、channel、alpha、normal、mip、Sprite pivot／PPUを表示する。
- Audio Previewはwaveform、loop、loudness、true peak、channel、codec A／B、stream costを表示する。
- Font Previewはrequired script、fallback、missing glyph、variation、Target raster差を表示する。
- Reimportはbefore／after、consumer closure、Importer／Profile version、lossをDiffし、破壊的Conflictを自動promotionしない。
- Import／Preview／Cook／bulk migrationはcancel可能なlong-running taskであり、partial outputを公開しない。

### 6.6 Debugging

Debug WorkspaceはDebugging Ownerのtyped Storeを投影し、Panelごとに独自log parser、別timestamp、別Object identityを持たない。選択したSession、Project revision、Build、Target、Simulation Advance sequence／render frame ID、recorded／current stateを上部Context barで固定表示する。

- Sessionは接続、recording tier、retention、gap、redaction、remote trust、crash／hang状態を表示する。
- Problemsは`diagnostic-view@1(diagnostic_set)`＋`diagnostic-row@1`、Consoleは`diagnostic-view@1(log_stream)`＋`log-line@1`＋`evidence-gap-row@1`を使う。Debugging Ownerが所有する`DebugEventEnvelopeV1`と`MirakanDiagnosticV1`をseverity／domain／phase／Stable IDでfilterし、元Event、Snapshot、source map、Replay pointへの登録済みreveal Actionを使う。表示row、message text、timestamp単独をidentityにせず、Panel固有のcollapse／resolved stateを作らない。
- Consoleのfollow-tail、表示pause、表示clearはWorkspace-only stateであり、recording、retention、canonical Storeを変更しない。UIはOwnerのsequence／dedup／suppressed countを再計算せず、dropped、not recorded、not indexed、redacted、clock uncertainを`evidence-gap-row@1`で明示する。
- Buildは`task-stage@1`と別のDiagnostics／Log viewを合成する。Task failedだけからDiagnosticを生成せず、Progress、stage、cancel／Receiptは`OperationTaskV1`、失敗原因はcanonical Diagnostic、経過本文はlog recordとして型を分離する。
- Profiler／Debug Timeline／Causalityはcounter／span／event／causal edgeをCanvas／Graph／Timeline／Tableで同じtimepointへ整列する。Profilerのsample／threshold線をDiagnosticへ変換せず、Debugging OwnerまたはValidatorが発行したcanonical performance Diagnosticだけを`diagnostic-view@1`へ併記し、presentation結果をauthoritative causeとして逆向きに結ばない。
- Breakpoint／Watchはtarget、condition、scope、hit count、suspend policyを型付きで表示する。Runtime pauseは要求時点で即時停止せずT110 safe pointで成立させ、Simulation Advance step／render-frame step／GameplayDefinition node stepを区別する。
- Replayはrecord→scrub→inspect、first divergence、recorded／current revision差分、欠損rangeを表示する。gapまたはredactionを値なしの正常状態として扱わない。
- Reproductionは選択Evidenceからbounded BundleをPreviewし、含有／除外file、secret／PII scan、hash、retention、export先を承認前に示す。
- External ToolsはIDE、PIX、RenderDoc、Perfetto、Instruments等を`ExternalDebuggerLaunchDescriptorV1`から起動し、Session／Process／Build／capture IDを戻す。外部Toolの表示だけをEngineの正本にしない。
- AI Partnerへは画面pixelや無制限raw traceではなく同じquery ref、record identity、Target、revision、gap／redactionを持つbounded `AiDebugContextV1`を渡す。AI回答はEvidence ID、仮説、反証、不足データ、confidence、次のbounded Queryを示し、未検証仮説をProblems severityへ表示しない。修正提案は既存Diff／Approval／Authoring Gatewayへ送る。

Diagnostics型のReference候補は[Editor UI Design System Catalog §15.6.10～§15.6.14](editor-ui-design-system-catalog.md#15610-widgetdiagnostics-sourcediagnostic-view1)を正本とする。Problems／Console／Build／ProfilerはPanel固有のrow、severity色、keyboard、UIA providerを追加せず、`fixture.diagnostics.reference@1`が定める同一record／Target／Evidenceへの収束を満たす。

### 6.7 Graph／Timeline

Graph／Timeline型の候補正本は[Editor UI Design System Catalog §15.6.20～§15.6.30](editor-ui-design-system-catalog.md#15620-widgetgraphgraph-surface1)とする。Graphは`graph-surface@1`＋`graph-node@1`＋`graph-port@1`＋`graph-edge@1`、Timelineは`timeline-surface@1`＋`time-ruler@1`＋`timeline-track@1`＋`timeline-span@1`＋`keyframe@1`＋`playhead@1`を合成する。Panel固有のnode、wire、key、playhead、port色、time parser、drag transaction、UIA providerを追加しない。

- Gameplay Definition Graphはcurrent `GameplayDefinition` schemaのRule／ECA、FSM等だけを表示し、Definition Stable ID、Project／graph revision、Source／Staging、Runtime projection revisionをcontext stripへ固定する。node layout、pan、zoomをGameplay意味またはAI targetにせず、node／port／edgeのtyped ChangeSetだけをProject editとして扱う。未確定のBehavior Tree等をgeneric nodeで先行公開しない。
- Animation Graphは`AnimationGraphDefinition`のstate、transition、blend、layer、parameterをowner projectionから表示する。Animation ownerのcurrent authoring Operation集合は`[]`、Capabilityは`not_activated`であるため、現状はinspectとdisabled reasonだけを出し、planned action vocabularyやfixture commandを実Actionとして見せない。将来activateしても同じGraph Patternと一ChangeSet transactionを使う。
- Animation TimelineはSource timebase、track／key／span、isolated preview cursor、optional Runtime cursorを同じPanelで別layer表示する。frame／secondはexact rationalの表示単位でありidentityではない。scrubはRuntime event、root motion、live instance、Projectを変更しない。auto-keyは将来のatomic Capability activation後も既定off、Humanの明示arm、exact Target／track／time表示を必須とし、AIはarmしない。
- Debug Timelineは`DebugSessionDescriptorV1`、record sequence、`DebugTimePointV1`、Query generation、recorded／current Project revision、gap／redaction／clock uncertaintyをread-only投影する。recorded cursorとRuntime current cursorを一playheadへ統合せず、gap bandを正常な空白、stale、warning severityへ変換しない。
- Topology GraphはWorld ownerのPortal／Anchor primitive、Render GraphはRenderer ownerのDerived plan、CausalityはDebugging ownerのbounded `CausalityGraphV1`を同じGraph Patternへ投影する。shared visual contractはownerのwrite authorityを拡張せず、Render／CausalityからProject writeを作らない。
- SourceとRuntimeのrevisionが異なる場合、Source graph／timelineをRuntime後の状態へ先行変更せず、`Source rN`、`Runtime rM`、active node／edge／cursorを同時表示する。validation rail、AI proposal dash、selection fill、focus ring、Runtime line、stale clockは別layerを維持する。
- Graphのpointer connect／reconnectとTimelineのkey／span dragはbase revision、Stable refs、owner query／snap、generationをfreezeしたlocal previewであり、release／Enter時だけ一ChangeSetを発行する。Escape、focus／capture／revision／lock／generation変更は全件cancelし、implicit conversion、nearest port、silent trim／merge、partial Undoを許可しない。
- keyboard／screen readerはvisual wireまたはscreen xを辿らず、virtualized companion List／DataGrid、node／port search、接続一覧、typed time field、Connect／Move／Retime Commandを使う。AIは同じStable node／port／edge／track／key／span、exact time、revision、registered Actionを使い、UIA、Bezier、layout座標、display label、float secondを読まない。
- `fixture.graph-timeline.reference@1`はGameplay、Animation、Topology、Render、Causality GraphとAnimation／Debug Timelineを一つのvisual／semantic基準にする。50,000 node、100,000 time itemでもviewport＋bounded queryだけをmaterializeし、total／omitted／continuationをHuman、UIA、AIで一致させる。

### 6.8 Cross-Panel State／Selection

五Reference間のselection同期、focus、reveal、Inspector pin、validation／Proposal／Runtime／recorded overlayは[Editor UI Design System Catalog §15.7](editor-ui-design-system-catalog.md#157-cross-panel-stateselection-contract)を候補正本とする。WorkspaceはPanel instanceごとの`EditorPanelContextBindingV1`と初期bindingを決めるだけで、Panel間callback、表示名／pathによるtarget推測、Panel固有global selectionを追加しない。

| Panel／surface | 既定channel／binding | 別channelがactiveなとき |
|---|---|---|
| Outliner、Scene、World Outline | `follow_channel(authoring_target)` | Object selectionを維持。別channelをselected表示へ変換しない |
| Asset Browser、Import | `follow_channel(asset)` | Asset selectionを維持。Authoring targetへ暗黙変換しない |
| Inspector | `follow_active(authoring_target | asset | document_element)` | 対応channelなら追従、Evidence／Timeなら最後のvalid contextをchip付きで維持。Userはexact contextへ`固定`できる |
| Gameplay／Animation／Topology Graph、Source／Diff | `follow_channel(document_element)` | relationがあっても明示的な`関連対象を選択`まではObject selectionを変更しない |
| Animation／Debug Timeline | `follow_channel(document_element | temporal)` | key／spanとexact timeを分離し、Scene previewは対応するTemporal projectionだけを消費する |
| Problems、Console、Build、Profiler | `follow_channel(evidence)` | row selectionはEvidenceだけを変更し、`対象を選択`／`Sourceを表示`でexact relationを明示実行する |
| AI Partner | User許可済みactive／関連channelのbounded read | focus／hover／Panel boundsをtargetにせず、selectionからauthorizationを得ない |

Reference 01でEntity AをOutlinerから選ぶとSceneは同じ`authoring_target`をoutlineし、InspectorはAへ追従する。ProblemsはAに関係するDiagnosticへ`対象を選択中`を補助表示できるが、Diagnostic rowをselectedにせず、Problems内selectionも変えない。InspectorをAへ固定してEntity Bを選ぶとOutliner／SceneはB、Inspectorはpin icon＋`固定 A`＋pinned／current revisionを表示する。固定はlockやapprovalではなく、`追従に戻す`までのPanel contextである。

Graph node、Animation key、Diagnostic、recorded timepointを選ぶと、それぞれ`document_element`、`evidence`、`temporal`だけがactiveになる。Scene／OutlinerのObject selectionは保存され、owner-issued relationがある場合だけ`関連対象を選択`を提示する。Runtime counterpart、validation、AI Proposalはselectionではなくexact Session／Build／mapping、validator revision、proposal base revisionを持つoverlayとし、選択移動による自動retargetを禁止する。

`fixture.cross-panel.attention@1`はProduction WorkspaceへOutliner、Scene、Inspector、Asset Browser、Gameplay Graph、Animation Timeline、Problems、Debug Timeline、AI Partnerを配置し、Entity A／B、Asset、Graph node、key、Diagnostic、recorded timepoint、Runtime counterpart、stale Proposalを固定する。select、focus、reveal、pin、exact relation、filter／virtualization、target delete、Project revision drift、Panel closeを順に検証し、五Reference、Human、keyboard、UIA、AIが同じAttention snapshotとowner stateへ収束するまでReference DesignのDefinition Closureを完了扱いにしない。

### 6.9 Reference Fixture Manifest／Evidence

Reference Designの実行可能な合否契約候補は[Editor UI Design System Catalog §15.8](editor-ui-design-system-catalog.md#158-reference-fixture-manifestevidence-contract)を正本とする。Workspaceは各fixtureのPanel配置、synthetic content、初期selection／focus／binding、scenario意図を所有し、oracle schema、署名、Performance thresholdを再定義しない。

初期Registryは現在Definitionを閉じた五ReferenceとCross-Panelだけを次の六Manifestへ割り当てる。`fixture.source-text-diff.reference@1`と`fixture.ai-task-proposal-review@1`はDefinition Closureの対象名であり、現在のRegistry row、placeholder manifest、planned screenshotではない。各IDはReference Contract、対応Widget Pattern、conformance scenario、七oracle、fixed source／environment／coverage tupleが同じContract setでmaterializeされた後だけ同じRegistry／Manifest schemaへ追加できる。それまではCapability activation、空oracle、想定Screenshot、別Contract setのReceiptで補完しない。

| fixture ID | Reference／固定内容 |
|---|---|
| `fixture.editor.reference-01@1` | Production Reference 01。Outliner四row、Inspector三row、Scene、Problems relation、AI Partner、同じAuthoring target／field／revision |
| `fixture.asset-browser.collection@1` | Tree／List。六Assetのlist／tile／columns、query selection、thumbnail state、typed columns、10万Asset |
| `fixture.canvas.reference@1` | Scene 2D／3D、Game、UI Designer、Map Preview、gizmo、safe area、input capture、device loss |
| `fixture.graph-timeline.reference@1` | 五Graph、Animation／Debug Timeline、50,000 node、100,000 time item、Source／Runtime／recorded／gap |
| `fixture.diagnostics.reference@1` | Problems、Console、Build、Profiler、severity、Evidence、gap／redaction、bounded export |
| `fixture.cross-panel.attention@1` | 五selection channel、Inspector follow／pin、exact relation、validation／Proposal／Runtime／recorded、revision recovery |

`fixture.editor.reference-01@1`のsourceはEntity A／B／C、親folder、locked target、Runtime target、warning／error、二targetを含む一件のstale AI Proposal、通常／runtime read-only／proposal property、A／Cを指すcanonical DiagnosticをStable refとfixed revisionで持つ。初期focusはOutlinerのEntity A row、`authoring_target=A`、Inspectorはfollow、ProblemsのEvidence selectionはemptyとする。ProblemsはA／Cとのrelationを表示するがselected rowを捏造しない。

初期六Manifestは`authoritative_state | semantic | layout | visual | uia_tree_event | command_trace | performance`の七oracleをexact required集合として持ち、各kindを最低一tuple含める。適用不能に見えるkindは省略せず、変更禁止、不在、上限未満等のtyped negative expectationで合否を閉じる。各Manifestは次のWorkspace checkpointを最低一件ずつ持つ。

| checkpoint | 必須状態 |
|---|---|
| `initial` | immutable source load、exact Workspace／Panel catalog、focus／selection／binding、pending Task 0 |
| `state-layer` | selected、focus、validation、Proposal、Runtime、recorded、stale、lockの適用可能な重畳 |
| `interaction` | logical intentをhuman supervised、pointer、keyboard、UIA、AI semantic contractで実行し、Stable target／revision／domain resultへ収束する。入力固有focus／eventは別expected subjectでexactに保持 |
| `scale-contrast` | Light／Dark、四High Contrast、三density、DPI／UI scale／200% Font、長い日本語／ASCII／code／数値／path |
| `motion` | full／reduced motionのstart／end exact virtual tick。Dock preview、Panel、Task等の許可motionだけ |
| `failure-recovery` | Referenceに適用するstale generation、target delete、provider／device loss、reload後の正本不変。Runner timeout分類は§15.8のmeta-conformanceで検証し、pass Manifestへ期待`infrastructure_error`を混入させない |
| `capacity` | Reference固有の最大collection／surfaceとPerformance Owner threshold |

Matrixはcheckpoint、driver、Environment Profile、oracleのrequired tupleを全件列挙する。Pointer、keyboard、UIA、AIを一枚のScreenshotから合格へせず、visual captureを要求しないtupleでもauthoritative／semantic／command evidenceを残す。Human-supervised tupleはReviewer確認に加えRunnerの観測artifactを必須とし、AI tupleは実Modelの自然言語成功率ではなくtyped Semantic Action contractを検証する。

#### 6.9.1 `fixture.editor.reference-01@1` concrete Manifest

本節は§4.3の文章を最初の具体Manifestへ落とす正本である。画面ごとのScreenshot集、Panelごとのscript、実行時に選ぶ「代表環境」は別入力にしない。Manifestの固定identityは次とし、`ref/hash` pairは対応Ownerの完成artifactからmaterializeする。

```text
fixture_id                    = fixture.editor.reference-01
fixture_version               = 1
owner_ref                     = exact owner.core.ui ref
contract_set_hash             = materializing Contract set
fixture_source_ref/hash       = fixture-source.editor.reference-01@1
fixture_seed                  = 2026072501
initial_project_revision      = 42
initial_project_root_hash     = canonical root of the completed fixture source
workspace_ref/hash            = workspace.production.reference-01@1
panel_catalog_hash            = current C1 Panel catalog
widget_pattern_registry_hash  = current closed Widget Pattern Registry
comparison_profile_registry_ref/hash
                              = registry.editor-reference.comparison-profile@1
baseline_registry_ref/hash    = registry.editor-reference.baseline.ref01@1
environment_profile_refs      = exact E00..E13 set below
scenario_refs                 = exact nine scenario refs below
coverage_matrix_ref/hash      = coverage.editor.reference-01@1
verification_gate_policy_ref/hash
                              = policy.verification.editor-reference@1
required_oracle_kinds         = [
  authoritative_state, semantic, layout, visual,
  uia_tree_event, command_trace, performance
]
manifest_content_hash         = §15.8.1 canonical hash
```

本書はartifactが存在しない段階でhash文字列を捏造しない。上記logical refのID／versionに加え、fixture source、Workspace、Panel catalog、Pattern Registry、Comparison Profile Registry、Baseline Registry、Contract set、Environment Profile、Coverage Matrix、Gate Policyの完成content hashが全件解決できたときだけ`EditorReferenceFixtureManifestV1`と実行Registry rowを生成する。一件でも未解決ならrowをpublishせず関連Capabilityを`not_activated`に保ち、zero hash、`TBD`、Markdown section hash、latest refを入れたManifestを作らない。初回はbaselineを参照しないExecution Definitionで166 artifactを`captured`として観測し、Definition Closure、166 initialize Change Item、Itemごとの二Review Receipt、candidate全件検証、Baseline Head CAS／Promotion read-backを通したRegistry／Manifestだけをpublishする。captureをpassへ読み替えない。完成ManifestのCapability対象外を示す`prohibited`と、未materializedを同じstateにしない。

##### Fixture sourceとStable identity

次表のUUIDはfixture sourceへ保存するexact Stable IDである。aliasは本書内の可読短記だけであり、Manifest、Command、Semantic、UIA、AIではUUID、owner、kind、revisionを持つtyped refを使う。

| alias | exact UUIDv7 | 内容 |
|---|---|---|
| `folder.root` | `019f9692-5400-7043-99f8-6606fd4bfaf5` | expanded親folder。表示名は`メインステージ／Gameplay_Prototype_ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789` |
| `entity.a` | `019f9692-5401-7c72-b192-1238eaa148d5` | 初期Authoring target。selected、focused、Runtime mapping、warning |
| `entity.b` | `019f9692-5402-7713-ac5b-8c1b1773a125` | selectableだがwriteはlocked。lock owner／reasonを持つ |
| `entity.c` | `019f9692-5403-7d21-9232-62bc70651db2` | stale AI Proposalのnegative scale change対象。current Source geometryを先行変更しない |
| `diagnostic.a` | `019f9692-5407-77e1-9bb8-e7c8bc94f973` | `entity.a`の`component.movement.max-speed`に対するwarning |
| `runtime.a` | `019f9692-5408-7202-bbe8-3de4d3d1c984` | exact Session／Build／Source mapping／generation付きRuntime counterpart |
| `proposal.ref01` | `019f9692-5409-7deb-b708-922ec3590967` | base revision 41、current revision 42、approval pendingのstale Proposal |
| `lock.b` | `019f9692-540a-7161-89a1-04061139f7c6` | `entity.b`のlock owner／reason projection |
| `relation.diagnostic-a` | `019f9692-540b-7e89-b56b-f726aee5ad48` | `diagnostic_target`として`diagnostic.a`から`entity.a`＋fieldへ向くexact relation |
| `diagnostic.c` | `019f9692-540c-75c4-92f8-9a37cb1b296c` | Proposal内の`entity.c` negative scaleに対するerror。current Source errorへ偽装しない |
| `relation.diagnostic-c` | `019f9692-540d-7e3a-89d8-9cedb0b8e942` | `diagnostic.c`からProposal target fieldへのexact relation |

Outlinerのexact四rowは`folder.root`、`entity.a`、`entity.b`、`entity.c`の順である。これは表示順をidentityにする規則ではなく、このfixtureのexpected projectionである。`entity.a`は`selected + focused + runtime + warning`、`entity.b`は`locked`、`entity.c`は`proposal(ai) + approval.pending + stale`を持つ。Proposal一件は`entity.c`の`component.transform.scale.x=-1.000`と`entity.a`の`component.identity.display-name=プレイヤーA_改`を別primitiveで持ち、base revision 41のcurrent値から表示するがProject revision 42へ適用しない。

Inspectorのexact三rowは次である。typed value、unit、before／afterを表示textからparseしない。

| canonical field ref | current／projection | stateとAction |
|---|---|---|
| `component.transform.position.x` | `12.500 m` | editable scalar。fixture edit後はexact `13.000 m` |
| `component.movement.max-speed` | Source `6.000 m/s`、Runtime `6.250 m/s` | `runtime_read_only + warning`。copy／focus可、Source write不可 |
| `component.identity.display-name` | current `Player_A`、proposal `プレイヤーA_改` | `proposal(ai) + approval.pending + stale`。Commit不可、rebase／discardだけ |

`entity.b`のInspector projectionは`component.identity.display-name=Locked_Target_B`一rowだけを持ち、`lock.b`によりread-only／locked reasonとownerを表示する。`entity.c`は`component.transform.scale.x=1.000`と`component.identity.display-name=Proposal_Target_C`を持ち、Proposalだけがscale.x=`-1.000`を提案する。B／CへAの三field、Runtime mapping、Diagnostic、Proposal targetをcopyまたはretargetしない。

Problemsは`diagnostic.a`と`diagnostic.c`をEvidence collectionとして表示するが初期Evidence selectionはemptyである。`diagnostic.a`には`entity.a`を指す`対象を選択` Actionを表示してもselected rowにせず、`diagnostic.c`はProposal validationであることをseverity label、proposal ref、base/current revisionで示す。Source metadataにはsynthetic path `C:\Miraikanai\Fixtures\Reference01\Player_A.asset`、ASCII identifier `Gameplay_Prototype_ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789`、数値`-1234567.890 m/s²`を含め、日本語UI、UI face、code face、tabular numericを同じ行で検証する。

Scene projectionは外部Assetやthumbnailを参照せず、owner-issued boundsを持つA／B／CをX=`0.000／2.000／4.000 m`に配置する。Aにはselection outline、Runtime＋warning overlay、Bにはlock status、Cにはcurrent boundsを維持したproposal dash＋stale statusを表示し、Proposal後のnegative scaleをScene正本へ先行反映しない。AI Partnerは`proposal.ref01`一件、二primitive、base/current revision 41／42、validation error、rebase／discardを表示し、Accept／Commitをenabledにしない。

`workspace.production.reference-01@1`は`panel-instance.ref01.outliner@1`、`scene@1`、`inspector@1`、`ai-partner@1`、`problems@1`、`asset-browser@1`、`console@1`、`build@1`、`animation-timeline@1`のexact九Panel instanceを固定する。短記は先頭以外も固定prefix `panel-instance.ref01.`を持つ。Outlinerは左320 lu、Sceneは中央残余、右440 luは上Inspector／下AI PartnerをQ16 `36045／65536`と`29491／65536`で縦splitする。下Dock 280 luのtab順はProblems、Asset Browser、Console、Build、Animation Timelineで、Problemsだけをactiveにする。inactive tabをsemantic treeへactive contentとして残さず、Panel instance ID、Dock node、split ratio、active tab、focus scopeをWorkspace artifactへ含める。

##### Exact Environment Profile集合

全Profileは`target.windows.editor` profile version 1、`clock.editor-reference.millisecond@1`、`locale=ja-JP`、`input_language=ja-JP`、lock済みfont／icon／renderer setを参照する。`A | B`は[Performance／Capacity §8](../04-runtime/performance-capacity.md#8-measurementregressionpromotion)の`profile.performance.windows-reference-a@1 | profile.performance.windows-reference-b@1`であり、構成値を本書へ複写しない。

各Profileの「lock済みfont／icon／renderer set」は、[Toolchain／Dependencies §2.1](../02-foundation/toolchain-dependencies.md#21-windows)の既存`target.windows.editor` visual asset tupleと同一`style_font_generation`へ解決する。tupleまたはgenerationが欠けるか不一致なら`diagnostic.toolchain.editor-visual-assets-unlocked`を発行し、capture／baseline比較を開始しない。別OS image、別font fallback、nearest DPI、別contrast theme、旧baselineを代用しない。

各Environment Profileの`reference_hardware_profile_ref`はAまたはB一件へexactに解決し、その`os_image_ref`は上記Toolchain tupleのHost UI font resolved-file evidenceが使ったCI OS imageとbyte equalityにする。GPU driver package、monitor topology／EDID、power planはPerformance Hardware Profileだけから読むため、`style_font_generation`、font／icon artifact、theme snapshotへ複写しない。この三者のhash／OS imageが不一致なら`infrastructure_error`であり、closest monitor、同GPU family、別driver、別OS imageをcapture開始条件へ読み替えない。

| 短記 | exact Environment Profile ID | hardware／topology | theme／density／scale／motion |
|---|---|---|---|
| `E00` | `env.editor.ref01.a-dark-standard-100@1` | A、2560×1440・100% single | Dark、standard、DPI 1.00、UI 1.00、Font 100%、full |
| `E01` | `env.editor.ref01.b-dark-standard-100@1` | B、2560×1440・100% single | E00と同じ |
| `E02` | `env.editor.ref01.a-dark-compact-100@1` | A、2560×1440・100% single | Dark、compact、1.00／1.00／100%、full |
| `E03` | `env.editor.ref01.a-dark-comfortable-100@1` | A、2560×1440・100% single | Dark、comfortable、1.00／1.00／100%、full |
| `E04` | `env.editor.ref01.a-dark-standard-dpi200@1` | A、3840×2160・200% single | Dark、standard、DPI 2.00、UI 1.00、Font 100%、full |
| `E05` | `env.editor.ref01.a-dark-standard-ui200@1` | A、2560×1440・100% single | Dark、standard、DPI 1.00、UI 2.00、Font 100%、full |
| `E06` | `env.editor.ref01.a-dark-standard-font200@1` | A、2560×1440・100% single | Dark、standard、DPI 1.00、UI 1.00、Font 200%、full |
| `E07` | `env.editor.ref01.a-contrast-aquatic@1` | A、2560×1440・100% single | `theme.windows.contrast.aquatic@1`、standard、full |
| `E08` | `env.editor.ref01.a-contrast-desert@1` | A、2560×1440・100% single | `theme.windows.contrast.desert@1`、standard、full |
| `E09` | `env.editor.ref01.a-contrast-dusk@1` | A、2560×1440・100% single | `theme.windows.contrast.dusk@1`、standard、full |
| `E10` | `env.editor.ref01.a-contrast-night-sky@1` | A、2560×1440・100% single | `theme.windows.contrast.night-sky@1`、standard、full |
| `E11` | `env.editor.ref01.a-dark-standard-reduced-motion@1` | A、2560×1440・100% single | E00＋`SPI_GETCLIENTAREAANIMATION=FALSE`／reduced |
| `E12` | `env.editor.ref01.a-dark-standard-multimonitor@1` | A、2560×1440・100%＋3840×2160・200% | E00から200%側へmoveするexact topology／EDID |
| `E13` | `env.editor.ref01.a-light-standard-100@1` | A、2560×1440・100% single | Light、standard、DPI 1.00、UI 1.00、Font 100%、full |

E06のFont 200%は`system_text_scale_rational=2/1`を意味するC1 visual baselineであり、Windows system text scaleの上限ではない。上限`9/4`への`TextScaleFactorChanged`遷移は[Editor UI Framework §13.2.1](../03-authoring/editor-ui-framework.md#1321-windows-system-text-scale)のPlatform Adapter conformanceで別に検証する。既存E13はLight standardのstatic visual baselineであり、225% font baselineへ転用しない。E06／E00のimageやnearest font scaleも225%のpassへ流用しない。

E07–E10は各built-in Contrast Themeのstatic visual baselineであり、起動後のtheme変更を証明しない。E13はLight standardのstatic visual baselineである。[Editor UI Framework §13.2.2](../03-authoring/editor-ui-framework.md#1322-windows-appearance-change)のPlatform Adapter conformanceは、同一running WindowでLight→Dark→Light→E07→E08→E09→E10→user-customized Contrast Theme→Lightを検証する。これはE14、coverage entry、visual baselineを追加せず、E07–E10またはE13のimage、scheme表示名、nearest Contrast Themeをcustom colorまたはruntime transitionのpassへ流用しない。

`motion_preference_snapshot_ref`は[Editor UI Framework §13.2.3](../03-authoring/editor-ui-framework.md#1323-windows-client-area-animation-preference)のexact `EditorMotionPreferenceSnapshotV1`へ解決する。E00–E10、E12、E13は`motion.windows.client-area-enabled@1`（`SPI_GETCLIENTAREAANIMATION=TRUE`／`full`）、E11は`motion.windows.client-area-disabled@1`（`FALSE`／`reduced`）を持つ。`reduced_motion`という表示名、clock profile、前回値、test runnerの既定値だけでmodeを決めない。E11はstatic visual baselineだけであり、同一running Windowでの`TRUE -> FALSE -> TRUE`設定遷移は同節のPlatform Adapter conformanceで別に記録する。このprobeはE14、coverage entry、visual baselineを追加せず、E00／E11のstatic imageをruntime transitionのEvidenceへ流用しない。

`advanced_effects_preference_snapshot_ref`は[Editor UI Framework §13.2.4](../03-authoring/editor-ui-framework.md#1324-windows-transparency-effects)のexact `EditorAdvancedEffectsPreferenceSnapshotV1`へ、`advanced_effects_policy_ref`は`effects.editor.opaque-only@1`へ解決する。E00–E13の全14 static Profileは外部Reference environmentで`UISettings.AdvancedEffectsEnabled=TRUE`を固定した`effects.windows.advanced-enabled@1`を持つが、Dark／Light、四Contrast Theme、E11を含めてdesktopを透過するsurface、backdrop、Mica、Acrylic、blurを使わない。`TRUE`をC1 materialの自動有効化、`FALSE`を別ScreenshotまたはE14の根拠へ読み替えない。同一running Windowの`TRUE -> FALSE -> TRUE`は同節のPlatform Adapter conformanceでevent／snapshot／opaque draw descriptorを別Evidence化し、E00–E13のstatic imageをruntime transitionのpassへ流用しない。

`scrollbar_preference_snapshot_ref`は[Editor UI Framework §13.2.5](../03-authoring/editor-ui-framework.md#1325-windows-automatic-scrollbar-preference)のexact `EditorScrollBarPreferenceSnapshotV1`へ、`scroll_chrome_contract_ref`は`scrollbar.chrome.editor@1`へ解決する。E00–E13の全14 static Profileは外部Reference environmentで`UISettings.AutoHideScrollBars=FALSE`を固定した`scrollbars.windows.persistent@1`を持つ。overflowするaxisだけを`persistent`、非overflow axisを`absent`とし、visual expected subjectはcurrent chrome presentationを明示する。`FALSE`を通常利用者runtimeの常時表示override、`TRUE`を別E14または静止Screenshotの自動要求へ読み替えない。同一running Windowの`FALSE -> TRUE -> FALSE`は同節のPlatform Adapter conformanceでworker callback、fresh snapshot、logical offset／extent／selection／focus、UIA Scroll、`persistent`／`revealed`／`indicator`／`hidden` chromeを別Evidence化し、E00–E13のstatic imageをruntime transitionのpassへ流用しない。

`message_duration_preference_snapshot_ref`は[Editor UI Framework §13.2.6](../03-authoring/editor-ui-framework.md#1326-windows-message-duration-preference)のexact `EditorMessageDurationPreferenceSnapshotV1`へ解決する。E00–E13の全14 static Profileは外部Reference environmentでread-backした`SPI_GETMESSAGEDURATION` snapshotを持つが、visual expected subjectは`notification.visible_initial`またはowner surfaceのpersistent recordだけとする。deadline、UIA notification event、`d1 -> d2 -> d1`の設定遷移、coalescing、redactionは同節のPlatform Adapter／Widget conformanceで別Evidence化し、固定秒数、wall clock、静止Screenshot、E14でauto-dismiss対応を推測しない。readまたはevent subscription failureはcaptureを開始しない。

Contrast theme refは名前だけで解決せず、lock済みWindows OS image上の`HCF_HIGHCONTRASTON`とcurrent `COLOR_WINDOW`／`COLOR_WINDOWTEXT`、`COLOR_HIGHLIGHT`／`COLOR_HIGHLIGHTTEXT`、`COLOR_GRAYTEXT`、`COLOR_HOTLIGHT`を含むsystem-color snapshot ref／hashを持つ。E12は`WM_DPICHANGED` transaction前後のmonitor、work area、suggested rect、Window／surface generationを固定し、E04またはE00のScreenshotを代用しない。

表中の2560×1440／3840×2160はmonitor physical modeであり、`window_client_extent_px`ではない。各Environment Profileはlock済みwork area、non-client metrics、maximize stateから得たexact client extentをFieldへ保存し、Runnerの観測値とbyte equalityにする。display modeをclient extentへ複写したり、別OS imageのframe metricsを流用したりしない。

##### Scenario、driver、checkpoint

Scenarioは毎回revision 42のimmutable sourceへreloadする。次表のdriver set記号は右列に列挙したexact binding ref集合であり、各bindingはlogical intent、actor、input channel、expected primary Intent／Command、input固有のordered auxiliary Intentを別recordとして持つ。

| set | exact driver binding refs |
|---|---|
| `D.observe` | `{driver.ref01.observe.harness@1}` |
| `D.select` | `{driver.ref01.select-b.human-pointer@1, driver.ref01.select-b.synthetic-pointer@1, driver.ref01.select-b.synthetic-keyboard@1, driver.ref01.select-b.uia@1, driver.ref01.select-b.ai-semantic@1}` |
| `D.edit` | `{driver.ref01.edit-position-x.human-pointer@1, driver.ref01.edit-position-x.synthetic-pointer@1, driver.ref01.edit-position-x.synthetic-keyboard@1, driver.ref01.edit-position-x.uia@1, driver.ref01.edit-position-x.ai-semantic@1}` |
| `D.reject` | `{driver.ref01.reject-runtime-field.human-pointer@1, driver.ref01.reject-runtime-field.synthetic-pointer@1, driver.ref01.reject-runtime-field.synthetic-keyboard@1, driver.ref01.reject-runtime-field.uia@1, driver.ref01.reject-runtime-field.ai-semantic@1, driver.ref01.reject-runtime-field.gateway@1}` |
| `D.motion` | `{driver.ref01.dock-preview.harness@1}` |
| `D.fault` | `{driver.ref01.stale-intent.harness@1, driver.ref01.uia-provider-loss.harness@1, driver.ref01.device-loss.harness@1}` |
| `D.capacity` | `{driver.ref01.capacity.harness@1}` |

Driver ID suffixは次のexact branchへ解決する。

| suffix | `driver_kind` | Editorが検証するactor／input | 追加規則 |
|---|---|---|---|
| `human-pointer` | `human_supervised` | `human／pointer` | 実入力。Runner観測とReviewer identityをBundleへ記録 |
| `synthetic-pointer` | `automation_pointer` | `human／pointer` | test processのsynthetic provenanceを別記 |
| `synthetic-keyboard` | `automation_keyboard` | `human／keyboard` | shortcut／focus traversalをexact key eventで実行 |
| `uia` | `uia_client` | `human／accessibility` | 標準property／pattern／eventだけを使用 |
| `ai-semantic` | `ai_semantic_contract` | `ai／internal_ai` | model推論なしのtyped Semantic Action call |
| `harness` | `test_harness` | `automation／test` | observe、clock、registered fault、capacityだけ |
| `gateway` | `test_harness` | `automation／test` | UIを迂回したdefense-in-depth Command rejection専用 |

`D.select`は全経路で`EditorAttentionIntentV1.kind=select_replace`、channel=`authoring_target`、target=`entity.b`へ収束する。Select intent自体はfocusを変更しない。Pointer／keyboardのinput contractが別`focus_element`を発行した場合はそのeventも、UIA／AIがfocusを発行しない場合はその不在もdriver別Semantic／UIA expected subjectへexact記録する。選択、Inspector follow、Project revision、Proposal／Runtime targetは全driverで一致し、focus差をselection差または権限差へ読み替えない。

`D.edit`は`component.transform.position.x=13.000 m`、expected Project revision 42を[Editor UI Framework §10.2.1](../03-authoring/editor-ui-framework.md#1021-command-registryのactivation境界)の`command.editor.property.commit-value@1`へ渡す。activation後は一件の`SetComponentField`とauthoring ChangeSet pipelineへ収束し、Commitは同節のtrusted-internal routeだけが実行する。`D.reject`のHuman／pointer／keyboard／UIA／AI五bindingは`component.movement.max-speed`のAction projectionを`enabled=false`、`reason.live-edit.runtime-read-only`として観測し、Command requestを発行しない。`gateway@1`だけが同target／value／expected revisionの`command.editor.property.commit-value@1`を直接要求し、同reasonのtyped rejectionへ閉じる。六bindingすべてでProject revision／root不変、ChangeSet／Receipt exact空集合とする。current Contract setへ未解決の間はこの挙動を実行済みManifest、baseline、Receiptへ読み替えない。

| scenario ref | checkpoint ref | exact期待 |
|---|---|---|
| `scenario.ref01.initial@1` | `checkpoint.ref01.initial@1` | revision 42、Authoring selection／focus A、Evidence empty、Inspector follow A、pending Task 0 |
| `scenario.ref01.state-layer@1` | `checkpoint.ref01.state-layer@1` | Outliner四row、Inspector三row、二Diagnostic、Runtime／lock／Proposalの別projection |
| `scenario.ref01.select-b@1` | `checkpoint.ref01.selected-b@1` | Authoring selection B、Inspector follow B、Project／Evidence／Proposal／Runtime target不変 |
| `scenario.ref01.edit-position-x@1` | `checkpoint.ref01.position-x-committed@1` | revision 43、Aのposition.x=`13.000 m`、一ChangeSet／一Commit Receipt |
| `scenario.ref01.reject-runtime-field@1` | `checkpoint.ref01.runtime-write-rejected@1` | revision 42／root不変、typed disabled／rejection、Receipt 0 |
| `scenario.ref01.scale-contrast@1` | `checkpoint.ref01.scale-contrast@1` | Stable target／Action不変、density寸法、reflow、clip／overlap／off-screen 0 |
| `scenario.ref01.dock-preview-motion@1` | `checkpoint.ref01.motion.full.t000@1`、`t042`、`t083`、`checkpoint.ref01.motion.reduced.t000-final@1` | full（`SPI_GETCLIENTAREAANIMATION=TRUE`）は0／42／83 tick、reduced（`FALSE`）はtriggerと同tickでfinal。drop前Workspace不変 |
| `scenario.ref01.failure-recovery@1` | `checkpoint.ref01.stale-intent-rejected@1`、`uia-provider-recovered`、`device-recovered` | stale intent拒否、old UIA provider unavailable＋fresh root一件、device再生成後も正本不変 |
| `scenario.ref01.capacity@1` | `checkpoint.ref01.capacity@1` | Reference 01 workload、Owner threshold、A／B別measurement、correctness artifact別pass |

##### Exact Coverage Matrix

次表の`×`は集合のCartesian productを全件`EditorReferenceCoverageMatrixV1.entries[]`へ展開する規範演算、`+`はdisjoint unionである。set記号は本節に列挙したexact refだけを含み、自然文pairwise、runner判断、未列挙fallbackを許さない。Scenario／checkpoint名は直前表のexact refから固定prefixと`@1`だけを省略した一対一短記、driver／environmentは本節のset／`E00..E13`短記である。Oracle短記`A/S/L/V/U/C/P`は同順に`authoritative_state／semantic／layout／visual／uia_tree_event／command_trace／performance`である。

| group | exact tuple expression | entry数 |
|---|---|---:|
| `G00 initial` | `initial × initial × D.observe × {E00,E01,E13} × {A,S,L,V,U,C,P}` | 21 |
| `G01 state` | `(state-layer × state-layer × D.observe × {E00,E13} × {A,S,L,V,U,C,P}) + (state-layer × state-layer × D.observe × {E01} × {P})` | 15 |
| `G02 select` | `select-b × selected-b × D.select × {E00} × {A,S,U,C}` | 20 |
| `G03 edit` | `edit-position-x × position-x-committed × D.edit × {E00} × {A,S,U,C}` | 20 |
| `G04 reject` | `reject-runtime-field × runtime-write-rejected × D.reject × {E00} × {A,S,U,C}` | 24 |
| `G05 scale` | `scale-contrast × scale-contrast × D.observe × {E02,E03,E04,E05,E06,E07,E08,E09,E10,E12} × {S,L,V,U}` | 40 |
| `G06 motion` | `{(dock-preview-motion,full.t000,E00),(dock-preview-motion,full.t042,E00),(dock-preview-motion,full.t083,E00),(dock-preview-motion,reduced.t000-final,E11)} × D.motion × {S,L,V}` | 12 |
| `G07 failure` | `(stale-intent-rejected × stale-intent.harness × {A,S,U,C}) + (uia-provider-recovered × uia-provider-loss.harness × {A,S,U}) + (device-recovered × device-loss.harness × {A,S,L,V})`。scenarioは全項`failure-recovery`、environmentはE00 | 11 |
| `G08 capacity` | `(capacity × capacity × D.capacity × {E00} × {A,P}) + (capacity × capacity × D.capacity × {E01} × {P})` | 3 |
| **合計** | `coverage.editor.reference-01@1` | **166** |

展開後entryが166件でない、同tuple／`coverage_entry_hash`が重複する、setに未列挙refがある、またはMatrixのtuple集合とEvidence Bundleがset equalityでない場合はManifest／runを拒否する。Markdownの短記と式は実行入力ではなく、完成Matrix artifactの166 entryだけがRunner inputである。

##### 七oracleのReference 01期待値

Coverage entryごとのsubject IDは`subject.ref01.<checkpoint-id>.<environment-id>.<driver-binding-id>.<oracle-kind>@1`から生成し、IDだけで共有しない。同じ見た目でもcheckpoint、environment、driverの一つが違えば別expected subject／hashである。

| oracle | Reference 01でexact比較する内容 |
|---|---|
| `authoritative_state` | revision 42のsource root、initial Attention generation 1、authoring=A、Evidence empty、Panel binding、owner projection。select後はauthoring=B／generation 2だけ、edit後はrevision 43とposition.xだけを変更。reject／motion／fault／device recoveryはProject root不変 |
| `semantic` | 九Panelのactive／inactive投影、Outliner四row、Inspector三row、二Diagnostic、Stable target／canonical field、full localized Name／Value／reason、直交state、Action enabled／disabled、omitted range。bounds、glyph、colorを含めない |
| `layout` | E02／E03の24／32 lu row・field、E00の28 lu、Dock寸法、focus order、inline／stacked reflow、required text／icon／hit target、clip／overlap／off-screen typed empty set。physical Screenshotから復元しない |
| `visual` | exact Environment Profileごとのpre-present RGBA8 sRGB baseline。selection fill＋focus、warning rail、Runtime badge、lock、proposal dash、stale border／clock、errorを同時に保持し、High Contrastでも形状＋icon／textの二手段を失わない |
| `uia_tree_event` | Outliner Tree／Selection／ItemContainer、TreeItem／ExpandCollapse／SelectionItem、property childのValue／RangeValue、Problems List、各Name／ItemStatus／LabeledBy／DescribedBy、correlation付きevent順。input固有focus eventはdriver別exact、passive relationをSelectionItemへしない |
| `command_trace` | driver固有actor／inputからlogical intent／Command、typed arguments、revision検査、terminal resultまでのordered trace。initial／state／scale／motionは完成empty mutation trace、selectはProject write 0、editは一Primitive／一ChangeSet／一Receipt。rejectのsurface五bindingはCommand request 0＋typed unavailable Action、gateway一bindingはCommand一件＋typed rejectionで、全件root不変／Receipt 0 |
| `performance` | initial=`workload.editor.ref01.open@1`をE00／E01／E13別、state-layer=`workload.editor.ref01.steady-state@1`とcapacity=`workload.editor.ref01.capacity@1`をE00／E01別にOwner sample policyへ比較する。E13のA hardware resultをE00またはE01へ読み替えず、全workloadはfresh process五run×120秒、nearest-rank P95のmedian、10分soak、absolute threshold＋5% regressionを必須とし、virtual clock、別GPUのbaseline、Visual passから値を作らない |

Semantic、UIA、Command subjectではdriver固有のfocus、provider event、actor／inputをnormalizationしない。全driverに要求する同値性はStable target、canonical field、typed value、Project before／after、logical intent／Command、terminal dispositionであり、異なる入力eventを同じtraceへ改変することではない。Visual baselineが0差でも別oracle failureはfailである。

##### Reference 01 Comparison Profile binding

`comparison_profile_registry_ref/hash`は[Editor UI Design System Catalog §15.8.5](editor-ui-design-system-catalog.md#1585-comparison-profile-registry)の初版Registry候補とbyte equalityにし、166 coverage entryのOracleは次の一件へだけ解決する。

| oracle | exact Comparison Profile | Reference 01の追加固定値 |
|---|---|---|
| `authoritative_state` | `comparison.editor.authoritative.byte-exact@1` | normalizationなし。revision／root／Attention／Panel／owner projectionのcompleted canonical artifactを全Field比較 |
| `semantic` | `comparison.editor.semantic.tree-exact@1` | normalizationなし。日本語、ASCII、code、数値、path、state、Action、relationをUTF-8／typed valueでexact比較 |
| `layout` | `comparison.editor.layout.q16-exact@1` | Q16.16 lu exact。24／28／32 lu、Dock、reflow、baseline、clip／overlap／off-screen empty setを比較 |
| `visual` | `comparison.editor.visual.rgba8-srgb-1lsb@1` | `dynamic-region-set.editor.empty@1`。全E00..E13で1 LSB＋linear aggregate＋contrast＋critical cue Gateを同じ値で使用 |
| `uia_tree_event` | `comparison.editor.uia.control-view-exact@1` | `normalization.editor.uia.ref01@1`。RuntimeId bijectionとtrusted monotonic timestamp値だけを正規化し、event sequence／Focus／Selectionはexact |
| `command_trace` | `comparison.editor.command.ordered-exact@1` | normalizationなし。surface五bindingのCommand 0とGateway一bindingのtyped rejectionを別traceとして保持 |
| `performance` | `comparison.editor.performance.absolute-and-regression@1` | `sample.editor.ref01.five-by-120s@1`、`aggregate.editor.ref01.median-of-five-p95@1`と三Threshold SetをA／B別に使用 |

Visual baselineは各E00..E13とcheckpoint／driver固有で、別theme、DPI、UI scale、font scale、hardware、focus traceの画像を代用しない。Reference 01ではcaretをvirtual clockで固定し、OS-owned transientをcapture surfaceへ含めず、dynamic regionを一件も登録しない。`widget.feedback.notification@1`はEditor-ownedであり、scenarioが要求する場合だけ`visible_initial`、resolved presentation route、message-duration snapshot、redaction済みsummaryをexact subjectとして比較する。OS-owned transientの除外、dynamic region、static Screenshotをin-window notificationのlayout／duration／UIA passへ流用しない。同一checkpointをfresh process三回×十captureで比較する`qualification.editor.visual-repeatability@1`に全件合格するまでbaselineをmaterializeしない。

UIA expected subjectはControl View treeのroot scope、親、AutomationId、ControlType、sibling occurrenceとStable semantic targetを保持する。raw RuntimeIdは永続identityにせずtree preorderのstable ordinalへbijectionし、timestamp値を除外してもevent件数、順序、source、kind、property／pattern、old／new value、correlationを変えない。AutomationId単独、RuntimeId、localized Name、row indexからtargetを復元しない。

Performanceのworkload／corpus／warm-up／sample／threshold／aggregation／soakは[Performance／Capacity §8.1](../04-runtime/performance-capacity.md#81-editor-reference-01-performance-profile)を正本とする。Workspaceは値を複写せず、Performance subjectの全ref/hashを同節の完成artifactへ解決する。A／B hardwareが`characterization_only`、capacity corpus未完成、baseline未承認、五runまたはrequired metric不足のいずれかならPerformance resultをpassにしない。

##### Materialization／Activation barrier

`fixture.editor.reference-01@1`を`required`へ変更できるのは、次のexact closureが同じContract set transactionで完成した場合だけである。

1. Toolchain OwnerがSegoe UI Variable／Yu Gothic UI resolved file、Noto Sans Mono CJK JP、Widget／Panel／Command presentationの全Editor-owned system icon consumer→`EditorIconTokenContractV1`→Fluent source ID／variant／usage context mappingとconversion、およびfont解決に使うWindows OS imageをhash付きでlockする。GPU driver／monitor EDIDはToolchain artifactへ複写せず、手順2のhardware profileだけが所有する。
2. Performance OwnerがA／B hardware profile（NVIDIA／AMD driver package、monitor topology／EDID、OS imageとのToolchain equalityを含む）、combined UI allocation model、三workload、capacity corpus、warm-up、sample、三Threshold Set、aggregation、soakとPerformance expected subject候補を完成させる。候補をapproved baselineと呼ばず、手順5の共通Review／Publicationへ渡す。
3. UI Ownerが九Panel instance、全interactive／semantic rootのPattern解決、clock、14 Environment Profile、9 scenario、全driver binding、全checkpoint、七oracle subject候補、exact七entryのComparison Profile Registry、visual LUT／contrast／empty dynamic-region set、UIA normalization profile、初回Baseline Definition Closure、Baseline Review surfaceのPattern／Semantic Actionを完成artifactにする。
4. Command／Authoring Ownerが`command.editor.property.commit-value@1`、`operation.authoring.changeset.commit`、typed rejection、Change Primitive／Receipt経路をcurrent Contract setへ解決する。
5. Registry compilerが166 entryのMatrix、Comparison Profile、baseline非依存Initial Execution Definition／Observation Bundle／Definition Closure、Change Item、Review policy、Gate Policyをcontent hashで閉じる。初回は166 Item全件にdomain owner＋independent reviewerの二Receiptを要求し、Publication Serviceがcandidate Baseline Registry／Manifest／Baseline Headをcompileして全166 entryを再検証し、source Head CAS／Promotion read-back後にだけ`registry.editor-reference.baseline.ref01@1`とManifestをpublishする。信頼済みRunnerはその完成ManifestからEvidence Bundleと`VerificationReceiptV1`を発行できる。

一部だけ完成した状態を`definition_only pass`、Screenshot-only preview、characterization passにしない。未完成中は実行Registry rowを発行せず、本節を実装順序ではなくDefinition Closureとして扱い、Capability activation、Product Gate、利用可能表示を先行させない。

##### C0 Definition Closure audit

次表は上の五項をC0の設計定義として監査するための境界である。`C0 definition closed`は必要なidentity、入力閉包、比較／失敗規則、owner境界が文書上で一意に定義済みであることだけを表す。hash付きasset、hardware、Registry、Manifest、baseline、Receipt、Capabilityを実在またはactiveと主張する値ではない。全行がC0で閉じていても、`fixture.editor.reference-01@1`はC1 materializationまで`required`へ遷移せず、current execution Registry／baseline／Product Gateを生成しない。

| closure | C0で閉じる定義 | C1へ持ち越す実体化境界 |
|---|---|---|
| 1. Toolchain／visual asset | [Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)のC1 Editor visual asset lock closureにより、Host UI font、bundled Noto、全Widget／Panel／Command presentation icon consumer、Fluent source／conversion、font解決用OS image、theme／motion／scrollbar／message-duration snapshotと失効規則を一つの入力閉包として定義する。driver／EDIDはrow 2のhardware profileへ分離する | resolved file／archive／conversion outputの実hash、license read-back、Performance profileのOS imageとのexact equality、同一Profileのfixture Evidence。未取得のplaceholder、別font／icon、静的Screenshotをlockにしない |
| 2. Performance | [Performance／Capacity §8.1](../04-runtime/performance-capacity.md#81-editor-reference-01-performance-profile)により、A／B profile（driver package、monitor topology／EDID、Toolchain OS image equality）、combined allocation、三workload、corpus、warm-up、five-by-120s、QPC clock evidence、threshold、aggregation、soak、A／B別expected subjectを定義する | locked hardware／OS／driver／EDIDとcorpus、six characterization／six soak、baseline非依存Harness Qualification。`characterization_only`をapproved baselineへ昇格しない |
| 3. UI／Reference fixture | 本書の九Panel、14 Environment Profile、9 scenario、全driver／checkpoint、七oracleと、[Editor UI Design System Catalog §15.8](editor-ui-design-system-catalog.md#158-reference-fixture-manifestevidence-contract)のPattern／clock／Comparison Profile／visual・UIA規則をexact集合として定義する | 全Panel instance、Pattern／Action projection、environment snapshot、subject、LUT、UIA normalization、initial Definition Closureとreview surfaceの完成artifact。静的mockupだけをfixture passにしない |
| 4. Command／Authoring | [Editor UI Framework §10.2.1](../03-authoring/editor-ui-framework.md#1021-command-registryのactivation境界)により、property commitのtyped input、単一`SetComponentField`、outer four-operation route、request／ChangeSet correlation、unavailable result、Runtime read-onlyのsurface／Gateway分離を定義する | `activation.authoring.changeset_execution.v1`による全Operation、Diagnostic、Policy、Validator、Receipt、Command／Action／presentation bindingの同時activation。planned Commandまたはreason keyだけをcurrent authorityにしない |
| 5. Registry compiler／baseline | 166 tuple、七Comparison Profile、baseline非依存Execution Definition／Observation Bundle／Definition Closure、Change Item、two-Review Receipt、candidate全166再検証、source-head CAS／Promotion read-backの順序を定義する | 166個の初回initialize Item、各Itemの二Receipt、candidate Registry／Manifest／Head、全再実行とPublication read-back。capture、画像一致、単独Receipt、partial initializeをbaseline publicationにしない |

この監査表のC0 statusは、上の五定義が同じ意味を別文書で矛盾なく参照できるかを確認するためだけに使う。実装進捗、Task完了、Capability activation、性能合格、baseline publishのstateをこの表へ重ねない。

##### Baseline Review Reference Design

Baseline Reviewは新しいPanelを追加せず、`History／Diff` Panelの`workspace.editor.baseline-review@1` modeとして開く。入口は`Test／Playtest`のfailed result、Initial Baseline Definition ClosureまたはBaseline Change Proposalにだけ置き、Panel roleは`source_text_diff + task_proposal_review + diagnostics`の合成とする。通常のProject undo history、AI Proposal review、baseline reviewを同じmodeへ混在させず、headerでsubject kind、fixture、source `EditorReferenceBaselineHeadV1` ref/hashまたは`genesis`、source Baseline Registry ref/hash、Batch hashを常時表示する。

2560×1440 physical、100% DPI、Dark、standard、`editor_ui_scale=1.00`では、client areaを上から48 luのheader、残余review body、44 luのguard／action railへ分ける。bodyは左320 luのChange Item list、中央残余のcomparison area、右400 luのDecision Inspectorとし、8 lu gutterを使う。中央は`Ground Truth | Incoming | Difference`を同じ幅の三paneで同時表示し、各pane headerへsubject ref/hash、Environment Profile、Comparison Profile、capture／artifact kindを表示する。画像paneは同期pan／zoomと一時的なblend sliderを持てるが、slider位置、zoom、pointer位置をreview evidenceまたはdecisionにしない。initialize ItemだけはGround Truth paneを`Baseline未作成`というtyped empty stateにし、zero image、空tree、Incomingの複製を表示しない。Difference paneは`difference_kind=extra`の初回候補とprerequisite qualificationを示し、replaceと同じ既存Baseline差分に見せない。

中央paneはOracle kindで描画形式を切り替える。VisualはGround Truth／Incoming／Difference image、Semantic／UIAはtyped tree、layoutはregion／constraint table、Commandはordered trace、authoritative stateはcanonical field diff、Performanceはmetric／unit／threshold／baseline delta tableであり、非visual artifactをScreenshotへ変換して判定しない。下部のtyped Difference listは`EditorReferenceDifferenceRecordV1`のkind、semantic region、requirement、old／incoming value、severityを表示し、`Next difference`／`Previous difference`でpaneと同期する。差分、warning、failureはcolorに加えてlabel、icon、outlineまたはhatchingを使い、High Contrastでsystem colorへ置換しても意味を維持する。

responsive layoutは次の三段だけとする。

| available width | comparison／side area |
|---|---|
| `>= 1680 lu` | 左Item list、三comparison pane、右Decision Inspectorを同時表示 |
| `1120..1679 lu` | Ground Truth／Incomingを上段二分、Differenceを下段全幅。Item listは固定、Decision Inspectorはbody右またはinline bottomへreflow |
| `< 1120 lu`または`editor_ui_scale=2.00` | Item list、Ground Truth、Incoming、Difference、typed Difference list、Decision Inspectorの順へ一列stack。list／Inspectorは明示drawerへ移せるが、三比較subjectをtabで相互排他にせず、現在Itemの三者をdocument順とaccessibility treeへ保持 |

200% Fontではpane header、hash、localized labelをwrapし、ellipsisだけに依存しない。Change Item listの行高は[Editor UI Design System Catalog §15.4](editor-ui-design-system-catalog.md#154-density)のcompact／standard／comfortableで24／28／32 luとし、decision controlは全densityで32 lu以上のaction heightを維持する。pane切替、Item切替、Difference navigationは即時で、同期pan／zoomに必須animationを付けない。drawer／Panel展開だけ[Editor UI Design System Catalog §15.5](editor-ui-design-system-catalog.md#155-motion)のbounded motionを使い、reduced motionでは同tickでfinal stateへ進む。

状態は次の独立axisとして保持し、一つの`approved` badgeへ潰さない。

| axis | 値／表示規則 |
|---|---|
| active Item selection | 左listのselected fill。keyboard focus ringとは別 |
| keyboard focus | exact controlだけに2 lu ring。Item選択、pane focus、decision focusを区別 |
| review progress | `not_viewed | viewed`。Workspace-localで、承認条件や署名subjectへ含めない |
| pending decision | `none | approve | reject | request_changes`。Workspace-localのdraft表示 |
| signed decision | `none | approved | rejected | changes_requested`。Reviewer、Role、issued／expiry、Receipt hashを表示 |
| freshness | `current | stale | superseded | expired | revoked`。source head／Item hash／Role／Receiptのどれが変わったかをtyped reasonで表示 |
| guard／publication | `guard_pass | guard_fail | publishing | published | publish_failed`。review decisionと別status |

`viewed`は`approved`を意味せず、画像paneのclick、Difference navigation、scroll終端からpending decisionを自動作成しない。Pending decisionはItem一件ずつ明示設定する。`Submit review`は選択したpending Itemの件数、各Item hash、decision、unresolved issue、exact approved scopeをconfirmationへ再表示し、User presence／Approval Service後にItemごとの`ReviewReceiptV1`を発行する。一回のceremonyで複数Itemを送信できてもBatch一件の包括Receipt、`Accept all`、未閲覧Itemの自動approveを作らない。domain ownerとindependent reviewerの二Receiptが同Itemへ揃うまではapproval-completeにしない。

Review surfaceが公開するlogical actionは次に閉じる。IDがWidget Pattern RegistryとCommand Registryのcurrent Contract setへ解決し、そこからcurrent target／revision／availabilityに対する`EditorSemanticActionV1`が投影される場合だけ該当controlを表示する。`EditorSemanticActionV1`はsnapshot projectionであり独立したSemantic Action Registryではない。keyboard bindingはCommand Registryから取得し、画面へhard-codeしない。

| Action ID | actor／効果 |
|---|---|
| `action.editor.reference-review.open-batch@1` | Human／UIA／AI read。immutable Batchを開く |
| `action.editor.reference-review.select-item@1` | Human／UIA／AI read。active Itemだけを変更 |
| `action.editor.reference-review.next-difference@1`／`previous-difference@1` | Human／UIA／AI read。typed Difference selectionとpane revealだけを変更 |
| `action.editor.reference-review.toggle-sync-view@1` | Human／UIA read。Workspace-local viewport preferenceだけを変更 |
| `action.editor.reference-review.set-pending-decision@1` | Human／accessibilityのみ。Item一件のWorkspace-local draftを変更 |
| `action.editor.reference-review.submit-review@1` | Humanのみ。User presence後にApproval Serviceへexact Item decisionを送る |
| `action.editor.reference-review.publish-approved@1` | Human release authorityのみ。Publication Serviceへapproved Item exact集合を送る |

AIは選択されたsynthetic／redacted Item、typed Difference、guard result、Requirementをreadしてbounded explanationまたは新Change Proposalを作れるが、`set-pending-decision`、Review Note、Receipt、publish requestを生成しない。AI説明、Reviewerの`viewed`、Visual一致だけでguard failureまたはnonvisual diffを補完しない。Publication controlはsource head current、対象Itemごとの二Receipt fresh、全guard passを満たすときだけrequest可能にするが、最終passはPublication Serviceがcandidate Registry／Manifestをcompileして166 entry全件を再実行し、CAS／Promotion read-backを完了した結果だけである。

`fixture.editor.baseline-review-01@1`は同じreplace Batchに次のexact三Itemを持つ。logical ref/hashは完成artifactから解決し、Markdownの表示値、zero hash、仮Receiptを実行入力にしない。

| Change Item ID | 変更とtyped expectation | 期待結果 |
|---|---|---|
| `item.review01.visual-font200-error-icon@1` | `G05 scale`のE06 visual entry。validated bug fixにより200% Fontでclipしたerror iconを正しいbounds／contrastへ置換し、同impact setのSemantic／layout／UIA guardをpass | domain owner＋independent reviewerのapproved Receipt後、三Item中これだけをpublish可能 |
| `item.review01.semantic-stale-label@1` | stale stateの日本語Name／reasonを変えるDesign Contract候補だが、localized Nameのapproved contract evidenceがmissing | `changes_requested`。画像差が妥当に見えてもpublish不可 |
| `item.review01.performance-threshold-only@1` | open P95 thresholdを200 msから240 msへ緩和しようとするが、新Profile version、design evidence、全impact set qualificationがない | pre-reviewで`unauthorized_relaxation`、`rejected`。pending approve controlもdisabled |

fixtureは三Itemを`pending=none`、`signed=none`から開始し、selection／focus／viewedを別々に動かす。I00だけを二Reviewerがapproved、I01をchanges requested、I02をrejectedとしてsubmitし、I00の部分publishでcandidate全166 entryを再検証する。成功時はI00だけを新Baseline Registryへ適用し、Baseline Head sequenceをexact `N+1`へ進め、I01／I02を`superseded`にする。Receipt失効、source Head ref/hash conflict、candidate一entry failureを一原因ずつ注入したrunではold Headを維持し、Change Registryをappendしない。

各runは完成`EditorReferenceEvidenceBundleV1`を`Test／Playtest`から開けるが、BundleはProject history、Workspace save、AI conversationへ混入しない。AI PartnerはUserが選んだbounded diffだけを説明でき、missing Evidenceを推測で埋めず、baseline／tolerance／pass resultを変更しない。`fixture.product.windows-empty-scene`へ渡すのはcurrent Capability snapshotから`required`へ解決した初期六Manifestのexact subsetに対応するpass `VerificationReceiptV1`集合であり、`prohibited` Manifest、component fixture ID自体、未実行rowをProduct Gate receiptへ読み替えない。
