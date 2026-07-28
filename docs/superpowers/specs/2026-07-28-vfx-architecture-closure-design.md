# Miraikanai Engine VFX Architecture Closure Design

- 状態: review
- 対象: VFX Authoring／Runtimeと直接連携するOwner文書の設計整理
- 非対象: Engine実装、Schema実装、Editor実装、テスト実装、実装Task、実装順序、担当、日程
- 判断日: 2026-07-28

## 1. 結論

MiraikanaiのVFXは、AI可読性、安全性、通常のゲーム制作で必要な表現自由度を両立するため、次の三層だけを採用する。

1. `core_graph`: Engine-owned closed Node Catalogだけで構成する型付きGraph。
2. `typed_subgraph`: Core GraphをVersion付きで再利用するSource Asset。Runtime inheritanceは作らず、Compilerがexact versionをinlineする。
3. `qualified_extension_operator`: CoreとSubgraphで表現不能な処理だけを、Target別Qualification、bounded resource、明示fallback付きで拡張する。

AIの自動ResolverとEditor／CLIの自動候補提示は`core_graph -> typed_subgraph -> qualified_extension_operator`の順で解決する。前段でMinimum Cueを満たす候補がある場合、後段を自動選択しない。Humanがexact Subgraphを明示選択することは許可し、exact Extensionを選ぶ場合はCore／Subgraphとの差分Cue、Target cost、fallback、ApprovalをChangeSetへ記録する。Extensionを便利な既定経路、任意code実行経路、Gameplay authorityへの迂回路にしない。

既存VFX計画はSemantic Intent、typed Graph、CPU／GPU artifact、deterministic cadence、LOD、Environment連携の方向は維持する。一方、現状のままではStyle Profile、Blend、Budget／Quality Profile、Artifact Evidence、Save、Render Graph Pass、Parameter適用時点、Product Work Packageが閉じていないため、Architecture計画書を同一変更で整合させる。

### 1.1 記法と版管理

- 本書で示すrecordはclosed objectであり、`optional`、`nullable`、tagged unionの非選択branch以外は全Field必須である。未知Field、重複Field、NaN／Inf、範囲外値を拒否する。
- `set<T>`と`refs[]`は各型のcanonical identity byte順にstrict sortし、duplicateを拒否する。順序自体に意味がある配列だけは記載順を保持する。
- `SHA-256`はlowercase hexadecimal 64文字である。`*_content_hash`は自己hash Fieldだけを除く全FieldをMCD canonical count／presence／length framingし、型ごとのASCII domain separatorを先頭へ置いて計算する。
- `exact Ref`はOwnerが定義するID、positive version／revision、content hashをすべて持ち、三Fieldのbyte equalityで解決する。ID-only、表示名、path、`latest`、同名別Owner、別Targetへの再解決を拒否する。
- 既存Ownerがすでに定義する`TargetProfileRefV1`、`ProjectRevisionRefV1`、`RuntimeEntryPackageRefV1`、`SimulationCadenceProfileRefV1`、`MirakanSignedRecordRefV1`は再定義しない。本書で新設するVFX固有Refは各節で全Fieldを示す。
- 現在のArchitecture文書は`review`、実装状態は`absent`、Operation／Capability activationは`not_activated`である。したがって本書は未実装V1設計の誤りを修正するが、実装済みSchema、保存データ、callable operationが存在するとは表明しない。

新設するcontent hash型のdomain separatorは次のexact値に固定する。

| 型 | ASCII domain separator |
|---|---|
| `VfxTypedSubgraphDocumentV1` | `MIRAKAN_VFX_TYPED_SUBGRAPH_DOCUMENT_V1` |
| `VfxSemanticRoleCatalogV1` | `MIRAKAN_VFX_SEMANTIC_ROLE_CATALOG_V1` |
| `VfxPatternCatalogV1` | `MIRAKAN_VFX_PATTERN_CATALOG_V1` |
| `VfxEffectIntentV1` | `MIRAKAN_VFX_EFFECT_INTENT_V1` |
| `VfxExtensionOperatorV1` | `MIRAKAN_VFX_EXTENSION_OPERATOR_V1` |
| `VfxStyleProfileV1` | `MIRAKAN_VFX_STYLE_PROFILE_V1` |
| `VfxBudgetProfileV1` | `MIRAKAN_VFX_BUDGET_PROFILE_V1` |
| `VfxQualityProfileV1` | `MIRAKAN_VFX_QUALITY_PROFILE_V1` |
| `VfxLodProfilePayloadV1` | `MIRAKAN_VFX_LOD_PROFILE_PAYLOAD_V1` |
| `VfxPerformanceBudgetAllocationV1` | `MIRAKAN_VFX_PERFORMANCE_BUDGET_ALLOCATION_V1` |
| `VfxSystemArtifactManifestV1` | `MIRAKAN_VFX_SYSTEM_ARTIFACT_MANIFEST_V1` |
| `VfxArtifactQualificationSubjectV1` | `MIRAKAN_VFX_ARTIFACT_QUALIFICATION_SUBJECT_V1` |
| `VfxArtifactQualificationReceiptPayloadV1` | `MIRAKAN_VFX_ARTIFACT_QUALIFICATION_RECEIPT_PAYLOAD_V1` |
| `VfxArtifactActivationBindingV1` | `MIRAKAN_VFX_ARTIFACT_ACTIVATION_BINDING_V1` |
| `VfxPassTemplateSetV1` | `MIRAKAN_VFX_PASS_TEMPLATE_SET_V1` |
| `PresentationReconstructionManifestV1` | `MIRAKAN_PRESENTATION_RECONSTRUCTION_MANIFEST_V1` |
| `VfxPresentationReconstructionProjectionV1` | `MIRAKAN_VFX_PRESENTATION_RECONSTRUCTION_PROJECTION_V1` |
| `RuntimeSessionPresentationCompanionV1` | `MIRAKAN_RUNTIME_SESSION_PRESENTATION_COMPANION_V1` |
| `PresentationReconstructionRequestV1` | `MIRAKAN_PRESENTATION_RECONSTRUCTION_REQUEST_V1` |

## 2. 調査結果

### 2.1 維持する強み

- `VfxSystemDocumentV1`をBeginner Stack、Graph、AI Intentの共通Sourceとする。
- GraphはStable ID、typed Port、closed Node Catalog、canonical serializationを使う。
- Semantic Role、Pattern、Minimum Cue、Fallbackを自然言語より先に解決する。
- VFXはPresentation専用であり、Particle、Visual Collision、Particle Light、GPU eventをGameplay、Physics、Damage、Navigation、Save digest、AI perceptionへ逆入力しない。
- CPU／GPU EmitterはSystem lifecycle、parameter、seed、transformを共有できるがParticle storageを共有しない。
- Simulation Advance、Weather generation、LOD、Render Snapshotへexact ref／hashで接続する。
- 24個のplanned VFX Operation IDはExecutable Contracts補助Catalogと一致する。
- 13個の既存VFX Capability IDはProduct Execution Registry候補と一致する。

### 2.2 解消する不明確点

1. Materialsの`VisualStyleProfileV1.vfx_style_profile_ref`に対応する`VfxStyleProfileV1`が存在しない。
2. VFXの`premultiplied_alpha | additive | multiply`とMaterialsの`opaque | mask | blend_premultiplied`が同じ型へ閉じていない。
3. `VfxBudgetProfileV1`と`VfxQualityProfileV1`にField、Ref、Target binding、hash規則がない。
4. VFX Artifactがplain Target／Quality IDを持ち、他Ownerのexact Ref方針と一致しない。
5. `VfxExecutionArtifactV1.verification_receipt_refs[]`がReceipt-free base規則と循環する。
6. `PersistentVfxDescriptorV1`のSession Save内の格納先とfailure policyがない。
7. VFX RuntimeがPass順を列挙する一方、Render GraphにVFX Pass Template Setがない。
8. `VfxParameterV1.update_rate=presentation_frame`がSimulation parameterとRender-only parameterを区別しない。
9. VFX standalone domain budgetとPerformance Ownerのaggregate budgetの合否関係に定義がない。
10. C1 VFX CapabilityがPhase 8の`wp.rendering.vfx-c2`だけに束縛され、Phase 3／6のFirst Playableと一致しない。
11. `wp.rendering.vfx-c2`のOwnerがRuntimeだけで、Authoring CapabilityとOperation activationのWork Packageがない。
12. Version付きSubgraph／Function Assetがなく、Pattern展開後の安全な再利用と一括更新ができない。
13. VFXから関連OwnerへのLinkはあるが、Materials、LOD、Scheduling、Performance、Persistence、Asset Lifecycleからの逆Linkが不足し、手動Dependency graphが不完全である。

### 2.3 AI可読性の評価

| 観点 | 設計確定後の評価 | 根拠／限界 |
|---|---|---|
| 意図理解 | 高い | Semantic Role、Pattern、Minimum Cue、Style Role、Target、dimensionをclosed ID／exact Refで分離する |
| 変更影響の追跡 | 高い | Source、Subgraph version、Profile、Artifact、Binding、Packageを一方向hash closureで追える |
| 安全な提案 | 高い | Core優先、Extension Approval、Gameplay逆入力禁止、Target fallbackを機械判定できる |
| 再利用の理解 | 高い | Subgraph dependency、call-site、origin map、resource boundを明示する |
| 現在の実利用性 | 未成立 | Schema、Validator、Catalog、Fixture、Runtime、Editor、AI Operationはすべて未実装／未Activationである |

したがって「AIが理解しやすい設計方針か」はyesだが、「現在AIが安全に編集・実行できるか」はnoである。Architecture文書の説明だけをtool availabilityまたは検証済み理解へ読み替えない。

## 3. 表現自由度の境界

### 3.1 Core Graph

Core Graphは通常の2D／3DゲームVFXを対象にする。Sprite、Billboard、Portable Facing Sprite、Flipbook、Trail、Mesh、Ribbon、Particle Light、Curve、Gradient、Counter RNG、Rate、Burst、Sub-emitter、Visual CollisionをCapabilityに従って扱う。

Core Nodeは次を満たす。

- stage、dimension、typed input／output、attribute read／write、scratch byte、cost formula、CPU kernel ID、GPU implementation IDを宣言する。
- GraphはDAGであり、implicit cast、unknown port、unbounded collection、pointer、native resource、file、network、OS APIを持たない。
- Particle attributeの複数writeは登録済みCombine Nodeだけを使う。
- CPU／GPU variantは同じSource meaningとMinimum Cueを維持する。
- Stateless Emitterは別Sourceまたは別Capabilityにせず、状態不要であることをCompilerが証明できるCore GraphのTarget別最適化とする。

### 3.2 Typed Subgraph

`VfxTypedSubgraphDocumentV1`をVFX Authoring Ownerへ追加する。

```text
VfxTypedSubgraphDocumentV1
  document_header
  subgraph_id: StableId
  subgraph_version: positive u32
  subgraph_content_hash: SHA-256
  stage: particle_spawn | particle_update | particle_event
  supported_dimensions: nonempty set<d2 | d3 | portable_2d_3d>
  input_ports[0..64]: VfxSubgraphInputPortV1
  output_ports[1..64]: VfxSubgraphOutputPortV1
  graph: VfxSubgraphGraphV1
  dependency_refs[0..32]: VfxTypedSubgraphRefV1
  capability_requirements[0..32]: CapabilityId
  resource_bound:
    maximum_inlined_nodes: positive u32 <= 1024
    maximum_inlined_edges: u32 <= 4096
    maximum_scratch_bytes_per_particle: u32
    maximum_custom_attributes: u32 <= 64

VfxSubgraphInputPortV1
  port_id: PortId
  value_type:
    bool | i32 | u32 | u64 | f32 | vec2f | vec3f | vec4f
    | color_linear_rgba | quaternionf | spatial_vector
    | curve1_ref | gradient_ref | texture2d_ref | mesh_ref | material_ref
  binding:
    {kind=required}
    | {kind=default_literal, value=exact value_type}
  target_bindings[1..4096]:
    {target_node_id: StableId, target_port_id: PortId}

VfxSubgraphOutputPortV1
  port_id: PortId
  value_type: same closed value type as VfxSubgraphInputPortV1
  source_node_id: StableId
  source_port_id: PortId

VfxSubgraphGraphV1
  nodes[1..1024]: VfxNodeV1
  edges[0..4096]: VfxEdgeV1

VfxTypedSubgraphRefV1
  subgraph_id: StableId
  subgraph_version: positive u32
  subgraph_content_hash: SHA-256
```

Subgraph内のNode／Edge、input target、output sourceは同じGraph内へ解決し、input targetは型一致するNode input、output sourceは型一致するNode outputでなければならない。同一input portは0件でなく1件以上のtargetを持てるが、同一Node target portへ通常EdgeとSubgraph inputの両方を接続しない。`dependency_refs[]`はGraph内`CallTypedSubgraph` Nodeが持つexact Ref集合とexact set equalityにする。

`CallTypedSubgraph`はAuthoring／Compilerだけに存在するNode Catalog entryであり、literalにexact `VfxTypedSubgraphRefV1`、動的portに解決先のinput／output portを持つ。CallerとCalleeのstageは同一、dimension集合はCallerのsubset、Capabilityはunionでなければならない。Dependency graphはacyclic、最大深度8とする。Compilerは全Refをexact解決してinlineし、inline後のEmitter全体がNode 1,024、Edge 4,096、custom attribute 64および全resource boundを満たすことを再検証する。Subgraph／call-site Stable IDのorigin mapをArtifact diagnosticへ保持し、`CallTypedSubgraph`、Subgraph object、virtual dispatch、inheritance、latest-version lookupをRuntime Artifactへ残さない。

Version更新は既存Refを上書きせず、新しい`subgraph_version`とhashを作る。利用Sourceの更新はsemantic diff、Cue diff、resource diffを持つ明示的Project ChangeSetで行う。AIは互換性を名前、minor label、類似Graphから推測しない。

`capability.vfx.typed_subgraph`をC1 Authoring Capabilityとして追加する。

### 3.3 Qualified Extension Operator

既存`VfxExtensionOperatorV1`は次のclosed schemaと境界へ固定する。

```text
VfxExtensionOperatorV1
  operator_id: StableId
  operator_version: positive u32
  stage: particle_spawn | particle_update | particle_event
  supported_dimensions: nonempty set<d2 | d3 | portable_2d_3d>
  input_ports[0..64]: VfxExtensionPortV1
  output_ports[1..64]: VfxExtensionPortV1
  readable_attribute_ids[0..64]: VfxAttributeId
  writable_attribute_ids[0..64]: VfxAttributeId
  maximum_scratch_bytes_per_particle: u32
  maximum_loop_iterations: u32
  maximum_neighbor_count: u32
  maximum_grid_cell_count: u32
  cost_model_id: StableId
  determinism_class: cpu_repeatable | visual_only
  implementations:
    cpu:
      optional {
        module_ref: NativeGameModuleRevisionRefV1,
        entry_point_id: StableId
      }
    gpu:
      optional {
        technique_ref: ProjectShaderTechniqueRevisionRefV1,
        port_id: vfx_simulation | vfx_render
      }
  operator_content_hash: SHA-256

VfxExtensionPortV1
  port_id: PortId
  value_type: same closed VFX value type as Core Graph

VfxExtensionOperatorRefV1
  operator_id: StableId
  operator_version: positive u32
  operator_content_hash: SHA-256
```

CPU／GPU implementationの少なくとも一方を必須にし、両方がある場合は同じPort、attribute meaning、Minimum Cueを持つ。0は「上限なし」ではなくloop／neighbor／grid機能不使用を意味し、該当処理を持つOperatorはpositive boundを必須にする。`VfxExtensionManifestV1.operator_ids[]`を`operator_refs[]: VfxExtensionOperatorRefV1[1..64]`、`semantic_role_ids[]`を`semantic_role_refs[]: VfxSemanticRoleRefV1[1..32]`、`supported_target_profiles[]`を`supported_target_profile_refs[]: TargetProfileRefV1[1..32]`、`fallback_pattern_refs[]`を`VfxPatternRefV1[0..16]`へ変更する。

- CPUはNative Game Moduleのbounded SoA spanだけを読み書きする。
- GPUはbounded typed IR、またはProject Shaderの`vfx_simulation | vfx_render` Portへ接続するQualification済みTechniqueだけを使う。
- 宣言可能なwriteはVFX Particle attribute、Emitter-local state、VFX render outputだけである。
- World、Physics、Damage、Quest、Save authority、Navigation、Audio正規event、AI perceptionへwriteできない。
- file、network、logging、native GPU API、mutable static state、runtime allocation、JIT、reflectionを禁止する。
- loopはcompile-time positive upper bound、neighbor／grid処理は要素数、cell数、iteration数、scratch byteの全上限をmanifestへ固定できる場合だけ許可する。
- Target、dimension、stage、attribute set、resource bound、determinism class、fallback、preserved Cue、Qualification Receiptを一つのActivation Projectionで閉じる。

Fluid、Volume、Grid Simulation、cross-backend Event BridgeはExtension Operatorの名前で暗黙許可しない。必要時は個別Product Capability、Owner、Artifact、Budget、fallback、Target Qualificationを先に追加する。

### 3.4 自由度の評価

| 表現領域 | 設計上の自由度 | 判断 |
|---|---|---|
| 2D／3D sprite、flipbook、trail、mesh、ribbon、light、sub-emitter | 高い | 一般的なゲームVFXの主領域をCore＋Subgraphで覆う |
| 色、curve、gradient、shape、motion、material blend、LOD | 高い | typed parameterとStyle／Material／LODの組合せで変更できる |
| 独自のbounded particle演算 | 中～高 | Qualified Extensionで可能だが、Target Qualificationとfallbackが必須 |
| 任意HLSL／C++、JIT、native GPU API | 低い／意図的に不許可 | 表現自由度より安全性、再現性、AI検証可能性を優先する |
| Fluid、Volume、Grid、GPU readbackを使うGameplay連携 | 未対応 | 別CapabilityとOwnerを定義するまでExtensionへ偽装しない |

この境界は、標準的なparticle表現では主要Engineに近い自由度を狙い、任意code注入ではNiagara／Visual Effect Graphの最も開放的な経路より狭い。狭さは欠落を隠した結果ではなく、AI authoring、cross-target fallback、Gameplay authority分離のための意図的なProduct判断である。ただしNode CatalogとQualificationが未実装の現在、実際の表現到達度を「同等」とは主張しない。

## 4. StyleとMaterial Blendの閉包

### 4.1 Semantic identity

AIとCompilerがRole、Pattern、Intentを名前から再解決しないよう、VFX Authoring Ownerの三型を次へ固定する。

```text
VfxSemanticRoleCatalogV1
  catalog_id: StableId
  catalog_version: positive u32
  roles[1..256]:
    role_id: StableId
    allowed_spatial_scopes: nonempty set<d2 | d3 | hybrid | portable_2d_3d>
    allowed_temporal_roles: nonempty set<one_shot | loop | continuous>
    allowed_gameplay_relations: nonempty set<
      cosmetic | authoritative_event_presentation | visual_collision_only>
    required_minimum_cue_contract: MinimumCueContractV1
    maximum_semantic_priority: LodSemanticPriorityV1
    presentation_event_type_ids[0..16]: registered Presentation Event Type ID
  catalog_content_hash: SHA-256

VfxSemanticRoleRefV1
  catalog_id: StableId
  catalog_version: positive u32
  catalog_content_hash: SHA-256
  role_id: StableId

VfxEffectIntentV1
  intent_id: StableId
  intent_version: positive u32
  semantic_role_ref: VfxSemanticRoleRefV1
  spatial_scope: d2 | d3 | hybrid | portable_2d_3d
  temporal_role: one_shot | loop | continuous
  gameplay_relation:
    cosmetic | authoritative_event_presentation | visual_collision_only
  vfx_style_profile_ref: VfxStyleProfileRefV1
  style_role_ids[0..16]: StableId
  target_profile_refs[1..16]: TargetProfileRefV1
  scale_envelope:
    maximum_concurrent_instances: positive u32
    maximum_visible_instances: positive u32
    maximum_spawn_per_second: positive u32
    maximum_burst_per_advance: positive u32
  semantic_priority: LodSemanticPriorityV1
  minimum_cue_contract: MinimumCueContractV1
  fallback_policy: semantic_equivalent_only | approval_required | no_fallback
  extension_policy: core_only | qualified_extension_allowed
  intent_content_hash: SHA-256

VfxEffectIntentRefV1
  intent_id: StableId
  intent_version: positive u32
  intent_content_hash: SHA-256
```

Role Catalogからcandidate Patternへのback-referenceは持たない。`VfxPatternCatalogV1`の各Patternがexact `VfxSemanticRoleRefV1`を持ち、Roleからの候補検索はPattern Catalogのreverse indexとして導出する。これにより`Role Catalog -> Pattern Catalog`の一方向hash依存とし、相互hash cycleを作らない。`VfxSystemDocumentV1.effect_intent_ref`は`VfxEffectIntentRefV1`へ変更し、SystemとIntentの`vfx_style_profile_ref`はbyte equality、選択Compile TargetはIntentの`target_profile_refs[]`のmember、Intentの各意味軸はRoleのallowlist内でなければならない。

### 4.2 `VfxStyleProfileV1`

VFX Authoring Ownerへ次を追加する。

```text
VfxStyleProfileV1
  profile_id: StableId
  profile_version: positive u32
  profile_content_hash: SHA-256
  style_roles[1..32]: VfxStyleRoleV1
  fallback_policy: semantic_equivalent_only | approval_required | no_fallback

VfxStyleProfileRefV1
  profile_id: StableId
  profile_version: positive u32
  profile_content_hash: SHA-256

VfxStyleRoleV1
  style_role_id: StableId
  compatible_semantic_role_refs[1..32]: VfxSemanticRoleRefV1
  allowed_spatial_domains: nonempty set<d2 | d3 | portable_2d_3d>
  allowed_render_outputs: nonempty set<
    sprite_2d | billboard_3d | portable_facing_sprite | flipbook
    | trail | mesh | ribbon | particle_light>
  allowed_alpha_modes: nonempty set<
    opaque | mask | blend_premultiplied | blend_additive | blend_multiply>
  allowed_color_classes: nonempty set<
    authored_linear | palette_constrained | monochrome_tint | gradient_mapped>
  allowed_motion_classes: nonempty set<
    ballistic | eased | orbital | turbulent | stepped | authored_curve>
  allowed_shape_classes: nonempty set<
    geometric | organic | pixel_aligned | mesh_authored | trail_authored>
  allowed_sampling: nonempty set<point | linear | anisotropic>
  allowed_pattern_refs[1..64]: VfxPatternRefV1
  minimum_cue_contract: MinimumCueContractV1

VfxPatternRefV1
  catalog_id: StableId
  catalog_version: positive u32
  catalog_content_hash: SHA-256
  pattern_id: StableId
```

`VfxPatternCatalogV1`は`catalog_id`、positive `catalog_version`、`catalog_content_hash`を持つReceipt-free catalogへ固定する。各Patternはexact `VfxSemanticRoleRefV1`を持つ。`VfxPatternRefV1`はその一件のCatalogと、そのCatalog内に一意な`pattern_id`へ解決する。

`VisualStyleProfileV1.vfx_style_profile_ref`、`VfxSystemDocumentV1.vfx_style_profile_ref`、解決済みCompile入力はbyte equalityにする。既存`visual_style_roles`と`VfxEffectIntentV1.style_role_ids`は`StableId[0..16]`のまま維持し、同じ`VfxStyleProfileV1.style_roles[].style_role_id`のsubsetでなければならない。Role IDはProfile内一意であり、別Profileの同名Roleへ再解決しない。

Target別particle数、execution target、memory、LODはStyleへ含めず、Budget／Quality／LOD Ownerへ残す。

### 4.3 Blend

Materials Ownerの`AlphaMode`を次のclosed setへ更新し、VFXは同じIDを参照する。

```text
opaque | mask | blend_premultiplied | blend_additive | blend_multiply
```

- `opaque`: blendingなし、depth write既定on。
- `mask`: coverage判定、採択pixelだけdepth write。
- `blend_premultiplied`: premultiplied source-over、depth write既定off。
- `blend_additive`: scene-linear additive、depth write off。
- `blend_multiply`: scene-linear multiply、depth write off。

`blend_additive | blend_multiply`はMaterial Domain `vfx`だけに許可し、他Domainでは`MIRAKAN-MATERIAL-DOMAIN_MISMATCH`で拒否する。scene-linear HDR colorに対するblend式を次に固定する。`Cs`はstraight source color、`As`はsource alpha、`Cd, Ad`はdestination、`Csp=Cs*As`であり、中間結果を`[0,1]`へclampしない。

```text
blend_premultiplied:
  Cout = Csp + Cd * (1 - As)
  Aout = As + Ad * (1 - As)
blend_additive:
  Cout = Csp + Cd
  Aout = max(As, Ad)
blend_multiply:
  Cout = Cd * ((1 - As) + Cs * As)
  Aout = Ad
```

VFX固有の別`Blend` enumを削除し、Render Outputはexact Material Definition refとそのAlphaModeを持つ。Source colorはstraight alpha、`blend_premultiplied`のartifact inputだけCook時にpremultiplyする。`blend_additive | blend_multiply`をpremultiplied source-overへ変換しない。Material InstanceとRuntime OverrideはAlphaModeを変更できない。glTF `BLEND`の既存mappingは`blend_premultiplied`のまま維持し、additive／multiplyへ推測変換しない。

## 5. Budget、Quality、LOD、Target

### 5.1 Exact Profile

VFX Authoring Ownerへ次を追加する。

```text
VfxBudgetProfileV1
  profile_id: StableId
  profile_version: positive u32
  profile_content_hash: SHA-256
  target_bindings[1..16]: VfxTargetBudgetBindingV1

VfxBudgetProfileRefV1
  profile_id: StableId
  profile_version: positive u32
  profile_content_hash: SHA-256

VfxTargetBudgetBindingV1
  target_profile_ref: TargetProfileRefV1
  performance_allocation_ref: VfxPerformanceBudgetAllocationRefV1
  budget_value_state: provisional | approved_ceiling
  backend_budgets[1..2]: VfxBackendBudgetV1

VfxBackendBudgetV1
  execution_target: cpu | gpu
  maximum_active_emitters: positive u32
  maximum_alive_project: positive u32
  maximum_alive_emitter: positive u32
  maximum_spawn_per_second_project: positive u32
  maximum_spawn_per_second_emitter: positive u32
  maximum_burst_per_advance_project: positive u32
  maximum_burst_per_advance_emitter: positive u32
  maximum_events_per_advance_project: u32
  maximum_events_per_advance_emitter: u32
  maximum_trail_points_project: u32
  maximum_trail_points_per_trail: u32
  maximum_particle_lights_project: u32
  maximum_particle_lights_emitter: u32
  persistent_memory_bytes: u64
  transient_memory_bytes: u64
  standalone_p95_ns: u64

VfxQualityProfileV1
  profile_id: StableId
  profile_version: positive u32
  profile_content_hash: SHA-256
  target_bindings[1..16]: VfxTargetQualityBindingV1

VfxQualityProfileRefV1
  profile_id: StableId
  profile_version: positive u32
  profile_content_hash: SHA-256

VfxTargetQualityBindingV1
  target_profile_ref: TargetProfileRefV1
  budget_profile_ref: VfxBudgetProfileRefV1
  lod_profile_ref: VfxLodProfileRefV1
  allowed_execution_targets: nonempty set<cpu | gpu>
  allowed_render_outputs: nonempty set<
    sprite_2d | billboard_3d | portable_facing_sprite | flipbook
    | trail | mesh | ribbon | particle_light>
  extension_policy: core_only | qualified_extension_allowed
  fallback_policy: semantic_equivalent_only | approval_required | no_fallback

VfxLodProfileRefV1
  profile_id: StableId
  profile_version: positive u32
  profile_content_hash: SHA-256

VfxPerformanceBudgetAllocationRefV1
  allocation_id: StableId
  allocation_revision: positive u64
  allocation_content_hash: SHA-256
```

Performance Ownerは次のread-only projectionを所有し、VFX BudgetはRefだけを持つ。

```text
VfxPerformanceBudgetAllocationV1
  allocation_id: StableId
  allocation_revision: positive u64
  target_profile_ref: TargetProfileRefV1
  cpu_critical_path_group_id:
    performance.cpu.postphysics_presentation
  cpu_group_aggregate_p95_cap_ns: u64
  gpu_pass_group_id:
    performance.gpu.transparent_vfx
  gpu_group_aggregate_p95_cap_ns: u64
  measurement_state: provisional | measured
  allocation_content_hash: SHA-256
```

`VfxLodProfileV1 = LodDomainProfileV1<VfxLodProfilePayloadV1>`とする。LOD Ownerは共通Envelopeの`lod_class`、`subject_scope_ref`、`profile_id`、positive `profile_version`、Representation Set、tier binding、`profile_content_hash`を所有し、VFX Ownerはclosed tier payloadと`payload_content_hash`を所有する。narrow `VfxLodProfileRefV1={profile_id, profile_version, profile_content_hash}`はclass=`vfx_presentation`のfull `LodDomainProfileRefV1`の同三Fieldとbyte equalityにし、残るFieldをIDや表示名から補完しない。`VfxSystemDocumentV1`は`VfxBudgetProfileRefV1`、`VfxQualityProfileRefV1`、`VfxLodProfileRefV1`を持つ。Compilerは選択Target Profileに一致するbindingをexact一件だけ解決し、0件、複数、stale、Target mismatchをCook前に拒否する。Qualityの`allowed_execution_targets`は同Targetの`backend_budgets[].execution_target`のsubsetでなければならない。

`VfxSystemArtifactManifestV1`と`VfxExecutionArtifactV1`の`target_profile_id`／`quality_profile_id`を`target_profile_ref`／`quality_profile_ref`へ変更し、`budget_profile_ref`と`lod_profile_ref`を追加する。

### 5.2 StandaloneとAggregate Budget

VFX Profileの`VfxBackendBudgetV1.standalone_p95_ns`はVFXだけを測るArtifact eligibility上限であり、共通Budgetの予約または保証ではない。Performance OwnerのCPU group、GPU pass group、memory working-set capは統合runのaggregate qualification上限であり、runtime hard deadlineまたはVFX予約値へ読み替えない。

`budget_value_state=approved_ceiling`かつ解決したPerformance Allocationの`measurement_state=measured`である場合だけ、Qualificationは次の両方を評価する。

1. VFX単体runがVFX Target bindingのstandalone上限とbyte上限を満たす。
2. Audio、Camera、UI、Environment、Transparent等を含む統合runがPerformance Ownerのaggregate上限を満たす。

単体合格で統合合格を代用せず、aggregateの未使用値をVFXへ予約済みとして扱わない。いずれかが`provisional`ならProduction Qualificationを`performance_envelope_unqualified`でblockedにし、予測値だけでpass／failを発行しない。既存の未計測数値は`provisional`のまま維持し、Measurement Receiptなしに`approved_ceiling`または`measured`へ昇格しない。

## 6. Artifact、Receipt、Activation

`VfxExecutionArtifactV1.verification_receipt_refs[]`を削除する。VFX Source、Profile、Compiler、ArtifactはReceipt-free baseであり、ReceiptまたはActivation Bindingを自身のcontent hashへ含めない。`VfxSystemArtifactManifestV1`へ`manifest_id`、positive `manifest_version`、`manifest_content_hash`を追加し、全Emitter Artifact ref／hashをそのpreimageへ含める。

```text
VfxSystemArtifactManifestRefV1
  manifest_id: StableId
  manifest_version: positive u32
  manifest_content_hash: SHA-256

VfxArtifactQualificationSubjectV1
  qualification_id: StableId
  qualification_version: positive u32
  manifest_ref: VfxSystemArtifactManifestRefV1
  target_profile_ref: TargetProfileRefV1
  quality_profile_ref: VfxQualityProfileRefV1
  budget_profile_ref: VfxBudgetProfileRefV1
  cadence_profile_ref: SimulationCadenceProfileRefV1
  fixture_refs[1..64]:
    exact {fixture_id, fixture_version, fixture_content_hash}
  toolchain_lock_hash: SHA-256
  input_closure_hash: SHA-256
  subject_content_hash: SHA-256

VfxArtifactQualificationSubjectRefV1
  qualification_id: StableId
  qualification_version: positive u32
  subject_content_hash: SHA-256

VfxArtifactQualificationReceiptPayloadV1
  subject_ref: VfxArtifactQualificationSubjectRefV1
  result: pass | fail
  measured_metric_set_hash: SHA-256
  diagnostic_ids[0..64]: StableId
  payload_content_hash: SHA-256

VfxArtifactActivationBindingV1
  binding_id: StableId
  binding_version: positive u32
  manifest_ref: VfxSystemArtifactManifestRefV1
  qualification_receipt_refs[1..64]:
    exact MirakanSignedRecordRefV1(purpose=vfx_artifact_qualification)
  binding_content_hash: SHA-256

VfxArtifactActivationBindingRefV1
  binding_id: StableId
  binding_version: positive u32
  binding_content_hash: SHA-256
```

`VfxArtifactQualificationReceiptPayloadV1`は`MirakanSignedRecordV1(purpose=vfx_artifact_qualification)`のclosed payloadである。Bindingの全Receiptは同じManifest、Target、Quality、Budget、Cadence、Toolchain closureに対するpass payloadへ解決し、必要Fixture集合とexact set equalityでなければならない。fail、stale、revoked、別Manifest、別Target、部分Fixture、duplicate Receiptを拒否する。

生成順は次だけを許す。

```text
VFX Source／Profile
  -> receipt-free System Artifact Manifest
  -> Qualification Subject
  -> signed Receipt
  -> root外Activation Binding
  -> Product／Runtime Package selection
```

Asset Lifecycleの`DerivedArtifactManifestV1`はVFX System Assetに対して次のexact二Roleだけを登録する。

- `artifact.role.vfx.system_artifact_set.v1`: receipt-free `VfxSystemArtifactManifestV1`と、そのManifestが列挙する全`VfxExecutionArtifactV1` bytes。
- `artifact.role.vfx.system_activation_binding.v1`: 完成`VfxArtifactActivationBindingV1` bytes。generic `dependency_keys[]`は同じAsset revision／Targetの`system_artifact_set.v1` artifact keyをexact一件含む。

両方を`ArtifactSubjectRefV1.kind=asset`で包み、generic artifact key、dependency closure、Catalog、promotionをAsset Lifecycleが所有する。Artifact Set側からActivation Binding keyへ依存せず、一方向`activation_binding -> system_artifact_set`だけを許す。Runtime Package assemblyはRuntime EntryのVFX Source reachabilityから両Roleを列挙し、Catalog dependencyを維持する。LoaderはActivation Bindingがfreshかつpassで、BindingのManifest refがreceipt-free RoleのManifestへbyte equalityの場合だけArtifact SetをReadyにする。VFX Ownerは二payload schemaと内部closureを所有し、generic Manifest、Catalog、Package Directoryを複写しない。

## 7. Runtime Parameterの適用時点

`VfxParameterV1.update_rate`を次へ変更する。

```text
instance_start
| simulation_advance_boundary
| presentation_frame_render_only
```

- `instance_start`: Spawn commandのparameter blockだけから設定し、instance開始後の変更を拒否する。
- `simulation_advance_boundary`: canonical command merge後、対象Simulation Advanceの`T90_PresentationBuild`開始時に一度適用し、CPU updateと同advanceのGPU Advance Recordが同じparameter revisionを使う。
- `presentation_frame_render_only`: render output propertyだけへbindでき、spawn／update／event graph、rate、lifetime、collision、event、seed、Particle attributeへ接続できない。Render Snapshot generationごとに適用し、Simulation／Save digest／authoritative Replayへ含めない。

`SetVfxParameterV1`を次に固定する。

```text
SetVfxParameterV1
  command_id: StableId
  system_handle: VfxSystemHandle
  parameter_id: StableId
  typed_value: exact parameter value_type
  expected_parameter_revision: u64
  apply_target:
    {kind=simulation_advance_boundary,
     target_advance_sequence=positive u64}
    | {kind=presentation_frame_render_only,
       target_render_snapshot_generation=positive u64}
```

`instance_start` parameterは`SpawnVfxSystemV1.parameter_block`だけで設定し、`SetVfxParameterV1`を受理しない。apply kindはSourceの`update_rate`と一致させる。current revisionとexpected revisionが一致する成功適用だけrevisionをexact `+1`し、同じ`command_id`と同じcanonical commandは同じ結果を返す。同じIDの別bytes、stale、duplicate target、phase mismatch、過去sequenceをcommand全体で拒否し、次frameまたは次advanceへ暗黙繰越ししない。

## 8. Render Graph連携

Render Graph Ownerへ`VfxPassTemplateSetV1`を追加する。

```text
VfxPassTemplateSetV1
  template_set_id: StableId
  template_set_version: positive u32
  template_set_content_hash: SHA-256
  pass_templates[7]: VfxPassTemplateV1

VfxPassTemplateSetRefV1
  template_set_id: StableId
  template_set_version: positive u32
  template_set_content_hash: SHA-256

VfxPassTemplateV1
  pass_template_id: StableId
  pass_type: compute | graphics
  queue_eligibility: nonempty set<graphics | async_compute>
  resource_accesses[0..32]: VfxPassResourceAccessV1
  dependencies[0..6]: VfxPassDependencyV1

VfxPassResourceAccessV1
  resource_role_id: StableId
  access: read | write | read_write
  condition:
    always | gpu_emitter_present | cpu_batch_present
    | sort_enabled | depth_attachment_selected

VfxPassDependencyV1
  predecessor_pass_template_id: StableId
  condition:
    always | simulation_advance_present | gpu_emitter_present | sort_enabled
```

Set内のPassは次のexact 7件である。`R`／`W`はlogical resource roleであり、実buffer handleやBackend resource IDではない。

| Pass Template ID | Type／Queue | Read | Write | Predecessor |
|---|---|---|---|---|
| `pass.vfx.gpu.reset_counters` | compute／`{graphics, async_compute}` | `[]` | `resource.vfx.counter; resource.vfx.event; resource.vfx.spawn_admission; resource.vfx.indirect` | `[]` |
| `pass.vfx.gpu.update_compact` | compute／`{graphics, async_compute}` | `resource.vfx.particle_ready; resource.vfx.counter` | `resource.vfx.particle_next; resource.vfx.alive_index; resource.vfx.counter; resource.vfx.event` | `reset_counters` |
| `pass.vfx.gpu.event_resolve` | compute／`{graphics, async_compute}` | `resource.vfx.particle_next; resource.vfx.event` | `resource.vfx.spawn_admission` | `update_compact` |
| `pass.vfx.gpu.admit_spawn` | compute／`{graphics, async_compute}` | `resource.vfx.spawn_admission; resource.vfx.counter` | `resource.vfx.particle_next; resource.vfx.alive_index; resource.vfx.counter` | `event_resolve` |
| `pass.vfx.gpu.sort` | compute／`{graphics, async_compute}` | `resource.vfx.particle_render_source; resource.vfx.alive_index; resource.vfx.counter` | `resource.vfx.sort_key; resource.vfx.sorted_index` | simulation時だけ`admit_spawn` |
| `pass.vfx.gpu.build_indirect` | compute／`{graphics, async_compute}` | `resource.vfx.particle_render_source; resource.vfx.alive_index; ?sort_enabled:resource.vfx.sorted_index; resource.vfx.counter` | `resource.vfx.indirect` | simulation時`admit_spawn`、sort時`sort` |
| `pass.vfx.draw` | graphics／`{graphics}` | `?gpu:resource.vfx.particle_render_source; ?gpu:resource.vfx.alive_index; ?sort:resource.vfx.sorted_index; ?gpu:resource.vfx.indirect; ?cpu:resource.vfx.cpu_draw_batch; resource.vfx.material` | `resource.render.selected_color_attachment; ?depth:resource.render.selected_depth_attachment` | GPU時だけ`build_indirect` |

表内短縮Predecessorは同じ`pass.vfx.*` prefixのexact IDを意味する。`?gpu`、`?cpu`、`?sort`、`?depth`はSchemaの同名conditionへ写像し、非選択時に空bufferまたは偽attachmentを作らない。`resource.vfx.particle_render_source`はReady／Next swap後のread-only logical roleで、simulationがあるinstanceでは新Ready、ないinstanceでは前回Readyへ一意にbindする。TemplateのRefは`{template_set_id, template_set_version, template_set_content_hash}`である。Setのhashは表の全Field、conditional predicate、resource role集合を含み、Backend queue統合またはGraph pass merge後もlogical dependencyとaccess集合を失わない。

VFX RuntimeはParticle意味、advance record、Ready／Next semantics、必要Pass順を所有する。Render GraphはPass Template ID、resource access、queue eligibility、barrier、alias、surface compositionを所有する。

`VfxExecutionArtifactV1.render_pass_template_ids[]`を`vfx_pass_template_set_ref: VfxPassTemplateSetRefV1`へ変更する。Graph CompilerはSetを展開し、resource hazard検証後だけPassをmergeできる。merge後も次の意味順を変えない。

```text
reset -> update_compact -> event_resolve -> admit_spawn
-> Ready/Next swap -> optional sort -> build_indirect -> draw
```

`VfxBatchSnapshotV1`がないframeはVFX passを登録しない。GPU Emitter recordがあるがadvance recordが0件の場合は`reset_counters`から`admit_spawn`までを登録せず、既存Ready stateに対して必要な`sort`、`build_indirect`、`draw`だけを登録する。CPU draw batchだけの場合も`pass.vfx.draw`だけを登録できるが、そのTemplate instanceはGPU simulation resource roleを参照せずCPU batch resourceへ束縛する。

## 9. PersistenceとReplay

既存`RuntimeSessionSaveBundleV1`、`RuntimeSessionLoadRequestV1`、authoritative digestへFieldを追加しない。互換性とfailure分離のため、Persistence Ownerへ非authoritativeなcompanion closureを追加する。

```text
PresentationReconstructionManifestV1
  manifest_id: StableId
  manifest_version: positive u32
  source_authoritative_bundle_ref: RuntimeSessionSaveBundleRefV1
  source_project_revision_ref: ProjectRevisionRefV1
  runtime_entry_package_ref: RuntimeEntryPackageRefV1
  domain_projection_refs[0..4096]: PresentationReconstructionProjectionRefV1
  manifest_content_hash: SHA-256

PresentationReconstructionProjectionRefV1
  owner_id: StableId
  projection_type_id: StableId
  projection_id: StableId
  projection_version: positive u32
  projection_content_hash: SHA-256

VfxPresentationReconstructionProjectionV1
  projection_id: StableId
  projection_version: positive u32
  source_authoritative_bundle_ref: RuntimeSessionSaveBundleRefV1
  source_project_revision_ref: ProjectRevisionRefV1
  runtime_entry_package_ref: RuntimeEntryPackageRefV1
  persistent_vfx_descriptors[1..4096]: PersistentVfxDescriptorV1
  projection_content_hash: SHA-256

RuntimeSessionPresentationCompanionV1
  companion_id: StableId
  companion_version: positive u32
  authoritative_bundle_ref: RuntimeSessionSaveBundleRefV1
  reconstruction_manifest_ref:
    exact {manifest_id, manifest_version, manifest_content_hash}
  companion_content_hash: SHA-256

RuntimeSessionPresentationCompanionRefV1
  companion_id: StableId
  companion_version: positive u32
  companion_content_hash: SHA-256

PresentationReconstructionRequestV1
  request_id: StableId
  idempotency_key: StableId
  play_session_id: StableId
  active_branch_generation: positive u64
  authoritative_bundle_ref: RuntimeSessionSaveBundleRefV1
  presentation_companion_ref: RuntimeSessionPresentationCompanionRefV1
  request_content_hash: SHA-256

SaveCatalogV3
  base_schema: SaveCatalogV2の全Field、型、上限、不変条件をbyte-identicalに継承
  catalog_schema_version: 3
  slots[].presentation_companion_ref:
    nullable<RuntimeSessionPresentationCompanionRefV1>
```

この記法は継承可能なRuntime objectを意味せず、UI OwnerがV2 Schemaを複写せずV3 deltaを審査するための定義である。V3 checksumはV2と同じcanonical規則へ新FieldのpresenceとRef bytesを加え、V2 checksumと相互代用しない。

将来の`SaveCatalogV3` slotだけが既存`runtime_session_save_bundle_ref`に加えてnullable `presentation_companion_ref`を持つ。current `SaveCatalogV2`、`RuntimeSessionSaveBundleV1`、Load Request V1は不変に保ち、V3が別Qualificationとmigrationを通るまでCompanionをactive Save機能と表示しない。V2→V3 migrationでは既存slotのCompanionを推測せずnullにし、Bundle ref、display metadata、content package set、statusを変更しない。Companion、Manifest、domain projectionはauthoritative state digest、Gameplay Replay digest、network state、`RuntimeSessionSaveBundleV1.bundle_content_hash`へ含めない。

`PersistentVfxDescriptorV1`は次へ固定し、VFX Runtimeが`VfxPresentationReconstructionProjectionV1`のpayloadとして所有する。

```text
PersistentVfxDescriptorV1
  artifact_subject_ref: ArtifactSubjectRefV1(kind=asset)
  instance_stable_id: StableId
  spatial_domain: d2 | d3
  transform_binding:
    {kind=entity_anchor,
     persistent_entity_identity_ref=PersistentEntityIdentityRefV1}
    | {kind=fixed_d2,
       position_m=finite vec2f,
       rotation_rad=finite f32,
       scale=finite positive vec2f}
    | {kind=fixed_d3,
       position_m=finite vec3f,
       rotation=normalized quaternionf,
       scale=finite positive vec3f}
  parameter_block: closed typed parameter ID／value map
  parameter_revision: u64
  system_seed: u64
  artifact_activation_binding_ref: VfxArtifactActivationBindingRefV1
  style_profile_ref: VfxStyleProfileRefV1
  budget_profile_ref: VfxBudgetProfileRefV1
  quality_profile_ref: VfxQualityProfileRefV1
  lod_profile_ref: VfxLodProfileRefV1
  cadence_profile_ref: SimulationCadenceProfileRefV1
  start_advance_sequence: positive u64
  saved_advance_sequence: positive u64
  paused: bool
```

`saved_advance_sequence >= start_advance_sequence`を必須にする。`entity_anchor`はauthoritative World再構築後に同じPersistent Entity Identityへ一件だけ解決し、missing／duplicate時はVFX projectionだけを拒否する。保存対象は明示的にpersistentと宣言したambient／loopだけで、one-shot、Particle、GPU buffer、sort key、Trail point、visual collision result、render historyを保存しない。

Load順は次とする。

1. Persistence Ownerがauthoritative Saveを検証しWorldを再構築する。
2. authoritative load commit後、同じCatalog generation／slotにV3 Companionが存在する場合だけ別`PresentationReconstructionRequestV1`を生成する。このRequestはSave Bundle ref、Companion ref、new active branch generation、idempotency keyを持つ。
3. Companion、Manifest、ProjectionのSave Bundle／Project revision／Runtime Entry Packageがbyte equalityで、VFX Artifact Subject／Activation Binding／Profile／Cadence refがcurrent Packageへ一致する場合だけVFX projectionを受理する。RuntimeはSource GraphまたはAuthoring Documentを読むことで照合しない。
4. `elapsed_advances = saved_advance_sequence - start_advance_sequence`をchecked計算する。`elapsed_advances <= 120`は同じCadence／seed／parameterでexact回数prewarmする。`elapsed_advances > 120`は同じSource／Artifact／Cadence／seed／loop phaseへ一致するBake Cacheがある場合だけCacheを使い、ない場合はloop phase 0から再開して`vfx_presentation_phase_restarted`を記録し、Replay completenessを`visual_incomplete`にする。
5. Companion missing、stale、corrupt、未移行、VFX再構築失敗はtyped Diagnosticを発行して当該Presentationだけを停止し、完了済みauthoritative load、World、Damage、Quest、Save Catalog、last-valid Saveをrollbackまたは失敗へ変更しない。

Save時はauthoritative Bundleを先にreceipt-freeで完成させ、そのRefを入力にCompanionを生成し、両payloadとV3 Catalogを同じSettings／Catalog staging transactionでpublishする。persistent VFXが0件ならCompanion refはcanonical nullである。persistent VFXが1件以上あるがCompanion生成だけ失敗した場合は、Bundleを持つslotとnull Companionをpublishし、Save結果へ`presentation_reconstruction_incomplete`を明示する。失敗Companion、部分Manifest、旧Companionを新Bundleへ流用しない。

Replayは受理済みauthoritative Presentation Event、seed、parameter command、Artifact／Profile refを記録できるが、GPU Particle stateを記録しない。Replay completenessはvisual complete／incompleteを区別し、incomplete visual replayをauthoritative mismatchと表示しない。

## 10. SchedulingとOwner境界

- Runtime Schedulingは`T90_PresentationBuild`、command merge、Simulation Advance、snapshot publishを所有する。
- VFX RuntimeはT90内のVFX command適用、CPU simulation、GPU Advance Record構築を所有する。
- Render Graphはpublish済み`VfxBatchSnapshotV1`だけを読み、World leaseまたはlive VFX stateを持たない。
- LODは`VfxLodProfileV1`の共通Envelopeとtier selectionを所有し、VFX Authoringはclosed payloadを所有し、VFX Source／Runtimeは選択済みtierを解釈する。
- MaterialsはMaterial Domain、Shading Model、AlphaMode、render-state intentを所有する。
- VFX AuthoringはStyle Profile、Semantic Role、Pattern、Graph、Subgraph、Extension、Compiler入力を所有する。
- Asset Lifecycleはgeneric Derived Artifact envelope、Catalog、promotion、Package assemblyを所有する。
- Runtime PackageはCatalog／Directory／Package rootとVFX二Roleのreachabilityを所有し、VFX payload FieldまたはQualification意味を再定義しない。
- Persistenceはauthoritative Session Save root、Save Catalog、Presentation Companion／Manifestを所有し、VFXはPresentation Domain projection payloadだけを所有する。
- Performanceはaggregate budget、measurement method、integrated qualificationを所有し、VFXはstandalone domain estimate／ceilingを所有する。

各Owner Headerの関連文書に双方向Linkを追加する。Link追加は規範依存を増やさず、既存DAGを循環させない。VFXを成立条件にしないOwnerでは関連文書としてだけ追加する。

## 11. Product CapabilityとWork Package

既存`wp.rendering.vfx-c2`一件へC1／C2、Authoring／Runtimeを混在させない。Product Execution Registry Proposalを次のexact 5 Work Packageへ分割する。`required capabilities`は全targetでrequiredなedgeだけ、Targetの異なるorderingは`requires WP`だけへ置く。

| Work Package／Phase／Owner | Targets | Required capabilities | Requires WP | Provides |
|---|---|---|---|---|
| `wp.rendering.vfx-authoring-c1`／`phase.manual-2d`／`mirakan.arch.rendering-vfx-authoring` | Windows、Android、Apple | `capability.rendering.render-graph-core` | `wp.authoring.asset-save-headless; wp.rendering.render-graph-core` | `semantic_intent; pattern_catalog; typed_subgraph` |
| `wp.rendering.vfx-runtime-2d-c1`／`phase.manual-2d`／`mirakan.arch.rendering-vfx-runtime` | Windows、Android、Apple | `capability.runtime.scheduling; capability.rendering.render-graph-core; capability.world.2d; capability.camera.2d` | `wp.rendering.vfx-authoring-c1; wp.runtime.scheduling-core; wp.rendering.render-graph-core; wp.rendering.world-2d; wp.rendering.camera-2d` | `system; particle_cpu; sprite_2d; trail` |
| `wp.rendering.vfx-runtime-3d-c1`／`phase.manual-3d-mvp-b`／`mirakan.arch.rendering-vfx-runtime` | Windows、Android、Apple | `capability.vfx.system; capability.vfx.particle_cpu; capability.rendering.render-graph-core; capability.world.3d; capability.camera.3d` | `wp.rendering.vfx-runtime-2d-c1; wp.rendering.world-3d; wp.rendering.camera-3d` | `billboard_3d` |
| `wp.rendering.vfx-authoring-c2`／`phase.production-capability`／`mirakan.arch.rendering-vfx-authoring` | Windows、Android、Apple | `capability.rendering.render-graph-core` | `wp.rendering.vfx-authoring-c1; wp.product.project-source-activation; wp.platform.windows-package; wp.platform.android-package; wp.platform.apple-package` | `bake_cache; extension_operator` |
| `wp.rendering.vfx-runtime-c2`／`phase.production-capability`／`mirakan.arch.rendering-vfx-runtime` | Windows、Android、Apple | `capability.vfx.system; capability.vfx.particle_cpu; capability.rendering.render-graph-core` | `wp.rendering.vfx-runtime-2d-c1; wp.rendering.vfx-authoring-c2; wp.platform.windows-package; wp.platform.android-package; wp.platform.apple-package` | `particle_gpu; mesh_ribbon; visual_collision; particle_light` |

各行のTargetはRegistryのexact IDへ`Windows=target.windows.desktop`、`Android=target.android.mobile`、`Apple=target.apple.mobile`として展開する。ここでAuthoring WPのTargetはAuthoring processを実行するHostではなく、そのSource／Compilerが生成・検証するdestination Targetである。Headless／Windows Editor上の保存・UI・AI operation availabilityは既存Authoring／Operation Activationが別に所有し、VFX Capabilityのdestination Target scopeへ混在させない。build-host orderingは`requires WP`だけで表し、Runtime Target supportはArtifact QualificationとRuntime Capability rowで判定する。

既存13 CapabilityのTier、Target scope、fallbackは変更せず、`capability.vfx.typed_subgraph`だけをC1、Windows／Android／Apple required、`fallback.rendering.vfx-core`で追加する。すなわち`semantic_intent, pattern_catalog, typed_subgraph, bake_cache, extension_operator`のowner WPはAuthoring、`system, particle_cpu, sprite_2d, trail, billboard_3d, particle_gpu, mesh_ribbon, visual_collision, particle_light`のowner WPはRuntimeである。Extension ArtifactのAndroid／Apple利用可否は既存どおりoptionalで、Target別Extension Activation ProjectionとRuntime Artifact Qualificationを満たす場合だけ選択する。

| Capability ID | Tier | Owner WP | Target scope | Fallback |
|---|---|---|---|---|
| `capability.vfx.semantic_intent` | C1 | `wp.rendering.vfx-authoring-c1` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.vfx.pattern_catalog` | C1 | `wp.rendering.vfx-authoring-c1` | Windows required; Android required; Apple required | `fallback.rendering.vfx-core` |
| `capability.vfx.typed_subgraph` | C1 | `wp.rendering.vfx-authoring-c1` | Windows required; Android required; Apple required | `fallback.rendering.vfx-core` |
| `capability.vfx.system` | C1 | `wp.rendering.vfx-runtime-2d-c1` | Windows required; Android required; Apple required | `fallback.rendering.vfx-core` |
| `capability.vfx.particle_cpu` | C1 | `wp.rendering.vfx-runtime-2d-c1` | Windows required; Android required; Apple required | `fallback.rendering.vfx-core` |
| `capability.vfx.sprite_2d` | C1 | `wp.rendering.vfx-runtime-2d-c1` | Windows required; Android required; Apple required | `fallback.rendering.vfx-cpu` |
| `capability.vfx.trail` | C1 | `wp.rendering.vfx-runtime-2d-c1` | Windows required; Android required; Apple required | `fallback.rendering.vfx-cpu` |
| `capability.vfx.billboard_3d` | C1 | `wp.rendering.vfx-runtime-3d-c1` | Windows required; Android required; Apple required | `fallback.rendering.vfx-cpu` |
| `capability.vfx.bake_cache` | C2 | `wp.rendering.vfx-authoring-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-core` |
| `capability.vfx.extension_operator` | C2 | `wp.rendering.vfx-authoring-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-core` |
| `capability.vfx.particle_gpu` | C2 | `wp.rendering.vfx-runtime-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-cpu` |
| `capability.vfx.mesh_ribbon` | C2 | `wp.rendering.vfx-runtime-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-cpu` |
| `capability.vfx.visual_collision` | C2 | `wp.rendering.vfx-runtime-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-core` |
| `capability.vfx.particle_light` | C2 | `wp.rendering.vfx-runtime-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-core` |

Product Phase registryはPhase 3へ`wp.rendering.vfx-authoring-c1; wp.rendering.vfx-runtime-2d-c1`、Phase 6へ`wp.rendering.vfx-runtime-3d-c1`を追加し、Phase 8の`wp.rendering.vfx-c2`を`wp.rendering.vfx-authoring-c2; wp.rendering.vfx-runtime-c2`へ置換する。

`wp.product.general-coverage-2d`はrequired Capabilityを`capability.vfx.system; capability.vfx.particle_cpu; capability.vfx.sprite_2d`、requires WPを`wp.rendering.vfx-runtime-2d-c1`へ固定する。`wp.product.general-coverage-3d`は`capability.vfx.system; capability.vfx.particle_cpu; capability.vfx.billboard_3d`、`wp.rendering.vfx-runtime-2d-c1; wp.rendering.vfx-runtime-3d-c1`へ固定する。両行から削除済み`wp.rendering.vfx-c2`を除き、C2 optional CapabilityをGeneral coverageのrequired edgeへ変換しない。C2二WPはPhase 8自身が追跡し、General coverageのC1 VFX成立条件と相互代用しない。

各WPの`provided fixtures`、`fallback`、`scheduling_state`は既存Registry語彙だけを使う。C1 Authoringは`fixture.product.authoring-transaction`とgenreless／Shooter／Platformer／Puzzle 2D、C1 Runtime 2Dはgenreless／Shooter／Platformer／Puzzle 2D、C1 Runtime 3DはShooter Arena 3D、C2二件は既存VFX C2のShooter／Platformer／Puzzle／Shooter Arena集合へ固定する。全5件は`declared`、fallbackはAuthoring C1=`fallback.capability.unavailable`、Runtime 2D=`fallback.rendering.vfx-core`、Runtime 3D=`fallback.rendering.vfx-cpu`、Authoring C2=`fallback.rendering.vfx-core`、Runtime C2=`fallback.rendering.vfx-core`とする。

24個のVFX AI OperationはWork Packageの存在からActivationしない。`activation.vfx.authoring_operations.v1`はPhase 4以後、AI Authoring Core、全24 Operation MCD、Policy、Validator、Diagnostic、Fixture、Receiptを同じContract set transactionで登録した時だけ有効になる。Manual Authoring CapabilityとAI Operation Activationを相互代用しない。

## 12. AI解決規則

AIは次の順で処理する。

1. Effect Intent、dimension、temporal role、gameplay relation、Style、Target、scale、Minimum Cueを解決する。
2. Core Pattern候補を最大3件提示する。
3. Core Patternで不足する場合だけVersion付きTyped Subgraph候補を最大3件提示する。
4. Core／SubgraphでCueを満たせない場合だけExtension requirement、Target cost、fallback、Approvalを提示する。
5. dimension、Gameplay authority、Target、Minimum Cue、Extension policyのいずれかが未解決なら質問へ停止する。
6. human lock、base revision、Source hash、Profile ref、Target ref、operation activationを検証してChangeSet proposalを作る。
7. Preview、validation、Approval、Commit、read-backが揃うまで成功と表示しない。

AIはCPU／GPU thread、buffer、shader model、sort implementationを初心者へ質問しない。CompilerがTarget ProfileとQualificationから解決できない場合は、AIが推測せずCapability Gapを返す。

## 13. 有名Engineから採用する点と採用しない点

調査対象は2026-07-28時点の公式資料である。

- Unreal Engine Niagaraから、System／Emitter／Module／Parameterの段階、Version付きModule／Emitter、Effect Type budget、Debugger、初心者向けSummary、Stateless Emitterの考え方を参照する。
- Unity Visual Effect Graphから、Context／Block／Operatorの分離、Subgraph Asset再利用、exposed property／event、step previewを参照する。
- Godotから、CPU fallback、Fixed FPSの明示、Visibility Bounds、簡潔なProperty authoringを参照する。

参考:

- <https://dev.epicgames.com/documentation/en-us/unreal-engine/overview-of-niagara-effects-for-unreal-engine>
- <https://dev.epicgames.com/documentation/en-us/unreal-engine/versioning-modules-and-emitters-in-niagara-effects-for-unreal-engine>
- <https://dev.epicgames.com/documentation/en-us/unreal-engine/performance-budgeting-using-effect-types-in-niagara-for-unreal-engine>
- <https://docs.unity3d.com/Manual/ChoosingYourParticleSystem.html>
- <https://docs.unity3d.com/Packages/com.unity.visualeffectgraph@17.0/manual/index.html>
- <https://docs.godotengine.org/en/4.6/tutorials/3d/particles/properties.html>

次は採用しない。

- Graph内の任意HLSL、任意C++、JIT、native GPU API。
- unversioned shared Moduleの変更を全利用Assetへ即時伝播する方式。
- GPU Particle readbackをGameplay判断へ利用する方式。
- CPU／GPU backendをruntime frame負荷で切り替える方式。
- Source名、Asset名、作品名、近い表示名からSemantic Roleを推測する方式。

## 14. Architecture計画書への反映範囲

| 文書 | 反映内容 |
|---|---|
| `06-rendering/vfx-authoring.md` | 三層構造、Typed Subgraph、VfxStyleProfile、exact Budget／Quality Profile、Parameter分類、Capability追加、Qualification |
| `06-rendering/vfx-runtime.md` | exact Profile Ref、receipt-free Artifact、Activation Binding、Parameter適用、Pass Template Set ref、Presentation reconstruction |
| `03-authoring/native-game-module.md` | VFX関連Link、CPU ExtensionがRevision Refとbounded entryだけを使う非所有境界 |
| `06-rendering/materials.md` | VFX関連Link、AlphaMode拡張、VfxStyleProfile exact ref規則 |
| `06-rendering/render-graph.md` | VfxPassTemplateSet、resource／queue／barrier ownership、Snapshot不在時のPass規則 |
| `06-rendering/lod.md` | VFX関連Link、VfxLodProfileRef、Quality bindingとのexact equality |
| `04-runtime/scheduling-lifetime.md` | VFX関連Link、T90 parameter／CPU／GPU record適用境界 |
| `04-runtime/performance-capacity.md` | VFX関連Link、standalone／aggregate budgetの二重Gate |
| `04-runtime/persistence-save.md` | VFX関連Link、Presentation Companion／Manifest、将来SaveCatalogV3、authoritative digest除外 |
| `04-runtime/runtime-package.md` | VFX二Artifact RoleのCatalog reachability、Activation Binding依存、load前Ready判定への委譲 |
| `03-authoring/asset-lifecycle.md` | VFX関連Link、VFX Artifact payloadとgeneric Derived Manifestの境界 |
| `07-platform/ui-text-localization-accessibility.md` | SaveCatalogV2を不変に保つV3 slot、Companion ref、Settings／Catalog atomic publication |
| `00-product/product-plan.md` | Continueのauthoritative V2経路維持と、V3後段Presentation再構築の非authoritative境界 |
| `02-foundation/executable-contracts.md` | Receipt-free Artifact、Qualification Subject、signed Receipt、root外Bindingの共通生成順との整合 |
| `appendices/executable-contracts-operation-planning-catalog.md` | exact 24 Operation集合維持、typed_subgraph操作が必要な場合の新family version方針 |
| `appendices/product-execution-registry-proposal.md` | 5 Work Packageへの分割、Phase 3／6／8、14 Capabilityへの更新 |
| `appendices/ai-evidence-envelope-fixture-catalog.md` | Core／Subgraph／Extension、Style、Blend、Receipt、Save、Pass、budgetのfixture軸 |

既存24 OperationへSubgraph作成／更新を黙って割り当てない。Subgraph編集をAI OperationとしてActivationする場合は、既存family version 1を変更せず`planning.operation_family.vfx_authoring@2`を新設し、正確なOperation集合、Risk class、Validator、Fixtureを同一ChangeSetで定義する。今回のArchitecture整理ではversion 2をactiveにせず、将来のActivation条件だけを記載する。

## 15. Validation

Architecture変更は次をすべて満たす。

1. 全相対Markdown Linkの対象fileが存在する。
2. VFX AuthoringとOperation Catalogのversion 1 Operation集合がexact 24件で一致する。
3. VFX AuthoringとProduct Capability表の既存13件に`capability.vfx.typed_subgraph`を加えたexact 14件が一致する。
4. `VfxStyleProfileV1`とRefがVFX Authoringに一意所有され、MaterialsはRefだけを持つ。
5. VFX固有Blend enumが残らず、全VFX outputがMaterials AlphaModeを参照する。
6. Receipt-free Artifactから`verification_receipt_refs[]`が除去され、ReceiptはQualification Subject後段だけに存在する。
7. `PersistentVfxDescriptorV1 -> VfxPresentationReconstructionProjectionV1 -> PresentationReconstructionManifestV1 -> RuntimeSessionPresentationCompanionV1 -> SaveCatalogV3 slot`の一方向参照経路が存在し、`RuntimeSessionSaveBundleV1`とauthoritative digestから到達しない。
8. VFX Pass IDがRender Graphに一意所有され、VFX RuntimeはSet refと意味順だけを持つ。
9. `presentation_frame`という曖昧なParameter update rateが残らない。
10. Phase 3／6／8とVFX Work Packageの対応が一意で、C1 CapabilityがPhase 8だけをOwnerにしない。
11. `VfxBudgetProfileV1`、`VfxQualityProfileV1`、Target／Budget／LOD Refのplain IDがArtifact closureに残らない。
12. VFX Artifact Catalogにexact二Roleだけがあり、Activation Bindingからreceipt-free Artifact Setへの一方向dependencyとPackage reachabilityが成立する。
13. 仮置きtoken、定義のないID／Field、明示されていない既定値を追加しない。
14. Git diffでVFX範囲外の無関係な設計変更がない。

## 16. 完了条件

この設計整理の完了は、Engine機能の実装完了ではない。次を満たした時点でArchitecture計画書の整理完了とする。

- §14の文書が同じ判断へ整合する。
- §15の全Validationがpassする。
- Owner、Schema、Ref、hash、生成順、failure、fallback、Phaseが相互に矛盾しない。
- 未実装の型、Registry、Operation、Fixtureは`review`／`absent`／`not_activated`として表示される。
- Runtime、Editor、AI、Packageで利用可能であるという主張を追加しない。
- Engine実装Task、担当、工数、日程、実装計画を作成しない。
