# Miraikanai Engine UI／Text／Localization／Accessibility Contract

- 文書ID: mirakan.arch.platform-ui-text-localization-accessibility
- 状態: review
- 正本範囲: Game UI document／widget／layout／style／binding／event／focus、Text storage／input／layout、Localization、glyph cache、Accessibility、UI authoring、UI固有capacity／failure／qualification
- 非正本範囲: Project ChangeSet／Asset lifecycle、common Renderer execution、Runtime phase／shared budget、Editor shell、Platform lifecycle／safe-area source、external library version・hash・license・URL、AI authorization／Evidence envelope。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Editor UI Framework](../03-authoring/editor-ui-framework.md)、[Editor workspace／UX](../03-authoring/editor-workspace-ux.md)、[Native game module](../03-authoring/native-game-module.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Render Graph](../06-rendering/render-graph.md)、[Windows](windows.md)、[Mobile Common](mobile-common.md)、[Android](android.md)、[Apple](apple.md)、[Input](input.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論

Miraikanai EngineのGame UIは、HTML／CSS／JavaScript、webview、汎用Script Widget、immediate-modeの描画命令列を正規形式にしない。Schema検証可能な`UiDocument`、typed `UiViewModel`、C++ Layout／Event Runtime、versioned Style／Font Assetから構築する独自retain-mode UIである。

TextはUTF-8を正規storageとし、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)が固定する次のLibraryを限定利用する。

- HarfBuzz: OpenType shaping
- FreeType: bundled Font検証、glyph index／metrics／rasterization
- ICU4C: BCP 47 locale、BiDi、grapheme／word／line boundary、plural／number／date／message formatting

Library型、Font object、ICU iterator、glyph atlas pointerをProject、Save、AI、NativeGameModuleへ公開しない。UI Document、Layout、Focus、Event、Binding、Localization schema、Accessibility semantic tree、memory／failureはMiraikanaiが所有する。

Windows Editor shellは本書の`UiRuntimeTree`、Layout、Event、Semantic contractを共有する独自`MirakanUi Core`上に構築する。Editor固有のDocking、Window、DirectWrite、TSF、UI Automation、D3D12 UI passはEditor UI Framework規約を正本とする。Shipping Game UIは全Targetで本書の共通Text Layoutを使い、bundled Fontを正本にする。

## 2. 決定権と対象外

| 主題 | 正本 |
|---|---|
| UiDocument、Widget、Layout、Binding、Focus、Event、Text、Localization、Accessibility | 本書 |
| UI source Commit、ChangeSet、Undo、Projection | Authoring規約 |
| Input Action、pointer／touch、Device、Remap | Input規約 |
| UI Render pass、GPU resource、composition | Rendering規約 |
| UTF／locale／Font Library version、Build、license | 基盤規約 |
| Android／Apple lifecycle、safe area、Text Adapter | Mobile規約 |
| Composite／Effect／Native WidgetのUI契約、AI UI生成workflow | 本書 |
| Project C++ artifact、C ABI、Source Worker、Build、Promotion | Native Game規約、AI実装・保守ガバナンス規約 |

C1ではHTML／CSS parser、DOM、JavaScript、webview、arbitrary expression binding、Runtime Font download、system Font依存のShipping layout、Rich Text markup parser、SVG Font、font editorを実装しない。C1は標準Widgetと宣言型`UiCompositeDefinition`をProduction対象にする。C2候補はlimited rich text span、MSDF、HRTF字幕連携、advanced vector UI、型付き`UiEffectGraph`、Governance decision refを持つProject C++による`UiNativeWidget`である。第三者binary Widget、Marketplace、Editor ProcessへProject Widget codeをloadする経路はC1／C2に含めず、C3で別Threat Model、ABI、署名、配布、revocation設計がOwner文書に追加されるまで使用しない。

## 3. ArchitectureとRuntime mapping

次のslot名と順序は[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)への参照であり、本書はphaseを再定義しない。本書が所有するのは各slotへ提出するUI input／output、generation、failureだけである。

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

1 Documentはnode最大65,535、tree depth最大64、一Nodeの直接child最大4096とする。Source Node IDは基盤規約どおりUUIDv7 `StableId`で、表示名やarray indexをidentityに使わない。Parent cycle、missing root、duplicate ID、orphan nodeをCookで拒否する。

### 4.2 `UiNode`

| Field | 規則 |
|---|---|
| `node_id` | UUIDv7 `StableId` |
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

Cookerは一つのexact `UiDocument` Artifact内でNode `StableId`をUUID byte順に並べ、1から`UiNodeRuntimeId uint32`を割り当てる。0はinvalidとし、Artifactへ対応表を含める。Runtime layout／dispatchは`UiNodeRuntimeId`、Source／Save／AI／EditorはNode `StableId`＋Document revisionを使い、別Document Artifactの同じ数値を比較しない。

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

### 4.4 Widget拡張model

Game UIの自由度は制約解除ではなく、Schemaへ登録する四層の拡張modelで提供する。

| Tier | Source | 適用Phase | 許可範囲 |
|---|---|---|---|
| 0 `Builtin Widget` | Engine MCD | C1 | 4.3節の標準Widget、標準Layout、標準Interaction |
| 1 `UiCompositeDefinition` | Project Authoring Document | C1 | 標準／登録済みWidgetの構成、named slot、型付きparameter、Binding、Style、Interaction |
| 2 `UiEffectGraph` | Project Authoring Document | C2 | 閉じたEffect node catalogによるmask、gradient、SDF、noise、distortion、transition、bounded geometry |
| 3 `UiNativeWidget` | Governance decision ref付きNativeGameModule | C2 | 宣言型表現で不足するProject固有のmeasure、presentation、interaction algorithm |

`UiCompositeDefinition`は次を持つ。

```text
UiCompositeDefinition
  composite_type_id
  schema_version
  root_template
  named_slots[]
  typed_parameters[]
  binding_contract[]
  interaction_contract[]
  semantic_template
  required_capabilities[]
  fallback_widget_type_id
```

Compositeは展開後も通常のNode、depth、binding、interaction、memory capへchargeする。再帰的Composite cycle、slot多重所有、型不一致、fallback cycleをCookで拒否する。AIとUI DesignerはCompositeを一Widgetとして配置できる一方、Diff、Accessibility、cost previewでは展開結果まで表示できる。

`UiEffectGraph`はEngine登録済みnode、finite parameter、静的loop上限、texture sample上限、primitive上限、Target Capability、reduced-motion fallbackを必須とする。Graphはoffline compileし、Target別qualified UI pipelineとparameter blockへ変換する。任意HLSL、raw GPU resource、command list、shader include、Runtime compileを許可しない。

`UiNativeWidgetManifestV1`は最低限、次を宣言する。

```text
widget_type_id
schema_version
native_capability_id
typed_properties[]
named_slots[]
input_fields[]
output_commands[]
semantic_contract
measure_budget_us
presentation_budget_us
scratch_bytes
primitive_cap
supported_targets[]
fallback_widget_type_id
```

Native Widgetは[Native game module](../03-authoring/native-game-module.md)のC ABI、Source Worker、Review、Promotionを通ったProject codeだけを使用する。UI Runtimeはvalue型property、Asset handle、bounded scratch、whitelist済みprimitive encoder、qualified Effect IDだけを渡す。Native codeは`UiRuntimeTree` pointer、Widget object、World pointer、GPU address、Renderer command、filesystem、network、OS APIを取得しない。Interaction結果はregistered `UiCommandId`、presentation結果はbounded primitiveだけとし、失敗時の部分outputを破棄する。

Native Widget codeをEditor Processへloadしない。UI DesignerはManifestとfallbackによる静的projectionを表示し、実code Previewは別ProcessのGameHostで実行する。Windows PreviewはGameHost再起動、ShippingとMobileはclean static linkを使用する。Manifest、binary、Contract lock、Target Profileのhash不一致はload前に拒否する。

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

Source `LocalizationKeyId`はUUIDv7 `StableId`で、Source文字列やEnglish本文をkeyにしない。Cookerは一つのexact Localization Catalog Artifact内でKey `StableId`をUUID byte順に並べ、1から`LocalizationKeyRuntimeId uint32`を割り当てる。0はinvalidとし、Runtime IDをSource、Save、別Catalog比較へ使用しない。各messageはICU MessageFormat相当のbounded ASTへoffline Cookする。

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

Font Source Importは[Asset lifecycle](../03-authoring/asset-lifecycle.md)の`FontImportSettingsV1`、`FontImportIRV1`、coverage Preview、Conversion Report、Reimport Conflictを使用する。Toolchain lockのOpenType baselineに対してtable directory、offset／length、checksum、cmap、GSUB／GPOS／variation、composite glyph boundsを検証し、required locale／script coverage不足をProduction package blockにする。Font名や埋込みmetadataだけで利用権を確定せず、Asset License Recordを必須とする。

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

次をUI domainのchild capとし、[Runtime performance／capacity](../04-runtime/performance-capacity.md)のTarget別parent scopeへchargeする。

| UI allocation | hard cap／charge |
|---|---|
| active UI tree／state／layout／text cache | 32 MiB |
| UI transient measure／packet | 8 MiB、Frame／Job transient |
| Localization catalog／ICU filtered runtime data | 24 MiB。formatted text cacheは上段32 MiBに含む |
| Glyph atlas | 13.1節、GPU texture budget |

Parent総量、他Domainの残量、loan／backpressureはRuntime ownerだけが決定する。Mobile Baselineはactive UI／Text CPU 16 MiB、Standard 24 MiB、High 32 MiBを[Mobile Common](mobile-common.md)のaggregate cap内へchargeし、ICU dataとglyph atlasをContent／GPU budgetへ個別記録する。

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

本書のdomain ceilingはRuntime ownerの共通frame envelopeを緩和しない。

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

### 16.1 AI UI生成workflow

内蔵AI、外部MCP Host、手動UI Designerは次の正規workflowへ収束する。

```text
Natural-language intent／manual edit
  -> AuthoringContextPack
  -> UiAuthoringIntentV1
  -> UiDocument／Composite／Style／Binding proposal
  -> AssetRequirement／GeneratedAssetStaging
  -> ProjectChangeSet
  -> schema／semantic／capability／budget validation
  -> responsive／locale／input／accessibility preview matrix
  -> AI Security／Approval ownerのauthorization decision
  -> AuthoringCommandGateway commit
```

`UiAuthoringIntentV1`は対象Screen／HUD、主要flow、情報優先順位、Target、orientation、safe area、入力方式、locale set、UI scale、Accessibility、Visual Style lock、Asset budget、完了条件を持つ。Blocking不足は質問し、High Impactの見た目または情報構造は最大三候補の低cost Previewを提示する。既存のComposite、Style token、Localization、Assetを検索してから新規生成し、同じ意味のWidgetや画像を無制限に複製しない。

AIの成果物は説明文やScreenshotではなく、Stable IDを対象にしたtyped Operationである。新IDはGatewayへ`Create*` Operationで要求し、存在しないWidget、Asset、ViewModel field、Action IDを推測しない。AIは`UiDocument`全置換を既定にせず、目的に必要なNode、Binding、Style、Asset参照だけを変更する。

### 16.2 画像・生成Assetの設定

C1は外部で生成済みの画像を通常Assetと同じImport、license、provenance、safety、quality Gateへ通し、合格したAsset IDだけを`image`、Composite、Styleへ参照できる。C2のAsset generation providerもoutputを`GeneratedAssetStaging`へ置き、Projectへ直接writeしない。

AIは画像生成とUI設定を一つの不可分な成功として扱わず、次を別artifactとして提示する。

1. `AssetRequirement`と候補、採用理由、style／semantic role
2. Source、provenance、license、safety receipt
3. Target別Import／Cook／memory結果
4. UiDocument／StyleからAsset IDを参照するChangeSet
5. 画像なし、last-valid、またはqualified placeholderのfallback

Text、価格、法的同意、認証情報、操作説明を画像へ焼き込まない。AI生成画像の見た目が合格しても、safe area、contrast、locale expansion、focus、semantic tree、Target cookを省略しない。

### 16.3 Preview、authorization参照、競合

UI Preview matrixは最低限、Projectのminimum／reference解像度、portrait／landscape、safe area、0.75／1.00／2.00 UI scale、全required locale、keyboard／controller／touch、High Contrast、reduced motionを含む。Screenshot／vision評価は視覚差の補助oracleであり、layout rect、binding type、semantic hash、focus graph、budgetの正規判定を置き換えない。

AI proposal作成中にProject revisionが変わった場合はstaleとしてCommitを禁止し、現在revisionへrebaseして全Validatorを再実行する。Navigation root、required Action、purchase／consent／credential UI、Accessibility semantics、Style lock、Production Asset採用、UiNativeWidget sourceのauthorization classとdelegation可否は[AI Security／Approval](../01-governance/ai-security-approval.md)だけが決定する。本書は判断入力となるUI impactを型付きで提示する。

### 16.4 拡張WidgetをAIが扱う条件

AIはTier 1～3を同じWidget名だけで扱わず、Manifestからproperty、slot、Binding、Interaction、semantic、budget、Target、fallbackを取得する。表現方法は`Builtin／Composite -> Effect Graph -> Native Widget`の順に選び、下位Tierで完了条件を満たせる場合に上位Tierを生成しない。

Tier 3を選ぶ場合、AIは理由、宣言型で不足するCapability、公開contract、Source Diff、test、performance予測、Target差、fallbackを`NativeCodeChangeSet`へ含める。Build成功だけでUIへ登録せず、[Native game module](../03-authoring/native-game-module.md)のPromotion Receiptと`RegisterNativeModuleRevision` Commit後に初めて使用可能にする。

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
| Composite cycle／slot／property不正 | ChangeSet／Cook reject、展開しない |
| Effect Graph compile／budget失敗 | generationをReadyにせずlast validまたは明示fallback |
| Native Widget manifest／binary／Target不一致 | load前reject、fallbackまたはScreen activation失敗 |
| Native Widget fault／timeout／primitive超過 | 当該output全破棄、DevelopmentはGameHost停止、Shippingはsafe fallback |
| AI UI proposal stale／invalid | Project revision不変、再base後に全Validator再実行 |

Required system menu UIがfaultした場合はGameを操作不能のまま続行せず、safe fallback screenをEngine-owned fixed Assetから表示する。Fallback screenもbundled Font、Exit／Report action、Accessibility semanticsを持つ。

## 18. Dependency Build境界

HarfBuzzはFreeType／ICU integrationを有効にし、不要なoptional integrationとtoolをShippingから除く。FreeTypeは本書が使用するTrueType／OpenType、CFF／CFF2、SFNT範囲だけを、ICU4Cはcommon／i18nとfiltered dataだけをShippingへ含める。exact version、commit、hash、license、取得元、Build option lockは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)だけが所有する。更新時は全conformance locale、golden layout、Font raster、package size、memory、performanceを再検証する。

Editor toolにはfull ICU dataを同梱できるが、Shipping GameはProject locale set、fallback、number／date／plural、boundary ruleに必要なdataをICU Data Filterで決定論的に絞る。Filter inputとoutput hashをPackage Receiptへ記録する。

## 19. TestとDefinition of Done

- UiDocument node／depth／cycle／capacity、全Layout kind、responsive／safe area
- Focus、directional navigation、modal、pointer capture、Widget event、stale hit test
- ViewModel type、one-way／two-way proposal、converter、fallback、no arbitrary expression
- UTF-8、grapheme、combining mark、ZWJ、variation selector、CJK、Arabic／Hebrew BiDi
- ICU plural／number／date、locale fallback、required key、pseudo localization
- HarfBuzz shapingとFreeType glyph metrics／raster golden
- OpenType table bounds／checksum、Font fallback cluster、missing glyph、COLR／CPAL v0、variation、license／required locale coverage
- IME composition、selection、clipboard、password zeroization／AI exclusion
- 100／125／150／200% DPI、0.75～2.0 UI scale、portrait／landscape、cutout
- Windows UIA、Android Accessibility、Apple UIAccessibility action
- keyboard／controller／touch／screen readerでTitle→Settings→Play→Pause→Exit
- glyph atlas eviction／submission lifetime、locale／Font／Style hot reload
- 131,072 Node、65,536 glyph、8,192 event上限と10分soak
- Windows、Android、Appleの同じUI fixtureでlogical layout／semantic hash一致
- Compositeのcycle／slot／parameter／fallback、展開前後Diff、AI／手動往復
- Effect Graphのnode／sample／loop／primitive cap、Target compile、reduced-motion fallback
- Native WidgetのABI／manifest／measure／presentation／semantic／fault、GameHost隔離、Target fallback
- Prompt→UiDocument／AssetRequirement→ChangeSet→Preview matrix→Governance decision ref→Commit→Undo／Redo
- AI生成画像のprovenance／license／safety／Target cookと、missing／reject時fallback

C1完了条件は、2D／3D縦切りのTitle、Settings、HUD、Pause、Result、Text Inputを標準Widgetと`UiCompositeDefinition`で構築し、全Target、conformance locale、input方式、accessibility bridgeで操作でき、AI生成と手動編集が同じUiDocument／ChangeSet／Validatorを通り、CPU／GPU／memory hard gateを満たすことである。Tier 2／3はC2 Gateであり、C1完了を偽装するfallbackに使わない。

## 20. 外部依存境界

Text／Font library、Unicode specification、Platform IME／Accessibility APIのexact release、取得元、integrity、license、一次根拠は[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)だけが所有する。Miraikanai固有のUI model、Widget、Layout、Binding、Event、domain budget、AI／manual workflowは本書が所有する。
