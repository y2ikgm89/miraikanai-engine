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

LOD0という名前を必ず最高品質と仮定せず、tierにはexplicit quality rankとstable IDを持たせる。distanceのみ、asset filename、array index、Backend feature名を選択の正本にしない。

## 3. LOD policyとrepresentation contract

`LODPolicy`はsubject／group scope、quality intent、importance class、error metric、enter／exit boundary、minimum／maximum tier、transition policy、pressure response、Target constraint、fallback、qualification refを持つ。

`RepresentationDescriptor`はStable ID、Domain kind、source identity、artifact generation ref、quality rank、geometric／semantic error bound、required capability、estimated cost class、dependency closure、transition compatibility、fallback targetを持つ。costの共通測定単位、budget、capacity envelopeは[Runtime performance／capacity](../04-runtime/performance-capacity.md)を参照する。

候補setは同じsource semanticsとversion closureに属し、missing tier、duplicate rank、fallback cycle、incompatible dependency、stale artifactを拒否する。低tierがgameplay collision、navigation reachability、authoritative identityを暗黙に変えない。

## 4. 共通選択契約

Resolver inputはsubject Stable ID、representation set generation、ViewFamily／view role、projection、viewport extent、bounds、importance、occlusion confidence、Target／Quality、Runtime pressure snapshot、previous selectionを含む。

projected errorは[Math／Core utilities](../02-foundation/math-core.md)の座標／projection意味を使い、CPU／GPU／Editor Previewで同じquantized comparisonを行う。非finite値、invalid bounds、projection欠損では低品質へ黙って落とさずconservative tierまたは明示errorを返す。

選択順はminimum semantics、Target capability、error bound、importance、pressure policy、previous selection／hysteresis、Stable ID tie-breakでcanonicalにする。worker completion順、frame timeの単一sample、hash-map iteration順を使わない。

enter／exit境界はdead bandを持ち、boundary付近のthrashを防ぐ。camera cut、teleport、projection／extent changeではselectionを再評価するが、historyを無条件に最低tierへ落とさない。

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

Qualificationはprojection／extent別selection、CPU／GPU resolver equivalence、hysteresis、camera cut、Target fallback、HLOD membership、Simulation handoff、Animation event guarantee、Material compatibility、residency lossをfixtureで検証する。

Evidence envelopeとgradingは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を参照する。distance-only selection、tier index coupling、silent hide、authoritative behavior loss、phase／budget複写が残る実装はRelease候補にしない。Capability maturityと導入順は[Product Plan](../00-product/product-plan.md)だけが所有する。
