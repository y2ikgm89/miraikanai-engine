# Miraikanai Engine C++23 Language／Public Surface Contract

- 文書ID: mirakan.arch.cpp23-modules
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: C++23 language profile、required language／library feature set、First-party C++ public surface、Named Modules／`import std` adoption state、Header／BMI禁止境界、将来の単一surface変更Gate
- 非正本範囲: Compiler・CMake・Ninja・SDKのexact version／hash／取得元、Target×Configuration flag、一般命名・Directory、Memory／Pointer、Native Game ABI、Platform package。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Core Architecture](core-architecture.md)、[Toolchain／Dependencies](toolchain-dependencies.md)、[Naming／Project Layout](naming-project-layout.md)
- 関連文書: [Product Plan](../00-product/product-plan.md)、[Executable Contracts](executable-contracts.md)、[Memory／Pointers](memory-pointers.md)、[Native Game Module](../03-authoring/native-game-module.md)、[Compatibility／Evolution](compatibility-evolution.md)、[C++23 Header Shipping／Toolchain Baseline Decision](../decisions/2026-07-30-cxx23-header-shipping-toolchain-baseline.md)
- 根拠区分: project-decision（C++ standard、Compiler capabilityはofficial-spec、採用Profileはproject-decision）
- 外部根拠確認日: 2026-07-30

## 1. 結論

Miraikanai initial V1のFirst-party CPU codeと公開C++ SDKは、ISO C++23 language modeを使う単一のHeader-based Shipping surfaceとする。C++ Named Modules、Standard Library Module `import std`、Header Unit、BMI配布はinitial V1で無効であり、Shipping prerequisiteではない。

これは旧方式、暫定fallbackまたは将来Modulesへの移行段階ではなく、initial V1の完成Public Contractである。Source、Build設定、SDK Header、Feature Probe、ReceiptはRepositoryに存在せず、本書は実装済みまたはqualifiedを意味しない。

```text
initial V1
  language_standard: cxx23
  public_cpp_surface: textual_headers
  named_modules: disabled
  import_std: disabled
  header_units: disabled
  distributed_bmi: forbidden
```

Named ModulesはCompiler frontend機能であり、Plugin ABI、binary compatibility、security sandbox、Process isolationまたはMemory safetyを与えない。Native Game ModuleのC ABI／Process／Promotion境界、AI Source Worker、Project ChangeSet validationを置き換えない。

## 2. Canonical Profile

```text
Cxx23LanguageProfileV1
  language_profile_id: cxx23.shipping.initial-v1
  language_profile_version: 1
  standard_identity: ISO_IEC_14882_2024
  language_mode: cxx23
  required_feature_ids[1..256]:
    sorted unique CxxFeatureIdV1
  forbidden_extension_ids[0..256]:
    sorted unique CxxExtensionIdV1
  target_compiler_bindings[3..3]:
    sorted unique {
      target_kind: windows | android | apple,
      toolchain_profile_ref: exact ToolchainProfileRefV1,
      language_mode_argument: non-empty canonical ASCII,
      standard_library_binding_ref: exact ArtifactSetRefV1,
      compiler_runtime_binding_ref: exact ArtifactSetRefV1
    }
  named_module_adoption_state_ref:
    exact CxxModuleAdoptionStateRefV1
  public_surface_profile_ref:
    exact CxxPublicSurfaceProfileRefV1
  language_profile_content_hash: SHA-256

CxxModuleAdoptionStateV1
  module_adoption_state_id: StableId
  module_adoption_state_version: 1
  named_modules: disabled
  import_std: disabled
  header_units: disabled
  distributed_bmi: forbidden
  production_module_interface_count: 0
  adoption_basis: initial_v1_single_header_surface
  module_adoption_state_content_hash: SHA-256

CxxPublicSurfaceProfileV1
  public_surface_profile_id: StableId
  public_surface_profile_version: 1
  public_cpp_surface: textual_headers
  public_c_surface: generated_c_abi_headers
  public_header_root: include/mirakan
  public_header_rules_ref: exact McdContractRefV1(kind=policy)
  public_contract_set_ref: exact PublicContractSetRefV1
  public_header_manifest_artifact_ref: exact ArtifactRefV1
  public_surface_profile_content_hash: SHA-256
```

| Ref | exact tuple |
|---|---|
| `Cxx23LanguageProfileRefV1` | `{language_profile_id, language_profile_version=1, language_profile_content_hash}` |
| `CxxModuleAdoptionStateRefV1` | `{module_adoption_state_id, module_adoption_state_version=1, module_adoption_state_content_hash}` |
| `CxxPublicSurfaceProfileRefV1` | `{public_surface_profile_id, public_surface_profile_version=1, public_surface_profile_content_hash}` |

各content hashは順にASCII `MIRAKAN_CXX23_LANGUAGE_PROFILE_V1`、`MIRAKAN_CXX_MODULE_ADOPTION_STATE_V1`、`MIRAKAN_CXX_PUBLIC_SURFACE_PROFILE_V1`と、自己hashを除くlength-framed canonical bytesをSHA-256する。Profile、Toolchain、Public Contract Set、Header Manifestの全Refをbyte equalityにし、ID-only、`latest`、近いCompiler、同名Headerへfallbackしない。

## 3. Required C++23 feature closure

`CxxFeatureIdV1`は次のclosed initial setを少なくとも含む。

### 3.1 Language

- `language.concepts_requires`
- `language.consteval`
- `language.if_consteval`
- `language.constexpr_dynamic_allocation`
- `language.designated_initializers`
- `language.explicit_object_parameter`
- `language.multidimensional_subscript`
- `language.lambda_template_parameter_list`
- `language.using_enum`
- `language.char8_t`
- `language.three_way_comparison`

### 3.2 Standard Library

- `library.expected`
- `library.span`
- `library.mdspan`
- `library.ranges`
- `library.jthread`
- `library.stop_token`
- `library.source_location`
- `library.format`
- `library.to_underlying`
- `library.byteswap`
- `library.unreachable`
- `library.bit_cast`
- `library.atomic_wait_notify`

各featureはMCD Feature record、official feature-test macro、positive compile／link／execute fixture、negative semantic fixture、Target三種のsame-result Receiptへexactly oneで解決する。CompilerがC++23全機能を実装したという一般claimではなく、Miraikanaiのrequired setを満たすことだけをShipping predicateにする。required set外のC++23機能をSourceへ導入する場合はProfile revisionと全Target fixtureを同じArchitecture Changeで更新する。

`__cplusplus`、Compiler version文字列、language mode flagの単独一致をFeature passにしない。Compiler extension、vendor intrinsic、implementation-defined ABIを使う場合はEngine-owned Adapter、explicit Extension record、Target scope、fallback、negative fixtureを必要とし、標準featureへ偽装しない。

## 4. Initial V1 public Header contract

全公開C++ Headerは次を満たす。

- exact Public Contract Setから生成または承認され、Header Manifestへcontent hash付きで列挙される。
- self-containedで、単体translation unitからcompileできる。
- own declarationに必要な標準Headerを直接includeし、transitive includeへ依存しない。
- include guardは`#pragma once`へ依存せず、Naming Ownerが定める一意macro guardを持つ。
- Vendor Header、Platform Header、private implementation Header、raw OS handle、Vendor型を公開signatureへ露出しない。
- raw owning pointer、unbounded container、exceptionによるcross-boundary error、RTTI identityを公開Contractにしない。
- inline／template bodyは公開ContractとしてConsumer InventoryとCompatibility判定へ含める。
- macroはgenerated C ABI、preprocessor bridge、Platform bridgeの明示Directory以外で公開APIにしない。

Public Header、generated C ABI Header、preprocessor bridge、Platform bridgeを次のrootへ分離する。

```text
include/mirakan/<component>/
include/mirakan/c_api/
include/mirakan/preprocessor/
include/mirakan/platform_bridge/
```

Source tree、CMake target、public Header Manifest、MCD public member集合を正逆方向にset equality検査する。Header file名、namespace、CMake target、public member IDから他のidentityを推測しない。

## 5. Named Modules／BMI禁止境界

Initial V1のproduction SourceとPackageでは次をfail closedにする。

- `.cppm`、`.ixx`、production module interface unit、module partition。
- `export module`、`import std`、Header Unit import、Vendor BMI import。
- `CMAKE_EXPERIMENTAL_CXX_IMPORT_STD`、`CXX_MODULE_STD`、`FILE_SET TYPE CXX_MODULES`。
- IFC／PCM／BMIをSDK、Runtime Package、Cache seedまたはthird-party distributionへ含めること。
- HeaderとModuleの二つのPublic C++ surfaceを同時に公開すること。

Internal compiler-generated PCHはPublic Contractまたはdistributed BMIではないが、Toolchain／Configuration／Source／flag hashへ閉じた破棄可能Cacheに限る。PCHからHeaderのself-contained性、dependency、ABIを推測しない。

## 6. Target binding

Target別Compiler、standard library、runtime、language mode argumentは[Toolchain／Dependencies](toolchain-dependencies.md)のexact `TargetConfigurationBuildPolicyV1`が所有する。本書は次のpredicateだけを所有する。

1. Windows、Android、AppleのShipping policyがすべて`Cxx23LanguageProfileRefV1`へbyte equalityで束縛される。
2. 各TargetのCompiler／standard library feature probeが§3のrequired setとset equalityである。
3. Public Header Manifestから同じAPI／ABI fixtureをbuildし、Target固有Adapter以外のpublic signature差が0件である。
4. exception、RTTI、visibility、hardening、LTO、ISA、symbol policyはToolchain OwnerのTarget×Configuration recordに従い、Header macroから上書きしない。

一Targetのpass、Compilerのmode受理、`std::expected`一件、Editorだけのbuild、Development artifactを全Target Shippingへ一般化しない。

## 7. AI／MCD projection

AIとGeneratorは同じMCD Public ContractからHeader宣言、C ABI、Documentation、Tool Schemaを投影する。AIが自由文からHeaderを直接作り、public macro、Vendor型、exception、RTTI、Module interfaceを追加しない。

```text
CxxPublicMemberProjectionV1
  public_contract_member_ref: exact PublicContractMemberRefV1
  cxx_header_declaration_ref: exact ArtifactRefV1
  c_abi_declaration_ref: null | exact ArtifactRefV1
  value_transfer_policy_ref: exact CppValueTransferPolicyRefV1
  error_contract_ref: exact McdContractRefV1(kind=type)
  target_availability_refs[1..3]:
    sorted unique exact TargetProfileRefV1
  projection_content_hash: SHA-256
```

同じmemberのHeader、C ABI、Documentation、MCP projectionで意味、authority、error、lifetime、threading、Target availabilityを変えない。Pointer／ownershipはMemory Owner、Native Game isolationはNative Game Module Owner、Operation authorizationはExecutable Contracts Ownerが所有する。

## 8. 将来Named Modulesを採用する条件

Named Modulesはinitial V1の未完了項目ではなく、現在のrequired universeに含めない。最初のPublic release後に採用を提案する場合だけ、architecturally significantなPublic Contract変更としてsuccessor ADRと新しいProfile versionを作る。

採用predicateは少なくとも次である。

- 全Targetのstable non-preview Compiler／standard library／CMake／Generatorで同じModule graphが成立する。
- `import std`、third-party Header境界、IDE、debugger、static analysis、sanitizer、package、cache invalidationのpositive／negative fixtureが揃う。
- Header surfaceとModule surfaceの同時公開を行わず、新ProfileのPublic C++ surfaceを一つにする。
- 公開済みHeader consumerへの影響をCompatibility Ownerのcomplete Consumer Inventory、Change classification、support policy、migration／rollback decisionへ束縛する。
- correctness、clean／incremental build、package size、startup、debuggabilityを同じTarget／Source／Toolchainで比較し、qualified Decisionを得る。

条件未成立時はinitial V1 Header surfaceを維持し、空Module、alias Module、dual generator、experimental Shipping routeを追加しない。本節は実装順、移行計画、採用予定またはModules推奨を表さない。

## 9. Qualification

必要なFixture／Evidenceは次である。

1. 全required featureのpositive／negative／feature-test macro fixture。
2. Windows、Android、AppleのDevelopment／Test／Profile／Shipping matrixで同じProfile refを使用すること。
3. 全公開Headerのstandalone compile、include-order permutation、transitive include欠落、ODR、visibility、ABI fixture。
4. generated Header／C ABI／Documentation／Tool projectionのmember set equality。
5. forbidden Module token、Module file、CMake property、BMI package entryが0件であること。
6. exceptions／RTTI／Vendor型／raw owning pointer／unbounded valueがPublic Contractへ流出しないこと。
7. Toolchain lock差、Compiler差、standard library差、Target差、Configuration差でBuild treeとCacheが共有されないこと。
8. Source、Header Manifest、Public Contract Set、Toolchain、Package、Receiptのhash lineageが再現すること。

文書、Profile名、Header規則の存在はSource、SDK、Compiler fixture、ABI、PackageまたはQualification Receiptの存在を意味しない。

## 10. Failure diagnostic

| code | condition | result |
|---|---|---|
| `MIRAKAN-BUILD-CXX23-FEATURE-MISSING` | required feature／macro／semantic fixture欠落 | Target×Configuration buildをArtifact生成前に拒否 |
| `MIRAKAN-BUILD-CXX23-PROFILE-MISMATCH` | Source、Toolchain、Header ManifestのProfile ref不一致 | configureを拒否 |
| `MIRAKAN-BUILD-PUBLIC-HEADER-NOT-SELF-CONTAINED` | standalone compileまたはinclude closure失敗 | SDK publicationを拒否 |
| `MIRAKAN-BUILD-PUBLIC-SURFACE-MEMBER-MISMATCH` | MCD、Header、C ABI、Documentationのmember差 | Public Contract promotionを拒否 |
| `MIRAKAN-BUILD-NAMED-MODULE-FORBIDDEN` | initial V1でModule source／token／propertyを検出 | configureを拒否 |
| `MIRAKAN-BUILD-BMI-DISTRIBUTION-FORBIDDEN` | SDK／Package／Cache seedにBMIを検出 | packageを拒否 |
| `MIRAKAN-BUILD-PUBLIC-ABI-POLICY-VIOLATION` | exception、RTTI、Vendor型、raw ownership流出 | Public Contract promotionを拒否 |

## 11. 一次資料

- [ISO C++ current standard publication information](https://www.iso.org/standard/83626.html)
- [Clang C++ implementation status](https://clang.llvm.org/cxx_status.html)
- [Clang 22.1 compiler command guide](https://releases.llvm.org/22.1.0/tools/clang/docs/CommandGuide/clang.html)
- [CMake 4.4 C++ Modules support](https://cmake.org/cmake/help/v4.4/manual/cmake-cxxmodules.7.html)
- [Microsoft `/std` language mode reference](https://learn.microsoft.com/en-us/cpp/build/reference/std-specify-language-standard-version)

Clangの公式statusがC++23を`Partial`と表示するため、MiraikanaiはCompiler名またはlanguage modeだけから完全適合を主張せず、§3のrequired feature closureをTargetごとに検証する。MSVCの`/std:c++23preview`はPreviewであり、initial V1 Shipping frontendに使わない。Named Modules資料は将来proposalの比較根拠に限り、initial V1のrequired implementationではない。
