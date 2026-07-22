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
- AI Orchestratorは別Processとし、EngineのmemoryやProject fileを直接変更しない。型付きIPC Operationは[Executable contracts](executable-contracts.md)、Authorizationは[AI Security／Approval](../01-governance/ai-security-approval.md)に従う。

## 4. Authoring状態とRuntime状態

Authoringの正規状態はrevision付きProject modelであり、RuntimeはCommit済みrevisionから生成したimmutable packageとephemeral stateを使う。Editor objectをRuntimeへ渡さず、Runtime objectをProject sourceへserializeしない。

状態変更経路は次の一つに閉じる。

```text
Intent / UI / AI proposal
  -> typed operation
  -> validation and authorization
  -> Project ChangeSet commit
  -> offline compile / cook
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

Runtimeのtick、phase DAG、lifetimeは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、共通Budget、capacity、backpressureは[Runtime performance／capacity](../04-runtime/performance-capacity.md)が所有する。ここでは数値を再掲しない。

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

Editor、AI、CLI、CIはallowlistされたBuild Gateway Operationだけを呼ぶ。AI向けBuild系OperationのMCD登録、Risk、Provider projection可否は[Executable contracts](executable-contracts.md)が所有し、実行semantics、task順序、Receipt構造は本節のBuild Gatewayと各Owner文書が所有する。Generator出力や内部databaseを解析・書換えず、CMake File APIとEngine-owned Receiptを読む。Schema／generated Header／Moduleのcodegen edgeは全Input、Output、Byproduct、Depfile、working directoryを宣言する。Build executorの成功だけでPackage成功やPromotion可能と判定しない。

### 9.1 `OperationTaskV1`

Build Gatewayは[Executable contracts](executable-contracts.md#20-ai向けdiscovery)のPackage／Device／Play／Debug Operationを同じclosed task envelopeで実行する。`operation.task.status`、`operation.task.read_receipt`、`operation.task.cancel`は既存Taskを対象にする同期Control Operationであり、入れ子のTaskを新規作成しない。

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

Operation Registryに列挙する全14 Receiptは、次の共通subjectとOperation固有payloadを一つの署名済みRecordへ閉じる。

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

Package／Device／Play／Debugの11 Operationは`invocation_kind=async_task`、対応する`OperationTaskV1.task_id`を必須とし、`control_invocation_id`を省略する。`operation.task.status | operation.task.read_receipt | operation.task.cancel`は`invocation_kind=synchronous_control`、各同期呼出しに一意な`control_invocation_id`を必須とし、`task_id`を省略する。対象Task identityは型固有payloadの`target_task_id`だけが持つ。discriminatorとOperation IDの不一致、両IDのmissing、両方present、control Operationによる対象Task IDのEnvelope流用をschema negative fixtureで拒否する。

| operation_id | payload contract | 完成Receipt alias |
|---|---|---|
| `operation.build.request_package` | `PackageReceiptPayloadV1` | `PackageReceiptV1` |
| `operation.device.install` | `DeviceInstallReceiptPayloadV1` | `DeviceInstallReceiptV1` |
| `operation.device.launch` | `DeviceLaunchReceiptPayloadV1` | `DeviceLaunchReceiptV1` |
| `operation.device.reset_data` | `DeviceDataResetReceiptPayloadV1` | `DeviceDataResetReceiptV1` |
| `operation.play.run_smoke` | `SmokeRunReceiptPayloadV1` | `SmokeRunReceiptV1` |
| `operation.debug.aggregate` | `DebugAggregateReceiptPayloadV1` | `DebugAggregateReceiptV1` |
| `operation.debug.query` | `DebugQueryReceiptPayloadV1` | `DebugQueryReceiptV1` |
| `operation.debug.read_causality` | `DebugCausalityReceiptPayloadV1` | `DebugCausalityReceiptV1` |
| `operation.debug.read_replay_slice` | `ReplaySliceReceiptPayloadV1` | `ReplaySliceReceiptV1` |
| `operation.debug.validate_finding` | `DebugFindingValidationReceiptPayloadV1` | `DebugFindingValidationReceiptV1` |
| `operation.debug.support-bundle.generate` | `SupportBundleReceiptPayloadV1` | `SupportBundleReceiptV1` |
| `operation.task.status` | `TaskStatusReceiptPayloadV1` | `TaskStatusReceiptV1` |
| `operation.task.read_receipt` | `TaskReceiptReadReceiptPayloadV1` | `TaskReceiptReadReceiptV1` |
| `operation.task.cancel` | `TaskCancellationReceiptPayloadV1` | `TaskCancellationReceiptV1` |

Debug系6 payloadは[Debugging／observability／replay §13](../04-runtime/debugging-observability-replay.md#13-ai-debug-contextとdiagnosis)が所有する。Build Gateway／Device／Play／Task Control payloadは次のFieldだけを持つ。

```text
PackageReceiptPayloadV1
  contract_set_hash
  toolchain_lock_hash
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

全Fieldは`?`を付けたFieldを除き必須で、配列は重複なしunsigned byte順、unknown Fieldは禁止する。Envelopeの`result=succeeded`では各payloadのsuccess output groupをすべて必須にする。Packageのartifact ref／hash／manifest hash、Installのtransaction hash、Launchのprocess instance、Resetのtransaction hash、Smokeのsession ref／result hash、Statusのsnapshot sequence／target state、Readのtarget terminal state、Cancellationのstate-before／converged state／target terminal Receipt ref／hashが各success output groupである。`result=failed | cancelled`ではそのgroupを全て省略し、`diagnostic_refs[]`を1件以上必須にする。同期ControlのEnvelope resultは`succeeded | failed`だけで、対象Taskがcancelledへ収束してもCancellation Envelope自身は`succeeded`である。

成功した`TaskStatusReceiptPayloadV1`のReceipt ref／hashは対象Taskがterminalの場合だけ対で必須、非terminalでは両方省略する。成功した`TaskCancellationReceiptPayloadV1`は`converged_state`の値にかかわらず対象Task自身のterminal Receipt ref／hashを必須にし、Cancellation Receiptを対象TaskのReceiptとして流用しない。対象Async Operationはcancelled時も同じ`task_id`を持つ型固有terminal Receiptを発行する。

完成した各`*ReceiptV1`は`OperationReceiptEnvelopeV1`をsubjectとする`MirakanSignedRecordV1`である。Async 11 Operationの`OperationTaskV1.receipt_ref`、前段Receipt ref、`*_receipt_sha256`は署名を含む完成Record全体のcanonical hashを指す。同期Control Receiptは`control_invocation_id`で呼出しを監査し、新しい`OperationTaskV1`またはTask receipt refを作らない。共通署名envelope、hash chain、保持は[AI Verification／Provenance §7](../01-governance/ai-verification-provenance.md#7-evidence-envelope)、algorithm、Key用途、Authorizationは[AI Security／Approval](../01-governance/ai-security-approval.md)を参照し、本書へ署名Fieldや独自Provenanceを複写しない。

Package→Install→Launch→Smokeでは全EnvelopeのProject revision、Candidate root、Targetと、全payloadのPackage artifact ref／hashをexact一致させる。各`authorization_envelope_hash`は当該Operation、同じsubject identity、Project／Candidate／Target／Device closureへ有効でなければならず、前段Authorizationを継承しない。Install／Launch／SmokeではDevice identity／generationも一致させ、SmokeはPackage、Install、Launchの`result=succeeded`完成Receipt ref／hashとfixture ref／hashをすべて必須にする。Resetも`result=succeeded`のPackage Receipt／artifact、Device、consent、R3 Approvalへ閉じる。前段Receiptのmissing、非success、署名／hash／subject差、revocation、Device generation差、fixture差を後段の成功として受理しない。

`TaskStatusReceiptV1`と`TaskReceiptReadReceiptV1`はControl requestと対象Task／Receipt hashを監査する同期read Receipt、`TaskCancellationReceiptV1`はcancel requestと収束結果を監査するControl Receiptであり、いずれも新しい`OperationTaskV1`を作らない。

許可遷移は`queued -> running`、`queued | running -> cancel_requested`、`running -> succeeded | failed`、`cancel_requested -> cancelled | succeeded | failed`だけである。Irreversible boundary通過後のcancelは結果不明にせず、Operationを収束させて`succeeded | failed`とReceiptを返す。Retryは同じcanonical requestなら同じidempotency keyを使い、request hashが異なる場合は新task IDにする。Terminal taskを再実行せず、`operation.task.read_receipt`で結果を読む。

Gatewayはenqueue時と実行直前にProject revision、Candidate root、Target Profile、Authorization、consent、Device identity／generation、入力Receiptのsubject／hash／freshnessを再検証する。一件でもdriftした場合は副作用開始前に失敗し、stale Candidate、Device交換、Package Receipt差替えを「最新」へ自動追従しない。Package生成、install、launch、reset、smoke、Debug、cancelは別Operationであり、前段のAuthorizationやApprovalを後段へ継承しない。

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
