# Miraikanai Engine Naming／Project Layout

- 文書ID: mirakan.arch.naming-project-layout
- 状態: review
- 正本範囲: 共通語彙、acronym、public型・Operation・Diagnostic・file・directory命名、Engine／Game Project root、Source／Derived／Intermediate／Package配置、generated file、module／namespace／target対応、lint／migration Gate
- 非正本範囲: 型・Schemaの構造、外部Tool version、Build Driver、Project revision、Asset lifecycle、Domain固有field。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Core architecture](core-architecture.md)、[Toolchain／Dependencies](toolchain-dependencies.md)、[Executable contracts](executable-contracts.md)、[C++23 modules](cpp23-modules.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論と優先順位

名前、Path、Stable ID、表示名を分離し、同じ意味へ一つの正規語を使う。技術識別子は人間とAIが検索、推論、生成、reviewできる明示性を優先し、省略、文脈依存alias、同義語の併存を避ける。

規則の優先順位は、MCDまたはDomain Ownerが定める公開Contract、本書、対象言語／Platform規則、局所Styleの順である。競合時は上位を採用し、例外を新しい暗黙規則にしない。

Projectの正式stemは次とする。

| 用途 | 正規形 |
|---|---|
| Product／表示 | `Miraikanai Engine` |
| C++ root namespace | `mirakan` |
| C++ module prefix | `mirakan` |
| C ABI prefix | `mirakan_` |
| CMake target prefix | `mirakan_` |
| Stable ID prefix | `mirakan.` |
| Environment／compile macro prefix | `MIRAKAN_` |

`miraikanai`、`mirai_kanai`、`mkn`、`mk`を技術識別子の別stemとして導入しない。

## 2. Vocabularyとacronym

同じ概念へ複数語を割り当てない。少なくとも次を正規語とする。

| 正規語 | 意味 | 混同しない語 |
|---|---|---|
| Project | Userが編集・CommitするGame制作単位 | Engine repository、Workspace |
| World | Runtime／Authoringの空間的root identity | Level、Scene、Map |
| Level | Worldへ配置・streamするAuthoring単位 | Worldそのもの |
| Asset | SourceからCooked Artifactまで追跡される内容identity | arbitrary file、Runtime resource |
| Artifact | Build／Cookが生成するimmutable出力 | Source document |
| Source | Userまたは承認済みToolが編集する正規入力 | Derived、cache |
| Derived | Sourceから再生成可能な内容出力 | Commit対象Source |
| Intermediate | 一回のBuild／Import中だけ有効な一時出力 | Package |
| Package | Target向けに検証・封入された配布候補 | Build tree |
| Operation | 認可・検証される型付き要求 | arbitrary command、action callback |
| Diagnostic | Stable IDと構造化contextを持つ結果 | free-form logだけ |
| Adapter | 外部／Platform APIをprivateに包む実装 | Domain public contract |
| Gateway | 認可、検証、Commitを強制する唯一の入口 | pass-through helper |

Acronymは語として読むものも含め、公開PascalCaseでは`Id`、`Api`、`Gpu`、`Cpu`、`Ui`、`Url`、`Json`、`Rpc`、`Sdk`、`Abi`を使う。全大文字連結を避ける。例: `AssetId`、`GpuBudget`、`JsonValue`。C ABI、macro、規格上の名称、Vendor名称はその規格表記を維持できる。

Booleanは肯定形の状態またはCapabilityを表し、`is_`／`has_`／`can_`を用いる。二重否定、`disable_* = false`、意味不明な`flag`を禁止する。単位を持つ数値は`timeout_ms`、`size_bytes`、`frequency_hz`のように単位suffixを付ける。

## 3. Public type、operation、diagnostic

### 3.1 C++とSchema type

| 対象 | 規則 | 例 |
|---|---|---|
| C++ public type | PascalCase、役割を名詞で表す | `ProjectRevision`、`AssetHandle` |
| C++ function／method | snake_case、動詞から始める | `resolve_asset`、`submit_change_set` |
| local／parameter／field | snake_case | `expected_revision` |
| constant／enumerator | PascalCase | `UnsupportedCapability` |
| concept | PascalCase、必要なら能力語 | `SerializableValue` |
| private implementation | 意味名＋`Impl`を乱用せず具体的役割 | `D3d12DeviceAdapter` |

`Manager`、`Helper`、`Util`、`Common`、`Misc`、`Data`、`Info`だけで責務を表す型を禁止する。型名へownershipを表すときは実際のlifetime契約と一致させ、`Shared`や`Ref`を曖昧な装飾として使わない。

Schema typeはPascalCase＋major suffixを用い、suffixなし最新版aliasを作らない。例: `ProjectChangeSetV1`。Fieldはlowercase snake_case、closed enum valueもlowercase snake_caseとする。構造の正本は[Executable contracts](executable-contracts.md)であり、本書は綴りだけを所有する。

### 3.2 Stable IDとOperation

Stable IDはlowercase ASCII dot-separated segmentとし、各segmentはlowercase snake_caseを使う。

```text
mirakan.<kind>.<domain>.<specific_name>.v<major>
```

Operation IDは`operation.<domain>.<resource>.<verb>`、Diagnostic IDは`diagnostic.<domain>.<condition>`を基本形とする。動詞は`get`、`list`、`resolve`、`validate`、`plan_change`、`preview`、`commit`など意味を限定し、`do`、`run`、`process`、`handle`を単独で使わない。

Operationの入力／出力型名は`<Intent>RequestV1`、`<Intent>ResultV1`のように役割を明示する。Provider表示名、localized message、Editor labelをStable IDとして使わない。

### 3.3 Diagnostic

Diagnostic IDは原因または拒否条件を表し、severityや翻訳文をIDへ埋め込まない。例:

```text
diagnostic.project.stale_revision
diagnostic.asset.invalid_source_path
diagnostic.build.toolchain_lock_mismatch
```

Messageはlocalized projectionでありidentityではない。動的値、path、UUID、versionをDiagnostic IDへ連結しない。Error codeの構造とcontext fieldはDomain Ownerまたは[Executable contracts](executable-contracts.md)が決める。

## 4. Fileとdirectory naming

| 対象 | 正規形 |
|---|---|
| Architecture Markdown | lowercase kebab-case `.md`、日付なし。Decisionだけ日付prefixを許可 |
| C++ source／header／module | lowercase snake_case、`.cpp`／`.hpp`／`.cppm` |
| Directory | lowercase snake_case。文書分類DirectoryはIndex規約でnumeric prefixを許可 |
| Schema／JSON | lowercase snake_case、型を表すfileはmajor suffixを含める |
| CMake module | lowercase snake_case `.cmake` |
| Test source | 対象名＋`_test` |
| Fixture | intentを表す名前＋`_valid`／`_invalid`等の期待結果 |
| Benchmark | 対象名＋`_benchmark` |
| Generated file | generator所有root内で正規type／projection名から決定 |

Caseだけが異なる名前、末尾space／dot、reserved device name、Unicode normalizationが異なる同一視名を拒否する。Path segmentはportableなlowercase ASCIIを原則とし、User表示名と物理Pathを分離する。

### 4.1 禁止

- `NewFolder`、`test2`、`final_final`、`misc`、`common`、`shared`、巨大な`utils`
- `FooManager`、`DataInfo`、`DoThing`のように責務を示さない名前
- `assetId`と`asset_id`、`ID`と`Id`、`colour`と`color`の混在
- date付き恒久仕様名、versionなしSchema型、拡張子違いで意味を区別するalias
- display name、absolute path、配列indexをStable IDにすること

### 4.2 許可

- Domainで意味が確立した`core`、`runtime`、`contracts`、`backends`
- 同じstemでもrootと型が異なり役割が明示されるもの
- 外部Format、Platform API、規格が要求するcase／FilenameをAdapterまたはPackage境界に隔離すること

## 5. Engine rootとGame Project root

Engine repository rootは[Core architecture](core-architecture.md)のRepository境界を正本とする。Game ProjectをEngine source treeの下へ入れず、Projectごとの独立rootとして扱う。

```text
<game-project>/
├─ mirakan.project.json
├─ source/
│  ├─ assets/
│  ├─ worlds/
│  ├─ gameplay/
│  ├─ ui/
│  ├─ localization/
│  └─ native/
├─ config/
├─ packages/
├─ derived/
├─ intermediate/
├─ staging/
└─ evidence/
```

Project root discoveryは`mirakan.project.json`の存在とschema validationで行う。Current working directory、Editor executable位置、親Directory名から推測しない。Project manifest内のProject IDとdisplay nameを物理Directory名から独立させる。

## 6. Source、Derived、Intermediate、Package

| Root | 所有内容 | Git／配布 | 書込み権限 |
|---|---|---|---|
| `source/` | Userまたは承認済みOperationが編集する正規入力 | 原則追跡 | Project ChangeSet Gatewayだけ |
| `config/` | Project選択とTarget別設定 | 追跡 | typed configuration Operationだけ |
| `derived/` | Source＋Toolchainから再生成可能なimport／cook出力 | 追跡しない | Build／Content Gateway |
| `intermediate/` | 一回のtask、compile、import中の一時file | 追跡・配布しない | task-scoped Worker |
| `packages/` | 検証済みTarget package候補とmanifest | sourceとして追跡しない | Packaging Gateway |
| `staging/` | AI／Import候補、Preview、未Commit proposal | 追跡・Runtime読込しない | 隔離Worker／Proposal Gateway |
| `evidence/` | Project変更に結び付くReceipt／Provenance参照 | Policyに従い追跡 | Evidence writer |

Sourceへcache、object、BMI、downloaded Dependency、generated Provider Schemaを置かない。DerivedとIntermediateをSourceとして再importせず、Packageを編集入力に戻さない。削除可能性は分類で決まり、Filename prefixや拡張子では推測しない。

## 7. AssetとAuthoring document配置

`source/assets/<domain>/`は外部Source Assetとimport設定を、`source/worlds/`はWorld／Level Authoring documentを、`source/gameplay/`はGameplayDefinitionを、`source/ui/`と`source/localization/`は各Domain sourceを所有する。

Asset identityはCatalogのStable IDであり、Filenameは人間向けのlocatorである。RenameやMoveでidentityを変更しない。Content hashはdeduplicationとIntegrityに使うが、User intentを表すidentityにしない。

同一Pathへcase違い、Unicode正規化違い、拡張子だけ異なる衝突を作らない。Import前に正規Path、Asset ID、content hash、source provenanceを検査し、衝突時は自動suffixで隠さずtyped Diagnosticを返す。

### 7.1 Provenanceと権利

外部取得、User提供、AI生成を問わず、Source候補はorigin URIまたは生成Operation、取得／生成時刻、content hash、宣言license、権利確認状態、生成Model／ToolのReceipt参照を持つ。licenseまたは利用権が不明な候補を`source/`へCommitせず、`staging/`で`diagnostic.asset.rights_unverified`として停止する。

AI生成物は生成したという事実だけで権利確認済みにならない。Provider規約、入力Assetの権利、出力利用条件をEvidenceへ結び付け、Approval後にProject ChangeSetでSourceへ昇格する。Provenance recordの構造と保持PolicyはGovernance／Asset Ownerが決定し、本書は配置と命名だけを決める。

## 8. Native Game Code

Project固有native codeは`source/native/`のNative Game Module単位へ置き、Engine private headerやVendor SDKへ直接依存しない。Module manifest、公開generated C ABI、source、testを同じmodule rootへまとめる。Build tree、generated binding、object、symbol、packageはProject `derived/`、`intermediate/`、`packages/`へ分離する。

Native targetとmodule名はProject display nameから自動生成せず、validated ASCII Stable IDから決定する。Engine target prefixと衝突する名前、Platform予約名、任意absolute output pathを拒否する。

## 9. Generated-file policy

Generated fileはgenerator ID、generator version binding、input artifact hashes、contract set hash、Target Profile、output hashを持つmanifestに結び付ける。手編集を禁止し、再生成で同じbytesになることをgolden testする。

- MCD projection、Provider Schema、generated C++／TypeScript、Reference docsはBuild treeに生成する。
- Source treeへCommitするのは正本入力と明示されたgolden hash／fixtureだけである。
- Generated bannerだけを手編集防止策にせず、write permission、lint、regeneration diffで強制する。
- Generated outputを別の正本へ昇格せず、入力が失われたoutputは無効とする。
- AI生成物は「generated」の名目で承認を迂回せず、Source候補なら`staging/`からChangeSetを通す。

## 10. Module、namespace、target mapping

一つのComponent stemから次を機械的に導出する。

| 種別 | 例: `render_graph` |
|---|---|
| Directory | `engine/rendering/render_graph/` |
| C++ namespace | `mirakan::rendering::render_graph` |
| Primary module | `mirakan.rendering.render_graph` |
| CMake target | `mirakan_rendering_render_graph` |
| Public include（移行期のみ） | `mirakan/rendering/render_graph/...` |
| Test target | `mirakan_rendering_render_graph_tests` |

Module segment、namespace segment、target segmentを意味の違うaliasへ変換しない。Primary moduleへversion suffixやPlatform suffixを付けず、Platform差はprivate Adapter targetで表す。ConsumerはpartitionやAdapter moduleを直接importしない。詳細なModule Cutoverは[C++23 modules](cpp23-modules.md)を参照する。

## 11. C ABI、CMake、HLSL、Tool language

- C ABI function／typeは`mirakan_`＋lowercase snake_case、macroは`MIRAKAN_`＋uppercase snake caseを使う。
- CMake targetはlowercase snake_case、public aliasを乱立させない。Preset、option、cache variableは所有scopeをprefixで明示する。
- HLSLのtype／function／resource名はShader Interface Ownerの語彙へ合わせ、Backend固有略語をMaterial Contractへ漏らさない。
- TypeScriptはtype／classをPascalCase、function／variableをcamelCaseとし、Schema fieldとStable IDは正規snake_caseを変換せず扱う。
- Language間projectionはgeneratorが担い、人手で別名表を維持しない。

## 12. Directory／file追加Operation

AI、Editor、Toolは任意File system writeではなく、分類、Owner、正規root、提案Path、identity、expected Project revisionを持つtyped Operationを使う。

1. Pathをseparator、case、Unicode、reserved name規則で正規化する。
2. Project root内か、分類に合う許可rootかを検査する。
3. 既存Path、Stable ID、content hash、Catalog entryとの衝突を検査する。
4. Source変更はPreviewとdiffを作り、必要なApprovalを得る。
5. 一つのProject ChangeSetでfile、manifest、Catalog、referenceをCommitする。
6. Derived／Intermediate／Package生成はSource revisionとToolchain hashへ結び付ける。

任意absolute path、`..`、symlink／junctionによるroot脱出、既存File上書き、暗黙rename、衝突時の自動連番を拒否する。

## 13. Invalid examples

| Invalid | 理由 | Correct direction |
|---|---|---|
| `engine/Common/Utils/` | case、曖昧責務 | Domainと役割で分割 |
| `class GPUAPIManager` | acronym、`Manager`、責務不明 | `GpuDeviceGateway`等の実役割 |
| `operation.asset.do` | 動詞が曖昧 | `operation.asset.source.plan_import` |
| `ERR_FILE_42` | Domain、条件、安定性不足 | `diagnostic.asset.invalid_source_path` |
| `My Game/FinalAssets2/` | display名とPath、世代suffix混在 | Project IDとCatalog identityを分離 |
| `source/generated/` | SourceとGeneratedのAuthority混同 | `derived/generated/`またはBuild tree |
| `packages/edit_me.json` | PackageをSource化 | `source/config/`を編集して再Package |
| `FooV1`とsuffixなし`Foo` | Schema最新版alias | major付き型だけを使用 |
| `engine/rendering/source/foo.cpp`をGameからinclude | Engine private境界違反 | public module／Portを利用 |

## 14. Lintとmigration Gate

Commit Gateは少なくとも次を機械検査する。

- File／Directory case、ASCII、separator、reserved name、Unicode normalization、長さ、衝突
- Project stem、acronym、suffix、単位、Boolean、Stable ID grammar
- Module、namespace、CMake target、Directoryの一対一mapping
- 禁止語だけで責務を表す型／Directoryと、日付付きactive仕様名
- Source root内のGenerated／Build／cache artifact、Git追跡されたDerived／Intermediate
- Generated fileのmanifest、input hash、再生成diff、手編集
- Operation／Diagnostic IDの重複、alias、動的segment

移行はinventory、deterministic rename map、reference rewrite、case-only rename用の二段階move、clean checkout検証を一つのChangeSetで行う。旧Path redirect、compatibility symlink、旧Target alias、二重Module名を残さない。

Migration完了条件:

1. 旧名と旧Pathの参照が0件である。
2. Windowsとcase-sensitive環境のclean checkoutで同じtreeを得る。
3. 全Project manifest、Catalog、Schema、CMake、Module import、docs linkが新名へ解決する。
4. Generated outputを全削除して再生成してもdiffがない。
5. Invalid fixtureがclosed Diagnosticで拒否される。

## 15. 明示的に採用しない方式

- 名前から型、ownership、Authorityを推測させるHungarian notation
- public識別子の任意略語、同義語、単数／複数の無規則混在
- OSごとのcase差、separator差、reserved name差を許容するPath
- 旧名alias、redirect、symlink、compatibility targetによる恒久互換
- display nameからStable ID、target、package identifierを暗黙生成すること
- AIがlint、Catalog、ChangeSetを通さずFile／Directoryを追加すること
