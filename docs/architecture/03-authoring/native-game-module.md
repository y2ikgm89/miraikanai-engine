# Miraikanai Engine NativeGameModuleアーキテクチャ規約

- 文書ID: mirakan.arch.native-game-module
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: NativeGameModule artifact／C ABI／entry、公開C++ source境界、Public API Catalogとsubject stability、lifecycle、Native descriptor、Target別link、Build identity、Preview、Packaging、Native failure、Governance handoff用build evidence
- 非正本範囲: GameplayDefinition、GameSystemSpecV1、System実装選択、Project test semantics、SDK配布／license／Documentation bundle、typed portsの意味、Project transaction、Toolchain固定値、Runtime ECS storage・query・access manifest、Runtime scheduling値、Risk分類、Approval／attestation／promotion authorization。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Project State](project-state.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[C++23 Language／Public Surface](../02-foundation/cpp23-modules.md)、[Gameplay Programming Model](gameplay-programming-model.md)
- 関連文書: [Runtime ECS Static Definition／Entity Reference Boundary Decision](../decisions/2026-08-03-runtime-ecs-static-and-entity-reference-boundary.md)、[Product Plan](../00-product/product-plan.md)、[Product Lifecycle](../00-product/product-lifecycle.md)、[Product Security](../01-governance/product-security.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[Core architecture](../02-foundation/core-architecture.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Naming／Project layout](../02-foundation/naming-project-layout.md)、[C++23 Language／Public Surface](../02-foundation/cpp23-modules.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Developer Testing](developer-testing.md)、[Runtime ECS](../04-runtime/entity-component-system.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Performance／capacity](../04-runtime/performance-capacity.md)、[Project state](project-state.md)、[Gameplay programming model](gameplay-programming-model.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-23

## 1. 結論

`NativeGameModule`は、構造化`GameplayDefinition`では表現できないProject固有algorithm、または同一fixtureで必要性を実測したhot pathをC++23で実装する信頼済みProject codeである。一般plugin、Platform SDK bridge、Engine private extension、Script代替ではない。Initial V1は[C++23 Language／Public Surface](../02-foundation/cpp23-modules.md)の単一Header-based Shipping APIを使用し、Named Modules、`import std`、BMIを使用しない。Process／C ABI／Promotion境界はC++ Source公開方式と独立する。

Game制作では`BoundedNativeGameProfileV1`（§5.3）へ適合するModuleだけを許可し、Engine本体、Extension、Adapter、公開SDK、Validator、Policyを変更しない。公開SDKで要求を意味同等に実現できない場合、Native C++で境界を迂回せず`capability_unavailable`とする。

Native implementationは単独のC++ classを正本にせず、active `GameSystemSpecV1`の一つの`Implementation Variant`として登録する。Engine StandardかProject-definedかにかかわらず、State owner、Command／Event／Snapshot、phase、Save／Replay、Target fallback、semantic equivalence fixtureを同じPublic System Contractへ一致させる。

ShippingではProject C++をGame binaryへ静的linkする。Windows Development Previewだけ、同じentry contractを持つDLLを新しい`GameHost` Processの起動時に一度loadできる。in-process unload、binary差替え、live code patchを行わず、変更時はGameHostを終了して再起動する。AndroidではProject static archiveをGame runtime `.so`へ、Appleではstatic archive／objectをapp executableへlinkする。

これによりShipping最適化とattack surface縮小を優先しながら、Windows Editorの反復速度をGameHost再起動で確保する。

外部要求の`Hot Reload`は、制作中のC++変更を迅速にPreviewへ反映する意味に限って本restart-based sequenceへ解決する。in-process object layout移行、DLL unload／replacement、live code patchまたは旧Play stateの無検証継続を必須機能にしない。[Product Lifecycle](../00-product/product-lifecycle.md)はこのPreview qualificationとShipping static-link Receiptを同じEngine release／Project revisionのDeveloper journeyへ束縛するconsumerであり、別のHot Reload Capabilityまたはload経路を作らない。

code signature、publisher certificate、content hash、provenance Receiptはartifact identityとintegrityのEvidenceであり、load権限、Engine private API、Target対応を付与するCapabilityではない。Shipping packageはPackage closureの外からdownload、追加、差替えられたnative code、汎用plugin DLL、任意pathのbinary、JIT生成codeをloadしない。Windows Development Preview DLLもexact Project／Engine／Contract／Toolchain／Target identity、Governance authorization、read-only artifact directoryを満たす一件だけを新規GameHost startup時にloadし、未署名codeをShipping、Release、MOD配布へ昇格させない。

[Product Plan](../00-product/product-plan.md)の`future.capability.unrestricted-project-scripting-runtime`は将来scopeを分解するincubation umbrellaであり、本書のC ABI、Preview DLL、static linkからsigned AOT native extension、dynamic plugin、sandbox VM、JIT対応を推測しない。将来のsigned AOT desktop extensionが独立Future Capabilityとして成立しても本`NativeGameModule`と別artifact family、別Owner、別ABI／trust／Target／Qualificationを持つ。interpreted／bytecode MODとJITはNativeGameModuleではない。

公式比較では、Unreal Engine 5.8はProject `Source`のprimary module、Unity 6.3 LTSはUnity 6 familyのnative plug-in境界、Godot 4.7.1はEngine再compileを不要にするGDExtension native libraryをそれぞれ公開している。本設計が採用する共通原則は「固定Engine baseline＋Project-owned extension＋明示ABI／Build／Target別Qualification」だけである。[Unreal Modules](https://dev.epicgames.com/documentation/unreal-engine/unreal-engine-modules?lang=en-US)、[Unity Native plug-ins](https://docs.unity3d.com/6000.0/Documentation/Manual/plug-ins-native.html)、[Godot 4.7 GDExtension](https://docs.godotengine.org/en/4.7/tutorials/scripting/gdextension/what_is_gdextension.html)を比較Evidenceとし、いずれのAPI互換、plugin ecosystem互換、hot reload挙動、機能同等性も保証しない。MiraikanaiのProject C++は本書のbounded ABI、Engine非改変、Process隔離、static Shipping link、Receipt Gateを正本とする。

Product C2では、宣言型UIで表現できないProject固有Widgetを`UiNativeWidget`として登録できる。ただしこれは一般Widget pluginではなく、UI規約の型付きManifest、bounded primitive、typed command、Accessibility、fallbackを満たすNativeGameModule Capabilityである。Project codeをEditor Processへloadせず、PreviewはGameHostだけで実行する。

## 2. 決定権と対象外

| 主題 | 正本 |
|---|---|
| C++／GameplayDefinition選択、GameSystemSpecV1、typed Port、System Bundle、Script VM不採用 | [Gameplay programming model](gameplay-programming-model.md) |
| NativeGameModule artifact、ABI、entry、lifecycle、Build、Package | 本書 |
| C++ language、compiler、public surface、memory、pointer、exception、target DAG | [C++23 Language／Public Surface](../02-foundation/cpp23-modules.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md) |
| Simulation Advance／Cadence、phase、World lease、command／event、queue、failure | [Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md) |
| Source Worker、Risk、Approval、Promotion authorization | [AI Security／Approval](../01-governance/ai-security-approval.md) |
| Game System ID、State owner、Implementation Variant、System Bundle、Target同値性 | [Gameplay programming model](gameplay-programming-model.md) |
| `UiNativeWidget`のproperty、slot、measure、presentation、interaction、semantic、budget、fallback | UI／Text／Localization／Accessibility規約 |

次をNativeGameModuleへ入れない。

- D3D12／Vulkan／Metal、Store、OS、device、filesystem、networkの直接API
- Box2D／Jolt／Recast／ozz／XAudio2等のvendor API
- Engine全Projectで再利用すべきSubsystem
- 汎用reflection、汎用plugin loader、任意console、Script interpreter
- Downloadしたbinary、package closure外の追加／差替えbinary、Runtime生成C++、Runtime compiler、JIT
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

`CppValueTransferPolicyV1`はC ABI shapeを直接生成しない。C ABIは固定幅standard-layout value、opaque handle、C function table、明示pointer＋byte-or-element count、caller-owned output bufferだけを使う。C++ reference、`std::span`、STL／PMR、`Result<T>`、exception、owner wrapperをABIへ出さない。

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

各System `invoke`には`MirakanNativeInvokeContextV1`を渡す。step、phase、`MirakanNativeCadenceInputV1`、immutable query batch、RNG state view、phase-private scratch Port、typed output writerを持ち、callback returnで全View／scratch／writerを無効化する。Project C++がpointer、span、writer、scratch allocationを次のstep、job、callbackへ保持することを禁止し、Development lease epochで検出する。

```text
MirakanNativeTypedContentRefV1
  struct_size: uint32
  type_id: generated package-local uint32
  type_version: uint32
  type_contract_set_hash: uint8[32]
  canonical_ref_bytes: call-lifetime canonical byte span

MirakanNativeCadenceInputV1
  struct_size: uint32
  cadence_profile_id: generated package-local uint32
  cadence_profile_version: uint32
  cadence_profile_content_hash: uint8[32]
  simulation_advance_interval_hash: uint8[32]
  advance_sequence: positive uint64
  cadence:
    kind: fixed
      interval_seconds_numerator: positive uint64
      interval_seconds_denominator: positive uint64
    | kind: variable
      interval_seconds_numerator: positive uint64
      interval_seconds_denominator: positive uint64
      monotonic_sample_sequence: positive uint64
    | kind: turn_based
      accepted_advance_command_ref: MirakanNativeTypedContentRefV1
      accepted_advance_command_sha256: uint8[32]
    | kind: explicit_step
      accepted_step_request_ref: MirakanNativeTypedContentRefV1
      accepted_step_request_sha256: uint8[32]
      request_step_ordinal: positive uint16
      logical_interval_seconds:
        null | {numerator: positive uint64, denominator: positive uint64}
```

`cadence`はRuntime Scheduling ownerの`SimulationCadenceProfileV1`と同じclosed discriminatorを使い、別branch FieldをABI payloadへ混在させない。共通Profile identity、`simulation_advance_interval_hash`、`advance_sequence`とbranch値は同じsealed `SimulationAdvanceIntervalV1`から投影し、fixed／variable intervalとexplicit-stepのnon-null logical intervalは既約な正の有理秒である。variableの`monotonic_sample_sequence`も同Intervalからdirect projectionし、Native側でclockをsampleまたは採番しない。`MirakanNativeTypedContentRefV1.canonical_ref_bytes`はScheduling record内のexact `SimulationAdvanceControlRefV1`を`McdCanonicalBinaryV1`でencodeしたcall-lifetime View、`type_id`は同Refの`control_type_ref`をPackage tableへ解決したlocal IDであり、version／Contract set hashも同Type refとbyte equalityにする。turn-basedのCommand Ref／SHA-256、explicit-stepのrequest Ref／SHA-256／ordinalはsealed Intervalの同Fieldから一対一で複写し、独自sequence、display ID、bare hash、直前requestから推測しない。Project C++はRef spanをcallback後へ保持または再解決せず、Profile、kind、interval、advance sequence、Command／request identityを選択または既定化しない。profile IDはPackage内dispatch用で永続化せず、completed Profile refのversion／content hashをBuild、Replay、Save、Qualificationとbyte equalityにする。

| `MirakanNativeInvokeContextV1` Field | 規則 |
|---|---|
| `struct_size`、`system_id` | `uint32`。`system_id`はCooked package内だけで有効なgenerated runtime ID、0 invalid |
| `advance_sequence`、`invoke_sequence` | `uint64`。前者は`cadence_input.advance_sequence`とbyte equality |
| `phase_id` | Runtime規約のserialized `TickPhaseId` |
| `cadence_input` | call中だけ有効なexact `MirakanNativeCadenceInputV1`。ProfileのrateまたはkindをABI literalから推測しない |
| `query_batches` | [Runtime ECS](../04-runtime/entity-component-system.md)の`RuntimeComponentAccessManifestV1`と`RuntimeQueryDispatchPlanV1`から生成したimmutable bounded View |
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

`MirakanNativeDiagnosticPortV1`は`{struct_size, context, emit}`だけを持つ。`emit`は`diagnostic_code: uint32`、closed severity enum、system ID、advance sequence／phase、最大8個のMCD scalar argument、任意のUTF-8 detail最大1,024 byteを値copyし、例外を投げない。Detail、表示名、log textをGame rule、dispatch、success判定へ使用せず、Secret、User text、pointer、pathを記録しない。1 invoke当たり64 recordを超えた分はcounterへ集約し、callbackをblockしない。

## 5. C++ source contract

### 5.1 公開API

Project sourceが宣言できるEngine依存は次のlogical public dependency IDだけとする。これらはC++ Named Module名ではなく、`CppDependencySetV1`からPublic Header include rootとCMake targetへ一意投影するidentityである。

```text
mirakan.foundation
mirakan.runtime.contracts
mirakan.gameplay
mirakan.native_game
mirakan.project.contracts
std
```

`CppDependencySetV1`へpublic／private dependency、closed `StdHeaderId`、closed Header例外を記録する。Initial V1は上記論理依存を`include/mirakan/`、個別標準Header、`<build>/generated/mirakan/project_contracts/`へ一意投影する。`engine/**/source`、vendor header、Platform header、generated backend binding、Editor headerをinclude pathへ加えない。CIはinclude graph、preprocessor trace、standalone Header compile、AST public surfaceを検査し、Named Module dependencyまたはBMIを入力にしない。

Generated C++ adapterは[Memory／Pointers](../02-foundation/memory-pointers.md)の`CppValueTransferPolicyV1`、`PointerContractV1`、`MemoryContractV1`へexact解決する。C ABIのpointer／count／alignment／call lifetimeを検証してcall-scope `std::span`またはgenerated bounded viewへ変換し、return後に保持しない。`unique_owner`／`move_sink`はABIを越えず、Memory Port、opaque handle、caller-owned bufferを使う。

Persistent factoryはdefault allocator、global `new`、default PMRへfallbackせず、exact Memory Portと`unique_owner` policyを使う。C++ adapterのprivate measured exceptionはC ABI validationまたはpublic C++ policyを緩和しない。

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

公開APIはEngine object pointerを返さない。`RuntimeEntityHandle`、`AssetVersionHandle`、`NativeSystemHandle`はindex＋generationまたはopaque fixed-width valueであり、保存や別session再利用を禁止する。callbackまたはSimulation Advanceを越えて同じruntime sessionのcurrent Entityを指す参照は[Runtime ECS](../04-runtime/entity-component-system.md)の`RuntimeEntityLiveRefV1`、seal済みpublicationへ固定する参照は`RuntimeEntitySnapshotRefV1`へ投影する。どちらもSave、Replay、Packageまたはcross-session identityにしない。

Native ABIは[Runtime ECS](../04-runtime/entity-component-system.md)のinitial V1 refとbounded projectionを直接参照し、本文書のC ABI規約だけから公開済みconsumer、old readerまたはcompatibility windowが存在すると推測しない。初回materialization後にABIを変更する場合だけ`native_abi` consumerとして[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)へrecordし、承認済みclassなしにABI aliasまたは旧handle projectionを追加しない。

#### 5.1.1 Public API Catalogとsubject stability

公開SDKへ含まれるHeader、namespace、type、function、constant、enum、error、callback、generated binding、build requirementは一件残らず`PublicApiCatalogV1`へ列挙する。Documentation、Sample、Project test、SDK distributionはこのCatalogと同じ`PublicContractSetV1`を消費し、include path、symbol export、autocompleteまたは使用例からpublic statusを推測しない。

```text
PublicApiSubjectV1
  api_subject_id: StableId
  api_subject_version: positive u32
  subject_kind:
    header | namespace | type | function | constant
    | enum | error | callback | generated_binding | build_requirement
  canonical_cpp_name: ASCII qualified name
  owning_public_contract_member_ref: exact PublicContractMemberRefV1
  declaration_artifact_ref: exact ArtifactSliceRefV1
  stability: stable | preview
  availability_profile_refs[1..64]:
    sorted unique exact PublicApiAvailabilityProfileRefV1
  ownership_contract_ref: exact PublicApiOwnershipContractRefV1
  threading_contract_ref: exact PublicApiThreadingContractRefV1
  error_contract_ref: exact PublicApiErrorContractRefV1
  documentation_entry_refs[1..64]:
    sorted unique exact DocumentationEntryRefV1
  api_subject_content_hash: SHA-256

PublicApiCatalogV1
  public_api_catalog_id: StableId
  public_api_catalog_version: positive u32
  public_contract_set_ref: exact PublicContractSetRefV1
  subject_refs[1..65535]:
    sorted unique exact PublicApiSubjectRefV1
  forbidden_include_root_refs[1..64]:
    sorted unique exact ArtifactPathPolicyRefV1
  catalog_content_hash: SHA-256
```

各exact `{public_contract_set_ref, target_profile_ref, toolchain_lock_sha256}`について、公開C++ callable集合を`C`、callable `f`のparameter数を`p(f)`、non-void returnを持つ時だけ1となる値を`r(f)`とする。Catalog publication前にchecked arithmeticで

```text
sum(f in C, p(f) + r(f)) <= 65536
```

を検証し、この左辺の各要素を[Memory／Pointers](../02-foundation/memory-pointers.md)のexactly one `CppValueTransferPolicyV1.bindings[]` rowへset equalityで投影する。上限超過、parameter／returnのmissing・orphan・duplicate bindingをCatalog全体のrejectionにし、複数Policyへの暗黙sharding、Catalog subsetまたはTarget／Toolchainを違えるPolicyの合成で上限を回避しない。

`stable` subjectは公開releaseで通常利用できる契約であり、意味、ownership、threading、error、Target availabilityを変更する時は[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)のexternal API assessmentを必要とする。`preview` subjectはProduction依存の既定選択にせず、exact release、Target、既知制限、終了／昇格条件をDocumentationへ表示する。Previewであっても同じrelease内でCatalogから消したり、同名で意味を差し替えたりしない。

最初のmaterialized／公開SDKより前のdesign revisionはCompatibility Ownerのclean initial version規則に従い、current Catalog、Subject、Refをすべて`V1`として直接定義する。旧draft alias、compatibility Header、deprecated wrapper、dual declarationを作らない。公開後のsubject removal／rename／signature／layout／exception／ownership／threading変更は、consumer inventoryとfinite migration／deprecation decisionなしに行わない。

Catalogのclosureは少なくとも次を検証する。

- declaration artifactの全public symbolがexactly one subjectへ解決し、Catalogだけのghost subjectがない。
- public C++ name、C ABI entry、generated binding、Documentation、Sample、Project testが同じcontract memberへ解決する。
- Engine private、Editor、vendor、Platform、test-only declarationがpublic include／link surfaceへ漏れない。
- Target／configurationで利用不能なsubjectはcompile-time typed diagnosticまたはavailability queryで明示し、空実装、no-op、別Backendへ意味変更しない。
- ownership、lifetime、nullability、thread／phase、allocation、error、determinism、Save可否がすべてbounded contractへ解決する。
- Catalog hash、public API hash、SDK artifact、Native build identity、Engine release bindingがsame candidateで一致する。

Public APIのsource互換、binary互換、semantic互換、Save／wire互換を一つの`compatible` Booleanへ畳み込まない。NativeGameModuleのShipping static linkはEngine releaseごとのrebuildを許せるが、これをsource／semantic breakの許可またはDocumentation省略の根拠にしない。

### 5.2 STL、RTTI、Exception

| 項目 | 規則 |
|---|---|
| STL | Source／private implementationで使用可。必要な個別標準Headerを直接includeし、ABI struct、callback parameter、Engine container ownershipへ出さない |
| RTTI | Compilerは基盤規約どおり有効。Engine reflection、serialization、Capability discoveryへ使わない |
| Exception | Module内部で使用可。ただし全generated trampolineでcatchしtyped `NativeModuleError`へ変換。ABI／Subsystem boundaryを越えない |
| Allocation | Engineは`MirakanNativeMemoryPortV1 {context, allocate, deallocate}`の固定C function tableを渡す。Moduleは必要ならこれをmodule-owned `std::pmr::memory_resource` Adapterで包むが、PMR objectを境界へ渡さない |
| Global state | immutable compile-time data以外のmutable global／function staticを禁止 |
| Thread | Moduleによる作成、detach、thread-local service cacheを禁止 |
| I/O | filesystem、socket、process、clockへの直接accessを禁止 |

Project C++の明示的な`new`／`delete`、`malloc`／`free`を禁止する。Module-owned persistent objectは`MirakanMakePersistent<T>(MirakanNativeMemoryPortV1, tag_id, args...) -> Result<MirakanUniqueOwner<T>>`だけで生成し、per-Simulation-Advance allocationはphase scratch Portだけを使用する。`MirakanUniqueOwner<T>`はmove-onlyで、取得時のPort、size、alignment、tagを保持し、destructorで`T`を一度破棄して同じPortへblockを返す。copy、`release()`による所有raw pointer流出、別Portへの移管を提供しない。

Module内部のPMR containerはMemory Portを包むmodule-owned `std::pmr::memory_resource` Adapterをconstructorで受け取る。通常の`std::make_unique`、default PMR resource、global operator newによってpersistent Portを迂回しない。cross-boundary objectをcaller側でdeleteせず、Shipping static linkでもこの規則を緩和しない。

### 5.3 `BoundedNativeGameProfileV1`

本書はGame制作の許可判定に使う`BoundedNativeGameProfileV1`の唯一のDomain ownerである。ProfileはEngine buildが生成するread-only制約集合であり、Project／Game制作AIは変更できない。Governance文書は本Profileを参照だけする。

| Field | 規則 |
|---|---|
| `schema_version`／`profile_id` | MCD共通Envelopeに従うgenerated ID |
| `engine_public_api_hash` | 適合検査対象のEngine公開APIをexactに固定するSHA-256 |
| `allowed_public_dependency_ids[]` | §5.1のlogical public dependency ID集合のclosed subset |
| `allowed_std_header_ids[]` | `CppDependencySetV1`のclosed `StdHeaderId` allowlist |
| `forbidden_operation_rule_ids[]` | §5.2のGlobal state／Thread／I-O／明示的allocation禁止に対応するclosed rule ID集合 |
| `memory_limit_refs[]` | `MirakanNativeMemoryPortV1`の`hard_limit_bytes`等、上限値へのexact参照 |
| `gate_ids[]` | §12のSource scan／manifest一致検査を含む検査Gate ID集合 |

Profileの各Fieldは本書の各節が所有する規則へのexact参照であり、閾値・上限のNormative値を複写しない。適合はSource Gate（§12のAST／include graph／link import scan）とload時照合（§7.1のGraph照合）で機械検査し、不適合ModuleはGame制作Taskで登録しない。

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
| `prepare_play` | Target／Asset／Component access検証、session-local state確保 | Simulation Advance実行、async job開始 |
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
| `read_component_set` | `RuntimeComponentAccessManifestV1`のread setかつcallbackのquery dispatch selectionに一致するsubset |
| `write_component_set` | `RuntimeComponentAccessManifestV1`のwrite setかつcallbackのquery dispatch selectionに一致するsubset |
| `structural_permission_set` | Manifestのoperation kind、target Component／transition、template、apply boundary、count／byte budgetに一致するsubset |
| `read_state_set` | `GameSystemSpecV1`とstate store bindingが許可するGameplayState field subset |
| `write_state_set` | `GameSystemSpecV1`のexact active State ownerに属するGameplayState field subset |
| `command_set`／`event_set` | 生成可能な型のsubset |
| `max_instances` | finite hard bound |
| `scratch_bytes` | phaseごとのhard bound |
| `budget_us` | Profile soft budget |
| `determinism_class` | `authoritative \| presentation_only` |
| `state_owner_set_hash` | Specのowned State Type集合と一致 |
| `invoke` | generated no-throw trampoline |

`GameSystemSpecV1.state_class`から`determinism_class`への写像は閉じる。`authoritative`は`authoritative`、`derived`と`presentation_only`は`presentation_only`へ写像する。ただし`derived`はauthoritative Component／Stateへのwrite access、authoritative Command target、Save field所有を一件も持たない場合だけ登録できる。`tooling_only`はGameHostの`NativeSystemDescriptorV1`へ登録せず、Editor-only presentation経路を使う。Spec、ECS Manifest、generated implementation binding、descriptorのread Component、write Component、structural permission、read／write State集合を正逆方向に検査し、descriptorがManifest権限を拡張する場合だけでなく、generated bindingが要求するaccessをdescriptorが欠落させる場合もLoad時にModule全体の登録失敗とする。より強いauthorityへの暗黙昇格、State writeをComponent writeとして代用すること、structural permissionを汎用commandとして代用することを禁止する。

Orchestratorだけがcallbackを呼ぶ。Load時にRuntime Packageのexact `GameSystemDependencyGraphRefV1`、`SystemImplementationSetRefV1`、`GameStateOwnerProjectionRefV1`を[Gameplay Programming Model §3.0.1](gameplay-programming-model.md#301-system-graphimplementation-setstate-owner-projection)へ解決し、System ID、Contract version、Variant hash、State owner、phase、Component read／write access、structural permission、State read／write access、Command／Event集合をbyte-exactに照合する。一件でも不一致ならModule全体を登録しない。callback inputはstep sequence、exact Cadence Profile identity、closed cadence branch input、immutable query batches、snapshot、RNG streamで、outputはprivate bounded bufferである。authoritative runtime state／Command／Event publicationはcallback成功後だけRuntime規約のcanonical merge順で行い、Worldを持たないUI-only／headless branchにも同じ規則を適用する。Module callbackが部分的にCommandを書いてから失敗した場合、そのinvokeの全outputを破棄する。

旧`fixed_delta_numerator／fixed_delta_denominator` Fieldはtarget `MirakanNativeInvokeContextV1`へ含めない。ABI freeze前のClock／Cadence TaskはSource／descriptor／Replay／fixtureを`MirakanNativeCadenceInputV1`へ同時更新し、reference default `60/1 Hz`をProfile instanceとしてだけ検証する。旧Fieldを互換alias、全Game System、UI-only、headless、将来Cadenceの恒久前提として残さない。

Moduleがworker処理を必要とする場合、Engine Job Portへbounded jobを提出する。JobはWorld viewをcaptureせず、owned input、owner generation、deadline advance sequenceを持ち、結果はRuntime Ownerが定める結果portと安全境界で検査される。Module独自worker poolを作らない。

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

実装方式の切替は検証済み`SystemImplementationSetRefV1`をPlay開始時に選択し、Packageが束縛した完成`SystemImplementationSetV1`以外を採用しない。互換なDefinition-only swap以外をPlay中に自動切替せず、Native変更は新しいRuntime Entry PackageとGameHostを起動する。

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
- generated Public Header／C ABI contract hash、Dependency Set、manifest、Capability／access／budget
- build identity、Target、Configuration、Toolchain lock、artifact hash
- primary／secondary compile、format、warning、static analysisの結果
- unit、property、fuzz、integration、replay、save／load、ASan、contract conformanceの結果
- 同一fixtureのdeterminism、semantic equivalence、performance測定と入力hash
- sandbox、resource limit、failure／cancellation status

`NativeModuleBuildEvidenceV1`はbuild事実のimmutable envelopeであり、Risk、Approval、review attestation、promotion、activationを決定しない。それらの分類、署名者、失効、authorizationは[AI Security／Approval](../01-governance/ai-security-approval.md)と[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)が所有する。`SystemBundleChangeSetV1`と実装切替は[Gameplay programming model](gameplay-programming-model.md)、Projectへの登録とCommitは[Project state](project-state.md)が所有する。

Source、generated Public Header／C ABI Header、Dependency Set、Build artifact、build evidenceのhashが一つでも一致しなければload候補にしない。Named Module、Header UnitまたはBMIを検出した候補は[C++23 Language／Public Surface](../02-foundation/cpp23-modules.md)の禁止境界違反として拒否し、Artifact identity、Cacheまたはfallback入力にしない。Governanceからauthorizationが返らないartifactはinactiveに保ち、Projectは直前のQualified Variantを参照し続ける。

### 9.3 AI生成SourceとCode owner gate

AIが新規Native Sourceを生成または既存Native Sourceを変更する前に、[AI Security／Approval](../01-governance/ai-security-approval.md#94-code-owner-assignmentとapproval)の`CodeOwnerAssignmentV1`をexact module／path scopeへ解決する。Assignmentは`role_ref=role.code_owner.native_module`、対象Native module／pathを全量包含するScope、current qualification／independence Receipt、`valid_from <= evaluation_time < expires_at`、`revoked_at=null`をすべて満たさなければならない。Assignment不在、Role欠落／unknown／Shader Role、期限切れ、失効、Scope外、qualification／independence Receipt不成立ではTaskを`AwaitingCodeOwner`、Editorを`awaiting_code_owner`にし、Source Workerを起動しない。

Source生成後は、同じSource revision、exact Diff hash、Target別Build Receipt集合、独立`review_receipt_ref`に対する`CodeOwnerApprovalV1.decision=approved`をPromotion前に必須とする。Diff、base Source revision、generated contract、Dependency Set、Toolchain、Target、Build artifactのいずれかが変われば再Build／再Review／再Approvalする。Gameplay intent承認、AI Reviewer、Compiler成功、prequalified Packの過去承認を新しいDiffのCode owner承認として流用しない。

Beginner MVPはDefinition-firstと既にQualification済みのPack／Variantだけを使用し、新規Native Sourceを生成しない。既存Packはexact package／module hash、license、Target、qualification、revocationを照合し、変更なしで利用する。RequirementがDefinition／Packで成立しない場合はNativeを暗黙生成せず`capability_unavailable`とする。Advanced Project Source ActivationはこのCode owner gate、Source sandbox、G0–G7、Promotionを独立に満たす場合だけ有効であり、Beginner First Playableの合格をActivation根拠にしない。

### 9.4 AI Source task、Patch Proposal、Promotion carrier

[Executable Contracts](../02-foundation/executable-contracts.md#211-既存domain文書から回収した未登録operation候補)の`planning.operation_family.native_game_module_source`と`planning.operation_family.project_source_promotion`を将来atomic Activationするtarget carrierを次へ固定する。current MCD Operation、Manifest、Service、Policy、Validator、Diagnostic、Receipt、Signer、Provider／MCP projectionはexact `[]`であり、本schemaの存在をSource生成またはPromotion権限として扱わない。

```text
NativeGameModuleIdentityRefV1
  project_id: UUIDv7
  module_id: UUIDv7

NativeGameModuleRevisionRefV1
  project_id: UUIDv7
  module_id: UUIDv7
  module_revision: uint64
  module_manifest_sha256: Sha256DigestV1
  source_revision_sha256: Sha256DigestV1

NativeModuleSourceRevisionV1
  project_id: UUIDv7
  module_id: UUIDv7
  source_revision: uint64
  source_tree_sha256: Sha256DigestV1
  module_manifest_sha256: Sha256DigestV1
  source_revision_content_hash: Sha256DigestV1

NativeSourcePathScopeRefV1
  scope_artifact_ref:
    exact ArtifactRefV1(
      artifact_kind=native_source_path_scope, schema_version=1)
  scope_id: StableId
  scope_version: uint32
  scope_content_hash: Sha256DigestV1

BoundedNativeGameProfileRefV1
  profile_id: StableId
  profile_schema_version: 1
  profile_content_hash: Sha256DigestV1

NativeSourceRequirementRefV1
  requirement_id: StableId
  requirement_version: uint32
  requirement_content_hash: Sha256DigestV1

NativeModuleSourceTaskV1
  task_id:
    urn:mirakan:native-module-source-task:sha256:<lowercase-hex-64>
  project_id: UUIDv7
  base_project_revision: uint64
  source_subject:
    action: create
      module_identity_ref: NativeGameModuleIdentityRefV1
    | action: update
      module_ref: NativeGameModuleRevisionRefV1
      base_source_revision_ref:
        exact ArtifactRefV1(
          artifact_kind=project_native_source_revision,
          schema_version=1)
  path_scope_refs[1..256]: NativeSourcePathScopeRefV1
  bounded_native_game_profile_ref: BoundedNativeGameProfileRefV1
  bounded_native_game_profile_sha256: Sha256DigestV1
  public_sdk_contract_set_hash: Sha256DigestV1
  engine_public_api_sha256: Sha256DigestV1
  toolchain_lock_sha256: Sha256DigestV1
  target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1
  requirement_refs[1..128]: NativeSourceRequirementRefV1
  code_owner_assignment_ref: CodeOwnerAssignmentRecordRefV1
  code_owner_assignment_sha256: Sha256DigestV1
  requested_change_summary: NfcUtf8StringV1(1..4096 bytes)
  requested_change_summary_sha256: Sha256DigestV1
  task_input_closure_sha256: Sha256DigestV1

NativeModulePatchProposalV1
  proposal_id:
    urn:mirakan:native-module-patch-proposal:sha256:<lowercase-hex-64>
  source_task_ref:
    exact ArtifactRefV1(
      artifact_kind=project_native_source_task, schema_version=1)
  source_task_sha256: Sha256DigestV1
  source_bundle_ref: ArtifactRefV1(
    artifact_kind=source_bundle, schema_version=1)
  worker_source_delta_ref: ArtifactRefV1(
    artifact_kind=source_delta, schema_version=1)
  broker_recomputed_diff_ref: ArtifactRefV1(
    artifact_kind=broker_recomputed_source_diff, schema_version=1)
  broker_recomputed_diff_sha256: Sha256DigestV1
  before_source_tree_sha256: Sha256DigestV1
  after_source_tree_sha256: Sha256DigestV1
  changed_path_refs[1..4096]: ProjectSourceChangedPathRefV1
  dependency_delta_ref: ArtifactRefV1(
    artifact_kind=native_dependency_delta, schema_version=1)
  generated_contract_delta_ref: ArtifactRefV1(
    artifact_kind=generated_contract_delta, schema_version=1)
  required_build_plan_ref: ArtifactRefV1(
    artifact_kind=native_build_test_plan, schema_version=1)
  diagnostic_refs[0..64]: DiagnosticCodeRefV1
  proposal_sha256: Sha256DigestV1

```

`NativeModuleSourceRevisionV1.source_revision_content_hash = SHA-256(ASCII "MIRAKAN_NATIVE_MODULE_SOURCE_REVISION_V1" || uint32_be(length(closed MCD canonical bytes excluding source_revision_content_hash)) || closed MCD canonical bytes excluding source_revision_content_hash))`とする。`ArtifactRefV1(artifact_kind=project_native_source_revision,schema_version=1).sha256`はこのFieldを含む完成record bytesのSHA-256へ一致させる。`NativeGameModuleRevisionRefV1.source_revision_sha256`は同じ完成record hash、`module_manifest_sha256`はrecord内とbyte equalityにし、Module／Source revisionを別Projectまたは別Moduleへ組み替えない。

`source_subject`は`action`をdiscriminatorとするclosed tagged unionである。`create`はGatewayが予約した未使用の`module_identity_ref`だけを必須にし、`module_ref`と`base_source_revision_ref`をcanonical omissionする。`update`はCommit済みexact `module_ref`と、そのModuleの同じProject／Module ID／source revisionへ解決する`base_source_revision_ref`を必須にし、`module_identity_ref`をcanonical omissionする。createで既存Module ID、updateでmissing／zero／latest base、branch外Field、Project／Module ID差、manifest／source hash差を拒否し、空refを省略の代用にしない。ProposalとPromotionの`revision_transition`はこのactionと一致しなければならない。

`path_scope_refs[]`はAssignmentのexact Scope kind／ref内に全量包含され、各`scope_artifact_ref`をAssignmentの`OperationMutationTypedScopeRefV1.scope_artifact_ref`とbyte equalityにし、解決したNFC正規化済みProject-relative pathまたはModule refのcanonical tuple順、重複禁止である。`NativeSourcePathScopeRefV1.scope_content_hash`はscope ID／versionと解決済みpathまたはModule tupleを含むclosed scope payloadのhashであり、同payloadの完成bytes hashは`scope_artifact_ref.sha256`と一致させる。表示path、glob text、Assignmentの配列indexをidentityにしない。absolute path、`..`、symlink escape、glob expansion後にAssignment外となるpath、Engine／Extension／Adapter／Validator／Policy directoryを拒否する。`changed_path_refs[]`は[Project state](project-state.md#51-envelope)のtyped `ProjectSourceChangedPathRefV1`で、同Ownerが定めるcanonical tuple順を使い、全recordの`source_kind=native_module`、Project一致、closed `BrokerRecomputedSourceDiffV1`とのset equality、全pathのScope包含を必須にする。`source_bundle_ref`はcreateではcanonical empty Native source treeとScope、updateではTaskのbase source revisionとScopeからBrokerが構築したbytesである。`worker_source_delta_ref`はuntrusted worker output、`broker_recomputed_diff_ref`はBrokerがclean StagingへDeltaを適用後に再計算した唯一のApproval対象である。Worker申告Diff hashを`broker_recomputed_diff_sha256`へ複写しない。

Task IDはASCII `MIRAKAN_NATIVE_MODULE_SOURCE_TASK_V1`、Proposal IDは`MIRAKAN_NATIVE_MODULE_PATCH_PROPOSAL_V1`と各自己IDを除く完成canonical bytesからSHA-256を計算して上記URNへ投影する。`bounded_native_game_profile_sha256`、`code_owner_assignment_sha256`、`source_task_sha256`はそれぞれ隣接refが解決する完成record hashとbyte equalityにする。`requested_change_summary_sha256`はNFC UTF-8 summary bytesを`uint32_be` length framingしてASCII `MIRAKAN_NATIVE_REQUESTED_CHANGE_SUMMARY_V1`と連結したSHA-256である。Target、Requirement、Scopeの各ref集合はID／version／content hash順でcanonical sortし、duplicate ID、同ID別hash、latest／display name lookupを拒否する。`proposal_sha256`は同じProposal bytesのdomain-separated digestで、`proposal_id` suffixとbyte equalityを必須にする。ref／隣接hash、Project／revision、before／after tree、Scope、Target、Toolchain、Candidateの一Field差を拒否する。

Promotionの共通`ProjectSourcePromotionSubjectV1`／`ProjectSourcePromotionReceiptV1`は[Project state §5.1](project-state.md#51-envelope)だけが所有する。Promotionは`source.source_kind=native_module`かつ`revision_transition.kind`がTask actionと一致するclosed branch、`trusted_internal`専用で、全changed pathを包含するcurrent `CodeOwnerAssignmentV1`、同じProposal Diffへの`CodeOwnerApprovalV1.decision=approved`、distinct independent reviewer、全TargetのNative Build ReceiptとCandidate Test Receipt、current revocationを再検証する。成功Receipt発行後だけ`RegisterNativeModuleRevision`が同じsource revision refを`ProjectChangeSetV1`へ登録できる。Provider／MCP／standard external clientへ`operation.project_source.promote_revision`を投影せず、Source task／Patch Proposal成功をPromotion、Project Commit、Capability Activationとして表示しない。

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
| 未宣言include／未許可Header／CMake dependency cycle | Source Gate失敗、近いHeaderまたはTargetへFallbackしない |
| `export module`／`import std`／Header Unit／BMI tokenまたはartifact検出 | C++ Public Surface Gate失敗、artifactを生成またはloadしない |
| Code owner Assignment不在／Role欠落・unknown・Shader Role／失効／Scope外 | Source Workerを起動せず`AwaitingCodeOwner`。BeginnerはDefinition／prequalified Packへ再Plan |
| Code owner ApprovalのDiff／revision／Receipt不一致 | Promotion／load拒否、Sourceはinactive Stagingに隔離 |
| Native Widget manifest／Capability／Target不一致 | Widget callback登録前にreject、UI規約のfallbackへ遷移 |
| Native Widget callback fault／timeout／capacity超過 | callback output全破棄、GameHost fault、同Module instance再利用禁止 |

CrashしたProject C++はEngine memoryへ到達可能な信頼済みCodeであり、runtime sandboxで安全化されたとは表現しない。

## 12. Performance、Memory、Security Gate

- Native callbackの時間はSystem ID別に測定し、[Performance／capacity](../04-runtime/performance-capacity.md)が所有するGameplay Logic合計budgetを緩和しない。
- Runtime callback内allocation countはsteady stateで0をWindows qualification baselineの目標とし、許可allocationはphase scratchからだけ行う。
- Persistent、session、scratch、command payloadを別telemetry counterへchargeする。
- C++化の成立条件は[Gameplay programming model](gameplay-programming-model.md) §1.1の選択境界に従い、性能根拠は[Performance／capacity](../04-runtime/performance-capacity.md)が所有する改善閾値を同一fixtureで満たすこと、または表現不能Capabilityだけとする。閾値のNormative値を本書へ複写しない。
- ASLR、DEP、CFG、CET互換、stack protection、warnings-as-errors等のWindows Shipping hardeningをEngine binaryと同じにする。
- Module Sourceに未宣言import、禁止Header、inline assembly、dynamic load、socket、process、environment／registry accessがないことをAST／Module graph／link import scanで検査する。
- Shipping import table、symbol、Capability manifest、Component access manifestの一致を検証する。
- C ABI Header AST scanでreference、`std::span`、STL／PMR、`Result<T>`、exception、owner wrapperが0件である。
- pointer nullability、count overflow、alignment mismatch、call-return後view access、Memory Port mismatchを各一原因で拒否する。
- source C++ callable binding、generated adapter、C ABI descriptor、Contract Set、Target Profile、Toolchain lockのhash不一致でModule全体をloadしない。
- old signature alias、dual descriptor、old readerをConsumer Inventoryなしで追加しない。

## 13. Testと受入条件

- static linkとWindows startup DLLが同一conformance fixtureを通る。
- C ABI struct size、alignment、unknown minor、wrong major、truncated tableをfuzzする。
- STL／exception／allocator／RTTI objectがABIを越えないことをheader／symbol scanする。
- createからdestroyまで各transitionへfailure injectionし、callback／job／allocationが残らない。
- phase外write、未宣言Component、invalid handle、queue overflowを拒否する。
- Definition実装とNative実装でcommand、event、Save、replay意味が一致する。
- Native descriptorのSystem ID、Contract version、Variant、State owner、phase、Component read／write、structural permission、State read／write accessがactive Game System Graphと一致しない場合にloadを拒否する。
- Cadence ABIはScheduling Ownerがsealした四branchの`SimulationAdvanceIntervalV1`から全Fieldをdirect projectionする。variableのmonotonic sample sequence、turn-basedのCommand Ref／hash、explicit-stepのrequest Ref／hash／ordinal、Type version／Contract set hashの一Fieldだけを差し替えたfixture、旧`accepted_advance_sequence`／`accepted_step_request_sequence`を持つfixture、Ref spanをcallback後へ保持するfixtureを拒否し、Native callbackを呼ばない。
- Target-specialized Variantが同じPublic System ContractとGameplay fidelity fixtureを通り、意味同等fallbackなしのTargetを非対応にする。
- Governance authorization後Project Commit失敗時に新Variantをloadせず、直前Qualified Variantを維持する。
- GameHostを100回再起動し、Editor Processのhandle／memoryが増加しない。
- Native Game ModuleのWindows qualification baselineはWindows Editor Previewと`target.windows.desktop@1`のclean static-link packageが同じModule revision hashを記録する。これはProduct First Playable C1、Capability tierまたは実装Phaseではない。Android／Appleはこのbaselineの完了条件と対応表示へ含めず、Windows ReceiptからMobile supportを推論しない。
- AI生成SourceがGovernance authorization前に正規Project／Editor／Shippingへloadされない。
- AI生成Sourceはexact `role.code_owner.native_module`、Native Scope、current Qualification、`revoked_at=null`を持つ`CodeOwnerAssignmentV1`なしに生成されず、exact Diffの`CodeOwnerApprovalV1`なしにPromotion／loadされない。missing／unknown／`role.code_owner.project_shader`／Scope差／revokedを一原因ずつ拒否する。
- Beginner Profileでは新規Native Source Taskが0件で、Definition／prequalified Pack不能な要求を`capability_unavailable`として停止する。
- Engine C++ Public Header、`CppDependencySetV1`、実際のinclude、CMake DAGが一致し、Named Module／BMI dependencyが0件である。
- Native artifactがTarget別`BuildDriverProfileV1`とBuild tree identityを記録し、Make／Ninja二重経路を持たない。
- Product C2の`UiNativeWidget`はManifest、ABI、pure callback、determinism、primitive cap、Accessibility、fallback、GameHost fault isolationをexact `{target.windows.desktop@1, target.android.mobile@1, target.apple.mobile@1}`で個別に検証する。
- ECS format migrationを伴うNative ABI変更では、native_abi Consumer Inventory record、全Evidence Requirementのpass satisfaction binding、Compatibility Change、Owner reference migration manifest、source／target Definition Closure、Definition Migration bindingが同じqualification closureへexact解決しなければload／releaseを許可しない。
- C ABI Header AST scan、pointer／count／alignment fuzz、call-return後view access、Memory Port mismatch、source binding／adapter／descriptor／Contract Set／Target／Toolchain hash不一致のnegative fixtureを通す。

Native Game ModuleのWindows qualification baseline完了条件は、Advanced Project Source Activation下の2D縦切りで一つのProject固有CapabilityをNativeGameModuleへ表現し、Code owner gate、Windows Editor Preview再起動、Windows Desktop clean static-link artifact、Definitionとのcontract conformance、fault recoveryをすべて合格することである。これはBeginner MVP／First PlayableのCompletion Gateでも、Shipping／Release readinessでもない。

Product C2のNative Game Module完了条件は、一つの宣言型では表現不能なWidgetを`UiNativeWidget`として表現し、Governance-authorized source、UI Designer fallback projection、Windows GameHost Preview、exact Windows／Android／Apple static-link package、semantic／layout／render golden、fault recoveryを同じCandidateで合格することである。これはWindows qualification baselineを置き換えず、Android／Apple package schemaまたはArchitecture文書の存在だけをSource qualificationへ数えない。

Android／AppleのProject native source qualificationは[Product Plan §5.3](../00-product/product-plan.md#53-c2-production-exact-product-boundary)のC2 requirementである。`future.capability.mobile-project-native-shader-source-qualification`というFuture ID、互換alias、Work Packageまたは後続Phaseを作らず、各TargetのABI、Compiler、static link、signing、fault、rollback、Store policyとfresh Qualificationがmaterializeするまで`not_activated`を維持する。

## 14. 一次資料

- [ISO C++ object model and language reference](https://isocpp.org/std/the-standard)
- [Microsoft DLL best practices](https://learn.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-best-practices)
- [Android NDK C++ library support](https://developer.android.com/ndk/guides/cpp-support)
- [Apple Distributing binary frameworks](https://developer.apple.com/documentation/xcode/distributing-binary-frameworks-as-swift-packages)

外部の汎用plugin ABIを採用せず、C ABIはWindows Development startup loadと静的linkの共通validation入口だけに限定する。
