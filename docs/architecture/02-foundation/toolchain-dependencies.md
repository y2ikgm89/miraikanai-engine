# Miraikanai Engine Toolchain／Dependencies

- 文書ID: mirakan.arch.toolchain-dependencies
- 状態: review
- 正本範囲: 外部Tool・SDK・Library・APIのexact version／release／commit、artifact size、hash／integrity、license、取得元、Toolchain lock、Dependency採用・更新Gate、Build Driver Profileのclosed set、CI実行基盤のrunner／hosting／capacity binding
- 非正本範囲: Product scope、Subsystem API・型・Budget、Runtime phase、Platform lifecycle、Dependency内部を包むEngine-owned Adapter契約。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core architecture](core-architecture.md)、[Executable contracts](executable-contracts.md)、[C++23 modules](cpp23-modules.md)、[Project Shader](../06-rendering/project-shader.md)
- 外部根拠検証日: 2026-07-23

## 1. 結論と一意所有

本文書はMiraikanai Engineが採用する外部Tool、SDK、Library、APIのversion、release、commit、artifact size、hash、integrity、license、取得URLを決める唯一の正本である。他のArchitecture仕様はDependency名と本文書へのLinkだけを記載し、固定値を複写しない。

ここにある値は2026-07-21に再検証した初期baselineであり、floatingな「最新」ではない。Vendorの最新推奨とMiraikanaiの採用判断を区別し、更新は本書、lock、CI image、SBOM、Receiptを一つのToolchain更新ChangeSetで変更する。

## 2. Target toolchain baseline

### 2.1 Windows

| 項目 | exact pin／判断 |
|---|---|
| Host OS | Windows 11 25H2 x64、OS build 26200以上。lock比較はbuild 26200、UBR 8875 |
| Target minimum OS（player実行環境） | 未固定。D3D12 Agility SDK 1.619.4／Enhanced Barriers必須要件と整合するWindows buildを更新ChangeSetでexact lockする。Host OS行（ビルドホスト）と区別し、混同しない |
| Visual Studio Build Tools | 2026 18.8.0 Stable、build 12009.203 |
| Primary compiler | MSVC Build Tools v14.51 x64/x86、`cl.exe`／`link.exe` 14.51.36231以上。CX0だけ`/std:c++23preview`、Shipping不可 |
| CX3 compiler condition | PreviewでないMSVC v14.52以降で正式`/std:c++23`を提供したreleaseを更新ChangeSetでexact lockする。未固定状態ではCX3を有効化しない |
| Secondary compiler | LLVM／clang-cl 22.1.8。Windows Shipping ABIはMSVCで統一 |
| Windows SDK | 10.0.26100.8249 |
| CMake | 4.4.0 |
| Generator／executor | Ninja Multi-Config、Ninja 1.13.2 |
| vcpkg registry | release 2026.06.24、builtin baseline `cd61e1e26a038e82d6550a3ebbe0fbbfe7da78e3` |
| Shader compiler | DXC tag v1.9.2602.24、commit `d355aa8364d34df3f0822ba0de8d1dfc75ae6f48` |
| Windows graphics runtime | Microsoft.Direct3D.D3D12 1.619.4、SDKVersion 619 |

MSVC v14.51には、third-party headerの`#include`後に`import std`を行うC++23 module partitionのbuildがC1001 internal compiler errorになる既知のcompiler bugがあり、修正はv14.52 Previewで告知済みである。CX1 probeは該当構成のnegative／compile fixtureを含め、回避不能なProject構成をCX0へ留める。同一translation unitでの標準C++ headerの`#include`と`import std`の混在は公式に禁止されている（C wrapper headerは除く）。

AI Source Worker隔離Backend `HyperVIsolatedWorkerV1`（[AI Security／Approval](../01-governance/ai-security-approval.md) §7.1）のHost要件は次のとおりとする（[Microsoft公式Hyper-V要件](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/host-hardware-requirements)による）: Hyper-Vを有効化できるWindows edition（Windows 10／11 ProfessionalまたはEnterprise。Homeでは有効化不可）、SLAT対応64-bit CPU、VM Monitor Mode extensions、BIOS／UEFIでの仮想化支援（Intel VT／AMD-V）とhardware DEP（XD／NX）有効化、最低4 GB RAM、およびGeneration 2 guestのSecure Bootを提供できるHyper-V platform。Host要件を満たさない環境（Home等）では同§7.1の「同等のremote hardware-VM Worker」を標準経路とし、非隔離fallbackを行わない。

### 2.2 Android

| 項目 | exact pin／判断 |
|---|---|
| compile／target SDK | Android API 36 |
| minimum SDK | API 29 |
| NDK | r29、29.0.14206865 |
| Android Gradle Plugin | 9.3.0 |
| Gradle | 9.5.0 |
| SDK Build Tools | 36.0.0 |
| JDK | Microsoft Build of OpenJDK 17.0.19 LTS |
| C++ build | CMake 4.4.0、Single-Config Ninja 1.13.2、`externalNativeBuild.cmake` |
| AndroidX Games | GameActivity 4.4.2、Controller 2.0.2、Frame Pacing 2.1.3 |
| ABI／STL | Shipping `arm64-v8a`、Developmentは必要なCIだけ`x86_64`を追加、全native dependencyを`c++_shared`へ統一 |

AGPのNDK既定値は採用根拠にしない。公式downloadで別version指定が許されることを確認したうえで、Miraikanaiは上表のNDKを明示固定する。

minimum SDK API 29は技術baselineであり、市場coverageの達成主張ではない。Google Playの[Target API要件](https://developer.android.com/google/play/requirements/target-sdk)は2026-08-31以降の新規app／updateへAndroid 16（API 36）以上のtargetを要求するが、minimum SDKを決めない。API 29の継続可否は[Android §5](../07-platform/android.md#5-device-testsfailurerelease-gate)の`AndroidMinSdkCoverageReceiptV1`で別途判定し、Receiptなしに「十分な端末をカバーする」と表現しない。

### 2.3 Apple

| 項目 | exact pin／判断 |
|---|---|
| Unsigned Build host | macOS Tahoe 26.2以降、arm64 |
| Xcode | 26.6 Stable |
| SDK | iOS／iPadOS 26.5 |
| Deployment target | 17.0 |
| C++ build | CMake 4.4.0。CX0はXcode Generator、CX1 ProbeはNinja Multi-Config 1.13.2、CX2–CX3はNinja C++ archiveとXcode App shell／最終link／archive |
| Shipping route | `AppleShippingRouteV1`のallowed tupleだけ。`{ build_driver_ref: driver.apple.xcode-cloud, delivery_profile_ref: none }`または`{ build_driver_ref: driver.apple.modules-ninja-xcode, delivery_profile_ref: delivery-profile.apple.self-hosted-split }` |

### 2.4 Graphics／shader

| Target | exact external baseline |
|---|---|
| Windows | Direct3D 12、Agility SDK 1.619.4、portable HLSL 2021、DXIL Shader Model 6.6、Root Signature 1.1、Enhanced Barriers必須 |
| Android | portable HLSL 2021をDXCでSPIR-Vへoffline compileし、Vulkan 1.1、Android Vulkan Baseline 2022 profile（`VP_ANDROID_baseline_2022`、要求Vulkan 1.1.106）、SPIRV-Tools validationを通す |
| Apple | portable HLSL 2021をDXCのSPIR-V intermediate、SPIRV-CrossのMSLへ変換し、iOS／iPadOS SDK 26.5のMetal compilerでmetallibをoffline生成 |

Apple経路（DXC→SPIR-V→SPIRV-Cross→MSL）は、ray系／mesh／amplification Stageの変換成立可否が未検証である。検証Ownerは[Project Shader](../06-rendering/project-shader.md)のOwnerとし、同§4のとおり検証完了までAppleの当該Stageは既定unsupportedとする。検証タスクと、代替経路（Metal Shader Converter等のDXIL→metallib経路）の採用可否検討は§9のDependency採用・更新Gateの検討事項として登録し、採用時のexact version／hashは検証時に`toolchain.lock.json`で固定する。

`ShaderCompilerProfileV1`はEngine buildが生成するTarget別closed Profileであり、次を固定する。

```text
schema_version
compiler_profile_id
target_profile_id
source_language_profile_id
compiler_lock_ref
translator_lock_refs[]
validator_lock_refs[]
generated_argument_set_hash
stage_profile_map
capability_map_hash
matrix_layout
clip_depth_convention
texture_coordinate_convention
binding_layout_policy_id
reflection_profile_id
source_map_profile_id
optimization_profile_id
debug_information_profile_id
```

Project、AI、Build scriptはargument、entry profile、register／space、optimization、layout、extension、validatorを上書きしない。Development／Profile／Shipping差はEngine-owned Profile IDで選び、Source directiveで切り替えない。DXCのpublic compile result、PDB、hash、reflection APIを安定Adapter境界として使い、hidden `-ast-dump` textはversion固定の破棄可能な解析Cacheに限定する。

### 2.5 External schema／AI protocol

| 項目 | exact pin／判断 |
|---|---|
| Internal validation dialect | JSON Schema Draft 2020-12 |
| MCP Tool protocol | Model Context Protocol 2025-11-25 |
| OpenAI API／SDK | Responses API、official TypeScript SDK 6.48.0 |
| Anthropic API／SDK | 未固定。MVPのAnthropic系接続は[Executable contracts](executable-contracts.md)のMCP経路（Claude CLI／Desktop→Miraikanai MCP Server）だけとし、direct Provider projection用`provider_profile`はexact pinを確定するDependency ChangeSetまで作成しない |
| 初期評価Model | `gpt-5.6-sol`、reasoning effort `medium` |

OpenAIはGA Modelでも最短6ヶ月通知で退役させるため、Model IDをOrchestrator codeへ埋め込まず、Toolchain lockの設定値としてだけ保持し、退役通知の検出をDependency更新Gateの起動条件へ含める。Responses API固有機能への依存はProvider Adapter一箇所へ閉じる。

## 3. Build Driver matrix

| Driver Profile | Target／Frontend | Configure driver | C++ Generator | Package owner |
|---|---|---|---|---|
| `driver.windows.cmake-ninja-multi` | Windows／CX0–CX3 | `cmake_preset` | `Ninja Multi-Config` | Windows Platform owner |
| `driver.android.gradle-ninja` | Android／CX0–CX3 | `gradle_external_native_build` | `Ninja` | Gradle |
| `driver.apple.cx0-xcode` | Apple／CX0 | `cmake_preset` | `Xcode` | Xcode |
| `driver.apple.modules-probe-ninja` | Apple／CX1 | `cmake_preset` | `Ninja Multi-Config` | なし、Promotion不可 |
| `driver.apple.modules-ninja-xcode` | Apple／CX2–CX3 | `cmake_preset` | `Ninja Multi-Config` | Xcode |
| `driver.apple.xcode-cloud` | Apple／CX2–CX3 Shipping | `xcode_cloud_workflow` | `Ninja Multi-Config`＋`Xcode` | Xcode Cloud |

本matrixは`BuildDriverProfileV1`のDriver Profile IDと許可組合せのclosed setの唯一の正本である。`AppleShippingRouteV1`は`build_driver_ref`とnullableな`delivery_profile_ref`を持ち、§2.3の二tupleだけを許可する。Xcode Cloud routeはchecked-in `ci_scripts`でNinja C++ archiveを作成してXcode App shellへ渡し、managed signing／TestFlight handoffをXcode Cloudへ委譲するため`delivery_profile_ref = none`とする。self-hosted routeだけが`driver.apple.modules-ninja-xcode`と`delivery-profile.apple.self-hosted-split`を組み合わせる。Driver IDとDelivery Profile IDを同じenumへ格納しない。First-party TargetでMakefiles系、raw Makefile、Android `ndk-build`を禁止する。Windows／Appleの通常入口はchecked-in Preset、Androidの通常入口は固定Gradle Wrapperとする。Target、Frontend Profile、Driver、Generator、toolchain hashが異なるBuild tree、object、BMI、log、Receiptを共有しない。

## 4. Tool artifact lock

| Tool | 取得元 | size／hash／integrity |
|---|---|---|
| Visual Studio Build Tools 18.8.0 | [fixed-version bootstrapper](https://download.visualstudio.microsoft.com/download/pr/e05c0bc8-d058-4b2b-937c-1c80073d7633/b62e8829c6a6c043aacf2ef657456213ab71099c7e46a610f95d6778bfc9beb0/vs_BuildTools.exe) | 5,687,056 bytes、SHA-256 `b62e8829c6a6c043aacf2ef657456213ab71099c7e46a610f95d6778bfc9beb0`、ProductVersion 18.8.0、FileVersion 18.8.12009.203 |
| CMake 4.4.0 | [Windows x64 zip](https://cmake.org/files/v4.4/cmake-4.4.0-windows-x86_64.zip) | 54,388,920 bytes、SHA-256 `156d70eb7625a7b469444df7d0861d2af8d5d0a437fce32c350372b08f5620e8` |
| Ninja 1.13.2 | [Windows zip](https://github.com/ninja-build/ninja/releases/download/v1.13.2/ninja-win.zip) | 291,570 bytes、SHA-256 `07fc8261b42b20e71d1720b39068c2e14ffcee6396b76fb7a795fb460b78dc65` |
| LLVM 22.1.8 | [Windows installer](https://github.com/llvm/llvm-project/releases/download/llvmorg-22.1.8/LLVM-22.1.8-win64.exe) | commit `ca7933e47d3a3451d81e72ac174dcb5aa28b59d1`、455,545,840 bytes、SHA-256 `16e5709785fef73c854646241c4a92c5cd574318d1b33c63330dd7721903e55c` |
| DXC v1.9.2602.24 | [release zip](https://github.com/microsoft/DirectXShaderCompiler/releases/download/v1.9.2602.24/dxc_2026_05_27.zip) | 27,108,038 bytes、SHA-256 `cf658aacf070d3045e31b8f1f8a696c2945f37c1095019481ef7c513368db3b4` |
| Node.js 24.18.0 LTS／npm 11.16.0 | [Windows x64 zip](https://nodejs.org/dist/v24.18.0/node-v24.18.0-win-x64.zip) | 37,176,245 bytes、SHA-256 `0ae68406b42d7725661da979b1403ec9926da205c6770827f33aac9d8f26e821` |
| TypeScript 7.0.2 | npm `typescript@7.0.2` | 365,612 bytes、integrity `sha512-8FYau96o3NKOhbjKi/qNvG/W5jhzxkbdm5sj9AbZ/5T5sWqn3hJgLfGx27sRKZWTvyzCP8dLRBTf5tBTSRVUNA==` |
| OpenAI TypeScript SDK 6.48.0 | npm `openai@6.48.0` | commit `ee5bce84fccb97135948a4838255804d4af1b7dd`、1,707,934 bytes、integrity `sha512-KhVp+FyV50QrXNextvL9hIU5l6ox5HYuKQjGVk7lIqprgJol90+dQXWONV6S1lRWsKA1bXjrow8RsUT14M1hNA==` |
| jsonc-parser 3.3.1 | [npm package](https://www.npmjs.com/package/jsonc-parser/v/3.3.1) | commit `3c9b4203d663061d87d4d34dd0004690aef94db5`、27,354 bytes、integrity `sha512-HUgH65KyejrUFPvHFPbqOY0rsFip3Bo5wb4ngvdi1EpCYWUQDC5V+Y7mZws+DLkr4M//zQJoanu1SP+87Dv1oQ==`、MIT |
| canonicalize 3.0.0 | [npm package](https://www.npmjs.com/package/canonicalize/v/3.0.0) | commit `aba9209d044f2729c51141d8a73b11e80816e42c`、6,020 bytes、integrity `sha512-yYLfHyDMIXRyRqsKBRLX023riFLpXY2YOfdtqKXZRZy9qsfOJ9U+4F9YZL7MEzL5+ziN2x2nlBvY/Voi3EBljA==`、Apache-2.0 |
| RFC 8785 JCS fixture | [official testdata at pinned commit](https://github.com/cyberphone/json-canonicalization/tree/19d51d7fe467d4706a3ff08adf8a748f29fc21e0/testdata) | commit `19d51d7fe467d4706a3ff08adf8a748f29fc21e0`、6組12 file／1,476 bytes、fixture root SHA-256 `49ebd08bec39f4da9e2db03cffc76b2de984912fd6fbc66ec4ee33852b7b84fb` |
| Microsoft OpenJDK 17.0.19 | [Windows x64 zip](https://aka.ms/download-jdk/microsoft-jdk-17.0.19-windows-x64.zip) | 186,907,952 bytes、SHA-256 `394d1d8253d58b462300f15f9c81369478cf8813f82dca914c3b5dfdef080f9f` |
| TLA+／TLC CLI 1.7.4 | [tla2tools.jar](https://github.com/tlaplus/tlaplus/releases/download/v1.7.4/tla2tools.jar) | 2,274,532 bytes、SHA-256 `936a262061c914694dfd669a543be24573c45d5aa0ff20a8b96b23d01e050e88` |

MSVCとWindows SDKは固定bootstrapperからoffline layoutを作り、resolved `VCToolsVersion`、実行binary、STL、SDK fileのsize、version、SHA-256、signerをlockする。Android SDK／NDK／Gradle、Apple Xcode／SDKも同じ粒度でresolved file manifestを保存する。

## 5. JavaScript／AI toolchain boundary

公式JavaScript toolchainはNode.js 24.18.0 LTS archiveと同梱npm 11.16.0の一組である。利用rootを`orchestrator/`、`tools/contract_compiler/`、`tools/contract_lint/`だけに限定し、各`package.json`は`private=true`、ES module、exact `engines`、`packageManager=npm@11.16.0`を要求する。PATH上のglobal Node、npm、Corepack、Bun、pnpm、Yarnを正規Buildへ混在させない。

通常Buildは事前充填したcontent-addressed cacheに対し`npm ci --ignore-scripts --offline --no-audit --no-fund`を実行する。install／prepare scriptを必要とするDependencyは専用ADR、exact package hash、閉じたscript allowlist、隔離Dependency Buildを先に承認する。

TypeScript 7.0.2は上記許可root（Orchestrator、contract compiler、contract lint）のcompileとlanguage-service CLIだけに使い、安定公開されていないprogrammatic compiler APIへ製品codeを依存させない。正式Artifactはstrict、single-threaded clean buildで生成する。OpenAI接続はResponses APIと公式TypeScript SDK 6.48.0を使い、strict function callingのSchema制約は[Executable contracts](executable-contracts.md)が所有する。

## 6. External Dependency baseline

| 分類 | Dependency | exact release／commit | License | 採用範囲 |
|---|---|---|---|---|
| Graphics | Microsoft.Direct3D.D3D12 | 1.619.4／SDKVersion 619 | package同梱`LICENSE.txt`／`LICENSE-CODE.txt` | Agility runtime、D3D12 header、Enhanced Barriers |
| Graphics | Microsoft.Direct3D.WARP | 未固定。lock ChangeSetでexact version／nupkg hashを確定 | package同梱license（lock ChangeSetで確認・固定） | app-local WARP redistributable。deterministic Development reference、性能Gate対象外 |
| Input | Microsoft.GameInput（Windows GameInput SDK） | 未固定。Microsoft.GameInput NuGetの正規経路から更新ChangeSetでexact version／hashを検証・固定 | 未固定（package同梱licenseを更新ChangeSetで確認・固定） | GameInput header、redistributable runtime DLL |
| Graphics | D3D12MA | v3.2.0／`1d86c1130f61453634b1df85782e1fecfd59a525` | MIT | D3D12 heap suballocation、budget stats |
| Graphics | Vulkan Memory Allocator | v3.3.0／`1d8f600fd424278486eade7ed3e877c99f0846b1` | MIT | Vulkan heap suballocation、budget、defrag primitive |
| Graphics | SPIRV-Cross | Vulkan SDK 1.4.350.0／`1a6169566c73d3da552748fc372fe2bbb856e46e` | Apache-2.0 | SPIR-V reflection、MSL生成 |
| Graphics | SPIRV-Tools | v2026.2／`0539c81f69a3daeb706fd3477dca61435b475156` | Apache-2.0 | SPIR-V validation、offline optimization |
| Image | KTX-Software | v4.4.2／`4d6fc70eaf62ad0558e63e8d97eb9766118327a6` | Apache-2.0 | Offline KTX／ASTC処理 |
| Image | DirectXTex | may2026／`4feb3e11a020f35b796fc769a74216a555d4f5ef` | MIT | Offline texture decode、mipmap、BC encoding |
| Audio | Oboe | 1.10.0／`a81bb9f87d4105b84b682685d3bfbb5beca371d1` | Apache-2.0 | Android low-latency audio stream |
| Audio | libopus | 1.6.1／source SHA-256 `6ffcb593207be92584df15b32466ed64bbec99109f007c82205f0194572411a1` | BSD-3-Clause | Streaming music／voice decode |
| Audio | libFLAC | 1.5.0／source SHA-256 `f2c1c76592a82ffff8413ba3c4a1299b6c7ab06c734dee03fd88630485c2b920` | Xiph.org BSD | Import Worker内のFLAC decode／metadata |
| Android | AndroidX Games GameActivity | 4.4.2 | Apache-2.0 | Activity／GameTextInput bridge |
| Android | AndroidX Games Controller | 2.0.2 | Apache-2.0 | Controller bridge |
| Android | AndroidX Games Frame Pacing | 2.1.3 | Apache-2.0 | Frame pacing bridge |
| Physics | Box2D 3.1.1 | v3.1.1／`8c661469c9507d3ad6fbd2fea3f1aa71669c2fe3` | MIT | private 2D collision／solver kernel |
| Physics | Jolt Physics 5.6.0 | v5.6.0／`e77f175595e64cb44218cc9d9d56fc365ad0e36a` | MIT | private CPU 3D collision／solver kernel |
| Navigation | Recast／Detour 1.6.0 | v1.6.0／`6dc1667f580357e8a2154c28b7867bea7e8ad3a7` | zlib | private 3D navmesh build／query kernel、32-bit `dtPolyRef` |
| Animation | ozz-animation | 0.16.0／`6cbdc790123aa4731d82e255df187b3a8a808256` | MIT | Skeleton compression、sampling、blend primitive |
| Text | HarfBuzz | 14.2.1／`56feae4035bdd48f62ba2b8d8c16232d4d89b3a4` | MIT | OpenType shaping |
| Text | FreeType | 2.14.1／`3bd82b5f543bc84ccf2b1d0cdb63b95218099ee6` | FreeType License | OTF／TTF validation、glyph metric／rasterization |
| Text | ICU4C | 78.3／`21d1eb0f306e1141c10931e914dfc038c06121da` | Unicode-3.0 | BCP 47、BiDi、boundary、plural／number／date／message format |

XAudio2はWindows SDK headerとOS in-box runtimeで提供され、本表への別途pinを持たない。GameInputのredistributable runtime DLLはMSIX同梱方式（§2.1のTarget minimum OSがin-box提供かredist必要かの判定を含む）を[Windows](../07-platform/windows.md)のpackage節が所有し、本書はexact version／hash／license／取得元だけを所有する。両者の未固定項目は更新ChangeSetで確定するまで採用Featureの実装開始Gateを通過できない（§6.1と同じ規律）。

全DependencyをEngine-owned Adapterへ隔離し、Vendor型を公開Contractや永続formatへ出さない。Box2D、Jolt、Recastは該当Capabilityの候補実装へのexact hash lockを提供するだけである。Activation stateは[Product Plan](../00-product/product-plan.md)のRegistryが`{capability_id, target_id}`行単位で所有し、本文書の記載や更新でActivationを昇格しない。Joltは`JPH_USE_DX12=OFF`、`JPH_USE_VK=OFF`、`JPH_USE_MTL=OFF`、`JPH_USE_CPU_COMPUTE=OFF`としてCPU rigid-body kernelだけをbuildする。

### 6.1 必須だが未固定のDependency

次の実装能力は他正本が要求するが、採用Dependencyまたは第一party実装の決定が未固定である。各行は9節のGateとADRで確定するまで`未固定`とし、確定するまで該当Featureは[Core architecture](core-architecture.md)のFeature実装開始Gateを通過できない。

| 要求能力 | 要求元 | 状態 |
|---|---|---|
| C++側の厳格JSON parser（duplicate field、invalid UTF-8拒否） | [Executable contracts](executable-contracts.md) §17.1 | 未固定 |
| JSON Schema Draft 2020-12検証器（contract compiler手順のvalidate） | [Executable contracts](executable-contracts.md) §14 | 未固定 |
| SHA-256実装（Runtimeのhash検証、canonical hash） | [Core architecture](core-architecture.md) §11、[Executable contracts](executable-contracts.md) §13 | 未固定 |
| C++ unit test framework | [Core architecture](core-architecture.md) §12 | 未固定 |
| MCP server実装SDK | [Executable contracts](executable-contracts.md) §16.2 | 未固定 |

第一party自作を選ぶ場合も9節のGateと同等の検証（公式test vector等）をADRへ記録し、本表の行をexact pinまたは実装先Directoryへ置換する。

### 6.2 Known unresolved decision queue

本queueは未固定値の正本を複製せず、最初のconsumer、必要Evidence、fail-closed境界を索引する。Waveは準備順であり、各行のTriggerが期限である。Triggerが先に到来した行は即時に前倒しする。Architecture Evolution Control Planeはlock済みNode.js／TypeScript CLIと標準Libraryだけを使い、新しいproduction dependencyを要求しないため、本queueのC++／Platform行を理由に`wp.architecture.control-plane`を停止しない。

| Wave | Unresolved input | Decision authority | Trigger／deadline | Required closure evidence | Blocked consumer／current state |
|---|---|---|---|---|---|
| A | C++ unit test frameworkまたはfirst-party harness | `mirakan.arch.toolchain-dependencies` | Runtime ECS Task 3で最初のC++ contract testを書く前 | §9のrelease／license／hash／Adapter確認、CTest discovery、failure／filter／parallel実行fixture。自作時は実装Directoryとself-test Receipt | `wp.runtime.ecs-e0`のC++ test着手を停止／未固定 |
| A | SHA-256 implementation | `mirakan.arch.toolchain-dependencies` | Runtime ECS Task 4のcanonical hash実装前 | exact実装、license／artifact lock、NIST known-answer、incremental／zero-length／large input、cross-target byte一致Receipt | `wp.runtime.ecs-e0`のhash実装を停止／未固定 |
| A | Microsoft.Direct3D.WARP | `mirakan.arch.toolchain-dependencies` | D3D12 Task 3 WARP conformance着手前 | Microsoft公式NuGetのexact version、nupkg SHA-256、同梱license、Agility互換、Development-only conformance Receipt | `wp.runtime.d3d12-backend` Task 3を停止／未固定 |
| A | Windows Target minimum OS | `mirakan.arch.toolchain-dependencies` | D3D12 Task 12 package binding前 | Agility／Enhanced Barriers／GameInput提供形態、Microsoft support lifecycleを満たすexact build、OS probe、package negative fixtureを持つADR | Windows package promotionを停止／未固定 |
| A | Microsoft.GameInput | `mirakan.arch.toolchain-dependencies` | Windows Input Adapter実装またはpackage同梱判定前 | Microsoft公式NuGetのexact version、nupkg SHA-256、license、header／runtime DLL manifest、minimum OS別in-box／redist matrix、Input conformance Receipt | `wp.product.editor-runtime-windows`のInput／package Gateを停止／未固定 |
| B | 厳格C++ JSON parser | `mirakan.arch.toolchain-dependencies` | [Executable contracts](executable-contracts.md) §17.1を使う最初のC++ acceptance path実装前 | exact release／license／hash、duplicate field、invalid UTF-8、trailing bytes、number overflow、depth／size boundのpositive／negative Receipt | 最初のC++ JSON consumerだけを停止／未固定 |
| B | JSON Schema Draft 2020-12 validator | `mirakan.arch.toolchain-dependencies` | [Executable contracts](executable-contracts.md) §14の最初のruntime validator実装前 | exact release／license／hash、official Draft 2020-12 test suite、unknown dialect／unsupported keyword／recursive ref／bound failure Receipt | 最初のruntime schema consumerだけを停止／未固定 |
| B | MCP server SDKまたはfirst-party server boundary | `mirakan.arch.toolchain-dependencies` | `wp.product.external-agent`実装前 | MCP 2025-11-25 conformance、transport／capability negotiation、message bound、cancel／disconnect、license／artifact lock。自作時は実装Directoryとprotocol fixture | Phase 5 external-agent WPを停止／未固定 |
| B | Android minimum SDK market coverage | `mirakan.arch.platform-android`、閾値承認は`mirakan.arch.product-plan` | `wp.platform.mobile-offline`開始前 | [Android §5](../07-platform/android.md#5-device-testsfailurerelease-gate)の`AndroidMinSdkCoverageReceiptV1` | Android Target Gateを停止／Play Console Evidence待ち |
| C | CX3 stable MSVC cutover | `mirakan.arch.toolchain-dependencies` | CX3 activation proposal前 | Microsoft stable release、正式`/std:c++23`、resolved toolset hash、`import std`／module partition C1001 regression fixture、全Target CX0↔CX3 ABI Receipt | CX3だけを停止しCX0を維持／非Preview v14.52+待ち |
| C | Anthropic direct API／SDK | `mirakan.arch.toolchain-dependencies` | direct Provider projectionがProduct WPへ登録された時 | official SDK exact version／integrity／license、Provider version、Schema keyword conformance、credential／error／rate-limit fixture | direct projectionだけを停止しMVPはMCP経路を使用／scope未登録 |
| Per target | CI runner／GPU・macOS host／mobile device poolとcapacity owner | `mirakan.arch.toolchain-dependencies` | 対応するProduct Target Gate開始前 | §8.1のqualified `CiExecutionProfileV1`、non-`unfixed` Owner、runner image／Toolchain lock／device matrix／capacity Receipt | 該当laneを`diagnostic.toolchain.ci-capacity-unresolved`で停止／Owner・capacity input待ち |

各行のclosureは「候補名を本文へ書く」ことではない。§9を満たすADRまたはMeasurement Receipt、exact Toolchain lock、SBOM／notice、positive／negative fixtureを同じChangeSetでread-backできた時だけ未固定の正本行を置換する。候補調査だけでFeature Gateを開かない。

## 7. Source artifact、license、取得先

| Dependency／artifact | 公式取得・Release根拠 | 追加lock |
|---|---|---|
| Agility SDK package | [NuGet package](https://www.nuget.org/packages/Microsoft.Direct3D.D3D12/1.619.4)、[flat-container artifact](https://api.nuget.org/v3-flatcontainer/microsoft.direct3d.d3d12/1.619.4/microsoft.direct3d.d3d12.1.619.4.nupkg) | 35,169,986 bytes、SHA-512 `6a275381027ed758714eedf1ccaeea446b1d9afeddc1f6b6bbc3c85939ef9ffd02b7fae780cd50da635b66b09f5fce99535788551cd64e3663b9e59fe6f7d9de` |
| SPIRV-Cross source | [official repository](https://github.com/KhronosGroup/SPIRV-Cross) | vcpkg source SHA-512 `f4f9f62a9ff15e9b707b820ce603bda1ea9fe7138bf505307791e55058063d9362e9bba6e508f5d302836a53b51e115b03b9ce7478fbc7b86a4b266b426eaa5d` |
| Box2D | [v3.1.1 release](https://github.com/erincatto/box2d/releases/tag/v3.1.1) | tagとcommitを照合 |
| Jolt Physics | [v5.6.0 release](https://github.com/jrouwe/JoltPhysics/releases/tag/v5.6.0) | tagとcommitを照合 |
| Recast Navigation | [v1.6.0 release](https://github.com/recastnavigation/recastnavigation/releases/tag/v1.6.0) | annotated tagとcommitを照合 |
| D3D12MA | [v3.2.0 release](https://github.com/GPUOpen-LibrariesAndSDKs/D3D12MemoryAllocator/releases/tag/v3.2.0) | source archive SHA-512とlicense hashをoverlay portで固定 |
| VMA | [v3.3.0 release](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator/releases/tag/v3.3.0) | source archive SHA-512とlicense hashをoverlay portで固定 |
| SPIRV-Tools | [v2026.2 release](https://github.com/KhronosGroup/SPIRV-Tools/releases/tag/v2026.2) | source archive SHA-512とlicense hashをoverlay portで固定 |
| KTX-Software | [v4.4.2 release](https://github.com/KhronosGroup/KTX-Software/releases/tag/v4.4.2) | source archive SHA-512とlicense hashをoverlay portで固定 |
| Oboe | [1.10.0 release](https://github.com/google/oboe/releases/tag/1.10.0) | source archive SHA-512とlicense hashをoverlay portで固定 |
| libopus | [official downloads](https://opus-codec.org/downloads/) | 上表のsource SHA-256とlicense file hashを固定 |
| libFLAC | [1.5.0 release](https://github.com/xiph/flac/releases/tag/1.5.0) | 上表のsource SHA-256とlicense file hashを固定 |
| ozz-animation | [0.16.0 release](https://github.com/guillaumeblanc/ozz-animation/releases/tag/0.16.0) | source archive SHA-512とlicense hashをoverlay portで固定 |
| HarfBuzz | [14.2.1 release](https://github.com/harfbuzz/harfbuzz/releases/tag/14.2.1) | source archive SHA-512とlicense hashをoverlay portで固定 |
| FreeType | [official downloads](https://freetype.org/download.html) | source archive SHA-512とFreeType License file hashをoverlay portで固定 |
| ICU4C | [78.3 release](https://github.com/unicode-org/icu/releases/tag/release-78.3) | filtered data hashとlicense hashを固定 |
| DirectXTex | [may2026 release](https://github.com/microsoft/DirectXTex/releases/tag/may2026) | source archive SHA-512とlicense hashをoverlay portで固定 |
| AndroidX Games | [official release notes](https://developer.android.com/jetpack/androidx/releases/games) | Google Maven artifact checksum、POM、licenseをGradle dependency verificationへ固定 |
| JSON Schema／MCP | [JSON Schema Draft 2020-12](https://json-schema.org/draft/2020-12)、[MCP Tools 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25/server/tools) | Dialect／protocol versionをToolchain lockへ記録 |
| OpenAI | [official SDKs](https://developers.openai.com/api/docs/libraries)、[model reference](https://developers.openai.com/api/docs/models/gpt-5.6-sol) | npm integrityとModel IDをToolchain lockへ記録 |

`third_party/ports`はexact source archive SHA-512、patch hash、compiler option、license file hashを固定する。CIはresolved source commitとこれらをSBOM、third-party notice、Build manifestへ出力する。Builtin portが別commitを指しても暗黙追従せずoverlay portを使う。

## 8. Toolchain lock contract

Repository rootの`toolchain.lock.json`はschema version 6とし、未知Field、`null`、重複ID、相対URL、version range、`latest`、wildcard、複数hash候補を拒否する。Profile、artifact、Driver、resolved fileをIDまたは正規化relative pathのunsigned UTF-8 byte順に保存し、canonical JSONのSHA-256をBuild manifestへ記録する。

| Field | 規則 |
|---|---|
| `lock_schema_version` | `uint32`、値6 |
| `profiles[].profile_id` | `target.windows.desktop`、`target.android.mobile`、`target.apple.mobile`を各一件。Product Plan Target Profile Registryと同じlogical ID |
| `profiles[].profile_version` | `uint32`、初期値1。logical IDへversionを埋め込まない |
| `profiles[].host` | OS、architecture、minimum version、CI image digest |
| `profiles[].target.deployment_target` | Target minimum OSのcanonical numeric string。値は各Profileの正本行（`target.apple.mobile`は[§2.3 Apple](#23-apple)のDeployment target行、`target.android.mobile`は[§2.2 Android](#22-android)のminimum SDK行、`target.windows.desktop`は[§2.1 Windows](#21-windows)のTarget minimum OS行）と一致する。正本行が未固定のProfileではField自体を拒否 |
| `profiles[].artifacts[]` | tool ID、exact version、source／resolved URL、size、SHA-256、source commit |
| `profiles[].resolved_files[]` | tool ID、relative path、size、file version、SHA-256、signer |
| `profiles[].build.cxx_bindings[]` | Frontend、language standard、compiler flag／full version、STL hash、CMake ID、experimental token、BMI policy、CRT mapping |
| `profiles[].build.driver_bindings[]` | Driver Profile hash、tool IDs、toolchain file hash、driver config hash |
| `ci_execution_profiles[]` | Verification laneとrunner／hosting／toolchain／device capacityのbinding。§8.1に従う |
| `shared.npm` | Node／npm exact version、許可root、lockfile SHA-256、package version／tarball URL／size／integrity |
| `shared.vcpkg.builtin_baseline` | `cd61e1e26a038e82d6550a3ebbe0fbbfe7da78e3` |

Windows installerはSHA-256に加えてAuthenticode chainとPublisher、GitHub artifactはrelease API digest、tag commit、取得後hash、npm packageはlockfileのintegrityを照合する。不一致はconfigure前に失敗させる。

schema version 5から6へのoffline migrationは次のexact mappingだけを許可する。旧IDをruntime aliasとして保持せず、migration後はold IDを拒否する。

| schema 5 `profile_id` | schema 6 `profile_id` | 追加Field |
|---|---|---|
| `windows_desktop_v1` | `target.windows.desktop` | `profile_version=1` |
| `android_mobile_v1` | `target.android.mobile` | `profile_version=1` |
| `apple_mobile_v1` | `target.apple.mobile` | `profile_version=1` |

### 8.1 CI execution profile

[AI Verification／Provenance §14](../01-governance/ai-verification-provenance.md#14-ci-lanes)がlaneの意味、Trigger、必須Evidenceを所有し、本書はlaneを実行するinfrastructure bindingだけを`CiExecutionProfileV1`として所有する。

| Field | Rule |
|---|---|
| `lane_id` | Verification正本のexact lane ID |
| `runner_class` | `windows_gpu`、`windows_hardware_vm`、`macos_build`、`android_device`、`apple_device`、`portable_linux`のclosed enum |
| `hosting_mode` | `managed`または`self_hosted` |
| `toolchain_profile_id` | `target.*`。native laneは`profiles[].profile_id`、`portable_linux`のarchitecture／JavaScript laneだけは`target.headless.host`を参照 |
| `device_matrix_ref` | physical deviceを使う`android_device`／`apple_device`で必須、他classでは`null` |
| `capacity_state` | `unfixed`、`qualified`、`unavailable`のclosed enum |
| `owner` | 調達、credential、patch、quota、保守、incident対応の責任主体。未決定時はliteral `unfixed` |

Entry identityは`{lane_id, runner_class, toolchain_profile_id}`のtupleとし、重複を拒否する。`capacity_state=qualified`はrunner image hash、Toolchain lock hash、isolation profile、同時実行上限、retention、device matrix（該当時）、fresh Qualification Receiptが揃う場合だけ許可する。`unfixed`または`unavailable`、`owner=unfixed`、Receipt失効、device欠落ではlane開始を`diagnostic.toolchain.ci-capacity-unresolved`で拒否し、local runner、別OS、別device、managed／self-hosted間へ暗黙fallbackしない。

ユーザーがrunner契約、self-hosted host、実機pool、担当Ownerをまだ指定していないため、本文書はcapacityや費用を推測しない。Productの見積りは[Product Plan §5.1](../00-product/product-plan.md#51-開発体制見積りrisk-contract)の`team_assumption_state=unfixed`を維持し、必要laneが`qualified`になるまで該当product target gateを開始しない。

## 9. Dependency採用・更新Gate

新規採用と更新は専用ChangeSetで次をすべて満たす。

1. 公式Project／Vendor、exact release／commit、license、取得URLを確認する。
2. Download size、cryptographic hash、signerまたはpackage integrityを記録する。
3. Adapter境界、公開型非露出、allocator、exception、threading、coordinate、determinismをconformance testする。
4. clean／incremental／cancel recovery、sanitizer、replay、serialized fixture、performance、Package inspectionを再実行する。
5. SBOM、third-party notice、overlay port、CI image、Toolchain lock、Build Receiptを同時更新する。
6. 旧版と新版を恒久併存させず、全Targetに同じChangeSetでCutoverする。
7. 不合格時は別versionや別Backendへ暗黙fallbackせず、原因、API差分、破棄条件をADRへ記録する。

## 10. Context7と公式一次資料

Context7で次のIDを2026-07-21～2026-07-22に解決し、指定queryで確認した。

| 対象 | Context7 ID | query／確認結果 |
|---|---|---|
| CMake | `/kitware/cmake` | C++ Module scanのGenerator、`import std`、`CXX_MODULE_STD`を照会。Ninja／Ninja Multi-ConfigとVisual Studio Generatorがscanを扱い、`import std`はNinja系に限定され、CMake 4.4時点でもExperimental token（`CMAKE_EXPERIMENTAL_CXX_IMPORT_STD`）を必要とすることを確認 |
| Microsoft C++ | `/microsoftdocs/cpp-docs` | named moduleと`import std`を照会。標準header include／header unitとの混在禁止をBuild graph制約として扱う |
| Box2D | `/erincatto/box2d` | C17、opaque ID handle、substep solver、multithreadingを照会。private Adapter採用と整合 |
| Jolt Physics | `/jrouwe/joltphysics` | compute backend build optionとmultithread integrationを照会。GPU backendを無効化する採用判断と整合 |
| Recast Navigation | `/recastnavigation/recastnavigation` | Recast buildとDetour runtime queryの分離を照会。private Backend境界と整合 |
| DirectX Shader Compiler | `/microsoft/directxshadercompiler` | `IDxcResult`のobject／PDB／shader hash／reflection出力と`IDxcUtils::CreateReflection`を照会。Fact Graphの安定証拠をpublic APIへ限定する判断と整合 |
| Unreal Engine 5.8 | `/websites/dev_epicgames_unreal-engine_5_8` | 収録範囲が概説中心だったため、RDG parameter／resource validationとGlobal ShaderはEpic公式5.8本文で補完 |
| Unity Graphics | `/unity-technologies/graphics` | HLSL function reflectionによるShader Graph Node生成とURP RenderGraph resource宣言を照会。Project Module／Technique分離の比較根拠に使用 |

直接参照する公式資料は[CMake C++ modules](https://cmake.org/cmake/help/latest/manual/cmake-cxxmodules.7.html)、[MSVC `import std` tutorial](https://learn.microsoft.com/en-us/cpp/cpp/tutorial-import-stl-named-module)、[Android 16 KiB page size](https://developer.android.com/guide/practices/page-sizes)、[OpenAI deprecations](https://developers.openai.com/api/docs/deprecations)、[OpenAI function calling](https://developers.openai.com/api/docs/guides/function-calling)、[Responses migration](https://developers.openai.com/api/docs/guides/migrate-to-responses)とする。Context7結果は検索補助であり、規範判断はリンク先の公式本文で再確認する。

Context7の内容はmain branchの挙動説明であり、exact release／commitの証拠は上記公式Release pageとtagで補完した。

Shader toolchainの補完一次根拠は[DXC v1.9.2602.24 release](https://github.com/microsoft/DirectXShaderCompiler/releases/tag/v1.9.2602.24)、[DXC API](https://github.com/microsoft/DirectXShaderCompiler/wiki/Using-dxc.exe-and-dxcompiler.dll)、[DXC HLSL options](https://github.com/microsoft/DirectXShaderCompiler/blob/main/include/dxc/Support/HLSLOptions.td)、[HLSL Specification Working Draft](https://microsoft.github.io/hlsl-specs/specs/index.html)である。Working Draftまたはmain branchの変化をBuild時に自動採用せず、上表のDXC tag／commitと`ShaderCompilerProfileV1`を実行正本にする。

CMakeのversion別根拠は[C++ Modules support](https://cmake.org/cmake/help/v4.4/manual/cmake-cxxmodules.7.html)、[`CXX_MODULE_STD`](https://cmake.org/cmake/help/v4.4/prop_tgt/CXX_MODULE_STD.html)、[Presets](https://cmake.org/cmake/help/v4.4/manual/cmake-presets.7.html)、[Ninja Multi-Config](https://cmake.org/cmake/help/v4.4/generator/Ninja%20Multi-Config.html)、[File API](https://cmake.org/cmake/help/v4.4/manual/cmake-file-api.7.html)で補完した。MSVCのbaselineとCutover条件は[stable release](https://devblogs.microsoft.com/cppblog/msvc-version-1451-available/)、[C++23 support status](https://devblogs.microsoft.com/cppblog/c23-support-in-msvc-build-tools-14-51/)、[Visual Studio release history](https://learn.microsoft.com/en-us/visualstudio/releases/2026/release-history)を一次根拠とする。v14.51の`import std`／module partition既知bug（C1001）と修正予定は[MSVC Build Tools Preview updates July 2026](https://devblogs.microsoft.com/cppblog/msvc-build-tools-preview-updates-july-2026/)、`import std`と`#include`の混在禁止は[Import the standard library with modules](https://learn.microsoft.com/en-us/cpp/cpp/tutorial-import-stl-named-module)を一次根拠とする。Android Vulkan profileの正式名は[VP_ANDROID_baseline_2022](https://github.com/KhronosGroup/Vulkan-Profiles/blob/main/profiles/VP_ANDROID_baseline_2022.json)を一次根拠とする。

AndroidはContext7に公式AGP資料がなかったため、[AGP 9.3.0 release notes](https://developer.android.com/build/releases/agp-9-3-0-release-notes)と[NDK downloads](https://developer.android.com/ndk/downloads)へフォールバックした。前者でGradle 9.5.0、Build Tools 36.0.0、JDK 17の互換条件、後者でNDK r29 revision 29.0.14206865を確認した。

Apple pinは[Xcode 26.6 release notes](https://developer.apple.com/documentation/Xcode-Release-Notes/xcode-26_6-release-notes)、[Xcode support matrix](https://developer.apple.com/support/xcode/)、Build分離は[Xcode Cloud security](https://developer.apple.com/xcode-cloud/security/)を一次根拠とする。

## 11. 明示的に採用しないもの

- floating version、range、`latest`、Preview artifactのShipping採用
- 複数Compiler、Generator、JavaScript runtime、package manager、lockfileの恒久併存
- Vendor sourceの手動copy、hashなしdownload、Build中のNetwork取得
- License未確認、SBOM未記載、public APIへVendor型を露出するDependency
- Toolchain不一致時のPATH fallback、別Generator fallback、旧version fallback
- Slang、GLSL、WGSL等の追加Source frontendを初期Project Shader baselineへ併設すること。採用する場合はDependency ChangeSet、言語意味差、全Target artifact／reflection、既存HLSL fixture、rollbackを独立Qualificationする
