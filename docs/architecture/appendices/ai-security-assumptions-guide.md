# AI Security Assumptions／Questions Guide

- 文書ID: mirakan.appendix.ai-security-assumptions-guide
- 文書種別: explanatory supplement
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 親Owner: [AI Security／Approval](../01-governance/ai-security-approval.md)
- 正本範囲: Beginner question、assumption、Project data境界の説明
- 非正本範囲: 親Ownerが所有する安定Architecture原則、実装Task、実装順序、生成済みArtifactまたはQualification結果
- 規範依存: [親Owner](../01-governance/ai-security-approval.md)
- 関連文書: [Architecture Governance](../01-governance/architecture-governance.md)
- 根拠区分: project-decision／provisional。実ArtifactがないRegistry、Catalog、Fixtureは候補
- 外部根拠確認日: 2026-07-27

> この補助文書の型、Registry、Catalog、Fixtureは、対応するRepository Artifactが存在しない限り未実装の設計候補である。親Ownerの安定原則や実装済み状態を上書きしない。
## 5. Beginner questions、assumptions、理解条件

上位理解recordのtop-level許可集合は`GameIntentDraftV1`、`GameBriefV1`、`GameSpecDocumentV1`、`QuestionRecordV1`、`AssumptionRecordV1`、`DecisionRecordV1`、`GameUnderstandingClosureV1`、`AiCatalogEntryV1`、`AiTaskContextCapsuleV1`のexact九Schemaだけとする。`AiTaskContextProjectionBindingV1`等の名前付きnested shapeはtop-level record、Catalog entry、権限、保存rootとして発行しない。全Field required、明示`| null`だけnullable、tagged unionはexact一branch、unknown Field禁止、配列boundsとcanonical order／unique、typed refのID／schema version／content hash exact解決、`MIRAKAN_CLOSED_RECORD_V1/<SchemaName>`の自己Field除外hashを本節の共通規則とbyte equalityにする。同名shadow schema、optional補完、自由文からのOperation／Authority／Target／Provider／Model／Host／Deployment推測を追加しない。

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

AiTaskContextProjectionBindingV1
  binding_kind:
    {kind: authoring_context}
    | {kind: architecture_explain}
    | {kind: game_understanding}
    | {kind: editor_context}
    | {kind: optimization_decision}
    | {kind: ai_debug_context}
  projection_schema_id: StableId
  projection_artifact_ref: TypedArtifactRefV1
  projection_content_sha256: Sha256
  owner_document_id: ArchitectureDocumentId
  project_revision_ref: ProjectRevisionRefV1
  source_revision_artifact_ref: TypedArtifactRefV1 | null
  requested_field_paths[1..256]:
    sorted unique CanonicalJsonPointer
  returned_field_paths[1..256]:
    sorted unique CanonicalJsonPointer
  completeness:
    {kind: complete, omitted_ranges: [], continuation: null}
    | {kind: bounded_query,
       omitted_ranges[0..64]:
         {collection_path: CanonicalJsonPointer,
          start_index: uint32,
          omitted_count: positive uint32},
       continuation:
         {kind: none}
         | {kind: present,
            continuation_ref: TypedArtifactRefV1,
            continuation_sha256: Sha256,
            expires_at: UtcTimestamp}}
  invalidation_condition_refs[1..32]: McdContractRefV1(kind=policy)

AiTaskContextCapsuleV1
  schema_version: 1
  capsule_id: StableId
  task_authorization_ref: TaskAuthorizationEnvelopeRefV1
  task_authorization_wrapper_sha256: Sha256
  project_ref: ProjectSnapshotRefV1
  project_revision_ref: ProjectRevisionRefV1
  active_operation_refs[1..64]: McdContractRefV1(kind=operation)
  context_projection_bindings[1..16]:
    AiTaskContextProjectionBindingV1
  issued_at: UtcTimestamp
  expires_at: UtcTimestamp
  content_hash: Sha256
```

`AiTaskContextProjectionBindingV1.binding_kind`と`projection_schema_id`は、`authoring_context=AuthoringContextPackV1`、`architecture_explain=ArchitectureExplainProjectionV1`、`game_understanding=GameUnderstandingClosureV1`、`editor_context=EditorContextSnapshotV1`、`optimization_decision=OptimizationDecisionProjectionV1`、`ai_debug_context=AiDebugContextV1`のexact一対一対応だけを許す。`projection_artifact_ref`と`projection_content_sha256`は完成Projection bytesへ一致し、Owner、Project lineage、source revision、field path、completeness、invalidation policyを表示名または最新revisionから補完しない。`source_revision_artifact_ref`はProjection rootが独立Source revisionを所有するときだけnon-nullにし、`optimization_decision`では`ArtifactCandidateBindingV1.source_revision_ref`とbyte equalityを必須にする。`GameUnderstandingClosureV1`と`OptimizationDecisionProjectionV1`は`completeness.kind=complete`だけを許し、欠落Field、omitted range、continuationを持つbindingを拒否する。complete branchの`continuation: null`はbounded branchの`{kind:none}`と異なるcanonical discriminatorであり、相互変換しない。

`AiTaskContextCapsuleV1.active_operation_refs[]`は、署名済みTask Authorizationのexact Operation allowlist、current operational MCD／Tool projection、Capability／Target Activation、route grant、Project／subject scopeのintersectionから導出した非空集合である。各bindingはその集合内Operationのregistered input schemaが要求するkind／field maskだけを含み、同じProject lineageとexact revisionへ閉じる。明示的なhistorical read Operationが登録されていない限りrevision混成を拒否する。`issued_at < expires_at`かつCapsule expiryはAuthorization、Projection、Policy、Receiptの最短expiryを超えず、dispatch直前にAuthorization wrapper、Operation集合、Project revision、Projection hash、invalidation condition、Receipt freshness／revocationをread-backする。一件でもdriftしたCapsuleをrebase、部分利用、別Projectionへのfallbackに使わず、新Capsuleを要求する。

Capsuleはread-only／Disposableな入力manifestであり、Operation、Authority、Approval、selection、continuation権限を新設しない。`architecture_explain`は完成Inventory、`optimization_decision`は完成sealed qualification closureがなければbinding不能である。current Architecture Inventoryが`absent`で、optimization explain／select Operationも未Activationであるため、両kindを含むcurrent Capsuleのmaterialized集合はexact `[]`とし、文書断片、raw benchmark log、AI生成要約で補完しない。

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
- GameSystemSpecV2、Project-defined System Contract。
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
