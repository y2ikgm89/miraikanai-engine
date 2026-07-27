# Miraikanai Engine Android Platform Contract

- 文書ID: mirakan.arch.platform-android
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Android Target／Distribution Profile、Toolchain Build mapping、GameActivity／controller／frame-pacing Adapter、Vulkan Target mapping、Android package／Play delivery、lifecycle、permission／privacy、16 KiB page compatibility、Android device qualification／release gate
- 非正本範囲: exact Tool／SDK／library version・hash・license・URL、Mobile共通schema／lifecycle意味／aggregate cap、Renderer共通contract、Input／Audio／UI意味、Asset lifecycle、AI authorization／Evidence envelope。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Mobile Common](mobile-common.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Render Graph](../06-rendering/render-graph.md)、[Input](input.md)、[Audio](audio.md)、[UI／Text](ui-text-localization-accessibility.md)
- 関連文書: [AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core architecture](../02-foundation/core-architecture.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Native game module](../03-authoring/native-game-module.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[Render Graph](../06-rendering/render-graph.md)、[Project Shader](../06-rendering/project-shader.md)、[Mobile Common](mobile-common.md)、[Input](input.md)、[Audio](audio.md)、[UI／Text](ui-text-localization-accessibility.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-27

## 1. ProfileとBuild mapping

`target.android.mobile`はphone／tablet／foldableのShipping候補Target、`package-profile.android.play`はGoogle Play Distribution候補Profileである。minimum／compile／target API、ABI、Vulkan Profile、NDK、Gradle／AGP、Build Tools、JDK、CMake／Ninja、AndroidX Games、Oboeの候補値は[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)のAndroid baselineだけが所有する。本書はそれらをProfile、Build、packageへ写像する。

根拠: official-spec／project-decision — [Google Play target API要件](https://developer.android.com/google/play/requirements/target-sdk)、[NDK revision history](https://developer.android.com/ndk/downloads/revision_history)、[Android Vulkan Profiles](https://developer.android.com/ndk/guides/graphics/android-vulkan-profile)、[48 dp touch target guidance](https://developer.android.com/design/ui/mobile/guides/foundations/accessibility)に従う。minimum API 29、NDK r29、Android Vulkan Profile 2022、対応Device範囲はMiraikanaiの候補判断であり、Googleの既定値や市場coverage保証ではない。

Shipping ABIはToolchain profileのprimary arm64 ABIだけ、secondary ABIはDevelopment／CIだけに許可する。STLはToolchain profileが指定する一種へappとnative dependencyを統一する。Runtime probeとpackage inspectionはresolved OS／ABI／graphics requirementsを[Mobile Common](mobile-common.md)の`MobileCapabilitySignatureV1`へ記録し、不足をquality fallbackで隠さない。AndroidはPlatform観測値の写像だけを所有し、共通field setを再定義しない。

正規DriverはToolchain ownerの`driver.android.gradle-ninja`である。checked-in WrapperがAndroid resource、manifest merge、DEX、native external build、APK／AAB assemblyを統括し、CMake／Ninjaはfirst-party C++ targetとnative artifactだけを所有する。Editor、AI、CIはBuild GatewayからDriverを呼び、raw `ninja`、`cmake -G`、daemon内部path、`ndk-build`、Makefiles系をProduct Build入口にしない。

| Mirakan configuration | Android variant meaning | C++ configuration | rule |
|---|---|---|---|
| `Development` | debug／debuggable | `Debug` | Shipping不可 |
| `Profile` | profiling、non-debuggable policy | `RelWithDebInfo` | Shipping不可 |
| `Shipping` | release | `Release` | Release gate合格時だけ |
| `ASan` | sanitizer test | Debug＋AddressSanitizer | test packageだけ |

variantは署名、debuggable、optimization、sanitizer、package suffixを明示し、unknown variantを近いbuild typeへfallbackしない。Build tree identityは`module_id × variant_id × ABI × C++ Frontend Profile × toolchain_lock_hash`で、Variant／ABI／Profileを同じ object、BMI、staging directoryへ混在させない。ReceiptはDriver Profile ID、Generator、resolved tree、Toolchain ref、package ownerを記録する。

## 2. GameActivity、lifecycle、Input／Audio Adapter

Application shellはToolchain lockのGameActivityを使用する。Kotlin／JavaはActivity、Platform service、permission、Store glueだけを担当し、Gameplay stateを所有しない。EngineとGame native artifactはToolchain ABI contractどおりlinkし、GameActivityからgenerated stable entry pointを呼ぶ。

generated manifestはToolchain profileのminimum／target API、required Vulkan feature、ABI、orientation／resize、permissionをTarget／Project specから生成する。OpenGL ES fallbackを暗黙に宣言しない。manifest filterだけで合格とせず、Store Device CatalogとRuntime probeを併用する。

GameActivity callbackは[Mobile Common](mobile-common.md)のclosed lifecycle stateへ正規化する。`SurfaceView`の`ANativeWindow`はmove-only wrapperでacquire／releaseを一対にし、`SurfaceGeneration`を記録する。surface destroy／process killでWorldを破棄せず、Save／recoveryはMobile Commonを使う。

JNI local referenceはcall scope、global referenceはRAII wrapper、thread attachはscope guardで所有する。native callbackからJNI objectを長期保持しない。Activity／UI操作はmain threadへdispatchし、render／audio callbackからJava UI、World、filesystemを直接呼ばない。

InputはGameActivity bufferをproducer／consumerでswapし、touch／key／mouseを[Input](input.md)の`DeviceReading`と`InputSnapshot`へ正規化する。Toolchain lockのcontroller Adapterはeventを同じAction mapへforwardし、Platform buttonをGameplayへ出さない。IME compositionはGameTextInput経由の`TextCompositionEvent`へ写像し、Back gesture／system navigationをGame Actionへ混同しない。

Androidのprimary interactive targetは48×48 dp以上とする。logical `ui_lu`の共通最小hit targetは[UI／Text](ui-text-localization-accessibility.md)が所有し、本節はAndroid物理単位Gateだけを所有する。視覚glyphだけでなく実際のhit rectangleで満たし、safe area、display scale、large text、隣接targetとの分離を含むUI fixtureで検証する。

Audioは[Audio](audio.md)のMixer／PCM ringをToolchain lockのAndroid output Adapterへ接続する。actual backend、sample rate、burst、latency、xrunを`MobileCapabilitySignatureV1`へ記録する。callback内allocation、mutex、file I/O、log、JNI、Asset lookupは禁止し、route／focus／lifecycle eventはAudio control threadへvalue eventとして渡す。

frame pacing AdapterはToolchain lockのAndroidX Games Frame Pacingをpresent timingへ使用し、busy wait／fixed sleep capを禁止する。Controller／Frame Pacing／GameActivity objectはprivate Adapterから公開しない。

## 3. Vulkan、shader、memory／thermal hardening

Android graphicsはToolchain ownerのVulkan／Android Vulkan Profile baselineを`target.android.mobile`へ写像する。起動時にloader API、device feature、sample／format／queue、memory budgetをprobeし、required baseline不足は`UnsupportedDevice`として起動前に説明する。新しいoptional Profileは`mobile_high` Capabilityにできるがbaseline contentの必須条件にしない。

Manifest filter、Play Device Catalog exclusion、Runtime Capability probeの三段をRelease manifestへ記録する。未知modelを自動合格にせず`unverified`としてphysical-device qualificationへ送る。Adreno系とMali系をminimum laneに含め、model／SoC／driver固有fallbackをSource AssetやGameplayへ焼き込まない。

orientation changeはswapchain transformとprojectionでpre-rotationし、compositorの余分なrotationを避ける。validation layerはDevelopment packageだけに含める。device lostは同一sessionで無条件復旧せず、crash-safe diagnosticとSaveを保護してrender fault画面またはsafe exitへ移る。

Render Graph、resource access／barrier、`GpuSubmissionSerial`、AA／temporal resolver、dynamic resolution、history reset、provider qualificationは[Render Graph](../06-rendering/render-graph.md)を参照し、MobileのFrames-in-flight、AA intent、dynamic resolution、Frame Generation選択policyは[Mobile Common](mobile-common.md)を参照する。AndroidはVulkan feature／format／sample count／tile memory／bandwidth／driver結果を`MobileCapabilitySignatureV1`とQualification Receiptへ供給し、MSAA、temporal、frame generation、ray／neural techniqueを実機Gateなしで有効化しない。

offline shader pipelineは[Project Shader](../06-rendering/project-shader.md)とToolchain lockのProfileを使い、Android専用`ProjectShaderArtifactSetV1`のSPIR-V artifactだけをApplication packageへ格納する。Package validatorはAndroid Target Profile、Engine baseline、`ProjectShaderQualificationReceiptV1`、artifact／interface hashを照合する。Shippingへcompiler、shader source、authoring reflectionを含めない。`renderer-profile.portable-mobile`はmesh／ray、unbounded bindless、wave-size依存、geometry／tessellationをrequired featureにせず、unsupported Module／Technique／Materialはexplicit Target fallbackなしにCookしない。

Android textureはPlay texture compression format targetingを使い、同一Asset IDへASTC primary artifactとETC2 fallback artifactを対応付けてpackageする。Package validatorは[Mobile Common](mobile-common.md)の7-field texture projection、`target_format`、content hash、Target ProfileをASTC／ETC2 payloadごとに照合する。ASTCまたはETC2の不足、同一Asset IDへの誤対応、format／manifest／payload不一致はpackage promotionを拒否し、単一artifactへのsilent fallbackやRuntime Basis／Universal Texture transcodeを行わない。Pixel Art／UI／maskはMobile CommonのRGBA8／用途別lossless規則を使う。

`onTrimMemory`とOS pressureは`MemoryPressureLevel`、ADPF／thermal signalはMobile Commonのpressure／thermal levelへ正規化する。API unavailableでもcorrectnessを失わない。Android canonical process footprintはrelease deviceのpackage PIDに対するTOTAL PSSで、RSS、allocator total、Vulkan allocationを別系列として保存する。判定は[Mobile Common](mobile-common.md)のaggregate cap表に対し、TOTAL PSSをProcess footprint列、Engine allocator totalをEngine CPU列、Vulkan allocation合計をGPU working set列へ対応させる。

全Shipping native libraryは16 KiB page size compatibleでなければならない。compiler defaultを信用せず、ELF segment alignment、ZIP entry alignment、AAB／generated APKをToolchain lockのinspection toolsで検査する。arm64 hardeningはbranch protection、RELRO、stack protection、FORTIFYを有効にし、例外はPlatform security decisionを必要とする。Development memory sanitizerとShipping sampled allocator diagnosticは別fixtureで検証する。

Android Shipping crashは[Mobile Common](mobile-common.md)の共通crash metadata契約に従う。native crash handlerはcrash-safe boundary内でpreallocated metadataだけを書き、in-process handlerで複雑なallocation、lock、network upload、Saveを行わない。dumpとcrash metadataはapp-private storageへ保存し、credential、token、personal data、source pathのredactionをpackage／device fixtureで検証する。自動uploadと外部crash serviceの利用はactivated Capabilityと[AI Security／Approval](../01-governance/ai-security-approval.md)のdecision refを必要とする。Crash recordとSession／Replayの関連付けは[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)を参照する。

## 4. Package、Play delivery、permission／privacy

Android packageはAABを正規Shipping artifactとする。`UnsignedMobilePackageV1`はTarget／Distribution Profile、Engine commit、Source root、Toolchain lock ref、Build Receipt、SBOM ref、各entryのnormalized path／size／content hash／executable kindを持つ。absolute／parent traversal／case・Unicode collision、symlink、hardlink、device、socket、alternate stream、sparse entry、manifest外fileを拒否する。

`AssetChunkManifest.delivery`を次へ写像する。

| Common delivery | Play mapping |
|---|---|
| `install` | base module |
| `essential` | install-time asset pack |
| `prefetch` | fast-follow asset pack |
| `on_demand` | on-demand asset pack |

baseへ収まらない`install`を自動で`essential`へ変えない。Asset packへnative library、DEX、shader source／binary／pipelineその他のexecutable contentを入れない。Play size／pack／cellular confirmation limitは[Mobile Common](mobile-common.md)の唯一の共通schema `StorePolicyLock`からvalidatorへ注入し、本書に物理lock schemaや時点依存値を複写しない。partial／hash mismatch／dependency mismatchはmountせずlast-valid namespaceを維持する。

default permissionは0である。Capabilityからmanifest candidateを生成するが、用途、user explanation、privacy declaration、authorizationは[AI Security／Approval](../01-governance/ai-security-approval.md)のdecision refがなければShipping manifestへ反映しない。unused permission、background mode、URL scheme、undeclared serviceをpackage inspectionで拒否する。

`ProjectPrivacySpec`とAndroid Data Safety declaration、manifest、embedded SDK behavior、runtime telemetry scanを一致させる。AI prompt、crash、analytics、generated contentのpurposeをまとめない。dynamic executable codeをremote sourceまたはAsset deliveryからloadしない。

Signing Serviceはfixed unsigned AABとgovernance ownerのRelease decision refだけを受け、Gradle、Source、Build scriptを受け取らない。upload keyとPlay API credentialは別identityへ分け、debug credential／debuggable packageをShipping入力として拒否する。正規AAB署名／verification toolとexact behaviorはToolchain ownerを参照する。`AndroidSigningReceiptV1`はunsigned／signed root、key ref、certificate-chain ref、profile hash、Tool ref、verification resultを持つ。

## 5. Device tests、failure、release gate

`AndroidMinSdkCoverageReceiptV1`はminimum SDKの市場妥当性だけを所有し、compile／target SDK要件やdevice conformanceの代用にしない。Product Ownerは計測結果を見る前に`target_regions[]`、`form_factors[]`、coverage metric、合格閾値、snapshot freshnessを承認する。Android Ownerは同じAABをPlay Consoleへuploadした後、[Reach and devices](https://support.google.com/googleplay/android-developer/answer/10770882?hl=en)のAndroid version別exportと[Device catalog](https://support.google.com/googleplay/android-developer/answer/7353455?hl=en)のsupported／excluded device exportを取得し、`aab_sha256`、`minimum_sdk`、`target_sdk`、`snapshot_at`、filter、metric、observed coverage、threshold、export SHA-256、Play Console account／app identityの非secret refをReceiptへ記録する。

閾値未承認、AAB未upload、export欠落、filter不一致、snapshot失効、coverage未達のいずれかではAPI 29を市場coverage達成として扱わず、`wp.platform.mobile-offline`のAndroid Target Gateをfail closedにする。公開Web上の一般的なOS分布、Device Catalogのmodel数、target API policy、別appの値を代用しない。minimum SDKを変更する場合はToolchain baseline、manifest、device matrix、Product Registry、Package Receiptを同じChangeSetで更新する。

Minimum laneはToolchain profileのminimum API／Vulkan Profileに合格するarm64 Adreno系1台とMali系1台、Reference laneはcurrent profileのphone／tablet／foldable、High laneはoptional graphics／high-refresh Profileとする。device交換はsame commit／package／input traceのbridge baselineを残す。Emulatorはfunctional smokeだけに使う。

Android fixtureはclean install／upgrade／uninstall、cold／warm start、background／foreground、process kill Save recovery、surface loss／rotation／resize／fold、touch／controller／IME、audio route、offline、PAD interruption／resume／hash mismatch、PSS pressure、GPU allocation failure、thermal／battery saver、Vulkan variant／golden、16 KiB package、permission／privacy scanを含む。texture fixtureは同一Asset IDのASTC primary＋ETC2 fallback package、各7-field照合、片方不足／誤対応／payload tamperのpromotion拒否、Runtime transcode不在を検証する。touch fixtureはprimary interactive targetの実hit rectangleが48×48 dp以上であることを検証する。crash fixtureは[Mobile Common](mobile-common.md)の共通crash metadata必須fieldの存在、除外対象の不在、redactionを検証する。10分performance、30分thermal、2時間endurance runをphysical deviceで行う。

| Failure | 結果 |
|---|---|
| Toolchain／Driver／variant mismatch | configure前に失敗 |
| GameActivity／JNI stale generation | callback／event破棄 |
| Vulkan baseline不足 | `UnsupportedDevice`、launch拒否 |
| Vulkan device lost | diagnostic＋Save保護、safe fault／exit |
| ASTC／ETC2 artifact不足／format・manifest不一致 | package promotion拒否、last-valid artifact維持 |
| 16 KiB alignment failure | package promotion拒否 |
| primary interactive targetが48×48 dp未満 | UI／device qualification失敗、release block |
| PAD partial／hash／dependency failure | mount拒否、last-valid維持 |
| permission／Data Safety mismatch | release block |
| crash metadata不足／private data混入 | crash／privacy qualification失敗、release block |
| debug／source／dynamic code混入 | release block |

Release候補はTarget／Distribution／Toolchain／Store policy refs、all-target Cook、AAB ABI／size／alignment／signature、Minimum／Reference device lifecycle／Input／Audio／Save／graphics、Mobile aggregate cap、thermal／endurance、PAD、permission／privacy、Shipping content scanへ合格する。Build identityにSigning／Upload secretがなく、Signing identityにSource／scriptがなく、Upload identityにSource／signing keyがないことをnegative testで証明する。`HyperVIsolatedWorkerV1`、approval class、one-shot `ReleaseTransactionV1`、Evidence gradingの意味はGovernance ownersへ委譲し、本書はAndroid package／service separationの合格入力だけを供給する。
