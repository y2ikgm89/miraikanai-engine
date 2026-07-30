# Miraikanai Engine 独自Editor UI Framework／Shellアーキテクチャ規約

- 文書ID: mirakan.arch.editor-ui-framework
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: MirakanUi Core、Editor Shell、Widget／Layoutの基本契約、Event／Focus／Command、UI Rendering、Window／Dock、Text／IME、Semantic Tree、Accessibility bridge、UI ownershipと検証
- 非正本範囲: Project transaction、Workspace journey、Asset operation、Gameplay model、外部Tool・SDK・Libraryの固定値、Runtime／Rendering／Platform内部。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Core Architecture](../02-foundation/core-architecture.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)
- 関連文書: [Editor UI Design System Catalog](../appendices/editor-ui-design-system-catalog.md)、[Product Plan](../00-product/product-plan.md)、[Core architecture](../02-foundation/core-architecture.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[C++23 Language／Public Surface](../02-foundation/cpp23-modules.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[Project state](project-state.md)、[Editor Workspace UX](editor-workspace-ux.md)、[UI／Text／Localization／Accessibility](../07-platform/ui-text-localization-accessibility.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-26

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
| C++ target、Public Header、Memory／Pointer、Dependency lock | 基盤規約とC++規約 |

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

Shipping `GameHost`のdependency closureに`mirakan.editor.*`、Authoring、Workspace、UIA Editor providerを含めない。CIはCMake graph、public／private include graph、link map、SBOMの四つで検査する。

EditorとGameで共有するのはalgorithmとcontractであり、Widget instance、Font cache、Focus、State Store、semantic generation、GPU resourceをProcess間共有しない。

## 7. Directory、CMake target、Public Header

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

| CMake alias | Public include root | 公開責務 |
|---|---|---|
| `mirakan::ui_core` | `mirakan/ui/core/` | tree、handle、snapshot、state |
| `mirakan::ui_layout` | `mirakan/ui/layout/` | typed layout、virtualization |
| `mirakan::ui_events` | `mirakan/ui/events/` | event、focus、hit test |
| `mirakan::ui_semantics` | `mirakan/ui/semantics/` | semantic contract |
| `mirakan::ui_text` | `mirakan/ui/text/` | text backend port |
| `mirakan::ui_rendering` | `mirakan/ui/rendering/` | draw packet、render port |
| `mirakan::ui_d3d12_adapter` | private | MirakanUiDrawPacketV1のD3D12変換 |
| `mirakan::ui_directwrite_adapter` | private | DirectWrite text layout／glyph analysis |
| `mirakan::ui_tsf_adapter` | private | TSF text input |
| `mirakan::ui_uia_adapter` | private | UI Automation provider |
| `mirakan::ui_harfbuzz_freetype_adapter` | private | Shipping Game text shaping／raster |
| `mirakan::editor_ole_adapter` | private | Editor Clipboard／OLE drag and drop |
| `mirakan::editor_ui` | `mirakan/editor/ui/` | Editor view／control |
| `mirakan::editor_shell` | `mirakan/editor/shell/` | shell／command composition |
| `mirakan::editor_docking` | `mirakan/editor/docking/` | docking／floating transaction |
| `mirakan::editor_semantics` | `mirakan/editor/semantics/` | AI／UIA用Editor semantics |

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

本catalogはControl capabilityのclosed集合であり、個別Panelの見た目、Semantic、Command、AI操作を暗黙に定義しない。すべてのinteractive Controlとsemantic composite rootは§15.6のexact `pattern_id`へ一件解決し、非interactiveなglyph、separator、rail等は親Patternの`anatomy_slots[]`として扱う。Panel固有class、表示名、Widget tree上の偶然の位置から未登録Patternまたはvariantを生成しない。

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

`kind`はpointer enter／leave／move／down／up／cancel／wheel、key down／up、text composition start／update／commit／cancel、focus gained／lost、window metrics changed、appearance changed、motion preference changed、advanced effects preference changed、scrollbar preference changed、message duration preference changed、drag enter／over／leave／dropのclosed enumとする。appearance／preference changeはOS設定の再読込を促す内部通知であり、設定値、Widget target、Commandをpayloadに持たず、`EditorCommandRequest`へ変換しない。

Eventはcapture、target、bubbleで配送する。Handler結果はhandled、focus request、pointer capture、local state delta、typed command request、announcementだけに限定する。HandlerからProject memory、D3D12 resource、Window procedure、AI Providerを直接呼ばない。

### 10.2 Command経路

```text
EditorCommandRequest
  command_request_id: UUIDv7
  command_id
  actor_kind
  input_channel
  source_element_key
  project_revision
  attention_snapshot_id
  typed_arguments
  authorization_context
```

`actor_kind`は`human | ai | automation`、`input_channel`は`pointer | keyboard | accessibility | menu | toolbar | context_menu | command_palette | internal_ai | mcp | cli | test`のclosed値とする。旧来の`source_kind`のように行為者と入力経路を一Fieldへ混在させない。validatorは`human`に`pointer`／`keyboard`／`accessibility`／UI command経路だけ、`ai`に`internal_ai`／`mcp`だけ、`automation`に`cli`／`test`だけを許可し、不正な組合せを拒否する。`attention_snapshot_id`は§15.7のexact snapshotを指すが、selectionをauthorizationへ昇格させない。Projectを変更するCommandは対象、expected revision、権限を再指定して必ずtyped `ProjectChangePrimitiveV1`へ変換し、完全登録済み外側MCD Operationが要求するbase revision、Risk、Validation、Approval、Receiptを省略しない。

`command_request_id`は一回のCommand attemptをcorrelateするUUIDv7であり、target、permission、idempotency key、Project revisionを代用しない。Reference 01のscalar property mutationがChangeSetまで到達した場合、`ProjectChangeSetV1.request_id`はこの値とbyte equalityにする。outer Operationのretry／idempotencyは既存MCD規約だけが所有し、Editorが同一IDから別Command、別ChangeSet、または別Commitを再生成しない。

Gateway directのunavailable rejectionは表示文や`disabled_reason_key`だけで済ませず、次のtrace subrecordへ閉じる。

```text
EditorCommandUnavailableResultV1
  command_request_id: UUIDv7
  command_id
  stable_target_ref
  canonical_field_ref
  expected_project_revision
  rejection_kind = unavailable
  reason_key
  diagnostic: MirakanDiagnosticV1
  change_set_ref: absent
  receipt_refs: []
```

これは`command_trace`内だけのEditor projectionであり、新しいMCD Type、Operation、Receipt、Registry、または権限rootではない。`diagnostic`は[Executable Contracts §12.1](../02-foundation/executable-contracts.md#121-mirakandiagnosticv1)の完全な`DiagnosticCodeRefV1`を持つ`MirakanDiagnosticV1`でなければならず、`reason_key`はその`message_key`とexact一致する。Gateway、AI、UIA、testがlocalized text、bare code、tooltipからこの結果を合成してはならない。`rejection_kind=unavailable`ではChangeSet refのpresenceまたはReceipt一件でもtraceをinvalidにし、Project root／revisionはbefore値とexact一致する。

Shortcut、menu、toolbar、context menu、Command palette、AI toolは同じ`EditorCommandId`を参照する。Mouse専用Commandを登録しない。

#### 10.2.1 Command Registryのactivation境界

`EditorCommandRegistry`は、current Contract setとownerの完全登録済みCommand bindingからcompilerが生成するread-only execution viewである。これはMCD kind、外側Operation、Capability、または独立したwrite authorityではない。Registry recordはlogical `command_id`、version、typed argument schema、target／revision binding規則、Action投影規則、参照するouter Operation／typed `ProjectChangePrimitiveV1` branch、availability bindingだけをexact refで持つ。Risk、Approval、Policy、Validator、Receipt、Project publication algorithmをCommand recordへ複写・上書きせず、`latest`、display label、Panel、Widget、shortcut、iconからtargetまたは権限を補完しない。

Project mutation Commandは、Registry recordがcurrent Contract setの完全なouter Operation routeとprimitive coverageへ解決する場合だけActionを投影できる。Command自身は`operation.authoring.changeset.commit`ではなく、`AuthoringCommandGateway`へtyped ChangeSet pipelineを要求するpresentation／semantic入口である。提案、Validation、Preview、Commitの各phaseとそのRisk／Approvalは外側Operationが決定し、Commitは既存規約どおり`trusted_internal`だけがfresh Approvalを再検証して実行する。Human、UIA、AI、MCP、CLI、testのいずれもCommand recordまたは`EditorSemanticActionV1`からCommit Operationを直接呼び出せない。

`command.editor.property.commit-value@1`は次のfuture `project_mutation` Commandとしてだけ定義する。これはcurrent Registry record、current MCD Operation、Provider／MCP Tool、Capabilityを作る記述ではない。

| 項目 | fail-closed contract |
|---|---|
| typed input closure | 一件のStable target ref、一件の`canonical_field_ref`、そのfield schemaへexactに解決する一件のtyped scalar value、`ProjectRevisionBindingV1`の`project_id`／`project_revision`／`document_set_sha256`、authorization contextを必須にする。`EditorCommandRequest.project_revision`はこのbindingのrevisionとbyte equalityでなければならない。表示文字列、unit文字列、row index、screen coordinate、Runtime valueを入力またはidentityにしない |
| primitive closure | Reference 01のscalar editは一件だけの`SetComponentField`へ変換する。自由形式`SetJsonPointer`、任意field write、複数fieldの暗黙展開、Component replacementを許さない。primitive target、field、typed value、expected document revision、ChangeSetのbase Project revisionは上のbindingとexact一致する |
| outer route | `operation.authoring.changeset.propose`、`validate`、`preview`、`commit`の四件と、そのprimitive coverage、Policy、Validator、Diagnostic、Receipt／Publication closureが同じatomic Contract set transactionで完全登録された後だけ実行可能にする。成功Commitは新しいEditor固有Receiptを作らず、[Project State §5.3](project-state.md#53-commit-algorithm)の`PublicCommitClosureV1`、`PublishedDomainReceiptV1`、`PublicPublicationMarkerV1`とbefore／after Project refへ閉じる |
| actor boundary | Human pointer／keyboard／accessibilityとAI `internal_ai`は、有効なAction projectionが返したtyped argumentだけを要求できる。AIはProposalを作れてもApprovalまたはtrusted-internal Commitを実行しない。`automation/test`のGateway direct requestはdefense-in-depth fixtureだけであり、Production Registry／AI Toolへ投影しない |
| runtime read-only | `live_edit=runtime_read_only`ではSemantic Actionを`enabled=false`、`disabled_reason_key=reason.live-edit.runtime-read-only`にし、surface bindingはCommand requestを一件も発行しない。Gateway direct fixtureだけは同じ`reason_key`と`DiagnosticCodeRefV1`を持つ`EditorCommandUnavailableResultV1`で拒否し、Project root／revision、ChangeSet、Receiptをすべて不変／emptyにする |
| current status | `planning.operation_family.authoring_changeset_execution`のcurrent集合は`[]`、Capabilityは`not_activated`である。したがってこのCommandのcurrent execution record／Action projectionは存在せず、未Activation時のcandidate dispatchは`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`で拒否する。Reference 01の`D.edit`／`D.reject`は上記activation後にだけmaterializeできるexpected behaviorであり、現在のManifest、baseline、Receiptを主張しない |

このCommandをmaterializeする唯一の入口は[Executable Contracts §21.1](../02-foundation/executable-contracts.md#211-既存domain文書から回収した未登録operation候補)の`activation.authoring.changeset_execution.v1`である。同一transactionでCommand record、argument／target binding、`SetComponentField` coverage、四outer Operation、availability binding、`reason.live-edit.runtime-read-only`とexactに対応する`DiagnosticCodeRefV1`、Action projection、Presentation binding、`EditorCommandUnavailableResultV1`、Reference 01 driver expectationを解決しなければならない。一部のread-only record、labelだけのCommand、proposalだけのshortcut、またはCommitだけの先行activationを許さない。

Command Registryの表示用subrecordは次だけを持つ。

```text
EditorCommandPresentationDescriptorV1
  presentation_id
  command_id
  presentation_context              # menu / toolbar / context_menu / command_palette
  label_key
  tooltip_key?
  shortcut_ref?
  presentation_icon_token_ref?
```

`presentation_id`はstable nonlocalized ID、`presentation_context`はclosed値である。`presentation_icon_token_ref`は§15.2のexact `EditorIconTokenContractV1` refだけを許し、source icon ID、SVG path、color、任意glyphを持たない。一つの表示用recordは一件のCommand Registry実行recordへ解決するが、同じCommandは複数のpresentation recordを持てる。表示用recordはCommand identity、target、Risk、authorization、Action可否を所有せず、visible labelがないicon-only command controlは`label_key`由来のaccessible nameと`tooltip_key`を必須にする。iconからAction、shortcut、permissionを推測しない。

### 10.3 AI Semantic Interface

AIへ提供する`EditorContextSnapshotV1`は、選択されたPanelのSemantic Snapshot、Project read projection、Problems、active Task、Userが許可したDocument断片から作る。AIが必要とするのは「どこをクリックするか」ではなく、「何が対象で、現在どの状態で、どのtyped actionが許可されるか」である。

```text
EditorContextSnapshotV1
  snapshot_version: 1
  snapshot_id: StableId
  attention_snapshot_ref
  attention_generation
  semantic_snapshot_ref
  semantic_content_hash
  project_revision_ref: ProjectRevisionRefV1
  selected_context_bindings[0..5]: AiTaskContextProjectionBindingV1
  elements[]                         # bounds、draw/style値を除く
  virtual_collections[]              # total、realized、omitted、continuation
  allowed_actions[]                  # command、argument schema、risk、approval、expected revision
  redaction_summary
  context_byte_count
  snapshot_content_hash: SHA-256
```

- Widget座標、Draw primitive、Font glyph、native handle、layout hashを含めない。
- Password、credential、secret、private clipboard、未許可Sourceを含めない。
- Context byte上限とredaction resultをReceiptへ記録する。
- `selected_context_bindings`は[AI Security／Approval §5](../01-governance/ai-security-approval.md#5-beginner-questionsassumptions理解条件)が所有するexact nested shapeを消費し、Userが送信Previewで許可したactive channelと明示的な関連sliceだけに限定する。自己循環を避けるため`binding_kind=editor_context`を禁止し、`authoring_context | architecture_explain | game_understanding | optimization_decision | ai_debug_context`だけを許す。keyboard focus、hover、range anchor、scroll anchor、Panel pinはauthoritative targetとして含めない。
- `game_understanding | optimization_decision`はcomplete Projectionだけを許す。query型の他bindingは省略範囲とcontinuationを保持し、取得済み範囲だけを説明する。Projection hash、Owner、Project／source revision、invalidation conditionの一件でも不一致ならSnapshot生成をfail closedにし、同kindの`latest`、文書断片、raw traceへ差し替えない。
- `optimization_decision`はTarget／Profile／Contract／Toolchain／fixture／input traceがPanelの対象と一致するread-only bindingだけを許す。AIはselected／rejected／not-evaluated状態とEvidenceを説明できるが、候補選択、threshold変更、Receipt補完、Project writeを`allowed_actions[]`へ生成しない。将来そのためのOperationが別途Activationされても、Snapshot自身はAuthorityにならない。
- AIは`element_key`とStable target refを説明・登録済みfocus／reveal request・Context選択に使用できるが、状態変更はCommand ID／typed `ProjectChangePrimitiveV1`で行う。
- `semantic_content_hash`、Project revision、selected Projectionの一つでも古いProposalを自動実行しない。staleは新しい`AiTaskContextCapsuleV1`を生成し、rebase後に全validatorを再実行するまでCommit不可である。
- Debug Panel選択時もraw trace、画面pixel、無制限event列をこのSnapshotへ埋め込まない。Debugging規約のbounded `AiDebugContextV1`を参照し、Session／Query／Evidence ID／recorded revision／gap／redactionを失わない。

画面の見た目を評価する必要がある場合だけ、Userが明示許可した次の別artifactを使う。

```text
VisualEvidenceSnapshotV1
  capture_id
  viewport_or_camera_ref
  capture_target_ref
  project_revision
  semantic_content_hash
  capture_purpose = visual_review | accessibility_review
  redaction_policy_ref
  redaction_result
```

`VisualEvidenceSnapshotV1.capture_target_ref`はcapture範囲の説明だけであり、Command target、authorization、Project write、selection identityの正本にならない。これにより、AI統合は画面操作macroではなくMiraikanaiのAuthoring契約となる。

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

#### 13.2.1 Windows system text scale

独自DirectWrite text surface、custom control、hard-coded row／control heightはWindowsのtext scalingを自動では受け取らない。Windows Adapterは`UISettings.TextScaleFactorChanged`を監視し、通知ごとに`UISettings.TextScaleFactor`を再読込する。Desktop WinRT API supportでは`UISettings.ColorValuesChanged`だけがunsupported eventとして列挙されるため、color通知へ代用せず`TextScaleFactorChanged`を独立に扱う。source値はfinite `float64`、許容範囲は`1.00 <= factor <= 2.25`であり、DPI、`editor_ui_scale`、font asset／`style_font_generation`、themeを別のaxisとして扱う。event通知、以前のscale、nearest profile、任意の丸め値から新factorを推測しない。C1は実際の`target.windows.editor` processでinitial readとevent subscriptionをprobeし、compile成功、OS build名、手入力Profileをsupportの根拠にしない。probeが失敗すれば`infrastructure_error`としてcaptureを開始せず、registry値、static 200%、別APIへの暗黙fallbackを使わない。

Adapterはfactor変化を同じ`window_metrics_changed` transactionへ直列化し、measure／arrange、text layout、glyph raster／atlas key、draw packet、UIA boundsとfocus navigationをfresh generationで作り直す。Project revision、Stable target、selection、Semantic action、Command authorizationを変更せず、古いtext layout、glyph、clip、UIA boundを再利用しない。必要contentが収まらない場合は§15.1のreflow／scroll／layout errorへ進み、fontを縮小して見かけだけ合わせない。

Editor text roleはC1でsource factorをそのまま適用する。`text_physical_px = text_em_lu × (effective_dpi / 96) × editor_ui_scale × system_text_scale_factor`とし、roleごとの隠れたscale curveを導入しない。Fluent iconはfont glyphではなく16／20 luのgeometryであり、system text factorを掛けない。text増大時はiconを拡大するのでなくcontainerをreflowし、iconのaccessible name、Command、hit targetを維持する。

Reference 01のE06（Font 200%）はvisual baselineとして維持するが、Windows supportの上限ではない。Platform Adapter conformanceは同じrunning Windowで`1/1 -> 9/4 -> 1/1`のsource changeを検証し、日本語／ASCII／code／numeric／path、wrap／end clip、keyboard／Narrator／UIA、Semantic target、glyph／layout generationが正しく更新されることを要求する。このprobeは新しいReference Manifest、Capability、baselineを作らず、E06または100% Screenshotを225%のEvidenceに読み替えない。

#### 13.2.2 Windows appearance change

C1のnon-contrast appearanceはWindows Color Modeへ追従し、`theme.editor.light@1`または`theme.editor.dark@1`を解決する。C1はeditor-localの「常にLight／Dark」overrideを持たず、LightをDarkの反転、OS accent、前回paletteから生成しない。両token profileは§15.1の別々のversioned Design System artifactであり、同じsemantic token名、state overlay順、typography、icon geometryを使う。High Contrastはこの二者より常に優先する。

Windows Adapterは`WM_THEMECHANGED`、`WM_SYSCOLORCHANGE`、`WM_SETTINGCHANGE`を「現在のappearanceを再読込する」通知として扱う。custom desktop appでは`UISettings.ColorValuesChanged`が公式にunsupportedであるため、これをsubscriptionまたはWin32通知欠落時のfallbackに使わない。一方で`UISettings.GetColorValue(UIColorType::Foreground)`は初期化時と各通知後のnon-contrast Color Mode readに使う。foregroundがlightかはMicrosoftのinteger式`(5*G + 2*R + B) > 8*128`で判定し、lightならDark mode、そうでなければLight modeとする。Window procedureは通知をUI session threadへqueueするだけでUiTree／draw packetを直接mutationせず、通知payload、前回snapshot、registry、theme表示名から色やmodeを推測しない。

初期化時およびqueueされた各通知後に、`HIGHCONTRASTW.cbSize=sizeof(HIGHCONTRASTW)`を設定した`SystemParametersInfoW(SPI_GETHIGHCONTRAST)`で`HIGHCONTRASTW.dwFlags & HCF_HIGHCONTRASTON`を再読込する。High Contrast時は`GetSysColor`から少なくとも`COLOR_WINDOW`／`COLOR_WINDOWTEXT`、`COLOR_HIGHLIGHT`／`COLOR_HIGHLIGHTTEXT`、`COLOR_GRAYTEXT`、`COLOR_HOTLIGHT`を採取し、non-contrast時はforeground colorと判定済み`light | dark`を採取する。`{High Contrast flag, non-contrast foreground/mode または required system-color vector}`を二回連続で読んでbyte-equalになることを一attemptとし、最大3 attemptで一つのstable snapshotとする。安定化しなければ`infrastructure_error`であり、old snapshotを採用しない。`WM_THEMECHANGED`で保持中のtheme handleがあれば無効化して再取得する。`WM_SETTINGCHANGE`はどの個別値が変わったかを信頼せず、使用するsystem setting全体を再読込する。

`HCF_HIGHCONTRASTON=false`ならsnapshotのmodeに一致するLightまたはDark token set、`true`ならimmutable system-color snapshotを解決する。`HIGHCONTRASTW.lpszDefaultScheme`、Aquatic等の表示名、既知palette、前回snapshotはidentityではない。snapshotはmodeまたはHigh Contrast flagと採取したcurrent color値のcontent hashを持ち、既存`theme_profile_ref`から参照する。初期read、通知subscription、再読込、必須color、またはmode判定のいずれかが失敗したReference／conformance captureは`infrastructure_error`として開始しない。static E07–E10、E13のstatic Light image、起動時だけのread、hard-coded palette、nearest theme／old snapshotへのfallbackでpassにしない。

stable snapshotがcurrent snapshotとbyte-equalならappearance event／generationを増やさない。異なる場合だけ`PlatformUiEventV1::appearance_changed`を発行し、一つのappearance transactionでtoken解決、contrast-only border／overlay、color-bearing draw packet、tinted icon instance、visual snapshotを新しいsnapshotへ切り替える。non-contrastのtop-level HWNDは同じtransactionで`DwmSetWindowAttribute(DWMWA_USE_IMMERSIVE_DARK_MODE)`へmodeと同じBOOL（Dark=`TRUE`、Light=`FALSE`）を渡し、clientとstandard title barを別themeにしない。C1 targetはWindows 11 25H2であり、このattributeのWindows 11 Build 22000以降という公式support条件を満たす。High Contrastへ入る際はapp固有のcaption／border／text color overrideを設定せず、standard non-client frameがsystem High Contrastを優先することをWindows 11 target image上でprobeする。このDWM×High Contrast挙動は推測で固定せず、locked OS image上のsuccessful API resultとnon-client captureをPlatform profileへ記録するまでC1 visual closureに使わない。DWM callまたはこのprobeが失敗すればそのcaptureは`infrastructure_error`であり、Light／Darkのclient screenshotで代用しない。theme固有のborder等でgeometryが変わる場合だけmeasure／arrangeとUIA boundsも同じtransactionで更新する。font asset、fallback resolution、icon geometryが不変なら`style_font_generation`とgrayscale glyph coverageを変えないが、旧snapshotのcolor-bearing packet／brush／visual baselineを再利用しない。Project revision、Stable target、selection、focus、validation／Proposal／Runtime state、Semantic action、AI contextの許可Action、Command authorizationはappearance changeで変更しない。

Platform Adapter conformanceは隔離したWindows test userで、同一running WindowをLight→Dark→Light→四つのbuilt-in Contrast Theme→少なくとも一つのuser-customized system color→Lightへ順に切り替える。各遷移で通知の実受信、再読込snapshot hash、Light／Dark token profileとnon-client mode、current system colorによる描画、状態を表すshape／icon／text label、Project／Semantic不変を検証する。Editor本体はこの検証のために`SPI_SETHIGHCONTRAST`、registry write、または利用者のtheme変更を発行しない。このruntime probeは新しいReference Manifest、Capability、E14、Reference 01 coverage entryを作らない。E13はstatic Light baselineであり、E07–E10またはE13のstatic imageをcustomized colorまたはruntime transitionのEvidenceに読み替えない。

#### 13.2.3 Windows client-area animation preference

C1はMicrosoftのuser animation preferenceを尊重し、custom client areaで`effective_motion`を決める唯一の値を`SystemParametersInfoW(SPI_GETCLIENTAREAANIMATION, 0, &enabled, 0)`とする。`enabled=TRUE`は`full`、`FALSE`は`reduced`である。`UISettings.AnimationsEnabled`は第二の判定値としてAND／ORせず、`SPI_GETCLIENTAREAANIMATION`と矛盾時に推測もしない。一方、Windows 10 version 2004以降の公式`UISettings.AnimationsEnabledChanged`はdesktop appでunsupportedと列挙されておらず、C1 targetのWindows 11 25H2では同じrefreshを開始する補助notificationとして必ずsubscriptionする。registry、前回値、app-localの「常にanimation」override、High Contrast、DPI、text scale、themeをmotion valueへ読み替えない。

```text
EditorMotionPreferenceSnapshotV1
  motion_preference_snapshot_id
  source_api = system_parameters_info_spi_getclientareaanimation
  client_area_animations_enabled: bool
  effective_motion = full | reduced
  motion_preference_snapshot_content_hash
```

Windows Adapterはprocess-scoped providerとしてfirst top-level HWND生成前に一つの`UISettings`を作成し、`AnimationsEnabledChanged`をsubscriptionしてからinitial `SPI_GETCLIENTAREAANIMATION` readを行う。いずれかのtop-level Windowが受けた全`WM_SETTINGCHANGE`または同eventは、一つのprocess-wide refreshをUI session threadへqueueする。callback payloadや`UISettings.AnimationsEnabled`値を直接採用せず、refresh時に必ず`SPI_GETCLIENTAREAANIMATION`を再読込する。refreshはMain／floating／owned top-level surfaceへ同じsnapshot generationを適用する。`WM_SETTINGCHANGE`の`wParam`／`lParam`はrefreshを省略する根拠にせず、Window procedure／WinRT callbackはUiTree／draw packetを直接mutationしない。`SPI_SETCLIENTAREAANIMATION`、registry write、Settings write、利用者のanimation設定変更をEditor本体から発行しない。initial readまたはevent subscriptionが失敗したlive runtimeは安全側の`reduced`へ縮退し、Reference／Platform Adapter conformance captureは`infrastructure_error`として比較を開始しない。前回snapshot、E11のstatic image、または任意の既定値をquery failureのpassへ使わない。

新snapshotのcontent hashがcurrentと同じならevent／generationを増やさない。異なる場合だけ`PlatformUiEventV1::motion_preference_changed`を発行し、一つのmotion transactionで処理する。`full -> reduced`では、次のpresentより前かつvirtual UI clockを進めずに、進行中またはqueue済みのvisual-only effectをcancelし、targetの最終geometry／opacity／visibilityへ解決する。対象はhover／pressed、scrollbar chrome、dock preview、panel／popover／tab transition、dock layout／workspace switch、indeterminate task、custom-drawn caret blinkである。semantic state、known-unit taskの実測値、Project revision、Stable target、selection、focus、Command authorization、Task／Receiptをpreference change自身で変更しない。indeterminate taskはcurrent stage textだけのstatic indicator、custom-drawn caretはvisibleなstatic caretとする。`reduced -> full`はcancel済みeffectをresume／replayせず、その後に新規に発火したeffectだけが§15.5のdurationを使える。

Platform Adapter conformanceは隔離したWindows test userで、Editorではない外部controllerが同一running Windowの設定を`TRUE -> FALSE -> TRUE`へ変える。`dock-preview`のfull tick 42、panel／tab transition、dock layoutまたはworkspace switch、indeterminate taskが各々in-flightの時点で`FALSE`へ切り替え、`AnimationsEnabledChanged`の実受信、`WM_SETTINGCHANGE`を受けた場合の同一refreshへのcoalesce、old／new snapshot hash、次presentでのfinal visual state、残留transform／opacity／clock-driven motionがないこと、preference change自身によるProject／Semantic／Command／Receiptの追加がないことを記録する。`TRUE`復帰後は新規triggerだけがfull durationへ戻ることも記録する。このruntime probeは新しいEnvironment Profile、E14、Reference 01 coverage entry、visual baselineを追加せず、E11のstatic reduced-motion baselineをrunning-process遷移のEvidenceへ読み替えない。

#### 13.2.4 Windows transparency effects

C1のmaterial policyはimmutableな`effects.editor.opaque-only@1`とする。`surface.canvas`／`workspace`／`panel`／`raised`／`input`、menu、popover、tooltip、dock previewを含む全interactive／content-bearing surfaceはopaque tokenをbaseにし、desktopまたは背後Windowを見せるper-pixel transparency、backdrop sampling、Mica／Mica Alt、Acrylic／Desktop Acrylic、Gaussian blurを使わない。C1は§13.1どおりstandard non-client frameを維持し、title barへcontentをextendしない。これは「透明効果がonなら必ずmaterialを使う」という意味ではない。MicaをWin32へ導入するにはWindows App SDK、WinRT、Compositor、DispatcherQueueという追加dependencyとpolicy handlingが必要であり、wallpaper／active stateへ依存するbase layerはlock済みC1 visual baselineへ含めない。

```text
EditorAdvancedEffectsPreferenceSnapshotV1
  advanced_effects_preference_snapshot_id
  source_api = uisettings_advancedeffectsenabled
  advanced_effects_enabled: bool
  c1_effective_material_policy = opaque_only
  advanced_effects_preference_snapshot_content_hash
```

Windows Adapterは同じprocess-scoped `UISettings`からinitial `AdvancedEffectsEnabled`をreadし、`AdvancedEffectsEnabledChanged`をsubscriptionする。eventはUI session threadへ一つの`advanced_effects_preference_changed` refreshをqueueし、refresh時にpropertyを再readする。`true`は将来のnamed materialを検討できる利用者permissionであってC1でMica／Acrylicを自動activateする命令ではなく、`false`は`opaque_only`を要求する。High Contrastは常にこのpermissionより優先し、system-colorによるopaque representationを使う。callback、registry、Settings write、app-local transparency overrideからmaterialを選ばず、snapshot変化はProject revision、Stable target、selection、focus、Semantic／UIA Action、Command authorization、Task／Receiptを変更しない。readまたはsubscription失敗時のlive runtimeは`opaque_only`を維持し、Reference／Platform Adapter conformance captureは`infrastructure_error`として比較を開始しない。

C1後にmaterialを提案する場合は、`advanced_effects_enabled=true`だけでactivateせず、別のapproved `advanced_effects_policy_ref`、Windows App SDK／Runtime等のDependency ChangeSet、opaque fallback token、High Contrast fallback、exact capture environment、same semantic／layout／UIA／Command Evidenceを先に閉じる。Micaは一Windowのlong-lived base layerだけ、Acrylicはlight-dismiss可能なtransient surfaceだけを候補にし、Panel、Inspector、Tree row、Property field、Canvas、persistent Dockへ重ねない。false／High Contrast／support probe失敗のどれでもopaque fallbackへ切り替え、contrast、hit target、focus、状態表示を弱めない。

Platform Adapter conformanceは隔離test userで同一running Windowの`AdvancedEffectsEnabled`を`TRUE -> FALSE -> TRUE`へ変え、event実受信、old／new snapshot hash、C1の`opaque_only` draw descriptor、Project／Semantic不変を記録する。C1にadvanced materialがないためstatic disabled Environment Profile、E14、Reference 01 coverage entry、visual baselineを追加しない。将来materialをactivateするChangeSetだけが、そのopaque fallbackを含む別profile／coverageを追加できる。

#### 13.2.5 Windows automatic scrollbar preference

C1はcustom collection／form／document viewportのscroll chromeについて、`UISettings.AutoHideScrollBars`を利用者設定の唯一の値とする。`true`は`auto_hide`、`false`は`persistent`である。このpropertyは特定Scrollbarの可視性ではなくcustom UI framework／custom scrollbarが尊重する設定であり、actual chromeはcontent extent、viewport、pointer proximity、focus、pointer captureからだけ決める。settingはlogical scrollability、scroll offset、selection、focus、Semantic／UIA Actionを変更しない。Canvas／Graph／Timelineのpan／orbitはScrollChromeではなくownerのviewport navigationであり、actual scrollbarを持つcompanion collectionだけが本契約を使う。

```text
EditorScrollBarPreferenceSnapshotV1
  scrollbar_preference_snapshot_id
  source_api = uisettings_autohidescrollbars
  auto_hide_scrollbars: bool
  effective_scroll_chrome_policy = auto_hide | persistent
  scrollbar_preference_snapshot_content_hash
```

Windows Adapterは既存process-scoped `UISettings`でfirst top-level HWND生成前に`AutoHideScrollBarsChanged`をsubscriptionし、initial readおよび同eventごとに`AutoHideScrollBars`を再読込する。このevent handlerは公式どおりworker threadから呼ばれるため、event args（stateを持たない）を採用せず、UI session threadへ一つの`PlatformUiEventV1::scrollbar_preference_changed` refreshをqueueするだけとする。callback、Window procedure、registry、`UISettingsController.SetAutoHideScrollBars`、Settings write、app-local「常に表示／常に非表示」overrideからpolicyを選ばず、UiTree／draw packetを直接mutationしない。

`ScrollChromeContractV1`はoverflowする軸だけにchromeを置く。`persistent`はoverflow中のfull track／thumb、`auto_hide`はpointerが`16 ui_lu`のreserved reveal gutterまたはchrome上にあるかthumb dragをcapture中なら`revealed`、pointerがviewport内またはscroll root内にkeyboard／accessibility focusがあるなら`indicator`、それ以外は`hidden`とする。content非overflowは常に`absent`である。reveal／hideにwall-clock idle timerを導入せず、`revealed`／`indicator`／`hidden`のvisual changeだけを§15.5の83 ms（reducedでは0 ms）で扱う。visibility changeはreserved gutter、logical extent／viewport／offset、layout／UIA boundsを変えず、active thumb drag中のpreference changeはpointer captureをcancelせずup／cancel後に新policyへ再評価する。

initial readまたはevent subscriptionが失敗したlive runtimeは安全側の`persistent`を使うが、Reference／Platform Adapter conformance captureは`infrastructure_error`として比較を開始しない。conformanceは隔離test userで外部controllerが同一running Windowを`FALSE -> TRUE -> FALSE`へ切り替え、worker callbackの実受信、UI session refresh、old／new snapshot hash、Tree／Table／Diagnostics／Sourceのscrollable harnessにおけるlogical offset／extent／selection／focus不変、`persistent`／`revealed`／`indicator`／`hidden` chrome、container UIA Scrollと必要時のScrollBar Control View、Project／Semantic／Command／Receipt不変を記録する。このruntime probeはE14、Reference 01 coverage entry、visual baselineを追加せず、E00–E13のstatic persistent screenshotをauto-hide transitionのEvidenceへ読み替えない。

#### 13.2.6 Windows message-duration preference

C1のin-window transient notificationは、`SystemParametersInfoW(SPI_GETMESSAGEDURATION, 0, &seconds, 0)`を利用者が読む時間の唯一の値とする。Microsoftは通知popupのdurationをhard-codeせずこの値に基づかせるため、`UISettings.MessageDuration`を第二のAND／OR判定、固定5秒、文字数推測、theme／motion／densityからのduration推測へ使わない。`seconds`はnotificationが初めてvisibleになった時点の`visible_since_tick`からauto-dismiss deadlineを決めるだけで、Project revision、Task実行、Validation、Proposal、selection、focus、Semantic Action、AI authorizationを変更しない。

```text
EditorMessageDurationPreferenceSnapshotV1
  message_duration_preference_snapshot_id
  source_api = system_parameters_info_spi_getmessageduration
  message_duration_seconds: uint32
  effective_auto_dismiss_policy = user_duration | manual_only
  message_duration_preference_snapshot_content_hash
```

`message_duration_seconds > 0`のsnapshotだけが`user_duration`を許す。`visible_since_tick + seconds`を同じ`EditorDeterministicUiClockProfileV1`のexact tickへ変換できない場合、またはvalueが0の場合、live runtimeは安全側の`manual_only`にし、Reference／Platform Adapter captureは`infrastructure_error`として比較を開始しない。通知本文の長さ、日本語／ASCII比率、severity、Task percent、pointer hover、OS既定値でdurationを短縮／延長／再開始しない。snapshot変更中にvisibleなauto-dismiss notificationは`visible_since_tick + new_seconds`へdeadlineを再計算し、new deadlineがcurrent tick以前なら次のdeterministic tickでpresentationだけをdismissする。

auto-dismissを許すのはactionなしの`information`またはownerが永続recordを持つ`action_completed`だけである。warning、error、blocking、Approval、security、source conflict、実行中Task、recovery actionを持つnotificationは`manual_only`であり、Problems、Task／Receipt、Approval等のowner surfaceへ同じtyped recordを残す。dismissはtransient chromeを閉じるpresentation actionであってDiagnostic／Task／Proposal／Receipt／Projectをresolve、acknowledge、cancel、approveしない。通知buttonは登録済みCommandを投影する場合だけButtonとして存在し、AIはbutton、UIA provider、activity ID、screen coordinateからAction権限を得ない。

Windows Adapterはfirst top-level HWND生成前にprocess-scoped `UISettings.MessageDurationChanged`をsubscriptionする。全top-level Windowの`WM_SETTINGCHANGE`と同eventは一つの`PlatformUiEventV1::message_duration_preference_changed` refreshをUI session threadへqueueするtriggerだけであり、callbackのthread affinity、event args、previous snapshotを値として採用しない。refreshで必ずSPI値を再読込し、Window procedure／WinRT callbackはUiTree、timer、draw packetを直接mutationしない。`SPI_SETMESSAGEDURATION`、registry／Settings write、app-local duration overrideをEditor本体から発行しない。initial readまたはevent subscription失敗時のlive runtimeは`manual_only`、Reference／Platform Adapter conformance captureはfail closedとする。

C1の`widget.feedback.notification@1`はin-window surfaceだけであり、Windows App SDK app notification／Toast、notification area icon、background activation、外部desktop popupを使わない。これらはpackage identity、elevation、activation、action protocol、separate dependency／policy／Evidenceを含むapproved ChangeSetなしに導入しない。Platform Adapter conformanceは隔離test userの外部accessibility controllerが同一running Windowのmessage durationを異なるpositive値`d1 -> d2 -> d1`へ変え、`WM_SETTINGCHANGE`と`MessageDurationChanged`の実受信、SPI re-read、old／new snapshot hash、auto-dismiss deadline、manual-only notification不変、UIA notificationのredaction／coalescing、Project／Semantic／Command／Receipt不変を記録する。このruntime probeはE14、Reference 01 coverage entry、visual baselineを追加せず、static screenshotをduration対応のEvidenceへ読み替えない。

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
  project_revision
  root_element_key
  semantic_content_hash
  layout_content_hash
  elements[]
  virtual_collections[]

EditorSemanticElementV1
  element_key
  parent_key
  role
  stable_target_ref
  canonical_field_ref?
  display_name_key
  display_text?                       # bounded、localization済み表示。identityではない
  description_key?
  typed_value?
  visual_state: EditorVisualStateV1
  actions[]
  bounds_physical_px
  privacy_class

EditorSemanticActionV1
  action_id
  command_id
  typed_argument_schema_ref
  enabled
  disabled_reason_key?
  risk_class
  approval_requirement
  expected_project_revision

EditorSemanticVirtualCollectionV1
  collection_key
  total_count
  realized_range
  omitted_count
  filter_ref?
  sort_ref?
  continuation?
```

`EditorSemanticActionV1`はcurrent `EditorCommandRegistry` recordとcurrent target／revision／availabilityから生成するsnapshot-scoped projectionであり、独立したSemantic Action Registryまたは永続的な権限recordではない。`action_id`はそのsnapshot内のopaque identityで、authorization、Approval、target、revisionを省略して別snapshotへ再利用しない。Platform Inputのsemantic-action binding registryをEditor ActionのRegistryとして流用・参照せず、Editor controlは必ずこのCommand recordからActionを投影する。

Semantic Snapshotは`EditorViewDescriptor`、read projection、committed Layoutから生成し、Draw Packet、color、glyph、hit-test結果を解析して作らない。`semantic_content_hash`はidentity、canonical target、typed value、state、action、collection queryだけを対象にし、bounds、palette、glyph、pixel、localized display textを含めない。`layout_content_hash`はcommitted boundsとfocus orderを検出する別hashであり、semantic hashの代用にしない。AIへ渡すprojectionは`bounds_physical_px`と`layout_content_hash`を必ず除外する。

Editor表示localeの変更は`display_name_key`から`display_text`、tooltip、accessible display textを再解決し、必要ならLayout generationを更新するpresentation changeである。同じProject revisionとread projectionに対してStable target、canonical field ref、typed value、state、Action集合、`semantic_content_hash`をbyte-identicalに保つ。UIAまたはAIがlocalized text、英語source text、翻訳順序からtarget、Field、Commandを逆算してはならない。Preferenceの解決、C1 locale、AI返答localeは[Editor Workspace／UX §7.3](editor-workspace-ux.md#73-editor表示localeとai返答locale)、Catalog／fallbackは[UI／Text／Localization／Accessibility §11](../07-platform/ui-text-localization-accessibility.md#11-localization)を正本とする。

virtualized Tree／Listは`total_count`だけ、Graphは見えているnodeだけを完全集合と偽装しない。AI、UIA、automationは`continuation`またはqueryを通じて必要な範囲を明示取得し、表示row index、画面座標、Panel内の偶然の順序をtarget identityに使わない。

### 14.2 Windows UI Automation

各top-level HWNDをUIA fragment rootとし、Windowless Widgetをfragment elementとして公開する。Providerは標準UIA control typeとpatternを優先する。

| MirakanUi control | 必須UIA pattern |
|---|---|
| command button／tool button | Button＋Invoke。icon-onlyもlocalized accessible name、tooltip、Command shortcutを持つ |
| menu／context menu／menu item | Menu／MenuItem。leafはInvoke、submenuだけExpandCollapse。Menu opened／closed eventを省略しない |
| toggle／checkbox | CheckBox＋Toggle |
| radio button | RadioButton＋SelectionItem。SelectionContainerを持ち、Toggleを公開しない |
| combo box | ComboBox＋ExpandCollapse。選択肢はList／ListItem、arbitrary textを許す場合だけValue。ComboBox自身へScrollを偽装しない |
| slider／spin field | SliderまたはSpinner＋RangeValue、rangeを正確に表せない／unboundedならValue |
| text input | Edit＋Text、TextEdit、Value。read-onlyはselection／copyを維持しmutation patternを公開しない |
| progress | ProgressBar。known-unitはread-only RangeValue、textual progressはread-only Value、indeterminateはstage textだけ |
| notification | noninteractive rootはGroupまたはStatusBar、登録済みactionだけButton。必要な一回限りのannouncementはUIA notification eventで別に送る |
| Source／Diff text | Text。`TextEdit`／Valueは`editability=editable`かつ対応Source Capabilityがactiveの場合だけ公開し、C1 read-only projectionではselection／copyを維持してmutation patternを公開しない |
| tree container | Tree。選択可能ならSelection、scroll可能ならScroll、virtualized itemを持つならItemContainer |
| tree item | TreeItem＋ExpandCollapse（leafはLeafNode state）。選択可能ならSelectionItem、scroll container内ならScrollItem、default actionがある場合だけInvoke。virtualized placeholderはVirtualizedItemだけを必須とする |
| asset collection view switcher | Pane＋MultipleView。current view childはlist／tileでList、columnsでDataGridとし、hidden view providerを残さない |
| flat list container | List＋Selection、scroll可能ならScroll、virtualized itemを持つならItemContainer。tile modeで空間navigationを公開する場合だけGrid |
| list row／asset tile | ListItem＋SelectionItem、scroll container内ならScrollItem、default actionがある場合だけInvoke。tile modeではGridItem、rename中だけValue。virtualized placeholderはVirtualizedItemだけを必須とする |
| table／DataGrid | 実際にrow／column／header関係を持つ場合だけDataGrid＋Grid／Table、選択可能ならSelection、scroll可能ならScroll、virtualized rowを持つならItemContainer。HeaderはHeader、sortable／resizable columnはHeaderItem＋Invoke／Transform、rowはDataItem＋SelectionItem／ScrollItem、read-only cellはDataItem＋GridItem／TableItem |
| diagnostics collection | List＋Selection、scroll可能ならScroll、virtualized itemを持つならItemContainer。新着通知用status regionはcollectionのchildにせず、別のStatusまたはGroupとしてLiveSettingを持つ |
| diagnostic／log item | ListItem＋SelectionItem、scroll container内ならScrollItem、default revealが登録済みの場合だけInvoke。virtualized placeholderはVirtualizedItemだけを必須とする |
| evidence gap row | 非選択のDataItem＋ScrollItem。欠損を正常itemへ偽装せず、回復Actionが登録されていない限りInvokeを公開しない |
| canvas surface | focus可能なPane。bounded UI design extentを実際にscrollする場合だけScrollを公開し、World cameraのpan／orbit／dollyへScrollまたはTransformを偽装しない。texture composite、Scene object、Game UI nodeをpixelからUIA childへ復元しない |
| canvas gizmo | Group＋ItemStatus／DescribedBy。visual handleはUIA TransformでProject objectを変更せず、ControllerForで登録済みTransform command、Inspector field、Outliner selectionへ結ぶ |
| graph surface | focus可能なPane。visual surfaceは現在realizeしたnodeだけをGroupとして公開し、pan／zoomをScroll／Transformへ偽装しない。全node／edgeの非空間代替は別のvirtualized List／DataGridとしてSelection、ItemContainer、VirtualizedItemを公開する |
| graph node／port／edge | nodeはGroup＋DescribedBy、portは接続Actionが登録済みの場合だけButton＋Invokeとする。edgeはpixel curveを独立elementにせず、source／destination port refをnodeの接続一覧へ公開する。UIA Transformでnode layoutまたはProject topologyを変更しない |
| timeline surface | focus可能なPaneとtrack用Tree／Listを合成する。有限rangeがUIAの値精度でexact表現できるpreview playheadだけSlider＋RangeValueを許し、それ以外はtyped time fieldのValueと登録済みseek Actionを使う。key／spanの非空間代替はvirtualized List／DataGridへ公開する |
| timeline key／span／playhead | key／spanはDataItem＋SelectionItem／ScrollItem、playheadはvariantに応じSliderまたはread-only Groupとする。UIA Transformでkey／spanを移動せず、ControllerForでexact time field、Inspector、registered move／retime Commandへ結ぶ |
| property group／row | group headerはGroup。row rootはUIA raw treeのstructural fragmentに留め、label Textと型に対応する標準child controlを公開。child controlはLabeledBy、必要時DescribedByを持つ |
| tab container／tab | Tab＋Selection／TabItem＋SelectionItem |
| docked／dockable Panel | Dock |
| floating top-level Panel Window | Window、Transform |
| drag source／drop target | Drag、DropTarget |

Providerはimmutable Semantic Snapshotを読み、UiNode pointerを保持しない。古いgenerationには`UIA_E_ELEMENTNOTAVAILABLE`を返す。ActionはUI threadへtyped Commandをqueueし、COM callback threadからProjectを変更しない。標準patternで表現できる場合はcustom UIA patternを登録しない。

- Secret／password ControlはUIA password propertyを設定し、Value、Text、name、descriptionへ内容を公開しない。
- UIA actionは`actor_kind=human`かつ`input_channel=accessibility`として、Human操作と同じRisk、Validation、Approvalを要求する。UIA clientの存在を追加権限として扱わない。
- virtualized Tree／List／Tableは`ItemContainer` queryと`VirtualizedItem` realizeを使い、UIA queryのために全rowをmaterializeしない。query／realize中にtree generation、filter、sort、Project revisionが変化した場合はold elementを再利用せず、fresh Snapshotから再取得させる。

### 14.3 AIとの分離

UIAはassistive technologyとblack-box UI testのWindows標準interfaceである。内蔵AI、MCP、CLIの正規操作経路には使用しない。AIは`EditorContextSnapshotV1`とtyped toolを使用し、UIA権限からAuthoring権限を得ない。

## 15. Design Systemと人間工学

詳細は[editor-ui-design-system-catalog](../appendices/editor-ui-design-system-catalog.md#15-design-systemと人間工学)へ分離した。本節はnavigationだけを持ち、Catalog／Fixture定義を複写しない。

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

Windows Reference hardware、2560×1440、100% DPI、Dark E00、standard density、`editor_ui_scale=1.00`、Production WorkspaceのReference 01を初期Performance計測条件とする。paired Light E13は同じvisual／semantic／layout／UIA／command profileとA hardware固有のinitial Performance subjectを持ち、E00のPerformance結果へ読み替えない。Reference hardware構成（CPU、RAM、storage、GPU、driver固定方針）の正本は[Performance／Capacity](../04-runtime/performance-capacity.md)であり、本書は構成値を定義しない。3840×2160・200% DPI、`editor_ui_scale=2.00`、High Contrast、multi-monitor moveは代替ではなく別fixtureで同じsemantic／commandを検証する。

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

`96 MiB`と`active UiNode 262,144`／`active semantic element 131,072`は、それぞれを別の楽観値として扱わない。C1前にbase tree、child／style／state、string・intern pool、semantic element、action、collection continuation、layout cache、snapshot leaseのbyte chargeを足し上げるcombined allocation modelを固定し、同時上限、fragmentation headroom、eviction順、failure thresholdをReference hardware上で測定する。このmodelと実測がない間、表はdesign ceilingであって、同時成立またはperformance qualificationのEvidenceではない。

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
- Widget Pattern ID／version／variant／required anatomy slot、state axis、forbidden state／Action rule、token refの整合
- Draw packet bounds、clip depth、primitive capacity、batch order preservation

### 19.2 Integration

- Main Window、floating Window、different-DPI multi-monitor、hot unplug
- Japanese／Chinese／Korean IME、emoji、BiDi、clipboard、Explorer file drop
- D3D12 device loss、Swap Chain resize、Font cache rebuild
- UIA Invoke、Value、Text、Selection、MultipleView、DataGrid／Header／HeaderItem、Grid／GridItem、Table／TableItem、Transform、ItemContainer／VirtualizedItem、Dock、Drag／DropTarget
- AI Contextからtyped Commandを実行し、screen coordinateを使用しないこと
- Manual edit、AI proposal、UIA action、Undoが同じCommand／ChangeSetへ収束すること
- semantic hashとlayout hashを別々に変化させ、boundsだけの変更がAI Proposalをtarget変更扱いにしないこと
- 同じfixtureを`en-US`、`ja-JP`、pseudo localeで投影し、display text／layout差があってもStable target、canonical field ref、typed value、Action集合、`semantic_content_hash`、AI typed Commandがbyte-identicalであること
- `fixture.cross-panel.attention@1`で五selection channel、Panel follow／pin、select／focus／reveal分離、exact relation、owner state projection、revision／generationを検証し、global untyped selection、Panel間callback、feedback loopを0件にすること
- duplicate／out-of-order／stale Attention Intent、target delete、filter／virtualization、Panel close後も一つのdeterministic snapshotへ収束し、Proposal／Runtime／recorded targetを暗黙retargetしないこと
- `EditorReferenceFixtureManifestV1`の同じlogical intentをhuman／pointer／keyboard／UIA／AI contract driverで実行し、typed barrier後のauthoritative／semantic／layout／visual／UIA／command oracleが同じcheckpointへ収束すること
- `fixture.editor.reference-01@1`がfixed Stable ID、revision 42 source、14 Environment Profile、9 scenario、input別driver、typed clock、七expected subject type、166 coverage entryへexact展開され、entry数、tuple、hashのmissing／extra／duplicateを拒否すること
- Reference 01のselectでStable target／selection／Inspector followは全driver一致、input固有focus／UIA eventはdriver別expected subjectへ保持し、focus差を消すnormalizationまたはselection差への読み替えを0件にすること
- required coverage tuple、Manifest／Registry／Candidate／Contract／Toolchain／Capability／Environment、Evidence Bundle／Verification Receiptがexact set／hash equalityを満たし、missing／extra／wrong profile／infrastructure errorをpassにしないこと
- visual baseline差、tolerance／mask変更、nonvisual oracle failureを個別検証し、current failing output、closest hardware baseline、AI説明からground truthを自動更新しないこと
- 100万Tree／10万Asset／50k Graph／10万Timeline itemで`total_count`、realized range、omitted range、continuationがUIA／AI／automationへ同じStable IDで投影されること
- `fixture.asset-browser.collection@1`でlist↔tile↔columns後もselection、focus target／column、range／scroll anchor、query revision、Project revisionが不変で、tile resize／DPIまたはtable width／visibility変更がAI Proposalをstaleにしないこと
- Asset Columnsでtyped column schema、six cell value state、最大3 sort、Stable ID tie-break、header Invoke／Transform、cell GridItem／TableItem、10万row virtualizationがmanual／keyboard／UIA／AIで同じStable target／columnへ収束すること
- Asset thumbnailのready／outdated／queued／running／failed／unavailableをAsset validationと分離し、Job完了順でsort、selection、GridItem identityを変えないこと
- `fixture.diagnostics.reference@1`でProblems／Console／Build／Profilerが同じcanonical Diagnostic／log identity、Target、revision、Evidence、gap／redactionを共有し、Task progress、metric sample、AI hypothesisをDiagnosticへ混在させないこと
- live Consoleでfollow-tail、anchor、新着件数、表示pause、表示clear、owner dedup、bounded copy／Export Taskがcanonical store／recordingを変更せず、UIA status通知が本文を連続announceしないこと
- `fixture.canvas.reference@1`でScene 2D／3D、Game、UI Designer、Map Previewが同じCanvas shellとowner別projectionへ解決し、navigation、pick、selection、gizmo preview、Game input routing、UI layout authorityがProject／Runtime境界を越えないこと
- CanvasのHuman pick／gizmoとAI typed Actionが同じStable target／expected revision／ChangeSetへ収束し、render texture、screen coordinate、pick buffer、UIA TransformからProject writeを生成しないこと
- `fixture.graph-timeline.reference@1`でGameplay／Animation／Topology／Render／Causality GraphとAnimation／Debug Timelineが共通surface＋owner別projectionへ解決し、layout、topology、exact time、Source／Runtime／recorded、gap、Capability activationを混同しないこと
- GraphのHuman connect／reconnectとAI typed Actionが同じStable node／port／edge、expected revision、一ChangeSetへ収束し、screen／layout座標、port色、UIA Transform、planned action名からProject writeを生成しないこと
- TimelineのHuman scrub／key moveとAI typed Actionが同じStable track／key／span、exact timebase tick、expected revisionへ収束し、display label／float second／screen xをidentityにせず、scrubからRuntime event／root motion／Project writeを生成しないこと
- Reference 01でOutliner、Scene、Inspectorの`authoring_target`とProblemsの`diagnostic_target` relationが一つの`TargetRef`／revisionへ収束し、Evidence selectionをObject selectionへ変換せず、selected、AI proposal、validation、Runtime、staleを個別axisとして同時検証できること
- Reference 01の全interactive Controlとsemantic composite rootがexact `pattern_id`へ一件解決し、同じinstanceからvisual anatomy、Semantic Element、UIA pattern、Command Actionが生成されること
- AIがPattern ID、表示label、UIA、screen coordinateから権限またはProject writeを導出せず、`EditorSemanticActionV1`とtyped Commandだけで状態を変更すること

### 19.3 Accessibility／UX

- keyboard-only、Narrator、NVDAでProject open、Scene選択、Inspector変更、Save、Play、Stop
- Accessibility Insightsでname、role、state、pattern、focus、bounds、event
- 100～250% DPI、High Contrast、200% Font、reduced motion
- drag代替、focus trapなし、Panel回収、shortcut remap
- Light／Dark／High Contrastでnormal、hover、pressed、focus、disabled、read-only、error、warning、stale、selected、AI proposal、runtimeを単独・重畳の両方で識別できること
- compact／standard／comfortableで同じCommand、focus order、semantic action、minimum hit targetが維持されること
- Collection selection、global attention、keyboard focus、reveal、Inspector pinを別semantic／eventとして公開し、High Contrastでもchannel chip、`固定`、selection、focus、staleをicon＋text／shapeで区別できること
- Diagnosticsでseverity、selection、focus、recorded／current、Runtime、stale、gap／redactionを色だけに依存せず、blocking／error新着を最大1秒単位のpolite statusとして読み上げること
- Canvasでsurface focus、active／multi-selection、XYZ／UI axis、validation、proposal、Runtime、stale、input routing、safe areaを色だけに依存せず、Outliner／Inspector／Commandによるdrag代替を完遂できること
- Graph／Timelineでnode／port／edge、track／key／span、preview／Runtime／recorded cursor、gapをshape＋labelで識別し、companion List／DataGrid、typed time field、Connect／Move Commandで全drag操作を代替できること

### 19.4 Performance／soak

- 100万Entity virtualized Outliner
- 10万Asset virtualized Browser
- 50,000 node Graphのviewport内materialization
- 100,000 key／span／event Timelineのtrack／time両軸virtualization
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
- Platform Adapter型をPublic Header、serialization、MCDへ公開すること
- Accessibilityを実装後の追加機能として扱うこと
- GPU device loss時にProject recovery手段を失うこと

## 22. 受入条件

C1独自Editor UI Frameworkは次をすべて満たした時点で完了する。

1. Editor binaryとSBOMに禁止GUI toolkitが含まれない。
2. MirakanUi CoreをEditorとGame UIが共有し、Shipping Game closureにEditor targetが含まれない。
3. Scene、Outliner、Inspector、Asset、Gameplay Definition Graph、Animation Graph、Topology Graph、Render Graph、Source、Session、Console、Problems、Profiler、Animation Timeline、Debug Timeline、Causality、Breakpoint／Watch、Replay、Reproduction、AI Partnerが独自Widgetで動作し、各Graph／TimelineはDomain ownerのtyped projectionだけを使う。
4. Dock、resize、tab reorder、floating、multi-monitor、複数Workspace保存と回復が成立する。
5. DirectWrite、TSF、UIAをAdapter内へ隔離し、UTF-8／IME／screen readerを満たす。
6. Human、keyboard、assistive technology、AIが同じtyped Command／ChangeSet／Validationを通る。
7. AIがWidget pointer、HWND、UIA、screen coordinateを使用せずゲームを作成・修正できる。
8. 全interactive Controlとsemantic composite rootがclosed Widget Pattern Registryへ解決し、visual、Semantic、UIA、Commandが同じtarget／revisionへ収束する。
9. DPI、High Contrast、200% Font、reduced motion、keyboard-only、Narrator、NVDA fixtureが合格する。
10. 100万Entity、10万Asset、50,000 node Graph、100,000 Timeline item、8時間soakでcapacity、memory、performance gateを満たす。
11. Device loss、corrupt Workspace、monitor消失、AI切断、TSF／UIA failureでProject正本を破損しない。
12. 五Referenceが`EditorAttentionSnapshotV1`、単一Reducer、typed relation、Panel follow／pinへ収束し、selection、focus、reveal、validation、Proposal、Runtime、recorded contextを混同しない。
13. 全Reference conformanceがversioned Manifest、explicit coverage tuple、typed barrier、七oracle、content-addressed Evidence Bundle、既存`VerificationReceiptV1`へ収束し、ScreenshotまたはAI説明だけでpassにならない。
14. `fixture.editor.reference-01@1`の14 Environment Profile、9 scenario、166 coverage entry、七typed expected subjectとA／B hardware／Toolchain／Command closureが同じContract setでmaterializeされ、未解決時にManifest／Registry row／Capability activationを仮生成しない。
15. 七oracleがexact七entryのComparison Profile Registryへ解決し、Visualの1 LSB／aggregate／contrast／空dynamic-region set、UIAの二Field normalization、PerformanceのA／B別absolute＋5% regression集計をRunner既定値なしで再現する。
16. 初回をbaseline非依存Execution Definition→166 capture→Definition Closureへ閉じ、Baseline RegistryをCoverage Matrixとexact 166 entryで一致させる。変更はatomic Change Item、immutable Review Batch、Itemごとのdomain owner＋independent reviewer二Receipt、24時間以内のfresh authorization、candidate全166再検証、`EditorReferenceBaselineHeadV1` CAS、Promotion read-back、append-only Change Registryへ閉じる。Review surfaceはGround Truth／Incoming／Differenceとtyped nonvisual diffを同時表示し、`Accept all`、画像click承認、AI decision、capture=`pass`、stale Receipt、partial initialize、hash cycleを許さない。
17. in-window notificationが`SPI_GETMESSAGEDURATION` snapshot、manual-only境界、owner record、Status bar一件／owner-inlineの表示経路、redaction済みUIA kind／processing／activity IDへ解決し、static E00–E13と`d1 -> d2 -> d1` runtime probeを混同しない。固定duration、`SPI_SETMESSAGEDURATION`、外部Toast、global floating stack、notification／dismiss／UIA／coordinate由来のAction・owner mutation、announce floodを許さない。
18. Widget、Panel、Command presentationの全Editor-owned system icon consumerが`EditorIconTokenContractV1`、Toolchain conversion manifest、16／20 lu output hashへ集合一致で解決し、source icon名・SVG・glyph・emojiをUI contractへ直接持たない。Project Asset thumbnail、ユーザー画像、Canvas contentは別image／thumbnail contractとして扱う。Regular／Filled、localized text／accessible name／tooltip、High Contrast、200% Font／UI scale、Semantic／UIAとAI typed Actionの分離を同じReference fixtureで検証する。

## 23. 一次資料と判断の対応

| 判断 | 一次資料 |
|---|---|
| 複雑なEditor UIにはRetained tree、Layout、debug toolingが必要 | [Unity UI Toolkit](https://docs.unity3d.com/Manual/UIElements.html)、[Unity UI systems comparison](https://docs.unity3d.com/Manual/UI-system-compare.html) |
| 独自C++ Editor UI FrameworkとDockingは成立する | [Unreal Engine Slate Overview](https://dev.epicgames.com/documentation/unreal-engine/slate-overview-for-unreal-engine?lang=en-US)、[Slate Widget Reflector](https://dev.epicgames.com/documentation/unreal-engine/using-the-slate-widget-reflector-in-unreal-engine?lang=en-US) |
| Engine自身のRenderer／UIでC++ Editorを構築できる | [Godot Introduction to editor development](https://docs.godotengine.org/en/stable/engine_details/editor/introduction_to_editor_development.html)、[Godot UI toolkit FAQ](https://docs.godotengine.org/en/latest/about/faq.html#what-user-interface-toolkit-does-godot-use) |
| Qtを利用するEngine Editorもあるが、本Projectはこの方式を選ばない | [O3DE Tools UI Developer's Guide](https://www.docs.o3de.org/docs/tools-ui/) |
| Diagnostics collectionは標準List／ListItem、Selection、virtualization、別Live regionで表現する | [Microsoft UI Automation Support for the List Control Type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supportlistcontroltype)、[Microsoft UI Automation Support for the ListItem Control Type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supportlistitemcontroltype)、[UI Automation Event Identifiers](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-event-ids)、[UI Automation Element Property Identifiers](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-automation-element-propids) |
| Console／Debuggerはfilter、選択詳細、source reveal、recorded contextを分ける | [Unity Console](https://docs.unity3d.com/6000.0/Documentation/Manual/Console.html)、[Unreal Engine Logging](https://dev.epicgames.com/documentation/unreal-engine/logging-in-unreal-engine?lang=en-US)、[Unreal Frontend Session Console](https://dev.epicgames.com/documentation/unreal-engine/using-the-unreal-frontend-tool?lang=en-US)、[Godot Debugger panel](https://docs.godotengine.org/en/stable/tutorials/scripting/debug/debugger_panel.html) |
| Canvasは共通navigation／tool shellとowner別Scene／Game／UI projectionを分ける | [Unity Scene view navigation](https://docs.unity3d.com/6000.0/Documentation/Manual/SceneViewNavigation.html)、[Unity Scene picking](https://docs.unity3d.com/6000.0/Documentation/Manual/ScenePicking.html)、[Unity Scene visibility](https://docs.unity3d.com/6000.0/Documentation/Manual/SceneVisibility.html)、[Unreal Viewport Controls](https://dev.epicgames.com/documentation/en-us/unreal-engine/viewport-controls-in-unreal-engine)、[Unreal Viewport Toolbar](https://dev.epicgames.com/documentation/en-us/unreal-engine/viewport-toolbar)、[Godot Introduction to 3D](https://docs.godotengine.org/en/stable/tutorials/3d/introduction_to_3d.html) |
| UI DesignerはHierarchy／Inspectorとexact layout、resolution／safe-area previewを組み合わせる | [Unity UI Builder](https://docs.unity3d.com/6000.0/Documentation/Manual/UIBuilder.html)、[Unreal UMG Safe Zones](https://dev.epicgames.com/documentation/unreal-engine/umg-safe-zones-in-unreal-engine)、[Godot UI](https://docs.godotengine.org/en/stable/tutorials/ui/)、[Godot Using Containers](https://docs.godotengine.org/en/stable/tutorials/ui/gui_containers.html) |
| Canvas accessibilityはPaneと標準child controlを使い、Project geometryをUIA Transformへ偽装しない | [Microsoft UI Automation Support for the Pane Control Type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supportpanecontroltype)、[UI Automation Control Types Overview](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-controltypesoverview) |
| Graphはnode／port／edge、breadcrumb、layout navigationを持つが、接続互換性とProject writeはDomain ownerへ残す | [Unity Animator window](https://docs.unity3d.com/6000.0/Documentation/Manual/AnimatorWindow.html)、[Unreal Blueprint Graph Editor](https://dev.epicgames.com/documentation/en-us/unreal-engine/graph-editor-for-the-blueprints-visual-scripting-editor-in-unreal-engine)、[Godot GraphEdit](https://docs.godotengine.org/en/stable/classes/class_graphedit.html) |
| Timelineはtrack list、time ruler、key／span、playhead、snap、preview／Runtime contextを分離する | [Unreal Sequencer Editor](https://dev.epicgames.com/documentation/en-us/unreal-engine/sequencer-cinematic-editor-unreal-engine)、[Unreal Editing Timelines](https://dev.epicgames.com/documentation/en-us/unreal-engine/editing-timelines-in-unreal-engine)、[Godot Introduction to animation features](https://docs.godotengine.org/en/stable/tutorials/animation/introduction.html) |
| Graph／Timeline accessibilityはvisual surfaceの全要素をpixelから列挙せず、standard controlとvirtualized companion collectionを併用する | [UI Automation control type／pattern mapping](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-controlpatternmapping)、[Working with Virtualized Items](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-workingwithvirtualizeditems)、[WCAG 2.2](https://www.w3.org/TR/WCAG22/) |
| Selection、keyboard focus、reveal、Inspector追従／固定を分離し、Cross-Panel同期をtyped targetへ閉じる | [Microsoft Selection and Focus](https://learn.microsoft.com/en-us/windows/win32/winauto/selection-and-focus-properties-and-methods)、[UI Automation Events Overview](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-eventsoverview)、[Godot EditorSelection](https://docs.godotengine.org/en/stable/classes/class_editorselection.html)、[Godot Inspector Dock](https://docs.godotengine.org/en/stable/tutorials/editor/inspector_dock.html)、[Unreal Engine Outliner](https://dev.epicgames.com/documentation/en-us/unreal-engine/outliner-in-unreal-engine)、[Unreal Editor Interface](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-editor-interface) |
| Reference fixtureはautomation result、Screenshot比較、UIA property／pattern／eventを別Evidenceとして取得する | [Unreal Automation Test Framework](https://dev.epicgames.com/documentation/en-us/unreal-engine/automation-test-framework-in-unreal-engine)、[Unreal Screenshot Comparison Tool](https://dev.epicgames.com/documentation/en-us/unreal-engine/screenshot-comparison-tool-in-unreal-engine)、[Using UI Automation for Automated Testing](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-usefortesting)、[Testing for accessibility](https://learn.microsoft.com/en-us/windows/win32/winauto/accessibility-testingtools) |
| Baseline reviewはGround Truth／Incoming／Differenceと差分navigationを同時に提供する | [Unreal Screenshot Comparison Tool](https://dev.epicgames.com/documentation/en-us/unreal-engine/screenshot-comparison-tool-in-unreal-engine)、[Unreal Engine Diff Tool](https://dev.epicgames.com/documentation/en-us/unreal-engine/ue-diff-tool-in-unreal-engine)。ただしUnrealのAlternative／closest運用は採用せず、Miraikanaiではexact Environment Profileと一Baseline Registry headへ閉じる |
| 複数Itemのreview中decisionと正式submitを分け、変更後のstale approvalを無効化する | [GitHub Reviewing proposed changes](https://docs.github.com/en/pull-requests/how-tos/review-pull-requests/reviewing-proposed-changes-in-a-pull-request)、[GitHub Approving a pull request with required reviews](https://docs.github.com/en/pull-requests/how-tos/review-pull-requests/approving-a-pull-request-with-required-reviews?apiVersion=2022-11-28)。ただしMiraikanaiはItemごとに二つの`ReviewReceiptV1`を発行し、Batch一括承認へ拡張しない |
| read-only textはText／TextRangeを保ち、IME／mutationはeditable textの別経路へ限定する | [UI Automation TextPattern Overview](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-ui-automation-textpattern-overview)、[About Text and TextRange Patterns](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-about-text-and-textrange-patterns)、[Text Services Framework](https://learn.microsoft.com/en-us/windows/win32/api/_tsf/) |
| Source Controlのstatus／history／revision Diffと外部変更のconflict解決は、現在Projectへの暗黙writeと分ける | [Unreal Engine Source Control](https://dev.epicgames.com/documentation/en-us/unreal-engine/source-control-in-unreal-engine)、[Unreal Engine Diff Tool](https://dev.epicgames.com/documentation/en-us/unreal-engine/ue-diff-tool-in-unreal-engine)、[GitHub Resolving a merge conflict](https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/resolving-a-merge-conflict-on-github)。Miraikanaiではこれをより厳格なowner-issued ref／revision／ChangeSet bindingへ落とす |
| Visual comparatorはexact sRGB surfaceをlinear評価し、意味のあるUI state／focusをcontrastとsemantic regionで別検査する | [DXGI color-space conversion](https://learn.microsoft.com/en-us/windows/win32/direct3ddxgi/converting-data-color-space)、[DXGI formats](https://learn.microsoft.com/en-us/windows/win32/api/dxgiformat/ne-dxgiformat-dxgi_format)、[WCAG 2.2 contrast minimum](https://www.w3.org/TR/WCAG22/#contrast-minimum)、[WCAG 2.2 non-text contrast](https://www.w3.org/WAI/WCAG22/Understanding/non-text-contrast.html) |
| UIA比較ではRuntimeIdを永続identityにせず、event kind／source／orderとFocus／Selectionの差を保持する | [`AutomationElement.GetRuntimeId`](https://learn.microsoft.com/en-us/dotnet/api/system.windows.automation.automationelement.getruntimeid?view=windowsdesktop-10.0)、[AutomationId property](https://learn.microsoft.com/en-us/dotnet/framework/ui-automation/use-the-automationid-property)、[Subscribing to UI Automation events](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-eventsforclients)、[Microsoft Selection and Focus](https://learn.microsoft.com/en-us/windows/win32/winauto/selection-and-focus-properties-and-methods) |
| Windows desktopはPer-Monitor V2を推奨し、DPI変更時にappが再配置する | [High DPI Desktop Application Development](https://learn.microsoft.com/en-us/windows/win32/hidpi/high-dpi-desktop-application-development-on-windows)、[`WM_DPICHANGED`](https://learn.microsoft.com/en-us/windows/win32/hidpi/wm-dpichanged) |
| DirectWrite layoutをcustom rendererへ渡し、grayscale bitmap glyphを生成できる | [Render Using a Custom Text Renderer](https://learn.microsoft.com/en-us/windows/win32/directwrite/how-to-implement-a-custom-text-renderer)、[`IDWriteBitmapRenderTarget1`](https://learn.microsoft.com/en-us/windows/win32/api/dwrite_1/nn-dwrite_1-idwritebitmaprendertarget1)、[DirectWrite Color Glyph Sample](https://learn.microsoft.com/en-us/samples/microsoft/windows-universal-samples/dwritecolorglyph/) |
| TSF applicationはThread ManagerとText Storeを実装する | [TSF Thread Manager](https://learn.microsoft.com/en-us/windows/win32/tsf/thread-manager)、[`ITextStoreACP`](https://learn.microsoft.com/en-us/windows/win32/api/textstor/nn-textstor-itextstoreacp) |
| Custom-drawn controlはUI Automation providerとpatternを実装する | [UI Automation Providers Overview](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-providersoverview)、[Interfaces for Providers](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-interfaces)、[Control Patterns](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-implementinguiautocontrolpatterns) |
| 標準Widget primitiveは独自patternでなく対応するUIA control type／patternへ投影する | [UIA control type／pattern mapping](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-controlpatternmapping)、[Button control type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supportbuttoncontroltype)、[Menu control type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supportmenucontroltype)、[ComboBox control type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supportcomboboxcontroltype)、[RadioButton control type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supportradiobuttoncontroltype)、[ProgressBar control type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supportprogressbarcontroltype)、[Thumb control type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supportthumbcontroltype)、[Tab control type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supporttabcontroltype) |
| Property Rowはvisual gridをAI／UIAの操作targetにせず、labelと標準editor Controlの関係を公開する | [Edit control type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supporteditcontroltype)、[Text control type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supporttextcontroltype)、[Automation element properties](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-automation-element-propids)、[Value control pattern](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-implementingvalue) |
| Tree Rowはhierarchy、focus、selection、expand stateを別semanticとして公開し、標準keyboard modelへ収束する | [Tree control type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supporttreecontroltype)、[TreeItem control type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supporttreeitemcontroltype)、[ExpandCollapse control pattern](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-implementingexpandcollapse)、[WAI-ARIA Tree View Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/treeview/) |
| Asset collection wrapperは同じ情報のlist／tile／columns表現を切り替え、current childだけを公開する | [MultipleView control pattern](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-implementingmultipleview)、[List control type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supportlistcontroltype)、[ListItem control type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supportlistitemcontroltype)、[WAI-ARIA Grid Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/grid/) |
| Asset columnsはheader／cellを持つ独立DataGridとし、sort／resize／keyboard focusを標準providerへ投影する | [DataGrid control type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supportdatagridcontroltype)、[DataItem control type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supportdataitemcontroltype)、[Header control type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supportheadercontroltype)、[HeaderItem control type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supportheaderitemcontroltype)、[Grid pattern](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-implementinggrid)、[Table pattern](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-implementingtable)、[Transform pattern](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-implementingtransform)、[WAI-ARIA Grid Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/grid/) |
| virtualized controlを全要素materializeせずUIAへ公開する | [ItemContainer control pattern](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-implementingitemcontainer)、[VirtualizedItem control pattern](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-implementingvirtualizeditem) |
| 主要EngineのAsset Browserはhierarchyとflat contentを分け、icon／tile／list等の複数Viewを提供する | [Unity 6 Project window](https://docs.unity3d.com/6000.0/Documentation/Manual/ProjectView.html)、[Unreal Engine Content Browser Settings](https://dev.epicgames.com/documentation/en-us/unreal-engine/content-browser-settings-in-unreal-engine)、[Godot FileSystemDock](https://docs.godotengine.org/en/stable/classes/class_filesystemdock.html) |
| Explorer連携drag/dropはOLE data transferを使用する | [Data Transfer Interfaces](https://learn.microsoft.com/en-us/windows/win32/com/data-transfer-interfaces)、[Shell Drag-and-Drop](https://learn.microsoft.com/en-us/windows/win32/shell/dragdrop) |
| ClipboardはUser commandへの応答として扱う | [About the Clipboard](https://learn.microsoft.com/en-us/windows/win32/dataxchg/about-the-clipboard)、[Clipboard Formats](https://learn.microsoft.com/en-us/windows/win32/dataxchg/clipboard-formats) |
| 独自DirectWrite／custom controlはWindows system text scaleを自動継承しないため、factor変更でtext layoutとcontainerをreflowし、font iconをtextと同率拡大しない | [Text scaling](https://learn.microsoft.com/en-us/windows/apps/develop/input/text-scaling)、[UISettings.TextScaleFactor](https://learn.microsoft.com/en-us/uwp/api/windows.ui.viewmanagement.uisettings.textscalefactor)、[UISettings.TextScaleFactorChanged](https://learn.microsoft.com/en-us/uwp/api/windows.ui.viewmanagement.uisettings.textscalefactorchanged) |
| Light／Dark、High Contrast、text、motion、transparencyを色やanimationだけに依存させず、running desktop processのtheme／system-color／client-area animation／advanced effects変更ではOSのcurrent valueを再読込する | [Support Dark and Light themes in Win32 apps](https://learn.microsoft.com/en-us/windows/apps/desktop/modernize/ui/apply-windows-themes)、[Contrast themes](https://learn.microsoft.com/en-us/windows/apps/design/accessibility/high-contrast-themes)、[High contrast parameter](https://learn.microsoft.com/en-us/windows/win32/winauto/high-contrast-parameter)、[`GetSysColor`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getsyscolor)、[`UISettings.GetColorValue`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.viewmanagement.uisettings.getcolorvalue)、[`DwmSetWindowAttribute`](https://learn.microsoft.com/en-us/windows/win32/api/dwmapi/nf-dwmapi-dwmsetwindowattribute)、[`SystemParametersInfoW`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-systemparametersinfow)、[`WM_THEMECHANGED`](https://learn.microsoft.com/en-us/windows/win32/winmsg/wm-themechanged)、[`WM_SYSCOLORCHANGE`](https://learn.microsoft.com/en-us/windows/win32/gdi/wm-syscolorchange)、[`WM_SETTINGCHANGE`](https://learn.microsoft.com/en-us/windows/win32/winmsg/wm-settingchange?view=windowsdesktop-10.0)、[Composition tailoring for WinUI apps](https://learn.microsoft.com/en-us/windows/apps/develop/composition/composition-tailoring)、[`UISettings.AnimationsEnabledChanged`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.viewmanagement.uisettings.animationsenabledchanged)、[`UISettings.AdvancedEffectsEnabled`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.viewmanagement.uisettings.advancedeffectsenabled)、[`UISettings.AdvancedEffectsEnabledChanged`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.viewmanagement.uisettings.advancedeffectsenabledchanged)、[Use Mica material in Win32 apps](https://learn.microsoft.com/en-us/windows/apps/desktop/modernize/ui/apply-mica-win32)、[Desktop WinRT API support](https://learn.microsoft.com/en-us/windows/apps/desktop/modernize/winrt-api-desktop-app-support)、[Accessible text requirements](https://learn.microsoft.com/en-us/windows/apps/design/accessibility/accessible-text-requirements)、[Timing and easing](https://learn.microsoft.com/en-us/windows/apps/design/motion/timing-and-easing) |
| custom scroll chromeはWindowsのauto-hide preferenceを再読込し、visible indicator／full thumbとUIA Scrollを別に扱う | [`UISettings.AutoHideScrollBars`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.viewmanagement.uisettings.autohidescrollbars)、[`UISettings.AutoHideScrollBarsChanged`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.viewmanagement.uisettings.autohidescrollbarschanged)、[Scroll viewer controls](https://learn.microsoft.com/en-us/windows/apps/develop/ui/controls/scroll-controls)、[Scroll control pattern](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-implementingscroll)、[ScrollBar control type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supportscrollbarcontroltype)、[Desktop WinRT API support](https://learn.microsoft.com/en-us/windows/apps/desktop/modernize/winrt-api-desktop-app-support) |
| in-window transient notificationはWindows message durationを再読込し、UIA notificationをredaction済みのcoalesced eventだけで発行する。短いactionなしfeedbackはStatus bar一件、状態／actionはowner-inlineとし、C1は外部App notification／Toastを使わない | [Message duration parameter](https://learn.microsoft.com/en-us/windows/win32/winauto/message-duration)、[`SystemParametersInfoW`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-systemparametersinfow)、[Accessibility parameters](https://learn.microsoft.com/en-us/windows/win32/winauto/accessibility-parameters)、[`WM_SETTINGCHANGE`](https://learn.microsoft.com/en-us/windows/win32/winmsg/wm-settingchange?view=windowsdesktop-10.0)、[`UISettings.MessageDurationChanged`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.viewmanagement.uisettings.messagedurationchanged)、[`UiaRaiseNotificationEvent`](https://learn.microsoft.com/en-us/windows/win32/api/uiautomationcoreapi/nf-uiautomationcoreapi-uiaraisenotificationevent)、[NotificationKind](https://learn.microsoft.com/en-us/windows/win32/api/uiautomationcore/ne-uiautomationcore-notificationkind)、[NotificationProcessing](https://learn.microsoft.com/en-us/windows/win32/api/uiautomationcore/ne-uiautomationcore-notificationprocessing)、[InfoBar](https://learn.microsoft.com/en-us/windows/apps/develop/ui/controls/infobar)、[App notifications overview](https://learn.microsoft.com/en-us/windows/apps/develop/notifications/app-notifications/) |
| Windows typography、CJK mono fontとsystem iconをHost／bundled assetへ分け、iconは小サイズでも意味が明確なcommand／navigation用途に限定し、label／accessible nameとsemantic tokenからlock済みassetへ解決する | [Windows Typography](https://learn.microsoft.com/en-us/windows/apps/design/signature-experiences/typography)、[International fonts](https://learn.microsoft.com/en-us/windows/apps/design/globalizing/loc-international-fonts)、[Icons in Windows apps](https://learn.microsoft.com/en-us/windows/apps/develop/ui/controls/icons)、[Noto Sans CJK 2.004](https://github.com/notofonts/noto-cjk/releases/tag/Sans2.004)、[Fluent UI System Icons 1.1.334](https://github.com/microsoft/fluentui-system-icons/tree/1.1.334) |
| Keyboard、Focus、target size、drag代替の人間工学原則 | [WCAG 2.2](https://www.w3.org/TR/WCAG22/) |

外部資料はPlatform APIと既存Engineの方式を示す。`MirakanUi Core`、`EditorViewDescriptor`、`EditorSemanticSnapshotV1`、AI／UIA分離、Command収束、UI固有contractとfixtureはMiraikanai Engine独自の規範決定である。
