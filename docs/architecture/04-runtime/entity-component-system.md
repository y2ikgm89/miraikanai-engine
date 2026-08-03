# Miraikanai Engine Runtime Entity Component System Contract

- 文書ID: mirakan.arch.runtime-entity-component-system
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Runtime Entity／Component identity、initializer・template・archetype／SoA layout、query・selection・dispatch・cache、Component access manifest、lease、structural transaction、runtime state binding、ECS AI contract graph、ECS固有diagnostic・qualification
- 非正本範囲: Runtime phase／job DAG、network connection／session／Network Object／authority／replication、generic artifact envelope／catalog、World package binary、Save／Replay payload、debug transport、AI authorization・route grant、各Domain Componentの意味・field。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Core Architecture](../02-foundation/core-architecture.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Scheduling／Lifetime](scheduling-lifetime.md)
- 関連文書: [Runtime ECS Static Definition／Entity Reference Boundary Decision](../decisions/2026-08-03-runtime-ecs-static-and-entity-reference-boundary.md)、[Architecture Governance](../01-governance/architecture-governance.md)、[Advanced Rendering／Multiplayer Ownership Decision](../decisions/2026-07-29-advanced-rendering-multiplayer-ownership.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Gameplay programming model](../03-authoring/gameplay-programming-model.md)、[Native Game Module](../03-authoring/native-game-module.md)、[Scheduling／Lifetime](scheduling-lifetime.md)、[Runtime Package](runtime-package.md)、[Persistence／Save](persistence-save.md)、[Debugging／Observability／Replay](debugging-observability-replay.md)、[Performance／Capacity](performance-capacity.md)、[Multiplayer Authority／Replication](../09-networking/multiplayer-authority-replication.md)、[Runtime ECS Design Closure Review](../appendices/runtime-ecs-design-closure-review.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-24

## 1. 状態と結論

本書はEngine-owned archetype／SoA Runtime ECSの目標正本である。比較対象EngineのAPI、型、保存形式、scheduler、World semanticsを採用しない。採るのは、archetypeにより同じComponent集合を列指向に配置すること、反復利用するqueryをcacheできること、iteration中のstructural mutationをdeferすること、世代付きhandleと宣言的accessを使うことだけである。ECSはEntity／Component固有のhandle、query lease、structural invalidationを所有し、一般pointer taxonomy、leaseの保存／capture禁止、memory resourceは[Memory／Pointers](../02-foundation/memory-pointers.md)のbindingを消費する。ECS独自のwrapperでこれらを再定義しない。

[Multiplayer Authority／Replication](../09-networking/multiplayer-authority-replication.md)は`NetworkObjectIdentityV1`とexact `RuntimeEntityGenerationRefV1`のbinding、replication Schema、authority、spawn／despawn／state commandを所有する。本書はEntity／Component generation、Schema、access、Structural Transactionを所有し、Network Object ID、connection、participant、authority lease、baselineをECS identityまたはstorage keyにしない。受信commandはScheduling boundaryで検証済みowner-typed requestとしてだけ適用し、network callbackまたはsenderがECS writer authorityを得ない。

本書は`owner.core.runtime_ecs`のinitial V1 Architecture Ownerである。Entity／Componentのruntime identity、storage、query、lease、structural transactionは本書だけが正本を持ち、[Gameplay programming model](../03-authoring/gameplay-programming-model.md)はGame Systemのauthoring semanticsとECS access projectionだけを所有する。旧Owner revision、移管元、alias、dual Registryまたはmigration readerをinitial V1へ定義しない。

本書の`review`状態と型定義は、ECS runtime、MCD、Operation、Tool、Native ABI、Package reader、Save readerが既にmaterializeまたはactiveであることを意味しない。Definition Closure、実装、Qualification、Capability activationはそれぞれArchitecture Governanceの独立した状態軸であり、文書承認だけから昇格させない。

### 1.1 Target contract closure

本書はtarget semanticsに加え、本文が使うECS固有Type／Ref、collection bound、canonical sort、uniqueness、branch制約、canonical encoding、self-excluding hash preimageを§1.3で閉じる。`CanonicalValueRefV1`、`ComponentSchemaRefV1`、`ComponentTransitionRefV1`、`EntityCreateToken`、`EntityTemplateRefV1`、`ImmutableBitsetRefV1`、`InitializerSpecRefV1`、`RuntimeChunkId`、`StableArchetypeId`を未解決のplaceholderとして残さない。

ただし、closed target designはmachine-readable Definition Closureまたは実装ではない。対応MCD、Schema、Registry、Fixture instance、golden vector、Contract compiler output、ReceiptはRepositoryに存在しない。§1.3の全recordがmaterializeし、全Refが解決し、§10のFixture contractとgolden hashが生成・検証されるまで、Contract compilerはRuntime ECS Definition Closureを生成せず、Phase Gate、Capability Activation、Native load、Package load、Save／Replay reader、AI complete explanationを成功にしない。監査状態とcross-owner論点は[Runtime ECS Design Closure Review](../appendices/runtime-ecs-design-closure-review.md)で追跡する。同Reviewは本書のsemanticsを上書きしない。

### 1.2 Initial V1 ownership boundary

初期Contract SetではGameplay programming model、Scheduling／Lifetime、Runtime Package、Persistence／Save、Debugging／Observability／Replay、Native Game Module、AI Tool／Explain projection、各Domain Component Ownerが、本書のexact Owner／Definition refを直接参照する。Gameplay authoringからstorage semanticsを推測する、Review文書の型名をSchemaとして読む、同一identityを複数Ownerへ登録する、またはunresolved refを近似名へfallbackすることを禁止する。

Runtime ECSのArchitecture承認、Definition Closure、実装、Qualification、Capability activationは別々に判定する。本書のArchitecture状態だけでRPG Genre Pack、Gameplay Feature、Native API、Package reader、Save／Replay readerまたはAI Capabilityをactiveとみなさない。initial V1が公開またはmaterializeされた後の変更に限り、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)のconsumer inventoryと承認済みCompatibility Changeを適用する。

### 1.3 ECS primitive、immutable record、exact Ref

本節のrecordはclosed canonical MCD valueである。文字列は明記したASCII token、整数はunsigned big-endian、配列は明記したkeyのcanonical byte順でstrict sorted unique、optionalは明記したFieldだけに許可する。各`*_content_hash`は`SHA-256(ASCII domain separator || uint32_be(payload_length) || canonical payload excluding self hash)`で計算する。

```text
ComponentSchemaV1
  component_type_id: StableId
  component_schema_version: positive u32
  semantic_owner_ref: exact OwnerRefV1
  mcd_type_ref: exact McdContractRefV1(kind=type)
  storage_class:
    inline_value | tag | external_handle | derived_index
  canonical_inline_bytes: u32 in [0,256]
  canonical_inline_alignment_bytes: one of 1 | 2 | 4 | 8 | 16 | 32 | 64
  enablement_allowed: bool
  canonical_default:
    {kind=absent}
    | {kind=canonical_zero}
    | {
        kind=owner_defined,
        canonical_bytes_artifact_ref: exact ArtifactRefV1,
        canonical_bytes_content_hash: SHA-256
      }
  component_schema_content_hash: SHA-256

CanonicalValueV1
  canonical_value_id: StableId
  canonical_value_version: 1
  component_schema_ref: exact ComponentSchemaRefV1
  canonical_value_bytes_artifact_ref: exact ArtifactRefV1
  canonical_value_byte_count: u32 in [0,1048576]
  canonical_value_content_hash: SHA-256

StableArchetypeId
  archetype_id_version: 1
  contract_set_ref: exact ContractSetRefV1
  component_schema_refs[1..8192]:
    sorted unique exact ComponentSchemaRefV1
  component_set_hash: SHA-256

ComponentTransitionV1
  transition_id: StableId
  transition_version: 1
  semantic_owner_ref: exact OwnerRefV1
  operation_kind: add_component | remove_component
  source_archetype_id: exact StableArchetypeId
  destination_archetype_id: exact StableArchetypeId
  target_component_schema_ref: exact ComponentSchemaRefV1
  transition_content_hash: SHA-256

ImmutableBitsetV1
  bitset_id: StableId
  bitset_version: 1
  bit_count: u32 in [0,1048576]
  words[0..16384]: ordered uint64
  lifetime_scope:
    {
      kind: query_callback,
      world_epoch: positive u64,
      logical_work_id: bytes32
    }
    | {
        kind: sealed_publication,
        world_publication_ref: exact RuntimeWorldPublicationRefV1
      }
  bitset_content_hash: SHA-256

InitializerSpecV1
  initializer_spec_id: StableId
  initializer_spec_version: 1
  semantic_owner_ref: exact OwnerRefV1
  initial_component_values[0..8192]:
    sorted unique {
      component_schema_ref: exact ComponentSchemaRefV1,
      value_ref: exact CanonicalValueRefV1
    }
  persistent_identity_disposition:
    ephemeral | authoring_bound | runtime_spawn_required
  persistent_identity_policy_ref:
    null | exact McdContractRefV1(kind=policy)
  lifetime_policy_ref: exact McdContractRefV1(kind=policy)
  initializer_spec_content_hash: SHA-256

EntityTemplateV1
  entity_template_id: StableId
  entity_template_version: 1
  semantic_owner_ref: exact OwnerRefV1
  component_schema_refs[1..8192]:
    sorted unique exact ComponentSchemaRefV1
  initializer_spec_refs[1..8192]:
    sorted unique exact InitializerSpecRefV1
  initial_archetype_id: exact StableArchetypeId
  entity_template_content_hash: SHA-256

EntityCreateToken
  token_version: 1
  producer_system_ref: exact GameSystemContractRefV1
  producer_phase_ref:
    exact RuntimeTimeRefV1(kind=simulation, phase_id=non-null)
  producer_logical_work_id: bytes32
  local_sequence: positive u32
  apply_boundary_ref: exact RuntimeStructuralCommitBoundaryRefV1
  declaration_key_hash: SHA-256
  token_hash: SHA-256

RuntimeChunkId
  chunk_id_version: 1
  world_instance_handle: RuntimeWorldInstanceHandle
  world_epoch: positive u64
  archetype_id: exact StableArchetypeId
  chunk_ordinal: u32
  chunk_generation: positive u32
  chunk_identity_hash: SHA-256

RuntimeStateStoreBindingV1
  state_store_binding_id: StableId
  state_store_binding_version: 1
  state_store_ref: exact McdContractRefV1(kind=type)
  state_class: authoritative | derived | presentation
  active_owner_ref: exact OwnerRefV1
  read_consumer_refs[0..4096]:
    sorted unique exact GameSystemContractRefV1
  write_system_ref: null | exact GameSystemContractRefV1
  persistence_policy_ref: exact McdContractRefV1(kind=policy)
  state_store_binding_content_hash: SHA-256

RuntimeStructuralPermissionV1
  permission_kind:
    create_entity | destroy_entity | add_component | remove_component
    | set_entity_enabled | set_component_enabled
  target_component_schema_refs[0..1024]:
    sorted unique exact ComponentSchemaRefV1
  transition_refs[0..2048]:
    sorted unique exact ComponentTransitionRefV1
  creatable_template_refs[0..1024]:
    sorted unique exact EntityTemplateRefV1
  apply_phase_id: exact TickPhaseId = T00_BoundaryApply
  maximum_delta_count_per_advance: positive u32
  maximum_payload_bytes_per_advance: positive u64
  lifetime_owner_ref: exact OwnerRefV1

RuntimePersistentSpawnIdentityRequestV1
  persistent_spawn_request_id: StableId
  persistent_spawn_request_version: 1
  world_root_id: StableId
  entity_template_ref: exact EntityTemplateRefV1
  producer_system_ref: exact GameSystemContractRefV1
  producer_phase_ref:
    exact RuntimeTimeRefV1(kind=simulation, phase_id=non-null)
  producer_logical_work_id: bytes32
  local_sequence: positive u32
  apply_boundary_ref: exact RuntimeStructuralCommitBoundaryRefV1
  identity_kind: runtime_spawn
  identity_owner_ref: exact OwnerRefV1
  idempotency_key: bytes32
  persistent_spawn_request_content_hash: SHA-256

RuntimeStructuralCommitBoundaryV1
  simulation_advance_interval_ref:
    exact SimulationAdvanceIntervalRefV1
  seal_phase_id: TickPhaseId (must T110_Publish)
  apply_phase_id: TickPhaseId (must T00_BoundaryApply)
  requested_apply_advance_sequence: positive u64
  expected_source_world_publication_generation: u64
  boundary_hash: SHA-256
```

| 型／Ref | exact rule |
|---|---|
| `ComponentSchemaRefV1` | `{component_type_id, component_schema_version, component_schema_content_hash}` |
| `CanonicalValueRefV1` | `{canonical_value_id, canonical_value_version=1, canonical_value_content_hash}` |
| `ComponentTransitionRefV1` | `{transition_id, transition_version=1, transition_content_hash}` |
| `ImmutableBitsetRefV1` | `{bitset_id, bitset_version=1, bit_count, bitset_content_hash}` |
| `InitializerSpecRefV1` | `{initializer_spec_id, initializer_spec_version=1, initializer_spec_content_hash}` |
| `EntityTemplateRefV1` | `{entity_template_id, entity_template_version=1, entity_template_content_hash}` |
| `RuntimeStateStoreBindingRefV1` | `{state_store_binding_id, state_store_binding_version=1, state_store_binding_content_hash}` |
| `RuntimePersistentSpawnIdentityRequestRefV1` | `{persistent_spawn_request_id, persistent_spawn_request_version=1, persistent_spawn_request_content_hash}` |
| `RuntimeComponentLayoutPolicyRefV1` | `{policy_version=1, component_schema_ref, qualification_profile_ref, policy_hash}` |
| `RuntimeEcsDefinitionFixtureContractRefV1` | `{fixture_contract_id, fixture_contract_version=1, fixture_contract_content_hash}` |

Domain separatorは順に`MIRAKAN_COMPONENT_SCHEMA_V1`、`MIRAKAN_CANONICAL_VALUE_V1`、`MIRAKAN_STABLE_ARCHETYPE_ID_V1`、`MIRAKAN_COMPONENT_TRANSITION_V1`、`MIRAKAN_IMMUTABLE_BITSET_V1`、`MIRAKAN_INITIALIZER_SPEC_V1`、`MIRAKAN_ENTITY_TEMPLATE_V1`、`MIRAKAN_ENTITY_CREATE_TOKEN_V1`、`MIRAKAN_RUNTIME_CHUNK_ID_V1`、`MIRAKAN_RUNTIME_STATE_STORE_BINDING_V1`、`MIRAKAN_RUNTIME_PERSISTENT_SPAWN_IDENTITY_REQUEST_V1`、`MIRAKAN_RUNTIME_STRUCTURAL_COMMIT_BOUNDARY_V1`である。

本文全域で宣言するtop-level ECS Type／Refは次のclosed inventoryである。inventory member名と本文宣言名をset equalityにし、同名に見えるnested row、external Owner型、Diagnostic引数またはprose tokenを追加Typeへ数えない。

| classification | exact top-level Type／Ref |
|---|---|
| `persisted_content_addressed` | `ComponentSchemaV1`; `ComponentSchemaRefV1`; `CanonicalValueV1`; `CanonicalValueRefV1`; `StableArchetypeId`; `ComponentTransitionV1`; `ComponentTransitionRefV1`; `InitializerSpecV1`; `InitializerSpecRefV1`; `EntityTemplateV1`; `EntityTemplateRefV1`; `RuntimeStateStoreBindingV1`; `RuntimeStateStoreBindingRefV1`; `RuntimePersistentSpawnIdentityRequestV1`; `RuntimePersistentSpawnIdentityRequestRefV1`; `RuntimeComponentLayoutPolicyV1`; `RuntimeComponentLayoutPolicyRefV1`; `RuntimeEntityConstructionSetV1`; `RuntimeEntityConstructionSetRefV1`; `RuntimeArchetypeLayoutPlanV1`; `RuntimeArchetypeLayoutPlanRefV1`; `RuntimeWorldBuildGatewayV1`; `RuntimeWorldBuildGatewayRefV1`; `RuntimeQueryV1`; `RuntimeQueryRefV1`; `RuntimeComponentAccessManifestV1`; `RuntimeComponentAccessManifestRefV1`; `RuntimeSystemBindingAndStateStoreSetV1`; `RuntimeSystemBindingAndStateStoreSetRefV1`; `RuntimeStructuralCommitBoundaryV1`; `RuntimeStructuralCommitBoundaryRefV1`; `RuntimeEcsContractGraphV1`; `RuntimeEcsContractGraphRefV1`; `RuntimeWorldPublicationV1`; `RuntimeWorldPublicationRefV1`; `ComponentSchemaEvolutionBindingV1`; `ComponentSchemaEvolutionBindingRefV1`; `RuntimeEcsDefinitionFixtureContractV1`; `RuntimeEcsDefinitionFixtureContractRefV1`; `RuntimeEcsAiReadinessBindingV1`; `RuntimeEcsAiReadinessBindingRefV1`; `RuntimeEcsTargetQualificationBindingV1`; `RuntimeEcsTargetQualificationBindingRefV1` |
| `process_local_ephemeral` | `ImmutableBitsetV1`; `ImmutableBitsetRefV1`; `EntityCreateToken`; `RuntimeChunkId`; `RuntimeEntityHandle`; `RuntimeWorldInstanceHandle`; `RuntimeEntitySnapshotRefV1`; `RuntimeEntityLiveRefV1`; `RuntimeQueryArchetypeCacheV1`; `RuntimeQueryArchetypeCacheRefV1`; `RuntimeQuerySelectionMaskV1`; `RuntimeQuerySelectionMaskRefV1`; `RuntimeQueryDispatchPlanV1`; `RuntimeComponentLeaseV1`; `RuntimeComponentLifecycleDeltaV1` |
| `nested_only_value` | `RuntimeStructuralPermissionV1` |

`persisted_content_addressed` recordは各recordのself hash名にかかわらず、record固有ASCII domain separator、`uint32_be(payload_length)`、自己hashだけを除くclosed canonical payloadでSHA-256を計算する。§1.3以外の固有separatorは`MIRAKAN_RUNTIME_COMPONENT_LAYOUT_POLICY_V1`、`MIRAKAN_RUNTIME_ENTITY_CONSTRUCTION_SET_V1`、`MIRAKAN_RUNTIME_ARCHETYPE_LAYOUT_PLAN_V1`、`MIRAKAN_RUNTIME_WORLD_BUILD_GATEWAY_V1`、`MIRAKAN_RUNTIME_QUERY_V1`、`MIRAKAN_RUNTIME_COMPONENT_ACCESS_MANIFEST_V1`、`MIRAKAN_RUNTIME_SYSTEM_BINDING_AND_STATE_STORE_SET_V1`、`MIRAKAN_RUNTIME_ECS_CONTRACT_GRAPH_V1`、`MIRAKAN_RUNTIME_WORLD_PUBLICATION_V1`、`MIRAKAN_COMPONENT_SCHEMA_EVOLUTION_BINDING_V1`、`MIRAKAN_RUNTIME_ECS_DEFINITION_FIXTURE_CONTRACT_V1`、`MIRAKAN_RUNTIME_ECS_AI_READINESS_BINDING_V1`、`MIRAKAN_RUNTIME_ECS_TARGET_QUALIFICATION_BINDING_V1`である。各Refは同名backing recordのidentity／version／自己hashへexact解決し、`RuntimeStructuralCommitBoundaryRefV1`は上記`RuntimeStructuralCommitBoundaryV1`へ解決する。Refだけ、backing recordだけ、same-name／different-hash、missing external bytesを拒否する。

`process_local_ephemeral`はcanonical in-process validation hashを持てるが、Package、Save、Replay、AI Evidence、cross-session digest、Contract Set memberまたはcontent-addressed artifactとして保存しない。`ImmutableBitsetV1.lifetime_scope=sealed_publication`も同じprocess内でseal済みpublicationにread lifetimeを束縛するだけで、Bitset自身の永続化を許可しない。`nested_only_value`は包含recordのcanonical payloadだけに符号化し、standalone Ref、artifact、Registry rowまたはhash identityを持たない。外部opaque refは`OwnerRefV1`／`ArtifactRefV1`／`McdContractRefV1`をFoundation、`GameSystemContractRefV1`をGameplay Programming Model、`RuntimeTimeRefV1`／`SimulationAdvanceIntervalRefV1`／`TickPhaseId`をScheduling、`RuntimePackageRefV1`／World image refをRuntime Package、`EvidenceRefV1`／`QualificationReceiptRefV1`をVerification Ownerへexact解決し、本書でbacking recordを再定義しない。

`ComponentSchemaV1`はECS storage bindingだけを閉じ、field意味は`semantic_owner_ref`のMCD Ownerが所有する。`tag`／`derived_index`はbytes 0、`inline_value`はbytes 1..256、`external_handle`はOwnerが固定したbounded handle bytesを要求する。`canonical_zero`は全Fieldのcanonical zeroがDomain MCDで有効な場合だけ、`owner_defined`はexact bytesとhashがSchemaのself hashへ一方向に束縛される場合だけ許す。`CanonicalValueV1`からSchemaへは参照してもSchemaからCanonical Valueへ逆参照せず、hash cycleを作らない。`StableArchetypeId.component_set_hash`はContract Set refとsorted Component Ref集合からdomain-separatedに導出し、表示名、template、layoutまたはruntime chunkを入力にしない。

`ComponentTransitionV1`のaddはdestination集合が`source ∪ {target}`、removeはdestination集合が`source − {target}`とset equalityでなければならず、それ以外のComponent差分を同じTransitionへ混ぜない。sourceに既に存在するadd、存在しないremove、別Contract Set、別semantic ownerを拒否する。`EntityTemplateV1`のComponent集合はinitial Archetype集合、initializerのComponent projectionとset equalityで、同じComponentを二つのinitializerで初期化しない。

`ImmutableBitsetV1.words`は`ceil(bit_count/64)`件、各wordのbit 0を最下位bitとして論理index順に並べ、canonical byte encodingでは各`uint64`を`uint64_be`とする。最後のwordのunused high bitは0で、`bit_count=0`ではemptyである。callback scope Bitset、`EntityCreateToken`、`RuntimeChunkId`はprocess-localかつ記載scopeだけで有効で、Save、Replay、Package、AI、cross-session digestへ保存しない。`EntityCreateToken.declaration_key_hash`は`producer_system_ref`、`producer_phase_ref`、`producer_logical_work_id`、`local_sequence`、`apply_boundary_ref`だけからdomain-separatedに導出し、Tokenを格納するDeltaのcontent hashを入力にしない。`token_hash`はToken自己hash Fieldと外側Deltaを除く完成Token payloadから導出し、自己参照hash cycleを作らない。Tokenは同じSystem／work／boundaryでより大きい`local_sequence`からだけ参照できる。Chunk generation wrapはslot retireとし、World epoch変更、structural commit、chunk reuseまたはWorld destroyで旧IDを無効化する。

`RuntimePersistentSpawnIdentityRequestV1`はruntime spawnのpersistent identityをECSからPersistenceへ渡す唯一のrequestである。ECSはID値を発行せず、Persistence Ownerが同request ref、同World root、同Template、同idempotency keyへexactly oneのPersistent Identity Allocation Bindingを返した場合だけpersistent createをpreflight成功にする。ephemeralまたはauthoring-bound initializerでこのrequestを持つこと、runtime-spawn-requiredで欠くこと、call siteがPersistent IDを直接指定することを拒否する。

`RuntimeStructuralPermissionV1`のbranch Fieldは次のmatrixで閉じる。共通Fieldは`apply_phase_id`、二budget、`lifetime_owner_ref`だけで、表の`empty`はexact `[]`を意味する。`apply_phase_id`はScheduling Ownerのserialized `TickPhaseId`であり、advance、interval、publicationまたはwall-clock identityを持たない。unknown Field、表外組合せまたは複数branch混在を包含`RuntimeComponentAccessManifestV1`のcanonical validation前に拒否する。

| `permission_kind` | `target_component_schema_refs` | `transition_refs` | `creatable_template_refs` |
|---|---|---|---|
| `create_entity` | empty | empty | non-empty |
| `destroy_entity` | empty | empty | empty |
| `add_component` | non-empty | non-empty。各transitionは同targetへの`add_component` | empty |
| `remove_component` | non-empty | non-empty。各transitionは同targetへの`remove_component` | empty |
| `set_entity_enabled` | empty | empty | empty |
| `set_component_enabled` | non-emptyかつ全Schemaが`enablement_allowed=true` | empty | empty |

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

RuntimeEntitySnapshotRefV1
  world_instance_handle: RuntimeWorldInstanceHandle
  entity_handle: RuntimeEntityHandle
  world_epoch: positive uint64
  world_publication_generation: positive uint64

RuntimeEntityLiveRefV1
  world_instance_handle: RuntimeWorldInstanceHandle
  entity_handle: RuntimeEntityHandle
```

各handleのall-zero値はinvalidである。slot reuse時はgenerationを増やし、wrapするslotはretireする。`RuntimeEntityHandle`は同一Worldに束縛されたcallback内だけで使える。seal済みpublicationのsnapshot-bound read、debug captureまたは同publication内の照合には`RuntimeEntitySnapshotRefV1`を使う。command、event、async result、debug request、Subsystem PortのようにcallbackまたはSimulation Advanceを越えて同じruntime session内のcurrent Entityを指すcarrierには`RuntimeEntityLiveRefV1`を使う。

いずれのRefもobject、lease、write authorityまたはlivenessを所有せず、Source、Save、Replay、Package、AI Evidence、cross-session digestへ保存しない。永続化、Replayまたは別sessionでの再同定が必要なEntityはPersistence Ownerが定めるPersistent Entity Identityへ投影し、持たないEntityはephemeral ordinalとして明示的に除外する。`RuntimeEntityLiveRefV1`をPersistent Entity Identityの短縮形またはfallbackにしない。

`world_epoch`は一つの`RuntimeWorldInstanceHandle`内にあるprivate working Worldの構造世代である。eligible `T00_BoundaryApply`でsealed structural batchのpreflightとcommitが全件成功した場合だけexact 1増加し、regular value write、T110 snapshot publish、失敗したstructural batchでは増加しない。`world_publication_generation`は外部consumerへ可視なseal済みWorld publicationの世代であり、Simulation Advance全体が成功して`T110_Publish`でpublishされた場合だけexact 1増加する。一つのpublicationは、そのsnapshotが表すexact `world_epoch`を記録する。

初期constructionを全件検証したprivate working Worldは`world_epoch=1`から開始し、最初の成功`T110_Publish`を`world_publication_generation=1`とする。published Refで0を許可しない。初回publication前を表すboundary preconditionだけは`expected_source_world_publication_generation=0`を許可し、その場合もstaged construction set、Contract Set、Target Profileのexact一致を要求する。

`RuntimeEntitySnapshotRefV1`のresolveはWorld instance、World epoch、publication generation、Entity generation、alive stateを全て一致させる。古いpublicationから取得したSnapshot Refを新しいworking epochまたはpublicationへ暗黙rebaseせず、同じEntity slot／generationに見えてもepochまたはpublication generationが異なればstaleとして拒否する。

`RuntimeEntityLiveRefV1`のresolveはdelivery／dispatch時のcurrent World instance generation、Entity slot generation、alive stateを検査する。無関係なEntityのstructural commitまたは新しいpublicationだけではLive Refを失効させないが、World instanceの破棄・再利用、対象Entityのdestroy、slot reuseまたはgeneration mismatchではstaleとして拒否する。resolve成功はComponent access、structural mutation、Gameplay authorityまたはleaseを付与せず、呼出し側は別途current Manifest、phase、scope、authorizationを満たさなければならない。

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
  initializer_spec_refs[1..8192]:
    sorted unique exact InitializerSpecRefV1
  entity_template_refs[1..4096]:
    sorted unique exact EntityTemplateRefV1
  component_schema_refs[1..8192]: sorted unique ComponentSchemaRefV1
  archetype_layout_plan_refs[1..8192]: sorted unique RuntimeArchetypeLayoutPlanRefV1
  persistent_identity_policy_refs[0..4096]:
    sorted unique exact McdContractRefV1(kind=policy)
  persistent_spawn_identity_request_refs[0..4096]:
    sorted unique exact RuntimePersistentSpawnIdentityRequestRefV1
  state_store_binding_refs[0..4096]:
    sorted unique exact RuntimeStateStoreBindingRefV1
  construction_set_hash: SHA-256
```

construction setはGatewayから独立したimmutable inputである。Gatewayがconstruction setを参照する一方向だけを許可し、相互Refによるhash cycleを作らない。initializerはinitial Component value、persistent identity disposition／policy、lifetime policyを閉じ、templateはinitializer集合と初期archetypeを一意に決める。templateのComponent集合、全initializerのComponent projection、initial archetypeのComponent集合をset equalityにし、同Componentのinitializerはexactly one、`tag`以外のmissing initializer、別Schema valueを拒否する。runtime create後に任意Componentを足して意味を完成させず、create permissionはmanifestがhash束縛したtemplateだけを指定できる。

### 4.2 Archetype layout

```text
RuntimeArchetypeLayoutPlanV1
  layout_plan_version: 1
  layout_algorithm_id: ecs_chunk_soa_v1
  layout_algorithm_version: 1
  archetype_id: exact StableArchetypeId
  required_component_schema_refs[1..8192]:
    sorted unique exact ComponentSchemaRefV1
  optional_enablement_component_refs[0..8192]:
    sorted unique exact ComponentSchemaRefV1
  payload_bytes: 16384
  payload_alignment_bytes: 64
  inline_component_max_bytes: 256
  row_capacity: positive uint32
  column_layouts[1..8193]:
    ordered {
      column_kind: entity_handle | component,
      component_schema_ref: null | exact ComponentSchemaRefV1,
      offset_bytes: u32,
      element_bytes: positive u32,
      element_alignment_bytes: one of 1 | 2 | 4 | 8 | 16 | 32 | 64,
      column_bytes: positive u32
    }
  entity_enablement_bitset:
    {offset_bytes: u32, bit_count: positive u32, byte_count: positive u32}
  component_enablement_bitsets[0..8192]:
    ordered {
      component_schema_ref: exact ComponentSchemaRefV1,
      offset_bytes: u32,
      bit_count: positive u32,
      byte_count: positive u32
    }
  column_layout_hash: SHA-256
```

target C1のchunk payloadは16 KiB、payload先頭alignmentは64 bytesである。`ecs_chunk_soa_v1`は次のNamed AlgorithmだけでPlanを生成する。

1. ArchetypeのComponent集合と`required_component_schema_refs[]`をset equalityにし、optional enablement集合がrequired集合のsubsetかつ全Schemaで`enablement_allowed=true`であることを検証する。`tag`と`derived_index`はpayload columnを作らない。
2. entity handle列を常にcolumn index 0、`element_bytes=8`、alignment 4、offset 0とする。残る`inline_value`／`external_handle`列はSchemaのcanonical bytes／alignmentを使い、alignment降順、同値はComponent Ref canonical byte順で並べる。
3. candidate row countを`floor(16384 / 8)=2048`から1まで降順に試す。各candidateについてentity handle列、ordered Component列をそれぞれ`align_up(previous_end, element_alignment)`へ配置し、`column_bytes=element_bytes×candidate`とする。
4. その後にentity enablement bitset、Component Ref順のenablement bitsetを配置する。各bitsetは8-byte alignment、`byte_count=align_up(ceil(candidate/8),8)`、unused bit 0とする。bitset bytesがcandidateに依存するため、この完全配置をcandidateごとに再計算する。
5. 最終endが16,384以下となる最初のcandidateを最大`row_capacity`として採用する。0まで成功しない場合は`diagnostic.runtime.ecs-chunk-capacity-zero`でcompile failureにする。全offset、size、alignment、row capacity、Component Ref、bitset metadataのcanonical bytesを`column_layout_hash`へ含める。

native `sizeof`、padding、compiler declaration order、template order、link order、Host ABIを入力にしない。全rangeはoverlapせず、endはpayload boundary以下、component columnのRef projectionはpayloadを持つrequired Component集合とset equality、enablement bitset projectionはoptional enablement集合とset equalityにする。Componentのadd／remove、Entity destroy、archetype移動は末尾rowをgapへmoveし、location tableを同一commitで更新する。Component address、row、span、chunk IDはstructural commit、phase終了、World destroyで無効になる。

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
  validation_receipt_refs[1..128]:
    sorted unique exact QualificationReceiptRefV1
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
  field_predicate_policy_ref:
    null | exact McdContractRefV1(kind=policy)
  partition_policy: canonical_range | deterministic_hash
  query_hash: SHA-256

RuntimeQueryRefV1
  query_id: StableId
  query_version: positive uint32
  query_hash: SHA-256

RuntimeQueryArchetypeCacheV1
  query_cache_version: 1
  query_ref: exact RuntimeQueryRefV1
  world_instance_handle: RuntimeWorldInstanceHandle
  world_epoch: positive uint64
  archetype_layout_set_hash: SHA-256
  matching_archetype_ids[0..8192]:
    sorted unique exact StableArchetypeId
  query_cache_content_hash: SHA-256

RuntimeQuerySelectionMaskV1
  query_ref: exact RuntimeQueryRefV1
  world_epoch: positive uint64
  archetype_id: exact StableArchetypeId
  chunk_id: exact RuntimeChunkId
  row_begin: uint32
  row_end_exclusive: uint32
  entity_enablement_mask_ref: null | exact ImmutableBitsetRefV1
  component_enablement_masks[0..1024]:
    sorted unique {
      component_schema_ref: exact ComponentSchemaRefV1,
      bitset_ref: exact ImmutableBitsetRefV1
    }
  predicate_result_mask_ref: null | exact ImmutableBitsetRefV1
  selected_row_count: uint32
  selection_hash: SHA-256
```

Query cache keyはexact `{query_ref, world_instance_handle, world_epoch, archetype_layout_set_hash}`である。cache buildは各Archetypeについてrequired集合がArchetype集合のsubset、excluded集合とのintersection empty、required-enabled集合がArchetype集合のsubsetの場合だけmatchingへ含める。field predicate、row、enable bit、selection mask、address、leaseをcacheへ入れない。成功structural commitはWorld epochを増やして旧keyを全てinvalidにし、新epochの差分更新結果は全Archetypeからのfull rebuildとset／hash equalityでなければならない。value writeとpublication generationだけではinvalidateしない。別World、別epoch、別layout-set hash、stale cacheをrejectし、表示名またはarchetype IDだけでreuseしない。

selection maskは`[row_begin, row_end_exclusive)`の物理row範囲だけを表す。presenceはcacheで確定済みのarchetype membership、selected bitsは`valid-row-mask ∩ entity-enable-mask ∩ all-required-component-enable-masks ∩ predicate-result-mask`で計算する。不要なoptional maskはnull、queryが要求するenablement Componentのmask projectionは`component_enablement_masks[]`とset equality、predicate policyがnullならpredicate maskもnullである。各Bitsetのbit countはrange length、scopeは同logical work、unused bitは0、`selected_row_count`はintersectionのpopcountと一致させる。maskはimmutableでcallback scopeに束縛し、row range、bitset length、World epoch、query hash、chunk capacityの不一致を拒否する。

### 5.2 Dispatch plan

```text
RuntimeQueryDispatchPlanV1
  plan_version: 1
  query_ref: exact RuntimeQueryRefV1
  world_epoch: positive uint64
  query_cache_ref: exact RuntimeQueryArchetypeCacheRefV1
  partition_policy: canonical_range | deterministic_hash
  work_units[0..65536]:
    {
      logical_work_id: bytes32,
      archetype_id: exact StableArchetypeId,
      chunk_id: exact RuntimeChunkId,
      row_begin: uint32,
      row_end_exclusive: uint32,
      selection_mask_ref: exact RuntimeQuerySelectionMaskRefV1
    }
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
  system_ref: exact GameSystemContractRefV1
  permitted_phase_ids[1..64]:
    sorted unique exact TickPhaseId
  query_refs[0..256]:
    sorted unique exact RuntimeQueryRefV1
  read_component_refs[0..1024]:
    sorted unique exact ComponentSchemaRefV1
  write_component_refs[0..1024]:
    sorted unique exact ComponentSchemaRefV1
  structural_permissions[0..256]:
    sorted unique exact RuntimeStructuralPermissionV1
  state_store_binding_refs[0..256]:
    sorted unique exact RuntimeStateStoreBindingRefV1
  budget_refs[1..64]:
    sorted unique exact BudgetScopeRefV1
  manifest_hash: SHA-256
```

read／write集合、query集合、phase、structural permission、state bindingはmanifestで閉じる。`RuntimeComponentAccessManifestV1`はcontent-addressedな静的Definitionであるため、Scheduling Ownerのserialized `TickPhaseId`だけを保持し、`SimulationAdvanceIntervalRefV1`、advance sequence、publication generationまたは`RuntimeTimeRefV1`を保持しない。CompilerはSystem registration、query contract、Component owner、template、initializer、persistence policy、capacity budgetとの一致を検証する。未宣言Component access、任意Componentを対象にするstructural permission、phase外callback、manifest外state storeを生成bindingから表現できないようにする。

`structural_permissions[]`はoperation kind、許可target Component／transition、create可能template hash、静的apply phase ID、count／byte budget、lifetime ownerを持つ。実行時にはOrchestratorがcurrent `RuntimeTimeRefV1.phase_id`と`apply_phase_id`をbyte equalityで照合するが、実行時Time RefをManifestへ書き戻さない。`create_entity`、`destroy_entity`、`set_entity_enabled`はEntity lifetime owner、`add_component`、`remove_component`、`set_component_enabled`は対象Component ownerからだけ発行できる。

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
    {
      system_ref: exact GameSystemContractRefV1,
      component_access_manifest_ref:
        exact RuntimeComponentAccessManifestRefV1,
      query_dispatch_contract_refs[0..256]:
        sorted unique exact RuntimeQueryRefV1,
      implementation_ref: exact ArtifactRefV1,
      owner_ref: exact OwnerRefV1
    }
  state_store_binding_refs[0..4096]:
    sorted unique exact RuntimeStateStoreBindingRefV1
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
  target_component_schema_ref: null | exact ComponentSchemaRefV1
  enabled_value: null | bool
  transition_ref: null | exact ComponentTransitionRefV1
  template_ref: null | exact EntityTemplateRefV1
  payload_ref: null | exact CanonicalValueRefV1
  persistent_spawn_identity_request_ref:
    null | exact RuntimePersistentSpawnIdentityRequestRefV1
  apply_boundary_ref: RuntimeStructuralCommitBoundaryRefV1
  content_hash: SHA-256
```

deltaはcallback中にprivate batchへ追加するだけであり、live Worldを変更しない。`EntityCreateToken`は同じprivate batch内で、より小さい`local_sequence`のcreate結果だけを参照できる。tokenを別batch、別work item、別System、次boundaryへ保存または渡すことを拒否する。

`kind`ごとのField形状は次のclosed unionである。`required`以外のbranch固有Fieldは`absent`でなければならず、unknown Fieldまたは複数branchの混在を拒否する。

| kind | target | required | conditional |
|---|---|---|---|
| `create_entity` | このdeltaが宣言する新しい`EntityCreateToken` | `template_ref` | `runtime_spawn_required`では同System／work／sequence／boundaryの`persistent_spawn_identity_request_ref`必須、他dispositionではnull |
| `destroy_entity` | `RuntimeEntityHandle`または同batchの先行`EntityCreateToken` | なし | `expected_component_presence_refs`だけをpreconditionに使用できる |
| `add_component` | existing handleまたは先行token | `target_component_schema_ref`、`transition_ref` | `inline_value`／`external_handle`でcanonical defaultがなければ`payload_ref`必須、`tag`では禁止 |
| `remove_component` | existing handleまたは先行token | `target_component_schema_ref`、`transition_ref` | `expected_component_presence_refs`に同Componentを含める |
| `set_entity_enabled` | existing handleまたは先行token | `enabled_value` | Component、transition、template、payload Fieldは全て禁止 |
| `set_component_enabled` | existing handleまたは先行token | `target_component_schema_ref`、`enabled_value` | 対象Schemaがenablementを許可しなければ拒否する |

runtime createはtemplateのinitializer、lifetime、persistent identity policyをそのまま消費する。`runtime_spawn_required`はPersistence Ownerのexact Allocation Bindingがrequestへ応答済みであることを全batch preflightで検証し、未応答、複数応答、別request／World／Template／owner、identity collisionをbatch全体のfailureにする。preflight後に成立するpersistent identityのdistinct集合を`P_next(W)`とし、current publicationの集合へ同batchのpersistent destroyを除外して成功createのallocated identityを加えたset equalityで構成する。`RuntimeWorldBuildGatewayV1.required_capacity_ref`が解決する`RuntimeWorldCapacityRecordV1`に対し、checked arithmeticで`|P_next(W)| <= max_persistent_entities <= 1048576`を満たす時だけbatchをcommit可能にする。createとdestroyの順序、重複request、section reservationまたは`max_runtime_entities`だけでこの上限を代用しない。call siteが任意Persistent IDをpayloadへ埋め込み、ephemeral templateをSave対象へ昇格し、commit前tokenをAI／Save／Eventへ公開することを禁止する。`content_hash`はbranch validation後のrecordについて、`SHA-256(ASCII "MIRAKAN_RUNTIME_COMPONENT_LIFECYCLE_DELTA_V1" || uint32_be(len(canonical delta bytes excluding content_hash)) || canonical delta bytes excluding content_hash)`で計算する。

### 7.2 Atomic commit

boundaryはdeltaを`{advance_sequence, producer_phase_id, producer_system_id, producer_logical_work_id, local_sequence}`でcanonical mergeする。commit前にtarget handle／generation、precondition、owner permission、transition、destination capacity、identity collision、budget、dependencyを全件検査し、staging mutation planと必要chunk reservationを構築する。

検査済みdeltaが全て成功した時だけ、chunk row、location table、archetype membership、enablementを一commit pointでprivate working Worldへ反映し、World epochをexact 1増加させる。一件でも失敗した場合はstagingを破棄し、previous World、location table、query cache、output packet、publication hashを変更しない。partial commit、callback中のdirect mutation、vendor callbackからのWorld mutation、completion順によるmergeを禁止する。成功時は同じepoch ruleに従い、影響するaddress、row、span、lease、query membershipをinvalidateする。

structural commitは次のadvanceの`T00_BoundaryApply`でprivate working Worldへ適用する内部commitであり、それ自体を外部publicationにしない。同advanceのauthoritative phaseが後でfaultした場合、変更済みworking World、value-write journal、unsealed outputを外部へpublishせず、別advanceの開始状態として再利用しない。SchedulerはPlay session faultまたはlast published checkpointからの明示recoveryを選び、ECSが完了済みphaseを暗黙rollbackしたと報告しない。外部consumerに対するauthorityは最後に成功した`T110_Publish`のまま維持する。

Stable identityはtyped generation handleだけが持つ。chunk ID、row、address、pool slotをidentity、Save ref、AI targetとして扱わない。Shipping pathにAoS、sparse-set、object-graph、old reader／writer fallbackを残さない。

## 8. ECS AI contract graph

AIにECSの構造を説明するため、OwnerがsealしたContract graphだけを使う。Graphは実行surfaceではなく、Operation、Tool、write権限、live World accessを追加しない。

`ArchitectureExplainProjectionV1`はRuntime ECSのOwner、文書状態、依存、consumer関係を説明し、`RuntimeEcsContractGraphV1`はECS内部のstorage、query、access、structural、publication意味を説明し、`OptimizationDecisionProjectionV1`は評価済みCandidateの状態とEvidenceを説明する。`AiTaskContextCapsuleV1`はTaskに必要なこれらのexact refを束縛するが、いずれか一つを他の代用にしない。Architecture文書名からECS nodeを生成せず、ECS graphから候補のselected状態を生成せず、Task CapsuleからOperationまたはApprovalを生成しない。

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
  layout_plan_version: positive uint32
  layout_algorithm_id: ecs_chunk_soa_v1
  archetype_id: StableArchetypeId
  column_layout_hash: SHA-256

RuntimeEcsContractGraphRefV1
  graph_version: 1
  world_publication_ref: RuntimeWorldPublicationRefV1
  graph_hash: SHA-256

RuntimeWorldBuildGatewayRefV1
  gateway_version: positive uint32
  gateway_hash: SHA-256

RuntimeQuerySelectionMaskRefV1
  world_epoch: positive uint64
  archetype_id: StableArchetypeId
  chunk_id: RuntimeChunkId
  selection_hash: SHA-256

RuntimeQueryArchetypeCacheRefV1
  query_ref: exact RuntimeQueryRefV1
  world_instance_handle: RuntimeWorldInstanceHandle
  world_epoch: positive uint64
  archetype_layout_set_hash: SHA-256
  query_cache_content_hash: SHA-256

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

### 9.1 Initial V1 consumer boundary

次表はinitial V1で各ECS projectionが参照してよい入力を分類する。行の存在は実consumer、reader、releaseまたはold-reader windowの存在を意味せず、Compatibility inventoryをmaterializeしない。各consumerを実装または公開する時はexact Owner／Definition refへ直接束縛し、初回materialization後に形式を変更する場合だけ[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)を適用する。

| ECS境界 | consumer class | 許可される入力 |
|---|---|---|
| Project Source／Asset import／approved Runtime entry | `authoring_source`／`derived_artifact` | committed Sourceとtarget Toolchain lockだけ |
| World Root／Section image／Runtime Package | `runtime_package` | immutable World image、Catalog、target Contract。old Package bytesは不可 |
| Save／Replay projector | `save`／`replay` | sealed publicationとpersistent projection。raw handleは不可 |
| Native Game Module／C ABI | `native_abi` | generated bounded view、typed ref、owned buffer。live leaseは不可 |
| Renderer／Audio／VFX | `runtime_projection` | sealed snapshotまたはpresentation projectionだけ |
| Debug transport／AI capture | `observability_projection` | redacted sealed graph／captureだけ |
| 外部API／配布先／documentation projection | `external_api`／`documentation` | approved current projectionだけ |

### 9.2 Component Schema evolution closure

Initial V1の`ComponentSchemaV1`は最初のcanonical versionを直接定義し、旧draft schema、alias、migrationまたはdual readerを持たない。初回materializationまたは公開後にComponent storage、canonical value、default、enablement、semantic owner bindingを変更する場合だけ次を必須にする。

```text
ComponentSchemaEvolutionBindingV1
  schema_evolution_binding_id: StableId
  schema_evolution_binding_version: 1
  source_component_schema_ref: exact ComponentSchemaRefV1
  target_component_schema_ref: exact ComponentSchemaRefV1
  compatibility_change_ref: exact CompatibilityChangeRefV1
  consumer_inventory_ref: exact CompatibilityConsumerInventoryRefV1
  affected_consumer_classes[1..10]:
    sorted unique CompatibilityConsumerClass
  value_transform_policy:
    {
      kind: no_value_transform,
      proof_evidence_requirement_ref: exact EvidenceRequirementRefV1
    }
    | {
        kind: deterministic_transform,
        transform_contract_ref: exact McdContractRefV1(kind=operation),
        failure_diagnostic_ref: exact McdContractRefV1(kind=diagnostic)
      }
  runtime_world_policy: reject_mixed_schema_versions
  derived_artifact_policy: full_recook | full_rebuild | not_applicable
  save_replay_policy:
    reject_old_format | bounded_versioned_reader
  native_abi_policy:
    regenerate_and_requalify | bounded_external_deprecation
  required_evidence_requirement_refs[1..128]:
    sorted unique exact EvidenceRequirementRefV1
  schema_evolution_binding_content_hash: SHA-256

ComponentSchemaEvolutionBindingRefV1
  schema_evolution_binding_id: StableId
  schema_evolution_binding_version: 1
  schema_evolution_binding_content_hash: SHA-256
```

sourceとtargetは同じ`component_type_id`、同じsemantic owner、`target.component_schema_version = source.component_schema_version + 1`で、Refは異ならなければならない。同じversionの別hash、version飛越し、meaningを別Ownerへ移す変更、targetからsourceへの逆edgeを拒否する。Componentの意味またはOwnerが変わる場合は新しいComponent type identityとし、schema revisionへ偽装しない。

`compatibility_change_ref`のsubject、source／target format、consumer inventory、affected class集合はBindingとbyte／set equalityにする。Inventoryが`complete`かつ`zero_verified`でなく、全Evidence Requirementにpass fulfillmentがない場合は`approved | applied`を受理しない。`clean_break`／`source_preserving_recook`ではold reader、old writer、aliasを禁止し、`versioned_reader_migration`だけが有限の`bounded_versioned_reader`を選べる。Save／Replay、Runtime Package、Native ABI、observability、external API、documentationの一件でもmaterialized consumerがあれば、そのclassをInventoryとBindingから省略しない。

Schema revision時は旧Schemaを含むArchetype、Layout Plan、Transition、Template、Query、System manifest、Package、Save／Replay projector、Native projectionをtarget Contract Setへ再解決する。live Worldまたは一つのArchetypeで同じComponent typeの複数Schema versionを混在させず、load／reconstruction／Package activation前に全変換またはtyped rejectionを完了する。field欠落、bytes length、default、canonical zero、display nameまたは旧hashからsilent migrationを推測しない。

### 9.3 Runtime ECS Diagnostic

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

```text
RuntimeEcsDefinitionFixtureContractV1
  fixture_contract_id: StableId
  fixture_contract_version: 1
  contract_set_ref: exact ContractSetRefV1
  target_profile_ref: exact TargetProfileRefV1
  canonicalization_profile_ref: exact McdContractRefV1(kind=profile)
  required_fixture_cases[26..65535]:
    sorted unique {
      fixture_case_id: StableId,
      case_kind:
        component_schema_hash | canonical_value_hash
        | archetype_identity | layout_plan | layout_capacity_zero
        | bitset_canonicalization | query_cache_membership
        | query_cache_invalidation | selection_mask_intersection
        | dispatch_determinism | create_token_scope
        | transition_add | transition_remove
        | persistent_spawn_allocation | structural_commit_success
        | structural_commit_atomic_failure | stale_entity_ref
        | lease_conflict | publication_hash | hash_cycle_negative
        | component_schema_evolution | type_inventory_closure
        | ref_resolution | structural_permission_branch
        | unknown_field_negative | ephemeral_persistence_negative,
      input_artifact_ref: exact ArtifactRefV1,
      expected_result:
        {
          kind=success,
          canonical_output_artifact_ref: exact ArtifactRefV1,
          golden_content_hash: SHA-256
        }
        | {
            kind=diagnostic,
            diagnostic_ref: exact McdContractRefV1(kind=diagnostic),
            golden_diagnostic_hash: SHA-256
          }
    }
  cross_language_projection_refs[3..16]:
    sorted unique exact ArtifactRefV1
  fixture_contract_content_hash: SHA-256

RuntimeEcsDefinitionFixtureContractRefV1
  fixture_contract_id: StableId
  fixture_contract_version: 1
  fixture_contract_content_hash: SHA-256
```

required casesは上記26 `case_kind`をそれぞれexactly one以上含み、C++／canonical binary／strict JSONの三projectionでvalue、presence、bound、sort、branch、hashを一致させる。`persisted_content_addressed` inventoryの各backing recordについて最低一つのpositive golden canonical bytes／hash vectorを持ち、全Refを同名backing recordへ解決する。layout caseは異なるalignment、tag、external handle、enablement bitset、境界ちょうど、capacity 0を含み、query caseはrequired／excluded／enabled／predicate、別World、epoch advance、full rebuild同値を含む。persistent spawn caseは同request idempotency、duplicate response、collision、ephemeral misuseを含み、structural caseは全件成功またはzero mutationを検証する。`type_inventory_closure`は本文宣言とのset equality、`ref_resolution`はmissing backing／same-name different-hash／external Owner違い、`structural_permission_branch`はmatrix外Field、`unknown_field_negative`は全closed recordのunknown sibling、`ephemeral_persistence_negative`はPackage／Save／Replay／AI／cross-sessionへの各混入をrejectする。schema evolution caseはsame-version別hash、version飛越し、incomplete Consumer Inventory、Save／Replay／Native consumer欠落、mixed runtime versionをrejectし、承認済みclean break／recook／bounded reader branchだけを受理する。対応Fixture record、artifact、golden hash、cross-language projectionは現Repositoryに存在せず、このSchemaの記載だけをFixture完了またはQualification passにしない。

### 10.1 E6 AI readiness binding

```text
RuntimeEcsAiReadinessBindingV1
  ai_readiness_binding_id: StableId
  ai_readiness_binding_version: 1
  project_revision_ref: exact ProjectRevisionRefV1
  target_profile_ref: exact TargetProfileRefV1
  contract_set_ref: exact ContractSetRefV1
  toolchain_profile_ref: exact ToolchainProfileRefV1
  architecture_explain_projection_ref:
    exact {
      projection_artifact_ref: ArtifactRefV1,
      projection_content_sha256: SHA-256
    }
  ecs_contract_graph_ref: exact RuntimeEcsContractGraphRefV1
  optimization_decision_projection_ref:
    exact OptimizationDecisionProjectionRefV1
  task_context_capsule_ref:
    exact {
      capsule_id: StableId,
      schema_version: 1,
      content_hash: SHA-256
    }
  required_operation_bindings[2..2]:
    sorted unique {
      family: ai_read | ai_explain,
      operation_ref: exact McdContractRefV1(kind=operation)
    }
  route_kind:
    engine_provider_adapter | standard_external_mcp | managed_external_host
  understanding_eval_verification_receipt_ref:
    exact EvidenceRefV1
  binding_expires_at: RFC 3339 timestamp
  ai_readiness_binding_content_hash: SHA-256

RuntimeEcsAiReadinessBindingRefV1
  ai_readiness_binding_id: StableId
  ai_readiness_binding_version: 1
  ai_readiness_binding_content_hash: SHA-256
```

Bindingの全Projection、Capsule、Operation、Eval Receiptは同じProject revision、Target、Contract Set、Toolchain、source revision、fixture、Provider／Model／Host／Deployment、redaction policyへexact一致し、dispatch時にfreshかつnon-revokedでなければならない。`required_operation_bindings[]`は[Product Plan](../00-product/product-plan.md)のRequired Product Operation Universeにある`ai_read`と`ai_explain`のexact二件で、Operation Activation Closure、Capsule allowlist、route grantのintersectionとset equalityにする。理解EvalはECS graph ownership、storage／query／structural意味、selected／rejected optimization、redacted／unknown境界を独立に検証し、一般Model Eval、Markdown読解、propose／commit Operationまたは別Target Receiptで代用しない。本Binding、Projection、Operation、Fixture、Receiptは未materializeであり、E6 AI readabilityは利用不可である。

### 10.2 E7 Target／device qualification binding

```text
RuntimeEcsTargetQualificationBindingV1
  target_qualification_binding_id: StableId
  target_qualification_binding_version: 1
  target_profile_ref: exact TargetProfileRefV1
  contract_set_ref: exact ContractSetRefV1
  toolchain_profile_ref: exact ToolchainProfileRefV1
  runtime_package_ref: exact RuntimePackageRefV1
  qualification_environment:
    {
      execution_kind: physical_device | qualified_windows_host,
      device_identity_hash: SHA-256,
      os_image_artifact_ref: exact ArtifactRefV1,
      cpu_identity_artifact_ref: exact ArtifactRefV1,
      gpu_identity_artifact_ref: exact ArtifactRefV1,
      driver_package_artifact_ref: exact ArtifactRefV1,
      power_thermal_profile_ref: exact McdContractRefV1(kind=profile)
    }
  required_reference_dimensions[2..2]:
    sorted unique two_d | three_d
  definition_fixture_contract_ref:
    exact RuntimeEcsDefinitionFixtureContractRefV1
  data_oriented_campaign_ref:
    exact {
      campaign_version: 1,
      campaign_hash: SHA-256
    }
  platform_package_qualification_receipt_ref:
    exact EvidenceRefV1
  verification_receipt_refs[1..4096]:
    sorted unique exact EvidenceRefV1
  required_product_requirement_refs[2..64]:
    sorted unique exact McdContractRefV1(kind=requirement)
  target_qualification_binding_content_hash: SHA-256

RuntimeEcsTargetQualificationBindingRefV1
  target_qualification_binding_id: StableId
  target_qualification_binding_version: 1
  target_qualification_binding_content_hash: SHA-256
```

Windows Desktop、Android Mobile、Apple Mobileは別Bindingとし、Target Profile、physical deviceまたはqualified Windows host、OS、CPU、GPU、driver、power／thermal、Toolchain、Runtime Package、Candidate、Contract Set、2D／3D campaignを相互流用しない。Android／Appleは`execution_kind=physical_device`だけを許す。`required_product_requirement_refs[]`はProduct Release Projectionの該当Target×2D／3D requirement identityとset equality、Receipt集合はDefinition fixture、六scenario Performance campaign、structural atomicity、Save／Replay、Package load、Native projectionのrequired class集合とset equalityにする。一Target、一dimension、simulator、WARP、Development configurationまたは別CandidateのpassからE7を推測しない。BindingとReceiptは未materializeであり、全Target対応またはProduct Releaseを意味しない。

target ECSは少なくとも次を証明する。

1. 同一construction inputから同じarchetype layout、query dispatch plan、structural merge、publication hashを二回生成する。
2. stale handle、generation wrap、cross-World ref、unselected row write、phase外lease、unmanifested accessをinvalid fixtureで拒否する。
3. structural command failureがlive World、location table、publicationを変更しない。
4. `canonical_range`と`deterministic_hash`でcallback unitがchunk境界を越えず、worker数や完了順でresultが変わらない。
5. value writeがseal前snapshot／AI／Save／Presentationへ露出せず、faulted advanceがpublicationされない。
6. package、Save、Replay、Debug、Native moduleとのowner境界がraw handle、raw pointer、旧aliasを渡さない。
7. 8／16／32 KiB比較、capacity、fragmentation、structural burst、query cache invalidationをPerformance evidenceで検証する。
8. Gameplay、Scheduling、Package、Save／Replay、Debug、Native、Domain consumerが同じinitial V1 Owner／Definition refを直接参照し、duplicate Owner、unresolved ref、近似名fallback、旧aliasが0件である。
9. hot callbackのgeneral-heap allocation countとupstream fallback countが両方exact 0で、query scratch、selection mask、command buffer、output packetがdispatch前に予約される。
10. `RuntimeComponentLayoutPolicyV1`のaccess cohortがactive System manifest、query、phase、read／write setから再導出でき、manual hot／cold labelまたはautomatic field splitを含まない。
11. T110でsealしたbatchが指定された次のT00でexactly once applyされ、T110でlive mutationされず、wrong generation／wrong phase／別hash再送を拒否する。
12. `world_epoch`と`world_publication_generation`が定義したイベントだけで増加し、faulted advanceのworking Worldが次advance、AI、Save、Replay、Presentationへ露出しない。
13. `GameSystemSpecV1`、ECS access manifest、Native descriptorのread Component、write Component、State read／write、structural permission集合が正逆方向に一致する。
14. 本書が参照する全Type／Ref、collection bound、branch constraint、canonical encoding、hash golden vectorがDefinition Closureへexact解決し、unresolved shorthandが0件である。
15. Component Schema変更はapproved Compatibility Change、complete／zero-verified Consumer Inventory、Save／Replay／Native／Package projection、全Evidence Requirementを同じBindingへ閉じ、mixed runtime Schema、silent migration、aliasを拒否する。
16. E6はAI Readiness Bindingの二Operation、Capsule、Projection、Eval Receipt、E7はTarget別Qualification Bindingの2D／3D、device、Toolchain、Package、campaign、Receiptがそれぞれset equalityである。

## 11. 外部一次資料から採る原則

[Unity Entitiesのsync point資料](https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/performance-sync-points.html)はstructural changeを予測可能なpointへまとめるためのCommand Buffer利用を説明する。[Flecs Queries資料](https://github.com/SanderMertens/flecs/blob/v4.1.2/docs/Queries.md)は反復利用queryのcacheとiteration中table mutationのdeferを説明する。[EnTT Entity資料](https://github.com/skypjack/entt/blob/v3.16.0/docs/md/entity.md)はiteration中の変更でreferenceが無効化され得ることを説明する。

これらは比較用の一次資料であり、MiraikanaiのAPI、chunk size、Schema、scheduler、binary formatを規定しない。特に16 KiB、64-byte alignment、256-byte上限はMiraikanaiのtarget Contractであり、vendorの推奨値として扱わない。
