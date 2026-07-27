# Miraikanai Engine Executable Contracts

- 文書ID: mirakan.arch.executable-contracts
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: MCD共通意味、Requirement、Type、Operation、State machine、Capability、Policy、Profile、Diagnostic、Service、canonicalization、Contract compiler、C++／TypeScript／MCP／Provider／Cooked projection
- 非正本範囲: 具体Operation／planning catalog、外部Tool・package固定、Product scope、AI authorization、Evidence envelope、Project transaction schema、Domain固有runtime semantics
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Core Architecture](core-architecture.md)、[Toolchain／Dependencies](toolchain-dependencies.md)
- 関連文書: [Operation／Planning Candidate Catalog](../appendices/executable-contracts-operation-planning-catalog.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Project State](../03-authoring/project-state.md)、[Memory／Pointers](memory-pointers.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-27

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

共通kindは`requirement | type | operation | state_machine | capability | policy | profile | diagnostic | service`のclosed setとする。Domainが独自kind、別Envelope、別version意味を追加しない。Domain固有意味は共通kindのpayloadとOwner refで表す。

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

<a id="81-project-runtime-entryruntime-scope"></a>

### 8.1 Project Runtime Entry／Runtime Scopeの正規Operation登録

Project Runtime Entry／Runtime Scope Operationの共通成立条件は、Project State ownerが所有するDocument identity、expected revision、Target selector、Scope owner、ChangeSet transactionとexact一致することである。具体MCD record、Policy、Service、Diagnostic、Fixture候補は[Operation／Planning Catalog](../appendices/executable-contracts-operation-planning-catalog.md#81-project-runtime-entryruntime-scopeの正規operation登録)へ分離する。

この節は`service.authoring_command_gateway`が完全登録済みOperationだけを処理し、Provider／MCPが直接Projectを変更しないというauthority境界を所有する。補助Catalogのrecord件数またはIDをactive inventoryと解釈しない。

### 8.1.2 Conditional legacy migration evidence gate

Legacy migration Operationは実在するlegacy Source、complete consumer inventory、Compatibility Change、migration fixture、fresh Qualification、Approvalがすべてmaterializeした場合だけclosed catalogへ追加できる。条件未成立時のcurrent migration Operation集合は空である。具体候補は[補助Catalog](../appendices/executable-contracts-operation-planning-catalog.md#812-conditional-legacy-migration-evidence-gate)を参照する。

### 8.2 Current Installed Product active Operation closure

Installed Productのactive Operation closureは、Product Definition、Contract set、Service allowlist、Policy、Validator、Diagnostic、Target bindingのset equalityから決定する。Markdown上の候補表、未Activation planning record、Provider discovery resultをactive集合へ混入しない。現Repositoryではmaterialized closureがないため、active集合を生成済みとは主張しない。

## 9. State machine定義

State machineはclosed state、initial state、event、guard、transition、terminal state、failure diagnosticを宣言する。Unknown state／eventを無視せず、同じinputから複数transitionを選ばない。Runtime implementationはMCD transition semanticsを再定義しない。

## 10. Capability定義

Capabilityはidentity、maturity、owner、required contract、Target availability、activation condition、qualification requirementを分離する。maturityまたはversionをCapability IDへ埋め込まず、文書の記載だけでactiveにしない。

すべての`McdContractRefV1(kind=capability)`は、同じContract set内の完全なCapability recordへexactly oneで解決しなければならない。Provider descriptor、Game System、Role、Fixture表にIDが現れるだけでは定義にならず、未定義IDを名前やProduct Capabilityから補完しない。

MCD CapabilityとProduct `CapabilityRegistryV1` rowは別のauthorityである。MCD Capabilityは実行Contractの存在と依存を表し、Product rowはTarget別の選択、成熟度表示、Activation evidenceを表す。両者は明示的なversion／hash付きBindingで接続し、同名、prefix、maturity、Targetの一致から相互生成しない。

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

## 17. Language／Runtime projection

C++、TypeScript、Gameplay Cooked binaryは同じType／Operation identityとcanonical fixtureを共有する。Host ABI、compiler padding、JavaScript number coercion、JSON serialization差をwire semanticsへ持ち込まない。

## 18. Round-tripとCross-language同値性

各Typeはcanonical bytes、JSON projection、C++、TypeScript間でvalue、presence、enum、bound、hashが一致するfixtureを持つ。Decode→encodeで別bytesになる、unknown Fieldを保持する、overflowを丸める実装を拒否する。

## 19. Schema変更とMigration

変更はcompatible addition、breaking replacement、offline migration、retirementへ分類する。既存versionをin-place変更せず、Consumer Inventory、Compatibility Change、migration Evidence、rollback／recook条件を束縛する。

## 20. AI向けDiscovery／Execution候補のplanning record（未Activation）

Discovery／Execution planning recordの具体候補は[Operation／Planning Catalog](../appendices/executable-contracts-operation-planning-catalog.md#20-ai向けdiscoveryexecution候補のplanning-record未activation)へ分離する。Planning recordはOperation登録、Provider exposure、authorization、Capability activationを発生させない。

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
- [Model Context Protocol specification](https://modelcontextprotocol.io/specification/)
- [OpenAI Structured Outputs](https://developers.openai.com/api/docs/guides/structured-outputs)
- [Protocol Buffers Encoding](https://protobuf.dev/programming-guides/encoding/)

## 24. 明示的に採用しないもの

- unrestricted scripting runtimeを契約正本にする方式
- Provider Tool SchemaをEngine Sourceへ逆変換する方式
- latest version／display name／pathによるref解決
- unknown Field、unknown enum、missing validatorの黙示受理
- validation成功とstate publicationの同一視
- Domainごとの独自Envelope、独自hash、独自Diagnostic wire format
