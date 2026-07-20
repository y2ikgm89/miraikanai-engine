# Miraikanai Engine NativeGameModuleアーキテクチャ規約

- 文書版: 1.7
- 作成日: 2026-07-19
- 最終更新日: 2026-07-20
- 対象: Project C++、source／binary境界、entry point、lifecycle、Game UI extension、Build、Preview、Packaging
- 状態: プロジェクト公式の規範設計レビュー版
- Game実装規約: [Miraikanai Engine C++実行コード・構造化ゲームデータ規約](./2026-07-19-cpp-structured-game-data-design.md)
- 基盤規約: [Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)
- C++言語・Modules規約: [Miraikanai Engine C++23・Named Modules・`import std`移行規約](./2026-07-20-cpp23-modules-import-std-transition-design.md)
- Engine命名正本: [Miraikanai Engine AI可読命名・技術識別子規約](./2026-07-20-ai-readable-engine-naming-convention-design.md)
- Game Project配置・命名正本: [Miraikanai Engine AI可読Game Project配置・命名規約](./2026-07-20-ai-readable-game-project-layout-naming-design.md)
- Authoring規約: [Miraikanai Engine Authoring Model／Project State規約](./2026-07-19-authoring-model-project-state-design.md)
- Runtime規約: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- Memory／Pointer規約: [Miraikanai Engine AI可読Memory／Pointerアーキテクチャ規約](./2026-07-20-ai-readable-memory-pointer-architecture-design.md)
- Platform規約: [Windows](./2026-07-19-windows-platform-distribution-design.md)／[Mobile](./2026-07-19-mobile-platform-architecture-design.md)
- Game System規約: [Miraikanai Engine Game System／AI Code Generationアーキテクチャ規約](./2026-07-20-game-system-ai-codegen-architecture-design.md)

## 1. 結論

`NativeGameModule`は、構造化`GameplayDefinition`では表現できないProject固有algorithm、または同一fixtureで必要性を実測したhot pathをC++23で実装する信頼済みProject codeである。一般plugin、Platform SDK bridge、Engine private extension、Script代替ではない。CX0ではModule-ready Header API、CX3ではNamed Modules＋`import std`を使用するが、Process／C ABI／Promotion境界は変えない。

Native implementationは単独のC++ classを正本にせず、active `GameSystemSpecV1`の一つの`Implementation Variant`として登録する。Engine StandardかProject-definedかにかかわらず、State owner、Command／Event／Snapshot、phase、Save／Replay、Target fallback、semantic equivalence fixtureを同じPublic System Contractへ一致させる。

ShippingではProject C++をGame binaryへ静的linkする。Windows Development Previewだけ、同じentry contractを持つDLLを新しい`GameHost` Processの起動時に一度loadできる。in-process unload、binary差替え、live code patchを行わず、変更時はGameHostを終了して再起動する。AndroidではProject static archiveをGame runtime `.so`へ、Appleではstatic archive／objectをapp executableへlinkする。

これによりShipping最適化とattack surface縮小を優先しながら、Windows Editorの反復速度をGameHost再起動で確保する。

C2では、宣言型UIで表現できないProject固有Widgetを`UiNativeWidget`として登録できる。ただしこれは一般Widget pluginではなく、UI規約の型付きManifest、bounded primitive、typed command、Accessibility、fallbackを満たすNativeGameModule Capabilityである。Project codeをEditor Processへloadせず、PreviewはGameHostだけで実行する。

## 2. 決定権と対象外

| 主題 | 正本 |
|---|---|
| C++／GameplayDefinition選択、Performance閾値、Script VM不採用 | Game実装規約 |
| NativeGameModule artifact、ABI、entry、lifecycle、Build、Package | 本書 |
| C++ language、compiler、memory、pointer、exception、target DAG | 基盤規約 |
| tick phase、World lease、command／event、queue、failure | Runtime規約 |
| Source Worker、Risk R3、Approval、Promotion | AI実装・保守ガバナンス規約 |
| Game System ID、State owner、Implementation Variant、System Bundle、Target同値性 | Game System規約 |
| `UiNativeWidget`のproperty、slot、measure、presentation、interaction、semantic、budget、fallback | UI／Text／Localization／Accessibility規約 |

次をNativeGameModuleへ入れない。

- D3D12／Vulkan／Metal、Store、OS、device、filesystem、networkの直接API
- Box2D／Jolt／Recast／ozz／XAudio2等のvendor API
- Engine全Projectで再利用すべきSubsystem
- 汎用reflection、汎用plugin loader、任意console、Script interpreter
- Downloadしたbinary、Runtime生成C++、Runtime compiler
- Anti-cheat、DRM、広告SDK、課金SDK等のPlatform／third-party integration

これらはEngine CapabilityまたはPlatform AdapterとしてR4で別設計する。

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
| `fixed_delta_numerator`／`fixed_delta_denominator` | `uint32`、C1は1／60 |
| `query_batches` | ComponentAccessManifestから生成したimmutable bounded View |
| `rng_stream` | Engine-owned deterministic RNG Port |
| `scratch_memory` | callback returnまで有効なsingle-owner Port |
| `output_writer` | 宣言済みCommand／Event／State deltaだけを受理するprivate writer |
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
- `CommandWriter`、`EventWriter`、`GameplayStateTransaction`
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
| `phase_mask` | `T30`、`T40`、`T70`の許可組合せ。`T00`等へ直接登録不可 |
| `read_component_set` | ComponentAccessManifest subset |
| `write_state_set` | GameplayState field subset |
| `command_set`／`event_set` | 生成可能な型のsubset |
| `max_instances` | finite hard bound |
| `scratch_bytes` | phaseごとのhard bound |
| `budget_us` | Profile soft budget |
| `determinism_class` | `authoritative \| presentation_only` |
| `state_owner_set_hash` | Specのowned State Type集合と一致 |
| `invoke` | generated no-throw trampoline |

Orchestratorだけがcallbackを呼ぶ。Load時にSystem ID、Contract version、Variant hash、State owner、phase、Component access、Command／Event集合をactive `GameSystemDependencyGraphV1`と照合し、一件でも不一致ならModule全体を登録しない。callback inputはtick、fixed delta、immutable query batches、snapshot、RNG streamで、outputはprivate bounded bufferである。World commitは成功後にRuntime規約のcanonical merge順で行う。Module callbackが部分的にCommandを書いてから失敗した場合、そのinvokeの全outputを破棄する。

Moduleがworker処理を必要とする場合、Engine Job Portへbounded jobを提出する。JobはWorld viewをcaptureせず、owned input、owner generation、deadline tickを持ち、結果は`T20`で検査される。Module独自worker poolを作らない。

### 7.2 Game UI extension

`UiNativeWidget`は`NativeSystemDescriptorV1`へUI phaseを追加して実装しない。NativeGameModuleの`capability_table`へ登録する`capability.ui.native_widget_v1`から、UI規約の`UiNativeWidgetManifestV1`と次のpure callback tableを取得する。

| Callback | Owner phase | Input | Output |
|---|---|---|---|
| `measure` | `T90` UI Layout | typed property、available size、Asset intrinsic metrics、bounded scratch | finite min／preferred／max size |
| `build_presentation` | `T90` UI Presentation | final rect、typed property、presentation-only ViewModel field、Asset handle、bounded scratch | whitelist済みprimitive／Effect parameter |
| `handle_interaction` | `T30` UI Interaction | semantic event、local state、typed property、bounded scratch | registered `UiCommandId`／local UiState delta |
| `build_semantics` | `T90` Accessibility | typed property、final rect、state | bounded semantic node descriptor |

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

absolute path、user、timestampをobjectの意味入力にしない。Target別`BuildDriverProfileV1`に従い、WindowsはNinja Multi-Config、AndroidはGradle→Single-Config Ninja、Apple Module archiveはNinja Multi-ConfigでBuildする。Makefiles／`ndk-build`、Generator override、異なるBuild tree identityのartifactをPromotionしない。Primary MSVCとsecondary Clangによるcompile、format、warning-as-error、static analysis、unit、ASan、integration、conformanceをclean Build treeで行う。

### 9.2 RiskとPromotion

AI生成またはAI変更の`NativeCodeChangeSet`はR3である。Source Workerはnetwork deny、Job Object／sandbox、path broker、CPU／memory／time capを持つ。Build成功は正規ProjectへのPromotion権限ではない。

Promotionには次を必須とする。

1. Source DiffとRequirement mapping
2. 生成／変更したCapability、access、budgetのmanifest
3. Primary／secondary compileとstatic analysis
4. Unit、property、fuzz、integration、replay、save／load
5. 10分×3回の同一fixture performance比較
6. Security／code-owner Review
7. signed Promotion Receipt
8. `SystemBundleChangeSetV1`の全hashとexpected dependency graphを照合
9. Authoring規約の`RegisterNativeModuleRevision`＋`SetSystemImplementationVariant` Commit

Source、generated Module／C ABI Header、Dependency Set、Build artifact、Receiptのhashが一つでも一致しなければloadしない。BMI hash自体はArtifact identityにせず、Toolchain／Configurationを含む破棄可能CacheとしてC++言語・Modules規約どおり分離する。

Source PromotionとProject Commitを一つの原子的transactionと偽らない。Source Promotion後にProject Commitが失敗したrevisionはinactiveのまま保持し、現在Projectが参照する直前のQualified Variantをloadする。再試行またはrevertは同じBundle hashと新しいReview Receiptを必要とする。

## 10. PreviewとPackage

Windows Preview sequenceを次で固定する。

1. Editorが現在のCommit済みProject revisionをBuild requestに固定する。
2. 隔離Workerが新しいartifact directoryへDLL、PDB、manifestを出す。
3.全Gate合格後にPreview artifactをread-only publishする。
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
| Native Widget manifest／Capability／Target不一致 | Widget callback登録前にreject、UI規約のfallbackへ遷移 |
| Native Widget callback fault／timeout／capacity超過 | callback output全破棄、GameHost fault、同Module instance再利用禁止 |

CrashしたProject C++はEngine memoryへ到達可能な信頼済みCodeであり、runtime sandboxで安全化されたとは表現しない。

## 12. Performance、Memory、Security Gate

- Native callbackの時間はSystem ID別に測定し、Gameplay Logic合計P95 1.50 msを緩和しない。
- Runtime callback内allocation countはsteady stateで0をC1目標とし、許可allocationはphase scratchからだけ行う。
- Persistent、session、scratch、command payloadを別telemetry counterへchargeする。
- C++化は同一fixtureで構造化実装より5%以上かつmeasurement noiseを超える改善、または表現不能Capabilityを根拠とする。
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
- Source Promotion後Project Commit失敗時に新Variantをloadせず、直前Qualified Variantを維持する。
- GameHostを100回再起動し、Editor Processのhandle／memoryが増加しない。
- Windows Shipping、Android、Appleのclean static-link packageが同じModule revision hashを記録する。
- AI生成SourceがPromotion前に正規Project／Editor／Shippingへloadされない。
- CX3ではEngine C++ Public Headerをincludeせず、`CppDependencySetV1`、実際のimport、CMake DAGが一致する。
- Native artifactがTarget別`BuildDriverProfileV1`とBuild tree identityを記録し、Make／Ninja二重経路を持たない。
- C2 `UiNativeWidget`はManifest、ABI、pure callback、determinism、primitive cap、Accessibility、fallback、GameHost fault isolationを全Target fixtureで検証する。

C1完了条件は、2D縦切りで一つのProject固有CapabilityをNativeGameModuleへ実装し、Windows Preview再起動、Shipping static link、Definitionとのcontract conformance、fault recoveryをすべて合格することである。

C2 UI extension完了条件は、一つの宣言型では表現不能なWidgetを`UiNativeWidget`として実装し、AI生成SourceのR3 Promotion、UI Designer fallback projection、Windows GameHost Preview、Windows／Android／Apple static-link package、semantic／layout／render golden、fault recoveryを合格することである。これはC1 NativeGameModule完了条件を置き換えない。

## 14. 一次資料

- [ISO C++ object model and language reference](https://isocpp.org/std/the-standard)
- [Microsoft DLL best practices](https://learn.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-best-practices)
- [Android NDK C++ library support](https://developer.android.com/ndk/guides/cpp-support)
- [Apple Distributing binary frameworks](https://developer.apple.com/documentation/xcode/distributing-binary-frameworks-as-swift-packages)

外部の汎用plugin ABIを採用せず、C ABIはWindows Development startup loadと静的linkの共通validation入口だけに限定する。
