# Miraikanai Engine Materials Contract

- 文書ID: mirakan.arch.rendering-materials
- 状態: review
- 正本範囲: Material／Shader source authoring、Material Domain／Shading Model、Visual Style／表現Profile、semantic material intent、Material IR／instance、offline compile／package、Material固有operation／diagnostic／qualification
- 非正本範囲: Render pass／queue／AA execution、Lighting物理意味、Post Process composition、LOD共通selection、Asset transaction、Runtime shared capacity、Tool／compiler version、AI authorization、Evidence envelope、共通Schema／projection。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Render Graph](render-graph.md)、[Lighting](lighting.md)、[Post Processing](post-processing.md)、[LOD](lod.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論と所有境界

Material authoringは人間とAIにsemantic intent、bounded parameter、Preview、Explainを公開し、Rendererへimmutable `CookedMaterialArtifact`とtyped instance bindingを渡す。shader source、graph、generated code、compiler invocation、reflection、pipeline variantはAuthoring／Cook境界の内側に置き、Project C++やRuntime AIへnative shader／descriptor操作を公開しない。

本書はMaterial、Shader、Visual Styleの意味だけを所有する。Render pass／resource／queue／AAは[Render Graph](render-graph.md)、physical light semanticsは[Lighting](lighting.md)、volume／effect compositionは[Post Processing](post-processing.md)、representation selectionは[LOD](lod.md)を参照する。

共通Source revision、ChangeSet、Undo／Redo、promotionは[Project state](../03-authoring/project-state.md)と[Asset lifecycle](../03-authoring/asset-lifecycle.md)、operation envelopeとprojectionは[Executable contracts](../02-foundation/executable-contracts.md)だけが所有する。

## 2. 制作モデルと正規Authoring object

Authoring surfaceは`MaterialSourceAsset`、`MaterialInstanceSourceAsset`、`VisualStyleProfileAsset`、`ShaderModuleSourceAsset`に分ける。

- `MaterialSourceAsset`: Stable ID、domain、shading model、surface intent、typed parameter declaration、texture／sampler role、graph／module ref、render-state intent。
- `MaterialInstanceSourceAsset`: parent material ref、parameter override、texture binding、style override、compatibility constraint。
- `VisualStyleProfileAsset`: palette／tone、shape／edge、surface response、detail／noise、lighting response、post-process hintのsemantic axes。
- `ShaderModuleSourceAsset`: approved source identity、entry interface、stage、capability requirement、include closure、authoring provenance ref。

Material graphはclosed typed node family、typed edge、single domain outputを持つ。arbitrary command、file／network access、native include path、Backend pragmaをSource Documentへ埋め込まない。unknown node、cycle、type mismatch、missing output、unbounded loop／resource、domain-incompatible nodeはCook前に拒否する。

Material Instanceはparentのdomain／shading model／interfaceを変更せず、宣言済みoverrideだけを持つ。継承chainはboundedでcycleを拒否し、Cook時にcanonical flat parameter setへ解決する。

## 3. Semantic CatalogとVisual Style

Material semantic vocabularyは物理値と表現intentを分離する。最低限、surface family、opacity behavior、normal response、roughness／specular intent、emissive intent、transmission／subsurface intent、two-sided intent、decal／overlay intentをclosed axesとして扱う。未知語を近い名前へ黙って補正せず、質問、候補、必要Capabilityを返す。

Visual Styleは特定shader graphのpresetではなく、複数MaterialとLighting／Post Processへ解決できるProject-owned semantic profileである。Styleの各axisは`inherit | override | forbid`を区別し、local Material overrideとProject defaultの優先順をcanonical resolverで決める。Style適用がMaterial DomainやTarget Capabilityと矛盾する場合は部分適用せずresolution errorを返す。

AI intent resolutionはsource request identity、Project revision、Catalog revision、resolved Material／Style refs、assumption／question、compatibility resultを束ねる。共通envelope fieldやhash表現は[Executable contracts](../02-foundation/executable-contracts.md)を参照し、本書はMaterial固有payloadの意味だけを決める。

## 4. Material DomainとShading Model

Material Domainはsurface、masked surface、transparent surface、decal、UI／sprite、particle、volumeをclosed familyとして扱う。各Domainは許可node、required output、depth／blend intent、lighting participation、shadow participation、motion-vector requirementを宣言する。

Shading Modelはunlit、physically based surface、foliage／cloth、subsurface、clear-coat等のEngine-owned semantic familyであり、Backend shader model名ではない。Targetが未対応の場合はMaterial意味が同等な登録済みfallbackだけを使い、opaque化、unlit化、output dropをsilentに行わない。

render-stateはraw API enumではなく、cull intent、depth test／write intent、blend semantic、alpha coverage、sort classをtyped intentで表す。[Render Graph](render-graph.md)がTarget capabilityとPass interfaceに照らして実行可能なPipeline keyへ解決する。

## 5. 表現Profileとparameter binding

公式表現ProfileはProjectのVisual Styleを再利用可能なnamed profileへ固定し、Material assetへの一括上書きではなくresolver inputとして参照する。Profileは対象Domain、required semantic axes、forbidden combination、fallback profile、qualification refを持つ。

Parameterはscalar、vector、color、texture role、enum、booleanのclosed typeとunit／color-space semantics、range、default、override policyを持つ。Runtime bindingはCook済みlayoutのStable Parameter IDを使い、文字列名、pointer、descriptor indexによるdispatchを禁止する。unknown parameter、type mismatch、stale layout generationはhard errorとする。

Texture roleはbase color、normal、mask、emissive、detail等の意味を表し、asset formatやchannel packingはCooked Artifactへ閉じる。Source revisionを跨ぐbinding混在を避け、Material、texture、shader artifactのgeneration closureを一つのpromotion単位として検証する。

## 6. Material IR、Shader compile、package

CookerはSource graph／moduleをBackend-neutral `MaterialIR`へlowerし、constant folding、dead-node removal、interface validation、variant canonicalizationを行う。IRはDomain output、typed operation、resource role、uniform layout、feature requirementを持ち、native bytecodeやcompiler-specific metadataをpublic contractにしない。

Shader compileはoffline build／cookだけで実行し、Shipping Runtimeにsource compiler、unapproved source、debug fallback shaderを含めない。compile結果はTarget Profile、Material IR hash、interface generation、toolchain lock ref、binary artifact、reflection summary、diagnosticを束ねる。compiler／translatorのexact version、commit、license、build optionは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)だけが所有する。

Variant keyはDomain、Shading Model、declared feature、vertex interface、Pass interface、Target capabilityからcanonicalに作り、arbitrary user keywordやruntime stringで増殖させない。Packageは使用closureからrequired variantを列挙し、missing critical variantをruntime compileで隠さない。

## 7. Material LOD境界

Materialは各representationで利用可能なMaterial artifact、feature reduction、texture residency requirement、意味同等fallbackを宣言する。[LOD](lod.md)がrepresentationとtransitionを選択し、Materialsはその選択に対応するbindingを返す。

距離、projected error、hysteresis、CPU／GPU pressureからMaterial tierを直接選ばない。Material側のfeature reductionはsurface identity、opacity、silhouetteに影響する意味変更を明示し、未宣言のshader simplificationを禁止する。

## 8. AI／Editor operationとPreview

Material operationはcreate／update material、create instance、bind texture role、apply style、set semantic parameter、compile preview、explain resolution、validate packageをDomain actionとして登録する。操作は[Executable contracts](../02-foundation/executable-contracts.md)の共通Discovery／Preview／Apply境界と[AI Security／Approval](../01-governance/ai-security-approval.md)のauthorityを使う。

Previewは対象revision、Target Profile、View／Lighting fixture ref、resolved Material／Style、compiled artifact generation、difference summary、diagnosticを返す。Preview結果をApply済みProject stateやProduction qualificationと表示しない。Explainは採用値、継承元、override、fallback、未解決questionをMaterial語彙で示す。

## 9. Diagnostic、failure、fallback

Material固有diagnosticはasset／node／parameter／style axis／variant key、source span、Target、error code、remediationを含む。少なくともunknown semantic、graph cycle、type mismatch、domain mismatch、missing resource、unsupported shading model、variant explosion、compile failure、reflection mismatch、stale bindingを区別する。

fallbackはSourceで宣言され、意味差とTarget範囲を持つ。compile失敗、missing texture、capacity不足時にdefault material、opaque、unlit、texture dropへ黙って切り替えない。共通capacity／backpressureは[Runtime performance／capacity](../04-runtime/performance-capacity.md)へ委譲する。

## 10. Qualificationと完了条件

Qualificationは各Domain／Shading Modelのgolden fixture、graph validation、parameter binding、style resolution、offline compile、reflection、variant closure、missing／corrupt asset、Target fallback、Render Graph interfaceを検証する。

Visual comparison、Evidence envelope、Eval grading、provenanceは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を使う。本書はfixtureのMaterial input、expected semantic resolution、allowed visual tolerance classだけを所有し、共通receipt schemaを再定義しない。

Runtime source compile、unbounded variant、string dispatch、stale artifact mix、silent default material、unqualified fallbackが残るPackageはRelease候補にしない。Capability maturityと実装順は[Product Plan](../00-product/product-plan.md)だけが決定する。
