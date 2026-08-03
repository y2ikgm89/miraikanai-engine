# Miraikanai Engine Network Transport／Connection Contract

- 文書ID: mirakan.arch.network-transport-connection
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: endpoint／listener／dial、connection identity、handshake／protocol negotiation、connection lifecycle、semantic delivery class、message envelope／packetization、ordering／reliability／duplication／loss／fragmentation、flow／congestion／backpressure、transport security binding、private Provider Adapter、transport diagnostic／fault injection／telemetry／Target qualification
- 非正本範囲: gameplay session／join／leave／player role、gameplay authority／replication／RPC meaning／prediction／rollback、Account／identity／entitlement、party／lobby／matchmaking、fleet hosting／region allocation、cloud persistence／economy／moderation、Simulation cadence、ECS state、Product security incident、shared capacity、Evidence envelope、実装Task／順序。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Product Security](../01-governance/product-security.md)
- 関連文書: [Product Plan](../00-product/product-plan.md)、[Advanced Rendering／Multiplayer Ownership Decision](../decisions/2026-07-29-advanced-rendering-multiplayer-ownership.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Runtime ECS](../04-runtime/entity-component-system.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[Debugging／Observability／Replay](../04-runtime/debugging-observability-replay.md)、[Multiplayer Authority／Replication](multiplayer-authority-replication.md)、[Windows](../07-platform/windows.md)、[Mobile Common](../07-platform/mobile-common.md)、[Android](../07-platform/android.md)、[Apple](../07-platform/apple.md)
- 根拠区分: project-decision／official-documentation comparison（未計測のMTU、timeout、retry、window、queue、bandwidth、connection countはprovisional）
- 外部根拠確認日: 2026-07-29

## 1. 結論、状態、layer境界

本書は「bounded message bytesを接続相手へ運ぶ」責務だけを所有する。endpointのlisten／dial、handshake、protocol revision、semantic delivery、connection／reconnect、packet／fragment、flow control、transport security、Provider Adapterを扱うが、messageがgameplay上で何を意味するか、誰がauthoritativeか、sessionへjoin済みかを判断しない。

[Multiplayer Authority／Replication](multiplayer-authority-replication.md)はtyped command／event／snapshot／delta／ackとgameplay sessionを所有し、本書へ`unreliable_latest | unreliable_sequenced | reliable_ordered | reliable_unordered`のsemantic delivery requirementを渡す。本書はreplicated object、Component、player、RPC function、Game System、rollback stateを知らない。

Account、platform identity、entitlement、party、lobby、matchmaking、backfill、fleet hosting、region placement、autoscale、cloud persistenceはOnline Servicesの別将来境界である。本書はそれらが将来発行するopaqueでexactなendpoint／credential／peer-auth bindingを任意入力として受けられるが、Serviceの存在を前提にしない。Relayをreplication、Lobbyをgameplay session、Matchmakerをtransportと呼び替えない。

[Product Plan](../00-product/product-plan.md)の`FutureTargetClosureRegistryV1`がcross-target Product consumerのclient／authority roleへ本Futureを束縛する。本書はroleごとに渡されたexact `TargetProfileRefV1`でTransport ProfileとQualificationを解決するだけで、role集合、topology、Dedicated Target、Product claimを所有しない。一TargetのTransport Receiptを別role／Targetへ流用しない。

本書の型とFuture entryは`review`／`absent`／`planning_only`である。current network Capability、Target support、Provider、socket、server、Service、実装はRepositoryに存在しない。

## 2. 不変条件とclosed語彙

1. connected transport、authenticated peer binding、joined gameplay session、spawned player、synchronized state、authoritative participationを別状態にする。
2. public ContractへIP address、socket handle、native channel number、Provider object、certificate private key、packet pointerを出さない。
3. delivery classはsemantic requirementで、特定Protocol／Provider／channel番号を意味しない。Adapterはexact Transport ProfileによりTargetへ写像する。
4. reliableはbounded retry／queue／expiry内のtransport guaranteeであり、無制限memory、eventual gameplay success、exactly-once gameplay effectを意味しない。
5. message identity、connection epoch、sequence space、fragment set、protocol revision、Target／Provider generationをexactに束縛し、reconnect後の旧epoch dataを受理しない。
6. handshake成功だけでpeerのAccount ownership、player entitlement、anti-cheat trust、gameplay authorityを保証しない。
7. timeout、loss、duplication、reorder、fragmentation、congestion、backpressureを明示状態／diagnosticにし、silent drop／retry storm／unbounded queueを許さない。
8. direct、platform network、relay、tunnelはprivate Provider Adapter候補である。同じsemantic conformanceとTarget Qualificationなしに相互fallbackしない。

```text
TransportEndpointRoleV1 =
  listener
  | dialer
  | accepted_peer

TransportDeliveryClassV1 =
  unreliable_latest
  | unreliable_sequenced
  | reliable_ordered
  | reliable_unordered

TransportConnectionStateV1 =
  configured
  | resolving
  | dialing
  | handshaking
  | active
  | reconnect_wait
  | closing
  | closed
  | fault

TransportCloseReasonV1 =
  local_request
  | remote_request
  | handshake_rejected
  | protocol_mismatch
  | authentication_failed
  | timeout
  | keepalive_failed
  | backpressure_limit
  | malformed_message
  | credential_revoked
  | provider_fault
  | target_lifecycle
  | superseded_epoch
```

unknown enum、inactive union payload、state／reason不一致をrejectする。`fault`と`closed`は別終端で、faultから同じepochをresumeしない。

## 3. Transport Profile、endpoint、Provider binding

```text
NetworkTransportDocumentV1
  document_id: StableId
  source_revision: positive u64
  profile_refs[1..64]: NetworkTransportProfileRefV1
  endpoint_binding_refs[0..1024]:
    TransportEndpointBindingRefV1
  target_policy_ref: exact TransportTargetPolicyRefV1
  document_content_hash: SHA-256

NetworkTransportProfileV1
  profile_id: StableId
  profile_version: positive u32
  owner_ref: exact OwnerRefV1
  protocol_family_id: registered StableId
  supported_protocol_revisions[1..16]:
    TransportProtocolRevisionV1
  delivery_profiles[1..4]:
    TransportDeliveryProfileV1
  handshake_profile_ref: exact TransportHandshakeProfileRefV1
  security_profile_ref: exact TransportSecurityProfileRefV1
  packetization_profile_ref: exact PacketizationProfileRefV1
  flow_control_profile_ref: exact TransportFlowControlProfileRefV1
  lifecycle_profile_ref: exact TransportLifecycleProfileRefV1
  provider_binding_refs[1..16]:
    exact TransportProviderBindingRefV1
  target_profile_refs[1..64]: exact TargetProfileRefV1
  fallback_policy_ref: exact TransportFallbackPolicyRefV1
  profile_content_hash: SHA-256
```

delivery profileは四classのexact setで、class ordinal順へstrict sortする。Target policyは一部classを`unsupported`にできるが、欠落を別classへ変換しない。Profile version、protocol revision、Provider binding、security、packetization、flow、lifecycleのいずれかが変わればResolved PlanとConnection epochを新しくする。

```text
TransportEndpointBindingV1
  endpoint_binding_id: StableId
  binding_version: positive u32
  endpoint_role: TransportEndpointRoleV1
  target_profile_ref: exact TargetProfileRefV1
  address_binding:
    { kind: direct,
      endpoint_ref: exact OpaqueDirectEndpointRefV1 }
    | { kind: platform,
        endpoint_ref: exact OpaquePlatformEndpointRefV1 }
    | { kind: relay,
        endpoint_ref: exact OpaqueRelayEndpointRefV1 }
    | { kind: tunnel,
        endpoint_ref: exact OpaqueTunnelEndpointRefV1 }
  peer_auth_binding_ref:
    optional exact PeerAuthenticationBindingRefV1
  credential_ref:
    optional exact CredentialRefV1
  expiration_policy_ref:
    exact EndpointExpirationPolicyRefV1
  binding_content_hash: SHA-256
```

address branchのopaque Refをlog、Replay、AI projection、Saveへraw展開しない。hostname／IP／port／relay allocation／platform user IDをStable gameplay identityにしない。credentialはsecret storeのRefだけを持ち、bytesをSource／Packageへ保存しない。

```text
TransportProviderBindingV1
  binding_id/version/content_hash
  provider_lock_ref: exact ProviderLockRefV1
  adapter_artifact_ref: exact SignedArtifactRefV1
  target_profile_refs[1..64]: exact TargetProfileRefV1
  supported_endpoint_kinds[1..4]:
    direct | platform | relay | tunnel
  supported_delivery_classes[1..4]:
    TransportDeliveryClassV1
  capability_limit_ref: exact TransportCapabilityLimitRefV1
  qualification_binding_ref:
    exact TransportProviderQualificationBindingRefV1

TransportProviderGenerationV1
  provider_binding_ref: exact TransportProviderBindingRefV1
  target_profile_ref: exact TargetProfileRefV1
  runtime_gateway_instance_nonce: opaque bytes[32]
  generation: positive u64
  generation_content_hash: SHA-256

TransportProviderGenerationRefV1
  provider_binding_ref: exact TransportProviderBindingRefV1
  target_profile_ref: exact TargetProfileRefV1
  runtime_gateway_instance_nonce: opaque bytes[32]
  generation: positive u64
  generation_content_hash: SHA-256
```

Provider version、hash、license、取得元、build optionは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)だけが固定する。Provider型、result code、callback、thread、socketをEngine-owned Contractへ漏らさない。

`TransportProviderGenerationV1`はEngine Transport Gatewayが所有するprocess-local／wire-visible identityであり、Provider Adapterまたはpeerが発行しない。Gateway instance生成時にCSPRNG由来のfresh `runtime_gateway_instance_nonce`と`generation=1`を作り、同process内でAdapterを置換、再初期化またはresetするたびexact `N+1`へ進める。process再起動ではgenerationを継承せずfresh nonceへ変える。wall clock、PID、address、pointer、Provider名またはartifact timestampをnonce／generationへ使用しない。content hashはASCII domain `MIRAKAN_TRANSPORT_PROVIDER_GENERATION_V1`、self hashを除くclosed payload length、closed payloadの順で計算する。Refはbacking recordの全Fieldとbyte equalityにする。

## 4. Handshakeとprotocol negotiation

```text
TransportHandshakeOfferV1
  handshake_id: StableId
  connection_attempt_ref:
    exact TransportConnectionAttemptIdentityRefV1
  offerer_connection_nonce: bounded opaque bytes
  protocol_family_id: registered StableId
  supported_protocol_revisions[1..16]:
    TransportProtocolRevisionV1
  transport_profile_ref: exact NetworkTransportProfileRefV1
  target_profile_ref: exact TargetProfileRefV1
  provider_binding_ref: exact TransportProviderBindingRefV1
  provider_generation_ref: exact TransportProviderGenerationRefV1
  delivery_capability_set[1..4]:
    TransportDeliveryClassV1
  message_envelope_limit_ref: exact EnvelopeLimitRefV1
  peer_auth_offer_ref:
    optional exact PeerAuthenticationOfferRefV1
  offer_content_hash: SHA-256

TransportHandshakeAcceptanceV1
  handshake_id: StableId
  offer_refs[2]: exact TransportHandshakeOfferRefV1
  selected_protocol_revision:
    exact TransportProtocolRevisionV1
  selected_delivery_capability_set[1..4]:
    TransportDeliveryClassV1
  selected_security_binding_ref:
    exact TransportSecurityBindingRefV1
  negotiated_limit_ref:
    exact NegotiatedTransportLimitRefV1
  acceptance_content_hash: SHA-256
```

`offer_refs[]`は両側の異なるConnection Attempt、nonce、Profile、Target、Provider binding、Provider generationへ解決し、offerer Attempt ref順へ正規化する。Offerの`provider_generation_ref`は同じOfferのConnection Attemptが持つRefとbyte equalityでなければならない。両Offerの`handshake_id`はAcceptanceとexact一致する。negotiationはこのexact二Offerの共通集合とProfile policyから一意に選び、表示versionの最大値、Provider順、arrival順を使わない。共通revision、required delivery class、security binding、limitが欠ける場合は`protocol_mismatch | handshake_rejected`で閉じる。片側Offerの省略、同じOfferの重複、別attemptのAcceptance、unknown optional fieldのsilent ignoreを拒否する。

`PeerAuthenticationBindingRefV1`はtransport peer／credential／attestationのopaque exact bindingで、Account、player、entitlement、gameplay roleを表さない。将来Identity Serviceと接続する場合も、そのService Receiptを本書のConnection identityへ明示bindingするだけで、名前やtoken claimからgameplay authorityを推測しない。

## 5. Connection identity、epoch、state machine

```text
TransportConnectionAttemptIdentityV1
  connection_id: StableId
  connection_epoch: positive u64
  connection_attempt_id: StableId
  previous_attempt_ref:
    null | exact TransportConnectionAttemptIdentityRefV1
  endpoint_binding_ref: exact TransportEndpointBindingRefV1
  transport_profile_ref: exact NetworkTransportProfileRefV1
  target_profile_ref: exact TargetProfileRefV1
  provider_binding_ref: exact TransportProviderBindingRefV1
  provider_generation_ref: exact TransportProviderGenerationRefV1
  attempt_content_hash: SHA-256

TransportConnectionIdentityV1
  connection_attempt_ref:
    exact TransportConnectionAttemptIdentityRefV1
  peer_connection_attempt_ref:
    exact TransportConnectionAttemptIdentityRefV1
  handshake_acceptance_ref:
    exact TransportHandshakeAcceptanceRefV1
  protocol_revision: exact TransportProtocolRevisionV1
  provider_generation_ref:
    exact TransportProviderGenerationRefV1
  peer_auth_binding_ref:
    optional exact PeerAuthenticationBindingRefV1
  identity_content_hash: SHA-256

TransportConnectionPairIdentityV1
  connection_pair_id: content-derived StableId
  handshake_acceptance_ref:
    exact TransportHandshakeAcceptanceRefV1
  connection_identity_refs[2]:
    exact TransportConnectionIdentityRefV1
  provider_generation_refs[2]:
    exact TransportProviderGenerationRefV1
  pair_content_hash: SHA-256
```

`connection_id`はlogical relation、`connection_epoch`は一回のattempt／active transport lifetimeを識別する。最初のAttemptだけ`connection_epoch=1`かつ`previous_attempt_ref=null`、reconnectは同じconnection ID、previous Attemptのexact ref、epoch `N+1`を必須にする。AttemptのProvider Binding／TargetとProvider Generation Refの同名Fieldはbyte equalityでなければならない。Acceptance後、二Offerの各Attemptについて一件ずつ完成`TransportConnectionIdentityV1`を作り、その二件をAttempt ref順の`TransportConnectionPairIdentityV1`へ閉じる。identityのown／peer Attempt refはAcceptanceの二Offer Attempt集合とset equalityで、ownとpeerを同じrefにせず、own `provider_generation_ref`はown Offer／Attemptと、protocol／security／limitはAcceptanceとexact一致させる。Pairの`provider_generation_refs[]`は`connection_identity_refs[]`と同じAttempt ref順で、両Identityのgeneration集合およびAcceptanceの両Offer generation集合とset equalityにする。別Acceptance、同じ向きのIdentity重複、generation欠落／置換を拒否する。旧Provider generationまたは旧connection epochのcallback、packet、sequence、fragment、ack、credential、gameplay channel bindingを新generation／epochへ流用しない。

許可遷移を次に固定する。

| From | To | 必須条件 |
|---|---|---|
| configured | resolving | exact endpoint／Profile／Target／Provider refs |
| resolving | dialing | endpoint resolution成功、expiry内 |
| dialing | handshaking | provider connection成立 |
| handshaking | active | Acceptance、security、limit、identity closure完成 |
| resolving／dialing／handshaking | reconnect_wait | lifecycle policyがreasonとattemptを許可 |
| active | reconnect_wait | retry可能disconnect、queue／epoch封印完了 |
| reconnect_wait | resolving | bounded backoff後にnew Attempt／epochを作り、そのgenesis Snapshotでfresh endpoint／credentialを検証 |
| configured／resolving／dialing／handshaking／active／reconnect_wait | closing | local／target lifecycle close |
| closing | closed | send／receive closureとresource retirement完了 |
| 任意非終端 | fault | 非回復failure、integrity違反、policy超過 |

`reconnect_wait -> resolving`だけは同一Snapshot chain内のedgeでなく、旧Attemptのsealed `reconnect_wait` headを確認して`previous_attempt_ref`へ束縛したnew Attempt chainをatomicに開始するcross-attempt edgeである。初回Attemptのgenesis Snapshotは`configured`、reconnect Attemptのgenesis Snapshotはこのcross-attempt authorizationを持つ`resolving`とする。表にない遷移、終端からのresume、同じepochの`active`再入、`fault -> reconnect_wait`をrejectする。retry回数、backoff、timeoutはProfileのbounded policyであり、本書へ未計測の固定値を確定しない。

```text
TransportConnectionSnapshotV1
  connection_attempt_ref:
    exact TransportConnectionAttemptIdentityRefV1
  accepted_connection_pair_identity_ref:
    optional exact TransportConnectionPairIdentityRefV1
  accepted_connection_identity_ref:
    optional exact TransportConnectionIdentityRefV1
  state: TransportConnectionStateV1
  state_sequence: positive u64
  previous_snapshot_ref:
    null | exact TransportConnectionSnapshotRefV1
  selected_security_binding_ref:
    optional exact TransportSecurityBindingRefV1
  negotiated_limit_ref:
    optional exact NegotiatedTransportLimitRefV1
  send_window_summary[0..4]:
    TransportDeliveryWindowSummaryV1
  receive_window_summary[0..4]:
    TransportDeliveryWindowSummaryV1
  backpressure_state:
    clear | constrained | saturated
  last_progress_time: monotonic duration value
  close_reason: optional TransportCloseReasonV1
  diagnostic_refs[0..64]
  snapshot_content_hash: SHA-256

TransportDeliveryWindowSummaryV1
  delivery_class: TransportDeliveryClassV1
  window_state:
    { kind: empty }
    | { kind: populated,
        sequence_floor: bounded u64,
        sequence_ceiling: bounded u64,
        message_count: bounded positive u32,
        byte_count: bounded positive u64 }
```

Snapshotはbounded immutable projectionで、packet content、secret、native address、pointer、Provider thread stateを含めない。window summaryはdelivery class uniqueとし、`populated`では`sequence_floor <= sequence_ceiling`を必須にする。最初のSnapshotだけ`state_sequence=1`かつ`previous_snapshot_ref=null`、後続は同じAttemptのcurrent head refとexact `N+1`を持ち、single-writer CASで一件だけ進める。new Attemptは新しいchainをsequence 1から開始し、previous Attempt refでconnection lineageを閉じる。`configured | resolving | dialing | handshaking`はaccepted Pair／identity、selected security、negotiated limitを持たず、window配列は空、close reasonはnullとする。`active`は同じAttemptへ解決するnon-null accepted identity、それをexact memberに持つPair、security／limitを必須とし、send／receive両window配列のdelivery class集合をAcceptanceの`selected_delivery_capability_set[]`とそれぞれset equalityにする。未選択classのdummy windowまたは選択classの欠落を拒否し、close reasonはnullとする。`reconnect_wait | closing | closed | fault`は直前にAcceptance済みなら同じsealed Pair／identity／security／limit／window summaryを保持し、未Acceptance attemptではそれらをnull／空のまま維持する。後四stateは遷移原因と一致するnon-null close reasonを持ち、inactive optional Field、別Attempt identity、state／reason不一致を拒否する。

## 6. Message envelopeとsemantic delivery

```text
TransportMessageEnvelopeV1
  envelope_version: positive u16
  connection_pair_identity_ref:
    exact TransportConnectionPairIdentityRefV1
  sender_connection_identity_ref:
    exact TransportConnectionIdentityRefV1
  recipient_connection_identity_ref:
    exact TransportConnectionIdentityRefV1
  message_type_ref: exact RegisteredMessageTypeRefV1
  message_schema_ref: exact MessageSchemaRefV1
  message_id: StableId
  delivery_class: TransportDeliveryClassV1
  delivery_stream_id: StableId
  delivery_sequence: bounded u64
  expiry:
    { kind: none }
    | { kind: monotonic_deadline,
        deadline: bounded monotonic duration value }
  payload_length: bounded u32
  payload_content_hash: SHA-256
  integrity_binding_ref: exact MessageIntegrityBindingRefV1
  envelope_content_hash: SHA-256

TransportMessageEnvelopeRefV1
  envelope_version: positive u16
  message_id: StableId
  envelope_content_hash: SHA-256
```

`envelope_content_hash`は自身を除くEnvelope全FieldのMCD canonical bytesに対するSHA-256で、Pair、sender／recipient方向、message type／Schema、delivery class／stream／sequence、expiry、payload length／hash、integrity bindingをすべて封印する。Refの三Fieldは完成Envelopeとbyte equalityにし、payload hash、message IDまたは表示名だけからEnvelope refを生成しない。payloadは登録済みSchemaへexact解決し、message type、schema、delivery class、max size、allowed consumer／producerが一致しなければ送受信しない。sender／recipient Identityは異なる向きで、Pairのidentity集合とset equalityにする。Connection ID、epoch、protocol revisionは各Identityからexactに解決し、Envelopeへ別値を複写しない。wire Adapterが必要なbounded routing値を符号化しても、受信時に同じPairと方向へ復元できないbytesをrejectする。本書はpayload bytesを解釈せず、Multiplayer ownerはpacket／fragmentを解釈しない。

delivery semanticsを次に固定する。

| class | 保証 | 明示的に保証しないもの |
|---|---|---|
| `unreliable_latest` | stream内で受理時点の最も新しい非失効messageだけをconsumerへ渡せる | delivery、全sequence、retransmit、order history |
| `unreliable_sequenced` | 受理したmessageをstream sequence非減少で渡し、旧sequenceを破棄する | delivery、gap fill、retransmit |
| `reliable_ordered` | bounded lifetime／window内で重複排除しstream sequence順に渡す | 無期限delivery、application success、別streamとの順序 |
| `reliable_unordered` | bounded lifetime／window内で重複排除し受理済みmessageを渡す | 相互順序、application success、exactly-once effect |

`message_id`の再受信は、同じConnection Pair／方向において`TransportMessageEnvelopeRefV1`の全Fieldがbyte equalityのときだけduplicateとする。同じPair／方向／`message_id`で`envelope_version`または`envelope_content_hash`が異なる場合はintegrity faultであり、payload hashの一致だけではduplicateにしない。expiry後のreliable messageはtransport failure／diagnosticであり、success扱いまたは別classへ降格しない。

## 7. Packetization、fragmentation、flow control

```text
TransportPacketPlanV1
  plan_version: 1
  plan_id: content-derived StableId
  envelope_ref: exact TransportMessageEnvelopeRefV1
  packetization_profile_ref: exact PacketizationProfileRefV1
  fragment_count: bounded positive u16
  fragment_records[1..65535]:
    fragment_index
    payload_range
    fragment_content_hash
  plan_content_hash: SHA-256

TransportPacketPlanRefV1
  plan_version: 1
  plan_id: StableId
  plan_content_hash: SHA-256
```

MTU、header overhead、fragment limit、reassembly byte／time limitはTarget／Provider-qualified Profileが決める。Adapterがpath MTUまたはProvider制約を観測した場合も、Profileの許可範囲で新Packet Planを作り、既存fragment setを別layoutへ継ぎ足さない。

`TransportPacketPlanRefV1`の全Fieldは完成Planとbyte equalityにし、`plan_content_hash`は自身を除くPlan全FieldのMCD canonical bytesに対するSHA-256とする。reassembly identityは`{connection_pair_identity_ref, sender_connection_identity_ref, recipient_connection_identity_ref, exact envelope_ref, exact packet_plan_ref}`へ閉じる。同じreassembly identity／`fragment_index`／`fragment_content_hash`だけをduplicate fragmentとし、同じindexでhashが異なる場合、別Planのfragment混入、overlap、gap、count差、上限超過、expiryをrejectする。message IDまたはpayload hashだけをreassembly／duplicate identityにせず、incomplete setをpartial payloadとしてconsumerへ渡さない。

```text
TransportFlowControlStateV1
  connection_identity_ref:
    exact TransportConnectionIdentityRefV1
  delivery_class: TransportDeliveryClassV1
  send_window_limit_ref: exact BoundedWindowLimitRefV1
  receive_window_limit_ref: exact BoundedWindowLimitRefV1
  queued_message_count: bounded u32
  queued_byte_count: bounded u64
  in_flight_count: bounded u32
  pressure_state:
    clear | constrained | saturated
  producer_disposition:
    accept | reject_new | replace_latest | close_connection
```

`replace_latest`は`unreliable_latest`の同じstreamでだけ許可する。reliable classのqueueをsilent drop、latestへ変換、diskへ無制限spillしない。pressure stateとdispositionはProfileで決まり、Gameplay importanceまたはplayer roleを本書で推測しない。共通CPU／memory／bandwidth measurementは[Performance／Capacity](../04-runtime/performance-capacity.md)へ委譲する。

## 8. Security、confidentiality、peer binding

`TransportSecurityProfileV1`はrequired peer authentication class、integrity、confidentiality、replay protection、key rotation、credential expiry／revocation reaction、allowed downgrade exact `[]`を持つ。algorithm、key size、library versionはProvider／Toolchain lockとProduct Security policyへexact参照し、自由文字列または本文既定値で採択しない。

transport encryptionは次を意味しない。

- Accountまたはplatform userの所有証明
- Product entitlement
- trusted client、anti-cheat合格
- gameplay role／server authority
- typed commandのvalidation成功
- cloud serviceまたはrelay operatorの全面的信頼

credential失効、peer auth mismatch、integrity failure、replay window違反では該当epochをfaultへ進め、same epoch key renegotiationで復帰しない。secret、raw token、private endpoint、full payloadをlog／AI projection／Replayへ保存しない。security case、incident、disclosureは[Product Security](../01-governance/product-security.md)だけが所有する。

## 9. Multiplayer、Scheduling、Runtime Packageとのhandoff

| Producer／Consumer | 本書が受け取るもの | 本書が返すもの |
|---|---|---|
| Multiplayer | exact registered message、Schema、delivery class、expiry、size／rate intent | connection-bound message receipt／delivery／failure。gameplay successではない |
| Scheduling | bounded poll／publish phase、lifetime／shutdown boundary | ready message batchとowner-typed wakeup。remote peerをphase ownerにしない |
| Runtime Package | Target／headless package内のProfile／Adapter／credential Ref closure | launch時Profile validation。headless supportを自動主張しない |
| Platform | lifecycle、network capability、private adapter bridge | normalized Target／Provider capabilityとfault |
| Performance | queue／memory／bandwidth envelope | demand、pressure、drop／reject／close reason |
| Debug／Replay | capture policy、redaction policy | metadata／causality／fault injection result。secret／full payloadは除外 |
| Future Online Services | opaque endpoint／relay／peer-auth binding | connection state。Lobby／match resultは返さない |

receive pathは`Provider bytes -> packet／fragment validation -> Envelope validation -> delivery semantics -> registered consumer message batch`の一方向である。consumerはsocket callback内でSimulation／ECSを直接変更せず、Schedulingの決めたpublication boundaryへ渡す。

send pathは`registered producer message -> Envelope validation -> flow decision -> Packet Plan -> private Provider Adapter`である。Provider callback、packet arrival、ack arrivalをSimulation timeまたはGameplay authorityにしない。

## 10. Reconnectとfallback

reconnectはTransport connectionの再確立で、gameplay session復帰、object resync、host migrationではない。new epochが`active`になった後、Multiplayer ownerがsession ticket、baseline、authorityを再検証し、`joined／synchronized`へ進めるかを決める。

```text
TransportReconnectPolicyV1
  policy_id/version/content_hash
  retryable_reason_set[1..16]: TransportCloseReasonV1
  maximum_attempt_count: bounded positive u16
  backoff_profile_ref: exact BoundedBackoffProfileRefV1
  endpoint_refresh:
    reuse_if_fresh | require_fresh_binding
  credential_refresh:
    reuse_if_fresh | require_fresh_binding
  preserve_logical_connection_id: true
  require_new_epoch: true
```

Provider／endpoint fallbackはProfileに列挙したexact Target-compatible Bindingへ、`reconnect_wait -> resolving`でだけ行う。direct→relay、platform→direct等でsecurity、delivery class、limit、cost／privacy meaningが変わる場合は別fallback dispositionとユーザー可視diagnosticを必要とする。fallback不能ならfault／closedとし、LAN、public internet、default relayを自動探索しない。

## 11. Diagnostic、fault injection、observability

最低限、次を別diagnosticにする。

- endpoint missing／expired／wrong Target、resolution failure
- Provider missing／unqualified／version mismatch／fault
- protocol family／revision／delivery capability mismatch
- peer authentication／integrity／confidentiality／credential failure
- illegal state transition、stale epoch、sequence／stream violation
- malformed／oversized envelope、unknown message Schema、hash mismatch
- fragment duplicate／overlap／gap／count／expiry／reassembly limit
- timeout、keepalive failure、loss、duplication、reorder、jitter
- send／receive window saturation、reliable expiry、backpressure close
- reconnect attempt／backoff／endpoint refresh violation
- Target lifecycle interruption、network capability loss

fault injectionはloss、duplication、reorder、latency、jitter、bandwidth clamp、MTU change、disconnect、credential revocation、Provider faultをbounded Profileとして指定し、production connectionと別Qualification identityを持つ。untrusted remote peerがfault injectorを制御できない。

Telemetryはconnection IDのredacted surrogate、epoch、state／reason、protocol revision、delivery class、message／byte aggregate、window／pressure、latency／jitter／loss aggregate、Provider／Target generation hash、diagnostic refを返す。IP、raw endpoint、Account ID、credential、payload、secretを含めない。

## 12. AI／Editor理解境界

AI／Editor projectionはProfile、Target support、endpoint kind、connection state、protocol revision、delivery class、limit、pressure、reconnect policy、selected fallback、Diagnostic、Qualification stateをboundedに示す。raw address、secret、packet、payload、Provider object、native errorを公開しない。

`host`、`connect`、`reconnect`、`disconnect`、`inject fault`、`explain`はplanned semantic vocabularyで、current MCD Operation、Service、Tool、Provider surfaceではない。完全なAuthority／Schema／Validator／Receipt／publication closureが承認されるまでaction名からIDを生成またはdispatchしない。

AIは「multiplayerを有効化」「LAN」「online」「low latency」からProtocol、Provider、relay、encryption、server topology、player countを自動選択しない。transport scope、Target、endpoint kind、required delivery、security、failure／cost dispositionを分離して説明する。

## 13. Qualification

Qualification subjectは少なくとも`{Transport Profile ref, protocol revision, Provider Binding ref, Target Profile ref, Platform capability signature ref, security binding class, delivery class set, fixture set ref}`へ閉じる。一Provider／Target／revision／security profileのReceiptを別tupleへ流用しない。

fixtureは次を含む。

1. listener／dial／accept、handshake、revision selection、active／close
2. four delivery classesのpositive guaranteeと明示的non-guarantee
3. loss、duplicate、reorder、latency、jitter、timeout、keepalive
4. packet／fragment boundary、MTU change、overlap／gap／bomb／expiry
5. queue／window saturation、backpressure、reliable expiry、bounded recovery
6. malformed／oversized／unknown Schema、stale epoch、replay、auth／credential failure
7. disconnect／reconnect、new epoch、old data rejection、endpoint／credential refresh
8. direct／platform／relay／tunnel candidateのsame-semantic conformance
9. desktop、mobile、headlessと将来Targetのlifecycle／capability差
10. redaction、fault injection isolation、Provider／device fault

negative fixtureはwrong Owner／ID／version／hash、duplicate、sort違反、inactive branch、count超過、stale／revoked Receiptを一原因ずつ検出する。connect成功、encrypted connection、loopback testをmultiplayer、dedicated server、internet reachability、production operationsのReceiptにしない。

## 14. 公式資料から採用する構造と限界

- [Unity Transport integration](https://docs.unity.com/en-us/relay/integration)はconnection abstractionがreliability、ordering、fragmentationを扱い、Relay／Netcodeとは別layerであることを示す。本書はsemantic Transport Contractとgameplay replicationを分離する。
- [Unity Relay servers](https://docs.unity.com/relay/relay-servers)はRelayがlisten-server styleを支援できてもdedicated game serverそのものではないことを示す。本書はrelay endpointをserver topologyと同一視しない。
- [Godot: High-level multiplayer](https://docs.godotengine.org/en/4.7/tutorials/networking/high_level_multiplayer.html)は`MultiplayerPeer`とRPC authority／deliveryを別関心として示す。本書はpeer transportだけを所有する。
- [Unreal Engine: Networking Overview](https://dev.epicgames.com/documentation/unreal-engine/networking-overview-for-unreal-engine?lang=en-US)はclient／server connectionとserver-authoritative replicationを区別する。本書はconnection成功をauthority成功にしない。

外部EngineのAPI名、Protocol、Provider、channel番号、既定timeout、package version、Service構成をMiraikanaiのcanonical IDまたは採用要件にしない。比較から採用するのはTransport／Netcode／Service／Targetのlayer分離だけである。

## 15. 明示的非目標と設計Closure

非目標は、gameplay session／authority／replication、Online Identity／Lobby／Matchmaking／Hostingの実装または採択、特定Protocol／Provider／Service／暗号algorithmの選定、Internet運用SLO、NAT traversal保証、実装Task／順序／担当／工数／日程の定義である。

本書の設計Closureは次をすべて満たす場合だけ成立する。

1. Transport connectionとgameplay session／authorityが別identity、別state machineである。
2. four delivery classesの保証と非保証、bounded queue／expiry／backpressureが機械判定できる。
3. reconnectがnew epochを必須とし、旧packet／fragment／message／credentialを受理しない。
4. direct／platform／relay／tunnelがprivate Adapterで、same-semantic Target Qualificationを必要とする。
5. security bindingがAccount／entitlement／anti-cheat／gameplay authorityを過大主張しない。
6. Online Servicesを暗黙採択せず、opaque exact bindingだけで将来接続できる。
7. 本書の存在をconnect実装済み、Multiplayer対応、Dedicated Server対応、Online対応の根拠にしない。
