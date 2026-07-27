# Miraikanai Engine Runtime Entity Component System Contract

- 文書ID: mirakan.arch.runtime-entity-component-system
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Runtime Entity／Component identity、initializer・template・archetype／SoA layout、query・selection・dispatch・cache、Component access manifest、lease、structural transaction、runtime state binding、ECS AI contract graph、ECS固有diagnostic・qualification
- 非正本範囲: Runtime phase／job DAG、generic artifact envelope／catalog、World package binary、Save／Replay payload、debug transport、AI authorization・route grant、各Domain Componentの意味・field。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Core Architecture](../02-foundation/core-architecture.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Scheduling／Lifetime](scheduling-lifetime.md)
- 関連文書: [Architecture Governance](../01-governance/architecture-governance.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Gameplay programming model](../03-authoring/gameplay-programming-model.md)、[Native Game Module](../03-authoring/native-game-module.md)、[Scheduling／Lifetime](scheduling-lifetime.md)、[Runtime Package](runtime-package.md)、[Persistence／Save](persistence-save.md)、[Debugging／Observability／Replay](debugging-observability-replay.md)、[Performance／Capacity](performance-capacity.md)、[Runtime ECS Design Closure Review](../appendices/runtime-ecs-design-closure-review.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-24

## 1. 状態と結論

本書はEngine-owned archetype／SoA Runtime ECSの目標正本である。比較対象EngineのAPI、型、保存形式、scheduler、World semanticsを採用しない。採るのは、archetypeにより同じComponent集合を列指向に配置すること、反復利用するqueryをcacheできること、iteration中のstructural mutationをdeferすること、世代付きhandleと宣言的accessを使うことだけである。ECSはEntity／Component固有のhandle、query lease、structural invalidationを所有し、一般pointer taxonomy、leaseの保存／capture禁止、memory resourceは[Memory／Pointers](../02-foundation/memory-pointers.md)のbindingを消費する。ECS独自のwrapperでこれらを再定義しない。

本書の`review`状態と型定義は、ECS runtime、MCD、Operation、Tool、Native ABI、Package reader、Save readerが既にactiveであることを意味しない。[Governance Migration Proposals](../appendices/governance-migration-proposals.md#2-runtime-ecs-canonicalization-candidate)の`RuntimeEcsCanonicalizationChangeSetV1`候補が別途承認され`applied`となり、complete Consumer Inventory、Compatibility Change、Owner reference migration manifest、Owner Registry revision 2、Foundation Definition Closure、affected Contract set、Definition Migration binding、全Evidence Requirementのpass satisfaction binding、qualification evidenceが同一closureで整合した後だけcurrent化できる。

current authorityが移管されるまで、`owner.core.runtime_ecs` revision 1のauthority documentは[Gameplay programming model](../03-authoring/gameplay-programming-model.md)である。本書はその移管先を定義し、実装Task Planや実装手順を定義しない。

### 1.1 Target contract closure

本書のMarkdown recordはtarget semanticsを固定するが、それだけでmachine-readable Definition Closureにはならない。current化前に、本文が参照する全Type／Refを一意なtop-level Schemaへ解決し、全collectionのelement type、最小／最大件数、canonical sort、uniqueness、branch制約、canonical encoding、self-excluding hash preimageを閉じる。少なくとも`CanonicalValueRefV1`、`ComponentSchemaRefV1`、`ComponentTransitionRefV1`、`EntityCreateToken`、`EntityTemplateRefV1`、`ImmutableBitsetRefV1`、`InitializerSpecRefV1`、`RuntimeChunkId`、`StableArchetypeId`は、Owner、version、identity、hash、lifetimeを持つexact定義へ解決されるまでunresolvedである。

unresolved Type、略記`refs[]`、未materialize fixture、missing golden hash、owner不明のdiagnosticが一件でもある場合、Contract compilerはRuntime ECS Definition Closureを生成せず、Owner移管、Phase Gate、Capability Activation、Native load、Package load、Save／Replay reader、AI complete explanationを成功にしない。監査状態とcross-owner論点は[Runtime ECS Design Closure Review](../appendices/runtime-ecs-design-closure-review.md)で追跡する。同Reviewは本書のsemanticsを上書きしない。

## 2. 設計不変条件

1. EntityとComponent storageはEngine-ownedであり、third-party ECS runtimeへ委譲しない。
2. Componentはdata-only MCD valueであり、virtual update、native object、unbounded payload、raw pointerをinlineに持たない。
3. Runtime Entity identityはtyped index＋generationであり、persistent identity、Source identity、Asset identityと混同しない。
4. 同一authoritative fieldの同一advanceにおけるwriterは一つである。parallel writeはmanifestとquery selectionが証明する非重複row範囲に限る。
5. structural mutationはiteration中にWorldを直接変更せず、deferred commandをT110でsealし、指定された次のT00でprivate working Worldへatomic commitし、成功advanceのT110でだけ外部publicationへ反映する。
6. query callbackは一つの物理的に連続したchunk row rangeだけを処理し、複数chunkをまたぐcallback unitを作らない。
7. regular value writeはScheduler scope内だけで可視であり、external consumerはseal済みsnapshot／publication以外からpartial advanceを観測しない。
8. live Entity handle、live memory、native pointerをAI input、Save、Replay、artifact、cross-session digestへ入れない。

## 3. EntityとComponentの境界

### 3.1 Runtime entity identity

```text
RuntimeEntityHandle
  index: uint32
  generation: uint32

RuntimeWorldInstanceHandle
  index: uint32
  generation: uint32

RuntimeEntityRefV1
  world_instance_handle: RuntimeWorldInstanceHandle
  entity_handle: RuntimeEntityHandle
  world_epoch: positive uint64
  world_publication_generation: positive uint64
```

各handleのall-zero値はinvalidである。slot reuse時はgenerationを増やし、wrapするslotはretireする。`RuntimeEntityHandle`は同一Worldに束縛されたcallback内だけで使える。command、event、async result、debug request、Subsystem Portのようにcallbackを越える参照は`RuntimeEntityRefV1`を使い、World instance、World epoch、publication generation、Entity generation、alive stateを全て検査する。

handleはobjectを所有せず、Source、Save、Replay、cross-session digestへ保存しない。永続化が必要なEntityはPersistence Ownerが定めるPersistent Entity Identityへ投影し、持たないEntityはephemeral ordinalとして明示的に除外する。

`world_epoch`は一つの`RuntimeWorldInstanceHandle`内にあるprivate working Worldの構造世代である。eligible `T00_BoundaryApply`でsealed structural batchのpreflightとcommitが全件成功した場合だけexact 1増加し、regular value write、T110 snapshot publish、失敗したstructural batchでは増加しない。`world_publication_generation`は外部consumerへ可視なseal済みWorld publicationの世代であり、Simulation Advance全体が成功して`T110_Publish`でpublishされた場合だけexact 1増加する。一つのpublicationは、そのsnapshotが表すexact `world_epoch`を記録する。

初期constructionを全件検証したprivate working Worldは`world_epoch=1`から開始し、最初の成功`T110_Publish`を`world_publication_generation=1`とする。published Refで0を許可しない。初回publication前を表すboundary preconditionだけは`expected_source_world_publication_generation=0`を許可し、その場合もstaged construction set、Contract Set、Target Profileのexact一致を要求する。

`RuntimeEntityRefV1`のresolveはWorld instance、World epoch、publication generation、Entity generation、alive stateを全て一致させる。古いpublicationから取得したRefを新しいworking epochへ暗黙rebaseせず、同じEntity slot／generationに見えてもepochまたはpublication generationが異なればstaleとして拒否する。

### 3.2 Component contract

Componentのfield構造、semantic owner、persistence policy、sensitivity、external handle projectorはDomain OwnerのMCDだけが定める。ECSはComponentを次のstorage classで扱う。

| storage class | ECS規則 |
|---|---|
| `inline_value` | fixed-size、trivially relocatable、canonical field encodingを持つ値列 |
| `tag` | presenceだけを持ち、payload列を作らない |
| `external_handle` | Domain Ownerのtyped handle列。native objectやraw pointerを埋め込まない |
| `derived_index` | owner Systemが再構築する補助index。Source／Saveへ二重保存しない |

inline Componentは最大256 bytes、fixed alignment、bounded field集合を満たす。可変長data、長いtext、Editor metadata、native object、non-trivially relocatable objectはchunkへ直接置かず、AssetまたはDomain-owned typed handleへ移す。native padding、`sizeof(T)` bytes、compiler declaration orderをcanonical hash、Save、Replay、AI captureに使わない。

```text
RuntimeComponentLayoutPolicyV1
  policy_version: 1
  component_schema_ref: ComponentSchemaRefV1
  storage_class:
    inline_value | tag | external_handle | derived_index
  access_manifest_refs[1..4096]:
    sorted unique RuntimeComponentAccessManifestRefV1
  access_cohort_hash: SHA-256
  update_frequency:
    every_advance | periodic | event_driven | load_only
  structural_frequency:
    stable | bounded_transition | burst_candidate
  inline_bytes: uint32
  inline_alignment_bytes: uint32
  layout_disposition:
    accepted_inline | accepted_tag | accepted_external
    | component_schema_revision_required
  qualification_profile_ref:
    exact {profile_version, target_profile_ref, contract_set_ref,
           toolchain_lock_sha256, profile_hash}
  policy_hash: SHA-256
```

`access_cohort_hash`はexact active Game System manifest、query ref、phase ref、read／write setから決定論的に導出し、手書きhot／cold labelを入力にしない。PolicyはComponentごとに次のlayout reviewを行う。

- 一つのComponent typeを一つのSoA columnへ置く。
- `inline_value` Componentのfieldはcanonical value layoutを維持する。
- 同じsemantic ownerとaccess cohortを持つfieldをcolocateする。
- large cold／infrequent stateをhot Componentから移す場合はDomain-approved schema revisionを必須にする。
- variable-length data、text、native object、large immutable shared dataはtyped handleへexternalizeする。
- semanticsがarchetype transitionを必要としないfrequently toggled stateはvalueまたはenablement mutationを使う。
- unbounded tag combinationとshared-value partitionを拒否する。
- row capacityがqualified ceilingに達している、またはindirection costが大きい場合、`sizeof`削減だけを理由にsplitしない。
- authoritative fieldごとにactive writer一件とexplicit persistent projection一件を維持する。

disjoint System cohortが一つのlarge Componentの別fieldを使う場合は`component_schema_revision_required`を返し、自動field rearrange／splitを行わない。

`policy_hash`は`SHA-256(ASCII "MIRAKAN_RUNTIME_COMPONENT_LAYOUT_POLICY_V1" || uint32_be(len(canonical policy bytes excluding policy_hash)) || canonical policy bytes excluding policy_hash)`で計算する。Policy instanceはimmutable Contract Set root確定後だけmaterializeし、bare ID、`latest`、ambient current、same-name substitution、same-logical-key／version different-hashを拒否する。

## 4. Construction、template、archetype

### 4.1 Construction set

```text
RuntimeEntityConstructionSetV1
  construction_set_version: 1
  initializer_spec_refs[0..4096]: sorted unique InitializerSpecRefV1
  entity_template_refs[0..4096]: sorted unique EntityTemplateRefV1
  component_schema_refs[1..8192]: sorted unique ComponentSchemaRefV1
  archetype_layout_plan_refs[1..8192]: sorted unique RuntimeArchetypeLayoutPlanRefV1
  persistent_identity_policy_refs[0..4096]
  state_store_binding_refs[0..4096]
  construction_set_hash: SHA-256
```

construction setはGatewayから独立したimmutable inputである。Gatewayがconstruction setを参照する一方向だけを許可し、相互Refによるhash cycleを作らない。initializerはEntity Template、initial Component value、persistent identity policy、lifetime policyを参照する。templateは初期archetypeを一意に決め、runtime create後に任意Componentを足して意味を完成させることを許可しない。create permissionはmanifestがhash束縛したtemplateだけを指定できる。

### 4.2 Archetype layout

```text
RuntimeArchetypeLayoutPlanV1
  layout_algorithm_id: ecs_chunk_soa_v1
  archetype_id: StableArchetypeId
  required_component_schema_refs[]: sorted unique ComponentSchemaRefV1
  optional_enablement_component_refs[]: sorted unique ComponentSchemaRefV1
  payload_bytes: 16384
  payload_alignment_bytes: 64
  inline_component_max_bytes: 256
  row_capacity: positive uint32
  column_layout_hash: SHA-256
```

target C1のchunk payloadは16 KiB、payload先頭alignmentは64 bytesである。payloadにはEntity handle列、Entity enable bitset、`inline_value` Component列、enablement ComponentのbitsetをSoAで置く。Component列はalignment降順、同値はComponent Type refのcanonical byte順で配置し、template declaration order、compiler layout、link orderから推測しない。

row capacityは全列とbitsetが16 KiBへ収まる最大整数であり、0はcompile errorである。Componentのadd／remove、Entity destroy、archetype移動は末尾rowをgapへmoveし、location tableを同一commitで更新する。Component address、row、span、chunk IDはstructural commit、phase終了、World destroyで無効になる。

16 KiBはMiraikanaiのtarget design valueであり、Unity互換の数値ではない。8／16／32 KiBを同一fixtureで比較し、Performance Ownerがevidenceを承認するまで変更しない。

C1 Shippingの唯一のaccepted layoutは、payload 16,384 bytes、payload alignment 64 bytes、inline Component上限256 bytesの`ecs_chunk_soa_v1`である。8,192／16,384／32,768-byte比較がtargetを変更できるのは新しいqualified profileとcontract revisionを通した場合だけで、Runtime fallback pathまたは三Shipping layoutを作らない。

### 4.3 World build gateway

```text
RuntimeWorldBuildGatewayV1
  gateway_version: 1
  world_root_image_ref: RuntimeWorldRootImageRefV1
  world_section_image_refs[0..4096]: sorted unique RuntimeWorldSectionImageRefV1
  construction_set_ref: RuntimeEntityConstructionSetRefV1
  runtime_package_ref: RuntimePackageRefV1
  target_profile_ref: TargetProfileRefV1
  contract_set_ref: ContractSetRefV1
  required_capacity_ref: RuntimeWorldCapacityRecordRefV1
  validation_receipt_refs[1..128]
  gateway_hash: SHA-256
```

Gatewayはimmutable Package／World imageを検証し、construction setをstagingしてRuntime Worldへ渡す境界である。Gatewayはlive World、Editor object、native objectを所有せず、Package payloadの詳細形式は[Runtime Package](runtime-package.md)が所有する。capacity reservation、missing section、schema mismatch、identity collision、contract mismatchのいずれかでWorldをpublishしない。

## 5. Query、selection、dispatch

### 5.1 Query definitionとcache

queryはrequired／excluded Component presence、enablement、field predicate、partition policyを持つimmutable Contractである。頻繁に実行するqueryはarchetype membershipだけをcacheできる。cacheはWorld epochとarchetype layout hashに束縛し、structural commit後に無効化または差分更新する。predicateの結果row、lease、component addressをcacheへ保持しない。

```text
RuntimeQueryV1
  query_id: StableId
  query_version: positive uint32
  required_component_schema_refs[1..1024]:
    sorted unique ComponentSchemaRefV1
  excluded_component_schema_refs[0..1024]:
    sorted unique ComponentSchemaRefV1
  required_enabled_component_refs[0..1024]:
    sorted unique ComponentSchemaRefV1
  field_predicate_policy_ref: optional McdContractRefV1(kind=policy)
  partition_policy: canonical_range | deterministic_hash
  query_hash: SHA-256

RuntimeQueryRefV1
  query_id: StableId
  query_version: positive uint32
  query_hash: SHA-256

RuntimeQuerySelectionMaskV1
  query_ref: RuntimeQueryRefV1
  world_epoch: positive uint64
  archetype_id: StableArchetypeId
  chunk_id: RuntimeChunkId
  row_begin: uint32
  row_end_exclusive: uint32
  entity_enablement_mask_ref: optional ImmutableBitsetRefV1
  component_enablement_mask_refs[0..64]
  predicate_result_mask_ref: optional ImmutableBitsetRefV1
  selected_row_count: uint32
  selection_hash: SHA-256
```

selection maskは`[row_begin, row_end_exclusive)`の物理row範囲だけを表す。presenceはarchetype membershipで決まり、enablementとfield predicateはmaskのintersectionで決まる。maskはimmutableでcallback scopeに束縛し、row数、bitset length、World epoch、query hash、chunk capacityの不一致を拒否する。

### 5.2 Dispatch plan

```text
RuntimeQueryDispatchPlanV1
  plan_version: 1
  query_ref: RuntimeQueryRefV1
  world_epoch: positive uint64
  partition_policy: canonical_range | deterministic_hash
  work_units[0..65536]:
    logical_work_id: bytes32
    archetype_id: StableArchetypeId
    chunk_id: RuntimeChunkId
    row_begin: uint32
    row_end_exclusive: uint32
    selection_mask_ref: RuntimeQuerySelectionMaskRefV1
  plan_hash: SHA-256
```

`work_units[]`は`archetype_id`、`chunk_id`、`row_begin`の昇順でcanonical orderにする。一つのwork unitは必ず一つのchunk内の連続row rangeであり、selection maskが示すselected rowsだけをcallbackへ公開する。`deterministic_hash`はlogical work assignmentを安定化するためのpolicyであって、複数chunkを一callbackへ束ねる意味ではない。各unitの`logical_work_id`はquery、World epoch、archetype、chunk、row range、selection hashからcanonicalに導出し、worker index、OS scheduling、completion順を入力にしない。

callbackは、selectionされていないrow、別unitのrow、別queryのrowへread／writeできない。work unitの分割・mergeは同じinputから同じplan hashを生成し、callback側の物理メモリ配置を意味へ露出しない。

Cached queryはarchetype membershipだけを保存する。一つのgenerated callbackは一つのchunk内のcontiguous row rangeだけを受け取り、declared columnとimmutable-selected rowだけを公開する。query scratch、selection mask、command buffer、output packetをdispatch前に全量reserveする。Callbackはgeneral heap allocation、upstream allocator fallback、unbounded container growth、shared ownership acquisition、structural mutationを行えない。create、destroy、add、remove、enablement changeはstructural boundary向けbounded deltaにする。

## 6. Access manifest、lease、state binding

### 6.1 Component access manifest

```text
RuntimeComponentAccessManifestV1
  manifest_version: 1
  system_ref: GameSystemContractRefV1
  permitted_phase_refs[1..64]
  query_refs[0..256]
  read_component_refs[0..1024]
  write_component_refs[0..1024]
  structural_permissions[0..256]
  state_store_binding_refs[0..256]
  budget_refs[0..64]
  manifest_hash: SHA-256
```

read／write集合、query集合、phase、structural permission、state bindingはmanifestで閉じる。CompilerはSystem registration、query contract、Component owner、template、initializer、persistence policy、capacity budgetとの一致を検証する。未宣言Component access、任意Componentを対象にするstructural permission、phase外callback、manifest外state storeを生成bindingから表現できないようにする。

`structural_permissions[]`はoperation kind、許可target Component／transition、create可能template hash、apply boundary、count／byte budget、lifetime ownerを持つ。`create_entity`、`destroy_entity`、`set_entity_enabled`はEntity lifetime owner、`add_component`、`remove_component`、`set_component_enabled`は対象Component ownerからだけ発行できる。

### 6.2 Leaseとwrite validation

```text
RuntimeComponentLeaseV1
  lease_kind: read | write
  world_epoch: positive uint64
  phase_ref: RuntimeTimeRefV1(kind=simulation, phase_id=non-null)
  query_dispatch_work_id: bytes32
  selection_mask_ref: RuntimeQuerySelectionMaskRefV1
  component_schema_ref: ComponentSchemaRefV1
  row_begin: uint32
  row_end_exclusive: uint32
  lease_token: opaque bytes32
```

write leaseはmanifestの`write_component_refs`、callbackの`RuntimeQueryDispatchPlanV1` work unit、selection mask、phase、World epochのintersectionでだけ有効である。generated accessorは対象rowがrange内かつselection maskでselectedであることを毎回検査する。別row、unselected row、別chunk、別Component、expired leaseへのwriteを拒否する。

dirty row rangeをcallbackの自己申告に委ねない。value writeの検証根拠はwork unitとimmutable selection maskであり、必要なchange epochはpayload外metadataへOwnerが記録する。callbackはstructural presence、chunk header、location table、query cacheを直接変更しない。

parallel dispatchの競合判定はSystem名、worker index、chunkだけから推測せず、`{world_epoch, phase_ref, component_schema_ref, selection_mask_ref, lease_kind}`の組で行う。同じWorld epoch／phase／Componentについてactual selected rowのintersectionが空なら並行可能、空でなければ`read+read`だけを並行可能とし、`read+write`と`write+write`はSchedulerが順序edgeを持たない限りdispatch前に拒否する。archetypeまたはchunkが異なることは十分条件になり得るが、optional mask、predicate mask、row rangeを無視した自己申告を非重複証明に使わない。

leaseは一つのcallback invocationとlogical work IDへ束縛し、別callback、別phase、別World epochへtransferしない。Schedulerがcallbackを実行するphysical threadを変更できるのはlease発行前だけであり、発行済みleaseを別thread、async continuation、nested callback、job packetへ渡すことを禁止する。

read／write lease、span、reference、pointerをmember、event、job packet、async requestへ保存しない。Job packetはowned value、immutable snapshot、typed handle、cancellation token、job専用memoryだけを持てる。

### 6.3 System bindingとstate store

```text
RuntimeSystemBindingAndStateStoreSetV1
  set_version: 1
  system_bindings[0..4096]:
    system_ref
    component_access_manifest_ref
    query_dispatch_contract_refs[]
    implementation_ref
    owner_ref
  state_store_bindings[0..4096]:
    state_store_ref
    state_class: authoritative | derived | presentation
    active_owner_ref
    read_consumer_refs[]
    write_manifest_ref: optional RuntimeComponentAccessManifestRefV1
  set_hash: SHA-256
```

authoritative state storeのactive ownerはexact一つである。derived stateはauthoritative inputから再構築し、presentation stateをauthoritative Gameplayへ逆writeしない。Schedulerはphase、DAG、callback lifetimeを所有し、ECSはbindingが指定するstate accessの正当性を検証する。

### 6.4 Observability boundary

regular value writeはSchedulerが管理するcallback／phase scope内だけで観測できる。external consumer、Renderer、Audio、VFX、Debug transport、AI、Save projectorはlive write leaseやpartial advanceを読むことができない。これらはboundaryでsealされたimmutable snapshot、publication、またはOwnerが生成したfield-projected recordを読む。

structural transactionのcommit前にlive location table、new Entity handle、archetype membership、Component presenceを外部へ公開しない。value writeとstructural publicationを同じ可視化イベントとして扱わず、advance failure時はそのadvanceのWorld snapshotをpublishしない。

## 7. Structural transaction

### 7.1 Lifecycle delta

```text
RuntimeComponentLifecycleDeltaV1
  delta_version: 1
  producer_system_ref: GameSystemContractRefV1
  producer_phase_ref: RuntimeTimeRefV1(kind=simulation, phase_id=non-null)
  producer_logical_work_id: bytes32
  local_sequence: positive uint32
  target: RuntimeEntityHandle | EntityCreateToken
  kind: create_entity | destroy_entity | add_component | remove_component
        | set_entity_enabled | set_component_enabled
  expected_world_epoch: positive uint64
  expected_component_presence_refs[0..1024]:
    sorted unique ComponentSchemaRefV1
  target_component_schema_ref: optional ComponentSchemaRefV1
  enabled_value: optional bool
  transition_ref: optional ComponentTransitionRefV1
  template_ref: optional EntityTemplateRefV1
  payload_ref: optional CanonicalValueRefV1
  apply_boundary_ref: RuntimeStructuralCommitBoundaryRefV1
  content_hash: SHA-256
```

deltaはcallback中にprivate batchへ追加するだけであり、live Worldを変更しない。`EntityCreateToken`は同じprivate batch内で、より小さい`local_sequence`のcreate結果だけを参照できる。tokenを別batch、別work item、別System、次boundaryへ保存または渡すことを拒否する。

`kind`ごとのField形状は次のclosed unionである。`required`以外のbranch固有Fieldは`absent`でなければならず、unknown Fieldまたは複数branchの混在を拒否する。

| kind | target | required | conditional |
|---|---|---|---|
| `create_entity` | このdeltaが宣言する新しい`EntityCreateToken` | `template_ref` | templateがPersistent Entity Identity policyを持つか、明示的にephemeralである |
| `destroy_entity` | `RuntimeEntityHandle`または同batchの先行`EntityCreateToken` | なし | `expected_component_presence_refs`だけをpreconditionに使用できる |
| `add_component` | existing handleまたは先行token | `target_component_schema_ref`、`transition_ref` | `inline_value`／`external_handle`でcanonical defaultがなければ`payload_ref`必須、`tag`では禁止 |
| `remove_component` | existing handleまたは先行token | `target_component_schema_ref`、`transition_ref` | `expected_component_presence_refs`に同Componentを含める |
| `set_entity_enabled` | existing handleまたは先行token | `enabled_value` | Component、transition、template、payload Fieldは全て禁止 |
| `set_component_enabled` | existing handleまたは先行token | `target_component_schema_ref`、`enabled_value` | 対象Schemaがenablementを許可しなければ拒否する |

runtime createはtemplateのinitializer、lifetime、persistent identity policyをそのまま消費する。call siteが任意Persistent IDをpayloadへ埋め込み、ephemeral templateをSave対象へ昇格し、commit前tokenをAI／Save／Eventへ公開することを禁止する。`content_hash`はbranch validation後のrecordについて、`SHA-256(ASCII "MIRAKAN_RUNTIME_COMPONENT_LIFECYCLE_DELTA_V1" || uint32_be(len(canonical delta bytes excluding content_hash)) || canonical delta bytes excluding content_hash)`で計算する。

### 7.2 Atomic commit

boundaryはdeltaを`{advance_sequence, producer_phase_id, producer_system_id, producer_logical_work_id, local_sequence}`でcanonical mergeする。commit前にtarget handle／generation、precondition、owner permission、transition、destination capacity、identity collision、budget、dependencyを全件検査し、staging mutation planと必要chunk reservationを構築する。

検査済みdeltaが全て成功した時だけ、chunk row、location table、archetype membership、enablementを一commit pointでprivate working Worldへ反映し、World epochをexact 1増加させる。一件でも失敗した場合はstagingを破棄し、previous World、location table、query cache、output packet、publication hashを変更しない。partial commit、callback中のdirect mutation、vendor callbackからのWorld mutation、completion順によるmergeを禁止する。成功時は同じepoch ruleに従い、影響するaddress、row、span、lease、query membershipをinvalidateする。

structural commitは次のadvanceの`T00_BoundaryApply`でprivate working Worldへ適用する内部commitであり、それ自体を外部publicationにしない。同advanceのauthoritative phaseが後でfaultした場合、変更済みworking World、value-write journal、unsealed outputを外部へpublishせず、別advanceの開始状態として再利用しない。SchedulerはPlay session faultまたはlast published checkpointからの明示recoveryを選び、ECSが完了済みphaseを暗黙rollbackしたと報告しない。外部consumerに対するauthorityは最後に成功した`T110_Publish`のまま維持する。

Stable identityはtyped generation handleだけが持つ。chunk ID、row、address、pool slotをidentity、Save ref、AI targetとして扱わない。Shipping pathにAoS、sparse-set、object-graph、old reader／writer fallbackを残さない。

## 8. ECS AI contract graph

AIにECSの構造を説明するため、OwnerがsealしたContract graphだけを使う。Graphは実行surfaceではなく、Operation、Tool、write権限、live World accessを追加しない。

```text
RuntimeEcsContractGraphV1
  graph_version: 1
  world_publication_ref: RuntimeWorldPublicationRefV1
  construction_set_ref: RuntimeEntityConstructionSetRefV1
  system_binding_set_ref: RuntimeSystemBindingAndStateStoreSetRefV1
  nodes[0..65536]:
    node_id
    node_kind: world | archetype | component | query | system | state_store
               | structural_boundary | package | persistence_projection
    owner_ref
    status: current | target_review | not_activated
    sensitivity
    ai_exposure_policy_ref
  edges[0..262144]:
    from_node_id
    relation: contains | queries | reads | writes | stages | publishes
              | projects_to | depends_on
    to_node_id
  graph_hash: SHA-256
```

### 8.1 ECS immutable recordとRef

以下のRefはOwnerがsealした同名recordのversionと自己hashへexactに解決する。`RuntimeWorldPublicationRefV1`はcross-sessionにも渡せるprovenance recordであり、process-localな`RuntimeWorldInstanceHandle`を含めない。

```text
RuntimeEntityConstructionSetRefV1
  construction_set_version: positive uint32
  construction_set_hash: SHA-256

RuntimeArchetypeLayoutPlanRefV1
  layout_algorithm_id: ecs_chunk_soa_v1
  archetype_id: StableArchetypeId
  column_layout_hash: SHA-256

RuntimeWorldBuildGatewayRefV1
  gateway_version: positive uint32
  gateway_hash: SHA-256

RuntimeQuerySelectionMaskRefV1
  world_epoch: positive uint64
  archetype_id: StableArchetypeId
  chunk_id: RuntimeChunkId
  selection_hash: SHA-256

RuntimeComponentAccessManifestRefV1
  manifest_version: positive uint32
  manifest_hash: SHA-256

RuntimeSystemBindingAndStateStoreSetRefV1
  set_version: positive uint32
  set_hash: SHA-256

RuntimeStructuralCommitBoundaryRefV1
  simulation_advance_interval_ref: SimulationAdvanceIntervalRefV1
  seal_phase_id: TickPhaseId (must T110_Publish)
  apply_phase_id: TickPhaseId (must T00_BoundaryApply)
  requested_apply_advance_sequence: positive uint64
  expected_source_world_publication_generation: uint64
  boundary_hash: SHA-256

RuntimeWorldPublicationV1
  publication_version: 1
  world_root_id: StableId
  runtime_package_ref: RuntimePackageRefV1
  contract_set_ref: ContractSetRefV1
  target_profile_ref: TargetProfileRefV1
  world_epoch: positive uint64
  world_publication_generation: positive uint64
  sealed_structural_commit_hash: SHA-256
  publication_hash: SHA-256

RuntimeWorldPublicationRefV1
  publication_version: positive uint32
  world_root_id: StableId
  runtime_package_ref: RuntimePackageRefV1
  contract_set_ref: ContractSetRefV1
  target_profile_ref: TargetProfileRefV1
  world_epoch: positive uint64
  world_publication_generation: positive uint64
  publication_hash: SHA-256
```

advance `N`のcallbackが生成したstructural batchは`N`の`T110_Publish`までにsealし、`requested_apply_advance_sequence=N+1`の`T00_BoundaryApply`でだけ適用する。T110はbatchのsealと成功snapshotの外部publicationを行うが、次回batchをlive Worldへ適用しない。T00はexpected source publication generationを再検査し、batch全体を一回だけprivate working Worldへcommitする。同じboundary hashの再送は同じ結果を返し、同じlogical boundaryに別hash、過去／飛越しadvance、wrong seal／apply phaseを指定したrecordを拒否する。

AI readはrequested field mask、Task Authorization、sensitivity、channel policyのintersectionで決める。route kindは[AI Security／Approval](../01-governance/ai-security-approval.md)の`engine_provider_adapter | standard_external_mcp | managed_external_host`だけを使う。ECSはroute alias、MCP固有grant、provider credentialを定義しない。

captureはseal済みpublication、bounded node／edge／byte上限、omitted range、continuation、redacted field reasonを持つ。`secret`、user text、credential、raw path、native pointer、live RuntimeEntityHandleをcaptureへ入れない。redactionによりownerまたはfailureを判定できない場合、AIは`insufficient_authorized_context`を返し、完全な説明や変更提案として扱わない。

### 8.2 ECS optimization explanation boundary

ECSのlayout／query／callback最適化は、`RuntimeEcsContractGraphV1`から候補状態を推測せず、[Performance／Capacity §8.4](performance-capacity.md#84-algorithm-optimization-candidate-qualification)の完成`OptimizationDecisionProjectionV1`を別のread-only bindingとして消費する。8／16／32 KiB layout、hot／cold split、query-cache実装、generated zero-allocation callbackは、Source revision、Target Profile、Contract Set、Toolchain lock、fixture、input trace、algorithm revision、implementation variantのいずれかが違えば別Candidateである。Graph node名、chunk payload値、benchmark labelから同一Candidateまたはselected状態を合成しない。

[AI Security／Approval §5](../01-governance/ai-security-approval.md#5-beginner-questionsassumptions理解条件)の`AiTaskContextCapsuleV1`へECS graphとoptimization decisionを同時に束縛する場合、両者のProject lineage／source revision／Target／Contractをexact一致させ、Projectionはcompleteかつfreshでなければならない。missing／stale／revoked Receipt、候補hash差、redactionによりselected／rejected理由を確定できない場合は`insufficient_authorized_context`とし、Graph、raw trace、設計文書、16 KiBのtarget design valueから理由を補完しない。これは説明契約だけであり、ECS Operation、candidate selection、Runtime dispatch、Capability Activation、write権限を追加しない。

## 9. 他Ownerとの境界

| Owner | ECSが渡すもの | ECSが所有しないもの |
|---|---|---|
| Scheduling／Lifetime | manifestに適合するlease・boundary要件 | phase、job DAG、worker lifetime、merge orchestration |
| Runtime Package | build gatewayに渡すconstruction set ref | World Root／Section image、package binary、loader |
| Persistence／Save | identityとcanonical field projection ref | Save record、reconstruction、replay transport、migration |
| Debugging／Observability／Replay | sealed snapshot／graph projection ref | capture transport、debug UI、trace storage |
| Asset Lifecycle | Componentが参照するtyped artifact ref | generic artifact manifest、catalog、cook、promotion |
| Domain Owner | Component schema／meaning、System semantic owner | ECS storage semantics、lease、structural transaction |
| AI Security／Approval | exposure policy refを消費 | authorization、approval、route grant、credential |

### 9.1 Compatibility consumer mapping

次表はECS boundaryを[Compatibility／Evolution](../02-foundation/compatibility-evolution.md#42-ecs-consumer-inventory-boundary)のconsumer classへ漏れなく分類するためのmappingである。行の存在は実consumer、reader、release、old-reader windowの存在を意味しない。current化前には各行をConsumer Inventoryのrecordまたはscope Requirementのpass satisfaction bindingへ解決し、`unresolved`を残さない。

| ECS境界 | consumer class | 許可される入力 |
|---|---|---|
| Project Source／Asset import／approved Runtime entry | `authoring_source`／`derived_artifact` | committed Sourceとtarget Toolchain lockだけ |
| World Root／Section image／Runtime Package | `runtime_package` | immutable World image、Catalog、target Contract。old Package bytesは不可 |
| Save／Replay projector | `save`／`replay` | sealed publicationとpersistent projection。raw handleは不可 |
| Native Game Module／C ABI | `native_abi` | generated bounded view、typed ref、owned buffer。live leaseは不可 |
| Renderer／Audio／VFX | `runtime_projection` | sealed snapshotまたはpresentation projectionだけ |
| Debug transport／AI capture | `observability_projection` | redacted sealed graph／captureだけ |
| 外部API／配布先／documentation projection | `external_api`／`documentation` | approved current projectionだけ |

### 9.2 Runtime ECS Diagnostic

```text
diagnostic.runtime.ecs-component-schema-revision-required
MIRAKAN-RUNTIME-ECS-COMPONENT-SCHEMA-REVISION-REQUIRED
arguments = component_schema_ref, access_cohort_hash
result = component_schema_revision_required; no silent split

diagnostic.runtime.ecs-chunk-capacity-zero
MIRAKAN-RUNTIME-ECS-CHUNK-CAPACITY-ZERO
arguments = archetype_id, column_layout_hash, payload_bytes
result = Cook／Contract compile failure

diagnostic.runtime.ecs-archetype-permutation-unbounded
MIRAKAN-RUNTIME-ECS-ARCHETYPE-PERMUTATION-UNBOUNDED
arguments = component_schema_ref, observed_count, declared_bound
result = Layout qualification failure

diagnostic.runtime.ecs-query-cache-invalid
MIRAKAN-RUNTIME-ECS-QUERY-CACHE-INVALID
arguments = query_ref, invalid_member_kind
result = Contract／negative fixture failure

diagnostic.runtime.ecs-structural-mutation-during-iteration
MIRAKAN-RUNTIME-ECS-STRUCTURAL-MUTATION-DURING-ITERATION
arguments = system_ref, operation_kind, logical_work_id
result = Typed rejection

diagnostic.runtime.ecs-structural-capacity-exceeded
MIRAKAN-RUNTIME-ECS-STRUCTURAL-CAPACITY-EXCEEDED
arguments = boundary_ref, required_bytes, available_bytes
result = Previous World remains published

diagnostic.runtime.ecs-entity-reference-stale
MIRAKAN-RUNTIME-ECS-ENTITY-REFERENCE-STALE
arguments = entity_ref, expected_world_epoch,
            actual_world_epoch, expected_publication_generation,
            actual_publication_generation
result = Typed rejection; no implicit rebase

diagnostic.runtime.ecs-lease-scope-violation
MIRAKAN-RUNTIME-ECS-LEASE-SCOPE-VIOLATION
arguments = system_ref, logical_work_id, component_schema_ref,
            violation_kind
result = Callback fault; no output publication

diagnostic.runtime.ecs-access-conflict
MIRAKAN-RUNTIME-ECS-ACCESS-CONFLICT
arguments = phase_ref, component_schema_ref,
            first_work_id, second_work_id, overlap_hash
result = Dispatch rejection before either callback starts

diagnostic.runtime.ecs-structural-delta-shape-invalid
MIRAKAN-RUNTIME-ECS-STRUCTURAL-DELTA-SHAPE-INVALID
arguments = producer_system_ref, kind, invalid_field_set_hash
result = Private batch rejection

diagnostic.runtime.ecs-structural-commit-conflict
MIRAKAN-RUNTIME-ECS-STRUCTURAL-COMMIT-CONFLICT
arguments = boundary_ref, first_delta_hash, second_delta_hash,
            conflict_kind
result = Entire structural batch rejected; previous publication retained

diagnostic.runtime.ecs-world-generation-mismatch
MIRAKAN-RUNTIME-ECS-WORLD-GENERATION-MISMATCH
arguments = boundary_ref, expected_world_epoch,
            actual_world_epoch, expected_publication_generation,
            actual_publication_generation
result = Boundary apply rejection

diagnostic.runtime.ecs-publication-hash-mismatch
MIRAKAN-RUNTIME-ECS-PUBLICATION-HASH-MISMATCH
arguments = publication_ref, expected_hash, actual_hash
result = Publication rejected; previous publication retained

diagnostic.runtime.ecs-persistent-identity-policy-required
MIRAKAN-RUNTIME-ECS-PERSISTENT-IDENTITY-POLICY-REQUIRED
arguments = template_ref, requested_projection_kind
result = Create or persistence projection rejected

diagnostic.product.ecs-data-oriented-core-failed
MIRAKAN-PRODUCT-ECS-DATA-ORIENTED-CORE-FAILED
arguments = requirement_id, campaign_hash, target_profile_ref,
            failed_diagnostic_refs[1..64]
result = aggregate Requirement evaluation fails
```

Product aggregateはRuntime個別failureを置き換えず、Requirement評価のclosureだけを表す。本書は[Performance／Capacity](performance-capacity.md)の`RuntimeDataOrientedQualificationProfileV1`／`RuntimeDataOrientedQualificationCampaignV1`、`fixture.runtime.ecs-data-oriented-core`、六scenario、complete mandatory metric set、hard predicate、8／16／32 KiB characterizationを参照し、thresholdまたはcampaign schemaを複写しない。

## 10. Qualification

target ECSは少なくとも次を証明する。

1. 同一construction inputから同じarchetype layout、query dispatch plan、structural merge、publication hashを二回生成する。
2. stale handle、generation wrap、cross-World ref、unselected row write、phase外lease、unmanifested accessをinvalid fixtureで拒否する。
3. structural command failureがlive World、location table、publicationを変更しない。
4. `canonical_range`と`deterministic_hash`でcallback unitがchunk境界を越えず、worker数や完了順でresultが変わらない。
5. value writeがseal前snapshot／AI／Save／Presentationへ露出せず、faulted advanceがpublicationされない。
6. package、Save、Replay、Debug、Native moduleとのowner境界がraw handle、raw pointer、旧aliasを渡さない。
7. 8／16／32 KiB比較、capacity、fragmentation、structural burst、query cache invalidationをPerformance evidenceで検証する。
8. Consumer Inventoryの全required scopeがcomplete、unknown consumerがzero verified、全Evidence Requirementがpass fulfillment済みであり、Compatibility Change、Owner reference migration manifest、source／target Definition Closure、Definition Migration bindingが同一closureへexact解決する。
9. hot callbackのgeneral-heap allocation countとupstream fallback countが両方exact 0で、query scratch、selection mask、command buffer、output packetがdispatch前に予約される。
10. `RuntimeComponentLayoutPolicyV1`のaccess cohortがactive System manifest、query、phase、read／write setから再導出でき、manual hot／cold labelまたはautomatic field splitを含まない。
11. T110でsealしたbatchが指定された次のT00でexactly once applyされ、T110でlive mutationされず、wrong generation／wrong phase／別hash再送を拒否する。
12. `world_epoch`と`world_publication_generation`が定義したイベントだけで増加し、faulted advanceのworking Worldが次advance、AI、Save、Replay、Presentationへ露出しない。
13. `GameSystemSpecV2`、ECS access manifest、Native descriptorのread Component、write Component、State read／write、structural permission集合が正逆方向に一致する。
14. 本書が参照する全Type／Ref、collection bound、branch constraint、canonical encoding、hash golden vectorがDefinition Closureへexact解決し、unresolved shorthandが0件である。

## 11. 外部一次資料から採る原則

[Unity Entitiesのsync point資料](https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/performance-sync-points.html)はstructural changeを予測可能なpointへまとめるためのCommand Buffer利用を説明する。[Flecs Queries資料](https://github.com/SanderMertens/flecs/blob/v4.1.2/docs/Queries.md)は反復利用queryのcacheとiteration中table mutationのdeferを説明する。[EnTT Entity資料](https://github.com/skypjack/entt/blob/v3.16.0/docs/md/entity.md)はiteration中の変更でreferenceが無効化され得ることを説明する。

これらは比較用の一次資料であり、MiraikanaiのAPI、chunk size、Schema、scheduler、binary formatを規定しない。特に16 KiB、64-byte alignment、256-byte上限はMiraikanaiのtarget Contractであり、vendorの推奨値として扱わない。
