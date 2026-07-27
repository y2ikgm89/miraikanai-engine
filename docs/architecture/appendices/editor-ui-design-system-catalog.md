# Editor UI Design System Catalog

- 文書ID: mirakan.appendix.editor-ui-design-system-catalog
- 文書種別: Owner supplement
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 親Owner: [Editor UI Framework](../03-authoring/editor-ui-framework.md)
- 正本範囲: Widget Pattern、visual token、semantic・UIA mapping、Reference Fixtureの候補Catalog
- 非正本範囲: 親Ownerが所有する安定Architecture原則、実装Task、実装順序、生成済みArtifactまたはQualification結果
- 規範依存: [親Owner](../03-authoring/editor-ui-framework.md)
- 関連文書: [Architecture Governance](../01-governance/architecture-governance.md)
- 根拠区分: project-decision／provisional。実ArtifactがないRegistry、Catalog、Fixtureは候補
- 外部根拠確認日: 2026-07-27

> この補助文書の型、Registry、Catalog、Fixtureは、対応するRepository Artifactが存在しない限り未実装の設計候補である。親Ownerの安定原則や実装済み状態を上書きしない。
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
- visual-state axis／overlay

<a id="151-dark-baselinescalehigh-contrast"></a>

### 15.1 Dark／Light baseline、scale、High Contrast

`theme.editor.dark@1`と`theme.editor.light@1`はC1のnon-contrast token profileであり、§13.2.2のcurrent system snapshotだけが選択する。色名でなく次のsemantic tokenを使用し、同じtoken keyを二profile間で欠落・別意味・別state overlayへ変えない。通常text／status text・iconは各`surface.panel`に対して最小4.5:1を満たす。disabledを任意opacityで薄くしてこの下限を壊さない。

#### Dark token profile `theme.editor.dark@1`

`surface.panel = #171E29`に対し、`text.primary`は14.21:1、`text.secondary`は9.28:1、`text.muted`は5.58:1、`text.disabled`は4.52:1、selection textは`selection.active`に対して5.22:1を満たす。Canvas axisは各row記載どおり`surface.canvas`に対して検査する。

| Token | 値 | 用途 |
|---|---|---|
| `surface.canvas` | `#0C1016` | Scene／Graph／Timelineの背景 |
| `surface.workspace` | `#111720` | Shell、Dock背景 |
| `surface.panel` | `#171E29` | Panel、List、Inspector |
| `surface.raised` | `#1D2633` | hover、menu、card |
| `surface.input` | `#101722` | editable field、pressed input |
| `border.subtle` | `#354153` | group、grid、inactive separator |
| `border.strong` | `#586984` | hover、active separator |
| `text.primary` | `#E6EDF7` | primary label／value |
| `text.secondary` | `#B7C2D0` | supporting label |
| `text.muted` | `#8B96A5` | metadata、empty state |
| `text.disabled` | `#798697` | disabled label／icon |
| `selection.active` | `#2D6D9B` | selected row／node background |
| `selection.text` | `#F4F8FC` | selected text |
| `focus` | `#75BCFF` | focus ring |
| `validation.error` | `#FF8580` | error rail／icon |
| `validation.warning` | `#F0C75E` | warning rail／icon |
| `validation.success` | `#66D28E` | successful validation only |
| `ai.proposal` | `#D29BFF` | AI proposal outline／badge |
| `runtime` | `#55D6C2` | Runtime context badge |
| `freshness.stale` | `#AAB4C3` | stale outline／clock |
| `canvas.axis.x` | `#FF6B6B` | X axis。`surface.canvas`に対して6.87:1 |
| `canvas.axis.y` | `#67D391` | Y axis。`surface.canvas`に対して10.26:1 |
| `canvas.axis.z` | `#70A7FF` | Z axis。`surface.canvas`に対して7.86:1 |
| `canvas.axis.ui` | `#F4C95D` | UI anchor／screen axis。`surface.canvas`に対して12.13:1 |

`theme.editor.light@1`はDark tableの値を反転またはbrightness変換して作らない。通常text／status text・iconは`surface.panel = #FFFFFF`に対して最小4.5:1、`text.primary`は16.27:1、`text.secondary`は8.71:1、`text.muted`は5.83:1、`text.disabled`は5.45:1、selection textは`selection.active`に対して11.56:1を満たす。Canvas axisは各row記載どおり`surface.canvas`に対して検査する。

| Token | 値 | 用途 |
|---|---|---|
| `surface.canvas` | `#F5F7FA` | Scene／Graph／Timelineの背景 |
| `surface.workspace` | `#EEF2F6` | Shell、Dock背景 |
| `surface.panel` | `#FFFFFF` | Panel、List、Inspector |
| `surface.raised` | `#F4F7FA` | hover、menu、card |
| `surface.input` | `#FFFFFF` | editable field、pressed input |
| `border.subtle` | `#C5CFDA` | group、grid、inactive separator |
| `border.strong` | `#7890AB` | hover、active separator |
| `text.primary` | `#172033` | primary label／value |
| `text.secondary` | `#3E4C61` | supporting label |
| `text.muted` | `#5B6675` | metadata、empty state |
| `text.disabled` | `#5D6B7B` | disabled label／icon |
| `selection.active` | `#CDE8FF` | selected row／node background |
| `selection.text` | `#102A43` | selected text |
| `focus` | `#005FB8` | focus ring |
| `validation.error` | `#B42318` | error rail／icon |
| `validation.warning` | `#7A5300` | warning rail／icon |
| `validation.success` | `#19723E` | successful validation only |
| `ai.proposal` | `#6E3D9A` | AI proposal outline／badge |
| `runtime` | `#006C5B` | Runtime context badge |
| `freshness.stale` | `#4B5D70` | stale outline／clock |
| `canvas.axis.x` | `#B42318` | X axis。`surface.canvas`に対して6.13:1 |
| `canvas.axis.y` | `#196C3B` | Y axis。`surface.canvas`に対して6.02:1 |
| `canvas.axis.z` | `#1D4ED8` | Z axis。`surface.canvas`に対して6.24:1 |
| `canvas.axis.ui` | `#795600` | UI anchor／screen axis。`surface.canvas`に対して6.22:1 |

Editor layoutは`ui_lu`で保持し、最終physical pixelだけを丸める。Editor固有のscaleは`editor_ui_scale`（0.75–2.00、既定1.00）とし、chromeの`physical_px = ui_lu × (effective_dpi / 96) × editor_ui_scale`で求める。`effective_dpi`は13.2のcurrent Window metrics transactionで確定したDPIである。Windows system text scaleは別axisであり、text emだけは§13.2.1の`system_text_scale_factor`をさらに掛ける。OS DPI、Windows text scale、Editor settingを同じ「scale」Fieldへ混在させない。densityのrow／field高さはchromeのminimumであってtext clipの上限ではない。scale後にrequired contentが入らない場合はtab化、reflow、scroll、layout errorの順で処理し、clipやoff-screen controlを成功扱いにしない。

High Contrastでは上表二profileのcolor tokenをWindows system colorへ置換する。基本surface／通常textは`COLOR_WINDOW`／`COLOR_WINDOWTEXT`、selected／hover／pressed／in-progressとfocus ring／selection outlineは`COLOR_HIGHLIGHT`／`COLOR_HIGHLIGHTTEXT`、disabledは`COLOR_GRAYTEXT`を`COLOR_WINDOW`上で、linkだけは`COLOR_HOTLIGHT`を`COLOR_WINDOW`上で使う。error／warningのrail、AI proposalのdash、Runtimeのbadge text、staleのdotted border／clock、Canvas axisのX／Y／Z／UI文字と異なるhandle形状は`COLOR_WINDOWTEXT`／`COLOR_WINDOW`を基本に残すが、non-contrast accent値をHigh Contrastへ持ち込まない。色、形状、icon、短いtext labelのうち最低二つで状態を表し、Light／Dark themeの色差だけに依存しない。Windowsの四つのbuilt-in Contrast Themeすべてとuser-customized system colorで同じsemantic token、focus、selected row、transient surfaceを検証し、特定themeの配色をhard-codeしない。

### 15.2 Typographyとicon

| Role | Font／rule |
|---|---|
| UI label、Japanese prose | Windows 11 25H2のsystem font `Segoe UI Variable`をLatin第一選択（Regular 400、title／group labelだけSemibold 600）、Japanese glyphは`Yu Gothic UI`へfallbackする。system fontはbundleせず、labelはLocalization keyから解決し、code identifierを翻訳しない |
| code、log、ID、path、diff | bundled mono faceはstatic OTFの`Noto Sans Mono CJK JP` 2.004、Regular／Boldだけとする。CFF2 variable fontをEditorで採用しない。exact archive／file hash、license、coverage、bundle manifestは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)でlockするまでC1 visual closureを主張しない |
| number、time、memory、progress | Japanese prose中の数値はUI face、editable numeric field・table column・timeline tick・Profiler値はmono face＋tabular figuresを使う。表示桁を固定し、unitはtyped valueのpresentationでありparse対象の文字列にしない |
| mixed Japanese／ASCII | Japanese sentenceはUI role、identifier／literal／file pathはcode roleに分ける。一つのidentifierの途中でfont fallbackを意味づけに使わず、同一行に混在してもfont size、baseline、copy／pasteのsemanticを変えない |

Iconは`Fluent UI System Icons 1.1.333`、tag commit `1953430cd73f928f3e43997e17a9d058b00d17b8`をbaselineとし、通常はRegular、selected／activeだけはFilledを使う。Tree／field内は16 lu、toolbar／primary actionは20 luを使い、同一semantic tokenで形状を変えない。exact source hash、license、16／20 luのbuild-time conversion output hashをToolchain lockへ固定する。runtime SVG／CSS parserは使わず、承認済みSVGをbuild時にbounded vector pathとDPI bucketごとのraster atlasへ変換する。Fluent geometryはfont-based iconではないため§13.2.1のWindows system text scaleを継承せず、DPI／`editor_ui_scale`だけでphysical sizeを決める。未知、曖昧、emojiのiconをprimary commandに使わない。

#### Icon semantic token contract

`EditorIconTokenContractV1`は新しいtop-level Registry、Capability、Toolchain lockではない。既存Widget Pattern RegistryのContract setからcompileするversionedな派生tableであり、実際のsource icon名をFramework、Workspace、Command artifactへ漏らさない。`icon_token_id`は`icon.<semantic-category>.<lower-kebab-token-key>@1`、実際の意味は`semantic_subject_ref`だけが持つ。token keyはstable nonlocalizedであり、localized label、表示順、Fluent source icon名、glyph path、colorを入れない。

| `semantic_category` | `semantic_subject_ref`のexact型 |
|---|---|
| `command` | `EditorCommandId` |
| `panel` | `panel_type_id` |
| `object_kind` | owner-issued object-kind ID |
| `state` | 既存visual state axisまたはowner status contractのclosed state key |
| `disclosure` | `expand`または`collapse` |

```text
EditorIconTokenContractV1
  icon_token_id
  semantic_category
  semantic_subject_ref
  allowed_consumer_refs[]          # sorted unique IconConsumerRefV1.consumer_ref
  allowed_logical_sizes_lu[]       # nonempty subset of { 16, 20 }
  normal_variant                   # Regular
  selected_or_active_variant?      # Filled only when the same token has selected or active overlay
  requires_visible_text_or_accessible_name
  requires_source_binding          # true

IconConsumerRefV1
  consumer_ref                     # widget:<pattern_id>:<anatomy_slot_id> | panel:<panel_type_id> | command_presentation:<presentation_id>
  icon_token_ref                   # exact EditorIconTokenContractV1 ref
```

Widget Patternの各`icon_consumer_refs[]`は`widget:<pattern_id>:<anatomy_slot_id>`、Panel descriptorの`icon_token_ref`は`panel:<panel_type_id>`、Command presentationの`presentation_icon_token_ref`は`command_presentation:<presentation_id>`へcanonicalに展開する。各Contract setでは、その三consumer ref集合、`EditorIconTokenContractV1.allowed_consumer_refs[]`の和集合、Toolchain generated conversion manifestのconsumer ref集合がbyte-exactに一致しなければならない。`command_presentation` consumerはmanifestに記録したusage contextが対応`presentation_context`とexactに一致する場合だけ有効とする。unknown、unused、重複consumer、source icon名を直接持つconsumer、tokenを持たないvisible Editor-owned system iconは拒否する。Project Asset thumbnail、ユーザー画像、Canvas content、外部tool previewは`image`／thumbnail contractのcontentであり、本icon contractのconsumerへ偽装せず、primary Commandやstatus意味を代用しない。source icon ID、Regular／Filled source variant、usage context、input／output hashはToolchainのmanifestだけが所有し、同じsource geometryを複数tokenで使う場合もtokenごとの明示bindingを要求する。

Filledはselectedまたはactive overlayに限り、hover、pressed、focus、disabled、read-only、validation、proposal、runtime、staleをFilledまたはicon colorだけで表現しない。command iconはvisible localized label、またはaccessible nameとtooltipを必須にし、state iconは同じsemantic nodeのlocalized short textまたはDescribedByを必須にする。icon tokenはPanel identity、target identity、Action、shortcut、authority、risk、AI permissionを代用せず、AIはtokenでなく`EditorSemanticActionV1`のtyped actionを読む。

### 15.3 Visual state contract

`visual_state`は一つのenumではない。以下のaxisを直交して保持し、Styleは固定したoverlay順で解決する。`runtime`はprovenanceではなくauthority layer、AI proposalは`authority_layer=proposal`と`provenance=ai`の組合せであり、`read_only`は`disabled`ではない。

| Axis | Closed values |
|---|---|
| interaction | `normal／hover／pressed／focused`。visual focus ringはinput modalityにより`focus_visible`のときだけ表示 |
| availability | `enabled／disabled` |
| editability | `editable／read_only／locked` |
| selection | `unselected／selected` |
| value | `set／mixed／unset` |
| authority layer | `source／draft／proposal／derived／runtime` |
| provenance | `human／ai／engine／importer／external_tool` |
| validation | `unknown／valid／info／warning／error` |
| freshness | `fresh／stale／superseded` |
| approval | `not_required／pending／approved／rejected／expired` |
| task | `idle／queued／running／blocked／failed／succeeded／cancelled` |
| live edit | `not_applicable／allowed／preview_only／restart_required／runtime_read_only` |

| State／overlay | 見た目と操作規則 |
|---|---|
| normal | panel／inputのbase token。意味を持たない装飾badgeを置かない |
| hover | `surface.raised`または`border.strong`を83 msで適用。hoverだけでselectionを解除しない |
| pressed | `surface.input`と1 lu inset keyline。pointer release／cancelで即時復帰し、Commit成功を示さない |
| focus | `focus`の2 lu outer ringと1 lu inner keyline。focus visibleはkeyboard／accessibilityを優先し、selectionやerror railに隠さない |
| disabled | `text.disabled`、no pointer／keyboard action、disabled reasonをSemantic actionへ持つ。read-onlyと同じlock iconを使わない |
| read-only／locked | read-onlyはcopy・focus・semantic readを許し、`surface.input`＋lock／`Read-only` labelでwriteを拒否する。lockedはlock owner／reasonを表示し、解放Commandがない限りeditしない |
| error | `validation.error`の2 lu left rail、circle-X icon、inline message。valueをwarning色だけで塗り替えない |
| warning | `validation.warning`の2 lu left rail、triangle icon、inline message。errorと同じblocking扱いにしない |
| stale | `freshness.stale`のdotted border、clock、base/current revisionを表示。Commitはdisabled、rebase／discardだけを提示する |
| selected | `selection.active` fill＋outline。validation、proposal、runtimeのbadge／railを消さない |
| AI proposal | `ai.proposal`のdashed outline、`AI提案` badge、before→after Diff。validationがvalidでもapprovalなしにCommit可能へ見せない |
| runtime | `runtime`のpersistent context badge（`Runtime`＋play glyph）。Source／Draftと同じfieldに見せず、live edit policyを併記する |

overlayはbase surface → selection fill → validation rail／stale border → authority・provenance badge → interaction → focus ring → High Contrast system-color replacementの順で合成する。`unknown` validation、`mixed` value、`derived`／`superseded`はvalid・editable・freshへ暗黙変換しない。tooltipは補足だけで、state label、reason、revision、approvalを置換しない。

### 15.4 Density

`compact`、`standard`、`comfortable`は同じControl／Command／Semanticを使う表示profileであり、別Widget実装にしない。値は`editor_ui_scale=1.00`の`ui_lu`である。toolbar／primary action hit target、focus ring、window最小値は[Editor Workspace UX §12.2](../03-authoring/editor-workspace-ux.md#122-sizeとdpi)を正本として消費する。

| Profile | Body size／line | Tree／List row | Field | horizontal padding | group gap | icon |
|---|---:|---:|---:|---:|---:|---:|
| compact | 13／18 | 24 | 24 | 6 | 4 | 16 |
| standard | 14／20 | 28 | 28 | 8 | 6 | 16 |
| comfortable | 15／22 | 32 | 32 | 10 | 8 | 20 |

### 15.5 Motion

motionは状態を隠すために使わず、durationは次のclosed setだけを使う。easingは`cubic-bezier(0.2, 0, 0, 1)`、spring、無限loop、layoutを伴う飾りanimationを持たない。

| 対象 | duration | 規則 |
|---|---:|---|
| validation、selection、approval、stale、runtimeなどのsemantic state | 0 ms | stateを即時に識別可能にする |
| hover、pressed、scrollbar chrome reveal／hide、transient notification entrance／exit、dock preview fade | 83 ms | interactionの反応だけ。drop先identityまたはnotification authorityを変えない |
| panel／popover、tab content | 167 ms | opacity／bounded transformだけ。inputを待たせない |
| dock layout、workspace switch | 最大250 ms | final geometryを先に確定し、途中に別のdrop targetを作らない |
| known-unit task progress | 実測更新時だけ | 受け取ったcompleted／total以外のpercentを作らない |
| indeterminate task | 1.2 s | current stage textを常時表示し、完了量を暗示しない |

`EditorMotionPreferenceSnapshotV1.effective_motion=reduced`の間は、semantic state以外のdurationを0 msへ縮退する。dock／panel／tab／workspaceは最終位置、dock previewは最終表示、scrollbar chromeは§13.2.5のcurrent `persistent／revealed／indicator／hidden` state、transient notificationは§13.2.6のcurrent visible／dismissed presentation、indeterminate taskはstatic stage indicator、custom-drawn caretはstatic visibleとして表示する。known-unit task progressは受信した実測値の更新だけを表示し、擬似percentや遅延した残留animationを作らない。これはapp-local settingで解除できず、`full`へ戻ってもcancel済みeffectを再生しない。Taskをmodalでblockせず、CancelはReceiptで確定するまで`cancel_requested`として示す。

共通規則として、Engine validation、AI proposal、User selection、Runtime stateを同じ色／iconだけで区別しない。Font／UI scale 200%、High Contrast、reduced motionで全C1 flowを操作可能にし、Drag-only操作にはCommand、keyboard、context menuの代替を持たせる。

### 15.6 Widget Pattern Contract

Widget Pattern Contractは、§9のControl capability、§14のSemantic Tree／UIA、§15.1～15.5のvisual tokenを、Editorで再利用する一つの規範単位へ束縛する。目的はAIにWidgetをclickさせることではなく、人間が見る対象・状態・操作可能性と、AIが`EditorContextSnapshotV1`から読む対象・状態・typed actionを一致させることである。AI、MCP、CLIはPatternをscreen coordinate、Widget pointer、UIA provider、表示labelから操作せず、実際の状態変更は§10のCommand Registryと`ProjectChangePrimitiveV1`を通す。

Patternは次の三層だけを持つ。

| layer | 責務 | 禁止 |
|---|---|---|
| `primitive` | Button、Field、Checkbox、Tab等の単独Controlのanatomy、input、focus、semantic role | Panel固有状態、Project write、任意callback |
| `composite` | Tree Row、Property Row、Diagnostic Row、Proposal Card等の登録済みPrimitive合成 | child Patternの暗黙上書き、表示順をidentityにすること |
| `surface` | Canvas、Graph、Timeline、Source等のviewport／collection rootとvirtual query境界 | Reference Workspace、Capability activation、Project正本の所有 |

Reference WorkspaceとPanel内のPattern構成は[Editor Panel／Reference Catalog §6.0](editor-panel-reference-catalog.md#60-reference-design)が所有し、同じ寸法、状態、Semantic規則を再定義しない。

```text
WidgetPatternContractV1
  pattern_id                         # widget.<family>.<lower-kebab-name>@1
  composition_level                 # primitive / composite / surface
  control_kind
  anatomy_slots[]
  allowed_variant_ids[]
  child_pattern_refs[]
  supported_state_axes[]
  forbidden_state_rules[]
  content_roles[]
  typography_token_refs[]
  spacing_size_token_refs[]
  icon_consumer_refs[]              # exact IconConsumerRefV1; one Widget anatomy slot per entry
  overflow_policy
  scroll_chrome_contract_ref?
  input_contract_ref
  focus_contract
  semantic_role
  stable_target_binding_kind
  canonical_field_binding_kind?
  uia_pattern_refs[]
  semantic_action_projection_rule
  motion_token_refs[]
  conformance_scenario_refs[]
```

- `pattern_id`はlogical identityであり、localized label、Control class、Panel type、配列indexを代用にしない。同じID／versionの別内容、unknown Field、未登録variant、missing required anatomy slotを拒否する。
- `anatomy_slots[]`はslot ID、required／optional、content role、最大行数、ellipsis／wrap／scroll、baseline、accessible name sourceを閉じる。装飾slotはtarget identityまたはActionを持たない。
- 寸法、色、font、icon、motionは本節のtoken refだけを参照し、Pattern内へraw hex、physical px、任意durationを再定義しない。density、DPI、`editor_ui_scale`で別Patternを作らない。
- `icon_consumer_refs[]`は§15.2の`widget:<pattern_id>:<anatomy_slot_id>`とsemantic tokenだけを列挙する。source icon名、font glyph、emoji、raw SVG／path、状態色、Command callbackを持たず、Panel／Command consumerとの集合一致を崩すtokenを登録しない。
- `overflow_policy`がscrollable viewportを許すPatternは`scroll_chrome_contract_ref=scrollbar.chrome.editor@1`を必須にする。単なるCanvas／Graph／Timelineのpan／orbit、またはcontentがoverflowしないsurfaceはこのrefを偽装して持たない。
- `supported_state_axes[]`は§15.3のclosed axisの部分集合であり、Pattern固有axisを追加しない。overlay順も§15.3を消費し、variantが優先順を変更しない。
- `input_contract_ref`と`focus_contract`はmouse、keyboard、touch／pen（対応surface）、assistive technologyの同じCommand収束、focus entry／exit、pointer capture cancel、drag代替を定義する。
- `semantic_role`、target／field binding、UIA patternはvisual anatomyと同じinstanceから生成する。Draw Packet、glyph、color、bounds解析からSemantic Elementを復元しない。
- `semantic_action_projection_rule`は、Command Registryが当該instanceへ束縛したActionのうち、現在のtarget、revision、Risk、Approval、stateで公開可能なものを投影する規則である。Patternは`ai_actions[]`、任意AI callback、Project write capabilityを所有せず、具体的なActionは`EditorSemanticActionV1`が所有する。
- AIは`element_key`を説明、focus request、Context選択に使用できるが、Pattern IDだけから操作権限を得ない。state変更は`command_id`、typed argument、`expected_project_revision`を持つActionだけを使用する。

C1初期Pattern Registryは次をclosed集合とする。表の「Pattern」はlogical IDの末尾であり、完全IDは`widget.<family>.<name>@1`とする。

| family | Pattern |
|---|---|
| `command` | `button`、`tool-button`、`menu-item` |
| `input` | `text-field`、`multi-line-text-editor`、`search-field`、`numeric-field`、`slider`、`reference-picker` |
| `choice` | `checkbox`、`toggle`、`radio`、`combo-box` |
| `collection` | `tree-view`、`tree-row`、`asset-view`、`list-row`、`asset-tile`、`table-view`、`column-header`、`table-row` |
| `property` | `property-row`、`property-group-header`、`component-field-group` |
| `shell` | `tab`、`panel-header`、`dock-handle` |
| `feedback` | `status-badge`、`validation-rail`、`progress`、`notification` |
| `canvas` | `canvas-surface`、`context-bar`、`selection-overlay`、`gizmo` |
| `graph` | `graph-surface`、`graph-node`、`graph-port`、`graph-edge` |
| `timeline` | `timeline-surface`、`time-ruler`、`timeline-track`、`timeline-span`、`keyframe`、`playhead` |
| `diagnostics-source` | `diagnostic-view`、`diagnostic-row`、`log-line`、`evidence-gap-row`、`source-line`、`diff-hunk` |
| `ai-task` | `proposal-card`、`approval-bar`、`task-stage`、`receipt-row` |

RegistryにないPatternが必要な場合は既存Patternのslot／variantで意味を保てるかを先に判定し、見た目だけの差で新IDを追加しない。新しいsemantic role、input model、target binding、UIA pattern、state combinationが必要な場合だけversioned Patternを追加し、既存Referenceとconformance scenarioへの影響を同じ変更で列挙する。

各Patternは適用する単独stateに加え、少なくとも`selected + runtime + warning`、`read_only + proposal(ai) + approval.pending`、`stale + approval.pending`のうち適用可能な重畳scenarioを持つ。次は共通のfail-closed規則である。

- `freshness=stale`または`superseded`でCommit Actionをenabledにしない。`rebase`／`discard`等の登録済み回復Actionだけを提示する。
- `editability=read_only`または`locked`でmutating Actionをenabledにしない。copy、focus、semantic read、明示されたunlock requestは別に評価する。
- Approval必須Actionは`approval=pending／rejected／expired`でCommit可能にしない。
- `live_edit=runtime_read_only`でSource変更Actionを公開しない。`allowed`または`preview_only`も対応Commandのpolicyを越えない。
- `availability=disabled`のActionは`disabled_reason_key`を必須とし、視覚上disabledでもSemantic Actionをenabledにしない。
- validation、selection、proposal、runtime、freshnessの一つを別axisのbadge、色、tooltipへ畳み込まない。

Conformanceは全状態のScreenshot直積を要求しない。state／Actionの許可規則、target／revision、Pattern／variant解決は全件を機械検査し、visual fixtureは各単独stateと上記必須重畳、Dark、Windowsの四つのbuilt-in contrast theme、compact／standard／comfortable、`editor_ui_scale=1.00／2.00`を既存DPI matrix上で覆う。Screenshotはvisual差だけ、Semantic Snapshotはrole／target／state／Action、Command Receiptは実際の変更経路だけを証明し、相互に代用しない。

#### Cross-pattern Scroll Chrome Contract

`scrollbar.chrome.editor@1`はTree／List、Asset View、Table、Property／Formのscroll container、Diagnostics、Source／Diffのようにactual scroll viewportを持つPatternの共通contractである。各axisは一つの`logical_extent_lu`、`logical_viewport_lu`、`logical_offset_lu`を持ち、表示状態は`absent | hidden | indicator | revealed | persistent`だけとする。`size.scroll-chrome.reveal-gutter=16 ui_lu`をoverflow axisごとにreserved overlay gutterとして確保し、両axisのcornerもnon-content areaにする。Chromeの表示／非表示、density、DPI、UI scale、motionでviewport／extent／offset、row／field layout、scroll anchor、hit target、selection／focus geometryを再計算しない。

pointerがgutterまたはfull chromeへ入った場合、そのaxisのscroll containerだけがpointerを受け、背後row／field／Canvasへpass-throughしない。thumb drag、track page、wheel、touch／pen pan、keyboard Page／Home／End、UIA Scroll／ScrollItemは同じlogical offset transactionへ収束する。scroll操作、chrome hover、pointer capture、preference changeはProject revision、Stable target、selection、keyboard focus、Command authorization、Task／Receiptを変更せず、AI、MCP、CLIにscreen coordinate、gutter、thumb、UIA childを操作経路として公開しない。

scrollable containerはvisual chromeが`hidden`／`indicator`でもUIA Scrollを保持し、virtual collectionの該当childはScrollItemを保持する。axisがscrollableでない場合はMicrosoft UIA規約どおり`ViewSize=100`、`ScrollPercent=UIA_ScrollPatternNoScroll (-1)`をexactに返す。`revealed`／`persistent`でfull chromeをControl Viewにmaterializeする場合だけ、childは`ControlType.ScrollBar`、axis orientation、`IsContentElement=false`を持ち、Scroll patternはcontainerだけが実装する。独自UIA pattern、ScrollBarからのProject Action、visibilityをscrollabilityまたはAccessibility availabilityへ読み替えることを禁止する。

Tree／Asset／Tableの`scrollbars` anatomy slotとDiagnosticsのscrollbarはこのcontractを参照する。Source／Diffのtext viewportも同じlogical offset／UIA規則を使う。Canvas、Graph、Timelineのpan／orbitをscrollbarへ偽装せず、actual companion collectionだけを対象にする。

##### Cross-pattern standard primitive contract

以下のRegistry PatternはPanel固有classやWidget callbackで意味を足さず、このmatrixをそのまま`WidgetPatternContractV1`へ解決する。`multi-line-text-editor@1`、collection、canvas、graph、timeline、source／task、notificationは後続の専用contractを優先する。

| Pattern | required anatomy／input | UIA projection | 禁止 |
|---|---|---|---|
| `command.button`／`tool-button` | visible labelまたはicon、localized accessible name、registered Command。icon-onlyはtooltipとshortcutを補足に持つ | Button＋Invoke | glyph、tooltip、pointer callbackだけからActionを作ること |
| `command.menu-item` | label、shortcut、optional submenu indicator、disabled reason。`invoke | toggle | radio | submenu`をclosed variantとし、一instanceで混在させない | MenuItem。`invoke`はInvoke、`toggle`はToggle、`radio`はSelectionItem＋exact SelectionContainer、`submenu`だけExpandCollapse。親Menuはopened／closed eventを出す | menu openをCommand execution、hoverをselectionまたはapprovalへ読むこと、bool／exclusive commandをInvokeだけのMenuItemまたは未登録variantへ偽装すること |
| `input.text-field`／`search-field` | label、editor、optional clear。search queryはpresentation stateでありProject fieldではない | Edit＋Text／TextEdit／Value。read-onlyではText selection／copyだけ | placeholder、search result row、typed display textをStable targetまたはwrite権限にすること |
| `input.numeric-field`／`slider` | label、typed numeric editor、optional unit、rangeが完全にschema化された場合だけslider | finite exact rangeはSlider／RangeValue、それ以外はEdit／Value | sliderだけを精密入力、formatted unitをparse identity、range外clampをCommit成功へ見せること |
| `input.reference-picker` | label、current Stable ref presentation、registered browse／clear／open action | ComboBoxまたはButton＋List／ListItem。current display nameはNameの補助で、saved valueはStable ref | display name、path、thumbnail、row indexからreferenceを確定すること |
| `choice.checkbox`／`toggle` | label、current bool、optional explanation | CheckBox＋Toggle | bool以外のenumやapprovalをtoggle stateへ畳み込むこと |
| `choice.radio` | group label、mutually-exclusive item、current selection | RadioButton＋SelectionItem、SelectionContainer。Toggleなし | radio group外のitemを同じSelectionContainerへ入れること |
| `choice.combo-box` | label、current selection、drop-down list。arbitrary valueを許すかschemaで固定 | ComboBox＋ExpandCollapse、List／ListItem、editable時だけValue | ComboBox rootへScroll、current valueをName、hidden option providerを残すこと |
| `property.property-group-header`／`component-field-group` | group label、optional expand control、registered child field Pattern | Group。expandable headerはchild ButtonまたはExpandCollapseを明示し、componentはchild fieldをその型で公開 | visual two-column grid、group title、component順をfield identityへすること |
| `shell.tab`／`panel-header` | tab label／dirty marker／close等のregistered child action、Panel titleとstatus slot | Tab＋Selection／TabItem＋SelectionItem。Panel headerはGroupとchild Button | active tab、Panel title、dock positionをProject target／Actionにすること |
| `shell.dock-handle` | resize grip、axis、registered keyboard resize Command | Thumb＋Transform。keyboard alternativeは同じWorkspace layout transactionへ収束 | drag-only resize、TransformからProject write、screen coordinateをAI経路にすること |
| `feedback.status-badge`／`validation-rail` | short localized state labelまたはicon＋text。reasonはDescribedBy／inline messageへ | noninteractive Text／Group。registered recovery actionがある場合だけseparate Button | color／iconだけ、badge click、tooltipだけにreason／Actionを置くこと |
| `feedback.progress` | task label、known-unit completed／totalまたはindeterminate stage、optional registered cancel | ProgressBar。known-unitはread-only RangeValue、textual progressはread-only Value | invented percent、ProgressBar自体をcancel／approval Actionにすること |

全primitiveは§15.3のstate axisだけを消費し、`disabled`、`read_only`、`locked`、`stale`、`approval`、Task stateを独自boolやcolorへ再符号化しない。label由来のUIA Nameはcurrent value、badge、shortcut、localized error全文を連結せず、必要なreasonはDescribedByまたはbounded notificationに置く。pointer、keyboard、UIAはいずれも同じregistered Command／local draft transactionへ収束し、AI、MCP、CLIはPattern、ControlType、UIA provider、screen coordinateを操作経路にしない。

`scenario.widget.standard-primitives.contract@1`は全matrix rowについてnormal／hover／pressed／focus／disabled／read-only／warning／error／stale、Dark／Light、四Contrast Theme、三density、200% Font、`editor_ui_scale=2.00`、keyboard／Narrator／NVDA／UIA、allowed／denied Commandを検証する。さらにWidget／Panel／Command presentationのEditor-owned system icon consumer集合、`EditorIconTokenContractV1`、generated conversion manifestがbyte-exactに一致し、Regular／Filledの適用がnormal／selected-or-activeだけ、16／20 lu、localized textまたはaccessible name／tooltip、状態iconのDescribedBy、AI typed Actionとの非同一性を同じfixtureで照合する。未登録Action、tooltip-only reason、曖昧なMenuItem variant、radio Toggle、combo root Scroll、icon-only unnamed button、source icon名直指定、emoji／unknown icon、dock drag-only、badge clickによるauthority変更を0件にする。

#### 15.6.1 `widget.property.property-row@1`

`property-row@1`はInspector、Import Inspector、Level Form等で一つの`canonical_field_ref`を表示・編集・reviewする`composite` Patternである。Panelごとにlabel列、validation、mixed value、proposal、read-only、Commit境界を再実装しない。rowはProject valueの正本を所有せず、current read projection、local draft、proposal、runtime projectionを別authority layerとして表示する。

Pattern instanceは次を一件ずつ束縛する。

- `AuthoringSelectionContextV1`または同等のbounded target集合とtarget set hash
- exact `canonical_field_ref`、field schema ref、expected Project revision
- `value=set`ではcurrent typed value、`mixed／unset`ではvalueを捏造しない明示state
- editability、authority、provenance、validation、freshness、approval、live-edit policy
- Command Registryから投影されたfocus、copy、set、reset、open picker／editor、proposal review、rebase／discard Action

`pattern_id`や表示labelはfield identity、target identity、Action、permissionを代用しない。同じfieldを複数Panelが表示しても、各row instanceは同じcanonical fieldとrevisionへ解決し、Project変更は同じtyped Change Primitiveへ収束する。

##### Property row anatomy

| slot | 必須 | 内容と規則 |
|---|---|---|
| `row-root` | required | semantic role=`property_row`。background、focus、validation、proposal、runtimeの合成範囲。Tab stopまたはProject write ownerにしない |
| `validation-rail` | optional | 2 lu left overlay。error／warning時だけ表示し、layout幅、target、Actionを持たない |
| `label` | required | localization済みfield labelをUI typographyで一行表示。ellipsis時もfull labelをaccessible nameとtooltipへ保持し、identifierを翻訳しない |
| `editor` | required | field schemaから選んだ登録済みchild Pattern。current valueをproposal／runtime valueで上書きしない |
| `unit` | optional | typed unit presentation。numeric editorのparse文字列、field identity、validation messageへ連結しない。wrapしない |
| `status` | optional | mixed、unset、read-only／lock、default／override、provenance、runtime、staleの短いlabel＋icon。validation railまたはproposal badgeを代用しない |
| `auxiliary-actions` | optional | reset、browse、open full editor等の登録済みToolButton。icon-onlyの場合もaccessible name、tooltip、shortcutをCommand Registryから取得 |
| `message` | optional | validation／lock／restart reasonを最大二行。超過はellipsis＋`詳細を開く` Actionとし、Semantic／UIAにはboundedな全文を保持 |
| `proposal` | optional | `widget.ai-task.proposal-card@1`のfield-level child。before／after、base／current revision、validation、approval、accept／rejectを持ち、`editor`のcurrent valueを置換しない |

`editor` slotのfield schema対応は次のclosed規則とする。該当しないschemaをstringへ暗黙変換せず、unsupported diagnosticを返す。

| field kind | child Pattern／規則 |
|---|---|
| boolean | 既定は`widget.choice.checkbox@1`。modeの即時ON／OFFを表すfieldだけ、metadataのclosed presentation指定で`widget.choice.toggle@1`を許可 |
| enum | `widget.choice.combo-box@1`。表示候補はlocalized label、保存値とAI typed argumentはenum ID |
| string／identifier／path | `widget.input.text-field@1`。identifier／pathはcode typography。複数行Sourceはrow内へ展開せずsummary＋`widget.input.multi-line-text-editor@1`を開くAction |
| bounded／unbounded numeric scalar | `widget.input.numeric-field@1`。bounded rangeとstepが全てschemaにある場合だけ補助`widget.input.slider@1`を許可し、numeric fieldを唯一の精密入力として残す |
| typed Object／Asset reference | `widget.input.reference-picker@1`。display nameは補助表示、保存・Action targetはStable ref。text入力だけから参照を確定しない |
| vector／color／rect／range等のtyped components | `widget.property.component-field-group@1`。component ID、順序、unitをschemaで閉じ、表示順や色swatchから意味を推測しない |
| derived／generated read-only | 対応するchild Patternを`editability=read_only`で表示し、copy／focus／semantic readを維持。Source Intentへの登録済み遷移Actionがある場合だけ提示 |

##### Size、reflow、content

size値は`editor_ui_scale=1.00`の`ui_lu`であり、`message-max-lines`だけは行数である。最終physical pixelへの変換は§15.1だけを使う。

| token | 値 |
|---|---:|
| `size.property-row.inline-min` | 320 |
| `size.property-row.stacked-min` | 192 |
| `size.property-row.label-min` | 96 |
| `size.property-row.label-preferred` | 128 |
| `size.property-row.label-max` | 160 |
| `size.property-row.editor-min` | 120 |
| `size.property-row.unit-max` | 64 |
| `size.property-row.message-max-lines` | 2 |

- available幅320 lu以上は`label | editor | unit | status／actions`を一行にし、labelは96～160 lu、preferred 128 lu、editorは残余を取り120 lu未満にしない。gap、padding、control高は§15.4のdensity tokenを使う。
- 192～319 luは`label | status／actions`、次行`editor | unit`へreflowし、`message`と`proposal`はその下へ置く。reflow前後でreading order、focus order、Semantic parent、Command targetを変えない。
- 192 lu未満ではchildをclip、重畳、off-screenにせずtyped layout constraintを返す。WorkspaceはPanel tab化、scroll、layout errorの順で回復し、row単位の水平scrollを作らない。
- labelとread-only summaryは一行ellipsisを許す。編集中のtext／numeric valueはcaret位置を可視にする内部scrollを使い、値をellipsisしたまま編集させない。
- Japanese labelはUI face、identifier／path／literal／editable numeric valueはcode face、unitはtyped presentation roleを使用し、baselineを同一にする。

##### Interaction、draft、Commit

- row rootはTab stopにせず、focus orderは`editor`、`auxiliary-actions`、`proposal review actions`で固定する。stacked reflowやRTL／Localizationで順序を入れ替えない。labelのpointer activationはeditorへのfocus requestだけを生成する。
- text／numeric／reference draftはcurrent Project valueと分離する。Enterまたはfocus transferはvalidationを実行し、validなら一ChangeSetを要求する。invalidならdraftとmessageを維持するがfocusをtrapせず、Project値をclamp、parse fallback、部分Commitしない。
- Escapeは未Commit draftを破棄してcurrent projectionへ戻す。Panel close、selection／Project revision変更ではsilent Commitせず、draftをstale化して明示的なdiscard／rebase policyへ渡す。
- slider、numeric scrub等のcontinuous inputはlocal draftだけを更新し、pointer releaseまたはEnterで一ChangeSetを要求する。Escape、pointer cancel、capture lossはdraftを破棄し、最後のpreview値をCommitしない。
- checkbox／toggle／enum等のdiscrete inputは一回の操作を一ChangeSetとし、同じvalidation、Risk、Approval、expected revisionを通す。
- multi-selectの`mixed`はem dash＋`Mixed` statusとして表示し、代表値、先頭値、平均値を作らない。Userが明示値を入力した場合だけ、exact target集合へ一ChangeSetを提案し、影響件数とfield refをCommit前に示す。

##### Property row state、proposal、Runtime

`property-row@1`はinteraction、availability、editability、value、authority、provenance、validation、freshness、approval、live-edit axisを使い、selectionとtask axisは固定`unselected／idle`とする。現在選択されているObjectはInspector contextで示し、rowをselected fillにしない。

- error／warningはrail、icon、message、Semantic validation stateを同時に更新する。errorでCommitを拒否し、warningはPolicyが明示的にblockする場合以外はwarningだけを理由にerrorへ昇格しない。
- `read_only + proposal(ai) + approval.pending`ではcurrent editorをfocus／copy可能なread-onlyとして残し、proposal childを別表示する。accept可能性はproposal ActionのPolicyが決め、rowのread-only表示だけから許可しない。
- stale／superseded proposalはbefore／afterを維持するがacceptをdisabledにし、base／current revisionとrebase／discardだけを提示する。
- RuntimeではSource、runtime value、live-edit policyを区別する。`runtime_read_only`はwrite Actionなし、`preview_only`はProject Commitなし、`restart_required`はSource ChangeSetとrestart reasonを併記する。
- AI proposal、Engine validation、human lock、Runtime、mixedを一つのstatus badgeへ畳み込まない。statusの表示順はfreshness → editability／lock → runtime／live edit → approval／proposal → provenance → value stateとし、validationはrail／messageに分離する。幅不足時もstateを省略せず、各icon＋短いlabelを`message`／`proposal`領域へ同じ順序でreflowする。

##### Semantic、UIA、AI

Semantic Snapshotではrow rootを`role=property_row`、同一target context、`canonical_field_ref`、current `typed_value`または明示`mixed／unset` stateで公開する。draftとproposalはcurrent valueを上書きせず別child element／refとして公開し、各Actionは`enabled`、`disabled_reason_key`、Risk、Approval、`expected_project_revision`を持つ。

UIAはvisualな二列配置をDataGridへ偽装しない。property group headerはGroup、row rootはraw treeのstructural fragment（`IsControlElement=false`、`IsContentElement=false`）、labelはText、editorとActionは型に対応する標準Control type／patternとして公開する。

- editorのNameはlocalized full labelから得て、current valueをNameへ連結しない。独立label Textがある場合は`LabeledBy`で参照する。
- validation、lock、restart reasonのText elementを`DescribedBy`へ結び、rowの色、rail、tooltipだけに置かない。状態変化は必要なUIA property／Text eventとbounded announcementを発行する。
- string inputはText＋Value、finite min／maxを持つnumeric rangeはRangeValue、unbounded numericはText＋Value、checkbox／toggleはToggle、enumはComboBoxの標準patternを使い、表現可能なControlへcustom UIA patternを追加しない。
- `read_only`はeditorをfocus／copy可能にしてValue／TextまたはRangeValueのread-only stateを返す。`disabled`はIsEnabled=falseとreasonを公開し、read-onlyと同一にしない。
- `mixed／unset`で単一numeric valueを捏造しない。editor childの`UIA_ItemStatusPropertyId`へlocalized `Mixed`／`Unset`、Valueへempty stringを公開し、明示入力でscalar draftが成立するまでRangeValueを公開しない。mixedとunsetの区別をplaceholder、em dash、空Valueだけへ依存させない。
- AIは同じrowのSemantic Elementを読むがUIA clientにはならない。Pattern ID、label、boundsからActionを作らず、Snapshotが公開した`EditorSemanticActionV1`だけをtyped argumentとexpected revision付きで要求する。

##### Property row required conformance scenarios

| scenario ID | 必須内容 |
|---|---|
| `scenario.widget.property-row.scalar-edit@1` | text／numericのdraft、valid Commit、Escape、focus transfer、Undoが同じfield／revisionへ収束 |
| `scenario.widget.property-row.continuous-cancel@1` | slider／scrubのpreview、release Commit、Escape／pointer cancel／capture lossでProject不変 |
| `scenario.widget.property-row.mixed-multiselect@1` | 2件以上のmixed表示、代表値なし、exact target集合への一ChangeSet、部分適用なし |
| `scenario.widget.property-row.validation@1` | error／warning、inline message、Problems focus、UIA DescribedBy、AI disabled reasonが同じDiagnosticへ収束 |
| `scenario.widget.property-row.runtime-read-only@1` | Runtime＋warning＋read-only、copy／focus可、write不可、Source Intent遷移またはrestart reason |
| `scenario.widget.property-row.ai-proposal@1` | current／before／after分離、AI provenance、approval pending、stale化、accept拒否、rebase／discard |
| `scenario.widget.property-row.reflow-localization@1` | compact／standard／comfortable、inline／stacked、Light／Dark／四High Contrast、`editor_ui_scale=1.00／2.00`で同じreading／focus／Semantic順 |
| `scenario.widget.property-row.ime-content@1` | Japanese IME composition、64文字ASCII identifier、長い日本語label、Windows path、`-1234567.890 m/s²`でclip／誤parse／target driftなし |

Reference 01ではInspectorの少なくとも三rowを固定fixtureにする。通常scalar、`runtime_read_only + warning`、`proposal(ai) + approval.pending + stale`を同時表示し、Problems、AI Partner、manual edit、UIAが同じtarget、field、revision、Diagnostic、Action可否へ収束するまで`property-row@1`をclosedと扱わない。

#### 15.6.2 `widget.collection.tree-row@1`

`tree-row@1`はHierarchy／Outliner、World Outline、Asset Browserのhierarchical viewで、一つのStable targetを表示、選択、展開し、許可されたrename／move Actionへ接続する`composite` Patternである。companion surfaceの`widget.collection.tree-view@1`がcollection query、selection／focus／expansion、scroll、virtualization、Tree UIA rootを所有し、`tree-row@1`をchildとしてmaterializeする。rowはProject hierarchy、display order、selection集合、proposalの正本を所有しない。current Project projection、Workspace内のfocus／expansion、local rename draft、AI proposal、Runtime projectionを別stateとして合成する。

Pattern instanceは次を一件ずつ束縛する。

- exact Stable target ref、target kind、tree collection key、parent Stable refまたはroot、semantic depth
- `has_children`と`expanded／collapsed／leaf`、Project revision、collection／filter／sort revision
- exact selection集合またはrevision-bound query selection、active item、focus item、range anchor
- editability、authority、provenance、validation、freshness、approval、live-edit policy
- Command Registryから投影されたfocus、select、expand／collapse、open／reveal、rename、move before／after／into、proposal review、rebase／discard Action

`pattern_id`、row index、表示label、同名Object、Hierarchy path、screen coordinateはtarget identityまたはAction permissionを代用しない。ExpansionはPanel instanceに属するpresentation state、selectionは`AuthoringSelectionContextV1`、rename／reparentはProject ChangeSetとして別々に扱う。expand、focus、revealはProject revisionを変更せず、rename／reparentだけが対応するtyped Change Primitiveとexpected revisionを要求する。

##### Companion surface: `widget.collection.tree-view@1`

`tree-view@1`はcomposition level=`surface`、semantic role=`tree`であり、次のanatomyだけを持つ。

| slot | 必須 | 内容と規則 |
|---|---|---|
| `collection-root` | required | collection key、query／Project revision、selection／focus／expansion generation、Tree Semantic／UIA root |
| `viewport` | required | `tree-row@1`のvisible range＋前後2 viewportをmaterializeする。row indexをidentityにしない |
| `selection-summary` | optional | exact／query selection総数とfilter-hidden件数。selected rowの代用にしない |
| `collection-feedback` | optional | empty、loading、query error、capacity errorをregistered feedback Patternで表示。targetを持たないplaceholder rowを作らない |
| `scrollbars` | conditional | content extentがviewportを超える軸だけ。`scrollbar.chrome.editor@1`を参照し、keyboard／UIA Scrollと同じlogical offsetを使う |

tree-view rootはinteraction、availability、task axisだけを使用し、Project itemのvalidation、proposal、Runtime、freshnessをroot badgeへ集約しない。input、selection、focus、virtualization、UIA、AI queryは本節のrow contractと一体で閉じ、Panelが別container modelを上書きしない。

##### Tree row anatomy

| slot | 必須 | 内容と規則 |
|---|---|---|
| `row-root` | required | semantic role=`tree_item`。一つのStable target、focus、selection、state overlay、pointer hit範囲を持つ。表示row indexをAutomation IDにしない |
| `validation-rail` | optional | 2 lu left overlay。error／warning時だけ表示し、layout幅、target、Actionを持たない |
| `indent-guides` | optional | semantic depthから導く装飾。親子identity、drop先、reading orderを線の有無から推測しない |
| `disclosure` | conditional | `has_children=true`だけ16 luの展開glyphを表示。row selectionを変更せずexpand／collapse presentation Commandだけを要求する。独立Tab stop／UIA childにせず、TreeItem rootのExpandCollapseへ投影する。leafは同幅の空き領域を持つがControlを公開しない |
| `type-icon` | required | target kindの登録済みicon。iconだけで種類を伝えず、Semantic／UIA ItemTypeへ同じlocalized kind名を公開する |
| `primary-label` | required | current target nameを一行表示。Object／Asset名はUI face、identifier／pathはcode faceとし、full textをSemantic／UIA Nameへ保持する |
| `secondary-metadata` | optional | Scene owner、type、path、license等のread projectionを一行表示。target identity、status、Action reasonを代用しない |
| `status` | optional | lock／read-only、runtime、validation、freshness、approvalのicon＋短いlabel。selection fill、validation rail、proposal markerを代用しない |
| `inline-actions` | optional | open、visibility等の登録済みToolButton。hoverだけで出現させず、keyboard／accessibilityから同じActionへ到達できる |
| `rename-editor` | conditional | rename中だけ`widget.input.text-field@1`を`primary-label`の位置へ表示。current label、target ref、local draftを別々に保持する |
| `proposal-marker` | optional | current rowを置換しない`ai.proposal` dash＋`AI提案`。before／after、proposal ref、base／current revisionは登録済みProposal Cardへ遷移して表示する |
| `drop-indicator` | optional | preview中だけbefore／afterの2 lu lineまたはintoのbounded outlineを表示。validityとreasonを伴い、Project hierarchyを先行変更しない |

##### Size、overflow、content

size値は`editor_ui_scale=1.00`の`ui_lu`である。row高、body typography、horizontal padding、iconは§15.4のdensity profileをそのまま使い、row内で再定義しない。

| token | 値 |
|---|---:|
| `size.tree-row.content-min` | 192 |
| `size.tree-row.indent-step` | 16 |
| `size.tree-row.disclosure` | 16 |
| `size.tree-row.label-min` | 64 |
| `size.tree-row.secondary-max` | 160 |
| `size.tree-row.rename-min` | 96 |
| `size.tree-row.drop-indicator` | 2 |
| `size.tree-row.focus-reveal-margin` | 8 |

- rowはsingle-lineの固定高とし、validation、proposal、Runtime、staleで高さを変えない。これによりcompact／standard／comfortableの24／28／32 luとvirtual rangeの対応を維持する。
- 幅は`indent + disclosure + type-icon + primary-label + status`を先に確保する。不足時はsecondary metadataを省略し、次にinline actionをcontext menu／Command paletteへ移す。statusはicon＋短いlabelを維持し、primary labelは64 luまで縮めてellipsisする。
- 上記でも足りない場合、row単位をreflow、clip、重畳せずtree containerのkeyboard操作可能なhorizontal scrollを使う。focus／selection時は対象rowを左右8 lu marginまでauto-revealする。192 lu未満はtyped layout constraintとし、stateやdisclosureを消して成功扱いにしない。
- depthはProject projectionの値を保ち、描画都合で浅く見せない。深いhierarchyはindent幅を確保してhorizontal scrollし、Domain ownerが拒否したdepthをUIだけで受理しない。
- 日本語名はUI face、64文字ASCII identifier、Windows path、Asset IDはcode faceを使う。同一row内でbaselineを揃え、ellipsis後もcopy、Semantic name、typed targetを短縮しない。

##### Selection、focus、keyboard

Outlinerとhierarchical Asset Browserは同じmulti-select modelを使い、focusとselectionを分離する。

- row本体のplain clickはfocusを移しselectionをその一件へ置換する。`Ctrl+click`はそのStable refをtoggleし、`Shift+click`は現在のvisible orderとrange anchorで連続範囲を選ぶ。disclosure、inline action、scrollbarの操作はselectionを暗黙変更しない。
- `Up／Down`は前後のvisible rowへfocusを移しselectionをその一件へ置換する。`Ctrl+Up／Down`はselectionを変えずfocusだけ、`Shift+Up／Down`はanchorから範囲を延長する。`Ctrl+Space`はfocused Stable refをtoggleする。
- `Right`はcollapsed parentを展開し、既にexpandedならfirst visible childへfocusする。`Left`はexpanded parentを畳み、既にcollapsed／leafならparentへfocusする。`Home／End`はfirst／last visible row、`PageUp／PageDown`は一viewport、`Enter`は登録済みdefault open／focus Action、`F2`はrename、`Escape`はrename／drag previewをcancelする。
- selected集合はStable refで保持し、rename、reparent、scroll、DPI変更では失わない。filterで非表示になったselected targetはselectionから削除せず、container summaryへ`選択 n件（非表示 m件）`を示す。deleted／permission-lost targetだけを理由付きで集合から除外する。
- range anchorはStable refとcollection／filter／sort revisionの組で保持する。revisionが変わった後の最初のrange操作は範囲を推測せずfocused rowを新anchorとして単一選択し、bounded announcementでanchor resetを通知する。
- `Ctrl+A`は「current filterとexpansion generationから得るvisible flattened collectionの全件」をrevision-bound query selectionとして表せる場合だけ有効にする。collapsed descendantを暗黙選択せず、100万件をrow列挙しない。query ref、count、exclusion、revisionを保持し、exact query selectionを表せないcontainerはdisabled reasonを公開する。
- focus ringはselected fill、validation rail、Runtime badgeの上へ描画する。multi-select時もfocused row一件だけにringを出し、selectedとfocusedを同じ色だけで表さない。

##### Rename、drag／drop、Commit

- `F2`またはregistered Rename Actionはcurrent labelからlocal draftを開始する。IME composition、selection、clipboard、horizontal caret scrollは`text-field@1`に従う。Enterはvalidation後に一Rename ChangeSetを要求し、Escape、selection／filter／sort／Project revision変更、Panel closeはsilent Commitせずdraftを破棄してProject不変を通知する。
- invalid renameはdraft、message、focusを維持し、trim、fallback名、自動suffix、部分Commitで成功させない。Commit後にsort／filterで位置または可視性が変わってもStable refでfocusを再解決し、非表示ならcontainer summaryへ結果とReveal Actionを出す。
- internal drag payloadはStable ref集合、source collection key、Project revisionだけを持つ。pointer address、row index、label、Hierarchy pathをpayloadにしない。drag開始時のselected集合をfreezeし、途中のhoverで追加選択しない。
- rowの上25%を`before`、中央50%を`into`、下25%を`after` preview zoneとする。`before／after`はtargetと同じparentのsibling位置、`into`はtargetのchild位置を意味する。leafまたはDomainがchildを許さないtargetで`into`を`after`へ読み替えず、disabled reasonを表示する。
- previewはcycle、World／Scene boundary、persistent owner、lock、capability、permission、expected revisionをdrop前に検証する。valid previewは2 lu line／bounded outline＋action label、invalid previewはerror／warning shape＋reasonを示す。pointer releaseはvalid previewからだけ一typed Move ChangeSetを要求する。
- hoverによる自動expandは行わない。collapsed targetへのvalid `into`はcollapsedのままCommitでき、内容確認は明示的なdisclosure／Right／Expand Commandで行う。pointer cancel、capture loss、Escape、revision driftはpreviewを破棄しProjectを変更しない。
- drag-onlyにせず、context menu／Command paletteから`前へ移動`、`後へ移動`、`子へ移動`をtarget picker付きで提供する。pointer dragとkeyboard／accessibilityからInvokeするMove Commandは同じpreview validationとtyped Move ChangeSetへ収束する。UIA Drag／DropTargetは進行中previewとeffectを公開するが、Pattern自体へ別のmove権限を追加しない。

##### Tree row state、proposal、Runtime

`tree-row@1`はinteraction、availability、editability、selection、authority、provenance、validation、freshness、approval、live-edit axisを使い、valueは固定`set`、taskは固定`idle`とする。

- `selected + runtime + warning`はselection fill、`Runtime` badge、warning rail＋triangle＋`警告` label、focus ringを別layerで同時表示する。full Diagnosticはstatusのaccessible descriptionとProblems focus Actionへ結び、tooltipだけへ置かない。どれか一つを残りの意味へ使わない。
- `read_only／locked`はfocus、select、copy、revealを維持し、rename／moveをdisabled reason付きで拒否する。targetが存在する通常rowは`availability=enabled`を保ち、各Actionを個別にdisableする。`availability=disabled`はStable target refを保持したままcontainerがsuspended／unavailableになった場合だけに限定してIsEnabled=false、reason、container側のRefresh／Recover Actionを公開し、選択可能なread-only rowと混同しない。targetを持たないloading placeholderは`tree-row@1`にせずfeedback Patternを使い、消失済みtargetのold elementは再取得を要求する。
- AIのrename／reparent proposalがあってもcurrent labelとcurrent hierarchy位置を置換しない。rowはdash＋`AI提案`＋proposal review Actionだけを示し、before／afterとaccept／rejectはproposal refを持つProposal Cardへ投影する。未Commit create proposalを実在するTree targetとしてghost row化しない。
- `proposal(ai) + approval.pending + stale`はdash、AI label、clock、base／current revisionを維持し、accept／rename／moveをdisabledにする。rebase／discardだけを現在のPolicyから再投影する。
- Runtime projectionは同じStable targetのcurrent runtime statusをbadgeとして重ねる。runtime-only／derived targetはSource targetへ偽装せずread-only reasonとSource Intentへの登録済み遷移Actionを示す。

##### Tree virtualization、Semantic、UIA、AI

- Tree containerはvisible rangeと前後2 viewportだけをmaterializeし、`EditorSemanticVirtualCollectionV1`の`total_count`、`realized_range`、`omitted_count`、filter／sort ref、continuationを公開する。`total_count`はcurrent filter、sort、expansion generationから得るvisible flattened collectionの件数であり、collapsed descendantを含む総Project件数へ偽装しない。unknownまたは計算中なら推測値を作らない。
- 展開、collapse、filter、sort、reparentでcollection generationが変わったら、old row／placeholderを再利用しない。target refが同じでもfresh Snapshotから新しいelement key、parent、depth、action、revisionを取得する。
- Semantic Snapshotはrowを`role=tree_item`として、Stable target、parent ref、depth、expanded state、selected／focused、kind、full label、state axis、現在許可されたActionを公開する。表示順、indent pixel、glyph、color、boundsはAI向けsemantic hashへ含めない。
- UIA rootはTree＋Selection、scroll可能ならScroll、virtualized itemを持つならItemContainerを公開する。realized rowはTreeItem＋ExpandCollapse（leafはLeafNode）、選択可能ならSelectionItem、scroll container内ならScrollItem、default Actionがある場合だけInvokeを公開する。Nameはfull primary label、ItemTypeはlocalized target kind、ItemStatusはbounded state summary、SelectionContainerは同じTree rootを返す。
- `FindItemByProperty`はAutomationId、Name、IsSelectedを少なくとも扱い、Stable targetから導くAutomationIdはpeer内で一意にする。virtualized matchはUI changeを起こさずVirtualizedItemだけを持つplaceholderを返し、`Realize`後にfull property／patternを公開する。viewportまたはgeneration変更で無効になったproviderは`UIA_E_ELEMENTNOTAVAILABLE`を返す。
- `Selection.GetSelection`はrealized selected itemだけをboundedに返し、Tree rootのItemStatus／DescribedByへselected総数とfilter-hidden／collapsed／unrealized件数を示す。expanded content内の残りは`FindItemByProperty(IsSelected)`で順次取得し、filter-hidden／collapsed itemはfilter／ancestor expansionが明示変更されるまでsummaryだけを公開する。selected itemを返すためだけに全rowをmaterializeしない。
- UIA treeはexpanded／visible hierarchyを親子構造として公開し、indent幅やdisplay pathを階層の代用にしない。collapsed descendantをvisible rowまたはroot直下placeholderへ偽装せず、clientはancestorを明示Expandしてから、expanded content内のoff-screen itemをItemContainer queryとRealizeで取得する。
- AI、MCP、CLIはUIA clientにならない。`EditorContextSnapshotV1`のtarget ref、collection query／continuation、state、`EditorSemanticActionV1`を読み、expand／focus／revealはpresentation Command、rename／moveはtyped argumentとexpected Project revisionを持つProject Commandとして要求する。

##### Tree required conformance scenarios

| scenario ID | 必須内容 |
|---|---|
| `scenario.widget.tree-row.selection-focus@1` | plain／Ctrl／Shift pointer、arrow／Ctrl／Shift keyboard、focusとmulti-selection、hidden selection summaryが同じStable ref集合へ収束 |
| `scenario.widget.tree-row.hierarchy-keyboard@1` | disclosure、Left／Right、Home／End、Page、leaf／collapsed／expanded、presentation-only expansionでProject revision不変 |
| `scenario.widget.tree-row.rename@1` | F2、Japanese IME、valid／invalid、Enter Commit、Escape／selection／revision drift cancel、sort／filter後のStable focus |
| `scenario.widget.tree-row.move-preview@1` | before／into／after、cycle／boundary／owner／lock拒否、pointer cancel、keyboard代替、一Move ChangeSet |
| `scenario.widget.tree-row.virtualization@1` | 100万Entity／10万Asset、前後2 viewport、ItemContainer query、VirtualizedItem Realize、generation失効、全row未materialize |
| `scenario.widget.tree-row.state-overlay@1` | `selected + focused + runtime + warning`、`read_only／locked`、`availability=disabled`をLight／Dark／四High Contrastで色だけに依存せず識別し、select可否とwrite可否を混同しない |
| `scenario.widget.tree-row.ai-proposal-stale@1` | current hierarchy不変、AI dash／label、approval pending、stale、accept拒否、rebase／discard、ghost targetなし |
| `scenario.widget.tree-row.density-content@1` | compact／standard／comfortable、`editor_ui_scale=1.00／2.00`、深さ32、長い日本語名、64文字ASCII identifier、Windows pathでrow高、focus、target、status不変 |
| `scenario.widget.tree-row.query-selection@1` | filter一致全件のquery selection、count／hidden／exclusion、sort／filter revisionでanchor reset、row index由来target 0件 |

Reference 01ではOutlinerの少なくとも四rowを固定fixtureにする。expanded parent、`selected + focused + runtime + warning`、`locked`（owner／reason付き）、`proposal(ai) + approval.pending + stale`を同時表示し、Asset Browserのhierarchical viewでも同じ`tree-view@1`＋`tree-row@1`、selection model、virtualization、UIA、AI Action投影を再利用する。manual、keyboard、UIA、AI、Scene／Inspector／Problemsが同じtarget、selection集合、revision、Diagnostic、Action可否へ収束するまで`tree-row@1`をclosedと扱わない。

#### 15.6.3 `widget.collection.asset-view@1`

`asset-view@1`はAsset Browserのflat content collectionとView切替を所有する`surface` Patternであり、C1では`list`、`tile`、`columns`の三Viewを持つ。`list`は`widget.collection.list-row@1`、`tile`は`widget.collection.asset-tile@1`、`columns`は`widget.collection.table-view@1`の下に`column-header@1`と`table-row@1`をmaterializeする。View切替はpresentation stateであり、Project、filter、sort、selection、Asset revisionを変更しない。

Unreal EngineのTiles／List／Columns、Unityのicon sizeからlistへの切替、Godotのdisplay modeは複数表現の有効性を示す。ただしMiraikanaiは`columns`をListのvisual variantにせず、独立したtable surface、column schema、cell focus、sort、resize、DataGrid／Table providerを必須にする。本節で設計契約を定義するが、実装が§15.6.9のconformanceを満たすまではCapability activationをfail-closedにし、list rowを列揃えして代替しない。

```text
EditorAssetCollectionViewStateV1
  collection_key
  view_mode                         # list | tile | columns
  filter_ref
  sort_ref
  query_revision
  total_count
  selection_ref                    # exact Stable refs or revision-bound query selection
  active_target_ref?
  focus_target_ref?
  focus_column_id?                  # columnsだけ。Stable column_id
  range_anchor_target_ref?
  range_anchor_query_revision?
  scroll_anchor_target_ref?
  scroll_anchor_offset_lu
```

- `collection_key`、filter／sort、query revision、Stable target refがidentityであり、visual row／column、tile coordinate、thumbnail ref、display pathをidentityにしない。
- list／tile／columns切替ではselection、active target、focus target、range anchorを保持する。columnsへ入るときは`focus_column_id`を以前の値、なければ`asset.name`へ解決し、columnsから出るときもpresentation stateとして保持する。scrollは先頭visible targetとそのlogical offsetをanchorにして再配置し、同じtargetがfilterで消えた場合だけ次のcanonical itemへ移してbounded announcementを出す。
- selected targetがfilterで非表示になってもselectionから削除せず、`選択 n件（非表示 m件）`をcontainer summaryへ示す。deleted／permission-lost targetだけを理由付きで除外する。
- `Ctrl+A`はcurrent filterに一致する全Assetをrevision-bound query selectionとして選択し、10万件をrow／tile列挙しない。query selectionは選択時のfilter ref／revisionへimmutableに束縛し、後のfilter変更で新結果へ暗黙retargetしない。sortはselection集合を変えず、filter／sort revision変更後の最初のrange操作はfocused targetを新anchorとする。
- view mode、filter、committed sort、selection summary、total countはSemantic Snapshotへ公開する。view mode、scroll anchor、tile coordinate、column width／visibilityはpresentation metadataとして`semantic_content_hash`とAI Action target hashから除外し、View切替／resizeだけでProposalをstaleにしない。AIは三Viewで同じcollection query、Stable target、typed cell value、Actionを読み、thumbnail画像、header label、visual row／columnを対象解決へ使わない。

##### Asset view anatomy

| slot | 必須 | 内容と規則 |
|---|---|---|
| `collection-root` | required | semantic role=`asset_collection_view`。collection key、view mode、query／Project revision、selection／focus generationを持つPane＋MultipleView wrapper |
| `viewport` | required | list／tileではList child、columnsではDataGrid childの一方だけをcurrent viewとしてmaterializeし、hidden viewのWidget／UIA providerを残さない |
| `query-summary` | required | filter／sort、result count、query freshnessをtext＋registered statusで表示。thumbnail Job countと混同しない |
| `selection-summary` | optional | exact／query selection総数とfilter-hidden件数。selected itemの代用にしない |
| `collection-feedback` | optional | empty、loading、query error、capacity errorをregistered feedback Patternで表示。targetなしのrow／tileを作らない |
| `scrollbars` | conditional | content extentがviewportを超える軸だけ。`scrollbar.chrome.editor@1`を参照し、keyboard／UIA Scrollと同じlogical offsetを使う |

asset-view rootはinteraction、availability、task axisだけを使う。各Assetのvalidation、proposal、Runtime、freshness、thumbnail taskをrootの一badgeへ集約せず、itemまたはfeedback childへ投影する。

##### Input、focus、view switch

- plain clickはfocusを移しselectionを一件へ置換、`Ctrl+click`はtoggle、`Shift+click`はcanonical sort order上のanchorから連続範囲を選ぶ。focus ringはselected fillの上へ描画し、multi-selectでもfocused item一件だけに出す。
- list modeの`Up／Down`は前後itemへfocusを移しselectionを一件へ置換する。`Ctrl+Up／Down`はfocusだけ、`Shift+Up／Down`はrangeを延長する。`Left／Right`はitem内Actionへ移動せず未処理とする。
- tile modeの`Left／Right`はvisual inline方向の隣接tile、`Up／Down`は同じvisual columnの前後rowへ移動する。行端でLeft／Rightをwrapし、最終不完全rowのUp／Downは同columnがなければそのrowのlast tileへclampする。RTL時はLeft／Rightのvisual方向だけを反転し、canonical sort orderを反転しない。
- columns modeのcell navigation、row selection、header focus、sort、resize、renameは`table-view@1`、`column-header@1`、`table-row@1`へ委譲し、asset-viewが別shortcutまたはselection modelを追加しない。
- tileのrange selectionはvisual矩形でなくcanonical sort orderの連続範囲である。resize／DPI／densityでcolumn数が変わっても既存selectionとanchorを変えない。spatial arrowが新たに選ぶtargetだけは新しいcommitted layoutから決める。
- `Home／End`はfirst／last query item、`PageUp／PageDown`は一viewport、`Ctrl+Space`はfocused targetをtoggle、`Enter`はregistered default open／focus Action、`F2`はrename、`Escape`はrename／drag previewをcancelする。type-aheadはlocalization済みprimary labelのprefixへfocusするがtarget identityをlabelに置換しない。
- `Tab`はcollectionへ一回だけ入りfocused itemへ置き、次の`Tab`でcollectionを出る。item内status／thumbnail overlayをTab stopにせず、`Shift+F10`またはMenu keyで同じregistered context menuを開く。rename中だけtext-fieldへfocusを移し、Escapeでitem navigationへ戻す。
- View switch commandとUIA MultipleView `SetCurrentView`は同じpresentation Commandへ収束し、Project revisionを変えない。version 1のUIA view IDは`1=list`（name key=`editor.asset_view.list`）、`2=tile`（`editor.asset_view.tile`）、`3=columns`（`editor.asset_view.columns`）とし、表示名はLocalizationから解決してunknown IDを拒否する。切替後はfocused targetをScrollIntoViewし、active childを一回置換してStructureChangedとLayoutInvalidatedを各一回発行する。

##### Virtualization、Semantic、UIA

- list modeはvisible range＋前後2 viewport、tile modeはvisible tile row＋前後2 viewport row、columns modeは`table-view@1`が定めるrow／cellだけをmaterializeする。`total_count`、realized／omitted range、continuationはcurrent filter／sort／query revisionへ結び、thumbnail Jobの完了順でitem順序を変えない。
- generation、filter、sort、view mode、Project revisionが変化したold element／placeholderは再利用せず、fresh SnapshotからStable targetを再取得させる。
- asset-view wrapperはPane＋MultipleViewだけを公開し、current childはlist／tileでList＋Selection／Scroll／ItemContainer、columnsで`table-view@1`のDataGridを公開する。tile childはcurrent queryのtotal countとcanonical indexが確定した場合だけGridも公開し、`row_count=ceil(total_count / column_count)`、`column_count=current committed layout`とする。count／index未確定中はitemを部分materializeせずcollection feedbackだけを示し、Grid／GridItemとspatial navigationを開始しない。resize／DPI／densityでtile column countが変わる場合はlayout generationだけを進め、GridItem row／column／boundsを更新してLayoutInvalidatedを発行するが、query generation、Stable target、selection、AI semantic hashを変えない。
- list row／asset tileはListItem＋SelectionItem＋ScrollItem、default Actionがある場合だけInvokeを公開する。tile modeのrealized itemはGridItemのrow／columnをcurrent committed layoutから返す。rename中はListItem control viewのEdit childとValueを公開し、content viewはListItem一件のままにする。
- `FindItemByProperty`はAutomationId、Name、IsSelectedを扱い、AutomationIdはcollection instance＋Stable targetからpeer-uniqueに導いてlabel／indexを使わない。virtualized matchはUI changeなしでVirtualizedItem placeholderを返す。`Realize`後にfull property／patternを公開し、viewport／generation driftには`UIA_E_ELEMENTNOTAVAILABLE`を返す。
- `Selection.GetSelection`はrealized selected itemだけをboundedに返し、rootのItemStatus／DescribedByへselected総数とhidden／unrealized件数を示す。残りは`FindItemByProperty(IsSelected)`で順次取得し、全itemをmaterializeしない。

#### 15.6.4 `widget.collection.list-row@1`

`list-row@1`はflat collectionの一Stable targetを高密度に表示、選択、open／revealし、許可されたrename Actionへ接続する`composite` Patternである。Asset Browserではname、kind、logical directory、readiness、Diagnostic要約を表示し、Source／Import revision、dependency等の詳細はInspectorへ投影する。column headerまたはcell focusを持たず、columnのように揃って見えてもTable／DataGridへ偽装しない。

##### Anatomy、size、overflow

| slot | 必須 | 内容と規則 |
|---|---|---|
| `row-root` | required | semantic role=`list_item`。一Stable target、focus、selection、state overlayの合成範囲 |
| `validation-rail` | optional | error／warningの2 lu left overlay。layout幅、target、Actionを持たない |
| `type-icon` | required | registered target-kind icon。thumbnailを使わず、ItemTypeへ同じlocalized kindを公開 |
| `primary-label` | required | current nameを一行。full textをSemantic／UIA Nameへ保持 |
| `secondary-metadata` | optional | kind、logical directory、readinessのうちView descriptorが選ぶ最大二項を一行。列header、target identity、Diagnostic reasonを代用しない |
| `status` | optional | validation、lock／read-only、runtime residency、freshness、approvalをicon＋短いlabelで別々に表示 |
| `rename-editor` | conditional | rename中だけ`widget.input.text-field@1`をprimary label位置へ表示し、current nameとlocal draftを分離 |
| `proposal-marker` | optional | current rowを置換しないAI dash＋`AI提案`。full Diff／reviewはProposal Cardへ遷移 |

row高、body typography、padding、iconは§15.4のTree／List row token、すなわちcompact／standard／comfortableの24／28／32 luを使う。

| token | 値 |
|---|---:|
| `size.list-row.content-min` | 240 |
| `size.list-row.label-min` | 96 |
| `size.list-row.metadata-max` | 192 |
| `size.list-row.status-min` | 56 |

- `type-icon + primary-label + status`を先に確保し、不足時はsecondary metadataを省略する。Actionは初めからEnter、context menu、Command paletteに置き、hover時だけ現れるrow actionを作らない。statusはicon＋短いlabelを維持し、primary labelは96 luまで縮めてellipsisする。
- 240 lu未満はrowをwrap、重畳、clipせずtyped layout constraintとする。containerはPanel tab化またはhorizontal scrollで回復し、stateやActionを黙って消さない。
- 日本語名はUI face、identifier／path／Asset IDはcode faceを使用する。同一baselineを維持し、ellipsis後もcopy、Semantic name、Stable targetを短縮しない。

##### Interaction、state、AI

`list-row@1`はinteraction、availability、editability、selection、authority、provenance、validation、freshness、approval、live-edit axisを使い、value=`set`、task=`idle`とする。thumbnail／background Jobのtask stateをlist rowへ載せない。

- selection、focus、query selection、keyboard、view switchは`asset-view@1`をそのまま消費し、list-row固有modelを作らない。
- `F2` renameはexact selection count=1かつactive target=focused targetの場合だけlocal draftを開始し、multi／query selectionではdisabled reasonとbulk rename候補だけを返す。Enterだけがvalidation後に一Rename ChangeSetを要求する。Escape、selection／query／Project revision変更、Panel closeはsilent Commitせずdraftを破棄し、invalid renameをtrim、fallback名、自動suffixで成功させない。
- rowはinternal dragのStable ref payload sourceになれるが、既定ではdrop targetにならない。pointer dragはCommand policyが定めるbounded exact selectionだけを許し、query selectionまたは上限超過はdisabled reasonとbulk move候補を返す。Assetへのdropをreplace／reimportへ推測変換せず、folder moveは`tree-row@1`、external importはcontainer drop surface、明示Operationはregistered Actionを使う。
- `selected + focused + runtime + warning`はselection fill、focus ring、`Runtime` badge、warning rail＋triangle＋`警告` labelを別layerで示す。`locked`はowner／reasonを示してrename／moveを拒否し、select／copy／revealを維持する。
- AI proposalはcurrent name、kind、path、thumbnailを置換しない。stale proposalはclock＋base／current revisionとrebase／discardだけを示し、acceptをdisabledにする。
- AI、MCP、CLIはUIA、label、row index、boundsからActionを作らず、Semantic SnapshotにあるStable targetと`EditorSemanticActionV1`だけを使う。

#### 15.6.5 `widget.collection.asset-tile@1`

`asset-tile@1`はAsset Browser tile modeで一Stable Asset targetと非正本thumbnail previewを表示する`composite` Patternである。thumbnail、type icon、display name、Runtime residency、validation、proposalを同じAssetへ束縛するが、thumbnail image、grid coordinate、cache keyをAsset identityまたは品質判定に使わない。

##### Asset tile anatomy

| slot | 必須 | 内容と規則 |
|---|---|---|
| `tile-root` | required | semantic role=`list_item`。Stable Asset target、focus、selection、state overlay、GridItem boundsの合成範囲 |
| `validation-rail` | optional | error／warningの2 lu left overlay。thumbnail frameを着色してseverityを代用しない |
| `thumbnail-frame` | required | fixed square preview surface。asset aspectを保つcontain、1 lu border、no crop。preview edit／turntable inputを持たない |
| `thumbnail-content` | conditional | current preview revisionと一致するbounded image、または`Preview outdated`を常時併記したlast-valid imageだけ表示。ImageはUIA／AI semantic targetにしない |
| `thumbnail-feedback` | conditional | queued／running／failed／outdated／unavailableをregistered feedback child、icon、短いtextで表示。Asset validationを代用しない |
| `type-icon` | required | ready preview時はframe左上、previewなしでは中央に表示。ItemTypeへ同じlocalized kindを公開 |
| `selection-indicator` | optional | selected時のcheck shape。selection fill／outlineと併用し、Actionまたはcheckbox Toggleにしない |
| `primary-label` | required | current Asset name。metadataなしなら最大二行、metadataありなら一行。full nameをSemantic／UIA Nameへ保持 |
| `secondary-metadata` | optional | kindまたはreadinessの一項を一行。path／revision／license全文をtileへ詰め込まずInspectorへ投影 |
| `status` | optional | runtime residency、lock、freshness、approvalをthumbnail右下のicon＋短いlabelで別表示。validation／selectionを代用しない |
| `rename-editor` | conditional | rename中だけtext-fieldをlabel領域へ置き、tile heightとGridItem positionを変えない |
| `proposal-marker` | optional | AI dash＋`AI提案`。current thumbnailとnameを提案後の見た目へ置換しない |

##### Size、grid layout、thumbnail

値は`editor_ui_scale=1.00`の`ui_lu`で、tileはstretchしない。padding、group gap、typographyは§15.4の同density tokenを使う。

| Profile | tile size | thumbnail frame | label area |
|---|---:|---:|---:|
| compact | 96 × 124 | 72 × 72 | 36 |
| standard | 120 × 158 | 96 × 96 | 40 |
| comfortable | 144 × 192 | 120 × 120 | 44 |

- `column_count=max(1, floor((available_width + group_gap) / (tile_width + group_gap)))`とし、row-major canonical sort orderで左上から配置する。tile／gapをstretchせず余りはinline endへ残す。masonry、thumbnail aspect依存のrow高、Job完了順による並替えを禁止する。
- label areaはprimary一行＋secondary一行、secondaryなしならprimary二行とする。超過はgrapheme境界でellipsisし、full nameをaccessible name、tooltip、copyへ保持する。
- overlay配置はtype icon=thumbnail左上、selection check=右上、preview feedback=左下、lock／Runtime status stack=右下、validation=left rail＋severity icon、proposal／stale=outer dash／dot＋label横のAI／clockとする。statusが重なる場合はsecondary metadata行を同じ順序のstatus rowへ置換し、重畳、状態省略、一つのbadgeへの統合をしない。
- thumbnailはasset全体をcontainし、crop、aspect distortion、preview成功の偽装をしない。透過、3D、Audio、Font等のpreview生成意味はAsset lifecycleが所有し、tileはrevision付きthumbnail refだけを消費する。
- C1 tileはdeterministic static thumbnailだけを表示し、real-time thumbnail、tile内3D orbit、waveform scrub、animation playbackを行わない。これらはPreview Panelへ遷移するregistered Actionとし、tile virtualizationとUI frameを汚染しない。
- current Asset revisionとthumbnail revisionが異なる場合はlast-valid imageを`Preview outdated` label付きで維持してbackground Jobを要求する。Job runningは§15.5のprogress rule、reduced motionではstatic stage indicatorを使う。失敗時はregistered type icon＋`Preview failed`＋retry／Problems Actionを表示し、Asset自体をvalidation errorへ昇格しない。
- thumbnail outputをAI Contextへ自動添付しない。Visual評価が必要な場合だけredaction済み`VisualEvidenceSnapshotV1`を別artifactとして明示取得し、画像からAsset ID、Action、permissionを導出しない。

##### Interaction、state、UIA、AI

`asset-tile@1` rootはinteraction、availability、editability、selection、authority、provenance、validation、freshness、approval、live-edit axisを使い、value=`set`、task=`idle`とする。thumbnail Jobのqueued／running／failedは`thumbnail-feedback` childのtask axisであり、Asset rootのvalidation／freshnessへ昇格しない。

- selection、spatial keyboard、view switch、query selection、virtualizationは`asset-view@1`をそのまま消費する。tile内overlayはTab stopにせず、Enterはdefault open、F2はrename、context menu／Command paletteは同じregistered Actionを使う。
- tileはStable ref drag sourceになれるが既定drop targetではない。selected集合をdrag開始時にfreezeし、thumbnail、tile coordinate、visual orderをpayloadに含めない。
- `selected + focused + runtime + warning`ではselection fill＋check、focus ring、`Runtime` badge、warning rail＋triangle＋`警告` labelを同時表示する。High Contrastでもshape、icon、textの最低二つを残す。
- `proposal(ai) + approval.pending + stale`ではAI dashed outline、`AI提案`、clock、base／current revisionを示し、current thumbnail／nameを維持してacceptをdisabledにする。
- UIAはtileをListItem＋SelectionItem＋ScrollItem＋GridItemとして公開し、primary Actionがある場合だけInvokeを追加する。thumbnail Imageはraw treeの装飾（IsControlElement=false、IsContentElement=false）とし、Nameはfull Asset name、ItemTypeはkind、ItemStatusはreadiness、validation、runtime、preview statusのbounded summaryとする。
- AIはlist viewと同じStable target／Action projectionを読み、GridItem row／column、thumbnail pixel、visual similarity、UIAからAsset操作権限を得ない。

#### 15.6.6 `widget.collection.table-view@1`

`table-view@1`はheaderを持つread-only data collectionのquery、column schema、cell focus、row selection、sort、scroll、virtualization、DataGrid／Table providerを所有する`surface` Patternである。Asset Browser Columns viewでは一行が一Stable Asset target、各cellが一Stable `column_id`のtyped projectionを表す。C1はcell内の直接編集、spreadsheet selection、row grouping、free column reorder、merged cellを持たない。Asset renameは`asset.name` cellから`table-row@1`の一件renameへ遷移する。

```text
EditorTableColumnSchemaV1
  column_id                         # stable namespaced ASCII ID
  label_key                         # localized display only
  value_kind                        # text | identifier | enum | status | revision | count
  binding_ref
  formatter_ref
  comparator_ref?
  text_role                         # ui | code | numeric
  alignment                         # start | end
  min_width_lu
  default_width_lu
  max_width_lu
  visibility                        # required | default | optional
  pinning                           # pinned_start | scroll
  sortable
  null_state_policy_ref

TableCellValueV1
  state                             # value | pending | unknown | unavailable | redacted | not_applicable
  typed_value?                      # state=valueだけ
  reason_key?                       # unavailable／redactedはrequired
  source_revision?

EditorTableSortV1
  keys[1..3]
    column_id
    direction                       # ascending | descending
    comparator_ref
  stable_tie_breaker                # Stable Asset ID ascending固定
  query_revision
```

- `column_id`、`binding_ref`、`value_kind`、`comparator_ref`はschema identityであり、localized header、visible index、width、screen xを代用にしない。unknown／duplicate column、型不一致formatter、`min_width_lu <= default_width_lu <= max_width_lu`を満たさない値、required columnのhide、non-sortable columnのsortをload時に拒否する。
- `TableCellValueV1.state`を空文字、em dash、`0`へ畳み込まない。表示は`pending=計算中`、`unknown=不明`、`unavailable=利用不可`、`redacted=非表示`、`not_applicable=対象外`とし、iconまたはtextを常時伴う。実値の空文字と0は`state=value`のtyped valueである。
- sortはdisplay textをparseせずquery serviceのversioned typed comparatorを使う。ascending／descendingとも実値を先に並べ、その中だけ方向を反転する。非実値bucketは`pending → unknown → unavailable → redacted → not_applicable`の順で末尾へ固定し、最後にStable Asset ID ascendingで決定的に結ぶ。表示locale、DPI、density、column widthで順序を変えない。
- committed sortの既定は`asset.name ascending`＋Stable IDである。sort request中はlast committed rowsを維持してquery summaryへ`更新中`とrequest generationを示し、完了した同generationのprojectionだけをatomicに置換する。old／cancelled responseを部分適用せず、selection、focus target、scroll anchorをStable refで再解決する。

##### Asset Columns schema

C1のcolumn順は次のcanonical順で固定し、default fiveだけを初期表示する。C1でdrag reorderは提供せず、header menu／Column chooserでoptional columnの表示だけを切り替える。`asset.name`はrequired、pinned-startであり、非表示またはscroll領域への移動を拒否する。

| `column_id` | value kind／text | visibility | min／default／max lu | sort |
|---|---|---|---:|---|
| `asset.name` | text／ui | required | 160／240／480 | yes、default |
| `asset.kind` | enum／ui | default | 80／112／192 | yes |
| `asset.readiness` | status／ui | default | 96／120／176 | yes |
| `asset.diagnostics` | status／ui | default | 96／144／240 | yes |
| `asset.target-residency` | status／ui | default | 112／144／224 | yes |
| `asset.logical-directory` | identifier／code | optional | 160／240／480 | yes |
| `asset.source-revision` | revision／code | optional | 112／160／240 | yes |
| `asset.import-revision` | revision／code | optional | 112／160／240 | yes |
| `asset.active-generation` | revision／code | optional | 112／160／240 | yes |
| `asset.license` | enum／ui | optional | 96／144／240 | yes |
| `asset.dependency-count` | count／numeric | optional | 80／96／128 | yes |

- column visibilityとwidthはversioned Workspace presentation stateに保存し、Project／Asset Documentへ書かない。fresh Workspaceは五つのdefault列を表示する。既存Workspaceのschema更新ではunknown column stateを理由付きで破棄し、新しいdefault／optional columnはUser配置を乱さないようhidden、新しいrequired columnだけdefault widthで追加する。width、visibility、horizontal scrollはsemantic hash、Project revision、Proposal freshnessを変えない。
- hidden sorted columnはsortから暗黙削除せず、query summaryに`非表示列: <label> ↑/↓ (priority)`を表示する。Column chooserでもsort priorityを併記し、sort解除は明示Commandだけにする。
- headerとrow高は§15.4のcompact／standard／comfortableで24／28／32 lu、bodyは同ProfileのBody size／lineを使う。UI textはUI face、path／revision／identifierはcode face、countはnumeric tabular glyph＋end alignmentとし、同じcell内でfont roleを文字列推測しない。
- pinned Nameを除く列は一つのhorizontal scroll領域を共有し、vertical scroll時にheaderを固定する。200% scaleでdefault fiveが収まらなければstatus、label、columnを省略せずhorizontal scrollへ移し、Nameの160 luと各列のmin widthを破らない。

##### Anatomy、focus、selection

| slot | 必須 | 内容と規則 |
|---|---|---|
| `table-root` | required | semantic role=`data_grid`。collection／query revision、column schema、sort、selection、focus generation、DataGrid UIA root |
| `column-header-band` | required | visible columnごとに`column-header@1`をcanonical順で一件。headerをbody rowに偽装しない |
| `body-viewport` | required | realized `table-row@1`とcell provider。empty／loading rowを作らない |
| `query-summary` | required | committed／pending sort、result count、query freshness、hidden sorted columnをtextで表示 |
| `selection-summary` | optional | exact／query selection総数とfilter-hidden件数 |
| `collection-feedback` | optional | empty、loading、query／capacity errorをregistered feedback Patternで表示 |
| `scrollbars` | conditional | verticalとunpinned horizontal領域。`scrollbar.chrome.editor@1`を参照し、keyboard／UIA Scrollと同じlogical offset |

- Gridは一つのcomposite Tab stopである。`Tab`はlast focused cell、なければactive targetの`asset.name` cellへ入り、次の`Tab`でGridを出る。cellのText／iconは別Tab stopにせず、row ActionはEnter、F2、Menu key／`Shift+F10`、Command paletteから呼ぶ。
- `Left／Right`は同じrowの前後visible cell、`Up／Down`は同じcolumnの前後rowへfocusを移す。headerをlogical focus rowとして含め、first data rowの`Up`は同column header、headerの`Down`はfirst data rowへ移る。端ではwrapしない。`Home／End`は同rowのfirst／last visible cell、`Ctrl+Home／Ctrl+End`はfirst／last query rowのfirst／last cell、`PageUp／PageDown`は一viewportを移動する。
- data cellのplain `Up／Down`はfocus先rowを一件選択し、`Ctrl+Up／Down`はfocusだけ、`Shift+Up／Down`はcanonical query orderのrangeを延長する。`Left／Right`はselectionを変えない。`Ctrl+Space`または`Shift+Space`はfocused rowをtoggle／selectし、`Ctrl+A`はasset-viewと同じrevision-bound query selectionを作る。cell selectionまたはcolumn selectionは作らない。
- data cellの`Enter`はfocused rowのregistered default Action、`F2`は`asset.name` cellかつexact one selectionでだけrenameを開始する。他column、multi／query selection、locked／read-onlyではdisabled reasonと代替Actionを返す。rename中だけName cellに`text-field@1`を置き、Escapeでcell navigationへ戻る。
- headerの`Enter`／Invokeはsingle sort、`Shift+Enter`はmulti-sort、`Shift+F10`／Menu keyはsort／visibility／width menuを開く。header action中もrow selectionとactive targetを変更しない。

##### Table virtualization、Semantic、UIA、AI

- 10万Assetでvisible row＋前後2 viewportだけをmaterializeし、realized rowではvisible全列のcell providerを作る。11列はbounded schemaでありhorizontal cell virtualizationを行わない。hidden columnはColumnCountとUIA treeへ含めず、visibility変更時はcolumn layout generationを進めてStructureChangedとLayoutInvalidatedを各一回発行する。
- `RowCount=total_count`、`ColumnCount=visible column count`はcommitted queryだけから返す。`Grid.GetItem(row, column)`は該当Stable target／column IDのcell providerを返し、off-screen rowはVirtualizedItem placeholderを経て`Realize`またはScrollItemで表示する。空／非適用cellでもproviderを返し、`TableCellValueV1.state`をItemStatusへ公開する。
- UIA rootはDataGrid＋Grid＋Table＋Selection、必要時Scroll／ItemContainerを公開する。HeaderはHeader、各`column-header@1`はHeaderItem、row rootはDataItem＋SelectionItem＋ScrollItem、default Actionがある場合だけInvoke、cellはDataItem＋GridItem＋TableItemとする。`GridItem.Row／Column`はzero-based committed query／visible schema、`TableItem.GetColumnHeaderItems`は同じ`column_id`のHeaderItem一件を返す。
- row rootのNameはfull Asset name、ItemTypeはkind、ItemStatusはselection外のbounded state summaryである。cell Nameはlocalized formatted value、AutomationIdはcollection instance＋Stable target＋column IDからpeer-uniqueに導く。`redacted` cellはreason以外のValue、raw text、sort keyをUIA／AIへ公開しない。
- filter／sort／visibility／query／Project generationが変わったold row、cell、placeholderは`UIA_E_ELEMENTNOTAVAILABLE`とし、row indexまたは以前のGrid coordinateで別Assetへretargetしない。Selectionはrow rootだけが所有し、cell providerはselectionを捏造しない。
- Semantic Snapshotはcolumn schema ID、committed sort、Stable target、`column_id`、typed value state／revisionを公開する。width、physical bounds、visual column index、localized headerはAI projectionから除外する。AI、MCP、CLIはtyped collection query／sort refと`EditorSemanticActionV1`を使い、UIA、header Invoke、screen x、cell text parseで操作または権限を得ない。

#### 15.6.7 `widget.collection.column-header@1`

`column-header@1`は一Stable `column_id`のlabel、sort状態、priority、visibility／width Commandを表す`composite` Patternであり、AssetまたはProject writeを所有しない。

| slot | 必須 | 内容と規則 |
|---|---|---|
| `header-root` | required | semantic role=`column_header`。column ID、sortability、width bounds、HeaderItem UIA provider |
| `label` | required | localization keyから解決した一行。ellipsisしてもfull accessible Nameとcolumn IDを維持 |
| `sort-indicator` | conditional | ascending／descending arrow＋localized text。色だけで方向を示さない |
| `sort-priority` | conditional | multi-sort時だけ`1`～`3`のnumeric badge＋accessible status |
| `menu-indicator` | required | sort、show／hide、width resetへ到達できることを示す。hover-only child buttonにしない |
| `resize-boundary` | conditional | scroll列のinline endに1 lu separatorと全header高×12 luのpointer hit strip。独立Tab stop／Actionにしない |

- plain click／Enterはそのcolumnだけをsortし、未sortならascending、既にsole keyならascending／descendingを交互に切り替える。`Shift+click／Shift+Enter`は最大3 keyへ追加し、既存keyなら方向をtoggleする。4件目は置換せず`sort-key-limit` reasonを返す。sort解除、priority変更、全sort resetはheader menuの明示Commandで行う。
- resizeはpointer down時のcommitted widthをlocal draftとしてcaptureし、drag中はmin／maxへclampしたlayout previewだけを更新する。releaseでWorkspace presentation stateへ一件Commit、Escape／capture lossで開始幅へ戻す。double-clickはcontent測定でなくschemaのdefault widthへ戻し、10万row scanを行わない。
- focused headerの`Alt+Left／Alt+Right`は8 lu、`Shift+Alt+Left／Right`は32 lu単位で狭める／広げる。Menuにも同じCommandと`既定幅に戻す`を置く。RTLでもLeft／Rightは物理方向でなくinline sizeの減少／増加としてlocalizeしたshortcut descriptionを公開する。
- UIAはHeaderItem＋Invoke（sortableの場合）＋Transform（resizableの場合）を公開し、Transformは`CanResize=true`、`CanMove=false`、`CanRotate=false`とする。`Resize(width,height)`はheight変更を拒否しwidthだけをluへ変換して同じclamp／Workspace Commandへ収束する。ItemStatusは`未sort`または`ascending/descending, priority n`をlocalized textで返す。
- normal／hover／pressed／focusは§15.3を消費し、sort変更とresizeは0 ms、header hoverだけ83 msとする。drag幅にeasingを入れず、reduced motionで意味を変えない。High Contrastではsystem text、separator、focus outline、arrow shape、priority textを残す。

#### 15.6.8 `widget.collection.table-row@1`

`table-row@1`は一Stable targetをrow selection、cell focus、default open、許可されたrenameへ接続する`composite` Patternである。cellは`column_id`と`TableCellValueV1`を持つsemantic leafであり、C1では独立Widget Patternまたは編集Controlにしない。

| slot | 必須 | 内容と規則 |
|---|---|---|
| `row-root` | required | semantic role=`data_row`。Stable target、selection、row state overlay、DataItem provider |
| `validation-rail` | optional | error／warningの2 lu left overlay。diagnostics cellの値を置換しない |
| `cells[]` | required | visible schema順に一件。column ID、typed value state、focus、GridItem／TableItem provider |
| `name-cell-content` | required | type icon＋current Asset name。row accessible Nameのsource |
| `cell-status` | conditional | pending／unknown／unavailable／redacted／not-applicableをicon＋短いlocalized textで表示 |
| `rename-editor` | conditional | Name cellだけ。`text-field@1`でcurrent valueとlocal draftを分離 |
| `proposal-marker` | optional | row外周のAI dash＋Name cell内`AI提案`。current cell valueを提案値へ置換しない |
| `row-status-summary` | optional | stale clock、approval、lockをName cell末尾にicon＋短いlabelで別表示 |

- base row → selected fill → validation rail／stale dotted edge → proposal dash／label → Runtimeを`asset.target-residency` cell内のplay glyph＋`Runtime` text → active cellの2 lu focus ringの順で重ねる。validation、selection、proposal、Runtime、freshnessをrow tint一色または一badgeへ統合しない。
- `selected + focused + runtime + warning`はselected row fill、focused cell ring、warning rail＋triangle＋`警告`、residency cellのplay glyph＋`Runtime`を同時表示する。`proposal(ai) + approval.pending + stale`はcurrent cellsを保ち、AI dash、`AI提案`、clock、base／current revisionを示してacceptをdisabledにする。
- focused cellだけfocus ringを持ち、selected rowの全cellをfocusedに見せない。hoverはrow baseへ83 msで適用するが、selected fill、cell focus、warning railを覆わない。pressedはpointerが同row内にある間だけで、open／rename成功を表さない。
- cell overflowは一行ellipsis、full typed valueはcopy／Semantic／UIAへ保持する。statusはicon＋textを先に確保し、値をclipして状態を残す。column min widthでも状態が収まらない場合はtooltipだけに逃がさず、cell内の短いregistered labelとrow ItemStatusへ同じ意味を残す。
- locked／read-onlyはselect、copy、focus、reveal、openを維持し、rename／move等のmutating Actionだけをowner／reason付きで拒否する。disabled rowはfocus／selection対象にせずreasonをfeedbackへ示す。stale proposalはrow自体のcurrent Asset freshnessと混同せずproposal Actionだけを拒否する。
- rowはlist／tileと同じStable ref drag payload sourceになれるが、cellは独立drag source／drop targetにならない。drag開始時にbounded exact selectionをfreezeし、query selection、column ID、cell value、Grid coordinateをpayloadへ含めない。Asset上のdropをreplace／reimportへ推測せず、同じregistered Actionとpreview validationを使う。
- pointer、keyboard、UIA、AIのopen／rename／context Actionは同じStable target、expected Project revision、registered Commandへ収束する。row、cell、header、Grid coordinate、formatted value、visual orderからProject writeを生成しない。

#### 15.6.9 Required collection conformance scenarios

| scenario ID | 必須内容 |
|---|---|
| `scenario.widget.asset-view.switch-preserve@1` | list↔tile↔columnsでselection、focus target、focus column、range anchor、scroll anchor、filter／sort／Project revision不変 |
| `scenario.widget.list-row.selection-keyboard@1` | plain／Ctrl／Shift pointer、Up／Down／Ctrl／Shift keyboard、hidden selection summaryが同じStable ref集合へ収束 |
| `scenario.widget.asset-tile.spatial-keyboard@1` | Left／Right wrap、Up／Down clamp、RTL、resize／DPIでcolumn変化後もfocus／selection identity driftなし |
| `scenario.widget.asset-view.query-virtualization@1` | 10万Asset、query selection、前後2 viewport、ItemContainer／VirtualizedItem、view generation失効、全item未materialize |
| `scenario.widget.asset-item.rename-drag@1` | list／tile／columnsのF2、Japanese IME、Enter Commit、cancel／drift、Stable ref drag source、implicit replace 0件 |
| `scenario.widget.asset-tile.thumbnail-lifecycle@1` | ready／outdated／queued／running／failed／unavailable、last-valid、retry、reduced motion、Asset validationとの分離 |
| `scenario.widget.asset-item.state-overlay@1` | list／tileの`selected + focused + runtime + warning`、locked、disabledをLight／Dark／四High Contrastで識別 |
| `scenario.widget.asset-item.ai-proposal-stale@1` | current name／thumbnail不変、AI provenance、approval pending、stale、accept拒否、rebase／discard |
| `scenario.widget.asset-item.density-content@1` | 三density、`editor_ui_scale=1.00／2.00`、長い日本語名、64文字ASCII identifier、Windows path、異なるaspectでclip／target driftなし |
| `scenario.widget.table-view.schema-null-sort@1` | 11列、six value state、typed comparator、最大3 sort、hidden sorted column、Stable ID tie-break、async generation失効を検証 |
| `scenario.widget.table-view.keyboard-selection@1` | header／cell arrow、Home／End、Page、Ctrl／Shift selection、query select-all、F2 Name限定、Tab一回を検証 |
| `scenario.widget.column-header.resize@1` | pointer capture／cancel、8／32 lu keyboard、default reset、UIA Transform、min／max、Workspace-only、10万row scan 0件 |
| `scenario.widget.table-row.state-overlay@1` | selected＋focused cell＋runtime＋warning、locked、proposal＋approval.pending＋staleをLight／Dark／四High Contrastで別axisとして識別 |
| `scenario.widget.table-view.virtualization-uia@1` | 10万row×visible 11 cell、DataGrid／Header／DataItem／Grid／Table、ItemContainer／VirtualizedItem、empty cell provider、generation失効、全row未materialize |
| `scenario.widget.asset-view.uia-ai-separation@1` | Pane／MultipleView、List／ListItem、tile Grid／GridItem、columns DataGrid／Header／Grid／Table、Selection、virtualizationを検証し、AIのUIA／thumbnail／coordinate／display text parse利用0件 |

`fixture.asset-browser.collection@1`は同じ六Assetをlist／tile／columnsで表示する。ready、`selected + focused + runtime + warning`、locked、`proposal(ai) + approval.pending + stale`は三Viewで検証し、thumbnail running／failedはtileだけ、six cell value stateとmulti-sort priorityはcolumnsだけに表示して他Viewのrow高／Asset validationを変えない。10万Asset fixture、Light／Dark、四High Contrast、三density、`editor_ui_scale=1.00／2.00`で同じStable target、selection、Diagnostic、Action可否へ収束するまで`asset-view@1`、`list-row@1`、`asset-tile@1`、`table-view@1`、`column-header@1`、`table-row@1`をclosedと扱わず、columns Capabilityをactivateしない。

#### 15.6.10 `widget.diagnostics-source.diagnostic-view@1`

`diagnostic-view@1`はProblems、Console、Build、Profiler、Replay／CausalityがDebugging Ownerのbounded queryを同じ規則で投影するcollection surfaceである。一つのinstanceは`diagnostic_set`または`log_stream`の一方だけをmaterializeし、logとDiagnosticを一つのvisual listへ混在させない。BuildはTask stage、Diagnostics、Logを別領域／tabとして合成し、Profilerはcounter／span／sampleをCanvas／Tableへ投影してcanonical threshold breach／anomalyだけを`diagnostic_set`へ出す。Task失敗、AI仮説、UI chart heuristicから`MirakanDiagnosticV1`を合成しない。

```text
EditorDiagnosticsCollectionStateV1
  collection_key
  projection_kind           # diagnostic_set | log_stream
  source_query_ref
  source_generation
  total_count
  filter_ref
  canonical_order           # diagnostic_priority | source_sequence
  selection_ref
  active_item_ref?
  focus_item_ref?
  range_anchor_ref?
  scroll_anchor_ref?
  follow_tail               # off | on。log_streamだけ
  display_pause_cursor?     # log_streamだけ
  clear_before_sequence?    # Workspace-only view boundary
```

- `diagnostic_set`のitem identityはowner発行のimmutable Result／Receipt／Event refとcanonical Diagnostic record hash、`log_stream`はSession refとimmutable sequenceまたはownerのdedup identity、gapはqueryが返したrange identityを使う。display message、localized text、list index、timestamp単独、Panel内順序からidentityを作らない。
- `diagnostic_set`の既定順は`blocking > error > warning > info`、同severity内はcanonical target／location、Diagnostic code、source record identityで安定化する。`log_stream`はsource sequence昇順を固定し、UIがsecond dedup、並べ替え、timestamp補正を行わない。current queryに存在しないDiagnosticをUIが`resolved`／`acknowledged`へ推測せず、履歴表示はownerのrevision／recorded queryだけで行う。
- filterはseverity、category、domain、phase、Target、recorded／current、gap／redactionをtyped fieldとして持つ。text searchはbounded query条件でありidentityではない。filter／sort／follow-tail／display pause／clear boundaryはWorkspace presentation state、Diagnosticのseverity、Target、Evidence、revision、query bindingはsemantic stateとして別hashへ入れる。
- anatomyは`query-toolbar`、`filter-summary`、`collection-viewport`、`new-items-status`、`gap-summary`、`details-view`、`feedback`、`scrollbar.chrome.editor@1`を使うscrollbarとする。幅960 lu以上はlist最小560 lu＋details 360 lu、960 lu未満はlist／detailsを同時圧縮せず相互排他的な内部viewへ切り替える。Enterまたは登録済み`Details`でdetailsへ進み、Escape／Backでselection、focus、scroll anchorを保持してlistへ戻る。
- query pageは最大256 record／256 KiB、cacheはDebugging Ownerのhard limitである4096 record／4 MiBを超えず、visible range＋前後2 viewportだけをrealizeする。page取得後にsource generation、query、Session、Project revisionが変わった場合は結果を破棄し、indexと表示位置を旧itemへ再束縛しない。
- Up／Down、Home／End、PageUp／PageDown、plain／Ctrl／Shift selection、Shift+F10、Ctrl+Cを共通とし、ProblemsではF8／Shift+F8を次／前のactive Diagnosticへ割り当てる。Enterは登録済みdefault revealまたはdetailsだけを実行し、Tabはcollectionへ一度だけ入り内部rowを順送りしない。
- Ctrl+Cはexact selectionを最大256 record／256 KiBでcopyし、redaction、gap、recorded／current contextを保持する。上限超過またはquery selectionはmaterializeせず、明示的なExport Taskへ送る。Ctrl+Aはownerがrevision-bound query selectionを表現できる場合だけcurrent filter全件を選ぶ。
- `log_stream`のfollow-tailは明示stateである。Userの上scrollまたはPageUpで`off`にしてanchorを固定し、新着件数をstatusへ蓄積する。Endまたは`最新へ`で`on`へ戻す。`表示一時停止`はdisplay cursorだけをfreezeしrecordingを継続する。`表示をクリア`は`clear_before_sequence`を更新するだけでcanonical storeを削除しない。
- live append、filter結果更新、gap挿入は0 msで適用し、rowをslide／fade／pulseさせずscroll anchorを維持する。list↔detailsだけ標準画面切替167 msを使用でき、reduced motionでは0 msにする。
- UIA rootはList＋Selection＋Scroll＋ItemContainer、itemはListItem、gapは非選択DataItemとする。severity、recorded／current、Target、gap／redactionはNameだけへ連結せずItemStatus／DescribedByにも型から投影する。新着ごとに本文をannounceせず、別status regionを`LiveSetting=Polite`として最大1秒に一回、blocking／error件数、新着件数、stream／pause状態だけを集約通知し、secretまたはmessage bodyを読まない。
- AIは同じ`source_query_ref`、item ref、revision、Target、Evidence、gap／redactionをtyped projectionで受け取り、UIA、pixel、row index、display textを解析しない。AI仮説はAI PartnerのProposal／Evidenceとして保持し、canonical Validatorが発行するまでProblemsのseverityへ昇格しない。

#### 15.6.11 `widget.diagnostics-source.diagnostic-row@1`

`diagnostic-row@1`は一件の`MirakanDiagnosticV1`を投影する。row anatomyは`severity-rail`（2 lu）、`severity-icon`、`severity-label`、`code`、`summary`、`target-location`、`context-badge`、`remediation-indicator`、`state-summary`である。messageはUI face、code／location／revisionはCode face、count／sequenceはtabular numericを使う。

| severity | 必須の非色表現 | visual token |
|---|---|---|
| `info` | info-circle＋`情報` | secondary／info |
| `warning` | triangle＋`警告` | `validation.warning` |
| `error` | circle-x＋`エラー` | `validation.error` |
| `blocking` | octagon-stop＋`ブロッキング` | `validation.error`。`fatal`へ改名しない |

- 幅640 lu以上はinlineでcompact／standard／comfortableを40／44／52 lu、360～639 luはsummaryとcontextをstackして56／64／72 luとする。360 lu未満はPanelをtab化またはdetails viewへ切り替え、状態label、code、Targetをclipして一行へ押し込まない。
- base → selected fill → severity rail → stale dotted edge → Runtime／recorded context badge → focus 2 lu ringの順に重ね、selection、focus、severity、recorded／current、Runtime、freshness、AI proposalを一色／一badgeへ統合しない。Engine Diagnostic rowへAI proposal outlineを付けず、関連Proposalは別ref／Actionで示す。
- default actionは登録済みtarget revealまたはdetailsであり、canonical Target refとexpected Project revisionを必要とする。location欠損時にfile／lineを推測せず、remediation IDがない時にrepair Actionを生成しない。

#### 15.6.12 `widget.diagnostics-source.log-line@1`

`log-line@1`は`DebugLogRecordV1`または同等のDebugging Owner recordを一件投影する。sequence／time、severity、channel／category、Target、message、recurrence／suppressed count、recorded／current、redactionをslotとして持つ。UIはownerのdedup key、first／last sequence、count、suppressed countをそのまま消費し、独自collapse、message-string dedup、timestampによる並べ替えを行わない。

- compact／standard／comfortableは24／28／32 luの一行でwrapせず、horizontal scrollとdetails viewを用意する。画面に省略したmessageもcopy、Semantic、UIAではredaction後のfull contentを維持する。
- sequence／time／countはtabular numeric、channel／source locationはCode face、localized messageはUI faceを使う。clockがない場合は`時刻なし`、不確実な場合はclock-uncertain labelを表示し、別clockから補間しない。
- source severityは元labelを保持してclosed visual classへmapし、unknown severityをwarning／errorへ推測しない。selected fill、focus ring、Runtime、recorded／current、redactionは別slot／layerで示す。

#### 15.6.13 `widget.diagnostics-source.evidence-gap-row@1`

`evidence-gap-row@1`はquery completenessの欠損をinlineで可視化する非選択rowである。closed kindは`dropped | not_recorded | not_indexed | redacted_range | clock_uncertain`とし、channel／sequenceまたはtime range、known count、reason code、source query refを表示する。compact／standard／comfortableは32／36／44 lu、2 luのneutral dotted railとbroken-chain／shield／clock icon＋localized textを使い、空のlog line、正常な0件、warning Diagnosticへ偽装しない。

gapは原因のEvidenceではなくEvidence不足である。欠損値、件数、時刻、回復可否を推測せず、ownerが再query／capture Actionを登録した場合だけInvokeを公開する。copy／export／AI Contextではrange identityとreasonを保持し、gapを除外して完全なqueryと表示しない。

#### 15.6.14 Required diagnostics conformance scenarios

| scenario ID | 必須内容 |
|---|---|
| `scenario.widget.diagnostic-view.source-identity@1` | message／locale／filter／layout変更でもowner-issued identity、Target、revision、selectionが不変。index／timestamp／text由来identity 0件 |
| `scenario.widget.diagnostic-view.query-virtualization@1` | 256 record／256 KiB page、4096 record／4 MiB cache上限、前後2 viewport、generation失効、ItemContainer／VirtualizedItem、全件materialize 0件 |
| `scenario.widget.diagnostic-view.keyboard-selection@1` | plain／Ctrl／Shift、Home／End／Page、Problems F8、Enter、Shift+F10、Tab一回が同じitem ref／Actionへ収束 |
| `scenario.widget.diagnostic-view.live-follow-tail@1` | append 0 ms、scrollでfollow-tail解除、新着件数、Latest再開、表示pause中もrecord継続、clearでstore削除0件 |
| `scenario.widget.diagnostic-row.severity-context@1` | info／warning／error／blocking、selected＋focused、recorded＋Runtime、staleをicon＋text＋layerで識別し、`fatal`生成0件 |
| `scenario.widget.log-line.sequence-dedup@1` | first／last sequence、count、suppressed、unknown／uncertain clock、horizontal scrollを検証し、UI second dedup／reorder 0件 |
| `scenario.widget.evidence-gap-row.completeness@1` | dropped／not_recorded／not_indexed／redacted_range／clock_uncertainが非選択DataItemとなり、正常値／原因へ推測0件 |
| `scenario.widget.diagnostic-view.copy-export-redaction@1` | bounded exact copy、上限超過Export Task、query selection、redaction／gap保持、secret復元0件 |
| `scenario.widget.diagnostic-view.uia-notification@1` | List／ListItem／DataItem／Selection／Scroll／ItemContainer／VirtualizedItem、polite集約通知、message本文announce flood 0件 |
| `scenario.widget.diagnostic-view.ai-evidence-separation@1` | AIはtyped query／Evidence／gapを使用し、UIA／pixel／row index／display text parseと未検証仮説のDiagnostic化0件 |
| `scenario.widget.diagnostic-view.cross-panel-projection@1` | Problems／Console／Build／Profiler／Replayが同じcanonical recordを共有し、Task progress／metric sample／AI hypothesisとの型混在0件 |
| `scenario.widget.diagnostic-view.density-content@1` | 三density、Light／Dark／四High Contrast、200% Font、`editor_ui_scale=1.00／2.00`、長い日本語、ASCII code、Windows pathで意味／Target不変 |

`fixture.diagnostics.reference@1`は、Problemsにselected＋focusedのcurrent blocking、recorded＋Runtime warning、stale index errorとgap、Consoleにcurrent info、37回のrepeated warning、redacted record、128 sequenceのdropped gap、Buildにrunning Task stageと別のcanonical error／log、Profilerにchart timepointへ結び付くcanonical threshold Diagnosticを固定表示する。Profiler sampleはCanvas／Table、Build progressは`task-stage@1`に留める。未検証AI仮説はAI Partnerだけに表示する。Light／Dark、四High Contrast、三density、200% Font、`editor_ui_scale=1.00／2.00`、keyboard、Narrator／NVDA、UIA、AIで同じrecord、Target、Evidence、Action可否へ収束するまで四Patternをclosedと扱わず、Panel固有rowへfallbackしない。

#### 15.6.15 `widget.canvas.canvas-surface@1`

Canvas型はPanelごとに完全分離する方式でも、Scene用Transform surfaceへGame／UI Designerを押し込む方式でもなく、共通のinteraction shellとowner別typed projectionを合成する。`canvas-surface@1`はScene、Game、UI Designer、Map Presentation Previewのviewport root、focus、view state、pick generation、overlay composition、Semantic境界を所有する。Project Source、World／Camera／UiDocument、Runtime Session、Render targetの正本は所有しない。

```text
EditorCanvasSurfaceStateV1
  surface_key
  canvas_kind                 # scene_2d | scene_3d | game_preview | ui_designer | map_preview
  content_projection_ref
  content_generation
  project_revision
  context_query_ref
  selection_context_ref?
  authority_layer             # source | staging | derived | runtime
  target_profile_ref?
  coordinate_domain_ref
  view_descriptor_ref
  active_tool_ref             # registered select／transform／navigation／measure／owner tool ID
  transform_space             # not_applicable | world | local | parent | screen_ui
  pivot_mode                  # not_applicable | active | median | individual | bounds_center
  snap_profile_ref?
  overlay_set_ref
  preview_session_ref?
  runtime_time_ref?
  input_routing               # editor | game
```

- `content_projection_ref`、generation、Project revision、Stable target、owner-issued coordinate／view／overlay refが意味identityである。render texture、depth／ID buffer、camera matrix、screen／world coordinate単独、pixel color、draw order、gizmo handle、pointer positionからProject targetまたはActionを作らない。
- pan／orbit／zoom、editor camera、grid visibility、overlay visibility、active tool、snap toggle、view modeは`EditorUserState`のpresentation stateでありProject Undoへ入れない。Source Cameraを選択していてもSceneのeditor cameraは別物であり、`Align source camera to view`等の登録済みChangeSetがない限りSource Cameraを変更しない。
- `semantic_content_hash`はcontent projection、Project revision、selection、authority、Target、coordinate domain、overlayのtyped data ref、登録済みActionを含み、physical bounds、render texture、editor camera pose、zoom、pointer、pick bufferを除外する。AI Context Planが`WorldAuthoringContextV1.viewport_bounds`を明示選択した場合だけそのContext hashへview boundを束縛し、後のcamera移動で無関係なProposalをstaleにしない。
- `scene_3d`はpan／orbit／dolly／frame、`scene_2d`と`ui_designer`はpan／zoom／frameを持つ。navigationはselectionとSourceを変更せず、frame selectedは登録済みStable selectionのbounded spatial resultだけを使う。`game_preview`はRuntime Camera／Render Viewをread-only表示し、free cameraへ黙って切り替えない。
- anatomyは`context-bar`、`tool-strip`、`content-viewport`、`owner-overlay-layer`、`selection-layer`、`gizmo-layer`、`legend-status`、`input-routing-status`、`focus-boundary`とする。Scene／Gameの最小contentは640×360 luであり、確保できないときは低priority Panelをtab化する。context bar／tool stripはcompact／standard／comfortableで32／32／36 lu、全interactive targetは32×32 lu以上とし、200% Font／UI scaleでsecondary controlsをoverflow popoverへ移してもauthority、Target、selection、input routingを隠さない。
- texture composite → Source content → Derived／owner overlay → selected target → validation／AI proposal／Runtime marker → gizmo → Canvas chrome → surface focus ringの順に合成する。selectionはSourceの色を塗り替えず、AI proposalはcurrent geometryを置換せずviolet dash＋`AI提案`、Runtimeはcyan badge＋`Runtime`、validationはicon＋severity labelを使う。
- semantic state、selection、proposal、Runtime、gapは0 msで更新し、row型animation、pulse、particleを付けない。camera navigationとgizmo previewはUser inputへ直接追従し、補間で最終値を変えない。frame／axis snapは0 ms、popoverだけ167 ms、reduced motionではpopoverも0 msにする。

##### Panel variant boundary

| `canvas_kind` | 必須binding | 編集可否 |
|---|---|---|
| `scene_2d`／`scene_3d` | `AuthoringSelectionContextV1`、必要時`WorldAuthoringContextV1`、ownerのbounded Scene projection | Source／Staging targetだけ。Derived／Runtimeはread-onlyで、typed Transform primitiveへ収束 |
| `game_preview` | Play Session、Build、Target、Runtime Camera／Render View、Simulation Advance／render frame | Project editingなし。Game input routingだけを明示capture |
| `ui_designer` | exact UiDocument revision、layout／semantic result、design extent、scale／safe-area／locale／input profile | UiDocument Stable nodeとlayout kindが許すfieldだけ。Native Widget codeはEditor Processで実行しない |
| `map_preview` | Presentation SourceまたはDerived Artifact、Target、generation | 既定read-only。Source Intent Viewへの登録済み遷移だけ |

Game inputは`入力をゲームへ`を明示実行したfocused `game_preview`一件だけへrouteし、surface border、gamepad icon、`ゲーム入力中` textを同時表示する。Escapeまたは`入力をエディタへ戻す`でreleaseし、focus loss、Session／surface generation変更、GameHost crashでもfail-closedに`editor`へ戻す。UIA、AI、Canvas Pattern IDをGame input privilegeとして扱わない。

UI DesignerはUI ownerの同じC++ layout／text／semantic resultを表示し、approximate rectangle、Screenshot、Editor独自CSSでUiDocumentを再現しない。`absolute` layoutだけがanchor／offsetのfree dragを許す。`stack | flex | grid` childのfree move／resizeはdisabled reasonを示し、registered reorder、slot変更、親layout field編集へ送る。safe area／cutoutはTarget Profile／DisplaySnapshot由来ref、orientation、scale、localeとともに表示し、device名からinsetを推測しない。UiNativeWidgetはManifest fallbackによる静的projection、実code Previewは別Process GameHostだけとする。

##### Overlay、pick、selection

```text
EditorCanvasOverlayDescriptorV1
  overlay_ref
  owner_ref
  data_projection_ref
  data_generation
  coordinate_domain_ref
  layer_slot                  # content | derived | screen | chrome
  interaction_policy          # passive | pickable | editing_tool
  priority
  capacity_profile_ref
  legend_entries[]
  semantic_summary_ref
  action_refs[]
```

- overlayは登録済みdescriptorだけを使用し、Physics、Collider、Navigation、Light、Camera、VFX、Audio、safe area等のowner dataをgeneric property bagへ複写しない。`passive`はpointer capture不可、`pickable`はowner targetのreveal／selectionだけ、`editing_tool`は同一surfaceで一件だけactiveにする。priority同値はoverlay ref byte順で決定し、描画順からauthorityを推測しない。
- visibility／isolationはWorkspace presentation stateであり、Source objectのenabled／visible field、Runtime visibility、Cook結果を変更しない。可視性とpickabilityを別stateにし、hidden、unpickable、locked、read-onlyを一つのeye／lock状態へ統合しない。
- pointer pickは同じcontent、view、layout generationから作るDisposable `EditorCanvasPickSnapshotV1`を使い、候補を`gizmo handle → active editing overlay → Source target`の順、同class内はowner depth／paint orderとStable refで安定化する。stale generation、unknown ref、omitted candidateではselectionを変更しない。重なり対象はbounded `Select under cursor`へStable name／kind／ownerを列挙し、同じ場所の繰り返しclickで非表示順を暗黙cycleしない。
- plain clickは一件replace、Ctrl+clickはStable target toggle、Shift+clickはadd、empty clickはmodifierなしのときだけclearする。Scene 2D／orthographicとUI Designerのbox selectはownerのbounded spatial queryがある場合だけ有効にし、Scene 3Dのfrustum selectを2D rectangle包含へ近似しない。`game_preview`はProject selectionを変更しない。
- Pick SnapshotはHuman pointer dispatch専用でAI Context、UIA、clipboard、Project ChangeSetへ渡さない。AIは`AuthoringSelectionContextV1`、`WorldAuthoringContextV1`、`SceneSliceV1`、UiDocument node／layout projection、registered `EditorSemanticActionV1`を使用する。

#### 15.6.16 `widget.canvas.context-bar@1`

`context-bar@1`は現在のCanvasが「何を、どのrevision／Target／authority／coordinateで見ているか」を固定位置で示すcompositeである。左からCanvas kind／dimension、World／Scene／UiDocument／Session、Project revision、Target、authority、coordinate／camera、selection summary、live-edit／input-routingを並べ、利用しないslotは空欄でなくvariant schemaから除外する。

- Sceneは2D／3D、editor camera projection、world／local／parent、pivot、snap、Source／Staging／Derived／Runtimeを表示する。GameはSession、Build、Target、Runtime Camera、resolution／aspect、Simulation Advance／render frame、input routingを表示する。UI DesignerはDocument revision、design extent、scale policy、device／orientation、safe area、locale、input profileを表示する。
- selection、authority、Runtime、stale、input routingはicon＋short textを常時持ち、色だけのdotにしない。ID／revision／resolution／unitはCode face＋tabular numeric、日本語labelはUI faceを使う。
- context barのControlは既存button／combo／toggle PatternとCommand Registryを使う。表示値からCamera、Target、authorityを推測して切り替えず、選択肢がowner-issued refへ解決しない場合はdisabled reasonを示す。

#### 15.6.17 `widget.canvas.selection-overlay@1`

`selection-overlay@1`は一つの`AuthoringSelectionContextV1`をCanvasへ投影し、Source object、Runtime object、Proposalを所有しない。active targetは2 lu solid outline＋四隅bracket＋name／kind、追加selected targetは1 lu solid outline、Canvas keyboard focusはsurface外周2 lu focus ringで別表示する。High Contrastではselected targetをすべて2 luにし、active targetの四隅bracketと`選択中` ItemStatusを維持する。

- 3D silhouetteが取得不能または完全遮蔽ならowner-provided screen bound／pivot marker＋方向indicatorへ明示fallbackし、selectionを消さない。UI Designerは同じlayout generationのarranged rect、baseline、anchor／safe-area relationを使い、rendered pixel edgeを検出しない。
- 最大1024 Stable targetをSelection Contextどおり扱い、viewport外／omitted／outline unavailable件数をstatusへ示す。見えているoutlineだけをselection全体と偽装せず、deleted／permission-lost targetだけを理由付きでselectionから除外する。
- warning／errorはtarget markerのtriangle／circle-X＋label、AI proposalはcurrent outlineと別のviolet dash＋`AI提案`、Runtimeはplay glyph＋`Runtime`を重ねる。selected、focused、validation、proposal、Runtime、staleをoutline一色へ畳み込まない。

#### 15.6.18 `widget.canvas.gizmo@1`

`gizmo@1`はselected Source／Staging targetのtyped Transform／Ui layout fieldを一ChangeSetへ変換するpreview compositeである。`game_preview`、`map_preview`、Derived／Runtime targetにはmaterializeしない。

```text
EditorCanvasGizmoTransactionV1
  transaction_key
  tool_id
  target_set_hash
  base_project_revision
  base_value_refs[]
  coordinate_domain_ref
  transform_space
  pivot_mode
  snap_profile_ref?
  active_handle_id
  pointer_capture_generation
  preview_values[]
  registered_change_primitive_ref
```

- pointer downでtarget集合、base value、Project revision、tool／space／pivot／snap、capture generationをfreezeし、move中はlocal previewだけを更新する。pointer release／Enterで全targetを再validateして一ChangeSet、Escape、capture loss、focus loss、revision／lock／generation drift、device lossでpreviewを全破棄する。moveごとにProject write、Undo entry、Runtime writeを発行しない。
- multi-selectionは同じcoordinate dimension、owner、live-edit policyでownerが一括Transformを定義する場合だけ使う。mixed Scene owner、2D／3D混在、Source／Runtime混在、query selection、1024件超、代表Transform捏造はdisabled reasonにする。
- translate／rotate／scaleはXをsquare cap＋`X`、Yをdiamond cap＋`Y`、Zをcircle cap＋`Z`、UI anchorをanchor-rectangle＋`UI`で示し、色だけに依存しない。shaftは2 lu、visual handleはcompact／standard／comfortableで12／14／16 lu、hit rectは常に32×32 lu以上、中心からaxis endは56／64／72 luとする。数値feedbackはCode face＋tabular numericでdelta、unit、snapを表示する。
- snapはowner-issued profile ref、enabled state、translation unit、rotation angle、scale stepをcontext barとdrag feedbackへ表示し、grid pixelやcamera zoomから値を推測しない。snap結果がnon-finite、capacity超過、constraint違反ならclampして成功させずpreviewをinvalid表示する。
- UI Designerの`absolute` nodeはanchor／offset／explicit size field、SceneはTransform fieldへ束縛する。stack／flex／grid child、read-only／locked、restart-required targetはhandleをdisabled表示し、Inspectorのexact field、registered reorder／layout command、Source Intent Viewをkeyboard／UIA代替として`ControllerFor`で結ぶ。UIA TransformはProject objectへ使用しない。

#### 15.6.19 Required canvas conformance scenarios

| scenario ID | 必須内容 |
|---|---|
| `scenario.widget.canvas-surface.owner-binding@1` | 五canvas kindがowner-issued projection／generation／authorityへexact解決し、Scene semanticsのGame／UI Designer流用とgeneric property bag 0件 |
| `scenario.widget.canvas-surface.navigation-separation@1` | pan／orbit／zoom／frame／axis snap後もProject revision、selection、Source Camera不変。editor cameraをSourceへwrite 0件 |
| `scenario.widget.canvas-surface.pick-generation@1` | gizmo／overlay／Source priority、重なりselector、stale／omitted／unknown拒否、Stable target収束、pixel／coordinate identity 0件 |
| `scenario.widget.selection-overlay.state-layer@1` | active／multi-selection、Canvas focus、warning／error、proposal、Runtime、stale、occluded／offscreenをLight／Dark／四High Contrastで別axisとして識別 |
| `scenario.widget.gizmo.transaction@1` | begin／preview／commit／Escape、capture／focus／revision／lock／device-loss cancel、一ChangeSet／Undo、partial Project write 0件 |
| `scenario.widget.gizmo.multi-target-read-only@1` | compatible multi-target、owner／dimension／authority混在拒否、locked／read-only／restart-required reason、代表値生成0件 |
| `scenario.widget.canvas-surface.ui-layout-authority@1` | absolute anchor／offset edit、stack／flex／grid free drag拒否、exact layout generation、safe area／locale／scale、Editor独自layout 0件 |
| `scenario.widget.canvas-surface.game-input-routing@1` | explicit capture、visible status、Escape／focus／generation／crash release、同時capture一件、Editor shortcut／UIA／AIからGame input privilege 0件 |
| `scenario.widget.canvas-surface.overlay-contract@1` | passive／pickable／editing_tool、priority、visibility／pickability／Source visibility分離、同時editing tool一件、hidden authority change 0件 |
| `scenario.widget.canvas-surface.uia-keyboard-alternative@1` | Pane、Group、standard child Controls、ControllerFor、Outliner／Inspector／Command代替を検証し、pixel由来childとUIA TransformによるProject write 0件 |
| `scenario.widget.canvas-surface.ai-semantic-separation@1` | AIがSelection／World Context／Scene Slice／UiDocument typed projectionを使い、render texture、UIA、pick buffer、screen coordinate、display text parse 0件 |
| `scenario.widget.canvas-surface.density-content@1` | 三density、Light／Dark／四High Contrast、200% Font、`editor_ui_scale=1.00／2.00`、640×360、長い日本語Target、XYZ／UI axis、safe areaで意味／hit target不変 |
| `scenario.widget.canvas-surface.cross-panel-projection@1` | Outliner／Scene／InspectorのAuthoring targetとProblems／AIのexact relationが同じStable target／revision／Diagnostic／Actionへ収束し、Evidence selectionとGame／UI Designerのowner境界を維持 |

`fixture.canvas.reference@1`は、Scene 3DにSource Entityの`selected + focused + warning`、world-space translate、snap、Source Camera frustum、Runtime overlay、stale AI proposal、Scene 2Dにmulti-selectionとpixel／PPU、GameにRuntime Session／Camera／frameと明示Game input capture、UI Designerにabsolute nodeのanchor／safe-area edit、stack childのfree drag拒否、pseudo locale／200% UI scale、Map PreviewにDerived read-onlyを固定表示する。Light／Dark、四High Contrast、三density、100／200% DPI、200% Font、`editor_ui_scale=1.00／2.00`、keyboard、Narrator／NVDA、UIA、AI、device lossで同じowner ref、Stable target、selection、Project revision、Action可否へ収束するまで四Canvas Patternをclosedと扱わず、Panel固有Canvasへfallbackしない。

#### 15.6.20 `widget.graph.graph-surface@1`

Graph型は単一のgeneric node bagでもPanelごとの別実装でもなく、共通surfaceとowner-issued typed projectionを合成する。`graph-surface@1`はGameplay Definition Graph、Animation Graph、Topology Graph、Render Graph、Causalityのfocus、view、selection、LOD、virtual query、layout binding、interaction generationを所有し、Gameplay／Animation／World／Renderer／Debuggingのnode、port、edge、validation、Runtime正本を所有しない。

検討した方式は、A=`UiCanvasSurface`へGraph／Timelineを同じgeneric primitiveとして押し込む、B=各Panelが独自node／track Editorを持つ、C=共通selection／view／transaction基盤の上へGraphとTimelineの別semantic surfaceを置く、の三つである。Aはedgeとtime item、topologyとplayback cursorの意味を曖昧にし、BはCommand／Undo／AI／UIA／state表現を重複させるため不採用とし、Cを正式採用する。

```text
EditorGraphSurfaceStateV1
  schema_version: 1
  graph_kind: gameplay_definition | animation | world_topology |
              render_derived | causality_derived
  owner_projection_ref
  project_revision
  graph_revision
  projection_generation
  authority: source | staging | derived_read_only | runtime_read_only
  target_ref
  context_path_refs[0..32]
  query_ref
  query_revision
  total_node_count
  total_edge_count
  selection_context_ref
  layout_binding: deterministic_read_only | personal_workspace | owner_shared
  layout_generation
  view_descriptor_ref
  runtime_projection_ref?
```

- `owner_projection_ref`、node／port／edge Stable ref、`graph_revision`、Runtime overlayはDomain ownerのexact projectionから取得する。Project display name、node title、配列index、layout座標、edgeのBezier control pointをidentityにしない。
- pan、zoom、focus history、open breadcrumb、minimap visibilityは`EditorUserState`でありProject Undo、Cook、Runtimeへ入れない。zoomは`0.10～4.00`、既定`1.00`に閉じ、`editor_ui_scale`またはDPIと乗算した値をProjectへ保存しない。
- `layout_binding=personal_workspace`のnode位置は`EditorUserState`、`owner_shared`はownerが明示登録したpresentation-only layout recordとCommand、`deterministic_read_only`はStable IDとtopologyから決まるread-only layoutを使う。shared layout変更は一つのlayout-only ChangeSetとし、Gameplay／Animation／World semantic hash、Cook artifact、Runtime挙動を変えない。Owner layout contractがないGraphを`owner_shared`へ昇格しない。
- semantic hashはowner topology、typed field、validation、authority、revisionから作り、node位置、pan、zoom、viewport、minimap、route control pointを除外する。layout hashは別に保持し、layoutだけの変更でAI proposalをstaleにしない。
- visible viewportと各方向一viewport分のoverscanだけをrealizeし、nodeとedgeの省略数、total count、query revision、continuationを保持する。50,000 nodeをUI tree、UIA tree、AI Contextへ全展開しない。render producerはviewportと交差するedgeだけを描き、budget超過時は一部edgeを別接続へ束ねずtyped capacity statusとomitted countを示す。
- LODはview zoom `>=0.75`で全anatomy、`0.45～<0.75`でheader、status、port shape／count、`0.20～<0.45`でnode kind、title、status、接続方向、`<0.20`でnode outline、kind glyph、status countだけを描く。selected／focused targetは`Focus`または`詳細を表示`で`>=0.75`へframeできる。LODはSemantic Element、Action可否、selection、Diagnosticを変更しない。
- plain clickは一件replace、Ctrl+clickはtoggle、Shift+clickはadd、empty clickはmodifierなしのときだけclearする。marqueeは完全／交差selection modeをcontext barへ明示し、1024件上限を越える候補はimplicit先着選択せずquery summaryとrefine Actionを示す。
- Human pointerのnode／edge hit snapshotは同じprojection／layout／view generationへ束縛する。stale、unknown、omitted hitではselectionまたはProjectを変更しない。AIはhit snapshot、minimap、layout座標、screen coordinateを受け取らず、bounded graph queryとregistered `EditorSemanticActionV1`を使う。

Owner variantは次で閉じる。

| `graph_kind` | Projectionと編集境界 |
|---|---|
| `gameplay_definition` | `GameplayDefinition`のcurrent schemaへexact束縛する。Rule／ECA、FSM等の未登録kindをgeneric nodeとして作らず、ownerが登録したChangeSetだけをSource／Stagingへ適用する |
| `animation` | `AnimationGraphDefinition`のstate、transition、blend、layer、parameter等をAnimation ownerから投影する。現行のAnimation authoring Operation集合は`[]`、Capabilityは`not_activated`であるため、Production UIはinspect／disabled reasonだけを示し、fixture名やplanned action名から編集Actionを生成しない |
| `world_topology` | Level、Portal、entry／exit AnchorをWorld ownerのStable refとprimitiveへ束縛し、片側edge、暗黙Scene owner変更、Level membership変更を作らない |
| `render_derived` | `RenderGraphDefinition`／canonical execution planのowner-issued Derived projectionをread-only表示する。pass／resource／queue／barrierを表示位置から編集せず、Project／AIからnative resourceまたは未登録passを追加しない |
| `causality_derived` | `CausalityGraphV1`のbounded query、frontier、gap、completenessをread-only表示する。欠落frontierをedgeなしまたはroot cause確定として扱わない |

#### 15.6.21 `widget.graph.graph-node@1`

`graph-node@1`は一つのowner Stable nodeを表すcompositeである。anatomyはvalidation rail、header（kind glyph、primary name、kind short label、authority／Runtime status）、body（typed field summary）、input port lane、output port lane、footer（Diagnostic count、lock／stale／proposal）に固定する。利用しないslotはvariant schemaから除外し、空のgeneric property rowを作らない。

- node幅は176～384 lu、既定224 luとし、headerとbody row高はcompact／standard／comfortableで24／28／32 luとする。primary nameは一行ellipsis、kindは一行、body valueは最大二行で、full localized name、identifier、type refをSemanticとdetailsへ保持する。Japanese labelはUI face、identifier／type refはCode face、数値はtabular figuresを使う。
- headerは`surface.raised`、bodyは`surface.panel`、borderは`border.subtle`を使う。selectedは`selection.active` fill＋outline、focusは別の2 lu ring、validationは左rail、AI proposalは外周dash＋`AI提案`、Runtime activeは上端2 lu secondary line＋play glyph＋`Runtime`、staleはdotted border＋revisionとする。一色またはheader fillだけへ状態を畳み込まない。
- node kindはkind glyph＋short labelで常時示す。Entry／Exit、State、Rule、Blend、Pass等を色だけで区別せず、unknown kindは描画せずtyped unsupported projectionをProblemsへ出す。
- bodyの折畳みはpresentation stateでありnode fieldまたはRuntime stateを変更しない。port groupが片側8件を越える場合はkind単位のcount rowへ畳み、focus／queryで展開できるようにする。隠れportとedgeの件数、Diagnosticを省略しない。
- node moveはtopology editでない。`personal_workspace`はWorkspace stateだけ、`owner_shared`は登録済みlayout Commandだけを一transactionで変更し、`deterministic_read_only`ではdisabled reasonを示す。node移動とedge作成、field変更を一つの曖昧なdrag transactionへ結合しない。

#### 15.6.22 `widget.graph.graph-port@1`

`graph-port@1`はowner-issued Stable port ref、`input | output`、exact type／contract ref、cardinality、connection policyを表す。左右配置はpresentationであり、port directionまたはidentityを決定しない。bidirectional意味を一portへ追加せず、必要ならownerがinput／outputの別Stable portを発行する。

| presentation class | visual shape | 必須表示 |
|---|---|---|
| `flow` | chevron | `Flow`／遷移等のowner short label |
| `value` | circle | exact value type short label |
| `reference` | square | target kind short label |
| `pose` | capsule | `Pose`とdimension／skeleton compatibility status |
| `event` | hexagon | exact Event contract short label |

presentation classは登録済みtype-to-presentation mappingからだけ解決し、classだけで接続互換性を判定しない。shapeはcompact／standard／comfortableで10／12／14 lu、hit targetは常に32×32 lu以上とする。input／output name、exact type、required／optional、current connection countをfocus、hover、Semanticへ示す。接続preview中のcompatible、incompatible、unknownはそれぞれcheck、ban＋reason、question＋`未判定`で示し、色だけのglowへ縮退しない。

```text
EditorGraphConnectionTransactionV1
  graph_ref
  base_project_revision
  graph_revision
  projection_generation
  source_port_ref
  candidate_target_port_ref?
  compatibility_query_ref
  compatibility_result: compatible | incompatible | unknown
  proposed_conversion_ref?
  pointer_or_keyboard_capture_generation
  phase: preview | commit_requested | committed | cancelled
```

- pointer downまたはkeyboardの`接続を開始`でbase revision、source port、generationをfreezeし、hover／navigation中はlocal previewだけを更新する。compatible targetへのrelease／Enterでowner登録済みCommandを一回だけ発行し、一ChangeSet／一Undo unitにする。
- Escape、capture／focus loss、Project／graph revision drift、lock／permission変更、port消失、owner query generation変更、device lossでcancelする。cancel時にedge、node、layout、Runtimeへpartial writeしない。
- 空白へのdropはsource portを保持したbounded compatible-node catalogを開く。node作成と接続をownerがatomic ChangeSetとして登録している場合だけ一括Commitし、それ以外は別操作に分ける。AIは空白座標を使わず、graph ref、source port ref、node kind ref、logical placement intent（auto／near Stable node／inside exact context）を受けるregistered Actionだけを使う。
- implicit cast、型名一致、表示色一致、最寄りport、cycle修復、既存edge置換を行わない。変換nodeがowner登録済みでcompatible resultにexact refとして含まれる場合も、User／AIが明示選択したときだけ使用する。

#### 15.6.23 `widget.graph.graph-edge@1`

`graph-edge@1`はowner Stable edge refとexact source／destination port refを持つ。通常線は2 lu、selectedは3 lu、pointer hit corridorは12 luとし、destination側に方向arrow、中央または最も長い可視segmentにedge kind／condition short labelを置く。edge crossingへjunction dotを描かず、実nodeまたはportのない交差を接続として見せない。

- edge意味は両端port shape、方向arrow、short labelで示し、色だけでflow／value／event／dependencyを区別しない。selectedは外側outline、focusは2 lu focus halo、warning／errorは最初のfailing endpoint marker＋label、AI proposalはviolet dashed halo＋`AI提案`、Runtime activeは平行するcyan secondary line＋play glyph＋`Runtime`、staleはclock＋revisionを使う。
- Runtime flowをmoving particle、無限pulse、edge速度で表現しない。Runtime state changeは0 msで切り替え、reduced motionと通常設定で同じactive edge、sequence、Diagnosticを読む。
- route control pointはlayoutだけで、semantic hash、AI Context、edge identity、execution orderへ入れない。edge bundlingまたはLOD省略は複数edgeを一edgeに見せず、endpointごとのconnection countとomitted countを示す。
- reconnectは§15.6.22と同じtransactionを使い、old edgeをCommit前に削除しない。Delete、Reconnect、Reveal source／targetはedge Stable refへ束縛されたregistered Commandだけを公開する。

#### 15.6.24 `widget.timeline.timeline-surface@1`

`timeline-surface@1`はAnimation TimelineとDebug Timelineのtrack header、time viewport、virtual query、selection、preview cursor、Runtime／recorded cursorを同期するsurfaceである。Graph surfaceとview／selection／transaction基盤を共有するが、node／edgeまたはgraph zoom semanticsを流用しない。

```text
EditorTimelineSurfaceStateV1
  schema_version: 1
  timeline_kind: animation_authoring | debug_recorded
  owner_projection_ref
  project_revision
  projection_generation
  authority: source | staging | derived_read_only | runtime_read_only
  target_ref
  track_query_ref
  track_query_revision
  total_track_count
  total_item_count
  coordinate:
    animation_timebase:
      timebase_ref
      rate: ReducedPositiveRationalV1
    | debug_record_sequence:
      debug_session_ref
  visible_interval
  display_unit: source_tick | frame | second | record_sequence
  selection_context_ref
  snap_profile_ref?
  preview_cursor_ref?
  runtime_or_recorded_cursor_refs[0..8]
  view_descriptor_ref
```

- `animation_authoring`のidentityはSource Stable track／key／span refとexact timebase tick、`debug_recorded`は`DebugSessionDescriptorV1`とrecord sequence／`DebugTimePointV1`である。screen x、tick label文字列、float second、render frame観測値をidentityへ使わない。
- Animationのframe／second表示はowner timebaseとexact rational変換できる場合だけ選べる。sub-frameは丸めず`frame + rational remainder`を表示し、display unit変更でkey time、playhead、selection、AI targetを変更しない。Debugのwall／GPU／Audio時刻はfresh `ClockCorrelationV1`がある場合だけ補助labelにし、record sequenceの正規順序を置換しない。
- pan、horizontal zoom、track scroll、header幅、collapsed track、follow cursorは`EditorUserState`である。track header幅はcompact／standard／comfortableで既定208／240／272 lu、160～480 luでresize可能とし、Projectまたはtime coordinateへ保存しない。
- trackは可視rangeと上下二viewport分、time itemはvisible intervalの前後一interval分だけquery／realizeする。100,000 key／span／event fixtureをUI tree、UIA、AI Contextへ全展開せず、total、returned、omitted、continuation、query revisionを保持する。
- Animation previewはownerのisolated preview instanceだけを評価し、Runtime event、Gameplay event、root-motion proposal、live instance clock／cursor／stateを変更しない。Debug Timelineは全variantでread-onlyとし、seekはEvidence selection／Replay queryを変更できてもProject／recorded Storeを書き換えない。
- Animation ownerのcurrent authoring Capabilityが`not_activated`の間、key／track／span mutating Action、auto-key、recordingをすべてdisabled reason付きで隠さず表示し、planned action vocabularyからdispatch IDを生成しない。Pattern conformanceの編集transactionはtest-only owner fixtureで検証し、Production Capability activationの証拠にしない。
- auto-keyはCapability、Owner Manifest、MCD、Policy、Validator、Receiptがatomic activationされた場合だけ表示する。既定`off`、Humanによる明示`armed`、実際にChangeSetを生成中の`recording`を別stateにし、armed中はrecord circle＋`Auto-key armed`、exact Target／track／timeを固定表示する。compatible track不在時に暗黙作成せず選択を要求する。AIはauto-keyをarmせず、exact key ChangeSetをproposalとして提出する。

#### 15.6.25 `widget.timeline.time-ruler@1`

`time-ruler@1`はcurrent coordinateとdisplay unitを投影するread-only scaleであり、time identityまたはSource durationを所有しない。高さはcompact／standard／comfortableで24／28／32 lu、major label間隔は72 lu以上、minor tickは12 lu以上を保つ。stepはpositive exact unitの`1／2／5 × 10^n`から条件を満たす最小値を選び、floating-point加算でtickを累積しない。

- tick labelはmono face＋tabular figures、unit／contextはUI faceを使う。timebase、FPS、record sequence、display rounding policyをcontext barへ表示し、`00:00`等の表示文字列をparseしてseekしない。
- ruler clickはpreview／recorded cursorのseek requestだけを作り、keyframeまたはProject fieldを作らない。selection range、playback range、recorded query rangeは異なるhead shape＋short labelを持ち、一つの半透明bandへ統合しない。
- ruler zoom、range handle drag、Home／Frame selectionはview stateだけを変更する。Source duration／playback rangeを変更するCommandはownerが別に登録した場合だけ公開する。

#### 15.6.26 `widget.timeline.timeline-track@1`

`timeline-track@1`は一つのowner Stable trackまたはtrack groupを表し、headerとtime laneを同じrowへ束縛する。row高はcompact／standard／comfortableで24／28／32 luとし、hierarchy depth、track kind glyph、primary name、target／property short label、lock／authority／validation、item countをheaderに置く。長い日本語名、identifier、pathはellipsisしてfull semantic textを保持する。

- groupはTreeItem相当のexpand／collapseを持つが、collapseはitem、Diagnostic、selectionを削除しない。hidden selected item countをheaderへ示す。
- row ordering、vertical coordinate、同名trackをidentityにしない。reorder、reparent、track creation／deletionはowner登録済みCommandのpreview／validation／一ChangeSetだけを使い、pointer dragのdrop位置をSource indexへ直接書かない。keyboard／context menuに同じ代替Actionを持つ。
- selected row、focused item、locked／read-only、Runtime binding、warning／error、AI proposal、staleを§15.3の別layerで表示する。mute、solo、visibility、enable等はowner schemaに存在するControlだけをslotへ出し、汎用eye／speaker toggleからSource意味を生成しない。

#### 15.6.27 `widget.timeline.timeline-span@1`

`timeline-span@1`は開始／終了を持つowner itemを`animation_clip | section | debug_span | evidence_gap` variantとして表示する。`animation_clip`／`section`はSource Stable ref、`debug_span`／`evidence_gap`はDebug record／gap refへ束縛し、variant間で編集Actionを共有しない。

- visual bodyはtrack rowから上下3 luを除いた高さ、最小可視幅2 lu、pointer hit corridorは32 lu以上とする。primary label、duration／event count、source／recorded statusを表示し、短すぎるspanは端marker＋details calloutへ縮退してidentityを省略しない。
- editable spanのstart／end handleはvisual 6 lu、hit target 32 lu以上とする。pointer downまたはtyped time fieldのeditでbase revision、全selected item、exact begin／end、snap profile、generationをfreezeし、previewはlocalだけ、release／Enterで一ChangeSetとする。Escape、focus／capture／revision／lock／generation変更で全件cancelし、partial trim／retimeを残さない。
- overlap、minimum duration、loop、blend、event crossingはowner validatorのresultを表示し、UIがsilent trim、merge、ripple、stretch、frame roundingを行わない。複数item retimeで一件でもinvalidなら全体をinvalid previewにする。
- `evidence_gap`は斜線band＋broken-link icon＋`欠損`＋lost range／reasonを表示し、正常な空白、stale、warning severityへ変換しない。gap上を補間、snap、原因推定、key生成しない。

#### 15.6.28 `widget.timeline.keyframe@1`

`keyframe@1`はowner Stable key ref、exact time、typed value／event ref、interpolation policyを表す。visual shapeは`step=square`、`linear=diamond`、`curve=circle`、`event=flag`に固定し、compact／standard／comfortableで10／12／14 lu、hit targetは32×32 lu以上とする。shapeだけでexact interpolationまたはEvent typeを決定せず、focus／details／Semanticへshort labelとexact refを出す。

- selectedは`selection.active` fill＋outline、focusは2 lu ring、warning／errorはcorner marker＋label、AI proposalはviolet dashed halo、Runtime fired／sampledはcyan play glyph＋`Runtime`、staleはclock＋revisionを使う。同時刻の複数keyはoffset stack＋件数で表示し、一keyへmergeしない。
- key moveは全selected keyのexact base timeと一つのtyped deltaをfreezeする。snap結果はowner-issued profileとexact timebaseから計算し、screen pixel、前回label、display FPSから逆算しない。release／Enterで一ChangeSet、invalid／out-of-range／collision policy unknownなら全件Commitしない。
- value、tangent、interpolation、Event payloadはInspectorの`property-row@1`または登録済みtyped editorで変更する。Canvas上のkey dragからvalueまたはEvent payloadを暗黙変更しない。

#### 15.6.29 `widget.timeline.playhead@1`

`playhead@1`は`editor_preview | runtime_current | recorded_selected` variantに閉じる。lineは2 lu、headはcompact／standard／comfortableで12／14／16 lu、editable preview hit targetは32 lu以上とする。

| variant | 見た目 | 操作 |
|---|---|---|
| `editor_preview` | downward triangle＋solid focus色line＋`Preview` | pointer／keyboard／typed fieldでseek可能。Project Sourceは変更しない |
| `runtime_current` | play glyph＋double runtime色line＋`Runtime` | read-only。exact Runtime snapshot refへ追従し、preview cursorから変更しない |
| `recorded_selected` | pin head＋solid secondary line＋`Recorded`／sequence | Debug query内のEvidence selectionだけを変更し、Store／Projectを変更しない |

- scrub requestはtimeline ref、exact destination、projection generation、preview request generationを持つ。UIは最新一件へcoalesceできるが、別generationのasync resultを表示しない。pending中はlast-valid previewへ`Preview pending`を併記し、destinationの評価成功を偽装しない。
- Animation scrubは§15.6.24どおりisolated previewだけを評価する。scrubでRuntime event、root motion、auto-key、Project changeを発火せず、preview結果をRuntime cursorまたはrecorded Evidenceへ書き戻さない。
- playback中のcursor位置はowner結果を0 msで直接反映し、easing、spring、trail、edge pulseを付けない。reduced motionではdecorative auto-panを停止し、cursor、exact time、play／pause状態は保持する。

#### 15.6.30 Required graph／timeline conformance scenarios

| scenario ID | 必須内容 |
|---|---|
| `scenario.widget.graph-surface.owner-binding@1` | 五graph kindがexact owner projection／revision／generation／authorityへ解決し、generic property bag、label／layout由来identity、Panel固有graph contract 0件 |
| `scenario.widget.graph-surface.layout-separation@1` | personal／shared／deterministic layout、pan／zoom／breadcrumb後もsemantic graph hash、Cook、Runtime不変。shared layoutだけ一layout ChangeSet、AI proposal stale化0件 |
| `scenario.widget.graph-node.state-layer@1` | node／edgeのselected、focus、warning／error、proposal、Runtime、stale、lockedをLight／Dark／四High Contrastで別axisとして識別 |
| `scenario.widget.graph-port.connection-transaction@1` | pointer／keyboard begin、compatible／incompatible／unknown、blank catalog、commit／Escape／generation cancel、一ChangeSet／Undo、partial edge／node write 0件 |
| `scenario.widget.graph-edge.no-implicit-conversion@1` | exact type、cardinality、cycle、conversion node明示、reconnect atomicityを検証し、色／名前／nearest portによる接続、silent replace／repair 0件 |
| `scenario.widget.graph-surface.selection-navigation@1` | plain／Ctrl／Shift／marquee、node／edge selection、1024上限、offscreen selected、frame／search、layout座標由来target drift 0件 |
| `scenario.widget.graph-surface.runtime-source-separation@1` | Source／Runtime revision差、active node／edge、Diagnostic、proposalを重畳し、Runtime overlayからSource topology write 0件 |
| `scenario.widget.graph-surface.activation-boundary@1` | Animation `not_activated`、Derived Render／Causality read-only、unknown kindをdisabled reason／unsupported projectionで処理し、planned action名dispatch 0件 |
| `scenario.widget.graph-surface.virtual-query@1` | 50,000 node、viewport＋overscan、四LOD、total／omitted／continuation、stale query、UIA／AI bounded queryを検証し、全materialize／false edge bundling 0件 |
| `scenario.widget.graph-surface.uia-keyboard-alternative@1` | Pane、realized Group、port Invoke、companion List／DataGrid、ItemContainer／VirtualizedItem、node／port／edge search／connect／deleteを検証し、pixel edge elementとUIA Transform write 0件 |
| `scenario.widget.graph-surface.ai-semantic-separation@1` | AIがStable node／port／edge、owner query、revision、registered Actionを使い、screen座標、layout、minimap、Bezier、UIA、display text parse 0件 |
| `scenario.widget.timeline-surface.time-domain@1` | animation exact timebase／rational frame・second、debug sequence／clock correlation、unit switch、sub-frame、large valueを検証し、float累積／label parse／wall time order 0件 |
| `scenario.widget.playhead.scrub-isolation@1` | pointer／keyboard／Value seek、async coalesce／generation reject、pending／last-valid、Runtime event／root motion／live cursor／Project write 0件 |
| `scenario.widget.keyframe.transaction@1` | single／multi key、exact delta／snap、collision／range invalid、commit／Escape／revision cancel、一ChangeSet／Undo、partial key write 0件 |
| `scenario.widget.timeline-span.transaction-gap@1` | trim／retime、owner overlap policy、Debug span、evidence gap、lost range／reasonを検証し、silent merge／trim／interpolate／gap正常化 0件 |
| `scenario.widget.timeline-surface.state-cursor-layer@1` | preview／Runtime／recorded cursor、selection、validation、proposal、stale、gapを形状＋labelで同時識別し、一playheadまたは一色への統合0件 |
| `scenario.widget.timeline-surface.activation-autokey@1` | current Animation disabled reason、test owner transaction、将来のoff／armed／recording、exact Target／track／time、AI arm 0件、暗黙track作成0件 |
| `scenario.widget.timeline-surface.virtual-query@1` | 100,000 key／span／event、track／time両軸virtualization、same-time stack、total／omitted／continuation、stale query、全materialize 0件 |
| `scenario.widget.timeline-surface.uia-keyboard-alternative@1` | Pane、track Tree／List、companion DataGrid、exact time Value、conditional RangeValue、key／span select／move／details、UIA Transform write 0件 |
| `scenario.widget.graph-timeline.density-content@1` | 三density、Light／Dark／四High Contrast、200% Font、`editor_ui_scale=1.00／2.00`、640×360、長い日本語、64文字identifier、shape／hit target／time精度不変 |
| `scenario.widget.graph-timeline.cross-panel-projection@1` | Graph／Timeline／Inspector／Scene／Problems／AIが同じStable ref、Project revision、Diagnostic、Runtime／recorded context、Action可否へ収束 |

`fixture.graph-timeline.reference@1`は、Gameplay FSM GraphにEntry／State／transition、selected＋focused state、type-incompatible port、Runtime active edge、warning、stale AI proposal、Animation GraphにSource／Runtime revision差と`not_activated`編集Action、Topology GraphにPortal validation、Render GraphにDerived read-only pass／resource relation、Causalityにunexpanded frontier／gap、Animation TimelineにSource timebase、同時刻key stack、preview／Runtime cursor、invalid multi-key retime、disabled auto-key、Debug Timelineにrecorded cursor、current revision差、span、gap／redactionを固定表示する。Light／Dark、四High Contrast、三density、100／200% DPI、200% Font、`editor_ui_scale=1.00／2.00`、keyboard、Narrator／NVDA、UIA、AI、50,000 node、100,000 time itemで同じowner ref、Stable target、exact time、selection、revision、Action可否へ収束するまで十Graph／Timeline Patternをclosedと扱わず、Panel固有surfaceへfallbackしない。

#### 15.6.31 Source／Text／Diff Pattern Contract

`widget.diagnostics-source.source-line@1`と`widget.diagnostics-source.diff-hunk@1`は、`widget.input.multi-line-text-editor@1`のread-only／editable text surfaceを構成する既存composite Patternである。Source surface、History／Diff、Reviewは[Editor Panel／Reference Catalog §6.4](editor-panel-reference-catalog.md#64-sourcetextdiff)の同じowner projectionを使い、Source text、history entry、generated output、external fileを一つのgeneric editable bufferへ変換しない。

- 両Patternはowner-issued document／source subject ref、displayed revision、current revision、該当時のcomparison base revision、range／hunk projection refを同じinstanceに束縛する。path、display line／column、row index、screen coordinate、search orderはpresentationであり、Stable target、revision、write scopeを代用しない。
- `source-line@1`のanatomyはpresentation gutter、code text、generated／read-only marker、diagnostic marker、selection、source contextである。gutterのline numberはcopy／focus時の補助表示だけであり、Source identityやAction targetを持たない。日本語の説明はUI face、identifier／path／literal／Diffはcode face、数値は必要に応じてtabular figureを使う。
- `diff-hunk@1`のanatomyはbefore range、after range、deleted／added text、base／current revision、proposal／validation link、context summaryである。three-way conflictは`base`／`ours`／`theirs`とunresolved reasonを別blockとして表示し、warning、空白、normal hunk、片側の自動採用へ偽装しない。conflictはowner-issued projection contentであり、Pattern固有のvisual state axisを追加しない。
- C1のsurfaceは`editability=read_only`である。copy、focus、semantic read、検索、registered revealは維持するが、Insert／Delete／Paste／Save／Apply／Commit／Promotion、AI Semantic mutation、UIA mutationは0件にする。`generated`、`derived`、`runtime`、`policy_locked`、`capability_not_activated`のreasonと`freshness=stale／superseded`、unresolved conflictを別slotで示す。`stale`では既存規則どおりCommitをdisabledにし、登録済みrebase／discard／refresh／reveal以外を公開しない。
- TSFはeditable Controlのlocal draftにだけ接続し、read-only Source／DiffへIME compositionを開始しない。read-only UIAはText／TextRange、selection、copyを維持し、TextEdit／Value mutationを公開しない。AIがbounded Source／Document／Diff projectionと登録済みtyped Actionを使用できるのは対応familyのActivation後だけであり、C1のSource AI Actionはexact `[]`に保つ。いずれの場合もAIはUIA、text buffer、pixel、line number、display textからAction権限を得ない。
- `editable`への遷移は、対応Source operation family、Policy、Validator、Receipt、sandboxが同一Activationでactiveになった後だけ許す。外部変更はProject Stateのthree-way comparisonとtyped `ExternalTool` ChangeSetへ渡し、Patternはauto-merge、implicit save、Source Promotion、binary loadを行わない。

#### 15.6.32 Task／Proposal／Review Pattern Contract

`widget.ai-task.proposal-card@1`、`approval-bar@1`、`task-stage@1`、`receipt-row@1`は[Editor Workspace UX §8.1.1](../03-authoring/editor-workspace-ux.md#811-taskproposalreview-reference-contract)のowner-issued recordを投影する。四Patternは既存のinteraction／availability／editability／authority／validation／freshness／approval／task axisだけを使用し、Proposal、Approval、Task、Receiptを一つのstatus badge、selection、Diagnostic、Runtimeへ畳み込まない。各mutating ActionはCommand Registryがtarget、expected revision、Risk、Authorization、Approvalに基づき投影した場合だけ公開する。

| Pattern | 必須projection | 禁止 |
|---|---|---|
| `proposal-card@1` | proposal ref、base／current revision、scope、before／after、Validation、Approval、stale、field／Document／primitive単位Action | current valueの置換、`Accept all`、Pattern由来のwrite権限 |
| `approval-bar@1` | Risk、exact subject hash、scope、approver role／independence、issued／expiry、revocation／freshness、pending／approved／rejected／expired、disabled reason | AI、Validator、Task結果、visual reviewをApprovalへ変換 |
| `task-stage@1` | Governance Taskのexact state、stage、実測progressまたはindeterminate stage、Question／Code Owner／Approval block、cancel policy | `OperationTaskV1`とのstate混同、擬似percent、modal block |
| `receipt-row@1` | purpose、receipt ref／hash、issuer、subject、result、read-back、freshness | Receipt存在だけからsuccess／authorityを導出 |

`task-stage@1`の`task` axisは視覚上の集約だけであり、Governance Task stateをaliasしない。exact state、Task ref、block／terminal reasonは必ず同時表示する。

| Governance Task state | `visual_state.task` |
|---|---|
| `Draft` | `idle` |
| `ResolvingRequirements`、`Ready` | `queued` |
| `AwaitingAuthorization`、`AwaitingCodeOwner`、`AwaitingApproval`、`AwaitingUserInput` | `blocked` |
| `Running`、`Validating`、`Promoting` | `running` |
| `Completed` | `succeeded` |
| `Cancelled` | `cancelled` |
| `Expired`、`Failed`、`Rejected` | `failed` |

`viewed`、Workspace-local pending decision、signed decision、stale／expired Receiptを別content roleとして表示する。pointer、keyboard、UIA、AI、Pattern ID、表示文字列、screen coordinateはAction権限にならず、AIはUIAを使わず登録済みtyped Actionだけを使う。partial accept後の新ChangeSet／再validation、stale後のrebase／discard、signed ApprovalとReceipt read-backは、それぞれのOwner結果がPatternへ戻った後だけ表示を更新する。

#### 15.6.33 Required Source／Task conformance scenarios

| scenario ID | 必須内容 |
|---|---|
| `scenario.widget.source-text-diff.read-only-conflict@1` | committed／staging candidate／generated／historical projection、before／after、external three-way conflict、stale Proposal、read-only reason、Japanese／ASCII identifier／path／数値、selection／copy／search／revealを検証し、C1のInsert／Delete／Paste／Save／Apply／Commit／Promotion、auto-merge、line-number／UIA／AI由来mutationを0件にする |
| `scenario.widget.source-text-diff.uia-ime-scale@1` | Light／Dark／四High Contrast、三density、200% Font、`editor_ui_scale=2.00`、keyboard、Narrator／NVDA、UIA Text／TextRange、C1 Source AI Action exact `[]`を検証する。read-onlyではTSF composition／TextEdit／Value mutationを出さず、editable activation後だけ同じbounded draft contractとAI typed semantic readへ接続する |
| `scenario.widget.ai-task.lifecycle-and-staleness@1` | blocking Question、Suggestion、Validation pass／fail、Approval pending、field／Document／primitive単位partial accept、stale、Rebase／Discard、signed Approval／rejection、Receipt read-backをhuman、keyboard、UIA、AI typed Actionで検証し、正規Project state不変の否定系とAIのUIA利用0件を含める |

`fixture.source-text-diff.reference@1`と`fixture.ai-task-proposal-review@1`は、上記scenario、七oracle、fixed source／environment／coverage tupleが同じContract setでmaterializeされるまで実行Registry rowを持たない。Pattern definition、Screenshot、AI explanation、planned Capabilityをfixture passやCapability activationへ読み替えない。

#### 15.6.34 `widget.feedback.notification@1`

`notification@1`はWindow内だけで表示する一時的なfeedback surfaceであり、Project、Diagnostic、Task、Proposal、Approval、Receiptの正本またはその状態遷移を所有しない。外部Toast、notification area、background activation、OSのnotification historyへ投影せず、C1では同じWindow内のowner surfaceと併用する。actionなし`information`だけは明示的なoriginだけを持てるが、`registered_action_refs`を持つnotificationと`action_completed | action_aborted | warning | error | blocking`は必ず同一revision／Sessionに解決できるowner recordを持つ。notificationを表示したこと、UIA event、dismiss、overlay座標、activity IDをowner record、Action権限、AI Contextへ昇格させない。

```text
EditorTransientNotificationV1
  notification_id
  origin_ref
  owner_record_ref optional
  owner_record_content_hash optional
  notification_kind = information | action_completed | action_aborted
                    | warning | error | blocking
  visible_since_tick
  presentation_state = visible | dismissed | expired | suppressed
  auto_dismiss_policy = user_duration | manual_only
  message_duration_preference_snapshot_ref optional
  display_message_key
  bounded_display_arguments[0..16]
  redaction_result
  registered_action_refs[0..2]
  uia_notification_kind = other | action_completed | action_aborted
  uia_notification_processing = most_recent | current_then_most_recent
                              | important_most_recent
  nonlocalized_activity_id
  notification_content_hash
```

##### 表示経路、anatomy、motion

C1はglobalなfloating toast、Canvas／Graphを覆うoverlay、通知stackを持たない。MicrosoftのInfoBar guidanceが「状態変化の通知はlayout内に置き、contentを覆わせない」こと、severityはicon／colorだけで表さないことを示すため、Miraikanaiでは次の二経路を`notification_kind`、owner record、`registered_action_refs`、`auto_dismiss_policy`から一意に解決する。これはWinUI InfoBarを採用する決定ではなく、独自Widgetのpresentation規約である。

| 解決条件 | 表示経路／root | 表示数と配置 | 禁止 |
|---|---|---|---|
| `information`、`registered_action_refs=[]`、`user_duration`（ownerはoptional） | `status_bar_transient`／noninteractive `StatusBar` | WindowごとにStatus barのfeedback lane一件だけ。最新`visible_since_tick`、同tickなら`notification_id` byte orderで一件を表示する。summaryは一visual lineだけで、overflowはellipsisする。ownerがある場合もdetailはowner surfaceに残す | floating card、stack、action button、owner stateの作成 |
| `action_completed`、ownerあり、`registered_action_refs=[]`、`user_duration` | `status_bar_transient`／noninteractive `StatusBar` | 同上。completionの永続detailはowner surfaceに残し、status laneは短いfeedbackだけを表す | completionをTask／Receiptのterminal stateやApprovalへ読み替えること |
| 上記以外のvalid notification（actionを持つもの、`action_aborted`、`warning`、`error`、`blocking`、またはownerを持つ`manual_only`） | `owner_inline`／`Group` | owner surfaceのcontent flow内に、同一`owner_record_ref`ごと一件のinline stripを置く。更新は同一ownerの最新summaryへcoalesceし、別ownerの通知をglobal stackへ集約しない | Window edgeへの自由配置、Panel内容を覆うoverlay、ownerなしmanual-only表示 |

ownerなし`information`が`manual_only`になった場合は`presentation_state=suppressed`とし、status root、UIA notification event、synthetic ownerを生成しない。これは既存のownerなしmanual-only通知を0件にする規則を表示経路にも適用するものである。`status_bar_transient`の置換、`owner_inline`のcoalesce、dismissはいずれもpresentationだけを変え、owner record、Project、Task、Proposal、Approval、Receipt、Commandを変更しない。

`status_bar_transient`は、densityに対応するicon token、localized category label、redaction済みsummaryを順に置く。`information`は情報role、`action_completed`は完了roleを使い、色だけで区別しない。visual textは一行ellipsisしても、rootのaccessible nameとUIA notification eventには同じbounded redaction済みsummaryを保持する。

`owner_inline`は`severity rail -> semantic icon -> localized category label -> summary -> registered action strip -> Dismiss`の順で構成する。railはwarning／error／blockingだけ既存の2 lu validation rail、iconとshort labelは`information | completed | aborted | warning | error | blocking`のsemantic role、summaryはUI typography、owner ref／revision／numeric metadataはcode／tabular roleを使う。padding、gap、icon size、body lineは§15.2／§15.4のdensity tokenだけを使い、raw hex、fixed physical px、font icon、unknown iconを足さない。summaryは最大二visual line、action stripは最大二つの既登録Command Buttonである。幅が足りないときは`summary -> action strip -> Dismiss`の順に縦reflowし、label、reason、Actionをtooltip、icon、ellipsisだけへ退避しない。詳細、raw log、secret、unbounded progressはowner surfaceにのみ残す。

`information`は`text.secondary`、`action_completed`は`validation.success`、`action_aborted`は`text.secondary`、warning／error／blockingは既存の`validation.warning`／`validation.error` tokenを使う。ただし全kindでsemantic iconとlocalized category labelを必須にし、`blocking`は`要対応`相当のlabelを持つ。High Contrastでは§15.1のsystem-color replacementを適用し、rail、icon、text label、focusを同時に残す。dismissはlocalized `閉じる`相当のaccessible nameを持つpresentation-only child Buttonであり、registered actionではない。`owner_inline`のregistered actionとDismissだけがchild Buttonになれ、status-bar routeはnoninteractiveのままとする。

enter／exitは§15.5の83 msでopacityだけを変え、slide、stack shift、pulse、progress animationを使わない。same-origin coalesceはvisible surfaceをflashさせずcontentだけを最新のbounded summaryへ置換する。reduced motionでは初回presentとdismissを0 msのfinal stateにし、既にcancelしたeffectをfull motion復帰後に再生しない。

`information`は`other + most_recent`、`action_completed | action_aborted`は同名UIA kind + `current_then_most_recent`、`warning`は`other + current_then_most_recent`、`error | blocking`は`other + important_most_recent`へだけ解決する。`NotificationProcessing_All`と`ImportantAll`はannounce floodを起こすためC1で禁止する。Task progress、同じsource recordのrepeated update、pointer hover、timer tickは別notificationや別UIA eventを発行せず、同じ`nonlocalized_activity_id`で最新のbounded displayだけへcoalesceする。`nonlocalized_activity_id`はstableなcoalescing keyであり、localized text、path、source、credential、Project名、display hash、Command targetを含めない。

`status_bar_transient` rootはnoninteractive `StatusBar`、`owner_inline` rootは`Group`である。registered actionとpresentation-only Dismissだけがchild `Button`になれ、registered actionだけがCommand Registryから投影した既存Commandを実行する。root自身をfocus trap、global selection、Menu、Dialog、custom UIA patternにせず、`Escape`または明示`Dismiss`はpresentationだけを`dismissed`へし、owner recordのresolve／acknowledge／cancel／approve／retry、Project変更、AI authorizationを行わない。`user_duration`は§13.2.6のactionなし`information`または永続owner recordを持つ`action_completed`に限り、action、warning、error、blockingは常に`manual_only`とする。時間によるdismiss、reduced-motionの0 ms、owner recordの更新はUIA／Command／Receiptを追加発行しない。

notificationがvisibleになりredaction済みlocalized summaryが確定した後、実際のroot providerへ一度だけ`UiaRaiseNotificationEvent`を発行する。`displayString`はbounded localized summary、kind／processingは上表、activity IDは同じnonlocalized coalescing keyにする。UIA eventのHRESULT失敗はhidden retry／announce spam、Action許可、owner mutationを起こさず、rootと永続owner recordを保つ。Reference／Platform Adapter captureは`infrastructure_error`として比較を開始しない。本文全文、secret、redacted range、raw log、unbounded Task progressをannouncementへ入れず、詳細はowner surfaceの標準Semantic／UIA treeで読む。

| scenario ID | 必須内容 |
|---|---|
| `scenario.widget.notification.duration-uia@1` | 同一running Windowの`d1 -> d2 -> d1`、SPI re-read、visible deadline再計算、manual-only不変、reduced-motionでの0 ms presentation、kind／processing／activity ID、summary redaction、同一sourceのcoalescing、`All`／`ImportantAll`／per-progress announce 0件を検証する |
| `scenario.widget.notification.persistent-authority@1` | information、completed、aborted、warning、error、blocking、registered button、Dismiss、owner recordをhuman／keyboard／Narrator／NVDA／UIA／AI typed Actionで分離し、OS Toast／background activation、UIA／activity ID／coordinate由来Action、dismissによるowner mutation、ownerなしのmanual-only通知を0件にする |
| `scenario.widget.notification.presentation-routing@1` | actionなしinformation／completedのStatus bar一件、owner-inlineのaction／aborted／warning／error／blocking、same-owner coalesce、global stack 0件、one-line／two-line／vertical reflow、Japanese／ASCII、三density、200% Font、Light／Dark／四Contrast Theme、full／reduced motion、noninteractive StatusBarとGroup／Button UIA境界を検証する |

### 15.7 Cross-Panel State／Selection Contract

本節はTree／List、Property／Form、Canvas、Graph／Timeline、Diagnosticsを同時に開いたときの「今どれを見ているか」を所有する。Project Source、Runtime、Diagnostic、Proposalの正本をUIへ移さず、Panel間で同じ対象を安全に参照するためのsession-level contractである。外部Engineで一般的なOutliner／Viewport selection同期とInspector追従は採用するが、すべてを一つのglobal selection listへ入れる方式と、Panel同士が`selection_changed` callbackを直接送り合う方式は採用しない。

採用方式は、owner-typedな複数channelをimmutable snapshotへ束ね、一つの`EditorAttentionReducer`だけがintentから次snapshotを生成する方式である。この方式ではObjectをOutlinerとSceneで共有しつつ、Graph node、keyframe、Diagnostic、recorded timepointをObject selectionへ偽装しない。Panelはsnapshotを購読するだけであり、受信したstateを別Panelへ再送しない。

#### 15.7.1 Snapshotとselection channel

```text
EditorAttentionSnapshotV1
  snapshot_id
  previous_snapshot_id optional
  session_id
  project_id
  project_revision
  contract_set_hash
  attention_generation
  active_channel optional
  channel_selections[0..5]
  keyboard_focus_ref optional
  accepted_intent_id optional

EditorAttentionChannelSelectionV1
  channel
  owner_selection_context_ref
  owner_revision
  owner_generation
  last_activated_attention_generation
  selection_content_hash
```

`channel_selections`はchannelごとに0または1件、次表のenum順、duplicateなしとする。空snapshotでは`active_channel=null`、一件以上ある場合は存在するchannel一件をactiveにする。Select／Toggle／Extendは対象channelを必ずactiveにし、`activate_channel`は非空の既存channelだけをactiveにする。active channelをClearして他channelが残る場合は`last_activated_attention_generation`最大のchannel、同値ならenum順の先頭をactiveにする。inactive channelは別channelの操作で暗黙clearせず、各owner selection contextの件数、primary、query、omitted range、continuationを保持する。

| channel | 内容と正本 | 自動同期する主surface |
|---|---|---|
| `authoring_target` | [Project State](../03-authoring/project-state.md)の`AuthoringSelectionContextV1`。Entity、World／Scene、owner-typed Source target | Outliner、Scene、World Outline、InspectorのAuthoring context |
| `asset` | Asset ownerのrevision-bound Asset selection／query context。10万件のquery selectionを列挙しない | Asset Browser、InspectorのAsset context、Import |
| `document_element` | owner-typed Graph node／port／edge、Timeline track／key／span、Source／Diff item | Graph、Timeline、Source、対応Inspector |
| `evidence` | canonical Diagnostic、log、Task failure、Debug event／span／query selection | Problems、Console、Build、Profiler、Debug details |
| `temporal` | exact animation tick／rangeまたは`DebugTimePointV1`／record range | Animation Timeline、Debug Timeline、明示的に対応するpreview |

Runtime objectとAI Proposalはselection channelにしない。Runtimeはexact Session／Build／Source mappingを持つoverlay projection、Proposalはproposal ID／base revision／target refを持つreview stateであり、現在選択が変わっても新targetへ付け替えない。`keyboard_focus_ref`はWindow内で実際にkeyboard inputを受ける一要素のID／generationであり、selection、Project target、AI authorizationを意味しない。hover、pointer capture、range anchor、focus column、scroll anchor、inline draftはPanel local stateに残す。

#### 15.7.2 Panel follow／pin binding

```text
EditorPanelContextBindingV1
  panel_instance_id
  mode = follow_active | follow_channel | pinned
  accepted_channels[1..5]
  followed_channel optional
  pinned_context_ref optional
  pinned_project_revision optional
  binding_generation
  last_resolved_attention_generation
```

- `follow_active`はactive channelが`accepted_channels`にある場合だけ更新する。非対応channelがactiveになった場合は最後のvalid contextを維持し、Panel headerに現在のcontext chipを表示する。別channelへ名前、位置、同一labelでfallbackしない。
- `follow_channel`は指定channelだけを追従する。Outliner／Sceneは既定で`authoring_target`、Problemsは`evidence`、Graph／Timelineは`document_element`と必要な`temporal`を使う。
- Inspectorの既定`accepted_channels`は`authoring_target | asset | document_element`とし、activeな対応channelのfieldを表示する。EvidenceとTemporalを選んでも最後のvalid Inspector contextを暗黙変更しない。
- `pinned`はPanel一件をexact contextへ固定するpresentation bindingであり、Project lock、file lock、edit permission、selection、approvalを変更しない。Panel headerへpin icon、`固定`、target short label、pinned／current revisionを文字で示す。選択変更後も固定対象を維持し、削除、permission loss、revision不整合では`missing | unavailable | stale`を明示して近い対象へ移さない。
- bindingはProjectまたはUndoへ保存しない。User Workspaceへ保存する場合はProject ID、schema-tagged Stable ref、owner、binding generationだけを保存し、再起動後に現在revisionで再解決する。解決不能ならempty／missing stateを表示する。

#### 15.7.3 Select、focus、reveal、pin

次の操作を一つの曖昧な「activate」へ統合しない。

| 操作 | 変更するstate | 変更しないstate |
|---|---|---|
| Select／Toggle／Extend／Clear | 指定channelのselection。非emptyなSelect／Toggle／Extendは必ず`active_channel`も更新 | keyboard focus、別channel、Project、Runtime、Proposal target |
| Focus element | `keyboard_focus_ref`一件 | selection、Panel binding、Project |
| Reveal target | scroll／expand／frame等のPanel local presentation | selection、focus、Project。必要なら別の登録済み`Select and Reveal`複合Actionを明示 |
| Pin／Unpin panel context | 指定Panel instanceのbinding | global selection、lock、authorization、Project |
| Select related target | owner-issued exact relationが指す別channelのselection | relation不明時の推測、元channelのselection |

```text
EditorAttentionIntentV1
  intent_id
  actor_kind
  input_channel
  source_panel_instance_id optional
  source_element_key optional
  target_element_key optional
  destination_panel_instance_id optional
  destination_resolver_id optional
  base_attention_generation
  base_binding_generation optional
  base_destination_generation optional
  project_revision
  kind = select_replace | select_toggle | select_extend | select_clear
       | activate_channel | focus_element | reveal_target | select_and_reveal
       | pin_panel_context | unpin_panel_context | select_related_target
  channel optional
  target_or_query_ref optional
  relation_ref optional
  propagation_id

EditorAttentionReductionResultV1
  intent_id
  status = accepted | rejected
  attention_snapshot_ref
  panel_binding_update_ref optional
  presentation_effect_ref optional
  feedback_ref optional
```

Humanの`pointer | keyboard | accessibility`では`source_panel_instance_id`と`source_element_key`をrequiredにする。`menu | toolbar | context_menu`ではsource elementをrequired、Panel-scoped Actionだけsource Panelもrequiredにし、Shell-scoped Actionではnullを許す。`command_palette | internal_ai | mcp | cli | test`では両方をnullにでき、その場合は登録済みSemantic Action／Command requestをsource identityとする。`propagation_id`は一つのuser intentから生じる`select_and_reveal`等の複合処理を一件へ束ね、Panel受信通知から新Intentを作るためには使用しない。

| intent kind | required | null／禁止 |
|---|---|---|
| `select_replace／toggle／extend` | channel、exact targetまたはowner query ref | relation ref |
| `select_clear／activate_channel` | channel。Activateは非emptyな既存channel | target／query、relation ref |
| `focus_element` | target element key／generation | channel、target／query、relation ref、destination resolver |
| `reveal_target` | exact target ref、destination Panel instanceまたはregistered resolverのexact一方、base destination generation | channel selection変更、target element key |
| `select_and_reveal` | channel、exact target ref、destination Panel instanceまたはregistered resolverのexact一方、base destination generation | implicit related target、分離された二Intent、Project transaction |
| `pin_panel_context` | destination Panel instance、schema-tagged exact context ref、base binding generation | destination resolver、relation ref、Project lock／authorization変更 |
| `unpin_panel_context` | destination Panel instance、base binding generation | target／query、destination resolver、relation ref |
| `select_related_target` | exact relation ref | caller指定のtarget override、name／path fallback |

Human Widget、keyboard、UIAと、登録済み`EditorSemanticActionV1`を使うAI／automationは同じIntent validatorへ収束する。C1の`pin_panel_context | unpin_panel_context`は`human`とfixtureの`automation/test`だけに許可し、AIはWorkspace／Panel bindingを変更しない。ReducerはUI session thread上でactor／inputの組合せ、source／target Panel・element generation、base attention generation、Project revision、owner selection／query revision、target存在、権限を検証する。結果は`attention_snapshot_ref`、0または1件の`panel_binding_update`、0または1件の`focus_or_reveal_effect`、typed feedbackを持つ。`accepted`はcurrentまたはnew snapshotを必ず返し、`rejected`はcurrent snapshotとrequired feedbackを返してbinding／effectをnullにする。Select／Activateでsemantic stateが変わる場合だけ新Attention snapshotを一件発行する。Focus intentはPlatform focus requestだけを発行し、実際の`focus gained` eventと同じWindow／element generationを確認してから`keyboard_focus_ref`を更新する。拒否／失敗時はfocus snapshotを先行変更しない。Pin／Unpinはbinding generationだけ、RevealはPanel presentation generationだけを更新し、attention generationを増やさない。`select_and_reveal`はsnapshot一件とreveal effect一件を同じreduction resultで返す。duplicate `intent_id`は同じresultを返し、semantic no-opは`accepted`＋current snapshotとしてgenerationとUIA eventを増やさない。stale generation、out-of-order intent、別Project target、omitted／unknown targetはtyped reasonで拒否する。

#### 15.7.4 Exact relationとcross-panel state projection

Graph nodeからEntity、Diagnosticからfield、Runtime objectからSource targetへ移る操作は、表示名や似たpathで関係を推測せず、ownerが発行する次のread-only relationだけを使う。

```text
EditorTargetRelationProjectionV1
  relation_ref
  source_kind = selection_channel | runtime_projection | proposal
  source_channel optional
  source_ref
  relation_kind = represents | belongs_to | bound_target
                | diagnostic_target | runtime_counterpart | proposal_target
  target_channel
  target_ref
  source_revision
  target_revision
  generation
```

relationがない、複数で曖昧、stale、generation不一致の場合は`Select related target`をdisabled reason付きで拒否する。Graph node、keyframe、Diagnosticを選んだだけでは`authoring_target`を変更しない。Runtime counterpartはSession、Build、Source mapping、generationがexact一致するときだけ関係を発行し、name fallbackを禁止する。

`source_kind=selection_channel`では`source_channel`をrequired、`runtime_projection | proposal`ではnullにする。`target_channel`は常にrequiredで、`target_ref`がそのowner selection contextへ適合することをrelation発行時とIntent受理時の両方で検証する。RelationはProjectまたはRuntime正本ではなく、exact owner refsから生成するDisposable projectionである。

各Panelが重ねる状態は同じselection snapshotへ埋め込まず、UI Frameworkがownerのread-only refを合成した次のprojectionを使う。

```text
EditorCrossPanelStateProjectionV1
  target_ref
  project_revision
  validation_refs[0..128]
  proposal_refs[0..128]
  runtime_projection_ref optional
  recorded_context_ref optional
  lock_refs[0..128]
  freshness
  semantic_action_projection_ref
  content_hash
```

validationはexact target／field／validator revision、Proposalはproposal ID／base revision、RuntimeはSession／Build／Source mapping／generation、recorded contextはDebug Session／timepoint／record revisionへ束縛する。`semantic_action_projection_ref`はactor、input channel、authorization context、Project revisionを固定した既存Semantic SnapshotのAction projectionであり、state projection自体へ権限を複製しない。Panelはowner stateを再判定せず、selection fill、focus ring、validation rail、proposal dash＋`AI提案`、Runtime glyph＋`Runtime`、recorded pin＋`Recorded`、stale clock＋revisionとして別layer表示する。色、motion、現在開いているPanelだけからauthorityまたはfreshnessを推測しない。

Shell statusはactive channelを、各Panel header／Context barは実際にbindingしたchannelをObject、Asset、Document、Evidence、Timeのicon＋localized textで示す。pinned Panelはさらにpin icon＋`固定`を表示する。Attention snapshot自体の変更は0 ms、revealのscroll／frameだけ§15.5の167 ms以内を許可し、reduced motionでは0 msにする。animation開始前に最終targetとdestinationを確定し、motion途中の座標をidentityにしない。

#### 15.7.5 Revision、削除、filter、Panel lifecycle

- Project revision更新後もStable refが一意に存在する場合だけowner contextを新revisionへ再発行する。曖昧、削除、permission lossでは該当targetだけを除外し、Authoring primaryの再選択はProject Stateのcanonical規則に従う。name、path、nearest itemへ移さない。
- filter、sort、virtualization、LODで非表示になったselectionは保持し、visible／hidden件数と`Reveal`を示す。omitted selectionを存在確認済みとしてProject writeへ使わない。
- active Panelを閉じてもselectionは維持する。keyboard focusは同Windowの直前focusable ancestor、active tab、shellの順で回復し、focus recoveryからselectionを作らない。
- Inspector pinの対象がstaleになってもcurrent selectionへ自動復帰しない。Userが`追従に戻す`または別targetを明示固定する。
- 選択変更はpending AI promptまたはProposalのtargetを変更しない。base attention generation／Project revisionが異なるものはstale表示し、rebaseまたは新しいContext Previewを要求する。

#### 15.7.6 AI、UI Automation、conformance

SelectionはAIへ渡せるContext候補であってauthorizationではない。AI ContextはUserがPreviewで許可したbounded channel slice、attention generation、Project revision、omitted rangeを含み、focus／hover／Panel boundsをtargetにしない。AIは登録済みSelect／Focus／Reveal Actionを要求できるが、Pin／Unpin、UIA、pointer、screen coordinate、Panel callbackを使わず、Project変更時は対象とexpected revisionをCommandへ再指定する。

UI Automationでは同じ`owner_selection_context_ref`を実際に投影するCollectionだけが標準Selection／SelectionItemを所有し、実keyboard focusだけがFocus changed eventを発行する。同じchannelのexact mirrorは同じselection stateへ更新できるが、relation、Inspector pin、validation、Runtime overlay等のpassive projectionをselected UIA itemへ変換しない。`EditorAttentionSnapshotV1`を一つの偽Selection containerまたはcustom UIA patternとして公開せず、passive projectionはItemStatus／descriptionで`選択対象`を示す。semantic no-opはeventなし、各providerは同一generationを一度だけ通知し、stale providerはElementNotAvailable相当で終了する。

| scenario ID | 必須内容 |
|---|---|
| `scenario.attention.typed-channel-isolation@1` | Object、Asset、Graph node、keyframe、Diagnostic、timepointを順に選び、別channelのselection、Proposal target、Runtime mappingを上書きしない |
| `scenario.attention.select-focus-reveal-pin@1` | select、focus、reveal、select-and-reveal、pin／unpinを個別検証し、暗黙選択、focus＝selection、pin＝lock／permissionを0件にする |
| `scenario.attention.reducer-order-idempotency@1` | duplicate intent、semantic no-op、stale／out-of-order generation、Panel再生成で一snapshot、一event、feedback loop 0件 |
| `scenario.attention.panel-follow-pin@1` | Inspectorの三accepted channel、unsupported channel時のlast-valid、Entity A固定後のEntity B選択、stale／missing固定対象を検証 |
| `scenario.attention.exact-related-target@1` | Graph／Diagnostic／Runtimeからexact relationで明示遷移し、missing／ambiguous／stale relationとname／path fallbackを拒否 |
| `scenario.attention.state-overlay-authority@1` | validation、Proposal、Runtime、recorded、lock、freshnessがexact owner ref／revisionへ解決し、一selection snapshotまたは一badgeへの統合0件 |
| `scenario.attention.revision-deletion-filter@1` | rename、re-shard、sort、filter、virtualization、target delete、permission loss、Project revision drift後もStable refまたは明示missingへ決定論的に収束 |
| `scenario.attention.ai-context-not-authority@1` | active／関連channelのbounded Preview、omitted range、stale Proposal、再指定target／expected revisionを検証し、focus／selection由来の追加権限0件 |
| `scenario.attention.uia-event-separation@1` | Collection selection event、keyboard focus event、passive ItemStatus、no-op抑止、stale providerをNarrator／NVDA／UIA clientで分離 |
| `scenario.attention.five-reference-integration@1` | Tree／List、Property／Form、Canvas、Graph／Timeline、Diagnosticsが同じsnapshot／relation／state projectionを使い、Panel固有同期contract 0件 |

`fixture.cross-panel.attention@1`はOutlinerのEntity A、Sceneの同Target、Entity Aに固定したInspector、Asset BrowserのAsset、Gameplay Graph node、Animation key、ProblemsのDiagnostic、Debug Timelineのrecorded timepoint、Runtime counterpart、Entity A向けstale AI Proposalを同時に持つ。Entity B選択、Graph node選択、Diagnosticの`対象を選択`、timepoint同期、Inspectorの追従復帰、filterで隠れたselection、Entity削除、Project revision更新をmanual／keyboard／Narrator／NVDA／UIA／AIで順に実行し、typed channel、focus、reveal、pin、relation、owner state、Action可否が一致するまでCross-Panel Contractをclosedと扱わない。

### 15.8 Reference Fixture Manifest／Evidence Contract

本節はReference Designの文章、個別Widget scenario、実行環境、期待結果、取得Evidenceを一つの再現可能な検証入力へ閉じる。Screenshot directoryだけをfixtureと呼ぶ方式と、Panelごとに手書きautomation scriptを持つ方式は採用しない。前者はSemantic／権限／eventを証明できず、後者はHuman、keyboard、UIA、AIでlogical intentと待機条件が分岐するためである。

採用方式は、closed `EditorReferenceFixtureManifestV1`からlogical intentとexplicit coverage tupleを読み、同じcheckpointをauthoritative state、Semantic、layout、visual、UIA、Command、performanceの独立oracleで判定し、content-addressed Evidence Bundleへ束ねる方式である。Manifest、baseline、Evidenceはtest artifactであり、Project、Workspace、Undo、Runtime正本へ保存せず、既定AI Contextへ自動添付しない。

#### 15.8.1 Registry、Manifest、applicability

```text
EditorReferenceFixtureRegistryV1
  registry_id
  registry_version
  entries[1..256]:
    fixture_id
    fixture_version
    owner_ref
    manifest_ref
    manifest_content_hash
    reference_role_refs[1..9]
    capability_applicability_ref
  registry_content_hash

EditorReferenceFixtureManifestV1
  fixture_id
  fixture_version
  owner_ref
  contract_set_hash
  fixture_source_ref
  fixture_source_content_hash
  fixture_seed
  initial_project_revision
  initial_project_root_hash
  workspace_ref
  workspace_content_hash
  panel_catalog_hash
  widget_pattern_registry_hash
  comparison_profile_registry_ref
  comparison_profile_registry_hash
  baseline_registry_ref
  baseline_registry_hash
  environment_profile_refs[1..64]
  scenario_refs[1..256]
  coverage_matrix_ref
  coverage_matrix_content_hash
  verification_gate_policy_ref
  verification_gate_policy_hash
  required_oracle_kinds[7]
  manifest_content_hash
```

Registry entryは`fixture_id`順、Manifestの配列は各typed ID順のcanonical order、全配列duplicateなしとする。`manifest_content_hash`はASCII `MIRAKAN_EDITOR_REFERENCE_FIXTURE_MANIFEST_V1`と自己Fieldだけを除くcount／length-framed canonical bytesから計算し、実行結果、baseline change approval、Verification Receiptをpreimageへ含めない。ManifestはProject fixture source、Workspace、Panel catalog、Widget Pattern Registry、Comparison Profile Registry、Baseline Registry、Coverage Matrix、Gate Policy、Contract setの一つでも変われば別version／hashとなり、旧Evidenceを再利用しない。

§15.8で定義する他の`*_content_hash`／`evidence_root_hash`も型名に対応するASCII domain separation、全required Field、presence bit、count、lengthを含むclosed canonical bytesから自己hash Fieldだけを除いて計算する。Refはlogical ID、version、content hashを持ち、同じlogical ID／versionの別hash、hash-only ref、unknown Fieldを拒否する。`fixture_seed`はunsigned 64-bit、`initial_project_revision`はexact Project revisionであり、zero／empty sentinelをoptionalの代用にしない。

`verification_gate_policy_ref`は`VerificationReceiptV1`を発行できる信頼済みRunner、必須input／output artifact、command ID、result、freshnessを固定する既存Gate Policyへ解決する。`required_oracle_kinds`は`authoritative_state | semantic | layout | visual | uia_tree_event | command_trace | performance`のexact七件をenum順、duplicateなしで持ち、Coverage Matrixは各kindを最低一tuple含まなければならない。適用不能に見えるkindも省略せず、変更禁止、不在、上限未満等のtyped negative expectationで判定する。禁止結果は自由文の別Registryへ置かず、negative scenario、expected typed rejection、corresponding oracleのexact tupleとして表す。

`capability_applicability_ref`はcurrent Capability snapshotから当該Manifestを`required | prohibited`のexact一方へ解決する。required集合はRegistry全体から決定論的に生成し、実行request／Evidence Bundleとset equalityにする。prohibited Manifestを`skipped`、`not_applicable pass`、空Evidenceとして成功へ含めず、未Activation surfaceを画像またはplanned Actionで補完しない。Reference ManifestはProduct Planの`FixtureRegistryV1`へ新しいProduct fixtureを追加せず、`fixture.product.windows-empty-scene`等の既存Product fixtureが消費するcomponent conformance入力である。

#### 15.8.2 Environment profileとdeterministic clock

```text
EditorReferenceEnvironmentProfileV1
  environment_profile_id
  target_profile_ref
  reference_hardware_profile_ref
  os_image_ref
  gpu_driver_profile_ref
  monitor_topology_ref
  window_client_extent_px
  dpi_scale_rational
  editor_ui_scale_rational
  system_text_scale_rational
  theme_profile_ref
  density = compact | standard | comfortable
  locale_ref
  input_language_ref
  motion_preference_snapshot_ref
  reduced_motion: bool
  advanced_effects_preference_snapshot_ref
  advanced_effects_policy_ref
  scrollbar_preference_snapshot_ref
  scroll_chrome_contract_ref
  message_duration_preference_snapshot_ref
  font_asset_set_hash
  icon_asset_set_hash
  renderer_configuration_hash
  ui_clock_profile_ref
  environment_profile_content_hash

EditorDeterministicUiClockProfileV1
  ui_clock_profile_id
  tick_period_ns
  initial_tick
  maximum_tick
  advance_policy = explicit_step_only
  ui_clock_profile_content_hash
```

OS、GPU、driver、hardwareのexact値はToolchain／Performance ownerのprofileを参照し、本書へ複写しない。`reference_hardware_profile_ref`は[Performance／Capacity §8.1](../04-runtime/performance-capacity.md#81-editor-reference-01-performance-profile)のA／B `ReferenceHardwareProfileV1`一件へ解決し、`os_image_ref`、`gpu_driver_profile_ref`、`monitor_topology_ref`は同profileの`os_image_ref`、`gpu_driver_package_ref`、`monitor_topology_edid_ref`とそれぞれartifact identityがexact一致しなければならない。`os_image_ref`はさらに[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)のHost UI font resolved-file evidenceが使うCI OS imageとbyte equalityにする。driver／EDIDを`style_font_generation`、font／icon asset hash、theme snapshotへ複写せず、逆にfont assetをhardware profileへ複写しない。三owner bindingの一Fieldでも不一致なら`infrastructure_error`であり、closest monitor、同GPU family、別driver／OS image、生成時刻から補完しない。`system_text_scale_rational`は既約なpositive rationalで`1/1 <= value <= 9/4`、E06は`2/1`とする。Windows Adapterが読む`UISettings.TextScaleFactor`はadapter-private finite `float64`のまま処理し、Reference capture時だけProfileのexact rationalへ一致しなければ`infrastructure_error`としてnearest percentへ丸めない。`theme_profile_ref`はDesign Systemの`theme.editor.light@1`または`theme.editor.dark@1`、またはPlatform Ownerが解決したWindows Contrast Theme／system-color snapshotのexact refである。Light／Dark refは§15.1の全token tableと`color_mode=light | dark`を同一content hashへ含め、Contrast refは`HCF_HIGHCONTRASTON`と§13.2.2のrequired `GetSysColor` valueを同一content hashへ含める。表示名、scheme名、順番、既知palette、previous snapshotをidentityにしない。Visual baselineはexact `environment_profile_id`ごとに一件だけ解決し、closest device、nearest DPI、同じGPU family、別High Contrast themeを代用しない。実環境がProfileと一致しなければtest failureではなく`infrastructure_error`とし、比較を開始しない。

`motion_preference_snapshot_ref`は§13.2.3の完成snapshotへID／version／hashで解決し、`reduced_motion`は`effective_motion == reduced`から導出するboolである。両Fieldが矛盾するProfile、`source_api`が異なるsnapshot、またはactual `SPI_GETCLIENTAREAANIMATION`値と一致しないcaptureは`infrastructure_error`として比較を開始しない。`full` Profileは`client_area_animations_enabled=true`、E11は`false`をexactに持つ。app-local preference、前回値、clock profileからmotion modeを推測しない。

`advanced_effects_preference_snapshot_ref`は§13.2.4の完成`EditorAdvancedEffectsPreferenceSnapshotV1`へID／version／hashで解決し、`advanced_effects_policy_ref`はC1ではimmutableな`effects.editor.opaque-only@1`だけを参照する。E00–E13は外部Reference environmentが固定した`effects.windows.advanced-enabled@1`（`UISettings.AdvancedEffectsEnabled=true`）をexactに持つが、E07–E10を含めてmaterial policyは緩和しない。全14 static profileは同じopaque token／layout／semanticを使い、`true`をMica／Acrylicの許可、falseを別visual baselineの根拠へ読み替えない。Profileとactual propertyまたはpolicy refが一致しないcaptureは`infrastructure_error`とし、同一running Windowでの`TRUE -> FALSE -> TRUE`は§13.2.4のPlatform Adapter conformanceで別Evidence化する。このprobeはE14、coverage entry、visual baselineを追加しない。

`scrollbar_preference_snapshot_ref`は§13.2.5の完成`EditorScrollBarPreferenceSnapshotV1`へID／version／hashで解決し、`scroll_chrome_contract_ref`は`scrollbar.chrome.editor@1`だけを参照する。E00–E13の全14 static Profileは外部Reference environmentで`UISettings.AutoHideScrollBars=false`を固定した`scrollbars.windows.persistent@1`を持つ。overflowするaxisは`persistent`、overflowしないaxisは`absent`であり、visual expected subjectはそのcurrent presentationを明示記録してproperty値、pointer座標、Screenshotだけから推測しない。Profileとactual property／contract refが一致しないcaptureは`infrastructure_error`とし、同一running Windowでの`FALSE -> TRUE -> FALSE`は§13.2.5のPlatform Adapter conformanceで別Evidence化する。このprobeはE14、coverage entry、visual baselineを追加しない。

`message_duration_preference_snapshot_ref`は§13.2.6の完成`EditorMessageDurationPreferenceSnapshotV1`へID／version／hashで解決する。E00–E13は外部Reference environmentでread-backしたexact SPI snapshotを持つが、static visual expected subjectはnotificationの`visible_initial`またはowner surfaceのpersistent recordだけで、auto-dismiss deadlineをScreenshot、wall clock、固定5秒から推測しない。Profileとactual SPI valueが一致しないcaptureは`infrastructure_error`とし、`d1 -> d2 -> d1`のrunning Window transitionは§13.2.6のPlatform Adapter conformanceで別Evidence化する。このprobeはE14、coverage entry、visual baselineを追加しない。

`ui_clock_profile_ref`は完成`EditorDeterministicUiClockProfileV1`へID／version／hashで解決し、`tick_period_ns`はpositive、`initial_tick <= maximum_tick`、tick加算はoverflowしないことを要求する。Reference Fixtureの既定Profile `clock.editor-reference.millisecond@1`は`tick_period_ns=1,000,000`、`initial_tick=0`、`maximum_tick=600,000`とする。同じlogical ID／versionの別値を許さず、`83 ms`はexact 83 tick、`167 ms`は167 tick、`250 ms`は250 tickへ変換する。1 tick未満のduration、fractional tick、wall-clockとの自動同期を禁止する。

`effective_motion=full`のUI motion、caret blink、Task display、Runtime preview、recorded cursorは同Profileのexact tickで進める。`reduced`では§13.2.3／§15.5のstatic final presentationを使い、blink／loopのclock advanceを発行しない。通常のconformanceでwall-clock sleep、frame count、CPU速度、network idleを完了条件にしない。Performance oracleだけがPerformance Ownerのmonotonic measurement clockを別に使い、virtual UI clockの値からdurationまたはSLOを作らない。

#### 15.8.3 Scenario、driver、barrier

```text
EditorReferenceScenarioV1
  scenario_id
  scenario_version
  initial_checkpoint_ref
  intent_step_refs[1..256]
  final_checkpoint_ref
  reset_mode = reload_immutable_fixture_source
  scenario_content_hash

EditorReferenceIntentStepV1
  logical_intent_id
  step_kind = editor_intent | semantic_action | typed_command
            | registered_fault_injection | clock_advance | observe
  target_ref optional
  action_or_fault_ref optional
  typed_arguments_ref optional
  typed_arguments_hash optional
  base_project_revision
  base_attention_generation optional
  driver_bindings[1..8]
  before_barrier_ref
  after_barrier_ref
  checkpoint_ref optional

EditorReferenceDriverBindingV1
  driver_binding_id
  logical_intent_id
  driver_kind = human_supervised | automation_pointer
              | automation_keyboard | uia_client
              | ai_semantic_contract | test_harness
  actor_kind
  input_channel
  source_element_key optional
  target_element_key optional
  driver_action_ref
  expected_primary_intent_or_command_ref optional
  expected_unavailable_action_ref optional
  expected_auxiliary_intent_refs[0..7]
  driver_binding_content_hash

EditorFixtureBarrierV1
  barrier_id
  barrier_kind = attention_generation | owner_projection_generation
               | command_terminal | task_terminal | render_presented
               | semantic_snapshot_published | uia_event_observed
               | virtual_clock_tick
  subject_ref
  expected_generation_or_state
  deadline_profile_ref
  barrier_content_hash
```

`driver_bindings`は`EditorReferenceDriverBindingV1` refであり、actor kind／input channelの組合せを§10.2とbyte equalityにする。`human_supervised`は実入力、`automation_pointer | automation_keyboard`はtest processだけがPlatform eventを注入してEditor側では`human/pointer | human/keyboard`として処理し、synthetic provenanceをBundleへ別記する。`uia_client`は標準Providerを外部clientから`human/accessibility`として操作し、`ai_semantic_contract`は`ai/internal_ai | mcp`、`test_harness`は`automation/test`だけを許す。

Action branchはclosed exact一方である。利用可能なEditor Intent／Semantic Action／Commandでは`expected_primary_intent_or_command_ref`をrequired、`expected_unavailable_action_ref=null`とする。disabled／read-only／locked等の利用不能性を検証するbindingではprimaryをnull、`expected_unavailable_action_ref`をenabled=false、typed reason、target、revision、forbidden Commandへ解決する完成Action projection refとしてrequiredにする。このbranchではCommand request、ChangeSet、Receiptを発行しない。Gatewayのdefense-in-depth rejectionは別`test_harness` bindingでprimary Commandを直接要求し、typed rejectionをCommand oracleへ記録する。`registered_fault_injection | clock_advance | observe`では両refをnullとする。

`expected_auxiliary_intent_refs`はprimaryまたはunavailable Actionに付随してinput contractが必ず発行するfocus／reveal等のEditor Intentだけをordered refで持ち、Project mutation、別target selection、primaryの代用を許さない。fault／clock／observeではexact `[]`とする。Faultの期待効果はproduction Intent／Commandを捏造せず、registered fault refと対応oracleで表す。test-only driver kindをProduction Command RegistryまたはAI Actionへ公開しない。

全driverは同じ`logical_intent_id`、target、typed arguments、primary Intent／Command、expected checkpointへ収束する。auxiliary Intentはdriverごとにexact比較し、pointer／keyboardのfocusをUIA／AIへ捏造したり、差をnormalizationしたりしない。Pointer driverは同じSemantic／layout generationのelement boundsを実行時に解決できるが、Manifestへscreen coordinateを保存しない。UIA driverは標準property／pattern／eventだけ、AI driverはmodel推論を介さない決定論的な`EditorSemanticActionV1` contract callだけを使う。実Modelのtool-selection品質はAI evaluationの別Evidenceであり、UI Framework conformanceを代用しない。

| `step_kind` | required branch | 禁止 |
|---|---|---|
| `editor_intent` | §15.7のkind別required fieldを満たすIntent template、適用可能ならtarget／typed arguments | Intent規則を緩めるfixture-only field |
| `semantic_action` | exact Semantic Action ref、Stable target ref、argument schemaに適合するtyped arguments | Pattern ID、label、UIA、座標からAction生成 |
| `typed_command` | exact Command ID、Command schemaが要求するtarget／revision／typed arguments | 表示text parse、expected revision省略 |
| `registered_fault_injection` | test-only Fault Registry ref、同Fault schemaが要求するtarget | Human／AI driver、任意callback、Project Source write |
| `clock_advance` | positive exact virtual tick delta、`test_harness` driver | wall-clock duration、frame count、Action／fault ref |
| `observe` | checkpoint ref、`test_harness`または`human_supervised` driver | state mutation、Action／fault ref、typed arguments |

`human_supervised`はReviewerのチェックだけでpassにせず、信頼済みRunnerが実ProcessのIntent、Semantic target、Command result、checkpoint artifactを観測する。`registered_fault_injection`はtest-only Registryのexact faultだけを使い、任意callback、Process memory write、Project Source改変を許さない。各scenario後はimmutable fixture sourceから再起動／reloadし、前scenarioのWorkspace state、selection、cache、Task、Runtime Sessionを次scenarioのpreconditionへ流用しない。

Barrierの`subject_ref`と`expected_generation_or_state`はkindごとのclosed tagged payloadである。Attention／owner projectionはexact ref＋generation、Command／Taskはrequest／task ref＋closed terminal result、renderはsurface ref＋content／layout／present generation、Semanticはsnapshot ref＋generation／hash、UIAはprovider root＋correlation ID＋expected event、clockはclock ref＋exact tickを要求し、別kindのpayloadまたはunknown fieldを拒否する。

Barrierは「何が完了したか」をtyped generation／terminal stateで表し、deadlineはhangを`infrastructure_error`へ分類する上限にだけ使う。deadline到達を期待状態成立とみなさず、sleep後のScreenshot、eventが来るまで無制限待機、async request件数0だけをquiescenceにしない。

#### 15.8.4 Checkpoint、coverage、oracle

```text
EditorReferenceCheckpointV1
  checkpoint_id
  expected_project_revision
  expected_project_root_hash
  expected_attention_generation optional
  expected_panel_binding_set_hash
  expected_owner_projection_set_hash
  expected_virtual_clock_tick
  checkpoint_content_hash

EditorReferenceCoverageMatrixV1
  coverage_matrix_id
  coverage_matrix_version
  fixture_id
  fixture_version
  entries[1..4096]:
    scenario_ref
    checkpoint_ref
    driver_binding_ref
    environment_profile_ref
    oracle_ref
    coverage_entry_hash
  coverage_matrix_content_hash

EditorReferenceOracleV1
  oracle_id
  oracle_version
  oracle_kind = authoritative_state | semantic | layout | visual
              | uia_tree_event | command_trace | performance
  expected_oracle_subject_ref
  expected_oracle_subject_hash
  comparison_profile_ref
  comparison_profile_hash
  registered_dynamic_region_set_ref
  registered_dynamic_region_set_hash
  oracle_content_hash

EditorReferenceComparisonResultV1
  comparison_result_id
  coverage_entry_hash
  oracle_kind
  expected_oracle_subject_ref
  expected_oracle_subject_hash
  observed_artifact_ref optional
  observed_artifact_hash optional
  comparison_profile_ref
  comparison_profile_hash
  result = pass | fail | infrastructure_error | cancelled
  diagnostic_refs[0..128]
  comparison_result_content_hash

EditorExpectedAuthoritativeStateSubjectV1
  subject_id
  subject_version
  checkpoint_ref
  environment_profile_ref
  expected_project_revision
  expected_project_root_hash
  expected_attention_snapshot_ref
  expected_attention_snapshot_hash
  expected_panel_binding_set_ref
  expected_panel_binding_set_hash
  expected_owner_projection_set_ref
  expected_owner_projection_set_hash
  expected_runtime_projection_set_ref optional
  expected_runtime_projection_set_hash optional
  subject_content_hash

EditorExpectedSemanticSubjectV1
  subject_id
  subject_version
  checkpoint_ref
  environment_profile_ref
  expected_semantic_snapshot_ref
  expected_semantic_snapshot_hash
  expected_semantic_content_hash
  required_element_set_ref
  required_element_set_hash
  forbidden_element_set_ref
  forbidden_element_set_hash
  subject_content_hash

EditorExpectedLayoutSubjectV1
  subject_id
  subject_version
  checkpoint_ref
  environment_profile_ref
  expected_layout_artifact_ref
  expected_layout_artifact_hash
  expected_layout_content_hash
  required_visible_region_set_ref
  required_visible_region_set_hash
  expected_clip_overlap_offscreen_set_ref
  expected_clip_overlap_offscreen_set_hash
  subject_content_hash

EditorExpectedVisualSubjectV1
  subject_id
  subject_version
  checkpoint_ref
  environment_profile_ref
  baseline_image_ref
  baseline_image_hash
  capture_surface_ref
  capture_surface_hash
  linear_color_profile_ref
  linear_color_profile_hash
  comparison_profile_ref
  comparison_profile_hash
  registered_dynamic_region_set_ref
  registered_dynamic_region_set_hash
  subject_content_hash

EditorExpectedUiaTreeEventSubjectV1
  subject_id
  subject_version
  checkpoint_ref
  environment_profile_ref
  expected_provider_tree_ref
  expected_provider_tree_hash
  expected_ordered_event_trace_ref
  expected_ordered_event_trace_hash
  required_property_pattern_set_ref
  required_property_pattern_set_hash
  forbidden_event_set_ref
  forbidden_event_set_hash
  normalization_profile_ref
  normalization_profile_hash
  subject_content_hash

EditorExpectedCommandTraceSubjectV1
  subject_id
  subject_version
  checkpoint_ref
  environment_profile_ref
  expected_ordered_command_trace_ref
  expected_ordered_command_trace_hash
  expected_terminal_result_ref
  expected_terminal_result_hash
  before_project_revision
  before_project_root_hash
  after_project_revision
  after_project_root_hash
  expected_receipt_set_ref
  expected_receipt_set_hash
  forbidden_command_set_ref
  forbidden_command_set_hash
  subject_content_hash

EditorExpectedPerformanceSubjectV1
  subject_id
  subject_version
  checkpoint_ref
  environment_profile_ref
  workload_profile_ref
  workload_profile_hash
  warmup_profile_ref
  warmup_profile_hash
  sample_policy_ref
  sample_policy_hash
  metric_threshold_set_ref
  metric_threshold_set_hash
  result_aggregation_ref
  result_aggregation_hash
  subject_content_hash
```

`coverage_entry_hash`はASCII `MIRAKAN_EDITOR_REFERENCE_COVERAGE_ENTRY_V1`と、scenario、checkpoint、driver binding、Environment Profile、oracleの各logical ID／version／content hashをcount／length-framed canonical bytesへ直列化して計算する。entriesは同tupleのtyped ID順、次いで各content hash順に並べ、同じ`coverage_entry_hash`、同じ五tuple、またはhash衝突を拒否する。Bundleとbaseline change subjectはこのhashを同一coverage identityとして参照する。

`EditorReferenceOracleV1.expected_oracle_subject_ref`のtypeは`oracle_kind`でclosedに決まる。`authoritative_state`、`semantic`、`layout`、`visual`、`uia_tree_event`、`command_trace`、`performance`は上記同順の`EditorExpected*SubjectV1`以外を参照できず、共通JSON、任意assertion script、Screenshot注釈、message textを期待値へ使わない。全subjectはcheckpointとEnvironment ProfileをOracleおよびCoverage entryとbyte equalityにし、ref/hash pairを完成artifactへ解決する。optional runtime projection ref／hashは両presentまたは両absent、Visual subjectのcapture surface、linear color、comparison profile、dynamic region set、UIA subjectのnormalization profile、Performance subjectのworkload、warm-up、sample、threshold、aggregationは全てref/hash pairを要求する。Visual subjectのcomparison profileとdynamic region setはOracleの同Fieldとbyte equality、UIA normalization profileはcomparison profileが参照するexact allowlistとbyte equalityにする。

`expected_clip_overlap_offscreen_set_ref`は期待される問題集合であり、通常は完成typed empty setを参照する。空ref、missing artifact、`[]`の表示文字列で代用しない。Command subjectのno-op／拒否caseも完成ordered trace、terminal result、before／after同値、完成empty Receipt setを持つ。Performance subjectは閾値と集計だけを所有し、correctness、Semantic、Visualのpassを内包しない。

Coverageを「全組合せ」「代表環境」等の自然文へ委ねない。Manifestが参照するMatrixの各entryをrequired tupleとし、Runnerの観測tuple集合とexact set equalityにする。Dark＋四High Contrast、三density、DPI／UI scale、200% Font、reduced motion、日本語／ASCII／code／数値／path、二Windows Reference hardware、各input driverの必要組合せは明示entryとして列挙する。組合せ爆発を避ける場合も、削除したtupleを暗黙pairwise coverageと呼ばず、Ownerが別Matrix versionで理由と保持するaxis pairを固定する。

| oracle | comparisonと所有範囲 |
|---|---|
| `authoritative_state` | Project root、Attention、owner projection、Runtime／recorded refのexact hash。UI表示結果から正本を逆算しない |
| `semantic` | bounds、native handle、glyph outline／run cacheを除く`EditorSemanticSnapshotV1`のlocalized Name／Value／message、role、Stable target、state axis、Action、revision、omitted rangeをexact canonical比較 |
| `layout` | logical unitのbounds、baseline、clip、z-order、hit target、reflow resultをexact profile比較。physical pixel screenshotから復元しない |
| `visual` | exact Environment Profileのreference image、linear color space、registered comparison profileでbounded diff。意味の合否を代用しない |
| `uia_tree_event` | realized UIA tree、property、control patternと、correlation ID付きevent traceのexact／ordered比較。runtime provider IDとtimestampだけ登録済みnormalizationで除外 |
| `command_trace` | Intent、Command request、typed result／rejection、ChangeSet／Receipt refのordered trace。表示message parseを使わない |
| `performance` | Performance Ownerのworkload／Target／metric／percentile／threshold refへ比較し、画像差またはvirtual clockから算出しない |

`comparison_profile_ref/hash`はoracle kindごとのclosed Registryへ解決し、arbitrary script、regex callback、plugin codeを持たない。normalizationが除外できるのはmanifest instanceで変化するruntime provider ID、trusted monotonic timestamp、content-addressed artifactの保管pathだけであり、target、revision、generation、role、state、Action、Diagnostic、event kind／orderを除外しない。個々のProfileがallowlistへ列挙しないFieldには、この上限内であってもnormalizationを適用しない。

`EditorReferenceComparisonResultV1`のexpected subject、oracle kind、comparison profile ref/hashはCoverage entryが参照するOracleとbyte equalityにする。`pass | fail`ではobserved artifact ref／hashを両required、`infrastructure_error | cancelled`では両presentまたは両absentのexact一方とし、partial pairを拒否する。Diagnosticは比較器またはRunnerが発行したtyped refだけを許し、message textをresultへ変換しない。

`registered_dynamic_region_set_ref/hash`は全oracleで必須とし、`oracle_kind=visual`以外はcanonical `dynamic-region-set.editor.empty@1`へ解決する。Visual comparisonはProfileごとにthresholdと最大global／semantic-region errorを固定し、失敗時に自動緩和しない。dynamic regionはcaret、OS-owned transient、明示的なrecorded uncertainty等のsemantic regionだけを許し、free rectangle、Panel全体、text／icon／focus／validation／Proposal／Runtime／stale領域をmaskしない。virtual clock、fixed seed、synthetic dataで固定できる揺らぎをmaskへ逃がさない。

#### 15.8.5 Comparison Profile Registry

比較規則はRunner実装、CI script、画像review UIへ分散させず、次のRegistryを正本にする。Registry entryは`comparison_profile_id`のUTF-8 byte順、同ID／versionの重複禁止、全ref/hash解決必須である。

```text
EditorReferenceComparisonProfileRegistryV1
  registry_id = registry.editor-reference.comparison-profile
  format_major = 1
  revision
  entries[7..256]:
    comparison_profile_ref
    comparison_profile_hash
    oracle_kind
    comparator_contract_ref
    comparator_contract_hash
  registry_content_hash

EditorReferenceComparisonProfileV1
  comparison_profile_id
  comparison_profile_version
  oracle_kind
  comparator_kind = canonical_artifact_exact | semantic_tree_exact
                  | layout_q16_exact | visual_rgba8_srgb_bounded
                  | uia_control_view_exact | command_trace_exact
                  | performance_absolute_and_regression
  comparator_parameter_ref
  comparator_parameter_hash
  normalization_profile_ref
  normalization_profile_hash
  difference_schema_ref
  difference_schema_hash
  qualification_ref optional
  qualification_hash optional
  comparison_profile_content_hash

EditorExactComparatorParametersV1
  parameter_id
  parameter_version
  oracle_kind = authoritative_state | semantic | command_trace
  canonical_schema_ref
  canonical_schema_hash
  order_policy = schema_declared | tree_preorder | sequence_index
  field_presence_policy = exact
  missing_extra_duplicate_policy = fail
  string_policy = canonical_utf8_byte_exact
  numeric_policy = typed_canonical_byte_exact
  parameter_content_hash

EditorLayoutComparatorParametersV1
  parameter_id
  parameter_version
  coordinate_encoding = signed_q16_16_lu
  coordinate_comparison = exact
  compared_field_set_ref
  compared_field_set_hash
  forbidden_physical_pixel_field_set_ref
  forbidden_physical_pixel_field_set_hash
  missing_extra_duplicate_policy = fail
  parameter_content_hash

EditorVisualComparatorParametersV1
  parameter_id
  parameter_version
  capture_stage
  pixel_format
  extent_policy
  row_pitch_policy
  linear_color_lut_ref
  linear_color_lut_hash
  encoded_rgb_channel_abs_lsb_max
  alpha_channel_abs_lsb_max
  global_mean_linear_rgb_error_q0_16_max
  semantic_region_mean_error_q0_16_max
  critical_cue_mean_error_q0_16_max
  pixel_over_channel_limit_count_max
  missing_or_extra_pixel_count_max
  contrast_assertion_profile_ref
  contrast_assertion_profile_hash
  dynamic_region_policy_ref
  dynamic_region_policy_hash
  parameter_content_hash

EditorUiaComparatorParametersV1
  parameter_id
  parameter_version
  tree_view = control_view
  root_scope_ref
  root_scope_hash
  element_match_policy = root_parent_automation-id_control-type_sibling-occurrence
  runtime_id_mapping = tree_preorder_bijection
  event_order_policy = exact_callback_sequence
  property_pattern_value_policy = typed_exact
  normalization_profile_ref
  normalization_profile_hash
  missing_extra_duplicate_policy = fail
  parameter_content_hash

EditorPerformanceComparatorParametersV1
  parameter_id
  parameter_version
  sample_policy_ref
  sample_policy_hash
  aggregation_ref
  aggregation_hash
  absolute_threshold_set_refs[1..32]
  absolute_threshold_set_hashes[1..32]
  relative_regression_ppm_max
  hardware_aggregation = independent_all_pass
  required_soak_profile_ref
  required_soak_profile_hash
  parameter_content_hash

EditorReferenceNormalizationProfileV1
  normalization_profile_id
  normalization_profile_version
  oracle_kind
  allowed_transform_kinds[0..3] =
    provider_runtime_id_to_stable_ordinal
    | omit_trusted_monotonic_timestamp_preserve_order
    | storage_path_to_artifact_hash
  allowed_schema_field_refs[0..16]
  rejected_schema_field_refs[0..128]
  normalization_profile_content_hash

EditorReferenceDifferenceRecordV1
  difference_id
  coverage_entry_hash
  oracle_kind
  difference_kind = missing | extra | duplicate | wrong_value | wrong_order
                  | outside_tolerance | threshold_exceeded | normalization_rejected
  schema_field_ref
  semantic_target_ref optional
  semantic_region_ref optional
  sequence_index optional
  metric_ref optional
  expected_value_artifact_ref optional
  expected_value_artifact_hash optional
  observed_value_artifact_ref optional
  observed_value_artifact_hash optional
  difference_content_hash
```

`EditorReferenceComparisonProfileV1.comparator_parameter_ref`のtypeは`comparator_kind`でclosedに決まる。`canonical_artifact_exact | semantic_tree_exact | command_trace_exact`は`EditorExactComparatorParametersV1`、`layout_q16_exact`は`EditorLayoutComparatorParametersV1`、`visual_rgba8_srgb_bounded`は`EditorVisualComparatorParametersV1`、`uia_control_view_exact`は`EditorUiaComparatorParametersV1`、`performance_absolute_and_regression`は`EditorPerformanceComparatorParametersV1`以外を参照できない。`normalization_profile_ref/hash`は全Profileで必須とし、normalizationなしは`normalization.editor.none@1`へ解決する。`qualification_ref/hash`はVisualとPerformanceだけrequired、他kindは両absentとし、partial pairを拒否する。Profileが参照するQualificationは同じcomparator parameterとprerequisite artifactを束縛するが、Profile自身へback-referenceせずcontent-hash cycleを作らない。

`EditorReferenceDifferenceRecordV1`は比較器が発行するbounded machine-readable差分であり、AIはこのrecordを説明できるが生成、削除、severity変更、passへの変換を行わない。`missing`はexpected pairだけ、`extra`はobserved pairだけ、その他は両pairをrequiredとし、ref/hashのpartial pairを拒否する。値そのものがsecretまたは上限超過なら値を埋めず、Ownerが定めるtyped redacted value artifactを参照する。自由文path、JSON Pointer、正規表現、pixel座標だけをidentityにせず、schema Field、Stable target、semantic region、sequence、metricの該当するFieldを使う。

初版Registry `registry.editor-reference.comparison-profile@1`は`revision=1`で次のexact七entryだけを持つ。別Fixtureが同じProfileを再利用してよいが、oracle kindの読み替え、同じID／versionでの閾値変更、Profile未登録時のRunner既定値を許さない。

| oracle kind | exact Comparison Profile | exact comparator contract | comparatorと初版規則 |
|---|---|---|---|
| `authoritative_state` | `comparison.editor.authoritative.byte-exact@1` | `comparator.editor.authoritative.byte-exact@1` | completed canonical owner artifactを全Field byte equality。missing／extra／duplicateはfail、normalizationなし |
| `semantic` | `comparison.editor.semantic.tree-exact@1` | `comparator.editor.semantic.tree-exact@1` | `EditorSemanticSnapshotV1`のtree preorder、element identity、UTF-8 localized text、typed value、state、Action、relationをexact比較。normalizationなし |
| `layout` | `comparison.editor.layout.q16-exact@1` | `comparator.editor.layout.q16-exact@1` | Q16.16 luのmeasure／arrange bounds、baseline、clip、z-order、hit target、reflow、focus orderをexact比較。physical pixelとfloating epsilonを入力にしない |
| `visual` | `comparison.editor.visual.rgba8-srgb-1lsb@1` | `comparator.editor.visual.rgba8-srgb-bounded@1` | exact capture surface、extent、pixel format、linearization、per-channel／global／region threshold、contrast assertion、dynamic region setを下記規則で比較 |
| `uia_tree_event` | `comparison.editor.uia.control-view-exact@1` | `comparator.editor.uia.control-view-exact@1` | realized Control View tree、property／pattern、event sequenceをexact比較し、下記二Fieldだけを決定論的にnormalization |
| `command_trace` | `comparison.editor.command.ordered-exact@1` | `comparator.editor.command.ordered-exact@1` | actor、input、Intent、Command、argument、revision、result、ChangeSet、Receiptのordered canonical traceをexact比較。normalizationなし |
| `performance` | `comparison.editor.performance.absolute-and-regression@1` | `comparator.editor.performance.absolute-and-regression@1` | exact hardware／workloadごとに全absolute thresholdとrelative regression guardをANDし、Owner sample policyで集計 |

七Profileの`difference_schema_ref/hash`は全て`schema.editor-reference-difference@1`へ解決する。UIAだけが`normalization.editor.uia.ref01@1`、他六Profileは`normalization.editor.none@1`を使う。`comparator_parameter_ref/hash`はProfile表と同順に`parameter.editor.authoritative.byte-exact@1`、`parameter.editor.semantic.tree-exact@1`、`parameter.editor.layout.q16-exact@1`、`parameter.editor.visual.rgba8-srgb-1lsb@1`、`parameter.editor.uia.control-view-exact@1`、`parameter.editor.command.ordered-exact@1`、`parameter.editor.performance.absolute-and-regression@1`へ解決する。Visualの`qualification_ref/hash`は`qualification.editor.visual-repeatability@1`、Performanceはbaseline非依存の`qualification.editor.performance-harness@1`、他五Profileは両absentとする。published Baseline Registryを使う`qualification.editor.performance-reference@1`は通常runのPerformance resultであり、Comparison Profileのpreimageへ含めない。

Authoritative、Semantic、Layout、Commandの四Profileはそれぞれ対象schemaのcanonical serializer version／hashを持ち、比較器が再serializeしたexpected／observed canonical bytesを比較する。hash equalityだけでpassにせず、hash collisionまたはserializer mismatchは`infrastructure_error`とする。Semanticのlocalized stringはNFC等へ比較時変換せずproducerのcanonical UTF-8 bytesを使い、Layoutは全座標をQ16.16 luで保存して`±epsilon`を持たない。Command traceにhost timestamp、process ID、evidence保管pathを入れず、必要な時刻はdeterministic UI clock tick、順序は`sequence_index`として正本化する。

Visual Profileは次のexact値を持つ。

```text
capture_stage                         = editor_surface_pre_present
pixel_format                          = DXGI_FORMAT_R8G8B8A8_UNORM_SRGB
extent_policy                         = exact Environment Profile client extent
row_pitch_policy                      = repack tightly before hashing
rgb_decode                            = IEC 61966-2-1 sRGB -> linear Q0.16 LUT
rgb_decode_rounding                   = round_ties_to_even
encoded_rgb_channel_abs_lsb_max       = 1
alpha_channel_abs_lsb_max             = 0
global_mean_linear_rgb_error_q0_16_max = 64
semantic_region_mean_error_q0_16_max   = 128
critical_cue_mean_error_q0_16_max      = 64
pixel_over_channel_limit_count_max     = 0
missing_or_extra_pixel_count_max       = 0
contrast_text_normal_min_q8_8          = 1152  # 4.5:1
contrast_text_large_min_q8_8           = 768   # 3.0:1
contrast_nontext_and_focus_min_q8_8    = 768   # 3.0:1
```

RGBは保存8-bit値の差と、256-entryのversioned sRGB→linear Q0.16 LUTによるaggregate差の両方を検査する。これにより疎な1 LSB raster差は許すが、画面全体の1 LSB tint、2 LSB以上の一pixel、alpha差、extent差はfailになる。`critical_cue`はfocus、selection、error、warning、Runtime、lock、Proposal、staleのsemantic regionで、通常regionより厳しい。Contrast assertionはLight／Dark Design tokenまたはWindows system-color snapshotのlinear RGBから算出し、画像のanti-alias edge sampleをtoken値へ逆算しない。Light、Dark、四Contrast theme、A／B hardwareはEnvironment Profile別baselineを持つが、閾値は同一でありclosest baselineを選ばない。

```text
EditorVisualDynamicRegionV1
  dynamic_region_id
  dynamic_region_version
  reason_kind = caret_phase | os_owned_transient | recorded_external_uncertainty
  semantic_region_ref
  expected_bounds_artifact_ref
  expected_bounds_artifact_hash
  active_clock_tick_range
  maximum_area_px
  dynamic_region_content_hash

EditorVisualDynamicRegionSetV1
  dynamic_region_set_id
  dynamic_region_set_version
  entries[0..8]:
    dynamic_region_ref
    dynamic_region_hash
  dynamic_region_set_content_hash

EditorVisualRepeatabilityQualificationV1
  qualification_id
  qualification_version
  comparison_profile_ref
  comparison_profile_hash
  environment_profile_refs[13]
  environment_profile_hashes[13]
  fresh_process_count = 3
  captures_per_process = 10
  candidate_capture_refs[14]
  candidate_capture_hashes[14]
  comparison_result_refs[406]
  comparison_result_hashes[406]
  maximum_observed_difference_ref
  maximum_observed_difference_hash
  verification_receipt_ref
  verification_receipt_hash
  result = pass | fail | infrastructure_error
  qualification_content_hash
```

Dynamic regionはregistered semantic regionのobserved boundsとexpected boundsが一致した後だけ適用し、面積上限超過、範囲外pixel、unknown reason、Panel全体、重複regionを拒否する。mask後も差分pixel数と除外面積をEvidenceへ残す。Reference 01はvirtual clock、synthetic source、pre-present captureで全揺らぎを固定し、全166 entryで`dynamic-region-set.editor.empty@1`を使う。Profileの実行可能性は各Environment Profileについてfresh process三回、各十capture、計30capture、全14環境で420 captureを同一のinitialまたは当該環境required visual checkpointで採る`qualification.editor.visual-repeatability@1`で先に検証する。各環境のprocess 1／capture 1をqualification専用candidateとし、同環境の残り29 captureをcandidateへ比較するためresultはexact 406件である。candidateはrepeatability passだけではbaselineにならず、§15.8.6のBaseline Change／Review Receiptを別に要求する。全406 comparison resultがpassの`VerificationReceiptV1`へ閉じない、または一件でも上記thresholdを超えた場合はbaselineを発行せずCapabilityを`not_activated`に保つ。

UIA Profileは`normalization.editor.uia.ref01@1`だけを参照し、allowlistを`RuntimeId`とtrusted monotonic event timestampの二Fieldに閉じる。`RuntimeId`はdesktop内でだけuniqueかつ再利用可能なopaque値なので、expected Control View treeのpreorderでrootを0、子を1..Nのstable ordinalへ写し、event sourceも同じbijectionを使う。tree外から初出するsource、同ordinalへの二RuntimeId、同RuntimeIdへの二ordinalはfailである。timestamp値は比較対象から除くが、capture時の単調増加、event `sequence_index`、event kind、source ordinal、property／pattern、old／new value、correlation ID、件数と順序はexact比較する。AutomationIdはroot scope＋親＋sibling内identityとして比較し、全treeでuniqueとは仮定しない。Focus eventとSelection event、同値再通知をdeduplicateまたは相互変換しない。

Performance ProfileはPerformance Ownerの`sample.editor.ref01.five-by-120s@1`と`aggregate.editor.ref01.median-of-five-p95@1`を参照する。各hardwareでfresh process五run、各runはtyped barrier後120秒、warm-up sample除外、各metricのrun P95はnearest-rank、最終値は五P95を昇順にした三番目、欠測／NaN／counter overflow／run不足は`infrastructure_error`とする。A／Bを混ぜず、それぞれ全absolute thresholdを満たし、同じProfileのapproved baselineからlatency／memory peak／allocation countが5%超悪化しないことをANDする。10分soakは別のrequired resultで、短時間runのsampleへ混ぜない。exact workload、warm-up、thresholdは[Performance／Capacity §8](../04-runtime/performance-capacity.md#8-measurementregressionpromotion)が所有する。

全Profile、normalization profile、LUT、dynamic region set、contrast assertion、workload、warm-up、sample、threshold、aggregationのcontent hashが同じContract setで解決できるまでComparison Profile Registry rowをpublishしない。Profile missing時の`exact`、画像toolの既定threshold、UIA clientの自動event coalescing、Performance harnessの平均値へfallbackしない。

#### 15.8.6 Baseline change、Evidence Bundle、署名

```text
EditorReferenceBaselineRegistryV1
  registry_id
  format_major = 1
  revision
  fixture_id
  fixture_version
  coverage_matrix_ref
  coverage_matrix_hash
  comparison_profile_registry_ref
  comparison_profile_registry_hash
  entries[1..4096]:
    coverage_entry_hash
    oracle_ref
    oracle_hash
    oracle_kind
    expected_oracle_subject_ref
    expected_oracle_subject_hash
    comparison_profile_ref
    comparison_profile_hash
  registry_content_hash

EditorReferenceBaselineHeadV1
  baseline_head_id
  head_sequence
  fixture_id
  fixture_version
  previous_baseline_head_ref optional
  previous_baseline_head_hash optional
  baseline_registry_ref
  baseline_registry_hash
  fixture_manifest_ref
  fixture_manifest_hash
  applied_change_item_set_hash
  baseline_head_content_hash

EditorReferenceInitialBaselineExecutionDefinitionV1
  execution_definition_id
  execution_definition_version
  fixture_id
  fixture_version
  contract_set_hash
  fixture_source_ref
  fixture_source_hash
  workspace_ref
  workspace_hash
  panel_catalog_hash
  widget_pattern_registry_hash
  comparison_profile_registry_ref
  comparison_profile_registry_hash
  environment_profile_refs[1..64]
  environment_profile_hashes[1..64]
  scenario_refs[1..256]
  scenario_hashes[1..256]
  coverage_matrix_ref
  coverage_matrix_hash
  verification_gate_policy_ref
  verification_gate_policy_hash
  execution_definition_content_hash

EditorReferenceInitialBaselineObservationBundleV1
  observation_bundle_id
  execution_definition_ref
  execution_definition_hash
  candidate_root_hash
  contract_set_hash
  toolchain_lock_hash
  observations[1..4096]:
    coverage_entry_hash
    observed_artifact_ref
    observed_artifact_hash
    capture_result_ref
    capture_result_hash
    result = captured | infrastructure_error | cancelled
  missing_coverage_entries[0..4096]
  extra_coverage_entries[0..4096]
  diagnostic_refs[0..1024]
  result = captured | infrastructure_error | cancelled
  evidence_root_hash

EditorReferenceInitialBaselineDefinitionClosureV1
  definition_closure_id
  definition_closure_version
  execution_definition_ref
  execution_definition_hash
  observation_bundle_ref
  observation_bundle_hash
  expected_subjects[1..4096]:
    coverage_entry_hash
    oracle_ref
    oracle_hash
    proposed_expected_subject_ref
    proposed_expected_subject_hash
  prerequisite_qualification_receipt_refs[1..512]
  prerequisite_qualification_receipt_hashes[1..512]
  definition_closure_content_hash

EditorReferenceBaselineReasonEvidenceV1
  reason_evidence_id
  reason_evidence_version
  reason_kind = approved_initial_baseline
              | approved_design_contract_change
              | validated_bug_fix
  initial_definition_closure_ref optional
  initial_definition_closure_hash optional
  approved_design_contract_change_subject_ref optional
  approved_design_contract_change_subject_hash optional
  validated_bug_report_ref optional
  validated_bug_report_hash optional
  fix_qualification_receipt_refs[0..128]
  fix_qualification_receipt_hashes[0..128]
  reason_evidence_content_hash

EditorReferenceBaselineChangeItemSubjectV1
  change_item_id
  change_item_version
  change_mode = initialize | replace
  reason_kind = approved_initial_baseline
              | approved_design_contract_change
              | validated_bug_fix
  reason_evidence_ref
  reason_evidence_hash
  requester_subject_ref
  created_at
  fixture_id
  fixture_version
  source_baseline_head_ref optional
  source_baseline_head_hash optional
  source_baseline_registry_ref optional
  source_baseline_registry_hash optional
  coverage_entry_hash
  oracle_ref
  oracle_hash
  oracle_kind
  scenario_ref
  checkpoint_ref
  driver_binding_ref
  environment_profile_ref
  old_expected_subject_ref optional
  old_expected_subject_hash optional
  incoming_observed_artifact_ref
  incoming_observed_artifact_hash
  proposed_expected_subject_ref
  proposed_expected_subject_hash
  old_comparison_profile_ref optional
  old_comparison_profile_hash optional
  proposed_comparison_profile_ref
  proposed_comparison_profile_hash
  difference_record_refs[1..128]
  difference_record_hashes[1..128]
  changed_semantic_region_refs[0..64]
  changed_semantic_region_hashes[0..64]
  impacted_coverage_entry_hashes[1..4096]
  guard_policy_ref
  guard_policy_hash
  guard_comparison_result_refs[1..4096]
  guard_comparison_result_hashes[1..4096]
  change_item_subject_hash

EditorReferenceBaselineChangeSubjectV1
  baseline_change_id
  baseline_change_version
  fixture_id
  fixture_version
  source_baseline_head_ref optional
  source_baseline_head_hash optional
  source_baseline_registry_ref optional
  source_baseline_registry_hash optional
  change_item_subject_refs[1..256]
  change_item_subject_hashes[1..256]
  change_item_set_hash
  affected_coverage_entry_hashes[1..4096]
  review_policy_ref
  review_policy_hash
  requester_subject_ref
  created_at
  baseline_change_subject_hash

EditorReferenceBaselineReviewNoteV1
  review_note_id
  review_note_version
  change_item_subject_ref
  change_item_subject_hash
  decision = approved | rejected | changes_requested
  rationale_kind = initial_fixture_matches | contract_matches | validated_fix_matches
                 | evidence_incomplete | guard_failed
                 | unauthorized_relaxation | wrong_scope | other
  reviewed_difference_record_refs[1..128]
  reviewed_difference_record_hashes[1..128]
  reviewed_requirement_refs[1..64]
  reviewed_requirement_hashes[1..64]
  unresolved_issue_refs[0..64]
  unresolved_issue_hashes[0..64]
  bounded_comment_artifact_ref
  bounded_comment_artifact_hash
  review_note_content_hash

EditorReferenceBaselinePublicationV1
  publication_id
  publication_version
  baseline_change_subject_ref
  baseline_change_subject_hash
  source_baseline_head_ref optional
  source_baseline_head_hash optional
  source_baseline_registry_ref optional
  source_baseline_registry_hash optional
  approved_change_item_subject_refs[1..256]
  approved_change_item_subject_hashes[1..256]
  review_receipt_refs[2..512]
  review_receipt_hashes[2..512]
  candidate_baseline_registry_ref
  candidate_baseline_registry_hash
  candidate_manifest_ref
  candidate_manifest_hash
  candidate_baseline_head_ref
  candidate_baseline_head_hash
  candidate_verification_receipt_ref optional
  candidate_verification_receipt_hash optional
  promotion_receipt_ref optional
  promotion_receipt_hash optional
  superseded_change_item_refs[0..255]
  superseded_change_item_hashes[0..255]
  result = published | rolled_back | failed_before_publish | infrastructure_error
  publication_content_hash

EditorReferenceBaselineChangeRegistryV1
  registry_id = registry.editor-reference.baseline-change
  format_major = 1
  revision
  publication_refs[0..4096]
  publication_hashes[0..4096]
  registry_content_hash

EditorReferenceEvidenceBundleV1
  evidence_bundle_id
  fixture_manifest_ref
  fixture_manifest_content_hash
  fixture_registry_content_hash
  candidate_root_hash
  contract_set_hash
  toolchain_lock_hash
  capability_snapshot_ref
  capability_snapshot_hash
  observed_environment_profile_ref
  observed_environment_content_hash
  scenario_results[1..256]:
    scenario_ref
    coverage_results[1..4096]:
      coverage_entry_hash
      observed_artifact_ref optional
      observed_artifact_hash optional
      comparison_result_ref
      comparison_result_hash
      result = pass | fail | infrastructure_error | cancelled
    result = pass | fail | infrastructure_error | cancelled
  missing_coverage_entries[0..4096]
  extra_coverage_entries[0..4096]
  diagnostic_refs[0..1024]
  result = pass | fail | infrastructure_error | cancelled
  evidence_root_hash
```

`EditorReferenceBaselineRegistryV1.entries[]`はCoverage Matrixの全entryと`coverage_entry_hash`でset equalityにし、Oracle ref/hash、kind、expected subject ref/hash、Comparison Profile ref/hashを各Coverage entryのOracleとbyte equalityにする。Reference 01の完成Registry `registry.editor-reference.baseline.ref01@1`はexact 166 entryを持つ。Manifestの`baseline_registry_ref/hash`はこのRegistryを参照するが、Baseline RegistryからManifestへ戻るrefを持たせずhash cycleを作らない。初回Registryが未承認の間はManifestとFixture Registry rowを発行しない。

Publication Serviceのauthoritative current pointerは`EditorReferenceBaselineHeadV1`一件である。初回はprevious pair absent、`head_sequence=1`、replaceはcurrent Head pair requiredかつsequence exact `N+1`とする。HeadのRegistryとManifestは相互のfixture／version／Baseline Registry bindingが一致し、`applied_change_item_set_hash`は当該publish対象Item集合と一致させる。HeadはPublication／Promotion Receiptをback-referenceせず、candidateとして決定論的にhashできる。CASはexpected previous Head ref/hashからcandidate Head ref/hashへの一回だけで、Promotion Receiptのbefore／after tree hashとread-back hashはこのHead transactionのauthoritative store rootへ一致させる。

初回captureは`EditorReferenceInitialBaselineExecutionDefinitionV1`だけをRunner inputにし、Baseline Registry、Manifest、expected subjectを一件も含めない。Environment／scenarioのref/hash配列は同じ長さで1対1、Coverage Matrixが参照するexact setと一致させる。Runnerは166 tupleを実行して`EditorReferenceInitialBaselineObservationBundleV1`を生成するが、比較対象が未承認なので`captured`を`pass`と呼ばず、通常の`EditorReferenceEvidenceBundleV1`または`VerificationReceiptV1`へ読み替えない。observationsはCoverage Matrixとset equality、全件`captured`、missing／extra emptyの場合だけ次へ進める。

`EditorReferenceInitialBaselineDefinitionClosureV1`はExecution Definition、Observation Bundle、そこからContract ownerがcanonicalizeした166 proposed expected subject、prerequisite qualificationを一方向に閉じるpre-publication artifactであり、Baseline Registry／Manifestを参照しない。`expected_subjects[]`はCoverage MatrixおよびObservation Bundleのexact 166 entryと`coverage_entry_hash`でset equality、Oracle ref/hashとproposed expected subjectは各entryのtype／scenario／checkpoint／driver／Environmentへ一致させる。prerequisite qualificationはVisual repeatability、Performance A／B×三workloadのabsolute threshold＋repeatability、schema／serializer、fixture source determinism、required nonvisual validationのfresh Receipt exact集合であり、初回expected subject自体の正しさを自動承認するReceiptではない。このcapture→closure→Item review→candidate全件検証→Publicationの一方向構造により、current output由来の自動passまたは未承認Baseline RegistryをManifestへ仮挿入するbootstrap cycleを作らない。

`EditorReferenceBaselineReasonEvidenceV1`のoptional Fieldは`reason_kind`でpresenceを閉じる。`approved_initial_baseline`はinitial definition closure pairだけをrequired、他Fieldとfix Receiptをabsent／emptyにする。`approved_design_contract_change`はapproved Design Contract change subject pairだけをrequiredにする。`validated_bug_fix`はvalidated bug report pairと一件以上のfix qualification Receipt pairをrequiredにする。二配列は同じ長さで1対1、missing／extra reason Field、unapproved contract draft、issue title、commit message、AI説明だけの根拠を拒否する。

Change Itemは一つの`coverage_entry_hash`と一つのoracle expected subjectだけを置換するatomic decision unitである。`change_mode=initialize`はsource Baseline Head、source Baseline Registry、old expected subject、old Comparison Profileを全てabsent、`reason_kind=approved_initial_baseline`だけを許し、reason evidenceのinitial definition closureにある同entry／proposed subjectとbyte equalityにする。`replace`はsource Head、source Registry、old subject、old Profileの四ref/hash pairを全てrequiredにし、Head内のRegistry pairとItemのsource Registry pairをbyte equalityにする。reasonは`approved_design_contract_change | validated_bug_fix`のexact一方とする。

Incoming artifactはRunnerが観測した事実、proposed expected subjectは採用候補であり、同じrefとして省略しない。old／proposed expected subjectのtype、Oracle kind、checkpoint、driver、Environment ProfileはCoverage entryと一致させる。`difference_record_refs/hashes`と`changed_semantic_region_refs/hashes`は各組で同じ長さの1対1、`impacted_coverage_entry_hashes`は同expected subject、Comparison Profile、LUT、normalization、dynamic region、thresholdから影響を受けるCoverage entryのexact transitive setとする。Guardは同じscenario／driver／environmentの変更対象外oracleと、影響set全件のrequired nonchanged oracleを含み、全Comparison Resultがpassでなければapproval-readyにしない。

initialize ItemのGround Truthは不在であり、画面やDifference artifactへzero image／empty treeを捏造しない。Difference Recordは`difference_kind=extra`、expected value pair absent、observed value pair=incoming artifactとして初回候補の存在を表す。replace Itemはold expectedをGround Truthとして通常比較する。どちらもproposed expected subjectをIncoming表示そのものへ暗黙aliasせず、canonical subject artifactとして別に解決する。

Expected subject内容だけの置換は一Change Itemにできる。Comparison Profile、comparator、LUT、normalization、dynamic region policy、Performance threshold／aggregationの変更は共有Contract変更なので、参照する全Coverage entryを`impacted_coverage_entry_hashes`へ展開し、新Profile version／hash、repeatabilityまたはPerformance qualification、全guard resultを要求する。tolerance／mask／thresholdだけを緩和するItem、closest baseline追加、Profile versionを変えないbytes変更はpre-review validationで`unauthorized_relaxation`として拒否する。

`EditorReferenceBaselineChangeSubjectV1`はReview Batchのimmutable subjectであり、Change Itemを`change_item_id`のUTF-8 byte順、duplicateなしで束ねる。全Itemは同じfixture、source Baseline Head、source Baseline Registryを持ち、initialize／replaceを混在させない。Batchのsource pairはItem集合とbyte equalityにする。`change_item_set_hash`はItem ref/hash集合、`affected_coverage_entry_hashes`はItem impact集合のunionとset equalityにする。画面上のselection、focus、`viewed`、pending decision、comment draftをBatch bytesへ入れず、Batch内容が一byte変われば別subject/hashにする。

Baseline reviewのReview Receiptは[AI Verification／Provenance §7.4](../01-governance/ai-verification-provenance.md#74-reviewreceiptv1)の既存`ReviewReceiptV1`だけを使い、新しい署名形式を作らない。`subject_kind=editor_reference_baseline_change_item`、`subject_sha256=change_item_subject_hash`、`requirement_coverage_hash`は`policy.editor-reference.baseline-review@1`のexact checklist hash、`verification_receipt_hashes[]`はItemのguard／qualification Receipt exact集合、`comment_ref`は同Item／decisionの完成`EditorReferenceBaselineReviewNoteV1`とする。approved scopeは次で閉じる。

Review NoteのDifference、Requirement、unresolved issueの各ref/hash配列も同じ長さで1対1、Requirement集合はreview policy checklistのrequired exact setとする。decision=`approved`ではDifference集合をItemのdifference exact set、unresolved issueをemptyにする。`rejected | changes_requested`ではDifference集合をItemのnon-empty exact subsetにできるが、少なくとも一つのunresolved issueまたはbounded comment内のtyped reasonを必要とし、自由文だけで`wrong_scope`や`guard_failed`を捏造しない。

```text
Operation = {operation.editor-reference.baseline.publish@1}
Path      = {}
Target    = {fixture_id, source_baseline_head_ref/hash or genesis,
             source_baseline_registry_ref/hash or genesis,
             change_item_subject_ref/hash, coverage_entry_hash, oracle_ref/hash,
             proposed_expected_subject_ref/hash}
Risk      = {risk.editor-reference.baseline-change@1, class=R4}
expires_at = min(issued_at + 24 hours, Authorization expires_at)
```

Approved Item一件につき、対象oracleの`role.domain.owner`一件と`role.review.independent`一件のapproved `ReviewReceiptV1`を要求し、二subjectはrequester、candidate generator、AI／Worker、互いからdistinctにする。Performance Itemのdomain ownerはPerformance、Semantic／layout／visual／UIA／Commandは各Contract owner、authoritative stateは正本ownerである。`rejected | changes_requested`は一件でもpublicationを許さず、新しいevidenceまたはproposed subjectは新Item hashとして再reviewする。期限切れ、revocation、Role／subject independence不一致、Item／source headの一byte変更でapproved Receiptを失効させる。

Review画面の`Approve | Reject | Request changes`はまずWorkspace-local pending decisionをItemごとに設定するだけで、ReceiptやRegistryを変更しない。`Submit review`はUser presenceと再認証後、pending Itemごとに別`ReviewReceiptV1`を発行する。一回のceremonyで複数pending decisionを送れても、Itemを未閲覧のまま一括approvedへ設定する`Accept all`、Batch hash一件への包括署名、一Receiptで複数Itemを承認することを許さない。AIはread／bounded explanationだけで、pending decision、Review Note、Receipt、publish requestを生成しない。

Publication Serviceはinitializeでは166 Item全件、replaceではUserがpublish対象として明示選択したapproved Item集合について、各二Receipt、fresh guard、current revocation、source Baseline Head／Registryを再検査する。承認Itemだけをsource Registryへ適用してcandidate Baseline Registry、それを参照する新Manifest version、candidate Baseline Headを決定論的にcompileし、全166 Coverage entryをcandidate baselineで再実行する。全required oracleがpassした`VerificationReceiptV1`後にだけ`operation.editor-reference.baseline.publish@1`をexpected source Headからcandidate HeadへCAS実行し、既存`PromotionReceiptV1.result=committed`のbefore／after／read-back hash一致を要求する。

Publicationのoptional Receipt pairはpartial pairを拒否し、superseded ref/hash配列も同じ長さで1対1にする。`published`はcandidate VerificationとPromotionの両pair required、Verification result pass、Promotion result committed、superseded集合は同Batchの非publish Item exact集合とする。`failed_before_publish`はPromotion pair absent、candidate Verification pairは検証を開始してReceiptが完成した場合だけpresentにする。`rolled_back | infrastructure_error`はPromotionを開始した場合にPromotion pair required、candidate VerificationはPromotion開始前にpassしていなければならない。失敗recordからcurrent Headを変更またはChange Registryへappendしない。

replaceの部分publishは許すが、同じBatchの非publish Itemはsource Head変更により`superseded`となり、自動rebaseしない。initializeは部分publishを禁止する。成功Publicationだけを`registry.editor-reference.baseline-change@1`の次revisionへappendし、current Baseline Registry／Manifestは最大revision、file mtime、Change Registry末尾から推測せずPublication Serviceのcurrent `EditorReferenceBaselineHeadV1`をread-backして解決する。CAS conflict、candidate再検証fail、Receipt失効ではnew Head／Registry／Manifestをpublishせずold Headを維持する。

全`coverage_results`長の合計はCoverage Matrix entry数と一致し、`coverage_entry_hash`集合はMatrixとset equalityにする。各comparison result ref／hashは完成artifactへbyte equality、Bundle内`result`は同Comparison Resultの`result`とbyte equalityにする。observed artifact ref／hashのpresenceはComparison Resultと一致させ、presentなpairは完成artifactへbyte equality、absentは`infrastructure_error | cancelled`で観測artifactが生成されなかった場合だけ許す。Bundleはrequired coverage全件がpass、missing／extraが空、Manifest／Registry／Candidate／Contract／Toolchain／Capability／Environmentがexact一致するときだけpassにする。一oracleのpassを別oracleへ流用せず、Visual差が0でもSemantic／UIA／Commandがmissingならfail、Runner／Device／provider異常は`infrastructure_error`であってpassまたはproduct defectにしない。Evidenceはsynthetic fixture dataだけを含み、credential、clipboard、User Project、個人path、未許可Sourceをcaptureしない。

新しい`EditorReferenceReceiptV1`は作らない。信頼済みRunnerは完成Bundleと各artifactを[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)の既存`VerificationReceiptV1.output_artifacts[]`へcontent hash付きで束縛する。Phase／Capability evidenceへ使う場合だけ、その完成Verification Receiptを既存`TechnicalQualificationReceiptV1`が一方向参照する。Manifest、Bundle、Verification Receipt、Technical Qualification、Approvalは別stageであり、相互に埋め戻し、署名代用、pass推測をしない。

Visual／Semantic／layout／UIA／Command／Performance baselineの初期作成と変更は上記Change Item、二Review Receipt、candidate全Coverage verification、CAS Publicationを全て要求する。failing current outputの`Accept all`、closest baseline追加、tolerance／mask／thresholdだけの緩和、AIによる自動承認を禁止する。AIはBundle差分の説明とChange Proposalを作成できるが、pending decision、baseline、comparison profile、result、Review Receipt、Publication、署名を変更しない。§10.3のUser-authorized `VisualEvidenceSnapshotV1`はvisual review入力であってexpected baselineまたはGate pass artifactではなく、Reference Runnerのexact Environment Profile付きcaptureを代用しない。BundleまたはdiffをAIへ渡す場合は既定Contextへ自動添付せず、Userが選んだbounded、synthetic、redacted sliceだけを別Context artifactにする。

#### 15.8.7 Required conformance scenarios

| scenario ID | 必須内容 |
|---|---|
| `scenario.reference-fixture.registry-set-equality@1` | Registry／Manifestのmissing、extra、duplicate、wrong owner／version／hash、Capability applicability driftを各一原因で拒否 |
| `scenario.reference-fixture.source-reset@1` | fixed seed、immutable fixture source、scenario間reloadによりProject／Workspace／selection／cache／Session流用0件 |
| `scenario.reference-fixture.driver-convergence@1` | human、pointer、keyboard、UIA、AI contract driverが同じlogical intent、Stable target、checkpointへ収束し、座標／label由来target 0件 |
| `scenario.reference-fixture.barrier-clock@1` | typed generation／terminal barrier、virtual clock、timeout infrastructure errorを検証し、sleep pass、frame-count identity、無制限wait 0件 |
| `scenario.reference-fixture.coverage-set-equality@1` | required tupleのmissing／extra、wrong driver／environment／oracle、自然文pairwise補完を拒否 |
| `scenario.reference-fixture.oracle-independence@1` | state／semantic／layout／visual／UIA／command／performanceを一つずつ壊し、他planeのpassで補完0件 |
| `scenario.reference-fixture.comparison-registry@1` | 七Profileのoracle kind／ref／hash／comparator contract、Profile missing、wrong kind、same-version drift、Runner既定値fallback、Profile↔Qualification hash cycleを各一原因で拒否 |
| `scenario.reference-fixture.visual-profile@1` | exact environment baseline、RGBA8 sRGB／linear LUT、1 LSB／aggregate threshold、semantic／critical region、High Contrast、二hardware、30-capture repeatabilityを検証し、closest baseline／auto tolerance／free mask 0件 |
| `scenario.reference-fixture.uia-event-trace@1` | provider Control View tree／property／pattern／event order、RuntimeId bijection、timestamp除外、stale providerを検証し、二Field以外のnormalization、event deduplicate／coalescing 0件 |
| `scenario.reference-fixture.performance-profile@1` | baseline非依存Harness Qualification、A／B別の五run×120秒、nearest-rank P95、median、absolute threshold、Publication後の5% regression、10分soakを検証し、未承認Baselineに対するrelative pass、hardware混合、平均、欠測0補完を0件 |
| `scenario.reference-fixture.baseline-change@1` | initial Execution Definition→166 capture→Definition Closure→initialize Itemとreplace Item、before／incoming／proposed／typed diff、initial Ground Truth不在、impact set、guard、pending decisionとReceipt分離、owner＋independent Reviewを検証し、capture=`pass`、zero Ground Truth、failing output／AI decision／Accept all 0件 |
| `scenario.reference-fixture.baseline-review-surface@1` | Ground Truth／Incoming／Differenceの同時表示、typed Difference list、同期pan／zoom、Item selection／keyboard focus／viewed／pending decision／signed decision／staleの直交状態、High Contrast／200% Font／`editor_ui_scale=2.00`でのreflow、AI read-onlyを検証し、画像click承認、色だけの差分、非表示pane、Batch一括decisionを0件 |
| `scenario.reference-fixture.baseline-publication@1` | initial 166 Item全件、replace部分publish、candidate全166再検証、`EditorReferenceBaselineHeadV1` sequence／source-head CAS、Promotion before／after／read-back、Change Registry append、nonselected supersedeを検証し、stale Receipt／partial initialize／latest推測／Head↔Publication hash cycle 0件 |
| `scenario.reference-fixture.evidence-receipt@1` | Bundle、Verification Receipt、Technical Qualificationの一方向binding、wrong Candidate／Contract／Toolchain／Environment、missing artifact、wrong purpose／署名を拒否 |
| `scenario.reference-fixture.privacy@1` | synthetic data、redaction、artifact path normalizationを検証し、credential／clipboard／User Project／個人path capture 0件 |
