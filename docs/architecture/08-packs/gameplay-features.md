# Miraikanai Engine Gameplay Feature Packs

- 文書ID: mirakan.arch.pack-gameplay-features
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: reusable Gameplay Featureの共通ownership、manifest、compatibility、Public Contract、State owner、Command／Event／Snapshot、Save／Replay、failure、qualification意味
- 非正本範囲: 具体Weapon／Damage／Vital／Score／Encounter／Pickup／Locomotion／RPG Feature Schema、Registry、Fixture、Genre composition、Pack lifecycle、Subsystem契約、Product roadmap
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Pack Contract](pack-contract.md)、[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Collision](../05-simulation/collision.md)、[Physics](../05-simulation/physics.md)、[Navigation](../05-simulation/navigation.md)
- 関連文書: [Feature Definition／Fixture Candidate Catalog](../appendices/gameplay-feature-definition-fixture-catalog.md)、[RPG Genre Pack](rpg.md)、[Scenario／Stage](scenario-stage.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[Debugging／Replay](../04-runtime/debugging-observability-replay.md)、[Input](../07-platform/input.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-27

## 1. 結論

Gameplay Feature PackはGenre非依存で再利用可能なCapabilityとPublic Contractを提供する。Platformer、Puzzle、action、simulation、Genre PackなしのProjectが同じFeatureを利用できる。

FeatureはEngine CoreへPlayer、Character、Weapon、Enemy、Score、Level等の固定階層を追加しない。必要なProjectだけがFeature Packを選び、PackなしのProjectもvalidである。

具体Feature Schema、Registry、Fixture候補は[Feature Definition／Fixture Catalog](../appendices/gameplay-feature-definition-fixture-catalog.md)へ分離する。

## 2. Ownership、manifest、compatibility

各Feature Packはstable `pack_id`、owner ref、version、provided Capability、required Pack、public contract、runtime port、configuration profile、authoring operation、validator、migration、fixture refsをmanifestで宣言する。

Featureの型、State、System、Diagnosticには一意Ownerを割り当てる。Gameplay Programming Model、Collision、Physics、Navigation、Input等の共通契約を複写せず、version／hash付きrefで利用する。

Packの追加・更新は次を満たす。

1. Required Pack graphがacyclicである。
2. Public Contract、Port、State owner、Save／Replay、failureが完全である。
3. FeatureなしのProjectを無効にしない。
4. 同一authoritative fieldのwriterを複数Featureへ割り当てない。
5. Contract set、Pack manifest、Validator、Fixture集合を同じCandidateへ束縛する。
6. 文書に候補があるだけでPackまたはCapabilityをactiveにしない。

## 3. Canonical data model

Feature dataはSource Definition、Runtime State、Command、Event、Snapshotを分離する。Definitionはimmutable content、StateはOwner scope内のmutable value、Commandはintent、Eventは発生済みfact、Snapshotはseal済み観測である。

Weapon、Fire Mode、Projectile、Ammo／Reload、Damage、Team、Vital、Score、Encounter、Pickup等の具体候補は[Feature Definition／Fixture Catalog](../appendices/gameplay-feature-definition-fixture-catalog.md#3-canonical-data-model)を参照する。

共通不変条件は次のとおりである。

- Source identity、Runtime instance identity、Entity handle、Asset refを混同しない。
- Physical unit、bound、closed enum、nullable branchを明示する。
- Physics／Collision／Navigation payloadをFeature Schemaへ複写しない。
- Definition変更を既存Runtime Stateへ黙示適用しない。
- Unknown Feature field、version、owner、hashをfail-closedで拒否する。

## 4. Game SystemとState owner

Feature-owned Game Systemは[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md#3-gamesystemspecv2)の`GameSystemSpecV2`を使用し、Feature owner、State owner、phase、read／write access、Port、implementation set、Budget、failureを宣言する。

System priorityまたはPack load orderだけでwriter authorityを決めない。同一fieldへ複数contributionが必要な場合は、Ownerがcontribution Type、selection／merge policy、final publisherを一件だけ定義する。

### 4.1 Feature-owned System

<a id="41-character-locomotion-binding-system"></a>

Character Locomotion等のFeature Systemはintent contributionを生成し、Navigation／Physics／Animationのauthoritative executorを直接置換しない。Capability selection、provider identity、final intent batchはSubsystem OwnerのRegistry／selection documentへ従う。

Character Locomotionの具体Spec、Requirement、Policy、Registry、Fixture候補は[Feature Definition／Fixture Catalog](../appendices/gameplay-feature-definition-fixture-catalog.md#41-feature-owned-system)へ分離する。`mirakan.arch.pack-gameplay-features#41-character-locomotion-binding-system`はこのOwner境界を参照するstable rationale anchorである。

### 4.2 Runtime data flow

標準flowは次である。

1. Input／AI／Project Systemがtyped Commandまたはintentを生成する。
2. Feature SystemがDefinitionとStateを読み、bounded contributionを作る。
3. Subsystem Ownerがselection、simulation、collision、motion等を解決する。
4. Feature Ownerが結果EventとFeature Stateを更新する。
5. Seal済みSnapshotをUI、Replay、Debugへ公開する。

FeatureはSubsystemのlive memory、raw pointer、mutable leaseを保持しない。

### 4.3 Transaction

Fire、Damage、Pickup、Score等のstate-changing actionはexpected State generation、Definition／Policy ref、authorization、input closureを束縛し、全成功または変更0件にする。Collision、Physics、Inventory等の別Owner更新を途中状態で公開しない。

### 4.4 Reusable RPG Feature family

RPG-first Product方向は新しいGeneric Core hierarchyを作らない。次の五familyをGenre非依存のReusable Featureとして本書の共通ownershipへ解決し、[RPG Genre Pack](rpg.md)はcomposition、Profile、Game Flow、command roleだけを所有する。

| Feature family | 一意Owner責務 | 他Ownerとの境界 |
|---|---|---|
| Command Battle | battle instance、turn ownership、legal command、deterministic resolution、interrupt、outcome、failure | Schedulingのglobal cadenceを変更せず、Input／AIからtyped Commandを受ける |
| Actor Progression | progression State、experience／level policy、derived-stat publication、cap、migration | Character／Entityの固定Core型を作らず、consumerへsealed Snapshotを公開する |
| Inventory／Equipment | item ownership、stack／capacity、equipment slot、equip／unequip／item-use transaction | Asset identity、UI presentation、Pickup／Grant producerを所有しない |
| Dialogue／Quest | dialogue choice、condition、quest objective／State、causality、Save identity | UI／Localization／Scenario／Worldへtyped refで接続する |
| Currency／Shop | currency ledger、offer／price policy、purchase／sell transaction、reject atomicity | UI、Product balance、store／platform commerceを所有しない |

五familyは一つのOwner文書に集約できるが、logical Owner、State、Command、Event、Snapshot、Save projection、failureを相互に混ぜない。Feature間の操作は各Ownerのprepared resultを一つのbounded transactionへ束縛し、全precondition成功時だけpublishする。equipment変更とderived stat、inventoryとcurrency、dialogue choiceとquest、battle commandとturn advanceを片側だけcommitしない。

これはOwner responsibilityのtarget designであり、Stable ID、Schema、Registry、Operation、CapabilityまたはFixtureをcurrentへ追加しない。ShooterのWeapon／Score／Pickup／Game Flowを名前変更してRPG Featureへ流用せず、意味一致する既存Public Contractだけを明示参照する。具体Feature contractをmaterializeする場合は、各familyについてDefinition／State／Command／Event／Snapshot、transaction、Save／Replay、Diagnostic、negative fixtureを同じreview changeへ閉じる。

## 5. Command、Event、Snapshot

Commandはrequest identity、actor／subject、Definition／Policy ref、requested advance、idempotencyを持つ。Eventはaccepted action、outcome、causality、advanceを持つ。SnapshotはFeature Stateのseal済みread modelである。

同じTypeをCommand／Event／Snapshotへ兼用せず、EventからCommandを推測しない。Unknown、stale、duplicate、wrong-owner requestをtyped resultで拒否する。

具体Type候補は[Feature Definition／Fixture Catalog](../appendices/gameplay-feature-definition-fixture-catalog.md#5-commandeventsnapshot)を参照する。

## 6. Save、Replay、Migration

SaveはFeature-owned persistent Stateだけを保存し、live Entity handle、Physics body、Navigation query、animation poseを含めない。Replayはinput、accepted Event、Definition／Policy identity、必要なseal済みSnapshotを記録する。

Migrationはsource／target Feature version、State mapping、Consumer Inventory、fallback、Fixture、Evidenceを必要とする。Missing Featureを別Featureまたはdefault値へ黙示変換しない。

## 7. FailureとDiagnostic

Feature共通failureはinvalid definition、unknown ref、stale generation、owner mismatch、capability unavailable、conflicting writer、subsystem rejection、budget exceeded、migration requiredを区別する。

Diagnostic code、severity、safe parameter、remediationは[Executable Contracts](../02-foundation/executable-contracts.md#121-mirakandiagnosticv1)へ従う。具体Feature Diagnostic候補は[Feature Definition／Fixture Catalog](../appendices/gameplay-feature-definition-fixture-catalog.md#7-failureとdiagnostic)を参照する。

Failure時はlast-valid Definition／Stateを維持し、partial Event、partial Score、partial inventory、partial projectile publicationを残さない。

## 8. TestとQualification

Feature qualificationは次を含む。

- Schema／Contract round-trip、unknown／bound／owner／hash negative
- Featureあり／なし、Feature組合せ、dependency graph
- Command重複、stale generation、wrong Target、failure rollback
- Subsystemとのwriter authority、selection、typed Port整合
- Save／Load、Replay、Migration
- Determinism、Budget、soak、capacity
- AI proposalのauthorization、validator、negative Eval

具体Fixture候補は[Feature Definition／Fixture Catalog](../appendices/gameplay-feature-definition-fixture-catalog.md#8-testとqualification)へ分離する。候補件数をQualification passと扱わない。

## 9. Feature authoring operation

Feature authoring Operationは完全なMCD Operation、Service authority、Policy、Validator、Diagnostic、Receipt、positive／negative Qualificationを同じContract setへ登録した場合だけactivateできる。未登録候補はplanning recordであり、Provider／MCPへ公開しない。

## 10. 公式資料

Subsystemの外部仕様は各Subsystem Ownerが所有する。本書は外部EngineのGameplay class hierarchy、API、保存形式を一般Feature契約として採用しない。
