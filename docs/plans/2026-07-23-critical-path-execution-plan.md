# Critical Path Execution Plan

> Product Planのapproved Active Definitionを実装へ展開する実行計画。本書はWork Packageの完了、Capability Activation、Release、Shippingを自己承認しない。

> **Status: execution-order consumer, not execution authority.** 実行入口、official-source の採用境界、historical 文書の扱いは[計画書の実行権限・準備状況](README.md)に従う。Wave は優先順であり、current Product snapshot／Owner／Task Authorization による lifecycle state を置換しない。

## 1. 結論

初回Control Plane bootstrap後の実行対象はProduct Planの75 Work Package全件である。ただし一括queueにはしない。

- `wp.architecture.control-plane`: bootstrap Task 10Bで既に`complete`となる1件。
- unconditional: Definition上`declared`で、dependencyとGateを満たした時だけ開始候補になる70件。
- conditional deferred: Product Decision Gateを満たすまで開始禁止の4件。

この集合は`1 + 70 + 4 = 75`のdisjoint unionであり、missing／extra／duplicateを許さない。Control Plane construction TaskをProduct Work Packageへ混ぜず、WP IDとTask IDを別namespaceとして扱う。

## 2. Authorityとentry condition

正本は次だけである。

1. [Product Plan](../architecture/00-product/product-plan.md)のcurrent Active Product Definitionとcurrent operational snapshot。
2. [Control Plane Design](2026-07-22-architecture-evolution-control-plane-design.md)の`CurrentControlPlaneBaselineBindingV1`。
3. 各Owner文書のCapability／fixture／Receipt契約。
4. [AI Security／Approval](../architecture/01-governance/ai-security-approval.md)と[AI Verification／Provenance](../architecture/01-governance/ai-verification-provenance.md)のAuthorization／Evidence規則。

各run開始時に次をread-backする。

- current signed Product snapshot wrapperとsequence。
- `active_product_definition_sha256`、current baseline binding、Toolchain lock、revocation snapshot。
- 対象WPのcurrent lifecycle head、definition seed binding、dependencies、Target、fallback、provided fixture、required capabilities。
- 必要なPhase／Decision Gate、Evidence freshness、same Candidate binding。

初回`ControlPlaneBootstrapApprovalV1`をcurrent authorityとして直接参照しない。current bindingが`bootstrap | rebaseline`のどちらでも同じ入口を使い、stale snapshot、filesystem `latest`、Markdown checkmarkからstateを推測しない。

## 3. Coverage manifest V2

実行前に次のsigned manifestを生成する。

```text
CriticalPathCoverageManifestV2
  manifest_id
  active_product_definition_sha256
  current_control_plane_baseline_binding: CurrentControlPlaneBaselineBindingV1
  source_product_snapshot_ref, source_product_snapshot_sha256, source_sequence
  critical_path_plan_ref, critical_path_plan_sha256
  bootstrap_completed_work_package_ref = wp.architecture.control-plane
  unconditional_work_package_refs[70]
  conditional_deferred_bindings[4]:
    {work_package_id, decision_gate_id, required_transition_policy_id,
     initial_state = deferred}
  generated_at, revocation_snapshot_ref
```

`manifest_id`は同Fieldを除くpayload JCS hash由来とする。三集合はWork Package Registryの75 IDとset equality、互いにdisjoint、UTF-8 byte順、重複なしでなければならない。

conditional exact setは次の4件だけである。

| Work Package | Decision Gate | Transition policy |
|---|---|---|
| `wp.foundation.cpp23-cx2-cutover` | `gate.product.reconsider-cpp23-cx2-cutover` | `policy.product.wp.defer-release.v1` |
| `wp.foundation.cpp23-cx3-shipping` | `gate.product.reconsider-cpp23-cx3-shipping` | `policy.product.wp.defer-release.v1` |
| `wp.product.general-coverage-3d` | `gate.product.reconsider-c2-3d` | `policy.product.wp.defer-release.v1` |
| `wp.product.production-release-binding` | `gate.product.reconsider-production-release-binding` | `policy.product.wp.defer-release.v1` |

Manifestはexecution coverageの証明であってlifecycle stateを変更しない。

## 4. Scheduler state machine

### 4.1 通常WP

許可される実行順は次だけである。

```text
Definition scheduling_state=declared
  -> lifecycle declared->ready
  -> ready->active
  -> active->complete | active->blocked
  -> blocked->ready
```

各edgeはProduct Planのexact lifecycle policy、expected previous head、Owner、Task Authorization、fresh Evidence、same Candidateを満たす`WorkPackageLifecycleRecordV1`一件で行う。snapshot nextはそのwrapperだけを`applied_change`へ持ち、一回CASする。複数WPを一つのtransitionへまとめない。

`declared->ready`条件:

- 全`requires_work_package_refs[]`がcurrent snapshotで`complete`。
- required Target／Toolchain／runner／device capacityがqualified。
- fallbackが選択可能か、fail-closed unavailableが明示されている。
- Definition seedはcurrent WP row／plan／task／fixture／dependency closureと一致する。

`ready->active`条件:

- exact task specification、input closure、resource budget、timeout、rollback point、Authorization Envelopeがある。
- AIまたは外部Hostを使う場合もeffective authorityが必要Operationの積集合内で、Approval／Commit／Activationを含まない。

`active->complete`条件:

- required task、test、negative fixture、Target qualification、provided fixtureの完成Receiptがある。
- Owner acceptanceと同一Candidate／Definition／Toolchain／Targetへ閉じる。
- Capability Activationは別Product transitionであり、WP completeから自動推測しない。

baseline-scopedなRepository bytesを変更するWPは、次の閉包を追加で必須とする。

```text
current bindingでWP active
  -> isolated Stagingでcandidate treeを作成
  -> candidate validation／preliminary Evidence
  -> affected document／codeのhuman approval
  -> same-definition Rebaseline A2->B2->C2->D2->T2->L2->P2
     （当該WP activeを含む非Control-Plane stateをbyte-exact保持）
  -> new current bindingへfinal Receiptを発行してread-back
  -> Owner acceptance
  -> active->complete
```

pre-rebaseline Evidenceは`preliminary`であり、WP completion、後続WP entry、Capability／Release Gateへ使用しない。subject Git tree内のartifactへcurrent binding、Envelope、Product snapshotを埋め込むとtree hashとbindingが循環するため禁止する。tree内artifactは安定したartifact bytes／hashだけを持ち、そのhashとnew current bindingの結合はGit tree外append-only Evidence storeのfinal Receiptだけが所有する。Repository bytesがparent treeとbyte-equalなWPはRebaselineを捏造せず、current bindingへ直接final Receiptを束縛する。

### 4.2 Deferred WP

deferred WPはGateがread-time `effective_state=satisfied`となっても自動遷移しない。exact `on_satisfied_action`、fresh R4 Product Decision、current parent、`policy.product.wp.defer-release.v1`を持つlifecycle Recordで`deferred->declared`を一度CASする。その後は通常WPと同じstate machineを使う。

Gate Evidenceが失効／revoked／driftした後は新しいdefer-release Recordを発行しない。既に正当にcommit済みの履歴を巻き戻さず、必要ならOwnerが通常blocked transitionを行う。

## 5. Execution waves

Waveは優先順であり、正本dependency DAGを上書きしない。同じWave内でもdependencyとGateを満たしたnodeだけを実行する。

### Wave 0 — Control Plane handoff

- [ ] current snapshot sequence 1以上をread-backする。
- [ ] `wp.architecture.control-plane` lifecycle sequence 1 `complete`とdefinition seedを検証する。
- [ ] Coverage Manifest V2を発行する。
- [ ] Control Plane Task 6 がpublication前に materialize したapproved Architecture Document Registry、Architecture Dependency Source Set／Edge Registry compiler、append-only Conformance Record Store、独立`role.architecture-dependency-conformance-validator` Signer／Key／revocation closureを current snapshot からread-backする。Owner 103、Source／Edge 516、production graph cycle 0、Core→Shooter reachability 0と全negative fixtureを**read-onlyで再計算**し、同Candidate／Product／Foundation／baseline／compositionを束縛した`ArchitectureDependencyConformanceReportV1(result.kind=pass)`をsidecarへ発行する。Report は completed `wp.architecture.control-plane`、Product snapshot、Operation Authorityを変更せず、Wave 1 のreadiness evidenceだけに使う。

### Wave 1 — Foundation／Headless authoring

- [ ] Generic Core baseline exact 6とcurrent Installed Product composition exact 10の定義、MCD／Owner contribution／Service allowlist／Trust／Receipt publication実装を準備する。Project State Origin六Operationが同じCandidateでTechnical Qualificationを満たした後、`activation.foundation.operation_domain_receipt_pipeline.v1`でbaseline六のSigner Policy／Role／assignment／Key／Service authority／Publication Control closureだけを一括materializeし、currentを`contract-active 10 / operational 6`へatomic遷移する。World一＋Shooter三のextension四はSigner Policyへ追加せず`contract-active / not operational`を維持し、baseline六の部分Activationを拒否する。
- [ ] CX0 Toolchain、Math／Memory、Runtime Scheduling、ECS **E0 contract baseline**を順に閉じる。E0のentryはRuntime ECS E0 Planのcurrent baseline／active WP条件で判定し、新ECS technical documentのapprovalをE0開始前に推測して要求しない。
- [ ] E0 Task 11のhuman approval、same-definition Rebaseline、new current binding、final Receiptが揃った後だけE1–E3を順に開始する。E0 draft、preliminary Evidence、plan checkboxをE1 entryやCapability Activationへ読み替えない。
- [ ] Project State、Asset Save、Headless Coreを閉じる。
- [ ] headless determinismとauthoring transaction Gateを同一Candidateで通す。

### Wave 2 — Windows Editor／Runtime shell

- [ ] ECS E4、Render Graph、D3D12、Input／Audio／UI portable coreとWindows adapterを閉じる。
- [ ] Windows package／Editor runtimeを閉じ、empty scene fixtureを物理Target profileで通す。

### Wave 3 — Manual 2D first playable

- [ ] Gameplay Core、Runtime Timer、2D World／Camera、Collision／Physics／Animation、Navigation CoreをFeature／Genre required edgeなしで閉じる。
- [ ] `fixture.product.genreless-core-2d`で全optional Feature／Genre Pack uninstall、zero-character／noncombat holdoutを実行し、同じCandidate／Product／Foundation／baseline／compositionのfresh `ArchitectureDependencyConformanceReportV1(result.kind=pass)`でCore→Feature／Genre required edgeおよびCore→Shooter production reachability 0を検証する。
- [ ] ECS E5 2DをCore subsystemだけで閉じた後、Reusable Feature PackとShooter Genre Pack／2D Reference Gameをconsumerとして統合する。
- [ ] Generic Core六、World core extension一、Shooter Genre三の全十OperationをWave 3の同じCandidateでfresh再Qualificationし、Wave 1 baseline六のSigner／Trust／Policy rowをbyte-exactに保持したうえで、`activation.installed_product.operation_composition_extensions.v1`によりWorld一＋Shooter三のextension四だけをSigner Policy／Role／assignment／Keyへ一括追加する。完了後のInstalled Product operational snapshotとSigner destinationをexact十件でset equalityにし、Wave 1 CandidateとのEvidence混在、extension一～三件だけの部分追加、baseline row再発行、Shooter未qualified entryを拒否する。
- [ ] Title→Result、Save／Load／Replayを含むmanual 2D Gateを通す。

ShooterはoptionalなShooter Genre Packと、それをconsumerとして使う2D／TPS Reference Gameである。Engine CoreをShooter仕様へ固定せず、Genre固有型をCore公開契約へ入れない。後続C2ではPlatformerとPuzzle／Dialogueを独立したGenre／Project fixtureとして使う。

### Wave 4 — AI-native authoring MVP-A

- [ ] exact八family、`planning.operation_family.authoring_discovery`、`planning.operation_family.game_system_discovery`、`planning.operation_family.world_discovery`、`planning.operation_family.gameplay_definition_authoring`、`planning.operation_family.asset_authoring`、`planning.operation_family.authoring_changeset_execution`、`planning.operation_family.build_candidate_test`、`planning.operation_family.build_device_play_debug_task`を各family単位のMCD／Manifest／Service／Policy／Validator／Diagnostic／Receipt／Signer／Trust／Provider closureでatomic activateする。列挙集合とのset equalityを必須とし、略称、prefix、family count、部分ActivationをWave失敗にする。
- [ ] AI Core、Debug／Replay、ECS E6、prequalified Source Pack、Windows E7 2Dを閉じ、Product `ExecutionSurfaceBindingV1`が必要family全件をcurrent `operational`へ再解決できることを確認する。
- [ ] Engine Provider Adapterのexact Engine Host＋Provider Runtime／Manifest＋Inference Deployment＋Model＋Tool Projection＋proposal-only Authority Profileをmaterializeし、Provider／Tool ConformanceとSchema／Eval Conformanceを同じsubject tuple／Target／Case inventoryでpassさせる。
- [ ] typed Query→Plan→ChangeSet Proposal→Validate→Preview→Human Approval→trusted-internal Commit→Candidate Validate／Cook／Build／Test→Package／Launch／Debug→修正Proposalの一方向chainを通す。
- [ ] MVP-A Thin Slice は別allowlistや別Product stateを作らない。上記exact八familyのatomic Activationと`AiCapabilityActivationMatrixV1`／`ExecutionSurfaceBindingV1`のcurrent operational read-back**後**に、固定したprompt／manual edit／human Approval／trusted-internal Commit／play oracleの同一Candidate witnessを発行する。八familyの一部だけをThin Slice名義でactivateすることを拒否する。
- [ ] `fixture.product.genreless-ai-project`のzero-character／noncombat／WorldなしUI／logic branchとShooter 2D branchを同じOperation／Validation／Debug／Build DAGで別々に完走する。
- [ ] AI／manual roundtrip、MVP completion、Title→Result、Save／Load／ReplayをGateで明示評価する。

### Wave 5 — External agent and Project source lane

- [ ] standard external MCPの少なくとも一つのexact Host＋Transport＋Tool Projection＋proposal-only Authority Profile＋MCP Grantをmaterializeし、Host／Transport ConformanceとSchema／Eval Conformanceを同じsubject tuple／Target／Case inventoryでpassさせる。Provider／Deployment／Model／Provider-Tool Receiptはnull／unattestedに維持する。
- [ ] current materialized Hostが0件の間は外部Client supportを主張しない。
- [ ] Wave 4でActivation済みのChangeSet 4／Candidate Build・Test 6 closureをexact read-backし、未ActivationのNative Source 5、Project Shader 16、shared Source Promotion 1だけをfamily単位でatomic activateする。Native Game Module／Project ShaderをEngine本体外のProject Source StagingでTask→Worker→Broker再計算Diff→Build／Test→独立Review→Code Owner Approval→trusted-internal Promotion→Project ChangeSet登録できるlaneを閉じる。
- [ ] standard external MCPのExternal Tool CatalogはQuery／Plan／Proposal／Validate／Previewだけを含み、外部からのBuild／Test／Debug希望はtyped Proposalとして記録する。実行は人間承認後にEngine-owned Orchestratorが別Task Authorizationで行い、外部HostはQueryでReceiptを読む。`operation.build.*`、`operation.test.*`、`operation.debug.*`、`operation.authoring.changeset.commit`、`operation.project_source.promote_revision`、Approval、Activation、Signing、Releaseの外部projectionをexact 0件にする。
- [ ] managed external Host edit／buildは`future.capability.managed-external-host-execution`、first-party local inferenceは`future.capability.first-party-local-inference`として別々に`planning_only`を維持し、通常のproposal-only Host conformanceへ混ぜない。

### Wave 6 — Manual 3D MVP-B

- [ ] 3D World／Camera、Collision／Physics／Animation、ECS E5／E7 Windows 3D、Shooter 3Dを閉じる。
- [ ] Title→Result、Save／Load／Replayを含む3D first playable Gateを通す。
- [ ] これはC1 3D fixtureであり、汎用Production 3DやFPSを意味しない。

### Wave 7 — Mobile

- [ ] Vulkan／Metal、mobile lifecycle／offline、Android／Apple I/O／UI adapter、E7 2D／3Dを閉じる。
- [ ] package、clean install、offline launch、resume、Save／Load／Replay、Target別input／UIを実機で検証する。
- [ ] AndroidとApple、2Dと3DのReceiptを相互流用しない。

### Wave 8 — General 2D Production capability

- [ ] `feature_authoring`、`camera_authoring`、`material_authoring`、`vfx_authoring`、`environment_authoring`、`input_binding_selection`、`navigation_binding_selection`、`physics_role_selection`のexact 8 familyを、各MCD／Manifest／Service／Policy／Validator／Diagnostic／Receipt／Signer／Trust／Provider closureとProduct `ExecutionSurfaceBindingV1` read-backを含むatomic transactionでactivateする。8件の一つでも`operational`でなければGeneral 2Dをqualifiedへ進めない。
- [ ] Platformer、Puzzle／Dialogue、Environment、VFX、Material、native UIのC2 provider WPを閉じる。
- [ ] Shooter、Platformer、Puzzle／Dialogueの三fixtureをWindows／Android／Appleで同一Candidateへ閉じる。
- [ ] CX2／CX3とProduction release bindingはconditional templateに従い、未解除ならPhase 8のdeferred行として残す。

### Wave 9 — Runtime generation deny boundary

- [ ] Runtime structured-data generationが未Activationの状態で、RuntimeからSource／Shader／Asset／Policy／binaryを生成・activateできないnegative fixtureを通す。
- [ ] positive runtime generationを実装済みと扱わない。Future promotion後だけ別Definitionで扱う。

## 6. Conditional templates

### C1 — CX2 cutover

1. Decision Gateが要求する5 Targetのnonexperimental `import std`、module DAG、Toolchain lock、cross-target cutover Evidenceを作る。
2. Gateをread-time再評価する。
3. R4 Product Decisionで`deferred->declared`を一回CASする。
4. CX2 WPを通常state machineで実行し、5 Targetの専用fixtureを完了する。
5. CX0 fallbackを削除またはCX3を開始する権限にはしない。

### C2 — CX3 shipping language mode

1. CX2 WP complete、正式stable language mode、5 Target release readiness、package／install／offline／rollback Evidenceを要求する。
2. Gate satisfied＋Product Decisionでだけ宣言する。
3. CX3 WP完了後もRelease Gateとcritical Risk closureなしにShippingを許可しない。

### C3 — General Production 3D

1. source DefinitionではPhase 6 Evidenceをread-backするが、未完成definition-change classを捏造してGateを`satisfied`にしない。
2. 非Shooter第二3D fixture、Requirement、Owner WP、Target binding、fallbackを含むdestination Active 14 Registry、signed row migration manifest、Control Plane Rebaseline candidateを作る。
3. A2→B2→C2→D2→T2→L2→P2でV1 full-reset migrationを一回CASする。完成`ActiveProductDefinitionMigrationV1`だけがdestinationのapproved definition-change class Evidenceになる。
4. destinationではsource lifecycle／Activation／Gate／Risk Evidenceをcarry-forwardせず、Control Plane WP以外をgenesisから再実行し、Phase 6を同じdestination Definition／Candidateへfresh re-Qualificationする。
5. destination currentで`gate.product.reconsider-c2-3d`をfresh Phase 6 Evidence＋完成definition-change classから再評価し、別R4 Product Decision後だけ`wp.product.general-coverage-3d`を`deferred->declared`へ進める。
6. Wave 4／8でoperational化したfamilyをexact read-backし、General Production 3Dだけが追加要求する`rendering_aa_discovery`、`lighting_discovery`、`post_process_discovery`、`math_semantic_authoring`、`lod_authoring`のexact五familyをSigner／Trust／Provider／Target qualificationと同じatomic closureでactivateする。五件の一つでも非operationalならGeneral 3Dをqualifiedへ進めない。

source DefinitionのGateはdefinition-change class不在のためeffective failであり、source内のunavailable 3D activation 3行を昇格しない。destination migration publicationとdeferred-release publicationを同じCASへまとめない。

### C4 — Production release binding

1. CX3とGeneral 2D WP complete、Phase 0／2／7／8 Gate、release policy／artifact plan Evidenceを要求する。
2. Product Plan OwnerとApplication Package／Release Ownerが、completed prerequisite WPのcurrent Receiptからsupport／rollback／signing／SBOM-provenance policy Evidenceとartifact／device-lab／store plan Evidenceを同WP開始前に独立発行する。不足時はsource Gateをblockedに保つ。source currentのActivation Binding全293行が`disabled_current_definition`／empty gate exact setであることからmode=`production_release_migration_authoring`を導出し、Gate satisfied＋Product Decisionで`wp.product.production-release-binding`を宣言する。同WPは上記Evidence ref／hashをread-backしてTarget別projectionとdestination Definition candidateへ組み込むだけで、Evidenceを自己生成しない。source WPは`active`までで、`complete`／Artifact Candidate／Release Evidence化を禁止する。
3. destinationでは全293 Activation bindingの`production_policy`を`disabled_current_definition`からTarget一致の`all_release_gates`へ変更し、Product Release Gateに変更があれば同じsigned row migration manifestへ含める。このDefinition変更をoptionalにしない。
4. A2→B2→C2→D2→T2→L2→P2でV1 full-reset migrationを一回CASし、source WP completion／Activation／Gate／Risk Evidenceをcarry-forwardしない。
5. destination currentのActivation Binding全293行が各Target exact一件の`all_release_gates`であることからmode=`production_release_binding_qualification`を導出する。Critical Path prerequisiteをgenesisから再実行し、Gate satisfied＋別Product Decisionで同WPを再宣言する。このmodeではDefinition／Registry／migration artifact変更を禁止し、同じCandidateへfresh Target／package／support／rollback／signing／SBOM-provenance／device-lab Evidenceを閉じ、独立Owner acceptance付きArtifact CandidateでだけWPをcompleteする。
6. 5 Release GateはPhase 9 deny Gate、destination currentのrequired WP、same Candidate、fresh Target Evidence、critical Risk predicateを毎回再評価し、成功後だけTarget別`qualified->production`とhuman Release Approvalを別transitionで行う。

## 7. AI execution policy

必要Operation familyのatomic ActivationとProduct `ExecutionSurfaceBindingV1`のcurrent operational判定が成立した後だけ、AIは次を要求できる。current planning stateでは24 family／191候補が全て`not_activated`であり、以下をcallableと表示しない。

- bounded Architecture／Product Query。
- task specificationとChangeSet proposal。
- Staging内のProject data編集、許可されたSource task request、Build／Test／Debug Operation。
- Receipt収集と差分説明。

AIはApproval発行、Commit実行、Source Promotion、Activation、Signing、Release、Policy overrideを行わない。人間Approval後のCommit／PromotionはEngine-owned trusted internal Gatewayが実行し、AIは結果Receiptを読み次のPlanを提案できる。Engine repositoryまたはcurrent Project storageへの外部Client direct writeは公式boundary外であり、External patchとして全検証へ戻す。

Model／Host対応はfamily別branchを作らず、signed profile tupleとConformanceで決める。Gemma、Kimi、Qwen、DeepSeek、Grok、GLM等は同じprovider／local profile契約へ入る。当該routeのProfileまたは必須Receiptが一件でも不在なら、そのrouteは必ず`not_activated`である。`proposal_only`にできるのは、別のcurrent standard MCP routeがproposal-only Authorityを含むfull tupleとexact二Receipt（`HostTransportConformanceReceiptV1`、`SchemaEvalConformanceReceiptV1`）を持ち、そのtupleから新しいCaller Contextを発行できる場合だけであり、不成立routeの自動降格ではない。

## 8. Required verification per WP

各WPのTask planは最低限次を持つ。

```text
WorkPackageTaskPlanV1
  work_package_id
  active_product_definition_sha256
  execution_mode = standard | production_release_migration_authoring | production_release_binding_qualification
  current_baseline_binding
  definition_seed_binding
  source_snapshot_ref, source_sequence
  exact_inputs[]
  expected_outputs[]
  positive_tests[]
  single_cause_negative_tests[]
  target_qualification_refs[]
  budget_and_timeout_profile_ref
  rollback_boundary_ref
  authorization_envelope_hash
```

`execution_mode=standard`は`wp.product.production-release-binding`以外の74 WPだけに、同WPはProduct Planの全293行判定から導出した二modeだけに使う。呼出元指定、自由文字列、source／destinationでのmode取り違えを拒否する。完了Receiptは計画、source、output、Toolchain、Target、Candidate、test結果をexactに束縛する。ただしmigration-authoring modeは完了Receiptを発行せずmigrationでepochを置換する。test skip、assertion緩和、budget引上げ、別Target Receipt、古いProfile、説明文だけのpassを拒否する。

### 8.1 Generic contract implementation handoff

Wave 1～5でMCD compiler、Gateway、Project mutation、Pack／AI surfaceを実装するTaskは、Architecture正本のcurrent contractを次の順で実装する。

1. `ContractSetSnapshotV2`のclosed local-member unionへMCD、Diagnostic、Trusted Service、Validator、Operation Validator Closureを収録し、全cross-member edgeをLocalRefで解決してからset rootと外部refをmaterializeする。
2. 全named Operation inputは`MIRAKAN_OPERATION_INTENT_V2 -> MutationAuthorizationBindingV2 -> MIRAKAN_OPERATION_REQUEST_V2`の一方向DAGを使う。Approval／Predelegationをfinal request hashへbindせず、未Activationだった循環shapeのcompat readerを作らない。
3. state-changing Operationは[Executable Contracts §8](../architecture/02-foundation/executable-contracts.md#8-operation定義)を唯一の正本として、`Prepared → staged postcondition → private durable commit marker read-back → secret-free PublicCommitClosureV1 candidate → canonical signed wrapper → PublicCommitClosureV1＋PublicPublicationMarkerV1＋after stateのatomic public CAS`だけを許可する。Project ChangeSet CommitはClosureの`domain_commitment.kind=project_change_set_commit`にexact ChangeSet ref／Candidate rootを必須化し、Source primitive 0件でもtyped branchを省略しない。Closureまたはsigned ReceiptのないProject／Registry／Runtime current headを公開せず、nested common Closure schemaを新しいMCD Operation／Type／Contract Set memberとして数えない。
4. current MCD／Manifest／Service allowlistへ入れるのは全Field、named I/O／Receipt、Policy、Diagnostic、Validator、rate／timeout、crash recovery、Qualificationが閉じたOperationだけとする。Wave 4／5が必要とするAI E2E familyは各activation work itemのcomplete setでだけactiveへ移し、その他のScenario／Stage authoring、generic Pack AI、Feature authoring、Physics／Navigation／Input selectionは該当Waveまでcurrent `not_activated`を維持する。名前、Schema、Provider aliasだけを復活させない。
5. Production Source／Recipe／Registry／Runtime PackageはFixture bodyを解決しない。owner-typed Qualification recordがFixture集合を所有し、Productionはexact signed Qualification Receiptだけを検証する。

このhandoffは実装順序であり、文書内schemaの存在、WP checkbox、Staging artifactをEngine実装完了、Capability Activation、Target Qualification、Release／Shipping Evidenceへ読み替えない。

## 9. Release／Shipping判定

現Definitionの初期状態はShipping No-Goである。理由は少なくとも次である。

- 4 conditional WPがdeferred。
- Activation 293行がgenesisで`not_activated`。
- AI E2Eを含むplanning 24 family／191 Operation候補が全て`not_activated`で、ChangeSet／Build／Debug／Sourceの公開実行surfaceは0件。
- Product Planのcritical Riskがopenで、Release predicateは`mitigated | closed`だけを許す。
- Product Release GateのTarget Receipt、support／rollback／signing closureが未作成。

Shipping Goは5 Release Gateそれぞれのread-time successと、該当Targetのproduction Activation、全critical Risk closure、human Release Approvalが同時にcurrentな場合だけTarget単位で出せる。技術デモ、MVP-A、MVP-B、Technology PreviewをShippingと読み替えない。

## 10. Progress and restart rules

- Progressはcurrent signed snapshotから再計算し、表のcheckboxをstate authorityにしない。
- worker crash後は同じtask ID／input hashでReceiptをread-backし、terminal taskを再実行しない。
- stale parent CASは失敗し、new currentからentry conditionを再評価する。
- Definition migration後は古いCoverage Manifestを無効化し、destination DefinitionでV2を再発行する。
- same-definition Rebaseline後もCoverageのbaseline bindingがstaleになるため再発行するが、Product Definitionの75 ID集合は変えない。

## 11. Plan completion Gate

この計画書自体の監査完了は次で判定する。

- [ ] Product WP 75のcoverageが1／70／4へexact partitionされる。
- [ ] dependency DAGにmissing／cycle／orphanがない。
- [ ] deferred 4件のGate／action／policyがProduct Planと一致する。
- [ ] Active/Future、Definition/state、WP completion/Activation、MVP/Shippingを混同していない。
- [ ] current baseline binding、rebaseline、full-reset migrationのrestart規則がある。
- [ ] baseline-scoped Repository変更WPがcandidate validation→same-definition Rebaseline→new-binding final Receipt→Owner acceptanceの順で閉じ、tree内artifactへbindingを埋め込んでいない。
- [ ] official external Host／local AIをConformance前に対応済みと主張していない。
