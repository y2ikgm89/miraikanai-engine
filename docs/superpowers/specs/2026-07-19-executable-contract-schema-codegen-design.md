# Miraikanai Engine 実行可能契約・Schema・Codegen規約

- 文書版: 1.13
- 作成日: 2026-07-19
- 調査基準日: 2026-07-20
- 対象: Requirement、Capability、Game System、Type、Operation、State Machine、Policy、AI Tool、C++／TypeScript／Cooked binary生成
- 状態: プロジェクト公式の規範設計レビュー版
- 上位文書: [Miraikanai Engine AI実装・保守ガバナンス規約](./2026-07-19-ai-engine-development-governance-design.md)
- Game実装方式: [Miraikanai Engine C++実行コード・構造化ゲームデータ規約](./2026-07-19-cpp-structured-game-data-design.md)
- 基盤規約: [Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)
- Math／Core Utilities正本: [Miraikanai Engine AI可読Math／Core Utilitiesアーキテクチャ規約](./2026-07-20-ai-readable-math-core-utilities-architecture-design.md)
- C++言語・Modules規約: [Miraikanai Engine C++23・Named Modules・`import std`移行規約](./2026-07-20-cpp23-modules-import-std-transition-design.md)
- Authoring規約: [Miraikanai Engine Authoring Model／Project State規約](./2026-07-19-authoring-model-project-state-design.md)
- 検証規約: [Miraikanai Engine AI検証・評価・来歴規約](./2026-07-19-ai-verification-evaluation-provenance-design.md)
- LOD正本: [Miraikanai Engine AI可読LODアーキテクチャ規約](./2026-07-20-ai-readable-lod-architecture-design.md)
- Renderer／Anti-alias実行正本: [Miraikanai Engine Rendering／Render Graphアーキテクチャ規約](./2026-07-19-rendering-render-graph-architecture-design.md)
- Lighting正本: [Miraikanai Engine Lighting／AI Authoringアーキテクチャ規約](./2026-07-20-lighting-ai-authoring-architecture-design.md)
- Post Process正本: [Miraikanai Engine Post Process／AI Authoringアーキテクチャ規約](./2026-07-20-post-process-ai-authoring-architecture-design.md)
- Game System正本: [Miraikanai Engine Game System／AI Code Generationアーキテクチャ規約](./2026-07-20-game-system-ai-codegen-architecture-design.md)
- World／Level／Map正本: [Miraikanai Engine World／Level／Map／AI Authoringアーキテクチャ規約](./2026-07-20-world-level-map-ai-authoring-architecture-design.md)
- Debugging正本: [Miraikanai Engine AI可読Debugging／Observability／Replayアーキテクチャ規約](./2026-07-20-ai-readable-debugging-observability-replay-architecture-design.md)

## 1. 結論

Miraikanai Engineの要件、型、操作、状態遷移、権限、Budget、Diagnosticを、prose、C++ header、TypeScript type、AI Tool Schemaへ別々に手書きしない。Repositoryの`/schemas/mirakan/`に置く**Miraikanai Contract Definition（MCD）**を唯一の機械可読正本とし、次を決定論的に生成する。

- Engine内部検証用JSON Schema 2020-12。
- C++23 wire type、enum、validator、serializer、dispatch table、Named Module interface、C ABI Header。
- TypeScript strict type、runtime validator、JSON-RPC binding。
- GameplayDefinition／CookedGameplayPackageのbinary descriptor、encoder、decoder。
- MCP 2025-11-25 Tool `inputSchema`／`outputSchema`。
- OpenAI strict function／Structured Output向けsubset。
- Anthropic Tool向けProvider projection。
- Editor form、Inspector metadata、human-readable reference。
- Game System Catalog、State owner table、dependency graph fragment、System conformance test。
- Schema fixture、round-trip test、state transition conformance test。

Provider向けSchemaは正本ではない。OpenAI、Anthropic、MCP等の対応Dialectやsubsetが異なるため、MCDを一つのProvider形式へ縮退させない。Provider出力はProvider projectionを通過した後も、C++ Command GatewayがMCDの完全な構造制約と意味制約で再検証する。

LODは単一の裸`lod_index`として公開せず、`LodIntentV1`、Domain別Policy、`LodResolutionPlanV1`、`LodQualificationReceiptV1`をMCDへ登録する。`operation.lod.*`は本規約8節のOperation形式、authority、revision、typed errorへ従い、Generated Artifactまたはruntime選択結果を直接編集するCommandを生成しない。

Anti-aliasも裸のmethod名またはconsole variableとして公開せず、`AntiAliasingIntentV1`、`ResolvedAntiAliasingPlanV1`、closed method／sample enum、互換Predicate、Target QualificationをMCDへ登録する。AIはIntentとbounded query／proposalだけを扱い、resolved Plan、Render Graph Pass、Provider activation、Pipeline rebuildを直接編集しない。

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
| Project data | GameSpec／World／Level／System Implementation Set／GameplayDefinition／Asset metadata | 現在Schemaで検証しChangeSet経由 |

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

正本FileはRFC 8259 JSON、UTF-8 without BOM、LF、末尾改行ありとする。JSON5、comment、trailing comma、NaN、Infinity、重複keyを禁止する。人間向け注釈は定義済み`description`、`rationale_refs`、`examples`へ記録する。

非RequirementのFile名は正本IDへ`.mirakan.json`を付けたものとし、例を`operations/operation.authoring.apply_changeset.mirakan.json`とする。RequirementはIDをASCII lowercase化して`-`を`_`へ変換し、`requirements/requirement.mirakan_ai_0001.mirakan.json`とする。この変換以外の略称を許可せず、File pathをIDから決定論的に導出する。同じIDを複数Fileへ定義しない。

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
| `requirement_refs` | Requirement ID array | 自身がrequirementの場合は空 |
| `rationale_refs` | ADR／spec anchor array | 最低1件 |
| `since_contract_set` | uint32 | 初回導入Contract set |
| `supersedes` | `{id, version}` array | 置換対象。空可 |
| `tags` | lowercase string array | ASCII昇順、重複不可 |

`id`をKind固有の別Fieldへ二重保存しない。`requirement`だけは`MIRAKAN-<DOMAIN>-<4桁以上の番号>`、それ以外は`<kind>.<namespace_path>`を使う。`namespace_path`は2～8個のdot区切りsegmentで、各segmentはASCII lowercase、先頭英字、以後英数字またはunderscore、1～48文字とする。例は`MIRAKAN-AI-0001`、`operation.authoring.apply_changeset`、`capability.render.material.toon_v1`、`game_system.engine.combat`、`remediation.authoring.refresh_context`である。

MCDへの永続参照を`McdContractRefV1 { id: string, version: uint32, contract_set_hash: SHA-256 }`へ固定する。`id`のkindと参照Fieldが要求するkindは一致し、`version`は同じContract set内で存在して`status=active`でなければならない。Bare IDは固定済み`contract_set_hash`を入力にするEditor／AIのread-only検索だけで使用でき、候補が厳密に1件でなければ解決しない。Project Source、Cooked Artifact、Save、Replay、Receipt、ChangeSetはbare IDまたはruntime numeric IDを永続参照に使用しない。`GameSystemContractRefV1`は`McdContractRefV1`のうちkindが`game_system`である型付きaliasとする。

`status=deprecated`は新規利用を拒否するが、offline migratorが旧Projectを読むための入力Schemaだけに残せる。Runtime、Editor、Game codeへdeprecated branchを生成しない。`retired`はcurrent Contract setの生成対象外である。

## 6. Requirement定義

Requirementは次を必須とする。

| Field | 型／値 |
|---|---|
| 共通`id` | `MIRAKAN-<DOMAIN>-<4桁以上の番号>` |
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
- `constraints`: length、range、pattern、cardinality、unit、coordinate space、frame、normalization、finite policy、canonicalization。
- `stability`: `stable_id`、`revision_bound`、`ephemeral`。
- `sensitivity`: `public`、`project_private`、`restricted`、`secret`。
- `description`。

`optional`はFieldの`presence`でありTypeではない。`nullable<T>`はFieldが存在した時の値Domainであり、両者を同義にしない。DefaultはField欠落時だけ適用し、明示`null`へ適用しない。AI Provider projectionがOptionalを表せない場合、optionalかつnon-nullableのFieldだけはrequired＋`nullable<T>`へ投影し、`null`を「欠落」へ逆変換できる。optionalかつ`nullable<T>`、または欠落と`null`を区別するFieldは、`{"present": bool, "value": nullable<T>}`の明示presence wrapperを必須とする。`present=false`では`value=null`だけ、`present=true`では元のTのDomainだけを許可する。Gateway受信後にMCDのpresence semanticsへ戻し、変換RuleをProjection Manifestへ記録する。

### 7.3 数値と単位

物理量は裸の`float`または意味なしvectorにしない。`semantic_role`、`dimension`、`scalar_type`、`unit`、`valid_range`、`coordinate_space`、`frame_kind`、`normalization`、`precision`、`finite_policy`、`canonicalization`、`wire_layout`を定義する。Position、Displacement、Direction、UnitDirection、Velocity、Scale、Quaternion、Transform、Color、UVを別Typeとして表し、初期Type catalogとC++ storage／semantic境界はMath／Core Utilities規約を正本とする。角度はAPIごとにdegree／radianを混在させず、正規Wireはradian、Editor表示だけdegreeを許可する。Lengthはmeter、timeはsecondまたは明示`duration_ns`、colorはlinear／sRGBを型で分ける。

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

未列挙Exception、stringだけのerror、部分成功を禁止する。部分結果が必要なOperationは、成功項目と失敗項目を型付きResultとして明示する。Commandはexpected base revisionをInputへ必須とし、stale revisionを`MIRAKAN-CONFLICT-REVISION_MISMATCH`で拒否する。

MCPへ公開するOperationは`provider_exposure=mcp_proposal`に限定する。正規Commit、Approval発行、Promotion、Releaseは`trusted_internal`とし、Provider projectionを生成しない。

Authoring Typeのfieldは`mutability = immutable | human_mutable | ai_mutable`と、変更可能なOperation ID集合を持つ。Contract compilerは`ai_mutable` fieldからtyped CommandまたはDomain Operationへの到達性を全件検査し、coverageが100%でなければ該当CapabilityのProvider／MCP projectionを生成しない。自由形式のJSON Pointer write、任意path write、Operationを迂回するSource writeをcoverageとして数えない。

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

### 10.1 Game System定義

MCD kind `game_system`はEngine Capabilityの利用者であるGameplay単位を表す。完全なField、namespace、State owner、dependency、implementation、Bundle規則はGame System／AI Code Generation規約を正本とし、MCDでは`GameSystemSpecV1`として次を必須にする。

- `system_origin`、`semantic_role_ids`、責務／非責務Requirement。
- Runtime instance scope、State class、authoritative State owner。
- accepted Command、emitted Event、read Snapshot。
- required／provided Capability、phase、dependency edge。
- Implementation Policy、Save／Replay、Target別Budget、fallback。
- fixture、compatibility invariant、extension policy。

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

`CxxFrontendProfileV1`は`cxx23_headers_bootstrap`、`cxx23_modules_probe`、`cxx23_modules_candidate`、`cxx23_modules_shipping`のclosed enum、許可遷移、Promotion可否を持つ。ProviderやAIがProfile IDを追加できず、Build Gatewayが`toolchain.lock.json`のCompiler／STL／CMake bindingと照合する。

`CppDependencySetV1`はowner component／Primary Module、public／private import、closed `StdHeaderId`、closed Header例外を正規化して表す。AIはraw include pathやCompiler flagをDependencyとして保存しない。Contract compilerはCX0で個別標準Header、CX1以降でNamed Module／`import std`へ投影し、Source scannerは実Sourceとの一致を検証する。Field、順序、Header例外、Cutover後のProjection停止条件はC++言語・Modules規約を基準とする。

`BuildDriverProfileV1`は`driver_profile_id`、`target_profile_id`、`allowed_frontend_profile_ids`、`configure_driver`、`cpp_generator`、`configuration_model`、`package_owner`を持つ。IDと組合せは基盤規約のclosed setだけを許可し、AI、Provider、Project、Environmentが任意Driver、Generator、commandを追加できない。Contract compilerはWindows／Appleのchecked-in CMake Preset検査表、Android Gradle CMake検査表、Build Gateway allowlistへ投影するが、CMake／Gradle SourceそのものをMCDへ埋め込まない。

ValidatorはTarget、C++ Frontend Profile、Driver Profileの全組合せを照合し、First-party Makefiles／`ndk-build`、Android Ninja Multi-Config、Generator override、異なるBuild tree identityの再利用を拒否する。

## 12. Diagnostic契約

### 12.1 `MirakanDiagnosticV1`

Engine、Contract compiler、Provider adapter、MCP、CLIは共通の`MirakanDiagnosticV1`を返す。

| Field | 型／規則 |
|---|---|
| `diagnostic_version` | 1 |
| `code` | `MIRAKAN-<DOMAIN>-<NAME>`、不変 |
| `severity` | `info`、`warning`、`error`、`blocking` |
| `category` | schema、semantic、permission、conflict、build、test、performance、security、provider、infrastructure |
| `message_key` | Localization key |
| `arguments` | primitive map。完成文だけを保存しない |
| `artifact_id`／`revision` | 対象 |
| `location` | JSON Pointerまたはnormalized source location |
| `target_stable_ids` | 実在確認済み対象ID。候補の場合は候補理由を`arguments`へ含める |
| `requirement_ids` | 1件以上。Infrastructureだけ例外 |
| `expected`／`actual` | redacted typed value |
| `remediation_ids` | 機械実行可能または人間向け修正案 |
| `retryability` | `never`、`after_input`、`after_change`、`transient` |
| `cause_chain` | 子Diagnostic ID array |
| `trace_id` | Verification trace参照 |

AIへ返すErrorはこの構造を維持する。Provider向け説明文だけへ変換してcode、location、expected、actualを失わない。Source／static analysis結果はこの形式を正本とし、外部Tool連携用にSARIF 2.1.0へexportする。

`MirakanDiagnosticV1`は検証結果と修復入口の正本であり、Debug Event全般の代替ではない。Debugging規約の`DebugEventEnvelopeV1`は必要時に`diagnostic_id`／`cause_chain`／`trace_id`を参照し、severity、location、expected／actualを別Schemaへ重複保存しない。反対に高頻度counter、span、frame marker、domain snapshotを`MirakanDiagnosticV1`として発行してはならない。

### 12.2 `RemediationV1`

`remediation_ids`は自由文ではなく、MCDの`RemediationV1`を参照する。

| Field | 規則 |
|---|---|
| `id`／`version` | `remediation.<domain>.<lower_snake_name>`、意味変更ごとにversion増加 |
| `applicable_codes` | `MirakanDiagnosticV1.code`のclosed set |
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

`tools/contract_compiler`は、基盤規約で固定したNode.js／TypeScript ESM toolchainでBuildするfirst-party CLIとする。Runtime dependencyではなく、offline Build toolである。TypeScript compiler programmatic APIへ依存せず、通常の`tsc`でcompiler自身をBuildする。

JSON treeには`jsonc-parser@3.3.1`、JCSにはRFC 8785 Appendix GがJavaScript実装として挙げる`canonicalize@3.0.0`をexact dependencyとして採用する。どちらもBuild-only、0 transitive dependencyである。Inputは`Buffer`から`TextDecoder("utf-8", {fatal:true})`でdecodeし、`parseTree`を`disallowComments=true`、`allowTrailingComma=false`で呼ぶ。Tree上の全Property occurrenceを走査してdecoded key重複を拒否した後だけData modelへ変換する。`parse()`または`JSON.parse`で重複情報を失ってから検査してはならない。

2026-07-19の採用検証では、上記exact packageをNode.js 24 ESMで実行し、comment／trailing comma拒否、`"a"`と`"\u0061"`のProperty occurrence保持、RFC 8785公式Repository commit `19d51d7fe467d4706a3ff08adf8a748f29fc21e0`の6組12 File、合計1,476 byteのinput／output fixtureでbyte一致を確認した。Fixture Artifact rootは13節のleaf／parent framingだけを再利用し、`p`をRepository相対の`testdata/input/<name>`または`testdata/output/<name>`、`d`を各Fixtureの**raw file byte**のSHA-256とする。入力Fixtureは意図的に非canonicalなため、MCD用のJCS document hashへ置き換えない。この定義によるrootは`49ebd08bec39f4da9e2db03cffc76b2de984912fd6fbc66ec4ee33852b7b84fb`である。これは一回限りの保証にせず、同じfixtureを`contract-fast` CIへ固定する。

処理順序を固定する。

1. RFC 8259としてparseし、重複keyを拒否する。標準`JSON.parse`だけでは重複keyを検出できないため、UTF-8 byte列へduplicate-aware tokenizerを先に適用し、Object scopeごとのdecoded key一致を検査する。
2. Kind別meta-schemaでvalidateする。
3. 全IDとversionをindex化する。
4. 参照解決、cycle、Game System State owner、phase edge、requirement coverageを検査する。
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
- Game Systemのauthoritative State owner、dependency cycle、Implementation Variant conformance。
- World／Scene／Level／Cell identity、Topology、Streaming PlanのSource／Derived境界。
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

Anthropic projectionはTool `name`、詳細な`description`、`input_schema`、対応Provider versionで利用可能な`strict`等をProvider Manifestから生成する。Providerが受理するJSON Schema keyword集合をconformance testで検出し、未検証keywordを使用しない。

複雑なToolにはMCDのvalid fixtureから少数の`input_examples`を生成できる。ただし、ExampleはPrompt tokenを消費するため、EvalでTool選択またはargument精度を改善した場合だけProvider Manifestへ有効化する。

### 16.5 CLI／Desktop App

Codex／Claude等のCLIとDesktop Appは原則MCP projectionを使う。Provider固有PluginがMCDを独自変換してはならず、Miraikanai MCP ServerのTool一覧とSchemaを取得する。Source直接編集はMCP Tool権限とは別に、AI実装・保守ガバナンス規約のSource Worker sandboxを適用する。

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

## 20. AI向けDiscovery

AIへ巨大な全Schemaを一括送信しない。次の二段階Discoveryを使う。

1. `capabilities.search`がID、title、tag、Target、maturity、短いsummaryを返す。
2. `capabilities.read`が選択したCapabilityのType、Operation、Constraint、Budget、Exampleを返す。

Search結果は現在Contract setのhashを含む。AIが古いhashのCapabilityでProposalを送った場合、Gatewayはstaleとして拒否し、差分を返す。AIがSchemaにないFieldやOperationを使った場合、fuzzyに推測して補正せず、候補ID付きDiagnosticを返す。

Authoring dataは同じDiscovery原則で次のR0 queryだけを公開する。

| Operation | 結果 |
|---|---|
| `operation.authoring.search` | kind、tag、Component、name token、spatial boundからStableId候補とscore理由を返す |
| `operation.authoring.read` | StableId、field mask、expected revisionからbounded `SceneSliceV1`またはDocument projectionを返す |
| `operation.authoring.dependencies` | inbound／outbound、Requirement、Capability、Decision、lockのbounded closureを返す |
| `operation.authoring.diff` | base／target revisionとStableId scopeからsemantic diff、storage-only diff、continuationを返す |

全Queryは`project_revision`、`contract_set_hash`、`authoring_index_revision`、`query_hash`、`omitted_ranges`、`continuation_cursor`を返す。別revisionへのfallback、表示index、曖昧な名前だけのtarget確定、任意JSON断片を禁止する。検索結果が複数候補ならAIが名前から推測せず、追加readまたは人間選択を行う。

LOD Discoveryは`lod_class`、semantic role、Target、Qualityで絞り込み、Intent、該当Domain Policy、fallback、選択metric、現在のqualification statusだけを返す。全DomainのLOD Schemaやruntime telemetryを常に一括送信しない。

Game System Discoveryは次のMCD OperationからProvider別Tool名を生成する。MCP／製品表示上のaliasはそれぞれ`mirakan.systems.search`、`mirakan.systems.read`、`mirakan.systems.plan`、`mirakan.systems.validate_bundle`とする。

| MCD Operation | Authority | 結果 |
|---|---|---|
| `operation.systems.search` | R0 query | Role、Target、maturity、originでCatalog entryを検索 |
| `operation.systems.read` | R0 query | exact System Contract、constraint、budget、fixtureを取得 |
| `operation.systems.plan` | R1 proposal | `SystemImplementationPlanV1`を提案 |
| `operation.systems.validate_bundle` | R0 query／job | Staging `SystemBundleChangeSetV1`を検証 |

World Discoveryも次のMCD Operationを正本とし、同じInput／Output Schemaから`mirakan.worlds.*` aliasを生成する。

| MCD Operation | Authority | 結果 |
|---|---|---|
| `operation.worlds.search` | R0 query | kind、role、tag、Target、spatial boundからWorld／Region／Level／Scene候補、StableId、score理由を返す |
| `operation.worlds.read` | R0 query | exact Stable ref、field mask、Viewport、Targetからbounded `WorldAuthoringContextV1`を返す |
| `operation.worlds.resolve_map_intent` | R0 query／R1 proposal | 6分類候補、Evidence、`resolved \| question_required \| rejected`を返す |
| `operation.worlds.plan_change` | R1 proposal | allowed Domain Operation ID／version、precondition、Budget、fixtureを持つ`WorldAuthoringPlanV1`を返す |
| `operation.worlds.validate_bundle` | R0 query／job | Staging BundleのSchema、semantic、reference、ownership、Topology、Budget Diagnosticを返す |
| `operation.worlds.preview_bundle` | R0 query／job | Source／Topology／Level／Target別Derived差分、playability、performance、fallback比較を返す |

`operation.worlds.read`はAuthoring規約の`AuthoringSelectionContextV1` hashを任意入力として受けられるが、screen coordinate、表示row、Hierarchy path、表示名だけをtargetへ変換しない。出力はProject revision、Contract set hash、Source Document hash、Source／Staging／Derived read-only／Runtime区分、omitted range、continuationを必須とする。Derived Artifactはread-only refだけを返し、Cell、Navmesh、HLOD、Runtime handleのwrite Operation Schemaを生成しない。

`operation.worlds.plan_change`が返すDomain OperationはWorld規約のCatalogに存在し、`ai_mutable` field coverage、expected Document revision、precondition hash、Risk、Approval、inverse availabilityを満たすものだけにする。`MoveEntityToScene`、`SetLevelSourceScenes`、Target別Cell生成を相互代替にせず、Scene永続化owner、Level membership、Cell assignmentを別constraintとしてProvider Schemaとserver-enforced semantic validatorへ投影する。

World CapabilityのMCD `examples`は最低でも、明確な`playable_level`、曖昧で`question_required`、共有Scene変更の影響Level列挙、Derived Cell直接write拒否、stale revision拒否を各1件含む。Exampleは説明文だけでなく、exact Input、expected disposition／Operation ID、expected Diagnostic code、変更されないinvariantを持つ。

`MapIntentResolutionV1.disposition=question_required`をCommit可能Proposalへ自動変換しない。System／WorldのActivation、Source Promotion、Project Commit OperationはProvider projectionへ含めない。

Anti-alias Discoveryは2D／3D機能計画の意味GoalとRenderer規約の実行制約を、次のbounded MCD Operationへ投影する。MCP aliasは同じInput／Output Schemaから`mirakan.rendering.aa.search`、`mirakan.rendering.aa.read`、`mirakan.rendering.aa.resolve_intent`、`mirakan.rendering.aa.plan_change`、`mirakan.rendering.aa.preview_change`を生成する。

| MCD Operation | Risk／kind | 結果 |
|---|---|---|
| `operation.rendering.aa.search` | R0 query | 意味Goal、Target、Renderer、Quality、maturityからIntent／method候補を検索し、候補ID、短い適合理由、制約要約を返す |
| `operation.rendering.aa.read` | R0 query | exact Intent／method／Profileの互換Predicate、sample count、layer scope、cost model、fallback、Diagnostic、必要Qualificationを返す |
| `operation.rendering.aa.resolve_intent` | R0 query | 永続変更なしでIntentをViewFamily単位Planへ決定的に解決し、採用候補、却下候補と理由、`resolved`／`question_required`／`unsupported`を返す |
| `operation.rendering.aa.plan_change` | R1 proposal | expected Project revisionに対するtyped `AntiAliasingChangeSetProposalV1`だけを生成する。Commit、Provider activation、Pipeline rebuildは行わない |
| `operation.rendering.aa.preview_change` | R0 query／job | ProposalをStagingで検証し、resolved Plan差分、Graph／history影響、GPU／memory／bandwidth見積り、visual fixture要求、fallback、必要Receiptを返す |

全Anti-alias Query／Proposal結果は`project_revision`、`contract_set_hash`、`capability_signature_hash`、`renderer_profile_revision`、`qualification_receipt_hashes`、`query_hash`を返し、ViewFamilyへ解決した結果は`view_family_id`と`source_intent_revision`も返す。bounded collectionは`omitted_ranges`と`continuation_cursor`を持つ。`plan_change`は`expected_project_revision`、`idempotency_key`、変更理由、対象scopeを必須にし、`preview_change`はProposal hashと同じrevisionを必須にする。stale revision、未知method、未Qualified Target、MSAA×temporal、Hybrid Deferred MSAA、異なるsample countの同一ViewFamily混在、pixel-locked layerへのAA適用をtyped Diagnosticでfail-closedにする。

`AntiAliasingIntentV1`の`mode_policy`は`auto | fixed`、`preferred_method`は`none | fxaa | smaa_1x | msaa | mirakan_taa_v1 | mirakan_taau_v1 | qualified_provider`、`msaa_samples`は`auto | 2 | 4 | 8`のclosed enum／unionとする。`msaa`以外で2／4／8を指定した入力を拒否し、`qualified_provider`はCatalogのexact Provider IDを必須にする。`none`は明示User指定、bit-exact diagnostic、AA対象外layerだけに許可し、AIの性能最適化候補にしない。Provider／MCPへ投影するのは上記五Operationだけであり、`ResolvedAntiAliasingPlanV1` write、arbitrary Render Graph write、Provider install／activate、Settings Apply、Source Promotion、Project CommitをTool listへ含めない。

Lighting DiscoveryはLighting規約の意味Role、物理単位、Target／Budget制約を次のbounded MCD Operationへ投影する。MCP aliasは同じInput／Output Schemaから`mirakan.lighting.*`を生成する。

| MCD Operation | Risk／kind | 結果 |
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

Lighting結果は`project_revision`、`contract_set_hash`、`lighting_catalog_hash`、`target_capability_hash`、`query_hash`を必須とする。Plan／Previewは`source_intent_revision`、`resolved_plan_hash`、`profile_revision`、`qualification_receipt_hashes`も返す。LightのCommit、native GPU resource／cluster buffer書込み、Shadow Technique追加、Provider activation、Project HLSLはTool listへ含めない。

Post Process DiscoveryはPost Process規約のIntent、Profile、Node Catalog、Volume、AA／Layer互換を次のbounded MCD Operationへ投影する。MCP aliasは同じInput／Output Schemaから`mirakan.post_process.*`を生成する。

| MCD Operation | Risk／kind | 結果 |
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

Post Process結果は`project_revision`、`contract_set_hash`、`post_node_catalog_hash`、`target_capability_hash`、`anti_aliasing_plan_hash`、`query_hash`を必須とする。Plan／Previewは`resolved_plan_hash`、`profile_revision`、`volume_set_hash`、`qualification_receipt_hashes`も返す。任意Render pass、native resource、history weight、Node stage並替え、Provider activation、Project Shader、Source Promotion、Project CommitはTool listへ含めない。

Lighting／Post ProcessのSearchは既定50件、最大200件、continuation付きとし、Read／Inspectはfield maskとbounded scopeを必須にする。どちらの`plan_change`も`expected_project_revision`と`idempotency_key`を必須にし、Provider OutputはInternal C++ validatorで完全再検証する。

## 21. Contract compilerのDefinition of Done

- 全MCD kindのmeta-schemaと最低1件のvalid／invalid fixtureがある。
- `game_system` kindからCatalog、Dependency Graph、State owner table、C++／TypeScript binding、conformance testを決定論的に生成する。
- Project-defined SystemがEngine Standardと同じContract validationを通り、固定WhitelistなしでCatalogへ登録できる。
- authoritative State owner欠落／重複、System dependency cycle、Presentation逆writeをinvalid fixtureで拒否する。
- `RemediationV1`のapplicable code、typed Operation template、Risk、Approval、禁止Category、適用上限を生成・検証できる。
- Duplicate key、unknown field、unbounded collection、untagged unionを拒否する。
- 同一Inputから二回生成したTree hashが一致する。
- C++／TypeScript／Cooked binary／MCP／OpenAI／Anthropic projectionが一つのMCDから生成される。
- Providerで表現できないConstraintがManifestから欠落しない。
- Provider Outputが必ずInternal validatorで再検証される。
- 全Operation errorが列挙され、string-only errorを持たない。
- `ai_mutable` Authoring fieldのtyped Operation coverageが100%で、未到達fieldを持つCapabilityのProvider projectionを拒否する。
- `AuthoringSelectionContextV1`と`WorldAuthoringContextV1`がC++／TypeScript／JSON Schema／MCPへ同じfield ID、bound、Source／Derived区分で生成される。
- World Discovery六Operationがexact revision／hash、omitted range、continuation、typed Diagnosticを返し、screen coordinate、表示row、Hierarchy pathだけのtarget指定を拒否する。
- Scene永続化owner、Level membership、Cell assignmentを別constraintとして生成し、`MoveEntityToScene`、`SetLevelSourceScenes`、Derived Cell writeを相互代替できないinvalid fixtureを持つ。
- World Capabilityの必須ExampleがProvider projectionとInternal validatorで同じdisposition、Operation ID、Diagnostic code、非変更invariantへ収束する。
- 全State machineにinvalid transition testがある。
- Cross-language round-tripとboundary fixtureが通る。
- Generated fileの直接編集をCIが検出する。
- Contract lock、Toolchain lock、Generated output hashがVerification Receiptへ記録される。
- `BuildDriverProfileV1`のvalid／invalid fixtureがあり、Makefiles、Android Multi-Config、Generator override、Driver／Target不一致を拒否する。
- Migrationなしの破壊的永続Schema変更をCIが拒否する。
- `authoring.search`／`read`／`dependencies`／`diff`がrevision、field mask、省略範囲、continuationを保持し、stale Indexと曖昧targetを拒否する。
- `systems.*`と`worlds.*`がContract／Project revision、Target、maturity、bounded resultを保持し、Activation／Commit authorityをProviderへ投影しない。
- LODの全closed enum、Domain別tagged union、enter／exit threshold、fallback、Operation errorにvalid／invalid fixtureがあり、presentation LODからauthoritative stateへ逆入力するReferenceをsemantic validatorが拒否する。
- Anti-aliasのIntent／method／sample／scope／dispositionがclosed enum／tagged unionであり、FXAA／SMAA／MSAA／Mirakan TAA／TAAUとQualified Providerの互換Predicate、fallback、history reset、Target Qualificationを同じMCDから生成する。
- `operation.rendering.aa.*`五OperationのC++／TypeScript／MCP／Provider projection、bounded result、stale revision、ambiguous／unsupported result、valid／invalid fixtureが一致し、Provider activation／Pipeline rebuild／Commit OperationをProviderへ投影しない。
- MSAA×temporal、Hybrid Deferred MSAA、未対応sample count、同一ViewFamilyのsample count混在、pixel-locked layer適用をInternal semantic validatorが同じDiagnostic code familyで拒否する。
- LightingのSource／Intent／Profile／Plan／Snapshot、物理単位tagged union、`operation.lighting.*`九Operation、bounded result、stale／lock／Target／Budget／overflowのvalid／invalid fixtureを同じMCDから生成する。
- Post ProcessのIntent／Profile／Camera Override／Volume／Node Catalog／Plan、固定stage、色空間、AA／Layer／history Predicate、`operation.post_process.*`九Operation、bounded resultのvalid／invalid fixtureを同じMCDから生成する。
- Lighting／Post ProcessのProvider projectionへCommit、native GPU resource、任意Render pass／Shader、history内部値、Capability activationを含めず、Internal validatorが未知fieldと未成熟Capabilityをfail-closedにする。

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
