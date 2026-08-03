# glTF Import Dependency Baseline

- 文書ID: mirakan.decision.gltf-import-dependency-baseline
- 状態: review
- 正本範囲: initial V1のglTF parser、tangent生成器、仕様適合性oracleを一つの閉じた依存構成として選定する判断理由
- 非正本範囲: dependency artifact／hash／SBOM／NOTICE、Toolchain lock、Importer API、Schema、Registry、Fixture、Receipt、C++実装方式、実装Task、実装順序、工程、工数、担当。各Owner文書を参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)、[Materials](../06-rendering/materials.md)
- 外部根拠検証日: 2026-08-03
- 文書種別: Architecture Decision／glTF import dependency baseline
- Decision owner document: `mirakan.arch.toolchain-dependencies`
- Decision日: 2026-08-03
- Supersedes: none

## Context

Asset LifecycleとMaterialsはglTF 2.0 Source、typed Scene IR、normal texture、tangent-space normalの意味を定義しているが、Production Asset生成に使用するparserとMikkTSpace generatorは`not_adopted`であった。このため全glTF importをfail closedにしており、`ARCH-C135`はLibrary候補名ではなくArchitecture選定の未解決事項として残っていた。

Khronos glTF 2.0.1は、tangentがない場合にnormal textureが参照するtexture coordinates、position、normalからdefault MikkTSpaceでtangentを生成することを推奨し、normalがない場合はflat normalを生成してprovided tangentを無視することを要求する。KhronosはglTF-Validatorを仕様検証Toolとして提供するが、MiraikanaiのProduction parserを指定していない。parser、隔離境界、対応subset、fail-closed policyの選択はMiraikanaiのproject-decisionである。

## Decision drivers

- untrusted Source parserをShipping／RuntimeとEngine公開C++型から隔離する。
- 初期V1の供給網、ABI、例外、依存closureを最小化する。
- 外部Scene object modelを正本にせず、Miraikanai固有のtyped IRへ変換する。
- parser support、Engine capability、Khronos仕様適合性を別々に検証する。
- missing tangentを近似、DCC依存、runtime生成またはsilent normal-map無効化へfallbackしない。
- 公開前のinitial V1に二重parser、legacy aliasまたは恒久fallbackを持たない。

## Considered options

### A. `cgltf`をparserとして選定する

採用する。`cgltf`はC99単一ファイル、外部依存なし、MITであり、memory allocatorとfile I/O callback、memory parse、`cgltf_validate`を提供する。C ABIのprivate boundaryに閉じ、Miraikanai-owned Adapterがbounded Source bytesから`SceneImportIRV1`へ直ちに変換できる。Libraryの型、pointer、URI resolverまたはfile accessをEngine公開面へ出さない。

### B. `fastgltf`をparserとして選定する

採用しない。modern C++17、`Expected<T>`、SIMD、custom buffer allocationは有力だが、初期V1ではC++型／compile option／ISAとembedded dependencyを含む固定・Qualification範囲が広い。測定済みのimport throughput不足がない段階で追加closureを負わない。将来再選定する場合も`cgltf`とのdual Production pathにはせず、新しいADRとDependency ChangeSetで置換する。

### C. first-party glTF parserを作る

採用しない。外部object modelを避けられる一方、JSON／GLB、buffer／accessor、URI、extension、malformed input、fuzzと仕様更新の責務を初期V1へ新規導入する。Miraikanaiの独自性はparserの再実装ではなく、typed IR、意味検証、AI-readable contract、transaction、Evidence境界に置く。

### D. 複数parserまたはbest-effort fallbackを持つ

採用しない。同じSourceに複数解釈、Diagnostic、決定性classが生じ、失敗を別Backend成功で隠す。initial V1は一つのparserだけを選び、不合格入力は拒否する。

## Decision

1. initial V1のProduction Asset生成用glTF 2.0 parserは`cgltf` tag `v1.15`、commit `bbeb5b0b070ddacddac6852fb72143eb68454937`をtarget baselineとして選定する。
2. tangent生成器はMorten S. Mikkelsenの原典MikkTSpace、commit `3e895b49d05ea07e4c2133156cfa94369e19e409`の`mikktspace.h`／`mikktspace.c`を未変更で使用するtarget baselineとする。
3. Khronos `glTF-Validator` commit `434283be08a668a8fb4e437145630ddbf93b0686`（source version `2.0.0-dev.3.11`）をDevelopment／Qualification専用の仕様適合性oracleとして選定する。Production parser、Shipping dependencyまたはEngine semantic validatorの代用にしない。
4. `cgltf`とMikkTSpaceはAsset Import Workerのprivate dependencyであり、Shipping Runtime、public C++ surface、MCD、Scene／Asset Schemaへ外部型を露出しない。
5. Source transportはBrokerが供給するbounded bytes／dependency handlesだけを使用し、default file API、arbitrary URI、network、environmentまたはworking-directory resolutionを許可しない。
6. parserが認識できるcore／extensionとMiraikanaiがImport可能なCapabilityを分離する。Ownerのallowlist、Target capability、semantic validationを満たさないextension、external decoder、Morphまたはunknown featureはrejectする。
7. validなprovided `TANGENT`を優先する。欠落しnormal textureが必要とする場合だけ、同textureのexact `TEXCOORD_n`、position、normalをMikkTSpaceへ渡す。出力はper-corner／unindexedとして扱い、既存indexへの平均・上書きを禁止する。
8. MikkTSpaceの内部allocationは原典を改変せず、Adapterの事前bound／overflow検査とWorkerのcommit-memory hard capで囲む。Source改変が必要になった場合は、このDecisionのbaselineとは別のpatch identity、altered-source表示、license、hash、determinism再Qualificationを必要とする。
9. 各Production Import Jobでは`cgltf_validate`とMiraikanaiのbounds／allowlist／semantic／Target validation、Importer baselineのDevelopment／QualificationではKhronos Validatorを要求し、相互に代用しない。各scopeでrequiredな検査不合格、non-finite、invalid handedness、生成失敗またはQualification Evidence欠落をProduction成功にしない。
10. exact source archive hash、file hash、license／NOTICE、build options、patch state、SBOM、Toolchain lock、Adapter、Fixture、Receipt、cross-host Qualificationがmaterializeするまで、選定を`adopted`、`locked`、`qualified`または`active`と表現せず、glTF Import Capabilityをfail closedに保つ。

## Consequences

- `ARCH-C135`の「どの依存構成を選ぶか」は`closed-in-target-design`になる。
- dependency materialization、license／provenance、Adapter、Conformance、Qualificationは未完了であり、現在のglTF importは引き続き利用不能である。
- AI-native C++面はMiraikanai-owned typed contractとC++ Adapterを保ち、第三者parserの言語やobject modelを公開設計へ持ち込まない。
- 外部EngineのImporter、Scene model、Asset database、API、workflowまたは名称を模倣しない。
- 将来parserを変更する場合は既存baselineを黙って併存させず、source／IR semantic equivalence、Diagnostic、determinism、consumer impactを持つ新しいDecisionで置換する。
- Repositoryにdependency archive、Toolchain lock、SBOM、Adapter、Schema、Fixture、ReceiptまたはEngine実装は存在せず、本Decisionは採用完了、Build成功、QualificationまたはActivationを主張しない。

## Canonical Owner documents

- dependency version／commit／license／取得元／materialization gate: [Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)
- sandbox、Source transport、typed IR、Conversion Report、Import Receipt、semantic validation: [Asset Lifecycle](../03-authoring/asset-lifecycle.md)
- provided／generated tangent、normal texture UV、handedness、PBR意味: [Materials](../06-rendering/materials.md)
- Morph initial exclusion: [Initial Morph Capability Boundary](2026-08-03-initial-morph-capability-boundary.md)

## Supersedes／Superseded by

- Supersedes: none
- Superseded by: none

## Official or primary sources

- [Khronos glTF Registry／glTF 2.0.1](https://registry.khronos.org/glTF/)
- [Khronos glTF 2.0 specification](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html)
- [`cgltf` v1.15 source](https://github.com/jkuhlmann/cgltf/tree/v1.15)
- [`cgltf` v1.15 license](https://raw.githubusercontent.com/jkuhlmann/cgltf/v1.15/LICENSE)
- [MikkTSpace exact source](https://github.com/mmikk/MikkTSpace/tree/3e895b49d05ea07e4c2133156cfa94369e19e409)
- [MikkTSpace exact header／license notice](https://raw.githubusercontent.com/mmikk/MikkTSpace/3e895b49d05ea07e4c2133156cfa94369e19e409/mikktspace.h)
- [Khronos glTF-Validator exact source](https://github.com/KhronosGroup/glTF-Validator/tree/434283be08a668a8fb4e437145630ddbf93b0686)
- [Khronos glTF-Validator license](https://raw.githubusercontent.com/KhronosGroup/glTF-Validator/434283be08a668a8fb4e437145630ddbf93b0686/LICENSE)
