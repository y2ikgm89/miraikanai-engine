# Product Lifecycle／Product Security Ownership

- 文書ID: mirakan.decision.product-lifecycle-security-ownership
- 状態: review
- 正本範囲: Product Lifecycle、Product Security、Scene composition、restart-based Native iterationのOwner選択とscope維持判断
- 非正本範囲: 各OwnerのSchema／Operation／failure／Qualification、Product Phase／Work Package、実装Task、実装順序、担当、工数、日程、Capability activation。各Owner文書を参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Product Plan](../00-product/product-plan.md)、[Product Lifecycle](../00-product/product-lifecycle.md)、[Product Security](../01-governance/product-security.md)、[World](../06-rendering/world.md)、[Native Game Module](../03-authoring/native-game-module.md)、[Architecture Plan Closure Review](../appendices/architecture-plan-closure-review.md)
- 外部根拠検証日: 2026-07-29
- 文書種別: Architecture Decision／cross-owner ownership
- Decision owner document: `mirakan.arch.architecture-governance`
- Decision日: 2026-07-29
- Supersedes: none

## Context

C++ Game Engine必須機能の外部回答とArchitecture全Ownerを照合した結果、Rendering、Runtime、Authoring、Platform等のdomain機能はほぼ既存Owner、明示的非目標または安全な意味置換でcoverageされていた。一方、第三者Developer向けProject bootstrap／Documentation／update／supportの製品横断compositionと、Product全体のvulnerability response／disclosure／incident ownershipには一意のOwnerがなかった。

既存Ownerへ横断責務を分散追加すると、Project StateがTemplate製品責任、ToolchainがNOTICE presentation、Debuggingがsupport policy、AI SecurityがProduct全体のvulnerability responseを重複所有する。外部機能名ごとに新Subsystemを追加すると、Plugin、Script VM、Multiplayer等を現Product scopeへ誤って昇格させる。

## Decision

1. `mirakan.arch.product-lifecycle`を追加し、Project bootstrap、Template／Sample／Documentation release binding、GUI／CLI／headless parity、update composition、repair／support window、NOTICE presentation、Product lifecycle E2E acceptanceを所有させる。
2. `mirakan.arch.product-security`を追加し、threat ownership registry、Product security baseline、vulnerability case、security update、disclosure、notification、Product security incidentを所有させる。
3. 両Ownerはdomain SchemaまたはEvidence envelopeを複写せず、各Ownerのexact artifact／Receiptをsame release、Project revision、Candidate、Target、caseへ束縛する。
4. `Scene Source`の再利用instance／nested composition／typed override／explicit rebaseはWorld Ownerを拡張して閉じ、`Prefab`という第二Ownerまたはlegacy aliasを作らない。
5. Windows Preview iterationは既存Native Game Moduleのrestart-based contractを維持し、新しいHot Reload Capabilityを作らない。
6. Plugin ecosystem、汎用Script VM／JIT、Multiplayer、large-world機能を現在のMVP必須要件へ追加しない。

## Alternatives

### A. Product PlanとAI Securityへ追記する

却下する。Product PlanはProduct intent／Gate、AI SecurityはAI task authorizationが正本であり、lifecycle Schemaまたは全Product vulnerability caseを所有すると責務が肥大化する。

### B. 各domain Ownerへ横断責務を分散する

却下する。Template、Documentation、update、support、security disclosureが複数Ownerへ分裂し、same-candidate closure、accountable Owner、failure時のlast-known-goodが一意にならない。

### C. 外部Engineの機能分類ごとにSubsystemを追加する

却下する。名称coverageは増えるが、Miraikanaiのscope、C++23＋GameplayDefinition、安全境界、Generic Core＋Pack構造と一致しない。

## Consequences

- Architecture Owner文書は53件から55件になる。
- Product LifecycleとProduct Securityのcurrent実装状態は`absent`、文書状態は`review`であり、追加だけでCapabilityが利用可能にはならない。
- `ARCH-C21`はOwner不在の`open-decision`から`closed-in-target-design`へ変わるが、Registry、Schema、Operation、Fixture、Receiptは未materializeのまま残る。
- `ARCH-C03`のArchitecture Inventory／Explain Projection blockerは維持する。
- World、Mobile Common、Windowsの既存Ownerには意味閉包と二つの明白な不整合修正だけを行う。
- 未公開内部SchemaはcleanなV1を直接定義し、旧称aliasまたはfallbackを作らない。将来公開済みProject／Save／Package／ReleaseはCompatibility Ownerのversioned migrationに従う。

## Verification

- Architecture Index、Owner header、文書ID、相対link、規範依存graphを再監査する。
- Product Lifecycle、Product Security、Scene compositionのOwner／非Owner、failure、Qualificationをcross-owner reviewする。
- Mobile Application stateから`SurfaceUnavailable`、Windows workspace mappingからlegacy `build/`が消えていることを確認する。
- Native Game ModuleをHot Reload不足と扱う記述、`Prefab` canonical alias、Plugin／Script VM／Multiplayerのscope追加がないことを確認する。

このDecisionは設計Ownerと責務境界だけを決める。実装Task、実装順序、担当、工数、日程またはCapability Activationを決めない。
