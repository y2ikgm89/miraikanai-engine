# Miraikanai Engine Game Production Loop Contract

- 文書ID: mirakan.arch.game-production-loop
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Game production subject、Intent session／draft、Game Brief／GameSpec／Game Decision Ledger、Question／Answer／Assumption／Decision、Requirement traceability、Game understanding closure、AI game generation lane、Experience Goal、Playtest observation／evaluation、Iteration Decision、Game production loop closure、およびこれらを扱うtarget Operation familyの意味
- 非正本範囲: AI Task Authorization／Risk／Consent／Approval／Provider trust、Evidence envelope／署名／freshness、Project identity／Document共通header／ChangeSet／Commit、automated test runner／result、Gameplay System／Capability／Stateのdomain意味、Asset provenance／rights／safety、Build／Cook／Package／Launch、Product claim／First Playable／Release／Completion、実装順序／工程／工数／担当
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Project State](project-state.md)、[Developer Testing](developer-testing.md)、[Gameplay Programming Model](gameplay-programming-model.md)
- 関連文書: [Product Lifecycle](../00-product/product-lifecycle.md)、[Asset Lifecycle](asset-lifecycle.md)、[Editor Workspace／UX](editor-workspace-ux.md)、[Editor UI Framework](editor-ui-framework.md)、[Gameplay Programming Model](gameplay-programming-model.md)、[Native Game Module](native-game-module.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Persistence／Save](../04-runtime/persistence-save.md)、[RPG Genre Pack](../08-packs/rpg.md)
- 根拠区分: project-decision
- 外部根拠確認日: none

## 1. 結論

Miraikanai EngineのGame制作は、自然言語Promptを直接Projectへ適用する処理ではない。Intentをimmutable inputとして記録し、未決事項をQuestion／Assumption／Decisionで閉じ、確認済みBriefとGameSpecをProject Documentとして発行し、Requirementから実行契約とEvidenceまでを追跡する。そのCandidateをtechnical validationとPlaytestの定性的評価へ通し、次のIteration Decisionへ戻すまでを一つの追跡可能なproduction loopとする。

AIと人間は同じrecord、Project ChangeSet、Validator、Evidence、Approvalを使用する。会話履歴、要約、Prompt、Screenshot、AIの「理解した」「面白い」という自己申告はProject authority、理解完了、Gameplay ApprovalまたはProduct completionではない。

本書はtarget Architecture contractだけを定義する。現RepositoryにMCD Schema、Registry、Operation、Service、Project、Fixture、ReceiptまたはC++実装は存在せず、以下の型と状態名の記載をmaterialized、active、qualifiedまたはreleasedと解釈しない。

## 2. Authorityと非代替境界

本書はGame制作の意味とloopを所有する。隣接Ownerのauthorityを次のように消費し、複製しない。

| concern | canonical Owner | 本書の使用 |
|---|---|---|
| Task scope、Risk、Consent、Approval、Code Owner、Provider trust | [AI Security／Approval](../01-governance/ai-security-approval.md) | exact Authorization／Approval refを検証する |
| Evidence identity、署名、admissibility、freshness、revocation | [AI Verification／Provenance](../01-governance/ai-verification-provenance.md) | `fresh`なexact Evidence refだけを数える |
| Project／revision、Document header、ChangeSet、Commit、bootstrap publication | [Project State](project-state.md) | prepared candidateをatomic Commitする |
| automated test suite／case／run result | [Developer Testing](developer-testing.md) | exact Result refをtechnical inputとして読む |
| Gameplay System、State owner、implementation variant | [Gameplay Programming Model](gameplay-programming-model.md) | exact graph／projection refを読む |
| Asset provenance、rights、safety、promotion | [Asset Lifecycle](asset-lifecycle.md) | lane別Asset closureを読む |
| Product claim、First Playable、Release／Completion | [Product Plan](../00-product/product-plan.md)と[Product Lifecycle](../00-product/product-lifecycle.md) | loop closureをProduct predicateの入力として供給する |

Technical testは人間のObservationを代替せず、人間のObservationはtechnical Gateを代替しない。`GameProductionLoopClosureV1`はHuman Gameplay Approvalを参照するが発行権限を持たず、Project Commit、Source Promotion、Activation、ReleaseまたはCompletionを実行しない。

## 3. 共通identity、canonicalization、bound

### 3.1 完成recordとRef

本書が所有する全top-level recordはclosed recordであり、`schema_version=1`、型固有`StableId`、自己hashを除く全Fieldから導出した`content_hash: SHA-256`を持つ。unknown sibling Field、default補完、display nameからのID推測、同じID／versionに対する異なるhashを拒否する。

各`XxxRefV1`は`{xxx_id, xxx_version=1, xxx_content_hash}`のexact tupleで、解決先完成recordの同Fieldとbyte equalityにする。裸ID、path、timestamp、Project revisionだけ、`latest`、別recordのhashまたは会話中の呼称をRefとして受理しない。

本文で特記しない配列はMCD canonical bytes全体のunsigned lexicographic順でstrict sortし、duplicateを拒否する。ordered stepだけは明示`ordinal: uint16`を持ち、`ordinal=0..count-1`の連続列にする。全hashは[Executable Contracts](../02-foundation/executable-contracts.md)のcanonical encodingを使い、JSON property order、locale、filesystem orderまたはproducer orderへ依存しない。

### 3.2 共通bound

本書のinitial V1では、自由prose一FieldをUTF-8で`1..16384` bytes、短いsummary／reasonを`1..4096` bytes、locale tagをBCP 47 canonical formで`1..63` ASCII bytes、一recordのEvidence refを`0..256`件、その他のref配列を`0..4096`件に制限する。より小さい個別boundはそのField定義を優先する。上限超過をtruncate、分割後の暗黙結合、外部URL参照またはAI要約で補わず、typed Diagnosticで拒否する。

### 3.3 lineage invariant

一つのloopに参加する全recordは同じ`GameProductionSubjectV1`、`ProjectRefV1`、開始時`ContractSetRefV1`およびTarget集合へ解決する。`existing_project`では全Project Document／Artifactを同じ`ProjectRevisionRefV1`またはその正規descendantへ結び、別branchのsame-number revisionを拒否する。`bootstrap`ではstable Project identityを先に確保するが、Project Stateのatomic publication成功前にrevision 1、部分Projectまたはcurrent Documentを公開しない。

## 4. Game production subjectとIntent session

```text
GameProductionSubjectV1
  schema_version: 1
  subject_id: StableId
  project_ref: exact ProjectRefV1
  subject:
    bootstrap:
      bootstrap_request_ref: exact ArtifactRefV1(
        artifact_kind=project.bootstrap_request)
      bootstrap_profile_ref: exact ArtifactRefV1(
        artifact_kind=project.bootstrap_profile)
    | existing_project:
      project_revision_ref: exact ProjectRevisionRefV1
  subject_content_hash: SHA-256

GameIntentSessionV1
  schema_version: 1
  game_intent_session_id: StableId
  production_subject_ref: exact GameProductionSubjectRefV1
  task_authorization_ref: exact TaskAuthorizationEnvelopeRefV1
  parent_session_ref: exact GameIntentSessionRefV1 | null
  contract_set_ref: exact ContractSetRefV1
  target_profile_refs[1..64]: sorted unique exact TargetProfileRefV1
  locale_profile_refs[1..32]: sorted unique exact LocaleProfileRefV1
  opened_at: UtcTimestamp
  game_intent_session_content_hash: SHA-256

GameIntentDraftV1
  schema_version: 1
  game_intent_draft_id: StableId
  game_intent_session_ref: exact GameIntentSessionRefV1
  input_evidence_ref: exact EvidenceRefV1
  original_input_sha256: SHA-256
  input_locale: Bcp47Tag
  attachment_evidence_refs[0..64]: sorted unique exact EvidenceRefV1
  external_input_evidence_refs[0..64]: sorted unique exact EvidenceRefV1
  captured_at: UtcTimestamp
  game_intent_draft_content_hash: SHA-256
```

`GameProductionSubjectV1`はexact一branchだけを持つ。`bootstrap`にProject revisionを追加せず、`existing_project`にbootstrap refを追加しない。`parent_session_ref`は同じsubjectの直前sessionだけを許可し、cycle、別Project、別Contract setのsessionを拒否する。新しいiterationは新しいTask AuthorizationとSessionを発行し、旧Authorizationの残り時間や会話continuationを権限として再利用しない。

DraftはUser入力の改変検知とEvidence lineageを保存するがProject Documentではない。秘密、credential、権利未確認payloadをDraft本文へ複写せず、Evidence Ownerのaccess policyを維持する。

## 5. Brief、Question、Assumption、Decision、GameSpec

### 5.1 Game BriefとGameSpec Document

`GameBriefDocumentV1`と`GameSpecDocumentV1`は[Project State §4](project-state.md#4-documentモデル)の共通Document headerをそのまま使用する。Document identity、kind、schema、owner、revision、hash、dependencyはProject Stateが所有し、本書はpayloadだけを所有する。

```text
GameBriefDocumentV1.payload
  game_intent_session_ref: exact GameIntentSessionRefV1
  intent_draft_refs[1..64]: sorted unique exact GameIntentDraftRefV1
  intended_player_refs[1..32]: sorted unique exact ArtifactRefV1
  experience_intents[1..64]: ordered {
    ordinal: uint16,
    intent_text: Text<4096>,
    observable_indicator_text: Text<4096>
  }
  core_loop_steps[1..64]: ordered {
    ordinal: uint16,
    player_action_text: Text<4096>,
    game_response_text: Text<4096>,
    progress_effect_text: Text<4096>
  }
  scope_constraint_refs[1..256]: sorted unique exact ArtifactRefV1
  target_profile_refs[1..64]: sorted unique exact TargetProfileRefV1
  input_requirement_refs[1..256]: sorted unique exact ArtifactRefV1
  save_requirement_refs[0..64]: sorted unique exact RequirementRefV1
  online_requirement_refs[0..64]: sorted unique exact RequirementRefV1
  content_rights_requirement_refs[1..256]: sorted unique exact RequirementRefV1
  accessibility_requirement_refs[1..256]: sorted unique exact RequirementRefV1
  locale_profile_refs[1..32]: sorted unique exact LocaleProfileRefV1
  performance_profile_refs[1..64]: sorted unique exact ArtifactRefV1
  known_exclusion_refs[0..256]: sorted unique exact ArtifactRefV1
  confirmed_by_subject_ref: exact TrustSubjectRefV1
  confirmation_decision_ref: exact GameDecisionRecordRefV1

GameSpecDocumentV1.payload
  game_brief_document_ref: exact GameBriefDocumentRefV1
  requirement_refs[1..4096]: sorted unique exact RequirementRefV1
  game_experience_goal_set_ref: exact GameExperienceGoalSetRefV1
  system_intent_refs[1..1024]: sorted unique exact ArtifactRefV1
  content_set_refs[1..1024]: sorted unique exact ArtifactRefV1
  test_intent_refs[1..1024]: sorted unique exact ArtifactRefV1
  budget_profile_refs[1..256]: sorted unique exact ArtifactRefV1
  style_lock_refs[1..256]: sorted unique exact ArtifactRefV1
  target_profile_refs[1..64]: sorted unique exact TargetProfileRefV1
  capability_requirement_refs[1..1024]: sorted unique exact CapabilityRefV1
  save_replay_contract_set_ref: exact ArtifactRefV1
  spec_decision_refs[1..1024]: sorted unique exact GameDecisionRecordRefV1

GameDecisionLedgerDocumentV1.payload
  production_subject_ref: exact GameProductionSubjectRefV1
  current_decision_record_refs[1..1024]:
    sorted unique exact GameDecisionRecordRefV1
  retained_superseded_decision_record_refs[0..4096]:
    sorted unique exact GameDecisionRecordRefV1
  decision_subject_projection_hash: SHA-256
```

`GameDecisionLedgerDocumentV1.current_decision_record_refs[]`は各logical decision subjectの非superseded current head exact一件とset equalityにし、retained集合はそのheadから`previous_decision_record_ref`で到達する全履歴とset equalityにする。Architecture ADR、Approval、runtime eventまたは会話logを混入させない。

自由proseだけでRequirement、Target、Capability、Input、Save、rights、accessibility、locale、budgetまたはtest intentを充足しない。Brief確認後の意味変更は同じDocumentを上書きせず、新Document revisionとDecisionをProject ChangeSetへ含める。

### 5.2 Question、Answer、Assumption、Decision

```text
GameQuestionRecordV1
  schema_version: 1
  game_question_record_id: StableId
  game_intent_session_ref: exact GameIntentSessionRefV1
  previous_question_record_ref: exact GameQuestionRecordRefV1 | null
  question_class: blocking | high | medium | low
  question_text: Text<4096>
  affected_subject_refs[1..64]: sorted unique exact ArtifactRefV1
  resolution:
    open: {}
    | answered:
      answer_text: Text<4096>
      answer_evidence_refs[0..32]: sorted unique exact EvidenceRefV1
      decision_ref: exact GameDecisionRecordRefV1
      answered_by_subject_ref: exact TrustSubjectRefV1
      answered_at: UtcTimestamp
    | withdrawn:
      reason_text: Text<2048>
      decision_ref: exact GameDecisionRecordRefV1
      withdrawn_by_subject_ref: exact TrustSubjectRefV1
      withdrawn_at: UtcTimestamp
  created_at: UtcTimestamp
  game_question_record_content_hash: SHA-256

GameAssumptionRecordV1
  schema_version: 1
  game_assumption_record_id: StableId
  game_intent_session_ref: exact GameIntentSessionRefV1
  previous_assumption_record_ref: exact GameAssumptionRecordRefV1 | null
  source_question_ref: exact GameQuestionRecordRefV1(
    question_class=medium,resolution=open)
  typed_default_value_ref: exact ArtifactRefV1
  default_summary: Text<2048>
  basis_evidence_refs[1..32]: sorted unique exact EvidenceRefV1
  valid_from: UtcTimestamp
  expires_at: UtcTimestamp
  revalidation_condition_refs[1..32]: sorted unique exact ArtifactRefV1
  disposition:
    proposed: {}
    | accepted:
      acceptance_decision_ref: exact GameDecisionRecordRefV1
      accepted_by_subject_ref: exact TrustSubjectRefV1
      accepted_at: UtcTimestamp
    | superseded:
      replacement_assumption_ref: exact GameAssumptionRecordRefV1
      supersession_decision_ref: exact GameDecisionRecordRefV1
      superseded_at: UtcTimestamp
  game_assumption_record_content_hash: SHA-256

GameDecisionRecordV1
  schema_version: 1
  game_decision_record_id: StableId
  game_intent_session_ref: exact GameIntentSessionRefV1
  previous_decision_record_ref: exact GameDecisionRecordRefV1 | null
  decision_kind:
    experience | requirement | constraint | content | target_profile |
    capability_disposition | implementation_selection | iteration_scope
  subject_ref: exact ArtifactRefV1
  selected_option_ref: exact ArtifactRefV1
  rejected_option_refs[0..32]: sorted unique exact ArtifactRefV1
  rationale_text: Text<4096>
  evidence_refs[0..64]: sorted unique exact EvidenceRefV1
  decided_by_subject_ref: exact TrustSubjectRefV1
  decided_at: UtcTimestamp
  game_decision_record_content_hash: SHA-256
```

Answerは独立した権威recordではなく、同じlogical Question IDを引き継ぐ新しい`GameQuestionRecordV1.resolution=answered` branchである。Question、Assumption、Decisionの`previous_*_ref`は同じstable IDの直前current headだけを指し、expected-previous CASで一方向に進める。過去recordを書き換えず、fork、cycle、same-ID different-previous、別subjectを拒否する。

Blocking／Highは`answered`またはDecision付き`withdrawn`まで閉じない。Mediumだけが有限期限のaccepted Assumption exact一件で一時的に閉じられる。Lowも暗黙無視せずanswered／withdrawn Decisionを必要とする。`expires_at <= valid_from`、期限切れ、根拠0、再検証条件0、複数active Assumptionを拒否する。

## 6. Requirement traceabilityと理解完了

```text
GameRequirementTraceabilityV1
  schema_version: 1
  game_requirement_traceability_id: StableId
  production_subject_ref: exact GameProductionSubjectRefV1
  project_revision_ref: exact ProjectRevisionRefV1
  game_spec_document_ref: exact GameSpecDocumentRefV1
  contract_set_ref: exact ContractSetRefV1
  entries[1..4096]: sorted unique {
    requirement_ref: exact RequirementRefV1,
    capability_refs[1..64]: sorted unique exact CapabilityRefV1,
    pack_refs[0..64]: sorted unique exact ArtifactRefV1,
    game_system_contract_refs[1..256]: sorted unique exact ArtifactRefV1,
    implementation_variant_refs[1..256]: sorted unique exact ArtifactRefV1,
    test_case_refs[1..256]: sorted unique exact ArtifactRefV1,
    artifact_refs[0..1024]: sorted unique exact ArtifactRefV1,
    evidence_requirement_refs[1..256]: sorted unique exact ArtifactRefV1
  }
  coverage_basis_hash: SHA-256
  game_requirement_traceability_content_hash: SHA-256

GameUnderstandingClosureV1
  schema_version: 1
  game_understanding_closure_id: StableId
  production_subject_ref: exact GameProductionSubjectRefV1
  game_intent_session_ref: exact GameIntentSessionRefV1
  project_revision_ref: exact ProjectRevisionRefV1
  contract_set_ref: exact ContractSetRefV1
  game_brief_document_ref: exact GameBriefDocumentRefV1
  game_spec_document_ref: exact GameSpecDocumentRefV1
  question_record_refs[0..512]: sorted unique exact GameQuestionRecordRefV1
  assumption_record_refs[0..256]: sorted unique exact GameAssumptionRecordRefV1
  decision_record_refs[1..1024]: sorted unique exact GameDecisionRecordRefV1
  requirement_traceability_ref: exact GameRequirementTraceabilityRefV1
  game_system_dependency_graph_ref: exact GameSystemDependencyGraphRefV1
  game_state_owner_projection_ref: exact GameStateOwnerProjectionRefV1
  system_implementation_set_ref: exact SystemImplementationSetRefV1
  capability_scope_ref: exact ArtifactRefV1
  save_replay_contract_set_ref: exact ArtifactRefV1
  test_plan_ref: exact ArtifactRefV1
  evidence_refs[1..256]: sorted unique exact EvidenceRefV1
  target_profile_refs[1..64]: sorted unique exact TargetProfileRefV1
  unsupported_required_capability_refs[0..256]:
    sorted unique exact CapabilityRefV1
  unresolved_blocking_question_count: uint16
  unresolved_high_question_count: uint16
  unresolved_medium_question_count: uint16
  unresolved_low_question_count: uint16
  disposition: ready_to_stage | capability_unavailable
  evaluated_at: UtcTimestamp
  game_understanding_closure_content_hash: SHA-256
```

Traceabilityの`entries[].requirement_ref` projectionはGameSpecのrequired Requirement集合とset equalityにし、各required RequirementをCapability、System、compatible Implementation、Test、Evidence requirementへ一意に到達可能にする。required Capabilityには`capability_refs`、System、Implementation、Test、Evidence requirementを空にできない。Artifactはstaging前には0件を許すが、存在しないArtifactを予定名やpathで補完しない。

Closureは全Question current headを過不足なく含め、四countを解決recordから再計算する。`ready_to_stage`にはBlocking／High／Low open 0、acceptedかつ有効AssumptionのないMedium open 0、Requirement coverage 100%、State owner collision 0、required Systemのmissing compatible Implementation 0、unsupported required Capability 0、`fresh`以外のrequired Evidence 0を全て要求する。required Capabilityが未対応なら`capability_unavailable`とし、質問closure、traceabilityまたはEvidence条件を緩和しない。二つ以外のdisposition、partial ready、AI overrideを認めない。

## 7. AI game generation lane

```text
AiGameGenerationLaneV1 =
  ai_composed_game |
  ai_generated_external_content |
  ai_generated_project_source
```

| lane | 対象 | 必須境界 |
|---|---|---|
| `ai_composed_game` | Project Document、Gameplay Definition、prequalified Pack、`domain_pack_reference | user_provided` Assetの構成 | Project ChangeSet、Requirement traceability、manual continuity |
| `ai_generated_external_content` | qualified Providerが生成する画像、音声、3D等 | Asset provenance、rights、safety、human review、promotion |
| `ai_generated_project_source` | Native C++／Project Shader Source | isolated generation、Code Owner、independent review、Build／Test、Source Promotion |

laneは互いを代替しない。`ai_composed_game`成功からoriginal Asset生成またはNative／Shader生成を主張せず、external content成功からProject Source supportを主張しない。Product claimは[Product Plan](../00-product/product-plan.md)が所有し、本書のlaneをexact集合として参照する。

## 8. Experience Goal、Playtest、Evaluation

```text
GameExperienceGoalSetV1
  schema_version: 1
  game_experience_goal_set_id: StableId
  production_subject_ref: exact GameProductionSubjectRefV1
  goals[1..256]: sorted unique {
    experience_goal_id: StableId,
    intended_player_refs[1..32]: sorted unique exact ArtifactRefV1,
    intended_experience_text: Text<4096>,
    observable_indicator_text: Text<4096>,
    failure_signal_text: Text<4096>,
    priority: required | important | optional,
    runtime_entry_refs[1..64]: sorted unique exact ArtifactRefV1,
    scenario_refs[1..64]: sorted unique exact ArtifactRefV1,
    target_profile_refs[1..64]: sorted unique exact TargetProfileRefV1
  }
  game_experience_goal_set_content_hash: SHA-256

PlaytestSessionDefinitionV1
  schema_version: 1
  playtest_session_definition_id: StableId
  production_subject_ref: exact GameProductionSubjectRefV1
  project_revision_ref: exact ProjectRevisionRefV1
  candidate_ref: exact ArtifactRefV1
  game_experience_goal_set_ref: exact GameExperienceGoalSetRefV1
  runtime_entry_ref: exact ArtifactRefV1
  target_profile_ref: exact TargetProfileRefV1
  build_receipt_ref: exact ArtifactRefV1
  package_receipt_ref: exact ArtifactRefV1
  launch_receipt_ref: exact ArtifactRefV1
  input_trace_ref: exact ArtifactRefV1 | null
  scenario_ref: exact ArtifactRefV1
  participant_class_ref: exact ArtifactRefV1
  consent_record_ref: exact ArtifactRefV1 | null
  automated_test_result_refs[0..256]:
    sorted unique exact ProjectTestRunResultRefV1
  playtest_session_definition_content_hash: SHA-256

PlaytestObservationSetV1
  schema_version: 1
  playtest_observation_set_id: StableId
  playtest_session_definition_ref: exact PlaytestSessionDefinitionRefV1
  observations[1..4096]: ordered {
    ordinal: uint16,
    observer_subject_ref: exact TrustSubjectRefV1,
    observed_at: UtcTimestamp,
    experience_goal_id: StableId,
    reproduction_context_ref: exact ArtifactRefV1,
    observation_kind:
      behavior | comprehension | pacing | affect | accessibility |
      usability | defect_signal,
    severity: blocking | high | medium | low | note,
    observation_text: Text<16384>,
    measurement_refs[0..64]: sorted unique exact ArtifactRefV1,
    evidence_refs[0..64]: sorted unique exact EvidenceRefV1
  }
  playtest_observation_set_content_hash: SHA-256

GameExperienceEvaluationV1
  schema_version: 1
  game_experience_evaluation_id: StableId
  playtest_session_definition_ref: exact PlaytestSessionDefinitionRefV1
  playtest_observation_set_ref: exact PlaytestObservationSetRefV1
  game_experience_goal_set_ref: exact GameExperienceGoalSetRefV1
  evaluations[1..256]: sorted unique {
    experience_goal_id: StableId,
    observation_ordinals[0..4096]: sorted unique uint16,
    disposition: met | partially_met | not_met | not_evaluated,
    rationale_text: Text<4096>,
    evaluator_subject_refs[1..16]: sorted unique exact TrustSubjectRefV1
  }
  evaluated_at: UtcTimestamp
  game_experience_evaluation_content_hash: SHA-256
```

Sessionはexact Candidate、Project revision、Runtime Entry、Target、Build／Package／Launch closureへ固定する。自動`play_scenario`結果はtechnical contextとして取り込めるがObservationを生成せず、Observation proseからtest passを推測しない。Consentが適用されるparticipant classで`consent_record_ref=null`を拒否する。

EvaluationのGoal projectionはGoal Setの全`required | important` Goalとset equalityにする。`not_evaluated`をpass、欠損を`met`、測定値を感情の代理へ変換しない。面白さを単一scoreに縮約せず、各Goalの判断主体と根拠を保持する。

## 9. Iteration Decisionとproduction loop closure

```text
GameIterationDecisionV1
  schema_version: 1
  game_iteration_decision_id: StableId
  production_subject_ref: exact GameProductionSubjectRefV1
  candidate_ref: exact ArtifactRefV1
  game_experience_evaluation_ref: exact GameExperienceEvaluationRefV1
  decision:
    accept_candidate:
      accepted_goal_ids[1..256]: sorted unique StableId
    | revise:
      affected_goal_ids[1..256]: sorted unique StableId
      observation_ordinals[1..4096]: sorted unique uint16
      requirement_delta_ref: exact ArtifactRefV1
      authorized_proposal_scope_ref: exact ArtifactRefV1
    | stop:
      stop_reason_ref: exact ArtifactRefV1
    | defer:
      defer_reason_ref: exact ArtifactRefV1
      reentry_condition_refs[1..32]: sorted unique exact ArtifactRefV1
  decided_by_subject_refs[1..16]: sorted unique exact TrustSubjectRefV1
  decided_at: UtcTimestamp
  game_iteration_decision_content_hash: SHA-256

GameProductionLoopClosureV1
  schema_version: 1
  game_production_loop_closure_id: StableId
  production_subject_ref: exact GameProductionSubjectRefV1
  game_understanding_closure_ref: exact GameUnderstandingClosureRefV1
  candidate_ref: exact ArtifactRefV1
  technical_validation_evidence_refs[1..256]:
    sorted unique exact EvidenceRefV1
  game_experience_evaluation_ref: exact GameExperienceEvaluationRefV1
  human_gameplay_approval_ref: exact ArtifactRefV1
  game_iteration_decision_ref: exact GameIterationDecisionRefV1
  closure_disposition: accepted | iteration_required | stopped | deferred
  evaluated_at: UtcTimestamp
  game_production_loop_closure_content_hash: SHA-256
```

`accept_candidate`はEvaluationの全required Goalが`met`、全important Goalが`met | partially_met`、blocking Observation 0でなければ発行しない。`revise.observation_ordinals[]`はEvaluationが参照する同じObservation Setの実ordinalだけを指し、解決対象Goal、Observation、Requirement delta、proposal scopeを欠落させない。`stop`と`defer`を成功扱いせず、`defer`は再入条件を必須にする。

Loop Closureの`accepted`にはUnderstanding `ready_to_stage`、全required technical Gate pass、required Evidence `fresh`、有効Human Gameplay Approval、Iteration `accept_candidate`を同一Candidate／Project lineage／Target集合へ束縛する。他branchはそれぞれIteration Decisionと一致させる。Closure発行だけでProject Commit、Product First Playable、ReleaseまたはCompletionへ昇格しない。

## 10. Target Operation familyとcurrent状態

target Operationは次の三familyへ分離する。

| family | target action | authority boundary |
|---|---|---|
| `game_intent_understanding` | session create、draft capture、question answer／withdraw、assumption accept／replace、brief confirm、spec publish、understanding close | state変更はprepared candidate、expected revision、Authorization／Approval、atomic Commitを必須にする |
| `game_experience_iteration` | playtest observation record、experience evaluate、iteration decide | Human Gameplay Approvalを発行せず、新Iterationには新Authorizationを要求する |
| `game_production_read` | bounded inspect、trace、explain | mutation Receipt、Approval、Commit、Promotionを発行しない |

Provider／MCP projectionへHuman Gameplay Approval、Project Commit、Source Promotion、ActivationまたはRelease authorityを公開しない。Editor、CLI、MCP、Providerごとにpayload semanticsやhidden defaultをforkしない。

現Repositoryの`materialized_operations`、`contract_active_operations`、`active_operations`、`operational_operations`はすべてexact `[]`で、三familyのcurrent stateは`not_activated`である。Operation表示名または本節のaction列からdispatchability、Service existence、Policy coverageまたはReceipt typeを合成しない。

## 11. Fail-closed規則

次を一件でも満たすCandidateを前状態のlast-valid recordを不変にして拒否し、原因別typed Diagnosticを返す。

- unresolved Question、期限切れAssumption、未確認Briefまたはtraceability欠損をAI要約で補う。
- 別Project、別revision lineage、別Contract set、別Targetまたは別CandidateのRecord／Evidence／Approvalを再利用する。
- unsupported Capabilityを近いPack、外部Engine概念、Native Source生成または手動作業へ暗黙変換する。
- `ai_composed_game`、external content、Project Source generationのlaneを相互代替する。
- ObservationからRequirement、severity、technical pass、ApprovalまたはAuthorizationを推測する。
- `not_evaluated`、Evidence state不明、期限不明、署名不明をpositive Evidenceへ数える。
- 会話continuation、Prompt、Screenshot、display name、path、裸IDまたは`latest`をexact refへ変換する。
- minimum Core、Product First Playable、ReleaseまたはCompletionをLoop Closureから推測する。

## 12. Target-design検証と完了条件

本Ownerのtarget-design closureは少なくとも次をArchitecture監査で満たす。

1. 本書の全top-level型とRef familyに正本Ownerがexact一件あり、appendixに同名shapeがない。
2. Project State、AI Security、AI Verification、Editor、Developer Testing、Gameplay、Asset、Productのconsumerがpayloadを複製せずexact Owner linkを持つ。
3. `GameUnderstandingClosureV1`のQuestion、Requirement、System graph、State owner、Implementation、Target、Evidence集合が同一lineageへ閉じる。
4. Observation→Evaluation→Iteration Decision→新Authorizationの循環が追跡可能で、technical test／human observation／Approvalが相互代替されない。
5. 三AI generation laneがProduct claimとAsset／Source gateへ非代替で接続する。
6. current Operation四集合がexact `[]`、implementationが`absent`のままである。

これらはArchitecture targetの整合性だけを示す。MCD materialization、C++実装、Build、Qualification、Activation、First Playable、ReleaseまたはProduct completionには、各Ownerが要求するRepository Artifactと署名済みEvidenceが別途必要である。
