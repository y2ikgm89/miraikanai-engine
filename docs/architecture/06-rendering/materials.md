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

Authoring surfaceのcanonical objectを次に固定する。下記と別のAsset名をalias／accepted typeとして設けない。

| Object | 正本field／責務 | 変更規則 |
|---|---|---|
| `MaterialDefinition` | Stable ID／revision、Domain、Shading Model、surface intent、typed graphまたは承認済みProject module ref、typed parameter declaration、texture／sampler role、render state、compile feature、`ShaderInterface` hash | Graphまたはcompile feature変更で新revision |
| `MaterialInstance` | Stable ID／revision、parent Definition ref、Texture binding、公開parameter override、style override、compatibility constraint | Domain、Shading Model、Alpha Mode、Depth policyを変更不可 |
| `MaterialTemplate` | Stable ID／revision、semantic role、既定Definition ref、公開parameter policy | `VisualStyleProfile`がroleごとに選択 |
| `MaterialSemanticCatalogV1` | role、意味、channel、互換性、例、qualification class | Engine build／Project extensionから生成 |
| `MaterialNodeCatalogV1` | Nodeの型、意味、cost、Target、制約 | Engine buildから生成しAI変更不可 |
| `ArtAssetProfile` | Stable ID／revision、Mesh／Sprite／Texture制作規則、Palette、semantic role | Importer、Generator、Validatorが同一revisionを共有 |
| `AnimationPresentationProfile` | Stable ID／revision、presentation sampling、pose hold、motion accent | Simulation meaningから分離し表示だけを変更 |
| `VisualStyleProfile` | Material、Light、Camera、Post、VFX、UI、Asset制作規則のStyle契約 | `StyleChangeSet`／Preview／承認を必須とし、下記exact fieldsを使う |
| `StyleCapabilityManifest` | `engine_build_id`、`target_profile_id`、`quality_profile_id`、利用可能なMaterial Domain／Shading Model／Node／Template／Style feature ID、required Capability／Qualification ref | Engine buildから生成しAI／Project data変更不可 |
| `VisualStyleDecision` | 候補、除外理由、選択理由、未解決事項、Capability、domain result | Decision Ledger／Governance参照を持ち、authority／approvalは所有しない |
| `MaterialExplanationV1` | Material判断の根拠、差、cost、fallback | Preview／Plan revisionへ紐付け |

全Source objectはStable ID、schema version、content hash、revisionを持つ。Runtime packageへEditor node位置、UI state、AI prompt、自由形式の判断理由を持ち込まない。

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

### 3.1 `MaterialSemanticCatalogV1`

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

初期role setは`surface.generic | surface.terrain | surface.foliage | surface.water | character.skin | character.hair | character.eye | character.cloth | prop.opaque | prop.transparent | sprite.actor | sprite.environment | sprite.effect | decal.surface | vfx.particle | ui.surface`である。synonym／example／counter-exampleは検索用で正規IDではなく、Asset名や作品名だけでroleを確定しない。Project roleはEngine roleを上書きせず`project.<project_id>.*` namespaceへ追加する。

### 3.2 `MaterialParameterSemanticV1`

`parameter_id: StableId`、`semantic_id: StableEnumId`、`value_type`、`unit`、`color_space: none | linear_rgb | srgb | data`、`valid_range`、`default_value`、`default_provenance`、`compile_time: bool`、`ai_mutable: bool`、`style_critical: bool`、`runtime_mutable: bool`を持つ。AIはexact parameter ID／typeを使い、表示名／shader variable名で変更しない。

### 3.3 `MaterialNodeCatalogV1`

各Node entryは`node_type_id`、`schema_version`、`display_name`、`semantic_description`、`input_ports[]`、`output_ports[]`、`allowed_domains[]`、`allowed_stages[]`、`required_capabilities[]`、`target_support[]`、`static_cost_estimate`、`resource_cost`、`determinism_class`、`failure_codes[]`、`examples[]`、`counter_examples[]`を持つ。Portはtypedで、implicit scalar／vector expansion、linear／sRGB混在、normal／color混在を許可しない。Graph layoutはsemantic hashに含めない。

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

`realistic_basic`はScene-linear HDR、物理単位Light、IBLを入力とし、Cook-Torrance BRDFをGGX normal distribution、Smith height-correlated visibility、Schlick Fresnelで評価する。Metallic-Roughness workflowのcanonical入力はbase color、metallic、perceptual roughness、tangent-space normal、occlusion、emissiveである。Authoringのmetallic／perceptual roughnessは0～1、shader内部FP32 roughnessは0.045～1へclampする。Opaque、alpha mask、premultiplied transparent、Environment IBL reflection、shadow、height fog、exposure、tone mappingを同じ意味契約で扱う。

glTF coreと基本extensionは次のMaterials-owned mappingへ固定する。

| glTF入力 | canonical Material意味 |
|---|---|
| core base color | Texture RGBをsRGBからscene-linear base colorへdecodeし、Aをopacityとする |
| core metallic-roughness | data textureのGをperceptual roughness、Bをmetallicとし、R／Aを意味入力に使わない |
| core normal／occlusion／emissive | normal RGBをtangent-space data、occlusion Rをlinear occlusion、emissive RGBをsRGBからscene-linear emissiveへdecodeする |
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

`authored_static`だけがLevel SourceとSave互換identityを持つ。impact等の`timed` DecalはHit／Interaction等のauthoritative Eventから再生成するpresentationであり、Damage、collision、visibility、surface friction、Gameplay state、SaveのいずれもDecalのresidencyまたはrender resultに依存しない。Replayも元Eventから再生成し、Decalの有無でauthoritative state hashを変化させない。

C1 reference Profileはactive 2,048、spawn 128／tick、visible 512／viewをhard boundとする。超過時は`ambient`、`gameplay_feedback`の順にcanonical evictionしてDiagnosticを残す。`critical_feedback`は黙って欠落させず、Target Profileのfallbackへ切替えるか`MIRAKAN-MATERIAL-DECAL_CRITICAL_FALLBACK_REQUIRED`で拒否する。その他のclosed failureは`MIRAKAN-MATERIAL-DECAL_SCHEMA_INVALID | MIRAKAN-MATERIAL-DECAL_STALE_COMMAND | MIRAKAN-MATERIAL-DECAL_RECEIVER_UNSUPPORTED | MIRAKAN-MATERIAL-DECAL_CAPACITY_EXCEEDED`とする。

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

`MaterialLodProfileV1`はtierごとに`material_interface_hash`、`allowed_feature_mask`、`texture_residency_floor`、`shadow_participation`、`depth_participation`、`visual_equivalence_tolerance`を持つ。Runtime shader生成、任意branch削除、Material mergeをLOD selection内で行わない。

距離、projected error、hysteresis、CPU／GPU pressureからMaterial tierを直接選ばない。Material側のfeature reductionはsurface identity、opacity、silhouetteに影響する意味変更を明示し、未宣言のshader simplificationを禁止する。

## 8. AI／Editor operationとPreview

Material operationはcreate／update material、create instance、bind texture role、apply style、set semantic parameter、compile preview、explain resolution、validate packageをDomain actionとして登録する。操作は[Executable contracts](../02-foundation/executable-contracts.md)の共通Discovery／Preview／Apply境界と[AI Security／Approval](../01-governance/ai-security-approval.md)のauthorityを使う。

canonical Operation IDは`operation.material.search`、`operation.material.read`、`operation.material.inspect`、`operation.material.preview`、`operation.material.explain`、`operation.material.estimate`、`operation.material.validate`、`operation.material.plan`、`operation.material.create_instance`、`operation.material.assign_template`、`operation.material.set_parameters`、`operation.material.create_definition`、`operation.material.edit_graph`、`operation.material.create_derived_style`、`operation.material.bind_surface_semantics`、`operation.material.propose_project_module`、`operation.material.propose_engine_extension`である。上位operationが下位権限を暗黙取得せず、変更operationはSourceを直接writeしない。

Previewは対象revision、Target Profile、View／Lighting fixture ref、resolved Material／Style、compiled artifact generation、difference summary、diagnosticを返す。Preview結果をApply済みProject stateやProduction qualificationと表示しない。Explainは採用値、継承元、override、fallback、未解決questionをMaterial語彙で示す。

AIの最初のbounded projectionである`MaterialContextSummaryV1`は`material_id`、`revision`、`semantic_role_id`、`template_id`、`definition_id`、`domain`、`shading_model`、`public_parameters[]`、`texture_dependencies[]`、`target_support[]`、`quality_support[]`、`variant_count`、`budget_summary`、`diagnostic_summary`、`available_operation_ids[]`だけを含む。上限超過時は配列を切らずcursorとtotal countを返す。

`MaterialAuthoringPlanV1`はIntentから生成するread-only Proposalであり、base revision、選択したsemantic role／template／definition、typed parameter／texture差分、代替候補と棄却理由、Target差、cost、fallback、risk、必要Approvalを持つ。共通Proposal／ChangeSet envelopeは[Executable contracts](../02-foundation/executable-contracts.md)の正本を再利用し、Plan自体にCommit権限はない。

`VisualStyleDecision`はMaterials-domain payloadとして`request_id`、`resolved_requirements[]`、`unknowns[]`、`conflicts[]`、`eligible_profile_ids[]`、`rejected_candidates[]`、optional `selected_profile_id`、`selection_reasons[]`、`production_cost_estimate`、`runtime_cost_estimate`、`required_capabilities[]`、`missing_capabilities[]`、`domain_result`、exact `decision_ledger_entry_ref`、exact `authorization_envelope_hash`を持つ。`decision_ledger_entry_ref`は[Project state](../03-authoring/project-state.md)の`DecisionLedgerDocument`、`authorization_envelope_hash`は[AI Security／Approval](../01-governance/ai-security-approval.md)の署名済み`TaskAuthorizationEnvelope`へのopaque referenceである。権限enum、委任cardinality、承認／人間確認条件、署名／失効規則をMaterialsに定義せずGovernanceの正本を消費する。

旧payloadの`decision_authority`、`delegation_record_id`、`requires_human_confirmation`はMaterials fieldとして非採用である。対応するauthority／delegation／confirmation状態は`TaskAuthorizationEnvelope`とGovernanceが発行する署名済みApproval recordの参照先だけで評価する。

`MaterialExplanationV1`は`request_id`、`material_id`、`source_revision`、`resolved_intents[]`、`selected_semantic_role_id`、`selected_template_id`、`changed_parameters[]`、`selection_reasons[]`、`rejected_candidates[]`、`assumptions[]`、`target_differences[]`、`predicted_cost`、optional `measured_cost`、`fallbacks[]`、`warnings[]`、`required_human_confirmations[]`、`confidence`を持つ。`confidence`はAI自己申告ではなく、unknown／conflict／capability／budget／Preview状態からEngineが再計算する。

描画MaterialとCollision Materialは別namespace／Asset／Validatorとする。`SurfaceSemanticBindingV1`は`binding_id: StableId`、`semantic_surface_id: StableEnumId`、optional `render_material_instance_id`、optional `collision_material_2d_id`、optional `collision_material_3d_id`、optional `audio_surface_id`、optional `vfx_surface_id`を持つ明示参照であり、見た目から摩擦／反発／Damage／足音を推測しない。

## 9. Diagnostic、failure、fallback

Material固有diagnosticはasset／node／parameter／style axis／variant key、source span、Target、error code、remediationを含む。少なくともunknown semantic、graph cycle、type mismatch、domain mismatch、missing resource、unsupported shading model、variant explosion、compile failure、reflection mismatch、stale bindingを区別する。

Diagnostic IDを`MIRAKAN-MATERIAL-SEMANTIC_ROLE_UNKNOWN | MIRAKAN-MATERIAL-DOMAIN_MISMATCH | MIRAKAN-MATERIAL-GRAPH_INVALID | MIRAKAN-MATERIAL-PARAMETER_INVALID | MIRAKAN-MATERIAL-TEXTURE_ENCODING_MISMATCH | MIRAKAN-MATERIAL-CAPABILITY_NOT_ACTIVATED | MIRAKAN-MATERIAL-BUDGET_EXCEEDED | MIRAKAN-MATERIAL-INTERFACE_MISMATCH | MIRAKAN-MATERIAL-VARIANT_LIMIT | MIRAKAN-MATERIAL-COMPILE_FAILED | MIRAKAN-MATERIAL-STYLE_LOCK_VIOLATION | MIRAKAN-MATERIAL-FALLBACK_REQUIRED | MIRAKAN-MATERIAL-PROVENANCE_MISSING | MIRAKAN-MATERIAL-PREVIEW_STALE | MIRAKAN-MATERIAL-UNAUTHORIZED_SOURCE | MIRAKAN-MATERIAL-COLLISION_NAMESPACE_MISMATCH | MIRAKAN-MATERIAL-DECAL_SCHEMA_INVALID | MIRAKAN-MATERIAL-DECAL_STALE_COMMAND | MIRAKAN-MATERIAL-DECAL_RECEIVER_UNSUPPORTED | MIRAKAN-MATERIAL-DECAL_CAPACITY_EXCEEDED | MIRAKAN-MATERIAL-DECAL_CRITICAL_FALLBACK_REQUIRED`に固定する。

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
| Decal | receiver mask、opaque／masked以外の拒否、同一面sort、timed fade、capacity丁度／+1、critical fallback、camera cut、Level deactivate、Windows／Mobile fallback、Decal不在時のauthoritative state hash一致 |

同一Reference GPU／driverのgolden imageはSSIM 0.995以上、絶対channel差2／255超のpixelが0.1%未満を既定Gateとする。Cross-vendorはparameter ordering、luminance、finite、outline width、pixel grid等のanalytic invariantを検証する。

Material AI Evalは10 suite（明示Intentのrole／Template、既存Instance最小変更、Template再利用判断、Graph型／単位／色空間、曖昧／矛盾質問、Target／fallback、Variant／resource、禁止HLSL／任意pass、Visual／Collision Material分離、Preview／ChangeSet／undo／redo／recook）、各12 fixture、合計120 fixtureを各3回実行する。hard gate違反、無権限Commit、unsupported成功表示は0件、exact Operation／Type／unit／range、Blocking確認、禁止操作拒否は360／360、role／Template選択100%、Preview hash／undo／redo／recook一致100%を要求する。意味、Schema、画像、GPU性能を別scoreとしhard failureを平均で相殺しない。

Visual Style Resolver Evalは明示、未指定、委任、矛盾、未対応を各12件、合計60 prompt、各3回実行し、hard gate違反と無権限Commit 0件、Blocking質問／明示Style保持／委任scope／unsupported拒否180／180を要求する。

Visual comparison、Evidence envelope、Eval grading、provenanceは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を使う。本書はfixtureのMaterial input、expected semantic resolution、allowed visual tolerance classだけを所有し、共通receipt schemaを再定義しない。

Runtime source compile、unbounded variant、string dispatch、stale artifact mix、silent default material、unqualified fallbackが残るPackageはRelease候補にしない。本書はdomain qualification evidenceを出力し、activationと導入順は[Product Plan](../00-product/product-plan.md)が決定する。
