# Miraikanai Engine Project Shader Contract

- 文書ID: mirakan.arch.rendering-project-shader
- 状態: review
- 正本範囲: bounded Project Shaderの自由度境界、HLSL source／semantic module、Project Node／Shading Model、Project Shader Technique、宣言的resource／pass、compile fact、AI context／operation、Shader Understanding Closure、Shader固有diagnostic／qualification
- 非正本範囲: Engine shader実装、native graphics API、Render Graph実行algorithm、Material／Lighting／Post Process／VFXのauthoring意味、Tool／compilerのexact version、AI authorization／Approval、共通Evidence envelope、Project transaction。各Owner文書を参照する
- 依存: [Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Math／Core Utilities](../02-foundation/math-core.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[Render Graph](render-graph.md)、[Materials](materials.md)、[Lighting](lighting.md)、[Post Processing](post-processing.md)、[VFX authoring](vfx-authoring.md)
- 外部根拠検証日: 2026-07-22

## 1. 結論と自由度境界

Project Shaderは、署名済みimmutable Engine baselineとnative Backendを変更せず、Engine-owned Portへ宣言的に接続する範囲で表現力を最大化する。ProjectはHLSL function／library、Material Node、既存Domain内のShading Model、Raster／Compute／Ray Stage、宣言済みStorage resource、複数Pass Techniqueを追加できる。

「最大自由度」はnative APIを無制限に公開する意味ではない。Projectが変更できるalgorithm／data flow／pass compositionを最大化し、resource lifetime、queue／barrier、descriptor、compiler option、Backend objectの所有をEngineへ固定する意味である。既存のEngine-owned Portで表現できない新Backendまたはnative API extensionはGame制作中に迂回せず`capability_unavailable`とする。

HLSL textだけを正本またはAI理解の証拠にしない。一つのProject Shader revisionは、意味を宣言する`ProjectShaderModuleV1`または`ProjectShaderTechniqueV1`、その実装Source、compilerが観測した`ShaderFactGraphV1`、Target artifact、fixture／measurement Evidenceのclosureである。宣言と観測事実が一致しないCandidateをcompile成功だけで昇格しない。

Shipping Runtimeはoffline compiled artifactだけを読む。Shader source、authoring semantic、compiler、AST、JIT、runtime download、未承認variant生成をPackageへ含めない。

## 2. Canonical objectとidentity

| Object | 所有する意味 | 生成／変更主体 |
|---|---|---|
| `BoundedProjectShaderProfileV1` | Source language、公開Shader SDK、許可Stage／Port／Resource／副作用／上限、Target、Compiler Profile、Qualification Policy | Engine buildが生成し、Project／AI変更不可 |
| `PublicShaderSdkCatalogV1` | Project HLSLから参照できる全Engine公開symbolの宣言、semantic、効果、Capability、Target、Evidence | Engine buildが生成し、Project／AI変更不可 |
| `ProjectShaderModuleV1` | Function、Node、Stage、Libraryのsemantic interfaceとSource closure | Project Source。R3 ChangeSet |
| `ProjectShaderTechniqueV1` | Engine-owned injection Portへ接続するlogical resourceとPass DAG | Project Source。R3 ChangeSet |
| `ProjectShaderNodeCatalogV1` | Qualification済みProject Node／Shading Modelの検索、型、意味、Target、cost | Module／Technique artifactから生成 |
| `ShaderFactGraphV1` | Compiler／reflection／validator／runtime-use traceが観測した構造と効果 | Trusted Shader Workerが生成 |
| `ShaderContextSliceV1` | AI／Editorへ返すboundedなModule／Technique／Fact／Evidence projection | Query Serviceが生成 |
| `ShaderUnderstandingClosureV1` | Shader固有のIdentity、Structure、Semantic、Behavior、Change-impact合否 | Trusted Verification Runnerが生成 |
| `ProjectShaderArtifactSetV1` | Target別binary、reflection、source map、interface、fact、receiptのhash closure | Cook／Build Serviceが生成 |

全Source objectはStable ID、schema version、revision、content hash、Project namespaceを持つ。Project namespaceは`project.<project_id>.*`に固定し、Engine IDを上書きしない。表示名、file名、function名、配列indexをidentityにしない。

`BoundedProjectShaderProfileV1`は最低限、次を持つ。

```text
schema_version
profile_id
engine_baseline_hash
source_language_profile_id
public_shader_sdk_source_hash
public_shader_sdk_catalog_hash
allowed_module_kinds[]
allowed_stages[]
allowed_technique_port_ids[]
allowed_resource_classes[]
allowed_side_effect_classes[]
allowed_capability_ids[]
target_profile_ids[]
compiler_profile_ids[]
limit_profile_ref
qualification_policy_hash
```

Profile外のStage、Port、Resource、side effect、Capability、Target、Compiler ProfileをSource宣言で追加できない。Profile更新は署名済みEngine baseline更新でありProject変更ではない。

`PublicShaderSdkCatalogV1`はCatalog ID、Engine baseline hash、public Shader SDK source hashと、全public entryを持つ。各entryはStable SDK Symbol ID、HLSL symbol／kind／declaration、typed value／resource interface、semantic role、unit／coordinate／color space、side effect、許可Stage、required Capability、Target support、determinism class、cost class、invariant、example／counter-example、生成元declaration／documentation hashを持つ。Projectから参照可能なfunction、type、constant、generated binding helperを100%列挙し、Engine private symbolは一件も含めない。SourceがCatalogにないEngine symbolへ解決した場合、compile可否にかかわらずProfile違反にする。

## 3. Capability ladder

Project Shaderの表現力を次のclosed levelで分類する。Levelは品質順位ではなく必要な契約とRiskの段階である。

| Level | Projectが追加できるもの | 必須接続 |
|---|---|---|
| S0 | Material Instance parameter／texture binding | `MaterialParameterSemanticV1` |
| S1 | closed typed Material Graph | `MaterialNodeCatalogV1` |
| S2 | HLSL function、Project Material Node、Shader Library | `ProjectShaderModuleV1` |
| S3 | 既存Material Domain内のProject Shading Model、Stage Module | Domain Output Contract＋`ShaderInterface` |
| S4 | Raster／Compute／Rayの複数Pass Technique | `ProjectShaderTechniqueV1`＋Engine-owned Technique Port |
| S5 | Project Renderer Feature | `ProjectRenderDomainPortV1`＋declarative Technique。native callback不可 |

S2以上は`ProjectShaderNodeCatalogV1`へProject namespaceで登録できる。Catalog entryはQualification済みartifactだけを参照し、Sourceの存在だけで利用可能にしない。S3のShading Modelは既存Domainの入力／出力、depth、alpha、motion、shadow、fallback契約を満たす。S5の新Domainは`ProjectRenderDomainPortV1`のclosed input／output semanticsで完全に表現できる場合だけ許可し、Port自体、Backend、native execution primitiveの追加はEngine Extensionとする。

S5の「Project Renderer Feature」は`project_render_domain` Portへ一つ以上のS4 Techniqueを接続したProject namespaceのFeature Bundleを指す。EngineのMaterial Domain enum、Render phase、Port Catalog、Backendを追加する意味ではない。Portのclosed semanticで不足する入力／出力／orderingが一つでもあればS5として近似せず`capability_unavailable`にする。

## 4. `ProjectShaderModuleV1`

```text
ProjectShaderModuleV1
  schema_version
  module_id
  revision
  namespace
  module_kind: function | material_node | shader_stage | shader_library
  purpose
  semantic_role_ids[]
  source_files[]
    project_relative_path
    content_hash
  dependency_module_refs[]
  exports[]
  entry_points[]
  value_interfaces[]
  resource_interfaces[]
  side_effects[]
  required_capability_ids[]
  target_support[]
  variant_dimensions[]
  budget_contract
  invariant_refs[]
  fixture_refs[]
  fallback_refs[]
  provenance_refs[]
```

`exports[]`はStable Export ID、HLSL symbol、kind、visibilityを持つ。public exportだけを別Module、Graph、Techniqueから参照できる。Module-relative private symbolを文字列名で外部接続しない。

`entry_points[]`はEntry ID、Stage、Export ID、threadgroup sizeまたはRaster／Ray interface、required Capabilityを持つ。Stageは`vertex | pixel | compute | mesh | amplification | ray_generation | ray_miss | ray_closest_hit | ray_any_hit | ray_intersection | callable`である。ProfileにないStageをSourceだけで有効化しない。

### 4.1 Value semantic

`ShaderValueSemanticV1`は次を持つ。

```text
value_id
direction: input | output | inout
value_type
semantic_kind
unit_id
coordinate_space
color_space
valid_range
precision_class
interpolation
default_value: optional
```

`semantic_kind`は`generic | color | position | direction | normal | tangent | texcoord | depth | motion | coverage | opacity | emissive | radiance | irradiance | attenuation | mask | index | count | time | distance | angle`、`coordinate_space`は`none | object | world | view | clip | tangent | uv | screen_pixel | screen_normalized`、`color_space`は`none | linear_rgb | srgb | data`である。物理単位は[Math／Core Utilities](../02-foundation/math-core.md)のUnit IDを参照し、同じ値へ単位名と倍率を二重保存しない。Normal／Direction／Position、linear／sRGB／dataのimplicit接続を禁止する。

`precision_class`は`exact_integer | fp32_required | fp16_allowed`であり、Backend native precision名をSource契約にしない。`interpolation`は`not_applicable | constant | flat | linear | perspective | no_perspective | centroid | sample`である。Stageに不適合な値を拒否する。

### 4.2 Resourceとside effect

`ShaderResourceInterfaceV1`はResource ID、semantic role、kind、element／texture type、format class、dimension、array bound、access、stage visibility、binding frequency、optional／required、Sampler semanticを持つ。`access`は`read | write | read_write | atomic`である。

宣言済みStorage Texture／Buffer、UAV相当のwrite／read_write、bounded atomicを許可する。native descriptor、register番号、GPU virtual address、Backend handleはSourceへ保存せず、generated bindingが決める。Resource array、dynamic index、indirect argumentはProfileのfinite boundを必須とする。

`side_effects[]`は`none | discard | depth_write | storage_write | atomic | group_memory_barrier | device_memory_barrier | ray_trace | indirect_argument_write`のclosed setである。Source／Fact Graphに現れたside effectが宣言にない場合、または宣言したeffectがStage／Port／Targetで許可されない場合は拒否する。

`none`は単独値だけを許可し、他のeffectと併記しない。`side_effects[] = [none]`のModuleからwrite／atomic／barrier／discard／ray／indirect effectを観測した場合も宣言不一致として拒否する。

### 4.3 Variant、Target、Budget

`ShaderVariantDimensionV1`はVariant ID、kind、closed values、default、selection source、mutual-exclusion groupを持つ。kindは`boolean | enum | target_capability | quality_profile`である。自由文字列keyword、runtime文字列、未列挙macro値をVariant identityにしない。全DimensionのCartesian productを暗黙生成せず、`allowed_variant_tuples[]`を列挙しProfile上限と使用closureの両方へ適合させる。

`ShaderTargetSupportV1`はTarget Profile ID、support state、required Capability IDs、Compiler Profile ID、Fallback Stable ID、Qualification Receipt refを持つ。support stateは`required | optional | unsupported`である。`unsupported`へArtifactまたは空Fallbackを対応付けず、`required`は全Gate合格を必須とする。

`ShaderBudgetContractV1`はTarget／Quality別に`max_pass_count`、`max_logical_resource_count`、`max_resource_array_elements`、`max_variant_count`、`max_threadgroup_threads`、`max_dispatch_elements`、`max_transient_bytes`、`max_persistent_bytes`、`max_predicted_gpu_us`、`max_compile_milliseconds`、`runtime_budget_envelope_ref`を持つ。値は[Runtime performance／capacity](../04-runtime/performance-capacity.md)とEngine-generated Profileから解決し、ProjectがTarget上限を引き上げない。Estimateとinstrumented peakを別FieldでReceiptへ記録し、Estimate以下であることだけを合格証拠にしない。

## 5. `ProjectShaderTechniqueV1`

```text
ProjectShaderTechniqueV1
  schema_version
  technique_id
  revision
  namespace
  purpose
  technique_kind: raster | compute | ray | mixed
  injection_port_id
  view_scope
  logical_resources[]
  passes[]
  explicit_dependencies[]
  required_capability_ids[]
  target_support[]
  budget_contract
  fixture_refs[]
  fallback_refs[]
  provenance_refs[]
```

`injection_port_id`はEngine-owned closed IDであり、少なくとも`material_surface | shadow | post_process_pre_tonemap | post_process_post_tonemap | vfx_simulation | vfx_render | environment_surface | project_render_domain`を区別する。Portは利用可能なScene／View semantic、許可出力、Layer、history、ordering boundaryを定義し、Projectが文字列stage名で挿入位置を作らない。

`ProjectShaderTechniquePortV1`はPort ID、version、allowed technique kinds／Stages、typed input／output semantic、import可能Resource role、export必須Resource role、allowed Layer、history policy、ordering predecessor／successor、allowed side effect、Target support、Budget Profile refを持つ。Port entryはEngine buildが生成し、Project／AI変更不可である。`ProjectRenderDomainPortV1`は`port_id = project_render_domain`のentryであり、同じSchemaを使う別名の契約を作らない。

`ProjectShaderLogicalResourceV1`はResource ID、semantic role、kind、ownership、format class、extent class、mip／layer／sample bound、usage、initialization rule、history key rule、byte estimateを持つ。ownershipは`transient | persistent | history | imported_port_input | exported_port_output`である。alias group、native heap、barrier stateをSourceへ持たない。

`ProjectShaderPassV1`はPass ID、pass type、queue intent、view scope、module Entry refs、accesses、color／depth attachment、dispatch／draw envelope、explicit dependency、side-effect class、declared costを持つ。pass typeは`raster | compute | ray`、queue intentは`graphics | async_compute_preferred`である。Queue確定、barrier、alias、merge、cullingはRender Graphが決める。

Techniqueは宣言済みPassとResourceだけを使用する。Sourceまたはruntime-use traceがManifest外Resource、追加dispatch、native barrier、別queue workを観測した場合、Technique全体を不合格にする。Running中の不一致は該当Techniqueの以後のpass／submissionを停止し、同一frameへfallbackを挿入しない。承認済みfallbackがある場合だけ次のGraph generationから切り替える。

## 6. Portable HLSL source profile

初期`source_language_profile_id`は`portable_hlsl_2021_v1`である。exact compiler、translator、validator、commit、artifact hashは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)が固定し、本書へ複写しない。

許可するSource dependencyは同じModule closureのproject-relative file、Qualification済みProject Shader Module、Engine public Shader SDKだけである。absolute path、parent traversal、Engine private include、Vendor include、Network、generated build directory、environment-dependent includeを拒否する。

Preprocessor symbolはModule manifestのvariant dimension、Target Capability、Engine public Shader SDKが生成した値だけを条件入力にできる。Source-local helper macroは外部状態を読まず、variantを増やすmacroは全値をManifestへ列挙する。Project指定`#pragma`、register／space固定、compiler flag、Backend conditional、unknown extensionを禁止する。

Loopはcompile-time finite bound、またはbinding時にProfile上限へ検証されるbounded parameterを上限式に持つcanonical loopだけを許可する。無上限`while`／`do`、再帰、自己変更、device address、unbounded bindless、function pointerによる未列挙dispatchを禁止する。Wave／Subgroup、atomic、derivative、ray operation、indirect operationはCapabilityとside effectを宣言し、Target fallbackを持つ。

Matrix layout、clip／depth convention、texture coordinate origin、front-face、precision lowering、binding layoutはEngine public Shader SDKが生成する。Project SourceがTarget別の暗黙規約を再実装しない。

## 7. Compile、Fact Graph、contract conformance

Shader Workerは次の固定順でCandidateを処理する。

1. Profile、Schema、Source path、dependency closureを検証する。
2. Preprocessし、全include／macro input／variantをhashで固定する。
3. Source frontendでsymbol、type、call、control、resource candidateを抽出する。
4. Targetごとにcompile、validate、reflection、source mapを生成する。
5. DXIL、SPIR-V、MSL／Metal artifact間のinterface、resource、Stage、precision、Capabilityを照合する。
6. Manifest、compiler reflection、static fact、runtime-use traceを照合する。
7. fixture、parameter sweep、visual／analytic invariant、performance、fault testを実行する。
8. `ProjectShaderArtifactSetV1`とVerification Receiptを組み立てる。

`ProjectShaderArtifactSetV1`は次を固定する。

```text
artifact_set_id
project_revision
engine_baseline_hash
bounded_project_shader_profile_hash
public_shader_sdk_catalog_hash
module_hashes[]
technique_hashes[]
source_tree_hash
target_profile_hash
compiler_profile_hash
binary_artifacts[]
reflection_artifact_hash
shader_interface_hashes[]
shader_fact_graph_hash
source_map_artifact_hash
fixture_result_root_hash
performance_receipt_hashes[]
verification_receipt_hashes[]
provenance_root_hash
```

`binary_artifacts[]`はkind、content hash、size、entry／Stage setを持ち、native pathやtimestampをidentityにしない。Artifact Setは一Target専用であり、DXIL、SPIR-V、metallib等の別Target payloadを一つのRuntime選択肢として混在させない。Development-only reflection／source map／FactはShipping payloadへ含めず、Package Catalogから署名済みReceipt hashだけを参照する。

Shader binaryは実行codeに準じるTarget application artifactであり、[Asset lifecycle](../03-authoring/asset-lifecycle.md)のContent Package、Content Patch、DLC、managed Asset deliveryへ格納しない。Project Shader更新は対象TargetのApplication updateとして配布し、Application package validatorがArtifact Set、Target Profile、Engine baseline、Qualification Receipt、署名のhash closureを照合する。Material／VFX／Post等のContent artifactはShader binaryを内包せず、同じApplication build内のQualification済みArtifact IDだけを参照する。

`ShaderFactGraphV1`は少なくとも次を持つ。

```text
fact_graph_id
project_revision
profile_hash
public_shader_sdk_catalog_hash
module_or_technique_hash
source_tree_hash
preprocessed_source_hashes[]
symbols[]
public_sdk_symbol_refs[]
entry_points[]
call_edges[]
value_flow_edges[]
resource_accesses[]
side_effects[]
control_bounds[]
variant_facts[]
target_facts[]
source_map_refs[]
diagnostics[]
producer_tool_ids[]
```

各FactはStable Fact ID、kind、subject Stable ID、source span、producer、Target／variant scopeを持つ。Compilerのpublic reflection、validator結果、artifact inspection、runtime-use traceを合否証拠にする。Compiler内部ASTのtext dumpはversion固定Adapter内の補助Cacheに限定し、public contract、Stable ID、長期保存Artifact、単独の合否Authorityにしない。

宣言とFactの不一致、Target間interface差、source map欠落、同じ入力からFact Graph hash不一致をhard failureにする。Compilerの一Target成功、AIの説明、Source commentを不一致の代替証拠にしない。

## 8. AI／Editor operationとContext

canonical Operation IDは次である。

- `operation.shader.search`
- `operation.shader.read_module`
- `operation.shader.read_technique`
- `operation.shader.inspect_symbol`
- `operation.shader.find_callers`
- `operation.shader.explain_dataflow`
- `operation.shader.explain_resource_effects`
- `operation.shader.compare_targets`
- `operation.shader.preview`
- `operation.shader.parameter_sweep`
- `operation.shader.estimate_cost`
- `operation.shader.validate_contract`
- `operation.shader.plan_module`
- `operation.shader.plan_technique`
- `operation.shader.propose_module`
- `operation.shader.propose_technique`

Search／Read／Inspect／Explain／Compare／Estimate／ValidateはR0、Preview／Sweepは状態を正規Projectへ反映しないbounded Job、Planはread-only Proposal、ProposeはR3 Source ChangeSetである。Modelへraw filesystem write、compiler command、artifact publish、commit、activation、policy overrideを公開しない。

`ShaderContextSliceV1`はquery hash、Project revision、Profile hash、public Shader SDK Catalog hash、Module／Technique IDとrevision、selected Project symbols、参照するSDK Symbol entry、typed value／resource interfaces、call／value／resource／control edge、Pass relation、`source_excerpts[]`、Target／variant差、budget、diagnostic、fixture／Evidence summary、available Operation ID、omitted range、cursor、total countを持つ。`source_excerpts[]`はproject-relative path、file content hash、start／end span、UTF-8 Source text、truncation stateを持ち、別revisionのtextを同じSliceへ混在させない。配列／byte上限時に黙って切らずcontinuationを返す。

AIへはauthoring Sourceのexact excerpt、参照する全public SDK entry、それに対応するpreprocessor input／Fact／source mapを同時に渡す。generated binding、Backend source、Compiler内部AST textをauthoring Sourceとして返さない。Full fileが必要な場合も同じrevision／file hashを保持したbounded sliceをcontinuationで取得し、途中までのSourceを完全なModuleとして扱わない。Engine baselineまたはpublic Shader SDK Catalog hashが変われば、Source未変更でも旧Fact、Context、Understanding Closureをstaleにする。

AIは最初にManifestとbounded Fact projectionを読み、必要なsymbol／caller／resource／Target sliceだけを追加取得する。全Repository、全preprocessed source、全variantを一度のContextへ詰めない。人間がSourceを変更した場合、旧Fact、Context、Preview、Understanding Closureをstaleにし、再index前の仮定で上書きしない。

## 9. `ShaderUnderstandingClosureV1`

AIの「Shaderを理解した」という自己申告を状態にしない。Trusted Runnerが次を固定する。

```text
closure_id
project_revision
profile_hash
public_shader_sdk_catalog_hash
module_hashes[]
technique_hashes[]
source_tree_hash
fact_graph_hashes[]
target_profile_hashes[]
artifact_set_hashes[]
fixture_set_hash
u0_identity_result
u1_structure_result
u2_semantics_result
u3_behavior_result
u4_change_impact_result
verification_receipt_hashes[]
unresolved_blocker_ids[]
unsupported_capability_ids[]
disposition
runner_id
signature fields
```

| Level | 必須証拠 |
|---|---|
| U0 Identity | Module／Technique／Source／Profile／Target／Artifactのexact hashとStable ID一致 |
| U1 Structure | Entry、Project／SDK symbol、call、value／control flow、Pass、Resource、access、side effect、variantの必要集合を100%再現 |
| U2 Semantics | 全public value／resource／outputと参照SDK entryのtype、unit、space、color、range、effect、fallbackが完全でFactと一致 |
| U3 Behavior | 必須Fixtureのparameter sweep、counterfactual、analytic／visual invariant、cost classのstructured predictionが測定許容内 |
| U4 Change impact | 変更対象から影響Module／Pass／Resource／Variant／Target／Fixtureのrequired set recall 100%、未検証対象の安全断定0件、全必須Target Gate合格 |

各resultは`pass | fail | infrastructure_error | not_applicable_by_policy`である。必須Levelのpass以外、Blocker 1件以上、unsupported Capability、stale Evidence、Target欠落があれば`ready_to_stage`にしない。Level別hard failureを平均scoreで相殺しない。ClosureはEvidenceでありAuthorization、Approval、Promotion権限を与えない。

## 10. Diagnostic、failure、fallback

Project Shader固有DiagnosticはModule／Technique／Export／Entry／Pass／Resource／Value／Target／Variant Stable ID、source span、contract fact、observed fact、remediationを含む。

Diagnostic IDを次に固定する。

```text
MIRAKAN-SHADER-PROFILE_MISMATCH
MIRAKAN-SHADER-SOURCE_DEPENDENCY_FORBIDDEN
MIRAKAN-SHADER-SOURCE_PROFILE_VIOLATION
MIRAKAN-SHADER-SEMANTIC_INCOMPLETE
MIRAKAN-SHADER-INTERFACE_MISMATCH
MIRAKAN-SHADER-RESOURCE_UNDECLARED
MIRAKAN-SHADER-SIDE_EFFECT_UNDECLARED
MIRAKAN-SHADER-CONTROL_UNBOUNDED
MIRAKAN-SHADER-VARIANT_LIMIT
MIRAKAN-SHADER-CAPABILITY_UNAVAILABLE
MIRAKAN-SHADER-COMPILE_FAILED
MIRAKAN-SHADER-REFLECTION_MISMATCH
MIRAKAN-SHADER-TARGET_DIVERGENCE
MIRAKAN-SHADER-BUDGET_EXCEEDED
MIRAKAN-SHADER-FIXTURE_FAILED
MIRAKAN-SHADER-UNDERSTANDING_INCOMPLETE
MIRAKAN-SHADER-PREVIEW_STALE
MIRAKAN-SHADER-FALLBACK_REQUIRED
MIRAKAN-SHADER-UNAUTHORIZED_SOURCE
```

Running中にTechnique Manifestと実resource useが不一致になった場合、Rendererはdomain-neutral event `ProjectShaderTechniqueValidationFailed`を発行する。このeventはTechnique／Artifact Set／Target／Graph generation／Pass／Resource Stable ID、declared access／side effect、observed access／side effect、対応する上記Diagnostic ID、fact／runtime-use Evidence hash、停止したsubmission境界を持つ。新しいfallback、復旧pass、再試行権限はeventへ含めない。Shadow等のDomain ownerは同じeventを追加意味へ変換せずDomain projectionを付加できる。

compile失敗、missing resource、Capability不足、Budget超過時にdefault shader、resource drop、pass skip、別format、unlit、別Target artifactへ黙って置換しない。FallbackはSourceでStable ID、Target、意味差、visual／performance tolerance、Qualification Receiptを宣言する。

Editor Previewだけが直前のvalid artifactを`stale_last_valid`と明示して表示できる。Candidate失敗をPreview成功またはProduction利用可能と表示しない。ShippingはPackage検証時に不合格Artifact、missing fallback、interface mismatchを拒否する。

## 11. Qualificationと完了条件

最低Qualificationを次に固定する。

- Schema、Profile、namespace、path、dependency、public Shader SDK conformance。
- `PublicShaderSdkCatalogV1`の全public symbol coverage、declaration／documentation hash、Project参照closure、private symbol 0件。
- 許可／禁止HLSL、bounded loop、resource index、variant、atomic／wave／ray Capabilityのpositive／negative fixture。
- 全Targetのclean offline compile、validator、reflection、source map、artifact hash再現。
- Manifest対static fact／reflection／runtime-use traceのresource、side effect、interface一致。
- D3D12／Vulkan／Metalのbinding、matrix、clip／depth、precision、texture coordinate、Stage interface差分fixture。
- Graph／Module／Techniqueのcycle、missing producer、unordered write、history、fallback、capacity fixture。
- Parameter sweep、counterfactual、NaN／Inf、range端、zero／max resource、deterministic seed。
- Reference image、analytic invariant、Target別visual tolerance、before／after diff。
- instruction、occupancy、bandwidth、transient／persistent byte、variant、compile time、runtime GPU budget。
- timeout、Worker crash、corrupt Source／artifact、stale revision、device loss、runtime-use mismatch、fallback切替のfault fixture。
- AI EvalのU0～U4、存在しないsymbol／resource／Target抑制、stale Context拒否、禁止Operation拒否、seeded contract／source mismatch検出。

AI Evalはpublic、holdout、adversarial、incident corpusを分離し、各required Caseを3回実行する。hard gate違反、無権限Commit、存在しないStable IDの最終提出、未対応Capabilityの成功表示、Manifest外Resource／Passの見逃しは0件とする。required U0～U4、Target／fallback、Preview／undo／redo／recookのexact一致は全Case／全runで100%を要求する。

Runtime source compile、undeclared resource、unbounded control／resource／variant、Target artifact混在、stale interface、silent fallback、Understanding Closure欠落が残るCandidateをRelease候補にしない。

全Gateの合否は[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)が所有する`ProjectShaderQualificationReceiptV1`へ束ねる。`ShaderUnderstandingClosureV1`だけ、compile成功だけ、Material／RendererのDomain ReceiptだけをProject Shader Qualificationの代替にしない。

## 12. 所有権と連携

- [Materials](materials.md)はMaterial Domain、Shading Model入力／出力、parameter／node意味を所有し、Project Module／Node／Shading ModelのSource境界は本書を参照する。
- [Render Graph](render-graph.md)はTechnique Manifestからcanonical Pass／Resource plan、queue、barrier、alias、lifetime、submissionを生成し、Project Sourceを解釈し直さない。
- [Post Processing](post-processing.md)、[VFX authoring](vfx-authoring.md)、[Lighting](lighting.md)は各DomainのIntent／Plan／semantic Portを所有し、native PassまたはShader Sourceを直接受けない。
- [Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)はcompiler／translator／validatorのexact pinと`ShaderCompilerProfileV1`を所有する。
- [Executable contracts](../02-foundation/executable-contracts.md)はOperation envelope、Schema projection、Task state、Diagnostic共通形を所有する。
- [AI Security／Approval](../01-governance/ai-security-approval.md)はR3、A1、Worker、Authorization、Approval、Promotionを所有する。
- [AI Verification／Provenance](../01-governance/ai-verification-provenance.md)はReceipt、Evidence、Eval corpus、Provenance、freshnessを所有する。

## 13. 有名Engine／Shader ecosystemとの比較と採用判断

| 比較対象 | 公式設計から確認した点 | Miraikanaiで採用する点 | そのまま採用しない点 |
|---|---|---|---|
| Unreal Engine | Global ShaderがMaterial外のShader拡張を担い、RDGがShader Parameter StructとResource dependencyを検証する | Material Moduleと複数Pass Techniqueを分離し、typed parameter／resource宣言をRender Graph validationへ接続する | Project module、native render callback、Backend objectをGame制作AIへ公開する方式 |
| Unity Shader Graph／URP | Custom Function Nodeがtyped PortからHLSL functionを呼び、URP RenderGraph passがinjection pointとresource useを宣言する | S2 ModuleとS4 Technique Portの二層、Project Catalog登録、宣言ResourceからのPass展開 | C# callbackまたはSource reflectionだけを長期正本／Qualification証拠にする方式 |
| Godot Visual Shader | Custom Nodeがtyped input／outputと生成codeを分離し、Shader APIがparameter情報を公開する | typed Project Node、Stable Catalog、Editor projection | 自由code textとparameter一覧だけでsemantic／behavior理解を完了とする方式 |
| MaterialX | NodeDefが型、unit、color spaceを持ち、Target implementationを分離する | `ShaderValueSemanticV1`のtype／unit／space／colorとModule実装の分離 | Target名またはimplementation選択だけをTarget互換性証明にする方式 |
| Slang | Module、Capability、Reflectionを言語機能として提供する | 将来Frontend評価時の比較基準 | 初期HLSL／DXC baselineとの同時併設。言語追加は独立Qualificationを必須とする |

比較結果から、自由度の単位を「raw HLSL file」ではなく、S2のtyped Module、S3のDomain適合Stage／Shading Model、S4／S5のdeclarative Techniqueに固定する。AI理解は各EngineのEditor向けmetadataだけに依存せず、宣言semantic、compiler fact、runtime-use trace、fixture、Target artifactを`ShaderUnderstandingClosureV1`で閉じる。

## 14. 一次根拠

- [Microsoft HLSL Specification Working Draft](https://microsoft.github.io/hlsl-specs/specs/index.html): HLSL language semantics。Working Draftを浮動参照せず、Toolchainでcompiler behaviorをexact pinする
- [DirectX Shader Compiler API](https://github.com/microsoft/DirectXShaderCompiler/wiki/Using-dxc.exe-and-dxcompiler.dll): compile result、PDB、hash、reflection API
- [DXC HLSL options](https://github.com/microsoft/DirectXShaderCompiler/blob/main/include/dxc/Support/HLSLOptions.td): `-ast-dump`がhidden workaroundである根拠
- [Unreal Engine Render Dependency Graph](https://dev.epicgames.com/documentation/en-us/unreal-engine/render-dependency-graph-in-unreal-engine): Shader Parameter Struct、resource dependency validation、RDG Insights
- [Unreal Engine Global Shaders](https://dev.epicgames.com/documentation/en-us/unreal-engine/adding-global-shaders-to-unreal-engine): Material外Shader、Post Process／Compute等のProject拡張比較
- [Unity Shader Graph Custom Function Node](https://docs.unity3d.com/Packages/com.unity.shadergraph@17.2/manual/Custom-Function-Node.html): typed PortとProject HLSL function
- [Unity HLSL node reflection](https://github.com/Unity-Technologies/Graphics/blob/master/Packages/com.unity.shadergraph/Documentation~/Custom-nodes-hlsl-create-node-by-reflection.md): HLSL functionとparameter hintからNodeを生成する比較
- [Unity URP custom render pass](https://docs.unity3d.com/6000.3/Documentation/Manual/urp/renderer-features/custom-rendering-pass-workflow-in-urp.html): RenderGraph resource宣言とinjection point比較
- [Godot VisualShaderNodeCustom](https://docs.godotengine.org/en/4.7/classes/class_visualshadernodecustom.html): custom Nodeのtyped Port／code生成比較
- [Godot Shader](https://docs.godotengine.org/en/4.7/classes/class_shader.html): uniform queryと生成variant inspection比較
- [MaterialX Specification](https://github.com/AcademySoftwareFoundation/MaterialX/blob/main/documents/Specification/MaterialX.Specification.md): typed NodeDef、unit、color space、target implementation。Target hintを互換性証明にしない
- [Slang Modules](https://shader-slang.org/slang/user-guide/modules)、[Slang Reflection](https://shader-slang.org/slang/user-guide/reflection.html): 将来Frontend候補のModule／Capability／Reflection。初期HLSL／DXC baselineを置換しない
