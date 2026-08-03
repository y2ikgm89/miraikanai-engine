# Miraikanai Engine Product Privacy／Data Governance Contract

- 文書ID: mirakan.arch.product-privacy-data-governance
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: MCD `kind=data_flow`のProduct-owned payload、Engine、Editor、Game Runtime、AI、telemetry、crash、supportにまたがるProduct data inventory、purpose、legal／consent basis、collection、disclosure、processor／region、retention、export／delete、User control、privacy release acceptance
- 非正本範囲: MCD共通Envelope／Ref、Security threat／credential／sandbox、AI Operation authorization、Platform permission API、Game Project自身が収集するPlayer dataの製品固有policy、SupportBundle field、network transport、Store disclosure形式。各Ownerを参照する
- 規範依存: [Architecture Governance](architecture-governance.md)、[Product Plan](../00-product/product-plan.md)、[AI Verification／Provenance](ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Native Game Module](../03-authoring/native-game-module.md)
- 関連文書: [Product Lifecycle](../00-product/product-lifecycle.md)、[Product Security](product-security.md)、[AI Security／Approval](ai-security-approval.md)、[AI Verification／Provenance](ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Project State](../03-authoring/project-state.md)、[Editor Workspace／UX](../03-authoring/editor-workspace-ux.md)、[Debugging／Observability／Replay](../04-runtime/debugging-observability-replay.md)、[Mobile Common](../07-platform/mobile-common.md)、[Android](../07-platform/android.md)、[Apple](../07-platform/apple.md)、[Windows](../07-platform/windows.md)
- 根拠区分: project-decision（法令、Store、PlatformまたはProvider要件を引用する箇所はofficial-spec。jurisdiction判断は製品releaseごとのlegal reviewを必要とする）
- 外部根拠確認日: none

## 1. 結論と所有境界

PrivacyはMobile permissionまたはAI promptだけの問題ではない。Engine install、Editor、Project、build、Game Runtime、telemetry、crash、support、update、AI Providerを通じてProductが扱うdataを一つのgovernance boundaryで追跡する。

Product releaseは、収集対象、目的、送信先、保存場所、保持期間、User controlが不明なdata flowを持ってはならない。未登録flowはdenyし、diagnostic、stabilityまたは「品質改善」を包括目的として個別data categoryを暗黙許可しない。

本書はMiraikanai Engine製品のprivacy責任を所有する。Game Developerが制作するGame固有のaccount、analytics、advertising、commerce、multiplayerまたはPlayer data policyは自動的にEngine policyへ包含されない。Engineは必要なCapabilityと開示材料を提供できるが、Game releaseのdata controller判断を代行しない。

対応するSchema、Registry、consent UI、telemetry pipeline、processor契約、export／delete Tool、ReceiptはRepositoryに存在せず、すべて未materializeである。

## 2. Data subjectとdata category

Product dataは最低限、次のsubjectを区別する。

| Subject | 例 | 禁止する混同 |
|---|---|---|
| Engine Developer | Engine／Editorを利用する第三者Developer、organization member | Game PlayerまたはProject content ownerと同一視しない |
| Game Player | Miraikanaiで作られたGameの利用者 | Engine telemetry consentでGame analyticsを許可しない |
| Project Collaborator | repository、Project、Packへ寄与する人 | local machine Userの設定とProject共有dataを混ぜない |
| Support Requester | support caseを提出する人 | support目的を継続telemetryへ転用しない |
| Minor／protected subject | release jurisdictionが追加保護を要求するsubject | 年齢または法的statusを推測して通常flowへ落とさない |

Data categoryは最低限、次へ分類する。

| Category | 例 | Default handling |
|---|---|---|
| Project content | Source asset、Game text、code、shader、scene、prompt添付 | local／Project-authorized。明示actionなしに外部送信しない |
| Project metadata | Project ID、revision、Target、dependency、path-derived identifier | purposeごとに最小化し、path／usernameを直接送信しない |
| User preference | Editor locale、layout、recent Project、accessibility setting | localを既定としProject共有と分離 |
| Identity／account | license entitlement、organization、support identity | 必要なsurfaceだけ。anonymous diagnosticと結合しない |
| Usage telemetry | command、feature usage、duration、success／failure aggregate | default off。明示opt-inと撤回を必要とする |
| Performance telemetry | frame、memory、device class、build duration | local profilingとremote telemetryを別purposeにする |
| Diagnostic／log | typed diagnostic、stack、operation history | secret／content redaction後も送信前previewを必要とする |
| Crash／dump | minidump、memory fragment、module list、call stack | sensitiveとして扱い、silent uploadを禁止する |
| AI request／response | prompt、context projection、attachment、tool result | exact Provider／purpose／retentionを明示し、Project authorizationを必要とする |
| Support bundle | Userが選択したlog、config、crash、Project excerpt | explicit export、preview、redaction、case-bound retention |
| Security evidence | abuse、integrity、revocation、audit record | accessを最小化し、privacyとsecurity retentionを区別 |
| Distribution／legal | SBOM、NOTICE、license、publisher identity | public／release dataとprivate contact dataを区別 |

credential、private key、access token、raw password、OS secret、unrelated browser／filesystem contentは収集対象にしてはならない。誤って検出した場合は送信せず、local redaction diagnosticを返す。

## 3. Purpose bindingとdata-flow inventory

各data flowはExecutable Contractsの共通`McdRecordV1(kind=data_flow)`へ、次のcomplete payloadを一件だけ持つclosed recordとして定義する。Privacy Ownerはpayload意味を所有するが、別Envelopeまたはlocal Refを作らない。

| 型 | ASCII domain separator |
|---|---|
| `ProductDataFlowDefinitionV1` | `MIRAKAN_PRODUCT_DATA_FLOW_DEFINITION_V1` |
| `ProductDataFlowInventoryV1` | `MIRAKAN_PRODUCT_DATA_FLOW_INVENTORY_V1` |
| `ProductPrivacyRequiredScopeProjectionV1` | `MIRAKAN_PRODUCT_PRIVACY_REQUIRED_SCOPE_PROJECTION_V1` |
| `ProductPrivacyAcceptanceV1` | `MIRAKAN_PRODUCT_PRIVACY_ACCEPTANCE_V1` |

```text
McdRecordV1(kind=data_flow)
  common_envelope: exact Executable Contracts McdRecordV1
  payload: exact ProductDataFlowDefinitionV1

ProductDataFlowDefinitionV1
  data_flow_id: StableId
  data_flow_version: positive u32
  product_surface: engine_installer | editor | build_tool | game_runtime
    | ai_adapter | telemetry | crash | support | update
  subject_kinds[1..8]: sorted unique DataSubjectKind
  data_category_refs[1..64]: sorted unique ProductDataCategoryRefV1
  purpose_ref: exact ProductDataPurposeRefV1
  collection_trigger: explicit_user_action | consented_background
    | required_local_operation | security_necessity
  source_boundary_ref: exact DataBoundaryRefV1
  destination_refs[0..16]: sorted unique DataDestinationRefV1
  processor_refs[0..16]: sorted unique DataProcessorRefV1
  region_policy_ref: exact DataRegionPolicyRefV1
  retention_policy_ref: exact DataRetentionPolicyRefV1
  consent_policy_ref: exact ProductConsentPolicyRefV1
  redaction_policy_ref: exact DataRedactionPolicyRefV1
  export_delete_policy_ref: exact DataSubjectControlPolicyRefV1
  security_control_refs[1..64]: sorted unique exact SecurityControlRefV1
  disclosure_entry_refs[1..16]: sorted unique PrivacyDisclosureEntryRefV1
  governing_requirement_refs[1..32]:
    sorted unique exact McdContractRefV1(
      kind=requirement, category=privacy)
  execution_scope_applicabilities[1..129]:
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
  locale_applicabilities[1..65]:
    sorted unique {
      kind=locale_profile,
      locale_profile_ref: exact LocaleProfileRefV1
    }
    | {kind=locale_independent}
  distribution_route_applicabilities[1..16]:
    sorted unique
      {kind=not_distributed}
      | {
          kind=distribution_route,
          platform_kind: windows | android | apple,
          channel_kind: direct_distribution | managed_store
        }
  data_flow_content_hash: SHA-256

ProductDataFlowInventoryV1
  data_flow_inventory_id: StableId
  data_flow_inventory_version: 1
  product_definition_ref: exact ActiveProductDefinitionRefV1
  candidate_ref: exact PreparedCandidateRefV1
  host_profile_refs[2..64]:
    sorted unique exact TargetProfileRefV1(
      profile_kind=build_host | editor_host)
  runtime_target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1(profile_kind=runtime_target)
  locale_profile_refs[2..64]:
    sorted unique exact LocaleProfileRefV1
  data_flow_refs[0..1024]:
    sorted unique exact McdContractRefV1(kind=data_flow)
  data_flow_inventory_content_hash: SHA-256

ProductPrivacyLegalScopeKeyV1
  data_flow_ref: exact McdContractRefV1(kind=data_flow)
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
  distribution_route:
    {kind=not_distributed}
    | {
        kind=distribution_route,
        platform_kind: windows | android | apple,
        channel_kind: direct_distribution | managed_store
      }
  locale_scope:
    {
      kind=locale_profile,
      locale_profile_ref: exact LocaleProfileRefV1
    }
    | {kind=locale_independent}
  jurisdiction_ref: exact DataJurisdictionRefV1
  processing_region_ref: exact DataProcessingRegionRefV1
  processor_scope:
    {kind=no_processor}
    | {kind=processor, processor_ref: exact DataProcessorRefV1}

ProductPrivacyRequiredScopeProjectionV1
  privacy_required_scope_projection_id: StableId
  privacy_required_scope_projection_version: 1
  product_definition_ref: exact ActiveProductDefinitionRefV1
  data_flow_inventory_ref: exact ProductDataFlowInventoryRefV1
  required_flow_scope_bindings[0..1024]:
    sorted unique {
      data_flow_ref: exact McdContractRefV1(kind=data_flow),
      required_execution_scopes[1..129]:
        sorted unique execution_scope,
      required_legal_scope_keys[1..262144]:
        sorted unique ProductPrivacyLegalScopeKeyV1
    }
  projection_algorithm_id: product_privacy_required_scope_projection
  projection_algorithm_version: 1
  projection_algorithm_content_hash: SHA-256
  privacy_required_scope_projection_content_hash: SHA-256
```

Purposeは具体的Outcomeへ限定する。`product_improvement`、`service`、`operations`のように複数flowを包含する裸のpurposeを禁止する。一つのdataを別purposeへ利用する場合は別flow、別basis、別disclosure、必要なら別consentを定義する。

Data Flowの唯一のexternal RefはExecutable Contractsの`McdContractRefV1(kind=data_flow) {id, version, contract_set_hash}`であり、解決先共通Envelopeの`id／version`とpayloadの`data_flow_id／data_flow_version`をbyte equalityにし、同じContract set内のpayload `data_flow_content_hash`をread-backする。Privacy-local narrow Data Flow Ref、三Field payload tuple、ID／versionだけ、endpoint、表示名またはPrivacy Policy textからRefを生成しない。`ProductDataFlowInventoryRefV1`は`{data_flow_inventory_id, data_flow_inventory_version=1, data_flow_inventory_content_hash}`、`ProductPrivacyRequiredScopeProjectionRefV1`は`{privacy_required_scope_projection_id, privacy_required_scope_projection_version=1, privacy_required_scope_projection_content_hash}`のclosed tupleで、解決先全Fieldとbyte equalityにする。各content hashは上表のdomain separatorと自己hashを除く全FieldのMCD canonical bytesを各`uint32_be` length framingした列のSHA-256である。

Data-flow inventoryはEngine-owned first-party flow、bundled third-party processor、AI Provider、crash endpoint、license endpoint、support tool、update serviceをすべて含むreceipt-free canonical rootである。InventoryのHost、runtime Target、locale集合はActive Product Definitionの対応membershipと各々set equalityである。`data_flow_refs[]`は`product_definition_ref`が解決する`required_product_data_flow_refs[]`とRef全Fieldでset equalityにし、Candidate作成者、Acceptance作成者またはPlatformごとのcallerが縮小／拡張できない。各Refは同じContract setのcomplete `McdRecordV1(kind=data_flow)`へ解決し、Privacy payload validatorを通過しなければ集合比較へ参加できない。Platform permission、SDK manifest、network hostnameまたはPrivacy Policy textだけをinventoryの代用にしない。Pack declaration、AI／crash／support／update surfaceまたは新endpointがflowを追加する変更は、先にActive Product Definitionのrequired membershipとexact Definitionを更新し、同じProduct Definitionを束縛するInventoryへ反映しなければCandidateをvalidにしない。optional／default-off flowもInventoryから除外せず、Product Definitionがexplicit emptyの時だけInventory emptyを許す。

各Flow Definitionのexecution scopeとlocale scopeはInventory membership内であり、Host／runtime Target／scope-independentおよびlocale／locale-independent branchを相互aliasにしない。`not_distributed`はdistribution route rowと同じFlowで混在できず、配布routeを持つFlowでは`governing_requirement_refs[]`のexact Product Requirement typed inputが持つ中立publication selectorのplatform／channel、execution scope、locale scopeにfull tupleでjoinできなければならない。Requirement selectorからartifact identityを逆算せず、FlowのProduct surface名、endpoint、既定OSまたは空配列からscopeを推測しない。

Privacy Required Scope ProjectionのNamed Algorithm v1は、Inventoryの各FlowについてDefinitionのexecution／locale／distribution applicability、`region_policy_ref`が解決する全jurisdiction／processing region、`processor_refs[]`または`no_processor` singletonをchecked Cartesian expansionし、`ProductPrivacyLegalScopeKeyV1`を生成する。distribution route rowは同Flowのgoverning privacy Requirementのneutral publication selectorへjoinし、`not_distributed`はそのjoinを行わない。outer Flow projectionはInventory flow集合、各`required_execution_scopes[]`はFlow Definitionのexecution scope集合、全legal keyのFlow projectionも同集合とset equalityにする。これはreceipt-free required universeであり、Legal review、Platform、Consentまたはretention Receiptを含めない。

Capacity Validity Algorithm v1は各Flowのexecution scopeを129件、locale scopeを65件、distribution routeを16件、jurisdictionを64件、processing regionを32件、processor branchを17件以下にし、checked multiplication後の各Flow legal keyとproduct-wide full keyを各262,144件以下にする。overflow、上限超過、Host／Target、locale、route、jurisdiction、regionまたはprocessorのcollapse、先頭N件へのtruncateはProjection生成失敗である。

## 4. Local-firstとnetwork boundary

- offline authoring、Project open／edit、local build、local test、local debug／profile、offline Game packageは、license modelが別途許す範囲でremote telemetryまたはAI Providerを必須にしない。
- AI機能は明示的に選択したProvider routeだけを使用し、AIを無効にしても手動Editor／SDK journeyが成立する。
- remote endpointへの最初の送信前に、purpose、data category、destination／processor、region、retention、costまたはlicense影響をUserへ提示する。
- DNS、connection、retry、offline queueを「未送信」と誤表示しない。queued dataにも同じretention、encryption、delete、revocation規則を適用する。
- development build、Editor preview、packaged Gameのnetwork flowを分離し、Engine DeveloperのconsentをGame Playerへ継承しない。
- telemetry、crash、AI、support、update、license validationは別endpoint allowlistと別purposeを持つ。相互fallbackしない。

## 5. ConsentとUser control

Consentが必要なflowは、default off、purpose別、surface別、subject別とする。bundled check box、利用継続、Privacy Policy閲覧、OS diagnostic setting、別製品のconsentを同意とみなさない。

```text
ProductConsentSubjectRefV1
  subject_kind: local_device_profile | account
  subject_id: StableId
  subject_scope_content_hash: SHA-256

ProductConsentDecisionV1
  consent_decision_id: StableId
  consent_decision_version: 1
  consent_subject_ref: exact ProductConsentSubjectRefV1
  data_flow_ref: exact McdContractRefV1(kind=data_flow)
  disclosure_version_ref: exact PrivacyDisclosureVersionRefV1
  decision: granted | denied | withdrawn
  decision_source: explicit_ui | managed_organization_policy
  effective_at_utc: RFC 3339 UTC
  expected_previous_decision_ref:
    null | exact ProductConsentDecisionRefV1
  consent_decision_content_hash: SHA-256

ProductConsentDecisionRefV1
  consent_decision_id: StableId
  consent_decision_version: 1
  consent_decision_content_hash: SHA-256
```

`ProductConsentSubjectRefV1`はProduct consent namespace内のopaque subject identityで、PII、display name、email、device serial、credentialまたはProvider account tokenを含めない。`subject_scope_content_hash`はsubject kind、stable ID、identity authorityが保持するscope commitmentから生成し、local consentをaccountへ、Engine Developer consentをGame Playerへ、別Game／別Product consentを相互変換しない。

同じsubject／data flowの最初のDecisionだけ`expected_previous_decision_ref=null`を許し、以後はそのchainのcurrent headへexact解決してsubject／flowをbyte equalityにする。`effective_at_utc`はpreviousより厳密に後、同じprevious refから成功できる後続Decisionはexact一件とし、CAS conflictは再読込後に新Decisionとして判断する。`withdrawn`後に再grantする場合もwithdrawn Decisionをpreviousへ明示し、旧granted refの再公開、時刻順sortまたはlast writer winsを禁止する。Decision content hashはASCII `MIRAKAN_PRODUCT_CONSENT_DECISION_V1`と自己hashを除くclosed MCD canonical bytesから計算し、Refは完成recordへexact解決する。

`security_control_refs[]`は[Product Security](product-security.md)の完成`SecurityControlV1`へexact解決し、Data Flowに適用するControl完全集合を持つ。Control ID文字列、Requirement名、実装方式またはEvidenceの存在からRefを補完しない。Consent subject／Decision、Control bindingはいずれもtarget contractであり、現RepositoryにConsent store、Schema、Registry、UI、ReceiptまたはSecurity control実装が存在するという主張ではない。不在中はconsent-required flowをdenyし、設定値、OS consent、会話または既定値から`granted`を合成しない。

撤回は将来の収集／送信を即時停止し、queued payloadを削除する。法的またはsecurity retentionが必要な既送信dataは、保持理由、範囲、期限をUser controlから確認できる。撤回を製品全体の利用停止と結び付けない。ただし機能成立に外部処理が不可欠なAI route等は、その機能だけを利用不能として代替手動journeyを示す。

Userは最低限、次を行える。

- 現在有効なdata flow、purpose、destination、retention、consentを確認する。
- optional flowを個別にgrant、deny、withdrawする。
- local telemetry／crash queueを確認し送信前に削除する。
- support bundleの内容をpreview、除外、exportする。
- applicable dataのexport、correction、delete requestへ到達する。
- data resetがProject Source、license entitlement、Game Saveへ与える影響を実行前に確認する。

## 6. AI、telemetry、crash、support

### 6.1 AI

AI requestは[AI Security／Approval](ai-security-approval.md)が許可したbounded Projectionだけを含み、送信前にProvider identity、Model／service route、Project scope、添付、tool access、retention policyを表示する。Provider側のtraining、retention、regional processing、subprocessor条件はToolchain lockまたはrelease-bound disclosureへ固定する。

Project contentをtraining、eval、product improvementまたはhuman reviewへ二次利用するflowは、通常AI inferenceとは別purposeとして明示opt-inを必要とする。Provider policyがこの分離を保証できない場合、そのrouteをProduction Projectへ提供しない。

### 6.2 Telemetry

Usage／performance telemetryはdefault offとし、local profiler、local crash evidence、release qualificationに必要なinternal test dataと分離する。remote telemetryへProject path、asset name、User-authored text、source code、prompt、raw device identifierを含めない。必要なdiagnostic dimensionはbounded enum、ephemeral pseudonymous IDまたはUser-reviewed attachmentへ変換する。

### 6.3 Crash

Crash dumpはProject content、secret、memory fragmentを含み得るsensitive dataとする。自動生成と自動送信を分離し、local保存先、最大保持、暗号化、preview可能なsummary、削除、明示送信を定義する。unattended／managed environmentの送信はorganization policyとrelease disclosureを必要とする。

### 6.4 Support

SupportBundleのField、redaction、capture failureはDebugging Ownerが所有する。本書はexport purpose、User preview、case identity、processor、region、retention、case終了後delete、二次利用禁止を所有する。Support担当者が追加dataを要求する場合も新しいexplicit attachmentとして扱い、remote filesystem browsing権限を暗黙付与しない。

## 7. Retention、export、delete

各categoryはevent起点の有限retentionまたは根拠付きexceptionを持つ。`indefinite`、`as needed`、account存在期間の包括保持を既定にしない。

Retention開始点はcollection、upload、case close、release end、security incident close等のexact eventを指定する。local cache、remote primary、backup、processor copy、log index、derived aggregateを別storage classとして扱い、primary deleteだけを完了としない。

Exportはmachine-readableなcanonical contentとhuman-readable summaryを区別する。Deleteは対象、除外理由、processor propagation、backup aging、完了またはpartial failureをReceiptへ残す。Project Source、distributed Game、第三者Pack、Git history、既に公開したNOTICE等、Engine Productがauthorityを持たないcopyを削除済みと主張しない。

匿名化dataを保持する場合、再識別可能なlink key、small cohort、free-form contentを除去し、anonymous判定をrelease privacy reviewで検証する。pseudonymous dataをanonymousと呼ばない。

## 8. Pack、Project、Gameのprivacy boundary

- Packはnetwork、telemetry、AI、filesystem、credential、Player data capabilityをmanifestで宣言し、install前にpublisher、purpose、destination、permissionを表示する。
- Pack publisherのPrivacy PolicyまたはsignatureだけでEngine permissionをgrantしない。
- Project設定はGame releaseのPrivacy disclosure材料を生成できるが、Game Developerのlegal判断またはStore submissionを自動承認しない。
- Template／Sampleはremote collectionをdefault enabledにせず、dummy endpointや共有credentialを含めない。
- Project repositoryへconsent decision、support bundle、raw telemetry、crash dump、account tokenをcommitしない。
- Game Saveとprivacy preferenceを同一reset actionで暗黙削除しない。

## 9. Privacy release acceptance

```text
ProductPrivacyAcceptanceV1
  privacy_acceptance_id: StableId
  privacy_acceptance_version: positive u32
  product_definition_ref: exact ActiveProductDefinitionRefV1
  toolchain_closure_ref: exact BuildToolchainClosureRefV1
  public_contract_set_ref: exact PublicContractSetRefV1
  host_profile_refs[2..64]:
    sorted unique exact TargetProfileRefV1(
      profile_kind=build_host | editor_host)
  runtime_target_profile_refs[1..64]:
    sorted unique exact TargetProfileRefV1(profile_kind=runtime_target)
  locale_profile_refs[2..64]:
    sorted unique exact LocaleProfileRefV1
  candidate_ref: exact PreparedCandidateRefV1
  data_flow_inventory_ref: exact ProductDataFlowInventoryRefV1
  privacy_required_scope_projection_ref:
    exact ProductPrivacyRequiredScopeProjectionRefV1
  flow_acceptance_bindings[0..1024]:
    sorted unique ProductPrivacyFlowAcceptanceBindingV1
  privacy_acceptance_content_hash: SHA-256

ProductPrivacyFlowAcceptanceBindingV1
  data_flow_ref: exact McdContractRefV1(kind=data_flow)
  disclosure_version_refs[1..16]:
    sorted unique exact PrivacyDisclosureVersionRefV1
  processor_contract_refs[0..16]:
    sorted unique exact DataProcessorContractRefV1
  scope_acceptance_results[1..262144]:
    sorted unique {
      legal_scope_key: exact ProductPrivacyLegalScopeKeyV1,
      platform_privacy_result:
        {
          kind=passed,
          qualification_receipt_ref: exact QualificationReceiptRefV1
        }
        | {
            kind=not_applicable,
            applicability_evidence_ref: exact EvidenceRefV1
          },
      consent_control_result:
        {
          kind=passed,
          qualification_receipt_ref: exact QualificationReceiptRefV1
        }
        | {
            kind=not_applicable,
            applicability_evidence_ref: exact EvidenceRefV1
          },
      retention_export_delete_result:
        {
          kind=passed,
          qualification_receipt_ref: exact QualificationReceiptRefV1
        }
        | {
            kind=not_applicable,
            applicability_evidence_ref: exact EvidenceRefV1
          },
      legal_review_result:
        {
          kind=approved,
          legal_review_evidence_refs[1..64]:
            sorted unique exact EvidenceRefV1
        }
        | {
            kind=not_applicable,
            applicability_evidence_ref: exact EvidenceRefV1
          }
    }

ProductPrivacyAcceptanceRefV1
  privacy_acceptance_id: StableId
  privacy_acceptance_version: positive u32
  privacy_acceptance_content_hash: SHA-256
```

`product_definition_ref`は[Product Plan](../00-product/product-plan.md)のexact `ActiveProductDefinitionRefV1`へ全Field byte equalityで解決する。ID／revisionだけ、同名Definition、`latest`、Release label、Manifestまたは別AcceptanceからRef Fieldを補完せず、同ID／revision別content hashを拒否する。

`data_flow_inventory_ref`は同じProduct Definition、Candidate、Host／runtime Target／locale集合を持つ完成`ProductDataFlowInventoryV1`へ全Field byte equalityで解決し、Acceptanceの三集合もInventoryおよびActive Product Definitionと各々set equalityにする。`privacy_required_scope_projection_ref`は同じDefinitionとInventoryから上記Named Algorithmで生成したexact Projectionへ解決する。`flow_acceptance_bindings[]`の`data_flow_ref` projectionはInventoryおよびProjectionのflow集合とset equalityで、各flowへexactly oneのBindingを持つ。Inventoryがexplicit emptyならProjection bindingとAcceptance bindingもexact emptyでなければならない。

各Flow BindingのDisclosureは解決先Flow Definitionの全`disclosure_entry_refs[]`を同じversioned disclosure内でexactにcoverし、Processor Contractが解決するProcessor projectionはFlow Definitionの`processor_refs[]`とset equalityにする。`scope_acceptance_results[].legal_scope_key` projectionは同FlowのRequired Scope Projection `required_legal_scope_keys[]`とexact set equalityである。各Platform、Consent、retention／export／delete、Legal review resultは`passed／approved`またはEvidence付き`not_applicable`のclosed branchを必ず一つ持ち、「receiptなし」、null、empty Evidenceまたは別keyの結果をnot applicableへ読み替えない。各Receipt／EvidenceはAI Verification OwnerのVerification Semantic Admissibility Predicate v1を通過し、generic scope／subject contractとowner-specific completed recordから同じProduct Definition、Candidate、flow、Host／runtime Target／scope-independent、route、locale branch、jurisdiction、processing region、processor scopeおよび証跡classをread-backする。Inventory外endpoint、別Candidate Inventory、F1だけを自己申告してF2を省略するAcceptance、optional flow欠落、bare endpoint集合、hash-only Ref、cross-flow／cross-scope Evidence reuse、global Legal Evidence poolまたはAcceptance内の独自required集合を拒否する。

Product Lifecycle acceptanceは、上記の`EngineReleaseBindingRefV1`を含まないpre-release Candidate closureを完成したEngine releaseへ一方向に束縛し、Product Definition、Host、runtime Target、locale、Candidate、Toolchain、public contract、Privacy Required Scope Projectionをbyte／set equalityにする。対象scopeはLifecycleのDistribution Coverage Projectionにある全Distribution Subjectとfull artifact bindingで、Pack Distribution Subjectも同じ`privacy_disclosure` control applicability、Pack manifestのnetwork／telemetry／AI／filesystem／credential／Player data declaration、および関連data flow inventoryへexactに閉じる。Product publication distribution subjectとPrivacyのdistribution route keyはplatform、channel、execution scope、locale scopeで整合させるが、Lifecycle artifact identityをPrivacy Projectionへ逆依存させない。Packを別privacy universeへ分離し、Pack publisher policyだけでEngine側acceptanceを満たさない。

1. 全outbound data flowがInventoryのexact entryへ解決し、未登録endpointをdenyする。
2. clean installでoptional telemetry、crash upload、AI secondary useがoffである。
3. consent grant、deny、withdrawがGUI／CLI／managed policyで同じeffective policyへ収束する。
4. offline authoring、local test、local build、offline Game launchがremote optional flowなしで成立する。
5. AI request、crash、support bundleの送信前previewとredaction negative scenarioが成立する。
6. retention expiry、queued delete、export、delete request、processor propagationを検証する。
7. Windows、Android、Apple等のPlatform permission／Store disclosureがInventoryと一致する。
8. Documentation、Privacy disclosure、runtime UI、support responseが同じversionを示す。
9. wrong consent subject、stale disclosure、別release policy、unknown endpoint、redaction failureをfail closedにする。

Legal reviewはArchitecture acceptanceまたはautomated testで代用しない。Required Scope Projectionが`approved`を要求するjurisdiction／route／locale／region／processor keyごとのreview Evidenceがないreleaseは、そのdistribution scopeを主張しない。`not_applicable`は同じfull keyのapplicability Evidenceを持つtyped branchだけを許し、keyをRequired Scope Projectionから削除する手段にしない。

## 10. Failureと禁止fallback

- 未登録flow、unknown destination、processor／region／retention欠落を送信しない。
- consent不明をgrant、OS設定をProduct consent、Editor consentをGame Player consentとして扱わない。
- network failure時に別endpoint、personal account、unapproved Providerへfallbackしない。
- redaction failure時にraw payloadを送らず、Userへlocal diagnosticを返す。
- Privacy review失敗をSecurity pass、Store upload成功、利用規約同意で代用しない。
- delete failureをsuccess表示せず、残存location、理由、retry／support routeを示す。
- telemetry opt-out Userへdark pattern、機能劣化、繰返しpromptを適用しない。

## 11. 明示的非目標

- 法的助言または全jurisdictionへの自動適合をArchitectureだけで保証すること。
- Game Project固有のPlayer account、advertising、commerce、analytics policyをEngineが所有すること。
- Product Security、Platform permission、AI authorization、Debug capture Schemaを本書へ複写すること。
- anonymousとpseudonymousを同一視すること。
- telemetry、AIまたはcloud accountをoffline Engine利用の暗黙前提にすること。

## 12. 完了条件

- Product全surfaceのdata category、subject、purpose、destination、processor、region、retention、controlが一意Ownerへ閉じる。
- optional remote collectionはdefault offで、purpose別consentと撤回がある。
- AI、telemetry、crash、supportが別flowとして扱われる。
- offline authoring／testing／build／playがoptional remote flowから独立する。
- export／delete／retentionとprocessor propagationのfailureが定義される。
- Pack、Project、Game、Engine Developer、Game Playerのprivacy authorityが混同されない。
- Product LifecycleがPrivacy Evidenceをsame release acceptanceへ束縛できる。
- Schema、UI、pipeline、Receiptが未materializeであることを実装済みと表現していない。
