# Governance Migration Proposals

- 文書ID: mirakan.appendix.governance-migration-proposals
- 文書種別: proposal appendix
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 親Owner: [Architecture Governance](../01-governance/architecture-governance.md)
- 正本範囲: Definition移管bindingとRuntime ECS Owner移管ChangeSetの未承認候補
- 非正本範囲: Governanceの安定規則、Compatibility policy、Runtime ECS semantics、実装Task、実装順序、生成済みArtifactまたは承認結果
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)
- 関連文書: [Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[Gameplay programming model](../03-authoring/gameplay-programming-model.md)、[Runtime ECS](../04-runtime/entity-component-system.md)、[Runtime ECS Design Closure Review](runtime-ecs-design-closure-review.md)
- 根拠区分: project-decision／provisional。本文の型、ChangeSet、Binding、Evidence集合はRepository Artifactが存在しない設計候補
- 外部根拠確認日: none

> 本書は実装計画ではない。候補Schemaを記載しても、Registry、Contract Set、Inventory、Receipt、Approval、ChangeSetまたはCapabilityが存在・承認・適用済みであることを意味しない。

## 1. Definition Migration binding candidate

Architecture ChangeSetとProduct側のActive Definition migrationを相互のhash preimageへ戻すと循環する。そのため、候補設計ではsource／target closureとOwner reference集合を先に確定したSubjectを作り、Bindingと各wrapperはSubjectを一方向に参照する。

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

## 2. Runtime ECS canonicalization candidate

Runtime ECSのOwner移管は、次の候補ChangeSetで検討する。これはmaterialize前のreview profileであり、完成`ArchitectureChangeSetV1`、実装指示またはCapability activationではない。

```text
RuntimeEcsCanonicalizationChangeSetV1
  change_set_id: architecture.runtime_ecs.canonicalization.v1
  state: review
  contract_activation_effect: none
  source_owner_selector:
    owner_id: owner.core.runtime_ecs
    owner_revision: 1
    authority_document_id: mirakan.arch.gameplay-programming-model
  target_owner_selector:
    owner_id: owner.core.runtime_ecs
    owner_revision: 2
    authority_document_id: mirakan.arch.runtime-entity-component-system
  moved_concept_ids:
    - runtime_entity_identity
    - runtime_component_contract
    - runtime_archetype_layout
    - runtime_query_and_selection
    - runtime_component_access_manifest
    - runtime_structural_transaction
    - runtime_ecs_ai_contract_graph
  retained_source_concept_ids:
    - gameplay_definition
    - game_system_authoring
    - generated_gameplay_bundle
```

候補を完成ChangeSetへ昇格する前に、少なくとも次の同一closureが必要である。

- base／resulting Architecture Inventory ref。
- source／target authority document hashを含む一件のOwnership Transfer。
- complete Consumer Inventoryと承認済みCompatibility Change。
- source／target Owner Registry、Contract Set、Foundation Definition Closure。
- complete Owner Reference Migration Manifest。
- 全Evidence Requirementのpass satisfaction binding。
- Definition Migration Bindingと独立したArchitecture Approval。
- [Runtime ECS Design Closure Review](runtime-ecs-design-closure-review.md)のcurrent化必須`open-blocker`が0件で、同Reviewが要求するmachine-readable Schema、canonical encoding、hash golden vector、cross-owner正逆参照がtarget Foundation Definition Closureへ含まれること。

`applied`になる前は、`owner.core.runtime_ecs` revision 1と[Gameplay programming model](../03-authoring/gameplay-programming-model.md)がcurrent authorityである。[Runtime ECS](../04-runtime/entity-component-system.md)はtarget Ownerであり、文書の存在を実装、登録または移管完了と解釈しない。

### 2.1 Current readiness

2026-07-27時点のRepositoryでは、この候補を承認または適用する入力を確認できない。

| Closure入力 | 確認状態 | 扱い |
|---|---|---|
| immutable Architecture Inventory | 生成Schema、Generator、Artifactがない | `absent` |
| source／target Owner Registry、Contract Set、Foundation Definition Closure | content-addressed Artifactがない | `absent` |
| complete Consumer Inventory／Compatibility Change | 完成ArtifactとApprovalがない | `absent` |
| Owner Reference Migration Manifest | source Closureがない | `absent` |
| Product active Definition anchors | 完成Definition Artifactがない | `absent` |
| Evidence fulfillment／Architecture Approval | trusted ReceiptとApproval refがない | `absent` |
| Runtime ECS design closure | Closure registerは存在するがcurrent化必須`open-blocker`が残る | `incomplete` |
| Definition Migration Binding／Architecture ChangeSet | 上記入力が不足 | 発行禁止、`contract_activation_effect=none` |

このsnapshotは実装Task、担当、見積りまたは作業順序を定義しない。外部consumerが存在しないことも証明しない。Consumer調査が必要になった場合は、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)のauthority profile、snapshot、receipt境界を使用する。
