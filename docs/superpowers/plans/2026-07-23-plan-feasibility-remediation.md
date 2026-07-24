# Plan Feasibility Remediation Implementation Plan

> **Status: superseded historical remediation plan; do not execute.** Current counts and execution contracts are Product Active 14／WP75、Future 3／25／Claim57, the current Control Plane plans, Critical Path, Future Inception, and the current remediation Review.

> **Historical instruction (superseded; must not be followed):** 当時のagent実行手順とcheckboxは完了記録としてのみ保持する。現行作業の指示として解釈しない。

**Goal:** 2026-07-23監査で確認した実装DAG、schema、Phase gate、AI Operation、Approval、Toolchain、将来Scopeの不整合を、公式一次資料と一つのProject正本へ閉じる。

**Architecture:** Product PlanをPhase／Work Package／Fixture bindingの正本にし、Control Planeと実装計画をそのschemaへ追随させる。AIのHost／Model互換性は直交Profileで表し、Build GatewayだけがPackage／Device／Debug jobを実行する。MVPはCX0 Headerで開始し、Modules Shippingとfirst-party local inferenceは独立Activationにする。

**Tech Stack:** Markdown、PowerShell 7、Node.js 24.18.0 LTS、npm 11.16.0、TypeScript 7.0.2 CLI、JSON Schema Draft 2020-12、Ajv 8.20.0、MCP 2025-11-25、CMake 4.4、MSVC 14.51 bootstrap。

## Global Constraints

- Authorityと型は`docs/superpowers/specs/2026-07-23-plan-feasibility-remediation-design.md`を一字違わず使用する。
- Product PlanがWork Package、Phase、Fixture、Target、Requirement、Capability、Future incubationの機械正本である。
- Work Package state Fieldは`scheduling_state`、Fixture寄与FieldはProduct Plan正本のtyped `provided_fixture_refs[]: ProvidedFixtureRefV1`とする。
- Phase completionは`ProductPhaseRegistryV1.exit_gate_refs[]`の各exact refを`PhaseFixtureBindingRegistryV1.gate_id`へ解決し、その全Gateだけから評価する。
- 外部Brand／Model名をEngineの権限分岐にしない。
- MCP Production baselineは`2025-11-25`、OpenAI direct既定explicit modelは`gpt-5.6-sol`とする。
- D3D12 MVPはCX0 self-contained Headerで実装し、`.ixx`／`.cppm`はCX1 fixtureまたはCX2 cutoverまでProduction componentへ追加しない。
- active文書のstateは`review`を維持し、Capability activationは`not_activated`を維持する。
- 未確認のVendor version、Hardware、Market coverage、人数、日付、費用を推測しない。

---

### Task 1: Product Registryと実装DAGを修正する

**Files:**
- Modify: `docs/architecture/00-product/product-plan.md`
- Modify: `docs/architecture/08-domain-packs/shooter.md`

**Interfaces:**
- Consumes: 設計§4、§5、§10。
- Produces: canonical `WorkPackageRegistryV1`、`PhaseFixtureBindingRegistryV1`、Phase 0～9 DAG、`FutureCapabilityIncubationRegistryV1`。

- [x] **Step 1: schema競合を再現する**

Run:

```powershell
rg -n 'schedule_state|exit_fixture_refs|completion_receipt_refs|fixture_refs\[\]' docs/architecture/00-product/product-plan.md docs/plans/2026-07-22-architecture-evolution-control-plane-*.md
```

Expected: ProductとControl Planeで異なるField名を検出する。

- [x] **Step 2: Product schemaをcanonical形へ変更する**

`WorkPackageRegistryV1`を設計§4.1の12 Fieldへ置換する。`ProductPhaseRegistryV1.exit_fixture_refs[]`を`exit_gate_refs[]`へ変更し、設計§4.2の`PhaseFixtureBindingRegistryV1`を追加する。

- [x] **Step 3: Phase gate表を追加する**

10 Phaseすべてについて`gate_id`、Phase、Fixture、評価Requirement、Target、固定literal `policy.product.same-candidate.v1`、適切な§8 freshness policyを列挙する。Phase 3はmanual 2D Requirementだけ、Phase 4はAI MVP Requirementだけ、Phase 6はFirst Playable 3Dだけ、Phase 8はC2 Requirementだけを評価する。

- [x] **Step 4: Phase 0～7のWork Package DAGを閉じる**

設計§5のFoundation、ECS E1～E7、Render Graph、2D／3D Simulation、Input／Audio／UI、World／Camera、Vulkan／Metal／package Work Packageを追加する。各行は実在Owner文書、Target、fallback、提供Fixture、required Capability、先行WPを持つ。Phase ordinalより後のWPへの依存を禁止する。

- [x] **Step 5: Phase 8依存を修正する**

Genre／Rendering／UI／Gameplay provider WPを先行させ、`wp.product.general-coverage-2d`と`wp.product.general-coverage-3d`を評価aggregateとして最後に依存させる。2D aggregateはShooter、Platformer、Puzzle／Dialogue三Fixtureをexit gateへ含める。

- [x] **Step 6: Future incubation registryを追加する**

設計§10の17 entryを`planning_only`で登録し、active Capability／Target／Phaseへ参照しないことを明記する。FPS、大人数network shooter、AAA photoreal renderingをShooter／network／renderingの既存Capabilityへ暗黙包含しない。

- [x] **Step 7: Product整合検査を実行する**

Run: Product表からTarget、Requirement、Fixture、Phase、WP、Capability、Gate、Future IDを抽出し、duplicate、missing ref、Phase↔WP不一致、dependency cycle、後続Phase dependencyを検査する。

Expected: 全countがnon-zero、duplicate／missing／cycle／forward dependencyが0。

### Task 2: Control Plane、bootstrap、状態、Freshnessを修正する

**Files:**
- Modify: `docs/plans/2026-07-22-architecture-evolution-control-plane-design.md`
- Modify: `docs/plans/2026-07-22-architecture-evolution-control-plane-implementation-plan.md`
- Modify: `docs/architecture/03-authoring/project-state.md`
- Modify: `docs/architecture/01-governance/ai-verification-provenance.md`

**Interfaces:**
- Consumes: Task 1のProduct schema、設計§8、§9のAjv pin。
- Produces: canonical schema consumer、`ControlPlaneBootstrapApprovalV1`、Target readiness、Technical Qualification freshness。

- [x] **Step 1: Work Package型をProduct正本へ追随させる**

Control Planeの`requirement_refs[]`、`exit_fixture_refs[]`、`schedule_state`、`completion_receipt_refs[]`を削除し、Task 1のcanonical Fieldへ置換する。completionの監査情報は`WorkPackageLifecycleRecordV1`へ移す。

- [x] **Step 2: Ajv validator bootstrapをTask 2より前へ追加する**

Ajv 8.20.0をToolchain lockとpackage lockからread-backし、`ajv/dist/2020`、`strict=true`、`allErrors=false`、`loadSchema`未設定、local `$id` allowlistでschemaをcompileするTaskを追加する。schema未存在testはValidator import成功後に実行する。

- [x] **Step 3: Control Plane bootstrap approvalを追加する**

5 Owner文書作成後、Phase 0正式開始前に`ControlPlaneBootstrapApprovalV1`を生成するTaskを追加する。承認主体、git tree、5 document hash、Product registry hash、Toolchain lock hash、Decision hash、発行／失効を必須にする。

- [x] **Step 4: Target readinessを統一する**

Project StateとPerformanceへの投影を`predicted | blocked | qualified`へ変更し、`optimization_required`を`blocked_reason_ref`へ移す。PascalCase stateとCapability側`not_activated`の混入をnegative fixtureにする。

- [x] **Step 5: Technical receipt freshnessを追加する**

設計§8.3のReceipt Fieldと3 policyをAI Verificationへ追加する。current time、input hash、revocation snapshotから四Freshness状態を導出し、期限切れReceipt再利用を拒否するfixtureを追加する。

- [x] **Step 6: Control Plane計画の全Task順を検査する**

Expected: Validator bootstrap→schema作成→Owner作成→bootstrap approval→metadata migration→Product generationの順であり、未承認Ownerを正式WPが使用しない。

### Task 3: AI E2E Operation、Caller Profile、Code ownerを閉じる

**Files:**
- Modify: `docs/architecture/02-foundation/executable-contracts.md`
- Modify: `docs/architecture/02-foundation/core-architecture.md`
- Modify: `docs/architecture/01-governance/ai-security-approval.md`
- Modify: `docs/architecture/03-authoring/editor-workspace-ux.md`
- Modify: `docs/architecture/03-authoring/native-game-module.md`

**Interfaces:**
- Consumes: 設計§6～8、Task 1のPhase 4／5 gate。
- Produces: Package／Device／Debug／Task Operation、caller/deployment/model Profile、Code owner契約、Beginner fallback。

- [x] **Step 1: Operation registryを拡張する**

設計§6の13 OperationをMCDへ追加する。各OperationにRisk、authority、required identity／hash、side effect、idempotency、cancel、Receipt、failure diagnosticを定義する。

- [x] **Step 2: `OperationTaskV1`を定義する**

`task_id`、`operation_id`、request hash、Project revision、Candidate root、Target、optional Device identity／generation、authorization、optional consent、idempotency key、state、Receipt refをclosed Fieldとして定義する。Task stateは`queued | running | cancel_requested | succeeded | failed | cancelled`とする。

- [x] **Step 3: Debug workflowをOperationへ束縛する**

Aggregate→Query→Causality／Replay→Finding validation→Proposalをexact Operationへ結線し、Support Bundleだけ別経路になる状態を解消する。

- [x] **Step 4: Caller／Provider／Deployment／Model Profileを追加する**

設計§7の6型を定義し、Host surface、Transport、Provider/runtime、Model snapshot、tool projection、authorityを別Fieldにする。hosted ChatGPT Workはplugin remote MCP Toolを使い、local Codex設定／direct local STDIO対応と表示しない。ChatGPT desktop appのCodex host、Codex CLI／IDE、Claude Desktop／Code、Cursorはsurface別conformance Receiptがある場合だけ対応表示する。

- [x] **Step 5: local inference境界を追加する**

model weights／quantization／license／provenance、RAM／VRAM、context、process／GPU isolation、IPC／loopback auth、schema／tool conformance、privacy／logging、resource exhaustion、explicit fallbackを`InferenceDeploymentProfileV1`へ定義する。first-party local inferenceをMVP gateにしない。

- [x] **Step 6: Code ownerとBeginner fallbackを追加する**

設計§8.1の2型、Role registry、absence policy、`awaiting_code_owner` UI stateを追加する。初心者MVPはDefinition-first／prequalified Pack、AI生成Native／ShaderはCode owner承認へ分離する。

- [x] **Step 7: AI E2E negative fixtureを追加する**

stale Candidate、Device差替え、Package Receipt不一致、無承認install/reset、偽Debug finding、Model／Host profile失効、silent cloud fallbackを拒否する。

### Task 4: Toolchain、D3D12 CX0、Shippingを修正する

**Files:**
- Modify: `docs/architecture/02-foundation/toolchain-dependencies.md`
- Modify: `docs/architecture/02-foundation/cpp23-modules.md`
- Modify: `docs/plans/2026-07-22-ai-readable-d3d12-backend-design.md`
- Modify: `docs/plans/2026-07-22-d3d12-backend-implementation-plan.md`

**Interfaces:**
- Consumes: 設計§9、Task 2のControl Plane baseline。
- Produces: Ajv pin、CX0 D3D12 surface、CX2 cutover WP、CX3 Shipping gate。

- [x] **Step 1: AjvをToolchain lockへ追加する**

Ajv `8.20.0`、MIT、npm integrityをexact記載し、Draft 2020-12 Control Plane schema validatorとしてだけ使用する。C++ runtime validatorの未固定行とは別Dependencyであることを明記する。

- [x] **Step 2: D3D12 public surfaceをCX0 Headerへ変更する**

`engine/rendering/d3d12/backend.ixx`を`engine/rendering/d3d12/include/mirakan/rendering/d3d12/backend.hpp`へ置換し、CMake Componentは将来Module名だけを登録する。CX1 fixture以外の`.ixx`／`.cppm`をTaskに残さない。

- [x] **Step 3: Control Plane baselineをschema refへ統一する**

D3D12設計の7 Field再列挙を削除し、`ControlPlaneBaselineV1`の全required Fieldをread-backする。`architecture_explain_schema_sha256`と`lint_version`を欠落させない。

- [x] **Step 4: PreviewとShippingを分離する**

CX0／CX1はDevelopment、Test、candidate Package、internal Technology Previewだけに許可し、Release Activation不可とする。CX2 cutoverとCX3 all-target activationを明示Work PackageとしてProductへ参照し、外部条件不成立時はCX0を維持する。

- [x] **Step 5: 公式Evidenceをread-backする**

MCP 2025-11-25、OpenAI MCP／GPT-5.6、Ajv Draft 2020-12、CMake 4.4、MSVC 14.51のURL、検証日、適用判断をToolchain evidence表へ追加する。

### Task 5: Closure追補と横断検証を行う

**Files:**
- Create: `docs/reviews/2026-07-23-plan-feasibility-remediation.md`
- Modify: `docs/reviews/2026-07-22-plan-review-closure.md`
- Modify: `docs/superpowers/plans/2026-07-22-plan-review-closure.md`

**Interfaces:**
- Consumes: Tasks 1～4とTask 6の最終diffと検査結果。
- Produces: remediation finding台帳、旧Closureとの関係、Go／No-Go Gate。

- [x] **Step 1: finding台帳を作成する**

Work Package schema、DAG、Phase binding、Phase 8 dependency、AI Operation、bootstrap approval、Code owner、Target readiness、Freshness、D3D12 CX0、Mobile split、Critical Path実行計画、Future inceptionを一行一Findingで記録し、changed documentsとverification evidenceを持たせる。

- [x] **Step 2: 旧Closureの意味を限定する**

253件terminalは2026-07-22 ledgerのDisposition closureであり、Engine implementation readinessではないことを追記し、新台帳を後続監査として参照する。

- [x] **Step 3: MarkdownとRegistryを検査する**

Run:

```powershell
git diff --check
rg -n 'TBD|TODO|schedule_state|backend\.ixx' docs/architecture docs/plans docs/superpowers
```

Expected: whitespace error 0、未説明placeholder 0、旧Work Package state Field 0、D3D12 Production `backend.ixx` 0。

- [x] **Step 4: 全local link／anchorを検査する**

Expected: missing path 0、missing anchor 0。

- [x] **Step 5: IDとDAGを検査する**

Expected: duplicate document ID 0、duplicate Product logical ID 0、unresolved ref 0、Work Package cycle 0、forward Phase dependency 0、Phase exit binding欠落0。

- [x] **Step 6: 最終差分を独立レビューする**

設計§11の8条件、外部Evidenceの適用範囲、active state非昇格、Scope外実装がないことを確認する。Critical／Important findingは同一ChangeSetで修正し再検査する。

### Task 6: Critical Pathと将来Capabilityの実行計画を作る

> 実行順序: Task番号は監査laneを維持するため6とするが、本TaskはTask 5の最終Closureより先に完了する。

**Files:**
- Create: `docs/plans/2026-07-23-critical-path-execution-plan.md`
- Create: `docs/plans/2026-07-23-future-capability-inception-plan.md`

**Interfaces:**
- Consumes: Task 1の最終Product Phase／Work Package／Future registry、Task 2～4のControl Plane・AI・Toolchain契約。
- Produces: Critical Path後半のtask-level plan、AAA／Open World／Online／Console等をactive scopeへ移す前のinception plan。

- [x] **Step 1: Critical Pathの未計画WPを全量列挙する**

Product registryからtask-level implementation planを持たないWPを抽出し、少なくともECS E1～E7、Headless Authoring、Editor Runtime、2D Shooter、AI Authoring MVP-A、Project Source Activation、External Agent、3D MVP-B、Mobile、C2 aggregateをPhase順に扱う。

- [x] **Step 2: 各Taskの実行契約を閉じる**

各Taskにprerequisite WP／baseline Receipt、Owner文書、予定成果物path、positive／negative fixture、verification command、completion receipt、rollback／last-validと、常在する`blocked_diagnostic_ref` Fieldを必須にする。Fieldは非blocked時`null`、blocked時は登録済みexact Diagnostic ID一件とし、未登録IDを推測しない。Critical PathのWPはProduct正本の`declared | deferred`をそのまま保持し、CX2／CX3／C2 3Dの3件を`declared`へ変えない。Future entryだけを`planning_only`で開始し、Work Package scheduling stateとFuture planning stateを同じclosed setへ混在させない。

- [x] **Step 3: 並列化とGateを明示する**

同一Phase内の安全な並列作業、join gate、Target別fan-out、Code owner／device／signingが必要な停止点をDAGで示す。calendar期間・人数は推測せず相対sizeと実測velocityからだけ更新する。

- [x] **Step 4: Future capabilityを製品単位へ分解する**

Open World、MMO／online／live service、大人数network shooter、vehicle、ragdoll、crowd、motion warping、AAA rendering、Terrain／Foliage／GI、Console、Web、XR、FPS、Asset generation、first-party local inference、runtime generationについて、architecture／threat model／licensing／Target／operations／positive＋negative fixture／rollbackのinception deliverableを定義する。

- [x] **Step 5: active registryへの移行条件を固定する**

Future entryはplaceholder APIや空Managerを作らない。Product Decision、Owner、Target、fallback、Requirement、positive＋negative fixture、WP、Risk、qualification policyが同一ChangeSetで揃った場合だけactive registryへ移す。モデル名・Provider名は能力保証に使わない。

- [x] **Step 6: 正本参照とリンクを検査する**

Product registryのexact IDだけを使い、存在しないWP／Capability／Gateを参照しない。local link、表列、`git diff --check`を検査する。
