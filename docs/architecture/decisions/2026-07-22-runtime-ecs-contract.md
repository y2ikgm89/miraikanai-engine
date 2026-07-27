# Miraikanai Engine Runtime ECS Contract Decision

- 文書ID: mirakan.decision.runtime-ecs-contract
- 状態: review
- 正本範囲: Engine-owned archetype／SoA Runtime ECSをRuntime Worldの標準Entity／Component storageとして選ぶ判断と、その判断理由
- 非正本範囲: ECS Schema・固定値・runtime挙動、Package binary、Save／Replay schema、Scheduling、Compatibility、AI authorization、qualification、実装Task。各Owner文書を参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[Runtime ECS](../04-runtime/entity-component-system.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Persistence／Save](../04-runtime/persistence-save.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)
- 外部根拠検証日: 2026-07-26
- 文書種別: Architecture Decision
- Decision owner document: `mirakan.arch.runtime-entity-component-system`
- Decision日: 2026-07-22
- Supersedes: none

## 1. Context

Runtime Worldには、Entityとdata-only Componentを標準的に格納し、queryと構造変更を決定論的な境界で扱えるstorageが必要である。これはSave／Replayへの安定したprojection、AIがboundedに説明できるcontract graph、およびlayout挙動を継続的に測定できる基盤にもなる。

本Decisionが選ぶのはRuntime ECSの方式だけである。Entity／Componentの現行Schemaと挙動はRuntime ECS、World imageとloaderはRuntime Package、Save／ReplayはPersistence、phaseとcallback lifetimeはScheduling、移行規則はCompatibility、AIの認可とrouteはAI Securityがそれぞれ所有する。本Decisionはそれらのauthorityを統合または置換しない。

## 2. Decision drivers

- Componentがdata-onlyであり、永続化や説明にnative layoutを露出しないこと。
- queryとstructural boundaryが明示され、決定論的なpublicationを構成できること。
- Save／Replayへowner-defined projectionを提供できること。
- AI explanationをimmutableかつboundedなcontract表現へ制限できること。
- storage layout、局所性、容量、fragmentationを同一条件で測定できること。

## 3. Considered options

| Option | Assessment |
|---|---|
| Engine-owned archetype／SoA | 採用。MiraikanaiのOwner境界、生成binding、publication、projection、qualificationへ直接束縛できる |
| third-party ECS runtime | 不採用。vendorのruntime semanticsとMiraikanaiのOwner、Package、Save、AI境界を分離し続ける必要がある |
| object-per-Entity virtual update | 不採用。走査局所性、明示query、structural publication、bounded explanationを標準経路として閉じにくい |
| iteration中の直接structural mutation | 不採用。reference、query、cacheの無効化とpartial publicationを予測可能な境界へ限定できない |
| ECS／Package／Saveを一つのauthorityに統合 | 不採用。単一Owner原則と局所的な変更・retrieval境界を損なう |

## 4. Decision

Miraikanai Engineは、Engine-owned archetype／SoA Runtime ECSをRuntime Worldの標準Entity／Component storageとして選択する。

この選択はvendor runtime、API、schemaとの互換性を採用しない。Unity Entities、Flecs、EnTT、Unrealの資料は比較根拠として利用するが、それらの型、World semantics、scheduler、保存形式、storage backendをMiraikanaiの契約にしない。現行の技術的なfield、値、Gate、runtime behaviorは本Decisionではなく各Canonical Owner文書だけが決定する。

## 5. Consequences

archetype／SoAにより、同じComponent集合の走査局所性、宣言的query、storage layoutの制御、容量とfragmentationの測定可能性を得られる。また、Miraikanai固有のMCD、Project revision、Artifact、Runtime boundary、Save／Replay projection、AI approvalへ一意に接続できる。

一方で、Engineはstorage、generated binding、structural publication、diagnosticを実装し、継続的にqualificationする費用を負う。外部ECS runtimeの更新でこれらの責任を代替できない。layoutと性能の採否は[Performance／Capacity](../04-runtime/performance-capacity.md)、移行とconsumer保護は[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、承認Evidenceとcurrent化は[Architecture Governance](../01-governance/architecture-governance.md)がそれぞれ閉じる。

## 6. Canonical Owner documents

| 正本 | 所有する内容 |
|---|---|
| [Runtime ECS](../04-runtime/entity-component-system.md) | Entity／Component identity、archetype／query／lease、access manifest、structural transaction、ECS AI graph |
| [Runtime Package](../04-runtime/runtime-package.md) | World Root／Section image、capacity record、package directory、loader、section publication |
| [Persistence／Save](../04-runtime/persistence-save.md) | Save、persistent identity projection、digest、reconstruction、Replay projection |
| [Asset lifecycle](../03-authoring/asset-lifecycle.md) | generic artifact manifest、tagged artifact subject、catalog、cook、promotion |
| [Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md) | phase、job DAG、boundary orchestration、callback lifetime |
| [Architecture Governance](../01-governance/architecture-governance.md) | Owner transfer、Architecture ChangeSet、AI Architecture projection |
| [Compatibility／Evolution](../02-foundation/compatibility-evolution.md) | clean break、reader／writer／alias policy、recook／migration evidence |

この表はauthorityへの案内であり、各OwnerのSchema、固定値、Gate、migration detailを本Decisionへ複写しない。

## 7. Currentization and compatibility

[Governance Migration Proposals](../appendices/governance-migration-proposals.md#2-runtime-ecs-canonicalization-candidate)がOwner移管、承認closure、Definition Migration binding、およびcurrent化の未承認候補をまとめる。同proposalが完成ChangeSetとして承認・適用されるまでは、target Runtime ECS文書をcurrent authorityまたは実装済みcontractとして扱わない。

[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)がRuntime ECS正本化の`source_preserving_recook`、consumer inventory、reader／writer／alias policy、migration evidenceを所有する。consumer保護と承認closureの詳細は両Owner文書へ委ね、本Decisionでは手順、record field、Gateを再定義しない。

## 8. Official comparison sources

- [Unity Entities archetypes](https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/concepts-archetypes.html)
- [Unity Entities structural changes](https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/concepts-structural-changes.html)
- [Unity Entities entity command buffers](https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/systems-entity-command-buffers.html)
- [Flecs Queries](https://github.com/SanderMertens/flecs/blob/v4.1.2/docs/Queries.md)
- [EnTT Entity documentation](https://github.com/skypjack/entt/blob/v3.16.0/docs/md/entity.md)
- [Unreal Engine common memory and CPU performance considerations](https://dev.epicgames.com/documentation/unreal-engine/common-memory-and-cpu-performance-considerations-in-unreal-engine)
- [Unreal Engine performance profiling and configuration](https://dev.epicgames.com/documentation/unreal-engine/introduction-to-performance-profiling-and-configuration-in-unreal-engine)

これらは選択肢の比較に用いる公式または一次資料であり、MiraikanaiのAPI、schema、storage値、scheduler、binary formatのauthorityではない。外部資料の更新は本DecisionまたはOwner契約を暗黙に変更しない。

## 9. Non-goals and relationships

本Decisionの`review`は、Runtime ECS capability、Package／Save reader、Shipping pathのactivationを意味しない。本Decisionは実装Taskや実装順を定義せず、MCD、Operation、Tool、MCP surfaceを登録またはactivateしない。

本DecisionがsupersedeするDecisionはなく、本DecisionをsupersedeするDecisionも現在はない。
