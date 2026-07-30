# Miraikanai Engine Product Publication／Completion Contract

- 文書ID: mirakan.arch.product-publication-completion
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Product-level publication identity、distribution channel結果集約、published／partially published／withdrawn／superseded state、Publication／Completion authority scope、support開始binding、実公開を要求するauthoritative Product Completion Decisionとapproved／rejected branch
- 非正本範囲: Product intent／required universe、Release authorization、Release Content Manifest、Platform signing／upload／submission／Store Receiptのdomain意味、Support Window base、汎用署名・Role・revocation。各Ownerを参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Product Plan](product-plan.md)、[Product Lifecycle](product-lifecycle.md)、[Product Release Decision](product-release-decision.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Windows](../07-platform/windows.md)、[Android](../07-platform/android.md)、[Apple](../07-platform/apple.md)
- 関連文書: [Product Security](../01-governance/product-security.md)、[Product Privacy／Data Governance](../01-governance/product-privacy-data-governance.md)
- 根拠区分: project-decision（Platform／Store事実は各Platform Owner、署名・TrustはGovernance Ownerを参照する）
- 外部根拠確認日: none

## 1. 結論と所有境界

署名済みRelease Decisionはpublication eligibilityであり、public availabilityではない。本書はRelease Decision後に各Platform／distribution channelで生じるsigning、upload／submission、approval、read-backを集約し、実際に公開されたTarget／channel／locale／claim scope、effective time、partial failure、withdrawal、supersession、current headをProduct-level identityへ閉じる。

Platform Ownerは本書へ依存せず、[Product Release Decision](product-release-decision.md)のcurrent署名済みDecisionを入力にPlatform Receiptを発行する。本書がそれらReceiptを下流から集約する。Release Content Manifest、Engine Release、Release DecisionまたはPlatform ReceiptへPublication／Completion refを埋め戻さない。

対応Schema、Publication store、channel Receipt、CAS head、support start binding、Completion authority、OperationはRepositoryに存在せず、未materialize／未Activationである。

## 2. 共通規則

- 全objectはclosed、全Refは解決先と全Field byte equality、配列はcanonical identity byte順のstrict sorted setとする。
- content hashは自己hashだけを除くclosed canonical bytesを、型固有ASCII domain separatorと`uint32_be` length framingでSHA-256する。
- Release approval、signing、upload、submission、Store approval、public read-back、Product publication、Completionを別subjectとして保持する。
- required channel universeはProduct PlanのRelease Requirement Projectionから導出し、Publication作成者がinlineで縮小できない。

| 型 | ASCII domain separator |
|---|---|
| `ProductPublicationProjectionV1` | `MIRAKAN_PRODUCT_PUBLICATION_PROJECTION_V1` |
| `ProductReleasePublicationV1` | `MIRAKAN_PRODUCT_RELEASE_PUBLICATION_V1` |
| `ProductPublicationAuthorityScopeSubjectV1` | `MIRAKAN_PRODUCT_PUBLICATION_AUTHORITY_SCOPE_SUBJECT_V1` |
| `ProductPublicationStateRecordV1` | `MIRAKAN_PRODUCT_PUBLICATION_STATE_RECORD_V1` |
| `ProductSupportPublicationStartBindingV1` | `MIRAKAN_PRODUCT_SUPPORT_PUBLICATION_START_BINDING_V1` |
| `ProductCompletionDecisionSubjectV1` | `MIRAKAN_PRODUCT_COMPLETION_DECISION_SUBJECT_V1` |
| `ProductCompletionDecisionRecordV1` | `MIRAKAN_PRODUCT_COMPLETION_DECISION_RECORD_V1` |
| `ProductCompletionAuthorityScopeSubjectV1` | `MIRAKAN_PRODUCT_COMPLETION_AUTHORITY_SCOPE_SUBJECT_V1` |
| `ProductCompletionAuthorityStateV1` | `MIRAKAN_PRODUCT_COMPLETION_AUTHORITY_STATE_V1` |

## 3. Product release publication

```text
ProductPublicationChannelKeyV1
  publication_requirement_ref:
    exact McdContractRefV1(kind=requirement)
  platform_kind: windows | android | apple
  channel_kind: direct_distribution | managed_store
  distribution_scope_kind:
    ProductCompletionDistributionScopeKindV1
  artifact_role_kind: ProductDistributionArtifactRoleKindV1
  execution_scope:
    {
      kind=host_profile,
      host_profile_ref: exact TargetProfileRefV1(
        profile_kind=build_host | editor_host)
    }
    | {
        kind=runtime_target_profile,
        runtime_target_profile_ref: exact TargetProfileRefV1(
          profile_kind=runtime_target)
      }
    | {kind=scope_independent}
  locale_scope:
    {
      kind=locale_profile,
      locale_profile_ref: exact LocaleProfileRefV1
    }
    | {kind=locale_independent}
  distribution_subject_ref: exact ProductDistributionSubjectRefV1
  distribution_artifact_ref: exact ArtifactRefV1

ProductPublicationProjectionV1
  publication_projection_id: StableId
  publication_projection_version: 1
  release_requirement_projection_ref:
    exact ProductReleaseRequirementProjectionRefV1
  distribution_coverage_projection_ref:
    exact ProductDistributionCoverageProjectionRefV1
  required_channel_keys[1..65535]:
    sorted unique ProductPublicationChannelKeyV1
  projection_algorithm_id: product_publication_projection
  projection_algorithm_version: 1
  projection_algorithm_content_hash: SHA-256
  publication_projection_content_hash: SHA-256

ProductPublicationRouteStepReceiptV1
  route_step:
    signing | upload_or_submission | approval | public_readback
  receipt_ref: exact EvidenceRefV1

ProductPublicationChannelResultV1
  channel_key: exact ProductPublicationChannelKeyV1
  outcome:
    {
      kind: success,
      completed_route_step_receipts[2..4]:
        ordered unique ProductPublicationRouteStepReceiptV1,
      public_availability_at: RFC 3339 timestamp
    }
    | {
        kind: failed,
        completed_route_step_receipts[0..3]:
          ordered unique ProductPublicationRouteStepReceiptV1,
        failure_evidence_ref: exact EvidenceRefV1
      }
    | {
        kind: pending,
        completed_route_step_receipts[0..3]:
          ordered unique ProductPublicationRouteStepReceiptV1,
        pending_evidence_ref: exact EvidenceRefV1,
        observed_at: RFC 3339 timestamp
      }

ProductReleasePublicationV1
  product_release_publication_id: StableId
  product_release_publication_version: 1
  release_decision_record_ref:
    exact ProductReleaseDecisionRecordRefV1
  release_decision_authority_state_ref:
    exact ProductReleaseDecisionAuthorityStateRefV1
  release_state_authorization_ref:
    exact ProductAuthorityStateAuthorizationRecordRefV1
  release_state_head_ref:
    exact ProductAuthorityStateHeadRecordRefV1
  release_requirement_projection_ref:
    exact ProductReleaseRequirementProjectionRefV1
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  release_content_manifest_ref:
    exact ProductReleaseContentManifestRefV1
  publication_projection_ref: exact ProductPublicationProjectionRefV1
  channel_results[0..65535]:
    sorted unique ProductPublicationChannelResultV1
  successful_channel_keys[0..65535]:
    sorted unique ProductPublicationChannelKeyV1
  failed_channel_keys[0..65535]:
    sorted unique ProductPublicationChannelKeyV1
  pending_channel_keys[0..65535]:
    sorted unique ProductPublicationChannelKeyV1
  missing_channel_keys[0..65535]:
    sorted unique ProductPublicationChannelKeyV1
  published_distribution_subject_refs[0..8192]:
    sorted unique exact ProductDistributionSubjectRefV1
  published_distribution_artifact_refs[0..65535]:
    sorted unique exact ArtifactRefV1
  published_distribution_scope_kinds[0..10]:
    sorted unique ProductCompletionDistributionScopeKindV1
  published_artifact_role_kinds[0..20]:
    sorted unique ProductDistributionArtifactRoleKindV1
  published_execution_scopes[0..129]:
    sorted unique {
      kind=host_profile,
      host_profile_ref: exact TargetProfileRefV1(
        profile_kind=build_host | editor_host)
    }
    | {
        kind=runtime_target_profile,
        runtime_target_profile_ref: exact TargetProfileRefV1(
          profile_kind=runtime_target)
      }
    | {kind=scope_independent}
  published_locale_scopes[0..65]:
    sorted unique {
      kind=locale_profile,
      locale_profile_ref: exact LocaleProfileRefV1
    }
    | {kind=locale_independent}
  published_claim_scope_ref: exact ProductClaimScopeRefV1
  publication_result: published | partially_published
  effective_publication_at: null | RFC 3339 timestamp
  product_release_publication_content_hash: SHA-256
```

Publication ProjectionのNamed Algorithm v1は、Lifecycle Distribution Coverage Projectionの各`publication_route_projection[]` rowについて`required_routes[]`をflattenし、同rowのDistribution Subject／artifact、artifact role、execution scope、locale scopeを結合した完全`ProductPublicationChannelKeyV1`集合を生成する。入力CoverageはLifecycleのDistribution Projection Capacity Validity Algorithm v1を満たし、full bindingは65,535件以下、required keyのproduct-wide flatten総数も65,535件以下でなければならない。checked arithmetic overflow、上限超過またはLifecycle validity未証明ならProjectionを生成しない。各routeはRelease Requirement Projectionの`required_publication_distribution_subjects[]`と全selector Fieldでbyte equalityのmemberで、同rowのrequired／forbidden完全分割、全full binding row coverage、Required Product subject projectionのset equality、Manifest／Coverage／Claim Scopeのbyte equalityを先に検証する。得られる完全tupleをcanonical sortし、同一tupleのduplicate、forbidden route、配布Subject、artifact、role、execution scopeまたはlocale scopeを欠くkey、Claim Scope外のextra keyを拒否する。Publication作成者はchannel、distribution scope、artifact role、Host、runtime Target、scope-independent、locale、locale-independentまたはSubject／artifact applicabilityを推論せず、required keyをinlineで追加、削除、集約または分割しない。

direct distributionのsuccess routeは`signing → public_readback`、managed storeは`signing → upload_or_submission → approval → public_readback`のexact ordered sequenceである。failed／pending branchはそのrouteのstrict prefixだけを持ち、順序飛ばし、余分なstep、public read-back後のfailed／pending、null failure／pending Evidenceを拒否する。各Receiptは同じsigned Release Decision、Release State Authorization／Head、Engine Release、Manifest、distribution scope、artifact role、Host／runtime Target／scope-independent execution scope、locale／locale-independent scope、Distribution Subject、artifact、channel keyへ解決する。

四つのkey集合はpairwise disjointで、unionをPublication Projectionのrequired key集合とset equalityにする。`channel_results[]`のkey projectionはsuccessful／failed／pending unionとset equality、outcome kind別projectionも各集合とset equalityにし、missing keyにはResultを持たせない。`published`ではsuccessfulがrequired集合とset equality、他三集合exact emptyである。published distribution subject／artifact／distribution scope／artifact role／execution scope／locale scope集合はsuccessful keyからの各distinct projectionと、Coverage Projectionがrequiredとする各完全集合にset equalityでなければならない。`effective_publication_at`は全required successの`public_availability_at`のmaximumである。

failed／pending／missingが一件でもあれば`partially_published`で、effective timeはnull、published Subject／artifact／scope集合はsuccessful keyのexact subsetだけを表し、Product-wide release、support開始、Product completionへ昇格しない。UploadまたはStore approvalだけをpublic read-backの代用にせず、別channel、別distribution scope、別artifact role、別Host／runtime Target／scope-independent branch、別locale branch、別Decision、別Distribution Subjectまたは別artifactのReceiptを合成しない。

## 4. Publication stateとcurrent head

```text
ProductPublicationAuthorityScopeSubjectV1
  active_product_definition_ref: exact ActiveProductDefinitionRefV1
  claim_scope_ref: exact ProductClaimScopeRefV1
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  publication_projection_ref: exact ProductPublicationProjectionRefV1
  publication_authority_scope_subject_content_hash: SHA-256

ProductPublicationStateRecordV1
  publication_state_id: StableId
  publication_state_version: positive u32
  authority_scope_subject_ref:
    exact ProductPublicationAuthorityScopeSubjectRefV1
  publication_ref: exact ProductReleasePublicationRefV1
  expected_previous_state_ref:
    null | exact ProductPublicationStateRecordRefV1
  publication_state:
    published | partially_published | withdrawn | superseded
  successor_publication_ref:
    null | exact ProductReleasePublicationRefV1
  withdrawal_results[0..65535]:
    sorted unique {
      channel_key: exact ProductPublicationChannelKeyV1,
      public_unavailability_evidence_ref: exact EvidenceRefV1
    }
  effective_at: RFC 3339 timestamp
  state_basis_evidence_refs[1..65535]:
    sorted unique exact EvidenceRefV1
  publication_state_content_hash: SHA-256
```

Publication authority scopeは同じActive Product Definition、Claim Scope、Engine Release、Publication Projectionのexact tupleであり、Authority Service global、channel一件、publication display nameまたはState ID単独へ縮退しない。scope recordをgeneric `ProductAuthorityScopeSubjectRefV1`へ投影するとき、ownerは`mirakan.arch.product-publication-completion`、typeは`ProductPublicationAuthorityScopeSubjectV1`、content hashはlocal recordとbyte equalityにする。

最初のStateはversion 1、expected previous nullで、Publicationの`publication_result`と同じstateだけを許す。更新は同じState IDの直前versionへCASし、branch、merge、skip、self、cycleを拒否する。`superseded`はsuccessor必須、`withdrawn`はsuccessor optionalである。`withdrawn`以外は`withdrawal_results=[]`とする。`withdrawn`では`withdrawal_results[].channel_key` projectionを同Publicationの`successful_channel_keys[]`とset equalityにし、各rowのEvidenceを同じDecision、Release、Manifest、Distribution Subject、artifact、role、Host／runtime Target／scope-independent execution scope、locale、channelへ解決するfresh non-revoked public-unavailability recordとしてread-backする。`state_basis_evidence_refs[]`はwithdrawal rowのEvidence distinct projectionとset equalityでなければならない。一件でもmissing／extra／wrong-scope／still-availableなら`revoke→withdrawn` Stateをpublishせず、旧current Stateを維持して失敗auditだけを残す。Publication Stateを`ProductAuthorityStateSubjectRefV1`へ投影するとき、ownerは`mirakan.arch.product-publication-completion`、typeは`ProductPublicationStateRecordV1`、scopeは同StateのPublication scopeを上記generic Refへ投影した値、ID／version／hashはlocal exact Refとbyte equalityにする。各Stateは[Product Release Decision](product-release-decision.md)の有効な`ProductAuthorityStateAuthorizationRecordV1`と、exact `{Authority Service,owner,type,scope}`から導出した`ProductAuthorityStateStreamKeyV1`を持つcurrent署名済み`ProductAuthorityStateHeadRecordV1`を必須とする。State／Head IDはRelease Decision §5のstream hash Named derivationに一致させ、CAS、sequence、dual-genesis、wrong-stream／scope rejectionもstream key単位に適用する。transition kindは`establish_current→published|partially_published`、`supersede→superseded`、`revoke→withdrawn`へ一致させる。unsigned State、未authorization State、stale Head、別Service、別scopeまたはlineage不一致をcurrent availabilityとして使用しない。

## 5. Support開始

```text
ProductSupportPublicationStartBindingV1
  support_publication_start_binding_id: StableId
  support_publication_start_binding_version: 1
  support_window_ref: exact ProductSupportWindowRefV1
  publication_ref: exact ProductReleasePublicationRefV1
  publication_state_ref: exact ProductPublicationStateRecordRefV1
  publication_state_authorization_ref:
    exact ProductAuthorityStateAuthorizationRecordRefV1
  publication_state_head_ref:
    exact ProductAuthorityStateHeadRecordRefV1
  effective_support_start_at: RFC 3339 timestamp
  support_publication_start_binding_content_hash: SHA-256
```

BindingはPublication `publication_result=published`、State `publication_state=published`、有効なState Authorization、Authority Serviceからread-backしたcurrent署名済みHead、PublicationとSupport Windowの同じEngine Release、`effective_support_start_at=publication.effective_publication_at`がすべてbyte equalityの場合だけ成立する。approved Decision、upload time、Store submission time、local package build time、unsigned Stateまたはcached Headからsupport開始を推測しない。

## 6. Authoritative Product Completion

```text
ProductCompletionDecisionSubjectV1
  product_completion_decision_subject_id: StableId
  product_completion_decision_subject_version: 1
  completion_requirement_projection_ref:
    exact ProductCompletionRequirementProjectionRefV1
  approved_release_decision_record_ref:
    exact ProductReleaseDecisionRecordRefV1
  release_decision_authority_state_ref:
    exact ProductReleaseDecisionAuthorityStateRefV1
  release_state_authorization_ref:
    exact ProductAuthorityStateAuthorizationRecordRefV1
  release_state_head_ref:
    exact ProductAuthorityStateHeadRecordRefV1
  published_release_ref: exact ProductReleasePublicationRefV1
  publication_state_ref: exact ProductPublicationStateRecordRefV1
  publication_state_authorization_ref:
    exact ProductAuthorityStateAuthorizationRecordRefV1
  publication_state_head_ref:
    exact ProductAuthorityStateHeadRecordRefV1
  required_completion_scope_bindings[1..8192]:
    sorted unique {
      requirement_ref: exact McdContractRefV1(kind=requirement),
      evidence_class_ref: exact EvidenceClassRefV1,
      required_host_scope:
        {kind=not_applicable}
        | {kind=host_independent}
        | {
            kind=exact_set,
            host_profile_refs[1..64]:
              sorted unique exact TargetProfileRefV1(
                profile_kind=build_host | editor_host)
          },
      required_target_scope:
        {kind=not_applicable}
        | {kind=target_independent}
        | {
            kind=exact_set,
            target_profile_refs[1..64]:
              sorted unique exact TargetProfileRefV1(
                profile_kind=runtime_target)
          },
      required_locale_scope:
        {kind=not_applicable}
        | {kind=locale_independent}
        | {
            kind=exact_set,
            locale_profile_refs[1..64]:
              sorted unique exact LocaleProfileRefV1
          },
      required_reference_dimension_scope:
        {kind=not_applicable}
        | {kind=dimension_independent}
        | {
            kind=exact_set,
            reference_dimensions[1..2]:
              sorted unique two_d | three_d
          },
      required_distribution_subject_refs[0..8192]:
        sorted unique exact ProductDistributionSubjectRefV1
    }
  completion_evidence_bindings[1..65535]:
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
          },
      satisfied_distribution_subject_refs[0..8192]:
        sorted unique exact ProductDistributionSubjectRefV1
    }
  completion_state: approved | rejected
  required_minus_satisfied_gap_content_hash: SHA-256
  rejection_reason_bindings[0..64]:
    sorted unique {
      reason_kind: evidence_gap | policy_or_authority_denial,
      evidence_ref: exact EvidenceRefV1
    }
  product_completion_decision_subject_content_hash: SHA-256

ProductCompletionDecisionRecordV1
  product_completion_decision_record_id: StableId
  product_completion_decision_record_version: 1
  subject_ref: exact ProductCompletionDecisionSubjectRefV1
  qualified_completion_authorities[1..16]:
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
      exact MirakanSignedRecordV1(purpose=product_completion_decision)
  product_completion_decision_record_content_hash: SHA-256

ProductCompletionAuthorityStateV1
  completion_authority_state_id: StableId
  completion_authority_state_version: positive u32
  authority_scope_subject_ref:
    exact ProductCompletionAuthorityScopeSubjectRefV1
  completion_decision_record_ref:
    exact ProductCompletionDecisionRecordRefV1
  expected_previous_state_ref:
    null | exact ProductCompletionAuthorityStateRefV1
  authority_state: current | superseded | revoked
  successor_completion_decision_record_ref:
    null | exact ProductCompletionDecisionRecordRefV1
  effective_at: RFC 3339 timestamp
  state_basis_evidence_refs[1..64]:
    sorted unique exact EvidenceRefV1
  completion_authority_state_content_hash: SHA-256

ProductCompletionAuthorityScopeSubjectV1
  active_product_definition_ref: exact ActiveProductDefinitionRefV1
  claim_scope_ref: exact ProductClaimScopeRefV1
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  completion_requirement_projection_ref:
    exact ProductCompletionRequirementProjectionRefV1
  published_release_ref: exact ProductReleasePublicationRefV1
  completion_authority_scope_subject_content_hash: SHA-256
```

`required_completion_scope_bindings[]`はCompletion Requirement Projectionの各bindingを同じpairでexactly one rowへ写し、Projectionが持つHost、runtime Target、locale、2D／3D Reference Dimensionのclosed branchをbyte／set equalityで保持する。各Projectionの`required_distribution_scope.kind=exact_set`はLifecycle Distribution Coverage Projectionへjoinし、kind一致に加えて、そのpairのrequired Host、Target、locale、DimensionとCoverageのhost／target／SDK／template／sample／documentation／Pack／workflow associationが一致するexact Subjectだけへ決定論的に解決する。`kind=not_applicable`はSubject集合exact emptyである。global subject kindはProjectionがそのkindを明示した場合だけ全Claim Scopeへ適用し、その集合を`required_distribution_subject_refs[]`へ保存する。同kindという理由だけで別Host／Target／locale／DimensionのSubjectを含めず、逆にpairのapplicabilityと一致するSubjectを省略しない。Projection pairのmissing／extra、別pairへのscope付替え、independent／not-applicable／exact-set branch変換、kindに属さないSubject、Publication Projection外Subjectを拒否し、Completion作成者がscopeをinlineで縮小または拡張しない。

exact Subject join後、各Completion pair `p`のHost、runtime Target、locale、Reference dimension cardinality `h(p), t(p), l(p), d(p)`は`exact_set`のmember件数、independentまたは`not_applicable` branchは1件とし、`s(p) = |required_completion_scope_bindings[p].required_distribution_subject_refs[]|`とする。四scalar軸とDistribution Subject unionを同じEvidence row集合で同時にcoverするsound上限を`completion_evidence_carrier_upper_bound(p) = h(p) + t(p) + l(p) + d(p) + max(1, s(p)) - 4`とする。`s(p)>0`ならrequired Subjectを少なくとも一件coverするEvidenceを、`s(p)=0`なら任意のsatisfying Evidenceをbase rowに選び、各scalar軸の未cover memberと、前者ではbase rowがcoverする一Subject以外の未cover Subjectごとに高々一rowを追加できるため、各axis-wise unionがsatisfiableならこの上限でcoverできる。全pairについてこの値をchecked sumした総数はCompletion Decision Subject生成前かつ署名前に65,535以下でなければならない。checked overflow、上限超過、truncate、別pair collapse、Evidenceが複数Distribution Subjectを必ずaggregateするという仮定またはProduct Planの四scalar軸sub-boundだけによる代用ではSubjectを生成・署名しない。

`approved` Completionは、Completion Requirement Projectionの`required_completion_bindings[]`、`required_completion_scope_bindings[]`、`completion_evidence_bindings[]`の`{requirement_ref,evidence_class_ref}` projectionを三者set equalityにし、pairごとにcanonical `required − satisfied` gapを再計算してempty hashへ一致させ、`rejection_reason_bindings=[]`を要求する。同一pairに複数Evidenceがある場合はそのpair内だけでHost、Target、locale、Dimension、Distribution Subjectのcanonical unionを計算し、同じpairのrequired scopeと各々set equalityにする。同じEvidence classの別requirement、同じrequirementの別class、全pairを跨ぐglobal unionまたはscope外Evidenceを代用しない。global unionはPublication／Claim Scopeとの補助整合性だけに使い、pair-level不足を補完しない。cross-host、cross-target、cross-locale、cross-dimension、cross-distribution-subject、cross-release、class-only collapse、一pairのempty scopeを別pairのoverclaimで埋めるcaseを拒否する。2Dと3Dが同じpairで要求される場合も両dimensionを同じRelease内のEvidence unionで満たし、一方を他方で代用しない。

Publication／Completionへ参加する全`EvidenceRefV1`／`QualificationReceiptRefV1`は、set比較前にAI Verification OwnerのVerification Semantic Admissibility Predicate v1を通過しなければならない。generic wrapperのsubject contract集合、verification scope、Scenario／branch、owner-specific completed record、signed recordをread-backし、hash-only Ref、invalid tupleの両辺複写、別route／Target／locale／dimensionでvalidなEvidenceを同pairへ数えない。

`rejected` Completionはimmutable audit Decisionとしてだけ保持でき、`rejection_reason_bindings[]`をnon-emptyにする。`reason_kind=evidence_gap`が一件でもあればcanonical gapはnon-emptyで、各Evidenceはgap内のexact requirement／Evidence class／scope不足へ解決しなければならない。`policy_or_authority_denial`だけの拒否はgap emptyまたはnon-emptyを許すが、Evidenceは同じCompletion scopeの有効なpolicy／authority denialへ解決する。approved＋reason non-empty、rejected＋reason empty、evidence-gap＋gap empty、canonical再計算と異なるgap hashを拒否する。branch discriminator、canonical gap hash、reason kind／Evidence集合はCompletion Subject hashと完成Decision Recordの署名subjectへ含める。rejected recordは未完了または拒否の監査Evidenceにはできるが、Completion Evidence、support開始、Product completed claimまたはProduct Plan completion gateへ数えない。

Release Decisionはapprovedで、Decision署名、Release State、State Authorization、current署名済みHead、freshness、revocationが有効でなければならない。Publicationは同じDecision／Engine Release／Manifest／Claim Scope／Publication Projectionを束縛し、Publication State `published`、有効なState Authorization、同じStateを指すcurrent署名済みHeadを必須とする。`partially_published`、missing、withdrawn、superseded、stale Head、upload-only、approval-onlyを受理しない。Architecture文書、Release／Completion subject、署名wrapperまたはPublication record自身をrequired Evidenceへ数えない。

Completion Recordの各`signed_records[].subject_sha256`は`signed_records`配列とRecord自己hashを除く完成Record payloadの同じRFC 8785 JCS SHA-256へ一致させる。Signer subject projectionは`qualified_completion_authorities[].authority_subject_ref`とset equalityにし、同一subjectを重複計上しない。署名、qualification Role、全quorum rule、independence、issued-at、expiry、current revocation snapshotはProduct Release Decisionと同じGovernance規則を使うが、transport Role／Key purposeはsingleton `product_completion_decision`としてRelease purposeから分離する。

Completion authority scopeは同じActive Product Definition、Claim Scope、Engine Release、Completion Requirement Projection、published Releaseのexact tupleであり、Product line global、Authority Service globalまたはCompletion Decision ID単独へ縮退しない。scope recordをgeneric `ProductAuthorityScopeSubjectRefV1`へ投影するとき、ownerは`mirakan.arch.product-publication-completion`、typeは`ProductCompletionAuthorityScopeSubjectV1`、content hashはlocal recordとbyte equalityにする。

Completion Decisionをauthoritativeなcurrent completionとして使用するには`completion_state=approved`なSubjectを指す`ProductCompletionAuthorityStateV1`、そのStateを署名承認する`ProductAuthorityStateAuthorizationRecordV1`、exact `{Authority Service,owner,type,scope}`から導出したstream keyのcurrent `ProductAuthorityStateHeadRecordV1`を一組で必要とする。`establish_current`はapproved Subjectだけを受理し、rejected Decisionを`authority_state=current`、support済みCompletionまたはcompleted Product stateへ遷移させない。Rejected DecisionはCompletion authority streamへcurrent Stateを作らず、immutable audit recordとしてのみ保持する。State Subject投影のownerは`mirakan.arch.product-publication-completion`、typeは`ProductCompletionAuthorityStateV1`、scopeは同StateのCompletion scopeを上記generic Refへ投影した値、ID／version／hashはlocal Refとbyte equalityにする。最初のState／Authorization／Head、更新CAS、transition kind、signature、quorum、freshness、revocation、sequence、dual-genesis、wrong-stream／scope rejection、lineageはRelease Decision §5と同じstream-key規則で、`current`はsuccessor null、`superseded`はsuccessor必須、`revoked`はsuccessor optionalとする。Completion Record単独、rejected Record、unsigned State、stale Headまたは最大version推測をProduct completionにしない。

## 7. exact Ref

| Ref | Field |
|---|---|
| `ProductPublicationProjectionRefV1` | `{publication_projection_id, publication_projection_version=1, publication_projection_content_hash}` |
| `ProductReleasePublicationRefV1` | `{product_release_publication_id, product_release_publication_version=1, product_release_publication_content_hash}` |
| `ProductPublicationAuthorityScopeSubjectRefV1` | `{active_product_definition_ref, claim_scope_ref, engine_release_binding_ref, publication_projection_ref, publication_authority_scope_subject_content_hash}` |
| `ProductPublicationStateRecordRefV1` | `{publication_state_id, publication_state_version, publication_state_content_hash}` |
| `ProductSupportPublicationStartBindingRefV1` | `{support_publication_start_binding_id, support_publication_start_binding_version=1, support_publication_start_binding_content_hash}` |
| `ProductCompletionDecisionSubjectRefV1` | `{product_completion_decision_subject_id, product_completion_decision_subject_version=1, product_completion_decision_subject_content_hash}` |
| `ProductCompletionDecisionRecordRefV1` | `{product_completion_decision_record_id, product_completion_decision_record_version=1, product_completion_decision_record_content_hash}` |
| `ProductCompletionAuthorityScopeSubjectRefV1` | `{active_product_definition_ref, claim_scope_ref, engine_release_binding_ref, completion_requirement_projection_ref, published_release_ref, completion_authority_scope_subject_content_hash}` |
| `ProductCompletionAuthorityStateRefV1` | `{completion_authority_state_id, completion_authority_state_version, completion_authority_state_content_hash}` |

## 8. failureと禁止fallback

- Release approval、signing、upload、submission、Store approval、public read-backを相互aliasにしない。
- partial publicationをProduct-wide success、support開始またはCompletionへ昇格しない。
- withdrawn／superseded Publication、expired／revoked／superseded Decision、unsigned State、未authorization State、stale／unsigned Headをcurrent availabilityへ使わない。
- caller時刻、Website表示、Store listing名、release tagからeffective publication timeまたはcurrent headを推測しない。
- Publication／Completion refをRelease Content Manifest、Engine Release、Release DecisionまたはPlatform Receiptへ埋め戻さない。
- rejected Completionをcurrent Authority State、support開始、Product completionまたはcompleted claimへ昇格しない。

## 9. Qualification

設計上必要なEvidence classは、Publication route applicability完全分割、Projection再現、Distribution Subject／artifact coverage、channel key uniqueness、success／failed／pending route branch、successful／failed／pending／missing partition、Decision／Manifest／Receipt byte equality、approval-not-publication negative、upload-not-publication negative、partial failure、effective time、public read-back、support start binding、Publication／Completion scope projector、stream-key-derived State／Head identity、dual-genesis／wrong-stream／wrong-scope negative、state authorization／current-head CAS、stale Decision／State／Head negative、requirement identity付きcompletion required／supplied set equality、pair-level scope equality、cross-target／locale／dimension／distribution／release／class negative、approved／rejected reason cardinality、evidence-gap／policy-denial branch、rejected-current negative、canonical gap、Completion authority signature／quorum／revocation、hash-cycle negativeである。

文書と型の存在は、Platform submission、Store approval、public availability、support開始、Product completion、Publication storeまたはauthorityが実在する証拠ではない。

## 10. 完了条件

- Release authorizationとactual publicationが別Owner／別identityである。
- required channel集合が外部Projectionから決まり、全成功時だけ`published`になる。
- support開始がexact Publication refとeffective timestampへ束縛される。
- Completionがcurrent published scopeと署名済みauthorityを必須とする。
- `Release Decision → Platform Receipt → Publication → Completion`の一方向DAGが成立する。
