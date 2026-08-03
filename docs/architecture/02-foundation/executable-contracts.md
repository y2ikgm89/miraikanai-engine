# Miraikanai Engine Executable Contracts

- 文書ID: mirakan.arch.executable-contracts
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: MCD共通意味、Requirement、Type、Operation、Data Flow共通kind／Envelope、generic Operation Receipt identity／type resolver、State machine、Capability、Policy、Profile、Diagnostic、Service、canonicalization、Contract compiler、C++／TypeScript／MCP／Provider／Cooked projection
- 非正本範囲: 具体Operation／planning catalog、外部Tool・package固定、Product scope、Product data-flow payload／Privacy semantics、AI authorization、Evidence envelope、Project transaction schema、Domain固有runtime semantics
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Core Architecture](core-architecture.md)、[Toolchain／Dependencies](toolchain-dependencies.md)
- 関連文書: [MCP Current Protocol Baseline Decision](../decisions/2026-08-03-mcp-current-protocol-baseline.md)、[Operation／Planning Candidate Catalog](../appendices/executable-contracts-operation-planning-catalog.md)、[Product Plan](../00-product/product-plan.md)、[Product Lifecycle](../00-product/product-lifecycle.md)、[Product Security](../01-governance/product-security.md)、[Product Privacy／Data Governance](../01-governance/product-privacy-data-governance.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Project State](../03-authoring/project-state.md)、[Memory／Pointers](memory-pointers.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-08-03

## 1. 結論

Miraikanai Contract Definition（MCD）はRequirement、Type、Operation、State machine、Capability、Policy、Profile、Diagnostic、Serviceを一つのversioned contract graphとして表す。C++、TypeScript、内部JSON Schema、MCP、Provider strict schema、Cooked binaryはMCDから決定論的に投影し、別々の手書き正本を持たない。

2026-07-27時点で、MCD Schema、Contract compiler、生成Artifact、active Contract setはRepositoryに存在しない。本書の型名、version、catalogは`review`候補であり、実装済みまたはactiveと表現しない。

## 2. AuthorityとArtifact分類

Source MCD、generated projection、runtime artifact、Evidenceを区別する。

| class | authority | rule |
|---|---|---|
| Source MCD | Ownerが承認したSource record | generated projectionから逆生成しない |
| Contract set | exact Source member closure | member追加・削除・順序差をhashへ反映 |
| Generated projection | compiler output | Source ref、compiler identity、Target profileを束縛 |
| Runtime artifact | qualified projectionのpackage result | SourceまたはSchema正本として編集しない |
| Evidence | verification OwnerのReceipt | Contract内容またはApprovalを代替しない |

## 3. DirectoryとFile規約

MCD Sourceは`/schemas/mirakan/`、generated artifactはRepository規約が定めるDerived rootへ置く候補とする。Sourceとgenerated outputを同じpathへ置かず、AI、Editor、CIはBuild Gateway／Contract compilerを経由する。実在しないpathを生成済みと表現しない。

## 4. MCD kind

共通kindは`requirement | type | operation | data_flow | state_machine | capability | policy | profile | diagnostic | service`のclosed setとする。Domainが独自kind、別Envelope、別version意味を追加しない。Domain固有意味は共通kindのpayloadとOwner refで表す。

`kind=data_flow`はProductが所有またはbundleするoutbound／persisted data flowをContract setへ閉じる共通Envelope kindである。payloadのcomplete schema、purpose、data category、destination、processor、region、retention、consent、User controlおよびPrivacy acceptance意味は[Product Privacy／Data Governance](../01-governance/product-privacy-data-governance.md)だけが所有する。`McdContractRefV1(kind=data_flow)`は同じContract setのexact一recordへ解決し、payloadのData Flow ID／version／content hashをread-backする。`kind=profile | type | requirement`への偽装、Privacy-local narrow Ref、bare endpoint、ID-only、display name、別Contract setまたは`latest`をdata-flow identityにしない。

## 5. MCD共通Envelope

全recordは少なくとも次を持つ。

```text
McdRecordV1
  mcd_version
  kind
  id
  version
  status
  title
  description
  owners[]
  requirement_refs[]
  rationale_refs[]
  since_contract_set
  supersedes[]
  tags[]
  payload
```

IDはkind固有namespaceのlower snake／dot token、versionは正整数、statusはclosed lifecycleを使用する。表示名、path、配列index、`latest`をidentityへ使わない。RefはID、version、Contract set hashまたはcontent hashを束縛する。

### 5.1 Foundation Definition Closure

Foundation Definition Closureは、Owner Registry、Contract set、Toolchain profile、生成projection manifest、Compatibility stateのexact root集合を束縛する。Closure不足時はMCD refをcurrent authorityへmaterializeしない。Closure候補の具体recordは[Operation／Planning Catalog](../appendices/executable-contracts-operation-planning-catalog.md#51-foundation-definition-closure)へ分離する。

## 6. Requirement定義

Requirementはowner、normative level、scope、statement、verification method、acceptance criteria、failure diagnosticを持つ。自由文だけのRequirement、検証不能な「高品質」、missing acceptance、別Target Evidenceの代用を禁止する。

## 7. Type system

Typeはbounded scalar、enum、record、tagged union、bounded array／map、typed refを許可する。Raw pointer、native object、unbounded string／collection、implicit null、open union、execution codeをwire Typeへ含めない。

Fieldはpresence、type、bound、unit、default policy、canonical orderingを明示する。defaultがないFieldをzero、empty、first enumへ補完しない。

Canonical binaryはField number、wire type、presence、length、normalized scalar encodingを固定し、JSON key順、locale、host ABIへ依存しない。

<a id="8-operation"></a>

## 8. Operation定義

Operationは次の共通Envelopeを持つ。

```text
OperationContractV1
  operation_ref
  operation_kind: query | command
  input_type_ref
  output_type_ref
  authority_service_ref
  risk_class
  side_effects[]
  idempotency
  transaction
  precondition_refs[]
  postcondition_refs[]
  error_refs[]
  validator_closure_ref
  timeout_ms
  rate_limit_policy_ref
  audit_level
  provider_exposure
  receipt_type_ref
```

State-changing Operationはauthorization、precondition、prepared candidate、postcondition、private publication preparation、signed wrapper read-back、Public Commit Closure／Marker／after-state atomic CASを分離する。Validation成功だけでpublicationせず、部分成功を残さない。Queryはstate-changing Receiptやpublicationを生成しない。

<a id="operation-receipt-identity"></a>

### 8.0 Generic Operation Receipt identity

Domain固有Receipt payloadの意味は各Ownerが所有し、本節は共通`OperationReceiptRefV1`がwrong Operation、payload Type、purpose、subjectまたはsigned recordへ付け替わらないためのidentity spineとclosed resolverだけを所有する。

| 型 | ASCII domain separator |
|---|---|
| `OperationReceiptTypeRegistryV1` | `MIRAKAN_OPERATION_RECEIPT_TYPE_REGISTRY_V1` |
| `OperationReceiptV1` | `MIRAKAN_OPERATION_RECEIPT_V1` |

```text
OperationReceiptTypeRegistryV1
  operation_receipt_type_registry_id: StableId
  operation_receipt_type_registry_version: 1
  receipt_type_entries[1..65535]:
    sorted unique {
      receipt_payload_type_ref: exact McdContractRefV1(kind=type),
      authority_service_ref: exact McdContractRefV1(kind=service),
      operation_refs[1..4096]:
        sorted unique exact McdContractRefV1(kind=operation),
      allowed_signed_purpose_refs[1..64]:
        sorted unique exact McdContractRefV1(kind=policy),
      required_receipt_subject_contract_refs[1..4096]:
        sorted unique exact McdContractRefV1(kind=type)
    }
  operation_receipt_type_registry_content_hash: SHA-256

OperationReceiptV1
  operation_receipt_id: StableId
  operation_receipt_version: positive u32
  operation_receipt_type_registry_ref:
    exact OperationReceiptTypeRegistryRefV1
  operation_ref: exact McdContractRefV1(kind=operation)
  receipt_payload_type_ref: exact McdContractRefV1(kind=type)
  signed_receipt_purpose_ref: exact McdContractRefV1(kind=policy)
  owner_receipt_record:
    owner_record_id: StableId
    owner_record_version: positive u32
    owner_record_content_hash: SHA-256
  receipt_subject_contract_refs[1..4096]:
    sorted unique exact McdContractRefV1(kind=type)
  receipt_subject_content_hash: SHA-256
  request_content_hash: SHA-256
  completed_signed_record_content_hash: SHA-256
  operation_receipt_content_hash: SHA-256
```

| Ref | Field |
|---|---|
| `OperationReceiptTypeRegistryRefV1` | `{operation_receipt_type_registry_id, operation_receipt_type_registry_version=1, operation_receipt_type_registry_content_hash}` |
| `OperationReceiptRefV1` | `{operation_receipt_id, operation_receipt_version, operation_ref, receipt_payload_type_ref, signed_receipt_purpose_ref, owner_record_id, owner_record_version, owner_record_content_hash, receipt_subject_contract_refs, receipt_subject_content_hash, request_content_hash, completed_signed_record_content_hash, operation_receipt_content_hash}` |

`OperationReceiptRefV1`はID／version／content hashだけでなく、exact Operation、payload Type、signed purpose、owner-specific backing record、subject contract集合、subject、requestおよびcompleted signed recordを同一tupleへ保持する。解決先`OperationReceiptV1`の全Fieldをbyte equalityにし、そのRegistry entryのpayload Type、Authority Service、Operation membership、purpose membership、required subject contract集合と一致させる。`OperationContractV1.receipt_type_ref`は同OperationのRegistry entryの`receipt_payload_type_ref`とbyte equalityでなければならない。generic RefからDomain successを推測せず、consumerは解決先owner recordをそのOwnerのvalidatorでread-backし、owner-specific completed subjectからsubject contract集合、full context、subject content hash、request content hashを再計算してRef／wrapperとbyte equalityにする。Target、Host、locale、Reference dimension、Project revision、Candidate、Pack、Distribution Subject等のどのcontext Fieldがrequiredかはowner payload Typeが所有し、generic wrapperへ不完全な固定context vectorを複写しない。wrong Operation、wrong payload Type、wrong purpose、wrong Authority、別subject、別request、context欠落、hash-only Ref、display action名またはReceiptへのRef併記による意味拡張を拒否する。

RegistryとReceiptのcontent hashは自己hashだけを除く全Fieldを型固有domain separator、algorithm `sha256`、algorithm version 1、schema順、`uint32_be` length framingでcanonical encodeして計算する。Registry memberはunsigned UTF-8 tuple bytes順にsortする。current initial V1へこのtupleを直接定義し、owner-specific Refを失う旧generic Ref、ID-only alias、dual Registry、`latest` resolverまたはmigration readerを設けない。現RepositoryにSchema、Registry、Receipt、resolverまたはsigned recordは存在しない。

<a id="81-project-runtime-entryruntime-scope"></a>

### 8.1 Project Runtime Entry／Runtime Scopeのtarget Operation候補

Project Runtime Entry／Runtime Scopeのtarget Operation候補が将来materializeされる場合の共通成立条件は、Project State ownerが所有するDocument identity、expected revision、Target selector、Scope owner、ChangeSet transactionとexact一致することである。具体MCD record、Policy、Service、Diagnostic、Fixture候補は[Operation／Planning Catalog](../appendices/executable-contracts-operation-planning-catalog.md#81-project-runtime-entryruntime-scopeのtarget-operation候補)へ分離する。

この節は`service.authoring_command_gateway`が完全登録済みOperationだけを処理し、Provider／MCPが直接Projectを変更しないというauthority境界を所有する。補助Catalogのrecord件数またはIDをactive inventoryと解釈しない。

<a id="812-conditional-legacy-migration-evidence-gate"></a>

### 8.1.2 Post-release migration admission rule

Initial V1へmigration Operationを登録しない。最初の公開後に実在する旧Source、complete consumer inventory、Compatibility Change、migration fixture、fresh Qualification、Approvalがすべてmaterializeした場合だけ、新しいversioned closed catalogへ追加できる。条件未成立時のcurrent migration Operation集合は空である。admission ruleの詳細は[補助Catalog](../appendices/executable-contracts-operation-planning-catalog.md#812-conditional-legacy-migration-evidence-gate)を参照する。

### 8.2 Target Installed Product Operation closureとcurrent empty state

Installed Productのtarget Operation closureは、Product Definition、Contract set、Service allowlist、Policy、Validator、Diagnostic、Target bindingのset equalityから決定する。Markdown上の候補表、未Activation planning record、Provider discovery resultをmaterialized／contract-active／active／operational集合へ混入しない。現Repositoryではclosure inputが一つもmaterializeしていないため、四つのcurrent集合はすべてexact `[]`である。

[Product Plan](../00-product/product-plan.md)の`RequiredProductOperationUniverseV1`がProduct claimからrequired `{family,operation,requirement,surface}`集合を導出し、`ProductOperationActivationClosureV1`がMCD Operation、Owner Manifest、Authority Service、Service allowlist、Validator、Diagnostic、Receipt type、surface projection、Activation Evidenceを同じContract setへ閉じる。本書の`OperationContractV1`は各exact Operation Refについてinput／output／authority／risk／side effect／transaction／precondition／postcondition／error／validator／timeout／rate limit／audit／provider exposure／Receiptを完全定義する。Product family名、UI action、CLI command、MCP tool名またはstate transition名からOperation Refを生成しない。

Product claimに必要なinstall／update／repair／uninstall、Project bootstrap／open、human author／preview／validate／commit、build／test／cook／package／launch、diagnostics／support、Pack acquire／install／apply／update／remove、AI read／explain／propose／validate／approve／commit、security case transition／update／disclosure／incident response、publication／withdrawalは[Product Plan](../00-product/product-plan.md)のclosed non-collapsed family universeであり、各familyはexact MCD Operationへ解決しなければならない。human authoringとAI authoring、cookとpackage、diagnosticsとsupport、Pack lifecycleとProduct lifecycleを同一familyへ縮退しない。AI／MCPはProvider projectionであって別Operationまたは別authorityではなく、proposalからcommitまでを単一write Toolへ融合しない。state-changing Operationだけがprepared candidate、required Approval、Authoring Command Gateway、atomic Commit Closure、typed Receiptを持ち、queryはProject mutationまたはCommit Receiptを持たない。

target designのrequired universeとgeneric Operation contractは以上で閉じるが、現RepositoryにはMCD Source、Contract set、Owner Manifest、Service allowlist、Policy、Validator、Diagnostic、Receipt Schema、surface projection、Activation Evidenceが存在しない。したがってRepository-wideの`materialized_operations`、`contract_active_operations`、`active_operations`、`operational_operations`はすべてexact `[]`であり、Architecture上のtarget Operation ID、target-complete candidateまたはfamilyを実装済み、登録済み、契約参照可能またはdispatch可能と扱わない。current Operation状態の唯一の正本は本節であり、appendix、AI Security、Domain文書またはtoken lintが非空集合へ上書きしない。

## 9. State machine定義

State machineはclosed state、initial state、event、guard、transition、terminal state、failure diagnosticを宣言する。Unknown state／eventを無視せず、同じinputから複数transitionを選ばない。Runtime implementationはMCD transition semanticsを再定義しない。

## 10. Capability定義

Capabilityはidentity、maturity、owner、required contract、Target availability、activation condition、qualification requirementを分離する。maturityまたはversionをCapability IDへ埋め込まず、文書の記載だけでactiveにしない。

すべての`McdContractRefV1(kind=capability)`は、同じContract set内の完全なCapability recordへexactly oneで解決しなければならない。Provider descriptor、Game System、Role、Fixture表にIDが現れるだけでは定義にならず、未定義IDを名前やProduct Capabilityから補完しない。

MCD CapabilityとProduct `CapabilityRegistryV1` rowは別のauthorityである。MCD Capabilityは実行Contractの存在と依存を表し、Product rowはTarget別の選択、成熟度表示、Activation evidenceを表す。両者は明示的なversion／hash付きBindingで接続し、同名、prefix、maturity、Targetの一致から相互生成しない。

<a id="target-capability-snapshot"></a>

### 10.1 Target Capability Snapshot共通Envelope

```text
TargetCapabilityEntryV1
  capability_ref: exact McdContractRefV1(kind=capability)
  availability: supported | unsupported
  platform_projection_entry_ref: exact ArtifactRefV1(
    artifact_kind=platform_capability_projection_entry,
    schema_version=1)
  constraint_profile_ref:
    null | exact McdContractRefV1(kind=profile)
  capability_entry_content_hash: SHA-256

TargetCapabilitySnapshotV1
  capability_snapshot_id: StableId
  capability_snapshot_version: 1
  target_profile_ref: exact TargetProfileRefV1
  contract_set_content_hash: SHA-256
  platform_projection_ref: exact ArtifactRefV1(
    artifact_kind=platform_capability_projection,
    schema_version=1)
  source_generation: positive u64
  capability_entries[1..4096]:
    sorted unique TargetCapabilityEntryV1
  capability_snapshot_content_hash: SHA-256

TargetCapabilitySnapshotRefV1
  capability_snapshot_id: StableId
  capability_snapshot_version: 1
  target_profile_ref: exact TargetProfileRefV1
  capability_snapshot_content_hash: SHA-256
```

各EntryのCapability refは`contract_set_content_hash`で束縛した同じMCD Contract setへexact一件解決する。`availability=supported`はPlatform Ownerの完成projection entryと、nullableならCapabilityが要求するconstraint Profileへ解決できる場合だけ許し、`unsupported`でも観測したPlatform entryを省略しない。Snapshotは対象Targetで評価するCapability完全集合をCapability refのclosed MCD canonical bytes unsigned lexicographic順へstrict sortし、duplicate ID、同ID別version／hash、Provider名、native feature bit、表示名または配列順からsupportを推測しない。`capability_entry_content_hash`と`capability_snapshot_content_hash`はそれぞれASCII `MIRAKAN_TARGET_CAPABILITY_ENTRY_V1`／`MIRAKAN_TARGET_CAPABILITY_SNAPSHOT_V1`と自己hashを除くclosed MCD canonical bytesから計算する。

Windows／Android／Apple等のPlatform Ownerはnative API、OS、driver、device、permission、surface条件を`platform_projection_entry_ref`へ写像するが、この共通Field集合を再定義しない。SnapshotはTarget supportのrevisioned read-only projectionで、Product Capability activation、Qualification pass、Release readinessまたはruntime fallbackを単独では意味しない。現RepositoryにMCD、Platform projection artifact、Snapshot、Registryまたは生成器は存在せず、Consumerは同名の自由JSON、native capability structまたは文書表から不足を補完しない。

## 11. Policy、Profile、Service

Policyは判断predicate、ProfileはTarget／環境ごとのclosed configurationを表す。Policyに副作用を持たせず、Profileの未指定FieldをHost defaultへ補完しない。Tool versionとDependency pinはToolchain Ownerが所有する。

ServiceはOperationを実行できるauthority identity、許可Operation集合、trust boundary、availability policy、version、content hashを表す。ServiceがDomain state、Operation semantics、Approvalを再定義せず、Provider discoveryまたは接続成功だけでServiceやOperationをactiveにしない。

## 12. Diagnostic契約

Diagnosticはstable identity、machine code、severity、owner、condition、subject ref、safe parameters、remediation refを持つ。自由文だけのerror、secretを含むparameter、異なるfailureへのcode再利用を禁止する。

### 12.1 `MirakanDiagnosticV1`

```text
MirakanDiagnosticV1
  diagnostic_ref
  severity
  subject_ref
  operation_ref: nullable
  safe_parameters{}
  cause_refs[]
  remediation_refs[]
  evidence_refs[]
```

具体code候補は[Operation／Planning Catalog](../appendices/executable-contracts-operation-planning-catalog.md#121-mirakandiagnosticv1)に隔離する。補助文書のcodeをRepository Registryなしにcurrent codeと扱わない。

## 13. CanonicalizationとHash

CanonicalizationはUnicode、number、Field ordering、map ordering、presence、binary encodingをversioned profileで固定する。Human-readable JSON、platform ABI、unordered iterationをhash inputにしない。Content hashはalgorithm、profile version、kind、payloadを含むdomain-separated preimageから生成する。

## 14. Contract compiler

CompilerはSource parse、cross-ref解決、Owner／Requirement closure、cycle検査、canonicalization、hash、Target projection、round-trip fixtureを一つのclean buildで行う。Error時はArtifactをpublicationせず、last-valid resultを維持する。

## 15. Internal JSON Schema projection

JSON Schemaは内部validation projectionでありSource MCDではない。Dialect、strictness、local `$ref` allowlist、unknown Field拒否をToolchain profileと一致させる。

## 16. Provider projection

MCP、OpenAI、Anthropic、CLI／Desktop projectionは同じOperationの許可されたField subsetである。Provider固有Schema制約のため意味、authority、Risk、side effect、errorを変更しない。Credential、Approval token、private EvidenceをTool Schemaへ含めない。

### 16.1 共通projection境界

Provider surfaceはOperation ref、input／output Schema ref、authority class、Risk、side effect、Diagnostic、bounded resultをMCDから投影する。surface名、Provider annotation、Host capability表示または接続成功からOperation membership、write authority、Capability activationを生成しない。

### 16.2 MCP 2026-07-28 projection

initial V1のMCP projectionはToolchain Ownerが固定するsupported-version set exact `[2026-07-28]`だけを受理する。各requestの`_meta["io.modelcontextprotocol/protocolVersion"]`を必須にし、Streamable HTTPでは同じ値の`MCP-Protocol-Version` headerも必須としてbyte equalityにする。stdioはHTTP headerを持たずrequest `_meta`の値だけを使う。`server/discover`は同じsingleton version、Server identity、実際に投影可能なcapability集合だけを返し、未materialize Operation、Provider-only Tool、内部Validatorまたはprivate Evidenceをadvertiseしない。

missing `_meta` key、`2026-07-28`以外、request `_meta`／HTTP header mismatch、discovery結果外capabilityまたはmutual version不在は副作用前にunsupported protocolとして拒否する。`2025-11-25` initialize、legacy lifecycle、旧Tool名alias、dual-version sessionまたは自動fallbackをinitial V1へ持たない。version適合はSchema、Authorization、Operation activation、Host／Transport ConformanceまたはTool execution成功を代替しない。MCP Server、Schema、Registry、Fixture、ReceiptおよびSDKはRepositoryに存在せず、本節は実装または相互運用を主張しない。

## 17. Language／Runtime projection

C++、TypeScript、Gameplay Cooked binaryは同じType／Operation identityとcanonical fixtureを共有する。Host ABI、compiler padding、JavaScript number coercion、JSON serialization差をwire semanticsへ持ち込まない。

## 18. Round-tripとCross-language同値性

各Typeはcanonical bytes、JSON projection、C++、TypeScript間でvalue、presence、enum、bound、hashが一致するfixtureを持つ。Decode→encodeで別bytesになる、unknown Fieldを保持する、overflowを丸める実装を拒否する。

## 19. Schema変更とMigration

変更はcompatible addition、breaking replacement、offline migration、retirementへ分類する。既存versionをin-place変更せず、Consumer Inventory、Compatibility Change、migration Evidence、rollback／recook条件を束縛する。

## 20. AI向けDiscovery／Execution候補のplanning record（未Activation）

Discovery／Execution planning recordの具体候補は[Operation／Planning Catalog](../appendices/executable-contracts-operation-planning-catalog.md#20-ai向けdiscoveryexecution候補のplanning-record未activation)へ分離する。Planning recordはOperation登録、Provider exposure、authorization、Capability activationを発生させない。

[Game Production Loop](../03-authoring/game-production-loop.md)の`game_intent_understanding`、`game_experience_iteration`、`game_production_read`も同じ`PlannedOperationFamilyV1`だけを使う。前三familyのstate-changing意味はprepared Proposalまでを外部投影上限とし、実Project変更は別のactive ChangeSet carrierとtrusted-internal Commitを必要とする。`game_production_read`はread-onlyで、三familyのProvider／MCP projectionへProject Commit、Human Gameplay Approval、Source Promotion、Activation、Signing、Releaseを含めない。現Repositoryでは三familyを含むreserved 27 family／207候補の全current集合とalias集合がexact `[]`、Capability stateが`not_activated`である。

## 21. Project Shader Discovery／Proposal候補

Project Shader候補も同じOperation共通意味と未Activation規則に従う。

### 21.1 既存Domain文書から回収した未登録Operation候補

回収候補は[補助Catalog](../appendices/executable-contracts-operation-planning-catalog.md#211-既存domain文書から回収した未登録operation候補)で追跡する。候補名をcurrent Operation setへ加算しない。

### 21.2 Architecture内`operation.*` tokenのclosed partition

Architecture内の完全な`operation.*` tokenは、active、conditional、planning、legacy inputのいずれか一つに分類する。重複、未分類、複数分類を拒否する。具体partition候補は[補助Catalog](../appendices/executable-contracts-operation-planning-catalog.md#212-architecture内operation-tokenのclosed-partition)を参照する。

## 22. Contract compilerの受入条件

受入条件は決定論的clean build、全ref解決、Owner一意性、Requirement closure、hash再現、全Target projection、round-trip、negative fixture、partial output 0である。実行結果とReceiptがない間は未検証とする。

## 23. 一次資料と採用根拠

- [JSON Schema Draft 2020-12](https://json-schema.org/draft/2020-12)
- [Model Context Protocol 2026-07-28 specification](https://modelcontextprotocol.io/specification/2026-07-28)
- [Model Context Protocol versioning](https://modelcontextprotocol.io/docs/2026-07-28/learn/versioning)
- [OpenAI Structured Outputs](https://developers.openai.com/api/docs/guides/structured-outputs)
- [Protocol Buffers Encoding](https://protobuf.dev/programming-guides/encoding/)

## 24. 明示的に採用しないもの

- unrestricted scripting runtimeを契約正本にする方式
- Provider Tool SchemaをEngine Sourceへ逆変換する方式
- latest version／display name／pathによるref解決
- unknown Field、unknown enum、missing validatorの黙示受理
- validation成功とstate publicationの同一視
- Domainごとの独自Envelope、独自hash、独自Diagnostic wire format
