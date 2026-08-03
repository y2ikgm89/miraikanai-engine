# Miraikanai Engine Toolchain／Dependencies

- 文書ID: mirakan.arch.toolchain-dependencies
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: 外部Tool・SDK・Library・APIの調査済み候補version／release／commit、artifact size、hash／integrity、license、取得元、C++23 Shipping compiler、Target×Configuration Build Policy、Toolchain lock要件、Dependency採用・更新Gate、Build Driver Profile、CI実行基盤要件
- 非正本範囲: Product scope、Subsystem API・型・Budget、Runtime phase、Platform lifecycle、Dependency内部を包むEngine-owned Adapter契約。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Product Plan](../00-product/product-plan.md)
- 関連文書: [glTF Import Dependency Baseline Decision](../decisions/2026-08-03-gltf-import-dependency-baseline.md)、[MCP Current Protocol Baseline Decision](../decisions/2026-08-03-mcp-current-protocol-baseline.md)、[AI Production Orchestration Ownership Decision](../decisions/2026-08-04-ai-production-orchestration-ownership.md)、[Architecture Plan Closure Review](../appendices/architecture-plan-closure-review.md)、[Product Plan](../00-product/product-plan.md)、[Product Lifecycle](../00-product/product-lifecycle.md)、[Product Security](../01-governance/product-security.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[AI Production Orchestration](../03-authoring/ai-production-orchestration.md)、[Core architecture](core-architecture.md)、[Executable contracts](executable-contracts.md)、[Memory／Pointers](memory-pointers.md)、[C++23 Language／Public Surface](cpp23-modules.md)、[Project Shader](../06-rendering/project-shader.md)、[Virtualized／Continuous Geometry](../06-rendering/virtualized-continuous-geometry.md)、[C++23 Header Shipping／Toolchain Baseline Decision](../decisions/2026-07-30-cxx23-header-shipping-toolchain-baseline.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-08-03

## 1. 結論と一意所有

本文書はMiraikanai Engineが評価する外部Tool、SDK、Library、APIのversion、release、commit、artifact size、hash、integrity、license、取得URLと、採用時に必要なlock条件を所有する。外部tag／releaseと公式互換表を確認済みであっても、Repositoryへlock、SBOM、取得Artifact、Build Receiptが存在しないDependencyを「採用済み」「locked」「qualified」と表現しない。他のArchitecture仕様はDependency名と本文書へのLinkだけを記載し、固定値を複写しない。

[Product Lifecycle](../00-product/product-lifecycle.md)は本書が生成するexact Toolchain closure、SBOM、license／third-party notice sourceをEngine release、Package、Documentation、User向けNOTICE presentationへ束縛するconsumerであり、dependency version、license判断、SBOM生成を所有しない。[Product Security](../01-governance/product-security.md)はDependency subject、vulnerability case、affected／fixed releaseへsame artifactを束縛するconsumerであり、stale SBOMまたは別release lockからunaffectedを推測しない。

2026-08-03時点のmaterialization状態は次のとおりである。

| 対象 | 状態 | 解釈 |
|---|---|---|
| Version／tag／release／upstream commit | `source-checked` | 公式release、tag、一次資料との一致を確認した候補baseline |
| Download artifact size／SHA-256 | `source-checked`または`provisional` | 各rowの記載に従う。Repository内のcurrent lockを意味しない |
| `toolchain.lock.json` | `absent` | 本文中のFieldとProfileは将来lockに必要な設計であり、生成済みSchemaではない |
| SBOM／license bundle | `absent` | dependency採用完了を主張しない |
| Build／Package／Target Qualification Receipt | `absent` | 互換性、Build成功、Shipping対応を主張しない |

以降の「lockする」「記録する」は、採用判断後に必要となるArtifact要件を表す。現存するlockまたは実装手順を表さない。

ここにある値は各行と外部Evidence read-backで2026-07-21～2026-08-03に再検証した初期baselineであり、floatingな「最新」ではない。Vendorのcurrent release／仕様とMiraikanaiの採用判断を区別し、更新は本書、lock、CI image、SBOM、Receiptを一つのToolchain更新ChangeSetで変更する。

## 2. Target toolchain baseline

### 2.1 Windows

| 項目 | exact pin／判断 |
|---|---|
| Host OS | Windows 11 25H2 x64、OS build 26200以上。lock比較はbuild 26200、UBR 8875 |
| Target minimum OS（player実行環境） | Windows 11 version 24H2、canonical deployment version `10.0.26100.0`、x86-64。MSIX MinVersionとruntime probeを同値にし、build 26100未満を起動前に拒否する。Host OS行と区別する |
| Visual Studio Build Tools release | Visual Studio 2026 18.8.2 Stable（official-spec） |
| Miraikanai Build Tools bootstrapper candidate | ProductVersion 18.8.2、FileVersion 18.8.12023.21の固定URL／hash候補（project-decision／provisional）。`12023.21`をVendor公式release versionまたは公式推奨buildとして扱わない |
| C++23 Shipping frontend | LLVM `clang-cl` 22.1.8、`/clang:-std=c++23`。Clang公式statusがC++23をPartialとするため、言語modeだけで完全適合を主張せず[C++23 Language](cpp23-modules.md)のrequired feature setを全件probeする |
| Linker | LLVM `lld-link` 22.1.8 |
| Windows ABI／STL／CRT | MSVC Build Tools v14.51、MSVC STL／UCRT／VCRuntime 14.51.36231以上、Windows x64 MSVC ABI。`cl.exe`／`link.exe`はABI comparison／diagnostic laneでありinitial V1 Shipping frontend／linkerではない |
| Secondary conformance lane | MSVC v14.51 `/std:c++23preview`はnon-Shipping compile probeだけ。Preview object、library、PDB、PackageをShippingへ昇格しない |
| Windows SDK | 10.0.26100.8249 |
| CMake | 4.4.1 |
| Generator／executor | Ninja Multi-Config、Ninja 1.13.2 |
| vcpkg registry | release 2026.06.24、builtin baseline `cd61e1e26a038e82d6550a3ebbe0fbbfe7da78e3` |
| Shader compiler | DXC tag v1.9.2602.24、commit `d355aa8364d34df3f0822ba0de8d1dfc75ae6f48` |
| Windows graphics runtime | Microsoft.Direct3D.D3D12 1.619.4、SDKVersion 619 |

#### Editor visual asset baseline

ここで固定するのはC0のsource／usage判断であり、C1のasset lock完了ではない。`target.windows.editor`のvisual assetはHost OSが供給するUI font、bundled code font、build-timeに変換するicon sourceを別々に扱う。system fontをRepositoryやPackageへ再配布せず、Host OS image上で解決されたfileだけを`resolved_files[]`へ記録する。

| Role | C0固定値 | C1 asset lockの必須条件 |
|---|---|---|
| UI label／Japanese prose | Windows 11 25H2の`Segoe UI Variable`（Regular 400／Semibold 600）をLatin第一選択、`Yu Gothic UI`をJapanese fallbackとする | DirectWriteが選ぶfont family／file version／file SHA-256、Japanese＋Latin混在baseline・truncation・copy fixture。Microsoft system fontはbundleしない |
| code／log／ID／path／numeric editor | `Noto Sans Mono CJK JP` 2.004のstatic OTF、`NotoSansMonoCJKjp-Regular.otf`と`NotoSansMonoCJKjp-Bold.otf`だけをbundle候補とする。sourceは`Sans2.004` tag／commit `523d033d6cb47f4a80c58a35753646f5c3608a78`、SIL Open Font License 1.1 | `06_NotoSansCJKjp.zip`、二font file、LICENSEのsize／SHA-256、coverage、bundle manifest、notice、DirectWrite load／fallback fixture。Windows上で注意書きのあるCFF2 variable fontを代用しない |
| command／status icon | Fluent UI System Icons 1.1.334、tag commit `f2f75a6e4814153d5c049c0f06e197731718326b`、MIT。Regularを通常、Filledをselected／activeだけに使う | `EditorIconTokenContractV1`のWidget／Panel／Command presentationにあるEditor-owned system icon consumer集合、approved source archive／selected SVGのSHA-256、LICENSE／NOTICE、icon ID allowlist、16／20 logical-size conversion output hash。runtime SVG／CSS parserを導入しない |

Windows system fontの見た目はHost OS buildだけでなくresolved fileに依存するため、Host OS名だけをvisual reproducibilityの根拠にしない。bundled OTFとiconのsource取得・hash照合に失敗した場合、fallback assetを黙って採用せず`diagnostic.toolchain.editor-visual-assets-unlocked`でReference Design fixtureをfail closedにする。

Initial V1はNamed Modules／`import std`を使用しないため、MSVC Preview mode、Module partition bugまたは将来の非Preview flagをShipping成立条件にしない。MSVC ABI／STL／CRT file setとclang-cl／lld-linkの組合せはexact lock、public ABI fixture、exception／RTTI境界、PDB／crash symbol、clean packageで別途検証する。

AI Source Worker隔離Backend `HyperVIsolatedWorkerV1`（[AI Security／Approval](../01-governance/ai-security-approval.md) §7.1）のHost要件は次のとおりとする（[Microsoft公式Hyper-V要件](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/host-hardware-requirements)による）: Hyper-Vを有効化できるWindows edition（Windows 10／11 ProfessionalまたはEnterprise。Homeでは有効化不可）、SLAT対応64-bit CPU、VM Monitor Mode extensions、BIOS／UEFIでの仮想化支援（Intel VT／AMD-V）とhardware DEP（XD／NX）有効化、最低4 GB RAM、およびGeneration 2 guestのSecure Bootを提供できるHyper-V platform。Host要件を満たさない環境（Home等）では同§7.1の「同等のremote hardware-VM Worker」を標準経路とし、非隔離fallbackを行わない。

##### C1 Editor visual asset lock closure

C1の「Editor visual asset lock」は、新しい`EditorVisualAssetLockV1`、top-level lock Field、Registry、Capabilityではない。既存`target.windows.editor` Profileの`host`、`artifacts[]`、`resolved_files[]`、および既存`style_font_generation`から導く閉包集合の名称である。font family名、OS build名、icon名、source repository名だけをidentityにせず、次表の役割ごとのinputとEvidenceを同じProfileへ束縛する。

| 閉包member | 既存の正本／格納先 | C1で固定・照合するもの |
|---|---|---|
| Host UI typography | `target.windows.editor`の`host`＋`resolved_files[]` | CI OS image digest、`ja-JP`のJapanese prose／ASCII labelに対するDirectWrite選択family・face・weight・fallback順、resolved fileのrelative path・file version・SHA-256・signer。system fontをbundleせず、別OS imageの同名familyまたは「近い」fallbackへ読み替えない |
| bundled code typography | `profiles[].artifacts[]` | Noto archive、Regular／Bold static OTF、LICENSEのsource commit・size・SHA-256・coverage・bundle manifest・notice。CFF2 variable font、別Noto family、Host fontによる代替を許さない |
| icon geometry | `profiles[].artifacts[]` | `EditorIconTokenContractV1`のexact consumer集合（全`WidgetPatternContractV1.icon_consumer_refs[]`、全`EditorPanelDescriptor.icon_token_ref`、全`EditorCommandPresentationDescriptorV1.presentation_icon_token_ref`を§15.2の`IconConsumerRefV1`へcanonical展開）→approved source icon ID／Regular・Filled variant／allowed usage contextのexact binding、selected SVG hash、converter ID／exact version／option、16／20 logical-sizeごとのconversion output hash。変換物はsemantic colorを焼き込まないgeometry／alpha sourceとし、unknown icon、emoji、runtime SVG／CSS parserへfallbackしない |
| reference capture hardware boundary | outer [Editor Reference Environment Profile](../appendices/editor-ui-design-system-catalog.md#1582-environment-profileとdeterministic-clock)と[Performance／Capacity §8.1](../04-runtime/performance-capacity.md#81-editor-reference-01-performance-profile)の`ReferenceHardwareProfileV1` | resolved Host UI fontが使うCI OS image digestはselected hardware profileの`os_image_ref`とexact一致する。GPU driver package、monitor topology／EDID、power planはPerformance Ownerだけがprofileに保持し、`target.windows.editor`のartifact hashまたは`style_font_generation`へ複写しない。outer Environment ProfileがA／B hardware ref／hashを欠く、OS imageが不一致、driver／EDIDを別Hardwareから補う場合はvisual captureを開始しない |
| theme color mapping | 既存Environment Profileのtheme／system-color snapshotと[Editor UI Framework §13.2.2](../03-authoring/editor-ui-framework.md#1322-windows-appearance-change)／§15.1 | `theme.editor.light@1`／`theme.editor.dark@1`のexact token profileとE07–E10のWindows system-color snapshot ref／hash。LightをDarkの反転またはOS accentから生成せず、snapshotは`HCF_HIGHCONTRASTON`とrequired current system colorsを含み、asset自身の色、scheme名、既知paletteでHigh Contrastを代用しない。system colorとshape／icon／text labelの複合表現を使う |
| provenance | `artifacts[]`、SBOM、third-party notice | source archive／tag commit／license file／conversion outputのchain、bundled OTF／icon notice、system fontを再配布していないこと。hashが未固定の候補をlock済みと表示しない |
| invalidation | 既存`style_font_generation` | font file、font fallback resolution、icon source、converter、conversion outputの一つでも差があればgenerationを変え、旧layout、glyph cache、draw packet、visual snapshotを再利用しない |
| runtime theme／color invalidation | Windows Adapterのexisting appearance transaction | `WM_THEMECHANGED`／`WM_SYSCOLORCHANGE`／`WM_SETTINGCHANGE`ごとにHigh Contrast flagとcurrent system colors、またはnon-contrast `UISettings.GetColorValue(UIColorType::Foreground)`と判定済みLight／Dark modeをstable snapshotとして再読込し、old color-bearing draw packet、brush、tinted icon instance、visual snapshotを破棄する。desktop appでunsupportedな`UISettings.ColorValuesChanged`をsubscription／fallbackに使わない。font／icon assetが不変なら`style_font_generation`は変えず、runtime color changeをasset lock更新またはstatic E07–E10／E13の再利用で隠さない |
| runtime motion preference | Windows Adapterのexisting motion transaction | first top-level HWND生成前にprocess-scoped `UISettings`の`AnimationsEnabledChanged`をsubscriptionし、initial readと同event／全`WM_SETTINGCHANGE`後に`SystemParametersInfoW(SPI_GETCLIENTAREAANIMATION)`をprocess-wideで再読込する。`TRUE=full`／`FALSE=reduced`のexact snapshot refをEnvironment Profileへ束縛し、`UISettings.AnimationsEnabled`値は第二の判定sourceにしない。`FALSE`ではvisual-only effectを次present前にfinal stateへ解決し、`SPI_SETCLIENTAREAANIMATION`、registry／Settings write、app-local override、old snapshotの再利用をしない。font／icon assetが不変なら`style_font_generation`を変えず、E11 static baselineをrunning-process transitionの代用にしない。event subscription failureはcaptureを開始しない |

| runtime transparency preference | Windows Adapterのexisting advanced-effects transaction | 同じprocess-scoped `UISettings`から`AdvancedEffectsEnabled`をinitial readし、`AdvancedEffectsEnabledChanged`をsubscriptionしてexact snapshot refをEnvironment Profileへ束縛する。C1は`effects.editor.opaque-only@1`を固定し、propertyが`true`でもper-pixel transparency、backdrop、Mica、Acrylic、blurをactivateしない。`false`、High Contrast、read／event subscription failureではopaque representationを維持し、captureを開始しない。font／icon assetが不変なら`style_font_generation`を変えず、C1のstatic false profileまたはE14を追加しない。materialは別Dependency／policy／fallback／profile ChangeSetなしに有効化しない |
| runtime scrollbar preference | Windows Adapterのexisting scroll-chrome transaction | same process-scoped `UISettings`でfirst top-level HWND生成前に`AutoHideScrollBarsChanged`をsubscriptionし、initial readとworker-thread callbackごとにUI session refreshへqueueして`AutoHideScrollBars`を再読込する。`FALSE= persistent`／`TRUE=auto_hide`のexact snapshot refと`scrollbar.chrome.editor@1`をEnvironment Profileへ束縛する。`FALSE`のE00–E13 static Profileはpersistent chromeだけを比較し、`TRUE`はpointer proximity／focus／captureから`revealed`／`indicator`／`hidden`を決める。`UISettingsController`、Settings／registry write、app-local override、old snapshotを使わない。live read／subscription failureはpersistent、captureは開始しない。font／icon assetが不変なら`style_font_generation`を変えず、E14を追加しない |
| runtime message-duration preference | Windows Adapterのexisting transient-notification transaction | same process-scoped `UISettings`でfirst top-level HWND生成前に`MessageDurationChanged`をsubscriptionし、initial readと同event／全`WM_SETTINGCHANGE`をUI session refresh triggerに`SystemParametersInfoW(SPI_GETMESSAGEDURATION)`を再読込する。exact snapshot refをEnvironment Profileへ束縛し、positive値はactionなしinformation／owner record付きcompletedのdeadlineだけ、zero／read／subscription failureはlive runtimeを`manual_only`へしcaptureを開始しない。static E00–E13は`visible_initial`またはowner recordだけを比較し、`d1 -> d2 -> d1`は別runtime probeへ置く。`SPI_SETMESSAGEDURATION`、Settings／registry write、app-local duration、Toast／background activation、UIA eventのretry floodを使わない。font／icon assetが不変なら`style_font_generation`、E14、coverage entryを増やさない |

既存`fixture.product.windows-empty-scene`と`fixture.editor.reference-01@1`は、次を同じProfile／generationでEvidence化するまでvisual asset lockを閉じない。

1. 日本語UI、ASCII identifier、code、numeric、Windows pathを同時に表示し、Host UI fontのface選択、Noto glyph coverage、baseline、ellipsis／copy、Widget／Panel／Command presentationの全Editor-owned system icon consumer集合、`EditorIconTokenContractV1`、source ID／variant／usage context mappingと16／20 lu conversion outputをlayout／semantic／visual oracleで照合する。
2. E00–E01のDarkとE13のLightでは[Editor UI Framework §15.1](../appendices/editor-ui-design-system-catalog.md#151-darklight-baselinescalehigh-contrast)の通常text contrast下限を満たし、E07–E10の四つのWindows Contrast Themeではsystem-color snapshotを用いて、selected／focused／disabled／error／warning／stale／AI proposal／Runtimeの意味を色だけに依存せず維持する。同一running processでLight→Dark→Light→四theme→少なくとも一つのuser-customized system color→Lightを遷移し、通知受信、current snapshot hash、old color-bearing packet不使用、Light／Dark title barとclientの一致、Project／Semantic不変を実証する。static image、scheme表示名、起動時readをruntime対応の代用にしない。
3. E04（DPI 200%）、E05（UI 200%）、E06（Font 200%）、E12（mixed-DPI move）で、同じasset tupleからreflow、baseline、clip、hit target、icon geometry、Semantic／UIA targetが崩れないことを示す。100%のScreenshot、別Hardware、別contrast themeを代用しない。
4. E00の`SPI_GETCLIENTAREAANIMATION=TRUE`とE11の`FALSE`をexact motion snapshotとして照合し、同一running Windowで外部controllerが`TRUE -> FALSE -> TRUE`へ変えるPlatform Adapter probeを別Evidence化する。`AnimationsEnabledChanged`の実受信と`WM_SETTINGCHANGE`の同一refreshへのcoalesceを記録し、in-flight dock preview／panel-tab／dock-workspace／indeterminate taskは次presentでstatic final stateとなり、preference change自身がProject／Semantic／Command／Receiptを増やさず、`TRUE`復帰後のnew triggerだけがfull durationを使うことを示す。E00／E11のstatic screenshot、clock、app-local settingをruntime対応の代用にしない。
5. E00–E13の全Profileが`UISettings.AdvancedEffectsEnabled=TRUE`のexact snapshotと`effects.editor.opaque-only@1`を持つことを照合し、同一running Windowで外部controllerが`TRUE -> FALSE -> TRUE`へ変えるPlatform Adapter probeを別Evidence化する。`AdvancedEffectsEnabledChanged`の実受信、old／new snapshot hash、opaque draw descriptor、Project／Semantic／Command／Receipt不変を記録し、E00–E13のstatic screenshotをruntime transitionまたはmaterial対応の代用にしない。
6. E00–E13の全Profileが`UISettings.AutoHideScrollBars=FALSE`のexact snapshotと`scrollbar.chrome.editor@1`を持ち、overflow axisは`persistent`、非overflow axisは`absent`であることを照合する。隔離test userの外部controllerが同一running Windowを`FALSE -> TRUE -> FALSE`へ変えるPlatform Adapter probeを別Evidence化し、worker callbackの実受信、fresh property read、logical extent／viewport／offset、reserved gutter、`persistent`／`revealed`／`indicator`／`hidden` chrome、container UIA Scroll、selection／focus／Project／Semantic／Command／Receipt不変を記録する。E00–E13のstatic screenshotをruntime transitionまたはauto-hide対応の代用にしない。
7. E00–E13の全Profileが`SPI_GETMESSAGEDURATION`のexact snapshotを持つことを照合し、static expected subjectを`notification.visible_initial`またはpersistent owner recordに限定する。隔離test userの外部controllerが同一running Windowをpositive `d1 -> d2 -> d1`へ変えるPlatform Adapter probeを別Evidence化し、`MessageDurationChanged`／`WM_SETTINGCHANGE`の実受信、fresh SPI read、old／new snapshot hash、deadline再計算、manual-only不変、UIA notificationのkind／processing／activity ID、redaction／coalescing、Project／Semantic／Command／Receipt不変を記録する。static screenshot、fixed seconds、wall clock、E14をruntime duration／announcement対応の代用にしない。
8. input、resolved file、source／conversion output、license／notice、environment snapshot、generationのいずれかが欠けるか不一致なら、比較を開始せず`diagnostic.toolchain.editor-visual-assets-unlocked`でfail closedにする。

本書はまだarchive／file／conversion outputの実SHA-256を捏造して埋めない。取得・license read-back・conversion・fixture実測を同一Dependency ChangeSetで完了するまでは、上記C1は未固定でありPhase 2のvisual closureを主張しない。

### 2.2 Android

| 項目 | exact pin／判断 |
|---|---|
| compile／target SDK | `compileSdk=37`、`targetSdk=36`。compile surfaceと公開target behaviorを同一値とみなさない |
| minimum SDK | API 29 |
| NDK | r29、29.0.14206865 |
| Android Gradle Plugin | Project selected pin `9.3.1`、official coordinate `com.android.tools.build:gradle:9.3.1`、Google Maven repository。dynamic `9.3.+`、未公開patch推測、Preview／RCへfallbackしない |
| Gradle | 9.5.0 |
| SDK Build Tools | 36.0.0 |
| JDK | Microsoft Build of OpenJDK 17.0.19 LTS |
| C++ build | NDK r29 Clang／LLD、`-std=c++23`、CMake 4.4.1、Single-Config Ninja 1.13.2、`externalNativeBuild.cmake` |
| AndroidX Games | GameActivity 4.4.2、Controller 2.0.2、Frame Pacing 2.1.3 |
| ABI／STL | Shipping `arm64-v8a`、Developmentは必要なCIだけ`x86_64`を追加、全native dependencyを`c++_shared`へ統一 |

根拠: official-spec — 2026-07-30確認時点で[Android Gradle plugin API reference](https://developer.android.com/reference/tools/gradle-api)はCurrent Releaseをexact `9.3.1`、Previewを`9.4.0-alpha07`として分離する。[Android Gradle plugin 9.3 release notes](https://developer.android.com/build/releases/agp-9-3-0-release-notes)は9.3 familyのCompatibilityとしてGradle 9.5.0、SDK Build Tools 36.0.0、default NDK 28.2.13676358、JDK 17を列挙する。[About Android Gradle plugin](https://developer.android.com/build/releases/about-agp)はdynamic versionを避けるよう要求する。[NDK revision history](https://developer.android.com/ndk/downloads/revision_history)と[NDK downloads](https://developer.android.com/ndk/downloads)はr29をcurrent stable、r27dをLTSとして案内する。

根拠: project-decision／provisional — Miraikanaiはclean initial baselineとしてcurrent stable patchと一致するAGP `9.3.1`と、公式defaultから意図的に更新するNDK r29を選択する。Gradle 9.5.0、Build Tools 36.0.0、JDK 17は9.3 family公式Compatibility、NDK r29は独立したMiraikanai pinとして同一baseline ChangeSetへ束縛する。Preview `9.4.0-alpha07`へfallbackせず、9.3.0またはdynamic `9.3.+`も選択pinの代用にしない。release artifactはGoogle Mavenのexact coordinateから取得し、Repository lock／Build Toolchain Closureがresolved artifact URL、size、SHA-256、repository provenanceを固定するまでimmutable artifact identityがmaterializeしたとみなさない。release artifact、既知問題、全native dependency、16 KiB page、GameActivity、Oboe、ASan、Packageのfresh Qualificationがない間はcompatible／qualified baselineとしない。

minimum SDK API 29は技術baselineであり、市場coverageの達成主張ではない。Google Playの[Target API要件](https://developer.android.com/google/play/requirements/target-sdk)は2026-08-31以降の新規app／updateへAndroid 16（API 36）以上のtargetを要求するが、minimum SDKを決めない。API 29の継続可否は[Android §5](../07-platform/android.md#5-device-testsfailurerelease-gate)の`AndroidMinSdkCoverageReceiptV1`で別途判定し、Receiptなしに「十分な端末をカバーする」と表現しない。

### 2.3 Apple

| 項目 | exact pin／判断 |
|---|---|
| Unsigned Build host | macOS Tahoe 26.2以降、arm64 |
| Xcode | 26.6 Stable |
| SDK | iOS／iPadOS 26.5 |
| Deployment target | 17.0 |
| C++ build | Xcode 26.6 AppleClang／libc++／Apple linker、`-std=c++23`、CMake 4.4.1 Xcode Generator |
| Shipping route | `AppleShippingRouteV1`のallowed tupleだけ。`{ build_driver_ref: driver.apple.xcode-cloud, delivery_profile_ref: none }`または`{ build_driver_ref: driver.apple.xcode, delivery_profile_ref: delivery-profile.apple.self-hosted }` |

### 2.4 Graphics／shader

| Target | exact external baseline |
|---|---|
| Windows | Direct3D 12、Agility SDK 1.619.4、portable HLSL 2021、DXIL Shader Model 6.6、Root Signature 1.1、Enhanced Barriers必須 |
| Android | portable HLSL 2021をDXCでSPIR-Vへoffline compileし、Vulkan 1.1、required `VP_ANDROID_vulkan_profile_2022`（AVP API version 1.1.106）、optional high `VP_ANDROID_vulkan_profile_2025`（AVP API version 1.1.128）、SPIRV-Tools validationを通す |
| Apple | portable HLSL 2021をDXCのSPIR-V intermediate、SPIRV-CrossのMSLへ変換し、iOS／iPadOS SDK 26.5のMetal compilerでmetallibをoffline生成 |

Apple経路（DXC→SPIR-V→SPIRV-Cross→MSL）は、ray系／mesh／amplification Stageの変換成立可否が未検証である。検証Ownerは[Project Shader](../06-rendering/project-shader.md)のOwnerとし、同§4のとおり検証完了までAppleの当該Stageは既定unsupportedとする。検証タスクと、代替経路（Metal Shader Converter等のDXIL→metallib経路）の採用可否検討は§9のDependency採用・更新Gateの検討事項として登録し、採用時のexact version／hashは検証時に`toolchain.lock.json`で固定する。

`ShaderCompilerProfileV1`はEngine buildが生成するTarget別closed Profileであり、次を固定する。

```text
schema_version
compiler_profile_id
target_profile_ref: exact TargetProfileRefV1(profile_kind=runtime_target)
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

Project、AI、Build scriptはargument、entry profile、register／space、optimization、layout、extension、validatorを上書きしない。Development／Profile／Shipping差はEngine-owned Profile IDで選び、Source directiveで切り替えない。DXCのpublic compile result、PDB、hash、reflection APIはentry signature、constant、resource、compiled interfaceを観測する安定Adapter境界として使うが、Engine-generated Shaderのcanonical semantic authorityにはしない。`typed_ir`の構造・意味は[Project Shader](../06-rendering/project-shader.md)の`TypedShaderIrV1`、`bounded_hlsl`は同書のdeclared／observed contractとfixture／measurementが所有する。hidden `-ast-dump` textはversion固定の破棄可能な解析Cacheに限定し、public contract、Stable ID、長期保存Artifact、単独の合否Authorityにしない。

### 2.5 External schema／AI protocol

| 項目 | exact pin／判断 |
|---|---|
| Internal validation dialect | JSON Schema Draft 2020-12 |
| Control Plane lint validator | Ajv 8.20.0。`ajv/dist/2020`、`strict=true`、`allErrors=false`、`loadSchema`未設定、local `$id`／`$ref` allowlistだけを使用 |
| MCP Tool protocol | Model Context Protocol 2026-07-28 |
| OpenAI API／SDK | Responses API、official TypeScript SDK 7.1.0（Node.js 22以上。固定Node.js 24.18.0で充足） |
| Anthropic API／SDK | 未固定。MVPのAnthropic系接続は[Executable contracts](executable-contracts.md)のMCP経路（Claude CLI／Desktop→Miraikanai MCP Server）だけとし、direct Provider projection用`provider_profile`はexact pinを確定するDependency ChangeSetまで作成しない |
| 初期評価Model | `gpt-5.6-sol`、reasoning effort `medium` |

OpenAIの公式deprecation policyは、一般GA modelを原則6か月、specialized GA variantを原則3か月、preview modelを約2週間の場合ありとし、安全・法令・compliance上の例外も認めるため、6か月を最小保証として扱わない。Model IDはOrchestrator codeへ埋め込まずToolchain lockへexplicit固定し、[公式deprecation feed](https://developers.openai.com/api/docs/deprecations)を定期read-backして、通知時にModelSnapshot Evalを期限切れへし、承認済みexplicit fallbackが同じEval／Policy Gateを通るまでdirect Provider routeをfail closedにする。Responses API固有機能への依存はProvider Adapter一箇所へ閉じる。

### 2.6 Target×Configuration C++ Build Policy

Initial V1のBuild意味は次のrecordだけが所有する。

```text
TargetConfigurationBuildPolicyV1
  build_policy_id: StableId
  build_policy_version: 1
  target_profile_ref: exact TargetProfileRefV1
  configuration:
    development | test | profile | shipping
  cxx_language_profile_ref: exact Cxx23LanguageProfileRefV1
  compiler_artifact_ref: exact ToolArtifactRefV1
  linker_artifact_ref: exact ToolArtifactRefV1
  standard_library_artifact_set_ref: exact ArtifactSetRefV1
  compiler_runtime_artifact_set_ref: exact ArtifactSetRefV1
  exception_policy: disabled_first_party
  rtti_policy: disabled_first_party
  visibility_policy: hidden_by_default_explicit_export
  optimization_policy:
    none | debug_optimized | optimized | optimized_thin_lto
  sanitizer_set:
    none | address_undefined
  assertion_policy: enabled | disabled
  debug_information_policy:
    full_private | line_tables_private | split_private
  hardening_policy_ref: exact McdContractRefV1(kind=policy)
  isa_policy_ref: exact McdContractRefV1(kind=profile)
  pgo_policy: disabled
  compiler_argument_set[1..256]:
    ordered unique canonical ASCII
  linker_argument_set[1..256]:
    ordered unique canonical ASCII
  build_policy_content_hash: SHA-256
```

`build_policy_content_hash`はASCII `MIRAKAN_TARGET_CONFIGURATION_BUILD_POLICY_V1`と自己hashを除くlength-framed canonical bytesをSHA-256する。Product current Target三種×Configuration四種の12 recordをexactly oneずつ要求し、Configuration名、CMake build type、IDE preset、package labelからFieldを補完しない。

共通First-party policyはC++ exceptionとRTTIを全Configurationで無効、Public ABIをstatus／result型とgenerated C ABIへ限定し、hidden visibility＋explicit export manifestを使う。Third-party libraryが内部でexception／RTTIを必要とする場合は別Target、別argument set、Engine-owned Adapterで隔離し、exception、`type_info`、Vendor objectをFirst-partyまたはPublic ABIへ越境させない。PGOは12 recordすべて`disabled`であり、training input、profile freshnessまたは暗黙compiler defaultをShippingへ使わない。

| Configuration | optimization | sanitizer | assertion | debug／symbol | LTO |
|---|---|---|---|---|---|
| `development` | none (`-O0`) | none | enabled | full private | disabled |
| `test` | debug optimized (`-O1`) | AddressSanitizer＋UndefinedBehaviorSanitizer | enabled | full private | disabled |
| `profile` | optimized (`-O2`) | none | disabled | line tables private、profiling marker enabled | disabled |
| `shipping` | optimized (`-O2`) | none | disabled | split private、public packageからdebug entry除外 | ThinLTO |

Target固有のcanonical argument setは次である。CMakeが順序を正規化しても、生成後argument vectorと本表のtoken projectionをset／order ruleで照合する。

| Target | common compiler arguments | Shipping additional compiler／linker arguments | hardening／runtime |
|---|---|---|---|
| Windows x86-64 | `/clang:-std=c++23 /clang:-fno-exceptions /GR- /permissive- /Zc:__cplusplus /W4 /WX /GS /MD /Brepro` | compiler `/O2 /DNDEBUG /clang:-flto=thin /guard:cf`、linker `/OPT:REF /OPT:ICF /INCREMENTAL:NO /Brepro /GUARD:CF /CETCOMPAT /DYNAMICBASE /NXCOMPAT /HIGHENTROPYVA` | clang-cl／lld-link 22.1.8、MSVC STL／UCRT／VCRuntime 14.51、compiler-side CFG instrumentation＋linker-side Guard metadata、CET-compatible image、private PDB |
| Android arm64-v8a | `-std=c++23 -fno-exceptions -fno-rtti -fvisibility=hidden -fvisibility-inlines-hidden -Wall -Wextra -Wpedantic -Werror -fstack-protector-strong -D_FORTIFY_SOURCE=2` | compiler `-O2 -DNDEBUG -flto=thin`、linker `-fuse-ld=lld -flto=thin -Wl,--gc-sections,-z,relro,-z,now,-z,noexecstack,--build-id=sha1` | NDK r29 Clang／LLD／libc++ shared、16 KiB page compatibility、private native symbols |
| Apple arm64 | `-std=c++23 -fno-exceptions -fno-rtti -fvisibility=hidden -fvisibility-inlines-hidden -Wall -Wextra -Wpedantic -Werror -fstack-protector-strong` | compiler `-O2 -DNDEBUG -flto=thin`、linker `-flto=thin -Wl,-dead_strip -Wl,-fatal_warnings` | Xcode 26.6 AppleClang／libc++／Apple linker、hardened code signingはPlatform Owner、private dSYM |

`development`は上表Shipping最適化tokenを`-O0`／`/Od`、full debugへ置換し、`test`は`-O1`と`-fsanitize=address,undefined`（Windows clang-clでは`/clang:-fsanitize=address,undefined`）、`profile`は`-O2`、line tables、profiling markerへ置換する。Sanitizer runtime availabilityとpackage exclusionを各Targetでprobeし、unsupported sanitizerをno-opまたはpassへ変換しない。Test artifact、sanitizer runtime、Profile marker、private symbolをShipping packageへ含めない。

ISA policyはWindows x86-64 baselineをSSE2とし、`-march=native`、build Host CPU、CPUID brand stringを禁止する。Optional variantは`SSE4.2`と`AVX2+FMA`だけで、Engine-owned dispatchがCPUID leafとOSXSAVE／XGETBVを検証した後に選択し、scalar／SSE2 oracleとsame-result fixtureを通す。Android `arm64-v8a`はABI-required Armv8-A＋Advanced SIMD、Appleはdeployment targetのarm64 ABIをbaselineとし、initial V1で追加ISA variantを持たない。ISA不足、unknown CPU、dispatch fixture failureではWindows baselineへ戻し、別semantic resultまたはillegal instructionを許さない。

Build Policyはexact compiler／linker binary、STL／runtime file set、Target SDK、argument vector、environment allowlist、Source tree、generated Source、public Header ManifestをBuild Receiptへ束縛する。Windows Shippingでは全first-party EXE／DLL／static library objectを生成するcompile actionがcompiler `/guard:cf`を持ち、final image linkが`/GUARD:CF`と`/DYNAMICBASE`を持つことをargument read-backでset equalityにする。一objectでもcompiler instrumentationを欠く場合、linker flagだけでCFG適合とみなさない。final EXE／DLLのload-config／header inspectionは`Guard` characteristic、`CF Instrumented`、`FID table present`をすべて要求し、一件欠落をPackage promotion failureにする。clean／incremental／cancel後rebuildで同じ input hashから同じ normalized output hash、link map、SBOM、Package inspectionを要求する。binary timestamp、PDB／dSYM identity、code signing等の意図的non-deterministic Fieldは別provenanceへ分離し、実行imageのsemantic hashへ混ぜない。

## 3. Build Driver matrix

| Driver Profile | Target／Frontend | Configure driver | C++ Generator | Package owner |
|---|---|---|---|---|
| `driver.windows.cmake-ninja-multi` | Windows／C++23 Header surface | `cmake_preset` | `Ninja Multi-Config` | Windows Platform owner |
| `driver.android.gradle-ninja` | Android／C++23 Header surface | `gradle_external_native_build` | `Ninja` | Gradle |
| `driver.apple.xcode` | Apple／C++23 Header surface | `cmake_preset` | `Xcode` | Xcode |
| `driver.apple.xcode-cloud` | Apple／C++23 Header surface | `xcode_cloud_workflow` | `Xcode` | Xcode Cloud |

本matrixは`BuildDriverProfileV1`のDriver Profile IDと許可組合せのclosed setの唯一の正本である。`AppleShippingRouteV1`は`build_driver_ref`とnullableな`delivery_profile_ref`を持ち、§2.3の二tupleだけを許可する。Xcode CloudはXcode project／schemeをbuildしてmanaged signing／TestFlight handoffを行い、self-hostedも同じXcode Driverとexact Toolchain Profileを使う。Driver IDとDelivery Profile IDを同じenumへ格納しない。First-party TargetでMakefiles系、raw Makefile、Android `ndk-build`を禁止する。Windows／Appleの通常入口はchecked-in Preset、Androidの通常入口は固定Gradle Wrapperとする。Target、Language Profile、Driver、Generator、Toolchain hashが異なるBuild tree、object、archive、log、Receiptを共有しない。Initial V1はBMIを生成または配布しない。

## 4. Tool artifact lock

| Tool | 取得元 | size／hash／integrity |
|---|---|---|
| Visual Studio Build Tools 18.8.2 bootstrapper candidate | [fixed-version bootstrapper](https://download.visualstudio.microsoft.com/download/pr/58aec969-7d60-47ab-a001-285ca0c69097/2818a86e05e8e4a3a7e27fa12c729a6484209109ab06b2352195ebb10aa33723/vs_BuildTools.exe) | project-decision／provisional: 5,686,296 bytes、SHA-256 `2818a86e05e8e4a3a7e27fa12c729a6484209109ab06b2352195ebb10aa33723`、ProductVersion 18.8.2、FileVersion 18.8.12023.21。Repository内`toolchain.lock.json`とresolved offline-layout manifestは未materialize |
| CMake 4.4.1 | [Windows x64 zip](https://cmake.org/files/v4.4/cmake-4.4.1-windows-x86_64.zip) | commit `22515316d11df7fcc74085d52b7cc3b432c592e3`、54,399,998 bytes、SHA-256 `091919e1cde162b69d2d5e0f3b1f5670c973e72133f78126fbb18042947d6f19` |
| Ninja 1.13.2 | [Windows zip](https://github.com/ninja-build/ninja/releases/download/v1.13.2/ninja-win.zip) | 291,570 bytes、SHA-256 `07fc8261b42b20e71d1720b39068c2e14ffcee6396b76fb7a795fb460b78dc65` |
| LLVM 22.1.8 | [Windows installer](https://github.com/llvm/llvm-project/releases/download/llvmorg-22.1.8/LLVM-22.1.8-win64.exe) | commit `ca7933e47d3a3451d81e72ac174dcb5aa28b59d1`、455,545,840 bytes、SHA-256 `16e5709785fef73c854646241c4a92c5cd574318d1b33c63330dd7721903e55c` |
| DXC v1.9.2602.24 | [release zip](https://github.com/microsoft/DirectXShaderCompiler/releases/download/v1.9.2602.24/dxc_2026_05_27.zip) | 27,108,038 bytes、SHA-256 `cf658aacf070d3045e31b8f1f8a696c2945f37c1095019481ef7c513368db3b4` |
| Node.js 24.18.0 LTS／npm 11.16.0 | [Windows x64 zip](https://nodejs.org/dist/v24.18.0/node-v24.18.0-win-x64.zip) | 37,176,245 bytes、SHA-256 `0ae68406b42d7725661da979b1403ec9926da205c6770827f33aac9d8f26e821` |
| TypeScript 7.0.2 | npm `typescript@7.0.2` | 365,612 bytes、integrity `sha512-8FYau96o3NKOhbjKi/qNvG/W5jhzxkbdm5sj9AbZ/5T5sWqn3hJgLfGx27sRKZWTvyzCP8dLRBTf5tBTSRVUNA==` |
| OpenAI TypeScript SDK 7.1.0 | npm `openai@7.1.0` | commit `83e6b4a3820bc9c6eac4466cf99d828e90a2ef8a`、1,795,359 bytes、npm shasum `a9f1e307b0dc34015f148f4dea1211cd08fc5939`、integrity `sha512-7xWJ9iO5z5u1dnIUGwoUmZHkSyrUYXX2cUxo2E/26iKFrSC8IdEak7z94d5UntU7z+S/Cid33hYymwMSab2fZQ==` |
| Ajv 8.20.0 | [exact npm tarball](https://registry.npmjs.org/ajv/-/ajv-8.20.0.tgz) | 217,611 bytes、SHA-256 `b2f0b3a893bbb8cc5efb6814f08b1499e19e31d5dd73683f5893382f48f6e7b3`、npm shasum `304b3636add88ba7d936760dd50ece006dea95f9`、integrity `sha512-Thbli+OlOj+iMPYFBVBfJ3OmCAnaSyNn4M1vz9T6Gka5Jt9ba/HIR56joy65tY6kx/FCF5VXNB819Y7/GUrBGA==`、MIT |
| jsonc-parser 3.3.1 | [npm package](https://www.npmjs.com/package/jsonc-parser/v/3.3.1) | commit `3c9b4203d663061d87d4d34dd0004690aef94db5`、27,354 bytes、integrity `sha512-HUgH65KyejrUFPvHFPbqOY0rsFip3Bo5wb4ngvdi1EpCYWUQDC5V+Y7mZws+DLkr4M//zQJoanu1SP+87Dv1oQ==`、MIT |
| canonicalize 3.0.0 | [npm package](https://www.npmjs.com/package/canonicalize/v/3.0.0) | commit `aba9209d044f2729c51141d8a73b11e80816e42c`、6,020 bytes、integrity `sha512-yYLfHyDMIXRyRqsKBRLX023riFLpXY2YOfdtqKXZRZy9qsfOJ9U+4F9YZL7MEzL5+ziN2x2nlBvY/Voi3EBljA==`、Apache-2.0 |
| RFC 8785 JCS fixture | [official testdata at pinned commit](https://github.com/cyberphone/json-canonicalization/tree/19d51d7fe467d4706a3ff08adf8a748f29fc21e0/testdata) | commit `19d51d7fe467d4706a3ff08adf8a748f29fc21e0`、6組12 file／1,476 bytes、fixture root SHA-256 `49ebd08bec39f4da9e2db03cffc76b2de984912fd6fbc66ec4ee33852b7b84fb` |
| Microsoft OpenJDK 17.0.19 | [Windows x64 zip](https://aka.ms/download-jdk/microsoft-jdk-17.0.19-windows-x64.zip) | 186,907,952 bytes、SHA-256 `394d1d8253d58b462300f15f9c81369478cf8813f82dca914c3b5dfdef080f9f` |
| TLA+／TLC CLI 1.7.4 | [tla2tools.jar](https://github.com/tlaplus/tlaplus/releases/download/v1.7.4/tla2tools.jar) | 2,274,532 bytes、SHA-256 `936a262061c914694dfd669a543be24573c45d5aa0ff20a8b96b23d01e050e88` |

MSVCとWindows SDKは固定bootstrapperからoffline layoutを作り、resolved `VCToolsVersion`、実行binary、STL、SDK fileのsize、version、SHA-256、signerをlockする。Android SDK／NDK／Gradle、Apple Xcode／SDKも同じ粒度でresolved file manifestを保存する。

## 5. JavaScript／AI toolchain boundary

本ProjectのJavaScript toolchain候補はNode.js 24.18.0 LTS archiveと同梱npm 11.16.0の一組である。利用rootを`orchestrator/`、`tools/architecture_lint/`、`tools/contract_compiler/`、`tools/contract_lint/`だけに限定し、各`package.json`は`private=true`、ES module、exact `engines`、`packageManager=npm@11.16.0`を要求する。PATH上のglobal Node、npm、Corepack、Bun、pnpm、Yarnを正規Buildへ混在させない。

`orchestrator/`は将来のTooling artifactをlockする候補rootであり、[AI Production Orchestration](../03-authoring/ai-production-orchestration.md)のlogical role、Workflow、Run、Context、route、Agent loop、stateまたはbinary名の正本ではない。ToolchainはAgent Host／Provider SDK／local inference runtime／Loader／Model artifactのexact version、hash、license、取得元、dependency closureを所有するが、artifactが存在する、SDK接続に成功する、Model IDを選択した、またはpackageをinstallしたことからWorkflow availability、Caller authority、surface support、Capability Activationを導出しない。

Workflow DefinitionとRoute SelectionはToolchain lockのexact artifact bindingを消費する。artifact、SDK、runtime、Model snapshotまたはMCP baselineが変われば既存Run Context／routeをstaleにできるが、本書のDependency updateだけでsilent fallback、Workflow rewrite、Run resume、Provider切替またはProduct supportを発行しない。first-party local inference runtime／loaderは引き続き未固定・MVP外・`not_activated`であり、外部Hostがlocal Modelを使えることをそのmaterialization Evidenceにしない。

通常Buildは事前充填したcontent-addressed cacheに対し`npm ci --ignore-scripts --offline --no-audit --no-fund`を実行する。install／prepare scriptを必要とするDependencyは専用ADR、exact package hash、閉じたscript allowlist、隔離Dependency Buildを先に承認する。

TypeScript 7.0.2は上記許可root（Orchestrator、architecture lint、contract compiler、contract lint）のcompileとlanguage-service CLIだけに使い、安定公開されていないprogrammatic compiler APIへ製品codeを依存させない。正式Artifactはstrict、single-threaded clean buildで生成する。Ajv 8.20.0はArchitecture Evolution Control PlaneのDraft 2020-12 schema lintだけに使い、Engine Runtime、C++ contract validator、MCD semantic validation、Authorizationへ持ち込まない。OpenAI接続はResponses APIと公式TypeScript SDK 7.1.0を使い、strict function callingのSchema制約は[Executable contracts](executable-contracts.md)が所有する。

## 6. External Dependency baseline

| 分類 | Dependency | exact release／commit | License | 採用範囲 |
|---|---|---|---|---|
| Graphics | Microsoft.Direct3D.D3D12 | 1.619.4／SDKVersion 619 | package同梱`LICENSE.txt`／`LICENSE-CODE.txt` | Agility runtime、D3D12 header、Enhanced Barriers |
| Graphics | Microsoft.Direct3D.WARP | 1.0.20、nupkg 15,319,749 bytes、SHA-256 `e5fe5de661ce98b58ef9cfb736e73c0a7a2623d3bbf5f14839b2d55566d87e40` | package同梱`LICENSE.TXT` 12,250 bytes、SHA-256 `5435f10305a92870b45735dfc169c5d6010617f6556e0859689a058d2c6b59c4` | Development／test／internal conformance専用のhost-local WARP。第三者配布Package、Shipping layout、Store upload、公開downloadへ含めない。性能Gate対象外 |
| Input | Microsoft.GameInput | 3.5.262、nupkg 2,116,174 bytes、SHA-256 `2654e45081588409f6326838e681d6b50ac533e2f24402421dd73c167744d24e` | package同梱`LICENSE.txt` SHA-256 `9e61041baca79359e84e2135450137655e19fff59fb490701970ef68833eb42e`、`NOTICE.txt` SHA-256 `3576e0a79a82e02ed70706abb27254b0beb6f0d1c3414a6a335d58f69a3fa1cb` | GameInput v3 header、static import library、PC redistributable。`GameInputRedist.msi`を通常installのprerequisiteとして適用し、DLLのside-by-side同梱を代用にしない |
| Graphics | D3D12MA | v3.2.0／`1d86c1130f61453634b1df85782e1fecfd59a525` | MIT | D3D12 heap suballocation、budget stats |
| Graphics | Vulkan Memory Allocator | v3.4.0／`3aa921224c154a0d2c43912bc88e1c42ce1f7607` | MIT | Vulkan heap suballocation、budget、defrag primitive |
| Graphics | SPIRV-Cross | Vulkan SDK 1.4.350.0／`1a6169566c73d3da552748fc372fe2bbb856e46e` | Apache-2.0 | SPIR-V reflection、MSL生成 |
| Graphics | SPIRV-Tools | v2026.2／`0539c81f69a3daeb706fd3477dca61435b475156` | Apache-2.0 | SPIR-V validation、offline optimization |
| Image | KTX-Software | v4.4.2／`4d6fc70eaf62ad0558e63e8d97eb9766118327a6` | multi-license。Repository固有fileはgenerally Apache-2.0、`lib/etcdec.cxx`は非open-sourceのEricsson terms、その他はfile-level SPDX／licenseに従う | Offline KTX container／ASTC処理候補。exact build／distribution file closure未固定 |
| Image | DirectXTex | may2026／`4feb3e11a020f35b796fc769a74216a555d4f5ef` | MIT | Offline texture decode、mipmap、BC encoding |
| Asset Import | cgltf | v1.15／`bbeb5b0b070ddacddac6852fb72143eb68454937` | MIT | Asset Import Worker内のprivate glTF 2.0 parser。target selected／artifact absent。Shipping／Runtime／public C++から除外 |
| Asset Import | MikkTSpace | upstream commit `3e895b49d05ea07e4c2133156cfa94369e19e409` | `mikktspace.h`／`.c`内のzlib-style notice | Asset Import Worker内の未変更private tangent generator。target selected／artifact absent |
| Qualification | Khronos glTF-Validator | source version 2.0.0-dev.3.11／`434283be08a668a8fb4e437145630ddbf93b0686` | Apache-2.0＋repository NOTICES | Development／Qualification専用glTF specification oracle。target selected／artifact absent。Production parser／Shippingから除外 |
| Audio | Oboe | 1.10.0／`a81bb9f87d4105b84b682685d3bfbb5beca371d1` | Apache-2.0 | Android low-latency audio stream |
| Audio | libopus | 1.6.1／source SHA-256 `6ffcb593207be92584df15b32466ed64bbec99109f007c82205f0194572411a1` | BSD-3-Clause | Streaming music／voice decode |
| Audio | libFLAC | 1.5.0／source SHA-256 `f2c1c76592a82ffff8413ba3c4a1299b6c7ab06c734dee03fd88630485c2b920` | multi-license source。採用候補の`libFLAC`／`libFLAC++`はXiph.org BSD-like、その他componentはfile-level LGPL／GPL／GFDLに従う | Import Worker内の`libFLAC` decode／metadataだけ。command-line program、plugin、documentationは採用範囲外 |
| Android | AndroidX Games GameActivity | 4.4.2 | Apache-2.0 | Activity／GameTextInput bridge |
| Android | AndroidX Games Controller | 2.0.2 | Apache-2.0 | Controller bridge |
| Android | AndroidX Games Frame Pacing | 2.1.3 | Apache-2.0 | Frame pacing bridge |
| Control Plane | Ajv | 8.20.0／npm shasum `304b3636add88ba7d936760dd50ece006dea95f9` | MIT | Architecture lintのDraft 2020-12 validationだけ。C++ Runtime validatorではない |
| Physics | Box2D 3.1.1 | v3.1.1／`8c661469c9507d3ad6fbd2fea3f1aa71669c2fe3` | MIT | private 2D collision／solver kernel |
| Physics | Jolt Physics 5.6.0 | v5.6.0／`e77f175595e64cb44218cc9d9d56fc365ad0e36a` | MIT | private CPU 3D collision／solver kernel |
| Navigation | Recast／Detour 1.6.0 | v1.6.0／`6dc1667f580357e8a2154c28b7867bea7e8ad3a7` | zlib | private 3D navmesh build／query kernel、32-bit `dtPolyRef` |
| Animation | ozz-animation | 0.16.0／`6cbdc790123aa4731d82e255df187b3a8a808256` | MIT | Skeleton compression、sampling、blend primitive |
| Text | HarfBuzz | 14.2.1／`56feae4035bdd48f62ba2b8d8c16232d4d89b3a4` | MIT | OpenType shaping |
| Text | FreeType | 2.14.1／`3bd82b5f543bc84ccf2b1d0cdb63b95218099ee6` | FreeType License | OTF／TTF validation、glyph metric／rasterization |
| Text | ICU4C | 78.3／`21d1eb0f306e1141c10931e914dfc038c06121da` | Unicode-3.0 | BCP 47、BiDi、boundary、plural／number／date／message format |

XAudio2はWindows SDK headerとOS in-box runtimeで提供され、本表への別途pinを持たない。GameInput 3.5.262のPC redistributableは[Microsoft公式NuGet](https://www.nuget.org/packages/Microsoft.GameInput/3.5.262)から取得する。Microsoftの[GameInput NuGet guidance](https://learn.microsoft.com/ja-jp/gaming/gdk/docs/features/common/input/overviews/input-nuget?view=gdk-2604)に従い、NuGet同梱`GameInputRedist.msi`は自動installされないprerequisiteとして扱い、non-GDK PC titleでは通常のGame install経路がinstall完了を所有する。MSIX contentへのDLL／MSI同梱、in-box runtimeの存在、Build Hostへの事前installをTarget machineでのprerequisite充足へ読み替えない。WARP 1.0.20も[Microsoft公式NuGet](https://www.nuget.org/packages/Microsoft.Direct3D.WARP/1.0.20)だけを取得元にするが、同梱`LICENSE.TXT`のtesting／internal-use制約によりDistribution artifactへ含めない。Windows package節はlayout、install、repair、uninstallを所有し、本書はversion、artifact hash、license、取得元を所有する。Repository内lock、Package、SBOM、NOTICE、conformance Receiptは未materializeであり、上記read-backを採用済みまたはqualifiedと表現しない。

根拠: official-spec — [KTX-Software v4.4.2 LICENSE](https://raw.githubusercontent.com/KhronosGroup/KTX-Software/v4.4.2/LICENSE.md)は、Repository固有fileがgenerally Apache-2.0である一方、取り込まれたproject由来の複数licenseを含み、`lib/etcdec.cxx`が非open-sourceでfile内のEricsson licenseに従うと明記する。同tagのlicense BOMはRepository rootで`reuse spdx`を実行してfile単位に取得する。

根拠: project-decision — KTX-Softwareは`selected／absent`候補のまま維持する。取得archive、tag commit、`reuse spdx`出力、exact compile input、linked object、配布file、license／NOTICEを同一Dependency ChangeSetへ閉じるまで、`lib/etcdec.cxx`がbuild／distribution closureから除外済みとも、Ericsson termsを承認済みとも主張しない。いずれかのbranchをlicense reviewで明示し、SBOM／noticeと一致するまではKTX／ASTC Qualification Evidenceを受理しない。

根拠: official-spec — [FLAC 1.5.0 README](https://raw.githubusercontent.com/xiph/flac/1.5.0/README.md)はsource distributionを複数licenseのcomponentで構成し、`libFLAC`／`libFLAC++`をXiph.org BSD-like、その他program／pluginをGPL、documentationをGNU FDLとして分離する。冒頭license noticeはその他libraryもfile-level LGPL／GPLを持ち得るため、source archive全体を単一BSD componentとして扱わない。[Xiph.Org FLAC license](https://www.xiph.org/flac/license.html)もreference implementation libraryとその他utility／pluginを分離する。

根拠: project-decision — Miraikanaiのinitial候補はImport Worker内の`libFLAC` decode／metadataだけである。exact 1.5.0 archive、`COPYING.Xiph`と各compile inputのlicense、CMake target／option、linked object、配布file、SBOM／NOTICEが同一Dependency ChangeSetへ閉じるまで、`flac`／`metaflac`、plugin、documentation、その他LGPL／GPL componentがbuild／distribution closureから除外済みと主張せず、FLAC import Qualification Evidenceを受理しない。将来別componentを採用する場合は、そのcomponentのlicenseと配布義務を別途承認する。

全DependencyをEngine-owned Adapterへ隔離し、Vendor型を公開Contractや永続formatへ出さない。Box2D、Jolt、Recastは該当Capabilityの候補実装へのexact hash lockを提供するだけである。Activation stateは[Product Plan](../00-product/product-plan.md)のRegistryが`{capability_id, target_id}`行単位で所有し、本文書の記載や更新でActivationを昇格しない。Joltは`JPH_USE_DX12=OFF`、`JPH_USE_VK=OFF`、`JPH_USE_MTL=OFF`、`JPH_USE_CPU_COMPUTE=OFF`としてCPU rigid-body kernelだけをbuildする。

### 6.1 Release-critical first-party component decision

次のRelease-critical能力は外部Libraryを選ばず、Engine-owned bounded componentとしてinitial V1 target designを固定する。これは実装計画、Source Directory、Schema、Artifact、FixtureまたはReceiptの存在を意味しない。materializationとQualificationがない間は各consumerをfail closedにする。

| 要求能力 | 要求元 | 採用判断／current materialization |
|---|---|---|
| C++ strict JSON parser | [Executable contracts](executable-contracts.md) §17.1 | `mirakan.json.strict.v1`。RFC 8259 UTF-8、duplicate name／invalid UTF-8／trailing bytes／nonfinite／overflow／depth・size超過をreject。first-party、実装 absent |
| C++ JSON Schema Draft 2020-12 validator | [Executable contracts](executable-contracts.md) §14 | `mirakan.schema.draft2020-12.v1`。Core／Applicator／Validation／Unevaluated／Format-Annotation vocabularyのclosed supported set、unsupported vocabulary／remote ref／unknown dialectをreject。first-party、実装 absent |
| SHA-256 | [Core architecture](core-architecture.md) §11、[Executable contracts](executable-contracts.md) §13 | `mirakan.crypto.sha256.v1`。FIPS 180-4 byte semantics、one-shot／incremental同値、NIST known-answer、全Target same bytes。first-party、実装 absent |
| C++ test harness | [Core architecture](core-architecture.md) §12 | `mirakan.test.harness.v1`＋CTest protocol。process isolation、typed fixture ID、filter、timeout、shard、structured result、crash／cancelをclosed contract化。third-party test frameworkなし、実装 absent |
| MCP server boundary | [Executable contracts](executable-contracts.md) §16.2 | `mirakan.mcp.server.v1`。MCP 2026-07-28のstdio／Streamable HTTP、request `_meta["io.modelcontextprotocol/protocolVersion"]`、`server/discover`、bounded message、cancel／disconnect、same Operation projection。外部SDKなし、実装 absent |
| managed external Host Broker／session・execution attestor | [AI Security／Approval](../01-governance/ai-security-approval.md) §8.3、Product Planの`future.capability.managed-external-host-execution` | 未固定。`planning_state=planning_only`、MVP外、Active Definition migrationで専用Work Packageを登録・承認するまで`not_activated` |
| first-party local inference runtime／loader | [AI Security／Approval](../01-governance/ai-security-approval.md) §8.4、Product Planの`future.capability.first-party-local-inference` | 未固定。MVP外、Future promotion前まで`not_activated` |

各first-party componentはstable semantic ID、version、Owner、bounded input／output、memory／time limit、diagnostic、fuzz／negative fixture、cross-target golden vectorをMCDへmaterializeし、§9のGate、Security review、same-release Toolchain lock、Qualification Receiptを満たすまでactiveにしない。AjvのControl Plane pass、Platform crypto、別JSON parser、CTest commandの存在をEngine componentのEvidenceに流用しない。

### 6.2 Materialization／future decision register

本registerはtarget designが選択済みでもArtifact／Receiptがないsubjectと、Product Futureだけに残る未選択decisionを分け、適用を許可できないconsumer、必要Evidence、fail-closed境界を索引する。Classはリスク分類であり、実装順、期限、担当または作業計画ではない。`selected／absent`を`unfixed`、`implemented`、`qualified`または`active`へ読み替えない。

| Class | Unresolved input | Decision authority | Required before | Required closure evidence | Blocked consumer／current state |
|---|---|---|---|---|---|
| A | `mirakan.test.harness.v1` selected／absent | `mirakan.arch.toolchain-dependencies` | C++ contract testをQualification Evidenceとして受理する前 | MCD、CTest discovery、failure／filter／shard／parallel／timeout／crash fixture、self-test Receipt | C++ test Evidenceの受理を停止／materialization待ち |
| A | `mirakan.crypto.sha256.v1` selected／absent | `mirakan.arch.toolchain-dependencies` | Runtime canonical hashを正本または検証結果として受理する前 | MCD、NIST known-answer、incremental／zero-length／large input、cross-target byte一致Receipt | Runtime hash consumerを停止／materialization待ち |
| A | Microsoft.Direct3D.WARP 1.0.20 selected／lock absent | `mirakan.arch.toolchain-dependencies` | WARP conformance ReceiptをD3D12 qualificationへ使用する前 | §6 exact nupkg／license read-back、Repository lock、SBOM、Agility互換、Development／test／internal-use限定、全Distribution closureからの除外、Development-only conformance Receipt | WARP qualification laneを停止／materialization待ち |
| A | Windows 11 24H2 build 26100 selected／Receipt absent | `mirakan.arch.toolchain-dependencies` | Windows package promotion条件を承認する前 | MSIX MinVersion／runtime probe同値、Agility／Enhanced Barriers／GameInput redist、Microsoft support lifecycle、OS negative fixture | Windows package promotionを停止／qualification待ち |
| A | `TargetConfigurationBuildPolicyV1` selected／records absent | `mirakan.arch.toolchain-dependencies`＋各Platform Owner＋`mirakan.arch.runtime-performance-capacity` | Target別最適化済みBuildをQualificationまたはShipping claimへ使用する前 | §2.6の12 exact policy、compiler／linker／CRT／STL、flag read-back、LTO、symbol、hardening、ISA dispatch、PGO disabled、size／startup／load／frame／memory／build metric、reproducible output、link map／SBOM／Package inspection Receipt | Configuration名だけから最適化を推測することを停止／materialization待ち |
| A | Microsoft.GameInput 3.5.262 selected／lock absent | `mirakan.arch.toolchain-dependencies` | Windows Input capabilityまたはpackage同梱判断を承認する前 | §6 exact nupkg／license／NOTICE read-back、Repository lock、header／runtime DLL manifest、MSIX redist、Input conformance Receipt | Windows Input／package Gateを停止／materialization待ち |
| A | KTX-Software v4.4.2 selected／license closure absent | `mirakan.arch.toolchain-dependencies` | KTX／ASTC import、cook、packageまたはQualification Evidenceを受理する前 | exact source archive／tag commit、LICENSE hash、`reuse spdx` file-level BOM、compile input／linked object／distribution file manifest、`lib/etcdec.cxx`のdeterministic exclusionまたはEricsson termsの承認記録、SBOM／NOTICE | KTX／ASTC consumerを停止／license・materialization closure待ち |
| A | §6のcgltf／MikkTSpace／Khronos glTF-Validator target selected／三役materialization absent | `mirakan.arch.toolchain-dependencies` | glTF Source analysis、Import Plan、Preview、Cook、promotion、Package eligibilityまたはQualification Evidenceを受理する前 | exact source archive／file hash、license／NOTICES、build option／patch state、Toolchain lock、SBOM、private Adapter、bounded Broker／Worker、parser／semantic／Khronos conformance、malformed／extension／tangent／determinism／cross-host FixtureとReceipt | 全glTF Importを`dependency_materialization_absent`で停止／Architecture選定済み・materialization待ち |
| A | libFLAC 1.5.0 selected／component closure absent | `mirakan.arch.toolchain-dependencies` | FLAC import、cook、packageまたはQualification Evidenceを受理する前 | exact source archive、`COPYING.Xiph`とcompile inputのfile-level license、CMake target／option、compile input／linked object／distribution file manifest、GPL／LGPL／GFDL componentのdeterministic exclusionまたは別途承認、SBOM／NOTICE | FLAC consumerを停止／license・materialization closure待ち |
| A | Windows Editor visual asset lock | `mirakan.arch.toolchain-dependencies` | `fixture.product.windows-empty-scene`の最初のrendered Reference Design fixture前 | §2.1のHost OS image／DirectWrite resolved file、Noto static OTF archive／file／license hash、Widget／Panel／Command presentationの全Editor-owned system icon consumer集合=`EditorIconTokenContractV1`=`conversion manifest`のexact mapping、Fluent source ID／variant／usage context、allowlist／converter／16・20 lu conversion hash、SBOM／notice、同一`style_font_generation`、Light／Dark／四High Contrast／DPI・UI・Font 200% text-icon fixture、running processの`WM_THEMECHANGED`／`WM_SYSCOLORCHANGE`／`WM_SETTINGCHANGE`後のcurrent Light／Dark modeまたはsystem-color snapshot re-read、standard title barとの一致、customized Contrast Theme transition。desktop appでunsupportedな`ColorValuesChanged`をfallbackにしないこと | Phase 2のReference Design visual closureを停止／未固定 |
| B | `mirakan.json.strict.v1` selected／absent | `mirakan.arch.toolchain-dependencies` | [Executable contracts](executable-contracts.md)のC++ acceptance pathを有効化する前 | RFC 8259、duplicate／UTF-8／trailing／overflow／depth・size positive／negative／fuzz Receipt | C++ JSON consumerを停止／materialization待ち |
| B | `mirakan.schema.draft2020-12.v1` selected／absent | `mirakan.arch.toolchain-dependencies` | [Executable contracts](executable-contracts.md)のruntime validationを有効化する前 | official Draft 2020-12 test suite、supported vocabulary set equality、unknown dialect／unsupported vocabulary／recursive ref／bound failure Receipt | runtime schema consumerを停止／materialization待ち |
| B | `mirakan.mcp.server.v1` selected singleton `2026-07-28`／implementation absent | `mirakan.arch.toolchain-dependencies` | external-agent capabilityをActivation候補へ昇格する前 | supported-version set exact `[2026-07-28]`、request `_meta["io.modelcontextprotocol/protocolVersion"]`、HTTP `MCP-Protocol-Version` header、`server/discover`、stdio／Streamable HTTP、capability negotiation、missing／unsupported／request-header mismatch、message bound、cancel／disconnect、Operation set equality Receipt | external-agent capabilityを`not_activated`／materialization・Conformance待ち。`2025-11-25` initialize、alias、fallbackまたはdual lifecycleを作らない |
| Future | managed external Host Broker／attestor boundary | `mirakan.arch.toolchain-dependencies`＋`mirakan.arch.ai-security-approval` | `future.capability.managed-external-host-execution`のActive promotion proposal前 | exact Host／version／binary、Transport／version／endpoint／auth、Provider／managed deployment／Model、Tool projection set、Targetを束縛する`HostTransportConformanceReceiptV1`／`ProviderToolConformanceReceiptV1`／`SchemaEvalConformanceReceiptV1`、Broker sandbox、session／execution attestation、Engine Build Receipt closure、全negative fixture | managed Source edit／Buildだけを`not_activated`／standard external MCP proposal laneとfirst-party local inferenceは非依存 |
| Future | first-party local inference runtime／loader | `mirakan.arch.toolchain-dependencies`＋`mirakan.arch.ai-security-approval` | `future.capability.first-party-local-inference`のActive promotion proposal前 | exact runtime release／artifact hash／license、DLL・GPU backend closure、supported model format、sandbox、OS IPC／authenticated loopback、CPU／GPU device matrix、Model Import／Schema／Tool Conformance。候補がllama.cppでもbuilt-in file／shell toolとMCP proxyは無効 | first-party local inferenceだけを`not_activated`／MVP・外部Host local model経路は非依存 |
| Future | virtualized geometry hierarchy builder／simplifier／page packer／codec／runtime provider | `mirakan.arch.toolchain-dependencies`＋`mirakan.arch.rendering-virtualized-continuous-geometry` | `future.capability.virtualized-continuous-geometry-lod`のActive promotion proposal前 | providerごとのexact version／commit／artifact hash／license／patent review／build option、deterministic hierarchy・page・root manifest、Source／Target／feature tuple conformance、corruption／overflow／device-loss fixture、SBOM／notice、in-house時は実装Directoryと同等self-test Receipt | virtualized geometryだけを`planning_only`。builder名、meshlet対応、graphics API対応からCapabilityを推測せずdiscrete LOD／HLODを維持 |
| B | Android minimum SDK market coverage | `mirakan.arch.platform-android`、閾値承認は`mirakan.arch.product-plan` | `wp.platform.mobile-offline`開始前 | [Android §5](../07-platform/android.md#5-device-testsfailurerelease-gate)の`AndroidMinSdkCoverageReceiptV1` | Android Target Gateを停止／Play Console Evidence待ち |
| Future | C++ Named Modules adoption | `mirakan.arch.cpp23-modules`＋`mirakan.arch.compatibility-evolution` | initial V1公開後のsuccessor ADR前 | 全Target stable Toolchain、Module／`import std`／IDE／analysis／sanitizer／Package fixture、complete public Header consumer inventory、single-surface Decision | required universe外。initial V1 Header Shippingに影響させない |
| C | Anthropic direct API／SDK | `mirakan.arch.toolchain-dependencies` | direct Provider projectionがProduct WPへ登録された時 | official SDK exact version／integrity／license、Provider version、Schema keyword conformance、credential／error／rate-limit fixture | direct projectionだけを停止しMVPはMCP経路を使用／scope未登録 |
| Per target | CI runner／GPU・macOS host／mobile device poolとcapacity owner | `mirakan.arch.toolchain-dependencies` | 対応するProduct Target Gate開始前 | §8.1のqualified `CiExecutionProfileV1`、non-`unfixed` Owner、runner image／Toolchain lock／device matrix／capacity Receipt | 該当laneを`diagnostic.toolchain.ci-capacity-unresolved`で停止／Owner・capacity input待ち |

各行のclosureは「名前を本文へ書く」ことではない。§9を満たすADRまたはMeasurement Receipt、exact Toolchain lock、SBOM／notice、positive／negative fixtureをread-backできた時だけmaterialization／Qualification状態を更新する。target selectionだけでFeature Gateを開かない。

## 7. Source artifact、license、取得先

| Dependency／artifact | 公式取得・Release根拠 | 追加lock |
|---|---|---|
| Agility SDK package | [NuGet package](https://www.nuget.org/packages/Microsoft.Direct3D.D3D12/1.619.4)、[flat-container artifact](https://api.nuget.org/v3-flatcontainer/microsoft.direct3d.d3d12/1.619.4/microsoft.direct3d.d3d12.1.619.4.nupkg) | 35,169,986 bytes、SHA-512 `6a275381027ed758714eedf1ccaeea446b1d9afeddc1f6b6bbc3c85939ef9ffd02b7fae780cd50da635b66b09f5fce99535788551cd64e3663b9e59fe6f7d9de` |
| LLVM／Clang／LLD 22.1.8 | [LLVM 22.1.8 release artifact](https://github.com/llvm/llvm-project/releases/tag/llvmorg-22.1.8)、[Clang C++ status](https://clang.llvm.org/cxx_status.html)、[Clang 22 command guide](https://releases.llvm.org/22.1.0/tools/clang/docs/CommandGuide/clang.html) | §4のWindows installer size／SHA-256、release signature／attestation、`clang-cl.exe`／`lld-link.exe` resolved file hash、C++23 required feature Receipt |
| Android Gradle Plugin 9.3.1 | [AGP API reference current release](https://developer.android.com/reference/tools/gradle-api)、[AGP 9.3 release notes](https://developer.android.com/build/releases/agp-9-3-0-release-notes)、[About AGP](https://developer.android.com/build/releases/about-agp)、[Google Maven](https://maven.google.com/web/index.html#com.android.tools.build:gradle) | coordinate `com.android.tools.build:gradle:9.3.1`、resolved repository URL、artifact size／SHA-256／provenance、Gradle 9.5.0、Build Tools 36.0.0、JDK 17、Miraikanai-selected NDK r29とのQualification |
| Windows 11 24H2 | [Windows 11 24H2 release health](https://learn.microsoft.com/en-us/windows/release-health/status-windows-11-24h2)、[Windows 11 release information](https://learn.microsoft.com/en-us/windows/release-health/windows11-release-information) | deployment version `10.0.26100.0`、MSIX MinVersion、runtime OS probe、current support state、build 26100未満negative fixture |
| Microsoft.GameInput 3.5.262 | [official NuGet](https://www.nuget.org/packages/Microsoft.GameInput/3.5.262)、[GameInput versioning](https://learn.microsoft.com/en-us/gaming/gdk/docs/features/common/input/overviews/input-versioning) | nupkg 2,116,174 bytes／SHA-256 `2654e45081588409f6326838e681d6b50ac533e2f24402421dd73c167744d24e`、LICENSE／NOTICE hash、header／lib／redist manifest |
| Microsoft.Direct3D.WARP 1.0.20 | [official NuGet](https://www.nuget.org/packages/Microsoft.Direct3D.WARP/1.0.20) | nupkg 15,319,749 bytes／SHA-256 `e5fe5de661ce98b58ef9cfb736e73c0a7a2623d3bbf5f14839b2d55566d87e40`、LICENSE hash、D3D10Warp.dll manifest |
| Windows Editor system UI font | [Windows Typography](https://learn.microsoft.com/en-us/windows/apps/design/signature-experiences/typography)、[International fonts](https://learn.microsoft.com/en-us/windows/apps/design/globalizing/loc-international-fonts) | bundleせず、`target.windows.editor` Host OSで解決したSegoe UI Variable／Yu Gothic UI file version・SHA-256を`resolved_files[]`へ固定 |
| Noto Sans Mono CJK JP | [Noto Sans CJK 2.004 release](https://github.com/notofonts/noto-cjk/releases/tag/Sans2.004) | `Sans2.004`／`523d033d6cb47f4a80c58a35753646f5c3608a78`、candidate archive `06_NotoSansCJKjp.zip`（94,832,242 bytes）、SIL Open Font License 1.1。archive／Regular／Bold／LICENSEのSHA-256は取得時に固定するまで未固定 |
| Fluent UI System Icons | [1.1.334 tag](https://github.com/microsoft/fluentui-system-icons/tree/1.1.334) | tag commit `f2f75a6e4814153d5c049c0f06e197731718326b`、MIT。selected SVG、conversion output、LICENSE／NOTICEのSHA-256を取得時に固定するまで未固定 |
| Windows Light／Dark・contrast／text／scale／motion | [Support Dark and Light themes in Win32 apps](https://learn.microsoft.com/en-us/windows/apps/desktop/modernize/ui/apply-windows-themes)、[Contrast themes](https://learn.microsoft.com/en-us/windows/apps/design/accessibility/high-contrast-themes)、[Accessible text requirements](https://learn.microsoft.com/en-us/windows/apps/design/accessibility/accessible-text-requirements)、[Text scaling](https://learn.microsoft.com/en-us/windows/apps/develop/input/text-scaling)、[Composition tailoring for WinUI apps](https://learn.microsoft.com/en-us/windows/apps/develop/composition/composition-tailoring)、[`SystemParametersInfoW`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-systemparametersinfow)、[`WM_SETTINGCHANGE`](https://learn.microsoft.com/en-us/windows/win32/winmsg/wm-settingchange)、[Accessibility testing](https://learn.microsoft.com/en-us/windows/apps/design/accessibility/accessibility-testing) | Light／Darkの通常visible textは各profileで最低4.5:1、四contrast themeはSystemColor snapshotで照合し、DPI／text scalingのlayout・rendering regressionをfixtureでfail closedにする。custom DirectWrite text surfaceはWindows system text scale 1.00–2.25のnotification／reflowを別途閉じる。custom client area motionは`SPI_GETCLIENTAREAANIMATION`を初期化時と`WM_SETTINGCHANGE`後にreadし、falseならstatic final presentationへ縮退する。これは新しいassetではなく、§2.1 tupleを使うEnvironment Evidenceである |
| SPIRV-Cross source | [official repository](https://github.com/KhronosGroup/SPIRV-Cross) | vcpkg source SHA-512 `f4f9f62a9ff15e9b707b820ce603bda1ea9fe7138bf505307791e55058063d9362e9bba6e508f5d302836a53b51e115b03b9ce7478fbc7b86a4b266b426eaa5d` |
| Box2D | [v3.1.1 release](https://github.com/erincatto/box2d/releases/tag/v3.1.1) | tagとcommitを照合 |
| Jolt Physics | [v5.6.0 release](https://github.com/jrouwe/JoltPhysics/releases/tag/v5.6.0) | tagとcommitを照合 |
| Recast Navigation | [v1.6.0 release](https://github.com/recastnavigation/recastnavigation/releases/tag/v1.6.0) | annotated tagとcommitを照合 |
| D3D12MA | [v3.2.0 release](https://github.com/GPUOpen-LibrariesAndSDKs/D3D12MemoryAllocator/releases/tag/v3.2.0) | source archive SHA-512とlicense hashをoverlay portで固定 |
| VMA | [v3.4.0 release](https://github.com/GPUOpen-LibrariesAndSDKs/VulkanMemoryAllocator/releases/tag/v3.4.0) | source archive SHA-512とlicense hashをoverlay portで固定 |
| SPIRV-Tools | [v2026.2 release](https://github.com/KhronosGroup/SPIRV-Tools/releases/tag/v2026.2) | source archive SHA-512とlicense hashをoverlay portで固定 |
| KTX-Software | [v4.4.2 release](https://github.com/KhronosGroup/KTX-Software/releases/tag/v4.4.2)、[v4.4.2 LICENSE](https://raw.githubusercontent.com/KhronosGroup/KTX-Software/v4.4.2/LICENSE.md) | source archive SHA-512、tag commit、LICENSE hash、`reuse spdx` file-level BOM、compile／link／distribution file manifest、`lib/etcdec.cxx`のexclusionまたは承認済みlicense branchをoverlay portで固定 |
| cgltf | [v1.15 source](https://github.com/jkuhlmann/cgltf/tree/v1.15)、[MIT license](https://raw.githubusercontent.com/jkuhlmann/cgltf/v1.15/LICENSE) | §6 exact tag／commitをread-backし、source archive／`cgltf.h`／LICENSEのsize・SHA-256、build define／warning／assert option、patch stateを固定。default file I/Oを有効化しない |
| MikkTSpace | [exact source](https://github.com/mmikk/MikkTSpace/tree/3e895b49d05ea07e4c2133156cfa94369e19e409)、[header／license notice](https://raw.githubusercontent.com/mmikk/MikkTSpace/3e895b49d05ea07e4c2133156cfa94369e19e409/mikktspace.h) | §6 exact commitをread-backし、`mikktspace.h`／`.c`のsize・SHA-256、unmodified state、license noticeを固定。altered sourceは別patch identity／notice／Qualificationなしに使用しない |
| Khronos glTF-Validator | [exact source](https://github.com/KhronosGroup/glTF-Validator/tree/434283be08a668a8fb4e437145630ddbf93b0686)、[Apache-2.0 license](https://raw.githubusercontent.com/KhronosGroup/glTF-Validator/434283be08a668a8fb4e437145630ddbf93b0686/LICENSE) | §6 exact source version／commit、source／resolved dependency closure、LICENSE／NOTICES、Qualification executableまたはpackage size／SHA-256、report schema／normalizationを固定。hosted service、floating main、third-party NuGetを代用しない |
| Oboe | [1.10.0 release](https://github.com/google/oboe/releases/tag/1.10.0) | source archive SHA-512とlicense hashをoverlay portで固定 |
| libopus | [official downloads](https://opus-codec.org/downloads/) | 上表のsource SHA-256とlicense file hashを固定 |
| libFLAC | [1.5.0 release](https://github.com/xiph/flac/releases/tag/1.5.0)、[1.5.0 README](https://raw.githubusercontent.com/xiph/flac/1.5.0/README.md)、[`COPYING.Xiph`](https://raw.githubusercontent.com/xiph/flac/1.5.0/COPYING.Xiph) | 上表のsource SHA-256、採用する`libFLAC` compile inputのfile-level license、CMake target／option、linked object／distribution manifest、GPL／LGPL／GFDL component除外、SBOM／NOTICEを固定 |
| ozz-animation | [0.16.0 release](https://github.com/guillaumeblanc/ozz-animation/releases/tag/0.16.0) | source archive SHA-512とlicense hashをoverlay portで固定 |
| HarfBuzz | [14.2.1 release](https://github.com/harfbuzz/harfbuzz/releases/tag/14.2.1) | source archive SHA-512とlicense hashをoverlay portで固定 |
| FreeType | [official downloads](https://freetype.org/download.html) | source archive SHA-512とFreeType License file hashをoverlay portで固定 |
| ICU4C | [78.3 release](https://github.com/unicode-org/icu/releases/tag/release-78.3) | filtered data hashとlicense hashを固定 |
| DirectXTex | [may2026 release](https://github.com/microsoft/DirectXTex/releases/tag/may2026) | source archive SHA-512とlicense hashをoverlay portで固定 |
| AndroidX Games | [official release notes](https://developer.android.com/jetpack/androidx/releases/games) | Google Maven artifact checksum、POM、licenseをGradle dependency verificationへ固定 |
| Ajv 8.20.0 | [Draft 2020-12 documentation](https://ajv.js.org/json-schema.html#draft-2020-12)、[npm metadata](https://registry.npmjs.org/ajv/8.20.0)、[exact tarball](https://registry.npmjs.org/ajv/-/ajv-8.20.0.tgz) | §4のexact shasum／integrity／MITをread-backし、Control Plane lintだけへ固定 |
| JSON Schema／MCP | [JSON Schema Draft 2020-12](https://json-schema.org/draft/2020-12)、[MCP current versioning](https://modelcontextprotocol.io/docs/2026-07-28/learn/versioning)、[MCP 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28) | Dialect、official current、Miraikanai-selected singleton protocol baseline、supported-version set、transport別version carrierをToolchain lockへ分離して記録 |
| OpenAI | [official SDKs](https://developers.openai.com/api/docs/libraries)、[model reference](https://developers.openai.com/api/docs/models/gpt-5.6-sol) | npm integrityとModel IDをToolchain lockへ記録 |

`third_party/ports`はexact source archive SHA-512、patch hash、compiler option、license file hashを固定する。CIはresolved source commitとこれらをSBOM、third-party notice、Build manifestへ出力する。Builtin portが別commitを指しても暗黙追従せずoverlay portを使う。

## 8. Toolchain lock contract

Repository rootの`toolchain.lock.json`はinitial canonical schema version 1とし、未知Field、`null`、重複ID、相対URL、version range、`latest`、wildcard、複数hash候補を拒否する。条件不該当Fieldは`null`でなくtagged branchから省略する。Profile、artifact、Driver、resolved fileをIDまたは正規化relative pathのunsigned UTF-8 byte順に保存し、canonical JSONのSHA-256をBuild manifestへ記録する。

| Field | 規則 |
|---|---|
| `lock_schema_version` | `uint32`、値1 |
| `profiles[].product_target_profile_ref` | [Product Plan §6.1](../00-product/product-plan.md#product-profile-identity)のexact `TargetProfileRefV1`。Active Product DefinitionのHost projectionにある`target.headless.host`、`target.windows.editor`、runtime Target projectionにある`target.windows.desktop`、`target.android.mobile`、`target.apple.mobile`を各一件とし、両projectionのtagged unionがlockの全Profile Ref集合とset equalityである。RefのID／version／kind／content hashを全Field read-backし、同ID／version別hash、同ID／hash別kind、local ID、表示名またはprefixへ縮退しない |
| `profiles[].technical_profile_content_hash` | `MIRAKAN_TOOLCHAIN_TARGET_PROFILE_BINDING_V1`、algorithm `sha256`、algorithm version 1、schema順、`uint32_be` length framingで、`product_target_profile_ref`と当該Profileの全technical Fieldをcanonical encodeしたper-profile hash。lock全体hashやProduct Profile hashで代用しない |
| `profiles[].host` | OS、architecture、minimum version、CI image digest |
| `profiles[].target.deployment_target` | Target minimum OSのcanonical numeric string。値は各Profileの正本行（`target.apple.mobile`は[§2.3 Apple](#23-apple)のDeployment target行、`target.android.mobile`は[§2.2 Android](#22-android)のminimum SDK行、`target.windows.desktop`は[§2.1 Windows](#21-windows)のTarget minimum OS行）と一致する。正本行が未固定のProfileではField自体を拒否 |
| `profiles[].artifacts[]` | tool ID、exact version、source／resolved URL、size、SHA-256、source commit。generated conversion manifestもartifact recordとして保持し、`EditorIconTokenContractV1`のicon token、全Widget／Panel／Command presentationのEditor-owned system icon consumer ref、Command presentationではexact `presentation_context`、source icon ID、Regular／Filled variant、allowed usage context、converter ID／version／commit、input SVG hash、option、logical size別output hashをそのcontent hashで束縛する |
| `profiles[].resolved_files[]` | tool ID、relative path、size、file version、SHA-256、signer |
| `profiles[].build.cxx_bindings[]` | Frontend、language standard、compiler flag／full version、STL hash、CMake ID、experimental token、BMI policy、CRT mapping |
| `profiles[].build.driver_bindings[]` | Driver Profile hash、tool IDs、toolchain file hash、driver config hash |
| `ci_execution_profiles[]` | Verification laneとrunner／hosting／toolchain／device capacityのbinding。§8.1に従う |
| `shared.npm` | Node／npm exact version、許可root、lockfile SHA-256、package version／tarball URL／size／integrity |
| `shared.vcpkg.builtin_baseline` | `cd61e1e26a038e82d6550a3ebbe0fbbfe7da78e3` |

Windows installerはSHA-256に加えてAuthenticode chainとPublisherを照合する。GitHub release artifactはrelease API digestが提供される場合はそれ、release tag commit、取得後archive hashを照合し、API digestがない旧releaseまたはtag-only sourceはverified tag commit、取得後archive hash、LICENSE／NOTICE file hashを必須にする。npm packageはlockfileのintegrityを照合する。不一致はconfigure前に失敗させる。

`target.windows.editor`ではbundled NotoとFluent source／conversion artifactを`profiles[].artifacts[]`、Host OSで解決されたSegoe UI Variable／Yu Gothic UI fileを`profiles[].resolved_files[]`へ記録する。§2.1のrole bindingはこの既存record群の閉包として解決し、新しいtop-level lock Fieldやfont fallbackの別schemaを作らない。recordを無標識のasset poolとして扱わず、UI label／code／icon role、source／converter、16／20 logical size、usage contextを同じProfileのEvidenceから再現できなければlockは未成立である。icon roleは`EditorIconTokenContractV1`のWidget／Panel／Command presentationにある全Editor-owned system icon consumer集合に対するsemantic token→source icon ID／variant／usage contextのexact bindingを必須とし、未使用のSVGをallowlistへ混ぜたり、consumer側でsource icon名を直接指定したりしない。font／icon assetの差は`style_font_generation`を変化させるため、旧generationで作られたlayout、glyph cache、draw packet、visual snapshotを再利用しない。

Fluent conversion manifestは既存`profiles[].artifacts[]`のgenerated artifactとして記録し、別schemaやmutable build logへ退避しない。manifestは各`EditorIconTokenContractV1` tokenについて全consumer ref、semantic category／subject、`command_presentation` consumerごとの`presentation_context`、source icon ID、Regular／Filled variant、allowed usage context、input SVG hash、converter ID／version／source commit／option、16／20 logical-size output hashを一recordに束縛し、同manifestのcontent hashから復元できなければならない。一つのsemantic tokenの複数source、variant／usage contextの省略、Registry consumer集合またはCommand presentation contextとの不一致を拒否する。source geometryを複数tokenで再利用する場合もtokenごとの明示recordを必須にし、暗黙aliasにしない。これによりconverter optionの同名既定値変更、input icon差し替え、outputだけの再生成を`style_font_generation`差として検出する。

`product_target_profile_ref.profile_kind`はtagged unionである。`build_host`はWindows x64 host OS／architecture／CI image／Build tool fieldsを必須とし`target`／deployment Fieldを禁止する。`editor_host`は§2.1のBuild／Editor Host OSとexact同値のWindows minimum OS、host、Editor driver／artifact fieldsを必須とする。`runtime_target`はTarget OS／architecture／deployment target、Target compiler／SDK／driver fieldsを必須とする。共通artifactをref共有してもProfile object、Product Refまたは`technical_profile_content_hash`を省略・alias化しない。

lockの`product_target_profile_ref.profile_kind=build_host | editor_host` projectionはActive Product Definitionの`host_profile_refs[]`、`profile_kind=runtime_target` projectionは`runtime_target_profile_refs[]`とRef全Fieldで各々set equalityである。各RefはDefinitionが束縛する`TargetProfileRegistryRefV1`からexactly one recordへ解決し、Product Definition側のHost／runtime Target membership、Registryまたはlock側のProfile、kind、hashにmissing／extra／duplicateがあればToolchain closureを生成しない。Locale ProfileはToolchain ProfileではなくActive Product Definitionの独立`locale_profile_refs[]`へ属し、Target Profile ID、OS localeまたはresolved fontからLocale membershipを推測しない。

対応するlock、Schema、Generator、Build manifestは未materializeであるため、上記五Product Profile Ref、kind-specific technical branch、per-profile hashを最初のcanonical schemaへ直接定義する。旧profile ID、consumer-local tuple、rename table、offline migrator、runtime alias、dual readerまたはProduct state migrationをinitial lock contractへ持ち込まない。

### 8.1 CI execution profile

[AI Verification／Provenance §14](../01-governance/ai-verification-provenance.md#14-ci-lanes)がlaneの意味、Trigger、必須Evidenceを所有し、本書はlaneを実行するinfrastructure bindingだけを`CiExecutionProfileV1`として所有する。

| Field | Rule |
|---|---|
| `lane_id` | Verification正本のexact lane ID |
| `runner_class` | `windows_gpu`、`windows_hardware_vm`、`macos_build`、`android_device`、`apple_device`のclosed enum |
| `hosting_mode` | `managed`または`self_hosted` |
| `toolchain_target_profile_ref` | `profiles[].product_target_profile_ref`のexact `TargetProfileRefV1`だけ。architecture／JavaScript laneもcurrent MVPではWindows x64 `target.headless.host` Refを使い、ID文字列へ縮退しない |
| `device_matrix_ref` | physical deviceを使う`android_device`／`apple_device` branchで必須。他runner branchではField自体を禁止する |
| `capacity_state` | `unfixed`、`qualified`、`unavailable`のclosed enum |
| `owner` | 調達、credential、patch、quota、保守、incident対応の責任主体。未決定時はliteral `unfixed` |

Entry identityは`{lane_id, runner_class, toolchain_target_profile_ref}`のtupleとし、重複を拒否する。`capacity_state=qualified`はrunner image hash、Toolchain lock hash、isolation profile、同時実行上限、retention、device matrix（該当時）、fresh Qualification Receiptが揃う場合だけ許可する。`unfixed`または`unavailable`、`owner=unfixed`、Receipt失効、device欠落ではlane開始を`diagnostic.toolchain.ci-capacity-unresolved`で拒否し、local runner、別OS、別device、managed／self-hosted間へ暗黙fallbackしない。

portable Linux CIを導入する場合はProduct Targetを偽装せず、別`ci.host.portable-linux` execution profileとしてdistro、kernel、libc、architecture、container／VM image digest、Node／JavaScript tool hashをすべて固定するDependency ChangeSetを先に承認する。現行lock／runner enumにはこのprofileをmaterializeせず、Linux Product supportはProduct Planのplanning-only Future entryのままとする。

runner契約、self-hosted host、実機pool、担当Ownerがmaterializeされていないため、本文書はcapacity、費用、工程または人員を推測しない。[Product Plan §5](../00-product/product-plan.md#5-mvp-scope)はProduct outcomeだけを所有し、必要laneが`qualified`になるまで該当Product Target Gateを成功扱いしない。

## 9. Dependency採用・更新Gate

新規採用と更新は専用ChangeSetで次をすべて満たす。

1. 公式Project／Vendor、exact release／commit、license、取得URLを確認する。
2. Download size、cryptographic hash、signerまたはpackage integrityを記録する。
3. Adapter境界、公開型非露出、allocator、exception、threading、coordinate、determinismをconformance testする。
4. clean／incremental／cancel recovery、sanitizer、replay、serialized fixture、performance、Package inspectionを再実行する。
5. SBOM、third-party notice、overlay port、CI image、Toolchain lock、Build Receiptを同時更新する。
6. 旧版と新版を恒久併存させず、全Targetに同じChangeSetでCutoverする。
7. 不合格時は別versionや別Backendへ暗黙fallbackせず、原因、API差分、破棄条件をADRへ記録する。

<a id="gltf-tangent-dependency-state"></a>

### 9.1 glTF Import dependency target baseline

[glTF Import Dependency Baseline Decision](../decisions/2026-08-03-gltf-import-dependency-baseline.md)により、initial V1の依存構成を次へ一意に選定する。Khronos公式仕様はtangent意味とValidatorの役割を裏づけるが、parserの選定はMiraikanaiのproject-decisionであり、Khronos公式推奨と表現しない。

根拠: official-spec — [Khronos glTF Registry／glTF 2.0.1](https://registry.khronos.org/glTF/)、[glTF 2.0 specification](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html)および[Khronos glTF-Validator](https://github.com/KhronosGroup/glTF-Validator)は、glTF仕様、tangent規則、公式Validatorの役割だけを裏づける。

根拠: project-decision — exact dependency identityは§6、選択理由は[glTF Import Dependency Baseline Decision](../decisions/2026-08-03-gltf-import-dependency-baseline.md)、current materialization／fail-closed状態は本節が所有する。

| Dependency role | target baseline | upstream／licenseのsource-checked事実 | current state | materialization／利用Gate |
|---|---|---|---|---|
| glTF 2.0 Production Asset parser | §6 `cgltf` exact baseline | Johannes Kuhlmann upstream、C99 single-file、external dependencyなし、MIT | Architecture target selected／artifact absent | exact source archive／file hash、license file、build option、patch state、SBOM／NOTICE、Toolchain lock、private Adapter、malformed-input／extension／determinism Qualificationが揃うまで全glTF Importを開始しない |
| tangent generator | §6 Morten S. Mikkelsen原典MikkTSpace exact baseline | standalone C source、header内のzlib-style license notice、outputはper-corner／unindexed、既存indexへのaverage／overwrite禁止、両生成entry pointはthread-safe | Architecture target selected／artifact absent | unmodified exact source、file hash／notice、Adapter input bound／overflow precondition、worker memory cap、degenerate／handedness／determinism／cross-host Qualificationが揃うまでtangent生成を成功扱いしない |
| glTF specification conformance oracle | §6 Khronos `glTF-Validator` exact baseline | KhronosGroup upstream、glTF 2.0 validation／JSON report、Apache-2.0＋repository NOTICES | Architecture target selected／Qualification Tool artifact absent | Development／Qualification専用。exact source／dependency closure／artifact hash／license／NOTICES／Toolchain lock／report normalizationを固定するまでEvidenceへ使用せず、Production parserまたはShipping dependencyにしない |

三役はinitial V1の一つのDependency ChangeSetでmaterializeし、一部だけを`adopted`、`locked`、`qualified`または`active`と表示しない。`cgltf`がparseできること、Khronos Validatorがpassすること、MiraikanaiがEngine CapabilityとしてImportできることは別判定である。parserが認識するextensionを自動的に対応subsetへ昇格せず、別decoder dependency、Morph、Owner allowlist外、Target非対応または意味変換不能なfeatureをfail closedにする。

`cgltf`はmemory parseとcustom allocator／file callbackを持つが、Miraikanaiはdefault file API、arbitrary URI、network、environmentまたはworking-directory resolutionを使用せず、Brokerが供給するbounded bytes／dependency handleだけをAsset Import Workerへ渡す。`cgltf`の型／pointerはprivate Adapter内で消費し、Engine-owned typed IR、public C++、MCD、Schema、RuntimeまたはShipping Packageへ露出しない。具体的なSource transport、validation順序、IR、Report、Receiptは[Asset Lifecycle](../03-authoring/asset-lifecycle.md#gltf-import-adapter-boundary)、tangent／normal-map意味は[Materials](../06-rendering/materials.md#41-canonical-pbrとgltf-mapping)が所有する。

MikkTSpaceは原典内部で`malloc`／`free`を使用するため、allocator差替えのためのsilent patchを行わない。Adapterがface／corner count、全積算size、index、finite inputを事前にboundし、Asset Import Workerのcommit-memory hard capで囲む。将来source改変が必要ならaltered-source表示、patch identity／hash、license noticeと全determinism Qualificationを持つ別Dependency ChangeSet／ADRを必要とする。provided tangent欠落を別algorithm、DCC名、filename、cross-productだけの近似、警告付きnormal map無効化またはruntime生成へfallbackしない。

2026-08-03時点で、上表はArchitecture上のexact target selectionだけを閉じる。source archive／file hash、license bundle、NOTICES、Toolchain lock、SBOM、Adapter、Schema、Fixture、Receipt、Build、ConformanceおよびQualificationはRepositoryに存在しない。このため全glTF Source analysis／Import Plan／typed IR／Preview／Cook／promotion／Package eligibilityを`dependency_materialization_absent`で拒否し、文書選定から採用完了または利用可能性を推論しない。

## 10. Context7と公式一次資料

Context7で次のIDを2026-07-21～2026-07-30に解決し、指定queryで確認した。

| 対象 | Context7 ID | query／確認結果 |
|---|---|---|
| CMake | `/websites/cmake_cmake_help` | C++ Module scanのGenerator、`import std`、`CXX_MODULE_STD`を照会。`import std`はNinja／Ninja Multi-Configに限定され、Visual Studio GeneratorはIMPORTED targetのBMIをbuildできず、CMake 4.4時点でもExperimental token（`CMAKE_EXPERIMENTAL_CXX_IMPORT_STD`）を必要とすることを確認 |
| Clang／LLVM | `/websites/clang_llvm` | C++ language modeとNamed Modulesのstandard mode一致を照会。mode flag受理を完全なC++23 feature conformanceへ一般化せず、exact 22.1.8 releaseとrequired feature closureはLLVM公式release／Clang statusでread-backする |
| Android NDK | `/android/ndk` | Gradle `ndkVersion`によるexact revision選択を照会。stable／LTS status、r29 revision、download artifactはAndroid公式NDK downloads／revision historyで別途read-backする |
| cgltf | `/jkuhlmann/cgltf` | C99 single-file、external dependencyなし、memory allocator／file callback、memory parse、`cgltf_validate`を照会。exact `v1.15` tag／commit、MIT license、source内容はupstreamのversion固定資料で別途read-backする |
| fastgltf | `/spnda/fastgltf` | C++17、`Expected<T>`、SIMD、custom buffer allocation、extension選択を代替案比較として照会。initial V1ではC++／ISA／embedded dependency closureを増やさないため非選定とし、非選定Libraryをlock対象へ追加しない |
| Ajv | `/ajv-validator/ajv` | Draft 2020-12専用class `ajv/dist/2020`、strict mode、`allErrors`を照会。Control Plane lintを`strict=true`／`allErrors=false`／local `$ref`だけへ固定する判断と整合 |
| Microsoft C++ | `/microsoftdocs/cpp-docs` | named moduleと`import std`を照会。標準header include／header unitとの混在禁止をBuild graph制約として扱う |
| Windows apps | `/websites/learn_microsoft_en-us_windows_apps` | Typography、International fonts、contrast themes、accessible text、text scaling、Light／Dark Win32 theme、desktop WinRT API supportを照会。Segoe UI Variable／Yu Gothic UIのHost解決、通常text 4.5:1、四Contrast Themeとuser-customized SystemColor、custom DirectWriteの`TextScaleFactorChanged`／1.00–2.25 reflow、`UISettings.GetColorValue(UIColorType::Foreground)`によるLight／Dark判定、`DwmSetWindowAttribute(DWMWA_USE_IMMERSIVE_DARK_MODE)`によるstandard title bar同期、desktop appでunsupportedな`ColorValuesChanged`を除外し`WM_THEMECHANGED`／`WM_SYSCOLORCHANGE`／`WM_SETTINGCHANGE`後にcurrent snapshotを再読込すること、DPI／text scaling regression testを§2.1 C1 closureへ反映 |
| Windows apps effects／motion／scrollbars | `/websites/learn_microsoft_en-us_windows_apps` | `UISettings.AdvancedEffectsEnabled`／`AdvancedEffectsEnabledChanged`によるcustom transparency effectのopaque fallback、`UISettings.AnimationsEnabled`／`AnimationsEnabledChanged`を尊重する公式guidance、`AutoHideScrollBars`が特定scrollbarのvisible stateではなくcustom UI／scrollbarが尊重する利用者設定であること、`AutoHideScrollBarsChanged`がworker threadから通知しevent argsにstateを持たないこと、Scroll／ScrollBar UIAの責務分離を照会。desktop appの`UISettings` unsupported eventは`ColorValuesChanged`だけであることを確認し、custom Win32 client areaは`SPI_GETCLIENTAREAANIMATION`を唯一のmotion value source、WinRT eventと`WM_SETTINGCHANGE`を同じrefresh triggerにする§13.2.3–§13.2.5の判断と整合 |
| Windows message duration／UIA notification | `/websites/learn_microsoft_en-us_windows_apps` | `MessageDuration`が秒単位のread-only app-view duration、`MessageDurationChanged`が値変更通知であることを照会し、Win32の`SPI_GETMESSAGEDURATION`、`WM_SETTINGCHANGE`、`UiaRaiseNotificationEvent`、NotificationKind／Processing、Windows App SDK app notificationの外部popup／elevation制約は公式一次資料でread-backする。custom in-window surfaceはSPIを唯一のduration値、UIA eventをredaction済みcoalesced announcementだけにし、C1で外部Toastを使わない§13.2.6／§15.6.34の判断と整合 |
| Box2D | `/erincatto/box2d` | 3.xのopaque ID、`b2WorldDef` task callback／worker、sleep、body creation、filterを照会。exact 3.1.1 pinに対するprivate Adapter候補は公式3.1.1 ReleaseとSimulation本文で再確認し、worker／sleep値を全Target defaultにしない |
| Jolt Physics | `/jrouwe/joltphysics` | 5.xのJobSystem／TempAllocator、`AddBodiesPrepare`／`AddBodiesFinalize`、BroadPhase layer／filter、sleep、`OptimizeBroadPhase`を照会。exact 5.6.0 pinへ限定し、GPU compute backendを無効化したCPU kernel候補と整合 |
| Recast Navigation | `/recastnavigation/recastnavigation` | Recast buildとDetour runtime queryの分離、`dtNavMeshQuery`のnode pool／sliced lifecycle、`dtTileCache` updateを照会。exact 1.6.0／32-bit `dtPolyRef`のprivate Backend境界と整合 |
| DirectX Shader Compiler | `/microsoft/directxshadercompiler` | `IDxcResult`のobject／PDB／shader hash／reflection出力と`IDxcUtils::CreateReflection`を照会。Fact Graphの安定証拠をpublic APIへ限定する判断と整合 |
| Unreal Engine 5.8 | `/websites/dev_epicgames_unreal-engine_5_8` | 収録範囲が概説中心だったため、RDG parameter／resource validationとGlobal ShaderはEpic公式5.8本文で補完 |
| Unity Graphics | `/unity-technologies/graphics` | HLSL function reflectionによるShader Graph Node生成とURP RenderGraph resource宣言を照会。Project Module／Technique分離の比較根拠に使用 |

直接参照する公式資料は[CMake C++ modules](https://cmake.org/cmake/help/latest/manual/cmake-cxxmodules.7.html)、[MSVC `import std` tutorial](https://learn.microsoft.com/en-us/cpp/cpp/tutorial-import-stl-named-module)、[Android 16 KiB page size](https://developer.android.com/guide/practices/page-sizes)、[OpenAI deprecations](https://developers.openai.com/api/docs/deprecations)、[OpenAI function calling](https://developers.openai.com/api/docs/guides/function-calling)、[Responses migration](https://developers.openai.com/api/docs/guides/migrate-to-responses)とする。Context7結果は検索補助であり、規範判断はリンク先の公式本文で再確認する。

Context7の内容はmain branchの挙動説明であり、exact release／commitの証拠は上記公式Release pageとtagで補完した。

### 10.1 公式Evidence read-back

| 対象 | 公式URL | 検証日 | Miraikanaiの適用判断 |
|---|---|---|---|
| glTF 2.0.1／cgltf／MikkTSpace／glTF-Validator | [Khronos glTF Registry](https://registry.khronos.org/glTF/)、[glTF 2.0 specification](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html)、[`cgltf` v1.15](https://github.com/jkuhlmann/cgltf/tree/v1.15)、[`cgltf` MIT license](https://raw.githubusercontent.com/jkuhlmann/cgltf/v1.15/LICENSE)、[MikkTSpace exact source](https://github.com/mmikk/MikkTSpace/tree/3e895b49d05ea07e4c2133156cfa94369e19e409)、[MikkTSpace license notice／API](https://raw.githubusercontent.com/mmikk/MikkTSpace/3e895b49d05ea07e4c2133156cfa94369e19e409/mikktspace.h)、[Khronos glTF-Validator exact source](https://github.com/KhronosGroup/glTF-Validator/tree/434283be08a668a8fb4e437145630ddbf93b0686)、[Validator license](https://raw.githubusercontent.com/KhronosGroup/glTF-Validator/434283be08a668a8fb4e437145630ddbf93b0686/LICENSE) | 2026-08-03 | Khronosのofficial factはglTF 2.0.1、missing tangent時のdefault MikkTSpace、normal欠落時のflat normal／provided tangent無効化、Validatorの仕様検証用途に限定する。`cgltf`／MikkTSpace／Validator exact commitの三役選定、Worker隔離、single-parser、fail-closed、materialization GateはMiraikanaiのproject-decisionである |
| Windows typography／international font／icon semantics | [Typography](https://learn.microsoft.com/en-us/windows/apps/design/signature-experiences/typography)、[International fonts](https://learn.microsoft.com/en-us/windows/apps/design/globalizing/loc-international-fonts)、[Icons in Windows apps](https://learn.microsoft.com/en-us/windows/apps/develop/ui/controls/icons) | 2026-07-26 | UI fontはHost OS image上のDirectWrite resolved fileとして固定し、Segoe UI VariableとYu Gothic UIのfamily名だけ、またはsystem fontのbundleを再現性の根拠にしない。code fontは別bundled static OTFとしてlockする。iconは小サイズで意味が明確なsemantic tokenだけを使い、Widget Patternのtoken→approved source ID／variant／usage contextをconversion manifestへ固定する |
| Windows Light／Dark・contrast／theme change／accessible text／scaling | [Support Dark and Light themes in Win32 apps](https://learn.microsoft.com/en-us/windows/apps/desktop/modernize/ui/apply-windows-themes)、[Contrast themes](https://learn.microsoft.com/en-us/windows/apps/design/accessibility/high-contrast-themes)、[High contrast parameter](https://learn.microsoft.com/en-us/windows/win32/winauto/high-contrast-parameter)、[`GetSysColor`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getsyscolor)、[`WM_THEMECHANGED`](https://learn.microsoft.com/en-us/windows/win32/winmsg/wm-themechanged)、[`WM_SYSCOLORCHANGE`](https://learn.microsoft.com/en-us/windows/win32/gdi/wm-syscolorchange)、[`WM_SETTINGCHANGE`](https://learn.microsoft.com/en-us/windows/win32/winmsg/wm-settingchange)、[Desktop WinRT API support](https://learn.microsoft.com/en-us/windows/apps/desktop/modernize/winrt-api-desktop-app-support)、[`UISettings.GetColorValue`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.viewmanagement.uisettings.getcolorvalue)、[`DwmSetWindowAttribute`](https://learn.microsoft.com/en-us/windows/win32/api/dwmapi/nf-dwmapi-dwmsetwindowattribute)、[Accessible text requirements](https://learn.microsoft.com/en-us/windows/apps/design/accessibility/accessible-text-requirements)、[Text scaling](https://learn.microsoft.com/en-us/windows/apps/develop/input/text-scaling)、[`UISettings.TextScaleFactor`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.viewmanagement.uisettings.textscalefactor)、[`UISettings.TextScaleFactorChanged`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.viewmanagement.uisettings.textscalefactorchanged)、[Accessibility testing](https://learn.microsoft.com/en-us/windows/apps/design/accessibility/accessibility-testing) | 2026-07-25 | 通常visible text 4.5:1をLight／Dark token profileの双方で検証し、non-contrast modeはcurrent foregroundから判定してstandard title barも同じmodeへ同期する。Contrast Themeはhard-coded colorでなくcurrent system-color snapshotへ解決する。custom DirectWrite text surfaceは1.00–2.25のsystem text factorを通知ごとに再読込してreflowし、font-based iconのtext scale継承を行わない。running desktop processはWin32 theme／system-color通知ごとにHigh Contrast flagとcurrent Light／Dark modeまたはsystem colorをstable snapshotとして再読込し、Light↔Dark、四built-in theme、user-customized colorの遷移を同じprocessで検証する。`UISettings.ColorValuesChanged`はdesktop appでunsupportedなため使用しない。DPI／text scalingのlayout・rendering regressionはReference fixtureのrequired Evidenceとし、High Contrastだけを通常themeの可読性不足の代用にしない |
| Windows client-area animation preference | [Composition tailoring for WinUI apps](https://learn.microsoft.com/en-us/windows/apps/develop/composition/composition-tailoring)、[`UISettings.AnimationsEnabledChanged`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.viewmanagement.uisettings.animationsenabledchanged)、[`SystemParametersInfoW`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-systemparametersinfow)、[`WM_SETTINGCHANGE`](https://learn.microsoft.com/en-us/windows/winmsg/wm-settingchange)、[Desktop WinRT API support](https://learn.microsoft.com/en-us/windows/apps/desktop/modernize/winrt-api-desktop-app-support) | 2026-07-25 | Microsoftのuser animation preference尊重guidanceをcustom Win32 client areaへ適用し、`SPI_GETCLIENTAREAANIMATION`を唯一のmotion value sourceとする。first top-level HWND前に`AnimationsEnabledChanged`をsubscriptionし、同eventと`WM_SETTINGCHANGE`ごとにSPI値を再読込する。`FALSE`ならvisual-only effectをnext present前にstatic final stateへ解決する。Editorは`SPI_SETCLIENTAREAANIMATION`、registry／Settings writeをせず、E11 static baselineだけでrunning-process transitionをpassにしない |
| Windows transparency effects | [Composition tailoring for WinUI apps](https://learn.microsoft.com/en-us/windows/apps/develop/composition/composition-tailoring)、[`UISettings.AdvancedEffectsEnabled`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.viewmanagement.uisettings.advancedeffectsenabled)、[`UISettings.AdvancedEffectsEnabledChanged`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.viewmanagement.uisettings.advancedeffectsenabledchanged)、[Acrylic material](https://learn.microsoft.com/en-us/windows/apps/design/style/acrylic)、[Use Mica material in Win32 apps](https://learn.microsoft.com/en-us/windows/apps/desktop/modernize/ui/apply-mica-win32)、[Desktop WinRT API support](https://learn.microsoft.com/en-us/windows/apps/desktop/modernize/winrt-api-desktop-app-support) | 2026-07-25 | Microsoftのcustom transparency effectに対する`AdvancedEffectsEnabled`／Changed尊重とopaque fallback guidanceを採用する。C1は`effects.editor.opaque-only@1`を固定し、`true`でもMica／Acrylic／backdrop／blurをactivateしない。false、High Contrast、support failureは同じopaque representationにする。将来materialはWindows App SDK等のDependency、named policy、opaque／High Contrast fallback、別profile Evidenceを閉じたChangeSetだけで検討する |
| Windows automatic scrollbars | [`UISettings.AutoHideScrollBars`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.viewmanagement.uisettings.autohidescrollbars)、[`UISettings.AutoHideScrollBarsChanged`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.viewmanagement.uisettings.autohidescrollbarschanged)、[Scroll viewer controls](https://learn.microsoft.com/en-us/windows/apps/develop/ui/controls/scroll-controls)、[Scroll Control Pattern](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-implementingscroll)、[ScrollBar Control Type](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-supportscrollbarcontroltype)、[`UISettingsController`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.viewmanagement.core.uisettingscontroller)、[Desktop WinRT API support](https://learn.microsoft.com/en-us/windows/apps/desktop/modernize/winrt-api-desktop-app-support) | 2026-07-25 | `AutoHideScrollBars`は特定Scrollbarのvisible stateではないため、custom chromeは利用者設定のread-only inputとして扱う。Changed callbackはworker threadからUI session refreshへqueueし、propertyを再読込する。C1のstatic E00–E13は`FALSE` persistent profile、runtimeは別の`FALSE -> TRUE -> FALSE` probeで検証する。chromeを隠してもcontainer UIA Scrollとlogical scroll stateを保ち、non-scroll axisは`ViewSize=100`／`ScrollPercent=-1`を返す。Editorは`UISettingsController`、registry／Settings writeで利用者設定を変更しない |
| Windows message duration／in-window notification | [Message duration parameter](https://learn.microsoft.com/en-us/windows/win32/winauto/message-duration)、[`SystemParametersInfoW`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-systemparametersinfow)、[Accessibility parameters](https://learn.microsoft.com/en-us/windows/win32/winauto/accessibility-parameters)、[`WM_SETTINGCHANGE`](https://learn.microsoft.com/en-us/windows/winmsg/wm-settingchange)、[`UISettings.MessageDuration`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.viewmanagement.uisettings.messageduration)、[`UISettings.MessageDurationChanged`](https://learn.microsoft.com/en-us/uwp/api/windows.ui.viewmanagement.uisettings.messagedurationchanged)、[`UiaRaiseNotificationEvent`](https://learn.microsoft.com/en-us/windows/win32/api/uiautomationcoreapi/nf-uiautomationcoreapi-uiaraisenotificationevent)、[NotificationKind](https://learn.microsoft.com/en-us/windows/win32/api/uiautomationcore/ne-uiautomationcore-notificationkind)、[NotificationProcessing](https://learn.microsoft.com/en-us/windows/win32/api/uiautomationcore/ne-uiautomationcore-notificationprocessing)、[InfoBar](https://learn.microsoft.com/en-us/windows/apps/develop/ui/controls/infobar)、[App notifications overview](https://learn.microsoft.com/en-us/windows/apps/develop/notifications/app-notifications/) | 2026-07-26 | popup durationはhard-codeせず`SPI_GETMESSAGEDURATION`を唯一のvalue sourceにする。`MessageDurationChanged`と`WM_SETTINGCHANGE`はrefresh triggerだけとし、SPIを再読込する。C1はin-window notificationだけを使い、actionなしshort feedbackをStatus bar一件、状態／actionをowner-inlineに分ける。auto-dismissはactionなしinformationまたはowner recordを持つcompleted、他はmanual-onlyにする。UIA notificationはredaction済みのbounded summaryをactivity IDでcoalesceし、`All`／`ImportantAll`を使わない。Editorは`SPI_SETMESSAGEDURATION`、registry／Settings write、Windows App SDK app notification／Toastを発行しない |
| MCP official current／Miraikanai-selected baseline | [MCP versioning](https://modelcontextprotocol.io/docs/2026-07-28/learn/versioning)、[current 2026-07-28](https://modelcontextprotocol.io/specification/2026-07-28) | 2026-08-03 | official currentは`2026-07-28`。Miraikanaiはpre-public clean initial V1のProject判断としてsupported-version setをexact singleton `[2026-07-28]`にする。request `_meta["io.modelcontextprotocol/protocolVersion"]`、`UnsupportedProtocolVersionError`、Streamable HTTP header、`server/discover`を同Revisionのlifecycleとして扱い、`2025-11-25` initialize、legacy alias、version fallbackまたはdual lifecycleを持たない。SDKが旧Revisionを支援していてもMiraikanaiの相互運用、実装またはConformanceを意味しない |
| MCP Authorization | [MCP 2026-07-28 Authorization](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization) | 2026-08-03 | Authorizationはoptionalとし、HTTP-based transportで有効化する場合は同仕様のOAuth discovery／token audienceを適用する。STDIOはMCP OAuth対象外だが、MiraikanaiではEngine／Project credentialをenvironmentから継承しない。AI Security §8の`none`またはTask専用短命Broker例外だけを許可する |
| OpenAI hosted ChatGPT Work MCP | [Model Context Protocol](https://learn.chatgpt.com/docs/extend/mcp#use-mcp-backed-tools-in-chatgpt-web) | 2026-07-24 | hosted ChatGPT Workはpluginが束ねるremote MCP Toolを使い、local Codex設定／command menu／direct local STDIOを読まない。plan／workspace／admin条件とremote serviceまたはSecure MCP TunnelをHost Profileへ固定する |
| OpenAI Codex-hosted MCP | [Model Context Protocol](https://learn.chatgpt.com/docs/extend/mcp) | 2026-08-03 | ChatGPT desktop appのCodex host、Codex CLI、IDEは同じCodex MCP config layerからSTDIO／Streamable HTTPを使える。hosted ChatGPT Workとは別Profileにし、MiraikanaiのMCP 2026-07-28 conformance Gateを別途必須にする |
| OpenAI GPT-5.6 | [GPT-5.6 migration guidance](https://developers.openai.com/api/docs/guides/upgrading-to-gpt-5p6-sol) | 2026-07-27 | direct Providerの既定explicit modelを`gpt-5.6-sol`、reasoning effortを`medium`とし、ModelSnapshot Profile／Evalなしにaliasへ追従しない |
| Ajv Draft 2020-12 | [Ajv JSON Schema versions](https://ajv.js.org/json-schema.html#draft-2020-12)、[Ajv 8.20.0 registry metadata](https://registry.npmjs.org/ajv/8.20.0) | 2026-07-23 | `ajv/dist/2020`をControl Plane lintだけへexact lockし、§4のtarball／integrity／MITをread-backする。C++ Runtimeは選定済みfirst-party `mirakan.schema.draft2020-12.v1`であり、Ajv ReceiptをmaterializationまたはQualificationへ流用しない |
| CMake C++ Modules | [latest manual](https://cmake.org/cmake/help/latest/manual/cmake-cxxmodules.7.html)、[4.4 manual](https://cmake.org/cmake/help/v4.4/manual/cmake-cxxmodules.7.html) | 2026-07-30 | initial V1ではNamed Modules、`import std`、experimental token、CXX_MODULES file setをすべて禁止する。資料はrequired Shipping inputではなく将来proposalの比較根拠だけ |
| CMake release | [CMake 4.4.1 release](https://github.com/Kitware/CMake/releases/tag/v4.4.1)、[CMake 4.4.2 release](https://github.com/Kitware/CMake/releases/tag/v4.4.2)、[4.4 release notes](https://cmake.org/cmake/help/v4.4/release/4.4.html) | 2026-08-03 | Miraikanai-selected baselineは4.4.1のまま固定する。公式4.4.2は2026-07-31公開でdocumented feature／interface変更なし、ecosystem対応／regression修正のpatchである。4.4.2をcurrent pinと暗黙解釈せず、採用にはartifact identity、Toolchain lock、全Target clean build／test／packageの別Dependency ChangeSetが必要である。RC／git-stage buildへfallbackしない |
| LLVM／Clang release | [LLVM 22.1.8 release](https://github.com/llvm/llvm-project/releases/tag/llvmorg-22.1.8)、[Clang C++ status](https://clang.llvm.org/cxx_status.html) | 2026-07-30 | signed release tag／binaryを22.1.8へ固定し、C++23全体の`Partial`表示をrequired feature passへ読み替えない |
| MSVC C++ language mode | [Microsoft `/std` reference](https://learn.microsoft.com/en-us/cpp/build/reference/std-specify-language-standard-version)、[MSVC 14.51 C++23 status](https://devblogs.microsoft.com/cppblog/c23-support-in-msvc-build-tools-14-51/) | 2026-07-30 | `/std:c++23preview`はMSVC comparison laneだけに許可し、Preview artifactをShippingへ使用しない。Initial V1 Shippingはclang-cl 22.1.8 `/clang:-std=c++23` |
| Windows Control Flow Guard | [Microsoft compiler `/guard:cf`](https://learn.microsoft.com/en-us/cpp/build/reference/guard-enable-control-flow-guard)、[Microsoft linker `/GUARD:CF`](https://learn.microsoft.com/en-us/cpp/build/reference/guard-enable-guard-checks)、[Clang command-line reference](https://clang.llvm.org/docs/ClangCommandLineReference.html) | 2026-07-30 | compiler instrumentationとlinker image metadataを分離し、全first-party objectへ`/guard:cf`、final linkへ`/GUARD:CF /DYNAMICBASE`、binary inspectionへ`Guard`／`CF Instrumented`／`FID table present`を要求する |
| Windows Host／minimum Target | [Windows 11 release information](https://learn.microsoft.com/en-us/windows/release-health/windows11-release-information)、[Windows 11 24H2 release health](https://learn.microsoft.com/en-us/windows/release-health/status-windows-11-24h2) | 2026-07-30 | Host lockは25H2 build 26200.8875、player minimumは24H2 deployment version `10.0.26100.0`として分離する |
| Android NDK stable | [NDK downloads](https://developer.android.com/ndk/downloads)、[NDK revision history](https://developer.android.com/ndk/downloads/revision_history) | 2026-07-30 | current stable r29 revision 29.0.14206865を選択し、Context7上のr30例またはpreview channelをstableへ読み替えない |
| Apple Xcode release | [Xcode 26.6 release](https://developer.apple.com/news/releases/?id=06252026a)、[Xcode 26.6 release notes](https://developer.apple.com/documentation/Xcode-Release-Notes/xcode-26_6-release-notes) | 2026-07-30 | Stable 26.6（17F113）、Apple SDK 26.5、required macOS Tahoe 26.2以降をselected Apple host baselineにする |
| C++ memory safety instrumentation | [Microsoft AddressSanitizer](https://learn.microsoft.com/en-us/cpp/sanitizers/asan?view=msvc-170)、[LLVM AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html)、[LLVM ThreadSanitizer](https://clang.llvm.org/docs/ThreadSanitizer.html) | 2026-07-26 | ASanはTarget Profileが対応するcompiler／runtime上のDevelopment／CI laneへ固定し、MSVCでは対応x86／x64構成だけを有効候補にする。sanitizer buildをShipping、PGO、通常performance baselineへ流用しない。race検証は別のsupported Toolchain／runner laneだけで実行し、MSVCまたは任意Targetが未対応でも実行済みと表示しない。未対応Targetはstatic／negative fixtureとsupported laneのEvidenceを別記録にし、同値のpassへ変換しない |
| Box2D 3.1.1 simulation optimization boundary | [Box2D Simulation](https://box2d.org/documentation/md_simulation.html)、[v3.1.1 release](https://github.com/erincatto/box2d/releases/tag/v3.1.1) | 2026-07-26 | task callback／worker、sleep、body creation position、filterはexact Target／World Profile候補として測定し、値をAPI既定またはCPU brandから推測しない。task callbackはprivate、thread-safe、allocation-freeで、Engine-owned semantic oracleを変更しない |
| Jolt Physics 5.6.0 optimization boundary | [Jolt Physics 5.6.0 documentation](https://jrouwe.github.io/JoltPhysicsDocs/5.6.0/)、[v5.6.0 release](https://github.com/jrouwe/JoltPhysics/releases/tag/v5.6.0) | 2026-07-26 | JobSystem／TempAllocator、batch insertion、BroadPhase layer／filter、sleepをTarget別候補にする。複数Bodyを一件ずつ追加する経路と毎frameの`OptimizeBroadPhase`を標準候補にせず、ContactListener rejectをearly filterと数えない |
| Recast／Detour 1.6.0 query optimization boundary | [`dtNavMeshQuery`](https://recastnav.com/classdtNavMeshQuery.html)、[`dtTileCache`](https://recastnav.com/classdtTileCache.html)、[v1.6.0 release](https://github.com/recastnavigation/recastnavigation/releases/tag/v1.6.0) | 2026-07-26 | `maxNodes` 1～65,535、sliced queryのinit／update／finalize占有、partial finalize、tile-cache staging／`upToDate`をEngine status／version契約へ正規化し、partial resultまたはstaging tileを通常successへしない |
| D3D12 alias／render-pass／indirect execution | [Memory aliasing and data inheritance](https://learn.microsoft.com/en-us/windows/win32/direct3d12/memory-aliasing-and-data-inheritance)、[Render passes](https://learn.microsoft.com/en-us/windows/win32/direct3d12/direct3d-12-render-passes)、[ExecuteIndirect sample](https://learn.microsoft.com/en-us/samples/microsoft/directx-graphics-samples/d3d12-execute-indirect-sample-win32/) | 2026-07-26 | transient aliasは非重複lifetime、aliasing barrier、next resource full initializationを必須にする。render passはTarget依存、GPU indirectはCPU visible-set oracleとcapacity failureを持つcandidateであり、全GPU共通defaultまたはsilent draw fallbackにしない |

Shader toolchainの補完一次根拠は[DXC v1.9.2602.24 release](https://github.com/microsoft/DirectXShaderCompiler/releases/tag/v1.9.2602.24)、[DXC API](https://github.com/microsoft/DirectXShaderCompiler/wiki/Using-dxc.exe-and-dxcompiler.dll)、[DXC HLSL options](https://github.com/microsoft/DirectXShaderCompiler/blob/main/include/dxc/Support/HLSLOptions.td)、[HLSL Specification Working Draft](https://microsoft.github.io/hlsl-specs/specs/index.html)である。Working Draftまたはmain branchの変化をBuild時に自動採用せず、上表のDXC tag／commitと`ShaderCompilerProfileV1`を実行正本にする。

CMakeのversion別根拠は[Presets](https://cmake.org/cmake/help/v4.4/manual/cmake-presets.7.html)、[Ninja Multi-Config](https://cmake.org/cmake/help/v4.4/generator/Ninja%20Multi-Config.html)、[File API](https://cmake.org/cmake/help/v4.4/manual/cmake-file-api.7.html)で補完した。Clang Shipping frontendは[Clang C++ status](https://clang.llvm.org/cxx_status.html)、[Clang 22 command guide](https://releases.llvm.org/22.1.0/tools/clang/docs/CommandGuide/clang.html)、[LLVM 22.1 release notes](https://releases.llvm.org/22.1.0/docs/ReleaseNotes.html)を一次根拠とする。Clang公式statusのC++23はPartialであるためrequired feature closureを使う。MSVCは[stable release](https://devblogs.microsoft.com/cppblog/msvc-version-1451-available/)、[C++23 support status](https://devblogs.microsoft.com/cppblog/c23-support-in-msvc-build-tools-14-51/)、[Visual Studio release history](https://learn.microsoft.com/en-us/visualstudio/releases/2026/release-history)をABI／STL／CRTおよびcomparison laneの一次根拠とし、Preview language modeをShippingへ使わない。Android Vulkan profileの正式名は[Android Vulkan Profiles](https://developer.android.com/ndk/guides/graphics/android-vulkan-profile)とKhronosの[Vulkan Profiles repository](https://github.com/KhronosGroup/Vulkan-Profiles/tree/main/profiles)を一次根拠とする。

Androidは[Android Gradle plugin API reference](https://developer.android.com/reference/tools/gradle-api)で2026-07-30時点のCurrent Release exact `9.3.1`とPreview `9.4.0-alpha07`の分離を確認し、[Android Gradle plugin 9.3 release notes](https://developer.android.com/build/releases/agp-9-3-0-release-notes)で9.3 familyのGradle 9.5.0、Build Tools 36.0.0、JDK 17を確認した。Google Mavenのexact coordinate `com.android.tools.build:gradle:9.3.1`を選択し、9.3.0、dynamic patchまたは9.4 Previewへ読み替えない。[NDK downloads](https://developer.android.com/ndk/downloads)と[NDK revision history](https://developer.android.com/ndk/downloads/revision_history)でcurrent stable r29 revision 29.0.14206865を確認した。公式default NDK 28.2.13676358とMiraikanai-selected r29を混同せず、全tupleは同一baselineのfresh Qualificationでだけ成立させる。

Apple pinは[Xcode 26.6 release notes](https://developer.apple.com/documentation/Xcode-Release-Notes/xcode-26_6-release-notes)、[Xcode support matrix](https://developer.apple.com/support/xcode/)、Build分離は[Xcode Cloud security](https://developer.apple.com/xcode-cloud/security/)を一次根拠とする。

## 11. 明示的に採用しないもの

- floating version、range、`latest`、Preview artifactのShipping採用
- 複数Compiler、Generator、JavaScript runtime、package manager、lockfileの恒久併存
- Vendor sourceの手動copy、hashなしdownload、Build中のNetwork取得
- License未確認、SBOM未記載、public APIへVendor型を露出するDependency
- Toolchain不一致時のPATH fallback、別Generator fallback、旧version fallback
- Slang、GLSL、WGSL等の追加Source frontendを初期Project Shader baselineへ併設すること。採用する場合はDependency ChangeSet、言語意味差、全Target artifact／reflection、既存HLSL fixture、rollbackを独立Qualificationする
