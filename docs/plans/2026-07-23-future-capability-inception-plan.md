# Future Capability Inception Plan

> **For agentic workers:** This plan creates decision and qualification inputs only. It does not authorize implementation, capability activation, shipping claims, dependency acquisition, target-program enrollment, or provider selection.

**Goal:** Open World、online／MMO、large-session network shooter、FPS、advanced simulation、AAA rendering、Terrain／Foliage／GI、Console、Web、XR、commercial asset generation、local inference、runtime generationを、現在のMVPへ混入させず、実装開始時にOwner、Target、security、license、fixture、fallbackを推測しなくてよいinception dossierへ閉じる。

**Authority:** [Product Plan §8](../architecture/00-product/product-plan.md#8-future-portfolio)の`FutureCapabilityIncubationRegistryV1`がidentityと`planning_state`の唯一の正本である。本書は各entryの調査・Decision・prototype・qualificationの順序を定めるconsumerであり、Future ID、Capability、Phase、Target、Activation stateを所有しない。

**Initial state:** 全entryは`planning_only`である。本書を完了してもactive `CapabilityRegistryV1`、`WorkPackageRegistryV1`、`ProductPhaseRegistryV1`、`PhaseFixtureBindingRegistryV1`、Shipping labelへ自動転記しない。

## 1. Non-negotiable boundary

1. placeholder API、空Manager、無効Backend、unknown enum、`future_*` feature flagをactive sourceへ追加しない。
2. Vendor、LLM、SDK、console、store、cloud、middlewareの名称だけで対応済みと表示しない。
3. Prototypeは隔離branch／sandbox Targetに置き、current Project schema、Save、Package、Release artifactへ読込ませない。
4. Product Decision、Owner、Target、fallback、Requirement、positive／negative Fixture、Work Package、Risk、qualification policyを一つのChangeSetで登録できないentryは`planning_only`に留める。
5. active registryへ移した直後の`CapabilityTargetActivationV1`は各required／optional Targetとも`not_activated`で開始する。Decision、prototype、benchmarkはActivation Receiptを代替しない。
6. external dependencyはexact version／artifact hash／license／SBOM／security／更新・撤退policyが確定するまで取得・導入しない。Console等のNDA情報は公開文書へ複写しない。
7. 人数、期間、費用、同時接続数、World面積、Asset量、画質、frame rateはRequirementとTarget Profileが承認される前に推測しない。

## 2. Inception dossier contract

各Future entryは次の成果物をすべて持つ。`not_applicable`は理由、承認Decision、再検討条件を伴う場合だけ許可し、Field省略で代用しない。

```text
FutureCapabilityInceptionDossierV1
  future_capability_id
  product_problem_statement_ref
  excluded_current_claims[]
  owner_candidate_document_refs[]
  prerequisite_capability_receipt_refs[]
  product_scope_decision_ref
  authority_model_decision_ref?
  data_model_decision_ref?
  threat_model_ref?
  target_program_decision_ref?
  provider_selection_decision_ref?
  licensing_decision_ref?
  operations_decision_ref?
  target_profile_candidate_refs[]
  fallback_candidate_ref
  positive_fixture_spec_refs[]
  negative_fixture_spec_refs[]
  prototype_receipt_refs[]
  qualification_policy_candidate_ref
  rollback_and_exit_plan_ref
  unresolved_blockers[]
  evidence_snapshot_ref
```

`required_decision_kinds[]`に対応するDecision refは必須である。`prototype_receipt_refs[]`は隔離prototypeの結果だけを指し、Product Capability Receiptへ型変換しない。`evidence_snapshot_ref`は公式仕様のURL、取得日、version、artifact hash、licenseと、Miraikanai固有判断を分離する。

## 3. Common execution stages

### Stage I0: Scope and non-claim

- [ ] Product problem、対象User、作れるGameの境界、作れないGame、品質level、Target候補を記述する。
- [ ] `excluded_current_product_claims[]`と公開表現を照合し、MVP／Technology Preview／C2へ誤表示されないnegative testを作る。
- [ ] Requirementを数値化できない項目は`unresolved_blockers[]`へ残し、仮値を合格値として使わない。

**Exit:** R4 Product Decision draft、non-claim test、変更対象外のactive Registry hashが揃う。

### Stage I1: Owner, data, authority, threat model

- [ ] owner候補を既存Ownerへroutingし、責務が新規なら新Owner文書の提案を作る。
- [ ] authoritative state、identity、serialization、Save／Replay、migration、failure、rollbackをData Modelへ閉じる。
- [ ] network、runtime generation、provider、user content、account、telemetry、device sensorを扱う場合はTrust boundary、abuse、privacy、retention、revocation、incident responseをThreat Modelへ閉じる。
- [ ] client、server、Editor、runtime、AI、human、providerのauthorityを列挙し、同じStateを複数Ownerへ置かない。

**Exit:** required Decision kindごとのapproved Decisionまたは明示blockerがあり、Owner競合とauthority ambiguityが0件。

### Stage I2: Target, dependency, license, operations

- [ ] Target Profile、minimum hardware／OS／runtime、build／package／distribution、device labを候補化する。
- [ ] provider／middleware候補を少なくともfirst-party案とexternal案で比較し、license、source availability、artifact pin、security、update、exit costを記録する。
- [ ] serviceを持つ場合はSLOではなく最初にfailure mode、backup／restore、key rotation、moderation、on-call、cost ceiling、shutdownを定義する。
- [ ] official program／SDKへ参加していないTargetは`target_program_blocked`とし、APIやcertification内容を推測しない。

**Exit:** Target／dependency／license／operations Decisionと撤退可能なfallbackが揃う。未固定dependencyをprototype以外へ持ち込まない。

### Stage I3: Isolated prototype and falsification

- [ ] Production APIを作る前に、捨てられるprototypeで最も高いtechnical riskを一つずつ検証する。
- [ ] positive fixtureと同じ入力closureから、disconnect、packet loss、corrupt data、resource exhaustion、device loss、provider failure、license／profile失効等のsingle-cause negative fixtureを作る。
- [ ] Candidate、Toolchain、Target、Device、input hash、policy、Receipt freshnessを記録する。
- [ ] prototype失敗時はactive Productへfallbackせず、sandbox artifactを破棄して`planning_only`を維持する。

**Exit:** 再現可能なprototype Receipt、反証結果、未解決risk、rollback結果が揃う。成功結果だけの報告を禁止する。

### Stage I4: Atomic activation proposal

- [ ] §7のatomic migration setを一つのChangeSetとして生成する。
- [ ] Product registry validatorでduplicate、missing owner／Target／fallback、phase cycle、forward dependency、Target closure、Fixture subset、hidden future referenceを検査する。
- [ ] independent reviewerがcurrent claim、security、license、operations、qualificationを確認する。

**Exit:** R4承認可能なChangeSetが完成するか、entryは理由付き`planning_only`のまま閉じる。Approval後もActivation stateは`not_activated`である。

## 4. World, network, and live-service lanes

| Future entry | Inception deliverables | Mandatory positive fixtures | Mandatory negative／scale fixtures | Fallback／exit |
|---|---|---|---|---|
| `future.capability.offline-large-world-continuous-streaming` | World partition identity、cell ownership、streaming scheduler、origin／precision、Save checkpoint、memory／I/O budget、authoring rebuild | deterministic traversal、save／load across cells、bounded background streaming、Target別memory budget | corrupt／missing cell、I/O stall、teleport thrash、save during unload、budget exhaustion | compact loaded Worldへ戻し、large-world dataをcurrent Saveへ書かない |
| `future.capability.headless-dedicated-server-session-transport-replication` | server authority、session lifecycle、replication schema、interest management、protocol version、deployment／key／operations | deterministic headless session、join／leave、state reconciliation、version negotiation | loss／latency／reorder／duplicate、disconnect、server restart、malformed client、credential rotation | offline single-playerを維持し、network schemaをlocal Save正本にしない |
| `future.capability.small-cooperative-multiplayer` | player-count Requirement、join-in-progress、host／dedicated choice、Save ownership、host migration、voice／moderation boundary | two-or-more-client complete game loop、join／leave、save／resume、Target cross-play候補 | host loss、late join、desync、grief／abuse、NAT／relay failure、version mismatch | single-playerまたはapproved local multiplayerへ明示downgrade |
| `future.capability.rollback-competitive-networking` | fixed simulation boundary、input delay／prediction／rollback window、anti-cheat trust、ranked integrity | deterministic resimulation、bounded rollback、reconnect、spectator/replay separation | packet impairment matrix、cheat input、clock drift、state divergence、rollback budget exhaustion | non-ranked authoritative sessionへ明示downgradeし、ranked claimを停止 |
| `future.capability.large-session-network-shooter` | exact player/concurrency Requirement、large-session interest management、server sharding、bandwidth／CPU budget、match lifecycle | full match at approved concurrency on qualified server/client Targets | churn、hotspot、loss／latency matrix、server crash、DDoS/resource exhaustion、degraded region | smaller approved session capへ戻し、未達人数を公開しない |
| `future.capability.persistence-live-service-moderation-operations` | account／entitlement、persistent data、migration、live update、privacy、retention、moderation、incident／rollback | backup／restore、schema migration、moderation appeal、safe rollout／rollback、audit read-back | data corruption、partial region failure、abuse burst、key compromise、bad update、provider outage | service read-only／shutdown modeとoffline Product境界を維持 |
| `future.capability.mmo-distributed-world-authority` | shard／region authority、handoff、global identity、economy integrity、persistence、social／moderation、capacity／operations | cross-region handoff、persistent progression、recovery、controlled scale run | split brain、duplicate item／currency、region loss、mass reconnect、migration failure、abuse scale | entryを停止し、dedicated-session／offline large-world Capabilityへ分離 |

`large-session network shooter`は`small cooperative`や`rollback competitive`の別名ではない。player count、interest management、server topology、operations、security、scale fixtureが別Requirementとして承認されるまで一つのCapabilityへ統合しない。

## 5. Gameplay, simulation, and presentation lanes

| Future entry | Required decomposition | Mandatory positive fixtures | Mandatory negative／quality fixtures | Fallback／exit |
|---|---|---|---|---|
| `future.capability.first-person-shooter-profile` | first-person camera、weapon/view model、body visibility、aim／recoil、interaction ray、animation、audio、accessibilityをShooter Coreから独立Profile化 | full title-to-result FPS fixture、save/load/replay、keyboard/controller、Target-specific performance | camera clipping、body/view divergence、high FOV、motion-sickness options、input-device swap、replay mismatch | TPS Profileを変更せずFPS Profileだけ非対応にする |
| `future.capability.vehicle-ragdoll-crowd-motion-warping` | `vehicle`、`ragdoll`、`crowd`、`motion_warping`を四Capability／四Owner contract／四budgetへ分割 | 各機能のdeterministic isolated fixtureと組合せfixture | solver divergence、authority conflict、crowd overload、warp collision、Save／Replay mismatch | 機能ごとにkinematic／animation／low-count fallbackへ戻す |
| `future.capability.production-terrain-foliage-gi` | Terrain、Foliage、GIを別artifact pipeline、streaming、cook、fallback、licenseへ分割 | authoring→cook→package、Target比較、quality reference、streaming／LOD | missing tile、shader/provider failure、memory spike、dense overdraw、light leak、device loss | mesh terrain、baked sparse foliage、baked lighting等のapproved C1 fallback |
| `future.capability.aaa-photoreal-rendering` | measurable image-quality rubric、PBR/material、lighting、shadow、reflection、post、VFX、character/environment asset budgetsを別Capabilityへ分割 | blind reference comparison、camera sequence、multiple content sets、qualified hardware/Target runs | temporal instability、artifact stress、memory／compile／load spike、fallback fidelity、provider unavailable | current C1/C2 rendererへ戻し、`AAA`／`photoreal` labelを発行しない |

`AAA`はfeature list、解像度、特定Vendor techniqueだけで合格にしない。承認済みquality rubric、content sample set、Target performance／memory／load、production workflow、fallback、blind review Receiptが同一Candidateへ閉じた場合だけProduct claim候補になる。

## 6. Target, content, and AI lanes

| Future entry | Inception deliverables | Mandatory positive fixtures | Mandatory negative fixtures | Fallback／exit |
|---|---|---|---|---|
| `future.capability.console-target-program` | program participation、NDA Owner、SDK/toolchain pin、platform services、certification、device lab、release operations | NDA環境内build/package/install/launch、save/input/network/store checks | certification failure、SDK change、signing/key loss、device generation change | Target rowを登録せずDesktop/Mobileを維持 |
| `future.capability.web-target-program` | browser/runtime、graphics API、threading、storage、download/cache、security headers、browser matrix | clean browser launch、save/load、input/audio、offline/cache policy、download budget | context loss、storage denial、tab suspension、network interruption、unsupported browser | Web Targetを非対応にしnative packageを維持 |
| `future.capability.xr-target-program` | runtime／device program、stereo render、tracking space、input、comfort、accessibility、frame／latency budget | device-native play loop、tracking/input/recenter、comfort options、performance | tracking loss、boundary change、motion-to-photon miss、device unplug、permission denial | non-XR 3D profileへ戻し、XR claimを停止 |
| `future.capability.commercial-asset-generation-license-quality-qualification` | model/provider provenance、input/output license、artist consent、style/IP policy、quality rubric、human review、takedown | blind quality evaluation、provenance manifest、editability、Target import/cook | prohibited input、license conflict、watermark/provenance loss、unsafe content、provider removal | human/approved-library asset workflowを維持 |
| `future.capability.first-party-local-inference` | runtime/model selection、weights/quantization/tokenizer hashes、license/provenance、hardware floor、sandbox、update/revocation、quality eval | offline tool/schema conformance、representative authoring eval、resource-limit enforcement | corrupt/revoked model、RAM/VRAM shortage、timeout、prompt/tool abuse、silent cloud fallback | external conformance Host／explicit cloud profileまたはAIなしのmanual path |
| `future.capability.runtime-structured-data-generation` | server/runtime authority、allowed Operation、bounded schema、quota、moderation、baseline/signature、rollback、Target failure policy | deterministic accepted proposal→validate→commit/rollback flow | adversarial output、quota exhaustion、schema drift、moderation failure、disconnect、replay mismatch | deny-only `capability.product.runtime-generation-boundary`を維持 |

Gemma、Kimi、Qwen、DeepSeek、Grok、GLM等のModel familyはFuture Capability IDにしない。`ModelSnapshotProfileV1`、`InferenceDeploymentProfileV1`、Tool／Schema Eval Receiptが同じ要求を満たすかで判定し、model名によるEngine branchを作らない。

## 7. Atomic migration set

Future entryをactive Product DAGへ移すChangeSetは、次をすべて同時に含む。欠落時はmergeとmaterializationを拒否する。

1. Future entryを対象とするR4 `ProductDecisionV1`と、移行後のcurrent claim／non-claim。
2. approvedまたは`review`開始条件を満たすOwner文書とreciprocal relation。
3. exact `TargetProfileV1`、Target-specific fallback、unsupported Targetの明示`excluded`。
4. stable `RequirementRegistryV1` entryとfailure Diagnostic。
5. positive Fixture、各主要failureのsingle-cause negative Fixture、scale／security／operations／target-device Fixture。
6. canonical 12-field `WorkPackageRegistryV1` entryと、同Phase内または先行Phaseだけを参照するDAG edge。
7. `CapabilityRegistryV1` entry、全required／optional Targetの初期`CapabilityTargetActivationV1(state=not_activated)`行。
8. `PhaseFixtureBindingRegistryV1` entryまたは独立Decision Gate。Candidate bindingとfreshness policyをexact参照する。
9. `ProductRiskRegistryV1` entry、mitigation、fallback、review gate。
10. dependency lock、license、SBOM、Threat Model、operations、rollback／last-validのうち該当する全Artifact。
11. duplicate／missing ref／DAG cycle／forward dependency／Target closure／Fixture subset／local link／table／schema testの0-error Receipt。

Future entryはactive registryから参照を受ける前に、`planning_only` recordをapproved Decisionでretireし、移行先IDとの一方向migration relationを残す。同じidentityをFutureとactiveの両方で同時に有効化しない。

## 8. Verification commands after Control Plane bootstrap

以下は[Architecture Evolution Control Plane implementation plan](2026-07-22-architecture-evolution-control-plane-implementation-plan.md)のTaskが実装され、baseline approvalが有効な場合だけ実行する。tool未実装をPASS扱いしない。

```powershell
npm ci --prefix tools/architecture_lint --ignore-scripts --offline --no-audit --no-fund
npx --prefix tools/architecture_lint tsc --build --force --singleThreaded
node --test tools/architecture_lint/test/*.test.mjs
node tools/architecture_lint/dist/main.js check
node tools/architecture_lint/dist/main.js generate-index --check
git diff --check
```

Expected:

- Future ID duplicate 0、active registryからFuture IDへの参照0。
- `planning_only` entryからCapability Target Activation行の生成0。
- atomic migration testのmissing Owner／Target／fallback／Requirement／Fixture／WP／Risk／policyが各一原因で拒否される。
- external dependency、Target program、provider、performance値の未承認推測0。
- local Markdown path／anchor missing 0、table column error 0。

## 9. Completion condition

本計画の「完了」はFuture機能の実装完了ではない。各entryが、(a) atomic activation proposalへ進める証拠を持つ、または(b) blocker、反証、撤退条件を持って`planning_only`に留まる、のどちらかへ一意に分類された時だけinception完了とする。Game制作で当該機能を利用できるとの公開表示は、active移行後のTarget別fresh Production Receiptまで禁止する。
