# Miraikanai Engine 不変Engine境界・初心者向けAI技術承認規約

- 文書版: 1.0
- 作成日: 2026-07-21
- 最終更新日: 2026-07-21
- 調査基準日: 2026-07-21
- 対象: Game制作AI、初心者UX、構造化Game data、Project C++、Project Shader、NativeGameModule、Build、検証、承認、有効化
- 状態: プロジェクト公式の規範設計レビュー版。実装待ち
- 上位文書: [AIネイティブ独自ゲームエンジン 設計計画書](./2026-07-18-ai-native-game-engine-authoring-design.md)
- Game System: [Miraikanai Engine Game System／AI Code Generationアーキテクチャ規約](./2026-07-20-game-system-ai-codegen-architecture-design.md)
- Game実装方式: [Miraikanai Engine C++実行コード・構造化ゲームデータ規約](./2026-07-19-cpp-structured-game-data-design.md)
- Project C++: [Miraikanai Engine NativeGameModuleアーキテクチャ規約](./2026-07-19-native-game-module-architecture-design.md)
- AI権限: [Miraikanai Engine AI実装・保守ガバナンス規約](./2026-07-19-ai-engine-development-governance-design.md)
- 検証: [Miraikanai Engine AI検証・評価・来歴規約](./2026-07-19-ai-verification-evaluation-provenance-design.md)

## 1. 結論

Game制作時のMiraikanai Engine本体を、**署名済み・content-addressed・read-onlyの不変Engine baseline**として固定する。Game制作AI、人間のGame author、Editor plugin、Project C++、外部AgentのいずれにもEngine本体を変更するOperation、Path grant、link境界、承認経路を与えない。

AIはGame全体を隔離Stagingで生成できる。AIが変更できるのは、Project固有の構造化Game data、Asset、Test、`GameSystemSpecV1`、公開SDK内の`NativeGameModule`、Engine-owned entry／binding内のbounded Project Shaderに限る。公開Capabilityだけでは要求を実現できない場合、Game制作TaskはEngine Extension、Engine Adapter、Engine core変更へfallbackせず、`capability_unavailable`で停止する。

初心者はC++ sourceを読んでCode ownerになる必要がない。AIはSource生成、独立Review、Test提案、修復を行えるが、AIの自己申告を技術承認にしない。変更不能なEngine-owned Validator、Compiler、Analyzer、Test、Benchmark、Policy Serviceがhard Gateを判定し、構造化data、bounded Project C++、bounded Project Shaderの適用Gateをすべて通過したSystem Bundleだけに署名済み`SystemTechnicalAttestationV1`を発行する。

承認と有効化は次の三層へ分離する。

```text
AI generation scope
  = Game全体をStagingで完成可能

Technical qualification scope
  = 一つのGame System Bundle

Integration approval scope
  = 戦闘等のFeature Bundle

Human gameplay approval／Activation scope
  = exact Game Candidate全体
```

最後の一括承認は、未確認Sourceをまとめて正当化する操作ではない。個別に技術適格化されたSystemと統合検証済みFeatureを、exact hashの`GameCandidateManifestV1`として一括有効化する操作である。

## 2. 規範上の優先順位

本書はGame制作ProfileにおけるEngine不変境界、初心者向け技術適格化、階層承認の正本である。

既存文書に次の記述がある場合、Game制作中は本書を優先する。

- Capability不足時にEngine Extension、Engine Adapter、Engine core変更へfallbackする。
- Game制作TaskからEngine保守ActivationまたはR4 Engine変更Toolを利用できる。
- 初心者本人がAI生成C++のSourceを読んでCode owner承認しなければならない。
- AI reviewerの合格判断だけで技術承認またはPromotionできる。

Engine製品自体の開発Repository、Engine maintainer、将来の署名済みEngine ReleaseはGame制作Profileの外である。Game制作TaskのPrompt、Approval、Receipt、Branch、権限をEngine製品開発へ継承してはならない。将来のEngine ReleaseへProjectを更新する場合も、旧Engineをその場で変更せず、別に署名された新Baselineへの明示的Migrationとして扱う。

## 3. 用語

| 用語 | 定義 |
|---|---|
| `ImmutableEngineBaselineV1` | Engine、Editor、GameHost、標準Capability、公開SDK、Validator、Policy、Engine-owned Testのexact versionとhashを固定する署名済みManifest |
| Game制作Profile | `GameAuthoringProfileV1`。Project contentとbounded Project C++だけを変更できる製品Profile |
| Engine製品開発 | Engine sourceを保守し、新しい署名済みBaselineを作る別Repository、別主体、別Release工程 |
| Bounded Native | `BoundedNativeGameProfileV1`へ適合し、公開SDKと許可済みC++ subsetだけを使うProject固有C++ |
| Bounded Project Shader | `BoundedProjectShaderProfileV1`へ適合し、Engine-owned entry、binding、pass、resource境界内でoffline compileするProject固有Shader source |
| 技術適格化 | Sourceの意味を人間が保証することではなく、固定Gateの全合格とexact evidenceをPolicy Serviceが証明すること |
| Gameplay承認 | 人間が意図、操作感、難易度、見た目、許可Capability、試遊結果を確認すること |
| System Bundle | 一つのSystem契約、Definition、Source、Test、Save／Replay、Budget、Receiptのdependency closure |
| Feature Bundle | 戦闘等、一つのUser-visible機能を成立させるSystem Bundle集合と統合Test |
| Game Candidate | exact Engine baseline、Project revision、System／Feature Attestation、Asset、Build artifactを固定した有効化候補 |
| `capability_unavailable` | 現在の公開SDKでは要求を実現できず、Game制作Task内でEngine変更へ進まず停止した状態 |

## 4. 不変Engine baseline

### 4.1 `ImmutableEngineBaselineV1`

```text
ImmutableEngineBaselineV1
  baseline_id
  engine_product_id
  engine_version
  engine_artifact_root_hash
  editor_artifact_root_hash
  game_host_artifact_root_hash
  public_sdk_hash
  public_capability_set_hash
  mcd_meta_schema_hash
  contract_compiler_hash
  validator_set_hash
  engine_owned_test_set_hash
  policy_bundle_hash
  toolchain_lock_hash
  target_profile_set_hash
  file_manifest_root_hash
  signer_identity
  signature
```

`baseline_id`はUUIDv7 `StableId`とする。各`*_hash`はcanonical byte列のSHA-256とし、`signature`はガバナンス規約の固定署名形式を使用する。Projectは`baseline_id`だけでなく全Baseline root hashを`ProjectLockV1`へ固定する。

### 4.2 配置とFilesystem境界

Game制作環境では次を必須とする。

- Engine sourceをGame Project workspaceへmaterializeしない。
- Engine、Editor、GameHost、公開SDK、Validator、Policy bundleを署名済みread-only packageとしてmountする。
- Game制作AIのPath grantをProject root内の明示allowlistへ限定する。
- `engine/`、`editor/`、`runtime/`、`toolchain/`、`policy/`、Engine-owned `schemas/`、Engine-owned `tests/`へのwriteを常時拒否する。
- symlink、junction、reparse point、hard link、case variation、alternate data stream、archive展開を使ったPath escapeをBrokerで拒否する。
- Project Build scriptがEngine include path、link path、Validator、Compiler optionを上書きできないようにする。
- Build前、Build後、Preview前、Package前にBaseline signatureと全参照hashを再検証する。

Baseline drift、署名不一致、未知File、欠損Fileは修復可能なProject errorとして扱わない。`MIRAKAN-ENGINE-BASELINE_MISMATCH`で停止し、AIにEngine Fileの生成または修復を許可しない。

### 4.3 Toolと権限

`GameAuthoringProfileV1`のTool catalogへ次を登録してはならない。

- Engine source patch、Engine module追加、Engine private API公開。
- Engine Extension／Engine Adapter生成またはPromotion。
- Engine Compiler／Validator／Policy／Testの変更。
- Engine repository branch、commit、merge、tag、release。
- Engine binary差替え、署名差替え、Baseline Manifest再発行。
- 任意Pathへ書けるShellまたはFile Tool。

Prompt、`AGENTS.md`、Skill、外部MCP Client、人間のGame Project Approvalは、存在しないEngine変更権限を作れない。

## 5. Game制作で変更可能な範囲

### 5.1 許可するProject Artifact

- Game Brief、GameSpec、Requirement、Decision。
- `GameSystemSpecV1`とProject-defined System Contract。
- GameplayDefinition、World、Level、Scene、UI、Asset、Material、Animation、Audio設定。
- Project Test、Fixture、Benchmark、Replay、Save migration。
- `BoundedNativeGameProfileV1`に適合する`NativeGameModule`。
- `BoundedProjectShaderProfileV1`に適合するMaterial／Shader sourceとTarget別offline artifact。
- 生成Bindingの入力となるProject Contract。ただし生成器本体とEngine-owned Schemaは変更不可。

Project-defined Systemは第一級Systemであり、既存SystemのWhitelistへ閉じない。ただし公開Capability、typed Command／Event／Snapshot、State owner、Save／Replay、Budget、Target、Test境界を越えない。

### 5.2 実装方式の選択順

```text
既存System／Capabilityのcomposition
  -> GameplayDefinition
  -> Cook／index／layout最適化
  -> Bounded NativeGameModule
  -> capability_unavailable
```

Engine Extension、Engine Adapter、Engine core変更はこの選択肢に含めない。

### 5.3 `BoundedNativeGameProfileV1`

初心者向け自動技術適格化の対象となるProject C++は、次をすべて満たす。

- generated public SDK、C ABI、許可済みPrimary Named Moduleだけを使用する。
- Engine private header、source、object、pointer、container、allocator、GPU／Physics native handleへ到達しない。
- Platform、vendor、Editor headerをincludeまたはimportしない。
- File、socket、process、environment、registry、OS service、dynamic library、native SDKへ直接accessしない。
- inline assembly、未知link import、自己書換えCode、runtime code generationを使用しない。
- `reinterpret_cast`、所有raw pointer、直接`new`／`delete`、`malloc`／`free`、custom allocatorを使用しない。
- 独自Thread、mutex、condition variable、TLSを作らず、Engine-owned Job Capabilityだけを使用する。
- wall clock、entropy source、global mutable stateを使わず、Engine提供のTime／RNG／State handleを使用する。
- ExceptionをABI境界から伝播させない。
- 外部Dependencyを追加しない。
- 宣言済みComponent access、phase、queue、scratch、CPU、Memory Budgetを越えない。
- State owner、Command、Event、Snapshot、Save／Replay意味をMCD Contractと一致させる。

Source表現の禁止だけに依存せず、AST、Module graph、object import、link map、binary import、runtime traceを照合する。

このProfile内でもC++はmemory-safeであると証明された言語ではなく、GameHost Process内のEngine memoryへ到達し得る。したがって本書は「絶対安全」を表示しない。Engine／Editor／credential／Host OSへの影響を分離し、検証・故障回復・RollbackでRiskを制限する。

### 5.4 `BoundedProjectShaderProfileV1`

Project ShaderはEngine-owned Material／Shader entryとgenerated bindingへ接続し、任意Render pass、UAV、native GPU resource、descriptor heap、Shader compiler option、Engine private includeを追加できない。Targetごとに隔離Workerでoffline compile／validateし、Shipping RuntimeでSource生成、download、JIT compileを行わない。新しいNode、Domain、Shading Model、Compiler、Backendが必要なら`capability_unavailable`とする。

## 6. AIの役割と承認権限

| Actor | 役割 | 技術承認 |
|---|---|---:|
| Builder AI | 設計、Source、Test、修復案を生成 | 不可 |
| Reviewer AI | Requirement欠落、契約違反、欠陥候補を独立探索 | 不可 |
| Adversarial Test AI | 境界、失敗、攻撃、反例Testを提案 | 不可 |
| Engine Policy owner | Gate、bounded Profile、閾値、holdout、署名Baselineを製品Releaseとして管理 | Candidateごとの例外承認不可 |
| Engine-owned Validator／Compiler／Test | 決定論的または実行ベースGateを判定 | 個別Gate結果のみ |
| Policy Service | 全必須Evidence、hash、Risk、Gate結果を照合してAttestationへ署名 | 可 |
| Beginner／Game Author | Game意図、Capability scope、試遊、最終Activationを承認 | Source安全性の保証は不要 |
| Approval Service | 人間を認証し、表示したexact Candidateと明示判断を`HumanGameplayApprovalV1`へ署名 | 判断の代行不可 |
| Activation Service | Attestation／Approval coverageを照合し、current Candidate pointerを原子的に切替 | Activationのみ |
| Release owner | exact CandidateのPackage／配布を承認 | Release範囲のみ |

Builder AIとReviewer AIはTask role、Prompt、Context、traceを分離する。可能なら異なるModelまたはProviderでも評価するが、異なるAIを使っただけで独立性を証明したとは扱わない。Reviewer AIのBlocking findingはGate入力にできるが、「findingなし」を合格根拠にしない。

Policy Service、Approval Service、Activation ServiceはArtifactを生成、修正、Test削除、閾値変更してはならない。署名対象hashとGate PolicyはAI Task開始前に固定する。Approval Serviceは人間の明示操作なしに承認を作れず、Activation Serviceは有効なApproval coverageなしにCandidateを切り替えられない。

初心者ではなくEngine Policy ownerが、bounded Profileと検証Policyの残余Riskに責任を持つ。個別CandidateのためにGateを緩和する場合は新しい署名済みPolicy／Baseline Releaseと全回帰を必要とし、Game制作中の例外承認にしない。

## 7. AIがGameを理解したと判定する条件

「AIが理解した」という自然言語の自己申告を状態にしない。次をすべて満たした`GameUnderstandingClosureV1`だけを実装開始条件とする。

```text
GameUnderstandingClosureV1
  project_revision
  game_brief_hash
  requirement_set_hash
  decision_set_hash
  system_graph_hash
  state_owner_map_hash
  capability_scope_hash
  save_replay_contract_hash
  target_profile_set_hash
  behavior_budget_set_hash
  test_plan_hash
  evidence_closure_hash
  unresolved_blocking_question_count
  unresolved_high_requirement_count
  unsupported_capability_ids[]
  disposition
```

`ready_to_stage`にはBlocking／High未解決件数0、RequirementからSystem、Implementation、Test、Artifactまでの追跡100%、State owner重複0、stale Evidence 0を必須とする。未対応Capabilityが必須Requirementに含まれる場合、`disposition=capability_unavailable`とする。

Repository全体を無差別にContextへ投入しない。System Graph、Requirement ID、Capability ID、State owner、dependency edgeから必要Evidenceを検索し、使用したrevisionとhashを記録する。

## 8. 技術適格化Gate

すべてのSystem Bundle CandidateをG0～G7で技術適格化する。`implementation_kind`に応じて適用しない検査は、未実行のまま省略せず、固定Policyが理由付き`not_applicable_by_policy`を発行する。`unexecuted | failed | unknown | expired`が一つでもあればAttestationを発行しない。

- GameplayDefinitionのみ: G0、G1、構造化data向けG2～G4、G5～G7。
- Nativeのみ: G0～G7のbounded Native lane。
- hybrid: GameplayDefinition laneとbounded Native laneの和集合。
- Project Shaderを含むBundle: 上記にbounded Project Shader laneを加える。

### 8.1 G0 Engine integrity

- `ImmutableEngineBaselineV1` signature、File root、公開SDK hash一致。
- Engine／Editor／Validator／Policy／Engine-owned TestのDiff 0。
- Project path外write、junction escape、Build option override 0。

### 8.2 G1 Scope／Contract

- Requirement ID、System Contract、State owner、Capability、phase、Budget、Targetが完全。
- Source、Definition、Test、Migration、Build manifestが一つのSystem Bundle closure。
- 未許可Path、未宣言File、Test削除、Assertion緩和、Budget引上げ0。

### 8.3 G2 Source／Dependency

- 全Bundleで宣言済みFile、Schema、Asset、Definition、Dependency closureと実体が一致する。
- GameplayDefinition laneでは、任意関数名、native handle、File／Network／OS Operation、unbounded loop／recursion、未知Executable payloadが0。
- Native laneでは`BoundedNativeGameProfileV1` AST／Module／include／import conformanceを満たす。
- Native laneではEngine private、Platform、vendor、OS API参照0。
- Project Shader laneではEngine-owned entry／binding適合、任意pass／UAV／native resource／private include参照0。
- 未承認Dependency、License不明、dynamic load、inline assembly、未知binary import 0。
- Native laneではSource scanとobject／link map／binary importが一致する。

### 8.4 G3 Build

- GameplayDefinition laneはSchema validate、semantic validate、canonicalize、Cookをclean Workerで2回行い、同じPackage hashを得る。
- Native laneはPrimary／secondary C++23 Compilerでclean Build成功。
- Native laneはwarnings-as-errors、format、generated binding drift、Module DAG、ODR、ABI、Target conformance成功。
- Project Shader laneは全対象Targetのoffline compile、validation、reflection／binding一致、artifact hash再現性に成功。
- Cook／BuildはNetwork deny、secretなし、resource cap付き隔離Workerで行う。

### 8.5 G4 Correctness／Memory／Security

- 全laneでUnit、property、fuzz、integration、negative Test。
- GameplayDefinition laneはParser、Validator、Cooker、C++ evaluatorへmalformed／boundary inputを与え、capacity、determinism、typed failureを検証する。
- Engine-owned invariant、Catalog golden fixture、restricted holdout。
- Native laneはAddressSanitizer、UndefinedBehaviorSanitizer、ThreadSanitizer等の該当Target lane。
- Native laneはstatic analysis、lifetime、integer、bounds、uninitialized、taint、unsafe API検査。
- Project Shader laneはShader validator、bounds／NaN／Inf／resource access検査、Engine-owned visual／fault fixtureを通過する。
- Builder AIが作ったTestだけで合格しない。

### 8.6 G5 Game semantics

- fixed-seed Replay一致。
- Save／Load round-trip、version、migration、old-save fixture。
- Command／Event順序、Snapshot immutable、State owner一意性。
- Feature内と依存System間のintegration scenario。
- Requirement別observable outcome。

### 8.7 G6 Performance／Reliability

- CPU、Memory、allocation、queue、startup、frame／tick deadline Budget。
- Native Source、Algorithm、State layout、spawn／scale、Budget、Targetへ影響する変更は10分×3回の固定fixture比較と最悪回。
- それ以外の変更は固定`ChangeImpactPolicyV1`が選ぶtargeted fixtureを実行し、full performance Receiptを再利用する場合は全performance input hash不変と非影響証明を必要とする。
- crash、timeout、resource exhaustion、invalid input、restart、last-known-good復旧。
- 必要なTarget実機またはReference hardwareでの確認。

### 8.8 G7 Provenance／Attestation

- 適用laneのSource／Definition、generated binding、Dependency Set、Cook／Build input、Toolchain、Test、Artifact、Receiptのhash closure。
- trusted Builder identity、signature、Build type、parameter、Artifact subject digest検証。
- `SystemTechnicalAttestationV1`へPolicy Serviceが署名。

AI ProfileのCorpus評価における95%等の成功率は、AI機能をProductionへ解放するための統計である。個別Game Candidateの必要Gateを95%で合格させてはならない。個別Candidateは全hard Gate合格を必要とする。

### 8.9 Staging PreviewとEvidence再利用

```text
ChangeImpactPolicyV1
  policy_id
  policy_version
  gate_dependency_graph_hash
  impact_analyzer_hash
  evidence_max_age_by_kind
  full_performance_trigger_field_ids[]
  target_change_requires_full: true
  native_source_change_requires_full: true
  signature
```

AIは編集中のStaging CandidateへSchema validate、incremental Cook／Build、targeted Test、操作可能Previewを高速に実行できる。ただしこれは`preview_only`であり、Technical AttestationまたはActivation権限を作らない。

`full_performance_trigger_field_ids`はspawn／scale、State layout、Budget、quality、Target、Algorithm等のperformance入力Fieldをclosed IDで列挙する。PolicyとAnalyzerはEngine baselineの署名対象であり、Game ProjectまたはAIが変更できない。

既存Evidenceを再利用できるのは、固定`ChangeImpactPolicyV1`がそのEvidenceの全入力、Toolchain、Policy、Engine baseline、依存Artifact、Targetがexact hashで不変と証明し、Receiptが失効していない場合だけである。AIの「影響しない」という説明、File拡張子、Diff行数だけでは再利用しない。再利用したGateは`reused_valid_evidence`と記録し、元Receipt hashと非影響証明をAttestation closureへ含める。

これにより数値や配置の試行ごとにEngine／C++を再CompileせずPreviewできる一方、実際に有効化するCandidateは変更影響に必要な全hard Gateを通る。

## 9. 階層承認

### 9.1 `SystemTechnicalAttestationV1`

```text
SystemTechnicalAttestationV1
  attestation_id
  project_revision
  engine_baseline_hash
  system_contract_ref
  system_bundle_hash
  implementation_kind
  target_profile_hashes[]
  definition_package_hashes[]
  source_tree_hash?
  native_artifact_hashes[]
  project_shader_artifact_hashes[]
  capability_scope_hash
  test_evidence_root_hash
  performance_receipt_hashes[]
  provenance_root_hash
  gate_applicability_hash
  bounded_native_profile_hash?
  bounded_project_shader_profile_hashes[]
  gate_policy_hash
  result
  signer_identity
  signature
```

`implementation_kind`は`gameplay_definition | native_game_module | hybrid | target_specialized_set`である。GameplayDefinitionを含む場合はTargetごとの`definition_package_hashes`、Nativeを含む場合は`source_tree_hash`、Targetごとの`native_artifact_hashes`、`bounded_native_profile_hash`を必須とする。Project Shaderを含む場合も`source_tree_hash`、Targetごとの`project_shader_artifact_hashes`を必須とし、使用したProfile hashを`bounded_project_shader_profile_hashes`へ記録する。各配列はTarget Profile IDのUTF-8 byte順でcanonical sortし、ArtifactとTargetの件数を一致させる。該当しない配列は空、optional fieldはcanonical `absent`とし、空hashを使用しない。

一つのSystem Bundleにだけ有効とする。Source、Definition、Contract、Capability、Test、Budget、Engine baselineのいずれかが変われば無効化する。

### 9.2 `FeatureIntegrationAttestationV1`

```text
FeatureIntegrationAttestationV1
  feature_id
  requirement_ids[]
  constituent_system_attestation_hashes[]
  dependency_graph_hash
  integration_test_root_hash
  replay_root_hash
  save_replay_impact_hash
  behavior_budget_receipt_hashes[]
  result
  signer_identity
  signature
```

戦闘等の複数Systemを一つのUser-visible機能として検証する。System Sourceの技術適格化を置き換えない。

Policy Serviceは全構成System Attestationの有効性とFeature integration Gateを照合した場合だけ署名する。AIまたは人間のGameplay承認で代替しない。

### 9.3 `HumanGameplayApprovalV1`

```text
HumanGameplayApprovalV1
  approval_id
  approval_subject_kind
  approval_subject_hashes[]
  base_game_candidate_hash?
  result_game_candidate_hash
  covered_change_set_hash
  approved_requirement_ids[]
  approved_capability_scope_hash
  reviewed_replay_ids[]
  reviewed_target_profiles[]
  known_limitation_ids[]
  approver_identity
  issued_at
  expires_at
  signature
```

`approval_subject_kind`は`system_bundle | feature_bundle | game_candidate`である。初回制作は`game_candidate`一件、更新は変更範囲に応じて一つ以上の`system_bundle`または`feature_bundle`を選べる。粒度にかかわらず、承認は`result_game_candidate_hash`へ結び付け、別Candidateへ流用しない。

初心者はC++ Sourceではなく、Game意図、許可Capability、試遊、Replay、性能、既知制限、Rollback先を確認する。

### 9.4 `GameCandidateManifestV1`

```text
GameCandidateManifestV1
  candidate_id
  engine_baseline_hash
  project_revision
  game_spec_hash
  system_attestation_hashes[]
  feature_attestation_hashes[]
  target_profile_hashes[]
  structured_package_hashes[]
  asset_package_hashes[]
  game_binary_hashes[]
  whole_game_test_receipt_hash
  target_package_receipt_hashes[]
  rollback_candidate_hash
```

Target別Artifact配列は`target_profile_hashes`と同じ順序、同じ件数で対応させる。Target共通byte列でもhashを各Target slotへ明示し、暗黙fallbackを作らない。

Candidate assemblyは全System／Feature Attestationの有効性に加え、Game起動から終了、Save／Load、主要Core loop、全Target package、whole-game Replay、統合performance／memory、fault／rollbackのEngine-owned Testを実行する。`whole_game_test_receipt_hash`または必要な`target_package_receipt_hashes`が欠落、失敗、staleならManifestをActivation候補にしない。Whole-game Gateは下位System／Feature Gateを置き換えない。

### 9.5 `GameActivationReceiptV1`

```text
GameActivationReceiptV1
  activation_id
  previous_candidate_hash
  activated_candidate_hash
  human_gameplay_approval_hashes[]
  approval_coverage_hash
  attestation_closure_hash
  rollback_candidate_hash
  activated_at
  activation_service_identity
  signature
```

Activation Serviceは、全変更Systemのdependency impact closureが有効なSystem／Feature AttestationとHuman Gameplay Approvalで覆われ、すべてが同じ`activated_candidate_hash`を参照する場合だけReceiptへ署名する。AI、Policy Service、Game authorはcurrent Candidate pointerを直接変更できない。

最終ActivationはManifest root hashをcurrent Candidate pointerへ原子的に切り替える。構成Artifactを個別copyして「ほぼ同じCandidate」を有効化しない。起動失敗時は`rollback_candidate_hash`のlast-known-goodへ戻す。

### 9.6 初回制作と更新

初回はAIがGame全体をStagingで完成させてよい。AIは人間承認待ちの間も、未承認Artifactを正規Projectへ昇格させず、別SystemのStaging作業を続けられる。

初回制作では、System技術適格化とFeature統合検証を内部で段階実行した後、人間にはexact Game Candidate全体のGameplay承認を一回提示する。

更新時は変更影響Graphから最小dependency closureを求める。System単位またはFeature単位の承認画面でも、承認対象が反映されたexact result Game Candidate、影響範囲、Rollback先を同時に固定する。

| 変更 | 再承認 |
|---|---|
| 既存Schema範囲内の数値、Asset、配置 | 対象Systemの構造化lane再適格化＋System／Feature Gameplay承認 |
| GameplayDefinitionのRule／State変更 | 新System Technical Attestation＋影響Feature Integration Attestation＋System／Feature Gameplay承認 |
| Native Source／Build input変更 | 新System Technical Attestation＋影響Feature Integration Attestation＋System／Feature Gameplay承認 |
| Command／Event／Snapshot／State owner変更 | 依存System再適格化＋Feature Integration Attestation＋Feature Gameplay承認 |
| Save／Replay意味変更 | 全影響System、Migration、Feature統合、該当Game Candidate全体のGameplay承認 |
| Engine baseline更新 | 全Attestation失効、明示Migration、全再検証、Game Candidate全体のGameplay承認 |

一つのUI Sessionで複数承認を表示できるが、内部Receiptを一つの無境界な「Approve all」にまとめない。最終一括Activationは全構成Attestationが有効な場合だけ表示する。

## 10. 初心者向け承認UI

既定画面ではC++ Sourceを要求せず、次を平易な日本語で表示する。

- このSystem／Featureが何をするか。
- 読み取るState、変更するState、State owner。
- 使用するEngine Capability。
- File、Network、OS、外部SDKへ直接accessしないこと。
- Test総数、必須Test、失敗0、未実行0。
- Replay、動画、操作可能Preview。
- CPU、Memory、Frame／Tick Budgetの実測。
- 既知の制限、未対応Target、fallback。
- 「固定Policyの全必須Gateに適格」であり「未知欠陥が絶対にない」という保証ではないこと。
- 有効化するexact Candidate hashとRollback先。

表示例は次とする。

> 敵追跡Systemは敵とPlayerの位置を読み、敵の移動Intentを書き換えます。File、Network、OS機能は使用しません。必須Test 1,284件、固定Replay 36件、30分連続実行、Memory検査、性能Budgetに合格しています。

「AIが安全と判断」「問題ないと思われます」等の根拠不明表示を禁止する。表示Statusは`qualified | blocked | capability_unavailable | stale | infrastructure_failure`のclosed enumとし、Evidence、最終検証時刻、Engine baseline、Project revisionを必ず示す。

面白さ、操作感、難易度、Art、Game意図は人間のGameplay承認対象であり、CompilerやAI graderで代替しない。

## 11. Failureと停止規則

| 条件 | 結果 |
|---|---|
| Engine hash／signature不一致 | 全Build／Preview／Activation停止 |
| Engine private／OS／vendor API要求 | Source Gate拒否。安全な公開Capabilityへ再設計 |
| 公開Capabilityで意味同等実装不能 | `capability_unavailable` |
| AI reviewerと決定論的Gateが不一致 | Gate失敗を優先。人間にSource安全性判断を委ねない |
| Test不足、holdout取得不能 | 合格ではなく`infrastructure_failure` |
| Sanitizer／fuzz／Replay／Budget失敗 | Artifact隔離、Promotionなし |
| AIがTest削除、Assertion緩和、Budget引上げを提案 | 対象変更と同等以上のRiskで拒否 |
| Model／Prompt／Tool／Policy変更 | 対応Evalを再実行するまで旧Qualified Profileを維持 |
| Runtime fault | Session停止、Artifact invalid化、last-known-goodへRollback |

AIはGate失敗を直すためにEngine、Validator、Test、Budget、Policyを変更してはならない。同じBlocking Diagnosticが再発した場合は自動反復を停止する。

## 12. Compile、Preview、Shipping

```text
BuildImpactReceiptV1
  project_revision
  engine_baseline_hash
  changed_system_bundle_hashes[]
  target_profile_hashes[]
  executed_engine_compile_command_count
  executed_project_compile_command_count
  executed_shader_compile_command_count
  validate_duration_ms
  cook_duration_ms
  configure_duration_ms
  compile_duration_ms
  link_duration_ms
  test_duration_ms
  qualification_duration_ms
  peak_worker_memory_bytes
```

- 構造化Game dataだけの変更はC++再Compileを行わず、Validate／Cookする。
- Native Source変更は影響するProject Moduleと依存closureだけをCompileする。
- Game制作でEngine本体を再Compileしない。
- Windows PreviewはQualified artifactをread-only publishし、別GameHostを再起動する。
- Mobile／ConsoleのNative変更はTarget packageをclean Build、再署名、再installする。
- Shipping RuntimeでSource、Compiler、JIT、汎用Script VM、動的Native downloadを持たない。
- Shipping packageはexact Engine baseline、Project artifact、SBOM、provenance、署名を検証する。

構造化変更では`executed_engine_compile_command_count=0`かつ`executed_project_compile_command_count=0`、Native変更でも`executed_engine_compile_command_count=0`をhard Gateとする。編集PreviewとProduction適格化を分け、dual Compiler、Sanitizer、fuzz、long-runを含む適格化は非同期に実行できる。

実装前に「Project C++は何秒」と固定値を保証しない。Phase I0／I2で小／中／大Reference Projectのclean、no-op、構造化leaf、Native leaf、Module interface、generated binding変更を各10回以上測り、P50／P95／最悪値と実行Command数を公開する。製品SLOはそのReceiptとReference hardwareを伴うADRで固定し、外部EngineのBuild時間またはAI Benchmarkから転用しない。

## 13. Securityと保証限界

Build隔離は生成C++のRuntime安全性を証明しない。Project C++はGameHost内で信頼済みNative codeとして動くため、GameHostをEditor、AI credential、Build credential、Project authoring stateからProcess／OS boundaryで分離する。

技術適格化は次を証明する。

- 承認対象とBuild／Test／Artifactが同一hash closureである。
- 固定Policyで必要Gateをすべて実行し合格した。
- Engine baselineと公開Capability境界を変更していない。
- 既知の禁止依存、禁止API、検出対象の欠陥がない。

次は証明しない。

- C++に未知の欠陥が絶対に存在しないこと。
- AIが未記述のGame意図を完全に推測できること。
- Testが観測しない全将来入力で正しいこと。
- Gameが必ず面白いこと。

この限界を理由にGateを弱めず、Riskを許容できない要求は構造化実装、既存Capability、未対応停止のいずれかへ戻す。

## 14. 実装順

### Phase I0: 不変境界

- `ImmutableEngineBaselineV1`、`GameAuthoringProfileV1`、Path grant、Tool catalog denyをMCD化。
- Engine package read-only mount、signature／hash verifier、Baseline drift negative Test。
- Engine sourceを含まないProject／Worker fixture。

### Phase I1: 構造化Game制作

- GameUnderstanding closure、GameplayDefinition、System／Feature Bundle、構造化laneのG0～G7と`SystemTechnicalAttestationV1`。
- Engine変更要求が`capability_unavailable`で停止するnegative Test。
- 初心者向けEvidence UI、`HumanGameplayApprovalV1`、`GameActivationReceiptV1`、atomic Candidate activation。

### Phase I2: Bounded Native

- `BoundedNativeGameProfileV1`のAST、Module、object、link、binary conformance。
- `BoundedProjectShaderProfileV1`のentry／binding、offline compile、Target validation、visual／fault conformance。
- dual Compiler、static、sanitizer、property、fuzz、Replay、Save、Budget、holdout Gate。
- Native／hybrid laneを`SystemTechnicalAttestationV1`とPolicy Service署名へ追加。

### Phase I3: Production評価

- 小／中／大の実Game System corpus。
- 公開、restricted holdout、adversarial、incident Case。
- Model、Prompt、Tool、Schema、Policy、Engine baseline変更時の継続Eval。
- Windows、Android、AppleのTarget別clean Build／device Gate。

## 15. Definition of Done

1. Game制作AIのTool catalog、Authorization、FilesystemからEngine write経路が0件である。
2. Engine sourceがGame Project、AI Context、Source Worker Bundleへ含まれない。
3. Engine／Editor／Validator／Policy／Engine-owned Testを1 byte変更した全negative fixtureが拒否される。
4. symlink、junction、archive、case、Build option、link pathを使った境界回避が拒否される。
5. 公開Capability不足時にEngine Extensionへfallbackせず`capability_unavailable`となる。
6. Project-defined Systemとbounded Native C++で標準Catalog外の代表Game algorithmを実装できる。
7. Builder AI、Reviewer AI、Policy Serviceの権限が分離され、AI署名だけでAttestationを発行できない。
8. 全System Candidateは適用されるG0～G7の未実行、失敗、不明が一つでもあればPromotionされず、非適用Gateには固定Policyの理由がある。
9. AI生成Testだけでは合格できず、Engine-owned invariantとrestricted holdoutを通る。
10. 初心者がSourceを読まず、意図、Capability、Replay、性能、制限、Rollbackを確認できる。
11. System、Feature、Game Candidateのhash階層があり、下位変更で該当上位Approvalが失効する。
12. 初回はGame Candidate全体、更新はSystem／Feature単位で承認でき、どの承認もexact result Candidateと変更影響closureへ結び付く。
13. 最終Activationが未適格Systemまたは未承認変更を含めず、署名済みReceiptを残し、失敗時にlast-known-goodへ戻る。
14. 構造化変更はEngine／C++再Compileなし、Native変更はProject Module closureだけをCompileする。
15. 個別Candidateのhard GateをAI Profileの95%統計で相殺しない。
16. C++の未知欠陥を「絶対安全」と表示しない。

## 16. 代替案の棄却

| 案 | 結論 | 理由 |
|---|---|---|
| Builder AIが自分のC++をReviewし承認 | 棄却 | 相関した見落としと自己評価の過信を防げない |
| 初心者本人が全C++ Sourceを承認 | 棄却 | 意味のある技術Reviewにならず、初心者向け製品目標と矛盾 |
| 全Game Sourceを最後に一括技術承認 | 棄却 | Review可能性、原因局所化、Rollback、変更影響の境界を失う |
| Capability不足時にEngineを自動変更 | 棄却 | Engine不変境界、Project再現性、初心者の権限理解を破壊 |
| Project C++を全面禁止 | 棄却 | 未想定Algorithmと計測済みhot pathの自由度を不必要に失う |
| 不変Engine＋bounded Project C++＋階層Attestation | 採用 | Engineを固定しつつGame algorithmの自由度と初心者UXを両立できる |

## 17. 調査根拠と適用範囲

- [Unreal Engine Source Code](https://dev.epicgames.com/documentation/unreal-engine/downloading-source-code-in-unreal-engine): UEは通常のProject Module／PluginとEngine Source Buildを分離する一方、Source利用者はEngineを変更できる。MiraikanaiはGame制作Profileから後者の経路を除去する。
- [Unity Source Code](https://unity.com/products/source-code): Unityも通常Project拡張と、Enterprise向けSource Code Adaptによる変更Engine Buildを分離する。Miraikanaiの不変境界は通常Game制作側をさらに機械強制する。
- [Unreal Engine Modules](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-modules): Module分割、private dependency、変更ModuleだけのCompileという先例。
- [NIST AI RMF Core](https://airc.nist.gov/airmf-resources/airmf/5-sec-core/): AI Riskの役割分離、継続監視、文書化、Lifecycle管理。
- [NIST Generative AI Profile](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf): automation bias、over-reliance、実運用条件での評価、能力の安易な外挿禁止。
- [SLSA v1.2 Artifact Verification](https://slsa.dev/spec/v1.2/verifying-artifacts): Builder identity、署名、Artifact digest、Build type、parameter、期待値照合。
- [SWE-bench, ICLR 2024](https://proceedings.iclr.cc/paper_files/paper/2024/file/edac78c3e300629acfe6cbe9ca88fb84-Paper-Conference.pdf): Repository作業が複数File、実行環境、長いContextを必要とする根拠。Python Issue benchmarkであり、Miraikanai C++精度の数値には使用しない。
- [FEA-Bench, ACL 2025](https://aclanthology.org/2025.acl-long.839/): Repository levelの新Feature実装が単純Code生成より難しい根拠。
- [RepoCoder, EMNLP 2023](https://aclanthology.org/2023.emnlp-main.151/): Repository情報の分散と反復retrievalの必要性。
- [Secure coding with AI, Empirical Software Engineering 2026](https://link.springer.com/article/10.1007/s10664-026-10812-8): C／C++／C#を含むAI生成CodeのSecurity検出・修復が不完全で、Static toolと手動検証を併用すべき根拠。
- [Modern Code Review, ICSE 2013](https://www.microsoft.com/en-us/research/publication/expectations-outcomes-and-challenges-of-modern-code-review/): Reviewで変更理解が主要課題となる根拠。
- [Google Small CLs](https://google.github.io/eng-practices/review/developer/small-cls.html): 小さく自己完結した変更、関連Test、理解可能性、Rollbackを重視する実務根拠。
- [JAMER 2026 preprint](https://arxiv.org/abs/2606.19830): Game project規模増大時の能力低下と、Compile成功がRuntime behavior品質を保証しない補助根拠。Godot／GDScript中心の査読前研究であり、Miraikanai C++の精度値には使用しない。

外部Benchmarkの成功率をMiraikanaiの任意C++生成精度へ転用しない。Production判断にはMiraikanai固有のC++、Game System、Feature integration、Save／Replay、Target、Performance、Security corpusを実装後に測定する。
