# Miraikanai Engine Collision Contract

- 文書ID: mirakan.arch.simulation-collision
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: 2D／3D geometry、Collider Source／Cooked Asset、Collision Material／Filter／Sensor、filter execution placement／early rejection eligibility、query request／result、contact／trigger／hit event semantics
- 非正本範囲: Body dynamics、solver、joint、character motor、Runtime phase／Simulation Advance／lifetime、共通capacity／backpressure、Asset transaction、AI authorization。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Math／Core Utilities](../02-foundation/math-core.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)
- 関連文書: [AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Math core](../02-foundation/math-core.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Gameplay programming model](../03-authoring/gameplay-programming-model.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Physics](physics.md)、[LOD](../06-rendering/lod.md)、[Virtualized／Continuous Geometry](../06-rendering/virtualized-continuous-geometry.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-26

## 1. 結論と所有境界

Collisionは「何に当たるか、どう絞り込むか、何を返すか」だけを所有する。2D／3D shape、Collider Asset、material、filter、sensor、query、normalized eventはEngine-owned contractであり、native solverの型、pointer、callback、subshape IDを公開しない。query resultのEngine handle、Asset lease、private Adapter allocationは[Memory／Pointers](../02-foundation/memory-pointers.md)のbindingを使い、Collision固有のversion／hit順／invalidate条件だけを本書が定める。

Bodyのmotion authority、mass、force、constraint、character motion、World stepは[Physics](physics.md)が所有する。CollisionはPhysicsを進行させず、Runtimeのcanonical execution slotを再定義しない。Asset import／reimport／promotionは[Asset lifecycle](../03-authoring/asset-lifecycle.md)、command／eventのmergeとleaseは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、共通上限とoverflow policyは[Runtime performance／capacity](../04-runtime/performance-capacity.md)を消費する。

render geometry LOD、HLOD、[virtualized geometry](../06-rendering/virtualized-continuous-geometry.md)のpage／micro-cluster／inner cut、hidden presentationはCollider Asset／shape slot／filter／query／eventを切り替えない。[LOD](../06-rendering/lod.md#7-simulation-lod境界)へCollision candidateを公開する場合は、Collision Ownerがexact `SimulationLodCandidateDescriptorV1`、query／contact／trigger semantic guarantee、retained state、wake condition、equivalence fixtureを独立して定義・Qualificationする。現行targetは`full`だけで、low-poly render mesh、shadow proxy、HLOD proxy、virtual pageまたはmicro-cluster geometryをCollider source、fallback、reduced candidateとして受理しない。

公開不変条件は次である。

- Source geometryはSI単位のfinite値で表し、cook時に一度だけ検証する。
- Runtime scaleはColliderへ暗黙適用せず、非identity scaleは新しいCooked Assetを要求する。
- queryとeventはEngine handle、Stable ID、shape slot、versionだけを公開する。
- native callback到着順、pointer値、worker indexを結果順へ使わない。
- invalid geometry、stale version、truncated resultを成功として扱わない。

## 2. GeometryとCollider Asset

2Dは`+X right, +Y up`、3Dは`+X right, +Y up, +Z third axis`、長さはmeter、角度はradianを正規表現とする。finiteでない値、zero axis、退化primitive、自己交差polygon、非正規rotationはcommitまたはcookを拒否し、clampして意味を変えない。座標値型と演算規則は[Math core](../02-foundation/math-core.md)を使う。

### 2.1 Source／Cooked model

| Object | 所有内容 | lifetime |
|---|---|---|
| `ColliderSourceAsset2DV1` | ordered shape source、local pose、material ref、filter ref、sensor flag | Authoring revision |
| `ColliderSourceAsset3DV1` | ordered shape source、local pose、material ref、filter ref、sensor flag | Authoring revision |
| `CookedColliderAssetV1` | validated immutable payload、shape-slot table、source／profile identity、diagnostic summary | Asset version lease |
| `ColliderInstanceRefV1` | Cooked Asset ref、instance generation、Physics body ref | Runtime World lifetime |
| `ColliderShapeSlotV1` | artifact内で安定なnon-zero slotとsource shape IDの対応 | artifact lifetime |

Source Assetは`StableId`でshapeを識別する。Cookerはcanonical source orderからartifact-localなnon-zero `shape_slot`を割り当て、Source IDとの対応表を保存する。slotを別artifact、Save identity、Project identityへ流用しない。共有型の構造とversioningは[Executable contracts](../02-foundation/executable-contracts.md)に従う。

CookはSource、Profile、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)が所有するlock identityからDerived Assetを生成する。生成物はimmutableで、activationは[Asset lifecycle](../03-authoring/asset-lifecycle.md)と[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)のcompatible boundaryだけで行う。旧Asset leaseはnative callbackのnormalizeとevent cache purgeが完了するまで保持する。generation交換を通常のTrigger Exit／Enterとして捏造せず、明示的なremoved eventと新generationの観測で再構築する。

### 2.2 2D shape set

2Dのtagged unionは次を持つ。

| kind | 必須意味 | 禁止 |
|---|---|---|
| `Circle2DV1` | positive radius、local center | zero／negative radius |
| `Capsule2DV1` | two endpoints、positive radius | coincident endpointsを暗黙circle化 |
| `Segment2DV1` | distinct endpoints、optional one-sided normal | zero length |
| `ConvexPolygon2DV1` | canonical winding、convex vertices | concave／self-intersecting source |
| `Chain2DV1` | ordered vertices、open／closed policy | dynamic bodyへの利用 |
| `TileCollider2DV1` | tile regionからcookしたcompound segments／polygons | tile renderer dataのruntime参照 |

Compoundはordered child shapeの集合であり、childごとにshape slot、material、filter、sensor semanticsを保つ。隣接tile edgeはcookで結合し、internal seamをpublic contactとして出さない。

### 2.3 3D shape set

3Dのtagged unionは次を持つ。

| kind | 必須意味 | 禁止 |
|---|---|---|
| `Box3DV1` | positive half extents、local pose | non-finite extent |
| `Sphere3DV1` | positive radius、local center | non-uniform runtime scale |
| `Capsule3DV1` | axis、half segment、radius | zero axis |
| `Cylinder3DV1` | axis、half height、radius | implicit axis correction |
| `ConvexHull3DV1` | validated convex point／plane representation | dynamic concave mesh代用 |
| `TriangleMesh3DV1` | cooked triangle stream、material slots、active-edge data | dynamic／kinematic bodyへの利用 |
| `HeightField3DV1` | sampled height、hole、material cell data | mutable runtime source |
| `Compound3DV1` | ordered immutable child shapes | native child pointer公開 |

Triangle meshとHeight Fieldはstatic geometry専用である。Visual Meshを直接Colliderとして参照せず、Asset cookがcanonical triangle stream、winding、material mapping、edge情報を固定する。Convex generationはcandidateをSource Assetへ書き戻す操作であり、検証なしにRuntimeへ投入しない。

## 3. Material、Filter、Sensor

`CollisionMaterialV1`は`friction`（finite 0～16）、`restitution`（finite 0～1）、positive finite `density`、surface tag、`friction_combine`、`restitution_combine`をEngine語彙で保持する。両combine Fieldのclosed setは`average | min | max | multiply`である。異なるMaterial間で指定が一致すればそのoperator、不一致ならMiraikanaiの優先順`multiply > max > min > average`で上位を一つ選ぶ。operatorはそれぞれ`(a+b)/2`、`min(a,b)`、`max(a,b)`、`a*b`で、計算後にfrictionを0～16、restitutionを0～1へclampし、[Math core](../02-foundation/math-core.md)のcanonical quantizationを一度だけ適用する。densityはpair combineしない。Vendor default、material object、combine callbackを公開せず、Backend結果が本規則と違う場合はAdapterがEngine値へ正規化する。Material変更はAsset version変更であり、live native shapeを外部からmutateしない。

`CollisionFilterProfileV1`はclosed channel ID、include／exclude relation、query visibility、event subscription classを持つ。pair許可は両側Profileから対称に解決し、片側だけが許可するconfigurationをinvalidとする。channel名はauthoring labelであり、Runtime dispatch keyにはstable contract IDを使う。

event subscription class `hit`はsolver contactからの派生である。`hit`を選ぶProfileはtyped hit ruleを必須とし、hit ruleは対象channel集合、最小approach speed閾値（SI単位。0は全contactを許可）、`first_contact_only`（trueなら同一body pair・shape slot pairの継続contactにつき成立時に高々一度だけ発火し、falseならcontact成立ごとに発火する）を持つ。hit判定はcontact manifoldのnormalized approach speedだけを入力とし、Backend固有の判定条件を追加しない。

Filterの決定順は次の意味を持つが、Runtime phase順ではない。

1. 同一World／dimensionとactive generationを検査する。
2. hard excludeを適用する。
3. pair relationから`ignore | overlap | block`を解決する。
4. sensorならsolver responseを無効にし、overlap semanticsだけを残す。
5. event subscriptionとquery visibilityを別々に評価する。

この意味順を保持した上で、private Backendの実行配置は、同じ判定を表現できる最も早いstageへ固定する。

1. Engine-owned channel matrix／broadphase layer maskで、candidate pair生成またはbroadphase交差前にrejectする。
2. 両Profileが必要なpair relationを、shape contact generation／narrow phase前のpair filterでrejectする。
3. group／shape filterは、1または2で表現できないimmutable shape属性だけに使う。
4. Contact／manifold listenerでのrejectは、contact作業後のlate filterとして記録し、early-filter改善へ算入しない。

declarative matrixで表現できないcustom filter callbackだけをprivate Adapter内の例外候補とし、thread-safe、deterministic、allocation／logging 0、Gameplay／AI／Asset／World mutation呼出し0をhard predicateにする。callbackはProfile semanticsを変更せず、Backend native ID、pointer、worker indexを判定に使わない。実行stageを変える候補は、input pair、最終`ignore | overlap | block`、sensor、event subscription、query visibilityがreferenceと一致する場合だけ適格である。

Sensorはmass、force、jointを持たず、overlap eventのsourceとなるCollider shape propertyである。SensorをCCD保証やauthoritative hitの代用にしない。Gameplay volume、hitbox、camera obstruction、Interaction Focusの対象発見（interaction）は用途別Profileを使い、hard-coded channel分岐をProject C++へ散在させない。interaction用途のSensor Profileはoverlapとversion付きQueryのsemanticsだけを提供し、solver responseとauthoritative hitを持たない。[Gameplay programming model](../03-authoring/gameplay-programming-model.md) §2.4の`spatial` Focus Queryだけが`InteractionDefinitionV1.space_binding.payload.query_shape_ref`経由で本Profileを参照し、Profile IDを直書きしない。`logical | ui` InteractionはCollision Sensor Profileを持たない。

## 4. Collision query contract

`CollisionQueryRequestV1`はWorld／scene version、dimension、geometry、transform、filter profile ref、result mode、caller-provided result bound、producer metadataを持つ。query kindは次のclosed setである。

- ray cast
- point／shape overlap
- shape cast
- closest point
- broad bounds query

`result mode`は`closest | any | all`である。`all`はcallerが有限のresult boundを渡す。実hitがboundを超えた場合はpartial successを返さず、typed failureにする。QueryはWorldやBodyを変更しない。query kind×result modeの有効組合せとkind別ordering第1 keyは次表で固定し、無効な組合せはtyped failureとして拒否する。

| query kind | 有効result mode | ordering第1 key |
|---|---|---|
| ray cast | closest／any／all | fraction |
| point／shape overlap | any／all | なし |
| shape cast | closest／any／all | fraction |
| closest point | closest | distance |
| broad bounds query | any／all | なし |

`CollisionQueryResultV1`はrequest ID、observed scene version、status、normalized hitのordered listを持つ。hitはEngine body handle／generation、Entity Stable ID、Collider Asset version、shape slot、subshape index、fraction（ray／shape cast）またはdistance（closest point）、position、normal、material／surface tagを持つ。overlapとbroad bounds queryのhitはfraction／distance fieldを持たない。native face、polygon reference、pointer、status bitsは含めない。

根拠: project-decision — Backend collectorのearly-outで最初に発見されたhitは、broadphase tree、worker、native ID、走査順によって変わり得る。事後sortは、収集されなかった候補を復元できない。

`all`の結果順は`kind別ordering第1 key（上表）, Entity Stable ID, shape slot, subshape index, point lexicographic`で固定する。`closest`はBackendが最初に返したhitをそのまま採用せず、最小fraction／distanceと同じ量子化bucketに入るtie候補を保持し、同じcanonical keyで一件を選ぶ。Backendがtie候補の列挙を保証できないquery／shape組合せはauthoritative `closest`をunsupportedとして拒否する。

`any`はperformance-orientedな存在確認だけに使用するnon-authoritative modeであり、どのhitを返すかのcross-run決定性を保証しない。`any`の結果をGameplay state変更、Save、Replay、network authority、AIのCommit判断、Qualification oracleへ使用しない。決定論的な存在確認が必要なcallerは有限bound付き`all`を要求し、canonical orderの先頭を選ぶ。bound超過は結果なしのtyped failureであり、`any`へfallbackしない。

packed engine body handleはsession-localでSaveへ保存されないため、ordering keyへ使わない。Entity Stable ID順のcanonical結果はSave／Load跨ぎ、Replay、cross-sessionで再現する。同値判定は[Math core](../02-foundation/math-core.md)のquantization／comparison契約を使用する。Queryが観測したscene versionとconsumerが要求したversionが一致しない場合は`stale`として破棄し、現Worldに再解決しない。

非同期queryのacceptance、deadline、lease、consume slotは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)の正本に従う。Collision文書は`T20_AsyncIntegrate`等のcanonical identifierを参照するだけで、新しいphaseや別順序を定義しない。

<a id="collision-capability-records"></a>

### 4.1 MCD Capability record closure

Collisionが公開するMCD Capabilityは次のexact二件である。これはProduct `capability.simulation.collision-2d | collision-3d` rowではなく、Target別Product Activation Bindingが参照するDomain Contractである。文書状態が`review`でmaterialized Contract setがない現在は設計候補であり、IDだけをactive refとして解決しない。

共通Envelopeは`mcd_version=1`、`kind=capability`、`version=1`、`status=active`、`owners=[owner.core.collision]`、`requirement_refs=[]`、`since_contract_set=1`、`supersedes=[]`である。Payloadは`maturity=C1`、`supported_targets=[target.android.mobile, target.apple.mobile, target.headless.host, target.windows.desktop, target.windows.editor]`、`conflicts=[]`、`authoring_types=[]`、`operations=[]`、`validators=[]`、`quality_profiles=[]`、`budgets=[]`、`examples=[]`、`ai_guidance=[]`を共通値とする。

| Capability ID | `title` | `description` | `required_capabilities[]` | `rationale_refs[]` | `tags[]` |
|---|---|---|---|---|---|
| `capability.simulation.collision_query` | `Collision query` | normalized、bounded、versioned Collision queryを提供する | `[]` | `[mirakan.arch.simulation-collision#4-collision-query-contract]` | `[collision, query]` |
| `capability.simulation.collision_response` | `Collision response semantics` | Filter、Sensor、Contact／Trigger／Hitのnormalized response semanticsを提供する。Body dynamics／solver authorityは含まない | `[]` | `[mirakan.arch.simulation-collision#3-materialfiltersensor]` | `[collision, response]` |

両recordの`failure_modes`は`[{diagnostic_code=MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED, fallback_id=fallback.capability.unavailable}]`である。Product Capability、Physics provider、Interaction Roleは同じContract set rootを持つexact RefとTarget Activation Bindingを使用し、`collision`等の未定義generic ID、dimension名、Product maturityからMCD Refを合成しない。

## 5. Contact、Trigger、Hit event

| event | 意味 |
|---|---|
| `ContactBeginV1` | solver contactが新しく成立した |
| `ContactPersistV1` | 前のsimulation stepからcontactが継続した |
| `ContactEndV1` | solver contactが終了した |
| `TriggerEnterV1` | Sensor overlapが新しく成立した |
| `TriggerExitV1` | Sensor overlapが終了した |
| `CollisionHitV1` | subscription class `hit`のProfileが持つhit rule（§3）を満たすsolver contactが成立した |
| `ColliderRemovedV1` | Body／Collider generationがboundaryでretireした |

共通payloadは`cadence_profile_ref`、`simulation_advance_interval_hash`、`advance_sequence`、ordered body pair、両Entity Stable ID、generation、Collider Asset version、shape slot、material／surface tag、event kindを持つ。Cadence三Fieldはcontactを生成したcanonical `SimulationAdvanceIntervalV1`の同値とbyte equalityにする。contact manifoldを持つeventではposition、AからBを向くunit normal、separation、normalized approach speedを格納する。solver impulseはBackend間で意味と観測時点が一致しないためGameplay eventへ含めず、non-authoritative telemetryへ分離する。

Body pairは両Entity Stable IDの昇順でA／Bを決め、packed Engine handleをpair順へ使わない。eventは`event kind, body A, body B, shape slot A, shape slot B, contact point`のcanonical keyで並べる。callback arrival順、native ID、worker completion順は使用しない。End／Exitはmanifoldを持たず、`separated | disabled | profile_changed | teleported`のreasonを持つ。destroyは通常End／Exitではなく`ColliderRemovedV1`で通知する。

Event subscriptionは`begin_end | persist | hit`を型付きで選ぶ。Sensorへ`persist`や`hit`を指定したconfigurationは拒否する。Runtime queue capacity、overflow、Simulation Advance publish policyは[Runtime performance／capacity](../04-runtime/performance-capacity.md)の正本を消費し、Collision固有の共有queue値を定義しない。

Replayは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)が所有するcanonical `T100_ReplayCheckpoint`に、正規化済みcommand／event、Collider generation、event pair stateの参照を供給する。旧phase名や別のReplay ownerを設けない。

## 6. Authoring、diagnostic、qualification

EditorとAIは同じCollider Source Asset、Project ChangeSet、validator、preview、cook経路を使う。geometry／asset／material／filter／sensor／query profileのinspect、create、update、remove、validate、previewは将来のsemantic action vocabularyであり、Stable Operation IDでもcurrent公開操作でもない。本書のcurrent MCD／Owner Manifest／Service allowlist／Provider／MCP Operation集合は空で、action名からIDを生成しない。将来work item `activation.collision.authoring_operations.v1`が採用するexact ID集合と完全なMCD／Service／Policy／Validator／Diagnostic／Receipt／publication closureを一transactionで登録するまで、authoring Operation要求は`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`でSource不変として拒否する。Body dynamics、character、solverのactionは[Physics](physics.md)へ意味上handoffするだけで、Physics Operationを暗黙生成しない。Risk分類、authorization、credential、commit可否は[AI Security／Approval](../01-governance/ai-security-approval.md)を参照し、本書で規則を複写しない。

filter最適化のAI／Editor説明は[Runtime performance／capacity §8.4](../04-runtime/performance-capacity.md#84-algorithm-optimization-candidate-qualification)の`OptimizationDecisionProjectionV1`をread-onlyで消費する。raw native callback、pair buffer、Backend object、全traceを公開せず、AIはfilter stage、Profile、threshold、candidate selection、Receiptを変更しない。

必須diagnostic classは次である。

- invalid／degenerate geometry
- unsupported shape／motion combination
- asymmetric filter
- non-identity runtime scale
- cook failure／incompatible artifact
- stale query／result bound exceeded
- event normalization invariant violation
- consumed lease／generation mismatch

Qualificationは2D／3D primitive、compound、mesh／height-field cook、filter matrix、sensor enter／exit、query ordering、event ordering、asset swap、stale result、fuzz inputを含む。query fixtureは、候補列挙順を変えても`all`とauthoritative `closest`が同じ結果になること、equal fraction／distanceのtie-break、bound超過、`any`をauthoritative consumerへ接続した場合の拒否を含む。同じfixtureを全private Backendへ与え、Engine-owned resultとdiagnosticが一致することを検査する。filter fixtureはinput bounds、candidate pair、broadphase reject、pair-filter reject、shape-filter reject、late-listener reject、narrow-phase／contact成立数をstage別に記録し、early stageへ移した候補でも最終pair relation、sensor、query、eventのsemantic hashをreferenceと一致させる。asymmetric matrix、callback allocation／nondeterminism、late rejectをearly rejectへ誤計上するcaseを一原因ずつ拒否する。Performanceの測定方法とcapacity promotionは[Runtime performance／capacity](../04-runtime/performance-capacity.md)、evidence envelopeは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を使用する。

次を採用しない。

- Vendor collision型、native pointer、callback、serializationのpublic API化
- Visual MeshやTile renderer dataのlive collision source化
- native callback内からのGameplay／AI／Audio／VFX呼出し
- partial query resultの成功扱い
- eventからPresentation結果をGameplayへ逆入力すること
