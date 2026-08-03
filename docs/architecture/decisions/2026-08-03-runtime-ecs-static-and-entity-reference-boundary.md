# Runtime ECS Static Definition／Entity Reference Boundary

- 文書ID: mirakan.decision.runtime-ecs-static-entity-reference-boundary
- 状態: review
- 正本範囲: ECS access Manifestの静的phase identityと、snapshot-bound／cross-advance Entity Refを分離する判断理由
- 非正本範囲: exact Type／Field、runtime resolve規則、Scheduling phase値、Native ABI、Persistence identity、実装Task、実装順序、C++表現。各Owner文書を参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Runtime ECS](../04-runtime/entity-component-system.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Native Game Module](../03-authoring/native-game-module.md)、[Persistence／Save](../04-runtime/persistence-save.md)
- 外部根拠検証日: none
- 文書種別: Architecture Decision／Runtime ECS identity boundary
- Decision owner document: `mirakan.arch.runtime-entity-component-system`
- Decision日: 2026-08-03
- Supersedes: none

## Context

content-addressedな`RuntimeComponentAccessManifestV1`が実行中のSimulation Advanceを指す`RuntimeTimeRefV1`を保持すると、静的System Definitionをadvanceごとに再生成するか、dynamic identityをstatic hashへ混在させる必要がある。また、publication generationへ固定した一種類のEntity Refをcommand／event／async resultへ使うと、対象Entityが存続していても次のpublicationで無条件にstaleになる。

必要なのは、静的permission、seal済みsnapshotの参照、同runtime session内で将来deliveryされる参照、永続identityを別のlifetimeとauthorityへ分けることである。

## Decision drivers

- content-addressed DefinitionのbytesをSimulation Advanceから独立させる。
- publication固定readとcross-advance deliveryのstaleness規則を型で区別する。
- Entity destroy／slot reuseは検出しつつ、無関係なpublicationでlive targetを失効させない。
- process-local RefをSave、Replay、Package、cross-session identityへ流用しない。
- Ref解決成功をComponent access、lease、writeまたはGameplay authorityにしない。

## Considered options

### A. Manifestをadvance生成artifactにする

採用しない。System Definitionと実行instanceのidentity、publication authority、hash lifecycleを不必要に結合し、同じpermission集合をadvanceごとに再materializeする。

### B. 一種類のpublication固定Entity Refだけを使う

採用しない。snapshotの厳密性は得られるが、command／event／async resultの通常deliveryまで次publicationでstaleにする。

### C. 一種類のlive Entity Refからpublication Fieldを除去する

採用しない。cross-advance carrierは成立するが、seal済みsnapshotと同じEntity generationを厳密に照合する型を失う。

### D. 静的phase IDと二種類のprocess-local Entity Refへ分離する

採用する。ManifestはScheduling Ownerのserialized phase IDだけを保持し、runtime leaseがexact Time Refを持つ。Entity Refはsnapshot-boundとlive deliveryへ分け、永続identityはPersistence Ownerへ委譲する。

## Decision

1. content-addressed ECS access Manifestとstructural permissionは静的`TickPhaseId`だけを保持する。
2. callback／lease／token／delta等の実行時carrierだけがexact `RuntimeTimeRefV1`を保持する。
3. seal済みpublicationに固定するEntity参照と、同runtime sessionでcallback／advanceを越えるcurrent Entity参照を別Typeにする。
4. snapshot参照はWorld epochとpublication generationを含むexact resolve、live参照はdelivery時のWorld instance／Entity generation／alive stateでresolveする。
5. どちらもprocess-localで、永続化または別session再同定にはPersistence OwnerのPersistent Entity Identityだけを使う。

## Consequences

- static Definition hashがadvance identityから独立する。
- snapshot consumerは暗黙rebaseを受けず、cross-advance consumerは無関係なpublicationだけで失効しない。
- destroy、World破棄、slot reuse、generation mismatchは引き続きfail closedになる。
- Native API等のconsumerは用途に応じて二Refを明示選択する必要がある。
- RepositoryにSchema、generator、runtime、Native ABI、FixtureまたはReceiptは存在せず、本Decisionは実装またはQualificationを主張しない。

## Canonical Owner documents

- exact Type／Field／resolve規則: [Runtime ECS](../04-runtime/entity-component-system.md)
- phase／Time Ref／advance: [Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)
- Project C++ projection: [Native Game Module](../03-authoring/native-game-module.md)
- durable／Replay identity: [Persistence／Save](../04-runtime/persistence-save.md)

## Supersedes／Superseded by

- Supersedes: none
- Superseded by: none

## Official or primary sources

外部仕様は使用していない。本DecisionはMiraikanai内部のlifetime／identity分離に関するproject-decisionである。
