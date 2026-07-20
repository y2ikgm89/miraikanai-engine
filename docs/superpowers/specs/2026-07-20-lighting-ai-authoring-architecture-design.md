# Miraikanai Engine Lighting／AI Authoringアーキテクチャ規約

- 文書版: 1.0
- 作成日: 2026-07-20
- 対象: 2D／3D Light Source、Lighting Intent、Editor、AI Operation、Light Selection／Cluster、Runtime Snapshot、Budget、Qualification
- 状態: プロジェクト公式の規範設計レビュー版
- 機能範囲: [Miraikanai Engine 2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)
- Renderer実行正本: [Miraikanai Engine Rendering／Render Graphアーキテクチャ規約](./2026-07-19-rendering-render-graph-architecture-design.md)
- Material規約: [Miraikanai Engine Material／Visual Style／AI Authoringアーキテクチャ規約](./2026-07-20-material-visual-style-ai-authoring-architecture-design.md)
- Environment規約: [Miraikanai Engine Environment Platform／AI Authoringアーキテクチャ規約](./2026-07-20-environment-platform-ai-authoring-architecture-design.md)
- Camera規約: [Miraikanai Engine Camera Platform／AI Authoring／Virtual Productionアーキテクチャ規約](./2026-07-20-camera-platform-ai-authoring-virtual-production-architecture-design.md)
- 契約規約: [Miraikanai Engine 実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)
- Runtime規約: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- Asset規約: [Miraikanai Engine Asset Pipeline／Content Package規約](./2026-07-19-asset-pipeline-content-packaging-design.md)
- AI権限規約: [Miraikanai Engine AI実装・保守ガバナンス規約](./2026-07-19-ai-engine-development-governance-design.md)
- AI検証規約: [Miraikanai Engine AI検証・評価・来歴規約](./2026-07-19-ai-verification-evaluation-provenance-design.md)

## 0. 30秒で分かる公式方式

Miraikanai EngineのLightingは、LightをGPU向け数値の寄せ集めとして扱わない。個々の光源をversion付き`LightSourceV1`、人間またはAIの意味要求を`LightIntentV1`、Project全体の見た目と制約を`LightingStyleProfileV1`として保存し、純粋で決定論的な`LightIntentResolverV1`がTarget別`ResolvedLightPlanV1`へ解決する。

```text
Human／AI lighting goal
  -> LightIntentV1
  -> validate + resolve
  -> ResolvedLightPlanV1
  -> approved ChangeSet
  -> LightSourceV1
  -> LightSnapshotV1
  -> Renderer-owned selection／cluster／pass
  -> D3D12／Vulkan／Metal
```

AIへCluster buffer、native texture format、descriptor index、shader binding、Render pass、GPU addressを公開しない。AIは「主人公を暖かいKey Lightで読みやすくする」「pixel artの輪郭を崩さない」などの意味を編集し、Engineが物理量、光源数、Target Capability、Shadow、Material、Budgetへ解決する。

## 1. 設計目標

1. 2D Sprite、3D Mesh、Toon、PBR、Pixel Artで共通のAI可読Lighting語彙を持つ。
2. lumen、lux、candela、nit、kelvin、meterを混同しない。
3. Source Intent、解決済みPlan、Runtime Snapshot、GPU resourceを分離する。
4. 同じ入力、Catalog version、Target Profileから同じPlan hashを得る。
5. Light、Material、Environment、Shadow、Post Processの責務を循環させない。
6. Target上限超過を無言の破棄にせず、事前Plan、fallback、Diagnosticとして説明する。
7. AIがContext全体を読まなくても、bounded Search／Inspect／Plan／Preview／Explainで安全に作業できる。
8. Portable Rasterを基準にし、高機能をCapabilityとQualificationで段階導入する。

## 2. 責務と非責務

| 主題 | 正本／Owner |
|---|---|
| Light type、物理単位、role、Intent、Profile、Resolver | 本規約 |
| `LightSourceV1`、`LightSnapshotV1`の論理契約 | 本規約 |
| Light selection／clusterの論理アルゴリズム、上限、overflow | 本規約 |
| Render Graph、pass、resource lifetime、native format／binding | Renderer規約 |
| Shadow Intent、Shadow technique、atlas、cache、RT Shadow | 2D／3D機能計画のShadow節 |
| BRDF、Shading Model、Material parameter、Visual Style | Material規約 |
| Sun／Moon／Sky／IBLの環境意味と生成artifact | Environment規約 |
| Camera exposure／lens／view family | Camera規約 |
| Gameplay visibility、stealth、damage、AI perception | Gameplay／Simulation |
| Asset import、IES／Cookie validation、cook | Asset／Asset Import規約 |

次を禁止する。

- Render結果の明るさからGameplay visibilityを自動決定する。
- Shadow techniqueの内部数値を`LightSourceV1`へ重複保存する。
- MaterialがLight listやShadow atlasを直接所有する。
- EnvironmentがScene内の通常Lightを暗黙生成する。生成が必要な場合は明示的な`LightSourceV1`参照を持つ。
- Editor表示objectをLightingの正本にする。

## 3. モジュール境界

```text
mirakan::lighting::contract
  LightSource／Intent／Profile／Plan／Diagnostic

mirakan::lighting::authoring
  Search／Inspect／Resolver／Planner／Preview／Explain

mirakan::lighting::runtime
  Extractor／Selector／ClusterBuilder／SnapshotPublisher

mirakan::lighting::cook
  IES／Cookie cooker、Target specialization、Budget proof

mirakan::render
  Render Graph、GPU resource、pass、backend adapter
```

`Authoring`はGPU APIをimportしない。`Runtime`はAuthoring DBを直接読まず、承認済みProject revisionとCooked artifactだけを読む。`Render`はLightの意味Intentを再解釈せず、`LightSnapshotV1`と`ResolvedLightPlanV1`の実行フィールドだけを消費する。

## 4. 正本データモデル

### 4.1 `LightSourceV1`

`LightSourceV1`はProjectに保存する個別光源の正本である。

| Field | 型／規約 |
|---|---|
| `light_id` | 永続Stable ID。Entity indexやGPU indexを使わない |
| `revision` | 楽観的排他とChangeSetのbase revision |
| `dimension` | `light_2d`／`light_3d` |
| `light_type` | `directional`／`point`／`spot`／`rectangle`／`disk` |
| `mobility` | `static`／`stationary`／`dynamic` |
| `enabled` | 明示bool。強度0による無効化を正規表現にしない |
| `transform_ref` | WorldのStable Transform ID。Directionalは位置を意味に使わない |
| `radiometry` | 4.3節のtagged union |
| `color_source` | 4.4節のtagged union |
| `range_m` | 有限局所光の影響半径。Directionalでは禁止 |
| `spot_shape` | Spotだけのinner／outer angle |
| `source_radius_m` | Point／Spotの有限Emitter半径。singularity回避だけの任意epsilonにしない |
| `emitter_shape` | Rectangle／Diskの物理寸法 |
| `channel_mask` | 32-bit論理Lighting Channel。予約bitを検証する |
| `role` | `key`等の意味role。描画計算には直接使わない |
| `group_ids` | Semantic groupのStable ID。最大8 |
| `importance` | `critical`／`important`／`decorative` |
| `priority_u8` | 同一importance内の0～255優先度 |
| `shadow_intent_ref` | version付きShadow Intent参照。nullable |
| `cookie_asset_id` | 検証済みTexture artifact参照。nullable |
| `ies_asset_id` | 検証済みIES artifact参照。nullable、C2 |
| `target_policy_ref` | Target別上限とfallback policy |
| `human_lock_mask` | AIが変更できないfield集合 |

2Dと3Dは同じ型名を使うが、`dimension`ごとに許可組合せをSchemaで閉じる。C1の2Dは`directional`、`point`を許可し、spot、area light、IESをC2とする。C1の3Dは`directional`、`point`、`spot`を許可し、rectangle／disk／IESをC2とする。

### 4.2 `LightIntentV1`

`LightIntentV1`は意味要求であり、GPU実装指定ではない。

```text
LightIntentV1
  intent_id
  scope
  role
  subject_group_ids[]
  mood
  contrast
  coverage
  direction_intent
  softness_intent
  color_intent
  mobility_intent
  importance
  target_selector
  quality_selector
  fallback_priority[]
  constraints
  base_revision
```

閉じた語彙は次とする。

- `role`: `key`、`fill`、`rim`、`environment`、`practical`、`accent`、`effect`、`decorative`
- `mood`: `neutral`、`warm`、`cool`、`dramatic`、`soft`、`high_key`、`low_key`、`toon_banded`、`pixel_crisp`、`custom_profile`
- `contrast`: `flat`、`balanced`、`strong`
- `coverage`: `subject`、`local`、`room`、`outdoor`、`world`
- `direction_intent`: `camera_left`、`camera_right`、`front`、`back`、`above`、`below`、`world_direction`、`match_reference`
- `softness_intent`: `hard`、`balanced`、`soft`
- `mobility_intent`: `prefer_static`、`allow_stationary`、`require_dynamic`

`constraints`は「既存のhuman lockを保持」「追加光源最大数」「変更可能group」「Shadowを増やさない」「Gameplay-critical materialの可読性を下げない」などの型付き制約だけを許可する。自由文は説明として保持できるが、実行条件には使わない。

### 4.3 `LightingStyleProfileV1`

ProjectまたはSceneの標準Styleを表し、次を所有する。

- profile ID、version、parent profile
- Default role recipe
- `VisualStyleProfile`参照と整合constraint
- Targetごとの許可light type／shadow tier
- Exposure基準との接続方針
- 2D normal lighting、Toon band、Pixel Art quantizationの許可方針
- 色温度、彩度、contrastの許容範囲
- critical／decorative別fallback priority
- Preview fixtureとQualification Receipt参照

Profile継承は最大4段とし、循環を禁止する。Profile mergeはfieldごとに`inherit`／`replace`を明示し、配列の暗黙appendを行わない。

### 4.4 `ResolvedLightPlanV1`

Resolverの出力は必ず次を含む。

- 入力Intent、Profile、Catalog、Target Capabilityのversionとhash
- 追加、更新、削除する`LightSourceV1`のexact patch
- 解決した物理量、色、type、range、mobility、channel、importance、priority
- Shadow Intent参照の追加または維持
- Light selection／cluster上限に対するworst-case proof
- 予測CPU時間、GPU時間、persistent／transient byte
- 保持したhuman lockと満たせなかった制約
- 採用案、棄却案、理由、fallback chain
- 必要なAsset cook、Preview、承認、Qualification
- `plan_hash`と有効期限

PlanはProjectを変更しない。承認済みChangeSetがCommitされた後だけ`LightSourceV1`が更新される。Catalog、Target Profile、base revisionのどれかが変わったPlanは再解決する。

## 5. 物理単位と数値規約

### 5.1 放射量tagged union

`radiometry`は次のどれか一つだけを持つ。

| Light type | 正規入力 | 単位 |
|---|---|---|
| Directional | `illuminance_lux` | lux |
| Point | `luminous_flux_lm` | lumen |
| Spot | `luminous_flux_lm`またはqualified IES | lumen／candela distribution |
| Rectangle／Disk | `luminance_nit` | cd/m² |

単位なしの`intensity`を正本にしない。Legacy値をimportする場合はsource unit、変換式、assumption、誤差をReceiptへ残す。

### 5.2 距離減衰

C1 Pointのscalar referenceは次で固定する。

```text
delta_m = light_position - surface_position
distance_sq_m2 = dot(delta_m, delta_m)
distance_m = sqrt(distance_sq_m2)
d2 = max(distance_sq_m2, max(source_radius_m, 0.01 m)^2)
x  = saturate(distance_m / range_m)
w  = x < 1 ? (1 - x^4)^2 : 0
I_cd = luminous_flux_lm / (4 * pi)
E_lux = I_cd * w / d2
```

Spotは単位積分が1となる`SpotDistributionV1`をCook時に生成し、その角度分布へ総光束を配る。inner／outer angleの単純乗算を行い、総光束がangleで変化する実装は禁止する。CPU scalar reference、HLSL、MSL、SPIR-V経路は同一fixtureで相対誤差を検証する。

`source_radius_m`はEmitter geometryまたはStyle ProfileからResolverが決め、AIがinverse-square singularity回避値を直接指定しない。

### 5.3 色

`color_source`は次のtagged unionとする。

- `linear_rgb`: scene-linear D65基準、各成分0以上
- `cct`: kelvin 1000～20000とtint -1～1
- `profile_default`: `LightingStyleProfileV1`のrole既定値

同じLightへRGBとCCTを同時保存しない。EditorのsRGB picker値はCommit前にscene-linearへ変換し、色空間と変換versionをReceiptへ記録する。

### 5.4 Geometry制約

- `range_m`は0より大きくTarget上限以下。
- Point／Spotの`source_radius_m`は有限な0以上かつ`range_m`以下。
- Spotは`0 <= inner_angle_deg < outer_angle_deg < 179`。
- Rectangle／Diskは0より大きい寸法を持つ。
- Directionalはrange、position attenuation、IESを持たない。
- CookieとIESはAsset type、dimension、license、cook statusを検証する。
- NaN、Inf、denormal依存、負の放射量を拒否する。

## 6. Light selectionとCluster

### 6.1 2D C1

2D C1のView上限は次を基準とする。

- Directional: 1
- 可視Point: 64
- Spriteごとの寄与Light: 8
- Shadow caster Light: 8

Spriteごとの候補は`channel match -> importance rank desc -> priority_u8 desc -> screen coverage desc -> light_id asc`の順で安定sortし、先頭8件を採用する。Authoring／Cook時に上限超過を検出した場合は失敗または明示fallbackとし、Runtimeで動的に超過した場合だけ同じsortでbounded selectionを行い`MIRAKAN-LIGHTING-RUNTIME-OVERFLOW`を集計する。

### 6.2 3D Clustered／Forward+

XY tileはC1で16×16 pixelを基準とする。Z slice数はTarget Profileが16／24／32から選ぶ。`near_m > 0`、`cluster_far_m = min(camera_far_m, 10000 m)`とし、境界は次で固定する。

```text
z[i] = near_m * pow(cluster_far_m / near_m, i / slice_count)
where i = 0..slice_count
```

境界配列はdeterministic scalar referenceがFP32へ一度だけ量子化してViewごとに生成し、CPU cullingとGPU cluster lookupの両方が同じ配列を使う。区間は最後を除き`[z[i], z[i+1])`、最後だけ閉区間とする。CPUとShaderで別々にlogを再計算してはならない。

Cluster listは固定capacityとoverflow listを持つ。上限はTarget Profileで宣言し、次を必須にする。

- clusterごとの最大Light数
- Viewごとの最大index数
- overflow発生時の安定sort
- worst-case persistent／transient byte
- build CPU／GPU時間
- overflow count／最大超過数telemetry

`critical -> important -> decorative`を最初のkeyとし、同一importanceでは`priority_u8 desc -> projected contribution desc -> distance asc -> light_id asc`で決定する。Critical LightをDecorative Lightより先に落とすfallbackは禁止する。

### 6.3 `LightSnapshotV1`

SimulationからRendererへpublishするimmutable snapshotは論理SoAとする。

```text
LightSnapshotV1
  generation
  view_family_id
  compact_light_ids[]
  type_and_flags[]
  position_and_range[]
  direction_and_cone[]
  color_and_radiometry[]
  shape_parameters[]
  channel_masks[]
  shadow_plan_refs[]
  source_revisions[]
```

GPU packingはMCDから生成する`LightGpuRecordV1`とBackend Adapterが所有する。`LightSnapshotV1`へnative handle、descriptor、GPU address、DXGI／Vk／Metal formatを保存しない。Compact indexはsnapshot generation内だけ有効で、Save、Replay、DiagnosticはStable `light_id`を使う。

## 7. Intent Resolver

`LightIntentResolverV1`は副作用を持たない。

```text
resolve(
  LightIntentV1,
  LightingStyleProfileV1,
  VisualStyleProfile,
  EnvironmentLightingSummaryV1,
  MaterialReadabilitySummaryV1,
  SceneLightingSummaryV1,
  TargetCapabilitySnapshotV1,
  LightingBudgetEnvelopeV1,
  PolicySnapshotV1
) -> ResolvedLightPlanV1 | LightingDiagnosticSetV1
```

解決順は固定する。

1. Schema、Stable ID、base revision、権限を検証する。
2. scope、subject、human lockを解決する。
3. Visual StyleとEnvironmentからrole recipeを選ぶ。
4. Targetで利用可能なLight type、Shadow tier、Assetを絞る。
5. 物理量、色、配置候補を生成する。
6. 可読性constraintとBudgetを評価する。
7. fallback chainを順に適用する。
8. Plan、理由、cost、risk、Preview差分を出力する。

同順位候補はCatalog順ではなく明示score tupleとStable IDで決定する。wall clock、unordered container走査順、GPU timingの瞬間値、LLMの再生成結果をPlan決定に使わない。

## 8. AI／Editor Operation

MCDの公開Operationは次に限定する。

| Operation | Risk | 結果 |
|---|---:|---|
| `operation.lighting.search` | R0 query | bounded ID／label／role一覧 |
| `operation.lighting.read` | R0 query | field mask付きversioned Source／Intent／Profile／Plan |
| `operation.lighting.inspect` | R0 query | Scene summary、constraint、cost、Diagnostic |
| `operation.lighting.resolve_intent` | R0 query | `ResolvedLightPlanV1` |
| `operation.lighting.plan_change` | R1 proposal | `LightingChangeSetProposalV1`、diff、risk、approval |
| `operation.lighting.preview_change` | R0 query／job | Preview artifact、visual metric、期限 |
| `operation.lighting.explain_plan` | R0 query | 採用／棄却／fallback理由 |
| `operation.lighting.estimate_cost` | R0 query | Target別予測、根拠、confidence |
| `operation.lighting.validate_change` | R0 query／job | Schema／semantic／Capability／Budget結果 |

外部MCP aliasは`mirakan.lighting.search`等とし、内部Operation IDへ一対一に写像する。検索は既定50件、最大200件、継続token付きとする。Readはfield maskを必須とし、大規模Scene全体を既定返却しない。

OperationはCommit、Build、Cook、Play、GPU submissionを実行しない。変更は共通`AuthoringCommandGateway`のChangeSet、Preview、承認、Commitを通る。Human lock、R2 Asset生成、Project HLSL、L3 Shadow Techniqueを必要とする場合は自動で権限を拡張せず、必要な別Operationと承認理由を返す。

### 8.1 AI向けContext Summary

`SceneLightingSummaryV1`は次だけをboundedに返す。

- Scene、View Family、Target、Visual Style、EnvironmentのID／version
- role／type／importance別Light数
- Critical Light一覧
- 上限、現在値、予測cost、overflow
- Shadow tier分布
- human lock件数
- active Diagnostic上位件数
- 詳細取得用Stable IDとcontinuation token

Raw transform配列、全Material、全cluster list、GPU buffer dumpは要求されたbounded inspect以外で返さない。

## 9. Preview、Explain、Evidence

Lighting Previewは最低限、次を同じcamera fixtureで生成する。

- Beauty
- unlit reference
- direct diffuse
- direct specular
- Light contribution heatmap
- Shadow mask
- overdraw／cluster occupancy
- before／after split

Preview artifactはProject revision、Plan hash、Target Profile、Renderer build、Material／Environment version、camera fixture、random seedを持つ。古いrevisionのPreviewをCommit根拠に使わない。

`LightingPlanExplanationV1`は、変更したLightごとに「どのIntent fieldがどのSource fieldへ解決されたか」、代替案を棄却した理由、予測cost、視覚的risk、fallback時に失われるcueを返す。

## 10. BudgetとFallback

LightingはRuntime規約の`Opaque／Material／Lighting`合計3.00 ms枠の内訳として測定し、別枠3.00 msを追加要求しない。Target Profileは最低限、次を宣言する。

- Light extraction CPU
- selection／cluster build CPU／GPU
- direct lighting GPU
- shadowはShadow側の内訳
- Light buffer persistent byte
- cluster transient／persistent byte
- visible Light、shadow caster、cluster index上限

Fallback順はProfileで固定し、既定を次とする。

1. Decorative LightのShadowを下げる。
2. Decorative Lightをbaked／probe／emissive表現へ置換する。
3. 重要度の低い局所Lightのrangeと更新頻度を下げる。
4. Area／IESをqualified近似へ置換する。
5. Decorative Lightを安定sortで除外する。
6. Critical Lightを維持できなければPlanを拒否する。

自動fallbackはGameplay stateを変更しない。Visual cueがGameplay上重要な場合、そのLightを`critical`とし、代替UI／Audio cueをGameplay／Accessibility仕様で別途定義する。

## 11. Diagnostic

Diagnostic IDは閉じたnamespaceを使う。

| ID | 意味 |
|---|---|
| `MIRAKAN-LIGHTING-SCHEMA-INVALID` | 型、enum、required field不正 |
| `MIRAKAN-LIGHTING-UNIT-MISMATCH` | Light typeと物理単位が不一致 |
| `MIRAKAN-LIGHTING-COLOR-AMBIGUOUS` | RGBとCCT等が重複 |
| `MIRAKAN-LIGHTING-GEOMETRY-INVALID` | range、angle、shape、transform不正 |
| `MIRAKAN-LIGHTING-ASSET-UNQUALIFIED` | Cookie／IES artifactが未検証 |
| `MIRAKAN-LIGHTING-TARGET-UNSUPPORTED` | Target Capabilityで実行不能 |
| `MIRAKAN-LIGHTING-BUDGET-EXCEEDED` | PlanがBudgetを満たさない |
| `MIRAKAN-LIGHTING-CRITICAL-DROPPED` | Critical Light維持不能 |
| `MIRAKAN-LIGHTING-RUNTIME-OVERFLOW` | 動的selection／cluster上限超過 |
| `MIRAKAN-LIGHTING-STALE-PLAN` | revision／Catalog／Target変更 |
| `MIRAKAN-LIGHTING-LOCK-CONFLICT` | human lockとIntentが競合 |
| `MIRAKAN-LIGHTING-NONDETERMINISTIC` | 同一入力でPlan hash不一致 |

全Diagnosticはseverity、owner、Stable ID、Target、evidence、推奨修正、fallback可否を持つ。

## 12. Capability段階

| 段階 | Lighting機能 |
|---|---|
| C0 | MCD、Source／Intent／Profile／Plan、scalar reference、headless Resolver、Validator、fixture |
| C1 | 2D／3D Directional・Point・Spot、2D bounded selection、3D Forward+／cluster、基本Shadow参照、Preview／Explain |
| C2 | Rectangle／Disk、IES／Cookie、advanced cluster、baked／probe連携、Target別高品質Profile |
| C3 | Neural／RT支援、Project Technique、研究機能。個別正式仕様とGateが必要 |

C1でLightの意味契約とPortable Raster基準を完成させる。C2／C3のためにC1 Sourceへbackend固有fieldを追加しない。

## 13. 検証とEval

### 13.1 Contract／Unit

- 全tagged unionのpositive／negative fixture
- lux／lumen／candela／nit変換のreference test
- Point attenuationの距離別golden
- Spot distributionの単位積分
- RGB／CCT変換の色差
- Profile継承、循環、merge
- Resolverの同一入力同一Plan hash
- human lock、stale revision、unsupported Target

### 13.2 Runtime／Renderer

- CPU／GPUのZ境界、cluster所属、overflow selection一致
- 2D sprite per-light selectionの安定性
- D3D12／Vulkan／Metalで同じlogical planを実行
- Light snapshot generationとCompact ID寿命
- device loss、surface resize、Target切替
- 最大Light数、最大cluster index、dynamic churnのsoak

### 13.3 Visual Eval

固定Sceneとして、2D unlit、2D normal map、pixel art、PBR室内、PBR屋外、Toon、透明物、VFX混在、極端な小／大スケールを持つ。各Sceneでbeauty、contribution、shadow、cluster、GPU costを比較し、Target別許容差をReceiptへ固定する。

AI Evalは最低限、次を含む。

- 「主人公を暖かく、背景を冷たく」
- 「低価格Androidで雰囲気を極力保つ」
- 「pixel artをぼかさず夜にする」
- 「既存の人手調整Key Lightを変えない」
- 「Shadow costを増やさず敵を読みやすくする」

正解は自然文一致ではなく、Schema妥当性、lock保持、Target適合、Budget、Plan再現性、説明、視覚metricで判定する。

## 14. 有名Engineから採用する原則

| Engine | 参考にする公開方式 | Miraikanai Engineでの採用 |
|---|---|---|
| Unreal Engine | 反射可能なProperty、Light Component、Editor scripting | Property発見性、Category、metadata、Editor自動化の考え方 |
| Unity SRP | serialized Component、Scriptable Render Pipeline、Graphics Settings | Asset化したProfileとRender実装の分離、Target別設定 |
| Godot | Node／Resource、Lightノード、Environment resource | Scene ownershipと再利用可能Resourceの明快さ |

採用しないものは、各EngineのAPI名、Scene形式、Shader binding、Editor object modelをMiraikanaiの公開契約へそのまま移すことである。Miraikanaiは`CapabilitySchema`、MCD、Typed Operation、Resolver、Preview、Explain、ChangeSetを中心にし、AIが「何を変更でき、何が禁止され、Targetでどう解決されたか」を機械的に取得できることを優先する。

参照する一次資料:

- [Unreal Engine Properties](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-uproperties)
- [Unreal Engine Physical Lighting Units](https://dev.epicgames.com/documentation/en-us/unreal-engine/using-physical-lighting-units-in-unreal-engine)
- [Unreal Engine Scripting and Automating the Editor](https://dev.epicgames.com/documentation/en-us/unreal-engine/scripting-and-automating-the-unreal-editor)
- [Unity Graphics repository](https://github.com/Unity-Technologies/Graphics)
- [Godot Engine classes and nodes](https://docs.godotengine.org/en/stable/tutorials/best_practices/what_are_godot_classes.html)
- [Godot Physical light and camera units](https://docs.godotengine.org/en/stable/tutorials/3d/physical_light_and_camera_units.html)

## 15. 実装順序

1. C0 MCDへLight Source／Intent／Profile／Plan／Diagnosticを定義する。
2. scalar radiometry、color、geometry Validatorを実装する。
3. headless Resolverと決定論fixtureを通す。
4. 2D LightSnapshot、bounded selection、unlit／normal-lit fixtureを接続する。
5. 3D cluster boundary共有、Point／Spot、PBR fixtureを接続する。
6. Shadow Intent参照、Material／Environment／Camera summaryを接続する。
7. Editor／AI Operation、Preview、Explain、Receiptを接続する。
8. Target別実測とQualification後にC1へ昇格する。
9. C2機能は一つずつCapability、fallback、比較fixtureを追加する。

## 16. Definition of Done

Lighting C1は次をすべて満たした時だけ完了する。

1. `LightSourceV1`、`LightIntentV1`、`LightingStyleProfileV1`、`ResolvedLightPlanV1`、`LightSnapshotV1`がMCDから生成される。
2. Light typeと物理単位の組合せがSchema／semantic validatorで閉じている。
3. Resolverが同一入力で同一Plan hashを生成する。
4. AI／Editorが同じTyped OperationとChangeSet経路を使う。
5. AIへGPU command、native format、binding、任意Shaderを公開していない。
6. 2D selectionと3D clusterが上限、安定sort、overflow Diagnosticを持つ。
7. CPU scalar referenceと全Shipping shader経路のconformance testが通る。
8. Shadow、Material、Environment、Camera、Post ProcessとのOwner境界に重複正本がない。
9. Before／After PreviewとExplainがPlan hashへ結び付く。
10. Target別CPU／GPU／Memory実測がRuntime Budget内にありReceiptへ固定される。
11. Cross-backend、最大負荷、device loss、stale revision、human lockのtestが通る。
12. C2／C3機能がC1の暗黙必須依存になっていない。
