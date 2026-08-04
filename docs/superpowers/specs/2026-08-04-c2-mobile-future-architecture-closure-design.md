# C2／Mobile／Future Architecture Closure Design

- 文書状態: approved design
- 正本性: non-normative design rationale
- 対象: Product C2のexact Host／runtime Target／locale／publication boundary、MobileのProduct tier境界、Future membership／Target closure
- 決定: 既存Owner chainを維持しつつscope外のFuture Owner割当を是正し、C2をWindows／Android／Appleの同一候補へ閉じ、MobileをProduct C2へ統合し、Futureを30件のplanning-only inventoryへクリーンに更新する
- 承認: 推奨案を2026-08-04にユーザー承認
- 実装状態: absent and unchanged
- 実装計画状態: intentionally absent
- Materialization状態: Product Definition、Target／Locale Profile、MCD、Registry、Project、Package、Fixture、Receipt、Publicationはabsent
- 外部根拠確認日: 2026-08-04

## 1. Status

本設計はArchitecture文書の追記、改善、更新、修正だけを扱う。C++、Shader、Asset、実行Schema、Registry instance、Fixture、Receipt、Build system、CI、実装Task、Work Package、Phase、順序、工程、工数、担当または日程を作成しない。

本書は採択理由とOwner routingだけを記録する非規範設計入力である。型、fixed value、required set、Product boundary、Future membership、GateまたはEvidence meaningはリンク先のArchitecture Ownerだけが所有する。

## 2. Problem

C1 First Playable boundaryのclosure後、C2、Mobile、Futureを別単位で監査した結果、次の不整合を確認した。

- Product PlanはC2を2D＋3Dのsame-release claimとする一方、exact Host、runtime Target、locale、public publication routeを固定していない。
- Native Game ModuleはC2でWindows／Android／Apple packageを要求する一方、Android／AppleとProject ShaderはMobile Project Sourceを後続Futureのように扱う。
- Mobile CommonはProduct C1がWindows-onlyであるにもかかわらず、`C1／C2 Mobile`、`2D C1`、`3D C1`というdomain-local語彙をProduct tierと区別していない。
- `ProjectMobileSpecV1`はRuntime generationをdeny-onlyとしながら、存在しないContent Safety Profileを必須Refにしている。
- Future 31件と25 `single_target`／6 `target_role_bundle`のclosureを閉じた監査記録とADRが残る一方、current Product Planにはcanonical membershipとTarget closureが存在しない。
- 旧31件のうちMobile native／shader source qualificationはcurrent C2 contractへ移っており、Futureに残すと同じProduct requirementをC2とFutureへ二重分類する。
- first-party local inference／managed external Hostのproduction route、persistence／live-service／moderation、shipping Runtime structured-data generationをAI Securityが一括所有する旧割当は、AI SecurityのAuthorization／Trust／Credential正本範囲を越える。後二件には専用domain Owner自体が未採択である。
- Future consumerが非規範Execution Proposalの`FutureToActivePromotionManifestV1`をcurrent canonical carrierとして参照し、receipt-free Product boundaryと未materialize execution schemaを混同できる。
- affected OwnerへPhase／Work Package名が漏れ、Product boundaryと非規範execution proposalを混同できる。

## 3. Considered approaches

### A. 旧31件のoperational RegistryをProduct Planへ復元する

却下する。実行Registry、promotion manifest、作業単位をProduct Ownerへ戻し、現行Governanceが禁止するArchitecture／execution混同を再導入する。Mobile native／shaderのC2 requirementとも衝突する。

### B. Closure Reviewだけを修正する

却下する。Closure Reviewは非規範Evidenceであり、C2 target set、Mobile tier、Future membershipの正本にはなれない。

### C. Existing Owner chain clean correction

採用する。Product PlanがC2 Product selectionとFuture membership／closure、Lifecycleがsame-candidate Evidence集約、Mobile CommonがMobile共通Target意味、Android／Apple／WindowsがPlatform package／publication、Native Game Module／Project ShaderがProject Source Target qualificationを所有する。Futureはreceipt-freeな30件のplanning-only inventoryだけをProduct Planへ戻し、execution Registryを作らない。Authoring AI routeはAI Production Orchestration、Authorization／Trust／CredentialはAI Security、artifact選定はToolchainへ分離する。専用domain OwnerのないProduct-level composite FutureはProduct Planがplanning identityだけを暫定所有し、promotion revisionで専用Ownerを採択するまでdomain contractを存在扱いしない。

## 4. Canonical Product boundary

### 4.1 C2

Product C2はC1へ機能を順次足す工程名ではない。exact build／authoring Hostは`target.headless.host@1`と`target.windows.editor@1`、runtime Targetは`target.windows.desktop@1`、`target.android.mobile@1`、`target.apple.mobile@1`、localeは`en-US`と`ja-JP`とする。各runtime Targetで2Dと3Dの両Referenceを同じEngine release／candidateへ閉じ、Project C++とProject Shaderを含む公開authoring surface、fallback、package、clean launch、privacy、license、supportを相互代用なしに成立させる。

公開publication baselineはWindowsのMicrosoft Store＋`package-profile.windows.msix`、AndroidのGoogle Play＋`package-profile.android.play`、AppleのApp Store＋`package-profile.apple.bundle`である。internal MSIX、Play test track、TestFlight、uploadまたはreview approvalはqualification inputであり、public publicationの代用ではない。`package-profile.apple.managed-assets`はminimum OSが異なる別variantであり、C2 baselineへ混在させない。offline completionに必要なReference contentはinstall-timeで完結させる。

この対象集合はMiraikanaiのproject-decisionであり、Microsoft、GoogleまたはAppleが三Platform同時release、locale、Reference contentまたはGame scopeを推奨したという意味ではない。

### 4.2 Mobile

Product C1はWindows-onlyであり、Mobileが初めてProduct claimへ参加するtierはC2である。Mobile側のCapability成熟度をProduct C1／C2と同名で表現しない。Android／Appleの2Dと3Dは同じGameplay、Save、Input Action、content identityを使い、Target差はadaptive layout、qualified presentation fallback、package、lifecycle、device budgetへ限定する。

Mobile C2はRuntime generationを`deny_all`、Content Safety Profileを不在に固定する。positive structured-data generationはFuture promotion時の新しいProduct Definition／Mobile spec revisionでのみ導入し、現在のV1へnullableな隠し有効化経路を置かない。

### 4.3 Future

Future inventoryはMobile native／shader source qualificationをC2へ移した30件とする。24件は`single_target`、persistence、small co-op、rollback、large-session、MMO、managed external Hostの6件は`target_role_bundle`である。全件は`planning_only`／`not_activated`で、C1、C2またはthird-party product releaseのrequired／optional hidden memberではない。

30件中29件のdirect promotion candidateはPortfolio構造上、事前のFuture ID分解を要求しないという分類であり、promotion-readyを意味しない。新しいActive Product Definition revisionで昇格する際はOwner contract、public boundary、exact Targetまたはrole bundle、Security、Privacy、Legal／License、support、fallback、same-scope Evidence requirementを同時に確定する。`future.capability.persistence-live-service-moderation-operations`と`future.capability.runtime-structured-data-generation`は専用domain Ownerも同じArchitecture revisionで採択する。`future.capability.unrestricted-project-scripting-runtime`だけは直接昇格禁止の`decomposition_required` subjectで、相互排他的な原子Futureへclean replacementした後に各新IDを別々に判断する。旧ID alias、umbrellaのActive Capability化、migration row、empty Moduleまたはplaceholder APIを作らない。

## 5. Official-source boundary

外部一次資料はPlatform factsだけに使用する。

- Microsoftのcurrent distribution guidanceは、多くのDeveloperにMicrosoft Storeを推奨し、MSIX submissionにStore signing／hosting／updateを提供する。
- Google Playは2026-08-31から新規app／updateへAndroid 16／API 36以上を要求する。Android公式資料は16 KiB ELF alignment、adaptive orientation／window、AVP profile選択、AAB texture targetingを定義する。
- Appleは2026-04-28からApp Store Connect uploadへiOS／iPadOS 26 SDK以上を要求し、Xcode 26.6のSDK／deployment support範囲を公開している。Managed Background AssetsはiOS／iPadOS 26以上の別delivery surfaceで、TestFlight／App Store経路を持つ。

MiraikanaiのAPI minimum、AVP 2022 baseline、ASTC＋ETC2、deployment target 17.0、C2 target set、Store選択、install-time content、managed-assetsをC2 baselineから分離する判断はproject-decisionである。「公式推奨」は外部資料が実際に推奨する事実にだけ使用し、Miraikanai固有scopeへ転用しない。

## 6. Clean-break rule

現Repositoryにpublic materializationがないため、削除する`future.capability.mobile-project-native-shader-source-qualification`へalias、supersession row、dual reader、migration、compatibility profileを設けない。Android／AppleのProject C++／Project ShaderはC2 requirementとしてのみ読む。旧Future operational Registryは非規範proposalのまま正本へ昇格せず、Product Planはexecution Schemaを所有しない。

## 7. Document changes

| Document | Change |
|---|---|
| Product Plan | exact C2 Host／Target／locale／publication、2D＋3D／Project Source non-substitution、30件Future inventory、24／6 closure、domain Owner未採択のcomposite planning identityを正本化 |
| Product Lifecycle | C2 exact selectionを再定義せず、Manifest／Acceptanceでsame-candidate set equalityを要求 |
| Mobile Common | Product C1／C2混同、Content Safety必須Ref、Phase語彙、2D／3D縮退の曖昧さを除去 |
| Android／Apple／Windows | C2 public distribution、official fact／project decision、Project Shader Target bindingを整合 |
| Native Game Module／Project Shader | Windows qualification baselineとProduct C1／C2を分離し、Phase／Work Package参照を除去 |
| Advanced Rendering／Multiplayer Ownership ADR | review中の31件判断を30件／24＋6へ更新し、Mobile source qualificationのC2移管を記録 |
| Product Execution Registry Proposal | Product Planをexecution Registry schema Ownerとする誤記とobsolete Mobile Future参照だけを訂正 |
| AI Production Orchestration／AI Security／Toolchain／AI Provider Supplement | Authoring AI route、Authorization／Trust、artifact選定を分離し、存在しない旧subsection／Registry／Phase参照を除去 |
| Animation／Multiplayer Authority | 未materialize Promotion Manifest依存を除き、Product Plan §8と新Active Product Definition revisionの境界へ統一 |
| Architecture Plan Closure Review | `ARCH-C150`としてC2／Mobile／Future post-C1 auditとcurrent evidence gapを追跡 |

## 8. Acceptance

1. C2のHost、runtime Target、locale、2D／3D、Project C++／Shader、public publication routeがexactに一意である。
2. Product C1にMobileを含めず、Mobileの最初のProduct claimをC2として一意に読める。
3. C2 2D／3DでGameplay意味をTarget都合に変更せず、presentation fallbackだけをOwner qualificationへ委譲する。
4. Runtime generation deny-onlyとContent Safety Profile不在が矛盾しない。
5. Future inventoryが30 unique ID、24 `single_target`、6 `target_role_bundle`で、Ownerとdirect prerequisiteが一意かつOwner正本範囲に整合し、構造上directな29件、domain Owner未採択2件、decomposition-required 1件を混同しない。
6. C2 Mobile source qualificationとFuture membershipに重複ID、alias、migrationまたはhidden optional branchがない。
7. official-spec factとMiraikanai project-decisionを混同しない。
8. 実装、実装計画、Work Package、Phase、Fixture、materialized Registry、Receipt、Qualification、ActivationまたはPublicationの存在を追加・主張しない。
