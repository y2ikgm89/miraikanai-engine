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

`PostProcessProfileAsset`はStable ID、effect entry集合、default scope、Target compatibility、fallback profile、qualification refを持つ。`PostProcessVolumeSource`はshape／global scope、priority、blend distance／weight、profile ref、override集合、view／layer filterを持つ。

Effect entryはEffect Stable ID、effect kind、enabled intent、parameter override、composition stage、ordering constraint、required input／history、Target capability、fallbackを持つ。任意shader、native pass、resource handle、command callbackをSourceへ埋め込まない。

Parameterはtyped value、unit／color-space semantics、valid range、blend operator、override stateを持つ。`unset`、明示default、override値を区別し、unknown field、non-finite値、type／unit mismatchを拒否する。

Volume shapeのgeometryとcontainment queryは既存Simulation contractを利用し、本書はblend semanticsだけを所有する。Runtime collision eventやPhysics bodyをVolume activationのauthoritative sourceにしない。

## 3. Effect Catalogとcomposition stage

Effect Catalogはtone／exposure adaptation、color transform、bloom／glare、depth／motion based effect、lens／camera presentation、stylization、spatial cleanup等をclosed familyとして登録する。各effectはinput color-space、output color-space、required buffers、parameter definition、history requirement、allowed scope、ordering relation、fallbackを宣言する。

Composition stageはscene-linear pre-light-composite、scene-linear pre-AA、AA connection、scene-linear post-AA、display mapping、display-referred overlay前等のEngine-owned semantic stageで表す。Backend pass名や外部Engineのinjection pointを正本語彙にしない。

Effectの順序はstage、explicit before／after constraint、Stable ID canonical orderから決定する。constraint cycle、同一exclusive slot、color-space discontinuity、missing required inputはPlan生成時に拒否し、authoring insertion順へfallbackしない。

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

`ResolvedPostProcessPlan`はsource revisions、ViewFamily ref、ordered effect entries、resolved typed parameters、stage、required inputs、history intent、compatibility result、fallback disposition、diagnostic refsを持つ。共通artifact／projection fieldは[Executable contracts](../02-foundation/executable-contracts.md)を参照する。

RendererはPlan内のEffect Catalog IDを登録済みPass Templateへ展開する。AI、Editor、Project C++がarbitrary pass、shader source、resource alias、queue、native formatをPlanへ挿入することを禁止する。

## 8. AI／Editor operationとPreview

Post Process operationはcreate／update profile、create／update volume、set effect parameter、reorder constraint、apply style hint、preview、explain、validateをDomain actionとして登録する。共通Discovery、Preview、Apply、ChangeSet、authorizationは[Executable contracts](../02-foundation/executable-contracts.md)、[Project state](../03-authoring/project-state.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)を参照する。

Previewは対象revision、ViewFamily fixture、contributing Volume、resolved order／parameters、color-space transition、AA compatibility、history reset、fallback、diagnosticを示す。Explainは各値のSource、priority、weight、blend operator、override理由を追跡可能にする。

## 9. Diagnostic、failure、fallback

Post Process固有diagnosticはProfile／Volume／Effect Stable ID、parameter path、scope、stage、Target、error code、remediationを含む。少なくともunknown effect、invalid parameter、scope conflict、order cycle、missing input、AA incompatibility、color-space mismatch、history invalid、unsupported Target、stale revisionを区別する。

missing effect、capacity不足、history欠損時にeffect drop、parameter clamp、順序変更をsilentに行わない。CatalogまたはProfileで宣言した意味差付きfallbackだけを使う。共通capacityとbackpressureは[Runtime performance／capacity](../04-runtime/performance-capacity.md)へ委譲する。

## 10. Qualificationと完了条件

QualificationはVolume overlap／priority／blend、effect ordering、typed parameter、color-space chain、AA connection、history reset、UI exclusion、unsupported Target、fallback、stale revisionをViewFamily fixtureで検証する。

visual Evidence、Eval、provenance envelopeは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を使う。本書はPost Process inputとexpected composition／image tolerance classだけを所有し、共通receipt fieldやthresholdを複写しない。

任意pass挿入、insertion順依存、silent effect drop、UIへの暗黙適用、stale history再利用、Render Graph実行規則の複写が残る実装はRelease候補にしない。Capability maturityと実装順は[Product Plan](../00-product/product-plan.md)だけが決定する。
