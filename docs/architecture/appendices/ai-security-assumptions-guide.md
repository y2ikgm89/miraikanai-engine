# AI Security Assumptions／Questions Guide

- 文書ID: mirakan.appendix.ai-security-assumptions-guide
- 文書種別: explanatory supplement
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 親Owner: [AI Security／Approval](../01-governance/ai-security-approval.md)
- 正本範囲: Beginner question、assumption、Project data境界とprovisional Game understanding候補の説明
- 非正本範囲: 親Ownerが所有する安定Architecture原則、実装Task、実装順序、生成済みArtifactまたはQualification結果
- 規範依存: [親Owner](../01-governance/ai-security-approval.md)
- 関連文書: [Architecture Governance](../01-governance/architecture-governance.md)
- 根拠区分: project-decision／provisional。実ArtifactがないRegistry、Catalog、Fixtureは候補
- 外部根拠確認日: 2026-07-27

> この補助文書の型、Registry、Catalog、Fixtureはprovisionalな設計候補である。対応する正本Ownerとexact MCD Typeが親Ownerで採択され、Repository Artifactが存在するまでcurrent Schema、Context input、実装済み状態または理解完了を意味しない。親Ownerの`AiTaskContextCapsuleV1`／projection bindingを上書きしない。
## 5. Beginner questions、assumptions、理解条件

上位理解recordの候補familyは`GameIntentDraftV1`、`GameBriefV1`、`GameSpecDocumentV1`、`QuestionRecordV1`、`AssumptionRecordV1`、`DecisionRecordV1`、`GameUnderstandingClosureV1`、`AiCatalogEntryV1`である。これは許可済みtop-level Schema集合ではない。各候補は正本Owner、全direct ref、bound、canonical order、hash、Requirement／Project／Evidence joinが同じArchitecture Changeで確定するまでMCD Registryへ登録せず、`AiTaskContextCapsuleV1`へ束縛しない。以下のshapeは論点保全用であり、不足型を同名shadow schema、自由JSONまたは表示名から補完しない。

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

current Capsule／Projection bindingのSchema、Operation intersection、freshness、authorityは[親Owner §5](../01-governance/ai-security-approval.md#5-beginner-questionsassumptions理解条件)だけが所有する。`GameUnderstandingClosureV1`候補は、その全direct refのOwner採択後にcomplete Projectionを要求する候補であり、現在はbinding不能である。Architecture Explainは完成Inventory、Optimization Decisionは完成sealed qualification closureがなければbinding不能で、文書断片、raw traceまたはAI生成要約で補完しない。

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
- GameSystemSpecV1、Project-defined System Contract。
- GameplayDefinition、World、Scene、Space、UI、Asset、Material、Animation、Audio設定、選択済みPackの`StageDefinitionV1`等のowner-typed Source。Level WorkspaceはProject artifactではない。
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
