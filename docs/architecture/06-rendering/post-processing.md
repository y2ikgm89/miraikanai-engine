# Miraikanai Engine Post Processing Contract

- 文書ID: mirakan.arch.rendering-post-processing
- 状態: review
- 正本範囲: Post Process Source／Volume、effect catalog／parameter semantics、volume blend／priority／scope、ordered effect composition、history intent、Post Process operation／diagnostic／qualification
- 非正本範囲: Render pass／resource／queue／AA execution、Material／Lighting semantics、Camera／Environment source、UI composition、Runtime shared capacity、AI authorization、Evidence envelope、共通Schema／projection。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Render Graph](render-graph.md)、[Materials](materials.md)、[Lighting](lighting.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論と所有境界

Post ProcessingはVolumeとEffectのauthoring semantics、blend、priority、scope、ordered compositionを一意に所有し、ViewFamilyごとにimmutable `ResolvedPostProcessPlan`を生成する。RendererはそのPlanを登録済みPass Template、resource、history、queueへ展開するが、effect順やparameter意味を変更しない。

AA／temporal provider、Render Graph pass／resource／queue、surface／UI compositeは[Render Graph](render-graph.md)が所有する。MaterialとLightingのsemantic valueをPost Process parameterへ複写せず、必要なscene inputだけをtyped dependencyとして宣言する。

Camera、Environment、UIのSource schemaは本書の対象外である。それらから届くview tag、exposure context、environment hint、pixel-locked layer policyを入力として消費し、Owner不在のfieldを先回りして定義しない。

## 2. 正本データモデル

`PostProcessProfileV1`を次に固定する。

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

Profile継承は最大4段とし循環を拒否する。各nodeは`inherit | disabled | override`を明示する。配列の暗黙append、自由なnode追加、同じNode IDの重複を禁止する。C1はProfile当たり最大32 `node_settings`、View Family当たり最大32 active Nodeで、Target Profileは上限を下げられるが増やせない。上限超過を切り捨てずfallbackまたはtyped Diagnosticを返す。

`PostProcessCameraOverrideV1`は`base_profile_id`、`base_profile_revision`、field mask、最大16件のpartial `node_settings`だけを持ち、Profile継承、Node追加、stage変更、Target Capability追加はできない。

`PostProcessVolumeV1`は`volume_id`、`shape_ref`、`profile_id`、`priority_i16`、`blend_radius_m`、`blend_weight`、`unbounded`、`enabled`、`target_selector`を持つ。Viewごとの交差Volumeは最大32とし、`priority -> volume_id`で安定sortする。weightはshape距離から0～1へ決定する。Blend可能fieldだけを線形または定義済みDomain blendし、enum、asset ref、Node enable等は最高priorityかつStable ID最小の一件を選ぶ。競合は`MIRAKAN-POST-VOLUME-CONFLICT`で説明する。

Effect entryはEffect Stable ID、effect kind、enabled intent、parameter override、固定composition stage、Catalog dependency ref、required input／history、Target capability、fallbackを持つ。Profile／AIはstageや順序を変更できない。任意shader、native pass、resource handle、command callbackをSourceへ埋め込まない。

Parameterはtyped value、unit／color-space semantics、valid range、blend operator、override stateを持つ。`unset`、明示default、override値を区別し、unknown field、non-finite値、type／unit mismatchを拒否する。

Volume shapeのgeometryとcontainment queryは既存Simulation contractを利用し、本書はblend semanticsだけを所有する。Runtime collision eventやPhysics bodyをVolume activationのauthoritative sourceにしない。

## 3. Effect Catalogとcomposition stage

Effect Catalogはtone／exposure adaptation、color transform、bloom／glare、depth／motion based effect、lens／camera presentation、stylization、spatial cleanup等をclosed familyとして登録する。各effectはinput color-space、output color-space、required buffers、parameter definition、history requirement、allowed scope、ordering relation、fallbackを宣言する。

`PostProcessNodeCatalogV1`の各Nodeはversion付きNode ID／Capability maturity、入力／出力logical resourceとcolor space、固定execution stage、required Camera／Renderer input、temporal historyの有無／format／reset reason、blend可能parameter／range、対応Target／HDR・SDR／AA mode／layer、予測cost model／persistent・transient byte式、fallback node／disable policy、Visual fixture／conformance test／Qualification refを宣言する。通常ProfileはCatalogにないNodeを作れない。

C1 Node／parameter contractを次に固定する。

- `ExposureProfileV1`: `mode: manual_ev100 | histogram_auto`、manual EV100 -16～32、histogram 256 bins、luminance percentile low 0.5%／high 99.5%、middle gray 0.18、output EV100 -6～16、adaptation speedは0より大きい1/second、nullableな検証済みmetering mask。`P15`のWorld HDRだけを測定しUI等を含めない。
- Tone map mode: `aces_fitted_v1 | neutral_v1 | pixel_preserve_v1`。White BalanceはCCT 1000～20000 K／tint -1～1、Color Grade LUTは検証済み3D LUT／intensity 0～1。`P50`順は`white_balance(scene-linear HDR) -> exposure_and_tone_map -> color_grade_lut(display-linear)`で固定する。
- `BloomProfileV1`: `enabled`、exposure-relative threshold EV -10～10、intensity 0～4、scatter 0～1。Pixel Crispは既定offでpixel-locked／UIを除外する。
- `MotionBlurProfileV1`: `enabled`、shutter angle 0～360 degree、maximum velocity clamp、camera／object contribution policy。Low Latency、VR、Pixel Crisp、camera cut直後は既定offまたはTarget policyで無効化する。
- `DepthOfFieldProfileV1`: focus source、subject groupまたはfocus distance 0.01～100000 m、maximum CoC 0～64 pixel、quality intent。Camera Lens fieldを複写しない。
- `VignetteProfileV1`: intensity 0～1、roundness 0～1、center。UI、Text、cursor、Accessibility overlayへ適用しない。

SSAO、SSR、高品質DOF、SMAA、Temporal Upscale ProviderはC2で、NodeごとにCapability、reference fallback、Visual fixture、Target実測、disable可能性を必要とする。SSR失敗はreflection probe／Environment fallback、SSAOはGameplay visibilityへ使わない。

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

SSAOはOpaque Lightingのambient visibility、SSRはlit Opaqueのreflection contributionとして`P10`前に合成する。表を一本のfull-screen chainとは解釈しない。同一stageの非可換NodeはCatalogのdependency edgeを必須とし、依存なしNodeはCatalogが`commutative=true`の場合だけNode ID順で実行する。曖昧な順序とcycleは`MIRAKAN-POST-STAGE-INVALID`で拒否する。`P20`と`P60`は`ResolvedAntiAliasingPlanV1`が選択した一方または許可組合せだけを実行し、ProfileはAA methodやsample countを指定しない。

## 4. Volume resolveとparameter blend

ResolverはViewFamily、view position／tags、active World／Level scope、Project default profile、intersecting Volume、explicit Camera override refから一つのPlanを作る。同priority VolumeはStable ID順でdeterministicに処理し、worker completion順やviewport selection順を使わない。

blend operatorはlinear、normalized weight、nearest／highest priority、boolean select、enum select等をparameter definitionごとに固定する。color、angle、exposure、curve等は型固有の補間意味を持ち、全parameterをscalar lerpへ落とさない。

global、World、Level、Cell、Camera／View scopeの優先関係は明示されたscope chainから解決する。scope外Volumeやstale revisionを無視したまま成功表示せずdiagnosticへ残す。

## 5. AA、Layer、UIとの互換

EffectはAA前後の必要stage、motion／depth／reactive input、pixel-locked互換性をCatalogで宣言する。[Render Graph](render-graph.md)はResolved AA PlanとPost Process Planの互換性を検証し、historyやjitter ownershipを一意にする。

UIとEditor overlayは既定でscene exposure、temporal reconstruction、depth effectの対象外とし、pixel-locked layer contractを維持する。Effectがoverlayを必要とする場合は専用Effect familyとqualificationを要求し、scene effectの対象scopeを拡張しない。

Material／LightingのSource valueをPost Process resolverが書き換えない。exposure compensation等がLightの物理値を変更したように見える場合も、表示変換とScene valueをExplainで分離する。

## 6. Temporal history intent

Temporal effectはhistory semantic、required input、initialization、reset mask、warm-up disposition、fallbackを宣言する。history keyはViewFamily、effect Stable ID、effect／provider generation、surface generation、extent、projectionへ束縛する。

camera cut、teleport、projection／extent／surface／effect generation変更、missing motion／depthではresetを要求する。実際のresource allocation、barrier、queue、lease、AA provider historyは[Render Graph](render-graph.md)が所有する。本書はhistoryの意味とreset要求だけを決める。

## 7. Resolver outputと実行境界

`ResolvedPostProcessPlanV1`は最低限、View Family／Camera／Profile／Volume／Target／AA Planのversion／hash、stage順のversion付きNode ID、全Nodeの解決済みexact parameter、入力／出力logical resource／color space／resolution class、history key／generation／reset条件、Layer inclusion／exclusion、予測CPU／GPU時間／persistent・transient byte、採用／棄却／fallback／非互換理由、Preview／承認／Qualification要否、`plan_hash`／有効期限を持つ。approval mechanicsと共通artifact／projection fieldは[AI Security／Approval](../01-governance/ai-security-approval.md)と[Executable contracts](../02-foundation/executable-contracts.md)を参照する。Planはnative pass listではない。

RendererはPlan内のEffect Catalog IDを登録済みPass Templateへ展開する。AI、Editor、Project C++がarbitrary pass、shader source、resource alias、queue、native formatをPlanへ挿入することを禁止する。

## 8. AI／Editor operationとPreview

Post Process operationはcreate／update profile、create／update volume、set enabled／effect parameter、apply style hint、preview、explain、validateをDomain actionとして登録する。Profile／AIによるstage変更、Node追加、固定execution order変更をOperationとして公開しない。共通Discovery、Preview、Apply、ChangeSet、authorizationは[Executable contracts](../02-foundation/executable-contracts.md)、[Project state](../03-authoring/project-state.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)を参照する。

Previewは対象revision、ViewFamily fixture、contributing Volume、resolved order／parameters、color-space transition、AA compatibility、history reset、fallback、diagnosticを示す。Explainは各値のSource、priority、weight、blend operator、override理由を追跡可能にする。

## 9. Diagnostic、failure、fallback

Post Process固有diagnosticはProfile／Volume／Effect Stable ID、parameter path、scope、stage、Target、error code、remediationを含む。少なくともunknown effect、invalid parameter、scope conflict、order cycle、missing input、AA incompatibility、color-space mismatch、history invalid、unsupported Target、stale revisionを区別する。

| Closed ID | 意味 |
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
| `MIRAKAN-POST-BUDGET-EXCEEDED` | Planが割当capacityを満たさない |
| `MIRAKAN-POST-ASSET-UNQUALIFIED` | LUT／mask／Project Technique未検証 |
| `MIRAKAN-POST-STALE-PLAN` | revision／AA／Catalog／Target変更 |
| `MIRAKAN-POST-LOCK-CONFLICT` | human lockとIntentが競合 |
| `MIRAKAN-POST-NONDETERMINISTIC` | 同一入力でPlan hash不一致 |

missing effect、capacity不足、history欠損時にeffect drop、parameter clamp、順序変更をsilentに行わない。CatalogまたはProfileで宣言した意味差付きfallbackだけを使う。共通capacityとbackpressureは[Runtime performance／capacity](../04-runtime/performance-capacity.md)へ委譲する。

## 10. Qualificationと完了条件

Qualificationは次のDomain fixtureを持つ。

- Intent、Profile、Volume、Catalog、Planのpositive／negative fixture、Profile継承／循環、Volume stable sort／blend。
- parameter range／unit／color space／fixed Stage、AA／Layer／HDR／Accessibility互換、同一入力同一Plan hash、stale revision、human lock、unsupported Target。
- 固定Stage順とdependency hash、camera cut、resize、dynamic resolution、AA切替、device lossのhistory reset。
- UI／Text／cursor／pixel-locked layer除外、SDR／scRGB／HDR10 output transform接続。
- D3D12／Vulkan／Metalのlogical Plan一致、最大32 Volume、multi-view、split-screen、rapid profile changeのsoak、capacity超過時fallback／hysteresis。
- 暗所から屋外への移動、高輝度emissive、pixel art、2D Sprite、PBR室内／屋外、Toon、透明／VFX、字幕／HUD、色覚Accessibility、激しいCamera motion、focus pullの固定Scene。
- AI corpusは「映画的だがHUDは読みやすく」「画面酔いを抑え、動く敵を鮮明に」「pixel artの輪郭を一切ぼかさない」「低価格Androidで色調を維持して軽量化」「露出のちらつきを抑え、屋内外を自然に移動」「既存の人手調整Color Gradeを変えない」を含み、Schema妥当性、lock保持、AA／UI互換、Plan再現性、visual metric、説明で判定する。

visual Evidence、Eval、provenance envelopeは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を使う。本書はPost Process inputとexpected composition／image tolerance classだけを所有し、共通receipt fieldやthresholdを複写しない。

任意pass挿入、insertion順依存、silent effect drop、UIへの暗黙適用、stale history再利用、Render Graph実行規則の複写が残る実装はRelease候補にしない。Capability maturityと実装順は[Product Plan](../00-product/product-plan.md)だけが決定する。
