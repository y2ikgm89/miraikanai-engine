# Miraikanai Engine Executable Contracts

- 文書ID: mirakan.arch.executable-contracts
- 状態: review
- 正本範囲: MCD、Requirement、Type、Operation、State machine、Capability、Policy、Diagnostic、canonicalization、Contract compiler、C++／TypeScript／MCP／Provider／Cooked projection
- 非正本範囲: 外部Tool・packageのversion／commit／hash／license、Product scope、AI authorization、Evidence envelope、Project transaction schema、Domain固有runtime semantics。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core architecture](core-architecture.md)、[Toolchain／Dependencies](toolchain-dependencies.md)、[Naming／Project layout](naming-project-layout.md)、[C++23 modules](cpp23-modules.md)、[Math／Core utilities](math-core.md)、[Project state](../03-authoring/project-state.md)、[Project Shader](../06-rendering/project-shader.md)
- 外部根拠検証日: 2026-07-23

## 1. 結論

Miraikanai Engineの要件、型、操作、状態遷移、権限、Budget、Diagnosticを、prose、C++ header、TypeScript type、AI Tool Schemaへ別々に手書きしない。Repositoryの`/schemas/mirakan/`に置く**Miraikanai Contract Definition（MCD）**を唯一の機械可読正本とし、次を決定論的に生成する。

- [Toolchain／Dependencies](toolchain-dependencies.md)が固定するJSON Schema dialectによるEngine内部検証Schema。
- C++23 wire type、enum、validator、serializer、dispatch table、Named Module interface、C ABI Header。
- TypeScript strict type、runtime validator、JSON-RPC binding。
- GameplayDefinition／CookedGameplayPackageのbinary descriptor、encoder、decoder。
- 固定MCP protocolのTool `inputSchema`／`outputSchema`。
- OpenAI strict function／Structured Output向けsubset。
- Anthropic Tool向けProvider projection。
- Editor form、Inspector metadata、human-readable reference。
- Game System Catalog、State owner table、dependency graph fragment、System conformance test。
- Schema fixture、round-trip test、state transition conformance test。

Provider向けSchemaは正本ではない。OpenAI、Anthropic、MCP等の対応Dialectやsubsetが異なるため、MCDを一つのProvider形式へ縮退させない。Provider出力はProvider projectionを通過した後も、C++ Command GatewayがMCDの完全な構造制約と意味制約で再検証する。

LODは単一の裸`lod_index`として公開せず、`LodIntentV1`、Domain別Policy、`LodResolutionPlanV1`、`LodQualificationReceiptV1`をMCDへ登録する。`operation.lod.*`はStable IDでない不完全prefixであり、§21.1のexact二候補だけを`not_activated`として予約する。将来そのfamilyをatomic Activationする場合に限り、本規約8節のOperation形式、authority、revision、typed errorへ従い、Generated Artifactまたはruntime選択結果を直接編集するCommandを生成しない。

Anti-aliasも裸のmethod名またはconsole variableとして公開せず、`AntiAliasingIntentV1`、`ResolvedAntiAliasingPlanV1`、closed method／sample enum、互換Predicate、Target QualificationをMCDへ登録する。§20のAnti-alias五Operation候補は`not_activated`でcurrent Tool集合が空である。familyのatomic Activation後だけAIはIntentとbounded query／proposalを扱え、その場合もresolved Plan、Render Graph Pass、Provider activation、Pipeline rebuildを直接編集しない。

## 2. 解決する問題

本規約は次の不具合を防ぐ。

- proseでは必須だがC++ validatorに存在しない。
- C++ enumへ値を追加したがAI Tool Schemaへ反映されない。
- OpenAIでは通るSchemaがMCPまたはClaudeで失敗する。
- AIが存在しないCapability、Field、Command、Error codeを推測する。
- Optional、`null`、default、unknown fieldの意味が言語ごとに異なる。
- State machineの遷移は文書化されているが実装が別の遷移を許す。
- Generated fileを人間またはAIが直接編集し、次の生成で消える。
- Schema更新でProject dataを黙って破棄する。

## 3. AuthorityとArtifact分類

| Artifact | 正本 | 編集方法 |
|---|---|---|
| Requirement、Type、Operation、State Machine、Capability、Game System、Policy、Profile、Provider Profile、Remediation | `/schemas/mirakan/**/*.mirakan.json` | Schema ChangeSetだけ |
| JSON Schema projection | Build treeのgenerated output | 直接編集禁止 |
| C++／TypeScript binding、Cooked binary codec | Build treeのgenerated output | 直接編集禁止 |
| MCP／Provider Tool Schema | Build treeのgenerated output | 直接編集禁止 |
| 人間向けAPI reference | generated docs | 直接編集禁止 |
| 意図、根拠、例、Architecture説明 | Markdown規約／ADR | 人間／AIがReview経由で編集 |
| 手書きsemantic validator | C++／TypeScript source | Requirement IDを付けて編集 |
| Project data | GameSpec／World／Scene／Space／Topology／owner-typed Document／System Implementation Set／GameplayDefinition／Asset metadata | 現在Schemaで検証しChangeSet経由 |

MCDは意図の説明を完全に置き換えない。MCDが機械的な合否、Markdown／ADRが理由とTrade-offを決める。両者が矛盾する場合、実行時合否はMCDに従い、矛盾自体をCI errorにする。

## 4. DirectoryとFile規約

```text
/schemas/
  /mirakan/
    /meta/
      mirakan_contract_v1.schema.json
      mirakan_requirement_v1.schema.json
      mirakan_type_v1.schema.json
      mirakan_operation_v1.schema.json
      mirakan_state_machine_v1.schema.json
      mirakan_capability_v1.schema.json
      mirakan_game_system_v1.schema.json
      mirakan_policy_v1.schema.json
      mirakan_profile_v1.schema.json
      mirakan_provider_profile_v1.schema.json
      mirakan_remediation_v1.schema.json
    /requirements/
    /types/
    /operations/
    /state_machines/
    /capabilities/
    /game_systems/
    /policies/
    /profiles/
    /providers/
    /remediations/
  contract.lock.json
/tools/
  /contract_compiler/
  /contract_lint/
/tests/
  /contracts/
    /fixtures/
      /valid/
      /invalid/
    /golden/
    /roundtrip/
    /provider_conformance/
```

正本FileはRFC 8259 JSON、UTF-8 without BOM、LF、末尾改行ありとする。JSON5、comment、trailing comma、NaN、Infinity、重複keyを禁止する。正本File内のJSON数値literalは±(2^53-1)以内だけを許可する。64-bit整数型のrange境界値やdefaultなどこの範囲を超え得る値はdecimal string表現を必須とし、JSON数値literalで書かない。parse後の値ではIEEE 754 double経由の精度喪失を検出できないため、この範囲検査は数値tokenのraw textに対してcontract lintで行う。人間向け注釈は定義済み`description`、`rationale_refs`、`examples`へ記録する。

全MCDのFile名は正本IDへ`.mirakan.json`を付け、kindに対応するDirectoryへ置く。綴りだけを示す非materialized例は`operations/operation.authoring.apply_changeset.mirakan.json`、`requirements/requirement.product.authoring-roundtrip.mirakan.json`とする。前者は§21.2のexample classでありcurrent／planning Operation登録を意味しない。Requirementだけの変換、略称、legacy連番を作らず、File pathをkindとIDから決定論的に導出する。同じIDを複数Fileへ定義しない。

## 5. MCD共通Envelope

全MCD documentは次のFieldを持つ。

| Field | 型 | 規則 |
|---|---|---|
| `mcd_version` | uint32 | 初期値1 |
| `kind` | enum | `requirement`、`type`、`operation`、`state_machine`、`capability`、`game_system`、`policy`、`profile`、`provider_profile`、`remediation` |
| `id` | string | Kind別Grammarに従う正本ID。Repository全体で一意 |
| `version` | uint32 | 意味変更ごとに増加、0禁止 |
| `status` | enum | `draft`、`active`、`deprecated`、`retired` |
| `title` | UTF-8 string | 人間向け短い名称 |
| `description` | UTF-8 string | 対象と非対象を明示 |
| `owners` | Owner ID array | 1件以上 |
| `requirement_refs` | `McdContractRefV1(kind=requirement)` array | 自身がrequirementの場合は空。bare IDを保存しない |
| `rationale_refs` | ADR／spec anchor array | 最低1件 |
| `since_contract_set` | uint32 | 初回導入Contract set |
| `supersedes` | `{id, version}` array | 置換対象。空可 |
| `tags` | lowercase string array | ASCII昇順、重複不可 |

`id`をKind固有の別Fieldへ二重保存しない。全Kindは`<kind>.<namespace_path>`を使い、Requirementだけの別文法を作らない。`namespace_path`は2～8個のdot区切りsegmentで、各segmentは[Naming／Project Layout §3.2](naming-project-layout.md#32-stable-idとoperation)のkind別`lower-token-path`に従い、1～48文字とする。綴り例は`requirement.product.authoring-roundtrip`、非materialized Operation例`operation.authoring.apply_changeset`、`capability.render.material.toon`、`game_system.extension.feature.combat`、`remediation.authoring.refresh_context`であり、例からcurrent recordを生成しない。

MCDへの外部永続参照を`McdContractRefV1 { id: string, version: uint32, contract_set_hash: SHA-256 }`へ固定する。`id`のkindと参照Fieldが要求するkindは一致し、`version`は同じContract set内で存在して`status=active`でなければならない。Bare IDは固定済み`contract_set_hash`を入力にするEditor／AIのread-only検索だけで使用でき、候補が厳密に1件でなければ解決しない。Project Source、Cooked Artifact、Save、Replay、Receipt、ChangeSetはbare IDまたはruntime numeric IDを永続参照に使用しない。`GameSystemContractRefV1`は`McdContractRefV1`のうちkindが`game_system`である型付きaliasとする。

Contract set自身のhashをContract set内のrecordへ埋め込む循環を禁止する。current正本は次の二段階`ContractSetSnapshotV2`であり、旧`ContractSetSnapshotV1`はoffline migratorの入力だけである。

```text
ContractSetLocalRefV1
  kind: §5のclosed MCD kind
  id: canonical MCD ID
  version: positive uint32

ContractSetMemberKindV2 =
  mcd | diagnostic | trusted_service | validator |
  operation_validator_closure

ContractSetMemberLocalIdentityV2
  member_kind: ContractSetMemberKindV2
  local_identity:
    mcd: ContractSetLocalRefV1
    | diagnostic: DiagnosticLocalRefV1
    | trusted_service: TrustedServiceLocalRefV1
    | validator: ValidatorLocalRefV1
    | operation_validator_closure: OperationValidatorClosureLocalRefV1

ContractSetMemberLocalRecordV2
  member_identity: ContractSetMemberLocalIdentityV2
  canonical_payload:
    当該memberの全normative Field。ただしContract set内の相互参照は
    上記五local identityのいずれかで、set root／外部refを含まない
  member_hash: SHA-256

ContractSetSnapshotV2
  contract_set_id
  snapshot_version: 2
  parent_contract_set_hash: SHA-256 | null
  members[1..65536]: ContractSetMemberLocalRecordV2
  contract_set_hash: SHA-256
```

生成DAGは厳密に`五kindのlocal identity集合を解決 → 全cross-member edgeをlocal ref化 → 各canonical payloadをhash → {member kind, local identity, member hash}列からset rootをhash → 外部refをmaterialize`の順である。member hashのdomain separatorはkindごとに`MIRAKAN_CONTRACT_SET_MCD_MEMBER_V2`、`MIRAKAN_CONTRACT_SET_DIAGNOSTIC_MEMBER_V2`、`MIRAKAN_CONTRACT_SET_TRUSTED_SERVICE_MEMBER_V2`、`MIRAKAN_CONTRACT_SET_VALIDATOR_MEMBER_V2`、`MIRAKAN_CONTRACT_SET_OPERATION_VALIDATOR_CLOSURE_MEMBER_V2`とし、`member_hash = SHA-256(domain || uint32_be(len(canonical bytes excluding member_hash)) || canonical bytes)`で計算する。`contract_set_hash = SHA-256(ASCII "MIRAKAN_CONTRACT_SET_SNAPSHOT_V2" || uint32_be(len(snapshot preimage)) || snapshot preimage)`とし、snapshot preimageはID、snapshot version、parent、member count、`member_kind`の上記enum順、同kind内のlocal identity canonical byte順へstrict sortした`{member_kind, local_identity, member_hash}`だけを持つ。duplicate identity、same identity／different hash、非canonical order、未解決local edgeを拒否する。`member_hash`自身、`contract_set_hash`自身、外部`McdContractRefV1`、`DiagnosticCodeRefV1`、`TrustedServiceRefV1`、`ValidatorRecordRefV1`、`OperationValidatorClosureRefV1`はどのlocal hash preimageにも入れない。

Operation→Service、Service→allowed Operation、Operation→Validator closure、closure→Operation、Validator→Type／Diagnosticのような相互edgeはすべてsnapshot内部で上記local refを使う。全member hashとset root確定後にだけ、Readerは`McdContractRefV1`、`DiagnosticCodeRefV1`、`TrustedServiceRefV1`、`ValidatorRecordRefV1`、`OperationValidatorClosureRefV1`をmaterializeする。外部refはlocal memberまたはrootへfeed backしない。したがってMCD、Diagnostic、Service allowlist、Validator、Validator closureのどのnormative byteが変わっても該当member hashとset rootが変わり、hash fixed pointは発生しない。旧Snapshotを参照するretained Project revision、Save、Replay、Artifact、Receiptの期間中は旧rootを保持し、欠落、root不一致、local ref未解決、member hash不一致、inactive targetをfail closedとする。current setへの移行は明示offline migratorが新revisionと新external refを生成し、旧objectをin-place更新しない。

Derived Artifactへの永続参照は`ArtifactRefV1 { artifact_kind: string, schema_version: uint32, sha256: SHA-256 }`へ固定する。本型の構造定義は本節だけが正本であり、Domain文書はexact refを消費してFieldを再定義しない。

`schema_version`という名のFieldはMCD全域で`uint32`固定とし、本規則を全Domain文書共通の正本とする。SemVer等の互換表現が必要な場合は`schema_version`へ文字列を載せず、`format_version`等の別名Fieldを別途定義する。

`status=deprecated`は新規利用を拒否するが、offline migratorが旧Projectを読むための入力Schemaだけに残せる。Runtime、Editor、Game codeへdeprecated branchを生成しない。`retired`はcurrent Contract setの生成対象外である。

## 6. Requirement定義

Requirementは次を必須とする。

| Field | 型／値 |
|---|---|
| 共通`id` | `requirement.<namespace_path>`。Naming正本のkind別grammarに従う |
| `normative_level` | `must`、`must_not`、`should`、`may` |
| `priority` | `blocking`、`high`、`medium`、`low` |
| `statement` | 一つの検証可能な規範文 |
| `scope` | Artifact／Target／Phase ID array |
| `verification_methods` | Gate ID array、1件以上 |
| `acceptance_criteria` | predicate IDまたは数値条件array |
| `failure_code` | Diagnostic code |
| `source_refs` | 外部一次資料または内部決定の参照 |
| `introduced_by` | ADR／ChangeSet ID |

一つのRequirementへ複数の独立した「かつ」を詰め込まない。別々に失敗し得る条件はIDを分ける。`must`／`must_not`に自動または人間ReviewのVerification methodがない定義を拒否する。

外部仕様の事実とMiraikanaiの選択を区別するため、`source_refs[].authority`を`external_normative`、`external_guidance`、`project_normative`、`evidence`のいずれかにする。

## 7. Type system

### 7.1 許可する型

| MCD type | 説明 |
|---|---|
| `bool` | `true`／`false` |
| `int32`、`uint32`、`int64`、`uint64` | 範囲を固定した整数 |
| `float32`、`float64` | finite値だけ |
| `decimal_string` | 浮動小数点へ丸めてはならない量の正規decimal文字列 |
| `string` | UTF-8、長さ／pattern制約 |
| `bytes_base64url` | RFC 4648 URL-safe alphabet、paddingなし |
| `blob_ref` | Artifact ID、revision、SHA-256、size、media type |
| `timestamp` | RFC 3339 UTC、秒以下桁数はType constraintで固定し、Profile間で精度を変えない |
| `duration_ns` | signed int64 nanosecond |
| `enum` | string-backed、unknown値禁止 |
| `struct` | named field集合 |
| `tagged_union` | 明示discriminator付きunion |
| `array` | 順序あり、length上限必須 |
| `set` | canonical sort規則を持つ重複なしarray |
| `map` | canonical string／closed enum keyと最大件数を固定 |
| `nullable<T>` | 値または`null` |
| `handle` | ID＋generation。pointerではない |
| `asset_ref` | Asset ID＋revision constraint |

`any`、暗黙変換、untagged union、pointer、memory address、platform handle、unbounded collectionを禁止する。Wire上の64-bit整数はProvider／JavaScriptの安全整数範囲を超え得るため、JSON projectionではcanonical decimal stringへ変換し、C++ bindingで範囲検査して整数へ戻す。

`decimal_string`は符号`-`、integer、任意fractionだけを許可し、`+`、指数、不要な先頭0、末尾fraction 0、`-0`を禁止する。Type constraintは`max_fraction_digits`を固定する。同じ数値の`1.2`と`1.20`を別値にしない。金額やFixed-point simulationで固定scale／最小単位が必要な場合は、通貨または単位を持つ専用Typeでinteger minor unitを`int64`／`uint64`として保存し、表示用decimal文字列を正本にしない。`set`のWire表現は各要素のJCS byte列をunsigned byte lexicographic順に並べる。`map`のJSON object keyはUnicode NFC後のUTF-8 byte順で一意にし、Provider projectionがdynamic propertyを表せない場合は`[{key,value}]`へ可逆変換する。この変換をProjection Manifestへ記録する。

### 7.2 Field定義

`struct.fields[]`は次を持つ。

- `field_id`: struct内で不変のuint32。
- `name`: lowercase snake_case。
- `type_ref`: exact Type IDとversion。
- `presence`: `required`または`optional`。
- `default`: 定数またはなし。
- `constraints`: length、range、pattern、cardinality、unit、coordinate space、frame、normalization、finite policy、canonicalization。
- `stability`: `stable_id`、`revision_bound`、`ephemeral`。
- `sensitivity`: `public`、`project_private`、`restricted`、`secret`。
- `description`。

`optional`はFieldの`presence`でありTypeではない。`nullable<T>`はFieldが存在した時の値Domainであり、両者を同義にしない。DefaultはField欠落時だけ適用し、明示`null`へ適用しない。AI Provider projectionがOptionalを表せない場合、optionalかつnon-nullableのFieldだけはrequired＋`nullable<T>`へ投影し、`null`を「欠落」へ逆変換できる。optionalかつ`nullable<T>`、または欠落と`null`を区別するFieldは、`{"present": bool, "value": nullable<T>}`の明示presence wrapperを必須とする。`present=false`では`value=null`だけ、`present=true`では元のTのDomainだけを許可する。Gateway受信後にMCDのpresence semanticsへ戻し、変換RuleをProjection Manifestへ記録する。

### 7.3 数値と単位

物理量は裸の`float`または意味なしvectorにしない。`semantic_role`、`dimension`、`scalar_type`、`unit`、`valid_range`、`coordinate_space`、`frame_kind`、`normalization`、`precision`、`finite_policy`、`canonicalization`、`wire_layout`を定義する。Position、Displacement、Direction、UnitDirection、Velocity、Scale、Quaternion、Transform、Color、UVを別Typeとして表し、初期Type catalogとC++ storage／semantic境界はMath／Core Utilities規約を正本とする。角度はAPIごとにdegree／radianを混在させず、正規Wireはradian、Editor表示だけdegreeを許可する。Lengthはmeter、timeはsecondまたは明示`duration_ns`、colorはlinear／sRGBを型で分ける。

浮動小数点の`-0`は`0`へ正規化し、NaN／Infinityを拒否する。Deterministic hash対象のfloatはIEEE 754 bit patternを直接JSON化せず、Typeごとに定義した最短round-trip decimalへ正規化する。

## 8. Operation定義

本書はMCD Operationの共通Envelope、projection、Contract compilerによる生成規則だけを所有する。`ProjectChangeSetV1`のdomain schema、Operation意味、transaction／Commit規則は[Project state](../03-authoring/project-state.md#5-projectchangesetv1)だけが所有し、本書はsuffixなしaliasまたは別Envelopeを定義しない。

各Operationは次を持つ。

| Field | 内容 |
|---|---|
| 共通`id` | 全MCDと同じ`<kind>.<lower-token-path>`。Operationでは先頭が`operation`で、versionをID文字列へ埋め込まない |
| `operation_kind` | `query`、`command`、`event`、`job` |
| `input_type`／`output_type` | `McdContractRefV1(kind=type)` |
| `authority` | `TrustedServiceRefV1 {service_id, service_version, service_content_hash}` |
| `risk_class` | R0からR5 |
| `side_effects` | Authoring、Source、Network、Process、Release等の集合 |
| `idempotency` | `pure`、`idempotent_with_key`、`non_idempotent` |
| `transaction` | `none`、`read_snapshot`、`authoring_changeset`、`source_promotion` |
| `preconditions` | pure evaluationの`McdContractRefV1(kind=policy)` array |
| `postconditions` | pure evaluationの`McdContractRefV1(kind=policy)` array |
| `errors` | `DiagnosticCodeRefV1`のclosed set |
| `timeout_ms` | uint32、0禁止 |
| `rate_limit_policy` | `McdContractRefV1(kind=policy)` |
| `audit_level` | `metadata`、`full_redacted`、`restricted` |
| `provider_exposure` | Provider／MCPへ公開可能か |
| `receipt_type` | `McdContractRefV1(kind=type)` |

未列挙Exception、stringだけのerror、部分成功を禁止する。部分結果が必要なOperationは、成功項目と失敗項目を型付きResultとして明示する。Commandはexpected base revisionをInputへ必須とし、stale revisionを`MIRAKAN-CONFLICT-REVISION_MISMATCH`で拒否する。

MCPへ公開するOperationは`provider_exposure=mcp_proposal`に限定する。正規Commit、Approval発行、Promotion、Releaseは`trusted_internal`とし、Provider projectionを生成しない。

Authoring Typeのfieldは`mutability = immutable | human_mutable | ai_mutable`と、変更可能な完全登録済み外側MCD Operation ID集合を持つ。Contract compilerは各`ai_mutable` fieldについて、外側MCD Operationのnamed inputからtyped `ProjectChangePrimitiveV1` branchへの到達性を全件検査し、coverageが100%でなければ該当CapabilityのProvider／MCP projectionを生成しない。自由形式のJSON Pointer write、任意path write、外側MCD Operationを迂回するSource writeをcoverageとして数えない。

### 8.1 Project Runtime Entry／Runtime Scopeの正規Operation登録

[Project state](../03-authoring/project-state.md#312-runtime-entryのclosed-operation-catalog)が所有する七Operationと[Gameplay programming model](../03-authoring/gameplay-programming-model.md#312-scope依存recordとoffline-migration)が所有する一Operationを、次の`McdOperationContractV1`で登録する。八recordは全Fieldを明示し、Registry compilerは既定値、別行参照、説明語だけのerror、裸IDを補完しない。

```text
McdOperationContractV1
  MCD common envelope: §5の全Field
  operation_kind: query | command | event | job
  input_type: McdContractRefV1(kind=type)
  output_type: McdContractRefV1(kind=type)
  authority: TrustedServiceRefV1 {service_id, service_version, service_content_hash}
  risk_class: R0 | R1 | R2 | R3 | R4 | R5
  side_effects[0..16]: closed SideEffectKindV1
  idempotency: pure | idempotent_with_key | non_idempotent
  transaction: none | read_snapshot | authoring_changeset | source_promotion
  preconditions[1..16]: McdContractRefV1(kind=policy)
  postconditions[1..16]: McdContractRefV1(kind=policy)
  errors[1..64]: DiagnosticCodeRefV1
  validator_closure_ref: OperationValidatorClosureRefV1
  timeout_ms: uint32
  rate_limit_policy: McdContractRefV1(kind=policy)
  audit_level: metadata | full_redacted | restricted
  provider_exposure: none | mcp_proposal | trusted_internal
  receipt_type: McdContractRefV1(kind=type)

SideEffectKindV1 =
  authoring | source | network | process | release

TrustedServiceRefV1
  service_id
  service_version: uint32
  service_content_hash: SHA-256

TrustedServiceLocalRefV1
  service_id
  service_version: uint32

TrustedServiceLocalRecordV2
  local_ref: TrustedServiceLocalRefV1
  executable_identity_ref/hash
  allowed_operation_local_refs[1..4096]: ContractSetLocalRefV1(kind=operation)
  authority_capability_local_refs[1..64]: ContractSetLocalRefV1(kind=capability)
  isolation_profile_local_ref: ContractSetLocalRefV1(kind=profile)
  service_local_content_hash: SHA-256

TrustedServiceRecordV1
  service_id
  service_version: uint32
  executable_identity_ref/hash
  allowed_operation_refs[1..4096]: McdContractRefV1(kind=operation)
  authority_capability_refs[1..64]: McdContractRefV1(kind=capability)
  isolation_profile_ref: McdContractRefV1(kind=profile)
  service_content_hash: SHA-256

TrustedServiceRegistryV2
  registry_id: trusted_service.registry.active
  registry_version: 2
  registry_content_hash: SHA-256
  records[1..1024]: TrustedServiceLocalRecordV2

OperationValidatorClosureRefV1
  closure_id
  closure_version: uint32
  closure_content_hash: SHA-256

OperationValidatorClosureLocalRefV1
  closure_id
  closure_version: uint32

ValidatorLocalRefV1
  validator_id
  validator_version: uint32

DiagnosticLocalRefV1
  diagnostic_id
  diagnostic_version: uint32

DiagnosticOwnerLocalRefV1
  owner_id
  owner_revision: positive uint64
  owner_content_hash: SHA-256

DiagnosticLocalRecordV2
  diagnostic_local_ref: DiagnosticLocalRefV1
  owner_local_ref: DiagnosticOwnerLocalRefV1
  code: MIRAKAN-<DOMAIN>-<CONDITION>
  severity: info | warning | error | blocking
  category:
    schema | semantic | permission | conflict | build | test |
    performance | security | provider | infrastructure
  message_key
  requirement_local_refs[0..64]:
    ContractSetLocalRefV1(kind=requirement)
  retryability: never | after_input | after_change | transient
  diagnostic_local_content_hash: SHA-256

ValidatorRecordRefV1
  validator_id
  validator_version: uint32
  validator_content_hash: SHA-256

ValidatorLocalRecordV2
  validator_local_ref: ValidatorLocalRefV1
  implementation_artifact_ref/hash
  input_type_local_ref: ContractSetLocalRefV1(kind=type)
  error_local_refs[1..64]: DiagnosticLocalRefV1
  validator_local_content_hash: SHA-256

ValidatorRecordV1
  validator_id
  validator_version: uint32
  materialized_from_contract_set_hash: SHA-256
  implementation_artifact_ref/hash
  input_type_ref: McdContractRefV1(kind=type)
  error_refs[1..64]: DiagnosticCodeRefV1
  validator_content_hash: SHA-256

OperationValidatorClosureLocalRecordV2
  closure_local_ref: OperationValidatorClosureLocalRefV1
  operation_local_ref: ContractSetLocalRefV1(kind=operation)
  validator_local_refs[1..64]: ValidatorLocalRefV1
  reachable_error_local_refs[1..64]: DiagnosticLocalRefV1
  reachable_error_set_hash: SHA-256
  closure_local_record_hash: SHA-256

OperationValidatorClosureV1
  closure_id
  closure_version: uint32
  operation_ref: McdContractRefV1(kind=operation)
  validator_refs[1..64]: exact ValidatorRecordRefV1
  reachable_error_refs[1..64]: DiagnosticCodeRefV1
  reachable_error_set_hash: SHA-256
  closure_content_hash: SHA-256

OperationPreconditionEvaluationInputV1
  operation_ref: McdContractRefV1(kind=operation)
  operation_input_ref/hash
  operation_intent_hash
  request_hash
  before_snapshot_refs[1..64]/hashes
  mutation_authorization_binding:
    exact MutationAuthorizationBindingV2
    | canonical omission: only R0/R1 non-mutation

PreparedCandidateRefV1
  candidate_id
  candidate_schema_ref: McdContractRefV1(kind=type)
  candidate_content_hash: SHA-256

PreparedCandidateV1
  candidate_id
  candidate_schema_ref: McdContractRefV1(kind=type)
  staging_root_hash
  before_state_ref/hash
  proposed_after_state_ref/hash
  prepared_artifact_refs[0..4096]/hashes
  candidate_content_hash: SHA-256

PublishedReceiptMaterializationPolicyV1
  policy_ref: McdContractRefV1(kind=policy)
  signing_profile_ref/hash: PublishedReceiptSigningProfileRefV1
  signer_subject_ref/hash
  signer_role_ref/hash
  key_id
  public_key_registry_snapshot_ref/hash
  key_retention_policy_ref/hash
  recovery_policy_ref/hash
  receipt_canonical_schema_ref: McdContractRefV1(kind=type)
  receipt_store_namespace_ref/hash
  policy_content_hash: SHA-256

PublishedReceiptSigningProfileRefV1
  profile_id
  profile_version: positive uint32
  profile_content_hash: SHA-256

PublishedReceiptSigningProfileV1
  profile_id: signing_profile.operation_receipt.ecdsa_p256_rfc6979
  profile_version: 1
  signature_algorithm: ecdsa-p256-sha256
  signature_format: ieee-p1363-raw
  nonce_derivation: rfc6979-sha256
  low_s_required: true
  canonical_subject: JCS(PublishedDomainReceiptPayloadV2)
  issued_at_source: exact pre-marker materialization context issued_at
  revocation_snapshot_source:
    exact pre-marker materialization context revocation_snapshot_ref/hash
  retry_rule: same subject/key/context produces byte-identical wrapper
  profile_content_hash: SHA-256

PublishedReceiptMaterializationContextRefV1
  context_id
  context_content_hash: SHA-256

PublishedReceiptMaterializationContextV1
  context_id
  operation_ref: McdContractRefV1(kind=operation)
  operation_intent_hash
  request_hash
  idempotency_key
  before_state_ref/hash
  staged_after_state_ref/hash
  issued_at
  revocation_snapshot_ref/hash
  materialization_policy_ref:
    exact {policy_ref, policy_content_hash}
  signing_profile_ref/hash
  signer_subject_ref/hash
  signer_role_ref/hash
  key_id
  public_key_registry_snapshot_ref/hash
  context_content_hash: SHA-256

PublishedReceiptMaterializationKeyPayloadV1
  operation_ref: McdContractRefV1(kind=operation)
  operation_intent_hash
  request_hash
  idempotency_key
  before_state_ref/hash
  staged_after_state_ref/hash
  receipt_type_ref: McdContractRefV1(kind=type)
  prepared_payload_count: 3
  prepared_payloads[3]:
    exact {payload_type_ref, payload_content_ref, payload_content_hash}
  private_commit_marker_hash
  materialization_policy_ref:
    exact {policy_ref, policy_content_hash}
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1

AtomicCommitPlanPayloadV1
  operation_ref: McdContractRefV1(kind=operation)
  operation_intent_hash
  request_hash
  idempotency_key
  prepared_candidate_ref: PreparedCandidateRefV1
  before_state_ref/hash
  proposed_after_state_ref/hash
  prepared_payload_count: 3
  prepared_payloads[3]:
    exact {payload_type_ref, payload_content_ref, payload_content_hash}
  published_receipt_materialization_policy_ref:
    exact {policy_ref, policy_content_hash}
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1

PreparedCommitEnvelopeV1
  operation_ref: McdContractRefV1(kind=operation)
  operation_intent_hash
  request_hash
  idempotency_key
  prepared_candidate_ref: PreparedCandidateRefV1
  before_state_ref/hash
  staged_after_state_ref/hash
  preview_receipt_payload_ref/hash
  validation_receipt_payload_ref/hash
  prepared_domain_receipt_payload_ref/hash
  published_receipt_materialization_policy_ref:
    exact {policy_ref, policy_content_hash}
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1
  atomic_commit_plan_hash
  envelope_hash

PreparedReceiptPublicationBindingV1
  operation_ref: McdContractRefV1(kind=operation)
  operation_intent_hash
  request_hash
  idempotency_key
  before_state_ref/hash
  staged_after_state_ref/hash
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1

PreparedPreviewReceiptPayloadV1
  publication_binding: exact PreparedReceiptPublicationBindingV1
  preview_result_ref/hash
  disposition: preview_ready
  prepared_payload_hash: SHA-256

PreparedValidationReceiptPayloadV1
  publication_binding: exact PreparedReceiptPublicationBindingV1
  validation_result_ref/hash
  disposition: satisfied
  diagnostic_refs: []
  prepared_payload_hash: SHA-256

OperationPostconditionEvaluationInputV2
  operation_ref: McdContractRefV1(kind=operation)
  request_hash
  prepared_operation_output_ref/hash
  before_snapshot_refs[1..64]/hashes
  unpublished_staging_snapshot_refs[1..64]/hashes
  prepared_commit_envelope_ref/hash

StagedPostconditionReceiptV1
  evaluated_input_hash
  prepared_candidate_ref: PreparedCandidateRefV1
  prepared_commit_envelope_ref/hash
  disposition: satisfied | rejected
  predicate_evidence_hash | diagnostics[1..64]
  receipt_payload_hash

PrivateDurableCommitMarkerV1
  marker_id
  operation_ref
  operation_intent_hash
  request_hash
  idempotency_key
  prepared_commit_envelope_ref/hash
  before_state_ref/hash
  staged_after_state_ref/hash
  staged_prepared_receipt_payload_count: 3
  staged_prepared_receipt_payloads[3]:
    exact {payload_type_ref, payload_content_ref, payload_content_hash}
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1
  staged_postcondition_receipt_ref/hash
  visibility: private_internal
  marker_hash

PublishedDomainReceiptPayloadV2
  prepared_domain_receipt_payload_ref/hash
  private_commit_marker_hash
  operation_ref
  operation_intent_hash
  request_hash
  idempotency_key
  before_state_ref/hash
  after_state_ref/hash
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1
  issued_at
  revocation_snapshot_ref/hash

PublishedDomainReceiptV2
  payload: PublishedDomainReceiptPayloadV2
  signed_record:
    exact MirakanSignedRecordV1

PublicPublicationMarkerV1
  publication_id
  operation_ref
  operation_intent_hash
  request_hash
  idempotency_key
  private_commit_marker_hash
  signed_domain_receipt_ref/hash
  before_state_ref/hash
  public_after_state_ref/hash
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1
  expected_previous_publication_ref/hash | null
  publication_sequence: positive uint64
  marker_hash

OperationPredicateResultV1
  disposition: satisfied | rejected
  satisfied:
    evaluated_input_hash
    predicate_evidence_hash
  rejected:
    diagnostics[1..64]: DiagnosticCodeRefV1

OperationRateLimitPolicyV1
  policy_ref: McdContractRefV1(kind=policy)
  scope: project | principal_project
  window_ns: uint64
  max_requests: uint32
  burst: uint32
  exceeded_error_ref: DiagnosticCodeRefV1
```

`DiagnosticOwnerLocalRefV1.owner_content_hash`はASCII `MIRAKAN_DIAGNOSTIC_OWNER_LOCAL_IDENTITY_V1`、owner ID、owner revisionをField順にlength-frameしたcanonical bytesから計算する。これはContract set rootを含まないimmutable local identityであり、Owner identity自身もContract set hash、Diagnostic ref、Diagnostic Registry hashを含めない。`DiagnosticLocalRecordV2.diagnostic_local_content_hash`はASCII `MIRAKAN_DIAGNOSTIC_LOCAL_RECORD_V2`と、同Fieldだけを除きOwner identityとRequirement edgeをLocalRefのまま保持した完成local recordのlength-framed canonical bytesから計算する。Contract setのDiagnostic `member_hash`はこの完成local recordを別domain `MIRAKAN_CONTRACT_SET_DIAGNOSTIC_MEMBER_V2`でhashする別値である。set root確定後、外部`DiagnosticCodeRecordV1.diagnostic_content_hash`はOwnerをexact external owner ref、Requirement edgeを`contract_set_hash`付き`McdContractRefV1`へ投影した外部recordから§12の規則で計算する第三の値であり、local hashまたはmember hashとの等値を要求しない。外部record／hash／refをlocal payload、member hash、set rootへfeed backしない。

`TrustedServiceLocalRecordV2.service_local_content_hash`はASCII `MIRAKAN_TRUSTED_SERVICE_LOCAL_RECORD_V2`と同Fieldだけを除く完成local recordから計算する。Contract Set `member_hash`は別domain `MIRAKAN_CONTRACT_SET_TRUSTED_SERVICE_MEMBER_V2`と、このlocal hashを含む完成local recordから計算する別値であり、両hashのbyte equalityを要求しない。Local Registry hashはASCII `MIRAKAN_TRUSTED_SERVICE_REGISTRY_V2`、Registry ID／version、record count、`service_id`／version順local record bytesを`uint32_be` length framingして計算する。set root確定後だけ、Operation／Capability／isolation profileのLocalRefをroot付きMCD refへ投影して`TrustedServiceRecordV1`を作り、`service_content_hash`をASCII `MIRAKAN_TRUSTED_SERVICE_RECORD_V1`と自己Fieldを除くexternal recordから計算する。`service_local_content_hash → Service member_hash／Local Registry hash → Contract set root → external service_content_hash／TrustedServiceRefV1`の一方向DAGとし、external record／hash／refをlocal record、member、Registry、rootへ戻さない。duplicate／stale／hash mismatch、OperationのLocalRefがServiceの`allowed_operation_local_refs[]`にない状態を拒否する。Core-only初期Registryの二recordは次のcanonical recordであり、`executable_identity_ref/hash`、`isolation_profile_local_ref`、Capability LocalRef、allowlistを省略しない。

| Service LocalRef | executable identity | exact allowed Operation LocalRefs | authority Capability LocalRefs | isolation profile LocalRef |
|---|---|---|---|---|
| `{service.authoring_command_gateway,1}` | `{artifact.service.authoring_command_gateway,1,artifact_hash}` | `{operation.project.runtime_entry.create,1}; {operation.project.runtime_entry.update,1}; {operation.project.runtime_target_selector.create,1}; {operation.project.runtime_target_selector.update,1}; {operation.project.runtime_entry_activation_policy.create,1}; {operation.project.runtime_entry_activation_policy.update,1}; {operation.project.runtime_entry.migrate_root_scene,1}` | `{capability.authoring.command_gateway,1}` | `{profile.isolation.authoring_command_gateway,1}` |
| `{service.offline_project_migrator,1}` | `{artifact.service.offline_project_migrator,1,artifact_hash}` | `{operation.runtime_scope.migrate_game_system,1}` | `{capability.authoring.offline_migration,1}` | `{profile.isolation.offline_project_migrator,1}` |

表のOperation集合は本節の初期八Operationを導入するbootstrap seedであり、current merged snapshotではない。他Domain／Pack由来Operationを暗黙追加せず、§8.2のcurrent merged二recordを正本にする。外部`TrustedServiceRefV1 {service_id,service_version,service_content_hash}`はset root確定後にmaterializeするだけでSnapshot preimageへ戻さない。DomainまたはPack Operationをactivate／removeするContract set transactionは、Owner ManifestのOperation LocalRef集合と当該OwnerがServiceへ寄与するallowlist LocalRef集合のset equalityを検査し、Service local record、Operation local record、set rootを一方向に再生成する。runtime中のallowlist mutation、旧Service hashと新Operationの混在、prefix／owner名による暗黙許可を禁止する。

predicate IO/resultのexact MCD Typeは`type.operation.precondition_evaluation_input` version 1、`type.operation.postcondition_evaluation_input` version 2、`type.operation.predicate_result` version 1である。Prepared auxiliary payloadのexact Typeは`type.operation.prepared_preview_receipt_payload` version 1と`type.operation.prepared_validation_receipt_payload` version 1である。旧postcondition input v1はoffline Receipt auditだけに残し、current Operation／Policy／projectionから参照しない。五current Type recordは上記Field、presence、boundを完全投影し、bare snapshot ID、評価中のRegistry query、時計、network、mutable pointer、published Commit Receiptを許可しない。Prepared二Typeの共通Envelopeはそれぞれ`mcd_version=1; kind=type; id=<上記ID>; version=1; status=active; owners=[owner.core.project_state]; requirement_refs=[]; rationale_refs=[mirakan.arch.executable-contracts#8-transactioncommitreceipt]; since_contract_set=1; supersedes=[]`を持ち、title／description／tagsはID固有、payload schemaは対応するnamed `PreparedPreviewReceiptPayloadV1`／`PreparedValidationReceiptPayloadV1`の全Field、presence、boundをexact投影する。二Type LocalRef／local record hash／member hashをContract Setへ含め、root確定後だけ同root付きMCD Type refへmaterializeする。rate-limit policyは`policy.authoring.runtime_entry.rate_limit` version 1と`policy.authoring.runtime_scope_migration.rate_limit` version 1をactive MCD recordとして登録し、前者は`scope=principal_project, window_ns=60000000000, max_requests=120, burst=20`、後者は`scope=project, window_ns=60000000000, max_requests=4, burst=1`、どちらも`exceeded_error_ref={diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash}`を持つ。Policy共通Envelope、payload、Type五record、Service二recordのmissing／wrong-kind／stale version／stale Contract set／content hash mismatchをOperation Registry compile errorにする。

`PreparedCandidateV1.candidate_content_hash`はASCII `MIRAKAN_PREPARED_CANDIDATE_V1`、candidate hash自身を除く全Fieldのcanonical bytesを`uint32_be` length framingして計算する。`PreparedCandidateRefV1`は完成candidateのID、schema ref、content hashの三者をexactにbindし、Record自身へhash付きRefを埋め戻さない。Staging root、before／proposed-after state、prepared Artifact集合の一Fieldでも変われば別Refになり、candidate IDだけ、latest candidate、別Staging rootの同IDへfallbackしない。`PublishedReceiptSigningProfileV1.profile_content_hash`はASCII `MIRAKAN_PUBLISHED_RECEIPT_SIGNING_PROFILE_V1`、`PublishedReceiptMaterializationPolicyV1.policy_content_hash`はASCII `MIRAKAN_PUBLISHED_RECEIPT_MATERIALIZATION_POLICY_V1`、`PublishedReceiptMaterializationContextV1.context_content_hash`はASCII `MIRAKAN_PUBLISHED_RECEIPT_MATERIALIZATION_CONTEXT_V1`と各自己Fieldを除くcanonical bytesから計算する。`context_id`はOperation ref、operation intent hash、request hash、idempotency key、before／staged-after state、materialization policy、signing profile、Signer／Role／Key、Public key registry snapshot、issued-at、revocation snapshotの同じcanonical bytesから決定論的に導出し、caller supplied UUIDや時計の再読取を使わない。Contextのpolicy／profile／Signer／Role／Key／Public key snapshotは解決したMaterialization Policyの同Fieldとbyte equalityでなければならない。Key retention policyは署名済みwrapper bytes、検証用public key、profile、発行時revocation snapshotをProject retention期間以上保持し、private key消失時に別Keyで再署名しない。Recovery policyは保存済みwrapperのbyte-exact復旧またはfail-closedだけを許し、issued-at更新、revocation snapshot差替え、alternate signatureを禁止する。

Commandの唯一のpublish順は`Preview → Validation → pre-marker materialization context → PreparedCandidate／PreparedCommitEnvelope → staged postcondition → private durable commit marker → canonical signed Domain Receipt store → public publication marker＋after state`である。`AtomicCommitPlanPayloadV1.prepared_payload_count`は配列長と一致し、membersはtype ref、content ref、hash順へstrict sortしてduplicate／same-ref different-hashを拒否する。exact三件は`type.operation.prepared_preview_receipt_payload@1`へ解決する`PreparedPreviewReceiptPayloadV1`、`type.operation.prepared_validation_receipt_payload@1`へ解決する`PreparedValidationReceiptPayloadV1`、当該Operationのnamed Domain prepared receipt payloadであり、Plan／Envelope／private Markerでtype ref／content ref／content hashのset equalityを必須にする。三payloadはすべてexact `publication_binding: PreparedReceiptPublicationBindingV1`を持つ。Preview／Validationの`prepared_payload_hash`はそれぞれASCII `MIRAKAN_PREPARED_PREVIEW_RECEIPT_PAYLOAD_V1`／`MIRAKAN_PREPARED_VALIDATION_RECEIPT_PAYLOAD_V1`と自己Fieldを除くcanonical bytesから計算する。Validation rejected、Diagnostic非空、missing preview resultのpayloadはPlanへ入れず、public objectを0件にする。Domain固有の`before_project_ref`／`after_project_ref`等はbindingの`before_state_ref/hash`／`staged_after_state_ref/hash`へ一意に投影し、duplicate Fieldもbyte equalityにする。Envelopeの三payload refsはPlan payload集合とset equality、EnvelopeのCandidate／policy／ContextはPlanの同Fieldとexact equalityでなければならない。Operation ref、intent hash、request hash、idempotency key、before state、staged-after state、ContextはPlan、Envelope、Context、三Prepared publication binding、private Markerの全てでbyte equalityにする。Candidateのbefore／proposed-after stateも同じbefore／staged-after stateへexact解決する。`atomic_commit_plan_hash = SHA-256(ASCII "MIRAKAN_ATOMIC_COMMIT_PLAN_V1" || uint32_be(len(MCD canonical plan payload bytes)) || plan payload bytes)`とし、Prepared Envelope自身、staged postcondition Receipt、private Marker、signed Receipt、public Markerをplan payloadへ含めない。`envelope_hash`はASCII `MIRAKAN_PREPARED_COMMIT_ENVELOPE_V1`、`receipt_payload_hash`はASCII `MIRAKAN_STAGED_POSTCONDITION_RECEIPT_V1`、private `marker_hash`はASCII `MIRAKAN_PRIVATE_DURABLE_COMMIT_MARKER_V1`、public `marker_hash`はASCII `MIRAKAN_PUBLIC_PUBLICATION_MARKER_V1`と各自己Fieldを除くcanonical bytesをlength-frameして計算する。postconditionは未発行Staging、Prepared Receipt payload、既に固定されたContextだけを読み、clock／revocation registryを再queryせず、private／public Marker、公開後state、最終Receiptを入力にしない。type refのwrong version／root／kind、preview↔validation type swap、同contentを別typeで再利用するfixtureを各一原因で拒否する。

Gatewayはpostcondition success後、Prepared Envelope、Preview／Validation／Domain semantic payload、Staged Postcondition Receipt、staged after state、`PublishedReceiptMaterializationContextV1`、`PrivateDurableCommitMarkerV1`を外部readerから到達不能なprivate durable transactionへcommitする。この時点でProject／Registry／Runtime current head、Document index、provider-visible Resultは一切変えない。private Marker preimageはPrepared Envelope ref／hash、staged postcondition ref／hash、prepared payload ref／hash、Context ref／hashを束縛するが、Marker自身、signed Receipt、public Markerを含めない。readback後、`PublishedDomainReceiptPayloadV2`を完成し、AI Verification／Provenanceが所有する`MirakanSignedRecordV1`をexact `$ref`する`PublishedDomainReceiptV2` wrapperをreceipt storeへput-if-absentする。inline signer／key／algorithm／signature FieldをDomain receiptに定義しない。Published payloadのprepared payload ref／hashはprivate Markerのexact三件集合中のDomain memberへ解決し、Operation ref、intent hash、request hash、idempotency key、before state、after state、Contextは解決したDomain publication bindingおよびprivate Markerの同Fieldとbyte equalityにする。Published `after_state_ref/hash`はprivate Markerの`staged_after_state_ref/hash`を名前だけ変えた同一値である。`issued_at`／`revocation_snapshot_ref/hash`は解決したContextの同Fieldとbyte equalityでなければならない。`signed_record.purpose=operation_domain_receipt`、`subject_sha256=SHA-256(JCS(payload))`、signer subject／Role／Keyは完成payloadとContext／Materialization Policyへbyte equalityにする。Published payloadは自己`payload_hash` Fieldを持たず、signed subject hashだけを完成JCS payloadから計算する。

署名済みwrapperの保存／readback後だけ、Gatewayは`PublicPublicationMarkerV1`とafter stateを一つのpublic CAS transactionでpublishする。Public MarkerのOperation ref、intent hash、request hash、idempotency key、before state、public-after state、Context、private marker hashはPublished payloadおよびprivate Markerの対応Fieldとbyte equalityにし、`signed_domain_receipt_ref/hash`はreadbackしたexact wrapperだけを指す。`public_after_state_ref/hash`はPublished payloadの`after_state_ref/hash`およびprivate Markerの`staged_after_state_ref/hash`と同一である。このafter stateはsigned Receipt、private／public Marker、publication projectionをFieldにもcontent-hash preimageにも含まないReceipt-free staged stateでなければならず、それらを含むDomain projectionをPublic Markerのafter stateに代用しない。Project／Registry／Runtime current headはPublic Markerが指すこのReceipt-free after stateだけをcurrentとして解決する。Domain Resultまたはroot外publication projectionだけがPublic Markerとsigned wrapperを結合し、unsigned prepared payload、private Marker、receipt-store単独存在をstate authorityにしない。これにより`candidate → Receipt-free staged after state／staged postcondition → private marker → signed wrapper → public marker＋same after state → root外projection`の一方向DAGとなる。postcondition failureまたはprivate commit失敗はpublic state／Receipt／Public Markerを0件にし、prepared payloadを権限証拠として外部公開しない。

private Markerのdurable commit後、signed wrapper保存前に停止した場合、recoveryはprivate Marker、Prepared Envelope、Prepared payload、materialization Context、staged postcondition、staged after stateをread-backし、上記identity／state／Context equalityを再検証してから、同じmaterialization key、Contextに固定されたissued-at／revocation／Key、deterministic profileから同じwrapper bytesを作る。clock、current Key選択、発行時revocation snapshotを再取得しない。wrapper保存後かつPublic Marker前に停止した場合、wrapper byte／signature／current revocationとprivate preimageを再検証し、同じexpected public predecessorへのCASをroll-forwardする。Public Marker後はrollbackせず、同じidempotency key＋request hashのretryへ同じResult／wrapper／Public Markerを返す。Markerと`PublishedReceiptMaterializationKeyPayloadV1`のpayload countは各配列長と一致させ、両membersはpayload type refのID／version／Contract set root、payload content refのcanonical bytes、content hash順へstrict sortする。両者のOperation ref、intent hash、request hash、idempotency key、before／staged-after state、ContextもPrepared Envelope／Domain publication bindingとbyte equalityにする。duplicate、same-ref different-hash、両集合のmissing／extra、identity／state／Context substitutionを拒否する。`key_payload_bytes`はこのclosed payloadをMCD canonical encodeしたbytesで、`receipt_materialization_key = SHA-256(ASCII "MIRAKAN_PUBLISHED_RECEIPT_MATERIALIZATION_V1" || uint32_be(len(key_payload_bytes)) || key_payload_bytes)`とし、key自身をpayloadへ含めない。別wrapper、alternate signature、二重Public Marker、既存bytes overwrite、署名なしpublish、public後rollbackをintegrity faultとして隔離する。private MarkerなしのPrepared artifactは非公開のまま破棄する。

Snapshot preimageでは`OperationValidatorClosureLocalRecordV2`を`member_kind=operation_validator_closure`としてroot化し、Operation、Validator、Diagnosticへの全edgeをLocalRefにする。`closure_local_record_hash`はASCII `MIRAKAN_OPERATION_VALIDATOR_CLOSURE_LOCAL_RECORD_V2`と同Fieldだけを除くlocal recordから計算する。Contract Set `member_hash`は別domain `MIRAKAN_CONTRACT_SET_OPERATION_VALIDATOR_CLOSURE_MEMBER_V2`と、このlocal hashを含む完成local recordから計算する別値であり、両hashのbyte equalityを要求しない。set root確定後にだけLocalRefを外部Operation ref、`ValidatorRecordRefV1`、`DiagnosticCodeRefV1`へ投影し、`OperationValidatorClosureV1`を作る。`closure_content_hash`はASCII `MIRAKAN_OPERATION_VALIDATOR_CLOSURE_V1`、外部closureの`closure_content_hash`自身を除くcanonical bytesから計算し、完成後にだけ`OperationValidatorClosureRefV1`をmaterializeする。この外部hashとRefはSnapshot preimageへ戻さない。外部closureのvalidator refはValidator Registryへexact解決し、各Validator recordが宣言する`error_refs[]`のunionをID／code／version／hash順へcanonicalizeした集合が`reachable_error_refs[]`と一致しなければならない。さらにその集合とOperation `errors[]`をset equalityで比較し、missing errorと到達不能extra errorの双方でOperation Registry全体を拒否する。`reachable_error_set_hash`はLocal形ではASCII `MIRAKAN_OPERATION_REACHABLE_ERROR_LOCAL_SET_V2`とDiagnostic LocalRef集合、外部形ではASCII `MIRAKAN_OPERATION_REACHABLE_ERROR_SET_V1`と各四Fieldrefから別々に計算し、どちらもerror countとcanonical bytesを`uint32_be` length framingして自己Fieldを入力にしない。compilerはLocalと外部のlogical Diagnostic集合が一対一であることも検査する。

Snapshot内部の正本は`ValidatorLocalRecordV2 {validator_local_ref, implementation_artifact_ref/hash, input_type_local_ref, error_local_refs[1..64], validator_local_content_hash}`である。Type／DiagnosticはLocalRefを使い、外部MCD refやset rootをlocal record hashへ含めない。`validator_local_content_hash = SHA-256(ASCII "MIRAKAN_VALIDATOR_LOCAL_RECORD_V2" || uint32_be(len(self-excluding canonical local bytes)) || local bytes)`、Validator Registry hashはASCII `MIRAKAN_VALIDATOR_REGISTRY_V2`、Registry ID／version、record count、validator ID／version順local record bytesから計算する。Contract SetのValidator `member_hash`は別domain `MIRAKAN_CONTRACT_SET_VALIDATOR_MEMBER_V2`と、このlocal hashを含む完成local recordから計算する第三の値であり、local hashとのbyte equalityを要求しない。duplicate、same-ID different-hash、非canonical sort、実装Artifact missing／hash mismatch、input type kind／version mismatch、Diagnostic LocalRef未解決を拒否する。

set root確定後だけ、compilerはLocalRefを同じrootのexact `McdContractRefV1(kind=type)`／`DiagnosticCodeRefV1`へ一対一投影し、`materialized_from_contract_set_hash=contract_set_hash`を持つ完全な`ValidatorRecordV1`を作る。`validator_content_hash = SHA-256(ASCII "MIRAKAN_VALIDATOR_RECORD_V1" || uint32_be(len(external record canonical bytes excluding validator_content_hash)) || external bytes)`であり、完成後だけ`ValidatorRecordRefV1 {validator_id,validator_version,validator_content_hash}`をmaterializeする。外部recordのinput Type refと全Diagnostic refのContract set rootは`materialized_from_contract_set_hash`と一致し、logical ID／version集合はlocal recordとexact equalityでなければならない。外部record／hash／refはlocal record、Validator Registry、member hash、set rootへ戻さない。別rootのValidator ref代用、local/external集合差、stale root、external hash mismatchをContract set compile errorにする。

八Operationが参照する初期Diagnostic Registry subsetを次へ閉じる。全rowは`diagnostic_version=1`、`requirement_local_refs=[]`、`message_key="<diagnostic_id>.message"`を持つ完全な`DiagnosticLocalRecordV2`であり、root確定後の外部projectionは`requirement_refs=[]`を持つ。exact Owner local inventoryは既存ownerだけを使う`{owner.core.project_state,1,owner_content_hash}`、`{owner.core.security_approval,1,owner_content_hash}`、`{owner.core.gameplay_programming_model,1,owner_content_hash}`の三件で、各hashは`MIRAKAN_DIAGNOSTIC_OWNER_LOCAL_IDENTITY_V1`規則から計算する。`diagnostic.authorization.denied`と`diagnostic.approval.required`は`owner.core.security_approval`、他の共通六件と`diagnostic.project.runtime_entry.*`は`owner.core.project_state`、`diagnostic.runtime_scope.*`は`owner.core.gameplay_programming_model`のexact `DiagnosticOwnerLocalRefV1`を`owner_local_ref`へ持ち、外部recordも同じ三Field Owner refへ投影する。各rowはOwnerを含む表の全Fieldからself-excluding `diagnostic_local_content_hash`を計算し、root後は別のself-excluding `diagnostic_content_hash`を計算する。表外code、同じcodeの別ID、説明語から合成したcode、owner prefixから補完したrefをOperation errorとして返さない。

| `diagnostic_id` | `code` | severity／category／retryability |
|---|---|---|
| `diagnostic.conflict.revision_mismatch` | `MIRAKAN-CONFLICT-REVISION_MISMATCH` | blocking／conflict／after_change |
| `diagnostic.authorization.denied` | `MIRAKAN-AUTHORIZATION-DENIED` | blocking／permission／never |
| `diagnostic.approval.required` | `MIRAKAN-APPROVAL-REQUIRED` | blocking／permission／after_input |
| `diagnostic.authoring.lock_conflict` | `MIRAKAN-AUTHORING-LOCK_CONFLICT` | blocking／conflict／after_change |
| `diagnostic.mcd.operation_predicate_invalid` | `MIRAKAN-MCD-OPERATION-PREDICATE_INVALID` | blocking／schema／after_change |
| `diagnostic.operation.timeout` | `MIRAKAN-OPERATION-TIMEOUT` | error／infrastructure／transient |
| `diagnostic.operation.rate_limit_exceeded` | `MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED` | error／permission／transient |
| `diagnostic.operation.idempotency_key_reuse` | `MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE` | blocking／conflict／after_input |
| `diagnostic.project.runtime_entry.migration_required` | `MIRAKAN-PROJECT-RUNTIME_ENTRY_MIGRATION_REQUIRED` | blocking／semantic／after_change |
| `diagnostic.project.runtime_entry.invalid` | `MIRAKAN-PROJECT-RUNTIME_ENTRY_INVALID` | blocking／semantic／after_input |
| `diagnostic.project.runtime_entry.target_unresolved` | `MIRAKAN-PROJECT-RUNTIME_ENTRY_TARGET_UNRESOLVED` | blocking／semantic／after_change |
| `diagnostic.project.runtime_entry.default_ambiguous` | `MIRAKAN-PROJECT-RUNTIME_ENTRY_DEFAULT_AMBIGUOUS` | blocking／semantic／after_change |
| `diagnostic.project.runtime_entry.branch_field_conflict` | `MIRAKAN-PROJECT-RUNTIME_ENTRY_BRANCH_FIELD_CONFLICT` | blocking／schema／after_input |
| `diagnostic.project.runtime_entry.dangling_reference` | `MIRAKAN-PROJECT-RUNTIME_ENTRY_DANGLING_REFERENCE` | blocking／semantic／after_change |
| `diagnostic.project.runtime_entry.document_hash_mismatch` | `MIRAKAN-PROJECT-RUNTIME_ENTRY_DOCUMENT_HASH_MISMATCH` | blocking／conflict／after_change |
| `diagnostic.project.runtime_entry.semantic_hash_mismatch` | `MIRAKAN-PROJECT-RUNTIME_ENTRY_SEMANTIC_HASH_MISMATCH` | blocking／semantic／after_input |
| `diagnostic.project.runtime_entry.schema_mismatch` | `MIRAKAN-PROJECT-RUNTIME_ENTRY_SCHEMA_MISMATCH` | blocking／schema／after_input |
| `diagnostic.project.runtime_entry.explicit_target_mismatch` | `MIRAKAN-PROJECT-RUNTIME_ENTRY_EXPLICIT_TARGET_MISMATCH` | blocking／semantic／after_input |
| `diagnostic.project.runtime_entry.identity_mismatch` | `MIRAKAN-PROJECT-RUNTIME_ENTRY_IDENTITY_MISMATCH` | blocking／semantic／after_input |
| `diagnostic.runtime_scope.catalog_invalid` | `MIRAKAN-RUNTIME-SCOPE-CATALOG_INVALID` | blocking／schema／after_change |
| `diagnostic.runtime_scope.owner_unavailable` | `MIRAKAN-RUNTIME-SCOPE-OWNER_UNAVAILABLE` | blocking／semantic／after_change |
| `diagnostic.runtime_scope.version_hash_mismatch` | `MIRAKAN-RUNTIME-SCOPE-VERSION_HASH_MISMATCH` | blocking／conflict／after_change |
| `diagnostic.runtime_scope.migration_conflict` | `MIRAKAN-RUNTIME-SCOPE-MIGRATION_CONFLICT` | blocking／conflict／after_input |
| `diagnostic.runtime_scope.contribution_invalid` | `MIRAKAN-RUNTIME-SCOPE-CONTRIBUTION_INVALID` | blocking／schema／after_change |
| `diagnostic.runtime_scope.receipt_binding_mismatch` | `MIRAKAN-RUNTIME-SCOPE-RECEIPT_BINDING_MISMATCH` | blocking／semantic／after_change |

下表はset root確定後のexternal materialized projectionである。表中の`{id,1,contract_set_hash}`は三Fieldを持つexact `McdContractRefV1`、Service refは`{service_id,service_version,service_content_hash}`、Diagnostic refは`{diagnostic_id,code,diagnostic_version,diagnostic_content_hash}`である。各行の外部MCD共通Envelopeは省略せず表内に全値を持つが、この外部形を`ContractSetMemberLocalRecordV2.canonical_payload`へ直接hashしない。

Snapshot compiler inputへのField projectionを次へ固定する。

| external field | snapshot-local exact field |
|---|---|
| `input_type; output_type; receipt_type; preconditions[]; postconditions[]; rate_limit_policy; requirement_refs[]` | 各`{kind,id,version}`の`ContractSetLocalRefV1`。`contract_set_hash`を除去 |
| `authority: TrustedServiceRefV1` | `{service_id,service_version}`の`TrustedServiceLocalRefV1`。`service_content_hash`を除去 |
| `errors[]: DiagnosticCodeRefV1` | `{diagnostic_id,diagnostic_version}`の`DiagnosticLocalRefV1`。DiagnosticはMCD kindではなく専用Registry recordで、code／content hashは同Registryのlocal record側で検証 |
| `validator_closure_ref` | `{closure_id,closure_version}`の`OperationValidatorClosureLocalRefV1`。closure content hashを除去 |
| closure内`operation_ref`／Service内`allowed_operation_refs[]` | Operationの`ContractSetLocalRefV1(kind=operation)` |
| closure内`validator_refs[]`／`reachable_error_refs[]` | `ValidatorLocalRefV1`／`DiagnosticLocalRefV1`。Validator／Diagnostic content hashを除去 |
| Validator内input／error ref | `ContractSetLocalRefV1(kind=type)`／`DiagnosticLocalRefV1` |

投影はFieldごとのtotal functionで、external ref一件をLocalRef一件へ変換し、配列cardinality／順序／duplicate statusを変えない。表示placeholder、名前prefix、current/latest lookup、hash欠落の補完を禁止する。compiler fixtureは八Operationについてexternal ref集合からhash Fieldだけを除いたlogical identity集合とlocal payload集合、root確定後に再materializeしたexternal集合をField別にset equalityにし、external hashをLocal preimageへ残す、kind変更、Service／Validatorのhash付き相互edge、配列一件欠落／extra／reorderを各一原因で拒否する。

| Operation MCD共通Envelope exact value | Operation固有Field exact value |
|---|---|
| `mcd_version=1; kind=operation; id=operation.project.runtime_entry.create; version=1; status=active; title=Create Runtime Entry; description=Create one identity-consistent Runtime Entry Document; owners=[owner.core.project_state]; requirement_refs=[]; rationale_refs=[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]; since_contract_set=1; supersedes=[]; tags=[authoring,runtime_entry]` | `operation_kind=command; input_type={type.project.runtime_entry.create_input,1,contract_set_hash}; output_type={type.project.runtime_entry.mutation_result,1,contract_set_hash}; authority={service_id=service.authoring_command_gateway,service_version=1,service_content_hash}; risk_class=R2; side_effects=[authoring]; idempotency=idempotent_with_key; transaction=authoring_changeset; preconditions=[{policy.operation.project.runtime_entry.create.precondition,1,contract_set_hash}]; postconditions=[{policy.operation.project.runtime_entry.create.postcondition,1,contract_set_hash}]; errors=[{diagnostic.conflict.revision_mismatch,MIRAKAN-CONFLICT-REVISION_MISMATCH,1,diagnostic_content_hash},{diagnostic.authorization.denied,MIRAKAN-AUTHORIZATION-DENIED,1,diagnostic_content_hash},{diagnostic.approval.required,MIRAKAN-APPROVAL-REQUIRED,1,diagnostic_content_hash},{diagnostic.authoring.lock_conflict,MIRAKAN-AUTHORING-LOCK_CONFLICT,1,diagnostic_content_hash},{diagnostic.mcd.operation_predicate_invalid,MIRAKAN-MCD-OPERATION-PREDICATE_INVALID,1,diagnostic_content_hash},{diagnostic.operation.timeout,MIRAKAN-OPERATION-TIMEOUT,1,diagnostic_content_hash},{diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash},{diagnostic.operation.idempotency_key_reuse,MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.invalid,MIRAKAN-PROJECT-RUNTIME_ENTRY_INVALID,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.target_unresolved,MIRAKAN-PROJECT-RUNTIME_ENTRY_TARGET_UNRESOLVED,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.default_ambiguous,MIRAKAN-PROJECT-RUNTIME_ENTRY_DEFAULT_AMBIGUOUS,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.branch_field_conflict,MIRAKAN-PROJECT-RUNTIME_ENTRY_BRANCH_FIELD_CONFLICT,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.dangling_reference,MIRAKAN-PROJECT-RUNTIME_ENTRY_DANGLING_REFERENCE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.semantic_hash_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SEMANTIC_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.schema_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SCHEMA_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.identity_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_IDENTITY_MISMATCH,1,diagnostic_content_hash}]; validator_closure_ref={validator_closure.operation.project.runtime_entry.create,1,closure_content_hash}; timeout_ms=30000; rate_limit_policy={policy.authoring.runtime_entry.rate_limit,1,contract_set_hash}; audit_level=full_redacted; provider_exposure=mcp_proposal; receipt_type={type.project.runtime_entry.mutation_receipt,1,contract_set_hash}` |
| `mcd_version=1; kind=operation; id=operation.project.runtime_entry.update; version=1; status=active; title=Update Runtime Entry; description=Update one Runtime Entry Document while preserving its identity; owners=[owner.core.project_state]; requirement_refs=[]; rationale_refs=[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]; since_contract_set=1; supersedes=[]; tags=[authoring,runtime_entry]` | `operation_kind=command; input_type={type.project.runtime_entry.update_input,1,contract_set_hash}; output_type={type.project.runtime_entry.mutation_result,1,contract_set_hash}; authority={service_id=service.authoring_command_gateway,service_version=1,service_content_hash}; risk_class=R2; side_effects=[authoring]; idempotency=idempotent_with_key; transaction=authoring_changeset; preconditions=[{policy.operation.project.runtime_entry.update.precondition,1,contract_set_hash}]; postconditions=[{policy.operation.project.runtime_entry.update.postcondition,1,contract_set_hash}]; errors=[{diagnostic.conflict.revision_mismatch,MIRAKAN-CONFLICT-REVISION_MISMATCH,1,diagnostic_content_hash},{diagnostic.authorization.denied,MIRAKAN-AUTHORIZATION-DENIED,1,diagnostic_content_hash},{diagnostic.approval.required,MIRAKAN-APPROVAL-REQUIRED,1,diagnostic_content_hash},{diagnostic.authoring.lock_conflict,MIRAKAN-AUTHORING-LOCK_CONFLICT,1,diagnostic_content_hash},{diagnostic.mcd.operation_predicate_invalid,MIRAKAN-MCD-OPERATION-PREDICATE_INVALID,1,diagnostic_content_hash},{diagnostic.operation.timeout,MIRAKAN-OPERATION-TIMEOUT,1,diagnostic_content_hash},{diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash},{diagnostic.operation.idempotency_key_reuse,MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.invalid,MIRAKAN-PROJECT-RUNTIME_ENTRY_INVALID,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.target_unresolved,MIRAKAN-PROJECT-RUNTIME_ENTRY_TARGET_UNRESOLVED,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.default_ambiguous,MIRAKAN-PROJECT-RUNTIME_ENTRY_DEFAULT_AMBIGUOUS,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.branch_field_conflict,MIRAKAN-PROJECT-RUNTIME_ENTRY_BRANCH_FIELD_CONFLICT,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.dangling_reference,MIRAKAN-PROJECT-RUNTIME_ENTRY_DANGLING_REFERENCE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.document_hash_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_DOCUMENT_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.semantic_hash_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SEMANTIC_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.schema_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SCHEMA_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.identity_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_IDENTITY_MISMATCH,1,diagnostic_content_hash}]; validator_closure_ref={validator_closure.operation.project.runtime_entry.update,1,closure_content_hash}; timeout_ms=30000; rate_limit_policy={policy.authoring.runtime_entry.rate_limit,1,contract_set_hash}; audit_level=full_redacted; provider_exposure=mcp_proposal; receipt_type={type.project.runtime_entry.mutation_receipt,1,contract_set_hash}` |
| `mcd_version=1; kind=operation; id=operation.project.runtime_target_selector.create; version=1; status=active; title=Create Runtime Target Selector; description=Create one identity-consistent Target Selector Document without attaching it to an entry; owners=[owner.core.project_state]; requirement_refs=[]; rationale_refs=[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]; since_contract_set=1; supersedes=[]; tags=[authoring,runtime_entry,target_selector]` | `operation_kind=command; input_type={type.project.runtime_target_selector.create_input,1,contract_set_hash}; output_type={type.project.runtime_entry.mutation_result,1,contract_set_hash}; authority={service_id=service.authoring_command_gateway,service_version=1,service_content_hash}; risk_class=R2; side_effects=[authoring]; idempotency=idempotent_with_key; transaction=authoring_changeset; preconditions=[{policy.operation.project.runtime_target_selector.create.precondition,1,contract_set_hash}]; postconditions=[{policy.operation.project.runtime_target_selector.create.postcondition,1,contract_set_hash}]; errors=[{diagnostic.conflict.revision_mismatch,MIRAKAN-CONFLICT-REVISION_MISMATCH,1,diagnostic_content_hash},{diagnostic.authorization.denied,MIRAKAN-AUTHORIZATION-DENIED,1,diagnostic_content_hash},{diagnostic.approval.required,MIRAKAN-APPROVAL-REQUIRED,1,diagnostic_content_hash},{diagnostic.authoring.lock_conflict,MIRAKAN-AUTHORING-LOCK_CONFLICT,1,diagnostic_content_hash},{diagnostic.mcd.operation_predicate_invalid,MIRAKAN-MCD-OPERATION-PREDICATE_INVALID,1,diagnostic_content_hash},{diagnostic.operation.timeout,MIRAKAN-OPERATION-TIMEOUT,1,diagnostic_content_hash},{diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash},{diagnostic.operation.idempotency_key_reuse,MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.invalid,MIRAKAN-PROJECT-RUNTIME_ENTRY_INVALID,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.target_unresolved,MIRAKAN-PROJECT-RUNTIME_ENTRY_TARGET_UNRESOLVED,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.dangling_reference,MIRAKAN-PROJECT-RUNTIME_ENTRY_DANGLING_REFERENCE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.semantic_hash_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SEMANTIC_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.schema_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SCHEMA_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.identity_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_IDENTITY_MISMATCH,1,diagnostic_content_hash}]; validator_closure_ref={validator_closure.operation.project.runtime_target_selector.create,1,closure_content_hash}; timeout_ms=30000; rate_limit_policy={policy.authoring.runtime_entry.rate_limit,1,contract_set_hash}; audit_level=full_redacted; provider_exposure=mcp_proposal; receipt_type={type.project.runtime_entry.mutation_receipt,1,contract_set_hash}` |
| `mcd_version=1; kind=operation; id=operation.project.runtime_target_selector.update; version=1; status=active; title=Update Runtime Target Selector; description=Update one Target Selector Document and revalidate default coverage; owners=[owner.core.project_state]; requirement_refs=[]; rationale_refs=[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]; since_contract_set=1; supersedes=[]; tags=[authoring,runtime_entry,target_selector]` | `operation_kind=command; input_type={type.project.runtime_target_selector.update_input,1,contract_set_hash}; output_type={type.project.runtime_entry.mutation_result,1,contract_set_hash}; authority={service_id=service.authoring_command_gateway,service_version=1,service_content_hash}; risk_class=R2; side_effects=[authoring]; idempotency=idempotent_with_key; transaction=authoring_changeset; preconditions=[{policy.operation.project.runtime_target_selector.update.precondition,1,contract_set_hash}]; postconditions=[{policy.operation.project.runtime_target_selector.update.postcondition,1,contract_set_hash}]; errors=[{diagnostic.conflict.revision_mismatch,MIRAKAN-CONFLICT-REVISION_MISMATCH,1,diagnostic_content_hash},{diagnostic.authorization.denied,MIRAKAN-AUTHORIZATION-DENIED,1,diagnostic_content_hash},{diagnostic.approval.required,MIRAKAN-APPROVAL-REQUIRED,1,diagnostic_content_hash},{diagnostic.authoring.lock_conflict,MIRAKAN-AUTHORING-LOCK_CONFLICT,1,diagnostic_content_hash},{diagnostic.mcd.operation_predicate_invalid,MIRAKAN-MCD-OPERATION-PREDICATE_INVALID,1,diagnostic_content_hash},{diagnostic.operation.timeout,MIRAKAN-OPERATION-TIMEOUT,1,diagnostic_content_hash},{diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash},{diagnostic.operation.idempotency_key_reuse,MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.invalid,MIRAKAN-PROJECT-RUNTIME_ENTRY_INVALID,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.target_unresolved,MIRAKAN-PROJECT-RUNTIME_ENTRY_TARGET_UNRESOLVED,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.default_ambiguous,MIRAKAN-PROJECT-RUNTIME_ENTRY_DEFAULT_AMBIGUOUS,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.dangling_reference,MIRAKAN-PROJECT-RUNTIME_ENTRY_DANGLING_REFERENCE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.document_hash_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_DOCUMENT_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.semantic_hash_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SEMANTIC_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.schema_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SCHEMA_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.identity_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_IDENTITY_MISMATCH,1,diagnostic_content_hash}]; validator_closure_ref={validator_closure.operation.project.runtime_target_selector.update,1,closure_content_hash}; timeout_ms=30000; rate_limit_policy={policy.authoring.runtime_entry.rate_limit,1,contract_set_hash}; audit_level=full_redacted; provider_exposure=mcp_proposal; receipt_type={type.project.runtime_entry.mutation_receipt,1,contract_set_hash}` |
| `mcd_version=1; kind=operation; id=operation.project.runtime_entry_activation_policy.create; version=1; status=active; title=Create Runtime Entry Activation Policy; description=Create one identity-consistent Runtime Entry Activation Policy Document; owners=[owner.core.project_state]; requirement_refs=[]; rationale_refs=[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]; since_contract_set=1; supersedes=[]; tags=[activation_policy,authoring,runtime_entry]` | `operation_kind=command; input_type={type.project.runtime_entry_activation_policy.create_input,1,contract_set_hash}; output_type={type.project.runtime_entry.mutation_result,1,contract_set_hash}; authority={service_id=service.authoring_command_gateway,service_version=1,service_content_hash}; risk_class=R2; side_effects=[authoring]; idempotency=idempotent_with_key; transaction=authoring_changeset; preconditions=[{policy.operation.project.runtime_entry_activation_policy.create.precondition,1,contract_set_hash}]; postconditions=[{policy.operation.project.runtime_entry_activation_policy.create.postcondition,1,contract_set_hash}]; errors=[{diagnostic.conflict.revision_mismatch,MIRAKAN-CONFLICT-REVISION_MISMATCH,1,diagnostic_content_hash},{diagnostic.authorization.denied,MIRAKAN-AUTHORIZATION-DENIED,1,diagnostic_content_hash},{diagnostic.approval.required,MIRAKAN-APPROVAL-REQUIRED,1,diagnostic_content_hash},{diagnostic.authoring.lock_conflict,MIRAKAN-AUTHORING-LOCK_CONFLICT,1,diagnostic_content_hash},{diagnostic.mcd.operation_predicate_invalid,MIRAKAN-MCD-OPERATION-PREDICATE_INVALID,1,diagnostic_content_hash},{diagnostic.operation.timeout,MIRAKAN-OPERATION-TIMEOUT,1,diagnostic_content_hash},{diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash},{diagnostic.operation.idempotency_key_reuse,MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.invalid,MIRAKAN-PROJECT-RUNTIME_ENTRY_INVALID,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.dangling_reference,MIRAKAN-PROJECT-RUNTIME_ENTRY_DANGLING_REFERENCE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.semantic_hash_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SEMANTIC_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.schema_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SCHEMA_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.identity_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_IDENTITY_MISMATCH,1,diagnostic_content_hash}]; validator_closure_ref={validator_closure.operation.project.runtime_entry_activation_policy.create,1,closure_content_hash}; timeout_ms=30000; rate_limit_policy={policy.authoring.runtime_entry.rate_limit,1,contract_set_hash}; audit_level=full_redacted; provider_exposure=mcp_proposal; receipt_type={type.project.runtime_entry.mutation_receipt,1,contract_set_hash}` |
| `mcd_version=1; kind=operation; id=operation.project.runtime_entry_activation_policy.update; version=1; status=active; title=Update Runtime Entry Activation Policy; description=Update one Runtime Entry Activation Policy Document and invalidate consumers; owners=[owner.core.project_state]; requirement_refs=[]; rationale_refs=[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]; since_contract_set=1; supersedes=[]; tags=[activation_policy,authoring,runtime_entry]` | `operation_kind=command; input_type={type.project.runtime_entry_activation_policy.update_input,1,contract_set_hash}; output_type={type.project.runtime_entry.mutation_result,1,contract_set_hash}; authority={service_id=service.authoring_command_gateway,service_version=1,service_content_hash}; risk_class=R2; side_effects=[authoring]; idempotency=idempotent_with_key; transaction=authoring_changeset; preconditions=[{policy.operation.project.runtime_entry_activation_policy.update.precondition,1,contract_set_hash}]; postconditions=[{policy.operation.project.runtime_entry_activation_policy.update.postcondition,1,contract_set_hash}]; errors=[{diagnostic.conflict.revision_mismatch,MIRAKAN-CONFLICT-REVISION_MISMATCH,1,diagnostic_content_hash},{diagnostic.authorization.denied,MIRAKAN-AUTHORIZATION-DENIED,1,diagnostic_content_hash},{diagnostic.approval.required,MIRAKAN-APPROVAL-REQUIRED,1,diagnostic_content_hash},{diagnostic.authoring.lock_conflict,MIRAKAN-AUTHORING-LOCK_CONFLICT,1,diagnostic_content_hash},{diagnostic.mcd.operation_predicate_invalid,MIRAKAN-MCD-OPERATION-PREDICATE_INVALID,1,diagnostic_content_hash},{diagnostic.operation.timeout,MIRAKAN-OPERATION-TIMEOUT,1,diagnostic_content_hash},{diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash},{diagnostic.operation.idempotency_key_reuse,MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.invalid,MIRAKAN-PROJECT-RUNTIME_ENTRY_INVALID,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.dangling_reference,MIRAKAN-PROJECT-RUNTIME_ENTRY_DANGLING_REFERENCE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.document_hash_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_DOCUMENT_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.semantic_hash_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SEMANTIC_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.schema_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SCHEMA_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.identity_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_IDENTITY_MISMATCH,1,diagnostic_content_hash}]; validator_closure_ref={validator_closure.operation.project.runtime_entry_activation_policy.update,1,closure_content_hash}; timeout_ms=30000; rate_limit_policy={policy.authoring.runtime_entry.rate_limit,1,contract_set_hash}; audit_level=full_redacted; provider_exposure=mcp_proposal; receipt_type={type.project.runtime_entry.mutation_receipt,1,contract_set_hash}` |
| `mcd_version=1; kind=operation; id=operation.project.runtime_entry.migrate_root_scene; version=1; status=active; title=Migrate Root Scene to Runtime Entry; description=Atomically create or update World Selector Policy and Runtime Entry Documents from one legacy root Scene closure; owners=[owner.core.project_state]; requirement_refs=[]; rationale_refs=[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]; since_contract_set=1; supersedes=[]; tags=[authoring,migration,runtime_entry]` | `operation_kind=command; input_type={type.project.runtime_entry.migrate_root_scene_input,1,contract_set_hash}; output_type={type.project.runtime_entry.mutation_result,1,contract_set_hash}; authority={service_id=service.authoring_command_gateway,service_version=1,service_content_hash}; risk_class=R3; side_effects=[authoring]; idempotency=idempotent_with_key; transaction=authoring_changeset; preconditions=[{policy.operation.project.runtime_entry.migrate_root_scene.precondition,1,contract_set_hash}]; postconditions=[{policy.operation.project.runtime_entry.migrate_root_scene.postcondition,1,contract_set_hash}]; errors=[{diagnostic.conflict.revision_mismatch,MIRAKAN-CONFLICT-REVISION_MISMATCH,1,diagnostic_content_hash},{diagnostic.authorization.denied,MIRAKAN-AUTHORIZATION-DENIED,1,diagnostic_content_hash},{diagnostic.approval.required,MIRAKAN-APPROVAL-REQUIRED,1,diagnostic_content_hash},{diagnostic.authoring.lock_conflict,MIRAKAN-AUTHORING-LOCK_CONFLICT,1,diagnostic_content_hash},{diagnostic.mcd.operation_predicate_invalid,MIRAKAN-MCD-OPERATION-PREDICATE_INVALID,1,diagnostic_content_hash},{diagnostic.operation.timeout,MIRAKAN-OPERATION-TIMEOUT,1,diagnostic_content_hash},{diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash},{diagnostic.operation.idempotency_key_reuse,MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.migration_required,MIRAKAN-PROJECT-RUNTIME_ENTRY_MIGRATION_REQUIRED,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.invalid,MIRAKAN-PROJECT-RUNTIME_ENTRY_INVALID,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.target_unresolved,MIRAKAN-PROJECT-RUNTIME_ENTRY_TARGET_UNRESOLVED,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.default_ambiguous,MIRAKAN-PROJECT-RUNTIME_ENTRY_DEFAULT_AMBIGUOUS,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.branch_field_conflict,MIRAKAN-PROJECT-RUNTIME_ENTRY_BRANCH_FIELD_CONFLICT,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.dangling_reference,MIRAKAN-PROJECT-RUNTIME_ENTRY_DANGLING_REFERENCE,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.document_hash_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_DOCUMENT_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.semantic_hash_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SEMANTIC_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.schema_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_SCHEMA_MISMATCH,1,diagnostic_content_hash},{diagnostic.project.runtime_entry.identity_mismatch,MIRAKAN-PROJECT-RUNTIME_ENTRY_IDENTITY_MISMATCH,1,diagnostic_content_hash}]; validator_closure_ref={validator_closure.operation.project.runtime_entry.migrate_root_scene,1,closure_content_hash}; timeout_ms=120000; rate_limit_policy={policy.authoring.runtime_entry.rate_limit,1,contract_set_hash}; audit_level=full_redacted; provider_exposure=mcp_proposal; receipt_type={type.project.runtime_entry.mutation_receipt,1,contract_set_hash}` |
| `mcd_version=1; kind=operation; id=operation.runtime_scope.migrate_game_system; version=1; status=active; title=Migrate Game System Runtime Scope; description=Atomically migrate one legacy Game System schema revision through an owner contribution; owners=[owner.core.gameplay_programming_model]; requirement_refs=[]; rationale_refs=[mirakan.arch.gameplay-programming-model#312-scope依存recordとoffline-migration]; since_contract_set=1; supersedes=[]; tags=[authoring,migration,runtime_scope]` | `operation_kind=job; input_type={type.runtime_scope.game_system_migration_input,1,contract_set_hash}; output_type={type.runtime_scope.game_system_migration_result,1,contract_set_hash}; authority={service_id=service.offline_project_migrator,service_version=1,service_content_hash}; risk_class=R3; side_effects=[authoring]; idempotency=idempotent_with_key; transaction=authoring_changeset; preconditions=[{policy.operation.runtime_scope.migrate_game_system.precondition,1,contract_set_hash}]; postconditions=[{policy.operation.runtime_scope.migrate_game_system.postcondition,1,contract_set_hash}]; errors=[{diagnostic.conflict.revision_mismatch,MIRAKAN-CONFLICT-REVISION_MISMATCH,1,diagnostic_content_hash},{diagnostic.authorization.denied,MIRAKAN-AUTHORIZATION-DENIED,1,diagnostic_content_hash},{diagnostic.approval.required,MIRAKAN-APPROVAL-REQUIRED,1,diagnostic_content_hash},{diagnostic.authoring.lock_conflict,MIRAKAN-AUTHORING-LOCK_CONFLICT,1,diagnostic_content_hash},{diagnostic.mcd.operation_predicate_invalid,MIRAKAN-MCD-OPERATION-PREDICATE_INVALID,1,diagnostic_content_hash},{diagnostic.operation.timeout,MIRAKAN-OPERATION-TIMEOUT,1,diagnostic_content_hash},{diagnostic.operation.rate_limit_exceeded,MIRAKAN-OPERATION-RATE_LIMIT_EXCEEDED,1,diagnostic_content_hash},{diagnostic.operation.idempotency_key_reuse,MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE,1,diagnostic_content_hash},{diagnostic.runtime_scope.catalog_invalid,MIRAKAN-RUNTIME-SCOPE-CATALOG_INVALID,1,diagnostic_content_hash},{diagnostic.runtime_scope.owner_unavailable,MIRAKAN-RUNTIME-SCOPE-OWNER_UNAVAILABLE,1,diagnostic_content_hash},{diagnostic.runtime_scope.version_hash_mismatch,MIRAKAN-RUNTIME-SCOPE-VERSION_HASH_MISMATCH,1,diagnostic_content_hash},{diagnostic.runtime_scope.migration_conflict,MIRAKAN-RUNTIME-SCOPE-MIGRATION_CONFLICT,1,diagnostic_content_hash},{diagnostic.runtime_scope.contribution_invalid,MIRAKAN-RUNTIME-SCOPE-CONTRIBUTION_INVALID,1,diagnostic_content_hash},{diagnostic.runtime_scope.receipt_binding_mismatch,MIRAKAN-RUNTIME-SCOPE-RECEIPT_BINDING_MISMATCH,1,diagnostic_content_hash}]; validator_closure_ref={validator_closure.operation.runtime_scope.migrate_game_system,1,closure_content_hash}; timeout_ms=120000; rate_limit_policy={policy.authoring.runtime_scope_migration.rate_limit,1,contract_set_hash}; audit_level=full_redacted; provider_exposure=mcp_proposal; receipt_type={type.runtime_scope.migration_receipt,1,contract_set_hash}` |

pre／postconditionは新kindではなく、次の16件のactive `policy` MCDである。全recordは`mcd_version=1`、`kind=policy`、`version=1`、`status=active`、`requirement_refs=[]`、`since_contract_set=1`、`supersedes=[]`、`tags=[operation_predicate,pure]`、`evaluation_mode=pure`、`side_effects=[]`、`result_type={id=type.operation.predicate_result,version=1,contract_set_hash}`を持ち、表が各recordの残り全値を明示する。preconditionのinput typeはexact Operation input ref／hashとread-only before snapshot refs／hashesを持つ`type.operation.precondition_evaluation_input` version 1、postconditionはrequest hash、prepared output、before snapshot、未発行Staging snapshot、Prepared Commit Envelopeを持つ`type.operation.postcondition_evaluation_input` version 2である。どちらもcanonical immutable valueだけを入力にし、Project／Registry／clock／networkを評価中にqueryしない。postconditionへCommit Markerまたは公開Receipt refを渡さない。

| policy `id` | title／description | owners／rationale／exact input type |
|---|---|---|
| `policy.operation.project.runtime_entry.create.precondition` | Create Runtime Entry Precondition／expected Project revision、draft hash、allocation、selector、policyを検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.precondition_evaluation_input,1,contract_set_hash}` |
| `policy.operation.project.runtime_entry.create.postcondition` | Create Runtime Entry Postcondition／identity三者一致の未発行Document一件とrevision増分候補を検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.postcondition_evaluation_input,2,contract_set_hash}` |
| `policy.operation.project.runtime_entry.update.precondition` | Update Runtime Entry Precondition／current ref、revision、content hash、semantic hashを検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.precondition_evaluation_input,1,contract_set_hash}` |
| `policy.operation.project.runtime_entry.update.postcondition` | Update Runtime Entry Postcondition／同ID新revision候補とconsumer invalidationを検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.postcondition_evaluation_input,2,contract_set_hash}` |
| `policy.operation.project.runtime_target_selector.create.precondition` | Create Target Selector Precondition／draft、Target集合、Project revisionを検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.precondition_evaluation_input,1,contract_set_hash}` |
| `policy.operation.project.runtime_target_selector.create.postcondition` | Create Target Selector Postcondition／未発行selector identity、hash、Project containmentを検証。entry attach／default coverageは変更しない | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.postcondition_evaluation_input,2,contract_set_hash}` |
| `policy.operation.project.runtime_target_selector.update.precondition` | Update Target Selector Precondition／current selector identity、hash、Target集合を検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.precondition_evaluation_input,1,contract_set_hash}` |
| `policy.operation.project.runtime_target_selector.update.postcondition` | Update Target Selector Postcondition／同ID新revision候補と全consumer coverageを検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.postcondition_evaluation_input,2,contract_set_hash}` |
| `policy.operation.project.runtime_entry_activation_policy.create.precondition` | Create Activation Policy Precondition／closed policy draft、hash、Project revisionを検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.precondition_evaluation_input,1,contract_set_hash}` |
| `policy.operation.project.runtime_entry_activation_policy.create.postcondition` | Create Activation Policy Postcondition／未発行policy identity三者一致と新revision候補を検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.postcondition_evaluation_input,2,contract_set_hash}` |
| `policy.operation.project.runtime_entry_activation_policy.update.precondition` | Update Activation Policy Precondition／current policy ref、revision、content／policy hashを検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.precondition_evaluation_input,1,contract_set_hash}` |
| `policy.operation.project.runtime_entry_activation_policy.update.postcondition` | Update Activation Policy Postcondition／同ID新revision候補とconsumer invalidationを検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.postcondition_evaluation_input,2,contract_set_hash}` |
| `policy.operation.project.runtime_entry.migrate_root_scene.precondition` | Root Scene Migration Precondition／legacy closure、active Target、Approval、Project revisionを検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.precondition_evaluation_input,1,contract_set_hash}` |
| `policy.operation.project.runtime_entry.migrate_root_scene.postcondition` | Root Scene Migration Postcondition／未発行World、selector、policy、entryの四Document候補を検証 | `[owner.core.project_state]`／`[mirakan.arch.project-state#312-runtime-entryのclosed-operation-catalog]`／`{type.operation.postcondition_evaluation_input,2,contract_set_hash}` |
| `policy.operation.runtime_scope.migrate_game_system.precondition` | Runtime Scope Migration Precondition／source／destination schema、owner contribution、Catalog、七dependency、identity mapping、Approvalを検証 | `[owner.core.gameplay_programming_model]`／`[mirakan.arch.gameplay-programming-model#312-scope依存recordとoffline-migration]`／`{type.operation.precondition_evaluation_input,1,contract_set_hash}` |
| `policy.operation.runtime_scope.migrate_game_system.postcondition` | Runtime Scope Migration Postcondition／未発行typed Game System revision、Source／Save／Replay mapping、prepared Receipt bindingを検証 | `[owner.core.gameplay_programming_model]`／`[mirakan.arch.gameplay-programming-model#312-scope依存recordとoffline-migration]`／`{type.operation.postcondition_evaluation_input,2,contract_set_hash}` |

八`OperationValidatorClosureV1`を次へ固定する。表内の各Validator refは`{validator_id,validator_version=1,validator_content_hash}`、closure refは`{closure_id,closure_version=1,closure_content_hash}`である。`reachable_error_refs`は表が指す当該Operationの完全な`errors[]`四Fieldref集合そのもので、Validator Registryの`error_refs[]` unionからmaterializeし、別の共通error categoryやcode prefix展開を行わない。

| closure ref／operation ref | exact `validator_refs[]` | `reachable_error_refs` |
|---|---|---|
| `validator_closure.operation.project.runtime_entry.create`／`operation.project.runtime_entry.create` v1 | `validator.operation.request_envelope; validator.operation.authorization; validator.operation.approval; validator.operation.revision_and_lock; validator.operation.pure_predicate; validator.operation.timeout_and_rate_limit; validator.project.runtime_entry.create_semantics; validator.project.runtime_entry.create_postcondition` | exact `errors[]` set of `operation.project.runtime_entry.create` v1 |
| `validator_closure.operation.project.runtime_entry.update`／`operation.project.runtime_entry.update` v1 | `validator.operation.request_envelope; validator.operation.authorization; validator.operation.approval; validator.operation.revision_and_lock; validator.operation.pure_predicate; validator.operation.timeout_and_rate_limit; validator.project.runtime_entry.update_semantics; validator.project.runtime_entry.update_postcondition` | exact `errors[]` set of `operation.project.runtime_entry.update` v1 |
| `validator_closure.operation.project.runtime_target_selector.create`／`operation.project.runtime_target_selector.create` v1 | `validator.operation.request_envelope; validator.operation.authorization; validator.operation.approval; validator.operation.revision_and_lock; validator.operation.pure_predicate; validator.operation.timeout_and_rate_limit; validator.project.runtime_target_selector.create_semantics; validator.project.runtime_target_selector.create_postcondition` | exact `errors[]` set of `operation.project.runtime_target_selector.create` v1 |
| `validator_closure.operation.project.runtime_target_selector.update`／`operation.project.runtime_target_selector.update` v1 | `validator.operation.request_envelope; validator.operation.authorization; validator.operation.approval; validator.operation.revision_and_lock; validator.operation.pure_predicate; validator.operation.timeout_and_rate_limit; validator.project.runtime_target_selector.update_semantics; validator.project.runtime_target_selector.update_postcondition` | exact `errors[]` set of `operation.project.runtime_target_selector.update` v1 |
| `validator_closure.operation.project.runtime_entry_activation_policy.create`／`operation.project.runtime_entry_activation_policy.create` v1 | `validator.operation.request_envelope; validator.operation.authorization; validator.operation.approval; validator.operation.revision_and_lock; validator.operation.pure_predicate; validator.operation.timeout_and_rate_limit; validator.project.runtime_entry_activation_policy.create_semantics; validator.project.runtime_entry_activation_policy.create_postcondition` | exact `errors[]` set of `operation.project.runtime_entry_activation_policy.create` v1 |
| `validator_closure.operation.project.runtime_entry_activation_policy.update`／`operation.project.runtime_entry_activation_policy.update` v1 | `validator.operation.request_envelope; validator.operation.authorization; validator.operation.approval; validator.operation.revision_and_lock; validator.operation.pure_predicate; validator.operation.timeout_and_rate_limit; validator.project.runtime_entry_activation_policy.update_semantics; validator.project.runtime_entry_activation_policy.update_postcondition` | exact `errors[]` set of `operation.project.runtime_entry_activation_policy.update` v1 |
| `validator_closure.operation.project.runtime_entry.migrate_root_scene`／`operation.project.runtime_entry.migrate_root_scene` v1 | `validator.operation.request_envelope; validator.operation.authorization; validator.operation.approval; validator.operation.revision_and_lock; validator.operation.pure_predicate; validator.operation.timeout_and_rate_limit; validator.project.runtime_entry.root_scene_migration_semantics; validator.project.runtime_entry.root_scene_migration_postcondition` | exact `errors[]` set of `operation.project.runtime_entry.migrate_root_scene` v1 |
| `validator_closure.operation.runtime_scope.migrate_game_system`／`operation.runtime_scope.migrate_game_system` v1 | `validator.operation.request_envelope; validator.operation.authorization; validator.operation.approval; validator.operation.revision_and_lock; validator.operation.pure_predicate; validator.operation.timeout_and_rate_limit; validator.runtime_scope.game_system_migration_semantics; validator.runtime_scope.game_system_migration_postcondition` | exact `errors[]` set of `operation.runtime_scope.migrate_game_system` v1 |

上記closureから到達する全22 Validator local recordを次へmaterializeする。全rowは`validator_version=1`、`implementation_artifact_ref={artifact.validator.<validator_id suffix>,1,artifact_hash}`、表のinput Type LocalRef、表のDiagnostic LocalRef配列、self-excluding `validator_local_content_hash`を持つ。表中のDiagnosticはすべて`kind=remediation`ではなくDiagnostic Registryのtyped LocalRefで、ID／version 1を保存する。空欄、省略、code prefix展開、Operation error集合への遅延参照を許可しない。set root後の外部22 `ValidatorRecordV1`は同じlogical rowをroot付きType／Diagnostic refsへ投影し、各外部`validator_content_hash`を別計算する。

| Validator ID | input Type LocalRef | exact `error_local_refs[]` Diagnostic ID |
|---|---|---|
| `validator.operation.request_envelope` | `type.operation.precondition_evaluation_input@1` | `diagnostic.operation.idempotency_key_reuse` |
| `validator.operation.authorization` | `type.operation.precondition_evaluation_input@1` | `diagnostic.authorization.denied` |
| `validator.operation.approval` | `type.operation.precondition_evaluation_input@1` | `diagnostic.approval.required` |
| `validator.operation.revision_and_lock` | `type.operation.precondition_evaluation_input@1` | `diagnostic.conflict.revision_mismatch; diagnostic.authoring.lock_conflict` |
| `validator.operation.pure_predicate` | `type.operation.precondition_evaluation_input@1` | `diagnostic.mcd.operation_predicate_invalid` |
| `validator.operation.timeout_and_rate_limit` | `type.operation.precondition_evaluation_input@1` | `diagnostic.operation.timeout; diagnostic.operation.rate_limit_exceeded` |
| `validator.project.runtime_entry.create_semantics` | `type.project.runtime_entry.create_input@1` | `diagnostic.project.runtime_entry.invalid; diagnostic.project.runtime_entry.target_unresolved; diagnostic.project.runtime_entry.default_ambiguous; diagnostic.project.runtime_entry.branch_field_conflict; diagnostic.project.runtime_entry.dangling_reference` |
| `validator.project.runtime_entry.create_postcondition` | `type.operation.postcondition_evaluation_input@2` | `diagnostic.project.runtime_entry.semantic_hash_mismatch; diagnostic.project.runtime_entry.schema_mismatch; diagnostic.project.runtime_entry.identity_mismatch` |
| `validator.project.runtime_entry.update_semantics` | `type.project.runtime_entry.update_input@1` | `diagnostic.project.runtime_entry.invalid; diagnostic.project.runtime_entry.target_unresolved; diagnostic.project.runtime_entry.default_ambiguous; diagnostic.project.runtime_entry.branch_field_conflict; diagnostic.project.runtime_entry.dangling_reference` |
| `validator.project.runtime_entry.update_postcondition` | `type.operation.postcondition_evaluation_input@2` | `diagnostic.project.runtime_entry.document_hash_mismatch; diagnostic.project.runtime_entry.semantic_hash_mismatch; diagnostic.project.runtime_entry.schema_mismatch; diagnostic.project.runtime_entry.identity_mismatch` |
| `validator.project.runtime_target_selector.create_semantics` | `type.project.runtime_target_selector.create_input@1` | `diagnostic.project.runtime_entry.invalid; diagnostic.project.runtime_entry.target_unresolved; diagnostic.project.runtime_entry.dangling_reference` |
| `validator.project.runtime_target_selector.create_postcondition` | `type.operation.postcondition_evaluation_input@2` | `diagnostic.project.runtime_entry.semantic_hash_mismatch; diagnostic.project.runtime_entry.schema_mismatch; diagnostic.project.runtime_entry.identity_mismatch` |
| `validator.project.runtime_target_selector.update_semantics` | `type.project.runtime_target_selector.update_input@1` | `diagnostic.project.runtime_entry.invalid; diagnostic.project.runtime_entry.target_unresolved; diagnostic.project.runtime_entry.default_ambiguous; diagnostic.project.runtime_entry.dangling_reference` |
| `validator.project.runtime_target_selector.update_postcondition` | `type.operation.postcondition_evaluation_input@2` | `diagnostic.project.runtime_entry.document_hash_mismatch; diagnostic.project.runtime_entry.semantic_hash_mismatch; diagnostic.project.runtime_entry.schema_mismatch; diagnostic.project.runtime_entry.identity_mismatch` |
| `validator.project.runtime_entry_activation_policy.create_semantics` | `type.project.runtime_entry_activation_policy.create_input@1` | `diagnostic.project.runtime_entry.invalid; diagnostic.project.runtime_entry.dangling_reference` |
| `validator.project.runtime_entry_activation_policy.create_postcondition` | `type.operation.postcondition_evaluation_input@2` | `diagnostic.project.runtime_entry.semantic_hash_mismatch; diagnostic.project.runtime_entry.schema_mismatch; diagnostic.project.runtime_entry.identity_mismatch` |
| `validator.project.runtime_entry_activation_policy.update_semantics` | `type.project.runtime_entry_activation_policy.update_input@1` | `diagnostic.project.runtime_entry.invalid; diagnostic.project.runtime_entry.dangling_reference` |
| `validator.project.runtime_entry_activation_policy.update_postcondition` | `type.operation.postcondition_evaluation_input@2` | `diagnostic.project.runtime_entry.document_hash_mismatch; diagnostic.project.runtime_entry.semantic_hash_mismatch; diagnostic.project.runtime_entry.schema_mismatch; diagnostic.project.runtime_entry.identity_mismatch` |
| `validator.project.runtime_entry.root_scene_migration_semantics` | `type.project.runtime_entry.migrate_root_scene_input@1` | `diagnostic.project.runtime_entry.migration_required; diagnostic.project.runtime_entry.invalid; diagnostic.project.runtime_entry.target_unresolved; diagnostic.project.runtime_entry.default_ambiguous; diagnostic.project.runtime_entry.branch_field_conflict; diagnostic.project.runtime_entry.dangling_reference` |
| `validator.project.runtime_entry.root_scene_migration_postcondition` | `type.operation.postcondition_evaluation_input@2` | `diagnostic.project.runtime_entry.document_hash_mismatch; diagnostic.project.runtime_entry.semantic_hash_mismatch; diagnostic.project.runtime_entry.schema_mismatch; diagnostic.project.runtime_entry.identity_mismatch` |
| `validator.runtime_scope.game_system_migration_semantics` | `type.runtime_scope.game_system_migration_input@1` | `diagnostic.runtime_scope.catalog_invalid; diagnostic.runtime_scope.owner_unavailable; diagnostic.runtime_scope.version_hash_mismatch; diagnostic.runtime_scope.migration_conflict; diagnostic.runtime_scope.contribution_invalid` |
| `validator.runtime_scope.game_system_migration_postcondition` | `type.operation.postcondition_evaluation_input@2` | `diagnostic.runtime_scope.receipt_binding_mismatch` |

Registryは上表をValidator IDのNFC UTF-8 byte順、version順へstrict sortする。各Artifact refは実行Targetのsigned implementation inventoryへexact一件、input LocalRefはactive Type recordへexact一件、各Diagnostic LocalRefはactive Diagnostic Registry recordへexact一件解決する。22件以外、missing、extra、duplicate、same-ID別hash、非canonical order、artifact／input／error hash不一致をRegistry全体のcompile errorにする。

各rowは`operation_local_ref`と`reachable_error_refs`を含むcanonical recordを保存する。fixtureは各Domain Validatorから一codeを削除、到達不能codeをOperationへ追加、ID同じcode違い、code同じID違い、Diagnostic hash stale、Validator hash staleを一原因ずつ注入し、set／ref equality不成立でRegistry全体を拒否する。特にRuntime Entry create／update／root migrationは`MIRAKAN-PROJECT-RUNTIME_ENTRY_BRANCH_FIELD_CONFLICT`、Runtime Entry create／update、Target Selector update、root migrationは`MIRAKAN-PROJECT-RUNTIME_ENTRY_DEFAULT_AMBIGUOUS`がsemantic Validatorから到達することをfixtureで証明する。Target Selector createは既存entryへattachしないためdefault coverageを変更せず、このDiagnosticをerrors／reachable setへ含めない。

Operation認可とrequest identityの唯一のDAGは本段落である。全Operation inputは選択したnamed input typeから`OperationIntentPayloadV2 {input_type_ref, operation_ref, risk_class, semantic_input_fields}`を作る。`semantic_input_fields`はそのschemaに存在する全意味Fieldをfield ID／presence discriminator込みで持ち、`operation_intent_hash`、`request_hash`、`MutationAuthorizationBindingV2`全体だけを除外する。別置き`authorization_ref`、anonymous approval shape、evidence hashをintentへ残さない。`operation_intent_hash = SHA-256(ASCII "MIRAKAN_OPERATION_INTENT_V2" || uint32_be(len(intent canonical bytes excluding operation_intent_hash)) || intent canonical bytes)`とし、count／array lengthはMCD canonical bytesへ明示、self-exclusionはintent hash自身だけである。Authorization、Approval、Predelegationはこの完成intent hashを共通subjectとして署名する。

状態変更inputは再計算済みintent hashとexact `MutationAuthorizationBindingV2`を必須にし、bindingのintent hash／risk／Operation／Project Scopeを照合する。R2はApprovalまたはPredelegationのexact一方、R3～R5はApprovalだけを許可する。R0／R1 non-mutationはbinding Field自体をcanonical omissionする。binding確定後、全Operation inputについて`canonical_input_without_request_hash = MCD canonical encode(選択input schemaに存在する全Fieldからrequest_hash Fieldだけを除外)`、`request_hash = SHA-256(ASCII "MIRAKAN_OPERATION_REQUEST_V2" || uint32_be(len(canonical_input_without_request_hash)) || canonical_input_without_request_hash)`で計算する。したがってfinal requestはintent hashとexact authority evidenceを含むが、evidenceはfinal request hashをsubjectにせず固定点を作らない。

Project-bound inputはOperation ref、input type ref、exact Project ref、policy refs、intent hash、named binding、Contract set root、全presence discriminatorを含む。projectless inputはProject refをschemaへ追加せず、後続Ownerが登録するexact workspace／catalog／resource ref等、そのinput schemaに実在する全Fieldを含む。sentinel／null Projectを捏造しない。input Type ref／schema discriminatorが異なるOperationは別canonical bytesになり、Project有無の差だけでhash式をversion-upしない。Domain文書は匿名sibling shapeや別式を定義しない。

`MIRAKAN_OPERATION_REQUEST_V2`と`MIRAKAN_OPERATION_INTENT_V2`は本計画更新時点でEngine、Project、Tool catalog、Receipt storeへ一度もActivation／materializationされていない設計契約であるため、version名を維持してActivation前に修正する。レビュー対象だった循環形のV2 bytesは永続artifactではなく、migration sourceとして受理しない。既存V1 retained artifactだけは直後のoffline-only migration recordで扱い、Task 4はこの二段階DAGとnamed bindingを実装入力とし、旧循環shapeの互換readerを作らない。

旧domain `MIRAKAN_OPERATION_REQUEST_V1`はretained Receiptの検証とoffline migration inputだけに許可する。Task 2では状態変更Operationを半登録せず、次の非current migration algorithm recordだけを定義する。

```text
RequestHashV1ToV2OfflineMigrationRecordV1
  migration_id: migration.contracts.request_hash_v1_to_v2
  migration_version: 1
  source_request_domain: MIRAKAN_OPERATION_REQUEST_V1
  destination_request_domain: MIRAKAN_OPERATION_REQUEST_V2
  source_input_schema_ref/hash
  destination_input_schema_ref/hash
  algorithm_artifact_ref/hash
  semantic_field_equality_policy_ref/hash
  fixture_refs[1..64]
  migration_record_hash: SHA-256

RequestHashV1ToV2MigrationCandidateV1
  migration_record_ref/hash
  source_input_ref/hash
  source_contract_set_hash
  source_request_hash
  destination_canonical_input_ref/hash
  destination_contract_set_hash
  destination_request_hash
  semantic_field_equality_evidence_ref/hash
  candidate_hash: SHA-256
```

本recordはMCD `operation`ではなく、Tool catalog、Trusted Service allowlist、current Operation／Policy／Validator Registryへ登録しない。algorithmは旧input bytes、旧hash、旧Contract set rootをread-onlyで検査し、意味Fieldを変えないV2 canonical candidateとequality evidenceだけをStagingへ生成する。Project revision、idempotency store、Receipt、Commit Markerを生成または変更しない。将来のgeneric schema-migration Operationが完全なMCD／Service／Policy／Validator／Approval／Commit DAGを伴って登録された場合だけ、このcandidateを入力に新Project revisionへ昇格できる。それまではV1 Projectを`migration_required / capability_unavailable`としてfail closedにする。`migration_record_hash`と`candidate_hash`は各ASCII `MIRAKAN_REQUEST_HASH_V1_TO_V2_OFFLINE_MIGRATION_RECORD_V1`、`MIRAKAN_REQUEST_HASH_V1_TO_V2_MIGRATION_CANDIDATE_V1`と自己Fieldを除くlength-framed canonical bytesから計算する。current Editor／MCP／CLI／Save／Replay／idempotency storeはV1 request hashを新規生成せず、V1とV2を同一keyとして比較しない。

```text
PreparedRuntimeEntryMutationReceiptPayloadV1
  publication_binding: exact PreparedReceiptPublicationBindingV1
  operation_ref: McdContractRefV1(kind=operation)
  operation_intent_hash
  request_hash
  idempotency_key
  before_project_ref: exact {project_id, project_revision, document_set_hash}
  after_project_ref: exact {project_id, project_revision, document_set_hash}
  affected_documents[1..4]: AffectedDocumentMutationV1
  root_scene_migration_plan_ref: RootSceneMigrationPlanRefV1 | null
  stable_id_allocation_mappings[0..4]
  preview_receipt_payload_ref/hash
  validation_receipt_payload_ref/hash
  materialization_context_ref/hash:
    PublishedReceiptMaterializationContextRefV1
  diagnostics[0..64]: DiagnosticCodeRefV1
  prepared_payload_hash: SHA-256

RuntimeEntryMutationReceiptV1
  published_receipt:
    exact PublishedDomainReceiptV2 whose
    prepared_domain_receipt_payload_ref/hash resolves
    PreparedRuntimeEntryMutationReceiptPayloadV1

AffectedDocumentMutationV1
  change_kind: created | updated
  document_kind: world
    before_world_document_ref: exact DocumentRef including content_hash | omitted
    after_world_document_ref: exact DocumentRef including content_hash
  | runtime_entry
    before_runtime_entry_document_ref: exact DocumentRef including content_hash | omitted
    before_runtime_entry_semantic_hash: RuntimeEntryPointSemanticHashV1 | omitted
    after_runtime_entry_document_ref: exact DocumentRef including content_hash
    after_runtime_entry_semantic_hash: RuntimeEntryPointSemanticHashV1
  | runtime_target_selector
    before_selector_document_ref: exact DocumentRef including content_hash | omitted
    before_selector_hash: RuntimeTargetSelectorHashV1 | omitted
    after_selector_document_ref: exact DocumentRef including content_hash
    after_selector_hash: RuntimeTargetSelectorHashV1
  | runtime_entry_activation_policy
    before_activation_policy_document_ref: exact DocumentRef including content_hash | omitted
    before_activation_policy_hash: RuntimeEntryActivationPolicyHashV1 | omitted
    after_activation_policy_document_ref: exact DocumentRef including content_hash
    after_activation_policy_hash: RuntimeEntryActivationPolicyHashV1
```

discriminator外branch Fieldを禁止する。`created`は選択branchの全before Fieldをcanonical omissionし、`updated`は選択branchの全before Fieldを必須にしてbefore／after Stable IDを一致させる。Worldへ普遍的なpayload semantic hashが存在すると仮定せず、Document content hashだけをexact ref内で記録する。entry、selector、activation policyは各Ownerが定義する別semantic hash型を使用し、generic `payload_semantic_hash`へ混同しない。

通常create／updateは対象一件、root Scene migrationは`world`、`runtime_target_selector`、`runtime_entry_activation_policy`、`runtime_entry`をこの順でexact一件ずつ持つ。通常Operationはplan refをnull、allocation mappingを空にする。root migrationのallocation mapping件数はProject Stateのexact `RootSceneMigrationPlanV1.document_mutations[]`にある`create` branch件数とexact equalityで0～4件、各create branchのallocation intentと一対一対応し、`update` branchはmappingを持たない。missing／duplicate／extra kind、null、zero hash、create count不一致、updateへのallocation、単数`affected_document`への圧縮を拒否する。`prepared_payload_hash`はASCII `MIRAKAN_PREPARED_RUNTIME_ENTRY_MUTATION_RECEIPT_PAYLOAD_V1`とself-excluding payloadから計算する。唯一のsigned subjectは共通`PublishedDomainReceiptPayloadV2`の完成JCS bytesであり、Domain固有Subject／署名wrapperを作らない。署名済みwrapper保存後、`PublicPublicationMarkerV1`とafter Projectを同じpublic CASで発行する。private Marker、prepared payloadだけを公開authorityにせず、同じ`idempotency_key`＋`request_hash`のretryは同じResult／signed Receipt／Public Markerを返し、同じkeyの別request hashは`MIRAKAN-OPERATION-IDEMPOTENCY_KEY_REUSE`で一切変更せず拒否する。

Operation Registry compilerは八Operation ID、16 predicate policy ID、二rate-limit policy ID、三predicate Type、二Prepared auxiliary payload Type、全domain input／output／receipt ref、Service二record、SideEffect enum、Diagnostic ref、Validator closureをexact Contract set／各Registryへ解決する。pre／postcondition refがmissing、`kind!=policy`、version／Contract set hash stale、policyが`evaluation_mode!=pure`、side effect非空、IO／result Type不一致、Prepared auxiliary Type／schema不一致、Service allowlist不一致、rate-limit payload不一致、Diagnostic四Field不一致、またはerrors／reachable set不一致ならRegistry全体を拒否する。fixtureは八Operationのmeta-schema compileとProject ownerとのset equalityに加え、各参照のwrong-kind、missing、stale version、stale Contract set／content hash、impure policyを一原因ずつ拒否する。

### 8.2 Current active Domain Operation closure

§8.1のbootstrap八件に加えてcurrent Contract setへ存在するDomain Operationは次の六件だけであり、current active Operation総数はexact 14件である。§§20～21のplanning candidate 159件、非活性Scenario vocabulary、§21.2のexample／pending／rejected ID、legacy aliasはこの集合に含めない。

| Owner | exact active Operation LocalRef集合 | exact active Policy LocalRef数 |
|---|---|---:|
| Performance | `{operation.performance.migrate_project_scale_envelope,1}` | 3 |
| Physics | `{operation.physics.intent_role.migrate,1}` | 3 |
| World | `{operation.world.allocate_generated_stable_ids,1}` | 3 |
| Shooter | `{operation.shooter.target_provider_binding.create,1}; {operation.shooter.target_provider_binding.update,1}; {operation.shooter.target_provider_binding.select,1}` | 7 |

各Ownerについて、`Owner Manifest Operation LocalRefs = active MCD Operation LocalRefs = Trusted Service allowlist owner contribution`をID／versionで比較する。さらに各Operationの`preconditions[] ∪ postconditions[] ∪ {rate_limit_policy}`はOwner文書が列挙するPolicy LocalRef subset、`errors[]`はValidator closureのreachable Diagnostic LocalRef union、`receipt_type`はOwner Manifest Receipt Type、`authority`はallowlistを持つService local recordとexact equalityでなければならない。Worldの`provider_exposure=trusted_internal`ではProvider／MCP projection集合を空、他五件の`provider_exposure=mcp_proposal`では完全登録済みOperation ref集合と生成projection集合をexact equalityにする。prefix、説明文、family名から集合を展開しない。

current `TrustedServiceRegistryV2`の二`TrustedServiceLocalRecordV2`は、bootstrap seedと上表のOwner contributionを同じContract set transactionでmergeした次のexact LocalRef配列を持つ。配列はOperation IDのASCII byte昇順、同IDではversion昇順であり、別順序、件数だけ一致する別ID、prefix展開を拒否する。

```text
service.authoring_command_gateway@1
  allowed_operation_local_refs[11] = [
    {operation.project.runtime_entry.create,1}
    {operation.project.runtime_entry.migrate_root_scene,1}
    {operation.project.runtime_entry.update,1}
    {operation.project.runtime_entry_activation_policy.create,1}
    {operation.project.runtime_entry_activation_policy.update,1}
    {operation.project.runtime_target_selector.create,1}
    {operation.project.runtime_target_selector.update,1}
    {operation.shooter.target_provider_binding.create,1}
    {operation.shooter.target_provider_binding.select,1}
    {operation.shooter.target_provider_binding.update,1}
    {operation.world.allocate_generated_stable_ids,1}
  ]
  service_local_content_hash =
    SHA-256(MIRAKAN_TRUSTED_SERVICE_LOCAL_RECORD_V2,
      self-excluding canonical current merged TrustedServiceLocalRecordV2)

service.offline_project_migrator@1
  allowed_operation_local_refs[3] = [
    {operation.performance.migrate_project_scale_envelope,1}
    {operation.physics.intent_role.migrate,1}
    {operation.runtime_scope.migrate_game_system,1}
  ]
  service_local_content_hash =
    SHA-256(MIRAKAN_TRUSTED_SERVICE_LOCAL_RECORD_V2,
      self-excluding canonical current merged TrustedServiceLocalRecordV2)
```

両`service_local_content_hash`は上記配列と既存のexecutable identity、Capability、isolation profileを含む完成local recordから再計算する。各Service member hashと`TrustedServiceRegistryV2.registry_content_hash`を別domainで生成し、Contract set root確定後にだけroot付きOperation／Capability／profile refを持つ二つの外部`TrustedServiceRecordV1.service_content_hash`と`TrustedServiceRefV1`を生成する。local hash、member hash、Registry hash、external hashの等値を要求せず、後段の値を前段preimageへ戻さない。bootstrap hashの再利用、Domain contributionだけの別record、11／3件の一件欠落／追加／入替えをcompile failureにする。

六OperationのContract set closureは、Operation 6件、参照Policy 16件、下表のdirect reachable Type exact 26件、World Requirement 1件、Service owner contribution、Validator local record、Operation Validator Closure local record、reachable Diagnostic exact 34件をすべて`ContractSetSnapshotV2.members[]`へ実recordとして含める。Type 26件は§8.1と共有するcommon五件を一度だけ数え、Owner固有21件とのdisjoint unionにする。各Typeはcomplete MCD local record、self-excluding local hash、Type member hashを持ち、nested schema refも同じSnapshot内のexact LocalRefへ再帰解決する。外部root付きType ref、bare ID、同名別versionをmember preimageへ入れない。

| Type owner | exact件数 | exact Type LocalRef ID集合（全version 1、postconditionだけversion 2） |
|---|---:|---|
| Common publication／predicate | 5 | `type.operation.precondition_evaluation_input; type.operation.postcondition_evaluation_input; type.operation.predicate_result; type.operation.prepared_preview_receipt_payload; type.operation.prepared_validation_receipt_payload` |
| Performance | 4 | `type.performance.project_scale_envelope_migration_input; type.performance.project_scale_envelope_migration_result; type.performance.project_scale_envelope_migration_receipt; type.performance.prepared_project_scale_envelope_migration_receipt_payload` |
| Physics | 4 | `type.physics.intent_role_migration_input; type.physics.intent_role_migration_result; type.physics.intent_role_migration_receipt; type.physics.prepared_intent_role_migration_receipt_payload` |
| World | 4 | `type.world.stable_id_allocation_input; type.world.stable_id_allocation_result; type.world.stable_id_allocation_receipt; type.world.prepared_stable_id_allocation_receipt_payload` |
| Shooter | 9 | `type.genre.shooter.target_provider_binding_create_input; type.genre.shooter.target_provider_binding_update_input; type.genre.shooter.target_provider_binding_select_input; type.genre.shooter.target_provider_binding_mutation_result; type.genre.shooter.target_provider_binding_selection_result; type.genre.shooter.target_provider_binding_mutation_receipt; type.genre.shooter.target_provider_binding_selection_receipt; type.genre.shooter.prepared_target_provider_binding_mutation_receipt_payload; type.genre.shooter.prepared_target_provider_binding_selection_receipt_payload` |

各Ownerのprepared Domain payload Type recordは対応named `Prepared*ReceiptPayloadV1`の全Fieldをexact投影し、その`publication_binding`を通じてcommon二Prepared auxiliary payloadと同じOperation／intent／request／idempotency／before／after／Contextへbindする。最終Receipt Typeとprepared payload Typeは別LocalRefであり、相互代用しない。26件の一件missing／extra／duplicate、common TypeのOwnerごとの複写、prepared↔final Receipt Type swap、wrong version／kind／schema hashをContract set compile failureにする。

Diagnostic 34件はcommon 8件とowner固有26件のdisjoint unionで、owner固有subsetを次に固定する。

| Owner | exact固有件数 | exact Diagnostic ID集合 |
|---|---:|---|
| Performance | 4 | `diagnostic.performance.scale_v1_source_invalid; diagnostic.performance.workload_domain_unresolved; diagnostic.performance.scale_migration_ambiguous; diagnostic.performance.scale_receipt_binding_mismatch` |
| Physics | 7 | `diagnostic.physics.intent_role.source_invalid; diagnostic.physics.intent_role.registry_invalid; diagnostic.physics.intent_role.contribution_missing; diagnostic.physics.intent_role.contribution_ambiguous; diagnostic.physics.intent_role.capability_unavailable; diagnostic.physics.intent_role.axis_mapping_invalid; diagnostic.physics.intent_role.receipt_binding_mismatch` |
| World | 6 | `diagnostic.world.generated_candidate_invalid; diagnostic.world.stable_id_project_binding_mismatch; diagnostic.world.stable_id_allocation_count_mismatch; diagnostic.world.stable_id_manifest_invalid; diagnostic.world.stable_id_receipt_signing_failed; diagnostic.world.stable_id_publication_conflict` |
| Shooter | 9 | `diagnostic.genre.shooter.target_provider.identity_mismatch; diagnostic.genre.shooter.target_provider.owner_mismatch; diagnostic.genre.shooter.target_provider.template_mismatch; diagnostic.genre.shooter.target_provider.type_mismatch; diagnostic.genre.shooter.target_provider.hash_mismatch; diagnostic.genre.shooter.target_provider.target_unsupported; diagnostic.genre.shooter.target_provider.fixture_in_production; diagnostic.genre.shooter.target_provider.registry_invalid; diagnostic.genre.shooter.target_provider.receipt_binding_mismatch` |

common exact 8件は`diagnostic.conflict.revision_mismatch; diagnostic.authorization.denied; diagnostic.approval.required; diagnostic.authoring.lock_conflict; diagnostic.mcd.operation_predicate_invalid; diagnostic.operation.timeout; diagnostic.operation.rate_limit_exceeded; diagnostic.operation.idempotency_key_reuse`である。各Owner Manifest Diagnostic subset、各Operation `errors[]`、Validator reachable union、Snapshot Diagnostic local memberのlogical集合をset equalityにし、Shooterのcomposition／profile等、active三Operationの17-ref closureに到達しない別Diagnosticを34件へ混入させない。

fixtureは型ごとに実在するFieldだけを変異させる。16 Policyはcommon envelopeまたはpredicate／rate payload、World Requirementはcommon envelopeまたはRequirement payload、34 Diagnosticの各recordは`owner_local_ref`、`code`、`severity`、`category`、`message_key`、`requirement_local_refs`、`retryability`のいずれか一Fieldだけを変えるcaseを最低一件ずつ生成し、該当local hash、member hash、Contract set root、外部record hashが変わり、旧Manifest／Service／Policy／Validator／Diagnostic／Receipt external refとのset equalityが失敗することを検証する。Owner ref mutationは別の有効Owner identityへ変更した場合も該当Owner Manifest subsetとのequalityを失敗させる。34件の一件missing／extra／重複／cross-owner substitution、bare ID、name-only active record、root確定前のexternal ref、member byte変更後も同じrootを受理する実装を拒否する。

### 8.3 Qualification依存のclosed分類

current文書、schema、plan、specに現れる`qualification_receipt*`／Qualification Receipt参照は、compilerが次の三classのexact一つへ分類する。未分類、複数class、名前だけからの推測を拒否する。

| class | 許可するedge | current exact対象 |
|---|---|---|
| `receipt_free_base` | Receipt／wrapper／Binding／Activation projection／Fixture refへのedgeは0件。self hash、Registry hash、Contract set rootを先に確定する | Game System Spec／auxiliary、Runtime Scope／Physics migration Contribution、Performance Domain／Intent／Envelope／Mapping／Dimension、Input Binding Record／Contribution、Physics Role、Navigation Provider（`UsageTaggedImplementationSystemBaseRefV1`だけ）／Binding、Procedural World／Blockout／World Plan／Bundle／generated Delta、Pack Recipe／Profile／Manifest、Project Shader Module／Target Support、Material／Post Process Node／Profile、VFX Extension Manifest |
| `safe_downstream_projection` | 完成base refをsubjectにするQualification subject→signed Receipt→Bindingの後段、またはそのBindingを消費するSelection／Manifest／Compile／Save／Replay／Derived resolution。Receipt subjectから当該projectionへ到達するedgeは0件 | 各owner-typed Qualification subject／Receipt／Activation Binding、Game System／Fixture System Activation Binding、System base refにそのActivation Bindingを加えた`UsageTaggedImplementationSystemRefV1`、同refを依存として署名するMotion Executor Provider subject、Shooter Target Provider Binding、Runtime Scope／Performance／Physics／Input activation catalog、Pack Recipe／World／Project Shader／Material Project Node／Post Process／VFX activation projection、Operation Manifest、Project `TargetReadinessV1(state=qualified)`、Renderer／Lighting／LOD resolved plan・evidence、Governance Attestation／Product gate／Toolchain qualified capacity |
| `historical_or_planning` | current Registry、Catalog、Manifest、Service allowlist、Runtime Packageへprojectionされないread-only記録だけ | `docs/superpowers/{plans,specs}`の設計履歴、retained ADRの旧shape、Product Planのplanning-only Future／Local multiplayer profile、§§20–21の未Activation candidate |

`safe_downstream_projection`はDAG stageごとのclosed unionである。Qualification Subject stageは共通`QualificationSubjectValidatorV1`がbase ref／subject ref、owner、Target集合、dependency集合を検証し、signed ReceiptまたはBinding refを入力にしない。signed Receipt stageは`QualificationReceiptValidatorV1`が完成Subject hash、wrapper purpose／subject／署名／freshness／revocationを検証し、Bindingを入力にしない。`TargetReadinessV1`、Technical Qualification、Attestation、Product gate等のdirect Evidence consumerは`DirectEvidenceProjectionValidatorV1`が先に固定したartifact／input closureとexact typed Receipt refのsubject／wrapperを照合し、存在しないDomain Activation Bindingを要求しない。この三stageへ後段Fieldを追加して循環させることをcompile failureにする。

次のvalidation-only normalized viewへtotalに写像する対象は、上表のconcrete Activation／Qualification Bindingと、そのBinding refを直接保持または解決する外部consumer projectionだけである。Subject／Receipt stageとdirect Evidence consumerはこのviewの対象外で、上記stage別validatorを使う。viewは永続MCD、base、Receipt、Binding、ProjectionのFieldではなく、hash preimageや新しい参照edgeを作らない。

```text
QualificationActivationBindingEnvelopeV1
  consumer_base_ref_canonical_hash: SHA-256
  binding_base_ref_canonical_hash: SHA-256
  subject_base_ref_canonical_hash: SHA-256
  receipt_subject_base_ref_canonical_hashes[1..256]: SHA-256
  signed_receipt_ref_set_hash: SHA-256
  owner_ref_canonical_hash: SHA-256
  subject_owner_ref_canonical_hash: SHA-256
  target_ref_set_hash: SHA-256
  receipt_target_ref_set_hash: SHA-256
  dependency_ref_set_hash: SHA-256
  receipt_dependency_ref_set_hash: SHA-256
  binding_generation: {binding_id, binding_version, binding_content_hash}
```

各hashは解決済みDomain typed valueまたはstrict sort／unique済みtyped集合のcount／length-framed canonical bytesから導出し、該当集合が空のDomainもcanonical empty-set hashを必須にする。共通`QualificationActivationBindingValidatorV1`はconsumer／Binding／Subject／全Receipt subjectのbase hashをbyte equality、owner hashをbyte equality、target／dependency set hashをexact equality、Receipt ref集合をpass／fresh／non-revoked／strict sort／unique、generation tupleを解決済みBinding refとexact equalityにする。BindingまたはBinding consumerのDomain固有schemaにconsumer base、owner、target、dependencyのいずれかを導出するFieldがなく、そのDomain契約でもcanonical emptyと宣言されていない場合はadapter未定義としてcompile failureにする。このviewのhash同士をDomain base、Subject、Receiptへ保存しない。

同じBinding validator fixture matrixを上表の全concrete BindingとBinding-consuming外部consumer projectionへ適用する。fixtureはstale base revision／content hash、別の有効base／owner／target／dependency／Bindingへのsubstitution、Receipt subject一件だけの差替え、Receipt missing／extra／duplicate／順序違反、target／dependency missing／extra、generation ID／version／hash不一致を各一原因で変異し、全件をrejectしなければならない。Subject stageはbase／owner／target／dependencyのsubstitution、Receipt stageはsubject／purpose／署名／freshness／revocation、direct Evidence stageはartifact closure／typed Receipt subject／wrapper refのsubstitutionを各専用matrixで拒否する。Domain契約がより強いpairing、cardinality、set-equality規則を持つ場合はそのfixtureを追加し、該当stageの共通matrixを省略しない。

`safe_downstream_projection`に属する、Field名として`qualification_receipt*`を現在保持するexact schemaは、`InferenceDeploymentProfileV1`、`SystemQualificationReceiptV1`、`SystemTechnicalAttestationV1`、`CodeOwnerAssignmentV1`、`GameSystemActivationBindingV1`、`FixtureImplementationSystemActivationBindingV1`、`RuntimeScopeMigrationQualificationBindingV1`、`SystemBundleChangeSetV1`、`TargetReadinessV1`とqualified Runtime Package Target closure、`PerformanceQualificationBindingV1`、`MotionIntentBindingQualificationBindingV1`、`MotionExecutorProviderActivationBindingV1`、`PhysicsIntentRoleActivationBindingV1`、ephemeral `PhysicsIntentResolutionV1`、`PhysicsIntentRoleMigrationQualificationBindingV1`、`PostProcessActivationBindingV1`、`ProjectShaderActivationBindingV1`、`ResolvedShadowPlanV1`、`VfxExtensionActivationBindingV1`、`WorldSourceActivationBindingV1`、`SemanticActionBindingQualificationBindingV1`、`PackRecipeActivationBindingV1`である。各Domainの同名concrete instanceはこのschemaのinstanceであり、別classを作らない。これ以外のcurrent architecture schemaへ同名Fieldを追加する変更は、この分類表を同じContract set transactionでversion-upしない限りcompile failureとする。

`historical_or_planning`のexact残存schemaはProduct Planの`LocalPlaySessionProfileV1`、retained ADRの`RuntimeWorldQualificationBindingV1`、`docs/plans`の`D3d12DecisionRecordV1`／`ControlPlaneArtifactQualificationManifestV1`、`docs/superpowers/specs`の旧`CompositionRecipeV1`／`CodeOwnerAssignmentV1`である。これらの文書内にある説明上のQualification／Receipt文字列も同classであり、current schema集合を増やさない。`docs/architecture`のproseに現れるField名以外のQualification／Receipt文字列は、上記safe schemaの検証規則、Receipt wrapper自身、または§§20–21の未Activation work itemを説明する参照であって、新しいedgeをmaterializeしない。

`receipt_free_base`のcurrent schema集合に`qualification_receipt_ref(s)`、`signed_receipt`、`MirakanSignedRecordV1`、Qualification／Activation Binding ref、fixture body refが一件でも存在すればcompile failureであり、current許容件数はexact 0である。compilerはField名検索だけでなく、各Qualification Receipt subjectから全hash／ref edgeを逆向きにwalkし、subjectがbindするbase self hash／Registry root／Contract set rootのpreimageへReceipt、wrapper、Binding、Activation projectionが到達しないことを検証する。許可する唯一の順序は`receipt-free base → base ref/root → Qualification subject → signed Receipt → root外Binding → downstream projection`である。

`safe_downstream_projection`はReceipt refをhashへ含められるが、そのprojection hashを同Receipt subject、base ref、base Registry、Contract set rootへ戻してはならない。Project StateのTechnical Qualification、Renderer／LightingのResolved plan、AI Verification／SecurityのAttestation、Product gate、Toolchain capacityは先に固定したartifact／input closureをsubjectにするEvidence projectionであり、projection自身をsubjectにしない。`historical_or_planning`はActivation時に自動昇格せず、将来current化するtransactionで同じDAGへ書き直す。fixtureは各classの誤分類、baseへのReceipt一Field追加、subjectへのdownstream hash追加、Bindingから別baseへのsubstitutionを一原因ずつrejectする。

## 9. State machine定義

State machineは次を必須とする。

- 共通`id`と`version`。
- finiteな`states`。
- 一つの`initial_state`。
- `terminal_states`。
- 型付き`events`。
- `transitions`の`from`、`event`、`guard`、`actions`、`to`。
- machine invariant。
- safety property。
- 必要なliveness property。
- timeout／cancel／crash recovery transition。
- invalid transitionのDiagnostic。

同じ`from`と`event`で複数transitionを許可する場合、Guardは相互排他的であることをContract lintで証明可能なpredicateへ限定する。証明できなければ定義を拒否する。default transition、暗黙self-loop、unknown event無視を禁止する。

State machineから次を生成する。

- C++ transition tableとexhaustive enum。
- TypeScript reducer。
- Mermaid state diagram。
- 全合法transitionのpositive test。
- 全非合法state／event組合せのnegative test。
- TLA+ modelとのtransition mapping file。

## 10. Capability定義

CapabilityはAIとEditorが「何を作れるか」を理解する正規単位である。

| Field | 内容 |
|---|---|
| 共通`id` | 例`capability.render.material.toon`。maturityとschema versionをIDへ埋め込まない |
| `maturity` | C0からC3 |
| `supported_targets` | Target Profile ID |
| `required_capabilities` | Dependency DAG |
| `conflicts` | 同時利用不可Capability |
| `authoring_types` | Component／Asset／Graph型 |
| `operations` | Query／Command／Event |
| `validators` | Structural／semantic／budget |
| `quality_profiles` | tier、fallback、disable条件 |
| `budgets` | CPU、GPU、memory、disk、network |
| `failure_modes` | Diagnosticとfallback |
| `examples` | 最小valid例、境界例、invalid例 |
| `ai_guidance` | 選択条件、禁止用途、質問条件 |

`ai_guidance`は説明補助であり、Validatorを置き換えない。Capability検索はID、tag、Target、maturity、dependencyから行い、AIへ全Capabilityを毎回送らない。

### 10.1 Game System定義

MCD kind `game_system`はEngine Capabilityの利用者であるGameplay単位を表す。完全なField、namespace、State owner、dependency、implementation、Bundle規則は[Gameplay programming modelの`GameSystemSpecV2`](../03-authoring/gameplay-programming-model.md#3-gamesystemspecv2)だけが所有する。MCDはそのexact Specを登録・検証し、Fieldの部分集合や別Envelopeを再定義しない。

Contract compilerはactive `game_system`集合から`GameSystemCatalogV1`、`GameSystemDependencyGraphV1`、State owner table、C++／TypeScript binding、Editor metadata、System conformance testを生成する。Project-defined Systemも同じkindを使い、`game_system.project.<project_namespace>.<path>`へ登録する。Engine標準Catalogを利用可能Systemの固定Whitelistとして扱わない。

Authoritative Typeはactive Contract set内で厳密に一つのGame Systemだけが所有できる。Owner欠落、複数Owner、Build／Cook cycle、同phase再入cycle、Presentationからauthoritative Stateへのwrite edgeをsemantic compile errorにする。

## 11. PolicyとProfile

### 11.1 Policy

PolicyはRisk、Approval、Network、Dependency、Data、Budget、Retry、Release gateを定義する。Policy値の変更はR3以上とし、既存Taskへ遡及適用しない。Task開始時のPolicy set hashをEnvelopeへ固定する。

### 11.2 Profile

ProfileはTarget、Quality、Device、Workspace、AI Provider、Benchmarkを表す。Profileの継承は一段だけ許可し、multiple inheritanceを禁止する。解決後Profileをflat canonical JSONとして生成し、そのhashをBuildとTaskへ保存する。

Profileに存在しない値を環境変数やProvider defaultから暗黙補完しない。Host固有値は`toolchain.lock.json`、Game／Runtime値はMCD Profileへ置く。

`provider_profile`はProvider、API version、受理するSchema keyword、strict機能、Tool／Schema／Context上限、refusal／incomplete形式、retention capabilityを表す正規入力である。Provider固有Projectionそのものは正本にせず、このProfileとType／OperationからBuild treeへ生成する。Provider Manifestはexact Provider Profile ID＋version＋hashを参照する。

### 11.3 C++ Frontend／Dependency／Build Driver

`CxxFrontendProfileV1`のclosed enum、許可遷移、Promotion可否は[C++23／Modules](cpp23-modules.md#4-一方向の移行state)だけが所有する。MCDはexact Profile IDを参照し、ProviderやAIによるProfile ID追加を拒否し、Build Gatewayが`toolchain.lock.json`のCompiler／STL／CMake bindingと照合する。

`CppDependencySetV1`はowner component／Primary Module、public／private import、closed `StdHeaderId`、closed Header例外を正規化して表す。AIはraw include pathやCompiler flagをDependencyとして保存しない。Contract compilerはCX0で個別標準Header、CX1以降でNamed Module／`import std`へ投影し、Source scannerは実Sourceとの一致を検証する。Field、順序、Header例外、Cutover後のProjection停止条件はC++言語・Modules規約を基準とする。

`BuildDriverProfileV1`は`driver_profile_id`、`target_profile_id`、`allowed_frontend_profile_ids`、`configure_driver`、`cpp_generator`、`configuration_model`、`package_owner`を持つ。IDと組合せは[Toolchain／Dependencies](toolchain-dependencies.md#3-build-driver-matrix)のBuild Driver matrixのclosed setだけを許可し、AI、Provider、Project、Environmentが任意Driver、Generator、commandを追加できない。Contract compilerはWindows／Appleのchecked-in CMake Preset検査表、Android Gradle CMake検査表、Build Gateway allowlistへ投影するが、CMake／Gradle SourceそのものをMCDへ埋め込まない。

ValidatorはTarget、C++ Frontend Profile、Driver Profileの全組合せを照合し、First-party Makefiles／`ndk-build`、Android Ninja Multi-Config、Generator override、異なるBuild tree identityの再利用を拒否する。

## 12. Diagnostic契約

### 12.1 `MirakanDiagnosticV1`

Engine、Contract compiler、Provider adapter、MCP、CLIは共通の`MirakanDiagnosticV1`を返す。

Diagnostic codeの正本は次のRegistry recordである。Operation、Validator、Result、Receiptは裸codeを保存せず、四FieldがRegistry recordとexact equalityの`DiagnosticCodeRefV1`だけを使う。

```text
DiagnosticCodeRecordV1
  diagnostic_id: diagnostic.<lower-token-path>
  code: MIRAKAN-<DOMAIN>-<CONDITION>
  diagnostic_version: uint32
  owner_ref:
    exact {owner_id, owner_revision, owner_content_hash}
  severity: info | warning | error | blocking
  category:
    schema | semantic | permission | conflict | build | test |
    performance | security | provider | infrastructure
  message_key
  requirement_refs[0..64]: McdContractRefV1(kind=requirement)
  retryability: never | after_input | after_change | transient
  diagnostic_content_hash: SHA-256

DiagnosticCodeRefV1
  diagnostic_id
  code
  diagnostic_version: uint32
  diagnostic_content_hash: SHA-256

DiagnosticCodeRegistryRefV1
  registry_id
  registry_version: uint32
  registry_content_hash: SHA-256

DiagnosticCodeRegistryV1
  registry_id: diagnostic_code.registry.active
  registry_version: uint32
  registry_content_hash: SHA-256
  records[1..65536]: DiagnosticCodeRecordV1
```

`diagnostic_id`と`code`は一対一であり、同じIDの別code、同じcodeの別ID、同じID／versionの別content hashを拒否する。`owner_ref`はroot前の`DiagnosticOwnerLocalRefV1`と三Fieldexact equalityであり、owner IDのprefixやManifest membershipから補完しない。recordは`diagnostic_content_hash`を除く全Field、Registryは`registry_content_hash`を除きASCII `MIRAKAN_DIAGNOSTIC_CODE_REGISTRY_V1`、Registry ID／version、record count、`diagnostic_id`のNFC UTF-8 byte順にstrict sortしたrecord canonical bytesを、各byte列の`uint32_be` length付きでhashする。duplicate、非canonical order、unknown version、owner mismatch、hash mismatchではRegistry全体をfail closedにする。`DiagnosticCodeRefV1`は四Fieldすべてを同一recordへ解決し、IDだけまたはcodeだけが一致するrefを受理しない。

| Field | 型／規則 |
|---|---|
| `code_ref` | exact `DiagnosticCodeRefV1`。instance内でID／code／versionを再宣言しない |
| `severity` | `code_ref`解決先recordとexact equalityの`info \| warning \| error \| blocking` |
| `category` | `code_ref`解決先recordとexact equalityのclosed category |
| `message_key` | `code_ref`解決先recordとexact equalityのLocalization key |
| `arguments` | primitive map。完成文だけを保存しない |
| `artifact_id`／`revision` | 対象 |
| `location` | JSON Pointerまたはnormalized source location |
| `target_stable_ids` | 実在確認済み対象ID。候補の場合は候補理由を`arguments`へ含める |
| `requirement_ids` | 1件以上。Infrastructureだけ例外 |
| `expected`／`actual` | redacted typed value |
| `remediation_ids` | 機械実行可能または人間向け修正案 |
| `retryability` | `code_ref`解決先recordとexact equalityの`never \| after_input \| after_change \| transient` |
| `cause_chain` | 子`DiagnosticCodeRefV1` array。裸ID／codeを許可しない |
| `trace_id` | Verification trace参照 |

AIへ返すErrorはこの構造を維持する。Provider向け説明文だけへ変換してcode、location、expected、actualを失わない。Source／static analysis結果はこの形式を正本とし、外部Tool連携用にSARIF 2.1.0へexportする。

`MirakanDiagnosticV1`は検証結果と修復入口の正本であり、Debug Event全般の代替ではない。Debugging規約の`DebugEventEnvelopeV1`は必要時に`code_ref`／`cause_chain`／`trace_id`を参照し、severity、location、expected／actualを別Schemaへ重複保存しない。反対に高頻度counter、span、frame marker、domain snapshotを`MirakanDiagnosticV1`として発行してはならない。

### 12.2 `RemediationV1`

`remediation_ids`は自由文ではなく、MCDの`RemediationV1`を参照する。

| Field | 規則 |
|---|---|
| `id`／`version` | `remediation.<domain>.<lower_snake_name>`、意味変更ごとにversion増加 |
| `applicable_codes` | exact `DiagnosticCodeRefV1`のclosed set |
| `required_queries` | exact query Operation ID＋version、field mask、最大件数 |
| `operation_template` | typed Command ID＋固定field／placeholder定義。任意JSON禁止 |
| `preconditions`／`postconditions` | Predicate ID array |
| `risk_class`／`required_approvals` | 元Taskより権限を弱めない |
| `retryable_categories` | `schema \| semantic \| conflict \| build \| test \| performance \| provider \| infrastructure`の許可subset |
| `forbidden_categories` | `permission`、`security`、lock／approval／revision driftを必須化 |
| `max_applications_per_task` | 1または2。0と3以上を禁止 |
| `human_message_key` | Localization key |

RemediationはDiagnosticの解決候補であり、適用権限ではない。Gatewayは現在revision、Envelope、Risk、Approval、preconditionを再検証する。該当しないtarget、未知placeholder、権限追加、Source直接writeを含むtemplateはContract compile errorとする。同じnormalized blocking Diagnostic集合へ同じRemediationを二回適用しても減少しないfixtureはinvalidとし、Providerへ公開しない。

## 13. CanonicalizationとHash

MCD JSONはparse後のData modelをRFC 8785 JSON Canonicalization Scheme（JCS）でcanonicalizeし、SHA-256を計算する。Source fileの空白やkey順ではなく、意味上のJSON値をhash対象にする。

`/schemas/contract.lock.json`は次を持つ。

- `contract_set_version`。
- MCD meta-schema version。
- 各正本Fileのpath、kind、ID、version、canonical SHA-256。
- 全集合のMerkle root。
- Contract compiler artifact hash。
- 各Projection generator version。
- Generated golden output root hash。
- 参照するToolchain lock hash。

Lock更新はContract compilerだけが行う。Inputが同じなら、時刻、machine path、random IDをGenerated contentへ含めず、byte-for-byte同じ出力を生成しなければならない。Timestamp、Runner、Build IDはGeneration Receiptへ分離する。

Merkle leafは正規File pathのUTF-8 byte列を`p`、DocumentのJCS SHA-256 32 byteを`d`として、`SHA-256(0x00 || uint32_be(len(p)) || p || d)`とする。Leafを`p`のunsigned byte lexicographic順に並べ、親を`SHA-256(0x01 || left32 || right32)`とする。奇数Nodeの最終要素は同じ値を`right`へ複製し、1 LeafならそのLeaf hashをrootとする。空Contract setは禁止する。`/schemas/contract.lock.json`自身とGenerated outputはこのMCD rootへ含めず、それぞれ別hashとしてLockへ記録して循環参照を作らない。

## 14. Contract compiler

`tools/contract_compiler`は、[Toolchain／Dependencies](toolchain-dependencies.md)で固定したJavaScript／TypeScript ESM toolchainでBuildするfirst-party CLIとする。Runtime dependencyではなく、offline Build toolである。TypeScript compiler programmatic APIへ依存せず、通常の`tsc`でcompiler自身をBuildする。

JSON treeとJCS実装は[Toolchain／Dependencies](toolchain-dependencies.md)が固定するBuild-only packageを使う。Inputは`Buffer`から`TextDecoder("utf-8", {fatal:true})`でdecodeし、`parseTree`を`disallowComments=true`、`allowTrailingComma=false`で呼ぶ。Tree上の全Property occurrenceを走査してdecoded key重複を拒否した後だけData modelへ変換する。`parse()`または`JSON.parse`で重複情報を失ってから検査してはならない。

固定packageと公式JCS fixture setのversion、commit、artifact rootは[Toolchain／Dependencies](toolchain-dependencies.md)が所有する。採用検証はcomment／trailing comma拒否、`"a"`と`"\u0061"`のProperty occurrence保持、公式input／output fixtureのbyte一致を確認する。Fixture Artifact rootは13節のleaf／parent framingだけを再利用し、`p`をRepository相対の`testdata/input/<name>`または`testdata/output/<name>`、`d`を各Fixtureの**raw file byte**のSHA-256とする。入力Fixtureは意図的に非canonicalなため、MCD用のJCS document hashへ置き換えない。この検証を`contract-fast` CIへ固定する。

処理順序を固定する。

1. RFC 8259としてparseし、重複keyを拒否する。標準`JSON.parse`だけでは重複keyを検出できないため、UTF-8 byte列へduplicate-aware tokenizerを先に適用し、Object scopeごとのdecoded key一致を検査する。同時に数値tokenのraw textが4節の±(2^53-1)制約を満たすことを検査し、超過literalを拒否する。
2. Kind別meta-schemaでvalidateする。
3. 全IDとversionをindex化する。
4. 参照解決、cycle、Game System State owner、phase edge、requirement coverageを検査する。
5. Semantic lintとPolicy lintを実行する。
6. Canonicalizeして`/schemas/contract.lock.json`と照合する。
7. Toolchain lockが指定するInternal JSON Schema dialectを生成する。
8. Language bindingを生成する。
9. Provider／MCP projectionを生成する。
10. Docs、fixture、transition testを生成する。
11. Golden output hashと再生成差分を検査する。

Generator plugin方式は初期に採用しない。全Generatorは同一Repository、同一Process、exhaustive dispatchで管理する。外部Provider追加はContract compiler sourceの明示変更とProvider conformance suiteを必要とする。

Duplicate-aware走査はString escapeをdecodeした後のKeyで比較するため、`"a"`と`"\u0061"`も重複として拒否する。ParserとJCSにはRFC 8259／8785のofficial test vector、invalid UTF-8、unpaired surrogate、深さ／byte上限、Property-based differential testを必須にする。Package更新はR3 Dependency Changeであり、同じfixtureからbyte-for-byte同じJCSを出す場合だけ昇格できる。

## 15. Internal JSON Schema projection

Internal validation projectionは[Toolchain／Dependencies](toolchain-dependencies.md)が固定するJSON Schema dialectを使い、`$schema`を必ず明示する。可能な構造制約をすべて表現するが、次は手書きまたは生成semantic validatorへ分離する。

- Entity間参照整合性。
- Target Capabilityの組合せ。
- Runtime phaseとwriter authority。
- Game Systemのauthoritative State owner、dependency cycle、Implementation Variant conformance。
- World／Scene／Space／Topology identity、owner-typed content、Streaming PlanのSource／Derived境界。
- Asset revisionの存在。
- CPU／GPU／memory budget。
- Permission、Approval、Risk。
- GameplayDefinition禁止Capability、NativeGameModule禁止API。
- Cross-fieldの複雑な算術制約。

JSON Schema合格だけを「Engine valid」と表示しない。`structural_valid`、`semantic_valid`、`policy_valid`、`budget_valid`を別結果にする。

## 16. Provider projection

### 16.1 共通原則

Provider SchemaはModelが正しい候補を作りやすくするためのInterfaceであり、Security validatorではない。全Tool callとStructured OutputをGatewayでInternal Schemaとsemantic validatorへ再投入する。Projection generatorは正規`provider_profile`に存在しないProvider capabilityを推測しない。

Provider projectionは次のArtifactを出す。

- Providerへ送るSchema。
- Canonical fieldからProvider fieldへのmapping。
- Optional／nullable／integer等の変換Rule。
- Provider Schemaで表現できなかったConstraint一覧。
- Gatewayで強制するvalidator ID。
- Projection conformance fixture。
- Schema hashとProvider Manifest互換範囲。

Constraintを黙って削除しない。未表現Constraintがある場合でもGateway validatorが存在すればProjectionを作れるが、`server_enforced_constraints`へRequirement IDとValidator IDを必ず列挙する。Safety／permission ConstraintにValidatorがなければ生成を失敗させる。

### 16.2 MCP projection

MCP projectionは`inputSchema`と`outputSchema`へToolchain lockが指定するJSON Schema dialectを明示する。Tool resultは`structuredContent`と互換用の同値JSON textを返す。ServerはOutputを送信前に検証し、Clientが検証しない場合でも安全性が変わらないようにする。

Tool annotationは非信頼表示Hintとして生成する。Access control、Risk、ApprovalはServer Policyで強制する。Tool名は`mirakan.<domain>.<verb>`、ASCII、128文字以下、Repository全体で一意とする。

### 16.3 OpenAI strict projection

OpenAI function／Structured Output projectionは次を必須にする。

- `strict: true`を明示する。
- 全objectに`additionalProperties: false`。
- `properties`の全Fieldを`required`へ列挙する。
- MCD optionalはnullableまたは明示presence wrapperへ変換する。
- OpenAIが対応しないJSON Schema keywordを出力しない。
- Provider requestの`strict: true`がSDK、Proxy、Providerで欠落または拒否された場合は送信または受信を失敗させる。
- Refusal、incomplete、tool argument parse errorをProposalと区別する。

Schema cacheとData retentionの制約がProject Policyに合わない場合、該当Projectionを使用しない。Full JSON Schema meta-schemaをOpenAI strict outputへ直接渡すことを禁止する。

### 16.4 Anthropic projection

Anthropic projectionはTool `name`、詳細な`description`、`input_schema`、対応Provider versionで利用可能な`strict`等をProvider Manifestから生成する。Providerが受理するJSON Schema keyword集合をconformance testで検出し、未検証keywordを使用しない。Anthropic APIのexact pinは[Toolchain／Dependencies](toolchain-dependencies.md)が所有し、pinが未固定の間は本projectionを生成対象外とする。その間のAnthropic系接続は16.5節のMCP経路だけを使う。

複雑なToolにはMCDのvalid fixtureから少数の`input_examples`を生成できる。ただし、ExampleはPrompt tokenを消費するため、EvalでTool選択またはargument精度を改善した場合だけProvider Manifestへ有効化する。

### 16.5 CLI／Desktop App

Codex／Claude等のCLIと対応Desktop Hostは原則MCP projectionを使う。ChatGPT Chat／Workのremote MCP appと、ChatGPT desktop app内のCodex hostが使うlocal MCP設定は別surfaceであり、相互互換とみなさない。Provider固有PluginがMCDを独自変換してはならず、Miraikanai MCP ServerのTool一覧とSchemaを取得する。Source直接編集はMCP Tool権限とは別に、AI実装・保守ガバナンス規約のSource Worker sandboxを適用する。

Tool projectionは[AI Security／Approval](../01-governance/ai-security-approval.md#83-callerproviderdeploymentmodel-profile)のexecution route別closed tupleに有効なConformance Receiptがある場合だけ生成する。`standard_external_mcp`は`{external Host profile, MCP Transport profile, tool projection, proposal-only Authority profile, MCP Grant, Host/Transport Conformance}`を必須にし、Provider Runtime／Manifest、Inference Deployment、Model Snapshotをnull／unattestedにする。`managed_external_host`は`{external Host, MCP Transport, Provider Runtime, Model Snapshot, tool projection, Authority, Host/Transport Conformance, Provider-Tool Conformance, fresh Host Session Attestation}`を必須にする。`engine_provider_adapter`は`{first-party Engine Host, Provider Runtime, Provider Manifest, Inference Deployment, Model Snapshot, tool projection, Authority, Provider-Tool Conformance}`を必須にし、MCP Transport／Grantをnullにする。cloud direct APIとfirst-party local IPCはEngine routeのDeployment branchで区別し、MCP Transportを捏造しない。Host名またはModel family名によるEngine分岐を作らず、routeに必要なReceipt／Grant／Attestationがない組合せは、別途current proposal-only Contextを再発行できなければ`not_activated`とする。ChatGPT Chat／Workのcustom MCP appはremote MCP（private endpointはSecure MCP Tunnel）であり、direct local STDIO Hostとして扱わない。ChatGPT desktop app内Codex host、Codex CLI／IDE、Claude Desktop／Claude Code、Cursorも、製品名だけで対応済みとせず、exact surface／Host version、Transport、Schema、Result、cancellationのConformance Receiptへ束縛する。

## 17. Language／Runtime projection

### 17.1 C++23

生成C++は次に従う。

- Namespaceは`mirakan::contract::<domain>`。
- Public wire structはstandard-layoutを要求せず、Field access APIを使う。
- Owning string／vectorまたは明示viewを型で分ける。
- Deserializerはunknown field、duplicate field、range超過、invalid UTF-8を拒否する。
- `std::expected`を基礎にしたEngine `Result<T>`を返し、`Error`／`MirakanDiagnostic`へtyped conversionする。Exceptionをwire境界から出さない。
- Enumはclosed enumとし、unknown値をdefaultへ変換しない。
- HandleはID＋generationでありpointerを含めない。
- CX0のGenerated Header、CX1以降のGenerated Module interface、永続C ABI HeaderにInput contract hashを記録する。
- Generated Sourceのimport／includeは`CppDependencySetV1`からだけ生成し、手書き依存と二重管理しない。
- CMake Preset／Android Gradle CMake設定のDriver、Generator、Configuration写像は`BuildDriverProfileV1`検査表と一致し、自由文字列のGeneratorを生成しない。

### 17.2 TypeScript

- `strict`、`noUncheckedIndexedAccess`、`exactOptionalPropertyTypes`を有効にする。
- Compile-time typeだけで信用せず、受信境界にgenerated runtime validatorを置く。
- `number`へ安全に入らない整数はdecimal string branded typeにする。
- `unknown`からvalidatorを通して型を得る。`as`による無検証castを生成しない。

### 17.3 GameplayDefinition／Cooked binary

- `runtime_cook = true`のMCD Typeだけにbinary descriptor、encoder、decoderを生成する。
- Field ID、wire type、offset、length、alignment、collection上限、Capability ID、State IDをContract setから生成する。
- Pointer、vtable、native padding、source path、filesystem、network、native library情報をencodeしない。
- GameplayDefinitionのWorld変更はtyped commandへ投影し、Runtime規約のconsume phaseで適用する。
- State Machineは一phase一transition、Behavior Treeは`max_node_visits_per_tick`必須、collectionは固定上限という規約をgenerated validatorへ含める。
- Binary headerへContract set、Definition set、Capability manifest、State layoutのhashを記録する。
- Decode errorはMirakanDiagnosticへ変換し、部分objectをRuntimeへ公開しない。

## 18. Round-tripとCross-language同値性

各TypeはC++／TypeScriptについて次のTestを自動生成し、`runtime_cook = true`のTypeだけCooked binaryを同じfixtureへ追加する。

1. MCD valid fixtureをC++、TypeScriptでdecodeし、対象TypeはCooked binaryでもdecodeする。
2. decode値を再encodeする。
3. Canonical JSON値が同値であることを確認する。
4. C++ encodeをTypeScript decode、TypeScript encodeをC++ decodeし、`runtime_cook = true`ではCooked binaryとのcanonical value同値も検査する。
5. Field順、unknown field、境界値、Unicode、empty、max length、invalid enumを検査する。
6. Provider projectionで生成したvalid sampleをInternal validatorへ通す。
7. Invalid sampleが全Languageで同じDiagnostic code familyになることを確認する。

Binary Runtime formatはMCDとJSON projectionを正本に維持し、binary codecのround-tripを同じfixtureへ含める。Binary formatからMCDを逆生成しない。

## 19. Schema変更とMigration

### 19.1 変更分類

| 変更 | Contract version | Risk |
|---|---:|---:|
| description、exampleだけ | 同version可 | R1 |
| optional field追加 | Type version増加 | R3 |
| required field、constraint、enum変更 | Type version増加＋Migration | R3／R4 |
| Operation authority／Risk／side effect変更 | Operation version増加 | R4 |
| State transition変更 | Machine version増加＋形式検証 | R4 |
| Field削除／意味変更 | 新version＋一方向Migration | R4 |

Pre-1.0ではRuntimeとEditorに旧version compatibility branchを残さない。永続Projectは`tools/project_migrator`がbackup、旧Schema検証、段階変換、Diff、現在Schema検証、atomic切替を行う。変換不能Fieldを黙って捨てない。

Field名を変更しても`field_id`を再利用して意味を変えない。削除済み`field_id`は予約し、別Fieldへ再利用しない。

### 19.2 Contract ChangeSet

Schema変更は次を同一ChangeSetへ含める。

- MCD source。
- RequirementとADR。
- Semantic validator。
- Migrationまたは「永続Data影響なし」の証拠。
- Generated output差分。
- valid／invalid／round-trip fixture。
- Provider conformance結果。
- 影響するGame sample、Editor、GameplayDefinition、C++ callerの更新。

## 20. AI向けDiscovery／Execution候補のplanning record（未Activation）

Activation後の受入条件として、AIへ巨大な全Schemaを一括送信せず、次の二段階planned Discovery semanticsを使う。

1. Capability検索段階がID、title、tag、Target、maturity、短いsummaryを返す。
2. Capability詳細読取段階が選択したCapabilityのType、Operation、Constraint、Budget、Exampleを返す。

Activation後のSearch結果はその時点のContract set hashを含む。AIが古いhashのCapabilityでProposalを送った場合、Gatewayはstaleとして拒否し、差分を返す。AIがSchemaにないFieldや完全登録済みでないOperationを使った場合、fuzzyに推測して補正せず、候補ID付きDiagnosticを返す。planned検索actionのbounded result受入値は、各表で別値を明示しない限り既定50件、最大200件、continuation付きとする。Activation前のcurrent Search／Read Operation集合は空であり、この挙動をcallableとみなさない。

以下の§§20～21に現れる159個の`operation.*`文字列は実行契約ではなく、将来の語彙衝突を防ぐ予約候補である。最初の八family 67件はDiscovery／Execution候補、後続十family 92件は既存Domain文書から回収した未登録authoring／selection候補である。MCD document、Owner Manifest、Contract set member、Trusted Service allowlist、Provider projection、MCP alias、CLI／Editor commandのいずれにも存在せず、全familyのcurrent集合は空、Capability stateは`not_activated`である。別途§8.1／§8.2で完全登録したcurrent active 14 Operationは維持し、このplanning ledgerへ移動しない。

```text
PlannedOperationFamilyV1
  planning_record_id
  planning_record_version: 1
  family_id
  reserved_candidate_ids[1..64]
  reserved_candidate_count: exact array count
  capability_state: not_activated
  current_owner_manifest_operation_local_refs: []
  current_mcd_operation_local_refs: []
  current_trusted_service_allowlist_local_refs: []
  current_policy_local_refs: []
  current_validator_local_refs: []
  current_validator_closure_local_refs: []
  current_diagnostic_local_refs: []
  current_receipt_type_local_refs: []
  current_provider_projection_refs: []
  current_mcp_tool_refs: []
  generated_aliases: []
  legacy_aliases: []
  activation_work_item_id
  activation_mode: atomic_family_contract_set_transaction
  unavailable_error_code: MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED
  planning_record_hash: SHA-256
```

`PlannedOperationFamilyV1`はMCD kindではなく、`ContractSetSnapshotV2`、Tool catalog、Package、Save、Replayへ入れないclosed planning recordである。hashはASCII `MIRAKAN_PLANNED_OPERATION_FAMILY_V1`と自己hashを除く全Fieldのlength-framed canonical bytesから計算する。候補要求はGateway dispatch前に上記errorで拒否し、Project／Source／Registry／Taskを変更しない。familyの一部だけ、name-only record、aliasだけ、read-only候補だけを先行activateしない。

| planning_record_id | version | family_id | exact候補数 | atomic activation work item |
|---|---:|---|---:|---|
| `planning.operation_family.authoring_discovery` | 1 | `authoring_discovery` | 4 | `activation.authoring.discovery_operations.v1` |
| `planning.operation_family.build_device_play_debug_task` | 1 | `build_device_play_debug_task` | 14 | `activation.build_gateway.operation_pipeline.v1` |
| `planning.operation_family.game_system_discovery` | 1 | `game_system_discovery` | 4 | `activation.systems.discovery_operations.v1` |
| `planning.operation_family.world_discovery` | 1 | `world_discovery` | 6 | `activation.worlds.discovery_operations.v1` |
| `planning.operation_family.rendering_aa_discovery` | 1 | `rendering_aa_discovery` | 5 | `activation.rendering.aa.discovery_operations.v1` |
| `planning.operation_family.lighting_discovery` | 1 | `lighting_discovery` | 9 | `activation.lighting.discovery_operations.v1` |
| `planning.operation_family.post_process_discovery` | 1 | `post_process_discovery` | 9 | `activation.post_process.discovery_operations.v1` |
| `planning.operation_family.project_shader_discovery` | 1 | `project_shader_discovery` | 16 | `activation.shader.discovery_proposal_operations.v1` |
| `planning.operation_family.math_semantic_authoring` | 1 | `math_semantic_authoring` | 6 | `activation.math.semantic_authoring_operations.v1` |
| `planning.operation_family.camera_authoring` | 1 | `camera_authoring` | 11 | `activation.camera.authoring_operations.v1` |
| `planning.operation_family.material_authoring` | 1 | `material_authoring` | 15 | `activation.material.authoring_operations.v1` |
| `planning.operation_family.vfx_authoring` | 1 | `vfx_authoring` | 24 | `activation.vfx.authoring_operations.v1` |
| `planning.operation_family.environment_authoring` | 1 | `environment_authoring` | 24 | `activation.environment.authoring_operations.v1` |
| `planning.operation_family.lod_authoring` | 1 | `lod_authoring` | 2 | `activation.lod.authoring_operations.v1` |
| `planning.operation_family.input_binding_selection` | 1 | `input_binding_selection` | 1 | `activation.input.semantic_action_binding_selection.v1` |
| `planning.operation_family.navigation_binding_selection` | 1 | `navigation_binding_selection` | 1 | `activation.navigation.motion_intent_binding_selection.v1` |
| `planning.operation_family.physics_role_selection` | 1 | `physics_role_selection` | 1 | `activation.physics.intent_role_selection.v1` |
| `planning.operation_family.feature_authoring` | 1 | `feature_authoring` | 7 | `activation.feature.authoring_operations.v1` |

上表18行と、この後に文書順で現れる第一列見出しが`reserved candidate ID`の18表を同じindexでzipするclosed expansion ruleを正本とする。各`PlannedOperationFamilyV1` instanceは、ledger列から`planning_record_id`、`planning_record_version`、`family_id`、`reserved_candidate_count`、`activation_work_item_id`を、対応候補表の第一列から行順を保った`reserved_candidate_ids[]`をmaterializeする。残る全Fieldは上のschemaに書かれたliteral、すなわち`capability_state=not_activated`、十のcurrent集合と二つのalias集合すべて`[]`、`activation_mode=atomic_family_contract_set_transaction`、`unavailable_error_code=MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`をField省略なしでmaterializeし、その後にだけ`planning_record_hash`を計算する。これは既定値補完ではない。ledger／候補表が18対18でない、候補数不一致、候補ID重複、別familyへの同一ID混入、literal Fieldの省略／変更、empty集合の省略をplanning record compile failureにする。

各work itemは、そのfamilyの採用exact ID集合、各OperationのMCD共通Envelope全Field、named input／result／Receipt Type、authority Serviceとallowlist、Risk、side effect、idempotency、transaction、pure pre／post Policy、rate／timeout、closed Diagnostic、Validator／closure、Provider exposure、canonical signed Receipt、private-to-public recovery、positive／negative Qualification、Owner Manifest／MCD／Service／Provider／aliasのlike-for-like equalityを同じContract set transactionで完備した場合だけfamily全体をactivateできる。候補IDを削除または分割する場合もplanning recordをversion-upし、実在しないIDをlegacy aliasにしない。以下の表の「予定意味」はactivation後に採用可否を再審査する入力であり、現行動作を記述しない。

Authoring dataの同じDiscovery原則について、次の四つをplanning candidateとして予約する。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.authoring.search` | kind、tag、Component、name token、spatial boundからStableId候補とscore理由を返す |
| `operation.authoring.read` | StableId、field mask、expected revisionからbounded `SceneSliceV1`またはDocument projectionを返す |
| `operation.authoring.dependencies` | inbound／outbound、Requirement、Capability、Decision、lockのbounded closureを返す |
| `operation.authoring.diff` | base／target revisionとStableId scopeからsemantic diff、storage-only diff、continuationを返す |

activation work itemがこのexact四件を採用する場合、全Queryは`project_revision`、`contract_set_hash`、`authoring_index_revision`、`query_hash`、`omitted_ranges`、`continuation_cursor`を返す契約にし、別revisionへのfallback、表示index、曖昧な名前だけのtarget確定、任意JSON断片を禁止する。それまでは四IDのdispatch、追加read、aliasのmaterializeを行わない。

Package／Device／Play／Debug／Task系はBuild Gatewayを入口候補とする次の14 IDをplanning recordへ予約する。これらはcanonical MCDでも公開Operationでもなく、[Core architecture](core-architecture.md#91-operationtaskv1)のplanned task／Receipt mappingはActivation後の意味候補に留まる。

| reserved candidate ID | 予定Risk／kind／実行Authority | 予定必須identity／hash | 予定Side effect | 予定Idempotency／cancel | 予定成功Receipt／結果 | 予定Failure Diagnostic |
|---|---|---|---|---|---|---|
| `operation.build.request_package` | R2 job／Build Gateway | Project revision、Candidate root、Target Profile、Contract／Toolchain lock、request hash、Authorization | StagingにTarget packageを生成。Commit／sign／installなし | request hash＋idempotency key。publish前までcooperative cancel | `PackageReceiptV1` | `diagnostic.operation.package-input-mismatch` |
| `operation.device.install` | R3 command／Device Bridge | Project revision、Candidate root、Target、Device identity＋generation、exact `PackageReceiptV1` ref／hash＋package artifact ref／hash、request hash、Authorization、明示consent、R3 Approval | 承認済みpackageを一Deviceへinstall | package hash＋Device generation＋idempotency key。Device transaction commit前だけcancel | `DeviceInstallReceiptV1` | `diagnostic.operation.device-install-binding-mismatch` |
| `operation.device.launch` | R1 command／Device Bridge | Project revision、Candidate root、Target、Device identity＋generation、exact `DeviceInstallReceiptV1` ref／hash＋package artifact ref／hash、request hash、固有Authorization | install済みCandidateのprocess起動 | launch request hash＋idempotency key。process spawn前だけcancel | `DeviceLaunchReceiptV1` | `diagnostic.operation.device-launch-binding-mismatch` |
| `operation.device.reset_data` | R3 command／Device Bridge | Project revision、Candidate root、Target、Device identity＋generation、exact `PackageReceiptV1` ref／hash＋package artifact ref／hash、request hash、Authorization、明示consent、R3 Approval | 対象ApplicationのDevice dataを消去 | package hash＋Device generation＋idempotency key。reset commit前だけcancel | `DeviceDataResetReceiptV1` | `diagnostic.operation.device-reset-consent-required` |
| `operation.play.run_smoke` | R1 job／Play Service | Project revision、Candidate root、Target、Device identity＋generation、exact Package／Install／Launch Receipt ref／hash、package artifact ref／hash、fixture ref／hash、request hash、固有Authorization | Candidateに対するbounded smoke sessionを実行 | input hash＋idempotency key。fixture boundaryでcooperative cancel | `SmokeRunReceiptV1` | `diagnostic.operation.smoke-input-mismatch` |
| `operation.debug.aggregate` | R0 query／Debug Query Service | Project revision、Candidate root、Target、Session、exact Build Receipt ref／hash、Store／Index generation、bounded selector hash、request hash、Authorization、remote Device identity＋generation | read-only aggregateを計算 | pure。cancelはquery中断のみ | `DebugAggregateReceiptV1` | `diagnostic.debug.aggregate-input-invalid` |
| `operation.debug.query` | R0 query／Debug Query Service | Project revision、Candidate root、Target、Session、Store／Index generation、remote Device identity＋generation、exact `DebugAggregateReceiptV1` ref／hash、bounded query hash、request hash、新Authorization | read-only record sliceを返す | pure。cancelはquery中断のみ | `DebugQueryReceiptV1` | `diagnostic.debug.query-input-invalid` |
| `operation.debug.read_causality` | R0 query／Debug Query Service | Project revision、Candidate root、Target、Session、Index generation、remote Device identity＋generation、exact `DebugQueryReceiptV1` ref／hash、root Evidence refs、bound hash、request hash、新Authorization | read-only causal subgraphを返す | pure。cancelはquery中断のみ | `DebugCausalityReceiptV1` | `diagnostic.debug.causality-input-invalid` |
| `operation.debug.read_replay_slice` | R0 query／Replay Service | Project revision、Candidate root、Target、Session、remote Device identity＋generation、exact Build Receipt、`DebugQueryReceiptV1`、`DebugCausalityReceiptV1`の各ref／hash、Replay closure／range hash、request hash、新Authorization | immutable Replay Sliceをmaterialize | request hash＋idempotency key。publish前までcooperative cancel | `ReplaySliceReceiptV1` | `diagnostic.debug.replay-slice-input-invalid` |
| `operation.debug.validate_finding` | R0 job／Debug Validation Service | Project revision、Candidate root、Target、Session、remote Device identity＋generation、exact Build Receipt、`DebugQueryReceiptV1`、`DebugCausalityReceiptV1`、`ReplaySliceReceiptV1`の各ref／hash、`DebugFindingV1` hash、Finding closure hash、request hash、新Authorization | append-only validation Evidenceを生成。Source／Project変更なし | finding＋closure hashでidempotent。validation step間でcancel | `DebugFindingValidationReceiptV1` | `diagnostic.debug.finding-evidence-invalid` |
| `operation.debug.support-bundle.generate` | R2 job／Debug Export Service | Project revision、Candidate root、Target、Session、exact Build Receipt ref／hash、source Debug Receipt ref／hash集合、component／redaction／policy hash、request hash、明示consent、Authorization、remote Device identity＋generation | redacted Support BundleをStagingへ生成。uploadなし | manifest input hash＋idempotency key。archive publish前までcooperative cancel | `SupportBundleReceiptV1` | `diagnostic.debug.support-bundle-manifest-mismatch` |
| `operation.task.status` | R0 query／Build Gateway Task Service | control invocation ID、target task／operation ID、target request hash、Project revision、Candidate root、control request hash、Authorization | なし。bounded task snapshotを返す | pure。cancel対象外 | `TaskStatusReceiptV1`＋`OperationTaskV1` snapshot | `diagnostic.operation.task-binding-mismatch` |
| `operation.task.read_receipt` | R0 query／Build Gateway Task Service | control invocation ID、terminal task／operation ID、target request hash、Project revision、Candidate root、exact terminal Receipt ref／hash、control request hash、Authorization | なし。immutable Receiptをread-only取得 | pure。cancel対象外 | `TaskReceiptReadReceiptV1`＋referenced exact Receipt | `diagnostic.operation.task-receipt-mismatch` |
| `operation.task.cancel` | R1 command／Build Gateway Task Service | control invocation ID、target task／operation ID、target request hash、Project revision、Candidate root、cancel request hash、original callerまたは委任済みAuthorization | cancellable taskを`cancel_requested`へ遷移 | task ID＋idempotency key。反復cancelは同じ結果 | `TaskCancellationReceiptV1` | `diagnostic.operation.task-not-cancellable` |

activation work itemがexact 14件を採用する場合、各成功Receiptは[Core architecture §9.1](core-architecture.md#91-operationtaskv1)のplanned 14行mappingからexact `signed_record_purpose`、`OperationReceiptEnvelopeV1.operation_id`、型固有payload contract、完成Receipt型を一行で選ぶ完成`MirakanSignedRecordV1`とする。前段Receipt refとhash、purpose、payload contract、Project／Candidate／Target／Device identity＋generation、request hash、Authorizationの一つでも異なれば後段を開始しない。install／resetのconsent／Approval非継承やDevice generation driftも同じactivation transactionのfixtureで閉じる。現在はReceipt、Task、allowlist、remote sessionを一件もmaterializeしない。

将来このfamilyをactivateしても、正規Commit、Approval発行、Promotion、Activation、Signing、Releaseは8節の`trusted_internal`境界を代替しない。AIの実行権限とCaller Profileは[AI Security／Approval](../01-governance/ai-security-approval.md)が所有する。

LOD Discoveryは`lod_class`、semantic role、Target、Qualityで絞り込み、Intent、該当Domain Policy、fallback、選択metric、現在のqualification statusだけを返す。全DomainのLOD Schemaやruntime telemetryを常に一括送信しない。

Game System Discoveryは次の四IDをplanning candidateとして予約する。current MCP／製品Tool名、generated alias、legacy aliasの集合は空である。

| reserved candidate ID | 予定Authority | 予定意味 |
|---|---|---|
| `operation.systems.search` | R0 query | Role、Target、maturity、originでCatalog entryを検索 |
| `operation.systems.read` | R0 query | exact System Contract、constraint、budget、fixtureを取得 |
| `operation.systems.plan` | R1 proposal | `SystemImplementationPlanV1`を提案 |
| `operation.systems.validate_bundle` | R0 query／job | Staging `SystemBundleChangeSetV1`を検証 |

World Discoveryも次の六IDをplanning candidateとして予約する。alias、Input／Output Schema、Provider projectionはcurrent集合に存在しない。

| reserved candidate ID | 予定Authority | 予定意味 |
|---|---|---|
| `operation.worlds.search` | R0 query | kind、role、tag、Target、spatial boundからWorld／Scene／Space／Topology／owner-typed候補、StableId、score理由を返す |
| `operation.worlds.read` | R0 query | exact Stable ref、field mask、Viewport、Targetからbounded `WorldAuthoringContextV1`を返す |
| `operation.worlds.resolve_map_intent` | R0 query／R1 proposal | 6分類候補、Evidence、`resolved \| question_required \| rejected`を返す |
| `operation.worlds.plan_change` | R1 proposal | allowed `WorldSourceChangePrimitiveKindV1` discriminator、precondition、Budget、fixtureを持つ`WorldAuthoringPlanV1`を返す |
| `operation.worlds.validate_bundle` | R0 query／job | Staging BundleのSchema、semantic、reference、ownership、Topology、Budget Diagnosticを返す |
| `operation.worlds.preview_bundle` | R0 query／job | Source／Topology／Space／owner-typed content／Target別Derived差分、activation、performance、fallback比較を返す |

activation work itemが`operation.worlds.read`を採用する場合はAuthoring規約の`AuthoringSelectionContextV1` hashを任意入力として受けられるが、screen coordinate、表示row、Hierarchy path、表示名だけをtargetへ変換しない。出力はProject revision、Contract set hash、Source Document hash、Source／Staging／Derived read-only／Runtime区分、omitted range、continuationを必須とする。Derived Artifactはread-only refだけを返し、Cell、Navmesh、HLOD、Runtime handleのwrite primitive schemaを生成しない。

activation後に`operation.worlds.plan_change`を採用する場合、返すchange primitive discriminatorはWorld規約の`WorldSourceChangePrimitiveKindV1` Catalogに存在し、`ai_mutable` field coverage、expected Document revision、precondition hash、Risk、Approval、inverse availabilityを満たすものだけにする。behavior-neutralな予定語彙は`CreateWorld | UpdateWorldComposition | CreateScene | UpdateSceneComposition | DefineSpace | UpdateTopology | BindOwnerTypedDocument`へ限定し、Scene永続化owner、World composition membership、Space／Topology relation、Derived Cell assignmentを別constraintとしてProvider Schemaとserver-enforced semantic validatorへ投影する。これらは現在公開されていない。`Level`、`SetLevelSourceScenes`、`playable_level`はcurrent Core MCD Operation／primitive kind／constraintではなく、Pack-owned migration inputにだけ存在できる。

World CapabilityのMCD `examples`は最低でも、明確な`world_structure`、曖昧で`question_required`、共有Scene変更の影響World列挙、Derived Cell直接write拒否、stale revision拒否を各1件含む。Exampleは説明文だけでなく、exact Input、expected disposition／Activation後のexact MCD Operation IDまたはchange primitive discriminator、expected Diagnostic code、変更されないinvariantを持つ。Pack-owned Stage等のcontent概念をCore World kindまたは暗黙空間前提として登録しない。

`MapIntentResolutionV1.disposition=question_required`をCommit可能Proposalへ自動変換しない。System／WorldのActivation、Source Promotion、Project Commit OperationはProvider projectionへ含めない。

Anti-alias Discoveryは2D／3D機能計画の意味GoalとRenderer規約の実行制約について次の五IDをplanning candidateとして予約する。current MCP aliasやProvider Toolをmaterializeしない。

| reserved candidate ID | 予定Risk／kind | 予定意味 |
|---|---|---|
| `operation.rendering.aa.search` | R0 query | 意味Goal、Target、Renderer、Quality、maturityからIntent／method候補を検索し、候補ID、短い適合理由、制約要約を返す |
| `operation.rendering.aa.read` | R0 query | exact Intent／method／Profileの互換Predicate、sample count、layer scope、cost model、fallback、Diagnostic、必要Qualificationを返す |
| `operation.rendering.aa.resolve_intent` | R0 query | 永続変更なしでIntentをViewFamily単位Planへ決定的に解決し、採用候補、却下候補と理由、`resolved`／`question_required`／`unsupported`を返す |
| `operation.rendering.aa.plan_change` | R1 proposal | expected Project revisionに対するtyped `AntiAliasingChangeSetProposalV1`だけを生成する。Commit、Provider activation、Pipeline rebuildは行わない |
| `operation.rendering.aa.preview_change` | R0 query／job | ProposalをStagingで検証し、resolved Plan差分、Graph／history影響、GPU／memory／bandwidth見積り、visual fixture要求、fallback、必要Receiptを返す |

activation work itemが五件を採用する場合、全Anti-alias Query／Proposal結果は`project_revision`、`contract_set_hash`、`capability_signature_hash`、`renderer_profile_revision`、`qualification_receipt_hashes`、`query_hash`を返し、ViewFamilyへ解決した結果は`view_family_id`と`source_intent_revision`も返す。bounded collection、revision、互換性、typed Diagnosticの規則も同じtransactionで登録する。

`AntiAliasingIntentV1`のclosed enum／tagged union規則はOperation Activationと独立したDomain契約として維持する。将来のProvider／MCP投影を採用する場合も候補五件以外を混ぜず、`ResolvedAntiAliasingPlanV1` write、arbitrary Render Graph write、Provider install／activate、Settings Apply、Source Promotion、Project CommitをTool listへ含めない。

Lighting DiscoveryはLighting規約の意味Role、物理単位、Target／Budget制約について次の九IDをplanning candidateとして予約する。current MCP aliasをmaterializeしない。

| reserved candidate ID | 予定Risk／kind | 予定意味 |
|---|---|---|
| `operation.lighting.search` | R0 query | Light／Profile／role／Target／maturityを検索 |
| `operation.lighting.read` | R0 query | field mask付きSource／Intent／Profile／Planを取得 |
| `operation.lighting.inspect` | R0 query | bounded Scene summary、lock、上限、cost、Diagnosticを取得 |
| `operation.lighting.resolve_intent` | R0 query | 永続変更なしで`ResolvedLightPlanV1`を決定的に生成 |
| `operation.lighting.plan_change` | R1 proposal | expected revisionに対する`LightingChangeSetProposalV1`を生成 |
| `operation.lighting.preview_change` | R0 query／job | before／after、contribution、cluster、Shadow、costを検証 |
| `operation.lighting.explain_plan` | R0 query | Intent→Source field、採用／棄却／fallback理由を取得 |
| `operation.lighting.estimate_cost` | R0 query | Target別CPU／GPU／Memory予測とconfidenceを取得 |
| `operation.lighting.validate_change` | R0 query／job | Schema／semantic／Capability／Budget／lockを検証 |

activation後に九件を採用する場合、結果は`project_revision`、`contract_set_hash`、`lighting_catalog_hash`、`target_capability_hash`、`query_hash`を必須とし、Plan／Previewは追加closure hashを返す。LightのCommit、native GPU resource／cluster buffer書込み、Shadow Technique Source追加、Provider activation、Project HLSL変更をTool listへ含めない。新規TechniqueへのhandoffもProject Shader familyが別途activateされた場合だけ可能である。

Post Process DiscoveryはIntent、Profile、Node Catalog、Volume、AA／Layer互換について次の九IDをplanning candidateとして予約する。current MCP aliasをmaterializeしない。

| reserved candidate ID | 予定Risk／kind | 予定意味 |
|---|---|---|
| `operation.post_process.search` | R0 query | Profile／Volume／Node／Target／maturityを検索 |
| `operation.post_process.read` | R0 query | field mask付きIntent／Profile／Volume／Planを取得 |
| `operation.post_process.inspect` | R0 query | View、active stage、history、layer、cost、Diagnosticを取得 |
| `operation.post_process.resolve_intent` | R0 query | 永続変更なしで`ResolvedPostProcessPlanV1`を決定的に生成 |
| `operation.post_process.plan_change` | R1 proposal | expected revisionに対する`PostProcessChangeSetProposalV1`を生成 |
| `operation.post_process.preview_change` | R0 query／job | before／after、色空間、layer、history、costを検証 |
| `operation.post_process.explain_plan` | R0 query | Intent→Node／parameter、採用／棄却／fallback理由を取得 |
| `operation.post_process.estimate_cost` | R0 query | Target別CPU／GPU／Memory予測とconfidenceを取得 |
| `operation.post_process.validate_change` | R0 query／job | Schema／stage／AA／Layer／Capability／Budgetを検証 |

activation後に九件を採用する場合、結果は`project_revision`、`contract_set_hash`、`post_node_catalog_hash`、`target_capability_hash`、`anti_aliasing_plan_hash`、`query_hash`を必須とし、Plan／Previewは追加closure hashを返す。raw Render pass、native resource、history weight、Node stage並替え、Provider activation、Project Shader Source／Technique mutation、Source Promotion、Project CommitはTool listへ含めない。Project Shaderへのhandoffは同familyのActivationを前提とする。

Lighting／Post Process familyを将来activateする場合、Searchは既定50件、最大200件、continuation付き、Read／Inspectはfield maskとbounded scope、`plan_change`は`expected_project_revision`と`idempotency_key`を必須にし、Provider OutputをInternal validatorで完全再検証する。

## 21. Project Shader Discovery／Proposal候補のplanning record（未Activation）

Project Shaderは[Project Shader](../06-rendering/project-shader.md)の`PublicShaderSdkCatalogV1`、Module、Technique、Fact、Understanding Closureについて次の16 IDをplanning candidateとして予約する。current MCP aliasをmaterializeしない。

| reserved candidate ID | 予定Risk／kind | 予定意味 |
|---|---|---|
| `operation.shader.search` | R0 query | Project／public SDK origin、semantic role、module kind、Stage、Technique Port、Target、Capability、qualificationからStable ID候補を検索 |
| `operation.shader.read_module` | R0 query | exact Module manifest、Source file index／bounded excerpt、public export、typed value／resource interface、Target、fallbackを取得 |
| `operation.shader.read_technique` | R0 query | exact Technique、Pass DAG、logical resource、Port、Target、fallbackを取得 |
| `operation.shader.inspect_symbol` | R0 query | Projectまたはpublic SDKのStable Symbol IDからdeclaration、semantic、type、source span、entry／export、Fact、diagnosticをbounded取得 |
| `operation.shader.find_callers` | R0 query | caller／callee、Pass／Entry到達性、Target／variant scopeを取得 |
| `operation.shader.explain_dataflow` | R0 query | value flow、unit／space／color変換、output寄与とEvidenceを取得 |
| `operation.shader.explain_resource_effects` | R0 query | resource access、side effect、Pass／queue intent、lifetime影響を取得 |
| `operation.shader.compare_targets` | R0 query | interface、Capability、precision、variant、fallback、Diagnostic差を取得 |
| `operation.shader.preview` | R0 query／job | exact Source／artifact／Targetでvisual／analytic fixtureを実行 |
| `operation.shader.parameter_sweep` | R0 query／job | bounded parameter setのcounterfactual、image／invariant／cost差を取得 |
| `operation.shader.estimate_cost` | R0 query | Target別instruction、resource、variant、GPU／memory予測と根拠を取得 |
| `operation.shader.validate_contract` | R0 query／job | Profile、semantic、Fact、reflection、Target、fixture、UnderstandingのDiagnosticを取得 |
| `operation.shader.plan_module` | R1 proposal | Sourceを変更しない`ProjectShaderModulePlanV1`を生成 |
| `operation.shader.plan_technique` | R1 proposal | Sourceを変更しない`ProjectShaderTechniquePlanV1`を生成 |
| `operation.shader.propose_module` | R3 source proposal | expected revisionに対するModule Source／contract ChangeSetをStagingへ生成 |
| `operation.shader.propose_technique` | R3 source proposal | expected revisionに対するTechnique／Module Source ChangeSetをStagingへ生成 |

activation work itemが16件を採用する場合、全結果は`project_revision`、`contract_set_hash`、`bounded_project_shader_profile_hash`、`source_tree_hash`、`query_hash`を必須とし、Fact／Target／Preview／Validate／Proposalごとのclosure hashをnamed Resultへ閉じる。Closure未生成を空hashで表さず、statusを`not_run | stale | failed | passed`として返す。

activation後のSearchは既定50件、最大200件、continuation付き、Read／Inspect／Explainはfield mask、Stable ID、Target／variant scope、最大node／edge／source byteを必須にする。上限超過時は`omitted_ranges`と`continuation_cursor`を返し、GraphまたはSourceを途中で正常完了扱いしない。

activation後のPlan／Proposeは`expected_project_revision`、`idempotency_key`、Profile hash、対象Module／Technique Stable ID、Target集合、Requirement、Budget、fixture、fallback、Riskを必須にする。ProposeはSource ChangeSetだけに限定し、compiler command、Project／Engine filesystem直接write、artifact publish、Commit、Activation、Approval、Policy変更を行わない。現在はPlan／Proposeを含む16件すべてを拒否する。

### 21.1 既存Domain文書から回収した未登録Operation候補

次の十表92件は、既存Domain文書が過去にcurrent／canonical／registeredとして記述していたname-only surfaceを、§20の`PlannedOperationFamilyV1`へ回収したclosed candidate集合である。各表は上のledger後半十行と同じ順で対応する。これらは予約語彙であってMCD Operationではなく、全current集合とalias集合は明示`[]`、Capability stateは`not_activated`である。要求はGateway dispatch前に`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`で拒否し、Source、Project、Registry、Taskを変更しない。

Math semantic authoringは次の六IDだけを予約する。Math文書にある`operation.camera.set_profile_projection`はCamera ownerの次表に属する同一候補への参照であり、Math familyへ複製しない。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.transform.set_world_position` | semantic positionの変更Proposal |
| `operation.transform.set_local_rotation` | semantic local rotationの変更Proposal |
| `operation.transform.set_local_scale` | semantic positive scaleの変更Proposal |
| `operation.physics.set_velocity` | Physics ownerへ渡すsemantic velocity変更Proposal |
| `operation.asset.set_pixels_per_unit` | Asset ownerへ渡すpixel density変更Proposal |
| `operation.ui.set_rect` | UI ownerへ渡すsemantic rectangle変更Proposal |

Camera authoringは次の11 IDだけを予約する。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.camera.resolve_intent` | Camera intentのread-only解決 |
| `operation.camera.create_profile` | Camera Profile作成Proposal |
| `operation.camera.set_profile_projection` | Projection semantic変更Proposal |
| `operation.camera.create_rig` | Rig作成Proposal |
| `operation.camera.add_rig_node` | typed Rig Node追加Proposal |
| `operation.camera.connect_rig_nodes` | typed Port接続Proposal |
| `operation.camera.set_director_rule` | Director rule変更Proposal |
| `operation.camera.set_presentation_profile` | Presentation Profile変更Proposal |
| `operation.camera.create_sequence` | Cinematic Sequence作成Proposal |
| `operation.camera.preview_candidate` | Staging candidateのPreview |
| `operation.camera.analyze_composition` | compositionのread-only解析 |

Material authoringは次の15 IDだけを予約する。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.material.search` | Material Catalog検索 |
| `operation.material.read` | bounded Material読取 |
| `operation.material.inspect` | Material closure検査 |
| `operation.material.preview` | Staging Material Preview |
| `operation.material.explain` | semantic resolution説明 |
| `operation.material.estimate` | Target別cost見積り |
| `operation.material.validate` | Material closure検証 |
| `operation.material.plan` | read-only変更Plan |
| `operation.material.create_instance` | Instance作成Proposal |
| `operation.material.assign_template` | Template割当Proposal |
| `operation.material.set_parameters` | typed parameter変更Proposal |
| `operation.material.create_definition` | Definition作成Proposal |
| `operation.material.edit_graph` | typed Graph変更Proposal |
| `operation.material.create_derived_style` | 派生Style作成Proposal |
| `operation.material.bind_surface_semantics` | Surface semantic binding Proposal |

VFX authoringは次の24 IDだけを予約する。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.vfx.inspect_system` | VFX System検査 |
| `operation.vfx.inspect_semantic_catalog` | semantic Catalog検査 |
| `operation.vfx.validate_changeset` | ChangeSet構造検証 |
| `operation.vfx.validate_semantic_preservation` | Cue semantic保存検証 |
| `operation.vfx.preview_changeset` | Staging Preview |
| `operation.vfx.resolve_effect_intent` | Effect intentのread-only解決 |
| `operation.vfx.set_effect_intent` | Effect intent変更Proposal |
| `operation.vfx.apply_pattern` | Pattern適用Proposal |
| `operation.vfx.create_system` | System作成Proposal |
| `operation.vfx.create_emitter` | Emitter作成Proposal |
| `operation.vfx.update_emitter` | Emitter更新Proposal |
| `operation.vfx.delete_emitter` | Emitter削除Proposal |
| `operation.vfx.add_node` | typed Node追加Proposal |
| `operation.vfx.update_node` | typed Node更新Proposal |
| `operation.vfx.delete_node` | typed Node削除Proposal |
| `operation.vfx.connect_nodes` | typed Port接続Proposal |
| `operation.vfx.disconnect_nodes` | typed Port切断Proposal |
| `operation.vfx.set_curve` | bounded Curve変更Proposal |
| `operation.vfx.set_gradient` | bounded Gradient変更Proposal |
| `operation.vfx.set_output` | Output変更Proposal |
| `operation.vfx.generate_fallback` | explicit fallback生成Proposal |
| `operation.vfx.capture_bounds` | bounded bounds capture |
| `operation.vfx.run_qualification` | Qualification実行Proposal |
| `operation.vfx.propose_extension_operator` | Extension operator Source Proposal |

Environment authoringは次の24 IDだけを予約する。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.environment.inspect_profile` | Environment Profile検査 |
| `operation.environment.list_presets` | bounded Preset列挙 |
| `operation.environment.resolve_intent` | Environment intentのread-only解決 |
| `operation.environment.validate_changeset` | ChangeSet検証 |
| `operation.environment.preview_changeset` | Staging Preview |
| `operation.environment.estimate_cost` | Target別cost見積り |
| `operation.environment.create_profile` | Profile作成Proposal |
| `operation.environment.apply_preset` | Preset適用Proposal |
| `operation.environment.set_intent` | Intent変更Proposal |
| `operation.environment.set_sky` | Sky変更Proposal |
| `operation.environment.set_sun_moon_link` | Sun／Moon relation変更Proposal |
| `operation.environment.set_height_distance_fog` | Height／distance fog変更Proposal |
| `operation.environment.set_volumetric_fog` | Volumetric fog変更Proposal |
| `operation.environment.create_local_fog_volume` | Local Fog Volume作成Proposal |
| `operation.environment.update_local_fog_volume` | Local Fog Volume更新Proposal |
| `operation.environment.delete_local_fog_volume` | Local Fog Volume削除Proposal |
| `operation.environment.set_atmosphere_preset` | Atmosphere Preset変更Proposal |
| `operation.environment.set_custom_atmosphere` | Custom Atmosphere変更Proposal |
| `operation.environment.set_cloud_layer` | Cloud Layer変更Proposal |
| `operation.environment.set_lighting` | Environment lighting変更Proposal |
| `operation.environment.bind_weather` | Weather binding Proposal |
| `operation.environment.generate_fallback` | explicit fallback生成Proposal |
| `operation.environment.bake` | offline bake Proposal |
| `operation.environment.run_qualification` | Qualification実行Proposal |

LOD authoringは次の二IDだけを予約する。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.lod.propose_policy` | read-only Policy Proposal |
| `operation.lod.apply_policy` | 承認済みPolicy ChangeSet Proposal |

Input selectionは次の一IDだけを予約する。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.input.semantic_action_binding.select` | Semantic Action Binding Selection作成／更新Proposal |

Navigation selectionは次の一IDだけを予約する。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.navigation.motion_intent_binding.select` | Motion Intent Binding Selection作成／更新Proposal |

Physics role selectionは次の一IDだけを予約する。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.physics.intent_role.select` | Physics Intent Role Selection作成／更新Proposal |

Feature authoringは次の七IDだけを予約する。

| reserved candidate ID | 予定意味 |
|---|---|
| `operation.feature.create_definition` | Feature Definition作成Proposal |
| `operation.feature.update_definition` | Feature Definition更新Proposal |
| `operation.feature.configure_system` | Feature System設定Proposal |
| `operation.feature.bind_runtime_port` | Runtime Port binding Proposal |
| `operation.feature.preview_change` | Feature変更Preview |
| `operation.feature.explain_contract` | Feature Contract説明 |
| `operation.feature.validate_contract` | Feature Contract検証 |

この92件についてもfamily単位のatomic activationだけを許可する。Math文書からCamera候補への一回のcross-owner参照を除き、159件全体の候補IDは重複なしである。`67 + 6 + 11 + 15 + 24 + 24 + 2 + 1 + 1 + 1 + 7 = 159`、planning family数は`8 + 10 = 18`であり、count、ID union、全empty current集合、work itemをcompiler fixtureでexact比較する。

### 21.2 Architecture内`operation.*` tokenのclosed partition

Contract lintは`docs/architecture/**/*.md`からまず完全なStable ID tokenを切り出し、complete tokenのkind prefixがexact `operation`であるものだけをOperation IDとして分類する。`policy.operation.*`、`validator.operation.*`、`validator_closure.operation.*`、`diagnostic.operation.*`、`type.operation.*`の内部substringは、それぞれ別kindの完全IDであってOperation IDではない。`operations/<id>.mirakan.json`というFile例はpathとsuffixを除いた`<id>`を分類する。`operation.lod.*`、`operation.feature.*`、`operation.packs.*`、`operation.shooter.*`のようなwildcard／不完全prefixはStable IDではなく、MCD ref、alias、dispatch keyとして必ず拒否する。

current architectureで完全なOperation IDとして現れるtokenは、次の互いに素な三classのexact一つへ分類する。

| class | exact集合／件数 | materialization |
|---|---|---|
| `active_complete` | §8.1／§8.2の14件 | 完全なcurrent MCD／Manifest／Service／Policy／Validator／Diagnostic／Receipt closureだけ |
| `reserved_not_activated` | §§20～21.1の18 family、159件 | `PlannedOperationFamilyV1`だけ。全current集合／alias集合`[]` |
| `example_pending_or_rejected` | 下表の11件 | current／planning／alias集合すべて`[]`。dispatch拒否 |

| exact non-current ID | reason |
|---|---|
| `operation.authoring.apply_changeset` | MCD filename／ID grammarの説明例だけ |
| `operation.build.package.validate` | Naming grammarの説明例だけ |
| `operation.asset.source.plan_import` | Namingのcorrect-direction例だけ |
| `operation.asset.do` | 明示invalid naming例 |
| `operation.debug.propose` | Debug findingからのgeneric proposalとして明示禁止 |
| `operation.performance.migrate_project_scale_envelope_v1_to_v2` | 一度もactivateされていないversion埋込み旧綴り |
| `operation.runtime_ecs.search` | ユーザー確認待ちADRの提案語彙 |
| `operation.runtime_ecs.describe_contract` | ユーザー確認待ちADRの提案語彙 |
| `operation.runtime_ecs.inspect_capture` | ユーザー確認待ちADRの提案語彙 |
| `operation.runtime_ecs.explain_access` | ユーザー確認待ちADRの提案語彙 |
| `operation.runtime_ecs.explain_failure` | ユーザー確認待ちADRの提案語彙 |

lintは完全ID token集合について`active_complete ∪ reserved_not_activated ∪ example_pending_or_rejected`とのset equality、三classのpairwise intersectionが空、current時点の未分類件数exact 0を検査する。説明上のaction名、C++関数、future tokenをOperationへ推測昇格しない。新しい完全`operation.*` tokenを文書へ追加する変更は、同じ変更で完全Activation closureまたはField省略なしのplanning family recordへ追加しなければ失敗する。

さらにprose claim lintは全architecture文書の`registered Operation`、`current Operation`、`public Operation`、`Operationを使う／公開する／登録する`に相当する現在形を検査する。MCDを意味するclaimはexact完全ID、version、`active_complete` membershipを必須とし、family名、action名、PascalCase variantだけのclaimを拒否する。ChangeSet内mutationはexact `ProjectChangePrimitiveV1`等の`*Primitive*`／`*JobKind*` discriminator、未登録Domain語彙は`planned semantic action vocabulary`＋current全投影集合`[]`＋`not_activated`＋future atomic work itemを必須にしてMCD Operation claimから除外する。説明対象が将来Activation後の挙動なら`Activation後`を明記する。これら三形のどれにも分類できない現在形Operation claimの許容件数はexact 0である。

## 22. Contract compilerのDefinition of Done

- 全MCD kindのmeta-schemaと最低1件のvalid／invalid fixtureがある。
- `game_system` kindからCatalog、Dependency Graph、State owner table、C++／TypeScript binding、conformance testを決定論的に生成する。
- Project-defined SystemがEngine Standardと同じContract validationを通り、固定WhitelistなしでCatalogへ登録できる。
- Project Shader planning familyをactivateするwork itemは、同じModule／Technique／Fact／Context／Understanding SchemaからC++、TypeScript、MCP bindingを生成し、R0／R1／R3と禁止Fieldを混同しないことを同じtransactionで検証する。Activation前はbinding集合を空に保つ。
- authoritative State owner欠落／重複、System dependency cycle、Presentation逆writeをinvalid fixtureで拒否する。
- `RemediationV1`のapplicable code、§21.2へ分類済みexact MCD Operation ref付きtemplate、Risk、Approval、禁止Category、適用上限を生成・検証できる。
- Duplicate key、unknown field、unbounded collection、untagged unionを拒否する。
- 同一Inputから二回生成したTree hashが一致する。
- C++／TypeScript／Cooked binary／MCP／OpenAI／Anthropic projectionが一つのMCDから生成される。
- Providerで表現できないConstraintがManifestから欠落しない。
- Provider Outputが必ずInternal validatorで再検証される。
- 全Operation errorが列挙され、string-only errorを持たない。
- 全active OperationについてOwner Manifest Operation集合、MCD Operation集合、Trusted Service allowlist owner contribution、Policy ref、Validator closure、Diagnostic reachable union、Receipt Type、Provider projection集合をそれぞれlike-for-likeで比較し、完全登録済みか明示`not_activated`の厳密に一方である。name-only、partial row、prefix展開を拒否する。
- `MIRAKAN_OPERATION_INTENT_V2` payloadから認可証拠へのedge、認可証拠からnamed `MutationAuthorizationBindingV2`へのedge、binding確定後の`MIRAKAN_OPERATION_REQUEST_V2`へのedgeが一方向であり、Approval／Predelegationからfinal request hashへのedgeが0件である。
- `ContractSetSnapshotV2.members[]`はMCD、Diagnostic、Trusted Service、Validator、Operation Validator Closureの全normative local memberとset equalityであり、各kindの一member byteを変えるmutation fixtureでset rootが必ず変わる。全cross-member edgeはlocal refだけで解決し、root確定前のexternal refを拒否する。
- 全state-changing Operationは`private durable commit marker -> MirakanSignedRecordV1 wrapper -> PublicPublicationMarkerV1＋public state`の順だけを持ち、署名済みwrapperなしのpublic current head、inline Domain signature、別Receipt substitution、二重publication、public後rollbackをcrash-window fixtureで拒否する。
- `ai_mutable` Authoring fieldの外側MCD Operation→typed `ProjectChangePrimitiveV1` coverageが100%で、未到達fieldを持つCapabilityのProvider projectionを拒否する。
- Authoring／World planning familyのatomic activation work itemは、`AuthoringSelectionContextV1`と`WorldAuthoringContextV1`をC++／TypeScript／JSON Schema／MCPへ同じfield ID、bound、Source／Derived区分で生成する。Activation前は両Contextのcurrent Provider／MCP projectionを生成しない。
- World planning familyのatomic activation work itemは、予約候補六Operationがexact revision／hash、omitted range、continuation、typed Diagnosticを返し、screen coordinate、表示row、Hierarchy pathだけのtarget指定を拒否するfixtureを同じtransactionで通す。Activation前は六Operation、Provider／MCP projection、alias集合を空に保つ。
- Scene永続化owner、World composition membership、Space／Topology relation、Derived Cell assignmentを別constraintとして生成し、`MoveEntityToScene`、`UpdateWorldComposition`、Derived Cell writeを相互代替できないinvalid fixtureを持つ。
- World planning familyのatomic activation work itemは、World Capabilityの必須ExampleをProvider projectionとInternal validatorで同じdisposition、Operation ID、Diagnostic code、非変更invariantへ収束させる。Activation前はこのExampleをcurrent Operation成功例として公開しない。
- 全State machineにinvalid transition testがある。
- Cross-language round-tripとboundary fixtureが通る。
- Generated fileの直接編集をCIが検出する。
- Contract lock、Toolchain lock、Generated output hashがVerification Receiptへ記録される。
- `BuildDriverProfileV1`のvalid／invalid fixtureがあり、Makefiles、Android Multi-Config、Generator override、Driver／Target不一致を拒否する。
- Migrationなしの破壊的永続Schema変更をCIが拒否する。
- Authoring planning familyのactivation fixtureは`search`／`read`／`dependencies`／`diff`がrevision、field mask、省略範囲、continuationを保持し、stale Indexと曖昧targetを拒否することを検証する。Activation前のcurrent集合は空である。
- Game System／World planning familyのactivation fixtureはContract／Project revision、Target、maturity、bounded resultを保持し、Activation／Commit authorityをProviderへ投影しないことを検証する。Activation前のcurrent集合は空である。
- LODの全closed enum、Domain別tagged union、enter／exit threshold、fallback、Operation errorにvalid／invalid fixtureがあり、presentation LODからauthoritative stateへ逆入力するReferenceをsemantic validatorが拒否する。
- Anti-aliasのIntent／method／sample／scope／dispositionがclosed enum／tagged unionであり、FXAA／SMAA／MSAA／Mirakan TAA／TAAUとQualified Providerの互換Predicate、fallback、history reset、Target Qualificationを同じMCDから生成する。
- Anti-alias planning familyのatomic activation fixtureは候補五件のC++／TypeScript／MCP／Provider projection、bounded result、stale revision、ambiguous／unsupported result、valid／invalid fixtureを一致させ、Provider activation／Pipeline rebuild／Commit OperationをProviderへ投影しない。Activation前の五候補はcurrent projectionを持たない。
- MSAA×temporal、Hybrid Deferred MSAA、未対応sample count、同一ViewFamilyのsample count混在、pixel-locked layer適用をInternal semantic validatorが同じDiagnostic code familyで拒否する。
- Lighting planning familyのatomic activation fixtureはSource／Intent／Profile／Plan／Snapshot、物理単位tagged union、候補九件、bounded result、stale／lock／Target／Budget／overflowのvalid／invalid fixtureを同じMCDから生成する。Activation前の候補集合は空である。
- Post Process planning familyのatomic activation fixtureはIntent／Profile／Camera Override／Volume／Node Catalog／Plan、固定stage、色空間、AA／Layer／history Predicate、候補九件、bounded resultのvalid／invalid fixtureを同じMCDから生成する。Activation前の候補集合は空である。
- Lighting／Post Processをactivateする場合もProvider projectionへCommit、native GPU resource、任意Render pass／Shader、history内部値、Capability activationを含めず、Internal validatorが未知fieldと未成熟Capabilityをfail-closedにする。
- §§20～21.1のclosed expansion ruleから18 `PlannedOperationFamilyV1`をField省略なしでmaterializeし、ledger／候補表が18対18、候補countが`4+14+4+6+5+9+9+16+6+11+15+24+24+2+1+1+1+7=159`、全current集合とalias集合が明示的な空配列、Capability stateが`not_activated`、family固有activation work itemが一件であることを検査する。159候補の一件でもMCD／Manifest／Service／Provider／MCP current集合へ混入したfixture、empty Fieldを省略して暗黙既定値にしたfixture、§8.1／§8.2のactive 14件をplanning集合へ移したfixtureを拒否する。
- Architecture内の完全な`operation.*` ID tokenを§21.2の三classへexact partitionし、active 14、reserved 159、example／pending／rejected 11、pairwise intersection 0、未分類0を検査する。別kind ID内の`operation.*` substring、不完全prefix、action名をOperation IDとして数えず、逆に完全IDをsubstring扱いで見落とさない。
- 全architecture proseのcurrent／registered／public Operation claimを§21.2のclaim lintへ通し、active完全ID、内部Primitive／Job kind、または明示`not_activated` planned actionのexact一つへ分類する。未分類の現在形claim、IDのないregistered family、Activation前候補を「使う」とする記述を0件にする。

## 23. 一次資料と採用根拠

- JSON Schema dialectのexact versionと取得元は[Toolchain／Dependencies](toolchain-dependencies.md)を参照する。
- [RFC 8259: The JavaScript Object Notation Data Interchange Format](https://www.rfc-editor.org/rfc/rfc8259): 正本JSONのsyntaxと相互運用基準。
- [RFC 8785: JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785): Canonical hashのbyte表現。
- [RFC 4648: Base-N Encodings](https://www.rfc-editor.org/rfc/rfc4648): `bytes_base64url`とSignature転送表現。
- [Microsoft jsonc-parser](https://github.com/microsoft/node-jsonc-parser): strict option付きTree／scanner APIとProperty occurrence保持。
- Parser、canonicalizer、JCS fixtureのexact package／commit／hash／取得元は[Toolchain／Dependencies](toolchain-dependencies.md)を参照する。
- MCP protocolのexact versionと取得元は[Toolchain／Dependencies](toolchain-dependencies.md)を参照する。
- [OpenAI Function calling: strict mode](https://developers.openai.com/api/docs/guides/function-calling#strict-mode): strict subset、全Field required、`additionalProperties: false`、fallback、cache制約。
- [OpenAI Structured Outputs](https://developers.openai.com/api/docs/guides/structured-outputs): Schema adherence、refusal、SchemaとTypeの乖離防止。
- [Anthropic Define tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools): Tool定義、`input_schema`、説明、Example。
- [BCP 14 / RFC 2119 and RFC 8174](https://www.rfc-editor.org/info/bcp14): 規範語の意味。

## 24. 明示的に採用しないもの

- C++ classからだけSchemaを生成し、Requirement／権限／State machineを別管理する方式。
- Provider SDKの型をEngine公開契約にする方式。
- OpenAI strict subsetをMCD全体の表現力上限にする方式。
- TypeScript typeだけをruntime validatorとして扱う方式。
- unknown fieldを黙って保持または無視するforward-compatible reader。
- Generated sourceをRepositoryの手書きSourceと混在させる方式。
- Timestampやabsolute pathをGenerated fileへ埋め込む非再現生成。
- AIが提案したSchemaをmeta-schema未検証で即座にToolとして公開する方式。
