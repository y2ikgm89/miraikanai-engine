# Miraikanai Engine LOD Contract

- 文書ID: mirakan.arch.rendering-lod
- 状態: review
- 正本範囲: LOD intent／policy、representation set／tier、projected-error／importance／pressure入力、hysteresis／transition、geometry／HLOD／simulation／animation／material／VFX／terrain representation selection、LOD固有operation／diagnostic／qualification
- 非正本範囲: representation asset生成／promotion、Render pass／visibility execution、World streaming activation、Simulation behavior、Runtime shared capacity／phase、Tool version、AI authorization、Evidence envelope、共通Schema／projection。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Math／Core utilities](../02-foundation/math-core.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[Animation](../05-simulation/animation.md)、[Physics](../05-simulation/physics.md)、[Navigation](../05-simulation/navigation.md)、[Render Graph](render-graph.md)、[Materials](materials.md)、[World](world.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論と所有境界

LODは距離だけでなく、projected error、semantic importance、view role、quality intent、runtime pressureを入力に、どのrepresentationを選ぶかを一意に所有する。各Subsystemは候補representationとcost／quality metadataを公開し、LOD Resolverが選択、hysteresis、transition、fallbackを決める。

[Render Graph](render-graph.md)は選択済みgeometry／material representationのvisibilityとdraw executionを所有する。[World](world.md)はCell sourceとstreaming planを所有し、Runtime scheduling／capacity ownerがactivationとpressureを決める。Physics、Navigation、Animationは各Domainのbehavior semanticsを所有し、LODが別のDynamics／Nav／Animation規則を作らない。

Asset import、cook、promotion、generation leaseは[Asset lifecycle](../03-authoring/asset-lifecycle.md)だけが所有する。本書はartifactを生成せず、同一source identityへ紐づく候補artifactの選択条件を定義する。

## 2. 設計判断と共通語彙

LOD語彙を次に固定する。

- `representation`: 同じSource意味を異なるcost／fidelityで表す候補。
- `tier`: 高品質からfallbackまでのcanonical順序を持つ候補識別子。
- `intent`: quality、importance、transition、minimum semanticsのauthoring要求。
- `selection`: View／Domain／subjectごとに解決されたtierと理由。
- `transition`: old／new representationの共存、blend、handoff、history reset条件。
- `pressure`: Runtime ownerが公開するcapacity signal。LODは測定法や閾値を再定義しない。
- `HLOD`: 複数subjectを一つのrender representationへ集約する候補。World identityの置換ではない。

Mesh／Sprite geometryのLOD0は常にSourceの最高detailであり、Sourceを削除／上書きしない。別Domainのtierはclass固有IDで表し、geometry LOD indexを暗黙転用しない。distanceのみ、asset filename、Backend feature名を選択の正本にしない。

`LodClassV1`を次のclosed enumとする。同じobjectは複数classのPolicyを持てるが、一classのtierを別classへ転用しない。

| 値 | 意味 | 主なOwner |
|---|---|---|
| `geometry_detail` | Mesh／Spriteの幾何・表示detail | Asset／Rendering |
| `representation` | individual、instanced、proxy、impostor、非表示 | Rendering |
| `simulation` | Full、契約済み低頻度、dormant record | Runtime／Gameplay |
| `animation_presentation` | pose評価、skinning、presentation bone | Animation／Rendering |
| `material_detail` | 承認済みMaterial feature variant | Material／Rendering |
| `vfx_presentation` | VFX branch、spawn、alive、output detail | Particle／VFX |
| `surface_detail` | Terrain／Foliage／Water／Snow Surfaceの表示detail | Domain Platform |
| `geometry_residency` | geometry cluster／LODのresident集合 | Asset／Rendering |

`LodSemanticPriorityV1`は`critical_gameplay_cue | interactive_subject | primary_subject | supporting_subject | decorative | ambient`のclosed enumである。`critical_gameplay_cue`はhit、危険範囲、parry timing、目標状態等に使用し、decorative／ambientを理由にauthoritative stateを削除しない。`experience_role`と矛盾する場合は`MIRAKAN-LOD-SEMANTIC_ROLE_CONFLICT`で拒否する。

## 3. LOD policyとrepresentation contract

`LODPolicy`はsubject／group scope、quality intent、importance class、error metric、enter／exit boundary、minimum／maximum tier、transition policy、pressure response、Target constraint、fallback、qualification refを持つ。

`RepresentationDescriptor`はStable ID、Domain kind、source identity、artifact generation ref、quality rank、geometric／semantic error bound、required capability、estimated cost class、dependency closure、transition compatibility、fallback targetを持つ。costの共通測定単位、budget、capacity envelopeは[Runtime performance／capacity](../04-runtime/performance-capacity.md)を参照する。

候補setは同じsource semanticsとversion closureに属し、missing tier、duplicate rank、fallback cycle、incompatible dependency、stale artifactを拒否する。低tierがgameplay collision、navigation reachability、authoritative identityを暗黙に変えない。

## 4. 共通選択契約

Resolver inputはsubject Stable ID、representation set generation、ViewFamily／view role、projection、viewport extent、bounds、importance、occlusion confidence、Target／Quality、Runtime pressure snapshot、previous selectionを含む。

共通metric IDと用途を次に固定する。

| Metric | 用途 | 禁止用途 |
|---|---|---|
| `projected_error_px_q16` | Mesh／surface geometry detail | Simulation |
| `projected_coverage_px_q16` | Representation、VFX、Material presentation | Gameplay relevancy |
| `distance_mm_u64` | HLOD cell／manual visibility range、bounded surface | FOV／解像度依存のMesh品質判定 |
| `gameplay_relevance_q16` | Simulation tier | Rendering、occlusion |
| `budget_pressure_q16` | 同一fidelity内の候補選択 | Gameplay fidelity floorの緩和 |

Cooked thresholdは整数へ量子化し、CPU／GPUは同じ比較方向、境界包含、fixtureを使う。NaN、Inf、負値、非単調thresholdを受理しない。

選択順はminimum semantics、Target capability、error bound、importance、pressure policy、previous selection／hysteresis、Stable ID tie-breakでcanonicalにする。worker completion順、frame timeの単一sample、hash-map iteration順を使わない。

`LodTransitionRuleV1`は`from_tier`、`to_tier`、`metric_id`、`enter_threshold`、`exit_threshold`、`minimum_residency_units`、`transition_mode`、`transition_extent`、`camera_cut_policy`、`missing_artifact_policy`を持つ。enter／exitを別に持ち境界往復を防ぐ。Presentationのminimum residencyはreal frame数、Simulationはfixed tick数で、同じunitを共有しない。`transition_mode`は`instant | dither | cross_fade | domain_blend`のclosed enumとし、未対応BackendではProfile登録済みの意味同等fallbackを使う。camera cut、projection／dynamic extent変更では古いvisibility historyを捨て、そのframeに必要なdetailへ即時再選択し、無効historyを理由に低detailを選ばない。

## 5. Mesh／Sprite geometry LOD

Mesh／Sprite representationはgeometry artifact ref、bounds／silhouette error、vertex／primitive cost class、material interface、skin／morph compatibility、shadow／collision proxy relationを宣言する。LODはgeometry candidateを選び、meshlet／indirect draw／occlusionの実行は[Render Graph](render-graph.md)へ委譲する。

silhouette、UV、normal／tangent、skin weight、sprite pivot／pixel lockに意味差があるtierは明示する。missing material interfaceやanimation bindingをdefaultへ置換せず、compatible fallbackへ戻す。

## 6. Representation LODとHLOD

Representation LODはmesh、impostor、billboard、proxy、hiddenをclosed familyとして扱う。`hidden`はimportance／minimum semanticsが許可した場合だけ候補にし、capacity不足による無断消去を禁止する。

HLOD descriptorはmember Stable IDs、source World／Cell revision、aggregate bounds、artifact generation、material／lighting assumptions、transition boundary、member fallbackを持つ。HLODをEntity identity、Save owner、Physics／Navigation objectの代替にしない。

World streaming planはCellとHLOD artifactのresidency dependencyを記述し、LOD selectionはresident candidateから選ぶ。必要candidateが非residentの場合のrequest／backpressureは[World](world.md)とRuntime ownerへ返し、unbounded blocking loadを起こさない。

## 7. Simulation LOD境界

Simulation representationはfull、reduced、dormant等のDomain-defined behavior candidate refとsemantic guaranteeを公開できる。LODは候補を選択するが、Physics integration、Collision response、Navigation query、authoritative writer、wake conditionの意味は各Simulation Ownerが所有する。

render tierとsimulation tierを同一indexで結ばない。off-screenでもauthoritative gameplayに必要なsimulationを停止せず、visibleでもcapacity／Targetが許さないsimulation featureを暗黙有効化しない。tier handoffはpublished state、generation、handoff resultを持つ。

## 8. Animation、Material、VFXとの境界

Animationはclip／pose／skin candidateとminimum event／root-motion semanticsを公開し、LODはrepresentationを選ぶ。event、root motion、IK、retargetの意味は[Animation](../05-simulation/animation.md)を参照し、低tierへの遷移でauthoritative eventを欠落させない。

Materialは各tierのcompatible artifactとfeature reductionを[Materials](materials.md)で宣言する。LODはtierを選ぶがshader variantやparameter意味を再定義しない。

VFX候補は将来のVFX Ownerがsimulation／render representationとemission／lifetime guaranteeを宣言するまでDomain opaque refとして扱う。本書は先回りしてVFX schema、budget、execution phaseを定義しない。

## 9. Terrain、Foliage、Water／Surface、residency

Terrain、foliage、water、snow／surfaceはDomain Ownerがtile／patch／cluster representation候補、seam constraint、interaction guaranteeを公開し、LODは選択だけを行う。隣接tierのseam、crack、normal／material continuity、interaction proxyの互換性をdescriptorで検証する。

residency requestはartifact generation、priority intent、deadline class、fallback candidateを持つが、queue、memory reservation、backpressure値を本書で決めない。候補のresidencyが失われた場合はgenerationを再検証し、stale GPU／streaming handleを再利用しない。

## 10. Preset、AI contract、Editor UX

LOD Presetはquality intent、importance mapping、error policy、transition policy、Domain overrideをProject-owned assetとして保持する。Preset名から数値やtierを推測せず、resolved policyをPreviewで表示する。

LOD operationはcreate／update policy、bind representation、apply preset、set importance、preview selection、explain transition、validate closureをDomain actionとして登録する。共通Discovery／Preview／Apply、ChangeSet、approvalは[Executable contracts](../02-foundation/executable-contracts.md)、[Project state](../03-authoring/project-state.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)を使う。

Editorはview／Target／Quality／pressure fixtureを切り替え、各subjectのcandidate、selected tier、projected error class、hysteresis state、missing dependency、fallback理由を表示する。Preview selectionをRuntime authoritative stateやProduction qualificationと混同しない。

## 11. Runtime、determinism、telemetry境界

LOD evaluationの実行slot、snapshot、writer、job dependency、handoff lifetimeは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)を参照し、phase表やtick frequencyを複写しない。selectionは同一input snapshotでdeterministicにし、Replayにはinput identityとselection／reasonをDomain projectionとして供給する。

LOD固有telemetryはcandidate／selected tier、transition、thrash、missing／nonresident representation、fallback reasonを公開する。共通frame、memory、queue、capacity、regression測定とEvidence envelopeはRuntime／Governance ownerへ委譲する。

## 12. Validation、failure、qualification

Diagnosticはsubject／policy／representation Stable ID、View／Target、previous／requested／selected tier、error／importance／pressure class、error code、remediationを含む。少なくともinvalid policy、missing candidate、dependency mismatch、unsupported capability、stale generation、transition incompatible、nonresident、selection oscillationを区別する。

| Condition | 結果 |
|---|---|
| 非単調level／threshold、NaN／Inf、unit不一致 | ChangeSet／Cook拒否 |
| fallback closure欠落 | Package promotion拒否 |
| interactive／mutable objectのHLOD混入 | HLOD Cook拒否 |
| generated meshのvisual／deformation error超過 | candidate破棄、Source chain／LOD0維持 |
| GPU selector容量超過／Backend fault | 次frameからCPU direct fallback、Diagnostic |
| LOD Artifact未resident | 同generationのresident fallback、miss記録 |
| simulation wake event欠落／queue overflow | hard failure。eventをdropしない |
| Reference simulation不一致 | Simulation LOD非Promotion |
| critical VFX cue floor未達 | Policy非Promotion |
| Target capacity未達 | Source維持、`OptimizationRequired` |

Diagnostic IDを`MIRAKAN-LOD-SCHEMA_UNKNOWN | MIRAKAN-LOD-SEMANTIC_ROLE_CONFLICT | MIRAKAN-LOD-NON_MONOTONIC | MIRAKAN-LOD-MISSING_FALLBACK | MIRAKAN-LOD-UNSUPPORTED_TRANSITION | MIRAKAN-LOD-GENERATION_ERROR_LIMIT | MIRAKAN-LOD-HLOD_INTERACTIVE_SOURCE | MIRAKAN-LOD-SIMULATION_VISIBILITY_INPUT | MIRAKAN-LOD-SIMULATION_EQUIVALENCE | MIRAKAN-LOD-CRITICAL_CUE_FLOOR | MIRAKAN-LOD-RESIDENCY_MISS | MIRAKAN-LOD-CAPACITY_EXCEEDED | MIRAKAN-LOD-TARGET_UNQUALIFIED`に固定する。

Qualificationは次のDomain fixtureを持つ。

- Domain schemaからgenerated C++／TypeScript／binary descriptor／MCP projectionが同じclosed field／enumを表し、unknown field／enum／majorを拒否する。projection mechanicsはFoundation ownerを使う。
- unit、range、monotonic、fallback closure、Preset version、policy lockのpositive／negative fixture。Human、AI、headless CLIが同じIntentから同じPolicy／Plan hashへ収束する。
- FOV、orthographic／perspective、resolution、dynamic extent、camera cut、split Editor Viewのprojected-error golden値。
- CPU direct／GPU indirectのtier、境界包含、hysteresis一致とsilhouette、normal、UV seam、vertex color、material interface、skinning、morph、shadow error fixture。
- camera path往復時のthrash、pop、cross-fade overdraw、residency miss。
- Source順序を変えてもHLOD cluster／Artifact hash／proxy boundsが一致し、interactive／mutable Physics／Save／animation objectを拒否する。
- cell境界、teleport、camera cut、Artifact promotion、memory pressureの同時発生と、HLOD on／offでCollision、Nav、Damage、Save、Replay結果が一致すること。
- Full referenceと各Production simulation tierのReplay hash、最終count、Damage、goal、pending event一致。enter／exit、minimum residency、wake trigger同時発生、event queue上限、Save／Load直後を検証する。
- camera、frustum、occlusion、VFX visibility、Target／Quality Profileを変えてもauthoritative simulation結果が一致すること。
- Animation required bone、root motion、hitbox、event timing、VFX critical cue floor、pixel artのpoint sampling／integer scale／palette／pixel-locked、Terrain／Foliage／WaterのCollision／Nav／CPU Query非逆入力。
- camera移動、LOD／HLOD遷移、spawn burst、Physics／Navigation／Animation、敵味方VFX、streaming、Asset promotionを同一runで発生させるIntegrated fixture。run条件とEvidence transportはRuntime／Governance ownerを使う。

Evidence envelopeとgradingは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を参照する。distance-only selection、tier index coupling、silent hide、authoritative behavior loss、phase／budget複写が残る実装はRelease候補にしない。Capability maturityと導入順は[Product Plan](../00-product/product-plan.md)だけが所有する。
