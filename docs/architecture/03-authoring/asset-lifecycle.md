# Miraikanai Engine Asset Lifecycle Contract

- 文書ID: mirakan.arch.asset-lifecycle
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Asset source／import identity、Import Profile／Plan／IR、Preview／Conversion Report、Reimport／dependency invalidation、Asset／World artifact共通のDerived／Cooked envelope・Catalog・content addressing、Asset Content Package assembly、Asset promotion、Editor／AI Asset operation、Asset diagnostics／qualification
- 非正本範囲: Project transaction、共有Schema基盤、外部Tool・SDK・Libraryのversion／hash／license／取得元、Runtime scheduling／lease／capacity、ECS storage、World Root／Section payload・Runtime Package binary、Save／Replay、各DomainのRuntime意味。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Project State](project-state.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)
- 関連文書: [AI-readable Asset／Memory／Async Loading Alignment](../decisions/2026-07-28-ai-asset-memory-async-alignment.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core architecture](../02-foundation/core-architecture.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Naming／Project layout](../02-foundation/naming-project-layout.md)、[Project state](project-state.md)、[Editor Workspace UX](editor-workspace-ux.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[World契約](../06-rendering/world.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-21

AssetをEditorが直接読むSource fileではなく、次の閉じたlifecycleとして扱う。Asset versionのgeneration、immutable read lease、promotion後のretireは本書がAsset固有の意味を所有し、公開型、保存／job capture、allocation／fallbackの一般規則は[Memory／Pointers](../02-foundation/memory-pointers.md)の`PointerMemoryConsumerBindingV1`へexactに束縛する。Asset readerがlive payload pointerやleaseをSave／Package／AI projectionへ渡すことはない。

```text
Source Asset
  -> Source analysis
  -> Import Profile／Plan
  -> typed Import IR
  -> Preview／Conversion Report／Approval
  -> deterministic Cook
  -> immutable Derived Artifact closure
  -> Catalog／Content Package assembly
  -> Runtime staging／atomic promotion
```

Runtime、Renderer、Physics、Navigation、Animation、AudioはSource fileを直接読まない。AI、Editor、CLIもCooked binary、Package index、GPU／Physics native objectを直接生成または変更しない。Import／ReimportとPackage assemblyはこの文書だけが所有し、前者の結果を後者が消費する同一lifecycleの別stageとする。

本文書の`AssetSourceDescriptor`、`AssetImportJob`、`AssetSourceAnalysisV1`、`AssetImportProfileV1`、`CommonImportSettingsV1`、Asset kind別Settings／IR、`AssetImportPlanV1`、`AssetConversionReportV1`、`AssetDiagnosticV1`、`AssetImportReceiptV1`、`AssetReimportConflictV1`、`TypedConflictValueV1`、`ArtifactSubjectRefV1`、`DerivedArtifactManifestV1`、`RuntimeAssetBudgetRefV1`、`MirakanArtifactCatalogV1`はAsset Lifecycle Ownerのcanonical MCD schemaである。[Executable contracts](../02-foundation/executable-contracts.md)は共通Envelope、projection、compiler規則を所有し、Asset固有field、Asset固有tag、cardinalityは本文書だけが所有する。World Root／Sectionのpayload fieldは[Runtime Package](../04-runtime/runtime-package.md)だけが所有する。`risk_class`、`RiskApprovalPolicyV1`等の共有Governance型とそのtag集合はこの所有宣言に含めず、[AI Security／Approval](../01-governance/ai-security-approval.md#4-risk-classとactivation)のcanonical type ID／versionを参照する。

## 1. Source／Import identity

### 1.1 Assetの四層

1. **Source Asset**: 人間、DCC、AIが作る原本。Project source tree内の相対pathとbytesを持つ。
2. **Import Document**: Stable ID、Source解析、意味、Profile、権利、来歴、revisionを持つAuthoring source。[Project state §3.1](project-state.md#31-正規document)の`AssetMetadataDocument`と同一のDocumentであり、そのcanonical本文schemaが本文書所有の`AssetSourceDescriptor`である。Project stateは役割とID／revision規律を、本文書がfield setを所有する。
3. **Derived Artifact**: 明示Targetへ決定論的にCookしたimmutable runtime product。
4. **Catalog／Content Package**: 配布、mount、依存closure、content addressingの単位。

`AssetId`はUUIDv7 `StableId`であり、rename、move、Reimportで変えない。`AssetRevision`はSource bytesまたは意味あるImport設定の変更ごとに単調増加する。同じSourceから複数Artifactを作る場合は`AssetId`とstable `ArtifactRoleId`で識別する。Path、display name、配列index、content hashだけを永続identityにしない。

`AssetSourceDescriptor`は次を持つ。

| Field | 規則 |
|---|---|
| `asset_id` | UUIDv7 `StableId` |
| `asset_type` | closed Asset Type ID |
| `asset_revision` | `uint64` |
| `source_kind` | `domain_pack_reference`、`user_provided`、`external_generated`のclosed enum |
| `source_origin` | `source_kind`で判別するclosed payload。以下の必須Fieldだけを持つ |
| `source_relative_path` | Project source root内のcanonical relative path |
| `source_sha256` | Source file bytesのSHA-256 |
| `source_media_type` | closed allowlist enum |
| `import_profile_id` | typed Profile `StableId` |
| `import_settings` | Asset Type別MCD object |
| `declared_dependencies` | `AssetDependencyRefV1[0..4096]`。各要素は`AssetId`＋role |
| `license_record_id` | `StableId`、必須 |
| `provenance_record_id` | `StableId`、必須 |
| `safety_receipt_id` | `optional StableId`。`external_generated`では必須 |
| `editor_tags` | `ClosedAssetTagId[0..64]` |

`source_origin`は`domain_pack_reference`で`pack_id`、`pack_version`、`pack_content_sha256`、`reference_asset_id`、`reference_asset_sha256`、`user_provided`で`project_source_record_id`と`ingest_receipt_ref`、`external_generated`で`generation_operation_ref`、`tool_model_lock_ref`、`generation_tool_receipt_ref`を必須とする。別kindのField、自由形式provider名、remote URLを混在させず、kind変更は新`asset_revision`とApprovalを必要とする。

Pathは`/`へ正規化し、absolute path、drive、UNC、`..`、empty segment、NUL、reserved device name、case-fold衝突を拒否する。logical path比較はUnicode NFCとProject canonical keyを使い、同一keyの二つのSourceを許可しない。Source dependencyはBrokerが解決したmanifestへ閉じ、Importerが実行中に任意pathを探索してはならない。

### 1.2 Import jobと状態

`AssetImportJob`は次のcanonical tupleを持つ。

```text
AssetImportJob
  job_id: StableId
  asset_id: StableId
  asset_revision: uint64
  source_hash: sha256
  import_profile_hash: sha256
  importer_id: ClosedImporterId
  importer_version_hash: sha256
  target_profile_id: StableId
  dependency_artifact_keys: sha256[0..4096]
  toolchain_lock_hash: sha256
  limits: AssetImportJobLimitsV1
```

`AssetImportJobLimitsV1`はCPU time、wall time、commit memory、output bytes、output file countのpositive hard limitだけを持つ。Job keyは上記tupleのcanonical bytesから作る。Timestamp、machine path、user、worker completion順をcache identityにしない。`import_profile_hash`はflatten済みProfile hash、`importer_version_hash`は[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)のexact lockを参照するhashであり、本文書へ外部versionを複写しない。

Job状態はclosed setであり、許可遷移を次表だけに限定する。

| from | to | 条件 |
|---|---|---|
| `Discovered` | `Analyzing` | Job実行開始 |
| `Analyzing` | `Planned` | Analysis成功かつProfile確定済み |
| `Analyzing` | `NeedsProfile` | Analysis成功だがProfile未確定（`AmbiguousSemanticRole`等） |
| `Analyzing` | `AnalysisRejected` | Analysis失敗 |
| `Planned` | `Previewing` | Plan成立、blocking question 0件 |
| `Previewing` | `AwaitingApproval` | Preview成功かつ`RiskApprovalPolicyV1`がApproval必要と判定 |
| `Previewing` | `Cooking` | Preview成功かつ`RiskApprovalPolicyV1`がApproval不要と判定。skip判定と根拠をReceiptへ記録 |
| `Previewing` | `PreviewRejected` | Preview失敗 |
| `AwaitingApproval` | `Cooking` | Approval取得 |
| `AwaitingApproval` | `PreviewRejected` | Approval却下 |
| `Cooking` | `Validating` | Cook出力完了 |
| `Cooking`／`Validating` | `CookRejected` | Cookまたはvalidation失敗 |
| `Validating` | `ReadyToPromote` | validation合格 |
| `ReadyToPromote` | `Active` | [Hot reload／promotion](#7-hot-reloadpromotion)のpromotion boundary成立 |
| 全非終端state | `Cancelled` | Cancel、timeout、worker crash、stale revision |

`Active`は成功の終端、`NeedsProfile`、`AnalysisRejected`、`PreviewRejected`、`CookRejected`、`Cancelled`は非成功の終端であり、同一Jobで成功pathへ再入しない。再試行はSourceまたはProfile変更のCommitが作る新しいJobとして`Discovered`から始め、Analysis結果の再利用はJob key cacheだけで行う。

Source解析はProjectを変更しない。Import設定変更はImport Documentへ対する[ProjectChangeSetV1](project-state.md#5-projectchangesetv1)であり、Commit後にだけ新しいAsset revisionとJobを作る。Cancel、timeout、worker crash、stale revisionはProject状態を変更せず、旧Active generationを維持する。

## 2. Import Profileと検出

### 2.1 Analysis、Profile、Plan

```text
AssetSourceAnalysisV1
  asset_id: StableId
  asset_revision: uint64
  source_media_type: ClosedMediaTypeId
  source_hash: sha256
  format_version: string
  format_features: ClosedFeatureId[0..256]
  source_coordinate_system: optional CoordinateSystemDescriptorV1
  source_unit_scale_meters: optional positive_f64
  source_color_encoding: optional ColorEncodingDescriptorV1
  source_audio_layout: optional AudioLayoutDescriptorV1
  dependency_candidates: SourceDependencyCandidateV1[0..4096]
  structural_summary: AssetStructuralSummaryV1
  unsupported_features: ClosedFeatureId[0..256]
  analyzer_receipt: ToolReceiptV1
```

Source名、Directory、DCC名だけからsemantic roleを確定しない。規範metadata、Project Profile、明示User設定だけを根拠とする。

```text
AssetImportProfileV1
  profile_id: StableId
  schema_version: uint32
  asset_kind: texture | sprite | scene_3d | mesh | skeleton
            | animation | audio | font
  target_profile_scope: StableId[1..16]
  common: CommonImportSettingsV1
  settings: tagged union by asset_kind (AssetKindImportSettingsV1)
  fallback_policy: forbid | allow_listed
  approval_policy: RiskApprovalPolicyV1
```

`CommonImportSettingsV1`は次の4 fieldだけを持つ。

| Field | 型／規則 |
|---|---|
| `dependency_policy` | `AssetDependencyPolicyV1` |
| `unknown_metadata_policy` | `UnknownMetadataPolicyV1` |
| `warning_promotion_policy` | `WarningPromotionPolicyV1` |
| `budget_class` | `AssetBudgetClassId` |

`AssetKindImportSettingsV1`は`asset_kind`と一致する次のclosed tagged unionである。

| `asset_kind` | `settings` variant | Duplicate disposition |
|---|---|---|
| `texture` | `TextureImportSettingsV1`、`sprite=none` | Texture共通fieldを別schemaへ複写しない |
| `sprite` | `TextureImportSettingsV1`、`sprite=some SpriteImportSettingsV1` | SpriteはTexture設定を内包しない |
| `scene_3d` | `SceneImportSettingsV1` | 旧Transform policy表をtyped schema化 |
| `mesh` | `MeshImportSettingsV1` | Scene transformはSource analysis／IRから継承 |
| `skeleton` | `SkeletonImportSettingsV1` | 旧Skeleton validation policyをtyped schema化 |
| `animation` | `AnimationImportSettingsV1` | Skeleton binding fieldを同schemaで参照 |
| `audio` | `AudioImportSettingsV1` | Audio以外のfieldを持たない |
| `font` | `FontImportSettingsV1` | Font以外のfieldを持たない |

Profile inheritanceは一段だけ許可し、最終的なflatten済みProfile hashをJob keyとReceiptへ保存する。Asset kind固有fieldをuntyped property bagへ入れず、tagとvariant不一致をSchema errorにする。

```text
AssetImportPlanV1
  plan_id: StableId
  asset_id: StableId
  base_asset_revision: uint64
  base_project_revision: uint64
  source_analysis_hash: sha256
  flattened_profile_hash: sha256
  selected_subresources: StableSourcePathV1[0..4096]
  explicit_conversions: ImportConversionV1[0..256]
  predicted_artifacts: ArtifactRoleId[1..256]
  predicted_runtime_cost: AssetCostEstimateV1
  blocking_questions: ImportQuestionV1[0..32]
  risk_class: Governance-owned Risk class reference
```

`risk_class`は[Governance Risk class](../01-governance/ai-security-approval.md#4-risk-classとactivation)のMCD `risk_class` type ID／versionを参照し、Asset schemaはvariantを列挙、制限、拡張しない。Blocking questionが一件でもあればCook承認対象にしない。AIが未回答を既定値で埋めることを禁止する。低影響で承認済みProfileが一意な項目だけEngineが解決し、由来をPlanへ記録する。

### 2.2 Asset kind settings

```text
TextureImportSettingsV1
  semantic_role: base_color | emissive | normal | orm | mask
               | height | hdr_environment | ui | sprite | data
  color_encoding: srgb | linear | source_icc_to_scene_linear
  alpha_mode: opaque | straight | premultiplied | mask
  normal_convention: none | tangent_plus_y | tangent_minus_y
  mip_policy: none | generate_color | generate_normal | preserve_source
  resize_policy: preserve | max_dimension
  max_dimension: uint32
  compression_profile: StableId
  streaming_policy: resident | streamed
  channel_mapping: TextureChannelMappingV1
  sprite: optional SpriteImportSettingsV1
```

`max_dimension`は`resize_policy=max_dimension`の時だけnonzeroを必須とする。`semantic_role`が`normal`、`orm`、`mask`、`height`、`data`の場合にsRGB Cookを拒否する。

```text
SpriteImportSettingsV1
  rect_mode: SpriteRectModeV1
  rects: SpriteRectV1[0..65535]
  grid_cell: optional SpriteGridCellV1
  grid_margin: optional Extent2uV1
  grid_spacing: optional Extent2uV1
  pivot: SpritePivotV1
  pixels_per_unit: positive_f32
  nine_slice_border: optional SpriteBorderV1
  trim_policy: SpriteTrimPolicyV1
  packing_policy: SpritePackingPolicyV1
  atlas_group: optional StableId
  allow_rotation: bool
  extrude_texels: uint32
```

Sprite rectはboundedで、単一Texture内の有効Sprite数は65,535以下、atlas pageは4,096×4,096 texel以下とする。Textureのcolor、alpha、mip、compression、streaming設定を`SpriteImportSettingsV1`へ重複保存しない。

旧3D Transform policy表を次のtyped settingsへ正規化する。

```text
SceneImportSettingsV1
  hierarchy_policy: preserve
  root_transform_policy: preserve
  pivot_policy: preserve
  placement_policy: preserve_source
  front_policy: preserve_source
  unit_policy: convert_to_meters
  matrix_policy: require_decomposable_trs
```

上記closed value以外は別versionのSchemaなしに追加しない。Static Meshの明示`bake_geometry`は`ImportConversionV1`であり、Scene settingsの暗黙defaultにしない。

```text
MeshImportSettingsV1
  lod_source_mode: disabled | source_chain | generated_chain | hybrid_chain
  source_lod_bindings: SourceLodBindingV1[0..16]
  mesh_lod_profile_id: StableId?
  preserve_boundaries: bool
  preserve_uv_seams: bool
  preserve_hard_normals: bool
  required_vertex_color_channels: ClosedChannelId[0..8]
  skin_policy: source_only | qualified_generated
  morph_policy: source_only | qualified_generated
```

`SkeletonImportSettingsV1`は旧Skeleton validation policyをfield化し、任意propertyを許可しない。

| Field | 型／規則 |
|---|---|
| `joint_path_policy` | `require_unique_stable_path` |
| `parent_policy` | `reject_cycle` |
| `bind_pose_policy` | `require_finite_invertible` |
| `weight_policy` | `require_finite_nonnegative_normalized` |
| `influence_cap` | `positive_uint32`、Target Profile上限以下 |

```text
AnimationImportSettingsV1
  skeleton_policy: require_embedded | bind_existing_exact
  clip_extraction: embedded_clips | explicit_ranges
  sample_policy: preserve_keys | bake_fixed_rate
  bake_rate_hz: optional positive_f32
  interpolation_policy: preserve_supported | bake_unsupported
  key_reduction_profile: StableId
  root_motion_policy: preserve_track | extract | discard
  root_joint: optional StableSourcePathV1
  event_source_policy: none | registered_metadata
  retarget_policy: none | approved_humanoid_profile
```

`bake_rate_hz`は`sample_policy=bake_fixed_rate`の時だけ必須である。Standalone Skeletonは`SkeletonImportSettingsV1`、Animationは`AnimationImportSettingsV1`を使い、同じ`SkinIRV1` identityを参照する。

```text
AudioImportSettingsV1
  semantic_role: sfx | ui | dialogue | music | ambience
  channel_policy: preserve_mono_stereo | downmix_to_mono | downmix_to_stereo
  sample_rate_policy: cook_48000
  trim_policy: preserve | trim_explicit_range
  gain_policy: preserve | apply_explicit_db
  loop_policy: none | explicit_frames | source_markers
  streaming_policy: auto_profile | resident | streamed
  codec_profile: StableId
  locale: optional LocaleId
```

`trim_explicit_range`、`apply_explicit_db`、`explicit_frames`のpayloadは`ImportConversionV1`に置き、settingsへuntyped scalarを追加しない。無承認の音量正規化を行わない。

`FontImportSettingsV1`は旧Font契約を次のrequired fieldへ固定する。

| Field | 型／規則 |
|---|---|
| `required_locales` | `LocaleId[1..256]` |
| `required_scripts` | `ScriptId[1..256]` |
| `glyph_coverage_policy` | `FontGlyphCoveragePolicyV1` |
| `variable_axis_policy` | `FontVariableAxisPolicyV1` |
| `color_glyph_policy` | `FontColorGlyphPolicyV1` |
| `hinting_policy` | `FontHintingPolicyV1` |
| `fallback_families` | `FontFamilyRefV1[0..64]` |
| `raster_policy` | `FontRasterPolicyV1`。atlas／Runtime rasterを区別 |
| `embedding_permission_record` | `StableId`、必須 |

### 2.3 typed Import IRとAsset別検出

Asset IRは次の4 canonical rootへ統合する。SpriteはTexture、Mesh／Skeleton／AnimationはScene rootを共有し、Source形式別IR aliasを増やさない。

```text
TextureImportIRV1
  extent: uint32 width／height／depth
  layers／faces／levels: bounded uint32
  source_pixel_encoding: ClosedPixelEncodingId
  color_encoding: ColorEncodingDescriptorV1
  alpha_mode: opaque | straight | premultiplied | mask
  semantic_role: ClosedTextureRoleId
  channel_mapping: TextureChannelMappingV1
  decoded_level_hashes: sha256[1..32]
  sprite_records: SpriteRecordIRV1[0..65535]

SceneImportIRV1
  scene_roots: SceneNodeId[1..256]
  nodes: SceneNodeIRV1[1..65536]
  meshes: MeshIRV1[0..16384]
  materials: MaterialBindingIRV1[0..16384]
  skins: SkinIRV1[0..1024]
  animations: AnimationIRV1[0..4096]
  cameras: CameraIRV1[0..256]
  lights: LightIRV1[0..256]
  source_coordinate_system: CoordinateSystemDescriptorV1
  canonicalization: SceneCanonicalizationV1

AudioImportIRV1
  sample_rate_hz: positive_uint32
  channel_layout: ClosedAudioLayoutId
  sample_format: pcm_s8 | pcm_s16 | pcm_s24 | pcm_s32 | float32
  frame_count: uint64
  loop_range: optional FrameRangeV1
  measured_loudness_lufs: finite_f32
  measured_true_peak_dbfs: finite_f32
  decoded_pcm_hash: sha256

FontImportIRV1
  face_records: FontFaceIRV1[1..64]
  unicode_coverage: UnicodeRangeSetV1
  script_coverage: ScriptCoverageV1[1..256]
  variation_axes: FontVariationAxisV1[0..64]
  color_capabilities: ClosedFontColorCapabilityId[0..16]
  embedding_permission: FontEmbeddingPermissionV1
  normalized_table_hashes: FontTableHashV1[1..256]
```

旧Aseprite節の`SpriteImportIRV1`名は独立rootとして残さず、frame、tag、duration、slice／pivot、layerを`TextureImportIRV1.sprite_records`とtyped Conversion Reportへ正規化する。Standalone Skeleton／Animationも`SceneImportIRV1.skins`／`animations`の`SkinIRV1`／`AnimationIRV1`を使用する。IRはSource native object、decoder pointer、DCC property bagを保存しない。Source形式が増えてもRuntime Asset schema、planned AI action vocabulary、Cook入口を分岐させない。

| Kind | 必須の検出／validation |
|---|---|
| Texture／Sprite | dimension、level、channel、alpha、color encoding、normal convention、Sprite rect／pivot／PPU、atlas bound |
| Scene／Mesh | accessor bounds、node cycle、finite transform、extension allowlist、index、degenerate、bounds、skin influence |
| Skeleton／Animation | stable joint path、parent cycle、bind pose、key順、Quaternion、root motion、Skeleton generation |
| Audio | sample rate、channel layout、duration、sample finite、loop point、decoded／stream cost |
| Font | table bounds、glyph／script coverage、variation、embedding permission、outline bounds |

TileSet／TilemapはEngine-native authoring経路であり、[World契約 §10.1](../06-rendering/world.md#101-tilemap-sourcecookpublication)の12型とそのvalidationが唯一の正本である。外部tilemap formatのimportは採用せず、本表へTileSet／Tilemap行を複写しない。Asset lifecycleはTileSetが参照するSpriteが本表のTexture／Sprite経路でimport済みであることだけを保証する。

Scene Importのcanonical spaceは右手系、`+Y` up、`+Z` object forward、meter、radian、column vector、`T * R * S`である。Hierarchy、root transform、pivot、placement、frontはSource意図を保持し、unit変換はSource metadataに基づく明示処理だけを適用する。非分解可能transformや意味不明な変換を推測で補正しない。

Textureはsemantic role、color／alpha、normal convention、mip、resize、compression Profile、streaming、channel mappingをtyped settingsにする。Sprite IDをframe番号や配列indexから導出せず、Reimport時にconsumer closureと対応候補を提示する。Normal、data、maskをsRGBとしてCookすることをhard errorにする。

AnimationはSkeleton binding、clip extraction、sample／interpolation、root motion、event、retarget Profileを明示する。Audioはchannel、sample rate、trim、gain、loop、residency、codec Profileを明示し、無承認の音量正規化を行わない。Fontはrequired locale／script、coverage、variation、color glyph、hinting、fallback、raster policy、embedding permissionを明示する。

外部format Adapterのversion、実行物hash、license、取得元は[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)だけが固定する。ここではAdapterをCapabilityとして扱い、未ActivationのformatをCatalogへ出さない。DCC converterは別sandboxで実行し、Engine-native typed IRへ変換する。Source IDやproperty名からComponent、Gameplay Event、Asset pathを推測生成しない。

## 3. PreviewとConversion Report

Previewは正式Artifactと同じIR、Validator、Cook codeを使い、Preview専用の黙った補正を禁止する。Preview Artifactはpresentation用であり、Production Artifact keyまたはShipping closureへ自動追加しない。

```text
AssetConversionReportV1
  source_analysis_hash: sha256
  plan_hash: sha256
  importer_id: ClosedImporterId
  importer_version_hash: sha256
  source_tool_receipts: ToolReceiptV1[0..8]
  applied_conversions: AppliedConversionV1[0..256]
  preserved_features: ClosedFeatureId[0..256]
  dropped_features: LossRecordV1[0..256]
  diagnostics: AssetDiagnosticV1[0..1024]
  before_summary: AssetStructuralSummaryV1
  after_summary: AssetStructuralSummaryV1
  preview_artifact_key: optional sha256
  deterministic_run_hashes: sha256[2]
```

`LossRecordV1`はSource path、feature ID、理由、visual／behavior impact、承認可否を持つ。unsupported featureをlossへ記録しただけで成功にせず、Profileが列挙したfallbackと必要なApprovalがある場合だけCook候補にする。

Asset種別Previewは少なくとも次を示す。

- Scene／Mesh: Source／Engine軸、root、pivot、bounds、transform、Hierarchy diff、Skeleton、Collider／Navigation予定、LOD source chain。
- Texture／Sprite: Source、scene-linear、Target compressed、alpha、channel、normal、mip、rect、pivot、PPU、GPU byte見積り。
- Animation: Skeleton tree、axis、bind／reference pose、skin weight、clip scrub、root trajectory、loop seam、圧縮error。
- Audio: waveform、sample-accurate loop、loudness、true peak、channel、trim、resident／stream cost、Target audition。
- Font: required script、fallback、missing glyph、variation、small／large size、Target raster差。

PreviewはSource、Profile、Target、Artifact、consumer impactを同時に比較できなければ承認可能状態にしない。Editor scrubはGameplay、Audio、VFXのauthoritative Eventを発火しない。

`AssetImportReceiptV1`はSource／Profile／Plan／IR／Artifact／Preview／Approval／Toolchain／Validatorの各hash、生成Hostの識別（toolchain lock `profiles[].host`に対応するOS／architecture）、Target、実行budget、全Diagnostic count、Package eligibilityを一つのhash chainで結ぶ。この列挙がReceiptのclosed field setであり、生成projectionはfieldを追加せずschema versionを上げる。Artifact hashの一致比較は[Core architecture](../02-foundation/core-architecture.md) §6の決定性同値クラスに従い、同一Host識別のReceipt間だけで行う。異なるHost間の一致は同節のfixture検証済みArtifact種別だけに要求する。Receipt不在、hash不一致、未承認Loss、Development-only Tool混入のArtifactをPackage候補にしない。

## 4. Reimportと依存invalidation

Reimportは既存Asset IDとProfileを維持して新しいAnalysisとPreviewを作る。Source変更だけで自動promotionしてはならない。次を破壊的Conflictとして扱う。

- Hierarchy、Skeleton、Material slot、Animation clip、Texture／Audio channel、Font coverageの意味変更。
- Importer lock、Profile schema、Source converter、Target codecの変更。
- 新しいwarning／loss、budget超過、dependency削除。
- Scene、Material、Cue、UI等のconsumerへ生じる意味Diff。

```text
AssetReimportConflictV1
  conflict_id: StableId
  asset_id: StableId
  conflict_kind: hierarchy | skeleton | material_slot | animation_clip
               | texture_channel | audio_layout | font_coverage
               | importer_version | profile_schema | dependency
  source_path: optional StableSourcePathV1
  before_value: TypedConflictValueV1
  after_value: TypedConflictValueV1
  consumers: AssetConsumerImpactV1[0..4096]
  severity: warning | destructive | blocking
  allowed_resolutions: ClosedConflictResolutionId[1..8]
```

`TypedConflictValueV1`は`conflict_kind`と同じtagを持つclosed unionである。各payloadは順に`HierarchyConflictValueV1`、`SkeletonConflictValueV1`、`MaterialSlotConflictValueV1`、`AnimationClipConflictValueV1`、`TextureChannelConflictValueV1`、`AudioLayoutConflictValueV1`、`FontCoverageConflictValueV1`、`ImporterVersionConflictValueV1`、`ProfileSchemaConflictValueV1`、`DependencyConflictValueV1`とする。before／afterは同じvariantでなければならず、任意JSONや名前一致による自動再接続を禁止する。Blocking conflictはSource修正、Profile変更、明示Migration ChangeSetのいずれかと新しいPreview Receiptが揃うまで解決済みにしない。

invalidation keyはSource hash、flattened Profile hash、Importer／Toolchain lock、dependency Artifact key、Target Profileを含む。いずれかが変われば影響nodeだけを再Cookする。Hard dependency cycle、missing、Target不一致をrejectし、Build-only dependencyをRuntime Catalogへ含めない。Optional dependencyは意味同等のfallback Asset IDを明示する。

Reimport、bulk migration、CookのCancel後にpartial outputをArtifact storeへpublishしない。失敗、disk full、worker crash、stale resultでは旧generationとProject revisionを維持する。

## 5. Derived／Cooked Artifact

`ArtifactSubjectRefV1`はgeneric artifactのsubjectを閉じる。

```text
ArtifactSubjectRefV1
  kind: asset | world_root | world_section
  payload:
    asset:
      asset_id: StableId
      asset_revision: positive uint64
    world_root:
      world_root_id: StableId
      source_project_revision_ref: ProjectRevisionRefV1
    world_section:
      world_root_id: StableId
      world_section_id: StableId
      section_revision: positive uint64

DerivedArtifactManifestV1
  artifact_key: SHA-256
  artifact_subject_ref: ArtifactSubjectRefV1
  artifact_role_id: ClosedArtifactRoleId
  target_profile_ref: TargetProfileRefV1
  schema_version: positive uint32
  payload_hash: SHA-256
  payload_size: uint64
  alignment: positive uint32
  dependency_keys[0..4096]: SHA-256
  producer_id: ClosedArtifactProducerId
  producer_version_hash: SHA-256
  toolchain_lock_hash: SHA-256
  capability_requirements[0..256]: ClosedCapabilityId
  runtime_budget_ref: RuntimeAssetBudgetRefV1
  manifest_hash: SHA-256

RuntimeAssetBudgetRefV1
  asset_budget_class_id: AssetBudgetClassId
  target_profile_ref: TargetProfileRefV1
  project_scale_envelope_ref: ProjectScaleEnvelopeRefV2
  budget_ref_hash: SHA-256
```

`kind = asset`だけがAsset import identityを使う。`world_root`と`world_section`のpayload内容、entity record、section dependency、loader semanticsは[Runtime Package](../04-runtime/runtime-package.md)が定め、Asset Lifecycleはsubject identity、artifact key、dependency closure、Cook、immutability、promotion envelopeだけを所有する。World Artifactへsynthetic Asset ID、asset-only revision、`asset://` URIを与えない。

Source拡張子の検出、decoderの存在、Preview成功はProduct採用またはArtifact publicationを意味しない。immutable Derived Artifactはexact Source revision／content hash、flattened Import Profile、Importer／Toolchain lock、dependency closure、Target Profile、Contract setから決定論的に生成し、全refとpayload hashが一致する場合だけpublishする。current Product採用済み名前付きformat集合が`[]`の場合、検出済みSourceを未登録formatへ補完せず、SourceとProject revisionを不変にして必要なProduct／Owner closureを返す。

Artifact keyは完成manifestとpayloadのcanonical encodingから作る。Payload headerはsubject、role、Target、schema、sizeを持ち、unknown major、truncation、trailing bytes、hash mismatchを拒否する。Artifact storeはcontent-addressedかつimmutableであり、成功Artifactを上書きしない。

Hard dependency closureが同一generationでReadyになるまでArtifactをpromotionできない。Geometry、Material、Skeleton、Animation、Physics、Navigation等のDomain artifactは各DomainのSource意味とvalidationを消費する。Backend native pointer、device固有command、driver依存objectをArtifact／Packageへ保存しない。

Garbage collectionはProject revision、Catalog、Package manifest、last-valid generation、active lease、Recovery snapshotからreachabilityを計算した後だけ実行する。SourceからCooked Artifactを逆生成せず、corrupt Cacheはhashで拒否してSourceから再Cookする。

## 6. Packagingとcontent addressing

### 6.1 CatalogとVFS

`MirakanArtifactCatalogV1`はAsset／World Artifact共通のCatalogであり、次を正本field setとする。

| Field | 型／規則 |
|---|---|
| `catalog_id`／`catalog_version` | UUIDv7／`uint64` |
| `target_profile_hash` | `sha256`。Packageと一致 |
| `package_set_hash` | `sha256`。mount closure |
| `entries` | `ArtifactCatalogEntryV1[]`。`{artifact_subject_ref canonical bytes, artifact_role_id}`順 |
| `dependencies` | `ArtifactCatalogDependencyV1[]`。Artifact key順 |
| `content_groups` | `ContentGroupV1[]`。base、optional、level、DLC等 |
| `capability_requirements` | `ClosedCapabilityId[]`。Target起動前検査 |
| `root_hash` | canonical catalog SHA-256 |

asset subjectのRuntime参照は`asset://<uuid>/<role>`またはtyped `AssetHandle`を使い、OS pathをGameplayDefinition、Save、Network payloadへ保存しない。World Root／Sectionはtyped `ArtifactRefV1`と`ArtifactSubjectRefV1`で参照し、asset URIまたはAssetHandleへ変換しない。

VFSはContent Packageのread-only mountであり、Save、setting、screenshot、crash dumpを保持しない。高priority Catalogによる置換は同じartifact subject＋role、schema compatibility、Capability、dependency closure、Package integrityがすべて合格する場合だけ許可する。Path一致で別Artifactへ置換しない。

### 6.2 Asset Content Package

`Mirakan Asset Content Package V1`はAsset／shared dependency artifactのHeader、bounded Payload Block、Indexを持つ。Headerはmagic、format major／minor、Package ID／revision、Target Profile hash、block size、Index位置、Catalog hash、Package root hashを持つ。C1 blockは64 KiB、最終blockだけ短くできる。Artifactをblock boundaryへ配置し、Indexがrange、logical size、payload hash、dependency、Catalog位置を持つ。World Root／Section imageとRuntime Package binaryは[Runtime Package](../04-runtime/runtime-package.md)が別に所有する。

Package root hashはhash fieldをzero化したHeader、block hash table、canonical Indexから作る。Mount前にbounds、overlap、duplicate、integer overflow、path、dependencyを検証する。Package全体のmemory mapを前提にせず、bounded range readを可能にする。

### 6.3 Package assembly

Package assemblyはImportの後段であり、この文書が唯一所有する。

1. Commit済みProject revisionとTarget Profileを固定する。
2. `ProjectManifest.runtime_entry_point_refs`からTarget別に選択したexact Runtime Entryを解決し、その`world_ref`／`ui_document_ref`／`startup_game_system_refs`のtransitive Source closure、always-loaded resource、Content GroupからAsset rootを列挙する。[Project State](project-state.md) §3.1.1.1のatomic activation後にworld entryがexact `RuntimeEntryPresentationBindingV1`を持つ場合だけ、その`root_ui_document_ref`とtransitive UI dependency closureも同じreachability rootへ追加する。current Binding集合はexact `[]`であり、V1 `world.ui_document_ref=null`を緩和、UiDocument名／HUD roleからBindingを推測、Bindingだけを先行使用しない。単数Root Scene、表示名、path、`latest`をcurrent reachability rootにしない。
3. Hard dependency closureを解決し、missing、cycle、Target不一致を拒否する。
4. Artifact hashとLicense／Provenance／Safety Receiptを照合する。
5. Content Groupごとにcanonical artifact subject／role順で配置する。asset subject以外のpayload layoutをここで再定義しない。
6. Catalog、Index、SBOM、Notice、Build Receiptを生成する。
7. clean assemblyを再実行してroot hash一致を検証する。
8. Platform application packageとPlatform validationへ渡す。

Development loose layoutとShipping Content Packageは同じCatalog conformanceを通す。Content PatchはArtifact／block単位で作り、Baseを上書きせず、新Package全体を検証してからmount setを切り替える。Binary executable、native library、Shader binaryをContent Patchで配布しない。DLCはContent Group、entitlement、Base範囲、dependency、Save behaviorを明示し、不在contentを別Assetへ黙って置換しない。

## 7. Hot reload／promotion

Source変更は新Asset revisionとImport Jobを作る。依存closure全体をStagingし、次が揃った場合だけ新generationをpromotion候補にする。

- 全Artifactが同じTarget、Toolchain lock、Contract setでReady。
- Interface／schema compatibilityとDomain invariantが合格。
- 必要resource reservationが成立。
- Import Receipt、Conversion Report、Approvalが同じhash chainを参照。
- Runtime ownerが定める安全なactivation boundaryへ到達。

一つでも不合格なら旧generation全体を維持する。Textureだけ新、Materialは旧のようなdependency closureの部分混在を禁止する。互換でない変更は`restart_play`を返す。Asset lifecycleはStaging closureとpromotion eligibilityを所有し、Runtime phase、lease、queue、retire時刻はRuntime Ownerが所有する。

Active generationより古い結果、cancel済みJob、owner generation不一致の結果をpublishしない。last-valid generationと必要なretiring generationを保持し、fault時にProject sourceまたはPackage setを破壊しない。

## 8. EditorとAI planned actions

Asset BrowserはStable IDをselection modelとし、logical directory、kind、semantic role、tag、license、readiness、Diagnosticでfilterする。Source、Import revision、Active generation、Target residency、dependency／reverse dependencyを表示する。表示row、thumbnail object、screen coordinateをOperation targetにしない。

Import Inspectorは`Source`、`Analysis`、`Profile`、`Preview`、`Conversion`、`Dependencies`、`Diagnostics`、`History`を持つ。Basic／Advanced viewは同じImport Documentのprojectionであり、別設定を持たない。Import、Preview、Cook、Reimport、bulk migrationをcancel可能なJobとして表示し、stage、progress、current asset、resource limit、Diagnostic countを示す。

Catalog、Source analysis、flattened Profile、Conversion Report、dependency closure、Reimport Conflictのreadと、Profile／設定変更／Preview／Reimport／bounded bulk migration／placeholder置換／LOD source bindingのproposalは、Stable IDでないplanned semantic action vocabularyである。Asset ownerのcurrent MCD／Owner Manifest／Service allowlist／Policy／Validator／Diagnostic／Receipt／Provider／MCP／alias Operation集合はすべて`[]`、Capability stateは`not_activated`である。future work item `activation.asset.authoring_operations.v1`が採用するexact ID集合と完全closureを一transactionで登録するまでdispatchせず、action名からIDを生成しない。Activation後のread actionだけがtyped result、proposal actionだけが[ProjectChangeSetV1](project-state.md#5-projectchangesetv1)候補を返し、直接Commitしない。

AIはAsset ID、Profile ID、Source pathを推測生成しない。未Activated format、未対応Target codec、Catalogにない選択肢には`CapabilityNotActivated`を返す。提案はProfile選択、evidence、変換、保持値、visual／behavior／memory／Package impact、未解決質問、必要Approval、rollbackを含み、自然言語だけでなくPlanとDiagnostic IDを正規出力にする。

外部またはAI生成Assetも通常Importを迂回しない。Asset固有Provenance recordのcontentは本節だけが所有し、共通Field `origin_uri_or_generation_operation_ref`、`acquired_or_generated_at`、`content_hash`、`creator_or_provider`、`terms_snapshot_ref`、`license`、`rights_confirmation_status`、`commercial_review`、`safety`、`content_credential_ref`、`modification_chain[]`を必須にする。originがgeneration operationの場合は`tool_model_lock_ref`、`request_hash`、`input_asset_refs[]`に加え、採用versionのlockとは別に実行証拠の`generation_tool_receipt_ref`を必須とし、外部取得またはUser提供Sourceではこれらgeneration専用Fieldを持たない。Provider outputを直接ProjectまたはCacheへ入れずStagingに置き、このRecordと共通Evidence envelopeを結ぶ。AIが品質または権利を自己承認できない。

[Product Plan](../00-product/product-plan.md)のMVP-A／MVP-Bでは`source_kind=domain_pack_reference | user_provided`だけを許可する。Asset生成ProviderはC2未満のPhase exit前提にせず、`external_generated`が明示要求されたのにqualified Provider、権利Record、safety Receiptのいずれかがない場合は`diagnostic.asset.external-generation-unavailable`を返してProposalを拒否する。Pack reference、user-provided placeholder、別Providerへ暗黙fallbackせず、元の要求とProject sourceを変更しない。

## 9. Platform specialization

Source／Import contract、Artifact identity、Catalog conformance、Package assembly順はTarget間で共通とする。Target差はProfileとDerived payloadへ閉じる。

- Texture／Audio／Mesh／Shader等はTarget Profileが選ぶcodec／layoutへCookする。
- Desktop、Mobile、Store配布の絶対budgetやpackage wrapperをこの文書で複写しない。
- Windows Development、Shipping、Mobileは同じArtifact／Catalog hash chainを維持する。
- Store delivery、Platform署名、install／entitlement、Application updateはPlatform Ownerへ渡す。
- Device／driver固有cacheは同一device identity、driver、API、build flag、geometry keyに限定し、Content Package、Patch、DLCへ配布しない。

Streaming requestのpriority、deadline、lease、queue、Domain activationはRuntime Ownerが決める。Asset lifecycleはPackage range、Artifact dependency、same-generation fallback、integrity metadataを提供する。Targetに意味同等fallbackがなければunsupportedとし、Source意味を黙って削らない。

## 10. Diagnostics

```text
AssetDiagnosticV1
  diagnostic_id: ClosedDiagnosticId
  severity: info | warning | error | fatal
  asset_id: StableId
  source_path: optional StableSourcePathV1
  field_path: optional McdFieldPath
  evidence: DiagnosticEvidenceV1[1..16]
  message_key: LocalizationKey
  remediation_ids: ClosedRemediationId[0..8]
  blocks_preview: bool
  blocks_cook: bool
  blocks_promotion: bool
```

Free-form messageだけをAI判断へ使わない。

| Diagnostic | 結果 |
|---|---|
| `UnsupportedSourceFormat` | Analysis拒否。許可形式を提示 |
| `CapabilityNotActivated` | Adapterを起動せずActivation条件を返す |
| `UnsupportedSourceFeature` | Preview／Cook停止。承認済みfallbackだけ別Plan化 |
| `AmbiguousSemanticRole` | `NeedsProfile`。AIが確定しない |
| `CoordinateIntentUnknown` | Sourceを保持し、意味変更をblock |
| `SkeletonBindingMismatch` | closure promotion拒否 |
| `ColorEncodingConflict` | Texture Cook拒否 |
| `AudioChannelLayoutUnknown` | Audio Cook拒否 |
| `FontCoverageMissing` | required localeのPackage assembly拒否 |
| `ImporterVersionConflict` | 旧generationを維持しMigration Preview要求 |
| `DeterminismMismatch` | 対象Importer lockをProduction禁止 |
| `ReimportConsumerConflict` | 自動promotion禁止 |
| `DependencyMissingOrCycle` | closure全体拒否 |
| `PackageIntegrityMismatch` | mount拒否 |
| `PromotionRejected` | old generation全体維持 |

Placeholderを許可するroleはProfileで明示する。Collision、Navigation、required UI Font、GameplayDefinition等のauthoritative／required Assetへ汎用placeholderを使わない。Diagnosticを理由に別Assetへ暗黙mappingしない。

## 11. Security

ImporterとSource converterはEditor／GameHostと別のsandbox workerで実行する。

- network deny、shell／plugin discovery／environment expansion禁止。
- Brokerが開いたbounded Source／dependency handleだけをread可能。
- Job固有temporary directoryだけをwrite可能。
- CPU、wall time、commit memory、output bytes、file count、archive depth／ratioをhard cap。
- child processはOrchestratorが別sandbox envelopeとして明示起動。
- untrusted filenameをcommand lineへ連結しない。
- crash、timeout、OOM、sanitizer failureをJob failureとして隔離。

Output manifest、全file hash、schema、semantic、size、dependency、determinismが合格した後だけArtifact storeへ原子的にpublishする。clean worker二回のIR、Report、Artifact hashが一致しなければProductionへ昇格しない。GPU driver依存のSource decode、Preview判定、Production Cookを禁止する。

Tool／SDK／Libraryのexact pin、hash、license、取得元、更新Gateは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)を参照し、本書へ複写しない。Asset ReceiptはそのToolchain lock hashだけを記録する。AI authorization、Credential、Approvalは[AI Security／Approval](../01-governance/ai-security-approval.md)、Evidence envelopeとProvenance gradingは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を参照する。

Runtimeによる任意Source Import、remote URL、native code／Shader／binary生成、Providerからの直接download、未承認model weight、自己更新ContentはShippingで禁止する。

## 12. Qualification

Asset lifecycleは次のGateをすべて満たすまで対象CapabilityをActivationしない。

### 12.1 Contractとdeterminism

- MCD type、Profile、Plan、Activation時に採用するexact Operation candidate、Diagnostic、stateがC++、Editor、AI Tool、CLIへ同じ正本から生成される。
- 各生成projectionはAsset contract ID、schema version、canonical content hashを記録し、本書の全named contractについてfield名、型、cardinality、closed tagが完全一致する。schema ownership検索で定義文書が本書一つだけであることを検証する。
- valid／invalid／boundary、truncation、overflow、NaN／Inf、cycle、duplicate、unknown feature fixture。
- clean二回Import／Cook／Package assemblyでIR、Report、Artifact、Package root hashが一致する。
- Source、Profile、Importer lock、Toolchain lock、dependency、Target変更を正確にinvalidateする。
- AI、手動Editor、headless CLIが同じPlan hash、Diagnostic、Artifactへ収束する。

### 12.2 AssetとReimport

- Scene座標、unit、pivot、Hierarchy、Skeleton／Animationのbefore／after意味を確認できる。
- Texture／Spriteのcolor、alpha、normal、mip、atlas、Target compressed結果を確認できる。
- Audioのchannel、loop、loudness、Target codec、Fontのcoverage、embedding、raster差を確認できる。
- Reimportは非破壊設定を保持し、破壊的変更をconsumer closure付きConflictとしてblockする。
- Cancel、timeout、worker crash、OOM、disk fullでProject revisionと旧generationを維持する。

### 12.3 Package、promotion、security

- Catalog missing／cycle／duplicate／Target mismatchとPackage bounds／hash／overflow fuzz。
- Cache削除後のfull rebuildで同じPackage root hashを得る。
- same-generation dependency closureのatomic promotionと部分Version混在拒否。
- Patch中断、disk full、integrity failureで旧mount setを維持する。
- License、SBOM、Notice、Provenance、Safety ReceiptがPackage closureへ揃う。
- malformed parser、decompression bomb、path traversal、fuzz、sandbox escape negative fixture。
- Shipping packageへSource、Development Tool、Credential、未承認native payloadが混入しない。

Format／Target／Providerの段階導入順とCapability maturityは[Product Plan](../00-product/product-plan.md)が所有する。各Capabilityはこの共通Gateに加え、対象formatの公式spec／validator／fixture、Target cook、memory、loss、Reimport migrationを満たす。外部versionやreleaseを本書に固定せず、検証時のexact lockは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)とReceiptから解決する。

明示的に採用しないものは、Runtime arbitrary import、Source fileをRuntime source of truthにする方式、untyped Import property bag、名前一致Reimport、Importer native objectの公開、SourceとCooked Artifactの混在、Package assemblyの別正本、部分generation promotion、汎用placeholderによるProduction成功、AIによる権利／品質の自己承認である。
