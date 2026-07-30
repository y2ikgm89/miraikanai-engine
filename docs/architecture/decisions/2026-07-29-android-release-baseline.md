# Android Compile／Target SDK and Vulkan Profile Baseline

- 文書ID: mirakan.decision.android-release-baseline
- 状態: review
- 正本範囲: Androidのcompile／target SDK分離、required／optional Android Vulkan Profileを選ぶ判断理由
- 非正本範囲: exact version／Profile ID、Target mapping、Device Qualification、Store Gate、実装Task、実装順序、担当、工数、日程。各Owner文書を参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Android](../07-platform/android.md)、[Mobile Common](../07-platform/mobile-common.md)
- 外部根拠検証日: 2026-07-29
- 文書種別: Architecture Decision／Android release baseline
- Decision owner document: `mirakan.arch.architecture-governance`
- Decision日: 2026-07-29
- Supersedes: none

## Context

Android buildは、compile時に利用可能なSDK surface、Google Playへ宣言するtarget behavior、minimum runtime、Vulkan device coverageを別々に扱う必要がある。これらを一つの「Android API最新版」または一つのgraphics tierへまとめると、compile可能性をruntime保証と誤認し、high-end Profileをminimum device requirementへ昇格させる。

2026-07-29時点の公式資料にはAndroid Vulkan Profile 2022と2025が存在する。Miraikanaiは2D／3D共通のMobile baselineとoptional high tierを同時に必要とするが、現在のRepositoryには実device coverage、package、Qualification Receiptがない。

## Decision drivers

- current SDKでbuildできることと、公開Target behaviorを明示的に分離する。
- required graphics baselineを過度に引き上げず、optional high capabilityをexact Profileとして検証できる。
- Profile名、year、近似feature集合ではなく、Khronos／Androidが定義するexact IDを使う。
- device coverageまたはQualification未測定の値を市場対応済みと表現しない。
- OpenGL ESまたはProfile外feature setへのsilent fallbackを作らない。

## Considered options

### A. compileSdk 37／targetSdk 36、AVP 2022 required／AVP 2025 optional

採用する。compile surfaceをcurrent SDKへ合わせつつ、公開target behaviorを別のexact値へ固定する。`VP_ANDROID_vulkan_profile_2022`をminimum required、`VP_ANDROID_vulkan_profile_2025`を`mobile_high`候補とし、後者をbaseline contentまたはStore eligibilityへ要求しない。

### B. AVP 2025を全Android Deviceのrequired baselineにする

採用しない。2D／3D共通Mobile baselineのcoverageを狭める可能性があり、現在は実device／Store Device Catalog Evidenceがない。high tierとしては保持できる。

### C. compileSdkとtargetSdkを同じ値として一括更新する

採用しない。compile可能なAPI surfaceとPlatform behavior／Store declarationの意味を混同する。各値は同じToolchain ChangeSetで管理するが、同値を不変条件にしない。

### D. Vulkan feature bitの独自集合だけを保存する

採用しない。Profileのexact identity、version、公式conformance semanticsを失い、Device／Toolchain／Qualification間で別集合を同一baselineと誤認し得る。

## Decision

1. Toolchain Ownerは`compileSdk=37`、`targetSdk=36`、minimum API 29を別Fieldとして固定する。
2. Android Ownerはrequired minimumを`VP_ANDROID_vulkan_profile_2022`（AVP API version 1.1.106）、optional highを`VP_ANDROID_vulkan_profile_2025`（AVP API version 1.1.128）へ写像する。
3. Profile ID、AVP API version、loader／device probe、Target Profile、package filter、Device Qualificationのexact closureがないDeviceを合格とみなさない。
4. optional highの不成立はrequired baseline contentの失敗にしない。required baselineの不成立はquality downgradeで隠さず`UnsupportedDevice`とする。
5. 市場coverage、minimum lane、Adreno／Mali、phone／tablet／foldableの合格は実測Receiptが必要であり、本DecisionまたはOwner文書の存在だけで主張しない。

## Consequences

- ToolchainとAndroid Ownerでcompile／target／minimum APIの役割が明示される。
- high graphics capabilityはProfile ID付きで検証できるが、current Product CapabilityまたはShipping supportにはならない。
- required baselineを変更する場合はToolchain、Android Target、package filter、Device matrix、Product release acceptanceを同じArchitecture ChangeSetで更新する。
- Google Play policyまたはProfile追加を自動追従せず、公式一次資料、exact artifact、Device Evidenceを再確認する。
- 本DecisionはSchema、Build、Package、Device Fixture、Receiptの実装またはmaterializationを主張しない。

## Canonical Owner documents

- exact SDK／tool／Profile pin: [Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)
- Android Target／Runtime probe／package／device qualification: [Android](../07-platform/android.md)
- Mobile共通resource／device workflow: [Mobile Common](../07-platform/mobile-common.md)
- Decision lifecycle: [Architecture Governance](../01-governance/architecture-governance.md)

## Supersedes／Superseded by

- Supersedes: none
- Superseded by: none

## Official or primary sources

- [Google Play target API requirements](https://developer.android.com/google/play/requirements/target-sdk)
- [Android Vulkan Profiles](https://developer.android.com/ndk/guides/graphics/android-vulkan-profile)
- [Khronos Vulkan Profiles repository](https://github.com/KhronosGroup/Vulkan-Profiles/tree/main/profiles)
