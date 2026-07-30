# Product Data Flow MCD Kind

- 文書ID: mirakan.decision.product-data-flow-mcd-kind
- 状態: review
- 正本範囲: Product data-flow identityをMCD common kind／Contract setへ収容し、Privacy payload Ownerと共通Envelope Ownerを分離する判断
- 非正本範囲: MCD共通kindのcurrent closed set、Data Flow payload Schema、Inventory、Privacy acceptance、materialized Contract set。各Owner文書を参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Product Plan](../00-product/product-plan.md)、[Product Privacy／Data Governance](../01-governance/product-privacy-data-governance.md)
- 外部根拠検証日: none
- 文書種別: Architecture Decision／cross-owner identity
- Decision owner document: `mirakan.arch.architecture-governance`
- Decision日: 2026-07-30
- Supersedes: none

## Context

Active Product DefinitionとPrivacy InventoryはProduct-owned data flowの完全membershipを同じidentityで閉じる必要がある。一方、Product PlanはPrivacy Ownerへ規範逆依存できず、Privacy-local `{id, version, content_hash}` RefをProduct rootへ複写すると二重authorityになる。

従来記述は`McdContractRefV1(kind=data_flow)`を使用していたが、Executable ContractsのMCD common kind closed setに`data_flow`が存在しなかった。この状態ではRefがContract setのcomplete recordへ解決できず、Product DefinitionとPrivacy Inventoryのset equalityも成立しない。

## Decision drivers

1. Product Plan、Privacy Inventory、Acceptanceが同じContract set rootの一つのRefを使用する。
2. 共通Envelope／RefとPrivacy payload semanticsのOwnerを分離する。
3. data flowを既存kindへ意味偽装せず、Domain-local kind／Refも作らない。
4. initial V1未materializeのためlegacy alias、dual reader、migrationを設けない。

## Considered options

### A. Privacy-local RefをProduct Planへ複写する

却下する。Product PlanからPrivacyへの逆依存または同じidentityの二重定義が必要になり、Contract set rootも保持できない。

### B. `kind=profile | type | requirement`のいずれかへ収容する

却下する。Data FlowはTarget／Toolchain Profile、payload Type、Requirementのいずれでもなく、kindによるvalidator／Owner routing／discoveryを誤らせる。

### C. MCD common kindへ`data_flow`を追加し、Privacyがpayloadだけを所有する

採用する。Executable ContractsがEnvelope／Ref、Privacyがcomplete payload／Inventory／acceptance、Product Planがopaque membershipだけを所有できる。

## Decision

1. Executable ContractsのMCD common kind closed setへ`data_flow`を追加する。
2. external identityは`McdContractRefV1(kind=data_flow) {id, version, contract_set_hash}`だけとする。
3. `ProductDataFlowDefinitionV1`は`McdRecordV1(kind=data_flow).payload`であり、Privacy Ownerだけがcomplete payload semanticsを定義する。
4. Product Definition、Privacy Inventory、Consent、Required Scope Projection、Acceptanceの全Data Flow refを同じgeneric MCD Refへ統一する。
5. Privacy-local narrow Ref、既存kind alias、bare endpoint、ID-only、`latest`、legacy readerを設けない。

## Consequences

- MCD common kind closed setはinitial V1 target designで10種類になる。
- FoundationはPrivacy payloadを解釈せず、Privacyは共通Envelope／Contract set identityを再定義しない。
- Product DefinitionとPrivacy InventoryはRef全Fieldでset equalityを行える。
- 現RepositoryにはMCD Schema、Contract set、Data Flow record、Inventory、resolver、AcceptanceまたはReceiptはmaterializeしていない。

## Canonical Owner documents

- MCD common kind／Envelope／Ref: [Executable Contracts](../02-foundation/executable-contracts.md)
- Product data-flow payload／Inventory／Acceptance: [Product Privacy／Data Governance](../01-governance/product-privacy-data-governance.md)
- Product-required membership: [Product Plan](../00-product/product-plan.md)
- Owner routing: [Architecture Governance](../01-governance/architecture-governance.md)

## Supersedes／Superseded by

- Supersedes: none
- Superseded by: none

## Verification

- Executable Contractsのclosed kind setに`data_flow`がexact一件存在する。
- Privacy-local narrow Data Flow RefのSchema定義が存在せず、全consumerがexact `McdContractRefV1(kind=data_flow)`を使用する。
- Product DefinitionとPrivacy InventoryのData Flow集合がRef全Fieldでset equalityになる。
- 規範依存graphにProduct Plan→Privacyの逆edgeがない。

## Official or primary sources

- none（外部仕様はCanonical Owner documentsへ委譲する）

このDecisionはOwner／identity境界だけを決める。MCD kind、Data Flow record、Inventory、Schema、resolver、Fixture、ReceiptまたはAcceptanceが実装／materializeされたことを意味しない。
