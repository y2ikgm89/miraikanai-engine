# AI-Readable Rendering Contract Design

## 1. Status

- Date: 2026-07-25
- Scope: architecture contract correction before implementation
- Decision: adopt the owner-boundary correction approach
- Implementation status: design only
- Migration status: not applicable because no released implementation or persisted artifact exists

This design corrects three pre-implementation issues:

1. structural 2D／3D selection is owned by World rather than Visual Style;
2. Toon is represented by a typed profile rather than one undifferentiated shading-model label;
3. AI-verifiable shader meaning is rooted in typed IR, while DXC reflection is limited to observed compiler facts.

No runtime code, implementation plan, compatibility migration, Operation activation, Capability promotion, or Production claim is created by this document.

## 2. Official design basis

The following primary sources define the external comparison baseline. They are
evidence for separating concerns, not templates for this engine. This design
does not copy external source code, serialized formats, API type names,
editor labels, shader text, or compatibility promises. Every `V1` schema,
identity rule, fallback rule, and AI-verification rule below is
Miraikanai-owned.

### 2.1 Unity Graphics

- Shader Graph `Target` defines generated-shader compatibility and generation requirements.
- Sprite Lit／Unlit shader graphs are documented separately from general
  Lit／Unlit graphs.
- Structured C# declarations can generate corresponding HLSL declarations.

Sources:

- <https://github.com/Unity-Technologies/Graphics/blob/master/Packages/com.unity.shadergraph/Documentation~/Graph-Target.md>
- <https://github.com/Unity-Technologies/Graphics/blob/master/Packages/com.unity.render-pipelines.core/Documentation~/generating-shader-includes.md>
- <https://github.com/Unity-Technologies/Graphics/blob/master/Packages/com.unity.shadergraph/Documentation~/Custom-nodes-hlsl-create-node-by-reflection.md>
- <https://docs.unity3d.com/ja/current/Manual/urp/prebuilt-shader-graphs-urp-sprite-unlit.html>

Adopted principle: target compatibility, authoring structure, and generated shader text are separate concerns.

### 2.2 Unreal Engine

- Material Domain and Shading Model describe material execution and lighting behavior rather than project-wide 2D／3D identity.
- UE 5.8 exposes experimental Substrate Toon shading with a separate Toon
  Profile.
- Toon response exposes structured diffuse／specular ramp behavior and self-shadowing controls.

Sources:

- <https://dev.epicgames.com/documentation/unreal-engine/unreal-engine-material-properties>
- <https://dev.epicgames.com/documentation/unreal-engine/API/Runtime/Engine/FSubstrateMaterialInfo>
- <https://dev.epicgames.com/documentation/unreal-engine/unreal-engine-5-8-release-notes>
- <https://dev.epicgames.com/documentation/unreal-engine/paper-2d-sprite-material-in-unreal-engine>

Adopted principle: Toon is not merely a display name; it needs a typed response
contract, while 2D sprite use remains a separate asset／domain concern. The
experimental status of Unreal's current feature is not a maturity claim for
Miraikanai.

### 2.3 Godot Engine

- `canvas_item` and `spatial` identify 2D and 3D shader contexts.
- `diffuse_toon` and `specular_toon` are render-mode choices inside a spatial
  shader, rather than project-wide 2D／3D switches.

Sources:

- <https://docs.godotengine.org/en/stable/tutorials/shaders/your_first_shader/your_first_2d_shader.html>
- <https://docs.godotengine.org/en/stable/tutorials/shaders/shader_reference/spatial_shader.html>

Adopted principle: spatial context and surface-response choice are orthogonal.
Godot API names and shader syntax are not adopted.

### 2.4 DirectX Shader Compiler

DXC public APIs provide compile status, diagnostics, object output, PDB, shader
hash, and reflection. Reflection exposes compiled interfaces such as entry
signatures, constants, and resources. This design therefore treats
compiler-private AST text and debug-container internals as non-authoritative
implementation details; that boundary is an engine decision, not a DXC promise.

Sources:

- <https://github.com/microsoft/DirectXShaderCompiler/wiki/Using-dxc.exe-and-dxcompiler.dll>
- <https://github.com/microsoft/DirectXShaderCompiler/wiki/Advanced-Error-Handling>
- <https://github.com/microsoft/DirectXShaderCompiler/blob/main/include/dxc/Support/HLSLOptions.td>

Adopted principle: DXC verifies generated artifacts and observable interfaces; it does not replace an engine-owned typed semantic IR.

## 3. Ownership correction

### 3.1 Current problem

`VisualStyleProfile` currently contains `scene_dimension` and `gameplay_space`, even though the same document describes them as structural inputs that branch Rendering, Physics, and Navigation.

This creates three risks:

- a visual-style change can appear to change authoritative simulation space;
- reusable styles must duplicate structural choices;
- Physics, Camera, Navigation, and Rendering can persist separate copies that drift.

### 3.2 Canonical owner

World owns spatial mode through `WorldSpaceProfileV1`.

```text
WorldSpaceProfileV1
  profile_id: StableId
  profile_version: positive uint32
  scene_dimension: two_d | three_d | hybrid
  hybrid_gameplay_space: two_d | three_d
    | canonical omission unless scene_dimension=hybrid
  required_capability_refs[0..32]: McdContractRefV1(kind=capability)
  profile_content_hash: SHA-256

WorldSpaceProfileRefV1
  profile_id: StableId
  profile_version: positive uint32
  profile_content_hash: SHA-256

WorldSpaceCompatibilityV1
  supported_scene_dimensions[0..3]:
    unique subset of {two_d, three_d, hybrid}
  supported_hybrid_gameplay_spaces[0..2]:
    unique subset of {two_d, three_d}
  supports_non_spatial: bool
```

Rules:

1. `two_d` selects authoritative 2D gameplay and 2D spatial semantics.
2. `three_d` selects authoritative 3D gameplay and 3D spatial semantics.
3. `hybrid` means mixed 2D／3D presentation with exactly one authoritative gameplay space. It is not a dual-authority physics or navigation mode.
4. UI-only and headless Runtime Entries do not invent a World or `WorldSpaceProfileV1`.
5. A Project may reference multiple Worlds with different profiles; dimension is not forced into a Project-global enum.
6. Style, Camera, Physics, Navigation, Animation, Collision, and Renderingのruntime resolution、attachment、Derived／Cooked recordはexact World profile referenceを消費する。再利用可能なSource Profileは`WorldSpaceCompatibilityV1`だけを宣言し、独立したcopied dimension valueを所有しない。
7. Coordinate handedness, axis, unit, angle, and matrix conventions remain fixed by Math／Core Utilities and are not configurable World-profile fields.
8. A derived artifact may materialize a resolved dimension only with the exact `WorldSpaceProfileRefV1` and its source hash. The materialized value is cacheable execution data, never an independent source of authority.
9. A true dual-authority 2D／3D simulation requires separate Worlds and an explicit cross-World bridge contract; it is outside this design.
10. `profile_content_hash` is `SHA-256(ASCII "MIRAKAN_WORLD_SPACE_PROFILE_V1" || uint32_be(length(receipt-free canonical bytes excluding profile_content_hash)) || canonical bytes)`.

`WorldDocumentV1` changes to:

```text
WorldDocumentV1
  world_id
  world_space_profile_ref: exact WorldSpaceProfileRefV1
  scene_document_refs[0..65535]
  global_composition_refs[]
  persistent_entity_refs[]
  spatial_topology_definition_ref: SpatialTopologyDefinitionV1 | null
```

### 3.3 Consumer compatibility

Reusable profiles declare compatibility without selecting structural state.

Validation rules:

- `supported_hybrid_gameplay_spaces[]` is empty when `hybrid` is absent.
- At least one scene dimension or `supports_non_spatial=true` is required.
- A selected World profile must match both dimension and hybrid gameplay-space constraints.
- A non-spatial Runtime Entry may use only profiles with `supports_non_spatial=true`.
- A compatibility failure is a typed blocking error; it never changes the World profile or silently selects another style.
- Dimension arrays are closed-set subsets, sorted by closed-enum ordinal, and have no duplicate. `supported_hybrid_gameplay_spaces[]` is sorted in the same way.

`VisualStyleProfile`, `CameraProfileDocumentV1`, and other reusable presentation profiles consume `WorldSpaceCompatibilityV1`. Runtime or authoring resolution additionally receives the exact selected `WorldSpaceProfileRefV1`.

Physics and Navigation resolution records replace copied `scene_dimension`／`hybrid_gameplay_space` fields with the exact World profile reference.

## 4. Typed Toon contract

### 4.1 Separation of responsibilities

- `toon_surface`, `sprite_toon`, and `hybrid_sprite_toon` remain Shading Model IDs.
- `ToonShadingProfileV1` owns surface response semantics.
- `OutlineStyleProfileV1` owns outline intent and invariants.
- Render Graph selects an Engine-owned qualified execution technique.
- Lighting resolves Light intent into the Toon response contract.
- Visual Style composes references; it does not duplicate Toon parameters.

### 4.2 Toon shading profile

```text
ToonRampRefV1
  ramp_id: StableId
  ramp_version: positive uint32
  source_asset_ref:
    exact {asset_id, asset_revision, source_sha256}
  input_signal:
    diffuse_ndotl_clamped | specular_lobe | shadow_attenuation
  dimension: one_d
  sample_filter: point | linear
  address_mode: clamp
  channel_semantic: scalar_multiplier | linear_rgb_multiplier
  ramp_content_hash: SHA-256
  color_space: linear_rgb

ToonBandResponseV1
  source: analytic_bands | ramp_asset
  input_signal:
    diffuse_ndotl_clamped | specular_lobe | shadow_attenuation
  band_count: integer in [1, 8]
    | required only for analytic_bands
  thresholds[0..7]: strictly increasing finite values in [0, 1]
    | count = band_count - 1 for analytic_bands
  softness[1..8]: finite values in [0, 1]
    | count = band_count for analytic_bands
  ramp_ref: ToonRampRefV1
    | required only for ramp_asset

ToonSpecularResponseV1
  mode: disabled | analytic_banded | ramp_asset | anisotropic_analytic_banded
  analytic_band_response: ToonBandResponseV1
    | required only for analytic_banded or anisotropic_analytic_banded
  ramp_ref: ToonRampRefV1
    | required only for ramp_asset
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
  analytic_band_response: ToonBandResponseV1
    | required only for banded
  ramp_ref: ToonRampRefV1
    | required only for profile_ramp

ToonEnergyPolicyV1
  kind: stylized_bounded | physically_bounded
  max_direct_lighting_multiplier: finite value in (0, 16]
  max_indirect_lighting_multiplier: finite value in (0, 16]

ToonFeatureSemanticV1
  role: generic | face | hair | eye | cloth | foliage
  normal_source:
    mesh_normal | authored_normal_map | bent_normal_map | role_profile
  shadow_source:
    engine_shadow | authored_mask | signed_distance_field | role_profile
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

Rules:

- Profile values are finite and canonicalized before hashing. `profile_content_hash` is `SHA-256(ASCII "MIRAKAN_TOON_SHADING_PROFILE_V1" || uint32_be(length(receipt-free canonical bytes excluding profile_content_hash)) || canonical bytes)`.
- `ramp_content_hash` is `SHA-256(ASCII "MIRAKAN_TOON_RAMP_REF_V1" || uint32_be(length(receipt-free canonical bytes excluding ramp_content_hash)) || canonical bytes)`.
- A ramp is a 1D data texture in linear color space. Its input signal, filter, clamp behavior, and channel interpretation are part of the reference and cannot be inferred from a filename or image dimensions.
- `ToonBandResponseV1.source=analytic_bands` requires the analytic fields and canonical omission of `ramp_ref`; `ramp_asset` requires canonical omission of the analytic fields and an exact `ramp_ref`.
- `ToonBandResponseV1.input_signal` is `diffuse_ndotl_clamped` for diffuse, `specular_lobe` for analytic specular, and `shadow_attenuation` for banded shadow response. A `ramp_ref` must use the same signal.
- `ToonSpecularResponseV1.mode=analytic_banded | anisotropic_analytic_banded` requires `analytic_band_response.source=analytic_bands`; `mode=ramp_asset` requires `ramp_ref.input_signal=specular_lobe`; `disabled` requires canonical omission of both response fields and a canonical zero specular contribution.
- `ToonShadowResponseV1.receive_mode=banded` requires `analytic_band_response.source=analytic_bands`; `profile_ramp` requires `ramp_ref.input_signal=shadow_attenuation`; `continuous` requires canonical omission of both response fields.
- `ToonEnergyPolicyV1.kind=physically_bounded` requires both maximum multipliers to be at most `1`; `stylized_bounded` permits values above `1` only up to the declared finite cap. Every qualified fixture checks non-negative finite response and the declared cap over its registered sweep domain.
- `feature_semantics[]` roles are unique.
- When either `ToonFeatureSemanticV1` source is `role_profile`, the exact referenced `ToonFeatureRoleProfileV1` must match its role, Material Domain, and selected World compatibility. Only the corresponding normal or shadow side resolves through that profile; a non-`role_profile` side remains unchanged. `mesh_normal`／`engine_shadow` require texture role `none`; the other closed sources require their matching `normal`／`bent_normal`／`shadow_mask`／`signed_distance_field` role. Texture roles resolve to typed Material bindings, never names. `ToonFeatureRoleProfileV1.profile_content_hash` uses the `MIRAKAN_TOON_FEATURE_ROLE_PROFILE_V1` domain with self-excluding length-framed canonical bytes.
- Face or hair behavior is never inferred from asset names or texture filenames.
- `ToonRimResponseV1.mode=disabled` requires canonical zero width, softness, and intensity.
- A profile does not create a Render pass, resource, or Backend object.
- Unsupported Target behavior requires an explicit qualified fallback or returns `capability_unavailable`.
- Fallback refs are strict priority order, unique, and acyclic over the transitive fallback graph. The resolver selects the first compatible, Target-qualified ref; a missing eligible candidate is a blocking failure, not a silent style substitution.

### 4.3 Outline style profile

```text
OutlineStyleProfileV1
  profile_id: StableId
  profile_version: positive uint32
  compatible_world_space: WorldSpaceCompatibilityV1
  technique_preference:
    geometry_only | screen_space_only | hybrid_qualified | disabled
  geometry_width_semantic:
    none | object_relative | world_meters
  geometry_width_value: finite nonnegative value
  screen_width_pixels: finite nonnegative value
  depth_threshold: finite nonnegative value
  normal_threshold: finite value in [0, 1]
  color: LinearColor4f
  occlusion_policy:
    visible_only | silhouette_and_crease | include_occluded
  temporal_policy:
    no_history | stable_history_required
  alpha_policy:
    respect_coverage | opaque_silhouette_only
  required_capability_refs[0..16]: McdContractRefV1(kind=capability)
  fallback_profile_refs[0..8]: OutlineStyleProfileRefV1
  profile_content_hash: SHA-256

OutlineStyleProfileRefV1
  profile_id: StableId
  profile_version: positive uint32
  profile_content_hash: SHA-256
```

The profile contains style intent only. `compatible_world_space` declares reusable-profile compatibility but never selects a structural dimension or hybrid gameplay authority. An Engine-owned resolver first validates the exact selected World profile, then maps the intent to a qualified Render Graph technique based on Target, anti-aliasing plan, depth／normal availability, budget, and fallback policy. It never exposes a native geometry-expansion or screen-space implementation as a project API.

`geometry_only` requires a non-`none` geometry width, zero screen width／depth threshold／normal threshold, and `temporal_policy=no_history`. `screen_space_only` requires `geometry_width_semantic=none`, a positive screen width, and nonzero depth or normal threshold. `hybrid_qualified` requires both a non-`none` geometry width and a positive screen width. `disabled` requires all widths and thresholds to be zero, `temporal_policy=no_history`, and an empty Capability／fallback set.

Outline fallback refs are strict priority order, unique, acyclic, and Target-qualified before the resolver can select one. `profile_content_hash` is `SHA-256(ASCII "MIRAKAN_OUTLINE_STYLE_PROFILE_V1" || uint32_be(length(receipt-free canonical bytes excluding profile_content_hash)) || canonical bytes)`.

Render Graph owns the concrete `OutlineTechniqueCatalogV1`, `OutlineInputAvailabilityV1`, `OutlineResolutionDiagnosticV1`, and `ResolvedOutlineExecutionPlanV1` contracts. The catalog binds each non-disabled technique to World compatibility, required inputs, AA／history support, Capability, and fresh qualification; the resolver consumes the read-only `RendererBudgetEnvelopeV1` owned by Runtime Performance／Capacity. `disabled` is a canonical null-technique plan, not an unrecorded fallback.

`VisualStyleProfile` replaces the untyped `outline_profile_id` with exact `outline_style_profile_ref`. It may optionally reference `toon_shading_profile_ref`, which is required when any selected Material Template uses a Toon Shading Model.

### 4.4 Qualification

Toon qualification includes:

- sphere, face, hair, transparent hair, eye, cloth, and foliage fixtures;
- analytic-band and ramp-asset input-signal／filter／clamp／channel boundaries;
- Toon Feature Role Profile role／texture-role／World-compatibility positive and negative cases;
- directional, point, spot, environment, key, fill, rim, and accent lights;
- cast／receive shadow combinations and self-shadow extinction;
- geometry-only, screen-space-only, hybrid-qualified, and disabled outline intent;
- fixed and dynamic resolution;
- no AA, MSAA, spatial AA, and temporal AA compatibility;
- 2D, 3D, and hybrid World profiles;
- Windows, Android, and Apple Target-specific fallback behavior;
- exact threshold boundary, finite-value, missing-ramp, stale-profile, domain-mismatch, and unsupported-Capability negative cases.

Golden-image evidence supplements typed invariants. It does not replace schema, capability, budget, or interface validation.

## 5. Typed shader IR and bounded HLSL

### 5.1 Understanding modes

```text
ShaderUnderstandingModeV1
  typed_ir
  bounded_hlsl
```

Every `ProjectShaderModuleV1` declares exactly one mode.

### 5.2 Typed IR mode

`TypedShaderIrV1` is the canonical semantic source for Engine-generated shader text.

```text
TypedShaderIrV1
  ir_id: StableId
  ir_version: positive uint32
  module_interface_ref
  entry_points[1..128]
  typed_values[0..65536]
  typed_resources[0..4096]
  nodes[1..65536]
  control_regions[0..16384]
  call_edges[0..131072]
  value_edges[0..262144]
  resource_access_edges[0..65536]
  side_effects[0..4096]
  variant_dimensions[0..64]
  source_origin_intents[0..65536]
  ir_content_hash: SHA-256
```

Required properties:

- closed node and operation catalog;
- explicit value type, unit, coordinate space, color space, range, and precision;
- structured finite control flow only;
- resource access and side effects declared before code generation;
- deterministic canonical order and hash;
- generated HLSL and source maps derived from the exact IR revision;
- no handwritten source substitution after generation.
- `ir_content_hash` is `SHA-256(ASCII "MIRAKAN_TYPED_SHADER_IR_V1" || uint32_be(length(receipt-free canonical bytes excluding ir_content_hash)) || canonical bytes)`; all arrays are strictly sorted by their stable edge or node key and reject duplicates.

For `typed_ir`, U0 Identity, U1 Structure, and U2 Semantics take the exact IR as their authority. Generated source maps and DXC reflection verify identity and observable compiled-interface agreement. Behavioral equivalence of the generated artifact is evaluated by registered fixtures and measurements rather than inferred from reflection.

### 5.3 Bounded HLSL mode

`bounded_hlsl` keeps advanced handwritten Project HLSL within `BoundedProjectShaderProfileV1`.

Its guarantees are limited to:

- declared public interface and semantic manifest;
- compiler success and diagnostics;
- public reflection for entry signatures, constants, resources, and compiled interface;
- validator and artifact inspection facts;
- declared bounded-resource and control restrictions;
- registered fixture behavior and measured budgets;
- explicit Code Owner review.

Compiler-private AST text, debug-container internals, AI explanation, comments, or source naming are not authoritative semantic IR.

### 5.4 Understanding closure correction

Coverage records are explicit and revisioned.

```text
ShaderVariantSelectionV1
  variant_id: StableId
  selected_value: one closed value of the exact dimension

ShaderVariantTupleV1
  selections[0..64]: ShaderVariantSelectionV1
  tuple_content_hash: SHA-256

ShaderVariantTupleRefV1
  module_ref: exact ProjectShaderModuleRefV1
  tuple_content_hash: SHA-256

ShaderCoverageCaseKeyV1
  target_profile_ref:
    exact {target_profile_id, target_profile_version, target_profile_content_hash}
  variant_tuple_ref: exact ShaderVariantTupleRefV1
  fixture_ref:
    exact {fixture_id, fixture_version, fixture_content_hash}

ShaderBehaviorCoverageCaseV1
  case_key: ShaderCoverageCaseKeyV1
  analytic_invariant_refs[0..256]
  visual_invariant_refs[0..256]
  measurement_receipt_refs[1..256]
  result: pass

ShaderBehaviorCoverageV1
  coverage_id: StableId
  coverage_version: positive uint32
  module_ref: exact ProjectShaderModuleRefV1
  fixture_set_ref:
    exact {fixture_set_id, fixture_set_version, fixture_set_content_hash}
  cases[1..4096]: ShaderBehaviorCoverageCaseV1
  coverage_content_hash: SHA-256

ShaderBehaviorCoverageRefV1
  coverage_id: StableId
  coverage_version: positive uint32
  coverage_content_hash: SHA-256

ProjectShaderPassRefV1
  technique_ref:
    exact {technique_id, revision, technique_content_hash}
  pass_id: StableId

ProjectShaderResourceRefV1
  technique_ref:
    exact {technique_id, revision, technique_content_hash}
  resource_id: StableId

ShaderChangeImpactCoverageV1
  coverage_id: StableId
  coverage_version: positive uint32
  module_ref: exact ProjectShaderModuleRefV1
  dependency_graph_hash: SHA-256
  required_module_refs[0..1024]: ProjectShaderModuleRefV1
  required_pass_refs[0..4096]: exact ProjectShaderPassRefV1
  required_resource_refs[0..4096]: exact ProjectShaderResourceRefV1
  required_case_keys[1..4096]: ShaderCoverageCaseKeyV1
  covered_behavior_ref: ShaderBehaviorCoverageRefV1
  coverage_content_hash: SHA-256

ShaderChangeImpactCoverageRefV1
  coverage_id: StableId
  coverage_version: positive uint32
  coverage_content_hash: SHA-256
```

The Module declares closed Variant Dimensions and `allowed_variant_tuples[]`; each tuple contains every declared Dimension exactly once, is strict-sort／unique, and has a `MIRAKAN_SHADER_VARIANT_TUPLE_V1` self-excluding length-framed hash. The required sets are derived from the exact canonical IR graph in `typed_ir` mode and from the declared／observed fact graph in `bounded_hlsl` mode. A behavior case covers only its exact Target／Variant／Fixture tuple; separate arrays never imply a Cartesian product. The compiled artifact, Fact Graph, runtime-use trace, and measurement receipt for that case carry the same exact Target／Variant ref. Case keys and all required reference arrays are strict-sort／unique. Every case's `variant_tuple_ref.module_ref` equals `ShaderBehaviorCoverageV1.module_ref`; `ShaderChangeImpactCoverageV1.module_ref` equals the module resolved by `covered_behavior_ref`; and `required_case_keys[]` must be an exact subset of passing cases in `covered_behavior_ref`. `ShaderBehaviorCoverageV1.coverage_content_hash` and `ShaderChangeImpactCoverageV1.coverage_content_hash` use their respective `MIRAKAN_SHADER_BEHAVIOR_COVERAGE_V1` and `MIRAKAN_SHADER_CHANGE_IMPACT_COVERAGE_V1` ASCII domains with self-excluding, length-framed canonical bytes. Coverage records do not grant authorization or promotion authority.

`ShaderUnderstandingClosureV1` adds:

```text
module_ref: exact ProjectShaderModuleRefV1
fixture_set_ref:
  exact {fixture_set_id, fixture_set_version, fixture_set_content_hash}
understanding_mode: typed_ir | bounded_hlsl
structural_authority:
  typed_ir_canonical | declared_and_observed_hlsl
behavior_coverage_ref: ShaderBehaviorCoverageRefV1
change_impact_coverage_ref: ShaderChangeImpactCoverageRefV1
review_requirement:
  engine_generated | code_owner_required
```

The Closure module ref must occur exactly once in its input closure. Its behavior／change-impact coverage refs, the latter's covered-behavior ref, and its fixture-set ref must all resolve to the same Module／fixture set. Mode, structural authority, and review requirement are derived only from that Module's declared mode; cross-Module, cross-mode, or cross-fixture substitution is blocking.

U-level interpretation becomes:

| Level | `typed_ir` | `bounded_hlsl` |
|---|---|---|
| U0 Identity | exact IR／source／artifact closure | exact source／manifest／artifact closure |
| U1 Structure | complete canonical IR graph | declared and observed required fact set; no claim of complete source semantics |
| U2 Semantics | complete public IR semantics | complete declared public semantics plus matching observed interface facts |
| U3 Behavior | registered fixture and measurement coverage | registered fixture and measurement coverage |
| U4 Change impact | exact structural dependency closure plus registered behavior coverage | declared／observed dependency closure plus registered behavior coverage and Code Owner review |

The phrase “required set recall 100%” applies only to the canonical dependency graph and registered coverage set. It does not claim that arbitrary visual consequences outside registered fixtures are mathematically complete.

Promotion rules:

- `typed_ir` may become eligible for normal automated staging after all required gates pass.
- `bounded_hlsl` always requires the Project Shader Code Owner approval gate.
- Missing or stale coverage produces `MIRAKAN-SHADER-UNDERSTANDING_INCOMPLETE`.
- Reflection success never upgrades `bounded_hlsl` to `typed_ir`.

## 6. Data flow

### 6.1 World and style

```text
WorldDocumentV1
  -> WorldSpaceProfileRefV1
  -> VisualStyle／Camera／Physics／Navigation compatibility validation
  -> typed resolved plans
  -> Cooked artifacts
```

No consumer rewrites the World profile. Style changes produce presentation proposals only.

### 6.2 Toon

```text
VisualStyleProfile
  -> ToonShadingProfileV1
  -> Material definition／template resolution
  -> Lighting response validation
  -> Material IR
  -> generated shader artifact

VisualStyleProfile
  -> OutlineStyleProfileV1
  -> Target／AA／budget resolver
  -> qualified Render Graph technique
```

### 6.3 Project shader

```text
typed_ir:
TypedShaderIrV1 -> generated HLSL -> DXC -> reflection／artifact
        |                                |
        +-- source identity and mapped --+
            compiled-interface agreement

bounded_hlsl:
Manifest + HLSL -> validator／DXC／reflection／fixtures
       -> declared-and-observed understanding closure
       -> Code Owner review
```

## 7. Failure behavior

The following failures are blocking and source-preserving:

- World／Style／Camera／Physics／Navigation dimension incompatibility;
- Toon profile domain mismatch;
- missing or stale ramp／role／outline profile;
- unsupported Toon or outline Capability without qualified fallback;
- non-finite, non-canonical, or invalid band thresholds;
- Typed IR versus generated-source identity mismatch, or a mapped public compiled-interface mismatch;
- handwritten changes to generated HLSL;
- bounded HLSL missing Code Owner review;
- stale behavior or change-impact coverage;
- any attempt to treat DXC internal AST output as the public semantic authority.

No failure silently changes dimension, shading model, style, outline technique, Target artifact, or fallback.

## 8. Canonical documents affected

The architecture correction is reflected only in the following canonical documents:

- `docs/architecture/03-authoring/project-state.md`
  - register World Space Profile references in the Project document model;
  - keep UI-only／headless projects valid without a World profile.
- `docs/architecture/05-simulation/physics.md`
  - consume exact World Space Profile refs instead of copied dimension fields.
- `docs/architecture/05-simulation/navigation.md`
  - bind 2D grid／3D navmesh selection to World Space Profile refs.
- `docs/architecture/06-rendering/world.md`
  - own `WorldSpaceProfileV1` and bind it from `WorldDocumentV1`.
- `docs/architecture/06-rendering/camera.md`
  - replace structural selection with compatibility constraints.
- `docs/architecture/06-rendering/materials.md`
  - remove structural dimension ownership;
  - add World compatibility, Toon, and Outline semantic contracts.
- `docs/architecture/06-rendering/lighting.md`
  - consume Toon response contracts without copying Material parameters.
- `docs/architecture/06-rendering/render-graph.md`
  - resolve Outline intent to qualified execution techniques.
- `docs/architecture/06-rendering/project-shader.md`
  - add understanding modes, Typed IR authority, and bounded-HLSL limits.
- `docs/architecture/02-foundation/toolchain-dependencies.md`
  - state the DXC public evidence boundary explicitly.
- `docs/architecture/01-governance/ai-verification-provenance.md`
  - distinguish canonical structural coverage from empirical behavior coverage.
- `docs/architecture/00-product/product-plan.md`
  - preserve Phase ordering while reflecting the corrected contracts.
- `docs/architecture/04-runtime/performance-capacity.md`
  - own the read-only Renderer／Lighting budget envelopes consumed by the corrected resolvers.

No implementation plan is created as part of this correction.

## 9. Acceptance criteria for the architecture correction

The canonical-document update is complete only when:

1. `VisualStyleProfile` no longer owns or selects structural scene dimension.
2. `WorldDocumentV1` resolves exactly one `WorldSpaceProfileV1`.
3. UI-only and headless Runtime Entries remain valid without a World.
4. all copied structural dimension fields in affected consumer contracts are replaced by an exact World profile ref or a reusable compatibility constraint;
5. Toon surface response and outline intent have complete typed profiles with ownership and failure rules;
6. Visual Style references Toon／Outline profiles without owning Render passes;
7. Project Shader distinguishes `typed_ir` and `bounded_hlsl`;
8. DXC reflection is limited to observed compiler-interface evidence;
9. every behavior-coverage record binds Target, Variant, Fixture, invariants, and measurements as one exact case rather than implying a Cartesian product;
10. U1／U4 completeness language is limited to canonical graphs and registered coverage;
11. bounded HLSL always requires Code Owner review;
12. external-engine sources are comparison evidence only; no external API schema, serialized format, source code, shader text, or compatibility claim is adopted;
13. no document claims that the design is implemented, activated, qualified, or Production-ready;
14. all affected references, diagnostic ownership, Target fallback rules, and qualification fixtures are internally consistent;
15. no compatibility migration or schema V2 is added before an actual persisted V1 implementation exists;
16. no implementation plan or source-code task is created.
17. every introduced Profile／Catalog／resolver input and diagnostic type has a concrete owner and schema rather than an unresolved reference;
18. Outline compatibility, catalog qualification, input availability, AA／history, budget, disabled state, and diagnostic output are all explicit;
19. a shader behavior case resolves a readable declared Variant tuple, and all coverage／closure refs bind to the same exact Module and fixture set.

## 10. Explicit non-goals

- implementing Renderer, Material, Shader compiler, Editor, AI, or Gateway code;
- creating C++／HLSL files or tests;
- activating Material or Project Shader Operations;
- selecting a Production Toon aesthetic;
- guaranteeing arbitrary handwritten HLSL semantic completeness;
- adding a new shader language or replacing DXC;
- creating migration bytes for an implementation that does not exist;
- scheduling Work Packages, assigning people, estimating dates, or creating an implementation plan.
