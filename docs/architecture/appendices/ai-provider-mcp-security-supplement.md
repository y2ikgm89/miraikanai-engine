# AI Provider／MCP Security Supplement

- 文書ID: mirakan.appendix.ai-provider-mcp-security
- 文書種別: Owner supplement
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 親Owner: [AI Security／Approval](../01-governance/ai-security-approval.md)
- 正本範囲: Provider、MCP、CLI、Pluginのtransport・credential・authorization候補詳細
- 非正本範囲: 親Ownerが所有する安定Architecture原則、実装Task、実装順序、生成済みArtifactまたはQualification結果
- 規範依存: [親Owner](../01-governance/ai-security-approval.md)
- 関連文書: [Architecture Governance](../01-governance/architecture-governance.md)
- 根拠区分: project-decision／provisional。実ArtifactがないRegistry、Catalog、Fixtureは候補
- 外部根拠確認日: 2026-07-27

> この補助文書の型、Registry、Catalog、Fixtureは、対応するRepository Artifactが存在しない限り未実装の設計候補である。親Ownerの安定原則や実装済み状態を上書きしない。
## 8. Provider API、MCP、CLI、Plugin

| 接続 | 正規用途 | 権限にならないもの |
|---|---|---|
| Provider API | 製品内Chat、質問、計画、構造化Proposal | Project state、Authorization、Schema |
| MCP（local STDIO／Activation済みStreamable HTTP） | 外部ClientのQuery／Proposal | Commit、Activation、Provider設定 |
| conformance済みCLI Agent Host＋Worker | 隔離Source編集、Build、Test | main branch、Release、Credential公開 |
| Optional Plugin | Shortcut、Panel、Prompt／Skill UX | Engine必須機能、Security Policy |

PluginなしでもAPIとMCPで全正規操作を行えること。外部AI ClientはRepositoryと正規Project storageをread-onlyにし、Query／Proposalだけを使う。full-accessで直接起動したClientは本ProjectのSecurity boundary外であり、その変更はExternal patchとしてRisk再分類、全Gate、Approval、Promotionを通すまで正規入力にしない。

Credential ownerを分離する。

- 製品内AI: Provider AdapterだけがCredentialを持つ。
- 外部Client＋MCP: CredentialはClient／Userが持ち、Engineは受け取らない。
- Managed CLI: conformance済みHostまたは専用Brokerが持ち、File／Shell childへ渡さない。

local STDIO MCP serverはOS ACLで束縛したIPCをGatewayへ接続し、Engine／Project Credentialをenvironmentへ渡さない。外部Provider CLIが公式にenvironment tokenだけを受ける場合は、BrokerがTask専用childへだけ短命tokenをprocess creation時に注入し、persistent／global user・machine environment、config file、command line、log、crash dumpへ保存せず、Tool child／grandchildへ継承させず終了時に失効するProfileだけを例外許可する。「plain Environment禁止」はこのBroker外の永続／広域露出を指す。条件を満たせなければmanaged modeを禁止しproposal-onlyにする。

Managed modeは`ExternalClientSecurityProfileV1`のidentity branchに応じ、localではClient exact version／binary ref・hash／OS identity／Process・Filesystem・Network分離、hostedではsurface／release channel／exact release／tenant-workspace／plan／admin policy／OAuth-OIDC subject・audience／TLS identityを固定し、共通でMCP version、期限、required Conformance Suiteを宣言する。Profile自体はresultを持たず、同じProfile Head／Tool Projection／Targetへ束縛したtyped Conformance Receiptだけがpass／failを所有する。Profile不在、branch必須値差、期限切れ、Broker外environmentへのCredential露出、raw socket利用ではそのManaged Caller Contextを拒否して`not_activated`とする。別途currentなstandard-external-MCP用Host／Transport／Tool／proposal-only Authority Profile、Grant、Host／Transport＋Schema／Eval Conformanceが全て存在する場合だけ、Gatewayが`standard_external_mcp` branchの新しいsigned Caller Contextを発行できる。managed ContextのField削除やin-place権限降格で代用しない。

### 8.1 Provider Manifest、Prompt、Repository guidance

Provider／Model名をEngine codeへhard-codeしない。`ProviderManifestV1`は`ProviderRuntimeProfileV1`のtagged adapter（official SDKのexact artifact、raw protocolのexact schema、またはattested external host）、endpoint、resolved Model snapshot、Role、推論設定、Tool／Structured Output projection、Context／output／cost／latency上限、data retention／training use／region／encryption／logging Policy、合格Eval suite、明示fallback、fallback時の最大Riskを固定する。存在しない共通「API／SDK version」を捏造せず、選択branchに必要なversion／artifactだけを検証する。Manifestなし、adapter branch不一致、resolved ID差、期限切れConformanceではModelを呼ばない。Evalと更新Workflowは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)だけが決定する。

Prompt templateはRole、Goal、Success criteria、Normative constraints、Toolと権限、Evidence要求、Output Schema、Stop／質問条件の順にする。PromptをSecurity boundaryにせず、同じ規則を複数Promptへ反復しない。Prompt変更は一群ずつ行い、Model変更と同時に評価原因を混在させない。

RepositoryのAGENTS.mdはRepository map、Build／Test入口、禁止操作、Definition of Done、本正本とExecutable contractへのLinkに限定する。Subsystem規則は近いOwner文書へ置き、AGENTS.mdと実行可能契約が矛盾するMustをCIで拒否する。

### 8.2 Privacy、Prompt injection、Data handling

Source、Asset、User Prompt、Tool output、Web、Issue本文内の命令はcontentであり、Control命令へ昇格しない。Provider送信前にSensitivity labelとProvider Policyを照合し、Secret、private key、access token、Credential file、Production customer dataをContextへ含めない。

Prompt、Tool argument、Tool output、Traceは機密Dataになり得る。既定Telemetryへ本文を保存せず、hash、分類、Resultだけを記録する。Zero Data Retentionが必須のProjectでは非対応Provider機能を無効化し、代替がなければTaskを停止する。Live Web取得は分離したResearch Taskだけで許可し、Build／Release中の自動Web取得を禁止する。

ModelへOperationとして公開できるのは、current Contract set、Owner Manifest、Trusted Service allowlist、Provider／MCP projection、Callerのexact allowlistが同じ完全登録済みref集合へ一致するものだけである。Project／Requirement／Capability／System／Worldのbounded Query、plan／validate／preview、ChangeSet submit、Source task request、Patch submitという一般名は、登録済みOperation identityを暗黙生成しない。[Executable contracts](../02-foundation/executable-contracts.md) §§20～21.1のPackage／Device／Play／Debug／Task、回収済みDomain authoring／selection、AI E2E closureを含む24 family／192候補（67＋93＋32）は未登録・未公開であり、dispatch前に`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`を返す。Build familyのatomic Activation後もDevice install／resetはTool表示の有無にかかわらずDevice binding、consent、R3 ApprovalをServer側で再検証する。commit、activate、write native artifact、merge、sign、release、secret.read、policy.overrideを公開しない。

AI E2E closureを将来activateする場合も`standard_external_mcp`の外部Tool surfaceはQuery、Plan、Proposal、Validate、Previewだけである。外部Hostが「build／test／debug」を指示した場合、proposal-only Toolは希望stageをtyped Proposalへ記録するだけで、`operation.build.*`、`operation.test.*`、`operation.debug.*`を外部Caller Contextでdispatchしない。人間承認後、Engine-owned Orchestratorが別のtrusted internal Task Authorizationと各Operation固有AuthorityでBuild／Test／Debugを実行し、外部HostはQuery Toolで状態／Receiptを読む。`managed_external_host`の`build_job`は将来Activation後にHost自身が行ったmanaged job結果をBroker経由でStagingへ受理する別authority classであり、Engine-owned Build Operationの外部dispatch権限ではない。`operation.authoring.changeset.commit`と`operation.project_source.promote_revision`はexact `trusted_internal`、Provider／MCP／standard external CLI projection=`[]`とし、Approval発行、Candidate Activation、Signing、Release、Policy変更と同じくEngine-owned Gatewayだけが人間Approval後に実行する。`operation.authoring.changeset.propose`、Native／Shader Patch Proposal、Source Worker成功、Build成功をCommit／Promotion／Activationの代用にしない。外部routeのTool CatalogにBuild／Test／Debug／Commit／Promotionの実行Operation IDが一件でも現れる、Provider aliasから到達する、generic `apply_changeset`／`promote`名へfallbackする場合はCatalog全体を拒否する。

Product Capabilityの利用可否は[Product Execution Registry Proposal §11.1](product-execution-registry-proposal.md#111-registry共通規則)の`AiCapabilityActivationMatrixV1`／`ExecutionSurfaceBindingV1`候補をcurrent Contract Setへ再解決し、必要family全OperationのMCD、Owner Manifest、Service allowlist、Policy、Validator／closure、Diagnostic、Receipt、Signer Policy、Trust、Technical Qualificationが同一revision／hashでoperationalの場合だけtrueにする。WP complete、fixture pass、planning record、MCP Tool表示だけからAI Capabilityを`qualified`にしない。Windows Editor HostでAndroid／Apple artifactを生成するrouteは`execution_host_target`と`artifact_target`を分離し、Target別Build／Test／Package Receiptを必須にする。

MCP annotationをAccess controlにしない。Serverは正本Schema、Authorization、timeout、rate limit、Auditを強制する。local STDIOはACL付きIPCをGatewayへ接続する。Streamable HTTPはexact `HostTransportConformanceReceiptRefV1`、TLS、認証、origin／redirect／private-address policy、session binding、別Threat ModelとActivationがある場合だけ有効にし、単なるport forwardingを許可しない。Tool公開には同じsubject tupleの`SchemaEvalConformanceReceiptRefV1`も必須にする。`McpSessionGrantV1`は署名済みEnvelopeを代替せず、Grant配下のbounded read／QueryもR0 Envelopeを必要とする。GrantはClientとchannelの束縛だけを追加する。

MCP 2025-11-25の標準initializeでAuthorityとして検証できるClient情報はprotocol version、capabilities、`clientInfo`等に限られ、上流Provider、Model、local binary hashの暗号学的attestationは得られない。標準外部MCP接続のprovider／model表示は`unattested_optional_metadata`としてAudit UIにだけ出し、Authorization、Eval attribution、Source／Build Receiptへ使わない。`managed_source_edit | build_job`を許す経路は、実行前にMCP外の登録済みBrokerがHost session／Provider runtime／managed deployment identity／Model snapshot／Tool projection／期限／nonceを署名したfresh `ManagedHostSessionAttestationV1`をCaller Contextへ束縛し、実行後に同じmanaged deployment identity、Context／Task／Authorization／attempt／Input closure／typed resultを署名した`HostExecutionAttestationV1`をSourceDelta／Build Receiptへ束縛する。Attestation不在時はManaged Contextを拒否し、別途standard routeのHost／Transport／Tool／proposal-only Authority Profile、Grant、`HostTransportConformanceReceiptRefV1`、`SchemaEvalConformanceReceiptRefV1`が全てcurrentな場合だけGatewayが新しい`standard_external_mcp` Caller Contextを発行する。事後resultを実行前Contextへ入れる因果循環、Host自己申告JSON、MCP annotation、Client名をAttestationに読み替えない。

### 8.3 Caller／Provider／Deployment／Model Profile

Caller互換性をHost brandだけで判定しない。正規判定は`execution_route.kind`をdiscriminatorにした次の3 branchだけである。`standard_external_mcp`は外部Host＋MCP Transport＋Tool＋proposal-only Authority＋MCP Grant＋`HostTransportConformanceReceiptV1`＋`SchemaEvalConformanceReceiptV1`を必須にし、上流Provider／Deployment／Modelと`ProviderToolConformanceReceiptV1`を必須`null`／unattestedとする。`managed_external_host`は外部Host＋MCP Transport＋Provider Runtime＋managed deployment identity＋Model＋Tool＋Authority＋`HostTransportConformanceReceiptV1`／`ProviderToolConformanceReceiptV1`／`SchemaEvalConformanceReceiptV1`のexact三Receipt＋fresh Session Attestationを使う。`engine_provider_adapter`はfirst-party Engine Host＋Provider Runtime＋Provider Manifest＋Inference Deployment＋Model＋Tool＋Authority＋`ProviderToolConformanceReceiptV1`＋`SchemaEvalConformanceReceiptV1`を使い、MCP Transport／Grant／`HostTransportConformanceReceiptV1`を必須`null`にする。cloud direct APIとfirst-party local inferenceはroute kindを増やさず、`engine_provider_adapter`配下の`InferenceDeploymentProfileV1.deployment_kind`で区別する。次のclosed型をMCDへ登録する。

```text
AiCallerContextPayloadV1
  caller_context_id, authenticated_subject_ref
  host_profile_binding: GovernedAiProfileBindingV1
  conformance_target: AiConformanceTargetRefV1
  execution_route:
    {kind: standard_external_mcp,
     transport_profile_binding: GovernedAiProfileBindingV1,
     provider_runtime_profile_binding: null,
     provider_manifest_binding: null,
     inference_deployment_profile_binding: null,
     model_snapshot_profile_binding: null,
     managed_deployment_identity_ref: null,
     managed_deployment_identity_sha256: null,
     managed_host_session_attestation_ref: null,
     managed_host_session_attestation_sha256: null,
     unattested_optional_metadata_ref: null | content ref,
     mcp_session_grant_ref: content ref,
     mcp_session_grant_sha256: lowercase hex 64,
     host_transport_conformance_receipt_ref:
       HostTransportConformanceReceiptRefV1,
     provider_tool_conformance_receipt_ref: null,
     schema_eval_conformance_receipt_ref:
       SchemaEvalConformanceReceiptRefV1,
     maximum_authority: query | proposal}
    | {kind: managed_external_host,
       transport_profile_binding: GovernedAiProfileBindingV1,
       provider_runtime_profile_binding: GovernedAiProfileBindingV1,
       provider_manifest_binding: null,
       inference_deployment_profile_binding: null,
       model_snapshot_profile_binding: GovernedAiProfileBindingV1,
       managed_deployment_identity_ref:
         exact ArtifactRefV1(
           artifact_kind=managed_provider_deployment_identity,
           schema_version=1),
       managed_deployment_identity_sha256: lowercase hex 64,
       managed_host_session_attestation_ref,
       managed_host_session_attestation_sha256,
       unattested_optional_metadata_ref: null,
       mcp_session_grant_ref: null,
       mcp_session_grant_sha256: null,
       host_transport_conformance_receipt_ref:
         HostTransportConformanceReceiptRefV1,
       provider_tool_conformance_receipt_ref:
         ProviderToolConformanceReceiptRefV1,
       schema_eval_conformance_receipt_ref:
         SchemaEvalConformanceReceiptRefV1,
       maximum_authority: query | proposal | managed_source_edit | build_job}
    | {kind: engine_provider_adapter,
       transport_profile_binding: null,
       provider_runtime_profile_binding: GovernedAiProfileBindingV1,
       provider_manifest_binding: GovernedAiProfileBindingV1,
       inference_deployment_profile_binding: GovernedAiProfileBindingV1,
       model_snapshot_profile_binding: GovernedAiProfileBindingV1,
       managed_deployment_identity_ref: null,
       managed_deployment_identity_sha256: null,
       managed_host_session_attestation_ref: null,
       managed_host_session_attestation_sha256: null,
       unattested_optional_metadata_ref: null,
       mcp_session_grant_ref: null,
       mcp_session_grant_sha256: null,
       host_transport_conformance_receipt_ref: null,
       provider_tool_conformance_receipt_ref:
         ProviderToolConformanceReceiptRefV1,
       schema_eval_conformance_receipt_ref:
         SchemaEvalConformanceReceiptRefV1,
       maximum_authority: query | proposal}
  tool_projection_binding: AiToolProjectionBindingV1
  authority_profile_binding: GovernedAiProfileBindingV1
  freshness_policy_binding: GovernedAiProfileBindingV1
  authorization_envelope_hash
  created_at, expires_at, revocation_snapshot_ref

AiCallerContextV1
  payload: AiCallerContextPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=ai_caller_context)

ExternalClientSecurityProfileV1
  host_profile_id, host_product_id, host_brand_display_name
  host_identity_kind = local_binary | hosted_service
  host_identity:
    {kind: local_binary,
     exact_version,
     client_binary_ref:
       exact ArtifactRefV1(
         artifact_kind=ai_client_binary, schema_version=1),
     client_binary_sha256,
     os_identity_binding,
     credential_storage_profile_ref, process_child_policy_ref,
     filesystem_policy_ref, network_policy_ref}
     | {kind: hosted_service,
        service_surface_id, service_release_channel,
        service_exact_release_id,
        tenant_workspace_ref, plan_tier_ref, admin_policy_ref,
       endpoint_identity_ref, oauth_or_oidc_subject_ref,
        token_audience, tls_identity_ref}
  supported_transport_profile_bindings[1..64]: GovernedAiProfileBindingV1
  credential_owner
  protocol_versions[]
  required_host_transport_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=host_transport)
  required_schema_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=schema)
  required_eval_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=eval)
  administrative_enablement = enabled | disabled
  issued_at, expires_at, revoked_at

EngineAiHostSecurityProfileV1
  host_profile_id
  host_surface = editor_ai_orchestrator | headless_ai_orchestrator
  engine_build_ref, engine_build_sha256
  process_artifact_ref, process_artifact_sha256
  os_identity_binding, process_child_policy_ref
  filesystem_policy_ref, network_policy_ref
  supported_provider_runtime_profile_bindings[1..64]: GovernedAiProfileBindingV1
  required_provider_tool_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=provider_tool)
  required_schema_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=schema)
  required_eval_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=eval)
  administrative_enablement = enabled | disabled
  issued_at, expires_at, revoked_at

McpTransportSecurityProfileV1
  transport_profile_id
  mcp_protocol_version = 2025-11-25
  transport_kind = local_stdio | streamable_http | secure_mcp_tunnel
  transport:
    {kind: local_stdio,
     server_binary_ref:
       exact ArtifactRefV1(
         artifact_kind=mcp_server_binary, schema_version=1),
     server_binary_sha256,
     os_acl_policy_ref, ipc_endpoint_identity_ref,
     inherited_credential_policy = none,
     maximum_message_bytes, request_timeout_ms,
     rate_limit_per_minute, maximum_concurrent_requests,
     maximum_session_seconds}
    | {kind: streamable_http,
       endpoint_origin, endpoint_identity_ref,
       tls_profile_ref, tls_profile_sha256,
       authorization:
         {kind: oauth_2_1,
          protected_resource_metadata_ref, protected_resource_metadata_sha256,
          authorization_server_metadata_ref, authorization_server_metadata_sha256,
          resource_indicator, token_audience,
          oidc_subject_policy_ref: null | content ref}
         | {kind: approved_private_mtls,
            mtls_profile_ref, private_service_ref, private_service_sha256,
            private_service_activation_ref, private_service_activation_sha256},
       origin_policy_ref, redirect_policy_ref,
       private_address_policy:
         {kind: deny}
         | {kind: approved_private_service,
            private_service_ref, private_service_sha256,
            private_service_activation_ref, private_service_activation_sha256},
       session_binding_policy_ref,
       maximum_message_bytes, request_timeout_ms,
       rate_limit_per_minute, maximum_concurrent_requests,
       maximum_session_seconds}
    | {kind: secure_mcp_tunnel,
       public_endpoint_origin, tunnel_endpoint_identity_ref,
       tunnel_client_artifact_ref:
         exact ArtifactRefV1(
           artifact_kind=mcp_tunnel_client_binary,
           schema_version=1),
       tunnel_client_artifact_sha256,
       local_service_binding_ref,
       tls_profile_ref, tls_profile_sha256,
       protected_resource_metadata_ref, protected_resource_metadata_sha256,
       authorization_server_metadata_ref, authorization_server_metadata_sha256,
       resource_indicator, token_audience,
       origin_policy_ref, redirect_policy_ref,
       private_address_policy = tunnel_local_service_only,
       session_binding_policy_ref, direct_inbound_allowed = false,
       maximum_message_bytes, request_timeout_ms,
       rate_limit_per_minute, maximum_concurrent_requests,
       maximum_session_seconds}
  required_host_transport_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=host_transport)
  issued_at, expires_at, revoked_at

AiAuthorityProfileV1
  authority_profile_id
  authority_class = query_only | proposal_only | managed_source_edit | build_job
  allowed_operation_refs[]
  maximum_risk_class
  required_evidence_policy_refs[]
  required_pre_execution_attestation_kinds[] = [] | [managed_host_session]
  required_post_execution_attestation_kinds[] = [] | [host_execution]
  forbidden_authorities[] = [approval, commit, activation, signing, release]
  issued_at, expires_at, revoked_at

AiExecutionFreshnessPolicyV1
  freshness_policy_id
  max_age_seconds_by_record_kind[]: {record_kind, max_age_seconds}
  clock_source_policy_ref
  issued_at, expires_at, revoked_at

AiConformanceTargetRefV1
  execution_target_profile_ref: TargetProfileRefV1
  artifact_target_profile_ref: null | TargetProfileRefV1

AiToolProjectionBindingV1
  projection_kind = mcp_tools | provider_native_tools
  projection_id, projection_version
  projection_ref:
    exact ArtifactRefV1(
      artifact_kind=ai_tool_projection, schema_version=1)
  projection_sha256: lowercase hex 64
  tool_set_sha256: lowercase hex 64
  operation_ref_set_sha256: lowercase hex 64
  input_schema_set_sha256: lowercase hex 64
  result_schema_set_sha256: lowercase hex 64

AiConformanceFixtureRefV1
  fixture_id, fixture_version
  fixture_polarity = positive_vector | negative_one_factor
  fixture_ref:
    exact ArtifactRefV1(
      artifact_kind=ai_conformance_fixture, schema_version=1)
  fixture_sha256: lowercase hex 64

AiConformanceFixtureSetV1
  fixture_set_id, fixture_set_version
  suite_kind = host_transport | provider_tool | schema | eval
  positive_fixture_count: positive uint32
  positive_fixture_refs[1..256]: AiConformanceFixtureRefV1(
    fixture_polarity=positive_vector)
  negative_fixture_count: positive uint32
  negative_fixture_refs[1..256]: AiConformanceFixtureRefV1(
    fixture_polarity=negative_one_factor)
  fixture_ref_set_sha256: lowercase hex 64
  fixture_set_content_hash: lowercase hex 64

AiConformanceTestCaseV1
  case_id, case_version
  suite_kind = host_transport | provider_tool | schema | eval
  case_polarity = positive_vector | negative_one_factor
  test_contract_ref:
    exact ArtifactRefV1(
      artifact_kind=ai_conformance_test_contract, schema_version=1)
  test_contract_sha256: lowercase hex 64
  fixture_ref: AiConformanceFixtureRefV1
  case_content_hash: lowercase hex 64

AiConformanceTestCaseRefV1
  case_id, case_version
  suite_kind = host_transport | provider_tool | schema | eval
  case_polarity = positive_vector | negative_one_factor
  case_ref:
    exact ArtifactRefV1(
      artifact_kind=ai_conformance_test_case, schema_version=1)
  case_sha256: lowercase hex 64

AiConformanceTestSuiteV1
  suite_id, suite_version
  suite_kind = host_transport | provider_tool | schema | eval
  fixture_set_ref:
    exact ArtifactRefV1(
      artifact_kind=ai_conformance_fixture_set, schema_version=1)
  fixture_set_sha256: lowercase hex 64
  required_case_count: uint32 >= 2
  required_case_refs[2..4096]: AiConformanceTestCaseRefV1
  required_case_set_sha256: lowercase hex 64
  required_negative_case_count: positive uint32
  required_negative_case_refs[1..4096]:
    AiConformanceTestCaseRefV1(case_polarity=negative_one_factor)
  required_negative_case_set_sha256: lowercase hex 64
  suite_content_hash: lowercase hex 64

AiConformanceTestSuiteRefV1
  suite_kind = host_transport | provider_tool | schema | eval
  suite_id, suite_version
  suite_ref:
    exact ArtifactRefV1(
      artifact_kind=ai_conformance_test_suite, schema_version=1)
  suite_sha256: lowercase hex 64
  fixture_set_ref:
    exact ArtifactRefV1(
      artifact_kind=ai_conformance_fixture_set, schema_version=1)
  fixture_set_sha256: lowercase hex 64
  required_case_count: uint32 >= 2
  required_case_set_sha256: lowercase hex 64
  required_negative_case_count: positive uint32
  required_negative_case_set_sha256: lowercase hex 64

AiConformanceCaseExecutionV1
  evaluated_test_suite_ref: AiConformanceTestSuiteRefV1
  required_case_count: uint32 >= 2
  required_case_refs[2..4096]: AiConformanceTestCaseRefV1
  required_case_set_sha256: lowercase hex 64
  executed_case_count: uint32 >= 2
  executed_case_refs[2..4096]: AiConformanceTestCaseRefV1
  executed_case_set_sha256: lowercase hex 64
  passed_case_count: uint32
  passed_case_refs[0..4096]: AiConformanceTestCaseRefV1
  passed_case_set_sha256: lowercase hex 64
  failed_case_count: uint32
  failed_case_refs[0..4096]: AiConformanceTestCaseRefV1
  failed_case_set_sha256: lowercase hex 64
  required_negative_case_count: positive uint32
  required_negative_case_refs[1..4096]:
    AiConformanceTestCaseRefV1(case_polarity=negative_one_factor)
  required_negative_case_set_sha256: lowercase hex 64
  executed_negative_case_count: positive uint32
  executed_negative_case_refs[1..4096]:
    AiConformanceTestCaseRefV1(case_polarity=negative_one_factor)
  executed_negative_case_set_sha256: lowercase hex 64

AiConformanceResultV1
  result:
    {kind: pass,
     case_execution: AiConformanceCaseExecutionV1,
     evidence_ref:
       exact ArtifactRefV1(
         artifact_kind=ai_conformance_result_evidence,
         schema_version=1),
     evidence_sha256: lowercase hex 64,
     diagnostic_refs: []}
    | {kind: fail,
       case_execution: AiConformanceCaseExecutionV1,
       evidence_ref:
         exact ArtifactRefV1(
           artifact_kind=ai_conformance_result_evidence,
           schema_version=1),
       evidence_sha256: lowercase hex 64,
       diagnostic_refs[1..256]}

AiConformanceSubjectTupleV1
  execution_route:
    {kind: standard_external_mcp,
     host_profile_binding:
       GovernedAiProfileBindingV1(
         profile_schema_id=ExternalClientSecurityProfileV1),
     transport_profile_binding:
       GovernedAiProfileBindingV1(
         profile_schema_id=McpTransportSecurityProfileV1),
     provider_runtime_profile_binding: null,
     provider_manifest_binding: null,
     inference_deployment_profile_binding: null,
     model_snapshot_profile_binding: null,
     managed_deployment_identity_ref: null,
     managed_deployment_identity_sha256: null,
     maximum_authority: query | proposal}
    | {kind: managed_external_host,
       host_profile_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=ExternalClientSecurityProfileV1),
       transport_profile_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=McpTransportSecurityProfileV1),
       provider_runtime_profile_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=ProviderRuntimeProfileV1),
       provider_manifest_binding: null,
       inference_deployment_profile_binding: null,
       model_snapshot_profile_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=ModelSnapshotProfileV1),
       managed_deployment_identity_ref:
         exact ArtifactRefV1(
           artifact_kind=managed_provider_deployment_identity,
           schema_version=1),
       managed_deployment_identity_sha256: lowercase hex 64,
       maximum_authority:
         query | proposal | managed_source_edit | build_job}
    | {kind: engine_provider_adapter,
       host_profile_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=EngineAiHostSecurityProfileV1),
       transport_profile_binding: null,
       provider_runtime_profile_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=ProviderRuntimeProfileV1),
       provider_manifest_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=ProviderManifestV1),
       inference_deployment_profile_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=InferenceDeploymentProfileV1),
       model_snapshot_profile_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=ModelSnapshotProfileV1),
       managed_deployment_identity_ref: null,
       managed_deployment_identity_sha256: null,
       maximum_authority: query | proposal}
  tool_projection_binding: AiToolProjectionBindingV1
  authority_profile_binding:
    GovernedAiProfileBindingV1(
      profile_schema_id=AiAuthorityProfileV1)
  conformance_target: AiConformanceTargetRefV1

HostConformanceIdentityV1
  identity:
    {kind: local_binary,
     host_product_id, exact_version,
     client_binary_ref:
       exact ArtifactRefV1(
         artifact_kind=ai_client_binary, schema_version=1),
     client_binary_sha256: lowercase hex 64,
     os_identity_binding}
    | {kind: hosted_service,
       host_product_id, service_surface_id,
       service_release_channel, service_exact_release_id,
       endpoint_identity_ref, tls_identity_ref,
       client_binary_ref: null,
       client_binary_sha256: null}

TransportConformanceIdentityV1
  transport_kind =
    local_stdio | streamable_http | secure_mcp_tunnel
  transport_profile_binding:
    GovernedAiProfileBindingV1(
      profile_schema_id=McpTransportSecurityProfileV1)

HostTransportConformanceReceiptPayloadV1
  receipt_id
  subject_tuple: AiConformanceSubjectTupleV1(
    route=standard_external_mcp | managed_external_host)
  host_identity: HostConformanceIdentityV1
  transport_identity: TransportConformanceIdentityV1
  test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=host_transport)
  result: AiConformanceResultV1
  qualifier_subject_ref
  qualifier_role_ref =
    role.ai-host-transport-conformance-qualifier
  issued_at, expires_at
  revoked_at: canonical UTC | null
  revocation_snapshot_ref

HostTransportConformanceReceiptV1
  payload: HostTransportConformanceReceiptPayloadV1
  signed_record:
    exact MirakanSignedRecordV1(
      purpose=host_transport_conformance)

HostTransportConformanceReceiptRefV1
  exact MirakanSignedRecordRefV1(
    purpose=host_transport_conformance)

ProviderDeploymentModelSnapshotV1
  deployment:
    {kind: managed_external_host,
     provider_runtime_profile_binding:
       GovernedAiProfileBindingV1(
         profile_schema_id=ProviderRuntimeProfileV1),
     managed_deployment_identity_ref:
       exact ArtifactRefV1(
         artifact_kind=managed_provider_deployment_identity,
         schema_version=1),
     managed_deployment_identity_sha256: lowercase hex 64,
     model_snapshot_profile_binding:
       GovernedAiProfileBindingV1(
         profile_schema_id=ModelSnapshotProfileV1)}
    | {kind: engine_provider_adapter,
       provider_runtime_profile_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=ProviderRuntimeProfileV1),
       provider_manifest_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=ProviderManifestV1),
       inference_deployment_profile_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=InferenceDeploymentProfileV1),
       model_snapshot_profile_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=ModelSnapshotProfileV1)}

ProviderToolConformanceReceiptPayloadV1
  receipt_id
  subject_tuple: AiConformanceSubjectTupleV1(
    route=managed_external_host | engine_provider_adapter)
  provider_deployment_model:
    ProviderDeploymentModelSnapshotV1
  observed_tool_projection_binding: AiToolProjectionBindingV1
  test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=provider_tool)
  result: AiConformanceResultV1
  qualifier_subject_ref
  qualifier_role_ref =
    role.ai-provider-tool-conformance-qualifier
  issued_at, expires_at
  revoked_at: canonical UTC | null
  revocation_snapshot_ref

ProviderToolConformanceReceiptV1
  payload: ProviderToolConformanceReceiptPayloadV1
  signed_record:
    exact MirakanSignedRecordV1(
      purpose=provider_tool_conformance)

ProviderToolConformanceReceiptRefV1
  exact MirakanSignedRecordRefV1(
    purpose=provider_tool_conformance)

SchemaEvalConformanceReceiptPayloadV1
  receipt_id
  subject_tuple: AiConformanceSubjectTupleV1
  schema_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=schema)
  eval_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=eval)
  schema_result: AiConformanceResultV1
  eval_result: AiConformanceResultV1
  overall_result = pass | fail
  qualifier_subject_ref
  qualifier_role_ref =
    role.ai-schema-eval-conformance-qualifier
  issued_at, expires_at
  revoked_at: canonical UTC | null
  revocation_snapshot_ref

SchemaEvalConformanceReceiptV1
  payload: SchemaEvalConformanceReceiptPayloadV1
  signed_record:
    exact MirakanSignedRecordV1(
      purpose=schema_eval_conformance)

SchemaEvalConformanceReceiptRefV1
  exact MirakanSignedRecordRefV1(
    purpose=schema_eval_conformance)

McpSessionGrantPayloadV1
  grant_id
  host_profile_binding: GovernedAiProfileBindingV1
  transport_profile_binding: GovernedAiProfileBindingV1
  client_identity:
    {kind: local_binary, client_binary_sha256, os_user_identity_ref,
     channel_binding_sha256}
    | {kind: hosted_service, service_session_subject_ref,
       oauth_or_oidc_subject_ref, token_audience, tls_identity_ref,
       channel_binding_sha256}
  project_id
  read_sensitivity_refs[], allowed_proposal_operation_refs[]
  issuer_subject_ref, issuer_role_ref
  issued_at, expires_at, nonce, revocation_snapshot_ref

McpSessionGrantV1
  payload: McpSessionGrantPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=mcp_session_grant)

ProviderRuntimeProfileV1
  provider_runtime_profile_id
  adapter_kind = official_sdk | raw_protocol | external_host_managed
  adapter:
    {kind: official_sdk, sdk_name, sdk_exact_version,
     sdk_artifact_ref, sdk_artifact_sha256}
    | {kind: raw_protocol, protocol_id, protocol_exact_version,
       request_response_schema_ref, request_response_schema_sha256}
    | {kind: external_host_managed,
       host_profile_binding: GovernedAiProfileBindingV1,
       required_host_execution_attestation = true}
  endpoint_identity_ref
  required_provider_tool_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=provider_tool)
  required_schema_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=schema)
  issued_at, expires_at, revoked_at

ProviderManifestV1
  provider_manifest_id
  provider_runtime_profile_binding: GovernedAiProfileBindingV1
  endpoint_identity_ref
  deployment_profile_binding: GovernedAiProfileBindingV1
  model_snapshot_profile_binding: GovernedAiProfileBindingV1
  role, inference_settings_sha256
  tool_projection_binding: AiToolProjectionBindingV1
  context_limit, output_limit, cost_limit_ref, latency_limit_ref
  privacy_policy_ref, retention_policy_ref, region_ref?, encryption_profile_ref, logging_policy_ref
  required_provider_tool_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=provider_tool)
  required_schema_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=schema)
  required_eval_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=eval)
  fallback:
    {kind: disabled, fallback_provider_manifest_binding: null,
     fallback_max_risk: null,
     fallback_requires_user_confirmation: false}
    | {kind: explicit_manifest,
       fallback_provider_manifest_binding:
         GovernedAiProfileBindingV1(
           profile_schema_id=ProviderManifestV1),
       fallback_max_risk,
       fallback_requires_user_confirmation: true}
  issued_at, expires_at, revoked_at

HostExecutionAttestationPayloadV1
  attestation_id
  caller_context_ref, caller_context_sha256
  host_session_ref, task_id, attempt_id
  managed_deployment_identity_ref:
    exact ArtifactRefV1(
      artifact_kind=managed_provider_deployment_identity,
      schema_version=1)
  managed_deployment_identity_sha256: lowercase hex 64
  task_specification_ref, task_specification_sha256
  authorization_envelope_hash
  input_closure_ref, input_closure_sha256
  freshness_policy_binding: GovernedAiProfileBindingV1
  execution_result:
    {kind: managed_source_edit,
     result_schema_id = urn:mirakan:schema:ai:managed-source-edit-result:v1,
     result_ref, result_sha256}
    | {kind: build_job,
       result_schema_id = urn:mirakan:schema:ai:managed-build-job-result:v1,
       result_ref, result_sha256}
  issued_at, expires_at, revocation_snapshot_ref

HostExecutionAttestationV1
  payload: HostExecutionAttestationPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=host_execution_attestation)

ManagedHostSessionAttestationPayloadV1
  session_attestation_id
  host_profile_binding: GovernedAiProfileBindingV1
  host_session_ref
  provider_runtime_profile_binding: GovernedAiProfileBindingV1
  model_snapshot_profile_binding: GovernedAiProfileBindingV1
  managed_deployment_identity_ref:
    exact ArtifactRefV1(
      artifact_kind=managed_provider_deployment_identity,
      schema_version=1)
  managed_deployment_identity_sha256: lowercase hex 64
  tool_projection_binding: AiToolProjectionBindingV1
  allowed_task_kinds[1..2]: non-empty subset of [managed_source_edit, build_job]
  maximum_authority_classes[1..2]: same set as allowed_task_kinds[]
  freshness_policy_binding: GovernedAiProfileBindingV1
  nonce
  issued_at, expires_at, revocation_snapshot_ref

ManagedHostSessionAttestationV1
  payload: ManagedHostSessionAttestationPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=managed_host_session_attestation)

ManagedSourceEditResultV1
  result_id
  result:
    {kind: completed,
     source_delta_manifest_ref, source_delta_manifest_sha256,
     changed_paths_manifest_ref, changed_paths_manifest_sha256,
     diagnostic_refs[]}
    | {kind: failed,
       source_delta_manifest_ref: null, source_delta_manifest_sha256: null,
       changed_paths_manifest_ref: null, changed_paths_manifest_sha256: null,
       diagnostic_refs[1..]}

ManagedBuildJobResultV1
  result_id
  result:
    {kind: completed,
     build_receipt_kind = package_receipt,
     build_receipt_ref, build_receipt_sha256,
     artifact_manifest_ref, artifact_manifest_sha256,
     diagnostic_refs[]}
    | {kind: failed,
       build_receipt_ref: null, build_receipt_sha256: null,
       artifact_manifest_ref: null, artifact_manifest_sha256: null,
       diagnostic_refs[1..]}

ManagedHostOutputAcceptanceReceiptPayloadV1
  receipt_id
  execution_kind = managed_source_edit | build_job
  caller_context_ref, caller_context_sha256
  task_id, attempt_id
  task_specification_ref, task_specification_sha256
  authorization_envelope_hash
  input_closure_ref, input_closure_sha256
  host_execution_attestation_ref, host_execution_attestation_sha256
  typed_result_ref, typed_result_sha256
  freshness_policy_binding: GovernedAiProfileBindingV1
  acceptance_result = accepted_to_staging | rejected
  accepted_at, expires_at, revocation_snapshot_ref

ManagedHostOutputAcceptanceReceiptV1
  payload: ManagedHostOutputAcceptanceReceiptPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=managed_host_output_acceptance)

InferenceDeploymentProfileV1
  deployment_profile_id, deployment_kind = cloud_endpoint | local_process_ipc
  provider_runtime_profile_binding: GovernedAiProfileBindingV1
  model_snapshot_profile_binding: GovernedAiProfileBindingV1
  deployment:
    {kind: cloud_endpoint,
     endpoint_identity_ref, network_policy_ref}
    | {kind: local_process_ipc,
       process_artifact_ref, process_artifact_sha256,
       runtime_loader_profile_binding: GovernedAiProfileBindingV1,
       model_import_qualification_receipt_ref,
       model_import_qualification_receipt_sha256,
       process_sandbox_profile_ref, gpu_isolation_profile_ref,
       inference_endpoint:
         {kind: os_ipc,
          ipc_endpoint_ref, ipc_auth_profile_ref,
          network_policy = deny_all}
         | {kind: authenticated_loopback,
            loopback_origin, loopback_endpoint_identity_ref,
            ipc_auth_profile_ref,
            network_policy = exact_loopback_only},
       required_ram_bytes, required_vram_bytes,
       cpu_limit, memory_limit_bytes, gpu_memory_limit_bytes,
       disk_limit_bytes, temp_cache_limit_bytes,
       maximum_weight_shard_count, maximum_weight_total_bytes,
       process_child_limit, maximum_batch_size,
       maximum_concurrent_inference, wall_time_limit_ms,
       custom_model_code_allowed = false,
       unsafe_serialization_allowed = false,
       archive_path_escape_allowed = false,
       runtime_network_fetch_allowed = false,
       model_license_acceptance_decision_ref,
       model_license_acceptance_decision_sha256,
       output_reproducibility:
         {kind: not_claimed}
         | {kind: deterministic_claim,
            runtime_artifact_sha256, gpu_profile_ref, gpu_profile_sha256,
            inference_settings_sha256, seed, sampler_profile_ref,
            sampler_profile_sha256, thread_policy_ref, thread_policy_sha256,
            repeatability_receipt_ref,
            repeatability_receipt_sha256}}
  context_limit
  required_provider_tool_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=provider_tool)
  required_schema_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=schema)
  required_eval_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=eval)
  privacy_policy_ref, logging_policy_ref, retention_policy_ref
  output_limit_bytes
  preflight_resource_policy = reject_before_start
  runtime_exhaustion_policy = terminate_process_tree_and_fail
  fallback_policy_owner = provider_manifest_only
  issued_at, expires_at, revoked_at

ModelLicenseAcceptanceDecisionPayloadV1
  decision_id
  local_model_artifact_manifest_binding: GovernedAiProfileBindingV1
  project_id, project_revision
  intended_use_profile_ref, intended_use_profile_sha256
  distribution_plan_ref, distribution_plan_sha256
  approved_use_kinds[]
  weight_redistribution = allowed | forbidden
  output_use = commercial_allowed | noncommercial_only | forbidden
  required_notices_ref, required_notices_sha256
  license_ref, license_sha256
  disposition = approved | rejected
  approver_subject_ref, approval_authority_ref
  issued_at, valid_until, revocation_snapshot_ref

ModelLicenseAcceptanceDecisionV1
  payload: ModelLicenseAcceptanceDecisionPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=model_license_acceptance_decision)

LocalModelArtifactManifestV1
  manifest_id
  model_format_id
  weights_encoding = native_precision | quantized
  weight_shards[]: {artifact_ref, artifact_sha256, size_bytes}
  encoding:
    {kind: native_precision, numeric_format = fp32 | fp16 | bf16}
    | {kind: quantized, quantization_id, quantization_artifact_ref,
       quantization_sha256}
  config_ref, config_sha256
  tokenizer_ref, tokenizer_sha256
  chat_template_ref, chat_template_sha256
  special_token_map_ref, special_token_map_sha256
  license_ref, license_sha256
  provenance_ref, provenance_sha256
  issued_at, expires_at, revoked_at

ModelSnapshotProfileV1
  model_snapshot_profile_id
  model_identity:
    {kind: provider_model_id,
     exact_provider_model_id,
     provider_terms_ref, provider_terms_sha256,
     license_ref, license_sha256,
     provenance_ref, provenance_sha256}
    | {kind: local_weights,
       local_model_artifact_manifest_binding: GovernedAiProfileBindingV1}
  declared_context_limit, effective_context_limit
  supported_schema_profile_refs[1..64]
  supported_tool_projection_bindings[1..64]:
    AiToolProjectionBindingV1
  required_eval_test_suite_ref:
    AiConformanceTestSuiteRefV1(suite_kind=eval)
  issued_at, expires_at, revoked_at

GovernedAiProfileBindingV1
  profile_schema_id
  logical_profile_id
  record_ref, record_sha256
  revision
  issuance_head_ref, issuance_head_sha256, issuance_head_sequence

GovernedAiProfileRecordPayloadV1
  profile_record_id
  profile_schema_id
  profile_ref, profile_sha256
  revision: positive safe integer
  previous_record_ref: null | content-addressed ref
  previous_record_sha256: null | lowercase hex 64
  issuer_subject_ref, issuer_role_ref
  issued_at, revocation_snapshot_ref

GovernedAiProfileRecordV1
  payload: GovernedAiProfileRecordPayloadV1
  signed_record: MirakanSignedRecordV1

GovernedAiProfileHeadPayloadV1
  head_id
  profile_schema_id, logical_profile_id
  current_record_ref, current_record_sha256, current_revision
  sequence: positive safe integer
  previous_head_ref: null | content-addressed ref
  previous_head_sha256: null | lowercase hex 64
  recorded_at, revocation_snapshot_ref

GovernedAiProfileHeadV1
  payload: GovernedAiProfileHeadPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=governed_ai_profile_head)

LocalInferenceLoaderProfileV1
  loader_profile_id
  runtime_artifact_ref, runtime_artifact_sha256
  build_toolchain_ref, build_toolchain_sha256
  supported_model_format_ids[]
  custom_model_code_allowed = false
  unsafe_serialization_allowed = false
  archive_path_policy = normalized_no_escape
  runtime_network_fetch_allowed = false
  built_in_file_or_shell_tools_allowed = false
  mcp_proxy_allowed = false
  issued_at, expires_at, revoked_at

ModelImportQualificationReceiptPayloadV1
  receipt_id
  loader_profile_binding: GovernedAiProfileBindingV1
  local_model_artifact_manifest_binding: GovernedAiProfileBindingV1
  process_artifact_ref, process_artifact_sha256
  import_fixture_refs[]
  freshness_policy_binding: GovernedAiProfileBindingV1
  result = pass | fail
  issued_at, expires_at, revocation_snapshot_ref

ModelImportQualificationReceiptV1
  payload: ModelImportQualificationReceiptPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=model_import_qualification)

LocalInferenceRunReceiptPayloadV1
  run_receipt_id
  runtime_artifact_ref, runtime_artifact_sha256
  model_snapshot_profile_binding: GovernedAiProfileBindingV1
  gpu_profile_ref, gpu_profile_sha256
  inference_settings_sha256, seed
  sampler_profile_ref, sampler_profile_sha256
  thread_policy_ref, thread_policy_sha256
  replay_input_set_ref, replay_input_set_sha256
  freshness_policy_binding: GovernedAiProfileBindingV1
  result:
    {kind: completed,
     exact_response_bytes_sha256,
     diagnostic_refs[]}
    | {kind: failed,
       exact_response_bytes_sha256: null,
       diagnostic_refs[1..]}
  issued_at, expires_at, revocation_snapshot_ref

LocalInferenceRunReceiptV1
  payload: LocalInferenceRunReceiptPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=local_inference_run)

LocalInferenceRepeatabilityReceiptPayloadV1
  receipt_id
  runtime_artifact_sha256
  model_snapshot_profile_binding: GovernedAiProfileBindingV1
  gpu_profile_ref, gpu_profile_sha256
  inference_settings_sha256, seed
  sampler_profile_ref, sampler_profile_sha256
  thread_policy_ref, thread_policy_sha256
  replay_input_set_ref, replay_input_set_sha256
  freshness_policy_binding: GovernedAiProfileBindingV1
  run_receipt_refs[3..]: unique content-addressed refs
  repetition_count = length(run_receipt_refs[])
  exact_response_bytes_sha256
  result = byte_equal | mismatch
  issued_at, expires_at, revocation_snapshot_ref

LocalInferenceRepeatabilityReceiptV1
  payload: LocalInferenceRepeatabilityReceiptPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=local_inference_repeatability)
```

三Conformance Receipt、そのRef、`AiConformanceFixtureSetV1`、`AiConformanceTestCaseV1`、`AiConformanceTestSuiteV1`、`AiConformanceCaseExecutionV1`は上記Fieldだけを持つclosed schemaであり、全Fieldをrequired、unknown Field、Field省略、empty object、hash-only refを拒否する。`AiToolProjectionBindingV1.projection_sha256`は`projection_ref`が解決する完成bytes、`tool_set_sha256`はTool名／version／Operation ref／input Schema ref／result Schema ref／annotationをcanonical順にした完全集合、Operation／input／result set hashは各部分集合のRFC 8785 JCS SHA-256へ一致させる。

Fixture refはID／version／polarity／型付きArtifact ref／完成bytes hash、Case refはID／version／suite kind／polarity／型付きArtifact ref／完成bytes hashを全て保存する。Fixture Setのpositive／negative配列、Suiteのrequired／required-negative配列はそれぞれrefの全FieldをUTF-8 byte順へstrict sortし、重複を拒否する。`fixture_ref_set_sha256`、`required_case_set_sha256`、`required_negative_case_set_sha256`はASCII domain `MIRAKAN_AI_CONFORMANCE_{FIXTURE|CASE|NEGATIVE_CASE}_SET_V1`と当該完成配列のRFC 8785 JCS bytesを`uint32_be` length framingして計算する。Fixture／Case／Suite自身のcontent hashは同様に`MIRAKAN_AI_CONFORMANCE_{FIXTURE_SET|TEST_CASE|TEST_SUITE}_V1`と自己hash Fieldを除くclosed recordから計算し、隣接Artifact hashとbyte equalityにする。Suiteは少なくとも一positive Caseと一negative Caseを持ち、`required_negative_case_refs[]`を`required_case_refs[]`の`case_polarity=negative_one_factor` exact部分集合、各CaseのFixtureを同SuiteのFixture Set exact entry、Case／Fixture／Suiteの`suite_kind`を同値にする。

`AiConformanceTestSuiteRefV1`のsuite／fixture ref、ID、version、kind、両count／set hashは解決したSuite／Fixture Setとbyte equalityにする。suite ID、version、kind、Case集合、Fixture集合の一つでも違えば別suiteであり、表示名、最新版、Provider推奨値へfallbackしない。subject tuple内の全Profileが同じsuite kindについて持つ`required_*_test_suite_ref`は互いにbyte equalityで、対応Receiptのsuite refおよび`AiConformanceResultV1.case_execution.evaluated_test_suite_ref`と一致させる。external routeのHost／Transport suiteはExternal Host＋MCP Transport、managed routeのProvider suiteはProvider Runtime、Schema suiteはExternal Host＋Provider Runtime、Eval suiteはExternal Host＋Model、Engine routeのProvider／Schema suiteはEngine Host＋Provider Runtime＋Manifest＋Deployment、Eval suiteはEngine Host＋Manifest＋Deployment＋Modelで一致させ、別version／hashを混成しない。Profileは必要suite定義だけを参照するreceipt-free declarationであり、ReceiptをProfileへ逆参照させない。依存順は`Profile／Head＋Tool Projection＋Fixture Set＋Case＋Suite＋Target -> Conformance Receipt -> Caller Context`の一方向だけである。

各`receipt_id`は同Fieldを除く完成payloadのJCS hashから、Host／Transport=`urn:mirakan:host-transport-conformance:sha256:<lowercase-hex>`、Provider／Tool=`urn:mirakan:provider-tool-conformance:sha256:<lowercase-hex>`、Schema／Eval=`urn:mirakan:schema-eval-conformance:sha256:<lowercase-hex>`として導出する。全Receiptで`issued_at < expires_at`、`revoked_at`は省略不能な`canonical UTC | null`、current利用は`revoked_at=null`かつcurrent revocation snapshot一致だけを許す。署名済みwrapperは`subject_sha256=SHA-256(JCS(payload))`、Signer subject／Role、singleton purpose Key、`issued_at`、revocation snapshotをpayloadへbyte-exact一致させる。期限延長再包装、別purposeの有効署名、Qualifier自己申告、過去Receiptのcurrent再利用を拒否する。

`AiConformanceSubjectTupleV1`はrouteごとのfull subject tupleである。全`GovernedAiProfileBindingV1`は発行時の完成Profile Recordと署名済みissuance Headへ解決し、Receipt発行時にcurrent Headでなければならない。`HostTransportConformanceReceiptV1.host_identity`は解決したExternal Host ProfileのHost product、exact version、local binary ref／hashまたはhosted releaseとbyte equalityにする。`transport_identity.transport_profile_binding`はsubject tupleのTransport bindingとbyte equalityにし、解決した完成`McpTransportSecurityProfileV1`がMCP protocol version、binary／endpoint、ACL、OAuth metadata、mTLS private-service Activation、origin／redirect／private-address／session policy、rate／timeout／concurrency上限を一つのclosed transport branchとして所有する。`transport_identity.transport_kind`はそのbranch discriminatorと一致させ、Receipt側に別の不完全なauthentication projectionを作らない。local Hostだけがbinaryをnon-null、hosted Hostだけがbinaryをexact nullにし、hosted releaseを架空binary hashへ変換しない。

`engine_provider_adapter`ではHost／Transport Receiptを作らず、Provider／Tool ReceiptとSchema／Eval Receiptの共通subject tupleにある`EngineAiHostSecurityProfileV1` bindingがEngine build ref/hash、process artifact ref/hash、OS identity、sandbox policy、issuance Headを束縛する。Provider Receiptの`provider_deployment_model`はsubject tupleとbyte equalityにする。`managed_external_host`ではProvider Runtime／Modelをnon-null、Provider Manifest／Inference Deploymentをnullにし、登録Brokerがattestする`managed_deployment_identity_ref/hash`をCaller Context payload、subject tuple、Provider／Tool Receipt snapshot、Managed Host Session Attestation、Host Execution Attestationで全てnon-nullかつbyte equalityにする。同じProvider Runtime／Modelでもmanaged deployment identityが一byte違えば別subjectでありReceipt、Session、Execution結果を流用しない。このProvider Runtimeは`adapter.kind=external_host_managed`で、その`host_profile_binding`をsubject tupleのExternal Host bindingへ一致させる。`standard_external_mcp | engine_provider_adapter`ではCaller Context／subject tupleのmanaged deployment identity両Fieldをexact nullにする。Engine routeではProvider Runtime／Manifest／Inference Deployment／Modelを全non-nullにし、Manifestが指すRuntime／Deployment／Model、Deploymentが指すRuntime／Modelをsubject tupleへ一致させる。`observed_tool_projection_binding`もsubject tupleのTool Projectionとbyte equalityにする。

`AiConformanceCaseExecutionV1`はSuiteのrequired Case全量をそのまま保存する。`required_case_refs/set hash/count`は解決したSuite、`executed_case_refs/set hash/count`はrequired集合、`required_negative_case_refs/set hash/count`と`executed_negative_case_refs/set hash/count`はSuiteのrequired-negative集合へそれぞれexact set equalityにする。`passed_case_refs[]`と`failed_case_refs[]`はexecuted集合の互いに素なexact partitionで、各countとset hashを実配列から再計算する。未実行、表外、duplicate、polarity変更、CaseまたはFixture hash差、countだけの一致を有効Resultにしない。

`AiConformanceResultV1.result.kind=pass`は`passed_case_refs=required_case_refs`、`failed_case_refs=[]`、全negative Case実行、Diagnostic exact `[]`の場合だけ許す。`kind=fail`は`failed_case_refs[]`とDiagnosticを各一件以上必須にし、passへ補正しない。Schema／Eval Receiptの`overall_result=pass`は`schema_result.result.kind=pass`かつ`eval_result.result.kind=pass`の論理積であり、一方のfail、未実行Case、部分Suite、自然言語result、欠落Evidenceをpassへ補正しない。`standard_external_mcp`のEvalは外部Hostが公開するTool Schema、typed result、error、cancellation、proposal-only境界のblack-box conformanceだけを評価し、標準MCPでattestできない上流Provider／Modelの品質・identityを主張しない。Provider／Model attributionを要するEvalは`managed_external_host | engine_provider_adapter`のfull tupleだけで発行する。`AiConformanceTargetRefV1.execution_target_profile_ref`は試験を実行したexact Host／runtime Target、`artifact_target_profile_ref`はToolまたはBuild結果がTarget固有のときだけnon-nullにする。Windows Editor Host上でAndroid／Appleを評価する場合は両Refを分離し、Editor Receiptをartifact Target Receiptへ流用しない。

Caller Context発行時、GatewayはContextから`AiConformanceSubjectTupleV1`を再投影し、各Receipt payloadのsubject tupleとbyte equalityにする。具体的にはHost／全Profileのschema・logical ID・Record ref/hash・revision・issuance Head ref/hash/sequence、Transport Profile binding、Provider／Deployment／managed deployment identity／Model、Tool Projectionと全set hash、Authority、execution／artifact Target、route ceilingを一Fieldずつ比較する。`standard_external_mcp`はHost／Transport Receipt＋Schema／Eval Receiptをexact非null、Provider／Tool Receiptをnullにする。`managed_external_host`は三Receiptすべてをexact非null、`engine_provider_adapter`はProvider／Tool Receipt＋Schema／Eval Receiptをexact非null、Host／Transport Receiptをnullにする。Receipt同士のsubject tupleもbyte-equalでなければならない。Context発行後もTool一覧取得直前と各dispatch直前に全Profile Head、Receipt expiry／revocation、Context tupleをread-backし、Head drift、Caller ContextだけのHost／managed deployment／Model／Target差替え、古いReceiptと新Profileの混成をfail closedにする。

`McpSessionGrantPayloadV1.expires_at`は`issued_at`から最大60分である。`grant_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:mcp-session-grant:sha256:<lowercase-hex>`として導出し、完成payloadを`role.mcp-session-grant-service`のsingleton-purpose Keyで署名する。signed recordのsubject、issuer subject／Role、issued_at、revocation snapshotをpayloadとexact一致させ、grantのoperation集合はQuery／Proposalだけに制限する。optional Fieldはkind／transportのtagged branchだけで必要条件を閉じ、local binaryをhosted serviceへ捏造またはその逆をしない。`ModelSnapshotProfileV1.model_identity`の固有Fieldは、`provider_model_id`なら`exact_provider_model_id`、`local_weights`ならcurrent `local_model_artifact_manifest_binding`だけを必須にして相互混在を拒否する。Local manifestは全weight shard、config、tokenizer、chat template、special-token map、license、provenanceとencoding branchをhash closureにし、native FP16／BF16へ架空quantizationを要求しない。identity正本はModel Snapshot＋Local manifestだけであり、`InferenceDeploymentProfileV1`へ複写しない。DeploymentとProvider Manifestの`model_snapshot_profile_binding`はschema／logical ID／record／revision／issuance Headをbyte-exact一致させる。`local_process_ipc`ではprocess artifact、`local_weights` Snapshot、認証済みOS IPCまたはloopback、local resource上限をすべて必須にする。Host display name、Provider名、Model family名は表示metadataであり、Transport、Tool Schema、Authorityを推測する入力にしない。

`InferenceDeploymentProfileV1.deployment.model_import_qualification_receipt_ref`は、receipt-freeな`LocalModelArtifactManifestV1`、`LocalInferenceLoaderProfileV1`、process artifactおよびimport fixture closureを先に固定して発行した`ModelImportQualificationReceiptV1`だけを参照する下流policy projectionである。Receipt payload／subjectはDeployment Profile、その`GovernedAiProfileRecordV1`、issuance Head、Binding、またはそれらのhashを含めてはならず、Deployment ProfileをReceipt subjectへ逆参照する循環をValidatorは拒否する。依存順は`local model／loader／process／fixture closure -> ModelImportQualificationReceiptV1 -> InferenceDeploymentProfileV1 -> GovernedAiProfileRecordV1／Head`の一方向だけとする。

`AiCallerContextV1`はGatewayだけが`role.ai-gateway-context-publisher`／singleton purpose `ai_caller_context`で発行する短命signed contextである。`caller_context_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:ai-caller-context:sha256:<lowercase-hex>`として導出し、signed recordのsubject hash、issued_at=`created_at`、revocation snapshotをexact一致させる。`created_at < expires_at`、current Freshness Policyの`record_kind=ai_caller_context`はexact一件かつ`max_age_seconds <= 600`、`expires_at=min(created_at+max_age_seconds, Authorization／当該branchのGrantまたはSession Attestation／全non-null profile・Receiptのexpiry)`を必須にする。各non-null binding、Grant、Conformance Receipt、Authorization Envelopeを発行時とTool実行直前にread-backし、`expires_at <= evaluation_time`、current Head drift、revocation、別Project／subject／channelでは拒否する。

`host_profile_binding.profile_schema_id`は`standard_external_mcp | managed_external_host`で`ExternalClientSecurityProfileV1`、`engine_provider_adapter`で`EngineAiHostSecurityProfileV1`だけを許す。外部2 routeではHost Profileが列挙するcurrent MCP Transport bindingとContextのbindingをbyte-exact一致させる。Engine routeではEngine Hostが列挙するcurrent Provider Runtime bindingとContextを一致させ、MCP Transportを要求または捏造しない。`standard_external_mcp`はProvider Runtime、Provider Manifest、Inference Deployment、Model、managed deployment identity、Managed Session Attestation、Provider／Tool Receiptを全てnull、Host／Transport ReceiptとSchema／Eval Receiptをnon-nullとし、MCP initialize由来のProvider／Model名はunattested metadataだけへ隔離する。`managed_external_host`だけはProvider／Model binding、managed deployment identity、三Conformance Receipt、実行前`ManagedHostSessionAttestationV1` ref／hashを全non-null、同一Host session／`tool_projection_binding`へ閉じる。`engine_provider_adapter`はmanaged deployment identityをnullにし、Provider Manifestが指すProvider Runtime、Inference Deployment、Model Snapshot、Tool ProjectionをContextとbyte-exact一致させ、Provider／Tool ReceiptとSchema／Eval Receiptをnon-nullにする。cloud direct APIとfirst-party local IPCは同じEngine routeのDeployment branchであり、MCP Transport Profileを流用しない。branch間Field流用、裸Context、caller自己署名を拒否する。

route別session bindingもclosedにする。`standard_external_mcp`はfresh `McpSessionGrantV1` ref／hashを両non-nullで必須、`engine_provider_adapter`は両方null、`managed_external_host`は両方nullかつ実行前`ManagedHostSessionAttestationV1`を専用Broker session bindingとして必須にする。effective operation集合は常に `route ceiling ∩ current AiAuthorityProfile.allowed_operation_refs[] ∩ signed TaskAuthorizationEnvelope.payload.allowed_operations[] ∩ Server Policy`、standard MCPではさらに`∩ McpSessionGrant.payload.allowed_proposal_operation_refs[]`である。managed routeではSession Attestationの`allowed_task_kinds[]`／authority classも積集合へ加える。いずれかのmissing、tuple差、空でない超過、より強いcaller申告、Profileの`forbidden_authorities[]`欠落をTool公開前と実行直前に拒否する。

read-time `effective_route_state`はProfile Fieldではなく、`supported | proposal_only | not_activated`のclosed projectionとしてGatewayだけが次の順で導出する。(1) routeが参照する全Profile bindingをcurrent signed `GovernedAiProfileRecordV1`／issuance Headへ解決し、Host Profileの`administrative_enablement=enabled`、他Profileのbranch整合を検査する。(2) §8.3のroute別full tuple、required Grant／Session Attestation、exact Tool Projection／Authority／Targetをset／byte equalityで検査する。(3) routeが要求するexact二または三Conformance Receiptをすべてcurrent `result=pass`として解決し、Suite全Case、freshness、expiry、revocation、Profile Head、subject tupleを再検証する。(4) Product／Capability ActivationとTask Authorizationを含む§7～§8のdispatch predicateを再評価する。全条件が成立した`standard_external_mcp`だけを`proposal_only`、成立した`managed_external_host | engine_provider_adapter`をroute ceiling内の`supported`、それ以外を`not_activated`とする。`administrative_enablement=disabled`、Profile／Receipt一件のmissing／fail／stale／revoked、tuple差は必ず`not_activated`であり、Profile revisionの自己申告、Host brand、過去Receiptから`effective_route_state`を復元しない。Managed route不成立から`proposal_only`へ降格せず、別のcurrent standard route full tupleから新しいCaller Contextを発行した場合だけ、その別routeを独立に`proposal_only`とする。

Attestation条件は因果順に評価する。query／proposal Authorityはpre／post集合を両方empty exact set、managed edit／build Authorityは`required_pre_execution_attestation_kinds=[managed_host_session]`、`required_post_execution_attestation_kinds=[host_execution]` exact setとする。Tool公開前・実行直前はpre集合だけを検査し、処理後にBrokerがtyped resultを得てからHost Execution Attestationを発行する。Staging受入れ、`GenerationReceiptV1`完成、`ManagedHostOutputAcceptanceReceiptV1`発行の各時点ではpre Attestationを再検証したうえでpost集合も必須にする。post Attestationを実行前Contextへ埋め込むこと、post欠落のresultをStagingへ受け入れること、R4判断やCommitをallowlistへ追加することを拒否する。

次のprofile payloadは必ず`GovernedAiProfileRecordV1`で署名する。`ExternalClientSecurityProfileV1`と`EngineAiHostSecurityProfileV1`の`administrative_enablement`は署名済みgovernance inputであり、`disabled`がrouteを停止する上限switch、`enabled`が後続Conformance評価を許可するだけである。Profile自身は`pass | fail`、`supported | proposal_only | not_activated`、またはOperation可用性を宣言しない。

| profile schema | exact purpose | issuer Role |
|---|---|---|
| `ExternalClientSecurityProfileV1` | `external_client_security_profile` | `role.ai-profile-security.r4` |
| `EngineAiHostSecurityProfileV1` | `engine_ai_host_security_profile` | `role.ai-profile-security.r4` |
| `McpTransportSecurityProfileV1` | `mcp_transport_security_profile` | `role.ai-transport-security.r4` |
| `AiAuthorityProfileV1` | `ai_authority_profile` | `role.ai-authority-policy.r4` |
| `AiExecutionFreshnessPolicyV1` | `ai_execution_freshness_policy` | `role.ai-execution-freshness-policy.r4` |
| `ProviderRuntimeProfileV1` | `provider_runtime_profile` | `role.ai-provider-runtime-owner.r4` |
| `ProviderManifestV1` | `provider_manifest` | `role.ai-provider-profile-owner.r4` |
| `InferenceDeploymentProfileV1` | `inference_deployment_profile` | `role.ai-deployment-profile-owner.r4` |
| `ModelSnapshotProfileV1` | `model_snapshot_profile` | `role.ai-model-profile-owner.r4` |
| `LocalModelArtifactManifestV1` | `local_model_artifact_manifest` | `role.ai-model-artifact-publisher` |
| `LocalInferenceLoaderProfileV1` | `local_inference_loader_profile` | `role.ai-local-runtime-security.r4` |

各logical profile IDごとに初回revision 1／previous=null、以後current完成wrapper ref／hashとexact `N+1`を持つrecord chainを作る。`profile_record_id`は同Fieldを除く完成`GovernedAiProfileRecordPayloadV1`のJCS hashから`urn:mirakan:governed-ai-profile-record:sha256:<lowercase-hex>`として導出する。`signed_record.subject_sha256`はprofile単体でなくIDを含む完成Record payload JCS hash、Signer subject／Role、issued_at、revocation snapshot、上表purposeをouter payloadとexact一致させる。`profile_ref/hash`は完成profile bytesへ解決し、Local Model Artifactを含む全profileで`profile.issued_at=record.payload.issued_at`、profile expiry／revocation Fieldとcurrent stateを検証する。全Governed profileの`revoked_at`は省略不能な`canonical UTC | null`で、current利用は`null`だけを許す。省略、空文字、sentinel、non-null、profileとcurrent revocation snapshotの不一致を拒否する。これにより署名済みprofileを維持したrevision／previous／issuer差替えを拒否する。

Record chainに加えてlogical profile IDごとの完成`GovernedAiProfileHeadV1`を必須にする。genesisはsequence 1／previous=null／current revision 1、後続はcurrent完成Headをprevious、sequenceとrevisionをexact `N+1`にし、`service.ai-profile-head-publisher`／`role.ai-profile-head-publisher`／singleton purpose `governed_ai_profile_head`で署名してexpected previousへのsingle CASを行う。`head_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:governed-ai-profile-head:sha256:<lowercase-hex>`として導出する。Bindingは発行時Head wrapper ref／hash／sequenceと、そのHeadが指すschema、logical ID、record ref／hash、revisionをexact固定する。新規実行、Authorization、Staging受入れ、Promotionでは発行時Headがcurrent Headと同一でなければ`stale`として拒否する。履歴監査では発行時Headから署名chainを再生し、当時有効なら`authentic_but_stale`として過去Receiptを保持するが、新規権限またはcurrent性能主張へ再利用しない。Head更新だけで過去Receiptを改竄扱いまたは削除せず、filesystem `latest`、mutable logical ID、古いbindingのcurrent利用を拒否する。unknown／expired／revoked Role、assignment、Key、branch、gap、同内容revision bumpも拒否する。

`McpSessionGrantV1`、`AiCallerContextV1`、`ManagedHostSessionAttestationV1`、`HostExecutionAttestationV1`は各専用Role／purposeで別途署名し、同じprofile chainへ混ぜない。Managed Host Sessionは登録Brokerの`role.managed-host-session-attestor`／purpose `managed_host_session_attestation`、post-executionは`role.managed-host-execution-attestor`／purpose `host_execution_attestation`を使い、payload JCS subject、型別issued_at、revocation snapshotをexact一致させる。

Profile外のsecurity recordは次のexact署名行だけを許す。全wrapperで`subject_sha256=SHA-256(JCS(payload))`、Signer subject／Role、singleton purpose Key、型別issued_at、payloadのrevocation snapshot、発行時と評価時のcurrent revocationを検証する。

| record | exact purpose | issuer Role | signed time | 利用可能result |
|---|---|---|---|---|
| `McpSessionGrantV1` | `mcp_session_grant` | `role.mcp-session-grant-service` | payload `issued_at` | expiry内のquery／proposal grant |
| `AiCallerContextV1` | `ai_caller_context` | `role.ai-gateway-context-publisher` | payload `created_at` | route/profile/envelope積集合がnon-empty |
| `AiTaskAttemptReservationV1` | `ai_task_attempt_reservation` | `role.ai-gateway-task-attempt-reservation` | payload `reserved_at` | current terminal Head、上限内count、current Context／Authorization |
| `AiTaskRepairAttemptHeadV1` | `ai_task_repair_attempt_head` | `role.ai-gateway-task-repair-attempt-head` | payload `recorded_at` | expected-previous CAS成功、closed reserved／recorded／aborted state |
| `StandardExternalProposalReceiptV1` | `standard_external_proposal_receipt` | `role.ai-gateway-standard-external-proposal-receipt` | payload `received_at` | current reserved Head／Context／Grant、typed Proposal |
| `HostTransportConformanceReceiptV1` | `host_transport_conformance` | `role.ai-host-transport-conformance-qualifier` | payload `issued_at` | `pass`かつcurrent Host／Transport／Target tuple |
| `ProviderToolConformanceReceiptV1` | `provider_tool_conformance` | `role.ai-provider-tool-conformance-qualifier` | payload `issued_at` | `pass`かつcurrent Provider／Deployment／Model／Tool tuple |
| `SchemaEvalConformanceReceiptV1` | `schema_eval_conformance` | `role.ai-schema-eval-conformance-qualifier` | payload `issued_at` | Schema／Eval両方`pass`かつ同じfull tuple |
| `ManagedHostSessionAttestationV1` | `managed_host_session_attestation` | `role.managed-host-session-attestor` | payload `issued_at` | allowed task kindだけ |
| `HostExecutionAttestationV1` | `host_execution_attestation` | `role.managed-host-execution-attestor` | payload `issued_at` | typed resultの完成branch |
| `ManagedHostOutputAcceptanceReceiptV1` | `managed_host_output_acceptance` | `role.managed-host-output-acceptor` | payload `accepted_at` | `accepted_to_staging`かつtyped result=`completed`だけ |
| `ModelImportQualificationReceiptV1` | `model_import_qualification` | `role.ai-model-import-qualifier` | payload `issued_at` | `pass`だけ |
| `LocalInferenceRunReceiptV1` | `local_inference_run` | `role.ai-local-inference-runner` | payload `issued_at` | result kind=`completed`だけ |
| `LocalInferenceRepeatabilityReceiptV1` | `local_inference_repeatability` | `role.ai-repeatability-runner` | payload `issued_at` | `byte_equal`だけ |
| `ModelLicenseAcceptanceDecisionV1` | `model_license_acceptance_decision` | `role.model-license-acceptance.r4` | payload `issued_at` | `approved`だけ |

三Conformance Qualifier RoleはProfile外security qualification wrapper専用であり、MCD Operation実行、Operation Receipt発行、Approval、Commit、Activation、Signing、Releaseを許可しない。currentの三Qualifier Identity／Role Registry row／Assignment／Key／Receipt instance集合はそれぞれexact `[]`で、schema上のRole literalだけから署名Authorityを合成しない。将来は使用するrouteのProfile／Suite／Fixture／Qualifier Trust closure／三Receipt schemaを一つの承認済みsecurity qualification transactionでmaterializeする。`OperationReceiptSignerPolicyV1.entries[]`、active／conditional／planning Operation集合、各Operation Signer destinationへ三Roleを追加せず、既存のOperation件数とSigner Policy件数を変更しない。

Freshness-bearing recordはcurrent `AiExecutionFreshnessPolicyV1` bindingの該当`record_kind`が一件だけ存在し、`expires_at = min(issued_at + max_age_seconds, 全参照profile／Receiptのexpires_at)`を満たす。Acceptance Receiptでは`issued_at`を`accepted_at`と読む。policy missing／duplicate、caller指定max age、expiry延長再包装、current Head差、source expiry／revocationを拒否する。License Decisionは同式でなくpayloadの`issued_at <= evaluation_time < valid_until`とProject／license driftを継続評価する。

| Host profile | 許可Transportの扱い | 対応表示条件 |
|---|---|---|
| Editor | direct Provider API、local STDIO MCP、またはActivation済みStreamable HTTP | Engine ProviderならProvider／Tool＋Schema／Eval、外部HostならHost／Transport＋Schema／Evalのtyped Receiptと製品内Policyが有効 |
| hosted ChatGPT Work | pluginが束ねるremote MCP Toolを使う。local Codex `config.toml`、local STDIO、Codex command menuを読み込まず、private network／developer machineは承認済みremote serviceまたはSecure MCP Tunnel経路だけ | plan／workspace条件、admin policy、exact plugin／remote Host／Transport＋Schema／Eval Receiptが有効 |
| ChatGPT desktop appのCodex host | Codex hostとしてlocal STDIO MCPまたはStreamable HTTPを使い、Codex CLI／IDEと同じCodex MCP config layerを共有できる。hosted ChatGPT Work plugin接続とは別Host Profile | exact desktop Codex host version／binary／Host／Transport＋Schema／Eval Receiptが有効 |
| Codex CLI／Codex IDE | local STDIO MCPまたはStreamable HTTP | exact client version／binary／Host／Transport＋Schema／Eval Receiptが有効 |
| Claude Desktop／Claude Code | Profileに列挙したlocal STDIO MCPまたはStreamable HTTP | exact client version／binary／Host／Transport＋Schema／Eval Receiptが有効 |
| Cursor | Profileに列挙したlocal STDIO MCPまたはStreamable HTTP | exact client version／binary／Host／Transport＋Schema／Eval Receiptが有効 |

この表は許容するprofile classであり、現時点の具体的なHost／version／binary／Transport組合せを`supported`と宣言しない。初期`ExternalClientSecurityProfileV1`、`EngineAiHostSecurityProfileV1`、`HostTransportConformanceReceiptV1`、`ProviderToolConformanceReceiptV1`、`SchemaEvalConformanceReceiptV1`、`AiConformanceFixtureSetV1`、`AiConformanceTestCaseV1`、`AiConformanceTestSuiteV1`、managed Host attestor、first-party local runtime、三routeの`AiCallerContextV1`のcurrent materialized instanceはそれぞれexact 0件である。Phase 4のAI CoreはEngine build／process／OS policyとProvider tupleを束縛したEngine Host Profileを、Phase 5 `wp.product.external-agent`は少なくとも一つのexact External Host＋MCP Transport＋Tool Projection＋proposal-only Authority組合せをそれぞれmaterializeし、positive／negative Conformanceを通す。ChatGPT、Codex、Claude、Cursor等の名前だけでは完了しない。`managed_source_edit | build_job` routeはProduct Future Registryのexact `future.capability.managed-external-host-execution`、`planning_state=planning_only`にだけ属する。そのFutureのActive Definition migrationで専用Work Package、Threat Model、Broker、session／execution attestor、Engine Build Receipt closureを同時登録し、通常lifecycleで承認・Qualificationするまで`not_activated`であり、MVPのstandard external MCP proposal laneへ混ぜない。これは別entry `future.capability.first-party-local-inference`のfirst-party local runtime計画と独立であり、一方のEvidence／Activationを他方へ流用しない。

typed Receipt不在、期限切れ、失効、version／binary／Transport endpoint／auth／Provider／Deployment／Model／Tool set／Schema／Eval／Target／Profile Head差では`supported`と表示しない。`proposal_only`を表示できるのは、失効したManaged Contextを降格した場合ではなく、別のcurrent `standard_external_mcp` Host／Transport／Tool／proposal-only Authority Profile、MCP Grant、Host／Transport＋Schema／Eval ConformanceからGatewayが新Caller Contextを発行した場合だけである。その標準経路も成立しなければ`not_activated`とする。Callerの最大Authorityは`query | proposal | managed_source_edit | build_job`のclosed setであり、Approval、Commit、Activation、Signing、ReleaseはどのHost、Provider、Modelにも付与しない。

`ExternalClientSecurityProfileV1.host_product_id`はsurface／modeを含むexact IDとし、hosted ChatGPT Work、ChatGPT desktop appのCodex host、Codex CLI、Codex IDEを別Profileとして発行する。desktop applicationという同じ容器、同じaccount、同じvendorであることを根拠にTransport、設定、権限、Conformance Receiptを共有しない。hosted ChatGPT Work用remote plugin ToolをCodex local MCPとして、またはCodex local MCP configをhosted ChatGPT Work pluginとして自動投影しない。

`McpTransportSecurityProfileV1`はtransport kindごとの全Fieldをrequiredにし、他branch Fieldをunknownとして拒否する。local STDIOはbinary hash、OS ACL、IPC identity、credential非継承を、Streamable HTTPはexact origin／TLS／MCP OAuth 2.1 Protected Resource Metadata／Authorization Server Metadata／resource indicator／token audience／redirect／private-address／session bindingを、Secure MCP Tunnelはpublic endpointとexact tunnel client artifact、local service binding、direct inbound禁止を検証する。OIDC subjectはOAuth 2.1 branchの補助identity policyであってOAuth authorizationを置換せず、mTLS-onlyはActivation済みprivate service branchに限定する。`approved_private_service`は別Threat Model、service ref／hash、Activation ref／hashがある場合だけ許し、名前がprivate、localhost、tunnelであることを根拠にしない。全branchでexact MCP protocol version、message／timeout／rate／concurrency／session上限を強制する。External Client Host Profileの`supported_transport_profile_bindings[]`は各current完成Governed MCP Transport Profile recordへ解決し、Profile／Transport／`HostTransportConformanceReceiptV1`／`SchemaEvalConformanceReceiptV1`のexpiry・revocation・hash差で接続前にfail closedにする。Engine Host ProfileはMCP Transportでなくcurrent Provider Runtime bindingを列挙し、Engine build／process artifact／OS／filesystem／network policy差でEngine routeをfail closedにする。

`future.capability.managed-external-host-execution`が`planning_only`から承認済みActivationへ進み、専用Work Packageがmaterializeされた場合だけ、Managed HostのSource edit／Build出力をそれぞれclosed `ManagedSourceEditResultV1`／`ManagedBuildJobResultV1`として保存し、処理完了後の`HostExecutionAttestationV1` ref／hashを`ManagedHostOutputAcceptanceReceiptV1`のrequired Fieldにする。`attestation_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:host-execution-attestation:sha256:<lowercase-hex>`、session attestation IDは同様に`urn:mirakan:managed-host-session-attestation:sha256:<lowercase-hex>`で導出する。flat result objectの`result_id`は同Fieldを除く完成object JCS hashからSource edit=`urn:mirakan:managed-source-edit-result:sha256:<lowercase-hex>`、Build=`urn:mirakan:managed-build-job-result:sha256:<lowercase-hex>`、Acceptance payloadの`receipt_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:managed-host-output-acceptance-receipt:sha256:<lowercase-hex>`として導出する。post-execution payloadのCaller Context、managed deployment identity、Task Specification、Authorization Envelope、attempt、Input closure、result branchのkind／schema ID／ref／hashはSession Attestation、Provider／Tool Receipt、Acceptance Receiptの対応入力とbyte-exactでなければならず、Broker署名とcurrent revocationを検証後にだけStagingへ受理する。Build familyとManaged Host routeの双方が将来Activationされた場合だけ、completed Buildの`build_receipt_ref/hash`はCore ArchitectureのEngine-owned `PackageReceiptV1`完成wrapperで、operation ID=`operation.build.request_package`、purpose=`operation_receipt:operation.build.request_package`、Build Gateway signer、result=`succeeded`、同じTask Specification／Authorization／Project revision／Candidate／Target／Toolchain／artifact manifestを必須にする。Host／Broker AttestationはprovenanceでありBuild Receiptを代替しない。Receiptは`service.managed-host-output-acceptor`／singleton purpose `managed_host_output_acceptance`で署名する。standard external MCP、Engine Provider Adapter、proposal-only経路はManaged Host Session／Execution Attestationを両方持てず、空Attestation、managed deployment identity差、事前署名result、kindとresult schema不一致、別attempt／別Input Attestationを拒否する。

### 8.4 Local inference境界

Local inferenceは`InferenceDeploymentProfileV1.deployment_kind=local_process_ipc`のclosed branchとしてcloud endpointと分離する。起動前にcurrent `model_snapshot_profile_binding`からSnapshot／Profile Record／issuance Headをread-backし、`model_identity.kind=local_weights`、current Local Manifest binding、全weight shard closureと選択encoding branch、config／tokenizer／chat template／special-token map、license／provenance、expiry／revocation、Eval／Schema／Tool conformanceがすべてcurrentであることを検証する。Deploymentの`process_artifact_ref/hash`はLoaderの`runtime_artifact_ref/hash`とImport Qualificationのprocess artifactにexact一致し、Import QualificationのLoader／Local Manifest bindingはDeploymentのLoader binding／Snapshot内Manifest bindingと一致し、License DecisionのManifest bindingも同じでなければならない。五辺の一つでも違う、Snapshotが`provider_model_id`、license／provenance欠落、Receipt失効なら`diagnostic.ai.model-snapshot-binding-mismatch`でprocess起動前に拒否する。

Local branchはprocess artifact ref／hash、current runtime／loader Profile binding、Import Qualification、process／GPU isolation、mutual-auth endpoint、RAM／VRAM／CPU／memory／GPU／disk／temp cache／shard数・総bytes／child process／batch／concurrent inference／wall-time／output上限を全て必須にする。`os_ipc`はnetwork deny-all、`authenticated_loopback`はexact loopback originだけを許して外部networkをdeny-allにし、両branch混在、wildcard bind、LAN bindを拒否する。Local manifestの`model_format_id`はLoaderの`supported_model_format_ids[]` exact一件に一致しなければならない。custom model code、unsafe serialization、archive path escape、runtime network fetch、built-in file／shell tool、MCP proxyを禁止し、すべてのTool実行をMiraikanai Gatewayへ戻す。llama.cppを候補Adapterにする場合も、公式serverでexperimentalかつuntrusted環境非推奨の`--tools`と`--ui-mcp-proxy`を無効のまま固定する（[公式server README](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)）。loaderがweights/config/tokenizer closure外のartifactを要求すれば拒否する。preflight不足は開始前拒否、実行中超過はprocess tree終了＋failed Receiptへ収束させ、System memoryへの無制限spill、GPU共有contextからの隔離省略、Schema非対応Toolの自然言語代替を禁止する。

License文字列の存在だけではActivationしない。`ModelLicenseAcceptanceDecisionV1`はexact license／manifest／Project revision／intended-use profile／distribution planに対し、game生成、commercial use、weight redistribution、output use、required noticeの各許可をR4 License主体が署名する。`approved_use_kinds[]`は`game_authoring | asset_generation | code_generation | evaluation | redistribution`のclosed subsetとする。Project用途が同集合に含まれ、redistribution／output policyがpackage／distribution planと一致し、Decision／Signer／Role／Key／validity／revocationがcurrentな場合だけlocal deploymentを有効化する。`decision_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:model-license-acceptance:sha256:<lowercase-hex>`として導出し、`role.model-license-acceptance.r4`／singleton purpose `model_license_acceptance_decision`、`subject_sha256=SHA-256(JCS(payload))`、signed issued-at／revocation equalityを必須にする。

Local inferenceは既定で`output_reproducibility.kind=not_claimed`とし、completed実行のexact response bytes hash、runtime／model／settings／seedを署名済み`LocalInferenceRunReceiptV1`へ保存するが、同一出力の再現性を主張しない。preflight／runtime failureはfailed branch、null response hash、1件以上のtyped Diagnosticで表す。`deterministic_claim`だけはruntime artifact、GPU profile、inference settings、seed、sampler、thread policy、replay input setを固定し、最低3件の重複しないfresh completed Run Receiptを列挙する。全Run Receiptの同入力bindingと`exact_response_bytes_sha256`が一致する場合だけRepeatabilityを`byte_equal`にする。未固定hardware／runtime、単一hash自己申告、回数だけの申告でdeterministicと表示しない。

fallbackの唯一の正本は`ProviderManifestV1.fallback`であり、`InferenceDeploymentProfileV1`は`fallback_policy_owner=provider_manifest_only`として別bindingを持たない。`disabled`またはcurrent exact `fallback_provider_manifest_binding`だけを許し、fallback Manifestは元Manifestと別logical ID、self edgeなし、fallback先の`fallback.kind=disabled`、最大depth 1とする。chain／cycle、二段fallback、Deploymentだけの差替えを拒否する。切替前にfallback ManifestからProvider Runtime、Inference Deployment、Model Snapshot、Tool Projection、privacy／region／retention／costを一つのfull Engine Provider Adapter tupleとして解決し、全Profile Head、Provider／Tool Conformance、Schema／Eval Conformance、Target、expiry／revocationをfresh検証する。元Caller ContextまたはReceiptを流用せず、fallback tupleへ新しいCaller Context／Task Authorizationを発行し、`fallback_max_risk`との積集合と明示User確認を必須にする。Localからcloudへ移る場合は送信data class、Provider、region、retention、費用を再Previewする。Network到達性、timeout、resource exhaustionを理由にcloudへ暗黙送信した場合は`diagnostic.ai.silent-cloud-fallback-forbidden`で失敗し、元Taskの状態とProjectを不変にする。

Gemma、Kimi、Qwen、DeepSeek、Grok、GLMその他のModel familyごとのEngine branchを作らない。任意のProvider／local runtime／Modelは同じProfileと三種のtyped Conformance Receiptで扱う。routeが要求するfull subject tuple Receiptがなければそのrouteは`not_activated`であり、`proposal_only`へ暗黙降格しない。`proposal_only`は、別のcurrent `standard_external_mcp` profile／Grant／Host／Transport Receipt／Schema／Eval ReceiptからGatewayが新Caller Contextを発行できる場合だけである。外部Conformance済みHostがlocal modelを使う経路と、Miraikanai自身がlocal runtimeを配布・運用するCapabilityを分ける。first-party local inferenceはFuture incubationであり、MVPまたは初心者First PlayableのCompletion Gateにしない。
