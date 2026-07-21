# Miraikanai Engine Collision Contract

- 文書ID: mirakan.arch.simulation-collision
- 状態: review
- 正本範囲: 2D／3D geometry、Collider Source／Cooked Asset、Collision Material／Filter／Sensor、query request／result、contact／trigger／hit event semantics
- 非正本範囲: Body dynamics、solver、joint、character motor、Runtime phase／tick／lifetime、共通capacity／backpressure、Asset transaction、AI authorization。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Math core](../02-foundation/math-core.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Gameplay programming model](../03-authoring/gameplay-programming-model.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Physics](physics.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論と所有境界

Collisionは「何に当たるか、どう絞り込むか、何を返すか」だけを所有する。2D／3D shape、Collider Asset、material、filter、sensor、query、normalized eventはEngine-owned contractであり、native solverの型、pointer、callback、subshape IDを公開しない。

Bodyのmotion authority、mass、force、constraint、character motion、World stepは[Physics](physics.md)が所有する。CollisionはPhysicsを進行させず、Runtimeのcanonical execution slotを再定義しない。Asset import／reimport／promotionは[Asset lifecycle](../03-authoring/asset-lifecycle.md)、command／eventのmergeとleaseは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、共通上限とoverflow policyは[Runtime performance／capacity](../04-runtime/performance-capacity.md)を消費する。

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

`CollisionMaterialV1`はfriction、restitution、density、surface tag、combine semanticsをEngine語彙で保持する。Vendor material objectやcombine callbackは公開しない。Material変更はAsset version変更であり、live native shapeを外部からmutateしない。

`CollisionFilterProfileV1`はclosed channel ID、include／exclude relation、query visibility、event subscription classを持つ。pair許可は両側Profileから対称に解決し、片側だけが許可するconfigurationをinvalidとする。channel名はauthoring labelであり、Runtime dispatch keyにはstable contract IDを使う。

Filterの決定順は次の意味を持つが、Runtime phase順ではない。

1. 同一World／dimensionとactive generationを検査する。
2. hard excludeを適用する。
3. pair relationから`ignore | overlap | block`を解決する。
4. sensorならsolver responseを無効にし、overlap semanticsだけを残す。
5. event subscriptionとquery visibilityを別々に評価する。

Sensorはmass、force、jointを持たず、overlap eventのsourceとなるCollider shape propertyである。SensorをCCD保証やauthoritative hitの代用にしない。Gameplay volume、hitbox、camera obstructionは用途別Profileを使い、hard-coded channel分岐をProject C++へ散在させない。

## 4. Collision query contract

`CollisionQueryRequestV1`はWorld／scene version、dimension、geometry、transform、filter profile ref、result mode、caller-provided result bound、producer metadataを持つ。query kindは次のclosed setである。

- ray cast
- point／shape overlap
- shape cast
- closest point
- broad bounds query

`result mode`は`closest | any | all`である。`all`はcallerが有限のresult boundを渡す。実hitがboundを超えた場合はpartial successを返さず、typed failureにする。QueryはWorldやBodyを変更しない。

`CollisionQueryResultV1`はrequest ID、observed scene version、status、normalized hitのordered listを持つ。hitはEngine body handle／generation、Entity Stable ID、Collider Asset version、shape slot、subshape index、fraction／distance、position、normal、material／surface tagを持つ。native face、polygon reference、pointer、status bitsは含めない。

結果順は`fraction or distance, packed engine body handle, shape slot, subshape index, point lexicographic`で固定する。同値判定は[Math core](../02-foundation/math-core.md)のquantization／comparison契約を使用する。Queryが観測したscene versionとconsumerが要求したversionが一致しない場合は`stale`として破棄し、現Worldに再解決しない。

非同期queryのacceptance、deadline、lease、consume slotは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)の正本に従う。Collision文書は`T20_AsyncIntegrate`等のcanonical identifierを参照するだけで、新しいphaseや別順序を定義しない。

## 5. Contact、Trigger、Hit event

| event | 意味 |
|---|---|
| `ContactBeginV1` | solver contactが新しく成立した |
| `ContactPersistV1` | 前のsimulation stepからcontactが継続した |
| `ContactEndV1` | solver contactが終了した |
| `TriggerEnterV1` | Sensor overlapが新しく成立した |
| `TriggerExitV1` | Sensor overlapが終了した |
| `CollisionHitV1` | Profileが定めるhit semanticsを満たした |
| `ColliderRemovedV1` | Body／Collider generationがboundaryでretireした |

共通payloadはtick ref、ordered body pair、両Entity Stable ID、generation、Collider Asset version、shape slot、material／surface tag、event kindを持つ。contact manifoldを持つeventではposition、AからBを向くunit normal、separation、normalized approach speedを格納する。solver impulseはBackend間で意味と観測時点が一致しないためGameplay eventへ含めず、non-authoritative telemetryへ分離する。

Body pairはpacked Engine handleの昇順でA／Bを決める。eventは`event kind, body A, body B, shape slot A, shape slot B, contact point`のcanonical keyで並べる。callback arrival順、native ID、worker completion順は使用しない。End／Exitはmanifoldを持たず、`separated | disabled | profile_changed | teleported`のreasonを持つ。destroyは通常End／Exitではなく`ColliderRemovedV1`で通知する。

Event subscriptionは`begin_end | persist | hit`を型付きで選ぶ。Sensorへ`persist`や`hit`を指定したconfigurationは拒否する。Runtime queue capacity、overflow、tick publish policyは[Runtime performance／capacity](../04-runtime/performance-capacity.md)の正本を消費し、Collision固有の共有queue値を定義しない。

Replayは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)が所有するcanonical `T100_ReplayCheckpoint`に、正規化済みcommand／event、Collider generation、event pair stateの参照を供給する。旧phase名や別のReplay ownerを設けない。

## 6. Authoring、diagnostic、qualification

EditorとAIは同じCollider Source Asset、Project ChangeSet、validator、preview、cook経路を使う。公開操作はgeometry／asset／material／filter／sensor／query profileのinspect、create、update、remove、validate、previewに限定する。Body dynamics、character、solverの操作は[Physics](physics.md)へ渡す。Risk分類、authorization、credential、commit可否は[AI Security／Approval](../01-governance/ai-security-approval.md)を参照し、本書で規則を複写しない。

必須diagnostic classは次である。

- invalid／degenerate geometry
- unsupported shape／motion combination
- asymmetric filter
- non-identity runtime scale
- cook failure／incompatible artifact
- stale query／result bound exceeded
- event normalization invariant violation
- consumed lease／generation mismatch

Qualificationは2D／3D primitive、compound、mesh／height-field cook、filter matrix、sensor enter／exit、query ordering、event ordering、asset swap、stale result、fuzz inputを含む。同じfixtureを全private Backendへ与え、Engine-owned resultとdiagnosticが一致することを検査する。Performanceの測定方法とcapacity promotionは[Runtime performance／capacity](../04-runtime/performance-capacity.md)、evidence envelopeは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を使用する。

次を採用しない。

- Vendor collision型、native pointer、callback、serializationのpublic API化
- Visual MeshやTile renderer dataのlive collision source化
- native callback内からのGameplay／AI／Audio／VFX呼出し
- partial query resultの成功扱い
- eventからPresentation結果をGameplayへ逆入力すること
