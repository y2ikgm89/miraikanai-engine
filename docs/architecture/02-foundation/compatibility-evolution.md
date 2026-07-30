# Miraikanai Engine Compatibility and Evolution Contract

- 文書ID: mirakan.arch.compatibility-evolution
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Architecture／Schema／artifact／runtime boundaryの互換性class、consumer inventory、clean-break procedure、reader／writer／aliasの可否、recook／rebuild／migration evidence、互換性ChangeSet
- 非正本範囲: 個別Domain Schemaのfield、Save payload、Package binary、runtime lease、Product Work Package、外部SDKのversion。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Executable Contracts](executable-contracts.md)
- 関連文書: [Architecture Governance](../01-governance/architecture-governance.md)、[Product Lifecycle](../00-product/product-lifecycle.md)、[Product Security](../01-governance/product-security.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Product plan](../00-product/product-plan.md)、[Executable contracts](executable-contracts.md)、[Memory／Pointers](memory-pointers.md)、[Project state](../03-authoring/project-state.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Runtime ECS](../04-runtime/entity-component-system.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Persistence／Save](../04-runtime/persistence-save.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-26

## 1. 結論

互換性は「旧名称を残すこと」ではない。保存済み入力、公開済みconsumer、外部API、配布済みartifactのどれに対して何を読めるかを、Owner、期間、検証、rollbackと共に明記する契約である。

[Product Lifecycle](../00-product/product-lifecycle.md)の`ProductUpdatePlanV1`は本書のCompatibility assessment、migration class、consumer inventory、rollback eligibilityをsame release／Project／CandidateのE2E updateへ束縛するconsumerであり、migration意味を再定義しない。同書の`ProductPublicationRecoveryPolicyV1`は本書が決定したeligibilityの後にCandidate publication、配布済みrelease、不可逆外部actionをorchestrateする別subjectであり、data reader／writer、migration、復元後semanticsまたはrollback可能性を所有しない。[Product Security](../01-governance/product-security.md)のsecurity updateも同じCompatibility契約を使用し、緊急性を理由にpartial migration、implicit reader、旧aliasまたはTarget置換を許可しない。

release済みconsumerを根拠にしない内部設計整理はclean breakを既定とする。旧type alias、dual reader、暗黙変換、synthetic identity、旧path redirectを残さず、committed Sourceから新形式を再生成する。公開済み互換性が必要な場合だけ、versioned migrationと有限のreader期間を承認済みCompatibility ChangeSetへ記録する。

設計文書にだけ存在し、対応するSchema、Generator、serializer、Repository artifact、配布release、公開SDKまたは外部consumerが一度もmaterializeされていないsubjectのrevisionは、互換性versionではない。この状態で名前、Fieldまたは構造を改める場合、current Ownerとcurrent supplementは最初のcanonical public／materialized versionを`V1`として直接定義し、旧draft名、migration、alias、reader、writerおよびconsumer inventoryを作らない。旧draft表記はimmutable ADRまたはreview transcriptだけに履歴として残せる。

`V2`以上へのversion bumpは、直前versionを消費するmaterialized Schema／Artifactまたは公開済みconsumerのRepository／release evidence、影響範囲、Compatibility class、reader／writer方針、rollback判断を同時に示せる場合だけ許す。設計の編集回数、日付、文書revision、候補比較、将来のmigration想定をversion bumpの根拠にしない。

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

## 4. Initial V1 direct-definition boundary

設計文書だけが存在し、Schema、Generator、serializer、Repository artifact、配布release、公開SDKおよび外部consumerが一度もmaterializeされていないsubjectは、§2のCompatibility Change subjectではない。Runtime ECS、Runtime Asset、Math Core、Memory／Pointers、Product Definition、Pack、Player Profile／Settingsを含むinitial V1 Architectureは、それぞれのOwnerが最初のcanonical identity、Field、version、参照方向を直接定義する。

initial V1では旧draft concept、source／target Owner、Owner revision migration、Product migration source、consumer migration manifest、migration history、alias、dual reader、旧path redirectまたはCompatibility Receiptを作らない。補助reviewは設計の検討履歴を説明できるが、current ArchitectureまたはDefinition Closureへ旧identityを登録せず、Owner文書のdirect definitionを上書きしない。

各Ownerはinitial V1 consumerをexact Owner／Definition refへ直接束縛し、次を検証する。

1. 同じidentity、type、fixed valueまたはoperationのcanonical Ownerがexactly oneである。
2. current Owner／Definition ref集合に旧draft名、近似名fallback、alias、source／target pairまたはV2以上の根拠なきversionが0件である。
3. unresolved Type、Schema、hash、FixtureまたはQualificationを文書だけからmaterialized／activeとみなさない。
4. Save、Replay、Package、Native ABI、external APIへlive pointer、lease、runtime slotまたはraw runtime layoutを保存しない。
5. initial V1がmaterializeまたは公開された後の変更だけを、新しい`CompatibilityConsumerInventoryV1`と承認済み`CompatibilityChangeSetV1`へ閉じる。

初回materialization後にretained Save、Package、Native ABI、external APIまたは配布済みconsumerが存在する変更では、§2～§3のrequired discovery scope、Evidence Requirement、reader／writer policy、rollbackを適用する。実consumerがないことはその時点のcomplete／zero-verified Inventoryでのみ証明し、Architecture review時のRepository観測や文書上の`absent`を将来の互換性判断へ流用しない。

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
