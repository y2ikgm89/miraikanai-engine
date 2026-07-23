# Miraikanai Engine Runtime ECS Contract 導入設計

- 文書種別: Architecture Decision／導入設計
- 状態: ユーザー確認待ち
- 提案日: 2026-07-22
- 対象: Runtime World、Entity／Component storage、Game System access、Authoring／Cook、Native Game Module、Subsystem integration、Debug／Replay、AI contract
- 実装方針: 後方互換性を設けない一括導入
- 外部根拠検証日: 2026-07-22

## 1. 結論

Miraikanai Engineは、Engine-ownedのarchetype／SoA／16 KiB chunk ECSをRuntime Worldの唯一の標準Entity／Component storageとして導入する。Flecs、EnTT、Unity Entities、Unreal MassのAPI、型、保存形式、scheduler、World semanticsは採用しない。公式一次資料から、同一Component集合によるarchetype化、列指向chunk、query cache、世代付きhandle、宣言的access、iteration中のstructural mutation禁止、deferred command、dataとlogicの分離という検証済み原則だけを採る。

新しい正本を`docs/architecture/04-runtime/entity-component-system.md`とする。現行の[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)、[Native Game Module](../03-authoring/native-game-module.md)に散在するECS固有規則は、同じ変更集合で新正本へ移す。旧記述、alias、互換wrapper、二重定義は残さない。

ECSはEngine全体を置き換えるobject modelではない。ECSが所有するのはPlay中のRuntime Entity identity、Component layout、archetype／chunk、query、access lease、structural transactionだけである。Authoring Document、Asset、System Public Contract、phase順、Subsystem native object、Save schema、Evidence、AI authorizationは既存Ownerが引き続き所有し、ECSはexact referenceとgenerated projectionで接続する。

Public Contractは一つの[Miraikanai Contract Definition（MCD）](../02-foundation/executable-contracts.md)からC++23 module、Native C ABI descriptor、TypeScript、JSON Schema、Provider／MCP schema、Editor metadata、human reference、conformance fixtureを決定論的に生成する。AIは同じContract Graphを読み、live Runtime ECSを直接変更しない。AIによる変更は完全登録済み外側MCD Operationのnamed inputへ閉じたtyped `ProjectChangePrimitiveV1`と`ProjectChangeSetV1`だけを通る。

## 2. 「公式推奨」の意味

ISO C++規格、Microsoft、Unity、Epic、Flecs、EnTTのいずれも、すべてのC++ Game Engineへ一つのECS実装を推奨してはいない。このため本設計で「公式推奨」と呼べる範囲を次に限定する。

1. 外部製品またはLibraryの挙動は、その提供元の公式documentation、公式repository、公式package metadataだけから確認する。
2. 外部実装の値やAPIをMiraikanaiの正本値として引用しない。採る原則と、独自に決めるContract／値を区別する。
3. Dependencyを採用する場合はToolchain Ownerがversion、commit、hash、license、Target、upgrade Gateを固定する。今回はRuntime dependencyを採用しないため登録しない。
4. 外部実装の「高速」という主張だけを採用根拠にしない。Miraikanai fixtureで正確性、決定性、memory、throughput、tail latencyを測定する。
5. Source、Save、Network、Public APIへ外部LibraryのID、型、layout、pointer、query DSLを露出しない。

したがって、16 KiB chunkはUnity互換のために選ぶ値ではなく、Miraikanaiが既に正本化したEngine-owned C1値を継承し、8／16／32 KiB比較で継続検証する。FlecsとEnTTは比較対象であり、暗黙Dependencyではない。

## 3. 解決する現行の曖昧さ

現行仕様には正しい方向性がある一方、実装開始に必要な責務とContractが次のように分散している。

| 現行記述 | 不足または衝突 | 本Decisionで固定する解決 |
|---|---|---|
| `Runtime World`がEntity locationとComponent storageを所有 | Component登録、型制約、World image、archetype planの正本がない | 新ECS正本が全Runtime storage contractを所有 |
| archetype chunk、SoA、16 KiB、64-byte alignment、256-byte上限 | MemoryとSchedulingに分散 | ECS正本へ値と意味を一括移管し、Memoryは一般memory規則だけを所有 |
| `EntityHandle`はindex＋generation | Authoring Entity StableId、Subsystem handleとの名前衝突 | Runtime専用名`RuntimeEntityHandle`へclean rename |
| `ComponentAccessManifest`を生成 | schema、access mode、queryとの一致、違反時挙動がない | `RuntimeComponentAccessManifestV1`をECS正本で定義 |
| `ReadLease`／`WriteLease`でquery | query schema、match、optional、iteration order、cache lifetimeがない | `RuntimeEntityQuerySpecV1`とtyped batchを定義 |
| Structural commandはstaging後にatomic commit | command集合、競合、temporary entity、failure scopeがない | `StructuralCommandBatchV1`のclosed operationとtransactionを定義 |
| `SceneEntityShardDocument`のEntity recordはAuthoring Componentを持つ | Runtime Componentへの変換、layout hash、Target差分がない | `RuntimeEntityProjectionV1`、`RuntimeComponentProjectionV1`、Root／Section imageを定義 |
| Game SystemはState ownerとtyped Portを持つ | ECS Component owner、read/write set、structural権限との結合がない | System ContractからAccess ManifestとContract Graphを生成 |
| Native moduleは`query_batches`を受ける | C++型安全性、ABI layout、lease invalidationの検証がない | generated C++ bindingとC ABI descriptorを同一hashへ束縛 |
| Renderer等はsnapshot／commandを消費 | どのSystemがECSから抽出し、どこでsealするかが不明 | SubsystemごとのIntegration Systemとphase boundaryを固定 |
| AIはAuthoringを検索・変更できる | ECS構成、System access、Runtime failureを読むbounded schemaがない | read-only ECS Catalog／Graph／Context Sliceを追加 |

本Decisionは既存仕様を否定するのではなく、既に選ばれた方針を実装可能な一意の契約へ閉じる。

## 4. 目的、完了条件、非目的

### 4.1 目的

1. Runtime Entityをdata-only Componentのcompositionとして保持し、同じComponent集合をarchetype／SoA chunkへ格納する。
2. Systemのread／write／structural accessを事前宣言し、scheduler、C++ compiler、runtime validator、AIが同じManifestを使う。
3. Authoring StableIdとRuntime handle、Source schemaとRuntime ABI、System Contractと実装を分離する。
4. iteration中のaddress安定性、structural mutation、並列write、handle再利用、失敗時atomicityを曖昧にしない。
5. Physics、Navigation、Animation、Rendering、Audio、VFX、UI、World streamingをpointer共有ではなくtyped Portで接続する。
6. SourceからRuntime Packageまでを決定論的に生成し、Save／Replayがraw ECS memoryへ依存しないようにする。
7. AIがComponent、System、owner、phase、query、transition、Budget、Diagnostic、根拠をboundedに取得できるようにする。

### 4.2 完了条件

本設計の導入は次をすべて満たした時に完了する。

- ECS固有の型、値、状態遷移、failure、Gateが新正本に一度だけ定義されている。
- ECS正本がArchitecture Indexへ厳密に一度掲載され、metadata由来のactive inventory再生成がArchitecture lintに合格している。固定件数を完了条件にしない。
- 旧`EntityHandle`と`ComponentAccessManifest`の曖昧な名称がactive仕様から消えている。
- Authoring→Compile→Load→Execute→Extract→Save／Replayの全境界に入力、出力、owner、hash、failureがある。
- 全active Systemがexact `RuntimeComponentAccessManifestV1`を持ち、未宣言accessをgenerated APIから表現できず、ABI迂回はQualification違反になる。
- 全Initializer Spec／初期Entityがexact lifetime ownerとComponent-owner initializerを持ち、全Entity Templateがidentity／runtime scope／Save policyとspawn boundを持つ。
- 全runtime archetypeと許可transitionがpackage load前にbounded planへ列挙される。
- 1／2／最大workerでauthoritative state digestとCommand／Event順が一致する。
- structural transactionへのallocation failure、stale handle、競合注入でlive Worldが部分変更されない。
- C++、TypeScript、JSON Schema、Provider／MCP、human referenceのContract fingerprintが一致する。
- AI変更経路のRuntime write operation数が0で、`ai_mutable` Authoring fieldの外側MCD Operation→typed change primitive coverageが100%である。

### 4.3 非目的

- Editor widget、Authoring Document、Asset Registry、Physics World、Render SceneをECS内部storageへ統合しない。
- Entityごとのvirtual `Update()`、inheritance hierarchy、service locator、global registryを提供しない。
- Flecs relationship／pair DSL、runtime string query、EnTT registry API互換を提供しない。
- C1でshared component、managed component、non-trivial inline component、arbitrary dynamic buffer、live component schema reloadを提供しない。
- C1でNetwork replication、distributed ECS、GPU-resident authoritative ECSを有効化しない。将来検討は`not_activated`であり、空interfaceを先行実装しない。
- ECSをGameplay System、Behavior Tree、Scene Graph、Asset Graph、Render Graphの代替にしない。

## 5. 比較した実装方式

| 案 | 利点 | 問題 | 採否 |
|---|---|---|---|
| A. Engine-owned archetype／SoA ECS | 既存16 KiB chunk、phase、State owner、MCD、AI projectionへ一対一で接続でき、Public ContractとBackendを分離できる | storage、query、tooling、検証を自前で実装する必要がある | 採用 |
| B. FlecsをRuntime Worldとして採用 | archetype、query cache、defer、staging、access modifier、toolingが成熟 | World／ID／relationship／query semanticsまでLibraryが所有し、MCD、single-writer、closed transition、AI schemaと二重正本になる | 不採用 |
| C. EnTT registryをRuntime Worldとして採用 | header-only、世代付きID、sparse-set、view／group、custom storageが柔軟 | Component別poolが標準で、既に固定したarchetype chunk／SoA、atomic structural plan、closed registrationと一致しない | 不採用 |
| D. Library adapter付きhybrid | 初期実装を短縮できる可能性がある | 二つのEntity ID、二つのquery、二つのlifetime、adapter copy、debug／AI projectionの分岐が恒久化する | 不採用 |

FlecsまたはEnTTをtest oracleやbenchmark比較へ使うことはできるが、Production binary、Public header、Source／Save形式、generated Contractへlinkしない。将来採否を変える場合もPublic ECS Contractを維持したBackend置換として別Decisionを必要とする。

## 6. 正本所有と関係Graph

### 6.1 Owner表

| 概念 | 唯一のOwner | ECSとの関係 |
|---|---|---|
| Product scope、Capability maturity | [Product Plan](../00-product/product-plan.md) | ECSをC1 Runtime基盤Capabilityとして参照 |
| MCD共通Envelope、Type／Operation／Diagnostic projection | [Executable Contracts](../02-foundation/executable-contracts.md) | ECS domain schemaを登録・生成 |
| 一般handle／lease／pointer／allocator規則 | [Memory／Pointers](../02-foundation/memory-pointers.md) | ECS handleとchunkが一般規則をspecialize |
| Authoring Entity／Component、StableId、ChangeSet | [Project State](../03-authoring/project-state.md) | Runtime projectionのSource |
| Cook／artifact／package envelope | [Asset Lifecycle](../03-authoring/asset-lifecycle.md) | Runtime World Root／Section imageを包み配布 |
| Game System Public Contract、State owner、typed Port | [Gameplay Programming Model](../03-authoring/gameplay-programming-model.md) | access manifest生成元 |
| Native ABI、module load、implementation promotion | [Native Game Module](../03-authoring/native-game-module.md) | generated ECS bindingを消費 |
| Entity／Component／archetype／chunk／query／structural transaction | 新しい`Entity Component System` | 本Decisionが新設する唯一のOwner |
| phase、tick、System order、lease boundary、message delivery | [Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md) | ECS queryとmutationの実行時点を決定 |
| capacity、memory、queue、structural command上限 | [Performance／Capacity](../04-runtime/performance-capacity.md) | ECS Profileがexact参照 |
| trace、capture、replay transport、crash evidence | [Debugging／Observability／Replay](../04-runtime/debugging-observability-replay.md) | ECS固有snapshot／diagnosticを収集 |
| World／Scene／Level／Cell、streaming intent | [World](../06-rendering/world.md) | Root／Section imageとactivation requestのSource |
| Domain native state | 各Simulation／Rendering／Platform Owner | ECSとはtyped Command／Event／Snapshotだけで接続 |
| Authorization、Approval | [AI Security／Approval](../01-governance/ai-security-approval.md) | AI ECS readとAuthoring proposalの権限を決定 |
| Evidence envelope、provenance | [AI Verification／Provenance](../01-governance/ai-verification-provenance.md) | ECS qualification resultを包む |

### 6.2 必須data flow

```text
Authoring Entity／Component (StableId, schema field)
  -> RuntimeComponentProjectionV1
  -> RuntimeWorldCompiler
  -> RuntimeWorldRootImageV1 + RuntimeWorldSectionImageV1 + RuntimeArchetypePlanV1
  -> DerivedArtifactManifestV1 + RuntimeWorldQualificationBindingV1 + Catalog
  -> RuntimeWorldLoader
  -> participant prepare + hidden attach
  -> RuntimeWorldPublicationHandle
  -> RuntimeWorld (RuntimeEntityHandle, archetype, chunk)
  -> RuntimeEntityQuerySpecV1
  -> Game System typed QueryBatch
  -> private Command／Event／StructuralCommandBatch
  -> Scheduler canonical merge and ECS atomic commit
  -> authoritative World state
  -> sealed Subsystem Snapshot／Command
  -> Physics／Navigation／Animation／Render／Audio／VFX／UI Adapter
  -> normalized result／Evidence／Replay
```

逆向きのwriteを許可しない。Subsystem callback、Renderer、Editor、AI、Debug UIはRuntime World pointerまたはComponent referenceを保持せず、owner Systemへtyped inputを返す。

## 7. Clean-break移行

現時点では実装済みECS package、公開SDK利用者、Save、release済みABIが存在しない。そのため移行costを将来へ持ち越さず、次を一括で行う。

| 旧概念 | 新しい正規概念 | 移行規則 |
|---|---|---|
| `EntityHandle` | `RuntimeEntityHandle` | active仕様を全置換。type aliasを残さない |
| `ComponentAccessManifest` | `RuntimeComponentAccessManifestV1` | fieldを新schemaへ移し、旧名称を残さない |
| 暗黙のWorld query | `RuntimeEntityQuerySpecV1` | 全Systemがexact query refを持つ |
| 自由なComponent registration | `RuntimeComponentSpecV1`＋generated registry | runtime `typeid`、名前hash、manual IDを禁止 |
| 動的に発生し得る任意archetype | `RuntimeArchetypePlanV1` | packageにないcomposition／transitionを拒否 |
| 任意の即時add／remove | `StructuralCommandBatchV1` | phase中APIを公開せず、指定boundaryでatomic commit |
| Runtime memory dump Save | `SaveReplayContractV1` field projection | raw chunk、padding、handleを永続化しない |
| live ECS mutation tool | typed `ProjectChangePrimitiveV1` | Runtime用R1／R2 write operationを生成しない |
| Asset専用`DerivedArtifactManifest` | tagged subjectを持つ`DerivedArtifactManifestV1` | AssetとWorld Artifactを一つのCatalog規則へ移し、旧readerとsynthetic Asset IDを残さない。released済み旧majorが存在しないため新設名はV1とする |
| symbolic／`bool` Native ECS descriptor | fixed-width codeの`MirakanNativeEcsColumnViewV1` | C ABIへ`uint32` codeと`abi_version = 1`だけを出す |
| 全AI channelへ一律MCP grant | AI Securityのsigned `AiCallerContextV1` execution route | MCP grantを`standard_external_mcp`だけへ限定し、Engine Provider／managed Hostは各正本Profile／Attestationを使う |

旧schema version、redirect、deprecated annotation、compatibility adapter、dual write、best-effort importは作らない。将来の正式release後にContractを変更する場合は、今回のclean breakと混同せず、明示的なversioned migrationを別Decisionで設計する。

`DerivedArtifactManifestV1`と`McdCanonicalBinaryV1`導入時は既存Derived Cache、Catalog、Package、Replay fixtureを全invalidateしてcommitted Sourceからfull recookする。旧suffixなしManifestのArtifactまたは旧binaryを新形式へ変換して再署名せず、Source Document／Asset Import Documentだけを入力にする。

## 8. Canonical object model

以下の疑似schemaで`[]`に数値を書かない箇所もunboundedを意味しない。E0 Contract compilerは各array／string／blobへ、型固有C1 cardinalityまたはexact Target／Capacity Profile refから解決した`min_count／max_count／max_bytes`を必ず埋め、そのresolved Profile hashを親Contractへ含める。未解決、0件不可なのにmin 0、Targetごとのbound欠落、`size_t`上限への依存が一件でもあればschema生成とCookを拒否する。本文はPerformance Ownerの既存数値を複写せず、fieldの意味、owner、bound参照箇所を固定する。

### 8.1 `RuntimeComponentSpecV1`

ComponentはEntityへ一種類につき最大一つ付与できるdata-onlyなRuntime Typeである。System logic、allocator、callback、virtual function、native object ownershipを持たない。

```text
RuntimeComponentSpecV1
  schema_version
  component_type_ref
  semantic_role_ids[]
  state_class: authoritative | derived | presentation_only
  owner_system_ref
  representation_kind: tag | inline_value | typed_external_handle
  external_store_contract_ref: optional exact ref
  external_handle_type_ref: optional exact Type ref
  external_state_projector_ref: optional exact ref
  runtime_add_initializer_contract_ref: optional exact ref
  runtime_add_parameter_schema_ref: optional exact ref
  runtime_add_external_reservation_contract_ref: optional exact ref
  enablement: structural_presence | enable_bit
  presence_save_policy: persistent | reset_to_base
  enablement_save_policy: not_applicable | persistent | reset_to_initial
  source_projection_refs[]
  field_type_refs[]
  runtime_layout_fact_ref
  save_replay_field_refs[]
  save_reconstruction_contract_ref: exact ref
  target_profile_refs[]
  invariant_refs[]
  ai_exposure_policy_ref
  diagnostic_refs[]
  fixture_refs[]
```

`component_type_ref`はMCDのexact Type ID＋versionであり、C++型名、display name、source path、`typeid`値、compiler固有hashをidentityにしない。`owner_system_ref`は同じContract set内で一つだけで、他Systemはreadまたはownerへのtyped Commandだけを持つ。

`tag`はfieldと三つのexternal fieldを持たずsize 0、`inline_value`は三つのexternal fieldを持たず一件以上のdata fieldを持つ。inline valueはfixed-size、standard-layout、trivially copyable、trivially destructible、最大256 bytes、alignment最大64 bytesとする。raw／smart pointer、reference、vtable、function pointer、`std::string`、`std::vector`、PMR container、OS／GPU／vendor handleを含めない。可変長または大容量dataはAsset、Domain-owned store、typed poolのいずれかが所有し、Componentには`typed_external_handle`だけを置く。

`tag`はpresenceだけが意味を持ち、`enablement = structural_presence`だけを選択できる。`typed_external_handle`は三つのexternal fieldをすべて必須とし、`field_type_refs[]`はexact handle Type一件だけを持つfixed-width trivial valueである。resolve authority、generation、lifetime、failureをexternal store ContractとPointer Contractへ登録する。handle値は`serializable=false`かつauthoritative digest対象外で、Save／Replay／Networkは`external_state_projector_ref`が出すPersistent Identityとauthoritative Fieldだけを使う。外部objectがauthoritativeならprojectorのSave／digest両projectionを必須とし、ephemeral slot／generationをstateそのものとして比較しない。C1ではshared valueでarchetypeを分割するShared Componentと、Component内dynamic bufferを定義しない。

`typed_external_handle` Componentへ`read_write` Leaseを生成しない。handle値はComponent存在中immutableで、create／add／persistent handoffのprepared transactionだけが設定する。外部objectの値変更はowner storeのtyped Port／State API、handleの置換は別boundaryのremove後にowner-approved addで行い、raw handle assignmentまたは汎用rebind APIを提供しない。

Archetype Planのadd transitionへ現れるComponentは、ownerが定義する`runtime_add_initializer_contract_ref`と空parameterでもexactな`runtime_add_parameter_schema_ref`を必須とする。このinitializerはComponent valueに加え、`enable_bit`なら初期enable bitも一意に出力する。`typed_external_handle`ならさらに`runtime_add_external_reservation_contract_ref`を必須とし、それ以外はこのFieldを持たない。add transitionへ現れないComponentは三Fieldをすべて持たない。これにより同じComponentのruntime addはsource archetypeに依存しない一つのtyped parameter contractを使い、commit時に選ばれたedgeごとに別schemaを推測しない。

`enable_bit`はComponentを移動せず一時的にquery対象外へできる。`inline_value`または`typed_external_handle`で`enablement = enable_bit`を選んだComponentは、chunk内のentity slotごとにexact 1 bitを持つ専用bitsetを一件持つ。bit indexはrow indexと同一で、複数Component間で共有しない。`tag`は`structural_presence`だけを選ぶためComponent enable bitsetを持たず、State StoreのsingletonにもECS Component bitsetを割り当てない。Entity enable bitsetはComponent enable bitsetと別の一件である。`structural_presence`は`enablement_save_policy = not_applicable`、`enable_bit`は`persistent | reset_to_initial`を必ず選ぶ。enable bitの変更はWorld mutationであり、owner SystemのManifestへ宣言し、指定boundaryで適用する。change versionはincremental extractionとdebugの派生metadataであり、authoritative Gameplay分岐、Save field、Stable IDに使わない。

`presence_save_policy = persistent`はSave対象Entityの最終Component集合へpresenceを投影し、`reset_to_base`はload時にAuthoring Section／Runtime Templateのbase compositionへ戻す。presentation-only／derived Componentをpersistentにする場合もownerのSave／rebuild Contractが必要で、Save fileが未知Componentを追加する権限にはならない。load後compositionは`RuntimeArchetypePlanV1`に存在し、各差分がComponent owner policyとmigrationに合格しなければならない。

全Componentはowner生成の`save_reconstruction_contract_ref`を持つ。Contractはbase Section entity recordまたはTemplate、Component presence policy、schema-versioned Save Field projection、external state projectorから、load後のtag presence／inline value／external preparationと、`enablement_save_policy = reset_to_initial`時のinitial bitをTarget別に一意に出力する。`persistent` Componentがbaseに存在しない場合もこのContractだけが構築し、raw runtime-add parameter、old handle、paddingをSaveへ暗黙追加しない。initializer parameterがpost-load authoritative stateへ影響するなら、その効果をownerのSave Fieldまたはexternal projectorへ完全投影できなければContract compileを拒否する。`reset_to_base`でbaseに存在しないComponentはField recordもexternal objectも復元せず、存在していたruntime stateをorphanとして残さない。

Project ComponentのC++ structはMCDから生成する。Engine Componentを手書きする場合もgenerated layout assertion、field ID table、schema fingerprintへ完全一致しなければ登録できない。unknown field、paddingを含むraw serialization、implicit numeric conversionを拒否する。

`RuntimeComponentRegistryV1`はContract set hash、Target／C++ ABI Profile、Component entry、registry hashを持つ。entryはComponent Type ref、`RuntimeComponentSpecV1` hash、layout fact refを持ち、Type refのcanonical byte順に`component_runtime_id: uint32`を1から割り当て、0をinvalidとする。runtime IDをSource、Save、Replay public schemaへ保存しない。RegistryはRoot load前に閉じ、ModuleまたはSectionが実行時にTypeを追加しない。

### 8.2 `RuntimeEntityHandle`

```text
RuntimeEntityHandle
  index: uint32
  generation: uint32
```

全bit 0をinvalidとし、valid index／generationは1から始める。destroyしたslotはgenerationを1増やして再利用し、generationがwrapするslotは永久retireする。allocationはcanonical commit順で最小の再利用可能indexを選び、なければ末尾へ追加する。同じRoot／bootstrap Sectionと同じcommand列から同じhandle列を得る。

HandleはWorld-local、session-local、非所有である。Source、Authoring、Save、Network、Artifact、別World、別sessionへ保存しない。64-bit Handle単体はWorld identityを含まないため、World-bound callback外のPublic Portは次を使う。

```text
RuntimeEntityRefV1
  world_instance_handle: RuntimeWorldInstanceHandle
  entity_handle: RuntimeEntityHandle
```

`RuntimeWorldInstanceHandle`もindex32＋generation32のtyped handleで、process-owned World slot poolに対しall zero invalid、destroy時generation increment、wrap retireを同じく適用する。同じWorldへ束縛済みの`RuntimeSystemExecutionContextV1`、QueryBatch、Component内部だけはbare `RuntimeEntityHandle`を使える。Command、Event、async request、Subsystem Port、Debug capture request等、callbackを越える値は`RuntimeEntityRefV1`を必須とする。resolveはWorld instance handle generation、current publication、Entity generation、alive stateを検査し、失敗時にnull objectまたは別Worldの同じslotへ変換しない。activation generationまたはtick freshnessが必要なPortは§12の`generation_and_stale_policy`を追加し、World instance generationと混同しない。

```text
RuntimeEphemeralEntityOrdinal
  value: uint64, 1..2^64-1
```

Persistent Entity Identityを持たないruntime createへ、Engineはsuccessful structural commitのcanonical create順でWorld内1から`RuntimeEphemeralEntityOrdinal`を割り当て、再利用しない。失敗transactionはcounterを消費せず、overflow前にWorld faultとしてwrapしない。live ephemeral Entityのhandle↔ordinalはECS-owned cold indexで保持する。per-Entity ordinalはGameplay query／allocation／Source／Artifact／Networkへ公開せず、authoritative digest、Replay、diagnostic projectionだけが使う。Saveはephemeral Entityまたは個別ordinalを保存せず、継続発行用のWorld-level last-issued counterだけを保存する。Section EntityはAuthoring identityを持ち、persistent runtime spawnはPersistent Identityを持つためordinal対象外である。

```text
PersistentEntityIdentityV1
  kind: authoring_entity | runtime_spawn
  authoring_entity_id: optional StableId
  owner_state_instance_id: optional StableId
  spawn_stream_id: optional registered StableId
  canonical_spawn_sequence: optional uint64, 1..2^64-1
```

`authoring_entity`は`authoring_entity_id`だけを必須とし、`runtime_spawn`は後三Fieldだけを必須とするclosed tagged unionである。不要Field、unknown kind、all-zero／未登録Stable IDを拒否し、表示名またはRuntime handleからidentityを推測しない。

Authoring起源またはSave対象Entityは`RuntimeIdentityIndex`で`PersistentEntityIdentityV1`と対応付ける。対応表と発行sequenceはchunk内のhot ComponentではなくWorld-owned cold side tableである。runtime spawnの発行規則はTemplateが参照する次のContractへ閉じる。

```text
RuntimeIdentitySequenceSlotSpecV1
  slot_spec_id: registered StableId
  owner_system_ref
  owner_runtime_instance_scope
  spawn_stream_id: registered StableId
  owner_state_persistent_id_contract_ref
  save_replay_contract_ref
  slot_spec_hash

RuntimeIdentitySequenceSlotRefV1
  slot_spec_ref
  owner_state_instance_id: StableId

RuntimePersistentIdentityPolicyV1
  policy_id: registered StableId
  mode: forbidden | optional | required
  issuer_system_ref
  sequence_slot_spec_ref: optional RuntimeIdentitySequenceSlotSpecV1 ref
  migration_contract_ref: optional exact ref
  policy_hash
```

全modeで`issuer_system_ref = Templateが参照するRuntimeEntityInitializerSpecV1.lifetime_owner_system_ref`とする。`forbidden`は全optional fieldを持たず常にephemeral、`required`は常にpersistent、`optional`はgenerated create parameter `persistence_disposition: ephemeral | persistent`だけを追加する。`optional | required`は同じissuerをownerに持つSequence Slot Specを必須とし、streamとSave／Replay ContractはSpecだけを正本にする。runtimeではSystem instanceのSave-stable `owner_state_instance_id`をSpecへ結び`RuntimeIdentitySequenceSlotRefV1`を一意に作る。scopeがpersistent owner state IDを発行できない場合、そのinstanceからpersistent Entityをspawnできない。Game Systemはidentity値またはsequenceを入力できない。Engineは`required`または`optional + persistent`のcreateだけでcanonical command順の次sequenceをstructural transaction内に予約し、ECS tables、Identity Index、sequence slotを同じpublicationで進め、失敗時はどれも消費しない。`forbidden`と`optional + ephemeral`はSequence Slotを読まず、Identity Index entryを作らない。

Slot Specの`{owner_system_ref, owner_runtime_instance_scope, spawn_stream_id}`はContract set内で一意、`owner_state_instance_id`は同じSave lineage内で一意かつ再利用不可とする。runtime generation handle、Level array index、display nameをowner state persistent IDへ変換しない。

runtime spawn identityは`{owner state instance, registered stream, canonical sequence}`からだけ発行し、accepted structural batchとReplayへ結果を記録する。wall clock、UUIDv7生成時刻、worker index、Runtime handle、乱数を使わない。sequence overflow、同一tuple重複、load時collisionはhard failureで、wrapまたは新UUIDへfallbackしない。Authoring identityの注入はCook／loader、保存済みruntime identityの復元はSave reconstruction gatewayだけに許可し、通常の`create_entity`へidentity値を渡さない。ephemeral Entityに暗黙persistent identityを生成しない。

Entity間関係はFlecs pair相当の汎用relationへ変換せず、`ParentEntityRef`、`TargetEntityRef`、`AttachmentRef`等のDomain所有MCD Componentとして定義する。同一World内のComponent値だけはbare `RuntimeEntityHandle`、callbackを越えるPortは`RuntimeEntityRefV1`、永続／cross-World参照は後述のPersistent Entity Identityを使い、loaderまたはowner Systemが明示的にresolveする。children、reverse target、spatial membership等の逆引きはowner Systemのderived indexであり、SourceまたはComponentへ二重保存しない。

C1のComponent内`RuntimeEntityHandle`はすべてsame-Worldの`weak_checked`参照で、generated accessorは`resolved | stale`のtyped resultを返し、pointer、reference、null objectへ暗黙変換しない。target destroyは参照元をcascade destroy、clear、retargetせず、関係ownerがderived reverse indexとtyped Commandで必要な変更をdestroy前のboundaryへ明示する。Save／handoffの`required` targetはcheckpoint／reconstruction／publication時の整合条件であり、Runtimeの所有権または暗黙cascadeを意味しない。strong owning relationと汎用cascade policyはC1で`not_activated`とする。

`RuntimeEntityHandle` fieldは`serializable=false`であり、Save projectorは対応するPersistent Entity Identity、Replay／authoritative digest projectorはPersistent Identityまたは`RuntimeEphemeralEntityOrdinal`へ置換するか、そのrelationをderivedとして明示除外する。raw index／generationをcross-session digestへ入れない。persistent identityを持たないtargetへのrelationをSaveへ近似せず、required relationならSave validationを失敗させる。

World／extension-owned scopeのStateを「singleton Entity」へ偽装しない。これらはcurrent `GameSystemSpecV2.runtime_scope_type_ref`に対応するSystem-owned State Storeまたはtyped Resource Portが所有する。Command、Event、Snapshotも一時Component／tagとして配送せず、Scheduling Ownerのtyped queueを使う。

各EntityはWorld-owned side metadataにexact `lifetime_owner_system_ref`と`RuntimeEntityScopeRefV1`を一つずつ持つ。lifetime ownerだけがそのEntityのdestroyを要求できる。scopeはAuthoring Section／handoffまたはTemplateの`runtime_scope_policy`からだけ設定し、通常Commandで変更しない。Component ownerはComponent値、add／remove／enableを所有し、Entity lifetimeまたはscopeを暗黙所有しない。初期Authoring Entityとruntime spawnのownerは、いずれも使用するInitializer Specが決める。

### 8.3 `RuntimeEntityInitializerSpecV1`と`RuntimeEntityTemplateV1`

Section初期Entity、persistent handoff、runtime spawnで別々の初期化規則を作らない。Component owner承認済みの共通Initializer Specを作り、runtime spawnだけがbounded Templateでそれを公開する。

```text
RuntimeEntityInitializerSpecV1
  schema_version
  initializer_spec_id
  source_recipe_or_definition_refs[]
  archetype_key
  lifetime_owner_system_ref
  component_initializers[]
    component_type_ref
    initializer_contract_ref
    initial_enablement_contract_ref: optional exact ref
    canonical_default_values
    parameter_bindings[]
    external_handle_reservation_contract_ref: optional exact ref
  parameter_schema_ref
  target_profile_refs[]
  initializer_spec_hash

RuntimeEntityTemplateV1
  schema_version
  template_id
  template_revision
  source_recipe_or_definition_refs[]
  initializer_spec_ref: exact RuntimeEntityInitializerSpecV1 ref
  persistent_identity_policy_ref: exact RuntimePersistentIdentityPolicyV1 ref
  runtime_scope_policy: world_root_runtime | producer_activation_group
  initial_entity_enabled: bool
  entity_lifetime_save_policy: not_saved | persistent
  entity_enablement_save_policy: not_saved | persistent | reset_to_initial
  spawn_budget_ref
  template_hash
```

各`initializer_contract_ref`はComponent ownerが定義し、変更可能parameter、range、unit、default、invariantを固定する。`enable_bit` Componentは同じownerの`initial_enablement_contract_ref`を必須とし、fixed boolまたはtyped parameter bindingから初期bitを一意に出力する。`structural_presence`はこのrefを持たない。`typed_external_handle` initializerは§10.3のowner発行Reservation Contractも必須とする。Section cooker、handoff compiler、Template creatorはいずれもInitializer Spec IDとtyped parameterだけを渡し、Component Type ID、field map、archetype、external handle値、persistent identity値を任意指定しない。Compilerは全initializer、owner、parameter binding、Target、archetypeを検証し、Initializer hashをTemplate、Section record、Handoff、Archetype Planへ束縛する。

Templateのeffective lifetime ownerはInitializer Specのownerであり、identity policyのissuerも同じownerでなければならない。identity policyが`forbidden`のTemplateは`entity_lifetime_save_policy = not_saved`かつ`entity_enablement_save_policy = not_saved`だけを許可し、scopeは二variantから選べる。`optional | required`は`runtime_scope_policy = world_root_runtime`、lifetime `persistent`、enablement `persistent | reset_to_initial`を必須とし、optional Templateのephemeral instanceも同じroot scopeだがSave対象外である。`producer_activation_group`はidentity forbiddenなephemeral instanceだけに許可し、create producerのexecution contextがexact一件のactive groupへ束縛されていなければ拒否する。callerはscopeをoverrideできない。

AuthoringのComposition RecipeまたはGameplay DefinitionはInitializer／TemplateのSourceになれるが、両方ともDerived Artifactである。AI／Editorは直接編集せず、Sourceへのtyped `ProjectChangePrimitiveV1`を提案する。Root／Section内の一回限り初期Entityもexact Initializer Specとresolved parameter hashを持ち、loader専用の抜け道を作らない。`create_entity`が公開できるのはTemplateだけで、Initializer Specを直接spawn APIへ渡せない。

### 8.4 `RuntimeArchetypePlanV1`

```text
RuntimeArchetypePlanV1
  schema_version
  component_contract_set_hash
  target_profile_ref
  entity_initializer_spec_refs[]
  entity_template_refs[]
  archetypes[]
    archetype_key
    component_type_refs[]
    layout_fact_ref
    initial_capacity
    hard_capacity
  allowed_transitions[]
    transition_id: registered StableId
    source_archetype_key
    operation: add_component | remove_component
    component_type_ref
    destination_archetype_key
    initializer_contract_ref: optional exact ref
    initializer_parameter_schema_ref: optional exact ref
    external_handle_reservation_contract_ref: optional exact ref
    transition_spec_hash
  plan_hash
```

`archetype_key`はComponent Type refの重複なし昇順集合からcanonical encodingで導出する。Runtime dense `archetype_id`はpackage内の`archetype_key` byte順に1から割り当て、0をinvalidとし、Source／Saveへ保存しない。初期Entityだけでなく、全active Systemのstructural権限から到達可能なtransition closureをCompilerが列挙する。循環は許可するが、未列挙Component、unbounded combination、到達不能archetype、hard capacityなしのplanを拒否する。

`add_component` transitionはdestination Component ownerの`RuntimeComponentSpecV1`にあるinitializer／parameter schema refをexact複写し、`typed_external_handle`では同じownerのexternal reservation refもexact複写する。それ以外のrepresentationではexternal refを持たない。`remove_component`は三つのoptional fieldをすべて持たない。同一`{source_archetype_key, operation, component_type_ref}`のedgeは厳密に一件とする。`transition_spec_hash`はtransition ID、source／destination、operation、Component Type、三契約refをcanonical encodingして導出し、Archetype Plan、Access Manifest、Structural Command validatorへ同じ値を束縛する。

Runtimeは新しいComponent組合せを発見してarchetypeを暗黙生成しない。planにないadd／removeは`ECS_TRANSITION_NOT_DECLARED`でtransaction全体を拒否する。これによりarchetype explosion、query cacheの実行中再構築、未計上memoryを防ぐ。

`RuntimeComponentAccessManifestV1.structural_permissions[]`はoperation、add／removeで許可するexact transition ref集合、`create_entity`で許可するTemplate ID＋hash集合、destroy／enableで許可するEntity／Component policy binding、apply boundary ID、count／byte Budget refを持つ。CompilerはInitializer owner、Templateのidentity／lifetime save policy、Authoring Entityのdestruction policy、Componentのpresence／enablement save policyもpermissionへhash bindingし、Manifestが永続化効果を上書きできないようにする。`create_entity`／`destroy_entity`／`set_entity_enabled`はEntity lifetime owner、`add_component`／`remove_component`／`set_component_enabled`は対象Component ownerだけに生成する。自由な「任意Componentを追加可能」というpermissionを定義しない。

### 8.5 Chunk layout

一つのchunk payloadは16 KiB、先頭alignmentは64 bytesである。Chunk control headerはEngine-owned fixed pool、payloadはECS chunk poolが所有し、両方をSystemへ公開しない。payloadにはEntity handle列、Entity enable bitset、`inline_value`のComponent列、Component enable bitsetをSoAで配置し、tagは列を持たない。

`RuntimeChunkLayoutFactV1`はlayout algorithm ID、Target ABI、Component Contract hash、row capacity、各列のoffset／stride／size／alignment、Entity列、Entity／Component enable bitset、payload used／slack bytesを持つ。Runtime loaderはCompilerが生成したFactを再解釈せず検証して使う。Factとgenerated C++ layout fingerprintが一つでも不一致ならPlay開始を拒否する。

C1のlayout algorithm IDは`ecs_chunk_soa_v1`である。候補row数`N`ごとに、payload offset 0から`RuntimeEntityHandle[N]`をalignment 8で置き、次にEntity enable bitsetをalignment 8、size `ceil(N / 64) * 8`で置く。続いてnon-tag Component列を`alignment`降順、同値はComponent Type refのcanonical byte順で置く。各列先頭をそのComponent alignmentへ切り上げ、列sizeを`sizeof(Component) * N`とする。最後に`enablement = enable_bit`のComponentだけについて、Component enable bitsetをComponent Type ref順、alignment 8、各size `ceil(N / 64) * 8`で置く。`structural_presence`のComponentはbitset列を持たない。1以上でpayload使用量が16 KiB以下となる最大`N`をrow capacityとする。Target compilerのcontainer layout、declaration順、link順から列順を推測しない。

row capacityは16 KiB payloadへ全列とbitsetが収まる最大整数で、0はCompile errorである。Componentの除去またはEntity破棄はchunk末尾rowをgapへmoveし、移動Entityのlocation tableを同じcommitで更新する。Component address、span、row、chunk IDはmove、structural commit、phase終了で無効になる。

Chunk IDは`uint32`の1からWorld内で単調増加し、0をinvalid、session中は再利用しない。staging失敗はChunk ID counterを消費せず、commitでだけ進める。Profileが証明したhard chunk countにより`UINT32_MAX`到達前にpreflightを拒否し、wrapしない。iteration順は`archetype_id, chunk_id, row`の昇順である。worker index、allocation address、hash map bucket、OS completion順をiteration identityにしない。

Chunkのfield値はgenerated decoderがMCD canonical field encodingから構築する。新しいrow storage全体をzero-fillしてからfieldを書き、native paddingをSource、Artifact、Save、Replay、digest、AI captureへ露出しない。Component digestはField IDとcanonical valueから計算し、`RuntimeEntityHandle`／typed external handleは各ownerのPersistent Identity／authoritative projectorへ置換または明示除外し、raw `sizeof(T)` bytesをhashしない。per-Component change epochはpayload外のpreallocated `RuntimeChunkRecord` metadataに置き、Gameplay query predicateまたはSave fieldにしない。

### 8.6 Runtime World Root／Section image

Play sessionで不変なのはComponent／System Contract、Registry、Template、Archetype Planを閉じるRoot imageと、個々のCook済みSection bytesである。live Entity集合はruntime spawn／destroyと、宣言済みSection activation／deactivationにより変化できる。

```text
RuntimeWorldRootImageV1
  schema_version
  project_revision
  source_document_root_hash
  contract_set_hash
  target_profile_ref
  runtime_ecs_profile_hash
  component_registry_hash
  entity_construction_set_hash
  archetype_plan_hash
  system_implementation_set_ref
  state_owner_table_hash
  world_streaming_plan_ref
  section_catalog_hash
  bootstrap_activation_group_ref
  exact_asset_dependency_refs[]
  save_replay_contract_set_hash
  qualification_policy_hash
  required_gate_set_hash
  root_image_hash

RuntimeWorldSectionImageV1
  schema_version
  root_image_hash
  section_id
  world_streaming_plan_ref
  activation_group_id
  ordered_cell_refs[]
  source_ref_merkle_root
  hard_dependency_activation_group_ids[]
  entity_records[]
  source_identity_table
  persistent_handoff_table_hash
  required_capacity_ref
  exact_asset_dependency_refs[]
  qualification_policy_hash
  required_gate_set_hash
  section_image_hash

RuntimeWorldSectionRefV1
  root_image_hash
  activation_group_id: uint32, 1..N
  section_image_hash

RuntimeWorldSectionCatalogEntryV1
  activation_group_id: uint32, 1..N
  section_artifact_subject_ref: ArtifactSubjectRefV1
  section_content_hash
  hard_dependency_activation_group_ids[]
  required_capacity_hash
```

Root／SectionはTarget別Cooked Artifactであり、Authoring正本ではない。Pointer、native handle、vtable、compiler process address、Editor metadata、Provider情報を含めない。一つのSectionは[World](../06-rendering/world.md)の一activation groupへ対応し、C1の`section_id`は同Planの`activation_group_id`と同じ`uint32`値で、独立したidentityを作らない。`ordered_cell_refs[]`は同PlanのCell ID昇順である。live／outer cross-record参照は`RuntimeWorldSectionRefV1`を使い、`section_image_hash`単体、表示名、Package offsetをidentityにしない。Root内のcatalogは後述のRoot非依存`section_content_hash`だけを持ち、最終`section_image_hash`またはArtifact keyを含めない。SectionはRootと同じComponent Registry、Entity Construction set、Archetype Planだけを使い、新Type、System、archetype、transitionを持ち込まない。

inner `exact_asset_dependency_refs[]`は`ArtifactSubjectRefV1.kind = asset`だけを許可し、World Root／Section Artifact key、Catalog hash、Qualification Bindingを含めない。World Artifact間の外側dependencyは`DerivedArtifactManifestV1.dependency_keys`とCatalog Launch Setだけが所有する。

初回PlayではPackage loaderがRoot、bootstrap activation group、全hard dependency、Target、layout、System implementation、State owner、capacityをstagingで検証し、全体成功時だけWorldをpublishする。部分bootstrap Worldをliveにしない。

Play中のSection activation／deactivationは、ECS storage、activation-group scopeのGame System State、Renderer、Collision、Physics、Navigation等を同じpublicationへ参加させる。participant contractを次へ固定する。

```text
RuntimeSectionPublicationParticipantV1
  participant_id: registered StableId
  participant_kind: ecs_storage | game_system_state | rendering | collision
                  | physics | navigation | animation | audio | vfx | ui
                  | typed_external_store
  owner_system_ref
  required_artifact_refs[]
  depends_on_participant_refs[]
  prepared_state_schema_ref
  capacity_ref
  prepare_contract_ref
  hidden_attach_contract_ref
  abort_contract_ref
  retire_contract_ref
  retire_fence_ref
  hidden_attach_order_key: uint32, 1..N
  participant_hash

RuntimeSectionReservationHandle
  index: uint32
  generation: uint32

RuntimeSectionParticipantReservationV1
  participant_ref
  base_world_epoch: uint64
  pending_activation_generation: uint64
  base_participant_generation: uint64
  prepared_participant_generation: uint64
  reserved_capacity_hash: bytes32
  prepared_state_hash: bytes32
  reservation_handle: RuntimeSectionReservationHandle

RuntimeSectionPublicationTransactionV1
  root_image_hash: bytes32
  base_world_epoch: uint64
  base_activation_generation: uint64
  pending_activation_generation: uint64
  ordered_section_changes[]
    operation: activate | deactivate
    section_ref: RuntimeWorldSectionRefV1
  ordered_participant_reservations[]
  persistent_handoff_table_hash: bytes32
  transaction_hash: bytes32

RuntimeWorldPublicationHandle
  index: uint32
  generation: uint32

RuntimeEcsStorageGenerationHandle
  index: uint32
  generation: uint32

RuntimeActiveSectionSetHandle
  index: uint32
  generation: uint32

RuntimeParticipantGenerationTableHandle
  index: uint32
  generation: uint32

RuntimeWorldPublicationRecordV1
  world_instance_handle
  root_image_hash
  artifact_catalog_snapshot_ref: ArtifactCatalogSnapshotRefV1
  runtime_world_launch_set_hash: bytes32
  world_epoch
  activation_generation
  ecs_storage_generation_handle: RuntimeEcsStorageGenerationHandle
  ecs_storage_topology_hash
  active_section_set_handle: RuntimeActiveSectionSetHandle
  active_section_set_hash
  participant_generation_table_handle: RuntimeParticipantGenerationTableHandle
  participant_generation_table_hash
  publication_record_hash
```

CompilerはRootのSystem／State binding、Section entity composition、external store、Subsystem Portからrequired participant setを生成する。`ecs_storage`は厳密に一件、activation-group scopeの各State Store instanceは対応する`game_system_state`、Section dataを持つ各Domain／external storeはowner participantを必須とする。Transactionのreservation集合は影響を受けるrequired participantをhidden attach orderで過不足なく一度ずつ持ち、未知、重複、optional omission、不要participantによる副作用を拒否する。State instanceのinitialize／load／retireも同じpublication generationで可視化し、EntityだけまたはStateだけを先行publishしない。

participant generationは1から始まる`uint64`で、影響を受けたparticipantだけがsuccessful publication(通常structural commitのpublicationとSection publicationの両方)でexact 1増え、未変更participantは同じ値を保つ。prepare／abortで消費せず、0、skip、decrement、wrapを許可しない。overflow前は新publicationを行わずWorld fault／teardownとする。

external storeの`store generation`は、そのstoreを所有するparticipant generationと同じcounterであり、別counterを持たない。ECS storage generation handle、owner participant generation、external store resolverの更新と可視性は次の一表だけに従う。

| Stage | ECS storage generation | 影響participant／store generation | visibility／stale rule |
|---|---|---|---|
| base capture | current publicationのhandleを固定 | current値を`base_*_generation`として固定 | acquireした同一publicationからだけ取得し、World／publication／base generation不一致はstale reject |
| prepare | pending storageを非公開で構築 | 影響ownerごとに`base + 1`をpending値として予約 | current counterを進めず、prepared object／reservationはconsumerからresolve不可 |
| pre-attach validation | base handle／epochを再照合 | 全ownerのcurrent値がbase値と一致することを再照合 | 一件でも不一致、欠落、重複、skip、overflowならattach前に全abortしrecoverable replan |
| hidden attach | pending generation handleへno-fail attach | prepared participant tableへpending値を記録 | publication slotはbaseを指し続け、pending storage／storeを通常resolverへ公開しない |
| atomic publish | 完成済みpending storage handleをRecordから参照 | 影響ownerだけexact 1増、未変更ownerは据え置き | 一回のrelease storeでpublication handleを切替え、acquire consumerへECSと全storeを同時公開 |
| abort／pre-publish failure | pending handleを破棄 | current値を消費せず全reservationをabort | base publicationが唯一の可視状態。部分generationは観測不能 |
| retire | old-publication Lease／fence完了後にold handleをretire | old store objectも同じfence後にownerがretire | 新resolverがold generationを受理せず、old Lease完了前の再利用を禁止 |

四種のgeneration handleはすべてall zero invalid、wrap retireの型別fixed pool handleである。`RuntimeWorldPublicationHandle`のcanonical packed value `(uint64(generation) << 32) | index`だけを一つの`atomic<uint64>` publication slotへ置く。Recordと三つの参照先metadataはpublish前に完成し、old publicationを参照するLease／fence完了までpoolからretireしない。`publication_record_hash`は同FieldをzeroにしたRecordのcanonical bytesをhashする。Record、四handle、record hashはsession-local integrityで、Source、Artifact、Save、Replay、authoritative digest、Entity allocationへ入力しない。ECS storage handleはchunk owner／location／identity／sequence／tombstone table、active Section set handleはcanonical Section refs、participant table handleはparticipant ID→prepared generation mappingを解決する。`ecs_storage_topology_hash`はarchetype／chunk ownershipとside-table topologyだけをhashし、phase中に変わるComponent valueを含めない。各解決時にWorld、Root、type、generation、record／metadata hashを検査する。

publishはrelease store、consumer captureはacquire loadとし、Target Profileは64-bit atomic lock-free conformanceを必須化する。不合格Targetへmultiword copyまたはmutex fallbackを暗黙導入せずunsupportedとする。

初回Root＋bootstrap publicationは`world_epoch = 1`、`activation_generation = 1`、全initial participant generation = 1とする。以後`world_epoch`は通常structural commitとSection publicationの成功ごとに1増やし、`activation_generation`はSection activation／deactivation／handoff成功時だけ1増やして通常structural commitでは維持する。0はinvalid、どちらもstaging失敗で消費せず、overflow前にWorld faultとしてwrapまたはresetしない。

activation／deactivation protocolは次の順だけを許可する。

`ordered_section_changes[]`はactivation group ID昇順・重複なしである。`activate`はbase active setに存在せずpending setへexact refを追加し、`deactivate`はbase setのexact refと一致してpending setから除去する。bootstrap groupのdeactivate、同groupのreplace、既activeへのactivate、非activeのdeactivateを拒否し、partial changeをskipしない。pending setはbootstrapと全active Sectionのtransitive hard-dependency closureを含み、active dependentが残るdependencyをdeactivateできない。Transactionの全changeとhandoffを適用して得たpending set hashは`RuntimeWorldPublicationRecordV1.active_section_set_hash`と一致しなければならない。

Section publicationはScheduling Ownerが生成する専用quiescent boundaryでだけcommitし、同じboundaryに通常`StructuralCommandBatchV1`を置かない。全Component／State Leaseを失効・回収し、先行structural boundaryを完了してからbase publicationを再取得する。Game System commandをSection transactionへ吸収せず、後続System phaseはSection publish成功後の新publicationから開始する。preflight中に先行structural commitでbase epochが変わった場合はhidden attach前に全Reservationをabortしてrecoverable replanし、stale Section transactionをsession faultにしない。

1. `RuntimeSectionPublicationPreflightV1`がI/O、inner／outer hash、dependency、generation、StableId、persistent handoff、Initializer／Template、capacity、Subsystem artifactをlive World変更なしで検証する。prepareはparticipant DAGのfrontierごとに並列化できるが、prepared result identityと後続順序にcompletion timeを使わない。全participantがcapacityとnative resourceを予約し、consumerからresolveできないprepared stateとReservationを返す。通常のI/O失敗、residency不足、cancel、stale Plan、prepare失敗はtyped recoverable resultであり、全Reservationをabortして旧publicationを維持する。
2. preflight成功時だけ`RuntimeSectionPublicationTransactionV1`を作る。Compilerは`depends_on_participant_refs[]`をDAG検証し、topological frontier内をparticipant ID canonical byte順にして重複のないdense `hidden_attach_order_key = 1..N`を割り当てる。participantは同key順でSchedulerのquiescent activation boundaryへ入る。base epoch、generation、token、prepared state hashを再検査し、live callback、I/O、allocationを行わずpending generation slotへhidden attachする。hidden attachはconsumerから観測不能で、失敗可能処理を含めない。
3. 全attach成功後、Runtime Orchestratorだけが一つのrelease atomic storeでpublication slotをpending `RuntimeWorldPublicationHandle`へ切り替える。ECS query、Subsystem adapter、external handle resolverはacquireしたHandleから`RuntimeWorldPublicationRecordV1`とgenerationを解決し、participant固有のlive flagまたは先行pointerを見ない。このstoreが唯一のexternal visibility pointである。
4. publish後callbackはnoexcept／no-allocation／no-failの通知だけとする。旧ECS chunk、Section、Subsystem state、external objectは旧publicationを参照するLeaseと`retire_fence_ref`が完了した後、participant逆順でretireする。

publish前のstale epochまたは失効Reservationは全participantをabortしてrecoverable replanへ戻す。hidden attach開始後にContract hash、precomputed mutation、location table、prepared generation等の内部不変条件が崩れた場合、新generationをpublishせず、旧generationを最後の可視状態に保ったままWorldをfaultさせる。publish後に失敗し得る処理を置かない。retire fenceのdeadlineとretired-generation backlogはparticipant `capacity_ref`でhard bound化し、超過時は新しいactivationを停止してWorld fault／teardownへ進み、旧resourceを早期解放または新publicationをrollbackしない。residency pressureをGame Systemのinvalid structural commandと同じsession faultにしない。

Section deactivationによる`section_owned` Authoring Entityと`producer_activation_group` runtime spawnの除去はGameplay `destroy_entity`ではなくstreaming lifecycleであり、Authoring tombstone、spawn sequence、destroy Commandを生成しない。再activationではAuthoring Entityだけをimmutable Section recordから再構築し、ephemeral runtime spawnは復元しない。`transition_persistent` Authoring Entityだけが§8.6.1 handoffで同じhandle／live stateを維持する。Game Systemは大量destroyでSection unloadを代用できず、Streaming workerもstructural command queueへ個別Entity commandを注入しない。

Section再activationはpending generation構築時に`RuntimePersistentTombstoneTable`を適用し、exact authoring identityのbase recordだけを抑止する。Section image／content hashは変更しない。tombstoneとpersistent handoff source／destinationが同identityを同時に要求する場合はpreflightを拒否し、tombstoneを無視またはhandoffへ暗黙変換しない。

#### 8.6.1 Persistent Entity handoff

Sectionを跨いで同じpersistent Entityを維持する場合は、暗黙のStable ID一致ではなく次をSection dependency payloadへ入れる。

```text
RuntimeEntityScopeRefV1
  kind: world_root_runtime | activation_group
  activation_group_id: optional uint32        # kind=activation_groupだけ必須、1..N

RuntimePersistentEntityHandoffV1
  handoff_id: registered StableId
  persistent_identity: PersistentEntityIdentityV1
  source_scope_ref: RuntimeEntityScopeRefV1
  destination_scope_ref: RuntimeEntityScopeRefV1
  expected_source_projection_hash: bytes32
  destination_projection_hash: bytes32
  expected_source_initializer_spec_ref: exact RuntimeEntityInitializerSpecV1 ref
  destination_initializer_spec_ref: exact RuntimeEntityInitializerSpecV1 ref
  destination_parameter_values
  destination_parameter_hash
  handle_policy: preserve_runtime_handle
  lifetime_owner_before_ref
  lifetime_owner_after_ref
  component_owner_approval_refs[]
  composition_cases[]
    source_archetype_key
    destination_archetype_key
    entity_enablement_disposition: preserve_live | destination_initial
    component_transfers[]
      component_type_ref
      source_present: bool
      destination_present: bool
      presence_disposition: preserve_live | initialize_destination
                          | remove_after_publish
      enablement_disposition: not_applicable | preserve_live | destination_initial
      field_transfers[]
        field_id
        transfer_kind: value | entity_relation
        transfer: closed union
          value
            disposition: preserve_live | destination_initial
                       | migrate | derived_rebuild
            migration_contract_ref: optional exact ref
          entity_relation
            target_requirement: required | optional
            disposition: preserve_target_identity | resolve_in_destination
                       | migrate | clear_optional
            migration_contract_ref: optional exact ref
      external_handle_transfer: optional
        disposition: preserve_live | prepared_rebuild | retire_after_publish
        reservation_contract_ref: optional exact ref
    composition_case_hash
  failure_policy: reject_new_generation_keep_old
  handoff_hash
```

C1のCooked handoffは`PersistentEntityIdentityV1.kind = authoring_entity`だけを対象とし、その`RuntimeEntityHandle`を維持して同じactivation publicationでSection ownership、lifetime owner、archetype／location、Identity Indexを更新する。source scopeにliveなidentityが厳密に一件あり、そのbase projection hashがexpected valueと一致し、destinationに同identityの通常Entity recordがないことをpreflightする。handoff recordに対応するdestination placeholderだけは重複ではなく、`destination_projection_hash`とexact Initializer Specで検証済みの移送先parameterとして扱い、それ以外の重複は`ECS_PERSISTENT_HANDOFF_INVALID`で拒否する。persistent runtime spawnはTemplateにより`world_root_runtime`へ置かれるためSection handoff対象にせず、runtime identityをCook時に予測または予約しない。

一つのactivation transactionで同じidentityをhandoffできるのは一回だけで、sourceはcurrent publicationでactive、destinationはpending publicationでactiveでなければならない。self handoff、A→B→C chain、複数destination、handoff cycleを拒否し、必要な最終destinationへ一件のdirect handoffをCookする。

Handoff compilerはsource Entityに到達可能な全archetypeを`composition_cases[]`へ厳密に一件ずつ列挙する。preflightはcurrent live archetypeでcaseを一件選び、0件／複数件を拒否する。各caseのsource／destination keyはArchetype Plan内に存在し、`component_transfers[]`は両compositionのunionをComponent Type canonical順で一度ずつ持つ。presence boolは各keyから導出した値と一致しなければならない。`preserve_live`は両方present、`initialize_destination`はdestination present、`remove_after_publish`はsource presentかつdestination absentだけを許可し、source／destinationともabsentのentryを作らない。`remove_after_publish`は`derived | presentation_only`かつSave／digest対象Fieldを持たないComponentだけに許可し、authoritativeまたは保存対象のpresence／stateをhandoffでdropしない。結果compositionを暗黙のadd／dropで補正しない。

Entity enablementは`RuntimeEntityProjectionV1.entity_enablement_save_policy = persistent`なら`preserve_live`、`reset_to_initial`なら`destination_initial`とする。Componentがdestinationにないか`structural_presence`なら`enablement_disposition = not_applicable`、destinationにある`enable_bit` Componentはsource absentなら`destination_initial`、source presentならComponent Specの`persistent -> preserve_live`、`reset_to_initial -> destination_initial`へexact一致させる。handoff callerまたはdestination Sectionがこのpolicyを上書きしない。

destinationに存在する`inline_value` Componentは全FieldをField ID順で一度ずつ`field_transfers[]`へ持つ。sourceに存在しないComponentは`destination_initial | derived_rebuild`だけを許可する。authoritative／Save対象Fieldはsource presentなら`preserve_live | migrate`、Section-local非authoritative Fieldだけが`destination_initial`、再生成可能なderived Fieldだけが`derived_rebuild`を選べる。`migrate`はexact migration Contractを必須とし、それ以外は持たない。entity relationはvalue transferと重複せず、required relationは`clear_optional`を選べず、relation migrationもexact Contractを必須とする。tagと`typed_external_handle`はField transferを持たない。

`typed_external_handle`は`external_handle_transfer`を厳密に一件持ち、他representationは持たない。`preserve_live`はsource／destinationともpresent、`prepared_rebuild`はdestination presentかつowner発行Reservation Contract必須、`retire_after_publish`はsource presentかつdestination absentだけを許可する。Component ownerごとにexact approval refを一件持ち、owner Contractはhandoff ID、全case hash、initializer／migration／reservation、failure policyを承認する。lifetime ownerが変わるhandoffはbefore／after両Game System Contractも同じhandoff hashを承認していなければCookしない。通常transitionはlive sourceを必須とし、Save resumeは保存Fieldから新session handleを作る別のreconstruction gatewayを使う。handoff preflight失敗時はsource Entityと旧publicationを変更しない。

Handoff stagingは全destination Entity／Identity Index slotを先に作り、その後にrelationをPersistent Identityからpending handleへ解決する。record順またはSection I/O完了順でrelationを解決せず、required targetがpending setに一件存在しなければpublishを拒否する。

#### 8.6.2 World Artifact envelopeとQualification binding

Root／Sectionは既存Asset IDを捏造せず、Asset Lifecycleのartifact subjectをclean breakで一般化して格納する。

```text
ArtifactSubjectRefV1
  kind: asset | runtime_world_root | runtime_world_section
  asset_id: optional StableId                 # kind=asset
  asset_revision: optional uint64             # kind=asset
  world_ref: optional StableId                # world kind
  project_revision: optional uint64           # world kind
  world_streaming_plan_ref: optional exact ref # world kind
  activation_group_id: optional uint32        # root=0、section=1..N

DerivedArtifactManifestV1
  artifact_key: sha256
  artifact_subject_ref: ArtifactSubjectRefV1
  artifact_role_id: ClosedArtifactRoleId
  target_profile_id: StableId
  schema_version: SemVer
  payload_hash: sha256
  payload_size: uint64
  alignment: positive_uint32
  dependency_keys: sha256[0..4096]
  qualification_binding_hash: optional bytes32
  producer_kind: asset_importer | runtime_world_compiler
  producer_contract_ref
  producer_version_hash: sha256
  toolchain_lock_hash: sha256
  capability_requirements: ClosedCapabilityId[0..256]
  runtime_budget: RuntimeAssetBudgetV1

RuntimeWorldQualificationBindingV1
  subject_kind: runtime_world_root | runtime_world_section
  artifact_subject_ref: ArtifactSubjectRefV1
  subject_image_hash: bytes32
  target_profile_ref: exact ref
  contract_set_hash: bytes32
  qualification_policy_hash: bytes32
  required_gate_set_hash: bytes32
  qualification_receipt_hashes[]: bytes32
  freshness_policy_ref: exact ref
  attestation_hash: bytes32
  binding_hash: bytes32

ArtifactCatalogSnapshotRefV1
  catalog_id: StableId
  catalog_generation: uint64, 1..2^64-1
  catalog_root_hash: bytes32

RuntimeWorldLaunchSetV1
  catalog_id: StableId
  catalog_generation: uint64, 1..2^64-1
  root_artifact_subject_ref: ArtifactSubjectRefV1
  root_artifact_ref: ArtifactRefV1
  bootstrap_sections[]
    section_artifact_subject_ref: ArtifactSubjectRefV1
    section_artifact_ref: ArtifactRefV1
  launch_set_hash
```

tagged subjectのvariant規則はclosedである。`asset`は`asset_id／asset_revision`だけを必須化して全world fieldを持たない。`runtime_world_root`は`world_ref／project_revision／world_streaming_plan_ref`と`activation_group_id = 0`だけを持つ。`runtime_world_section`は同じworld fieldとPlan内の`activation_group_id = 1..N`を持つ。不要field、0以外のRoot group、0のSection groupをrejectする。World variantの`qualification_binding_hash`は厳密に一件、Asset variantはAsset role policyに従う。

`DerivedArtifactManifestV1`は現行のsuffixなし`DerivedArtifactManifest`を置換し、旧reader、dual schema、synthetic Asset IDを残さない。Asset subjectだけが従来の`asset_id／asset_revision` variantを使う。World Rootのroleは`runtime_world_root`、Sectionは`runtime_world_section`である。Catalog keyとcanonical orderは`{artifact_subject_ref canonical bytes, artifact_role_id}`へ変更し、そのvalueに[Executable Contracts](../02-foundation/executable-contracts.md)所有のexact `ArtifactRefV1`を一件置く。World loaderはtyped subject＋roleを検索して`ArtifactRefV1`を得る。Domain固有のArtifact ref構造を再定義せず、`asset://` URIはAsset variantだけに使う。

Asset subjectは`producer_kind = asset_importer`、World subjectは`runtime_world_compiler`だけを許可し、World payload alignmentは16に固定する。`artifact_key`は同Fieldをall zeroにしたV1 manifestの`McdCanonicalBinaryV1` bytesとpayload bytesの連結へSHA-256を適用する。`binding_hash`も同Fieldをall zeroにしたQualification Bindingの`McdCanonicalBinaryV1` bytesへSHA-256を適用する。ManifestとBindingのartifact subject／Targetは完全一致しなければならない。Manifestの`qualification_binding_hash`はbinding hashを参照するため一方向であり、Binding内のReceipt／Attestationは`{artifact_subject_ref, subject_image_hash, target_profile_ref, contract_set_hash, qualification_policy_hash, required_gate_set_hash}`だけをsubjectにする。CompilerはArtifact／Binding／inner imageのhash dependency DAGを出力し、cycle、self edge、未解決nodeをCook errorにする。

Rootとbootstrap hard-dependency closureは`base` Content Groupへ置く。`WorldStreamingPlanV1.activation_groups[]`はexact `content_group_ref`を必須化し、非bootstrap Sectionはそのgroupへ置く。Sectionのhard dependencyは同じgroup、`base`、またはCatalogが先行mountを保証するgroupだけを参照できる。Package assemblyはこのgroup closureを検証し、依存Sectionを別groupの偶然のmount順へ委ねない。

outer Catalogは`{world_ref, project_revision, world_streaming_plan_ref, target_profile_id}`ごとにexact一件の`RuntimeWorldLaunchSetV1`を持ち、各subjectがCatalog entryから同じArtifact refへ解決することを検証してRootとbootstrap activation group＋transitive hard dependency Section closureを同じCatalog generationへ束縛する。Catalog ID／generationはassembly開始時に確定し、Launch Setとsigned Catalog envelopeへ同値を入れる。generationはCatalog内1から単調増加し、failed assemblyで消費せず、再利用／decrement／wrapしない。`catalog_root_hash`はCatalog内容から最後に計算してLaunch Set内へ逆参照させず、loaderが外側の`ArtifactCatalogSnapshotRefV1`として保持する。bootstrap entryはactivation group順・重複なしである。Root Manifestの`dependency_keys`はWorld Section keyを含めず、Root用Assetだけを参照する。Section ManifestはRoot Artifact keyを必須dependencyとし、hard dependency Section keyをPlan DAG順で参照できる。Rootからbootstrap Sectionへの逆dependencyを作らず、Catalog launch setを起動root集合にすることでArtifact key cycleを禁止する。

World loadは署名検証済みCatalog snapshot refとLaunch Set hashを最初の`RuntimeWorldPublicationRecordV1`へpinし、以後の全Section preflightは同じsnapshotからsubject／roleを解決する。新しいCatalog generation、renewed Receipt、同じinner imageを持つ別Artifact keyへlive Worldを自動rebindしない。C1 freshnessは`valid_at_accept`で、Root／bootstrap ReceiptはWorld load acceptance、追加Section Receiptは各preflight acceptanceで期限とrevocation snapshotを評価し、accept後のwall-clock経過だけでactive simulationを分岐または停止しない。preflight時点で不足すれば旧publicationを維持してtyped rejectionを返し、Catalog世代混在または未検証latest lookupへfallbackしない。緊急revocationによるsession stopはQualification Ownerの署名済みcontrol Eventをtick boundaryで記録して実行し、暗黙timerにしない。新Catalogの採用はWorld teardown後の新しいPlay／load transactionだけで行う。

Qualification Receiptはinner imageへ埋め込まない。inner Root／Sectionは`qualification_policy_hash`と`required_gate_set_hash`だけをhashし、outer Manifestが`RuntimeWorldQualificationBindingV1`を参照する。ReceiptとAttestationは上記subject tupleへ束縛し、artifact keyまたはbinding hashをsubjectにして循環させない。Receipt更新または再Qualificationはbinding／outer Manifest／Catalog generationだけを更新でき、同じinner image hashを維持する。Root binding更新でRoot Artifact keyが変わる場合は、全Section outer ManifestのRoot dependency key、そこから到達するSection Artifact key、Launch Set／Catalog generationを同じassembly transactionで再計算する。Section単独更新はそのSectionと、それをdependency keyに持つ下流SectionだけをPlan DAG順に再計算する。旧／新Artifact keyを一つのLaunch Setへ混在させない。Loaderはouter署名、subject／Target一致、binding、freshness、Receipt、inner image hashをすべて検証してからpreflightへ渡す。

#### 8.6.3 `RuntimeWorldImageBinaryV1`

RootとSectionのinner binaryは同じfixed directory形式を使う。Root magicはASCII `MKECSR1\0`、Section magicは`MKECSS1\0`、byte orderはlittle-endianである。Outer Asset envelopeの署名、compression、package placementはAsset Lifecycleが所有し、ECS loaderは展開・hash検証済みinner bytesだけを読む。

```text
RuntimeWorldImageHeaderV1
  magic: byte[8]
  schema_version: uint32 = 1
  header_size: uint32 = 96
  directory_entry_count: uint32
  reserved_zero: uint32
  total_size_bytes: uint64
  image_hash: bytes32
  root_image_hash: bytes32                 # Rootではall zero、Sectionではexact Root hash

RuntimeWorldImageDirectoryEntryV1
  payload_kind: uint32
  flags: uint32 = 0
  alignment: uint32 = 16
  reserved_zero: uint32
  offset_bytes: uint64
  size_bytes: uint64
  payload_hash: bytes32
```

HeaderとDirectoryはnative struct paddingを持たないfield順のpacked canonical bytesで、Headerは96 bytes、Directory entryは64 bytesである。integer fieldはすべてlittle-endian、`bytes32`は32 raw bytesである。Rootは9 entry、Sectionは6 entryを厳密に持ち、payload mappingを次へ閉じる。

| image | `payload_kind` | canonical payload |
|---|---:|---|
| Root | 1 | `RuntimeWorldRootManifestPayloadV1`。Rootのscalar／ref／subpayload hash。`root_image_hash`とsubpayload本文を含めない |
| Root | 2 | `RuntimeComponentRegistryV1` |
| Root | 3 | `RuntimeEntityConstructionSetV1`。Initializer Spec、Persistent Identity Policy、Templateを各Stable ID canonical byte順に保持 |
| Root | 4 | `RuntimeArchetypePlanV1` |
| Root | 5 | `RuntimeQueryRegistryV1` |
| Root | 6 | `RuntimeSystemBindingAndStateStoreSetV1`。System ref順のManifest binding、State Store spec、Identity Sequence Slot spec |
| Root | 7 | `RuntimeWorldSectionCatalogV1`。`RuntimeWorldSectionCatalogEntryV1[]`をactivation group順に保持し、Root非依存`section_content_hash`をcommitする |
| Root | 8 | `RuntimeExactAssetDependencySetV1`。Artifact subject／role／key順 |
| Root | 9 | `RuntimeWorldQualificationPolicyPayloadV1`。policy ref／hashとrequired Gate ID集合。Receiptを含めない |
| Section | 16 | `RuntimeWorldSectionManifestPayloadV1`。Sectionのscalar／ref／subpayload hash。`section_image_hash`、`root_image_hash`、subpayload本文を含めない |
| Section | 17 | `RuntimeWorldSectionEntityRecordSetV1` |
| Section | 18 | `RuntimeWorldSectionIdentityTableV1` |
| Section | 19 | `RuntimeWorldSectionDependencyAndHandoffSetV1`。Section dependencyと`RuntimePersistentEntityHandoffV1[]` |
| Section | 20 | `RuntimeWorldCapacityRecordV1` |
| Section | 21 | `RuntimeWorldQualificationPolicyPayloadV1`。Receiptを含めない |

各payload objectはMCD canonical binary encodingを使う。Root manifestは`schema_version | project_revision | source_document_root_hash | contract_set_hash | target_profile_ref | runtime_ecs_profile_hash | component_registry_hash | entity_construction_set_hash | archetype_plan_hash | system_implementation_set_ref | state_owner_table_hash | world_streaming_plan_ref | section_catalog_hash | bootstrap_activation_group_ref | exact_asset_dependency_set_hash | save_replay_contract_set_hash | qualification_policy_hash | required_gate_set_hash`をこのMCD Field ID順で持つ。Section manifestは`schema_version | section_id | world_streaming_plan_ref | activation_group_id | ordered_cell_ref_set_hash | source_ref_merkle_root | hard_dependency_set_hash | entity_record_set_hash | source_identity_table_hash | persistent_handoff_table_hash | required_capacity_hash | exact_asset_dependency_set_hash | qualification_policy_hash | required_gate_set_hash`を持つ。配列本文は対応する専用payloadだけに一度置く。

DirectoryはHeader直後のbyte 96から始め、`payload_kind`昇順で重複不可である。C1は全payload alignmentを16に固定し、各offsetは`align_up(previous_end, 16)`と厳密に一致して任意gapを許可しない。offset／size／end計算はchecked `uint64`、rangeは重複せずHeader／Directoryより後、canonical alignment paddingはzeroとする。magic／schema version／header size／entry count／alignment不一致、`total_size_bytes`と最終range終端不一致、unknown／欠落payload kind、unknown flag、nonzero reserved、hash不一致、trailing byte、integer overflowを拒否する。`payload_hash`はpayload bytes、`image_hash`は同FieldだけをzeroにしたHeader、Directory、全padding、payloadの連結へSHA-256を適用する。

Root／Sectionの相互hash循環を禁止するため、Section cookerは最初にHeaderの`image_hash`と`root_image_hash`を両方all zeroにした同じcanonical bytesへSHA-256を適用し、`section_content_hash`を得る。Rootの`RuntimeWorldSectionCatalogV1`はこのcontent hashへcommitしてRoot `image_hash`を確定する。その後だけSection Headerへexact Root hashを入れ、`image_hash`だけをzeroにした通常式で最終`section_image_hash`を計算する。Root inner bytesは最終Section image hash、Section Artifact key、outer bindingを一切含めない。Outer CatalogはRootとSectionの最終Artifact keyを同一generationで束縛する。LoaderはSectionのcontent hashを再計算してRoot catalog entryと比較し、次にHeaderのRoot hashと最終image hashを比較する。

```text
Section payloads + zero root/image fields -> section_content_hash
all section_content_hash values + Root payloads -> root_image_hash
Section payloads + root_image_hash -> section_image_hash
image hash + subject/Target/Policy + Receipts -> qualification_binding_hash
inner payload + binding ref + dependency keys -> artifact_key
Root/Section artifact refs -> Catalog generation + RuntimeWorldLaunchSetV1
```

矢印を逆向きに持たせない。特にRoot inner imageから最終Section image／Artifact key、Receiptからinner image、Artifact manifest／payloadからCatalog root hashへのedgeを禁止する。

高水準`RuntimeWorldRootImageV1.root_image_hash`はRoot Headerの`image_hash`、`RuntimeWorldSectionImageV1.section_image_hash`はSection Headerの`image_hash`を表す投影名であり、payloadへ重複encodeしない。Root Headerの`root_image_hash`はall zero、Section Headerの同Fieldはexact Root Header `image_hash`である。outer Manifest、Qualification Binding、Receiptはこのinner hash計算へ含めない。

Entity recordはPersistent Entity Identityのcanonical byte順、ComponentはType ref順、FieldはMCD Field ID順で格納する。値はMCD canonical binary encodingを使い、native Component bytes、padding、pointer、endianness依存値を含めない。capacity recordは`entity_count | chunk_count | byte_count | command_count`のunit IDと`uint64`値を必須にし、無単位整数を拒否する。Layout Fact計算もchecked `uint64`で行い、最終offset／row capacityを`uint32`へ収める。

#### 8.6.4 `McdCanonicalBinaryV1`

E0は[Executable Contracts](../02-foundation/executable-contracts.md)へ全`runtime_cook = true`型共通の`McdCanonicalBinaryV1`を正本化し、Root／Section payloadはそのgenerated encoder／decoderだけを使う。wire規則は次へ閉じる。

- byte orderはlittle-endian、alignment paddingはなく、長さ／件数prefixは`uint32`である。resolved boundが`UINT32_MAX`を超える型は`runtime_cook = true`にできない。全offset／累積length検査はchecked `uint64`で行う。
- `bool`は`uint8`の0／1、signed／unsigned integerは宣言bit幅、floatは宣言したIEEE 754幅のbit patternとする。floatはencode前に`-0`を`+0`へ正規化し、NaN／Infinityを拒否する。
- fixed bytesは宣言長のraw bytes、可変bytes／UTF-8 string／decimal string／string-backed closed enumは`uint32 byte_length`＋bytesとする。UTF-8、pattern、NFC／decimal canonicalization等のMCD constraintをencode前とdecode後に検査する。
- structはField ID昇順である。先頭に全Field分のpresence bitsetをField ID順、各byte内least-significant bitから置き、tail bitを0にする。required Fieldのbitは1、optional absentは0とし、present Fieldの値だけを同順で直列化する。duplicate／unknown Fieldを受ける拡張領域は作らない。
- arrayはcount＋要素、setはcount＋各要素のcanonical binary bytesをunsigned lexicographic昇順に並べ、重複を拒否する。mapはcount＋key／valueで、keyをMCD canonical key bytes順に並べ重複を拒否する。
- tagged unionはclosed discriminatorのcanonical string encodingに続けて選択variant payloadを一件置く。unknown discriminator、variantとpayload型の不一致、不要variant Fieldを拒否する。nullableは0／1の`uint8`に続き、1の場合だけ値を置く。
- decoderはexact descriptor、nesting depth、collection bound、payload sizeを先に持ち、underflow、overflow、non-canonical order、nonzero tail bit、constraint違反、trailing byteを拒否して部分objectを返さない。decode後の再encode bytesが入力と一致しない値をnon-canonicalとして拒否する。

`payload_kind`からtop-level MCD Typeは一意に決まり、Root manifestをbuilt-in schema version 1で先にdecodeしてexact Contract setを選び、そのContract setのgenerated descriptorで残りpayloadをdecodeする。SectionはHeaderのRoot hashから既にvalidatedなRoot Contract setを選び、Sectionだけで別schemaを選べない。JSON／Provider projectionのJCS規則とbinary bytesを混同せず、canonical value同値をcross-language fixtureで検証する。

### 8.7 `RuntimeWorldStateV1`

```text
staging -> live
staging -> destroying -> destroyed
live -> destroying -> destroyed
live -> faulted -> destroying -> destroyed
```

他の遷移を許可しない。recoverableなSection preflight rejectionは`live`のままである。unrecoverable ECS／System faultはWorldを`faulted`へ一方向遷移させ、Runtime OrchestratorがEditor processを[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)の`Playing -> PlayStopping`へ遷移させる。Worldの`faulted`とEditor processの`Faulted`は別状態であり、後者はHost／Editor自体がsafe stopできないfaultだけに使う。

## 9. Query、access、System実行

### 9.1 `RuntimeEntityQuerySpecV1`

```text
RuntimeEntityQuerySpecV1
  query_id: registered StableId
  terms[]: RuntimeEntityQueryTermV1
  entity_enablement: require_enabled | require_disabled | ignore
  ordering: canonical_archetype_chunk_row
  partition_policy: single | fixed_range | deterministic_hash
  partition_bound_ref: optional exact ref
  result_bound_ref
  query_spec_hash

RuntimeEntityQueryTermV1
  component_type_ref
  match_kind: all | any | none | optional
  access: presence | read | read_write
  enablement: require_enabled | require_disabled | ignore
```

`all`はすべて、`any`は全any term中一つ以上、`none`は除外、`optional`はfilterせず存在時だけ列を返す。System queryは`all`または`any` termを最低一つ必要とし、全Entity scanを暗黙許可しない。同じTypeの複数term、ManifestにないType、`none`／`presence`以外の矛盾accessをCompile errorにする。

Compilerはtermを`component_type_ref`のcanonical byte順へ正規化し、Source記述順をquery identityまたはABI列順に使わない。`query_spec_hash`はquery ID、正規化済みterm、Entity enablement、ordering、partition policy、resolved partition bound、resolved result boundを`McdCanonicalBinaryV1`でhashする。Batch columnは正規化termのうち`all | any | optional`だけを同順で一件ずつ持ち、`none`はfilterにだけ使ってcolumn descriptorを生成しない。

`partition_policy`は次のclosed setであり、partitionを実行するのはSchedulerだけで、Systemまたはbindingがoverrideしない。

| Policy | `partition_bound_ref` | canonical work assignment |
|---|---|---|
| `single` | absent | query全体をcanonical順の一つのlogical workとして実行 |
| `fixed_range` | required | boundが固定する最大row数で、canonical archetype／chunk順にhalf-open row rangeへ分割 |
| `deterministic_hash` | required | boundが固定するpartition countへ、`SHA-256(McdCanonicalBinaryV1({query_id, archetype_id, chunk_id})) mod partition_count`でchunk単位に割当て |

`fixed_range`の各rangeと`deterministic_hash`の各partition内は`archetype_id, chunk_id, row`順を維持する。boundは1以上のclosed upper boundを解決し、欠落、0、上限超過、policy不一致をContract compile errorにする。worker数、worker index、completion順、address、runtime hash-map順をpartition count、割当て、結果順へ使わない。

`all`／`any`／`none`はstructural presenceを判定する。`enable_bit`を持つ`all | any` termは`require_enabled`を既定とし、明示時は`require_enabled | require_disabled | ignore`を選ぶ。structural-only Componentは`ignore`だけを許可する。`none | optional`は`enablement = ignore`だけを許可し、`none`はenable状態にかかわらずComponentが存在すれば除外する。optional enable状態はColumnに付随するread-only enable maskで読む。

一つのEntityが複数`any` termに一致しても結果は一度だけである。`any`または`optional`の非存在列はbatch単位の`OptionalColumnRead`／`OptionalColumnWrite`として`present=false`を返し、empty raw pointerまたはdefault Componentを捏造しない。同じEntityをmatching termごとに重複返却しない。archetype単位で一致した後、Entity／Component enable bitでrowをskipしても残るrowのcanonical順を変えない。

`entity_enablement`は省略時`require_enabled`である。Authoring `enabled=false`またはruntime `set_entity_enabled(false)`のEntityは、`require_disabled`または`ignore`を明示したlifecycle／debug query以外へ現れない。

一つの`QueryBatch`は連続するphysical row rangeと`RuntimeQuerySelectionMaskV1`を持つ。Maskのbit `i`は`row_begin + i`に対応し、64-bit word内least-significant bitから昇順、range外tail bitは0である。`require_enabled`はComponent enable mask、`require_disabled`はrange内bitを反転したmask、`ignore`は全1を返す。存在しない`any` termのmaskは全0である。

最終selectionは`EntityEnableMask AND (全all term maskのAND) AND (any termがなければ全1、あれば全any term maskのOR)`である。`entity_enablement`も同じenabled／disabled／ignore変換を使う。各`any` columnはterm match mask、各enableable `any | optional` columnは実enable maskをread-only metadataとして公開し、Systemが「どのtermで一致したか」と「Component自体がenabledか」を区別できる。Systemは最終selectionのset bitだけをrow昇順で処理し、mask false rowをread／writeしない。

Hostは`row_count >= 1`かつ`selected_count >= 1`のBatchだけをinvokeし、all-zero maskをskipする。skipは結果順を変えず、logical work IDを別workへ再利用しない。

generated `SelectedRowsView`はmask set rowだけを返し、maskが全1の場合だけ`DenseSelectedRowsView`へのchecked変換でcontiguous spanを公開する。sparse batchからraw dense spanを取得できない。Mask生成scratchはQuery Profileでpreallocateし、iteration中heap allocationを行わない。partitionはphysical range、canonical output順はlogical work ID内のselected row昇順である。

access modeは`presence | read | read_write`のclosed setである。`tag`は`presence`だけ、`typed_external_handle`は`presence | read`だけ、`inline_value`は三modeを許可する。C1では未初期化または全row上書きを検証できない`write_only`を提供しない。`optional`の列はbatch単位の`OptionalColumnRead`または`OptionalColumnWrite`であり、raw nullable pointerを返さない。

Query cacheはWorld load時に`RuntimeArchetypePlanV1`全体へmatchし、execution中にarchetype discoveryまたはheap allocationを行わない。結果はcanonical archetype／chunk／row順で、filter predicateを任意functionまたはstring DSLとして埋め込まない。Component値による絞り込みはSystem codeがtyped column上で行う。

`RuntimeQueryRegistryV1`はactive queryをquery Stable IDのcanonical byte順に並べ、`query_runtime_id: uint32`を1から割り当て、0をinvalidとする。entryはQuery Spec hash、normalized term hash、matching archetype ID集合、Manifest refs、result boundを持つ。`query_runtime_id`はpackage／Play session内だけで、Source／Save／AI identityにはquery Stable IDを使う。

### 9.2 `RuntimeComponentAccessManifestV1`

```text
RuntimeComponentAccessManifestV1
  system_contract_ref
  implementation_variant_ref
  allowed_phase_ids[]
  query_bindings[]
  read_component_type_refs[]
  write_component_type_refs[]
  enablement_write_type_refs[]
  structural_permissions[]
  state_store_accesses[]
  external_store_access_refs[]
  entity_construction_set_hash
  state_owner_table_hash
  manifest_hash
```

Manifestは`GameSystemSpecV2`、State owner、query、structural operation、Implementation VariantからContract compilerが生成し、System codeが拡張しない。write／enablement writeはComponent owner Systemだけに許可するが、`tag`と`typed_external_handle`へvalue `read_write`を生成しない。他Systemはtyped Commandをownerへ送り、同じComponentの共同writerを作らない。

各`query_bindings[]`はquery ref、callback entry ID、許可phase ID集合、Query Specの`partition_policy`と`partition_bound_ref`へのexact refを持つ。bindingはpolicyまたはboundを上書きできず、`single`だけが一つのcallback work、他二policyはScheduler生成partitionを消費する。ECSを使わないSystemも空のManifestを厳密に一つ持ち、query、Component access、structural permissionを0件とする。空Manifestを理由にunchecked World pointerを渡さない。

typed external handleの列readだけではexternal object accessを与えない。resolver／store viewは一致する`external_store_access_refs[]`、participant generation、phase、modeを持つgenerated bindingだけが取得でき、handle generationとcurrent publicationを毎回検査する。store viewもcallback外へ保存できず、Component accessからVendor pointerへ暗黙変換しない。

Native descriptor、generated C++ template、scheduler dependency graph、Runtime validator、AI Contract Graphは同じ`manifest_hash`を使う。登録時のC++ callback signature、query binding、phase、Component layoutがManifestと一致しなければModule全体をloadしない。

同じphaseで二つのSystemのComponentまたはState Store accessが交差し、一方がwriteの場合は同時実行しない。既存の`GameSystemDependencyEdgeV1`が明示順を決めるか、同じowner System内のgenerated callback orderで逐次化する。どちらもない競合をSystem ID順へ暗黙整列せず、`ECS_ACCESS_ORDER_AMBIGUOUS`としてContract compileを拒否する。read／readだけは並列化できる。

### 9.3 Leaseとparallel partition

Queryはmove-onlyな`ReadLease<T...>`、`WriteLease<T...>`またはgenerated `QueryBatch`を返す。LeaseはWorld ID、World epoch、phase ID、logical work ID、有効row rangeを持つ。copy、heap保存、member保存、event／command／job packetへのcapture、C ABI外へのpointer返却を禁止する。

```text
RuntimeLogicalWorkIdV1
  work_kind_code: uint32
  system_runtime_id: uint32
  binding_runtime_id: uint32
  archetype_id: uint32
  chunk_id: uint32
  work_ordinal: uint32
```

`work_kind_code`は`0 invalid | 1 query_partition | 2 system_callback | 3 targeted_batch`である。`system_runtime_id／binding_runtime_id／work_ordinal`は常に1以上とする。query partitionはbindingにquery runtime IDを置き、archetype／chunk／row-range ordinalをすべて1以上にする。system callbackはgenerated callback entry runtime ID、archetype／chunkを0、ordinalを1とする。targeted batchはCommand runtime IDと対象partitionのarchetype／chunk／ordinalを1以上にする。不要Fieldへ任意値を入れない。

Rootの`RuntimeSystemBindingAndStateStoreSetV1`はactive System Contract refのcanonical byte順に`system_runtime_id`を1から、`{System ref, callback entry Stable ID}`順にcallback entry runtime IDを1から割り当てる。Executable Contractsはactive Command Type ref順にCommand runtime IDを1から割り当て、§9.1はquery runtime IDを割り当てる。各ID空間は`work_kind_code`で区別し、0 invalid、Root内immutable、Source／Save／AI identityへ保存しない。Module登録順、link順、dispatch順から採番しない。

```text
RuntimeSystemExecutionContextV1
  world_instance_handle: RuntimeWorldInstanceHandle
  base_publication_handle: RuntimeWorldPublicationHandle
  working_tick: uint64
  phase_id: uint32
  partition: RuntimeExecutionPartitionV1
  read_snapshot_ref: callback-scoped exact ref
  write_batch_ref: callback-scoped exact ref
  diagnostic_sink_ref: callback-scoped exact ref
  logical_work_id: RuntimeLogicalWorkIdV1
  bound_activation_group_id: optional uint32, 1..N
  context_token: opaque uint64

RuntimeExecutionPartitionV1
  policy: single | fixed_range | deterministic_hash
  partition_ordinal: uint32, 1..partition_count
  partition_count: uint32, >= 1
  archetype_id: optional uint32
  chunk_id: optional uint32
  row_begin: optional uint32
  row_end: optional uint32
```

`RuntimeSystemExecutionContextV1`のschemaはECS正本が所有し、値はSchedulerがcallback invokeごとに束縛して渡す。`partition`はQuery Specのpolicyと一致し、`single`ではcount／ordinalとも1でrange Fieldを持たず、`fixed_range`ではexact chunkと`row_begin < row_end`、`deterministic_hash`ではpartition ordinal／countを必須としてrange Fieldを持たない。`read_snapshot_ref`はbase publication、tick、phaseへ固定したread-only Query／State viewを解決する。`write_batch_ref`はそのlogical work専用のprivate Command／Event／Structural outputを所有し、callback成功後にだけcanonical mergeへ渡す。`diagnostic_sink_ref`はProfileで件数／bytesをbounded化し、World mutationまたはlog-only failureへ変換しない。

System callbackはこのcontextからだけQueryBatch、Lease、`EntityReadPort`、`StructuralCommandWriter`、preparation portを取得し、§8.2のbare `RuntimeEntityHandle`許可境界はこのcontextのWorld束縛で判定する。contextと三つのrefはmove-only・callback-scopedで、copy、heap保存、callbackを越える持ち出しを禁止する。`bound_activation_group_id`は`producer_activation_group` scopeのcreateを発行できるexact一件のactive group(§8.3)を示し、束縛のないcontextはこのFieldを持たない。`context_token`はlease tokenと同じepoch／phase失効検査へ入力するsession-local integrityで、Source、Save、Replayへ保存しない。

Schedulerだけがchunkまたはhalf-open row rangeへpartitionする。write exclusion keyは`{component_type_ref, chunk_id, row_begin, row_end}`で、write／writeおよびwrite／readの交差を同時実行しない。`logical_work_id`は上記variantのfixed-width tupleから生成し、dispatch順に採番しない。並列producerはworker indexではなくこのIDを受け、private output bufferへ書く。

Leaseはstructural commit、phase終了、tick終了、World fault、World破棄で失効する。generated view取得、indexed accessor、Host callbackは毎回lease token／epochを検査し、失効時は`ECS_LEASE_EXPIRED`としてtickをpublishしない。Developmentはretired range poison、guard、sanitizerでescaped borrowをbest-effort検出する。

C++で一度得たraw reference／pointerの全dereferenceをHostがinterceptすることはできない。Project codeがaddressを保存してgenerated accessorを迂回する行為はNative Contract違反であり、source／AST／binary conformance、sanitizer fixture、reviewでModule promotionを拒否する。違反ModuleのShipping UBを「必ずtrapしてrecoverできる」とは保証しない。Native C++はmemory-safe sandboxではなくR3 trusted codeである。

Handle指定の少数Entityを読む場合はManifest-declared `EntityReadPort<T...>`を使い、callback内だけ有効なread leaseを得る。Component値を変更するtargeted CommandはSchedulerがhandle／generationを検証し、owner System向けに`archetype_id, chunk_id, row, command_order`で整列した`TargetedCommandBatch`へ変換する。owner Systemはこのbatchのgenerated write leaseからだけ対象Stateを更新する。任意のunchecked `get<T>(handle)`／`set<T>(handle)`、他System向けrandom write APIは提供しない。

### 9.4 Value writeとtick publish

`WriteLease`はworking tickのlive Component列を直接更新し、同じtickの後続phaseは確定したphase順でその値を読める。一般的なpage copyまたはtick全体のrollbackは行わない。SystemはLease取得後にallocation、exception、外部I/O、未検証入力処理を行わず、失敗し得る処理をprivate output生成時に完了させてからbounded commit callbackへ入る。

それでもSystem fault、non-finite value、invariant違反が発生した場合、そのtickのCommand／Event／Presentation／Save checkpointをpublishせず、`RuntimeWorldStateV1`を`live -> faulted`へ移す。Orchestratorはprocessを`Playing -> PlayStopping`へ進めてWorldを破棄する。部分更新したworking tickをretryまたは継続しない。直前にseal済みのPresentation SnapshotとSave checkpointだけが外部可視の最終成功状態である。Structural transactionのatomicityはstorage整合性を保証し、任意System codeのrollbackを意味しない。

### 9.5 System-owned State Store

World／extension-owned scope等のglobal Stateはsingleton Entityにせず、`GameSystemSpecV2.owned_state_type_refs`から生成する次のRuntime storageへ置く。Stateの意味とSave fieldはGameplay Programming Model、Runtime instance storageとaccessはECS正本が所有する。

```text
RuntimeSystemStateStoreSpecV1
  schema_version
  state_store_spec_id
  owner_system_ref
  runtime_instance_scope
  state_type_refs[]
  state_layout_fact_refs[]
  initialization_contract_ref
  save_replay_contract_ref
  authoritative_digest_contract_ref
  target_profile_refs[]
  capacity_ref
  store_spec_hash

RuntimeSystemStateStoreAccessV1
  state_store_spec_ref
  state_type_refs[]
  access: read | read_write
  allowed_phase_ids[]
```

一つのstore instance keyは`{state_store_spec_id, scope_instance_ref}`である。`scope_instance_ref`はplay session、World、Level、Encounter、Entity、UI sessionのgeneration-bearing typed refで、表示名または配列indexを使わない。`authoritative_digest_contract_ref`は各authoritative State TypeのField projectionと、Runtime handleを使わないscope identity projectionを固定する。persistent scopeはSave-stable instance ID、session-only scopeはRoot、registered scope kind、canonical activation／creation ordinalから導出し、generation handle値をdigest identityにしない。owner Systemだけが`read_write`、他Systemは`read`またはownerへのtyped Commandを持つ。

generated `StateReadLease<T...>`／`StateWriteLease<T...>`はComponent leaseと同じphase／tick lifetimeを持つが、chunk／archetypeへ格納しない。scheduler exclusion keyは`{state_store_instance_key, state_type_ref}`である。State TypeはMCDのbounded layout、initialization、memory capacityを持ち、可変長dataはpreallocated State domainまたはtyped external storeをexact refで使う。

System instance activationは全Stateをstagingでinitialize／load／migrateしてからpublishし、部分Stateをliveにしない。owner writeのfault、Save checkpoint、Replay、World teardownは§9.4と`SaveReplayContractV1`に従う。Native ABIはgenerated State viewを明示し、既存`output_writer`へ同じStateの第二write経路を残さない。

## 10. Structural transaction

### 10.1 `StructuralCommandBatchV1`

許可operationは次の六つだけである。

```text
RuntimeStructuralProducerKeyV1
  apply_boundary_id: uint32
  producer_phase_id: uint32
  producer_system_id: uint32
  logical_work_id: RuntimeLogicalWorkIdV1
  local_sequence: uint32, 1..N

EntityCreateToken
  producer_key: RuntimeStructuralProducerKeyV1

RuntimeCreateParameterEntityRefV1
  kind: live_entity | earlier_create
  live_entity_handle: optional RuntimeEntityHandle
  earlier_create_token: optional EntityCreateToken

StructuralCommandBatchV1
  world_instance_handle: RuntimeWorldInstanceHandle
  base_publication_handle: RuntimeWorldPublicationHandle
  working_tick: uint64
  commands[]: StructuralCommandV1
  batch_hash

StructuralCommandV1
  producer_key: RuntimeStructuralProducerKeyV1
  kind: create_entity | destroy_entity | add_component | remove_component
      | set_component_enabled | set_entity_enabled
  payload: closed union
    create_entity
      template_ref: exact RuntimeEntityTemplateV1 ref
      parameter_values
      parameter_hash
      persistence_disposition: optional ephemeral | persistent
      spawn_preparation_ticket_hash: optional bytes32
    destroy_entity
      target: RuntimeEntityHandle
    add_component
      target: RuntimeEntityHandle
      component_type_ref
      initializer_parameter_values
      initializer_parameter_hash
      external_reservation_handle: optional RuntimeExternalReservationHandle
    remove_component
      target: RuntimeEntityHandle
      component_type_ref
    set_component_enabled
      target: RuntimeEntityHandle
      component_type_ref
      enabled: bool
    set_entity_enabled
      target: RuntimeEntityHandle
      enabled: bool
```

Command unionはkindと同名payloadを厳密に一件だけ持つ。不要variant Field、unknown kind、base publication／working tick不一致を拒否する。`persistence_disposition`はTemplate policyが`optional`の場合だけ必須で、ticketがある場合は同じ値／parameter hash／boundaryでなければならない。`add_component.initializer_parameter_values`はComponent Specが固定するowner initializer parameter schemaだけを使い、任意Field mapではない。Commit validatorはtransaction内の直前virtual source archetype、operation、Component TypeからArchetype Planのedgeを厳密に一件選び、そのtransition hash、destination、Access Manifest binding、Component Specのinitializer bindingを照合する。0件または複数件なら拒否し、callerにtransition IDを選ばせない。`RuntimeCreateParameterEntityRefV1`もtagged unionで、liveはhandleだけ、earlier createは同じWorld／batch／logical work／boundaryかつ小さいlocal sequenceのtokenだけを持つ。

| operation | precondition | effect |
|---|---|---|
| `create_entity` | Template許可済み、typed parameter valid、issuerがInitializer Specのlifetime owner、identity policy valid、必要なexternal reservationがowner発行かつ有効 | Templateが束縛するInitializer Specで新Entityを作成し、identityはEngineがpolicyから発行 |
| `destroy_entity` | handleがlive、issuerがEntity lifetime owner、同boundaryで他operation対象でない、destruction save policy valid | Entityと全Componentを除去し、必要なAuthoring tombstoneを同publicationで作成 |
| `add_component` | live Entity、issuerがComponent owner、Component未存在、transition／initializer宣言済み、必要なexternal reservationがowner発行かつ有効 | owner-approved初期値でdestination archetypeへmove |
| `remove_component` | live Entity、issuerがComponent owner、Component存在、transition宣言済み | destination archetypeへmove |
| `set_component_enabled` | enable bit Componentが存在し、issuerがComponent owner | 次boundaryからquery predicateへ反映 |
| `set_entity_enabled` | live Entity、issuerがEntity lifetime owner | 次boundaryからEntity enable predicateへ反映 |

通常のComponent値変更は`WriteLease`だけで行い、汎用`set_component` commandを提供しない。別SystemのStateを変更する場合はGameplay Ownerのtyped Commandを使う。

各parallel work itemはprivate batchを持ち、各commandへ`RuntimeStructuralProducerKeyV1`を付ける。`producer_system_id`は`logical_work_id.system_runtime_id`と一致し、`local_sequence`は同一logical work内で1からgapなし、command Budget以下とする。`apply_boundary_id`はworking tick内で1から始まりScheduling仕様がproducer phaseごとに生成し、Systemが任意値を選ばない。SchedulerはField順のunsigned fixed-width lexicographic順でmergeし、各boundaryを別transactionとしてcommitする。commit前のqueryは旧composition、commit後に開始するphaseのqueryは新compositionを見る。worker index、thread ID、completion時刻、pointer、hash iteration順を使わない。

`create_entity`はprivate batch内で一意な`EntityCreateToken`を返せる。後続の同一batch create parameterは、より小さい`local_sequence`で生成済みのtokenだけをtyped Entity referenceとして使える。tokenをdestroy／add／remove／Component enable／Entity enableのtargetにせず、Templateのinitial archetypeをcreate後に組み替えない。forward reference、別batch、別work item、別System、次boundaryへのtoken保存を拒否する。merge後にcanonical allocation順で`RuntimeEntityHandle`へ一度だけ解決し、結果はcommit後のtyped Eventで必要なconsumerへ通知する。

merge後、target Entityごとにlive composition／enable stateからvirtual stateを作り、global canonical command順にpreconditionとtransitionを適用する。異なるComponentへのadd／removeはこのvirtual state上でchainでき、各commandのsource／destination archetypeを一段ずつArchetype Planへexact照合する。実storageはstagingで構築し、途中compositionをconsumerへ公開しない。同じboundaryで`destroy_entity`と他operationが同一Entityを対象にする場合、同一`{Entity, Component}`へ複数structural operationがある場合、または同一Entityへ`set_entity_enabled`が複数ある場合は競合である。last-write-wins、暗黙deduplicate、存在時addをreplaceとして扱う補正は行わない。

### 10.2 Commit algorithm

1. 全batchをcanonical orderへmergeする。
2. 全handle、generation、lifetime／Component owner、Manifest、Template／initializer、identity policy、external reservationをlive World snapshotに対して検証し、Entityごとのvirtual stateをcanonical command順に進めて全transition／enable preconditionを検証する。
3. destination chunk、location table、identity index、identity sequence slot、persistent tombstone table、enable bit、diagnostic bufferに必要な全capacityを予約する。external storeは必要objectをprepared／unresolvable stateへ構築する。
4. live tableを変更せずstaging mutation planを構築し、全move、swap-back後のlocation、identity発行／tombstone、external handle値を計算する。
5. Schedulerのquiescent boundaryで全prepared stateをpending publication generationへhidden attachする。live callback、I/O、allocationを行わず、全attachが成功した後だけECS tables、Identity Index、sequence／tombstone table、participant generationを参照するpending RecordのHandleをpublication slotへ一回release storeし、World epochを一度増やす。
6. old chunkをretireし、全旧Leaseを失効させ、canonical `RuntimeComponentLifecycleDeltaV1`をsealする。削除したexternal objectはconsumer Lease／fence完了までowner storeが保持する。

一件でも失敗した場合、boundary全体をcommitせずstagingだけを破棄する。ECSのstructural state、Entity allocator、generation、identity／sequence／tombstone table、query visible compositionは変更しない。該当tickはpublishせず、Scheduling OwnerのRuntime fault transitionへ進む。同じworking tickで既に行われたComponent value writeは外部公開せず、World破棄で捨てる。一般heap fallback、部分成功、次tick retry、invalid commandだけのskipは行わない。

このsession fault規則は、active Game Systemが発行したManifest／precondition違反、または予約済みcommitの内部不変条件違反に適用する。World streamingは§8.6の専用preflightで通常のI/O、capacity、cancel、stale generationをcommit提出前にrecoverable rejectionへ分離する。Section preflight rejectionでtickまたはWorldをfaultさせない。

World load中の初期構築はScheduler停止下の`RuntimeWorldBuildGateway`だけが即時APIを使える。Game System、Native module、AI、Editor、Subsystem Adapterへ即時structural APIを公開しない。

`RuntimeComponentLifecycleDeltaV1`はWorld ID、apply boundary ID、commit後World epochと、canonical command順のentryを持つ。各entryはoperation、commit sequence、Runtime Entity handle、lifetime owner System ref、optional Persistent Entity Identity、optional Component Type ref、必要なold／new typed external handle copied valueを持つ。raw Component bytesまたはaddressを含めない。

Component constructor／destructor callback、observer、Subsystem callbackをchunk move中に呼ばない。Deltaはowner Systemへ次の宣言済みintegration phaseで配送するが、新しいexternal objectを事後作成する指示には使わない。create／addに必要なobjectはpublish前のReservationで既にpreparedされる。remove／destroyのDeltaはretirement開始通知であり、Domain-owned objectは宣言済みconsumer Lease／fenceが完了してからreleaseする。World停止時はSchedulingのDomain teardownを完了してからECS Worldを破棄する。

### 10.3 `typed_external_handle` transaction

`typed_external_handle`はDomain-owned storeのobjectを指す非所有・fixed-width・generation handleである。Component Typeによりexternal storeとresolverを一意に決め、raw OS／GPU／vendor handleまたは任意store IDをComponentへ格納しない。

```text
RuntimeExternalPreparationRequestKeyV1
  world_instance_handle: RuntimeWorldInstanceHandle
  request_working_tick: uint64
  request_phase_id: uint32
  requester_system_id: uint32
  logical_work_id: RuntimeLogicalWorkIdV1
  request_sequence: uint32, 1..N
  target_working_tick: uint64
  target_apply_boundary_id: uint32

RuntimeExternalReservationHandle
  index: uint32
  generation: uint32

RuntimeExternalHandleReservationV1
  reservation_handle: RuntimeExternalReservationHandle
  preparation_request_key: RuntimeExternalPreparationRequestKeyV1
  consumer_kind: template_spawn | component_add
  consumer_binding: closed union
    template_spawn
      template_ref: exact RuntimeEntityTemplateV1 ref
      typed_parameter_hash
      persistence_disposition: optional ephemeral | persistent
    component_add
      target: RuntimeEntityHandle
      initializer_contract_ref: exact RuntimeComponentSpecV1 runtime-add ref
      initializer_parameter_hash
  external_store_ref
  component_type_ref
  owner_system_ref
  prepared_handle_value: exact external_handle_type_ref fixed-width value
  base_store_generation: uint64
  pending_store_generation: uint64
  prepared_object_hash: bytes32
  consumption_binding_hash: bytes32
  abort_contract_ref
  retire_contract_ref
  reservation_hash: bytes32

RuntimeEntitySpawnPreparationTicketV1
  preparation_request_key: RuntimeExternalPreparationRequestKeyV1
  template_ref
  issuer_system_ref
  typed_parameter_hash
  persistence_disposition: optional ephemeral | persistent
  external_reservation_handles: RuntimeExternalReservationHandle[]
  ticket_hash: bytes32
```

`RuntimeExternalPreparationRequestKeyV1`は通常Command／Eventとは別のScheduling-owned bounded preparation portでだけ生成する。SchedulerがManifestのpreparation bindingからWorld、request／target tick、phase、target boundary、requester runtime ID、logical work IDを埋め、`request_sequence`を同一logical workのpreparation request内で1からgapなしに付ける。Game Systemはこれらの順序Fieldを任意指定できない。targetはrequestより前にできず、Profileの最大preparation lead以内でなければならない。準備失敗はtyped failure Eventを返してrequestを閉じ、未発行のStructural Command keyやlocal sequenceを予約しない。

`base_store_generation`／`pending_store_generation`は対象external storeのowner participantが持つ§8.6 participant generationと同一のcounterであり、store専用の別counterを導入しない。baseはReservation発行時のcurrent publicationにおける値、pendingはこのReservationを消費するpublicationが公開する値である。このcounterは消費publicationが通常structural commitかSection publicationかを問わず、影響を受けたparticipantでexact 1進む(§8.6)。

`RuntimeSectionReservationHandle`と`RuntimeExternalReservationHandle`はいずれもindex32＋generation32、all zero invalid、wrap retireのruntime-only capability handleであり、Source、Artifact、Save、Replayへ保存しない。Section handleはpreflight完了時のcanonical participant順、external reservationとprepared external handleは`RuntimeExternalPreparationRequestKeyV1`＋Component Type ref順で割り当て、worker完了順を使わない。Section `transaction_hash`はRoot／epoch／pending generation／activation group順のexact Section change／各participant ref・prepared state hash・reserved capacity hash／handoff table hashをhashし、reservation handle値を除外する。Reservation／spawn ticket hashはstale／改変検出用のephemeral integrityで、Entity allocation、authoritative digest、Replay分岐へ入力しない。

`prepared_state_hash`はparticipant schemaに従うArtifact key、canonical descriptor、capacity reservation fact、input generationだけをhashし、native address、OS／GPU／vendor handle値、allocation orderを含めない。native objectとhashの対応はparticipant-owned tableとQualification Receiptが検証し、hashをnative handleの代替resolverとして使わない。

ReservationはComponent ownerのgenerated preparerだけが発行し、callerはhandle値を指定、改変、再利用できない。`consumer_binding`はkindと同名payloadを一件だけ持つ。spawn bindingはTemplate／parameter／persistence、add bindingはtarget handle／Component Specのruntime-add initializer／parameterをexactに固定し、`consumption_binding_hash`はpreparation request key、consumer binding、store／Component／owner、base／pending generationをcanonical hashする。external handleを含むTemplateでは、Scheduling-owned generated spawn coordinatorが宣言済みpreparation phaseで各Component ownerへtyped prepare commandを送り、全Reservationが揃った時だけsingle-use `RuntimeEntitySpawnPreparationTicketV1`をsealする。`persistence_disposition`はidentity policyが`optional`の場合だけ必須で、それ以外はabsentとする。lifetime ownerはticketのtarget tick／boundaryで同じTemplate／parameter／persistenceを持つ`create_entity`を発行し、異なる値への流用、部分ticket、期限後利用を拒否する。external handleを含まないTemplateはticket不要である。`add_component`はComponent owner自身のadd binding Reservationを同じtarget tick／boundaryのcommandへ束縛し、commitがvirtual sourceから選んだedgeのinitializer bindingとも一致させる。

CoordinatorはTarget Profileでboundedなticket tableを持ち、`ticket_hash`をkeyにexact recordを保持する。create commandの`spawn_preparation_ticket_hash`はこのtableから一件だけresolveし、command producer Systemがticketのissuer、BatchのWorld／working tickとcommandのapply boundaryがpreparation request targetに一致することを検証する。commit成功またはabort／期限切れでrecordを除去する。lookup失敗、二回目利用、issuer／Template／parameter／persistence／World／tick／boundary不一致では全関連Reservationをabortし、別ticketまたはraw handleへfallbackしない。

Template createまたはadd initializerはReservation tokenを受け、structural preflightがstore、owner、Component Type、transaction binding、generation、期限、重複を検査する。prepared objectとhandleはpending generationへ置き、publication slotがpending `RuntimeWorldPublicationHandle`へ切り替わるまでresolverから見えない。通常のprepare失敗はcommandをstructural batchへ提出する前のtyped failureであり、全prepared reservationをabortしてlive Worldを変更しない。

commit中のhidden attachはno-allocation／no-failでなければならない。内部不変条件違反時は新publicationを公開せずWorld faultとし、次phaseを開始しない。publish後はECS Componentとexternal resolverが同じpublication generationを見るため、「Componentは見えるがhandleが未生成」またはその逆の状態を作らない。remove／destroy時はECSから先に不可視化しても、old publication Leaseを持つconsumerのためobjectを保持し、`RuntimeComponentLifecycleDeltaV1`とretire fence完了後にownerだけがgenerationを進めて解放する。

## 11. Authoring、Cook、Playの関係

### 11.1 `RuntimeEntityProjectionV1`

```text
RuntimeEntityProjectionV1
  source_entity_id
  source_scene_ref
  persistent_identity
  world_streaming_plan_ref
  runtime_activation_group_id: optional uint32, 1..N
  initial_entity_enabled: bool
  entity_lifetime_save_policy: not_saved | persistent
  entity_enablement_save_policy: not_saved | persistent | reset_to_initial
  lifetime_owner_system_ref
  lifecycle_disposition: section_owned | transition_persistent | runtime_owned
  destruction_save_policy: persistent_tombstone | reset_to_source
  parent_disposition: typed_parent_component | derived_hierarchy_index | editor_only
  parent_projection_ref: optional
  tag_projections[]
  component_projection_refs[]
  source_recipe_ref: optional
  runtime_initializer_spec_ref: exact RuntimeEntityInitializerSpecV1 ref
  initializer_parameter_hash
  projection_hash
```

Authoring Entity recordは次の一意な規則でCookする。

| Authoring field | Runtime disposition |
|---|---|
| `entity_id` | `PersistentEntityIdentityV1.kind = authoring_entity`とIdentity Index |
| `enabled` | ChunkのEntity enable bit。通常queryはdisabled Entityを除外 |
| `lifecycle = scene_owned` | 所属SectionとWorld／Scene lifecycle ownerに束縛 |
| `lifecycle = persistent` | exact Save／transition handoff contractとpersistent lifetime ownerを必須化 |
| `lifecycle = streamed` | World Streaming PlanのSection／activation group ownershipに束縛 |
| `parent_entity_id` | `parent_disposition`に従いtyped Component、derived index、editor-onlyの一つへ投影 |
| `sibling_order_key` | hierarchy projectorが要求する場合だけderived indexへ入力。それ以外はEditor-only |
| `name` | Editor-only。Runtimeで必要なら明示したString／Localization Componentへ別投影 |
| `tags` | 各Tagのregistered projectionによりRuntime tagまたはEditor-only。未登録dispositionはCook error |
| `components` | `RuntimeComponentProjectionV1`へ入力 |
| `recipe_instance` | Initializer Spec／parameter生成元。Runtime instance pointerとして保存しない |
| `editor_metadata` | 常にstrip |

`scene_owned | streamed`は`lifecycle_disposition = section_owned`かつ所属activation group scopeへ投影する。`persistent`はWorld／Levelのregistered persistence policyが、always-residentな`runtime_owned + world_root_runtime`またはCook済みhandoff graphを持つ`transition_persistent + activation_group`のどちらか厳密に一つを選ぶ。`section_owned | transition_persistent`は`runtime_activation_group_id`を必須、`runtime_owned`はこのFieldを持たない。`runtime_owned`のimmutable base recordはRoot payloadへ例外追加せずbootstrap Section entity record setに格納するが、live scopeはworld rootでありbootstrap content ownershipとlifetime ownershipを同一視しない。Source lifecycleだけから距離、名前、current Cellで推測せず、policy欠落／重複／handoff case欠落をCook errorにする。

registered persistence policyは各Authoring Entityの`entity_lifetime_save_policy`も一意に決める。`not_saved`はenablement `not_saved`、destruction `reset_to_source`だけを許可し、Save recordを作らない。`persistent`はenablement `persistent | reset_to_initial`と、destruction `persistent_tombstone | reset_to_source`を明示する。`lifecycle = persistent`は必ずSave `persistent`であり、scene-owned／streamed EntityをSave対象にする場合もexact World Save policyとowner field Contractを必須とする。

Scene／Level／Cell membershipはWorld PlanとSection side metadataが所有し、Gameplay Componentへ重複保存しない。親が別Sectionへ存在する場合はWorld dependencyまたはpersistent handoffが明示されなければCookを拒否する。Entity-level `enabled`を全Componentのenable bitへ複写しない。

Compilerは全Section entity recordとhandoff placeholderを同時にindex化する。一つの`PersistentEntityIdentityV1`は通常entity recordとしてRoot全体に厳密に一件だけ存在でき、handoff placeholderは同identity・handoff ID・source／destination activation groupが一致する一件だけを追加できる。activation groupが同時activeにならないというPlan条件を理由に通常duplicateを許可せず、display name、Section順、後勝ちで統合しない。

複数Authoring Componentが同じRuntime Component Typeへ出力することを既定で拒否する。必要な場合は一つの`RuntimeCompositeComponentProjectorV1`が入力Type集合、canonical input順、field reducer、競合、provenance、fixtureを所有し、最終Runtime Componentを一度だけ出力する。implicit last-write-wins、document順、display orderでmergeしない。

### 11.2 `RuntimeComponentProjectionV1`

```text
RuntimeComponentProjectionV1
  authoring_component_type_ref
  runtime_component_type_refs[]
  projector_id
  projector_version
  target_profile_refs[]
  field_mappings[]
  unit_coordinate_conversion_refs[]
  default_and_missing_policy
  validation_refs[]
  fallback_refs[]
  fixture_refs[]
```

一つのAuthoring Componentを0～複数のRuntime Componentへ投影できる。0件はEditor-onlyであることをMCDへ明示する。Field mapping、単位、座標、量子化、default、Target除外はclosed schemaとfixtureを必要とし、unknown fieldのdrop、暗黙narrowing、display nameによる対応を禁止する。

### 11.3 Compile／load pipeline

1. `ProjectRevision`、Document hash、MCD、StableId参照を検証する。
2. Authoring Entity、hierarchy、lifecycle、tag、ComponentをTarget別Runtime Entity／Componentへ投影する。
3. Component Registry、State owner table、System implementation、Access Manifestを閉じる。
4. Runtime Entity Initializer Spec、Template、lifetime owner、identity／Save policy、owner-approved parameterとspawn boundを閉じる。
5. initial archetypeと全structural transition closureを生成する。
6. chunk layout、initial capacity、hard capacity、query cacheを生成する。
7. StableId順のEntity inputとWorld Streaming PlanからSection payload／Directoryを作り、Root hash fieldをzeroにした`section_content_hash`を計算する。
8. content hashだけを持つSection CatalogからRoot inner imageをCookし、`root_image_hash`を確定する。
9. exact Root hashを各Section Headerへ入れて最終`section_image_hash`を計算し、Artifact dependency DAG、outer Manifest／Qualification Binding、Catalog／Launch Setを生成する。
10. Package envelope、dependency、hash、Qualification Receiptを検証する。
11. LoaderがRoot、bootstrap Section、全participantをstagingし、全System registrationを照合する。
12. Rootとbootstrap activation groupの全Gate成功時だけpending publication Handleをrelease storeし、Runtime Worldをpublishして`PlayPreparing`から進む。

C1のRoot image、各Section image、Component Registry、Entity Construction set、Archetype PlanはPlay session中immutableである。Authoring変更は新Project revisionとして保存できるがactive Worldへ自動適用しない。同じRootが列挙したSection activation／deactivationと、Runtime Templateによるspawn／destroyだけがlive Entity集合を変更できる。Authoring由来のComponent schema、plan、Section bytes、初期値変更は次回Play開始で反映する。ECS contract／Section hot reloadは`not_activated`で、互換判定用の空APIを実装しない。

### 11.4 Save／Replay

```text
RuntimeEntityLifecycleSaveRecordV1
  persistent_identity: PersistentEntityIdentityV1
  origin_kind: authoring_entity | runtime_spawn
  disposition: present | destroyed_tombstone
  runtime_template_ref: optional                 # present runtime_spawnだけ必須
  entity_scope_ref: optional RuntimeEntityScopeRefV1
  lifetime_owner_system_ref: optional

RuntimeEntityCompositionSaveRecordV1
  persistent_identity: PersistentEntityIdentityV1
  base_composition_hash
  persistent_presence_component_type_refs[]
  composition_record_hash

RuntimeEntityEnableSaveRecordV1
  persistent_identity: PersistentEntityIdentityV1
  enabled: bool

RuntimeComponentEnableSaveRecordV1
  persistent_identity: PersistentEntityIdentityV1
  component_type_ref
  enabled: bool

RuntimeIdentitySequenceSaveRecordV1
  slot_spec_ref
  owner_state_instance_id
  last_issued_sequence: uint64

RuntimeEphemeralOrdinalSaveRecordV1
  last_issued_ordinal: uint64

RuntimeEntityReplayRefV1
  kind: persistent_identity | replay_entity_ordinal
  persistent_identity: optional PersistentEntityIdentityV1
  replay_entity_ordinal: optional RuntimeEphemeralEntityOrdinal

StructuralReplayRecordV1
  tick
  apply_boundary_id
  producer_system_ref
  producer_binding_ref
  stable_work_ordinal
  local_sequence
  operation
  entity_refs[]: RuntimeEntityReplayRefV1
  typed_payload
  typed_payload_hash
  commit_result

RuntimeAuthoritativeWorldDigestV1
  root_image_hash
  tick: uint64
  active_section_refs[]: RuntimeWorldSectionRefV1
  persistent_entity_state_hashes[]
  ephemeral_entity_state_hashes[]
  authoritative_state_store_hashes[]
  identity_sequence_table_hash
  ephemeral_last_issued_ordinal: uint64
  persistent_tombstone_table_hash
  world_digest: bytes32
```

Digestは全required structural boundaryとSystem validationが成功し、Command／Event acceptanceがsealされたtick publish boundaryでだけ作る。active Section refをactivation group順、persistent EntityをPersistent Identity canonical byte順、ephemeral Entityを`RuntimeEphemeralEntityOrdinal`順、State Storeを`{store spec ref, authoritative digest scope identity, State Type ref}`順に並べる。Entity stateはstable identity、lifetime owner、scope、Entity enable bit、authoritative Componentのpresence／enable bit／Field ID順canonical valueをhashし、relationはPersistent Identityまたはephemeral ordinal、authoritative external stateはprojector出力へ置換する。tag presenceを含む。derived／presentation-only Component、Runtime handle、archetype／chunk／row、address、padding、change epoch、publication／external handle値、worker順を除外する。各subhashと`world_digest`は該当hash Fieldをzeroにした`McdCanonicalBinaryV1` bytesへSHA-256を適用する。tickとRoot／Section集合をdigest subjectに含め、異なる時点またはcontent generationの偶然同値を同一checkpointと扱わない。

Saveは`SaveReplayContractV1`が列挙したauthoritative fieldを`PersistentEntityIdentityV1`、Component Type ID、Field ID、schema versionで保存する。全record集合はPersistent Identity canonical byte順、Component Type ref順である。Authoring projectionまたはpersistent runtime Template policyでlifetime Save対象となり、checkpoint時にpresentな全Entityはlifecycleとcomposition recordを一件ずつ持つ。Composition recordは`presence_save_policy = persistent`のうち最終的に存在するComponentだけを列挙し、非列挙をabsent、`reset_to_base` Componentをbase compositionどおりと解釈する。Field recordはこのpolicy適用後も存在するComponentだけに許可し、absent／reset-away Componentのorphan Fieldを拒否する。復元結果のarchetypeと各遷移がplan／owner policyにない場合はSaveを拒否する。

Lifecycle recordのvariant規則もclosedで、`origin_kind`は`persistent_identity.kind`と一致しなければならない。`present`は`entity_scope_ref`とlifetime ownerを必須とし、`origin_kind = runtime_spawn`だけがexact Template refを持ち、authoring originは持たない。persistent runtime spawnのscopeはTemplateどおり`world_root_runtime`でなければならない。`destroyed_tombstone`はauthoring originだけを許可し、Template／scope／owner Fieldを持たない。Compositionの`base_composition_hash`はauthoringならcurrent `entity_scope_ref`とHandoff graphが選ぶexact Section／bootstrap projection hash、runtime spawnならexact Templateのbase hashと一致し、Component refはcanonical byte順・重複なしである。

Authoring Entityをdestroyする時、`destruction_save_policy = persistent_tombstone`ならWorld-owned cold `RuntimePersistentTombstoneTable`へidentity、owner、destroy commit sequenceを同じstructural publicationで移し、Saveへ`destroyed_tombstone`を出す。`reset_to_source`はtombstoneを作らず次回loadでSection baseから復元する。runtime spawnのdestroyはlifecycle recordを削除し、sequenceを巻き戻さない。Save fileはruntime spawnのtombstone、未知Authoring identity、policy外tombstoneを作れない。

Entity／Component enable bitは通常Fieldへ偽装せず、policyが`persistent`の場合だけ上記built-in recordへ保存し、`reset_to_initial`はTemplate／Section初期値を使う。load時はidentityが一意、Componentが存在し`enable_bit`対応、owner policy一致を検証してから同じreconstruction transactionでbitを適用する。runtime spawnはTemplateのentity policy、各ComponentはRegistryのpolicyを使い、callerまたはSave fileがpolicyを上書きしない。

Save envelopeはexact Root image hash、active `RuntimeWorldSectionRefV1`集合、activation generation、Save／Replay Contract set hashを持つ。Section refはactivation group順で、missing image、content hash不一致、Root不一致、Plan migrationなしのSection差替えを拒否する。save可能なruntime spawnはexact Runtime Entity Template ref／revision、persistent identity、owner-owned save field、各streamの`last_issued_sequence`を記録する。0は未発行、`UINT64_MAX`は次発行でoverflow failureを意味し、loadで値を減らさない。各runtime spawn identityはTemplate policyのSequence Slot Specと同じowner state／streamを持ち、sequenceが`1..last_issued_sequence`内でなければならない。Slot recordの重複／欠落、identity重複、identityのsequenceより小さいcounterを拒否する。さらにWorld-level `RuntimeEphemeralOrdinalSaveRecordV1`をexact一件持ち、ephemeral Entity stateは持たず、load後の次ordinalだけを`last + 1`から継続する。runtime spawnのpersistent identityがないEntityをSave対象にしない。

Save loadは、exact Root／Section／Contract／migration検証、base Section staging、Authoring tombstone適用、runtime spawn Template再構築、persistent composition決定、owner Field migration、各Componentのsave reconstruction／external preparation、enable bit適用、identity sequence／ephemeral ordinal counter復元の順で一つのpending publicationを構築する。全Identity／owner／archetype／handle／capacity／digest検証後だけpublication Handleをstoreし、途中失敗ではstagingとReservationを破棄して旧WorldまたはTitleを維持する。Runtime handle、archetype ID、chunk ID、row、padding、raw Component bytes、change versionを保存または優先しない。

ReplayはRoot image hash、bootstrap／追加Section image hashとactivation generation、accepted input／async result、typed Command／Event、projected structural record、RNG mapping、commit result、authoritative digestを記録する。raw `StructuralCommandBatchV1`またはRuntime handleを保存しない。persistent Entity targetはPersistent Identity、ephemeral EntityはECSがcreate時に割り当てた`RuntimeEphemeralEntityOrdinal`へ投影する。Replay refはkindと同名Fieldだけを持ち、unknown／両方／どちらもなしを拒否する。typed payloadはoperation固有のTemplate／parameter／Component／enabled値をcanonical MCDで保持し、全Entity referenceをReplay refへ置換する。

Create tokenはcommit後ordinalへ解決し、seek checkpointはordinal→reconstructed Entity対応をReplay Ownerのtyped recordとして持つ。Replay実行時はstable System／binding refとordinalからcurrent handleへ解決し、記録handle値を復元しない。worker scheduling、address、chunk allocation時刻を記録または再現条件にしない。

ECS-owned cold ordinal indexはReplay recordingの有無にかかわらずlive ephemeral Entityについて存在し、Entity Capacity refでboundedである。Replay recorderはpublish safe boundaryでそのimmutable projectionを読むだけで、ordinalを発行または変更しない。Debug／Replay capture bufferが上限を超えた場合はordinalをwrapまたは不完全Replayをcomplete表示せず、captureを`truncated`として停止する。authoritative simulationとECS ordinal indexは変更せず、required Replay Qualificationだけを失敗させる。

mid-session recording開始はpublish safe boundaryでcurrent live ephemeral Entityを既存ordinal順に列挙し、current next ordinalとinitial checkpointを同時にsealする。新しい番号を振り直さない。初期列挙またはcheckpoint capacityが不足する場合はrecording開始自体を拒否し、部分indexから開始しない。

## 12. Subsystem integration

各接続は説明文ではなく、Domain Ownerが登録しContract compilerが束ねる次のbindingを持つ。

```text
RuntimeSubsystemPortBindingV1
  binding_id
  producer_system_ref
  consumer_system_ref
  port_type_ref
  authority_class: authoritative | derived | presentation_only
  source_phase_id
  delivery: same_boundary | next_phase | next_tick | async_accept
  target_phase_id
  seal_boundary_id
  identity_contract_ref
  generation_and_stale_policy
  capacity_ref
  overflow_policy
  failure_and_recovery_policy
  replay_contract_ref
  evidence_refs[]
  positive_fixture_refs[]
  negative_fixture_refs[]
  binding_hash
```

`identity_contract_ref`は`RuntimeEntityRefV1`、Persistent Entity Identity、Domain generation handleのいずれを運ぶかを固定する。`async_accept`はrequest ID、request／deadline tick、owner generation、input revision、Target versionの照合を必須とする。`overflow_policy`と`failure_and_recovery_policy`は登録済みclosed IDで、drop、delay、fallback、session faultを自由文から推測しない。

§12の表はbinding集合のhuman projectionである。E0で各Domain正本へexact System ref、Port Type／version、T10～T110のphase、same／next tick、seal、capacity、stale／overflow、fixtureを追加し、bindingがないAdapterをE5へ進めない。

| Consumer／Producer | ECSから出るもの | ECSへ戻るもの | 禁止する結合 |
|---|---|---|---|
| Physics | body creation／motion command、immutable step input | normalized contact／pose resultをPhysics Integration Systemが所有Componentへ反映 | native callbackからWorld write、body pointerのComponent保存 |
| Navigation | obstacle／agent snapshot、versioned query request | request ID順のpath resultをNavigation ownerへ統合 | live Physics／ECS pointer参照、worker完了順採用 |
| Animation | state／parameter／motion input snapshot | root motion／event／pose resultをAnimation ownerへ統合 | bone pointerをECSへ保存、Physicsから最終poseへ直接write |
| Rendering | presentation extraction SystemがsealしたRender Snapshot | authoritative ECSへ戻さない | RendererによるWorld lease保持、GPU visibilityをGameplay authorityに使用 |
| Audio | Audio Command／listener snapshot | completion／underrun EventをAudio ownerへ返す | callback threadからECS access／allocation |
| VFX | VFX spawn／parameter Command | presentation Evidenceだけ。Gameplay結果は別authoritative Systemが所有 | particle stateをauthoritative Componentにする |
| Input | tick開始でlatchしたInput Snapshot | Gameplay Command | Device callbackからWorld mutation |
| UI | Game UI Snapshot、typed UI Command target | verified Interaction Request | pixel／widget pointerをEntity identityにする |
| World streaming | Cell activation／deactivation intent、Root／Section Artifact ref | participant Reservationと`RuntimeSectionPublicationTransactionV1` | streaming workerからlive chunk変更、個別Entity commandでunload代用 |
| Debug／Replay | bounded captured ECS snapshot、Contract Graph、diagnostic | control requestはScheduling ownerへ | Debug UIからComponent memory write |

各Integration Systemも通常の`GameSystemSpecV2`、Manifest、phase、State owner、Budget、fixtureを持つ。Section publicationへ参加するDomainはPort bindingとは別に`RuntimeSectionPublicationParticipantV1`を一件持ち、通常tick data flowとpublication lifecycleを同じcallbackへ混在させない。Adapter固有schemaは各Domain Owner、ECSへのdata placementとleaseはECS Owner、順序はScheduling Ownerが決める。二つの文書が同じFieldまたはphaseを再定義しない。

## 13. C++23 APIとNative ABI

Contract compilerはComponentとQueryごとにnamed C++23 moduleを生成する。Project C++はgenerated moduleをimportし、Engine private header、chunk header、location table、registry、allocatorへinclude／linkしない。

概念上のtyped surfaceは次に限定する。

```text
RuntimeEntityHandle
RuntimeWorldInstanceHandle
RuntimeEntityRefV1
ReadLease<T...>
WriteLease<T...>
OptionalColumnRead<T>
OptionalColumnWrite<T>
SelectedRowsView<QueryId>
DenseSelectedRowsView<QueryId>
QueryBatch<QueryId>
EntityReadPort<T...>
TargetedCommandBatch<CommandId>
StateReadLease<T...>
StateWriteLease<T...>
StructuralCommandWriter<PermissionSet>
EntityCreateToken
RuntimeEntityTemplateRef
RuntimeSystemExecutionContextV1
```

`QueryBatch`はEntity handle view、typed Component column、row range、logical work IDだけを返す。World singleton、runtime string lookup、unchecked random Entity writeを提供しない。単一Entityの例外的参照はManifest-declared `EntityReadPort`またはownerへのtyped Commandを使い、任意registry lookupをhot path APIにしない。

### 13.1 Native ECS batch ABI

既存[Native Game Module](../03-authoring/native-game-module.md)の`NativeSystemDescriptorV1`と`MirakanNativeInvokeContextV1.query_batches`をclean breakで次へ置換する。

```text
MirakanNativeEcsColumnViewV1
  struct_size: uint32
  abi_version: uint32 = 1
  component_runtime_id: uint32
  component_schema_version: uint32
  access_code: uint32
  present_u32: uint32
  element_size: uint32
  element_alignment: uint32
  stride_bytes: uint32
  row_count: uint32
  layout_hash: bytes32
  enable_mask_words: callback-scoped const uint64 pointer | null
  enable_mask_word_count: uint32
  term_match_mask_words: callback-scoped const uint64 pointer | null
  read_base: callback-scoped const byte pointer | null
  write_base: callback-scoped byte pointer | null

MirakanNativeEcsBatchViewV1
  struct_size: uint32
  abi_version: uint32 = 1
  world_instance_handle: RuntimeWorldInstanceHandle
  world_epoch: uint64
  phase_id: uint32
  query_runtime_id: uint32
  logical_work_id: RuntimeLogicalWorkIdV1
  row_begin: uint32
  row_count: uint32
  selected_count: uint32
  selection_mask_words: callback-scoped const uint64 pointer
  selection_mask_word_count: uint32
  entity_handles: callback-scoped const RuntimeEntityHandle pointer
  columns: callback-scoped const MirakanNativeEcsColumnViewV1 pointer
  column_count: uint32
  lease_token: opaque uint64
```

`RuntimeLogicalWorkIdV1`は§9.3の6個の`uint32`からなるfixed-width tagged tupleである。Native ECS batchでは`work_kind_code = 1`、`binding_runtime_id = query_runtime_id`を必須とし、Compilerがcanonical partition planから割り当てる。worker／dispatch順を使わない。

C ABIのclosed codeは`access_code = 0 invalid | 1 presence | 2 read | 3 read_write`である。`present_u32`は0または1だけを許可し、C／C++ `bool`、enumのcompiler依存underlying type、bitfieldをABIへ出さない。`phase_id`、runtime ID、全closed codeは`uint32`で、0 invalid、unknown valueを拒否する。`RuntimeEntityHandle`と`RuntimeWorldInstanceHandle`は各二つの`uint32`、`bytes32`は`uint8[32]`として投影する。すべてのABI structは`struct_size`と`abi_version = 1`を先頭に持ち、reserved fieldは0、pointerはHost process内のcallback-scoped addressである。

Native descriptorはbinary imageのpacked形式ではなく、exact Toolchain／C ABI Profileが生成するnatural-layout C structである。Generatorは各Field offset、alignment、`sizeof`、pointer width、endiannessをABI descriptor hashへ含め、HostとModuleのC／C++ headerへ`static_assert`を生成する。`struct_size`はgenerated exact sizeと等しい場合だけ受理し、prefix／larger-size compatibilityを提供しない。implicit tail paddingをwire／hashへserializeせず、Profile不一致を`struct_size`だけで近似受理しない。

Hostはcallback直前に§9.1の正規化query term順でdescriptorを作り、`none` termは列に含めない。非存在optional／any列は`present_u32=0`、両base／全mask pointer null、mask count 0とし、§9.1のconceptual all-zero term maskをこのcanonical absent表現から復元する。存在する`presence` accessは`present_u32=1`で両base null、必要なenable／term maskだけを持つ。`read`はread baseだけ、`read_write`は同じlive columnを指すread／write baseを持つ。baseは`element_alignment`を満たし、`stride_bytes = sizeof(generated T)`、`row_count`はBatchと一致する。Tagは`present_u32=1`でも両base null、element size／stride 0である。全descriptorは存在有無にかかわらずregistered Typeのschema version／layout hashを持つ。

enableable columnの`enable_mask_word_count`は`ceil(row_count / 64)`、mask pointerはnon-null、structural-only Componentはcount 0／nullである。`any` termの`term_match_mask_words`は同じword countを持ち、それ以外はnullである。すべてのtail bitを0にする。

selection mask word countは`ceil(row_count / 64)`で1以上、mask pointerはnon-nullである。selected countはmask popcountと一致し、tail bitは0でなければならない。generated Native wrapperも`SelectedRowsView`だけをProject callbackへ渡し、mask false rowのbase addressを返さない。

generated C++ wrapperだけがruntime ID、schema、layout hash、size、alignment、access、lease tokenを検査して`ReadLease`／`WriteLease`／`OptionalColumn`へ変換する。Project callbackへEngine Component owner、chunk header、location table、allocatorを渡さない。descriptorとbaseのlifetimeはcallback returnまでで、Hostはreturn直後にtokenをinvalidateする。

callback return後、次のSystemを開始する前にHostはreturn code、lease token、declared written row rangeと、generated field validatorによるfinite／enum／range／cross-field invariantを検査する。合格後だけCommand／Event／Structural outputをsealする。不合格は`ECS_COMPONENT_VALUE_INVALID`でtickをpublishせず、部分値を後続Systemへ見せない。

### 13.2 Descriptor、State、output writer

`NativeSystemDescriptorV1`へContract set hash、Toolchain／C++ ABI Profile hash、ECS Profile hash、Component Registry hash、Entity Construction set hash、Archetype Plan hash、Access Manifest hash、Query descriptor table hash、State Store descriptor hashを必須化する。HostはModule load時にgenerated callback signatureと全hashを照合し、一件でも不一致ならModule全体を拒否する。

System-owned Stateは同じ原則の`MirakanNativeStateViewV1`で渡し、`struct_size`、`abi_version = 1`、State Store ref、scope instance ref、State Type／layout hash、`access_code: uint32`、base、lease tokenを持つ。Stateの`access_code`は`0 invalid | 2 read | 3 read_write`だけを許可する。`output_writer`は宣言済みCommand、Event、Structural Commandだけを受理し、Component valueまたはSystem State deltaを書けない。Component／Stateのwrite経路をdirect generated leaseへ一本化する。

ABIを越えるのはfixed-width scalar、generation handle、POD descriptor、Host-owned callback table、callback-scoped byte rangeだけである。STL type、exception、RTTI、owner object、lease ownerを渡さない。Native moduleはHostとexact Toolchain／C++ ABI Profileへ一致する必要があり、異なるcompiler／runtime／packingをC ABIだけで近似対応しない。

generated fileの直接編集をCIで拒否する。Project handwritten codeはgenerated bindingを利用するだけで、Component Type ID、field offset、Manifest、query、phaseを再宣言しない。raw descriptor／baseの保存、cast、address escapeはNative Contract違反であり、§9.3のQualification境界を適用する。

## 14. AI可読Contract

### 14.1 `RuntimeEcsContractGraphV1`

Contract compilerは次のnode／edgeを持つread-only graphを生成する。

- Component node: Type、意味、state class、owner、Source projection、layout、Save、Target。
- System node: Contract、Implementation、phase、query、Budget、qualification。
- Archetype node: Component集合、capacity、initial／reachable、transition。
- Entity Initializer／Template／Identity Policy node: Source、archetype、lifetime owner、runtime scope、Component initializer／initial enablement、spawn parameter、persistence／Save、Budget。
- System State Store node: owner、scope、State Type、access、Save、capacity。
- Port node: Command、Event、Snapshot、external store、Subsystem Adapter。
- World Artifact node: Root／Section subject、activation group、content／image hash、dependency、Content Group、Qualification。
- Publication Participant／Persistent Handoff node: owner／approval、prepare／retire、generation、Section change、source／destination scope、archetype case、Component／Field／enablement disposition。
- Edge: `owns`、`reads`、`writes`、`queries`、`adds`、`removes`、`initializes`、`scoped_to`、`transitions_to`、`projects_from`、`emits`、`consumes`、`extracts_to`、`depends_on`、`published_with`、`hands_off_to`、`approved_by`、`qualified_by`、`stored_in_content_group`。

各edgeはexact Stable ID／Contract ref、version、phase、delivery、required／optional、fallback、evidence refを持つ。自由文だけで関係を表さない。Graph hashはComponent Registry、System dependency graph、State owner table、Access Manifest、Entity Template／Identity Policy set、Archetype Plan、Section Catalog、participant Contract、persistent handoff、Qualification Policyのhash closureであり、AIだけの別Graphを手編集しない。live publication generation／Entity valueはContract Graph hashへ混ぜず、inspect captureがexact Root／Section／publication／tick refで別に束縛する。

### 14.2 Query operation候補（未採用）

本Decisionは状態が`ユーザー確認待ち`であり、次の五IDは採用前の提案語彙だけである。[Executable contracts](../02-foundation/executable-contracts.md#212-architecture内operation-tokenのclosed-partition)の`example_pending_or_rejected` classに属し、current MCD／Owner Manifest／Service allowlist／Policy／Validator／Diagnostic／Receipt／Provider／MCP／generated alias／legacy alias集合はすべて`[]`で、dispatchしない。本Decisionが採用されても、別のatomic activation work itemが五件の完全closureを登録するまではcurrentへ自動昇格しない。

| proposed non-current ID | 予定入力 | 予定出力 |
|---|---|---|
| `operation.runtime_ecs.search` | semantic role、Type、System、phase、Target、qualification | Stable ref候補と選択理由 |
| `operation.runtime_ecs.describe_contract` | exact Component／System／Query／Archetype／Template／Identity Policy／Root／Section／Participant／Handoff ref、field mask | bounded Contract Graph slice |
| `operation.runtime_ecs.inspect_capture` | capture ID、tick、Persistent Entity Identityまたはquery ref、field mask | immutable captured values、redacted field、omitted range、provenance |
| `operation.runtime_ecs.explain_access` | System ref、Component ref、phase | owner、read／write、dependency、conflict、remediation |
| `operation.runtime_ecs.explain_failure` | diagnostic ID、trace ref | violated invariant、affected owner、safe next planned semantic action |

```text
RuntimeEcsAiRequestContextV1
  ai_caller_context_ref, ai_caller_context_sha256
  task_authorization_envelope_hash
  project_revision
  contract_set_hash
  capture_binding:
    {capture_id, world_generation, tick, capture_content_sha256}
```

将来五候補を完全Activationする場合は、AI SecurityがGateway署名したcurrent `AiCallerContextV1`と、そのContextに束縛された`TaskAuthorizationEnvelope` hashを全named inputへ必須にする。`engine_provider_adapter`、`standard_external_mcp`、`managed_external_host`のpresence／authority／Profile／Freshness規則をそのまま使い、ECS独自channelまたはgrantを発明しない。standard MCPだけが`McpSessionGrantV1`を持ち、managed routeはfresh `ManagedHostSessionAttestationV1`を持つ。現Product Definitionでmanaged edit／buildは`not_activated`であり、この提案R0 Queryから権限昇格しない。unknown route、branch間Field混在、stale Head、期限切れ／Project不一致grant、Capture binding差をfail closedにする。現在はR0であっても製品内Provider、MCP、CLIへ公開せず、要求を`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`でstate不変として拒否する。

live `RuntimeEntityHandle`またはlive memoryをAI tool inputにしない。Debug Ownerがphase boundaryでsealしたimmutable captureか、Authoring StableIdとWorld／tick／revisionを必須にする。検索はbounded result、readはfield mask、node／edge／byte上限、omitted range、continuationを持つ。partial responseを完全なWorldとして扱わない。

各MCD Fieldの`sensitivity = public | project_private | restricted | secret`と`ai_exposure_policy_ref`を、Task Authorization、requested field mask、channel固有Policyのintersectionで評価する。`product_internal`はProvider Policy、`local_mcp`はMCP grantのread sensitivity、`managed_cli`はHost／session grantとProvider利用時のProvider Policyを使う。許可外Fieldは`redacted_fields[]`へField IDとreasonを載せ、default値、null、field不存在へ置換しない。`secret`、User text、Credential、native pointer、raw pathをcapture contextへ含めない。redactionまたはomissionでowner／failure判断に必要なFieldが欠ける場合、AIは完全な説明または変更提案へ昇格せず`insufficient_authorized_context`を返す。

R1／R2の`runtime_ecs.create`、`set_component`、`add_component`、memory write operationは生成しない。`operation.authoring.search`／`operation.authoring.read`は現在`planning.operation_family.authoring_discovery`のreserved candidateであり、MCD／Manifest／Service／Tool集合は空、`not_activated`である。将来familyがatomic Activationされた場合だけ、AIはこれらでSourceを確認し、typed `ProjectChangePrimitiveV1`を含む`ProjectChangeSetV1`を提案する。それまでは別名read Toolへfallbackせず`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`を返す。Gateway validation、human approval、Cook、次回Playという既存経路を迂回しない。

### 14.3 AI向け生成物

各Component／Systemはpurpose、non-responsibility、owner、valid／invalid example、unit／coordinate、range、phase、failure、Budget、Source／Derived区分、Target、qualificationを持つ。Contract compilerはC++、TypeScript、JSON Schema、MCP、human referenceへ同じField IDとclosed enumを投影する。

AI contextは最初にCatalogとGraphの候補だけを返し、必要なComponent／System／fixtureを段階取得する。全World dump、全Source、全Subsystem schemaを一つのPromptへ投入しない。未知IDを表示名や近似文字列へ自動補正せず、候補と差分を返す。

## 15. Failure、diagnostic、atomicity

`detection_stage`は`contract_compile | cook | module_load | play_prepare | section_preflight | runtime_access | structural_commit | system_execute | retirement | qualification`のclosed enumである。一つのDiagnostic IDとstageの組は次のstate effectへ一意に対応する。

| Diagnostic ID | detection stage | 条件 | 必須挙動 |
|---|---|---|---|
| `ECS_COMPONENT_CONTRACT_INVALID` | `contract_compile` | Type、owner、field、表現kind違反 | Contract compile失敗 |
| `ECS_ACCESS_ORDER_AMBIGUOUS` | `contract_compile` | 同phaseの競合accessにdependencyがない | Contract compile失敗。ID順へ暗黙整列しない |
| `ECS_ARCHETYPE_CAPACITY_INVALID` | `cook` | row capacity 0、hard bound不足 | Cook失敗 |
| `ECS_ENTITY_INITIALIZER_INVALID` | `cook` | lifetime owner、Component owner、parameter、archetype、external reservation不正 | Cook失敗 |
| `ECS_ENTITY_TEMPLATE_INVALID` | `cook` | Initializer ref、identity／Save policy、spawn bound不正 | Cook失敗 |
| `ECS_SOURCE_IDENTITY_CONFLICT` | `cook` | Authoring／Section identity重複またはscope違反 | Cook失敗 |
| `ECS_PERSISTENT_HANDOFF_INVALID` | `cook \| section_preflight` | source／destination、Field disposition、owner、identity重複不正 | Cookまたはpreflight拒否。旧publication維持 |
| `ECS_ACCESS_DESCRIPTOR_MISMATCH` | `module_load` | callback、Manifest、query、layout、ABI hash不一致 | Module全体のload拒否 |
| `ECS_ARTIFACT_QUALIFICATION_BINDING_INVALID` | `play_prepare \| section_preflight` | outer署名、subject image hash、policy、Gate、Receipt、freshness不一致 | Play開始またはSection preflight拒否。inner image非publish |
| `ECS_ROOT_IMAGE_INCOMPATIBLE` | `play_prepare` | Root hash、Target、Contract、owner不一致 | Play開始拒否。World非publish |
| `ECS_SECTION_PREFLIGHT_REJECTED` | `section_preflight` | I/O、capacity、dependency、participant prepare、cancel、stale generation | old World generation維持。全Reservation abort、typed retry／fallback可 |
| `ECS_ENTITY_WORLD_MISMATCH` | `runtime_access` | `RuntimeEntityRefV1`のWorld不一致 | resolve失敗。null／別World slotへfallbackしない |
| `ECS_ENTITY_HANDLE_STALE` | `runtime_access` | index、generation、alive state不一致 | typed stale result。read／async pathはWorldを変更しない |
| `ECS_ACCESS_CAPABILITY_VIOLATION` | `runtime_access` | loaded ModuleがManifest外Type／mode／phaseへaccess | tick非publish、World fault |
| `ECS_LEASE_EXPIRED` | `runtime_access` | checked accessorでepoch／phase／tick失効を検出 | tick非publish、World fault |
| `ECS_STRUCTURAL_MUTATION_DURING_ITERATION` | `runtime_access` | 即時structural API試行 | operation拒否、tick非publish、World fault |
| `ECS_TRANSITION_NOT_DECLARED` | `structural_commit` | plan外add／remove | transaction reject、tick非publish、World fault |
| `ECS_STRUCTURAL_TARGET_INVALID` | `structural_commit` | stale／別World target、owner／precondition違反 | transaction reject、tick非publish、World fault |
| `ECS_STRUCTURAL_CONFLICT` | `structural_commit` | 同一targetへの競合command | transaction reject、tick非publish、World fault |
| `ECS_STRUCTURAL_BUDGET_EXCEEDED` | `structural_commit` | Game System command／staging／chunk上限超過 | commit前reject、tick非publish、World fault。heap fallback禁止 |
| `ECS_RUNTIME_IDENTITY_CONFLICT` | `structural_commit` | spawn identity重複／sequence overflow | commit前reject、tick非publish、World fault |
| `ECS_EXTERNAL_HANDLE_RESERVATION_INVALID` | `structural_commit` | preparation request、owner、consumer binding、World／tick／boundary、generation、prepared state不一致 | transaction reject、tick非publish、World fault |
| `ECS_SECTION_COMMIT_INVARIANT_BROKEN` | `structural_commit` | preflight済みSection transactionのhidden attach／内部不変条件違反 | new generationをpublishせず旧generationを最後の可視状態としてWorld fault |
| `ECS_COMPONENT_VALUE_INVALID` | `system_execute` | owner write後のnon-finite／field invariant違反 | tick非publish、World fault、部分tick破棄 |
| `ECS_PUBLICATION_RETIRE_STALLED` | `retirement` | retire fence deadlineまたはretired-generation backlog上限超過 | 新activation停止、World fault／teardown。旧resource早期解放とnew generation rollback禁止 |
| `ECS_NATIVE_BORROW_ESCAPE` | `qualification` | borrow保存／raw ABI迂回をstatic／sanitizerで検出 | Module promotion拒否 |

Diagnostic envelope、severity、provenance、trace transportは既存Ownerを参照し、本書はECS固有conditionとstate effectだけを所有する。全ErrorはComponent／System／World／tick／phase／Contract hashのうち利用可能なexact ref、期待値、観測値、安全な修正Operationを持つ。

## 16. Performance、capacity、risk

### 16.1 Performance原則

- hot iterationは事前生成query cacheと既存chunkだけを使い、steady-state allocationを0にする。
- per-Entity virtual dispatch、unordered iteration、runtime RTTI、string lookup、Component別heap objectをhot pathに置かない。
- structural changeは指定boundaryへ集約し、複数の暗黙sync pointを作らない。
- Systemは必要列だけを取得し、optional／cold dataをhot archetypeへ無条件追加しない。
- Presentation extractionは必要なsealed snapshotだけを作り、Renderer等へWorldを貸さない。
- Archetype数、chunk occupancy、move bytes、query match数、structural queue、lease faultを常時telemetry化する。

### 16.2 正本Budgetとの関係

Entity数、chunk memory、archetype数、query数、command数、staging bytes、System time、snapshot bytesのhard／soft boundは[Performance／Capacity](../04-runtime/performance-capacity.md)のTarget Profileが所有する。既存のStructural command `16,384 / simulation step`、`2 MiB`、Simulation command総量をECS文書へ複写せずexact refで消費する。

ECS ProfileはBudget値ではなく、参照するProfile ID、Component制約、layout algorithm、partition policy、fault policyを束ねる。ProjectがSource Component追加でTarget上限を引き上げない。上限未測定、Receipt期限切れ、Target fixture不足は`qualified`にしない。

### 16.3 主なriskと対策

| Risk | 対策 |
|---|---|
| archetype explosion | 全composition／transitionをoffline列挙しhard bound化 |
| structural churn | enable bit、owner command、telemetry、boundary集約 |
| large Componentでoccupancy低下 | inline 256-byte上限、cold store／Asset handle、layout receipt |
| false sharing／並列競合 | chunk／row partition、exclusive key、logical work ID |
| stale pointer／ABA | 32＋32 generation、World ref、wrap retire、lease epoch、callback scope、borrow escape Qualification |
| nondeterministic iteration | canonical archetype／chunk／row順、canonical command merge |
| ContractとC++ layout drift | generated struct、layout fingerprint、load-time hash Gate |
| Saveがinternal layoutへ依存 | Field ID projectionだけを保存、raw chunk禁止 |
| AIがlive stateを破壊 | Runtime R0のみ、変更はAuthoring Gateway経由 |
| custom Backendの保守負荷 | Public Contract固定、small kernel、differential／fuzz／benchmark Gate |

## 17. Verification Gate

### 17.1 Contract／static Gate

- 全Component Type ID、Field ID、System ref、Query ID、Diagnostic IDの一意性。
- Component owner 100%、authoritative Componentのsingle writer 100%。
- 全Componentのbase／persistent presence／Field／external projector／reset enableを閉じるowner save reconstruction contract coverage 100%。
- Initializer Spec／初期Entityのlifetime owner 100%、Component initializerのowner approval 100%、Templateのidentity／runtime scope／spawn policy coverage 100%。
- 全SystemのManifest、query、phase、Native descriptor、State owner table hash一致。
- 全State Storeのscope instance、single owner、access、initialization、Save projection一致。
- 全structural permissionからのarchetype transition closureがfiniteかつplan内で、add transitionのowner initializer／parameter schema／external reservation bindingがCommand validatorまで一致。
- C++／TypeScript／JSON Schema／MCP／human referenceのfield、enum、bound、hash一致。
- `McdCanonicalBinaryV1` descriptor、Field order、presence、wire type、boundのC++／TypeScript generator一致。
- inner image、Qualification binding、Artifact Manifest、Catalogのhash dependency DAGにself edge／cycle 0件。
- 旧名称、ECS値の重複正本、未解決Marker、不完全な節が0件。

### 17.2 Correctness Gate

- create／destroy／add／remove／Component enable／Entity enableのvalid、invalid、競合、temporary identity test。
- Template外field指定、非owner create／destroy／add／remove／enableを全拒否するauthorization test。
- tag／typed external handleへのvalue write、raw handle assignment、same-boundary remove＋add rebindをCompile／transactionで拒否する。
- generation reuse、wrap retire、別World、double destroy、stale resolve test。
- same-World weak relationのtarget destroy後にtyped staleだけを返し、暗黙clear／cascade／slot再利用先への誤resolveが0件であることを確認。
- all／any／none／optional／enable predicateのtruth-tableと、Source term順を変えてもquery hash／ABI column順が同一になる正規化test。
- any-of複数一致でもEntity重複0、optional／disabled列のpresence解釈一致。
- random operation列を単純reference modelと比較するdifferential test。
- swap-back後のlocation table、identity index、query resultの全件一致。
- allocation、capacity、validator failureを全commit stepへ注入し、live digest不変を確認。
- identity policyのforbidden／optional／required、persistent sequence／ephemeral ordinalのexact／overflow、失敗時counter非消費、通常createからidentity値注入不可を確認。
- Authoring destroyのpersistent tombstone／reset-to-source、runtime spawn destroy、runtime-add Componentのpersistent／reset composition、owner Field／external reconstruction、enable bit、identity sequence／ephemeral last-issued counterをSave→loadし、plan外archetypeとpolicy外／orphan recordを拒否する。
- external handleのprepare／abort／hidden attach／publish／retireへfailureとstale World／tick／boundary tokenを注入し、未解決handle公開0、旧Lease中の早期解放0、prepare失敗時のStructural Command sequence消費0を確認。
- Section preflightのI/O／capacity／participant prepare／cancel／stale failureでold generation維持、hidden attach不変条件違反でnew generation非publishとWorld faultを確認。
- pinned Catalog snapshotと異なるgeneration／Root dependency key／Launch Set hashのSectionを拒否し、Receipt renewal後もlive Worldが世代混在しないことを確認。
- ECS、Rendering、Collision、Physics、Navigation participantの各順序へfailureを注入し、部分generationが全consumerから観測不能であることを確認。
- persistent handoffのlive source不足、到達可能source archetype case欠落、重複destination、Component／Field未列挙、owner approval不足、migration／external handle transfer失敗で旧Entity／handle／publication不変を確認。
- malformed Root／Section image、layout mismatch、unknown Type／Field／enumのnegative test。
- `McdCanonicalBinaryV1`のgolden byte、cross-language encode／decode／re-encode、non-canonical order／tail bit／trailing byte拒否test。
- Receipt更新でinner image hashが不変、Root更新時は下流Section outer dependency／Artifact keyとLaunch Set／CatalogをDAG順に同時更新し、旧新key混在とbinding hash cycleがないことを確認。
- Section content hashをRoot hash埋込み前後で再計算し、Root catalogはcontent hashだけ、outer Catalogは最終image／Artifact refだけを持つことを確認。
- generated Component、chunk move、lease invalidation、structural fuzzをaddress／undefined-behavior／thread sanitizer構成で検証。

### 17.3 Determinism／Replay Gate

- 同じRoot／Section集合／inputを1、2、最大workerで各100回実行し、authoritative digest、Entity allocation、Command／Event、structural commit順を一致させる。
- worker完了順、chunk address、allocation patternを乱しても結果一致。
- Save→load、Replay→seek→continue後のPersistent Entity Identity／field digest一致。
- Physics／Navigation async resultのaccept tickとowner generation不一致discardを再現。

### 17.4 Performance／endurance Gate

適用時期はCapability成熟度で段階化し、E0／C1 baselineとC2／stress Qualificationを同じCompletion Gateへ混在させない。

E0-defined C1 Gate（E0はcontractを固定し、Runtime実装後に`wp.gameplay.core-c1`が実行）:

- Product RegistryのFirst Playable fixtureを使い、fixtureが参照するTarget ProfileとECS Contract setをReceiptへ固定する。
- entity数は[Performance／Capacity](../04-runtime/performance-capacity.md)がTarget別に定めるcanonical C1 capacity以内とし、固定のsynthetic件数へ置換しない。
- 8／16／32 KiB chunkを同一fixtureで比較し、16 KiBを無測定で変更しない。
- steady-state query iteration allocation 0、一般heap fallback 0。
- chunk occupancy、cache miss、iteration throughput、structural move bytes、P50／P95／P99.9、peak memoryをTarget別記録。
- 30分連続soakでleak、generation異常、query cache drift、unbounded archetype増加0。

C2／stress Qualification（`wp.product.general-coverage-2d`／`wp.product.general-coverage-3d`）:

- 100万Entity synthetic scale、全archetype、最大宣言query、Section churn。
- persistent handoffについて到達可能なsource／destination archetypeの全matrixをpositive／negative fixtureで検証する。
- 2時間enduranceでleak、generation異常、query cache drift、unbounded archetype増加0。

C2／stressは[Product Plan](../00-product/product-plan.md)の共通Runtime C2 fault／soak／scale Qualificationと[Performance／Capacity](../04-runtime/performance-capacity.md)のProduction enduranceを所有する将来Qualification Work Packageで実行し、E0完了またはC1 baselineの条件にしない。

### 17.5 Integration／AI Gate

- Authoring→Cook→Load→Play→Save／Replay round tripをWindows、Android、Appleのrequired Targetで検証。
- Physics、Navigation、Animation、Render、Audio、VFX、Input、UI、World streamingの各Portにpositive／negative fixture。
- Subsystem callbackからWorld write 0、Renderer／Audio callbackのWorld lease 0。
- AI Catalog／Graphから任意ComponentのSource、owner、reader、writer、phase、archetype、failure、fixtureへ到達可能。
- Runtime write operation 0、AI Authoring mutation coverage 100%、bounded responseとcontinuation conformance。
- product internal／local MCP／managed CLI各variantの必須／禁止field、Authorization／channel grant／sensitivity不足、restricted／secret Field、redaction／omissionをdefault値と区別するnegative fixture。

Qualificationはcompile成功またはmicrobenchmark一件で代替しない。全required Target、統合負荷、failure injection、provenanceを持つReceiptだけを`qualified`へ昇格できる。

## 18. 実装順序と依存関係

本設計は一つの巨大Taskとして実装しない。各段階は前段のContractとtest oracleを完了条件にし、未完の下位層をmock成功扱いして上位層を昇格しない。

E0 Task 0を含む全Taskは、外部Schedulerがcurrent Product／Control Planeから次の開始条件をすべて満たし、`wp.runtime.ecs-e0`を正当に`active`へ進めた後だけ開始する。Task 1で作るECS technical documentはE0中`review`、E0 output後のhuman `approve`をE1 entry条件とし、E0開始条件へ循環させない。文書stateを自己申告せずcurrent approvalとWP lifecycleを別々に検証する。

| Exact entry condition | Required value |
|---|---|
| `current_control_plane_baseline_binding` | kind別完成wrapperへ解決し、current Product snapshotと一致 |
| `active_product_definition_sha256` | current Product snapshot／binding／ECS WP definition seedで一致 |
| Owner approvals | Product operational Owner `runtime-scheduling-lifetime`、Governance、Compatibility／Evolution、Persistence／Saveがcurrent approved。新ECS technical document approvalはE1から必須 |
| `wp.runtime.scheduling-core` lifecycle | `complete` |
| `wp.runtime.ecs-e0` lifecycle | `active`かつdefinition seed valid。外部Schedulerが`declared->ready->active`を完了済み |
| Toolchain／revocation | current、non-stale、non-revoked |

baseline read-backは[Architecture Evolution Control Plane実装計画](../../plans/2026-07-22-architecture-evolution-control-plane-implementation-plan.md)とDesignのclosed `CurrentControlPlaneBaselineBindingV1`だけを使い、Reader側でField集合を再定義しない。kind=`bootstrap | rebaseline`の完成Approval／Envelope／Transaction、Baseline Core、Local Schema Catalog、Authority Binding Source Catalog、Toolchain、Trust closure、Active Definition hashを再計算する。欠落／不一致／staleは`diagnostic.architecture.baseline-mismatch`、Owner非承認は`diagnostic.architecture.owner-unapproved`、WP state／seed差は`diagnostic.product.work-package-entry-invalid`、dirty treeは`diagnostic.architecture.dirty-baseline`で停止する。Branch名、現在HEAD、日時、`latest`、初回Bootstrap Approvalの直接参照をbaseline identityにしない。E3以降が追加Owner approvalを必要とする場合は各段階Gateに置き、E0 entryへ先取りしない。

| 段階 | 実装範囲 | 入力 | 出力／Gate |
|---|---|---|---|
| E0 Contract baseline | ECS正本、MCD schema、ID、Diagnostic、Root／Section binary、Artifact subject／Qualification binding、Port／publication binding、生成物 | 本Decisionと既存Owner | cross-doc／schema／codegen conformance |
| E1 Storage kernel | World state、handle allocator、registry、archetype plan、chunk、location／identity／sequence／tombstone table、State Store、publication Handle／Record | user-approved E0 Contract baseline | reference model、failure-atomic storage test |
| E2 Query／mutation | query cache、typed lease、partition、StructuralCommandBatch | E1＋Scheduling phase | truth table、race／determinism／transaction test |
| E3 Cook／load | Entity／Component projection、persistent handoff、layout generator、Root／Section binary、Artifact envelope／binding、loader | E0～E2＋Authoring／Asset | reproducible image、hash非循環、malformed input、Target ABI Gate |
| E4 Game System binding | Access Manifest、generated C++、Native batch／State ABI、scheduler graph | E0～E3＋Gameplay／Native | generated surface外accessのQualification拒否、1／N worker一致 |
| E5 Subsystem integration | World Section publication participant、typed external store、Physics、Navigation、Animation、Presentation、Audio、UI Port | E4＋各Domain | all-participant atomic visibility、recoverable preflight fixture、pointer逆流0 |
| E6 Debug／AI | authorized capture、telemetry、Contract Graph、R0 operations | E0～E5 | sensitivity、bounded context、explainability、provenance Gate |
| E7 Qualification | scale、Target、endurance、Save／Replay、failure injection | 全段階 | C1 receipt set(§17.4のC1昇格Gate適用分)とproduction昇格判断。§17.4のC2昇格GateはC2昇格時に実行する |

E0承認前にE1以降の実装を開始しない。E1とE3のschema prototypeはE0 fixtureとして作れるが、Production storageまたはpackageとして扱わない。E5はSubsystemごとに並行化できるが、共通ECS／Scheduling Contractをforkしない。

## 19. 正規仕様の変更計画

ユーザーが本Decisionを承認した後、同一clean-break ChangeSetで次を変更する。表中の`architecture-governance.md`、`compatibility-evolution.md`、`persistence-save.md`、`runtime-package.md`の4文書は現時点で未作成であり、[Architecture Evolution Control Plane実装計画](../../plans/2026-07-22-architecture-evolution-control-plane-implementation-plan.md)が新規作成する成果物である。当該4行は既存文書の編集ではなく、同計画完了後の新設文書への登録・接続を意味する。

| Path | 変更 |
|---|---|
| `docs/architecture/04-runtime/entity-component-system.md` | 新規正本。§8～§17のnormative contractを配置 |
| `docs/architecture/01-governance/architecture-governance.md` | ECS文書metadata、Owner、direct `requires`、reciprocal Contract ID、ChangeSet impactを登録 |
| `docs/architecture/02-foundation/compatibility-evolution.md` | ECS contract major、clean-break ID migration、Save／Runtime Package compatibility classを登録 |
| `docs/architecture/04-runtime/persistence-save.md` | ECS save projector、persistent identity、tombstone、migration、load stagingの集約Ownerへ接続。ECSがSave transactionを再定義しない |
| `docs/architecture/04-runtime/runtime-package.md` | Root／Section image、Component／System contract set、layout fingerprint、migration setをRuntime Package closureへ接続 |
| `docs/architecture/README.md` | RuntimeにECSを追加し、metadataからactive inventoryを再生成する。固定件数を規範にしない |
| `docs/architecture/00-product/product-plan.md` | ECSをC1 Runtime foundation Capabilityへexact refで接続。詳細を複写しない |
| `docs/architecture/02-foundation/executable-contracts.md` | ECS MCD kind、R0 operation、生成projectionと`McdCanonicalBinaryV1`の共通wire／cross-language fixtureを正本化 |
| `docs/architecture/02-foundation/memory-pointers.md` | 16 KiB／64-byte／256-byteのECS固有値を削除しECSへ参照。一般handle／lease規則だけ残す |
| `docs/architecture/01-governance/ai-security-approval.md` | ECS captureのproduct internal／local MCP／managed CLI caller variant、Task Authorization、channel grant、sensitivity／redaction接続を追加 |
| `docs/architecture/01-governance/ai-verification-provenance.md` | Root／Section inner image hashをQualification subjectとして登録し、Receipt renewalはouter bindingだけを更新する規則を追加 |
| `docs/architecture/03-authoring/project-state.md` | Authoring Entity／ComponentからRuntime projectionへのrefを追加 |
| `docs/architecture/03-authoring/asset-lifecycle.md` | `DerivedArtifactManifestV1`のtagged subjectへclean置換し、generic Catalog key／snapshot ref、World Root／Section envelope、Content Group、outer Qualification bindingを正本化 |
| `docs/architecture/03-authoring/gameplay-programming-model.md` | GameSystemSpec、Component／State access、Entity lifetime owner、Contract Graphの関係を追加 |
| `docs/architecture/03-authoring/native-game-module.md` | `RuntimeEntityRefV1`、mutable／readonly列descriptor、State view、Manifest hashへclean rename／接続。immutable query batch／Component State deltaを削除 |
| `docs/architecture/04-runtime/scheduling-lifetime.md` | ECS storage本文を削除し、新正本参照、World state、participant prepare／hidden attach／single publication boundary、retire fence、process fault接続だけを残す |
| `docs/architecture/04-runtime/performance-capacity.md` | ECS Budget owner／telemetry linkを追加。既存値はここに保持 |
| `docs/architecture/04-runtime/debugging-observability-replay.md` | ECS capture／diagnostic／Contract Graph transport、authoritative World digest、lifecycle／composition／enable／identity sequence／ephemeral ordinal counter Save／Replay envelopeを追加 |
| `docs/architecture/05-simulation/*.md` | live pointerを使わないIntegration System／Port参照を整合 |
| `docs/architecture/06-rendering/*.md` | Presentation extraction、Root／Section streaming、activation group `content_group_ref`、persistent handoff、Subsystem publication participantを整合 |
| `docs/architecture/07-platform/*.md` | Input／Audio／UIとTarget qualificationを整合し、`mobile-common.md`の旧suffixなしArtifact Manifest参照を`DerivedArtifactManifestV1` subjectへ移行 |
| `docs/architecture/08-domain-packs/shooter.md` | Shooter固有System／Component compositionをECS正本参照へ変更 |

外部Dependencyを追加しないため`toolchain-dependencies.md`へFlecs／EnTTのpinは追加しない。比較根拠のURLは本DecisionとECS正本の外部根拠節に置き、Runtime manifestへLibrary名を混入させない。

## 20. 明示的に閉じる判断

1. Runtime ECS BackendはEngine-ownedである。
2. Storageはarchetype＋SoA chunkで、sparse-setを標準Backendにしない。
3. 16 KiB payload、64-byte alignment、inline Component最大256 bytesをC1値とする。
4. Entity handleは`uint32 index + uint32 generation`で、0 invalid、wrap retireとし、callback外はWorld handleを含む`RuntimeEntityRefV1`を使う。
5. Componentはgenerated／registered data-only typeで、non-trivial objectをinlineに置かない。
6. Archetypeとtransitionはoffline planへ列挙し、Runtime暗黙生成を禁止する。
7. Queryはtyped、cached、boundedで、runtime string DSLを公開しない。
8. authoritative Componentはsingle owner／single writerである。
9. structural mutationは六つのclosed operationとし、Game System boundary全体をatomicにする。
10. Root／Section bytesとContractはPlay中immutableとし、同じRoot内のatomic Section activationは許可するがECS hot reloadを実装しない。
11. Save／ReplayはPersistent Entity IdentityとField IDを使い、raw chunkまたはRuntime handleを保存しない。
12. Subsystemは`RuntimeSubsystemPortBindingV1`で接続し、ECS pointerまたはleaseを保持しない。
13. AIのRuntime ECS surfaceは認可済みimmutable captureに対するR0 read-onlyで、変更はAuthoring ChangeSetだけを使う。
14. 旧名称、旧schema、Library互換API、dual Backendを残さない。
15. Network／distributed／GPU-authoritative ECSは`not_activated`である。
16. Entity relationはDomain-owned typed Component／indexで表し、generic pair DSLまたはsingleton Entityを導入しない。
17. runtime spawnはowner-approved `RuntimeEntityTemplateV1`だけを使い、全Entityにlifetime ownerを一つ割り当てる。
18. World／Level／Encounter global Stateはgenerated State Storeへ置き、singleton Entityまたは二重write経路を作らない。
19. Native Component／State writeはManifest準拠のcallback-scoped batch viewへ一本化し、output writerはCommand／Event／Structuralだけに使う。
20. 通常のSection preflight failureは旧Worldを維持し、Contract／内部不変条件違反だけをWorld faultにする。
21. Section activationは全participantのprepared stateをhidden attachし、`RuntimeWorldPublicationHandle`のrelease storeだけを唯一の可視commit pointにする。
22. persistent EntityのSection移送は`RuntimePersistentEntityHandoffV1`に全Field dispositionを列挙し、C1ではRuntime handleを維持する。
23. Root／Section inner imageはQualification Receiptを含めず、outer bindingがimage hashへReceiptを束縛してhash cycleを作らない。
24. `DerivedArtifactManifestV1`はAsset／World Root／World Sectionのtagged subjectを使い、旧Manifest、synthetic Asset ID、dual Catalog keyを残さない。
25. `typed_external_handle` objectはpublish前Reservationでpreparedし、同じpublication generationで可視化し、Lifecycle Delta後かつconsumer fence完了後にretireする。
26. Native C ABIのclosed code、presence、versionは固定幅整数で表し、C／C++ `bool`またはcompiler依存enumを渡さない。
27. Runtime ECS AI operationはTask Authorizationを全channelで要求し、MCP grantは`local_mcp`だけ、製品内とmanaged CLIは各channel固有Policy／attestationを使う。
28. Save／Loadはpersistent Entityのlifecycle、composition、enable bit、tombstone、identity sequenceとWorld-level ephemeral last-issued counterをtyped recordで再構築し、ephemeral Entity state、raw handle／archetype ID／chunk bytesを保存しない。
29. RootはSection content hashだけへcommitし、最終Section image／Artifact keyはouter Catalogへ置いてhash cycleを禁止する。

これらの判断は実装計画で再選択しない。変更する場合は本Decisionを改訂またはsupersedeし、根拠、影響、再Qualificationを明示する。

## 21. 外部一次資料と採用した教訓

| 一次資料 | 固定Evidence | 確認した事実 | Miraikanaiでの扱い |
|---|---|---|---|
| [Unity Entities Archetypes](https://docs.unity3d.com/Packages/com.unity.entities%401.4/manual/concepts-archetypes.html) | 2026-07-22取得、page表示1.4.8、UTF-8 response SHA-256 `2c1c94b105ab2e9b063411c48b2a788f30aafa52bda1d4a1bb6a33e1c18a4758` | 同一Component集合をarchetype化し、Component列とEntity ID列をchunkへ格納する。add／removeでarchetype間移動する | archetype／SoA／moveの妥当性を裏付ける。Unity API／formatは採用しない |
| [Unity Entities Structural changes](https://docs.unity3d.com/Packages/com.unity.entities%401.4/manual/concepts-structural-changes.html) | 2026-07-22取得、page表示1.4.8、UTF-8 response SHA-256 `1ad3a1a3282cd86babea95fd0f7819cf27c074dc4127360fb4c3ed74a9effbbf` | create／destroy／add／removeはchunk再編とsync pointを生む | phase中即時mutation禁止とbatch boundaryの根拠にする |
| [Unreal Engine 5.8 MassEntity Overview](https://dev.epicgames.com/documentation/en-us/unreal-engine/overview-of-mass-entity-in-unreal-engine?application_version=5.8) | Engine documentation 5.8、2026-07-22取得、UTF-8 response SHA-256 `7dd0df00d93f5d94489c9546aae6225fc645ab69e5fe748ee32a1b7df74af7b3` | Fragmentはdata-only、同compositionをarchetype／chunkへ格納し、ProcessorはQueryを使い、composition変更をCommand Bufferでbatch処理する | Component／System分離、query batch、deferred mutationの根拠にする |
| [Flecs v4.1.2 Queries](https://github.com/SanderMertens/flecs/blob/a53b4715c0b91e366bbfce53d9abd2b61534f7a3/docs/Queries.md) | tag `v4.1.2` commit `a53b4715c0b91e366bbfce53d9abd2b61534f7a3`、raw UTF-8 SHA-256 `8f2fef1d674358fb08f397f406edfd499c153a089a51dfd81ab1c72d793bdcc1` | archetype単位のquery cache、read／write access、iteration中moveの危険とdeferが文書化される | query cache、access manifest、defer原則を採る。DSL／relationship／World APIは採らない |
| [Flecs v4.1.2 Systems／Staging](https://github.com/SanderMertens/flecs/blob/a53b4715c0b91e366bbfce53d9abd2b61534f7a3/docs/Systems.md) | 同commit、raw UTF-8 SHA-256 `e4789dd957318022443a9ef41e3e93cc0c3b9d8a28b80a55481bbd8651111aa7` | staging中のstructural operationはcommandとしてenqueueされ、merge後に反映される | private batchとcanonical mergeの比較根拠にする |
| [EnTT v3.16.0 Entity documentation](https://github.com/skypjack/entt/blob/b4e58bdd364ad72246c123a0c28538eab3252672/docs/md/entity.md) | tag `v3.16.0` commit `b4e58bdd364ad72246c123a0c28538eab3252672`、raw UTF-8 SHA-256 `7e05ad7f4b7ac263a6807d9a84dce55fc5a0a3589633e9073b8267793127e68f` | sparse-set pool、view／group、entity version、optional pointer stabilityとiteration制約を提供する | 世代検査と比較benchmarkの参考にする。標準storageには採らない |

外部資料にある値、API、内部挙動が更新されても、本DecisionのMiraikanai Contractは自動変更されない。更新はToolchain／Architecture reviewとMiraikanai fixtureを経る。

## 22. 承認境界

本Decisionの承認は、Engine-owned archetype ECS、正本所有、clean-break名称、Authoring／Runtime分離、System access、structural transaction、Subsystem Port、AI read-only surface、実装段階、Verification Gateへの承認を意味する。

詳細Task、exact file、MCD、C++ interface、test、command、expected result、clean migrationは[Runtime ECS E0 Implementation Plan](../../plans/2026-07-22-runtime-ecs-e0-implementation-plan.md)に事前定義した。同計画の存在は本Decisionの承認、E0 entry gate通過、Capability activationを意味しない。

E0は§18のexact五条件で開始し、本DecisionとECS active正本を`review`のままcontract／fixture／baselineとして検証できる。E0完了も自動approvalではない。ユーザー承認後だけ本Decisionとactive正本を`approved`へ更新し、E1 storage kernel以降のRuntime実装とCapability activationへ進む。承認前にE1以降のEngine Runtime codeを変更しない。
