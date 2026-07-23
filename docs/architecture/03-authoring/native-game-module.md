# Miraikanai Engine NativeGameModuleアーキテクチャ規約

- 文書ID: mirakan.arch.native-game-module
- 状態: review
- 正本範囲: NativeGameModule artifact／C ABI／entry、公開C++ source境界、lifecycle、Native descriptor、Target別link、Build identity、Preview、Packaging、Native failure、Governance handoff用build evidence
- 非正本範囲: GameplayDefinition、GameSystemSpecV1、System実装選択、typed portsの意味、Project transaction、Toolchain固定値、Runtime scheduling値、Risk分類、Approval／attestation／promotion authorization。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core architecture](../02-foundation/core-architecture.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Naming／Project layout](../02-foundation/naming-project-layout.md)、[C++23 modules](../02-foundation/cpp23-modules.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Performance／capacity](../04-runtime/performance-capacity.md)、[Project state](project-state.md)、[Gameplay programming model](gameplay-programming-model.md)
- 外部根拠検証日: 2026-07-23

## 1. 結論

`NativeGameModule`は、構造化`GameplayDefinition`では表現できないProject固有algorithm、または同一fixtureで必要性を実測したhot pathをC++23で実装する信頼済みProject codeである。一般plugin、Platform SDK bridge、Engine private extension、Script代替ではない。CX0ではModule-ready Header API、CX3ではNamed Modules＋`import std`を使用するが、Process／C ABI／Promotion境界は変えない。

Game制作では`BoundedNativeGameProfileV1`（§5.3）へ適合するModuleだけを許可し、Engine本体、Extension、Adapter、公開SDK、Validator、Policyを変更しない。公開SDKで要求を意味同等に実現できない場合、Native C++で境界を迂回せず`capability_unavailable`とする。

Native implementationは単独のC++ classを正本にせず、active `GameSystemSpecV1`の一つの`Implementation Variant`として登録する。Engine StandardかProject-definedかにかかわらず、State owner、Command／Event／Snapshot、phase、Save／Replay、Target fallback、semantic equivalence fixtureを同じPublic System Contractへ一致させる。

ShippingではProject C++をGame binaryへ静的linkする。Windows Development Previewだけ、同じentry contractを持つDLLを新しい`GameHost` Processの起動時に一度loadできる。in-process unload、binary差替え、live code patchを行わず、変更時はGameHostを終了して再起動する。AndroidではProject static archiveをGame runtime `.so`へ、Appleではstatic archive／objectをapp executableへlinkする。

これによりShipping最適化とattack surface縮小を優先しながら、Windows Editorの反復速度をGameHost再起動で確保する。

公式比較では、Unreal Engine 5.8はProject `Source`のprimary module、Unity 6はnative plug-in境界、GodotはEngine再compileを不要にするGDExtension native libraryをそれぞれ公開している。本設計が採用する共通原則は「固定Engine baseline＋Project-owned extension＋明示ABI／Build／Target別Qualification」だけである。[Unreal Modules](https://dev.epicgames.com/documentation/unreal-engine/unreal-engine-modules?lang=en-US)、[Unity Native plug-ins](https://docs.unity3d.com/6000.0/Documentation/Manual/plug-ins-native.html)、[Godot GDExtension](https://docs.godotengine.org/en/latest/engine_details/engine_api/gdextension/what_is_gdextension.html)を比較Evidenceとし、いずれのAPI互換、plugin ecosystem互換、hot reload挙動、機能同等性も保証しない。MiraikanaiのProject C++は本書のbounded ABI、Engine非改変、Process隔離、static Shipping link、Receipt Gateを正本とする。

C2では、宣言型UIで表現できないProject固有Widgetを`UiNativeWidget`として登録できる。ただしこれは一般Widget pluginではなく、UI規約の型付きManifest、bounded primitive、typed command、Accessibility、fallbackを満たすNativeGameModule Capabilityである。Project codeをEditor Processへloadせず、PreviewはGameHostだけで実行する。

## 2. 決定権と対象外

| 主題 | 正本 |
|---|---|
| C++／GameplayDefinition選択、GameSystemSpecV1、typed Port、System Bundle、Script VM不採用 | [Gameplay programming model](gameplay-programming-model.md) |
| NativeGameModule artifact、ABI、entry、lifecycle、Build、Package | 本書 |
| C++ language、compiler、memory、pointer、exception、target DAG | [C++23 modules](../02-foundation/cpp23-modules.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md) |
| tick phase／fixed delta値、World lease、command／event、queue、failure | [Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md) |
| Source Worker、Risk、Approval、Promotion authorization | [AI Security／Approval](../01-governance/ai-security-approval.md) |
| Game System ID、State owner、Implementation Variant、System Bundle、Target同値性 | [Gameplay programming model](gameplay-programming-model.md) |
| `UiNativeWidget`のproperty、slot、measure、presentation、interaction、semantic、budget、fallback | UI／Text／Localization／Accessibility規約 |

次をNativeGameModuleへ入れない。

- D3D12／Vulkan／Metal、Store、OS、device、filesystem、networkの直接API
- Box2D／Jolt／Recast／ozz／XAudio2等のvendor API
- Engine全Projectで再利用すべきSubsystem
- 汎用reflection、汎用plugin loader、任意console、Script interpreter
- Downloadしたbinary、Runtime生成C++、Runtime compiler
- Anti-cheat、DRM、広告SDK、課金SDK等のPlatform／third-party integration

これらはGame制作Taskの対象外である。必要なCapabilityは別Repository／別AuthorizationのEngine製品開発でのみ検討でき、Game制作TaskはPatchや権限へ自動昇格せず`capability_unavailable`で停止する。

## 3. Artifactとlink方式

| Target／Configuration | Artifact | Load方式 |
|---|---|---|
| Windows Development／Profile Preview | `mirakan_game_<project_slug>.dll`＋同basenameのPDB＋manifest | 新規GameHost startup時に絶対pathとhashを検証後、一度load |
| Windows Shipping | static library／LTO object | `mirakan_game.exe`へ静的link |
| Android全Configuration | static archive／object | appのEngine-owned native `.so`へlink |
| Apple全Configuration | static archive／object | app executableへlink |
| Unit test | static library | test hostへlink |

DLLとShipping static linkは同じgenerated entry header、Capability manifest、conformance suiteを使用する。DLLだけに存在するGameplay機能を禁止する。Preview DLLはEditor Processへloadしない。

Pre-1.0では第三者binary互換性を保証しない。Engine revision、Contract lock、Toolchain lock、CRT、compiler flagsの完全一致を必須とし、不一致binaryをshimで動かさない。Windows CRTは基盤規約の`Development/ASan=/MDd`、`Profile/Shipping=/MD`をEngine、NativeGameModule、static Vendor libraryで統一する。

## 4. Versioned entry contract

### 4.1 C ABI entry

唯一のbinary entryを次の意味で固定する。

```cpp
extern "C" MIRAKAN_NATIVE_EXPORT
MirakanNativeStatus MIRAKAN_NATIVE_CALL
MirakanGetNativeGameModuleV1(
    const MirakanNativeHostDescriptorV1* host,
    MirakanNativeGameModuleDescriptorV1* out_module) noexcept;
```

- Symbol名の`V1`はC ABI majorであり、MCD schema versionとは別である。
- Functionはallocation、thread生成、World access、Capability登録を行わず、descriptorを返すだけとする。
- `host->abi_major != 1`、struct size不足、hash不一致はload前に拒否する。
- static linkでも同じ関数を直接呼ぶ。
- C ABI structは標準layoutの固定幅integer、byte span、opaque handle、function pointerだけを持つ。
- bool、enumのunderlying size、`wchar_t`、`long`、pointer-sized integerをwire fieldに使わない。

### 4.2 Descriptor

`MirakanNativeGameModuleDescriptorV1`は次を持つ。

| Field | 規則 |
|---|---|
| `struct_size` | `uint32` |
| `abi_major`／`abi_minor` | 1／0 |
| `module_stable_id` | 16 byte UUID network order |
| `module_revision_hash` | SHA-256 |
| `contract_lock_hash` | SHA-256 |
| `capability_manifest_hash` | SHA-256 |
| `game_system_dependency_graph_hash` | SHA-256。`GameSystemContractRefV1`↔package内runtime `system_id`対応表を含む |
| `system_implementation_set_hash` | SHA-256 |
| `component_access_manifest_hash` | SHA-256 |
| `required_host_feature_bits` | 登録済みbitのみ |
| `create`／`destroy` | lifecycle function |
| `prepare_play`／`stop_play` | session lifecycle |
| `system_table` | phase-bound system descriptor span |
| `capability_table` | typed Capability implementation span |

Descriptor内の文字列は診断表示用UTF-8 byte spanだけとし、identityやdispatchに使わない。未知feature bit、重複System ID、未宣言Capability、許可外phaseはloadを拒否する。

### 4.3 Host、Create、Invoke context

`MirakanNativeHostDescriptorV1`はentry呼出中だけ有効な値Viewであり、次を持つ。

| Field | 規則 |
|---|---|
| `struct_size`、`abi_major`／`abi_minor` | `uint32`、1／0 |
| `engine_revision_hash` | 32 byte |
| `engine_public_api_hash` | 32 byte |
| `contract_lock_hash` | 32 byte |
| `target_profile_hash` | 32 byte |
| `platform_id`／`configuration` | MCD closed `uint32` enum |
| `host_feature_bits` | 登録済みbitだけ |

ModuleはentryからHost pointer／spanを保持しない。`create`時に別の`MirakanNativeCreateContextV1`を渡し、module instance ID、immutable config byte span、persistent `MirakanNativeMemoryPortV1`、typed Diagnostic Portを提供する。config spanは`create` returnまでだけ有効で、必要な値はpersistent Portへcopyする。Memory／Diagnostic Portのcontextとfunction tableは`destroy`開始まで有効である。

| `MirakanNativeCreateContextV1` Field | 規則 |
|---|---|
| `struct_size` | `uint32` |
| `module_instance_handle` | session-local opaque fixed-width handle |
| `config_schema_id`／`config_schema_version` | MCD generated ID／`uint32` |
| `config_bytes`／`config_hash` | call中だけ有効なcanonical byte span／SHA-256 |
| `persistent_memory` | `MirakanNativeMemoryPortV1`を値で保持 |
| `diagnostics` | `MirakanNativeDiagnosticPortV1`を値で保持 |

各System `invoke`には`MirakanNativeInvokeContextV1`を渡す。tick、phase、fixed delta numerator／denominator、immutable query batch、RNG state view、phase-private scratch Port、typed output writerを持ち、callback returnで全View／scratch／writerを無効化する。Project C++がpointer、span、writer、scratch allocationを次のtick、job、callbackへ保持することを禁止し、Development lease epochで検出する。

| `MirakanNativeInvokeContextV1` Field | 規則 |
|---|---|
| `struct_size`、`system_id` | `uint32`。`system_id`はCooked package内だけで有効なgenerated runtime ID、0 invalid |
| `tick_id`、`invoke_sequence` | `uint64` |
| `phase_id` | Runtime規約のserialized `TickPhaseId` |
| `fixed_delta_numerator`／`fixed_delta_denominator` | `uint32`。Runtime Ownerが供給する既約有理数を読み取り、Native側で既定値を選ばない |
| `query_batches` | ComponentAccessManifestから生成したimmutable bounded View |
| `rng_stream` | Engine-owned deterministic RNG Port |
| `scratch_memory` | callback returnまで有効なsingle-owner Port |
| `output_writer` | 宣言済みCommand／Event／Structural Commandだけを受理するprivate writer。Component value／System State deltaは受理しない |
| `lease_epoch` | return時にinvalidateする`uint64` |

`MirakanNativeMemoryPortV1`を次に固定する。

| Field | 規則 |
|---|---|
| `struct_size` | `uint32` |
| `context` | Host-owned opaque `void*`。Project codeはdereferenceしない |
| `memory_domain_id` | registered fixed-width ID |
| `hard_limit_bytes` | このPortから同時に保持できる上限 |
| `allocate` | `(context, size: uint64, alignment: uint32, tag_id: uint32) -> void*` |
| `deallocate` | `(context, ptr, size: uint64, alignment: uint32, tag_id: uint32) -> void` |

`size`は1以上、`alignment`は1～4096の2冪とし、失敗時は`allocate`がnullを返す。FunctionはC ABIを越えて例外を投げず、persistent PortはEngine workerからの並行callに対応し、invoke scratch Portは当該invokeだけのsingle-ownerである。`deallocate`は取得時と同じPort、size、alignment、tagを必須とし、不一致はDevelopment assertion／Release session faultとする。Moduleはfunction tableをcopyできるが、context lifetimeを越えてcallしない。

`MirakanNativeDiagnosticPortV1`は`{struct_size, context, emit}`だけを持つ。`emit`は`diagnostic_code: uint32`、closed severity enum、system ID、tick／phase、最大8個のMCD scalar argument、任意のUTF-8 detail最大1,024 byteを値copyし、例外を投げない。Detail、表示名、log textをGame rule、dispatch、success判定へ使用せず、Secret、User text、pointer、pathを記録しない。1 invoke当たり64 recordを超えた分はcounterへ集約し、callbackをblockしない。

## 5. C++ source contract

### 5.1 公開API

Project sourceが宣言できるEngine依存は次のPrimary Named Moduleだけとする。

```text
mirakan.foundation
mirakan.runtime.contracts
mirakan.gameplay
mirakan.native_game
mirakan.project.contracts
std
```

`CppDependencySetV1`へpublic／private import、closed `StdHeaderId`、closed Header例外を記録する。CX0は上記論理依存を`include/mirakan/`、個別標準Header、`<build>/generated/mirakan/project_contracts/`へ投影し、CX3はPrimary Named Moduleと`import std;`へ投影する。`engine/**/source`、vendor header、Platform header、generated backend binding、Editor headerをinclude pathへ加えない。CIはCX0のinclude graph／preprocessor trace、CX1以降のModule dependency scan／ASTを検査する。

MCDから生成するProject C++ APIは次を提供する。

- typed Component read view
- typed immutable snapshot
- `CommandWriter`、`EventWriter`、`StructuralCommandWriter`
- Manifestのaccessに応じたgenerated `MirakanNativeStateViewV1`（read／read_write）
- versioned Asset handleとmetadata view
- deterministic RNG stream
- fixed C ABIのbounded scratch memory port
- telemetry counter／span
- typed Result／Error

公開APIはEngine object pointerを返さない。`EntityHandle`、`AssetVersionHandle`、`NativeSystemHandle`はindex＋generationまたはopaque fixed-width valueであり、保存や別session再利用を禁止する。

### 5.2 STL、RTTI、Exception

| 項目 | 規則 |
|---|---|
| STL | Module内部で使用可。CX1以降は原則`import std;`。ABI struct、callback parameter、Engine container ownershipへ出さない |
| RTTI | Compilerは基盤規約どおり有効。Engine reflection、serialization、Capability discoveryへ使わない |
| Exception | Module内部で使用可。ただし全generated trampolineでcatchしtyped `NativeModuleError`へ変換。ABI／Subsystem boundaryを越えない |
| Allocation | Engineは`MirakanNativeMemoryPortV1 {context, allocate, deallocate}`の固定C function tableを渡す。Moduleは必要ならこれをmodule-owned `std::pmr::memory_resource` Adapterで包むが、PMR objectを境界へ渡さない |
| Global state | immutable compile-time data以外のmutable global／function staticを禁止 |
| Thread | Moduleによる作成、detach、thread-local service cacheを禁止 |
| I/O | filesystem、socket、process、clockへの直接accessを禁止 |

Project C++の明示的な`new`／`delete`、`malloc`／`free`を禁止する。Module-owned persistent objectは`MirakanMakePersistent<T>(MirakanNativeMemoryPortV1, tag_id, args...) -> Result<MirakanUniqueOwner<T>>`だけで生成し、per-tick allocationはphase scratch Portだけを使用する。`MirakanUniqueOwner<T>`はmove-onlyで、取得時のPort、size、alignment、tagを保持し、destructorで`T`を一度破棄して同じPortへblockを返す。copy、`release()`による所有raw pointer流出、別Portへの移管を提供しない。

Module内部のPMR containerはMemory Portを包むmodule-owned `std::pmr::memory_resource` Adapterをconstructorで受け取る。通常の`std::make_unique`、default PMR resource、global operator newによってpersistent Portを迂回しない。cross-boundary objectをcaller側でdeleteせず、Shipping static linkでもこの規則を緩和しない。

### 5.3 `BoundedNativeGameProfileV1`

本書はGame制作の許可判定に使う`BoundedNativeGameProfileV1`の唯一のDomain ownerである。ProfileはEngine buildが生成するread-only制約集合であり、Project／Game制作AIは変更できない。Governance文書は本Profileを参照だけする。

| Field | 規則 |
|---|---|
| `schema_version`／`profile_id` | MCD共通Envelopeに従うgenerated ID |
| `engine_public_api_hash` | 適合検査対象のEngine公開APIをexactに固定するSHA-256 |
| `allowed_named_modules[]` | §5.1のPrimary Named Module集合のclosed subset |
| `allowed_std_header_ids[]` | `CppDependencySetV1`のclosed `StdHeaderId` allowlist |
| `forbidden_operation_rule_ids[]` | §5.2のGlobal state／Thread／I-O／明示的allocation禁止に対応するclosed rule ID集合 |
| `memory_limit_refs[]` | `MirakanNativeMemoryPortV1`の`hard_limit_bytes`等、上限値へのexact参照 |
| `gate_ids[]` | §12のSource scan／manifest一致検査を含む検査Gate ID集合 |

Profileの各Fieldは本書の各節が所有する規則へのexact参照であり、閾値・上限のNormative値を複写しない。適合はSource Gate（§12のAST／Module graph／link import scan）とload時照合（§7.1のGraph照合）で機械検査し、不適合ModuleはGame制作Taskで登録しない。

## 6. Lifecycle

```text
Discovered
  -> ContractValidated
  -> Created
  -> SystemsRegistered
  -> PlayPrepared
  -> Running
  -> PlayStopping
  -> SystemsUnregistered
  -> Destroyed
```

| Transition | 許可処理 | 禁止 |
|---|---|---|
| `create` | Module context、bounded persistent memory、immutable config copy | World query、thread、file、GPU／Physics access |
| register | System／Capability descriptor登録 | Entity作成、Runtime state変更 |
| `prepare_play` | Target／Asset／Component access検証、session-local state確保 | tick実行、async job開始 |
| Running | 宣言phaseのcallback、typed output作成 | phase外write、再入、構造変更直接実行 |
| `stop_play` | 新規job停止、owned job join、state破棄準備 | timeout後のbackground access |
| unregister | System／Capability登録解除 | callback残留 |
| `destroy` | 全Module memory解放 | Engine service呼出し、例外伝播 |

`prepare_play`が失敗した場合はPlayを開始せず、`stop_play`を呼ばずにunregister／destroyする。Running中faultは当該Play sessionを停止し、同Process内でModule instanceを再利用しない。

## 7. System registrationとphase

### 7.1 Gameplay System

`NativeSystemDescriptorV1`は次を必須とする。

| Field | 規則 |
|---|---|
| `system_id` | Cooked package内runtime `uint32` ID、generated。永続化／別Package比較禁止 |
| `system_contract_version` | `GameSystemSpecV1.version`からgenerated |
| `implementation_variant_hash` | Source、generated binding、manifest、configを結ぶSHA-256 |
| `phase_mask` | 重複なしの`TickPhaseId[1..16]`。`GameSystemSpecV1`からphase ordinal順に生成された許可集合だけを消費し、Native側でphaseを追加しない |
| `read_component_set` | ComponentAccessManifest subset |
| `write_state_set` | GameplayState field subset |
| `command_set`／`event_set` | 生成可能な型のsubset |
| `max_instances` | finite hard bound |
| `scratch_bytes` | phaseごとのhard bound |
| `budget_us` | Profile soft budget |
| `determinism_class` | `authoritative \| presentation_only` |
| `state_owner_set_hash` | Specのowned State Type集合と一致 |
| `invoke` | generated no-throw trampoline |

`GameSystemSpecV1.state_class`から`determinism_class`への写像は閉じる。`authoritative`は`authoritative`、`derived`と`presentation_only`は`presentation_only`へ写像する。ただし`derived`はauthoritative Component／Stateへのwrite access、authoritative Command target、Save field所有を一件も持たない場合だけ登録できる。`tooling_only`はGameHostの`NativeSystemDescriptorV1`へ登録せず、Editor-only presentation経路を使う。Spec、Manifest、descriptorの写像不一致をLoad時にModule全体の登録失敗とし、より強いauthorityへ暗黙昇格しない。

Orchestratorだけがcallbackを呼ぶ。Load時にSystem ID、Contract version、Variant hash、State owner、phase、Component access、Command／Event集合をactive `GameSystemDependencyGraphV1`と照合し、一件でも不一致ならModule全体を登録しない。callback inputはtick、fixed delta、immutable query batches、snapshot、RNG streamで、outputはprivate bounded bufferである。World commitは成功後にRuntime規約のcanonical merge順で行う。Module callbackが部分的にCommandを書いてから失敗した場合、そのinvokeの全outputを破棄する。

Moduleがworker処理を必要とする場合、Engine Job Portへbounded jobを提出する。JobはWorld viewをcaptureせず、owned input、owner generation、deadline tickを持ち、結果はRuntime Ownerが定める結果portと安全境界で検査される。Module独自worker poolを作らない。

### 7.2 Game UI extension

`UiNativeWidget`は`NativeSystemDescriptorV1`へUI phaseを追加して実装しない。NativeGameModuleの`capability_table`へ登録する`capability.ui.native_widget`から、UI規約の`UiNativeWidgetManifestV1`と次のpure callback tableを取得する。schema versionはManifest側のschema versionで表し、capability IDへversion suffixを埋め込まない。

| Callback | Owner phase | Input | Output |
|---|---|---|---|
| `measure` | UI Layout role | typed property、available size、Asset intrinsic metrics、bounded scratch | finite min／preferred／max size |
| `build_presentation` | UI Presentation role | final rect、typed property、presentation-only ViewModel field、Asset handle、bounded scratch | whitelist済みprimitive／Effect parameter |
| `handle_interaction` | UI Interaction role | semantic event、local state、typed property、bounded scratch | registered `UiCommandId`／local UiState delta |
| `build_semantics` | Accessibility role | typed property、final rect、state | bounded semantic node descriptor |

各callbackはUI Runtimeだけがcanonical Node ID順で呼び、World、ECS、Renderer、GPU、Platform、filesystem、networkへ到達させない。callback inputのView、scratch、writerはreturn時にinvalidateし、保持をDevelopment lease epochで検出する。clock、RNG、thread、async job、global mutable stateを禁止し、同じinputから同じoutputを生成しなければならない。

`measure`は子Widgetを直接走査せず、UI Runtimeが渡すnamed slotの集約済みmetricsだけを読む。`build_presentation`はraw vertex pointerを受け取らず、UI規約のcapacity付きprimitive encoderだけを使う。`handle_interaction`はGameplay stateやWorldを変更せず、typed commandを返す。`build_semantics`を省略できるのはManifestが明示`decorative_only=true`でfocus、input、meaningを持たない場合だけである。

一callbackがerror、exception、timeout、non-finite、capacity超過を返した場合、そのcallbackの全outputを破棄する。Development PreviewはGameHostをfault停止し、EditorへDiagnosticとfallback Previewを返す。ShippingはManifestのfallback Widgetへ切り替え、required Screenに有効なfallbackがなければUI規約のsafe fallback screenへ遷移する。Native Widget fault後に同じModule instanceを再使用しない。

## 8. GameplayDefinitionとの併用

Capabilityの公開contractは実装方式に依存させない。

- `GameplayDefinition`とNativeGameModuleは同じGame System ID、Capability ID、command、event、snapshot、Save fieldを使用する。
- 一System内でDefinitionがparameter／state machine、Native C++がalgorithm／batch kernelを担当できる。
- DefinitionからC++ function名、pointer、vtable indexを参照しない。
- Native実装への昇格でStable State ID、Save field ID、event意味を変更しない。
- 二つの実装を同時にauthoritative writerとして登録しない。

実装方式の切替は検証済み`SystemImplementationSetV1`をPlay開始時に選択する。互換なDefinition-only swap以外をPlay中に自動切替せず、Native変更は新しいGameHostを起動する。

## 9. Build、検証、Promotion

### 9.1 Build identity

Native artifact keyは最低限、次のhashから作る。

```text
source_tree_hash
generated_contract_hash
game_system_dependency_graph_hash
system_implementation_set_hash
native_module_manifest_hash
engine_public_api_hash
cpp_frontend_profile_hash
cpp_dependency_set_hash
module_graph_hash
build_driver_profile_hash
build_tree_identity_hash
contract_lock_hash
toolchain_lock_hash
target_profile_hash
configuration
```

absolute path、user、timestampをobjectの意味入力にしない。Target別`BuildDriverProfileV1`に従い、WindowsはNinja Multi-Config、AndroidはGradle→Single-Config Ninja、Apple Module archiveはNinja Multi-ConfigでBuildする。Makefiles／`ndk-build`、Generator override、異なるBuild tree identityのartifactをbuild evidenceへ混在させない。Primary MSVCとsecondary Clangによるcompile、format、warning-as-error、static analysis、unit、ASan、integration、conformanceをclean Build treeで行う。

### 9.2 Build evidenceとGovernance handoff

Native buildは`NativeModuleBuildEvidenceV1`として次のdomain evidenceだけを出力する。

- Source delta／source tree hashとRequirement mapping
- generated Module／C ABI contract hash、Dependency Set、manifest、Capability／access／budget
- build identity、Target、Configuration、Toolchain lock、artifact hash
- primary／secondary compile、format、warning、static analysisの結果
- unit、property、fuzz、integration、replay、save／load、ASan、contract conformanceの結果
- 同一fixtureのdeterminism、semantic equivalence、performance測定と入力hash
- sandbox、resource limit、failure／cancellation status

`NativeModuleBuildEvidenceV1`はbuild事実のimmutable envelopeであり、Risk、Approval、review attestation、promotion、activationを決定しない。それらの分類、署名者、失効、authorizationは[AI Security／Approval](../01-governance/ai-security-approval.md)と[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)が所有する。`SystemBundleChangeSetV1`と実装切替は[Gameplay programming model](gameplay-programming-model.md)、Projectへの登録とCommitは[Project state](project-state.md)が所有する。

Source、generated Module／C ABI Header、Dependency Set、Build artifact、build evidenceのhashが一つでも一致しなければload候補にしない。BMI hash自体はArtifact identityにせず、Toolchain／Configurationを含む破棄可能CacheとしてC++言語・Modules規約どおり分離する。Governanceからauthorizationが返らないartifactはinactiveに保ち、Projectは直前のQualified Variantを参照し続ける。

### 9.3 AI生成SourceとCode owner gate

AIが新規Native Sourceを生成または既存Native Sourceを変更する前に、[AI Security／Approval](../01-governance/ai-security-approval.md#94-code-owner-assignmentとapproval)の`CodeOwnerAssignmentV1`をexact module／path scopeへ解決する。Assignmentは`role_ref=role.code_owner.native_module`、対象Native module／pathを全量包含するScope、current qualification／independence Receipt、`valid_from <= evaluation_time < expires_at`、`revoked_at=null`をすべて満たさなければならない。Assignment不在、Role欠落／unknown／Shader Role、期限切れ、失効、Scope外、qualification／independence Receipt不成立ではTaskを`AwaitingCodeOwner`、Editorを`awaiting_code_owner`にし、Source Workerを起動しない。

Source生成後は、同じSource revision、exact Diff hash、Target別Build Receipt集合、独立`review_receipt_ref`に対する`CodeOwnerApprovalV1.decision=approved`をPromotion前に必須とする。Diff、base Source revision、generated contract、Dependency Set、Toolchain、Target、Build artifactのいずれかが変われば再Build／再Review／再Approvalする。Gameplay intent承認、AI Reviewer、Compiler成功、prequalified Packの過去承認を新しいDiffのCode owner承認として流用しない。

Beginner MVPはDefinition-firstと既にQualification済みのPack／Variantだけを使用し、新規Native Sourceを生成しない。既存Packはexact package／module hash、license、Target、qualification、revocationを照合し、変更なしで利用する。RequirementがDefinition／Packで成立しない場合はNativeを暗黙生成せず`capability_unavailable`とする。Advanced Project Source ActivationはこのCode owner gate、Source sandbox、G0–G7、Promotionを独立に満たす場合だけ有効であり、Beginner First Playableの合格をActivation根拠にしない。

## 10. PreviewとPackage

Windows Preview sequenceを次で固定する。

1. Editorが現在のCommit済みProject revisionをBuild requestに固定する。
2. 隔離Workerが新しいartifact directoryへDLL、PDB、manifestを出す。
3. Native build検証合格後、Governance authorizationを受けたPreview artifactをread-only publishする。
4. 旧GameHostへgraceful stopを要求し、timeout時はProcessを終了する。
5. 新GameHostがEngine／Contract／Module hashを検証してstartup loadする。
6. Project packageをloadし、Play sessionを新規作成する。
7. 失敗時はEditor状態を維持し、必要なら直前のPreview artifactで別GameHostを起動できる。

Save互換検証なしに旧Play stateを新Processへ移さない。Preview artifactをShipping directoryへcopyして昇格しない。Shippingはclean sourceからstatic link、LTO、symbol split、package signingを再実行する。

## 11. Failure policy

| Failure | 結果 |
|---|---|
| ABI／hash／feature不一致 | Process起動時に拒否、Module codeを呼ばない |
| entry exception／SEH | artifact invalid、継続しない |
| create／prepare失敗 | Play開始せずdestroy |
| callback typed error | invoke output破棄。authoritativeならPlay session fault |
| scratch／queue上限超過 | fallback heapを使わずtyped budget fault |
| timeout／job残留 | Play stopをfault、Process終了 |
| invalid handle／lease | command reject、authoritative invariantならsession fault |
| DLL file lock／load failure | Editorは継続、last valid artifactを明示表示 |
| Mobile C++変更 | rebuild、re-sign、reinstallなしのPreviewを禁止 |
| 未宣言import／未許可Header／Module cycle | Source Gate失敗、Header方式へFallbackしない |
| `import std`／BMI／Module tooling不成立 | Active C++ Frontend Profile失敗、artifactを生成しない |
| Code owner Assignment不在／Role欠落・unknown・Shader Role／失効／Scope外 | Source Workerを起動せず`AwaitingCodeOwner`。BeginnerはDefinition／prequalified Packへ再Plan |
| Code owner ApprovalのDiff／revision／Receipt不一致 | Promotion／load拒否、Sourceはinactive Stagingに隔離 |
| Native Widget manifest／Capability／Target不一致 | Widget callback登録前にreject、UI規約のfallbackへ遷移 |
| Native Widget callback fault／timeout／capacity超過 | callback output全破棄、GameHost fault、同Module instance再利用禁止 |

CrashしたProject C++はEngine memoryへ到達可能な信頼済みCodeであり、runtime sandboxで安全化されたとは表現しない。

## 12. Performance、Memory、Security Gate

- Native callbackの時間はSystem ID別に測定し、[Performance／capacity](../04-runtime/performance-capacity.md)が所有するGameplay Logic合計budgetを緩和しない。
- Runtime callback内allocation countはsteady stateで0をC1目標とし、許可allocationはphase scratchからだけ行う。
- Persistent、session、scratch、command payloadを別telemetry counterへchargeする。
- C++化の成立条件は[Gameplay programming model](gameplay-programming-model.md) §1.1の選択境界に従い、性能根拠は[Performance／capacity](../04-runtime/performance-capacity.md)が所有する改善閾値を同一fixtureで満たすこと、または表現不能Capabilityだけとする。閾値のNormative値を本書へ複写しない。
- ASLR、DEP、CFG、CET互換、stack protection、warnings-as-errors等のWindows Shipping hardeningをEngine binaryと同じにする。
- Module Sourceに未宣言import、禁止Header、inline assembly、dynamic load、socket、process、environment／registry accessがないことをAST／Module graph／link import scanで検査する。
- Shipping import table、symbol、Capability manifest、Component access manifestの一致を検証する。

## 13. TestとDefinition of Done

- static linkとWindows startup DLLが同一conformance fixtureを通る。
- C ABI struct size、alignment、unknown minor、wrong major、truncated tableをfuzzする。
- STL／exception／allocator／RTTI objectがABIを越えないことをheader／symbol scanする。
- createからdestroyまで各transitionへfailure injectionし、callback／job／allocationが残らない。
- phase外write、未宣言Component、invalid handle、queue overflowを拒否する。
- Definition実装とNative実装でcommand、event、Save、replay意味が一致する。
- Native descriptorのSystem ID、Contract version、Variant、State owner、phase、accessがactive Game System Graphと一致しない場合にloadを拒否する。
- Target-specialized Variantが同じPublic System ContractとGameplay fidelity fixtureを通り、意味同等fallbackなしのTargetを非対応にする。
- Governance authorization後Project Commit失敗時に新Variantをloadせず、直前Qualified Variantを維持する。
- GameHostを100回再起動し、Editor Processのhandle／memoryが増加しない。
- Phase 5 C1ではWindows Editor PreviewとWindows Desktopのclean static-link packageが同じModule revision hashを記録する。Android／AppleはProduct Registryで`excluded`のまま、別のTarget Profile、Work Package、Capability Target binding、fresh Target Qualificationが承認されるまで本C1完了条件と対応表示へ含めない。
- AI生成SourceがGovernance authorization前に正規Project／Editor／Shippingへloadされない。
- AI生成Sourceはexact `role.code_owner.native_module`、Native Scope、current Qualification、`revoked_at=null`を持つ`CodeOwnerAssignmentV1`なしに生成されず、exact Diffの`CodeOwnerApprovalV1`なしにPromotion／loadされない。missing／unknown／`role.code_owner.project_shader`／Scope差／revokedを一原因ずつ拒否する。
- Beginner Profileでは新規Native Source Taskが0件で、Definition／prequalified Pack不能な要求を`capability_unavailable`として停止する。
- CX3ではEngine C++ Public Headerをincludeせず、`CppDependencySetV1`、実際のimport、CMake DAGが一致する。
- Native artifactがTarget別`BuildDriverProfileV1`とBuild tree identityを記録し、Make／Ninja二重経路を持たない。
- C2 `UiNativeWidget`はManifest、ABI、pure callback、determinism、primitive cap、Accessibility、fallback、GameHost fault isolationを全Target fixtureで検証する。

本Native CapabilityのC1完了条件は、Advanced Project Source Activation下の2D縦切りで一つのProject固有CapabilityをNativeGameModuleへ実装し、Code owner gate、Windows Editor Preview再起動、Windows Desktop clean static-link artifact、Definitionとのcontract conformance、fault recoveryをすべて合格することである。これはBeginner MVP／First PlayableのCompletion Gateでも、Shipping／Release readinessでもない。

C2 UI extension完了条件は、一つの宣言型では表現不能なWidgetを`UiNativeWidget`として実装し、Governance-authorized source、UI Designer fallback projection、Windows GameHost Preview、Windows／Android／Apple static-link package、semantic／layout／render golden、fault recoveryを合格することである。これはC1 NativeGameModule完了条件を置き換えない。

## 14. 一次資料

- [ISO C++ object model and language reference](https://isocpp.org/std/the-standard)
- [Microsoft DLL best practices](https://learn.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-best-practices)
- [Android NDK C++ library support](https://developer.android.com/ndk/guides/cpp-support)
- [Apple Distributing binary frameworks](https://developer.apple.com/documentation/xcode/distributing-binary-frameworks-as-swift-packages)

外部の汎用plugin ABIを採用せず、C ABIはWindows Development startup loadと静的linkの共通validation入口だけに限定する。
