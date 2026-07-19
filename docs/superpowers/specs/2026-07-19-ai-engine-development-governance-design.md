# Miraikanai Engine AI実装・保守ガバナンス規約

- 文書版: 1.5
- 作成日: 2026-07-19
- 調査基準日: 2026-07-20
- 対象: Game制作AI、GameplayDefinition／C++生成AI、Engine実装・保守AI、Editor、外部CLI／Desktop App、API Provider
- 状態: プロジェクト公式の規範設計レビュー版
- 上位文書: [AIネイティブ独自ゲームエンジン 設計計画書](./2026-07-18-ai-native-game-engine-authoring-design.md)
- Game実装規約: [Miraikanai Engine C++実行コード・構造化ゲームデータ規約](./2026-07-19-cpp-structured-game-data-design.md)
- Authoring規約: [Miraikanai Engine Authoring Model／Project State規約](./2026-07-19-authoring-model-project-state-design.md)
- Native Game規約: [Miraikanai Engine NativeGameModuleアーキテクチャ規約](./2026-07-19-native-game-module-architecture-design.md)
- C++言語・Modules規約: [Miraikanai Engine C++23・Named Modules・`import std`移行規約](./2026-07-20-cpp23-modules-import-std-transition-design.md)
- Asset規約: [Miraikanai Engine Asset Pipeline／Content Package規約](./2026-07-19-asset-pipeline-content-packaging-design.md)
- 機械可読契約: [Miraikanai Engine 実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)
- 検証規約: [Miraikanai Engine AI検証・評価・来歴規約](./2026-07-19-ai-verification-evaluation-provenance-design.md)

## 1. 結論

採用する方式は、**Hybrid High-Assurance AI Development**である。AIへ自由な内部操作権を与えず、次の三層を分離する。

1. 人間とAIが読む設計意図、要件、説明。
2. Engineが検証する機械可読契約、変更提案、Test、予算。
3. 人間または信頼済みPolicy Serviceだけが発行できる許可、承認、昇格、Release権限。

AIはGame制作とEngine開発の両方で実装主体になれる。ただし、AIが自分で権限を宣言し、自分の出力を検証済みとみなし、正規Project、main branch、署名鍵、配布先を直接変更することは禁止する。AIは隔離領域へGameSpec、GameplayDefinition、ChangeSet、C++、Shader、Test、文書、Patchを生成し、決定論的Validator、Compiler、Test、Budget gate、必要な人間Reviewを通過した成果物だけを信頼済みGatewayが昇格する。

この方式は次を同時に満たす。

- 初心者は自然言語だけでGame全体を作り始められる。
- 経験者はEditor、GameplayDefinition、C++を手動編集できる。
- AIは構造化編集またはC++を要件と計測から選べ、両者を明示的なCapability境界で併用できる。
- Codex、Claude等のCLI／Desktop AppはMCP経由で同じ契約を利用できる。
- 製品内の統合AIはProvider APIを使い、構造化出力、会話、監査、UXを一体管理できる。
- Engine本体の実装・保守AIは隔離WorktreeでSourceを変更できる。
- AI Provider、Model、Prompt、Toolの更新を評価なしでProductionへ自動適用しない。

## 2. 採用前監査の結論

### 2.1 採用する判断

| 判断 | 結論 | 理由 |
|---|---|---|
| 機械可読契約を正本にする | 採用 | proseだけでは型、権限、状態遷移、予算の欠落をCIで検出できない |
| Provider APIを製品内AIの主経路にする | 採用 | 構造化出力、会話状態、監査、失敗処理、初心者UXをEngine側で統制できる |
| MCPを外部AI接続の標準境界にする | 採用 | Codex、Claude、Desktop App、CLIを一つのEngine契約へ接続できる |
| Pluginを補助UXとして許可する | 採用 | Panel、Command、Prompt template等を追加できるが、必須機能や権限の正本にはしない |
| AIにGameplayDefinition／C++生成を許可する | 採用 | 構造化編集を第一選択とし、未提供Capability、新規Algorithm、計測済みhot pathをNativeGameModule（Project C++）、Platform統合をR4 Engine Adapterで補う |
| 選択的な形式モデルを使う | 採用 | 承認、Commit、非同期公開、Version切替等の小さな状態機械に有効 |
| ModelとToolを用途別に使い分ける | 採用 | 全Taskを最大Modelへ送る方式はCost、Latency、評価範囲を不必要に増やす |
| Stable-firstの技術更新 | 採用 | Previewの即時導入と「最新」のfloating指定は再現性を壊す |

### 2.2 そのままでは採用しない判断

| 当初案 | 問題 | 修正版 |
|---|---|---|
| 一つのJSON Schemaを全Providerへ直接渡す | Providerごとに対応Dialect／subsetが異なる | 正本からProvider別Schemaを生成し、Engineは常に正本で再検証する |
| MCPに危険Toolを出さなければ安全 | CLIはMCP外でもFileやShellを操作できる | OS sandbox、隔離Worktree、Network deny、差分Promotionを併用する |
| `AiTaskPacket`にRiskと権限を含める | AIがTask本文を書換えて自己昇格できる | AI可視の`TaskSpecification`と署名済み`TaskAuthorizationEnvelope`を分離する |
| 構造化実装使用率50%／80%で機械選択する | Game規模やSubsystem予算を無視した比率は根拠がない | 個別Behavior Budgetと実測P95／peak、決定論性、Platform制約で判定する |
| 形式モデルが通れば実装が正しい | ModelとC++実装の乖離を証明できない | Model Checkに加えて遷移Conformance Testを生成する |
| 最新Model／SDKを自動採用する | 挙動、Cost、Schema、保持Policyが無通知で変わり得る | exact Provider Manifestと評価後の明示昇格を必須にする |
| 修復を最大5回自動反復する | 同じ誤りの反復、Cost増加、破壊的変更の拡大を招く | Risk別上限と同一診断の即時停止を採用する |
| Job Object／restricted tokenだけで生成Buildを隔離する | Resource制限とACLはできるが、任意Native codeに対する独立Kernel境界にならない | Job Objectはguest内の二次制限とし、Promotion可能実行はhardware VM境界へ置く |
| Windows Sandbox CLIをSource Worker正本にする | Disposableだがprocess output取得がなく、固定Toolchain imageとhost共有なしのProtocolを所有できない | 手動診断に限り、A1／A2は`HyperVIsolatedWorkerV1`を使う |
| Experimental CreateProcessInSandbox APIを先行採用する | 公式にExperimentalで仕様変更可能 | Technology radarだけに置き、Stable化とconformance後に別ADRで再評価 |

## 3. 規範の意味と決定権

本文の**必須**、**禁止**、**推奨**、**任意**は基盤アーキテクチャ規約と同じ意味を持つ。外部Providerの公式推奨は外部APIの事実であり、本書の「Miraikanai公式規約」と区別する。

矛盾時の優先順位は次のとおりである。

1. 法令、Platform／Storeの強制要件、Security incident response。
2. 署名済み`TaskAuthorizationEnvelope`とRepositoryの機械可読Policy。
3. C++実行コード・構造化ゲームデータ規約、実行可能契約、Runtime規約、基盤規約、Platform規約。
4. 本書のWorkflow規約。
5. Task固有の人間指示。
6. `AGENTS.md`、Prompt、Skill、Provider向け説明。
7. AIの推測。

下位層は上位層の権限、禁止事項、Budget、承認条件を緩和できない。`AGENTS.md`とPromptはAIへの案内であり、Security boundaryまたは権限の正本ではない。

## 4. Scope

### 4.1 対象に含む

- 初回自然言語PromptからGame Brief、GameSpec、First Playableを作る。
- 会話による追加、修正、調整、説明、Undo。
- Scene、Rule、UI、Asset、Material、Shader、Animation、Audio設定の構造化編集。
- GameplayDefinition、NativeGameModule（Project C++）、Shader source、Testの生成と直接編集。
- Engine C++、Editor、Orchestrator、Build、Schema、Documentationの実装・保守。
- Codex／Claude等のAPI、CLI、Desktop App、MCP Clientとの接続。
- AI Model、Prompt、Tool schema、Context構成の評価と更新。

### 4.2 初期対象に含めない

- AIによるmain branchへの直接push、merge、tag、Release。
- AIによるSigning key、Store credential、Production secretの読取または使用。
- 出荷Game内AIによるNative code生成、動的Library読込、Engine binary差替え。
- Provider固有Agent frameworkをEngineの正規Project形式にすること。
- Providerの会話履歴だけをTask状態、監査、Undoの正本にすること。
- LLM judgeだけでSecurity、Correctness、性能の合否を決めること。

## 5. ActorとTrust boundary

| Actor | 信頼Level | 許可される役割 | 禁止事項 |
|---|---|---|---|
| Human Author | 認証済み主体 | 要件入力、承認、手動編集、Review | Policy外の権限Token作成 |
| Editor UX | 信頼済みClient | Proposal表示、Diff、Approval要求 | Validatorを迂回したProject書込 |
| Policy Service | 信頼済みAuthority | Risk分類、Policy解決、Authorization署名 | Artifact生成、自己Approval、Promotion |
| Approval Service | 信頼済みAuthority | 人間認証、事前委任／Review Receipt署名 | AI判断だけでのApproval、Artifact変更 |
| AI Orchestrator | 制限付きService | Task分解、Provider呼出、Tool調停、Receipt作成 | 自己承認、Secret保持、main昇格 |
| Provider Adapter | Secret分離Service | Provider credential使用、Request送信、response metadata採取 | Project書込、Approval、CredentialのContext／Log出力 |
| Model Provider | 非信頼外部処理系 | 構造化提案と説明の生成 | 権限決定、最終検証、正規状態書込 |
| External AI Client | 非信頼Client | MCP query、Proposal提出 | Commit／merge／sign／release |
| Source Worker | 隔離された非信頼実行系 | Worktree内のFile編集、Build、Test | 許可外Path／Network／credential |
| C++ Command Gateway | 信頼済みAuthority | Authoring変更の検証、Transaction確定、Project revision発行 | 未承認または不正Proposalの適用 |
| Source Promotion Service | 信頼済みAuthority | Patch検証、承認照合、Git commit／Branch昇格 | AI自己申告だけでの昇格 |
| Release Coordinator | 信頼済みControl plane | R5証拠照合、分離Service起動、Receipt連結 | Source実行、Signing key／Store credential保持 |
| Release Build Worker | 隔離された非信頼実行系 | clean Build、package inspection、unsigned artifact生成 | Signing key／Store credential、正規Repository write |
| Platform Signing Service | 最高権限Service | 承認済みunsigned artifactの固定形式検査、Sign、署名検証 | Project source／Build script／compiler実行、Store upload |
| Store Upload Service | 最高権限Service | 承認済みsigned artifactのStore／配布先upload | Source／Build script／Signing key保持 |

AIが生成したText、Tool argument、Patch、Test結果要約、Risk自己申告はすべて非信頼Inputとして扱う。Validator、Compiler、Test runnerの終了CodeとArtifact hashは、信頼済みRunnerが採取した値だけを採用する。

Threat modelは「Modelが通常は従う」ことを前提にしない。次を攻撃者能力としてin-scopeにする。

- Model／External Clientが任意のmalformed Tool argument、Patch、Source、GameplayDefinition、Process commandを提出する。
- Source、Asset、Web、Issue、Prompt内の命令がControl命令を装う。
- Source Worker内の生成CodeとBuild toolが任意Code executionを得て、Path escape、child process、Network、Secret探索、resource exhaustionを試みる。
- Network endpoint、DNS、Redirect、download Artifact、Dependency metadataが悪意または侵害済みである。
- Approval／Receipt／Envelopeのstale、replay、差替え、並行Project revision競合が起きる。
- Provider refusal、partial output、Schema drift、Client update、Runner crashが任意時点で起きる。

OS kernel／hypervisor、TPM、信頼済みService binary、Policy／Approval／Release秘密鍵、組織Identity Providerの侵害後も安全とは保証しない。これらの兆候は通常Task failureではなくSecurity incidentとして全未完了Envelope失効、Key revocation、Artifact隔離、clean environment再構築を行う。Repository ownerまたはAdministratorが意図的にProtectionを解除する場合も同じTrust boundary外である。各Activation前に、このActor、Asset、Entry point、Attack、Mitigation、Residual riskをRequirement IDへ対応付けたversioned Threat ModelをReviewする。

## 6. Taskの正規構造

### 6.1 `TaskSpecification`

`TaskSpecification`はAIへ提示できる作業内容である。AIまたは人間が提案できるが、権限を与えない。

| Field | 型 | 必須規則 |
|---|---|---|
| `task_id` | `MiraId` | Gateway発行、変更不可 |
| `spec_revision` | uint32 | 1開始。`ResolvingRequirements`中だけ単調増加し、Envelope発行後は固定 |
| `task_kind` | enum | `game_authoring`、`project_source`、`engine_source`、`schema`、`docs`、`research`、`verification`、`release` |
| `goal` | UTF-8 string | 一つの検証可能な結果 |
| `success_criteria` | Requirement ID array | 1件以上、自然文だけは禁止 |
| `non_goals` | string array | Scope外を明示。空配列可 |
| `input_revisions` | `{artifact_id, revision, sha256}` array | 使用する正規Inputを完全固定 |
| `target_profiles` | Profile ID array | 最低1件 |
| `cxx_frontend_profile_id` | nullable `CxxFrontendProfileId` | C++を生成、変更、CompileするTaskでは必須。非C++ Taskだけnull |
| `build_driver_profiles` | `{target_profile_id, driver_profile_id}` array | C++ Compile／Package Taskでは全`target_profiles`をexactly一回覆い、基盤規約のclosed `BuildDriverProfileV1`と一致。非Build Taskは空配列 |
| `cpp_dependency_sets` | `{component_id, dependency_set_sha256}` array | C++ Source Taskではcomponent ID昇順で全対象を固定。非C++ Taskは空配列 |
| `requested_outputs` | Artifact kind array | 生成物を列挙 |
| `open_questions` | Question ID array | Blockingが残る間は実装開始不可 |
| `context_pack_id` | MiraId | immutable ContextPackを参照 |
| `context_pack_sha256` | lowercase hex | 実際にProviderへ渡すManifestを固定 |

### 6.2 `TaskAuthorizationEnvelope`

`TaskAuthorizationEnvelope`は信頼済みPolicy Serviceだけが生成し、AIが変更できない。通常TaskはR0を含めてすべて署名を必須にし、未署名で許可する移行Phaseを設けない。署名鍵を作る前の`BootstrapDiscovery`だけは通常Task state machineの外に置き、Buildへ固定したlocal system情報の読取り、Key生成、Public key registry初期化だけを許可する。Provider、External client、Project読取り、Source Worker、任意Path、Network、変更操作を許可せず、初期化完了Marker後は再実行できない。

初期署名Profileを`MiraSignedRecordV1`として次に固定する。

- AlgorithmはECDSA P-256 with SHA-256、識別子は`ecdsa_p256_sha256`。
- `signature`以外の全FieldをRFC 8785 JCSでcanonicalizeし、SHA-256を1回計算して署名する。`signature_algorithm`、`signature_format`、`key_id`も署名対象に含める。
- Signature表現はP-256の`r`と`s`を各32-byte unsigned big-endianへ左0埋めし、`r || s`の64 byteとしたIEEE P1363／JOSE形式。識別子は`p1363_fixed_64`、転送表現はbase64url without paddingとする。
- ECDSA malleabilityを除くため、SignerはP-256 group order `n`に対して`s <= n / 2`となるlow-Sへ正規化し、Verifierはhigh-Sを拒否する。
- Windows初期実装はCNGのECDSA P-256とSHA-256を使う。Native表現が異なるPlatform adapterは必ず`p1363_fixed_64`へ変換し、RFC 7518のES256 Test vectorとcross-platform round-tripで検査する。
- Policy Serviceの秘密鍵はCNG Key Storage Providerへ永続化し、Export policyを許可しない。Key ACLは専用Policy Service identityだけに許可し、AI Orchestrator／Workerを別の低権限identityで実行する。TPM利用可能環境はMicrosoft Platform Crypto ProviderをProduction既定、開発機はMicrosoft Software Key Storage Providerを許可する。
- Public key wire表現はSEC 1 uncompressed form `0x04 || X32 || Y32`の65 byteをbase64url without paddingにする。`key_id`はその65 byteのSHA-256を`sha256:<64 lowercase hex>`で表す。Verifierは座標範囲、曲線上、無限遠でないことを検査する。
- 信頼済みPublic key registryは`key_id`、public key、issuer、用途、`not_before`、`not_after`、revocation状態を持つ。失効、未知、用途不一致、期間外のKeyはfail closedで拒否する。
- 署名対象RecordはRFC 8259 UTF-8 JSON、unknown field禁止、duplicate-aware parse、有限数値だけに限定する。重複Key、invalid UTF-8、unpaired surrogate、Schema外FieldをJCS処理前に拒否し、Parserごとの差を署名検証結果へ持ち込まない。

このProfileは量子耐性を保証しない。Post-quantum署名への移行は`signature_algorithm`追加、dual-sign期間、既存Receipt検証方針を伴う別ADRとし、Windows Insider PreviewだけのAlgorithmをProduction baselineへ採用しない。

| Field | 型 | 必須規則 |
|---|---|---|
| `envelope_version` | uint32 | 初期値1 |
| `task_id` | MiraId | `TaskSpecification.task_id`と一致 |
| `spec_sha256` | lowercase hex | Task本文のCanonical hash |
| `issued_at` | UTC timestamp | Policy Serviceが発行した時刻 |
| `not_before` | UTC timestamp | これより前は無効 |
| `risk_class` | enum | R0からR5 |
| `contract_set_hash` | lowercase hex | 使用する`/schemas/contract.lock.json`を固定 |
| `policy_set_hash` | lowercase hex | 解決済みPolicy集合を固定 |
| `resolved_profile_hashes` | `{profile_id, sha256}` array | Target／Quality／AI／Runner Profileを固定 |
| `tool_catalog_hash` | lowercase hex | 公開Operation ID＋version＋Schemaを固定 |
| `allowed_operations` | `{operation_id, version}` array | exact allowlist。wildcard禁止 |
| `path_grants` | `PathGrantV1` array | read／write／create／deleteを個別許可 |
| `network_policy` | `NetworkPolicyV1` | 既定`deny_all` |
| `dependency_policy` | enum | `no_change`、`existing_lock_only`、`proposal_only` |
| `secret_policy` | enum | AI Taskは常に`no_secret_access` |
| `resource_limits` | `ResourceLimitsV1` | Process treeと全出力のhard limit |
| `required_gates` | `{gate_id, version}` array | Risk classの最低Gateを減らせない |
| `required_approvals` | `ApprovalRequirementV1` array | Role、人数、identity分離を明示 |
| `long_running_grant` | nullable `LongRunningGrantV1` | 既定null。R3／R4の長時間検証またはR5 Releaseだけに許可 |
| `expires_at` | UTC timestamp | 過期限は開始も再開も不可 |
| `nonce` | `bytes_base64url` | CSPRNGで生成した正確に16 byte、paddingなし。replay防止 |
| `issuer` | Service ID | 許可済みIssuer |
| `signature_algorithm` | enum | 初期値`ecdsa_p256_sha256` |
| `signature_format` | enum | 初期値`p1363_fixed_64` |
| `key_id` | Key ID | 信頼済みPublic key registryを参照 |
| `signature` | base64url string | `signature`を除くEnvelope全Fieldの署名 |

`TaskSpecification`の内容、Prompt、MCP argument、AI出力からEnvelopeを生成してはならない。人間の明示選択とRepository PolicyからPolicy Serviceが決定する。

`not_before <= issued_at <= expires_at`を必須とし、Clockは信頼済みOS time sourceを使う。最大有効期間はR0／R1が8時間、R2が4時間、R3が2時間、R4が1時間、R5が15分とする。開始時、再開時、各副作用Tool呼出し直前に再検証し、期限切れ後は新しい副作用を開始しない。再承認は新しい`nonce`とEnvelopeを発行し、旧Approvalを自動継承しない。

通常Operationのhard timeoutは開始時点の`expires_at - now`以下にする。例外はModelへ公開しない次の一回限りOperationだけである。`LongRunningGrantV1`はoperation ID、exact input artifact root、GateまたはDestination、Runner／Service profile hash集合、Idempotency key、`start_before`、`complete_by`、resource limits、Cancellation／read-back policyを固定し、全FieldをEnvelope署名対象にする。`start_before`は`expires_at`以下とする。

| Operation | Risk／上限 | 追加制約 |
|---|---|---|
| `verification.long_run.v1` | R3／R4、`complete_by <= issued_at + 24 hours` | 既Build Artifactのfault／soak／device検証だけ。Source編集、Build、Secret、Promotion、raw Networkなし。Deviceはexact profileのRunner-owned Brokerだけ。OutputはVerification Receiptとbounded diagnosticだけ |
| `release.transaction.execute.v1` | R5、`complete_by <= issued_at + 6 hours` | 承認済みunsigned root、Target／Channel／Application／Version、Signing profile、Store listing revision、最大転送byteを追加固定 |

担当Authorityは`start_before`より前にEnvelope nonceとIdempotency keyを永続Ledgerで原子的に消費し、同じOperationだけを継続できる。開始後にEnvelopeまたはReview Receiptが期限切れになっても新しいInput、Destination、Process種別、再試行を追加せず、`complete_by`まで完了、取消、read-backのいずれかへ収束させる。失敗または期限超過後のresume／retryは現在状態を読戻した上で、新しいTask、必要Approval、Envelope、Idempotency keyを必要とする。Releaseではremote submission IDと既送信byte範囲も読戻す。Store側のprocessing／review／rollout待ちはTransactionへ含めず、Credentialを持たないR0 query Taskで追跡する。

`PathGrantV1`は`root`、`recursive`、`operations[]`、`max_file_bytes`、`allowed_extensions[]`を持つ。`root`はRepository相対のUnicode NFC／`/`区切りで、空、absolute、drive、UNC、`..`、wildcardを禁止する。Windows比較はinvariant case-fold後、最終判定は開いたHandleから得た解決先で行う。新規Fileは既存の最長Parentを同じ方法で検査する。

`NetworkPolicyV1`は`deny_all`または`allowlist`である。初期`allowlist` entryは`https`、ASCII lowercase exact host、port 443、HTTP method、path prefix、query policy、request／response byte上限、最大Redirect 3を持ち、wildcard host、IP literal、userinfo、別port、fragmentを禁止する。Path prefixはASCII absolute path、`/`区切り、percent escapeなし、dot segmentなしで登録する。BrokerはRequest targetを一つのURI parserで一度だけparseし、invalid percent escape、raw control／non-ASCII、backslash、encoded slash／backslash、decoded dot segmentを拒否する。Unreserved percent escapeをdecodeし、RFC 3986 dot-segment除去後のsegment境界でprefix比較する。Queryは既定`none`、必要時だけASCII exact key allowlistと値byte上限を持ち、同名Key重複を拒否する。Broker所有ProxyがDNS解決後と接続直前にloopback、link-local、private、multicast、reserved addressを拒否し、Redirect先を新規Requestとして全条件で再検証する。Source Workerへraw socketと独自DNSを許可しない。

`ResourceLimitsV1`は`wall_time_ms`、`cpu_time_ms`、`peak_commit_bytes`、`max_processes`、`max_open_handles`、`max_files_created`、`max_single_file_bytes`、`max_total_write_bytes`、`max_stdout_bytes`、`max_stderr_bytes`をすべて正整数で持つ。Process job／container単位で子孫へ強制し、超過時はProcess treeを終了して`resource_limit_exceeded`とする。

`nonce`はTask実行単位で一意とし、Policy Serviceの永続Nonce ledgerへ`task_id`、`spec_sha256`、有効期限、状態を記録する。同じEnvelopeから二つ目のWorkerを開始せず、Service再起動後も期限までは再利用を拒否する。Task内の個別Operation再試行はEnvelope nonceを再利用せず、OperationごとのIdempotency keyとAttempt IDで管理する。

Public key registry、Nonce ledger、Bootstrap完了MarkerはProject／Repository外のPolicy Service stateへ置き、AI OrchestratorとSource Workerにwrite ACLを与えない。Registry更新は既存の信頼済みKeyとApproval Serviceの両方で承認し、Key喪失時のRecoveryはinteractive owner認証、全未完了Envelope失効、新Root hash表示、Audit exportを必須にする。古いPublic keyは過去Receipt検証用に保持し、秘密鍵を復元または再利用しない。

### 6.3 `ContextPack`

ContextPackはProviderへ渡す情報のManifestであり、各Itemを次のChannelへ分離する。

| Channel | 内容 | 扱い |
|---|---|---|
| `control` | Task goal、許可の説明、出力形式 | Orchestratorだけが構成。Contentから変更不可 |
| `normative` | MUST／禁止／Budget／State machine | 原文とhashを保持。要約だけへの置換禁止 |
| `evidence` | Source、Test、公式資料、Benchmark | 引用元、revision、取得日を保持 |
| `content` | Asset metadata、User text、Source comment | Prompt injectionを含み得る非信頼Data |

各Itemは`resource_id`、`channel`、`uri`、`revision`、`sha256`、`requirement_ids`、`inclusion_reason`、`sensitivity`、`byte_count`を持つ。Providerへ送信した実際のItem集合と順序をContextPack hashへ含める。

Blocking／High要件は原文を`normative`へ含める。Context windowに収まらない場合は古い要件を黙って要約せず、Taskを分割するか、Schema付きResource toolで必要部分を取得させる。最終GatewayはContextに関係なく全正規要件で再検証するため、ContextPackはSecurity boundaryではない。

## 7. Task state machine

正規状態集合は次の14状態だけとする。

```text
Main nonterminal:
  Draft, ResolvingRequirements, AwaitingAuthorization, Ready,
  Running, Validating, AwaitingApproval, Promoting

Paused nonterminal:
  AwaitingUserInput

Terminal:
  Completed, Cancelled, Expired, Failed, Rejected
```

`Promoting`以外の非終端状態は、CleanupとAuthoritative state不変の読戻し後に`Cancelled`、`Expired`、`Failed`へ遷移できる。`Promoting`は下表の専用遷移だけを許可する。消費済み`verification.long_run.v1`で`Validating`中の場合だけ、Envelope expiry後も`complete_by`まで同じRunner process treeを監督し、新しいProcess／Inputを追加しない。`AwaitingUserInput`は`ResolvingRequirements`、`Running`、`Validating`からだけ入れる。Authorization発行前の回答は同じTaskのSpecification draftをversion更新して要件解決を続けられる。Authorization発行後は、回答が既存Authorization範囲内なら直前状態へ戻り、Authorizationへ影響するなら元Taskを`Cancelled`へ終端させて新Taskを作る。`Rejected`はApproval結果の終端状態であり、自動修復へ戻さない。

| From | Event | Guard | To |
|---|---|---|---|
| Draft | `SubmitSpecification` | Schema valid | ResolvingRequirements |
| ResolvingRequirements | `NeedUserInput` | Blocking questionあり | AwaitingUserInput |
| ResolvingRequirements | `RequirementsResolved` | Blocking question 0 | AwaitingAuthorization |
| AwaitingAuthorization | `EnvelopeIssued` | signature、hash、expiry valid | Ready |
| Ready | `Start` | Input revision一致、Runner確保 | Running |
| Running | `NeedUserInput` | active副作用なし、Cleanup／Artifact hash保存済み | AwaitingUserInput |
| Running | `ProposalProduced` | output size内 | Validating |
| Validating | `NeedUserInput` | active副作用なし、診断とArtifact hash保存済み | AwaitingUserInput |
| AwaitingUserInput | `PreAuthorizationAnswerAccepted` | `resume_state=ResolvingRequirements`、Specification draft revisionとContextPackを更新 | ResolvingRequirements |
| AwaitingUserInput | `AuthorizedAnswerAccepted` | `resume_state`がRunning／Validating、Authorization影響なし、Input revision一致、Envelope有効 | `{resume_state}` |
| AwaitingUserInput | `AuthorizationAffectingAnswer` | Envelope発行済み、`superseded_by_task_id`を記録、新Task作成済み | Cancelled |
| Validating | `AllAutomaticGatesPassed` | Gate receipt一致、Approval必要 | AwaitingApproval |
| Validating | `AllAutomaticGatesPassed` | Gate receipt一致、Approval不要、昇格Artifactあり | Promoting |
| Validating | `AllAutomaticGatesPassed` | R0または昇格Artifactなし | Completed |
| Validating | `RepairableFailure` | retry policy内 | Running |
| AwaitingApproval | `Approve` | 必須Reviewerとrevision一致 | Promoting |
| AwaitingApproval | `RejectOrChangesRequested` | signed Review Receipt一致 | Rejected |
| Promoting | `CancelRequested` | critical section開始前かつAuthoritative state不変を読戻し確認 | Cancelled |
| Promoting | `EnvelopeExpired` | critical section開始前かつAuthoritative state不変を読戻し確認 | Expired |
| Promoting | `AtomicPromotionSucceeded` | post-validate一致 | Completed |
| Promoting | `AtomicPromotionFailedBeforeCommit` | Authoritative state不変を読戻し確認 | Failed |
| Promoting | `AtomicPromotionRolledBack` | before hashへ復旧し読戻し確認 | Failed |

次を不変条件とする。

- Authorization前にSource Workerまたは変更Toolを起動しない。
- Envelopeが許可しないOperation、Path、Network、Dependencyを実行しない。
- Input revisionが変化したTaskを継続せず、再作成する。
- Approvalは表示したDiff、Gate receipt、Artifact hashと完全一致するrevisionだけに有効とする。
- `Rejected`、`Failed`、`Expired`のArtifactを正規状態へ昇格しない。
- `Completed`は、R0／昇格ArtifactなしならResult hashの確定後、状態変更TaskならAuthoritative revisionまたはSource commitの読戻し照合後だけにする。

Atomic commit、有効な`verification.long_run.v1`、または有効な`ReleaseTransactionV1`のcritical section開始後のCancel／ExpiryはOperationを未記録のまま中断せず、`cancel_requested_at`または`expired_during_operation`をAuditへ記録する。担当Authorityは完了、rollback／cancel、read-backのいずれかへ必ず収束させ、成功なら次の合法状態、commit前失敗またはrollback成功なら`Failed`とする。結果未確定のまま`Cancelled`／`Expired`へ見せない。`long_running_grant`のない副作用Operationのhard timeoutは開始時のEnvelope残時間以下とし、Source Worker等の非信頼Processはtimeout時にProcess treeを終了してCleanup後に`Expired`とする。

`AwaitingUserInput`へ入る時は`resume_state`と現在Artifact hashを保存し、副作用を停止する。`resume_state=ResolvingRequirements`ではEnvelopeがまだ存在しないため、回答を同じTask IDの新しいSpecification draft revisionとContextPackへ反映し、質問を閉じて要件解決を続ける。旧draft hashもAuditへ残し、Blocking question 0になった版だけをAuthorization対象にする。

Envelope発行後の回答がTask goal、success criteria、Input revision、Risk、Operation、Path、Network、Dependency、Gate、Approvalのいずれかを変える場合、元Taskへ追記せず新しいSpecification、ContextPack、Envelopeを持つ新Taskにする。変更しない補足回答だけ、同じInput revisionと有効Envelopeを再検証して`resume_state`へ戻す。副作用Operationは開始時点で`expires_at - now >= operation.timeout_ms`を満たさなければ開始しない。ただし、署名済み`LongRunningGrantV1`を原子的に消費した一回限りOperationは、その`complete_by`をhard deadlineとして扱う。

## 8. 要件解決と質問Policy

### 8.1 質問分類

| Class | 例 | 動作 |
|---|---|---|
| Blocking | 対象Platform、Online要否、課金、年齢層、権利不明Asset | 回答まで該当Scopeの実装を開始しない |
| High | Camera、Control、勝敗、Save、主要Art direction | 推奨案と影響を提示し回答を求める |
| Medium | 敵数、Animation細部、UI配置 | Defaultを明示して進行可能 |
| Low | 内部命名、微細な装飾 | Project規約から自動決定 |

AIは質問を無制限に並べない。初回はBlockingとHighを最大7問へまとめ、各質問へ推奨Default、選択による差、後で変更可能かを付ける。7問を超える場合は「Game core」「Visual」「Platform／Business」の順に分割する。

### 8.2 大まかなPromptからの流れ

1. Promptを`GameIntentDraft`へ抽出する。
2. Capability、Platform、権利、Online、Save、年齢、Performanceに対して不足を検査する。
3. Blocking／Highだけを質問する。
4. AIが理解した`GameBrief`を、人間向け要約と機械可読要件の両方で提示する。
5. 人間が承認または修正する。
6. First PlayableのScope、Non-goal、品質Targetを固定する。
7. System単位でGameplayDefinitionまたはC++を選択し、必要なら明示的なCapability境界で併用する。
8. 短縮Game全体と一つの代表完成区間を生成する。
9. Test、試遊、Budget、Accessibility、Platform gateを実行する。
10. 会話または手動Editorから次のChangeSetを作る。

## 9. Risk classと自動化範囲

| Class | 代表変更 | AI生成 | 自動検証後の自動昇格 | 必須Approval |
|---|---|---:|---:|---|
| R0 | 読取、検索、説明、Report | 可 | 状態変更なし | 不要 |
| R1 | 文書、非実行Sample、Editor layout個人設定 | 可 | 可。ただしprotected branch外 | Owner不要、Gate必須 |
| R2 | GameSpec、Scene、UI、Asset設定、GameplayDefinition | 可 | 署名済み事前委任Policyのallowlistにある可逆Operationを非Release branchへ適用する場合だけ | Author 1名、または同等Scopeの有効な事前委任 |
| R3 | NativeGameModule（Project C++）、Shader、Build設定、Dependency、Schema互換変更 | 可 | 不可 | Code owner 1名＋全Gate |
| R4 | Engine core、Memory、Threading、Serialization、Security、Source Gate | 可 | 不可 | Domain owner＋独立Reviewer |
| R5 | merge、tag、sign、Store upload、Production secret、公開Release | Proposalだけ可 | 禁止 | Human release owner＋分離Release pipeline |

R4でもAIは隔離WorktreeへPatchとTestを生成できる。「提案だけ」とはSourceを生成できない意味ではなく、正規Branchへ自動昇格できない意味である。R5 OperationはModelへToolとして公開しない。

Riskは変更後の最大影響で決める。文書TaskでもBuild scriptやPolicyを変更すればR3以上である。Test削除、Assertion緩和、Budget引上げ、Schema制約削除、Approval条件削除は対象実装と同じか一段高いRiskとする。

R2の事前委任はOperation ID＋version、PathGrant、最大Entity／byte数、Target branch、有効期限、Rollback可能性を固定し、Save schema、Public API、Asset license、Dependency、Security、課金、公開配布を含めない。Promotion Serviceは実DiffからRiskを再分類する。実RiskがEnvelopeより高い、または委任Scopeを外れる場合は昇格せず、新しいEnvelopeとApprovalを要求する。AIのRisk自己申告でRiskを下げない。

### 9.1 Activation Gate

設計済みであることと、製品で有効化済みであることを分ける。Activation IDの番号は実装上の識別子であって全面的な直列順序ではなく、次の依存DAGに従う。あるOperationに必要なGateが未完成なら、低いGateへfallbackしない。

| Activation | 依存Activation | 有効にする範囲 | 解放条件 |
|---|---|---|---|
| A0 Authoring Core | なし | R0、R1、R2の構造化編集とGameplayDefinition | MCD最小Contract、署名Envelope、C++ Gateway、ChangeSet transaction、Definition Validator／Cooker、R0–R2 Policy test、Undo、adversarial Eval |
| A1 Project Source | A0 | R3のNativeGameModule（Project C++）、Shader、Build設定 | `HyperVIsolatedWorkerV1`または同等remote Worker、Base image／IPC conformance、Promotion、全Path／Network／Process negative test、primary／secondary compiler、static／sanitizer、Code owner Review |
| A2 Engine Maintenance | A1 | R4のEngine／Editor／Orchestrator保守 | 5 State modelとtransition conformance、独立Reviewer、threat／lifetime analysis、full regression、fault／soak |
| A3 Release | A0 | R5のmerge／tag／sign／Store提出 | 分離Release Coordinator／Build Worker／Signing／Upload Service、clean reproducible build、SBOM、DSSE SLSA provenance、Platform署名、device／Store gate |

現在ActivationはMCD ProfileとEditor UIへ表示する。未解放OperationはTool catalogへ出さず、内部呼出しにも`MIRA-POLICY-CAPABILITY_NOT_ACTIVATED`を返す。A3はA2を必須としないが、Release Coordinatorは入力ごとに、そのArtifactを生成したCapabilityのActivationとRisk Gateを検査する。したがってR3 Sourceを含むReleaseにはA1が必要であり、R4 AI保守で生成したEngineを含むReleaseにはA2も必要になる。A0だけでGameplayDefinitionを使う2D Manual First Playableと構造化AI loopを完成できるようにする一方、NativeGameModule生成をMVP-A成功条件へ含めるため、正式なMVP-A完了にはA1を必須とする。未Activation機能を「将来対応」と偽って成功表示しない。

## 10. Game実装方式の選択

### 10.1 固定原則

すべてのGame機能は、次の順に検討する。

1. 正規GameSpec、Component、Asset、Rule、FSM、Behavior Tree、Quest、Dialogue、Ability、UI Flow等の`GameplayDefinition`で表現する。
2. 既存Capabilityの組合せ、参照解決、offline Cook、event index、data layout最適化で解決する。
3. 未提供Gameplay Capability、新規Algorithm、または同一fixtureでBudget超過が確認されたhot pathだけを`NativeGameModule`へ実装する。
4. Platform／Native SDK統合はProject Gameplay moduleへ入れず、R4のEngine／Platform Adapter変更とする。
5. 大きなSystemは`GameplayDefinition`と`NativeGameModule`を、MCDで定義されたtyped command／event／snapshot境界で分割する。

汎用Game scripting runtime、bytecode interpreter、JIT、FFIは選択肢に含めない。AIはGameの「大きさ」やGenre名だけでC++を選んではならない。各Systemのデータ量、呼出頻度、Latency、Memory、決定論性、Platform、Security、再利用性を根拠にする。

### 10.2 選択Matrix

| 条件 | GameplayDefinition | C++ |
|---|---:|---:|
| Scene、数値、Rule table、Dialogue、Quest、Ability、UI Flow | 第一選択 | 原則不要 |
| 頻繁に調整するGame固有Rule | 第一選択 | 未提供Capabilityだけ |
| 1 frameに大量entityを処理 | Data定義とCook最適化 | 計測後のKernel候補 |
| Physics／Render／AudioのLow-level統合 | 登録済みCommand設定 | Engine Adapter |
| Native SDK／Platform API | 設定とCapability参照だけ | Engine Adapterだけ |
| Security境界、Serializer、Allocator、Job system | Schemaで参照だけ | Engine core |
| Runtime生成内容 | 検証済みDataだけ | 生成、download、load禁止 |

### 10.3 性能による昇格基準

Systemごとに`BehaviorBudget`がない場合、AIは性能を理由にGameplayDefinitionまたはC++を確定してはならず、`MissingImplementationBudget`をBlocking診断にする。

候補実装は同じReference fixture、同じTarget Profile、clean processで10分間を3回測定し、最悪回のP95、peak memory、deadline miss、allocation countを使う。

- P95 timeとpeak memoryが各割当Budgetの80%以下、deadline miss 0、決定論／安全性Gate合格: GameplayDefinitionを維持する。
- 80%超100%以下: 自動確定せず、DefinitionのCook／index／layout最適化とC++候補を同一fixtureで比較する。
- 100%超またはdeadline miss 1件以上: NativeGameModule候補、またはGameplayDefinition＋C++のtyped境界へ昇格する。
- Platform／Native SDK APIが必要: R4 Engine／Platform Adapterとして別提案にする。Engine禁止APIが必要な設計はC++化で迂回せず拒否する。
- C++化で改善が5%未満または測定Noise内: 調整容易性を優先しGameplayDefinitionへ戻す。

80%は20%の変動余裕を確保するMiraikanai初期Policyであり、Vendor公式値ではない。Reference hardwareの実測とADRによってSystem別に変更できるが、Task内のAI判断だけでは変更できない。

## 11. Source生成・直接編集の隔離

### 11.1 Workspace

信頼済みSource Promotion Service内のWorkspace BrokerがTaskごとに、許可済みbase commitからHost側Staging Worktreeを作る。Source WorkerへHost Pathを渡さず、Brokerがbase Treeと許可Inputをcanonical Source Bundleへexportし、Worker内のguest workspaceへ転送する。Workerが返した`SourceDeltaV1`／ArtifactをBrokerがuntrusted byte列として検査し、Staging Worktreeへ適用してから新たにDiffを計算する。Source Worker自身へWorktree作成、branch ref作成、`.git` read／writeを許可しない。main worktree、Host staging、credential directory、ユーザーhome、他Projectを直接mountまたはwrite可能にしない。

転送形式はMCD Typeの`SourceBundleV1`と`SourceDeltaV1`だけにする。任意tar／zip、unified diff、guest filesystem imageをHostへ直接展開しない。

- `SourceBundleV1`は`task_id`、`base_commit`、`source_tree_hash`、`contract_set_hash`、昇順のEntry array、Bundle hashを持つ。Entry contentはbase commitのGit blob byteまたは正規Artifact byteから取り、Host Working treeの`autocrlf`結果を使わない。Entryはnormalized Repository-relative path、`regular_0644`／`regular_0755`、byte size、SHA-256、content-addressed Blob IDだけを持ち、`.git`、symlink、junction、reparse point、hardlink、alternate data stream、device、sparse fileを含めない。
- `SourceDeltaV1`は同じbase、`create`／`update`／`delete`、path、expected before hash／mode、after hash／mode／size／Blob IDを持つ。Renameは暗黙検出せずdelete＋createとして表し、PathGrantは両Pathを個別検査する。
- ManifestはRFC 8785 JCS、Entry pathのunsigned UTF-8 byte順、重複Path禁止とする。Blobは非圧縮raw bytesで初期実装し、Manifest size、Entry数、単一Blob、総byte数をEnvelopeで制限する。圧縮はdecompression ratio、dictionary、streaming limitを定義した別versionまで追加しない。
- Text対象はUTF-8 without BOM、LF、invalid Unicode拒否とし、BinaryはOperationとextensionの双方で明示許可された場合だけ受け取る。HostはBlob hashと全Manifest制約を検証してから、resolved-handle Path検査付きでStagingへ適用する。

Windows A1／A2のPromotion可能なSource実行Backendは`HyperVIsolatedWorkerV1`に固定する。

- HostはWindows 11 25H2 Pro／Enterprise、SLAT、hardware virtualization、Hyper-V有効を必須にする。Editor本体の閲覧／構造化AuthoringはHomeでも利用できるが、HomeまたはHyper-V unavailableではlocal Source実行を`CapabilityNotActivated`にし、同じProfileのremote Workerを使えなければfail closedにする。
- WorkerはGeneration 2 Windows VM、Secure Boot有効、vGPU／virtual NIC／Enhanced Session／clipboard／device／host folder共有なしとする。Source WorkerへSecretとProduction Dataを入れないためvTPMを必須にせず、Base image integrityはHost側の署名ManifestとSHA-256で検証する。
- Toolchainを焼いたimmutable Base VHDXをcontent-addressed Artifactとして固定し、Taskごとに一段だけのdifferencing VHDXを作る。VM停止後にOutput取得とReceipt確定を行い、Task diskを破棄する。Differencing chainの再利用、checkpointからのTask継続、別Taskへのdisk持越しを禁止する。
- Windows guest imageをEngine installerへ同梱できるとは仮定しない。Base VHDXは、組織またはUserが利用権を持つMicrosoft mediaと固定ToolchainからImage Builderで作成するか、適切なLicenseを持つremote Worker Serviceが提供する。OS／ToolchainのLicense EvidenceがないImage、第三者が再配布した不明VHDX、Evaluation期限切れImageをA1／A2へ使用しない。
- Source BundleとOutput BundleはHyper-V socket `AF_HYPERV`／`SOCK_STREAM`／`HV_PROTOCOL_RAW`上のlength-prefixed protocolでだけ転送する。接続はHostが指定したVM ID＋Service GUID、Task nonce、Envelope hash、Bundle hashを相互照合し、Protocol外Messageを拒否する。SMB、PowerShell Direct、Guest Services file copy、shared clipboardをData pathにしない。
- Source Worker VMは初期`NetworkPolicyV1=deny_all`とし、virtual NIC自体を付与しない。Dependencyは承認済みcontent-addressed mirror ArtifactをSource Bundleへ含め、Build中にdownloadしない。Networkが必要なResearchはSource Taskと分離した別Worker／Envelopeで行う。
- HostはEnvelopeの`peak_commit_bytes`とCPU上限をHyper-V設定で、guest内Process treeをJob Objectで二重に制限する。wall timeout、VM boot timeout、Hyper-V socket idle timeoutを別値として固定し、いずれか超過でVMを停止してOutputを昇格しない。
- Hyper-V管理Service、Base VHDX、Host staging、AF_HYPERV endpointへAI Orchestrator、Agent Host、guest WorkerのACLを与えない。VM作成／停止／disk破棄OperationはModel Tool catalogへ公開しない。
- Administrator操作は初回Feature／Service installとBase image登録に限定する。Editor、AI Orchestrator、Provider Adapter、MCP Adapterをelevatedで起動せず、専用Hyper-V管理Serviceが署名EnvelopeとACL付きIPCを検証して固定VM Operationだけを実行する。

必須防御は次のとおりである。

- `HyperVIsolatedWorkerV1`または同等条件を満たすremote hardware-VM Worker。Job Object／restricted token／AppContainerだけをPromotion可能なR3／R4実行境界にしない。
- 既定Network deny。必要時はEnvelopeのexact domain allowlist。
- Network許可はBroker Proxy経由だけとし、DNS rebinding、Redirect、private address、response sizeを`NetworkPolicyV1`で強制する。
- Process tree全体へのwall time、CPU、commit memory、child count、output size上限。
- SecretをEnvironment、command line、working treeへ渡さない。
- Reparse point、junction、symlink、hardlink、submodule境界を解決後Pathで検査する。
- Repository外へ解決するPath、case-fold後の衝突、reserved device nameを拒否する。
- AIのShell permissionやMCP annotationをSecurity boundaryとして信用しない。
- Sandboxを起動できない場合はfail closedとし、非隔離実行へfallbackしない。

### 11.2 Promotion

Source Worker終了後、信頼済みBrokerが返却PatchをHost Staging Worktreeへ適用し、base commitとの差分を新たに計算する。AIが提出したFile listとguest内Diffは参考情報であり、昇格対象の正本にしない。

Promotion Gateは次を確認する。

1. base commitとTask input revisionが一致する。
2. 変更PathがEnvelope allowlist内である。
3. binary、generated file、submodule、dependency、Policy変更が明示許可されている。
4. Test／Schema／Budget／Security条件を弱める変更が分類されている。
5. required gateが信頼済みRunnerで成功している。
6. Approval対象Diffと現在Diffのhashが一致する。
7. 昇格後のbranchをclean buildし、同じ結果を再検証する。
8. C++ Source／Build Taskでは`CppDependencySetV1`、Sourceの実import／include、CMake／Module DAG、Active `CxxFrontendProfileV1`、Target別`BuildDriverProfileV1`が一致し、Experimental CX1 artifact、Makefiles／`ndk-build`経路、BMIが昇格対象に含まれない。

AI Processへ`.git` writeを許可せず、AIは`git commit`、tag、branch refを作成しない。Source Workerの出力はWorking tree差分とContent-addressed Artifactだけとする。正規履歴へ取り込むcommitはPromotion Serviceが検証済みTreeから作成し、Author／Generator／Reviewer来歴を別Fieldへ記録する。

### 11.3 Releaseの秘密分離

SourceとBuild scriptを実行するProcessへSigning keyまたはStore credentialを渡してはならない。Release Serviceという論理境界を一つのhost、OS account、container、credential setとして実装せず、次の固定pipelineへ分離する。

```text
Release Coordinator
  -> preparation Taskのephemeral Release Build Worker [sourceあり、secretなし]
  -> UnsignedReleaseArtifactV1 + Build Receipt
  -> R5 ReleaseTransactionV1
  -> Platform Signing Service [source／script／compilerなし、signing keyあり]
  -> SignedReleaseArtifactV1 + Signing Receipt
  -> Store Upload Service [source／signing keyなし、短命upload credentialあり]
```

Releaseのclean build、full test、SBOM、provenance、unsigned package生成は、Artifact由来のRiskに対応するR3またはR4のpreparation TaskとしてR5より前に完了させる。長時間Buildやsoakは独立Jobへ分割し、各Receiptをunsigned artifactへ連結する。R5の15分EnvelopeはSourceを実行せず、完成済みunsigned artifactを一回の`ReleaseTransactionV1`へ投入する開始窓である。

`UnsignedReleaseArtifactV1`はTarget、Source tree hash、Toolchain／Contract／Policy hash、SBOM hash、entryごとのnormalized path／size／SHA-256／kind、Build Receipt hashを固定する。Signing ServiceはManifest外Entry、hash差、path traversal、link／device、未許可Executable、Entitlement／Capability差を拒否し、受信byteを再hashする。Signing commandはPlatformごとの固定Operationだけで、任意shell、Project File、Build system、pre／post hookを公開しない。Signing keyはOS key store、HSMまたはPlatform-managed cloud signingに置き、Model、Orchestrator、Worker、Environment、Logへ秘密materialを返さない。

Unsigned package自体も悪意あるInputとして扱う。鍵を持たないPackage Inspection Serviceが、sandbox内でformat parser、path、size、Executable、Entitlement／Capability、nested contentを検査し、正規Manifestと検証Receiptを作る。Platform Signing ServiceはこのReceiptと同じbyteだけを新しいephemeral signing workspaceへmaterializeし、固定hashのPlatform toolと非搬出Key handleだけを持つ。Key BrokerはR5 Envelope、Approval、unsigned root、Signing profile、one-shot transaction IDを再検証してTask限定Key leaseを発行し、Transaction終了時に失効させる。Signing instanceへ一般Network、他Key、他Artifact、Build cacheを与えず、Platform timestamp／certificate serviceが必要な場合だけBroker allowlistを使う。

署名後は鍵を持たない別Verifierが全Entryを再parseする。Androidは署名前から存在するAAB entryの解凍後byte hashを不変とし、`META-INF`署名EntryとallowlistしたZIP構造metadata以外の意味変更を拒否する。AppleはMach-O code-signature region、`_CodeSignature`、embedded provisioning、署名で正規に変化するbundle metadata以外のbyteと全Executable集合を不変とし、Platform fixtureで差分規則を固定する。新規Executable、Code／Resource変化、未承認Entitlement、未知の署名差分があればArtifactを破棄する。Parser／Signer侵害時にもKey export、別Artifact署名、検査外code変更を許さないことをnegative testとKey-use Auditで検査する。

R5の人間Approvalは少なくともunsigned artifact root、Target／channel、Version、Signing profile ID、Store listing revisionを固定する。Signing後のReceiptはunsigned rootとsigned rootを両方記録し、Upload ServiceはReceipt chain、Approval、Store policy lock、有効期限を再検証する。Upload credentialはTask単位の最小Role／短命Tokenとし、Signing Serviceと共有しない。取消、期限切れ、検証失敗後は新しい外部副作用を開始しない。

Platformがmanaged signingを提供し、Build runnerへprivate keyを露出しない場合は、同一製品内の論理pipelineでも構わない。ただし、Project codeからkey／upload credentialを読めないこと、ephemeral isolation、承認hashとの一致、署名／upload ReceiptをPlatform別conformance testで証明してからActivationする。証明できないPlatformは、credential-bearing Build hostへfallbackせずReleaseを停止する。

## 12. Provider API、MCP、CLI、Pluginの役割

### 12.1 公式構成

| 接続方式 | 公式用途 | 正本にならないもの |
|---|---|---|
| Provider API | 製品内Chat、質問、計画、構造化提案、継続会話、評価 | Project状態、権限、Schema |
| MCP Server | 外部Codex／Claude／Desktop App／CLIからの共通操作 | Commit authority、Provider設定 |
| Conformance済みCLI Agent Host＋Source Worker | Engine／Project Sourceの隔離編集、Build、Test | main branch、Release、Credential公開 |
| Optional Plugin | Provider UIへのShortcut、Panel、Prompt、Skill | Engine必須機能、Security Policy |

EngineのGame制作機能をPluginだけに実装してはならない。Pluginが未導入でもAPIとMCPで全公式操作を実行できなければならない。

外部AI Clientの公式安全境界を次に固定する。

- Authoringだけを行うCodex／Claude／Desktop AppはRepositoryと正規Project storageをread-onlyにし、MCPのQuery／Proposal Toolだけを使う。
- Source編集を行うCLIは、Engine管理Launcherが署名Envelopeを検証してからSource Worker identity、専用Worktree、OS sandbox、Broker Networkで起動する。この経路だけを「隔離Source生成」と表示する。
- ユーザーが任意のCLI／Desktop Appを通常User権限またはfull accessでRepositoryへ直接起動した場合、そのClientは公式Security boundary外である。MCP Tool制限によってFile／Shell直接操作を阻止できるとは表示しない。
- Security boundary外で作られたFile変更は正規Project revisionまたは検証済みSource commitではない。External patchとして再取得し、Risk再分類、全Gate、必要Approval、Promotionを通すまでBuild／Release入力にしない。
- Repository ownerと同じ権限を持つ外部ProcessによるWorking tree破壊、情報持出し、履歴改変をEngineだけで防げるとは保証しない。公式Launcher、OS ACL、VCS／backup、Secret分離を必要条件にする。

Credentialの所有経路も混在させない。

| Mode | Provider CredentialのOwner | Source編集 |
|---|---|---|
| 製品内AI | Provider Adapterだけ | OrchestratorのProposalをSource Workerが適用 |
| 外部Client＋MCP | Codex／Claude等のClientとUser | MCP Proposalだけ。EngineはClient Credentialを受け取らない |
| Managed CLI Agent Host＋Source Worker | conformance済みClient hostまたは専用Credential broker | 専用Worktreeだけ |

Managed CLIを許可するMCD `profile` instanceの`ExternalClientSecurityProfile`はClient名、exact version／hash、認証方式、Credential storage、Model API endpoint、親ProcessとTool childのtoken／environment分離、filesystem／network sandbox、MCP version、更新期限、conformance receiptを固定する。Agent HostはSource Worker境界外でModel APIだけを扱い、File／Shell Tool childはSecretを持たないrestricted tokenとBroker経由で動かす。Credentialをplain environment、command line、Working treeへ置くClient、Tool childがCredential fileを読めるClient、EngineのBrokerを迂回してraw socketを使うClientはManaged modeへ昇格しない。Profile不在、version差、conformance期限切れではMCP Proposal modeだけを許可する。

### 12.2 外部AIへ公開するMCP Tool

初期Tool setを次に固定する。Tool名とSchemaは実行可能契約から生成する。

| Tool | 種別 | 効果 |
|---|---|---|
| `mira.project.describe` | Query | Project revision、Target、Profile概要 |
| `mira.requirements.list` | Query | Requirementと未解決Question |
| `mira.capabilities.search` | Query | Capability ID検索 |
| `mira.capabilities.read` | Query | Schema、制約、Budget、例 |
| `mira.snapshot.read` | Query | 許可されたAuthoring snapshot |
| `mira.changeset.validate` | Pure validation | Proposalを正本に対して検証 |
| `mira.changeset.preview` | Pure preview | Diff、影響、予測Costを返す |
| `mira.changeset.submit` | Proposal write | StagingへProposalを保存。正規Projectは不変 |
| `mira.source_task.create` | Proposal write | Source Taskの承認要求を作る |
| `mira.source_patch.submit` | Proposal write | 隔離PatchをStagingへ登録 |
| `mira.task.status` | Query | State、Diagnostic、Gate進捗 |
| `mira.approval.request` | Proposal write | 人間UIへ承認依頼。承認そのものではない |

`commit`、`merge`、`sign`、`release`、`secret.read`、`policy.override`を含むToolはModelへ公開しない。Authoring Project revisionはC++ Command Gatewayが、署名Envelopeと必要な事前委任または`ReviewReceiptV1`を照合してTransaction確定する。SourceのGit commitはSource Promotion Serviceだけが同じ証拠を照合して作成する。Policy Service、Editor Client、AI Orchestrator、Modelは正規Project revisionまたはGit commitを直接作成しない。

MCP ToolのannotationはClient表示の補助であり、Server側のAccess controlを置き換えない。InputとOutputはServerが正本Schemaで再検証し、timeout、rate limit、auditを必須にする。

初期MCP transportはlocal stdioだけを公式対応にする。外部Clientが起動する薄いMCP AdapterはProject Fileを直接開かず、ACL付きlocal IPCでC++ Gatewayへ接続する。Editorのinteractive pairingで発行する署名済み`McpSessionGrantV1`はClient表示名、Adapter binary hash、OS user SID、Project ID、read sensitivity上限、Proposal Operation allowlist、発行時刻、有効期限最大60分、session nonceを固定する。Session GrantはQuery／Proposalの接続認証であり、Task Authorization、Approval、Promotion権限を与えない。Adapter再起動、Project切替、binary hash差、期限切れで再Pairingする。

MVPではTCP listen、Streamable HTTP、remote MCP、port forwardingを無効にする。将来のremote transportはTLS、OAuth resource server、redirect／token audience、tenant分離、revocation、rate limit、remote threat model、penetration testを含む別ADRとActivation Gateを必要とし、local stdioのSession Grantを転用しない。

## 13. Model routingとProvider Manifest

Provider／Model名をEngine codeへhard-codeしない。`ProviderManifest`が次を固定する。

- Provider、endpoint、API version、SDK exact version。
- Model exact IDまたはProviderが保証するsnapshot。
- Role: `planner`、`implementer`、`reviewer`、`classifier`。
- Reasoning／thinking設定、temperature等の有効Parameter。
- Tool／Structured OutputのProjection version。
- Context、output、tool call、cost、latency上限。
- Data retention、training use、region、encryption、logging Policy。
- 合格したEval suite ID、run ID、基準日。
- fallback Modelと、fallback時に許可するRisk class。

2026-07-19時点の調査Baselineでは、OpenAIは新規ProjectへResponses APIと`gpt-5.6`系を案内し、複雑なCodingには`gpt-5.6-sol`を提供している。AnthropicのTool文書は複雑なToolと曖昧なQueryにClaude Opus 4.8を案内している。これは現況記録であり、Architecture定数ではない。導入時にProvider Manifestへexact IDを固定し、同じEvalを通過させる。

Provider API credentialはAI Orchestratorと別の低権限OS ProcessであるProvider AdapterだけがSecret storeから読取り、ACL付きIPCでRequestを受ける。Model Context、Tool argument、AI Orchestrator log、Source Worker environmentへCredentialを渡さない。`secret_policy=no_secret_access`はProvider呼出し自体を禁止する意味ではなく、非信頼ProcessからCredentialを分離する規則である。

大規模設計、Engine core、複雑なDebugは最上位Coding／reasoning Role、日常的な実装とReviewはbalanced Role、分類と簡単な抽出はsmall Roleを候補にする。Role割当はCostだけで決めず、Task別Evalに合格したModelだけを使う。

## 14. 自動修復Policy

| Risk | 自動修復 | 上限 |
|---|---|---:|
| R1 | format、link、明白なSchema error | 3回 |
| R2 | Validator／Test診断に基づく局所修正 | 2回 |
| R3 | compile／targeted testの局所修正 | 1回 |
| R4 | 自動修復なし。診断と修正案を提出 | 0回 |
| R5 | 実行不可 | 0回 |

次の場合は残回数に関係なく停止する。

- 同じ`diagnostic_code`と同じSource locationが再発した。
- 修正ScopeがEnvelope外へ広がる。
- Requirement、Test、Budget、Policyの緩和が必要になる。
- 新規Dependency、Network、Secret、Migrationが必要になる。
- Input revisionまたはProvider Manifestが途中で変化した。

Retryは元Proposalへ上書きせず、親Revisionを持つ新しいAttemptとして保存する。Cost、Token、時間のBudget超過時もfail closedにする。

## 15. 初心者と上級者の同居

Beginner Workspaceは複雑な権限を隠すが、省略しない。

- AIが質問、推奨案、Game Brief、進捗、Play Test結果を平易な日本語で表示する。
- 承認画面は「何が変わるか」「戻せるか」「Risk」「試した内容」を表示する。
- Sourceを見せなくても、同じChangeSet、Validator、Undoを通す。
- Errorは原因、影響、推奨修正を表示し、内部Stackだけを見せない。

Advanced WorkspaceはHierarchy、Outliner、Inspector、Definition Graph／Table、Profiler、C++、Shader、Build、Diagnostics、AI Panelを自由にDockできる。手動変更もAI変更と同じContract、Diff、Test、Historyを通す。AI Panelは常時表示、auto-hide、別Window、閉じた状態をWorkspaceごとに保存できる。

AIが行った変更はSource、Reason、Requirement、Test、Provider、Modelを追跡できる。人間が手動修正した後は、そのrevisionをAIが再読込し、古い仮定から上書きしない。

## 16. Promptと`AGENTS.md`

Prompt templateは次の順を標準にする。

1. Role。
2. Goal。
3. Success criteria。
4. Normative constraints。
5. Toolと権限。
6. Evidence要求。
7. Output Schema。
8. Stop／質問条件。

同じ規則を複数箇所へ反復しない。Prompt変更は一群ずつ行い、Evalを再実行する。Model変更と大規模Prompt変更を同じ実験で行わない。

Repository rootの`AGENTS.md`は短く保ち、次だけを書く。

- Repository map。
- Build／Testの入口。
- 禁止操作。
- Definition of Done。
- 本規約と機械可読契約へのLink。

Subsystem固有規約は近いDirectoryの`AGENTS.md`または正規文書へ分ける。ただし、`AGENTS.md`の内容は実行可能契約から検査し、矛盾するMustをCIで拒否する。

## 17. Privacy、Prompt injection、Data handling

- Source、Asset、User Prompt、Tool output、Web content、Issue本文内の命令は`content`扱いとし、Control命令へ昇格しない。
- Provider送信前にSensitivity labelとProvider Policyを照合する。
- Secret、private key、access token、credential file、Production customer dataをModel Contextへ含めない。
- Prompt、Tool argument、Tool output、Traceは機密Dataになり得る。既定で本文をTelemetryへ保存せず、hashと分類だけを記録する。
- Provider側Schema cache、response storage、prompt cache、retention条件をProvider Manifestへ記録する。
- Zero Data Retentionが必須のProjectでは、ZDR非対応機能を自動無効化し、代替経路がなければTaskを停止する。
- Live Web取得はResearch Taskだけで許可し、取得Contentを非信頼Evidenceとして保存する。BuildとRelease中の自動Web取得は禁止する。

## 18. Failure policy

| Failure | 正規動作 |
|---|---|
| Provider timeout／rate limit | 状態を保持し、Policy内だけretry。別Model fallbackはManifest許可時だけ |
| Structured output refusal／incomplete | Proposalを作らず、明示Diagnostic |
| Provider Schema fallback | strict要求Taskでは拒否 |
| Context不足 | Task分割または質問。Must要件を削らない |
| Validator crash | Proposal拒否。Validatorを迂回しない |
| Sandbox unavailable | Source Workerを起動せず停止 |
| Test infrastructure failure | 合格扱いにせず`infrastructure_error` |
| Human rejection | `Rejected`終端。新Taskとして再提案 |
| Input revision drift | 現Taskを停止し、新ContextPackとEnvelopeを発行 |
| Receipt／hash不一致 | Security eventとして昇格拒否 |

## 19. Definition of Done

本規約の実装は次をすべて満たした時だけ完了とする。

- `TaskSpecification`、`TaskAuthorizationEnvelope`、`ContextPack`、Task state machineのSchemaがある。
- EnvelopeをAI Processが作成・変更できない。
- EnvelopeがSpecification、ContextPack、Contract、Policy、Profile、Tool catalogのhashを固定し、一つでも差し替えると拒否される。
- CNG／cross-platformのP-256署名round-trip、low-S、high-S拒否、未知／失効Key、期限、Nonce replayのnegative testがある。
- `LongRunningGrantV1`は二つの内部Operation以外、期限上限超過、Input／Runner／Destination差、二重Start、開始後のProcess種別追加を拒否し、完了／cancel／read-backへ収束する。
- R0からR5のPolicy testが全組合せで通る。
- 実DiffのRisk再分類がEnvelope超過とR2事前委任超過を停止する。
- External MCPに正規Commit／merge／sign／release Toolが存在しない。
- API、MCP、CLIのProposalが同じC++ Gateway validatorへ到達する。
- Source Workerがsandbox不能時にfail closedになる。
- Release Build Worker、Platform Signing Service、Store Upload Serviceが別identity／credentialで動き、悪意あるBuild scriptがSigning key／Upload credentialを取得できない。
- Signing ServiceがSource、Project、Build script、任意shell、Manifest外Fileを拒否し、Signing Receiptが承認済みunsigned rootとsigned rootを連結する。
- Keyless package inspectionとephemeral signing instanceを分け、Key Brokerが承認済みrootだけへone-shot非搬出Key leaseを発行し、別Artifact署名とKey exportを拒否する。
- `SourceBundleV1`／`SourceDeltaV1`がGit blob byteとbase hashを固定し、duplicate path、hash／mode差、symlink／reparse／ADS、未許可Binary、size超過を拒否する。
- C++ Source／Build Taskが`CxxFrontendProfileV1`、全`CppDependencySetV1` hash、Target別`BuildDriverProfileV1`を固定し、未宣言import／include、Module cycle、Makefiles／`ndk-build`、Generator override、CX1 artifact／BMIのPromotionを拒否する。
- `HyperVIsolatedWorkerV1`がBase image hash、Generation 2／Secure Boot、no NIC／no host mount、AF_HYPERV task binding、Task disk破棄、Hyper-V＋guest二重resource limitのconformanceを通る。
- Path escape、junction、symlink、submodule、DNS rebinding、Redirect、private address、Network、Secretのnegative testがある。
- Managed CLIは有効な`ExternalClientSecurityProfile`なしではSource Worker modeへ入れず、CredentialがTool childへ漏れない。
- Local MCPは`McpSessionGrantV1`のProject、Adapter hash、SID、Scope、期限を検証し、remote transportがMVPでlistenしない。
- Beginner／Advanced両Workspaceが同じChangeSetとHistoryを使用する。
- GameplayDefinition／C++選択がBudgetとBenchmark receiptを参照し、Genre名だけで決まらない。
- Provider ManifestなしのModel呼出をCIとRuntimeが拒否する。
- Prompt／Model／Tool変更に検証規約のEvalが適用される。
- 全State transitionが形式モデルまたは生成Conformance Testで検査される。

## 20. 一次資料と採用根拠

- [OpenAI Function calling: strict mode](https://developers.openai.com/api/docs/guides/function-calling#strict-mode): strict Schema、`additionalProperties: false`、全field required、Schema cacheと保持上の制約。
- [OpenAI Structured Outputs](https://developers.openai.com/api/docs/guides/structured-outputs): Structured Outputs、Schema divergence防止、refusal／mistake handling。
- [OpenAI Agent approvals & security](https://learn.chatgpt.com/docs/agent-approvals-security): OS sandbox、approval、Network deny、protected path、危険なfull accessの境界。
- [OpenAI GPT-5.6 Sol migration guide](https://developers.openai.com/api/docs/guides/upgrading-to-gpt-5p6-sol): 盲目的置換を避け、Role、Tool、Schema、Evalを維持して移行する方針。
- [OpenAI GPT-5.6 prompting guide](https://developers.openai.com/api/docs/guides/prompt-guidance-gpt-5p6): Goal、Constraint、Evidence、Completion bar、Tool境界、検証を明示する方針。
- [Anthropic Define tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools): Tool name、詳細説明、JSON Schema `input_schema`、例の役割。
- [Claude Code Security](https://code.claude.com/docs/en/security): Permission、sandbox、prompt injection、write scopeの防御。
- [Claude Code Sandboxing](https://code.claude.com/docs/en/sandboxing): Filesystem／NetworkのOS強制分離とfail-closed設定。
- [Model Context Protocol 2025-11-25 Tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools): Tool input／output Schema、structured result、access control、confirmation、audit。
- [Microsoft CNG Algorithm Identifiers](https://learn.microsoft.com/en-us/windows/win32/seccng/cng-algorithm-identifiers): Windows標準Algorithm IDとECDSA P-256対応範囲。Ed25519を初期標準実装の前提にしない根拠。
- [Microsoft CNG Key Storage Providers](https://learn.microsoft.com/en-us/windows/win32/seccertenroll/cng-key-storage-providers): Software KSPのECDSA P-256対応。
- [Microsoft: How Windows uses the TPM](https://learn.microsoft.com/en-us/windows/security/hardware-security/tpm/how-windows-uses-the-tpm): Platform Crypto ProviderによるTPM保護と非搬出Key。
- [Microsoft CNG Key Storage Property Identifiers](https://learn.microsoft.com/en-us/windows/win32/seccng/key-storage-property-identifiers): `NCRYPT_EXPORT_POLICY_PROPERTY`とExport許可Flag。
- [Microsoft Hyper-V system requirements](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/host-hardware-requirements): Windows 11 Pro／Enterprise、SLAT、hardware virtualizationのHost要件。
- [Hyper-V Generation 2 VM security](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/generation-2-virtual-machine-security-features): Generation 2、Secure Boot、vTPM／Shieldingの適用範囲。
- [Hyper-V `New-VHD`](https://learn.microsoft.com/en-us/powershell/module/hyper-v/new-vhd?view=windowsserver2025-ps): Base VHDXからTask別differencing VHDXを作る公式Interface。
- [Hyper-V socket integration service](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/make-integration-service): Network stackを使わない`AF_HYPERV`／`SOCK_STREAM` Host–guest IPC。
- [Windows Sandbox command line](https://learn.microsoft.com/en-us/windows/security/application-security/application-isolation/windows-sandbox/windows-sandbox-cli): Sandbox start／exec／share／stopとprocess output非対応の現行範囲。
- [Create Process in Sandbox APIs](https://learn.microsoft.com/en-us/windows/win32/secauthz/createprocessinsandbox): composable sandbox APIがExperimentalであることと現在のcontainment field。
- [SEC 1 v2.0](https://www.secg.org/sec1-v2.pdf): Elliptic curve point octet encodingとPublic key validation。
- [RFC 7518 §3.4](https://www.rfc-editor.org/rfc/rfc7518#section-3.4): ES256の`R || S`固定長Signature表現。

## 21. 明示的な非保証

- Schema準拠はGame designの面白さやC++の意味的正しさを保証しない。
- 形式モデル合格はC++実装全体の数学的証明ではない。
- SandboxとApprovalはPrompt injectionを消滅させず、被害範囲を制限する。
- Eval合格は未観測Inputでの完全性を保証しない。
- 最新Modelが既存Taskで最良とは限らない。
- AI生成Testだけでは生成実装の独立検証にならないため、Engine-owned invariantとgolden fixtureを別に持つ。
- Repository owner権限で任意に起動されたfull-access外部CLIのFile／Network操作は、MCPまたはEngineだけでは封じ込められない。
