# Miraikanai Engine Developer Testing Contract

- 文書ID: mirakan.arch.developer-testing
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Game Project Developer向けtest suite／case／assertion／parameter／fixture input、GUI／CLI／headless runner、filter／shard／retry、isolation、result、report、public C++ test API、Project test release binding
- 非正本範囲: Engine内部Unit／CI実装、AI Eval／Evidence envelope、Domain qualificationの意味、performance budget、Debug capture Schema、build／package semantics、Platform device farm。各Ownerを参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Project State](project-state.md)、[Native Game Module](native-game-module.md)
- 関連文書: [Product Plan](../00-product/product-plan.md)、[Product Lifecycle](../00-product/product-lifecycle.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core Architecture](../02-foundation/core-architecture.md)、[Native Game Module](native-game-module.md)、[Editor Workspace／UX](editor-workspace-ux.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Persistence／Save](../04-runtime/persistence-save.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[Debugging／Observability／Replay](../04-runtime/debugging-observability-replay.md)
- 根拠区分: project-decision（外部test frameworkの仕様を引用する箇所はofficial-spec、未計測timeout／capacityはprovisional）
- 外部根拠確認日: none

## 1. 結論と所有境界

第三者Developerが自分のGame Projectをrelease前に自動検証できることは、Engine内部testまたはReference Game qualificationで代用できない。MiraikanaiはProject sourceとしてversion管理できるpublic test contractと、Editor GUI、CLI、headlessから同じ意味で実行できるrunnerを製品surfaceとして持つ。

Project testは本番Game Runtimeのauthorityを持たず、隔離されたtest execution contextで公開API、structured Gameplay、World、Input、Save、Render、Audioのobservable contractを検証する。test専用native backdoor、private Engine型、raw pointer、GPU／Physics handleへのaccessを公開しない。

[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)はEvidenceの署名、集約、freshness、retry／quarantine policyを所有する。本書はGame Developerが書くtestの意味とrunner resultを所有し、Project testのpassだけからEngine qualification、Capability activationまたはProduct releaseを発行しない。

対応するC++ API、Schema、runner、Editor panel、reporter、Fixture、ReceiptはRepositoryに存在せず、すべて未materializeである。

## 2. Test taxonomy

| Kind | 対象 | 許される観測 | Productでの用途 |
|---|---|---|---|
| `unit` | pure Project C++／data transform | input／output、Project-owned state | 高速local feedback |
| `contract` | 公開Engine APIとProject contract | typed result、diagnostic、state projection | SDK compatibility |
| `integration` | 複数SubsystemとProject state | bounded World／Service projection | Project feature closure |
| `play_scenario` | Runtime EntryからのGame journey | action、state、event、Save／Load | playable regression |
| `visual` | declared render output | exact Target／profile／tolerance付きimage evidence | presentation regression |
| `performance` | declared workloadとmarker | duration、memory、counter、budget ref | Project baseline regression |
| `package_smoke` | cooked／packaged candidate | install、launch、offline journey、exit | distribution acceptance |

Taxonomyはtest名、directoryまたはtagから推測しない。各caseがkind、required environment、isolation、timeout policy、determinism profile、assertion setを明示する。

Visualとperformanceのpass criteriaはRendering／Performance Ownerのmeasure semanticsを参照し、本書がpixel tolerance、frame budget、hardware classを一律固定しない。package smokeはProduct LifecycleとPlatform Ownerのcandidateを消費し、runnerが独自Packageを生成しない。

## 3. Public test model

| 型 | ASCII domain separator |
|---|---|
| `ProjectTestSuiteV1` | `MIRAKAN_PROJECT_TEST_SUITE_V1` |
| `ProjectTestCaseV1` | `MIRAKAN_PROJECT_TEST_CASE_V1` |
| `ProjectTestCatalogV1` | `MIRAKAN_PROJECT_TEST_CATALOG_V1` |
| `ProjectTestRequirementSetV1` | `MIRAKAN_PROJECT_TEST_REQUIREMENT_SET_V1` |
| `ProjectTestRunRequestV1` | `MIRAKAN_PROJECT_TEST_RUN_REQUEST_V1` |
| `ProjectTestSelectionClosureV1` | `MIRAKAN_PROJECT_TEST_SELECTION_CLOSURE_V1` |
| `ProjectTestCaseAttemptResultV1` | `MIRAKAN_PROJECT_TEST_CASE_ATTEMPT_RESULT_V1` |
| `ProjectTestRunResultV1` | `MIRAKAN_PROJECT_TEST_RUN_RESULT_V1` |

```text
ProjectTestSuiteV1
  suite_id: ProjectStableId
  suite_version: positive u32
  display_name: localized presentation ref
  case_refs[1..4096]: sorted unique exact ProjectTestCaseRefV1
  default_tag_refs[0..64]: sorted unique ProjectTestTagRefV1
  suite_content_hash: SHA-256

ProjectTestCaseV1
  case_id: ProjectStableId
  case_version: positive u32
  test_kind: ProjectTestKind
  source_location_ref: exact ProjectSourceLocationRefV1
  required_capability_refs[0..64]: sorted unique exact CapabilityDefinitionRefV1
  required_target_profile_refs[0..16]: sorted unique exact TargetProfileRefV1
  input_binding_refs[0..256]: sorted unique exact ProjectTestInputBindingRefV1
  parameter_set_ref: optional exact ProjectTestParameterSetRefV1
  isolation_profile_ref: exact ProjectTestIsolationProfileRefV1
  timeout_policy_ref: exact ProjectTestTimeoutPolicyRefV1
  determinism_profile_ref: exact ProjectTestDeterminismProfileRefV1
  assertion_definition_refs[1..1024]:
    sorted unique exact ProjectAssertionDefinitionRefV1
  tag_refs[0..64]: sorted unique ProjectTestTagRefV1
  test_case_content_hash: SHA-256

ProjectTestCatalogV1
  project_test_catalog_id: ProjectStableId
  project_test_catalog_version: 1
  suite_refs[1..4096]:
    sorted unique exact ProjectTestSuiteRefV1
  project_test_catalog_content_hash: SHA-256

ProjectTestRequirementSetV1
  project_test_requirement_set_id: StableId
  project_test_requirement_set_version: 1
  project_test_catalog_ref: exact ProjectTestCatalogRefV1
  target_requirements[1..64]:
    sorted unique {
      target_profile_ref: exact TargetProfileRefV1,
      case_requirements[1..65536]:
        sorted unique {
          suite_ref: exact ProjectTestSuiteRefV1,
          case_ref: exact ProjectTestCaseRefV1,
          parameter_identity:
            {kind=none}
            | {
                kind=expanded,
                parameter_set_ref: exact ProjectTestParameterSetRefV1,
                parameter_value_content_hash: SHA-256
              },
          test_kind: ProjectTestKind,
          expected_result_branch: passed
        }
    }
  project_test_requirement_set_content_hash: SHA-256
```

SuiteとcaseはProject sourceの一部としてProject State Ownerのrevisionへ束縛する。Editor-only mutable database、User preference、recent result cacheを正本にしない。case IDはrename／moveで変わらず、同じIDの意味変更はversionとcontent hashを変える。

本書Ownerのexact Refは次のclosed tupleで、解決先全Fieldとbyte equalityにする。

| Ref | Field |
|---|---|
| `ProjectTestSuiteRefV1` | `{suite_id, suite_version, suite_content_hash}` |
| `ProjectTestCaseRefV1` | `{case_id, case_version, test_case_content_hash}` |
| `ProjectTestCatalogRefV1` | `{project_test_catalog_id, project_test_catalog_version=1, project_test_catalog_content_hash}` |
| `ProjectTestRequirementSetRefV1` | `{project_test_requirement_set_id, project_test_requirement_set_version=1, project_test_requirement_set_content_hash}` |
| `ProjectTestRunRequestRefV1` | `{test_run_request_id, test_run_request_version=1, request_content_hash}` |
| `ProjectTestSelectionClosureRefV1` | `{selection_closure_id, selection_closure_version=1, selection_closure_content_hash}` |
| `ProjectTestCaseAttemptResultRefV1` | `{case_attempt_result_id, case_attempt_result_version=1, case_attempt_result_content_hash}` |
| `ProjectTestRunResultRefV1` | `{test_run_result_id, test_run_result_version=1, result_content_hash}` |

CatalogはProject Source内の全release-required suite refのreceipt-free exact setであり、test結果、runner、CandidateまたはProduct releaseを含めない。全recordのcontent hashは上表の型固有domain separatorと、自己hashだけを除く全FieldのMCD canonical bytesを各`uint32_be` length framingした列からSHA-256する。ID-only、display name、source path、`latest`、同ID／version別hashまたは別Project revisionのRefへfallbackしない。

`ProjectTestRequirementSetV1(catalog, targets)` Named Algorithmは、Catalogの全Suiteを解決し、各Case、bounded Parameter展開、CaseのTarget applicabilityを各入力Targetへjoinしたfull `{Suite,Case,Parameter identity,Target,test kind,expected=passed}`集合を生成する。`required_target_profile_refs[]`がemptyのCaseは全入力Target、non-emptyのCaseはそのexact setとのintersectionだけへ展開する。各TargetのCase requirementは65,536件以下でなければならず、overflowまたは超過はRequirement Set生成を拒否する。Suite／Case／Parameter／Targetの欠落、表示名やtagによるcollapse、empty selectionへの変換、Case一件による別Case充足を禁止する。Requirement SetはCatalog／Target入力だけを参照するdetached receipt-free recordで、CatalogまたはProject SourceからRequirement Set／Resultを逆参照しない。

Parameter setはbounded scalar、enum、Project stable ref、approved Asset／World／Save input refだけを持つ。native pointer、filesystem wildcard、environment secret、arbitrary command、network URLをparameterにしない。組合せ展開には最大case数と拒否diagnosticを持たせ、unbounded Cartesian expansionを禁止する。

## 4. Assertion contract

Public assertionはtyped subjectとfailure diagnosticを持つ。

| Assertion family | 例 | 禁止する代用 |
|---|---|---|
| Value | equality、range、approximation、ordering | locale依存text比較をcanonical value比較にしない |
| State | Entity／Component／Gameplay／UI state projection | private storage layoutやpointerを比較しない |
| Event | event presence、count、order、causal relation | log text substringをevent contractにしない |
| Diagnostic | exact code、severity、subject ref | message文言だけをstable identityにしない |
| Save／Replay | semantic digest、round-trip、reconstruction | raw address、compression bytes、timestampを意味にしない |
| Visual | image artifact、mask、color-space、tolerance profile | 別GPU／profile baselineへ暗黙fallbackしない |
| Performance | marker、counter、sample window、budget profile | 単発wall-clockやEditor UI値をrelease evidenceにしない |
| Fault | injected failure、expected recovery、last-valid state | crashしなかったことだけをrecovery passにしない |

Assertion failureはcase、parameter、subject、expected、observed、source location、diagnostic codeを保持する。secret、raw Project content、large binaryをmessageへ埋めず、access-controlled artifact refとして分離する。

Expected valueの自動更新、golden image再生成、snapshot acceptはProject mutationであり、通常test runと同時にCommitしない。explicit ChangeSet、Diff、Approvalを必要とし、失敗時のobserved outputをexpectedへ自動昇格しない。

## 5. Execution requestとsurface parity

```text
ProjectTestRunRequestV1
  test_run_request_id: StableId
  test_run_request_version: 1
  project_test_catalog_ref: exact ProjectTestCatalogRefV1
  project_revision_ref: exact ProjectRevisionRefV1
  engine_candidate_ref: exact PreparedCandidateRefV1
  public_contract_set_ref: exact PublicContractSetRefV1
  selection_basis:
    {kind=all_catalog_cases}
    | {
        kind=explicit_scope,
        suite_or_case_refs[1..4096]:
          sorted unique
            {kind=suite, suite_ref: exact ProjectTestSuiteRefV1}
            | {kind=case, case_ref: exact ProjectTestCaseRefV1}
      }
  target_profile_ref:
    exact TargetProfileRefV1(profile_kind=runtime_target)
  runtime_entry_ref: optional exact RuntimeEntryRefV1
  filter_expression_ref: optional exact ProjectTestFilterRefV1
  shard_spec_ref: optional exact ProjectTestShardSpecRefV1
  retry_policy_ref: exact ProjectTestRetryPolicyRefV1
  runner_artifact_ref: exact ArtifactRefV1
  execution_environment_ref: exact TestExecutionEnvironmentRefV1
  artifact_output_policy_ref: exact TestArtifactOutputPolicyRefV1
  request_content_hash: SHA-256

ProjectTestSelectionClosureV1
  selection_closure_id: StableId
  selection_closure_version: 1
  request_ref: exact ProjectTestRunRequestRefV1
  case_dispositions[1..65536]:
    sorted unique {
      suite_ref: exact ProjectTestSuiteRefV1,
      case_ref: exact ProjectTestCaseRefV1,
      parameter_identity:
        {kind=none}
        | {
            kind=expanded,
            parameter_set_ref: exact ProjectTestParameterSetRefV1,
            parameter_value_content_hash: SHA-256
          },
      target_profile_ref:
        exact TargetProfileRefV1(profile_kind=runtime_target),
      test_kind: ProjectTestKind,
      disposition:
        {kind=selected, assigned_shard_index: non-negative u32}
        | {
            kind=excluded_by_explicit_scope,
            exclusion_basis_ref: exact EvidenceRefV1
          }
        | {
            kind=filtered_out,
            filter_expression_ref: exact ProjectTestFilterRefV1
          }
        | {
            kind=quarantined,
            quarantine_evidence_ref: exact EvidenceRefV1
          }
        | {
            kind=unsupported_environment,
            unsupported_evidence_ref: exact EvidenceRefV1
          }
    }
  selection_algorithm_id: project_test_selection_closure
  selection_algorithm_version: 1
  selection_algorithm_content_hash: SHA-256
  selection_closure_content_hash: SHA-256
```

Editor GUI、CLI、headlessは同じrequestを構築し、同じrunner、case discovery、filter、shard assignment、retry、result semanticsを使う。GUIは選択、progress、result navigationを提供できるが、Editorだけが利用できるtest kindまたはhidden pass条件を持たない。

CLI／headlessはinteractive prompt、message box、display server、Editor layout、User profileへ依存しない。必要なTarget capabilityがない場合は`unsupported_environment`としてtyped errorを返し、passまたはskipへ暗黙変換しない。

Selection ClosureのNamed Algorithm v1はCatalogの全Suite／Case／bounded Parameterをrequest Targetへ展開したfull `{Suite,Case,Parameter,Target,test kind}` universeを生成し、各identityへexactly one dispositionを割り当てる。`selection_basis`、filter、environment support、quarantine、deterministic shard assignmentの順をAlgorithm content hashへ固定し、disposition projectionをfull universeとset equalityにする。selected rowのshard indexはShard spec内で、同一identityはexactly one shardだけへ属する。explicit scope外、filter除外、quarantine、unsupportedをmissing rowまたはselectedへ変換しない。

Filterはexact ID、tag、kind、source scope、required capabilityをboolean expressionで組み合わせる。表示名substring、filesystem enumeration order、locale、previous result cacheからselectionを変えない。空selected集合は`passed`でなくtyped selection errorである。selection closureとcontent hashをresultへ保存する。

## 6. Isolation、determinism、time

Isolation profileは最低限、`process`、`runtime_session`、`world_instance`、`case_state`の境界を明示する。case間でmutable singleton、random state、Save slot、temp directory、network queue、GPU history、audio voice、clockを暗黙共有しない。

各caseはdeterminism expectationを宣言する。

- `strict_semantic`: same inputとTarget profileでsemantic digestが一致する。
- `bounded_tolerance`: numeric／visual／timing tolerance profile内で一致する。
- `observable_invariant`: Backend固有orderingを固定せず公開invariantだけを検証する。
- `nondeterministic_sampled`: sample countとstatistical ruleをPerformance Ownerへ束縛する。

seedを指定できるrandom sourceはrunnerがcase／parameterごとに導出しresultへ記録する。OS time、frame presentation、worker order、hardware counterを暗黙seedにしない。virtual timeを利用するcaseはScheduling Ownerのtime contractを参照し、real timeoutとsimulation timeoutを分離する。

timeoutはcase failure、runner infrastructure failure、User cancellationを区別する。termination後にProcess、file lock、device session、temp artifactを残さず、cleanup failureを元のfailureへ隠さない。

## 7. Retry、flaky、quarantine

通常caseのdefault retryは0とする。retryは新しいattemptを追加するだけで最初のfailureを消さない。`pass_after_retry`をpassへ畳み込まず、flaky evidenceとして扱う。

Quarantineはcaseをrelease Gateから消す機能ではない。Owner、reason、issue ref、開始時、期限、対象Target、代替coverageを必要とし、required scenarioのquarantine中はRelease Gateを満たさない。期限切れquarantineはrunner errorにする。

Infrastructure retryはcase retryと分離し、device lost、worker unavailable、artifact upload failure等のtyped causeだけを対象にする。assertion failure、crash、timeout、data corruptionをinfrastructure failureへ再分類しない。

## 8. Shardingとparallel execution

Shard assignmentはselection closure、case stable ID、parameter identity、shard count、algorithm versionから決定論的に導出する。同じcase／parameterを複数shardへ割り当てず、全選択集合をexactly onceで覆う。

parallel-safe宣言がないcaseは直列isolationへ置く。global GPU／audio／port／device／license seat／Project write access等のresource requirementはleaseとして宣言し、待機、timeout、deadlock diagnosticを持つ。Project Sourceへのmutation、golden accept、package publicationをparallel testへ許可しない。

shardごとのresultを集約する時はrequest、selection、Project revision、Engine candidate、public contract set、Target、environment、runner versionのexact equalityを要求する。別runの不足caseを合成しない。

## 9. Resultとreport

```text
ProjectTestCaseAttemptResultV1
  case_attempt_result_id: StableId
  case_attempt_result_version: 1
  request_ref: exact ProjectTestRunRequestRefV1
  resolved_selection_ref: exact ProjectTestSelectionClosureRefV1
  suite_ref: exact ProjectTestSuiteRefV1
  case_ref: exact ProjectTestCaseRefV1
  parameter_identity:
    {kind=none}
    | {
        kind=expanded,
        parameter_set_ref: exact ProjectTestParameterSetRefV1,
        parameter_value_content_hash: SHA-256
      }
  target_profile_ref:
    exact TargetProfileRefV1(profile_kind=runtime_target)
  test_kind: ProjectTestKind
  attempt_ordinal: positive u32
  retry_lineage:
    {kind=initial}
    | {
        kind=retry,
        previous_attempt_ref:
          exact ProjectTestCaseAttemptResultRefV1,
        retry_reason:
          case_policy_retry | infrastructure_policy_retry
      }
  project_revision_ref: exact ProjectRevisionRefV1
  engine_candidate_ref: exact PreparedCandidateRefV1
  public_contract_set_ref: exact PublicContractSetRefV1
  runner_artifact_ref: exact ArtifactRefV1
  execution_environment_ref: exact TestExecutionEnvironmentRefV1
  started_at: RFC 3339 timestamp
  completed_at: RFC 3339 timestamp
  attempt_status:
    passed | assertion_failed | crashed | timed_out | cancelled
    | unsupported_environment | infrastructure_error
  assertion_evidence_refs[0..65536]:
    sorted unique exact EvidenceRefV1
  artifact_refs[0..65536]:
    sorted unique exact ArtifactRefV1
  diagnostic_refs[0..65536]:
    sorted unique exact DiagnosticCodeRefV1
  case_attempt_result_content_hash: SHA-256

ProjectTestRunResultV1
  test_run_result_id: StableId
  test_run_result_version: 1
  request_ref: exact ProjectTestRunRequestRefV1
  resolved_selection_ref: exact ProjectTestSelectionClosureRefV1
  project_revision_ref: exact ProjectRevisionRefV1
  engine_candidate_ref: exact PreparedCandidateRefV1
  public_contract_set_ref: exact PublicContractSetRefV1
  target_profile_ref:
    exact TargetProfileRefV1(profile_kind=runtime_target)
  runner_artifact_ref: exact ArtifactRefV1
  execution_environment_ref: exact TestExecutionEnvironmentRefV1
  started_at: RFC 3339 timestamp
  completed_at: RFC 3339 timestamp
  run_status:
    passed | failed | cancelled | infrastructure_error | flaky
  case_attempt_result_refs[1..262144]:
    sorted unique exact ProjectTestCaseAttemptResultRefV1
  final_case_outcomes[1..65536]:
    sorted unique {
      suite_ref: exact ProjectTestSuiteRefV1,
      case_ref: exact ProjectTestCaseRefV1,
      parameter_identity:
        {kind=none}
        | {
            kind=expanded,
            parameter_set_ref: exact ProjectTestParameterSetRefV1,
            parameter_value_content_hash: SHA-256
          },
      target_profile_ref:
        exact TargetProfileRefV1(profile_kind=runtime_target),
      test_kind: ProjectTestKind,
      terminal_attempt_ref: exact ProjectTestCaseAttemptResultRefV1,
      final_status:
        passed | failed | cancelled | unsupported_environment
        | infrastructure_error | flaky
    }
  aggregate_algorithm_id: project_test_run_result_aggregate
  aggregate_algorithm_version: 1
  aggregate_algorithm_content_hash: SHA-256
  coverage_summary_refs[0..256]: sorted unique exact ProjectCoverageSummaryRefV1
  artifact_refs[0..65536]: sorted unique exact ArtifactRefV1
  diagnostic_refs[0..65536]: sorted unique exact DiagnosticCodeRefV1
  result_content_hash: SHA-256
```

Attemptのfull identityはSelection Closureのexactly one `selected` rowへ解決し、request、Project revision、Candidate、public contract、Target、runner、environmentはRequest、Selection、全Attempt、Run Resultでbyte equalityにする。同一case identityのattempt ordinalは1から連続し、ordinal 1だけが`initial`、以後は直前ordinalのexact Attempt Refを持つ`retry`でなければならない。branch、skip、duplicate ordinal、別case／requestへのretry、policy外retryまたは上限超過を拒否する。

Case attempt statusは`passed | assertion_failed | crashed | timed_out | cancelled | unsupported_environment | infrastructure_error`を区別する。`attempt_status=assertion_failed`では`|diagnostic_refs| >= 1`を必須にし、各Refを[Executable Contracts補助Catalog §12.1](../appendices/executable-contracts-operation-planning-catalog.md#121-mirakandiagnosticv1)のexact四Fieldの`DiagnosticCodeRefV1`へ解決する。`skipped`は実行結果ではなく、selectionから除外した理由付きdispositionとしてSelection Closureに記録する。`final_case_outcomes[]`のidentity projectionはselected identity集合とset equalityで、terminal Attemptは同identityの最大連続ordinalである。最初から`passed`だけならfinal `passed`、先行non-pass後にpassedなら`flaky`、それ以外はterminal statusからclosed mappingする。`pass_after_retry`をpassedへ畳み込まず、required caseが未選択、unsupported、quarantined、flakyまたはAttempt欠落ならrunをpassedにしない。

Run statusはfinal outcomeの決定論的aggregateである。全selected identityが初回`passed`なら`passed`、一件でもflakyで他にhard failureがなければ`flaky`、case failure／unsupportedは`failed`、User cancellationは`cancelled`、runner／environment failureは`infrastructure_error`とし、複数failure classの優先順位をRun aggregate Algorithm v1のcontent hashへ固定する。Result作成者がsummary statusを選ばず、Case Attempt、final outcome、Selectionの三集合から再計算してbyte equalityにする。

Human-readable report、JUnit等の外部format、Editor panelはcanonical resultのProjectionでありauthorityではない。Projection failure、truncation、locale差でcanonical statusを変えない。reportにはomitted artifact、redaction、next pageまたは取得失敗を明示する。

## 10. Public C++ test API

Project C++ authorは公開SDKのtest namespaceからsuite／case登録、lifecycle hook、typed assertion、bounded Project／Runtime test contextへ到達できる。API catalogとsubject stabilityは[Native Game Module](native-game-module.md)が所有し、本書はtest semanticsを所有する。

test binaryはShipping Gameへ既定でlink／packageしない。test-only symbol、assertion message、golden artifact、secret inputをShipping artifactへ混入させない。Shipping configurationでtest APIを参照した場合はlinkまたはpackage前にtyped diagnosticで拒否する。

setup／teardown failureはcase body failureと区別する。teardownはsetup成功範囲だけを逆順に解放し、failureがあっても残りcleanupをboundedに試み、全failureを保持する。destructor exception、process abort、silent swallowを公開contractにしない。

## 11. Editor journey

Editorは最低限、次を提供する。

- Project revisionにあるsuite／caseの発見、filter、run configuration、Target選択。
- source locationからcaseへ、failureからProject object／code／assetへ往復するnavigation。
- live progress、attempt、stdout／diagnostic／artifactのbounded表示。
- previous same-case／same-Target／same-environment resultとの比較。
- golden／snapshot updateを通常runと分離したreviewable ChangeSetとして提示。
- cancelled、crashed、timed out、unsupported、quarantined、flakyを異なる状態として表示。
- headlessで再現するexact CLI commandまたはrequest artifactのexport。

Editorを閉じてもheadless runは同じsemanticsで完了できる。Editor crashをtest case failureとして記録せず、runner processとresult publicationのauthorityを分離する。

## 12. Product release binding

Product Lifecycleはsame release candidateについて最低限、次を集約する。

1. Template／Sampleに含まれるProject testsがclean checkoutで発見できる。
2. public SDK snippetとcontract testsがsame public contract setを参照する。
3. 2D Referenceと3D Referenceのrequired Project test scenarioがnon-substituting集合として完走する。
4. GUI／CLI／headlessのselection、result、diagnostic parityが成立する。
5. required Windows／Android／Apple等Targetの実行可能environmentとunsupported境界が明示される。
6. retry、flaky、quarantine、timeout、crash、wrong revision、stale goldenのnegative scenarioを拒否する。
7. test-only code／dataがShipping packageに混入しない。
8. Resultがsame Project revision、Engine candidate、public contract set、Target、runner、environmentへ束縛される。

Project test passはDomain qualification、Security、Privacy、Accessibility、Store acceptanceを代用しない。Product releaseは各OwnerのEvidenceと本書のProject Developer journeyを両方必要とする。

## 13. Failureと禁止fallback

- case discovery failure、duplicate ID、unknown tag、parameter overflowを空selectionまたはpassにしない。
- wrong Project revision、Engine candidate、public contract set、Target、runner、environmentのresultを再利用しない。
- unsupported environmentをskip、crashをassertion failure、timeoutをcancelledへ変換しない。
- retry後passで最初のfailureを削除しない。
- quarantined required caseをcoverage済みとしない。
- visual／performance baselineを近いGPU、別color space、別build configurationへfallbackしない。
- golden outputをfailure観測値で自動更新しない。
- Editor表示、log substring、process exit 0だけをcanonical test resultにしない。
- Project test runnerへarbitrary shell、unbounded filesystem、credential、network authorityを与えない。

## 14. 明示的非目標

- Engine内部CI、compiler conformance、AI EvalまたはDomain qualificationを本書へ統合すること。
- 特定のthird-party test framework型をpublic contractへ露出すること。
- test codeをShipping Runtime scriptingとして利用すること。
- record／playbackだけでassertionを自動生成し正しさを保証すること。
- flaky testをretryまたはquarantineでrelease可能に見せること。

## 15. 完了条件

- public suite、case、parameter、assertion、runner、resultの一意Ownerがある。
- GUI、CLI、headlessが同じrequest、selection、execution、result semanticsへ収束する。
- filter、shard、retry、isolation、timeout、determinism、flaky、quarantineが定義される。
- Unit、contract、integration、play、visual、performance、package smokeの境界がある。
- Project C++ test APIがShipping artifactと分離される。
- 2D／3D Referenceと第三者Developer release journeyへProject testing Evidenceを束縛できる。
- Engine内部testまたはAI VerificationをProject testの代用にしていない。
- API、runner、Schema、Fixture、Receiptが未materializeであることを実装済みと表現していない。
