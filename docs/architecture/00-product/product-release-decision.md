# Miraikanai Engine Product Release Decision Contract

- 文書ID: mirakan.arch.product-release-decision
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Product release eligibilityのauthoritative subject、same-scope Legal／IP readiness binding、署名済みRelease Decision record、release authority／quorum／freshness／revocation、Product authority scope／stream identity、state transition authorization、署名済みcurrent head、current／superseded／revoked state
- 非正本範囲: Product intent／required universe導出、Release Content Manifest、Lifecycle／Security／Legal-IP acceptanceのdomain意味、Platform signing／upload／submission、実公開、Product completion、汎用署名・Role・Trust semantics。各Ownerを参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Product Plan](product-plan.md)、[Product Lifecycle](product-lifecycle.md)、[Product Legal／IP Governance](../01-governance/product-legal-ip-governance.md)、[Product Security](../01-governance/product-security.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)
- 関連文書: [Product Publication／Completion](product-publication-completion.md)、[Windows](../07-platform/windows.md)、[Android](../07-platform/android.md)、[Apple](../07-platform/apple.md)
- 根拠区分: project-decision（署名primitive、Role、Trust、revocationは参照先Governance Ownerの正本に従う）
- 外部根拠確認日: none

## 1. 結論と所有境界

Release Decisionは「公開してよい」というauthorizationであり、Release label、Build成功、Lifecycle Acceptance、Security Binding、unsigned subject、Platform uploadまたはStore availabilityではない。本書はProduct Planのpure predicateを、qualified release authorityがpurpose-separated signatureで承認または拒否する最終authority recordへ閉じる。

Release authorityは法的判断を代行しない。Release subjectは[Product Legal／IP Governance](../01-governance/product-legal-ip-governance.md)のsame-scope current `approved` DecisionとHeadを必須入力としてread-backし、jurisdiction／market／channel／role／Candidateが一件でも不一致、未review、期限切れ、revoked、supersededまたはscope-changedならReleaseをfail closedにする。

実際のpublic availability、partial publication、withdrawal、supersession、support開始、Product completionは[Product Publication／Completion](product-publication-completion.md)だけが所有する。Platform Ownerは本書の有効な署名済みDecisionをsigning／upload／submission authorizationとして消費するが、本書はPlatform Receiptをhash preimageへ含めない。

対応Schema、Role assignment、Key、Trust Registry、Decision store、Operation、ReceiptはRepositoryに存在せず、未materialize／未Activationである。

## 2. 共通規則

- 全objectはclosedであり、未知Field、重複Field、非canonical順、範囲外値を拒否する。
- Refは全Fieldを解決先とbyte equalityにする。ID-only、display name、`latest`、同release label、同Decision ID別hashへfallbackしない。
- 配列はcanonical identity byte順のstrict sorted setであり、duplicateを拒否する。
- content hashは自己hashだけを除くclosed canonical bytesを、型固有ASCII domain separatorと`uint32_be` length framingでSHA-256する。
- Decision subject、authority payload、署名wrapper、authority stateを同一recordまたは相互hash参照へ統合しない。

| 型 | ASCII domain separator |
|---|---|
| `ProductReleaseDecisionSubjectV1` | `MIRAKAN_PRODUCT_RELEASE_DECISION_SUBJECT_V1` |
| `ProductReleaseDecisionRecordV1` | `MIRAKAN_PRODUCT_RELEASE_DECISION_RECORD_V1` |
| `ProductReleaseAuthorityScopeSubjectV1` | `MIRAKAN_PRODUCT_RELEASE_AUTHORITY_SCOPE_SUBJECT_V1` |
| `ProductAuthorityStateStreamKeyV1` | `MIRAKAN_PRODUCT_AUTHORITY_STATE_STREAM_KEY_V1` |
| `ProductReleaseDecisionAuthorityStateV1` | `MIRAKAN_PRODUCT_RELEASE_DECISION_AUTHORITY_STATE_V1` |
| `ProductAuthorityStateAuthorizationRecordV1` | `MIRAKAN_PRODUCT_AUTHORITY_STATE_AUTHORIZATION_RECORD_V1` |
| `ProductAuthorityStateHeadRecordV1` | `MIRAKAN_PRODUCT_AUTHORITY_STATE_HEAD_RECORD_V1` |

## 3. Release Decision subject

```text
ProductReleaseDecisionSubjectV1
  product_release_decision_subject_id: StableId
  product_release_decision_subject_version: 1
  release_requirement_projection_ref:
    exact ProductReleaseRequirementProjectionRefV1
  required_operation_universe_ref:
    exact RequiredProductOperationUniverseRefV1
  required_operation_journey_projection_ref:
    exact RequiredProductOperationJourneyProjectionRefV1
  operation_activation_closure_ref:
    exact ProductOperationActivationClosureRefV1
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  product_lifecycle_acceptance_ref:
    exact ProductLifecycleAcceptanceRefV1
  product_legal_applicability_profile_ref:
    exact ProductLegalApplicabilityProfileRefV1
  product_legal_ip_decision_ref: exact ProductLegalIpDecisionRefV1
  product_legal_ip_decision_head_ref:
    exact ProductLegalIpDecisionHeadRefV1
  product_security_release_binding_ref:
    exact ProductSecurityReleaseBindingRefV1
  production_activation_set_ref:
    exact ProductProductionActivationSetRefV1
  supplemental_evidence_bindings[0..65535]:
    sorted unique {
      requirement_ref: exact McdContractRefV1(kind=requirement),
      evidence_class_ref: exact EvidenceClassRefV1,
      evidence_ref: exact EvidenceRefV1,
      satisfied_host_scope:
        {kind=not_applicable}
        | {kind=host_independent}
        | {
            kind=exact_set,
            host_profile_refs[1..64]:
              sorted unique exact TargetProfileRefV1(
                profile_kind=build_host | editor_host)
          },
      satisfied_target_scope:
        {kind=not_applicable}
        | {kind=target_independent}
        | {
            kind=exact_set,
            target_profile_refs[1..64]:
              sorted unique exact TargetProfileRefV1(
                profile_kind=runtime_target)
          },
      satisfied_locale_scope:
        {kind=not_applicable}
        | {kind=locale_independent}
        | {
            kind=exact_set,
            locale_profile_refs[1..64]:
              sorted unique exact LocaleProfileRefV1
          },
      satisfied_reference_dimension_scope:
        {kind=not_applicable}
        | {kind=dimension_independent}
        | {
            kind=exact_set,
            reference_dimensions[1..2]:
              sorted unique two_d | three_d
          }
    }
  decision_state: approved | rejected
  required_minus_satisfied_gap_content_hash: SHA-256
  rejection_reason_evidence_refs[0..64]:
    sorted unique exact EvidenceRefV1
  product_release_decision_subject_content_hash: SHA-256
```

`approved`では次をすべて満たす。

1. Requirement Projection、Required Operation Universe、Required Operation Journey Projection、Operation Activation Closure、Engine Release、Lifecycle Acceptance、Legal Applicability Profile／Decision／Head、Security Binding、Activation SetのActive Product Definition、Claim Scope、Candidate、Toolchain、public contract、Host／runtime Target／locale集合を各Ownerのprojection規則でbyte／set equalityにする。
2. Requirement Projectionの`required_activation_subjects[]`とActivation Setのsupplied subject projectionをset equalityにする。
3. Requirement Projectionの全`required_acceptance_subjects[]`が要求する`{requirement_ref,evidence_class_ref,required Host scope,required Target scope,required locale scope,required Reference dimension scope}`と、Lifecycle／Legal／Security／Privacy／License／Support／Reference／Pack acceptanceまたは`supplemental_evidence_bindings[]`から正規化した同じpair固有のsatisfied scopeをexact set equalityにする。`required_evidence_class_refs[]`はそのdistinct class projectionとset equalityにする。
4. 各required pairを同じrequirement identityのOwner acceptanceまたはsupplemental Evidenceへ一意に解決する。Legal pairはcurrent approved Legal／IP DecisionのReview Subjectにある`supplied_domain_evidence_bindings[]`から、Requirement Bindingの`product_requirement_ref`とrowの`evidence_class_ref`をread-backして正規化する。同一pairに複数Evidenceがある場合はそのpair内だけでHost、runtime Target、locale、Reference dimensionのcanonical unionを計算し、required branchと各々exact equalityにする。`not_applicable`、`host_independent`／`target_independent`／`locale_independent`／`dimension_independent`、`exact_set`は相互変換せず、independentまたはnot-applicable branchとexact setの混在、empty exact set、別pairを跨ぐunionを拒否する。Legal Decision record単独、Decision reason、source snapshotまたはauthorized reviewer identityをEvidence pairの代用にしない。
5. 各Evidence／Qualification ReceiptへAI Verification OwnerのVerification Semantic Admissibility Predicate v1を適用し、generic wrapper、owner-specific completed record、subject contract集合、Host／Target／locale／Reference dimension scope、Scenario／branch、Candidate、freshness、non-revocationを発行元Ownerでread-backする。content hashまたはset equalityだけで意味制約を省略しない。
6. canonical `required − satisfied`差分を再計算し、そのcanonical empty projection hashを`required_minus_satisfied_gap_content_hash`とbyte equalityにする。
7. `rejection_reason_evidence_refs=[]`とする。
8. Required Operation UniverseとOperation Activation Closureの`{family,operation,surface}` projectionをset equalityにし、全Operation Activation Evidenceがfreshかつnon-revokedである。
9. Required Operation Journey Projectionのfull `{Claim Scope,Requirement,semantic group,family,Operation,surface,Host scope,Target scope,locale scope,Reference dimension scope,scenario,branch,Evidence class}`集合とLifecycle Acceptanceのtyped Journey Evidence projectionをexact set equality、branch／Evidence classを除く同scopeのforbidden surface集合とのintersectionをexact emptyにし、Activation Evidence、別localeまたは別dimensionのJourney Evidenceをjourney qualificationへ数えない。
10. Required Operation Journey Projectionのdistinct `{operation_family_kind,operation_ref}` projectionをRequired Operation Universeの同projectionとset equalityにし、各Universe pairに少なくとも一つのrequired `success` rowと、同じHost／Target／locale／Reference dimension scopeをread-backするfresh Journey Evidenceがあることを検証する。
11. Claim Scopeへ入る全claim-facing Requirementについて、Product Planの唯一のtyped InputとMinimum bindingの`{requirement_ref,requirement_category}`をbyte equalityにし、pre-publication、Completion、JourneyのEvidence class canonical unionをnon-emptyにする。`third_party_product_release`ではdistinct category projectionを全`ProductRequirementCategoryV1`とexact set equalityにし、category alias／override、Evidence obligation三集合empty、別Requirementまたはglobal class unionによる補完を拒否する。

加えてLegal Applicability Profileの全distribution bindingがLifecycle Acceptanceの`distribution_coverage_projection_ref`から解決する全required publication routeとexact scope equalityであり、Legal DecisionがそのProfileのrequired binding／Domain Evidence／Independent Design closureを承認し、Headが同Decisionをcurrentとして指すことをProduct Legal／IP Ownerのeffective-state規則でread-time再検証する。Release Decision自身、Security pass、SBOM、license label、AI回答またはArchitecture reviewをLegal Approvalへ代用しない。

`rejected`では一件以上のrejection reasonを必須とし、Release signing、publication、claim、support開始またはCompletionのauthorizationに使用しない。Decision作成者がrequired集合、gap、acceptance kind、Evidence classまたはpair固有scopeをinlineに追加・削除するFieldは存在しない。`supplemental_evidence_bindings[]`はOwner aggregateに含まれないexact Evidenceを同じrequired pairへ束縛するだけで、required universeまたはscopeの選択元ではない。

## 4. 署名済みauthority record

```text
ProductReleaseAuthorityQuorumRuleV1
  allowed_authority_role_refs[1..16]:
    sorted unique exact AuthorityQualificationRoleRefV1
  minimum_distinct_subjects: positive u32
  independence_class:
    none | independent_from_requester
    | independent_from_release_owner
    | independent_from_requester_and_release_owner

ProductReleaseDecisionRecordV1
  product_release_decision_record_id: StableId
  product_release_decision_record_version: 1
  subject_ref: exact ProductReleaseDecisionSubjectRefV1
  qualified_release_authorities[1..16]:
    sorted unique {
      authority_subject_ref,
      qualification_role_ref: exact AuthorityQualificationRoleRefV1
    }
  authority_quorum_rules[1..8]:
    sorted unique ProductReleaseAuthorityQuorumRuleV1
  issued_at: RFC 3339 timestamp
  expires_at: RFC 3339 timestamp
  revocation_snapshot_ref
  signed_records[1..16]:
    sorted unique
      exact MirakanSignedRecordV1(purpose=product_release_decision)
  product_release_decision_record_content_hash: SHA-256
```

各`signed_records[].subject_sha256`は、`signed_records`配列とRecord自己hashを除く完成Record payloadの同じRFC 8785 JCS SHA-256とbyte equalityにする。`signed_records[]`のSigner subject projectionは`qualified_release_authorities[].authority_subject_ref`とset equalityで、同一subjectの複数Key／複数署名をquorumへ重複計上しない。各wrapperの`issued_at`と`revocation_snapshot_ref`はRecordの同Fieldへbyte equality、transport signing Role／Keyはpurpose singleton `product_release_decision`だけを許可する。各`authority_subject_ref`と`revocation_snapshot_ref`は[AI Security／Approval](../01-governance/ai-security-approval.md)のcurrent Trust closureが定義するexact identity／snapshot shapeであり、本書は別型を作らない。Qualification Role、assignment scope、全quorum rule、minimum distinct subject、independence、issued-at、expiry、current revocation snapshotを同Ownerで再検証する。transport Roleをqualification Roleまたはquorumの代用にしない。

`approved` subjectのunsigned Ref、有効期限切れ、revoked Key／Role／subject、quorum不足、purpose違い、別subject hash、別Engine Releaseまたは別Requirement ProjectionのRecordを拒否する。

## 5. current authority state

```text
ProductAuthorityScopeSubjectRefV1
  authority_scope_owner_document_id: non-empty canonical ASCII
  authority_scope_type_id: non-empty canonical ASCII
  authority_scope_subject_content_hash: SHA-256

ProductReleaseAuthorityScopeSubjectV1
  active_product_definition_ref: exact ActiveProductDefinitionRefV1
  claim_scope_ref: exact ProductClaimScopeRefV1
  release_requirement_projection_ref:
    exact ProductReleaseRequirementProjectionRefV1
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  product_legal_applicability_profile_ref:
    exact ProductLegalApplicabilityProfileRefV1
  release_authority_scope_subject_content_hash: SHA-256

ProductAuthorityStateStreamKeyV1
  authority_service_ref: exact McdContractRefV1(kind=service)
  state_owner_document_id: non-empty canonical ASCII
  state_type_id: non-empty canonical ASCII
  authority_scope_subject_ref: exact ProductAuthorityScopeSubjectRefV1
  state_stream_key_content_hash: SHA-256

ProductReleaseDecisionAuthorityStateV1
  decision_authority_state_id: StableId
  decision_authority_state_version: positive u32
  authority_scope_subject_ref:
    exact ProductReleaseAuthorityScopeSubjectRefV1
  decision_record_ref: exact ProductReleaseDecisionRecordRefV1
  expected_previous_state_ref:
    null | exact ProductReleaseDecisionAuthorityStateRefV1
  authority_state: current | superseded | revoked
  successor_decision_record_ref:
    null | exact ProductReleaseDecisionRecordRefV1
  effective_at: RFC 3339 timestamp
  state_basis_evidence_refs[1..64]:
    sorted unique exact EvidenceRefV1
  decision_authority_state_content_hash: SHA-256

ProductAuthorityStateSubjectRefV1
  authority_state_owner_document_id: non-empty canonical ASCII
  authority_state_type_id: non-empty canonical ASCII
  authority_scope_subject_ref: exact ProductAuthorityScopeSubjectRefV1
  authority_state_id: StableId
  authority_state_version: positive u32
  authority_state_content_hash: SHA-256

ProductAuthorityStateAuthorizationRecordV1
  state_authorization_record_id: StableId
  state_authorization_record_version: 1
  state_subject_ref: exact ProductAuthorityStateSubjectRefV1
  expected_previous_state_authorization_ref:
    null | exact ProductAuthorityStateAuthorizationRecordRefV1
  transition_kind: establish_current | supersede | revoke
  qualified_state_authorities[1..16]:
    sorted unique {
      authority_subject_ref,
      qualification_role_ref: exact AuthorityQualificationRoleRefV1
    }
  authority_quorum_rules[1..8]:
    sorted unique ProductReleaseAuthorityQuorumRuleV1
  issued_at: RFC 3339 timestamp
  expires_at: RFC 3339 timestamp
  revocation_snapshot_ref
  signed_records[1..16]:
    sorted unique exact MirakanSignedRecordV1(
      purpose=product_authority_state_transition)
  state_authorization_record_content_hash: SHA-256

ProductAuthorityStateHeadRecordV1
  state_head_record_id: StableId
  state_head_record_version: positive u32
  state_stream_key: exact ProductAuthorityStateStreamKeyV1
  authorized_state_subject_ref: exact ProductAuthorityStateSubjectRefV1
  state_authorization_record_ref:
    exact ProductAuthorityStateAuthorizationRecordRefV1
  expected_previous_state_head_ref:
    null | exact ProductAuthorityStateHeadRecordRefV1
  authority_service_ref: exact McdContractRefV1(kind=service)
  authority_service_sequence: positive u64
  issued_at: RFC 3339 timestamp
  expires_at: RFC 3339 timestamp
  revocation_snapshot_ref
  signed_records[1..16]:
    sorted unique exact MirakanSignedRecordV1(
      purpose=product_authority_state_head)
  state_head_record_content_hash: SHA-256
```

Release authority scopeは同じActive Product Definition、Claim Scope、Release Requirement Projection、Engine Release、Legal Applicability Profileのexact tupleであり、Product line global、Authority Service global、release labelまたはDecision ID単独へ縮退しない。Decision Subjectの同五Fieldをscopeとbyte equalityにし、別market／channel／role Profileを同じauthority streamへ付け替えない。`ProductReleaseAuthorityScopeSubjectV1`をgeneric `ProductAuthorityScopeSubjectRefV1`へ投影するとき、ownerは`mirakan.arch.product-release-decision`、typeは`ProductReleaseAuthorityScopeSubjectV1`、content hashはlocal recordとbyte equalityにする。

最初のStateはversion 1、`expected_previous_state_ref=null`である。更新は同じState IDの直前versionへCASし、branch、merge、skip、self、cycleを拒否する。`current`はsuccessor null、`superseded`はsuccessor必須、`revoked`はsuccessor optionalである。Release Stateを`ProductAuthorityStateSubjectRefV1`へ投影するとき、ownerは`mirakan.arch.product-release-decision`、typeは`ProductReleaseDecisionAuthorityStateV1`、scopeは同StateのRelease scopeを上記generic Refへ投影した値、残りのID／version／hashはlocal exact Refとbyte equalityにする。

State Recordはunsignedなimmutable transition subjectであり、それだけではcurrent authorityにならない。各Stateはexactly oneの有効なState Authorizationを持ち、最初のauthorizationはprevious null、以後は直前authorizationへCASする。`transition_kind`はStateの`authority_state`と`establish_current→current`、`supersede→superseded`、`revoke→revoked`で一致させる。Authorizationのsignature subjectは署名配列と自己hashを除く完成payloadで、qualified authority、quorum、independence、freshness、revocationを§4と同じ規則で再検証する。

Authority State streamはexact `{authority_service_ref,state_owner_document_id,state_type_id,authority_scope_subject_ref}`を一つの`ProductAuthorityStateStreamKeyV1`へcanonical encodeし、自己hashを除いてhashする。State IDは`mirakan.authority.state.h<lowercase state_stream_key_content_hash>`、Head IDは`mirakan.authority.head.h<lowercase state_stream_key_content_hash>`のNamed derivation v1だけを許可する。`h` prefixはNaming OwnerのStable ID grammarでhash segmentの数字開始を防ぐ固定prefixであり、caller生成ID、display scope、list positionまたは時刻を使わない。同じstream keyから別State ID／Head IDを作る入力、同じIDを別Service／owner／type／scopeへ再利用する入力を拒否する。

Authority Serviceは有効なAuthorizationを受理した同じatomic transactionで、stream key単位に新Headを発行する。最初のHeadはversion 1、previous null、sequence 1であり、同じstream keyのHeadが不存在であることをCAS preconditionにする。以後は同じderived Head IDの直前version、同じstream key、`previous.sequence + 1`へCASする。Headのstream key、authorized State Subjectのowner／type／scope、AuthorizationのState Subject、Authority Serviceはbyte equalityにし、Head署名subjectは署名配列と自己hashを除く完成payloadと一致させる。dual genesis、branch、merge、skip、sequence再利用、wrong stream／scope、別Service、unsigned、expired、revoked、State／Authorization／Head lineage不一致を拒否する。異なるRelease scopeまたはPublication／Completion scopeは異なるstream keyとして共存できる。Consumerはconfigured Authority Serviceとexact stream keyからread-backしたHeadだけを使い、caller提供のHead ID、最大version／sequence推測、時刻、一覧sortまたはcached headをcurrentの代用にしない。

Platform signing／upload／submissionとProduct claimは、`decision_state=approved`、Decision署名／freshness／revocation有効、Release State `current`、有効なState Authorization、同じStateを指すAuthority Serviceのcurrent署名済みHeadを一組で受理する。後続Headが`superseded`または`revoked`を指す旧Decisionをcurrentへ戻さない。

## 6. exact Ref

| Ref | Field |
|---|---|
| `ProductAuthorityScopeSubjectRefV1` | `{authority_scope_owner_document_id, authority_scope_type_id, authority_scope_subject_content_hash}` |
| `ProductReleaseAuthorityScopeSubjectRefV1` | `{active_product_definition_ref, claim_scope_ref, release_requirement_projection_ref, engine_release_binding_ref, product_legal_applicability_profile_ref, release_authority_scope_subject_content_hash}` |
| `ProductAuthorityStateSubjectRefV1` | `{authority_state_owner_document_id, authority_state_type_id, authority_scope_subject_ref, authority_state_id, authority_state_version, authority_state_content_hash}` |
| `ProductReleaseDecisionSubjectRefV1` | `{product_release_decision_subject_id, product_release_decision_subject_version=1, product_release_decision_subject_content_hash}` |
| `ProductReleaseDecisionRecordRefV1` | `{product_release_decision_record_id, product_release_decision_record_version=1, product_release_decision_record_content_hash}` |
| `ProductReleaseDecisionAuthorityStateRefV1` | `{decision_authority_state_id, decision_authority_state_version, decision_authority_state_content_hash}` |
| `ProductAuthorityStateAuthorizationRecordRefV1` | `{state_authorization_record_id, state_authorization_record_version=1, state_authorization_record_content_hash}` |
| `ProductAuthorityStateHeadRecordRefV1` | `{state_head_record_id, state_head_record_version, state_head_record_content_hash}` |

## 7. failureと禁止fallback

- bare Subject、content hashだけ、Approval label、人間名、issue stateをauthorityにしない。
- Lifecycle、Security、Privacy、License、Support、Reference、Pack、ActivationまたはEvidenceの一部欠落を別class成功で補わない。
- Legal／IP Decisionのmissing、partial scope、expired、revoked、supersededまたはscope-changedを別market、別Candidate、license scan、Security passまたはrelease authority署名で補わない。
- rejected、expired、revoked、superseded Decision、unsigned State、未authorization State、stale／unsigned HeadをPlatform actionまたはpublic claimへ使わない。
- Decision RecordまたはStateをRelease Content Manifest、Engine Release、Acceptance、Activation Bindingへ埋め戻さない。
- Platform publication ReceiptまたはCompletion RecordをDecision hashへ含めない。

## 8. Qualification

設計上必要なEvidence classは、requirement／required journey projection再現、JourneyのHost／Target／locale／Reference dimension scalar scope保持、requirement identity付きrequired／supplied set equality、canonical gap empty、subject／wrapper byte equality、qualified release Role／quorum／independence、purpose-separated signature、freshness、current revocation snapshot、unsigned／expired／revoked／superseded／cross-release／cross-locale／cross-dimension substitution negative、scope projector、stream-key-derived State／Head ID、state／authorization／headのstream単位三重CAS、dual genesis／wrong-stream／wrong-scope／stale head／branch／sequence skip negative、document／type hash DAGである。

文書が`review`であること、上記型が記載されていること、署名形式が定義されていることは、Decision発行経路、qualified authority、Key、Trust closureまたはRelease authorizationが実在する証拠ではない。

## 9. 完了条件

- Required universeがProduct Planのexact Projectionだけから決まり、Decisionにinline required集合がない。
- Release subject、authority scope／stream、signature、freshness、revocation、quorum、current stateが一意に閉じる。
- Platform Ownerがbare subjectではなくcurrent署名済みDecision Recordを要求する。
- Product Release subjectがsame-scope current Legal／IP Decision／Headを必須とする。
- Release DecisionからPlatform Receipt、Publication、Completionへの一方向DAGが成立する。
