# Miraikanai Engine AI可読命名・技術識別子規約

- 文書版: 1.0
- 作成日: 2026-07-20
- 最終更新日: 2026-07-20
- 対象: First-party Engine C++、C ABI、Named Modules、CMake、HLSL、Tool、Schema、Codegen、Test、Diagnostic、File／Directory
- 対象外: ゲーム作品のWorld／Level／Entity／Asset表示名、Localization本文、Third-party source
- 状態: プロジェクト公式の規範設計
- 上位文書: [Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)
- C++言語・Modules規約: [Miraikanai Engine C++23・Named Modules・`import std`移行規約](./2026-07-20-cpp23-modules-import-std-transition-design.md)
- 実行可能契約規約: [Miraikanai Engine 実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)
- AI実装規約: [Miraikanai Engine AI実装・保守ガバナンス規約](./2026-07-19-ai-engine-development-governance-design.md)
- Game Project配置・命名規約: [Miraikanai Engine AI可読Game Project配置・命名規約](./2026-07-20-ai-readable-game-project-layout-naming-design.md)

## 1. 目的

本書は、Miraikanai Engineを人間とAIが同じ意味で読み、生成し、検索し、Reviewし、自動検査できるように、Engine所有の名前と技術識別子を一意に定める。命名は装飾ではなく、型、責務、identity、lifetime、boundary、failureをSourceから復元するための契約である。

完成条件は、命名表を読めることではない。次をすべて満たすことである。

1. 同じ対象kindには一つのcase、prefix、suffix、grammarだけがある。
2. 一つの概念には一つの正規語があり、同義語をSubsystemごとに作らない。
3. Public API、Schema、Generated binding、Module、CMake target、File pathの対応を決定論的に導出できる。
4. C++識別子は`clang-tidy`、非C++識別子はrepository linterでCI拒否できる。
5. AIへ渡すNaming Policy projectionとCI判定が同じ正本から生成される。
6. 名前だけを見て、Stable ID、version-bound reference、runtime handle、index、content hashを混同しない。
7. Third-partyの命名を改変せず、Adapter境界でFirst-party規則へ変換する。

## 2. 規範の優先順位

命名規則の根拠を次の順に固定する。

1. C++、C、HLSL、CMake、TypeScript、PowerShell、JSON等の言語仕様と予約識別子。
2. C++ Core Guidelinesの一貫性、型情報を名前へ重複符号化しない原則、scopeに応じた名前長、macroだけの`ALL_CAPS`、underscore style推奨。
3. LLVM／Clang公式Toolが機械検査できる識別子分類。
4. 各言語の確立したecosystem convention。
5. 上記が一意に決めない範囲だけを本書のMiraikanai house styleで固定する。

外部Style Guideを丸ごと正本にしない。外部Guideの版更新でEngine Public APIを暗黙変更せず、本書の改訂、互換性判定、Migrationを必須とする。

参考一次資料:

- [C++ Core Guidelines: NL.5、NL.7～NL.10](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#S-naming)
- [LLVM clang-tidy: readability-identifier-naming](https://clang.llvm.org/extra/clang-tidy/checks/readability/identifier-naming.html)
- [Google C++ Style Guide: Naming](https://google.github.io/styleguide/cppguide.html#Naming)

## 3. 正規Product stem

自然言語の製品名と技術識別子を次へ固定する。

| 用途 | 正規表記 |
|---|---|
| 自然言語の製品名 | `Miraikanai Engine` |
| ASCII lowercase stem | `mirakan` |
| PascalCase stem | `Mirakan` |
| UPPER_SNAKE stem | `MIRAKAN` |

Engine所有識別子へ新しく`mira`、`miraikanai`、`mkan`、`mk`等の別stemを導入しない。公式規範文書は`mirakan`へcutover済みであり、`mira`は明示されたMigration Table、negative example、historical fixture、外部引用の入力としてだけ認識する。新規実装の互換aliasとして残さない。

| Surface | 正規形 | 例 |
|---|---|---|
| C++ root namespace | `mirakan` | `mirakan::render` |
| Public Header root | `include/mirakan/` | `include/mirakan/render/render_graph.hpp` |
| Primary Named Module | `mirakan.<domain>[.<role>]` | `mirakan.render`, `mirakan.runtime.contracts` |
| CMake alias | `mirakan::<name>` | `mirakan::foundation` |
| CMake real target | `mirakan_<name>` | `mirakan_foundation` |
| Macro／environment variable | `MIRAKAN_` | `MIRAKAN_ASSERT`, `MIRAKAN_TOOLCHAIN_ROOT` |
| C ABI type／entry prefix | `Mirakan` | `MirakanNativeGameModuleDescriptorV1` |
| CLI executable | `mirakan` | `mirakan build` |
| Git repository slug／default clone root | `mirakan-engine` | `github.com/<owner>/mirakan-engine` |
| Project manifest | `mirakan.project.json` | `mirakan.project.json` |
| MCD document suffix | `.mirakan.json` | `operation.authoring.apply_changeset.mirakan.json` |
| Content package suffix | `.mirakanpack` | `game_content.mirakanpack` |
| Engine metadata directory | `.mirakan/` | `.mirakan/journal/` |
| Schema root | `schemas/mirakan/` | `schemas/mirakan/types/` |
| Requirement／Diagnostic prefix | `MIRAKAN-` | `MIRAKAN-AI-0001` |

## 4. 共通意味規則

### 4.1 名前は責務を表す

- Typeは名詞または名詞句、functionは動詞または動詞句、bool queryはpredicate、conceptは性質または能力とする。
- 名前に実装手段だけを置かず、Domain上の責務を置く。`VectorStore`は許可できるが、責務不明な`DataProcessor`は禁止する。
- Public名は検索結果が単独でも意味を持つ具体性を持たせる。短いlocal名をPublic APIへ昇格しない。
- 名前と型へ同じ情報を重複させない。`string_name`、`ptr_resource`、`int_count`等を禁止する。
- 否定bool名を避ける。`is_disabled`より`is_enabled`を正本とし、二重否定を作らない。

### 4.2 scopeと名前長

| Scope | 規則 |
|---|---|
| Public API、Schema、Module、Artifact | Domainと役割を省略せず、repository全体検索で識別できる |
| Class／function内部 | 周辺文脈で一意なら短縮できる |
| 数式、loop、座標成分 | `i`、`j`、`x`、`y`、`z`、`t`を限定scopeで許可する |
| Global／namespace scope | 一文字名、一般名詞だけの名前を禁止する |

文字数の一律下限／上限で意味を代替しない。ただしPublic technical IDのsegment長とSchema上限は各grammarで機械制限する。

### 4.3 禁止する曖昧語

次の語を単独のType、namespace、Directory、Module、target、File stemとして禁止する。

```text
base
common
data
general
global
helper
info
manager
misc
object
shared
stuff
temp
thing
util
utils
```

語がDomain上の正規概念である場合だけ複合語として許可できる。例として`AssetMetadata`は契約上のmetadataを表すため許可し、責務を示さない`RenderData`は拒否する。例外はNaming Exception Manifestへrule、fully-qualified symbol、理由、owner、失効条件を登録する。

### 4.4 boolとpredicate

| 意味 | Prefix | 例 |
|---|---|---|
| 現在状態 | `is_` | `is_visible` |
| 所有／包含 | `has_` | `has_pending_work` |
| 能力／許可 | `can_` | `can_submit` |
| Policy判断 | `should_` | `should_retry` |
| Requirement | `requires_` | `requires_restart` |
| Availability | `supports_` | `supports_ray_tracing` |

`enabled`のようなSchema fieldはclosed contextで許可する。Function predicateは`is_enabled()`とし、`check_enabled()`、`get_enabled()`、`enabled_flag()`を使わない。

### 4.5 単位、space、lifetime

- Public境界は`Duration`、`Radians`、`WorldPosition`等のsemantic typeを優先する。
- Wire、C ABI、OS API、測定値等でprimitiveが必要な境界だけ、`timeout_ms`、`size_bytes`、`frequency_hz`のように単位suffixを付ける。
- `position`、`direction`、`transform`等はspaceが型から判別できなければ`world_position`、`local_direction`のように明示する。
- ownership、lease、generationは型で表現する。`owned_ptr`等のHungarian notationへ戻さない。

## 5. C++23命名

### 5.1 識別子case

| 対象 | 規則 | 例 |
|---|---|---|
| Root／subnamespace | lowercase `snake_case` | `mirakan::render_graph` |
| Class、struct、union | `PascalCase` | `RenderGraph`, `FramePacket` |
| Enum type | `PascalCase` | `QueueType` |
| Type alias | `PascalCase` | `ResourceId` |
| Concept | `PascalCase` | `Renderable`, `ContiguousRange` |
| Function、method | lowercase `snake_case` | `compile_graph()` |
| Local variable、parameter | lowercase `snake_case` | `frame_index` |
| Public struct field | lowercase `snake_case` | `frame_count` |
| Private／protected data member | lowercase `snake_case`＋末尾`_` | `frame_index_` |
| Program-lifetime constant | `k`＋`PascalCase` | `kInvalidIndex` |
| Local invocation-dependent `const` | lowercase `snake_case` | `combined_name` |
| Scoped enum value | `PascalCase` | `QueueType::Compute` |
| Type template parameter | `PascalCase` | `Resource`, `Allocator` |
| Value template parameter | `k`＋`PascalCase` | `kExtent`, `kAlignment` |
| Macro | `MIRAKAN_`＋`UPPER_SNAKE_CASE` | `MIRAKAN_ASSERT` |

Classとstructでdata memberのsuffix意味を変える。Classはinvariantを隠すため全data memberをprivateまたはprotectedとし末尾`_`を付ける。Structは不変条件を持たないrecordとしてpublic fieldを持ち、末尾`_`を付けない。

Namespace-scope mutable global variableをFirst-party codeで禁止する。許可済みC ABI symbol、constant、immutable registry descriptor以外へglobal命名規則を設けない。

### 5.2 関数の意味

| Verb／形 | 契約 |
|---|---|
| `make_*` | Process-local value／ownerを構築する。永続登録やI/Oを暗黙実行しない |
| `create_*` | identity発行またはowned state登録を伴う |
| `find_*` | 副作用なし。未発見は正常なoptional結果 |
| `read_*` | 既存sourceからbounded dataを読む |
| `load_*` | I/O、decode、dependency解決を含み得るためfailureを返す |
| `resolve_*` | intent／reference／policyを一つのcanonical resultへ解決する |
| `validate_*` | Sourceを変更せずDiagnosticを返す |
| `compile_*` | Source contractからportable derived representationを生成する |
| `cook_*` | Target、profile、dependencyを固定した配布用Artifactを生成する |
| `enqueue_*` | 既存queueへworkを追加する |
| `schedule_*` | 実行時刻、phase、dependencyを決める |
| `submit_*` | 所有境界またはBackend境界へworkを渡す |
| `on_*` | Event callback entryだけに使用する |
| `set_*` | 既存fieldの明示mutation。transaction／validationを迂回しない |
| `to_*` | 意味を保つ表現変換 |
| `parse_*` | text／bytesからtyped valueを検証付きで生成する |

Accessorは`frame_index()`、mutatorは`set_frame_index()`とする。単純accessorへ`get_`を付けない。`try_`、`do_`、`process_`、`handle_`、`run_`を責務の代わりに使わず、failureと副作用を`Result`、型、具体的動詞で表す。Protocol handler等で`handle_*`がDomain上の正確な役割である場合は、対象を必ず続けて`handle_device_lost_event()`のようにする。

### 5.3 Interface、ownership、version

- Interfaceへ`I` prefixを付けず、役割名を使う。例: `RendererBackend`。
- Abstract implementationを`Base` suffixだけで表さない。共有実装が必要なら責務名を付け、単なる継承都合のbase classを作らない。
- RAII owner、view、lease、handleは型名で区別する。例: `ImageOwner`、`ImageView`、`ImageLease`、`ImageHandle`。
- Public Schema／ABI versionを持つ型だけ`V1` suffixを使う。通常のC++実装型へ将来予測の`V2`を付けない。
- Error型は`<Domain>Error`、Diagnostic payloadは`<Domain>Diagnostic`、Result aliasは共通`Result<T>`を使う。

### 5.4 略語

略語はNaming Policyのallowlistへ登録し、PascalCaseでは通常語としてcase変換する。

| 正規語 | PascalCase | snake_case | UPPER_SNAKE_CASE |
|---|---|---|---|
| artificial intelligence | `Ai` | `ai` | `AI` |
| application programming interface | `Api` | `api` | `API` |
| application binary interface | `Abi` | `abi` | `ABI` |
| central processing unit | `Cpu` | `cpu` | `CPU` |
| graphics processing unit | `Gpu` | `gpu` | `GPU` |
| identifier | `Id` | `id` | `ID` |
| uniform resource locator | `Url` | `url` | `URL` |
| universally unique identifier | `Uuid` | `uuid` | `UUID` |
| user interface | `Ui` | `ui` | `UI` |
| JavaScript Object Notation | `Json` | `json` | `JSON` |
| temporal anti-aliasing | `Taa` | `taa` | `TAA` |
| temporal anti-aliasing upsampling | `Taau` | `taau` | `TAAU` |

例は`GpuResourceId`、`parse_json()`、`ui_root_id`、`MIRAKAN_GPU_VALIDATION`である。`GPUResourceID`、`parseJSON()`、`ui_root_identifier`を混在させない。Vendorが定めた正式商標、C ABI field、OS symbolはAdapter内で原表記を保持できる。

## 6. Identityと参照語彙

次のsuffixはcase上の装飾ではなく、永続性と比較可能範囲を表す。

| Suffix | 意味 | 永続化 |
|---|---|---|
| `_id` | owner範囲で一意なidentity。`StableId`ならProjectを越える契約に従う | 契約で許可 |
| `_ref` | IDだけでなくversion、revision、hash等を固定したtyped reference | 許可 |
| `_handle` | generation付きruntime lookup token | 原則禁止 |
| `_index` | 一つのcollection／Artifact内の位置 | ownerと順序契約がある場合だけ |
| `_key` | map、sort、cacheの比較規則を持つkey | 契約次第 |
| `_hash` | bytesまたはcanonical valueのintegrity digest | 許可 |
| `_revision` | 同じidentityの単調なsource revision | 許可 |
| `_version` | Schema／Contractの意味version | 許可 |
| `_instance_id` | Source identityとは別のinstance identity | scopeを明記 |

`asset_id`をruntime handleへ、`resource_index`を永続Asset IDへ、`content_hash`をmutable identityへ流用しない。名前に`id`が付く値は型、bit幅、発行者、scope、0／null規則、再利用、永続性をSchemaまたはType contractに必ず持つ。

## 7. File、Directory、Module

### 7.1 FileとDirectory

Repository／Game Projectの物理・論理構造を指す規範語は`Directory`とする。`Folder`はOS API／UI label／外部引用の正式語に限り、別の構造概念として扱わない。

| 対象 | 規則 | 例 |
|---|---|---|
| Directory | lowercase `snake_case` | `render_graph/` |
| Private implementation root | `source/` | `engine/rendering/source/` |
| C++ Header | lowercase `snake_case.hpp` | `render_graph.hpp` |
| C++ implementation | lowercase `snake_case.cpp` | `render_graph.cpp` |
| Primary Module interface | Primary Module名＋`.cppm` | `mirakan.render.cppm` |
| Module partition | lowercase `snake_case.cppm` | `resource_registry.cppm` |
| HLSL | lowercase `snake_case.hlsl` | `visibility_buffer.hlsl` |
| CMake include | lowercase `snake_case.cmake` | `compiler_policy.cmake` |
| First-party executable／library artifact | lowercase `snake_case`＋Platform extension | `mirakan_game_host.exe`, `mirakan_runtime.dll` |
| Test | `<subject>_test.cpp` | `render_graph_test.cpp` |
| Benchmark | `<subject>_benchmark.cpp` | `render_graph_benchmark.cpp` |
| Fuzz target | `<subject>_fuzz.cpp` | `contract_decoder_fuzz.cpp` |
| Generated source | `<subject>.generated.<ext>` | `render_contracts.generated.hpp` |

First-party componentのprivate implementation rootは`source/`へ固定し、repository所有Pathへ`src/`を新設しない。外部Repository URL、Vendor subtree、Platform SDKが要求するPathはpath-scoped exceptionとする。

`CMakeLists.txt`、`AppxManifest.xml`等のTool／Platform必須名、Platform SDK必須名、license file、Third-party binary basenameは例外とする。`new_`、`old_`、`final_`、`copy_`、日付、担当者名をSource file versioningへ使わない。Version管理はGitとSchema versionで行う。

Platform／Backend suffixは実際に実装が分かれるprivate sourceだけへ使用する。

```text
swap_chain_windows.cpp
surface_android.cpp
command_queue_d3d12.cpp
command_queue_vulkan.cpp
command_queue_metal.mm
```

Public contract fileへPlatform suffixを付けてPublic APIを分岐させない。

### 7.2 Namespace、Module、target、pathの対応

一つの公開componentは次を一対一で所有する。

```text
Directory:       engine/runtime/contracts/
CMake target:    mirakan_runtime_contracts
CMake alias:     mirakan::runtime_contracts
Primary Module:  mirakan.runtime.contracts
C++ namespace:   mirakan::runtime::contracts
Header root:     include/mirakan/runtime/contracts/
```

Module名はlowercase ASCIIとdot区切りの`mirakan.<domain>[.<role>]`、partitionは`mirakan.<primary>:<cohesive_contract>`とする。`common`、`shared`、`utils`、`detail`を公開Module／namespaceへ使わない。Module-private実装だけ`detail` namespaceを許可するが、export、Diagnostic、Artifact、Schemaへ露出しない。

## 8. C ABI、CMake、HLSL、Tool言語

### 8.1 C ABI

- Exported type、enum、functionは`Mirakan` prefix＋`PascalCase`を使う。
- ABI fieldはlowercase `snake_case`とする。
- ABI versionはtype／entry point末尾の`V1`で明示する。
- Macroとcompile definitionは`MIRAKAN_` prefixを使う。
- Reserved fieldは`reserved`＋連番ではなく、size付きstructとversion negotiationを優先する。

```c
typedef struct MirakanNativeGameModuleDescriptorV1 {
    uint32_t struct_size;
    uint32_t abi_version;
    MirakanStableId module_id;
} MirakanNativeGameModuleDescriptorV1;

MirakanStatus MirakanGetNativeGameModuleV1(
    const MirakanNativeHostApiV1* host_api,
    MirakanNativeGameModuleDescriptorV1* out_module);
```

### 8.2 CMake

| 対象 | 規則 | 例 |
|---|---|---|
| Project-owned command | `mirakan_`＋lowercase snake | `mirakan_add_cpp_component()` |
| Real target | `mirakan_`＋lowercase snake | `mirakan_render_core` |
| Public alias | `mirakan::`＋lowercase snake | `mirakan::render_core` |
| Cache variable／option | `MIRAKAN_`＋UPPER_SNAKE | `MIRAKAN_ENABLE_ASAN` |
| Local variable | lowercase snake | `module_name` |
| Preset／profile ID | lowercase snake＋必要なversion | `windows_desktop_v1` |

Builtin CMake command／variableは原表記を保持する。Third-party optionをMirakan名へ複製せず、dependency Adapterで明示mappingする。

### 8.3 HLSL

- Struct、enum、type aliasは`PascalCase`。
- Function、resource、variable、parameterはlowercase `snake_case`。
- Entry pointはstageを明示した`vs_main`、`ps_main`、`cs_main`、`ms_main`、`as_main`等とする。
- Engine macroは`MIRAKAN_`＋`UPPER_SNAKE_CASE`。
- HLSL semantic、register、Vendor intrinsicは公式表記を保持する。
- C++／HLSL共有fieldはSchemaから生成し、手作業でcase変換表を持たない。

### 8.4 TypeScript、JSON、CLI、PowerShell

- TypeScriptのclass、type、interface、enum、type parameterは`PascalCase`、function、method、variable、parameterは`camelCase`とする。
- TypeScript private fieldは`#camelCase`を使い、`_private` prefixを作らない。
- JSON、YAML、TOML、MCD fieldとstring-backed enum valueはlowercase `snake_case`とする。
- Serialized fieldはTypeScript conventionへrenameせず、generated codecがwire名とのmappingを所有する。
- CLI subcommandはlowercase `snake_case`ではなく一般的な単語、optionは`--kebab-case`とする。例: `mirakan build --target-profile windows_desktop_v1`。
- Environment variableは`MIRAKAN_`＋`UPPER_SNAKE_CASE`とする。
- PowerShell公開functionはapproved verb＋`Mirakan` nounを使う。例: `Invoke-MirakanBuild`。

## 9. Contract、Schema、Diagnostic

### 9.1 MCD ID

非Requirement Contract IDは次のgrammarを使う。

```text
<kind>.<namespace_path>
```

`namespace_path`は2～8個のdot区切りsegment、各segmentはASCII lowercase、先頭英字、以後英数字またはunderscore、1～48文字とする。File名はIDへ`.mirakan.json`を付け、略称や別pathを作らない。

```text
operation.authoring.apply_changeset
capability.render.material.toon_v1
game_system.engine.combat
operation.authoring.apply_changeset.mirakan.json
```

ContractのkindはID先頭にすでに存在するため、`mirakan.operation.*`のようにProduct stemを重複させない。Schema root、file suffix、repository scopeがProduct ownershipを表す。

### 9.2 RequirementとDiagnostic

Requirement IDは`MIRAKAN-<DOMAIN>-<4桁以上の番号>`、Diagnostic codeは`MIRAKAN-<DOMAIN>-<CONDITION>`とする。

```text
MIRAKAN-AI-0001
MIRAKAN-NAMING-INVALID_CASE
MIRAKAN-ASSET-AMBIGUOUS_REFERENCE
```

- Domainは登録済みUPPER_SNAKE tokenとする。
- Conditionは原因または違反を表し、表示文を埋め込まない。
- `ERROR`、`FAILED`、`INVALID`だけのconditionを禁止する。
- Diagnostic messageはLocalization keyとtyped fieldを持ち、codeを人間向け文章として使わない。
- 一度Release artifact、Save、Receipt、外部APIへ出たID／codeをrenameまたは再利用しない。置換は新ID＋Migrationを使う。

### 9.3 Schema field

- Fieldはlowercase `snake_case`。
- Collectionは複数形、countは`*_count`、byte sizeは`*_size_bytes`とする。
- Timestampは基準を型で固定し、境界表現は`*_utc`、durationは単位型または`*_duration_ns`を使う。
- Nullable fieldへ`maybe_`、`optional_` prefixを付けず、Schemaのpresenceで表す。
- Enum valueはlowercase `snake_case`。`unknown`を暗黙fallbackにせず、契約上必要な場合だけclosed valueとして定義する。

## 10. Test、Fixture、Benchmark

Test case名は`<subject>_<condition>_<expected_result>`とする。

```text
handle_stale_generation_is_rejected
render_graph_with_cycle_reports_dependency_cycle
asset_move_preserves_stable_id
```

- `test1`、`works`、`basic`、`happy_path`だけの名前を禁止する。
- Failure testは期待するDiagnosticまたは状態を名前に含める。
- Property testは`<property>_holds_for_<domain>`、round-tripは`<format>_round_trip_preserves_<semantic>`とする。
- Benchmarkは対象、scale、metricを分離してmetadataへ持たせ、名前へ測定値を埋め込まない。
- Fixture IDはlowercase `snake_case`＋必要なversion、Directoryは`valid/`、`invalid/`、`golden/`へ分類する。

## 11. Generated codeと外部境界

- Generated symbolも手書きSourceと同じ公開命名規則に従う。
- `Generated`、`Auto`、`Gen`を全symbolへ機械的に付けない。生成物であることはpath、manifest、file suffixで示す。
- Generatorはcanonical Contract IDからsymbol、file、module、namespaceを決定論的に導出し、任意省略語を作らない。
- 生成時の衝突は連番suffixで回避せず、Contract側の名前衝突として拒否する。
- Third-party symbolはVendor原表記を保持し、First-party Public APIへ再exportしない。
- AdapterでVendor型をFirst-party型へ変換し、同じscopeにVendor styleとMirakan styleを混在させない。

## 12. AI可読Naming Policy

### 12.1 機械可読正本

Phase N1で`config/naming_policy.toml`を追加し、次をversion付きで保持する。

```text
policy_version
product_stem
identifier_kind_rules
approved_abbreviations
canonical_terms
forbidden_terms
boolean_prefixes
semantic_suffixes
module_roles
platform_backend_suffixes
diagnostic_domains
external_exemptions
```

本書が意味規範、`config/naming_policy.toml`が機械可読projection、repository rootの`.clang-tidy`と非C++ naming linterが実行projectionである。CIは三者のversionとhashを照合し、手動で別々に進化させない。

### 12.2 AIへの提示

AIへrepository全文の命名文書を毎回渡さず、Task scopeに必要な次のbounded projectionを渡す。

- 対象言語とidentifier kindのcase表。
- 使用可能namespace、Module、CMake target。
- Taskに出現するcanonical termと略語。
- identity／reference suffix契約。
- 禁止語と該当する正しい代替。
- valid／invalid example。
- Policy versionとcontent hash。

AIは未知略語、未知Module role、未知Diagnostic domainを推測せず、Catalog queryまたはNaming Diagnosticを返す。Caseだけの違反は安全なautofix候補にできるが、意味語、identity suffix、単位、ownershipの違反を文字置換で自動修復しない。

## 13. 自動検査

### 13.1 C++／HLSL

`.clang-tidy`の`readability-identifier-naming`で少なくともNamespace、Class、Struct、Union、Enum、ScopedEnumConstant、TypeAlias、Concept、Function、Method、Parameter、LocalVariable、PrivateMember、ProtectedMember、Constant、TemplateParameter、MacroDefinitionを設定する。

追加AST checkで次を検査する。

- bool function／variableのpredicate prefix。
- Program-lifetime constantとlocal invocation-dependent constの区別。
- Public APIの禁止語、未登録略語、曖昧なidentity suffix。
- Interfaceの`I` prefix、Hungarian notation、予約識別子。
- Global mutable variable。
- Unit／spaceを要求するboundary primitive。

HLSLはDXC compileに加えてtoken／reflection based naming lintを行い、semanticとVendor intrinsicをexemption Catalogから除外する。

### 13.2 Repository全体

Repository linterは次を検査する。

- File／Directory、Module、partition、namespace、CMake target、include rootの一対一対応。
- CMake command、option、preset、profile ID。
- MCD ID、Requirement ID、Diagnostic code、Schema field、enum value。
- Test／Benchmark／Fixture名。
- `mira`等のlegacy stem、禁止語、未登録略語。
- Generated file manifestとsymbol mapping。
- Windows case-fold、Unicode NFC、reserved filename衝突。

Third-party、external evidence、historical migration fixture、引用はpath-scoped exemptionとする。Repository全体の無差別置換を行わない。

### 13.3 例外

Naming exceptionは次をすべて持つ。

```text
rule_id
exact_symbol_or_path
external_owner_or_internal_owner
rationale
introduced_revision
review_by_revision_or_removal_condition
```

Wildcard exception、Directory全体の`NOLINT`、期限のない「legacy」理由を禁止する。Generated／Vendor exemptionもmanifest hashとscopeを固定する。

## 14. 移行計画

### Phase N0: 規範固定と文書cutover（完了）

- 本書を命名の正本にする。
- Product technical stemを`mirakan`へ固定する。
- Game作品側の表示名規則を本書から分離する。
- 公式Review setの規範例、Path、technical stemを`mirakan`へ統一する。
- `mira`表記をMigration Table、negative example、historical fixture、外部引用へ限定する。

### Phase N1: Policyとrename map

- `config/naming_policy.toml`、Naming Exception Manifest、positive／negative fixtureを追加する。
- 実装、Schema、generated artifact、scaffoldに残る`mira`から`mirakan`へのrename mapをsurface別に作る。
- External citation、自然言語製品名、Contract ID、File suffix、ABI symbolを分類し、blind replacementを禁止する。
- `.clang-tidy`とrepository naming linterを追加する。

### Phase N2: Atomic prefix cutover

- Repository slug／default clone root、Namespace、include root、Named Module、CMake alias／target、macro、C ABI、Schema root、MCD suffix、Content package suffix、manifest、metadata Directory、Requirement／Diagnostic prefixを同じ実装ChangeSetで切り替える。
- Hosting repository、local clone、CI、badge、bootstrap、submodule／dependency参照を`mirakan-engine`へ同時更新し、旧URL redirectを正規参照または恒久aliasとして使わない。
- Pre-1.0で実装前のため、`mira` compatibility namespace、Module alias、Header forwarding、Diagnostic aliasを残さない。
- Generated golden、example、fixtureを同じPolicy versionで再生成する。

### Phase N3: Scaffold gate

- Engine component scaffoldはNaming Policyからpath、Module、target、namespaceを生成する。
- 不正名ではFile作成前にDiagnosticを返す。
- First-party source、generated source、AI proposalの全経路で同じlintを実行する。

### Phase N4: Vertical slice qualification

- Foundation、Math、Runtime Contracts、Renderingの最小縦切りを新規則だけで実装する。
- C++、C ABI、HLSL、MCD、CMake、Testのcross-surface name mappingを検証する。
- 人間実装、AI生成、Codegenの三経路で同じDiagnosticと修正候補を得る。

## 15. Qualification Gate

実装開始前の命名Gateを次へ固定する。

1. First-party technical identifierの100%がNaming Policyのkindへ分類される。
2. Engine-owned active Source／Schema／Build定義に未許可の`mira` stemが0件である。
3. Public APIに未登録略語、禁止曖昧語、Hungarian notation、`I` interface prefixが0件である。
4. Module、namespace、target、path対応の不一致が0件である。
5. Stable ID、ref、handle、index、key、hashの誤用negative fixtureをすべて拒否する。
6. `clang-tidy`とrepository linterのpositive／negative fixtureが全Targetで同じ結果になる。
7. AIへNaming Policy projectionを渡した3回の独立生成で、case違反、legacy stem、架空略語、曖昧identity suffixが最終提出0件である。
8. Third-party／Vendor symbolがFirst-party Public APIへ漏出しない。
9. Naming exceptionがexact scope、owner、理由、失効条件を持つ。
10. 本書、Naming Policy、`.clang-tidy`、linterのPolicy version／hashが一致する。

## 16. 正例と反例

| 正例 | 反例 | 理由 |
|---|---|---|
| `mirakan::render::RenderGraph` | `mira::render::RenderManager` | legacy stemと曖昧な`Manager` |
| `compile_graph()` | `ProcessData()` | caseと責務が不明 |
| `frame_index_` | `m_nFrame` | Hungarian notation |
| `is_device_lost()` | `device_lost_flag()` | predicateとして読めない |
| `GpuResourceId` | `GPUResourceID` | 略語caseが不統一 |
| `timeout_ms` | `timeout` | primitive境界で単位不明 |
| `AssetRef` | `asset_id`にversionを暗黙付加 | identityとreferenceの混同 |
| `mirakan.runtime.contracts` | `mirakan.common` | 責務不明Module |
| `MIRAKAN-ASSET-MISSING_DEPENDENCY` | `MIRAKAN-ASSET-ERROR` | 原因を特定できない |
| `asset_move_preserves_stable_id` | `asset_test_1` | 条件と期待結果がない |

## 17. 明示的に採用しない方式

- 全識別子を同じcaseへ揃え、kind情報を失う方式。
- Unreal、Unity、Godot、Vendor SDKのprefix／suffixを模倣する方式。
- Interfaceの`I` prefix、pointerの`p`、memberの`m_`等のHungarian notation。
- `Manager`、`Helper`、`Util`を生成時のdefault名にする方式。
- 名前、path、array index、content hashを永続identityとして使う方式。
- AIが自由に略語、Module role、Diagnostic domainを生成する方式。
- Reviewだけに依存し、自動検査とnegative fixtureを持たない方式。
- 実装前からlegacy aliasと複数stemを維持する方式。

## 18. 実装順

1. 本書、基盤規約、Game Project配置・命名規約、公式Review setの決定権と規範表記を整合させる（完了）。
2. 実装対象のLegacy inventoryとsurface別rename mapを作る。
3. Naming Policy Schema、TOML、exception manifest、fixtureを実装する。
4. `.clang-tidy`のidentifier naming projectionを実装する。
5. Repository naming linterを実装する。
6. `mirakan` prefixを一つのatomic ChangeSetでSchema、source、generated artifact、scaffoldへ反映する。
7. Component scaffoldとCodegenをNaming Policyへ接続する。
8. Foundation縦切りでQualification Gateを通す。

この順序を入れ替えて、先に大量renameまたはEngine実装を開始しない。
