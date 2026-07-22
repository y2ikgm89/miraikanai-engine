# Miraikanai Engine Memory／Pointers

- 文書ID: mirakan.arch.memory-pointers
- 状態: review
- 正本範囲: Pointer taxonomy、ownership、typed handle、lease／view、Memory domain、arena／pool、allocation metadata、OOM、AI contract、failure、telemetry、Qualification
- 非正本範囲: 外部Library・Tool version／hash／license、Runtime共通budget／phase、GPU residency、一般命名・Directory、Schema共通構造。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Core architecture](core-architecture.md)、[Toolchain／Dependencies](toolchain-dependencies.md)、[Executable contracts](executable-contracts.md)、[Naming／Project layout](naming-project-layout.md)、[Math／Core utilities](math-core.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論

Miraikanai Engineの公式方式は、**契約駆動のhybrid memory management**とする。全面GC、全面reference count、全面custom general-purpose allocatorのいずれも採用しない。

1. 長寿命C++ objectはRAIIとsingle ownershipを既定にする。
2. Runtime object、Asset、GPU／Physics／Audio resourceはtyped generation handleで参照する。
3. Component、snapshot、bufferの短期accessはscope／phase／epoch付きleaseまたはviewで表現する。
4. Frame、RenderFrame、Job scratchはbounded arena、実測で有効なprivate固定size objectだけpoolを使う。
5. Game／AI公開面へowning raw pointer、World内部pointer、native SDK object、allocator objectを公開しない。
6. 所有権、寿命、thread、allocation、失効、OOMをMCDの機械可読contractとして保持し、C++ API、static check、diagnostic、testを同じcontractから生成する。
7. unsafe raw storage操作はFoundation／Adapter private implementationへ隔離し、Capability、Review、sanitizer、benchmarkを通過した場合だけ使用する。

この方式は、Unreal Engineの`UObject` GC、Unityのmanaged GC、Godotのmanual／reference-counted混在を導入する判断ではない。既存Engineから採用するのは、用途別pointer型、generation ID、archetype chunk、temporary allocator、opaque resource handleという実証済みの境界であり、Miraikanai固有のAI可読contractと明示的失敗規則を上位に置く。

## 2. 選択理由

### 2.1 比較した方式

| 方式 | 長所 | 不採用理由／採用範囲 |
|---|---|---|
| 全面GC | Gameplay authoringが容易、manual freeを削減 | stop／incremental work、native resource寿命、C++ ABI、deterministic hot pathと一致しないため不採用 |
| 全面reference count | localな所有関係を型で表現しやすい | atomic count、cycle、destruction point分散、cache costのためRuntime objectへ不採用 |
| 全面custom allocator | 全allocationを制御できる | 初期実装Risk、platform allocator／sanitizer互換、profile前の過剰最適化になるため不採用 |
| 標準heapだけ | 単純で保守しやすい | frame transient、audio、physics、render submissionのallocation／fragmentation制御が不足するため非hot pathの基準としてだけ採用 |
| 契約駆動hybrid | 寿命別allocator、typed handle、AI validationを分離できる | 公式採用。schema／codegen／telemetryをPhase 0で実装する |

### 2.2 既存Engineから採用する教訓

- Unreal Engineのように、強参照、弱参照、soft Asset参照、非`UObject`所有を用途別の型へ分ける。ただし、`UPROPERTY`の有無でGC安全性が変わる暗黙契約は導入せず、contract fieldと生成型で明示する。
- Unity Entitiesからは、同じComponent集合をarchetypeとしてまとめ、Component種別ごとの配列をchunkへ格納するdata-oriented layoutを採用する。Unity固有のEntity表現やchunk容量は移植しない。
- Miraikanaiのruntime handleはindex＋generation、ECS archetype chunkは16 KiBを正本とする。これは本書が独立に定めるEngine-owned designであり、Unityの現行実装値を根拠にした判断ではない。handle規則は[§4.3](#43-generation-handle)、chunk容量の継続検証は[§9.3](#93-performanceendurance)に定める。
- Unity Collectionsのように、一時allocationと永続allocationを寿命で分ける。ただし、hot pathのarena枯渇を一般heapへ暗黙fallbackさせない。
- Godotの`ObjectID`／`RID`のように、低level subsystemをopaque handleで参照する。ただし、owner、free authority、generation、serialization可否をcontractに含める。

## 3. 決定権

本書はMemory／Pointerの**機械可読表現、公開型、AI生成規則、unsafe境界**を決定する。数値budget、Runtime phase、Subsystem固有capacity、GPU Backend詳細はそれぞれRuntime、Rendering、Mobile、Subsystem規約が決定する。

矛盾時は次の順で扱う。

1. 所有権、pointer表現、公開可否、AI生成規則は本書。
2. C++ Module／directory／dependencyは基盤規約。
3. phase、lease失効、Runtime budget、failure publishはRuntime規約。
4. ABI、Module instance lifetime、Memory PortはNativeGameModule規約。
5. Backend native lifetimeとsubmission retireはRendering／Platform規約。

## 4. 公式Pointer taxonomy

### 4.1 公開型

| 意味 | 公式表現 | owner | 保存 | Job capture | AI生成 |
|---|---|---|---|---|---|
| 小さな値 | `T` | container／scope | 可 | 可 | 可 |
| Engine内部single owner | `std::unique_ptr<T>`またはmove-only RAII wrapper | 一つ | session内だけ | owned packet時だけ | Engine保守R4だけ |
| NativeGameModule persistent owner | `MirakanUniqueOwner<T>` | Module instance | session内だけ | 禁止 | 生成factory経由だけ可 |
| 必須同期borrow | `T&`／`const T&` | 呼出側 | 禁止 | 禁止 | 生成API内部だけ |
| 任意同期borrow | `T*`／`const T*` | 呼出側 | 禁止 | 禁止 | 公開生成APIでは禁止 |
| 連続領域borrow | `std::span<T>`またはgenerated bounded view | 呼出側 | 禁止 | 禁止 | generated bounded viewだけ可 |
| Component access | `ReadLease<T...>`／`WriteLease<T...>` | World | 禁止 | 禁止 | 可 |
| Runtime object | `Handle<Tag>` | Domain registry | Runtime handleは保存禁止 | 値capture可 | 可 |
| Asset payload | `AssetVersionHandle<T>`＋`AssetReadLease<T>` | Asset Registry | Stable Asset IDだけ保存可 | handleだけ可 | 可 |
| GPU resource | move-only owner＋`ResourceHandle<T>` | Renderer | 禁止 | handleだけ可 | Project／AIはhandle操作もtyped Port経由 |
| 真の共有immutable Tool data | `std::shared_ptr<const T>` | 列挙済み複数owner | Tool session内だけ | Tool jobだけ | Engine Tool保守R3以上 |

`std::weak_ptr`は許可済みTool共有objectのcycle切断だけに使用する。Entity、Component、Asset payload、GPU／Physics／Audio resourceの所有へ`shared_ptr`／`weak_ptr`を使わない。

### 4.2 Safe APIとunsafe実装

公開面を二層に固定する。

- **Safe generated API**: Project C++、GameplayDefinition、AI、Editor automationが使用する。value、typed handle、lease、immutable view、bounded writer、`Result<T>`だけを公開する。
- **Unsafe private implementation**: Foundation allocator、ECS storage、Adapter、serialization kernelだけが使用する。placement new、pointer arithmetic、native pointer、raw storageを許可するが、公開Module interfaceへexportしない。

`unsafe`はnamespace名だけで権限を表現しない。CMake target、Named Module partition、include path、Capability manifest、AST scanのすべてで境界を強制する。

### 4.3 generation handle

```cpp
template<class Tag>
struct Handle final {
    std::uint32_t index;
    std::uint32_t generation;
};
```

- `{0, 0}`はinvalid。
- indexとgenerationは1から開始する。
- slot再利用時にgenerationをincrementする。
- wrapするslotは永久retireする。
- handleは非所有であり、owner registryだけがresolveする。
- handle bitをnative SDK ID、pointer、file offsetとして解釈しない。
- Save、Project、NetworkへRuntime handleを保存せず、Stable IDまたは専用protocol IDへ変換する。
- resolveはhandle kindごとの`Result<ReadLease<T>>`または`Result<ImmutableView<T>>`を返し、失敗をnull objectへ変換しない。
- owner thread外ではresolveせず、handle＋immutable snapshotまたはcommandを渡す。

Slot metadataはindex順のcontiguous arrayを基準とし、hot resolveごとのheap allocation、reference count、global mutexを禁止する。atomicが必要なregistryはowner／reader topologyとcontention benchmarkをADRへ記録する。

### 4.4 leaseとepoch

Leaseはmove-onlyで、少なくとも次を保持する。

```text
owner_registry_id
owner_generation
created_phase
created_epoch
thread_affinity
access_mode
bounded_range
```

Development／ASan Configurationはaccessごとにphase、epoch、thread、rangeを検査する。Shippingは検査を削減できるが、layout、owner、失効条件を変えない。

Lease、view、span、reference、writer、scratch blockをmember、global、event、command、job packet、lambda capture、coroutine stateへ保存しない。長く保持する必要があるdataは値copy、immutable snapshot、typed handleのいずれかへ変換する。

## 5. 公式Memory architecture

### 5.1 二軸分類

すべてのEngine／Vendor allocationは次の二軸を同時に持つ。

```text
charge_domain          budget owner
memory_resource_class  lifetime／release policy
```

`charge_domain`はCore World、Rendering、Physics、Navigation、Animation、Audio、Asset streaming等である。`memory_resource_class`はSystem、Frame、RenderFrame、Scratch、Pool、Streaming、Editor、Testである。二軸を一つのenumへ統合しない。

### 5.2 Resource stack

```text
SystemMemoryResource
  -> TrackingMemoryResource
  -> BudgetMemoryResource
  -> MonotonicArenaResource | PoolMemoryResource | pmr container
```

- `SystemMemoryResource`はOS／CRT aligned allocationの唯一の一般upstream。
- `TrackingMemoryResource`はdomain、class、tag、size、alignment、thread、frame／job、source locationを記録する。
- `BudgetMemoryResource`はupstream call前にhard capを検査する。
- `MonotonicArenaResource`はFrame、RenderFrame、Scratchをbump allocateし、一括resetする。
- `PoolMemoryResource`はsize分布とchurnが安定し、同一fixtureで有意な改善があるprivate typeだけに使う。
- `FailureMemoryResource`はTest専用で、N回目、size、domain、phaseを条件に失敗させる。

Process global default PMR resourceを変更しない。ResourceはComposition Rootから明示注入し、Resource lifetimeがcontainer／ownerより長いことをcontract testで保証する。

### 5.3 allocation policy

| Path | 許可allocation | 枯渇時 |
|---|---|---|
| Render submission | RenderFrame arena／事前予約pool | frame fault。一般heap fallback禁止 |
| Physics step | Physics scratch／pool | tick非publish、Physics fault |
| Navigation query batch | Job scratch／lease pool | request失敗またはtick規約のfault |
| Animation sample／blend | instance／job scratch | animation result非publish |
| Audio callback | preallocated ring／bufferだけ | zero-fill＋underrun diagnostic |
| Entity iteration | 既存chunk／query scratchだけ | iteration開始前に拒否 |
| Loading／Cook | Budget付きSystem／Streaming | job失敗、partial artifact非公開 |
| Editor noncritical Tool | Editor domain | cache eviction後、一度だけretry |

hot path用Resourceは`null_memory_resource`相当またはBudget Resourceを最終upstreamとし、Shippingでも暗黙のSystem heap fallbackを行わない。Developmentはfallback試行自体をperformance failureとして記録する。

### 5.4 NativeGameModule persistent owner

Module persistent objectは次の生成factoryだけを使用する。

```cpp
template<class T, class... Args>
Result<MirakanUniqueOwner<T>> MirakanMakePersistent(
    MirakanNativeMemoryPortV1 memory,
    std::uint32_t tag_id,
    Args&&... args);
```

`MirakanUniqueOwner<T>`はmove-onlyで、object pointer、Memory Port、size、alignment、tagを保持する。destructorは`T`のdestructorを一度だけ呼び、取得時と同じPort、size、alignment、tagでdeallocateする。copy、releaseによる所有raw pointer流出、別Portへの移管を禁止する。

default構築状態とmove後状態は空ownerとし、destructorは何もしない。Memory Portのallocateがnullを返した場合は`MemoryBudgetExceeded`、`T`のconstructorが例外を投げた場合は取得blockを同じPortへ返して`NativeObjectConstructionFailed`を返す。例外をC ABIまたはRuntime phase境界へ伝播させない。

Project C++の明示`new`／`delete`、`malloc`／`free`を禁止する。Module内部のPMR containerは`MirakanNativeMemoryPortV1`を包むmodule-owned Adapterをconstructorで受け取る。ABIを越えてfactory、owner、PMR object、STL containerを渡さない。

## 6. AI可読Contract

### 6.1 `PointerContractV1`

API parameter、return、fieldごとに次を必須にする。

```text
contract_id
type_id
ownership = value | unique_owner | shared_immutable | borrow | lease | handle
nullability = non_null | nullable | invalid_zero
lifetime = scope | callback | phase | tick | frame | job | submission | session | persistent
owner_id
thread_affinity
access = read | write | read_write
invalidation_events[]
storable
job_capturable
serializable
networkable
resolve_authority
unsafe_capability_id?
```

自由文字列をsuccess、dispatch、permission判定へ使用しない。すべてclosed enumまたはregistered IDとする。

### 6.2 `MemoryContractV1`

allocation siteまたはcontainer ownerごとに次を必須にする。

```text
contract_id
charge_domain_id
memory_resource_class
allocation_policy
alignment
capacity_source
hard_limit_bytes
steady_state_allocation_count
allowed_phases[]
allowed_threads[]
reset_or_release_event
upstream_fallback = forbidden | budgeted_system
oom_behavior
telemetry_tag_id
source_owner
```

`capacity_source`はProfile、Cooked Artifact、fixed ABI、measured P99.9＋headroomのいずれかである。Source codeの説明なしmagic numberを許可しない。

`allocation_policy`は`system | monotonic_arena | size_class_pool | streaming_cache | external_adapter`、`upstream_fallback`は`forbidden | budgeted_system`のclosed enumとする。`oom_behavior`はDiagnostic Catalogへ登録されたclosed ID、domain、class、phase、thread、ownerはMCD登録済みIDとし、Providerが自由文字列を追加しない。

### 6.3 生成物

Contract compilerは同じMCDから次を決定論的に生成する。

1. C++ safe API型とfunction signature。
2. `PointerContractManifest.bin`と`MemoryContractManifest.bin`。
3. clang-tidy／AST scan用allowlist。
4. Development lease／thread／domain assertion。
5. AI Provider向け、native addressを含まないschema projection。
6. Diagnostic code、argument schema、修正候補。
7. Unit／negative／property test fixture。

Generated fileを手編集しない。Source contract、generated Header／Module、binary manifestのhash不一致はBuildを拒否する。

### 6.4 AI生成規則

AIは次を生成しない。

- owning raw pointer、address arithmetic、placement new。
- 明示`new`／`delete`、`malloc`／`free`。
- Runtime objectの`shared_ptr`／`weak_ptr`。
- lease、span、reference、writer、scratch blockの保存またはasync capture。
- World、GPU、Physics、Audio、Platform native pointer。
- Process global allocator変更、unbounded allocator／container。
- budget、alignment、failure policyを持たないpersistent allocation。

AIはcontract IDから許可型とfactoryを選ぶ。判断不能時はraw pointerで補完せず、`MissingPointerContract`または`MissingMemoryContract`をBlocking diagnosticとして返す。

## 7. FailureとDiagnostic

| Code | 条件 | Development | Shipping／Tool |
|---|---|---|---|
| `MIRAKAN-MEMORY-CONTRACT_MISSING` | allocation siteにcontractなし | Build failure | 未昇格artifactに含めない |
| `MIRAKAN-MEMORY-DOMAIN_MISMATCH` | free元、domain、tag不一致 | fail-fast | session fault |
| `MIRAKAN-MEMORY-BUDGET_EXCEEDED` | hard cap超過 | allocation拒否＋capture | 規定evict後一度retry、再失敗はdomain fault |
| `MIRAKAN-MEMORY-CONSTRUCTION_FAILED` | `MirakanMakePersistent`のconstructor失敗 | 取得blockを同一Portへ返却しtyped failure | typed failure。partial objectを公開しない |
| `MIRAKAN-MEMORY-HOT_PATH_FALLBACK` | hot pathが一般heapを要求 | performance test failure | fallbackせず当該phase fault |
| `MIRAKAN-POINTER-STALE_HANDLE` | generation不一致 | owner／create／destroy tickを表示 | typed failure |
| `MIRAKAN-POINTER-BORROW_EXPIRED` | epoch／phase失効後access | fail-fast | invalid actionをpublishしない |
| `MIRAKAN-POINTER-THREAD_AFFINITY` | 非owner threadでresolve／access | fail-fast | command／job resultをreject |
| `MIRAKAN-POINTER-UNSAFE_CAPABILITY_REQUIRED` | unsafe API権限なし | Build failure | artifact promotion拒否 |
| `MIRAKAN-POINTER-CONTRACT_MISSING` | 公開parameter／return／fieldにcontractなし | Build failure | 未昇格artifactに含めない |

C++公開`Error` enumerator（PascalCase）とDiagnostic codeは次の1:1対応を正本とし、対応を持たないErrorまたはcodeを追加しない。

| Error | Diagnostic code |
|---|---|
| `MemoryBudgetExceeded` | `MIRAKAN-MEMORY-BUDGET_EXCEEDED` |
| `NativeObjectConstructionFailed` | `MIRAKAN-MEMORY-CONSTRUCTION_FAILED` |
| `MissingMemoryContract` | `MIRAKAN-MEMORY-CONTRACT_MISSING` |
| `MissingPointerContract` | `MIRAKAN-POINTER-CONTRACT_MISSING` |

OOM diagnostic用storageはEmergency reserveから事前確保し、OOM pathで一般container、format allocation、logger queue拡張を行わない。

## 8. PerformanceとTelemetry

Development／Profileはallocationごとにdomain、class、tag、size、alignment、thread、frame／job、source locationを記録する。Shippingもdomain current／peak、budget、OOM、hot fallback、GPU retire backlogを保持する。

必須metricは次である。

- reserved、committed、resident、live、peak。
- allocation／free count、largest allocation。
- arena high-water、upstream request、unused tail。
- pool live／free、size class、reuse、internal fragmentation。
- ECS chunk utilization、archetype fragmentation、structural copy bytes。
- Streaming cache hit、eviction、retire待ちbytes。
- GPU committed／resident、alias saving、deferred release bytes／serial backlog。
- handle resolve count、failure count、retired slot。
- lease violation、thread-affinity violation。

性能改善を公式採用するには、[Runtime performance／capacityの共通promotion threshold](../04-runtime/performance-capacity.md#8-measurementregressionpromotion)を満たすか、同一fixtureでpeak memoryを15%以上改善し、correctness、visual、fault、load timeを規定値以上悪化させない。全面pool化、lock-free化、custom allocator化を名称だけで最適化扱いしない。

## 9. TestとQualification

### 9.1 correctness

- generation slotのcreate、destroy、reuse、random invalid、wrap retire、space exhaustion。
- `ReadLease`／`WriteLease`のphase、epoch、thread、overlap、structural mutation失効。
- `MirakanUniqueOwner`のconstructor failure、move、destructor一回、Port／size／alignment／tag一致。
- arena reset、pool reuse、double free、wrong resource、alignment 1～4096。
- Asset version leaseとGPU／Audio／Physics retireの同時条件。
- GPU multi-queue submission completion前のallocation／binding再利用禁止。

### 9.2 negative／static

- raw owning pointer、明示allocation、Runtime `shared_ptr`、borrow captureをcompile／AST Gateで拒否する。
- ABI HeaderへSTL、PMR、exception、owner wrapper、native typeが流出しないことをscanする。
- Contract、generated API、manifestのhash driftを拒否する。
- ASan poison／unpoison、Full PageHeap、HWASan、GWP-ASan、Apple ASan／TSanをTarget規約どおり実行する。

### 9.3 performance／endurance

- hot path一般heap allocation 0。
- ECS 8／16／32 KiB比較は既存Runtime fixtureを使い、16 KiBを無測定で変更しない。
- 120秒×5 performance run、10分soakを通常Gateとする。
- Windows release候補は2時間resource churn、Asset hot reload、World create／destroy、GPU retireを追加する。
- R4 Memory／Pointer変更は最低60分、release候補は対象Targetのendurance規約を満たす。
- Mobileは10分実機run、30分thermal、2時間enduranceを満たす。

## 10. 導入順序

1. `PointerContractV1`、`MemoryContractV1`、Diagnostic codeをMCDへ追加する。
2. generation handle、slot registry、`Result`、memory tagをFoundationへ実装する。
3. System／Tracking／Budget／Failure Resourceを実装する。
4. Frame／RenderFrame／Scratch arenaとASan poisonを実装する。
5. `ReadLease`／`WriteLease`、epoch、thread-affinity検査を実装する。
6. Native Memory Port Adapterと`MirakanUniqueOwner` factoryを実装する。
7. Contract compilerからC++ API、manifest、static rule、test fixtureを生成する。
8. ECS 16 KiB archetype chunk、hot path allocation Gate、telemetryを接続する。
9. GPU allocator、submission retire、Asset version leaseを接続する。
10. Windows、Mobile、Native Module、AI生成negative／performance／endurance Gateを合格させる。

後段機能が前段の未実装を独自pointer wrapperやlocal allocatorで迂回してはならない。

## 11. Definition of Done

1. Public APIの全pointer／view／handleに`PointerContractV1`がある。
2. Engine／Vendorの全allocation siteに`MemoryContractV1`または承認済みAdapter mappingがある。
3. Project／AI公開APIにowning raw pointer、native object、allocator objectが存在しない。
4. Runtime objectのsingle ownerとresolve authorityを一意に説明できる。
5. hot path一般heap allocation 0をCIが検査する。
6. OOM、stale handle、expired borrow、wrong thread、wrong resourceがtyped failureになる。
7. ASan、race検証、OOM injection、2時間Windows churn、Mobile enduranceが合格する。
8. Reference fixtureがCPU／GPU P95、memory、allocation、fragmentation Gateを満たす。
9. AIが欠落contractを推測で補わず、Blocking diagnosticを返す。
10. 生成API、manifest、Provider projection、test fixtureのhashが同一MCDへtraceできる。

## 12. 根拠と判断

| 根拠／Owner | 採用した判断 |
|---|---|
| [Unreal Engine Object Pointers](https://dev.epicgames.com/documentation/en-us/unreal-engine/object-pointers-in-unreal-engine) | 強、弱、soft、scope strong参照を用途別型へ分離する |
| [Unreal Engine Smart Pointers](https://dev.epicgames.com/documentation/en-us/unreal-engine/smart-pointers-in-unreal-engine) | non-intrusive unique／shared／weakとreference-count costを区別する |
| [Unreal Engine Memory and CPU Performance Considerations](https://dev.epicgames.com/documentation/en-us/unreal-engine/common-memory-and-cpu-performance-considerations-in-unreal-engine) | poolは生成破棄costをprofileしたobjectだけに使用する |
| [Unity Managed Memory](https://docs.unity3d.com/jp/current/Manual/performance-managed-memory-introduction.html) | GC allocation／collection spikeをMiraikanai hot pathへ導入しない |
| [Unity Unmanaged Memory](https://docs.unity3d.com/ja/current/Manual/performance-unmanaged-memory.html) | temporary、job、persistent allocationを寿命で分ける |
| [Unity Technical Articles: The DOTS packages and features](https://discussions.unity.com/t/from-the-new-e-book-the-dots-packages-and-features/368224) | 同じComponent集合をarchetypeとしてまとめ、Component種別ごとの配列をchunkへ格納するlayoutだけを外部先例として採用する |
| [本書 §4.3](#43-generation-handle)／[§9.3](#93-performanceendurance) | index＋generation handleと16 KiB archetype chunkはMiraikanaiが所有する正本値とし、Unityの現行実装仕様として扱わない |
| [Godot Object ownership](https://docs.godotengine.org/en/stable/engine_details/architecture/object_class.html) | 非ownerの長期保持にraw pointerを使わずIDへ変換する |
| [Godot RID](https://docs.godotengine.org/en/stable/classes/class_rid.html) | low-level resourceをsession-local opaque handleで公開する |
