# Miraikanai Engine C++23 Modules

- 文書ID: mirakan.arch.cpp23-modules
- 状態: review
- 正本範囲: C++23 language profile、Named Module境界、`import std`移行state、CMake module表現、Header例外、BMI identity、Cutover Gate
- 非正本範囲: Compiler・CMake・Ninja・SDKのexact version／hash／取得元、一般命名・Directory、Memory／Pointer、Native Game ABI、Platform package。各Owner文書を参照する
- 依存: [Product Plan](../00-product/product-plan.md)、[Core architecture](core-architecture.md)、[Toolchain／Dependencies](toolchain-dependencies.md)、[Executable contracts](executable-contracts.md)、[Naming／Project layout](naming-project-layout.md)、[Memory／Pointers](memory-pointers.md)
- 外部根拠検証日: 2026-07-23

## 1. 結論

Miraikanai EngineのC++製品基準と最終Source構成を次に固定する。

1. Engine、Editor、Tool、GameHost、NativeGameModuleのFirst-party CPU codeはC++23とする。
2. First-party C++公開境界はC++ Named Modulesへ移行する。
3. Named Module内の標準Library利用は原則`import std;`へ統一する。
4. C++26はShipping基準にせず、同じC++23 Sourceを`/std:c++latest`または`-std=c++2c`でもCompileするreadiness CIだけを持つ。
5. Modules／`import std`は採用済みの最終方式であり、今後のGateは採否を再検討するものではなく、安全に正式有効化できる時点を判定する。
6. 現行Toolchainに残る制限を製品Buildへ持ち込まないため、Header準備期、Probe期、Cutover候補期、正式Module期の一方向移行を行う。
7. 正式移行後はHeader版公開APIとの恒久的二重Buildを持たず、Engine C++ Public Headerを削除する。C ABI、Preprocessor macro、Objective-C／JNI等の言語境界だけを小さなHeaderとして残す。

ModulesはCompilerが強制するSource公開境界であり、Plugin ABI、Binary互換層、Security sandboxではない。NativeGameModuleのProcess／C ABI／Promotion境界、AI Source Worker、構造化ChangeSet検証は従来どおり必要である。

## 2. 用語と決定権

| 用語 | 本書での意味 |
|---|---|
| CMake target | Compile、Link、使用要件、依存edgeを所有するBuild単位 |
| C++ Named Module | `export module mirakan.<name>;`で宣言するC++ Source公開単位 |
| Module partition | `mirakan.<name>:<partition>`に属し、同じNamed Moduleを分割する単位 |
| NativeGameModule | Project固有C++を隔離Build／PromotionするMiraikanai製品概念。C++ Named Moduleとは別物 |
| Standard Library Module | C++23の`std` Named Module。本書では`import std;`だけを正式利用する |
| BMI | Compilerが生成するBinary Module Interface。MSVCのIFCを含む総称 |
| Textual Header | Preprocessorの`#include`で展開するHeader |
| Module graph | Named Module間の`import`依存DAG |

言語Version、Named Module名、CMake統合、Header例外、移行State、正式Cutover Gateは本書が決定する。Memory、Pointer、Exception、Directory一般則は基盤規約、NativeGameModuleのC ABIと信頼境界はNative Game規約、Apple package／署名はモバイル規約が決定する。

## 3. 外部Toolchain capabilityの扱い

Tool、Compiler、SDKのexact release、version、hash、取得元と、そのversionで検証したcapability evidenceは[Toolchain／Dependencies](toolchain-dependencies.md)だけが所有する。本書は検証済みcapabilityに対する製品判断だけを固定する。

| 確認事項 | Miraikanaiの判断 |
|---|---|
| Named Modules | C++23 Source公開境界として正式採用する |
| `import std` | 標準名は`import std;`、必要macroは限定Headerで明示する。`std.compat`は採用しない |
| Module scan | `FILE_SET CXX_MODULES`を正式なBuild表現とする |
| Experimental `import std` | Probeに限りExperimental gateを隔離利用し、正式期は非Experimental supportを必須にする |
| Apple build | portable C++ Module graphとApp shell／最終Link／Archive／署名をDriver Profileで分離する |
| Header Units | 移行手段にも正式方式にも採用しない |

CompilerやBuild Systemの対応表は「その組合せなら製品に安全」という保証ではない。固定Toolchain、全Target fixture、IDE、Sanitizer、Static Analysis、Package testを本書のGateで追加検証する。

## 4. 一方向の移行State

`CxxFrontendProfileV1`は次の四状態だけを持つ。文字列、遷移、Artifact用途を固定し、独自Profileを追加する場合は本書改訂を必要とする。

| State | Profile ID | Source公開方式 | Standard Library | Artifact用途 |
|---|---|---|---|---|
| CX0 | `cxx23_headers_bootstrap` | self-contained Public Header | 標準Header | Development、Test、candidate Package、internal Technology Previewだけ。Release Activation不可 |
| CX1 | `cxx23_modules_probe` | 限定Named Module fixture | Named Module内は`import std` | Development、Test、candidate Package、internal Technology Previewだけ。Release Activation不可 |
| CX2 | `cxx23_modules_candidate` | 全First-party公開APIをNamed Modulesへ変換したCutover branch | `import std` | 全Target候補検証。公開Release不可 |
| CX3 | `cxx23_modules_shipping` | Named Modulesが唯一のC++公開方式 | `import std` | Development、Profile、Shipping、ASanの正式方式 |

MCD recordは`{profile_id, state, source_api_mode, standard_library_mode, promotion_allowed, shipping_allowed}`を持ち、上表から生成する。Compiler、STL、CMakeとの対応はMCDへ埋め込まず、Target別`toolchain.lock.json`の`profiles[].build.cxx_bindings[]`へ次の`CxxToolchainBindingV1`として固定する。Configure入口とGeneratorは別契約の`BuildDriverProfileV1`が所有する。

`shipping_allowed`はProduction Release候補を署名／配布してよいかを表し、Shipping Configurationで内部Buildできるかを表さない。初期値はCX3だけ`true`、CX0～CX2は`false`である。CX0／CX1のDevelopment、Test、candidate Package、internal Technology Previewと、CX2の全Target候補検証は内部Evidence生成に限り、Release Activationへ入力できない。CX3の`shipping_allowed=true`もToolchain binding、全Target Gate、Package／Release Receiptを省略する許可ではない。

```text
CxxToolchainBindingV1
  frontend_profile_id: closed CxxFrontendProfileId
  language_standard: c++23
  compiler_standard_flag: exact string
  compiler_tool_id: closed ToolArtifactId
  compiler_full_version: exact string
  standard_library_hash: sha256
  cmake_tool_id: closed ToolArtifactId
  experimental_import_std_token: exact string
  module_cache_policy: toolchain_and_configuration_local
```

CX1だけが`experimental_import_std_token`にexact tokenを持ち、他Stateは空文字とする。配列はProfile ID順、重複不可とし、要求TargetにActive ProfileのbindingがなければBuild GatewayはProfile activationを失敗させる。Configure入口、Generator、Configuration identity、Package ownerは[Toolchain／Dependencies](toolchain-dependencies.md#3-build-driver-matrix)の`BuildDriverProfileV1`が所有し、C++ Frontend bindingへ重複保存しない。

許可遷移は`CX0 -> CX1 -> CX2 -> CX3`だけである。CX1はCX0とCI上で並行実行できるが、同じProduction artifact内でHeader公開APIとModule公開APIを選択可能にしない。CX2は隔離Cutover branchで行い、CX3へ昇格する一つのChangeSetに全変換、生成物更新、Header削除、Gate Receiptを含める。

CX3移行後にCompiler defectが見つかった場合は、別Compilerへ暗黙FallbackせずReleaseを停止する。未公開Cutover commit自体をGitでRevertすることはできるが、Shipping ProjectへHeader／Module選択Switchを提供しない。

## 5. C++23言語Profile

### 5.1 CX0の制限付きC++23

CX0はSource言語をC++23とし、Windowsでは[Toolchain／Dependencies](toolchain-dependencies.md)が固定するbootstrap compilerのpreview flag、Android／Apple／secondary CIでは同文書が固定するClangの`-std=c++23`を使用する。`/std:c++latest`をCX0の代替にしない。

CX0 compilerで未完の次の機能へProduction Sourceを依存させない。

- P2564R3 `consteval` immediate escalation。
- P0533R9 `constexpr <cmath>`／`<cstdlib>`。

Polyfill、Compiler別の別Algorithm、Feature macroによるRuntime意味分岐は作らない。CX3の全Compiler conformance fixtureに合格した後だけ利用を解放する。

C++23の`std::expected`を採用し、Foundationの公開Source表記は次のaliasへ統一する。

```cpp
template<class T>
using Result = std::expected<T, Error>;
```

`Result<void>`も同じaliasを使用する。C ABI、serialized data、NativeGameModule descriptorへ`std::expected`、`Error` object、STL objectを出さない。

### 5.2 CX3の完全C++23

CX3は各公式Targetで次を満たす。

- MSVCはStable toolsetの正式な`/std:c++23`を使用する。
- Clang系は`-std=c++23`を使用する。
- CMake targetは`target_compile_features(<target> PUBLIC cxx_std_23)`を宣言する。
- `__cplusplus`／`_MSVC_LANG`、P2564R3、P0533R9、`std::expected`、Named Modules、`import std`のcompile／negative fixtureに合格する。
- Compiler extensionへ依存せず、MSVCは`/permissive-`、Clangは`-Wpedantic -Werror`を維持する。

### 5.3 C++26 readiness

`cxx26_readiness`は同じC++23 Source treeを、Windowsでは`/std:c++latest`、Clangでは`-std=c++2c`でCompileする非Promotion CIである。

- C++26専用構文、Library API、Feature macro branchをProduction Sourceへ追加しない。
- Warning、削除予定機能、名前衝突、Modules regressionを早期検出する。
- Object、Library、Package、Generated SDKをShippingまたは性能比較の正本にしない。
- C++26移行時は本書を改訂し、C++23から一括Cutoverする。C++23／26二重製品Profileを作らない。

## 6. Named Moduleの境界と命名

一つの公開CMake targetは、最大一つのPrimary Named Moduleを所有する。Primary名はlowercase ASCIIとdot区切りとし、`mirakan.<domain>`、`mirakan.<domain>.<role>`、または`mirakan.<domain>.<backend>.adapter`のいずれかの形をとる。CMake aliasと1対1で固定する。公開Aliasを持たないtarget（conformance、benchmark、test）はPrimary Named Moduleを所有しない。

| CMake alias | Primary Named Module |
|---|---|
| `mirakan::foundation` | `mirakan.foundation` |
| `mirakan::math` | `mirakan.math` |
| `mirakan::runtime_contracts` | `mirakan.runtime.contracts` |
| `mirakan::gameplay` | `mirakan.gameplay` |
| `mirakan::native_game` | `mirakan.native_game` |
| `mirakan::ui_core` | `mirakan.ui.core` |
| `mirakan::ui_layout` | `mirakan.ui.layout` |
| `mirakan::ui_events` | `mirakan.ui.events` |
| `mirakan::ui_semantics` | `mirakan.ui.semantics` |
| `mirakan::ui_text` | `mirakan.ui.text` |
| `mirakan::ui_rendering` | `mirakan.ui.rendering` |
| `mirakan::ui_d3d12_adapter` | `mirakan.ui.d3d12.adapter` |
| `mirakan::ui_directwrite_adapter` | `mirakan.ui.directwrite.adapter` |
| `mirakan::ui_tsf_adapter` | `mirakan.ui.tsf.adapter` |
| `mirakan::ui_uia_adapter` | `mirakan.ui.uia.adapter` |
| `mirakan::ui_harfbuzz_freetype_adapter` | `mirakan.ui.harfbuzz_freetype.adapter` |
| `mirakan::editor_ui` | `mirakan.editor.ui` |
| `mirakan::editor_shell` | `mirakan.editor.shell` |
| `mirakan::editor_docking` | `mirakan.editor.docking` |
| `mirakan::editor_semantics` | `mirakan.editor.semantics` |
| `mirakan::editor_ole_adapter` | `mirakan.editor.ole.adapter` |
| `mirakan::rendering_d3d12` | `mirakan.rendering.d3d12.adapter` |
| `mirakan::<domain>_port` | `mirakan.<domain>.port` |
| `mirakan::<domain>_runtime` | `mirakan.<domain>.runtime` |
| `mirakan::<domain>_<backend>_adapter` | `mirakan.<domain>.<backend>.adapter` |

規則:

- ConsumerがimportするのはPrimary Named Moduleだけとする。
- Partition名は`mirakan.<primary>:<cohesive_contract>`とし、同じPrimary Moduleの実装・公開契約分割だけに使う。
- ConsumerによるPartition直接import、exported import cycle、Module名alias、version suffix、Platformごとの別Primary名を禁止する。
- CMake target DAGとModule graphのedgeを一致させる。どちらか一方にだけ存在する依存をconfigure errorにする。
- `mirakan.common`、`mirakan.shared`、`mirakan.utils`を作らない。
- Adapter ModuleをPort／Runtime Moduleからexportしない。Composition Rootだけがconcrete Adapterへ依存する。
- Module名はSource API identityであり、BMI filename、filesystem path、DLL名、NativeGameModule IDには流用しない。§7のModule interface filename（`mirakan.<component>.cppm`）だけを唯一の例外とする。

## 7. DirectoryとSource配置

CX0から次の標準形を使用する。

```text
<component>/
├─ CMakeLists.txt
├─ include/mirakan/<component>/       # CX0だけの移行用Public Header
├─ modules/
│  ├─ mirakan.<component>.cppm        # Primary interface
│  └─ <partition>.cppm                # cohesive partition
├─ source/                            # implementation unit／private source
├─ tests/
└─ benchmarks/
```

CX0では`modules/`にCX1 fixture以外の見せかけの空interfaceを置かず、production targetの`.ixx`／`.cppm`を0件とする。D3D12のCX0 public surfaceは`engine/rendering/d3d12/include/mirakan/rendering/d3d12/backend.hpp`だけであり、`mirakan::rendering_d3d12`のCMake Component登録は将来のPrimary Module名`mirakan.rendering.d3d12.adapter`だけを固定して、存在しないModule interfaceを登録しない。Header directory、CMake target、Module名の対応を機械検査する。

CX2で低level dependencyから`.cppm`を追加し、CX3 Cutover ChangeSetで`include/mirakan/<component>/`のEngine C++公開Headerを削除する。Generated C ABI、Preprocessor bridge、言語bridgeは`include/mirakan/c_api/`、`include/mirakan/preprocessor/`、`include/mirakan/platform_bridge/`へ分離し、Named Module APIと混在させない。

## 8. CMake契約

### 8.1 Component登録

Phase 0で一つのProject-owned関数`mirakan_add_cpp_component()`を定義し、First-party C++ targetを直接`add_library()`で公開登録しない。

```cmake
mirakan_add_cpp_component(
    TARGET mirakan_foundation
    ALIAS mirakan::foundation
    MODULE_NAME mirakan.foundation
    HEADER_API_ROOT "${CMAKE_CURRENT_SOURCE_DIR}/include"
    PUBLIC_DEPENDENCIES
)
```

CX0では`MODULE_NAME`だけで将来のPrimary名を固定し、存在するPublic Headerを登録する。存在しない`MODULE_INTERFACE`を指定しない。CX1は隔離probe targetだけが実在するfixture `.cppm`を`MODULE_INTERFACE`へ登録し、production componentへ投影しない。CX2／CX3では実在するproduction `.cppm`を`MODULE_INTERFACE`として必須指定し、次のCMake表現へ投影する。

```cmake
target_sources(mirakan_foundation
    PUBLIC
        FILE_SET cxx_modules TYPE CXX_MODULES
        FILES modules/mirakan.foundation.cppm
)
target_compile_features(mirakan_foundation PUBLIC cxx_std_23)
set_property(TARGET mirakan_foundation PROPERTY CXX_MODULE_STD ON)
```

関数はTarget名、Module名、Public dependency、Source file、生成MCD dependencyを`CxxComponentGraphV1`へ出力する。AI、CI、Documentationはこの生成Graphを読み、CMake fileを別Parserで推測しない。

### 8.2 Build Driver／Generator規則

CMakeを全First-party C++ targetの唯一のBuild定義とし、MCDの`BuildDriverProfileV1`と[Toolchain／Dependencies](toolchain-dependencies.md#3-build-driver-matrix)のexact Toolchain bindingに一致する入口だけを公式経路とする。

| Target／State | Driver Profile ID | 正規入口 | C++ Generator | Configuration単位 | 後段 |
|---|---|---|---|---|---|
| Windows／CX0–CX3 | `driver.windows.cmake-ninja-multi` | checked-in CMake Preset | `Ninja Multi-Config` | configuration | Windows Distribution |
| Android／CX0–CX3 | `driver.android.gradle-ninja` | 固定Gradle Wrapper＋`externalNativeBuild.cmake` | `Ninja` | Variant × ABI × C++ ProfileのSingle-Config tree | Gradleが`.so`をAPK／AABへpackage |
| Apple／CX0 | `driver.apple.cx0-xcode` | checked-in CMake Preset | `Xcode` | Xcode configuration | Xcode |
| Apple／CX1 | `driver.apple.modules-probe-ninja` | checked-in CMake Preset | `Ninja Multi-Config` | Probe configuration | Packageなし |
| Apple／CX2–CX3 | `driver.apple.modules-ninja-xcode` | checked-in CMake Preset | `Ninja Multi-Config` | C++ archive configuration | XcodeがC ABI App shell、最終Link、Archiveを所有 |

規則:

- CMakeの`Unix Makefiles`、`NMake Makefiles`、`NMake Makefiles JOM`、`MinGW Makefiles`、`MSYS Makefiles`、raw Makefile、First-party Android `ndk-build`を禁止する。
- CMake／Ninjaを呼ぶだけの互換`Makefile`、Make target名を維持するwrapper、MakeからNinjaへの段階移行期間を作らず、Phase 0から正規Driverだけを実装する。
- MakeとNinjaを選ぶProject option、Preset、Environment fallback、二重CIを作らない。
- AndroidはNinja Multi-Configを使わず、Gradle Variant × ABI × C++ ProfileごとのSingle-Config Ninjaへ固定する。
- Target、C++ Profile、Driver、Generator、Toolchain hashが異なるBuild treeを共有しない。既存Build treeの`CMAKE_GENERATOR`を書き換えず、別treeを作る。
- Windows／Appleの通常入口は`cmake --preset`／`cmake --build --preset`、AndroidはGradle Wrapperとし、利用者、AI、CIがProduct Buildで`ninja`を直接起動しない。
- CI／Promotionはcommand-line `-G`、`CMAKE_GENERATOR`環境変数、`CMakeUserPresets.json`による公式Generatorの上書きを拒否する。
- Makeしか提供しないThird-partyは隔離Dependency Buildでのみ許可し、検証済みimmutable artifactまたはCMake imported targetへ変換する。Engine／ProjectのBuild modeとして公開しない。
- NinjaはCMake生成DAGの実行器に限定し、Asset Cook、Shader Package、APK／AAB、Apple archive、Signing、Promotionの正本にしない。全製品工程はBuild Gatewayが型付きTaskとして順序付ける。
- Editor、AI、CIは`build.ninja`、`build-<Config>.ninja`、`.ninja_deps`、`.ninja_log`を解析または変更せず、CMake File APIとEngine-owned Build ReceiptだけからTarget、Configuration、Artifact、Diagnosticを取得する。
- checked-in CMake PresetのGenerator、binary directory、toolchain、configurationを正本とし、生成済みNinja fileはBuild treeとともに破棄可能でなければならない。

Makefiles系は現行CMakeのC++ Module scan対象に含まれず、`import std`はNinja／Ninja Multi-Configだけが対応するため、Makeを互換経路として残さない。Visual Studio GeneratorはModule scanに対応してもIMPORTED targetのBMIをbuildできないため、`import std`のBMI Shipping経路として使用しない。

### 8.3 許可するC++ Frontend組合せ

内部選択値は次の四組だけとし、任意のBoolean組合せを作らない。

| Profile | First-party API | Standard Library |
|---|---|---|
| `cxx23_headers_bootstrap` | Textual Header | Textual Header |
| `cxx23_modules_probe` | Named Modules | `import std` |
| `cxx23_modules_candidate` | Named Modules | `import std` |
| `cxx23_modules_shipping` | Named Modules | `import std` |

「Engine Modules＋標準Header」を長期Profileにしない。CX2内部で問題切分けの一時Buildを実行してもPromotion Receiptを発行せず、CMake Presetへ公開しない。

### 8.4 Experimental gateの隔離

採用CMakeの`CMAKE_EXPERIMENTAL_CXX_IMPORT_STD` tokenは、CX1専用`cmake/experimental/import_std_probe.cmake`だけに置く。Root `CMakeLists.txt`はCX1 Profileのときだけ、このFileを最初の`project(... LANGUAGES CXX)`より前にincludeし、CXX toolchain discovery前にtokenを設定する。exact CMake artifact、token、Generator、Compiler、STL hashは[Toolchain／Dependencies](toolchain-dependencies.md)に従って`toolchain.lock.json`へ記録する。

- Product targetの通常CMake fileでExperimental変数を参照しない。
- CMake version更新時に以前のtokenを再利用しない。
- CX1 artifactは隔離candidate Packageのlayout／loader testだけに渡し、Editor、GameHost、NativeGameModule Promotion、Release Packageへ渡さない。
- CMakeが`import std`を正式化した時点でProbe fileとtokenを削除してからCX2へ進む。

## 9. CX0のModule-ready Header規則

CX0のHeaderは一時的でも、次をすべて満たす。

- self-containedで単独Compileできる。
- Include What You Useに従い、推移的includeを契約にしない。
- Public Header間、CMake target間にcycleを作らない。
- `#include`をnamespace、class、function、macro expansion内へ置かない。
- Public APIの意味をmacro定義、include順、事前`#define`で変えない。
- Platform、Vendor、Compiler専用型とHeaderを公開しない。
- Mutable global、Header内Registry副作用、静的初期化順依存を持たない。
- 所有権はvalue、handle、`std::unique_ptr`、viewで表し、private implementationが必要ならPImplまたはEngine-owned handleを使う。
- Export対象の宣言とinline／template実装を、将来一つのPrimary ModuleまたはPartitionへ移せる責務単位に保つ。
- Public Header testは単独Compile、全順列ではなく依存DAG順と逆順のaggregate Compile、macro汚染scanを行う。

## 10. `import std`規則

CX1以降のFirst-party C++ Module unitは標準Library名を`import std;`で取得する。

```cpp
export module mirakan.foundation;

import std;

export namespace mirakan {
    // Public declarations
}
```

Textual includeが必要なModule unitはGlobal Module Fragmentへ限定する。

```cpp
module;
#include <cassert>

export module mirakan.foundation;

import std;
```

規則:

- `std.compat`をimportしない。
- `assert`、`errno`、`offsetof`、`va_arg`等が必要なSourceだけ、対応HeaderをGlobal Module Fragmentでincludeする。
- 標準Library実装内部名、underscore-prefixed helper、STL vendor extensionを使用しない。
- `import std;`を利用したことをRuntime最適化として記録しない。効果はBuild throughputとSource境界で測定する。
- `import std;`を理由に使用標準型をPublic C ABI、Save、Wire format、NativeGameModule descriptorへ出さない。
- Standard Library ModuleのBMIをRepository、SDK、Package、remote cacheへ配布しない。

## 11. Third-party／Platform境界

- Third-party LibraryをMiraikanaiの公開Named Moduleへ再exportしない。
- Vendor HeaderはAdapter実装unitまたはprivate Module partitionのGlobal Module Fragmentだけでincludeする。
- Header Unit化、`export import <vendor-header>`、Vendor BMIの再配布を禁止する。
- C、Objective-C、Objective-C++、JNI、OS callback境界はGenerated C ABI Header、opaque handle、caller-owned bufferを使用する。
- Preprocessor bridgeはBuild configuration、symbol visibility、assertion stringification、C ABI decorationに限定する。Gameplay型やDomain APIをHeaderへ戻さない。
- HLSL／MSL／SPIR-Vの「module」は本書のC++ Named Moduleと無関係であり、Shader規約に従う。

## 12. AI生成C++とMCD

MCDへ`CppDependencySetV1`を追加し、AIとContract compilerはraw `#include`／`import`文字列ではなく論理依存を扱う。

Generated／First-party C++の全public callable subjectは[Memory／Pointers](memory-pointers.md)の`CppValueTransferPolicyV1`へexact解決する。AI、hand-written Module、CX0 Header、CX3 Moduleで別規則を持たず、missing bindingを推測補完しない。

AST Gateは`const T&&`、const objectからのmove、conditional sink move、destroy／assign以外のmoved-from access、unqualified nontrivial by-value input、scalar／enum／typed handleの理由なき`const T&`、未writeのnon-const ref、returnで足りるoutput parameter、NRVO対象localの`return std::move(local)`を拒否する。

```text
CppDependencySetV1
  owner_component_id: StableId
  owner_module_name: closed ModuleName
  imports[]:
    module_name: closed ModuleName
    visibility: public | private
  standard_header_ids[]: closed StdHeaderId
  textual_includes[]:
    include_id: closed HeaderExceptionId
    reason: c_abi | required_macro | platform_bridge | vendor_private
```

制約:

- `imports`はModule名のunsigned UTF-8 byte順、重複不可、自己import不可とする。
- `visibility=public`はCX1以降の`export import`、`private`は`import`へ投影する。Adapter、Vendor、Platform依存をpublic importにできない。
- `standard_header_ids`は`std.vector`、`std.string`、`std.expected`のようなMCD登録済みIDをunsigned UTF-8 byte順で持つ。任意Header path、重複、対応表にないIDを拒否する。
- `textual_includes`はMCD登録済みIDだけを許可し、任意Pathを受理しない。
- CX0では`standard_header_ids`を対応する個別標準Headerへ投影し、Include What You Use検査と一致させる。CX1以降では配列が非空なら`import std;`を一度だけ生成する。
- `vendor_private`はprivate partition／implementation unitだけに許可し、Primary interfaceとexported partitionでは拒否する。
- GeneratorはCX0 Header projectionとCX2 Module projectionを同じMCDから作り、手書きで二重管理しない。
- ValidatorはSource scan結果と`CppDependencySetV1`を比較し、未宣言import、未許可include、cycle、private dependencyのpublic exportを拒否する。
- CX3後はEngine C++ Public Header projectionを生成せず、Module interface、C ABI Header、Preprocessor bridgeだけを生成する。
- AIがModule名、Partition、CMake edge、Experimental flag、Compiler optionを自由文字列で追加することを禁止する。新しい公開ModuleはR4、人間承認、DAG／ABI／Build Gateを必要とする。

## 13. Apple Build分離

### 13.1 CX0

CX0では既存規約どおりCMake Xcode GeneratorでC++23 Header Source、Objective-C++ Adapter、Metal、App shellをBuildする。これは移行前Profileであり、Module対応の証明に使わない。

### 13.2 CX3

CX3の`AppleUnsignedBuildWorkerV1`はBuildを次に分離する。

1. Ninja Multi-Configが、portable C++23 Named Module graph、Engine、NativeGameModuleをarm64 static archive／objectへCompileする。
2. BMIはNinja Build tree内部だけに保持し、Xcode projectへ渡さない。
3. [Toolchain／Dependencies](toolchain-dependencies.md)が固定するApple toolchain profileのXcodeが、最小App shell、Objective-C／Objective-C++ C ABI bridge、Platform resource、Entitlement、最終Link、Archive validationを担当する。
4. Xcode側SourceはC++ Named Moduleをimportせず、Generated C ABI Headerとopaque handleだけでNinja生成archiveへ接続する。
5. Metal compiler／binary archiveはShader toolchain lockに従い、C++ BMIと共有Cacheを持たない。
6. `UnsignedApplePayloadV1`はNinja Build Receipt、Xcode Build Receipt、両者が参照したSource root／Toolchain lock、最終Mach-O hashを含む。

Ninja側にSigning key、Provisioning profile、Store credentialを置かず、XcodeのSource-bearing unsigned stageと後段Signing／Upload分離はモバイル規約を維持する。

Ninja configureは同じXcode bundleのAppleClangとSDKだけを使用し、次を固定する。

```text
CMAKE_SYSTEM_NAME=iOS
CMAKE_OSX_SYSROOT=iphoneos
CMAKE_OSX_ARCHITECTURES=arm64
CMAKE_OSX_DEPLOYMENT_TARGET=<toolchain_lock.profiles[target.apple.mobile].target.deployment_target>
CMAKE_TRY_COMPILE_TARGET_TYPE=STATIC_LIBRARY
```

`xcrun --find clang++`と`xcrun --sdk iphoneos --show-sdk-path`の解決結果、AppleClang full version、SDK build、libc++ file-set hashをToolchain lockへ保存する。Homebrew／MacPorts／User PATHのClang、別XcodeのSDK、Ninja側Code Signingを許可しない。

## 14. BMIとCache identity

BMI／IFCは次をすべて含むCache keyで分離する。

```text
source_root_hash
module_graph_hash
compiler_executable_sha256
compiler_full_version
standard_library_file_set_hash
language_standard
compiler_flags_hash
target_triple
sdk_hash
cmake_version
generator
configuration
sanitizer_mode
module_name
module_source_hash
```

- 一項目でも異なれば再利用しない。
- Development、Profile、Shipping、ASan間で共有しない。
- MSVC、clang-cl、Android Clang、Apple Clang間で共有しない。
- Absolute path、User名、timestampをsemantic keyにしない。
- Stale BMI検出時は該当Targetの隔離Build treeだけを破棄し、Source、Project data、Asset cacheを削除しない。
- BMIはBuild artifact manifestに形式とsizeを記録できるが、Package／SBOMの配布Componentとして扱わない。

## 15. CX2 Cutover順序

CX2は依存DAGの下位から次の順で変換する。

1. Foundation value type、Error、Result、StableId。
2. Runtime contracts、generated MCD contract。
3. Domain Port。
4. Domain Runtime。
5. private AdapterとPlatform-independent Tool。
6. Host Composition Root、Editor、GameHost、WorkerHost。
7. NativeGameModule用Project C++ APIとAI generated code。
8. Apple／Android／Windowsの言語bridgeとPackage fixture。

各stepでPrimary Module、partition、CMake edge、Unit test、dependency scanを同じCommitに含める。変換済みTargetの宣言をHeaderと`.cppm`へ手書き複製しない。CX2 branchの中間状態はRelease branchへmergeせず、CX3 Gate合格後に一つのCutover ChangeSetとして昇格する。

## 16. CX3正式有効化Gate

次をすべて満たすまで`cxx23_modules_shipping`を有効にしない。

### 16.1 Toolchain

- Windows Primary compilerが[Toolchain／Dependencies](toolchain-dependencies.md)のCX3条件を満たし、正式な`/std:c++23`を提供する。
- CMakeの`import std`がExperimental tokenなしで利用でき、`CMAKE_CXX_COMPILER_IMPORT_STD`がC++23を列挙する。
- `import std`を使うproduction C++ archiveはNinja／Ninja Multi-Configでbuildし、Visual Studio Generator由来BMIをShipping packageへ含めない。
- Windows／Android／AppleのCompiler、STL、SDK、CMake、Ninja、Xcodeをexact version／hashで`toolchain.lock.json`へ固定する。
- Module dependency scan、`FILE_SET CXX_MODULES`、install／archiveがWindows／Appleの`Ninja Multi-Config`とAndroidのSingle-Config `Ninja`で成功する。

### 16.2 Build matrix

次の全組合せをclean Build treeで成功させる。

| Target | Configuration |
|---|---|
| Windows EditorHost／GameHost／WorkerHost | Development、Profile、Shipping、ASan |
| Windows NativeGameModule DLL／static | Development DLL、Shipping static、ASan test host |
| Android arm64-v8a | Development、Profile、Shipping、ASan test |
| Apple arm64 Engine archive／App shell | Development、Profile、Shipping、ASan test |
| Secondary clang-cl／portable Linux Clang | Compile、Unit、Static Analysis、Sanitizer対応lane |

### 16.3 Source／Tooling

- 未宣言import、cycle、Engine C++ Public Header include、Header Unit、`std.compat`が0件。
- Module interface、partition、implementation unitでwarning 0件。
- IntelliSenseまたは公式Language Service fixtureがdefinition、reference、rename、diagnostic、completionをPrimary Moduleとpartitionで成功させる。
- clang-format、clang-tidy、MSVC Static Analysis、ASan、利用可能なUBSan／TSan laneがModule Sourceを処理する。
- AI生成NativeGameModuleが`CppDependencySetV1`から生成され、Primary／secondary CompileとSource Gateへ合格する。

### 16.4 正しさ

- CX0とCX2の同一fixtureでUnit、Integration、Save、Replay、Golden image、Package、ABI boundaryの結果が一致する。
- C ABI symbol、NativeGameModule descriptor、serialized ID／field、Save／Replay schemaをModule移行だけで変更しない。
- Header版とModule版の公開宣言を比較する一時Cutover検査で欠落0件を確認し、CX3 merge時に検査とHeader版を削除する。
- source API subject集合、`CppValueTransferPolicyV1` binding集合、generated signature集合を同じContract Set、Target Profile、Toolchain lockで一対一にし、各generated output hashを同じSource contractへtraceする。
- `PointerContractManifest.bin`、`MemoryContractManifest.bin`、`PointerMemoryConsumerBindingManifest.bin`、`CppValueTransferPolicyManifest.bin`をexact四件として解決し、missing／extra／stale manifestをrejectする。
- C ABI Headerはfixed-width value、opaque handle、function table、caller-owned bufferだけを持ち、C++ reference、`std::span`、STL／PMR、`Result<T>`、exception、owner wrapperを0件にする。

### 16.5 Build性能

固定CI hardware、同一Source、同一最適化、warm-up 1回後の10回測定でCX0とCX2を比較する。

- Clean Build medianはCX0比5%を超えて悪化しない。
- Leaf implementation変更後のincremental Build medianはCX0比5%を超えて悪化しない。
- 両測定のP95はCX0比10%を超えて悪化しない。
- no-op `cmake --build --preset`、単一leaf `.cpp`変更、直接importされるModule interface変更、generated Header変更を別系列で測定し、rebuild対象数と理由をReceiptへ保存する。
- clean Buildとincremental mutation系列の最終artifact hash、generated descriptor、test結果が一致し、不要な未再Buildまたは過剰な全体再Buildを0件にする。
- compile／link／code generationを各1回中断し、同じPresetの再実行だけで正しいartifactへ収束する。
- Peak compiler process tree memoryはTool process hard cap内である。
- 計測値、Compiler trace、Module graph、Cache hit／missをBuild Performance Receiptへ保存する。

性能条件に失敗した場合もModules採用を撤回せず、CX2のまま原因を修正する。Unity Build、PCH、Header Unitで数値だけを補正しない。同一構成の完全測定cycleで3回連続して不合格となった場合は、閾値と測定方式の再評価をR4承認のADRとして起草する。再評価はModules採用と§4の一方向移行を再検討の対象にしない。5%／10%閾値の妥当性は§18項10のCX0／CX1 Build Performance Receiptを基準測定として検証し、再評価ADRの入力へ含める。

## 17. FailureとDiagnostic

| Diagnostic | 条件 | 処理 |
|---|---|---|
| `MIRAKAN-BUILD-CXX_PROFILE_INVALID` | 未定義Profileまたは許可されない組合せ | configure失敗 |
| `MIRAKAN-BUILD-DRIVER_PROFILE_INVALID` | Target／C++ ProfileとDriver／Generatorがclosed matrixに一致しない | configure前に失敗 |
| `MIRAKAN-BUILD-MAKE_GENERATOR_FORBIDDEN` | First-party targetがMakefiles系または`ndk-build`を要求 | configure前に失敗。Ninjaへの暗黙Fallbackもしない |
| `MIRAKAN-BUILD-CXX23_STABLE_REQUIRED` | Shipping要求に正式C++23 Toolchainがない | Artifact生成前に失敗 |
| `MIRAKAN-BUILD-MODULE_CYCLE` | Module graph cycle | configure失敗、cycle pathを表示 |
| `MIRAKAN-BUILD-MODULE_IMPORT_NOT_DECLARED` | Source importとMCD不一致 | Source Gate失敗 |
| `MIRAKAN-BUILD-TEXTUAL_INCLUDE_NOT_ALLOWED` | allowlist外Header | Source Gate失敗 |
| `MIRAKAN-BUILD-BMI_IDENTITY_MISMATCH` | BMI key不一致 | 該当Build treeを無効化して一度だけclean rebuild |
| `MIRAKAN-BUILD-IMPORT_STD_UNAVAILABLE` | Active Compiler／Generatorが`import std`非対応 | Profile activation失敗。HeaderへFallbackしない |
| `MIRAKAN-BUILD-MODULE_TOOLING_UNAVAILABLE` | 必須IDE／analysis fixture失敗 | CX3 Promotionを停止 |

Clean rebuild後も同じBMI errorが再発した場合は自動Retryを止め、Toolchain defectとしてBuild Receiptを失敗確定する。

## 18. Phase 0成果物

Phase 0の実装計画は次を独立taskへ分解する。

1. `CxxFrontendProfileV1`、`CppDependencySetV1`、`BuildDriverProfileV1`のMCD。
2. `mirakan_add_cpp_component()`と`CxxComponentGraphV1`生成。
3. C++23 Header bootstrap compiler policy。
4. P2564R3、P0533R9、`std::expected`、language modeのconformance fixture。
5. `mirakan.foundation`のCX1 Named Module／`import std` probe。
6. Module graph cycle、undeclared import、Header exception、Makefiles／Generator overrideのnegative fixture。
7. BMI identity／configuration isolation test。
8. `cxx26_readiness` compile-only CI。
9. Windows Ninja Multi-Config、Android Gradle→Single-Config Ninja、Apple Ninja–XcodeのCX3候補Build recipeとC ABI link fixture。
10. CX0／CX1 Build Performance Receiptと`VerificationReceiptV1` gate `mirakan.build.ninja_adoption.v1`。no-op、leaf変更、Module interface fan-out、generated Header invalidation、中断復旧、clean／incremental成果物一致を含む。
11. `CppValueTransferPolicyManifest.bin`とsource API subject／policy binding／generated signatureのset-equality static Gate。

Phase 0はCX3へ移行しない。Phase 0完了にはCX0のC++23 Development／Test／candidate Package／internal Technology Preview基盤とCX1 probeの再現可能な成功または明示的なToolchain failure Receiptが必要であり、Preview不具合を隠して成功扱いしない。CX0／CX1のReceiptをRelease Activationへ使用しない。

## 19. Definition of Done

本設計は次を満たした時に仕様として完全である。

1. C++23が全First-party CPU codeの唯一の言語基準として全正式仕様に反映されている。
2. Modules／`import std`が採用済みの最終方式として記録されている。
3. CX0、CX1、CX2、CX3の用途、遷移、Promotion可否が一意である。
4. CMake target、Named Module、NativeGameModuleの用語が混同されない。
5. Module名、Directory、CMake表現、AI dependency schemaが対応している。
6. Header例外がC ABI、macro、言語bridgeへ閉じている。
7. `BuildDriverProfileV1`がWindows、Android、Appleの入口、Generator、Configuration、Package ownerを一意にする。
8. First-party Makefiles／`ndk-build`とMake／Ninja二重対応が禁止されている。
9. AppleのNinja C++ Module buildとXcode package責務が分離されている。
10. BMIが配布ABIまたはcross-toolchain cacheとして扱われない。
11. CX3 GateがToolchain、全Target、Tooling、正しさ、性能を検証する。
12. C++26 readinessがShippingと分離されている。
13. NinjaがCMake生成C++ DAGの実行器へ限定され、Editor統合がCMake File APIとBuild Receiptを使用し、生成Ninja fileを公開契約にしていない。
14. source API subject集合、policy binding集合、generated signature集合が同じContract Set／Target／Toolchainで一対一である。
15. C ABI HeaderのC++ reference／STL typeが0件で、fixed-width value、opaque handle、function table、caller-owned bufferだけを持つ。
16. CX3 cutover時に旧Header signature、alias Module、dual generated APIを残さない。

## 20. 一次資料

- Compiler、CMake、Ninja、SDKのversion別Evidenceは[Toolchain／Dependencies](toolchain-dependencies.md)を参照する。
- [Microsoft C++ `module`／`import`／`export`](https://learn.microsoft.com/en-us/cpp/cpp/import-export-module)
- [Microsoft Header／PCH／Header Unit／Named Module比較](https://learn.microsoft.com/en-us/cpp/build/compare-inclusion-methods)
- [Microsoft Standard Library Module tutorial](https://learn.microsoft.com/en-us/cpp/cpp/tutorial-import-stl-named-module)
- [Ninja manual](https://ninja-build.org/manual.html)
- [Android CMake／Ninja configuration](https://developer.android.com/studio/projects/configure-cmake)
- [WG21 P2564R3 `consteval` needs to propagate up](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p2564r3.html)
- [WG21 P0533R9 `constexpr` for `<cmath>` and `<cstdlib>`](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2021/p0533r9.pdf)
