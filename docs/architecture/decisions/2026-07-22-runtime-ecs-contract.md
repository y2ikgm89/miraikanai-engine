# Miraikanai Engine Runtime ECS Contract Decision

- 文書ID: mirakan.decision.runtime-ecs-contract
- 文書種別: Architecture Decision／target contract canonicalization
- 状態: review
- 正本範囲: Engine-owned Runtime ECS採用判断、正本責務の分割先、clean-break採否、current化の承認境界、外部比較資料から採る原則
- 非正本範囲: ECS Schema・固定値・runtime挙動、Package binary、Save／Replay schema、AI authorization、実装Task Plan、実装。各正本を参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[Runtime ECS](../04-runtime/entity-component-system.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Persistence／Save](../04-runtime/persistence-save.md)、[Gameplay programming model](../03-authoring/gameplay-programming-model.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)
- 外部根拠検証日: 2026-07-24

## 1. 決定

Miraikanai Engineは、Engine-owned archetype／SoA Runtime ECSをRuntime Worldの標準Entity／Component storageとして採用する目標設計を維持する。Flecs、EnTT、Unity Entities、Unreal Massのruntime、API、型、保存形式、scheduler、World semanticsは採用しない。

この判断を単一の巨大ADRで保持せず、正本を次の責務へ分割する。

| 正本 | 所有する内容 |
|---|---|
| [Runtime ECS](../04-runtime/entity-component-system.md) | Entity／Component identity、archetype／query／lease、access manifest、structural transaction、ECS AI graph |
| [Runtime Package](../04-runtime/runtime-package.md) | World Root／Section image、capacity record、package directory、loader、section publication |
| [Persistence／Save](../04-runtime/persistence-save.md) | Save、persistent identity projection、digest、reconstruction、Replay projection |
| [Asset lifecycle](../03-authoring/asset-lifecycle.md) | generic artifact manifest、tagged artifact subject、catalog、cook、promotion |
| [Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md) | phase、job DAG、boundary orchestration、callback lifetime |
| [Architecture Governance](../01-governance/architecture-governance.md) | Owner transfer、Architecture ChangeSet、AI Architecture projection |
| [Compatibility／Evolution](../02-foundation/compatibility-evolution.md) | clean break、reader／writer／alias policy、recook／migration evidence |

本Decisionはこの文書刷新を承認対象へまとめるものであり、ECSの実装、MCD／Operation登録、Package／Save readerのactivation、Task Planの作成を指示しない。

## 2. 採用理由

Runtime Worldには、data-only Component、明示的なState owner、決定論的なquery／structural boundary、Save／Replay projection、AIがboundedに説明できるcontract graphが必要である。Engine-owned設計なら、これらをMiraikanaiのMCD、Project revision、Artifact、Runtime boundary、AI approvalへ直接束縛できる。

archetype／SoAを選ぶ理由は、同じComponent集合の走査局所性、queryの宣言性、storage layoutの決定性、容量・fragmentationの測定可能性にある。16 KiB chunk、64-byte alignment、256-byte inline Component上限は外部Engine互換の値ではなく、Miraikanaiがqualificationで継続検証するtarget Contractである。

## 3. 採らない案

| 案 | 判断 |
|---|---|
| third-party ECS runtimeを標準storageにする | 不採用。MCD、Package、Save、AI projection、Owner Registryとの一意の責務境界が弱くなる |
| Entityごとのvirtual updateとheap objectを標準経路にする | 不採用。query、determinism、structural transaction、bounded AI explanationを一貫して保証できない |
| iteration中にlive Worldを直接構造変更する | 不採用。reference／query／cacheの無効化とpartial publicationを防げない |
| 旧名称・old reader・dual schemaを残す | 不採用。release consumerが必要な場合だけversioned migrationを別ChangeSetで承認する |
| ECS、Package、Save、AIを一冊の正本に集約する | 不採用。単一Owner原則と局所的なAI retrievalを損なう |

## 4. target不変条件

target Contractは次を一体で満たす。

1. Runtime Entityはtyped index＋generationであり、callback外ではWorld publicationを含むtyped refへ包む。
2. Componentはdata-onlyで、canonical field encodingから構築し、raw layoutをpersistent／digest／AI入力へ出さない。
3. 同じComponent集合をarchetype化し、chunk内をSoAで配置する。
4. query callbackは一chunkの連続row rangeだけを扱い、`deterministic_hash`でも複数chunkを一callbackへ束ねない。
5. read／write／structural permissionは`RuntimeComponentAccessManifestV1`で閉じ、selected row以外へのwriteを生成bindingで拒否する。
6. structural deltaはdeferし、全preconditionとcapacityを検査した後の単一boundaryでpublishする。
7. regular value writeはScheduler scope内だけで可視にし、外部consumerはseal済みsnapshot／publicationだけを読む。
8. Package、Save、Replay、AI captureはraw handle、pointer、chunk row、worker順を保存または入力にしない。
9. AIはimmutable contract graphとfield maskだけを読み、Task Authorization、sensitivity、route policyを迂回しない。

詳細なfield setとvalidationはこのDecisionに再掲せず、各正本だけが所有する。

## 5. 正本移管とcurrent化

`owner.core.runtime_ecs`のcurrent authorityは、[Architecture Governance](../01-governance/architecture-governance.md#4-runtime-ecs正本化changeset)に記す`RuntimeEcsCanonicalizationChangeSetV1`が`applied`になるまで[Gameplay programming model](../03-authoring/gameplay-programming-model.md)のrevision 1である。

targetのrevision 2へ移すには、少なくとも次が同じ承認closureで必要である。

1. [Compatibility／Evolution](../02-foundation/compatibility-evolution.md#42-ecs-consumer-inventory-boundary)のcomplete／zero-verified Consumer Inventoryと全scope Requirementのpass fulfillmentを発行し、unknown consumerを残さない。
2. Owner Registryのrevision 2、authority document、source／target Foundation Definition Closureを更新する。
3. affected MCD、Owner Manifest、Diagnostic、Runtime Scope、Game System、generated binding、retained artifactのowner refを`ArchitectureOwnerReferenceMigrationManifestV1`で一意に更新する。
4. ECS clean-breakのCompatibility ChangeSetと、そのConsumer Inventory refを承認する。
5. [Architecture Governance](../01-governance/architecture-governance.md#33-definition-migration-binding)のDefinition Migration bindingとProduct側Active Definition migrationが同じsource／target closureを参照することを承認する。
6. Asset Catalog、World Root／Section image、Runtime Package、Save／Replay fixtureをSourceから再生成する。
7. ECS、Package、Persistence、Scheduling、Debug、AI securityのqualification evidenceと全Evidence Requirementのpass satisfaction bindingをread-backする。

文書の追加やProduct Work Packageの表記変更だけではcurrent化しない。移管前に新正本を`review`として読める状態にすることと、runtime capabilityをactivateすることを分離する。

## 6. Clean break

targetの既定は`source_preserving_recook`である。旧`EntityHandle`、`ComponentAccessManifest`、suffixなし`DerivedArtifactManifest`、asset-only world artifact、ad-hoc Save／Replay projectionを、target名・target subject・target recordへ置き換える。

old alias、old reader、dual schema、synthetic Asset ID、redirect documentをcurrent boundaryへ残さない。committed Source、Asset Import document、承認済みRuntime entryだけからDerived Artifact、Catalog、World image、Package、fixtureを再生成する。公開済みSave、external Native ABI、配布済みPackageなど、source recookだけでは保護できないconsumerが見つかった場合は、このDecisionのclean break前提を停止し、versioned reader migrationを明示承認する。Inventoryが未完成、scope evidence不足、endpoint不明、またはsource rebuildが未検証である状態も「consumerなし」ではなくclean-break不成立として扱う。

## 7. AIとroute境界

ECSのAI explanationは[Runtime ECS](../04-runtime/entity-component-system.md#8-ecs-ai-contract-graph)の`RuntimeEcsContractGraphV1`と[Architecture Governance](../01-governance/architecture-governance.md#5-ai向けarchitecture-explain-projection)の`ArchitectureExplainProjectionV1`だけから行う。live World、live handle、native pointer、credential、raw path、unsealed partial writeを入力にしない。

route kindは[AI Security／Approval](../01-governance/ai-security-approval.md)の`engine_provider_adapter | standard_external_mcp | managed_external_host`だけである。ECS文書はroute alias、MCP grant、provider credential、Task Authorizationを定義しない。

このDecisionは新たな`operation.*` ID、Tool、MCP surfaceを追加しない。候補語彙は[Executable contracts](../02-foundation/executable-contracts.md)のclosed partitionで`not_activated`のままとする。

## 8. 比較資料

[Unity Entitiesのsync point資料](https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/performance-sync-points.html)からはstructural changeを予測可能なboundaryにまとめる原則を、[Flecs Queries資料](https://github.com/SanderMertens/flecs/blob/v4.1.2/docs/Queries.md)からはquery cacheとdeferの原則を、[EnTT Entity資料](https://github.com/skypjack/entt/blob/v3.16.0/docs/md/entity.md)からはiteration中変更によるreference無効化の注意を採る。

これらは比較用の公式一次資料であり、Miraikanaiの数値、API、schema、scheduler、binary formatの正本ではない。外部資料の更新は本Decisionを暗黙に変更せず、対象Ownerの外部根拠検証とChangeSetで評価する。

## 9. 承認前の検証

current化前に次を確認する。

1. 文書ID、Header、relative Link、Owner scope、Inventoryに重複と欠落がない。
2. 新正本がそれぞれ1,000行未満で、同一field・固定値・Gateを二重所有しない。
3. active仕様から非canonical route aliasと旧ECS type名が除去され、history／migration sourceだけに限定される。
4. query range、selection mask、write validation、structural transaction、value-write observabilityが型とfixtureで閉じている。
5. Package／Persistence／Asset Lifecycle／Debug／Schedulingの境界がraw handle、live memory、generic envelopeの重複を持たない。
6. review profileは`contract_activation_effect = none`のままcurrent化を行わず、完成Architecture ChangeSetだけがapproved Definition Migration bindingを参照する。
7. Consumer Inventory、全Evidence Requirementのpass satisfaction binding、Compatibility Change、Owner reference migration manifest、source／target Definition Closure、Product Active Definition migrationが同じclosureへexact解決する。
8. 実装開始の可否は、このDecisionのreview完了ではなく、approved definition migrationとProduct Work Packageの条件で判断する。

## 10. 計画書上の扱い

このDecisionは実装Task Planではない。ここで定めるのは、正本化を確認するArchitecture上の順序だけである。

1. GovernanceとCompatibilityでConsumer Inventory、Owner transfer、clean-break条件を閉じる。
2. ECS、Package、PersistenceでSchemaと境界を一意にする。
3. Owner reference migration manifest、Definition Migration binding、Product Active Definition migrationの参照を閉じる。
4. 既存Authoring、Runtime、Platform、Product文書の参照を移し、旧語彙と重複記述を除く。
5. Owner Registry／MCD／runtime実装を変更する必要があるかを、承認時に別途判断する。

この順序は実装を許可せず、実装task、担当、見積り、実行順、commitを含まない。
