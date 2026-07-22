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
  product_scope_decision_ref: ref | null
  authority_model_decision_ref: ref | null
  data_model_decision_ref: ref | null
  threat_model_decision_ref: ref | null
  target_program_decision_ref: ref | null
  provider_selection_decision_ref: ref | null
  licensing_decision_ref: ref | null
  operations_decision_ref: ref | null
  qualification_decision_ref: ref | null
  target_profile_candidate_refs[]
  fallback_candidate_ref
  registry_reset_transition_policy_ref: ref | null
  positive_fixture_spec_refs[]
  negative_fixture_spec_refs[]
  prototype_receipt_refs[]
  qualification_policy_candidate_ref
  rollback_and_exit_plan_ref
  unresolved_blockers[]
  evidence_snapshot_ref
```

`required_decision_kinds[]`とDossier Fieldの対応は次だけを許す。各refは[Control Plane Design §7.1](2026-07-22-architecture-evolution-control-plane-design.md#71-header-format)のapproved Owner documentと、そのcurrent bytesのSHA-256／Git blob IDをread-backできるnon-null Architecture Approval Decision refへ解決する。現在のactive schemaにない専用Decision型を作って代用しない。

| required decision kind | exact dossier field | approvalを要求するStage |
|---|---|---|
| `product_scope` | `product_scope_decision_ref` | I1 |
| `authority_model` | `authority_model_decision_ref` | I1 |
| `data_model` | `data_model_decision_ref` | I1 |
| `threat_model` | `threat_model_decision_ref` | I1 |
| `target_program` | `target_program_decision_ref` | I2 |
| `provider_selection` | `provider_selection_decision_ref` | I2 |
| `licensing` | `licensing_decision_ref` | I2 |
| `operations` | `operations_decision_ref` | I2 |
| `qualification` | `qualification_decision_ref` | I4 |

Registry entryの`required_decision_kinds[]`にあるkindは対応Fieldを必須、ないkindは`null`とし、別Field、自由文字列、配列順、Provider名から推論しない。`not_applicable`は対応Owner documentが理由、承認主体、再検討条件をDecisionとして承認したrefで表し、Field省略やliteral値で表さない。`prototype_receipt_refs[]`は隔離prototypeの結果だけを指し、Product Capability Receiptへ型変換しない。`evidence_snapshot_ref`は公式仕様のURL、取得日、version、artifact hash、licenseと、Miraikanai固有判断を分離する。

`registry_reset_transition_policy_ref`はDecision kindではなく、I0～I3では`null`、active migrationを組み立てるI4ではnon-nullとする。refはProduct Ownerが登録しcurrent bytesをread-backできるapproved exact transition policy revisionへ解決し、DossierがPolicy IDやFieldを発明しない。PolicyはProduct Registryのapproved major／revision移行だけを対象とし、`ready | active | blocked | complete -> declared`の降格だけを許可し、昇格、`deferred`解除、Receipt carry-forward、artifact削除を禁止するclosed規則を持つ。登録先Owner、current document hash、approval ref、policy content hashをmigration manifestへ含め、一件でも解決できなければI4とmaterializationを拒否する。

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

**Exit:** `product_scope | authority_model | data_model | threat_model`のうち当該entryが要求するkindだけにapproved Decision refまたは明示blockerがあり、Owner競合とauthority ambiguityが0件。I2／I4で所有するkindをこのStageの成功条件へ前倒ししない。

### Stage I2: Target, dependency, license, operations

- [ ] Target Profile、minimum hardware／OS／runtime、build／package／distribution、device labを候補化する。
- [ ] provider／middleware候補を少なくともfirst-party案とexternal案で比較し、license、source availability、artifact pin、security、update、exit costを記録する。
- [ ] serviceを持つ場合はSLOではなく最初にfailure mode、backup／restore、key rotation、moderation、on-call、cost ceiling、shutdownを定義する。
- [ ] official program／SDKへ参加していないTargetは、`unresolved_blockers[]`へ「Target program未参加」、根拠Evidence ref、Owner、解除条件、再検討Decision kindを記録し、未定義state tokenやAPI／certification内容を推測しない。

**Exit:** `target_program | provider_selection | licensing | operations`のうち当該entryが要求するkindだけにapproved Decision refまたは明示blockerがあり、撤退可能なfallbackが揃う。`qualification`はI4が所有し、未固定dependencyをprototype以外へ持ち込まない。

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

**Exit:** `qualification_decision_ref`を含む全`required_decision_kinds[]`のapproved refと、active migrationではnon-null `registry_reset_transition_policy_ref`が揃い、R4承認可能なChangeSetが完成するか、entryは理由付き`planning_only`のまま閉じる。Approval後もActivation stateは`not_activated`である。

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

1. Future entryを対象とするapproved Architecture Decision document、そのnon-null Architecture Approval Decision ref、移行後のcurrent claim／non-claim、および`future_capability_id`から新しいactive Registry ID集合へのexact mapping。active schemaにない専用Decision型を使用しない。
2. current bytesをApproval Decisionからread-backできる`state=approved`のOwner文書とreciprocal relation。`review`文書ではactive migrationしない。
3. exact `TargetProfileRegistryV1` entry、exact `FallbackRegistryV1` entry、unsupported Targetの明示`excluded`。現行WP／Capabilityの`fallback_ref`は単数でTarget mapを持たないため、Targetごとに意味が異なるfallbackが必要ならCapability／WPを分割するか、明示的なProduct schema migrationを同じChangeSetへ含め、隠れたTarget別分岐を作らない。
4. stable `RequirementRegistryV1` entryとfailure Diagnostic。
5. positive `FixtureRegistryV1` entry、各主要failureのsingle-cause negative Fixture、scale／security／operations／target-device Fixture。各FixtureのRequirement／Target集合外をGateから参照しない。
6. canonical 12-field `WorkPackageRegistryV1` entryと、同Phase内または先行Phaseだけを参照するDAG edge。
7. Product scope Decisionが既存Phaseを選ぶ場合は、その`ProductPhaseRegistryV1.work_package_refs[]`へ同じWP ID、`outcome_requirement_refs[]`へ同じRequirement ID、`exit_gate_refs[]`へ項目9のGate IDを追加するreciprocal update。新Phaseが必要な場合は、unique `phase_id`／order、outcome Requirement、全WP、全exit gateを持つ`ProductPhaseRegistryV1` entryを同じChangeSetへ追加する。どちらもWP→PhaseとPhase→WPの片側更新、unreferenced Gate、duplicate order、forward dependencyを拒否する。
8. `CapabilityRegistryV1` entry、全required／optional Targetの初期`CapabilityTargetActivationV1(state=not_activated)`行。
9. outcome Requirementを評価する`PhaseFixtureBindingRegistryV1` entry。bindingの`phase_id`は項目7のPhaseと一致し、その`gate_id`を同じPhaseの`ProductPhaseRegistryV1.exit_gate_refs[]`からexact一回参照する。Candidate bindingとfreshness policyをexact参照する。`ProductDecisionGateRegistryV1`は再検討条件に必要なら追加できるが、Phase exit gateを代替せず`ProductPhaseRegistryV1.exit_gate_refs[]`へ含めない。
10. `ProductRiskRegistryV1` entry、mitigation、fallback、review gate。
11. dependency lock、license、SBOM、Threat Model、operations、rollback／last-validのうち該当する全Artifact。
12. destination `WorkPackageRegistry`の全IDを一件ずつ覆う更新済み[Critical Path Execution Plan](2026-07-23-critical-path-execution-plan.md)。新規／変更WPごとに`CriticalPathTaskV1`のprerequisite、Owner、artifact root、positive／negative fixture、verification、completion Receipt、rollback／last-valid、size、常在`blocked_diagnostic_ref`を閉じ、Phase DAGと並列laneを更新する。destination WP setとの差、unknown ID、Phase mismatchが一件でもあれば開始しない。
13. duplicate／missing ref／DAG cycle／forward dependency／Phase↔WP reciprocal／Critical Path task coverage／Target closure／Fixture subset／local link／table／schema testの0-error Receipt。
14. [Control Plane Design §6.1](2026-07-22-architecture-evolution-control-plane-design.md#61-control-plane-bootstrap-approval)と同じ二段tree手順で再発行し、destination Product Registry hashへ束縛されたcurrent `ControlPlaneBootstrapApprovalV1`。旧Approval、Architecture Approval Decisionだけ、baseline hashの文書更新だけで代用しない。

現行`FutureCapabilityIncubationRegistryV1`は`planning_state=planning_only`以外のstateとmigration relation Fieldを持たない。required Field追加とclosed enum拡張はmajor変更であるため、V1をin-place変更、`retire`、削除、暗黙mappingしてはならない。最初のactive移行ChangeSetは`FutureCapabilityIncubationRegistryV2`を次のclosed形で導入し、既存17行を同じChangeSetでV2へ全量変換する。

```text
FutureCapabilityIncubationRegistryV2
  registry_id: exact V1 registry_id (logical identityを維持)
  format_major: 2
  revision: 1
  entries[]:
    future_capability_id
    owner_document_id
    planning_state: planning_only | migrated
    prerequisite_capability_refs[]
    required_decision_kinds[]
    candidate_target_kinds[]
    qualification_fixture_kinds[]
    activation_trigger
    excluded_current_product_claims[]
    migration_decision_ref: null | Architecture Approval Decision ref
    active_capability_refs[]
    active_work_package_refs[]
```

`registry_id`はschema versionをidentityにしないProduct共通規則に従ってV1 rootからbyte-exactに維持し、`format_major=2, revision=1`をV2の初期rootとする。V1から継承する最初の9 entry Fieldは、`planning_state`のclosed enumへ`migrated`を追加する点を除き、名前、型、必須性、配列順序非依存、参照先、validation semanticsを変更しない。V2は上記4 root Fieldと12 entry Fieldだけを持つclosed schemaであり、追加のoptional Field、unknown Field、Field省略を受理しない。`planning_only`は`migration_decision_ref=null`かつ両active ref配列が空、`migrated`はnon-null approved Decision refと1件以上の両active refを必須とする。active refは同じChangeSetで登録されたIDへexact解決し、Future Registryからactive Registryへの一方向参照だけを許す。active RegistryからFuture IDへの逆参照は禁止する。移行対象行だけを`migrated`へ変更し、他16行を`planning_only`のまま明示保持する。同じFuture IDを削除／再利用せず、同じactive IDを複数Future行へ割り当てない。

V1→V2 migration manifestは、sourceの`registry_id`、`format_major=1`、read-backしたcurrent `revision`、canonical V1 schema SHA-256、canonical V1 root SHA-256と、destinationの同一`registry_id`、`format_major=2`、`revision=1`、canonical V2 schema SHA-256、canonical V2 root SHA-256を必須Fieldとして署名対象へ含める。source root／schema hashのread-back不一致、destination root／schema hashの再計算不一致、registry ID差は移行全体を拒否する。ChangeSetは全17 rowのID集合／内容比較、migration manifest、Product Planの共通root規則とcanonical型名、Control Plane schema／URN allowlist／baseline hash、generator、validator、全consumer ref、fixture、generated indexを同時更新する。V1／V2のdual read、optional new Field、default値、compatibility aliasを設けず、current treeにV1 consumerが残る場合はmaterializationを拒否する。V2全量生成とread-backが失敗した場合はChangeSet全体をrollbackし、V1と全entryの`planning_only`をlast-validとして維持する。

初回cutover後に残るentryは、V2 same-major revision migrationで一件ずつ移す。source rootはcurrent `registry_id, format_major=2, revision=N`、destination rootは同一`registry_id, format_major=2, revision=N+1`とし、`N`はsource bytesからread-backする。revisionのskip、再利用、reset、schemaが許す上限からのoverflowを拒否し、entry Fieldまたはclosed enumを変える場合はsame-majorにせず別のapproved major migrationへ戻す。

V2 revision manifestはsource／destinationのregistry ID、major、revision、同一canonical V2 schema SHA-256、両root SHA-256、selected `future_capability_id`、§7項目1のapproved Architecture Approval Decision refを署名対象にする。source／destinationとも現行17 IDのset equalityを必須とし、全17行のcanonical row hashを比較する。selected rowはsourceで`planning_only`、destinationで`migrated`でなければならず、変更できるFieldは`planning_state`、`migration_decision_ref`、`active_capability_refs[]`、`active_work_package_refs[]`だけである。他8 Fieldはbyte-exactに維持する。non-selected 16行はrow全体がbyte-exact、既に`migrated`の行はDecision refとactive refを含めimmutableとする。一ChangeSetで複数entryを選択、IDを追加／削除／再利用、active refを別Futureへ再割当した場合は拒否する。Future portfolioのmembership自体を変える場合は本migrationと分離し、Product Planと本書のexpected setを先にapproved revisionとして更新する。

各V2 `N -> N+1`も§7の全atomic setと§7.1のreset、二段Bootstrap Approval再発行、全面revalidationを省略しない。失敗時はsource V2 revision、source Approval、source last-valid artifactへ一組で復帰し、destination rootまたはselected rowの一部だけを保存しない。

両migration manifestに専用署名wrapperや未登録purposeを作らない。canonical manifest hashを§7項目1のapproved Architecture Decision documentへexact Artifact refとして含め、そのcurrent Decision bytes hashを新しい`ControlPlaneBootstrapApprovalV1.payload.decision_sha256`へ束縛する。この二段read-backを本文の「署名対象」とし、Decision／Bootstrap Approvalのどちらか一方だけでは移行を許可しない。

### 7.1 current ApprovalとEvidenceの再確立

Product Registry projectionが一byteでも変わると、旧`ControlPlaneBootstrapApprovalV1.payload.product_registry_sha256`はcurrent bytesと一致せず`revoked`評価になる。移行は次の順序だけを許す。

1. source tree、current Bootstrap Approval、全Registry root hash、lifecycle head、Activation row、Phase Gate Receiptをread-backしてmigration manifestへ固定する。不一致があれば開始しない。
2. 第一段target treeへ§7の全Artifact、destination Product Registry、初回ならV2 schema、全consumer更新、destination WP ID集合をexact coverageする更新済みCritical Path Plan、およびDossierの`registry_reset_transition_policy_ref`が指すapproved Product-owned Policy revisionを置く。`declared`は維持し、`deferred`は`defer_reason`と`reconsideration_gate_refs[]`をbyte-exactに維持する。Policy refが未解決、未承認、または本節のclosed降格規則と不一致なら停止する。
3. 降格対象WPごとにappend-only `WorkPackageLifecycleRecordV1`を追加する。`from_scheduling_state`はsource lifecycle headの`to_scheduling_state`値とbyte-exact、`to_scheduling_state=declared`、`product_registry_sha256`はdestination hash、`candidate_binding_hash`はsource lifecycle headの同名Field値とbyte-exact、`transition_policy_ref`はDossierの`registry_reset_transition_policy_ref`とbyte-exact、`receipt_refs=[]`、`decision_ref`は§7項目1のapproved Architecture Approval Decision refとbyte-exactにする。missing head、state不一致、別Candidateへの置換はtarget treeを拒否する。
4. 全required／optional `CapabilityTargetActivationV1`をdestination treeで`state=not_activated, candidate_ref=null, receipt_refs=[], evidence_freshness=expired`へ降格する。旧Activation Receipt、旧Phase Gate Receipt、旧Product label／milestone／Completion Receiptはimmutableな監査履歴としてだけ残し、destination current projectionへ入力しない。
5. 第一段target treeのexact Git tree ID、destination Product Registry hash、current 5 Owner hash、Toolchain lock、§7項目1の同じapproved Architecture Approval Decision refを使い、独立R4主体が新しい`ControlPlaneBootstrapApprovalV1`を署名する。第二段treeへApproval Recordを格納し、署名、Role／assignment／independence／revocation、全hash、ancestor、current bytesをread-backする。target tree作成からread-back成功まで新しいlifecycle transition、Activation昇格、Phase success、Product milestone、authoritative materializationを禁止する。
6. read-back成功後だけdestinationをcurrentとして一回で切り替え、target treeでdestination ID集合とのset equalityを確認済みの更新版[Critical Path Execution Plan](2026-07-23-critical-path-execution-plan.md)をPhase順に再実行する。全non-deferred WPはdestination hashのfresh lifecycle chain、全Target Activationはcurrent Candidate／Target／Policyのfresh Receipt、全Phase Gateはdestination revisionのfresh Receiptを新規生成する。既存Phase entryを変更しなかった場合もPhase GateとProduct milestoneは再実行し、旧aggregate Receiptを流用しない。

現行Product／`WorkPackageLifecycleRecordV1`契約にはentry-local hashで安全性を証明するEvidence rollover型がないため、上記全面revalidationに例外を設けない。将来carry-forwardを導入する場合は、本移行とは別のapproved major ChangeSetでProduct-owned closed schema、transitive closure、signature purpose、freshness／revocation、negative fixtureを先に定義してから本節を改訂する。未定義の同値判定、同じ表示名、同じartifact path、同じCandidate名をcarry-forward根拠にしない。第二段Approvalまたは再実行が一件でも失敗した場合はdestinationをcurrentにせず、source last-valid tree（初回はV1、以後はcurrent V2 revision）、そのlast-valid artifact、source Approvalを一組として復帰し、二つのProduct Registry revisionを同時materializeしない。

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
