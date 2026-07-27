# Miraikanai Engine Architecture Governance Contract

- 文書ID: mirakan.arch.architecture-governance
- 状態: review
- 正本範囲: Architecture文書の識別・状態・inventory・一意所有、Architecture ChangeSet、Definition Migration binding、正本責務移管、AI向けArchitecture Explain projection、Architecture文書の分割・統廃合規則
- 非正本範囲: Product capabilityの成熟度、MCD／Operation activation、実装Taskの順序、各DomainのSchema・固定値・runtime挙動、AIの認可・実行route。各Owner文書を参照する
- 依存: [AI Security／Approval](ai-security-approval.md)、[AI Verification／Provenance](ai-verification-provenance.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Runtime ECS](../04-runtime/entity-component-system.md)
- 外部根拠検証日: 2026-07-26

## 1. 結論

Architectureは説明文の集合ではなく、各概念の責務、状態、依存、移管履歴を機械的に追跡できる正本集合として扱う。

Indexの記載件数や表示名を正本にせず、全Active文書のHeaderから生成するInventoryを唯一の件数根拠とする。Architecture文書の追加、統廃合、正本責務の移管は、本文の編集だけでcurrent stateを変えない。対象文書、旧新Owner、対象revision、影響するContract set、検証、承認境界を閉じたArchitecture ChangeSetで記録し、承認済みdefinition migrationが存在する時だけcurrent Owner Registryへ反映する。

本書は文書・責務の統治だけを定める。MCD type、Operation、Tool、Service、Provider projection、runtime capabilityを登録またはactivateしない。

## 2. 文書Inventory

### 2.1 Header record

全Active Architecture文書は次のHeaderを同じ順で持つ。

```text
- 文書ID: immutable ASCII ID
- 状態: review | normative
- 正本範囲: この文書だけが決定する事項
- 非正本範囲: この文書が決定してはならない事項と参照先
- 依存: 相対Linkの一覧
- 外部根拠検証日: YYYY-MM-DD
```

`review`は文書の承認状態であり、capability、MCD、Operation、Toolのactivation状態ではない。将来挙動は各Ownerが`not_activated`、`candidate_locked`、`qualified`、`production`のいずれかを明記し、Headerから推測しない。

Headerから次のrecordを生成する。

```text
ArchitectureDocumentRecordV1
  document_id: ASCII stable ID
  canonical_path: repository-relative normalized path
  document_status: review | normative
  canonical_scope: normalized text hash
  noncanonical_scope: normalized text hash
  dependency_document_ids[]: sorted unique ArchitectureDocumentId
  supersedes_document_ids[]: sorted unique ArchitectureDocumentId
  line_budget_class: compact | standard | split_required
  source_content_hash: SHA-256

ArchitectureInventoryV1
  inventory_version: positive uint32
  generated_at_revision: ProjectRevisionRefV1
  documents[]: ArchitectureDocumentRecordV1 sorted by document_id
  inventory_content_hash: SHA-256
```

`canonical_path`は実在する一意pathであり、redirect、旧path互換stub、同一`document_id`の複製を許可しない。相対Linkが示す対象はInventoryの一件へ解決しなければならない。`README.md`は閲覧用Indexであり、Inventory件数、文書ID、状態、pathを手入力で正本化しない。

### 2.2 件数とline budget

過去の再編Decisionにある42件は2026-07-21時点の移行基準値であり、current Inventoryの固定値ではない。追加・統廃合後の件数は必ず`ArchitectureInventoryV1.documents[]`から算出する。

新規または大幅に再編集する正本仕様は原則1,000行未満に保つ。1,000行以上が必要な場合は、次のいずれかを同じChangeSetに含める。

1. 型・固定値・Gateの単一Ownerを保った責務分割。
2. generated appendixまたは履歴資料への退避。appendixは正本規則を再定義しない。
3. 一時的に`split_required`を設定し、分割対象、残存期間、検証を明示した承認済みChangeSet。

行数だけを理由に意味的に密結合なSchemaを分裂させない。一方、Decision、runtime storage、永続化、package binary、AI projectionを一冊へ集約して検索範囲を不必要に広げることも禁止する。

### 2.3 現行のsplit-required文書

次の既存文書は1,000行以上のため、次の生成Inventoryでは`line_budget_class = split_required`として扱う。内容を増やすChangeSetは、単一Ownerを保つ責務分割、generated appendixへの退避、または明示的な例外承認のいずれかを同じclosureへ含める。これは実装Taskの順序ではなく、二重正本を増やさないための文書統治ruleである。

| Document ID | 現在の主な密結合領域 | 次の大幅改訂で必要な扱い |
|---|---|---|
| `mirakan.arch.product-plan` | capability／requirement／phase／Work Package registry | Product意図とregistry schemaの単一Ownerを保った責務分割、またはgenerated appendix |
| `mirakan.arch.ai-security-approval` | authorization／risk／trust／approval policy | identity・authorization・approval evidenceのOwner境界を保った責務分割 |
| `mirakan.arch.ai-verification-provenance` | verification／Eval／Evidence envelope／Receipt／provenance／external evidence | signed envelopeの単一Ownerを保ち、Requirement／Receipt／external evidenceの規則とgenerated appendixを責務別に分離 |
| `mirakan.arch.executable-contracts` | MCD core／operation registry／planned vocabulary／projection | current Contract coreとplanning ledgerを二重定義しない責務分割 |
| `mirakan.arch.simulation-physics` | Physics domain semantics／fixture／AI projection | semantic contractとgenerated／fixture appendixの分離、または行数をbudget内へ縮小 |
| `mirakan.arch.gameplay-programming-model` | GameplayDefinition／GameSystemSpec／State owner／codegen／promotion | stable Programming Model、generated contract projection、qualification appendixを単一Owner境界のまま分離 |
| `mirakan.arch.pack-gameplay-features` | reusable Feature contract／schema／state／fixture catalog | Feature単位の正本またはgenerated catalog appendixへ分割し、共通Pack boundaryを重複させない |
| `mirakan.arch.project-state` | Project aggregate／revision／ChangeSet／Target readiness／recovery | transaction core、Target readiness、recovery／fixture appendixを責務別に分離 |
| `mirakan.arch.editor-ui-framework` | MirakanUi Core／Shell、Widget Pattern、visual state、Accessibility、Reference fixture／baseline evidence | retained UI core／platform bridgeと、versioned Widget・semantic／UIA・fixture catalogを単一Owner境界のまま分離するか、後者をgenerated appendixへ退避。source icon・font lockはToolchain Ownerへ複写しない |
| `mirakan.arch.editor-workspace-ux` | Workspace／Panel、Reference Design、journey、baseline review／publication UX | workspace／Panel contractと、Reference fixture・review surfaceのprojection catalogを単一Owner境界のまま分離するか、後者をgenerated appendixへ退避。Widget token／UIA・Command規則をFramework Ownerへ複写しない |
| `mirakan.arch.pack-shooter` | Shooter composition／Profile／Game Flow／fixture | Genre compositionとfixture appendixを分け、Feature Public ContractをPackへ複写しない |
| `mirakan.arch.runtime-performance-capacity` | budget／capacity／measurement／qualification | normative budget／capacity ruleとgenerated measurement・qualification appendixを分離 |
| `mirakan.arch.rendering-world` | World source identity／spatial topology／partition／procedural source | source compositionとspatial／partition planを一意Ownerのまま分離 |

## 3. 一意所有と責務移管

### 3.1 Owner record

Architectureの正本責務はOwner Registryの`owner_id`とdocument revisionの組で参照する。path、見出し、表示名、リンク先からOwnerを推測しない。

```text
ArchitectureOwnershipRefV1
  owner_id: OwnerId
  owner_revision: positive uint32
  authority_document_id: ArchitectureDocumentId
  authority_document_content_hash: SHA-256

ArchitectureOwnershipTransferV1
  transfer_id: StableId
  subject_id: closed ArchitectureSubjectId
  source_owner_ref: ArchitectureOwnershipRefV1
  target_owner_ref: ArchitectureOwnershipRefV1
  source_scope_hash: SHA-256
  target_scope_hash: SHA-256
  moved_concept_ids[1..256]: sorted unique ClosedConceptId
  retained_concept_ids[0..256]: sorted unique ClosedConceptId
  affected_contract_set_refs[0..64]: sorted unique ContractSetRefV1
  compatibility_change_ref: CompatibilityChangeRefV1
  evidence_requirement_refs[1..64]: sorted unique EvidenceRequirementRefV1
  evidence_satisfaction_bindings[0..64]:
    sorted unique EvidenceSatisfactionBindingV1
  approval_ref: optional ArchitectureApprovalRefV1
  transfer_state: proposed | approved | applied | rejected
  content_hash: SHA-256

ArchitectureOwnershipTransferRefV1
  transfer_id: StableId
  content_hash: SHA-256
```

`source_owner_ref.owner_revision`をin-placeで書き換えない。authority documentの変更は同じ`owner_id`のrevisionをexactに一つ進め、新しいOwner Registry root、Foundation Definition Closure、参照するMCD／manifest／diagnosticのowner refを一つの承認済みdefinition migrationで更新する。説明文、相対Link、Product Work Packageの変更だけで`applied`にしない。

移管前のsource文書は旧名称や二重の詳細schemaを保持せず、current ownerと移管中の対象範囲だけを短く参照する。target文書は`proposed`または`review`の間、目標Contractを定義できるが、current runtime capabilityやOperationが既に存在すると主張してはならない。

### 3.2 Architecture ChangeSet

```text
ArchitectureChangeSetV1
  change_set_id: StableId
  purpose: document_restructure | ownership_transfer | compatibility_cutover
           | canonical_schema_move | document_retirement
  base_inventory_ref: ArchitectureInventoryRefV1
  resulting_inventory_ref: ArchitectureInventoryRefV1
  document_mutations[1..256]:
    add | revise | split | merge | retire
  ownership_transfers[0..64]: ArchitectureOwnershipTransferV1
  compatibility_change_refs[0..64]: CompatibilityChangeRefV1
  definition_migration_binding_ref: optional ArchitectureDefinitionMigrationBindingRefV1
  contract_activation_effect: none | approved_definition_migration
  evidence_requirement_refs[1..128]: sorted unique EvidenceRequirementRefV1
  evidence_satisfaction_bindings[0..128]:
    sorted unique EvidenceSatisfactionBindingV1
  approval_ref: optional ArchitectureApprovalRefV1
  state: draft | review | approved | applied | rejected
  content_hash: SHA-256
```

`contract_activation_effect = none`が既定であり、この時`definition_migration_binding_ref`はcanonical omissionする。Architecture文書を作成・更新・分割しても、MCD current set、Operation registry、Tool registry、runtime package、binary format、Save reader、AI route grantは変化しない。これらを変える場合だけ`contract_activation_effect = approved_definition_migration`とし、approved `ArchitectureDefinitionMigrationBindingV1`を明示参照する。`applied`には、bindingが指すsource／target Definition Closure、owner ref migration manifest、Compatibility Change、consumer inventory、verification、Approvalがread-backで一致することを必要とし、文書本文、相対Link、Product Work Packageの変更だけでcurrent化しない。

`retire`はGit履歴へ戻せる文書のみを対象にし、current Inventory、README、依存Link、Owner transferが整合した後に実行する。曖昧な旧文書をredirectとして残すことは互換性ではなく二重正本であるため禁止する。

### 3.3 Definition Migration binding

Architecture ChangeSetとProduct側のActive Definition migrationを相互にhashへ埋め戻すとcycleになる。そのため、先にsource／target根とowner ref集合だけを持つSubjectを確定し、bindingはSubjectを参照する。Product migrationとArchitecture ChangeSetは同じbindingを一方向に参照し、binding自身は後段wrapperへ逆参照しない。

```text
ArchitectureOwnerReferenceMigrationManifestV1
  manifest_id: StableId
  source_foundation_definition_closure_ref: FoundationDefinitionClosureRefV1
  target_foundation_definition_closure_ref: FoundationDefinitionClosureRefV1
  source_owner_ref: OwnerIdentityLocalRefV1
  target_owner_ref: OwnerIdentityLocalRefV1
  entries[1..65536]:
    reference_kind: active_mcd_owner | owner_manifest_contribution
                  | diagnostic_owner | runtime_scope_owner | game_system_owner
                  | generated_binding_owner | retained_artifact_owner
    logical_subject_ref: immutable content-addressed ref
    source_owner_ref: OwnerIdentityLocalRefV1
    target_owner_ref: OwnerIdentityLocalRefV1
  manifest_content_hash: SHA-256

ArchitectureDefinitionMigrationSubjectV1
  migration_subject_id: StableId
  base_inventory_ref: ArchitectureInventoryRefV1
  resulting_inventory_ref: ArchitectureInventoryRefV1
  ownership_transfer_refs[1..64]: sorted unique ArchitectureOwnershipTransferRefV1
  compatibility_change_refs[1..64]: sorted unique CompatibilityChangeRefV1
  consumer_inventory_refs[1..64]: sorted unique CompatibilityConsumerInventoryRefV1
  source_contract_set_ref: ContractSetRefV1
  target_contract_set_ref: ContractSetRefV1
  source_foundation_definition_closure_ref: FoundationDefinitionClosureRefV1
  target_foundation_definition_closure_ref: FoundationDefinitionClosureRefV1
  source_active_product_definition_sha256: SHA-256
  target_active_product_definition_sha256: SHA-256
  owner_reference_migration_manifest_ref: immutable content-addressed ref
  evidence_requirement_refs[1..128]: sorted unique EvidenceRequirementRefV1
  evidence_satisfaction_bindings[0..128]:
    sorted unique EvidenceSatisfactionBindingV1
  subject_content_hash: SHA-256

ArchitectureDefinitionMigrationSubjectRefV1
  migration_subject_id: StableId
  subject_content_hash: SHA-256

ArchitectureDefinitionMigrationBindingV1
  binding_id: StableId
  binding_version: positive uint32
  migration_subject_ref: ArchitectureDefinitionMigrationSubjectRefV1
  architecture_approval_ref: optional ArchitectureApprovalRefV1
  binding_state: prepared | approved | rejected
  binding_content_hash: SHA-256

ArchitectureDefinitionMigrationBindingRefV1
  binding_id: StableId
  binding_version: positive uint32
  binding_content_hash: SHA-256
```

`ArchitectureOwnerReferenceMigrationManifestV1.entries[]`は、source Closureから到達するsource Ownerのcurrent typed reference全体とset equalityにする。単に`owner_id`をgrepした結果、path、表示名、説明文をentryにしない。target側では全entryの`target_owner_ref`がtarget Closureのselected active rowへexact解決し、旧revisionがcurrent Contract set、Manifest、Diagnostic、Runtime Scope、Game System、generated bindingへ残らないことを検証する。retained source artifactはsource Closureと共に監査用に残せるが、target current dispatchへ混入させない。

Subjectの`source_active_product_definition_sha256`／`target_active_product_definition_sha256`はProduct Ownerが発行する完成Definitionのopaque equality anchorであり、GovernanceがProduct schema、Registry row、Product stateを所有することを意味しない。

SubjectのInventory、Ownership Transfer、Compatibility Change、Consumer Inventory、source／target Contract Set、Definition Closure、owner ref manifest、Evidence Requirement／pass fulfillment集合は相互にexact一致する。Bindingが`approved`になるにはSubjectの全ref、全Requirementのpass fulfillment、Architecture Approvalが一致する。BindingのApproval subjectは`migration_subject_ref`と`subject_content_hash`だけにexact束縛し、Architecture ChangeSet、Binding wrapper、Product migration wrapperをapproval hash preimageへ入れない。Bindingはimmutableなapproved recordのままにし、`applied`は同じBindingを参照するProduct側のDefinition migrationがsource／target active definition hashと同じclosureをatomicに切替えたことをread-backしたArchitecture ChangeSetだけの状態とする。bindingからProduct wrapperへの逆ref、Architecture ChangeSetからProduct wrapperをhash preimageへ戻すこと、latest検索、部分集合、状態だけを変えた別Bindingの発行を禁止する。

Architecture Definition Migrationで選ぶEvidence Requirementは次の閉じた用途に限る。Requirementのpass fulfillmentはBinding発行前に読み戻し、Architecture Approvalを技術Evidenceで代用しない。

| 検証対象 | `EvidenceRequirementV1.evidence_kind` | `acceptance_predicate` | pass時のexact条件 |
|---|---|---|---|
| source／target Foundation Definition Closure | `definition_closure_set_equality` | `closure_validation_pass` | Contract Set、Owner Registry、Product Definition anchor、source／target closureのcross-refが一致する |
| Owner reference migration manifest | `owner_reference_set_equality` | `exact_set_equality` | source Closure到達typed ref全量とmanifest entry集合、target selected owner refが一致する |
| Compatibility Consumer Inventory／ChangeSet | `compatibility_change_validation` | `closure_validation_pass` | Consumer Inventory／Compatibility Change／source format／affected class／rollback policyがSubjectと一致する |
| discovered reader／writer endpoint | `consumer_endpoint_inventory` | `endpoint_inventory_complete` | Consumer recordのendpoint／owner／rebuild／old reader・writer policyがInventoryと一致する |

Requirement、fulfillment subject、Technical Qualification Receipt、Subject、Bindingは同一immutable inputを別々に再hashし、document本文やlocal search結果だけを`pass`へ昇格させない。

## 4. Runtime ECS正本化ChangeSet

Runtime ECSの整理は次のtarget ChangeSetとして管理する。以下はmaterialize前のreview profileであり、完成した`ArchitectureChangeSetV1`そのものではない。これは実装指示、実装Task Plan、またはcurrent capability activationではない。

```text
RuntimeEcsCanonicalizationChangeSetV1
  change_set_id: architecture.runtime_ecs.canonicalization.v1
  materialization_type: ArchitectureChangeSetV1
  state: review
  contract_activation_effect: none
  definition_migration_binding_ref: absent
  source_owner_selector:
    owner_id: owner.core.runtime_ecs
    owner_revision: 1
    authority_document_id: mirakan.arch.gameplay-programming-model
  target_owner_selector:
    owner_id: owner.core.runtime_ecs
    owner_revision: 2
    authority_document_id: mirakan.arch.runtime-entity-component-system
  moved_concept_ids:
    - runtime_entity_identity
    - runtime_component_contract
    - runtime_archetype_layout
    - runtime_query_and_selection
    - runtime_component_access_manifest
    - runtime_structural_transaction
    - runtime_ecs_ai_contract_graph
  retained_source_concept_ids:
    - gameplay_definition
    - game_system_authoring
    - generated_gameplay_bundle
  target_document_ids:
    - mirakan.arch.runtime-entity-component-system
    - mirakan.arch.runtime-package
    - mirakan.arch.persistence-save
    - mirakan.arch.compatibility-evolution
  required_at_approval:
    - base and resulting Architecture Inventory refs
    - exact one ArchitectureOwnershipTransferV1 with full source and target hashes
    - complete CompatibilityConsumerInventoryV1 and approved CompatibilityChangeSetV1
    - profile-bound complete Registry Snapshot／Receipt closure for every external discovery scope
    - source and target Owner Registry, Contract Set, and Foundation Definition Closure refs
    - complete ArchitectureOwnerReferenceMigrationManifestV1
    - approved ArchitectureDefinitionMigrationBindingV1
    - exact Evidence Requirement／pass satisfaction-binding closure and required fixture refs
```

`RuntimeEcsCanonicalizationChangeSetV1`を完成`ArchitectureChangeSetV1`へmaterializeする時、source／target `ArchitectureOwnershipRefV1`はauthority document hashを含め、`ownership_transfers[]`、`compatibility_change_refs[]`、`definition_migration_binding_ref`、Evidence Requirement／satisfaction binding、Approvalを省略しない。known discovery seedはcurrent Owner Identity Registryで規定する`owner.core.runtime_ecs` selected rowと`scope.core.entity`のRuntime Scope owner refである。このselectorをmaterialized Registry content refまたはApprovalの存在と読み替えない。これらだけを完全集合と見なさず、source Foundation Definition Closureから到達するMCD、Owner Manifest、Diagnostic、Runtime Scope、Game System、generated binding、retained artifactの全typed refをmanifestで閉じる。

`RuntimeEcsCanonicalizationChangeSetV1`が`applied`になる前は、current Owner Registryの`owner.core.runtime_ecs` revision 1と[Gameplay programming model](../03-authoring/gameplay-programming-model.md)が現行authorityである。新しいECS、Runtime Package、Persistence文書は目標正本であり、実装済み・登録済みを意味しない。review profileの`contract_activation_effect = none`を文書編集だけで変更せず、complete consumer inventory、approved Compatibility ChangeSet、approved Definition Migration bindingを持つ完成recordを新たに発行する。

### 4.1 現在のapproval-readiness snapshot

以下は2026-07-24にlocal Gitとpublic GitHub metadataだけを調査したreview snapshotである。baseline `41929d0`には追跡pathが51件（`docs/**/*.md` 48件、`.codex/config.toml`、`.gitattributes`、`.gitignore`各1件）、Git tagは0件である。`origin`の公開refは`main`だけで、[GitHub Releases](https://github.com/y2ikgm89/miraikanai-engine/releases)も同日時点でrelease 0件を表示した。これは未署名の観測であり、外部release／distribution／ABI／API registryにconsumerが存在しない証明、またはEvidence Requirementのfulfillmentではない。現在のworktreeには未commitのArchitecture文書変更もあるため、このsnapshot自体をsource／target immutable refにしない。

| closure入力 | localで確認できる状態 | 安全な扱い |
|---|---|---|
| `ArchitectureInventoryV1` | Header／linkのreviewはできるが、immutable generated Inventory recordはない | `absent` |
| source／target Owner Registry、Contract Set、Foundation Definition Closure | document上のtarget記述だけで、content-addressed current／target recordがない | `unresolved` |
| `CompatibilityConsumerInventoryV1` | source／target format ref、署名済みscope fulfillment、外部Registry Profile／snapshot／Receiptがない | `collecting`／`unresolved` |
| external Registry authority profile／snapshot／collector Receipt | provider／tenant／namespace、official APIまたはsigned export、全page／manifest closure、trusted collector evidenceがない | `unresolved` |
| `CompatibilityChangeSetV1` | consumer inventoryとformat refが未materialize | `absent` |
| `ArchitectureOwnerReferenceMigrationManifestV1` | source Definition Closureがないため到達typed ref集合を算出できない | `absent` |
| Product active Definition anchors | 完成Active Product Definition record／state snapshotがない | `absent` |
| Evidence fulfillment | trusted Runner発行の対応`TechnicalQualificationReceiptV1`がない | `absent` |
| Architecture Approval | 人間Approval refと承認Authorityが提示されていない | `absent` |
| Definition Migration Binding／Architecture ChangeSet | 上記入力不足のため発行禁止 | `absent`、`contract_activation_effect=none` |

この表は実装Task、担当、見積り、実行順を定義しない。目的は、文書の存在、ローカル検索結果、またはAIの説明だけからBindingやApprovalを発行する誤りを防ぐことにある。外部scopeの実際の観測は[Compatibility／Evolution](../02-foundation/compatibility-evolution.md#45-外部registry-closureの提出境界)が定めるProfile／Snapshot／Receiptへ変換するまで、`unresolved`のまま保持する。この記載は外部Registryの照会、credential利用、export取得、Approval発行を許可しない。

## 5. AI向けArchitecture Explain projection

AIがArchitectureを理解する際、巨大文書全文、live memory、秘密値、未認可Sourceを入力にしない。Document InventoryとOwnerがsealしたimmutable projectionから、許可された範囲だけを取得する。

```text
ArchitectureExplainProjectionV1
  projection_version: 1
  project_revision_ref: ProjectRevisionRefV1
  architecture_inventory_ref: ArchitectureInventoryRefV1
  requested_scope_ids[1..64]: sorted unique ArchitectureScopeId
  documents[1..128]:
    document_id
    document_status
    canonical_scope_summary
    owner_refs[]
    source_or_derived: source | derived
    dependency_document_ids[]
    invariant_ids[]
    evidence_refs[]
  contract_nodes[0..1024]:
    node_id
    owner_ref
    status: current | target_review | not_activated
    exposure_policy_ref
  edges[0..4096]:
    from_node_id
    edge_kind: owns | depends_on | projects | verifies | supersedes
    to_node_id
  requested_field_mask
  returned_field_mask
  redacted_fields[]
  omitted_ranges[]
  continuation: optional ContinuationToken
  projection_content_hash: SHA-256
```

`ArchitectureExplainProjectionV1`をmaterializeする前提は、`architecture_inventory_ref`が`project_revision_ref`と同じrevisionから生成された完成`ArchitectureInventoryV1`へexact解決し、要求scopeから到達する全Document recordの`canonical_path`、`document_id`、`source_content_hash`、dependency集合がそのrevisionのSourceとbyte equalityになることである。Inventoryが`absent`、stale、部分生成、same-ID different-hash、または要求scopeの到達Documentを欠く場合、Projectionを生成せず、Markdown本文や検索結果を代替Inventoryとして補完しない。§4.1のcurrent snapshotではimmutable generated Inventoryが`absent`であるため、current `ArchitectureExplainProjectionV1`のmaterialized集合はexact `[]`である。

Inventory未materialize時も、認可済みDocument断片を`unverified_document_context`としてread-only参照できるが、Architecture全体、Owner closure、Activation、利用可能Capability、Evidence充足を確定する回答には使わず、[AI Security／Approval](ai-security-approval.md)の`AiTaskContextCapsuleV1`へ`architecture_explain` bindingとして格納しない。materialize後も`omitted_ranges[]`または`continuation != null`のquery型Projectionは取得済み範囲だけを説明でき、完全なArchitecture closureを要求するOperation inputまたは適合Caseを満たさない。

`status = target_review`は設計済みの目標であってcurrent MCDやruntime存在の証拠ではない。`source_or_derived = derived`のprojectionをSourceへ逆書込みしない。AIが取得・説明できることは変更権限を与えず、変更は[AI Security／Approval](ai-security-approval.md)のTask Authorizationと承認済みChangeSetを必要とする。

routeのcanonical enumはAI Securityの`engine_provider_adapter | standard_external_mcp | managed_external_host`だけである。本書はroute aliasを追加しない。MCP、provider、host固有のgrant評価はAI Security Ownerだけが決定する。

## 6. 文書作成・統廃合の検証

Architecture ChangeSetは少なくとも次を検証する。

1. Header必須Field、document ID、path、relative Link、依存先、Inventory hashが一致する。
2. 正本範囲が他文書の型、固定値、Gate、Operationを重複所有しない。
3. split／merge後に旧文書の詳細定義が残らず、移動先Ownerを一意に参照する。
4. Owner transferはrevision増分、Definition Closure、owner ref migration manifest、Compatibility Consumer Inventory、Compatibility Change、Evidence Requirement／pass fulfillment、Definition Migration binding、Approvalを同一closureで検証する。
5. 新規の完全な`operation.*` tokenは[Executable contracts](../02-foundation/executable-contracts.md)のclosed partitionへ同じ変更で分類する。本文書だけでOperationを登録しない。
6. AI projectionはInventory／Project revision／Source hash、field mask、sensitivity、redaction、omitted range、continuationを検証し、missing／stale InventoryをDocument検索で補完せず、raw credential、native pointer、live runtime memoryを含めない。
7. Definition Migration SubjectとBindingの生成順がhash cycleを作らず、BindingのApproval subjectがSubjectだけへ束縛され、Requirement／pass fulfillment集合、source／target root、Product active definition hash、owner ref manifestがexact一致する。
8. line budget、fenced block、見出し階層、リンク、重複document IDをlintする。
9. 公開handle／lease／view／owner、memory resource、native Adapter allocation、またはそれらの保存・job capture・retire規則を持つDomainは、[Memory／Pointers](../02-foundation/memory-pointers.md)の`PointerMemoryConsumerBindingV1`へconsumer conceptを正逆参照する。一般pointer taxonomy、allocation policy、live pointerの永続化禁止をDomain文書が再定義してはならず、実際に非該当ならownerと理由をclosed recordで明示する。

## 7. 変更時の読順

Architectureの変更は、(1) Inventoryと一意所有、(2)互換性、(3)対象Domainの正本、(4)境界Owner、(5)ProductのWork Package宣言、(6)実装・qualificationの順に検証する。ここでいう順序は文書の整合確認順であり、実装Taskの実行順を定義しない。
