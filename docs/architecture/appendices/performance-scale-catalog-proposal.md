# Performance Scale Catalog Proposal

- 文書ID: mirakan.appendix.performance-scale-catalog-proposal
- 文書種別: proposal appendix
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 親Owner: [Performance／Capacity](../04-runtime/performance-capacity.md)
- 正本範囲: Owner-typed workload scale、Registry、AI action、integrated fixtureの候補
- 非正本範囲: 親Ownerが所有する安定Architecture原則、実装Task、実装順序、生成済みArtifactまたはQualification結果
- 規範依存: [親Owner](../04-runtime/performance-capacity.md)
- 関連文書: [Architecture Governance](../01-governance/architecture-governance.md)
- 根拠区分: project-decision／provisional。実ArtifactがないRegistry、Catalog、Fixtureは候補
- 外部根拠確認日: 2026-07-27

> この補助文書の型、Registry、Catalog、Fixtureは、対応するRepository Artifactが存在しない限り未実装の設計候補である。親Ownerの安定原則や実装済み状態を上書きしない。
## 9. Owner-typed workload scale modelと`ProjectScaleEnvelopeV2`

ScaleはWorldやGameplayの存在を前提にせず、ownerが登録したworkload domainと数値dimensionの集合で表す。UI-only、strict headless、Editor tool、resource service、content-only Projectは、存在しないWorld／Entity／authoritative gameplay用の偽axisやfidelity floorを作らない。World／spatialは選択domainが明示要求する場合だけ追加する。

本節の全`owner_ref`候補は[Gameplay programming model §Owner identity registry](../03-authoring/gameplay-programming-model.md#owner-identity-registry)が所有する`OwnerIdentityLocalRefV1`へ解決する。`owner_id`だけ、`latest`／`current revision`または文書hashをwire値として許可しない。次の短記は設計上の参照名であり、現在のRegistry rowまたは生成済みhashではない。完成Owner Registryをmaterializeした時点で三Fieldを取得し、同一active rowとbyte equalityを検証する。

| 規範的な値短記 | exact `OwnerIdentityLocalRefV1` |
|---|---|
| `performance_owner_ref_v1` | `{owner_id=owner.core.performance, owner_revision=<registry-selected>, owner_content_hash=<generated>}` |
| `world_owner_ref_v1` | `{owner_id=owner.core.world, owner_revision=<registry-selected>, owner_content_hash=<generated>}` |
| `asset_lifecycle_owner_ref_v1` | `{owner_id=owner.core.asset_lifecycle, owner_revision=<registry-selected>, owner_content_hash=<generated>}` |
| `project_state_owner_ref_v1` | `{owner_id=owner.core.project_state, owner_revision=<registry-selected>, owner_content_hash=<generated>}` |
| `security_approval_owner_ref_v1` | `{owner_id=owner.core.security_approval, owner_revision=<registry-selected>, owner_content_hash=<generated>}` |

```text
WorkloadDomainTypeRefV1
  domain_type_id
  domain_type_version: positive uint32
  domain_type_content_hash: SHA-256

WorkloadIntentKindRefV1
  intent_kind_id
  intent_kind_version: positive uint32
  intent_kind_content_hash: SHA-256

WorkloadIntentKindRecordV1
  intent_kind_id
  intent_kind_version: positive uint32
  owner_ref: OwnerIdentityLocalRefV1
  required_owner_intent_branch:
    content | authoring | authority
  allowed_authority_classes[1..5]:
    authoritative_simulation | presentation | ui | tooling | resource_service
  allowed_semantic_requirement_modes[1..5]:
    authoritative_equivalence | presentation_fidelity |
    functional_contract | resource_slo | none
  intent_kind_content_hash: SHA-256

WorkloadIntentKindRegistryRefV1
  registry_id
  registry_version: positive uint32
  registry_content_hash: SHA-256

WorkloadIntentKindRegistryV1
  registry_id: performance.workload_intent_kind.registry.active
  registry_version: positive uint32
  records[1..256]: WorkloadIntentKindRecordV1
  registry_content_hash: SHA-256

WorkloadOwnerDefinitionRefV1
  registry_ref: WorkloadOwnerDefinitionRegistryRefV1
  definition_id
  definition_version: positive uint32
  definition_content_hash: SHA-256
  definition_kind: workload_domain_owner | owner_scale_intent_owner
  owner_ref: OwnerIdentityLocalRefV1

WorkloadOwnerDefinitionRecordV1
  definition_id
  definition_version: positive uint32
  definition_kind: workload_domain_owner | owner_scale_intent_owner
  owner_ref: OwnerIdentityLocalRefV1
  schema_kind:
    workload_domain_intent_v1 |
    world_scale_intent_v1 | content_scale_intent_v1 |
    authoring_scale_intent_v1 | authority_scale_intent_v1
  owner_intent_branch:
    world | content | authoring | authority |
    canonical omission when definition_kind=workload_domain_owner
  allowed_domain_type_refs[0..256]: WorkloadDomainTypeRefV1
  definition_content_hash: SHA-256

WorkloadOwnerDefinitionRegistryRefV1
  registry_id
  registry_version: positive uint32
  registry_content_hash: SHA-256

WorkloadOwnerDefinitionRegistryV1
  registry_id: performance.workload_owner_definition.registry.active
  registry_version: positive uint32
  records[1..4096]: WorkloadOwnerDefinitionRecordV1
  registry_content_hash: SHA-256

PerformanceTargetProfileRefV1
  target_profile_id
  target_profile_version: positive uint32
  target_profile_content_hash: SHA-256

PerformanceQualificationReceiptRefV1
  qualification_id
  qualification_version: positive uint32
  qualification_subject_hash: SHA-256
  signed_record_hash: SHA-256

PerformanceQualificationBindingRefV1
  qualification_binding_id
  qualification_binding_version: positive uint32
  qualification_binding_hash: SHA-256

PerformanceDecisionRecordRefV1
  decision_id
  decision_version: positive uint32
  decision_content_hash: SHA-256

WorkloadDomainTypeRecordV1
  domain_type_id
  domain_type_version: positive uint32
  owner_ref: OwnerIdentityLocalRefV1
  authority_class:
    authoritative_simulation | presentation | ui | tooling | resource_service
  spatial_requirement: forbidden | optional | required
  required_intent_kind_refs[0..32]: WorkloadIntentKindRefV1
  allowed_scale_dimension_refs[1..256]: RuntimeScaleIntentDimensionRefV1
  semantic_requirement_mode:
    authoritative_equivalence | presentation_fidelity |
    functional_contract | resource_slo | none
  domain_type_content_hash: SHA-256

WorkloadDomainTypeRegistryRefV1
  registry_id
  registry_version
  registry_content_hash

WorkloadDomainTypeRegistryV1
  registry_id: performance.workload_domain_type.registry.active
  registry_version
  intent_kind_registry_ref: WorkloadIntentKindRegistryRefV1
  records[1..4096]: WorkloadDomainTypeRecordV1
  registry_content_hash

WorkloadDomainIntentV1
  intent_id: StableId
  intent_version: positive uint32
  domain_type_ref: WorkloadDomainTypeRefV1
  owner_definition_ref: WorkloadOwnerDefinitionRefV1
  dimension_values[1..64]: RuntimeScaleIntentDimensionValueV1
  requirement_refs[0..64]: McdContractRefV1(kind=requirement)
  equivalence_or_fidelity_policy_refs[0..64]: McdContractRefV1(kind=policy)
  intent_content_hash

WorkloadDomainIntentRefV1
  intent_id: StableId
  intent_version: positive uint32
  intent_content_hash: SHA-256

SelectedWorkloadDomainRecordBindingV1
  domain_type_ref: WorkloadDomainTypeRefV1
  registry_ref: WorkloadDomainTypeRegistryRefV1
  selected_record_hash: exact domain_type_ref.domain_type_content_hash

OwnerScaleIntentRefV1
  intent_kind:
    world | content | authoring | authority
  intent_ref:
    world: exact {intent_id, intent_version, intent_content_hash,
                  owner_ref: OwnerIdentityLocalRefV1,
                  owner_definition_ref: WorkloadOwnerDefinitionRefV1}
    | content: exact {intent_id, intent_version, intent_content_hash,
                     owner_ref: OwnerIdentityLocalRefV1,
                     owner_definition_ref: WorkloadOwnerDefinitionRefV1}
    | authoring: exact {intent_id, intent_version, intent_content_hash,
                       owner_ref: OwnerIdentityLocalRefV1,
                       owner_definition_ref: WorkloadOwnerDefinitionRefV1}
    | authority: exact {intent_id, intent_version, intent_content_hash,
                       owner_ref: OwnerIdentityLocalRefV1,
                       owner_definition_ref: WorkloadOwnerDefinitionRefV1}

ProjectScaleEnvelopeV2
  scale_envelope_id: StableId
  project_id: StableId
  schema_version: 2
  source_project_ref:
    exact {project_id, project_revision, document_set_hash}
  target_profile_refs[1..16]: PerformanceTargetProfileRefV1
  workload_domain_registry_ref: WorkloadDomainTypeRegistryRefV1
  workload_intent_kind_registry_ref: WorkloadIntentKindRegistryRefV1
  workload_owner_definition_registry_ref:
    WorkloadOwnerDefinitionRegistryRefV1
  scale_dimension_registry_ref: RuntimeScaleIntentDimensionRegistryRefV1
  workload_domain_intents[1..64]: WorkloadDomainIntentV1
  selected_domain_record_bindings[1..64]:
    SelectedWorkloadDomainRecordBindingV1
  owner_intent_refs[0..4]: OwnerScaleIntentRefV1
  decision_refs[0..64]: PerformanceDecisionRecordRefV1
  envelope_hash: SHA-256

ProjectScaleEnvelopeRefV2
  scale_envelope_id: StableId
  schema_version: 2
  envelope_hash: SHA-256

PerformanceQualificationSubjectRefV1
  subject_kind:
    workload_domain | workload_intent | integrated_envelope |
    scale_dimension | migration
  subject_ref:
    workload_domain: WorkloadDomainTypeRefV1
    | workload_intent: WorkloadDomainIntentRefV1
    | integrated_envelope: ProjectScaleEnvelopeRefV2
    | scale_dimension: RuntimeScaleIntentDimensionRefV1
    | migration: ProjectScaleDomainMappingRefV1

PerformanceQualificationSubjectV1
  qualification_id
  qualification_version: positive uint32
  owner_ref: OwnerIdentityLocalRefV1
  subject: PerformanceQualificationSubjectRefV1
  target_profile_refs[1..16]: PerformanceTargetProfileRefV1
  fixture_refs[1..64]: exact {fixture_id, fixture_version, fixture_content_hash}
  input_closure_hash
  result: pass | fail
  qualification_subject_hash: SHA-256

PerformanceQualificationReceiptV1
  subject: PerformanceQualificationSubjectV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=performance_qualification)

PerformanceQualificationBindingV1
  qualification_binding_id
  qualification_binding_version: positive uint32
  subject: exact PerformanceQualificationSubjectRefV1
  qualification_receipt_ref: PerformanceQualificationReceiptRefV1
  qualification_binding_hash: SHA-256

ProjectScaleActivationProjectionV1
  projection_id
  projection_version: positive uint32
  scale_envelope_ref/hash: exact receipt-free ProjectScaleEnvelopeV2
  workload_domain_qualification_binding_refs[1..64]:
    PerformanceQualificationBindingRefV1
  workload_intent_qualification_binding_refs[1..64]:
    PerformanceQualificationBindingRefV1
  integrated_envelope_qualification_binding_ref:
    PerformanceQualificationBindingRefV1
  scale_dimension_qualification_binding_refs[1..256]:
    PerformanceQualificationBindingRefV1
  projection_hash: SHA-256
```

`WorkloadDomainTypeRefV1`、`WorkloadIntentKindRefV1`、`WorkloadOwnerDefinitionRefV1`、`WorkloadDomainIntentRefV1`、`ProjectScaleEnvelopeRefV2`は各Receipt-free Record外の参照形であり、Recordのlogical ID／version／self-excluding content hashからmaterializeする。Domain record hashはASCII `MIRAKAN_WORKLOAD_DOMAIN_TYPE_RECORD_V1`、Intent Kind record hashはASCII `MIRAKAN_WORKLOAD_INTENT_KIND_RECORD_V1`、Owner Definition record hashはASCII `MIRAKAN_WORKLOAD_OWNER_DEFINITION_RECORD_V1`と当該hash Fieldだけを除くReceipt-free canonical bytesから計算し、Record自身へhash付きRef、Qualification Receipt／Bindingを埋め戻さない。Intent Kind Registry hashはASCII `MIRAKAN_WORKLOAD_INTENT_KIND_REGISTRY_V1`、Owner Definition Registry hashはASCII `MIRAKAN_WORKLOAD_OWNER_DEFINITION_REGISTRY_V1`、各Registry ID／version、record count、logical IDのNFC UTF-8 bytes／version順へstrict sortした完成record bytesを各`uint32_be` length framingして計算し、各`registry_content_hash`だけを除外する。Intent／Envelope／Dimensionも同様に各自己hashだけを除くReceipt-free preimageを持つ。`migration` branchとMapping baseはwire designを予約するが、§9のsigned evidence gate成立前はsubject／base／Binding／Registryをmaterializeしない。

`WorkloadOwnerDefinitionRefV1`の全Fieldは同じRegistry rowとbyte equalityでなければならない。`definition_kind=workload_domain_owner`は`schema_kind=workload_domain_intent_v1`、`owner_intent_branch` canonical omission、`allowed_domain_type_refs`非空である。`owner_scale_intent_owner`はbranchと同名の`*_scale_intent_v1` schema、allowed Domain集合`[]`である。discriminator外schema／branch、Owner ID prefixからのschema推論、RegistryなしのDefinition refを拒否する。各`WorkloadDomainIntentV1.owner_definition_ref`はEnvelopeの`workload_owner_definition_registry_ref`と同Registryを指す`workload_domain_owner`で、そのallowed Domain集合へ当該`domain_type_ref`を含み、Intent Qualification subject ownerはDefinition rowの`owner_ref`とbyte equalityでなければならない。各`OwnerScaleIntentRefV1.intent_ref.owner_ref`はbranchごとに`world_owner_ref_v1 | asset_lifecycle_owner_ref_v1 | project_state_owner_ref_v1 | security_approval_owner_ref_v1`の対応するexact三Field、`owner_definition_ref`は`owner_scale_intent_owner`、branch／schema／同じexact Owner refがenclosing union branchとbyte equalityでなければならない。

`PerformanceQualificationSubjectRefV1.subject_kind`のwire schemaは上記五branchを予約するが、current materialization可能集合は`workload_domain | workload_intent | integrated_envelope | scale_dimension`のexact四branchである。`migration` branchのcurrent Subject／Receipt／Binding／Mapping集合はexact `[]`で、signed evidence gate成立後のatomic activationだけが五番目を有効化できる。discriminator外branch、ID／version／content hash欠落、別kind ref、activation前の`migration`を拒否する。Subject `owner_ref`はkind別に、`workload_domain`ではRefが解決する`WorkloadDomainTypeRecordV1.owner_ref`、`workload_intent`ではIntentの`owner_definition_ref`が解決する上記Registry rowの`owner_ref`、`integrated_envelope`では`performance_owner_ref_v1`、`scale_dimension`ではRefが解決する`RuntimeScaleIntentDimensionRecordV1.owner_ref`とbyte equalityにする。Activation後の`migration`はRefが解決する`ProjectScaleDomainMappingRecordV1.owner_ref`とbyte equalityにする。表示名、ID prefix、signer自己申告からownerを補完しない。Intent Kind registryは`intent_kind_id`／version順、Owner Definition Registryは`definition_id`／version順、Domain registry recordは`domain_type_id`／version順、Envelopeのintentとselected bindingはdomain type ID／intent ID／intent version順、owner intentは上記kind順へstrict sortし、duplicate、same-ID／version different-hash、owner偽装を拒否する。Domain Registryの`intent_kind_registry_ref`、Envelopeの`workload_intent_kind_registry_ref`、Envelopeが参照するDomain Registryの同Refはbyte equalityでなければならない。Envelope内の全Definition refのRegistryはEnvelopeの`workload_owner_definition_registry_ref`とbyte equalityである。Envelopeのintent集合、`selected_domain_record_bindings[]`、Registryから選択したrecord集合はdomain type ref／record hashでset equalityを必須にし、各bindingのRegistry refはEnvelopeの一件とbyte equalityでなければならない。これにより選択Domain、Intent Kind、Owner Definition rowの一Field変更はRegistryとEnvelope preimageを必ず変更する。

Qualification生成順はcurrent四subject kind、およびgate成立後の`migration`で`receipt-free base → base ref → PerformanceQualificationSubjectV1 → signed Receipt → PerformanceQualificationBindingV1 → root外Activation projection`である。`qualification_subject_hash`はASCII `MIRAKAN_PERFORMANCE_QUALIFICATION_SUBJECT_V1`、binding hashはASCII `MIRAKAN_PERFORMANCE_QUALIFICATION_BINDING_V1`、projection hashはASCII `MIRAKAN_PROJECT_SCALE_ACTIVATION_PROJECTION_V1`と各自己Fieldを除くcount／length-framed canonical bytesから計算する。Receipt refのID／version／subject hash／signed hashはwrapper内subject／signed recordとexact equalityで、BindingのsubjectはReceipt subjectのbase refとbyte equalityにする。Receipt／Binding／Projectionをbase、Registry、Envelope hashへ戻さない。

Projectionのdomain Binding subject集合はEnvelope `selected_domain_record_bindings[]`が解決するDomain ref集合、intent Binding subject集合はEnvelope `workload_domain_intents[]`からmaterializeした`WorkloadDomainIntentRefV1`集合、Dimension Binding subject集合は同Intent群の`dimension_values[].dimension_ref`のunionと、それぞれexact set equalityにする。`integrated_envelope_qualification_binding_ref`はexact一件で、そのBinding／Receipt subjectはProjectionの`scale_envelope_ref`とbyte equalityにする。四Binding fieldはsubject kindごとに分離し、各配列をsubject logical ID／version／content hash、Binding ID／version順へstrict sortし、duplicate、別kind、missing、extraを拒否する。全Bindingが同じProjectionのEnvelope closureから導出されたTarget集合を持つことを検証し、別Envelopeで有効なDomain／Intent／Dimension Bindingを混在させない。Production consumerはProjectionが指すsigned Receiptのsubject／result=`pass`／freshness／revocationだけを検証し、Fixture bodyを解決しない。current四branchそれぞれで正しいbase refのままSubject ownerだけを別の有効Ownerへ差し替えるfixture、Binding subjectだけを別baseへ差し替えるfixtureに加え、Domain／Intent／Dimension各集合のmissing／extra、cross-envelope Binding、integrated Bindingだけ別Envelope、canonical順序違反を各一原因で拒否する。`migration` branchの同等fixtureは§9のActivation Qualificationにだけ属する。

`spatial_requirement=required`のactive domainが一件以上なら`owner_intent_refs[]`にworld branchをexact一件必須、全active domainが`forbidden`ならworld branchを禁止する。`optional`が一件以上かつ`required`が0件ならworld branch有無の両方を許すが、存在時はexact World owner intentへ解決し、当該optional domainのspatial dimension closureへ含めなければならない。`forbidden` domainへspatial dimensionまたはWorld refを結び付けない。content／authoring／authority branchは選択Domain群の`required_intent_kind_refs[]`をEnvelope固定のIntent Kind Registryへ全件exact解決し、そのrecord集合が要求する`required_owner_intent_branch`ごとにexact一件を必須とする。同じbranchを複数kindが要求してもOwner intent refは一件だけで、どのkindも要求しないbranchは禁止する。各Domain recordの`authority_class`と`semantic_requirement_mode`は参照する全Intent Kind recordの各allowed集合へ含まれなければならない。discriminator外branch、同kind／branch重複、owner不一致、unknown／stale kind ref、bare ref、Kind Registry差し替えを拒否する。

`semantic_requirement_mode`が`authoritative_equivalence`ならequivalence policy、`presentation_fidelity`ならfidelity policy、`functional_contract`なら機能Requirement、`resource_slo`ならSLO Requirementを1件以上必須にする。`none`は`authority_class=tooling | resource_service`かつDomain recordが明示許可する場合だけ使用し、他branchのpolicy／requirementをcanonical omissionする。全Project共通のGameplay fidelity floorや最低Entity数を置かない。

初期Core registryは次の完全五Receipt-free recordだけを持つ。全rowは`domain_type_version=1`、exact `owner_ref=performance_owner_ref_v1`、表のtyped dimension ref、self-excluding content hashを持つ。`none` branchはtooling／resource serviceだけに許可する。

| domain type ID | authority／spatial | required intent kind refs | allowed dimension refs | semantic mode |
|---|---|---|---|---|
| `workload.core.authoritative_simulation` | `authoritative_simulation`／`optional` | `intent_kind.performance.authoritative_state@1` | `scale.dimension.instance.peak_active_authoritative@1; scale.dimension.lifecycle.peak_create_per_simulation_step@1; scale.dimension.lifecycle.peak_retire_per_simulation_step@1; scale.dimension.event.peak_authoritative_per_simulation_step@1` | `authoritative_equivalence` |
| `workload.core.presentation` | `presentation`／`forbidden` | `intent_kind.performance.presentation_fidelity@1` | `scale.dimension.instance.peak_visible@1; scale.dimension.presentation.peak_active@1` | `presentation_fidelity` |
| `workload.core.ui` | `ui`／`forbidden` | `intent_kind.performance.functional_contract@1` | `scale.dimension.instance.peak_live@1; scale.dimension.presentation.peak_active@1` | `functional_contract` |
| `workload.core.tooling` | `tooling`／`forbidden` | `[]` | `scale.dimension.instance.total_authored@1; scale.dimension.instance.peak_live@1` | `none` |
| `workload.core.resource_service` | `resource_service`／`forbidden` | `intent_kind.performance.resource_slo@1` | `scale.dimension.instance.total_authored@1; scale.dimension.instance.peak_live@1` | `resource_slo` |

初期Intent Kind Registryは次の完全四Receipt-free recordだけを持つ。全rowは`intent_kind_version=1`、exact `owner_ref=performance_owner_ref_v1`、表のbranch／allowed集合、self-excluding `intent_kind_content_hash`を持つ。Domain表の`@1` refはこの四recordのID／version／content hashとbyte equalityで、ID文字列に`@1`を含めない。

| intent kind ID | required owner-intent branch | allowed authority classes | allowed semantic modes |
|---|---|---|---|
| `intent_kind.performance.authoritative_state` | `authority` | `[authoritative_simulation]` | `[authoritative_equivalence]` |
| `intent_kind.performance.presentation_fidelity` | `content` | `[presentation]` | `[presentation_fidelity]` |
| `intent_kind.performance.functional_contract` | `authoring` | `[ui]` | `[functional_contract]` |
| `intent_kind.performance.resource_slo` | `content` | `[resource_service]` | `[resource_slo]` |

四record以外をCore Registryへ暗黙追加せず、extension ownerの追加recordは自身のexact owner ref、closed branch、非空allowed集合を持つ。Registry ref／record refのID、version、content hashの一Fieldmutation、四Core recordのmissing／extra／duplicate／noncanonical order、owner substitution、Domainが許可外authorityまたはsemantic modeを使うcase、必要owner-intent branchのmissing／extra／duplicate、Domain RegistryとEnvelopeのKind Registry ref差を一原因ずつrejectし、last-valid Envelope／Activation projectionを不変にする。

初期Owner Definition Registryのbuilt-in inventoryは次の完全九recordである。全rowは`definition_version=1`、表のexact owner ref／kind／schema／branch／allowed Domain集合、self-excluding content hashを持つ。最初の五recordはCore Domain用`workload_domain_owner`、後半四recordはcanonical owner intent用`owner_scale_intent_owner`である。

| definition ID | exact owner | kind／schema／branch | allowed Domain refs |
|---|---|---|---|
| `definition.performance.workload.authoritative_simulation` | `performance_owner_ref_v1` | `workload_domain_owner`／`workload_domain_intent_v1`／omitted | `[workload.core.authoritative_simulation@1]` |
| `definition.performance.workload.presentation` | `performance_owner_ref_v1` | `workload_domain_owner`／`workload_domain_intent_v1`／omitted | `[workload.core.presentation@1]` |
| `definition.performance.workload.ui` | `performance_owner_ref_v1` | `workload_domain_owner`／`workload_domain_intent_v1`／omitted | `[workload.core.ui@1]` |
| `definition.performance.workload.tooling` | `performance_owner_ref_v1` | `workload_domain_owner`／`workload_domain_intent_v1`／omitted | `[workload.core.tooling@1]` |
| `definition.performance.workload.resource_service` | `performance_owner_ref_v1` | `workload_domain_owner`／`workload_domain_intent_v1`／omitted | `[workload.core.resource_service@1]` |
| `definition.performance.owner_intent.world` | `world_owner_ref_v1` | `owner_scale_intent_owner`／`world_scale_intent_v1`／`world` | `[]` |
| `definition.performance.owner_intent.content` | `asset_lifecycle_owner_ref_v1` | `owner_scale_intent_owner`／`content_scale_intent_v1`／`content` | `[]` |
| `definition.performance.owner_intent.authoring` | `project_state_owner_ref_v1` | `owner_scale_intent_owner`／`authoring_scale_intent_v1`／`authoring` | `[]` |
| `definition.performance.owner_intent.authority` | `security_approval_owner_ref_v1` | `owner_scale_intent_owner`／`authority_scale_intent_v1`／`authority` | `[]` |

Extension Domainは同じRegistryへowner自身の`workload_domain_owner` recordを寄与し、allowed Domain ref集合をexactに宣言する。built-in九recordの一件missing／extra／duplicate、schema／branch／owner／allowed Domainの一Fieldmutation、Definition RefのRegistry／kind／ownerだけを別valid rowへ差し替えるcase、EnvelopeとIntentのRegistry ref不一致を一原因ずつrejectする。Registry／Definition refはReceipt-freeであり、Qualification Receipt／Binding／Activation projectionをhash preimageへ戻さない。

各row固定後に同logical suffixの`qualification.performance.workload_domain.core.*@1` subject／Receiptと`qualification_binding.performance.workload_domain.core.*@1`をroot外で一件ずつ作る。Activation projectionのdomain binding集合はEnvelopeが選択したDomain record集合とexact set equalityであり、§9のProjection四集合規則を同じvalidatorで検証する。

表の`@1`はexact version／content hashまたはContract set root付きrefで保存する。World ownerは必要なProjectに`workload.world.spatial`を寄与し、Feature／Genre／Project ownerは自身のdomainを寄与する。CoreはShooter、enemy、vehicle、level、quest、terrain等の語彙を登録しない。Network authority variantsは専用仕様、Threat Model、Product activation前は`not_activated`である。

`envelope_hash`はASCII `MIRAKAN_PROJECT_SCALE_ENVELOPE_V2`と自己Fieldを除く全Receipt-free Fieldのlength-framed canonical bytesから計算する。selected Registry row hash、exact Project triple、Target、owner intent branch、Decisionの一Fieldでも変われば別Envelopeである。Qualification Receipt／Binding／Activation projectionはEnvelope hash入力ではない。Production Envelope／Domain／Intent／Dimension recordはFixture bodyを解決せず、root外Activation projectionから署名済みQualification Receiptのsubject／result／freshness／revocationだけを検証する。表示用`scale_class`をSourceへ保存せず、Projectionはdomain closureとEnvelope hashから`compact_reference | medium_candidate | large_local_candidate | distributed_candidate`を決定的に導出する。Target readinessは[Project State §3.4](../03-authoring/project-state.md#34-target-readiness)の`TargetReadinessV1`をread-only投影し、`state`は`predicted | blocked | qualified`だけ、性能未達の理由は`blocked_reason_ref`だけに置く。

`ProjectScaleEnvelopeV1`のcanonical bytesとそれを使用する旧Projectが実在することは現計画からは証明されておらず、current Source、Editor、AI projection、Compile Manifest、deserializerへ登録しない。`operation.performance.migrate_project_scale_envelope@1`は[Executable Contracts §8.1.2](../02-foundation/executable-contracts.md#812-conditional-legacy-migration-evidence-gate)のconditional legacy migrationで、current状態は`not_activated`である。この移行に固有のcurrent MCD／Owner Manifest／Service allowlist／Policy／Validator／migration Diagnostic／Operation Receipt／Provider／MCP／alias、migration Qualification subject／Receipt／Binding、Activation Catalog／projection subset、`ProjectScaleDomainMappingRecordV1`／Registry、Migration Manifestはすべてexact `[]`である。`service.offline_project_migrator`、`capability.authoring.offline_migration`、`profile.isolation.offline_project_migrator`のcurrent集合もexact `[]`で、移行要求をdispatchしない。

将来Activationするには、実在する旧Envelope／Project bytes、旧MCD local record／common envelope／payload、source `ContractSetSnapshotV2`、Owner Identity Registry、Named Algorithm Registry、`FoundationDefinitionClosureV1`、全retained artifact ref／hash、正負fixtureを推測なしで列挙したsigned `LegacyMigrationInventoryV1`が§8.1.2 gateを満たさなければならない。その同じ承認済みContract set transactionだけが、Operation／Type／Policy／Validator／migration Diagnostic／Receipt、offline Service／Capability／Isolation Profile、Service allowlist、Mapping Registry／records、Qualification Receipt／Binding、Manifest、Provider／MCP projectionを完全closureとして同時にmaterializeできる。次のschemaとrecord値はpost-activation destination templateであり、block内の`status=active`、Service ref、Policy ref、Provider exposureをcurrent refまたはcurrent product surfaceとして解釈しない。

```text
ProjectScaleEnvelopeMigrationManifestV1
  manifest_id: performance.project_scale_envelope.migration.v1_to_v2
  manifest_version: 1
  operation_ref: McdContractRefV1(
    kind=operation,
    id=operation.performance.migrate_project_scale_envelope,
    version=1, contract_set_hash)
  input_type_ref: McdContractRefV1(
    kind=type, id=type.performance.project_scale_envelope_migration_input,
    version=1, contract_set_hash)
  output_type_ref: McdContractRefV1(
    kind=type, id=type.performance.project_scale_envelope_migration_result,
    version=1, contract_set_hash)
  receipt_type_ref: McdContractRefV1(
    kind=type, id=type.performance.project_scale_envelope_migration_receipt,
    version=1, contract_set_hash)
  precondition_policy_ref: McdContractRefV1(
    kind=policy, id=policy.operation.performance.scale_migration.precondition,
    version=1, contract_set_hash)
  postcondition_policy_ref: McdContractRefV1(
    kind=policy, id=policy.operation.performance.scale_migration.postcondition,
    version=1, contract_set_hash)
  rate_limit_policy_ref: McdContractRefV1(
    kind=policy, id=policy.authoring.performance_scale_migration.rate_limit,
    version=1, contract_set_hash)
  validator_closure_ref: OperationValidatorClosureRefV1
  trusted_service_ref: TrustedServiceRefV1(
    service_id=service.offline_project_migrator, service_version=1,
    service_content_hash)
  trusted_service_allowlist_operation_local_refs[1]:
    ContractSetLocalRefV1(
      kind=operation,
      id=operation.performance.migrate_project_scale_envelope,
      version=1)
  diagnostic_refs[12]: DiagnosticCodeRefV1
  qualification_binding_refs[1..64]:
    exact PerformanceQualificationBindingRefV1(
      subject_kind=migration)
  manifest_hash: SHA-256

operation.performance.migrate_project_scale_envelope@1
  MCD common envelope:
    mcd_version=1; kind=operation;
    id=operation.performance.migrate_project_scale_envelope;
    version=1; status=active;
    title=Migrate Project Scale Envelope V1 to V2;
    description=Atomically migrate one legacy five-axis scale envelope
      through exact owner-typed workload-domain mappings;
    owners=[owner.core.performance]; requirement_refs=[];
    rationale_refs=[mirakan.arch.runtime-performance-capacity#9-owner-typed-workload-scale-modelとprojectscaleenvelopev2];
    since_contract_set=2; supersedes=[];
    tags=[authoring,migration,performance]
  operation_kind: command
  input_type: McdContractRefV1(
    kind=type, id=type.performance.project_scale_envelope_migration_input,
    version=1, contract_set_hash)
  output_type: McdContractRefV1(
    kind=type, id=type.performance.project_scale_envelope_migration_result,
    version=1, contract_set_hash)
  authority: TrustedServiceRefV1(
    service_id=service.offline_project_migrator, service_version=1,
    service_content_hash)
  risk_class: R3
  side_effects: [authoring]
  transaction: authoring_changeset
  idempotency: idempotent_with_key
  preconditions:
    [McdContractRefV1(
      kind=policy,
      id=policy.operation.performance.scale_migration.precondition,
      version=1, contract_set_hash)]
  postconditions:
    [McdContractRefV1(
      kind=policy,
      id=policy.operation.performance.scale_migration.postcondition,
      version=1, contract_set_hash)]
  validator_closure_ref:
    {closure_id=validator_closure.operation.performance.scale_migration,
     closure_version=1, closure_content_hash}
  timeout_ms: 120000
  rate_limit_policy: McdContractRefV1(
    kind=policy,
    id=policy.authoring.performance_scale_migration.rate_limit,
    version=1, contract_set_hash)
  audit_level: full_redacted
  provider_exposure: mcp_proposal
  receipt_type: McdContractRefV1(
    kind=type, id=type.performance.project_scale_envelope_migration_receipt,
    version=1, contract_set_hash)
  errors[12]: exact DiagnosticCodeRefV1 records for
    diagnostic.conflict.revision_mismatch
    diagnostic.authorization.denied
    diagnostic.approval.required
    diagnostic.authoring.lock_conflict
    diagnostic.mcd.operation_predicate_invalid
    diagnostic.operation.timeout
    diagnostic.operation.rate_limit_exceeded
    diagnostic.operation.idempotency_key_reuse
    diagnostic.performance.scale_v1_source_invalid
    diagnostic.performance.workload_domain_unresolved
    diagnostic.performance.scale_migration_ambiguous
    diagnostic.performance.scale_receipt_binding_mismatch

ProjectScaleEnvelopeMigrationInputV1
  input_type_ref: McdContractRefV1(
    kind=type, id=type.performance.project_scale_envelope_migration_input,
    version=1, contract_set_hash)
  operation_ref: McdContractRefV1(
    kind=operation, id=operation.performance.migrate_project_scale_envelope,
    version=1, contract_set_hash)
  before_project_ref:
    exact {project_id, expected_project_revision, document_set_hash}
  operation_intent_hash
  request_hash
  idempotency_key
  source_foundation_definition_closure_ref:
    FoundationDefinitionClosureRefV1
  retained_source_mcd_ref: McdContractRefV1(
    kind=type, id=type.performance.project_scale_envelope,
    version=1, source_contract_set_hash)
  source_envelope_v1_ref/hash
  source_axis_and_intent_closure_hash
  destination_domain_registry_ref: WorkloadDomainTypeRegistryRefV1
  destination_owner_definition_registry_ref:
    WorkloadOwnerDefinitionRegistryRefV1
  destination_dimension_registry_ref: RuntimeScaleIntentDimensionRegistryRefV1
  domain_mapping_registry_ref: ProjectScaleDomainMappingRegistryRefV1
  selected_domain_mapping_refs[1..64]: ProjectScaleDomainMappingRefV1
  preview_policy_ref: McdContractRefV1(kind=policy)
  validation_policy_ref: McdContractRefV1(kind=policy)
  mutation_authorization_binding: exact MutationAuthorizationBindingV2(
    risk_class=R3, authority_evidence=approval)

ProjectScaleEnvelopeMigrationResultV1
  disposition: migrated | rejected
  migrated:
    before_project_ref
    after_project_ref
    destination_envelope_v2_ref/hash
    destination_domain_registry_ref/hash
    destination_owner_definition_registry_ref/hash
    destination_dimension_registry_ref/hash
    preview_receipt_ref/hash
    validation_receipt_ref/hash
    public_publication_marker_ref/hash
    migration_receipt_ref/hash
  rejected:
    diagnostics[1..12]: DiagnosticCodeRefV1

PreparedProjectScaleEnvelopeMigrationReceiptPayloadV1
  publication_binding: exact PreparedReceiptPublicationBindingV1
  operation_ref
  operation_intent_hash
  request_hash
  idempotency_key
  before_project_ref
  after_project_ref
  source_foundation_definition_closure_ref:
    FoundationDefinitionClosureRefV1
  retained_source_mcd_ref: McdContractRefV1(
    kind=type, id=type.performance.project_scale_envelope,
    version=1, source_contract_set_hash)
  source_envelope_v1_ref/hash
  destination_envelope_v2_ref/hash
  destination_domain_registry_ref/hash
  destination_owner_definition_registry_ref/hash
  destination_dimension_registry_ref/hash
  domain_mapping_registry_ref/hash
  selected_domain_mapping_refs[1..64]: ProjectScaleDomainMappingRefV1
  omitted_legacy_axis_refs[0..5]
  preview_receipt_payload_ref/hash
  validation_receipt_payload_ref/hash
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1
  diagnostics[0..12]: DiagnosticCodeRefV1
  prepared_payload_hash

ProjectScaleEnvelopeMigrationReceiptV1
  published_receipt:
    exact PublishedDomainReceiptV2 whose
    prepared_domain_receipt_payload_ref/hash resolves
    PreparedProjectScaleEnvelopeMigrationReceiptPayloadV1

ProjectScaleDomainMappingRefV1
  mapping_id
  mapping_version: positive uint32
  mapping_content_hash: SHA-256

ProjectScaleDomainMappingRecordV1
  mapping_id
  mapping_version: positive uint32
  owner_ref: OwnerIdentityLocalRefV1
  source_axis_ref/hash
  source_intent_predicate_ref: McdContractRefV1(kind=policy)
  destination_domain_type_ref: WorkloadDomainTypeRefV1
  destination_owner_definition_ref: WorkloadOwnerDefinitionRefV1
  destination_intent_schema_ref: McdContractRefV1(kind=type)
  semantic_field_mapping_policy_ref: McdContractRefV1(kind=policy)
  mapping_content_hash: SHA-256

ProjectScaleDomainMappingRegistryRefV1
  registry_id
  registry_version: positive uint32
  registry_content_hash: SHA-256

ProjectScaleDomainMappingRegistryV1
  registry_id: performance.project_scale_domain_mapping.registry.active
  registry_version: 1
  records[1..4096]: ProjectScaleDomainMappingRecordV1
  registry_content_hash: SHA-256
```

Activation後の`ProjectScaleDomainMappingRecordV1.mapping_content_hash`はASCII `MIRAKAN_PROJECT_SCALE_DOMAIN_MAPPING_RECORD_V1`と自己Fieldを除くReceipt-free length-framed canonical bytes、Registry hashはASCII `MIRAKAN_PROJECT_SCALE_DOMAIN_MAPPING_REGISTRY_V1`、Registry ID／version、record count、mapping ID／version順の全Receipt-free record bytesから計算して自己Fieldを除く。Ref三FieldはRegistry内のexact一件へ解決し、selected ref集合はRegistry member subset、source axis closure、destination Domain intent集合とset equalityでなければならない。各Mappingの`destination_owner_definition_ref`は`definition_kind=workload_domain_owner`、fixed destination Owner Definition Registry、allowed Domain集合へ同recordの`destination_domain_type_ref`を含み、Definition rowの`owner_ref`はMappingの`owner_ref`とbyte equalityである。生成する`WorkloadDomainIntentV1.owner_definition_ref`は選択Mappingの同Refをそのまま保存し、同じDomainを許可する別DefinitionやID prefixから再選択しない。Migration inputの`destination_owner_definition_registry_ref`は生成するEnvelopeの`workload_owner_definition_registry_ref`、Result／Prepared payloadの同Registry ref／hashとbyte equalityであり、全destination Intent／Owner intentのDefinition refはこの固定Registryだけへ解決する。destination Intent Kind Registryはinputのexact `destination_domain_registry_ref`が解決する`WorkloadDomainTypeRegistryV1.intent_kind_registry_ref`から決定し、生成Envelopeの`workload_intent_kind_registry_ref`とbyte equalityにする。独立したdestination-head lookupを行わない。Mapping Registry確定後、各Mapping refを`PerformanceQualificationSubjectV1(subject_kind=migration)`で署名し、root外`PerformanceQualificationBindingV1`を作る。Migration Manifestのbinding集合と選択Mapping集合はexact set equalityで、Receipt／BindingをMapping／Registry hashへ戻さない。0件／複数match、same source predicateへの複数active mapping、noncanonical sort、owner／policy／Domain／Definition／Type／Qualification BindingまたはReceipt hash mismatchを全migration rejectにする。

Activation後のlogical Operation ID候補はversion-neutral `operation.performance.migrate_project_scale_envelope`だけであり、同じatomic transactionのMCD／Manifest／Service allowlistへ登録する。現時点では三集合ともexact `[]`である。レビュー対象の旧綴り`operation.performance.migrate_project_scale_envelope_v1_to_v2`は一度もActivation／materializationされていない計画上の名前であり、alias、redirect、current refを作らない。将来、signed Inventoryが別の実在source spellingを証明した場合だけ、Tool catalog外のoffline alias migration recordとしてそのexact spellingを同じactivation transactionへ保存できる。

Activation後にOperationが参照する三Policyのdestination値は次の完全なactive MCD recordである。表の`status=active`はatomic activation完了後の値だけを表し、current Policy集合はexact `[]`である。表の共通Envelope列とpayload列を連結した値がrecord全体であり、別段落の既定値、bare ID、説明からFieldを補完しない。

| Policy MCD共通Envelope exact value | Policy payload exact value |
|---|---|
| `mcd_version=1; kind=policy; id=policy.operation.performance.scale_migration.precondition; version=1; status=active; title=Performance Scale Migration Precondition; description=Validate the exact V1 source envelope, Project base, workload registries, selected owner mappings, authorization, and idempotency snapshot without mutation; owners=[owner.core.performance]; requirement_refs=[]; rationale_refs=[mirakan.arch.runtime-performance-capacity#9-owner-typed-workload-scale-modelとprojectscaleenvelopev2]; since_contract_set=2; supersedes=[]; tags=[operation_predicate,performance,pure]` | `evaluation_mode=pure; side_effects=[]; input_type={id=type.operation.precondition_evaluation_input,version=1,contract_set_hash}; result_type={id=type.operation.predicate_result,version=1,contract_set_hash}` |
| `mcd_version=1; kind=policy; id=policy.operation.performance.scale_migration.postcondition; version=1; status=active; title=Performance Scale Migration Postcondition; description=Validate the unpublished V2 envelope, owner mapping closure, prepared Receipt payload, and atomic Project revision increment; owners=[owner.core.performance]; requirement_refs=[]; rationale_refs=[mirakan.arch.runtime-performance-capacity#9-owner-typed-workload-scale-modelとprojectscaleenvelopev2]; since_contract_set=2; supersedes=[]; tags=[operation_predicate,performance,pure]` | `evaluation_mode=pure; side_effects=[]; input_type={id=type.operation.postcondition_evaluation_input,version=2,contract_set_hash}; result_type={id=type.operation.predicate_result,version=1,contract_set_hash}` |
| `mcd_version=1; kind=policy; id=policy.authoring.performance_scale_migration.rate_limit; version=1; status=active; title=Performance Scale Migration Rate Limit; description=Bound migration requests per Project without changing migration semantics; owners=[owner.core.performance]; requirement_refs=[]; rationale_refs=[mirakan.arch.runtime-performance-capacity#9-owner-typed-workload-scale-modelとprojectscaleenvelopev2]; since_contract_set=2; supersedes=[]; tags=[authoring,performance,rate_limit]` | `policy_ref={id=policy.authoring.performance_scale_migration.rate_limit,version=1,contract_set_hash}; scope=project; window_ns=60000000000; max_requests=4; burst=1; exceeded_error_ref={diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash}` |

Activation destination Contract set内部では三Policyを`ContractSetLocalRefV1(kind=policy)`へ投影し、self refはlocal identityだけにする。Manifest `precondition_policy_ref`／`postcondition_policy_ref`／`rate_limit_policy_ref`、Operation三ref、Performance ownerのPolicy local subsetはexact三件でset equalityである。三recordの共通Envelopeまたはpayloadの実在Fieldを一つだけ変えるfixtureはPolicy member hashとset rootを変更し、旧Manifest／Operation external refを解決不能にする。

Activation後のmigration固有Diagnostic destination値は次の完全な`DiagnosticLocalRecordV2`である。current migration Diagnostic集合はexact `[]`である。全rowは`diagnostic_version=1`、`owner_local_ref=performance_owner_ref_v1`、`requirement_local_refs=[]`、`message_key="<diagnostic_id>.message"`、Ownerを含むself-excluding `diagnostic_local_content_hash`を持つ。root確定後だけ同じ三Field Owner ref、`requirement_refs=[]`、別のself-excluding `diagnostic_content_hash`を持つ外部Registry recordへ投影する。共通八件はActivation先Foundationの共通recordを参照し、本書のcurrent Diagnosticへ複写しない。

| Diagnostic ID | code | severity／category／retryability |
|---|---|---|
| `diagnostic.performance.scale_v1_source_invalid` | `MIRAKAN-PERFORMANCE-SCALE-V1-SOURCE-INVALID` | blocking／schema／after_input |
| `diagnostic.performance.workload_domain_unresolved` | `MIRAKAN-PERFORMANCE-WORKLOAD-DOMAIN-UNRESOLVED` | blocking／semantic／after_change |
| `diagnostic.performance.scale_migration_ambiguous` | `MIRAKAN-PERFORMANCE-SCALE-MIGRATION-AMBIGUOUS` | blocking／semantic／after_input |
| `diagnostic.performance.scale_receipt_binding_mismatch` | `MIRAKAN-PERFORMANCE-SCALE-RECEIPT-BINDING-MISMATCH` | blocking／semantic／after_change |

Activation後の`validator_closure.operation.performance.scale_migration@1` destinationは次のexact Validator recordで閉じる。current Validator／Validator closure集合はexact `[]`である。各recordはversion 1、実装Artifact ref／hash、表のinput Type LocalRef、表のDiagnostic LocalRef、self-excluding content hashを持つ。

| Validator | input | exact reachable Diagnostic |
|---|---|---|
| `validator.operation.request_envelope` | migration input | idempotency key reuse |
| `validator.operation.authorization` | migration input | authorization denied |
| `validator.operation.approval` | migration input | approval required |
| `validator.operation.revision_and_lock` | migration input | revision mismatch; lock conflict |
| `validator.operation.pure_predicate` | migration input | operation predicate invalid |
| `validator.operation.timeout_and_rate_limit` | migration input | timeout; rate limit exceeded |
| `validator.performance.scale_migration_semantics` | migration input | V1 source invalid; workload domain unresolved; migration ambiguous |
| `validator.performance.scale_migration_postcondition` | postcondition input v2 | Receipt binding mismatch |

wrong-kind、stale version／Contract set／content hash、impure policy、rate payload mismatch、Validator Artifact／input／error mismatchをManifest compile前に拒否する。

Activation Qualificationでは旧五axisを名前だけで自動変換しない。World axisはWorld Document／intentが実在する場合だけ`workload.world.spatial`、populationは対応owner mapping recordが一意な場合だけそのdomain、content／authoring／authorityは各Owner intentへ解決する。UI-only、headless tool、resource-only fixtureはWorld／Gameplay domain 0件のV2へ変換できなければならない。共通四fixture bodyは各選択Mappingをsubjectにする別`PerformanceQualificationSubjectV1(subject_kind=migration).fixture_refs[]`のexact四件としてだけ解決する。Activation後のProduction Manifestの`qualification_binding_refs[1..64]`は選択Mapping ref集合とexact set equalityで、各Bindingは一つのMapping refをsubjectにする署名済みReceiptへexact解決する。ManifestはReceipt refを直接保持せず、Fixture bodyを解決しない。Mapping数とfixture数を等置せず、1件または5～64件のvalid Mapping closureも同じ四fixtureを各subjectで検証できなければならない。

Activation後のInputの`source_foundation_definition_closure_ref`は`retained_source_mcd_ref={type.performance.project_scale_envelope,1,source_contract_set_hash}`、signed Inventoryが列挙したV1 Envelope record、そのOwner ref、同時代Named Algorithm Registryをexact source Closureへ解決する。InputとPrepared payloadのsource Closure refおよび`retained_source_mcd_ref`はbyte equalityで、Operation／input／output／Policy／Validator／Diagnostic、全destination Registry／Mapping policy／V2 schema、request Algorithm bindingはdispatch時のdestination Foundation Closureだけへ解決する。両Fieldはoperation intentのsemantic inputとPublic Receiptから到達するPrepared payloadへ保持し、missing、別名、wrong source root、同じContract Setの別Owner／Algorithm root、input–Receipt差、sourceをdestinationへalias、destination refのsource downgradeを一原因ずつrejectしてSourceと既存idempotency resultを不変にする。

Activation Qualificationは[Executable Contracts §8](../02-foundation/executable-contracts.md#8-operation定義)のcanonical publicationを再利用し、`Source → Preview → Validation → Prepared payload → private Marker read-back → secret-free PublicCommitClosureV1 candidate → signed wrapper read-back → PublicCommitClosureV1＋PublicPublicationMarkerV1＋after Projectのatomic CAS → reload → Compile`を検証する。Closureの`domain_commitment.kind`は`owner_typed_state_commit`、`domain_owner_ref`はexact `performance_owner_ref_v1`、committed artifact集合はPrepared payloadが束縛したreceipt-free artifact ref集合とし、Closure Ref／hashの生成・比較は同節をそのまま使って本書で別式を定義しない。0件／複数mapping、偽World生成、Gameplay floor捏造、partial migrationをrejectする。destination Owner Definition Registry ref／version／hashの一Fieldだけをinput、Envelope、Result、Prepared payloadのいずれかで差し替えるcase、Domain Registry内Intent Kind Registry refとEnvelopeのrefをずらすcase、retry時にRegistry driftから別Envelopeを生成するcaseを一原因ずつrejectし、Sourceと既存idempotency resultを不変にする。Operation `errors[]`、Validator reachable errors、Manifest `diagnostic_refs[]`は上記12件のID／code／version／content hashでset equalityにする。ManifestのOperation LocalRef集合と`service.offline_project_migrator`へのallowlist contributionはexact一件でset equalityとし、同じatomic activation transactionでService local recordとset rootを生成する。Prepared payload Typeはexact `type.performance.prepared_project_scale_envelope_migration_receipt_payload@1`、hashはASCII `MIRAKAN_PREPARED_PROJECT_SCALE_ENVELOPE_MIGRATION_RECEIPT_PAYLOAD_V1`とself-excluding canonical bytesから計算する。最終Receipt Typeと相互代用しない。唯一のsigned subject／wrapperはExecutable Contractsの`PublishedDomainReceiptPayloadV2`／`PublishedDomainReceiptV2`とする。Domain固有Subject／alternate wrapperを作らず、Closure bodyまたは同Closureを束縛するsigned wrapperを欠くPublic Marker／after-state current authorityを拒否する。同じidempotency key＋request hashのretryはbyte-identical Result／`PublicCommitClosureV1`／signed Receipt／Public Markerを返し、同じkey＋別requestはidempotency reuse errorでSourceを変更しない。

Project固有の同時workload製品Envelopeは現時点で未校正であり、数値を仮定しない。この項目のOwnerは本書、readiness envelopeのOwnerはProject Stateである。Target Profileごとに次の入力が揃うまでは`state=blocked`、`blocked_reason_ref=performance_envelope_unqualified`を返す。

1. CPU世代／core、RAM、storage、GPU／driver、OS、Device generationを固定した実機Target Profile。
2. `ProjectScaleEnvelopeV2`が参照する全`WorkloadDomainIntentV1`について、選択domainが登録したinstance、lifecycle、event、spatial、presentation、UI、tool、resource dimensionのboundを数値化したProject Requirement。
3. その数値を丸めず同時発生させる`IntegratedScaleFixtureV1`とcanonical input trace。
4. Source、Contract、Toolchain、Target、Device、Quality、Representation Planを束ねた同一`input_closure_hash`。
5. §8／§13のcorrectness、Replay、memory、hitch、fault、10分×3 run、2時間enduranceを通過したfresh `policy.evidence.target-device.v1` Technical Qualification Receipt。

上記Receiptが同じclosureでfreshな場合だけ`qualified`へ遷移できる。安全なRepresentation Planは作れるが製品Envelopeとは別の小規模入力を測定しただけなら、その小規模入力closureに限り`predicted`または`qualified`を判定し、製品workload closure全体へ外挿しない。Mobile commonが所有するbaseline pixel／render budget表は変更せずTarget Profile入力として保持するが、それ単独で他domainのreadinessを解除しない。

Scale dimensionは次の型付きregistryで所有する。CoreはGenre／object role／event名を列挙せず、Feature Pack、Genre Pack、Projectが自身の語彙を同じcontractへ寄与する。

```text
RuntimeScaleIntentDimensionRefV1 {
  dimension_id,
  dimension_version,
  dimension_content_hash
}

PerformanceScaleUnitSemanticRefV1 {
  unit_id,
  unit_version,
  unit_content_hash
}

PerformanceScaleUnitSemanticRecordV1 {
  unit_id,
  unit_version,
  owner_ref: OwnerIdentityLocalRefV1,
  quantity_kind: count | distance,
  canonical_scalar_encoding: uint64_be | ieee754_binary64_be,
  canonical_base_unit: one_count | si.meter,
  canonical_symbol: count | m,
  conversion_to_canonical: exact rational scale and offset,
  value_constraint: nonnegative_integer | nonnegative_finite,
  unit_content_hash
}

PerformanceScaleUnitSemanticRegistryRefV1 {
  registry_id,
  registry_version,
  registry_content_hash
}

PerformanceScaleUnitSemanticRegistryV1 {
  registry_id,
  registry_version,
  record_count,
  records[2..256],
  registry_content_hash
}

RuntimeScaleIntentDimensionRecordV1 {
  dimension_ref,
  owner_ref: OwnerIdentityLocalRefV1,
  measurement_schema_ref: McdContractRefV1(kind=type),
  unit_ref: PerformanceScaleUnitSemanticRefV1,
  authority_class,
  fidelity_contract_ref?: McdContractRefV1(kind=policy),
  semantic_equivalence_contract_ref?: McdContractRefV1(kind=policy),
}

RuntimeScaleIntentDimensionRegistryRefV1 {
  registry_id,
  registry_version,
  registry_content_hash
}

RuntimeScaleIntentDimensionRegistryV1 {
  registry_id,
  registry_version,
  unit_registry_ref: PerformanceScaleUnitSemanticRegistryRefV1,
  records[1..4096],
  registry_content_hash
}
```

Performance Scale Unit Semantic Registry候補はRuntime scale envelopeで使う量のwire意味だけを所有し、Math型、Editor表示単位またはPack固有の換算表を所有しない。logical ID候補は`registry.performance.scale_unit_semantic`、初期候補は`registry_version=1`、`record_count=2`である。`registry_content_hash`と各record hashはArtifact materialization時に生成し、Markdownへcurrent値として埋め込まない。初期候補recordは次の二件で、両方の`owner_ref`はresolved `performance_owner_ref_v1`、`conversion_to_canonical={scale_numerator="1", scale_denominator="1", offset_numerator="0", offset_denominator="1"}`とする。

| `unit_id` | version | quantity／wire／base／symbol | value constraint | `unit_content_hash` |
|---|---:|---|---|---|
| `unit.count` | 1 | `count`／`uint64_be`／`one_count`／`count` | `nonnegative_integer` | generated |
| `unit.meter` | 1 | `distance`／`ieee754_binary64_be`／`si.meter`／`m` | `nonnegative_finite` | generated |

`unit_content_hash`候補はASCII `MIRAKAN_PERFORMANCE_SCALE_UNIT_SEMANTIC_RECORD_V1`と、当該hash Fieldだけを除くclosed recordのRFC 8785 JCS UTF-8 bytesを`uint32_be` length framingしてSHA-256する。JCS projectionではkeyをUnicode code point順、全stringをNFC、`unit_version`をsafe JSON integer、`owner_revision`とrational四値をcanonical unsigned decimal string、SHA-256をlowercase hexadecimal exact 64文字にする。`registry_content_hash`候補はASCII `MIRAKAN_PERFORMANCE_SCALE_UNIT_SEMANTIC_REGISTRY_V1`と、当該hash Fieldだけを除き`records[]`を`unit_id`のNFC UTF-8 bytes／version／content hash順へstrict sortしたclosed Registryの同じJCS framingから計算する。Artifact生成後はpreimageから再計算できることを検証し、ID-only、表示symbol、latest versionまたは別Registry rootによる補完を禁止する。将来単位を追加する場合は新Registry versionとrecord hashを発行し、Project／Pack固有表示変換は別Owner contributionとしてQualificationする。

`dimension_id`はowner namespaceを含むversion非依存logical ID、`dimension_version`は正の`uint32`とする。`dimension_content_hash`はASCII `MIRAKAN_RUNTIME_SCALE_INTENT_DIMENSION_RECORD_V1`と、当該hash Fieldだけを除くReceipt-free Record canonical MCD bytesを`uint32_be` length framingしてSHA-256する。`authority_class`は`authoritative_state | authoritative_event | presentation_only | resource_only`のclosed enumとする。`records`は`dimension_id`のUTF-8 byte昇順、同一IDまたは同一Ref重複を拒否し、RefはRegistry内でちょうど一件へ解決する。`authoritative_state | authoritative_event`は`semantic_equivalence_contract_ref`必須、`presentation_only`は`fidelity_contract_ref`必須、該当しないoptionalはcanonical omissionする。Dimension Registry確定後に各Dimension refを`PerformanceQualificationSubjectV1(subject_kind=scale_dimension)`へbindし、signed Receiptとroot外Qualification Bindingを作る。Receipt／BindingをDimension record／Registryへ戻さない。

Registryのlogical ID候補は`registry.performance.runtime_scale_intent_dimension`、initial `registry_version=1`とする。`unit_registry_ref`は完成Artifactの`{registry.performance.scale_unit_semantic, 1, <generated hash>}`へ解決する。`registry_content_hash`はASCII `MIRAKAN_RUNTIME_SCALE_INTENT_DIMENSION_REGISTRY_V1`、Registry ID／version、unit Registry ref、record count、strict sort済み全Record canonical bytesを各`uint32_be` length framingしてSHA-256し、自己Fieldを除外する。`RuntimeScaleIntentDimensionRegistryRefV1`は三Fieldすべてを同一active Registryへexact解決し、ID-only、latest version、hash fallbackを許可しない。

Core-owned初期Record候補は次の九件とする。表の`count-bound`は`measurement_schema_ref={type.performance.bounded_count, version=1, <Contract set hash>}`／`unit_ref={unit.count, 1, <generated hash>}`、`distance-bound`は`{type.performance.bounded_distance, version=1, <Contract set hash>}`／`unit_ref={unit.meter, 1, <generated hash>}`を表す。全Receipt-free Recordの`owner_ref`はmaterialize時にresolved `performance_owner_ref_v1`へ束縛し、文書revision／文書content hashまたはactive Ownerの暗黙追従値へ置換しない。

| `dimension_id` | schema | `authority_class` | required contract |
|---|---|---|---|
| `scale.dimension.instance.total_authored` | count-bound | `resource_only` | optional refs omitted |
| `scale.dimension.instance.peak_live` | count-bound | `resource_only` | optional refs omitted |
| `scale.dimension.instance.peak_active_authoritative` | count-bound | `authoritative_state` | `policy.performance.authoritative-state-equivalence` |
| `scale.dimension.instance.peak_visible` | count-bound | `presentation_only` | `policy.performance.presentation-fidelity` |
| `scale.dimension.lifecycle.peak_create_per_simulation_step` | count-bound | `authoritative_state` | `policy.performance.authoritative-state-equivalence` |
| `scale.dimension.lifecycle.peak_retire_per_simulation_step` | count-bound | `authoritative_state` | `policy.performance.authoritative-state-equivalence` |
| `scale.dimension.event.peak_authoritative_per_simulation_step` | count-bound | `authoritative_event` | `policy.performance.authoritative-event-equivalence` |
| `scale.dimension.spatial.maximum_interaction_radius` | distance-bound | `authoritative_state` | `policy.performance.authoritative-state-equivalence` |
| `scale.dimension.presentation.peak_active` | count-bound | `presentation_only` | `policy.performance.presentation-fidelity` |

九Dimension refに対応するroot外Qualification Bindingは、同じ意味を共有するReceiptを許可してもentryごとにexact subject refを持つ。Activation projectionの`scale_dimension_qualification_binding_refs[]`が解決するsubject集合と、Envelope全Intentの`dimension_values[].dimension_ref` unionをexact set equalityにし、subject ref／Binding refのcanonical順とduplicate不在を検証する。count-bound等のReceipt IDだけから複数baseへ暗黙展開せず、Dimension missing／extra／別Envelope substitutionを一原因fixtureで拒否する。

三policy refは本書のexact revision／content hashを持ち、`authoritative-state-equivalence`は§12のstate／random-stream gate、`authoritative-event-equivalence`はowner schemaごとのevent ID／apply step／canonical payload hash／ordering equality、`presentation-fidelity`はowner fixtureのvisual／audio tolerance、critical cue、timing floorを要求する。これらを説明名だけのpolicy、別revisionの同ID、Genre固有の暗黙比較へ置換しない。

Project固有のunit群、役割別instance数、イベント別peak、車両／群衆／弾体等のobject分類はCore enumへ追加せず、所有Pack／Projectが新しいRecord、measurement schema、fidelity／equivalence、別Qualification record／signed Receiptを一緒に登録する。Production recordはFixture bodyへ依存しない。登録されていないdimension、owner不一致、schema不一致、hash不一致は`ambiguous requirement`として拒否する。

```text
BoundedScaleQuantityV1 {
  unit_ref,
  minimum_required,
  target_value,
  maximum_expected
}

RuntimeScaleIntentDimensionValueV1 {
  dimension_ref: RuntimeScaleIntentDimensionRefV1,
  measurement_schema_ref: McdContractRefV1(kind=type),
  quantity: BoundedScaleQuantityV1
}

LegacyRuntimeScaleIntentV1 {
  intent_id: UUIDv7 StableId,
  schema_version: uint32,
  owner_definition_ref:
    exact {definition_id, definition_version, definition_content_hash},
  dimension_values[1..64],
  target_profile_refs[1..16]: exact Target Profile refs,
  fidelity_floor_refs[1..64]: McdContractRefV1(kind=requirement),
  semantic_equivalence_refs[0..64]: McdContractRefV1(kind=policy),
  reference_fixture_refs[1..64]:
    exact {fixture_id, fixture_version, fixture_content_hash},
  intent_content_hash: SHA-256
}
```

`LegacyRuntimeScaleIntentV1`は§9のpost-activation source schema templateで、current retained artifact／Source／Editor／AI／Compile／Save／Replay／Registry集合はexact `[]`である。signed `LegacyMigrationInventoryV1`が旧`ProjectScaleEnvelopeV1`とIntent bytesを束縛するまでoffline migration inputとして受理しない。旧logical type `RuntimeScaleIntentV1`をcurrent aliasとしてdeserializeせず、Activation後も`ProjectScaleDomainMappingRecordV1`がexact一件対応する場合だけcanonical `WorkloadDomainIntentV1` candidateへ変換する。新しいauthorityはdomain-discriminated `WorkloadDomainIntentV1`だけである。

current `WorkloadDomainIntentV1.dimension_values[]`はEnvelopeのexact Dimension Registryへ`dimension_ref`を解決する。Activation後のlegacy `dimension_values`はsigned Inventoryが束縛したsource Envelopeのexact Dimension Registryだけへ解決する。各Valueの`measurement_schema_ref`は解決先`RuntimeScaleIntentDimensionRecordV1.measurement_schema_ref`、`quantity.unit_ref`は同recordの`unit_ref`と全Field byte equalityで、さらにDimension RegistryがpinしたPerformance Scale Unit Semantic Registry内のexact一件へ解決しなければならない。三quantity値はそのmeasurement schemaでcanonical decode後に再encodeして入力canonical bytesと一致し、同じsemantic typeのfiniteかつ非負値、`minimum_required <= target_value <= maximum_expected`であることを検証する。schemaがcountならinteger count、distanceならfinite SI meterというrecord固有制約も同じdecoderが適用し、unit表示名やdimension IDから変換・補完しない。

配列は`dimension_id`のNFC UTF-8 byte／version／content hash順でstrict sortし、同一dimensionを拒否する。正しいDimension refのままcount schemaをdistance schemaへ、meter unitをcount unitへ、quantityの一値を非canonical encodingへ、Registryだけを別valid versionへ差し替えるfixtureをcurrent Envelopeで一原因ずつrejectする。legacy側の同等fixtureはActivation Qualificationにだけ属し、migration Sourceとlast-valid Resultを不変にする。Activation後のlegacy `intent_content_hash`はASCII `MIRAKAN_RUNTIME_SCALE_INTENT_V1`と、自身だけを除いた全Fieldのcanonical MCD bytesを`uint32_be` length framingして検証するが、新規生成しない。Target／fidelity／Fixture refはmigration Qualificationだけが読み、destination Production intentへFixture bodyをコピーせずsigned Receiptへ置換する。unknownを0、最大値、空optional、無制限へ補正しない。

World extent／coordinate／cell／streaming fieldは[World](../06-rendering/world.md)、LOD strategy／predicate／transition fieldは[LOD](../06-rendering/lod.md)、Authoring writer／Document／ChangeSet fieldは[Project state](../03-authoring/project-state.md)、content／build／cook fieldは[Asset lifecycle](../03-authoring/asset-lifecycle.md)と[Core architecture](../02-foundation/core-architecture.md)が所有する。本書はそれらをEnvelopeへexact refで束ねるだけで、field listを複写しない。

Envelope変更は通常の`ProjectChangeSetV1`であり、Before／After axisとdimension値、Target、Capability、Artifact、fixture、Decision closure、fidelity差分、再Cook／再Qualification、Save／Replay互換性、last-valid rollback refを必要とする。owner-typed authoritative instance／event bound、semantic-equivalence requirement、registered collision evidence、World範囲等を下げる変更は性能最適化ではなくGame behavior changeとして人間承認を必要とする。

## 10. Canonical sourceとDomain resolver

Scaleの四層を分離する。

| layer | content | mutation |
|---|---|---|
| Canonical Source | Requirement、World、Entity、Asset metadata、GameplayDefinition、Scale Intent、Decision | ChangeSetだけ |
| Derived Plan | streaming、representation、residency、LOD、cook work | direct edit禁止 |
| Runtime State | generation、queue、active／resident set、chunk | Runtime ownerだけ |
| Evidence | Trace、Benchmark、Diff、Explanation、Qualification | append-only、Governance envelope参照 |

Runtime StateまたはEvidenceからSourceへ値を自動write-backしない。Stable IDはrename、repartition、recook、HLOD、instance化、Simulation LODで変えない。Runtime handle、vendor ID、cell-local／plan-local IDをSource／Saveへ保存しない。Save／Replayはexact Source、Contract、Envelope、Plan hashを使う。

万能な`ScaleManager`を作らない。各Domainは同じEnvelopeと自身のIntentを読み、自身のDerived Planを所有する。`ScalePlanSetV1`はplan本文を埋め込まず、exact Artifact ref、Source revision、Target Profile、Capability signature、dependency edge、`TargetReadinessV1` refを束ねるmanifestである。required planのSource／Contract／Target／Capability hashが一件でもstale、missing、unqualifiedなら新setをpublishせずlast-valid playable setを維持する。

Population resolverはFull Entity、pool、archetype／SoA、instanced Presentation、reduced-frequency simulation、dormant record、aggregate simulation、HLOD、presentation-effect Artifactからclosed strategyを選ぶ。各strategyはentry／exit predicate、owner、state／Save mapping、recovery、fallback、budgetを持つ。distance／visibilityだけでauthoritative instanceを削らない。

## 11. AI scale action候補とbounded explanation

`search | read_envelope | dependencies | resolve_preview | explain_plan | propose_envelope_change | validate_transition`はStable IDでないplanned semantic action vocabularyであり、registered MCD Operationではない。Performance ownerのcurrent MCD Operation集合はexact `[]`である。`operation.performance.migrate_project_scale_envelope@1`は§9のdestination templateを持つconditional legacy migration exact一件、Scale AI actionは別の未Activation候補であり、どちらもcurrent Operationへ数えない。Scale AI actionのcurrent MCD／Owner Manifest／Service allowlist／Policy／Validator／Diagnostic／Receipt／Provider／MCP／alias集合はすべて`[]`、Capability stateは`not_activated`とし、future work item `activation.performance.scale_ai_operations.v1`が採用するexact ID集合と完全closureを一transactionで登録するまでdispatchしない。将来ActivationしたQuery／read／explainは[AI Security／Approval](../01-governance/ai-security-approval.md)が許可するread-only範囲、preview／changeは同OwnerのRisk／Approvalを消費する。本書はRisk値を再定義しない。

ProviderへProject Commit、Plan write、Capability activation、baseline緩和、Source直接write、server authority移動を公開しない。Queryはrevision、Envelope hash、Target、Capability signature、index revision、query hash、selected items、omitted ranges、cursor、Governance Evidence refを返す。全World／全Project dumpを行わない。

`ScaleExplanationReceiptV1`はSource revision、Envelope hash、Target、Plan set hash、selected／rejected closed strategy、fidelity proof ref、cost measurement ref、fallback chain、`TargetReadinessV1` ref、invalidation conditionをDomain evidenceとして生成する。共通Receipt envelope、Provenance、署名、保持は[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を参照し、本書で再掲しない。

algorithm optimizationの説明は§8.4の`OptimizationDecisionProjectionV1`だけを消費し、raw benchmark logや設計文書の全量をAIへ投入して不足Fieldを推測させない。Project Source／文書抜粋はread-only contextであり、Projectionまたはregistered Operationの代用ではない。AIの説明、Tool visibility、Provider schema適合はselection、promotion、Commit authorityを付与しない。

最低Diagnosticはmissing envelope、ambiguous requirement、unqualified capability、stale plan、fidelity violation、invalid reference closure、budget exceeded、partial activation rejected、distributed authority not activatedを区別する。unknownを近いenum、0、最大値、current Target defaultへ補正しない。

## 12. MediumからLargeへの非破壊遷移

許可する変更は、Partition Intent／Target追加、同じSourceからのinstance／batch／HLOD／Simulation LOD生成、Asset closure分割、Authoring re-shard、Build work item分割、Target別residency／fallback追加である。Source Stable ID、Save schema、Gameplay authorityを変えず、Derived Planだけを置換する。

禁止する変更は、Large専用owner typeへのSource一括変換、Medium／Large別Save fork、cell／shard／build／server IDの混同、HLOD／GPU instanceのSave authoritative record化、unqualified planのProduction表示、Medium fallback削除、性能のための無承認authoritative semantics変更である。

同じSource revisionとinput traceに対するMedium／Large planは、Save field／Stable ID、Input→Command→Event順序、registered runtime-entry／transition outcomeを一致させる。authoritative stateの同値Gateは二層とする。(a) 両planでSimulation LODを適用しないfull fidelity対象entityは、[Persistence／Save](../04-runtime/persistence-save.md)の`RuntimeAuthoritativeStateDigestV1`が定めるsealed World publicationで採取したentity projection hashと、当該entityへ帰属するdeterministic random stream消費を同一`advance_sequence`で一致させる。(b) いずれかのplanでSimulation LODを適用するentityは、[LOD](../06-rendering/lod.md)の`authoritative_equivalence_contract`と`reference_fixture_id`により、active owner schema registryが定めるauthoritative state／event outcome、registered collision／navigation evidence、wake後のstate収束をsemantic同値として判定する。full fidelity対象集合は両planのSimulation LOD適用集合の補集合として決定的に導出し、runごとに変えない。Presentation bitwise一致は不要でも、visual／audio tolerance、critical cue、event timing、fallback Gateを満たす。

Large World coordinate、continuous streaming、partition-owned multi-writer、distributed build、distributed simulation／authorityは専用Owner仕様がactivationされるまで`not_activated`である。現在のbounded Sourceへ空Manager、server field、RPC、global double座標を先回り追加しない。要求された場合は明示Diagnosticでfail closedする。

## 13. Integrated fixtureとqualification

共有Contract `IntegratedScaleFixtureV1`は本節だけが所有し、各ProjectのEnvelopeとactive `RuntimeScaleIntentDimensionRegistryV1`から、全owner-typed instance dimension、create／retire lifecycle dimension、authoritative event dimension、Physics／Navigation／Animation、Game System、presentation effect、Audio、view、streaming、LOD、Asset activationを、実際に同時発生し得る一つのdeterministic integrated fixtureへ生成する。Subsystem最大値を別runへ分離して同時性を隠さず、Compilerは宣言された`maximum_expected`を丸めない。未登録dimensionまたはfixture recipe欠落はqualification開始前に拒否する。

fixtureは次を全て満たす。

1. frame、Subsystem、memory、queue、GPU resource、streamingのhard Gate。
2. registered authoritative create／retire／state／event record drop 0。
3. 各workload ownerが登録したauthoritative state／event、functional result、resource SLO、Replay oracleがreferenceと一致。
4. Presentation degradationがpriority、style、critical cue floorを満たす。
5. registered lifecycle burst、streaming boundary、presentation-effect burstのP99.9がdeadlineを満たす。

`medium_candidate / qualified`にはProject固有Envelope、exact runtime-entryと選択workload owner closure、必要な場合だけWorld／spatial closure、Save／Replay／Package、content totalとactive working setの分離、incremental Import／Cook、Target budget、2時間endurance、migration／load、bounded AI edit、last-valid recoveryを必要とする。

`large_local_candidate / qualified`にはMedium Gateに加え、利用するLarge Capabilityの専用仕様／Receipt、Project固有traversal／population trace、partition boundary／reference closure／load deadline／memory pressure／recovery、same-source Medium fallback、repartition後のStable ID／Save／Replay、bounded context、incremental／partial Cook同値、10分×3 run、2時間endurance、failure injectionを必要とする。

本節のqualification計測run（§8のScale qualification 10分×3 run、Production endurance 2時間runを含む）は、`predicted` TargetのDevelopment Play実行モードで実行できる（[Project state](../03-authoring/project-state.md#9-runtime-compile境界)）。計測run自体の開始に`qualified`を要求せず、同じ`input_closure_hash`へ束縛されたfresh Receipt確定後にだけ`qualified`へ昇格する。`blocked_reason_ref=performance_envelope_unqualified`の製品Envelopeは、§9の5入力を揃えた専用qualification harnessだけを開始でき、通常Development Playを許可しない。

Distributed qualification Gateは本書でactivationしない。専用Authority仕様、Threat Model、server実機、loss／latency／abuse／recovery fixture、人間承認が揃うまでCatalogへactive Gateを掲載しない。
