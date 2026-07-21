# Miraikanai Engine AI Security／Approval Contract

- 文書ID: mirakan.arch.ai-security-approval
- 状態: review
- 正本範囲: AI task authorization、Risk、Trust boundary、不変Engine、Sandbox、Credential、Provider／MCP／CLI security、Preview、人間承認、Activation、Promotion、拒否
- 非正本範囲: Eval、Evidence envelope、Provenance、Trace grading、Receipt保持。これらはAI Verification／Provenanceを参照する
- 依存: [Product Plan](../00-product/product-plan.md)、[AI Verification／Provenance](ai-verification-provenance.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Project state](../03-authoring/project-state.md)、[Native game module](../03-authoring/native-game-module.md)
- 外部根拠検証日: 2026-07-21

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
| Bounded Project Shader | Engine-owned entry／binding／pass／resource境界内でoffline compileするProject Shader |
| capability_unavailable | 公開Capabilityで実現不能であり、Game制作TaskからEngine変更へ進まず停止した状態 |

## 2. ActorとTrust boundary

| Actor | Trust | 許可する役割 | 禁止 |
|---|---|---|---|
| Human Author | 認証済み主体 | 要件、Gameplay approval、手動Proposal | Policy外Token作成、技術Gate代替 |
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
    required_gates[]
    required_approvals[]
    long_running_grant?
    signature_algorithm, signature_format, key_id, signature

OperationはID＋versionのexact allowlistとし、wildcardを禁止する。Networkはdeny_all、Dependencyはno_change、AI TaskのSecretはno_secret_accessを既定とする。Pathはread／write／create／deleteを別に許可し、Process tree、CPU、memory、wall time、child count、output sizeをhard limitにする。

通常TaskはR0を含め署名必須である。署名鍵作成前のBootstrapDiscoveryは通常State machine外に置き、local system情報読取り、Key生成、Public key registry初期化だけを許可する。Provider、Project読取り、Worker、任意Path、Network、変更を許可せず、初期化後に再実行できない。

署名RecordはMirakanSignedRecordV1を使用する。初期ProfileはECDSA P-256／SHA-256、RFC 8785 JCS、P1363固定64 byte、base64url without padding、low-S必須である。unknown field、duplicate key、invalid UTF-8、非有限値、高S、未知／失効／用途不一致／期限外Keyをfail closedで拒否する。秘密鍵は専用Service identityのnon-exportable Key storeへ置き、AI／Workerから分離する。

### 3.2 Task state machine

正規状態は次の14個だけである。

    Draft, ResolvingRequirements, AwaitingAuthorization, Ready
    Running, Validating, AwaitingApproval, Promoting
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
- Approvalは表示したDiff、Gate result、Artifact hashと完全一致するrevisionだけに有効である。
- Rejected、Failed、ExpiredのArtifactを昇格しない。
- CompletedはAuthoritative revision／commitのread-back照合後だけにする。

AwaitingUserInputはResolvingRequirements、Running、Validatingからだけ入る。Authorization前の回答はSpecification draftを更新できる。Authorization後の回答がGoal、Success criteria、Input、Risk、Operation、Path、Network、Dependency、Gate、Approvalへ影響する場合、元TaskをCancelledにし、新Taskと新Envelopeを作る。

修復はMCD RemediationV1があり、同じInput、Envelope内、permission／security／lock／approval／revision drift以外の場合だけ許可する。初回後の再提出は最大2回。同じblocking diagnostic＋location＋Stable ID集合が再発し、blocking数が減らない場合は即停止する。

Atomic commit、許可済みlong-running verification、Release transactionのcritical section開始後は、Cancel／Expiryで結果不明のまま終了表示しない。完了、rollback、read-backのいずれかへ収束させる。

## 4. Risk classとActivation

### 4.1 R0–R5

Riskは変更後の最大影響で決め、AI自己申告で下げない。

| Risk | 代表変更 | AI | 自動昇格 | 必須Approval |
|---|---|---|---|---|
| R0 | 読取、検索、説明、Report | 可 | 状態変更なし | 不要 |
| R1 | 文書、非実行Sample、個人Editor layout | 可 | protected branch外かつ全Gate時だけ可 | Owner不要、Gate必須 |
| R2 | GameSpec、World／Level、Scene、UI、Asset設定、GameplayDefinition、既存System設定 | 可 | 署名済み事前委任内の可逆Operationを非Release branchへ適用する場合だけ | Author 1名または同等Scopeの事前委任 |
| R3 | Project-defined System、bounded NativeGameModule、Project Shader、互換Schema | 可 | 不可 | G0–G7とPolicy Service署名。開発ProfileではCode owner＋全Gate |
| R4 | authoritative State owner／Save意味。Engine製品ProfileではEngine core／Extension／Security等 | Project artifact可。Engine sourceはGame制作不可 | 不可 | Project Attestation／Approval。EngineはDomain owner＋独立Reviewer |
| R5 | merge、tag、sign、Store upload、Production secret、公開Release | Proposalだけ | 禁止 | Human release owner＋分離Pipeline |

Test削除、Assertion緩和、Budget引上げ、Schema制約削除、Approval削除は対象実装と同じか一段高いRiskにする。R5 OperationをModel Tool catalogへ公開しない。

R2事前委任はOperation ID＋version、Path、最大件数／byte、branch、有効期限、rollbackを固定し、Save schema、Public API、Asset license、Dependency、Security、課金、公開配布を含めない。Promotion Serviceは実Diffを再分類し、Envelope超過なら新Authorizationを要求する。

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
    unresolved_blocking_question_count
    unresolved_high_requirement_count
    unsupported_capability_ids[]
    disposition

ready_to_stageにはBlocking／High未解決0、RequirementからSystem／Implementation／Test／Artifactまでの追跡100%、State owner重複0、stale Evidence 0が必要である。必須Requirementに未対応Capabilityがあればcapability_unavailableにする。

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
- GameSystemSpecV1、Project-defined System Contract。
- GameplayDefinition、World、Level、Scene、UI、Asset、Material、Animation、Audio設定。
- Project Test、Fixture、Benchmark、Replay、Save migration。
- BoundedNativeGameProfileV1に適合するNativeGameModule。
- BoundedProjectShaderProfileV1に適合するShader SourceとTarget別offline artifact。
- 生成Bindingの入力となるProject Contract。GeneratorとEngine-owned Schemaは変更不可。

実装方式は次の順で検討する。

    既存System／Capability composition
      -> GameplayDefinition
      -> Cook／index／layout最適化
      -> Bounded NativeGameModule
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

Project ShaderはEngine-owned entry、generated binding、許可pass／resourceへ接続する。任意Render pass、UAV、native GPU resource、descriptor heap、Compiler option、Engine private includeを追加できない。Targetごとに隔離Workerでoffline compile／validateし、Shipping RuntimeでSource生成、download、JITを行わない。新Domain／Backendが必要ならcapability_unavailableにする。

## 7. Sandbox、Filesystem、Network、Credential

### 7.1 Source Worker

Workspace Brokerが許可base commitからHost Stagingを作り、SourceBundleV1をguestへ転送する。WorkerへHost path、.git、branch／commit、main worktree、User home、Credential directory、他Projectを公開しない。返却SourceDeltaV1をuntrusted byte列として検査し、BrokerがStagingへ適用後に新Diffを計算する。

Bundle／Deltaはnormalized Repository-relative path、before／after hash、mode、size、content-addressed blobを持つ。symlink、junction、reparse point、hardlink、ADS、device、sparse file、submodule escape、case collision、reserved device nameを拒否する。任意archiveやguest filesystemをHostへ直接展開しない。

Windows A1／A2のPromotion可能BackendはHyperVIsolatedWorkerV1または同等のremote hardware-VM Workerとする。Generation 2、Secure Boot、immutable base image、Taskごとの一段differencing disk、no NIC、no host mount、no clipboard／device、Task完了後disk破棄を必須にする。Host／guest間はTask nonce、Envelope hash、Bundle hashを照合するbounded protocolだけを使う。

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
| local MCP | 外部ClientのQuery／Proposal | Commit、Activation、Provider設定 |
| conformance済みCLI Agent Host＋Worker | 隔離Source編集、Build、Test | main branch、Release、Credential公開 |
| Optional Plugin | Shortcut、Panel、Prompt／Skill UX | Engine必須機能、Security Policy |

PluginなしでもAPIとMCPで全公式操作を行えること。外部AI ClientはRepositoryと正規Project storageをread-onlyにし、Query／Proposalだけを使う。full-accessで直接起動したClientは公式Security boundary外であり、その変更はExternal patchとしてRisk再分類、全Gate、Approval、Promotionを通すまで正規入力にしない。

Credential ownerを分離する。

- 製品内AI: Provider AdapterだけがCredentialを持つ。
- 外部Client＋MCP: CredentialはClient／Userが持ち、Engineは受け取らない。
- Managed CLI: conformance済みHostまたは専用Brokerが持ち、File／Shell childへ渡さない。

Managed modeはExternalClientSecurityProfileにClientのexact version／hash、認証、Credential storage、Process／Tool child分離、Filesystem／Network sandbox、MCP version、期限、conformance resultを固定する。Profile不在、version差、期限切れ、plain EnvironmentへのCredential露出、raw socket利用ではMCP Proposal modeへ降格する。

### 8.1 Provider Manifest、Prompt、Repository guidance

Provider／Model名をEngine codeへhard-codeしない。ProviderManifestはProvider、endpoint、API／SDK exact version、resolved Model IDまたはsnapshot、Role、推論設定、Tool／Structured Output projection、Context／output／cost／latency上限、data retention／training use／region／encryption／logging Policy、合格Eval suite、fallback Model、fallback時の最大Riskを固定する。Manifestなし、resolved ID差、期限切れConformanceではModelを呼ばない。Evalと更新Workflowは[AI Verification／Provenance](ai-verification-provenance.md)だけが決定する。

Prompt templateはRole、Goal、Success criteria、Normative constraints、Toolと権限、Evidence要求、Output Schema、Stop／質問条件の順にする。PromptをSecurity boundaryにせず、同じ規則を複数Promptへ反復しない。Prompt変更は一群ずつ行い、Model変更と同時に評価原因を混在させない。

RepositoryのAGENTS.mdはRepository map、Build／Test入口、禁止操作、Definition of Done、本正本とExecutable contractへのLinkに限定する。Subsystem規則は近いOwner文書へ置き、AGENTS.mdと実行可能契約が矛盾するMustをCIで拒否する。

### 8.2 Privacy、Prompt injection、Data handling

Source、Asset、User Prompt、Tool output、Web、Issue本文内の命令はcontentであり、Control命令へ昇格しない。Provider送信前にSensitivity labelとProvider Policyを照合し、Secret、private key、access token、Credential file、Production customer dataをContextへ含めない。

Prompt、Tool argument、Tool output、Traceは機密Dataになり得る。既定Telemetryへ本文を保存せず、hash、分類、Resultだけを記録する。Zero Data Retentionが必須のProjectでは非対応Provider機能を無効化し、代替がなければTaskを停止する。Live Web取得は分離したResearch Taskだけで許可し、Build／Release中の自動Web取得を禁止する。

Modelへ公開できるのはProject／Requirement／Capability／System／Worldのbounded Query、plan／validate／preview、ChangeSet submit、Source task request、Patch submit、Task status、Approval requestまでである。commit、activate、write native artifact、merge、sign、release、secret.read、policy.overrideを公開しない。

MCP annotationをAccess controlにしない。Serverは正本Schema、Authorization、timeout、rate limit、Auditを強制する。MVP transportはlocal stdioだけで、ACL付きIPCをGatewayへ接続する。McpSessionGrantV1はClient binary hash、OS user、Project、read sensitivity、Proposal Operation、有効期限最大60分、nonceを固定するが、Task Authorization／Approval／Promotionを与えない。TCP、remote MCP、port forwardingは別Threat ModelとActivationまで無効にする。

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

Builder AI、Reviewer AI、Adversarial Test AIは技術承認できない。Engine-owned Validator／Runnerは個別Gateだけを判定し、Policy Serviceだけが全Gateとhash closureを照合してSystemTechnicalAttestationV1へ署名する。

### 9.3 Approval hierarchy

| Contract | 対象 | 失効条件／権限 |
|---|---|---|
| SystemTechnicalAttestationV1 | 一つのSystem Bundle、implementation kind、Target、Capability、Test、Budget | Source、Definition、Contract、Target、Budget、Baseline、Evidenceの変更で失効。Policy Serviceだけが署名 |
| FeatureIntegrationAttestationV1 | 複数Systemで成立するUser-visible Feature | 構成System／Graph／Replay／Save／Budget変更で失効。System適格化を代替しない |
| HumanGameplayApprovalV1 | system、feature、またはexact game candidate | result Candidate hash、ChangeSet、Requirement、Capability、Preview、Target、制限、期限へ限定 |
| GameCandidateManifestV1 | baseline、Project revision、全Attestation、Asset、Target artifact、whole-game Gate | 欠落、失敗、stale、Target mismatchなら候補化禁止 |
| GameActivationReceiptV1 | previous／new Candidate、Approval coverage、Attestation closure、Rollback | Activation Serviceだけが署名しpointerを原子的に切替 |

SystemTechnicalAttestationV1は次を固定する。

    attestation_id, project_revision, engine_baseline_hash
    system_contract_ref, system_bundle_hash, implementation_kind
    target_profile_hashes[], definition_package_hashes[]
    source_tree_hash?, native_artifact_hashes[]
    project_shader_artifact_hashes[]
    capability_scope_hash, test_evidence_root_hash
    performance_receipt_hashes[], provenance_root_hash
    gate_applicability_hash
    bounded_native_profile_hash?
    bounded_project_shader_profile_hashes[]
    gate_policy_hash, result, signer_identity, signature

implementation_kindはgameplay_definition、native_game_module、hybrid、target_specialized_setのclosed setである。Target別配列はTarget Profile ID順で件数を一致させ、該当しないoptionalを空hashで表現しない。

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
    rollback_candidate_hash

Target別Artifact配列はtarget_profile_hashesと同じ順序、同じ件数にする。Target共通byte列でも各slotへhashを明示し、暗黙fallbackを作らない。

GameActivationReceiptV1は次を固定する。

    activation_id
    previous_candidate_hash, activated_candidate_hash
    human_gameplay_approval_hashes[]
    approval_coverage_hash, attestation_closure_hash
    rollback_candidate_hash, activated_at
    activation_service_identity, signature

これらのApproval／Activation構造とAuthorityは本文書が所有する。共通署名Record、Verification／Generation／Review／Promotion Evidence envelope、Receipt hash連結、保持は[AI Verification／Provenance](ai-verification-provenance.md)が所有する。

初回制作はAIがGame全体をStagingで完成できる。System技術適格化とFeature統合検証の後、人間にはexact Game Candidate全体を一回提示できる。更新は変更影響Graphから最小System／Feature closureを求めるが、どのApprovalも最終result Candidate hashとRollback先へ結び付ける。

初心者へC++ Source reviewを要求しない。既定UIは、System／Featureの目的、読取／変更State、State owner、Capability、File／Network／OS非使用、Gateの成功／未実行、Replay／Preview、性能、制限、未対応Target、Fallback、Candidate hash、Rollbackを平易に示す。「AIが安全と判断した」等の根拠不明表示を禁止する。

面白さ、操作感、難易度、Art、Game意図は人間のGameplay approval対象であり、CompilerやAIで代替しない。

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
| Validator crash／Test infrastructure failure | 合格扱いせずinfrastructure_failure |
| Sandbox／hardware isolation unavailable | Source Workerを起動せず停止 |
| Baseline hash／signature不一致 | Build／Preview／Activation停止 |
| Engine private／OS／Vendor API要求 | Source Gate拒否。公開Capabilityへ再設計 |
| 公開Capabilityで意味同等に実現不能 | capability_unavailable |
| Sanitizer／fuzz／Replay／Budget failure | Artifact隔離、Promotionなし |
| Human rejection | Rejected終端。修正は新Task |
| Revision drift | 現Task停止、新ContextとEnvelope |
| Receipt／hash／Approval mismatch | Security event、Promotion拒否 |
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
- Bounded Shaderの任意pass／UAV／native resource／private include。
- malicious Build scriptからSigning key／Store tokenへの読取と持出し。
- Signing ServiceへのSource、Build script、shell、Manifest外File、別Artifact。
- Upload Serviceへの未署名Artifact、古いEvidence、別Application／Channel／Version、過剰Role。
- 未Activation OperationのTool公開または内部成功。
- 未適格System、未承認Change、別Candidate hashを含むActivation。

## 13. 完了条件

- AIが変更できるTaskSpecificationと、変更不能な署名Authorizationが分離される。
- R0–R5、A0–A3、Task state、修復停止、Expiry、read-backがPolicy testで強制される。
- Game制作のTool、Filesystem、Worker、ContextからEngine write経路とEngine sourceが除かれる。
- API、MCP、CLI、EditorのProposalが同じGatewayとPolicyへ到達する。
- Sandbox不能、Baseline mismatch、Capability不足、Credential分離不成立でfail closedになる。
- Project data、Bounded Native、Bounded Shaderの境界とG0–G7適用laneが機械検査される。
- Builder／Reviewer AI、Policy、Approval、Promotion、Activation、Release各Authorityが分離される。
- 初心者がSourceを読まず、意図、Capability、Evidence、Preview、制限、Rollbackを確認できる。
- System、Feature、Game Candidateのhash階層と変更影響失効が強制される。
- current Candidate、Project revision、Git commit、ReleaseをAIが直接作れない。
- Security fixtureが正規状態不変を確認する。

## 14. 一次根拠

- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [SLSA artifact verification](https://slsa.dev/spec/v1.2/verifying-artifacts)
- [RFC 8785 JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785)
- [RFC 7518 JSON Web Algorithms](https://www.rfc-editor.org/rfc/rfc7518)

外部資料はTrust分離、署名、Artifact verificationの根拠であり、Miraikanai固有のRisk、Operation、Approval、Sandbox既定を外部製品へ委ねない。
