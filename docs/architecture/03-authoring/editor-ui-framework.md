# Miraikanai Engine 独自Editor UI Framework／Shellアーキテクチャ規約

- 文書ID: mirakan.arch.editor-ui-framework
- 状態: review
- 正本範囲: MirakanUi Core、Editor Shell、Widget／Layout／Style、Event／Focus／Command、UI Rendering、Window／Dock、Text／IME、Semantic Tree、Accessibility bridge、UI ownershipと検証
- 非正本範囲: Project transaction、Workspace journey、Asset operation、Gameplay model、外部Tool・SDK・Libraryの固定値、Runtime／Rendering／Platform内部。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan](../00-product/product-plan.md)、[Core architecture](../02-foundation/core-architecture.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[C++23 modules](../02-foundation/cpp23-modules.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[Project state](project-state.md)、[Editor Workspace UX](editor-workspace-ux.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論

Miraikanai EngineはDear ImGui、Qt、WinUI、WPF、Windows Forms、GTK、wxWidgets、Electron、Chromium Embedded FrameworkをEditor UI Frameworkとして採用しない。C++23で独自の`MirakanUi Core`と`MirakanEditor Shell`を構築する。

`MirakanUi Core`はRetained Modeの`UiRuntimeTree`、typed Layout、Style、Event、Focus、Hit Test、Semantic Tree、Render Packetを所有する。Scene View、Graph、Timeline、Profiler等の高頻度可視化は、Retained Widgetである`UiCanvasSurface`の内部に限り、Engine登録済みのtyped Immediate Canvas producerを使用できる。Immediate描画命令列はUI状態、入力状態、Accessibility、Workspace、Projectの正本にならない。

EditorとShipping Game UIは`MirakanUi Core`を共有するが、FrontendとPlatform Adapterを分離する。

- Editor: C++ `EditorViewDescriptor`、Editor command、Docking、Workspace、Windows system font、DirectWrite、TSF、UI Automation
- Game: Schema検証済み`UiDocument`、typed ViewModel、bundled Font、HarfBuzz／FreeType／ICU、Target別Accessibility bridge

独自UIとは、OS、Graphics API、Unicode algorithm、Font rasterizerまで再実装することではない。Miraikanaiが所有するのはWidget model、Layout、Style、Event、Command、Docking、Workspace、Semantic Tree、Rendering policy、AI操作契約、検証、寿命、性能である。Win32、DXGI／D3D12、DirectWrite、TSF、UI Automation、OLEはPlatform Adapter内の低水準serviceとして利用する。

## 2. 決定権、用語、対象外

### 2.1 決定権

| 主題 | 正本 |
|---|---|
| MirakanUi C++ target、Editor UI architecture、Docking engine、Editor Semantic Interface、禁止GUI dependency | 本書 |
| Game `UiDocument`、Layout algorithm、Style、Binding、Text／Localization、Game Accessibility | UI規約 |
| Panel配置、Workspace、操作、AI Partner、人間工学、Editor性能の製品要件 | Editor UX規約 |
| Project state変更、ChangeSet、Commit、Undo、Recovery | Authoring規約 |
| UI Render Pass、GPU resource、D3D12 device／submission | Rendering規約 |
| Win32 HWND、DPI、Process、Package、OS service | Windows規約 |
| C++ target、Named Module、Memory／Pointer、Dependency lock | 基盤規約とC++規約 |

同じ主題で表現が異なる場合は上表のOwnerを優先する。本書はGame UIのSchemaやEditorの画面構成を重複定義しない。

### 2.2 用語

| 用語 | 定義 |
|---|---|
| `MirakanUi Core` | Editor／Gameで共有するRetained UI Runtime |
| `MirakanEditor Shell` | MirakanUi上に構築するEditor専用Window、Panel、Dock、Menu、Command、Workspace層 |
| `UiRuntimeTree` | Widgetの構造、local presentation state、computed style参照を保持する非永続Retained tree |
| `UiPresentationSnapshot` | Layout、Hit Test、Semantic、Draw Packetのimmutableな一Frame snapshot |
| `EditorViewDescriptor` | Editor C++ viewをtypedに構成する入力。Project正本ではない |
| `EditorSemanticSnapshotV1` | Accessibility、AI context、automation testが読む意味情報のimmutable snapshot |
| `UiCanvasSurface` | Engine登録済みproducerだけがbounded primitiveを出力できるRetained Widget |
| Platform Adapter | Win32／DirectWrite／TSF／UIA／OLE型をMirakanUi型へ変換するprivate実装 |

### 2.3 C1対象外

C1では次を実装しない。

- Editor用HTML／CSS／JavaScript、DOM、WebView
- Public Editor extension SDK、Marketplace、任意binary Widget plugin
- Mobile／Web／VR Editor
- Editor UIのProjectごとの任意C++ callback
- RuntimeからのEditor layout変更
- UIを画像認識またはscreen coordinate clickで操作する内蔵AI
- SVG／CSS parser、browser互換Layout、OS native controlの外観複製
- custom title bar。C1はOS non-client frameを使用する

## 3. 調査結果と方式選択

### 3.1 公式資料から確認した事実

| Engine | 公式に確認できるEditor UI方式 | Miraikanaiへの教訓 |
|---|---|---|
| Unity | Editorの複雑なtoolにはRetained ModeのUI Toolkitを推奨し、IMGUIを代替として残す。Visual Tree、Flexbox、UXML、USS、Data Binding、専用Rendererを持つ | 大規模Editorは永続Widget tree、Layout、debugger、data bindingを必要とする。ただしUXML／USS／C# APIは採用しない |
| Unreal Engine | Unreal EditorはEpic独自のC++ UI FrameworkであるSlate上に構築され、宣言的Widget、Docking、Widget Reflectorを持つ | 独自C++ FrameworkとDocking debuggerは成立する。ただしSlate macro、Widget API、layout modelは採用しない |
| Godot | EditorはC++で記述され、Godot自身のRendererとControl UI systemで描画し、GTK／Qtへ依存しない | Engine UIをEditorで実証する共有Coreは有効。ただしNode／SceneをEditor Widget identityへ流用しない |
| O3DE | Editor tool UIはQtとAzQtComponentsを利用する | Third-party toolkit方式は開発速度を得られるが、本Projectの独自UI要件には合わない |

Unityの`IMGUI`はUnity固有APIでありDear ImGuiではない。上表は方式比較の根拠であり、各Engineの内部API、画面配置、名称、Asset、serializationをMiraikanaiへ移植する根拠ではない。

### 3.2 検討した三方式

| 方式 | 利点 | 欠点 | 判定 |
|---|---|---|---|
| A. Qt等の完成GUI toolkit | Control、IME、Accessibility、Dockingを早く得られる | 外部Widget modelとlifecycleが製品境界になり、AI Semantic契約とGPU統合の自由度が下がる | 不採用 |
| B. Immediate ModeをEditor全体の正本にする | 初期toolとdebug UIを短期間で作れる | Text editor、virtualized tree、Dock persistence、Accessibility、Automation、animation、複雑な状態管理で別のRetained modelが必要になる | 不採用 |
| C. 独自Retained Core＋限定Immediate Canvas | UI意味、Command、Accessibility、AI、Renderを一つのtyped modelから生成できる | 初期実装量と検証量が最大 | 正式採用 |

採用理由は「他Engineが採用しているから」ではない。Miraikanai固有のAuthoring Model、AI安全境界、Project C++生成、Receipt、Accessibilityを同じ意味木へ結合するためである。

## 4. 独自性と外部利用の境界

### 4.1 Miraikanaiが独自所有するもの

- Widget catalogと型付きproperty
- Retained tree、state store、dirty propagation、generation
- Measure／Arrange、Dock layout、virtualization
- Style token、selector、theme、density、High Contrast
- Event routing、Focus、pointer capture、shortcut、Command mapping
- Draw primitive、batch policy、clip、atlas、D3D12 UI pass integration
- Panel、Workspace、multi-window、recovery
- Semantic Tree、UIA mapping、AI context projection
- Memory budget、thread ownership、failure、telemetry、test

### 4.2 Platform serviceとして利用するもの

| Service | Engine-owned Port | 利用範囲 | 公開禁止型 |
|---|---|---|---|
| Win32 | `IApplicationSurface`／`IUiWindowService` | top-level HWND、message、cursor、monitor、work area、non-client frame | `HWND`、`WPARAM`、`LPARAM` |
| DXGI／D3D12 | `IUiRenderBackend` | Swap Chain、texture、command submission | COM interface、descriptor handle |
| DirectWrite | `IUiTextBackend` | Editor text shaping／layout、system Font fallback、glyph analysis | `IDWrite*` |
| TSF | `ITextInputService` | IME composition、selection、candidate、input scope | `ITf*`、`ITextStoreACP*` |
| UI Automation | `IUiAccessibilityBridge` | custom control provider、control pattern、event | `IRawElementProvider*` |
| OLE Data Transfer | `IEditorDataTransferService` | Explorerとのfile drag/drop、rich clipboard transfer | `IDataObject`、`IDropTarget` |
| OS dialog | `IPlatformDialogService` | File／Folder picker、credential／permission、fatal recovery | Platform dialog object |

OS dialogはbootstrap、File／Folder selection、credential／permission、fatal recoveryに限定する。Inspector、Outliner、Asset Browser、Property editor、Editor menu、通常confirmationにはOS native controlを使わない。

## 5. Architectureとデータフロー

```text
Human pointer／keyboard／IME
  -> WindowsUiAdapter
  -> PlatformUiEventV1
  -> MirakanUi Event／Focus Router
  -> EditorCommandRequest
  -> EditorCommandRegistry
  -> AuthoringCommandGateway／WorkspaceService

AI Provider／MCP／CLI
  -> TaskAuthorizationEnvelope
  -> typed Editor command／Project change primitive
  -> AuthoringCommandGateway

ProjectRevision + EditorUserState
  -> Editor Read Projection
  -> EditorViewDescriptor
  -> UiRuntimeTree
  -> Measure／Arrange／HitTest／Semantic／Paint
  -> UiPresentationSnapshot
       ├─ MirakanUiDrawPacketV1 -> Rendering UI Pass -> D3D12
       ├─ EditorSemanticSnapshotV1 -> Windows UIA Provider
       └─ EditorContextSnapshotV1 -> bounded AI context
```

規則:

- Human、assistive technology、AIは最終的に同じ`EditorCommandId`へ収束し、Project Source変更は同じtyped `ProjectChangePrimitiveV1`を完全登録済み外側MCD Operationのnamed inputへ載せる。
- AIは`PlatformUiEventV1`、screen coordinate、Widget pointer、HWND、UIA providerを操作経路にしない。
- UIA client actionはSemantic Nodeに対応する登録済みCommandへ変換し、Widget callbackを直接呼ばない。
- Editor ViewはProject revisionのread projectionであり、Widget stateをProjectへ直接writeしない。
- Project、Workspace、Panel local stateは別Storeとし、一つのserializationへ混在させない。
- Render PacketとSemantic Snapshotは同じcommitted Layout generationから作る。

## 6. Shared Core、Editor、Gameの分離

| Capability | MirakanUi Core | MirakanEditor Shell | Game UI Frontend |
|---|---:|---:|---:|
| UiRuntimeTree／generation | 所有 | 利用 | 利用 |
| Layout／Style／Event／Focus | 所有 | 利用 | 利用 |
| Hit Test／Semantic／Draw Packet | 所有 | 利用 | 利用 |
| `EditorViewDescriptor` | 禁止 | 所有 | 禁止 |
| Dock／Panel／Workspace | primitiveのみ | 所有 | 禁止 |
| `UiDocument`／ViewModel／Screen stack | primitiveのみ | 禁止 | 所有 |
| AuthoringCommandGateway | 禁止 | typed Portだけ利用 | 禁止 |
| DirectWrite | `IUiTextBackend` Portのみ | `mirakan.ui.directwrite.adapter`をcompose | 使用しない |
| TSF | `ITextInputService` Portのみ | `mirakan.ui.tsf.adapter`をcompose | Windowsでtext inputを持つGameだけ同じAdapterをcompose |
| UI Automation | `IUiAccessibilityBridge` Portのみ | `mirakan.ui.uia.adapter`をcompose | Windows Accessibilityを有効にするGameだけ同じAdapterをcompose |
| HarfBuzz／FreeType／ICU | Portのみ | ICUだけをlocale、boundary、message処理へ利用。HarfBuzz／FreeTypeはEditorHostへlinkしない | Target共通Backend |

Shipping `GameHost`のdependency closureに`mirakan.editor.*`、Authoring、Workspace、UIA Editor providerを含めない。CIはCMake graph、Named Module graph、link map、SBOMの四つで検査する。

EditorとGameで共有するのはalgorithmとcontractであり、Widget instance、Font cache、Focus、State Store、semantic generation、GPU resourceをProcess間共有しない。

## 7. Directory、CMake target、Named Module

基盤規約のdirectoryを次の具体形にする。

```text
engine/ui/
├─ core/                       # tree、state、generation、invalidation
├─ layout/                     # measure／arrange、virtualization
├─ events/                     # event、focus、hit test、command intent
├─ semantics/                  # platform非依存semantic snapshot
├─ text/                       # text contractとbackend port
├─ rendering/                  # draw packetとRendering Port
└─ backends/
   ├─ d3d12/
   ├─ windows/
   │  ├─ directwrite/
   │  ├─ tsf/
   │  └─ uia/
   ├─ harfbuzz_freetype/       # Game text backend
   ├─ android/
   └─ apple/
editor/
├─ ui/                         # EditorViewDescriptor、control catalog
├─ shell/                      # menu、toolbar、status、command palette
├─ docking/                    # DockLayoutTreeV1、drag transaction
├─ panels/                     # Domain read projection panel
├─ workspaces/                 # EditorWorkspaceV1
├─ semantics/                  # EditorSemanticSnapshotV1、AI projection
├─ backends/
│  └─ windows_ole/             # Editor用Clipboard／Explorer drag and drop
└─ recovery/                   # last-valid layout、safe recovery entry
```

| CMake alias | Named Module | 公開責務 |
|---|---|---|
| `mirakan::ui_core` | `mirakan.ui.core` | tree、handle、snapshot、state |
| `mirakan::ui_layout` | `mirakan.ui.layout` | typed layout、virtualization |
| `mirakan::ui_events` | `mirakan.ui.events` | event、focus、hit test |
| `mirakan::ui_semantics` | `mirakan.ui.semantics` | semantic contract |
| `mirakan::ui_text` | `mirakan.ui.text` | text backend port |
| `mirakan::ui_rendering` | `mirakan.ui.rendering` | draw packet、render port |
| `mirakan::ui_d3d12_adapter` | `mirakan.ui.d3d12.adapter` | MirakanUiDrawPacketV1のD3D12変換 |
| `mirakan::ui_directwrite_adapter` | `mirakan.ui.directwrite.adapter` | DirectWrite text layout／glyph analysis |
| `mirakan::ui_tsf_adapter` | `mirakan.ui.tsf.adapter` | TSF text input |
| `mirakan::ui_uia_adapter` | `mirakan.ui.uia.adapter` | UI Automation provider |
| `mirakan::ui_harfbuzz_freetype_adapter` | `mirakan.ui.harfbuzz_freetype.adapter` | Shipping Game text shaping／raster |
| `mirakan::editor_ole_adapter` | `mirakan.editor.ole.adapter` | Editor Clipboard／OLE drag and drop |
| `mirakan::editor_ui` | `mirakan.editor.ui` | Editor view／control |
| `mirakan::editor_shell` | `mirakan.editor.shell` | shell／command composition |
| `mirakan::editor_docking` | `mirakan.editor.docking` | docking／floating transaction |
| `mirakan::editor_semantics` | `mirakan.editor.semantics` | AI／UIA用Editor semantics |

依存は次のDAGに固定する。`A <- B`は「BがAへ依存する」を意味し、逆方向の依存を許可しない。

```text
foundation <- ui.{core,text}
ui.core <- ui.{layout,events,semantics,rendering}
ui.{core,layout,events,semantics,text,rendering} <- editor.ui
editor.ui <- editor.{docking,semantics}
editor.{ui,docking,semantics} <- editor.shell

ui.* Port <- {ui.d3d12,ui.directwrite,ui.tsf,ui.uia,ui.harfbuzz_freetype}.adapter
editor data-transfer Port <- editor.ole.adapter
EditorHost composition root
  -> platform.windows
  -> ui.d3d12.adapter
  -> ui.directwrite.adapter
  -> ui.tsf.adapter
  -> ui.uia.adapter
  -> editor.ole.adapter
Windows GameHost composition root
  -> platform.windows
  -> ui.d3d12.adapter
  -> ui.harfbuzz_freetype.adapter
  -> ui.tsf.adapter             # text input Capabilityが有効なGameだけ
  -> ui.uia.adapter             # Windows Accessibility Capabilityが有効なGameだけ
```

AdapterをCoreからimportしない。DirectWrite、TSF、UIA、Win32、D3D12 headerは`engine/ui/backends/windows`と`engine/platform/windows`のGlobal Module Fragmentまたはprivate implementation以外でincludeしない。

## 8. Core tree、ID、Frontend

### 8.1 TreeとID

```text
UiNodeHandle
  surface_id
  node_id
  tree_generation

UiRuntimeTree
  tree_generation
  root_node_id
  nodes[]
  state_store
  dirty_sets
```

- `UiNodeHandle`は非所有generation handleであり、Frameを越えて保存しない。
- Nodeの所有は`UiRuntimeTree`のdense storageが持ち、parent／childはNode IDで表す。
- Event handler、Render Packet、UIA provider、AI snapshotへraw pointerを渡さない。
- Node削除後の古いgenerationは`UiElementUnavailable`を返す。
- Tree mutationはUI threadの`UiMutationBatch`だけが行い、event dispatchまたはlayout traversal中に構造を変更しない。
- Mutationはdispatch終了後に順序付きで適用し、一回だけgenerationを進める。

### 8.2 二つのFrontend

EditorはC++ `EditorViewDescriptor`から、Gameはversioned `UiDocument`から同じvalidated internal node descriptorを生成する。

```text
EditorViewDescriptor
  view_type_id
  view_instance_id
  root_control_id
  controls[]
  command_bindings[]
  semantic_bindings[]

EditorControlDescriptor
  control_id
  control_type
  layout
  style_classes[]
  state_binding
  command_binding
  semantic
```

`EditorViewDescriptor`はarbitrary property bag、function pointer、`std::function`、RTTI class name、native handleを持たない。C++ view factoryはMCDで登録したControl typeとCommand IDだけを構成する。

### 8.3 Invalidation

Dirty reasonを次のclosed setにする。

- `structure`
- `layout`
- `style`
- `paint`
- `text`
- `hit_test`
- `semantics`

各reasonは必要なancestor／descendantだけへ伝播する。変更がないWindowはDraw Packetを再構築しない。Animation、caret、progress等のvolatile nodeは明示flagを持ち、Tree全体を毎Frame invalidationしない。

## 9. Layout、Style、Widget

共通Layout algorithm、logical unit、capacity、Style解決はUI規約を正本とする。Editor profileは追加で次を固定する。

- 座標は96 DPIを1.0とするlogical unit。Layout中にphysical pixelへ丸めない。
- Split ratioはunsigned Q16 fixed pointで保存し、Arrange時にlogical extentへ変換する。
- Text baseline、separator、focus outlineだけを最終DPIでpixel alignする。
- Scroll／Virtual Listはvisible rangeと前後2 viewportまでをmaterializeする。
- Outliner 100万EntityとAsset Browser 10万Assetでも全row Widgetを生成しない。
- Style selectorはControl type、class、stateだけで、CSS parser、descendant selector、arbitrary predicateを持たない。

C1 Editor Widget catalog:

- text、icon、image
- button、toggle、checkbox、radio、slider、spin field
- single／multi-line text editor
- combo、menu、context menu、tooltip
- tree、virtual list、table、property grid
- tab stack、split view、scroll view
- toolbar、status、notification、progress
- modal／non-modal overlay、focus scope
- graph surface、timeline surface、scene surface、profiler surface

Scene／Graph／Timeline／Profiler surfaceは`UiCanvasSurface`を使用する。ProducerはEngine C++ catalogへ登録した`EditorCanvasProducerId`で選び、Project C++またはAIが任意draw callbackを注入できない。

## 10. Event、Focus、Command、AI

### 10.1 Platform event

```text
PlatformUiEventV1
  event_id
  surface_id
  timestamp_ns
  device_id
  kind
  pointer_id
  position_lu
  delta_lu
  key_physical
  modifiers
  text_composition_ref
  window_generation
```

`kind`はpointer enter／leave／move／down／up／cancel／wheel、key down／up、text composition start／update／commit／cancel、focus gained／lost、window metrics changed、drag enter／over／leave／dropのclosed enumとする。

Eventはcapture、target、bubbleで配送する。Handler結果はhandled、focus request、pointer capture、local state delta、typed command request、announcementだけに限定する。HandlerからProject memory、D3D12 resource、Window procedure、AI Providerを直接呼ばない。

### 10.2 Command経路

```text
EditorCommandRequest
  command_id
  source_kind
  source_element_key
  project_revision
  selection_snapshot_id
  typed_arguments
  authorization_context
```

`source_kind`はhuman、keyboard、accessibility、AI、automation testを区別する。Projectを変更するCommandは必ずtyped `ProjectChangePrimitiveV1`へ変換し、完全登録済み外側MCD Operationが要求するbase revision、Risk、Validation、Approval、Receiptを省略しない。

Shortcut、menu、toolbar、context menu、Command palette、AI toolは同じ`EditorCommandId`を参照する。Mouse専用Commandを登録しない。

### 10.3 AI Semantic Interface

AIへ提供する`EditorContextSnapshotV1`は、選択されたPanelのSemantic Snapshot、Project read projection、Problems、active Task、Userが許可したDocument断片から作る。

- Widget座標、Draw primitive、Font glyph、native handleを含めない。
- Password、credential、secret、private clipboard、未許可Sourceを含めない。
- Context byte上限とredaction resultをReceiptへ記録する。
- AIはControl IDを説明とfocus requestに使用できるが、状態変更はCommand ID／typed `ProjectChangePrimitiveV1`で行う。
- `EditorContextSnapshotV1`のgenerationが古いProposalを自動実行しない。
- Debug Panel選択時もraw trace、画面pixel、無制限event列をこのSnapshotへ埋め込まない。Debugging規約のbounded `AiDebugContextV1`を参照し、Session／Query／Evidence ID／recorded revision／gap／redactionを失わない。

このSemantic Interfaceにより、AI統合は画面操作macroではなくMiraikanaiのAuthoring契約となる。

## 11. Rendering

### 11.1 Packet

```text
MirakanUiDrawPacketV1
  surface_generation
  layout_generation
  clip_table[]
  texture_refs[]
  glyph_atlas_refs[]
  primitives[]
```

C1 primitiveを次に限定する。

- solid／linear-gradient rect
- rounded rect／border
- line／polyline
- image／nine-slice image
- glyph run
- triangle mesh
- bounded cubic Bezier path
- Scene／Game View texture composite

Arbitrary shader、raw GPU address、descriptor、command list、HLSL sourceをPacketへ入れない。Custom UI effectはEngine登録済みpipeline ID、parameter schema、budgetを追加する仕様変更が必要である。

座標、color、width、radiusはfiniteでなければPacket validationを失敗させる。一つのtriangle meshはvertex 65,535、index 196,605、cubic Bezierは一path 4,096 segmentを上限とし、Frame aggregateは16.3節のhard capへ従う。TessellationはDPIとerror toleranceから決定するが、上限到達時に粗くして成功させずproducerをcapacity errorにする。

### 11.2 D3D12 UI pass

- `MirakanUiDrawPacketV1`をRendering Portへ渡し、D3D12 Adapterがvertex／index upload、pipeline、descriptor、clipを解決する。
- UIはdisplay resolutionでScene tone map後に描画し、World dynamic resolution、TAA、motion blurの対象外にする。
- 順序を変えない範囲でpipeline、texture、clipをbatchする。
- GPU resourceはsubmission serialまで保持し、UiTreeまたはPanel破棄で即時freeしない。
- one top-level Windowにつき一つのsurface generationを持ち、Swap Chain再作成時に古いPacketをrejectする。

### 11.3 DirectWrite text

EditorはDirectWriteでtext layoutとglyph runを生成し、private Windows Adapterが`IDWriteTextRenderer`を実装してglyph run、underline、strikethrough、inline objectをEngine packetへ変換する。C1の通常glyphは`IDWriteBitmapRenderTarget1`をgrayscale antialiasへ設定してR8 Editor atlasへ取り込み、D3D12 UI passでalpha compositeする。ClearType subpixel textureはtranslucent Panel、HDR、異なるpixel geometryで結果が変わるためC1では使用しない。

Color Fontは`IDWriteFactory4::TranslateColorGlyphRun`でCOLR／CPAL v0 layerを列挙し、各layerをR8 coverage＋typed colorとして描画する。SVG、COLR v1、embedded bitmap color glyphはC1 Editor rendererの対応外とし、DirectWriteが返すmonochrome base glyphを表示してDevelopment diagnosticを記録する。文字自体を欠落させない。Direct2DをEditor Widget／Layout／Rendering Frameworkとして採用しない。

Editor textの正規storageはUTF-8、DirectWrite／TSF境界だけUTF-16とする。System FontはEditor UIで利用できるが、Shipping Game layoutの正本にはしない。

### 11.4 Device loss

Device loss中は新しいGPU UI frameを生成せず、Authoring journalとWorkspaceをflushする。再作成に成功すればlast committed UiTreeから全GPU cacheを再構築する。再作成に失敗した場合は独立したWin32 recovery entryへ移り、Project recovery、log保存、終了だけを提供する。GPU failureを通常Editor UIが無表示になる状態で放置しない。

## 12. Text input、IME、Clipboard、Drag and Drop

### 12.1 TSF

Windows UI threadはCOM STAとOLEを初期化し、TSF Thread Manager、Document Manager、Contextを所有する。編集可能Controlごとにtext modelを保持し、focus時だけactive TSF documentへ接続する。

`WindowsTextStoreAdapter`は`ITextStoreACP`をprivate実装し、次を満たす。

- UTF-8 byte offset、grapheme index、UTF-16 ACP offsetの`TextIndexMap`を保持する。
- selection、composition、caret rectangle、screen pointからACP、ACPからscreen rectangleを双方向変換する。
- TSF lock protocolを守り、lock callback中にUiTreeを再入変更しない。
- composition updateはlocal text draftであり、commit後にだけEditor CommandまたはUiState deltaを発行する。
- password／secret fieldのcomposition、clipboard、AI context、logをredactする。
- IME candidate UIをMiraikanaiが偽装せず、TSFのsystem integrationへ必要なgeometryを返す。

Japanese Microsoft IME、Chinese Pinyin、Korean IME、dead key、emoji、surrogate pair、combining sequenceをconformance fixtureにする。

### 12.2 Clipboard

- Copy／Cut／PasteはUser commandへの応答時だけ実行する。
- Windows plain textは`CF_UNICODETEXT`、Engine内textはvalid UTF-8とする。Asset／Entity参照はversioned registered formatとportable text fallbackを同時提供する。
- Project objectをraw pointer、memory dump、native pathだけで渡さない。
- Secret fieldはCopyを既定拒否し、明示許可された値だけOS clipboardへ出す。
- Clipboardからの構造化PasteはSchema、size、path、Project、Capabilityを検証してChangeSetへ変換する。

### 12.3 OLE drag and drop

Explorerとのfile transferはOLE `IDataObject`／`IDropTarget` Adapterを使用する。Internal Panel dockingとEntity／Asset dragはMirakanUi typed drag payloadを使用し、OLE objectをUiTreeへ保持しない。Dropはpreview validationを通し、pointer releaseだけでProjectへ直接writeしない。

## 13. Window、DPI、Docking、Workspace

### 13.1 Window model

- Main Windowと各floating Windowだけがtop-level HWNDとSwap Chainを持つ。
- 通常Widget、Panel、Tab、Menu、Tooltipごとにchild HWNDを作らない。
- C1はOS non-client frame、caption、minimize／maximize／resizeを使用する。
- PopupはWindow内overlayを既定とし、Window boundsを越えるmenu／tooltipだけowned top-level surfaceを使う。
- Window closeはtyped requestであり、Window procedure内でSave／Build／AI cancelを同期実行しない。

### 13.2 DPI

Process起動直後にPer-Monitor V2を設定する。Windows Adapterは`WM_DPICHANGED`で提示されたWindow rectangleを`SetWindowPos`へ適用し、`PlatformUiEventV1::window_metrics_changed`を発行して次を一つのWindow metrics transactionで更新する。

- logical／physical extent
- pixel scale
- monitor／work area
- text raster scale
- cursor／hit target scale
- Swap Chain surface generation

古いDPI generationのpointer、Layout、Packetを新Windowへ適用しない。100、125、150、175、200、250%と異なるDPIのmonitor間移動をtestする。

### 13.3 Dock model

```text
DockLayoutTreeV1
  schema_version = 1
  root_node_id
  nodes[]
    SplitNode
      node_id
      axis
      ratio_q16
      first_child
      second_child
    TabStackNode
      node_id
      panel_instance_ids[]
      active_panel_instance_id
  floating_windows[]
    window_id
    root_node_id
    logical_rect
    monitor_signature
    last_dpi
```

- Treeはcycle、duplicate Panel、missing child、empty split、ratio overflow、minimum size違反を拒否する。
- Dock drag中は`DockPreviewTransaction`を使い、drop完了前に正規layoutを書き換えない。
- Dock zoneはleft、right、top、bottom、center-tabの五つ。
- KeyboardからPanel選択、dock zone選択、resize、floating、main Windowへ戻す操作を提供する。
- Workspace saveはlast-valid snapshotを原子的に置換し、corrupt current snapshotからProject stateを変更しない。
- Monitor消失時はWindowの20%以上が全work area外ならprimary work areaへ回収する。

C1はMain Windowを含むtop-level Window最大32、Panel instance最大512、Dock node最大2,048とする。上限超過Workspaceは一部だけを開かず、`EditorWorkspaceCapacityExceeded`でloadを拒否してlast-valid Workspaceを維持する。

Workspaceの製品構成、保存先、built-in profileはEditor UX規約を正本とする。

## 14. Semantic Tree、Accessibility、Automation

### 14.1 Snapshot

```text
EditorSemanticSnapshotV1
  surface_generation
  tree_generation
  layout_generation
  root_element_key
  elements[]

EditorSemanticElementV1
  element_key
  parent_key
  role
  name
  description
  value
  state
  actions[]
  bounds_physical_px
  command_ids[]
  privacy_class
```

Semantic Snapshotは`EditorViewDescriptor`、read projection、committed Layoutから生成し、Draw Packet、color、glyph、hit-test結果を解析して作らない。

### 14.2 Windows UI Automation

各top-level HWNDをUIA fragment rootとし、Windowless Widgetをfragment elementとして公開する。Providerは標準UIA control typeとpatternを優先する。

| MirakanUi control | 必須UIA pattern |
|---|---|
| button | Invoke |
| toggle／checkbox | Toggle |
| slider／spin field | RangeValueまたはValue |
| text input／Source editor | Text、TextEdit、Value |
| tree／list／table | Selection、SelectionItem、Scroll、ItemContainer、VirtualizedItem、Grid／Table |
| tab | Selection、SelectionItem |
| docked／dockable Panel | Dock |
| floating top-level Panel Window | Window、Transform |
| drag source／drop target | Drag、DropTarget |

Providerはimmutable Semantic Snapshotを読み、UiNode pointerを保持しない。古いgenerationには`UIA_E_ELEMENTNOTAVAILABLE`を返す。ActionはUI threadへtyped Commandをqueueし、COM callback threadからProjectを変更しない。標準patternで表現できる場合はcustom UIA patternを登録しない。

- Secret／password ControlはUIA password propertyを設定し、Value、Text、name、descriptionへ内容を公開しない。
- UIA actionの`source_kind=accessibility`はHuman操作と同じRisk、Validation、Approvalを要求し、UIA clientの存在を追加権限として扱わない。

### 14.3 AIとの分離

UIAはassistive technologyとblack-box UI testのWindows標準interfaceである。内蔵AI、MCP、CLIの正規操作経路には使用しない。AIは`EditorContextSnapshotV1`とtyped toolを使用し、UIA権限からAuthoring権限を得ない。

## 15. Design Systemと人間工学

`EditorStyleProfileV1`は次のtyped tokenを持つ。

- semantic color
- typography role
- spacing
- size
- border／radius
- elevation／overlay
- icon
- motion duration／easing
- density
- focus／selection／validation／AI state

規則:

- Engine validation、AI proposal、User selection、Runtime stateを同じ色／iconだけで区別しない。
- `compact`、`standard`、`comfortable`は同じControlを使い、hit target、row、spacing、text scaleだけをProfile化する。
- Font scale 200%、High Contrast、reduced motionで全C1 flowを操作可能にする。
- Tooltipだけにrequired情報を置かない。
- Drag-only操作にCommand、keyboard、context menuの代替を持つ。
- Focus indicatorとprimary action hit targetの最小値は[Editor Workspace UX §12.2](editor-workspace-ux.md#122-sizeとdpi)を正本として消費し、本書で値を再定義しない。
- Iconにはaccessible nameまたは隣接textを持たせ、形状だけの未知iconをprimary commandにしない。

## 16. Ownership、Memory、Thread

### 16.1 Ownership

- UiTree、Layout cache、State StoreはUI session ownerがRAIIで所有する。
- Widget parent／child、Focus、Hit Test、Semantic参照はID／generationで表す。
- OS COM objectはWindows AdapterのRAII wrapperだけが所有する。
- `std::shared_ptr`を全Widget treeの既定ownershipにしない。
- Callback captureでPanel、Project、Window、GPU resourceの寿命を延長しない。
- Render resource解放はGPU submission serial、Semantic provider解放はsnapshot lease終了を待つ。

### 16.2 Thread

| Thread | 所有処理 | 禁止 |
|---|---|---|
| Windows UI thread／STA | HWND、message、OLE、TSF、UiTree mutation、event、focus、layout commit | file I/O、Build、Provider call、長時間validation |
| Render thread | immutable Draw Packet、D3D12 UI pass、GPU cache | UiTree mutation、TSF、Authoring write |
| Worker | text shaping request、search、thumbnail、immutable Panel projection候補 | HWND／COM STA object、final tree mutation |
| UIA callback | immutable Semantic Snapshot read、typed action queue | UiTree pointer、Project write、Build wait |

Worker resultはProject revision、Panel generation、Tree generation、Style／Font generationをUI threadで再検査してから統合する。

### 16.3 Editor UI budget

Windows Reference hardware、2560×1440、100% DPI、Production Workspaceを初期計測条件とする。Reference hardware構成（CPU、RAM、storage、GPU、driver固定方針）の正本は[Performance／Capacity](../04-runtime/performance-capacity.md)であり、本書は構成値を定義しない。

| Budget | C1 hard cap |
|---|---:|
| UiTree、Layout、Style、State、Semantic | 96 MiB CPU |
| Text model、DirectWrite result、CPU glyph cache | 64 MiB CPU |
| Per-frame UI transient | 16 MiB CPU |
| UI glyph／icon atlas、UI geometry、Panel chrome | 128 MiB GPU |
| active UiNode | 262,144 |
| active semantic element | 131,072 |
| draw primitive／Window／frame | 262,144 |
| draw primitive／EditorHost／frame | 524,288 |
| generated UI vertex／EditorHost／frame | 262,144 |
| generated UI index／EditorHost／frame | 786,432 |
| clip depth | 32 |

generated UI vertex／indexへはtriangle meshとbounded cubic Bezier pathのtessellation結果だけを計上する。solid／linear-gradient rect、rounded rect／border、line／polyline、image／nine-slice image、glyph run、texture compositeはinstanced primitiveとしてdraw primitive上限だけへ計上し、vertex／index budgetへ計上しない。primitive上限とvertex／index上限は同時に適用し、先に到達した上限でproducerをtyped capacity errorにする。

PanelのDomain data projection、thumbnail、Scene render target、Source file bufferは上表へ重複計上せず、それぞれEditor／Asset／Rendering／Source budgetへchargeする。上限超過時に一部Controlを黙って消さず、typed capacity error、virtualization不足、owner tagをProblemsへ出す。

## 17. PerformanceとTelemetry

| Metric | C1 gate |
|---|---:|
| Idle時の不要UiTree mutation | 0／frame |
| dirty subtree event＋style＋layout＋packet CPU P95 | 4.0 ms以下 |
| Full Window layout＋packet CPU P95 | 8.0 ms以下 |
| UI GPU pass P95、2560×1440 | 1.0 ms以下 |
| Dock preview update P95 | 16.67 ms以下 |
| steady state heap allocation | 0／idle frame |

Input→visual response、Panel／Workspace操作、UI thread blockingのend-to-end acceptanceは[Editor Workspace／UX](editor-workspace-ux.md#13-performance)だけが所有する。本節はその達成に必要なFramework内部costを所有し、同じ受入値を再定義しない。

TelemetryはWindow、Panel、Control type、dirty reason、layout、paint、glyph、primitive、UIA request、event latencyをtag付けする。User text、Project path、AI prompt、credentialをtraceへ記録しない。

性能最適化はfull redraw回避、virtualization、batch、atlas、immutable snapshot、dirty subtreeの順で行う。Profile根拠なくWidget pool、lock-free tree、custom general allocatorを導入しない。

## 18. Failure、Security、Recovery

| Failure | 規定動作 |
|---|---|
| invalid `EditorViewDescriptor` | 対象View生成を拒否しProblemsへ型付きerror。既存Viewを維持 |
| stale event／handle／snapshot | 操作を拒否し最新generationから再取得 |
| corrupt Dock／Workspace | last-valid、次にbuilt-in Productionへfallback。Project不変 |
| missing monitor／DPI change | work areaへ回収し一Transactionでre-layout |
| DirectWrite／Font failure | fallback Fontとdiagnostic。文字を無表示にしない |
| TSF failure | 対象Text controlをread-onlyへし、plain key eventでIMEを偽装しない |
| UIA provider failure | Release gate失敗。Accessibilityを無効にして配布しない |
| D3D12 device loss | GPU cache破棄、再作成。失敗時はWin32 recovery entry |
| Canvas producer capacity超過 | 当該surfaceをtyped error viewへ切り替え、他Panelを維持 |
| AI context projection失敗 | AI actionを停止。Manual Editorを継続 |

Project由来のtext、icon、Asset metadata、drag payload、clipboard payloadをuntrusted inputとしてsize、UTF、Schema、path、Asset typeを検証する。Text、tooltip、log、AI explanationをformat string、markup、shader、commandとして再解釈しない。

C1で任意binary Editor pluginを読み込まない。First-party PanelもCommand Registry、Capability、MCD、CMake graphへ登録されていないControl type、Canvas producer、Platform serviceを使用できない。

## 19. Verification

### 19.1 Unit／property／fuzz

- Tree cycle、duplicate ID、generation、mutation batch、stale handle
- Measure／Arrange、min／max、baseline、clip、virtualization、dirty propagation
- capture／target／bubble、focus recovery、pointer capture cancel、shortcut conflict
- Dock tree cycle、ratio、minimum size、tab reorder、floating、monitor recovery
- UTF-8／UTF-16／grapheme／ACP mapping、selection、composition
- Semantic role／state／action、generation、privacy redaction
- Draw packet bounds、clip depth、primitive capacity、batch order preservation

### 19.2 Integration

- Main Window、floating Window、different-DPI multi-monitor、hot unplug
- Japanese／Chinese／Korean IME、emoji、BiDi、clipboard、Explorer file drop
- D3D12 device loss、Swap Chain resize、Font cache rebuild
- UIA Invoke、Value、Text、Selection、VirtualizedItem、Dock、Drag／DropTarget
- AI Contextからtyped Commandを実行し、screen coordinateを使用しないこと
- Manual edit、AI proposal、UIA action、Undoが同じCommand／ChangeSetへ収束すること

### 19.3 Accessibility／UX

- keyboard-only、Narrator、NVDAでProject open、Scene選択、Inspector変更、Save、Play、Stop
- Accessibility Insightsでname、role、state、pattern、focus、bounds、event
- 100～250% DPI、High Contrast、200% Font、reduced motion
- drag代替、focus trapなし、Panel回収、shortcut remap

### 19.4 Performance／soak

- 100万Entity virtualized Outliner
- 10万Asset virtualized Browser
- 50,000 node Graphのviewport内materialization
- 8時間Editor soak、1,000 Workspace switch、10,000 dock transaction
- Performance／Capacity規約が定める両Windows Reference構成でCPU、GPU、memory、allocation、device loss

### 19.5 Dependency negative gate

CIはsource include、CMake graph、link map、SBOM、license noticeを検査し、Editor closureにDear ImGui、Qt、WinUI control、WPF、Windows Forms、GTK、wxWidgets、Electron、CEFが入ったBuildを失敗させる。文字列だけで判定せず、resolved package、import library、binary dependencyを検査する。

## 20. PhaseとPromotion Gate

Phase順序、Capability maturity、release threshold、promotion authorizationは[Product Plan](../00-product/product-plan.md)だけが所有する。本書はProduct Planが有効化したUI Capabilityを消費し、phase番号や共有thresholdを割り当てない。

- 有効化されたCapabilityだけにUI surfaceを公開し、未有効化ならtyped `CapabilityNotActivated`を返す。
- 各UI成果物は本書のWidget／Layout／Event／Semantic／Accessibility／Performance contractへの適合evidenceを提供する。
- Editor shell、Game UI、Platform backendはProduct Planのactive capability setを同じ入力として使用する。
- release候補には本書のQualification結果を添付するが、promotion可否はProduct PlanとGovernance Ownerへ渡す。

## 21. 明示的に採用しないもの

- Dear ImGuiを初期版だけ使用して後から置換する二重実装
- Qt等を隠す薄いwrapper
- UI状態をFrameごとのC++ local variableだけに置く設計
- Widgetごとの`shared_ptr` ownership
- Widget、Panel、AIへraw Engine pointerを公開すること
- PixelからSemantic Treeを復元すること
- AIがUIA、mouse macro、screen coordinateを正規操作経路にすること
- CSS／HTML互換を目指すこと
- Project C++がEditor Widget callbackをProcess内注入すること
- Platform Adapter型をNamed Module、serialization、MCDへ公開すること
- Accessibilityを実装後の追加機能として扱うこと
- GPU device loss時にProject recovery手段を失うこと

## 22. Definition of Done

C1独自Editor UI Frameworkは次をすべて満たした時点で完了する。

1. Editor binaryとSBOMに禁止GUI toolkitが含まれない。
2. MirakanUi CoreをEditorとGame UIが共有し、Shipping Game closureにEditor targetが含まれない。
3. Scene、Outliner、Inspector、Asset、Source、Session、Console、Problems、Profiler、Animation Timeline、Debug Timeline、Causality、Breakpoint／Watch、Replay、Reproduction、AI Partnerが独自Widgetで動作し、Debug dataはDebugging規約のtyped projectionだけを使う。
4. Dock、resize、tab reorder、floating、multi-monitor、複数Workspace保存と回復が成立する。
5. DirectWrite、TSF、UIAをAdapter内へ隔離し、UTF-8／IME／screen readerを満たす。
6. Human、keyboard、assistive technology、AIが同じtyped Command／ChangeSet／Validationを通る。
7. AIがWidget pointer、HWND、UIA、screen coordinateを使用せずゲームを作成・修正できる。
8. DPI、High Contrast、200% Font、reduced motion、keyboard-only、Narrator、NVDA fixtureが合格する。
9. 100万Entity、10万Asset、50,000 node Graph、8時間soakでcapacity、memory、performance gateを満たす。
10. Device loss、corrupt Workspace、monitor消失、AI切断、TSF／UIA failureでProject正本を破損しない。

## 23. 一次資料と判断の対応

| 判断 | 一次資料 |
|---|---|
| 複雑なEditor UIにはRetained tree、Layout、debug toolingが必要 | [Unity UI Toolkit](https://docs.unity3d.com/Manual/UIElements.html)、[Unity UI systems comparison](https://docs.unity3d.com/Manual/UI-system-compare.html) |
| 独自C++ Editor UI FrameworkとDockingは成立する | [Unreal Engine Slate Overview](https://dev.epicgames.com/documentation/unreal-engine/slate-overview-for-unreal-engine?lang=en-US)、[Slate Widget Reflector](https://dev.epicgames.com/documentation/unreal-engine/using-the-slate-widget-reflector-in-unreal-engine?lang=en-US) |
| Engine自身のRenderer／UIでC++ Editorを構築できる | [Godot Introduction to editor development](https://docs.godotengine.org/en/stable/engine_details/editor/introduction_to_editor_development.html)、[Godot UI toolkit FAQ](https://docs.godotengine.org/en/latest/about/faq.html#what-user-interface-toolkit-does-godot-use) |
| Qtを利用するEngine Editorもあるが、本Projectはこの方式を選ばない | [O3DE Tools UI Developer's Guide](https://www.docs.o3de.org/docs/tools-ui/) |
| Windows desktopはPer-Monitor V2を推奨し、DPI変更時にappが再配置する | [High DPI Desktop Application Development](https://learn.microsoft.com/en-us/windows/win32/hidpi/high-dpi-desktop-application-development-on-windows)、[`WM_DPICHANGED`](https://learn.microsoft.com/en-us/windows/win32/hidpi/wm-dpichanged) |
| DirectWrite layoutをcustom rendererへ渡し、grayscale bitmap glyphを生成できる | [Render Using a Custom Text Renderer](https://learn.microsoft.com/en-us/windows/win32/directwrite/how-to-implement-a-custom-text-renderer)、[`IDWriteBitmapRenderTarget1`](https://learn.microsoft.com/en-us/windows/win32/api/dwrite_1/nn-dwrite_1-idwritebitmaprendertarget1)、[DirectWrite Color Glyph Sample](https://learn.microsoft.com/en-us/samples/microsoft/windows-universal-samples/dwritecolorglyph/) |
| TSF applicationはThread ManagerとText Storeを実装する | [TSF Thread Manager](https://learn.microsoft.com/en-us/windows/win32/tsf/thread-manager)、[`ITextStoreACP`](https://learn.microsoft.com/en-us/windows/win32/api/textstor/nn-textstor-itextstoreacp) |
| Custom-drawn controlはUI Automation providerとpatternを実装する | [UI Automation Providers Overview](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-providersoverview)、[Interfaces for Providers](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-interfaces)、[Control Patterns](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-implementinguiautocontrolpatterns) |
| Explorer連携drag/dropはOLE data transferを使用する | [Data Transfer Interfaces](https://learn.microsoft.com/en-us/windows/win32/com/data-transfer-interfaces)、[Shell Drag-and-Drop](https://learn.microsoft.com/en-us/windows/win32/shell/dragdrop) |
| ClipboardはUser commandへの応答として扱う | [About the Clipboard](https://learn.microsoft.com/en-us/windows/win32/dataxchg/about-the-clipboard)、[Clipboard Formats](https://learn.microsoft.com/en-us/windows/win32/dataxchg/clipboard-formats) |
| Keyboard、Focus、target size、drag代替の人間工学原則 | [WCAG 2.2](https://www.w3.org/TR/WCAG22/) |

外部資料はPlatform APIと既存Engineの方式を示す。`MirakanUi Core`、`EditorViewDescriptor`、`EditorSemanticSnapshotV1`、AI／UIA分離、Command収束、UI固有contractとfixtureはMiraikanai Engine独自の規範決定である。
