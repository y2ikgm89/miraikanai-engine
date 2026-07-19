# Miraikanai Engine 実行可能契約・Schema・Codegen規約

- 文書版: 1.1
- 作成日: 2026-07-19
- 調査基準日: 2026-07-19
- 対象: Requirement、Capability、Type、Operation、State Machine、Policy、AI Tool、C++／TypeScript／Cooked binary生成
- 状態: プロジェクト公式の規範設計レビュー版
- 上位文書: [Miraikanai Engine AI実装・保守ガバナンス規約](./2026-07-19-ai-engine-development-governance-design.md)
- Game実装方式: [Miraikanai Engine C++実行コード・構造化ゲームデータ規約](./2026-07-19-cpp-structured-game-data-design.md)
- 基盤規約: [Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)
- 検証規約: [Miraikanai Engine AI検証・評価・来歴規約](./2026-07-19-ai-verification-evaluation-provenance-design.md)

## 1. 結論

Miraikanai Engineの要件、型、操作、状態遷移、権限、Budget、Diagnosticを、prose、C++ header、TypeScript type、AI Tool Schemaへ別々に手書きしない。Repositoryの`/schemas/mira/`に置く**Mira Contract Definition（MCD）**を唯一の機械可読正本とし、次を決定論的に生成する。

- Engine内部検証用JSON Schema 2020-12。
- C++20 wire type、enum、validator、serializer、dispatch table。
- TypeScript strict type、runtime validator、JSON-RPC binding。
- GameplayDefinition／CookedGameplayPackageのbinary descriptor、encoder、decoder。
- MCP 2025-11-25 Tool `inputSchema`／`outputSchema`。
- OpenAI strict function／Structured Output向けsubset。
- Anthropic Tool向けProvider projection。
- Editor form、Inspector metadata、human-readable reference。
- Schema fixture、round-trip test、state transition conformance test。

Provider向けSchemaは正本ではない。OpenAI、Anthropic、MCP等の対応Dialectやsubsetが異なるため、MCDを一つのProvider形式へ縮退させない。Provider出力はProvider projectionを通過した後も、C++ Command GatewayがMCDの完全な構造制約と意味制約で再検証する。

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
| Requirement、Type、Operation、State Machine、Capability、Policy、Profile、Provider Profile | `/schemas/mira/**/*.mira.json` | Schema ChangeSetだけ |
| JSON Schema projection | Build treeのgenerated output | 直接編集禁止 |
| C++／TypeScript binding、Cooked binary codec | Build treeのgenerated output | 直接編集禁止 |
| MCP／Provider Tool Schema | Build treeのgenerated output | 直接編集禁止 |
| 人間向けAPI reference | generated docs | 直接編集禁止 |
| 意図、根拠、例、Architecture説明 | Markdown規約／ADR | 人間／AIがReview経由で編集 |
| 手書きsemantic validator | C++／TypeScript source | Requirement IDを付けて編集 |
| Project data | GameSpec／World／GameplayDefinition／Asset metadata | 現在Schemaで検証しChangeSet経由 |

MCDは意図の説明を完全に置き換えない。MCDが機械的な合否、Markdown／ADRが理由とTrade-offを決める。両者が矛盾する場合、実行時合否はMCDに従い、矛盾自体をCI errorにする。

## 4. DirectoryとFile規約

```text
/schemas/
  /mira/
    /meta/
      mira_contract_v1.schema.json
      mira_requirement_v1.schema.json
      mira_type_v1.schema.json
      mira_operation_v1.schema.json
      mira_state_machine_v1.schema.json
      mira_capability_v1.schema.json
      mira_policy_v1.schema.json
      mira_profile_v1.schema.json
      mira_provider_profile_v1.schema.json
    /requirements/
    /types/
    /operations/
    /state_machines/
    /capabilities/
    /policies/
    /profiles/
    /providers/
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

正本FileはRFC 8259 JSON、UTF-8 without BOM、LF、末尾改行ありとする。JSON5、comment、trailing comma、NaN、Infinity、重複keyを禁止する。人間向け注釈は定義済み`description`、`rationale_refs`、`examples`へ記録する。

非RequirementのFile名は正本IDへ`.mira.json`を付けたものとし、例を`operations/operation.authoring.apply_changeset.mira.json`とする。RequirementはIDをASCII lowercase化して`-`を`_`へ変換し、`requirements/requirement.mira_ai_0001.mira.json`とする。この変換以外の略称を許可せず、File pathをIDから決定論的に導出する。同じIDを複数Fileへ定義しない。

## 5. MCD共通Envelope

全MCD documentは次のFieldを持つ。

| Field | 型 | 規則 |
|---|---|---|
| `mcd_version` | uint32 | 初期値1 |
| `kind` | enum | `requirement`、`type`、`operation`、`state_machine`、`capability`、`policy`、`profile`、`provider_profile` |
| `id` | string | Kind別Grammarに従う正本ID。Repository全体で一意 |
| `version` | uint32 | 意味変更ごとに増加、0禁止 |
| `status` | enum | `draft`、`active`、`deprecated`、`retired` |
| `title` | UTF-8 string | 人間向け短い名称 |
| `description` | UTF-8 string | 対象と非対象を明示 |
| `owners` | Owner ID array | 1件以上 |
| `requirement_refs` | Requirement ID array | 自身がrequirementの場合は空 |
| `rationale_refs` | ADR／spec anchor array | 最低1件 |
| `since_contract_set` | uint32 | 初回導入Contract set |
| `supersedes` | `{id, version}` array | 置換対象。空可 |
| `tags` | lowercase string array | ASCII昇順、重複不可 |

`id`をKind固有の別Fieldへ二重保存しない。`requirement`だけは`MIRA-<DOMAIN>-<4桁以上の番号>`、それ以外は`<kind>.<domain>.<lower_snake_name>`を使う。例は`MIRA-AI-0001`、`operation.authoring.apply_changeset`、`capability.render.material.toon_v1`である。Referenceにbare IDを使う場合、現在のContract set内で`status=active`のversionが厳密に1件でなければContract compilerを失敗させる。永続ArtifactとReceiptはbare IDに加えて`contract_set_hash`を持つ。

`status=deprecated`は新規利用を拒否するが、offline migratorが旧Projectを読むための入力Schemaだけに残せる。Runtime、Editor、Game codeへdeprecated branchを生成しない。`retired`はcurrent Contract setの生成対象外である。

## 6. Requirement定義

Requirementは次を必須とする。

| Field | 型／値 |
|---|---|
| 共通`id` | `MIRA-<DOMAIN>-<4桁以上の番号>` |
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
| `timestamp` | RFC 3339 UTC、秒以下桁をProfileで固定 |
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
- `constraints`: length、range、pattern、cardinality、unit、coordinate space。
- `stability`: `stable_id`、`revision_bound`、`ephemeral`。
- `sensitivity`: `public`、`project_private`、`restricted`、`secret`。
- `description`。

`optional`はFieldの`presence`でありTypeではない。`nullable<T>`はFieldが存在した時の値Domainであり、両者を同義にしない。DefaultはField欠落時だけ適用し、明示`null`へ適用しない。AI Provider projectionがOptionalを表せない場合、optionalかつnon-nullableのFieldだけはrequired＋`nullable<T>`へ投影し、`null`を「欠落」へ逆変換できる。optionalかつ`nullable<T>`、または欠落と`null`を区別するFieldは、`{"present": bool, "value": nullable<T>}`の明示presence wrapperを必須とする。`present=false`では`value=null`だけ、`present=true`では元のTのDomainだけを許可する。Gateway受信後にMCDのpresence semanticsへ戻し、変換RuleをProjection Manifestへ記録する。

### 7.3 数値と単位

物理量は裸の`float`にしない。`unit`、`valid_range`、`coordinate_space`、`precision`を定義する。角度はAPIごとにdegree／radianを混在させず、正規Wireはradian、Editor表示だけdegreeを許可する。Lengthはmeter、timeはsecondまたは明示`duration_ns`、colorはlinear／sRGBを型で分ける。

浮動小数点の`-0`は`0`へ正規化し、NaN／Infinityを拒否する。Deterministic hash対象のfloatはIEEE 754 bit patternを直接JSON化せず、Typeごとに定義した最短round-trip decimalへ正規化する。

## 8. Operation定義

各Operationは次を持つ。

| Field | 内容 |
|---|---|
| 共通`id` | `operation.<domain>.<lower_snake_name>` |
| `operation_kind` | `query`、`command`、`event`、`job` |
| `input_type`／`output_type` | exact Type ID＋version |
| `authority` | 実行できる信頼済みService |
| `risk_class` | R0からR5 |
| `side_effects` | Authoring、Source、Network、Process、Release等の集合 |
| `idempotency` | `pure`、`idempotent_with_key`、`non_idempotent` |
| `transaction` | `none`、`read_snapshot`、`authoring_changeset`、`source_promotion` |
| `preconditions` | Predicate ID array |
| `postconditions` | Predicate ID array |
| `errors` | 列挙済みDiagnostic code |
| `timeout_ms` | uint32、0禁止 |
| `rate_limit_policy` | Policy ID |
| `audit_level` | `metadata`、`full_redacted`、`restricted` |
| `provider_exposure` | Provider／MCPへ公開可能か |

未列挙Exception、stringだけのerror、部分成功を禁止する。部分結果が必要なOperationは、成功項目と失敗項目を型付きResultとして明示する。Commandはexpected base revisionをInputへ必須とし、stale revisionを`MIRA-CONFLICT-REVISION_MISMATCH`で拒否する。

MCPへ公開するOperationは`provider_exposure=mcp_proposal`に限定する。正規Commit、Approval発行、Promotion、Releaseは`trusted_internal`とし、Provider projectionを生成しない。

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
| 共通`id` | 例`capability.render.material.toon_v1` |
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

## 11. PolicyとProfile

### 11.1 Policy

PolicyはRisk、Approval、Network、Dependency、Data、Budget、Retry、Release gateを定義する。Policy値の変更はR3以上とし、既存Taskへ遡及適用しない。Task開始時のPolicy set hashをEnvelopeへ固定する。

### 11.2 Profile

ProfileはTarget、Quality、Device、Workspace、AI Provider、Benchmarkを表す。Profileの継承は一段だけ許可し、multiple inheritanceを禁止する。解決後Profileをflat canonical JSONとして生成し、そのhashをBuildとTaskへ保存する。

Profileに未定義の値を環境変数やProvider defaultから暗黙補完しない。Host固有値は`toolchain.lock.json`、Game／Runtime値はMCD Profileへ置く。

`provider_profile`はProvider、API version、受理するSchema keyword、strict機能、Tool／Schema／Context上限、refusal／incomplete形式、retention capabilityを表す正規入力である。Provider固有Projectionそのものは正本にせず、このProfileとType／OperationからBuild treeへ生成する。Provider Manifestはexact Provider Profile ID＋version＋hashを参照する。

## 12. Diagnostic契約

Engine、Contract compiler、Provider adapter、MCP、CLIは共通の`MiraDiagnosticV1`を返す。

| Field | 型／規則 |
|---|---|
| `diagnostic_version` | 1 |
| `code` | `MIRA-<DOMAIN>-<NAME>`、不変 |
| `severity` | `info`、`warning`、`error`、`blocking` |
| `category` | schema、semantic、permission、conflict、build、test、performance、security、provider、infrastructure |
| `message_key` | Localization key |
| `arguments` | primitive map。完成文だけを保存しない |
| `artifact_id`／`revision` | 対象 |
| `location` | JSON Pointerまたはnormalized source location |
| `requirement_ids` | 1件以上。Infrastructureだけ例外 |
| `expected`／`actual` | redacted typed value |
| `remediation_ids` | 機械実行可能または人間向け修正案 |
| `retryability` | `never`、`after_input`、`after_change`、`transient` |
| `cause_chain` | 子Diagnostic ID array |
| `trace_id` | Verification trace参照 |

AIへ返すErrorはこの構造を維持する。Provider向け説明文だけへ変換してcode、location、expected、actualを失わない。Source／static analysis結果はこの形式を正本とし、外部Tool連携用にSARIF 2.1.0へexportする。

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

`tools/contract_compiler`は、基盤規約で固定したNode.js／TypeScript ESM toolchainでBuildするfirst-party CLIとする。Runtime dependencyではなく、offline Build toolである。TypeScript compiler programmatic APIへ依存せず、通常の`tsc`でcompiler自身をBuildする。

JSON treeには`jsonc-parser@3.3.1`、JCSにはRFC 8785 Appendix GがJavaScript実装として挙げる`canonicalize@3.0.0`をexact dependencyとして採用する。どちらもBuild-only、0 transitive dependencyである。Inputは`Buffer`から`TextDecoder("utf-8", {fatal:true})`でdecodeし、`parseTree`を`disallowComments=true`、`allowTrailingComma=false`で呼ぶ。Tree上の全Property occurrenceを走査してdecoded key重複を拒否した後だけData modelへ変換する。`parse()`または`JSON.parse`で重複情報を失ってから検査してはならない。

2026-07-19の採用検証では、上記exact packageをNode.js 24 ESMで実行し、comment／trailing comma拒否、`"a"`と`"\u0061"`のProperty occurrence保持、RFC 8785公式Repository commit `19d51d7fe467d4706a3ff08adf8a748f29fc21e0`の6組12 File、合計1,476 byteのinput／output fixtureでbyte一致を確認した。Fixture Artifact rootは13節のleaf／parent framingだけを再利用し、`p`をRepository相対の`testdata/input/<name>`または`testdata/output/<name>`、`d`を各Fixtureの**raw file byte**のSHA-256とする。入力Fixtureは意図的に非canonicalなため、MCD用のJCS document hashへ置き換えない。この定義によるrootは`49ebd08bec39f4da9e2db03cffc76b2de984912fd6fbc66ec4ee33852b7b84fb`である。これは一回限りの保証にせず、同じfixtureを`contract-fast` CIへ固定する。

処理順序を固定する。

1. RFC 8259としてparseし、重複keyを拒否する。標準`JSON.parse`だけでは重複keyを検出できないため、UTF-8 byte列へduplicate-aware tokenizerを先に適用し、Object scopeごとのdecoded key一致を検査する。
2. Kind別meta-schemaでvalidateする。
3. 全IDとversionをindex化する。
4. 参照解決、cycle、owner、requirement coverageを検査する。
5. Semantic lintとPolicy lintを実行する。
6. Canonicalizeして`/schemas/contract.lock.json`と照合する。
7. Internal JSON Schema 2020-12を生成する。
8. Language bindingを生成する。
9. Provider／MCP projectionを生成する。
10. Docs、fixture、transition testを生成する。
11. Golden output hashと再生成差分を検査する。

Generator plugin方式は初期に採用しない。全Generatorは同一Repository、同一Process、exhaustive dispatchで管理する。外部Provider追加はContract compiler sourceの明示変更とProvider conformance suiteを必要とする。

Duplicate-aware走査はString escapeをdecodeした後のKeyで比較するため、`"a"`と`"\u0061"`も重複として拒否する。ParserとJCSにはRFC 8259／8785のofficial test vector、invalid UTF-8、unpaired surrogate、深さ／byte上限、Property-based differential testを必須にする。Package更新はR3 Dependency Changeであり、同じfixtureからbyte-for-byte同じJCSを出す場合だけ昇格できる。

## 15. Internal JSON Schema projection

Internal validation projectionはJSON Schema Draft 2020-12を使い、`$schema`を必ず明示する。可能な構造制約をすべて表現するが、次は手書きまたは生成semantic validatorへ分離する。

- Entity間参照整合性。
- Target Capabilityの組合せ。
- Runtime phaseとwriter authority。
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

### 16.2 MCP 2025-11-25

MCP projectionは`inputSchema`と`outputSchema`へJSON Schema 2020-12を明示する。Tool resultは`structuredContent`と互換用の同値JSON textを返す。ServerはOutputを送信前に検証し、Clientが検証しない場合でも安全性が変わらないようにする。

Tool annotationは非信頼表示Hintとして生成する。Access control、Risk、ApprovalはServer Policyで強制する。Tool名は`mira.<domain>.<verb>`、ASCII、128文字以下、Repository全体で一意とする。

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

Anthropic projectionはTool `name`、詳細な`description`、`input_schema`、対応Provider versionで利用可能な`strict`等をProvider Manifestから生成する。Providerが受理するJSON Schema keyword集合をconformance testで検出し、未検証keywordを使用しない。

複雑なToolにはMCDのvalid fixtureから少数の`input_examples`を生成できる。ただし、ExampleはPrompt tokenを消費するため、EvalでTool選択またはargument精度を改善した場合だけProvider Manifestへ有効化する。

### 16.5 CLI／Desktop App

Codex／Claude等のCLIとDesktop Appは原則MCP projectionを使う。Provider固有PluginがMCDを独自変換してはならず、Miraikanai MCP ServerのTool一覧とSchemaを取得する。Source直接編集はMCP Tool権限とは別に、AI実装・保守ガバナンス規約のSource Worker sandboxを適用する。

## 17. Language／Runtime projection

### 17.1 C++20

生成C++は次に従う。

- Namespaceは`mira::contract::<domain>`。
- Public wire structはstandard-layoutを要求せず、Field access APIを使う。
- Owning string／vectorまたは明示viewを型で分ける。
- Deserializerはunknown field、duplicate field、range超過、invalid UTF-8を拒否する。
- `std::expected`相当のEngine `Result<T, MiraDiagnostic>`を返し、Exceptionをwire境界から出さない。
- Enumはclosed enumとし、unknown値をdefaultへ変換しない。
- HandleはID＋generationでありpointerを含めない。
- Generated headerにInput contract hashを記録する。

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
- Decode errorはMiraDiagnosticへ変換し、部分objectをRuntimeへ公開しない。

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

## 20. AI向けDiscovery

AIへ巨大な全Schemaを一括送信しない。次の二段階Discoveryを使う。

1. `capabilities.search`がID、title、tag、Target、maturity、短いsummaryを返す。
2. `capabilities.read`が選択したCapabilityのType、Operation、Constraint、Budget、Exampleを返す。

Search結果は現在Contract setのhashを含む。AIが古いhashのCapabilityでProposalを送った場合、Gatewayはstaleとして拒否し、差分を返す。AIがSchemaにないFieldやOperationを使った場合、fuzzyに推測して補正せず、候補ID付きDiagnosticを返す。

## 21. Contract compilerのDefinition of Done

- 全MCD kindのmeta-schemaと最低1件のvalid／invalid fixtureがある。
- Duplicate key、unknown field、unbounded collection、untagged unionを拒否する。
- 同一Inputから二回生成したTree hashが一致する。
- C++／TypeScript／Cooked binary／MCP／OpenAI／Anthropic projectionが一つのMCDから生成される。
- Providerで表現できないConstraintがManifestから欠落しない。
- Provider Outputが必ずInternal validatorで再検証される。
- 全Operation errorが列挙され、string-only errorを持たない。
- 全State machineにinvalid transition testがある。
- Cross-language round-tripとboundary fixtureが通る。
- Generated fileの直接編集をCIが検出する。
- Contract lock、Toolchain lock、Generated output hashがVerification Receiptへ記録される。
- Migrationなしの破壊的永続Schema変更をCIが拒否する。

## 22. 一次資料と採用根拠

- [JSON Schema Draft 2020-12](https://json-schema.org/draft/2020-12): Engine内部SchemaのDialectとmeta-schema。
- [JSON Schema 2020-12 Release Notes](https://json-schema.org/draft/2020-12/release-notes): tuple、dynamic reference、unevaluated等のDialect差。
- [RFC 8259: The JavaScript Object Notation Data Interchange Format](https://www.rfc-editor.org/rfc/rfc8259): 正本JSONのsyntaxと相互運用基準。
- [RFC 8785: JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785): Canonical hashのbyte表現。
- [RFC 4648: Base-N Encodings](https://www.rfc-editor.org/rfc/rfc4648): `bytes_base64url`とSignature転送表現。
- [Microsoft jsonc-parser](https://github.com/microsoft/node-jsonc-parser): strict option付きTree／scanner APIとProperty occurrence保持。
- [npm canonicalize](https://www.npmjs.com/package/canonicalize/v/3.0.0): RFC 8785 Appendix G掲載実装の固定Build-only package。
- [RFC 8785 implementation test data, pinned commit](https://github.com/cyberphone/json-canonicalization/tree/19d51d7fe467d4706a3ff08adf8a748f29fc21e0/testdata): JCSのcross-implementation input／output fixture。
- [Model Context Protocol 2025-11-25 Tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools): `inputSchema`／`outputSchema`、JSON Schema 2020-12 default、structured result、validation。
- [OpenAI Function calling: strict mode](https://developers.openai.com/api/docs/guides/function-calling#strict-mode): strict subset、全Field required、`additionalProperties: false`、fallback、cache制約。
- [OpenAI Structured Outputs](https://developers.openai.com/api/docs/guides/structured-outputs): Schema adherence、refusal、SchemaとTypeの乖離防止。
- [Anthropic Define tools](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools): Tool定義、`input_schema`、説明、Example。
- [BCP 14 / RFC 2119 and RFC 8174](https://www.rfc-editor.org/info/bcp14): 規範語の意味。

## 23. 明示的に採用しないもの

- C++ classからだけSchemaを生成し、Requirement／権限／State machineを別管理する方式。
- Provider SDKの型をEngine公開契約にする方式。
- OpenAI strict subsetをMCD全体の表現力上限にする方式。
- TypeScript typeだけをruntime validatorとして扱う方式。
- unknown fieldを黙って保持または無視するforward-compatible reader。
- Generated sourceをRepositoryの手書きSourceと混在させる方式。
- Timestampやabsolute pathをGenerated fileへ埋め込む非再現生成。
- AIが提案したSchemaをmeta-schema未検証で即座にToolとして公開する方式。
