# Miraikanai Engine Collision／Colliderアーキテクチャ規約

- 文書版: 1.5
- 作成日: 2026-07-19
- 最終更新日: 2026-07-20
- 対象: 2D／3D Collision、Rigid Body接続、Collision Query、Editor、AI Authoring、Asset Cook
- 状態: プロジェクト公式の規範設計レビュー版
- 上位文書: [AIネイティブ独自ゲームエンジン 設計計画書](./2026-07-18-ai-native-game-engine-authoring-design.md)
- 基盤規約: [Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)
- C++公開境界: [Miraikanai Engine C++23・Named Modules・`import std`移行規約](./2026-07-20-cpp23-modules-import-std-transition-design.md)
- Runtime規約: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- Physics Engine規約: [Miraikanai Engine 独自Physics Platform／Dynamicsアーキテクチャ規約](./2026-07-20-physics-engine-architecture-design.md)
- Simulation連携規約: [Miraikanai Engine Physics／Navigation／Animation連携規約](./2026-07-19-physics-navigation-animation-architecture-design.md)
- Game実装規約: [Miraikanai Engine C++実行コード・構造化ゲームデータ規約](./2026-07-19-cpp-structured-game-data-design.md)
- 機能範囲: [Miraikanai Engine 2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)
- モバイル規約: [Miraikanai Engine モバイルPlatformアーキテクチャ規約](./2026-07-19-mobile-platform-architecture-design.md)
- AI実装・保守規約: [Miraikanai Engine AI実装・保守ガバナンス規約](./2026-07-19-ai-engine-development-governance-design.md)
- 実行可能契約規約: [Miraikanai Engine 実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)
- AI検証規約: [Miraikanai Engine AI検証・評価・来歴規約](./2026-07-19-ai-verification-evaluation-provenance-design.md)
- AI意味契約: [Miraikanai Engine Physics AI Semantic Capability Catalog規約](./2026-07-20-physics-ai-semantic-capability-catalog-design.md)

## 1. 結論

CollisionはPhysicsの付属機能ではなく、Gameplay、Character、Camera、Navigation、Animation、VFX、Audio、Editor、AI Authoringを接続するFirst-class Capabilityとする。

Miraikanai Engineは、AI、`GameplayDefinition`、人間、`NativeGameModule`（Project C++）へBox2D／Joltのbody、shape、pointer、callbackを公開しない。公開するのはEngine-ownedのCollider Asset、Physics Body Component、Collision Filter、Query、Event、Typed Command、Validator、Previewだけである。2Dの検出と応答にはBox2D 3.1.1、3DにはJolt Physics 5.6.0をprivate Adapter候補として採用し、独自Physics Platform規約のTarget別Qualification合格後だけProduction利用する。次はMiraikanai Engineが独自に所有する。

- 正規Authoring dataとRuntime contract
- Stable ID、generation、Asset version、shape slot
- Collision channel／filterの意味
- Query結果とContact／Trigger eventの正規化
- Cook、validation、hot reload、failure policy
- EditorのCollider編集、可視化、Profiler
- AI用Capability、Operation、質問、Diff、承認
- `GameplayDefinition`へ許可する型付きOperationと`NativeGameModule`向けPort
- 性能、memory、mobile、test、release gate

「独自Collision system」はsolver数式を一から再実装する意味ではない。Engine固有の制作・検証・実行契約を所有し、vendor kernelを交換してもGameSpec、Project source、AI Tool、Saveがvendor APIへ依存しない状態を意味する。

## 2. 決定権、規範語、対象外

### 2.1 文書ごとの決定権

| 主題 | 正本 |
|---|---|
| Collider、shape、material、filter、query、event、cook、Editor／AI操作 | 本書 |
| Physics World、Solver、Dynamics、Joint／Constraint、Character Motor、Backend build、Save／Replay | 独自Physics Platform規約 |
| Physics phase、event配送時点、handle寿命、queue overflow、global memory配分 | Runtime規約 |
| C++所有権、pointer、module、directory、vendor version | 基盤規約 |
| GameplayDefinition、NativeGameModule、AI実装選択 | Game実装規約 |
| 2D／3D Capability成熟度とGenre package | 2D／3D機能計画 |
| Android／Apple絶対budget、lifecycle、thermal | モバイル規約 |
| MCD、Schema projection、Codegen | 実行可能契約規約 |
| AI Risk、Approval、Source sandbox | AI実装・保守規約 |
| Physics／Collision自然言語Intent、canonical role、意味解決、質問、Assumption、Semantic Eval | Physics AI Semantic Capability Catalog規約 |

本書はRuntime規約のphase、queue capacity、memory上限を緩和しない。本書と既存文書のCollision意味論が矛盾する場合は本書を優先し、同一ChangeSetで参照元を修正する。

本文では「必須」をValidator／CIで拒否する規則、「禁止」を実装へ導入しない規則、「推奨」を例外時にADRと計測を要求する規則として用いる。

### 2.2 C1／C2の対象外

次はCollision Coreへ暗黙に含めない。

- 2D shapeと3D shapeの直接衝突
- cross-platform bitwise lockstep
- Runtime AIによる任意triangle mesh／convex decomposition生成
- Dynamic body上のtriangle mesh／heightfield
- GPU rigid-body collision、GPU hair、GPU soft body
- cloth、fluid、fracture solver、voxel destruction
- multiplayer lag compensation、server rewind、client prediction
- arbitrary user callbackによるcontact solver変更

Soft body、GPU simulation、network collision、large-world floating originはC3 Research Capabilityであり、専用ADR、threat model、memory／determinism fixtureが承認されるまでMCD Catalogへ公開しない。

## 3. 設計原則

1. **BodyとColliderを分離する。** Bodyは運動と質量、Colliderは形状、filter、sensor、materialを所有する。
2. **SourceとCookedを分離する。** SceneやAIが編集するSource Assetと、Runtimeが読むimmutable Cooked Assetを同一objectにしない。
3. **2Dと3Dを混ぜない。** 共通概念は共有するが、shape、座標、mass、backend、query型を別にする。
4. **単位を固定する。** 距離はmeter、角度はradian、時間はsecond、質量はkg。2D密度はkg/m²、3D密度はkg/m³とする。
5. **ScaleをRuntimeへ持ち込まない。** Collider geometryへscaleをCook時にbakeし、Runtime Collider transformはtranslation＋rotationだけとする。
6. **有限値だけを受理する。** NaN、Inf、非正規quaternion、退化形状、範囲外値をfallbackせず拒否する。
7. **QueryとSimulationを分ける。** QueryはWorldを変更せず、結果順序をEngineが正規化する。
8. **TriggerとBlockを分ける。** Sensorはoverlap eventだけを生成し、solver responseとmassへ寄与しない。
9. **Callbackを公開しない。** Native callbackはpreallocated bufferへ値をcopyするだけで、World、Gameplay Logic、AIを呼ばない。
10. **失敗を成功に見せない。** Cook、capacity、queue、invariantの失敗時はlast valid revisionを維持し、部分的なPhysics stateをpublishしない。

## 4. 座標、scale、数値範囲

### 4.1 Coordinate contract

| Space | 正規座標 |
|---|---|
| 2D | 右手系のXY平面、+X right、+Y up、rotation正方向は+Z軸周り、meter／radian |
| 3D | 右手系、+X right、+Y up、+Zは座標系の第三軸、meter／radian |
| glTF import | glTF 2.0の右手系、+Y up、meterをImporter境界で保持 |
| Pixel Asset | `pixels_per_unit`でtexelからmeterへ一度だけ変換 |

3Dの「前方」はCamera、Character、Asset profileが明示し、Collision座標系から推測しない。Authoring quaternionは`(x, y, z, w)`、長さ`[0.99999, 1.00001]`を受理し、Cook時に一度正規化する。長さ0または範囲外を単位quaternionへ黙って置換しない。

### 4.2 `PhysicsScaleProfile::ReferenceV1`

| Field | C1公式値 |
|---|---:|
| `hard_min_full_extent_m` | 0.001 |
| `recommended_min_full_extent_m` | 0.02 |
| `recommended_max_full_extent_m` | 100 |
| `hard_max_full_extent_m` | 10,000 |
| `hard_max_abs_body_position_m` | 10,000 |
| `max_linear_speed_mps` | 1,000 |
| `max_angular_speed_radps` | 200 |

Hard範囲外はerror、recommended範囲外はwarningとReference fixture追加要求にする。10 kmを超えるWorld、1 mm未満のauthoritative Collider、1,000 m/s超のbodyはC3 large／extreme-scale ADRなしに許可しない。Teleportは速度ではなく`TeleportBody` commandで表現し、sweepやCCDを実行したことにしない。

Velocity／Impulse／Kinematic Target commandがhard speedを要求した場合は適用前に拒否する。Backend stepの結果がhard speedまたはfinite invariantを破った場合はclampして継続せず、`MIRA-COLLISION-NATIVE_INVARIANT`としてtickをpublishしない。Box2D world maximum linear speedとJolt motion propertyには1,000 m/sを明示設定し、未記録のvendor defaultを使わない。

Visual Transformのscale変更はColliderへ暗黙反映しない。Editorは`Regenerate Collider` Diffを提示し、承認後にgeometry寸法を更新して再Cookする。負scaleはImporterが頂点とwindingへbakeできるStatic sourceだけ許可し、Runtimeでは拒否する。

`TeleportBody`はT30 commandとして提出し、T40で通常のmotion intentより先に適用する。旧pairは`end_reason=teleported`で終了し、新poseのpairは同tickのstep結果からBegin／Enterを生成する。Teleportはsweep、CCD、途中経路のTriggerを生成しない。途中経路がGameplayへ必要ならTeleportを使わず、Shape Cast結果を検証してからkinematic targetまたは通常移動を提出する。

## 5. 正規Data Model

### 5.1 Asset、Component、Runtime object

```text
CollisionMaterial2DAsset ─┐
CollisionFilterProfile ───┼─> Collider2DSourceAsset ─Cook─> Collider2DCookedAsset
Shape2DAuthoring ─────────┘                                  │
PhysicsBody2DComponent ──────────────────────────────────────┘

CollisionMaterial3DAsset ─┐
CollisionFilterProfile ───┼─> Collider3DSourceAsset ─Cook─> Collider3DCookedAsset
Shape3DAuthoring ─────────┘                                  │
PhysicsBody3DComponent ──────────────────────────────────────┘
```

- Source AssetはStable ID、schema version、revision、provenanceを持つ。
- Cooked AssetはSource revision、Target Profile、backend ABI version、Cook profile、content hashを持つimmutable Derived Assetである。
- Runtime bodyは`PhysicsBodyHandle { index32, generation32 }`で参照する。
- Native `b2BodyId`、`b2ShapeId`、Jolt `BodyID`、`Shape*`をComponent、Event、Save、GameplayDefinitionへ保存しない。
- 一つのEntityへauthoritative `PhysicsBody2DComponent`と`PhysicsBody3DComponent`を同時付与しない。
- 一つのBodyは一つのCooked Collider Asset versionを参照する。CompoundはAsset内の複数shapeで表現する。
- Colliderを持たないBodyはEditor previewだけ許可し、Play／Cookを拒否する。

### 5.2 `PhysicsBody2DComponent`

| Field | 型／範囲 | 規則 |
|---|---|---|
| `body_kind` | `static \| kinematic \| dynamic` | 必須 |
| `collider_asset_id` | Stable `AssetId<Collider2D>` | Runtime compileでversionへ解決 |
| `position_m` | finite float2 | Authoring source |
| `rotation_rad` | finite float | `[-pi, pi)`へcanonicalize |
| `initial_linear_velocity_mps` | finite float2 | staticではzero |
| `initial_angular_velocity_radps` | finite float | staticではzero |
| `declared_peak_linear_speed_mps` | float `(0, 1,000]` | Authoring予測／CCD validation |
| `declared_peak_angular_speed_radps` | float `(0, 200]` | Authoring予測／fixture選択 |
| `gravity_scale` | float `[-16, 16]` | static／kinematicでは0へ固定せずfield自体を禁止 |
| `linear_damping_per_s` | float `[0, 100]` | dynamic／kinematic |
| `angular_damping_per_s` | float `[0, 100]` | dynamic／kinematic |
| `mass_mode` | `from_shapes \| override` | dynamicだけ |
| `mass_override_kg` | float `(0, 1e9]` | override時だけ |
| `inertia_override_kg_m2` | float `(0, 1e12]` | override時だけ |
| `fixed_rotation` | bool | dynamic／kinematic |
| `sleep_policy` | `allow \| prevent \| start_sleeping` | dynamicだけ |
| `motion_quality` | `discrete \| bullet` | dynamicだけ |

### 5.3 `PhysicsBody3DComponent`

| Field | 型／範囲 | 規則 |
|---|---|---|
| `body_kind` | `static \| kinematic \| dynamic` | 必須 |
| `collider_asset_id` | Stable `AssetId<Collider3D>` | Runtime compileでversionへ解決 |
| `position_m` | finite float3 | Authoring source |
| `rotation_xyzw` | normalized float4 | 4.1節 |
| `initial_linear_velocity_mps` | finite float3 | staticではzero |
| `initial_angular_velocity_radps` | finite float3 | staticではzero |
| `declared_peak_linear_speed_mps` | float `(0, 1,000]` | Authoring予測／CCD validation |
| `declared_peak_angular_speed_radps` | float `(0, 200]` | Authoring予測／fixture選択 |
| `gravity_scale` | float `[-16, 16]` | dynamicだけ |
| `linear_damping_per_s` | float `[0, 100]` | dynamic／kinematic |
| `angular_damping_per_s` | float `[0, 100]` | dynamic／kinematic |
| `mass_mode` | `from_shapes \| override` | dynamicだけ |
| `mass_override_kg` | float `(0, 1e12]` | override時だけ |
| `inertia_override_kg_m2` | positive finite float3 | override時だけ |
| `allowed_dofs` | 6 bool | dynamic／kinematic。全falseは拒否してstaticを提案 |
| `sleep_policy` | `allow \| prevent \| start_sleeping` | dynamicだけ |
| `motion_quality` | `discrete \| linear_cast` | dynamicだけ |

`PhysicsBodyReferenceDefaultsV1`は、各body kindで存在するfieldについてinitial velocityをzero、declared peak linear speedを20 m/s、declared peak angular speedを20 rad/s、gravity scale 1、linear／angular dampingを0、mass modeを`from_shapes`、2D `fixed_rotation=false`、3D allowed DOFsを全true、sleepを`allow`、motion qualityを`discrete`とする。Body kind、Collider Asset、poseにはdefaultを設けない。Authoring Componentは採用値を全fieldへ展開し、Cooked Componentに「vendor defaultを使う」という欠落状態を残さない。Declared peakはValidatorとbudgetの入力であり、solver clamp値ではない。Runtime telemetryが連続60 tickで宣言値を超えた場合は`MIRA-COLLISION-DECLARED_MOTION_PROFILE_EXCEEDED` warningを記録する。

BodyのRuntime poseにscale fieldを設けない。`mass_mode=from_shapes`ではenabledかつnon-sensorのshapeだけからmassとinertiaを計算する。Sensor、triangle mesh、heightfieldはmassへ寄与しない。Dynamic bodyにmassへ寄与するshapeがなく、overrideもない場合は`MIRA-COLLISION-MASS_SOURCE_MISSING`で拒否する。

Runtime compilerは`collider_asset_id`を承認revisionの`AssetVersionHandle<Collider2D／3D>`へ解決し、Runtime Componentにはlogical IDとversion handleの両方を保存する。Simulation中にlogical IDを再resolveせず、T00 promotionまたはPlay restartだけでversionを変更する。

### 5.4 Collider Source Asset共通Envelope

| Field | 型／上限 |
|---|---|
| `asset_id` | Stable ID |
| `dimension` | `two_d \| three_d` |
| `schema_version` | positive integer |
| `source_revision` | monotonic uint64 |
| `shapes` | Stable IDを持つ非空array |
| `cook_profile_id` | version付きProfile参照 |
| `provenance` | 人間、AI、Importer、generator、source Asset、parameter、license |

各shapeは次を持つ。

| Field | 型／規則 |
|---|---|
| `shape_id` | Asset内で一意なStable ID |
| `enabled` | bool。Shipping Cookではenabled shapeが1個以上必要 |
| `local_position_m` | finite float2／float3 |
| `local_rotation` | 2D radian／3D normalized quaternion |
| `geometry` | dimension別tagged union |
| `material_asset_id` | 同じdimensionのMaterial |
| `filter_profile_id` | `CollisionFilterProfileV1` |
| `is_sensor` | bool |
| `event_mode` | `none`、または`begin_end`を含む`begin_end \| persist \| hit`の重複なしset |
| `semantic_role` | `solid \| hurtbox \| hitbox \| interaction \| volume \| camera_blocker \| custom` |

`semantic_role`はAI説明、検索、Validator presetの入力であり、runtime filterを置き換えない。`is_sensor=true`で`semantic_role=solid`、または`is_sensor=false`で`hurtbox／hitbox／interaction／volume`を指定した場合はerrorにする。`custom`はfilter、sensor、eventをすべて明示した場合だけ許可する。

Cookerはenabled shapeを`shape_id`のunsigned UTF-8 byte順に並べ、0始まりの`shape_slot:uint32`を割り当てる。EventとQueryは`Collider AssetVersionHandle + shape_slot`を返す。Source array順、native subshape ID、pointer順を公開順序に使用しない。

RuntimeはAssetを変更せず、Bodyごとにsparseな`ColliderRuntimeState`を持てる。C1で許可するoverrideは次の二つだけで、いずれもT00 commandとして適用する。

- Sensor shapeの`active`切替。Hitbox、Hurtbox、Interaction volumeの時間制御に使用する。
- Shapeの`filter_profile_id`を、PlayPreparingでCook済みの別Profileへ切り替える。

Solid shapeのactive切替、geometry、local pose、material、sensor flagの変更はmass、contact、shape topologyを変えるためC1 Runtimeでは禁止する。Sensorをinactiveにした時は保持中pairごとに`TriggerExitV1`を生成してからnative filterを無効化する。既存Profileへの切替はpair cacheを再評価し、次のT60で旧pairのEnd／Exitと新pairのBegin／Enterをcanonical順に配送する。Runtime state、選択Profile、active SensorはSave／Replay対象である。

## 6. 2D Shape

### 6.1 C1 shape set

| Shape | Authoring field | 制約 |
|---|---|---|
| `circle` | `radius_m` | radius `>= 0.0005` |
| `box` | `half_extents_m: float2`、`corner_radius_m` | 各half extent `>= 0.0005`、corner `>= 0`かつ`<= min(half_extents)` |
| `capsule` | `point_a_m`、`point_b_m`、`radius_m` | segment長 `>= 0.001`、radius `>= 0.0005` |
| `convex_polygon` | `vertices_m`、`corner_radius_m` | 3～8頂点、strict convex、非自己交差、面積正 |
| `segment` | `point_a_m`、`point_b_m` | 長さ `>= 0.001`、static bodyだけ |
| `chain` | `vertices_m`、`is_loop` | 4～65,536点、static bodyだけ、one-sided |

Box2D 3.1.1の`B2_MAX_POLYGON_VERTICES=8`をC1上限として固定する。Polygon頂点はSourceで反時計回りへcanonicalizeし、連続重複、collinearだけの集合、自己交差、concave入力を拒否する。C1でconcave polygonを一つのshapeとして黙ってconvex hullへ変更しない。複数のconvex shape、Tile Collider、または明示`GenerateConvexPieces2D` previewを使用する。

Chainは次を必須とする。

- Source順から決まる片面collisionをScene Viewへnormal矢印で表示する。
- 隣接点距離を0.001 m以上とする。
- 自己交差をCookerで検査し、Box2Dが検査しない場合もEngineが拒否する。
- open chainの先頭／末尾edgeにcollisionがないことをPreviewへ表示する。
- seamを接続する場合は終端側3点と開始側3点の一致をCookerが検証する。
- Chain segmentを個別shapeとして編集、破棄、永続参照しない。

### 6.2 2D CompoundとTile Collider

- 一つのdynamic／kinematic Collider Assetはenabled shape最大64、staticは最大65,535とし、Target全体のlive shape上限を別に検査する。
- 同じBody内のshape同士は衝突しない。
- Dynamic compoundの全shapeは合計AABB full extentが4.2節のhard範囲内でなければならない。
- Tilemapから生成するColliderはSource tile ID配列ではなく、region、tile collision tag、merge profile、source revisionを持つDerived Assetとする。
- Tile mergeは同一filter、material、sensor、one-way設定だけを統合する。異なるsemanticを一つのpolygonへ混ぜない。
- 変更regionの1 tile外側まで再Cookしてseamを検査し、staging version全体がvalidになるまで公開しない。

`GenerateConvexPieces2D`はC2 Capabilityである。C1では手動compoundとTile ColliderだけをProduction対応とし、自動分解結果をProductionへ昇格しない。

## 7. 3D Shape

### 7.1 C1 shape set

| Shape | Authoring field | 制約 |
|---|---|---|
| `box` | `half_extents_m: float3`、`convex_radius_m` | 各half extent `>= 0.0005` |
| `sphere` | `radius_m` | `>= 0.0005` |
| `capsule` | `half_segment_length_m`、`radius_m` | half segment `>= 0.0005`、radius `>= 0.0005`、local +Y軸 |
| `cylinder` | `half_height_m`、`radius_m`、`convex_radius_m` | half height／radius `>= 0.0005`、local +Y軸 |
| `convex_hull` | finite point array、`convex_radius_m` | Source 4～65,536点、Cook済みHull最大256頂点 |
| `triangle_mesh` | Mesh Asset subset | static bodyだけ |
| `heightfield` | square sample grid | static bodyだけ、sample side 4～16,384 |
| `compound` | Asset内shape array | childは本表のshape、compoundの再帰nestは禁止 |

Convex HullはSource pointからinterior／duplicate pointを除去して構築する。Cook後のHullが4非coplanar頂点未満、volumeが0、またはJolt 5.6.0の256 hull point上限を超える場合は拒否する。頂点を無通知で間引かない。`GenerateConvexHull3D`は結果のpoint数、volume差、AABB差、予測memoryをPreviewし、人間または事前委任されたR2 Policyが承認してからSource Assetを変更する。

`convex_radius_m`は0以上、対象shapeの最小full extentの25%以下を必須とする。Reference Generatorは`min(0.05 m, 0.10 * minimum_full_extent_m)`をSource fieldへ明示保存する。Joltが値を自動縮小した場合はCook成功にせず、`MIRA-COLLISION-CONVEX_RADIUS_OUT_OF_RANGE`としてSource修正を要求する。

Triangle Meshは次を必須とする。

- static body専用とし、kinematic／dynamic bodyでは`MIRA-COLLISION-DYNAMIC_TRIANGLE_MESH_FORBIDDEN`を返す。
- finite position、index範囲、3つの異なるindex、非退化triangleを検証する。
- Source windingを明示し、Cook後のsimulation triangleは片面とする。
- Queryごとのback-face modeを8.3節で明示する。
- Render Meshを暗黙にColliderへ流用しない。明示したMesh subset、Cook profile、生成provenanceを保存する。
- Open meshはStatic surfaceとして許可するが、inside／point containment判定へ使用しない。
- per-triangle materialは最大256 slotとし、Cook済みtriangleから`shape_slot`とmaterial slotを解決できる表を保持する。

Heightfieldは穴を`no_collision` sampleとして明示でき、Material slotは0～255とする。C1は`block_size=2`、`sample_side / block_size`が2以上の2冪を必須とする。Cooked sampleは8 bitから開始し、最大許容world-space誤差0.01 mを超える場合は必要bit数を16まで上げる。16 bitでも誤差を満たせないAssetは拒否する。sample side、block size、bit数、誤差をCooked metadataへ保存し、Runtimeで変更しない。

### 7.2 Scale、Mesh、Compound制約

- Non-uniform scaleはSource geometryへCook時にbakeし、normal、winding、mass propertyを再計算する。
- Runtime body／shape scaleは常にidentityである。
- Compound child数はdynamic／kinematicで最大64、staticで最大4,096とする。大規模Static Worldは複数streaming cellへ分割する。
- Dynamic compoundにtriangle mesh、heightfield、chain相当surfaceを含めない。
- Mutable CompoundをGameplay APIへ公開しない。形状追加／削除は新Asset versionのCookとして扱う。
- Tapered capsule、tapered cylinder、plane、mutable compoundはC2で個別Capability化するまでSchemaで拒否する。

## 8. Material、Filter、Sensor

### 8.1 Collision Material

`CollisionMaterial2DAsset`と`CollisionMaterial3DAsset`を分ける。共通fieldは次のとおり。

| Field | 範囲 | Reference default |
|---|---:|---:|
| `friction` | `[0, 4]` | 0.6 |
| `restitution` | `[0, 1]` | 0 |
| `surface_tag` | Stable enum ID | `surface.default` |
| `audio_surface_id` | optional Asset ID | none |
| `vfx_surface_id` | optional Asset ID | none |

2Dは`density_kg_m2`、3Dは`density_kg_m3`を持ち、範囲は`[0.001, 100,000]`、Reference defaultは2Dが1、3Dが1,000とする。Sensorではdensityを0としてAdapterへ渡し、Body mass計算から必ず除外する。

C1のMaterial combineはBackend defaultへ委ねず、次へ固定する。

```text
combined_friction    = sqrt(friction_a * friction_b)
combined_restitution = max(restitution_a, restitution_b)
```

結果はfinite検査し、frictionを`[0, 4]`、restitutionを`[0, 1]`へ入力段階で保証する。per-material combine mode、surface velocity、rolling resistance、anisotropic frictionはC2 Capabilityであり、C1 Assetへfieldを予約しない。

### 8.2 `CollisionFilterProfileV1`

Authoringでraw integer bitmaskを入力させない。Projectは最大32個の`CollisionChannel`を持ち、各Channelに厳密に一つの`CollisionFilterProfileV1`を対応させる。

`CollisionChannel`:

| Field | 型／規則 |
|---|---|
| `channel_id` | 0～31のStable numeric ID |
| `stable_name` | ASCII lower snake、Project内一意 |
| `display_name` | localizable |
| `query_visible` | bool |

`CollisionFilterProfileV1`:

| Field | 型／規則 |
|---|---|
| `profile_id` | Stable ID |
| `channel_id` | 対応するChannel ID |
| `collides_with` | channel IDの重複なしset |

Channel 0～7はEngine templateで次のstable nameを予約する。

| ID | Stable name |
|---:|---|
| 0 | `world_static` |
| 1 | `world_dynamic` |
| 2 | `character` |
| 3 | `vehicle` |
| 4 | `projectile` |
| 5 | `gameplay_sensor` |
| 6 | `destructible` |
| 7 | `debris` |

`CollisionChannelTable::ReferenceV1`は全8 Channelを`query_visible=true`とし、unordered pair `world_static–world_static`、`world_static–gameplay_sensor`、`projectile–projectile`の三組だけを非collision、残る33組をcollisionにする。Body kindとSensorによる最終動作は後掲の表を適用する。Genre templateはこのMatrixを変更できるが、32×32の対称Matrix全値とProfile hashをGameSpecへ保存し、未記録のReference default変更を既存Projectへ遡及適用しない。

Projectはdisplay nameとpair設定を変更できるが、IDとstable nameを別用途へ再利用しない。8～31をProject定義に使用する。Profile PとQのpairは、`P.collides_with`がQのchannelを含み、かつ`Q.collides_with`がPのchannelを含む場合だけsimulation候補になる。片側だけの指定は`MIRA-COLLISION-FILTER_ASYMMETRIC`でCommitを拒否し、Engineが自動で反対側を追加しない。異なるmaskが必要なGameplay stateは同じChannelのProfileを増やさず、8～31に`character_ghost`等の明示Channelを作り、Runtimeで既存Profileへ切り替える。

Box2D Adapterはchannelを`categoryBits`、`collides_with`を`maskBits`へcompileする。Jolt AdapterはPlayPreparingで使用中ProfileとmobilityからObject Layer tableを構築し、Object-vs-BroadPhase、Object-pair filterへcompileする。Jolt Object Layer番号はCooked Project Profile内部値であり、Save、Event、AIへ公開しない。

JointまたはConstraintの`collide_connected=false`はBody pair単位の明示overrideとして許可する。C1ではarbitrary group index、negative group、callback filterを公開しない。

Filter通過後のBody kind規則を両Backendで次へ統一する。

| Pair | Solid response／Contact | Sensor overlap |
|---|---|---|
| 同じBody内のshape | なし | なし |
| static–static | なし | 片方以上がSensorならあり |
| static–kinematic | なし | 片方以上がSensorならあり |
| kinematic–kinematic | なし | 片方以上がSensorならあり |
| dynamic–static | あり | 片方以上がSensorならoverlapだけ |
| dynamic–kinematic | あり | 片方以上がSensorならoverlapだけ |
| dynamic–dynamic | あり | 片方以上がSensorならoverlapだけ |

両方がSensorのpairもTriggerとして扱う。Solid kinematicをstatic／kinematicへ安全に沿わせる用途にはsolver contactを期待せず、Character Motorまたは明示Shape Castを使う。Jolt Adapterもこの表にないsolid pairをBackend固有設定で追加しない。

### 8.3 Sensor

- `is_sensor=true`のshapeはContact responseを生成しない。
- Sensorはmassとinertiaへ寄与しない。
- Sensor eventは`begin_end`を必須とし、`none`を拒否する。
- Sensorから`persist`を要求した場合、毎tickのStay eventではなく`QueryCurrentOverlaps`を使用する。C1ではSensor Stay eventを生成しない。
- SensorをPlay中にsolidへ、solidをsensorへ変更しない。新Collider generationとPlay restartを要求する。
- Sleepingだけを理由にTrigger Exitを生成しない。AdapterはEngine-owned overlap pair cacheを維持し、native removalがsleep由来ならoverlapを再検証する。
- Body／Collider破棄では無効native IDからTrigger Exitを作らず、`ColliderRemovedEvent`でpair cacheをpurgeする。
- Box2D Sensorにcontinuous collisionがないため、高速Triggerは9節のswept Shape Queryを使用する。`is_sensor=true`と`motion_quality=bullet`だけで高速overlapが保証されると表示しない。

## 9. Collision Query

### 9.1 Query種類

| Query | 2D | 3D | C1 |
|---|---|---|---|
| Ray cast | yes | yes | Production |
| Shape cast | circle／box／capsule／convex | sphere／box／capsule／convex | Production |
| Shape overlap | yes | yes | Production |
| Point containment | circle／box／capsule／convex | closed convexだけ | Production |
| AABB broad query | Editor／Profiler internal | Editor／Profiler internal | public Gameplay APIにはしない |
| Closest point／distance | primitive／convex | primitive／convex | C2 |

Triangle Mesh、Heightfield、Chainをcastするquery shapeとして使用しない。World側のtargetには使用できる。

`CollisionQueryProfileV1`はStable ID、dimension、include channel set、`include_sensors`、include body kind set、back-face mode、initial-overlap policy、result mode、max hitsを持つ。Authoring／GameplayDefinitionはProfileを参照し、Runtime requestはProfile ID、Profile hash、上記のCook済み全値を持つ。Profile更新を実行中Requestへ遡及適用しない。

`CameraCollisionQueryProfile::ReferenceV1`は`include_channels={world_static, world_dynamic, vehicle, destructible}`、`include_sensors=false`、全Body kind、`front_only`、`initial_overlap_policy=report`、`result_mode=closest`、`max_hits=1`とする。Camera ownerのBody handleを各requestの`ignored_bodies`へ必須指定する。

### 9.2 `CollisionQueryRequestV1`

| Field | 型／制約 |
|---|---|
| `request_id` | callerごとに単調増加uint64 |
| `dimension` | `two_d \| three_d` |
| `physics_scene_version` | request作成時のversion |
| `query_profile_id／hash` | `CollisionQueryProfileV1`の固定revision |
| `query_kind` | `ray \| shape_cast \| overlap \| point` |
| `origin_pose` | dimension別finite pose、scaleなし |
| `translation_m` | castだけ。finite、長さ`(0, 10,000]` |
| `query_shape` | query_kindに対応するconvex geometry |
| `include_channels` | 1～32個のchannel ID set |
| `include_sensors` | bool |
| `include_body_kinds` | nonempty set |
| `ignored_bodies` | `PhysicsBodyHandle`最大16 |
| `back_face_mode` | `front_only \| both`。Mesh／Heightfield target用 |
| `initial_overlap_policy` | `ignore \| report`。Shape castだけ |
| `result_mode` | `any \| closest \| all` |
| `max_hits` | `any／closest=1`、`all=1～256` |

Rayのoriginがsolid内部にある場合、そのsolidのexit faceをhitとして返さない。内部判定が必要なcallerはPoint／Overlapを先に行う。Shape castの`initial_overlap_policy=report`は同じposeでOverlapを先に実行し、penetration depthを持つ`started_overlapping=true`のhitをfraction 0で返した後にsweepする。`ignore`ではBackendの初期overlapをhitへ変換しない。

`any`は`has_hit: bool`だけを返し、Bodyやshape identityを返さないためBackendは最初の有効候補で停止できる。`report`かつ`any`ではfraction 0のoverlapがあれば`has_hit=true`としてsweepを実行しない。`report`かつ`closest`ではfraction 0のoverlapが一つ以上あればcanonical順の先頭だけを返し、sweepを実行しない。`all`ではfraction 0のoverlapをcanonical順で格納してから残capacityへsweep hitを追加する。実hit数が`max_hits`を超えた`all` Queryは`max_hits + 1`件を観測した時点で停止し、partial listを成功として返さず、`MIRA-COLLISION-QUERY_HIT_CAPACITY_EXCEEDED`と`observed_at_least = max_hits + 1`を返す。Global query result queue自体の上限超過はAuthoritative queue overflowとしてtickをfaultする。

### 9.3 Result normalization

`CollisionQueryResultV1`は`any -> {has_hit}`、`closest -> {hit: optional<CollisionQueryHitV1>}`、`all -> {hits: CollisionQueryHitV1[]}`のtagged unionである。`CollisionQueryHitV1`は次を持つ。

```text
request_id
physics_scene_version
body_handle
collider_asset_version
shape_slot
fraction              // [0, 1]
distance_m
position_m
normal
started_overlapping
penetration_depth_m   // started_overlappingだけ正数、それ以外0
material_asset_id
surface_tag
```

全floatをbinary32へ変換し、`-0`を`+0`へ正規化する。normalは長さ`[0.999, 1.001]`、fractionは`[0,1]`を必須とする。`closest`のtieと`all`結果は`{fraction bits, body_handle, collider_asset_version, shape_slot, position bits, normal bits}`の昇順に決める。Native callback到着順、tree順、thread順を保持しない。

### 9.4 実行時点

- Gameplay Logicは`T30_PrePhysics`でtyped requestを生成する。
- Physics Adapterは`T40_MotionIntent`で直前に完了したPhysics worldをread-only queryする。
- `T60_PhysicsIntegrate`でversion／generation検査後にResultをpublishする。
- `T70_PostPhysics` consumerは同tickにResultを読めるが、構造変更は次tick`T00` commandになる。
- Camera等の`T20` consumerは次tickにversion一致Resultだけを読む。
- Character Motor内部queryだけはPhysics ownerが`T40`中に同期利用でき、GameplayDefinition／AI／Project callbackへその入口を公開しない。
- Render thread、Audio thread、AI OrchestratorはPhysics worldをqueryしない。

## 10. Contact、Trigger、Hit

### 10.1 Event型

| Event | 意味 |
|---|---|
| `ContactBeginV1` | 二つのsolid shapeが新しくsolver contactを持った |
| `ContactPersistV1` | 前stepから有効なsolver contactが継続した |
| `ContactEndV1` | solver contactが終了した |
| `TriggerEnterV1` | Sensorと対象shapeが新しくoverlapした |
| `TriggerExitV1` | 実overlapが終了した |
| `CollisionHitV1` | Begin／Persistのclosing speedがProfile threshold以上 |
| `ColliderRemovedV1` | BodyまたはCollider generationがT00でretireした |

`ContactEventV1`の共通payloadはbody pair、Collider version、shape slot、両側のMaterial Asset ID／surface tag、event kind、tick、manifoldを持つ。Triangle Mesh／Heightfieldではnative subshapeから当該triangle／cellのMaterial slotをAdapter内で解決する。2D manifoldは最大2点、3Dは最大4点である。Manifold pointはposition、AからBを向くunit normal、separation、`approach_speed_mps`を持つ。接触点におけるstep開始時のlinear＋angular velocityを`v_a`、`v_b`、AからBへのnormalを`n`として、`approach_speed_mps = max(0, dot(v_a - v_b, n))`へ正規化する。End／Exitはmanifoldを持たず、`contact_point_index=0`と`separated | disabled | profile_changed | teleported`の`end_reason`を持つ。Body／Collider破棄は通常End／Exitではなく`ColliderRemovedV1`で通知する。Gameplay公開payloadへvendor solver impulseを含めない。Box2DとJoltで取得時点と意味が一致しないためである。Solver impulseはProfiler専用のnon-authoritative telemetryとしてBackend名とsourceを付けて別streamへ出す。

Bodyまたはislandがsleepへ入ったことだけを理由にContact End／Trigger Exitを生成しない。Engine-owned pair cacheはsleep中もpair identityを保持し、wake後も実際に分離していなければ新しいBegin／Enterを生成しない。Sleep中はContact Persistを生成しない。Native Backendがsleep時にcontact removalを通知する場合、Adapterがその通知をGameplay eventから除外する。

Trigger payloadはmanifoldを持たず、`sensor_a`、`sensor_b`を持つ。Sensor–Sensor pairをBackendが両Sensor視点から二回通知しても、Engineはcanonical body／shape pairへdeduplicateして一件だけ配送する。Sensorのどちらを「trigger owner」とするかをcallback順から推測せず、Gameplayはsemantic roleと`shape_slot`で判定する。

`ColliderRemovedV1`は`removed_entity_stable_id`、retire直前の`body_handle_bits`、`collider_asset_version`、`tick`を持つ。Handleは識別値であり、受信側がresolveしてはならない。Event sortでは`body_a=body_handle_bits`、`body_b=0`、両shape slotと`contact_point_index=0`として扱う。Handle slotは当該T70配送とpair cache purgeが完了するまで再利用しない。

`CollisionHitV1`のReference thresholdは1.0 m/sで、Project Profile範囲は`[0, 1,000]`である。同じbody／shape pairには1 tick最大一件だけ生成し、全manifold pointの最大`approach_speed_mps`を使う。同値pointは10.2節のcanonical point順の先頭を採用する。Pairは開始時armed、armedかつthreshold以上で一件発行してdisarmし、speedが`0.5 * threshold`未満またはContact Endになった時だけrearmする。armed stateはpair cacheとReplay stateへ含める。Damage、Audio、VFXはhit eventを購読できるが、VFX／Audio結果をGameplayへ逆入力しない。

### 10.2 Canonical ordering

- packed `PhysicsBodyHandle`の小さい方をAにする。
- Bodyを入れ替えた場合はnormalをAからBへ反転する。
- Event kind順は`ContactBegin=0`、`ContactPersist=1`、`ContactEnd=2`、`TriggerEnter=3`、`TriggerExit=4`、`CollisionHit=5`、`ColliderRemoved=6`とする。
- Manifold pointはposition、normal、separation、approach speedのbinary32 bit pattern順に並べ、`contact_point_index`を0から再採番する。
- 最終Eventは`{event_kind, body_a, body_b, collider_a_version, shape_a_slot, collider_b_version, shape_b_slot, contact_point_index, payload_bits}`順にsortする。
- NaN、Inf、zero normal、無効generation、retire済みAsset versionを含むEventは配送せず`MIRA-COLLISION-NATIVE_INVARIANT`としてtickをpublishしない。

Native Contact callback中はpreallocated thread-local bufferへのcopy以外を禁止する。allocation、logging、World access、Body lock取得、Physics変更、Gameplay Logic、AI、Audio、VFX呼出しを行わない。Jolt callbackは複数threadから同時に呼ばれるため、bufferはworker別にする。Box2D event arrayはstep後に一度だけconsumeする。

### 10.3 Event subscriptionとoverflow

- `begin_end`はBegin／End、SensorではEnter／Exitを有効にする。
- `persist`はContact Persistを追加する。Sensorには指定できない。
- `hit`はCollision Hitを追加する。Sensorには指定できない。
- Contact eventが不要なColliderでは生成を無効化し、全Contactを常時配送しない。
- Runtime規約のNormalized Physics Event上限65,536件／tick、4 MiB arenaを変更しない。
- Authoritative Event queue overflowではeventをdrop、merge、sampleせず、`MIRA-COLLISION-EVENT_OVERFLOW`を記録して当該tickをpublishしない。Runtime session状態はRuntime規約の`AuthoritativeQueueOverflow`へ遷移する。

## 11. Collision responseとContinuous Collision

### 11.1 Block response

Filterを相互に通過し、両shapeがnon-sensorの場合だけsolver responseを生成する。Friction／restitutionは8.1節でEngineがmixする。GameplayがBody transformを直接上書きせず、dynamic bodyにはForce／Impulse／Velocity command、kinematic bodyにはTarget commandを送る。

C1／C2のcontact modificationはEngine同梱の次のtyped ruleだけに限定し、Capability列より前に公開しない。

| Rule | 成熟度 | 対象 | 動作 |
|---|---|---|---|
| `none` | C1 | 全 | 通常contact |
| `disable_connected_pair` | C1 | Joint／Constraint body pair | pair contactを無効化 |
| `one_way_2d` | C2 | static／kinematic 2D platform | platform local normal、approach side、skinでcontactを有効化 |

`one_way_2d`はplatform normalをunit float2、許可角を`[0, 89]` degree、skinを`[0.001, 0.1]` mで必須指定する。BodyまたはWorldへcallbackからアクセスせず、pre-copied velocity／poseだけで判定する。3D one-way、conveyor surface、custom friction overrideはC2 Capabilityである。

### 11.2 CCD

| Dimension | C1 mode | 規則 |
|---|---|---|
| 2D | `discrete \| bullet` | dynamic bodyだけ。Bulletは必要な高速bodyに限定 |
| 3D | `discrete \| linear_cast` | dynamic bodyだけ |

Validatorは`declared_peak_linear_speed_mps / 60 > 0.5 * minimum_solid_extent_m`の場合に`MIRA-COLLISION-TUNNELING_RISK` warningを出し、対応modeと厚み変更を提案する。AIはwarningだけで設定を自動Commitせず、Diffへ性能影響を表示する。

CCDはTeleport、Sensor overlap、zero-thickness geometry、全回転sweepを完全に保証しない。高速ProjectileのGameplay hitはswept Shape Queryを正規経路とし、表示用Rigid Bodyのcontactだけへ依存しない。

## 12. Character、Camera、Gameplay volume

### 12.1 Character Motor

Characterはdynamic Rigid Bodyへ直接速度を設定する方式をC1既定にせず、Engine-owned Kinematic Character Motorを使用する。本書はMotorが利用するcapsule、overlap、shape cast、hit ordering、filter semanticsを所有する。Reference Profile、state、T40のoverlap recovery／slide／step／slope／ground snap順、moving platform、root motion、AI Operation、fixtureは独自Physics Platform規約11～17節を正本とする。RuntimeでVendor Character型を保持しない。

### 12.2 Camera collision

Camera Collisionは3D sphere cast、2Dではcircle castを使用する。3D Reference値はradius 0.20 m、skin 0.05 m、最大補正距離10 mである。Query versionが一致しない結果を使用せず、Runtime規約のstale policyに従う。Camera channelではなく`CameraCollisionQueryProfile`のinclude channel setを使用し、Sensorを既定で除外する。

### 12.3 Gameplay volume

Hurtbox、Hitbox、Interaction、Quest、Damage volumeはSensor Colliderとして表現し、Particle、Render mesh、Audio volumeをauthoritative判定へ流用しない。高速Hitboxは前poseから現poseまでShape Castし、Trigger Eventだけに依存しない。同じattack instanceによる多重hit抑止はGameplayのStable attack IDで行い、Contact pair cacheへ隠さない。

## 13. Cook、Import、生成

### 13.1 Cook pipeline

```text
Source revision
  -> Structural validation
  -> Unit／axis／scale normalization
  -> Geometry validation
  -> Filter／Material／Body compatibility validation
  -> Backend-independent canonical shape
  -> Backend Cook in isolated worker
  -> Query／mass／AABB conformance test
  -> Content hash＋RuntimeInterfaceHash
  -> Staging Asset version
  -> T00 promotion
```

Importer／CookerはEditorとは別Processで、networkなし、許可pathだけ、timeout、memory capを持つ。Crash、timeout、out-of-memory、vendor assertは`MIRA-COLLISION-COOK_FAILED`へ変換し、Editor processを落とさない。

Cooked Assetは次を必須metadataとする。

- Source Asset ID、revision、content hash
- Cook Profile ID、version、hash
- Target Profile、dimension、backend、exact vendor version／commit
- canonical AABB、mass property、shape count、triangle／sample count
- `shape_id -> shape_slot` table
- filter／material dependency version
- Cook time、peak memory、warning、provenance

`MeshColliderCookProfile3D::ReferenceV1`は`max_triangles_per_leaf=8`、`active_edge_threshold_degrees=5`、`build_quality=favor_runtime_performance`、`per_triangle_user_data=false`、interacting Bodyの`enhanced_internal_edge_removal=true`へ固定する。`HeightFieldCookProfile3D::ReferenceV1`は7.1節の`block_size=2`、8～16 bit adaptive量子化、最大world-space誤差0.01 m、active edge threshold 5°を全値保存する。Vendor default objectをそのままserializeせず、Engineが渡した全fieldとvendorが返したCook resultをManifestへ記録する。

### 13.2 Generator

| Generator | C1 | 出力 |
|---|---|---|
| Boundsから2D box／circle／capsule | Production | Primitive Source Asset |
| Sprite metadataの明示polygon | Production | 検証済みconvex／compound |
| Tilemap collision tag merge | Production | Static 2D Collider Derived Asset |
| Mesh boundsから3D box／sphere／capsule | Production | Primitive Source Asset |
| Mesh pointから3D convex hull | Production | Convex Hull Source Asset |
| Concave 2D自動分解 | C2 | 専用Capability承認後 |
| 3D convex decomposition | C2 | Dependency ADRとquality fixture承認後 |
| Render Mesh全体からtriangle mesh | Staticだけ | 明示subset／Cook Profile必須 |
| Text promptからraw vertex生成 | 禁止 | Primitive／既存Mesh／Generator parameterへ分解 |

AI生成結果も人間生成結果と同じstaging、preview、validation、license／provenance、budget gateを通る。Generatorは元Mesh更新時に自動上書きせず、stale diagnosticとRegenerate Diffを作る。

### 13.3 Hot Reload

| 対象 | Play中 |
|---|---|
| Static Collider | staging worldでbuild、query、AABB、filter test合格後にT00 swap |
| Dynamic／Kinematic Collider geometry | 不可。Play restart |
| Sensor flag | 不可。Play restart |
| Filter Profile | 使用中Worldでは不可。Play restart |
| Material friction／restitution | C1では不可。Play restart |
| Editor非Play Preview | valid staging versionへ即時切替可能 |

Static swap時は旧Asset leaseをEvent normalize完了まで保持する。新旧generation間に通常のTrigger Exit／Enterを捏造せず、`ColliderRemovedV1`と新generationの次step結果で状態を再構築する。Shipping RuntimeはEditor hot reload pathを含めない。

表のFilter ProfileはProfile定義またはChannel tableの編集を指す。PlayPreparingでCook済みの既存ProfileへBody／shape instanceを切り替えることと、Sensor shapeのactive切替は5.4節の`ColliderRuntimeState`として許可する。

## 14. Runtime、所有権、thread

### 14.1 Module境界

```text
engine/physics/contracts
engine/physics/core
engine/physics/core/joints
engine/physics/core/character
engine/physics/core/save_replay
engine/physics/collision
engine/physics/diagnostics
engine/physics/backends/box2d
engine/physics/backends/jolt
authoring/collision
editor/panels/collision
tools/asset_compiler/collision
```

`engine/physics/contracts`はEngine-owned value typeとPort、`engine/physics/core`は独自Physics Platform規約のWorld／Dynamics／Joint／Character／Save、`engine/physics/collision`はbackend-independentなQuery／Event正規化とRuntime state、`authoring/collision`はSource model／Validator／Preview、`tools/asset_compiler/collision`はisolated Cookを所有する。Box2D、Jolt headerは各Adapter private implementationからだけincludeする。AdapterはWorldへ依存せず、Command／Snapshot／Asset payloadを消費し、copied resultを返す。

### 14.2 Lifetime

- `PhysicsBodyHandle`はPhysics AdapterのT40～T60だけresolveできる。
- Collider Asset payloadはimmutable `AssetVersionHandle`と`AssetReadLease`で保持する。
- Body destroyはT00 commandで行い、native object破棄前にEngine mappingをretire stateへ移す。
- Query／EventはhandleとAsset generationをT60で再検査する。
- Native pointer、span、Body lock、callback viewをtick、job、thread境界へ持ち越さない。
- Jolt複数Body accessはmulti-lock APIを使い、任意順の個別lockを重ねない。
- Box2D IDは利用前にvalidityを検査し、destroy後のSensor End IDをdereferenceしない。

### 14.3 Step

2D／3Dはexactly 60 Hzでstepする。Box2Dは`sub_step_count=4`、Joltは`collision_steps=1`をReference値とし、PlayPreparingで固定してReplay headerへ保存する。Sceneが両方を持つ場合はRuntime規約どおり順番にstepし、workerをjoinしてから次Worldへ進む。二つのWorldを同時stepしない。

JoltのTempAllocatorはWorldごとに32 MiB固定、C1 GameHostのactive Worldは2D一つ／3D一つ、Box2D／JoltはEngine allocatorとworker poolを使用する。Step中の一般heap fallback、独立worker pool、callback内allocationを禁止する。World／Worker／Solverの全値と上限は独自Physics Platform規約を正本とする。

## 15. Editor

### 15.1 Collider Editing Mode

Scene／Canvas Viewへ`Collider Editing Mode`を設け、次を提供する。

- Body kind、sleep、motion quality、mass modeのInspector
- Asset内shape listとStable ID
- Primitive handle、vertex、local pose、normal、one-way directionのGizmo
- Solid、Sensor、sleep、contact、AABB、center of massの重ね表示
- Collision channel／filter matrix editor
- Material previewとfriction／restitution比較fixture
- Ray／Shape Cast／OverlapのQuery probe
- Contact／Trigger event timeline
- Broadphase、pair、contact、query、Cook memory／time Profiler
- Source Mesh、Generated Collider、Cooked Colliderの差分表示
- Invalid shape、oversized body、tunneling risk、stale generatorへの直接navigation

Gizmo drag中はSource draftだけを更新し、throttleしたPreview Cookをbackground jobへ送る。Mouse release時に一つのChangeSetへまとめる。Cook中のpartial shapeをPlay Worldへ反映しない。

### 15.2 UXとAccessibility

- Scene colorだけで状態を区別せず、line pattern、icon、labelを併用する。
- Collider選択はHierarchy、Shape list、Scene overlayで同期する。
- Local／World座標、meter値、snap値を常時表示する。
- Vertex数、triangle数、memory、body／pair budgetへの影響をCommit前に示す。
- Undo／RedoはSource ChangeSet単位で、Cooked binaryを履歴正本にしない。
- Keyboard、command palette、screen reader semantic treeから主要操作へ到達できる。
- `Level` WorkspaceはScene、Outliner、Inspector、Navigation、Physics、AI Partnerを既定表示する。

## 16. AI Authoring

### 16.1 Capability

MCDへ次のCapabilityを定義する。

```text
capability.collision.authoring_2d_v1
capability.collision.authoring_3d_v1
capability.collision.filter_v1
capability.collision.query_v1
capability.collision.events_v1
capability.collision.debug_v1
```

AIはCapability discovery後に選択したSchemaだけを取得する。Box2D／Jolt型、raw bitmask、native layer、pointer、callbackはTool Schemaへ含めない。
Character Motorは`capability.physics.character_motor_v1`と`operation.physics.*`を使用し、Collision Capabilityはshape／query／event意味だけを提供する。

### 16.2 Typed Operation

| Operation ID | 動作 | Risk |
|---|---|---|
| `operation.collision.inspect_scene` | Body、Collider、Filter、warningを読む | R0 |
| `operation.collision.validate_changeset` | 構造、意味、budgetを検証 | R0 |
| `operation.collision.preview_changeset` | Collider／Query／性能差をPreview | R0 |
| `operation.collision.create_body_2d` | 2D Body Componentを作る | R2 |
| `operation.collision.create_body_3d` | 3D Body Componentを作る | R2 |
| `operation.collision.create_collider_asset_2d` | 2D Source Assetを作る | R2 |
| `operation.collision.create_collider_asset_3d` | 3D Source Assetを作る | R2 |
| `operation.collision.add_shape` | 型付きshapeを追加 | R2 |
| `operation.collision.update_shape` | 寸法、pose、material、filterを変更 | R2 |
| `operation.collision.remove_shape` | shapeを削除 | R2 |
| `operation.collision.assign_material` | Material Assetを割当 | R2 |
| `operation.collision.assign_filter_profile` | Filter Profileを割当 | R2 |
| `operation.collision.set_sensor` | Sensor semanticsを変更 | R2 |
| `operation.collision.set_motion_quality` | CCD modeを変更 | R2 |
| `operation.collision.set_runtime_filter_profile` | Cook済みProfileへのGameplay state切替をAuthoringする | R2 |
| `operation.collision.set_sensor_active_rule` | Hitbox等のactive制御RuleをAuthoringする | R2 |
| `operation.collision.generate_primitive` | Boundsから候補を生成 | R2 |
| `operation.collision.generate_convex_hull_3d` | Meshから候補を生成 | R2 |
| `operation.collision.bake_tile_collider_2d` | Tile regionをCook | R2 |
| `operation.collision.create_query_profile` | Query filterを作る | R2 |
| `operation.collision.modify_channel_table` | Project全体のpair semanticsを変更 | R2＋明示impact approval |

Schema、Engine code、Adapter、budget上限自体の変更はR3／R4であり、上表のAuthoring権限では実行できない。

### 16.3 Level 0自然言語

自然言語Intentの正規object、closed vocabulary、Capability比較、Diagnostic、Semantic EvalはPhysics AI Semantic Capability Catalog規約を正本とする。本節はCollision固有の解決順と安全規則を定める。

AIは自然言語を次の順で解決する。

1. gameplay intentを`solid、sensor、character、projectile、camera blocker、volume`へ分類する。
2. 2D／3D gameplay spaceをGameSpecから取得する。
3. body kind、shape候補、filter、event、motion qualityを選ぶ。
4. 不足が結果を大きく変える場合だけ一問ずつ質問する。
5. Source ChangeSet、Preview、Validator結果、performance差を提示する。
6. 承認または有効なR2事前委任後にCommitする。

次はBlocking questionにする。

- 「当たる」が押し返すsolidか、通知だけのsensorか不明
- 動くobjectがkinematicかdynamicかでGameplayが変わる
- 2D／3D gameplay spaceが未確定
- Render Mesh精度とprimitive性能のどちらを優先するかで結果が大きく変わる
- Project全体のCollision channel matrix変更が必要
- 高速Projectileでsweep、CCD、Triggerのどれをauthoritativeにするか不明

次は安全なReference defaultを選び、AssumptionとしてDiffへ記録できる。

- 壁、床、地形はstatic solid
- Door／pickup／quest areaはsensor
- Player／NPCはCharacter Motor
- 小物の落下はdynamic convex／primitive
- Camera collisionはSensorを除外
- Render形状が複雑でもdynamic bodyへtriangle meshを使わない

AIはValidator errorを理由にColliderを削除、Sensorへ変更、Filterを全許可、budgetを引上げて成功扱いしない。

## 17. GameplayDefinitionとNativeGameModule

### 17.1 GameplayDefinition

GameplayDefinition schemaへ次の型付きOperationだけを定義する。

- Collision Query requestとversion付きresult
- Contact／Trigger／Hit eventを条件とするRule
- Dynamic bodyへのForce／Impulse／Velocity command
- Kinematic targetとCharacter move intent
- 明示`TeleportBody` command
- Cook済みProfileへのT00 filter切替
- Sensor shapeのT00 active切替
- BodyのEngine-visible state snapshot

これらは関数bindingではなく、MCDで列挙されたCapability ID、型付き引数、対象Stable ID、consume phase、上限を持つ宣言である。Cookerは参照、権限、phase、回数、query hit上限を検証し、C++ evaluatorがbounded commandへ変換する。任意関数呼出し、loop、recursion、callback登録、FFIは存在しない。

GameplayDefinitionからCollider geometry、Sensor flag、Filter table定義、Material、Body kindをPlay中に変更しない。許可するのは5.4節の既存Profile選択とSensor activeだけである。Native ID、pointer、World step、lock、allocatorをdataへ保存または公開しない。

### 17.2 NativeGameModule

NativeGameModule（Project C++）は`collision_port`のtyped command／query／eventを使用する。Vendor headerをProject targetへlinkまたはincludeしない。C1ではProject独自Contact callbackを登録できない。Gameplay拡張はEvent consumerまたはMCD operationとして実装する。

新しいContact Rule、Shape type、Backend設定、Physics phase、queue、memory budgetを追加するC++はEngine Architecture変更としてR4 Reviewを要求する。

## 18. Memory、性能、容量

### 18.1 `CollisionBudgetProfileV1`

| Field | Windows C1 | Mobile baseline | Mobile standard | Mobile high |
|---|---:|---:|---:|---:|
| live body | 16,384 | 8,192 | 16,384 | 32,768 |
| live shape slot | 65,535 | 32,768 | 65,535 | 65,535 |
| active dynamic／kinematic body | 8,192 | 4,096 | 8,192 | 16,384 |
| broadphase body pair | 131,072 | 65,536 | 131,072 | 262,144 |
| contact constraint | 32,768 | 16,384 | 32,768 | 65,536 |
| live trigger pair | 16,384 | 8,192 | 16,384 | 32,768 |
| public query／tick | 4,096 | 2,048 | 4,096 | 4,096 |
| query hit／tick | 65,536 | 32,768 | 65,536 | 65,536 |
| static mesh triangle／Asset | 500,000 | 125,000 | 250,000 | 500,000 |
| heightfield sample／Asset | 4,194,304 | 262,144 | 1,048,576 | 4,194,304 |

上限はTarget Profileへ全値をCookし、Play中に変更しない。Projectが低い値を選ぶことはできる。高い値はReference scene、memory、2.50 ms Physics P95、10分soak、mobile thermal gateを再実行するADRなしに許可しない。

Jolt `max body pairs`または`max contact constraints`を超えるとcollision欠落を起こし得るため、AdapterはUpdate errorを`MIRA-COLLISION-CAPACITY_EXCEEDED`へ変換し、そのtickをpublishしない。Box2D／Engine allocatorの容量不足も一般heapへfallbackせず同様にfaultする。

### 18.2 Windows memory内訳

Runtime規約のPhysics 96 MiBを次へ固定する。

| 用途 | MiB |
|---|---:|
| Normalized Physics event二面buffer | 12 |
| Jolt step TempAllocator | 32 |
| live Box2D／Jolt world、body、shape、joint、broadphase | 40 |
| Engine mapping、trigger pair cache、query、Character、Joint registry、Replay scratch | 8 |
| Physics domain non-lendable reserve | 4 |
| **合計** | **96** |

Cooked Collision Assetのstaging payloadはReadyまではAsset streaming、T00 promotion後のlive payloadはPhysics 40 MiBへchargeする。2Dと3Dを同時利用しても96 MiBを二重に割り当てない。40 MiBを超えるProjectはCook時に拒否し、別Profileへ無通知で移行しない。

Mobileはモバイル規約のEngine-owned CPU aggregateへ同じcounterをchargeし、18.1節のTarget別capacityを使う。Apple unified memoryでCPUとGPUを単純加算しない。

### 18.3 Performance gate

- Windows ReferenceのPhysics全体P95は2.50 ms以下。
- Collision QueryはPhysics 2.50 ms内に含め、query総時間P95 0.50 ms、単一public query P99 0.10 msをsoft diagnosticとする。
- Step中の一般heap allocation countは0。
- Collider Cookはcancel可能なbackground jobとし、Primitive P95 2 ms、256-point Convex Hull P95 100 ms、250k triangle Static Mesh P95 2 sをEditor soft targetとする。
- Performance値はwarm-up後10分、5 runのmedian、Runtime規約のnearest-rank定義で測る。
- Mobileは30分thermal soakと2時間enduranceを追加し、30／60 fpsでもSimulationは60 Hzを維持する。

Soft target超過をCollider簡略化の無通知適用で隠さない。AIとEditorはprimitive化、shape削減、static merge、event subscription削減、query削減を候補Diffとして提示する。

## 19. Serialization、Save、Replay

- Project sourceはMCD JSON、Collider Source Asset、Profile参照をtext-diffable形式で保存する。
- Cooked vendor binaryをProject source of truthにしない。
- SaveはBodyのEngine-visible pose、velocity、sleep、Gameplay-owned state、Asset Stable ID＋generation互換情報を保存する。
- Saveへnative Body ID、pointer、broadphase node、contact cache、solver islandを保存しない。
- Load時は現在のCooked Asset versionとSave compatibilityを検証し、不一致を近似shapeへfallbackしない。
- Replay headerはBox2D／Jolt exact version、Cook Profile hash、Collision Profile hash、60 Hz、sub-step／collision step、`shared_worker_pool_count`、`physics_worker_count`を持つ。
- Determinism保証は同一version、platform、compiler、thread設定のReplay範囲である。2Dと3D、Windowsとmobile、Box2DとJoltのbitwise一致を要求しない。
- Pre-1.0 Schema変更はoffline Project Migratorでbackup、旧Schema検証、変換、Diff、新Schema検証、atomic切替を行い、Runtime compatibility branchを残さない。

Physics World、Joint、Characterを含むSave field、Load再構築順、Replay Build／SIMD環境、最初の不一致reportは独自Physics Platform規約14節を正本とする。

## 20. DiagnosticとFailure

初期Diagnostic IDを次へ固定する。

| ID | 条件 | 動作 |
|---|---|---|
| `MIRA-COLLISION-INVALID_GEOMETRY` | 退化、自己交差、非convex、NaN／Inf | Commit／Cook拒否 |
| `MIRA-COLLISION-CONVEX_RADIUS_OUT_OF_RANGE` | 3D convex radiusがshape範囲外またはvendorが縮小 | Commit／Cook拒否 |
| `MIRA-COLLISION-SCALE_NOT_BAKED` | Runtime scaleがidentityでない | Play拒否 |
| `MIRA-COLLISION-DYNAMIC_TRIANGLE_MESH_FORBIDDEN` | dynamic／kinematic mesh | Commit拒否 |
| `MIRA-COLLISION-FILTER_ASYMMETRIC` | pairの片側だけ許可 | Commit拒否 |
| `MIRA-COLLISION-MASS_SOURCE_MISSING` | dynamic massを計算不能 | Play拒否 |
| `MIRA-COLLISION-TUNNELING_RISK` | 速度／厚みheuristic超過 | warning＋候補Diff |
| `MIRA-COLLISION-DECLARED_MOTION_PROFILE_EXCEEDED` | 実速度が宣言peakを連続60 tick超過 | warning＋Profile見直し |
| `MIRA-COLLISION-SENSOR_CCD_UNSUPPORTED` | SensorをCCD保証として使用 | error |
| `MIRA-COLLISION-COOK_FAILED` | Worker crash／timeout／invalid output | last valid維持 |
| `MIRA-COLLISION-CAPACITY_EXCEEDED` | body／pair／contact／query上限 | tick非publish／Play fault |
| `MIRA-COLLISION-QUERY_STALE` | Scene version不一致 | result破棄 |
| `MIRA-COLLISION-QUERY_HIT_CAPACITY_EXCEEDED` | `all` Queryの実hit数がcaller上限超過 | partial resultなし |
| `MIRA-COLLISION-EVENT_OVERFLOW` | 65,536件または4 MiB超過 | tick非publish |
| `MIRA-COLLISION-NATIVE_INVARIANT` | invalid ID、normal、callback、lock違反 | tick非publish／fault |
| `MIRA-COLLISION-RESTART_REQUIRED` | Play中非互換変更 | old generation維持 |

DiagnosticはStable ID、Entity、Collider Asset、shape ID／slot、Source path、Profile、Target、数値、上限、修正候補を含む。Vendor assert textだけをユーザーへ返さず、原文は開発者詳細へ添付する。

## 21. Testと合格条件

### 21.1 Schema／Unit

- 全shapeのvalid境界値と、NaN、Inf、zero、negative、hard範囲外
- Quaternion、rotation canonicalization、scale禁止
- Body kindとmass、shape、sensor、motion qualityの組合せ
- 32 channel、Channelごとに一つのProfile、symmetric pair、reserved ID
- Material mix式とSensor mass除外
- Query request、max hit、initial overlap、canonical sort
- Diagnostic IDとSource location

### 21.2 Adapter conformance

Box2D 3.1.1とJolt 5.6.0について次を各Backendで固定fixture化する。

- Primitive AABB、mass、center of mass
- Static／kinematic／dynamic pair matrix
- Filter、Sensor、connected-pair disable
- Ray、Shape Cast、Overlap、initial overlap
- Begin／Persist／End、Enter／Exit、sleep、body destroy
- High-speed body、thin wall、swept Projectile
- Worker count 1／Reference最大、callback順序撹乱
- Invalid／reused native ID、Asset generation swap
- Capacity fault、TempAllocator exhaustion、Event overflow

Cross-backendでbitwise同一のtrajectoryを要求しない。Engine-visible invariant、event kind、filter結果、query集合、sort順、failure policyを一致させる。

### 21.3 Gameplay fixture

| Fixture | 合格条件 |
|---|---|
| 2D stack | 100 body、10分、NaN／escapeなし、budget内 |
| 2D chain seam | 接続部でghost hit／落下なし |
| 2D one-way（C2有効時） | 下から通過、上から支持、edge条件がProfileどおり |
| 2D fast projectile | swept queryがthin targetを1回検出 |
| 3D stack／ramp | sleep、friction、50°境界が再現 |
| 3D mesh internal edge | Reference characterがseamで跳ねない |
| 3D character stair | 0.35 m step成功、上限超過失敗 |
| Trigger sleep | sleepだけでExitを生成しない |
| Destroy in contact | invalid Eventなし、次T00で安全にretire |
| Static hot reload | staging test後T00 swap、old lease解放 |
| Camera obstruction | stale policy、radius／skin、最大補正を満たす |
| Hybrid scene | 2D／3D直接collisionなし、Gameplay event bridgeだけ |

### 21.4 Fuzz、determinism、performance

- Geometry、filter、query MCDへproperty-based fuzzを最低100,000 case／release実行する。
- Cook Workerへmalformed Mesh、巨大count、duplicate、degenerate、cancel、process killを注入する。
- Native callback arrival順をrandomizeし、Engine event digestが同一になることを検証する。
- 同一環境Replayを100回実行し、最初のdigest不一致tickを0件とする。
- Windows両Reference GPU環境でCPU Physics P95、memory、allocation、queueを測定する。Physics CPU gateはGPUに依存しないが、Game fixture全体を両環境で通す。
- Android／Apple各device classで30分thermal、2時間endurance、background／resume後のPhysics scene再構築を検証する。

### 21.5 Editor／AI

- Gizmo編集、Undo／Redo、Source／Cooked Diff、invalid shape navigation
- Collision matrixのkeyboard操作、screen reader label、color以外の状態表現
- AI promptをsolid／sensor／character／projectile／tile／mesh／高速／曖昧／矛盾／unsupportedに各12件、合計120件、各3回実行する。
- 無効Collider Commit、dynamic triangle mesh、非対称filter、Sensor CCD誤保証、無権限channel変更を360 runで0件にする。
- Blocking question、Reference default、Assumption、Preview、Risk分類の期待値一致を95%以上にする。
- AIが作成したColliderと人間がInspectorで修正したColliderを同じChangeSet／Validator／Cook経路で往復できることを確認する。

## 22. Capability成熟度と実装順

| Capability | C0 | C1 | C2 | C3 |
|---|---|---|---|---|
| Contract | MCD、Validator、Diagnostic | C++／TS／Cooked binary／MCP projection | Profile拡張 | Specialized |
| 2D Shape | Preview Cook | Primitive、convex、segment、chain、tile | concave分解、one-way | specialized |
| 3D Shape | Preview Cook | Primitive、convex、static mesh／heightfield | tapered、decomposition、streaming | soft／GPU |
| Filter／Sensor | Schema、matrix | Production | advanced contact rule | network filter |
| Query | Canonical fixture | Ray／Cast／Overlap／Point | distance／batch | rewind |
| Event | Normalize fixture | Contact／Trigger／Hit | advanced surface event | network |
| Character | Profile／Preview | 2D、3D Kinematic Motor | advanced crowd／vehicle | specialized |
| Editor／AI | Inspector、Tool Schema | Gizmo、Diff、Level 0 | batch optimization | learned assist |

実装順は次へ固定する。

1. C0 MCD、canonical value type、Validator、Diagnostic、Cook envelope
2. Collision Editor preview、primitive geometry、filter matrix
3. Box2D Adapter、2D Query／Event、top-down action fixture
4. 2D Tile／Chain、独自Physics Platform規約のCharacter2D
5. Jolt Adapter、3D primitive／convex、Query／Event
6. Static Mesh／Heightfield、独自Physics Platform規約のCharacter3D、Camera collision
7. Mobile capacity／lifecycle／thermal conformance
8. C2のone-way、concave分解等はCapabilityごとのADRとfixture合格後に一つずつ昇格

2Dと3DのAdapterを同時実装しない。C0の共通contractを完成させ、2D C1をProductionにした後で3D C1へ進む。

## 23. 機能連携表

| Consumer | Collisionから受けるもの | 禁止接続 |
|---|---|---|
| Gameplay | Query Result、Contact／Trigger／Hit Event | native callback、pointer |
| Character | T40 internal convex query、resolved pose | Animationとの二重transform |
| Camera | version付きCast Result | Render thread direct query |
| Navigation | Cook済みstatic source generation、Physics transform snapshot | live shape pointer |
| Animation | resolved Character pose、Ragdoll input | Collider direct write |
| VFX | presentation Event | Particleをauthoritative Collider化 |
| Audio | Hit Event、surface tag | Audio resultをGameplayへ戻す |
| Rendering | immutable debug snapshot | Physics object dereference |
| Save／Replay | Engine-visible body state、Profile hash | solver cache／native ID |
| AI／Editor | MCD、Preview、Diagnostic、Diff | live World mutation |

Navigationへ渡すStatic collision sourceはCollider Source revisionから別Derived AssetとしてCookする。Navigation cellとColliderを同一binaryへせず、一方のpromotion失敗で部分更新しない。VFX、Audio、Camera、NavigationがContact callback内で処理を開始しない。

## 24. Riskと採用しない案

| Risk | 対策 |
|---|---|
| Vendor APIがProjectへ漏れる | Port／Adapter、public include CI、ABI conformance |
| AIが複雑すぎるMesh Colliderを生成 | Primitive優先、Generator、Preview、budget gate |
| Layer／Mask設定が非対称 | Profile相互検査、raw bitmask禁止 |
| 高速objectがすり抜ける | heuristic warning、CCD、authoritative swept query |
| Sensorがsleepで誤Exit | Engine overlap cache、revalidation |
| Contact callbackのrace／deadlock | copy-only、no lock／allocation、thread-local buffer |
| Hot Reloadでpairが壊れる | immutable version、staging test、T00 swap、restart rule |
| Collisionがmemoryを使い切る | fixed capacity、Domain charge、no heap fallback |
| 2Dと3D結果を同一視する | dimension別型、cross-world直接collision禁止 |
| 「独自」を理由にsolverを再発明する | Engine契約を独自所有し、検証済みkernelを隔離利用 |

次の案は採用しない。

- ColliderをRender Mesh Componentのflagとして実装する。
- Body、Collider、Material、Filterを一つの巨大Componentへ統合する。
- AIにLayer bitmask、native Body ID、vertex buffer、callback sourceを直接書かせる。
- Sensor、Query、Contactを同じ無型`OnCollision` eventへ統合する。
- Dynamic triangle meshを自動でConvex Hullへ置換する。
- Filter不整合をEngineが片側だけ自動修正する。
- Event overflow時に古いEvent、Sensor Event、低priority Contactを黙って捨てる。
- Runtime scale変更時にmain threadでColliderを再Cookする。
- Box2D／Jolt default変更をversion更新時に無検証で採用する。

## 25. 完了条件

本仕様のCollision C1は、次をすべて満たした時だけProductionと表示する。

1. 本書のMCD type、operation、capability、profile、diagnosticが正本化される。
2. C++／TypeScript／Cooked binary descriptor／MCP projectionが同じMCDから生成される。
3. Editor／AI／人間によるGameplayDefinition編集が同じChangeSet／Validator／Cook経路を使う。
4. Box2D／Jolt型がCX0 Public Header、CX3 Module interface、Project source、Save、Eventへ出ない。
5. 2D C1または3D C1の該当Adapter conformanceが全合格する。
6. Query／Eventのcanonical ordering、lifetime、overflow、fault injectionが合格する。
7. Reference gameplay fixture、10分soak、memory、2.50 ms P95を満たす。
8. Mobile対応を名乗るTargetは実機thermal／endurance gateを満たす。
9. AI Evalでunsafe Commit 0、期待挙動95%以上を満たす。
10. Source、Cooked Asset、Build manifest、Verification Receipt、provenanceがhashで連結される。

一部だけ合格した場合はCapability Manifestへそのdimension／maturityだけを掲載する。2D C1合格を3D C1、Windows合格をAndroid／Apple合格として表示しない。

## 26. 公式資料と採用根拠

調査基準日は2026-07-19である。外部資料はLibraryの事実と制約を確認するために使い、Miraikanai固有のSchema、上限、event順、Editor、AI Policyを外部Engineからコピーしない。

| 公式資料 | 確認事項 | 本書への反映 |
|---|---|---|
| [Box2D v3.1.1 release](https://github.com/erincatto/box2d/releases/tag/v3.1.1) | 固定Stable release | exact version／commitをToolchain lock |
| [Box2D 3.1 Simulation](https://box2d.org/documentation/md_simulation.html) | body、filter、sensor、event、query、CCD、event data lifetime | ID検査、step後consume、Query sort、Sensor制約 |
| [Box2D Shape API](https://box2d.org/documentation/group__shape.html) | Sensor eventは既定off、Sensorはresponseなし、Chain制約 | Engine event preset、Chain Validator |
| [Box2D v3.1.1 collision header](https://github.com/erincatto/box2d/blob/v3.1.1/include/box2d/collision.h) | Polygon最大8頂点 | 2D convex上限8 |
| [Box2D v3.1.1 types header](https://github.com/erincatto/box2d/blob/v3.1.1/include/box2d/types.h) | 64-bit category／mask、ID based API | Engine 32 channelを安全にcompile |
| [Jolt Physics v5.6.0 release](https://github.com/jrouwe/JoltPhysics/releases/tag/v5.6.0) | 固定Stable release、GPU／hair追加 | CPU rigid bodyだけをBuild |
| [Jolt Architecture v5.6.0](https://github.com/jrouwe/JoltPhysics/blob/v5.6.0/Docs/Architecture.md) | Broad／Narrow phase、Layer filter、Body locking、multithread | Filter compile、Query、lock contract |
| [Jolt ContactListener v5.6.0](https://github.com/jrouwe/JoltPhysics/blob/v5.6.0/Jolt/Physics/Collision/ContactListener.h) | callbackは複数thread、body lock中、Physics変更不可 | copy-only callback |
| [Jolt ConvexHullShape v5.6.0](https://github.com/jrouwe/JoltPhysics/blob/v5.6.0/Jolt/Physics/Collision/Shape/ConvexHullShape.h) | Cooked hull最大256 point | Convex Cook hard gate |
| [Jolt MeshShape v5.6.0](https://github.com/jrouwe/JoltPhysics/blob/v5.6.0/Jolt/Physics/Collision/Shape/MeshShape.h) | triangle winding、single-sided simulation、static制約 | Static-only Mesh、back-face policy |
| [Jolt HeightFieldShape v5.6.0](https://github.com/jrouwe/JoltPhysics/blob/v5.6.0/Jolt/Physics/Collision/Shape/HeightFieldShape.h) | static-only、1～16 bit sample | Heightfield Cook Profile |

Library更新時はrelease noteだけでなく、Adapter conformance、collision fixture、performance、memory、Save／Replay environment、Toolchain lock hashを再検証する。Version文字列だけを変更して採用しない。
