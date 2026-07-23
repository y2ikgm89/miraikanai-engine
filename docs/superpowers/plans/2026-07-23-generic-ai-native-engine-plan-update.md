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

- [ ] **Step 5: Update navigation and Product references**

Architecture Indexを`Packs`へ変更し、新3文書を一度ずつ登録する。Productとcurrent plansのlinkを`08-packs`へ変更し、ShooterをFeature Packの例として扱う記述を0件にする。

- [ ] **Step 6: Verify and commit Task 1**

Run:

```powershell
rg -n '08-domain-packs|DomainPackManifest|capability\.gameplay\.shooter_core|domain\.action_2d|domain\.tps_single_player' docs/architecture docs/plans --glob '*.md'
rg -n 'scenario-stage|pack-scenario|Scenario/Stage|Scenario／Stage' docs/architecture/06-rendering/world.md
rg -n 'ShooterProjectileCapacityExceeded' docs/architecture/08-packs/shooter.md
git diff --check
```

Expected: historical引用を除くcurrent Architecture／current planの旧path／旧identityが0件、WorldからScenario／Stage参照0件、Shooter Feature capacity failure再定義0件、旧15 Diagnosticと旧crowded fixture 2件がmigration tableで各1件、`git diff --check` exit 0。

Commit:

```powershell
git add docs/architecture docs/plans
git commit -m "docs: establish generic pack architecture"
```

### Task 2: World、Runtime Entry、Scope、Locomotionを汎用化する

**Files:**
- Modify: `docs/architecture/03-authoring/project-state.md`
- Modify: `docs/architecture/03-authoring/gameplay-programming-model.md`
- Modify: `docs/architecture/04-runtime/scheduling-lifetime.md`
- Modify: `docs/architecture/05-simulation/physics.md`
- Modify: `docs/architecture/05-simulation/navigation.md`
- Modify: `docs/architecture/05-simulation/animation.md`
- Modify: `docs/architecture/06-rendering/world.md`
- Modify: `docs/architecture/08-packs/scenario-stage.md`

**Interfaces:**
- Consumes: `PackManifestV1`とScenario／Stage owner。
- Produces: `RuntimeEntryPointV1`、generic `WorldDocumentV1`、`SpatialTopologyDefinitionV1`、`RuntimeScopeTypeCatalogV1`、`MotionExecutorPortV1`。

- [ ] **Step 1: Add Project runtime entry points**

`ProjectManifest`の`root scene`を`runtime_entry_point_refs[1..64]`へ置換し、Design §4.1の`RuntimeEntryPointV1`と`world | ui | headless` tagged validationを追加する。World／Scene／Topology／Stageを全Projectの必須Documentにしない。Authoring selectionのStage参照はoptional Feature projectionとする。

- [ ] **Step 2: Neutralize the Core World model**

`WorldDocumentV1`をScene 0件可、spatial topology 0～1件へ変更する。`LevelDefinitionV1`、`LevelRuntimeStateV1`、Objective、Completion、Encounter、`level_gameplay` ownerをWorld ownerから削除し、Scenario／Stage ownerへlinkする。Topologyをspace node／transition edge／optional activation entryで表し、`player_or_party_transfer_refs`をtyped `transfer_subject_refs[]`へ置換する。

- [ ] **Step 3: Generalize runtime preparation**

`PlayPreparing`は選択された`RuntimeEntryPointV1` branch、Runtime package、System Graph、Target Plan、branch固有closureだけを検証する。World／Level／Topologyを常時要求しない。UI-only、headless、world branchのpositive fixtureとbranch field混在のnegative fixtureを記載する。

- [ ] **Step 4: Replace the closed gameplay scope enum**

`GameSystemSpecV1`へ`runtime_scope_type_ref`を導入し、Design §5.1の5 Core Scopeと`RuntimeScopeTypeCatalogV1`登録規則を追加する。`play_session`を`scope.core.runtime_session`へ、Shooter score／flowとScenario Stage／EncounterをPack登録Scopeへ移す。

- [ ] **Step 5: Add a generic locomotion port**

Navigationに`MotionExecutorPortV1`を追加し、Path Followingの`movement_profile_ref`をProvider-owned profileへ変更する。Physics Character MotorをCharacter Locomotion Featureのreference Providerとして明記する。Animation root motionはCharacter Motorでなくselected Motion Executorへのproposalとし、Character Motor固有conformanceはProvider fixtureへ移す。

- [ ] **Step 6: Verify and commit Task 2**

Run:

```powershell
rg -n 'topology_definition_ref.*厳密に1件|level_definition_refs.*1～|level_gameplay|player_or_party_transfer_refs|play_session.*world_instance.*level_instance.*encounter_instance|movement_profile_ref.*CharacterMotorProfileV1|World／Level／Topology' docs/architecture --glob '*.md'
git diff --check
```

Expected: Core正本で旧必須Level／Scope／Character Motor固定が0件、Pack owner内の意図したGenre／Feature記述だけが残る。`git diff --check` exit 0。

Commit:

```powershell
git add docs/architecture
git commit -m "docs: remove stage and locomotion assumptions from core"
```

### Task 3: Clock、Pause、Input／Camera／Audio／LODのGenre漏出を除去する

**Files:**
- Modify: `docs/architecture/04-runtime/scheduling-lifetime.md`
- Modify: `docs/architecture/06-rendering/vfx-runtime.md`
- Modify: `docs/architecture/06-rendering/camera.md`
- Modify: `docs/architecture/06-rendering/lod.md`
- Modify: `docs/architecture/07-platform/input.md`
- Modify: `docs/architecture/07-platform/audio.md`
- Modify: `docs/architecture/08-packs/shooter.md`
- Modify: `docs/architecture/00-product/product-plan.md`

**Interfaces:**
- Consumes: Core Scope Catalog、Shooter Genre Pack。
- Produces: `SimulationCadenceProfileV1`、optional Pause、generic semantic action／view impulse／critical cue vocabulary。

- [ ] **Step 1: Introduce the cadence profile**

Design §6の`SimulationCadenceProfileV1`をRuntime正本へ追加する。60 Hz専用式を`rate_numerator_hz/rate_denominator`の有理数式へ一般化し、C1／C2 active値を`60/1`、catch-up 4とする。alternate rate／explicit stepは未適格化時に`capability_unavailable`とする。Replay headerとProfile hashの一致を必須にする。

- [ ] **Step 2: Remove independent 60 literals**

VFX lifetime／delta、Input latency fixture、Timer／Replay記述をCadence Profile参照へ変更する。60/1 fixture値は残してよいが、Subsystem固有contractとして60を再定義しない。

- [ ] **Step 3: Make Pause optional and genre-neutral**

`PausePolicyV1.support_mode=unsupported | global`を追加し、policy refをoptionalにする。Pause ownerを`scope.core.runtime_session`へ変更し、unsupported commandの`pause_not_supported` Diagnosticを追加する。Weapon、Encounter、Shooter Game Flow、常時保存されるGame Flow stateを共通Runtime記述から削除する。

- [ ] **Step 4: Move Genre vocabulary to Shooter**

Input CoreからShooter Semantic Action Templateを削除し、Pack-owned semantic action template extension contractだけを残す。`PlayerLookInput`を`ViewControlInput`、`recoil_channels[]`をgeneric `view_impulse_channels[]`へ変更する。AudioのShooter supersonic記述、LODのWeapon／Combat presetをgeneric bound／critical-authoritative cueへ置換し、Shooter具体値はShooter Packへ移す。

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
