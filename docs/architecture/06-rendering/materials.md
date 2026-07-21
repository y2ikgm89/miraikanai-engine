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

Visual Styleは特定shader graphのpresetではなく、Material、Light、Camera、Post、VFX、UI、Asset制作規則の整合を所有する。`VisualStyleProfile`の最低fieldを次に固定する。

```text
schema_version
profile_id
source_profile_id: optional
display_name
scene_dimension
art_direction
composition
composition_variant: native | crisp_sprite_over_high_res_3d | unified_low_resolution
gameplay_space: canvas_2d | world_3d
camera_profile_id
lighting_profile_id
post_process_profile_id
vfx_style_profile_id
ui_style_profile_id
art_asset_profile_id
animation_presentation_profile_id
allowed_material_domains[]
allowed_shading_models[]
material_template_by_semantic_role{}
texture_policy
  color_texture_encoding: srgb
  data_texture_encoding: linear
  default_filter: point | linear | anisotropic
  mip_policy: none | nearest | linear
  integer_scale_policy: not_applicable | letterbox | crop | fractional_scale
  pixels_per_unit: null | positive_number
  reference_resolution: null | { width: positive_integer, height: positive_integer }
  transform_policy: unrestricted | axis_aligned_pixel | logical_resolution_rasterized
  world_texel_density: null | object
    reference_distance_m: positive_number
    min_screen_pixel_ratio: positive_number
    max_screen_pixel_ratio: positive_number
outline_profile_id: optional
palette_profile_id: optional
quality_profile_id
fallback_policy: forbid | allow_listed
allowed_fallbacks[]
style_critical_fields[]
reference_assets[]
  asset_id
  content_hash
  source_uri_or_local_provenance
  license_or_usage_basis
  extracted_attributes[]
locked_fields[]
```

Engine同梱Profileはimmutable templateである。Project変更時は全fieldを解決した派生Profileを新規作成し、Runtime inheritance、複数親、自動伝播chainを持たない。`gameplay_space`がPhysics／Navigationの所有空間を決め、見た目からdimensionを推測しない。

`pixel_2d`と`unified_low_resolution`は正整数`reference_resolution`、1以上の`pixels_per_unit`、`not_applicable`以外のinteger scale policyを必須とする。`crisp_sprite_over_high_res_3d`は`reference_resolution = null`としてCamera Profileの出力解像度を使う。`world_texel_density`は`pixel_diorama`でだけ必須で、`min_screen_pixel_ratio <= max_screen_pixel_ratio`、既定0.8～1.2とする。Camera変更時に`reference_distance_m`を暗黙更新しない。

`style_critical_fields`と`locked_fields`はProfile内JSON Pointerである。`fallback_policy = allow_listed`は1件以上のfallback、`forbid`は空配列を必須とする。`license_or_usage_basis`は`user_owned | licensed | public_domain | reference_only`に固定する。利用根拠のないreference AssetをCommitしない。

AI intent resolutionはsource request identity、Project revision、Catalog revision、resolved Material／Style refs、assumption／question、compatibility resultを束ねる。共通envelope fieldやhash表現は[Executable contracts](../02-foundation/executable-contracts.md)を参照し、本書はMaterial固有payloadの意味だけを決める。

## 4. Material DomainとShading Model

Graph出力はDomainごとに固定し、任意Render passやGPU resourceへ接続させない。

| Domain | 許可Shading Model | 主な出力 |
|---|---|---|
| `surface_3d` | `pbr_metal_rough`、`toon_surface`、`unlit_surface` | normal、color、opacity、model固有parameter |
| `sprite_2d` | `sprite_unlit`、`sprite_lit`、`sprite_toon` | premultiplied color、opacity、2D normal、emissive、mask |
| `hybrid_sprite_3d` | `hybrid_sprite_unlit`、`hybrid_sprite_lit`、`hybrid_sprite_toon` | Sprite出力、depth coverage、shadow coverage |
| `decal` | `pbr_decal`、`toon_decal` | 許可されたbase color／normal／roughness channel |
| `vfx` | `vfx_unlit`、`vfx_lit` | color、opacity、distortion、emissive |
| `ui` | `ui_unlit` | premultiplied color、opacity、clip mask |
| `post_process` | 登録済みPost node | HDR colorまたはmask。Depth／motionはread-only |

`AlphaMode`は`opaque | mask | blend_premultiplied`に固定し、Instance変更を禁止してDefinitionのcompile featureとする。`blend_premultiplied`はdepth write既定off、`mask`はcoverage判定後にdepthを書く。Domain／Shading Model／output不一致は`MIRAKAN-MATERIAL-DOMAIN_MISMATCH`、AlphaMode／parameter不正は`MIRAKAN-MATERIAL-PARAMETER_INVALID`で拒否し、別familyへ黙って近似しない。

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

Diagnostic IDを`MIRAKAN-MATERIAL-SEMANTIC_ROLE_UNKNOWN | MIRAKAN-MATERIAL-DOMAIN_MISMATCH | MIRAKAN-MATERIAL-GRAPH_INVALID | MIRAKAN-MATERIAL-PARAMETER_INVALID | MIRAKAN-MATERIAL-TEXTURE_ENCODING_MISMATCH | MIRAKAN-MATERIAL-CAPABILITY_NOT_ACTIVATED | MIRAKAN-MATERIAL-BUDGET_EXCEEDED | MIRAKAN-MATERIAL-INTERFACE_MISMATCH | MIRAKAN-MATERIAL-VARIANT_LIMIT | MIRAKAN-MATERIAL-COMPILE_FAILED | MIRAKAN-MATERIAL-STYLE_LOCK_VIOLATION | MIRAKAN-MATERIAL-FALLBACK_REQUIRED | MIRAKAN-MATERIAL-PROVENANCE_MISSING | MIRAKAN-MATERIAL-PREVIEW_STALE | MIRAKAN-MATERIAL-UNAUTHORIZED_SOURCE | MIRAKAN-MATERIAL-COLLISION_NAMESPACE_MISMATCH`に固定する。

Editor previewだけがcompile失敗時に直前のvalid pipelineを明示表示できる。Shippingは不合格Artifact、missing／invalid Materialを含むPackageまたはScene loadを失敗させる。Capability不足時に別Styleへ黙って変更しない。

fallbackはSourceで宣言され、意味差とTarget範囲を持つ。compile失敗、missing texture、capacity不足時にdefault material、opaque、unlit、texture dropへ黙って切り替えない。共通capacity／backpressureは[Runtime performance／capacity](../04-runtime/performance-capacity.md)へ委譲する。

## 10. Qualificationと完了条件

Qualificationは次のDomain fixtureを持つ。

| 対象 | 必須Test |
|---|---|
| PBR | Khronos glTF Asset Generator／Sample Assets／Validator |
| Toon | Sphere、顔、髪、透明髪、outline、Key／accent Light |
| Pixel 2D | 720p、1080p、1440p、ultrawide、4Kのscale／letterbox |
| Pixel Diorama | depth、occlusion、shadow coverage、Fog、DOF、Bloom、TAA分離 |
| Compiler | invalid Graph、resource上限、禁止HLSL、binding不一致、cache再現 |
| Target | Windows、Android、Appleのoffline compile、pipeline、fallback |

同一Reference GPU／driverのgolden imageはSSIM 0.995以上、絶対channel差2／255超のpixelが0.1%未満を既定Gateとする。Cross-vendorはparameter ordering、luminance、finite、outline width、pixel grid等のanalytic invariantを検証する。

Material AI Evalは10 suite（明示Intentのrole／Template、既存Instance最小変更、Template再利用判断、Graph型／単位／色空間、曖昧／矛盾質問、Target／fallback、Variant／resource、禁止HLSL／任意pass、Visual／Collision Material分離、Preview／ChangeSet／undo／redo／recook）、各12 fixture、合計120 fixtureを各3回実行する。hard gate違反、無権限Commit、unsupported成功表示は0件、exact Operation／Type／unit／range、Blocking確認、禁止操作拒否は360／360、role／Template選択100%、Preview hash／undo／redo／recook一致100%を要求する。意味、Schema、画像、GPU性能を別scoreとしhard failureを平均で相殺しない。

Visual Style Resolver Evalは明示、未指定、委任、矛盾、未対応を各12件、合計60 prompt、各3回実行し、hard gate違反と無権限Commit 0件、Blocking質問／明示Style保持／委任scope／unsupported拒否180／180を要求する。

Visual comparison、Evidence envelope、Eval grading、provenanceは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を使う。本書はfixtureのMaterial input、expected semantic resolution、allowed visual tolerance classだけを所有し、共通receipt schemaを再定義しない。

Runtime source compile、unbounded variant、string dispatch、stale artifact mix、silent default material、unqualified fallbackが残るPackageはRelease候補にしない。Capability maturityと実装順は[Product Plan](../00-product/product-plan.md)だけが決定する。
