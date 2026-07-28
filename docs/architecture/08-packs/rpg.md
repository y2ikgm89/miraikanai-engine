# Miraikanai Engine RPG Genre Pack

- 文書ID: mirakan.arch.pack-rpg
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: RPG固有Genre composition、RPG Profile、Game Flow vocabulary、command role mapping、Reusable RPG Featureの組合せ、Reference Gameへの非Production fixture binding
- 非正本範囲: Generic Core、Reusable FeatureのPublic Contract／State／Schema、Scenario／Stage semantics、UI／Localization／Save／Replay、RPG Reference Gameのoriginal content／balance／World composition、Product outcome／Gate、current Registry／Operation／Capability activation、実装Task／順序
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Pack Contract](pack-contract.md)、[Gameplay Feature Packs](gameplay-features.md)、[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)、[Scenario／Stage](scenario-stage.md)
- 関連文書: [Product Plan](../00-product/product-plan.md)、[Shooter Genre Pack](shooter.md)、[Runtime ECS](../04-runtime/entity-component-system.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Persistence／Save](../04-runtime/persistence-save.md)、[Debugging／Replay](../04-runtime/debugging-observability-replay.md)、[Navigation](../05-simulation/navigation.md)、[Animation](../05-simulation/animation.md)、[World](../06-rendering/world.md)、[Input](../07-platform/input.md)、[Audio](../07-platform/audio.md)、[UI／Text／Localization／Accessibility](../07-platform/ui-text-localization-accessibility.md)
- 根拠区分: project-decision（未固定のFeature ID、Profile値、Fixture件数はprovisional）
- 外部根拠確認日: none

## 1. 結論と状態

RPGは`pack_kind=genre`のGenre Packであり、Generic Engine Core、Editor、AI、Build、Runtime、PlatformへRPG依存を追加しない。[Gameplay Feature Packs](gameplay-features.md)がcommand battle、actor progression、inventory／equipment、dialogue／quest、currency／shopの再利用可能Feature familyを所有し、本書はそれらのPublic Contract、State、transaction、Save／Replay、failureを再定義しない。

[Product Plan](../00-product/product-plan.md)はcompact 2D command RPGを最初のProduct-facing Reference Gameとして選び、MVP outcome、acceptance、Core holdout、Shooter sourceからRPG destinationへのProduct Definition Migrationを所有する。本書はReference Gameのoriginal content、balance、World layout、localized text、asset selectionまたはProduct completionを所有しない。Reference Gameは通常のGame Projectとして本Genre Packを消費し、Production PackからFixture／Reference Gameへ逆依存しない。

本書はtarget Owner設計であり、current Installed Product、Pack Registry、Capability Registry、Operation、Work Package、Receiptを変更しない。RPG Feature／Genre／Referenceは`not_activated`を維持し、Shooter IDのrename、Shooter Receiptの流用、未materialize IDのcurrent登録を行わない。

## 2. 四層境界

| 層 | 所有するもの | 所有しないもの |
|---|---|---|
| Generic Engine Core | Genre非依存のAuthoring／Runtime／Simulation／Rendering／Platform契約 | RPG turn、inventory、quest、shop、RPG flow |
| Reusable RPG Feature | FeatureごとのState、Command／Event／Snapshot、transaction、Save／Replay、failure | RPG Genre Profile、Reference content |
| RPG Genre Pack | Feature composition、Genre vocabulary、Profile、Game Flow、command role、fixture binding | Feature private state、Core contract、Reference content |
| RPG Reference Game | original content、balance、World composition、localized presentation、acceptance fixture | Pack／CoreのPublic Contract |

Game ProjectはRPG Genre Packを使わずFeatureを直接構成できる。RPG Genre Packは別Genre Packへ依存せず、Feature Packだけを参照する。複数Genreの合成はGame Projectが明示し、RPGからShooter、Platformer、Puzzle等への隠れた依存を作らない。

## 3. Reusable RPG Feature mapping

| Feature family | Feature Ownerの正本責務 | RPG Genreが所有するもの |
|---|---|---|
| Command Battle | battle instance／turn ownership、legal command validation、deterministic resolution、outcome、interrupt／failure | battleをRPG flowへ組み込むProfile、command role、presentation binding |
| Actor Progression | progression state、experience／level policy、derived-stat publication、cap／migration | RPG Profileで必要なFeature versionとrole mapping |
| Inventory／Equipment | item ownership、stack／capacity、equipment slot、equip transaction、item-use request | Genre vocabulary、default composition、UI role binding |
| Dialogue／Quest | dialogue node／choice、condition、quest objective／state、causality、Save identity | RPG flowとScenario／Worldへのcomposition |
| Currency／Shop | currency ledger、offer／price policy、purchase／sell transaction、reject atomicity | shop role、Genre Profile、Reference fixture binding |

表のFeature familyはOwner responsibilityを示し、current Stable ID、Schema、Registry、OperationまたはCapabilityを生成しない。複数familyを一つの巨大State ownerへ統合せず、同じOwner文書内に置く場合もlogical Owner、State、Command、failure、Save projectionを分離する。

既存Combat、Interaction、Scenario／Stage、Pickup／Grant、UI、Localization、Save／Replayは、Public ContractとRPG要件が意味一致する場合だけ再利用する。Shooter固有Weapon、Score、Encounter、Game Flow、Perception bindingをRPGのbattle、progression、inventory、quest、economyへ暗黙変換しない。

## 4. Genre composition

RPG compositionはFeature dependency closure、選択Profile、Game Flow、command role、Scenario／World／UI binding、Reference fixture bindingを一つのPack revisionへ閉じる。Featureのoptional／conditional dependencyは選択Recipeに記録し、Manifest全体のunconditional dependencyへ昇格させない。

compact command RPGのtarget compositionは、少なくとも次の関係を表現できなければならない。

- field／town／dungeon traversalはWorld、Navigation、Interaction、Scenario／StageのPublic Contractを使う。
- battle開始／終了はCommand Battleのinstance lifecycleとScenario／Worldのtyped transitionを接続し、Engine全体のSimulation Cadenceをturn-basedへ変更しない。
- progression、equipment、inventory、currency、questはそれぞれのFeature State ownerだけが変更する。
- dialogue、shop、battle、inventory UIはtyped requestとsealed Snapshotを使い、authoritative Feature Stateを書かない。
- Save／Loadは各FeatureのOwner-typed projectionとGenre flow projectionをPersistenceへ束縛し、UI label、localized text、array indexをidentityにしない。

Genre Recipe、Profile、Game Flow、Action roleの具体record候補は、Parent Ownerの意味を再定義しない補助Catalogへ分離できる。本書から未承認Schema、fixed balance、content件数、Operation inventoryまたはTarget budgetをcurrentとして作らない。

## 5. RPG Profile

RPG Profileは必要Feature closure、World／Camera／Input／UI／Audio／Animationのbinding intent、command presentation mode、Save／Replay requirement、Accessibility／Localization requirement、Target compatibilityを組み合わせる。Profileは各Subsystemのparameter schema、fixed value、execution semanticsを所有しない。

最初のProduct Reference向けProfileは2D command presentationを要求できるが、2Dをdepth-zero 3Dとして扱わず、Genre PackがSprite、Tile、Material、Camera、Text、Audio、Navigationのruntime authorityを所有しない。3D、action hybrid、real-time battle等は、必要FeatureとQualificationが別に閉じるまで同Profileのimplicit modeにしない。

Profile名、表示label、Reference Game titleからFeature依存を推測しない。Profileが宣言したexact Feature／Owner／version／hashと、Projectが選択したPack closureだけを使う。

## 6. Game FlowとRuntime Entry境界

RPG Genreは、RPG固有のflow vocabularyとFeature／Scenario outcomeから次のGenre stateへ遷移する条件を所有できる。Runtime Entry definitionはProject State、top-level branch transitionはScheduling、Package stagingはRuntime Package、World spatial transitionはWorld、finite Stage progressionはScenario／Stage、Screen navigationはUIが所有する。

Title、Settings、Loading、HUD、Pause、ResultのscreenまたはRuntime Entry詳細をRPG private state machineへ複写しない。Town、Field、Dungeon、Boss等のReference destination名をGenreの永久closed enumにせず、Reference GameがWorld／Scenario compositionとして所有する。Genre flowはtyped outcomeとdestination roleを参照し、display name、path、Scene名から遷移先を推測しない。

## 7. Command roleとInput／UI境界

RPG command roleは、menu navigation、confirm、cancel、interact、pause、battle command selection、target selection、context action等のGenre vocabularyを表す。Input OwnerがProject適用時のAction identity、device binding、remap、gesture、haptics、replay inputを所有する。Genreはkey code、controller button、touch coordinate、dead zoneを固定しない。

UIはcommand roleとFeature Snapshotからpresentationを構築し、typed requestを発行する。UI focus、Screen Stack、Text、Localization、AccessibilityはUI Ownerが所有し、RPGが別UI runtimeを作らない。locale変更はStable ID、rule、balance、Save identity、command resultを変更しない。

## 8. Cross-feature transaction

Featureをまたぐ操作は、各Feature Ownerのvalidationとprepared resultを一つのbounded transactionへ束縛し、全precondition成功後に一回だけpublishする。Genreは各Featureのprivate Stateを書かず、Feature間のPort、causality、commit／reject条件だけをcompositionする。

最低限、次のatomicityを文書上閉じる。

- command受理、cost消費、target結果、battle turn advanceをpartial publicationしない。
- equipment変更とderived stat再計算を異なるactive generationへ分けない。
- inventory減算とcurrency／shop ledger更新を片側だけcommitしない。
- dialogue choiceとquest transitionのSave identity／causalityを分離しない。
- battle内／field内item useの許可Ownerとreject結果をcontext labelから推測しない。
- capacity、funds、prerequisite、stale Snapshot、duplicate requestのreject時に全Feature Stateを不変にする。

transaction failureをUI animation、Audio cue、VFX、loading表示から成功と推測しない。Presentationはcommitted Event／Snapshotだけを消費する。

## 9. State、Save、Replay、migration

Genre-owned stateはFeature compositionとRPG flowに限定し、battle／progression／inventory／quest／currencyのauthoritative stateを複写しない。Save BundleはGenre flow projectionと各Feature Owner projectionを同じProject、Pack closure、Runtime Session、Contract set、last committed Simulation boundaryへ束縛する。

Replayはaccepted Command、Feature Event、Genre flow transition、causality、RNG bindingをOwner別projectionとして保持する。Presentation timing、localized text、UI layout、asset residencyをGameplay replay identityにしない。deterministic battleのReplay成功をWorld transition、Save migration、Target Qualificationの代用にしない。

Shooter sourceからRPG destinationへのProduct Definition Migrationは、Product、Pack、Feature、Genre、Reference Game、Capability、Evidenceのsource／destinationを一つのatomic changeとして扱う。RPG側の一部Ownerだけをcurrent化し、ShooterとRPGをdual current Product Definitionにしない。ECS移管とは別Closureであり、ECS migration successだけからRPG Product Definitionをactivateしない。

## 10. AI Authoring境界

AIはGame Briefから必要Feature、RPG Profile、Game Flow、command role、Reference requirementを説明し、missing dependency、ambiguous intent、unsupported capabilityを質問できる。AIはFeature private State、Runtime memory、Registry current head、Capability activationを直接変更せず、Pack／Projectのactive typed Operationを通す。

bounded semantic projectionは、selected Recipe、Feature dependency closure、Owner mapping、Profile、Genre flow、Action role、Reference binding、unresolved ref、Diagnostic、Evidence gapを返す。localized label、Editor panel、screen coordinate、Shooter類似名からRPG identityやFeatureを補完しない。

## 11. Failureとlast-valid

| Failure | 正規結果 |
|---|---|
| missing／stale Feature／Pack ref | Recipe apply、CookまたはPackageをrejectし、last-valid Project／Pack closureを維持 |
| Genre間dependency、Feature cycle、version／hash conflict | resolverでrejectし、synthetic edgeを作らない |
| invalid command／capacity／funds／prerequisite | typed normal rejectionとし、全関連Feature Stateを不変にする |
| cross-feature commit failure | prepared resultを公開せず、partial inventory／currency／quest／battle stateを残さない |
| Save／Replay identity差 | approved migrationまたはtyped incompatibility。類似recordをloadしない |
| missing locale／asset／provenance | Gameplay identityを変えず、affected presentation／packageをfail closed |
| missing Qualification／Target Evidence | RPG Capabilityを`not_activated`に保ち、Shooter／Core Evidenceで代用しない |

## 12. Qualification

Qualificationは少なくとも次を独立Evidence classとして扱う。

- 各Reusable RPG Featureのcontract、failure、Save／Replay、manual／AI authoring。
- RPG Genre composition、Profile、Game Flow、command role、conditional dependency。
- Reference GameのTitleからEndingまでのacceptance、original content、`en-US`／`ja-JP` semantic invariance。
- Genre-neutral Core holdout。RPG／Shooter／Scenario PackなしのWorldless UI／logic、2D／3D Project、Save、Cook、Package、Diagnostics。
- Shooter technical fixture。高頻度Input／Collision／Physics／ECS／Rendering stressをRPG acceptanceと別に保持する。
- Windows／Mobile等のTarget別clean package、install、offline launch、lifecycle、device session。

Feature ReceiptをGenre Receipt、Genre ReceiptをReference acceptance、Reference acceptanceをGeneric Core Release、Shooter ReceiptをRPG Evidenceへ流用しない。Fixture件数、balance値、performance thresholdはProduct／Performance／Platformのfresh Evidenceなしに本書へ固定しない。

## 13. Target design closure

RPG Product Definitionのtarget design closureは次を要求する。

1. Product §5.0の全RPG requirementがGeneric、Reusable Feature、RPG Genre、Reference Gameの一つへ解決する。
2. five Feature familyのState、Command、transaction、Save／Replay、failure Ownerが一意である。
3. 本書がGenre compositionだけを所有し、Feature Schema、Core、Reference contentへ侵入しない。
4. Product outcome、RPG Genre、Reference Gameの責務とEvidence非代替が一致する。
5. Shooter source／RPG destination、ECS migration、Capability activationを別axisとしてatomicに追跡できる。

このclosureはPack Manifest、Schema、Registry、Operation、Fixture artifact、Receipt、current Product Definition、Capability activationまたは実装の存在を意味しない。

## 14. 明示的に採用しないもの

次を禁止する。

- command battle、progression、inventory、quest、economyをGeneric Coreへ置く。
- Feature familyごとに内容のないOwner文書を増殖させる。
- RPG Genre PackへFeature private State／Schemaを複写する。
- Reference GameをProduction Runtimeの第5層またはGenre Packのdependencyにする。
- Shooter Combat／Pickup／Score／Game FlowをRPGへ名前変更して流用する。
- UI、Save、Scenario、World、Runtime EntryをRPG private authorityへ吸収する。
- RPG-first方向、文書追加、Profile名をCapability active／qualified／implementedの証拠にする。
- Product Phase、実装Task、実装順、担当、工数、日程、ロードマップを本書から生成する。
