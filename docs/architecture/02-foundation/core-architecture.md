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

Operation Registryに列挙する各Receiptは、少なくともoperation ID、対象task ID、request hash、Project revision、Candidate root、Target、optional Device identity／generation、結果hash、Diagnostic refsを同じ値で束縛する。`TaskStatusReceiptV1`と`TaskReceiptReadReceiptV1`はControl requestと対象Task／Receipt hashを監査する同期read Receipt、`TaskCancellationReceiptV1`はcancel requestと収束結果を監査するControl Receiptであり、いずれも新しい`OperationTaskV1`を作らない。

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
