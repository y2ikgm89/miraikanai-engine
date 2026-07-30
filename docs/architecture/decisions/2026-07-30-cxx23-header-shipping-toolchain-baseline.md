# C++23 Header Shipping／Toolchain Baseline

- 文書ID: mirakan.decision.cxx23-header-shipping-toolchain-baseline
- 状態: review
- 正本範囲: initial V1のC++23 public surface、Named Modules adoption state、Target別Shipping frontendとminimum Windows target baselineの採用理由
- 非正本範囲: current Schema／Profile／flag／hash／Dependency pin、Build／Package／Qualification、実装Task、実装順序、担当、工数、日程、将来のNamed Modules採用。各Owner文書を参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[C++23 Language／Public Surface](../02-foundation/cpp23-modules.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)
- 外部根拠検証日: 2026-07-30
- 文書種別: Architecture Decision／language and toolchain baseline
- Decision owner document: `mirakan.arch.architecture-governance`
- Decision日: 2026-07-30
- Supersedes: none

## Context

製品レベルの初期 V1 をC++23で構築する一方、WindowsのMSVCは公式資料上C++23 modeを`/std:c++23preview`として提供し、Clangの公式statusもC++23全体を`Partial`としている。Named Modules、`import std`、header units、BMIはCompiler、standard library、Generator、IDE、analysis、cache、Packageに別の可用性と互換境界を持つ。

初期Public Contractを「HeaderからModulesへ後でcutoverする暫定段階」とすると、Shippingの成立条件が将来Toolchainへ依存し、HeaderとModuleの二surface、Preview mode、BMI配布またはTarget別fallbackを正当化しやすい。反対に「C++23対応」をCompiler名またはmode flagだけで判定すると、Engineが実際に必要とするlanguage／library featureの不足を検出できない。

## Decision drivers

1. 初期 V1 自体を完成した単一Public Contractとし、移行前提を持たせない。
2. Windows、Android、Appleで同じrequired C++23 feature semanticsを要求する。
3. Public SDKをToolchain固有BMI、Vendor型、Preview compiler modeへ依存させない。
4. Compiler全機能claimではなく、Engineが使うclosed feature setをTargetごとに検証する。
5. Toolchain、Target×Configuration、minimum OS、Dependencyを再現可能なexact profileへ束縛する。

## Considered options

### A. Initial V1からNamed Modules／`import std`をShipping prerequisiteにする

却下する。全Targetのstable frontend、standard library、Generator、IDE、analysis、sanitizer、Package、cache invalidationを一つのcurrent contractへ閉じられず、Public surfaceの成立を将来条件へ委ねる。

### B. WindowsだけMSVC Preview C++23 modeを使い、Targetごとに近い言語modeへfallbackする

却下する。Preview modeとTarget別feature差が同じPublic Contractへ混入し、同一Source／ABI／SDK claimをmode名だけで一般化できない。

### C. C++20をShipping baselineとし、C++23を部分利用する

却下する。製品のinitial language identityとSource規則が二重になり、C++23 required featureの利用可否がcall siteまたはTarget別macroへ漏れる。

### D. C++23 required feature closure＋Header-based single Shipping surface

採用する。Windows Shipping frontendはClang系、AndroidはNDK Clang、AppleはAppleClangを使い、Public Contractはself-contained Header一種とする。Named Modules、`import std`、header units、distributed BMIはinitial V1 required universe外とする。

## Decision

1. Initial V1のFirst-party CPU codeと公開C++ SDKはISO C++23 language modeを使用する。
2. C++ public surfaceはself-contained textual Header一種とする。Named Modules、`import std`、header units、BMI配布は無効であり、fallback、移行元、Technology PreviewまたはShipping prerequisiteにしない。
3. Shipping適合は[C++23 Language／Public Surface](../02-foundation/cpp23-modules.md)のclosed required feature setを、Windows、Android、Appleのpositive／negative fixtureで同じ結果にすることで判定する。Compilerの一般的な「C++23対応」claim、version文字列、mode受理だけをEvidenceにしない。
4. Windows Shipping frontendはClang 22.1.8 `clang-cl`＋`lld-link`、MSVC 14.51はWindows ABI／STL／CRTと比較laneに限定し、`/std:c++23preview`をShipping compileへ使わない。
5. AndroidはNDK r29 Clang／LLD、AppleはXcode 26.6 AppleClang／libc++をTarget baselineとする。exact flag、runtime、library、artifact、Target×Configuration policyはToolchain Ownerが所有する。
6. Windows player minimum OSはWindows 11 version 24H2、deployment version `10.0.26100.0`、x86-64とし、Host build環境と分離する。
7. Named Modulesを将来提案する場合は、初回Public release後にsuccessor ADR、new Public Surface Profile、complete consumer impact、single-surface切替、全Target qualificationを必要とする。これはcurrent V1の未完了項目ではない。

## Consequences

- 初期 V1にHeader／Moduleの二Public surface、BMI distribution、Preview language modeまたはTarget別language fallbackを持たない。
- Clang公式statusの`Partial`はShipping不可を直接意味しないが、required feature集合の一件でも不合格なら該当Targetをqualifiedにしない。
- Header方式の採用はBuild性能、Runtime性能、ABI安定性またはPackage適合の測定済みEvidenceではない。
- Toolchain profile、Header manifest、feature fixture、Build、Package、SBOM、ReceiptはRepositoryに存在せず、本Decisionは実装、lock、qualificationまたはReleaseを意味しない。

## Canonical Owner documents

- Language／feature／Public Header／Modules禁止境界: [C++23 Language／Public Surface](../02-foundation/cpp23-modules.md)
- Compiler／SDK／Dependency／Target×Configuration／minimum OS: [Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)
- Public consumer impact／将来surface変更: [Compatibility／Evolution](../02-foundation/compatibility-evolution.md)
- Native C ABI／Game module isolation: [Native Game Module](../03-authoring/native-game-module.md)
- Target package／signing／OS lifecycle: [Windows](../07-platform/windows.md)、[Android](../07-platform/android.md)、[Apple](../07-platform/apple.md)

## Supersedes／Superseded by

- Supersedes: none
- Superseded by: none

## Official or primary sources

- [ISO/IEC 14882:2024 publication](https://www.iso.org/standard/83626.html)
- [Clang C++ implementation status](https://clang.llvm.org/cxx_status.html)
- [Clang command guide](https://releases.llvm.org/22.1.0/tools/clang/docs/CommandGuide/clang.html)
- [Microsoft `/std` language mode reference](https://learn.microsoft.com/en-us/cpp/build/reference/std-specify-language-standard-version)
- [Windows 11 version 24H2 release health](https://learn.microsoft.com/en-us/windows/release-health/status-windows-11-24h2)

このDecisionは採用理由だけを記録する。current profile、fixed value、diagnostic、Gateまたはfuture predicateはOwner文書を正本とし、実装計画を定めない。
