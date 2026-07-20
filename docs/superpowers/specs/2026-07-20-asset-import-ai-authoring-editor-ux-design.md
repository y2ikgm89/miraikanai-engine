# Miraikanai Engine Asset Import／AI Authoring／Editor UXアーキテクチャ規約

- 文書版: 1.0
- 作成日: 2026-07-20
- 対象: Texture、Sprite、3D Scene／Mesh、Skeleton／Animation、Audio、Font、Import Profile、Conversion Report、Preview、Reimport、Asset Browser、AI Operation
- 状態: プロジェクト公式の規範設計レビュー版
- Asset基盤正本: [Miraikanai Engine Asset Pipeline／Content Package規約](./2026-07-19-asset-pipeline-content-packaging-design.md)
- 2D／3D正本: [Miraikanai Engine 2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)
- Audio正本: [Miraikanai Engine Audio／Mixer／Spatial規約](./2026-07-19-audio-mixer-spatial-architecture-design.md)
- Animation正本: [Miraikanai Engine Physics／Navigation／Animation連携規約](./2026-07-19-physics-navigation-animation-architecture-design.md)
- Editor正本: [Miraikanai Engine Editor／Workspace／UX規約](./2026-07-19-editor-workspace-ux-design.md)
- Authoring正本: [Miraikanai Engine Authoring Model／Project State規約](./2026-07-19-authoring-model-project-state-design.md)
- Schema正本: [Miraikanai Engine 実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)

## 1. 結論

Miraikanai EngineのAsset Importは、Source形式ごとの任意処理ではなく、**Source解析、型付きImport Plan、共通Import IR、意味Validation、Preview、Cook、Receiptを分離した一つの閉じたPlatform**として実装する。

有名Engineの実用上の長所であるImport preset、軸／単位補正、per-object preview、reimport設定保持、conflict表示、scriptable pipelineを採用する。一方、Project scriptやAIがImporter内部objectを直接変更する方式は採らず、MCDから生成した型付きProfile、Operation、Diagnosticだけを公開する。

3D Sourceの成熟度を次に固定する。

| Capability | 段階 | Production方針 |
|---|---|---|
| glTF 2.0／GLB | C1 | 正規3D interchange。Khronos仕様、Validator、Asset Generator、Sample Assetsをconformance基準にする |
| `.blend` | C2 | `.blend`を解析せず、承認済み公式Blenderを隔離Source Conversion Workerとして実行し、固定設定でglTF／GLBへ変換する |
| FBX | C2 | `ufbx`を第一候補とする隔離Adapter。Dependency ADR、version／commit lock、license、fuzz、loss fixture合格前はCapabilityを公開しない |
| USD／USDZ | C3 | DCC collaboration／large scene assembly用。OpenUSD Stageをbounded resolveし、選択rootを共通IRへ変換する。Project／Runtimeの正本にしない |

形式が増えてもRuntime Asset、Scene schema、AI Operationを分岐させない。全経路を同じ`AssetImportPlanV1`、種別別IR、`AssetConversionReportV1`、Validator、Cookへ収束させる。

## 2. 決定権と非対象

| 主題 | 正本 |
|---|---|
| Job identity、Worker隔離、Derived Artifact、Catalog、Package、Streaming、Hot Reload | Asset基盤規約 |
| Import Profile、Source解析、種別別IR、Conversion／Loss Report、Preview、Reimport、AI／Editor操作 | 本書 |
| World座標、Material／Texture意味、Mesh、Skeleton／Animation製品Capability | 2D／3D機能計画とAnimation正本 |
| Audio Clip、loudness、loop、streaming意味 | Audio正本 |
| Panel、Workspace、Diff、History、Accessibility | Editor正本 |
| ProjectRevision、ChangeSet、Undo、Recovery、Stable ID | Authoring正本 |

次は本書のC1／C2対象外である。

- Runtime arbitrary import、Shipping Runtimeの外部tool起動、remote URL参照
- Source fileをRuntime source of truthまたはSave形式にすること
- FBX、Blend、USDの意味を完全losslessと宣言すること
- DCC pluginの自動install、任意Python startup、環境依存plugin discovery
- AIによるSource bytes、Cooked payload、Importer native object、OS pathへの直接write
- Sourceの意図を推測した無承認のorigin移動、grounding、front変更、root削除
- Import failureをmagenta、silence、別Animation等の汎用placeholderでProduction成功へ変えること

## 3. 規範資料と採用原則

### 3.1 優先順位

1. 形式を所有する標準化団体または公式Projectの規範仕様
2. 公式Validator、reference implementation、conformance fixture
3. DCC／OS／Platformの公式Export／Import資料
4. Unity、Unreal Engine、Godotの公式資料はUX、運用、failure事例の比較

外部Engineの型名、file配置、Inspector layout、既定値をMiraikanaiの正規契約へコピーしない。

### 3.2 Source形式の基準

| 種別 | C1 Source | C2／C3 | 規範基準 |
|---|---|---|---|
| Texture／Sprite | PNG、JPEG、OpenEXR、KTX2、DDS | 追加形式は個別ADR | PNG Third Edition、JPEG 1、OpenEXR、KTX 2.0、Microsoft DDS／DXGI |
| 3D Scene／Mesh | glTF 2.0／GLB | Blend／FBX C2、USD／USDZ C3 | Khronos glTF、Blender、`ufbx`、OpenUSD |
| Skeleton／Animation | glTF 2.0 | Blend／FBX C2、USD C3 | glTF animation／skin、ozz offline contract |
| Audio | WAV PCM／IEEE float、native FLAC | Ogg OpusはSource追加候補、Cooked streamはOpus | WAVEFORMATEXTENSIBLE、RFC 9639、RFC 7845 |
| Font | OpenType OTF／TTF | Variable／Color featureはCapability別 | OpenType 1.9.1 |
| Material／Style | MCD＋glTF material subset | MaterialX／USD Shadeは別Capability | MCD、glTF extension allowlist |

APNG animation、OpenEXR deep／multipart、JPEG 1の未Activated coding process、DDS unknown DXGI format、WAV compressed codec、OpenType embedded SVG等は、C1で黙って縮退させない。JPEG baseline／progressive DCTはC1 decoder fixture対象とし、それ以外はProfileが明示対応していなければ`UnsupportedSourceFeature`で停止する。

## 4. 共通Import architecture

```text
Source Asset
  -> Source Broker
  -> Format Adapter／Source Conversion Worker
  -> AssetSourceAnalysisV1
  -> AssetImportPlanV1
  -> TextureImportIRV1 | SceneImportIRV1 | AudioImportIRV1
     | FontImportIRV1
  -> Structural Validator
  -> Semantic／Budget Validator
  -> Preview Artifact＋AssetConversionReportV1
  -> Human／authorized AI ChangeSet approval
  -> Deterministic Cook
  -> Derived Artifact closure
  -> AssetImportReceiptV1
  -> Atomic promotion
```

Source解析はProjectを変更しない。Import設定変更は`AssetMetadataDocument`内のProfile参照と型付きsettingsを対象とするChangeSetであり、承認後に新しいAsset revisionとJobを生成する。Previewは正式Artifactと同じIR、Validator、Cook codeを使用し、Preview専用の黙った補正を禁止する。

### 4.1 状態遷移

```text
Discovered
  -> Analyzing
  -> NeedsProfile | AnalysisRejected
  -> Planned
  -> Previewing
  -> AwaitingApproval | PreviewRejected
  -> Cooking
  -> Validating
  -> ReadyToPromote | CookRejected
  -> Active
```

- `NeedsProfile`はSource意味またはHigh Impact設定が一意に決まらない状態である。
- `AwaitingApproval`から`Cooking`へ進めるのは、Risk classに必要な承認とbase ProjectRevisionが一致するときだけである。
- Cancel、timeout、Worker crash、stale revisionはProject状態を変更せず、旧Active generationを維持する。
- 同じJob key、同じ承認Planから異なるIRまたはArtifactが出たImporterをProduction禁止にする。

## 5. 共通MCD contract

### 5.1 `AssetSourceAnalysisV1`

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

Sourceの名前、folder、DCC名だけからsemantic roleを確定しない。ファイルが持つ規範metadata、Project Profile、明示User設定を根拠にする。

### 5.2 `AssetImportProfileV1`

```text
AssetImportProfileV1
  profile_id: StableId
  schema_version: SemVer
  asset_kind: texture | sprite | scene_3d | mesh | skeleton
            | animation | audio | font
  target_profile_scope: StableId[1..16]
  common: CommonImportSettingsV1
  settings: tagged union by asset_kind
  fallback_policy: forbid | allow_listed
  approval_policy: RiskApprovalPolicyV1
```

`CommonImportSettingsV1`はdependency policy、unknown metadata policy、warning promotion policy、budget classだけを持つ。Asset種別固有fieldをuntyped property bagへ入れない。Profile inheritanceは一段だけ許可し、最終的なflatten済みProfile hashをJob keyとReceiptへ保存する。

### 5.3 `AssetImportPlanV1`

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
  risk_class: R0 | R1 | R2 | R3
```

`blocking_questions`が1件以上あるPlanはCook承認対象にできない。AIが未回答項目を既定値で埋めることを禁止する。低影響で公式Profileが一意な項目だけEngineが解決し、その由来を`explicit_conversions`へ記録する。

### 5.4 `AssetConversionReportV1`

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

`LossRecordV1`はSource path、feature ID、理由、visual／behavior impact、承認可否を持つ。未対応featureを`dropped_features`へ記録するだけで成功させず、`fallback_policy=allow_listed`かつ対象featureの承認済み変換がある場合だけCook候補にする。

### 5.5 `AssetDiagnosticV1`

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

Free-form messageだけをAI判断の入力にしない。`diagnostic_id`、evidence、remediation、block phaseをAI ToolとEditorの共通契約にする。

### 5.6 `AssetImportReceiptV1`

ReceiptはSource／Profile／Plan／IR／Artifact／Preview／Approval／Toolchain／Validatorのhash chain、Target、実行budget、全Diagnostic count、Package eligibilityを持つ。Production packageはReceipt不在、hash不一致、未承認Loss、Development-only toolの混入を拒否する。

### 5.7 Asset種別IR

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
  sprite_records: SpriteRecordIRV1[0..65536]

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

Standalone Skeleton／Animationも`SceneImportIRV1.skins`と`SceneImportIRV1.animations`と同じ`SkinIRV1`／`AnimationIRV1`を使う。Source形式ごとのnative object、decoder pointer、DCC property bagをIRへ保存しない。

### 5.8 `AssetReimportConflictV1`

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

`TypedConflictValueV1`はkindごとのtagged unionであり、任意JSONを持たない。Blocking conflictはSource修正、Profile変更、明示Migration ChangeSetのいずれかと、新しいPreview Receiptが揃うまで解決済みにしない。

## 6. 3D Scene／Mesh Import

### 6.1 Engine canonical space

- 右手系、+Y up、+Z object forward、meter、radian
- column vector、column-major storage
- local transform合成は`T * R * S`
- Quaternionは`x,y,z,w`、finite、normalized、canonical sign

glTF 2.0は右手系、+Y up、+Z forward、meter、radianでEngine canonical spaceと一致する。C1 glTF Importerは軸または単位を無理由に変換しない。

### 6.2 `SceneImportIRV1`

```text
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
```

各Nodeはstable source path、parent、local transform、object binding、visibility、semantic extrasを持つ。Source配列indexまたは重複nameを永続IDにせず、Importerがcanonical hierarchy pathとSource object identityからJob内stable IDを作る。

### 6.3 Transform policy

| Policy | C1既定 | 規則 |
|---|---|---|
| `hierarchy_policy` | `preserve` | 空Rootを性能目的で削除しない。削除はAnimation／reference整合を変えるため明示変換 |
| `root_transform_policy` | `preserve` | Source rootの位置・回転・scaleを保持 |
| `pivot_policy` | `preserve` | Source pivot／node originを保持 |
| `placement_policy` | `preserve_source` | bounds center移動、grounding、原点移動を自動実行しない |
| `front_policy` | `preserve_source` | glTF +Zを保持。意味上の前方向が異なる場合はmetadataとPreviewで解決 |
| `unit_policy` | `convert_to_meters` | Source metadataに基づく明示変換だけを適用 |
| `matrix_policy` | `require_decomposable_trs` | finiteで安定にTRS分解できないshear／singular matrixをC1で拒否 |

Static Meshだけは明示`bake_geometry`により、unit、non-uniform scale、negative determinantをvertexへCookできる。変換後にwinding、normal、tangent、bounds、collision／nav sourceを再計算する。Skinned Meshのnegative determinant、shear、singular bind poseはC1で拒否し、無理にbakeしない。

Import Previewは必ず次を表示する。

- Source軸とEngine軸、+X／+Y／+Z gizmo
- Root／Pivot／bounds center／ground plane
- Source local transformとEngine local transform
- negative determinant、non-uniform scale、shear、単位倍率
- Skeleton bind pose、Animation root track、Collider／Nav生成予定
- 変換前後のhierarchy Diff

### 6.4 Format Adapter

#### glTF／GLB C1

- Khronos glTF 2.0.1を規範とし、extension ID allowlistをProfile versionへ固定する。
- 公式glTF Validatorを独立oracleとしてCIで実行し、Engine Validatorとの重大error差を0件にする。
- Khronos glTF Asset GeneratorとSample Assetsをfixture manifestへhash固定する。
- external URIはSource BrokerがProject内dependencyへ解決し、absolute path、network、data size超過を拒否する。
- Node matrixとTRSの同時指定、cycle、accessor bounds、sparse accessor、NaN／Inf、不正Quaternionを拒否する。

#### Blend C2

- `.blend` parserをEngineへ実装しない。
- Brokerが承認済み公式Blender executable、export script、startup configをexact version／hashで起動する。
- Blenderはnetworkなし、factory startup、user script／addon無効、Job専用read handle／temporary writeだけで実行する。
- 出力はGLB、export log、Blender version、export settings hash、Source dependency manifestである。
- Blenderの任意addon、driver、external scriptが必要なSourceは`UnsupportedBlendDependency`で停止する。
- Source Conversion WorkerはImporterのchild processではなく、Job Orchestratorが別sandbox envelopeとして起動する。

#### FBX C2

- 第一候補はMIT選択の`ufbx`とし、C2開始時のDependency ADRでtag／commit、source hash、compiler、license textを固定する。
- `master`やfloating releaseをProduction toolchainへ入れない。
- binary／ASCII version、axis、unit、inheritance mode、geometric pivot、pre／post rotation、bind pose、animation layer、material mappingをConversion Reportへ記録する。
- Importer変更はhierarchy、rest pose、material orderを変え得るため、既存Assetの自動切替を禁止する。旧Importer generationを保持し、Reimport Conflict Previewと明示Migration ChangeSetを必須とする。
- Autodesk FBX SDKは再配布、license、sandbox、Target tool availabilityの別ADRなしに追加しない。

#### USD／USDZ C3

- OpenUSD StageはDCC collaboration入力であり、Project Revision、SceneDocument、Asset identityを置換しない。
- allowed schema、resolver、plugin、layer、payload、prim、variant、time sample、dependency byteにhard capを設ける。
- network resolver、arbitrary plugin discovery、Python plugin、unresolved asset pathを禁止する。
- selected rootのcompositionを決定論的にresolveし、layer／reference／payload／variant由来をLoss Reportへ保存する。
- flatten結果だけを保存してSource composition由来を失わず、元Stage hashとselectionをReceiptへ保持する。
- USD Shade、Physics、Skeleton等は個別Capability allowlistにない限りrejectし、名前一致でMiraikanai Componentへ推測変換しない。

## 7. Texture／Sprite Import

### 7.1 `TextureImportSettingsV1`

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

AIはfile名の`_n`、`normal`等だけでnormal mapを確定しない。名前はadvisory evidenceであり、Project naming policy、画像統計、Material slot、User確認を組み合わせる。Normal、data、maskをsRGBとしてCookすることをhard errorにする。

### 7.2 形式別規則

- PNGはW3C PNG Third Editionのchunk bounds、CRC、color space優先順位、alpha、APNGを検証する。C1のTexture／Sprite ProfileはAPNGを拒否する。
- JPEGはorientation metadataをPreviewへ反映し、適用結果をReportへ記録する。Alphaを要求するroleでは拒否する。
- OpenEXR C1はfiniteなscanline／tiled single-part、HALF／FLOATの承認channelだけを受理する。Deep、multipart、unknown channel semanticsを拒否する。
- KTX2はlevel／face／layer／DFD／supercompression boundsを検証し、KTX Softwareの`ktx2check`を独立oracleにする。BasisLZ／UASTCのSource color／alpha metadataを保持する。
- DDSはmagic、header、DX10 header、dimension、mip、array、cube、block alignment、payload sizeを検証する。承認DXGI format以外を拒否する。

### 7.3 PreviewとCook

Previewはsource／scene-linear／target compressedの三表示、alpha checker、channel solo、normal sphere、mip、sprite rect／pivot／PPU、estimated GPU bytesを持つ。Target compressed previewがない状態でProduction compression Profileを承認できない。

CookはSource bytesをRuntimeへ持ち込まず、Target別BCn／ASTC／ETC2 Artifactを決定論的に生成する。Codec／encoder version、quality、thread count、RDO parameter、Target formatをArtifact keyへ含める。

## 8. Skeleton／Animation Import

### 8.1 `AnimationImportSettingsV1`

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

C1は同一Skeletonのexact stable joint path、parent、bind poseによるClip bindingだけをProduction対応する。Humanoid retargetはC2であり、semantic bone mapping、reference pose、scale compensation、twist policy、before／after pose corpusを持つProfileだけを許可する。

### 8.2 検証

- joint path一意、parent cycleなし、bind matrix finite／invertible
- weight finite／非負、influence cap、正規化、joint index範囲
- key time strict order、duration範囲、value finite
- Quaternion normalized、shortest-path continuity、scale非zero
- Clip range、loop seam、root delta、event order
- Skeleton／Clip generationとProfile hash一致
- Source hierarchy変換後もskin bind equationとreference poseが一致

Animation optimizerは既定で有効にできるが、元Clipとの最大translation、rotation、scale errorをProfile閾値で測定する。閾値超過時は圧縮率を下げて再試行し、hard budget内で成立しなければCookを拒否する。

### 8.3 Preview

Skeleton tree、joint axis、bind／reference pose、skin weight heatmap、clip scrub、root trajectory、loop seam、retarget before／after、compression errorを同じPreview Sceneで表示する。Editor scrubはGameplay／Audio／VFX Eventを発火しない。

## 9. Audio Import

### 9.1 `AudioImportSettingsV1`

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

Import時に音量を自動正規化してSource意図を変えない。Integrated loudnessとtrue peakは測定metadataであり、`gain_policy=apply_explicit_db`の承認なしにsampleへ焼かない。

### 9.2 形式と検証

- WAVはRIFF／RF64 bounds、WAVEFORMATEXTENSIBLE、valid bits、sample format、channel mask、block alignment、chunk overlapを検証する。C1はPCM整数とIEEE floatだけを受理する。
- FLACはRFC 9639に従い、STREAMINFO、frame、CRC、sample rate、channel、bit depth、total sampleを検証する。
- Cooked streamはRFC 7845に整合するOpus packet、pre-skip、output gain、granule mappingをEngine-owned manifestへ正規化する。RuntimeはSource Ogg parserを使用しない。
- Decode sampleにNaN／InfがあるSource、負duration、範囲外loop、unknown channel layoutを拒否する。

### 9.3 Preview

Waveform、sample-accurate loop、loudness、true peak、channel、trim、resident／stream cost、Target codec A／B auditionを提供する。Preview gainとCook gainを別状態にせず、Planの設定から同じdecode／resample／encode codeを使う。

## 10. Font Import

`FontImportSettingsV1`はrequired locale／script、glyph coverage、variable axis policy、color glyph policy、hinting、fallback family、atlas／runtime raster policy、embedding permission recordを持つ。

- OpenType 1.9.1のtable directory、offset／length、checksum、glyph、cmap、GSUB／GPOS／variation boundsを検証する。
- Font内の名前、license URLだけで商用利用可と判定せず、Asset License Recordを必須にする。
- Project required localeのcoverage不足はwarningでなくProduction package blockとする。
- SVG／bitmap color glyph等の未対応tableを黙ってoutlineへ縮退しない。
- Previewはrequired script sample、fallback発生、missing glyph、variable axis、small／large size、Target raster差を表示する。

## 11. Editor Asset workflow

### 11.1 Asset Browser

Asset Browserは次を一つのStable ID selection modelで提供する。

- logical folder、type、semantic role、tag、license、Production readiness、diagnosticでfilter
- thumbnail／waveform／font sample／3D turntable
- Source、Import revision、Active generation、Target residency、dependency／reverse dependency
- duplicate content hash候補と「同一Assetである」という意味判断の分離
- missing／stale／failed／awaiting approval／ready状態
- multi-selectへ同一Schema fieldだけをbatch edit

表示index、file path、thumbnail objectをOperation targetにしない。

### 11.2 Import Inspector

Import Inspectorは`Source`、`Analysis`、`Profile`、`Preview`、`Conversion`、`Dependencies`、`Diagnostics`、`History`を持つ。Basic viewはsemantic roleとProfile候補、High Impact質問だけを表示し、Advanced viewは全型付きfieldとevidenceを表示する。両者は同じDocumentのProjectionであり、別設定を持たない。

### 11.3 Reimport

Source変更時は既存Profileを自動適用してAnalysisとPreviewまで実行できるが、次の場合は自動promotionしない。

- Source hierarchy、Skeleton、Material slot、Animation clip、Texture channel、Audio channel／duration、Font coverageの破壊的変更
- Importer ID／major version、Source converter、Profile schema、Target codecの変更
- 新しいwarning／loss、budget超過、dependency削除
- Assetを参照するScene／Material／Cue／UIへ意味Diffが発生

`AssetReimportConflictV1`は対象Stable ID、before／after、参照consumer、severity、許可Operationを持つ。ConflictはSource修正、Profile変更、明示Migrationのいずれかで解決し、名前一致による自動再接続を禁止する。

### 11.4 長時間Job

Import、Preview、Cook、Reimport、bulk migrationはcancel可能なJobとして表示する。Stage、progress unit、current asset、CPU／memory／output budget、Diagnostic countを表示し、modal dialogでEditor全体をblockしない。Cancel後のpartial outputをArtifact storeへpublishしない。

## 12. AI contract

### 12.1 Read Operation

| Operation | 出力 |
|---|---|
| `asset.query_catalog` | Stable ID、type、role、readiness、budget、diagnostic summary |
| `asset.inspect_source` | `AssetSourceAnalysisV1`とSource dependency |
| `asset.inspect_import_profile` | flattened Profile、由来、許可範囲 |
| `asset.inspect_conversion_report` | applied conversion、loss、before／after |
| `asset.inspect_dependencies` | forward／reverse dependency closure |
| `asset.inspect_reimport_conflicts` | typed conflictとconsumer impact |

### 12.2 Proposal Operation

| Operation | Risk |
|---|---|
| `asset.propose_import_profile` | R1。High Impact質問があればPlanを作らず質問を返す |
| `asset.propose_import_settings_change` | R1～R2。対象fieldとconsumer impactを提示 |
| `asset.request_preview` | R0。Project非変更 |
| `asset.propose_reimport` | R1～R3。破壊的Conflictは人間承認必須 |
| `asset.propose_bulk_profile_migration` | R3。最大100 Asset／ChangeSet、closure preview必須 |
| `asset.propose_placeholder_replacement` | R1。required roleへ汎用placeholder不可 |

AIはProfile ID、Asset ID、Source pathを推測生成しない。Catalogにない選択肢、未Activated Format Capability、Target非対応codecを提案した場合は`CapabilityNotActivated`を返す。

### 12.3 説明可能性

AI提案は次を同時に返す。

- 選択したProfileと候補から除外したProfile
- Source metadata、Project Profile、Target、budgetのevidence
- 適用される変換と保持される値
- visual／behavior／memory／package impact
- 未解決質問、承認者、rollback

自然言語だけでなく`AssetImportPlanV1`とDiagnostic IDを正規出力にする。

## 13. Security、determinism、toolchain

- 全parserはBrokerが渡したbounded read handleだけを読む。
- Source dependencyはmanifest化し、Import中の新規path探索を禁止する。
- archive／embedded resourceは展開後byte、file count、depth、ratioにhard capを持つ。
- 外部Toolはexact executable／library／script hash、version、license、SBOM、sandbox profileを`ToolReceiptV1`へ記録する。
- Blender／OpenUSD plugin search path、Python user site、environment expansion、networkを無効化する。
- Parser、decoder、font shaper、image codecはmalformed、OOM、timeout、integer overflow、decompression bomb corpusを持つ。
- 同一Jobをclean workerで二回実行し、IR canonical bytes、Report、Artifact hashを一致させる。
- GPU driver依存のSource decode、preview判定、Production Cookを禁止する。
- Preview thumbnail等の非決定的presentation Artifactを正式Artifact hashへ含めない。

## 14. Failure policy

| Diagnostic | 結果 |
|---|---|
| `UnsupportedSourceFormat` | Analysis reject。別形式への変換方法を案内 |
| `CapabilityNotActivated` | 該当Importerを実行せず、Activated形式を提示 |
| `UnsupportedSourceFeature` | Preview／Cook停止。承認済みfallbackがある場合だけ別Plan |
| `AmbiguousSemanticRole` | `NeedsProfile`。AIが推測確定しない |
| `CoordinateIntentUnknown` | Source値を保持し、front／placement変更をblock |
| `NonDecomposableTransform` | C1 Scene Cook拒否。Static bake候補だけ提示 |
| `SkeletonBindingMismatch` | Skeleton／Clip closure promotion拒否 |
| `ColorEncodingConflict` | Texture Cook拒否 |
| `AudioChannelLayoutUnknown` | Audio Cook拒否 |
| `FontCoverageMissing` | Required localeのPackage拒否 |
| `ImporterVersionConflict` | 旧generation維持、Migration Preview要求 |
| `ExternalToolUnavailable` | Source conversion失敗、Project revision不変 |
| `DeterminismMismatch` | Importer versionをProduction禁止 |
| `ReimportConsumerConflict` | 自動promotion禁止、Conflict Viewへ遷移 |

## 15. Qualification fixture

### 15.1 共通

- valid最小／最大、truncated、overflow、NaN／Inf、cycle、duplicate、unknown extension
- clean二回Import／CookのIR、Report、Artifact hash一致
- Source、Profile、Importer、Tool、dependency、Target変更の正確なinvalidation
- cancel、timeout、Worker crash、OOM、disk fullで旧generation維持
- AI／手動Editor／headless CLIが同じPlan hash、Diagnostic、Artifactへ収束
- Reimportで非破壊変更は設定保持、破壊的変更はConflict block

### 15.2 3D

- Khronos glTF Asset Generator、Sample Assets、Validator
- +X／+Y／+Z軸矢印、1 m cube、原点外配置、複数Root、空Root
- parent T／R／S、negative determinant、non-uniform scale、shear、singular matrix
- pivot、geometric transform、front metadata、unit 1 m／cm／mm
- static／skinned／morph、bind pose、Animation、root motion、loop
- Blender→GLB、FBX→IR、USD selected root→IRの同一意味fixture
- Importer version変更によるhierarchy／rest pose／material order Conflict

### 15.3 Texture／Sprite

- PNG Third Edition test、KTX2 validation、OpenEXR scanline／tiled、DDS BC
- sRGB／linear／ICC、straight／premultiplied alpha、normal ±Y、data texture
- odd size、mip、cube／array、sprite pivot／PPU／padding
- Target圧縮後のPSNR／normal angular error／alpha coverage／GPU byte gate

### 15.4 Animation

- exact Skeleton、missing／extra／reordered joint、noninvertible bind
- step／linear／cubic、reverse、loop、seek、clip slice、root trajectory
- key reduction threshold、retarget pose corpus、Animation Event非発火scrub

### 15.5 Audio

- WAV PCM8／16／24／32、float32、RF64、channel mask、unknown chunk
- FLAC CRC、seek、corrupt frame、44.1／48／96 kHz
- Opus pre-skip、gain、seek、loop seam、malicious oversized packet
- loudness／true peak golden、resident／stream boundary

### 15.6 Font

- required locale coverage、complex script shaping、variation、color glyph
- corrupt table、overlap、recursive composite glyph、oversized outline
- embedding permission、fallback chain、missing glyph

## 16. Definition of Done

Asset Import機能は次をすべて満たすまでC1／C2／C3へ昇格しない。

1. MCD type、Profile、Operation、Diagnostic、Stateが同じ正本からC++／Editor／AI Tool／CLIへ生成される。
2. Source Analyzer、IR、Validator、Preview、Cook、Receiptが一つのhash chainで追跡できる。
3. Basic／Advanced Editor、AI、headless CLIが同一Planへ収束する。
4. Import前後の座標、色、音声、Skeleton／Animation意味とlossを確認できる。
5. Reimportが設定を保持し、破壊的差分をconsumer closure付きでblockする。
6. 全Targetでmemory、codec、format、Package validationが合格する。
7. malformed、fuzz、OOM、timeout、determinism、rollback、hot reloadを合格する。
8. Requirement Coverage MatrixがRequirement、MCD、Validator、Editor、AI Operation、test、実装Symbol、Receiptを追跡する。
9. User documentation、AI tool description、Import Profileが同じschema versionを参照する。
10. Activated formatだけがEditorとAI Catalogへ現れ、未成熟形式を選択できない。

## 17. 段階導入

| Work Package | 到達点 |
|---|---|
| AS0 Contract | MCD、Profile、Plan、IR、Diagnostic、Report、Receipt、state、headless fixture |
| AS1 Texture／Sprite | PNG／JPEG／OpenEXR／KTX2／DDS、Preview、Target Cook、Sprite |
| AS2 Audio／Font | WAV／FLAC、Opus Cook、loop／loudness、OpenType coverage |
| AS3 glTF Scene | glTF／GLB、Transform／Mesh／Material、Skeleton／Animation、Preview |
| AS4 Editor／AI | Asset Browser、Import Inspector、Diff、Reimport Conflict、AI Operation |
| AS5 C1 Integration | Package、Streaming、Hot Reload、2D／3D vertical slice |
| AS6 Blend／FBX C2 | Blender Worker、`ufbx` Adapter、Migration／loss fixture |
| AS7 USD C3 | OpenUSD Stage resolver、composition／variant／payload、selected-root import |

各Packageは前段のContractとfixtureを使用し、独立した実装計画とReview Gateを持つ。AS6／AS7をAS0～AS5と並行実装してC1完成を遅らせない。

## 18. 一次資料

### 18.1 規範仕様

- [Khronos glTF 2.0 Specification](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html)
- [Khronos glTF Validator](https://github.com/KhronosGroup/glTF-Validator)
- [Khronos glTF repository／Asset Generator／Sample Assets](https://github.com/KhronosGroup/glTF)
- [KTX 2.0 Specification](https://registry.khronos.org/KTX/specs/2.0/ktxspec.v2.html)
- [W3C PNG Third Edition](https://www.w3.org/TR/png-3/)
- [JPEG 1／ISO/IEC 10918 overview](https://jpeg.org/jpeg/)
- [OpenEXR Technical Introduction](https://openexr.com/en/latest/TechnicalIntroduction.html)
- [Microsoft WAVEFORMATEXTENSIBLE](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/ksmedia/ns-ksmedia-waveformatextensible)
- [RFC 9639: FLAC](https://www.rfc-editor.org/rfc/rfc9639.html)
- [RFC 7845: Ogg Opus](https://www.rfc-editor.org/rfc/rfc7845.html)
- [OpenType 1.9.1](https://learn.microsoft.com/en-us/typography/opentype/spec/)
- [OpenUSD Introduction](https://openusd.org/release/intro.html)
- [Blender USD／orientation／unit documentation](https://docs.blender.org/manual/en/dev/files/import_export/usd.html)
- [`ufbx` official repository](https://github.com/ufbx/ufbx)

### 18.2 製品UX比較

- [Unity Model Import Settings](https://docs.unity3d.com/Manual/FBXImporter-Model.html)
- [Unity Managing importers with scripts](https://docs.unity3d.com/Manual/ScriptedImporters.html)
- [Unreal Engine Interchange Framework](https://dev.epicgames.com/documentation/en-us/unreal-engine/importing-assets-using-interchange-in-unreal-engine)
- [Unreal Engine `UFbxAssetImportData`](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Editor/UnrealEd/UFbxAssetImportData)
- [Godot Available 3D formats](https://docs.godotengine.org/en/4.7/tutorials/assets_pipeline/importing_3d_scenes/available_formats.html)
- [Godot Advanced Import Settings](https://docs.godotengine.org/en/stable/tutorials/assets_pipeline/importing_3d_scenes/advanced_import_settings.html)
- [Godot `EditorImportPlugin`](https://docs.godotengine.org/en/stable/classes/class_editorimportplugin.html)

外部資料は形式、API、既知のinteroperability問題、実用UXを確認する根拠である。Miraikanai固有のIR、Profile、Operation、Risk、Diagnostic、Receipt、段階導入は本書を正本とする。
