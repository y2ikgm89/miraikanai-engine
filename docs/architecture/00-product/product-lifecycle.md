# Miraikanai Engine Product Lifecycle Contract

- 文書ID: mirakan.arch.product-lifecycle
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: 第三者Developer向けProject bootstrap、Engine source／Host／Editor／Tool／Installer／Target別配布Packageを閉じるrelease content identity、Project source provenance、claim-derived Target×2D／3D Reference carrierと全Reference requirement Qualification、Manifest／SDK／Documentationをflattenするrelease Sample universeとWorkflow Sample全件のoperation-specific Qualification、Runtime Entry launch root／exact Launch Selection composition、SDK／Template／Sample／Documentation／first-party licenseのrelease binding、GUI／CLI／headless／Native SDK／external IDE／MCP／AI automationのclaim-derived Operation journey parity、versioned client／Agent Host profile別cross-surface conformance、Legal／IP reviewへ渡すrelease content／distribution／license domain evidence、Engine／Project update composition、Pack lifecycle集約、repair／support window、全配布container別NOTICE presentation、Product lifecycle E2E acceptance
- 非正本範囲: Product intent／MVP／release／completion Decision、Legal／IP applicability・法的判断・review authorityの意味、AI Production Run／Agent Host profileの意味、ProjectRevision／ChangeSet、public C++ APIの意味／stability、Project test semantics、Privacy data-flow意味、Build／Cook／Package／Signing／Pack transactionのdomain semantics、dependency version／SBOM source、migration class、SupportBundle schema、Evidence envelope。各Ownerを参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Product Plan](product-plan.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Product Privacy／Data Governance](../01-governance/product-privacy-data-governance.md)、[Project State](../03-authoring/project-state.md)、[AI Production Orchestration](../03-authoring/ai-production-orchestration.md)、[Game Production Loop](../03-authoring/game-production-loop.md)、[Developer Testing](../03-authoring/developer-testing.md)、[Native Game Module](../03-authoring/native-game-module.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[Pack Contract](../08-packs/pack-contract.md)
- 関連文書: [Product Release Decision](product-release-decision.md)、[Product Publication／Completion](product-publication-completion.md)、[Product Legal／IP Governance](../01-governance/product-legal-ip-governance.md)、[Product Security](../01-governance/product-security.md)、[Core Architecture](../02-foundation/core-architecture.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Editor Workspace／UX](../03-authoring/editor-workspace-ux.md)、[Debugging／Observability／Replay](../04-runtime/debugging-observability-replay.md)、[Windows](../07-platform/windows.md)、[Android](../07-platform/android.md)、[Apple](../07-platform/apple.md)、[P0 Architecture／Legal-IP Ownership Decision](../decisions/2026-08-04-p0-architecture-and-legal-ip-ownership.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec）
- 外部根拠確認日: 2026-08-04

## 1. 結論と所有境界

Product Lifecycleは、第三者DeveloperがEngine releaseと利用条件を取得し、Projectを作成し、学習し、公開SDKで拡張し、Project testを実行し、build／cook／package／launchし、Privacy／NOTICE／redistribution条件を確認して配布し、更新、repair、診断、supportを受けるまでの製品横断compositionとacceptanceを所有する。個別domainのSchemaまたは処理を再定義せず、各Ownerのexact artifact／Receiptを同じrelease、Project revision、Candidate、Targetへ束縛する。

Product intent、MVP、Product release／stop／completion Gateは[Product Plan](product-plan.md)、Legal source／jurisdiction／applicability／authorized review／Decisionは[Product Legal／IP Governance](../01-governance/product-legal-ip-governance.md)、AI Production surface／Agent Host Profileは[AI Production Orchestration](../03-authoring/ai-production-orchestration.md)、Product data flow／consent／retention／export／deleteは[Product Privacy／Data Governance](../01-governance/product-privacy-data-governance.md)、`ProjectRevision`とatomic commitは[Project State](../03-authoring/project-state.md)、Project test semanticsは[Developer Testing](../03-authoring/developer-testing.md)、public C++ API catalog／stabilityは[Native Game Module](../03-authoring/native-game-module.md)、Operation authorization／Audit bindingは[Executable Contracts](../02-foundation/executable-contracts.md)と[AI Security／Approval](../01-governance/ai-security-approval.md)、Build／Cook／Package／SigningはCore、Asset、Runtime Package、各Platform、dependency lock／third-party license／SBOM／notice sourceは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、migration class、rollback eligibility、data／public-contract復元意味は[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、`SupportBundleV1`は[Debugging／Observability／Replay](../04-runtime/debugging-observability-replay.md)、Evidence envelope／signature／retention／revocationは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)だけが所有する。本書はfirst-party Engine／Editor／SDK license grantとそのpresentation、Product surfaceとversioned client profileのrelease compositionを所有し、Release Content Manifest、Distribution Coverage、NOTICE／license EvidenceをLegal reviewの一方向inputとして供給する。Lifecycle Acceptance自身はLegal Decisionを発行、検証または内包せず、[Product Release Decision](product-release-decision.md)がLifecycle AcceptanceとLegal／IP Decisionを同じRelease subjectへ合流させる。`ProductPublicationRecoveryPolicyV1`はCompatibility判断後のCandidate publication、配布済みrelease、不可逆な外部actionの製品横断orchestrationだけを所有し、migration／rollback可能性または復元後data semanticsを決定しない。

本書はProduct lifecycle契約のtarget designである。対応するSchema、Operation、Template、Sample、Documentation bundle、Fixture、ReceiptはRepositoryに存在せず、すべて未materialize／未Activationである。

## 2. 共通規則

- 全objectはclosedであり、未知Field、重複Field、NaN／Inf、範囲外値を拒否する。
- 全Refは参照Ownerが定義するID、positive version／revision、content hashを持つexact Refである。ID-only、表示名、path、`latest`、近いversion、同名別Owner、別Targetへfallbackしない。
- `sorted unique`配列は各要素のcanonical identity byte順にstrict sortし、duplicateを拒否する。
- content hashは自己hash Fieldだけを除くclosed canonical bytesを、型ごとのASCII domain separatorとlength framingしてSHA-256する。
- Editor GUI、CLI、headless、Native SDK、external IDE、MCP、AI automationは、Product Planがapplicableとしたjourneyで同じtyped request、Operation、authorization、validation、candidate hash、semantic result、Receipt／diagnosticを使用する。表示、transport、interactive prompt、progress projectionだけをsurface固有にできる。SDK、MCPおよびAI Agentを一つのsurfaceへcollapseしない。
- clean-breakは未公開かつ未materializeの内部Schemaにだけ適用する。一度公開したProject、Save、Package、ReleaseはCompatibility Ownerのversioned migrationとlast-known-good規則に従う。

本書が新設するhash型のASCII domain separatorは次である。

| 型 | ASCII domain separator |
|---|---|
| `EngineSourceSnapshotV1` | `MIRAKAN_ENGINE_SOURCE_SNAPSHOT_V1` |
| `ProductReleaseHostDistributionV1` | `MIRAKAN_PRODUCT_RELEASE_HOST_DISTRIBUTION_V1` |
| `ProductReleaseTargetPackageV1` | `MIRAKAN_PRODUCT_RELEASE_TARGET_PACKAGE_V1` |
| `ProductReleaseTargetRuntimeEntrySetHashV1` | `MIRAKAN_PRODUCT_RELEASE_TARGET_RUNTIME_ENTRY_SET_V1` |
| `ProductDistributionSubjectV1` | `MIRAKAN_PRODUCT_DISTRIBUTION_SUBJECT_V1` |
| `ProductDistributionControlApplicabilityV1` | `MIRAKAN_PRODUCT_DISTRIBUTION_CONTROL_APPLICABILITY_V1` |
| `ProductDistributionCoverageProjectionV1` | `MIRAKAN_PRODUCT_DISTRIBUTION_COVERAGE_PROJECTION_V1` |
| `ProductReleaseContentManifestV1` | `MIRAKAN_PRODUCT_RELEASE_CONTENT_MANIFEST_V1` |
| `EngineReleaseBindingV1` | `MIRAKAN_ENGINE_RELEASE_BINDING_V1` |
| `SdkDistributionManifestV1` | `MIRAKAN_SDK_DISTRIBUTION_MANIFEST_V1` |
| `ProductLicenseGrantV1` | `MIRAKAN_PRODUCT_LICENSE_GRANT_V1` |
| `ProjectBootstrapProfileV1` | `MIRAKAN_PROJECT_BOOTSTRAP_PROFILE_V1` |
| `ProjectTemplateManifestV1` | `MIRAKAN_PROJECT_TEMPLATE_MANIFEST_V1` |
| `SampleProjectManifestV1` | `MIRAKAN_SAMPLE_PROJECT_MANIFEST_V1` |
| `ReleaseProjectTestRequirementSetV1` | `MIRAKAN_RELEASE_PROJECT_TEST_REQUIREMENT_SET_V1` |
| `DocumentationEntryV1` | `MIRAKAN_DOCUMENTATION_ENTRY_V1` |
| `DocumentationLinkV1` | `MIRAKAN_DOCUMENTATION_LINK_V1` |
| `DocumentationSnippetFixtureV1` | `MIRAKAN_DOCUMENTATION_SNIPPET_FIXTURE_V1` |
| `DocumentationQualificationRequirementSetV1` | `MIRAKAN_DOCUMENTATION_QUALIFICATION_REQUIREMENT_SET_V1` |
| `ProductSurfaceClientProfileV1` | `MIRAKAN_PRODUCT_SURFACE_CLIENT_PROFILE_V1` |
| `ProductCrossSurfaceConformanceProfileV1` | `MIRAKAN_PRODUCT_CROSS_SURFACE_CONFORMANCE_PROFILE_V1` |
| `ProductSurfaceParityReceiptV1` | `MIRAKAN_PRODUCT_SURFACE_PARITY_RECEIPT_V1` |
| `DocumentationQualificationReceiptV1` | `MIRAKAN_DOCUMENTATION_QUALIFICATION_RECEIPT_V1` |
| `DocumentationBundleManifestV1` | `MIRAKAN_DOCUMENTATION_BUNDLE_MANIFEST_V1` |
| `ProductUpdatePlanV1` | `MIRAKAN_PRODUCT_UPDATE_PLAN_V1` |
| `ProductInstallationChannelV1` | `MIRAKAN_PRODUCT_INSTALLATION_CHANNEL_V1` |
| `ProductInstalledStateV1` | `MIRAKAN_PRODUCT_INSTALLED_STATE_V1` |
| `ProductLifecycleTransitionRequirementSetV1` | `MIRAKAN_PRODUCT_LIFECYCLE_TRANSITION_REQUIREMENT_SET_V1` |
| `SdkQualificationRequirementSetV1` | `MIRAKAN_SDK_QUALIFICATION_REQUIREMENT_SET_V1` |
| `SupportQualificationRequirementSetV1` | `MIRAKAN_SUPPORT_QUALIFICATION_REQUIREMENT_SET_V1` |
| `ProductPublicationRecoveryPolicyV1` | `MIRAKAN_PRODUCT_PUBLICATION_RECOVERY_POLICY_V1` |
| `ProductNoticePresentationV1` | `MIRAKAN_PRODUCT_NOTICE_PRESENTATION_V1` |
| `ProductNoticePresentationEntryV1` | `MIRAKAN_PRODUCT_NOTICE_PRESENTATION_ENTRY_V1` |
| `ProductLifecycleAcceptanceV1` | `MIRAKAN_PRODUCT_LIFECYCLE_ACCEPTANCE_V1` |
| `ProductSupportWindowV1` | `MIRAKAN_PRODUCT_SUPPORT_WINDOW_V1` |
| `ProductSupportWindowClosureV1` | `MIRAKAN_PRODUCT_SUPPORT_WINDOW_CLOSURE_V1` |
| `ProductSupportChannelV1` | `MIRAKAN_PRODUCT_SUPPORT_CHANNEL_V1` |
| `ProductBootstrapPolicyV1` | `MIRAKAN_PRODUCT_BOOTSTRAP_POLICY_V1` |

本書Ownerのexact Refは次のclosed tupleである。

| Ref | Field |
|---|---|
| `EngineSourceSnapshotRefV1` | `{engine_source_repository_id, engine_source_schema_version=1, engine_source_revision, engine_source_snapshot_content_hash}` |
| `ProductReleaseHostDistributionRefV1` | `{host_distribution_id, host_distribution_version, host_distribution_content_hash}` |
| `ProductReleaseTargetPackageRefV1` | `{target_package_id, target_package_version, target_package_content_hash}` |
| `ProductDistributionSubjectRefV1` | `{distribution_subject_id, distribution_subject_version=1, distribution_subject_content_hash}` |
| `ProductDistributionControlApplicabilityRefV1` | `{control_applicability_id, control_applicability_version=1, control_applicability_content_hash}` |
| `ProductDistributionCoverageProjectionRefV1` | `{distribution_coverage_projection_id, distribution_coverage_projection_version=1, distribution_coverage_projection_content_hash}` |
| `ProductReleaseContentManifestRefV1` | `{release_content_manifest_id, release_content_manifest_version, release_content_manifest_content_hash}` |
| `EngineReleaseBindingRefV1` | `{engine_release_id, engine_release_version, release_binding_content_hash}` |
| `SdkDistributionManifestRefV1` | `{sdk_distribution_id, sdk_distribution_version, sdk_distribution_content_hash}` |
| `ProductLicenseGrantRefV1` | `{license_grant_id, license_grant_version, license_grant_content_hash}` |
| `ProjectBootstrapProfileRefV1` | `{bootstrap_profile_id, bootstrap_profile_version, bootstrap_profile_content_hash}` |
| `ProjectTemplateManifestRefV1` | `{template_id, template_version, template_content_hash}` |
| `SampleProjectManifestRefV1` | `{sample_id, sample_version, sample_content_hash}` |
| `ReleaseProjectTestRequirementSetRefV1` | `{release_project_test_requirement_set_id, release_project_test_requirement_set_version=1, release_project_test_requirement_set_content_hash}` |
| `DocumentationEntryRefV1` | `{documentation_entry_id, documentation_entry_version, documentation_entry_content_hash}` |
| `DocumentationLinkRefV1` | `{documentation_link_id, documentation_link_version, documentation_link_content_hash}` |
| `DocumentationSnippetFixtureRefV1` | `{snippet_fixture_id, snippet_fixture_version, snippet_fixture_content_hash}` |
| `DocumentationQualificationRequirementSetRefV1` | `{documentation_requirement_set_id, documentation_requirement_set_version=1, documentation_requirement_set_content_hash}` |
| `ProductSurfaceClientProfileRefV1` | `{surface_client_profile_id, surface_client_profile_version, surface_client_profile_content_hash}` |
| `ProductCrossSurfaceConformanceProfileRefV1` | `{cross_surface_profile_id, cross_surface_profile_version, cross_surface_profile_content_hash}` |
| `ProductSurfaceParityReceiptRefV1` | `{parity_receipt_id, parity_receipt_version, parity_receipt_content_hash}` |
| `DocumentationQualificationReceiptRefV1` | `{documentation_receipt_id, documentation_receipt_version, documentation_receipt_content_hash}` |
| `DocumentationBundleManifestRefV1` | `{documentation_bundle_id, documentation_bundle_version, bundle_content_hash}` |
| `ProductUpdatePlanRefV1` | `{update_plan_id, update_plan_version, update_plan_content_hash}` |
| `ProductInstallationChannelRefV1` | `{installation_channel_id, installation_channel_version, installation_channel_content_hash}` |
| `ProductInstalledStateRefV1` | `{installed_state_id, installed_state_version=1, installed_state_content_hash}` |
| `ProductLifecycleTransitionRequirementSetRefV1` | `{transition_requirement_set_id, transition_requirement_set_version=1, transition_requirement_set_content_hash}` |
| `SdkQualificationRequirementSetRefV1` | `{sdk_qualification_requirement_set_id, sdk_qualification_requirement_set_version=1, sdk_qualification_requirement_set_content_hash}` |
| `SupportQualificationRequirementSetRefV1` | `{support_qualification_requirement_set_id, support_qualification_requirement_set_version=1, support_qualification_requirement_set_content_hash}` |
| `ProductPublicationRecoveryPolicyRefV1` | `{recovery_policy_id, recovery_policy_version, recovery_policy_content_hash}` |
| `ProductNoticePresentationRefV1` | `{presentation_id, presentation_version, presentation_content_hash}` |
| `ProductNoticePresentationEntryRefV1` | `{presentation_entry_id, presentation_entry_version, presentation_entry_content_hash}` |
| `ProductSupportWindowRefV1` | `{support_window_id, support_window_version, support_window_content_hash}` |
| `ProductSupportWindowClosureRefV1` | `{support_window_closure_id, support_window_closure_version, support_window_closure_content_hash}` |
| `ProductSupportChannelRefV1` | `{channel_id, channel_version, channel_content_hash}` |
| `ProductBootstrapPolicyRefV1` | `{bootstrap_policy_id, bootstrap_policy_version, bootstrap_policy_content_hash}` |
| `ProductLifecycleAcceptanceRefV1` | `{acceptance_id, acceptance_version, acceptance_content_hash}` |

各RefのFieldは解決先recordとbyte equalityにする。別recordのhash、ID／versionだけ、隣接するrelease、同じpackage pathから補完しない。

## 3. Engine release binding

```text
EngineSourceSnapshotV1
  engine_source_repository_id: StableId
  engine_source_schema_version: 1
  engine_source_revision: non-empty immutable ASCII revision
  source_tree_artifact_ref: exact ArtifactRefV1
  public_source_policy_ref: exact SourceAvailabilityPolicyRefV1
  engine_source_snapshot_content_hash: SHA-256

ProductReleaseHostDistributionV1
  host_distribution_id: StableId
  host_distribution_version: positive u32
  host_profile_ref:
    exact TargetProfileRefV1(profile_kind=build_host | editor_host)
  engine_source_snapshot_ref: exact EngineSourceSnapshotRefV1
  editor_artifact_refs[1..64]:
    sorted unique exact ArtifactRefV1
  engine_host_runtime_artifact_refs[1..64]:
    sorted unique exact ArtifactRefV1
  project_browser_launcher_artifact_ref: exact ArtifactRefV1
  cli_headless_runner_artifact_refs[1..32]:
    sorted unique exact ArtifactRefV1
  build_cook_package_debug_repair_tool_artifact_refs[1..256]:
    sorted unique exact ArtifactRefV1
  installer_layout_manifest_artifact_ref: exact ArtifactRefV1
  candidate_ref: exact PreparedCandidateRefV1
  toolchain_closure_ref: exact BuildToolchainClosureRefV1
  public_contract_set_ref: exact PublicContractSetRefV1
  distribution_entry_set_hash: SHA-256
  host_distribution_content_hash: SHA-256

ProductReleaseTargetPackageV1
  target_package_id: StableId
  target_package_version: positive u32
  package_role: game_project | reference_2d | reference_3d
  target_profile_ref: exact TargetProfileRefV1
  source_repository_snapshot_ref: exact ProjectRepositorySnapshotRefV1
  source_closure_ref: exact ProjectSourceClosureRefV1
  source_transport_artifact_ref:
    exact ArtifactRefV1(
      artifact_kind=project_source_transport,
      schema_version=1)
  source_project_revision_ref: exact ProjectRevisionRefV1
  package_artifact_ref: exact ArtifactRefV1
  runtime_entry_distribution_package_manifest_ref:
    exact RuntimeEntryDistributionPackageManifestRefV1
  runtime_entry_package_refs[1..64]:
    sorted unique exact RuntimeEntryPackageRefV1
  runtime_entry_qualification_scenario_bindings[1..64]:
    sorted unique {
      runtime_entry_package_ref: exact RuntimeEntryPackageRefV1,
      qualification_scenario_refs[1..256]:
        sorted unique exact QualificationScenarioRefV1
    }
  package_entry_set_hash: ProductReleaseTargetRuntimeEntrySetHashV1
  target_package_content_hash: SHA-256

ProductDistributionSubjectV1
  distribution_subject_id: StableId
  distribution_subject_version: 1
  subject:
    {kind=engine_source,
     engine_source_snapshot_ref: exact EngineSourceSnapshotRefV1}
    | {kind=host_distribution,
       host_distribution_ref: exact ProductReleaseHostDistributionRefV1}
    | {kind=target_package,
       target_package_ref: exact ProductReleaseTargetPackageRefV1}
    | {kind=sdk_distribution,
       sdk_distribution_ref: exact SdkDistributionManifestRefV1}
    | {kind=project_template,
       project_template_ref: exact ProjectTemplateManifestRefV1}
    | {kind=sample_project,
       sample_project_ref: exact SampleProjectManifestRefV1}
    | {kind=documentation_bundle,
       documentation_bundle_ref: exact DocumentationBundleManifestRefV1}
    | {kind=product_license,
       product_license_grant_ref: exact ProductLicenseGrantRefV1}
    | {kind=support_material,
       support_window_ref: exact ProductSupportWindowRefV1}
    | {kind=pack_distribution,
       pack_contract_ref: exact PackContractRefV1}
  distributed_artifact_bindings[1..65535]:
    sorted unique {
      distribution_artifact_ref: exact ArtifactRefV1,
      artifact_role_kind: ProductDistributionArtifactRoleKindV1,
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
        | {kind=scope_independent},
      locale_scope:
        {
          kind=locale_profile,
          locale_profile_ref: exact LocaleProfileRefV1
        }
        | {kind=locale_independent}
    }
  distribution_entry_set_hash: SHA-256
  distribution_subject_content_hash: SHA-256

ProductDistributionControlApplicabilityV1
  control_applicability_id: StableId
  control_applicability_version: 1
  distribution_subject_ref: exact ProductDistributionSubjectRefV1
  required_control_kinds[0..9]:
    sorted unique ProductDistributionControlKindV1
  forbidden_control_kinds[0..9]:
    sorted unique ProductDistributionControlKindV1
  control_applicability_content_hash: SHA-256

ProductDistributionCoverageProjectionV1
  distribution_coverage_projection_id: StableId
  distribution_coverage_projection_version: 1
  product_definition_ref: exact ActiveProductDefinitionRefV1
  claim_scope_ref: exact ProductClaimScopeRefV1
  release_requirement_projection_ref:
    exact ProductReleaseRequirementProjectionRefV1
  distribution_subject_refs[9..8192]:
    sorted unique exact ProductDistributionSubjectRefV1
  control_applicability_refs[9..8192]:
    sorted unique exact ProductDistributionControlApplicabilityRefV1
  host_coverage[1..64]:
    sorted unique {
      host_profile_ref: exact TargetProfileRefV1(
        profile_kind=build_host | editor_host),
      host_distribution_subject_ref:
        exact ProductDistributionSubjectRefV1(kind=host_distribution)
    }
  target_dimension_coverage[0..128]:
    sorted unique {
      target_profile_ref: exact TargetProfileRefV1,
      reference_dimension: two_d | three_d,
      reference_requirement_ref:
        exact McdContractRefV1(kind=requirement),
      target_package_subject_ref:
        exact ProductDistributionSubjectRefV1(kind=target_package)
    }
  sdk_target_coverage[1..64]:
    sorted unique {
      target_profile_ref: exact TargetProfileRefV1,
      sdk_distribution_subject_ref:
        exact ProductDistributionSubjectRefV1(kind=sdk_distribution),
      target_sdk_artifact_ref: exact TargetSdkArtifactRefV1
    }
  template_target_coverage[1..16384]:
    sorted unique {
      target_profile_ref: exact TargetProfileRefV1,
      project_template_subject_ref:
        exact ProductDistributionSubjectRefV1(kind=project_template)
    }
  sample_reference_coverage[0..36864]:
    sorted unique {
      target_profile_ref: exact TargetProfileRefV1,
      reference_dimension: two_d | three_d,
      reference_requirement_ref:
        exact McdContractRefV1(kind=requirement),
      sample_project_subject_ref:
        exact ProductDistributionSubjectRefV1(kind=sample_project)
    }
  pack_distribution_coverage[0..1024]:
    sorted unique {
      pack_requirement_ref: exact McdContractRefV1(kind=requirement),
      pack_contract_ref: exact PackContractRefV1,
      pack_distribution_subject_ref:
        exact ProductDistributionSubjectRefV1(kind=pack_distribution)
    }
  documentation_coverage[1..65535]:
    sorted unique {
      locale_profile_ref: exact LocaleProfileRefV1,
      audience:
        beginner | advanced_user | project_cpp_author | operator,
      documentation_kind:
        getting_started | tutorial | public_api_reference | concept
        | migration | troubleshooting | known_limitation | legal_notice,
      documentation_entry_ref: exact DocumentationEntryRefV1,
      documentation_bundle_subject_ref:
        exact ProductDistributionSubjectRefV1(kind=documentation_bundle)
    }
  workflow_coverage[12..49152]:
    sorted unique {
      workflow_kind:
        install | project_bootstrap | author | build | test | package
        | launch | update | repair | diagnose | support | uninstall,
      host_profile_ref: null | exact TargetProfileRefV1,
      target_profile_ref: null | exact TargetProfileRefV1,
      basis_requirement_refs[1..4096]:
        sorted unique exact McdContractRefV1(kind=requirement),
      required_distribution_subject_refs[1..8192]:
        sorted unique exact ProductDistributionSubjectRefV1
    }
  publication_route_projection[1..65535]:
    sorted unique {
      distribution_subject_ref: exact ProductDistributionSubjectRefV1,
      distribution_artifact_ref: exact ArtifactRefV1,
      artifact_role_kind: ProductDistributionArtifactRoleKindV1,
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
        | {kind=scope_independent},
      locale_scope:
        {
          kind=locale_profile,
          locale_profile_ref: exact LocaleProfileRefV1
        }
        | {kind=locale_independent},
      required_routes[0..65535]:
        sorted unique {
          publication_requirement_ref:
            exact McdContractRefV1(kind=requirement),
          platform_kind: windows | android | apple,
          channel_kind: direct_distribution | managed_store,
          distribution_scope_kind:
            ProductCompletionDistributionScopeKindV1,
          artifact_role_kind: ProductDistributionArtifactRoleKindV1,
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
            | {kind=scope_independent},
          locale_scope:
            {
              kind=locale_profile,
              locale_profile_ref: exact LocaleProfileRefV1
            }
            | {kind=locale_independent}
        },
      forbidden_routes[0..65535]:
        sorted unique {
          publication_requirement_ref:
            exact McdContractRefV1(kind=requirement),
          platform_kind: windows | android | apple,
          channel_kind: direct_distribution | managed_store,
          distribution_scope_kind:
            ProductCompletionDistributionScopeKindV1,
          artifact_role_kind: ProductDistributionArtifactRoleKindV1,
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
            | {kind=scope_independent},
          locale_scope:
            {
              kind=locale_profile,
              locale_profile_ref: exact LocaleProfileRefV1
            }
            | {kind=locale_independent}
        }
    }
  projection_algorithm_id: product_distribution_coverage
  projection_algorithm_version: 1
  projection_algorithm_content_hash: SHA-256
  distribution_coverage_projection_content_hash: SHA-256

ProductReleaseContentManifestV1
  release_content_manifest_id: StableId
  release_content_manifest_version: positive u32
  product_definition_ref: exact ActiveProductDefinitionRefV1
  toolchain_closure_ref: exact BuildToolchainClosureRefV1
  public_contract_set_ref: exact PublicContractSetRefV1
  candidate_ref: exact PreparedCandidateRefV1
  engine_source_snapshot_ref: exact EngineSourceSnapshotRefV1
  host_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1(
      profile_kind=build_host | editor_host)
  target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1(profile_kind=runtime_target)
  locale_profile_refs[1..64]:
    sorted unique exact LocaleProfileRefV1
  installation_channel_refs[1..64]:
    sorted unique exact ProductInstallationChannelRefV1
  host_distribution_refs[1..64]:
    sorted unique exact ProductReleaseHostDistributionRefV1
  target_package_refs[1..256]:
    sorted unique exact ProductReleaseTargetPackageRefV1
  sdk_distribution_manifest_ref: exact SdkDistributionManifestRefV1
  project_template_manifest_refs[1..256]:
    sorted unique exact ProjectTemplateManifestRefV1
  sample_project_manifest_refs[0..256]:
    sorted unique exact SampleProjectManifestRefV1
  pack_contract_refs[0..1024]:
    sorted unique exact PackContractRefV1
  documentation_bundle_ref: exact DocumentationBundleManifestRefV1
  product_license_grant_ref: exact ProductLicenseGrantRefV1
  support_window_ref: exact ProductSupportWindowRefV1
  distribution_coverage_projection_ref:
    exact ProductDistributionCoverageProjectionRefV1
  release_content_manifest_content_hash: SHA-256

EngineReleaseBindingV1
  engine_release_id: StableId
  engine_release_version: positive u32
  product_definition_ref: exact ActiveProductDefinitionRefV1
  toolchain_closure_ref: exact BuildToolchainClosureRefV1
  supported_host_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1(
      profile_kind=build_host | editor_host)
  supported_target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1(profile_kind=runtime_target)
  supported_locale_profile_refs[1..64]:
    sorted unique exact LocaleProfileRefV1
  installation_channel_refs[1..64]:
    sorted unique exact ProductInstallationChannelRefV1
  public_contract_set_ref: exact PublicContractSetRefV1
  release_content_manifest_ref: exact ProductReleaseContentManifestRefV1
  sdk_distribution_manifest_ref: exact SdkDistributionManifestRefV1
  pack_contract_refs[0..1024]:
    sorted unique exact PackContractRefV1
  product_license_grant_ref: exact ProductLicenseGrantRefV1
  documentation_bundle_ref: exact DocumentationBundleManifestRefV1
  support_window_ref: exact ProductSupportWindowRefV1
  distribution_coverage_projection_ref:
    exact ProductDistributionCoverageProjectionRefV1
  release_binding_content_hash: SHA-256

EngineReleaseBindingRefV1
  engine_release_id: StableId
  engine_release_version: positive u32
  release_binding_content_hash: SHA-256
```

`ProductDistributionControlKindV1`は`malware_scan | vulnerability_scan | sbom | third_party_notice | first_party_license_presentation | signature | provenance | privacy_disclosure | support_disclosure`のclosed enumである。各Applicabilityのrequired／forbidden集合はdisjointで、両集合のunionをこの全enumとset equalityにする。空欄、暗黙default、subject kindだけからの推測を認めない。あるcontrolについてrequiredならexactly one以上の有効なEvidenceまたはpresentationを要求し、forbiddenならReceipt、Refまたは成功扱いを拒否する。

`ProductReleaseContentManifestV1`はReceipt、Security Binding、Lifecycle Acceptance、Release Decision、PublicationまたはEngine Release refを含まないreceipt-free baseである。ManifestのHost、runtime Target、locale集合はRelease Requirement Projectionの対応集合と各々set equalityで、Product Definition外profile、Minimum欠落またはHost／Target／localeの型間aliasを拒否する。Installation Channel集合は全Host Distributionのinstaller／managed layoutとProductが提供するPlatform Store distribution contractの完全なsetで、Engine Releaseの同集合とset equalityにする。`EngineSourceSnapshotV1`はGame Projectの`ProjectRepositorySnapshotRefV1`と別identityであり、Engine repository revision、Source tree artifact、source availability policyを閉じる。各Host Distributionは一つのBuild／Editor Host ProfileについてEditor、Engine host runtime、Project Browser／Launcher、CLI／headless runner、build／cook／package／debug／repair tool、installer／managed layoutの全配布artifactを閉じる。Host DistributionのHost projectionはManifestのHost集合とset equalityである。別Host、別architecture、別Candidate、別Toolchainまたは別Engine Sourceを合成しない。

各`ProductReleaseTargetPackageV1`は一つのTarget Profile、一つの独立したProject repository snapshot、そのSnapshotが解決するexact Project Source closure／canonical transport artifact、Snapshotの`project_revision_ref`とbyte equalityの一つのcommitted Project revision、一つの配布Package artifact、一つのRuntime Package Owner typed distribution Manifest、一件以上の外側`RuntimeEntryPackageRefV1` launch rootを閉じ、別Target／revision／artifactを近似で束ねない。Source Snapshot、closure ref／content hash、transport artifactおよびProject revisionは`RuntimeEntryDistributionPackageManifestV1`とPackage success Receiptの同Fieldへbyte equalityにし、同revision別closure／Snapshot、Snapshot外transportまたはSample S1＋Package S2 stitchを拒否する。

`runtime_entry_distribution_package_manifest_ref`は同じPackage artifact、Candidate、Target、Project revision、Runtime／public Contract Setをread-backする。`runtime_entry_package_refs[]`が解決する各`RuntimeEntryPackageV1`の`source_project_revision_ref`と`target_profile_ref`はTarget Packageの同Fieldへbyte equalityにする。この集合は同Project revision／Targetについて選択された全`RuntimeEntryPointV1`からcompileされた外側Runtime Entry集合、typed distribution Manifestの`runtime_entry_members[]` projection、Package artifact内の全launchable entry集合および§9のTarget Package Runtime Entry binding集合とset equalityである。`runtime_entry_qualification_scenario_bindings[]`のentry projectionも同じ集合とset equalityで、各entryへPackage／install／launchで要求するowner-typed scenario集合を一意に割り当てる。そのflatten `{runtime_entry_package_ref, qualification_scenario_ref}`集合はRequired Operation Journey Projectionの同Targetにapplicableなpackage／install／launch success scenarioをRuntime Entryへ解決した集合とset equalityであり、scenario名、entry名または配列位置から対応を推論しない。`package_entry_set_hash`はASCII `MIRAKAN_PRODUCT_RELEASE_TARGET_RUNTIME_ENTRY_SET_V1`と、sorted `runtime_entry_package_refs[]`のMCD canonical bytesを各`uint32_be` length framingした列だけからSHA-256し、typed集合をread-backできない単独hash、path、entry名またはinner `RuntimePackageRefV1`を代用しない。各outer entryはRuntime Package Ownerのclosed branch matrixへ従い、`entry_kind=world`だけがexact一件のinner `world_package_ref`を持ち、`ui | headless`はnullとする。UI／headless用の偽World Package、worldのnull、outer／inner Ref型の相互代用、Target Package集合、typed Manifest集合とPackage artifact集合のmissing／extraを拒否する。

Manifest内の`package_role=reference_2d | reference_3d` subsetを`{target_profile_ref, package_roleから得るreference_dimension}`へ投影した集合は、Release Requirement Projectionの`required_target_dimension_bindings[]`から得るdistinct `{target_profile_ref, reference_dimension}`集合とset equalityにし、各pairへexactly one Packageを要求する。required bindingがemptyならReference Package subsetもexact emptyである。`package_role=game_project`は一般のGame Project Packageとして別に存在できるがReference cardinalityへ数えない。2Dと3D Referenceが同じProject revisionであることは要求せず、同一Target／dimensionのduplicate、wrong Target、cross-dimension、Projection外Reference Packageを拒否する。Manifestの全runtime Target Profile projectionは`target_package_refs[]`から得られるdistinct Target集合であり、Manifestの`target_profile_refs[]`、Engine Releaseの`supported_target_profile_refs[]`とset equalityにする。ManifestのHost／locale集合もEngine Releaseの`supported_host_profile_refs[]`／`supported_locale_profile_refs[]`、Release Requirement Projectionの対応集合とset equalityで、Product definition、Toolchain、public contractは両recordでbyte equalityにする。各supported Targetは少なくとも一つのTarget Packageを持ち、ManifestのCandidate、Engine Source、全Host Distributionが同じSource closureへ解決しなければならない。

ManifestのSDK、Template、Pack、Documentation、License、Support Window refはEngine Releaseの同Fieldまたはそれらから投影した集合とbyte／set equalityにする。`sample_project_manifest_refs[]`はrelease Sampleのdirect carrierであり、SDK／Documentationから到達するSampleを除外した全Sample正本ではない。releaseに属する完全なSample集合、Reference subsetおよびworkflow subsetは§5.2の`ReleaseSampleSetV1` Named Algorithmだけが定義する。Templateは§4のrelease-independent baseであり、SampleとPackもEngine Releaseを含まない。Manifestの`pack_contract_refs[]`はDistribution Coverageの`pack_distribution_coverage[].pack_contract_ref` projection、Pack Distribution Subjectのbranch ref projection、およびProduct Definitionのrequired／bundled Pack requirementからLifecycleが解決したexact Pack集合とset equalityにする。Release bindingはProduct definition、Toolchain、Host／runtime Target／locale集合、public contract、Candidate、Engine source、Host distribution、Project／Reference Target Package、SDK、Template、`ReleaseSampleSetV1`、Pack、first-party license、Documentation、support window、Distribution Coverage Projectionを一つのimmutable identityへ閉じる。Release label、Git tag、package version、Store listingのいずれも単独ではrelease bindingにならない。[Product Security](../01-governance/product-security.md)は完成Release refと同じ`ProductReleaseContentManifestRefV1`を`ProductSecurityReleaseBindingV1`からSecurity baseline／Threat Registryへ一方向に束縛し、相互hash参照または規範依存cycleを作らない。[Product Privacy／Data Governance](../01-governance/product-privacy-data-governance.md)のacceptanceは`EngineReleaseBindingRefV1`を含まないpre-release Candidate closureを束縛し、本書のLifecycle acceptanceが同じProduct definition、Toolchain、public contract、Host／runtime Target／locale集合、Candidateを完成Release refへ照合する。

Distribution Coverage ProjectionのNamed Algorithm v1は、Product Claim MinimumとRelease Requirement Projectionからrequired Host、runtime Target、locale、Target×dimension、Documentation tuple、workflowを導出し、Manifestが列挙するbase artifactからDistribution Subjectを一意構築する。`distribution_subject_refs[]`はEngine Source、全Host Distribution、全Target Package、SDK、全Template、§5.2 `ReleaseSampleSetV1`の全Sample、全Pack、Documentation Bundle、Product License、Support Materialのcanonical unionとset equalityにする。特に`kind=sample_project` Subject projectionは`ReleaseSampleSetV1`とset equalityであり、Manifest direct fieldだけへ縮約しない。`target_dimension_coverage[]`の`{target_profile_ref, reference_dimension, reference_requirement_ref}` projectionはRelease Requirement Projectionの`required_target_dimension_bindings[]`とset equalityにする。`sample_reference_coverage[]`の同三Field projectionもそのrequired集合とset equalityに加え、Sample Subjectをexact Sample refへ解決したfull `{Sample, Target, dimension, reference requirement}` projectionを§5.2 `ReferenceSampleCoverageSetV1(M)`とset equalityにする。各Reference Sampleの全supported Targetへexactly one rowを要求し、同Targetを別Sampleだけでcoverする、distinct Sample集合だけを維持して一SampleのTargetを落とす、4,096件で打ち切ることを拒否する。Projectionが0件／2D-only／3D-only／各Targetの2D＋3Dなら、両Coverageもそれぞれexact 0件／2D-only／3D-only／同じ各Targetの両dimensionとする。`pack_distribution_coverage[]`のPack requirement projectionはProduct Definitionの`required_pack_requirement_refs[] ∪ bundled_pack_requirement_refs[]`とset equality、Pack Contract／Subject projectionはManifestの`pack_contract_refs[]`とset equalityにし、一requirementを別Packで充足、同Packの別version／hash、missing／extra Packを拒否する。各Subjectの`distributed_artifact_bindings[]`から得るdistinct artifact projectionとentry setは選択branchのbase recordが再帰的に配布するartifactの完全なsetであり、各bindingはexactly oneの主artifact role、Host／runtime Target／scope-independent execution scope、locale／locale-independent scopeを持つ。同じartifactが複数scopeへ配布される場合はfull bindingを別rowにし、artifact hash、container path、表示名、subject kindまたはOS名からscopeを補完しない。Host、SDK Target、Template Target、Pack、Documentation `{locale,audience,kind}`、Workflowの各required keyはexactly one以上のcoverage rowを持ち、Referenceは上記Projection-derived exact setを持つ。余分なrowはClaim Scope内でもReference requirement外なら拒否する。各Subjectはexactly one control Applicabilityを持ち、Applicability subject projectionとDistribution Subject集合をset equalityにする。Packを含む全Subjectは同じmalware、vulnerability、SBOM、NOTICE、first-party license、signature、provenance、privacy、support control universeでrequired／forbiddenを完全分割し、Pack固有の別control Registryを作らない。

`WorkflowCoverageKeySetV1(M)` Named Algorithmは、Manifest `M`が束縛するRelease Requirement Projectionの全Requirementをexact typed `ProductRequirementProjectionInputV1`へjoinし、各`workflow_applicabilities[]`を次のclosed規則で展開する。`host_independent`は`host_profile_ref=null`一件だけ、Host exact setは各Hostを一件ずつ生成する。`target_independent`は`target_profile_ref=null`一件だけ、`all_requirement_targets`はtyped Requirement target applicabilityをClaim Scopeへ展開した各Target、Target exact setは各Targetを一件ずつ生成する。同じworkflow kindについてindependent branchとexact／all branchを異なるRequirement間で混在させず、混在はProjection invalidとする。各Requirementの`required_distribution_scope_kinds[]`をCoverageのexact Subject branchへjoinし、同じ`{workflow,Host,Target}` keyを要求する全RequirementとSubjectをそれぞれcanonical unionする。

`workflow_coverage[]`のfull key projectionは`WorkflowCoverageKeySetV1(M)`とset equality、各rowの`basis_requirement_refs[]`と`required_distribution_subject_refs[]`は同keyのAlgorithm結果と各々set equalityである。12 workflow kind、最大64 Host、最大64 Targetのexplicit cross-productからouter carrier上限を`12 × 64 × 64 = 49152`とし、independent branchは64件のaggregateではなくnull identity一件としてだけ数える。Host／Targetの任意省略、null aggregateとexplicit cross-productの併用、別workflowのSubject流用、64 Subjectでの打切りを拒否する。これは配布物から到達できるdocumented workflow導線だけを表し、Product Operation journey qualificationではない。12 workflow kindの成功、Manifest membershipまたはOperation Activation Evidenceを、Product Planの全non-collapsed family、Native SDK／external IDE／MCP／AI automation、expected rejectionまたはfailure recoveryの代用にしない。

`publication_route_projection[]`は各Distribution Subjectの各`distributed_artifact_bindings[]` full tupleについてexactly one rowを持つ。Subject branchから得る`ProductCompletionDistributionScopeKindV1`、bindingのartifact role、execution scope、locale scopeをRelease Requirement Projectionの各`required_publication_distribution_subjects[]`へexact joinし、全Fieldがbyte equalityのrouteだけをrequired、それ以外をforbiddenへ決定論的に投影する。各rowのrequired／forbidden集合はdisjointで、unionをProduct Planの完全publication distribution subject universeとset equalityにする。全rowの`{subject,artifact,artifact role,execution scope,locale scope}` projectionは全Distribution Subjectの全binding集合とset equalityである。required routeが空のbindingは「そのartifact bindingを公開しない」という有効な完全分割であり、Productのrequired distribution subjectが全rowで一度もrequiredへ投影されない場合はmissingとしてProjection全体を拒否する。Hostをruntime Targetへ、scope-independentを一Targetへ、locale-independentを既定localeへ縮約せず、missing、extra、cross-channel、cross-scope、cross-role、cross-Host／Target、cross-locale、required／forbidden重複、同Subject外artifactまたは自己申告routeを拒否する。Publication Ownerはこの関係を推論せずexact flatten joinとして消費する。

Distribution Projection Capacity Validity Algorithm v1はManifestからProjectionを作る前に、各Subjectの再帰distinct Artifact件数が65,535以下、全Subjectのfull `{subject,artifact,artifact role,execution scope,locale scope}` binding総数が65,535以下であることをchecked arithmeticで検証する。Product Planのpublication distribution subject universeは65,535件以下で、binding総数との完全partition直積は8,388,608件以下、Algorithmが決めるrequired routeのSubject／Artifact identity付きproduct-wide flattenは65,535件以下、forbidden flattenは8,388,608件以下でなければならない。各Product publication distribution subjectのdistinct projectionはrequired flattenの同projectionとset equalityにする。Documentationのsource／rendered、normalized URL、Snippet input、Sample source／noticeを含む再帰Artifactを省略せず、Template／Subject／Artifact／role／Host／Target／scope-independent／locale／locale-independent／Requirement identityのcollapse、上限でのtruncateまたはchecked multiplication overflowをProjection生成失敗以外へ変換しない。これらは各base recordの個別上限ではなく、同じfixed carrierへ安全に投影できるrelease Manifestのderived validity条件である。

`release_content_manifest_ref`が束縛するHost Distribution、Target Package、SDK、Template、Sample、Documentation、License、Support WindowはすべてEngine releaseを含まないreceipt-free base recordである。Engine release bindingがbase refを束縛してrelease固有identityを完成し、いずれのbaseからもRelease bindingを逆参照しない。生成順は`Engine Source／Project Source／Candidate／distribution base → Product Release Content Manifest → Engine Release Binding → Lifecycle／Security acceptance → signed Product Release Decision → Platform publication Receipt → Product Publication → signed Product Completion`とし、後段refを前段hash preimageへ戻さない。

### 3.1 SDK distributionとfirst-party license

```text
SdkDistributionManifestV1
  sdk_distribution_id: StableId
  sdk_distribution_version: positive u32
  public_contract_set_ref: exact PublicContractSetRefV1
  public_api_catalog_ref: exact PublicApiCatalogRefV1
  target_sdk_entries[1..64]:
    sorted unique {
      target_profile_ref: exact TargetProfileRefV1,
      target_sdk_artifact_ref: exact TargetSdkArtifactRefV1
    }
  build_integration_artifact_refs[1..64]:
    sorted unique exact ArtifactRefV1
  debug_symbol_policy_ref: exact DebugSymbolDistributionPolicyRefV1
  sample_manifest_refs[1..64]:
    sorted unique exact SampleProjectManifestRefV1
  documentation_entry_refs[1..4096]:
    sorted unique exact DocumentationEntryRefV1
  sdk_distribution_content_hash: SHA-256

SdkQualificationRequirementBindingV1
  host_profile_ref:
    exact TargetProfileRefV1(profile_kind=build_host | editor_host)
  target_profile_ref:
    exact TargetProfileRefV1(profile_kind=runtime_target)
  target_sdk_artifact_ref: exact TargetSdkArtifactRefV1
  action:
    {kind=acquire}
    | {kind=install}
    | {kind=repair}
    | {kind=uninstall}
    | {
        kind=public_snippet_build,
        snippet_fixture_ref: exact DocumentationSnippetFixtureRefV1
      }
  qualification_scenario_ref: exact QualificationScenarioRefV1
  expected_result_branch:
    success | expected_policy_rejection | domain_failure_recovery

SdkQualificationRequirementSetV1
  sdk_qualification_requirement_set_id: StableId
  sdk_qualification_requirement_set_version: 1
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  release_content_manifest_ref: exact ProductReleaseContentManifestRefV1
  sdk_distribution_manifest_ref: exact SdkDistributionManifestRefV1
  sdk_distribution_subject_ref:
    exact ProductDistributionSubjectRefV1(kind=sdk_distribution)
  public_contract_set_ref: exact PublicContractSetRefV1
  public_api_catalog_ref: exact PublicApiCatalogRefV1
  required_sdk_qualification_bindings[1..262144]:
    sorted unique SdkQualificationRequirementBindingV1
  projection_algorithm_id: sdk_qualification_requirement_set
  projection_algorithm_version: 1
  projection_algorithm_content_hash: SHA-256
  sdk_qualification_requirement_set_content_hash: SHA-256

ProductLicenseGrantV1
  license_grant_id: StableId
  license_grant_version: positive u32
  license_text_artifact_ref: exact LicenseArtifactRefV1
  covered_product_artifact_kinds[1..16]:
    sorted unique
      engine_source | editor | engine_runtime | sdk | template
      | sample | tools | installer | documentation
      | license_material | support_material
  permitted_use_refs[1..32]: sorted unique exact LicensePermissionRefV1
  redistribution_policy_ref: exact RedistributionPolicyRefV1
  source_availability_policy_ref: exact SourceAvailabilityPolicyRefV1
  attribution_requirement_refs[0..32]:
    sorted unique exact AttributionRequirementRefV1
  warranty_disclaimer_ref: exact LegalClauseRefV1
  support_entitlement_policy_ref: exact SupportEntitlementPolicyRefV1
  legal_review_evidence_ref: exact EvidenceRefV1
  license_grant_content_hash: SHA-256
```

SDK distributionは公開API catalogとexact public contract setに一致するPublic Header、C ABI Header、library、generated Project binding、build integration、必要なdebug symbol policy、Sample、API DocumentationをTarget別artifactとして閉じる。Named Module interface、BMI、Engine private Header、Editor private API、Vendor Header、未qualified Target library、別release Sampleを混入させない。SDKの取得、install、repair、uninstallとpublic snippet buildをclean environmentで再現できなければ配布可能としない。

`target_sdk_entries[]`の`target_profile_ref` projectionはuniqueで、Engine Releaseのsupported Target集合およびDistribution Coverageの`sdk_target_coverage[].target_profile_ref` projectionとset equalityにする。各Targetへexactly one `{Target, Target SDK Artifact}` rowを持ち、Coverageのfull `{Target, SDK Subject, Target SDK Artifact}` rowへexactly oneで解決する。同Targetの二Artifact、Targetだけのaggregate、ManifestにないArtifact、65件目のTargetを受理しない。

SDK Qualification Requirement SetのNamed Algorithm v1は、Release Manifestの全Host、SDK Manifestの全Target entry、Coverageのexact SDK Subject／artifact、public contract／API catalog、Documentation Qualification Requirement Setのpublic snippet Target requirementを入力にする。acquire／install／repair／uninstallは各`{Host,Target}`へRequired Operation Journeyが要求するscenario／branchを展開し、public snippet buildは各`{Host,Snippet,Target}`へexact fixture expected branchを展開する。全rowは同じSDK Subject、Target artifact、public contract、API catalogへ解決し、Host／Target／action／scenario／branchまたはSnippetを別rowから補わない。

checked `Host × Target × action/scenario`と`Host × snippet Target requirement`のcanonical unionは262,144件以下でなければならない。Requirement Setのfull binding projection、Lifecycle AcceptanceのSDK result key projectionをset equalityにし、generic Journey Receipt、SDK manifest membership、Target一件のinstall成功、Documentation snippet resultまたはflat Receipt poolを他のHost／Target／actionへ流用しない。

First-party licenseはMiraikanai所有artifactの利用、Game distribution、SDK利用、Template／Sample改変、attribution、source availability、warranty、support entitlementを明示する。third-party dependency license、Pack publisher license、Game Developer自身のlicenseを包含しない。Web page、installer checkboxまたはpackage内textのいずれか一つだけをauthorityにせず、exact license artifactとlegal review Evidenceをrelease bindingへ固定し、Editor、SDK install、Documentation、redistributable packageから到達できるようにする。

## 4. Project bootstrap

```text
ProjectBootstrapProfileV1
  bootstrap_profile_id: StableId
  bootstrap_profile_version: positive u32
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  project_template_ref: exact ProjectTemplateManifestRefV1
  requested_target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1
  foundation_definition_closure_ref: exact FoundationDefinitionClosureRefV1
  bootstrap_policy_ref: exact ProductBootstrapPolicyRefV1
  bootstrap_profile_content_hash: SHA-256

ProductBootstrapPolicyV1
  bootstrap_policy_id: StableId
  bootstrap_policy_version: positive u32
  destination_precondition: must_not_exist
  publication_policy: single_atomic_project_revision
  staging_authority: non_authoritative
  collision_policy: reject
  unknown_template_entry_policy: reject
  maximum_initial_document_count: positive u32
  maximum_source_entry_count: positive u32
  maximum_total_source_bytes: positive u64
  bootstrap_policy_content_hash: SHA-256

ProjectTemplateManifestV1
  template_id: StableId
  template_version: positive u32
  source_tree_artifact_ref: exact ArtifactRefV1
  initial_document_refs[1..4096]:
    sorted unique exact ProjectDocumentRefV1
  required_capability_refs[0..256]:
    sorted unique exact CapabilityDefinitionRefV1
  supported_target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1
  project_test_catalog_ref: exact ProjectTestCatalogRefV1
  third_party_notice_source_artifact_refs[1..256]:
    sorted unique exact ArtifactRefV1
  template_content_hash: SHA-256
```

requested Target集合はreleaseとTemplateのsupported Target集合のintersectionのsubsetでなければならない。TemplateのSource tree、initial Document集合、Capability集合、notice集合は同じcontent hashへ閉じる。empty default Project、別release Template、同名Template、近いTargetへ暗黙置換しない。

`ProjectTemplateManifestV1`はrelease-independentなreceipt-free baseであり、Engine Release refを持たない。`ProjectBootstrapProfileV1`が完成Engine Releaseとexact Template baseを外側から束縛するrelease-bound bindingである。TemplateをRelease Content Manifestへ含めてもTemplate→Engine Releaseの逆参照がないためhash cycleを作らない。

完成`ProductReleaseContentManifestV1`を`M`とした時、`TemplateTargetCoverageSetV1(M)`は次のreceipt-free exact setだけである。

```text
TemplateTargetCoverageSetV1(M) =
  { {
      project_template_manifest_ref: template,
      target_profile_ref: target
    }
    | template ∈ M.project_template_manifest_refs[],
      target ∈ resolve(template).supported_target_profile_refs[] }

TemplateBootstrapRequirementSetV1(M) =
  { {
      project_template_manifest_ref: template,
      requested_target_profile_refs:
        canonical set intersection(
          resolve(template).supported_target_profile_refs[],
          resolve(M.distribution_coverage_projection_ref)
            .release_requirement_projection_ref
            .target_profile_refs[])
    }
    | template ∈ M.project_template_manifest_refs[] }
```

各Templateのsupported TargetはEngine Releaseのsupported Target subsetで、`TemplateTargetCoverageSetV1(M)`は最大`256 × 64 = 16384`件である。Distribution Coverageの`template_target_coverage[]`をSubjectからTemplate refへ解決したfull `{Template,Target}` projectionは同setとexact equalityにする。`TemplateBootstrapRequirementSetV1(M)`の各rowはrequested Targetがnon-emptyで、§9のBootstrap Receiptをexactly one件へ解決する。Receiptのrequestが束縛する`ProjectBootstrapProfileV1`は同Templateと同requested Target setへbyte／set equalityで、全Templateを最大256件のReceiptでcoverする。同Targetの別Template、Template表示名、Target distinct集合だけによる代用、4,096件でのCoverage打切りまたは一Receiptの別Templateへの流用を拒否する。

bootstrap requestは新規Project ID、destinationのProject-scoped root、exact profile ref、Userが明示したProject display metadataだけを入力にする。Authoring Command Gatewayが全Document、Source tree、manifest、Target selectionを一つのprepared candidateとして検証し、成功時だけ最初の`ProjectRevision`をatomic publishする。失敗時はcurrent `ProjectRevision`、openable partial Project、権威を持つdestination manifestを残さない。診断用stagingはProject State Ownerの非authority candidateとしてのみ保持できる。

## 5. Documentation bundle

```text
DocumentationEntryV1
  documentation_entry_id: StableId
  documentation_entry_version: positive u32
  kind:
    getting_started | tutorial | public_api_reference | concept
    | migration | troubleshooting | known_limitation | legal_notice
  locale_profile_ref: exact LocaleProfileRefV1
  audience: beginner | advanced_user | project_cpp_author | operator
  source_artifact_ref: exact ArtifactRefV1
  rendered_artifact_ref: exact ArtifactRefV1
  public_contract_member_refs[0..4096]:
    sorted unique exact PublicContractMemberRefV1
  documentation_entry_content_hash: SHA-256

DocumentationLinkV1
  documentation_link_id: StableId
  documentation_link_version: positive u32
  source_entry_key:
    {
      documentation_entry_id: StableId,
      documentation_entry_version: positive u32,
      expected_entry_content_hash: SHA-256
    }
  destination:
    {
      kind: internal_entry,
      entry_key:
        {documentation_entry_id: StableId, documentation_entry_version: positive u32},
      expected_entry_content_hash: SHA-256
    }
    | {
        kind: public_contract_member,
        public_contract_member_ref: exact PublicContractMemberRefV1
      }
    | {
        kind: approved_external_url,
        normalized_url_artifact_ref: exact ArtifactRefV1
      }
  documentation_link_content_hash: SHA-256

DocumentationSnippetFixtureV1
  snippet_fixture_id: StableId
  snippet_fixture_version: positive u32
  documentation_entry_ref: exact DocumentationEntryRefV1
  public_contract_member_refs[1..256]:
    sorted unique exact PublicContractMemberRefV1
  target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1(profile_kind=runtime_target)
  fixture_kind: compile | validate | execute | package
  input_artifact_ref: exact ArtifactRefV1
  expected_result:
    {kind=success, output_contract_ref: exact QualificationOutputContractRefV1}
    | {kind=diagnostic, diagnostic_contract_ref: exact DiagnosticContractRefV1}
  snippet_fixture_content_hash: SHA-256

SampleProjectManifestV1
  sample_id: StableId
  sample_version: positive u32
  sample_role: reference_2d | reference_3d | workflow
  reference_dimension: none | two_d | three_d
  source_repository_snapshot_ref:
    exact ProjectRepositorySnapshotRefV1
  source_closure_ref: exact ProjectSourceClosureRefV1
  source_project_artifact_ref: exact ArtifactRefV1
  expected_project_revision_ref: exact ProjectRevisionRefV1
  supported_target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1
  project_test_catalog_ref: exact ProjectTestCatalogRefV1
  qualification_requirements[1..16384]:
    sorted unique {
      target_applicability:
        {kind=target_profile,
         target_profile_ref: exact TargetProfileRefV1}
        | {kind=target_independent,
           target_profile_ref: null},
      qualification_scenario_ref: exact QualificationScenarioRefV1,
      expected_result_branch:
        success | expected_policy_rejection | domain_failure_recovery,
      operation_family_kind: operation_family_kind,
      operation_ref: exact McdContractRefV1(kind=operation),
      execution_branch:
        authoring | build | test | package | install | launch
    }
  documentation_entry_refs[1..256]:
    sorted unique exact DocumentationEntryRefV1
  third_party_notice_source_artifact_refs[1..256]:
    sorted unique exact ArtifactRefV1
  sample_content_hash: SHA-256

DocumentationBundleManifestV1
  documentation_bundle_id: StableId
  documentation_bundle_version: positive u32
  public_contract_set_ref: exact PublicContractSetRefV1
  entry_refs[1..65535]:
    sorted unique exact DocumentationEntryRefV1
  link_refs[0..262144]:
    sorted unique exact DocumentationLinkRefV1
  snippet_fixture_refs[1..4096]:
    sorted unique exact DocumentationSnippetFixtureRefV1
  tutorial_scenario_requirements[1..4096]:
    sorted unique {
      documentation_entry_ref: exact DocumentationEntryRefV1,
      qualification_scenario_ref: exact QualificationScenarioRefV1,
      target_applicability:
        {kind=target_independent}
        | {
            kind=exact_set,
            target_profile_refs[1..64]:
              sorted unique exact TargetProfileRefV1
          },
      expected_result_branch:
        success | expected_policy_rejection | domain_failure_recovery
    }
  sample_project_refs[0..256]:
    sorted unique exact SampleProjectManifestRefV1
  link_graph_content_hash: SHA-256
  bundle_content_hash: SHA-256

DocumentationQualificationRequirementSetV1
  documentation_requirement_set_id: StableId
  documentation_requirement_set_version: 1
  documentation_bundle_ref: exact DocumentationBundleManifestRefV1
  entry_requirement_refs[1..65535]:
    sorted unique exact DocumentationEntryRefV1
  link_requirement_refs[0..262144]:
    sorted unique exact DocumentationLinkRefV1
  snippet_target_requirements[1..262144]:
    sorted unique {
      snippet_fixture_ref: exact DocumentationSnippetFixtureRefV1,
      target_profile_ref: exact TargetProfileRefV1,
      expected_result:
        exact resolve(snippet_fixture_ref).expected_result
    }
  tutorial_requirements[1..262144]:
    sorted unique {
      documentation_entry_ref: exact DocumentationEntryRefV1,
      entry_locale_profile_ref: exact LocaleProfileRefV1,
      entry_source_artifact_ref: exact ArtifactRefV1,
      entry_rendered_artifact_ref: exact ArtifactRefV1,
      qualification_scenario_ref: exact QualificationScenarioRefV1,
      target_profile_ref: null | exact TargetProfileRefV1,
      expected_result_branch:
        success | expected_policy_rejection | domain_failure_recovery
    }
  documentation_requirement_set_content_hash: SHA-256

ReleaseProjectTestRequirementSetV1
  release_project_test_requirement_set_id: StableId
  release_project_test_requirement_set_version: 1
  release_content_manifest_ref:
    exact ProductReleaseContentManifestRefV1
  project_requirements[1..832]:
    sorted unique {
      project_subject:
        {
          kind=project_template,
          project_template_manifest_ref:
            exact ProjectTemplateManifestRefV1
        }
        | {
            kind=sample_project,
            sample_project_manifest_ref:
              exact SampleProjectManifestRefV1
          },
      project_test_catalog_ref: exact ProjectTestCatalogRefV1,
      project_test_requirement_set_ref:
        exact ProjectTestRequirementSetRefV1
    }
  release_project_test_requirement_set_content_hash: SHA-256

```

### 5.1 Documentation Qualification Requirement Named Algorithm v1

完成`DocumentationBundleManifestV1`を`B`とした時、receipt-free detached `DocumentationQualificationRequirementSetV1(B)`は次で生成する。

1. `entry_requirement_refs[]`を`B.entry_refs[]`とset equalityにする。
2. `link_requirement_refs[]`を`B.link_refs[]`とset equalityにする。
3. 各Snippet Fixtureの各`target_profile_refs[]`へfull `{Snippet Fixture,Target,expected result}` rowを一件生成する。最大件数は`4096 × 64 = 262144`である。
4. 各`tutorial_scenario_requirements[]`の`documentation_entry_ref`を`B.entry_refs[]`のexact一memberへ解決し、`kind=tutorial`、Entry RefのID／version／content hash、Entryの`locale_profile_ref`、source／rendered artifactをread-backする。non-tutorial Entry、Bundle外Entry、同ID／version別hashまたはEntry内容からのscenario推測を拒否する。
5. 各bindingについて`target_independent`は`target_profile_ref=null`一件、`exact_set`は各Target一件へ展開し、full `{Documentation Entry,Entry locale,Entry source artifact,Entry rendered artifact,scenario,Target applicability,expected branch}`を保持する。最大件数はbinding数`4096 × 64 = 262144`であり、Entry数ではなく完成bindingからchecked arithmeticで計算する。
6. `B.entry_refs[]`のうち`kind=tutorial`であるEntryのdistinct projectionと`tutorial_scenario_requirements[].documentation_entry_ref`のdistinct projectionをset equalityにし、各Tutorial Entryへ一件以上のbinding、各bindingへexactly one Tutorial Entryを要求する。Tutorial Entry without binding、orphan binding、duplicate full keyを拒否する。

Requirement Setは完成Bundleだけを参照し、BundleからRequirement SetまたはReceiptを逆参照しない。Entry、Link、SnippetまたはTutorialをTarget／Scenarioだけでcollapseせず、Snippet S1の結果をS2、Entry E1のrenderまたはtutorial実行結果をE2、en-US Entryをja-JP Entry、Link L1の到達結果をL2へ流用しない。空Link集合だけはexact emptyを許し、Snippet／Tutorial／Entryのrequired集合欠落、別Bundle、別public contract、carrier truncateを拒否する。

### 5.2 Release Sample universe Named Algorithm v1

完成`ProductReleaseContentManifestV1`を`M`とした時、release Sample集合は次の純粋projectionだけで計算する。

```text
ContentManifestSampleSetV1(M) =
  canonical set(M.sample_project_manifest_refs[])

SdkSampleSetV1(M) =
  canonical set(
    resolve(M.sdk_distribution_manifest_ref).sample_manifest_refs[])

DocumentationSampleSetV1(M) =
  canonical set(
    resolve(M.documentation_bundle_ref).sample_project_refs[])

ReleaseSampleSetV1(M) =
  canonical set union(
    ContentManifestSampleSetV1(M),
    SdkSampleSetV1(M),
    DocumentationSampleSetV1(M))

ReferenceReleaseSampleSetV1(M) =
  {sample ∈ ReleaseSampleSetV1(M)
   | resolve(sample).sample_role ∈ {reference_2d, reference_3d}}

WorkflowReleaseSampleSetV1(M) =
  ReleaseSampleSetV1(M) - ReferenceReleaseSampleSetV1(M)

SampleQualificationRequirementSetV1(sample) =
  { {
      sample_project_manifest_ref: sample,
      target_applicability: requirement.target_applicability,
      qualification_scenario_ref:
        requirement.qualification_scenario_ref,
      expected_result_branch:
        requirement.expected_result_branch,
      operation_family_kind: requirement.operation_family_kind,
      operation_ref: requirement.operation_ref,
      execution_branch: requirement.execution_branch
    }
    | requirement
      ∈ resolve(sample).qualification_requirements[] }

ReferenceSampleCoverageSetV1(M) =
  { {
      sample_project_manifest_ref: sample,
      target_profile_ref: target,
      reference_dimension:
        if resolve(sample).sample_role=reference_2d
          then two_d
        else if resolve(sample).sample_role=reference_3d
          then three_d
        else reject,
      reference_requirement_ref:
        exact unique binding.reference_requirement_ref
    }
    | sample ∈ ReferenceReleaseSampleSetV1(M),
      target
        ∈ resolve(sample).supported_target_profile_refs[],
      binding
        ∈ resolve(
            resolve(M.distribution_coverage_projection_ref)
              .release_requirement_projection_ref)
          .required_target_dimension_bindings[],
      binding.target_profile_ref=target,
      binding.reference_dimension=
        (if resolve(sample).sample_role=reference_2d
           then two_d else three_d) }

ReferenceSampleQualificationRequirementSetV1(M) =
  { {
      sample_project_manifest_ref:
        coverage.sample_project_manifest_ref,
      target_profile_ref: coverage.target_profile_ref,
      reference_dimension: coverage.reference_dimension,
      reference_requirement_ref:
        coverage.reference_requirement_ref,
      qualification_scenario_ref:
        requirement.qualification_scenario_ref,
      expected_result_branch: success,
      operation_family_kind: project_launch,
      operation_ref: requirement.operation_ref,
      execution_branch: launch
    }
    | coverage ∈ ReferenceSampleCoverageSetV1(M),
      requirement
        ∈ resolve(coverage.sample_project_manifest_ref)
          .qualification_requirements[],
      requirement.target_applicability =
        {kind=target_profile,
         target_profile_ref=coverage.target_profile_ref} }

WorkflowSampleQualificationRequirementSetV1(M) =
  canonical set union(
    SampleQualificationRequirementSetV1(sample)
    for sample ∈ WorkflowReleaseSampleSetV1(M))
```

`canonical set union`はexact `SampleProjectManifestRefV1`のcanonical bytesでdeduplicateする。同じexact Refが複数carrierに現れる場合だけ一件として数え、同名、同role、同Artifact hash、同scenario名または同Project revisionの別Refをcollapseしない。三carrier以外のpath、Website、SDK archive内探索、表示CatalogまたはDocument linkからSample membershipを補完しない。`ReferenceSampleCoverageSetV1(M)`が参照するRelease Requirement Projectionは`M.distribution_coverage_projection_ref`が解決する同Refである。各`{Target, dimension}`へexact一行だけ解決できなければ、zero／multipleのどちらもProjection生成を拒否する。

carrier上限から`|ReleaseSampleSetV1(M)| <= 256 + 64 + 256 = 576`、各SampleのTarget上限から`|ReferenceSampleCoverageSetV1(M)| <= 576 × 64 = 36864`、各Sampleのrequirement上限からflattenしたQualification tuple上限を`576 × 16384 = 9437184`とする。`sample_reference_coverage[0..36864]`はReference Sample×supported Targetごとのflat row、`workflow_sample_qualification_bindings[0..576]`はWorkflow Sampleごとのouter row、`reference_project_qualification_bindings[0..9437184]`はrequired Reference tupleのflat rowであり、実数が少ない場合にdummy、duplicateまたはcarrier外Sampleで下限／上限を埋めない。

`ProductDistributionSubjectV1(kind=sample_project)`の`sample_project_ref` projectionは`ReleaseSampleSetV1(M)`とset equalityである。`sample_reference_coverage[]`のfull `{Sample, Target, dimension, reference requirement}` identity projectionは`ReferenceSampleCoverageSetV1(M)`、§9の`reference_project_qualification_bindings[]`のfull identity projectionは`ReferenceSampleQualificationRequirementSetV1(M)`とset equalityにする。両者のdistinct Sample projectionはそれぞれ`ReferenceReleaseSampleSetV1(M)`とset equalityである。§9の`workflow_sample_qualification_bindings[]`から得るdistinct Sample ref集合は`WorkflowReleaseSampleSetV1(M)`とset equality、そのfull identity projectionは`WorkflowSampleQualificationRequirementSetV1(M)`とset equalityにする。Release Requirement Projectionのrequired Target×dimension集合がemptyならReference集合、Reference Sample Subject、Reference CoverageおよびReference Qualification bindingをすべてexact emptyとし、SDKまたはDocumentation carrierへReference Sampleを隠さない。Workflow集合はReference集合のempty条件から導出せず、三carrierのunionにworkflow Sampleが0件ならWorkflow requirement／bindingだけをexact emptyにする。

非Reference Sampleはすべて`sample_role=workflow`かつ`reference_dimension=none`であり、Reference Sampleは`reference_2d→two_d | reference_3d→three_d`のclosed matrixへ従う。各Sampleの`qualification_requirements[]`にある`kind=target_profile` rowのdistinct Target projectionは`supported_target_profile_refs[]`とset equalityである。Reference Sampleの全rowは`kind=target_profile`、`expected_result_branch=success`、`operation_family_kind=project_launch`かつ`execution_branch=launch`で、`operation_ref`は同Release Required Operation Universeの`project_launch` familyにあるexactly one Operationへ解決する。各Sampleのrequirement Target projection、supported Target projectionおよび同じSampleのReference Coverage Target projectionをset equalityにする。各`{Sample, Target}`へexactly one Coverage rowを持ち、そのSampleの同Targetにある全scenario requirementをReference Qualificationへexactly once展開する。同Targetを別Sampleだけでcoverする、Sampleの一Targetだけを落としてdistinct Sample集合を維持する、同じ`{Sample, Target}`をduplicate Coverageで展開することを拒否する。

Workflow Sampleの`execution_branch`と`operation_family_kind`は`authoring→authoring_author | authoring_preview | authoring_validate | authoring_commit`、`build→project_build`、`test→project_test`、`package→project_package`、`install→product_install`、`launch→project_launch`のclosed matrixへ従い、`operation_ref`はRequired Operation Universeの同family exact rowへ解決する。`authoring | build | test`はtarget-specificまたは`target_independent`を許すが、`package | install | launch`は必ず`kind=target_profile`とする。`kind=target_independent`の`target_profile_ref`補完、Target名／Scenario名／nullからのapplicability推測、同じScenarioの別expected branch／family／Operationへの置換、workflow／Reference roleとdimensionの交差、Coverage外Reference、orphan Sample Subject、Subject外release Sample、direct Manifestを空にしたtransitive Reference除外を拒否する。

各`SampleProjectManifestV1`の`expected_project_revision_ref`は`source_repository_snapshot_ref.project_revision_ref`とbyte equalityである。`source_closure_ref`はSnapshotが解決するexact `ProjectSourceClosureRefV1`、`source_project_artifact_ref`はそのClosureのexact `canonical_transport_artifact_ref`とbyte equalityでなければならず、Artifact hash、path、名前、provider revisionまたはProject revision単独からmembershipを推論しない。§9のowner-typed Qualification Receipt subjectはSample ref、Snapshotの`{project_ref, project_revision_ref, source_closure_hash, snapshot_content_hash}`全Field、Source closure ref／content hash、canonical transport artifactおよびsource entry membershipを同時にread-backし、Snapshotへ属さないArtifact、missing／extra／modified Source entry、同revision別closureまたは同revision別snapshot bytesを拒否する。これはrelease集合とprovenanceのtarget contractであり、Sample、Snapshot、Source Closure、Artifact、Schema、ValidatorまたはReceiptがmaterializeしたことを意味しない。

`ReleaseProjectTestRequirementSetV1(M)` Named Algorithmは、`M.project_template_manifest_refs[]`と`ReleaseSampleSetV1(M)`のtagged disjoint unionをProject subject集合とする。各Templateは`ProjectTemplateManifestV1.project_test_catalog_ref`と`supported_target_profile_refs[]`、各Sampleは`SampleProjectManifestV1.project_test_catalog_ref`と`supported_target_profile_refs[]`をDeveloper Testing Ownerの`ProjectTestRequirementSetV1(catalog, targets)`へ入力し、解決したRequirement Set refを同Project subject rowへ束縛する。outer Project subject上限は`256 + 576 = 832`で、full subject projectionをこのexact unionとset equalityにする。同じCatalogを共有する別Template／Sampleをcollapseせず、Manifest direct Sampleだけに縮約せず、Catalog、Target、Suite、CaseまたはParameterをcaller入力で追加／削除しない。Requirement SetはRelease Manifestを逆参照せず、release-specific外側recordだけがManifest、Project subject、Catalog、Requirement Setを一方向に束縛する。

Documentation graphはEntryからLinkを逆参照しない。Bundleがexact Entry ref集合とexact Link ref集合を一方向に束縛する。`internal_entry`の`entry_key`はRefではなくBundle内node locatorであり、同じBundleの`entry_refs[]`に同じID／versionかつ`expected_entry_content_hash`がbyte equalityのnodeがexactly one件必要である。source keyも同様にexactly one件へ解決し、self edgeは許してもhash参照を作らない。`link_graph_content_hash`は`MIRAKAN_DOCUMENTATION_LINK_GRAPH_V1`、canonical Entry ref集合、canonical Link ref集合、各Linkが解決したsource／destination node keyのclosed bytesから計算し、Entry hashまたはLink hashからBundle hashを逆参照しない。このDAGによりEntry↔LinkまたはEntry A↔Entry Bのcontent-hash cycleを作らず、ID／versionだけのstale destinationも受理しない。

External URLのredirect先、HTTP status、resolved content hashはQualification時のEvidenceであり、URL文字列だけを有効性証明にしない。`normalized_url_artifact_ref`は承認済みnormalized URL bytesをexactに指し、live WebsiteをDocumentation authorityにしない。

Tutorial scenarioは開始Project ref、typed action列、expected Project revision／artifactを持ち、自由shellまたはUI座標を正規stepにしない。Documentation Bundleのentry、snippet、tutorial、Sample集合はpublic contract setのreachable memberをcoverageし、orphan ref、duplicate、別contract setを拒否する。

broken internal／external link、public signatureと異なるsnippet、unrunnable sample、別release向けtutorial、missing locale fallback declarationをProduct Release Gateで失敗させる。Documentation bundleはrelease artifactであり、Websiteのcurrent pageまたはChat回答を正本にしない。

<a id="product-cross-surface-parity"></a>

## 6. Product cross-surface parity

```text
Editor GUI / CLI / Headless / Native SDK / External IDE / MCP
AI automation (Editor built-in / first-party Host / external Host)
  -> versioned ProductSurfaceClientProfile
  -> same typed request
  -> same registered semantic Operation
  -> same authorization and validation
  -> same Authoring Command Gateway
  -> atomic ProjectRevision or no current mutation
  -> same typed Receipt／diagnostic
```

Surface adapterは入力をtyped requestへ投影し、Operation resultを各presentationへ投影するだけである。次を禁止する。

- GUIだけのhidden defaultまたはsilent repair
- CLIだけのfilesystem mutation
- headlessだけのpartial success
- surface別のauthorization、validation、idempotency、hash規則
- Widget label、command text、exit codeをsemantic Operation identityにすること

```text
ProductSurfaceClientProfileV1
  surface_client_profile_id: StableId
  surface_client_profile_version: positive u32
  product_surface_kind: ProductSurfaceKindV1
  adapter_artifact_ref: exact ArtifactRefV1
  adapter_version_identity: non-empty normalized ASCII
  applicable_operation_family_kinds[1..64]:
    sorted unique operation_family_kind
  client_binding:
    {
      kind=canonical_surface_adapter,
      transport_kind:
        editor_projection | cli | headless | native_sdk
        | external_ide | mcp,
      ai_surface_kind: null,
      ai_agent_host_profile_ref: null
    }
    | {
        kind=ai_agent_host_adapter,
        transport_kind:
          editor_projection | cli | native_sdk | mcp,
        ai_surface_kind: AiAgentHostSurfaceKindV1,
        ai_agent_host_profile_ref:
          exact AiAgentHostConformanceProfileRefV1
      }
  surface_policy_content_hash: SHA-256
  surface_client_profile_content_hash: SHA-256

ProductCrossSurfaceConformanceProfileV1
  cross_surface_profile_id: StableId
  cross_surface_profile_version: positive u32
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  active_product_definition_ref: exact ActiveProductDefinitionRefV1
  claim_scope_ref: exact ProductClaimScopeRefV1
  required_surface_client_profile_refs[1..256]:
    sorted unique exact ProductSurfaceClientProfileRefV1
  cross_surface_profile_content_hash: SHA-256

ProductSurfaceParityReceiptV1
  parity_receipt_id: StableId
  parity_receipt_version: positive u32
  cross_surface_conformance_profile_ref:
    exact ProductCrossSurfaceConformanceProfileRefV1
  surface_client_profile_ref:
    exact ProductSurfaceClientProfileRefV1
  required_operation_journey_projection_ref:
    exact RequiredProductOperationJourneyProjectionRefV1
  claim_scope_ref: exact ProductClaimScopeRefV1
  requirement_ref: exact McdContractRefV1(kind=requirement)
  semantic_equivalence_group_id: StableId
  operation_family_kind: operation_family_kind
  operation_ref: exact McdContractRefV1(kind=operation)
  surface_kind: ProductSurfaceKindV1
  host_scope:
    {kind=host_independent}
    | {
        kind=host_profile,
        host_profile_ref:
          exact TargetProfileRefV1(
            profile_kind=build_host | editor_host)
      }
  target_scope:
    {kind=target_independent}
    | {
        kind=target_profile,
        target_profile_ref:
          exact TargetProfileRefV1(profile_kind=runtime_target)
      }
  locale_scope:
    {kind=not_applicable}
    | {kind=locale_independent}
    | {
        kind=locale_profile,
        locale_profile_ref: exact LocaleProfileRefV1
      }
  reference_dimension_scope:
    {kind=not_applicable}
    | {kind=dimension_independent}
    | {
        kind=reference_dimension,
        reference_dimension: two_d | three_d
      }
  qualification_scenario_ref: exact QualificationScenarioRefV1
  expected_result_branch:
    success | expected_policy_rejection | domain_failure_recovery
  evidence_class_ref: exact EvidenceClassRefV1
  request_content_hash: SHA-256
  operation_authorization_binding_ref:
    exact OperationAuthorizationAuditBindingRefV1
  prepared_candidate_ref: null | exact PreparedCandidateRefV1
  before_project_revision_ref: null | exact ProjectRevisionRefV1
  operation_receipt_ref: null | exact OperationReceiptRefV1
  owner_typed_journey_evidence_ref: exact EvidenceRefV1
  observed_result_branch:
    success | expected_policy_rejection | domain_failure_recovery
  semantic_result_content_hash: SHA-256
  resulting_project_revision_ref: null | exact ProjectRevisionRefV1
  diagnostic_set_content_hash: SHA-256
  evidence_refs[1..256]:
    sorted unique exact EvidenceRefV1
  parity_receipt_content_hash: SHA-256

DocumentationQualificationReceiptV1
  documentation_receipt_id: StableId
  documentation_receipt_version: positive u32
  documentation_bundle_ref: exact DocumentationBundleManifestRefV1
  documentation_requirement_set_ref:
    exact DocumentationQualificationRequirementSetRefV1
  candidate_ref: exact PreparedCandidateRefV1
  observed_public_contract_set_ref: exact PublicContractSetRefV1
  entry_results[1..65535]:
    sorted unique {
      documentation_entry_ref: exact DocumentationEntryRefV1,
      result: passed,
      evidence_refs[1..256]:
        sorted unique exact EvidenceRefV1
    }
  link_results[0..262144]:
    sorted unique {
      documentation_link_ref: exact DocumentationLinkRefV1,
      result: passed,
      evidence_refs[1..256]:
        sorted unique exact EvidenceRefV1
    }
  snippet_target_results[1..262144]:
    sorted unique {
      snippet_fixture_ref: exact DocumentationSnippetFixtureRefV1,
      target_profile_ref: exact TargetProfileRefV1,
      observed_result:
        exact resolve(snippet_fixture_ref).expected_result,
      evidence_refs[1..256]:
        sorted unique exact EvidenceRefV1
    }
  tutorial_results[1..262144]:
    sorted unique {
      documentation_entry_ref: exact DocumentationEntryRefV1,
      entry_locale_profile_ref: exact LocaleProfileRefV1,
      entry_source_artifact_ref: exact ArtifactRefV1,
      entry_rendered_artifact_ref: exact ArtifactRefV1,
      qualification_scenario_ref: exact QualificationScenarioRefV1,
      target_profile_ref: null | exact TargetProfileRefV1,
      observed_result_branch:
        success | expected_policy_rejection | domain_failure_recovery,
      evidence_refs[1..256]:
        sorted unique exact EvidenceRefV1
    }
  documentation_receipt_content_hash: SHA-256
```

`ProductSurfaceClientProfileV1`のclosed matrixでは、`canonical_surface_adapter`のProduct surfaceとtransportを`editor_gui→editor_projection | cli→cli | headless→headless | native_sdk→native_sdk | external_ide→external_ide | mcp→mcp`へexact一致させ、`ai_automation`を禁止する。`ai_agent_host_adapter`はProduct surfaceを`ai_automation`に固定し、AI surfaceをAI Production Orchestrationのexact Profileへ一致させる。`editor_builtin→editor_projection`、`first_party_agent_host→cli | native_sdk`、`external_agent_host→mcp | cli | native_sdk`だけを許可する。Vendor名はenumにせず、製品、adapter、version、transport、surface policyを`AiAgentHostConformanceProfileRefV1`で固定する。generic MCP Conformanceは`mcp` Product surface、個別Agent経由のMCP Conformanceは`ai_automation`として別Receiptを要求し、相互代用しない。

Cross-surface Profileは同じEngine Release、Active Product Definition、Claim Scopeへ閉じ、ReleaseがProduct上で`supported`と表示する全client／adapter versionをexactly once列挙する。各Required Journey rowについて、同じsurfaceとoperation familyを宣言するClient Profile集合を決定論的に選ぶ。集合がemptyならProfile自体を拒否し、各`{required journey row,client profile}` pairへexactly one Parity Receiptを要求する。全rowのapplicable Client Profile件数をchecked sumしたpair総数は65,535以下でなければならず、overflow、上限超過、truncate、profile間collapseまたは一Receiptによる複数pair代用ではProfile／Acceptanceを生成しない。Profileに列挙しないAgent、adapter、IDE extensionまたはversionをsupportedと表示せず、同vendor別version、同transport別Agent、generic MCP Fixtureを一件へcollapseしない。

各Parity ReceiptはRequired Operation Journey Projectionのexactly one required rowとCross-surface Profileのexactly one Client Profileへ、Requirement、semantic equivalence groupを含む全identity Fieldで解決し、forbidden rowまたはProjection外tupleへ解決してはならない。Receiptの`surface_kind`はClient Profileの`product_surface_kind`とbyte equalityである。`observed_result_branch`はrequired rowの`expected_result_branch`と一致し、`owner_typed_journey_evidence_ref`は同じRequirement、semantic group、Evidence class、family、Operation、surface、Client Profile、Host scope／Target scope／locale scope／Reference dimension scope、scenario、branchを持つfresh non-revoked owner-typed signed Evidenceへ解決する。state-changing successはexact Operation Receiptとafter Project revision、read-only successはmutation Receiptなし、expected rejection／failure recoveryは規定されたtyped diagnosticとbefore Project不変を要求する。

同じ`{requirement_ref,semantic_equivalence_group_id}`にはProjectionがrequiredとするsurface rowとapplicable Client ProfileのCartesian pairがexactly one件ずつ存在し、request hash、authorization、Candidate、before Project、Operation semantic result、resulting Project、diagnostic setをbranchの意味に従って同値にする。全7 Product surfaceを無条件要求せず、forbidden surfaceを欠落Receiptとして扱わない。反対にGUI／CLI／headless成功をNative SDK／external IDE／MCP／AI automationへ、SDK FixtureをMCPへ、MCP transport FixtureをCodex／Claude等の個別Agent Profileへ、あるAgent versionを別versionへ、別Requirement、別semantic group、別family、別Operation、別Targetまたは別scenarioの成功をrequired pairへ代用しない。presentation bytes、Model prose、Tool-call serializationまたはlatencyの一致は要求しない。

Documentation ReceiptのRequirement Setは同じBundleを参照し、Entry、Link、Snippet Target、Tutorialの各result full identity projectionをRequirement Setの対応配列と各々set equalityにする。Snippetのobserved resultはFixture expected result、Tutorialのobserved branchはrequired branchとbyte equalityで、全resultは同じCandidate／public contract setのfresh non-revoked owner Evidenceへ解決する。Bundle-level `passed`一件、Target／Scenarioだけのaggregate、Entry／Link未検査、Snippet S1によるS2充足、4,096件での打切り、失敗、取消済みEvidence、別Candidateまたは別public contract setをAcceptanceへ入れない。

## 7. Product update

```text
ProductInstallationChannelV1
  installation_channel_id: StableId
  installation_channel_version: positive u32
  kind: offline_installer | managed_layout | platform_store
  installation_scope_bindings[1..128]:
    sorted unique
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
  platform_owner_document_id: ArchitectureDocumentId
  distribution_contract_artifact_ref: exact ArtifactRefV1
  installation_channel_content_hash: SHA-256

ProductPublicationRecoveryPolicyV1
  recovery_policy_id: StableId
  recovery_policy_version: positive u32
  last_known_good_required: true
  project_publication: single_atomic_promotion
  prepublication_failure: discard_candidate
  postpublication_failure:
    explicit_release_rollback | forward_repair_required
  user_data_policy: preserve_and_version_validate
  irreversible_external_action_policy:
    retain_evidence_and_require_compensating_decision
  recovery_policy_content_hash: SHA-256

ProductUpdatePlanV1
  update_plan_id: StableId
  update_plan_version: positive u32
  source_engine_release_binding_ref: exact EngineReleaseBindingRefV1
  destination_engine_release_binding_ref: exact EngineReleaseBindingRefV1
  source_project_revision_ref: exact ProjectRevisionRefV1
  candidate_ref: exact PreparedCandidateRefV1
  compatibility_assessment_ref: exact CompatibilityAssessmentRefV1
  migration_plan_ref: exact MigrationPlanRefV1
  installation_channel_refs[1..64]:
    sorted unique exact ProductInstallationChannelRefV1
  publication_recovery_policy_ref: exact ProductPublicationRecoveryPolicyRefV1
  target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1(profile_kind=runtime_target)
  required_qualification_scenario_refs[1..4096]:
    sorted unique exact QualificationScenarioRefV1
  support_window_ref: exact ProductSupportWindowRefV1
  update_plan_content_hash: SHA-256

ProductInstalledStateV1
  installed_state_id: StableId
  installed_state_version: 1
  host_profile_ref:
    exact TargetProfileRefV1(profile_kind=build_host | editor_host)
  installation_channel_ref: exact ProductInstallationChannelRefV1
  installer_layout_manifest_artifact_ref: exact ArtifactRefV1
  state_kind: absent | installed | partial | corrupt
  installed_engine_release_binding_ref:
    null | exact EngineReleaseBindingRefV1
  installed_distribution_subject_refs[0..8192]:
    sorted unique exact ProductDistributionSubjectRefV1
  installed_artifact_refs[0..65535]:
    sorted unique exact ArtifactRefV1
  product_owned_data_classes[0..10]:
    sorted unique
      binaries | tool_cache | product_configuration | license_state
      | telemetry_queue | crash_data | support_data
      | update_staging | signature_metadata | installation_log
  retained_user_project_data_classes[0..4]:
    sorted unique
      project_source | project_derived_data | user_preferences
      | user_exported_artifacts
  installed_state_content_hash: SHA-256

ProductLifecycleTransitionRequirementBindingV1
  host_profile_ref:
    exact TargetProfileRefV1(profile_kind=build_host | editor_host)
  installation_channel_ref: exact ProductInstallationChannelRefV1
  transition:
    {
      kind=install,
      destination_engine_release_binding_ref:
        exact EngineReleaseBindingRefV1
    }
    | {
        kind=update,
        update_plan_ref: exact ProductUpdatePlanRefV1,
        source_engine_release_binding_ref:
          exact EngineReleaseBindingRefV1,
        destination_engine_release_binding_ref:
          exact EngineReleaseBindingRefV1
      }
    | {
        kind=repair,
        engine_release_binding_ref: exact EngineReleaseBindingRefV1
      }
    | {
        kind=uninstall,
        source_engine_release_binding_ref:
          exact EngineReleaseBindingRefV1
      }
  before_installed_state_ref: exact ProductInstalledStateRefV1
  expected_after_installed_state_ref: exact ProductInstalledStateRefV1
  qualification_scenario_ref: exact QualificationScenarioRefV1
  expected_result_branch:
    success | expected_policy_rejection | domain_failure_recovery
  product_owned_data_disposition:
    install_exact | update_exact | repair_exact | remove_all
  retained_user_project_data_disposition: preserve_exact

ProductLifecycleTransitionRequirementSetV1
  transition_requirement_set_id: StableId
  transition_requirement_set_version: 1
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  release_content_manifest_ref: exact ProductReleaseContentManifestRefV1
  update_plan_refs[0..64]:
    sorted unique exact ProductUpdatePlanRefV1
  required_transition_bindings[1..262144]:
    sorted unique ProductLifecycleTransitionRequirementBindingV1
  projection_algorithm_id:
    product_lifecycle_transition_requirement_set
  projection_algorithm_version: 1
  projection_algorithm_content_hash: SHA-256
  transition_requirement_set_content_hash: SHA-256
```

Updateはlive Projectまたはinstalled releaseをin-place mutationしない。source release／Projectから独立Candidateを作り、次のexact closureを得た後だけCompatibility Ownerがmigration／rollback eligibilityを認め、Product Lifecycleが`ProductPublicationRecoveryPolicyV1`に従う一回のpublicationを行う。Compatibility Ownerはdata／public-contract復元後の意味とlast-known-good eligibility、本書はpublication境界、不可逆外部action、失敗後の製品横断状態だけを所有し、互いのSchemaを複写しない。

1. source inventoryとconsumer inventory
2. compatibility assessmentとversioned migration
3. Project validation
4. Target別cook／build／package
5. install／launch／offline play
6. Save／Load／Replay、必要なrecook／rebuild
7. diagnosis／Support Bundle
8. SBOM／NOTICE／license presentation
9. same-candidate qualificationとrelease acceptance

旧release／旧Projectをlast-known-goodとして維持する。途中成功をcurrentにせず、別Candidate、別Target、別revisionのReceiptを合成しない。不可逆なStore提出、通知または外部公開の失敗をlocal rollback済みと表現しない。

`repair`はcurrent release bindingに対するmanifest／artifact verificationと同一bytesの復元だけを意味する。別version、latest、別channelへのupdateをrepairとして実行しない。

`ProductInstalledStateV1`はReceiptを含まない製品全体の状態表現である。`absent`はrelease null、installed Subject／artifact／product-owned data exact empty、`installed`はrelease non-nullで同Release、Host、channel、layoutに必要なDistribution Subject／artifactとproduct-owned dataの完全closure、`partial | corrupt`は観測された不完全closureを明示する。全branchでretained User／Project dataはproduct-owned dataとdisjointである。uninstallはproduct-owned binaries、tool cache、configuration、license state、queue、crash／support data、staging、signature metadata、installation logのpolicy指定集合だけを除去し、Project Source、Project derived data、User preference、User exportを暗黙削除しない。Product Target Packageのdevice uninstallはPlatform／Runtime Package Ownerの別transitionであり、Host上のEngine／Editor製品uninstallを代用しない。

Lifecycle Transition Requirement SetのNamed Algorithm v1は、Engine Release、Release Content Manifestの全Host、各`ProductInstallationChannelV1.installation_scope_bindings[]`、Required Operation Journeyの`product_install | product_update | product_repair | product_uninstall` scenario、`update_plan_refs[]`を入力にfull transition keyを生成する。Host `h`とChannel `c`のpairは、`c.installation_scope_bindings[]`にexact `{kind=host_profile,host_profile_ref=h}`が存在する場合だけHost lifecycleへ展開する。各required Hostは少なくとも一つのHost-applicable Channelを持たなければならず、runtime-target-only ChannelはHost transition集合から除外してPlatform／Runtime Package Ownerのdevice transitionへ残す。installはbefore `absent`からdestination releaseの`installed`、repairは同一releaseの`installed`から同一bytesの`installed`、uninstallはsource releaseの`installed`から`absent`を要求する。update successはexact `ProductUpdatePlanV1`を必須とし、そのsource／destination release、Host-applicable Channel、scenarioをtransition branchとbyte／set equalityにする。Planの`installation_channel_refs[]`とのintersectionも同じHost-applicable集合だけに限定し、runtime-target bindingまたは別Host bindingをupdate rowへ流用しない。update planがないsource releaseはsuccess update rowを作らず、unsupported sourceのtyped policy rejection rowを独立に持つ。別release、別Host、別channel、別layoutまたは別Planを同actionへ付け替えない。

各actionはsuccessに加え、Required Journeyが要求するexpected policy rejection、cancel、fault、recovery scenarioを独立rowへ投影する。success after stateはexpected exact state、rejectionはbefore state不変、domain failure／recoveryは`ProductPublicationRecoveryPolicyV1`が定めるlast-known-good `installed`または明示`partial | corrupt`＋repair pathへ一致させる。before／after、remove／preserve集合、failure／cancel／recovery Evidenceを一つのtransition resultからread-backし、flat update／rollback／repair receipt poolを作らない。required transitionは262,144件以下、`Host × host-applicable channel × action/scenario`のchecked expansion overflowまたは上限超過はRequirement Set生成失敗である。

## 8. Support windowとNOTICE presentation

```text
ProductSupportWindowV1
  support_window_id: StableId
  support_window_version: positive u32
  supported_target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1
  support_channel_refs[1..64]:
    sorted unique exact ProductSupportChannelRefV1
  support_start_policy: at_release_publication
  support_end:
    {kind=explicit_date, date: RFC 3339 full-date}
    | {kind=ongoing_until_superseding_decision}
  security_update_support: required
  known_limitation_document_ref: exact DocumentationEntryRefV1
  required_qualification_scenario_bindings[1..4096]:
    sorted unique {
      qualification_scenario_ref: exact QualificationScenarioRefV1,
      target_applicability:
        {kind=all_supported_targets}
        | {kind=target_independent}
        | {
            kind=exact_set,
            target_profile_refs[1..64]:
              sorted unique exact TargetProfileRefV1(
                profile_kind=runtime_target)
          },
      expected_result_branches[1..3]:
        sorted unique
          success | expected_policy_rejection | domain_failure_recovery
    }
  end_notification:
    minimum_notice_days: positive u32
    channel_refs[1..64]:
      sorted unique exact ProductSupportChannelRefV1
  support_window_content_hash: SHA-256

ProductSupportChannelV1
  channel_id: StableId
  channel_version: positive u32
  kind: offline_documented_process | web_form | email
  endpoint_artifact_ref: exact ArtifactRefV1
  locale_profile_refs[1..64]:
    sorted unique exact LocaleProfileRefV1
  authentication: none | user_account_required
  accepted_attachment_kind:
    none | redacted_support_bundle
  channel_content_hash: SHA-256

SupportQualificationRequirementBindingV1
  support_channel_ref: exact ProductSupportChannelRefV1
  locale_profile_ref: exact LocaleProfileRefV1
  target_scope:
    {kind=target_independent}
    | {
        kind=target_profile,
        target_profile_ref:
          exact TargetProfileRefV1(profile_kind=runtime_target)
      }
  authentication: none | user_account_required
  accepted_attachment_kind: none | redacted_support_bundle
  qualification_scenario_ref: exact QualificationScenarioRefV1
  expected_result_branch:
    success | expected_policy_rejection | domain_failure_recovery

SupportQualificationRequirementSetV1
  support_qualification_requirement_set_id: StableId
  support_qualification_requirement_set_version: 1
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  support_window_ref: exact ProductSupportWindowRefV1
  required_support_qualification_bindings[1..262144]:
    sorted unique SupportQualificationRequirementBindingV1
  projection_algorithm_id: support_qualification_requirement_set
  projection_algorithm_version: 1
  projection_algorithm_content_hash: SHA-256
  support_qualification_requirement_set_content_hash: SHA-256

ProductNoticePresentationV1
  presentation_id: StableId
  presentation_version: positive u32
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  distribution_subject:
    exact ProductDistributionSubjectRefV1
  control_applicability_ref:
    exact ProductDistributionControlApplicabilityRefV1
  sbom_ref: null | exact SbomRefV1
  notice_bundle_ref: null | exact ThirdPartyNoticeBundleRefV1
  product_license_grant_ref: null | exact ProductLicenseGrantRefV1
  presentation_entry_refs[0..4096]:
    sorted unique exact ProductNoticePresentationEntryRefV1
  locale_profile_refs[1..64]:
    sorted unique exact LocaleProfileRefV1
  presentation_content_hash: SHA-256

ProductNoticePresentationEntryV1
  presentation_entry_id: StableId
  presentation_entry_version: positive u32
  notice_subject_ref:
    {kind=dependency, dependency_lock_entry_ref: exact DependencyLockEntryRefV1}
    | {
        kind=redistributed_artifact,
        redistributed_artifact_ref: exact RedistributedArtifactRefV1
      }
  license_ref: exact LicenseArtifactRefV1
  notice_text_artifact_ref: exact ArtifactRefV1
  locale_profile_ref: exact LocaleProfileRefV1
  presentation_location:
    editor_about | installed_game_legal | distribution_bundle_document
  package_entry_ref: null | exact PackageEntryRefV1
  presentation_entry_content_hash: SHA-256

ProductSupportWindowClosureV1
  support_window_closure_id: StableId
  support_window_closure_version: positive u32
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  closed_support_window_ref: exact ProductSupportWindowRefV1
  expected_previous_closure_ref: null | exact ProductSupportWindowClosureRefV1
  closure_basis:
    {
      kind: scheduled_explicit_date,
      release_decision_evidence_ref: exact EvidenceRefV1
    }
    | {
        kind: superseding_product_decision,
        signed_product_decision_evidence_ref: exact EvidenceRefV1
      }
  effective_end_at: RFC 3339 timestamp
  notification_receipt_refs[1..4096]:
    sorted unique exact NotificationReceiptRefV1
  notified_channel_refs[1..64]:
    sorted unique exact ProductSupportChannelRefV1
  support_window_closure_content_hash: SHA-256
```

`support_start_policy=at_release_publication`は時刻やRelease approvalを直接表すFieldではない。実効開始は[Product Publication／Completion](product-publication-completion.md)のexact `ProductSupportPublicationStartBindingV1`だけが所有し、同じEngine Releaseのcurrent `published` Publication、Publication State、`effective_publication_at`へbyte equalityで束縛する。Decision approval、signing、upload、submission、Store approval、local buildまたはcaller時刻からsupport開始を推測しない。

Toolchain Ownerが生成したSBOM／third-party notice sourceと、`distribution_subject`が解決する配布artifact entry集合をexact set equalityで照合する。Subjectは同じEngine ReleaseのDistribution Coverage Projectionに列挙されたexact一件で、ApplicabilityもそのSubjectへbyte equalityでなければならない。`sbom`、`third_party_notice`、`first_party_license_presentation`の各Field／entryはrequired controlでexactly oneの有効なbinding、forbidden controlでnull／emptyにする。Product Lifecycleはthird-party license判断またはSBOM生成を所有せず、Userがsource distribution、Editor、installed Gameまたは配布bundleのdocumented locationからrequired noticeへ到達できることを判定する。first-party licenseは§3.1のexact grantをrequired subjectの同じsurfaceから提示する。別buildのSBOM、Applicability違反、空NOTICE、欠落locale、到達不能UIを成功にしない。

Engine release bindingがSupport Window base refを束縛することでrelease、Target、support channel、security update policy、end dateまたは明示的ongoing、known limitation disclosureを閉じる。`supported_target_profile_refs[]`は同じEngine Release、Release Content ManifestおよびRelease Requirement Projectionのruntime Target集合とset equalityで、Target omission、extra、duplicate、Host substitutionを拒否する。`end_notification.channel_refs[]`は`support_channel_refs[]`のnon-empty subsetであり、ID、version、hashをbyte equalityにする。

Support Qualification Requirement SetのNamed Algorithm v1は、Engine Releaseが束縛するexact Support Window、各Support Channelの全locale、authentication、attachment kind、Windowの全qualification scenario／expected branchをchecked Cartesian expansionする。`all_supported_targets`はWindowの全runtime Targetへscalar展開し、`exact_set`はWindow Target集合のnon-empty subsetだけを各Targetへ展開する。`target_independent`はScenarioのcanonical `QualificationScenarioV1`がTarget-independentと明示する場合だけ一件の独立branchを生成し、Target-specific Scenarioの全Target充足へ流用しない。各result keyのchannel、locale、Target scope、auth、attachment、scenario、branchはRequirement Setのexact rowとbyte equalityで、success、policy rejection、redaction／attachment failure recoveryを別branchとして保持する。channel existence、generic support Journey、Support Bundle export一件、locale aggregateまたは一Targetの結果をQualificationへ数えない。`channel × locale × Target scope × scenario × branch`は262,144件以下で、overflow、truncate、既定Target／locale／auth／attachment補完をRequirement Set生成失敗以外へ変換しない。

Support終了時は別immutable `ProductSupportWindowClosureV1`を作り、同じEngine Releaseが束縛するexact Support Window、expected previous Closure、closure basis、effective time、全required channelのNotification Receiptを閉じる。`explicit_date` windowは`scheduled_explicit_date`だけを許し、`effective_end_at`をbase dateのend-of-day UTCへ正規化して一致させる。`ongoing_until_superseding_decision`は無期限保証ではなく、`superseding_product_decision`と署名済みProduct decision Evidenceを必須にする。`notified_channel_refs[]`はbase windowの`end_notification.channel_refs[]`とset equalityでなければならない。最初のClosureは`expected_previous_closure_ref=null`かつversion 1、訂正は同じclosure IDの直前versionを参照し、self、branch、merge、cycleを拒否する。base recordまたはEngine ReleaseへClosure refを埋め戻さず、一方向DAGを維持する。`latest releaseだけsupport`、Website text、issue tracker labelからsupport状態を推測しない。`SupportBundleV1`のField、redaction、consent、failureはDebugging Ownerだけが所有する。

## 9. Product lifecycle acceptance

```text
ProductLifecycleAcceptanceV1
  acceptance_id: StableId
  acceptance_version: positive u32
  engine_release_binding_ref: exact EngineReleaseBindingRefV1
  candidate_ref: exact PreparedCandidateRefV1
  first_playable_definition_ref: exact FirstPlayableDefinitionRefV1
  first_playable_project_revision_ref: exact ProjectRevisionRefV1
  first_playable_ai_generation_claim_scope_ref:
    exact AiGameGenerationClaimScopeRefV1
  first_playable_game_production_loop_closure_ref:
    exact GameProductionLoopClosureRefV1
  first_playable_manual_journey_evidence_refs[1..256]:
    sorted unique exact EvidenceRefV1
  first_playable_ai_journey_evidence_refs[1..256]:
    sorted unique exact EvidenceRefV1
  host_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1(
      profile_kind=build_host | editor_host)
  target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1(profile_kind=runtime_target)
  locale_profile_refs[1..64]:
    sorted unique exact LocaleProfileRefV1
  distribution_coverage_projection_ref:
    exact ProductDistributionCoverageProjectionRefV1
  required_operation_journey_projection_ref:
    exact RequiredProductOperationJourneyProjectionRefV1
  cross_surface_conformance_profile_ref:
    exact ProductCrossSurfaceConformanceProfileRefV1
  bootstrap_receipt_refs[1..256]:
    sorted unique exact OperationReceiptRefV1
  parity_receipt_refs[1..65535]:
    sorted unique exact ProductSurfaceParityReceiptRefV1
  documentation_requirement_set_ref:
    exact DocumentationQualificationRequirementSetRefV1
  documentation_qualification_receipt_ref:
    exact DocumentationQualificationReceiptRefV1
  sdk_qualification_requirement_set_ref:
    exact SdkQualificationRequirementSetRefV1
  sdk_qualification_results[1..262144]:
    sorted unique {
      requirement_binding: exact SdkQualificationRequirementBindingV1,
      qualification_receipt_ref: exact QualificationReceiptRefV1
    }
  host_distribution_receipt_refs[1..256]:
    sorted unique exact QualificationReceiptRefV1
  release_project_test_requirement_set_ref:
    exact ReleaseProjectTestRequirementSetRefV1
  project_test_qualification_bindings[1..832]:
    sorted unique {
      project_subject:
        {
          kind=project_template,
          project_template_manifest_ref:
            exact ProjectTemplateManifestRefV1
        }
        | {
            kind=sample_project,
            sample_project_manifest_ref:
              exact SampleProjectManifestRefV1
          },
      project_test_catalog_ref: exact ProjectTestCatalogRefV1,
      project_test_requirement_set_ref:
        exact ProjectTestRequirementSetRefV1,
      observed_project_revision_ref: exact ProjectRevisionRefV1,
      target_result_bindings[1..64]:
        sorted unique {
          target_profile_ref: exact TargetProfileRefV1,
          project_test_run_result_refs[1..65536]:
            sorted unique exact ProjectTestRunResultRefV1
        }
    }
  target_package_runtime_entry_bindings[1..16384]:
    sorted unique {
      target_package_ref: exact ProductReleaseTargetPackageRefV1,
      runtime_entry_distribution_package_manifest_ref:
        exact RuntimeEntryDistributionPackageManifestRefV1,
      runtime_entry_package_ref: exact RuntimeEntryPackageRefV1,
      runtime_entry_launch_selection_ref:
        exact RuntimeEntryLaunchSelectionRefV1,
      package_receipt_ref:
        exact OperationReceiptRefV1(
          operation_ref=operation.build.request_package),
      qualification_scenario_receipt_bindings[1..256]:
        sorted unique {
          qualification_scenario_ref:
            exact QualificationScenarioRefV1,
          install_qualification_receipt_ref:
            exact QualificationReceiptRefV1,
          launch_qualification_receipt_ref:
            exact QualificationReceiptRefV1
        }
    }
  reference_project_qualification_bindings[0..9437184]:
    sorted unique {
      reference_requirement_ref:
        exact McdContractRefV1(kind=requirement),
      target_profile_ref: exact TargetProfileRefV1,
      reference_dimension: two_d | three_d,
      target_package_ref: exact ProductReleaseTargetPackageRefV1,
      sample_project_manifest_ref: exact SampleProjectManifestRefV1,
      source_repository_snapshot_ref:
        exact ProjectRepositorySnapshotRefV1,
      source_closure_ref: exact ProjectSourceClosureRefV1,
      source_project_artifact_ref: exact ArtifactRefV1,
      source_project_revision_ref: exact ProjectRevisionRefV1,
      candidate_ref: exact PreparedCandidateRefV1,
      qualification_scenario_ref: exact QualificationScenarioRefV1,
      expected_result_branch: success,
      operation_family_kind: project_launch,
      operation_ref: exact McdContractRefV1(kind=operation),
      execution_branch: launch,
      runtime_entry_distribution_package_manifest_ref:
        exact RuntimeEntryDistributionPackageManifestRefV1,
      scenario_runtime_entry_launch_bindings[1..64]:
        sorted unique {
          runtime_entry_package_ref: exact RuntimeEntryPackageRefV1,
          runtime_entry_launch_selection_ref:
            exact RuntimeEntryLaunchSelectionRefV1
        },
      qualification_receipt_ref: exact QualificationReceiptRefV1
    }
  workflow_sample_qualification_bindings[0..576]:
    sorted unique {
      sample_project_manifest_ref: exact SampleProjectManifestRefV1,
      source_repository_snapshot_ref:
        exact ProjectRepositorySnapshotRefV1,
      source_closure_ref: exact ProjectSourceClosureRefV1,
      source_project_artifact_ref: exact ArtifactRefV1,
      source_project_revision_ref: exact ProjectRevisionRefV1,
      candidate_ref: exact PreparedCandidateRefV1,
      qualification_requirement_bindings[1..16384]:
        sorted unique {
          target_applicability:
            {kind=target_profile,
             target_profile_ref: exact TargetProfileRefV1}
            | {kind=target_independent,
               target_profile_ref: null},
          qualification_scenario_ref: exact QualificationScenarioRefV1,
          expected_result_branch:
            success | expected_policy_rejection
            | domain_failure_recovery,
          operation_family_kind: operation_family_kind,
          operation_ref: exact McdContractRefV1(kind=operation),
          execution_branch:
            authoring | build | test | package | install | launch,
          operation_evidence:
            {
              kind: authoring_build_test,
              owner_request_sha256: SHA-256,
              owner_operation_receipt_ref:
                exact OperationReceiptRefV1
            }
            | {
              kind: package_success,
              owner_request_sha256: SHA-256,
              produced_target_package_ref:
                exact ProductReleaseTargetPackageRefV1,
              produced_distribution_package_manifest_ref:
                exact RuntimeEntryDistributionPackageManifestRefV1,
              produced_runtime_entry_package_refs[1..64]:
                sorted unique exact RuntimeEntryPackageRefV1,
              owner_operation_receipt_ref:
                exact OperationReceiptRefV1(
                  operation_ref=operation.build.request_package)
            }
            | {
              kind: package_non_success,
              owner_request_sha256: SHA-256,
              last_valid_before_target_package_ref:
                null | exact ProductReleaseTargetPackageRefV1,
              last_valid_after_target_package_ref:
                null | exact ProductReleaseTargetPackageRefV1,
              before_state_evidence_ref: exact EvidenceRefV1,
              after_state_evidence_ref: exact EvidenceRefV1,
              owner_operation_receipt_ref:
                exact OperationReceiptRefV1(
                  operation_ref=operation.build.request_package)
            }
            | {
              kind: install,
              owner_request_sha256: SHA-256,
              input_target_package_ref:
                exact ProductReleaseTargetPackageRefV1,
              input_distribution_package_manifest_ref:
                exact RuntimeEntryDistributionPackageManifestRefV1,
              before_state_evidence_ref: exact EvidenceRefV1,
              after_state_evidence_ref: exact EvidenceRefV1,
              owner_operation_receipt_ref:
                exact OperationReceiptRefV1(
                  operation_ref=operation.device.install)
            }
            | {
              kind: launch,
              owner_request_sha256: SHA-256,
              input_target_package_ref:
                exact ProductReleaseTargetPackageRefV1,
              input_distribution_package_manifest_ref:
                exact RuntimeEntryDistributionPackageManifestRefV1,
              before_state_evidence_ref: exact EvidenceRefV1,
              after_state_evidence_ref: exact EvidenceRefV1,
              entry_launch_receipt_bindings[1..64]:
                sorted unique {
                  runtime_entry_package_ref:
                    exact RuntimeEntryPackageRefV1,
                  runtime_entry_launch_selection_ref:
                    exact RuntimeEntryLaunchSelectionRefV1,
                  owner_operation_receipt_ref:
                    exact OperationReceiptRefV1(
                      operation_ref=operation.device.launch)
                }
            },
          qualification_receipt_ref: exact QualificationReceiptRefV1
        }
    }
  build_cook_package_receipt_refs[1..4096]:
    sorted unique exact OperationReceiptRefV1
  install_launch_offline_receipt_refs[1..8388608]:
    sorted unique exact QualificationReceiptRefV1
  lifecycle_transition_requirement_set_ref:
    exact ProductLifecycleTransitionRequirementSetRefV1
  lifecycle_transition_results[1..262144]:
    sorted unique {
      requirement_binding:
        exact ProductLifecycleTransitionRequirementBindingV1,
      observed_before_installed_state_ref:
        exact ProductInstalledStateRefV1,
      observed_after_installed_state_ref:
        exact ProductInstalledStateRefV1,
      observed_result_branch:
        success | expected_policy_rejection | domain_failure_recovery,
      qualification_receipt_ref: exact QualificationReceiptRefV1,
      failure_cancel_recovery_evidence_refs[0..64]:
        sorted unique exact EvidenceRefV1
    }
  support_qualification_requirement_set_ref:
    exact SupportQualificationRequirementSetRefV1
  support_qualification_results[1..262144]:
    sorted unique {
      requirement_binding:
        exact SupportQualificationRequirementBindingV1,
      qualification_receipt_ref: exact QualificationReceiptRefV1
    }
  privacy_acceptance_ref: exact ProductPrivacyAcceptanceRefV1
  pack_lifecycle_acceptance_refs[0..4096]:
    sorted unique exact PackLifecycleAcceptanceRefV1
  notice_presentation_refs[1..4096]:
    sorted unique exact ProductNoticePresentationRefV1
  distribution_control_evidence_bindings[1..65535]:
    sorted unique {
      distribution_subject_ref: exact ProductDistributionSubjectRefV1,
      control_kind: ProductDistributionControlKindV1,
      evidence_binding:
        {kind: evidence, evidence_ref: exact EvidenceRefV1}
        | {
            kind: notice_presentation,
            presentation_ref: exact ProductNoticePresentationRefV1
          }
    }
  acceptance_content_hash: SHA-256
```

First Playable六Fieldは[Product Plan §5.1](product-plan.md#51-first-playable-outcome)のexact `FirstPlayableDefinitionV1`からrequired Project lineage、AI lane、journey、Runtime Entry role、scenario、Requirement、Evidence classを導出する。Loop Closureは同じCandidate／Project revision、`accepted`、`GameUnderstandingClosureV1=ready_to_stage`、Experience Evaluation、Human Gameplay Approval、Iteration `accept_candidate`へ解決し、AI claim scopeはDefinitionが要求するexact lane setとbyte equalityにする。manualとAI journey EvidenceはそれぞれDefinitionのrequired journey projectionとset equalityで全件`fresh`にし、一方を他方へ流用しない。Definitionが所有するinitial exact Host／runtime Target／locale／Input device class、RPG Feature family selection、playable coverageおよびWindows clean-install requirementをRequirement／scenario／Evidence projectionからread-backし、consumer側で集合を再定義または縮小しない。partial Project、manual-only、AI-only、Shooter、3D、別Genre、wrong runtime Target／distribution route、別Candidate、別Project revisionまたは別Runtime EntryのEvidenceを拒否する。

C2のLifecycle入力は[Product Plan §5.3](product-plan.md#53-c2-production-exact-product-boundary)からexact Host `{target.headless.host@1, target.windows.editor@1}`、runtime Target `{target.windows.desktop@1, target.android.mobile@1, target.apple.mobile@1}`、locale `{en-US, ja-JP}`、各runtime Targetの`two_d`／`three_d`、Project C++／Project Shader、三Platform publication requirementをread-backし、本書で再定義または縮小しない。Reference Sample／Target Package／Acceptance bindingのTarget×dimension distinct projectionは三Target×二dimensionのexact六pairとset equalityにし、Windows 2D／3D、Android 2D／3D、Apple 2D／3Dを別Target、別dimension、別Candidateまたは一つのShooter Sampleで代用しない。Project C++／ShaderのTarget qualificationは各三Targetへ個別に束縛し、Windows ReceiptまたはEngine-owned precompiled artifactからAndroid／Apple Project Source supportを推論しない。各required Referenceのoffline completion pathに必要なcontentは対応Target Packageのinstall-time closureへ含め、fast-follow、on-demandまたはmanaged assetだけの取得をlaunch後の必須前提にしない。

C2のWindows Microsoft Store＋MSIX、Google Play＋AAB、Apple App Store＋bundleはCompletionへ投影するexact public routeである。Lifecycle Acceptanceは対応するpackage、signing、submission前qualification、update／withdrawal、Platform Ownerのroute identityを同じCandidateへ束縛するが、internal MSIX、Play test track、TestFlight、uploadまたはreview approvalをactual public publication Receiptとして発行しない。signed Release Decision後のPlatform publication Receipt、public read-back、Product Publication、Completionは[Product Publication／Completion](product-publication-completion.md)の一方向後段であり、本Acceptanceへ先回りして埋め戻さない。`package-profile.apple.managed-assets`は別minimum-OS variantで、C2 baselineの`package-profile.apple.bundle` Distribution Subjectへ混在させない。

AcceptanceはReceipt発行元のdomain meaningを変更せず、same release、same candidate、same Engine source、required Host／runtime Target／locale集合、freshness、non-revocation、required scenario set equalityを検証する。`candidate_ref`はRelease Content Manifestの`candidate_ref`、`host_profile_refs[]`、`target_profile_refs[]`、`locale_profile_refs[]`はManifest、Engine Release、Release Requirement Projectionの対応集合と各々byte／set equalityにする。First PlayableのTarget Packageとinstall／launch ReceiptはWindows Ownerからdistribution profile、GameInput／VCLibs prerequisite result、clean-machine before／after state、uninstall resultをread-backし、exact `package-profile.windows.msix`と一致させる。MSIX payloadだけの生成、prerequisite済みHost、managed layoutまたはportable artifactをclean-install successにしない。Host Distribution ReceiptはManifestの全Host Distributionへexactly one件対応し、Editor、Engine runtime、Launcher、CLI／headless、全tool、installer／layout、entry set、Toolchain、public contract、Engine sourceの同一性を検証する。`bootstrap_receipt_refs[]`のTemplate projectionは`TemplateBootstrapRequirementSetV1(M)`とset equalityで、各Receiptが束縛するexact `ProjectBootstrapProfileV1`のTemplateとrequested Target setを同rowへbyte／set equalityにし、clean destination bootstrap、first Project revision atomic publication、open／build／test、failure時no partial Projectをcoverする。Documentation Requirement SetはManifestのexact Bundleから§5.1 Algorithmで生成し、AcceptanceのRefとReceiptのRefをbyte equalityにする。Project test resultはsame public contract setとProject revision、Privacy acceptanceはreleaseと同じProduct definition、Toolchain、public contract set、Host／runtime Target／locale集合、Candidateへ一致させる。Legal Applicability Profile／Decision／HeadはAcceptance Fieldではなく、Release Decisionが本Acceptanceと同じProduct／Candidate／Manifest／Claim Scopeへ別Owner inputとして束縛する。

本書の全Named Algorithmは、`QualificationScenarioRefV1`を持つrowを生成する前に[AI Verification／Provenance](../01-governance/ai-verification-provenance.md#verification-identity-spine)のVerification Semantic Admissibility Predicate v1を適用する。rowのHost／Target／locale／Reference dimension scalar scope、expected branch、Evidence Class、subject contract集合がScenario／Class／owner Typeにadmissibleでない場合はRequirement SetまたはProjection生成を失敗させ、無効rowをrequired集合へ複写しない。Acceptanceは全`EvidenceRefV1`／`QualificationReceiptRefV1`について同predicateを再実行し、generic wrapperの`verification_scope`、`subject_contract_refs[]`、scenario、observed branchをowner-specific completed recordとread-backする。Refまたはcontent hashだけ、scopeなしgeneric Receipt、別ScenarioでvalidなReceipt、set equalityへの同じ無効tuple複写を成功にしない。

Documentation Receiptの`entry_results[]`、`link_results[]`、`snippet_target_results[]`、`tutorial_results[]`の各full projectionはRequirement Setの対応集合とexact set equalityにする。Tutorialは`{Documentation Entry Ref,Entry locale,Entry source artifact,Entry rendered artifact,scenario,Target applicability,branch}`をReceiptから全Field read-backし、exact Entry content hash、Candidate、public contract setを同一subjectへ束縛する。Entry render successだけをTutorial実行successへ、generic scenario成功だけを複数Entryへ、en-US Entryをja-JP Entryへ、content hash変更前のEvidenceを変更後Entryへ数えない。

`sdk_qualification_requirement_set_ref`は同じRelease Manifest、SDK Manifest、SDK Distribution Subject、public contract、API catalog、Documentation snippet requirementから§3.1 Algorithmで生成したexact setへ解決する。`sdk_qualification_results[].requirement_binding` projectionはRequired SDK binding集合とset equalityで、各Receiptは同じHost、Target、SDK Subject／artifact、action、Snippet、scenario、branch、Candidate、public contract、API catalogをread-backする。acquire成功をinstall／repair／uninstallへ、Target一件を別Targetへ、snippet compileを全public APIへ、generic JourneyまたはDocumentation ReceiptをSDK Qualificationへ流用しない。

`lifecycle_transition_requirement_set_ref`は同じEngine Release、Manifest、全Installation Channel、Acceptanceが束縛するexact `ProductUpdatePlanRefV1`集合から§7 Algorithmで生成する。`lifecycle_transition_results[].requirement_binding` projectionはrequired transition集合とset equalityで、observed before stateはRequirementのbefore state、successのobserved afterはexpected after stateへbyte equalityにする。policy rejectionはbefore／after不変、domain failure／recoveryはRecovery Policyが許すafter branchとEvidence集合を必須にする。update resultのPlan RefはRequirement Set、Result、ProductUpdatePlanのsource／destination release、channel、scenarioへ全Field一致し、install／repair／uninstallへPlanを付け替えない。flat update／rollback／repair Receipt、Target Package uninstall、artifact一件の復元またはprocess exit codeを製品全体transitionへ数えない。

`support_qualification_requirement_set_ref`は同じEngine ReleaseとSupport Windowから§8 Algorithmで生成する。Windowのsupported Target集合はEngine Release、ManifestおよびAcceptanceのruntime Target集合とset equalityにする。`support_qualification_results[].requirement_binding` projectionはrequired Support binding集合とset equalityで、各Receiptはchannel、locale、Target scope、authentication、attachment、scenario、branch、release／windowをread-backする。別Target、別locale、別auth、Support Bundle export一件またはgeneric support Journeyを不足keyへ流用せず、`target_independent`をTarget-specific Scenarioへ代用しない。

`release_project_test_requirement_set_ref`は同じManifestから生成した`ReleaseProjectTestRequirementSetV1(M)`へ解決する。`project_test_qualification_bindings[]`のProject subject projectionは同Requirement Setの全Project subjectとset equalityで、各bindingのCatalog／Project Requirement Setを対応rowへbyte equalityにする。Target binding projectionは解決先`ProjectTestRequirementSetV1.target_requirements[]`とset equalityである。各Targetにある全Run ResultのSelection Closureで`selected`となったfull `{Suite,Case,Parameter identity,Target,test kind}` projection、`final_case_outcomes[]`の同identity projection、同Targetの`case_requirements[]`から`expected=passed`を除いたprojectionを三者set equalityにし、excluded／filtered／quarantined／unsupported dispositionとのintersectionをexact emptyにする。各required identityのterminal Attemptはordinal 1、`retry_lineage.kind=initial`、`attempt_status=passed`で、final statusとRun statusも`passed`でなければならず、retry後pass／flakyをrelease passへ数えない。Templateの`observed_project_revision_ref`は同Template Bootstrap Receiptがatomic publishしたrevision、SampleではManifestの`expected_project_revision_ref`とbyte equalityである。Request、Selection、Attempt、Resultは同revision、Candidate、public contract set、Target、runner、environmentを全Field read-backし、failed／cancelled／infrastructure error、unsupported／quarantined required Caseをpassedへ数えない。Sample SのT2欠落、別Template／Sample Result、別revision、Catalog外Case、4,096件でのglobal打切り、複数Run間の同Case duplicateまたは一Runの別Target流用を拒否する。

`target_package_runtime_entry_bindings[]`は、Manifestの各Target Packageが列挙する全`runtime_entry_package_refs[]`の各`{target_package_ref, runtime_entry_package_ref}` pairへexactly one対応する。そのpair projectionは全Target Packageのtyped outer entryをflattenした集合とset equalityであり、各Target Packageの`runtime_entry_distribution_package_manifest_ref`がread-backする全launchable outer entry集合およびPackage artifactのcanonical inspection集合ともset equalityにする。各bindingのdistribution Manifest refはTarget PackageとPackage success Receiptの同Ref／hashへbyte equalityにし、Manifestのpackage artifact、Source Snapshot／closure／transport、Candidate、Target、Project revision、Runtime／public Contract Setを同じTarget Packageへ閉じる。`runtime_entry_launch_selection_ref`はそのManifestの同entry row、entry artifactおよびlaunch descriptor artifactへexactly one解決する。各binding内の`qualification_scenario_receipt_bindings[]`のscenario projectionは、対応するTarget Packageの同entry `runtime_entry_qualification_scenario_bindings[].qualification_scenario_refs[]`とset equalityにする。

各outer entryが解決する`source_project_revision_ref`と`target_profile_ref`はTarget Packageへbyte equalityで、Candidate、Package artifact、typed distribution Manifest、selected runtime Contract Set、Engine Releaseのpublic contract setはPackage／install／launchの三Receiptを含む同じowner-typed Qualification subjectへ閉じる。`package_receipt_ref`は同じTarget PackageのPackage artifact、Snapshot content hash、Source closure ref／content hash／transport artifact、distribution Manifest ref／hashを生成時にread-backしたsigned success Receiptである。各scenario rowのInstall Qualification Receiptはexact Package artifact／Manifestをpackage-level install subjectとして、Launch Qualification Receiptは同じManifestとexact Launch Selectionをentry-level launch subjectとして同じDevice／scenarioでread-backするfresh non-revoked Receiptへ解決する。Install Receiptへ存在しないper-entry選択、Launch Receiptへ存在しない別entryまたはscenario意味を付加せず、scenario／expected branchはQualification ReceiptがOwner Receiptのrequest／resultとともに束縛する。world branchではSelectionが選ぶouter entryの`world_package_ref`とQualification subjectのinner `RuntimePackageRefV1`をbyte equalityにし、UI-only／worldless headless branchではinner Runtime Packageを要求せずnullを維持する。複数entryのmissing／extra、別Target／revision／Source closure／Candidate／Package／Manifest／Selection／scenario／contract set、W1 ManifestをW2 artifact／Selectionへ付けるstitch、W1をqualifiedしてW2をlaunchするstitch、E1 launchでE2をclaim、descriptor hashだけ、UI／headlessへの偽World、worldのnull、outer／inner Ref型の代用を拒否する。`package_entry_set_hash`またはbare Manifest hashだけの一致、同じReceiptへのref列挙またはPackage／launch成功の一方だけをentry closureの代用にしない。`build_cook_package_receipt_refs[]`のPackage Operation subsetと`target_package_runtime_entry_bindings[].package_receipt_ref` projection、`install_launch_offline_receipt_refs[]`のinstall／launch subsetと全Target Package nested scenario rowのinstall／launch Qualification Receipt projectionはそれぞれset equalityにする。後者の上限は`16384 × 256 × 2 = 8388608`から導出し、cook、offlineまたは別OperationのReceiptをpackage／install／launchへ数えない。Workflow-specific Owner／Qualification Receiptは`workflow_sample_qualification_bindings[]`内のoperation-specific unionだけでexactly once保持し、このrelease Target Package用flat vectorへ重複複写しない。

`reference_project_qualification_bindings[]`のfull `{Sample, Target, dimension, reference requirement, scenario, expected=success, family=project_launch, operation, execution=launch}` identity projectionは§5.2 `ReferenceSampleQualificationRequirementSetV1(M)`とset equalityで、各tupleへexactly one rowを持つ。各bindingのTarget PackageはManifest内で同Targetかつdimension対応roleのexact Package、Sampleは同Targetをsupportする同dimension roleのexact Sampleでなければならない。同じSampleの各supported Targetへexactly one `sample_reference_coverage[]` rowが存在し、そのTargetの全requirementがbindingへ現れる。同Targetを別Sampleだけで充足する、一requirementを複数Coverage rowから重複展開する、distinct Sample集合だけを維持して一Target／scenarioを落とすことを拒否する。Sample、Target Packageおよびbindingの`source_repository_snapshot_ref`はProject State Ownerが定義する`{project_ref, project_revision_ref, source_closure_hash, snapshot_content_hash}`全Field、`source_closure_ref`およびcanonical transport artifactでbyte equalityにし、Sampleの`expected_project_revision_ref`、Target Packageの`source_project_revision_ref`およびbindingの`source_project_revision_ref`はそのSnapshotの`project_revision_ref`へbyte equalityにする。Qualification Receipt subjectはSampleの`source_project_artifact_ref`が同Snapshot Source closureのexact canonical transport artifactであり、Target PackageのPackage Receiptが同じSnapshot／closure／transportをread-backしたことを検証する。generic ArtifactRef、同revision表示またはReceiptへのref併記からmembershipを推論しない。

各Reference bindingの`runtime_entry_distribution_package_manifest_ref`はTarget Package、Target Package Runtime Entry bindingおよびPackage Receiptの同Refへbyte equalityにする。`scenario_runtime_entry_launch_bindings[]`はTarget Packageとtyped distribution Manifestのouter entry集合から選ぶnon-empty exact subsetで、各rowのSelectionは同Manifestの同outer entry／entry artifact／descriptor artifactへexactly one解決する。各`{target_package_ref, runtime_entry_package_ref, launch_selection_ref, qualification_scenario_ref}` tupleは`target_package_runtime_entry_bindings[]`のexact一行と、その一行のnested scenario rowへ解決する。同じReference Target Packageについて全required scenario bindingのentry projectionをcanonical unionした集合は、そのTarget Packageとtyped distribution Manifestの全outer Runtime Entry集合とset equalityにする。CandidateはAcceptanceおよびRelease Content Manifest、scenario／expected branchはSampleのexact requirement、Qualification ReceiptはSample／Snapshot／Source closure／canonical transport artifact／Target Package／Package artifact／typed distribution Manifest／Launch Selection／outer Runtime Entry／branch closure／worldの場合のinner Runtime Package／Target／Project revision／Candidate／scenario／public contractを同一subjectへ束縛するfresh non-revoked owner-typed Receiptへ解決する。required Target×dimension集合がemptyならReference Sample、Coverageおよびbinding集合もexact emptyである。missing、extra、duplicate、wrong Target、cross-dimension、cross-Candidate、cross-Project snapshot／revision／source closure、cross-Package／Manifest／Selection／outer entry、scenario substitution、workflow SampleによるReference充足を拒否し、Package、Sample、Artifact、Receipt、descriptor hashまたはinner Runtime Packageの存在だけからQualification成功を導出しない。

`workflow_sample_qualification_bindings[]`のdistinct Sample projectionは`WorkflowReleaseSampleSetV1(M)`とset equalityで、各Workflow Sampleへexactly one outer rowを持つ。全outer rowの`qualification_requirement_bindings[]`をSample refと組にしてflattenしたidentity projectionは§5.2 `WorkflowSampleQualificationRequirementSetV1(M)`とset equalityにし、各required `{Sample, target applicability, scenario, expected result branch, operation family, operation, execution branch}` tupleへexactly one nested rowを持つ。outer rowのSample、Snapshot四Field、Source closure ref／content hash、canonical transport artifact、Project revisionおよびCandidate、nested rowのTarget applicability、scenario、expected branch、operation family、Operationおよびexecution branchはSample manifestとbyte equalityである。各nested `qualification_receipt_ref`はそのfull tupleを同一subjectへ束縛し、Owner ReceiptのOperation Refをbyte equalityでread-backするfresh non-revoked owner-typed signed Receiptであり、Sample carrier membership、Distribution Subject、SDK／Documentation Receipt、Project test resultまたはReference bindingだけをWorkflow Sample Qualificationへ数えない。

Workflow bindingは次のoperation-specific closed unionへ従う。

| `execution_branch`／expected branch | `operation_evidence.kind` | Target／Package／Manifest／Selection | Owner Receiptが証明する意味 |
|---|---|---|---|
| `authoring | build | test`／全branch | `authoring_build_test` | `target_independent`またはexact Target。Package／Manifest／Selectionなし | 同Operationのrequest、Project、Candidate、Target applicabilityとowner result。scenario／expected branchはQualification Receiptが束縛 |
| `package`／`success` | `package_success` | exact Target、生成済みTarget Package／Manifest／Manifest全outer entry集合。Selectionなし | Package success output group全体と同request |
| `package`／`expected_policy_rejection | domain_failure_recovery` | `package_non_success` | exact Target。attempt出力のTarget Package／Manifest／entryは存在しない。last-valid before／afterは独立state | Package failed／cancelled Receiptの同request、typed Diagnostic、success output group canonical omission |
| `install`／全branch | `install` | exact input Target Package／Manifest。per-entry Selectionなし | Package-level install input／result。per-entryまたはscenario意味を付加しない |
| `launch`／全branch | `launch` | exact input Target Package／Manifest、各entryのexact Launch Selection | 各SelectionごとのLaunch request／Task／Receipt result |

`owner_request_sha256`はOwner Receipt Envelopeの`request_sha256`とbyte equalityにし、outer Sample／Snapshot／Candidate／Target applicabilityおよびQualification Receiptのscenario／expected branch／operation family／Operation Ref／execution branchと同一subjectへ閉じる。`package_non_success`のOwner ReceiptはCoreが規定する成功output groupを全て省略し、生成されなかったTarget Package、Manifestまたはouter entryをQualification wrapperへ出力として付けない。`last_valid_before_target_package_ref`と`last_valid_after_target_package_ref`は両方nullまたは両方exactで、Owner recovery policyが要求する不変／規定recoveryでは両Refをbyte equalityにする。before／after State Evidenceは同じState Owner／scopeをread-backし、stale成功Packageをfailed attemptの出力へ付ける、ReceiptへのRef併記だけで回復を主張することを拒否する。

`install`はPackage artifact／Manifest／Deviceをinput subjectとするpackage-level Receiptであり、outer entry集合をOwner Receiptの意味へ追加しない。`launch`の`entry_launch_receipt_bindings[]`は各Selectionと各`operation.device.launch` Receiptをone-to-oneで束縛し、同じWorkflow Sample／Target Packageのsuccess launch requirementsに現れるentry projectionのcanonical unionを、そのSample requirementがlaunch対象として選択した全entry集合とset equalityにする。non-success Install／Launchではtyped Diagnosticとbefore／after State Evidenceを必須にし、success output presence、状態不変またはOwner規定recoveryをexpected branch別に検証する。E1 ReceiptをE2 Selectionへ付ける、descriptor SHAだけ、Package Receiptを一entry Receiptへ流用する、Install Receiptをentry選択証明へ流用する、Qualification Receipt内の併記だけでOwner Receipt subjectを拡張することを拒否する。

`target_independent`へTargetを補完する、authoring／build／testへ偽Package／Manifest／Selectionを注入する、package non-successへ存在しない完成成果物を要求する、installでper-entry意味を主張する、launchでTarget／Package／Selectionを省略する、direct Manifest-only Sampleを除外する、同名／同role／同Artifact hash／同revisionの別Sample、Reference Sample Receipt、wrong Target／scenario／expected branch／operation family／Operation Ref／execution branchを代用することを拒否する。Workflow Sampleが0件ならrequirement集合とbinding集合をともにexact emptyにし、Reference集合の有無からWorkflow集合を推測しない。

Acceptanceの`required_operation_journey_projection_ref`はEngine ReleaseのProduct Definition／Claim Scope／Release Requirement Projection／Required Operation Universeへexact解決する。Cross-surface Profileも同じEngine Release／Product Definition／Claim Scopeへ解決する。`parity_receipt_refs[]`の`{claim scope,requirement,semantic group,family,operation,surface,client profile,host scope,target scope,locale scope,reference dimension scope,scenario,expected branch,evidence class}` projectionは、Required Journey Projectionの各required rowと同surface／familyへapplicableなCross-surface Client Profile集合を組み合わせたexact pair集合とset equalityにする。Receiptからclient profile、branch／Evidence classを除いたJourney projectionと、semantic groupを含む`forbidden_surface_entries[]`のintersectionはexact emptyでなければならない。missing、extra、wrong Claim Scope、wrong Requirement、wrong semantic group、wrong family、wrong Operation、wrong surface、wrong client／Agent Host Profile、wrong Host／Target／locale／Reference dimension scope、wrong scenario、wrong branch、wrong Evidence classをそれぞれ拒否する。Operation Activation Closure、`workflow_coverage[]`、generic Evidence、別Requirement、別semantic group、別surface、別client versionまたは別familyの成功をJourney Evidenceへ数えない。一localeまたは一dimensionのReceiptを別locale、別dimension、`*_independent`または`not_applicable` rowへ流用しない。

AcceptanceのDistribution Coverage Projection refはManifestとEngine Releaseの同Fieldへbyte equalityにする。`distribution_control_evidence_bindings[]`の`{subject,control}` projectionは全Applicabilityのrequired control pairとset equality、forbidden pairとのintersection exact emptyにする。Notice Presentation subject集合は`third_party_notice`または`first_party_license_presentation`がrequiredなSubject集合とset equalityにし、各required source／containerへexactly one件以上の到達可能なpresentationを要求する。`pack_lifecycle_acceptance_refs[]`の`{pack_requirement_ref,pack_contract_ref}` projectionはCoverageの`pack_distribution_coverage[]`とset equalityで、Product Definitionがrequiredまたはbundledとする全Pack journeyへexactly one件ずつ対応する。各recordのProduct definition、Toolchain、public contract、Candidate、Target、Pack identityと、全action bindingのbefore／after Project revision、Pack Registry head、installed closure、dependency closure、typed signed Receipt lineageをRelease Content Manifestの`pack_contract_refs[]`、Coverageの同pair row、Pack Distribution Subject、Required Operation Journey Projectionへexact照合する。別requirementへの付替え、Pack ID、version、artifact hash、Target、scenario、action lineageまたはpublication routeが一つでもmissing／extra／mismatchならacceptanceを拒否する。first-party licenseはManifest／Engine Releaseのexact `ProductLicenseGrantRefV1`、そのlegal review Evidence、Applicabilityがrequiredにした全Subjectの`ProductNoticePresentationV1`、`first_party_license_presentation` control Evidenceからだけ導出する。独立した`product_license_receipt_refs[]`、installer同意、Website textまたは一Subjectのpresentationを別のlicense authorityにしない。

全`QualificationScenarioRefV1`、`EvidenceClassRefV1`、`EvidenceRefV1`、`QualificationReceiptRefV1`はAI Verification OwnerのScenario／Class／record-type root Registryからcomplete recordへ、全`OperationReceiptRefV1`はExecutable Contracts OwnerのReceipt Type Registryからgeneric wrapperとowner-specific backing recordへ解決する。Lifecycleはclass、purpose、record kind、owner Type、owner record ID／version／hash、subject hash、completed signed-record hashを全Field read-backし、consumer-local tuple、ID-only、hash-only、表示名、prefix、`latest`、wrong class／purpose／Operation／Target／locale／dimension substitutionを拒否する。Product Release Decision Ownerはこのacceptanceを入力にできるが、本書自身がRelease Approval、Product completionまたはCapability Activationを発行しない。

## 10. failureと禁止fallback

- release、public contract、Target、hash、license、qualificationのexact不一致はmutation前に拒否する。
- `latest`、同名、近いversion、別Target、別Sample、既定Templateへ置換しない。
- bootstrap failureではcurrent Projectまたはopenable partial Projectを作らない。
- update failureでは旧release／旧Projectを維持し、新Project revision成功を旧Package成功で代用しない。
- Documentationのbroken link／snippet、stale tutorial、unrunnable Sampleをwarningだけでreleaseしない。
- direct ManifestだけをSample universeとして扱わず、SDK／Documentationに隠れたReference／Workflow Sample、Release Sample集合外Subject、Subject集合外Sampleをrelease acceptance成功にしない。
- Sample、Target Package、Acceptance binding、ReceiptのProject snapshot四Field、Source closure ref／content hash、canonical transport artifact、source entry membershipまたはProject revisionが一致しない場合、同role／Target／dimension／scenarioでもQualificationへ数えない。
- Workflow SampleのTarget applicability／scenario／expected branch／operation family／Operation Ref／execution branchのmissing、extra、duplicate、carrier membershipだけの充足、Reference Receipt流用、target-independent rowへのTarget補完、package non-successへの偽完成Package／Manifest、Install Receiptへのper-entry意味付加またはLaunch Selection欠落をrelease acceptance成功にしない。
- SDK artifact／public API catalog／Documentation mismatch、Project test欠落をrelease acceptance成功にしない。
- Target Packageのouter Runtime Entry集合とtyped distribution Manifest／Package artifact／Launch Selection／package／install／launch Evidence集合が一致しない場合、inner World Package、entry-set hash、bare Manifest／descriptor hash、別entryのlaunch成功または同じPackage名で埋めない。
- Release Content Manifestに存在しないEngine source／Host Distribution／Candidate／Target Package、配布container集合の一部だけのSBOM／NOTICE、別Project revisionのPackageをrelease acceptance成功にしない。
- Privacy acceptance、Pack lifecycle acceptance、first-party license、SBOM／NOTICE、User到達可能なPackage別presentation、support windowの欠落をrelease acceptance成功にしない。
- surface adapter failureを別surfaceまたは別client versionの成功で埋めず、GUI／CLI／headless／Native SDK／external IDE／MCP／AI automationのどれか一つをsemantic authorityにしない。

## 11. Qualification

最低限、同一release candidateについて次を検証する。

1. clean environmentでProject createから最初のatomic `ProjectRevision`まで完走する。
2. Editor GUI／CLI／headlessが同じrequest meaning、candidate hash、result、typed diagnosticを返す。
3. TemplateとSampleからProject test、cook、package、clean install／launch、offline playまで完走し、全Target PackageのSource Snapshot／closure／canonical transport、typed distribution Manifest、outer Runtime Entry集合、exact Launch Selection、Package artifact membership、branch closureおよびpackage／install／launch Receipt集合をexact set equalityで照合する。
4. SDK install／repair／uninstall、public API snippet、tutorial、link graph、`ReleaseSampleSetV1`がsame public contract setに一致し、Manifest／SDK／Documentationの各carrierとSample Distribution Subjectのcanonical unionを完全にread-backする。全`WorkflowReleaseSampleSetV1` memberのtyped requirement集合とoperation-specific Acceptance unionをexact equalityにし、direct Manifest-only Sampleを含む全Target／target-independent scenarioを完走する。Package rejection／recoveryでは完成Package／Manifest出力を省略し、Installはpackage-level、LaunchはSelectionごとのOwner Receiptとして検証する。
5. clean install、supported update、failed update、rollback、repairを独立fixtureで検証する。
6. claim-derived Reference Dimensionについて、`none`は全carrierのReference subset exact empty、2D-only／3D-onlyは各required Targetの該当dimensionだけ、2D＋3D claimは各required Targetの両dimensionを同じrelease candidateで別scenario集合として完走する。各Reference Sampleの全supported Target×全requirementを`ReferenceSampleQualificationRequirementSetV1(M)`とAcceptanceのfull identity projectionでexactly once照合し、TargetごとのCoverage、Target Package、binding、Receiptを同じProject snapshot四Field／Project revision／Source closure／canonical transport artifact、typed distribution Manifest、実launchしたLaunch Selection／outer Runtime Entry／branch closureへexact joinする。
7. diagnosis、`SupportBundleV1` export、support window、data resetを検証する。
8. Privacy consent／withdraw／export／delete、Pack acquisition／install／update／remove、first-party license、SBOM、NOTICEのpresentationがexact release／全Target Packageと一致しUserから到達できる。
9. wrong release／Target／hash、stale docs、partial Project、cross-candidate Receipt、cross-Project snapshot／Source closure／transport、hidden／orphan／unqualified Workflow Sample、Reference Sampleの一Target／requirement欠落、4,096件でのCoverage打切り、typed distribution Manifestのmissing／extra／artifact stitch、Launch Selectionの別entry／別Manifest／descriptor hash代用、package failureへの偽完成成果物、Install Receiptのper-entry流用、outer Runtime Entryのmissing／extra／substitution、world inner Packageとlaunch rootのstitch、Package集合の欠落、Privacy／Pack／license不一致をnegative fixtureで拒否する。

Windows Preview iterationは[Native Game Module](../03-authoring/native-game-module.md)のrestart-based contractを使用する。Shipping static link、Preview DLLのprocess起動時一回load、変更時`GameHost` restart、in-process unload／binary replacement／live code patch禁止を維持し、Hot Reloadという別Capabilityを追加しない。

## 12. CMake／Platform projection

[CMake Presets 4.4](https://cmake.org/cmake/help/v4.4/manual/cmake-presets.7.html)はproject／user presetとconfigure／build／test／package／workflow presetを定義する。MiraikanaiはCMake Presetをtyped Build requestのprojectionとして使用できるが、Preset、CMake cacheまたはIDE profileをProject／Build authorityにしない。

Windows package identity、version、differential update、signature、device trustは[Microsoft MSIX package updates](https://learn.microsoft.com/en-us/windows/msix/app-package-updates)と[MSIX signing overview](https://learn.microsoft.com/en-us/windows/msix/package/signing-package-overview)をPlatform OwnerがTarget別に比較する。Product Lifecycleは証明書、Store policy、package schemaを所有せず、それらのfresh Receiptをsame release acceptanceへ束縛する。

Android、Apple、headless distributionも同じProduct lifecycle meaningを使うが、Windows Receiptを流用しない。C2ではAndroid／Appleを後続Mobile phaseまたはoptional Targetとして扱わず、Product Planのexact runtime Target集合に含むrequired Targetとして集約する。各Platform Ownerがpackage、signing、install、update、rollback、device qualificationを所有し、本書はTarget-specific resultを同じCandidateへ束縛するだけである。

## 13. 明示的非目標

- Product Lifecycleを新しいBuild、Package、Migration、Evidence authorityにすること
- Plugin marketplaceまたは任意native Plugin ecosystem
- 汎用Game Script VM、JIT、Game向けFFI
- Multiplayer、Account、Cloud、広告、課金
- Runtime code generation
- large open worldまたはhigh-end renderingをMVP lifecycle前提にすること
- `build/` root、latest lookup、silent repair、部分migration、互換aliasを再導入すること

## 14. 完了条件

- Candidate、Project snapshot四Field／Project revision／Source closure／canonical transport artifact、全Target Packageのtyped distribution Manifest／outer Runtime Entry集合／Launch Selection、Engine release、SDK、first-party license、Template、`ReleaseSampleSetV1`と全Reference／Workflow Qualification requirement、Documentation、update、support、Privacy、Pack、NOTICEがexact Refで一方向に閉じる。
- 全local RefがID、version／revision、content hashを持ち、Documentation Entry／Link／Bundle、Support Window／Closure、Release／Acceptanceのhash graphがDAGである。
- GUI／CLI／headless／Native SDK／external IDE／MCP／AI automationが同じOperationへ収束し、surface固有authorityを持たない。Product-supportedな各Agent Host／adapter versionは個別ProfileとReceiptを持つ。
- bootstrapはatomic、updateはseparate Candidate＋single promotion、failureはlast-known-goodを維持する。
- Product Release GateがSDK、Project testing、claim-derived 0／1／2 dimension Referenceの全Sample×Target×requirement、release Sample carrier union、Workflow Sample全件のoperation-specific typed Qualification、Target Package Source provenance／typed distribution Manifest／Runtime Entry launch closure／exact Launch Selection、Documentation、Privacy、Pack lifecycle、first-party license、support、全Target PackageのNOTICEをBuild成功と別に検証できる。
- Product Lifecycleが各domain Schemaを複写せず、各Ownerのfresh Receiptだけを集約する。
- Schema、Operation、Fixture、Receiptが未materialize／未ActivationであることをArchitecture InventoryとClosure Reviewが保持する。
