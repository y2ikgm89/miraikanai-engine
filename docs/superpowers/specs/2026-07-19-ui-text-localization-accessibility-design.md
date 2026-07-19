# Miraikanai Engine UI／Text／Localization／Accessibility規約

- 文書版: 1.1
- 作成日: 2026-07-19
- 対象: Game UI、Layout、Widget、Focus、Text、IME、Localization、Font、Accessibility、UI Rendering
- 状態: プロジェクト公式の規範設計レビュー版
- Authoring規約: [Miraikanai Engine Authoring Model／Project State規約](./2026-07-19-authoring-model-project-state-design.md)
- Runtime規約: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- Rendering規約: [Miraikanai Engine Rendering／Render Graphアーキテクチャ規約](./2026-07-19-rendering-render-graph-architecture-design.md)
- Asset規約: [Miraikanai Engine Asset Pipeline／Content Package規約](./2026-07-19-asset-pipeline-content-packaging-design.md)
- Input規約: [Miraikanai Engine Input／Action／Device規約](./2026-07-19-input-action-device-architecture-design.md)
- Editor規約: [Miraikanai Engine Editor／Workspace／UX規約](./2026-07-19-editor-workspace-ux-design.md)
- Editor UI Framework規約: [Miraikanai Engine 独自Editor UI Framework／Shellアーキテクチャ規約](./2026-07-20-editor-ui-framework-architecture-design.md)
- Platform規約: [Windows](./2026-07-19-windows-platform-distribution-design.md)／[Mobile](./2026-07-19-mobile-platform-architecture-design.md)

## 1. 結論

Miraikanai EngineのGame UIは、HTML／CSS／JavaScript、webview、汎用Script Widget、immediate-modeの描画命令列を正規形式にしない。Schema検証可能な`UiDocument`、typed `UiViewModel`、C++ Layout／Event Runtime、versioned Style／Font Assetから構築する独自retain-mode UIである。

TextはUTF-8を正規storageとし、次の検証済みLibraryを限定利用する。

- HarfBuzz 14.2.1: OpenType shaping
- FreeType 2.14.1: bundled Font検証、glyph index／metrics／rasterization
- ICU4C 78.3: BCP 47 locale、BiDi、grapheme／word／line boundary、plural／number／date／message formatting

Library型、Font object、ICU iterator、glyph atlas pointerをProject、Save、AI、NativeGameModuleへ公開しない。UI Document、Layout、Focus、Event、Binding、Localization schema、Accessibility semantic tree、memory／failureはMiraikanaiが所有する。

Windows Editor shellは本書の`UiRuntimeTree`、Layout、Event、Semantic contractを共有する独自`MiraUI Core`上に構築する。Editor固有のDocking、Window、DirectWrite、TSF、UI Automation、D3D12 UI passはEditor UI Framework規約を正本とする。Shipping Game UIは全Targetで本書の共通Text Layoutを使い、bundled Fontを正本にする。

## 2. 決定権と対象外

| 主題 | 正本 |
|---|---|
| UiDocument、Widget、Layout、Binding、Focus、Event、Text、Localization、Accessibility | 本書 |
| UI source Commit、ChangeSet、Undo、Projection | Authoring規約 |
| Input Action、pointer／touch、Device、Remap | Input規約 |
| UI Render pass、GPU resource、composition | Rendering規約 |
| UTF／locale／Font Library version、Build、license | 基盤規約 |
| Android／Apple lifecycle、safe area、Text Adapter | Mobile規約 |

C1ではHTML／CSS parser、DOM、JavaScript、webview、arbitrary expression binding、Runtime Font download、system Font依存のShipping layout、Rich Text markup parser、SVG Font、font editorを実装しない。C2候補はlimited rich text span、MSDF、HRTF字幕連携、advanced vector UIである。

## 3. Architectureとphase

```text
UiDocument／Style／Localization／Font Assets
  -> UiRuntimeTree
InputSnapshot + previous UiHitTestSnapshot
  -> T30 UiInteraction
  -> typed UiCommand／UiState delta
UiViewModelSnapshot
  -> T90 Binding／Layout／Text／Accessibility
  -> UiPresentationSnapshot
  -> T110 RenderSnapshot
  -> R20 UI packet
  -> UI／Text Render Pass
```

| Phase | UI処理 |
|---|---|
| `T00` | Screen stack、Document／Style generation、Focus Contextを交換 |
| `T10` | Input Action、pointer／touchをlatch |
| `T30` | 前回publish済みHit Testからtargetを解決し、Focus／Interaction／typed commandを確定 |
| `T70` | GameplayがUiCommand結果を処理し、次ViewModel stateを作る |
| `T90` | ViewModel bind、layout、text、draw／semantic／hit-test snapshot構築 |
| `T110` | immutable UI PresentationをRenderSnapshotへpublish |
| `R20` | clip、batch、glyph／image packetを抽出 |

UI interactionはpixel hit resultからWorldを直接変更せず、registered `UiCommandId`を発行する。`T90`で作ったHit Test Snapshotは次tickの`T10／T30`で使用する。Surface generation、Document generation、layout generation不一致のpointer eventはtargetへ配送しない。

## 4. UiDocument

### 4.1 Document header

```text
UiDocument
  document_id
  schema_version
  root_node_id
  design_extent_lu
  scale_policy
  safe_area_policy
  style_sheet_refs[]
  font_set_refs[]
  localization_namespace
  view_model_schema_id
  nodes[]
```

1 Documentはnode最大65,535、tree depth最大64、一Nodeの直接child最大4096とする。Node IDはDocument内Stable numeric IDで、表示名やarray indexをdispatchに使わない。Parent cycle、missing root、duplicate ID、orphan nodeをCookで拒否する。

### 4.2 `UiNode`

| Field | 規則 |
|---|---|
| `node_id` | Stable numeric ID |
| `widget_type` | registered closed ID |
| `parent_id` | root以外必須 |
| `sibling_order_key` | `uint64` |
| `visibility` | visible、hidden、collapsed |
| `enabled` | bool |
| `layout` | typed Layout Spec |
| `style_classes` | 最大16 |
| `inline_style` | Schemaで許可されたoverrideだけ |
| `bindings` | 最大32 |
| `interactions` | 最大16 |
| `accessibility` | role、name、description、state、order |
| `editor_metadata` | Runtime Cook対象外 |

### 4.3 C1 Widget Catalog

- container
- image
- text
- button
- toggle
- slider
- progress
- scroll view
- virtual list
- text input
- modal layer
- focus scope
- spacer
- world／scene viewport presentation

Widgetごとにrequired field、event、focus、accessible roleをMCDで別typeにする。汎用property bag、Widget C++ class名、arbitrary draw callbackをUiDocumentへ保存しない。

## 5. Coordinate、Scale、Safe Area

`ui_lu`をPlatform非依存のlogical unitとする。`IApplicationSurface`が`logical_extent_lu`、`physical_extent_px`、`pixel_scale`、safe-area insetを提供する。Layoutはlogical unit、Renderは最終physical pixelを使う。

`UiScalePolicy`を次で固定する。

| Policy | 規則 |
|---|---|
| `native_logical` | Surface logical sizeをそのまま使用 |
| `reference_fit` | Reference extentを保ちletterbox |
| `reference_expand` | 短辺scaleを保ち余剰領域を拡張 |
| `reference_match` | width／height match係数`[0,1]` |
| `pixel_locked` | integer scale、Point、Visual Styleが許可時だけ |

Game UIの既定は`reference_expand`、Reference 1920×1080 luとする。Mobileはsafe areaをroot paddingへ適用し、Nodeが明示`allow_unsafe_area=true`を持つ背景以外をcutout／system gesture領域へ置かない。

User UI scaleは0.75～2.00、既定1.00で、Textとhit targetを同じ比率でscaleする。Scale後にrequired contentが入らない場合はclipせず、responsive breakpoint、scroll、layout errorの順に処理する。

## 6. Layout

### 6.1 Layout kind

| Kind | 用途 |
|---|---|
| `stack` | row／column、gap、alignment |
| `flex` | grow／shrink／basis、wrap |
| `grid` | bounded track、row／column placement |
| `overlay` | 同一領域へ重ねる |
| `absolute` | anchor／offset。HUD等へ限定 |
| `leaf` | intrinsic／explicit size |

CSS syntaxやcascadeを採用せず、typed fieldをMCDで持つ。Lengthは`auto | lu | percent | viewport_percent | content`のclosed unionで、calc式、任意unit文字列を禁止する。

### 6.2 Measure／Arrange

Layoutは次の二段階を一つのimmutable inputで実行する。

1. bottom-up `Measure(available)`でminimum／preferred／maximumを計算
2. top-down `Arrange(final_rect)`でNode rect、clip、baselineを確定

同一inputとTarget Profileから同じlogical rectを生成する。floatはfinite、position／sizeは1/64 luへcanonical roundし、negative final sizeを0へclampして成功させずLayout errorにする。

Intrinsic text／image sizeはversioned Asset metricsを使用する。Asset未Ready時に0 sizeでlayoutを確定せず、required AssetはScreen activationを待ち、optional Assetは明示placeholder metricsを使う。

Layout dirty propagationは変更Nodeから必要ancestor／descendantだけへ限定する。Virtual Listはvisible range＋overscan最大2 viewportだけをmaterializeし、全item Widgetを生成しない。

### 6.3 Capacity

| 項目 | C1 hard cap |
|---|---:|
| active Screen | 32 |
| active UI node | 131,072 |
| layout depth | 64 |
| clip depth | 32 |
| draw primitive／frame | 131,072 |
| visible glyph／frame | 65,536 |
| focusable node／scope | 16,384 |
| UI interaction／tick | 8,192 |

上限超過を一部非表示にして成功させず、Screen activation／Frameをtyped `UiCapacityExceeded`で失敗させる。

## 7. Style

`UiStyleSheet`はsemantic tokenとtyped ruleを持つ。

```text
UiStyleSheet
  style_sheet_id
  visual_style_profile_id
  tokens
    color
    typography
    spacing
    radius
    border
    motion
  rules[]
```

Rule selectorはWidget Type、Style Class、Stateの三要素だけとし、descendant selector、path selector、arbitrary predicateを禁止する。解決優先順位はEngine default、Visual Style、Document Style、Class、inline overrideで固定する。

Stateはnormal、hover、pressed、focused、disabled、selected、invalid。Color以外にborder、icon、text、focus indicatorの少なくとも一つで意味を示す。

Style変更はVisualStyleProfile dependency closureとしてPreview／Cookし、Font、UI、Material、Postと部分generationを混在させない。

## 8. ViewModelとBinding

### 8.1 `UiViewModelSchema`

ViewModelはMCDで定義したimmutable snapshotである。Field typeはbool、fixed-width integer、finite float、enum、LocalizedText、Asset reference、bounded list、registered nested structに限定する。

Bindingは次のclosed formを使う。

```text
UiBinding
  source_field_id
  target_property_id
  direction
  converter_id
  format_spec_id
  fallback_value
```

- `direction`は`one_way`、User setting／Text Inputの許可fieldだけ`two_way_proposal`
- converterはEngine Catalogのtyped converter
- arbitrary expression、reflection path string、C++ function、Scriptを禁止
- binding failureをempty string／0へ黙って変換せず、Development diagnosticと明示fallback

Two-wayはViewModel memoryへ直接writeせず、typed `UiCommand`／`SettingChangeProposal`を発行する。

### 8.2 Screen state

Focus、scroll、selection、expanded、text draft等のpresentation stateは`UiStateStore`がScreen instance IDごとに所有する。Persistent Game stateやSaveが必要な値はUiCommandでGameplay／Setting Storeへ移す。Widget object layoutをSaveへserializeしない。

## 9. Event、Hit Test、Focus

### 9.1 Dispatch

UI Eventはcapture、target、bubbleの順でNode depth最大64を走査する。Handlerはregistered Interaction Descriptorで、結果は次に限定する。

- handled／unhandled
- focus request
- pointer capture／release
- typed UiCommand
- local UiState delta
- accessibility announcement

任意function callback、World pointer、async task起動をHandlerにしない。

Pointer captureはDevice ID＋contact／pointer ID＋Node generationを持ち、up、cancel、focus loss、Node removal、surface changeで必ず解除する。

### 9.2 Focus

- Focusable Nodeは明示`focus_policy`を持つ。
- Focus orderは`focus_order`、同値はtree order。
- Focus scopeからkeyboard／controllerで退出でき、trapを作らない。
- Directional navigationは候補rectのdirection、distance、overlap、focus order、Node IDでcanonicalizeする。
- Modalは一つのactive Focus Scopeを作るが、Cancel／Close Actionを必須とする。
- Mouse hoverとkeyboard focusを同一stateにしない。

Screen swap、Node disable／delete時は同Scopeの次候補、parent Scope、root defaultの順にFocusを復旧する。候補がなければNoneを明示する。

## 10. Text storageとInput

### 10.1 Storage

- Source、Localization、ViewModel textはvalid UTF-8。
- Source key、locale tag、Font family tagはNFCへcanonicalizeする。
- User-visible本文を意味なくnormalizeして保存し直さない。
- Offsetのwire表現はUTF-8 byte offset、cursor／selection APIはgrapheme boundary indexを使用する。
- NUL、invalid UTF-8、unpaired surrogate由来入力をrejectする。
- Untrusted diagnostic textだけU+FFFD置換を許可し、counterを記録する。

### 10.2 Text input／IME

`ITextInputService`はcommit text、composition text、selection、candidate／composition range、delete-surrounding、clipboard commandを値としてUIへ渡す。

Text Input Widgetは次を持つ。

- single／multi line
- max UTF-8 bytes、max grapheme
- allowed content policy
- selection／caret／composition
- undo／redo
- password／secure mode
- submit／cancel Action

IME compositionをAction keyboardから再構築しない。Composition中のtextをGameplayへCommitせず、確定eventで一度だけdraftへ反映する。

Password／secret fieldはplaintext log、History、AI Context、clipboard、crash metadata、accessibility valueへ出さない。Process内では必要最短寿命のsecure bufferを使い、focus loss／submit／cancelでzeroizeする。

## 11. Localization

### 11.1 Resource

```text
LocalizationCatalog
  namespace_id
  source_locale
  required_locales[]
  entries[]
    localization_key_id
    developer_description
    argument_schema
    messages_by_locale
```

`LocalizationKeyId`はStable numeric IDで、Source文字列やEnglish本文をkeyにしない。各messageはICU MessageFormat相当のbounded ASTへoffline Cookする。

### 11.2 Message AST

C1 nodeを次に限定する。

- literal
- typed argument
- number
- date／time
- select
- plural／selectordinal
- pound substitution

AST depth最大16、node最大256、argument最大32、formatted UTF-8 output最大64 KiBとする。Arbitrary function、nested code、file／network access、loopを持たず、汎用Script VMではない。

Argument schemaはstring、integer、finite number、date-time instant、duration、enumに限定し、型不一致をplaceholderへ黙って変換しない。

### 11.3 Localeとfallback

Locale tagはBCP 47としてICUでcanonicalizeする。Fallback順を次で固定する。

```text
requested exact locale
-> Project-declared parent locale
-> language-only locale
-> Project source locale
```

同じlocaleを二度訪れず最大8段とする。Required localeのrequired key不足、argument schema不一致、plural branch不足はPackage build errorである。Optional textだけSource locale fallbackを許可し、Localization diagnosticsへ記録する。

ProjectはShipping locale setを明示し、Asset CookerがLocalization、Font coverage、Audio dialogue、ICU dataを同じContent Groupへ閉包する。

### 11.4 C1 conformance locale

Engine fixtureは最低限、`en-US`、`ja-JP`、`zh-Hans`、`zh-Hant`、`ko-KR`、`de-DE`、`fr-FR`、`es-ES`、`pt-BR`、`ru-RU`、`ar`、`he`を検証する。これは全Projectへの同梱強制ではなく、Latin、CJK、Cyrillic、RTL、plural／formatのEngine conformance setである。

Pseudo localeを二つ持つ。

- `qps-ploc`: 30～50% expansion、accent、bracket
- `qps-plocm`: RTL mirror／BiDi stress

## 12. Text layout

### 12.1 Pipeline

1. Localization Message ASTをtyped argumentでformatする。
2. UTF-8、grapheme、paragraph、line-break候補をICUで解析する。
3. Paragraph base directionをexplicit UI propertyまたはUnicode BiDiで決める。
4. Script、direction、Font fallback、style spanでrunへitemizeする。
5. HarfBuzz bufferへscript、language、direction、featuresを明示しshapeする。
6. Glyph advanceとICU line-break候補からlineを決定する。
7. 行ごとにBiDi visual order、alignment、justification／ellipsisを解決する。
8. FreeType metrics／rasterized glyphをversioned atlasへ解決する。
9. Glyph quad、cluster、caret、selection、accessibility text rangeを出力する。

HarfBuzz buffer propertyを推測任せにせず、Engineが解析結果を設定する。同じFont Asset、Library version、locale、text、style、widthから同じlogical glyph sequenceを生成する。

### 12.2 Font fallback

`FontSetAsset`は次を持つ。

```text
font_set_id
primary_font
fallback_fonts[0..7]
script_overrides
emoji_policy
missing_glyph
allowed_features
```

Shippingはbundled OTF／TTF Fontだけを使い、OS installed fontにlayoutを依存させない。全Fontはlicense／embedding、table bounds、glyph coverage、variation axis、hinting policyをImport時に検証する。

Fallbackはgrapheme／shaping clusterを分割しない最小runで行う。Fallback 8件でglyphが見つからない場合はFont Setの明示missing glyphを使い、無表示にしない。Missing glyph countをPackage preflightとRuntime telemetryへ出す。

C1 color fontはCOLR／CPAL v0を検証対象とする。SVG glyph、CBDT／CBLC PNG、COLR v1、system emojiはC2であり、C1ではmonochrome fallbackをPackageに含める。

### 12.3 Font sizeとmetrics

Font sizeは`ui_lu`、1/64単位へcanonicalizeする。Line heightはFont metrics、explicit multiplier `[0.5,4]`、paragraph spacingから計算する。Baseline、ascent、descent、line gapをFontごとに保持し、fallback glyphで行全体がframeごとに揺れないようFont Setのaggregate line metricsをCookする。

## 13. Glyph cacheとRendering

### 13.1 Cache

Glyph keyを次で固定する。

```text
font_asset_version
face_index
glyph_id
size_1_64_lu
variation_coordinates
render_mode
pixel_scale_bucket
```

C1 render modeはgrayscale R8とCOLR／CPAL v0 RGBAだけである。Atlas pageは2048×2048、1 pixel padding、page generation付きとする。

| Profile | R8 page | RGBA page | GPU hard cap |
|---|---:|---:|---:|
| Windows | 12 | 4 | 112 MiB以内 |
| Mobile Baseline | 4 | 2 | 48 MiB以内 |
| Mobile Standard／High | 8 | 3 | 80 MiB以内 |

実際のGPU chargeはRendering texture budgetに含め、別budgetを追加しない。Pageはsubmission serialとUI snapshot lease終了後だけevictする。Frameで使用中glyphを再利用しない。

Screen activation前にrequired glyph setをprewarmする。Runtime missはbounded worker raster requestを作り、Atlas promotionまで明示missing glyphまたはlast valid glyphを使う。T90／render threadでFreeType rasterを同期実行しない。

### 13.2 UI Render packet

UI Rendererは次だけを受け取る。

- rect／transform
- clip rect
- image／glyph Asset version
- color／border／radius
- z／order
- accessibility debug flag

UIはWorld Post Process後、display resolutionでrenderし、dynamic resolution、TAA、World tone mapの影響を受けない。HDR surfaceではStyleのSDR reference whiteから明示変換する。Draw orderはScreen layer、z、tree order、Node ID、primitive indexでcanonicalizeする。

## 14. Accessibility

### 14.1 Semantic tree

各interactive／informative Nodeは次を持つ。

| Field | 規則 |
|---|---|
| `role` | button、text、heading、checkbox、slider、list、listitem、textbox等 |
| `name` | Localization keyまたはsafe dynamic text |
| `description` | optional |
| `value` | roleに応じたtyped value |
| `state` | enabled、checked、selected、expanded、invalid等 |
| `actions` | invoke、toggle、increment、set value、scroll等 |
| `bounds` | final physical screen rect |
| `order` | explicitまたはfocus order |

Semantic treeはrender primitive、pixel、Backend objectから逆算せず、UiDocumentとLayoutから直接作る。Editorは同じ原則を`EditorViewDescriptor`と`EditorSemanticSnapshotV1`へ適用する。

### 14.2 Platform bridge

- Windows: UI Automation provider
- Android: Accessibility node providerをPlatform Java／Kotlin Adapterから投影
- Apple: UIKit／UIAccessibility elementをObjective-C++ Adapterから投影

Platform bridgeはSemantic Node IDとgenerationを使い、UI Runtime pointerを長期保持しない。Assistive actionもtyped UiCommandへ変換する。

### 14.3 Game UI要件

- keyboard／controller／touch／screen readerでCore menuを操作可能
- focus visible、focus not obscured、keyboard trapなし
- primary touch target最低44×44 lu、desktop最低32×32 lu
- text 200% UI scale、High Contrast、color以外の状態表現
- captions、subtitle size／background／speaker option
- reduced motion、flash／camera shake settingとのStyle連携
- timeoutを必要とするUIは延長／停止policyを明示

## 15. Memory、Thread、Performance

### 15.1 Charge

Windowsでは次を既存Parent budget内のchild capとする。

| UI allocation | hard cap／charge |
|---|---|
| active UI tree／state／layout／text cache | 32 MiB、Core World／SaveのECS／World 128 MiB child |
| UI transient measure／packet | 8 MiB、Frame／Job transient |
| Localization catalog／ICU filtered runtime data | 24 MiB、Asset streamingのhot cache 192 MiB child。formatted text cacheは上段32 MiBに含む |
| Glyph atlas | 13.1節、GPU texture budget |

Runtime message queue予約後のECS／World残量111 MiBからactive UI 32 MiBを差し引き、79 MiBをEntity／Component状態へ残す。Frame 32 MiBからInput latch 4 MiBとUI transient 8 MiBを差し引き、20 MiBを他のFrame transientへ残す。これはUnassigned headroomを再配分しない。Mobile Baselineはactive UI／Text CPU 16 MiB、Standard 24 MiB、High 32 MiBをaggregate Engine CPU cap内へchargeし、ICU dataとglyph atlasをContent／GPU budgetへ個別記録する。

### 15.2 Thread

- UI State／Focus writerはSimulation thread。
- LayoutはT90 ownerだが、immutable screen／textのpre-layoutとshaping／glyph rasterをWorkerへ出せる。
- Worker resultはDocument、Style、Font、locale、ViewModel、surface generationをT20／T90統合時に再検査する。
- ICU `BreakIterator`等のstateful objectをthread共有せず、thread／job-local instanceまたはcloneを使う。
- HarfBuzz buffer、FreeType face access、glyph output bufferのownershipをjobごとに分離する。
- T90でfile I/O、Font parse、glyph raster、locale data loadを行わない。

### 15.3 Budget

- `T30` UI interaction P95 0.15 ms以下
- `T90` UI bind／layout／packet P95 0.40 ms以下
- UI／Composite GPU P95 0.50 ms以下
- cached text shaping P95 0.10 ms／1000 grapheme
- Screen activation P95 100 ms、required Asset loadを除く
- steady-state UI Runtime allocation 0／frame
- 10分soakでglyph page、layout cache、semantic nodeが上限内

Runtime規約のPostPhysics＋Presentation 0.75 ms、full frame 14 msを緩和しない。

## 16. AI／Editor Authoring

AIと人間はUiDocument、Style token、Localization entry、Font Set、ViewModel schema、Interaction、responsive breakpointをtyped ChangeSetで編集する。

AIは次を直接生成しない。

- draw list／glyph position／atlas coordinate
- HarfBuzz／FreeType／ICU object
- Platform accessibility object
- arbitrary binding expression／C++ callback
- system Font path
- Runtime downloaded Font

UI DesignerはCanvas、Hierarchy、Inspector、Layout／safe-area overlay、Focus graph、Localization preview、pseudo locale、Text／glyph diagnostic、Accessibility treeを表示する。同じUIをmouse、keyboard、controller、touch previewで試験できる。

AIがUIを生成する場合、GameSpecの主要flow、Target、safe area、required Action、locale expansion、accessibility、Style lockを先に検証し、見た目だけのScreenshot合格にしない。

## 17. Failure policy

| Failure | 結果 |
|---|---|
| Invalid UiDocument／layout cycle | Cook／Screen activation拒否 |
| Missing required Widget Asset／Font／Localization | PackageまたはScreen activation失敗 |
| Missing glyph | 明示missing glyph＋diagnostic。無表示にしない |
| Binding type mismatch | ChangeSet／Cook reject。Runtimeは明示fallback |
| UI capacity／transient cap | Frameを部分描画せずUI fault |
| Stale hit-test／pointer capture | event破棄／capture cancel |
| IME／Text Adapter unavailable | Text Input disabled、Actionから文字推測しない |
| Glyph worker stale／deadline | result破棄、last valid／missing glyph |
| Accessibility bridge stale | action reject、最新Semantic treeをpublish |
| ICU／shaping non-finite／invalid output | text run reject、Screen diagnostic |
| Password field error | content破棄／zeroize、log禁止 |

Required system menu UIがfaultした場合はGameを操作不能のまま続行せず、safe fallback screenをEngine-owned fixed Assetから表示する。Fallback screenもbundled Font、Exit／Report action、Accessibility semanticsを持つ。

## 18. Dependency Build規約

| Dependency | exact baseline | Build範囲 |
|---|---|---|
| HarfBuzz | 14.2.1／`77a832110d40b0179636f5be8f8781f8299d7e50` | FreeType＋ICU integration。GLib、Cairo、Graphite2、docs／toolsをShipping無効 |
| FreeType | 2.14.1／`3bd82b5f543bc84ccf2b1d0cdb63b95218099ee6` | TrueType／OpenType、CFF／CFF2、SFNT。BZip2、Brotli／WOFF2、PNG、SVG optional moduleをC1無効 |
| ICU4C | 78.3／`21d1eb0f306e1141c10931e914dfc038c06121da` | `common`＋`i18n`とfiltered data。samples／tests／extras、legacy conversionをShipping無効 |

Source archive hash、patch hash、compiler option、license file hashを`toolchain.lock.json`、vcpkg overlay、SBOMへ固定する。HarfBuzz／FreeType／ICU更新は全conformance locale、golden layout、Font raster、package size、memory、performanceを再検証する。

Editor toolにはfull ICU dataを同梱できるが、Shipping GameはProject locale set、fallback、number／date／plural、boundary ruleに必要なdataをICU Data Filterで決定論的に絞る。Filter inputとoutput hashをPackage Receiptへ記録する。

## 19. TestとDefinition of Done

- UiDocument node／depth／cycle／capacity、全Layout kind、responsive／safe area
- Focus、directional navigation、modal、pointer capture、Widget event、stale hit test
- ViewModel type、one-way／two-way proposal、converter、fallback、no arbitrary expression
- UTF-8、grapheme、combining mark、ZWJ、variation selector、CJK、Arabic／Hebrew BiDi
- ICU plural／number／date、locale fallback、required key、pseudo localization
- HarfBuzz shapingとFreeType glyph metrics／raster golden
- Font fallback cluster、missing glyph、COLR／CPAL v0、license／coverage
- IME composition、selection、clipboard、password zeroization／AI exclusion
- 100／125／150／200% DPI、0.75～2.0 UI scale、portrait／landscape、cutout
- Windows UIA、Android Accessibility、Apple UIAccessibility action
- keyboard／controller／touch／screen readerでTitle→Settings→Play→Pause→Exit
- glyph atlas eviction／submission lifetime、locale／Font／Style hot reload
- 131,072 Node、65,536 glyph、8,192 event上限と10分soak
- Windows、Android、Appleの同じUI fixtureでlogical layout／semantic hash一致

C1完了条件は、2D／3D縦切りのTitle、Settings、HUD、Pause、Result、Text Inputを全Target、conformance locale、input方式、accessibility bridgeで操作でき、AI生成と手動編集が同じUiDocument／ChangeSet／Validatorを通り、CPU／GPU／memory hard gateを満たすことである。

## 20. 一次資料

- [HarfBuzz 14.2.1](https://github.com/harfbuzz/harfbuzz/releases/tag/14.2.1)
- [HarfBuzz Shaping](https://harfbuzz.github.io/shaping-opentype-features.html)
- [FreeType 2 Documentation](https://freetype.org/freetype2/docs/)
- [FreeType Glyph Management](https://freetype.org/freetype2/docs/reference/ft2-glyph_management.html)
- [ICU User Guide](https://unicode-org.github.io/icu/userguide/)
- [ICU Boundary Analysis](https://unicode-org.github.io/icu/userguide/boundaryanalysis/)
- [Unicode Bidirectional Algorithm UAX #9](https://www.unicode.org/reports/tr9/)
- [Unicode Line Breaking UAX #14](https://www.unicode.org/reports/tr14/)
- [Unicode Text Segmentation UAX #29](https://www.unicode.org/reports/tr29/)
- [Unicode Locale Data Markup Language UTS #35](https://www.unicode.org/reports/tr35/)
- [Microsoft IME requirements](https://learn.microsoft.com/en-us/windows/apps/develop/input/input-method-editor-requirements)
- [Microsoft UI Automation Providers](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-providersoverview)
- [Android Accessibility](https://developer.android.com/guide/topics/ui/accessibility)
- [Apple Accessibility](https://developer.apple.com/accessibility/)

外部LibraryとUnicode規格は文字処理の事実とalgorithmを提供する。Miraikanai固有のUI model、Widget、Layout、Binding、Event、budget、AI／manual workflowは本書が所有する。
