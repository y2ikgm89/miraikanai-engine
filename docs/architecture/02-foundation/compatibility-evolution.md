# Miraikanai Engine Compatibility and Evolution Contract

- 文書ID: mirakan.arch.compatibility-evolution
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Architecture／Schema／artifact／runtime boundaryの互換性class、consumer inventory、clean-break procedure、reader／writer／aliasの可否、recook／rebuild／migration evidence、互換性ChangeSet
- 非正本範囲: 個別Domain Schemaのfield、Save payload、Package binary、runtime lease、Product Work Package、外部SDKのversion。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Executable Contracts](executable-contracts.md)
- 関連文書: [Architecture Governance](../01-governance/architecture-governance.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Product plan](../00-product/product-plan.md)、[Executable contracts](executable-contracts.md)、[Memory／Pointers](memory-pointers.md)、[Project state](../03-authoring/project-state.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Runtime ECS](../04-runtime/entity-component-system.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Persistence／Save](../04-runtime/persistence-save.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-26

## 1. 結論

互換性は「旧名称を残すこと」ではない。保存済み入力、公開済みconsumer、外部API、配布済みartifactのどれに対して何を読めるかを、Owner、期間、検証、rollbackと共に明記する契約である。

release済みconsumerを根拠にしない内部設計整理はclean breakを既定とする。旧type alias、dual reader、暗黙変換、synthetic identity、旧path redirectを残さず、committed Sourceから新形式を再生成する。公開済み互換性が必要な場合だけ、versioned migrationと有限のreader期間を承認済みCompatibility ChangeSetへ記録する。

本書は互換性の判定枠組みを所有するだけで、Schema、MCD、Operation、binary readerをactive化しない。

## 2. 互換性classとconsumer inventory

```text
CompatibilityClass
  clean_break
  source_preserving_recook
  versioned_reader_migration
  external_api_deprecation

CompatibilityConsumerClass
  authoring_source | derived_artifact | runtime_package | save | replay
  | native_abi | external_api | runtime_projection
  | observability_projection | documentation

CompatibilityConsumerEndpointRefV1
  endpoint_id: StableId
  endpoint_kind: reader | writer | reader_writer | sealed_projection
  endpoint_content_ref: immutable content-addressed ref
  endpoint_content_hash: SHA-256

CompatibilityConsumerDiscoveryScopeKind
  repository_worktree | reachable_git_history | release_registry
  | distribution_registry | native_abi_registry | external_api_registry

CompatibilityConsumerDiscoveryScopeV1
  scope_kind: CompatibilityConsumerDiscoveryScopeKind
  disposition: complete | not_applicable
  evidence_requirement_refs[1..64]: sorted unique EvidenceRequirementRefV1
  evidence_satisfaction_bindings[0..64]:
    sorted unique EvidenceSatisfactionBindingV1

CompatibilityConsumerRecordV1
  consumer_id: StableId
  subject_id: ClosedCompatibilitySubjectId
  consumer_class: CompatibilityConsumerClass
  owner_ref: OwnerIdentityLocalRefV1
  boundary: source | generated | in_process | retained_release | external
  source_format_refs[1..64]: sorted unique format ref
  reader_endpoint_refs[0..64]: sorted unique CompatibilityConsumerEndpointRefV1
  writer_endpoint_refs[0..64]: sorted unique CompatibilityConsumerEndpointRefV1
  source_rebuild_status: verified | impossible | not_applicable | unverified
  old_reader_requirement: absent | bounded_until_release
  old_writer_requirement: absent | bounded_until_release
  rollback_policy: source_rebuild | old_reader_window | release_rollback
  evidence_requirement_refs[1..64]: sorted unique EvidenceRequirementRefV1
  evidence_satisfaction_bindings[0..64]:
    sorted unique EvidenceSatisfactionBindingV1
  record_content_hash: SHA-256

CompatibilityConsumerInventoryV1
  inventory_id: StableId
  inventory_version: positive uint32
  subject_id: ClosedCompatibilitySubjectId
  source_format_refs[0..64]: sorted unique format ref
  required_discovery_scope_kinds[1..16]:
    sorted unique CompatibilityConsumerDiscoveryScopeKind
  discovery_scopes[1..16]: sorted unique CompatibilityConsumerDiscoveryScopeV1
  consumer_records[0..4096]: sorted unique CompatibilityConsumerRecordV1
  discovery_state: collecting | complete | rejected
  unknown_consumer_state: unresolved | zero_verified
  evidence_requirement_refs[1..128]: sorted unique EvidenceRequirementRefV1
  evidence_satisfaction_bindings[0..128]:
    sorted unique EvidenceSatisfactionBindingV1
  inventory_content_hash: SHA-256

CompatibilityConsumerInventoryRefV1
  inventory_id: StableId
  inventory_version: positive uint32
  inventory_content_hash: SHA-256

CompatibilityChangeRefV1
  compatibility_change_id: StableId
  compatibility_version: positive uint32
  content_hash: SHA-256

CompatibilityChangeSetV1
  compatibility_change_id: StableId
  subject_id: ClosedCompatibilitySubjectId
  class: CompatibilityClass
  source_format_refs[0..64]
  target_format_refs[1..64]
  consumer_inventory_ref: CompatibilityConsumerInventoryRefV1
  affected_consumer_classes[0..32]: sorted unique CompatibilityConsumerClass
  old_reader_policy: absent | bounded_until_release
  old_writer_policy: absent | bounded_until_release
  alias_policy: forbidden | bounded_deprecation
  source_preservation_policy: retain | explicit_export_required | unavailable
  regeneration_policy: none | full_recook | full_rebuild | explicit_migration
  rollback_policy: source_rebuild | old_reader_window | release_rollback
  evidence_requirement_refs[1..128]: sorted unique EvidenceRequirementRefV1
  evidence_satisfaction_bindings[0..128]:
    sorted unique EvidenceSatisfactionBindingV1
  approval_ref: optional ArchitectureApprovalRefV1
  state: proposed | approved | applied | retired | rejected
  content_hash: SHA-256
```

`CompatibilityConsumerInventoryV1`は「見つからなかった」という説明文ではない。各discovery scopeのRequirement、署名済みfulfillment、source format、consumer endpoint、rebuild可否、old reader／writer requirementを一つのimmutable recordに閉じる。各`evidence_requirement_refs[]`と`evidence_satisfaction_bindings[].evidence_requirement_ref`は、`complete`時にexact set equalityでなければならない。`required_discovery_scope_kinds[]`と`discovery_scopes[].scope_kind`もset equality、各scope kindはexactly once、`not_applicable`もscope固有Requirementのpass fulfillmentを必須にする。`consumer_records=[]`を「consumerなし」と解釈できるのは、`discovery_state=complete`、`unknown_consumer_state=zero_verified`、全required scopeが`complete`またはpass fulfillment付き`not_applicable`である時だけである。`collecting`、`unresolved`、Requirement不足、binding不足、scope集合差、format集合差のいずれかはunknown consumerとしてfail closedにする。

`CompatibilityChangeSetV1`の`consumer_inventory_ref.subject_id`と`source_format_refs`はChangeSet自身とexact equalityにする。`affected_consumer_classes`はInventory recordから導くclass集合とset equalityにし、class名だけでconsumerの存在を推測しない。`approved`または`applied`は、Inventoryが`complete`かつ`zero_verified`で、ChangeSet自身とInventoryの全Evidence Requirementがpass fulfillmentを持つ場合だけ許す。Inventoryが`rejected`ならChangeSetも`rejected`へ戻し、旧readerの要否を説明文やrelease名から補完しない。

`clean_break`は旧reader、old writer、aliasを持たない。`source_preserving_recook`はSourceの正本を保持し、Derived Artifact、Catalog、Runtime Package、fixtureを全再生成する。`versioned_reader_migration`と`external_api_deprecation`だけが、対象release、終了日、read/write組合せ、rollback、telemetryを明記した有限の旧readerを許可できる。

同じChangeSet内で旧writerを停止せず新readerだけを追加する、または新writerを出して旧reader期限を決めない状態を禁止する。互換性を主張するために、未検証の古いbytesを新formatとして再署名してはならない。

## 3. Clean-break procedure

clean breakは次の順に扱う。

1. `CompatibilityConsumerInventoryV1`でSource、Project revision、authoring document、release／distribution／Native ABI／外部APIのconsumerをInventoryし、required discovery scopeをcompleteにする。
2. Inventoryのunknown consumerをzero verifiedにし、保持可能な正本とsource rebuild可否を確定する。`unresolved`のまま次へ進まない。
3. 旧format・旧alias・旧path・旧reader・旧writerの対象を列挙し、削除または非current履歴として分離する。
4. target SchemaとOwner transferを確定し、対象Derived Artifact、Catalog、Runtime Package、Save／Replay fixtureのregeneration policyを固定する。
5. stale cacheをcurrent inputとして使わず、committed Sourceとtarget Toolchain lockからfull recookまたはfull rebuildする。
6. canonical hash、owner ref、dependency closure、qualification evidenceをread-backし、旧形式の混入を拒否する。
7. source rebuildが不可能なconsumer、またはretained release／external consumerが一件でもある場合はclean breakを適用せず、`versioned_reader_migration`または`external_api_deprecation`へclassを変更して承認を取り直す。

Sourceを保持することはruntime handle、native pointer、chunk location、worker順、raw memory bytesを保存することではない。これらは再構築できないsession-local値であり、Save／Packageへ擬似互換fieldを追加しない。

## 4. Runtime ECS canonicalizationの互換性

Runtime ECS正本化の`source_preserving_recook`はreview中の候補classである。これは[Governance Migration Proposals](../appendices/governance-migration-proposals.md#2-runtime-ecs-canonicalization-candidate)の`RuntimeEcsCanonicalizationChangeSetV1`候補と一対一に結ぶが、completeかつzero verifiedな`CompatibilityConsumerInventoryV1`とapproved Compatibility ChangeSetが生成されるまで承認済みclassではない。同ChangeSetが`applied`になるまではcurrent formatを変更しない。

| 旧concept | target concept | target Owner | cutover規則 |
|---|---|---|---|
| `EntityHandle` | `RuntimeEntityHandle` | Runtime ECS | old aliasを残さない。callbackを越える参照は`RuntimeEntityRefV1`へ明示投影する |
| `ComponentAccessManifest` | `RuntimeComponentAccessManifestV1` | Runtime ECS | read/write/structural permission、query selection、phase、budgetをtarget schemaで閉じる |
| suffixなし`DerivedArtifactManifest` | `DerivedArtifactManifestV1` | Asset Lifecycle | subject tagged unionとgeneric catalog keyへ再Cookする |
| asset-only catalog entry | `MirakanArtifactCatalogV1` entry | Asset Lifecycle | keyを`{artifact_subject_ref, artifact_role_id}`へ移し、synthetic Asset IDを禁止する |
| ad-hoc World payload | `RuntimeWorldRootImageV1`／`RuntimeWorldSectionImageV1` | Runtime Package | generic Artifact envelopeを再定義せず、payload contractを分離する |
| ad-hoc Save／Replay projection | `RuntimeWorldSaveRecordSetV1`／`RuntimeReplayProjectionV1` | Persistence／Save | ECS identity・field projectorを参照し、raw runtime handleを保存しない |

対象の旧名称は、承認・適用後にActive仕様、current Owner Registry、MCD、native ABI、catalog、diagnostic、生成bindingから消す。history、ChangeSetのsource side、migration fixtureの入力識別子だけは旧形式を判別するために保持できるが、それをcurrent type、alias、dispatch keyとして解釈しない。

complete Consumer Inventoryの全retained consumerがcommitted Sourceから再構築できる場合だけ、次のconditional policyを使う。

```text
change_class = source_preserving_recook
old_reader_policy = absent
old_writer_policy = absent
alias_policy = forbidden
source_preservation_policy = retain
regeneration_policy = full_recook | full_rebuild
rollback_policy = source_rebuild
```

regeneration inputはcommitted Source、Asset import document、approved Runtime entry documentだけである。stale cache、old package bytes、raw Runtime handle、chunk row、address、old generated bindingを入力にしない。

適用時は次をcurrent boundaryからatomicに除去する。

- legacy or suffixless type aliases
- object-address or pool-slot identity
- pointer-backed inline Component payload
- per-Entity virtual update storage
- Runtime shared_ptr ownership
- alternate Shipping AoS／sparse-set storage
- direct mutation during iteration
- old query cache entries and persisted row selections
- dual Package／Save／Replay Runtime-layout projections
- old generated API signatures that violate `CppValueTransferPolicyV1`

committed Project Source、Asset provenance、published Save、Native ABI consumer、distributed Package、external API consumerは、それぞれのapproved migrationが明示しない限り削除しない。

```text
diagnostic.compatibility.ecs-consumer-inventory-unresolved
MIRAKAN-COMPATIBILITY-ECS-CONSUMER-INVENTORY-UNRESOLVED
arguments = change_set_ref, discovery_scope, discovery_state
result = clean-break application rejected

diagnostic.compatibility.ecs-retained-external-consumer
MIRAKAN-COMPATIBILITY-ECS-RETAINED-EXTERNAL-CONSUMER
arguments = change_set_ref, consumer_class, consumer_ref
result = switch to separately approved finite migration
```

release、published Save、Native ABI、distribution、external consumerのいずれかがrebuild不能ならclean breakを停止する。finite reader window、target release、telemetry、rollback、removal Gateを持つ別承認の`versioned_reader_migration`または`external_api_deprecation`へ切り替え、readerを推測または恒久化しない。

### 4.1 再生成対象

```text
RuntimeEcsCleanBreakRegenerationSetV1
  source_inputs:
    Project source documents
    Asset import documents
    approved Runtime entry documents
  invalidate:
    Derived artifact cache
    artifact catalog
    runtime world root and section images
    runtime package manifests and binaries
    save/replay fixture artifacts
    generated native and AI projection bindings
  prohibited_inputs:
    old manifest bytes
    old package bytes
    raw runtime handles
    synthetic asset identities
    unsealed live-world capture
```

このsetは「既存fileを破棄する操作」ではなく、target形式のcurrent candidateとして受理しない入力範囲を示す。物理的なcache cleanupやimplementation手順は実装承認後のOwner手順で決定する。

### 4.2 ECS consumer inventory boundary

ECS正本化では次のdiscovery scopeを必須にする。これはconsumerの存在を主張する表ではなく、Inventoryが未調査対象を残さないための最低調査境界である。

| 調査境界 | inventory class | 完了条件 |
|---|---|---|
| Project Source／Authoring document | `authoring_source` | committed Sourceから対象形式を再生成できることをRequirementのpass fulfillmentで確認する |
| Derived Artifact／Catalog | `derived_artifact` | old manifest bytesをcurrent inputにせずrecookできることをpass fulfillmentで確認する |
| World Root／Section／Runtime Package | `runtime_package` | Package reader／writer、distribution対象、release rollback要否をRequirementとfulfillmentで列挙する |
| Save／Replay | `save`／`replay` | retained Save／Replay、reader／writer、migration failureとrollback要否をRequirementとfulfillmentで列挙する |
| Native Game Module／公開C ABI | `native_abi` | externalまたはrelease済みABIのreader、writer、compatibility windowをRequirementとfulfillmentで列挙する |
| Renderer／Audio／VFX | `runtime_projection` | live leaseではなくsealed projectionだけを読む境界と再生成可否をpass fulfillmentで確認する |
| Debug transport／AI capture | `observability_projection` | capture／projectionのreader、redaction、retentionをRequirementとfulfillmentで列挙する |
| 外部API／配布先／文書projection | `external_api`／`documentation` | 公開surface、distribution、deprecated alias、consumer通知の要否をRequirementとfulfillmentで列挙する |

Runtime ECS正本化の`required_discovery_scope_kinds[]`は`repository_worktree`、`reachable_git_history`、`release_registry`、`distribution_registry`、`native_abi_registry`、`external_api_registry`のexact setとする。存在しないRegistryを省略せず、`not_applicable`とscope固有Requirementのpass fulfillmentで明示する。

materialization時のscope Requirementは次のkind／predicateを使う。これはRequirement instanceの選択規則であって、現在のrecord、Receipt、Approvalを発行するものではない。各instanceは対象source format／Owner／closureを持つimmutable subjectへ束縛する。

| discovery scope | `EvidenceRequirementV1.evidence_kind` | `acceptance_predicate` | pass時に確定すること |
|---|---|---|---|
| `repository_worktree` | `repository_tree_inventory` | `exact_set_equality` | frozen source tree内のformat、reader／writer endpoint、derived inputの全量 |
| `reachable_git_history` | `reachable_history_inventory` | `exact_set_equality` | reachable ref全体でretainされたformat／reader／release metadataの全量 |
| `release_registry` | `release_registry_inventory` | `registry_export_complete` | subjectに関係するpublished releaseとrollback／old-reader windowの全量 |
| `distribution_registry` | `distribution_registry_inventory` | `registry_export_complete` | distribution済みPackage／contentとconsumer通知対象の全量 |
| `native_abi_registry` | `native_abi_registry_inventory` | `registry_export_complete` | published native ABI、header／manifest、consumer compatibility windowの全量 |
| `external_api_registry` | `external_api_registry_inventory` | `registry_export_complete` | public API／SDK／caller registrationとdeprecation対象の全量 |
| 発見済みconsumer endpoint | `consumer_endpoint_inventory` | `endpoint_inventory_complete` | 各recordのreader／writer、owner、old reader／writer requirement、rebuild可否 |

scope exportまたはclosed source treeが対象format／consumerを0件と示す場合だけ、対応Requirementの`acceptance_predicate=no_match_in_closed_scope`を使用できる。export未取得、read権限なし、subject format不明、署名／freshness不成立は0件の根拠ではなく`unresolved`である。

current Architectureには、このECS clean breakに対するmaterialized `CompatibilityConsumerInventoryV1`、公開済みconsumer inventory、またはapproved old-reader windowを置かない。この欠落をconsumerゼロと解釈しない。Inventoryがcompleteになる前、または実consumerが見つかった時は、ChangeSetを`review`または`rejected`へ戻し、clean-break approvalを発行しない。

### 4.3 Saveとrelease boundary

Save、Replay、外部Native ABI、配布済みcontent packageに実consumerが存在する場合、source recookだけでは互換性を満たさない。その場合は対象ごとに`versioned_reader_migration`または`external_api_deprecation`を選び、reader／writer version、互換期限、migration failure、rollback、qualification targetをChangeSetへ追加する。

release済みconsumerがないことは`CompatibilityConsumerInventoryV1`のcomplete／zero-verifiedかつscope Requirementのpass fulfillmentでのみ証明する。したがって、将来の実装開始前にconsumer inventoryが見つかった場合だけでなく、Inventoryが未完成、Requirement／binding不足、endpoint不明、またはrebuild statusが`unverified`である場合も、clean breakの前提が未成立としてChangeSetを`rejected`または再reviewへ戻す。

### 4.4 Current local review observation

これは`CompatibilityConsumerInventoryV1`、Compatibility ChangeSet、Evidence Requirement fulfillment、release inventoryではない。2026-07-24にlocal Git baseline `41929d0`とpublic GitHub metadataへ行った未署名の探索結果だけを、将来の再調査時に「何をまだ証明していないか」を失わないため記録する。baselineには追跡pathが51件あり、48件は`docs/**/*.md`、残る3件は`.codex/config.toml`、`.gitattributes`、`.gitignore`である。全reachable historyの変更pathも文書または同3設定だけである。Git tagは0件、`origin`の公開refは`main`だけで、[GitHub Releases](https://github.com/y2ikgm89/miraikanai-engine/releases)も同日時点でrelease 0件を表示した。Source／test／schema／Package／Save／Replay／native header／release artifactはこのbaselineまたはreachable historyに追跡されていない。

| required discovery scope | local観測 | inventoryへ転記できない理由 | 現在状態 |
|---|---|---|---|
| `repository_worktree` | baselineは文書と設定だけで、ECS source formatまたはconsumer endpointを発見していない | worktreeは未commitで、観測は署名Receiptでもclosed source snapshotでもない | `collecting` |
| `reachable_git_history` | tag 0件、全reachable historyの変更pathは文書または設定だけ | 全reachable refの署名済みformat／consumer inventoryを発行していない | `collecting` |
| `release_registry` | `origin`にtagなし、public [GitHub Releases](https://github.com/y2ikgm89/miraikanai-engine/releases)も観測時release 0件 | public page／Git refはProfileに束縛されたcomplete snapshotでもtrusted collector Receiptでもない | `unresolved` |
| `distribution_registry` | local repository内にdistribution inventoryはない | 外部authority profile、照会／export、complete snapshot、trusted collector Receiptがない | `unresolved` |
| `native_abi_registry` | tracked native header／ABI manifestはない | 公開済みABIの不存在をlocal pathの欠落から推測できず、authority profile／snapshot／Receiptもない | `unresolved` |
| `external_api_registry` | tracked SDK／公開API caller registryはない | 外部consumer・published APIの不存在をlocal pathの欠落から推測できず、authority profile／snapshot／Receiptもない | `unresolved` |

この観測から`consumer_records=[]`、`zero_verified`、`not_applicable`、clean break、old-reader不要を導かない。各行は[AI Verification／Provenance](../01-governance/ai-verification-provenance.md#711-evidence-requirementとfulfillment-binding)の`EvidenceRequirementV1`、closed fulfillment subject、freshな`TechnicalQualificationReceiptV1`を揃えた時だけmaterialized Inventoryへ進める。外部registryのauthority profileに基づく照会またはsigned export、complete snapshot、Receiptが与えられない限り、最も安全な結論は`unresolved`である。

### 4.5 外部Registry closureの提出境界

残る4つの外部scopeは、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md#712-外部registry-snapshotとauthority-boundary)の`ExternalRegistryAuthorityProfileV1`、同Profileに従う`ExternalRegistrySnapshotV1`、trusted collectorのpass `VerificationReceiptV1`、freshな`TechnicalQualificationReceiptV1`の順で閉じる。これは実装Task、外部サービスへの書込み、資格情報の提出要求、または承認の代行ではなく、外部Ownerが渡すべき最小のevidence interfaceである。

| required discovery scope | Profileが固定するauthorityと完了境界 | normalized inventoryの最小内容 | `complete`にできない場合 |
|---|---|---|---|
| `release_registry` | repository／owner、公式release endpointまたはprovider signed export、API／export version、全pageまたはclosed manifest | release／tag、target revision、published／draft／prerelease、asset digest、rollback・old reader window | Profile未確定、認可なし、page未完、source errorは`unresolved` |
| `distribution_registry` | provider、tenant／namespace、delivery endpoint、全配布channel／regionの閉包 | 配布済みPackage／content、digest、配布先、rollback、consumer通知対象 | provider／namespace未提示、export未取得、範囲不明は`unresolved` |
| `native_abi_registry` | 公開SDK／header／ABI manifestのauthority、配布namespace、version／compatibility policyの閉包 | ABI／header／manifest version、公開consumer、互換window、retirement／rollback | public ABI sourceまたはconsumer registryが不明なら`unresolved` |
| `external_api_registry` | public API contract／SDK／caller registrationのauthority、公開namespace、deprecation channelの閉包 | API／SDK version、registered caller／consumer、deprecation通知、sunset／rollback | authority、caller registry、公開範囲のいずれか不明なら`unresolved` |

GitHubを`release_registry`のauthorityとして選ぶ場合、[公式Releases REST API](https://docs.github.com/en/rest/releases/releases)のList releases responseはraw snapshot inputにできる。現行の公開Releases page観測やGit ref照会は、そのProfile、全page closure、trusted collector Receiptを持たないため、これを置換しない。現行reviewでは4 scopeのProfile／Snapshot／Receiptはいずれも未materializeであり、外部照会も発行していない。

| snapshotの検証結果 | Consumer Inventoryへの扱い |
|---|---|
| `complete`かつnormalized inventoryが空 | 当該Requirementのpredicateに従ってpass候補にできる。他scope、endpoint inventory、Requirement／binding集合もcompleteになるまで`zero_verified`にしない。 |
| `complete`かつrecordあり | recordごとにconsumer class、reader／writer、source rebuild、rollbackをInventoryへ入れ、`clean_break`ではなく適切なmigration classを再選択する。 |
| `authority_denied`、`source_error`、`pagination_incomplete`、Profile／signature／freshness不一致 | empty record、`not_applicable`、old reader不要、clean breakの根拠にせず`unresolved`を維持する。 |

Profile、Snapshot、Receiptの本文へcredential、access token、cookie、外部Ownerの不要な個人情報を複写しない。provider signed exportが利用できる場合はProfileどおりに検証するが、GitHub artifact attestationをRegistryの空集合証明へ流用しない。artifact attestationは実際のrelease artifactのbuild provenance用であり、artifactが存在した時だけ別Evidenceとして検証する。

### 4.6 Foundation Pointer／Memory contractのclean-break境界

[Memory／Pointers](memory-pointers.md)と[Product plan](../00-product/product-plan.md)が定めるMath／Memoryの責務分離は、`subject_id=foundation.pointer-memory-contract`の**planning-only** clean-break候補とする。これは旧combined Work Package／Capabilityを新しいMath CoreとMemory／Pointersの二つのcurrent identityへ置換する対象であり、旧ID、type alias、dual reader、implicit conversion、旧path redirectをtarget current集合へ残さない。

この文書更新は`CompatibilityConsumerInventoryV1`、Compatibility ChangeSet、MCD migration、runtime reader、external通知をmaterializeしない。したがって、current Architectureの旧combined IDが除去されたことだけを外部consumerゼロや実装適用の許可と解釈しない。実装または公開Schemaへ適用する前には、§2のrequired discovery scope全てについてconsumer inventoryをcompleteにし、未知consumerを`zero_verified`へ閉じる。

適用候補の`clean_break`は、少なくとも次を同一ChangeSetで検査する。

1. Product registry、Architecture Document Registry、MCD source、generated projection、current diagnostic／fixture inputに旧combined IDまたはaliasが0件である。
2. [Memory／Pointers](memory-pointers.md)の`PointerMemoryConsumerBindingV1`で列挙した全consumerが、新しいOwner、reference form、保存、job capture、retire、qualificationの正逆参照を閉じる。
3. Save、Replay、Package、Native ABI、external APIにlive pointer、lease、span、allocator object、旧IDを擬似互換fieldとして保存しない。
4. externalまたはretained consumerが一件でも見つかった場合は、clean breakを勝手に例外化せず、対象だけを`versioned_reader_migration`または`external_api_deprecation`として新しいChangeSetへ分離する。

この対象はFoundationの一般語彙とProduct registry identityの更新であり、Entity、Asset、GPU、Physics、Audioなど個別Subsystemの固有handle／retire意味を移管しない。それらは各Ownerがbindingを更新して閉じる。

## 5. Aliasと命名の規則

次はcurrent boundaryで禁止する。

- 旧type名を新typeへtypedef、using、JSON alias、MCD alias、operation aliasとして残すこと。
- readerがfield欠落を旧schemaと推測して補完すること。
- Asset subjectではないWorld Artifactへsynthetic Asset IDや`asset://` URIを与えること。
- raw memory layout、runtime slot、chunk row、native pointerを新しいpersistent identityとして扱うこと。
- documentationだけを更新し、Owner Registry revisionやaffected Contract refを旧Ownerのまま残すこと。

人間向けの説明では旧名称を「旧称」として一度だけ示せるが、copy-paste可能なcurrent API、Schema、ABI、diagnostic、AI tool inputには含めない。

## 6. 互換性検証

Compatibility ChangeSetは少なくとも次を検証する。

1. consumer inventory ref、subject、source format集合、record class集合、required scope集合、Evidence Requirement／satisfaction binding集合がChangeSetとexact一致し、Inventoryがcomplete／zero verifiedである。
2. required discovery scopeのset equality、scope固有Requirementのpass fulfillment、consumer endpoint、release boundary、source rebuild statusをread-backし、unknown consumerをゼロにする。
3. target Owner、Owner revision、Schema／format version、Contract set hashが全artifactとbindingに一致する。
4. clean breakではold reader、old writer、alias、synthetic identityがcurrent集合に存在しない。
5. recook／rebuildを同じSourceから二回実行した時、canonical output hashとdependency closureが一致する。
6. Save／Replay／Package／native ABIを含む場合は、各々のOwnerがmigrationまたはreject behaviorをfixtureで証明する。
7. rollbackはsource rebuild、有限old-reader window、release rollbackのいずれか一つを明記し、混在時の優先順位を明示する。
8. AI explain projectionはsource／target status、redaction、変更不能境界を区別し、旧aliasをcurrent suggestionとして返さない。

## 7. 非目的

本書はimplementation Task Plan、code change、cache削除、release migrationの実行を指示しない。各実施は対象Ownerの承認済み定義移行とProduct Work Packageの条件が満たされた後に初めて開始できる。
