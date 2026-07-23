# Miraikanai Engine AI Security／Approval Contract

- 文書ID: mirakan.arch.ai-security-approval
- 状態: review
- 正本範囲: AI task authorization、Risk、Trust boundary、不変Engine、Sandbox、Credential、Provider／MCP／CLI security、Preview、人間承認、Activation、Promotion、拒否
- 非正本範囲: Eval、Evidence envelope、Provenance、Trace grading、Receipt保持。これらはAI Verification／Provenanceを参照する
- 依存: [Product Plan](../00-product/product-plan.md)、[AI Verification／Provenance](ai-verification-provenance.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Project state](../03-authoring/project-state.md)、[Native game module](../03-authoring/native-game-module.md)、[Project Shader](../06-rendering/project-shader.md)
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

    envelope_version, task_id, spec_sha256
    issued_at, not_before, expires_at, nonce
    risk_class
    contract_set_hash, policy_set_hash
    resolved_profile_hashes[]
    tool_catalog_hash
    allowed_operations[]
    path_grants[]
    network_policy
    dependency_policy
    secret_policy
    resource_limits
    repair_attempt_limit
    required_gates[]
    required_approvals[]
    long_running_grant?
    signature_algorithm, signature_format, key_id, signature

OperationはID＋versionのexact allowlistとし、wildcardを禁止する。Networkはdeny_all、Dependencyはno_change、AI TaskのSecretはno_secret_accessを既定とする。Pathはread／write／create／deleteを別に許可し、Process tree、CPU、memory、wall time、child count、output sizeをhard limitにする。

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

署名RecordはMirakanSignedRecordV1を使用する。初期ProfileはECDSA P-256／SHA-256、RFC 8785 JCS、P1363固定64 byte、base64url without padding、low-S必須である。unknown field、duplicate key、invalid UTF-8、非有限値、高S、未知／失効／用途不一致／期限外Keyをfail closedで拒否する。秘密鍵は専用Service identityのnon-exportable Key storeへ置き、AI／Workerから分離する。AI Orchestratorにはgeneration_receipt用途専用のService identityとnon-exportable Keyだけを割り当て、この用途KeyをVerification、Approval、Promotion、Release用途へ流用しない。ActorのSecret保持禁止はProvider Credential等の可搬Secretを指し、このnon-exportable署名identityを含まない。

Operation Receiptの署名権限は次のclosed `OperationReceiptSignerPolicyV1`だけが与える。

```text
OperationReceiptSignerPolicyV1
  entries[] {
    operation_id
    execution_authority
    signer_role_ref
    allowed_signed_record_purpose
  }
```

全Fieldはrequired、unknown Fieldは禁止し、entriesは次の14件とexact一致させる。

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

各`signer_role_ref`は同じ行のpurpose一件だけを許可する。Public key registryの各`key_id`も、その実行Authority subject、exact Signer Role、`allowed_signed_record_purposes[]`が同じ一件だけのsingletonでなければならない。同じServiceが複数Operationを実行してもRoleとnon-exportable KeyをOperationごとに分離し、generic Operation Receipt Role／Keyを作らない。通常Rotationで新旧Keyを重複有効にする場合も、両KeyのAuthority、Role、singleton purposeを同一に保つ。Coreの14行mapping、MCDのexecution Authority、本PolicyのOperation／Role／purposeが一致しないRecordをVerifierは拒否する。

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

本節の15状態はAI Orchestrator TaskのGovernance stateである。[Core architecture](../02-foundation/core-architecture.md#91-operationtaskv1)の`OperationTaskV1.state = queued | running | cancel_requested | succeeded | failed | cancelled`は個々のPackage／Device／Play／Debug実行ledgerであり、相互の状態名をaliasにせず、Operation Receiptから親Taskへ結果を投影する。

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

状態変更Operationの認可presenceは次のtagged contractに一意化する。

```text
MutationAuthorizationBindingV2
  risk_class: R2 | R3 | R4 | R5
  authorization_ref/hash
  authority_evidence:
    approval:
      approval_ref/hash
    | predelegated:
      predelegation_ref/hash
```

R2は`approval | predelegated`の厳密に一方を許可し、Predelegationは上記Scope、Operation、Project、期限、上限、rollbackとrequest hashを全てcoverしなければならない。R3～R5は`approval`だけを許可し、`predelegated` branchを拒否する。状態変更で`authority_evidence`欠落、両branch混在、期限切れ、Scope不足、request hash不一致の場合は、Domainにかかわらずexact `DiagnosticCodeRefV1`の`diagnostic.approval.required / MIRAKAN-APPROVAL-REQUIRED`を返す。`authorization_ref`だけ、optional `approval_ref`、空hash、callerの「承認済み」文字列を代用しない。R0／R1 non-mutationは本型自体をcanonical omissionする。

### 4.2 Activation Gate

| Activation | 有効にする範囲 | 解放条件の要点 |
|---|---|---|
| A0 Authoring Core | R0–R2構造化編集とGameplayDefinition | 契約、署名Envelope、Gateway、Transaction、Validator／Cooker、Undo、adversarial security fixture |
| A1 Project Source | R3 bounded NativeGameModule、Project Shader、Engine-owned Build template | hardware-VM Worker、Broker、Promotion、Path／Network／Process negative test、G0–G7、署名Attestation |
| A2 Engine Maintenance | 別Repository／AuthorizationのR4 Engine保守 | Game制作DAG外。独立Review、state model、threat／lifetime、full regression、fault／soak |
| A3 Release | R5 merge／tag／sign／Store提出 | 分離Coordinator／Build／Signing／Upload、reproducible Build、SBOM、Provenance、Platform／device Gate |

未解放OperationはTool catalogへ出さず、内部呼出しにもMIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATEDを返す。A3はA2を暗黙依存しないが、R3 Artifactを含むReleaseはA1を必要とする。

## 5. Beginner questions、assumptions、理解条件

質問を次に分類する。

| Class | 動作 |
|---|---|
| Blocking | 回答まで該当Scopeの実装開始不可 |
| High | 推奨案、影響、変更可能性を示し回答を求める |
| Medium | Defaultと仮定を明示して進行可 |
| Low | Project規約から決め、Decisionへ記録 |

初回はBlockingとHighを最大7問へまとめる。超える場合はGame core、Visual、Platform／Businessの順に分割する。初心者向けUIは複雑さを隠せるが、質問、仮定、Risk、Approval、制限を省略してはならない。

大まかなPromptはGameIntentDraft、Capability／Platform／権利／Online／Save／年齢／Performance検査、質問、GameBrief、人間確認、First Playable Scope、実装方式、Staging、Gate、Approvalの順で処理する。

AIの「理解した」という自己申告を状態にしない。GameUnderstandingClosureV1は少なくとも次をexact hashで閉じる。

    project_revision
    game_brief, requirement_set, decision_set
    system_graph, state_owner_map, capability_scope
    save_replay_contract, target_profile_set
    behavior_budget_set, test_plan, evidence_closure
    project_shader_understanding_closure_hashes[]
    unresolved_blocking_question_count
    unresolved_high_requirement_count
    unsupported_capability_ids[]
    disposition: ready_to_stage | capability_unavailable

`GameUnderstandingClosureV1`は終端Recordであり、未解決のBlocking／High質問を含むdraftからは発行しない。質問中の状態はQuestion／Decision draftが所有し、`disposition`へ第三の状態や自由文字列を追加しない。ready_to_stageにはBlocking／High未解決0、RequirementからSystem／Implementation／Test／Artifactまでの追跡100%、State owner重複0、stale Evidence 0が必要である。Project Shaderを含むSystemは全参照Module／Techniqueに有効な`ShaderUnderstandingClosureV1`と`ProjectShaderQualificationReceiptV1`を必要とし、欠落／stale／Target不一致をGame全体の理解で相殺しない。必須Requirementに未対応Capabilityがあればcapability_unavailableにする。

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

Managed modeは`ExternalClientSecurityProfileV1`のidentity branchに応じ、localではClient exact version／binary hash／OS identity／Process・Filesystem・Network分離、hostedではsurface／release channel／tenant-workspace／plan／admin policy／OAuth-OIDC subject・audience／TLS identityを固定し、共通でMCP version、期限、conformance resultを束縛する。Profile不在、branch必須値差、期限切れ、Broker外environmentへのCredential露出、raw socket利用ではそのManaged Caller Contextを拒否して`not_activated`とする。別途currentなstandard-external-MCP用Host／Transport／Tool／proposal-only Authority Profile、Grant、Conformanceが全て存在する場合だけ、Gatewayが`standard_external_mcp` branchの新しいsigned Caller Contextを発行できる。managed ContextのField削除やin-place権限降格で代用しない。

### 8.1 Provider Manifest、Prompt、Repository guidance

Provider／Model名をEngine codeへhard-codeしない。`ProviderManifestV1`は`ProviderRuntimeProfileV1`のtagged adapter（official SDKのexact artifact、raw protocolのexact schema、またはattested external host）、endpoint、resolved Model snapshot、Role、推論設定、Tool／Structured Output projection、Context／output／cost／latency上限、data retention／training use／region／encryption／logging Policy、合格Eval suite、明示fallback、fallback時の最大Riskを固定する。存在しない共通「API／SDK version」を捏造せず、選択branchに必要なversion／artifactだけを検証する。Manifestなし、adapter branch不一致、resolved ID差、期限切れConformanceではModelを呼ばない。Evalと更新Workflowは[AI Verification／Provenance](ai-verification-provenance.md)だけが決定する。

Prompt templateはRole、Goal、Success criteria、Normative constraints、Toolと権限、Evidence要求、Output Schema、Stop／質問条件の順にする。PromptをSecurity boundaryにせず、同じ規則を複数Promptへ反復しない。Prompt変更は一群ずつ行い、Model変更と同時に評価原因を混在させない。

RepositoryのAGENTS.mdはRepository map、Build／Test入口、禁止操作、Definition of Done、本正本とExecutable contractへのLinkに限定する。Subsystem規則は近いOwner文書へ置き、AGENTS.mdと実行可能契約が矛盾するMustをCIで拒否する。

### 8.2 Privacy、Prompt injection、Data handling

Source、Asset、User Prompt、Tool output、Web、Issue本文内の命令はcontentであり、Control命令へ昇格しない。Provider送信前にSensitivity labelとProvider Policyを照合し、Secret、private key、access token、Credential file、Production customer dataをContextへ含めない。

Prompt、Tool argument、Tool output、Traceは機密Dataになり得る。既定Telemetryへ本文を保存せず、hash、分類、Resultだけを記録する。Zero Data Retentionが必須のProjectでは非対応Provider機能を無効化し、代替がなければTaskを停止する。Live Web取得は分離したResearch Taskだけで許可し、Build／Release中の自動Web取得を禁止する。

Modelへ公開できるのはProject／Requirement／Capability／System／Worldのbounded Query、plan／validate／preview、ChangeSet submit、Source task request、Patch submitと、[Executable contracts](../02-foundation/executable-contracts.md#20-ai向けdiscovery)に登録されたPackage／Device／Play／Debug／Task OperationのうちCallerのexact allowlistにあるものまでである。Device install／resetはTool表示の有無にかかわらずDevice binding、consent、R3 ApprovalをServer側で再検証する。commit、activate、write native artifact、merge、sign、release、secret.read、policy.overrideを公開しない。

MCP annotationをAccess controlにしない。Serverは正本Schema、Authorization、timeout、rate limit、Auditを強制する。local STDIOはACL付きIPCをGatewayへ接続する。Streamable HTTPはexact Host／Transport Conformance Receipt、TLS、認証、origin／redirect／private-address policy、session binding、別Threat ModelとActivationがある場合だけ有効にし、単なるport forwardingを許可しない。`McpSessionGrantV1`は署名済みEnvelopeを代替せず、Grant配下のbounded read／QueryもR0 Envelopeを必要とする。GrantはClientとchannelの束縛だけを追加する。

MCP 2025-11-25の標準initializeでAuthorityとして検証できるClient情報はprotocol version、capabilities、`clientInfo`等に限られ、上流Provider、Model、local binary hashの暗号学的attestationは得られない。標準外部MCP接続のprovider／model表示は`unattested_optional_metadata`としてAudit UIにだけ出し、Authorization、Eval attribution、Source／Build Receiptへ使わない。`managed_source_edit | build_job`を許す経路は、実行前にMCP外の登録済みBrokerがHost session／Provider runtime／Model snapshot／Tool projection／期限／nonceを署名したfresh `ManagedHostSessionAttestationV1`をCaller Contextへ束縛し、実行後に同Context／Task／Authorization／attempt／Input closure／typed resultを署名した`HostExecutionAttestationV1`をSourceDelta／Build Receiptへ束縛する。Attestation不在時はManaged Contextを拒否し、別途standard routeのHost／Transport／Tool／proposal-only Authority Profile、Grant、Conformanceが全てcurrentな場合だけGatewayが新しい`standard_external_mcp` Caller Contextを発行する。事後resultを実行前Contextへ入れる因果循環、Host自己申告JSON、MCP annotation、Client名をAttestationに読み替えない。

### 8.3 Caller／Provider／Deployment／Model Profile

Caller互換性をHost brandだけで判定しない。正規判定は`execution_route.kind`をdiscriminatorにした次の3 branchだけである。`standard_external_mcp`は外部Host＋MCP Transport＋Tool＋proposal-only Authority＋MCP Grant＋Host/Transport Conformanceを必須にし、上流Provider／Deployment／Modelを必須`null`／unattestedとする。`managed_external_host`は外部Host＋MCP Transport＋Provider Runtime＋Model＋Tool＋Authority＋Host/Transport／Provider-Tool Conformance＋fresh Session Attestationを使う。`engine_provider_adapter`はfirst-party Engine Host＋Provider Runtime＋Provider Manifest＋Inference Deployment＋Model＋Tool＋Authority＋Provider-Tool Conformanceを使い、MCP Transport／Grantを必須`null`にする。cloud direct APIとfirst-party local inferenceはroute kindを増やさず、`engine_provider_adapter`配下の`InferenceDeploymentProfileV1.deployment_kind`で区別する。次のclosed型をMCDへ登録する。

```text
AiCallerContextPayloadV1
  caller_context_id, authenticated_subject_ref
  host_profile_binding: GovernedAiProfileBindingV1
  execution_route:
    {kind: standard_external_mcp,
     transport_profile_binding: GovernedAiProfileBindingV1,
     provider_runtime_profile_binding: null,
     provider_manifest_binding: null,
     inference_deployment_profile_binding: null,
     model_snapshot_profile_binding: null,
     managed_host_session_attestation_ref: null,
     managed_host_session_attestation_sha256: null,
     unattested_optional_metadata_ref: null | content ref,
     mcp_session_grant_ref: content ref,
     mcp_session_grant_sha256: lowercase hex 64,
     host_transport_conformance_receipt_ref: content ref,
     host_transport_conformance_receipt_sha256: lowercase hex 64,
     provider_tool_conformance_receipt_ref: null,
     provider_tool_conformance_receipt_sha256: null,
     maximum_authority: query | proposal}
    | {kind: managed_external_host,
       transport_profile_binding: GovernedAiProfileBindingV1,
       provider_runtime_profile_binding: GovernedAiProfileBindingV1,
       provider_manifest_binding: null,
       inference_deployment_profile_binding: null,
       model_snapshot_profile_binding: GovernedAiProfileBindingV1,
       managed_host_session_attestation_ref,
       managed_host_session_attestation_sha256,
       unattested_optional_metadata_ref: null,
       mcp_session_grant_ref: null,
       mcp_session_grant_sha256: null,
       host_transport_conformance_receipt_ref: content ref,
       host_transport_conformance_receipt_sha256: lowercase hex 64,
       provider_tool_conformance_receipt_ref: content ref,
       provider_tool_conformance_receipt_sha256: lowercase hex 64,
       maximum_authority: query | proposal | managed_source_edit | build_job}
    | {kind: engine_provider_adapter,
       transport_profile_binding: null,
       provider_runtime_profile_binding: GovernedAiProfileBindingV1,
       provider_manifest_binding: GovernedAiProfileBindingV1,
       inference_deployment_profile_binding: GovernedAiProfileBindingV1,
       model_snapshot_profile_binding: GovernedAiProfileBindingV1,
       managed_host_session_attestation_ref: null,
       managed_host_session_attestation_sha256: null,
       unattested_optional_metadata_ref: null,
       mcp_session_grant_ref: null,
       mcp_session_grant_sha256: null,
       host_transport_conformance_receipt_ref: null,
       host_transport_conformance_receipt_sha256: null,
       provider_tool_conformance_receipt_ref: content ref,
       provider_tool_conformance_receipt_sha256: lowercase hex 64,
       maximum_authority: query | proposal}
  tool_projection_ref, tool_projection_sha256
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
     exact_version, client_binary_sha256, os_identity_binding,
     credential_storage_profile_ref, process_child_policy_ref,
     filesystem_policy_ref, network_policy_ref}
    | {kind: hosted_service,
       service_surface_id, service_release_channel,
       tenant_workspace_ref, plan_tier_ref, admin_policy_ref,
       endpoint_identity_ref, oauth_or_oidc_subject_ref,
       token_audience, tls_identity_ref}
  supported_transport_profile_bindings[1..]: GovernedAiProfileBindingV1
  credential_owner
  protocol_versions[], conformance_receipt_refs[]
  support_state = supported | proposal_only | not_activated
  issued_at, expires_at, revoked_at

EngineAiHostSecurityProfileV1
  host_profile_id
  host_surface = editor_ai_orchestrator | headless_ai_orchestrator
  engine_build_ref, engine_build_sha256
  process_artifact_ref, process_artifact_sha256
  os_identity_binding, process_child_policy_ref
  filesystem_policy_ref, network_policy_ref
  supported_provider_runtime_profile_bindings[1..]: GovernedAiProfileBindingV1
  support_state = supported | not_activated
  issued_at, expires_at, revoked_at

McpTransportSecurityProfileV1
  transport_profile_id
  mcp_protocol_version = 2025-11-25
  transport_kind = local_stdio | streamable_http | secure_mcp_tunnel
  transport:
    {kind: local_stdio,
     server_binary_ref, server_binary_sha256,
     os_acl_policy_ref, ipc_endpoint_identity_ref,
     inherited_credential_policy = none,
     maximum_message_bytes, request_timeout_ms,
     rate_limit_per_minute, maximum_concurrent_requests,
     maximum_session_seconds,
     conformance_receipt_ref, conformance_receipt_sha256}
    | {kind: streamable_http,
       endpoint_origin, endpoint_identity_ref,
       tls_profile_ref,
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
       maximum_session_seconds,
       conformance_receipt_ref, conformance_receipt_sha256}
    | {kind: secure_mcp_tunnel,
       public_endpoint_origin, tunnel_endpoint_identity_ref,
       tunnel_client_artifact_ref, tunnel_client_artifact_sha256,
       local_service_binding_ref, tls_profile_ref,
       protected_resource_metadata_ref, protected_resource_metadata_sha256,
       authorization_server_metadata_ref, authorization_server_metadata_sha256,
       resource_indicator, token_audience,
       origin_policy_ref, redirect_policy_ref,
       private_address_policy = tunnel_local_service_only,
       session_binding_policy_ref, direct_inbound_allowed = false,
       maximum_message_bytes, request_timeout_ms,
       rate_limit_per_minute, maximum_concurrent_requests,
       maximum_session_seconds,
       conformance_receipt_ref, conformance_receipt_sha256}
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
  issued_at, expires_at, revoked_at

ProviderManifestV1
  provider_manifest_id
  provider_runtime_profile_binding: GovernedAiProfileBindingV1
  endpoint_identity_ref
  deployment_profile_binding: GovernedAiProfileBindingV1
  model_snapshot_profile_binding: GovernedAiProfileBindingV1
  role, inference_settings_sha256, tool_projection_ref
  context_limit, output_limit, cost_limit_ref, latency_limit_ref
  privacy_policy_ref, retention_policy_ref, region_ref?, encryption_profile_ref, logging_policy_ref
  eval_receipt_ref, conformance_receipt_ref
  fallback:
    {kind: disabled, fallback_deployment_profile_binding: null,
     fallback_max_risk: null}
    | {kind: explicit_profile,
       fallback_deployment_profile_binding: GovernedAiProfileBindingV1,
       fallback_max_risk}
  issued_at, expires_at, revoked_at

HostExecutionAttestationPayloadV1
  attestation_id
  caller_context_ref, caller_context_sha256
  host_session_ref, task_id, attempt_id
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
  tool_projection_ref, tool_projection_sha256
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
  schema_conformance_receipt_ref, tool_conformance_receipt_ref
  privacy_policy_ref, logging_policy_ref, retention_policy_ref
  output_limit_bytes
  preflight_resource_policy = reject_before_start
  runtime_exhaustion_policy = terminate_process_tree_and_fail
  fallback:
    {kind: disabled, fallback_deployment_profile_binding: null,
     fallback_requires_user_confirmation: false}
    | {kind: explicit_profile,
       fallback_deployment_profile_binding: GovernedAiProfileBindingV1,
       fallback_requires_user_confirmation: true}
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
  supported_schema_profile_refs[], supported_tool_projection_refs[]
  eval_receipt_refs[], issued_at, expires_at, revoked_at

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

`McpSessionGrantPayloadV1.expires_at`は`issued_at`から最大60分である。`grant_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:mcp-session-grant:sha256:<lowercase-hex>`として導出し、完成payloadを`role.mcp-session-grant-service`のsingleton-purpose Keyで署名する。signed recordのsubject、issuer subject／Role、issued_at、revocation snapshotをpayloadとexact一致させ、grantのoperation集合はQuery／Proposalだけに制限する。optional Fieldはkind／transportのtagged branchだけで必要条件を閉じ、local binaryをhosted serviceへ捏造またはその逆をしない。`ModelSnapshotProfileV1.model_identity`の固有Fieldは、`provider_model_id`なら`exact_provider_model_id`、`local_weights`ならcurrent `local_model_artifact_manifest_binding`だけを必須にして相互混在を拒否する。Local manifestは全weight shard、config、tokenizer、chat template、special-token map、license、provenanceとencoding branchをhash closureにし、native FP16／BF16へ架空quantizationを要求しない。identity正本はModel Snapshot＋Local manifestだけであり、`InferenceDeploymentProfileV1`へ複写しない。DeploymentとProvider Manifestの`model_snapshot_profile_binding`はschema／logical ID／record／revision／issuance Headをbyte-exact一致させる。`local_process_ipc`ではprocess artifact、`local_weights` Snapshot、認証済みOS IPCまたはloopback、local resource上限をすべて必須にする。Host display name、Provider名、Model family名は表示metadataであり、Transport、Tool Schema、Authorityを推測する入力にしない。

`AiCallerContextV1`はGatewayだけが`role.ai-gateway-context-publisher`／singleton purpose `ai_caller_context`で発行する短命signed contextである。`caller_context_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:ai-caller-context:sha256:<lowercase-hex>`として導出し、signed recordのsubject hash、issued_at=`created_at`、revocation snapshotをexact一致させる。`created_at < expires_at`、current Freshness Policyの`record_kind=ai_caller_context`はexact一件かつ`max_age_seconds <= 600`、`expires_at=min(created_at+max_age_seconds, Authorization／当該branchのGrantまたはSession Attestation／全non-null profile・Receiptのexpiry)`を必須にする。各non-null binding、Grant、Conformance Receipt、Authorization Envelopeを発行時とTool実行直前にread-backし、`expires_at <= evaluation_time`、current Head drift、revocation、別Project／subject／channelでは拒否する。

`host_profile_binding.profile_schema_id`は`standard_external_mcp | managed_external_host`で`ExternalClientSecurityProfileV1`、`engine_provider_adapter`で`EngineAiHostSecurityProfileV1`だけを許す。外部2 routeではHost Profileが列挙するcurrent MCP Transport bindingとContextのbindingをbyte-exact一致させる。Engine routeではEngine Hostが列挙するcurrent Provider Runtime bindingとContextを一致させ、MCP Transportを要求または捏造しない。`standard_external_mcp`はProvider Runtime、Provider Manifest、Inference Deployment、Model、Managed Session Attestation、Provider-Tool Conformanceを全てnullとし、MCP initialize由来のProvider／Model名はunattested metadataだけへ隔離する。`managed_external_host`だけはProvider／Model bindingと実行前`ManagedHostSessionAttestationV1` ref／hashを全non-null、同一Host session／tool projectionへ閉じる。`engine_provider_adapter`はProvider Manifestが指すProvider Runtime、Inference Deployment、Model Snapshot、Tool projectionをContextとbyte-exact一致させる。cloud direct APIとfirst-party local IPCは同じEngine routeのDeployment branchであり、MCP Transport Profileを流用しない。branch間Field流用、裸Context、caller自己署名を拒否する。

route別session bindingもclosedにする。`standard_external_mcp`はfresh `McpSessionGrantV1` ref／hashを両non-nullで必須、`engine_provider_adapter`は両方null、`managed_external_host`は両方nullかつ実行前`ManagedHostSessionAttestationV1`を専用Broker session bindingとして必須にする。effective operation集合は常に `route ceiling ∩ current AiAuthorityProfile.allowed_operation_refs[] ∩ signed TaskAuthorizationEnvelope.allowed_operations[] ∩ Server Policy`、standard MCPではさらに`∩ McpSessionGrant.payload.allowed_proposal_operation_refs[]`である。managed routeではSession Attestationの`allowed_task_kinds[]`／authority classも積集合へ加える。いずれかのmissing、tuple差、空でない超過、より強いcaller申告、Profileの`forbidden_authorities[]`欠落をTool公開前と実行直前に拒否する。

Attestation条件は因果順に評価する。query／proposal Authorityはpre／post集合を両方empty exact set、managed edit／build Authorityは`required_pre_execution_attestation_kinds=[managed_host_session]`、`required_post_execution_attestation_kinds=[host_execution]` exact setとする。Tool公開前・実行直前はpre集合だけを検査し、処理後にBrokerがtyped resultを得てからHost Execution Attestationを発行する。Staging受入れ、`GenerationReceiptV1`完成、`ManagedHostOutputAcceptanceReceiptV1`発行の各時点ではpre Attestationを再検証したうえでpost集合も必須にする。post Attestationを実行前Contextへ埋め込むこと、post欠落のresultをStagingへ受け入れること、R4判断やCommitをallowlistへ追加することを拒否する。

次のprofile payloadは必ず`GovernedAiProfileRecordV1`で署名し、payload単体や自己申告`support_state`をcurrentにしない。

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
| `ManagedHostSessionAttestationV1` | `managed_host_session_attestation` | `role.managed-host-session-attestor` | payload `issued_at` | allowed task kindだけ |
| `HostExecutionAttestationV1` | `host_execution_attestation` | `role.managed-host-execution-attestor` | payload `issued_at` | typed resultの完成branch |
| `ManagedHostOutputAcceptanceReceiptV1` | `managed_host_output_acceptance` | `role.managed-host-output-acceptor` | payload `accepted_at` | `accepted_to_staging`かつtyped result=`completed`だけ |
| `ModelImportQualificationReceiptV1` | `model_import_qualification` | `role.ai-model-import-qualifier` | payload `issued_at` | `pass`だけ |
| `LocalInferenceRunReceiptV1` | `local_inference_run` | `role.ai-local-inference-runner` | payload `issued_at` | result kind=`completed`だけ |
| `LocalInferenceRepeatabilityReceiptV1` | `local_inference_repeatability` | `role.ai-repeatability-runner` | payload `issued_at` | `byte_equal`だけ |
| `ModelLicenseAcceptanceDecisionV1` | `model_license_acceptance_decision` | `role.model-license-acceptance.r4` | payload `issued_at` | `approved`だけ |

Freshness-bearing recordはcurrent `AiExecutionFreshnessPolicyV1` bindingの該当`record_kind`が一件だけ存在し、`expires_at = min(issued_at + max_age_seconds, 全参照profile／Receiptのexpires_at)`を満たす。Acceptance Receiptでは`issued_at`を`accepted_at`と読む。policy missing／duplicate、caller指定max age、expiry延長再包装、current Head差、source expiry／revocationを拒否する。License Decisionは同式でなくpayloadの`issued_at <= evaluation_time < valid_until`とProject／license driftを継続評価する。

| Host profile | 許可Transportの扱い | 対応表示条件 |
|---|---|---|
| Editor | direct Provider API、local STDIO MCP、またはActivation済みStreamable HTTP | exact組合せのReceiptと製品内Policyが有効 |
| ChatGPT Chat／Work | custom MCP appは現行公式提供範囲のChatGPT webからremote Streamable HTTPへ接続する。private network／developer machineはSecure MCP Tunnelを使い、direct local STDIOは不可。desktopのChat／Work UIはcustom MCP appがweb-onlyである間`not_activated` | plan／workspace条件、admin publish／action control、remote Host／TransportまたはSecure MCP Tunnel Receiptが有効 |
| ChatGPT desktop app内Codex host | Codex modeとしてlocal STDIO MCPまたはStreamable HTTP。ChatGPT Chat／Workのapp接続とは別Host Profile | exact Codex host version／binary／Transport Receiptが有効 |
| Codex CLI／Codex IDE | local STDIO MCPまたはStreamable HTTP | exact client version／binary／Transport Receiptが有効 |
| Claude Desktop／Claude Code | Profileに列挙したlocal STDIO MCPまたはStreamable HTTP | exact client version／binary／Transport Receiptが有効 |
| Cursor | Profileに列挙したlocal STDIO MCPまたはStreamable HTTP | exact client version／binary／Transport Receiptが有効 |

この表は許容するprofile classであり、現時点の具体的なHost／version／binary／Transport組合せを`supported`と宣言しない。初期`ExternalClientSecurityProfileV1`、`EngineAiHostSecurityProfileV1`、Host Conformance Receipt、managed Host attestor、first-party local runtimeのcurrent materialized instanceは0件である。Phase 4のAI CoreはEngine build／process／OS policyとProvider tupleを束縛したEngine Host Profileを、Phase 5 `wp.product.external-agent`は少なくとも一つのexact External Host＋MCP Transport＋Tool Projection＋proposal-only Authority組合せをそれぞれmaterializeし、positive／negative Conformanceを通す。ChatGPT、Codex、Claude、Cursor等の名前だけでは完了しない。`managed_source_edit | build_job` routeは専用Future Work PackageとThreat Model、Broker、session／execution attestor、Engine Build Receipt closureが承認されるまで`not_activated`であり、MVPのstandard external MCP proposal laneへ混ぜない。first-party local inferenceもProduct PlanのFuture entryのままである。

Receipt不在、期限切れ、version／binary／Transport／Schema差では`supported`と表示しない。`proposal_only`を表示できるのは、失効したManaged Contextを降格した場合ではなく、別のcurrent `standard_external_mcp` Host／Transport／Tool／proposal-only Authority Profile、MCP Grant、read／proposal ConformanceからGatewayが新Caller Contextを発行した場合だけである。その標準経路も成立しなければ`not_activated`とする。Callerの最大Authorityは`query | proposal | managed_source_edit | build_job`のclosed setであり、Approval、Commit、Activation、Signing、ReleaseはどのHost、Provider、Modelにも付与しない。

`ExternalClientSecurityProfileV1.host_product_id`はsurface／modeを含むexact IDとし、ChatGPT Chat／Work、ChatGPT desktop app内Codex host、Codex CLI、Codex IDEを別Profileとして発行する。desktop applicationという同じ容器、同じaccount、同じvendorであることを根拠にTransport、設定、権限、Conformance Receiptを共有しない。ChatGPT Chat／Work用remote MCP appをCodex local MCPとして、またはCodex local MCP設定をChatGPT Chat／Work用appとして自動投影しない。

`McpTransportSecurityProfileV1`はtransport kindごとの全Fieldをrequiredにし、他branch Fieldをunknownとして拒否する。local STDIOはbinary hash、OS ACL、IPC identity、credential非継承を、Streamable HTTPはexact origin／TLS／MCP OAuth 2.1 Protected Resource Metadata／Authorization Server Metadata／resource indicator／token audience／redirect／private-address／session bindingを、Secure MCP Tunnelはpublic endpointとexact tunnel client artifact、local service binding、direct inbound禁止を検証する。OIDC subjectはOAuth 2.1 branchの補助identity policyであってOAuth authorizationを置換せず、mTLS-onlyはActivation済みprivate service branchに限定する。`approved_private_service`は別Threat Model、service ref／hash、Activation ref／hashがある場合だけ許し、名前がprivate、localhost、tunnelであることを根拠にしない。全branchでexact MCP protocol version、message／timeout／rate／concurrency／session上限を強制する。External Client Host Profileの`supported_transport_profile_bindings[]`は各current完成Governed MCP Transport Profile recordへ解決し、Profile／Transport／Conformanceのexpiry・revocation・hash差で接続前にfail closedにする。Engine Host ProfileはMCP Transportでなくcurrent Provider Runtime bindingを列挙し、Engine build／process artifact／OS／filesystem／network policy差でEngine routeをfail closedにする。

Managed HostのSource edit／Build出力はそれぞれclosed `ManagedSourceEditResultV1`／`ManagedBuildJobResultV1`として保存し、処理完了後の`HostExecutionAttestationV1` ref／hashを`ManagedHostOutputAcceptanceReceiptV1`のrequired Fieldにする。`attestation_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:host-execution-attestation:sha256:<lowercase-hex>`、session attestation IDは同様に`urn:mirakan:managed-host-session-attestation:sha256:<lowercase-hex>`で導出する。flat result objectの`result_id`は同Fieldを除く完成object JCS hashからSource edit=`urn:mirakan:managed-source-edit-result:sha256:<lowercase-hex>`、Build=`urn:mirakan:managed-build-job-result:sha256:<lowercase-hex>`、Acceptance payloadの`receipt_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:managed-host-output-acceptance-receipt:sha256:<lowercase-hex>`として導出する。post-execution payloadのCaller Context、Task Specification、Authorization Envelope、attempt、Input closure、result branchのkind／schema ID／ref／hashはAcceptance Receiptの同名入力とbyte-exactでなければならず、Broker署名とcurrent revocationを検証後にだけStagingへ受理する。completed Buildの`build_receipt_ref/hash`はCore ArchitectureのEngine-owned `PackageReceiptV1`完成wrapperで、operation ID=`operation.build.request_package`、purpose=`operation_receipt:operation.build.request_package`、Build Gateway signer、result=`succeeded`、同じTask Specification／Authorization／Project revision／Candidate／Target／Toolchain／artifact manifestを必須にする。Host／Broker AttestationはprovenanceでありBuild Receiptを代替しない。Receiptは`service.managed-host-output-acceptor`／singleton purpose `managed_host_output_acceptance`で署名する。standard external MCP、Engine Provider Adapter、proposal-only経路はManaged Host Session／Execution Attestationを両方持てず、空Attestation、事前署名result、kindとresult schema不一致、別attempt／別Input Attestationを拒否する。

### 8.4 Local inference境界

Local inferenceは`InferenceDeploymentProfileV1.deployment_kind=local_process_ipc`のclosed branchとしてcloud endpointと分離する。起動前にcurrent `model_snapshot_profile_binding`からSnapshot／Profile Record／issuance Headをread-backし、`model_identity.kind=local_weights`、current Local Manifest binding、全weight shard closureと選択encoding branch、config／tokenizer／chat template／special-token map、license／provenance、expiry／revocation、Eval／Schema／Tool conformanceがすべてcurrentであることを検証する。Deploymentの`process_artifact_ref/hash`はLoaderの`runtime_artifact_ref/hash`とImport Qualificationのprocess artifactにexact一致し、Import QualificationのLoader／Local Manifest bindingはDeploymentのLoader binding／Snapshot内Manifest bindingと一致し、License DecisionのManifest bindingも同じでなければならない。五辺の一つでも違う、Snapshotが`provider_model_id`、license／provenance欠落、Receipt失効なら`diagnostic.ai.model-snapshot-binding-mismatch`でprocess起動前に拒否する。

Local branchはprocess artifact ref／hash、current runtime／loader Profile binding、Import Qualification、process／GPU isolation、mutual-auth endpoint、RAM／VRAM／CPU／memory／GPU／disk／temp cache／shard数・総bytes／child process／batch／concurrent inference／wall-time／output上限を全て必須にする。`os_ipc`はnetwork deny-all、`authenticated_loopback`はexact loopback originだけを許して外部networkをdeny-allにし、両branch混在、wildcard bind、LAN bindを拒否する。Local manifestの`model_format_id`はLoaderの`supported_model_format_ids[]` exact一件に一致しなければならない。custom model code、unsafe serialization、archive path escape、runtime network fetch、built-in file／shell tool、MCP proxyを禁止し、すべてのTool実行をMiraikanai Gatewayへ戻す。llama.cppを候補Adapterにする場合も、公式serverでexperimentalかつuntrusted環境非推奨の`--tools`と`--ui-mcp-proxy`を無効のまま固定する（[公式server README](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)）。loaderがweights/config/tokenizer closure外のartifactを要求すれば拒否する。preflight不足は開始前拒否、実行中超過はprocess tree終了＋failed Receiptへ収束させ、System memoryへの無制限spill、GPU共有contextからの隔離省略、Schema非対応Toolの自然言語代替を禁止する。

License文字列の存在だけではActivationしない。`ModelLicenseAcceptanceDecisionV1`はexact license／manifest／Project revision／intended-use profile／distribution planに対し、game生成、commercial use、weight redistribution、output use、required noticeの各許可をR4 License主体が署名する。`approved_use_kinds[]`は`game_authoring | asset_generation | code_generation | evaluation | redistribution`のclosed subsetとする。Project用途が同集合に含まれ、redistribution／output policyがpackage／distribution planと一致し、Decision／Signer／Role／Key／validity／revocationがcurrentな場合だけlocal deploymentを有効化する。`decision_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:model-license-acceptance:sha256:<lowercase-hex>`として導出し、`role.model-license-acceptance.r4`／singleton purpose `model_license_acceptance_decision`、`subject_sha256=SHA-256(JCS(payload))`、signed issued-at／revocation equalityを必須にする。

Local inferenceは既定で`output_reproducibility.kind=not_claimed`とし、completed実行のexact response bytes hash、runtime／model／settings／seedを署名済み`LocalInferenceRunReceiptV1`へ保存するが、同一出力の再現性を主張しない。preflight／runtime failureはfailed branch、null response hash、1件以上のtyped Diagnosticで表す。`deterministic_claim`だけはruntime artifact、GPU profile、inference settings、seed、sampler、thread policy、replay input setを固定し、最低3件の重複しないfresh completed Run Receiptを列挙する。全Run Receiptの同入力bindingと`exact_response_bytes_sha256`が一致する場合だけRepeatabilityを`byte_equal`にする。未固定hardware／runtime、単一hash自己申告、回数だけの申告でdeterministicと表示しない。

fallbackは`disabled`またはcurrent exact `fallback_deployment_profile_binding`だけである。Localからcloudへ移る場合は送信data class、Provider、region、retention、費用を再Previewし、新しいCaller Context／Authorizationと明示User確認を必要とする。Network到達性、timeout、resource exhaustionを理由にcloudへ暗黙送信した場合は`diagnostic.ai.silent-cloud-fallback-forbidden`で失敗し、元Taskの状態とProjectを不変にする。

Gemma、Kimi、Qwen、DeepSeek、Grok、GLMその他のModel familyごとのEngine branchを作らない。任意のProvider／local runtime／Modelは同じProfileとConformance Receiptで扱う。full-tuple routeのReceiptがなければそのrouteは`not_activated`であり、`proposal_only`へ暗黙降格しない。`proposal_only`は、別のcurrent `standard_external_mcp` profile／Grant／ConformanceからGatewayが新Caller Contextを発行できる場合だけである。外部Conformance済みHostがlocal modelを使う経路と、Miraikanai自身がlocal runtimeを配布・運用するCapabilityを分ける。first-party local inferenceはFuture incubationであり、MVPまたは初心者First PlayableのCompletion Gateにしない。

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

    attestation_id, project_revision, engine_baseline_hash
    system_contract_ref, system_bundle_hash, implementation_kind
    capability_scope_hash
    system_qualification_receipt_hash
    gate_policy_hash, result, signer_identity, signature

implementation_kindは`gameplay_definition | native_game_module | project_shader_module | project_shader_technique | hybrid | target_specialized_set`のclosed setである。`project_shader_module`はS2／S3 Moduleだけ、`project_shader_technique`はS4／S5 Techniqueを含むSystem、`hybrid`はGameplayDefinition、Native、Project Shaderのうち二種類以上を一つのSystem Bundleへ結ぶ場合に使う。SystemTechnicalAttestationV1はEvidence bundleを再掲せず、exactly oneのSystemQualificationReceiptV1をhash参照する。Policy ServiceはReceiptのproject_revision、engine_baseline_hash、system_contract_ref、system_bundle_hash、implementation_kind、capability_scope_hash、gate_policy_hashがAttestation subjectと一致し、resultがpassで、署名用途がsystem_qualificationであり、失効していない場合だけ署名する。

関係は一方向である。

    Verification Runner
      -> SystemQualificationReceiptV1 [Evidence closure、権限なし]
      -> Policy Service validation
      -> SystemTechnicalAttestationV1 [Policy判断、Receipt hash参照]

SystemQualificationReceiptV1はAuthorization、Approval、Promotion、Activation権限を与えない。SystemTechnicalAttestationV1はReceipt内のTest、Performance、Provenance、Target artifact Fieldを複写せず、Receiptなしでは成立しない。

FeatureIntegrationAttestationV1は次を固定する。

    feature_id, requirement_ids[]
    constituent_system_attestation_hashes[]
    dependency_graph_hash, integration_test_root_hash
    replay_root_hash, save_replay_impact_hash
    behavior_budget_receipt_hashes[]
    result, signer_identity, signature

HumanGameplayApprovalV1は次を固定する。

    approval_id, approval_subject_kind
    approval_subject_hashes[]
    base_game_candidate_hash?
    result_game_candidate_hash
    covered_change_set_hash
    approved_requirement_ids[]
    approved_capability_scope_hash
    reviewed_replay_ids[], reviewed_target_profiles[]
    known_limitation_ids[]
    approver_identity, issued_at, expires_at, signature

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

    activation_id
    previous_candidate_hash?, activated_candidate_hash
    human_gameplay_approval_hashes[]
    approval_coverage_hash, attestation_closure_hash
    rollback_candidate_hash?, activated_at
    activation_service_identity, signature

previous_candidate_hashとrollback_candidate_hashは、先行Candidateが存在しない初回Activationだけ省略できる。省略を空hashやplaceholderで表現しない。current pointerが既に存在する場合、Activation Serviceは両Fieldを省略したReceiptの発行を拒否する。初回Activationの起動失敗はlast-known-goodを持たないため、current pointerを未設定へ戻し、該当Candidateを有効化しない。

これらのApproval／Activation構造とAuthorityは本文書が所有する。共通署名Record、Verification／Generation／Review／Promotion Evidence envelope、Receipt hash連結、保持は[AI Verification／Provenance](ai-verification-provenance.md)が所有する。

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
  path_or_module_scope_refs[]
  qualification_receipt_ref
  independence_policy_ref
  valid_from
  expires_at
  revoked_at: canonical UTC | null

CodeOwnerApprovalV1
  assignment_ref
  exact_diff_hash
  source_revision
  build_receipt_refs[]
  review_receipt_ref
  decision
  issued_at
```

`CodeOwnerAssignmentV1`は上記9 Fieldだけを持つclosed subject schemaであり、9 Fieldをすべてrequired、`path_or_module_scope_refs[]`を1件以上の重複なしunsigned byte順、unknown Fieldを拒否とする。`revoked_at`だけがnullableで、Field省略、空文字、sentinel時刻を`null`へ補正しない。このsubjectのcanonical hashを`MirakanSignedRecordV1.subject_sha256`へ束縛し、署名を含む完成Assignment Record hashを参照とrevocation判定に使う。

`CodeOwnerAssignmentV1.role_ref`は`CodeOwnerRoleRegistryV1.role_ref`のexact一件で必須であり、display name、前方一致、ScopeからのRole推測を拒否する。`path_or_module_scope_refs[]`の各refはRole entryの`allowed_scope_kinds[]`の一件に一致し、Native RoleへShader scope、Shader RoleへNative scope、independent reviewerへDiff approvalを与えない。`revoked_at`はField省略を許さないrequired nullableで、`null`だけが未失効を表し、canonical UTC時刻ならその時刻以後revokedである。`valid_from <= evaluation_time < expires_at`かつ`revoked_at=null`の場合だけactiveとし、未来開始、期限切れ、失効を別stateへ推測補正しない。

`CodeOwnerAssignmentV1`は認証済みProject role administratorの要求をApproval ServiceがRole、Qualification／Scope／independence registryと照合して署名した場合だけ有効である。Policy Serviceは署名済みAssignmentのsubject、exact `role_ref`、全Scope、Qualification Receiptのsubject／Role／Scope／freshness、independence policy、期間を検証し、信頼済みrevocation registryの署名済みlatest headをcurrent snapshotとして毎回read-backする。発行時snapshotからcurrent sequenceまでのchainが連続し、current snapshotの署名、issuer、sequence、freshnessが有効であることを必須にする。subject内の`revoked_at=null`でもcurrent snapshotがAssignment Recordまたはsubject identityをrevokedとした場合はSource Worker起動とApproval使用を拒否する。current snapshotのmissing／stale／invalid、sequence rollback／gap、missing／unknown Role、RoleとScope kindの不一致、QualificationのRole／Scope差、`revoked_at` non-nullもfail closedにする。AI、Source Worker、割当対象者は自己発行／自己revocationできず、Policy Serviceは主体選定を代行しない。

`CodeOwnerApprovalV1.decision`は`approved | rejected`のclosed enumであり、共通署名、signer identity、revocationは`MirakanSignedRecordV1` envelopeが所有する。Policy ServiceはAssignmentのsubject、Role、Scope、qualification、independence、期間、revocationと、Approvalのexact Diff、Source revision、全Build Receipt、独立Review Receiptを照合する。Source、Diff、Build input、Toolchain、Target、Assignmentのいずれかが変わればApprovalを失効させる。Code owner判断はG0–G7、Technical Attestation、Human Gameplay Approvalのいずれも代替しない。

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
- 未適格System、未承認Change、別Candidate hashを含むActivation。
- `operation.build.request_package`後にProject revisionまたはCandidate rootを差し替え、古いTaskを継続する試行。
- pair済みDeviceを同名の別Deviceまたは新generationへ交換し、古いinstall／launch／reset／remote Debug grantを再利用する試行。
- Candidate、Target、artifact hashのいずれかが異なるPackage Receiptによるinstall／reset。
- InstallのPackage Receipt／artifact、LaunchのInstall Receipt／artifact、SmokeのPackage／Install／Launch Receiptまたはfixtureについて、ref／hash／署名／payload contractを一原因ずつ差し替える試行。各後段を副作用前に拒否する。
- Operation Receipt mapping 14件すべてのpositive fixtureで、exact purpose、subject Operation ID、payload contract、完成Receipt alias、execution Authority、Signer Role、singleton-purpose Keyの同一行bindingと署名成功を確認する。
- 各positive fixtureをbaseに、別Operation purpose、unknown／generic purpose、別実行AuthorityのKeyをそれぞれ一原因だけ変更し、payloadとsubjectが他は同一でも署名検証を拒否する。`OperationReceiptEnvelopeV1`のOperation IDとpayload型不一致、async 11件の`task_id` missing、sync Control 3件の`control_invocation_id` missing、両ID present、Control対象Task IDのEnvelope `task_id`への流用も一原因ずつ拒否する。
- consentまたはR3 Approvalなしの`operation.device.install`／`operation.device.reset_data`と、install Approvalをlaunch／smoke／Debugへ権限継承させる試行。
- Evidence ref不在、別Session／revision、gap隠蔽、reproductionなしの偽`validated_cause`を`operation.debug.validate_finding`へ渡す試行。`diagnostic.debug.finding-evidence-invalid`で拒否する。
- 期限切れ／失効／binary hash不一致の`ExternalClientSecurityProfileV1`、Engine build／process／OS policy差の`EngineAiHostSecurityProfileV1`、またはinvalidな`ProviderManifestV1`、`InferenceDeploymentProfileV1`、`ModelSnapshotProfileV1`によるTool／推論呼出し。routeに対するHost schema差、External routeのMCP Transport欠落、Engine routeのnon-null MCP Transport／Grant、Deployment／Provider Manifestの`model_snapshot_profile_binding`差、issuance Head差、local deploymentから`provider_model_id` Snapshot参照、Snapshotのweight shard／encoding branch／license／provenance／Conformance欠落または失効を一原因ずつ拒否する。
- Caller ContextのFreshness binding欠落、TTL 600秒超過、source expiryを超えるContext、Profile Head更新後のcurrent実行を拒否し、同じ過去Receiptは履歴監査で`authentic_but_stale`として検証できることを確認する。
- 全routeのTask Attempt chainで、initialのcount非0、repair Reservationのprevious／sequence／count gap、別Task previous、reserved中の二重開始、stale Head、同一Headへの並行CAS、Host／Transport／Grant／Model／Provider／Attempt変更によるcount reset、上限超過、期限前abort、結果ReceiptとReservation差、unsigned latestを一原因ずつ拒否する。standard MCPではProvider／Model／完全responseを`StandardExternalProposalReceiptV1`へ記録またはGeneration Receiptへ捏造しない。
- managed routeでpre Session Attestation欠落の実行開始、実行前の偽post Execution Attestation、処理後post欠落resultのStaging受入れ、別Task／attempt／Input／typed resultを指すpost Attestationを一原因ずつ拒否する。
- Managed AcceptanceからTask SpecificationまたはAuthorization hashを落とす、Host AttestationだけでBuild成功を申告する、別Project／Candidate／Target／Toolchainの`PackageReceiptV1`を束縛する、failed typed resultをStagingへacceptする試行。
- Local Deployment、Loader、Import Qualification、Snapshot内Manifest、License Decisionのbindingまたはprocess artifactを一辺だけ差し替える試行。OS IPCでnetworkを許す、loopback branchをwildcard／LANへbindする、built-in file／shell toolまたはMCP proxyを有効化する試行。
- RepeatabilityでRun Receiptが3件未満、重複ref、入力／runtime／model／settings差、exact response bytes hash差、failed run混在を一原因ずつ拒否する。
- ChatGPT Chat／Workをdirect local STDIO Hostとして登録する試行、ChatGPT desktopのChat／Work UIをweb-only期間にcustom MCP app対応と表示する試行、ChatGPT Chat／Workとdesktop内Codex hostのProfile／Receiptを流用する試行、またはClaude Desktop／Claude Code／CursorをConformance Receiptなしで`supported`表示する試行。
- Local runtime timeout、RAM／VRAM不足、Tool Schema不一致を契機に、Preview、User確認、新Authorizationなしでcloudへ送る試行。送信byte 0と`diagnostic.ai.silent-cloud-fallback-forbidden`を確認する。
- Assignmentの`role_ref`欠落／unknown、Native RoleへのShader scope、Shader RoleへのNative scope、Qualification ReceiptのRole／Scope差、`revoked_at` Field省略、`revoked_at` non-null、unknown extra Field、期限切れ、current snapshotでAssignment Recordだけをrevoked、current snapshotでsubject identityだけをrevoked、current snapshotのmissing／stale／invalid、または別Diff／Source revisionの`CodeOwnerApprovalV1`を一原因ずつ注入する。各fixtureはSource Worker起動前またはPromotion前に拒否し、`revoked_at=null`でもcurrent snapshot失効を上書きしない。

## 13. 完了条件

- AIが変更できるTaskSpecificationと、変更不能な署名Authorizationが分離される。
- R0–R5、A0–A3、Task state、修復停止、Expiry、read-backがPolicy testで強制される。
- Game制作のTool、Filesystem、Worker、ContextからEngine write経路とEngine sourceが除かれる。
- API、MCP、CLI、EditorのProposalが同じGatewayとPolicyへ到達する。
- Package／Device／Play／Debug／Taskの14 Operationが同じ`OperationTaskV1` binding、Receipt、cancel規則へ到達する。
- Host／Transport／Provider runtime／Model snapshot／Tool projection／Authorityが別Profileで評価され、対応表示は有効なConformance Receiptに束縛される。
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
- [OpenAI: Developer mode and MCP apps in ChatGPT](https://help.openai.com/en/articles/12584461-developer-mode-and-mcp-apps-in-chatgpt)
- [OpenAI: Codex host MCP support](https://learn.chatgpt.com/docs/extend/mcp?surface=cli)
- [Anthropic: Desktop Extensions and local MCP servers](https://support.claude.com/en/articles/10949351-getting-started-with-local-mcp-servers-on-claude-desktop)
- [Anthropic: Connect Claude Code to tools via MCP](https://code.claude.com/docs/en/mcp)
- [Cursor: Model Context Protocol](https://cursor.com/docs/mcp)
- [Ollama: OpenAI compatibility](https://docs.ollama.com/api/openai-compatibility)
- [llama.cpp: OpenAI-compatible server and MCP proxy](https://github.com/ggml-org/llama.cpp/tree/master/tools/server)

外部資料は2026-07-23に検証した。MCPの標準Transport、各Host／runtimeが公開する接続方式、local endpointの性質を確認する根拠であり、Miraikanai固有のRisk、Operation、Approval、Sandbox、Conformance結果を外部製品へ委ねない。OpenAI互換endpointという表示だけではTool／Schema conformanceを意味しない。
