# Miraikanai Engine Post Process／AI Authoringアーキテクチャ規約

- 文書版: 1.0
- 作成日: 2026-07-20
- 対象: HDR／SDR Post Process、Exposure、Tone Mapping、Color、Bloom、DOF、Motion Blur、SSAO／SSR、Volume、AI Operation、History、Budget、Qualification
- 状態: プロジェクト公式の規範設計レビュー版
- 機能範囲: [Miraikanai Engine 2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)
- Renderer／AA実行正本: [Miraikanai Engine Rendering／Render Graphアーキテクチャ規約](./2026-07-19-rendering-render-graph-architecture-design.md)
- Material規約: [Miraikanai Engine Material／Visual Style／AI Authoringアーキテクチャ規約](./2026-07-20-material-visual-style-ai-authoring-architecture-design.md)
- Lighting規約: [Miraikanai Engine Lighting／AI Authoringアーキテクチャ規約](./2026-07-20-lighting-ai-authoring-architecture-design.md)
- Camera規約: [Miraikanai Engine Camera Platform／AI Authoring／Virtual Productionアーキテクチャ規約](./2026-07-20-camera-platform-ai-authoring-virtual-production-architecture-design.md)
- Environment規約: [Miraikanai Engine Environment Platform／AI Authoringアーキテクチャ規約](./2026-07-20-environment-platform-ai-authoring-architecture-design.md)
- UI規約: [Miraikanai Engine UI／Text／Localization／Accessibility規約](./2026-07-19-ui-text-localization-accessibility-design.md)
- 契約規約: [Miraikanai Engine 実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)
- Runtime規約: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- AI権限規約: [Miraikanai Engine AI実装・保守ガバナンス規約](./2026-07-19-ai-engine-development-governance-design.md)
- AI検証規約: [Miraikanai Engine AI検証・評価・来歴規約](./2026-07-19-ai-verification-evaluation-provenance-design.md)

## 0. 30秒で分かる公式方式

Miraikanai EngineのPost Processは、任意Shaderを順番に積むraw graphではない。人間またはAIの見た目要求を`PostProcessIntentV1`、保存可能な設定を`PostProcessProfileV1`、実行可能nodeを`PostProcessNodeCatalogV1`として管理し、決定論的`PostProcessIntentResolverV1`がView Familyごとの`ResolvedPostProcessPlanV1`へ解決する。

```text
Human／AI visual goal
  -> PostProcessIntentV1
  -> validate + resolve
  -> ResolvedPostProcessPlanV1
  -> approved Profile ChangeSet
  -> View／Volume blend
  -> Renderer-owned Render Graph expansion
  -> history／resource／pass execution
  -> display output + telemetry
```

AIは「映画的」「画面酔いを抑える」「pixel artを鮮明に保つ」「低価格端末で色調を維持する」を指定できる。AIへRender pass順、native texture、history weight、sample count、thread group、GPU handle、任意Shader sourceを直接編集させない。

## 1. 設計目標

1. Effect名だけでなく、入力色空間、実行stage、履歴、AA、UI、Target制約を契約化する。
2. Anti-alias、Exposure、Bloom、Tone Mapping、Color、Lens effectの順序をEngine-ownedにする。
3. Source Intent、保存Profile、解決済みPlan、Runtime historyを分離する。
4. View Family、Camera Profile、Volumeの合成を決定論的にする。
5. Pixel Art、UI／Text、Accessibility、HDR出力を意図せず破壊しない。
6. AIがbounded Operationで変更範囲、cost、fallback、visual差分を理解できる。
7. D3D12、Vulkan、Metalで同じ意味Planを実行し、backend固有差をAdapterに閉じる。
8. C1のPortable Postを基準とし、SSAO／SSR、高品質DOF等をCapabilityで段階導入する。

## 2. 責務と非責務

| 主題 | 正本／Owner |
|---|---|
| Post Process Intent、Profile、Node Catalog、Resolver、Plan | 本規約 |
| 非AA Post effectのstage、依存、history、Volume blend | 本規約 |
| Render Graph展開、resource、barrier、queue、native format | Renderer規約 |
| AA Intentの意味、互換表、成熟度 | 2D／3D機能計画 |
| AA sample／resolve、Temporal history実行 | Renderer規約 |
| Lens physical property、Camera cut、View Family | Camera規約 |
| BRDF、Material tone response、Visual Style | Material規約 |
| Fog、Cloud、Sky、Environment Lighting | Environment規約 |
| UI／Text／pixel-locked layer、Accessibility semantics | UI規約 |
| Post shader asset import、LUT cook、Project Technique | Asset／Asset Import規約 |

`PostProcessProfileV1`はExposure、Tone Mapping、Color、Bloom、Lens effectの保存正本である。Camera ProfileはPost Process ProfileのStable IDと型付きoverrideを参照し、同じfieldを別形式で重複保存しない。

EnvironmentのFog／CloudをPost Process nodeとして再定義しない。SSAO／SSRは画面空間効果だがOpaqueのdepth／normalを必要とするため、本規約が意味と品質を所有し、RendererがWorld composition内の正しいstageへ展開する。

## 3. 正本データモデル

### 3.1 `PostProcessIntentV1`

```text
PostProcessIntentV1
  intent_id
  scope
  goal
  target_selector
  quality_selector
  exposure_intent
  tone_intent
  color_intent
  bloom_intent
  focus_intent
  motion_clarity_intent
  ambient_occlusion_intent
  reflection_intent
  composition_constraints
  accessibility_constraints
  fallback_priority[]
  base_revision
```

閉じた主要語彙は次とする。

- `scope`: `project_default`、`view_family`、`camera_profile`、`volume`
- `goal`: `balanced`、`cinematic`、`gameplay_clarity`、`low_gpu_cost`、`low_latency`、`pixel_crisp`、`accessibility_safe`、`offline_reference`
- `exposure_intent`: `manual`、`stable_auto`、`responsive_auto`、`match_reference`
- `tone_intent`: `neutral`、`filmic`、`high_contrast`、`low_contrast`、`pixel_preserve`、`custom_profile`
- `bloom_intent`: `off`、`subtle`、`balanced`、`strong`
- `focus_intent`: `off`、`camera_lens`、`distance`、`subject_group`
- `motion_clarity_intent`: `crisp`、`balanced`、`cinematic`
- `ambient_occlusion_intent`: `off`、`subtle`、`balanced`、`strong`
- `reflection_intent`: `off`、`rough_only`、`balanced`、`high_quality`

Intentは「何を達成するか」を表し、kernel radius、tap count、history blend、mip数、Render pass、Shader permutationを持たない。

### 3.2 `PostProcessProfileV1`

Profileは型付きexact parameterを保存する。

```text
PostProcessProfileV1
  profile_id
  version
  parent_profile_id
  node_settings[]
  output_color_policy
  layer_policy
  target_overrides[]
  human_lock_mask
  qualification_receipt_refs[]
```

Profile継承は最大4段とし、循環を拒否する。各nodeは`inherit`、`disabled`、`override`を明示する。配列の暗黙append、自由なnode追加、同じNode IDの重複を禁止する。

C1はProfile当たり最大32 `node_settings`、View Family当たり最大32 active Nodeとする。Target Profileは上限を下げられるが増やせない。上限超過を末尾切捨てせず、Resolverがfallbackまたはtyped Diagnosticを返す。

Camera固有差分は`PostProcessCameraOverrideV1`で表す。これは`base_profile_id`、`base_profile_revision`、field mask、最大16件のpartial `node_settings`だけを持ち、Profile継承、Node追加、stage変更、Target Capability追加はできない。Camera Documentはこの型を参照し、ExposureやTone MappingをCamera独自fieldへ複製しない。

### 3.3 `PostProcessVolumeV1`

World内の局所変更は次で表す。

```text
PostProcessVolumeV1
  volume_id
  shape_ref
  profile_id
  priority_i16
  blend_radius_m
  blend_weight
  unbounded
  enabled
  target_selector
```

Viewごとに交差するVolumeは最大32をC1上限とし、`priority -> volume_id`で安定sortする。Weightはshape距離から0～1へ決定する。Blend可能fieldだけを線形または定義済みdomain blendし、enum、asset参照、Node enable等の非Blend fieldは最高priorityかつStable ID最小の一件を選ぶ。競合は`MIRAKAN-POST-VOLUME-CONFLICT`で説明する。

### 3.4 `PostProcessNodeCatalogV1`

Node Catalogは各Nodeへ次を宣言する。

- version付きNode IDとCapability maturity
- 入力／出力のlogical resourceと色空間
- 固定execution stage
- required Camera／Renderer input
- temporal historyの有無、format、reset reason
- blend可能parameterと範囲
- 対応Target、HDR／SDR、AA mode、layer
- 予測cost model、persistent／transient byte式
- fallback nodeまたはdisable policy
- Visual fixture、conformance test、Qualification Receipt

通常ProfileはCatalogにないNodeを作れない。C3のProject Post Techniqueは承認済みShader asset、固定MCD descriptor、offline compile、security／performance reviewを通し、通常AI Operationからは作成しない。

### 3.5 `ResolvedPostProcessPlanV1`

Planは最低限、次を持つ。

- View Family、Camera、Profile、Volume、Target、AA Planのversion／hash
- stage順に並んだversion付きNode ID
- 全nodeの解決済みexact parameter
- 入力／出力logical resource、色空間、resolution class
- history key、generation、reset条件
- Layer inclusion／exclusion
- 予測CPU／GPU時間、persistent／transient byte
- 採用、棄却、fallback、非互換理由
- Preview、承認、Qualification要否
- `plan_hash`と有効期限

PlanはRender Graphのnative pass listではない。RendererはCatalog compilerを通じてPlanを検証済みpass templateへ展開する。

## 4. 固定Execution Stage

実行stageは次の順で固定し、ProfileやAIが並べ替えない。

| Stage | 内容 | 主入力 |
|---|---|---|
| `P00_OPAQUE_AO` | SSAOをOpaque Lightingへ供給 | depth、normal |
| `P05_OPAQUE_REFLECTION` | SSRをlit Opaqueへ合成 | HDR opaque、depth、normal、motion |
| `P10_WORLD_COMPOSITE` | Transparent、VFX、Water、EnvironmentとのRenderer合成点 | World HDR |
| `P15_EXPOSURE_MEASURE` | luminance histogram／manual exposure | World HDR |
| `P20_TEMPORAL_RESOLVE` | TAA／TAAU／Provider。AA Planが所有 | HDR、depth、motion、history |
| `P30_LENS` | Motion Blur、Depth of Field | resolved HDR、depth、motion、Camera Lens |
| `P40_BLOOM` | Bloom抽出、downsample、upsample、合成 | lens後HDR、exposure |
| `P50_TONE_COLOR` | White Balance、Tone Mapping、Color Grade | HDR、exposure、LUT |
| `P60_DISPLAY_AA` | FXAA／SMAA等のspatial AA。AA Planが所有 | display-linear color |
| `P70_DISPLAY_STYLE` | Vignette等の許可済みdisplay effect | display-linear color |
| `P80_PIXEL_LOCKED_UI` | UI／Text／cursor／pixel-locked layer合成点 | display target |
| `P90_ACCESSIBILITY_OUTPUT` | UI規約が要求する最終display transform／overlay | composited display |

SSAOはOpaque Lightingのambient visibility入力として消費し、SSRはlit Opaqueへreflection contributionとして合成してから`P10`へ進む。表の順番を「全effectが一本のfull-screen chainである」と解釈しない。Rendererは同一stage内のデータ依存からpassを展開するが、Catalogにないstage移動はできない。

同一stage内の非可換NodeはCatalogに明示dependency edgeを必須とする。依存のないNodeはCatalogが`commutative=true`を宣言した場合だけNode ID順で実行できる。曖昧な順序とcycleはGraph compile前に`MIRAKAN-POST-STAGE-INVALID`で拒否する。

`P20`と`P60`は`ResolvedAntiAliasingPlanV1`が選択した一方または許可された組合せだけを実行する。Post Process ProfileがAA methodやsample countを独自指定しない。

## 5. C1 NodeとParameter

### 5.1 Exposure

`ExposureProfileV1`は次を持つ。

- mode: `manual_ev100`／`histogram_auto`
- manual EV100: -16～32
- histogram: 256 bins
- luminance percentile: low 0.5%、high 99.5%
- middle gray: 0.18
- output EV100 clamp: -6～16
- adaptation speed up／down: 0より大きい1/second
- metering mask asset: nullable、検証済みTexture

Auto Exposureは`P15`のWorld HDRから測定し、Bloom、Vignette、UI、Accessibility overlayを入力に含めない。履歴はCamera／View Family単位で、Camera cutまたはmanual／auto切替時にresetする。

### 5.2 Tone Mapping／Color

Tone map modeはC1で次に閉じる。

- `aces_fitted_v1`
- `neutral_v1`
- `pixel_preserve_v1`

White BalanceはCCT 1000～20000 Kとtint -1～1を使う。Color Grade LUTは検証済み3D LUT artifact、intensity 0～1とする。Input／output color space、LUT size、shaper、hashをCook Receiptへ固定し、不一致LUTを推測変換しない。

`P50_TONE_COLOR`のC1依存順は`white_balance(scene-linear HDR) -> exposure_and_tone_map -> color_grade_lut(display-linear)`で固定する。別のdomainを要求するLUTはCatalogに別versionのNode IDと変換を宣言し、同じNode IDの意味をTarget別に変えない。

### 5.3 Bloom

`BloomProfileV1`は`enabled`、exposure-relative threshold EV -10～10、intensity 0～4、scatter 0～1を持つ。Mip count、kernel、sample数はTarget quality policyが決める。Pixel Crispでは既定offとし、明示Intentがある場合もpixel-locked layerとUIを除外する。

### 5.4 Motion Blur

`MotionBlurProfileV1`は`enabled`、shutter angle 0～360 degree、最大velocity clamp、camera／object寄与policyを持つ。Sample countとtile sizeはEngine-ownedである。Low Latency、VR、Pixel Crisp、Camera cut直後は既定offまたはTarget policyで無効化する。

### 5.5 Depth of Field

`DepthOfFieldProfileV1`はfocus source、subject groupまたはfocus distance 0.01～100000 m、最大CoC 0～64 pixel、quality intentを持つ。Focal length、aperture、sensor sizeはCamera Lens Profileを参照し、Post Process Profileへ複製しない。対象subjectが消失した場合のfallbackは`camera_lens`または最後の有効距離をProfileで明示する。

### 5.6 Vignette

`VignetteProfileV1`はintensity 0～1、roundness 0～1、centerを持つ。UI、Text、cursor、Accessibility overlayへ適用しない。Gameplay clarityまたはAccessibility Safeでは最大値をTarget Policyで制限する。

### 5.7 C2 Node

SSAO、SSR、高品質DOF、SMAA、Temporal Upscale ProviderはC2とする。Nodeごとに別Capability、reference fallback、Visual fixture、Target実測、無効化可能性が必要である。SSR失敗時に未定義色を返さず、reflection probe／Environment fallbackへ戻る。SSAOをGameplay visibilityへ使わない。

## 6. AA、Layer、UIとの互換

### 6.1 AA接続

本規約は`ResolvedAntiAliasingPlanV1`を入力として参照し、AA Resolverを重複実装しない。次を禁止する。

- TAAとTAAU／Temporal Providerの同時実行
- Hybrid DeferredのG-bufferへMSAAを暗黙適用
- Pixel-locked UI／Textをtemporal historyへ混入
- `P60`のFXAA／SMAAをUI合成後に実行
- Post Profileからjitter sequence、history weight、resolve sample数を上書き

### 6.2 Layer policy

`PostProcessLayerPolicyV1`は最低限、`world_opaque`、`world_transparent`、`vfx`、`pixel_locked_world`、`ui_text`、`cursor`、`accessibility_overlay`を区別する。Node Catalogは対象layerを閉じたbit maskで宣言する。

World内pixel artはTarget Profileにより`P50`のpixel-preserve tone mappingまでは許可できるが、Temporal、Bloom、Motion Blur、DOF、Display AAの対象外とする。UI／Text／cursorは`P80`で合成し、World Postの入力にしない。

### 6.3 HDR／SDR

Nodeはscene-linear HDR、display-linear、encoded outputのどれを入出力するか宣言する。Tone Mapping前にdisplay-referred effectを置かず、Tone Mapping後にscene luminanceを必要とするeffectを置かない。HDR10／scRGB／SDR変換はRenderer／PlatformのOutput Transformが所有し、Post Profileは出力policyだけを参照する。

## 7. Temporal History

各temporal Nodeは`PostHistoryDescriptorV1`を持つ。

```text
PostHistoryDescriptorV1
  node_id
  view_family_id
  camera_id
  algorithm_version
  extent
  logical_format
  quality_revision
  generation
  valid_region
```

次で必ずresetする。

- Camera cutまたはView Family変更
- extent、dynamic resolution scale、format変更
- Node enable、algorithm version、quality revision変更
- Resolved AA／Post Plan hashの互換性破壊変更
- surface generation、device generation変更
- projection mode、jitter policy、world originの非互換変更
- Replay seek、time discontinuity

Reset reasonはtelemetryとReplay Evidenceへ記録する。AIはhistory weight、generation、reset抑制を変更できない。

## 8. Resolver

`PostProcessIntentResolverV1`は副作用を持たない。

```text
resolve(
  PostProcessIntentV1,
  PostProcessProfileV1,
  PostProcessVolumeSummaryV1,
  VisualStyleProfile,
  CameraPresentationSummaryV1,
  ResolvedAntiAliasingPlanV1,
  LayerCompositionSummaryV1,
  TargetCapabilitySnapshotV1,
  PostProcessBudgetEnvelopeV1,
  AccessibilityPolicySnapshotV1
) -> ResolvedPostProcessPlanV1 | PostProcessDiagnosticSetV1
```

解決順は固定する。

1. Schema、base revision、scope、権限を検証する。
2. Project、Camera、Volumeを安定sortしてProfileを合成する。
3. Visual Style、Camera、AA、Layer、Accessibility制約を適用する。
4. CatalogからTargetで実行可能なNodeだけを選ぶ。
5. exact parameter、history、resource、costを解決する。
6. Budget超過と非互換へfallback chainを適用する。
7. Plan、理由、Preview、承認要否を出力する。

LLMは候補Intent生成を支援できるが、Plan順序、互換、parameter clamp、fallback、Plan hashはResolverが決める。

## 9. AI／Editor Operation

| Operation | Risk | 結果 |
|---|---:|---|
| `operation.post_process.search` | R0 query | Profile／Volume／Nodeのbounded一覧 |
| `operation.post_process.read` | R0 query | field mask付きversioned Intent／Profile／Volume／Plan |
| `operation.post_process.inspect` | R0 query | View summary、active node、cost、history、Diagnostic |
| `operation.post_process.resolve_intent` | R0 query | `ResolvedPostProcessPlanV1` |
| `operation.post_process.plan_change` | R1 proposal | `PostProcessChangeSetProposalV1`、typed diff、risk、approval |
| `operation.post_process.preview_change` | R0 query／job | before／after artifact、visual metric |
| `operation.post_process.explain_plan` | R0 query | stage、採用／棄却、fallback理由 |
| `operation.post_process.estimate_cost` | R0 query | Target別予測とconfidence |
| `operation.post_process.validate_change` | R0 query／job | Schema／semantic／Capability／Budget結果 |

外部MCP aliasは`mirakan.post_process.search`等とする。Searchは既定50件、最大200件、継続token付きとし、InspectはView Familyとfield maskを必須にする。Full-resolution history image、全frame capture、GPU memory dumpは通常応答へ含めず、別のbounded Debug Operationと権限を使う。

OperationはProfile／Volumeを直接保存しない。共通`AuthoringCommandGateway`のPlan、Preview、承認、Commitを通る。Project Technique、任意Shader、外部LUT生成、R2 Asset変更は別Operationと承認を要求する。

### 9.1 AI向けSummary

`PostProcessContextSummaryV1`は次を返す。

- View Family、Camera、Target、Visual Style、AA PlanのID／version
- active Profile／Volumeと上位32件
- stage別active Node ID、quality、履歴状態
- HDR／SDR、layer policy、pixel-locked有無
- 予測／実測GPU時間、persistent／transient byte
- active Diagnosticとfallback
- 詳細取得用Stable ID

これによりAIはRenderer全文やRender Graph dumpを読まずに変更可能範囲を理解できる。

## 10. PreviewとExplain

Post Previewは固定camera、fixed timestep、fixed exposure seedで最低限、次を生成する。

- Before／After beauty
- pre-exposure HDR reference
- post-tone display reference
- bloom contribution
- motion／CoC／AO／reflection debug view
- UI exclusion mask
- history reset timeline
- GPU stage timingとresource summary

`PostProcessPlanExplanationV1`はIntent fieldからNode／parameterへの写像、AA／UI／Accessibilityによる制約、棄却Node、fallbackで失われる見た目、予測cost、Plan hashを返す。主観語だけで「良くなった」と説明しない。

## 11. BudgetとFallback

Runtimeの`Post／Exposure`枠は通常1.00 msを合計上限とする。Temporal Reconstructionを選ぶTargetでは合計1.50 msの置換枠とし、1.00 msと1.50 msを加算しない。各Target Profileはstage／Nodeごとに次を宣言する。

- GPU時間の基準resolutionとscale式
- CPU Plan／Volume resolve時間
- persistent history byte
- transient resource byteとalias可能性
- bandwidth estimate
- dynamic resolution、HDR、View数の倍率

既定fallback順は次とする。

1. Vignette、Decorative Bloom等の装飾品質を下げる。
2. Bloom mip、Motion Blur、DOF品質を下げる。
3. SSRをprobe／Environment reflectionへ戻す。
4. SSAO品質を下げ、必要なら無効化する。
5. Temporal Providerをqualified portable AAへ戻す。
6. Tone Mapping、Color policy、UI／Accessibility、critical clarityを維持できなければPlanを拒否する。

fallbackはProfile Sourceを破壊せずTarget別Planだけを変える。Targetが回復した場合もframeごとに揺れないよう、quality transitionはhysteresisとcooldownをTarget Runtime Policyで管理する。

## 12. Diagnostic

| ID | 意味 |
|---|---|
| `MIRAKAN-POST-SCHEMA-INVALID` | Profile／Intent／Node parameter不正 |
| `MIRAKAN-POST-NODE-UNKNOWN` | CatalogにないNode |
| `MIRAKAN-POST-STAGE-INVALID` | Nodeのstage／dependency不正 |
| `MIRAKAN-POST-COLORSPACE-MISMATCH` | 入出力色空間不一致 |
| `MIRAKAN-POST-AA-CONFLICT` | AA PlanとPost Nodeが非互換 |
| `MIRAKAN-POST-LAYER-VIOLATION` | UI／pixel-locked／Accessibility対象違反 |
| `MIRAKAN-POST-VOLUME-CONFLICT` | 非Blend fieldのVolume競合 |
| `MIRAKAN-POST-HISTORY-INVALID` | history descriptor／generation不正 |
| `MIRAKAN-POST-TARGET-UNSUPPORTED` | Target Capabilityで実行不能 |
| `MIRAKAN-POST-BUDGET-EXCEEDED` | GPU／CPU／Memory Budget超過 |
| `MIRAKAN-POST-ASSET-UNQUALIFIED` | LUT／mask／Project Technique未検証 |
| `MIRAKAN-POST-STALE-PLAN` | revision／AA／Catalog／Target変更 |
| `MIRAKAN-POST-LOCK-CONFLICT` | human lockとIntentが競合 |
| `MIRAKAN-POST-NONDETERMINISTIC` | 同一入力でPlan hash不一致 |

全Diagnosticはseverity、owner、View／Profile／Node Stable ID、Target、Evidence、fallback、推奨修正を持つ。

## 13. Capability段階

| 段階 | Post Process機能 |
|---|---|
| C0 | MCD、Intent／Profile／Volume／Catalog／Plan、headless Resolver、Validator、色変換reference |
| C1 | Manual／Auto Exposure、Tone Mapping、White Balance、Color LUT、Bloom、簡易DOF、Vignette、AA接続、UI分離 |
| C2 | Motion Blur、SSAO、SSR、高品質DOF、SMAA、TAAU／qualified Provider、HDR Target強化 |
| C3 | Project Post Technique、Neural effect、offline reference。個別正式仕様とGateが必要 |

C1は任意Graphを提供せず、閉じたCatalogとProfileで完成させる。C2 Nodeは一つずつreference fallback、cross-target fixture、Budget Receiptを伴って昇格する。

## 14. 検証とEval

### 14.1 Contract／Resolver

- Intent、Profile、Volume、Catalog、Planのpositive／negative fixture
- Profile継承、循環、Volume安定sort／blend
- parameter range、unit、色空間、Stage検証
- AA、Layer、HDR、Accessibility互換表
- 同一入力同一Plan hash
- stale revision、human lock、unsupported Target

### 14.2 Runtime／Renderer

- 固定Stage順と依存hash
- Camera cut、resize、dynamic resolution、AA切替、device lossのhistory reset
- UI／Text／cursor／pixel-locked layerの除外
- SDR、scRGB、HDR10 Targetのoutput transform接続
- D3D12／Vulkan／Metalのlogical Plan一致
- 最大32 Volume、multi-view、split-screen、rapid profile changeのsoak
- Budget超過時の安定fallbackとhysteresis

### 14.3 Visual Eval

固定Sceneとして、暗所から屋外への移動、高輝度emissive、pixel art、2D Sprite、PBR室内／屋外、Toon、透明／VFX、字幕／HUD、色覚Accessibility、激しいCamera motion、focus pullを持つ。

AI Evalは最低限、次を含む。

- 「映画的だがHUDは読みやすく」
- 「画面酔いを抑え、動く敵を鮮明に」
- 「pixel artの輪郭を一切ぼかさない」
- 「低価格Androidで色調を維持して軽量化」
- 「露出のちらつきを抑え、屋内外を自然に移動」
- 「既存の人手調整Color Gradeを変えない」

Schema妥当性、lock保持、AA／UI互換、Budget、Plan再現性、visual metric、説明で判定する。

## 15. 有名Engineから採用する原則

| Engine | 参考にする公開方式 | Miraikanai Engineでの採用 |
|---|---|---|
| Unreal Engine | Post Process Volume、Camera／Volume override、Editor Property | Scope、priority、override、bounded Volume blend |
| Unity SRP | Volume Framework、Scriptable Render Pipeline、Render Graph | Profile Assetと実行Pipelineの分離、Target別Catalog |
| Godot | Environment Resource、Camera attributes、明示的設定 | Scene参照と再利用可能Profileの明快さ |

Miraikanai固有の追加点は、Intentとexact Profileを分離し、全変更へTyped Operation、Resolver、Plan、Preview、Explain、Budget、Diagnostic、Qualificationを要求することである。他Engineのraw Component API、custom pass injection、Scene serializationをAI公開契約としてコピーしない。

参照する一次資料:

- [Unreal Engine Post Process Effects](https://dev.epicgames.com/documentation/en-us/unreal-engine/post-process-effects-in-unreal-engine)
- [Unity Graphics repository](https://github.com/Unity-Technologies/Graphics)
- [Unity Render Graph introduction](https://github.com/Unity-Technologies/Graphics/blob/master/Packages/com.unity.render-pipelines.high-definition/Documentation~/render-graph-introduction.md)
- [Unity Custom Pass reference](https://github.com/Unity-Technologies/Graphics/blob/master/Packages/com.unity.render-pipelines.high-definition/Documentation~/custom-pass-reference.md)
- [Godot Environment and post-processing](https://docs.godotengine.org/en/stable/tutorials/3d/environment_and_post_processing.html)

## 16. 実装順序

1. C0 MCDへIntent／Profile／Volume／Catalog／Plan／Diagnosticを定義する。
2. 色空間、parameter、Profile merge、Volume blendのheadless referenceを作る。
3. Rendererへ固定Stageとlogical resource contractを接続する。
4. Manual Exposure、Tone Mapping、Color、Bloomの最小vertical sliceを作る。
5. Auto Exposure、AA Plan、history reset、UI／pixel-locked分離を接続する。
6. 簡易DOF、Vignetteを一つずつfixtureとBudget付きで追加する。
7. Editor／AI Operation、Preview、Explain、Receiptを接続する。
8. D3D12／Vulkan／MetalとTarget実機でC1をQualificationする。
9. Motion Blurを含むC2 Nodeを一つずつCapabilityとして追加する。

## 17. Definition of Done

Post Process C1は次をすべて満たした時だけ完了する。

1. `PostProcessIntentV1`、`PostProcessProfileV1`、`PostProcessCameraOverrideV1`、`PostProcessVolumeV1`、`PostProcessNodeCatalogV1`、`ResolvedPostProcessPlanV1`がMCDから生成される。
2. Nodeの色空間、Stage、依存、history、Target、Layer、costがCatalogで閉じている。
3. Resolverが同一入力で同一Plan hashを生成する。
4. AI／Editorが同じTyped OperationとChangeSet経路を使う。
5. AIへRender pass順、native resource、history weight、任意Shaderを公開していない。
6. AA Resolverを重複せず、`ResolvedAntiAliasingPlanV1`との互換検証が通る。
7. UI／Text／cursor／pixel-locked／Accessibility layerを誤ってWorld Postへ通さない。
8. Camera cut、resize、AA／quality切替、device lossのhistory resetが再現可能である。
9. Before／After Preview、Explain、Plan hash、Target costが同じEvidenceへ結び付く。
10. SDR／HDR、D3D12／Vulkan／Metalのcross-target fixtureが通る。
11. 1.00 msまたはTemporal置換1.50 ms枠、Memory上限、fallbackを実測Receiptで証明する。
12. C2／C3 NodeがC1の暗黙必須依存になっていない。
