# Miraikanai Engine Material／Visual Style／AI Authoringアーキテクチャ規約

- 文書版: 1.0
- 作成日: 2026-07-20
- 対象: 描画Material、Visual Style、Shader authoring、Editor、AI Operation、Preview、Explain、Validator、Eval
- 基盤規約: [Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)
- Authoring規約: [Miraikanai Engine AIネイティブゲームエンジン制作設計書](./2026-07-18-ai-native-game-engine-authoring-design.md)
- 契約規約: [Miraikanai Engine 実行可能契約・Schema Code Generation規約](./2026-07-19-executable-contract-schema-codegen-design.md)
- Renderer規約: [Miraikanai Engine Rendering／Render Graphアーキテクチャ規約](./2026-07-19-rendering-render-graph-architecture-design.md)
- Asset規約: [Miraikanai Engine Asset Pipeline／Content Package規約](./2026-07-19-asset-pipeline-content-packaging-design.md)
- Asset Import規約: [Miraikanai Engine Asset Import／AI Authoring／Editor UX規約](./2026-07-20-asset-import-ai-authoring-editor-ux-design.md)
- LOD規約: [Miraikanai Engine AI可読LODアーキテクチャ規約](./2026-07-20-ai-readable-lod-architecture-design.md)
- Collision規約: [Miraikanai Engine Collision／Colliderアーキテクチャ規約](./2026-07-19-collision-collider-architecture-design.md)
- AI権限規約: [Miraikanai Engine AI実装・保守ガバナンス規約](./2026-07-19-ai-engine-development-governance-design.md)
- AI検証規約: [Miraikanai Engine AI検証・評価・来歴規約](./2026-07-19-ai-verification-evaluation-provenance-design.md)

## 0. 30秒で分かる公式方式

Miraikanai Engineの描画Materialは、**型付きMaterial IR、複数Shading Model、version付きVisualStyleProfile、意味Catalog、型付きAI Operation**を一つの契約として扱う。

人間とAIは、次の順で最小限の自由度を選ぶ。

```text
Intent
  -> MaterialTemplate
  -> MaterialInstance
  -> MaterialDefinition／typed Graph
  -> 承認済みProject HLSL
```

通常の制作はTemplateとInstanceで完了させる。GraphはTemplateで表現できない場合、Project HLSLはGraphで表現不能または人間が明示要求した場合だけ使用する。AIへraw GPU command、任意Render pass、native binding、未検証Shader sourceを公開しない。

Material変更は必ず次を通る。

```text
Catalog検索
  -> bounded inspect
  -> Plan
  -> Preview／Explain／Estimate
  -> Validator
  -> typed ChangeSet
  -> Approval
  -> Commit
  -> offline Cook／Compile
  -> Receipt
```

「画面に出た」だけでは完成としない。意味、Target、Style、budget、failure、fallback、画像、性能、AI操作の全Gateを満たしたCapabilityだけをProduction表示する。

### 目的別の読み方

| 目的 | 読む節 |
|---|---|
| Materialを作る／調整する | 3、4、5、9、10 |
| Visual Styleを決める | 6、8、9.3、10 |
| Graph／Shaderを実装する | 5.3、7、11、12 |
| AI Toolを実装する | 5、9、10、14、15 |
| 問題を診断する | 11、14、15 |
| Phase／完了条件を確認する | 16、17 |

## 1. 決定権と境界

### 1.1 本書が所有するもの

- `MaterialSemanticCatalogV1`
- `MaterialNodeCatalogV1`
- `MaterialDefinition`
- `MaterialInstance`
- `MaterialTemplate`
- Material Domain、Shading Model、Alpha Mode
- `VisualStyleProfile`
- `StyleCapabilityManifest`
- Material／Style向けAI OperationとContext projection
- Material Preview、Explain、Validator、Diagnostic、Eval
- Shader authoring Level、variant policy、Material budget
- Material LODの意味不変条件
- C0～C3の成熟度とPromotion Gate

### 1.2 他規約が所有するもの

| 対象 | 正本 |
|---|---|
| RenderSnapshot、Render Graph、GPU resource lifetime、Backend Adapter | Renderer規約 |
| Source import、Texture encoding、Cook、Package、Hot Reload | Asset／Asset Import規約 |
| 共通LOD Intent、Target別Plan、transition、Receipt | LOD規約 |
| 摩擦、反発、密度、contact surface | Collision規約 |
| ChangeSet、revision、approval、R0～R5 | Authoring／AI権限規約 |
| MCD Type／Operation、code generation | 契約規約 |

本書と他規約で同じfieldを重複定義しない。本書はMaterialの意味を、各Subsystem規約は実行境界とArtifactを所有する。

### 1.3 描画MaterialとCollision Material

描画MaterialとCollision Materialは別namespace、別Asset、別Validatorとする。

```text
rendering.material.*
collision.material.*
```

見た目から摩擦、反発、Damage、足音を推測しない。必要なProjectは明示的な`SurfaceSemanticBindingV1`を使用する。

```text
SurfaceSemanticBindingV1
  binding_id: StableId
  semantic_surface_id: StableEnumId
  render_material_instance_id: optional AssetId
  collision_material_2d_id: optional AssetId
  collision_material_3d_id: optional AssetId
  audio_surface_id: optional AssetId
  vfx_surface_id: optional AssetId
```

Bindingは参照を接続するだけで各Domainの値を複製しない。描画Material変更でPhysics値を、Collision Material変更でShader parameterを暗黙変更しない。

## 2. 設計原則

1. **意味を名前より優先する。** File名、folder名、Shader名、自然言語だけを正規identityにしない。
2. **一つのAuthoring SourceからTarget別Artifactを作る。** Windows、Android、Apple用に別Materialを手作業で維持しない。
3. **Instanceを通常経路にする。** 色やTexture変更でGraph revisionやvariantを増やさない。
4. **Domainごとに出力を閉じる。** Graphから任意pass、UAV、native resourceへ接続させない。
5. **AIの自由度を段階化する。** Intent、Template、Instance、Graph、Project HLSLの順で必要最小限を選ぶ。
6. **Discoveryを段階化する。** AIへ全Catalogや巨大Graphを常時渡さない。
7. **PreviewとExplainをCommit前に行う。** Compile成功だけを見た目の承認にしない。
8. **ValidatorをModelから独立させる。** LLMの自己評価をhard gateに使用しない。
9. **StyleとQualityを混同しない。** Quality低下でRealisticをToonへ、PixelをIllustratedへ変更しない。
10. **Runtime compileを禁止する。** Shipping ArtifactはTarget別にoffline compileする。
11. **失敗を隠さない。** missing Material、unsupported feature、fallback欠落を成功表示しない。
12. **未成熟Capabilityを発見不能にする。** Catalogへ掲載されない機能をAIは選択できない。

## 3. 人間向け制作モデル

### 3.1 段階的な編集Level

| Level | 利用者が扱うもの | 既定用途 | Risk |
|---|---|---|---|
| 0 Intent | 「濡れた石」「3段影の青い髪」等 | 目的と制約の入力 | R0／R1 |
| 1 Template／Instance | role、Texture、公開parameter | 日常的な制作 | R2 ChangeSet |
| 2 Material Graph | 型付きNode、公開parameter定義 | Templateで不足する表現 | R2 ChangeSet |
| 3 Safe expression | 閉じた式subset | 小さな数式拡張 | R2 ChangeSet |
| 4 Project HLSL | Engine entry／binding内のmodule | Graphで表現不能なProject表現 | R3 |
| 5 Engine拡張 | Node、Domain、Shading Model、Compiler | 複数Project向けCapability | R4 |

AIは低いLevelで要件を満たせる場合に高いLevelを選ばない。

### 3.2 Editor projection

Basic、Advanced、Expertは同じ正規DocumentのProjectionとする。

| View | 主な表示 |
|---|---|
| Basic | Intent、semantic role、Template、主要parameter、Preview、warning |
| Advanced | 全公開parameter、Texture channel、Target／Quality、dependency、budget |
| Expert | Graph、variant、ShaderInterface、compile、pass participation、diagnostic |

別View間で値を複製しない。Basicで変更した値は同じDocument revisionとしてAdvanced／Expertへ即時反映する。

### 3.3 通常workflow

1. semantic roleまたは自然言語Intentを指定する。
2. Catalogから対応Templateを最大3件提示する。
3. 既存Materialとの再利用可能性を確認する。
4. Instance parameterだけで成立する案を優先する。
5. representative fixtureと実Sceneの両方をPreviewする。
6. Explain、Target差、cost、fallback、dependency Diffを確認する。
7. ChangeSetとしてCommitする。
8. offline Cook／Compile結果とReceiptを確認する。

## 4. 正規Authoring object

| Object | 責務 | 変更規則 |
|---|---|---|
| `MaterialDefinition` | Domain、Shading Model、typed graph、render state、compile feature | Graphまたはcompile feature変更で新revision |
| `MaterialInstance` | Textureと公開parameterの上書き | Domain、Shading Model、Alpha Mode、Depth policyを変更不可 |
| `MaterialTemplate` | semantic role、既定Definition、公開parameter policy | Style Profileがroleごとに選択 |
| `MaterialSemanticCatalogV1` | role、意味、channel、互換性、例、成熟度 | Engine build／Project extensionから生成 |
| `MaterialNodeCatalogV1` | Nodeの型、意味、cost、Target、制約 | Engine buildから生成。AIは変更不可 |
| `ArtAssetProfile` | Mesh／Sprite／Texture制作規則、Palette、role | Importer、Generator、Validatorが共有 |
| `AnimationPresentationProfile` | sampling、pose hold、motion accent | Simulationから分離した表示規則 |
| `VisualStyleProfile` | Material、Light、Camera、Post、VFX、UIのStyle契約 | ChangeSet＋Preview＋承認 |
| `StyleCapabilityManifest` | build／Target／Qualityで利用可能な機能 | Engine buildから生成。AIは変更不可 |
| `VisualStyleDecision` | 候補、除外理由、選択理由、未解決事項 | Decision Ledgerへ記録 |
| `MaterialExplanationV1` | Material判断の根拠、差、cost、fallback | Preview／Plan revisionへ紐付け |

全AssetはStable ID、schema version、content hash、revisionを持つ。Runtime packageへEditor node位置、UI state、AI prompt、自由形式の判断理由を持ち込まない。

## 5. Semantic Catalog

### 5.1 `MaterialSemanticCatalogV1`

Catalogは機械検索可能なsummaryと、必要時に読むexact entryを分離する。

```text
MaterialSemanticCatalogV1
  schema_version
  catalog_revision
  engine_build_id
  target_profile_ids[]
  quality_profile_ids[]
  entries[]
```

```text
MaterialSemanticEntryV1
  semantic_role_id: StableEnumId
  display_name
  description
  search_synonyms[]
  positive_examples[]
  counter_examples[]
  allowed_domains[]
  allowed_shading_models[]
  required_texture_channels[]
  optional_texture_channels[]
  parameter_semantics[]
  compatible_style_profile_ids[]
  required_capabilities[]
  target_support[]
  default_template_id
  allowed_fallback_template_ids[]
  preview_fixture_ids[]
  production_maturity: c0 | c1 | c2 | c3
```

`search_synonyms`、例、反例は検索と説明用であり、正規IDではない。Source Asset名や既存作品名だけからroleを確定しない。

初期role setは少なくとも次を持つ。

```text
surface.generic
surface.terrain
surface.foliage
surface.water
character.skin
character.hair
character.eye
character.cloth
prop.opaque
prop.transparent
sprite.actor
sprite.environment
sprite.effect
decal.surface
vfx.particle
ui.surface
```

Project固有roleはEngine roleを上書きせず、`project.<project_id>.*` namespaceへR3で追加する。

### 5.2 Parameter semantic

公開parameterは表示名だけでなく、次を持つ。

```text
MaterialParameterSemanticV1
  parameter_id: StableId
  semantic_id: StableEnumId
  value_type
  unit
  color_space: none | linear_rgb | srgb | data
  valid_range
  default_value
  default_provenance
  compile_time: bool
  ai_mutable: bool
  style_critical: bool
  runtime_mutable: bool
```

AIは`parameter_id`とexact typeを使用し、表示名またはShader内の文字列変数名で変更しない。

### 5.3 `MaterialNodeCatalogV1`

各Node entryは次を持つ。

```text
node_type_id
schema_version
display_name
semantic_description
input_ports[]
output_ports[]
allowed_domains[]
allowed_stages[]
required_capabilities[]
target_support[]
static_cost_estimate
resource_cost
determinism_class
failure_codes[]
examples[]
counter_examples[]
```

Nodeは型付きportを持ち、暗黙のscalar／vector拡張、linear／sRGB混在、normal／color混在を許可しない。Graph layoutはEditor専用で、semantic hashへ含めない。

## 6. Visual Style契約

`VisualStyleProfile`はProject全体の画風を一つのShader名へ縮約せず、Material、Light、Camera、Post、VFX、UI、Asset制作規則の整合を所有する。

最低fieldは次とする。

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

Engine同梱Profileはimmutable templateとする。Projectで変更する場合は全fieldを解決した派生Profileを新規作成し、Runtime inheritance、複数親、自動伝播chainを持たない。

`gameplay_space`がPhysics／Navigationの所有空間を決める。`pixel_diorama`等の見た目からPhysics dimensionを推測しない。Style-critical fieldの変更は、MaterialだけでなくLighting、Camera、Post、VFX、UI、Importへのdependency closureを一つの`StyleChangeSet`としてPreviewする。

`pixel_2d`と`unified_low_resolution`は正整数の`reference_resolution`、1以上の`pixels_per_unit`、`not_applicable`以外のinteger scale policyを必須とする。`crisp_sprite_over_high_res_3d`は`reference_resolution = null`とし、Camera Profileの出力解像度を使う。`world_texel_density`は`pixel_diorama`でだけ必須とし、`min_screen_pixel_ratio <= max_screen_pixel_ratio`、既定0.8～1.2とする。Camera変更時に`reference_distance_m`を暗黙更新しない。

`style_critical_fields`と`locked_fields`はProfile内を指すJSON Pointerである。`fallback_policy = allow_listed`は1件以上のfallback、`forbid`は空配列を必須とする。参考Assetの`license_or_usage_basis`は`user_owned | licensed | public_domain | reference_only`に固定する。

参考AssetはPalette、輪郭、陰影段数、Texture密度、Camera、構図、Material傾向へ分解する。既存作品名や作者名をProfile、Shader、Assetの正規名へ保存しない。利用根拠のない参考AssetはCommitしない。

## 7. Material DomainとShading Model

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

`AlphaMode`は`opaque`、`mask`、`blend_premultiplied`に固定する。Instance変更を禁止し、Definitionのcompile featureとする。`blend_premultiplied`はdepth write既定off、`mask`はcoverage判定後にdepthを書く。

## 8. 公式表現Profile

### 8.1 C1 `realistic_basic`

- Scene-linear HDR、物理単位Light、IBL
- Cook-Torrance、GGX、Smith height-correlated visibility、Schlick Fresnel
- Metallic-Roughness workflow
- Base color、metallic、perceptual roughness、normal、occlusion、emissive
- Opaque、alpha mask、premultiplied transparent
- Reflection probe、shadow、height fog、exposure、tone mapping
- glTF core、`KHR_materials_unlit`、`KHR_materials_emissive_strength`、`KHR_texture_transform`

Authoring上のroughness／metallicは0～1とし、shader内部のFP32 roughnessは0.045～1へclampする。

### 8.2 C2 `realistic_advanced`

- `KHR_materials_clearcoat`
- `KHR_materials_sheen`
- `KHR_materials_specular`
- `KHR_materials_ior`
- `KHR_materials_transmission`＋`KHR_materials_volume`
- `KHR_materials_anisotropy`
- `KHR_materials_iridescence`
- `KHR_materials_dispersion`
- `KHR_materials_variants`から独自`MaterialVariantSet`への変換
- Skin、Cloth、Hair、Eye、Foliage template
- Parallax occlusion mapping

未対応glTF extensionを黙って破棄しない。Import停止または承認済みfallback Previewのどちらかにする。

### 8.3 C3 Realistic Research

- offline mesh refinementを使うdisplacement
- spectral／thin-film reference
- offline path-traced Material reference
- neural Material approximation

C3はC1／C2の基準経路と意味同等fallbackを保持し、個別Qualification前にProduction表示しない。

### 8.4 Toon

ToonはPBR出力の最終Posterizeではなく、独立した`toon_surface` Shading Modelとする。Shaderだけで完成と判定せず、Silhouette、Texture、Palette、Outline、Animation timing、Camera、VFXを同じStyleとして検証する。

`toon_basic`は1D diffuse ramp、banded specular、rim、normal、emissive、Light Channel、一意なKey Lightを持つ。`toon_character`はFace Shadow Map、hair highlight／fringe、Skin／Hair／Eye／Cloth templateを追加する。

OutlineはC2でinverted hullを既定、screen-space方式をoptional、mesh edge／suggestive contourをC3とする。幅はsemantic roleごとに`screen_pixels`または`world_units`を明示する。

### 8.5 2D

| Profile | Texture | Camera／scale | Light／Material | 成熟度 |
|---|---|---|---|---|
| `pixel_2d` | Point、原則no mip | logical resolution、integer scale、pixel snap | Unlit／banded 2D light | C1 |
| `illustrated_2d` | Linear、mip optional | Subpixel可 | Unlit／2D normal light | C1 |
| `lit_sprite_2d` | Profileでfilter固定 | Orthographic | 2D normal、SDF shadow、emissive | C1 |
| `vector_graphic_2d` | tessellation cache | Resolution independent | fill、stroke、gradient | C2 |

`pixel_2d`の既定は`integer_scale_policy = letterbox`、`transform_policy = axis_aligned_pixel`とする。表示translationはpixel grid、rotationは90度の整数倍、scaleは整数倍へ制限するが、Simulation transformはfloatのまま保持する。任意角回転は`logical_resolution_rasterized`を明示選択する。整数倍表示が不可能な場合にfractional scaleへ黙って変更しない。

### 8.6 `pixel_diorama`

`pixel_diorama`は一つのShaderではなく、2D Pixel Artと3D空間を合成するProfileである。固有作品名、Palette、Camera、Post、Assetを模倣する正規presetを作らない。

公式composition modeは次の二つとする。

- `crisp_sprite_over_high_res_3d`
- `unified_low_resolution`

Spriteと3Dのdepth、occlusion、shadow coverage、Fog、transparent sortを明示し、Pixel-locked layerへTAA、motion blur、temporal upscalerを適用しない。

3D Materialのtexel密度とSpriteの見かけ上のpixel sizeはCamera基準距離で比較する。`screen_pixel_ratio = 3d_pixels_per_texel / sprite_pixels_per_texel`とし、Profile既定0.8～1.2の外をwarningにする。crisp modeは最終display pixel、unified modeはupscale前のlogical pixelで測定する。

## 9. AI向けDiscoveryとOperation

### 9.1 Progressive disclosure

AIへ最初に返す`MaterialContextSummaryV1`は次だけを含む。

```text
material_id
revision
semantic_role_id
template_id
definition_id
domain
shading_model
public_parameters[]
texture_dependencies[]
target_support[]
quality_support[]
variant_count
budget_summary
diagnostic_summary
available_operation_ids[]
```

Graph全体、全Node Catalog、compiler log、全Target Artifactは明示的なR0 readで必要sliceだけを取得する。Context上限超過時に配列を途中で切らず、cursorとtotal countを返す。

### 9.2 Operation Catalog

| Operation ID | Risk | 動作 |
|---|---:|---|
| `operation.material.search` | R0 | role、Style、Domain、Target、成熟度でCatalog検索 |
| `operation.material.read` | R0 | exact Material／Template／Node entryを取得 |
| `operation.material.inspect` | R0 | bounded summary、dependency、budget、diagnosticを取得 |
| `operation.material.preview` | R0 | fixture／Scene／Target／Quality Preview job |
| `operation.material.explain` | R0 | 選択理由、差、却下候補、fallbackを説明 |
| `operation.material.estimate` | R0 | resource、variant、GPU costを予測／実測区別付きで返す |
| `operation.material.validate` | R0 | Staging ChangeSetを検証 |
| `operation.material.plan` | R1 | Intentから`MaterialAuthoringPlanV1`を提案 |
| `operation.material.create_instance` | R2 | 既存Definition／TemplateからInstance ChangeSet作成 |
| `operation.material.assign_template` | R2 | semantic roleへTemplateを割当 |
| `operation.material.set_parameters` | R2 | exact parameter IDと型付き値を変更 |
| `operation.material.create_definition` | R2 | typed Graphを持つDefinition draft作成 |
| `operation.material.edit_graph` | R2 | Node／edge／公開parameterを変更 |
| `operation.material.create_derived_style` | R2 | 全field解決済みStyle draft作成 |
| `operation.material.bind_surface_semantics` | R2 | 明示Surface Binding ChangeSet作成 |
| `operation.material.propose_project_module` | R3 | Project HLSL／Project role／Template拡張を提案 |
| `operation.material.propose_engine_extension` | R4 | Node、Domain、Shading Model、Compiler拡張を提案 |

R2 Operationは正規Assetを直接writeせず、base revision、precondition、expected impact、Preview hashを持つChangeSetを作る。CommitはAuthoring Gatewayが権限、revision、Approvalを再検証して実行する。

### 9.3 Resolver

Resolverの優先順位は次に固定する。

1. 人間がlockした決定
2. 現在の明示要求
3. 承認済みGame Brief／GameSpec
4. Provenance確認済み参考Assetから抽出した一般属性
5. 既存Assetとの互換性
6. Target、FPS、memory、制作量、期間
7. Project default
8. 明示委任された場合のAI推奨

GenreだけでStyleを決めない。Hard gateはEngineが行い、残った候補だけをAIが順位付けする。2D／3D／Hybrid、gameplay space等のBlocking事項は回答まで停止する。High Impact事項は明示または一件限定の委任がなければ最大3候補のPreviewを提示する。

`VisualStyleDecision`は次を必須出力する。

```text
request_id
decision_authority: explicit_human | delegated_ai | confirmed_recommendation
delegation_record_id: optional
resolved_requirements[]
unknowns[]
conflicts[]
eligible_profile_ids[]
rejected_candidates[]
selected_profile_id: optional
selection_reasons[]
production_cost_estimate
runtime_cost_estimate
required_capabilities[]
missing_capabilities[]
confidence: high | medium | low
requires_human_confirmation
```

`delegation_record_id`は一件限定の署名済み委任にだけ使用する。`medium`／`low`、Blocking unknown、Conflict、missing Capability、budget未計測のいずれかがあるDecisionは人間確認を必須とする。

## 10. Preview、Explain、Estimate

### 10.1 Preview fixture

| Fixture | 対象 |
|---|---|
| Sphere／Plane | 一般PBR、Texture channel、roughness／metallic sweep |
| Character head／hair cards | Skin、Hair、Eye、Toon、alpha |
| Sprite／pixel grid | 2D、filter、mip、pixel snap、integer scale |
| 3D＋Sprite scene | Pixel Diorama、depth、shadow、Fog、Post分離 |
| Target／Quality matrix | Windows、Android、Apple、Low／Medium／High |
| Actual Scene slice | dependency closure、Lighting、Camera、VFX、budget |

Previewは`source_revision`、`plan_hash`、`target_profile_id`、`quality_profile_id`、`artifact_hash`を返す。Commit直前にrevisionまたはhashが変わったPreviewを無効化する。

### 10.2 `MaterialExplanationV1`

```text
request_id
material_id
source_revision
resolved_intents[]
selected_semantic_role_id
selected_template_id
changed_parameters[]
selection_reasons[]
rejected_candidates[]
assumptions[]
target_differences[]
predicted_cost
measured_cost: optional
fallbacks[]
warnings[]
required_human_confirmations[]
confidence
```

`confidence`はAI自己申告でなく、Engineがunknown、Conflict、Capability、budget、明示／委任、Preview状態から再計算する。

### 10.3 Cost

予測値と実測値を同じ数値として扱わない。Prototype前は`predicted`、Reference Scene計測後は`measured`とし、Production GateはTarget別実測値だけを使用する。

## 11. ValidatorとBudget

Validatorは次の順で実行する。

1. **Semantic**: role、Template、Style、lock
2. **Graph**: cycle、port型、Domain出力、finite、range
3. **Asset**: Texture dimension、encoding、normal、alpha、dependency closure
4. **Capability**: Target、Quality、Shading Model、Node
5. **Budget**: Texture、Sampler、Node、Variant、optional pass、GPU時間
6. **Security**: include、filesystem、recursion、未証明loop、UAV、任意pass
7. **Compile**: DXIL、SPIR-V、MSL、ShaderInterface、pipeline
8. **Visual**: golden、parameter sweep、Style invariant

初期hard limitは次とする。

| Quality | Texture SRV／Material | Unique sampler | Graph node | Static variant／Definition | Optional Engine pass |
|---|---:|---:|---:|---:|---:|
| Low | 8 | 2 | 128 | 16 | 1 |
| Medium | 12 | 4 | 256 | 32 | 2 |
| High | 16 | 8 | 384 | 64 | 3 |

Instruction数だけをhard gateにしない。Compiler estimateはwarningと順位付けに使い、最終合否はReference SceneのGPU timing、frame P95、regressionで決める。

Compile-time featureはDefinitionだけが持つ。Instance parameterでvariantを増やさない。上限超過時はdynamic parameter化、semantic role別Definition分割、不要feature削除の順に修正し、上限を自動緩和しない。

## 12. Shader Compiler／Package

```text
Material Graph
  -> Domain／type validation
  -> Material IR
  -> Shading Model specialization
  -> portable HLSL 2021
  -> isolated Target compiler
       Windows: DXC -> DXIL -> validation
       Android: DXC -spirv -> SPIR-V -> SPIRV-Tools
       Apple: SPIR-V -> SPIRV-Cross -> MSL -> metal／metallib
  -> Engine ShaderInterface
  -> Target pipeline／Shader cache
```

- Binding layoutはEngineがDomain別に所有する。
- Material、Project code、AIはRoot Signature、descriptor set、Metal argument layoutを定義しない。
- Source、include、IR、Shading Model version、define、compiler、Target、optimizationからcache keyを作る。
- DXIL／SPIR-V／MSL reflectionを独自`ShaderInterface`へ正規化する。
- 複数Target ProjectはCapability intersectionで全variantをcompileする。
- Target限定featureは承認済みfallbackを必須とする。
- Shippingは全variantをoffline compileする。
- RuntimeでAI出力、Project source、download contentからShaderをcompile／loadしない。

初期Toolchain baselineはHLSL 2021、DXC v1.9.2602.24、SPIRV-Tools v2026.2、SPIRV-Cross Vulkan SDK 1.4.350.0とし、`toolchain.lock.json`へ取得元とhashを固定する。WindowsはShader Model 6.6とRoot Signature 1.1、AndroidはVulkan 1.1／Android Vulkan Profile 2022、AppleはA12／Apple family 5とXcode 26.6のoffline `metal`／`metallib`を最低基準とする。Toolchain更新は同じconformance、golden、Target compileを再実行してから昇格する。

Hot ReloadはMaterial、Texture、pipeline、Style dependency closureを一つのgenerationとして昇格する。Textureだけ新、Materialは旧の部分Version混在を禁止する。

## 13. Material LOD

Material LODはoffline compile済みvariantだけを選ぶ。次を削除または変更しない。

- Style-critical ramp、Palette、outline
- Alpha semantics、pixel sampling
- combat cue emissive、Gameplay visibility cue
- required Material interface
- depth／shadow参加の意味

`MaterialLodProfileV1`はtierごとにMaterial interface hash、allowed feature mask、texture residency floor、shadow／depth participation、visual equivalence toleranceを持つ。Runtime Shader生成、任意branch削除、Material mergeをLOD selection内で行わない。

## 14. DiagnosticとFailure

最低Diagnostic IDを次に固定する。

```text
MIRA-MATERIAL-SEMANTIC_ROLE_UNKNOWN
MIRA-MATERIAL-DOMAIN_MISMATCH
MIRA-MATERIAL-GRAPH_INVALID
MIRA-MATERIAL-PARAMETER_INVALID
MIRA-MATERIAL-TEXTURE_ENCODING_MISMATCH
MIRA-MATERIAL-CAPABILITY_NOT_ACTIVATED
MIRA-MATERIAL-BUDGET_EXCEEDED
MIRA-MATERIAL-INTERFACE_MISMATCH
MIRA-MATERIAL-VARIANT_LIMIT
MIRA-MATERIAL-COMPILE_FAILED
MIRA-MATERIAL-STYLE_LOCK_VIOLATION
MIRA-MATERIAL-FALLBACK_REQUIRED
MIRA-MATERIAL-PROVENANCE_MISSING
MIRA-MATERIAL-PREVIEW_STALE
MIRA-MATERIAL-UNAUTHORIZED_SOURCE
MIRA-MATERIAL-COLLISION_NAMESPACE_MISMATCH
```

DiagnosticはStable ID、revision、field／node／port、Target、Quality、actual、expected、consumer closure、修正候補を持つ。Vendor compiler logだけをユーザーへ返さず、原文は開発者詳細へ添付する。

- Editor previewはcompile失敗時に直前のvalid pipelineを明示表示できる。
- Shipping buildは不合格Artifactを含めない。
- Capability不足時に別Styleへ黙って変更しない。
- fallbackを使った場合は見た目とcostの差をBuild reportへ記録する。
- missing Profile／MaterialはEditorだけ診断Materialを表示する。
- Shippingはmissing／invalid MaterialでPackageまたはScene loadを失敗させる。

## 15. TestとMaterial専用Eval

### 15.1 Compiler／Visual

| 対象 | 必須Test |
|---|---|
| PBR | Khronos glTF Asset Generator／Sample Assets／Validator |
| Toon | Sphere、顔、髪、透明髪、outline、Key／accent Light |
| Pixel 2D | 720p、1080p、1440p、ultrawide、4Kのscale／letterbox |
| Pixel Diorama | depth、occlusion、shadow coverage、Fog、DOF、Bloom、TAA分離 |
| Compiler | invalid Graph、resource上限、禁止HLSL、binding不一致、cache再現 |
| Target | Windows、Android、Appleのoffline compile、pipeline、fallback |

同一Reference GPU／driverのgolden imageはSSIM 0.995以上、絶対channel差2／255超のpixelが0.1%未満を既定Gateとする。Cross-vendorはpixel完全一致でなく、parameter ordering、luminance、finite、outline width、pixel grid等のanalytic invariantを検証する。

### 15.2 AI Eval corpus

Material Evalは10 suite、各12 fixture、合計120 fixtureを持ち、各3回実行する。

1. 明示Intentからrole／Templateを選択
2. 既存Instanceの最小変更
3. Template再利用と新規Graphの必要性判断
4. Graph編集と型／単位／色空間
5. 曖昧、矛盾、High Impact質問
6. Target、Quality、fallback
7. Variant、resource、GPU budget
8. 禁止HLSL、任意pass、GPU resource要求
9. Visual Material／Collision Material分離
10. Preview、ChangeSet、undo／redo、cook再現

合格条件を次に固定する。

- Engine hard gate違反: 0件
- 無権限Commit: 0件
- 未対応機能の成功表示: 0件
- exact Operation／Type／unit／range適合: 360／360
- Blocking／High Impact確認動作: 360／360
- 禁止操作の拒否: 360／360
- 明示Intentのrole／Template選択: 100%
- 推奨候補順位: 承認済みrubricに対して95%以上
- Preview hash、undo／redo、再Cookの一致: 100%

意味選択、Schema適合、画像、GPU性能を別scoreとして保持し、平均点でhard failureを相殺しない。

### 15.3 Visual Style Resolver Eval

Style選択はMaterial編集Evalと分離し、明示、未指定、委任、矛盾、未対応を各12件、合計60 prompt、各3回実行する。Engine hard gate違反と無権限Commitは0件、Blocking質問、明示Style保持、委任scope、unsupported拒否は180／180、推奨候補順位は承認済みrubricに対して95%以上を合格条件とする。

## 16. Capability成熟度と実装順

| Capability | C0 | C1 | C2 | C3 |
|---|---|---|---|---|
| Semantic Catalog／Operation | Schema、fixture、R0 query | Template／Instance R2 | Graph／Style R2 | Project／Engine拡張 |
| VisualStyleProfile／Resolver | Schema、Validator、Decision | 2D候補 | 3D／Hybrid比較 | Custom style補助 |
| 2D | Pixel／Illustrated schema | `pixel_2d`、`illustrated_2d`、`lit_sprite_2d` | Vector、advanced post | Specialized |
| Realistic | Material IR、PBR Preview | `realistic_basic` | advanced、Skin／Hair／Eye／Cloth | RT／offline reference |
| Toon | Schema、ramp／outline Preview | 非Production | `toon_basic`、`toon_character` | advanced contour |
| Pixel Diorama | Schema、composition Preview | 非Production | 両composition mode | Large-world／advanced Camera |
| Project HLSL | Interface設計 | 非Production | sandbox prototype | Stable extension Gate後 |

Phase配置は次とする。

- Phase 2まで: Material IR、Catalog、Validator、Preview、offline compiler基盤
- Phase 3: 2D C1 Profile
- Phase 4: AI候補生成、推薦、委任時選択、Material R0～R2
- Phase 6: `realistic_basic`
- Phase 7: Mobile Target validation
- Phase 8: `realistic_advanced`、Toon、Pixel Dioramaを一Capabilityずつ昇格
- Phase 8以後: Project HLSL、RT／offline reference等の個別Gate

未完成Profile、Template、Node、OperationをCapability Manifestへ掲載せず、AIへ選択させない。

## 17. 完了の定義

Material機能は次をすべて満たした場合だけ完了とする。

1. MCDでType、Operation、Capability、Requirementを定義している。
2. C++、TypeScript、Cooked descriptor、AI Toolが同じ正本から生成される。
3. `ai_mutable` fieldのtyped Operation coverageが100%である。
4. Basic／Advanced／Expert Editorで作成、編集、Preview、undoできる。
5. AIがCatalog検索、Plan、Explain、Preview、ChangeSet提案を実行できる。
6. Semantic、Graph、Asset、Capability、Budget、Security Validatorがある。
7. 全Targetへoffline Cook／Compileできる。
8. Diagnostic、Debug view、telemetryがある。
9. Quality tierと意味同等fallbackがある。
10. Unit、conformance、integration、visual、performance、AI Evalに合格する。
11. Invalid input、OOM、device loss、missing Asset、compile失敗の動作が定義される。
12. Reference SceneでTarget budgetを満たす。
13. Runtime packageへEditor Graph layoutやAI promptを含めない。
14. 未成熟CapabilityをAIが発見または選択できない。

## 18. 他Engineから採用する原則

| Engine | 採用する原則 | Miraikanaiで追加する契約 |
|---|---|---|
| Unreal Engine | Domain、Shading Model、Material Instance、Material Function、parameter grouping | semantic role、bounded AI Operation、Capability hard gate |
| Unity | Material＋Shader property、Shader Graph、Sub Graph、Keyword／variant管理 | Stable parameter ID、Target共通IR、variant hard budget |
| Godot | Resourceの単純さ、Standard Material、ShaderMaterial、VisualShader | schema version、typed unit／color space、Preview／Explain Receipt |

既存EngineのAsset形式、Editor API、Shader schemaを正本として複製しない。比較は制作上有効な原則の確認に使い、MiraikanaiのStable ID、MCD、Capability、ChangeSetを置き換えない。

## 19. 一次資料

- [Khronos glTF 2.0 Specification](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html)
- [Khronos glTF Extension Registry](https://github.com/KhronosGroup/glTF/tree/main/extensions)
- [Khronos glTF Sample Assets](https://github.com/KhronosGroup/glTF-Sample-Assets)
- [DirectX Shader Compiler](https://github.com/microsoft/DirectXShaderCompiler)
- [SPIR-V Specification](https://registry.khronos.org/SPIR-V/)
- [SPIRV-Tools](https://github.com/KhronosGroup/SPIRV-Tools)
- [SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross)
- [Apple Metal Shading Language Specification](https://developer.apple.com/metal/Metal-Shading-Language-Specification.pdf)
- [Unreal Engine Materials](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-materials)
- [Unity Shader Graph](https://docs.unity3d.com/Manual/com.unity.shadergraph.html)
- [Godot Standard Material 3D](https://docs.godotengine.org/en/stable/tutorials/3d/standard_material_3d.html)
- [Godot VisualShader](https://docs.godotengine.org/en/stable/classes/class_visualshader.html)
