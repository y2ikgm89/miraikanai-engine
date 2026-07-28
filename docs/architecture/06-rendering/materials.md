# Miraikanai Engine Materials Contract

- 文書ID: mirakan.arch.rendering-materials
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Material Domain／Shading Modelの意味、Visual Style／表現Profile、semantic material intent、Material Graph／Function／IR／instance、MaterialからProject Shaderへのtyped接続、Material compile／package、Material固有operation／diagnostic／qualification
- 非正本範囲: Project HLSL source profile／semantic Module／Technique／Shader AI理解、Render pass／queue／AA execution、Lighting物理意味、Post Process composition、LOD共通selection、Asset transaction、Runtime shared capacity、Tool／compiler version、AI authorization、Evidence envelope、共通Schema／projection。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Render Graph](render-graph.md)、[Project Shader](project-shader.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)
- 関連文書: [Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[World](world.md)、[Render Graph](render-graph.md)、[Project Shader](project-shader.md)、[Lighting](lighting.md)、[Post Processing](post-processing.md)、[LOD](lod.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-28

## 1. 結論と所有境界

Material authoringは人間とAIにsemantic intent、bounded parameter、Preview、Explainを公開し、Rendererへimmutable `CookedMaterialArtifactV1`とtyped instance bindingを渡す。Graph、generated code、compiler invocation、reflection、pipeline variantはAuthoring／Cook境界の内側に置き、Project C++やRuntime AIへnative shader／descriptor操作を公開しない。Project HLSLを使う場合は[Project Shader](project-shader.md)のQualification済み`ProjectShaderModuleV1`／Node／Shading ModelだけをStable IDとinterface hashで参照する。

本書はMaterial、Shader、Visual Styleの意味だけを所有する。Render pass／resource／queue／AAは[Render Graph](render-graph.md)、physical light semanticsは[Lighting](lighting.md)、volume／effect compositionは[Post Processing](post-processing.md)、representation selectionは[LOD](lod.md)を参照する。

共通Source revision、ChangeSet、Undo／Redo、promotionは[Project state](../03-authoring/project-state.md)と[Asset lifecycle](../03-authoring/asset-lifecycle.md)、operation envelopeとprojectionは[Executable contracts](../02-foundation/executable-contracts.md)だけが所有する。

## 2. 制作モデルと正規Authoring object

Authoring surfaceのcanonical objectを次に固定する。下記と別のAsset名をalias／accepted typeとして設けない。

| Object | 正本field／責務 | 変更規則 |
|---|---|---|
| `MaterialDefinitionV1` | Stable ID／revision、Domain、Shading Model、surface intent、typed graphまたは承認済みProject module ref、typed parameter declaration、texture／sampler role、render state、compile feature、`ShaderInterface` hash | Graphまたはcompile feature変更で新revision |
| `MaterialFunctionDefinitionV1` | Stable ID／revision、typed input／output、closed function graph、exact function dependency、Domain／Stage／Target制約、semantic content hash | Port契約、Graph、依存revision、制約変更で新revision。参照元は暗黙追従しない |
| `MaterialInstanceV1` | Stable ID／revision、parent Definition ref、optional parent Instance ref、Texture binding、公開parameter override、style override、compatibility constraint | Domain、Shading Model、Alpha Mode、Depth policy、compile feature、`ShaderInterface`を変更不可 |
| `MaterialTemplateV1` | Stable ID／revision、semantic role、既定Definition ref、公開parameter policy | `VisualStyleProfileV1`がroleごとに選択 |
| `MaterialSemanticCatalogV1` | role、意味、channel、互換性、例、qualification class | Engine build／Project extensionから生成 |
| `MaterialNodeCatalogV1` | Engine Nodeの型、意味、cost、Target、制約 | Engine buildから生成しAI／Project変更不可 |
| `ProjectShaderNodeCatalogV1` | Qualification済みProject Node／Shading Modelの型、意味、cost、Target、Module ref | Project Shader artifactから生成し、Sourceから直接編集不可 |
| `ArtAssetProfile` | Stable ID／revision、Mesh／Sprite／Texture制作規則、Palette、semantic role | Importer、Generator、Validatorが同一revisionを共有 |
| `AnimationPresentationProfile` | Stable ID／revision、presentation sampling、pose hold、motion accent | Simulation meaningから分離し表示だけを変更 |
| `VisualStyleProfileV1` | Material、Light、Camera、Post、VFX、UI、Asset制作規則のStyle契約 | `StyleChangeSet`／Preview／承認を必須とし、下記exact fieldsを使う |
| `VisualStyleSemanticRegistryV1` | Style axisごとのCore defaultとOwner-qualified Feature／Genre／Project contribution | dependency closureとQualificationから生成しSourceから直接編集不可 |
| `StyleCapabilityManifest` | `engine_build_id`、`target_profile_id`、`quality_profile_id`、利用可能なMaterial Domain／Shading Model／Node／Template、active Visual Style Registry／Projection ref、required Capability／Qualification ref | Engine buildとProject／Pack dependency closureから生成しAI／Project data変更不可 |
| `VisualStyleDecision` | 候補、除外理由、選択理由、未解決事項、Capability、domain result | Decision Ledger／Governance参照を持ち、authority／approvalは所有しない |
| `MaterialExplanationV1` | Material判断の根拠、差、cost、fallback | Preview／Plan revisionへ紐付け |

全Source objectはStable ID、schema version、content hash、revisionを持つ。Runtime packageへEditor node位置、UI state、AI prompt、自由形式の判断理由を持ち込まない。

`authoring_body.kind=typed_graph`のHuman向け主Authoring projectionにはnode-based Material Editorを採用する。ただし正本はCanvas pixel、Node位置、wire形状、display labelでなく、typed Node／Port／Edge、Stable ID、semantic ref、exact revisionである。AI、CLI、Editorは同じbounded graph queryとChangeSetを使い、AIがscreen coordinateまたはUI automationでGraph意味を推測しない。自由度はqualified Node catalog、Material Function、parameter／instance、[Project Shader](project-shader.md)へのtyped escape hatchで拡張し、generic code node、JIT、native handleでGraph contractを迂回しない。Unreal Material Editor、Unity Shader Graph、Godot VisualShaderはいずれもnode graphをMaterial／Shader Authoringへ使うため採用妥当性の比較Evidenceとするが、各製品のObject model、暗黙cast、custom code、update挙動は継承しない。[Unreal Material Editor](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-material-editor-user-guide)、[Unity Shader Graph 17.2](https://docs.unity3d.com/Packages/com.unity.shadergraph@17.2/manual/index.html)、[Godot 4.6 VisualShaders](https://docs.godotengine.org/en/4.6/tutorials/shaders/visual_shaders.html)

Material graphは`MaterialNodeCatalogV1`と`ProjectShaderNodeCatalogV1`のQualification済みentry、およびexact revisionへ固定した`MaterialFunctionDefinitionV1` callからなるclosed typed node family、typed edge、single domain outputを持つ。arbitrary command、file／network access、native include path、Backend pragmaをSource Documentへ埋め込まない。unknown／unqualified node、cycle、type mismatch、missing output、unbounded loop／resource、domain-incompatible node、stale Function refはCook前に拒否する。

`MaterialDefinitionV1`と`MaterialInstanceV1`のSource最小形を次に固定する。

```text
MaterialDefinitionV1
  schema_version: 1
  material_definition_id: StableId
  revision: positive uint64
  domain: MaterialDomainId
  shading_model: MaterialShadingModelId
  alpha_mode: opaque | mask | blend_premultiplied
  depth_policy: MaterialDepthPolicyV1
  surface_intent_refs[1..32]: MaterialSemanticRefV1
  authoring_body:
    kind: typed_graph | qualified_project_shader
    graph_ref: MaterialGraphRefV1
      | required only for typed_graph
    project_shader_module_ref: ProjectShaderModuleRefV1
      | required only for qualified_project_shader
  parameter_declarations[0..256]: MaterialParameterSemanticV1
  texture_role_declarations[0..64]: MaterialTextureRoleDeclarationV1
  render_state_intent: MaterialRenderStateIntentV1
  compile_features[0..64]: MaterialCompileFeatureId
  shader_interface_hash: SHA-256
  pbr_conformance_profile_ref: MaterialPbrConformanceProfileRefV1 | null
  target_support_refs[1..16]: MaterialTargetSupportRefV1
  content_hash: SHA-256

MaterialInstanceV1
  schema_version: 1
  material_instance_id: StableId
  revision: positive uint64
  parent_definition_ref:
    exact {material_definition_id, revision, content_hash}
  parent_instance_ref:
    optional exact {material_instance_id, revision, content_hash}
  parameter_overrides[0..256]: MaterialParameterOverrideV1
  texture_role_bindings[0..64]: MaterialTextureRoleBindingV1
  style_overrides[0..32]: MaterialStyleOverrideV1
  compatibility_constraint_refs[0..32]
  content_hash: SHA-256
```

`authoring_body`はtagと一致しないFieldをcanonical omissionする。`pbr_conformance_profile_ref`は`pbr_metal_rough`で必須、その他で`null`とする。Definition hashはASCII `MIRAKAN_MATERIAL_DEFINITION_V1`、Instance hashはASCII `MIRAKAN_MATERIAL_INSTANCE_V1`と、各自己hash Fieldだけを除くcount／presence／length-framed canonical bytesから計算する。Receipt、Qualification Binding、Cooked Artifact、Runtime Override、Renderer planをSource hashへ含めない。

`MaterialInstanceV1.parent_instance_ref`が存在する場合、参照先のultimate `parent_definition_ref`、Domain、Shading Model、Alpha Mode、Depth policy、compile feature、`ShaderInterface` hashは当該Instanceとbyte equalityでなければならない。親深度は自身を含め最大8、cycle、diamond／multiple parent、別Definitionへの横断を拒否する。Instanceは宣言済みoverrideだけを持ち、Cook時にroot Definitionから親順でcanonical flat parameter／texture setへ解決する。同じParameterまたはTexture roleを一revision内で複数回overrideせず、unknown、type／unit／color-space違い、range外、style lock違反をSource validationで拒否する。

`MaterialRuntimeOverrideSetV1`はSource Assetでも`MaterialInstanceV1`の子でもなく、Play／Render境界でだけ有効なephemeral projectionである。

```text
MaterialRuntimeOverrideSetV1
  schema_version: 1
  runtime_override_id: StableId
  material_instance_ref:
    exact {material_instance_id, revision, content_hash}
  artifact_generation_ref: CookedMaterialArtifactGenerationRefV1
  override_sequence: monotonic uint64
  lifetime_scope: frame | play_session | explicit_lease
  parameter_overrides[0..128]: MaterialParameterOverrideV1
  texture_role_overrides[0..16]: MaterialTextureRoleBindingV1
  source_event_or_command_ref: optional
  override_hash: SHA-256
```

Runtime OverrideはParameterでは`runtime_mutable=true`、Texture roleでは`override_scope=runtime_instance`の宣言だけを変更でき、compile-time Parameter、Domain、Shading Model、Alpha Mode、Depth policy、compile feature、resource layout、`ShaderInterface`を変更しない。Source Document、Project revision、Save、authoritative Replay、AI Commitへ書き戻さず、必要な再現はSource Eventまたはauthoritative Commandから同じoverride sequenceを再生成する。stale Instance／Artifact generation、lease終了、重複sequence、未宣言Texture role、型／range違反はoverride全体を拒否し、部分適用やRuntime shader compileを行わない。`override_hash`はASCII `MIRAKAN_MATERIAL_RUNTIME_OVERRIDE_SET_V1`と自己hash Fieldを除くcount／presence／length-framed canonical bytesから計算する。

### 2.1 Material Function／Subgraph

再利用可能なMaterial部分Graphの唯一のSource型を`MaterialFunctionDefinitionV1`とする。`Subgraph`はEditor表示名であり別Asset型、inline匿名Graph、文字列macro、copy-paste provenanceのないNode集合を正本にしない。Unreal EngineのMaterial FunctionとUnity Shader GraphのSub Graphは再利用性を確認する比較Evidenceに限り、各Engineの更新伝播、暗黙cast、serializationをMiraikanaiへ継承しない。[Unreal Material Functions](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-material-functions-overview)、[Unity Shader Graph Sub Graph](https://docs.unity3d.com/Packages/com.unity.shadergraph%4017.2/manual/Sub-graph.html)

```text
MaterialFunctionDefinitionV1
  schema_version: 1
  material_function_id: StableId
  revision: positive uint64
  input_ports[0..64]: MaterialFunctionPortDeclarationV1
  output_ports[1..32]: MaterialFunctionPortDeclarationV1
  function_graph: MaterialFunctionGraphV1
  function_dependency_refs[0..64]:
    exact {material_function_id, revision, content_hash}
  allowed_domain_ids[1..8]: MaterialDomainId
  allowed_stage_ids[1..8]: MaterialStageId
  required_capability_refs[0..32]
  target_support_refs[1..16]: MaterialTargetSupportRefV1
  semantic_content_hash: SHA-256
  content_hash: SHA-256

MaterialFunctionPortDeclarationV1
  port_id: StableId
  direction: input | output
  value_type: MaterialValueTypeV1
  unit_semantic_ref: optional exact semantic ref
  color_space_semantic_ref: optional exact semantic ref
  coordinate_space_semantic_ref: optional exact semantic ref
  value_semantic_ref: exact semantic ref
  default_value: optional MaterialTypedLiteralV1
```

Port集合は`port_id`のunsigned byte順、重複なしで、`direction`と格納先配列を一致させる。`default_value` Fieldは`direction=input`だけが持て、型、unit、color space、coordinate space、value semanticをPort宣言とexact一致させる。Function Graphはclosed Node／Port／Edge、exact一件のFunction Output Node、宣言outputとのset equalityを持ち、Material Domain output、Render state、Alpha Mode、Depth policy、compile feature、hidden Parameter、Runtime mutable stateを所有しない。Input defaultはPort宣言だけが所有し、呼出し側未接続時のBackend default、暗黙scalar／vector変換、linear／sRGB変換、normal／color変換を禁止する。

Function callはexact `{material_function_id, revision, content_hash}`と宣言Port Stable IDだけを参照する。`latest`、display name、Asset path、配列indexで再解決しない。Function依存GraphはDAG、展開深度最大16とし、direct／indirect recursion、dependency ref重複、依存先Port契約差、展開後のNode／resource／variant上限超過をSource validationで拒否する。Function改訂後も参照元は旧revisionを保持し、明示ChangeSetがdependency refを更新して影響するDefinition／Instance／variant／Target、Preview、recook結果を示した場合だけ移行する。単一Function保存による全参照元のsilent updateを行わない。

`semantic_content_hash`はASCII `MIRAKAN_MATERIAL_FUNCTION_SEMANTIC_V1`と、Port契約、Function Graphのsemantic Node／Edge、解決済み依存Functionの`semantic_content_hash`、Domain／Stage／Capability／Target制約をcount／presence／length-framed canonical bytesで束縛し、`material_function_id`、`revision`、`content_hash`、Editor view stateを除外する。`content_hash`はASCII `MIRAKAN_MATERIAL_FUNCTION_DEFINITION_V1`と、自己hash Field以外の全Fieldを同じcanonical byte規則で束縛する。Graph layout、comment、group、reroute、zoomはEditor-owned view stateとして別保存し、Function Definition、semantic hash、content hash、Cooked artifact identityへ含めない。CookerはFunctionを`MaterialIRV1`へinlineまたは共有IR単位としてlowerできるが、Function／call-site Stable ID、Source revision、origin mapを保持し、両方式が同じtyped IR意味、Diagnostic、artifact dependency closureを生成することをQualificationする。AI／Human／CLIは同じFunction dependency projectionを読み、Function内部を未取得のまま完全理解または安全な変更影響を主張しない。

## 3. Semantic CatalogとVisual Style

Material semantic vocabularyは物理値と表現intentを分離する。最低限、surface family、opacity behavior、normal response、roughness／specular intent、emissive intent、transmission／subsurface intent、two-sided intent、decal／overlay intentをclosed axesとして扱う。未知語を近い名前へ黙って補正せず、質問、候補、必要Capabilityを返す。

Visual Styleは特定shader graphのpresetではなく、Material、Light、Camera、Post、VFX、UI、Asset制作規則の整合を所有する。`VisualStyleProfileV1`の最低fieldを次に固定する。

```text
schema_version
profile_id
profile_version: positive uint32
profile_content_hash: SHA-256
source_profile_ref:
  optional exact {profile_id, profile_version, profile_content_hash}
display_name
compatible_world_space: WorldSpaceCompatibilityV1
art_direction_ref: VisualStyleSemanticRefV1(axis_id=art_direction)
composition_variant_ref: VisualStyleSemanticRefV1(axis_id=composition_variant)
visual_style_semantic_registry_ref:
  exact {registry_id, registry_version, registry_content_hash}
visual_style_semantic_activation_projection_ref:
  exact {projection_id, projection_version, projection_content_hash}
camera_profile_ref:
  exact {profile_id, profile_version, profile_content_hash}
lighting_profile_ref:
  exact {profile_id, profile_version, profile_content_hash}
post_process_profile_ref:
  exact {profile_id, profile_version, profile_content_hash}
vfx_style_profile_ref:
  exact {profile_id, profile_version, profile_content_hash}
ui_style_profile_ref:
  exact {profile_id, profile_version, profile_content_hash}
art_asset_profile_ref:
  exact {profile_id, profile_version, profile_content_hash}
animation_presentation_profile_ref:
  exact {profile_id, profile_version, profile_content_hash}
allowed_material_domains[]
allowed_shading_models[]
material_template_by_semantic_role{}:
  semantic role -> exact {template_id, template_version, template_content_hash}
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
toon_shading_profile_ref: optional ToonShadingProfileRefV1
outline_style_profile_ref: optional OutlineStyleProfileRefV1
palette_profile_ref:
  optional exact {profile_id, profile_version, profile_content_hash}
quality_profile_ref:
  exact {profile_id, profile_version, profile_content_hash}
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

```text
VisualStyleProfileRefV1
  profile_id: StableId
  profile_version: positive uint32
  profile_content_hash: SHA-256

PaletteTokenV1
  token_id: StableId
  token_version: positive uint32
  token_content_hash: SHA-256
  linear_srgb_rgba: {r: f32[0,64], g: f32[0,64], b: f32[0,64], a: f32[0,1]}
  allowed_usages: nonempty set<surface | fog_scattering | light | vfx | ui | outline>

PaletteTokenRefV1
  palette_profile_ref:
    exact {profile_id, profile_version, profile_content_hash}
  token_id: StableId
  token_version: positive uint32
  token_content_hash: SHA-256

VisualStyleSemanticRefV1
  axis_id: art_direction | composition_variant
  semantic_id: namespace付きStableId
  semantic_version: positive uint32
  semantic_content_hash: SHA-256

VisualStyleSemanticContributionV1
  semantic_ref: VisualStyleSemanticRefV1
  owner_ref: exact {owner_id, owner_revision, owner_content_hash}
  contribution_layer: core | feature_pack | genre_pack | project
  display_name
  semantic_description
  search_synonyms[]
  constraint_schema_ref
  required_profile_role_refs[]
  required_capability_refs[]
  target_support_refs[]

VisualStyleSemanticRegistryV1
  registry_id: registry.material.visual_style_semantic
  registry_version
  contributions[1..1024]
  registry_content_hash: SHA-256

VisualStyleSemanticActivationProjectionV1
  projection_id
  projection_version: positive uint32
  registry_ref: exact {registry_id, registry_version, registry_content_hash}
  selected_semantic_refs[1..1024]: VisualStyleSemanticRefV1
  qualification_binding_refs[1..1024]:
    exact {binding_id, binding_version, binding_content_hash}
  projection_content_hash: SHA-256
```

`semantic_content_hash`はASCII `MIRAKAN_VISUAL_STYLE_SEMANTIC_CONTRIBUTION_V1`と自己hashを除くReceipt-free Contribution canonical bytes、Registry hashはASCII `MIRAKAN_VISUAL_STYLE_SEMANTIC_REGISTRY_V1`、Registry ID／version、axis／semantic ID／version順の全Contribution canonical bytesから計算する。selected refsはaxis／semantic ID／version／hash順、Binding refsは解決したsubject refの同じ順にstrict sortし、duplicateを拒否する。Activation Projectionのselected ref集合とQualification Bindingが解決する合格かつfreshなsubject集合はexact set equalityで、Receipt／BindingをContribution／Registry hashへ戻さない。Feature／Genre contributionは所有Pack identity、Project contributionはProject owner identityへexact解決する。各Ownerは自己namespaceだけへ追加でき、Core entryの上書き、同一logical IDの別hash、unknown、stale owner／version／hash、未Qualification、Target／Capability不成立をfail closedにする。Registry materializationはProject／Pack dependency closureを解決したCompilerが行い、Generic Engine CoreからPackへのdependency edgeを生成しない。

Engine同梱Profileはimmutable templateである。Project変更時は全fieldを解決した派生Profileを新規作成し、Runtime inheritance、複数親、自動伝播chainを持たない。Rendering／Physics／Navigationの実行空間は[World](world.md)のexact `WorldSpaceProfileRefV1`だけが選択する。`VisualStyleProfileV1`は`compatible_world_space`によって使用可能範囲を宣言できるが、scene dimensionまたはhybrid gameplay authorityを所有・推測・変更しない。表現語彙である`art_direction_ref`と`composition_variant_ref`だけをRegistryで拡張し、同じactive Registry／Activation Projectionへexact解決する。全Profile／Template refは三Fieldのbyte equalityで解決し、ID-only、`latest`、display name、別Ownerの同名Profileを受理しない。`VisualStyleProfileRefV1`は解決先Profileの三Fieldとbyte equalityにする。`PaletteTokenRefV1`は解決先Palette Profileと`PaletteTokenV1`のID／version／hashへ全Fieldでexact解決し、Token表示名、色値一致、配列順から代替Tokenを推測しない。Token hashはASCII `MIRAKAN_PALETTE_TOKEN_V1`と自己hashを除く全Fieldのlength-framed canonical bytesから計算する。Consumerは自己usageが`allowed_usages`に含まれることと自己Field rangeを再検証し、Token側rangeを理由に緩和しない。`profile_content_hash`はASCII `MIRAKAN_VISUAL_STYLE_PROFILE_V1`と自己hash Fieldだけを除く全Fieldのcount／presence／length-framed canonical bytesから計算し、Activation Projection、Qualification Receipt、Preview、派生Renderer planをpreimageへ含めない。

### 3.1 Toon／Outline semantic contract

ToonとOutlineは特定のshader text、Render pass、native GPU objectではなく、Materialが所有するtyped style intentである。`VisualStyleProfile.toon_shading_profile_ref`は選択されたTemplateが`toon_surface`、`sprite_toon`、`hybrid_sprite_toon`のいずれかを使うとき必須とし、`outline_style_profile_ref`はuntyped IDを使わないexact refとする。LightingはToon responseを消費し、[Render Graph](render-graph.md)はOutline intentをqualified execution techniqueへ解決するが、両者は下記parameterの複製所有者にならない。

```text
ToonRampRefV1
  ramp_id: StableId
  ramp_version: positive uint32
  source_asset_ref: exact {asset_id, asset_revision, source_sha256}
  input_signal: diffuse_ndotl_clamped | specular_lobe | shadow_attenuation
  dimension: one_d
  sample_filter: point | linear
  address_mode: clamp
  channel_semantic: scalar_multiplier | linear_rgb_multiplier
  color_space: linear_rgb
  ramp_content_hash: SHA-256

ToonBandResponseV1
  source: analytic_bands | ramp_asset
  input_signal: diffuse_ndotl_clamped | specular_lobe | shadow_attenuation
  band_count: integer in [1, 8] | required only for analytic_bands
  thresholds[0..7]: strictly increasing finite values in [0, 1]
    | count = band_count - 1 for analytic_bands
  softness[1..8]: finite values in [0, 1]
    | count = band_count for analytic_bands
  ramp_ref: ToonRampRefV1 | required only for ramp_asset

ToonSpecularResponseV1
  mode: disabled | analytic_banded | ramp_asset | anisotropic_analytic_banded
  analytic_band_response: ToonBandResponseV1
    | required only for analytic_banded or anisotropic_analytic_banded
  ramp_ref: ToonRampRefV1 | required only for ramp_asset
  roughness_range: closed finite range within [0.045, 1]
  anisotropy_range: closed finite range within [-1, 1]
    | required only for anisotropic_analytic_banded

ToonRimResponseV1
  mode: disabled | lit_side | shadow_side | view_fresnel
  width: finite value in [0, 1]
  softness: finite value in [0, 1]
  intensity: finite nonnegative value

ToonShadowResponseV1
  receive_mode: continuous | banded | profile_ramp
  self_shadow_extinction: finite value in [0, 1]
  cast_shadow: bool
  analytic_band_response: ToonBandResponseV1 | required only for banded
  ramp_ref: ToonRampRefV1 | required only for profile_ramp

ToonEnergyPolicyV1
  kind: stylized_bounded | physically_bounded
  max_direct_lighting_multiplier: finite value in (0, 16]
  max_indirect_lighting_multiplier: finite value in (0, 16]

ToonFeatureSemanticV1
  role: generic | face | hair | eye | cloth | foliage
  normal_source: mesh_normal | authored_normal_map | bent_normal_map | role_profile
  shadow_source: engine_shadow | authored_mask | signed_distance_field | role_profile
  role_profile_ref: ToonFeatureRoleProfileRefV1
    | required only when either source is role_profile

ToonFeatureRoleProfileRefV1
  profile_id: StableId
  profile_version: positive uint32
  profile_content_hash: SHA-256

ToonFeatureRoleProfileV1
  profile_id: StableId
  profile_version: positive uint32
  role: generic | face | hair | eye | cloth | foliage
  normal_source: mesh_normal | authored_normal_map | bent_normal_map
  normal_texture_role: none | normal | bent_normal
  shadow_source: engine_shadow | authored_mask | signed_distance_field
  shadow_texture_role: none | shadow_mask | signed_distance_field
  compatible_material_domains[1..8]
  compatible_world_space: WorldSpaceCompatibilityV1
  required_capability_refs[0..16]: McdContractRefV1(kind=capability)
  profile_content_hash: SHA-256

ToonShadingProfileRefV1
  profile_id: StableId
  profile_version: positive uint32
  profile_content_hash: SHA-256

ToonShadingProfileV1
  profile_id: StableId
  profile_version: positive uint32
  compatible_material_domains[1..8]
  compatible_world_space: WorldSpaceCompatibilityV1
  diffuse_response: ToonBandResponseV1
  specular_response: ToonSpecularResponseV1
  rim_response: ToonRimResponseV1
  shadow_response: ToonShadowResponseV1
  feature_semantics[0..6]: ToonFeatureSemanticV1
  energy_policy: ToonEnergyPolicyV1
  required_capability_refs[0..32]: McdContractRefV1(kind=capability)
  fallback_profile_refs[0..8]: ToonShadingProfileRefV1
  profile_content_hash: SHA-256
```

Rampはlinear RGBの1D data textureであり、入力signal、filter、clamp、channel解釈をfilenameや画像dimensionから推測しない。`ToonBandResponseV1.source=analytic_bands`はanalytic fieldをrequiredかつ`ramp_ref`をcanonical omission、`ramp_asset`はanalytic fieldをcanonical omissionかつ`ramp_ref`をrequiredにする。analytic diffuseは`diffuse_ndotl_clamped`、analytic specularは`specular_lobe`、banded shadowは`shadow_attenuation`だけを受理し、rampも同じsignalでなければならない。`ToonSpecularResponseV1.mode=disabled`はresponse fieldのcanonical omissionとzero contributionを必須にする。analytic modeは`analytic_band_response.source=analytic_bands`、ramp modeは`ramp_ref.input_signal=specular_lobe`を必須とする。Shadowの`continuous`はresponse fieldをomission、`banded`はanalytic response、`profile_ramp`は`shadow_attenuation` rampを必須にする。Rim disabledはwidth／softness／intensityをcanonical zeroにする。

`ToonFeatureSemanticV1`のいずれかのsourceが`role_profile`である場合、`role_profile_ref`は同じrole、Material Domain、exact World Profileにcompatibleな`ToonFeatureRoleProfileV1`へexact解決する。`role_profile`側だけが参照先の対応するnormalまたはshadow source／texture roleへ展開し、もう一方のnon-`role_profile` sourceを上書きしない。`mesh_normal`／`engine_shadow`は対応texture roleを`none`、`authored_normal_map`／`bent_normal_map`はそれぞれ`normal`／`bent_normal`、`authored_mask`／`signed_distance_field`はそれぞれ`shadow_mask`／`signed_distance_field`を必須にする。texture roleはMaterial Instanceのtyped bindingへexact解決し、Asset名、channel名、filenameから補完しない。`ToonFeatureRoleProfileV1.profile_content_hash`はASCII `MIRAKAN_TOON_FEATURE_ROLE_PROFILE_V1`と自己Fieldを除くreceipt-free canonical bytesを`uint32_be` length framingしてSHA-256する。

`physically_bounded`のdirect／indirect上限はともに1以下、`stylized_bounded`でも16以下である。全fixtureはregistered sweep domain上でnon-negative finite responseとこの上限を検査する。feature roleはuniqueであり、face／hair動作をAsset名またはtexture filenameから推測しない。fallbackはstrict priority、unique、transitively acyclicで、最初のcompatibleかつTarget-qualified refだけを選ぶ。候補がなければ`capability_unavailable`としてfailし、別styleへsilent substitutionしない。`profile_content_hash`はASCII `MIRAKAN_TOON_SHADING_PROFILE_V1`と自己Fieldを除くreceipt-free canonical bytesを`uint32_be` length framingしてSHA-256する。`ramp_content_hash`は同じ規約でdomain separatorを`MIRAKAN_TOON_RAMP_REF_V1`に置き換えて計算する。

```text
OutlineStyleProfileRefV1
  profile_id: StableId
  profile_version: positive uint32
  profile_content_hash: SHA-256

OutlineStyleProfileV1
  profile_id: StableId
  profile_version: positive uint32
  compatible_world_space: WorldSpaceCompatibilityV1
  technique_preference:
    geometry_only | screen_space_only | hybrid_qualified | disabled
  geometry_width_semantic: none | object_relative | world_meters
  geometry_width_value: finite nonnegative value
  screen_width_pixels: finite nonnegative value
  depth_threshold: finite nonnegative value
  normal_threshold: finite value in [0, 1]
  color: LinearColor4f
  occlusion_policy: visible_only | silhouette_and_crease | include_occluded
  temporal_policy: no_history | stable_history_required
  alpha_policy: respect_coverage | opaque_silhouette_only
  required_capability_refs[0..16]: McdContractRefV1(kind=capability)
  fallback_profile_refs[0..8]: OutlineStyleProfileRefV1
  profile_content_hash: SHA-256
```

Outline Profileはstyle intentだけを持つ。`compatible_world_space`は再利用可能Profileの互換性だけを宣言し、scene dimensionまたはhybrid gameplay authorityを選択しない。Resolverはexact World Profileがこのconstraintを満たす場合だけ候補を評価する。`geometry_only`はnon-`none` geometry width、zero screen width／depth threshold／normal threshold、`no_history`を必須にする。`screen_space_only`は`geometry_width_semantic=none`、positive screen width、nonzero depthまたはnormal thresholdを、`hybrid_qualified`はnon-`none` geometry widthとpositive screen widthを、`disabled`は全width／threshold zero、`no_history`、empty Capability／fallback setを必須にする。fallbackはstrict priority、unique、acyclic、Target-qualifiedである。`profile_content_hash`はASCII `MIRAKAN_OUTLINE_STYLE_PROFILE_V1`と自己Fieldを除くreceipt-free canonical bytesを`uint32_be` length framingしてSHA-256する。ProfileはRender pass、resource、Backend objectを作らない。

初期Core defaultは`style.art_direction.realistic@1 | style.art_direction.toon@1 | style.art_direction.pixel_2d@1 | style.art_direction.pixel_diorama@1`と`style.composition.native@1 | style.composition.crisp_sprite_over_high_res_3d@1 | style.composition.unified_low_resolution@1`のexact七entryであり、従来Profileの意味とfixtureを維持する開始値であってclosed上限ではない。Feature／Genre／Projectはwatercolor、voxel、technical visualization等のqualified semantic entryを追加できるが、名前だけでShading Model、Render pass、Camera、Post、VFX、UIを生成せず、Contributionのtyped constraintと`required_profile_role_refs[]`へ完全解決する。unknown／unqualified ref、axis mismatch、0件または複数の意味同等候補はBlocking questionまたはtyped rejectにし、`realistic`、`native`、近い表示名へ黙ってfallbackしない。

旧open field `composition`と旧scalar field `art_direction`／`composition_variant`、`scene_dimension`、`gameplay_space`、`outline_profile_id`は新Profileで受理しない。Visual Styleの表現方向とpresentation composition方式はqualified exact semantic refだけが所有し、Worldの構造的空間選択はWorld Profileに委譲する。自由文字列、別名field、未登録variantを拒否する。実在する旧bytesを移行する場合はsource schema bytes／Owner／Named Algorithm／immutable fixtureを束縛した別の承認済みschema migrationを先にactivateし、旧valueまたは表示名だけで自動変換しない。

`art_direction_ref = style.art_direction.pixel_2d@1`または`composition_variant_ref = style.composition.unified_low_resolution@1`の場合は正整数`reference_resolution`、1以上の`pixels_per_unit`、`not_applicable`以外のinteger scale policyを必須とする。`composition_variant_ref = style.composition.crisp_sprite_over_high_res_3d@1`は`reference_resolution = null`としてCamera Profileの出力解像度を使う。`world_texel_density`は`art_direction_ref = style.art_direction.pixel_diorama@1`でだけ必須で、`min_screen_pixel_ratio <= max_screen_pixel_ratio`、既定0.8～1.2とする。Camera変更時に`reference_distance_m`を暗黙更新しない。追加Contributionは同じ制約を名前から継承せず、自身の`constraint_schema_ref`で必要field、互換性、fallback禁止条件を宣言する。

`style_critical_fields`と`locked_fields`はProfile内JSON Pointerである。`fallback_policy = allow_listed`は1件以上のfallback、`forbid`は空配列を必須とする。`license_or_usage_basis`は`user_owned | licensed | public_domain | reference_only`に固定する。利用根拠のないreference AssetをCommitしない。

上記Registry、Projection、Core default／extension fixtureはtarget contractであり、実装済み、active Gateway surface、Production Qualification済みという主張ではない。

AI intent resolutionはsource request identity、Project revision、Catalog revision、resolved Material／Style refs、assumption／question、compatibility resultを束ねる。共通envelope fieldやhash表現は[Executable contracts](../02-foundation/executable-contracts.md)を参照し、本書はMaterial固有payloadの意味だけを決める。

### 3.2 `MaterialSemanticCatalogV1`

```text
MaterialSemanticCatalogV1
  schema_version
  catalog_revision
  engine_build_id
  target_profile_ids[]
  quality_profile_ids[]
  entries[]
```

`MaterialSemanticEntryV1`は`semantic_role_id: StableEnumId`、`display_name`、`description`、`search_synonyms[]`、`positive_examples[]`、`counter_examples[]`、`allowed_domains[]`、`allowed_shading_models[]`、`required_texture_channels[]`、`optional_texture_channels[]`、`parameter_semantics[]`、`compatible_style_profile_ids[]`、`required_capabilities[]`、`target_support[]`、`default_template_id`、`allowed_fallback_template_ids[]`、`preview_fixture_ids[]`、`production_maturity`を持つ。`production_maturity`の値とfeature割当は[Product Plan](../00-product/product-plan.md)を参照し本書へ複写しない。

初期role setは`surface.generic | surface.terrain | surface.foliage | surface.water | surface.snow | character.skin | character.hair | character.eye | character.cloth | prop.opaque | prop.transparent | sprite.actor | sprite.environment | sprite.effect | decal.surface | vfx.particle | ui.surface`である。[Environment／Water／Weather／Snow](environment-surfaces.md)が参照するWater／Snow surfaceの正規IDはそれぞれ`surface.water`／`surface.snow`であり、`water_surface`、`snow_surface`、Asset名による別名を契約IDとして受理しない。synonym／example／counter-exampleは検索用で正規IDではなく、Asset名や作品名だけでroleを確定しない。Project roleはEngine roleを上書きせず`project.<project_id>.*` namespaceへ追加する。

### 3.3 `MaterialParameterSemanticV1`

`parameter_id: StableId`、`semantic_id: StableEnumId`、`value_type`、`unit`、`color_space: none | linear_rgb | srgb | data`、`valid_range`、`default_value`、`default_provenance`、`compile_time: bool`、`override_scope: definition_only | source_instance | runtime_instance`、`batching_class: uniform_across_batch | per_render_instance`、`ai_mutable: bool`、`style_critical: bool`、`runtime_mutable: bool`を持つ。AIはexact parameter ID／typeを使い、表示名／shader variable名で変更しない。

`override_scope`は最大許可範囲であり、`definition_only`はDefinition内だけ、`source_instance`はDefinitionとSource Instance、`runtime_instance`はDefinition、Source Instance、Runtime Overrideでの変更を許可する。`compile_time=true`は`override_scope=definition_only`、`batching_class=uniform_across_batch`、`runtime_mutable=false`を必須にする。`override_scope=runtime_instance`は`compile_time=false`かつ`runtime_mutable=true`、それ以外は`runtime_mutable=false`とする。`batching_class=uniform_across_batch`の値差は別`MaterialBatchCompatibilityKeyV1`を生成し、`per_render_instance`の値差だけをCook済みper-instance layoutへ格納できる。どちらもParameter名、値の近似、Backend capabilityから動的にclassを変更しない。

`MaterialTextureRoleDeclarationV1`はrole ID、texture class、dimension、sample type、color-space、sampler policy、required／optional、`override_scope: definition_only | source_instance | runtime_instance`、`batching_class: uniform_across_batch | per_render_instance_indexed`、Target support、fallback refを持つ。Parameterと同じ最大許可範囲を使い、`runtime_instance`以外のTexture roleをRuntime Overrideから変更しない。`per_render_instance_indexed`はTarget-qualifiedなCooked resource indexing layoutが存在する場合だけ有効で、native descriptor indexをSource、Instance、AI Contextへ公開しない。

### 3.4 `MaterialNodeCatalogV1`

各Engine Node entryは`node_type_id`、`schema_version`、exact `owner_ref`、`display_name`、`semantic_description`、`input_ports[]`、`output_ports[]`、`allowed_domains[]`、`allowed_stages[]`、`required_capabilities[]`、`target_support[]`、`static_cost_estimate`、`resource_cost`、`determinism_class`、`failure_codes[]`、`examples[]`、`counter_examples[]`、self-excluding `node_content_hash`を持つReceipt-free base recordである。Project entryは同じbase projectionに`project_shader_module_ref`と`export_id`だけを追加する。Project Nodeの利用可否は次のroot外projectionが所有する。

```text
MaterialProjectNodeActivationEntryV1
  target_support_ref: exact ShaderTargetSupportRefV1
  project_shader_activation_binding_ref:
    ProjectShaderActivationBindingRefV1

MaterialProjectNodeActivationProjectionV1
  projection_id/version
  node_ref: exact {node_type_id, schema_version, node_content_hash}
  project_shader_module_ref: ProjectShaderModuleRefV1
  entries[1..128]: MaterialProjectNodeActivationEntryV1
  projection_hash: SHA-256
```

Project Node `owner_ref`と`project_shader_module_ref`が解決するModule ownerはbyte equalityで、NodeのModule ref／export IDは同じModule／public exportへexact解決する。各entryのProject Shader BindingはProjectionと同じModule、entryと同じTarget Support、同じownerを解決し、そのsigned Receipt subjectも同じowner／Module／Target tupleを持たなければならない。entry集合はProject Node `target_support[]`の`required`集合とexact set equality、explicitにactivateする`optional`は0または1 entry、`unsupported`は0 entryとする。entriesはTarget Profile ID／support hash／Binding ID／version／hash順へstrict sortし、duplicate Target／Bindingとsame Targetへの複数Bindingを拒否する。`projection_hash`はASCII `MIRAKAN_MATERIAL_PROJECT_NODE_ACTIVATION_PROJECTION_V1`と自己Fieldだけを除くcount／length-framed canonical bytesから計算する。Node／Module／owner／Target／Bindingのstaleまたはsubstitution、required entry missing／extra、duplicate／順序違反を各一原因fixtureで拒否する。Qualification Receipt／Binding／ProjectionをNode／Catalog／Module hashへ戻さない。意味とSource境界、Module Qualification DAGは[Project Shader](project-shader.md)が所有する。Portはtypedで、implicit scalar／vector expansion、linear／sRGB混在、normal／color混在を許可しない。Graph layoutはsemantic hashに含めない。

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

### 4.1 canonical PBRとglTF mapping

`realistic_basic`／`realistic_advanced`は`pbr_metal_rough` Shading Modelのcompile feature tierを表すclosed IDであり、Material Domain、Shading Model、`VisualStyleProfileV1`の別IDではない。`realistic_basic`は`pbr_metal_rough`の必須基底tier、`realistic_advanced`は`realistic_basic`を包含するoptional上位tierである。tierのactivationと導入順は[Product Plan](../00-product/product-plan.md)が決定する。

PBRの意味、Target差、Reference比較を一つに束ねるSource側契約を次に固定する。

```text
MaterialPbrConformanceProfileRefV1
  profile_id: StableId
  profile_version: positive uint32
  profile_content_hash: SHA-256

MaterialPbrConformanceProfileV1
  schema_version: 1
  profile_id: StableId
  profile_version: positive uint32
  feature_tier: realistic_basic | realistic_advanced
  gltf_core_spec_ref:
    exact {spec_id, spec_version, source_commit_sha, specification_closure_hash}
  material_model: gltf2_metallic_roughness
  roughness_mapping: alpha_equals_perceptual_roughness_squared
  dielectric_ior_default: 1.5
  tangent_basis_policy:
    provided: gltf_vec4_xyz_normalized_w_handedness
    missing: mikktspace_from_position_vertex_normal_normal_texture_uv
    bitangent: cross_normal_tangent_times_handedness
  normal_texture_space: tangent_space
  color_encoding_profile_ref:
    exact {profile_id, profile_version, profile_content_hash}
  ibl_sampling_contract_ref:
    exact {contract_id, contract_version, contract_content_hash}
  supported_extension_spec_refs[0..32]:
    exact {extension_id, source_commit_sha, status_at_adoption=ratified, specification_closure_hash}
  target_support_refs[1..16]: MaterialTargetSupportRefV1
  reference_fixture_refs[1..256]
  analytic_invariant_profile_ref:
    exact {profile_id, profile_version, profile_content_hash}
  image_tolerance_profile_ref:
    exact {profile_id, profile_version, profile_content_hash}
  profile_content_hash: SHA-256
```

`feature_tier=realistic_basic`はcore Metallic-Roughnessと本節のPBR-compatible basic extensionだけ、`realistic_advanced`はbasic全件と本節のadvanced extension subsetを持つ。`KHR_materials_unlit`はImporterが別`unlit_surface` Definitionへ変換する分岐であり、PBR Profileの`supported_extension_spec_refs[]`に入れない。`gltf_core_spec_ref`はKhronos glTF repositoryの採用commitにあるcore 2.0 normative source closureを、各extension refは同repositoryの採用commitにあるextension README／Schemaのnormative source closureをSHA-256まで固定する。`status_at_adoption=ratified`は同commitのExtension Registryとextension statusの両方がRatifiedを示す場合だけ有効で、後日のstatus変化をProfileへ暗黙反映しない。配列は本節でProfile tierへ選択したratified extensionのexact subsetであり、Khronos repositoryの可変`main`、公開時点の全extension、Release Candidate、将来ratified entryを暗黙追加しない。Sourceに現れないextensionを自動有効化せず、Target非対応extensionを名前の近いlobeへ変換しない。全Profile／Contract／Specification refはID／version／commit SHA／closure hashのbyte equalityで解決する。PBR Profile refはDefinitionのSource hashへ入るが、Qualification Receiptは入れない。`profile_content_hash`はASCII `MIRAKAN_MATERIAL_PBR_CONFORMANCE_PROFILE_V1`と自己hash Fieldだけを除くcount／presence／length-framed canonical bytesから計算する。

glTF `TANGENT`はXYZ normalized、Wを±1のhandednessとして保持し、bitangentを`cross(normal, tangent.xyz) * tangent.w`で得る。Tangent欠落時だけ、normal textureが使うUV、position、normalからMikkTSpaceを生成する。generatorのexact version／artifact hashは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)へ固定し、Importerごとの別algorithm、filename、DCC名から推測しない。normal欠落時はglTF規則に従うflat normal生成とprovided tangent無効化を同じConversion Reportへ記録する。Tangent／normal生成結果、color encoding、IBL prefilter／BRDF integration artifact、Target Profileの一つでも異なる場合は別PBR input closureとして再Qualificationする。

`realistic_basic`はScene-linear HDR、物理単位Light、IBLを入力とし、Cook-Torrance BRDFをGGX normal distribution、Smith height-correlated visibility、Schlick Fresnelで評価する。Metallic-Roughness workflowのcanonical入力はbase color、metallic、perceptual roughness、tangent-space normal、occlusion、emissiveである。Authoring／Importの`perceptual_roughness_source`は0～1の値をそのまま保持する。glTF metallic-roughness BRDFの意味は`alpha=roughness^2`であり、Miraikanaiの評価時だけ`perceptual_roughness_eval=max(perceptual_roughness_source, 0.045)`、`alpha=perceptual_roughness_eval^2`とする。0.045はTarget別analytic／image Qualificationで確定するまでprovisionalなMiraikanai固有のFP32数値安定化PolicyであってglTF要件ではなく、Source、Import Report、Material Instance値をclampまたは上書きしない。Opaque、alpha mask、premultiplied transparent、Environment IBL reflection、shadow、height fog、exposure、tone mappingを同じ意味契約で扱う。

glTF coreと基本extensionは次のMaterials-owned mappingへ固定する。

| glTF入力 | canonical Material意味 |
|---|---|
| core base color | Texture RGBをsRGBからscene-linear base colorへdecodeし、Aをopacityとする |
| core metallic-roughness | data textureのGをperceptual roughness、Bをmetallicとし、R／Aを意味入力に使わない |
| core normal／occlusion／emissive | normal RGBをtangent-space data、occlusion Rをlinear occlusion、emissive RGBをsRGBからscene-linear emissiveへdecodeする |
| core `alphaMode`／`alphaCutoff` | `OPAQUE`／`MASK`／`BLEND`を`AlphaMode`の`opaque`／`mask`／`blend_premultiplied`へ写像し、`alphaCutoff`は`mask`のcoverage閾値parameterとして保持する。`BLEND`のstraight alphaはCook時にcanonical変換としてpremultiplied colorへ変換し、Source texture pixelへ破壊的bakeしない |
| core `doubleSided` | render-stateのcull intent（two-sided intent）として保持し、shading意味を変更しない |
| `KHR_materials_unlit` | `unlit_surface`へ写像し、Light／IBL BRDFを通さずbase color、opacity、emissiveを保持する |
| `KHR_materials_emissive_strength` | emissive factor／textureへ掛ける非負のscene-linear emission倍率として保持する |
| `KHR_texture_transform` | 対象texture bindingのUV offset、rotation、scale、texCoord選択として保持し、画像pixelへ破壊的bakeしない |

`realistic_advanced`は`realistic_basic`を意味的基底とし、Local reflection probe、Skin／Cloth／Hair／Eye／Foliage template、Parallax occlusion mappingに加えて次のclosed mappingを持つ。

| glTF extension | canonical advanced feature |
|---|---|
| `KHR_materials_clearcoat` | base BRDF上のdielectric coat factor、coat roughness、coat normal |
| `KHR_materials_sheen` | microfiber sheen colorとsheen roughness |
| `KHR_materials_specular` | dielectric Fresnel F0のstrength／color。metallicへ暗黙変換しない |
| `KHR_materials_ior` | dielectric IORとF0の関係 |
| `KHR_materials_transmission`＋`KHR_materials_volume` | surface transmissionとvolume thickness／attenuation color／distanceを一つの透過意味として保持 |
| `KHR_materials_anisotropy` | anisotropy strengthとtangent-space direction／rotation |
| `KHR_materials_iridescence` | thin-film factor、IOR、thickness range |
| `KHR_materials_dispersion` | transmissive mediumのwavelength dispersion strength |
| `KHR_materials_variants` | Sourceのprimitive／Material variant対応をcanonical `MaterialVariantSet`へ変換 |

`MaterialVariantSet`はSource variant identity、対象primitive／Material slot、対応するMaterial Definition／Instance Stable IDを保持する決定論的変換結果であり、display name、配列index、Importer内部pointerをidentityにしない。Reimportでは同じSource variant identityを維持し、missing／ambiguous mappingを推測で別Materialへ接続しない。

未知extension、上記mappingを満たせないextension、Targetで未対応のadvanced featureはextension IDを付けた`MIRAKAN-MATERIAL-CAPABILITY_NOT_ACTIVATED` hard failureとしてImportとArtifact promotionを停止する。Governanceで承認されたfallbackがある場合も`MIRAKAN-MATERIAL-FALLBACK_REQUIRED`の別Previewとして提示し、元Importを成功扱いせず、extension／channel／lobeを黙って破棄、近似、downgradeしない。

glTF Parser／Importerのexact version、artifact hash、license、取得元は[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)の`toolchain.lock.json`だけが所有する。Source transport、sandbox Job、typed IR、Conversion Report、Receipt、transaction／promotionは[Asset lifecycle](../03-authoring/asset-lifecycle.md)の`AssetImportJob`、`AssetImportPlanV1`、`SceneImportIRV1`、`AssetConversionReportV1`、`AssetImportReceiptV1`だけが所有する。これらはMaterialsのchannel／BRDF／extension／variant意味を変更しない。

### 4.2 Decal source／command／packet境界

Decalの唯一のSourceは`DecalDefinitionV1`である。`DecalSpawnCommandV1`はRuntimeがSourceまたはauthoritative Eventから発行する要求、`DecalPacketV1`はRender Snapshotへ渡すpresentation recordであり、いずれもGameplay stateではない。Decal Materialは本書の`decal` Domainと閉じたreceiver channelを使い、Render Graphのpass、queue、native resourceを指定しない。

```text
DecalDefinitionV1
  decal_id
  material_instance_ref
  projection_extents_m
  receiver_channel_mask
  surface_normal_limit_rad
  blend_mode: color | normal | roughness | emissive
  sort_layer
  lifetime_policy: authored_static | timed
  lifetime_seconds
  fade_seconds
  budget_class: critical_feedback | gameplay_feedback | ambient
  target_fallback_ref

DecalSpawnCommandV1
  command_id
  decal_definition_ref
  source_revision
  source_event_ref: optional
  world_transform
  runtime_spawn_sequence
  target_profile_ref

DecalPacketV1
  decal_id
  definition_revision
  world_transform
  material_instance_ref
  receiver_channel_mask
  blend_mode
  sort_key
  lifetime_state
  fallback_ref
```

`projection_extents_m`の各軸はfiniteな0.001～16 m、`sort_layer`は0～31とする。`timed`は`lifetime_seconds`を0.016～120 s、`fade_seconds`を0～lifetimeに必須とし、`authored_static`は両値を0に固定して不要fieldの任意値を許可しない。receiverはMaterial／Renderableのclosed channel maskで選ぶ。projection volume内であってもUI、transparent、Particle、Character、terrainへ暗黙投影しない。

C1はForward+のopaque／masked receiverだけへcolor、normal、roughness、emissiveを適用する。transparent receiver、animated atlas、deferred-only DBufferはC2 Capabilityであり、Mobileを含む未対応Targetは`target_fallback_ref`でQualification済みのmesh／particle／UI presentationへ明示的にfallbackする。同一pixelの採択順は`sort_layer`、budget priority、`decal_id`、`runtime_spawn_sequence`の順にcanonicalである。stale `source_revision`、invalid／unsupported receiver、未qualified fallback、capacity超過はそれぞれtyped rejectとし、別Material、receiver、またはsilent dropへ置換しない。

`authored_static`だけがowner World／Scene SourceとSave互換identityを持つ。impact等の`timed` DecalはHit／Interaction等のauthoritative Eventから再生成するpresentationであり、Damage、collision、visibility、surface friction、Gameplay state、SaveのいずれもDecalのresidencyまたはrender resultに依存しない。Replayも元Eventから再生成し、Decalの有無でauthoritative state hashを変化させない。

C1 reference Profileはactive 2,048、spawn 128／presentation update、visible 512／viewをhard boundとする。ここでpresentation updateはMaterial presentation queueのconsume boundaryであり、Simulation Advance、render frame、wall-clock秒へ暗黙変換しない。超過時は`ambient`、`gameplay_feedback`の順にcanonical evictionしてDiagnosticを残す。`critical_feedback`は黙って欠落させず、Target Profileのfallbackへ切替えるか`MIRAKAN-MATERIAL-DECAL_CRITICAL_FALLBACK_REQUIRED`で拒否する。その他のclosed failureは`MIRAKAN-MATERIAL-DECAL_SCHEMA_INVALID | MIRAKAN-MATERIAL-DECAL_STALE_COMMAND | MIRAKAN-MATERIAL-DECAL_RECEIVER_UNSUPPORTED | MIRAKAN-MATERIAL-DECAL_CAPACITY_EXCEEDED`とする。

## 5. 表現Profileとparameter binding

標準表現Profile候補はProjectのVisual Styleを再利用可能なnamed profileへ固定し、Material assetへの一括上書きではなくresolver inputとして参照する。Profileは対象Domain、required semantic axes、forbidden combination、fallback profile、qualification refを持つ。

Parameterはscalar、vector、color、texture role、enum、booleanのclosed typeとunit／color-space semantics、range、default、override policyを持つ。Runtime bindingはCook済みlayoutのStable Parameter IDを使い、文字列名、pointer、descriptor indexによるdispatchを禁止する。unknown parameter、type mismatch、stale layout generationはhard errorとする。

Texture roleはbase color、normal、mask、emissive、detail等の意味を表し、asset formatやchannel packingはCooked Artifactへ閉じる。Source revisionを跨ぐbinding混在を避け、Material、texture、shader artifactのgeneration closureを一つのpromotion単位として検証する。

RendererへのMaterial frame入力の正式名は`ResolvedMaterialBindingV1`である。[Render Graph](render-graph.md) §2はこのobjectだけをMaterial入力として受け、Material parameter名やtexture handleを再解釈しない。

```text
CookedMaterialArtifactGenerationRefV1
  artifact_id: StableId
  artifact_generation: positive uint64
  artifact_content_hash: SHA-256
  target_profile_ref:
    exact {target_profile_id, target_profile_version, target_profile_content_hash}
  quality_profile_ref:
    exact {quality_profile_id, quality_profile_version, quality_profile_content_hash}

MaterialBatchCompatibilityKeyV1
  schema_version: 1
  target_profile_ref:
    exact {target_profile_id, target_profile_version, target_profile_content_hash}
  quality_profile_ref:
    exact {quality_profile_id, quality_profile_version, quality_profile_content_hash}
  artifact_generation_ref: CookedMaterialArtifactGenerationRefV1
  shader_interface_hash: SHA-256
  pipeline_variant_key_hash: SHA-256
  vertex_interface_hash: SHA-256
  pass_interface_hash: SHA-256
  batch_uniform_parameter_set_hash: SHA-256
  batch_uniform_texture_binding_set_hash: SHA-256
  immutable_resource_layout_hash: SHA-256
  per_render_instance_layout_hash: SHA-256
  alpha_mode: AlphaMode
  depth_policy_hash: SHA-256
  sort_class: StableEnumId
  batch_key_hash: SHA-256

ResolvedMaterialBindingV1
  schema_version: 1
  material_instance_ref:
    exact {material_instance_id, revision, content_hash}
  source_definition_ref:
    exact {material_definition_id, revision, content_hash}
  artifact_generation_ref: CookedMaterialArtifactGenerationRefV1
  resolved_source_parameter_set_hash: SHA-256
  texture_role_binding_set_hash: SHA-256
  runtime_override_ref:
    optional exact {runtime_override_id, override_sequence, override_hash}
  resolved_parameter_block_ref:
    exact {block_id, block_generation, layout_hash, block_content_hash}
  resolved_texture_binding_block_ref:
    exact {block_id, block_generation, layout_hash, block_content_hash}
  shader_interface_hash: SHA-256
  batch_compatibility_key: MaterialBatchCompatibilityKeyV1
  per_render_instance_payload_ref:
    exact {payload_id, payload_generation, layout_hash, payload_content_hash}
  source_revision: positive uint64
  binding_hash: SHA-256
```

`MaterialBatchCompatibilityKeyV1`のTarget／Quality refは`artifact_generation_ref`内のrefとbyte equalityにする。このkeyの一致はMaterial側のbatch必要条件であり、十分条件ではない。Source InstanceとRuntime Overrideを解決した後、`uniform_across_batch`のParameter値またはTexture role bindingの差は対応するbatch-uniform hashとkeyを変え、`per_render_instance`／`per_render_instance_indexed`の差だけを宣言済みCooked layoutのpayloadへ格納できる。異なるkeyを同じbatchへ入れず、同じkeyでもgeometry generation、LOD、View、Pass、layer、sort等のRender Graph条件を満たすまでbatch可能とみなさない。Stable render ID、transform、Gameplay identityはkeyから除外し、batch化でobject identityを統合しない。

`ResolvedMaterialBindingV1`はrenderableごとのimmutable bindingであり、Cook時に解決したcanonical flat parameter／texture集合とRuntime OverrideをEngine-stable block／payload refへ閉じる。refは公開Schemaでnative pointer／descriptorを示さず、Artifact世代と同じResource Registry generationへ一件解決する。Artifact、block、payload generation、Source revision、`ShaderInterface`、resource layout、override sequenceのいずれかがstaleまたは不一致ならbinding全体を拒否し、部分適用、文字列名による再解決、descriptor index／native handleの公開、同一frameの別Materialへの差し替えを行わない。`batch_key_hash`と`binding_hash`はそれぞれ型名のASCII domain separatorと、自己hash Fieldを除くcount／presence／length-framed canonical bytesから計算し、参照先hashは各Ownerの式で再検証する。

## 6. Material IR、Shader compile、package

CookerはSource Graphとexact Function dependency closureをBackend-neutral `MaterialIRV1`へlowerする。`MaterialIRV1`はschema version、Source closure hash、Function dependency closure hash、Domain output、Stable ID順のtyped IR node／edge、Function／call-site origin map、resource role、uniform／per-render-instance layout、declared feature requirement、Project Shader opaque node ref、`ShaderInterface` hash、IR hashだけを持つ。constant folding、dead-node removal、interface validation、variant canonicalization後の同値IRを同じcanonical hashへ閉じる。Qualification済み`ProjectShaderModuleV1`はSourceを高水準IRへ逆変換せず、Module／Export exact ref、semantic interface hash、`ShaderFactGraphV1` hashを持つtyped opaque nodeとして参照する。native bytecode、compiler-specific metadata、source path、native include、descriptor indexはpublic contractにしない。

Shader compileはoffline build／cookだけで実行し、Shipping Runtimeにsource compiler、unapproved source、debug fallback shaderを含めない。`CookedMaterialArtifactV1`はartifact ID／generation、Source Definition compile closure hash、Target／Quality Profile ref、`MaterialIRV1` hash、shader artifact set refs、`ShaderInterface` hash、parameter／resource／per-render-instance layout hash、batch compatibility template hash、variant manifest ref、toolchain／provenance refs、content hashを束ねる。Instanceのflattened value／texture closureをartifact identityへ含めず、compile feature、interface、layoutが同じInstanceは同じArtifact generationを共有できる。Project ShaderのSource／Fact／Target conformanceは[Project Shader](project-shader.md)、compiler／translatorのexact version、commit、license、build optionは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)だけが所有する。

`MaterialVariantManifestV1`はTarget／Quality Profile、使用Source closure、Stable ID順のvariant key、各shader artifact exact ref、required vertex／Pass interface、feature requirement、resource／pipeline budget、manifest hashを持つ。Variant keyはDomain、Shading Model、declared compile-time feature、vertex interface、Pass interface、Target capabilityからcanonicalに作り、arbitrary user keywordやruntime stringで増殖させない。Packageは使用closureからrequired variantを列挙し、missing critical variant、stale generation、reflection／layout不一致をRuntime compileや代替shaderで隠さない。

## 7. Material LOD境界

Materialは各representationで利用可能なMaterial artifact、feature reduction、texture residency requirement、意味同等fallbackを宣言する。[LOD](lod.md)がrepresentationとtransitionを選択し、Materialsはその選択に対応するbindingを返す。

`MaterialLodProfileV1`はtierごとに`material_interface_hash`、`allowed_feature_mask`、`texture_residency_floor`、`shadow_participation`、`depth_participation`、`visual_equivalence_tolerance`を持つ。Runtime shader生成、任意branch削除、Material mergeをLOD selection内で行わない。

距離、projected error、hysteresis、CPU／GPU pressureからMaterial tierを直接選ばない。Material側のfeature reductionはsurface identity、opacity、silhouetteに影響する意味変更を明示し、未宣言のshader simplificationを禁止する。

## 8. AI／Editor operationとPreview

Materialのcreate／update、instance、texture role、style、semantic parameter、compile preview、explain、validateは将来のDomain action vocabularyであり、現在のOperation登録ではない。

予約候補IDは`operation.material.search`、`operation.material.read`、`operation.material.inspect`、`operation.material.preview`、`operation.material.explain`、`operation.material.estimate`、`operation.material.validate`、`operation.material.plan`、`operation.material.create_instance`、`operation.material.assign_template`、`operation.material.set_parameters`、`operation.material.create_definition`、`operation.material.edit_graph`、`operation.material.create_derived_style`、`operation.material.bind_surface_semantics`のexact 15件であり、[Executable contracts](../02-foundation/executable-contracts.md#211-既存domain文書から回収した未登録operation候補)の`planning.operation_family.material_authoring@1`だけに属する。Capability stateは`not_activated`、current MCD／Owner Manifest／Service allowlist／Policy／Validator／Diagnostic／Receipt／Provider／MCP／generated alias／legacy alias集合はすべて`[]`である。`operation.material.plan`と`operation.material.create_derived_style`はActivation後にだけactive Visual Style Registryからexact semantic refを選択する予定候補であり、現在のselection権限ではない。Visual Style contribution自体のauthoring／activationは別計画語彙`planning.material.visual_style_semantic_contribution_authoring@1`とし、reserved Operation ID集合`[]`、current MCD／Owner Manifest／Service allowlist／Policy／Validator／Diagnostic／Receipt／Provider／MCP／CLI／Editor／generated alias／legacy alias集合もexact `[]`、Capability stateも`not_activated`である。Material Functionの作成、Port契約変更、dependency revision更新も別計画語彙`planning.material.function_authoring@1`とし、reserved Operation ID集合とcurrent MCD／Owner Manifest／Service allowlist／Policy／Validator／Diagnostic／Receipt／Provider／MCP／CLI／Editor／generated alias／legacy alias集合をexact `[]`、Capability stateを`not_activated`とする。既存`operation.material.edit_graph`からFunction Asset作成、Function契約変更、参照元一括更新の権限を推測しない。Foundationが専用Operationをatomic登録するまで15候補からRegistryまたはFunction authoring権限を推測しない。`activation.material.authoring_operations.v1`が15件を同じContract set transactionで完全登録するまでGatewayはdispatchせず、要求を`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`でSource不変として拒否する。上位候補が下位権限を暗黙取得せず、予定変更候補はSourceを直接writeしない。新しいHLSL Module／Techniqueが必要な場合の`MaterialAuthoringPlanV1`とProject Shader handoffも各familyのActivation後にだけ有効であり、Material候補名からProject ShaderまたはEngine Extension権限を生成しない。

Previewは対象revision、Target Profile、View／Lighting fixture ref、resolved Material／Style、compiled artifact generation、difference summary、diagnosticを返す。Preview結果をApply済みProject stateやProduction qualificationと表示しない。Explainは採用値、継承元、override、fallback、未解決questionをMaterial語彙で示す。

[Lighting](lighting.md)のIntent Resolver入力である`MaterialReadabilitySummaryV1`は本書所有のread-only／revisioned projectionである。`schema_version`、`project_revision`、`catalog_revision`、`scope_ref`、`entries[]`、`cursor`、`total_count`を持ち、各entryは`material_instance_id`、`revision`、`semantic_role_id`、`domain`、`shading_model`、`gameplay_critical`、§3のsemantic axesのうちopacity behavior／emissive intent／two-sided intent、`style_critical`だけを含む。参照するMaterial revisionまたはCatalog revisionの変更でstaleとし、Lighting側からの書き戻しを受けない。上限超過時は配列を切らずcursorとtotal countを返す。

AIの最初のbounded projectionである`MaterialContextSummaryV1`は`material_id`、`revision`、`semantic_role_id`、exact Template／Definition／Instance ref、`domain`、`shading_model`、`pbr_feature_tier`、exact PBR conformance profile ref、`public_parameters[]`、各Parameterのtype／unit／color-space／range／override scope／runtime mutability／batching class／compile impact、`texture_dependencies[]`、各Texture roleのoverride scope／batching class、`material_function_refs[]`、Functionごとのexact revision／semantic content hash／Port contract hash／dependency completeness、`project_shader_module_refs[]`、`shader_understanding_closure_hashes[]`、`target_support[]`、`quality_support[]`、`variant_count`、artifact generation、batch eligibility summary、`budget_summary`、`diagnostic_summary`、`available_operation_ids[]`だけを含む。上限超過時は配列を切らずcursorとtotal countを返す。native descriptor、Backend handle、Runtime Overrideの値そのもの、Function Graph内部、Module内部のsymbol／call／resource／Target差は含めず、Function内部は同revisionを保持したbounded Function query、Module内部は`ShaderContextSliceV1`を別Queryで取得する。

`MaterialAuthoringPlanV1`はIntentから生成するread-only Proposalであり、base revision、選択したsemantic role／template／definition、typed parameter／texture差分、代替候補と棄却理由、Target差、cost、fallback、risk、必要Approval、optional `project_shader_requirement`を持つ。`project_shader_requirement`はRequirement ID、必要なS2～S5 Level、semantic input／output、Target、Budget、fallbackだけを持ち、HLSL SourceやPassを埋め込まない。共通Proposal／ChangeSet envelopeは[Executable contracts](../02-foundation/executable-contracts.md)の正本を再利用し、Plan自体にCommit権限はない。

`VisualStyleDecision`はMaterials-domain payloadとして`request_id`、exact Registry／Activation Projection ref、`resolved_semantic_refs[]`、`resolved_requirements[]`、`unknowns[]`、`conflicts[]`、`eligible_profile_ids[]`、`rejected_candidates[]`、optional `selected_profile_id`、`selection_reasons[]`、`production_cost_estimate`、`runtime_cost_estimate`、`required_capabilities[]`、`missing_capabilities[]`、`domain_result`、exact `decision_ledger_entry_ref`、exact `authorization_envelope_hash`を持つ。`decision_ledger_entry_ref`は[Project state](../03-authoring/project-state.md)の`DecisionLedgerDocument`、`authorization_envelope_hash`は[AI Security／Approval](../01-governance/ai-security-approval.md)の署名済み`TaskAuthorizationEnvelope`へのopaque referenceである。権限enum、委任cardinality、承認／人間確認条件、署名／失効規則をMaterialsに定義せずGovernanceの正本を消費する。

旧payloadの`decision_authority`、`delegation_record_id`、`requires_human_confirmation`はMaterials fieldとして非採用である。対応するauthority／delegation／confirmation状態は`TaskAuthorizationEnvelope`とGovernanceが発行する署名済みApproval recordの参照先だけで評価する。

`MaterialExplanationV1`は`request_id`、`material_id`、`source_revision`、`resolved_intents[]`、`selected_semantic_role_id`、`selected_template_id`、`changed_parameters[]`、`selection_reasons[]`、`rejected_candidates[]`、`assumptions[]`、`target_differences[]`、`predicted_cost`、optional `measured_cost`、`fallbacks[]`、`warnings[]`、`required_human_confirmations[]`、`confidence`を持つ。`confidence`はAI自己申告ではなく、unknown／conflict／capability／budget／Preview状態からEngineが再計算する。

描画MaterialとCollision Materialは別namespace／Asset／Validatorとする。`SurfaceSemanticBindingV1`は`binding_id: StableId`、`semantic_surface_id: StableEnumId`、optional `render_material_instance_id`、optional `collision_material_2d_id`、optional `collision_material_3d_id`、optional `audio_surface_id`、optional `vfx_surface_id`を持つ明示参照であり、見た目から摩擦／反発／Damage／足音を推測しない。

## 9. Diagnostic、failure、fallback

Material固有diagnosticはasset／node／parameter／style axis／variant key、source span、Target、error code、remediationを含む。少なくともunknown semantic、graph cycle、type mismatch、domain mismatch、missing resource、unsupported shading model、variant explosion、compile failure、reflection mismatch、stale bindingを区別する。

Diagnostic IDを`MIRAKAN-MATERIAL-SEMANTIC_ROLE_UNKNOWN | MIRAKAN-MATERIAL-DOMAIN_MISMATCH | MIRAKAN-MATERIAL-GRAPH_INVALID | MIRAKAN-MATERIAL-FUNCTION_INVALID | MIRAKAN-MATERIAL-FUNCTION_REVISION_STALE | MIRAKAN-MATERIAL-PARAMETER_INVALID | MIRAKAN-MATERIAL-TEXTURE_ENCODING_MISMATCH | MIRAKAN-MATERIAL-CAPABILITY_NOT_ACTIVATED | MIRAKAN-MATERIAL-BUDGET_EXCEEDED | MIRAKAN-MATERIAL-INTERFACE_MISMATCH | MIRAKAN-MATERIAL-VARIANT_LIMIT | MIRAKAN-MATERIAL-COMPILE_FAILED | MIRAKAN-MATERIAL-STYLE_LOCK_VIOLATION | MIRAKAN-MATERIAL-FALLBACK_REQUIRED | MIRAKAN-MATERIAL-PROVENANCE_MISSING | MIRAKAN-MATERIAL-PREVIEW_STALE | MIRAKAN-MATERIAL-UNAUTHORIZED_SOURCE | MIRAKAN-MATERIAL-COLLISION_NAMESPACE_MISMATCH | MIRAKAN-MATERIAL-DECAL_SCHEMA_INVALID | MIRAKAN-MATERIAL-DECAL_STALE_COMMAND | MIRAKAN-MATERIAL-DECAL_RECEIVER_UNSUPPORTED | MIRAKAN-MATERIAL-DECAL_CAPACITY_EXCEEDED | MIRAKAN-MATERIAL-DECAL_CRITICAL_FALLBACK_REQUIRED`に固定する。

Editor previewだけがcompile失敗時に直前のvalid pipelineを明示表示できる。Shippingは不合格Artifact、missing／invalid Materialを含むPackageまたはScene loadを失敗させる。Capability不足時に別Styleへ黙って変更しない。

fallbackはSourceで宣言され、意味差とTarget範囲を持つ。compile失敗、missing texture、capacity不足時にdefault material、opaque、unlit、texture dropへ黙って切り替えない。共通capacity／backpressureは[Runtime performance／capacity](../04-runtime/performance-capacity.md)へ委譲する。

現在、realistic／toon／unlitを相互置換できる汎用かつ意味保存のMaterial fallbackは定義されていない。したがって`fallback.rendering.material-default`は本書Ownerの有効fallbackではなく、Product execution registryにも登録しない。将来追加する場合は、対象semantic、Target、意味差、owner、Diagnostic、Qualification fixtureを持つexact fallback recordと参照元を同じProduct Definition revisionでatomicに導入する。それまでは共通`fallback.capability.unavailable`で失敗を明示する。

## 10. Qualificationと完了条件

Qualificationは次のDomain fixtureを持つ。

| 対象 | 必須Test |
|---|---|
| PBR | [Khronos glTF 2.0](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html) metallic-roughness、linear／sRGB channel、perceptual roughness二乗、dielectric IOR、提供Tangentと未提供時MikkTSpace、tangent-space normal、IBL／BRDF analytic invariant、Asset Generator／Sample Assets／Validator、basic／advanced extension境界、Target未対応extension拒否 |
| Toon／Outline | sphere、face、hair、transparent hair、eye、cloth、foliage、analytic band／ramp signal・filter・clamp・channel、Toon Feature Role Profileのrole／texture role／World compatibility、Key／fill／rim／accent Light、cast／receive shadow、energy cap、geometry-only／screen-space-only／hybrid-qualified／disabled outline、World Profile／Target fallback |
| Pixel 2D | 720p、1080p、1440p、ultrawide、4Kのscale／letterbox |
| Pixel Diorama | depth、occlusion、shadow coverage、Fog、DOF、Bloom、TAA分離 |
| Visual Style Registry | 初期Core七entryの既存Profile／render hash golden、Genre Pack 0件のneutral Project、qualified `project.board_game.paper_cutout@1` contribution。unknown ref、未Qualification、axis違い、owner／hash stale、Core ID上書き、同一ID別hash、Pack未選択、display name／synonymだけの選択を各一原因で拒否 |
| Compiler | invalid Graph、Project Shader Profile違反、未宣言resource／side effect、binding／reflection不一致、Target差、cache再現 |
| Material Function | Portの型／unit／color／coordinate／semantic一致、input default、output set equality、direct／indirect recursion、依存深度16／17、exact revision pin、stale ref、明示upgradeの影響Preview、layout-only変更のsemantic hash不変、inline／共有IR lowering同値、origin map、Target／capacity超過 |
| Material Instance／Runtime Override | 親深度0／8／9、cycle／別Definition、flatten再現、unknown／型／unit／color-space／range／style lock、Source InstanceとRuntime Overrideの許可scope、sequence重複／stale generation／lease終了、Source／Save／Replay hash非変更 |
| Batch compatibility | 同一Definitionの多数Instanceが非compile overrideでArtifact generationを共有、compile-time／uniform-across-batch／per-render-instance差、同一key必要条件、異なるkey非結合、geometry／LOD／View／Pass差、individual／instancedで同じStable render ID集合とvisual oracle、capacity丁度／+1でsilent dropなし |
| Target | Windows、Android、Appleのoffline compile、pipeline、fallback |
| Decal | Forward+、MSAA 1x／2x／4x、receiver mask、opaque／masked以外とskinned receiver除外、同一面sort、timed fade、capacity丁度／+1、critical fallback、camera cut、owner World／Scene deactivate、Windows／Mobile fallback、固定Camera／Materialによるangle／depth bias／normal blend／fadeのgolden regression、Decal不在時のauthoritative state hash一致 |

同一Reference GPU／driverのgolden imageはSSIM 0.995以上、絶対channel差2／255超のpixelが0.1%未満を既定Gateとする。Cross-vendorはparameter ordering、luminance、finite、outline width、pixel grid等のanalytic invariantを検証する。

Material AI Evalは10 suite（明示Intentのrole／Template、既存Instance最小変更、Template再利用判断、Graph／Functionの型／単位／色空間／依存revision、曖昧／矛盾質問、Target／fallback、Variant／resource、Project Node選択と未宣言Shader効果拒否、Visual／Collision Material分離、Preview／ChangeSet／undo／redo／recook）を各12 fixture、合計120 fixtureとして各3回実行する。Functionを含むCaseはexact revision、Port契約、依存completenessを読み、Function内部未取得、silent latest追従、再帰、参照元一括更新を安全な変更として扱わない。全suiteでPBR feature tier、Source／Runtime override scope、compile影響、batching classをContextから説明し、許可されない変更を計画へ混入しないことを横断条件にする。hard gate違反、無権限Commit、unsupported成功表示は0件、exact planned semantic action token／Type／unit／range、Blocking確認、禁止操作拒否は360／360、role／Template選択100%、Preview hash／undo／redo／recook一致100%を要求する。Project Shader内部のU0～U4理解は[Project Shader](project-shader.md)のEvalで別に100%閉じ、Material scoreで代替しない。意味、Schema、画像、GPU性能を別scoreとしhard failureを平均で相殺しない。

Visual Style Resolver Evalは明示、未指定、委任、矛盾、未対応を各12件、合計60 prompt、各3回実行し、Core default、Feature／Genre／Project contribution、Pack非選択、neutral non-game Projectを含める。hard gate違反と無権限Commit 0件、Blocking質問／明示Style保持／委任scope／unsupported拒否180／180、Registry／semantic ref／qualificationのexact一致100%を要求する。

Visual comparison、Evidence envelope、Eval grading、provenanceは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を使う。本書はfixtureのMaterial input、expected semantic resolution、allowed visual tolerance classだけを所有し、共通receipt schemaを再定義しない。

Runtime source compile、unbounded variant、string dispatch、stale artifact mix、silent default material、unqualified fallbackが残るPackageはRelease候補にしない。本書はdomain qualification evidenceを出力し、activationと導入順は[Product Plan](../00-product/product-plan.md)が決定する。
