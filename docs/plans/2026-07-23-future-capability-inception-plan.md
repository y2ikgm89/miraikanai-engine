# Future Capability Inception Plan

> planning-only Capabilityを調査・試作・正式移行するための計画。本書の完了は実装、Dependency取得、Target program参加、Capability Activation、Product claim、Releaseを承認しない。

> **Status: planning-only, not an implementation authorization.** 実行入口、official-source の採用境界、historical 文書の扱いは[計画書の実行権限・準備状況](README.md)に従う。I0–I2の調査記録は current Product／Future stateを変更せず、I3以降は current Future approval、Owner Decision、isolated candidate、exact Toolchain／Target Evidence が揃うまで開始しない。

## 1. 結論

現行MVPを過大化せず、将来要求を捨てない最適方針は、Future PortfolioをActive Product Definitionから完全に分離し、十分なEvidenceが揃った項目だけをRebaseline付きActive Definition migrationで昇格することである。

current exact portfolioは次である。

- Future capability: 25件。
- Product claim definition: 57件。
- Future policy: 2件。
- Definition closure: `FutureCapabilityIncubationRegistryV1`、`ProductClaimDefinitionRegistryV1`、`FuturePortfolioPolicyRegistryV1`のexact 3 Registry。
- Initial state: 全25件`planning_only`。

MVP-A、MVP-B、Technology PreviewはActive ProductのPhase／Release Gateが所有し、Future claimには含めない。

## 2. Authorityとcurrent性

identity、Owner、prerequisite、required decision kind、candidate Target kind、fixture kind、activation trigger、excluded claimは[Product Plan §8／§11](../architecture/00-product/product-plan.md)だけが所有する。本書はそのconsumerである。

作業開始時に次をread-backする。

- current `FuturePortfolioDefinitionClosureV1`と`future_portfolio_definition_sha256`。
- current完成`FuturePortfolioApprovalV1` wrapper ref／hash／sequence。
- `FuturePortfolioPolicyRegistryV1` exact 2 rows。
- current Active Product Definition hashとoperational snapshot。
- current Control Plane baseline binding、Toolchain、Trust／revocation。

Approvalがexpired／revoked、Head branch、definition hash差なら調査結果を履歴として保持しても新規prototype／promotionへ使わない。初回`ControlPlaneBootstrapApprovalV1`を後続promotionへ流用しない。

## 3. Exact portfolio

| Track | Future capability ID | 主要な閉じる範囲 |
|---|---|---|
| World | `future.capability.offline-large-world-continuous-streaming` | offline partition、streaming authority、Save、memory budget |
| Simulation | `future.capability.alternate-simulation-cadence-and-substep` | alternate fixed／variable／turn-based／explicit-step、`SimulationAdvanceIntervalV1`、Physics／Navigation、Animation、VFX、Input／Replay、Debug hang／step、LOD、Gameplay Feature timers／cadence、Pause、Native ABI、Save、Target qualification |
| Network | `future.capability.headless-dedicated-server-session-transport-replication` | headless Target、server authority、session transport、replication、operations |
| Network | `future.capability.small-cooperative-multiplayer` | player count、join／leave、Save、host migration |
| Network | `future.capability.rollback-competitive-networking` | determinism、input delay、rollback、anti-cheat、ranked operation |
| Network | `future.capability.large-session-network-shooter` | region／server authority、scale、recovery、SLO |
| Operations | `future.capability.persistence-live-service-moderation-operations` | persistence、update、moderation、privacy、retention、incident response |
| Network | `future.capability.mmo-distributed-world-authority` | shard／region、distributed world、failure recovery |
| Simulation | `future.capability.vehicle-ragdoll-crowd-motion-warping` | four independent capabilities、provider、scale／Target qualification |
| Domain | `future.capability.first-person-shooter-profile` | first-person camera／aim／presentation、comfort、Accessibility、independent fixture |
| Rendering | `future.capability.production-terrain-foliage-gi` | separate Terrain／Foliage／GI pipeline、fallback、quality／budget |
| Rendering | `future.capability.aaa-photoreal-rendering` | content pipeline、lighting／material／environment／VFX fidelity、device budget |
| Platform | `future.capability.console-target-program` | exact program、NDA Owner、SDK、certification、device lab、release operation |
| Platform | `future.capability.web-target-program` | graphics、storage、threading、download、browser matrix |
| Platform | `future.capability.xr-target-program` | runtime、tracking、comfort、input、render budget、device matrix |
| AI Asset | `future.capability.commercial-asset-generation-license-quality-qualification` | provider／output license、provenance、quality rubric、human review、takedown |
| AI Local | `future.capability.first-party-local-inference` | runtime／loader、model license、hardware floor、sandbox、update／revocation、Eval |
| AI Host | `future.capability.managed-external-host-execution` | exact Host／Transport／Provider Runtime／managed deployment identity／Model／Tool／Authority、`HostTransportConformanceReceiptV1`／`ProviderToolConformanceReceiptV1`／`SchemaEvalConformanceReceiptV1`、attestation、managed Source edit／Build、resource／credential budget |
| AI Runtime | `future.capability.runtime-structured-data-generation` | authority、quota、moderation、rollback、Target failure policy |
| Governance | `future.capability.proof-carry-forward-definition-migration` | row-level proof、selective carry-forward、adversarial migration fixture |
| Platform | `future.capability.linux-desktop-target-program` | exact distro／kernel／libc／architecture、I/O、package、device matrix |
| Platform | `future.capability.macos-desktop-target-program` | exact macOS／architecture、Metal／I/O、signing／notarization、device matrix |
| Platform | `future.capability.mobile-project-native-shader-source-qualification` | ABI／compiler／shader compiler／sandbox／Store policy per mobile Target |
| Product | `future.capability.local-multiplayer-split-screen` | player assignment、viewport、focus、audio listener、profile／Save、Accessibility |
| Scripting | `future.capability.unrestricted-project-scripting-runtime` | VM／language、sandbox、determinism、hot reload、debug、package、mod trust |

表の説明はroutingであり、RegistryのFieldを置換しない。ID集合はProduct Planの25件とset equality、duplicate／missing／extra 0でなければならない。

## 4. Inception dossier contract

各Future IDごとに、authorityを持たないcontent-addressed unsigned planning artifactとして次を作る。Dossier自身のhashは改変検知だけに使い、Decision／Approval／Promotion権限を与えない。Field省略や`TBD`文字列で`not_applicable`を表さない。

```text
FutureProposedTargetProfileV1
  proposal_id
  proposal_version: positive uint32
  row_change:
    {kind: add,
     expected_current_target_profile_ref: null}
    | {kind: update,
       expected_current_target_profile_ref: TargetProfileRefV1}
  proposed_target_id
  proposed_owner_document_id
  proposed_profile_version: positive uint32
  proposed_target_kind:
    headless_server | desktop | mobile | console |
    web | xr | distributed_cluster
  proposed_surface_role:
    execution_host | artifact_runtime | both
  proposed_qualification_requirement:
    {kind: current_requirement,
     requirement_id:
       exact RequirementRegistryV1.requirement_id}
    | {kind: proposed_destination_requirement,
       requirement_candidate_ref:
         exact ArtifactRefV1(
           artifact_kind=future_target_requirement_candidate,
           schema_version=1),
       requirement_candidate_sha256: lowercase hex 64}
  target_program_evidence_ref: EvidenceArtifactRefV1
  profile_document_candidate_ref:
    exact ArtifactRefV1(
      artifact_kind=future_target_profile_document_candidate,
      schema_version=1)
  profile_document_candidate_sha256
  proposal_content_hash

FutureTargetRequirementCandidateV1
  requirement_id
  owner_document_id
  verification_kind
  failure_diagnostic_id
  requirement_candidate_content_hash

FutureTargetProfileCandidateRefV1 =
  {kind: current_profile,
   target_profile_ref: TargetProfileRefV1}
  | {kind: proposed_destination_profile,
     proposed_profile_ref:
       exact ArtifactRefV1(
         artifact_kind=future_proposed_target_profile,
         schema_version=1),
     proposed_profile_sha256: lowercase hex 64}

FutureCapabilityInceptionDossierV1
  dossier_id
  source_future_portfolio_definition_sha256
  source_future_portfolio_approval_ref
  source_future_portfolio_approval_sha256
  future_capability_id
  problem_statement_ref, problem_statement_sha256
  owner_candidate_document_refs[]
  prerequisite_active_capability_evidence_refs[]
  prerequisite_future_promotion_manifest_refs[]
  decision_inputs[]:
    {decision_kind, owner_document_id, status,
     candidate_ref: null | content ref,
     candidate_sha256: null | lowercase hex 64,
     approval_ref: null | content ref}
  target_kind_resolution[]:
    {candidate_target_kind:
       headless_server | desktop | mobile | console |
       web | xr | distributed_cluster,
     target_profile_refs[1..64]:
       FutureTargetProfileCandidateRefV1}
  target_profile_candidates[]:
    FutureTargetProfileCandidateRefV1
  execution_surface_candidates[]:
    {execution_target_profile_ref:
       FutureTargetProfileCandidateRefV1,
     artifact_target_profile_ref:
       null | FutureTargetProfileCandidateRefV1}
  fallback_candidate_ref, fallback_candidate_sha256
  positive_fixture_spec_refs[]
  negative_fixture_spec_refs[]
  prototype_receipt_refs[]
  qualification_policy_candidate_ref
  dependency_candidate_refs[]
  license_and_provenance_closure_ref
  threat_model_ref
  operations_and_exit_plan_ref
  unresolved_blockers[]
  evidence_snapshot_ref, evidence_snapshot_sha256
  generated_at, revocation_snapshot_ref
```

`decision_inputs[]`はsource Future rowの`required_decision_kinds[]`とset equalityにし、各kindのOwner、candidate、Approvalを推測しない。`status`は`unresolved | proposed | approved | rejected | not_applicable_approved`のclosed enumである。`approved`と`not_applicable_approved`だけがnon-null completed human Decisionを持つ。

`FutureProposedTargetProfileV1`はDossier用のauthority-free planning artifactであり、Target対応、Target Profile registration、Build／Package／Shipping、ConformanceまたはActivationを表さない。`proposal_content_hash`はASCII `MIRAKAN_FUTURE_PROPOSED_TARGET_PROFILE_V1`と自己hashを除くclosed recordのRFC 8785 JCS bytesを`uint32_be` length framingして計算し、Artifact ref／隣接hashとbyte equalityにする。`row_change.kind=add`は`proposed_target_id`がsource Active Definitionの`TargetProfileRegistryV1`に存在せずexpected refをnull、`update`は同IDのcurrent完成Profile refをnon-nullにし、`proposed_profile_version`をexact `N+1`とする。addで既存ID、updateでmissing／別ID／別version、profile document／evidence hash差を拒否する。

`proposed_qualification_requirement.kind=current_requirement`はsource Active Definitionのexact `RequirementRegistryV1` rowだけを参照する。新Targetに必要なRequirementが未登録なら`proposed_destination_requirement`を使い、完成`FutureTargetRequirementCandidateV1`のhashをASCII `MIRAKAN_FUTURE_TARGET_REQUIREMENT_CANDIDATE_V1`と自己hashを除くclosed JCS bytesから計算してRef／隣接hashへ一致させる。このcandidateもauthority-freeであり、current Requirement、Qualification Policyまたは合格Evidenceとして使わない。`target_program_evidence_ref`は取得元／version／retrieved-at／artifact hash／licenseを持つEvidence、profile document candidateはexact content-addressed planning documentに限定し、URL文字列、vendor名、NDA本文の公開inline、`latest`を受理しない。

`target_kind_resolution[]`のkind集合はsource Future rowの`candidate_target_kinds[]`とset equality、kindは重複禁止、各Profile集合はnon-emptyとする。`current_profile` branchはsource Active Definitionのcurrent `TargetProfileRegistryV1` rowと完成Profileへ解決し、rowの`target_kind`／`surface_role`を使う。`proposed_destination_profile` branchは完成`FutureProposedTargetProfileV1`へ解決し、その`proposed_target_kind`／`proposed_surface_role`だけを使う。どちらもID文字列、path、OS名、Future説明からkind／roleを推測しない。解決したkindはentryの`candidate_target_kind`と一致させ、同じcurrent refまたはproposal refを複数kindへ分類せず、row外kindを追加しない。全entryのProfile-candidate unionは`target_profile_candidates[]`とset equalityにし、coarse kindから具体Profile候補への機械的かつ全量の対応を固定する。

`execution_surface_candidates[]`はHost上でTool／Buildを実行するFutureだけnon-emptyとし、それ以外はexact `[]`にする。各tupleのexecution Target candidateは`surface_role=execution_host | both`、artifact Target candidateは`artifact_runtime | both`へ解決し、artifactを作らないquery／proposalは後者を`null`にする。両Refを同じTargetへ圧縮せず、tupleに現れる全non-null candidateは`target_profile_candidates[]`へ含める。Prototype Receiptはこのcandidate refをEvidenceとして束縛できるが、`proposed_destination_profile`をactive `TargetProfileRefV1`またはProduction Receiptへ読み替えない。

`future.capability.managed-external-host-execution`の現行Dossierはtuple集合をrequired non-emptyとし、全entryを`current_profile` branchに限定する。各branchから解決したexact `TargetProfileRefV1`集合をThreat Model、Host／Transport・Provider／Tool・Schema／Evalの三Conformance subject、Prototype／Build Receiptのexecution／artifact tuple unionとset equalityにする。現行Future rowのkindは`desktop | headless_server`だけなのでmobile artifact tupleは不許可であり、mobile kindと必要prerequisiteを加えた新Portfolio revision、destination Target Profile migration、Target別Qualificationが成立するまでWindows Host Receiptをmobile artifactへ流用しない。flat Target集合のcross product、未列挙tupleの暗黙追加を拒否する。

公式仕様EvidenceはURL、document／release version、取得UTC、artifact hash、license、該当する原文節とMiraikanai側判断を分離する。外部製品のcurrent仕様をArchitecture定数へコピーせず、Promotion時にfreshnessを再検証する。二次記事、Model回答、vendor名だけを選定Evidenceにしない。

## 5. Common stages

### I0 — Non-claim and problem definition

- [ ] 対象User、game／workflow、明示的に含む範囲、含まない範囲、品質軸、candidate Targetを定義する。
- [ ] source rowの57-claim vocabulary内`excluded_current_product_claims[]`をUI／documentation negative fixtureへ結ぶ。
- [ ] 数値化できないscale、品質、人数、latency、costを`unresolved_blockers[]`へ残す。

**Exit:** problem statement、non-claim fixture、current Future Approval binding。

### I1 — Authority、data、threat、product scope

- [ ] source rowが要求する`product_scope | authority_model | data_model | threat_model` Decision candidateをOwner別に作る。
- [ ] failure、rollback、Save、determinism、privacy、moderation、anti-cheat、credential境界を該当範囲で閉じる。
- [ ] AI機能はquery／proposal／managed execution／runtime authorityを分離する。

**Exit:** required I1 decisionすべてapproved、またはentryは`planning_only`のまま停止。

### I2 — Target、provider、license、operations

- [ ] exact Target program／OS／device／server profileを選ぶ。`desktop`等のkindをShipping targetへ読み替えない。
- [ ] source Active DefinitionにないTargetは`FutureProposedTargetProfileV1(kind=add)`、既存Targetの意味を更新する場合はcurrent refを束縛した`kind=update`として固定する。proposalをcurrent Profileへ登録または対応済み表示しない。
- [ ] Dependency候補はexact release／artifact／hash／license／SBOM／security／update／撤退planを持つ。
- [ ] Target program、provider selection、licensing、operationsのrequired Decisionを閉じる。
- [ ] Console NDA情報はaccess-controlled artifactに置き、公開Registryへ複写しない。

**Exit:** required I2 decisionとresource／owner／facilityがapproved。費用・device・service capacity未固定なら停止。

### I3 — Isolated prototype

- [ ] current Engine／Project schemaから隔離したsandboxで最小prototypeを作る。
- [ ] positive fixtureと同数以上のsingle-cause negative fixtureを実行する。
- [ ] functional、performance、scale、security、content quality、Target deviceのうちsource rowが要求するqualification kindを測る。
- [ ] prototype artifactをActive package、Save、Capability Catalog、Releaseへ読込ませない。

**Exit:** typed prototype Receipt。成功でもActivationまたはpromotionを行わない。

### I4 — Promotion candidate

- [ ] Owner、Target、fallback、Requirement、Fixture、Work Package、Risk、Capability、Activation binding、Release policy候補を一つのdestination Active Definition ChangeSetへ作る。
- [ ] 全`proposed_destination_profile`をdestination `TargetProfileRegistryV1` rowへ一対一投影し、Target ID／Owner／version／kind／surface role／qualification Requirementをbyte equalityにする。`proposed_destination_requirement`は同じdestinationの`RequirementRegistryV1` rowとrow migration manifestへ一対一投影し、`current_requirement`はretained exact rowへ解決する。`current_profile`はdestinationでretained、明示update proposalはexpected source rowからexact replacementにする。
- [ ] source Future Approval、Dossier、Decision、prototype、Dependency、Threat Model、qualificationをpromotion input closureへ束縛する。
- [ ] destination 14 Registryの全row union、references、DAG、Target、fallback、policy、Activation bindingを検証する。
- [ ] `FutureToActivePromotionManifestV1`候補とsigned Product row migration manifestを作る。

**Exit:** destination Definition candidate。まだcurrentへpublishしない。

### I5 — Active Definition migration

順序を変えない。

1. **A2:** destination Architecture／Product bytes、Schema Catalog、Toolchain、source Future approvalを閉じたRebaseline subjectを固定する。
2. **B2:** `ControlPlaneRebaselineCoreV1`を作る。
3. **C2:** independent `ControlPlaneRebaselineApprovalV1`を得る。
4. **D2:** Rebaseline Envelopeを完成させる。
5. **T2:** Rebaseline Transactionを完成させる。ここではProduct currentをpublishしない。
6. **L2:** destination epochのControl Plane lifecycle sequence 1 `declared->complete`を完成させる。
7. **P2:** source Future approval ref／hashとfuture IDを持つProduct migration Decision、`ActiveProductDefinitionMigrationV1`、destination snapshotを一つのpublicationへ閉じ、expected source snapshotへsingle CASする。全Activationは`not_activated`、Gate／Riskはdefinition genesis、他WP headはnull／0とする。
8. commit済みmigration wrapperを指す`FutureToActivePromotionManifestV1`を署名する。

初回Bootstrap Approval、source operational Evidence、WP complete、Activation、Gate／Risk evaluationをcarry-forwardしない。選択的carry-forwardは対応Future capability自身が別migrationでActiveになるまで禁止する。

### I6 — Destination implementation and claim release

- [ ] destination Critical Path Coverage Manifestを再生成する。
- [ ] 新WP、dependencies、Target fixtureを通常lifecycleで実行する。
- [ ] promoted Capabilityのrequired／optional Target全行をsame Candidateで`production`へ上げる。
- [ ] Product Release Gate、critical Risk 0、fresh production Receiptを満たす。
- [ ] `FutureProductClaimReleaseV1`をsource rowのclaim subsetに対して発行する。

Promotionだけでclaimを解除しない。read-time `effective_release`はcurrent Definition、Activation、Candidate、Receipt freshness、Risk、Decision validityを毎回再評価し、drift時はfail closedでunreleasedへ戻す。

## 6. Dependency ordering

Future dependency DAGはProduct Planの`prerequisite_future_capability_refs[]`だけを正本とする。最低限、次を強制する。

- dedicated server／transport／replicationがActiveかつ必要Target productionになる前にco-op、rollback、large-session、live service、MMOをpromoteしない。
- rollback networkingがActive／productionになる前にlarge-session network shooterをpromoteしない。
- offline large world、dedicated server、live-service operationsが揃う前にMMO distributed worldをpromoteしない。
- production Terrain／Foliage／GIがActive／productionになる前にAAA photoreal renderingをpromoteしない。

Future IDの存在、Dossier completion、prototype successを前提Activeと読み替えない。self edge、missing ID、cycleを拒否する。

## 7. Track-specific minimum Gates

### Network／MMO／live service

- authoritative simulation、replication scope、join／leave、migration、rollback window、failure partition、anti-cheat、privacy、moderation、retention、incident responseを別契約へ分ける。
- player count、tick rate、region、SLO、data volumeはapproved Requirementとscale Receiptなしに宣言しない。
- internet-facing service、distributed cluster、production customer dataはMVP local runtimeの延長で実装しない。

### Open World／advanced simulation／AAA rendering

- World partition、streaming ownership、Save consistency、asset budget、LOD／occlusion、Terrain、Foliage、GIを一つの巨大Capabilityにしない。
- vehicle、ragdoll、crowd、motion warpingは独立provider／fallback／fixture／budgetを持つ。
- AAA／photorealは機能数でなくapproved content-quality rubric、blind comparison、Target frame／memory budgetで判定する。

### Console／Web／XR／desktop expansion

- exact target program、SDK、certification、signing、store、device lab、support windowをTarget Profileへ閉じる。
- Linux／macOSをCI hostの存在からProduct supportへ昇格しない。
- Project C++／ShaderはTargetごとのABI、compiler、sandbox、signing／Store policyを別途資格化する。

### FPS／local multiplayer／scripting

- FPSはGeneric Engine Coreへ依存させず、optional Shooter Genre Packの共有Ranged Combat／character-locomotion Featureを再利用し、first-person camera、aim、weapon presentation、comfort、Accessibility、performanceを独立Profile／fixtureにする。
- split screenはinput assignment、viewport、UI focus、audio listener、profile／SaveをTarget別に統合検証する。
- unrestricted scriptingはVM／language、sandbox、determinism、hot reload、debug、package、mod trustを閉じ、Engine C++内部書換えの代用にしない。

### AI asset／local inference／runtime generation

- Model family別Engine branchを作らない。Gemma、Kimi、Qwen、DeepSeek、Grok、GLM等は同じsigned Provider／Deployment／Model／Host profileで扱う。
- first-party local inferenceのpromotion prerequisiteは`capability.authoring.ai-core`であり、standard external MCP／Host conformanceを必須にしない。外部Clientからlocal runtimeを使う組合せはoptional integration fixtureとして別に資格化する。
- first-party local inferenceはruntime／loader artifact、Model Manifest、license Decision、sandbox、OS IPC／authenticated loopback、resource budget、Import／Schema／Tool Conformanceを必須にする。
- managed external Host executionはproposal-only external-agentとは別Capabilityであり、exact Host／Transport／Provider Runtime／managed deployment identity／Model／Tool／Authority、Host／Transport・Provider／Tool・Schema／Evalの三Conformance、全Case inventory、pre／post attestation、managed Source edit／Build、Engine Build Receipt、filesystem／process／network／credential budgetを一つのTarget別closureで資格化する。通常のMCP接続成功、proposal-only Receipt、別managed deploymentのReceiptをmanaged executionの証拠にしない。
- llama.cpp等を選ぶ場合も候補Adapterであり、exact release／artifact／license／security評価前に標準採用しない。built-in file／shell toolとMCP proxyは無効にし、ToolはMiraikanai Gatewayだけを通す。
- runtime generationはstructured dataのbounded operationから始め、Source、Shader、Asset binary、Policy、Engine codeのruntime mutationを許可しない。
- commercial asset generationはtraining／output license、provenance、quality rubric、human review、notice、takedownが閉じるまで商用品質を主張しない。

## 8. Research and official-source policy

各Dossierの外部事実はpromotion時点の公式primary sourceで検証する。

- Engine／platform／SDK: vendor公式documentation、release note、support lifecycle、program terms。
- Open standard: published specificationとofficial conformance suite。
- Library／runtime: official repository、signed release、artifact registry、license、security policy。
- Provider／Model: official API／model card／terms／license／privacy documentation。

本文書のURLや2026-07-23時点の調査結果を永続定数にしない。source change、terms change、artifact replacement、security advisoryでEvidenceをstaleにし、再調査する。

## 9. Verification matrix

| 対象 | Positive | Single-cause negative |
|---|---|---|
| Portfolio | Future25／Claim57／Policy2、3 Registry closure | missing／extra／duplicate、expired Approval、Head branch |
| Dossier | required decision kind set equality | omitted decision、wrong Owner、自由文字TBD、stale source closure |
| Dependency | acyclic prerequisite closure | self／missing／cycle、prototypeをActiveと偽装 |
| Prototype | isolated artifactとtyped Receipt | current schema／package／Saveへの混入、unsupported Target |
| Promotion | source Approval＋destination rows＋Rebaseline＋migration | Bootstrap再利用、partial row manifest、Approval前publish |
| Migration | full reset、only CP lifecycle complete | Evidence／Activation／WP carry-forward、multi-CAS |
| Claim | all promoted Target production、fresh Release、Risk0 | partial Target、stale Candidate、expired Evidence、free-text claim |
| External facts | official source、version、retrieved_at、artifact hash | secondary-only、alias追従、unverified license |

## 10. No-Go conditions

次の一件でもあれば対象Future entryを`planning_only`に留める。

- Product scope、Owner、authority、data、threat、Target、provider、license、operations、qualificationのrequired Decisionが未承認。
- fallback、negative fixture、rollback／exit plan、resource ownerが未確定。
- exact Target device／service／SDK／Dependency artifactを入手・検証できない。
- Active 14 Registryを一つのvalid destination closureにできない。
- Rebaseline、full-reset migration、current CASのいずれかを省略しようとしている。
- prototype、Marketing、LLM回答、競合Engineの存在だけを実現可能性Evidenceにしている。
- 全Asset自動生成、AAA品質、MMO scale等を測定可能Requirementなしに保証しようとしている。

## 11. Completion report

Dossier作業の報告は、各Future IDについて次だけを返す。

```text
future_capability_id
source_future_portfolio_definition_sha256
source_approval_sequence
current_stage = I0 | I1 | I2 | I3 | I4 | I5 | I6
approved_decisions[]
unresolved_blockers[]
prototype_receipts[]
promotion_state = not_proposed | candidate | migrated
effective_claims[]
next_owner_action
```

`current_stage`をcompletion percentageへ変換せず、未解決blocker、必要authority、次のDecisionを明示する。
