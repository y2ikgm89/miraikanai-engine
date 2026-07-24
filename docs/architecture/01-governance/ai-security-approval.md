# Miraikanai Engine AI Security／Approval Contract

- 文書ID: mirakan.arch.ai-security-approval
- 状態: review
- 正本範囲: AI task authorization、Risk、Trust boundary、不変Engine、Sandbox、Credential、Provider／MCP／CLI security、Preview、人間承認、Activation、Promotion、拒否
- 非正本範囲: Eval、Evidence envelope、Provenance、Trace grading、Receipt保持。これらはAI Verification／Provenanceを参照する
- 依存: [Product Plan](../00-product/product-plan.md)、[AI Verification／Provenance](ai-verification-provenance.md)、[Core architecture](../02-foundation/core-architecture.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Project state](../03-authoring/project-state.md)、[Native game module](../03-authoring/native-game-module.md)、[Project Shader](../06-rendering/project-shader.md)
- 外部根拠検証日: 2026-07-23

## 1. 結論、優先順位、用語

Miraikanai EngineはHybrid High-Assurance AI Developmentを採用する。次の三層を分離する。

1. 人間とAIが読む意図、要件、説明。
2. Engineが検証する機械可読契約、変更Proposal、Test、Budget。
3. 人間または信頼済みAuthorityだけが発行できるAuthorization、Approval、Promotion、Activation、Release権限。

AIはGame制作とEngine製品開発で実装Proposalを作れるが、自分で権限を宣言し、自分の出力を検証済みとみなし、Project revision、正規Git履歴、署名鍵、Storeを直接変更してはならない。

Game制作とEngine製品開発は別Profile、別Repository、別Authorizationである。GameAuthoringProfileV1ではEngine baselineを不変にし、Engine source、Extension、Adapter、Validator、Policyの変更Toolを公開しない。Engine製品開発のR4／A2をGame制作へ継承してはならない。

### 1.1 規範の優先順位

矛盾時は上位を優先する。

1. 法令、Platform／Store強制要件、Security incident response。
2. 署名済みTaskAuthorizationEnvelopeと機械可読Policy。
3. Executable contract、Authoring、Runtime、Foundation、Platformの正本。
4. 本文書。
5. Task固有の認証済み人間指示。
6. AGENTS.md、Prompt、Skill、Provider説明。
7. AIの推測。

下位層は上位の権限、禁止、Budget、Gate、Approvalを緩和できない。Prompt、AGENTS.md、MCP annotation、人間のProject ApprovalはSecurity boundaryではない。

### 1.2 主要用語

| 用語 | 定義 |
|---|---|
| Proposal | 正規状態を変えないStaging上の変更候補 |
| Authorization | exact Task、入力、Operation、Path、Network、Gate、Approvalを許可する署名済みEnvelope |
| Technical qualification | 固定Policyが要求するGateの全合格をPolicy Serviceが照合すること |
| Human gameplay approval | 人間が意図、体験、Capability scope、Preview、制限、Rollbackを確認すること |
| Activation | exact Candidate rootをcurrent pointerへ原子的に切り替えること |
| Promotion | 検証・承認済みProposalをAuthoritative revisionまたはSource commitへ昇格すること |
| ImmutableEngineBaselineV1 | Engine、Editor、GameHost、SDK、Validator、Policy、Engine-owned Testのversionとhashを固定する署名済みManifest |
| Bounded Native | 公開SDKと許可C++ subsetだけを使うProject固有NativeGameModule |
| Bounded Project Shader | `BoundedProjectShaderProfileV1`とEngine-owned Port内で、semantic Moduleまたはdeclarative Techniqueとしてoffline compileするProject Shader |
| capability_unavailable | 公開Capabilityで実現不能であり、Game制作TaskからEngine変更へ進まず停止した状態 |

## 2. ActorとTrust boundary

| Actor | Trust | 許可する役割 | 禁止 |
|---|---|---|---|
| Human Author | 認証済み主体 | 要件、Gameplay approval、手動Proposal | Policy外Token作成、技術Gate代替 |
| Code Owner | Qualification済みの認証済み主体 | 割当ScopeのNative／Shader exact Diff review、`CodeOwnerApprovalV1`判断 | Gameplay approval代替、自己割当、Scope外承認、Gate／Policy代替 |
| Editor Shell | 信頼済みClient | Proposal、Diff、Approval要求、typed Command | Validator迂回書込 |
| Policy Service | 信頼済みAuthority | Risk分類、Policy解決、Envelope／Attestation署名 | Artifact生成、自己Approval |
| Approval Service | 信頼済みAuthority | 人間認証、exact CandidateへのApproval署名 | 判断代行、Artifact変更 |
| AI Orchestrator | 制限Service | Task分解、Provider調停、Proposal組立 | 自己承認、Secret保持、main／current Candidate変更 |
| Provider Adapter | Secret分離Service | Credential使用、Provider request、metadata採取 | Project書込、CredentialをContext／Logへ出力 |
| Model Provider | 非信頼外部処理 | 構造化Proposalと説明 | 権限決定、最終検証、正規書込 |
| External AI Client | 非信頼Client | Query、Proposal提出 | Commit、activate、merge、sign、release |
| Source Worker | 隔離された非信頼実行系 | 許可Sourceの編集、Build、Test | 許可外Path／Network／Credential、Git ref操作 |
| C++ Command Gateway | 信頼済みAuthority | Authoring validation、Transaction、Project revision | 未承認Proposal適用 |
| Source Promotion Service | 信頼済みAuthority | Patch、Gate、Approval照合、Git commit | AI自己申告による昇格 |
| Activation Service | 信頼済みAuthority | Attestation／Approval coverage照合、Candidate pointer切替 | Artifact生成、無承認Activation |
| Release Coordinator | 信頼済みControl plane | R5証拠照合、分離Service起動 | Source実行、署名鍵／Store Credential保持 |
| Release Build Worker | 隔離された非信頼実行系 | clean Build、検査、unsigned artifact | Secret、正規Repository write |
| Platform Signing Service | 最高権限Service | 承認済みbyteの固定検査と署名 | Source／Build script／任意shell／Store upload |
| Store Upload Service | 最高権限Service | 承認済みsigned artifactの提出 | Source、Build script、Signing key |

AIが生成したText、Tool argument、Patch、Test要約、Risk自己申告は非信頼Inputである。合否には信頼済みRunnerが採取した終了状態とArtifact hashだけを使う。

Threat ModelはModelが規則に従うことを前提にしない。malformed argument、Prompt injection、Path escape、任意Code execution、resource exhaustion、malicious endpoint、stale／replay／差替え、Provider refusal、Schema drift、Runner crashをin-scopeにする。

OS kernel／hypervisor、TPM、信頼済みService binary、Policy／Approval／Release秘密鍵、組織Identity Provider、Repository administratorの侵害後まで保証しない。侵害兆候では未完了Envelope失効、Key revocation、Artifact隔離、clean environment再構築を行う。

## 3. Task authorization

### 3.1 TaskSpecificationとTaskAuthorizationEnvelope

TaskSpecificationはAIへ提示できる作業内容であり、権限を与えない。最低限、次を固定する。

    task_id, spec_revision, task_kind, goal
    success_criteria[], non_goals[]
    input_revisions[{artifact_id, revision, sha256}]
    target_profiles[]
    cxx_frontend_profile_id
    build_driver_profiles[]
    cpp_dependency_sets[]
    requested_outputs[], open_questions[]
    context_plan_id, context_plan_sha256
    context_pack_id, context_pack_sha256

goalは一つの検証可能な結果、success_criteriaは一件以上のRequirement IDとする。Blocking questionが残る間は変更実行を開始しない。C++／Buildを伴うTaskは該当Profileと全Dependency hashを固定する。

TaskAuthorizationEnvelopeはPolicy Serviceだけが発行し、AIが作成、変更、再署名してはならない。

```text
OperationMutationTypedScopeRefV1
  scope_type_ref: McdContractRefV1(kind=type)
  scope_artifact_ref: ArtifactRefV1

ProjectMutationScopeGrantV1
  base_project_ref:
    exact {project_id, project_revision, document_set_hash}
  max_revision_advance: uint32

TaskAuthorizationSubjectScopeGrantV1
  project:
    project_scope: ProjectMutationScopeGrantV1
    allowed_subject_scope_refs[0..1024]:
      OperationMutationTypedScopeRefV1
  | non_project:
    allowed_typed_scope_refs[1..1024]:
      OperationMutationTypedScopeRefV1

AuthorityQualificationRoleRefV1
  role_ref
  role_entry_sha256: SHA-256

OperationApprovalQuorumRuleV1
  allowed_approver_role_refs[1..8]:
    AuthorityQualificationRoleRefV1
  independence_class:
    none | independent_from_requester |
    independent_from_owner_and_requester
  minimum_distinct_subjects: positive uint8

TaskAuthorizationEnvelopePayloadV1
  envelope_version, task_id, spec_sha256
  requester_subject_ref
  issued_at, not_before, expires_at, nonce
  risk_class
  contract_set_hash, policy_set_hash
  foundation_definition_closure_ref: FoundationDefinitionClosureRefV1
  current_control_plane_baseline_binding:
    exact CurrentControlPlaneBaselineBindingV1
  resolved_profile_hashes[]
  tool_catalog_hash
  allowed_operations[]
  subject_scope_grants[0..64]:
    TaskAuthorizationSubjectScopeGrantV1
  path_grants[]
  network_policy
  dependency_policy
  secret_policy
  resource_limits
  repair_attempt_limit
  required_gates[]
  required_approvals[0..8]:
    OperationApprovalQuorumRuleV1
  long_running_grant?
  policy_service_subject_ref
  policy_service_role_ref
  revocation_snapshot_ref

TaskAuthorizationEnvelopeV1
  payload: TaskAuthorizationEnvelopePayloadV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=task_authorization_envelope)

TaskAuthorizationEnvelopeRefV1
  task_id
  envelope_version: 1
  wrapper_sha256: SHA-256
```

OperationはID＋versionのexact allowlistとし、wildcardを禁止する。Networkはdeny_all、Dependencyはno_change、AI TaskのSecretはno_secret_accessを既定とする。Pathはread／write／create／deleteを別に許可し、Process tree、CPU、memory、wall time、child count、output sizeをhard limitにする。

`TaskAuthorizationEnvelopeRefV1.wrapper_sha256`は、署名検証済みの完成`TaskAuthorizationEnvelopeV1`全体をRFC 8785 JCSでencodeしたbytesのSHA-256である。Refの`task_id`／`envelope_version`は解決先payloadとbyte equalityにし、同じIDで別wrapper、payloadだけのhash、署名前wrapper、別purpose署名を拒否する。`requester_subject_ref`はTask作成要求を認証したcurrent Trust identityへexact一件解決し、caller文字列、Host名、Model名、task IDから補完しない。`foundation_definition_closure_ref`は[Executable Contracts §5.1](../02-foundation/executable-contracts.md#51-foundation-definition-closure)のexact refで、`contract_set_hash`は解決したClosure内のContract Set rootとbyte equalityにする。`current_control_plane_baseline_binding`は発行時のcurrent Product operational headにinline保存された同名値のbyte-exact copyであり、そのBootstrap／Rebaseline Coreが束縛するFoundation ClosureとpayloadのClosure refを一致させる。Policy Serviceは発行時と各dispatch直前にcurrent Product head、Baseline binding、Foundation Closureのchainを再検証し、Baseline bindingが変わった古いEnvelope、retained Closureへの差替え、同じContract Set rootだけを持つ別Owner／Algorithm rootを拒否して新Envelopeを要求する。通常のCapability／Risk／WP state更新でProduct snapshot headだけが進みBaseline bindingがbyte-identicalな場合、Envelopeをその理由だけで失効させない。ただしcurrent Capability activation／Target Qualification／Policy／Tool Catalogはdispatchごとに別途再評価し、Envelope発行時のsnapshot stateを権限として保持しない。

`AuthorityQualificationRoleRefV1.role_entry_sha256 = SHA-256(RFC 8785 JCS(exact completed AuthorityRoleRegistryV1 entry))`とし、current Trust closureが指すRole Registryへ`role_ref`とhashでexact一件解決する。ID-only、display name、Role prefix、同ID別row、transport signing Roleへの置換を拒否する。

`subject_scope_grants[]`、各nested scope集合、`required_approvals[]`はbranch discriminator、typed ref bytes、各quorum groupの`AuthorityQualificationRoleRefV1`集合、independence class順のstrict sorted setとし、duplicate、同じlogical refの別hash、project／non-project sibling Field混在を拒否する。一つの`OperationApprovalQuorumRuleV1`は`allowed_approver_role_refs[]`のany-of threshold group、複数ruleはall-ofである。Project grantはexact base tripleから開始し、後続revisionを同じTaskの検証済みsigned Domain Receipt／Public Marker chainだけで辿る。`expected_project_revision`はbase以上、checked `base + max_revision_advance`以下で、各直前`document_set_hash`と一致しなければならない。外部Task、手動Commit、別Project、同revision別document setを「revision範囲内」として許可しない。状態変更Operationは少なくとも一つのscope grantにcoverされ、Project Change Primitiveから投影した全typed scopeがproject branchの`allowed_subject_scope_refs[]`にも含まれなければならない。empty scope grantはProject／Source／Release mutationをcoverせず、R0 global queryだけがOperation固有Policyにより使用できる。

Task AuthorizationはTask／Operation／Project／typed subject／Path／Network／Budget等の上限を許可するscope grantであり、まだ存在しない個別Operation inputの`operation_intent_hash`を捏造して署名しない。個別状態変更のexact intentは後述のOperation Approval SetまたはPredelegation Useが束縛し、Policy Serviceが両者のcoverageを交差検証する。

repair_attempt_limitはTaskAuthorizationEnvelopeの必須uint8 Fieldであり、Policy Serviceが署名時に確定する。適用単位は同一task_idのTask全体において、初回Proposal後に許可する修復Proposal再提出回数である。初回Proposalは数えず、Validatingでblocking diagnosticを得てRunningへ戻し、次の修復Attemptを予約する時点で1を消費する。全route共通のtask-scoped current `AiTaskRepairAttemptHeadV1.repair_attempt_count`だけを正本とし、Gatewayはsigned `AiTaskAttemptReservationV1`と`state=reserved` Headのexpected-previous CAS成功後にだけProvider／Host実行またはProposal受入れを開始する。Envelopeの更新／再発行、Worker、Host、Transport、Grant、Model、Provider、Attemptの変更、再起動、AwaitingUserInputではresetしない。Goal、Input、Riskまたは権限変更により新Taskと新Envelopeを発行した場合だけ別の適用単位とする。

署名Policyの既定値と上限を次に固定する。Envelopeは該当Riskの値以下へ縮小できるが、増加できない。

| Risk | repair_attempt_limit |
|---|---:|
| R0 | 0 |
| R1 | 2 |
| R2 | 2 |
| R3 | 1 |
| R4 | 0 |
| R5 | 0 |

通常TaskはR0を含め署名必須である。製品内Chat等の連続する読取専用QueryはSession単位のR0 Envelope一つへ束ねてよい。このEnvelopeはread Operation allowlistと有効期限を固定し、Proposal組立へ進む時点で新Taskと新Envelopeを必要とする。署名鍵作成前のBootstrapDiscoveryは通常State machine外に置き、local system情報読取り、Key生成、Public key registry初期化だけを許可する。Provider、Project読取り、Worker、任意Path、Network、変更を許可せず、初期化後に再実行できない。

署名RecordはMirakanSignedRecordV1を使用する。Task Authorization wrapperは`subject_sha256=SHA-256(JCS(payload))`、`signed_record.signer_subject_ref=payload.policy_service_subject_ref`、`signer_role_ref=payload.policy_service_role_ref`、`issued_at=payload.issued_at`、`revocation_snapshot_ref=payload.revocation_snapshot_ref`をbyte equalityにする。初期ProfileはECDSA P-256／SHA-256、RFC 8785 JCS、P1363固定64 byte、base64url without padding、low-S必須である。unknown field、duplicate key、invalid UTF-8、非有限値、高S、未知／失効／用途不一致／期限外Keyをfail closedで拒否する。秘密鍵は専用Service identityのnon-exportable Key storeへ置き、AI／Workerから分離する。AI Orchestratorにはgeneration_receipt用途専用のService identityとnon-exportable Keyだけを割り当て、この用途KeyをVerification、Approval、Promotion、Release用途へ流用しない。ActorのSecret保持禁止はProvider Credential等の可搬Secretを指し、このnon-exportable署名identityを含まない。

Operation Receiptの署名権限は次のclosed `OperationReceiptSignerPolicyV1`だけが与える。

```text
OperationReceiptSignerPolicyEntryV1
  operation_ref: McdContractRefV1(kind=operation)
  execution_authority_ref: TrustedServiceRefV1
  signer_subject_ref
  signer_role_ref
  allowed_signed_record_purpose

OperationReceiptSignerPolicyV1
  policy_id: operation_receipt_signer_policy.active
  policy_version: positive uint32
  entry_count: uint32
  entries[0..4096]: OperationReceiptSignerPolicyEntryV1
  policy_content_hash: SHA-256

OperationReceiptSignerPolicyRefV1
  policy_id: operation_receipt_signer_policy.active
  policy_version: positive uint32
  policy_content_hash: SHA-256
```

`policy_content_hash = SHA-256(ASCII "MIRAKAN_OPERATION_RECEIPT_SIGNER_POLICY_V1" || uint32_be(len(RFC 8785 JCS(closed Policy object excluding policy_content_hash))) || RFC 8785 JCS(closed Policy object excluding policy_content_hash))`とする。DigestはJCS内でlowercase hexadecimal exact 64文字、uint32はsafe JSON integerとする。entriesはOperation refのID／version／Contract set hash、Authority ref、Signer subject、Role、purpose bytes順へstrict sortし、`entry_count`を配列長と一致させる。同じOperation refはexact一件で、duplicate、same Operation different signer、prefix展開、RoleまたはKeyだけの存在を拒否する。row集合を変える場合だけ`policy_version=N+1`とし、旧Policy root、Trust rows、Receiptをretentionする。

`OperationReceiptSignerPolicyV1`のLocal Schema Catalog root IDはexact `urn:mirakan:schema:ai-security:operation-receipt-signer-policy:v1`、Ownerは`mirakan.arch.ai-security-approval`で、closed object、unknown Field禁止、signature slot exact `[]`とする。schema annotationは`x-mirakan-governance-policy-id-kind=fixed_logical`、許可logical ID exact `[operation_receipt_signer_policy.active]`であり、content-derived ID branchを許可しない。このschemaはAI Security OwnerのTask 2 complement memberであり、Architecture Governance Ownerの「追加Control Plane関連contract exact 30」へ重複加算しない。Catalog memberの`schema_ref/schema_sha256`は完成Draft 2020-12 schema bytesへ解決し、`GovernancePolicyConfigRegistryV1.policy_schema_id`はこのIDとbyte equalityにする。

current Signer Policyの唯一の選択点は、`GovernancePolicyConfigRegistryV1`で`policy_id=operation_receipt_signer_policy.active`を持つexact一rowである。これはRegistry全体を一rowに限定せず、他の承認済みControl Plane Policyを削除しない。完成Policy objectの`policy_object_sha256 = SHA-256(RFC 8785 JCS(completed OperationReceiptSignerPolicyV1 including policy_content_hash))`を計算し、rowを`{policy_id=operation_receipt_signer_policy.active, policy_schema_id=urn:mirakan:schema:ai-security:operation-receipt-signer-policy:v1, policy_ref=ArtifactRefV1(artifact_kind=governance.operation_receipt_signer_policy,schema_version=1,sha256=policy_object_sha256), policy_sha256=<policy_object_sha256 lowercase-hex>}`へ固定する。`policy_ref`を解決した完成bytesのSHA-256は`policy_ref.sha256`および`policy_sha256`、解決したobjectの`{policy_id,policy_version,policy_content_hash}`は`OperationReceiptSignerPolicyRefV1`とbyte equalityにする。完成object全体の`policy_object_sha256`と、自己Fieldを除くsemantic hashである`policy_content_hash`は別値であり、両者の等値を要求しない。Registry、Trust Registry Closure、Trust Headを同じCAS transactionでcurrent化し、self-consistentだがこのcurrent rowから到達しないPolicy、bare latest、別Trust closureのPolicyをAuthorityにしない。current planning baselineにおけるこのSigner rowだけは`policy_version=1`、`entry_count=0`、`entries=[]`を指し、MCD `status=active`またはProfile存在から署名Authorityを合成しない。

Receipt publicationが同Registryから解決するrequired subsetは、このSigner rowと[Executable Contracts §8](../02-foundation/executable-contracts.md#8-operation定義)の`published_receipt_verification_retention_policy.active`、`published_receipt_recovery_policy.active`、`published_receipt_store_namespace_policy.active`のdisjoint union、exact四rowである。後三rowはExecutable Contracts Ownerの一closed tagged-union schema、fixed logical ID、completed-object SHAとsemantic hashを使い、Receiptごとの`PublishedReceiptMaterializationPolicyV1`自身はMCDにも、それ自身がpinするGovernance Registryにも登録しない。三row追加はGovernance Policy Config Registry revision／Trust closure／Headを更新するが、Trust Registry種類は六、Trust closure member数はexact 15のままである。Control Policy、Materialization Policy、Authorization Audit Bindingの三unsigned root schemaはTask 0でpre-ceremony full Local Schema Catalog／Materialization Planへ固定し、Task 2がbytesをmaterializeするExecutable Contracts Owner exact三rootで、Authority Binding Source Catalog slotとArchitecture Governance Owner exact 30を増やさない。

Foundationのcurrent Installed Product compositionにあるactive 10 Operationは契約参照可能だが、上記empty Signer Policyのためまだoperationalではない。[Executable Contracts §8.1](../02-foundation/executable-contracts.md#81-project-runtime-entryruntime-scopeの正規operation登録)の`GenericCoreOperationBaselineRefV1`はProject State bootstrap exact六entry、[§8.2](../02-foundation/executable-contracts.md#82-current-installed-product-active-operation-closure)の`InstalledProductOperationCompositionRefV1`は同baseline六entryとWorld exact一entry、Shooter Genre Pack exact三entryのunionである。所有レイヤーと構成所属を混同せず、Worldは`production_owner_layer=core; composition_role=installed_core_extension`、Shooterは`production_owner_layer=genre_pack; composition_role=genre_pack`として検証する。

`activation.foundation.operation_domain_receipt_pipeline.v1`は`generic_core_operation_baseline_ref: GenericCoreOperationBaselineRefV1`と`installed_product_operation_composition_ref: InstalledProductOperationCompositionRefV1`を必須Fieldとしてpinする。Baseline refはCompositionの`generic_core_baseline_ref`とbyte equality、Baseline exact六entryはComposition内の`composition_role=generic_core_baseline` exact subset、Composition exact十entryはcurrent MCD／Owner Manifest contribution union／Service allowlist／下表十Signer destinationとset equalityにする。そのうえでProject State Origin六OperationのType／Policy／Validator／Diagnostic、Service／Isolation Profile、下表先頭六Signer row、三Publication Control Policy row、Authorization Audit Binding／Materialization Policy／Context schemaとfixture、既存六Trust RegistryのIdentity／Role／Assignment／Public Key／Governance Policy Config required subset、fresh Service executable／Target Technical Qualificationだけを一つの承認済みtransactionで有効化する。Signer Policyはcurrent empty version `N`からbaseline六row exact集合を持つversion `N+1`へ進め、対応Trust／Governance currentを同じCASで発行する。World／Shooter extension四row、対応Role／assignment／Keyは追加せず、contract-active十のうちoperational集合をexact baseline六にする。baseline一～五件だけ、extension混入、stale Baseline／Composition ref、Owner／layer／role／Origin／Service／Signer／purpose／Audit Binding差があればtransaction全体を拒否し、operational 0を維持する。

`activation.installed_product.operation_composition_extensions.v1`は、上記baseline六がcurrent operationalであること、Signer Policy／Trust／Governance rowが同じcompleted baseline transactionから到達すること、current Installed Product compositionの`entries - generic_core_baseline entries`がWorld Origin一＋Shooter Origin三のexact四件であることを入力にする。Wave 3の同じCandidateへbaseline六をfresh再Qualificationし、extension四もTarget／Service／Contract／Owner単位でfresh Qualificationした場合だけ、baseline六のSigner／Role／assignment／Key bytesを保持してWorld一＋Shooter三の下表四rowだけを一括追加する。Signer Policy、変更対象Trust Registry、Governance Policy Configは各current `N`からexact `N+1`へ進め、完了後のSigner destination、Service operational contribution union、Installed Product operational snapshotを同じexact十件へset equalityにする。Wave 1とWave 3 CandidateのEvidence混在、baseline row再署名、extension一～三件だけ、同一Origin内の部分Activation、別Composition ref、World／Shooter未qualifiedを拒否し、失敗時は既存operational baseline六を不変に保つ。Product Registryにconsumer Capability rowがあるOperationはそのTarget Activationも必要とし、Product rowを持たないservice-authority Capabilityは後述のoperational contribution predicateを使う。`operation.project.runtime_entry.migrate_root_scene`、`operation.runtime_scope.migrate_game_system`、`operation.performance.migrate_project_scale_envelope`、`operation.physics.intent_role.migrate`はExecutable Contracts §8.1.2のsigned legacy evidence gateを満たしておらず、いずれのtransaction、current Signer Policy destination、Trust assignment／Key集合にも含めない。四候補用Receipt Role／assignment／Keyのcurrent集合もexact `[]`で、将来の各atomic activation transactionだけが対象Operation固有rowを追加できる。

| operation_id | execution authority = signer subject | signer_role_ref | allowed_signed_record_purpose |
|---|---|---|---|
| `operation.project.runtime_entry.create` | `service.authoring_command_gateway` | `role.operation_domain_receipt.project_runtime_entry_create` | `operation_domain_receipt` |
| `operation.project.runtime_entry.update` | `service.authoring_command_gateway` | `role.operation_domain_receipt.project_runtime_entry_update` | `operation_domain_receipt` |
| `operation.project.runtime_entry_activation_policy.create` | `service.authoring_command_gateway` | `role.operation_domain_receipt.project_runtime_entry_activation_policy_create` | `operation_domain_receipt` |
| `operation.project.runtime_entry_activation_policy.update` | `service.authoring_command_gateway` | `role.operation_domain_receipt.project_runtime_entry_activation_policy_update` | `operation_domain_receipt` |
| `operation.project.runtime_target_selector.create` | `service.authoring_command_gateway` | `role.operation_domain_receipt.project_runtime_target_selector_create` | `operation_domain_receipt` |
| `operation.project.runtime_target_selector.update` | `service.authoring_command_gateway` | `role.operation_domain_receipt.project_runtime_target_selector_update` | `operation_domain_receipt` |
| `operation.shooter.target_provider_binding.create` | `service.authoring_command_gateway` | `role.operation_domain_receipt.shooter_target_provider_binding_create` | `operation_domain_receipt` |
| `operation.shooter.target_provider_binding.select` | `service.authoring_command_gateway` | `role.operation_domain_receipt.shooter_target_provider_binding_select` | `operation_domain_receipt` |
| `operation.shooter.target_provider_binding.update` | `service.authoring_command_gateway` | `role.operation_domain_receipt.shooter_target_provider_binding_update` | `operation_domain_receipt` |
| `operation.world.allocate_generated_stable_ids` | `service.authoring_command_gateway` | `role.operation_domain_receipt.world_allocate_generated_stable_ids` | `operation_domain_receipt` |
各Role assignmentは表のOperation ref一件とProject／Target scopeだけを許可し、Roleとnon-exportable Keyの`allowed_signed_record_purposes[]`をsingleton `[operation_domain_receipt]`にする。同じpurpose文字列でもRole／assignment／Policy rowが別Operationを許可しないため、Keyだけを別Operationへ流用できない。Shooter Genre Packを取り外す場合は、Executable Contractsの`contribution.owner.genre.shooter.target_provider_binding` exact三entry、上表のShooter exact三row、対応Role／assignment／Key、MCD／Service／Provider projectionを一つの承認済みdefinition migrationで同時に除去し、新しいInstalled Product composition／Signer Policy／Trust・Foundation closureを再発行する。`GenericCoreOperationBaselineRefV1`、Project State bootstrap六entryおよびそのbaseline hashはbyte-exactに保持する。旧三rowをCoreへ付替えること、baselineへ混入すること、aliasとしてcurrentに残すことを拒否する。この記述は将来の構成変更規則であり、current active 10／conditional 4／reserved 191の件数を変更しない。

同じSigner Policy mappingは`planning.operation_family.build_device_play_debug_task`のActivation受入条件でもある。`activation.build_gateway.operation_pipeline.v1`が14 Operation、MCD、Manifest、Service allowlist、Receipt、Diagnostic、Validator closureをactivateする場合だけ、destination Policyのbuild-family subsetを次の14件とexact一致させ、既にoperationalな他family subsetはbyte-exactに保持する。表だけ、Roleだけ、Keyだけの先行materializeを禁止する。

| operation_id | execution_authority | signer_role_ref | allowed_signed_record_purpose |
|---|---|---|---|
| `operation.build.request_package` | `build_gateway` | `role.operation_receipt.build_request_package` | `operation_receipt:operation.build.request_package` |
| `operation.device.install` | `device_bridge` | `role.operation_receipt.device_install` | `operation_receipt:operation.device.install` |
| `operation.device.launch` | `device_bridge` | `role.operation_receipt.device_launch` | `operation_receipt:operation.device.launch` |
| `operation.device.reset_data` | `device_bridge` | `role.operation_receipt.device_reset_data` | `operation_receipt:operation.device.reset_data` |
| `operation.play.run_smoke` | `play_service` | `role.operation_receipt.play_run_smoke` | `operation_receipt:operation.play.run_smoke` |
| `operation.debug.aggregate` | `debug_query_service` | `role.operation_receipt.debug_aggregate` | `operation_receipt:operation.debug.aggregate` |
| `operation.debug.query` | `debug_query_service` | `role.operation_receipt.debug_query` | `operation_receipt:operation.debug.query` |
| `operation.debug.read_causality` | `debug_query_service` | `role.operation_receipt.debug_read_causality` | `operation_receipt:operation.debug.read_causality` |
| `operation.debug.read_replay_slice` | `replay_service` | `role.operation_receipt.debug_read_replay_slice` | `operation_receipt:operation.debug.read_replay_slice` |
| `operation.debug.validate_finding` | `debug_validation_service` | `role.operation_receipt.debug_validate_finding` | `operation_receipt:operation.debug.validate_finding` |
| `operation.debug.support-bundle.generate` | `debug_export_service` | `role.operation_receipt.debug_support_bundle_generate` | `operation_receipt:operation.debug.support-bundle.generate` |
| `operation.task.status` | `build_gateway_task_service` | `role.operation_receipt.task_status` | `operation_receipt:operation.task.status` |
| `operation.task.read_receipt` | `build_gateway_task_service` | `role.operation_receipt.task_read_receipt` | `operation_receipt:operation.task.read_receipt` |
| `operation.task.cancel` | `build_gateway_task_service` | `role.operation_receipt.task_cancel` | `operation_receipt:operation.task.cancel` |

`planning.operation_family.build_candidate_test`は上記14件とは別のdestination subsetである。`activation.build.candidate_test_operations.v1`が[Core architecture §9.2](../02-foundation/core-architecture.md#92-build-candidatetestのtyped-execution-closure)のnamed Input／Task／success・failure・cancel payload、Receipt wrapper、Validator／Diagnostic、Service allowlistをexact六件すべて同時にmaterializeする場合だけ、Signer Policyへ次の六行を追加できる。current六行のPolicy／Role／assignment／Key／Receipt instance集合はexact `[]`であり、表の存在、同じ`build_gateway` Authority、後段Package候補から署名権限を合成しない。

| operation_id | execution_authority | signer_role_ref | allowed_signed_record_purpose |
|---|---|---|---|
| `operation.build.request_validate` | `build_gateway` | `role.operation_receipt.build_request_validate` | `operation_receipt:operation.build.request_validate` |
| `operation.build.request_cook` | `build_gateway` | `role.operation_receipt.build_request_cook` | `operation_receipt:operation.build.request_cook` |
| `operation.build.request_native_module` | `build_gateway` | `role.operation_receipt.build_request_native_module` | `operation_receipt:operation.build.request_native_module` |
| `operation.build.request_project_shader` | `build_gateway` | `role.operation_receipt.build_request_project_shader` | `operation_receipt:operation.build.request_project_shader` |
| `operation.build.request_game_candidate` | `build_gateway` | `role.operation_receipt.build_request_game_candidate` | `operation_receipt:operation.build.request_game_candidate` |
| `operation.test.request_run` | `candidate_test_service` | `role.operation_receipt.test_request_run` | `operation_receipt:operation.test.request_run` |

Activation後、各`signer_role_ref`は同じ行のpurpose一件だけを許可する。Public key registryの各`key_id`も、その実行Authority subject、exact Signer Role、`allowed_signed_record_purposes[]`が同じ一件だけのsingletonでなければならない。同じServiceが複数Operationを実行してもRoleとnon-exportable KeyをOperationごとに分離し、generic Operation Receipt Role／Keyを作らない。通常Rotationで新旧Keyを重複有効にする場合も、両KeyのAuthority、Role、singleton purposeを同一に保つ。Foundation／Build 14件／Build Candidate 6件の各subset、MCDのexecution Authority、本PolicyのOperation／subject／Role／purposeが一致しないRecordをVerifierは拒否する。六件Activation時は既にoperationalなFoundation／14件subsetをbyte-exactに保持し、`operation.build.request_package`のRole／purposeをValidate、Cook、Source Build、Game CandidateまたはTestへ流用しない。

鍵期限到来時の通常RotationはSecurity incidentではなく、BootstrapDiscoveryの再実行でもない。信頼済みAuthorityが発行する署名済みKeyRotation Operationとして、新Key生成、Public key registry更新、新旧Keyの重複有効期間の設定、失効リスト配布を行う。重複期間中は新旧両Keyでの検証を許可し、期限後の旧Keyは署名用途から除外する。過去Receiptの検証用に旧公開鍵と失効情報をregistryへ保持し、削除しない。侵害時のKey revocationとclean environment再構築を定期Rotationの代替にしない。

### 3.2 Task state machine

正規状態は次の15個だけである。

    Draft, ResolvingRequirements, AwaitingAuthorization, Ready
    Running, Validating, AwaitingCodeOwner, AwaitingApproval, Promoting
    AwaitingUserInput
    Completed, Cancelled, Expired, Failed, Rejected

主経路は次である。

    Draft -> ResolvingRequirements -> AwaitingAuthorization -> Ready
      -> Running -> Validating
      -> AwaitingApproval? -> Promoting? -> Completed

不変条件を次に固定する。

- Authorization前にSource Workerまたは変更Toolを起動しない。
- Envelope外のOperation、Path、Network、Dependencyを実行しない。
- Input revision、Contract、Policy、Profile、Tool catalogが変わったTaskを継続しない。
- Native／Shader Source生成前に有効なCode owner assignmentがなく、またはPromotion前にexact DiffへのCode owner approvalがなければ進めない。
- Approvalは表示したDiff、Gate result、Artifact hashと完全一致するrevisionだけに有効である。
- Rejected、Failed、ExpiredのArtifactを昇格しない。
- CompletedはAuthoritative revision／commitのread-back照合後だけにする。

AwaitingUserInputはResolvingRequirements、Running、Validatingからだけ入る。Authorization前の回答はSpecification draftを更新できる。Authorization後の回答がGoal、Success criteria、Input、Risk、Operation、Path、Network、Dependency、Gate、Approvalへ影響する場合、元TaskをCancelledにし、新Taskと新Envelopeを作る。

`AwaitingCodeOwner`は、Source生成前の`ResolvingRequirements`またはSource検証後の`Validating`からだけ入る。前者は有効な`CodeOwnerAssignmentV1`が得られるかDefinition／事前Qualification済みPackへPlanを再解決するまでSource Workerを起動しない。後者は同じSource revisionとexact Diff hashへの`CodeOwnerApprovalV1`が得られるまでPromotionへ進めない。解決後は元のstateへ戻して全preconditionを再検証し、このstateから`Running`、`Promoting`、`Completed`へ直接遷移しない。Editor projection名は`awaiting_code_owner`とする。

修復はMCD RemediationV1があり、同じInput、Envelope内、permission／security／lock／approval／revision drift以外の場合だけ許可する。修復再実行はValidatingからRunningへ戻る遷移であり、正規15状態へ状態を追加しない。修復Proposal再提出はEnvelopeのrepair_attempt_limitを超えてはならない。同じblocking diagnostic＋location＋Stable ID集合が再発し、blocking数が減らない場合は残数があっても即停止する。

Atomic commit、許可済みlong-running verification、Release transactionのcritical section開始後は、Cancel／Expiryで結果不明のまま終了表示しない。完了、rollback、read-backのいずれかへ収束させる。

本節の15状態はAI Orchestrator TaskのGovernance stateである。Build familyのatomic Activation後、[Core architecture](../02-foundation/core-architecture.md#91-operationtaskv1)のplanned `OperationTaskV1.state = queued | running | cancel_requested | succeeded | failed | cancelled`は個々のPackage／Device／Play／Debug実行ledgerになり、相互の状態名をaliasにせず、Operation Receiptから親Taskへ結果を投影する。current `OperationTaskV1` instanceは0件である。

## 4. Risk classとActivation

### 4.1 R0–R5

Riskは変更後の最大影響で決め、AI自己申告で下げない。

| Risk | 代表変更 | AI | 自動昇格 | 必須Approval |
|---|---|---|---|---|
| R0 | 読取、検索、説明、Report | 可 | 状態変更なし | 不要 |
| R1 | 文書、非実行Sample、個人Editor layout | 可 | protected branch外かつ全Gate時だけ可 | Owner不要、Gate必須 |
| R2 | GameSpec、World／Scene／Space／Topology、owner-typed Project Document、UI、Asset設定、GameplayDefinition、既存System設定 | 可 | 署名済み事前委任内の可逆Operationを、Release候補化されていないProject revisionへ昇格する場合だけ | Author 1名または同等Scopeの事前委任 |
| R3 | Project-defined System、bounded NativeGameModule、Project Shader、互換Schema | 可 | 不可 | G0–G7とPolicy Service署名。開発ProfileではCode owner＋全Gate |
| R4 | authoritative State owner／Save意味。Engine製品ProfileではEngine core／Extension／Security等 | Project artifact可。Engine sourceはGame制作不可 | 不可 | Project Attestation／Approval。EngineはDomain owner＋独立Reviewer |
| R5 | merge、tag、sign、Store upload、Production secret、公開Release | Proposalだけ | 禁止 | Human release owner＋分離Pipeline |

Test削除、Assertion緩和、Budget引上げ、Schema制約削除、Approval削除は対象実装と同じか一段高いRiskにする。R5 OperationをModel Tool catalogへ公開しない。

R2事前委任はOperation ID＋version、対象DocumentのStable ID／Shard等のtyped scope、最大件数／byte、適用先Project、有効期限、rollbackを固定し、Save schema、Public API、Asset license、Dependency、Security、課金、公開配布を含めない。R2構造化編集の適用可否は[Project state](../03-authoring/project-state.md)の語彙(Project revision、Staging Candidate、Release候補化の有無)で判定し、Git branchへ依存しない。branch条件はGit連携を有効化したR1文書系とR3以上のSource系だけに適用する。Promotion Serviceは実Diffを再分類し、Envelope超過なら新Authorizationを要求する。

#### Mutation authorization binding V2

Task scope grantと個別Mutation intentの承認証拠を混同しない。状態変更Operationが参照できる認可wrapper／Refを次のclosed型に固定する。

```text
OperationMutationSubjectScopeV1
  project:
    project_ref: exact {project_id, expected_project_revision, document_set_hash}
  | non_project:
    typed_scope_ref: OperationMutationTypedScopeRefV1

OperationMutationApprovalPayloadV1
  approval_id
  authorization_ref: TaskAuthorizationEnvelopeRefV1
  authorization_hash: SHA-256
  requester_subject_ref
  operation_intent_hash: SHA-256
  operation_ref: McdContractRefV1(kind=operation)
  input_type_ref: McdContractRefV1(kind=type)
  risk_class: R2 | R3 | R4 | R5
  subject_scope: OperationMutationSubjectScopeV1
  decision: approved
  approver_subject_ref
  approver_role_ref: AuthorityQualificationRoleRefV1
  issued_at
  expires_at
  revocation_snapshot_ref

OperationMutationApprovalV1
  payload: OperationMutationApprovalPayloadV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=operation_mutation_approval)

OperationMutationApprovalRefV1
  approval_id
  approval_version: 1
  wrapper_sha256: SHA-256

OperationMutationApprovalSetV1
  approval_set_id
  authorization_ref: TaskAuthorizationEnvelopeRefV1
  authorization_hash: SHA-256
  requester_subject_ref
  operation_intent_hash: SHA-256
  operation_ref: McdContractRefV1(kind=operation)
  input_type_ref: McdContractRefV1(kind=type)
  risk_class: R2 | R3 | R4 | R5
  subject_scope: OperationMutationSubjectScopeV1
  applied_quorum_rules[1..8]: OperationApprovalQuorumRuleV1
  approvals[1..16]:
    {approval_ref: OperationMutationApprovalRefV1,
     approval_hash: SHA-256}
  set_content_hash: SHA-256

OperationMutationApprovalSetRefV1
  approval_set_id
  approval_set_version: 1
  set_content_hash: SHA-256

R2OperationPredelegationGrantPayloadV1
  grant_id
  grant_nonce: exact 16 random bytes;
    JSON/JCS projection=base64url without padding exact 22 ASCII chars
  allowed_operation_refs[1..64]: McdContractRefV1(kind=operation)
  project_scope: ProjectMutationScopeGrantV1
  allowed_subject_scope_refs[1..1024]: OperationMutationTypedScopeRefV1
  max_mutation_count: positive uint32
  max_total_bytes: positive uint64
  issued_at
  valid_from
  expires_at
  rollback_policy_ref: ArtifactRefV1
  prohibited_effects:
    [asset_license, billing, dependency, public_api,
     public_distribution, save_schema, security_policy]
  delegator_subject_ref
  delegator_role_ref: AuthorityQualificationRoleRefV1
  revocation_snapshot_ref

R2OperationPredelegationGrantV1
  payload: R2OperationPredelegationGrantPayloadV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=r2_operation_predelegation_grant)

R2OperationPredelegationGrantRefV1
  grant_id
  grant_version: 1
  wrapper_sha256: SHA-256

R2OperationPredelegationConsumptionHeadRefV1
  grant_ref: R2OperationPredelegationGrantRefV1
  head_generation: uint64
  head_content_hash: SHA-256

R2OperationPredelegationConsumptionHeadV1
  grant_ref: R2OperationPredelegationGrantRefV1
  grant_hash: SHA-256
  head_generation: uint64
  consumed_mutation_count: uint64
  consumed_total_bytes: uint64
  last_use_ref: OperationPredelegationUseRefV1 | null
  last_use_hash: SHA-256 | null
  head_content_hash: SHA-256

OperationPredelegationUsePayloadV1
  use_id
  grant_ref: R2OperationPredelegationGrantRefV1
  grant_hash: SHA-256
  authorization_ref: TaskAuthorizationEnvelopeRefV1
  authorization_hash: SHA-256
  requester_subject_ref
  operation_intent_hash: SHA-256
  operation_ref: McdContractRefV1(kind=operation)
  input_type_ref: McdContractRefV1(kind=type)
  risk_class: R2
  subject_scope: OperationMutationSubjectScopeV1(project)
  used_scope_refs[1..1024]: OperationMutationTypedScopeRefV1
  mutation_count: positive uint32
  total_bytes: positive uint64
  previous_consumption_head_ref:
    R2OperationPredelegationConsumptionHeadRefV1
  resulting_consumed_mutation_count: positive uint64
  resulting_consumed_total_bytes: positive uint64
  rollback_plan_ref: ArtifactRefV1
  disposition: covered
  policy_service_subject_ref
  policy_service_role_ref
  issued_at
  expires_at
  revocation_snapshot_ref

OperationPredelegationUseV1
  payload: OperationPredelegationUsePayloadV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=operation_predelegation_use)

OperationPredelegationUseRefV1
  use_id
  use_version: 1
  wrapper_sha256: SHA-256

R2PredelegationScopeProjectionV1
  primitive_id
  primitive_type
  scope_refs[1..16]: OperationMutationTypedScopeRefV1
  projection_content_hash: SHA-256
```

`approval_id`、`grant_id`、`use_id`は各ID Fieldを除く完成payloadのRFC 8785 JCS SHA-256から、それぞれ`urn:mirakan:operation-mutation-approval:sha256:<lowercase-hex>`、`urn:mirakan:r2-operation-predelegation-grant:sha256:<lowercase-hex>`、`urn:mirakan:operation-predelegation-use:sha256:<lowercase-hex>`として決定論的に導出する。Approval Setは固定点を避け、`set_content_hash = SHA-256(ASCII "MIRAKAN_OPERATION_MUTATION_APPROVAL_SET_V1" || uint32_be(len(RFC 8785 JCS(closed Set object excluding approval_set_id and set_content_hash))) || RFC 8785 JCS(closed Set object excluding approval_set_id and set_content_hash))`を先に計算し、`approval_set_id=urn:mirakan:operation-mutation-approval-set:sha256:<set_content_hash lowercase-hex>`とする。Scope Projectionの`projection_content_hash`はASCII `MIRAKAN_R2_PREDELEGATION_SCOPE_PROJECTION_V1`と自己Fieldを除くRFC 8785 JCS bytesから計算する。`allowed_operation_refs[]`、`allowed_subject_scope_refs[]`、`used_scope_refs[]`、quorum rule、Approval entry、Projection scopeは各型のcanonical identity bytes順のstrict sorted setで、duplicate、same identity different hash、非canonical orderを拒否する。`subject_scope`はexact一branchだけを持ち、discriminator外Fieldをcanonical omissionする。Grantの`prohibited_effects`は上記exact七値をASCII byte順で全件持ち、省略、追加、並替え、emptyを許可しない。全署名wrapperはpayload `issued_at < expires_at`、`signed_record.issued_at=payload.issued_at`を満たす。Grantはさらに`issued_at <= valid_from < expires_at`、Useは`expires_at=min(Grant expires_at, Task Authorization expires_at, current Policyのpredelegation-use上限)`を満たす。Approvalの上限はRisk別Approval freshness Policyから導出し、caller時刻またはModel出力を使わない。

`grant_nonce`はOS CSPRNGから生成したexact 16 bytesで、all-zeroを拒否し、同じTrust domain内の全retained Grant Wrapper Registryで再利用を許可しない。JCS hash入力はRFC 4648 base64url without paddingのexact 22 ASCII文字だけであり、hex、padded base64、UUID、数値、Unicode別表現を受理しない。

三signed Refの`wrapper_sha256`と隣接`*_hash`は同じ完成wrapperの`SHA-256(JCS(wrapper))`へbyte equalityにする。Approval Set ref／hashは同じ完成unsigned content-addressed Setへ解決し、各entryは個別の署名済みApproval wrapperへ解決する。Approvalはexact intentを人間が承認したwrapper、Approval SetはRisk／Task／Operation Policyが要求するquorumを満たす個別wrapper集合、Predelegation Useは有効なGrantのclosed制約がexact intentをcoverするとPolicy Serviceが判定し、Grant単位の消費ledgerへ競合安全に予約した一回の使用wrapperである。Grant単体または個別Approval一件をMutation bindingの承認証拠へ使わない。Approval Set／各Approval／Useの`authorization_ref/hash`はBindingのTask Authorization wrapperと、`requester_subject_ref`はそのEnvelope payloadとbyte equalityにする。Approval Set内の全Approval、Predelegation Use、Mutation bindingの`authorization_ref/hash`は一つの同じTask Authorization Envelopeへ解決し、別Task、同task ID別wrapper、payloadだけのhashを許可しない。Operation、input Type、risk、subject scope、intent hashも再計算したOperation inputと一致させる。署名purpose、Signer／Role、発行時刻、期限、current revocation、GrantのOperation／Project／scope／件数／byte／rollback／禁止effectのいずれかが不一致ならcoverageなしとする。Projectless OperationへProject sentinelを作らず、typed `non_project` branchだけを使う。

Core baselineのqualification-only Role rowは次のexact六件である。各rowは`permission_ids=[]`、`allowed_signed_record_purposes=[]`で、`role_entry_sha256`は上記完成row式から再計算する。表のRole名は表示labelではなくexact `role_ref`である。

| role_ref | independence_class |
|---|---|
| `role.project.author` | `none` |
| `role.project.owner` | `none` |
| `role.source.code-owner` | `none` |
| `role.domain.owner` | `none` |
| `role.review.independent` | `independent_from_owner_and_requester` |
| `role.release.owner` | `none` |

Policy ServiceはTask Authorizationの`required_approvals[]`とRisk／authority classのbaseline ruleをcanonical unionし、Approval Setの`applied_quorum_rules[]`とset equalityにする。Baseline exact groupはR2 direct=`{allowed=[role.project.author, role.project.owner], independence=none, minimum_distinct_subjects=1}`、R3 Source=`{allowed=[role.source.code-owner, role.domain.owner], independence=none, minimum_distinct_subjects=1}`、Engine R4=`{allowed=[role.domain.owner], independence=none, minimum=1}`と`{allowed=[role.review.independent], independence=independent_from_owner_and_requester, minimum=1}`のall-of二group、R5追加group=`{allowed=[role.release.owner], independence=none, minimum=1}`である。各allowed値の保存形はRole Registryから再計算した`AuthorityQualificationRoleRefV1`で、上記表示IDだけを保存しない。Project R4／R5が追加または縮小するgroupはcurrent Governance Policy Configのcompleted Risk Policy ref／hashへexact解決し、Task Authorizationへbyte-exact投影する。未登録Role、表示語`Author`等、Role IDだけ、Role hash差をcanonical unionへ入れない。

R5のCoordinator／Build／Signing／Upload分離はApproval人数へ数えず、別Service authority Gateとして検証する。Owner subject集合はsubject scopeが解決するOwner Identity rowとcurrent Authority assignmentから決定し、Task payloadの`requester_subject_ref`と共にApproval／Setへ束縛する。同じsubjectの複数Key、同じhumanの複数Role、requesterまたはOwnerをindependent Reviewerとして水増しせず、Role assignment、independence class、current revocationをTrust closureから検証する。

R2事前委任を使用できる外側Operationは、immutable Staging Candidateが持つ正本`ProjectChangeSetV1.change_primitives[]`と、各primitiveをtyped subject scopeへ写すdeterministic scope projectorをMCD Validator closureへ完全登録しなければならない。登録単位はOperation ref、projector implementation Artifact ref／hash、exact input Change Primitive schema ref、`R2PredelegationScopeProjectionV1` output schema、Projector Validator record、closed Diagnostic set、positive／negative fixtureを一つのContract set transactionで束縛する。`mutation_count`は正本primitive列長、`total_bytes`は各`ProjectChangePrimitiveV1`に実在するprimitive ID、closed primitive type、target Stable ID、typed argument、dependency、expected Document revision、declared costをcanonical topological orderでMCD canonical encodeした各bytes長のchecked uint64和である。scope projectorは各primitive ID／typeをexact一回持つ`R2PredelegationScopeProjectionV1`を生成し、`used_scope_refs[]`は全Projection `scope_refs[]`の一意unionである。Primitiveに存在しないbefore／after／tombstone Fieldを捏造せず、scopeはProjectionだけから導出する。Candidate／ChangeSet／projector implementation hash／全Projection hashをOperation intentのsemantic inputへ含める。入力自己申告値、概算値、圧縮後size、表示上のDocument数を使わず、projector未登録、primitive missing／extra／duplicate、Project Change Primitiveへ100%投影できないOperation、Grantの`allowed_subject_scope_refs[]`に含まれない一scopeでもある場合は事前委任を使用できない。current完全登録済みProjector binding集合はexact `[]`であるため、current全Operationは`predelegated` branchを`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`でSource不変として拒否し、direct Approval Setだけを使用する。将来のatomic Projector Activationまでschema branchの存在を実行権限にしない。

Grant発行transactionは、`grant_id`をkeyとするexpected-empty Grant Wrapper Registryへ完成wrapper ref／hashをexact一件登録し、同じ`grant_id`の別wrapper、別Key、別issued-at、別signatureを拒否する。同payload／KeyのretryはRFC 6979によるbyte-identical wrapperだけを返す。正当なrenewal／Key rotationはfresh `grant_nonce`と`issued_at`を持つ新payload／新grant ID／新Authorizationとして発行し、旧budgetを継承またはresetしない。同じtransactionで`head_generation=0`、両消費値0、`last_use_ref/hash=null`の初期Headをexact一件作る。`head_content_hash = SHA-256(ASCII "MIRAKAN_R2_PREDELEGATION_CONSUMPTION_HEAD_V1" || uint32_be(len(self-excluding RFC 8785 JCS Head bytes)) || self-excluding RFC 8785 JCS Head bytes)`とし、同じgrant IDにcurrent Headを複数持たせない。Grant／Use／Headの全`uint64` FieldはJCSでは範囲検査済みcanonical unsigned decimal stringとしてencodeし、JSON number、指数、先頭`+`、不要な先頭0、`-0`を拒否する。Use発行時はcurrent Headをread-backし、`previous_consumption_head_ref`へexact保存し、checked additionで`resulting_consumed_* = previous consumed_* + this use`を計算する。結果がGrantの`max_mutation_count`または`max_total_bytes`を超える場合、整数overflow、別Grant Head、stale Head、head generation gapの場合はwrapperを発行しない。

Policy Serviceは完成Use wrapperをprivate put-if-absentした後、`head_generation=N+1`、結果累積値、当該Use ref／hashを持つ新Headへのexpected-previous CASを一つのconsumption-ledger transactionで行い、CAS成功後だけUseをdispatch可能にする。競合CAS敗者のwrapperは権限証拠として到達不能にし、新current Headから別`use_id`を再計算する。Useの予約量は後続Operation失敗時も返却せず、上限を保守的に消費する。同じ`use_id`のretryは同じwrapper／committed Headを返し、別intent、別request、別Grantへ再利用しない。GatewayはUseが成功したHead chainへexact一件含まれること、Use ref／hashを含むfinal requestのidempotency ledgerが同じrequest hashだけを受理することを検証する。これにより複数Host／再起動／並行発行でも、Grant全体の件数／byte上限をper-use上限へ誤読しない。

三署名purposeは新しいTrust Registry kindを作らない。wrapper root schemaとLocal／Authority Catalog slotを次のexact集合へ固定する。field pathはRFC 6901 JSON Pointerで、empty stringはroot wrapper自身を表す。

| schema ID | signature slot ID | branch | wrapper path | purpose | issued-at path | revocation path | fixed signing Role |
|---|---|---|---|---|---|---|---|
| `urn:mirakan:schema:ai-security:operation-mutation-approval:v1` | `slot.operation-mutation-approval.wrapper` | `always` | `""` | `operation_mutation_approval` | `/payload/issued_at` | `/payload/revocation_snapshot_ref` | `role.operation-mutation-approval-signer` |
| `urn:mirakan:schema:ai-security:r2-operation-predelegation-grant:v1` | `slot.r2-operation-predelegation-grant.wrapper` | `always` | `""` | `r2_operation_predelegation_grant` | `/payload/issued_at` | `/payload/revocation_snapshot_ref` | `role.r2-operation-predelegation-grant-signer` |
| `urn:mirakan:schema:ai-security:operation-predelegation-use:v1` | `slot.operation-predelegation-use.wrapper` | `always` | `""` | `operation_predelegation_use` | `/payload/issued_at` | `/payload/revocation_snapshot_ref` | `role.operation-predelegation-use-signer` |

三schemaはOwner `mirakan.arch.ai-security-approval`のclosed Draft 2020-12 rootで、AI Security正本の各Payload／Wrapper／Refをexact投影する。Local Schema Catalog annotationとrowは各々`authority_binding_time=runtime_instance`、`issued_at_source_kind=payload_field`、`revocation_context_kind=payload_field`、`authority_source_kind=trust_registry`を持つ。Authority Binding Source Catalogは同じschema／slot／branch／purpose／time／pathをbyte-exact複写し、subject derivationをApproval=`exact_ref_field_v1(/payload/approver_subject_ref)`、Grant=`exact_ref_field_v1(/payload/delegator_subject_ref)`、Use=`exact_ref_field_v1(/payload/policy_service_subject_ref)`へ固定する。transport signing Role assignmentのscope derivationは`rfc8785-jcs-sha256-v1`で、Approval=`{/payload/approver_role_ref/role_ref,/payload/approver_role_ref/role_entry_sha256}`、Grant=`{/payload/delegator_role_ref/role_ref,/payload/delegator_role_ref/role_entry_sha256}`、Use=`{/payload/policy_service_role_ref}`だけをpath byte順closed objectへ投影する。Project、Operation、intent、expected revision、Grant budget等をtransport Role assignmentごとにTrust changeしてはならず、それらは同subjectの資格Role assignmentと後述のApproval／Grant／Use semantic verifierがwrapperごとに検査する。array index、display Role、Project名、caller文字列を導出入力にしない。

固定transport signing Role rowは次のexact三件である。各`permission_ids[]`と`allowed_signed_record_purposes[]`は示したsingleton、`independence_class=none`であり、quorumの独立性をtransport Roleから導出しない。

| role_ref | permission_ids[] | allowed_signed_record_purposes[] |
|---|---|---|
| `role.operation-mutation-approval-signer` | `[permission.sign.operation_mutation_approval]` | `[operation_mutation_approval]` |
| `role.r2-operation-predelegation-grant-signer` | `[permission.sign.r2_operation_predelegation_grant]` | `[r2_operation_predelegation_grant]` |
| `role.operation-predelegation-use-signer` | `[permission.sign.operation_predelegation_use]` | `[operation_predelegation_use]` |

`OperationMutationApprovalV1.signed_record`のSigner subjectは`payload.approver_subject_ref`、signing Roleは固定`role.operation-mutation-approval-signer`とする。`payload.approver_role_ref`はquorum資格を表す別`AuthorityQualificationRoleRefV1`で、同じhuman subjectが発行時と検証時のTrust closureでそのexact row hashを持つcurrent assignmentを持つことを検証する。GrantもSigner subject=`payload.delegator_subject_ref`、signing Role=`role.r2-operation-predelegation-grant-signer`とし、`payload.delegator_role_ref`を同型の委任資格として別検証する。UseはSigner subject=`payload.policy_service_subject_ref`、signing Roleと`payload.policy_service_role_ref`を共にexact `role.operation-predelegation-use-signer`へ固定する。これによりCatalogのRoleをpayloadから推測せず、多様な資格Roleを署名transport Roleと混同しない。

三wrapperすべてで`signed_record.subject_sha256 = SHA-256(RFC 8785 JCS(payload))`とし、payload全Fieldを署名subjectにする。`signed_record.purpose`、`issued_at`、`revocation_snapshot_ref`、Signer subject、signing Roleは、上表schema slot、payload Field、Authority Binding Source Catalogから導出した値の三者でbyte equalityにする。payload部分hash、semantic intent hashだけの署名、別schema／slot／purposeへのcross-schema replayを拒否する。

既存六Trust Registryだけを使う。Task 0 genesisはpre-ceremonyに固定したfull Local Schema Catalog／Authority Binding Source Catalogと、そのrequired fixed Role unionを一度だけprovisionし、上表三transport signing Roleを必ず含める。Task 2はCatalogが固定した三schema artifact bytesとsemantic verifierをmaterializeしてCatalog ref／hashへbyte-exact一致させるだけで、Catalog member、Authority template、fixed Roleを追加・変更しない。Engine routeのOperation runtime Activationは三Catalog member、三runtime-instance template、上表三fixed signing Roleをread-backし、未Provisionまたはbyte差なら失敗する。その後、Core baseline qualification-only Role exact六rowを未登録の場合だけ一つの承認済み通常Trust Changeで追加し、Receipt Serviceのactual Identity／transport Assignment／singleton-purpose Key、Foundation operation Signer Policy、Technical Qualificationを同じ承認済みchange／rebaseline chainでcurrent化してread-backする。六qualification Roleの一部だけの追加、既存row上書き、Task 2またはEngine ActivationによるCatalog／fixed Role再発行を拒否する。

Role rowはglobal logical rowでありhuman subjectごとに複製しない。将来Projectの全human subjectをEngine route Activation時に要求または捏造しない。Project／Task onboardingの通常Trust Changeで0～N件追加するのはactual subjectごとのIdentity、qualification Assignment、別transport signing Assignment、singleton-purpose Keyであり、既存六qualification Role rowを再追加しない。wrapper発行／dispatch前に全rowがcurrent Trust closureへ存在しなければならない。特定Projectで「Approval経路が利用可能」とclaimする場合だけ、そのProjectのRisk Policyが要求するquorum groupを満たすactual subject集合とscope assignmentをActivation evidenceへ含める。Catalogにslotがあるだけ、Roleだけ、Keyだけ、MCD `status=active`だけでは運用権限にならない。

Approval／Grantの資格Role assignmentは存在確認だけでなく、再計算したProject descendant、Operation ref、typed subject scope、Risk／authority classをassignment scopeがcoverすることを必須にする。UseのPolicy Service assignmentはGrant、Operation、Project、typed scopeとHead CAS service boundaryをcoverしなければならない。transport signing Role assignmentは上表purposeの署名可否だけを表し、資格Roleのscope coverage、quorum、independence、Grant semantic coverageを代替しない。

状態変更Operationの認可presenceは次のtagged contractに一意化する。

```text
MutationAuthorizationBindingV2
  operation_intent_hash: SHA-256
  risk_class: R2 | R3 | R4 | R5
  request_hash_algorithm_binding:
    exact OperationRequestAlgorithmBindingV1
  authorization_ref: TaskAuthorizationEnvelopeRefV1
  authorization_hash: SHA-256
  authority_evidence:
    approval:
      approval_set_ref: OperationMutationApprovalSetRefV1
      approval_set_hash: SHA-256
    | predelegated:
      predelegation_ref: OperationPredelegationUseRefV1
      predelegation_hash: SHA-256
```

`operation_intent_hash`は[Executable contracts §8.1](../02-foundation/executable-contracts.md#81-project-runtime-entryruntime-scopeの正規operation登録)の`MIRAKAN_OPERATION_INTENT_V2`だけで再計算する。Task Authorizationはintentを直接署名せず、解決したTask／Operation／Project／typed subject／Path／Budget scopeがintentをcoverする。Approval Set内の全ApprovalまたはPredelegation Useだけが同じexact intent hashを署名subjectへ含め、後段の`request_hash`をsubjectにしない。Policy Serviceはbindingのintent hash／risk、exact typed Authorization ref／hash、選択evidence ref／hash、`request_hash_algorithm_binding`を解決先とbyte equalityで照合する。R2は`approval | predelegated`の厳密に一方を許可し、Predelegation Useは上記Grant Scope、Operation、Project descendant chain、期限、上限、rollbackとoperation intentを全てcoverしなければならない。R3～R5は`approval`だけを許可し、`predelegated` branchを拒否する。authoritative Project／Source／Release状態変更で`authority_evidence`欠落、両branch混在、quorum不足、期限切れ、Scope不足、intent hash／risk／Algorithm binding不一致の場合は、Domainにかかわらずexact `DiagnosticCodeRefV1`の`diagnostic.approval.required / MIRAKAN-APPROVAL-REQUIRED`を返す。裸ref、`authorization_ref`だけ、optional individual `approval_ref`、Grant単体、空hash、callerの「承認済み」文字列を代用しない。本型はR2～R5のauthoritative mutation専用である。R0 queryとauthoritative stateを変えないR1はcanonical omissionし、将来のR1 Task control等はそのfamilyが完全登録する別typed control authorizationを使って本型へ暗黙昇格しない。

公開ReceiptのAuthorization auditは[Executable Contractsの`OperationAuthorizationAuditBindingV1`](../02-foundation/executable-contracts.md#8-operation定義)だけを使用する。本書は別の認可意味論を追加せず、完成final requestのinline `MutationAuthorizationBindingV2`とAudit BindingのOperation／input Type／risk／intent／request Algorithm binding／Task Authorization ref・hash／Approval SetまたはPredelegation Use ref・hashをbyte equalityにする。Audit Binding objectとref／hash commitmentはPublic Markerから到達可能にするが、request／Authorization／ApprovalのPII bodyはpublic inlineせずimmutable access-controlled CASへ保持する。public verifierはcommitment、認可済みAudit verifierはbodyとscope／quorumまで検証する。Audit Binding欠落、両evidence branch、Context作成後の差替え、hash一致しないbody、PII bodyの公開graph混入を拒否する。

### 4.2 Activation Gate

| Activation | 有効にする範囲 | 解放条件の要点 |
|---|---|---|
| A0 Authoring Core | R0–R2構造化編集とGameplayDefinition | 契約、署名Envelope、Gateway、Transaction、Validator／Cooker、Undo、adversarial security fixture |
| A1 Project Source | R3 bounded NativeGameModule、Project Shader、Engine-owned Build template | hardware-VM Worker、Broker、Promotion、Path／Network／Process negative test、G0–G7、署名Attestation |
| A2 Engine Maintenance | 別Repository／AuthorizationのR4 Engine保守 | Game制作DAG外。独立Review、state model、threat／lifetime、full regression、fault／soak |
| A3 Release | R5 merge／tag／sign／Store提出 | 分離Coordinator／Build／Signing／Upload、reproducible Build、SBOM、Provenance、Platform／device Gate |

未解放OperationはTool catalogへ出さず、内部呼出しにもMIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATEDを返す。A3はA2を暗黙依存しないが、R3 Artifactを含むReleaseはA1を必要とする。

## 5. Beginner questions、assumptions、理解条件

上位理解recordの許可集合は`GameIntentDraftV1`、`GameBriefV1`、`GameSpecDocumentV1`、`QuestionRecordV1`、`AssumptionRecordV1`、`DecisionRecordV1`、`GameUnderstandingClosureV1`、`AiCatalogEntryV1`、`AiTaskContextCapsuleV1`のexact九Schemaだけとする。全Field required、明示`| null`だけnullable、tagged unionはexact一branch、unknown Field禁止、配列boundsとcanonical order／unique、typed refのID／schema version／content hash exact解決、`MIRAKAN_CLOSED_RECORD_V1/<SchemaName>`の自己Field除外hashを本節の共通規則とbyte equalityにする。同名shadow schema、optional補完、自由文からのOperation／Authority／Target／Provider／Model／Host／Deployment推測を追加しない。

`GameIntentDraftV1 -> GameBriefV1 -> GameSpecDocumentV1 -> GameUnderstandingClosureV1`は各content-addressed refで一方向に連結する。Briefはhuman確認subject／時刻、SpecはProject snapshot、ClosureはBrief／Spec／Question／Assumption／DecisionとRequirementからArtifactまでのEvidence closureをexact hashで束縛する。`AiCatalogEntryV1.production_owner_layer=core | feature_pack | genre_pack | game_project`はproduction ownership、`artifact_role=production | cross_cutting_control_plane | reference_game | fixture | qualification`はartifact用途であり、二軸を混同しない。Provider／Model／Host／Deployment Profileも独立四ref集合とし、brand別schema branchを禁止する。`AiTaskContextCapsuleV1`はTask Authorizationとactive Operation refの積集合を持つread-only／Disposable projectionで、Capsule自身、自由文のselection reasonまたはcontinuation tokenは権限にならない。

質問、Default、DecisionのSecurity-owned stateは次のclosed shapeである。Scalar、Ref、presence、ordering、hash規則は上記九Schema共通規則を使用する。

```text
QuestionRecordV1
  schema_version: 1
  question_id: StableId
  project_ref: ProjectSnapshotRefV1
  subject_ref: TypedArtifactRefV1
  question_class: blocking | high | medium | low
  question_text: Text<2048>
  impact_text: Text<2048>
  resolution:
    open: {}
    | answered:
        answer_text: Text<4096>
        answer_evidence_refs[0..32]: EvidenceArtifactRefV1
        decision_ref: DecisionRecordRefV1
        answered_by_subject_ref: TrustSubjectRefV1
        answered_at: UtcTimestamp
    | withdrawn:
        reason_text: Text<2048>
        decision_ref: DecisionRecordRefV1
        withdrawn_by_subject_ref: TrustSubjectRefV1
        withdrawn_at: UtcTimestamp
  created_at: UtcTimestamp
  content_hash: Sha256

AssumptionRecordV1
  schema_version: 1
  assumption_id: StableId
  project_ref: ProjectSnapshotRefV1
  source_question_ref: QuestionRecordRefV1(question_class=medium,resolution=open)
  assumption_class: medium_default
  default_value_ref: TypedAssumptionValueRefV1
  default_summary: Text<2048>
  default_basis_refs[1..32]: EvidenceArtifactRefV1
  valid_from: UtcTimestamp
  expires_at: UtcTimestamp
  revalidation_condition_refs[1..32]: RevalidationConditionRefV1
  acceptance:
    proposed: {}
    | accepted:
        acceptance_decision_ref: DecisionRecordRefV1
        accepted_by_subject_ref: TrustSubjectRefV1
        accepted_at: UtcTimestamp
    | superseded:
        replacement_assumption_ref: AssumptionRecordRefV1
        supersession_decision_ref: DecisionRecordRefV1
        superseded_at: UtcTimestamp
  content_hash: Sha256

DecisionRecordV1
  schema_version: 1
  decision_id: StableId
  project_ref: ProjectSnapshotRefV1
  decision_kind:
    experience | requirement | constraint | implementation |
    target_profile | capability_disposition
  subject_ref: TypedArtifactRefV1
  selected_option_ref: TypedDecisionOptionRefV1
  rejected_option_refs[0..32]: TypedDecisionOptionRefV1
  rationale_text: Text<4096>
  evidence_refs[0..64]: EvidenceArtifactRefV1
  decided_by_subject_ref: TrustSubjectRefV1
  decided_at: UtcTimestamp
  content_hash: Sha256

GameUnderstandingClosureV1
  schema_version: 1
  closure_id: StableId
  project_ref: ProjectSnapshotRefV1
  game_brief_ref: GameBriefRefV1
  game_spec_ref: GameSpecDocumentRefV1
  question_record_refs[0..512]: QuestionRecordRefV1
  assumption_record_refs[0..256]: AssumptionRecordRefV1
  decision_record_refs[1..1024]: DecisionRecordRefV1
  requirement_set_ref: RequirementSetRefV1
  requirement_traceability_ref: RequirementTraceabilityRefV1
  system_graph_ref: SystemGraphRefV1
  state_owner_map_ref: StateOwnerMapRefV1
  capability_scope_ref: CapabilityScopeRefV1
  save_replay_contract_set_ref: SaveReplayContractSetRefV1
  target_profile_refs[0..64]: TargetProfileRefV1
  behavior_budget_refs[1..128]: BudgetProfileRefV1
  test_plan_ref: TestPlanRefV1
  evidence_closure_ref: EvidenceClosureRefV1
  project_shader_understanding_closure_refs[0..256]:
    ShaderUnderstandingClosureRefV1
  unsupported_capability_refs[0..256]: CapabilityRefV1
  unresolved_blocking_question_count: uint16
  unresolved_high_question_count: uint16
  unresolved_medium_question_count: uint16
  unresolved_low_question_count: uint16
  disposition: ready_to_stage | capability_unavailable
  content_hash: Sha256
```

質問を次に分類する。

| Class | 動作 |
|---|---|
| Blocking | 回答まで該当Scopeの実装開始不可 |
| High | 推奨案、影響、変更可能性を示し回答を求める |
| Medium | typed Default、根拠、期限、再検証条件を持つaccepted Assumptionを明示して進行可 |
| Low | Project規約から決め、Decisionへ記録 |

初回はBlockingとHighを最大7問へまとめる。超える場合はGame core、Visual、Platform／Businessの順に分割する。初心者向けUIは複雑さを隠せるが、質問、仮定、Risk、Approval、制限を省略してはならない。

大まかなPromptはGameIntentDraft、Capability／Platform／権利／Online／Save／年齢／Performance検査、質問、GameBrief、人間確認、First Playable Scope、実装方式、Staging、Gate、Approvalの順で処理する。

AIの「理解した」という自己申告を状態にしない。`QuestionRecordV1.resolution`はopen／answered／withdrawnのexact一branchで、回答文字列だけ、callerの「回答済み」、Decision refなしのansweredを拒否する。`AssumptionRecordV1`はMedium open質問だけをsourceにし、typed `default_value_ref`、根拠1件以上、有限の`expires_at`、再検証条件1件以上を全て必須にする。根拠0、無期限sentinel、期限切れ、再検証条件0、Closure Question集合のsame-ID different-hash、同一質問への複数active assumptionでは進行しない。条件成立またはEvidence freshness切れ時は、期限前でも新Decision／Assumption hashを発行して再検証する。

`GameUnderstandingClosureV1`は終端Recordであり、Blocking／High／Low openを0件にする。Medium openは同じQuestion refに対する`acceptance=accepted`かつ`valid_from <= evaluation time < expires_at`のAssumption exact一件だけを許可する。四countは解決済みQuestion集合から再計算し、Brief／Spec／Question／Assumption／DecisionのProject lineageと全ref hashを一致させる。`disposition`へ第三のstateや自由文字列を追加しない。ready_to_stageには上記質問closure、RequirementからCapability／Pack／System／Implementation／Test／Artifactまでの追跡100%、State owner重複0、stale Evidence 0、unsupported required Capability 0が必要である。Project Shaderを含むSystemは全参照Module／Techniqueに有効な`ShaderUnderstandingClosureV1`と`ProjectShaderQualificationReceiptV1`を必要とし、欠落／stale／Target不一致をGame全体の理解で相殺しない。必須Requirementに未対応Capabilityがあればcapability_unavailableにするが、質問／Assumption closureを緩和しない。

## 6. Project data、Project C++／Shader／Native module

### 6.1 Immutable Engine boundary

Game制作時のEngineを署名済み、content-addressed、read-only baselineに固定する。Projectはbaseline IDだけでなくEngine、Editor、GameHost、Public SDK、Capability set、Schema／compiler、Validator、Engine-owned Test、Policy、Toolchain、Target Profile、File manifestのroot hashをProjectLockV1へ固定する。

- Engine sourceをProject workspace、AI Context、Source Worker Bundleへ入れない。
- Engine package、SDK、Validator、Policyをread-only mountする。
- engine、editor、runtime、toolchain、policy、Engine-owned schemas／testsへのwriteを拒否する。
- Build、Preview、Packageの前後で署名と全hashを再検証する。
- Project Build scriptによるinclude／link path、Validator、Compiler option上書きを拒否する。
- Baseline mismatch、未知／欠損FileをMIRAKAN-ENGINE-BASELINE_MISMATCHで停止し、AIに修復させない。

Game制作Tool catalogへEngine source patch、Engine module／Extension／Adapter、private API、Validator／Policy／Test変更、Engine Git／Release、binary／signature差替えを登録しない。新しい署名済みEngine Releaseへの更新は明示Migrationであり、その場のEngine変更ではない。

### 6.2 許可するProject artifact

- Game Brief、GameSpec、Requirement、Decision。
- GameSystemSpecV2、Project-defined System Contract。
- GameplayDefinition、World、Level、Scene、UI、Asset、Material、Animation、Audio設定。
- Project Test、Fixture、Benchmark、Replay、Save migration。
- BoundedNativeGameProfileV1に適合するNativeGameModule。
- [Project Shader](../06-rendering/project-shader.md)の`BoundedProjectShaderProfileV1`に適合する`ProjectShaderModuleV1`／`ProjectShaderTechniqueV1` Source、`ShaderFactGraphV1`、`ShaderUnderstandingClosureV1`、Target別`ProjectShaderArtifactSetV1`。
- 生成Bindingの入力となるProject Contract。GeneratorとEngine-owned Schemaは変更不可。

Gameplay／System実装は次の順で検討する。

    既存System／Capability composition
      -> GameplayDefinition
      -> Cook／index／layout最適化
      -> Bounded NativeGameModule
      -> capability_unavailable

Material／Rendering実装は次の順で検討する。

    既存Material／Template／Pass Capability composition
      -> Material Instance／closed typed Graph
      -> Project Shader Function／Node／Library
      -> Project Shading Model／Stage Module
      -> declarative Project Shader Technique／Project Renderer Feature
      -> capability_unavailable

Engine Extension、Engine Adapter、Engine core変更をGame制作のfallbackにしない。

### 6.3 Bounded Native

Project C++はgenerated public SDK、C ABI、許可Primary Moduleだけを使う。次を禁止する。

- Engine private、Platform、Vendor、Editor header／object／pointer／container／allocator／native handle。
- File、socket、process、environment、registry、OS service、dynamic library、native SDKの直接access。
- inline assembly、未知link import、自己書換え、runtime code generation。
- reinterpret cast、所有raw pointer、直接new／delete、malloc／free、custom allocator。
- 独自Thread、mutex、condition variable、TLS。
- wall clock、entropy、global mutable state。
- ABI境界を越えるException、外部Dependency。
- 未宣言Component、phase、queue、scratch、CPU／Memory、State owner、Save／Replay意味。

Source scanだけに依存せず、AST、Module graph、object import、link map、binary import、runtime traceを照合する。C++はmemory-safeを証明された言語ではないため「絶対安全」と表示しない。

### 6.4 Bounded Project Shader

Project ShaderのSource／semantic／resource／pass／AI理解／qualification境界は[Project Shader](../06-rendering/project-shader.md)だけが定義する。ProjectはProfile内でHLSL Function／Node／Shading Model／Stage、宣言済みStorage／UAV相当access、複数Pass Techniqueを追加できるが、Manifest外Pass／Resource／side effect、native GPU resource／descriptor／barrier／queue、Compiler option、Engine private includeへ到達できない。

Targetごとにhardware-VMの隔離Workerでoffline compile／validateし、Source contract、compiler fact、reflection、runtime-use trace、fixtureを照合する。Shipping RuntimeでSource生成、download、JITを行わない。既存`ProjectRenderDomainPortV1`で表現できない新Domain、Port自体、Backend、native execution primitiveが必要なら`capability_unavailable`にする。

## 7. Sandbox、Filesystem、Network、Credential

### 7.1 Source Worker

Workspace Brokerが許可base commitからHost Stagingを作り、SourceBundleV1をguestへ転送する。WorkerへHost path、.git、branch／commit、main worktree、User home、Credential directory、他Projectを公開しない。返却SourceDeltaV1をuntrusted byte列として検査し、BrokerがStagingへ適用後に新Diffを計算する。

Bundle／Deltaはnormalized Repository-relative path、before／after hash、mode、size、content-addressed blobを持つ。symlink、junction、reparse point、hardlink、ADS、device、sparse file、submodule escape、case collision、reserved device nameを拒否する。任意archiveやguest filesystemをHostへ直接展開しない。

Windows A1／A2のPromotion可能BackendはHyperVIsolatedWorkerV1または同等のremote hardware-VM Workerとする。Generation 2、Secure Boot、immutable base image、Taskごとの一段differencing disk、no NIC、no host mount、no clipboard／device、Task完了後disk破棄を必須にする。「同等」の判定は同じ必須特性のconformance fixture合格で行い、自己申告にしない。Host／guest間はTask nonce、Envelope hash、Bundle hashを照合するbounded protocolだけを使う。

Networkはdeny_allを既定とし、承認済みcontent-addressed mirrorをInputにする。例外はBroker Proxyのexact domain allowlist、DNS rebinding／Redirect／private address／response size検査を必要とする。Sandbox unavailable時は非隔離へfallbackせず停止する。

SecretをEnvironment、command line、working tree、Context、Tool output、Log、crash dumpへ渡さない。WorkerのProcess tree全体へwall time、CPU、commit memory、child count、output sizeを強制し、timeout時は子Processを含め終了する。

### 7.2 Release credential separation

Source／Build scriptを実行するProcessへSigning keyまたはStore Credentialを渡さない。

    Release Coordinator
      -> ephemeral Release Build Worker [sourceあり、secretなし]
      -> approved unsigned artifact
      -> Platform Signing Service [sourceなし、signing keyあり]
      -> verified signed artifact
      -> Store Upload Service [signing keyなし、短命upload credentialあり]

Signing ServiceはManifest外File、hash差、path traversal、未知Executable、未許可Entitlementを拒否し、固定Operation以外のshell／hookを持たない。Keyはnon-exportable handleとし、one-shot transactionへ限定する。署名後は鍵を持たないVerifierが許可差分だけを確認する。

Upload Serviceはsigned artifactとEvidence chainだけを受け取り、Project source、Build script、Signing keyを持たない。CredentialはTask、Application、Channel、Versionへ限定した短命最小Roleとする。

### 7.3 Trusted service isolation

Contract set内のTrusted Serviceは「信頼済み」という名称だけで無制限なHost権限を得ない。Executable Contractsのcurrent一Profileと、legacy evidence成立後だけ登録可能なconditional一Profileは、次のclosed payload schemaをexactに使用する。

```text
TrustedServiceIsolationProfileV1
  profile_kind: trusted_service_isolation
  base_profile_local_ref: ContractSetLocalRefV1(kind=profile) | null
  execution_mode:
    in_process_trusted_service | isolated_offline_worker
  network_access: denied
  engine_baseline_access: read_only_exact_lock
  project_store_access:
    transactional_authoring_changeset | read_only_snapshot
  staging_access: private_transaction_candidate | none
  project_source_access: none
  host_filesystem_paths[0..0]: []
  input_artifact_access: exact_content_refs_only
  output_policy:
    domain_publication_pipeline | job_result_bundle_only
  ephemeral_scratch: brokered_bounded_private
  process_spawn: denied
  environment_access: none
  credential_access:
    brokered_non_exportable_operation_purpose_key | none
  secret_export: denied
```

current exact Profileは次の一recordだけである。`base_profile_local_ref=null`で、Field省略、環境既定、Profile継承による権限追加を許可しない。

| Profile LocalRef | execution | Project Store | Staging | output | Credential |
|---|---|---|---|---|---|
| `{profile.isolation.authoring_command_gateway,1}` | `in_process_trusted_service` | `transactional_authoring_changeset` | `private_transaction_candidate` | `domain_publication_pipeline` | `brokered_non_exportable_operation_purpose_key` |

表で省略した共通Fieldはschema literal、すなわち`network_access=denied`、`engine_baseline_access=read_only_exact_lock`、`project_source_access=none`、`host_filesystem_paths=[]`、`input_artifact_access=exact_content_refs_only`、`ephemeral_scratch=brokered_bounded_private`、`process_spawn=denied`、`environment_access=none`、`secret_export=denied`へexact一致させる。Gatewayのbrokered Keyは許可Operationと署名purposeへ限定したnon-exportable handleであり、可搬Secret、汎用sign、Provider credential、Release keyへ昇格しない。

`profile.isolation.offline_project_migrator@1`、対応Service／Capability、Trust assignment／Keyのcurrent集合はexact `[]`である。Executable Contracts §8.1.2のsigned legacy inventoryを満たすmigration Operation、または将来の汎用schema migration Operationをatomic activateする場合だけ、このProfile候補を同じContract set／Trust transactionで初めてmaterializeする。activation後の`offline`はnetwork、任意Process、Host filesystem、Environment、Project Sourceへのaccessを持たない隔離実行modeを意味し、read-only jobを意味しない。Serviceは自身がauthorityである実allowlist Operationだけについて、Brokerが発行するOperation／request／Project／期限／署名purpose限定のnon-exportable handleから一つの`transactional_authoring_changeset`を開始し、入力Project snapshot、Contract set、Owner contribution、Policy、Authorization、Approval、idempotencyを再検証する。state-changing migrationのpublicationは[Executable Contracts §8](../02-foundation/executable-contracts.md#8-operation定義)をcanonical reuseし、`private transaction candidate → postcondition → private Marker read-back → secret-free PublicCommitClosureV1 candidate → signed wrapper read-back → PublicCommitClosureV1＋PublicPublicationMarkerV1＋after stateのatomic CAS`を自Service境界内で完遂する。Closureの`domain_commitment.kind`は`owner_typed_state_commit`とし、exact allowlisted Operation ownerおよびPrepared payloadが束縛したreceipt-free committed artifact ref集合を保持する。Closure Ref／hash規則は同節を再利用して本書で複写せず、Closure bodyまたは同Closureを束縛するsigned wrapperを欠くPublic Marker／after-state current authorityを拒否する。空allowlist、四legacy候補の一括推測activation、Profileだけの先行登録を拒否する。

このProfileはProject Storeの任意write、raw database handle、current pointer直接write、Gateway credential共有、任意Receipt署名、別Operationへの権限昇格を許可しない。Brokerはallowlist外Operation、request／Project／purpose差、期限切れ、別materialization contextでのhandle再利用を拒否する。同じtransaction、同じ`receipt_materialization_key`、同じContext／subject hash／Key IDのcrash retryだけは例外で、Brokerのdurable idempotency journalから既存のbyte-identical RFC 6979 signature／wrapperを返し、再署名、issued-at更新、Key再選択を行わない。journal recordの作成はsignatureをServiceへ返す前にdurableとし、同じkeyでContextまたはsubjectが一byteでも異なる場合はintegrity faultとして停止する。Migratorはtransaction API以外からProject current head、Receipt Store、Public Marker、Registry current pointerを変更できず、失敗時はpublic objectを0件にしてSource／before stateを維持する。結果bundleだけを外部Authorityとして採用する経路、およびGatewayが未登録の暗黙apply Operationで再公開する経路は存在しない。

Service identityのauthorityはControl Planeが所有する既存六Registry、すなわち`IdentityRegistryV1`、`AuthorityRoleRegistryV1`、`AuthorityRoleAssignmentRegistryV1`、`AuthorityPublicKeyRegistryV1`、`AuthorityRevocationRegistryV1`、`GovernancePolicyConfigRegistryV1`とexact 15-member `TrustRegistryClosureV1`だけで構成する。新しいServiceを追加する通常差分はIdentity、Role、generic Assignment、Public Keyの四Registryへ各exact一rowを追加し、signed purposeをRoleとKeyのsingleton集合へ同じ値で保存する。実際の失効またはPolicy変更がない限りRevocation／Policy Registryのrow差分は0件である。`activation.foundation.operation_domain_receipt_pipeline.v1`はService追加だけでなく明示的なPolicy activationでもあるため、Signer Policyをemptyからbaseline六rowへ更新し、Executable Contracts OwnerのPublication Control Policy exact三rowを同じtransactionで追加する例外であり、通常Service追加の0-row規則から暗黙導出しない。`activation.installed_product.operation_composition_extensions.v1`は既存Publication Control Policy三rowをbyte-exact保持し、Signer Policy／対応Role・assignment・Keyだけをbaseline六からInstalled十へ`N+1`更新する。別のPurpose Registry、Service Role Registry、Profile内の自己署名Roleを追加せず、assignment IDとKey IDはControl Planeの既存導出規則を使う。

Profile MCD recordの`status=active`はContract setで参照可能な隔離契約を意味するだけで、Service executable、Key provisioning、Trust Registry変更、Product Capability、Target Qualificationがoperationalに有効であることを示さない。Service起動時はProfile local ref、Service member hash、Contract set root、current Product operational headのBaseline binding、同Bindingが解決するFoundation Closure、Trust closure、Governance Policy Config Registryが選択するcurrent `OperationReceiptSignerPolicyRefV1`、executable identity、Key purposeを同時にread-backする。

Service `authority_capability_refs[]`のactivation predicateはProduct selectionとservice authorityを混同しない。Capability IDがcurrent Product `CapabilityRegistryV1`に存在する場合は、実行TargetのActivation rowとActivation Policyが要求するfresh Technical Qualification Receiptをexact ref／hashで解決し必要state以上にする。Product rowが存在しない場合はProduct Capabilityを補完せずservice-authority branchとする。read-time `operational contribution`集合は、current Installed Product compositionの`contribution_origin_ref`単位で、同じOriginのOperation全件がcurrent Signer Policyに同Service subject／exact Role／purposeで存在し、Service full allowlistに含まれ、同じCandidateのfresh Service executable／Profile／Contract set／Target Technical Qualificationを持つOriginだけから導出する。選択Originの`OwnerOperationContributionV1.operation_local_refs[]` unionとcurrent Signer Policy中の同Service Operation集合をset equalityにし、Service full allowlistはそのunionのsupersetとする。Origin内の一部だけ、Signerだけ、allowlistだけ、別Candidate Qualificationをoperational contributionにしない。初回はProject State baseline Origin一件／六Operation、extension transaction後はProject State＋World＋Shooter三Origin／十Operationがexact集合である。dispatch対象Operationはこのselected unionのmemberでなければならず、full allowlistに存在するだけのcontract-active extensionを実行しない。Product rowなしをProduct support／shipping表示へ投影せず、consumer Operationが別Product Capability／Gateを要求する場合はそのdispatch Gateも独立に検証する。どちらのbranchでも一件でも欠落／staleならfallbackせず停止する。起動時成功をleaseまたは永続権限にせず、各dispatchでもTask AuthorizationにpinしたBaseline binding、current Signer Policy row、該当Productまたはservice-authority branch、Qualification freshness、Trust revocationを再検証する。

## 8. Provider API、MCP、CLI、Plugin

| 接続 | 公式用途 | 権限にならないもの |
|---|---|---|
| Provider API | 製品内Chat、質問、計画、構造化Proposal | Project state、Authorization、Schema |
| MCP（local STDIO／Activation済みStreamable HTTP） | 外部ClientのQuery／Proposal | Commit、Activation、Provider設定 |
| conformance済みCLI Agent Host＋Worker | 隔離Source編集、Build、Test | main branch、Release、Credential公開 |
| Optional Plugin | Shortcut、Panel、Prompt／Skill UX | Engine必須機能、Security Policy |

PluginなしでもAPIとMCPで全公式操作を行えること。外部AI ClientはRepositoryと正規Project storageをread-onlyにし、Query／Proposalだけを使う。full-accessで直接起動したClientは公式Security boundary外であり、その変更はExternal patchとしてRisk再分類、全Gate、Approval、Promotionを通すまで正規入力にしない。

Credential ownerを分離する。

- 製品内AI: Provider AdapterだけがCredentialを持つ。
- 外部Client＋MCP: CredentialはClient／Userが持ち、Engineは受け取らない。
- Managed CLI: conformance済みHostまたは専用Brokerが持ち、File／Shell childへ渡さない。

local STDIO MCP serverはOS ACLで束縛したIPCをGatewayへ接続し、Engine／Project Credentialをenvironmentへ渡さない。外部Provider CLIが公式にenvironment tokenだけを受ける場合は、BrokerがTask専用childへだけ短命tokenをprocess creation時に注入し、persistent／global user・machine environment、config file、command line、log、crash dumpへ保存せず、Tool child／grandchildへ継承させず終了時に失効するProfileだけを例外許可する。「plain Environment禁止」はこのBroker外の永続／広域露出を指す。条件を満たせなければmanaged modeを禁止しproposal-onlyにする。

Managed modeは`ExternalClientSecurityProfileV1`のidentity branchに応じ、localではClient exact version／binary ref・hash／OS identity／Process・Filesystem・Network分離、hostedではsurface／release channel／exact release／tenant-workspace／plan／admin policy／OAuth-OIDC subject・audience／TLS identityを固定し、共通でMCP version、期限、required Conformance Suiteを宣言する。Profile自体はresultを持たず、同じProfile Head／Tool Projection／Targetへ束縛したtyped Conformance Receiptだけがpass／failを所有する。Profile不在、branch必須値差、期限切れ、Broker外environmentへのCredential露出、raw socket利用ではそのManaged Caller Contextを拒否して`not_activated`とする。別途currentなstandard-external-MCP用Host／Transport／Tool／proposal-only Authority Profile、Grant、Host／Transport＋Schema／Eval Conformanceが全て存在する場合だけ、Gatewayが`standard_external_mcp` branchの新しいsigned Caller Contextを発行できる。managed ContextのField削除やin-place権限降格で代用しない。

### 8.1 Provider Manifest、Prompt、Repository guidance

Provider／Model名をEngine codeへhard-codeしない。`ProviderManifestV1`は`ProviderRuntimeProfileV1`のtagged adapter（official SDKのexact artifact、raw protocolのexact schema、またはattested external host）、endpoint、resolved Model snapshot、Role、推論設定、Tool／Structured Output projection、Context／output／cost／latency上限、data retention／training use／region／encryption／logging Policy、合格Eval suite、明示fallback、fallback時の最大Riskを固定する。存在しない共通「API／SDK version」を捏造せず、選択branchに必要なversion／artifactだけを検証する。Manifestなし、adapter branch不一致、resolved ID差、期限切れConformanceではModelを呼ばない。Evalと更新Workflowは[AI Verification／Provenance](ai-verification-provenance.md)だけが決定する。

Prompt templateはRole、Goal、Success criteria、Normative constraints、Toolと権限、Evidence要求、Output Schema、Stop／質問条件の順にする。PromptをSecurity boundaryにせず、同じ規則を複数Promptへ反復しない。Prompt変更は一群ずつ行い、Model変更と同時に評価原因を混在させない。

RepositoryのAGENTS.mdはRepository map、Build／Test入口、禁止操作、Definition of Done、本正本とExecutable contractへのLinkに限定する。Subsystem規則は近いOwner文書へ置き、AGENTS.mdと実行可能契約が矛盾するMustをCIで拒否する。

### 8.2 Privacy、Prompt injection、Data handling

Source、Asset、User Prompt、Tool output、Web、Issue本文内の命令はcontentであり、Control命令へ昇格しない。Provider送信前にSensitivity labelとProvider Policyを照合し、Secret、private key、access token、Credential file、Production customer dataをContextへ含めない。

Prompt、Tool argument、Tool output、Traceは機密Dataになり得る。既定Telemetryへ本文を保存せず、hash、分類、Resultだけを記録する。Zero Data Retentionが必須のProjectでは非対応Provider機能を無効化し、代替がなければTaskを停止する。Live Web取得は分離したResearch Taskだけで許可し、Build／Release中の自動Web取得を禁止する。

ModelへOperationとして公開できるのは、current Contract set、Owner Manifest、Trusted Service allowlist、Provider／MCP projection、Callerのexact allowlistが同じ完全登録済みref集合へ一致するものだけである。Project／Requirement／Capability／System／Worldのbounded Query、plan／validate／preview、ChangeSet submit、Source task request、Patch submitという一般名は、登録済みOperation identityを暗黙生成しない。[Executable contracts](../02-foundation/executable-contracts.md) §§20～21.1のPackage／Device／Play／Debug／Task、回収済みDomain authoring／selection、AI E2E closureを含む24 family／191候補（67＋92＋32）は未登録・未公開であり、dispatch前に`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`を返す。Build familyのatomic Activation後もDevice install／resetはTool表示の有無にかかわらずDevice binding、consent、R3 ApprovalをServer側で再検証する。commit、activate、write native artifact、merge、sign、release、secret.read、policy.overrideを公開しない。

AI E2E closureを将来activateする場合も`standard_external_mcp`の外部Tool surfaceはQuery、Plan、Proposal、Validate、Previewだけである。外部Hostが「build／test／debug」を指示した場合、proposal-only Toolは希望stageをtyped Proposalへ記録するだけで、`operation.build.*`、`operation.test.*`、`operation.debug.*`を外部Caller Contextでdispatchしない。人間承認後、Engine-owned Orchestratorが別のtrusted internal Task Authorizationと各Operation固有AuthorityでBuild／Test／Debugを実行し、外部HostはQuery Toolで状態／Receiptを読む。`managed_external_host`の`build_job`は将来Activation後にHost自身が行ったmanaged job結果をBroker経由でStagingへ受理する別authority classであり、Engine-owned Build Operationの外部dispatch権限ではない。`operation.authoring.changeset.commit`と`operation.project_source.promote_revision`はexact `trusted_internal`、Provider／MCP／standard external CLI projection=`[]`とし、Approval発行、Candidate Activation、Signing、Release、Policy変更と同じくEngine-owned Gatewayだけが人間Approval後に実行する。`operation.authoring.changeset.propose`、Native／Shader Patch Proposal、Source Worker成功、Build成功をCommit／Promotion／Activationの代用にしない。外部routeのTool CatalogにBuild／Test／Debug／Commit／Promotionの実行Operation IDが一件でも現れる、Provider aliasから到達する、generic `apply_changeset`／`promote`名へfallbackする場合はCatalog全体を拒否する。

Product Capabilityの利用可否は[Product Plan §11.1](../00-product/product-plan.md#111-registry共通規則)の`AiCapabilityActivationMatrixV1`／`ExecutionSurfaceBindingV1`をcurrent Contract Setへ再解決し、必要family全OperationのMCD、Owner Manifest、Service allowlist、Policy、Validator／closure、Diagnostic、Receipt、Signer Policy、Trust、Technical Qualificationが同一revision／hashでoperationalの場合だけtrueにする。WP complete、fixture pass、planning record、MCP Tool表示だけからAI Capabilityを`qualified`にしない。Windows Editor HostでAndroid／Apple artifactを生成するrouteは`execution_host_target`と`artifact_target`を分離し、Target別Build／Test／Package Receiptを必須にする。

MCP annotationをAccess controlにしない。Serverは正本Schema、Authorization、timeout、rate limit、Auditを強制する。local STDIOはACL付きIPCをGatewayへ接続する。Streamable HTTPはexact `HostTransportConformanceReceiptRefV1`、TLS、認証、origin／redirect／private-address policy、session binding、別Threat ModelとActivationがある場合だけ有効にし、単なるport forwardingを許可しない。Tool公開には同じsubject tupleの`SchemaEvalConformanceReceiptRefV1`も必須にする。`McpSessionGrantV1`は署名済みEnvelopeを代替せず、Grant配下のbounded read／QueryもR0 Envelopeを必要とする。GrantはClientとchannelの束縛だけを追加する。

MCP 2025-11-25の標準initializeでAuthorityとして検証できるClient情報はprotocol version、capabilities、`clientInfo`等に限られ、上流Provider、Model、local binary hashの暗号学的attestationは得られない。標準外部MCP接続のprovider／model表示は`unattested_optional_metadata`としてAudit UIにだけ出し、Authorization、Eval attribution、Source／Build Receiptへ使わない。`managed_source_edit | build_job`を許す経路は、実行前にMCP外の登録済みBrokerがHost session／Provider runtime／managed deployment identity／Model snapshot／Tool projection／期限／nonceを署名したfresh `ManagedHostSessionAttestationV1`をCaller Contextへ束縛し、実行後に同じmanaged deployment identity、Context／Task／Authorization／attempt／Input closure／typed resultを署名した`HostExecutionAttestationV1`をSourceDelta／Build Receiptへ束縛する。Attestation不在時はManaged Contextを拒否し、別途standard routeのHost／Transport／Tool／proposal-only Authority Profile、Grant、`HostTransportConformanceReceiptRefV1`、`SchemaEvalConformanceReceiptRefV1`が全てcurrentな場合だけGatewayが新しい`standard_external_mcp` Caller Contextを発行する。事後resultを実行前Contextへ入れる因果循環、Host自己申告JSON、MCP annotation、Client名をAttestationに読み替えない。

### 8.3 Caller／Provider／Deployment／Model Profile

Caller互換性をHost brandだけで判定しない。正規判定は`execution_route.kind`をdiscriminatorにした次の3 branchだけである。`standard_external_mcp`は外部Host＋MCP Transport＋Tool＋proposal-only Authority＋MCP Grant＋`HostTransportConformanceReceiptV1`＋`SchemaEvalConformanceReceiptV1`を必須にし、上流Provider／Deployment／Modelと`ProviderToolConformanceReceiptV1`を必須`null`／unattestedとする。`managed_external_host`は外部Host＋MCP Transport＋Provider Runtime＋managed deployment identity＋Model＋Tool＋Authority＋`HostTransportConformanceReceiptV1`／`ProviderToolConformanceReceiptV1`／`SchemaEvalConformanceReceiptV1`のexact三Receipt＋fresh Session Attestationを使う。`engine_provider_adapter`はfirst-party Engine Host＋Provider Runtime＋Provider Manifest＋Inference Deployment＋Model＋Tool＋Authority＋`ProviderToolConformanceReceiptV1`＋`SchemaEvalConformanceReceiptV1`を使い、MCP Transport／Grant／`HostTransportConformanceReceiptV1`を必須`null`にする。cloud direct APIとfirst-party local inferenceはroute kindを増やさず、`engine_provider_adapter`配下の`InferenceDeploymentProfileV1.deployment_kind`で区別する。次のclosed型をMCDへ登録する。

```text
AiCallerContextPayloadV1
  caller_context_id, authenticated_subject_ref
  host_profile_binding: GovernedAiProfileBindingV1
  conformance_target: AiConformanceTargetRefV1
  execution_route:
    {kind: standard_external_mcp,
     transport_profile_binding: GovernedAiProfileBindingV1,
     provider_runtime_profile_binding: null,
     provider_manifest_binding: null,
     inference_deployment_profile_binding: null,
     model_snapshot_profile_binding: null,
     managed_deployment_identity_ref: null,
     managed_deployment_identity_sha256: null,
     managed_host_session_attestation_ref: null,
     managed_host_session_attestation_sha256: null,
     unattested_optional_metadata_ref: null | content ref,
     mcp_session_grant_ref: content ref,
     mcp_session_grant_sha256: lowercase hex 64,
     host_transport_conformance_receipt_ref:
       HostTransportConformanceReceiptRefV1,
     provider_tool_conformance_receipt_ref: null,
     schema_eval_conformance_receipt_ref:
       SchemaEvalConformanceReceiptRefV1,
     maximum_authority: query | proposal}
    | {kind: managed_external_host,
       transport_profile_binding: GovernedAiProfileBindingV1,
       provider_runtime_profile_binding: GovernedAiProfileBindingV1,
       provider_manifest_binding: null,
       inference_deployment_profile_binding: null,
       model_snapshot_profile_binding: GovernedAiProfileBindingV1,
       managed_deployment_identity_ref:
         exact ArtifactRefV1(
           artifact_kind=managed_provider_deployment_identity,
           schema_version=1),
       managed_deployment_identity_sha256: lowercase hex 64,
       managed_host_session_attestation_ref,
       managed_host_session_attestation_sha256,
       unattested_optional_metadata_ref: null,
       mcp_session_grant_ref: null,
       mcp_session_grant_sha256: null,
       host_transport_conformance_receipt_ref:
         HostTransportConformanceReceiptRefV1,
       provider_tool_conformance_receipt_ref:
         ProviderToolConformanceReceiptRefV1,
       schema_eval_conformance_receipt_ref:
         SchemaEvalConformanceReceiptRefV1,
       maximum_authority: query | proposal | managed_source_edit | build_job}
    | {kind: engine_provider_adapter,
       transport_profile_binding: null,
       provider_runtime_profile_binding: GovernedAiProfileBindingV1,
       provider_manifest_binding: GovernedAiProfileBindingV1,
       inference_deployment_profile_binding: GovernedAiProfileBindingV1,
       model_snapshot_profile_binding: GovernedAiProfileBindingV1,
       managed_deployment_identity_ref: null,
       managed_deployment_identity_sha256: null,
       managed_host_session_attestation_ref: null,
       managed_host_session_attestation_sha256: null,
       unattested_optional_metadata_ref: null,
       mcp_session_grant_ref: null,
       mcp_session_grant_sha256: null,
       host_transport_conformance_receipt_ref: null,
       provider_tool_conformance_receipt_ref:
         ProviderToolConformanceReceiptRefV1,
       schema_eval_conformance_receipt_ref:
         SchemaEvalConformanceReceiptRefV1,
       maximum_authority: query | proposal}
  tool_projection_binding: AiToolProjectionBindingV1
  authority_profile_binding: GovernedAiProfileBindingV1
  freshness_policy_binding: GovernedAiProfileBindingV1
  authorization_envelope_hash
  created_at, expires_at, revocation_snapshot_ref

AiCallerContextV1
  payload: AiCallerContextPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=ai_caller_context)

ExternalClientSecurityProfileV1
  host_profile_id, host_product_id, host_brand_display_name
  host_identity_kind = local_binary | hosted_service
  host_identity:
    {kind: local_binary,
     exact_version,
     client_binary_ref:
       exact ArtifactRefV1(
         artifact_kind=ai_client_binary, schema_version=1),
     client_binary_sha256,
     os_identity_binding,
     credential_storage_profile_ref, process_child_policy_ref,
     filesystem_policy_ref, network_policy_ref}
     | {kind: hosted_service,
        service_surface_id, service_release_channel,
        service_exact_release_id,
        tenant_workspace_ref, plan_tier_ref, admin_policy_ref,
       endpoint_identity_ref, oauth_or_oidc_subject_ref,
        token_audience, tls_identity_ref}
  supported_transport_profile_bindings[1..64]: GovernedAiProfileBindingV1
  credential_owner
  protocol_versions[]
  required_host_transport_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=host_transport)
  required_schema_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=schema)
  required_eval_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=eval)
  administrative_enablement = enabled | disabled
  issued_at, expires_at, revoked_at

EngineAiHostSecurityProfileV1
  host_profile_id
  host_surface = editor_ai_orchestrator | headless_ai_orchestrator
  engine_build_ref, engine_build_sha256
  process_artifact_ref, process_artifact_sha256
  os_identity_binding, process_child_policy_ref
  filesystem_policy_ref, network_policy_ref
  supported_provider_runtime_profile_bindings[1..64]: GovernedAiProfileBindingV1
  required_provider_tool_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=provider_tool)
  required_schema_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=schema)
  required_eval_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=eval)
  administrative_enablement = enabled | disabled
  issued_at, expires_at, revoked_at

McpTransportSecurityProfileV1
  transport_profile_id
  mcp_protocol_version = 2025-11-25
  transport_kind = local_stdio | streamable_http | secure_mcp_tunnel
  transport:
    {kind: local_stdio,
     server_binary_ref:
       exact ArtifactRefV1(
         artifact_kind=mcp_server_binary, schema_version=1),
     server_binary_sha256,
     os_acl_policy_ref, ipc_endpoint_identity_ref,
     inherited_credential_policy = none,
     maximum_message_bytes, request_timeout_ms,
     rate_limit_per_minute, maximum_concurrent_requests,
     maximum_session_seconds}
    | {kind: streamable_http,
       endpoint_origin, endpoint_identity_ref,
       tls_profile_ref, tls_profile_sha256,
       authorization:
         {kind: oauth_2_1,
          protected_resource_metadata_ref, protected_resource_metadata_sha256,
          authorization_server_metadata_ref, authorization_server_metadata_sha256,
          resource_indicator, token_audience,
          oidc_subject_policy_ref: null | content ref}
         | {kind: approved_private_mtls,
            mtls_profile_ref, private_service_ref, private_service_sha256,
            private_service_activation_ref, private_service_activation_sha256},
       origin_policy_ref, redirect_policy_ref,
       private_address_policy:
         {kind: deny}
         | {kind: approved_private_service,
            private_service_ref, private_service_sha256,
            private_service_activation_ref, private_service_activation_sha256},
       session_binding_policy_ref,
       maximum_message_bytes, request_timeout_ms,
       rate_limit_per_minute, maximum_concurrent_requests,
       maximum_session_seconds}
    | {kind: secure_mcp_tunnel,
       public_endpoint_origin, tunnel_endpoint_identity_ref,
       tunnel_client_artifact_ref:
         exact ArtifactRefV1(
           artifact_kind=mcp_tunnel_client_binary,
           schema_version=1),
       tunnel_client_artifact_sha256,
       local_service_binding_ref,
       tls_profile_ref, tls_profile_sha256,
       protected_resource_metadata_ref, protected_resource_metadata_sha256,
       authorization_server_metadata_ref, authorization_server_metadata_sha256,
       resource_indicator, token_audience,
       origin_policy_ref, redirect_policy_ref,
       private_address_policy = tunnel_local_service_only,
       session_binding_policy_ref, direct_inbound_allowed = false,
       maximum_message_bytes, request_timeout_ms,
       rate_limit_per_minute, maximum_concurrent_requests,
       maximum_session_seconds}
  required_host_transport_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=host_transport)
  issued_at, expires_at, revoked_at

AiAuthorityProfileV1
  authority_profile_id
  authority_class = query_only | proposal_only | managed_source_edit | build_job
  allowed_operation_refs[]
  maximum_risk_class
  required_evidence_policy_refs[]
  required_pre_execution_attestation_kinds[] = [] | [managed_host_session]
  required_post_execution_attestation_kinds[] = [] | [host_execution]
  forbidden_authorities[] = [approval, commit, activation, signing, release]
  issued_at, expires_at, revoked_at

AiExecutionFreshnessPolicyV1
  freshness_policy_id
  max_age_seconds_by_record_kind[]: {record_kind, max_age_seconds}
  clock_source_policy_ref
  issued_at, expires_at, revoked_at

AiConformanceTargetRefV1
  execution_target_profile_ref: TargetProfileRefV1
  artifact_target_profile_ref: null | TargetProfileRefV1

AiToolProjectionBindingV1
  projection_kind = mcp_tools | provider_native_tools
  projection_id, projection_version
  projection_ref:
    exact ArtifactRefV1(
      artifact_kind=ai_tool_projection, schema_version=1)
  projection_sha256: lowercase hex 64
  tool_set_sha256: lowercase hex 64
  operation_ref_set_sha256: lowercase hex 64
  input_schema_set_sha256: lowercase hex 64
  result_schema_set_sha256: lowercase hex 64

AiConformanceFixtureRefV1
  fixture_id, fixture_version
  fixture_polarity = positive_vector | negative_one_factor
  fixture_ref:
    exact ArtifactRefV1(
      artifact_kind=ai_conformance_fixture, schema_version=1)
  fixture_sha256: lowercase hex 64

AiConformanceFixtureSetV1
  fixture_set_id, fixture_set_version
  suite_kind = host_transport | provider_tool | schema | eval
  positive_fixture_count: positive uint32
  positive_fixture_refs[1..256]: AiConformanceFixtureRefV1(
    fixture_polarity=positive_vector)
  negative_fixture_count: positive uint32
  negative_fixture_refs[1..256]: AiConformanceFixtureRefV1(
    fixture_polarity=negative_one_factor)
  fixture_ref_set_sha256: lowercase hex 64
  fixture_set_content_hash: lowercase hex 64

AiConformanceTestCaseV1
  case_id, case_version
  suite_kind = host_transport | provider_tool | schema | eval
  case_polarity = positive_vector | negative_one_factor
  test_contract_ref:
    exact ArtifactRefV1(
      artifact_kind=ai_conformance_test_contract, schema_version=1)
  test_contract_sha256: lowercase hex 64
  fixture_ref: AiConformanceFixtureRefV1
  case_content_hash: lowercase hex 64

AiConformanceTestCaseRefV1
  case_id, case_version
  suite_kind = host_transport | provider_tool | schema | eval
  case_polarity = positive_vector | negative_one_factor
  case_ref:
    exact ArtifactRefV1(
      artifact_kind=ai_conformance_test_case, schema_version=1)
  case_sha256: lowercase hex 64

AiConformanceTestSuiteV1
  suite_id, suite_version
  suite_kind = host_transport | provider_tool | schema | eval
  fixture_set_ref:
    exact ArtifactRefV1(
      artifact_kind=ai_conformance_fixture_set, schema_version=1)
  fixture_set_sha256: lowercase hex 64
  required_case_count: uint32 >= 2
  required_case_refs[2..4096]: AiConformanceTestCaseRefV1
  required_case_set_sha256: lowercase hex 64
  required_negative_case_count: positive uint32
  required_negative_case_refs[1..4096]:
    AiConformanceTestCaseRefV1(case_polarity=negative_one_factor)
  required_negative_case_set_sha256: lowercase hex 64
  suite_content_hash: lowercase hex 64

AiConformanceTestSuiteRefV1
  suite_kind = host_transport | provider_tool | schema | eval
  suite_id, suite_version
  suite_ref:
    exact ArtifactRefV1(
      artifact_kind=ai_conformance_test_suite, schema_version=1)
  suite_sha256: lowercase hex 64
  fixture_set_ref:
    exact ArtifactRefV1(
      artifact_kind=ai_conformance_fixture_set, schema_version=1)
  fixture_set_sha256: lowercase hex 64
  required_case_count: uint32 >= 2
  required_case_set_sha256: lowercase hex 64
  required_negative_case_count: positive uint32
  required_negative_case_set_sha256: lowercase hex 64

AiConformanceCaseExecutionV1
  evaluated_test_suite_ref: AiConformanceTestSuiteRefV1
  required_case_count: uint32 >= 2
  required_case_refs[2..4096]: AiConformanceTestCaseRefV1
  required_case_set_sha256: lowercase hex 64
  executed_case_count: uint32 >= 2
  executed_case_refs[2..4096]: AiConformanceTestCaseRefV1
  executed_case_set_sha256: lowercase hex 64
  passed_case_count: uint32
  passed_case_refs[0..4096]: AiConformanceTestCaseRefV1
  passed_case_set_sha256: lowercase hex 64
  failed_case_count: uint32
  failed_case_refs[0..4096]: AiConformanceTestCaseRefV1
  failed_case_set_sha256: lowercase hex 64
  required_negative_case_count: positive uint32
  required_negative_case_refs[1..4096]:
    AiConformanceTestCaseRefV1(case_polarity=negative_one_factor)
  required_negative_case_set_sha256: lowercase hex 64
  executed_negative_case_count: positive uint32
  executed_negative_case_refs[1..4096]:
    AiConformanceTestCaseRefV1(case_polarity=negative_one_factor)
  executed_negative_case_set_sha256: lowercase hex 64

AiConformanceResultV1
  result:
    {kind: pass,
     case_execution: AiConformanceCaseExecutionV1,
     evidence_ref:
       exact ArtifactRefV1(
         artifact_kind=ai_conformance_result_evidence,
         schema_version=1),
     evidence_sha256: lowercase hex 64,
     diagnostic_refs: []}
    | {kind: fail,
       case_execution: AiConformanceCaseExecutionV1,
       evidence_ref:
         exact ArtifactRefV1(
           artifact_kind=ai_conformance_result_evidence,
           schema_version=1),
       evidence_sha256: lowercase hex 64,
       diagnostic_refs[1..256]}

AiConformanceSubjectTupleV1
  execution_route:
    {kind: standard_external_mcp,
     host_profile_binding:
       GovernedAiProfileBindingV1(
         profile_schema_id=ExternalClientSecurityProfileV1),
     transport_profile_binding:
       GovernedAiProfileBindingV1(
         profile_schema_id=McpTransportSecurityProfileV1),
     provider_runtime_profile_binding: null,
     provider_manifest_binding: null,
     inference_deployment_profile_binding: null,
     model_snapshot_profile_binding: null,
     managed_deployment_identity_ref: null,
     managed_deployment_identity_sha256: null,
     maximum_authority: query | proposal}
    | {kind: managed_external_host,
       host_profile_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=ExternalClientSecurityProfileV1),
       transport_profile_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=McpTransportSecurityProfileV1),
       provider_runtime_profile_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=ProviderRuntimeProfileV1),
       provider_manifest_binding: null,
       inference_deployment_profile_binding: null,
       model_snapshot_profile_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=ModelSnapshotProfileV1),
       managed_deployment_identity_ref:
         exact ArtifactRefV1(
           artifact_kind=managed_provider_deployment_identity,
           schema_version=1),
       managed_deployment_identity_sha256: lowercase hex 64,
       maximum_authority:
         query | proposal | managed_source_edit | build_job}
    | {kind: engine_provider_adapter,
       host_profile_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=EngineAiHostSecurityProfileV1),
       transport_profile_binding: null,
       provider_runtime_profile_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=ProviderRuntimeProfileV1),
       provider_manifest_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=ProviderManifestV1),
       inference_deployment_profile_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=InferenceDeploymentProfileV1),
       model_snapshot_profile_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=ModelSnapshotProfileV1),
       managed_deployment_identity_ref: null,
       managed_deployment_identity_sha256: null,
       maximum_authority: query | proposal}
  tool_projection_binding: AiToolProjectionBindingV1
  authority_profile_binding:
    GovernedAiProfileBindingV1(
      profile_schema_id=AiAuthorityProfileV1)
  conformance_target: AiConformanceTargetRefV1

HostConformanceIdentityV1
  identity:
    {kind: local_binary,
     host_product_id, exact_version,
     client_binary_ref:
       exact ArtifactRefV1(
         artifact_kind=ai_client_binary, schema_version=1),
     client_binary_sha256: lowercase hex 64,
     os_identity_binding}
    | {kind: hosted_service,
       host_product_id, service_surface_id,
       service_release_channel, service_exact_release_id,
       endpoint_identity_ref, tls_identity_ref,
       client_binary_ref: null,
       client_binary_sha256: null}

TransportConformanceIdentityV1
  transport_kind =
    local_stdio | streamable_http | secure_mcp_tunnel
  transport_profile_binding:
    GovernedAiProfileBindingV1(
      profile_schema_id=McpTransportSecurityProfileV1)

HostTransportConformanceReceiptPayloadV1
  receipt_id
  subject_tuple: AiConformanceSubjectTupleV1(
    route=standard_external_mcp | managed_external_host)
  host_identity: HostConformanceIdentityV1
  transport_identity: TransportConformanceIdentityV1
  test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=host_transport)
  result: AiConformanceResultV1
  qualifier_subject_ref
  qualifier_role_ref =
    role.ai-host-transport-conformance-qualifier
  issued_at, expires_at
  revoked_at: canonical UTC | null
  revocation_snapshot_ref

HostTransportConformanceReceiptV1
  payload: HostTransportConformanceReceiptPayloadV1
  signed_record:
    exact MirakanSignedRecordV1(
      purpose=host_transport_conformance)

HostTransportConformanceReceiptRefV1
  exact MirakanSignedRecordRefV1(
    purpose=host_transport_conformance)

ProviderDeploymentModelSnapshotV1
  deployment:
    {kind: managed_external_host,
     provider_runtime_profile_binding:
       GovernedAiProfileBindingV1(
         profile_schema_id=ProviderRuntimeProfileV1),
     managed_deployment_identity_ref:
       exact ArtifactRefV1(
         artifact_kind=managed_provider_deployment_identity,
         schema_version=1),
     managed_deployment_identity_sha256: lowercase hex 64,
     model_snapshot_profile_binding:
       GovernedAiProfileBindingV1(
         profile_schema_id=ModelSnapshotProfileV1)}
    | {kind: engine_provider_adapter,
       provider_runtime_profile_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=ProviderRuntimeProfileV1),
       provider_manifest_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=ProviderManifestV1),
       inference_deployment_profile_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=InferenceDeploymentProfileV1),
       model_snapshot_profile_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=ModelSnapshotProfileV1)}

ProviderToolConformanceReceiptPayloadV1
  receipt_id
  subject_tuple: AiConformanceSubjectTupleV1(
    route=managed_external_host | engine_provider_adapter)
  provider_deployment_model:
    ProviderDeploymentModelSnapshotV1
  observed_tool_projection_binding: AiToolProjectionBindingV1
  test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=provider_tool)
  result: AiConformanceResultV1
  qualifier_subject_ref
  qualifier_role_ref =
    role.ai-provider-tool-conformance-qualifier
  issued_at, expires_at
  revoked_at: canonical UTC | null
  revocation_snapshot_ref

ProviderToolConformanceReceiptV1
  payload: ProviderToolConformanceReceiptPayloadV1
  signed_record:
    exact MirakanSignedRecordV1(
      purpose=provider_tool_conformance)

ProviderToolConformanceReceiptRefV1
  exact MirakanSignedRecordRefV1(
    purpose=provider_tool_conformance)

SchemaEvalConformanceReceiptPayloadV1
  receipt_id
  subject_tuple: AiConformanceSubjectTupleV1
  schema_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=schema)
  eval_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=eval)
  schema_result: AiConformanceResultV1
  eval_result: AiConformanceResultV1
  overall_result = pass | fail
  qualifier_subject_ref
  qualifier_role_ref =
    role.ai-schema-eval-conformance-qualifier
  issued_at, expires_at
  revoked_at: canonical UTC | null
  revocation_snapshot_ref

SchemaEvalConformanceReceiptV1
  payload: SchemaEvalConformanceReceiptPayloadV1
  signed_record:
    exact MirakanSignedRecordV1(
      purpose=schema_eval_conformance)

SchemaEvalConformanceReceiptRefV1
  exact MirakanSignedRecordRefV1(
    purpose=schema_eval_conformance)

McpSessionGrantPayloadV1
  grant_id
  host_profile_binding: GovernedAiProfileBindingV1
  transport_profile_binding: GovernedAiProfileBindingV1
  client_identity:
    {kind: local_binary, client_binary_sha256, os_user_identity_ref,
     channel_binding_sha256}
    | {kind: hosted_service, service_session_subject_ref,
       oauth_or_oidc_subject_ref, token_audience, tls_identity_ref,
       channel_binding_sha256}
  project_id
  read_sensitivity_refs[], allowed_proposal_operation_refs[]
  issuer_subject_ref, issuer_role_ref
  issued_at, expires_at, nonce, revocation_snapshot_ref

McpSessionGrantV1
  payload: McpSessionGrantPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=mcp_session_grant)

ProviderRuntimeProfileV1
  provider_runtime_profile_id
  adapter_kind = official_sdk | raw_protocol | external_host_managed
  adapter:
    {kind: official_sdk, sdk_name, sdk_exact_version,
     sdk_artifact_ref, sdk_artifact_sha256}
    | {kind: raw_protocol, protocol_id, protocol_exact_version,
       request_response_schema_ref, request_response_schema_sha256}
    | {kind: external_host_managed,
       host_profile_binding: GovernedAiProfileBindingV1,
       required_host_execution_attestation = true}
  endpoint_identity_ref
  required_provider_tool_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=provider_tool)
  required_schema_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=schema)
  issued_at, expires_at, revoked_at

ProviderManifestV1
  provider_manifest_id
  provider_runtime_profile_binding: GovernedAiProfileBindingV1
  endpoint_identity_ref
  deployment_profile_binding: GovernedAiProfileBindingV1
  model_snapshot_profile_binding: GovernedAiProfileBindingV1
  role, inference_settings_sha256
  tool_projection_binding: AiToolProjectionBindingV1
  context_limit, output_limit, cost_limit_ref, latency_limit_ref
  privacy_policy_ref, retention_policy_ref, region_ref?, encryption_profile_ref, logging_policy_ref
  required_provider_tool_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=provider_tool)
  required_schema_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=schema)
  required_eval_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=eval)
  fallback:
    {kind: disabled, fallback_provider_manifest_binding: null,
     fallback_max_risk: null,
     fallback_requires_user_confirmation: false}
    | {kind: explicit_manifest,
       fallback_provider_manifest_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=ProviderManifestV1),
       fallback_max_risk,
       fallback_requires_user_confirmation: true}
  issued_at, expires_at, revoked_at

HostExecutionAttestationPayloadV1
  attestation_id
  caller_context_ref, caller_context_sha256
  host_session_ref, task_id, attempt_id
  managed_deployment_identity_ref:
    exact ArtifactRefV1(
      artifact_kind=managed_provider_deployment_identity,
      schema_version=1)
  managed_deployment_identity_sha256: lowercase hex 64
  task_specification_ref, task_specification_sha256
  authorization_envelope_hash
  input_closure_ref, input_closure_sha256
  freshness_policy_binding: GovernedAiProfileBindingV1
  execution_result:
    {kind: managed_source_edit,
     result_schema_id = urn:mirakan:schema:ai:managed-source-edit-result:v1,
     result_ref, result_sha256}
    | {kind: build_job,
       result_schema_id = urn:mirakan:schema:ai:managed-build-job-result:v1,
       result_ref, result_sha256}
  issued_at, expires_at, revocation_snapshot_ref

HostExecutionAttestationV1
  payload: HostExecutionAttestationPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=host_execution_attestation)

ManagedHostSessionAttestationPayloadV1
  session_attestation_id
  host_profile_binding: GovernedAiProfileBindingV1
  host_session_ref
  provider_runtime_profile_binding: GovernedAiProfileBindingV1
  model_snapshot_profile_binding: GovernedAiProfileBindingV1
  managed_deployment_identity_ref:
    exact ArtifactRefV1(
      artifact_kind=managed_provider_deployment_identity,
      schema_version=1)
  managed_deployment_identity_sha256: lowercase hex 64
  tool_projection_binding: AiToolProjectionBindingV1
  allowed_task_kinds[1..2]: non-empty subset of [managed_source_edit, build_job]
  maximum_authority_classes[1..2]: same set as allowed_task_kinds[]
  freshness_policy_binding: GovernedAiProfileBindingV1
  nonce
  issued_at, expires_at, revocation_snapshot_ref

ManagedHostSessionAttestationV1
  payload: ManagedHostSessionAttestationPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=managed_host_session_attestation)

ManagedSourceEditResultV1
  result_id
  result:
    {kind: completed,
     source_delta_manifest_ref, source_delta_manifest_sha256,
     changed_paths_manifest_ref, changed_paths_manifest_sha256,
     diagnostic_refs[]}
    | {kind: failed,
       source_delta_manifest_ref: null, source_delta_manifest_sha256: null,
       changed_paths_manifest_ref: null, changed_paths_manifest_sha256: null,
       diagnostic_refs[1..]}

ManagedBuildJobResultV1
  result_id
  result:
    {kind: completed,
     build_receipt_kind = package_receipt,
     build_receipt_ref, build_receipt_sha256,
     artifact_manifest_ref, artifact_manifest_sha256,
     diagnostic_refs[]}
    | {kind: failed,
       build_receipt_ref: null, build_receipt_sha256: null,
       artifact_manifest_ref: null, artifact_manifest_sha256: null,
       diagnostic_refs[1..]}

ManagedHostOutputAcceptanceReceiptPayloadV1
  receipt_id
  execution_kind = managed_source_edit | build_job
  caller_context_ref, caller_context_sha256
  task_id, attempt_id
  task_specification_ref, task_specification_sha256
  authorization_envelope_hash
  input_closure_ref, input_closure_sha256
  host_execution_attestation_ref, host_execution_attestation_sha256
  typed_result_ref, typed_result_sha256
  freshness_policy_binding: GovernedAiProfileBindingV1
  acceptance_result = accepted_to_staging | rejected
  accepted_at, expires_at, revocation_snapshot_ref

ManagedHostOutputAcceptanceReceiptV1
  payload: ManagedHostOutputAcceptanceReceiptPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=managed_host_output_acceptance)

InferenceDeploymentProfileV1
  deployment_profile_id, deployment_kind = cloud_endpoint | local_process_ipc
  provider_runtime_profile_binding: GovernedAiProfileBindingV1
  model_snapshot_profile_binding: GovernedAiProfileBindingV1
  deployment:
    {kind: cloud_endpoint,
     endpoint_identity_ref, network_policy_ref}
    | {kind: local_process_ipc,
       process_artifact_ref, process_artifact_sha256,
       runtime_loader_profile_binding: GovernedAiProfileBindingV1,
       model_import_qualification_receipt_ref,
       model_import_qualification_receipt_sha256,
       process_sandbox_profile_ref, gpu_isolation_profile_ref,
       inference_endpoint:
         {kind: os_ipc,
          ipc_endpoint_ref, ipc_auth_profile_ref,
          network_policy = deny_all}
         | {kind: authenticated_loopback,
            loopback_origin, loopback_endpoint_identity_ref,
            ipc_auth_profile_ref,
            network_policy = exact_loopback_only},
       required_ram_bytes, required_vram_bytes,
       cpu_limit, memory_limit_bytes, gpu_memory_limit_bytes,
       disk_limit_bytes, temp_cache_limit_bytes,
       maximum_weight_shard_count, maximum_weight_total_bytes,
       process_child_limit, maximum_batch_size,
       maximum_concurrent_inference, wall_time_limit_ms,
       custom_model_code_allowed = false,
       unsafe_serialization_allowed = false,
       archive_path_escape_allowed = false,
       runtime_network_fetch_allowed = false,
       model_license_acceptance_decision_ref,
       model_license_acceptance_decision_sha256,
       output_reproducibility:
         {kind: not_claimed}
         | {kind: deterministic_claim,
            runtime_artifact_sha256, gpu_profile_ref, gpu_profile_sha256,
            inference_settings_sha256, seed, sampler_profile_ref,
            sampler_profile_sha256, thread_policy_ref, thread_policy_sha256,
            repeatability_receipt_ref,
            repeatability_receipt_sha256}}
  context_limit
  required_provider_tool_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=provider_tool)
  required_schema_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=schema)
  required_eval_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=eval)
  privacy_policy_ref, logging_policy_ref, retention_policy_ref
  output_limit_bytes
  preflight_resource_policy = reject_before_start
  runtime_exhaustion_policy = terminate_process_tree_and_fail
  fallback_policy_owner = provider_manifest_only
  issued_at, expires_at, revoked_at

ModelLicenseAcceptanceDecisionPayloadV1
  decision_id
  local_model_artifact_manifest_binding: GovernedAiProfileBindingV1
  project_id, project_revision
  intended_use_profile_ref, intended_use_profile_sha256
  distribution_plan_ref, distribution_plan_sha256
  approved_use_kinds[]
  weight_redistribution = allowed | forbidden
  output_use = commercial_allowed | noncommercial_only | forbidden
  required_notices_ref, required_notices_sha256
  license_ref, license_sha256
  disposition = approved | rejected
  approver_subject_ref, approval_authority_ref
  issued_at, valid_until, revocation_snapshot_ref

ModelLicenseAcceptanceDecisionV1
  payload: ModelLicenseAcceptanceDecisionPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=model_license_acceptance_decision)

LocalModelArtifactManifestV1
  manifest_id
  model_format_id
  weights_encoding = native_precision | quantized
  weight_shards[]: {artifact_ref, artifact_sha256, size_bytes}
  encoding:
    {kind: native_precision, numeric_format = fp32 | fp16 | bf16}
    | {kind: quantized, quantization_id, quantization_artifact_ref,
       quantization_sha256}
  config_ref, config_sha256
  tokenizer_ref, tokenizer_sha256
  chat_template_ref, chat_template_sha256
  special_token_map_ref, special_token_map_sha256
  license_ref, license_sha256
  provenance_ref, provenance_sha256
  issued_at, expires_at, revoked_at

ModelSnapshotProfileV1
  model_snapshot_profile_id
  model_identity:
    {kind: provider_model_id,
     exact_provider_model_id,
     provider_terms_ref, provider_terms_sha256,
     license_ref, license_sha256,
     provenance_ref, provenance_sha256}
    | {kind: local_weights,
       local_model_artifact_manifest_binding: GovernedAiProfileBindingV1}
  declared_context_limit, effective_context_limit
  supported_schema_profile_refs[1..64]
  supported_tool_projection_bindings[1..64]:
    AiToolProjectionBindingV1
  required_eval_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=eval)
  issued_at, expires_at, revoked_at

GovernedAiProfileBindingV1
  profile_schema_id
  logical_profile_id
  record_ref, record_sha256
  revision
  issuance_head_ref, issuance_head_sha256, issuance_head_sequence

GovernedAiProfileRecordPayloadV1
  profile_record_id
  profile_schema_id
  profile_ref, profile_sha256
  revision: positive safe integer
  previous_record_ref: null | content-addressed ref
  previous_record_sha256: null | lowercase hex 64
  issuer_subject_ref, issuer_role_ref
  issued_at, revocation_snapshot_ref

GovernedAiProfileRecordV1
  payload: GovernedAiProfileRecordPayloadV1
  signed_record: MirakanSignedRecordV1

GovernedAiProfileHeadPayloadV1
  head_id
  profile_schema_id, logical_profile_id
  current_record_ref, current_record_sha256, current_revision
  sequence: positive safe integer
  previous_head_ref: null | content-addressed ref
  previous_head_sha256: null | lowercase hex 64
  recorded_at, revocation_snapshot_ref

GovernedAiProfileHeadV1
  payload: GovernedAiProfileHeadPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=governed_ai_profile_head)

LocalInferenceLoaderProfileV1
  loader_profile_id
  runtime_artifact_ref, runtime_artifact_sha256
  build_toolchain_ref, build_toolchain_sha256
  supported_model_format_ids[]
  custom_model_code_allowed = false
  unsafe_serialization_allowed = false
  archive_path_policy = normalized_no_escape
  runtime_network_fetch_allowed = false
  built_in_file_or_shell_tools_allowed = false
  mcp_proxy_allowed = false
  issued_at, expires_at, revoked_at

ModelImportQualificationReceiptPayloadV1
  receipt_id
  loader_profile_binding: GovernedAiProfileBindingV1
  local_model_artifact_manifest_binding: GovernedAiProfileBindingV1
  process_artifact_ref, process_artifact_sha256
  import_fixture_refs[]
  freshness_policy_binding: GovernedAiProfileBindingV1
  result = pass | fail
  issued_at, expires_at, revocation_snapshot_ref

ModelImportQualificationReceiptV1
  payload: ModelImportQualificationReceiptPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=model_import_qualification)

LocalInferenceRunReceiptPayloadV1
  run_receipt_id
  runtime_artifact_ref, runtime_artifact_sha256
  model_snapshot_profile_binding: GovernedAiProfileBindingV1
  gpu_profile_ref, gpu_profile_sha256
  inference_settings_sha256, seed
  sampler_profile_ref, sampler_profile_sha256
  thread_policy_ref, thread_policy_sha256
  replay_input_set_ref, replay_input_set_sha256
  freshness_policy_binding: GovernedAiProfileBindingV1
  result:
    {kind: completed,
     exact_response_bytes_sha256,
     diagnostic_refs[]}
    | {kind: failed,
       exact_response_bytes_sha256: null,
       diagnostic_refs[1..]}
  issued_at, expires_at, revocation_snapshot_ref

LocalInferenceRunReceiptV1
  payload: LocalInferenceRunReceiptPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=local_inference_run)

LocalInferenceRepeatabilityReceiptPayloadV1
  receipt_id
  runtime_artifact_sha256
  model_snapshot_profile_binding: GovernedAiProfileBindingV1
  gpu_profile_ref, gpu_profile_sha256
  inference_settings_sha256, seed
  sampler_profile_ref, sampler_profile_sha256
  thread_policy_ref, thread_policy_sha256
  replay_input_set_ref, replay_input_set_sha256
  freshness_policy_binding: GovernedAiProfileBindingV1
  run_receipt_refs[3..]: unique content-addressed refs
  repetition_count = length(run_receipt_refs[])
  exact_response_bytes_sha256
  result = byte_equal | mismatch
  issued_at, expires_at, revocation_snapshot_ref

LocalInferenceRepeatabilityReceiptV1
  payload: LocalInferenceRepeatabilityReceiptPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=local_inference_repeatability)
```

三Conformance Receipt、そのRef、`AiConformanceFixtureSetV1`、`AiConformanceTestCaseV1`、`AiConformanceTestSuiteV1`、`AiConformanceCaseExecutionV1`は上記Fieldだけを持つclosed schemaであり、全Fieldをrequired、unknown Field、Field省略、empty object、hash-only refを拒否する。`AiToolProjectionBindingV1.projection_sha256`は`projection_ref`が解決する完成bytes、`tool_set_sha256`はTool名／version／Operation ref／input Schema ref／result Schema ref／annotationをcanonical順にした完全集合、Operation／input／result set hashは各部分集合のRFC 8785 JCS SHA-256へ一致させる。

Fixture refはID／version／polarity／型付きArtifact ref／完成bytes hash、Case refはID／version／suite kind／polarity／型付きArtifact ref／完成bytes hashを全て保存する。Fixture Setのpositive／negative配列、Suiteのrequired／required-negative配列はそれぞれrefの全FieldをUTF-8 byte順へstrict sortし、重複を拒否する。`fixture_ref_set_sha256`、`required_case_set_sha256`、`required_negative_case_set_sha256`はASCII domain `MIRAKAN_AI_CONFORMANCE_{FIXTURE|CASE|NEGATIVE_CASE}_SET_V1`と当該完成配列のRFC 8785 JCS bytesを`uint32_be` length framingして計算する。Fixture／Case／Suite自身のcontent hashは同様に`MIRAKAN_AI_CONFORMANCE_{FIXTURE_SET|TEST_CASE|TEST_SUITE}_V1`と自己hash Fieldを除くclosed recordから計算し、隣接Artifact hashとbyte equalityにする。Suiteは少なくとも一positive Caseと一negative Caseを持ち、`required_negative_case_refs[]`を`required_case_refs[]`の`case_polarity=negative_one_factor` exact部分集合、各CaseのFixtureを同SuiteのFixture Set exact entry、Case／Fixture／Suiteの`suite_kind`を同値にする。

`AiConformanceTestSuiteRefV1`のsuite／fixture ref、ID、version、kind、両count／set hashは解決したSuite／Fixture Setとbyte equalityにする。suite ID、version、kind、Case集合、Fixture集合の一つでも違えば別suiteであり、表示名、最新版、Provider推奨値へfallbackしない。subject tuple内の全Profileが同じsuite kindについて持つ`required_*_test_suite_ref`は互いにbyte equalityで、対応Receiptのsuite refおよび`AiConformanceResultV1.case_execution.evaluated_test_suite_ref`と一致させる。external routeのHost／Transport suiteはExternal Host＋MCP Transport、managed routeのProvider suiteはProvider Runtime、Schema suiteはExternal Host＋Provider Runtime、Eval suiteはExternal Host＋Model、Engine routeのProvider／Schema suiteはEngine Host＋Provider Runtime＋Manifest＋Deployment、Eval suiteはEngine Host＋Manifest＋Deployment＋Modelで一致させ、別version／hashを混成しない。Profileは必要suite定義だけを参照するreceipt-free declarationであり、ReceiptをProfileへ逆参照させない。依存順は`Profile／Head＋Tool Projection＋Fixture Set＋Case＋Suite＋Target -> Conformance Receipt -> Caller Context`の一方向だけである。

各`receipt_id`は同Fieldを除く完成payloadのJCS hashから、Host／Transport=`urn:mirakan:host-transport-conformance:sha256:<lowercase-hex>`、Provider／Tool=`urn:mirakan:provider-tool-conformance:sha256:<lowercase-hex>`、Schema／Eval=`urn:mirakan:schema-eval-conformance:sha256:<lowercase-hex>`として導出する。全Receiptで`issued_at < expires_at`、`revoked_at`は省略不能な`canonical UTC | null`、current利用は`revoked_at=null`かつcurrent revocation snapshot一致だけを許す。署名済みwrapperは`subject_sha256=SHA-256(JCS(payload))`、Signer subject／Role、singleton purpose Key、`issued_at`、revocation snapshotをpayloadへbyte-exact一致させる。期限延長再包装、別purposeの有効署名、Qualifier自己申告、過去Receiptのcurrent再利用を拒否する。

`AiConformanceSubjectTupleV1`はrouteごとのfull subject tupleである。全`GovernedAiProfileBindingV1`は発行時の完成Profile Recordと署名済みissuance Headへ解決し、Receipt発行時にcurrent Headでなければならない。`HostTransportConformanceReceiptV1.host_identity`は解決したExternal Host ProfileのHost product、exact version、local binary ref／hashまたはhosted releaseとbyte equalityにする。`transport_identity.transport_profile_binding`はsubject tupleのTransport bindingとbyte equalityにし、解決した完成`McpTransportSecurityProfileV1`がMCP protocol version、binary／endpoint、ACL、OAuth metadata、mTLS private-service Activation、origin／redirect／private-address／session policy、rate／timeout／concurrency上限を一つのclosed transport branchとして所有する。`transport_identity.transport_kind`はそのbranch discriminatorと一致させ、Receipt側に別の不完全なauthentication projectionを作らない。local Hostだけがbinaryをnon-null、hosted Hostだけがbinaryをexact nullにし、hosted releaseを架空binary hashへ変換しない。

`engine_provider_adapter`ではHost／Transport Receiptを作らず、Provider／Tool ReceiptとSchema／Eval Receiptの共通subject tupleにある`EngineAiHostSecurityProfileV1` bindingがEngine build ref/hash、process artifact ref/hash、OS identity、sandbox policy、issuance Headを束縛する。Provider Receiptの`provider_deployment_model`はsubject tupleとbyte equalityにする。`managed_external_host`ではProvider Runtime／Modelをnon-null、Provider Manifest／Inference Deploymentをnullにし、登録Brokerがattestする`managed_deployment_identity_ref/hash`をCaller Context payload、subject tuple、Provider／Tool Receipt snapshot、Managed Host Session Attestation、Host Execution Attestationで全てnon-nullかつbyte equalityにする。同じProvider Runtime／Modelでもmanaged deployment identityが一byte違えば別subjectでありReceipt、Session、Execution結果を流用しない。このProvider Runtimeは`adapter.kind=external_host_managed`で、その`host_profile_binding`をsubject tupleのExternal Host bindingへ一致させる。`standard_external_mcp | engine_provider_adapter`ではCaller Context／subject tupleのmanaged deployment identity両Fieldをexact nullにする。Engine routeではProvider Runtime／Manifest／Inference Deployment／Modelを全non-nullにし、Manifestが指すRuntime／Deployment／Model、Deploymentが指すRuntime／Modelをsubject tupleへ一致させる。`observed_tool_projection_binding`もsubject tupleのTool Projectionとbyte equalityにする。

`AiConformanceCaseExecutionV1`はSuiteのrequired Case全量をそのまま保存する。`required_case_refs/set hash/count`は解決したSuite、`executed_case_refs/set hash/count`はrequired集合、`required_negative_case_refs/set hash/count`と`executed_negative_case_refs/set hash/count`はSuiteのrequired-negative集合へそれぞれexact set equalityにする。`passed_case_refs[]`と`failed_case_refs[]`はexecuted集合の互いに素なexact partitionで、各countとset hashを実配列から再計算する。未実行、表外、duplicate、polarity変更、CaseまたはFixture hash差、countだけの一致を有効Resultにしない。

`AiConformanceResultV1.result.kind=pass`は`passed_case_refs=required_case_refs`、`failed_case_refs=[]`、全negative Case実行、Diagnostic exact `[]`の場合だけ許す。`kind=fail`は`failed_case_refs[]`とDiagnosticを各一件以上必須にし、passへ補正しない。Schema／Eval Receiptの`overall_result=pass`は`schema_result.result.kind=pass`かつ`eval_result.result.kind=pass`の論理積であり、一方のfail、未実行Case、部分Suite、自然言語result、欠落Evidenceをpassへ補正しない。`standard_external_mcp`のEvalは外部Hostが公開するTool Schema、typed result、error、cancellation、proposal-only境界のblack-box conformanceだけを評価し、標準MCPでattestできない上流Provider／Modelの品質・identityを主張しない。Provider／Model attributionを要するEvalは`managed_external_host | engine_provider_adapter`のfull tupleだけで発行する。`AiConformanceTargetRefV1.execution_target_profile_ref`は試験を実行したexact Host／runtime Target、`artifact_target_profile_ref`はToolまたはBuild結果がTarget固有のときだけnon-nullにする。Windows Editor Host上でAndroid／Appleを評価する場合は両Refを分離し、Editor Receiptをartifact Target Receiptへ流用しない。

Caller Context発行時、GatewayはContextから`AiConformanceSubjectTupleV1`を再投影し、各Receipt payloadのsubject tupleとbyte equalityにする。具体的にはHost／全Profileのschema・logical ID・Record ref/hash・revision・issuance Head ref/hash/sequence、Transport Profile binding、Provider／Deployment／managed deployment identity／Model、Tool Projectionと全set hash、Authority、execution／artifact Target、route ceilingを一Fieldずつ比較する。`standard_external_mcp`はHost／Transport Receipt＋Schema／Eval Receiptをexact非null、Provider／Tool Receiptをnullにする。`managed_external_host`は三Receiptすべてをexact非null、`engine_provider_adapter`はProvider／Tool Receipt＋Schema／Eval Receiptをexact非null、Host／Transport Receiptをnullにする。Receipt同士のsubject tupleもbyte-equalでなければならない。Context発行後もTool一覧取得直前と各dispatch直前に全Profile Head、Receipt expiry／revocation、Context tupleをread-backし、Head drift、Caller ContextだけのHost／managed deployment／Model／Target差替え、古いReceiptと新Profileの混成をfail closedにする。

`McpSessionGrantPayloadV1.expires_at`は`issued_at`から最大60分である。`grant_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:mcp-session-grant:sha256:<lowercase-hex>`として導出し、完成payloadを`role.mcp-session-grant-service`のsingleton-purpose Keyで署名する。signed recordのsubject、issuer subject／Role、issued_at、revocation snapshotをpayloadとexact一致させ、grantのoperation集合はQuery／Proposalだけに制限する。optional Fieldはkind／transportのtagged branchだけで必要条件を閉じ、local binaryをhosted serviceへ捏造またはその逆をしない。`ModelSnapshotProfileV1.model_identity`の固有Fieldは、`provider_model_id`なら`exact_provider_model_id`、`local_weights`ならcurrent `local_model_artifact_manifest_binding`だけを必須にして相互混在を拒否する。Local manifestは全weight shard、config、tokenizer、chat template、special-token map、license、provenanceとencoding branchをhash closureにし、native FP16／BF16へ架空quantizationを要求しない。identity正本はModel Snapshot＋Local manifestだけであり、`InferenceDeploymentProfileV1`へ複写しない。DeploymentとProvider Manifestの`model_snapshot_profile_binding`はschema／logical ID／record／revision／issuance Headをbyte-exact一致させる。`local_process_ipc`ではprocess artifact、`local_weights` Snapshot、認証済みOS IPCまたはloopback、local resource上限をすべて必須にする。Host display name、Provider名、Model family名は表示metadataであり、Transport、Tool Schema、Authorityを推測する入力にしない。

`InferenceDeploymentProfileV1.deployment.model_import_qualification_receipt_ref`は、receipt-freeな`LocalModelArtifactManifestV1`、`LocalInferenceLoaderProfileV1`、process artifactおよびimport fixture closureを先に固定して発行した`ModelImportQualificationReceiptV1`だけを参照する下流policy projectionである。Receipt payload／subjectはDeployment Profile、その`GovernedAiProfileRecordV1`、issuance Head、Binding、またはそれらのhashを含めてはならず、Deployment ProfileをReceipt subjectへ逆参照する循環をValidatorは拒否する。依存順は`local model／loader／process／fixture closure -> ModelImportQualificationReceiptV1 -> InferenceDeploymentProfileV1 -> GovernedAiProfileRecordV1／Head`の一方向だけとする。

`AiCallerContextV1`はGatewayだけが`role.ai-gateway-context-publisher`／singleton purpose `ai_caller_context`で発行する短命signed contextである。`caller_context_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:ai-caller-context:sha256:<lowercase-hex>`として導出し、signed recordのsubject hash、issued_at=`created_at`、revocation snapshotをexact一致させる。`created_at < expires_at`、current Freshness Policyの`record_kind=ai_caller_context`はexact一件かつ`max_age_seconds <= 600`、`expires_at=min(created_at+max_age_seconds, Authorization／当該branchのGrantまたはSession Attestation／全non-null profile・Receiptのexpiry)`を必須にする。各non-null binding、Grant、Conformance Receipt、Authorization Envelopeを発行時とTool実行直前にread-backし、`expires_at <= evaluation_time`、current Head drift、revocation、別Project／subject／channelでは拒否する。

`host_profile_binding.profile_schema_id`は`standard_external_mcp | managed_external_host`で`ExternalClientSecurityProfileV1`、`engine_provider_adapter`で`EngineAiHostSecurityProfileV1`だけを許す。外部2 routeではHost Profileが列挙するcurrent MCP Transport bindingとContextのbindingをbyte-exact一致させる。Engine routeではEngine Hostが列挙するcurrent Provider Runtime bindingとContextを一致させ、MCP Transportを要求または捏造しない。`standard_external_mcp`はProvider Runtime、Provider Manifest、Inference Deployment、Model、managed deployment identity、Managed Session Attestation、Provider／Tool Receiptを全てnull、Host／Transport ReceiptとSchema／Eval Receiptをnon-nullとし、MCP initialize由来のProvider／Model名はunattested metadataだけへ隔離する。`managed_external_host`だけはProvider／Model binding、managed deployment identity、三Conformance Receipt、実行前`ManagedHostSessionAttestationV1` ref／hashを全non-null、同一Host session／`tool_projection_binding`へ閉じる。`engine_provider_adapter`はmanaged deployment identityをnullにし、Provider Manifestが指すProvider Runtime、Inference Deployment、Model Snapshot、Tool ProjectionをContextとbyte-exact一致させ、Provider／Tool ReceiptとSchema／Eval Receiptをnon-nullにする。cloud direct APIとfirst-party local IPCは同じEngine routeのDeployment branchであり、MCP Transport Profileを流用しない。branch間Field流用、裸Context、caller自己署名を拒否する。

route別session bindingもclosedにする。`standard_external_mcp`はfresh `McpSessionGrantV1` ref／hashを両non-nullで必須、`engine_provider_adapter`は両方null、`managed_external_host`は両方nullかつ実行前`ManagedHostSessionAttestationV1`を専用Broker session bindingとして必須にする。effective operation集合は常に `route ceiling ∩ current AiAuthorityProfile.allowed_operation_refs[] ∩ signed TaskAuthorizationEnvelope.payload.allowed_operations[] ∩ Server Policy`、standard MCPではさらに`∩ McpSessionGrant.payload.allowed_proposal_operation_refs[]`である。managed routeではSession Attestationの`allowed_task_kinds[]`／authority classも積集合へ加える。いずれかのmissing、tuple差、空でない超過、より強いcaller申告、Profileの`forbidden_authorities[]`欠落をTool公開前と実行直前に拒否する。

read-time `effective_route_state`はProfile Fieldではなく、`supported | proposal_only | not_activated`のclosed projectionとしてGatewayだけが次の順で導出する。(1) routeが参照する全Profile bindingをcurrent signed `GovernedAiProfileRecordV1`／issuance Headへ解決し、Host Profileの`administrative_enablement=enabled`、他Profileのbranch整合を検査する。(2) §8.3のroute別full tuple、required Grant／Session Attestation、exact Tool Projection／Authority／Targetをset／byte equalityで検査する。(3) routeが要求するexact二または三Conformance Receiptをすべてcurrent `result=pass`として解決し、Suite全Case、freshness、expiry、revocation、Profile Head、subject tupleを再検証する。(4) Product／Capability ActivationとTask Authorizationを含む§7～§8のdispatch predicateを再評価する。全条件が成立した`standard_external_mcp`だけを`proposal_only`、成立した`managed_external_host | engine_provider_adapter`をroute ceiling内の`supported`、それ以外を`not_activated`とする。`administrative_enablement=disabled`、Profile／Receipt一件のmissing／fail／stale／revoked、tuple差は必ず`not_activated`であり、Profile revisionの自己申告、Host brand、過去Receiptから`effective_route_state`を復元しない。Managed route不成立から`proposal_only`へ降格せず、別のcurrent standard route full tupleから新しいCaller Contextを発行した場合だけ、その別routeを独立に`proposal_only`とする。

Attestation条件は因果順に評価する。query／proposal Authorityはpre／post集合を両方empty exact set、managed edit／build Authorityは`required_pre_execution_attestation_kinds=[managed_host_session]`、`required_post_execution_attestation_kinds=[host_execution]` exact setとする。Tool公開前・実行直前はpre集合だけを検査し、処理後にBrokerがtyped resultを得てからHost Execution Attestationを発行する。Staging受入れ、`GenerationReceiptV1`完成、`ManagedHostOutputAcceptanceReceiptV1`発行の各時点ではpre Attestationを再検証したうえでpost集合も必須にする。post Attestationを実行前Contextへ埋め込むこと、post欠落のresultをStagingへ受け入れること、R4判断やCommitをallowlistへ追加することを拒否する。

次のprofile payloadは必ず`GovernedAiProfileRecordV1`で署名する。`ExternalClientSecurityProfileV1`と`EngineAiHostSecurityProfileV1`の`administrative_enablement`は署名済みgovernance inputであり、`disabled`がrouteを停止する上限switch、`enabled`が後続Conformance評価を許可するだけである。Profile自身は`pass | fail`、`supported | proposal_only | not_activated`、またはOperation可用性を宣言しない。

| profile schema | exact purpose | issuer Role |
|---|---|---|
| `ExternalClientSecurityProfileV1` | `external_client_security_profile` | `role.ai-profile-security.r4` |
| `EngineAiHostSecurityProfileV1` | `engine_ai_host_security_profile` | `role.ai-profile-security.r4` |
| `McpTransportSecurityProfileV1` | `mcp_transport_security_profile` | `role.ai-transport-security.r4` |
| `AiAuthorityProfileV1` | `ai_authority_profile` | `role.ai-authority-policy.r4` |
| `AiExecutionFreshnessPolicyV1` | `ai_execution_freshness_policy` | `role.ai-execution-freshness-policy.r4` |
| `ProviderRuntimeProfileV1` | `provider_runtime_profile` | `role.ai-provider-runtime-owner.r4` |
| `ProviderManifestV1` | `provider_manifest` | `role.ai-provider-profile-owner.r4` |
| `InferenceDeploymentProfileV1` | `inference_deployment_profile` | `role.ai-deployment-profile-owner.r4` |
| `ModelSnapshotProfileV1` | `model_snapshot_profile` | `role.ai-model-profile-owner.r4` |
| `LocalModelArtifactManifestV1` | `local_model_artifact_manifest` | `role.ai-model-artifact-publisher` |
| `LocalInferenceLoaderProfileV1` | `local_inference_loader_profile` | `role.ai-local-runtime-security.r4` |

各logical profile IDごとに初回revision 1／previous=null、以後current完成wrapper ref／hashとexact `N+1`を持つrecord chainを作る。`profile_record_id`は同Fieldを除く完成`GovernedAiProfileRecordPayloadV1`のJCS hashから`urn:mirakan:governed-ai-profile-record:sha256:<lowercase-hex>`として導出する。`signed_record.subject_sha256`はprofile単体でなくIDを含む完成Record payload JCS hash、Signer subject／Role、issued_at、revocation snapshot、上表purposeをouter payloadとexact一致させる。`profile_ref/hash`は完成profile bytesへ解決し、Local Model Artifactを含む全profileで`profile.issued_at=record.payload.issued_at`、profile expiry／revocation Fieldとcurrent stateを検証する。全Governed profileの`revoked_at`は省略不能な`canonical UTC | null`で、current利用は`null`だけを許す。省略、空文字、sentinel、non-null、profileとcurrent revocation snapshotの不一致を拒否する。これにより署名済みprofileを維持したrevision／previous／issuer差替えを拒否する。

Record chainに加えてlogical profile IDごとの完成`GovernedAiProfileHeadV1`を必須にする。genesisはsequence 1／previous=null／current revision 1、後続はcurrent完成Headをprevious、sequenceとrevisionをexact `N+1`にし、`service.ai-profile-head-publisher`／`role.ai-profile-head-publisher`／singleton purpose `governed_ai_profile_head`で署名してexpected previousへのsingle CASを行う。`head_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:governed-ai-profile-head:sha256:<lowercase-hex>`として導出する。Bindingは発行時Head wrapper ref／hash／sequenceと、そのHeadが指すschema、logical ID、record ref／hash、revisionをexact固定する。新規実行、Authorization、Staging受入れ、Promotionでは発行時Headがcurrent Headと同一でなければ`stale`として拒否する。履歴監査では発行時Headから署名chainを再生し、当時有効なら`authentic_but_stale`として過去Receiptを保持するが、新規権限またはcurrent性能主張へ再利用しない。Head更新だけで過去Receiptを改竄扱いまたは削除せず、filesystem `latest`、mutable logical ID、古いbindingのcurrent利用を拒否する。unknown／expired／revoked Role、assignment、Key、branch、gap、同内容revision bumpも拒否する。

`McpSessionGrantV1`、`AiCallerContextV1`、`ManagedHostSessionAttestationV1`、`HostExecutionAttestationV1`は各専用Role／purposeで別途署名し、同じprofile chainへ混ぜない。Managed Host Sessionは登録Brokerの`role.managed-host-session-attestor`／purpose `managed_host_session_attestation`、post-executionは`role.managed-host-execution-attestor`／purpose `host_execution_attestation`を使い、payload JCS subject、型別issued_at、revocation snapshotをexact一致させる。

Profile外のsecurity recordは次のexact署名行だけを許す。全wrapperで`subject_sha256=SHA-256(JCS(payload))`、Signer subject／Role、singleton purpose Key、型別issued_at、payloadのrevocation snapshot、発行時と評価時のcurrent revocationを検証する。

| record | exact purpose | issuer Role | signed time | 利用可能result |
|---|---|---|---|---|
| `McpSessionGrantV1` | `mcp_session_grant` | `role.mcp-session-grant-service` | payload `issued_at` | expiry内のquery／proposal grant |
| `AiCallerContextV1` | `ai_caller_context` | `role.ai-gateway-context-publisher` | payload `created_at` | route/profile/envelope積集合がnon-empty |
| `AiTaskAttemptReservationV1` | `ai_task_attempt_reservation` | `role.ai-gateway-task-attempt-reservation` | payload `reserved_at` | current terminal Head、上限内count、current Context／Authorization |
| `AiTaskRepairAttemptHeadV1` | `ai_task_repair_attempt_head` | `role.ai-gateway-task-repair-attempt-head` | payload `recorded_at` | expected-previous CAS成功、closed reserved／recorded／aborted state |
| `StandardExternalProposalReceiptV1` | `standard_external_proposal_receipt` | `role.ai-gateway-standard-external-proposal-receipt` | payload `received_at` | current reserved Head／Context／Grant、typed Proposal |
| `HostTransportConformanceReceiptV1` | `host_transport_conformance` | `role.ai-host-transport-conformance-qualifier` | payload `issued_at` | `pass`かつcurrent Host／Transport／Target tuple |
| `ProviderToolConformanceReceiptV1` | `provider_tool_conformance` | `role.ai-provider-tool-conformance-qualifier` | payload `issued_at` | `pass`かつcurrent Provider／Deployment／Model／Tool tuple |
| `SchemaEvalConformanceReceiptV1` | `schema_eval_conformance` | `role.ai-schema-eval-conformance-qualifier` | payload `issued_at` | Schema／Eval両方`pass`かつ同じfull tuple |
| `ManagedHostSessionAttestationV1` | `managed_host_session_attestation` | `role.managed-host-session-attestor` | payload `issued_at` | allowed task kindだけ |
| `HostExecutionAttestationV1` | `host_execution_attestation` | `role.managed-host-execution-attestor` | payload `issued_at` | typed resultの完成branch |
| `ManagedHostOutputAcceptanceReceiptV1` | `managed_host_output_acceptance` | `role.managed-host-output-acceptor` | payload `accepted_at` | `accepted_to_staging`かつtyped result=`completed`だけ |
| `ModelImportQualificationReceiptV1` | `model_import_qualification` | `role.ai-model-import-qualifier` | payload `issued_at` | `pass`だけ |
| `LocalInferenceRunReceiptV1` | `local_inference_run` | `role.ai-local-inference-runner` | payload `issued_at` | result kind=`completed`だけ |
| `LocalInferenceRepeatabilityReceiptV1` | `local_inference_repeatability` | `role.ai-repeatability-runner` | payload `issued_at` | `byte_equal`だけ |
| `ModelLicenseAcceptanceDecisionV1` | `model_license_acceptance_decision` | `role.model-license-acceptance.r4` | payload `issued_at` | `approved`だけ |

三Conformance Qualifier RoleはProfile外security qualification wrapper専用であり、MCD Operation実行、Operation Receipt発行、Approval、Commit、Activation、Signing、Releaseを許可しない。currentの三Qualifier Identity／Role Registry row／Assignment／Key／Receipt instance集合はそれぞれexact `[]`で、schema上のRole literalだけから署名Authorityを合成しない。将来は使用するrouteのProfile／Suite／Fixture／Qualifier Trust closure／三Receipt schemaを一つの承認済みsecurity qualification transactionでmaterializeする。`OperationReceiptSignerPolicyV1.entries[]`、active／conditional／planning Operation集合、各Operation Signer destinationへ三Roleを追加せず、既存のOperation件数とSigner Policy件数を変更しない。

Freshness-bearing recordはcurrent `AiExecutionFreshnessPolicyV1` bindingの該当`record_kind`が一件だけ存在し、`expires_at = min(issued_at + max_age_seconds, 全参照profile／Receiptのexpires_at)`を満たす。Acceptance Receiptでは`issued_at`を`accepted_at`と読む。policy missing／duplicate、caller指定max age、expiry延長再包装、current Head差、source expiry／revocationを拒否する。License Decisionは同式でなくpayloadの`issued_at <= evaluation_time < valid_until`とProject／license driftを継続評価する。

| Host profile | 許可Transportの扱い | 対応表示条件 |
|---|---|---|
| Editor | direct Provider API、local STDIO MCP、またはActivation済みStreamable HTTP | Engine ProviderならProvider／Tool＋Schema／Eval、外部HostならHost／Transport＋Schema／Evalのtyped Receiptと製品内Policyが有効 |
| hosted ChatGPT Work | pluginが束ねるremote MCP Toolを使う。local Codex `config.toml`、local STDIO、Codex command menuを読み込まず、private network／developer machineは承認済みremote serviceまたはSecure MCP Tunnel経路だけ | plan／workspace条件、admin policy、exact plugin／remote Host／Transport＋Schema／Eval Receiptが有効 |
| ChatGPT desktop appのCodex host | Codex hostとしてlocal STDIO MCPまたはStreamable HTTPを使い、Codex CLI／IDEと同じCodex MCP config layerを共有できる。hosted ChatGPT Work plugin接続とは別Host Profile | exact desktop Codex host version／binary／Host／Transport＋Schema／Eval Receiptが有効 |
| Codex CLI／Codex IDE | local STDIO MCPまたはStreamable HTTP | exact client version／binary／Host／Transport＋Schema／Eval Receiptが有効 |
| Claude Desktop／Claude Code | Profileに列挙したlocal STDIO MCPまたはStreamable HTTP | exact client version／binary／Host／Transport＋Schema／Eval Receiptが有効 |
| Cursor | Profileに列挙したlocal STDIO MCPまたはStreamable HTTP | exact client version／binary／Host／Transport＋Schema／Eval Receiptが有効 |

この表は許容するprofile classであり、現時点の具体的なHost／version／binary／Transport組合せを`supported`と宣言しない。初期`ExternalClientSecurityProfileV1`、`EngineAiHostSecurityProfileV1`、`HostTransportConformanceReceiptV1`、`ProviderToolConformanceReceiptV1`、`SchemaEvalConformanceReceiptV1`、`AiConformanceFixtureSetV1`、`AiConformanceTestCaseV1`、`AiConformanceTestSuiteV1`、managed Host attestor、first-party local runtime、三routeの`AiCallerContextV1`のcurrent materialized instanceはそれぞれexact 0件である。Phase 4のAI CoreはEngine build／process／OS policyとProvider tupleを束縛したEngine Host Profileを、Phase 5 `wp.product.external-agent`は少なくとも一つのexact External Host＋MCP Transport＋Tool Projection＋proposal-only Authority組合せをそれぞれmaterializeし、positive／negative Conformanceを通す。ChatGPT、Codex、Claude、Cursor等の名前だけでは完了しない。`managed_source_edit | build_job` routeはProduct Future Registryのexact `future.capability.managed-external-host-execution`、`planning_state=planning_only`にだけ属する。そのFutureのActive Definition migrationで専用Work Package、Threat Model、Broker、session／execution attestor、Engine Build Receipt closureを同時登録し、通常lifecycleで承認・Qualificationするまで`not_activated`であり、MVPのstandard external MCP proposal laneへ混ぜない。これは別entry `future.capability.first-party-local-inference`のfirst-party local runtime計画と独立であり、一方のEvidence／Activationを他方へ流用しない。

typed Receipt不在、期限切れ、失効、version／binary／Transport endpoint／auth／Provider／Deployment／Model／Tool set／Schema／Eval／Target／Profile Head差では`supported`と表示しない。`proposal_only`を表示できるのは、失効したManaged Contextを降格した場合ではなく、別のcurrent `standard_external_mcp` Host／Transport／Tool／proposal-only Authority Profile、MCP Grant、Host／Transport＋Schema／Eval ConformanceからGatewayが新Caller Contextを発行した場合だけである。その標準経路も成立しなければ`not_activated`とする。Callerの最大Authorityは`query | proposal | managed_source_edit | build_job`のclosed setであり、Approval、Commit、Activation、Signing、ReleaseはどのHost、Provider、Modelにも付与しない。

`ExternalClientSecurityProfileV1.host_product_id`はsurface／modeを含むexact IDとし、hosted ChatGPT Work、ChatGPT desktop appのCodex host、Codex CLI、Codex IDEを別Profileとして発行する。desktop applicationという同じ容器、同じaccount、同じvendorであることを根拠にTransport、設定、権限、Conformance Receiptを共有しない。hosted ChatGPT Work用remote plugin ToolをCodex local MCPとして、またはCodex local MCP configをhosted ChatGPT Work pluginとして自動投影しない。

`McpTransportSecurityProfileV1`はtransport kindごとの全Fieldをrequiredにし、他branch Fieldをunknownとして拒否する。local STDIOはbinary hash、OS ACL、IPC identity、credential非継承を、Streamable HTTPはexact origin／TLS／MCP OAuth 2.1 Protected Resource Metadata／Authorization Server Metadata／resource indicator／token audience／redirect／private-address／session bindingを、Secure MCP Tunnelはpublic endpointとexact tunnel client artifact、local service binding、direct inbound禁止を検証する。OIDC subjectはOAuth 2.1 branchの補助identity policyであってOAuth authorizationを置換せず、mTLS-onlyはActivation済みprivate service branchに限定する。`approved_private_service`は別Threat Model、service ref／hash、Activation ref／hashがある場合だけ許し、名前がprivate、localhost、tunnelであることを根拠にしない。全branchでexact MCP protocol version、message／timeout／rate／concurrency／session上限を強制する。External Client Host Profileの`supported_transport_profile_bindings[]`は各current完成Governed MCP Transport Profile recordへ解決し、Profile／Transport／`HostTransportConformanceReceiptV1`／`SchemaEvalConformanceReceiptV1`のexpiry・revocation・hash差で接続前にfail closedにする。Engine Host ProfileはMCP Transportでなくcurrent Provider Runtime bindingを列挙し、Engine build／process artifact／OS／filesystem／network policy差でEngine routeをfail closedにする。

`future.capability.managed-external-host-execution`が`planning_only`から承認済みActivationへ進み、専用Work Packageがmaterializeされた場合だけ、Managed HostのSource edit／Build出力をそれぞれclosed `ManagedSourceEditResultV1`／`ManagedBuildJobResultV1`として保存し、処理完了後の`HostExecutionAttestationV1` ref／hashを`ManagedHostOutputAcceptanceReceiptV1`のrequired Fieldにする。`attestation_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:host-execution-attestation:sha256:<lowercase-hex>`、session attestation IDは同様に`urn:mirakan:managed-host-session-attestation:sha256:<lowercase-hex>`で導出する。flat result objectの`result_id`は同Fieldを除く完成object JCS hashからSource edit=`urn:mirakan:managed-source-edit-result:sha256:<lowercase-hex>`、Build=`urn:mirakan:managed-build-job-result:sha256:<lowercase-hex>`、Acceptance payloadの`receipt_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:managed-host-output-acceptance-receipt:sha256:<lowercase-hex>`として導出する。post-execution payloadのCaller Context、managed deployment identity、Task Specification、Authorization Envelope、attempt、Input closure、result branchのkind／schema ID／ref／hashはSession Attestation、Provider／Tool Receipt、Acceptance Receiptの対応入力とbyte-exactでなければならず、Broker署名とcurrent revocationを検証後にだけStagingへ受理する。Build familyとManaged Host routeの双方が将来Activationされた場合だけ、completed Buildの`build_receipt_ref/hash`はCore ArchitectureのEngine-owned `PackageReceiptV1`完成wrapperで、operation ID=`operation.build.request_package`、purpose=`operation_receipt:operation.build.request_package`、Build Gateway signer、result=`succeeded`、同じTask Specification／Authorization／Project revision／Candidate／Target／Toolchain／artifact manifestを必須にする。Host／Broker AttestationはprovenanceでありBuild Receiptを代替しない。Receiptは`service.managed-host-output-acceptor`／singleton purpose `managed_host_output_acceptance`で署名する。standard external MCP、Engine Provider Adapter、proposal-only経路はManaged Host Session／Execution Attestationを両方持てず、空Attestation、managed deployment identity差、事前署名result、kindとresult schema不一致、別attempt／別Input Attestationを拒否する。

### 8.4 Local inference境界

Local inferenceは`InferenceDeploymentProfileV1.deployment_kind=local_process_ipc`のclosed branchとしてcloud endpointと分離する。起動前にcurrent `model_snapshot_profile_binding`からSnapshot／Profile Record／issuance Headをread-backし、`model_identity.kind=local_weights`、current Local Manifest binding、全weight shard closureと選択encoding branch、config／tokenizer／chat template／special-token map、license／provenance、expiry／revocation、Eval／Schema／Tool conformanceがすべてcurrentであることを検証する。Deploymentの`process_artifact_ref/hash`はLoaderの`runtime_artifact_ref/hash`とImport Qualificationのprocess artifactにexact一致し、Import QualificationのLoader／Local Manifest bindingはDeploymentのLoader binding／Snapshot内Manifest bindingと一致し、License DecisionのManifest bindingも同じでなければならない。五辺の一つでも違う、Snapshotが`provider_model_id`、license／provenance欠落、Receipt失効なら`diagnostic.ai.model-snapshot-binding-mismatch`でprocess起動前に拒否する。

Local branchはprocess artifact ref／hash、current runtime／loader Profile binding、Import Qualification、process／GPU isolation、mutual-auth endpoint、RAM／VRAM／CPU／memory／GPU／disk／temp cache／shard数・総bytes／child process／batch／concurrent inference／wall-time／output上限を全て必須にする。`os_ipc`はnetwork deny-all、`authenticated_loopback`はexact loopback originだけを許して外部networkをdeny-allにし、両branch混在、wildcard bind、LAN bindを拒否する。Local manifestの`model_format_id`はLoaderの`supported_model_format_ids[]` exact一件に一致しなければならない。custom model code、unsafe serialization、archive path escape、runtime network fetch、built-in file／shell tool、MCP proxyを禁止し、すべてのTool実行をMiraikanai Gatewayへ戻す。llama.cppを候補Adapterにする場合も、公式serverでexperimentalかつuntrusted環境非推奨の`--tools`と`--ui-mcp-proxy`を無効のまま固定する（[公式server README](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)）。loaderがweights/config/tokenizer closure外のartifactを要求すれば拒否する。preflight不足は開始前拒否、実行中超過はprocess tree終了＋failed Receiptへ収束させ、System memoryへの無制限spill、GPU共有contextからの隔離省略、Schema非対応Toolの自然言語代替を禁止する。

License文字列の存在だけではActivationしない。`ModelLicenseAcceptanceDecisionV1`はexact license／manifest／Project revision／intended-use profile／distribution planに対し、game生成、commercial use、weight redistribution、output use、required noticeの各許可をR4 License主体が署名する。`approved_use_kinds[]`は`game_authoring | asset_generation | code_generation | evaluation | redistribution`のclosed subsetとする。Project用途が同集合に含まれ、redistribution／output policyがpackage／distribution planと一致し、Decision／Signer／Role／Key／validity／revocationがcurrentな場合だけlocal deploymentを有効化する。`decision_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:model-license-acceptance:sha256:<lowercase-hex>`として導出し、`role.model-license-acceptance.r4`／singleton purpose `model_license_acceptance_decision`、`subject_sha256=SHA-256(JCS(payload))`、signed issued-at／revocation equalityを必須にする。

Local inferenceは既定で`output_reproducibility.kind=not_claimed`とし、completed実行のexact response bytes hash、runtime／model／settings／seedを署名済み`LocalInferenceRunReceiptV1`へ保存するが、同一出力の再現性を主張しない。preflight／runtime failureはfailed branch、null response hash、1件以上のtyped Diagnosticで表す。`deterministic_claim`だけはruntime artifact、GPU profile、inference settings、seed、sampler、thread policy、replay input setを固定し、最低3件の重複しないfresh completed Run Receiptを列挙する。全Run Receiptの同入力bindingと`exact_response_bytes_sha256`が一致する場合だけRepeatabilityを`byte_equal`にする。未固定hardware／runtime、単一hash自己申告、回数だけの申告でdeterministicと表示しない。

fallbackの唯一の正本は`ProviderManifestV1.fallback`であり、`InferenceDeploymentProfileV1`は`fallback_policy_owner=provider_manifest_only`として別bindingを持たない。`disabled`またはcurrent exact `fallback_provider_manifest_binding`だけを許し、fallback Manifestは元Manifestと別logical ID、self edgeなし、fallback先の`fallback.kind=disabled`、最大depth 1とする。chain／cycle、二段fallback、Deploymentだけの差替えを拒否する。切替前にfallback ManifestからProvider Runtime、Inference Deployment、Model Snapshot、Tool Projection、privacy／region／retention／costを一つのfull Engine Provider Adapter tupleとして解決し、全Profile Head、Provider／Tool Conformance、Schema／Eval Conformance、Target、expiry／revocationをfresh検証する。元Caller ContextまたはReceiptを流用せず、fallback tupleへ新しいCaller Context／Task Authorizationを発行し、`fallback_max_risk`との積集合と明示User確認を必須にする。Localからcloudへ移る場合は送信data class、Provider、region、retention、費用を再Previewする。Network到達性、timeout、resource exhaustionを理由にcloudへ暗黙送信した場合は`diagnostic.ai.silent-cloud-fallback-forbidden`で失敗し、元Taskの状態とProjectを不変にする。

Gemma、Kimi、Qwen、DeepSeek、Grok、GLMその他のModel familyごとのEngine branchを作らない。任意のProvider／local runtime／Modelは同じProfileと三種のtyped Conformance Receiptで扱う。routeが要求するfull subject tuple Receiptがなければそのrouteは`not_activated`であり、`proposal_only`へ暗黙降格しない。`proposal_only`は、別のcurrent `standard_external_mcp` profile／Grant／Host／Transport Receipt／Schema／Eval ReceiptからGatewayが新Caller Contextを発行できる場合だけである。外部Conformance済みHostがlocal modelを使う経路と、Miraikanai自身がlocal runtimeを配布・運用するCapabilityを分ける。first-party local inferenceはFuture incubationであり、MVPまたは初心者First PlayableのCompletion Gateにしない。

## 9. Preview、technical qualification、人間承認

### 9.1 Preview

AIはStaging CandidateへSchema validate、incremental Cook／Build、targeted Test、操作可能Previewを実行できる。Previewはpreview_onlyであり、Technical Attestation、Approval、Activation権限を作らない。

Evidence再利用は[AI Verification／Provenance](ai-verification-provenance.md)のexact input ruleに従う。AIの「影響しない」という説明、File拡張子、Diff行数でGateを省略しない。

### 9.2 Technical qualification

System Bundle Candidateは実装laneに応じG0–G7を通る。

| Gate | Authority subject |
|---|---|
| G0 Engine integrity | baseline signature、Engine diff 0、Path／Build override 0 |
| G1 Scope／Contract | Requirement、State owner、Capability、Target、Bundle closure |
| G2 Source／Dependency | structured／Native／Shaderの禁止境界、Dependency、License |
| G3 Build | validate／cook再現性、Compiler、Module／ABI、offline Shader |
| G4 Correctness／Memory／Security | invariant、property、fuzz、integration、analyzer、sanitizer、negative fixture |
| G5 Game semantics | Replay、Save／Load、Command／Event、Feature outcome |
| G6 Performance／Reliability | Target Budget、fault、restart、last-known-good |
| G7 Evidence closure | exact Input、Toolchain、Test、Artifactの有効なEvidence |

unexecuted、failed、unknown、expiredが一つでもあればAttestationを発行しない。非適用Gateは固定Policyがnot_applicable_by_policyと理由を出す。AI Profile全体の成功率を個別Candidateのhard Gateへ転用しない。

Builder AI、Reviewer AI、Adversarial Test AIは技術承認できない。Engine-owned Validator／Runnerは個別Gateを判定し、[AI Verification／Provenance](ai-verification-provenance.md)のSystemQualificationReceiptV1へEvidence closureを束ねる。Policy Serviceだけが、そのReceiptの署名、subject、result、freshness、gate_policy_hashをSecurity Policyと照合し、SystemTechnicalAttestationV1へ署名する。

### 9.3 Approval hierarchy

| Contract | 対象 | 失効条件／権限 |
|---|---|---|
| SystemTechnicalAttestationV1 | 一つのSystem Bundleに対するPolicy判断とSystemQualificationReceipt hash | 参照Receiptの失効、subject／Policy差、Source、Definition、Contract、Target、Budget、Baseline変更で失効。Policy Serviceだけが署名 |
| FeatureIntegrationAttestationV1 | 複数Systemで成立するUser-visible Feature | 構成System／Graph／Replay／Save／Budget変更で失効。System適格化を代替しない |
| HumanGameplayApprovalV1 | system、feature、またはexact game candidate | result Candidate hash、ChangeSet、Requirement、Capability、Preview、Target、制限、期限へ限定 |
| GameCandidateManifestV1 | baseline、Project revision、全Attestation、Asset、Target artifact、whole-game Gate | 欠落、失敗、stale、Target mismatchなら候補化禁止 |
| GameActivationReceiptV1 | previous／new Candidate、Approval coverage、Attestation closure、Rollback | Activation Serviceだけが署名しpointerを原子的に切替 |

SystemTechnicalAttestationV1は次を固定する。

```text
SystemTechnicalAttestationPayloadV1
  attestation_id, project_revision, engine_baseline_hash
  system_contract_ref, system_bundle_hash, implementation_kind
  capability_scope_hash
  system_qualification_receipt_hash
  gate_policy_hash, result
  policy_service_subject_ref
  policy_service_role_ref
  issued_at
  revocation_snapshot_ref

SystemTechnicalAttestationV1
  payload: SystemTechnicalAttestationPayloadV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=system_technical_attestation)
```

implementation_kindは`gameplay_definition | native_game_module | project_shader_module | project_shader_technique | hybrid | target_specialized_set`のclosed setである。`project_shader_module`はS2／S3 Moduleだけ、`project_shader_technique`はS4／S5 Techniqueを含むSystem、`hybrid`はGameplayDefinition、Native、Project Shaderのうち二種類以上を一つのSystem Bundleへ結ぶ場合に使う。SystemTechnicalAttestationV1はEvidence bundleを再掲せず、exactly oneのSystemQualificationReceiptV1をhash参照する。Policy ServiceはReceiptのproject_revision、engine_baseline_hash、system_contract_ref、system_bundle_hash、implementation_kind、capability_scope_hash、gate_policy_hashがAttestation subjectと一致し、resultがpassで、署名用途がsystem_qualificationであり、失効していない場合だけ署名する。

関係は一方向である。

    Verification Runner
      -> SystemQualificationReceiptV1 [Evidence closure、権限なし]
      -> Policy Service validation
      -> SystemTechnicalAttestationV1 [Policy判断、Receipt hash参照]

SystemQualificationReceiptV1はAuthorization、Approval、Promotion、Activation権限を与えない。SystemTechnicalAttestationV1はReceipt内のTest、Performance、Provenance、Target artifact Fieldを複写せず、Receiptなしでは成立しない。

FeatureIntegrationAttestationV1は次を固定する。

```text
FeatureIntegrationAttestationPayloadV1
  feature_id, requirement_ids[]
  constituent_system_attestation_hashes[]
  dependency_graph_hash, integration_test_root_hash
  replay_root_hash, save_replay_impact_hash
  behavior_budget_receipt_hashes[]
  result
  policy_service_subject_ref
  policy_service_role_ref
  issued_at
  revocation_snapshot_ref

FeatureIntegrationAttestationV1
  payload: FeatureIntegrationAttestationPayloadV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=feature_integration_attestation)
```

HumanGameplayApprovalV1は次を固定する。

```text
HumanGameplayApprovalPayloadV1
  approval_id, approval_subject_kind
  approval_subject_hashes[]
  base_game_candidate_hash?
  result_game_candidate_hash
  covered_change_set_hash
  approved_requirement_ids[]
  approved_capability_scope_hash
  reviewed_replay_ids[], reviewed_target_profiles[]
  known_limitation_ids[]
  approver_subject_ref
  approver_role_ref
  issued_at, expires_at
  revocation_snapshot_ref

HumanGameplayApprovalV1
  payload: HumanGameplayApprovalPayloadV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=human_gameplay_approval)
```

approval_subject_kindはsystem_bundle、feature_bundle、game_candidateのclosed setである。Approvalは常にresult Game Candidateへ結び付け、別Candidateへ流用しない。

GameCandidateManifestV1は次を固定する。

    candidate_id, engine_baseline_hash, project_revision
    game_spec_hash
    system_attestation_hashes[]
    feature_attestation_hashes[]
    target_profile_hashes[]
    structured_package_hashes[]
    asset_package_hashes[], game_binary_hashes[]
    whole_game_test_receipt_hash
    target_package_receipt_hashes[]
    rollback_candidate_hash?

Target別Artifact配列はtarget_profile_hashesと同じ順序、同じ件数にする。Target共通byte列でも各slotへhashを明示し、暗黙fallbackを作らない。

GameActivationReceiptV1は次を固定する。

```text
GameActivationReceiptPayloadV1
  activation_id
  previous_candidate_hash?, activated_candidate_hash
  human_gameplay_approval_hashes[]
  approval_coverage_hash, attestation_closure_hash
  rollback_candidate_hash?, activated_at
  activation_service_subject_ref
  activation_service_role_ref
  revocation_snapshot_ref

GameActivationReceiptV1
  payload: GameActivationReceiptPayloadV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=game_activation_receipt)
```

previous_candidate_hashとrollback_candidate_hashは、先行Candidateが存在しない初回Activationだけ省略できる。省略を空hashやplaceholderで表現しない。current pointerが既に存在する場合、Activation Serviceは両Fieldを省略したReceiptの発行を拒否する。初回Activationの起動失敗はlast-known-goodを持たないため、current pointerを未設定へ戻し、該当Candidateを有効化しない。

これらのApproval／Activation構造とAuthorityは本文書が所有する。五wrapperはinline署名Fieldを持たず、`signed_record.subject_sha256=SHA-256(JCS(payload))`を必須にする。Task Authorization、System Technical、Feature Integrationは各payloadのPolicy Service subject／Role、`issued_at`、revocation snapshot、Human Approvalはapprover subject／Role、`issued_at`、revocation snapshot、Game ActivationはActivation Service subject／Role、`activated_at`、revocation snapshotを`MirakanSignedRecordV1`の対応Fieldとbyte equalityにする。purposeごとにsingleton Role／Key allowlistを分離し、別purposeの有効wrapper、payload signer自己申告、inline algorithm／key／signatureを代用しない。共通署名Record、Verification／Generation／Review／Promotion Evidence envelope、Receipt hash連結、保持は[AI Verification／Provenance](ai-verification-provenance.md)が所有する。

初回制作はAIがGame全体をStagingで完成できる。System技術適格化とFeature統合検証の後、人間にはexact Game Candidate全体を一回提示できる。更新は変更影響Graphから最小System／Feature closureを求めるが、どのApprovalも最終result Candidate hashとRollback先へ結び付ける。

初心者へC++ Source reviewを要求しない。既定UIは、System／Featureの目的、読取／変更State、State owner、Capability、File／Network／OS非使用、Gateの成功／未実行、Replay／Preview、性能、制限、未対応Target、Fallback、Candidate hash、Rollbackを平易に示す。「AIが安全と判断した」等の根拠不明表示を禁止する。

面白さ、操作感、難易度、Art、Game意図は人間のGameplay approval対象であり、CompilerやAIで代替しない。

### 9.4 Code owner assignmentとapproval

`CodeOwnerRoleRegistryV1`は`entries[] { role_ref, allowed_scope_kinds[], allowed_decision_kind, required_independence_class }`だけを持つclosed registryである。`allowed_scope_kinds[]`は1件以上、重複なしunsigned byte順で、次の3 entryだけを開始値とし、Roleの追加またはqualification policy変更はR4 Product Decisionとする。

| Role ID | 対象Scope | 許可する判断 | 分離条件 |
|---|---|---|---|
| `role.code_owner.native_module` | assigned Native module／path | bounded C++ exact Diffの承認／拒否 | Source生成者と別subject、または承認済みindependence policy |
| `role.code_owner.project_shader` | assigned Shader module／Technique／path | Project Shader exact Diffの承認／拒否 | Source生成者と別subject、または承認済みindependence policy |
| `role.code_owner.independent_source_reviewer` | assignmentが要求するNative／Shader review | `review_receipt_ref`発行 | Builder／生成Model／Source Workerと別subject |

```text
CodeOwnerAssignmentV1
  assignment_id
  subject_identity_ref
  role_ref
  path_or_module_scope_refs[1..256]:
    OperationMutationTypedScopeRefV1
  qualification_receipt_ref
  independence_policy_ref
  valid_from
  expires_at
  revoked_at: canonical UTC | null

CodeOwnerAssignmentRecordV1
  subject: CodeOwnerAssignmentV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=code_owner_assignment)

CodeOwnerAssignmentRecordRefV1
  exact MirakanSignedRecordRefV1(purpose=code_owner_assignment)

CodeOwnerSourceRevisionRefV1
  source:
    source_kind: native_module
      revision_ref:
        exact ArtifactRefV1(
          artifact_kind=project_native_source_revision,
          schema_version=1)
    | source_kind: project_shader
      revision_ref:
        exact ArtifactRefV1(
          artifact_kind=project_shader_source_revision,
          schema_version=1)

CodeOwnerBuildReceiptSetV1
  build:
    source_kind: native_module
      receipt_refs[1..64]: NativeModuleBuildReceiptRefV1
    | source_kind: project_shader
      receipt_refs[1..64]: ProjectShaderBuildReceiptRefV1

IndependentSourceReviewV1
  assignment_ref: CodeOwnerAssignmentRecordRefV1
  exact_diff_hash: Sha256DigestV1
  source_revision: CodeOwnerSourceRevisionRefV1
  build_receipts: CodeOwnerBuildReceiptSetV1
  independence_policy_ref
  decision: pass | fail
  issued_at

IndependentSourceReviewRecordV1
  subject: IndependentSourceReviewV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=independent_source_review)

IndependentSourceReviewReceiptRefV1
  exact MirakanSignedRecordRefV1(purpose=independent_source_review)

CodeOwnerApprovalV1
  assignment_ref: CodeOwnerAssignmentRecordRefV1
  exact_diff_hash: Sha256DigestV1
  source_revision: CodeOwnerSourceRevisionRefV1
  build_receipts: CodeOwnerBuildReceiptSetV1
  review_receipt_ref: IndependentSourceReviewReceiptRefV1
  decision: approved | rejected
  issued_at

CodeOwnerApprovalRecordV1
  subject: CodeOwnerApprovalV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=code_owner_approval)

CodeOwnerApprovalRecordRefV1
  exact MirakanSignedRecordRefV1(purpose=code_owner_approval)
```

`CodeOwnerAssignmentV1`は上記9 Fieldだけを持つclosed subject schemaであり、9 Fieldをすべてrequired、`path_or_module_scope_refs[]`を1件以上256件以下の重複なし`scope_type_ref`／`scope_artifact_ref` unsigned byte順、unknown Fieldを拒否とする。各`scope_artifact_ref`はscope type固有のID／version／content hashを持つclosed payloadへ解決し、Domain task carrierのScope refとbyte equalityにする。display path、glob text、Module名、配列indexからScope refを補完しない。`revoked_at`だけがnullableで、Field省略、空文字、sentinel時刻を`null`へ補正しない。このsubjectのcanonical hashを`CodeOwnerAssignmentRecordV1.signed_record.subject_sha256`へ束縛し、署名を含む完成Assignment Record hashを`CodeOwnerAssignmentRecordRefV1.signed_record_hash`およびrevocation判定に使う。裸のAssignment subject、hash-only ref、別purposeの有効署名をAssignment refとして受理しない。

`qualification_receipt_ref`は、Assignment発行前に固定したsubject identity、Role、training／eligibility evidence、許可Scope種別およびindependence-class closureをsubjectにするQualification Receiptだけを参照する下流projectionである。Qualification Receiptのsubject／payloadは`assignment_id`、具体的な`path_or_module_scope_refs[]`、Assignment subject hash、完成Assignment Record hash、署名、期間、revocation stateを含めてはならず、AssignmentからReceiptへ戻るedgeは存在しない。依存順は`identity／Role／eligibility／independence qualification closure -> Qualification Receipt -> CodeOwnerAssignmentV1 -> MirakanSignedRecordV1`の一方向だけとし、ReceiptからAssignmentまたは署名済みRecordへの逆参照をPolicy Serviceは拒否する。

`CodeOwnerAssignmentV1.role_ref`は`CodeOwnerRoleRegistryV1.role_ref`のexact一件で必須であり、display name、前方一致、ScopeからのRole推測を拒否する。`path_or_module_scope_refs[]`の各refはRole entryの`allowed_scope_kinds[]`の一件に一致し、Native RoleへShader scope、Shader RoleへNative scope、independent reviewerへDiff approvalを与えない。`revoked_at`はField省略を許さないrequired nullableで、`null`だけが未失効を表し、canonical UTC時刻ならその時刻以後revokedである。`valid_from <= evaluation_time < expires_at`かつ`revoked_at=null`の場合だけactiveとし、未来開始、期限切れ、失効を別stateへ推測補正しない。

`CodeOwnerAssignmentV1`は認証済みProject role administratorの要求をApproval ServiceがRole、Qualification／Scope／independence registryと照合して署名した場合だけ有効である。Policy Serviceは署名済みAssignmentのsubject、exact `role_ref`、全Scope、Qualification Receiptのsubject／Role／Scope／freshness、independence policy、期間を検証し、信頼済みrevocation registryの署名済みlatest headをcurrent snapshotとして毎回read-backする。発行時snapshotからcurrent sequenceまでのchainが連続し、current snapshotの署名、issuer、sequence、freshnessが有効であることを必須にする。subject内の`revoked_at=null`でもcurrent snapshotがAssignment Recordまたはsubject identityをrevokedとした場合はSource Worker起動とApproval使用を拒否する。current snapshotのmissing／stale／invalid、sequence rollback／gap、missing／unknown Role、RoleとScope kindの不一致、QualificationのRole／Scope差、`revoked_at` non-nullもfail closedにする。AI、Source Worker、割当対象者は自己発行／自己revocationできず、Policy Serviceは主体選定を代行しない。

`IndependentSourceReviewV1`は上記7 Fieldをすべてrequiredにするclosed subject schemaである。`decision=pass`はAssignment、Diff、Source revision、Build集合、independence policyがreview時点で一致し、`IndependentSourceReviewRecordV1.signed_record.signer_role_ref=role.code_owner.independent_source_reviewer`、SignerがSource Worker／生成Model subject／Builder／Code Ownerと分離される場合だけ許す。完成subject JCS hashを署名subject、完成wrapper hashを`IndependentSourceReviewReceiptRefV1.signed_record_hash`へ一致させ、裸subject、hash-only ref、別purpose、別Diff／Build集合を拒否する。

`CodeOwnerApprovalV1`は上記7 Fieldをすべてrequiredにするclosed subject schemaで、unknown Field、空Build集合、裸のAssignment／Review refを拒否する。`decision`は`approved | rejected`のclosed enumであり、共通署名、signer identity、revocationは`CodeOwnerApprovalRecordV1.signed_record`が所有する。`source_revision.source_kind`と`build_receipts.source_kind`は同じ値でなければならず、Native SourceへShader Build Receipt、Shader SourceへNative Build Receiptを混在させない。Build refは[Core architecture](../02-foundation/core-architecture.md)のcanonical aliasを再利用し、Nativeはexact `purpose=operation_receipt:operation.build.request_native_module`、Shaderは`purpose=operation_receipt:operation.build.request_project_shader`、どちらも`source_authorization.kind=prepromotion_candidate`を必須にする。Promotion Receiptを要求する`promoted_revision` BuildをPromotion前のCode Owner判断へ流用しない。Policy ServiceはAssignmentのsubject、Role、Scope、qualification、independence、期間、revocationと、Approvalのexact Diff、Source revision、全Build Receipt、`decision=pass`の`IndependentSourceReviewReceiptRefV1`を照合する。完成Approval subjectのJCS SHA-256は`CodeOwnerApprovalRecordV1.signed_record.subject_sha256`、完成wrapper hashは`CodeOwnerApprovalRecordRefV1.signed_record_hash`とbyte equalityにする。Source、Diff、Build input、Toolchain、Target、Assignment、Reviewのいずれかが変わればApprovalを失効させる。Code owner判断はG0–G7、Technical Attestation、Human Gameplay Approvalのいずれも代替しない。

AIが新規NativeGameModuleまたはProject Shader Sourceを生成する前に有効なAssignmentがなければTaskを`AwaitingCodeOwner`、Editorを`awaiting_code_owner`にしてSource Workerを起動しない。上級者はowner割当を待つかTaskを取消できる。初心者MVPではSource laneへ進めず、同じRequirementをDefinition-firstで再解決し、表現可能ならGameplayDefinition、Qualification済みのexact Pack／Variantがあればprequalified Pack、どちらもなければ`capability_unavailable`にする。fallbackはPlan／Diffへ明示し、新規Native／Shaderを既存Packと偽装しない。

初心者のGameplay intent承認はCode owner承認を代替せず、初心者にC++／Shader Source reviewを要求して代用しない。Beginner First PlayableのDefinition／prequalified Pack gateと、Advanced Project Source Activation、およびExternal AgentのProposal／Source／Promotion gateは独立である。前者の合格を理由に後者をactivateしない。

## 10. PromotionとActivation

Source Worker終了後、Brokerは返却DeltaをHost Stagingへ適用し、base commitとの差分を再計算する。AI提出File listやguest diffを正本にしない。

Promotionは次をすべて確認する。

1. base commit、Task input revision、Contract／Policy／Profileが一致する。
2. 実Diffの全Path、binary、generated、submodule、Dependency、Policy変更がEnvelope内である。
3. Test／Schema／Budget／Securityを弱める変更がRisk再分類される。
4. required GateとApprovalが同じDiff hashへ結び付く。
5. 昇格後のAuthoritative branch／Projectをread-backし、同じ結果を再検証する。
6. 失敗時はAuthoritative state不変またはbefore hashへのRollbackを確認する。

AI Processへ.git write、Git commit、branch ref、tagを許可しない。Project revisionはC++ Gateway、Source commitはPromotion Service、Candidate pointerはActivation Serviceだけが作る。

Activationは全変更Systemのdependency closureが有効なAttestationとHuman Approvalで覆われ、全Recordが同じCandidate rootを参照する場合だけ行う。構成Artifactを個別copyせず、Manifest rootをcurrent pointerへ原子的に切り替える。起動失敗時は署名済みlast-known-goodへRollbackする。

Engine baseline更新は全Attestationを失効させ、明示Migration、全再検証、Game Candidate全体のGameplay approvalを必要とする。Save／Replay意味変更は全影響System、Migration、Feature、Game Candidate closureを再承認する。

## 11. Failure、拒否、停止

| Failure | 正規動作 |
|---|---|
| Provider timeout／rate limit | 状態保持。Manifest許可内だけretry／fallback |
| refusal／incomplete／strict Schema不成立 | Proposalを作らずDiagnostic |
| Context不足／stale revision | 質問または新Task。Mustを削らない |
| 同一blocking反復／修復上限 | 自動loop停止、Attempt保持 |
| Validator crash／Test infrastructure failure | 合格扱いせずinfrastructure_error |
| Sandbox／hardware isolation unavailable | Source Workerを起動せず停止 |
| Baseline hash／signature不一致 | Build／Preview／Activation停止 |
| Engine private／OS／Vendor API要求 | Source Gate拒否。公開Capabilityへ再設計 |
| 公開Capabilityで意味同等に実現不能 | capability_unavailable |
| Sanitizer／fuzz／Replay／Budget failure | Artifact隔離、Promotionなし |
| Human rejection | Rejected終端。修正は新Task |
| Revision drift | 現Task停止、新ContextとEnvelope |
| Receipt／hash／Approval mismatch | Security event、Promotion拒否 |
| Candidate／Device generation／Package Receipt mismatch | Operation開始前に失敗。最新対象へ自動追従しない |
| Host／Model／Deployment Profile失効 | 当該Caller Contextの新規Tool／推論呼出しを停止して`not_activated`。`proposal_only`は、別のcurrent `standard_external_mcp` Host／Transport／Tool／proposal-only Authority Profile、Grant、ConformanceからGatewayが新Caller Contextを発行した場合だけ |
| Code owner assignment／approval不在または失効 | `AwaitingCodeOwner`。Source生成／Promotionなし、BeginnerはDefinition／prequalified Packへ再Plan |
| Local inferenceから未確認cloud fallback | `diagnostic.ai.silent-cloud-fallback-forbidden`、送信0、Project不変 |
| Runtime fault | Session停止、Artifact invalid化、last-known-goodへRollback |
| Credential／Authorization侵害成功 | Incident、Provider／Worker停止、Key／Envelope失効、Artifact隔離 |

AIはGate失敗を直すためにEngine、Validator、Engine-owned Test、Budget、Policy、Approval条件を変更してはならない。未知欠陥が絶対にない、AIが意図を完全理解する、全将来入力で正しい、Gameが必ず面白いとは保証しない。

## 12. Security fixtures

最低限、次のnegative fixtureで「AIが試さない」ではなく「試してもOS／Broker／Policyが阻止し、正規状態が不変」を証明する。

三Conformance Receiptは各routeの一つのpositive vectorから次のexact一原因negative fixtureを生成する。各fixtureはReceipt発行前とCaller Context発行前の両方で拒否し、Receipt Registry、Profile Head、Caller Context Registry、Tool Catalog、Task、Projectをbyte不変にする。

| fixture ID | positive vectorから変えるexact一Field | 必須拒否点 |
|---|---|---|
| `fixture.ai.conformance.host-version-substitution` | Host `exact_version` | Host／Transport Receipt subject |
| `fixture.ai.conformance.host-binary-substitution` | local Host `client_binary_ref/hash` | Host／Transport Receipt subject |
| `fixture.ai.conformance.transport-version-substitution` | `mcp_protocol_version` | Host／Transport Receipt subject |
| `fixture.ai.conformance.transport-endpoint-substitution` | IPC endpointまたはHTTP／Tunnel origin | Host／Transport Receipt subject |
| `fixture.ai.conformance.transport-auth-substitution` | Transport ProfileのACL／OAuth metadata／mTLS private-service／session policyまたはbinding／Head | Host／Transport Receipt subject |
| `fixture.ai.conformance.provider-substitution` | Provider Runtime Profile binding／issuance Head | Provider／Tool Receipt subject |
| `fixture.ai.conformance.deployment-substitution` | managed deployment identityまたはInference Deployment binding | Provider／Tool Receipt subject |
| `fixture.ai.conformance.model-substitution` | Model Snapshot binding／issuance Head | Provider／Tool Receipt subject |
| `fixture.ai.conformance.tool-projection-substitution` | projection ref/hashまたはTool／Operation／input／result set hash | Provider／ToolまたはSchema／Eval Receipt subject |
| `fixture.ai.conformance.test-suite-substitution` | suite／fixture ref/hashまたはsuite kind | 該当Receipt subject |
| `fixture.ai.conformance.partial-case-execution` | required Caseを一件未実行、またはexecuted／passed／negative set hashを一件だけ差し替え | Result validator |
| `fixture.ai.conformance.target-substitution` | execution Targetまたはartifact Target ref | 該当Receipt subject |
| `fixture.ai.conformance.failed-result-as-pass` | failed caseを残して`result=pass`または`overall_result=pass` | Result validator |
| `fixture.ai.conformance.signer-role-substitution` | Qualifier Role／purpose／Keyの一つ | signed wrapper verifier |
| `fixture.ai.conformance.expired-or-revoked` | expiry境界または`revoked_at`／snapshot | Context発行およびdispatch read-back |
| `fixture.ai.conformance.profile-head-drift` | Profile recordを維持してissuance Headだけ更新 | Context発行およびdispatch read-back |
| `fixture.ai.conformance.caller-context-tuple-substitution` | Receipt発行後のContext tupleを一Field変更 | Gateway tuple byte-equality check |
| `fixture.ai.conformance.standard-route-provider-injection` | standard routeのProvider／Deployment／Model／Provider Receiptをnon-null | route union validator |
| `fixture.ai.conformance.engine-route-transport-injection` | Engine routeのMCP Transport／Grant／Host Receiptをnon-null | route union validator |

この表は受入契約のfixture logical IDを予約するだけで、現時点のFixture Set／Case／Suite Artifact、pass／fail Receipt、Caller Contextをmaterializeしない。各Suiteは自分の`suite_kind`に適用可能なpositive vectorと一原因negative fixtureをclosed Case inventoryへ固定し、同じrouteで要求される全Suiteのunionが上表のroute-applicable fixtureをexactに覆わなければならない。Phase 4のEngine Provider Adapter GateはProvider／Tool＋Schema／Eval、Phase 5のstandard external MCP GateはHost／Transport＋Schema／Evalを消費する。managed external HostのHost／Transport＋Provider／Tool＋Schema／Evalおよび全managed-specific negative fixtureはFuture `future.capability.managed-external-host-execution`をActive Definitionへ移行するときに追加する専用Work Package／Gateだけが消費し、Phase 4／5の成立またはproposal-only Receiptを代用しない。いずれもpartial Fixture／Case set、自己申告pass、別Targetの結果を拒否する。

- Prompt、Source comment、Asset名、Issue本文からの指示昇格と自己承認。
- malformed Tool、unknown Operation、Tool name collision、偽annotation。
- absolute／parent／UNC path、device name、case collision、symlink、junction、reparse、hardlink、submodule、archive、ADS。
- .git、Policy、Credential、SSH、cloud config、Engine packageへのwrite。
- DNS rebinding、Redirect、loopback、private address、unexpected domain、raw socket。
- Environment、process、log、crash dump、Context、Tool childからのSecret取得。
- stale Approval、nonce replay、expired Envelope、hash／revision差替え。
- long-running grantの未知Operation、二重消費、Input／Runner／Destination差替え、期限後Process。
- Test削除、skip、Assertion緩和、Budget引上げ、Validator／Policy無効化。
- fork／background process、timeout後Process、resource exhaustion。
- Baseline、SDK、Validator、Policy、Engine-owned Testの1 byte変更。
- Bounded Nativeのprivate／OS／Vendor API、未知binary import、direct allocation／thread／dynamic load。
- Bounded ShaderのManifest外pass／Resource／UAV access／side effect、native resource、private include。
- malicious Build scriptからSigning key／Store tokenへの読取と持出し。
- Signing ServiceへのSource、Build script、shell、Manifest外File、別Artifact。
- Upload Serviceへの未署名Artifact、古いEvidence、別Application／Channel／Version、過剰Role。
- 未Activation OperationのTool公開または内部成功。
- AI E2E familyの一部だけ、Role／Signer rowだけ、Provider aliasだけ、MCP Toolだけを先行Activationする試行。24 family／191候補のplanning partitionとProduct `ExecutionSurfaceBindingV1.required_operation_ids[]`を再計算し、該当Capabilityを`not_activated`のまま正規状態不変で拒否する。
- External Client Catalogへ`operation.authoring.changeset.commit`、`operation.project_source.promote_revision`、Approval／Activation／Signing／Release Toolを一件ずつ混入する試行。managed routeを含め全てCatalog compile時に拒否する。
- `ProjectChangeSetV1`のstale base revision、Scope外primitive、unknown union branch、4096件超過、8 MiB超過、Validation／Preview hash差、Approval対象ChangeSet差を一原因ずつ拒否し、Project revisionとPublic Markerを不変にする。
- Native／Shader Source WorkerをCode Owner Assignmentなしで起動する、Worker自己申告DiffをApprovalへ渡す、Broker再計算Diff／before-after tree／Build Receipt／Code Owner Approvalの一byteをPromotion前に差し替える試行。Source revisionとProjectを不変にする。
- Package成功payloadからProject Validation、Cook、Game Candidate Build、Candidate Test、selected SourceのPromotion／Native／Shader Build Receiptを一件ずつ欠落または別Project／Candidate／Target／Toolchainへ差し替える試行。Package artifact publication前に拒否する。
- Build Candidate六familyのActivation acceptanceでは、Core architecture §9.2のexact六行についてnamed Input、Operation ID、payload contract、完成Receipt alias、execution Authority、Signer Role、singleton purpose Keyを同一行へbindし、Validate→Cook／Source Build→Game Candidate→Test→Packageの各前段集合をset equalityで検証する。各positive fixtureからProject revision、Candidate、Target、Toolchain、Source revision、Promotion Receipt、前段Receipt、fixtureの一つだけを差し替えたnegative fixtureを作り、成功artifact publication前に拒否する。currentでは六候補をdispatch前に`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`で拒否し、Task／Project／artifact byte不変を確認する。
- Build Candidate六familyのidempotency fixtureでは同じOperation／Authorization／key／request hashを同一Taskへ収束し、同じkeyの別request hashを`MIRAKAN-BUILD-IDEMPOTENCY_CONFLICT`で拒否する。cancel fixtureでは不可逆境界前だけ`cancelled` Receipt、境界後は`succeeded | failed` Receiptへ収束させる。Task Control familyが未Activationのとき、caller request、Provider alias、signal fileでcancelできないことも確認する。
- 未適格System、未承認Change、別Candidate hashを含むActivation。
- 以下のBuild／Device／Play／Debug／Task固有fixtureは`activation.build_gateway.operation_pipeline.v1`のatomic Activation受入条件である。current fixtureは14候補すべてをdispatch前に`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`で拒否し、Project／Task／Device／export byte不変を検査する。
- Activation acceptanceでは`operation.build.request_package`後にProject revisionまたはCandidate rootを差し替え、古いTaskを継続する試行を拒否する。
- Activation acceptanceではpair済みDeviceを同名の別Deviceまたは新generationへ交換し、古いinstall／launch／reset／remote Debug grantを再利用する試行を拒否する。
- Activation acceptanceではCandidate、Target、artifact hashのいずれかが異なるPackage Receiptによるinstall／resetを拒否する。
- Activation acceptanceではInstallのPackage Receipt／artifact、LaunchのInstall Receipt／artifact、SmokeのPackage／Install／Launch Receiptまたはfixtureについて、ref／hash／署名／payload contractを一原因ずつ差し替え、各後段を副作用前に拒否する。
- Build familyのatomic Activation acceptance fixtureでは、Operation Receipt mapping 14件すべてについてexact purpose、subject Operation ID、payload contract、完成Receipt alias、execution Authority、Signer Role、singleton-purpose Keyの同一行bindingと署名成功を確認する。currentでは14件すべてが未materializeであることを確認する。
- Activation acceptanceでは各positive fixtureをbaseに、別Operation purpose、unknown／generic purpose、別実行AuthorityのKeyをそれぞれ一原因だけ変更し、payloadとsubjectが他は同一でも署名検証を拒否する。`OperationReceiptEnvelopeV1`のOperation IDとpayload型不一致、async 11件の`task_id` missing、sync Control 3件の`control_invocation_id` missing、両ID present、Control対象Task IDのEnvelope `task_id`への流用も一原因ずつ拒否する。
- Activation acceptanceではconsentまたはR3 Approvalなしの`operation.device.install`／`operation.device.reset_data`と、install Approvalをlaunch／smoke／Debugへ権限継承させる試行を拒否する。
- Activation acceptanceではEvidence ref不在、別Session／revision、gap隠蔽、reproductionなしの偽`validated_cause`を`operation.debug.validate_finding`へ渡し、`diagnostic.debug.finding-evidence-invalid`で拒否する。
- `administrative_enablement=disabled`なのに過去pass Receiptからrouteを有効化する、`enabled`だけでReceiptなしに対応表示する、standard routeを`supported`、managed／Engine routeを`proposal_only`へ自己申告する試行を拒否する。exact full tuple＋current pass Receipt＋freshness／revocation＋Activation／Authorizationが成立するpositive fixtureだけを、standard=`proposal_only`、managed／Engine=`supported`へ導出し、managed不成立時は別standard tupleなしに降格しない。
- 期限切れ／失効／binary hash不一致の`ExternalClientSecurityProfileV1`、Engine build／process／OS policy差の`EngineAiHostSecurityProfileV1`、またはinvalidな`ProviderManifestV1`、`InferenceDeploymentProfileV1`、`ModelSnapshotProfileV1`によるTool／推論呼出し。routeに対するHost schema差、External routeのMCP Transport欠落、Engine routeのnon-null MCP Transport／Grant、Deployment／Provider Manifestの`model_snapshot_profile_binding`差、issuance Head差、local deploymentから`provider_model_id` Snapshot参照、Snapshotのweight shard／encoding branch／license／provenance／Conformance欠落または失効を一原因ずつ拒否する。
- Caller ContextのFreshness binding欠落、TTL 600秒超過、source expiryを超えるContext、Profile Head更新後のcurrent実行を拒否し、同じ過去Receiptは履歴監査で`authentic_but_stale`として検証できることを確認する。
- 全routeのTask Attempt chainで、initialのcount非0、repair Reservationのprevious／sequence／count gap、別Task previous、reserved中の二重開始、stale Head、同一Headへの並行CAS、Host／Transport／Grant／Model／Provider／Attempt変更によるcount reset、上限超過、期限前abort、結果ReceiptとReservation差、unsigned latestを一原因ずつ拒否する。standard MCPではProvider／Model／完全responseを`StandardExternalProposalReceiptV1`へ記録またはGeneration Receiptへ捏造しない。
- managed routeでpre Session Attestation欠落の実行開始、実行前の偽post Execution Attestation、処理後post欠落resultのStaging受入れ、別Task／attempt／Input／typed resultを指すpost Attestationを一原因ずつ拒否する。
- Managed AcceptanceからTask SpecificationまたはAuthorization hashを落とす、Host AttestationだけでBuild成功を申告する、別Project／Candidate／Target／Toolchainの`PackageReceiptV1`を束縛する、failed typed resultをStagingへacceptする試行。
- Local Deployment、Loader、Import Qualification、Snapshot内Manifest、License Decisionのbindingまたはprocess artifactを一辺だけ差し替える試行。OS IPCでnetworkを許す、loopback branchをwildcard／LANへbindする、built-in file／shell toolまたはMCP proxyを有効化する試行。
- RepeatabilityでRun Receiptが3件未満、重複ref、入力／runtime／model／settings差、exact response bytes hash差、failed run混在を一原因ずつ拒否する。
- hosted ChatGPT Workをdirect local STDIO Hostとして登録する、local Codex `config.toml`をhosted plugin Tool一覧として読む、hosted ChatGPT Workとdesktop Codex hostのProfile／Receiptを流用する、またはChatGPT desktop Codex／Codex CLI／IDE、Claude Desktop／Claude Code／CursorをConformance Receiptなしで`supported`表示する試行。
- Local runtime timeout、RAM／VRAM不足、Tool Schema不一致を契機に、Preview、User確認、新Authorizationなしでcloudへ送る試行。送信byte 0と`diagnostic.ai.silent-cloud-fallback-forbidden`を確認する。
- Assignmentの`role_ref`欠落／unknown、Native RoleへのShader scope、Shader RoleへのNative scope、Qualification ReceiptのRole／Scope差、`revoked_at` Field省略、`revoked_at` non-null、unknown extra Field、期限切れ、current snapshotでAssignment Recordだけをrevoked、current snapshotでsubject identityだけをrevoked、current snapshotのmissing／stale／invalid、または別Diff／Source revisionの`CodeOwnerApprovalV1`を一原因ずつ注入する。各fixtureはSource Worker起動前またはPromotion前に拒否し、`revoked_at=null`でもcurrent snapshot失効を上書きしない。

## 13. 完了条件

- AIが変更できるTaskSpecificationと、変更不能な署名Authorizationが分離される。
- R0–R5、A0–A3、Task state、修復停止、Expiry、read-backがPolicy testで強制される。
- Game制作のTool、Filesystem、Worker、ContextからEngine write経路とEngine sourceが除かれる。
- API、MCP、CLI、EditorのProposalが同じGatewayとPolicyへ到達する。
- Build familyを将来activateする場合、Package／Device／Play／Debug／Taskの14 Operationが同じ`OperationTaskV1` binding、Receipt、cancel規則へ到達する。currentでは全14候補が未materializeである。
- AI E2E closureを将来activateする場合、ChangeSet 4、Native Source 5、Source Promotion 1、Candidate Build／Test 6、GameplayDefinition 6、Asset 10をfamily単位で登録し、Product Matrixが要求するfamilyの一件でも非operationalならCapability昇格を拒否する。Commit／PromotionのProvider／MCP投影は常に0件である。
- Host／Transport／Provider Runtime／Manifest／Deployment／Model Snapshot／Tool Projection／Authority／Targetがtyped full tupleで評価され、対応表示はrouteが要求する`HostTransportConformanceReceiptV1`／`ProviderToolConformanceReceiptV1`／`SchemaEvalConformanceReceiptV1`のcurrent pass集合に束縛される。
- 全routeのrepair countがsigned Reservation／task-scoped current Head、managed routeのpre／post Attestationが因果順で強制され、route変更や再起動でresetされない。
- Sandbox不能、Baseline mismatch、Capability不足、Credential分離不成立でfail closedになる。
- Project data、Bounded Native、Bounded Shaderの境界とG0–G7適用laneが機械検査される。
- Builder／Reviewer AI、Policy、Approval、Promotion、Activation、Release各Authorityが分離される。
- 初心者がSourceを読まず、意図、Capability、Evidence、Preview、制限、Rollbackを確認できる。
- Code owner不在時にNative／Shader Source生成が停止し、初心者がDefinition-first／prequalified Packへ明示的にfallbackする。
- System、Feature、Game Candidateのhash階層と変更影響失効が強制される。
- current Candidate、Project revision、Git commit、ReleaseをAIが直接作れない。
- Security fixtureが正規状態不変を確認する。

## 14. 一次根拠

- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [RFC 8785 JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785)
- [RFC 7518 JSON Web Algorithms](https://www.rfc-editor.org/rfc/rfc7518)
- [Model Context Protocol 2025-11-25 specification](https://modelcontextprotocol.io/specification/2025-11-25)
- [OpenAI: Model Context Protocol for hosted ChatGPT Work and local Codex hosts](https://learn.chatgpt.com/docs/extend/mcp)
- [Anthropic: Desktop Extensions and local MCP servers](https://support.claude.com/en/articles/10949351-getting-started-with-local-mcp-servers-on-claude-desktop)
- [Anthropic: Connect Claude Code to tools via MCP](https://code.claude.com/docs/en/mcp)
- [Cursor: Model Context Protocol](https://cursor.com/docs/mcp)
- [Ollama: OpenAI compatibility](https://docs.ollama.com/api/openai-compatibility)
- [llama.cpp: OpenAI-compatible server and MCP proxy](https://github.com/ggml-org/llama.cpp/tree/master/tools/server)

外部資料は2026-07-24に検証した。MCPの標準Transport、各Host／runtimeが公開する接続方式、local endpointの性質を確認する根拠であり、Miraikanai固有のRisk、Operation、Approval、Sandbox、Conformance結果を外部製品へ委ねない。OpenAI互換endpointという表示だけではTool／Schema conformanceを意味しない。
