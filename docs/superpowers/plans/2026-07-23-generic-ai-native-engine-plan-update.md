# Generic AI-Native Engine Plan Update Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 承認済みの4層構造とAI可読契約を全current Architecture正本、Product Registry、実行計画へ同期し、Shooterおよび有限Level前提を汎用Coreから除去する。

**Architecture:** `Generic Engine Core -> Reusable Feature Packs -> optional Genre Packs -> Game Projects`だけをProduction依存階層とし、Reference Gamesは検証Projectに限定する。World／Runtime／GameplayはGenre-neutral contractを所有し、Scenario／Stage、Character Locomotion、Shooterを任意Packへ分離する。AIはMCDから生成したCatalog、bounded Context Capsule、型付きPack Operationを使用する。

**Tech Stack:** Markdown、PowerShell 7、Git、ripgrep。Engine実装、Schema compiler実装、Capability Activation、Release／Shippingは行わない。

## Global Constraints

- Authorityと型は`docs/superpowers/specs/2026-07-23-generic-ai-native-engine-architecture-design.md`を一字違わず使用する。
- Product構造は`Generic Engine Core`、`Reusable Feature Packs`、`Genre Packs`、`Game Projects`の4層とし、Reference GamesをProduction Runtime層にしない。
- CoreはPack／Project／Fixtureへ、Feature PackはGenre／Project／Fixtureへ、Genre Packは別Genre／Project／Fixtureへ依存しない。
- Game ProjectはGenre Packを使わずFeature Packを直接利用できる。
- `capability.gameplay.shooter_core`、`domain.action_2d`、`domain.tps_single_player`、`DomainPackManifest`をcurrent正本から廃止する。
- `GameIntentDraftV1`、`GameBriefV1`、`GameSpecDocumentV1`はPlayer、Character、Combat、Objective、Completion、World、Genre Packを必須にしない。
- C1／C2で適格化するSimulation cadenceは`fixed 60/1`だけだが、Core schemaは`SimulationCadenceProfileV1`を使用し、60をABIへ固定しない。
- 現在のoffline single-player、Windows-first、Mobile、60 Hz等のProduct Scopeは対応済み表示を拡大せず、恒久Core制約からだけ分離する。
- active Architecture文書の状態は`review`を維持する。
- Historical review／superseded planの本文を現行仕様へ書き換えない。必要な場合はcurrent successorへの明示linkだけを追加する。
- ファイル編集には`apply_patch`を使用する。
- 各Taskは変更範囲の`git diff --check`、旧identity scan、local Markdown link検査を通してからcommitする。

---

### Task 1: Pack四層構造とShooter分離を正本化する

**Files:**
- Move: `docs/architecture/08-domain-packs/domain-pack-contract.md` → `docs/architecture/08-packs/pack-contract.md`
- Move: `docs/architecture/08-domain-packs/shooter.md` → `docs/architecture/08-packs/shooter.md`
- Create: `docs/architecture/08-packs/gameplay-features.md`
- Create: `docs/architecture/08-packs/scenario-stage.md`
- Modify: `docs/architecture/06-rendering/world.md`
- Modify: `docs/architecture/README.md`
- Modify: `docs/architecture/00-product/product-plan.md`
- Modify: `docs/superpowers/specs/2026-07-23-generic-ai-native-engine-architecture-design.md`
- Modify: current files under `docs/plans/` that link to the moved Pack documents

**Interfaces:**
- Consumes: Design §§1～3、4.3、9～10。
- Produces: `PackManifestV1`、Pack dependency rules、Scenario／Stage owner、Shooter Genre Pack、Feature Capability mapping。

- [ ] **Step 1: Record the pre-change Pack references**

Run:

```powershell
rg -n '08-domain-packs|DomainPackManifest|capability\.gameplay\.shooter_core|domain\.action_2d|domain\.tps_single_player' docs/architecture docs/plans --glob '*.md'
```

Expected: current正本とcurrent planに旧path／旧identityが存在する。

- [ ] **Step 2: Move and rewrite the Pack contract**

`pack-contract.md`へDesign §2～3の依存規則、`PackManifestV1`全Field、`CompositionRecipeV1.required_feature_pack_refs[]`、Recipe closure／hash／validation failure／last-valid規則、Feature／Genreの責務、Profile ownership、install／update／removal時のdependency／migration／last-valid規則を記載する。Feature間依存だけをDAGとして許可し、Genre間依存を拒否する。未選択Recipeの条件依存をinstalled closureへ追加せず、旧`DomainPackManifest` aliasを残さない。

- [ ] **Step 3: Convert Shooter into a Genre Pack**

`shooter.md`の正本範囲をShooter固有composition、Profile scale、Game Flow、Action role、Shooter fixtureへ限定する。Damage／Vital／Faction、Ranged Combat、Encounter、Scoring、Pickup、Interaction、Character Locomotion、Path FollowingをPack共通Feature参照へ変換し、Scenario／Stageは有限Recipeだけの`CompositionRecipeV1.required_feature_pack_refs[]`へ置く。endless Recipeでは同依存を省略する。Feature pool／capacity failureを再定義せず、旧Shooter診断15件とcrowded fixture 2件をexact 1:1 migration tableへ記録する。`genre.shooter.top_down_2d`と`genre.shooter.third_person_3d`を正規identityにする。

- [ ] **Step 4: Add the optional Scenario／Stage Feature owner**

`scenario-stage.md`へDesign §4.3の`StageDefinitionV1`、`completion_mode` tagged rule、Stage Scope、transition、Save／Replay、AI Operation、fixture、`completion_mode=none` negative／positive caseを記載する。World／RuntimeへLevel契約を再定義せず、依存方向をScenario／StageからWorldへの一方向にする。WorldからScenario／Stageの文書名、Pack identity、linkを参照しない。Worldにはstreaming stale／cancel／retry／partial activation、procedural nondeterminism／invalid output、presentation authority、derived Cell writeのDiagnosticと汎用fixture／acceptanceを保持する。

`world.md`では`MapIntentResolutionV1`の1～6候補、score順、tagged disposition、Evidence／affected Stable ID／Question boundを閉じる。`WorldAuthoringPlanV1`、`WorldAuthoringBundleV1`、`WorldAuthoringContextV1`はWorld／Scene／Space／Topology／owner-typed refへ限定して全Fieldとboundを列挙し、Plan／Bundleをproposal-only、Contextをread-only／Disposableとする。ContextはSource hash、omitted range、hash-bound continuationを持ち、Source／Commit／Replayへ保存しない。Bundleはbounded typed refとChangeSet hashだけを束ね、本文、Commit権限、Receiptを複写しない。`LoadingProgressPlanV1`／`LoadingProgressSnapshotV1`はinitial activation／spatial transition／Save resume共通のclosed subject、work unit、phase、generation、stale／Cancel／Retry／partial規則を持つ。procedural determinism fixtureはNavigation Artifact削除後の同一再生成hashとlast-valid維持まで検証する。

- [ ] **Step 5: Update navigation and Product references**

Architecture Indexを`Packs`へ変更し、新3文書を一度ずつ登録する。Productとcurrent plansのlinkを`08-packs`へ変更し、ShooterをFeature Packの例として扱う記述を0件にする。

- [ ] **Step 6: Verify and commit Task 1**

Run:

```powershell
rg -n '08-domain-packs|DomainPackManifest|capability\.gameplay\.shooter_core|domain\.action_2d|domain\.tps_single_player' docs/architecture docs/plans --glob '*.md'
rg -n 'scenario-stage|pack-scenario|Scenario/Stage|Scenario／Stage' docs/architecture/06-rendering/world.md
rg -n 'MapIntentResolutionV1|WorldAuthoringPlanV1|WorldAuthoringBundleV1|WorldAuthoringContextV1|LoadingProgressPlanV1|LoadingProgressSnapshotV1' docs/architecture/06-rendering/world.md
rg -n 'Navigation Artifact.*削除.*再生成.*canonical output.*Artifact hash.*last-valid' docs/architecture/06-rendering/world.md
rg -n 'ShooterProjectileCapacityExceeded' docs/architecture/08-packs/shooter.md
git diff --check
```

Expected: historical引用を除くcurrent Architecture／current planの旧path／旧identityが0件、WorldからScenario／Stage／Pack参照0件、WorldのMap／Authoring／Loading六Schemaと全bound／tagged invariantが検出され、Navigation Artifact再生成fixtureが1件、Shooter Feature capacity failure再定義0件、旧15 Diagnosticと旧crowded fixture 2件がmigration tableで各1件、`git diff --check` exit 0。

Commit:

```powershell
git add docs/architecture docs/plans
git commit -m "docs: establish generic pack architecture"
```

### Task 2: World、Runtime Entry、Scope、Locomotionを汎用化する

**Files:**
- Modify: `docs/architecture/02-foundation/executable-contracts.md`
- Modify: `docs/architecture/03-authoring/project-state.md`
- Modify: `docs/architecture/03-authoring/gameplay-programming-model.md`
- Modify: `docs/architecture/04-runtime/scheduling-lifetime.md`
- Modify: `docs/architecture/04-runtime/performance-capacity.md`
- Modify: `docs/architecture/05-simulation/physics.md`
- Modify: `docs/architecture/05-simulation/navigation.md`
- Modify: `docs/architecture/05-simulation/animation.md`
- Modify: `docs/architecture/06-rendering/world.md`
- Modify: `docs/architecture/08-packs/gameplay-features.md`
- Modify: `docs/architecture/08-packs/scenario-stage.md`
- Modify: `docs/architecture/08-packs/shooter.md`
- Modify: `docs/superpowers/specs/2026-07-23-generic-ai-native-engine-architecture-design.md`
- Modify: `docs/superpowers/plans/2026-07-23-generic-ai-native-engine-plan-update.md`
- Local verification report（gitignored、commit対象外）: `.superpowers/sdd/generic-task-2-report.md`

**Interfaces:**
- Consumes: `PackManifestV1`とScenario／Stage owner。
- Produces: `RuntimeEntryPointV1`、generic `WorldDocumentV1`、`SpatialTopologyDefinitionV1`、`RuntimeScopeTypeCatalogV1`、`MotionExecutorPortV1`、`PhysicsIntentRoleRegistryV1`／Selection、`ProjectScaleEnvelopeV2`、`WorkloadDomainTypeRegistryV1`、`RuntimeScaleIntentDimensionRegistryV1`。

- [ ] **Step 1: Add Project runtime entry points**

`ProjectManifest`の`root scene`をexact `DocumentRef<RuntimeEntryPointDocumentV1>`の`runtime_entry_point_refs[1..64]`へ置換し、Design §4.1の`RuntimeEntryPointV1`全Field、Project-owned selector／policy Document、共通header、branch validationを追加する。三Documentは`DocumentRef.stable_id == header.document_id == payload ID`を必須にする。selected entry hashはentry payload semantic hash、Document hashはref content hashへ分離する。selectorはhash Fieldを持たない`RuntimeTargetSelectorHashPayloadV1 {selector_id, selector_version, target_profile_ref_count, target_profile_refs[]}`をID／version／hash順へcanonicalizeし、domain separator＋byte length＋payload bytesからhashする。Compile Manifestはowner-typed providerを選んだ時だけ`selected_provider_binding_set_hash`を持ち、post-commit Project revision／document set、Registry、Binding Document content／semantic hashをclosureへ入れる。Runtime Package自己hashをinputへ含めない。

entry／selector／policyのcreate／updateとroot migrationの七Operation、Scope migration一Operationを完全`McdOperationContractV1`として登録する。八recordはMCD共通Envelope、exact input／output／rate-limit／receipt ref、authority、risk、side effect、idempotency、transaction、16 pure policy pre／post refs、closed DiagnosticCodeRef set、timeoutを全行で明示し、カテゴリ散文や別行補完を使わない。Project ownerは七exact Operation refだけを参照する。`request_hash`はExecutable Contractsが所有するV2式だけを参照し、Domain文書へ別式を複写しない。root migration Receiptは四Document tagged before／afterへ固定する。Operation meta-schema／Registry compile、Project↔MCD set equality、predicate wrong-kind／missing／stale fixtureを追加する。

- [ ] **Step 2: Neutralize the Core World model**

`WorldDocumentV1`をScene 0件可、spatial topology 0～1件へ変更し、Core Worldをspace node／transition edge／optional activation entryとtyped transfer subjectだけへ限定する。closed Player interest enumをregistered typed interest-source contract refへ置換する。Task 1のstreaming stale／Cancel／Retry／partial rollback、procedural、presentation authority、derived Cell、authoring／loading schemaとfixtureを維持する。

`WorldAuthoringPlanV1.affected_world_refs`は既存編集1～64件、新規World作成kind厳密1件のbranchだけ0件を許可する。`WorldAuthoringContextV1`はCommit後だけexact World ref付きで生成する。`StageDefinitionV1.world_ref`をrequired nullableにし、world branchだけanchor／spatial spawnを許可する。WorldなしDialogue／Visual Novel／UI workflowとheadless StageのSave／Replay／transition fixtureを追加する。

`ProceduralWorldDefinitionV1`／`WorldAuthoringPlanV1`／precommit `GeneratedWorldSemanticCandidateV1`へexact Binding Document ref/content hashと別`resolved_binding_closure_hash`、output schemaを伝播する。fresh 3-runはGateway／Broker call 0件でlocal-ID正規化graph bytes、order、`semantic_graph_hash`、candidate Artifact semantic hashだけを比較する。合格後にGatewayを一度だけ呼び、local ID全件と`delta_id`を一つのallocation Manifest／Receiptで発行し、Source／validation／Preview／Cook／Commitが同じmappingを共有する。二回目allocation、local ID残存、各hash単独tamperを拒否する。truth table五fixture、last-valid維持、明示選択generic hard closure、World-owned spatial destinationを維持する。

Stage transitionはDestination V2六kind、Policy V1、Request V2、Port V2へ固定する。RequestはPolicy ref／hashだけを持ちdestinationを複製しない。ui／headless／world-spaceはexact Runtime Entry ref／payload hashを通し、world-spaceだけWorld-owned spatial destination MCD refを追加する。Policy／Request／Destination／Port inventory、六kind round-trip、branch外Field、missing anchor、stale World／Space／Edge／hash fixtureを閉じる。

- [ ] **Step 3: Generalize runtime preparation**

`PlayPreparing`は選択Runtime Entry、activation policy hash、Runtime package、System Graph、Target Plan、全startup systemを含むbranch固有closureだけを検証する。World／UI session／startup systemsとPresentationをbranch別optionalにし、first activation groupをbranch activation setへ一般化する。`Starting`はbranch set、surface unavailableはoptional Presentation child stateとし、Application stateやsimulation停止条件にしない。strict headless dependency 0、actual branch DAG reverse teardown、Project owner fixture二系統をintegration manifest mappingで束ねる。

- [ ] **Step 4: Replace the closed gameplay scope enum**

current `GameSystemSpecV2`へversion／hash付き`RuntimeScopeTypeRefV1`を導入し、Scope 7 Fieldをtyped ref化する。さらに一般schemaを`semantic_role_refs`、requirement refs、`dependency_edge_refs`、`implementation_policy_ref`、authoritativeだけ必須のSave Replay、budget／fixture／invariant refs、`auxiliary_ref_set_hash`へ一意化し、非MCD refを各`{id,version,content_hash}`へ閉じる。arrayはID／version／hash strict sorted set、duplicate／same-ID different-hashを拒否し、nonauthoritative Save refはcanonical omissionする。`GameSystemSpecV1`はoffline sourceだけで、`operation.runtime_scope.migrate_game_system` inputがsource schema version 1／destination version 2をexact指定する。Coreはgeneric contribution registryだけ、Feature／Genre mapping／adapter／fixtureは各Pack ownerが登録する。

Common Interaction汎用化とRecipe gateを維持する。Shooter target provider payload ownerはtagged kind＋stable project IDだけとし、expected base revision／document setはOperation input、post-commit値はReceipt／Registry／Compile closureだけへ置く。Binding semantic hashからrevisionを除外し、Document containment／Registry membershipで所有証明する。N create→N+1 reload／select／compile、unrelated N+2、cross-project／self-assert spoof、World／Physics／PerceptionなしProduction fixtureを追加する。

- [ ] **Step 5: Add a generic locomotion port**

Navigationを7 Field `MotionExecutorPortV1`、Provider-neutral Catalog／record、`MotionExecutorProviderRecordRefV1`、exact transport batchの唯一Ownerにする。RecordRefはCatalog ref/version/hashとprovider ID/version/content hashをbindし、Batch／Selection State／Save／Replayで一意に使う。Physics productionも同refを使い、stale Catalog／provider hash、fixture-in-productionを拒否する。

`GameSystemSpecV2`へ`emitted_port_message_type_refs`を持たせ、Character bindingのMCD Envelope、2 Feature type、role／3 requirement／dependency／policy／Save Replay／budget／fixture／3 invariantをFeature ownerでexact解決する。Characterはproposal／adapter contributionだけを所有し、Navigation／Core-owned `game_system.engine.navigation.motion_intent_batch_publisher`が全owner contributionをcanonical mergeしてBatchを一度だけ発行する。Coreはgeneric motion contribution／selected executor Portを所有し、Character adapter／Provider binding／fixtureをPackへ置く。BatchをEventにしない。Scenario Stageは`StageTransitionCrossOwnerReachableClosureV1`がStage／World／Runtime rootからowner別Validation Receipt付きでmaterializeしたdestination／request／policy／World spatial destination／port messageのexact 5件へ閉じ、Runtime delivery contractを六件目として混ぜない。public／runtime／global MCD／Port closure／aggregateを別々のlike-for-like gateで検証する。MCD IDはcanonical `<kind>.<lower-token-path>`、version separateへ統一する。

Physics AI Coreはbehavior-neutralな`PhysicsIntentRoleRegistryV1`の六つの完全なrole recordを所有し、Project Sourceの`PhysicsIntentRoleSelectionDocumentV1`が選択を固定する。object／Genre role、default mapping、adapter／fixtureはPack／Project ownerへ移す。role Refとmotion／collision／hit／shape／speed axisを独立検証し、旧object enumは完全なmigration contribution Registry、MCD Operation、Validator、Prepared payload、外側Receipt、Manifestを介してexact一件へ一意解決できる時だけ移行する。Performanceのcurrent正本は`ProjectScaleEnvelopeV2`であり、`WorkloadDomainTypeRegistryV1`とowner-typed `WorkloadDomainIntentV1`を束ねる。World／spatialは`required`なら必須、全domainが`forbidden`なら禁止、`optional`だけならnull／exact World intentの双方を許し、UI-only、strict headless、tooling、resource-onlyを偽World／Gameplay floorなしで有効とする。旧V1はoffline migration inputだけに残し、Project EnvelopeとIntegrated fixtureはowner-typed全dimension、measurement schema、unit、fidelity／equivalence／functional contract／resource SLO、fixture closureを同時に検査する。

- [ ] **Step 6: Verify and commit Task 2**

Run:

```powershell
rg -n 'topology_definition_ref.*厳密に1件|level_definition_refs.*1～|level_gameplay|player_or_party_transfer_refs|play_session.*world_instance.*level_instance.*encounter_instance|movement_profile_ref.*CharacterMotorProfileV1|World／Level／Topology' docs/architecture --glob '*.md'
rg -n 'RuntimeEntryPointV1|default_for_selected_targets|selected_runtime_entry_point_ref|entry_branch_closure_hash|scope\.core\.(application|runtime_session|world|entity|ui_session)|RuntimeScopeTypeCatalogV1|scope\.feature\.|scope\.genre\.|policy_denied|fixture\.genre\.shooter\.target-practice-minimal' docs/architecture --glob '*.md'
rg -n 'MotionExecutorPortV1|executor_capability_ref|movement_profile_schema_ref|accepted_intent_schema_refs|resolved_motion_schema_ref|compatibility_predicate_ref|failure_diagnostic_refs' docs/architecture docs/superpowers/specs/2026-07-23-generic-ai-native-engine-architecture-design.md --glob '*.md'
rg -n 'RuntimeTargetSelectorV1|RuntimeEntryActivationPolicyV1|activation_policy_hash|entries\\[5\\.\\.4096\\]|StageTransitionDestinationV2|type\\.feature\\.character_locomotion\\.gameplay_motion_intent|type\\.animation\\.root_motion_proposal|target-practice-minimal-project-provider|procedural-provider-selected-valid' docs/architecture docs/superpowers --glob '*.md'
rg -n 'mirakan\\.(schema|contract|port|state|event)\\.|[A-Z][A-Za-z0-9]*@1|MotionExecutorPortV1.*6 Field|selected executorへ直接提出' docs/architecture docs/superpowers/specs/2026-07-23-generic-ai-native-engine-architecture-design.md --glob '*.md'
rg -n 'CharacterLocomotionBindingStateV1|accepted_intent_schema_refs:.*CharacterMoveIntentV1|mirakan\.contract\.animation\.root_motion_proposal(?!@1)|両Profile' docs/architecture docs/superpowers --glob '*.md' --pcre2
rg -n 'scenario-stage|pack-scenario|Scenario/Stage|Scenario／Stage|feature\.' docs/architecture/06-rendering/world.md
git diff --check
```

加えて八Operation meta-schema／Registry compile、Project↔MCD set equality、Motion RecordRef、GameSystem meta-schema／12 record、World closure／semantic hash、Selector hash唯一性、Stage owner／aggregate set、Shooter fixed-point不在、MCD grammarをassertする。全Markdown path＋fragment link、fence、`git diff --check`を検査する。

Commit:

```powershell
git add docs/architecture docs/superpowers
git commit -m "docs: eliminate remaining runtime contract ambiguity"
```

### Task 3: Clock、Pause、Input／Camera／Audio／LODのGenre漏出を除去する

**Files:**
- Modify: `docs/architecture/04-runtime/scheduling-lifetime.md`
- Modify: `docs/architecture/06-rendering/vfx-runtime.md`
- Modify: `docs/architecture/06-rendering/camera.md`
- Modify: `docs/architecture/06-rendering/lod.md`
- Modify: `docs/architecture/07-platform/input.md`
- Modify: `docs/architecture/07-platform/audio.md`
- Modify: `docs/architecture/03-authoring/native-game-module.md`
- Modify: `docs/architecture/08-packs/shooter.md`
- Modify: `docs/architecture/00-product/product-plan.md`

**Interfaces:**
- Consumes: Core Scope Catalog、Shooter Genre Pack。
- Produces: `SimulationCadenceProfileV1`、optional Pause、generic semantic action／view impulse／critical cue vocabulary。

- [ ] **Step 1: Introduce the cadence profile**

Design §6の`SimulationCadenceProfileV1`をRuntime正本へ追加する。60 Hz専用式を`rate_numerator_hz/rate_denominator`の有理数式へ一般化し、C1／C2 active値を`60/1`、catch-up 4とする。alternate rate／explicit stepは未適格化時に`capability_unavailable`とする。Replay headerとProfile hashの一致を必須にする。Native ABI freeze前に`MirakanNativeInvokeContextV1.fixed_delta_numerator／denominator`をfixed／explicit-step tagged cadence inputへ移行し、descriptor、Source binding、Replay、fixtureを同時更新する。単数`fixed delta`を恒久ABIにしない。

- [ ] **Step 2: Remove independent 60 literals**

VFX lifetime／delta、Input latency fixture、Timer／Replay記述をCadence Profile参照へ変更する。60/1 fixture値は残してよいが、Subsystem固有contractとして60を再定義しない。

- [ ] **Step 3: Make Pause optional and genre-neutral**

`PausePolicyV1.support_mode=unsupported | global`を追加し、policy refをoptionalにする。Pause ownerを`scope.core.runtime_session`へ変更し、unsupported commandの`pause_not_supported` Diagnosticを追加する。Weapon、Encounter、Shooter Game Flow、常時保存されるGame Flow stateを共通Runtime記述から削除する。

- [ ] **Step 4: Move Genre vocabulary to Shooter**

Input CoreからShooter Semantic Action Templateを削除し、owner付きsemantic action→typed Command binding registryだけを残す。role、required／optional Action Profile、evaluator、fixtureは各Pack ownerが登録する。`PlayerLookInput`を`ViewControlInput`、`recoil_channels[]`をgeneric `view_impulse_channels[]`へ変更する。AudioのShooter supersonic記述、LODのWeapon／Combat presetをgeneric bound／critical-authoritative cueへ置換し、Shooter具体値はShooter Packへ移す。

- [ ] **Step 5: Register cadence Future inception**

Product Future Portfolioへalternate fixed cadence、explicit-step、physics substepのowner、data model、determinism、Replay、Target qualificationを持つplanning-only entryを追加する。現在Capability／Phaseへ暗黙activateしない。

- [ ] **Step 6: Verify and commit Task 3**

Run:

```powershell
rg -n 'fixed_tick_hz: 60|exactly 60 Hz|Shooter Semantic Action Template|PlayerLookInput|recoil_channels|Game Flow state|Weapon cadence|supersonic.*Shooter' docs/architecture --glob '*.md'
git diff --check
```

Expected: 旧Core literal／Shooter語彙が0件。Shooter Packの意図したGenre用語とC1の`60/1` qualificationだけが残る。`git diff --check` exit 0。

Commit:

```powershell
git add docs/architecture
git commit -m "docs: generalize cadence pause and platform semantics"
```

### Task 4: AI上位Schema、Catalog、Pack Discovery、反例Evalを閉じる

**Files:**
- Modify: `docs/architecture/01-governance/ai-security-approval.md`
- Modify: `docs/architecture/01-governance/ai-verification-provenance.md`
- Modify: `docs/architecture/02-foundation/executable-contracts.md`
- Modify: `docs/architecture/03-authoring/project-state.md`
- Modify: `docs/architecture/08-packs/pack-contract.md`
- Modify: `docs/architecture/00-product/product-plan.md`

**Interfaces:**
- Consumes: `PackManifestV1`、Runtime Entry、Scope、Cadence。
- Produces: `GameIntentDraftV1`、`GameBriefV1`、`GameSpecDocumentV1`、`AiCatalogEntryV1`、`AiTaskContextCapsuleV1`、`operation.packs.*`、holdout Eval。

- [ ] **Step 1: Register field-level game understanding inputs**

Project StateへDesign §7の3 Schemaを完全なfield cardinalityとtagged validation付きで追加する。AI SecurityはSchemaを参照し、GameIntent→Brief→Spec→Understanding Closureのstate transition、質問上限、unsupported dispositionを所有する。Player／Character／World／Combat／Objective／Completion／Genre Packを必須にしない。

- [ ] **Step 2: Add the generated AI Catalog**

Executable ContractsへDesign §8の`AiCatalogEntryV1`全FieldとContract compiler projectionを追加する。owner、layer、purpose／non-goal、I/O、State、Scope、phase、read/write、dependency、Target、maturity、budget、Operation、Risk、Diagnostic、example／counterexample、migration、Project SDKを欠落不可にする。

- [ ] **Step 3: Add Pack Discovery operations**

8個の`operation.packs.*`をRisk、input identity、side effect、Receipt／result、failure Diagnostic付きで登録する。Search／read／explainはread-only、plan／preview／validate／removeはProject ChangeSetを直接CommitせずProposal／Stagingだけを生成する。

- [ ] **Step 4: Add bounded task context**

`AiTaskContextCapsuleV1`をAuthoring Context Indexから生成するread-only／Disposable projectionとして追加する。選択理由、source hash、field mask、token estimate、omitted range、continuation、allowed operation IDsを保持し、Security署名内部や全Schemaを通常Creative Contextへ送らない。

- [ ] **Step 5: Add genre-neutral holdout fixtures**

AI VerificationとProduct Phase Gateへturn-based zero-character、endless continuous simulation、UI-only／tool-like、headless simulationを登録する。Phase 4にはCombatなしでzero-characterまたはWorldなしの中立Fixtureを最低1件含める。未知ID、`question_required`回避、Genre Pack強制、偽Level／偽Completion生成をfailure metricにする。

- [ ] **Step 6: Verify and commit Task 4**

Run:

```powershell
rg -n 'GameIntentDraftV1|GameBriefV1|GameSpecDocumentV1|AiCatalogEntryV1|AiTaskContextCapsuleV1|operation\.packs\.(search|read|resolve_dependencies|explain_composition|plan_apply|preview_apply|validate|plan_remove)' docs/architecture --glob '*.md'
git diff --check
```

Expected: 5 Schema名と8 Operationが各canonical ownerに存在し、`git diff --check` exit 0。

Commit:

```powershell
git add docs/architecture
git commit -m "docs: close generic AI understanding contracts"
```

### Task 5: Product DAG、Network、Target Release、current plansを同期する

**Files:**
- Modify: `docs/architecture/00-product/product-plan.md`
- Modify: `docs/plans/2026-07-22-architecture-evolution-control-plane-design.md`
- Modify: `docs/plans/2026-07-22-architecture-evolution-control-plane-implementation-plan.md`
- Modify: `docs/plans/2026-07-23-critical-path-execution-plan.md`
- Modify: `docs/plans/2026-07-23-future-capability-inception-plan.md`
- Modify: `docs/reviews/2026-07-23-plan-feasibility-remediation.md`

**Interfaces:**
- Consumes: 全更新済みArchitecture正本。
- Produces: Shooter非依存DAG、Target-local Release closure、更新済みcounts／status、current plan navigation。

- [ ] **Step 1: Rebuild Product Pack and fixture dependencies**

Shooter以外のWPからShooter fixture／Capabilityへのdirect dependencyを除去する。AI、Project C++、Project Shader、Build、Mobile、general coverageはGenre-neutral fixtureまたは必要Feature Capabilityへ依存させる。Shooter Reference GameだけがShooter Genre Capabilityへ依存する。

- [ ] **Step 2: Separate generic network futures**

`small-cooperative-multiplayer`と`rollback-competitive-networking`のShooter prerequisiteをsession transport、replication、authority、deterministic input／rollback Capabilityへ置換する。`large-session-network-shooter`はShooter consumerとして残す。

- [ ] **Step 3: Split Target-local release closure**

Windows、Android、Apple、headlessのpackage／release Work PackageとRelease GateをTarget単位に閉じる。全Target coverageは別Portfolio projectionへし、一Targetの未完了で別Target Releaseを停止しない。

- [ ] **Step 4: Recompute registries and current plans**

追加・分割したCapability、Fixture、Gate、WP、Future entryのexact countをProduct Planから再集計し、Critical Path、Control Plane、Future Inceptionの固定count／ID／dependencyを更新する。Reviewは新設計反映後も`planning_go / implementation_and_shipping_no_go`であることと、Engine実装未実施を明記する。

- [ ] **Step 5: Add explicit genericity gates**

Product completionへShooter Pack removal、GenreなしProject、UI-only、headless、noncombat、zero-character、Pack dependency direction、Target-local Release、small-model Pack DiscoveryのGateを登録する。

- [ ] **Step 6: Verify and commit Task 5**

Run:

```powershell
rg -n 'future\.capability\.(small-cooperative-multiplayer|rollback-competitive-networking).*capability\.gameplay\.shooter|wp\.product\.production-release-binding' docs/architecture/00-product/product-plan.md
rg -n 'Active Product Definition|Work Package|Future capability|Phase Gate|Release Gate' docs/architecture/00-product/product-plan.md docs/plans/2026-07-23-critical-path-execution-plan.md docs/reviews/2026-07-23-plan-feasibility-remediation.md
git diff --check
```

Expected: generic Network FutureのShooter prerequisite 0、旧全Target release binding 0、current count記述がProduct正本と一致、`git diff --check` exit 0。

Commit:

```powershell
git add docs/architecture/00-product docs/plans docs/reviews/2026-07-23-plan-feasibility-remediation.md
git commit -m "docs: synchronize generic product execution plan"
```

### Task 6: 全文監査、link検査、独立Reviewを完了する

**Files:**
- Modify: findingsに対応するcurrent Architecture／Product／plan文書
- Modify: `docs/superpowers/plans/2026-07-23-generic-ai-native-engine-plan-update.md`

**Interfaces:**
- Consumes: Tasks 1～5の全commit。
- Produces: 旧identity 0、broken local link 0、依存逆流0、current count一致、Review済みdiff。

- [ ] **Step 1: Run the forbidden-assumption scan**

Run:

```powershell
rg -n '08-domain-packs|DomainPackManifest|capability\.gameplay\.shooter_core|domain\.action_2d|domain\.tps_single_player|Shooter Semantic Action Template|PlayerLookInput|recoil_channels|player_or_party_transfer_refs|game_flow_disallowed|Game Flow state|fixed_tick_hz: 60' docs/architecture docs/plans --glob '*.md'
rg -n 'CharacterLocomotionBindingStateV1|accepted_intent_schema_refs:.*CharacterMoveIntentV1|mirakan\.contract\.animation\.root_motion_proposal(?!@1)|両Profile|relative_source_path' docs/architecture docs/superpowers --glob '*.md' --pcre2
```

Expected: 旧identity／旧path／Core漏出0件。Migration表や明示的な禁止例だけが残る場合は、その文脈を個別確認する。

- [ ] **Step 2: Verify local Markdown links**

PowerShellで全current `docs/architecture/**/*.md`と`docs/plans/*.md`のrelative Markdown link targetを解決し、`http:`, `https:`, fragment-onlyを除外して`Test-Path -LiteralPath`を行う。

Expected: missing target 0。

- [ ] **Step 3: Verify dependency direction and required fixtures**

Product RegistryとPack manifestからCore→Pack、Feature→Genre、Genre→Genre、Production→Fixtureの禁止edgeを抽出する。Shooter removal、Genreなし、UI-only、headless、turn-based、endless、中立AI FixtureのIDがProduct Gateへ解決することを確認する。

Expected: forbidden edge 0、required genericity Gate missing 0。

- [ ] **Step 4: Review the complete diff**

Run:

```powershell
git diff --check
git status --short
git diff --stat HEAD~5..HEAD
```

Expected: whitespace error 0、意図したMarkdown変更だけ、未追跡scratch artifact 0。

- [ ] **Step 5: Apply independent review findings**

独立ReviewerへDesign Spec、Task plan、全branch diffを渡す。Critical／Important findingを一括修正し、同じ全文監査とlink検査を再実行する。指摘が0になるまで完了扱いにしない。

- [ ] **Step 6: Mark the plan complete and commit**

全Task checkboxを`[x]`へ更新し、実行した検証commandとexact結果を末尾のExecution Recordへ追記する。

Commit:

```powershell
git add docs
git commit -m "docs: complete generic engine plan audit"
```

## Execution Record

- Baseline: `acf967e`
- Work branch: `codex/plan-review-closure`
- Completion requires: Tasks 1～6 reviewed、full link／identity／dependency／count audit current、`git diff --check` exit 0。

### Task 2 independent audit closure（2026-07-23）

- Audit base: `0386626a95f1534389cb4f2a6ec71ff254d06b68`
- Scope: Critical 3／Important 7／Minor 2の文書契約closure。Engine実装、Capability activation、Shipping readinessは未主張。
- Closed roots: Diagnostic／Service／Policy／Operation Registry、八Operation reachable-error equality、generic Scope migration contribution、current `GameSystemSpecV2`、auxiliary set、Selector hash、Stage exact-five／five-gate、World precommit three-run／single allocation、Shooter Registry／three Operation、root Receipt union、Physics neutral Role Registry、owner-typed Scale Dimension Registry。
- Fresh contract assertions: `35/35 PASS`。
- Core→Pack forbidden exact ref、old active contract identity、Physics object／Genre hardcode、Performance旧Genre scale field: 各`0`。
- MCD common envelope `11`件、typed ref `9`件のlogical ID grammar failure: `0`。
- 全Markdown `71`件、relative local link `1,478`件、fragment `143`件、missing path／fragment `0`、unbalanced fence `0`。
- `git diff --check`: `PASS`。一時監査planはcommit対象から削除済み。

### Task 2 final contract-DAG／genericity closure（2026-07-23）

- Audit base: `ace0913249db5e1d664dfa6bc645ee5ba2613b5d`
- Scope: final re-auditのCritical／Important／Minor指摘とpost-Marker crash recovery。Engine実装、Capability activation、Shipping readinessは未主張。
- Contract set: `ContractSetLocalRefV1`→self-excluding local record hash→sorted set root→外部MCD／Service／Validator refの一方向生成へ統一し、Operation／Service／Validator closureの相互edgeをLocalRefだけにした。
- Commit: `PreparedCandidateV1` content identity、Prepared Envelope、staged postcondition、state＋Prepared payload＋Marker publish、readback、外側signed Receipt／Resultへ閉じた。Marker後crashはcanonical materialization-key payloadとdeterministic signing policyのatomic put-if-absentでexact一回回復し、別署名／二重Receipt／rollbackを禁止した。
- Request: V2式の正本をExecutable Contracts一箇所だけにし、project-bound／projectlessとも選択input schemaの実在Fieldをencodeする。V1はread-only offline migration record／candidateだけで、未登録Operation IDを作らない。
- Generic runtime: owner-typed `ProjectScaleEnvelopeV2`、optional spatial semantics、六Physics Role＋Selection／migration、Navigation-owned generic motion publisher、Input type-role＋Selection、Shooter Source Selection／Derived Registry、Stage cross-owner exact-five、Root Scene exact-four planを正本化した。
- Manifest: Pack Operation／Validator／migration／test inventoryとTrusted Service allowlist contributionをexact version／hash／LocalRef set equalityへ閉じた。
- Fresh contract assertions: `27/27 PASS`。Shooter Operation `3/3`×Diagnostic `17/17`、initial Validator `22/22`、Physics Role `6/6`、Target Selector createのdefault-ambiguous error `0`。
- Core→Pack exact hard dependency `0`、invalid `McdContractRefV1` kind `0`、旧current alias／省略error／Receipt循環の禁止pattern `0`。
- 全Markdown `71`件、relative local link `1,482`件、fragment `147`件、missing path／fragment `0`、unbalanced fence `0`。
- `git diff --check`: `PASS`。
- Remaining risk: Schema／Registry compiler、Gateway、deterministic signing Store、Runtime、Save／Replay、Build、Target実機qualificationは後続実装／conformance testが必要である。
