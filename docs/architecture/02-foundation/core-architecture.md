# Miraikanai Engine Core Architecture

- 文書ID: mirakan.arch.core-architecture
- 状態: review
- 正本範囲: 基盤Layer、Host／Process境界、状態変更Gateway、ID・所有権、Thread／Job原則、Error規則、Build layer、Repository境界、Test／CI、Feature開始Gate
- 非正本範囲: 外部Tool・SDK・Libraryのversion／hash／license／取得元、命名、Memory／Pointer詳細、Runtime scheduling／budget／observability、Schema構造。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](toolchain-dependencies.md)、[Executable contracts](executable-contracts.md)、[Naming／Project layout](naming-project-layout.md)、[C++23 modules](cpp23-modules.md)、[Math／Core utilities](math-core.md)、[Memory／Pointers](memory-pointers.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論と規範

Miraikanai EngineはModular MonolithとPorts and Adaptersを採用する。製品の正規data model、公開Contract、編集Protocol、Validation、Lifecycle、UXをFirst-partyが所有し、Platform APIと外部Libraryはprivate Adapterへ隔離する。目的は、正しさを検証でき、計測に基づいて最適化でき、AI生成物にも同じ規則を適用できる基盤である。

- **必須**: 違反をReviewまたはCIで拒否する。
- **禁止**: First-party codeへ導入しない。
- **推奨**: 原則として採用し、例外はADRへ根拠と計測値を残す。
- **任意**: Project ProfileまたはSubsystem要件で選択できる。

Capability成熟度とPhase順序は[Product Plan](../00-product/product-plan.md)、外部toolchainの固定値は[Toolchain／Dependencies](toolchain-dependencies.md)だけが決定する。

## 2. 後方互換性を持たないClean実装

Pre-1.0のEngine内部API、source layout、generated formatには旧設計互換layerを作らない。旧APIのalias、旧Directoryへのredirect、二重serializer、二重Build入口を残さず、変更時は同じChangeSetでcaller、fixture、documentationを更新する。

ただし、Commit済みProject source、Asset provenance、Save data、公開Packageは捨てない。永続形式の変更は、versioned migration、Before／After fixture、rollback可能なProject revision、Evidenceを備える。互換性を持たない対象は未公開のEngine内部設計であり、User dataではない。

## 3. LayerとHost境界

依存方向を次に固定する。

```text
Host / Editor / Tools
        ↓
Application orchestration / Build Gateway
        ↓
Domain contracts and runtime ports
        ↓
Foundation / Math / Memory contracts
        ↓
Private platform and vendor adapters
```

- Domain間の状態共有はtyped command、event、snapshot、query portを通す。別Domainのconcrete implementationを直接include／linkしない。
- Hostだけがconcrete Adapterを組み立てる。Domain codeはService Locatorやglobal mutable singletonでAdapterを探索しない。
- Vendor型、native handle、allocator型を公開API、MCD、永続formatへ露出しない。
- `EditorHost`はAuthoring状態、`GameHost`はCook済みRuntime状態、`WorkerHost`は隔離されたBuild／Import／Validation taskだけを扱う。
- AI Orchestratorは別Processとし、EngineのmemoryやProject fileを直接変更しない。現在契約参照できる型付きIPC Operationは[Executable contracts](executable-contracts.md)の`active_complete` exact 10 IDだけであり、同節のconditional legacy migration 4 IDとreserved 191 IDはIPC／Provider／MCPへ投影しない。さらにcurrent Signer Policyがemptyな間は10件もoperationalではなく、Operation名、文書内template、未Activation候補からdispatchを推測しない。Authorizationは[AI Security／Approval](../01-governance/ai-security-approval.md)に従う。

## 4. Authoring状態とRuntime状態

Authoringの正規状態はrevision付きProject modelであり、RuntimeはCommit済みrevisionから生成したimmutable packageとephemeral stateを使う。Editor objectをRuntimeへ渡さず、Runtime objectをProject sourceへserializeしない。

状態変更経路は次の一つに閉じる。

```text
Intent / UI / AI proposal
  -> exact outer MCD Operation plus typed Project change primitives
  -> validation and authorization
  -> Source変更時だけbase revision Nでprepromotion Validate / Build / Test
  -> Source変更時だけSource promotion
  -> Project ChangeSet commit N -> N+1
  -> committed revision N+1でfinal Validate / Cook / Game Candidate / Test / Package
  -> immutable runtime package activation
```

Preview、Build、Cook、RuntimeはProject revisionを勝手に進めない。ActivationとPromotionは成功Receiptだけでなく、正しいAuthorization、入力hash、Target Profileを必要とする。

## 5. ID、参照、所有権

- 永続identityは意味のある型付きIDを使い、display name、path、配列index、native pointerをidentityにしない。
- IDの構造と共通Envelopeは[Executable contracts](executable-contracts.md)が所有する。各DomainはIDが表す対象とlifecycleだけを所有する。
- 所有者は一つとし、共有される対象はownerがlifetimeを管理し、利用者は非所有参照を持つ。
- C++のpointer taxonomy、arena、allocator、OOMは[Memory／Pointers](memory-pointers.md)を唯一の正本とする。
- `null`を通常の失敗表現にしない。欠損が意味を持つ場合だけoptionalを使い、失敗はtyped resultとstable diagnosticで返す。

## 6. Thread、Job、決定性

- mutable stateはowner threadまたは明示scheduler phaseだけが変更する。
- Job inputはimmutable snapshotまたはJob lifetime中に固定されたspanとし、captureしたraw owner pointerへ依存しない。
- cancelは協調的に行い、cancel後のpartial outputを正規Artifactへ昇格しない。
- callback、job、event配送中の再入的なWorld構造変更を禁止する。
- 同じProject revision、Target Profile、toolchain lock、seedから同じ正規Artifact hashを得る。この一致はthread数、Job実行順序、実行時刻に依存してはならない。hash一致の同値クラスはtoolchain lockの`profiles[].host`が定めるHost（OS／architecture）単位とし、異なるHost間のhash一致は各Domainがfixtureで検証したArtifact種別だけに要求する。性能目的の順序変更が意味結果を変える場合はDomainの明示Contractとfixtureを必要とする。

RuntimeのSimulation Advance、phase DAG、lifetimeは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、共通Budget、capacity、backpressureは[Runtime performance／capacity](../04-runtime/performance-capacity.md)が所有する。ここでは数値を再掲しない。

## 7. Error、Exception、Assertion

| 種別 | 用途 | 規則 |
|---|---|---|
| typed result | User入力、Asset、IO、Capability不足、回復可能な実行失敗 | closed statusとstable diagnosticを返す |
| exception | Host／Tool境界で捕捉できる予期しないLibrary失敗 | Domain／frame hot pathの制御フローに使わない |
| assertion | First-party invariant違反 | Development／CIで即時失敗し、外部入力検証の代用にしない |
| fatal | 継続が状態破損を招く場合 | crash evidenceを保存し、部分Commitしない |

Diagnostic IDの命名は[Naming／Project layout](naming-project-layout.md)、Evidence envelopeは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)が所有する。

## 8. C++とModule境界

First-party CPU codeはC++23を用いる。言語frontend state、Named Module、`import std`、CMake module edge、BMI identity、Cutoverは[C++23 modules](cpp23-modules.md)を唯一の正本とする。Compiler、CMake、Ninja、SDKのexact pinは[Toolchain／Dependencies](toolchain-dependencies.md)だけが所有する。

Foundationは次だけを共通原則とする。

- Public Contractとprivate implementationを別Targetにする。
- Header、Module interface、generated projectionへ同じ宣言を手書き複製しない。
- RTTIによるEngine reflection、巨大Header-only Engine、unity build前提、循環Moduleを禁止する。
- Mathの型、座標、数値、失敗契約は[Math／Core utilities](math-core.md)を参照する。

## 9. Build architecture

Buildを次の閉じた層へ分ける。外部toolのexact versionや取得情報は[Toolchain／Dependencies](toolchain-dependencies.md)へ集約する。

| Layer | 所有する責務 | 所有しない責務 |
|---|---|---|
| Build Gateway | 型付きRequest、Authorization、Target／Frontend／Driver Profile照合、resource予約、task順序、cancel、Diagnostic、Receipt | 任意shell、署名secret、C++依存の手書き複製 |
| CMake | First-party C++ target、compile definition、Module依存、code generation edge、native test／install artifact | Project revision、Content package、Store署名 |
| Build executor | 生成済みcommand DAGのincremental実行、compile／link／declared codegen | Product workflow、Toolchain選択、権限判断 |
| Content Build | Import、Cook、Shader、Derived Data、Target別Content Package | C++ target graph |
| Platform owner | Platform shell、resource、最終package、archive、署名 | portable C++ source graphの再定義 |

Build Gateway familyのatomic Activation後に限り、Editor、AI、CLI、CIはcurrent MCDとService allowlistへ同時登録されたBuild Gateway Operationだけを呼ぶ。現在のBuild系Operation集合、Owner Manifest、Service allowlist、Provider projectionはexact 0件であり、候補名によるdispatchを拒否する。AI向けBuild系OperationのMCD登録、Risk、Provider projection可否は[Executable contracts](executable-contracts.md)が所有し、Activation後の実行semantics、task順序、Receipt構造は本節のBuild Gatewayと各Owner文書が所有する。Generator出力や内部databaseを解析・書換えず、CMake File APIとEngine-owned Receiptを読む。Schema／generated Header／Moduleのcodegen edgeは全Input、Output、Byproduct、Depfile、working directoryを宣言する。Build executorの成功だけでPackage成功やPromotion可能と判定しない。

### 9.1 `OperationTaskV1`

本節のPackage／Device／Play／Debug／Task Control 14 IDと以下のTask／Receipt型は、[Executable contracts](executable-contracts.md)の`planning.operation_family.build_device_play_debug_task`に属する未Activation候補である。current MCD、Owner Manifest、Service allowlist、Signer Policy、Provider／MCP Tool、`OperationTaskV1` instance、完成Receipt集合はすべて空であり、現在はdispatchしない。以下の現在形は`activation.build_gateway.operation_pipeline.v1`がfamily全体を一つのContract set transactionでactivateした後の受入条件だけを表す。

atomic activation後、Build GatewayはPackage／Device／Play／Debug Operationを同じclosed task envelopeで実行する。`operation.task.status`、`operation.task.read_receipt`、`operation.task.cancel`は既存Taskを対象にする同期Control Operationであり、入れ子のTaskを新規作成しない。

```text
OperationTaskV1
  task_id
  operation_id
  request_sha256
  project_revision
  candidate_root_sha256
  target_profile_ref
  device_identity_ref?
  device_generation?
  authorization_envelope_hash
  consent_record_ref?
  idempotency_key
  state = queued | running | cancel_requested | succeeded | failed | cancelled
  receipt_ref?
```

`device_identity_ref`と`device_generation`はDeviceまたはremote Debugを対象にするOperationでは対で必須、それ以外では省略する。`consent_record_ref`はOperation Registryが明示consentを要求する場合だけ必須であり、空値や別Operationのconsentで代用しない。`receipt_ref`は非終端stateでは省略し、`succeeded | failed | cancelled`では同じtask ID、request hash、Project revision、Candidate root、Target、Device bindingを持つimmutable Receiptへ必須参照する。失敗詳細はReceiptが参照するtyped `MirakanDiagnosticV1`から取得し、自由文だけをTaskへ保存しない。

Activation後にOperation Registryへ同時登録する全14 Receiptは、次の共通subjectとOperation固有payloadを一つの署名済みRecordへ閉じる。

```text
OperationReceiptEnvelopeV1
  receipt_id
  operation_id
  invocation_kind = async_task | synchronous_control
  task_id?
  control_invocation_id?
  request_sha256
  project_revision
  candidate_root_sha256
  target_profile_ref
  device_identity_ref?
  device_generation?
  authorization_envelope_hash
  payload_contract_ref
  payload
  payload_sha256
  result = succeeded | failed | cancelled
  diagnostic_refs[]
  issued_at
```

`payload`は`payload_contract_ref`が指す次のclosed型一件で、`payload_sha256`はそのcanonical JCS bytesのSHA-256である。Operation IDとpayload型の組合せはclosed mappingであり、unknown Field、別Operationのpayload、型名だけ一致する任意JSONを拒否する。

Activation後、Package／Device／Play／Debugの11 Operationは`invocation_kind=async_task`、対応する`OperationTaskV1.task_id`を必須とし、`control_invocation_id`を省略する。`operation.task.status | operation.task.read_receipt | operation.task.cancel`は`invocation_kind=synchronous_control`、各同期呼出しに一意な`control_invocation_id`を必須とし、`task_id`を省略する。対象Task identityは型固有payloadの`target_task_id`だけが持つ。discriminatorとOperation IDの不一致、両IDのmissing、両方present、control Operationによる対象Task IDのEnvelope流用をschema negative fixtureで拒否する。

| operation_id | signed_record_purpose | payload contract | 完成Receipt alias |
|---|---|---|---|
| `operation.build.request_package` | `operation_receipt:operation.build.request_package` | `PackageReceiptPayloadV1` | `PackageReceiptV1` |
| `operation.device.install` | `operation_receipt:operation.device.install` | `DeviceInstallReceiptPayloadV1` | `DeviceInstallReceiptV1` |
| `operation.device.launch` | `operation_receipt:operation.device.launch` | `DeviceLaunchReceiptPayloadV1` | `DeviceLaunchReceiptV1` |
| `operation.device.reset_data` | `operation_receipt:operation.device.reset_data` | `DeviceDataResetReceiptPayloadV1` | `DeviceDataResetReceiptV1` |
| `operation.play.run_smoke` | `operation_receipt:operation.play.run_smoke` | `SmokeRunReceiptPayloadV1` | `SmokeRunReceiptV1` |
| `operation.debug.aggregate` | `operation_receipt:operation.debug.aggregate` | `DebugAggregateReceiptPayloadV1` | `DebugAggregateReceiptV1` |
| `operation.debug.query` | `operation_receipt:operation.debug.query` | `DebugQueryReceiptPayloadV1` | `DebugQueryReceiptV1` |
| `operation.debug.read_causality` | `operation_receipt:operation.debug.read_causality` | `DebugCausalityReceiptPayloadV1` | `DebugCausalityReceiptV1` |
| `operation.debug.read_replay_slice` | `operation_receipt:operation.debug.read_replay_slice` | `ReplaySliceReceiptPayloadV1` | `ReplaySliceReceiptV1` |
| `operation.debug.validate_finding` | `operation_receipt:operation.debug.validate_finding` | `DebugFindingValidationReceiptPayloadV1` | `DebugFindingValidationReceiptV1` |
| `operation.debug.support-bundle.generate` | `operation_receipt:operation.debug.support-bundle.generate` | `SupportBundleReceiptPayloadV1` | `SupportBundleReceiptV1` |
| `operation.task.status` | `operation_receipt:operation.task.status` | `TaskStatusReceiptPayloadV1` | `TaskStatusReceiptV1` |
| `operation.task.read_receipt` | `operation_receipt:operation.task.read_receipt` | `TaskReceiptReadReceiptPayloadV1` | `TaskReceiptReadReceiptV1` |
| `operation.task.cancel` | `operation_receipt:operation.task.cancel` | `TaskCancellationReceiptPayloadV1` | `TaskCancellationReceiptV1` |

本§9.1 family内の`signed_record_purpose`集合は上表のclosed exact 14値であり、`operation_receipt`や`receipt`等のgeneric値、unknown値、別行の値を許可しない。これは`MirakanSignedRecordV1.purpose`の全family共通上限を14値へ固定する宣言ではない。§9.2のBuild Candidate／Test familyを同時Activationするdestinationでは、同family固有のexact六purposeを別subsetとして追加し、active Build系unionは`14 + 6 = 20`、重複0とする。一方だけ未Activationならそのsubsetはexact `[]`であり、別familyのpurposeを合成しない。Verifierはfamily内の一行をatomic mappingとして選び、purpose、subject `OperationReceiptEnvelopeV1.operation_id`、`payload_contract_ref`、完成Receipt aliasを同一行へ束縛する。4項目の一つでも別行、unknown、欠落なら、payload hashと暗号署名が個別に正しくても完成Receiptとして受理しない。

Debug系6 payloadは[Debugging／observability／replay §13](../04-runtime/debugging-observability-replay.md#13-ai-debug-contextとdiagnosis)が所有する。Build Gateway／Device／Play／Task Control payloadは次のFieldだけを持つ。

```text
PackageReceiptPayloadV1
  contract_set_hash
  toolchain_lock_hash
  project_publication_binding:
    BuildProjectPublicationBindingV1(kind=committed_revision)
  project_validation_receipt_ref?: ProjectValidationReceiptRefV1
  project_validation_receipt_sha256?
  cook_receipt_ref?: CookReceiptRefV1
  cook_receipt_sha256?
  game_candidate_build_receipt_ref?: GameCandidateBuildReceiptRefV1
  game_candidate_build_receipt_sha256?
  candidate_test_receipt_refs[1..64]?: CandidateTestReceiptRefV1
  source_build_closure?
    kind: no_project_source
    | kind: selected_project_source
      project_source_promotion_receipt_refs[1..64]:
        MirakanSignedRecordRefV1(purpose=project_source_promotion)
      native_module_build_receipt_refs[0..64]: NativeModuleBuildReceiptRefV1
      project_shader_build_receipt_refs[0..64]: ProjectShaderBuildReceiptRefV1
  package_artifact_ref?
  package_artifact_sha256?
  package_manifest_sha256?

DeviceInstallReceiptPayloadV1
  package_receipt_ref
  package_receipt_sha256
  package_artifact_ref
  package_artifact_sha256
  consent_record_ref
  approval_record_ref
  install_transaction_sha256?

DeviceLaunchReceiptPayloadV1
  device_install_receipt_ref
  device_install_receipt_sha256
  package_artifact_ref
  package_artifact_sha256
  launch_descriptor_sha256
  process_instance_ref?

DeviceDataResetReceiptPayloadV1
  package_receipt_ref
  package_receipt_sha256
  package_artifact_ref
  package_artifact_sha256
  consent_record_ref
  approval_record_ref
  reset_scope_sha256
  reset_transaction_sha256?

SmokeRunReceiptPayloadV1
  package_receipt_ref
  package_receipt_sha256
  device_install_receipt_ref
  device_install_receipt_sha256
  device_launch_receipt_ref
  device_launch_receipt_sha256
  package_artifact_ref
  package_artifact_sha256
  fixture_ref
  fixture_sha256
  smoke_session_ref?
  smoke_result_sha256?

TaskStatusReceiptPayloadV1
  target_task_id
  target_operation_id
  target_request_sha256
  snapshot_sequence?
  target_state?
  target_receipt_ref?
  target_receipt_sha256?

TaskReceiptReadReceiptPayloadV1
  target_task_id
  target_operation_id
  target_request_sha256
  target_terminal_state?
  target_receipt_ref
  target_receipt_sha256

TaskCancellationReceiptPayloadV1
  target_task_id
  target_operation_id
  target_request_sha256
  target_state_before?
  converged_state? = cancelled | succeeded | failed
  target_terminal_receipt_ref?
  target_terminal_receipt_sha256?
```

全Fieldは`?`を付けたFieldを除き必須で、配列は重複なしunsigned byte順、unknown Fieldは禁止する。Envelopeの`result=succeeded`では各payloadのsuccess output groupをすべて必須にする。Packageのsuccess output groupはcommitted Project Publication binding、Project Validation、Cook、Game Candidate Build、1件以上のCandidate Test、Source build closure、package artifact ref／hash／manifest hashである。Installのtransaction hash、Launchのprocess instance、Resetのtransaction hash、Smokeのsession ref／result hash、Statusのsnapshot sequence／target state、Readのtarget terminal state、Cancellationのstate-before／converged state／target terminal Receipt ref／hashが各success output groupである。`result=failed | cancelled`ではそのgroupを全て省略し、`diagnostic_refs[]`を1件以上必須にする。同期ControlのEnvelope resultは`succeeded | failed`だけで、対象Taskがcancelledへ収束してもCancellation Envelope自身は`succeeded`である。

Packageの前段Receiptは[Executable Contracts](executable-contracts.md#211-既存domain文書から回収した未登録operation候補)の`planning.operation_family.build_candidate_test`をatomic Activationするときに、§9.2の`ProjectValidationReceiptV1`、`CookReceiptV1`、`NativeModuleBuildReceiptV1`、`ProjectShaderBuildReceiptV1`、`GameCandidateBuildReceiptV1`、`CandidateTestReceiptV1`というnamed closed payload／wrapperへ固定する。各Refは§9.2のexact purposeを持つ`MirakanSignedRecordRefV1`で、隣接`*_receipt_sha256`または配列要素の`signed_record_hash`は同じ完成wrapperのRFC 8785 JCS SHA-256へ一致させる。current familyは`not_activated`でMCD／Manifest／Service／Policy／Validator／Diagnostic／Receipt／Signer／Provider／MCP集合がexact `[]`のため、現在はPackage success closureも成立しない。Receipt名、上表Field、文書中のOperation候補をcallableと解釈しない。

`source_build_closure.kind=no_project_source`はbranch外三配列をcanonical omissionし、Game Candidate Compile Manifestのselected Native／Shader source集合がともにexact `[]`の場合だけ許す。`selected_project_source`はPromotion Receiptを1件以上、Native／Shader Build Receiptの少なくとも一方を1件以上必須にし、Compile Manifestが選択した全Source revisionとPromotion／Build Receipt集合をkind、revision、hash、Targetのset equalityで照合する。Nativeだけ、Shaderだけ、両方を許すが、未選択Sourceのextra Receipt、選択Sourceのmissing Receipt、空のselected branchを拒否する。Packageの`project_publication_binding`は常に`committed_revision`で、Sourceを選択した場合は同bindingのlate source-promotion authorization entry集合と`source_build_closure.project_source_promotion_receipt_refs[]`をPromotion Receipt wrapper hashでset equalityにし、Sourceなしの場合はbinding内Ref集合をexact `[]`にする。

成功した`TaskStatusReceiptPayloadV1`のReceipt ref／hashは対象Taskがterminalの場合だけ対で必須、非terminalでは両方省略する。成功した`TaskCancellationReceiptPayloadV1`は`converged_state`の値にかかわらず対象Task自身のterminal Receipt ref／hashを必須にし、Cancellation Receiptを対象TaskのReceiptとして流用しない。対象Async Operationはcancelled時も同じ`task_id`を持つ型固有terminal Receiptを発行する。

完成した各`*ReceiptV1`は`OperationReceiptEnvelopeV1`をsubjectとし、上表のexact `signed_record_purpose`を持つ`MirakanSignedRecordV1`である。Async 11 Operationの`OperationTaskV1.receipt_ref`、前段Receipt ref、`*_receipt_sha256`は署名を含む完成Record全体のcanonical hashを指す。同期Control Receiptは`control_invocation_id`で呼出しを監査し、新しい`OperationTaskV1`またはTask receipt refを作らない。共通署名envelope、hash chain、保持は[AI Verification／Provenance §7](../01-governance/ai-verification-provenance.md#7-evidence-envelope)、algorithm、Signer Role／Key用途、Authorizationは[AI Security／Approval](../01-governance/ai-security-approval.md)を参照し、本書へ署名Fieldや独自Provenanceを複写しない。

Packageはcommitted Project revision `N+1`で実行した最終Validation→Cook→Game Candidate Build→Candidate Testの完成Receiptを署名込みでread-backし、これらとPackage EnvelopeのProject revision `N+1`、Candidate root、Target、Contract set、Toolchain lock、exact `BuildProjectPublicationBindingV1(kind=committed_revision)`を一致させる。bindingの`PublicPublicationMarkerRefV1`は完成`PublicPublicationMarkerV1`へ、Markerとsigned Domain Receiptの同一`PublicCommitClosureRefV1`は完成`PublicCommitClosureV1`へ解決する。Marker／Receipt／ClosureのOperationを`operation.authoring.changeset.commit`、before Projectを`N`、public-after Projectを`N+1`とし、Closureの`domain_commitment.kind=project_change_set_commit`からexact `project_change_set_ref`と`candidate_root_sha256`を読み、ClosureのPrepared Candidate、late source-promotion authorization binding集合とともにbindingへbyte equalityにする。Source registration primitiveが0件でもProject branch、ChangeSet、Candidate rootを必須にし、late binding集合だけを`[]`にする。Marker／Receiptの`public_commit_closure_hash`とRef内`closure_ref.sha256`は完成Closure object SHA、Refの`closure_content_hash`はsemantic hashとして別々に検証し、相互代用しない。private Marker bodyまたはPrepared Envelope bodyの公開lookupをPackage成立条件にせず、ClosureのEnvelope／private Marker commitmentはPublic Marker、signed Receipt、retained internal audit objectのhashと一致させる。Sourceを選択した場合、Promotion Receiptとそれがexact参照するprepromotion Build／Test Receiptだけは即時predecessorであるbase Project revision `N`へ束縛し、Project ID、Candidate root、Target、Toolchain、promoted Source revision、before／after tree、Code Owner Approval、および同late bindingをset equalityで照合する。これらPromotion用Receiptの`project_revision=N`を最終Envelopeの`N+1`へ書き換えず、最終Validation／Cook／Game Candidate／Candidate Test Receiptとして流用しない。Package→Install→Launch→Smokeでは全EnvelopeのProject revision、Candidate root、Targetと、全payloadのPackage artifact ref／hashをexact一致させる。各`authorization_envelope_hash`は当該Operation、同じsubject identity、Project／Candidate／Target／Device closureへ有効でなければならず、前段Authorizationを継承しない。Install／Launch／SmokeではDevice identity／generationも一致させ、SmokeはPackage、Install、Launchの`result=succeeded`完成Receipt ref／hashとfixture ref／hashをすべて必須にする。Resetも`result=succeeded`のPackage Receipt／artifact、Device、consent、R3 Approvalへ閉じる。前段Receiptのmissing、非success、署名／hash／subject差、revocation、Device generation差、fixture差を後段の成功として受理しない。

`TaskStatusReceiptV1`と`TaskReceiptReadReceiptV1`はControl requestと対象Task／Receipt hashを監査する同期read Receipt、`TaskCancellationReceiptV1`はcancel requestと収束結果を監査するControl Receiptであり、いずれも新しい`OperationTaskV1`を作らない。

Activation済みTaskの許可遷移は`queued -> running`、`queued | running -> cancel_requested`、`running -> succeeded | failed`、`cancel_requested -> cancelled | succeeded | failed`だけである。Irreversible boundary通過後のcancelは結果不明にせず、Operationを収束させて`succeeded | failed`とReceiptを返す。Retryは同じcanonical requestなら同じidempotency keyを使い、request hashが異なる場合は新task IDにする。Terminal taskを再実行せず、`operation.task.read_receipt`で結果を読む。

Gatewayはenqueue時と実行直前にProject revision、Project Publication binding、Candidate root、Target Profile、Authorization、consent、Device identity／generation、入力Receiptのsubject／hash／freshnessを再検証する。Packageはtyped `PublicPublicationMarkerRefV1`でcurrent committed headと確認できる`N+1`だけを入力にし、予定revision、prepromotion base `N`、未公開Prepared Candidate、未定義aliasまたは表示名だけのMarkerをPackage revisionとして受理しない。一件でもdriftした場合は副作用開始前に失敗し、stale Candidate、Device交換、Package Receipt差替えを「最新」へ自動追従しない。Package生成、install、launch、reset、smoke、Debug、cancelは別Operationであり、前段のAuthorizationやApprovalを後段へ継承しない。

### 9.2 Build Candidate／Testのtyped execution closure

本節は`planning.operation_family.build_candidate_test`のexact六Operationを将来atomic Activationするための受入契約である。current family stateは`not_activated`であり、六Operationに対応するMCD Operation、Owner Manifest、Service allowlist、Policy、Validator／closure、Diagnostic、Receipt schema／instance、Signer Policy／Role／Key、Provider／MCP／CLI／Editor投影、Task instanceはすべてexact `[]`である。以下の型名、Authority名、purpose、状態遷移は`activation.build.candidate_test_operations.v1`が全六行と全依存を一transactionでmaterializeした後だけ有効で、現時点の実装済み／呼出し可能性を表さない。

#### 9.2.1 共通identity、Input、Task

```text
BuildProjectRevisionRefV1
  project_id: UUIDv7
  project_revision: uint64
  document_set_hash: Sha256DigestV1

BuildProjectPublicationBindingV1
  binding:
    {kind: prepromotion_base}
    | {kind: committed_revision,
       public_publication_marker_ref: PublicPublicationMarkerRefV1,
       public_commit_closure_ref: PublicCommitClosureRefV1,
       before_project_ref: BuildProjectRevisionRefV1,
       after_project_ref: BuildProjectRevisionRefV1,
       project_change_set_ref: ProjectChangeSetArtifactRefV1,
       prepared_candidate_ref: PreparedCandidateRefV1,
       candidate_root_sha256: Sha256DigestV1,
       project_source_promotion_authorization_binding_refs[0..1]:
         ProjectSourcePromotionAuthorizationBindingRefV1}

BuildCandidateRefV1
  prepared_candidate_ref: PreparedCandidateRefV1
  candidate_root_sha256: Sha256DigestV1
  candidate_manifest_ref: ArtifactRefV1(
    artifact_kind=game_candidate_input_manifest, schema_version=1)
  candidate_manifest_sha256: Sha256DigestV1

BuildTargetClosureRefV1
  target_profile_ref: DocumentRef<TargetProfileDocument>
  target_profile_sha256: Sha256DigestV1
  target_readiness_ref: ArtifactRefV1(
    artifact_kind=target_readiness, schema_version=1)
  target_readiness_sha256: Sha256DigestV1

BuildToolchainClosureRefV1
  toolchain_lock_ref: ArtifactRefV1(
    artifact_kind=toolchain_lock, schema_version=1)
  toolchain_lock_sha256: Sha256DigestV1
  build_driver_profile_ref: ArtifactRefV1(
    artifact_kind=build_driver_profile, schema_version=1)
  build_driver_profile_sha256: Sha256DigestV1
  compiler_profile_refs[1..16]: ArtifactRefV1(
    artifact_kind=compiler_profile, schema_version=1)
  closure_sha256: Sha256DigestV1

BuildCandidateRequestBaseV1
  request_id: UUIDv7
  project_ref: BuildProjectRevisionRefV1
  project_publication_binding: BuildProjectPublicationBindingV1
  candidate_ref: BuildCandidateRefV1
  target_ref: BuildTargetClosureRefV1
  toolchain_ref: BuildToolchainClosureRefV1
  contract_set_hash: Sha256DigestV1
  authorization_envelope_hash: Sha256DigestV1
  request_hash_algorithm_binding
  idempotency_key: UUIDv7
  execution_deadline
  request_sha256: Sha256DigestV1

ProjectValidationRequestV1
  common: BuildCandidateRequestBaseV1
  validation_policy_ref: ArtifactRefV1(
    artifact_kind=project_validation_policy, schema_version=1)
  validation_policy_sha256: Sha256DigestV1

CookRequestV1
  common: BuildCandidateRequestBaseV1
  project_validation_receipt_ref: ProjectValidationReceiptRefV1
  cook_profile_ref: ArtifactRefV1(
    artifact_kind=content_cook_profile, schema_version=1)
  cook_profile_sha256: Sha256DigestV1

NativeModuleBuildSourceAuthorizationV1 =
  {kind: prepromotion_candidate,
   source_task_ref: ArtifactRefV1(
     artifact_kind=project_native_source_task, schema_version=1),
   patch_proposal_ref: ArtifactRefV1(
     artifact_kind=project_native_patch_proposal, schema_version=1),
   broker_recomputed_diff_ref: ArtifactRefV1(
     artifact_kind=broker_recomputed_source_diff, schema_version=1),
   code_owner_assignment_ref:
     MirakanSignedRecordRefV1(purpose=code_owner_assignment)}
  | {kind: promoted_revision,
     promotion_receipt_ref:
       MirakanSignedRecordRefV1(purpose=project_source_promotion)}

ProjectShaderBuildSourceAuthorizationV1 =
  {kind: prepromotion_candidate,
   source_task_ref: ArtifactRefV1(
     artifact_kind=project_shader_source_task, schema_version=1),
   patch_proposal_ref: ArtifactRefV1(
     artifact_kind=project_shader_patch_proposal, schema_version=1),
   broker_recomputed_diff_ref: ArtifactRefV1(
     artifact_kind=broker_recomputed_source_diff, schema_version=1),
   code_owner_assignment_ref:
     MirakanSignedRecordRefV1(purpose=code_owner_assignment)}
  | {kind: promoted_revision,
     promotion_receipt_ref:
       MirakanSignedRecordRefV1(purpose=project_source_promotion)}

NativeModuleBuildRequestV1
  common: BuildCandidateRequestBaseV1
  project_validation_receipt_ref: ProjectValidationReceiptRefV1
  source_revision_ref: ArtifactRefV1(
    artifact_kind=project_native_source_revision, schema_version=1)
  source_revision_sha256: Sha256DigestV1
  source_tree_sha256: Sha256DigestV1
  source_authorization: NativeModuleBuildSourceAuthorizationV1
  native_build_test_plan_ref: ArtifactRefV1(
    artifact_kind=native_build_test_plan, schema_version=1)
  native_build_test_plan_sha256: Sha256DigestV1

ProjectShaderBuildRequestV1
  common: BuildCandidateRequestBaseV1
  project_validation_receipt_ref: ProjectValidationReceiptRefV1
  source_revision_ref: ArtifactRefV1(
    artifact_kind=project_shader_source_revision, schema_version=1)
  source_revision_sha256: Sha256DigestV1
  source_tree_sha256: Sha256DigestV1
  source_authorization: ProjectShaderBuildSourceAuthorizationV1
  shader_build_test_plan_ref: ArtifactRefV1(
    artifact_kind=project_shader_build_test_plan, schema_version=1)
  shader_build_test_plan_sha256: Sha256DigestV1

GameCandidateBuildRequestV1
  common: BuildCandidateRequestBaseV1
  project_validation_receipt_ref: ProjectValidationReceiptRefV1
  cook_receipt_ref: CookReceiptRefV1
  compile_manifest_ref: ArtifactRefV1(
    artifact_kind=game_candidate_compile_manifest, schema_version=1)
  compile_manifest_sha256: Sha256DigestV1
  source_build_closure:
    kind: no_project_source
    | kind: prepromotion_project_source
      native_module_build_receipt_refs[0..64]:
        NativeModuleBuildReceiptRefV1
      project_shader_build_receipt_refs[0..64]:
        ProjectShaderBuildReceiptRefV1
    | kind: promoted_project_source
      promotion_receipt_refs[1..64]:
        MirakanSignedRecordRefV1(purpose=project_source_promotion)
      native_module_build_receipt_refs[0..64]:
        NativeModuleBuildReceiptRefV1
      project_shader_build_receipt_refs[0..64]:
        ProjectShaderBuildReceiptRefV1

CandidateTestRequestV1
  common: BuildCandidateRequestBaseV1
  game_candidate_build_receipt_ref: GameCandidateBuildReceiptRefV1
  test_plan_ref: ArtifactRefV1(
    artifact_kind=candidate_test_plan, schema_version=1)
  test_plan_sha256: Sha256DigestV1
  fixture_refs[1..256]: ArtifactRefV1(
    artifact_kind=candidate_test_fixture, schema_version=1)
  fixture_set_sha256: Sha256DigestV1

BuildCandidateOperationTaskV1
  task_id: UUIDv7
  operation_id:
    operation.build.request_validate |
    operation.build.request_cook |
    operation.build.request_native_module |
    operation.build.request_project_shader |
    operation.build.request_game_candidate |
    operation.test.request_run
  input_contract_ref: McdContractRefV1(kind=type)
  request_sha256: Sha256DigestV1
  project_ref: BuildProjectRevisionRefV1
  candidate_ref: BuildCandidateRefV1
  target_ref: BuildTargetClosureRefV1
  toolchain_ref: BuildToolchainClosureRefV1
  authorization_envelope_hash: Sha256DigestV1
  idempotency_key: UUIDv7
  state:
    queued | running | cancel_requested |
    succeeded | failed | cancelled
  cancellation_cause?:
    caller_request | deadline_expired | authorization_revoked |
    executor_shutdown | resource_revoked
  receipt_ref?: ProjectValidationReceiptRefV1 | CookReceiptRefV1 |
    NativeModuleBuildReceiptRefV1 | ProjectShaderBuildReceiptRefV1 |
    GameCandidateBuildReceiptRefV1 | CandidateTestReceiptRefV1
```

全objectはclosed、全Field必須（`?`だけ省略可）、Digestはlowercase hexadecimal exact 64文字である。`BuildProjectRevisionRefV1`は上位仕様の`ProjectSnapshotRefV1`と同じexact `{project_id, project_revision, document_set_hash}`で、別Project、同revision別document set、revisionだけの参照を拒否する。prepromotion Validate／Cook／Source Build／Game Candidate／Candidate Testでは`project_ref`をlive base Project revision `N`、`project_publication_binding.kind=prepromotion_base`へ固定し、`candidate_ref`が予定after revision `N+1`の不変`PreparedCandidateRefV1`／Candidate rootへ解決しても`project_revision=N+1`またはcommittedと表明しない。Project Commit後の最終Validate／Cook／Game Candidate／Candidate Testでは`project_publication_binding.kind=committed_revision`を必須にし、typed `PublicPublicationMarkerRefV1`から完成`PublicPublicationMarkerV1`、bindingとMarkerのtyped `PublicCommitClosureRefV1`から完成`PublicCommitClosureV1`を解決する。Closureの`domain_commitment.kind=project_change_set_commit`を必須にし、同branchの`project_change_set_ref`／`candidate_root_sha256`、ClosureのPrepared Candidate、before／after、late binding集合をbindingへbyte equalityにすることで、private objectを公開lookupせずcurrent committed Project revision `N+1`と`document_set_hash`を得る。bindingの`after_project_ref`と`project_ref`もbyte equalityにする。Source primitive 0件でもProject branchをowner-typed branchへ置換せず、late binding集合だけを空にする。prepromotion ReceiptのProject ref、未定義aliasまたは表示名だけのMarkerを再利用しない。canonical公開解決経路は`PublicPublicationMarkerRefV1 -> PublicPublicationMarkerV1 -> PublicCommitClosureRefV1 -> PublicCommitClosureV1 -> project_change_set_ref + PreparedCandidateRefV1 + candidate_root_sha256`であり、private Marker／Prepared EnvelopeはClosureのtyped hash commitmentとretained internal auditで照合するが、Packageの公開graphからbodyを解決しない。`ArtifactRefV1.sha256`と隣接`*_sha256`は同じ解決済み完成bytes、`DocumentRef<TargetProfileDocument>`のcontent hashと`target_profile_sha256`も同じ完成Target documentへ一致させる。`BuildToolchainClosureRefV1.closure_sha256`はToolchain lock、Driver Profile、compiler Profile集合をcanonical tuple順へ並べ、ASCII `MIRAKAN_BUILD_TOOLCHAIN_CLOSURE_V1`と各完成RefのMCD canonical bytesを`uint32_be` length framingしてSHA-256する。配列はcanonical tupleのunsigned byte順、重複禁止で、empty compiler集合、bare「current／latest」、path、display nameからの再解決を拒否する。

`request_sha256`は`BuildCandidateRequestBaseV1.request_sha256`自身だけを除き、派生Inputに存在する全Fieldを宣言順にMCD canonical encodeし、ASCII `MIRAKAN_BUILD_CANDIDATE_REQUEST_V1`、exact Operation ID、`uint32_be(input_byte_length)`、input bytesを順にSHA-256する。従って同じ共通baseでもOperation、前段Receipt、Source、Plan、Fixtureの一Fieldが異なれば別requestである。`request_hash_algorithm_binding`はExecutable Contractsのcurrent Algorithm Registry／closureへ解決し、独自hash algorithmへ置換しない。

Receiptの`input_closure_sha256`は、派生Inputから`request_id`、`authorization_envelope_hash`、`request_hash_algorithm_binding`、`idempotency_key`、`execution_deadline`、`request_sha256`だけを除いた残りの全Field、すなわち`project_ref`、`project_publication_binding`、`candidate_ref`、`target_ref`、`toolchain_ref`、`contract_set_hash`と当該named Inputの`common`外Fieldすべてを宣言順にMCD canonical encodeしたbytesを`I`とし、`SHA-256(ASCII "MIRAKAN_BUILD_CANDIDATE_INPUT_CLOSURE_V1" || uint32_be(len(exact Operation ID UTF-8 bytes)) || exact Operation ID UTF-8 bytes || uint32_be(len(I)) || I)`で計算する。同じ内容のretryは同じclosure、Authorization／deadlineだけの更新は同じclosureになり、Project／Publication Marker／late binding／Candidate／Target／Toolchain／Source／Plan／Fixture／前段Receiptの一Field差は必ず別closureになる。payload commonの同値はこの再計算値で検証し、caller supplied digestを信頼しない。

Gatewayは同じ`{operation_id, authorization_envelope_hash, idempotency_key}`について、同じ`request_sha256`なら同一`task_id`と既存terminal Receiptを返し、別request hashなら`MIRAKAN-BUILD-IDEMPOTENCY_CONFLICT`で新Taskを作らない。別idempotency keyは別Taskであるが、同一Project／Candidate／Targetに対する重複副作用はcontent-addressed put-if-absentとsingle-flightで一つへ収束させる。Terminal Taskは再実行せず、非終端Taskのduplicate requestは同じTask状態を返す。

許可状態遷移は§9.1と同じで、`queued -> running`、`queued | running -> cancel_requested`、`running -> succeeded | failed`、`cancel_requested -> cancelled | succeeded | failed`だけである。副作用不可逆境界前のcancelは`cancelled`へ収束し、境界後は作業を収束して`succeeded | failed`へする。六familyだけをActivationした状態では外部cancel surfaceはexact `[]`であり、deadline、Authorization失効、executor shutdown、resource revokeだけが内部原因になれる。caller cancelは別familyの`operation.task.cancel`が同一Contract setでoperationalになり、対象Operation allowlistへこの六IDすべてをexact追加した場合だけ受理する。候補名、signal file、process kill、Provider独自cancelをTask state authorityにしない。

#### 9.2.2 Receipt payload、wrapper、Signer mapping

```text
BuildCandidateReceiptCommonV1
  request_sha256: Sha256DigestV1
  project_ref: BuildProjectRevisionRefV1
  project_publication_binding: BuildProjectPublicationBindingV1
  candidate_ref: BuildCandidateRefV1
  target_ref: BuildTargetClosureRefV1
  toolchain_ref: BuildToolchainClosureRefV1
  contract_set_hash: Sha256DigestV1
  input_closure_sha256: Sha256DigestV1
  outcome: succeeded | failed | cancelled
  failure?:
    failure_class:
      invalid_input | authorization_lost | stale_project |
      stale_candidate | target_unavailable | toolchain_mismatch |
      source_closure_mismatch | prerequisite_receipt_invalid |
      executor_failed | resource_limit | deadline_exceeded |
      internal_error
    retry_disposition: retry_same_request | new_request_required | not_retryable
  cancellation?:
    cause:
      caller_request | deadline_expired | authorization_revoked |
      executor_shutdown | resource_revoked
    irreversible_boundary_crossed: false

ProjectValidationReceiptPayloadV1
  common: BuildCandidateReceiptCommonV1
  success?:
    validated_candidate_manifest_ref: ArtifactRefV1(
      artifact_kind=validated_candidate_manifest, schema_version=1)
    validated_candidate_manifest_sha256: Sha256DigestV1
    validation_closure_sha256: Sha256DigestV1

CookReceiptPayloadV1
  common: BuildCandidateReceiptCommonV1
  project_validation_receipt_ref: ProjectValidationReceiptRefV1
  success?:
    cooked_content_manifest_ref: ArtifactRefV1(
      artifact_kind=cooked_content_manifest, schema_version=1)
    cooked_content_manifest_sha256: Sha256DigestV1
    cooked_content_root_sha256: Sha256DigestV1

NativeModuleBuildReceiptPayloadV1
  common: BuildCandidateReceiptCommonV1
  project_validation_receipt_ref: ProjectValidationReceiptRefV1
  source_revision_ref: ArtifactRefV1(
    artifact_kind=project_native_source_revision, schema_version=1)
  source_revision_sha256: Sha256DigestV1
  source_tree_sha256: Sha256DigestV1
  source_authorization: NativeModuleBuildSourceAuthorizationV1
  success?:
    native_build_manifest_ref: ArtifactRefV1(
      artifact_kind=native_module_build_manifest, schema_version=1)
    native_build_manifest_sha256: Sha256DigestV1
    native_artifact_set_sha256: Sha256DigestV1
    native_test_evidence_set_sha256: Sha256DigestV1

ProjectShaderBuildReceiptPayloadV1
  common: BuildCandidateReceiptCommonV1
  project_validation_receipt_ref: ProjectValidationReceiptRefV1
  source_revision_ref: ArtifactRefV1(
    artifact_kind=project_shader_source_revision, schema_version=1)
  source_revision_sha256: Sha256DigestV1
  source_tree_sha256: Sha256DigestV1
  source_authorization: ProjectShaderBuildSourceAuthorizationV1
  success?:
    shader_build_manifest_ref: ArtifactRefV1(
      artifact_kind=project_shader_build_manifest, schema_version=1)
    shader_build_manifest_sha256: Sha256DigestV1
    shader_artifact_set_sha256: Sha256DigestV1
    shader_test_evidence_set_sha256: Sha256DigestV1

GameCandidateBuildReceiptPayloadV1
  common: BuildCandidateReceiptCommonV1
  project_validation_receipt_ref: ProjectValidationReceiptRefV1
  cook_receipt_ref: CookReceiptRefV1
  compile_manifest_ref: ArtifactRefV1(
    artifact_kind=game_candidate_compile_manifest, schema_version=1)
  source_build_closure:
    kind: no_project_source
    | kind: prepromotion_project_source
      native_module_build_receipt_refs[0..64]:
        NativeModuleBuildReceiptRefV1
      project_shader_build_receipt_refs[0..64]:
        ProjectShaderBuildReceiptRefV1
    | kind: promoted_project_source
      promotion_receipt_refs[1..64]:
        MirakanSignedRecordRefV1(purpose=project_source_promotion)
      native_module_build_receipt_refs[0..64]:
        NativeModuleBuildReceiptRefV1
      project_shader_build_receipt_refs[0..64]:
        ProjectShaderBuildReceiptRefV1
  success?:
    game_candidate_manifest_ref: ArtifactRefV1(
      artifact_kind=game_candidate_build_manifest, schema_version=1)
    game_candidate_manifest_sha256: Sha256DigestV1
    game_candidate_artifact_root_sha256: Sha256DigestV1

CandidateTestReceiptPayloadV1
  common: BuildCandidateReceiptCommonV1
  game_candidate_build_receipt_ref: GameCandidateBuildReceiptRefV1
  test_plan_ref: ArtifactRefV1(
    artifact_kind=candidate_test_plan, schema_version=1)
  fixture_refs[1..256]: ArtifactRefV1(
    artifact_kind=candidate_test_fixture, schema_version=1)
  success?:
    test_result_manifest_ref: ArtifactRefV1(
      artifact_kind=candidate_test_result_manifest, schema_version=1)
    test_result_manifest_sha256: Sha256DigestV1
    executed_fixture_set_sha256: Sha256DigestV1
    passed_fixture_set_sha256: Sha256DigestV1

ProjectValidationReceiptRefV1 =
  MirakanSignedRecordRefV1(
    purpose=operation_receipt:operation.build.request_validate)
CookReceiptRefV1 =
  MirakanSignedRecordRefV1(
    purpose=operation_receipt:operation.build.request_cook)
NativeModuleBuildReceiptRefV1 =
  MirakanSignedRecordRefV1(
    purpose=operation_receipt:operation.build.request_native_module)
ProjectShaderBuildReceiptRefV1 =
  MirakanSignedRecordRefV1(
    purpose=operation_receipt:operation.build.request_project_shader)
GameCandidateBuildReceiptRefV1 =
  MirakanSignedRecordRefV1(
    purpose=operation_receipt:operation.build.request_game_candidate)
CandidateTestReceiptRefV1 =
  MirakanSignedRecordRefV1(
    purpose=operation_receipt:operation.test.request_run)
```

`outcome=succeeded`では`success`だけを必須にし`failure/cancellation`を省略する。`failed`では`failure`だけを必須、`success/cancellation`を省略し、Envelopeの`diagnostic_refs[1..64]`を必須にする。`cancelled`では`cancellation`だけを必須、`success/failure`を省略し、同じくDiagnosticを1件以上必須にする。`cancellation.irreversible_boundary_crossed`はexact `false`だけで、境界後にcancel要求を受けてもTask／Receipt outcomeを`cancelled`へ偽装しない。成功artifact Fieldはfailure／cancelled branchからcanonical omissionし、partial artifactを下流へ到達可能化しない。

六完成Receiptは§9.1の`OperationReceiptEnvelopeV1`をsubjectとする`MirakanSignedRecordV1`であり、全て`invocation_kind=async_task`、対応する`BuildCandidateOperationTaskV1.task_id`必須、`control_invocation_id`省略である。EnvelopeのProject revision、Candidate root、Target、request、Authorization、payload hash、resultはpayload commonおよびTaskとbyte equalityにする。`result`と`payload.common.outcome`が異なるRecord、wrong payload contract、wrong purpose、wrong Operation IDは署名が暗号学的に正しくても拒否する。

| operation_id | named input | payload contract | 完成Receipt alias | execution authority = signer subject | signer_role_ref | exact signed purpose |
|---|---|---|---|---|---|---|
| `operation.build.request_validate` | `ProjectValidationRequestV1` | `ProjectValidationReceiptPayloadV1` | `ProjectValidationReceiptV1` | `build_gateway` | `role.operation_receipt.build_request_validate` | `operation_receipt:operation.build.request_validate` |
| `operation.build.request_cook` | `CookRequestV1` | `CookReceiptPayloadV1` | `CookReceiptV1` | `build_gateway` | `role.operation_receipt.build_request_cook` | `operation_receipt:operation.build.request_cook` |
| `operation.build.request_native_module` | `NativeModuleBuildRequestV1` | `NativeModuleBuildReceiptPayloadV1` | `NativeModuleBuildReceiptV1` | `build_gateway` | `role.operation_receipt.build_request_native_module` | `operation_receipt:operation.build.request_native_module` |
| `operation.build.request_project_shader` | `ProjectShaderBuildRequestV1` | `ProjectShaderBuildReceiptPayloadV1` | `ProjectShaderBuildReceiptV1` | `build_gateway` | `role.operation_receipt.build_request_project_shader` | `operation_receipt:operation.build.request_project_shader` |
| `operation.build.request_game_candidate` | `GameCandidateBuildRequestV1` | `GameCandidateBuildReceiptPayloadV1` | `GameCandidateBuildReceiptV1` | `build_gateway` | `role.operation_receipt.build_request_game_candidate` | `operation_receipt:operation.build.request_game_candidate` |
| `operation.test.request_run` | `CandidateTestRequestV1` | `CandidateTestReceiptPayloadV1` | `CandidateTestReceiptV1` | `candidate_test_service` | `role.operation_receipt.test_request_run` | `operation_receipt:operation.test.request_run` |

各aliasは`{subject: OperationReceiptEnvelopeV1, signed_record: MirakanSignedRecordV1(purpose=同一行exact signed purpose)}`だけで、別wrapper schemaを作らない。Activation transactionはこの六行をOperation Registry、Owner Manifest、Service allowlist、Signer Policy、Role assignment、non-exportable singleton-purpose Key、payload mappingへset equalityで投影する。`build_device_play_debug_task`の既存14行は別subsetとしてbyte-exactに保持し、`operation.build.request_package` purpose／Roleやgeneric `operation_receipt`を六行へ流用しない。

#### 9.2.3 前段／後段のset equality

全成功Receiptは、入力Refを署名込みでread-backし、purpose、subject、wrapper hash、Project、Project Publication binding、Candidate、Target、Toolchain、Contract set、request hash、revocationを再検証してから発行する。Receipt commonの`project_publication_binding`はRequestとbyte equalityにし、Project／Marker／late binding／Candidate／Target／Toolchainのどれかを「最新」へ追従せず、一Field差でも新requestを要求する。

- CookのValidation集合はexact singletonで、同じCandidate input manifestを検証した成功`ProjectValidationReceiptV1`と一致する。
- Native／Shader BuildのValidation集合はexact singletonである。`source_authorization`は`kind`をdiscriminatorとするclosed tagged unionで、branch外Fieldをcanonical omissionする。`prepromotion_candidate`はSource Task、Patch Proposal、Broker再計算Diff、`purpose=code_owner_assignment`のAssignmentをInput／resolved Candidate／Build payloadでbyte equalityにし、Task／Proposal／Diffから導出したSource revision／treeだけを隔離Buildできる。このbranchの成功ReceiptはSource PromotionのBuild evidence専用であり、それ自体をPromotion、Project Commit、load、Package authorizationとして扱わない。`promoted_revision`は`purpose=project_source_promotion`の完成ReceiptとSource revision／treeをbyte equalityにし、既にPromotion済みのSourceだけを再Buildできる。prepromotion branchへPromotion Receipt、promoted branchへTask／Proposal／Diff／Assignmentを混在させず、wrong source kind、別Target、別Toolchainの成功Receiptを拒否する。
- Source Promotionは`prepromotion_candidate` branchの成功Build Receiptだけを受理し、同ReceiptのTask／Proposal／Diff／Assignment／Source revision／Target／ToolchainをPromotion subjectとexact一致させる。Promotion Receiptを必須にする`promoted_revision` Build ReceiptをPromotionの前提にしないため、依存順は`Task -> Proposal -> prepromotion Build -> independent review／Code Owner Approval -> Promotion`の一方向である。
- Game CandidateのValidation／Cook集合は各exact singletonである。Compile Manifestが列挙するselected Source revision集合と、`prepromotion_project_source`のNative／Shader Build集合、または`promoted_project_source`のPromotion／Native／Shader Build集合を`{source_kind, source_revision_ref, source_tree_sha256, target_profile_ref, toolchain_lock_sha256}`でset equalityにする。prepromotion branchの全Build Receiptは`source_authorization.kind=prepromotion_candidate`、promoted branchでは各SourceについてBuild Receiptが対応Promotion subjectの前段Build refと同一、または`source_authorization.kind=promoted_revision`で同じPromotion refを持つことを必須にする。Manifestのselected Source集合が空のときだけ`no_project_source`を許し、missing／extra／duplicate／異種Receipt、空のSource branchを拒否する。
- Candidate TestのGame Candidate Build集合はexact singletonである。Test Planのrequired fixture集合とInput `fixture_refs[]`、成功payloadのexecuted fixture集合をset equalityにし、`passed_fixture_set_sha256`は全required fixtureがpassした集合と一致する場合だけ`outcome=succeeded`を許す。Source Promotionが参照できるCandidate Test Receiptは、下位Game Candidate Build Receiptの`source_build_closure.kind=prepromotion_project_source`、同じCandidate root、対象Source revision／Build Receipt集合を持つものだけである。`promoted_project_source`または`no_project_source`のTestをPromotion前Evidenceへ流用しない。
- §9.1 Package successのValidation／Cook／Game Candidate Build集合は各exact singleton、Candidate Test集合はPackage policyが同じGame Candidate Manifestに要求するTest Plan集合とset equalityである。PackageはGame Candidate Build Receiptの`source_build_closure.kind=promoted_project_source`だけを`PackageReceiptPayloadV1.source_build_closure.kind=selected_project_source`へ投影でき、Promotion ref／Build ref集合をbyte equalityにする。`prepromotion_project_source`のGame Candidate／Test成功をPackage authorizationへ流用せず、前段六family以外のself-declared artifactやunsigned hashをReceiptとして数えない。

下流開始前とReceipt発行直前の二回、全前段wrapper、current revocation、対象stageのProject revision head、typed Project Publication binding、Candidate root、Target closure、Toolchain closureをread-backする。prepromotion closureは`kind=prepromotion_base`、live base `N`のValidate／Cook／Source Build／Game Candidate／Candidate TestからPromotionまで、final closureは`kind=committed_revision`、exact `PublicPublicationMarkerRefV1`後のcurrent committed `N+1`のValidate→Cook→Game Candidate→Candidate Test→Packageまでであり、二つを一つのsame-revision chainへ合成しない。Promotion Receipt→late authorization binding→Project Commit Public Markerは両closureを結ぶ唯一の境界で、Promotion Receiptが参照するprepromotion Build／Test Receiptは`N`のまま保持する。drift時はterminal `failed`とtyped Diagnosticを発行し、成功artifactをpublishしない。各OperationのAuthorizationは独立であり前段Authorizationを後段へ継承しない。

## 10. Repository境界

Engine repositoryの正規rootを次に固定する。各Directoryの命名grammarとGame Project rootは[Naming／Project layout](naming-project-layout.md)が所有する。

```text
/
├─ CMakeLists.txt
├─ CMakePresets.json
├─ toolchain.lock.json
├─ store_policy.lock.json
├─ vcpkg.json
├─ cmake/
├─ config/
├─ schemas/mirakan/
├─ evidence/
├─ formal/tla/
├─ docs/
├─ engine/{foundation,math,platform,world,assets,jobs,runtime,rendering,physics,navigation,animation,audio,input,content,ui,gameplay,vfx}/
├─ authoring/{model,changes,validation,assets,build}/
├─ editor/{app,ui,shell,docking,panels,workspaces,semantics,recovery}/
├─ orchestrator/
├─ integrations/
├─ tools/
├─ templates/
├─ hosts/{editor_host,game_host,worker_host}/
├─ tests/{contracts,integration,conformance,security,performance,fixtures}/
├─ evals/
├─ samples/
└─ third_party/{ports,patches,notices}/
```

`engine/runtime/contracts`はDomain実装を持たないcommand／event／snapshotだけを所有する。`engine/runtime/orchestration`はphase順序、`engine/runtime/package`はversioned binary manifestとloader、`engine/runtime/compiler`はCommit済みAuthoring revisionからのpackage生成を所有する。

各C++ componentの標準形は次とする。

```text
<component>/
├─ CMakeLists.txt
├─ include/mirakan/<component>/
├─ modules/
├─ source/
├─ tests/
└─ benchmarks/
```

`benchmarks/`はhot pathを持つcomponentだけ必須とする。Generated source、object、cache、downloaded dependencyをsource treeへ置かない。`common`、`misc`、`shared`、巨大な`utils` Directoryを作らない。外部sourceを`third_party/`へ手動copyせず、patchとnoticeだけを追跡する。

## 11. Serialization、Schema、Artifact

- 共有Contractは`schemas/mirakan/`のMCDを正本とし、C++、TypeScript、MCP、Provider、Cooked binaryを生成する。詳細は[Executable contracts](executable-contracts.md)に従う。
- Runtimeはversion不明、hash不一致、Target不一致、未知必須FieldのArtifactをfail-closedで拒否する。
- Generated source、Provider Schema、Reference docsはBuild treeへ生成し、正本とgolden hashだけを追跡する。
- `evidence/`は外部claim、URL、hash、取得日、期限を保持し、Web page全文やBuild中のfloating取得を保存しない。
- `formal/tla/`は有限State machineだけを対象とし、C++実装全体を証明済みと表明しない。

## 12. Test、CI、AI生成物

全ComponentはUnit testを持ち、Port／Adapter境界はconformance test、永続Artifactはgolden／migration test、Concurrencyはstress／cancel test、hot pathはBenchmarkを持つ。CIはTarget Profile、toolchain lock、configuration別にBuild treeを分離し、clean build、incremental build、sanitizer、contract lint、package inspectionを実行する。

sanitizerは全sanitizerを全Targetで実行する意味ではなく、Toolchain Ownerが固定する該当lane matrixである。Windows MSVC laneはASan、portable Linux Clang laneはASan＋UBSan jobと独立TSan jobを要求する。Concurrency stress／cancel testは全必須Targetで常時実行し、対応laneではsanitizerを重ねる。必須laneの未実行はGate失敗とし、Toolchainが非対応のsanitizerをpass、skip成功、または別sanitizerのReceiptで代用しない。Profile ID、compiler version、sanitizer option、除外理由は[Toolchain／dependencies](toolchain-dependencies.md)とVerification Receiptへ固定する。

AI生成のSchema、C++、Build変更も人間作成物と同じlint、compile、test、Authorization、Reviewを通す。AIは未知Fieldの受理、Validation迂回、外部Dependency追加、Toolchain更新、source treeへのGenerated file配置を行えない。

Observabilityは[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、performance telemetryとregression thresholdは[Runtime performance／capacity](../04-runtime/performance-capacity.md)が所有する。FoundationはEvidenceへの入力を生成するだけで、数値を再定義しない。

## 13. Feature実装開始Gate

Feature実装は次を満たすまで開始しない。

1. Product scope、Capability maturity、Owner文書が確定している。
2. Target Profile、Frontend Profile、Build Driver、Toolchain lockが一意に解決する。
3. Foundation、Math、Memory、Executable Contractの最低限Targetがbuildとtestに合格する。
4. Project state変更が唯一のtyped Gatewayを通り、直接File mutationのnegative testがある。
5. External dependencyのlicense、source、artifact hash、SBOM、Adapter boundaryが検証済みである。
6. clean／incremental／cancel recovery、sanitizer、package inspectionのReceiptがある。
7. AI OperationにAuthorization、bounded input／output、stable Diagnostic、Evidenceがある。

Gate不合格時は別Backend、旧Path、legacy Schema、未固定toolへ暗黙fallbackしない。

## 14. 明示的に採用しないもの

- Entityごとのheap objectとvirtual update
- Service Locatorとglobal mutable singleton
- Vendor型を公開API、serialization、Game contractへ露出すること
- Domain間の直接呼出しと相互pointer保持
- Runtime内legacy schema分岐と互換stub
- 未固定dependency、floating download、preview artifactのShipping採用
- Profile結果のないdata-oriented化、pool化、lock-free化
- callback、job、event配送中の再入的なWorld構造変更
- Asset dependency closureの一部だけをlive化するhot reload

例外は再現可能なBenchmark、代替案、破棄条件、影響範囲を持つADRで承認する。
