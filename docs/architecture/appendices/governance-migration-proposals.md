# Governance Migration Proposals

- 文書ID: mirakan.appendix.governance-migration-proposals
- 文書種別: proposal appendix
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 親Owner: [Architecture Governance](../01-governance/architecture-governance.md)
- 正本範囲: 初回公開後のDefinition／Owner変更に使用できる非正本migration binding候補
- 非正本範囲: Governanceの安定規則、Compatibility policy、Runtime ECS semantics、実装Task、実装順序、生成済みArtifactまたは承認結果
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)
- 関連文書: [Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[Runtime ECS](../04-runtime/entity-component-system.md)
- 根拠区分: project-decision／provisional。本文の型、ChangeSet、Binding、Evidence集合はRepository Artifactが存在しない設計候補
- 外部根拠確認日: none

> 本書は実装計画ではない。候補Schemaを記載しても、Registry、Contract Set、Inventory、Receipt、Approval、ChangeSetまたはCapabilityが存在・承認・適用済みであることを意味しない。

## 1. Post-public Definition Migration binding candidate

initial V1がmaterializeまたは公開された後にArchitecture ChangeSetとProduct側のActive Definition migrationが必要になった場合、両者を相互のhash preimageへ戻すと循環する。そのため、候補設計ではsource／target closureとOwner reference集合を先に確定したSubjectを作り、Bindingと各wrapperはSubjectを一方向に参照する。現在のinitial V1 Architectureへこのmigration subject、source Owner、target Owner、Compatibility Receiptまたはold readerを生成しない。

```text
ArchitectureOwnerReferenceMigrationManifestV1
  manifest_id
  source_foundation_definition_closure_ref
  target_foundation_definition_closure_ref
  source_owner_ref
  target_owner_ref
  entries[]:
    reference_kind
    logical_subject_ref
    source_owner_ref
    target_owner_ref
  manifest_content_hash

ArchitectureDefinitionMigrationSubjectV1
  migration_subject_id
  base_inventory_ref
  resulting_inventory_ref
  ownership_transfer_refs[]
  compatibility_change_refs[]
  consumer_inventory_refs[]
  source_contract_set_ref
  target_contract_set_ref
  source_foundation_definition_closure_ref
  target_foundation_definition_closure_ref
  source_active_product_definition_sha256
  target_active_product_definition_sha256
  owner_reference_migration_manifest_ref
  evidence_requirement_refs[]
  evidence_satisfaction_bindings[]
  subject_content_hash

ArchitectureDefinitionMigrationBindingV1
  binding_id
  binding_version
  migration_subject_ref
  architecture_approval_ref
  binding_state: prepared | approved | rejected
  binding_content_hash
```

候補Manifestの`entries[]`は、source Closureから到達する移管対象Ownerのtyped reference全体とset equalityにする。`owner_id`の文字列検索、path、表示名または説明文をentryにしない。target側では全entryのtarget Owner refがtarget Closureのselected rowへ一意に解決し、旧revisionがcurrent Contract Set、Manifest、Diagnostic、Runtime Scope、Game Systemまたはgenerated bindingへ残らないことを検証対象にする。

Product Definition hashはProduct Ownerが発行する完成Definitionへのopaque equality anchorであり、GovernanceがProduct schemaやProduct stateを所有することを意味しない。Bindingを`approved`にできるのは、全参照と全Evidence Requirementのpass fulfillmentが同じSubjectへ束縛され、独立したArchitecture Approvalが存在する場合だけである。技術Evidenceを人間Approvalの代用にしない。

本候補を採用する場合でも、完成SchemaのOwner、canonical encoding、hash domain、size bound、signer／trust policy、retention、revocationおよびatomic current-pointer切替は別のArchitecture判断と実Artifactで確定する。この文書のMarkdown Schemaだけからhash、BindingまたはApprovalを発行しない。

## 2. Initial V1 exclusion

[Runtime ECS](../04-runtime/entity-component-system.md)を含む現Architecture Ownerは、最初のcanonical V1を各Owner文書へ直接定義する。Schema、Generator、serializer、Repository artifact、配布release、公開SDKまたは外部consumerが未materializeのsubjectについて、本Appendixのmigration bindingをbootstrap手段として使用しない。

初回公開後にOwner変更が必要になった時だけ、少なくとも次を同じclosed subjectへ束縛する。

- base／resulting Architecture Inventory ref。
- source／target authority document hashを含む一件のOwnership Transfer。
- complete Consumer Inventoryと承認済みCompatibility Change。
- source／target Owner Registry、Contract Set、Foundation Definition Closure。
- complete Owner Reference Migration Manifest。
- 全Evidence Requirementのpass satisfaction binding。
- Definition Migration Bindingと独立したArchitecture Approval。

この候補Schemaは実装Task、担当、見積り、作業順序、current migration、外部consumerの不存在またはCapability activationを意味しない。
