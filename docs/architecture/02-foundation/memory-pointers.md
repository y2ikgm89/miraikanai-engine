# Miraikanai Engine Memory／Pointers

- 文書ID: mirakan.arch.memory-pointers
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Pointer taxonomy、ownership、typed handle、lease／view、Memory domain、arena／pool、allocation metadata、OOM、AI contract、failure、telemetry、Qualification
- 非正本範囲: 外部Library・Tool version／hash／license、Runtime共通budget／phase、ECS storage layout・query・lease、GPU residency、一般命名・Directory、Schema共通構造。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Core Architecture](core-architecture.md)、[Math／Core Utilities](math-core.md)、[Naming／Project Layout](naming-project-layout.md)
- 関連文書: [AI-readable Asset／Memory／Async Loading Alignment](../decisions/2026-07-28-ai-asset-memory-async-alignment.md)、[Core architecture](core-architecture.md)、[Toolchain／Dependencies](toolchain-dependencies.md)、[Executable contracts](executable-contracts.md)、[Compatibility／Evolution](compatibility-evolution.md)、[Naming／Project layout](naming-project-layout.md)、[Math／Core utilities](math-core.md)、[Product plan](../00-product/product-plan.md)、[Runtime ECS](../04-runtime/entity-component-system.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[Runtime ECS Design Closure Review](../appendices/runtime-ecs-design-closure-review.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-27

## 1. 結論

Miraikanai Engineの採用候補は、**契約駆動のhybrid memory management**とする。これはProject判断であり、外部Vendorの公式推奨を意味しない。全面GC、全面reference count、全面custom general-purpose allocatorのいずれも採用しない。

1. 長寿命C++ objectはRAIIとsingle ownershipを既定にする。
2. Runtime object、Asset、GPU／Physics／Audio resourceはtyped generation handleで参照する。
3. Component、snapshot、bufferの短期accessはscope／phase／epoch付きleaseまたはviewで表現する。
4. Frame、RenderFrame、Job scratchはbounded arena、実測で有効なprivate固定size objectだけpoolを使う。
5. Game／AI公開面へowning raw pointer、World内部pointer、native SDK object、allocator objectを公開しない。
6. 所有権、寿命、thread、allocation、失効、OOMをMCDの機械可読contractとして保持し、C++ API、static check、diagnostic、testを同じcontractから生成する。
7. unsafe raw storage操作はFoundation／Adapter private implementationへ隔離し、Capability、Review、sanitizer、benchmarkを通過した場合だけ使用する。

この方式は、Unreal Engineの`UObject` GC、Unityのmanaged GC、Godotのmanual／reference-counted混在を導入する判断ではない。既存Engineから採用するのは、用途別pointer型、generation ID、temporary allocator、opaque resource handleという実証済みの境界であり、Miraikanai固有のAI可読contractと明示的失敗規則を上位に置く。ECS storageの設計は[Runtime ECS](../04-runtime/entity-component-system.md)が所有する。

## 2. 選択理由

### 2.1 比較した方式

| 方式 | 長所 | 不採用理由／採用範囲 |
|---|---|---|
| 全面GC | Gameplay authoringが容易、manual freeを削減 | stop／incremental work、native resource寿命、C++ ABI、deterministic hot pathと一致しないため不採用 |
| 全面reference count | localな所有関係を型で表現しやすい | atomic count、cycle、destruction point分散、cache costのためRuntime objectへ不採用 |
| 全面custom allocator | 全allocationを制御できる | 初期実装Risk、platform allocator／sanitizer互換、profile前の過剰最適化になるため不採用 |
| 標準heapだけ | 単純で保守しやすい | frame transient、audio、physics、render submissionのallocation／fragmentation制御が不足するため非hot pathの基準としてだけ採用 |
| 契約駆動hybrid | 寿命別allocator、typed handle、AI validationを分離できる | 本Projectの採用候補。Schema／codegen／telemetryを独立した受入subjectとして検証する |

### 2.2 既存Engineから採用する教訓

- Unreal Engineのように、強参照、弱参照、soft Asset参照、非`UObject`所有を用途別の型へ分ける。ただし、`UPROPERTY`の有無でGC安全性が変わる暗黙契約は導入せず、contract fieldと生成型で明示する。
- Unity Entitiesからは、同じComponent集合をarchetypeとしてまとめ、Component種別ごとの配列をchunkへ格納するdata-oriented layoutを採用する。Unity固有のEntity表現やchunk容量は移植しない。
- Miraikanaiのruntime handleはindex＋generationを使う。Runtime Entity handle、ECS archetype chunk、chunk容量、query lease、継続検証の正本は[Runtime ECS](../04-runtime/entity-component-system.md)と[Performance／Capacity](../04-runtime/performance-capacity.md)であり、本書は一般handle／lease規則だけを所有する。
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

### 3.1 Cross-owner consumer binding

本書の型名を各Subsystemが自由に再定義したり、参照リンクだけで寿命規則を推測したりしない。Phase 0で`PointerMemoryConsumerBindingV1`をMCDへ追加し、各consumer conceptについて次をclosed fieldで宣言する。

```text
consumer_document_id
consumer_concept_id
reference_form = value | handle | lease | immutable_view | unique_owner | external_adapter
lifetime_owner_document_id
allocation_contract_required
retire_or_invalidation_owner_document_id
storable_form = none | stable_id | project_id | protocol_id
job_capture_form = none | value | handle | immutable_snapshot | owned_packet
qualification_owner_document_id
```

このbindingは一般Pointer／Memory語彙と各Subsystemの固有意味を混同しないための接続層である。`Handle`、lease、allocation resource、公開禁止型の定義は本書だけが所有し、Entity slot、Asset version、GPU fence、Physics query、Audio callbackなどの固有ID・retire条件・容量値は各Owner文書が所有する。bindingに未登録の公開pointer／view／handle、またはallocation siteは`MissingPointerContract`／`MissingMemoryContract`として扱う。

| consumer領域 | conceptの正本 | bindingで閉じる境界 |
|---|---|---|
| Foundation／Contract | [Core architecture](core-architecture.md)、[Executable contracts](executable-contracts.md)、[Toolchain／Dependencies](toolchain-dependencies.md) | MCD type closure、safe／unsafe境界、sanitizer lane、生成物hash |
| Runtime | [Runtime ECS](../04-runtime/entity-component-system.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Persistence／Save](../04-runtime/persistence-save.md) | Entity／query lease、job hand-off、Runtime handleの非永続化、load／retire |
| Authoring | [Asset Lifecycle](../03-authoring/asset-lifecycle.md)、[Native Game Module](../03-authoring/native-game-module.md)、[Editor UI Framework](../03-authoring/editor-ui-framework.md) | Asset version access、Module owner、Tool-only shared immutable data |
| Simulation | [Collision](../05-simulation/collision.md)、[Physics](../05-simulation/physics.md)、[Navigation](../05-simulation/navigation.md)、[Animation](../05-simulation/animation.md) | native adapterの隔離、query／snapshot lease、job scratch、retire |
| Rendering／Platform | [Render Graph](../06-rendering/render-graph.md)、[World](../06-rendering/world.md)、[Audio](../07-platform/audio.md)、[Input](../07-platform/input.md)、[Mobile common](../07-platform/mobile-common.md) | opaque resource handle、submission／callback境界、preallocated buffer、Target adapter |
| Qualification | [Performance／Capacity](../04-runtime/performance-capacity.md)、[Debugging／Observability／Replay](../04-runtime/debugging-observability-replay.md)、[Product plan](../00-product/product-plan.md) | telemetry、negative／endurance、受入Gate、Evidence closure |

Save、Replay、Package、AI projection、Network、job packetへlive pointer、reference、lease、span、writer、allocator objectを保存しない。保存可能な値は上表の`storable_form`で明示したStable／Project／Protocol IDだけとし、jobは`job_capture_form`で許可した値、handle、immutable snapshot、owned packetだけを受け渡す。これらの禁止を例外的なlocal wrapper、alias、dual readerで回避しない。

Math／Core UtilitiesとMemory／Pointersはinitial V1から独立したOwner、Product Work Package、Capability identityを持ち、本書はMemory／Pointer側だけを直接定義する。predecessorとなるcombined identity、旧型alias、source／target migration、dual RegistryまたはCompatibility Receiptをinitial V1へ作らない。初回materializationまたは公開後に変更する場合だけ、[Compatibility／Evolution](compatibility-evolution.md)のconsumer inventoryと承認済みCompatibility Changeを適用する。

## 4. 標準Pointer taxonomy

### 4.1 公開型

| 意味 | 標準表現 | owner | 保存 | Job capture | AI生成 |
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
- **Unsafe private implementation**: Foundation allocator、ECS storage、Adapter、serialization kernelだけが使用する。placement new、pointer arithmetic、native pointer、raw storageを許可するが、Public Headerまたはgenerated bindingへ露出しない。

`unsafe`はnamespace名だけで権限を表現しない。CMake target、private source root、public include allowlist、Capability manifest、AST scanのすべてで境界を強制する。

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

## 5. 標準Memory architecture

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

process memoryのreclamationとpersistent Artifactの削除は別authorityである。Memory Ownerはgeneration handle、lease、submission、reader pinのretire完了とresource解放を判定するが、Artifact storeのreachabilityや削除を決定しない。Project、Catalog、Package、last-valid、Recoveryから到達するArtifactの保持／sweepは[Asset Lifecycle](../03-authoring/asset-lifecycle.md)が所有し、process allocationが0になったことをArtifact非到達の証拠にしない。

### 5.3 allocation policy

| Path | 許可allocation | 枯渇時 |
|---|---|---|
| Render submission | RenderFrame arena／事前予約pool | frame fault。一般heap fallback禁止 |
| Physics step | Physics scratch／pool | advance非publish、Physics fault |
| Navigation query batch | Job scratch／lease pool | request失敗またはSimulation Advance規約のfault |
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

default構築状態とmove後状態は空ownerとし、destructorは何もしない。Memory Portのallocateがnullを返した場合は`MemoryBudgetExceeded`を返す。First-partyの全Target／Configurationは[Toolchain／Dependencies](toolchain-dependencies.md#2-c23-languageとbuild-policy)の`exception_policy=disabled_first_party`へ固定するため、`MirakanMakePersistent<T>`が受理する`T`は`noexcept` construction／destructionを満たさなければならず、throwing constructorをcatchしてtyped failureへ変換するbranchまたはconstructor-failure Diagnosticを持たない。生成後初期化が失敗し得る型は、未公開の一時ownerに対するowner-defined `Result`返却initializationを完了してからだけpublishし、失敗時は一時ownerのdestructorと同じPort／size／alignment／tagで回収する。例外有効なThird-party objectはToolchain Ownerの隔離Target／Adapter内で完結し、このfactory、First-party型またはPublic ABIへ渡さない。

Project C++の明示`new`／`delete`、`malloc`／`free`を禁止する。Module内部のPMR containerは`MirakanNativeMemoryPortV1`を包むmodule-owned Adapterをconstructorで受け取る。ABIを越えてfactory、owner、PMR object、STL containerを渡さない。

## 6. AI可読Contract

本書がAI向けに供給するのはpointer taxonomy、ownership、lifetime、allocation policy、failure、aggregate telemetryのtyped fragmentである。ArchitectureのOwner／文書状態／依存は[Architecture Governance](../01-governance/architecture-governance.md)の`ArchitectureExplainProjectionV1`、ECS layout／query／structural semanticsは[Runtime ECS](../04-runtime/entity-component-system.md)の`RuntimeEcsContractGraphV1`、候補の評価状態と選択理由は[Performance／Capacity](../04-runtime/performance-capacity.md)の`OptimizationDecisionProjectionV1`、AI route／authorization／Task contextは[AI Security／Approval](../01-governance/ai-security-approval.md)、Eval／Receipt／freshnessは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)が所有する。本書はそれらのField、状態、権限またはEvidenceを複写しない。

同じAI contextへ複数fragmentを含める場合は、Project lineage、source revision、Target Profile、Contract Set、Toolchain、fixture、freshnessをexact一致させる。native address、allocator object、live lease、full allocation trace、credentialを投影せず、必要な集約値がredactionまたは未materializeにより確定できない場合は推測値ではなくBlocking diagnosticを返す。Contract fragmentの可読性はallocation、budget、pool化、alignment、failure policyを変更するauthorityを付与しない。

### 6.1 `PointerContractV1`

API parameter、return、fieldごとに次を必須にする。

```text
contract_id
type_id
ownership = value | unique_owner | shared_immutable | borrow | lease | handle
nullability = non_null | nullable | invalid_zero
lifetime = scope | callback | phase | simulation_advance | frame | job | submission | session | persistent
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
MemoryContractV1
  contract_id
  charge_domain_id
  memory_resource_class
  allocation_policy:
    system | monotonic_arena | size_class_pool
    | streaming_cache | external_adapter
  alignment
  capacity_source:
    profile | cooked_artifact | fixed_abi | measured_p999_headroom
  hard_limit_bytes
  steady_state_allocation_count
  allowed_phases[]
  allowed_threads[]
  reset_or_release_event
  upstream_fallback: forbidden | budgeted_system
  oom_behavior
  telemetry_tag_id
  source_owner
  storage_layout:
    contiguous_inline | contiguous_handle | chunk_column
    | ring | segmented | node_based | external
  element_storage: inline_value | typed_handle | external_adapter
  access_pattern:
    sequential | indexed | sparse_lookup | producer_consumer | infrequent
  growth_policy:
    pre_reserved_fixed | boundary_grow | budgeted_non_hot | forbidden
  address_stability:
    not_required | scope_bound | handle_resolved
  hot_path: bool
```

従来のallocation、budget、phase、thread、OOM、telemetryに関する15 Fieldはすべて保持する。`capacity_source`は一Fieldだけを持ち、従来proseのProfile、Cooked Artifact、fixed ABI、measured P99.9＋headroomを`profile | cooked_artifact | fixed_abi | measured_p999_headroom`のclosed enumへ固定する。末尾の`storage_layout`、`element_storage`、`access_pattern`、`growth_policy`、`address_stability`、`hot_path`だけが追加Fieldである。Source codeの説明なしmagic numberを許可しない。

`allocation_policy`は`system | monotonic_arena | size_class_pool | streaming_cache | external_adapter`、`upstream_fallback`は`forbidden | budgeted_system`のclosed enumとする。`oom_behavior`はDiagnostic Catalogへ登録されたclosed ID、domain、class、phase、thread、ownerはMCD登録済みIDとし、Providerが自由文字列を追加しない。

Generic containerとallocationは次のclosed ruleへ従う。

1. `std::vector<T>`は、exact `MemoryContractV1`がdefault System memory resourceを許すnon-ECS contiguous inline valueにだけ使う。
2. `std::pmr::vector<T>`はcontainerより長寿命のComposition Root-injected resourceとだけ使い、process-global default PMR resourceを変更しない。
3. ECS columnはECS-private chunk storageとgenerated bounded viewを使い、独立allocationを持つvectorにしない。
4. `std::vector<Handle<T>>`と`std::vector<T*>`はcontiguous referenceであり、`storage_layout=contiguous_inline`を満たさない。
5. known cardinalityはhot phaseへ入る前にreserveする。hot callback内の`reserve()`はreallocationしなくてもContract failureである。
6. reallocationでinvalidになるpointer、reference、iterator、spanをowning scopeまたはstructural boundaryの外へ出さない。
7. `storage_layout=node_based`をsequential hot traversalに使わない。infrequent control-plane lookupまたはintegrated Evidenceを持つmeasured exceptionだけを許す。
8. poolingはstable churnとmeasured improvementを持つprivate fixed-size objectに限定し、cheap C++ valueをEvidenceなしでpoolしない。
9. Frame、RenderFrame、Job scratchはbounded arenaを既存lifetime boundaryでresetし、exhaustion時にgeneral heapへfallbackしない。
10. hot callbackのgeneral-heap allocation countとupstream fallback countは両方exact `0`である。

Container ownerは一つのregistered `MemoryContractV1.contract_id`へ解決し、次の組を明示する。

| Owner class | `storage_layout` | `element_storage` | `access_pattern` | `capacity_source` | `growth_policy` | `address_stability` | `hot_path` |
|---|---|---|---|---|---|---|---|
| non-ECS dynamic value collection | `contiguous_inline` | `inline_value` | `sequential | indexed` | `profile | measured_p999_headroom` | `boundary_grow | budgeted_non_hot` | `not_required | scope_bound` | owner contract |
| typed handle collection | `contiguous_handle` | `typed_handle` | `sequential | indexed | sparse_lookup` | owner contract | owner contract | `handle_resolved` | owner contract |
| ECS Component column | `chunk_column` | `inline_value | typed_handle` | `sequential` | `cooked_artifact | profile` | `pre_reserved_fixed | forbidden` | `scope_bound | handle_resolved` | `true` |
| command／event queue | `ring | segmented` | `inline_value | typed_handle` | `producer_consumer` | `profile | cooked_artifact` | `pre_reserved_fixed | boundary_grow` | `scope_bound | handle_resolved` | owner contract |
| infrequent control-plane index | `contiguous_inline | node_based` | `inline_value | typed_handle` | `indexed | sparse_lookup | infrequent` | `profile | measured_p999_headroom` | `budgeted_non_hot` | owner contract | `false` |
| Vendor／OS adapter | `external` | `external_adapter` | owner contract | `fixed_abi | profile` | `forbidden | budgeted_non_hot` | `scope_bound | handle_resolved` | owner contract |

`owner contract`はそのrowのOwnerが登録するexact Field値であり、ambient defaultを意味しない。上表のunionから一つのclosed valueを選び、選択を`contract_id`のcanonical recordへ含める。

### 6.3 `CppValueTransferPolicyV1`

```text
CppValueTransferPolicyV1
  policy_version: 1
  contract_set_ref: ContractSetRefV1
  target_profile_ref:
    exact TargetProfileRefV1
  toolchain_lock_sha256: SHA-256
  target_abi_facts:
    data_model: ilp32 | lp64 | llp64
    abi_word_bytes: 4 | 8
    pointer_bytes: 4 | 8
  bindings[1..65536]:
    callable_key:
      api_contract_id: StableId
      api_contract_version: positive uint32
      api_contract_hash: SHA-256
    subject: parameter | return_value
    ordinal: optional uint16
    type_ref: McdContractRefV1(kind=type)
    pointer_contract_id: StableId
    semantic: input | input_output | output | sink | bounded_range
    transfer_form:
      value | const_borrow | mutable_borrow | return_value
      | move_sink | bounded_view | unique_owner
    value_class:
      scalar | enumeration | typed_handle | compact_trivial
      | bounded_view | nontrivial | move_only
    moved_from_policy: not_applicable | destroy_or_assign_only
    binding_hash: SHA-256
  policy_hash: SHA-256
```

`callable_key`はsource API contractを識別し、generated C++ signatureをhashしない。`pointer_contract_id`は`contract_set_ref`内でexact一件へ解決し、missing、multiple、cross-Contract-set resolutionをcompile failureにする。ABI factsはexact Target／Toolchainからだけ取得し、materialization前に`sizeof`、`alignof`、C++ trait、data model、word size、pointer sizeと照合する。

各`binding_hash`は次で計算する。

```text
SHA-256(
  ASCII "MIRAKAN_CPP_VALUE_TRANSFER_BINDING_V1"
  || uint32_be(len(canonical binding bytes excluding binding_hash))
  || canonical binding bytes excluding binding_hash
)
```

Bindingを`{callable_key canonical bytes, subject enum, ordinal presence, ordinal}`でstrict sort／deduplicateした後、自己`policy_hash`だけを除く同じframingとASCII domain `MIRAKAN_CPP_VALUE_TRANSFER_POLICY_V1`で`policy_hash`を計算する。Instanceはimmutable Contract Set root確定後にだけmaterializeし、自身の`contract_set_ref`が同じroot preimageへ戻るinstanceを挿入しない。

| Subject semantics | C++ form |
|---|---|
| scalar／enum／typed generation handle input | value |
| trivially copyable non-owner input up to two Target ABI words | value |
| other synchronous input | `const T&` |
| input／output | `T&` |
| bounded sequence | value-passed `std::span<const T>` or `std::span<T>` |
| ordinary output | return valueまたは既存`Result<T>` |
| move-only unique owner sink | value |
| unconditional non-owner sink | `T&&`、exact一回move |

Static Gateは次をrejectする。

```text
const T&&
return std::move(local) when local is copy-elision eligible
std::move from a const object
conditional move from a sink
moved-from access other than destruction or assignment
by-value input of an unqualified nontrivial type
const T& for scalar, enum, or typed handle without an approved measured exception
non-const reference parameter that is never written
output parameter where an ordinary return value is sufficient
shared_ptr parameter that neither expresses nor retains shared ownership
```

Return-by-valueはcopy elision／NRVOへ委ね、named localを直接returnする。explicit moveはownership hand-off、container insertion、最後のpre-move access後のdeclared sinkにだけ使う。private measured exceptionはexact Target Profile、fixture、baseline、improvement、static suppression scopeを持つADRを要求し、generated public APIへ継承しない。

### 6.4 生成物

Contract compilerは同じMCDから次を決定論的に生成する。

1. C++ safe API型とfunction signature。
2. `PointerContractManifest.bin`、`MemoryContractManifest.bin`、`PointerMemoryConsumerBindingManifest.bin`、`CppValueTransferPolicyManifest.bin`。
3. clang-tidy／AST scan用allowlistと、§6.3の全static rejection rule。
4. Development lease／thread／domain assertion。
5. AI Provider向け、native addressを含まないschema projection。
6. Diagnostic code、argument schema、修正候補。
7. Unit／negative／property test fixture。

Generated fileを手編集しない。Source contract、generated Header／Module、binary manifestのhash不一致はBuildを拒否する。

### 6.5 AI生成規則

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
| `MIRAKAN-MEMORY-HOT_PATH_FALLBACK` | hot pathが一般heapを要求 | performance test failure | fallbackせず当該phase fault |
| `MIRAKAN-POINTER-STALE_HANDLE` | generation不一致 | owner／create／destroy advance sequenceを表示 | typed failure |
| `MIRAKAN-POINTER-BORROW_EXPIRED` | epoch／phase失効後access | fail-fast | invalid actionをpublishしない |
| `MIRAKAN-POINTER-THREAD_AFFINITY` | 非owner threadでresolve／access | fail-fast | command／job resultをreject |
| `MIRAKAN-POINTER-UNSAFE_CAPABILITY_REQUIRED` | unsafe API権限なし | Build failure | artifact promotion拒否 |
| `MIRAKAN-POINTER-CONTRACT_MISSING` | 公開parameter／return／fieldにcontractなし | Build failure | 未昇格artifactに含めない |

Value-transfer／hot-callbackのexact Diagnostic登録は次である。

```text
diagnostic.memory.value-transfer-binding-missing
MIRAKAN-MEMORY-VALUE-TRANSFER-BINDING-MISSING
arguments = api_contract_id, api_contract_version, subject, ordinal?

diagnostic.memory.value-transfer-invalid
MIRAKAN-MEMORY-VALUE-TRANSFER-INVALID
arguments = api_contract_id, subject, ordinal?, rule_id

diagnostic.memory.hot-callback-allocation
MIRAKAN-MEMORY-HOT-CALLBACK-ALLOCATION
arguments = campaign_hash, scenario_id, payload_bytes, observed_count

diagnostic.memory.hot-callback-upstream-fallback
MIRAKAN-MEMORY-HOT-CALLBACK-UPSTREAM-FALLBACK
arguments = campaign_hash, scenario_id, payload_bytes, observed_count
```

先頭二件はBuild／static failure、後二件はPerformance Gate failureであり、failureしたcallback outputをpublishしない。

C++公開`Error` enumerator（PascalCase）とDiagnostic codeは次の1:1対応を正本とし、対応を持たないErrorまたはcodeを追加しない。

| Error | Diagnostic code |
|---|---|
| `MemoryBudgetExceeded` | `MIRAKAN-MEMORY-BUDGET_EXCEEDED` |
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
- handle resolve latency（P50／P95／P99）、stale／invalid resolve率、retire backlog。
- lease validation latency、expired／range／thread拒否率。
- hot path allocation数、upstream fallback試行数、arena reset後のhigh-water、pool contention／reuse／fragmentation。
- container ownerごとのstorage layout、element storage、access pattern、capacity source、growth policy、address stability、hot-path分類。

hardware counter（cache miss、branch miss、memory bandwidth）は、同一Target Profileで再現性を確認できる場合だけ補助Evidenceにする。counter未対応Targetで値を推測したり、sanitizer実行値を通常performance baselineとして採用したりしない。

hot callbackのgeneral-heap allocation countとupstream fallback countは両方exact `0`であり、missing sampleを0へnormalizeしない。

性能改善を本Projectの標準候補へ昇格するには、[Runtime performance／capacityの共通promotion threshold](../04-runtime/performance-capacity.md#8-measurementregressionpromotion)を満たすか、同一fixtureでpeak memoryを15%以上改善し、correctness、visual、fault、load timeを規定値以上悪化させない。全面pool化、lock-free化、custom allocator化を名称だけで最適化扱いしない。

## 9. TestとQualification

### 9.1 correctness

- generation slotのcreate、destroy、reuse、random invalid、wrap retire、space exhaustion。
- `ReadLease`／`WriteLease`のphase、epoch、thread、overlap、structural mutation失効。
- `MirakanMakePersistent`のallocation failure、`noexcept` construction、fallible explicit initialization失敗時の非公開owner回収、move、destructor一回、Port／size／alignment／tag一致。
- arena reset、pool reuse、double free、wrong resource、alignment 1～4096。
- Asset version leaseとGPU／Audio／Physics retireの同時条件。
- GPU multi-queue submission completion前のallocation／binding再利用禁止。

### 9.2 negative／static

- raw owning pointer、明示allocation、Runtime `shared_ptr`、borrow captureをcompile／AST Gateで拒否する。
- ABI HeaderへSTL、PMR、exception、owner wrapper、native typeが流出しないことをscanする。
- Contract、generated API、manifestのhash driftを拒否する。
- binding Matrixの全consumerが`reference_form`、保存、job capture、retire owner、qualification ownerを明示し、逆参照も一対一で閉じることを検査する。
- source callable subject集合と`CppValueTransferPolicyV1.bindings[]`のset equalityを検査し、unbound／orphan／duplicate ordinalを拒否する。
- `const T&&`、const objectからのmove、conditional sink move、moved-from reuse、copy-elision可能localへの`return std::move`、unqualified nontrivial by-value、unused mutable reference、不要output parameter、ownershipを表さない`shared_ptr` parameterをrejectする。
- C ABIへC++ reference、`std::span`、STL／PMR、`Result<T>`、exception、owner wrapperが流出するnegative fixtureを拒否する。
- Save／Replay／Package／AI projection／job packetにlive pointer、lease、span、allocator objectが混入するnegative fixtureを拒否する。
- ASan poison／unpoison、Full PageHeap、HWASan、GWP-ASan、Apple ASan／TSanを、[Toolchain／Dependencies](toolchain-dependencies.md)でTargetごとに実行可能と判定されたlaneだけで実行する。未対応laneをpassとして代用しない。

### 9.3 performance／endurance

- hot path一般heap allocation 0。
- ECS layout比較、chunk容量変更、query／structural workloadのfixtureは[Runtime ECS](../04-runtime/entity-component-system.md#10-qualification)と[Performance／Capacity](../04-runtime/performance-capacity.md)が所有する。本書は一般memory telemetryを提供する。
- 120秒×5 performance run、10分soakを通常Gateとする。
- Windows release候補は2時間resource churn、Asset hot reload、World create／destroy、GPU retireを追加する。
- R4 Memory／Pointer変更は最低60分、release候補は対象Targetのendurance規約を満たす。
- Mobileは10分実機run、30分thermal、2時間enduranceを満たす。

## 10. 導入順序

1. `PointerContractV1`、`MemoryContractV1`、`PointerMemoryConsumerBindingV1`、`CppValueTransferPolicyV1`、Diagnostic codeを同じPhase 0 definition closureへ追加する。
2. generation handle、slot registry、`Result`、memory tagをFoundationへ実装する。
3. System／Tracking／Budget／Failure Resourceを実装する。
4. Frame／RenderFrame／Scratch arenaとASan poisonを実装する。
5. `ReadLease`／`WriteLease`、epoch、thread-affinity検査を実装する。
6. Native Memory Port Adapterと`MirakanUniqueOwner` factoryを実装する。
7. Contract compilerからC++ API、四manifest、static rule、test fixtureを生成する。
8. ECS Ownerが要求するgeneral memory telemetryとhot path allocation Gateを接続する。ECS layout値やstorage実装を本書で再定義しない。
9. GPU allocator、submission retire、Asset version leaseを接続する。
10. Productの`gate.product.phase-0-memory-pointer-contract`、Windows、Mobile、Native Module、AI生成negative／performance／endurance Gateを合格させる。

後段機能が前段の未実装を独自pointer wrapperやlocal allocatorで迂回してはならない。

## 11. 受入条件

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
11. `PointerMemoryConsumerBindingV1`が全consumer conceptの正方向・逆方向を閉じ、非該当理由もregistered IDで記録する。
12. Product Phase 0のPointer／Memory gateが、contract closure、negative fixture、supported sanitizer lane、performance baselineのEvidenceを別々に検査する。
13. Definition closureが`PointerContractV1`、`MemoryContractV1`、`PointerMemoryConsumerBindingV1`、`CppValueTransferPolicyV1`のexact四Typeと四manifestを持ち、全generated C++ consumerが同じpolicyへ解決する。
14. C ABI adapterはC++ projectionと分離し、固定幅値、opaque handle、caller-owned bufferだけを公開する。

## 12. 根拠と判断

| 根拠／Owner | 採用した判断 |
|---|---|
| [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines) | raw pointer／referenceを非所有として扱い、single ownershipには`unique_ptr`相当を優先する |
| [C++ standard library: `unique_ptr`](https://eel.is/c++draft/unique.ptr)／[`span`](https://eel.is/c++draft/views.span)／[memory resources](https://eel.is/c++draft/mem.res) | owner、bounded view、memory resourceを標準語彙で表し、独自の公開pointer型を増やさない |
| [Microsoft AddressSanitizer](https://learn.microsoft.com/en-us/cpp/sanitizers/asan?view=msvc-170)／[LLVM AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html)／[LLVM ThreadSanitizer](https://clang.llvm.org/docs/ThreadSanitizer.html) | memory error検出は対応Targetの開発／CI laneへ隔離し、非対応laneの成功を捏造しない |
| [Unreal Engine Object Pointers](https://dev.epicgames.com/documentation/en-us/unreal-engine/object-pointers-in-unreal-engine) | 強、弱、soft、scope strong参照を用途別型へ分離する |
| [Unreal Engine Smart Pointers](https://dev.epicgames.com/documentation/en-us/unreal-engine/smart-pointers-in-unreal-engine) | non-intrusive unique／shared／weakとreference-count costを区別する |
| [Unreal Engine Memory and CPU Performance Considerations](https://dev.epicgames.com/documentation/en-us/unreal-engine/common-memory-and-cpu-performance-considerations-in-unreal-engine) | poolは生成破棄costをprofileしたobjectだけに使用する |
| [Unity Managed Memory](https://docs.unity3d.com/jp/current/Manual/performance-managed-memory-introduction.html) | GC allocation／collection spikeをMiraikanai hot pathへ導入しない |
| [Unity Unmanaged Memory](https://docs.unity3d.com/ja/current/Manual/performance-unmanaged-memory.html) | temporary、job、persistent allocationを寿命で分ける |
| [Unity Entities 1.4 manual: Archetypes concepts](https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/concepts-archetypes.html) | 同じComponent型集合をarchetypeとしてまとめ、同一archetypeのEntity／Componentをchunkへ格納し、Component型ごとの配列を持つlayoutだけを外部先例として採用する |
| [本書 §4.3](#43-generation-handle)／[Runtime ECS](../04-runtime/entity-component-system.md) | index＋generationという一般handle規則は本書、Runtime Entity handleとECS layoutはRuntime ECSが所有し、Unityの現行実装仕様として扱わない |
| [Godot Object ownership](https://docs.godotengine.org/en/stable/engine_details/architecture/object_class.html) | 非ownerの長期保持にraw pointerを使わずIDへ変換する |
| [Godot RID](https://docs.godotengine.org/en/stable/classes/class_rid.html) | low-level resourceをsession-local opaque handleで公開する |
