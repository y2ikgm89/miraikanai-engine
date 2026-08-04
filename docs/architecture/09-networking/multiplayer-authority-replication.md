# Miraikanai Engine Multiplayer Authority／Replication Contract

- 文書ID: mirakan.arch.multiplayer-authority-replication
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: multiplayer topology／participant role／authority、gameplay session lifecycle、network object identity、spawn／despawn、state／event／typed command replication、snapshot／delta／baseline／ack、relevancy／interest／dormancy／priority、network time mapping、prediction／reconciliation／rollback／resimulation／interpolation、join／leave／late join／resync／reconnect、qualified host migration／authority handoff、domain diagnostic／qualification
- 非正本範囲: endpoint／socket／packet／fragment／encryption／transport reconnect、Account／identity／entitlement、party／lobby／matchmaking／hosting／region placement、Simulation phase／cadence、ECS storage／query／structural transaction、Input action semantics、Save／Replay envelope、anti-cheat incident、cloud persistence／economy／moderation、shared capacity、Evidence envelope、実装Task／順序。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Network Transport／Connection](network-transport-connection.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime ECS](../04-runtime/entity-component-system.md)
- 関連文書: [Product Plan](../00-product/product-plan.md)、[Advanced Rendering／Multiplayer Ownership Decision](../decisions/2026-07-29-advanced-rendering-multiplayer-ownership.md)、[Product Security](../01-governance/product-security.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Persistence／Save](../04-runtime/persistence-save.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[Debugging／Observability／Replay](../04-runtime/debugging-observability-replay.md)、[World](../06-rendering/world.md)、[Input](../07-platform/input.md)、[Shooter Genre Pack](../08-packs/shooter.md)
- 根拠区分: project-decision／official-documentation comparison（未計測のtick、rollback window、snapshot rate、bandwidth、object／participant count、timeoutはprovisional）
- 外部根拠確認日: 2026-07-29

## 1. 結論、状態、authority境界

本書は「誰が何を決定し、どのgameplay stateをどの相手へいつ投影するか」を所有する。dedicated server、listen server、peer authority、将来のsharded／distributed authorityをclosed topologyとして区別し、role、object authority、state replication、typed command、prediction／rollback、join／resync／handoffを同じauthority modelへ閉じる。

[Network Transport／Connection](network-transport-connection.md)はmessage bytesの接続／deliveryを所有する。本書はsocket、packet、fragment、IP、native channel、encryption、transport retryを知らず、登録済みmessageとsemantic delivery classだけを使う。Transport `active`はgameplay session `joined`または`synchronized`を意味しない。

[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)はSimulation Advance、phase、job dependency、publication、lifetimeを所有し、[Runtime ECS](../04-runtime/entity-component-system.md)はEntity／Component identity、storage、query、lease、structural transactionを所有する。本書はexact snapshot／command boundaryを宣言するが、remote peerをphase ownerにせず、network arrival callbackからECSを直接変更しない。

Account、entitlement、Lobby、Matchmaking、Hosting、region allocation、cloud persistenceは別将来Ownerである。本書はopaque external participant／session-discovery／hosting bindingを任意入力として受けられるが、それらをgameplay authorityまたはTransportへ統合しない。

[Product Plan §8](../00-product/product-plan.md#8-future-portfolio)はsmall co-op、rollback、large-session、MMOというProduct consumerのreceipt-free bundle profile、client／authority／operations role集合、direct Future prerequisite role mapping、claim release boundaryを所有する。本書はpromotion subjectで選択されたexact role／Target Profile bindingを`MultiplayerProfileV1`へ投影し、topology／authority／replicationをQualificationするが、Product bundle membership、claimまたはexecution Registryを再所有しない。`headless_server` kindだけからdedicated topologyを選ばず、dedicated profileでは[Runtime Package](../04-runtime/runtime-package.md)のFuture prerequisite bindingを必須にする。

本書の型とFuture entryは`review`／`absent`／`planning_only`である。current Multiplayer、server、replication、rollback、Online、Target support、Provider、実装はRepositoryに存在しない。

## 2. 不変条件とclosed語彙

1. authorityは明示的role／scope／epoch／leaseで決まり、connection ownership、message sender、object proximity、first writer、local player、host processから推測しない。
2. Network Object identity、Runtime ECS identity、World／Save identity、Transport connection identity、Account identityを別型にし、exact bindingだけで接続する。
3. state、event、command、RPC、snapshot、delta、baseline、ackを別message kindにし、受信bytesまたはfunction名から意味を推測しない。
4. predictionはpresentation／local proposalであり、authoritative stateを置き換えない。correction、reconciliation、rollback、resimulation、interpolationをProfileで明示する。
5. rollbackはMultiplayer Owner内のoptional subprofileで、すべてのMultiplayer／large-session topologyへ強制しない。
6. join、late join、reconnect、resync、host migration、authority handoffは別state／protocolである。Transport reconnectだけでsession復帰しない。
7. reliable transportをexactly-once gameplay effectにしない。command／eventはidempotency、authority、validation、expiry、orderingをdomain Schemaで持つ。
8. fallbackでserver authorityをclient authorityへ、dedicatedをlistenへ、rollbackをsnapshot interpolationへsilent変更しない。

```text
MultiplayerTopologyV1 =
  dedicated_server
  | listen_server
  | peer_authority
  | sharded_authority
  | distributed_authority

MultiplayerParticipantRoleV1 =
  authority_server
  | owning_client
  | simulated_client
  | spectator
  | handoff_participant

MultiplayerSessionStateV1 =
  created
  | awaiting_transport
  | authenticating_binding
  | joining
  | synchronizing
  | active
  | resynchronizing
  | migrating_authority
  | leaving
  | left
  | fault

ReplicationPayloadKindV1 =
  spawn
  | despawn
  | state_snapshot
  | state_delta
  | event
  | command
  | acknowledgement
  | baseline_control
  | authority_control
```

`sharded_authority | distributed_authority`はFuture topology branchで、単に複数server connectionがある状態ではない。shard／region scope、cross-authority transaction、handoff、failure policy、Target／operations Qualificationがない場合は`unsupported`とする。

## 3. Multiplayer Profileとtopology

```text
MultiplayerDocumentV1
  document_id: StableId
  source_revision: positive u64
  profile_refs[1..64]: MultiplayerProfileRefV1
  session_template_refs[0..256]:
    MultiplayerSessionTemplateRefV1
  target_policy_ref: exact MultiplayerTargetPolicyRefV1
  document_content_hash: SHA-256

MultiplayerProfileV1
  profile_id: StableId
  profile_version: positive u32
  owner_ref: exact OwnerRefV1
  topology_profile_ref: exact MultiplayerTopologyProfileRefV1
  participant_role_catalog_ref:
    exact ParticipantRoleCatalogRefV1
  authority_policy_ref: exact MultiplayerAuthorityPolicyRefV1
  replication_profile_ref: exact ReplicationProfileRefV1
  time_mapping_profile_ref: exact NetworkTimeMappingProfileRefV1
  prediction_profile_ref:
    optional exact PredictionReconciliationProfileRefV1
  rollback_profile_ref:
    optional exact RollbackResimulationProfileRefV1
  session_lifecycle_profile_ref:
    exact MultiplayerSessionLifecycleProfileRefV1
  transport_requirement_ref:
    exact MultiplayerTransportRequirementRefV1
  security_requirement_ref:
    exact MultiplayerSecurityRequirementRefV1
  target_binding:
    { kind: standalone_single_target,
      target_profile_ref: exact TargetProfileRefV1 }
    | { kind: product_role_bundle,
        source_promotion_manifest_ref:
          exact FutureToActivePromotionManifestRefV1,
        bundle_profile_id: StableId,
        bundle_profile_content_hash: SHA-256,
        role_target_bindings[2..8]:
          { product_role_id: StableId,
            participant_role_set[1..5]:
              MultiplayerParticipantRoleV1,
            target_profile_refs[1..16]:
              exact TargetProfileRefV1 } }
  capacity_limit_ref: exact MultiplayerCapacityLimitRefV1
  qualification_policy_ref:
    exact MultiplayerQualificationPolicyRefV1
  profile_content_hash: SHA-256
```

`product_role_bundle`は[Product Plan §8](../00-product/product-plan.md#8-future-portfolio)に従ってpromotion subjectが選んだTarget closure bindingを再定義しないread-only projectionである。`bundle_profile_id`／hashを同じpromotion subjectの選択Profileと一致させ、選択Profileの`client | authority_*` role ID集合、各roleのexact Target集合と`role_target_bindings[]`をset equalityにする。各`participant_role_set[]`はTopology Profileがそのproduct roleへ許可したnon-empty subsetで、同じparticipant roleを相反するproduct roleへ重複させない。role bindingはproduct role ID、Target refsはexact Target tuple順へ正規化する。`operations | execution_host | artifact_target` roleを本Profileへ複写せず、別roleのTargetをflat unionまたはimplicit cross productから補完しない。`standalone_single_target`ではbundle Fieldを禁止する。promotion bindingのmachine-readable carrierはcurrent Repositoryで未採択／未materializeであり、旧Proposalの`FutureToActivePromotionManifestV1`をcurrent sourceとして要求または推測しない。

```text
MultiplayerCapacityLimitV1
  limit_id: StableId
  limit_version: positive u32
  maximum_participants: bounded positive u32
  maximum_runtime_entries: bounded positive u32
  maximum_transport_bindings_per_participant:
    bounded positive u32
  maximum_authority_leases: bounded positive u32
  maximum_network_objects: bounded positive u32
  maximum_baseline_object_states: bounded positive u32
  maximum_acknowledged_messages: bounded positive u32
  maximum_missing_ranges: bounded positive u32
  maximum_projection_objects: bounded positive u32
  maximum_handoff_leases: bounded positive u32
  maximum_handoff_acknowledgements: bounded positive u32
  limit_content_hash: SHA-256
```

各symbolic array boundは同じProfileの`capacity_limit_ref`にある同名`maximum_*` Fieldへ解決する。Field欠落、zero、schema hard maximum超過を拒否する。上限はportable safety／allocation boundであってproduction player countまたはscale claimではなく、runtime測定値からSource limitを逆変更しない。

`rollback_profile_ref`がnon-nullなら`prediction_profile_ref`、rollback可能state boundary、Input Snapshot、deterministic fixture、Save／Replay mappingを必須にする。nullの場合、rollback fieldを別Profile名やTopologyから推測しない。

```text
MultiplayerTopologyProfileV1
  topology_profile_id/version/content_hash
  topology: MultiplayerTopologyV1
  authority_role_set[1..16]:
    MultiplayerParticipantRoleV1
  required_runtime_target_refs[1..64]:
    exact RuntimeTargetRequirementRefV1
  authority_scope_model_ref:
    exact AuthorityScopeModelRefV1
  host_failure_disposition:
    session_fault
    | qualified_host_migration
    | external_replacement_authority
  external_hosting_binding_ref:
    optional exact OpaqueHostingBindingRefV1
```

`dedicated_server`は[Runtime Package](../04-runtime/runtime-package.md)のqualified headless Target closureを必須にする。`listen_server`はlocal presentation clientとauthority processが同居できるがrole／authority／failureを統合しない。`peer_authority`は全peerが全objectを決定する意味ではなく、scope／lease／conflict policyを必須にする。host migrationは明示branchと専用Qualificationがある場合だけ許す。

## 4. Participant、gameplay session、Transport binding

```text
MultiplayerParticipantIdentityV1
  participant_id: StableId
  participant_epoch: positive u64
  previous_participant_ref:
    null | exact MultiplayerParticipantRefV1
  session_id: StableId
  session_epoch: positive u64
  role: MultiplayerParticipantRoleV1
  external_identity_binding_ref:
    optional exact OpaqueExternalIdentityBindingRefV1
  transport_connection_binding_refs[
    1..capacity.maximum_transport_bindings_per_participant]:
    exact SessionTransportConnectionBindingRefV1
  transport_route_policy_ref:
    exact ParticipantTransportRoutePolicyRefV1
  participant_content_hash: SHA-256

SessionTransportConnectionBindingV1
  binding_id/version/content_hash
  session_id: StableId
  session_epoch: positive u64
  participant_id: StableId
  transport_connection_pair_identity_ref:
    exact TransportConnectionPairIdentityRefV1
  participant_connection_identity_ref:
    exact TransportConnectionIdentityRefV1
  permitted_message_type_refs[1..256]:
    exact RegisteredMessageTypeRefV1
  binding_epoch: positive u64
  previous_binding_ref:
    null | exact SessionTransportConnectionBindingRefV1
```

Participantの初版だけ`participant_epoch=1`かつ`previous_participant_ref=null`、後続は同じparticipantと同じ`{session_id,session_epoch}` lifecycle、current head ref、exact `N+1`を持つ。Transport connection集合またはrole変更時は旧Binding／Participant refをretireし、新Binding集合とParticipant epochをatomic publishする。Bindingも各logical bindingの初版だけepoch 1／previous null、後続はcurrent headとexact `N+1`である。Participantと全Bindingの`{session_id,session_epoch}`は包含Session lifecycleとbyte equalityにし、旧session epochのParticipant／Bindingを新lifecycleへ再束縛しない。Pairはparticipant Connection Identityをexact一件含み、相手向きIdentityをparticipant自身へ誤束縛しない。複数Connectionはshard／handoff／redundant path等のProfile-declared routeだけに使い、同じmessageをarrival順で競合適用しない。route policyはmessage type／authority scopeごとにexact一経路または明示的冗長deduplicationを決め、permitted message type unionはParticipantに必要なProfile集合とset equalityにする。Authority leaseのcurrent集合はSessionの`authority_binding.active_authority_lease_refs[]`が所有し、Participant identityへ埋め戻さない。participant IDをIP、socket、connection ID、Account名から導出しない。external identity bindingはAccount／entitlement Serviceが将来提供するopaque exact Refで、role／authorityを自動付与しない。

```text
MultiplayerSessionV1
  session_id: StableId
  session_epoch: positive u64
  profile_ref: exact MultiplayerProfileRefV1
  runtime_entry_refs[1..capacity.maximum_runtime_entries]:
    exact RuntimeEntryRefV1
  target_profile_refs[1..64]: exact TargetProfileRefV1
  state: MultiplayerSessionStateV1
  state_sequence: positive u64
  previous_session_ref:
    null | exact MultiplayerSessionRefV1
  authority_binding:
    { state: unassigned }
    | { state: assigned,
        authority_epoch: positive u64,
        active_authority_lease_refs[
          1..capacity.maximum_authority_leases]:
          exact AuthorityLeaseRefV1 }
  baseline_binding:
    { state: unavailable }
    | { state: current,
        baseline_ref: exact ReplicationBaselineRefV1,
        baseline_epoch: positive u64 }
  participant_refs[0..capacity.maximum_participants]:
    exact MultiplayerParticipantRefV1
  diagnostic_refs[0..128]
  session_content_hash: SHA-256
```

Sessionの`target_profile_refs[]`は、standalone Profileでは単一Target、product bundle ProfileではProfileが投影したclient／authority role Targetの重複除去unionとset equalityにする。`runtime_entry_refs[]`はTopologyが要求するauthority／client Runtime Entry closureのexact集合で、各EntryのTargetはこのTarget集合へ解決する。listen topologyでは同じRuntime Entryがclient／authority roleを満たせるが重複refを持たず、distributed topologyを一つの代表Entryへ潰さない。operations等の非Multiplayer role Target、近いkind、別promotion subjectのTargetを追加しない。

許可遷移を次に固定する。

| From | To | 必須条件 |
|---|---|---|
| created | awaiting_transport | Profile／Runtime Entry／Target closure完成 |
| awaiting_transport | authenticating_binding | required Transport connectionがactive |
| authenticating_binding | joining | participant／identity／role binding検証成功 |
| joining | synchronizing | join admissionとauthority policy承認 |
| synchronizing | active | exact baseline、World／ECS generation、time mapping、ack完成 |
| active | resynchronizing | baseline gap／desyncがProfileの回復範囲内 |
| resynchronizing | active | new baseline epochとstate convergence Receipt |
| active | migrating_authority | qualified migration／handoff policy開始 |
| migrating_authority | active | new authority epoch、lease cutover、baseline、ack完成 |
| 任意非終端 | leaving | local／remote／authority-approved leave |
| leaving | left | authority／object／transport binding retirement完了 |
| 任意非終端 | fault | non-recoverable desync、authority loss、security／policy violation |

初回lifecycleのgenesisだけ`session_epoch=1`、`state_sequence=1`、`previous_session_ref=null`とする。同じepochの後続Snapshotはcurrent head refとstate sequence exact `N+1`を持ち、single-writer CASで一件だけ進める。同じsession IDを新しいlifecycleへ再利用する場合は前lifecycleのterminal headを`previous_session_ref`へ持ち、`session_epoch`をexact `N+1`、`state_sequence=1`にしたnew chainとする。new epoch genesisをnull parentにせず、同じepochのsequenceを1へ戻さない。

`authority_binding=unassigned`と`baseline_binding=unavailable`はactive authority／baselineの不存在を表し、架空のepoch値を予約しない。`synchronizing | active | resynchronizing | migrating_authority`はassigned authorityを必須とする。`synchronizing`はbaseline unavailableを許すが、発行予定Baselineのauthority epochをassigned authority epochへ固定し、Transport peerまたはmessage senderからauthorityを補完しない。`active`は同じ`{session_id,session_epoch}` lifecycle／authority epochへ解決するcurrent Baselineを必須にする。assigned lease集合はそのSnapshotと同じsession lifecycleかつ同authority epochで有効なLeaseのexact集合、participant集合と全Transport Bindingも同じsession lifecycle、baseline Fieldは参照先Baselineのsession lifecycle／authority epoch／baseline epochとexact一致しなければならない。表にない遷移、`left | fault`からの同一epoch resume、Transport reconnectだけによる`active`復帰、baseline未完成の`active`、同じauthority epochでのhandoffをrejectする。

## 5. Network Object identityとECS／World binding

```text
NetworkObjectIdentityV1
  network_object_id: StableId
  object_epoch: positive u64
  session_id: StableId
  session_epoch: positive u64
  spawn_sequence: positive u64
  previous_network_object_ref:
    null | exact NetworkObjectRefV1
  authority_scope_ref: exact AuthorityScopeRefV1
  runtime_binding:
    { kind: ecs_entity,
      generation_ref: exact RuntimeEntityGenerationRefV1 }
    | { kind: world_object,
        generation_ref: exact WorldObjectGenerationRefV1 }
    | { kind: session_object,
        generation_ref: exact SessionObjectGenerationRefV1 }
  persistence_binding_ref:
    optional exact PersistentIdentityBindingRefV1
  network_object_content_hash: SHA-256
```

union外runtime branchを禁止し、pointer、Entity index、World path、display name、spawn packet sequenceだけをidentityにしない。初回spawnだけ`object_epoch=1`かつ`previous_network_object_ref=null`、despawn後の同じnetwork object ID再利用はterminalなcurrent Object ref、object epoch exact `N+1`、増加したspawn sequence、明示spawn／migrationなしに禁止する。non-null `previous_network_object_ref`の解決先は同じ`network_object_id`と同じ`{session_id,session_epoch}` lifecycleを持たなければならず、別session epochでは新しいObject chainをprevious nullから開始する。`MultiplayerSessionV1.previous_session_ref`だけがnew session lifecycle genesisで許すcross-lifecycle edgeであり、Network Object chainへ一般化しない。current write authorityはSessionのauthority epochと`AuthorityLeaseV1`からscopeで解決し、mutable lease refをObject identityへ埋め戻さない。

```text
AuthorityLeaseV1
  lease_id: StableId
  lease_version: positive u32
  session_id: StableId
  session_epoch: positive u64
  authority_epoch: positive u64
  holder_participant_ref:
    exact MultiplayerParticipantRefV1
  authority_scope_ref: exact AuthorityScopeRefV1
  permitted_operation_refs[1..256]:
    exact AuthorityOperationRefV1
  valid_advance_range:
    closed SimulationAdvanceRangeV1
  handoff_predecessor_ref:
    optional exact AuthorityLeaseRefV1
  lease_content_hash: SHA-256
```

Authority scopeはsession、World region／Cell set、object set、Component／property class、command familyのclosed tagged unionである。overlapはAuthority Policyがexplicit shared-read／single-writeまたはtransaction protocolを定義する場合だけ許可する。二つのwrite lease、expired range、wrong epoch、unknown operationをrejectする。

ECS structural changeは本書のspawn／despawn／state commandをScheduling publication boundaryへ渡し、ECS ownerのStructural Transactionとしてだけ適用する。Network ObjectがECS storage layout、query、pointer、archetypeを保存しない。

## 6. Replication Schema、state、event、typed command

```text
ReplicationSchemaV1
  schema_id: StableId
  schema_version: positive u32
  owner_ref: exact DomainOwnerRefV1
  source_type_ref: exact McdTypeRefV1
  field_bindings[1..256]:
    field_id: StableId
    source_field_ref: exact McdFieldRefV1
    replication:
      { kind:
          authoritative_state | owner_state | presentation_state,
        quantization_ref:
          optional exact QuantizationProfileRefV1,
        change_detection_ref:
          exact ChangeDetectionProfileRefV1,
        visibility_policy_ref:
          exact ReplicationVisibilityPolicyRefV1 }
      | { kind: event_only,
          event_schema_ref:
            exact ReplicatedEventSchemaRefV1 }
      | { kind: never_replicate }
  command_refs[0..128]: exact ReplicatedCommandSchemaRefV1
  event_refs[0..128]: exact ReplicatedEventSchemaRefV1
  schema_content_hash: SHA-256
```

`replication`はclosed tagged unionで、union外Fieldを禁止する。state三branchだけがquantization／change detection／visibility Fieldを持ち、`event_only`は同じSchemaの`event_refs[]`に含まれるexact event一件、`never_replicate`は追加payloadなしとする。private／credential／pointer／native handle／Editor-only／non-deterministic cache fieldは`never_replicate`またはSchema外とし、reflectionで全Fieldを自動列挙しない。field ID、unit、quantization、authority、visibility、Target limitをexplicitにする。

```text
ReplicatedCommandSchemaV1
  command_type_id: StableId
  command_version: positive u32
  payload_schema_ref: exact McdTypeRefV1
  sender_role_set[1..16]: MultiplayerParticipantRoleV1
  required_authority_ref: exact AuthorityRequirementRefV1
  validation_policy_ref: exact CommandValidationPolicyRefV1
  idempotency:
    { kind: idempotent_with_key,
      key_field_ref: exact McdFieldRefV1,
      session_binding: envelope_session_lifecycle,
      authority_epoch_binding: envelope_authority_epoch,
      deduplication_scope:
        { kind: per_participant }
        | { kind: per_object,
            network_object_ref_field_ref:
              exact McdFieldRefV1 }
        | { kind: per_authority_scope,
            authority_scope_ref_field_ref:
              exact McdFieldRefV1 }
        | { kind: per_session },
      retention_policy_ref:
        exact BoundedDeduplicationRetentionPolicyRefV1 }
    | { kind: non_idempotent }
  ordering_requirement:
    per_sender | per_object | per_authority_scope
  expiry_policy_ref: exact CommandExpiryPolicyRefV1
  rate_size_limit_ref: exact CommandRateSizeLimitRefV1
  transport_delivery_class:
    TransportDeliveryClassV1
  command_content_hash: SHA-256

ReplicatedCommandEnvelopeV1
  command_invocation_id: StableId
  envelope_version: positive u32
  command_schema_ref: exact ReplicatedCommandSchemaRefV1
  session_id: StableId
  session_epoch: positive u64
  authority_epoch: positive u64
  sender_participant_ref: exact MultiplayerParticipantRefV1
  payload_bytes:
    bounded MCD canonical bytes matching
    command_schema_ref.payload_schema_ref
  payload_content_hash: SHA-256
  envelope_content_hash: SHA-256

ReplicatedCommandEnvelopeRefV1
  command_invocation_id: StableId
  envelope_version: positive u32
  envelope_content_hash: SHA-256
```

`idempotency`はclosed tagged unionで、union外Fieldを禁止する。`idempotent_with_key`の`session_binding=envelope_session_lifecycle`はEnvelopeの`{session_id, session_epoch}`、`authority_epoch_binding=envelope_authority_epoch`はEnvelopeの`authority_epoch`だけをdeduplication domainに使用する。key Fieldはpayload Schema内のexact一Fieldへ解決し、その型／canonical encodingを含める。scope identityは`per_participant`ならEnvelopeのsender、`per_object`なら指定Fieldのexact Network Object ref、`per_authority_scope`なら指定Fieldのexact Authority Scope ref、`per_session`ならEnvelopeのsession lifecycleであり、inactive branch Fieldを禁止する。deduplication identityを`{command_schema_ref, session_id, session_epoch, authority_epoch, scope identity, key canonical bytes}`へ閉じ、bounded retention内のbyte equalityだけをduplicateにする。Receiverはcommand名、Field名、participant／object IDまたはsequence慣習からkey／scopeを推測しない。

`envelope_content_hash`は自身を除くCommand Envelope全FieldのMCD canonical bytesに対するSHA-256で、Schema、session lifecycle、authority epoch、sender、payload bytes／hashをすべて封印する。Refの三Fieldは完成Envelopeとbyte equalityにする。同じsession lifecycleのcurrent assigned authority epochと一致しないCommandはdeduplication前にstaleとしてrejectし、旧epochのkeyをnew epochへ持ち越さない。同じ`command_invocation_id`でRefがbyte equalityならEnvelope duplicate、versionまたはcontent hashが異なればintegrity faultとする。RPCはこのtyped command／event Schemaのpresentation名であり、任意native function call、function pointer、reflection dispatchではない。remote position、inventory、damage、combat result、currency、Save stateを受信しただけでauthoritative stateにしない。

`ReplicatedEventSchemaV1`はevent identity、producer authority、recipient visibility、ordering、expiry、deduplication、replay disposition、delivery classを持つ。eventを永続stateまたはstate deltaとして再解釈しない。

## 7. Snapshot、delta、baseline、ack

```text
ReplicationBaselineV1
  baseline_id: StableId
  baseline_epoch: positive u64
  previous_baseline_ref:
    null | exact ReplicationBaselineRefV1
  session_id: StableId
  session_epoch: positive u64
  authority_epoch: positive u64
  source_advance_sequence: positive u64
  world_generation_ref: exact WorldGenerationRefV1
  ecs_generation_ref: exact RuntimeEcsGenerationRefV1
  object_state_refs[0..capacity.maximum_baseline_object_states]:
    exact NetworkObjectStateRefV1
  schema_set_hash: SHA-256
  baseline_content_hash: SHA-256

ReplicationStateEnvelopeV1
  replication_message_id: StableId
  envelope_version: positive u32
  session_id: StableId
  session_epoch: positive u64
  authority_epoch: positive u64
  source_advance_sequence: positive u64
  network_tick: bounded u64
  payload:
    { kind: snapshot,
      baseline_ref: exact ReplicationBaselineRefV1,
      snapshot_payload_ref: exact SnapshotPayloadRefV1 }
    | { kind: delta,
        base_baseline_ref: exact ReplicationBaselineRefV1,
        delta_payload_ref: exact DeltaPayloadRefV1 }
  recipient_projection_ref:
    exact RecipientProjectionRefV1
  schema_set_hash: SHA-256
  payload_content_hash: SHA-256
  envelope_content_hash: SHA-256

ReplicationStateEnvelopeRefV1
  replication_message_id: StableId
  envelope_version: positive u32
  envelope_content_hash: SHA-256
```

`envelope_content_hash`は自身を除くEnvelope全FieldのMCD canonical bytesに対するSHA-256で、session／authority epoch、advance／tick、snapshot／delta branch、Baseline、recipient projection、Schema set、payload hashをすべて封印する。Refの三Fieldは完成Envelopeとbyte equalityにし、payload hashまたはmessage IDだけからRefを生成しない。`recipient_projection_ref`の解決先`{session_id,session_epoch}`はEnvelopeの同pairとbyte equalityでなければならず、content hashが正しくても旧epoch Projectionを新lifecycleへ再束縛しない。Baselineの初版だけ`baseline_epoch=1`かつ`previous_baseline_ref=null`、後続は同じsessionのcurrent Baseline refとexact `N+1`を持つ。authority epoch cutover後の最初のBaselineも旧authorityのfinal Baselineをpreviousに保持し、epochを巻き戻さない。snapshot branchは同じsession／authority／source advanceへ解決する完成Baseline、delta branchは適用元のexact Baseline ID／epoch／hashを持つ。近いbaseline、最新baseline、同じtickの別recipient baselineを代用しない。baseline missing／stale／authority epoch mismatchでは適用せずresyncを要求する。union外payload、partial object setをfull baselineとして扱わない。

```text
ReplicationAcknowledgementV1
  acknowledgement_id: StableId
  session_id: StableId
  session_epoch: positive u64
  participant_ref: exact MultiplayerParticipantRefV1
  authority_epoch: positive u64
  baseline_epoch: positive u64
  received_replication_message_refs[
    0..capacity.maximum_acknowledged_messages]:
    exact ReplicationStateEnvelopeRefV1
  applied_advance_sequence: positive u64
  missing_range_refs[0..capacity.maximum_missing_ranges]:
    exact ReplicationMissingRangeRefV1
  acknowledgement_content_hash: SHA-256
```

ackはtransport delivery ackではなく、recipientがSchema／authority／baseline validation後にapplication boundaryへ適用した状態を表す。Transport ackからReplication ackを推測しない。

## 8. Relevancy、interest、dormancy、priority、bandwidth

```text
RecipientProjectionV1
  projection_id: content-derived StableId
  session_id: StableId
  session_epoch: positive u64
  recipient_participant_ref:
    exact MultiplayerParticipantRefV1
  authority_epoch: positive u64
  source_advance_sequence: positive u64
  interest_profile_ref: exact InterestProfileRefV1
  relevant_object_refs[0..capacity.maximum_projection_objects]:
    exact NetworkObjectRefV1
  dormant_object_refs[0..capacity.maximum_projection_objects]:
    exact NetworkObjectRefV1
  selected_object_refs[0..capacity.maximum_projection_objects]:
    exact NetworkObjectRefV1
  priority_records[0..capacity.maximum_projection_objects]:
    network_object_ref: exact NetworkObjectRefV1
    priority_class: registered bounded class
    age_class: registered bounded class
  omission_reason_records[0..capacity.maximum_projection_objects]:
    ReplicationOmissionReasonRecordV1
  projection_content_hash: SHA-256
```

set algebra、interest評価結果のseal、`projection_content_hash`計算またはpublicationより前に、`recipient_participant_ref`の解決先と、`relevant_object_refs[]`、`dormant_object_refs[]`、`selected_object_refs[]`、`priority_records[].network_object_ref`、`omission_reason_records[]`のobject projectionに現れる全`NetworkObjectRefV1`の解決先について、`{session_id,session_epoch}`をRecipient Projectionの同pairとbyte equalityにする。旧epoch Refはexact Ref／content hash／deterministic interest result／後続集合関係が正しくてもrejectし、同じ`network_object_id`または表示上のsession名からcurrent epochへrebaseしない。各object集合はuniqueとする。`dormant_object_refs[]`は`relevant_object_refs[]`のsubset、`selected_object_refs[]`は`relevant - dormant`のsubsetである。priority recordのobject集合は`relevant - dormant`、omission reasonのobject集合は`relevant - selected`とそれぞれset equalityにする。snapshot／delta payloadが当該recipientへ含めるobject集合は`selected_object_refs[]`とexact一致し、dormantまたはomitted objectを暗黙送信しない。relevancyはGameplay／World Ownerが提供するtyped scope、visibility policy、participant role、authority、distance class等のregistered inputだけで決める。Render visibility、occlusion query、Camera frustumだけをGameplay relevancy authorityにしない。interest resolverはsame inputからsame set／orderを返し、hash map順、packet budget後の偶然、arrival順を使わない。

dormancyはstate unchanged／wake conditionのreplication optimizationで、object deletion、World unload、authority removalを意味しない。priorityはbounded classとage ruleで、starvation limitとomission reasonを持つ。bandwidth envelope不足時はProfileのbackpressure／defer／reduced-frequency／session faultから選び、fieldをsilent dropまたはquantization変更しない。

## 9. Network time mapping

```text
NetworkTimeMappingProfileV1
  profile_id/version/content_hash
  simulation_cadence_profile_ref:
    exact SimulationCadenceProfileRefV1
  authoritative_tick_mapping_ref:
    exact AuthoritativeTickMappingRefV1
  presentation_interpolation_ref:
    optional exact PresentationInterpolationProfileRefV1
  clock_sample_policy_ref:
    exact BoundedClockSamplePolicyRefV1
  discontinuity_policy_ref:
    exact NetworkTimeDiscontinuityPolicyRefV1

NetworkTimeSnapshotV1
  session_id: StableId
  session_epoch: positive u64
  authority_epoch: positive u64
  local_advance_sequence: positive u64
  estimated_authoritative_advance_sequence: positive u64
  network_tick: bounded u64
  interpolation_alpha: optional finite normalized scalar
  uncertainty_ref: exact BoundedTimeUncertaintyRefV1
  discontinuity_reason:
    optional registered reason
```

network tickはSimulation Advanceの別authorityではない。[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)のcadenceとexplicit mappingを持つ。packet arrival time、wall clock、render frame、audio timeからauthoritative tickを直接決めない。uncertainty上限超過、authority epoch変更、Replay seek、pause／time discontinuityではProfileに従いresync／interpolation reset／session faultへ進める。

## 10. Prediction、reconciliation、rollback、resimulation

predictionとrollbackはProfile内の独立subprofileである。

```text
PredictionReconciliationProfileV1
  profile_id/version/content_hash
  predictable_command_refs[1..128]:
    exact ReplicatedCommandSchemaRefV1
  predictable_state_refs[1..256]:
    exact ReplicationFieldRefV1
  input_snapshot_ref: exact InputSnapshotSchemaRefV1
  prediction_horizon_ref: exact BoundedAdvanceRangeRefV1
  reconciliation_policy_ref:
    exact ReconciliationPolicyRefV1
  correction_visibility_ref:
    exact CorrectionVisibilityPolicyRefV1
  non_predictable_field_policy:
    authoritative_only

RollbackResimulationProfileV1
  profile_id/version/content_hash
  rollback_state_schema_ref:
    exact RollbackStateSchemaRefV1
  rollback_window:
    minimum_advances: bounded u32
    maximum_advances: bounded u32
  snapshot_interval_ref:
    exact RollbackSnapshotIntervalRefV1
  resimulation_system_set_ref:
    exact ResimulationSystemSetRefV1
  deterministic_input_set_ref:
    exact DeterministicInputSetRefV1
  external_effect_policy_ref:
    exact DeferredExternalEffectPolicyRefV1
  overflow_disposition:
    resync | disconnect | session_fault
```

rollback stateはECS／Gameplay Ownerが明示したField／Systemだけを含み、Renderer、Audio device、network queue、pointer、resident resource、wall clock、random global stateをsnapshotしない。resimulation system setはScheduling access manifestとECS access closureへexact解決する。

`minimum_advances <= maximum_advances`を必須にし、Target／capacity policyがminimumを満たせないProfileはrollback `unsupported`とする。実行時pressureからwindowをminimum未満へsilent縮小しない。

予測commandはlocal proposalとして記録し、authority acceptance前にSave／persistent economy／external side effectへcommitしない。authoritative resultとの差はProfileのreconcile policyでcorrectionし、予測値を正しいものとしてserverへ上書きしない。

rollback window外、missing Input、state hash mismatch、non-deterministic System、external effect already committedではsilent partial rewindを禁止し、Profileのoverflow dispositionへ進める。small co-op、large-session shooter、MMOへrollbackを暗黙必須にしない。

## 11. Join、late join、resync、reconnect

```text
MultiplayerJoinRequestV1
  join_request_id: StableId
  session_template_ref: exact MultiplayerSessionTemplateRefV1
  participant_binding_ref:
    exact ProposedParticipantBindingRefV1
  requested_role: MultiplayerParticipantRoleV1
  transport_connection_pair_identity_ref:
    exact TransportConnectionPairIdentityRefV1
  requester_connection_identity_ref:
    exact TransportConnectionIdentityRefV1
  client_schema_set_hash: SHA-256
  client_profile_ref: exact MultiplayerProfileRefV1
  join_request_content_hash: SHA-256
```

requester IdentityはPairのexact memberで、Join messageのsender Identityと一致しなければならない。admissionはProfile／Schema／Target／role／identity binding／capacity／security／authority policyを検証する。Lobby membership、connected socket、possessed local object、matching display nameをadmissionにしない。

late joinはcurrent authority epoch、exact baseline、World／ECS generation、object spawn set、time mappingを取得し、全required ack後に`active`へ進む。snapshotの一部到着でactiveにしない。

resyncは新baseline epochを発行し、recipientの旧delta chain、prediction history、rollback history、dormancy cacheをProfileに従い封印／resetする。同じbaseline IDの内容変更、missing objectの近似補完、server／clientの多数決を禁止する。

Transport reconnect後はnew `SessionTransportConnectionBindingV1`、participant epoch／session ticket validation、authority epoch check、baseline resyncを必須にする。old connection epoch messageを受理せず、participantが既にleft／revokedならnew joinを別requestとして扱う。

## 12. Host migrationとauthority handoff

host migrationは`listen_server | peer_authority`で明示QualificationされたProfileだけが使用できる。dedicated authorityのreplacement、sharded region handoffも同じ「authority epoch cutover」原則を使うが、別Topology fixtureとoperations closureを必要とする。

```text
AuthorityHandoffPlanV1
  handoff_id: StableId
  session_id: StableId
  session_epoch: positive u64
  source_authority_epoch: positive u64
  destination_authority_epoch: positive u64
  source_authority_lease_refs[
    1..capacity.maximum_handoff_leases]:
    exact AuthorityLeaseRefV1
  destination_participant_ref:
    exact MultiplayerParticipantRefV1
  destination_authority_lease_refs[
    1..capacity.maximum_handoff_leases]:
    exact AuthorityLeaseRefV1
  cutover_advance_sequence: positive u64
  baseline_ref: exact ReplicationBaselineRefV1
  participant_ack_refs[
    1..capacity.maximum_handoff_acknowledgements]:
    exact ReplicationAcknowledgementRefV1
  failure_disposition:
    abort_to_source | session_fault
  authorization_ref: exact AuthorityHandoffAuthorizationRefV1
  handoff_content_hash: SHA-256
```

`destination_authority_epoch = source_authority_epoch + 1`をoverflowなしで必須にする。source／destination leaseのauthority scope集合はset equalityで、destination側は同じscopeを新epoch／destination holderへ完全写像する。participant ack集合はHandoff Policyが要求するcurrent participant集合とset equalityにする。cutoverまではsource leaseだけ、cutover後はdestination leaseだけがwrite authorityを持つ。両方、どちらもないgap、同じepoch、partial scope、stale baseline、missing／extra participant ackをrejectする。source failure後にcomplete baseline／authorizationがなければ別peerを自動選択せずsession faultにする。

participantのlatency、hardware性能、接続順、Account名を単独でhost選択authorityにしない。election policyを採択する場合はexact bounded Profile、tie-break、security／capacity input、negative fixtureを必要とする。

## 13. Persistence、Replay、Debug、Security handoff

[Persistence／Save](../04-runtime/persistence-save.md)はpersistent identity、save state、reconstructionを所有する。本書はsession／participant／network object／authority／baselineのephemeral identityと、Saveへ投影可能なexact persistent bindingだけを渡す。connection、queue、ack、prediction buffer、rollback snapshot、dormancy cacheをSaveしない。

[Debugging／Observability／Replay](../04-runtime/debugging-observability-replay.md)はcapture／causality／Replay envelopeを所有する。本書はauthority decision、validated command、state hash、baseline／delta relation、correction、rollback／resim、handoff、diagnosticをredacted eventとして渡す。credential、raw packet、private identity、secret、full player inputを無条件に記録しない。

[Product Security](../01-governance/product-security.md)はanti-cheat、abuse、incident、vulnerability caseを所有する。本書はsender role、authority、Schema、rate／size、expiry、idempotency、state transitionを検証し、invalid／malicious commandをauthoritative stateへ適用しない。Transport encryption、client prediction result、local files、client timestamp、replicated visibilityをtrust proofにしない。

## 14. Failure、fallback、diagnostic

最低限、次を別diagnosticにする。

- Profile／Topology／Target／Schema mismatch
- participant／session／connection／object／authority epoch stale
- illegal session transition、join admission denied、late-join baseline incomplete
- unknown／duplicate object、spawn／despawn sequence violation
- wrong sender role／authority lease／scope／operation
- malformed／oversized／expired／rate-exceeded command、idempotency collision
- missing／stale baseline、delta chain gap、schema set hash mismatch
- relevancy／interest input invalid、dormancy wake failure、priority starvation
- network time uncertainty／discontinuity、tick／advance mapping mismatch
- prediction horizon／rollback window overflow、state hash mismatch、nondeterministic resim
- reconnect binding mismatch、resync failure、authority loss
- handoff dual authority／authority gap／baseline／ack／authorization failure
- capacity／bandwidth policy exceeded、security binding revoked

fallbackはProfileのclosed branchに限る。

| failure | 許可候補 | silentにしてはいけない変更 |
|---|---|---|
| prediction unavailable | authoritative-only presentation | authority model、state meaning |
| rollback unavailable／overflow | resync／disconnect／session fault | snapshot方式への暗黙変更、state捏造 |
| late join baseline failure | retry bounded resync／join reject | partial active |
| listen host loss | qualified host migration／session fault | arbitrary peer election |
| bandwidth pressure | defer／frequency profile／reject／fault | unregistered field drop、quantization change |
| authority handoff failure | abort before cutover／session fault | dual writer、epoch reuse |

fallbackでdedicated→listen、server→client authority、multiplayer→local simulationを同じsession identityのまま変更しない。

## 15. AI／Editor理解境界

AI／Editor projectionはTopology、participant role、session state、authority scope／epoch、Network Object binding、Schema kind、baseline／delta、relevancy／priority、time mapping、prediction／rollback、join／resync／handoff、Diagnostic、Target Qualificationをboundedに示す。raw packet、credential、private identity、full state／input、memory pointer、native callbackを公開しない。

`create session`、`join`、`leave`、`replicate`、`resync`、`migrate host`、`explain authority`はplanned semantic vocabularyで、current MCD Operation、Service、Tool、Provider surfaceではない。完全なAuthority／Schema／Validator／Receipt／publication closureが承認されるまでaction名からIDを生成またはdispatchしない。

AIは「multiplayer」「co-op」「competitive」「MMO」「server authoritative」からTopology、player count、rollback、tick、hosting、Lobby、anti-cheat、Targetを自動選択しない。Product goal、authority、join／failure、latency／scale、persistence、Target／operationsを別axisとして説明する。

## 16. CapacityとQualification

shared CPU／memory／bandwidth／latency measurement、budget、backpressureは[Performance／Capacity](../04-runtime/performance-capacity.md)が所有する。本書はparticipant／object／message／baseline／history count、snapshot／delta／rollback demand、selected policy、reasonをowner-typed measurementとして返す。本文の上限はportable safety bound候補で、production capacity claimではない。

Qualification subjectは少なくとも`{Multiplayer Profile ref, Topology Profile ref, Replication Schema set hash, Target Profile refs, Transport Profile／Qualification Binding refs, Simulation Cadence Profile ref, ECS／World schema generation refs, fixture set ref}`へ閉じる。一Topology／Target／Schema／tick／Transport tupleのReceiptを別tupleへ流用しない。

fixtureは次を独立に含む。

1. dedicated、listen、peerと将来sharded／distributedのapplicable role／authority
2. join、leave、late join、disconnect、reconnect、resync
3. spawn／despawn、state／event／command、snapshot／delta／baseline／ack
4. relevancy、interest、dormancy、priority、bandwidth pressure
5. network time、interpolation、prediction、correction、rollback、resimulation
6. stale／out-of-order／duplicate／malformed／oversized／rate-abusive command
7. baseline gap、desync、rollback overflow、network partition、authority loss
8. qualified host migration／authority handoffとdual-writer／gap negative case
9. Save／Load、Replay seek、World Cell activation、Runtime Entry transition
10. small co-op、rollback competitive、large-session scaleを別Product fixtureとして保持すること
11. desktop、mobile、headlessのTarget／lifecycle／capacity差
12. wrong Owner／ID／version／hash、inactive union、count／sort、stale／revoked Receipt

small co-opのReceiptでrollback、large-session、MMOを、rollback Receiptでlarge-sessionを、headless package ReceiptでMultiplayerをsupportしない。

## 17. 公式資料から採用する構造と限界

- [Unreal Engine: Networking Overview](https://dev.epicgames.com/documentation/unreal-engine/networking-overview-for-unreal-engine?lang=en-US)はserver authority、client／server model、replicationを区別する。本書はauthorityとconnectionを別Contractにする。
- [Unreal Engine: Iris](https://dev.epicgames.com/documentation/en-us/unreal-engine/introduction-to-iris-in-unreal-engine)はreplication state、filtering、prioritization、serializer、data stream、gameplay bridgeを分離する。本書はSchema、projection、priority、Transport handoffを分ける。
- [Unreal Engine: Replication Graph](https://dev.epicgames.com/documentation/en-us/unreal-engine/replication-graph-in-unreal-engine)は多数object／connection向けrelevancy処理を専用構造にする。本書はRecipient Projectionをexplicitにする。
- [Unity Netcode for GameObjects](https://docs.unity3d.com/jp/current/Manual/com.unity.netcode.gameobjects.html)はhigh-level Netcodeがunderlying transportの上にあることを示す。本書はReplicationとTransportを別Ownerにする。
- [Godot: High-level multiplayer](https://docs.godotengine.org/en/4.7/tutorials/networking/high_level_multiplayer.html)はpeer、RPC authority／delivery、dedicated export、authenticationを別関心として示す。本書はtyped command、role authority、headless Targetを別Owner refへ閉じる。

外部EngineのAPI名、Class、RPC annotation、replication algorithm、tick、既定値、package version、Service topologyをMiraikanaiのcanonical IDまたは採用要件にしない。比較から採用するのはauthority、replication、interest、Transport、Target、Serviceの責務分離だけである。

## 18. 明示的非目標と設計Closure

非目標は、Transport、Online Identity／Lobby／Matchmaking／Hosting／Persistence Serviceの統合、特定Netcode／Protocol／Providerの選定、すべてのgameでrollbackを要求すること、MMO／large-session／anti-cheat／operationsの実装または対応主張、実装Task／順序／担当／工数／日程の定義である。

本書の設計Closureは次をすべて満たす場合だけ成立する。

1. Transport connectionとgameplay session、participant、Network Object、authorityが別identity／epochである。
2. Topology、role、scope、lease、state／event／command、baseline／deltaがclosed Schemaで機械判定できる。
3. Scheduling／ECS／World／Input／Persistence／Replay／Securityのauthorityが既存Ownerに残る。
4. predictionとrollbackがoptional subprofileで、authoritative stateを置き換えない。
5. reconnect、resync、host migration、authority handoffが別protocolで、dual writer／authority gapを許さない。
6. Online Servicesを暗黙採択せず、opaque exact bindingだけで将来接続できる。
7. 本書の存在をMultiplayer、Dedicated Server、rollback、large-session、MMO、Online対応済みの根拠にしない。
