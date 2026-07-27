# Miraikanai Engine Math／Core Utilities

- 文書ID: mirakan.arch.math-core
- 状態: review
- 正本範囲: Foundation utilityとMath target、semantic／compact型、座標・単位・matrix・quaternion、floating-point、失敗契約、AI projection、Interchange／Shader境界、Qualification
- 非正本範囲: 外部Library・Tool version／hash／license、一般命名・配置、Memory／Pointer taxonomy、Runtime budget／phase、Product capability maturity。各Owner文書を参照する
- 依存: [Product Plan](../00-product/product-plan.md)、[Core architecture](core-architecture.md)、[Toolchain／Dependencies](toolchain-dependencies.md)、[Executable contracts](executable-contracts.md)、[Naming／Project layout](naming-project-layout.md)、[Memory／Pointers](memory-pointers.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論

Miraikanai EngineのMathとCore Utilitiesは、汎用`Vector3`と雑多な`utils`を全Subsystemへ公開する方式にしない。次を公式方式とする。

1. `mirakan::foundation`はID、Result／Error、Diagnostic、Hash、Endian、Time、bounded access、memory tag等の非数学基盤を所有する。
2. `mirakan::math`はscalar、角度、vector、matrix、quaternion、transform、geometry、数値検証を所有し、`mirakan::foundation`だけへ依存する。
3. hot path内部ではcompactな`Vec2f`／`Vec3f`／`Mat4f`等を使えるが、AI、Editor、Authoring、Save、Subsystem公開境界では位置、方向、単位、座標空間、正規化条件を持つsemantic typeを使う。
4. Math／Utilityの正本はMiraikanai Contract Definition（MCD）と本書であり、C++型、Editor metadata、serialization descriptor、MCP／Provider Schema、Validator、Diagnostic、testを同じ定義から生成する。
5. AIはraw memory、任意matrix、内部SIMD型、Platform math型を直接編集しない。意味付きFieldまたはtyped Project change primitiveを提案し、C++ Gatewayが完全再検証する。
6. Phase 0でportable scalar reference、MCD projection、unit／property／golden／cross-language testを完成させる。SIMD、DirectXMath、GPU近似、高度Geometry最適化は実測後の別Promotionとする。
7. `utils`、`helpers`、`common`という責務不明Directory／namespace／targetを作らない。

この方式は、Unityのserialization／generic property編集、Unreal EngineのReflection／Property metadata／Toolset Schema、GodotのVariant／ClassDBから有効な原則を採用しつつ、意味、権限、budget、failure、remediationをMCDへ統合するMiraikanai固有のcontract-first設計である。

## 2. 目的

MathとCore Utilitiesはほぼ全Subsystemの下位依存であり、ここで意味が曖昧だと次の問題が全Engineへ伝播する。

- AIが位置と方向、meterとpixel、worldとlocal、degreeとradianを混同する。
- Quaternion、matrix、TRSの合成順がSubsystemごとに変わる。
- zero-length normalize、singular inverse、NaN／Infを暗黙fallbackして障害原因を失う。
- Renderer、Physics、Asset Import、Shaderでtranspose、handedness、alignmentが不一致になる。
- `utils`へ所有者不明の関数が集まり、依存cycle、重複実装、互換性のないepsilonが増える。
- SIMD最適化が意味またはReplayを変えても検出できない。

本書の目的は、AIと人間が同じ型、同じ制約、同じDiagnosticを理解し、portable referenceからTarget最適化まで意味同等性を検証できる基盤を作ることである。

## 3. 決定権

### 3.1 本書が決定する事項

- Math／Core Utilitiesのtarget、namespace、dependency、Directory。
- scalar、vector、matrix、quaternion、transform、geometryの公開意味。
- semantic typeと内部storage typeの境界。
- finite、normalization、inverse、decomposition、comparison、canonicalizationの規則。
- Authoritative／Presentation別floating-point policy。
- AI discovery、MCD type／operation、Diagnostic、Remediation、Projection。
- portable reference、SIMD／Platform backend、Qualification、test、Phase導入順。

### 3.2 他文書が決定する事項

- 製品のhandedness、up／forward、meter／radian、clip／depth、matrix合成規約は2D／3D機能計画。
- Replay保証範囲、Simulation Cadence／Advance、authoritative RNG algorithm、state hashはRuntime規約。
- MCD共通Field、Projection、canonical JSON、Provider制約は実行可能契約規約。
- ownership、handle、lease、allocator、memory budgetはMemory／Pointer規約。
- Physics tolerance、collision geometry、solver値はCollision／Physics規約。
- Camera projection値とCamera-specific interpolationはCamera規約。
- GPU buffer layout、Shader ABI、Backend bindingはRendering規約。
- Import format固有変換、TRS受理条件、Conversion ReportはAsset Import規約。

数値が重複する場合、Domain固有値はDomain規約、本書は共通型と実行意味を所有する。Domainが本書の共通意味を変更せず、より狭いvalid rangeを設定することは許可する。

## 4. 公式Targetと依存

### 4.1 CMake target

| Target | 公開Alias | 責務 | 許可依存 |
|---|---|---|---|
| `mirakan_foundation` | `mirakan::foundation` | Result／Error、Diagnostic、ID、Hash、Endian、Time、bounded access、memory tag | C++ standard library、明示承認されたPlatform-free Build dependency |
| `mirakan_math` | `mirakan::math` | scalar、unit、vector、matrix、quaternion、transform、geometry、numeric validation | `mirakan::foundation` |
| `mirakan_math_conformance` | なし | golden、property、fuzz、CPU／Shader／Adapter conformance | `mirakan::math`、test-only dependency |
| `mirakan_math_benchmark` | なし | scalar referenceと候補backendの比較 | `mirakan::math`、benchmark-only dependency |

`mirakan::foundation`は`mirakan::math`へ依存しない。Core Services、Authoring Model、Runtime、Renderer、Physics、Navigation、Animation、Asset Import、Editorは必要に応じて両方へ依存できる。Platform／Vendor型との変換は各Adapter private targetが所有し、`mirakan::math`へD3D12、Vulkan、Metal、Box2D、Jolt、Recast型を持ち込まない。

### 4.2 Directory

`engine/foundation/`と`engine/math/`は各一つのcomponent rootであり、[C++23 modules](cpp23-modules.md#7-directoryとsource配置)の標準形（CMakeLists.txt、include、modules、source、tests、benchmarks）へ従う。次の概念dirは`source/`配下の実装内部構成（partitionは`modules/`）であり、合成後の正規treeは本節を正本とする。

```text
engine/
├─ foundation/
│  ├─ modules/
│  └─ source/
│     ├─ result/
│     ├─ diagnostics/
│     ├─ ids/
│     ├─ hash/
│     ├─ binary/
│     ├─ time/
│     ├─ bounded/
│     └─ memory/
└─ math/
   ├─ modules/
   └─ source/
      ├─ scalar/
      ├─ units/
      ├─ linear/
      ├─ transforms/
      ├─ geometry/
      ├─ validation/
      └─ backends/
         ├─ scalar/
         └─ qualified/
```

`foundation/utils`、`math/helpers`、Root `common`を禁止する。新しい機能は所有する概念名へ置き、既存概念に属さなければ新しい小さな責務targetをReviewする。

## 5. 二層の型モデル

### 5.1 Storage math type

Engine内部の計算とSoA／GPU uploadへ使う型である。

| Type | 意味 |
|---|---|
| `Vec2f`、`Vec3f`、`Vec4f` | componentを持つfloat32 vector。位置／方向等の意味を単独では持たない |
| `Vec2i32`、`Vec3i32`、`Vec4i32` | signed integer vector |
| `Vec2u32`、`Vec3u32`、`Vec4u32` | unsigned integer vector |
| `Mat3f`、`Mat4f` | column-major storage、column vector演算 |
| `Quatf` | raw `(x,y,z,w)` storage。公開rotationとして直接受理しない |
| `RectF`、`RectI64`、`Aabb3f` | min inclusive／max exclusiveまたはDomainが明示した閉区間 |

Storage typeはprivate implementation、generated cooked layout、明示されたRuntime Contractで使用できる。AI Tool引数、Authoring Field、Project Saveへ意味なしStorage typeをそのまま公開しない。

### 5.2 Semantic math type

公開境界では次の属性をMCD typeへ必須にする。

```text
semantic_role
dimension
scalar_type
unit
coordinate_space
frame_kind
normalization
valid_range
finite_policy
canonicalization
wire_layout
```

初期公式typeは次とする。

| Semantic type | Storage | 必須意味 |
|---|---|---|
| `WorldPosition2f` | `Vec2f` | 2D World、meter、finite |
| `WorldPosition3f` | `Vec3f` | 3D World、meter、finite |
| `LocalPosition2f`／`LocalPosition3f` | `Vec2f`／`Vec3f` | Owner local space、meter、finite |
| `Displacement2f`／`Displacement3f` | `Vec2f`／`Vec3f` | spaceを明示、meter、finite |
| `Direction2f`／`Direction3f` | `Vec2f`／`Vec3f` | unitなし、zero可否をFieldが明示 |
| `UnitDirection2f`／`UnitDirection3f` | `Vec2f`／`Vec3f` | normalized、zero禁止 |
| `Velocity2f`／`Velocity3f` | `Vec2f`／`Vec3f` | meter／second、space明示 |
| `Acceleration2f`／`Acceleration3f` | `Vec2f`／`Vec3f` | meter／second²、space明示 |
| `Scale2f`／`Scale3f` | `Vec2f`／`Vec3f` | dimensionless、finite、Domain別nonzero制約 |
| `Radians` | `float32` | radian、finite |
| `AngularVelocity3f` | `Vec3f` | radian／second、space明示 |
| `NormalizedQuaternion` | `Quatf` | `(x,y,z,w)`、unit length、canonical sign |
| `Transform2f` | typed fields | translation、radian rotation、scale |
| `Transform3f` | typed fields | translation、normalized rotation、scale |
| `LinearColor4f` | `Vec4f` | scene-linear、rangeはDomain指定 |
| `NormalizedUv2f` | `Vec2f` | Asset API top-left origin、通常`[0,1]` |
| `PixelExtent2u32` | `Vec2u32` | positive pixel count |
| `DurationNs` | `int64` | nanosecond |
| `DurationSeconds` | `float32`／`float64` | second、Domainがprecisionを固定 |

型名だけで全意味を重複表現しない。C++生成型名は可読なsemantic nameを使い、完全な単位、space、range、canonicalizationはMCD descriptorを正本とする。

このCatalogのうち`DurationNs`／`DurationSeconds`の実装Ownerは`mirakan::foundation/time`、`LinearColor4f`の色空間と許容範囲のOwnerはRendering／2D・3D規約である。`mirakan::math`は共通storageと検証演算を提供しても、clock policyまたは色管理policyを所有しない。

### 5.3 公開型の混同禁止

- Position＋Positionを公開Operationとして定義しない。Position＋Displacement→Positionとする。
- Position−Position→Displacementとする。
- UnitDirectionへ任意Vectorを暗黙変換しない。
- degreeとradianを暗黙変換しない。Editor表示だけdegreeへ投影する。
- 2Dと3D、WorldとLocal、UI DIPとWorld meterを暗黙変換しない。
- Colorとgeneric vector、Scaleとgeneric vectorを暗黙変換しない。
- `Quatf`から`NormalizedQuaternion`は検証factoryだけで生成する。
- MatrixからTransformへの暗黙分解を禁止する。

内部hot pathは検証済みsemantic typeをunwrapしてStorage typeを使えるが、別space／unitへ再公開する前にtyped Adapterで再構築する。

## 6. 座標、行列、Quaternion

本Engineの共通規約は次である。

- 3D Worldは右手系、+Y up、object forward +Z、Camera view forward −Z、meter、radian。
- 2D Worldは+X right、+Y up、meter、radian。
- Engine clipはX／Y `[-1,1]`、Z `[0,1]`、reversed-Zでnear 1、far 0。
- vectorはcolumn vector、matrix storageはcolumn-major。
- `world = parent_world * T * R * S`。
- HLSLは`column_major float4x4`と`mul(matrix, vector)`を使い、境界で暗黙transposeしない。
- Quaternion fieldは`x,y,z,w`、`w`がscalar。
- rotation合成は`q_world = q_parent * q_local`。
- `q`と`-q`は、`w > 0`、`w == 0`なら最初の非0成分が正となる側へcanonicalizeする。

`Mat4f`のserialization、GPU buffer、debug printはすべて`m00,m10,m20,m30,m01,...,m33`のcolumn-major順を明記する。constructor引数順、index API、display tableが異なる場合は型名と関数名で区別し、同じ16 scalarを暗黙再解釈しない。

## 7. 数値とfloating-point policy

### 7.1 Precision

| 用途 | 公式scalar |
|---|---|
| C1／C2 Runtime transform、Physics公開値、Rendering、Animation、VFX | `float32` |
| Project／MCD wireの一般物理量 | Domainが指定する`float32`または`float64`、finite必須 |
| Import解析、offline fit、Camera calibration、誤差評価 | 必要な処理だけ`float64` |
| ID、Simulation Advance sequence、size、offset、count | 明示幅integer |
| 金額、固定小数、丸め禁止量 | integer minor unitまたは`decimal_string` |

`double`を「念のため」全Runtimeへ導入しない。`float64`から`float32`へ縮小する境界はrange、rounding、loss metricをConversion Reportへ記録する。

### 7.2 Finiteとcanonicalization

- Public／wire／Saveのfloatはfiniteだけを受理する。
- `-0`はAuthoring、Save、hash、Diagnostic argumentで`0`へcanonicalizeする。
- NaN／Infを0、identity、前回値へ黙って置換しない。
- signaling NaN payload、padding、未初期化byteをhashまたはserializationへ含めない。
- subnormalを一律invalidにせず、Process globalのFTZ／DAZをMath libraryが変更しない。
- Domainがsubnormalを0へclampする場合はthreshold、適用phase、Diagnostic／telemetryをそのDomain Profileへ明示する。

### 7.3 Authoritative profile

Authoritative gameplay、Replay hash、Physics／Navigation入力へ使うC0／C1 CPU mathは次を必須にする。

- MSVCは`/fp:precise`。
- Clangは`-fno-fast-math -ffp-contract=off`。
- `fast-math`、reciprocal近似、暗黙FMA contraction、環境依存rounding modeを使用しない。
- Process global floating-point environmentをSubsystemが変更しない。
- 同一Engine version、同一Platform、同一Build、同一seed、同一inputというRuntime規約のReplay保証範囲を維持する。

異なるCPU architecture、Compiler、Physics kernel間のbitwise一致を初期保証に追加しない。ただしMCD、Save、Cooked Artifact、CPU／Shader boundaryのcanonical valueと意味同等性はTarget別conformanceで検査する。

Presentation-only mathは、CapabilityとTarget Profileに明示し、reference pathとの差をvisual／performance fixtureで測定した場合だけ近似、FMA、SIMD backendを使える。その結果をGameplay、Physics、Saveへ戻さない。

### 7.4 Comparison

- floatの`operator==`はexact value comparisonだけとし、approximate comparisonへしない。
- global `epsilon`を定義しない。
- `is_near_abs_rel(a,b,tolerance)`は呼出側がMCD／Domain Profile由来のtyped toleranceを渡す。
- length、angle、color、matrix、projectionは別tolerance typeを使う。
- Golden testの許容差はTest fixture versionへ固定し、失敗時に自動緩和しない。
- sort keyへapproximate comparisonを使わない。canonical integer key、StableId、quantized key等をDomainが明示する。

## 8. 失敗契約

### 8.1 安全な生成

| Math function | 成功結果 | 失敗 |
|---|---|---|
| `make_unit_direction(v)` | `UnitDirection*` | `NonFinite`、`ZeroLength`、`BelowMinLength` |
| `make_normalized_quaternion(q)` | canonical `NormalizedQuaternion` | `NonFinite`、`ZeroLength`、`OutsideNormalizationTolerance` |
| `try_inverse(matrix)` | inverse | `NonFinite`、`Singular`、`IllConditioned` |
| `try_decompose_trs(matrix)` | typed T／R／S＋reflection情報 | `NonFinite`、`Singular`、`ShearUnsupported`、`UnstableDecomposition` |
| `make_projection(profile)` | typed projection＋culling data | invalid FOV、near／far、viewport、aspect、non-finite |
| `intersect(query, geometry)` | tagged hit／no-hit | invalid geometry、non-finite query、capacity failure |

失敗可能Math functionは`Result<T>`を返す。identity、zero、+X axis、前回値への暗黙fallbackを禁止する。

### 8.2 明示fallback

Presentationでfallbackが必要な場合は`normalize_or(v, fallback_unit_direction)`のようにfallbackを必須引数とし、Math callable descriptorへ`presentation_only=true`、fallback reason、telemetry IDを持たせる。Authoritative codeではこのAPIを使わない。

### 8.3 Diagnostic

初期closed Diagnostic IDは次を含む。

```text
MIRAKAN-MATH-NON_FINITE
MIRAKAN-MATH-ZERO_LENGTH
MIRAKAN-MATH-NOT_NORMALIZED
MIRAKAN-MATH-QUATERNION_SIGN_NON_CANONICAL
MIRAKAN-MATH-SINGULAR_MATRIX
MIRAKAN-MATH-ILL_CONDITIONED_MATRIX
MIRAKAN-MATH-SHEAR_UNSUPPORTED
MIRAKAN-MATH-SPACE_MISMATCH
MIRAKAN-MATH-UNIT_MISMATCH
MIRAKAN-MATH-RANGE_VIOLATION
MIRAKAN-MATH-LAYOUT_MISMATCH
MIRAKAN-MATH-UNQUALIFIED_BACKEND
MIRAKAN-FOUNDATION-BOUNDS
MIRAKAN-FOUNDATION-ENDIAN
MIRAKAN-FOUNDATION-HASH_MISMATCH
```

Diagnosticはfield path、type ID／version、expected unit／space／range、actualのredacted typed value、related requirement、許可されたRemediationを返す。

## 9. Core Utilities

### 9.1 Foundation所有

| 概念 | 公式表現 |
|---|---|
| 成功／失敗 | `Result<T> = std::expected<T, Error>` |
| 永続ID | RFC 9562 UUIDv7 `StableId` |
| Runtime参照 | typed generation handle |
| Hash | content／artifact／replayはSHA-256。非security lookup hashは用途別に明示 |
| Endian | Wire／SaveはFormatごとに固定。暗黙host-endian書込禁止 |
| Binary access | bounded reader／writer、offset＋length、overflow検査 |
| Time | `DurationNs`、clock domain、monotonic／UTCの型分離 |
| Borrow | `span`／view／lease。保存とasync capture禁止 |
| Memory | memory tag、PMR resource、arena／pool ownership |
| Canonical order | comparator ID、stable tie-break、duplicate policy |

### 9.2 所有しない事項

- Cryptographic RNGとcredentialはPlatform Crypto Adapter。
- Authoritative gameplay RNG algorithmはRuntime規約。
- JSON canonicalizationとMCD hash treeは実行可能契約規約。
- Filesystem path resolutionとsandboxはPlatform／AI Governance。
- Unicode normalization、Localization、Text shapingはUI／Text規約。
- Image、Mesh、Audio decodeはAsset Import。

Core UtilitiesがDomain policyを吸収しない。汎用関数へ見えても、意味とfailureがDomain固有ならDomain targetが所有する。

## 10. AI可読Contract

### 10.1 AIが理解する内容

AIへ型名だけを渡さず、MCDから次を投影する。

- semantic role、unit、coordinate space、frame、range、normalization。
- valid、boundary、invalid example。
- 許可Operationと禁止Operation。
- precondition、postcondition、failure code、fallback。
- Authoritative／Presentation、read／write authority、Risk class。
- Target、maturity、qualified backend、budget、cost。
- canonicalizationとserialization form。
- 関連するEditor control、debug visualization、test fixture。

### 10.2 AIの編集単位

AIは`dot`、`cross`、`inverse`等の低水準Math関数を通常のAuthoring Toolとして呼ばない。次の七文字列は将来の意味付きauthoring vocabularyであり、current Operationではない。

```text
operation.transform.set_world_position
operation.transform.set_local_rotation
operation.transform.set_local_scale
operation.camera.set_profile_projection
operation.physics.set_velocity
operation.asset.set_pixels_per_unit
operation.ui.set_rect
```

このうちCamera ownerの`operation.camera.set_profile_projection`は[Executable contracts](executable-contracts.md#211-既存domain文書から回収した未登録operation候補)のCamera family一件だけに属し、残る六件は`planning.operation_family.math_semantic_authoring@1`のexact候補集合である。両familyともCapability stateは`not_activated`、current MCD／Owner Manifest／Service allowlist／Policy／Validator／Diagnostic／Receipt／Provider／MCP／generated alias／legacy alias集合はすべて`[]`である。Gatewayは`activation.math.semantic_authoring_operations.v1`またはCamera側work itemが各familyを完全登録するまでdispatchせず、要求を`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`でSource不変として拒否する。将来ActivationされたGatewayはsemantic typeを検証して必要なMath処理を内部で実行する。VFX／Material／GameplayDefinition等でMath Nodeを編集する場合も、Node Catalogに型、unit、zero policy、cost、fallbackを持たせ、任意式文字列を受理しない。

### 10.3 Discovery

Providerへ全Math APIを常時送らない。MCDから次の階層を生成する。

```text
list_math_domains
describe_math_types(domain, target, maturity)
describe_math_operations(domain, type_ids)
query_math_diagnostic(code)
```

この遅延DiscoveryはTool一覧のtoken量を抑え、現在のTargetとCapabilityに関係する型だけを提示する。これはplanning-only internal discovery actionで、current Tool／MCD Operationではない。将来Activationされた検索結果は`contract_set_hash`とType／planned semantic action descriptor versionを返し、stale Proposalを拒否する。

### 10.4 Projection

同じMCDから次を決定論的に生成する。

- C++ semantic wrapper、factory、validator。
- C ABIのplain wire structとerror code。
- Editor field metadata、unit表示、range、gizmo、debug label。
- Cooked binary descriptor。
- MCP／OpenAI／Anthropicのsubset Schema。
- TypeScript typeとruntime validation descriptor。
- valid／boundary／invalid fixture。
- Documentation tableとAI description。

Provider projectionは正本ではない。Providerが`unit`等のcustom keywordを受理しない場合はdescriptionとclosed enum wrapperへ可逆投影し、C++ Gatewayが元のMCDで完全再検証する。

## 11. 外部Libraryと最適化

### 11.1 Public API

Unity、Unreal Engine、Godot、DirectXMath、GLM、Eigen、Box2D、Jolt等のMath型をMiraikanai public API、MCD、Save、Cooked portable descriptorへ公開しない。

Phase 0の`mirakan::math`は、C++23 standard libraryとportable scalar referenceだけでC0型・演算を実装する。目的は全数学を研究から再発明することではなく、Miraikanaiの意味とfailureを固定する最小referenceを所有することである。

### 11.2 DirectXMath

MicrosoftはDirectXMathをDirectX application向けのSIMD-friendly C++ Mathとして提供し、Windows x86／x64／ARMでSSE、AVX、ARM-NEON最適化を持つ。これはWindows private backend候補として有効だが、Miraikanaiのportable public contractまたはAuthoring表現にはしない。

DirectXMath採用はC2 Dependency／Performance ChangeSetとし、次を満たす場合だけ`engine/math/source/backends/qualified/directxmath/`へ追加する。

- exact source／SDK version、hash、license、Target Profile。
- scalar referenceとの全golden／property／failure一致。
- column-major／column-vectorのEngine意味を変えない明示Adapter。
- Renderer、Animation等の実fixtureで有意なP95またはmemory改善。
- Windows以外のreference pathを退行させない。
- public Header、MCD、Save、ABIへDirectXMath型を露出しない。

### 11.3 SIMD／intrinsic

- Phase 0でcustom SIMD abstractionを作らない。
- Compiler auto-vectorizationを基準にscalar referenceを測定する。
- SSE／AVX／NEON backendはTarget variant、Build Profile、Receiptを別にする。
- SIMD backendはAPIや結果のfailure policyを変更しない。
- alignment、lane padding、inactive laneをserializationまたはhashへ含めない。
- microbenchmarkだけで採用せず、実Subsystemのcritical path改善を必須にする。

## 12. Interchange／Shader境界

### 12.1 glTF

glTF 2.0の右手系、+Y up、+Z forward、meter、radian、XYZW unit quaternion、column-major matrixはMiraikanaiの3D Source conventionと整合する。Importerは無理由なaxis変換やtransposeを行わず、meaning上のfront差、negative determinant、shear、singular、precision lossをConversion Reportへ記録する。

glTF nodeがmatrixを持つ場合、C1はfiniteで安定にTRS分解できる場合だけ受理する。matrixとTRSの同時指定、非unit quaternion、NaN／Inf、cycleを拒否する。

### 12.2 HLSL

Shader boundaryはMCD generated descriptorでCPU field offset、scalar type、matrix layout、alignment、Shader symbolを照合する。

- `column_major`をSourceへ明示する。
- `mul(matrix, vector)`だけをEngine-generated HLSLで使用する。
- raw constant bufferへC++ object representationを`memcpy`しない。generated packerを使う。
- transposeはOperationとして明示し、upload pathの暗黙補正を禁止する。
- CPUとHLSLのgolden transform、projection、normal transformをWARP／reference GPUで比較する。

## 13. TestとQualification

### 13.1 Unit

- vector arithmetic、dot、cross、length、normalize。
- matrix identity、composition、inverse、transpose。
- Quaternion identity、multiply、inverse、rotate、shortest-path slerp、canonical sign。
- TRS compose／decompose、negative scale、non-uniform scale、reflection。
- Rect／AABB／Ray／Plane／Frustumの境界。
- unit、space、range、finite、normalizationのValidator。
- `-0` canonicalization、NaN／Inf rejection。
- Result／Error／Diagnosticのclosed mapping。

### 13.2 Property／fuzz

- `inverse(M) * M`はwell-conditioned fixtureでidentity許容差内。
- normalized quaternionのnormとcanonical sign。
- compose→decompose→composeの意味同等性。
- arbitrary invalid float bit pattern、singular／near-singular matrix。
- integer overflow、offset＋length、alignment、endian round-trip。
- space／unit mismatchがcompileまたはContract validationで拒否される。
- fallback APIが明示fallbackなしでcallできない。

### 13.3 Golden

- identity、translation、axis rotation、non-uniform scale、parent-child transform。
- Camera perspective／orthographic、reversed-Z、viewport。
- glTF 2.0 node TRS／matrix、animation quaternion。
- Box2D／Jolt／Recast Adapterのposition、direction、normal、rotation。
- CPU／HLSL transform、projection、normal matrix。
- MSVC／clang-clの同一Build Profile。
- generated C++／C ABI／TypeScript／binary／MCP round-trip。

Golden更新は仕様変更または妥当性が説明されたBug fixだけに許可し、実装出力へ合わせる自動更新を禁止する。

### 13.4 Performance

次を測定する。

- vector／matrix／quaternion／transform batchのthroughputとlatency。
- Renderer extract、Animation pose、Physics event normalize、Camera updateの実Subsystem寄与。
- instruction count、cache、alignment、code size、compile time。
- scalar reference、Compiler auto-vectorization、候補backendのBefore／After。

Math単体のns改善だけでProduction採用しない。Runtime規約のReference fixtureでcritical pathまたはSubsystem soft capへの寄与を示し、correctness、Replay、visual、memoryを悪化させないことを必須にする。

### 13.5 Target Gate

| Gate | 必須Target |
|---|---|
| M0 Phase 0 | Windows MSVC／clang-cl、scalar reference、MCD round-trip／invalid fixture |
| M1 2D C1 | Windows 2D reference scene、Box2D Adapter |
| M2 3D C1 | Windows RTX 3060／RX 6600、Jolt／Recast／Renderer／Camera、WARP／HLSL Shader conformance |
| M3 Mobile | Android arm64 Vulkan、Apple arm64 Metal、NEON／alignment／thermal |
| M4 Optimized backend | BackendごとのTarget variant、Before／After、fallback |

WARP／HLSL Shader conformanceはD3D12 backend実装（`wp.runtime.d3d12-backend`）を前提とするためM0では実行しない。glTFのmath-level TRS／matrix goldenはM0で実行し、Importer側のconformance closureはAsset Import実装後のentry条件とする。未合格backendはCapability Manifestへ掲載せず、portable scalar referenceを維持する。

## 14. 実装順序

### 14.1 Phase 0 Work Package

Phase 0の`WP0_foundation_measurement`へ次を独立taskとして追加する。

| Task | 成果物 | Gate |
|---|---|---|
| `MATH0_contract` | Math Type／planned semantic action／Diagnostic MCD candidate、generated descriptor | schema、round-trip、invalid fixture |
| `FOUNDATION0_core` | Result、Error、Diagnostic、StableId、Hash、Endian、Duration、bounded reader／writer | unit、property、ASan、no-allocation failure path |
| `MATH1_scalar_reference` | `mirakan::math` C0 storage／semantic typeとportable scalar演算 | unit、property、golden、MSVC／clang-cl |
| `MATH2_transform` | matrix、Quaternion、Transform2／3、projection共通部 | compose、inverse、decompose、reversed-Z |
| `MATH3_projection` | C++／C ABI／Editor／TypeScript／MCP descriptor | deterministic generation、Gateway再検証 |
| `MATH4_conformance` | math-level glTF TRS／matrix golden、HLSL／Adapter conformance fixture定義 | CPU golden、layout、invalid input。Shader／Adapter実行conformanceはM1／M2 entryで合格 |
| `MATH5_baseline` | scalar performance／code size／compile time Receipt | 測定Availability。最適化採用は行わない |

`MATH0_contract`と`FOUNDATION0_core`は並行可能とし、両方の完了を`MATH1_scalar_reference`のEntry Gateにする。その後は`MATH1_scalar_reference → MATH2_transform → MATH3_projection → MATH4_conformance → MATH5_baseline`の順とする。generated C++は`Result`／Diagnostic contract確定後に接続する。

### 14.2 Phase 1～2

- Headless Authoringがsemantic TransformをChangeSet、Save、Load、Replayできる。
- Editor Inspectorがunit、space、range、degree表示、invalid DiagnosticをMCDから生成する。
- `Rect`／AABB／Ray／Plane／FrustumとC1共通Geometryを追加する。
- Render Graph、Camera、Asset Importが同じMath descriptorを使用する。
- Windows private最適化候補はbaseline測定後にprototypeできるが、Promotionしない。

### 14.3 Phase 3以降

- 2D C1でCanvas、Box2D、pixel mapping、Cameraのconformanceを閉じる。
- 3D C1でRenderer、Jolt、Recast、Animation、glTF、HLSLを閉じる。
- MobileでARM64、NEON、Vulkan／Metal layoutをQualificationする。
- C2で実測上位ボトルネックだけにSIMD／Platform backendを追加する。

## 15. AI Eval

次のEvalを3 run実行し、最悪値をGateに使う。

1. World position要求にLocalPositionまたはUI DIPを選択しない。
2. degree入力を正規Wire radianへ変換し、単位を記録する。
3. zero vectorからUnitDirectionを生成せずDiagnosticを返す。
4. non-normalized quaternionを黙ってidentityへ置換しない。
5. singular matrixをTRSとしてCommitしない。
6. Camera、Physics、RendererのTransform authorityを混同しない。
7. raw `Vec3f`ではなくDomain typed change primitiveまたはplanned semantic actionを選択する。
8. stale Type／planned semantic action descriptor versionを再検索する。
9. failure後に許可Remediationだけを提案し、permission／security failureを自動修復しない。
10. 未Qualified SIMD／Platform backendをProductionとして選択しない。

通常CaseのType／planned semantic action選択正答率98%以上、unit／space mismatch Commit成功0、silent fallback 0、存在しないMath action提出0をC0合格条件とする。

## 16. Definition of Done

1. `mirakan::foundation`と`mirakan::math`の依存が一方向でcycleがない。
2. `utils`／`helpers`／`common` targetまたはDirectoryがない。
3. 全公開Math Fieldにunit、space、range、finite、precisionがある。
4. Position、Direction、Velocity、Scale、Color、UVがAI／Authoring境界で区別される。
5. Quaternion、matrix、TRS、projectionのstorage、composition、canonicalizationが一意である。
6. normalize、inverse、decompose等の失敗がtyped `Result`で表現される。
7. silent zero／identity／previous-value fallbackがない。
8. Authoritative mathがlocked floating-point Profileへ合格する。
9. MCDからC++、C ABI、Editor、binary、TypeScript、MCP／Provider projectionが決定論的に生成される。
10. valid、boundary、invalid、property、fuzz、golden fixtureがある。
11. CPU／HLSL、glTF、Physics／Navigation Adapter conformanceが対象Milestoneで合格する。
12. AI Evalがunit／space mismatch Commit成功0を満たす。
13. scalar referenceが常に利用でき、optimized backendを無効化できる。
14. optimized backendはTarget別ReceiptなしにProduction表示されない。
15. Requirement Coverage MatrixがType、planned semantic action candidate、Validator、Diagnostic、test、実装Symbol、Receiptを追跡する。

## 17. 主要リスクと対策

| リスク | 対策 |
|---|---|
| semantic typeが増えAPIが重くなる | 公開境界だけstrong type、hot pathは検証後Storage typeへunwrap |
| AI向けSchemaが巨大になる | Domain／Target／maturity別遅延Discovery |
| scalar referenceが遅い | correctness oracleとして維持し、実測後にprivate backendを追加 |
| Platform math型が公開APIへ漏れる | Adapter-only include／link、public Header scan、generated ABI test |
| toleranceが乱立する | global epsilon禁止、Domain Profileのtyped tolerance |
| SIMDでReplayが変わる | Authoritative／Presentation分離、Build Profile別Receipt、reference comparison |
| matrix conventionが再び分裂する | MCD layout、generated packer、CPU／HLSL golden |
| Mathが巨大な万能Libraryになる | C0最小型、Domain固有algorithmはDomain owner、Work Package別Promotion |
| `utils`が再発する | Directory／namespace lintとDependency Review |

## 18. 有名Engineから採用する原則

| 参照 | 採用する原則 | 採用しないもの |
|---|---|---|
| Unity serialization／`SerializedObject` | 組込み型の機械可読Field、generic編集、Undoへ接続 | `Vector3`だけで位置／方向／速度を兼用する公開意味 |
| Unreal Reflection／Property metadata | Type、editability、range、serialization、Tool Schemaの生成 | Macro／Vendor Object model、Editor-only metadataをRuntime意味の正本にする方式 |
| Unreal MCP Toolset Registry | 小さなTool、structured return、型と説明からSchema、遅延Tool search | unauthenticated Editor server、Tool wrapperの独立正本化 |
| Godot Variant／ClassDB | Math型、Property、Editor、serializationの共通登録 | universal Variantをauthoritative Runtime／AI契約の正本にする方式 |
| DirectXMath | Windows SIMD backendの比較候補 | portable public API、Authoring型、無測定の標準backend化 |
| glTF 2.0 | 右手系、+Y up、+Z forward、meter、radian、XYZW quaternion、column-major interchange | glTF object／JSONをRuntime正本にする方式 |
| HLSL | matrix layoutとmultiply方向の明示、CPU／Shader conformance | Backendが暗黙transposeする方式 |

## 19. 公式資料

- [Unity Serialization rules](https://docs.unity3d.com/Manual/script-serialization-rules.html)
- [Unity `SerializedObject`](https://docs.unity3d.com/ScriptReference/SerializedObject.html)
- [Unreal Engine Metadata Specifiers](https://dev.epicgames.com/documentation/unreal-engine/metadata-specifiers-in-unreal-engine)
- [Unreal Engine Properties](https://dev.epicgames.com/documentation/unreal-engine/unreal-engine-uproperties)
- [Unreal MCP](https://dev.epicgames.com/documentation/unreal-engine/unreal-mcp-in-unreal-editor)
- [Godot Variant](https://docs.godotengine.org/en/stable/engine_details/architecture/variant_class.html)
- [Godot Object／ClassDB／PropertyInfo](https://docs.godotengine.org/en/stable/engine_details/architecture/object_class.html)
- [Microsoft DirectXMath](https://learn.microsoft.com/en-us/windows/win32/dxmath/directxmath-portal)
- [HLSL Specification](https://microsoft.github.io/hlsl-specs/specs/index.html)
- [Khronos glTF 2.0 Specification](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html)
- [C++ working draft `std::expected`](https://eel.is/c++draft/expected)
- [C++ working draft numerics](https://eel.is/c++draft/numerics)

## 20. 非目標

- Computer algebra、symbolic math、arbitrary precision、general scientific computing。
- C0での全Geometry algorithm、全SIMD ISA、全GPU mathの統一。
- 外部EngineのMath API／Asset互換。
- 異なるCPU／Compiler／Physics kernel間の初期bitwise deterministic simulation。
- AIへ任意C++式、HLSL式、raw matrix editorを公開すること。
- Math最適化を理由にGameplay、visual quality、precision、Targetを黙って変更すること。
