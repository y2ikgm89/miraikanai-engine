# Miraikanai Engine Apple Platform Contract

- 文書ID: mirakan.arch.platform-apple
- 状態: review
- 正本範囲: Apple Target／Distribution Profile、Toolchain Build mapping、C ABI／Objective-C++ bridge、UIKit／Metal／Audio Adapter、Apple Asset package、unsigned-build／signing／upload separation、Xcode Cloud mapping、Privacy Manifest、TestFlight／App Store、Apple device qualification／release gate
- 非正本範囲: exact Tool／SDK／OS／library version・hash・license・URL、Mobile共通schema／lifecycle意味／aggregate cap、Renderer共通contract、Input／Audio／UI意味、Asset lifecycle、AI authorization／Evidence envelope。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core architecture](../02-foundation/core-architecture.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Native game module](../03-authoring/native-game-module.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[Render Graph](../06-rendering/render-graph.md)、[Mobile Common](mobile-common.md)、[Input](input.md)、[Audio](audio.md)、[UI／Text](ui-text-localization-accessibility.md)
- 外部根拠検証日: 2026-07-21

## 1. Profile、Build mapping、C ABI boundary

`apple_mobile_v1`はiPhone／iPadのarm64 Shipping Targetである。`apple_bundle_v1`はbundle＋self-hosted unmanaged delivery、`apple_managed_assets_v1`はToolchain／Store policyが定めるminimum OSを持つ別variantのmanaged deliveryであり、同じbinary variantへ混在させない。Simulatorはfunctional test用で実機packageの代替にしない。

host OS、Xcode、SDK、deployment target、CMake／Ninja、Metal tool、generator、supported device／GPU baselineのexact valuesは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)のApple profileだけが所有する。本書はそれらをBuild Driver、C ABI App shell、archive、Distributionへ写像する。

| Driver Profile | purpose | package result |
|---|---|---|
| `apple_cx0_xcode_v1` | baseline header frontend、App shell／link | unsigned development payload |
| `apple_modules_probe_ninja_v1` | Named Module compile-only probe | package／promotion不可 |
| `apple_modules_ninja_xcode_v1` | portable C++ archive＋Xcode App shell／final link | `UnsignedApplePayloadV1` |

Driver Profile ID、Generator、resolved Build tree、Toolchain lock、package ownerは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)のclosed matrixへ一致させる。C++ Frontend／Target／Configuration／Toolchainが異なるobject、BMI、archive、log、Receiptを共有しない。

language boundaryはportable C++ core、generated C ABI header、opaque handle、Objective-C／Objective-C++ Adapterである。Xcode App shellはC++ Named Moduleをimportせず、generated C ABIへ接続する。Objective-C objectをCommon C++ object、MCD、Saveへ保存せず、UIKitはmain thread、Adapter `.mm`はARC、C++／Metal wrapperはmove-only generation handleで所有する。

WindowsはApple用Source generation、common Asset Cook、portable shader validationまで行えるが、Apple platform binary、final Metal library、arm64 final link、archiveを生成しない。Apple WorkerはToolchain lockを再検証し、C++ archive、Metal artifact、App shellを一つのunsigned payload manifestへ結ぶ。

## 2. UIKit、lifecycle、Input／Audio、Metal

Application shellはscene-based lifecycleを使い、sceneごとに`SurfaceGeneration`を管理する。drawing surfaceはToolchain-approved Metal view／layer Adapterで、drawableをframeの可能な限り遅い時点で取得し、CPU処理中に長期保持しない。safe area、display scale、orientation、preferred frame rateを`DisplaySnapshot`へpublishする。

scene callbackは[Mobile Common](mobile-common.md)のclosed lifecycle stateへ正規化する。inactive／background時はCommon checkpoint、audio／GPU停止、surface retirementを行い、termination callbackだけへSaveを依存させない。

required device capabilityはToolchain profileのarm64／Metal／minimum performance classからgenerated metadataへ出力する。公開後にminimum capabilityを強める変更は同じProfileのsilent updateにせず、新しいTarget Profileとmigration decisionを要求する。

InputはUIKit touch／pointerをprimary path、Game Controllerをoptional device pathとして[Input](input.md)の`DeviceReading`／Actionへ正規化する。controller-required Gameでもprimary menu／play操作にtouch fallbackを持つ。text composition、selection、software keyboardは`ITextInputService`へ集約する。motion inputはactivated Capability、privacy declaration、sampling budgetがある場合だけ使う。

Audioは`AVAudioSession`でcategory／route／interruption、AudioUnit callbackで[Audio](audio.md)のMixer／PCM ringを接続する。interruption、route change、media service resetはgeneration付きvalue eventにし、callback内allocation、lock、log、World callを禁止する。

Metal Adapterはheap／memoryless attachment、offline shader library／binary archive、command buffer completionを[Render Graph](../06-rendering/render-graph.md)へ写像する。drawable、command buffer、resourceはsubmission serial完了前にreuse／releaseしない。drawable timeout、background、device faultを別failureにする。

AA／temporal／dynamic resolution／history reset／provider resolverはRender Graph ownerを参照する。Appleはfeature family、sample count、tile memory／bandwidth、format／resolve、API availability、device resultを`CapabilitySignature`とQualification Receiptへ供給し、MSAA、MetalFX、indirect、mesh／ray／neural techniqueを実機Gateなしで有効化しない。minimum deviceにないoptional converter／argument featureをbaseline shader pathへ要求しない。

offline shader pathはToolchain lockのportable sourceからintermediate、MSL、Metal libraryへ変換する。Shippingへcompiler、shader source、authoring reflectionを含めない。unqualified Target-limited Materialはexplicit fallbackとvisual diffなしにCookしない。

process physical footprint、available memory、memory warning、Metal allocated size、allocator domainを別系列として取得し、Mobile Common aggregate capへ判定する。thermal signalは`Nominal | Warm | Serious | Critical`へ変換する。Development sanitizer、thread／main-thread checker、Metal validationは別jobで実行し、同時有効化でfailure sourceを混同しない。

## 3. Asset packaging、privacy、runtime content

`apple_bundle_v1`は`install`をapp bundle、`essential | prefetch | on_demand`を同名policyのself-hosted unmanaged deliveryへ写像する。`essential`を使用しないProjectはfirst playableに必要な全Assetを`install`へ含める。deprecated delivery mechanismを新規採用しない。

`apple_managed_assets_v1`はToolchain／Store policyが許す別minimum-OS variantだけでApple-hosted managed deliveryを使う。bundle variantとmanaged variantのTarget／Distribution ref、package、TestFlight lane、Save compatibilityを分離する。Store size／pack limitsとminimum OSは`store_policy.lock`からvalidatorへ注入し、本書へ時点依存値を複写しない。

`AssetChunkManifest`のsize、content hash、signature、dependency closure、Target Profileが全合格するまでmountしない。delivery packageへMach-O、dynamic library、shader source／runtime compiler、その他executable contentを混入しない。初回起動にはtutorialまたはmeaningful first playableをinstall contentとして含め、空のdownloader screenだけにしない。

`ProjectPrivacySpec`、`PrivacyInfo.xcprivacy`、required-reason API declaration、embedded third-party SDK privacy manifest／signature、actual binary behaviorを一致させる。telemetry、crash、AI prompt、generated contentを同じconsent purposeにまとめず、credential、token、signing key、personal dataをAI prompt／Build log／crash metadataへ含めない。

Shipping Runtimeはstructured dataだけを変更し、post-review executable code／shader pipelineをdownload／loadしない。Runtime content safety、rollback、offline last-validは[Mobile Common](mobile-common.md)を使用する。

## 4. Unsigned Build、Signing、Upload separation

Shipping backendは`apple_xcode_cloud_v1 | apple_self_hosted_split_v1`のclosed setである。前者はToolchain profileのXcode Cloud build／managed signing／TestFlight handoffへ写像し、独自distribution keyをBuild scriptへ渡さない。後者は`AppleUnsignedBuildWorkerV1`、Apple Signing Service、Store Upload Serviceを別identity／workspace／credentialへ分離する。

`AppleUnsignedBuildWorkerV1`はnon-admin task identity、signed immutable base、taskごとのephemeral VM／volume、no general egress、brokered content-addressed input／output、bounded CPU／memory／process／file／output／wall-timeを必須とする。User home、Keychain、other Project、Source-control credential、Signing／Upload endpointをmountしない。task後にReceipt確定後diskを破棄し、workspace削除だけをclean workerとみなさない。

Worker outputは`UnsignedApplePayloadV1`とbounded log／Receiptだけである。payloadはTarget／Distribution ref、Engine／Source／Toolchain ref、C ABI archive、Metal artifact、App shell、bundle manifest、entry path／size／content hash／executable kindを持つ。guest filesystem archive、absolute／parent path、case／Unicode collision、symlink／hardlink、undeclared nested codeを拒否する。

Apple Signing Serviceはfixed `UnsignedApplePayloadV1`、approved entitlement／provisioning ref、[AI Security／Approval](../01-governance/ai-security-approval.md)のRelease decision refだけを受ける。Source、Project、workspace、Build script、compiler、arbitrary shell／environmentを受け取らない。first-party packagerがbundle path、Mach-O、nested code、Info、entitlement、Privacy Manifest、provisioning match、content hashを検査し、nested signing、archive／export／validationを行う。private keyをfile、environment、stdio、logへ展開しない。

Store Upload Serviceはsigned package、`AppleSigningReceiptV1`、short-lived upload credentialだけを持ち、Source／Build script／signing keyを持たない。Receiptはunsigned／signed root、key ref、certificate-chain ref、entitlement／profile hash、Tool ref、validation resultを記録する。UploadはreceiptとRelease decision refが一致しなければ拒否する。

Remote Apple serviceはmutual-authenticated transportでrole別versioned RPC `status | submit_unsigned_build | submit_signing | submit_upload | cancel | fetch_artifact | fetch_receipt | fetch_log`だけを公開する。arbitrary shell／path／environmentをWindows Editorへ公開しない。inputはcontent-addressed manifest、lock digest、signed request schema、outputはsecret-free artifact／Receiptだけとする。

`apple_self_hosted_split_v1`は次のsource-free signing conformanceを全て満たすまでactivateしない。

1. WorkerにKeychain identity、provisioning material、Store credentialがない状態で同一inputからunsigned arm64 payloadを再現する。
2. Signing ServiceにSource、Project、Build script、compilerを置かず、fixed RPC／toolだけでnested signing、archive／export、signature／archive validationを完了する。
3. internal TestFlightへuploadし、minimum deviceでinstall／launch／saveする。
4. malicious Build scriptのKeychain／environment／network exfiltrationがsecretを得られないnegative fixtureを通す。
5. Signing ServiceがSource／script、path trick、undeclared nested code、entitlement replacement、payload hash mismatchを拒否する。

不合格時はcredential-bearing combined workerへfallbackせず、`apple_xcode_cloud_v1`を使用するか`UnsupportedCapability`でApple Shippingを停止する。Toolchain profile更新ごとに同じconformanceを再実行する。

## 5. Device tests、failure、release gate

Minimum laneはToolchain profileのminimum iPhone一台とiPad一台、Reference laneはnewer phone／tablet class、High laneはoptional graphics／high-refresh classとする。device交換はsame commit／package／input traceのbridge baselineを残す。Simulatorはfunctional smokeだけで、GPU、audio／touch latency、memory、thermal合否に使わない。

Apple fixtureはclean install／upgrade、cold／warm start、inactive／background／foreground、termination Save recovery、scene／surface recreation、rotation／safe area、touch／controller／IME、audio route／interruption／media reset、offline Asset interruption／resume／hash mismatch、physical footprint pressure、Metal allocation failure、thermal／battery saver、shader／golden、archive／entitlement／Privacy Manifest、TestFlight installを含む。10分performance、30分thermal、2時間enduranceをphysical deviceで行う。

| Failure | 結果 |
|---|---|
| Toolchain／Driver／C ABI mismatch | configure／link前に失敗 |
| stale scene／drawable／callback generation | event／drawable破棄 |
| Metal capability不足 | `UnsupportedDevice`、launch拒否 |
| drawable unavailable | surface unavailableとして待機、GPU faultと混同しない |
| Metal execution fault | diagnostic＋Save保護、safe fault／exit |
| privacy／entitlement／provisioning mismatch | signing／release block |
| signing separation conformance failure | Cloud backendまたはfail closed |
| Asset partial／hash／dependency failure | mount拒否、last-valid維持 |
| debug／source／dynamic code混入 | release block |

Release候補はTarget／Distribution／Toolchain／Store policy refs、portable＋Apple Cook、C ABI archive／Metal library／archive ABI・size・signature、Minimum／Reference device lifecycle／Input／Audio／Save／graphics、Mobile aggregate cap、thermal／endurance、Asset delivery、Privacy Manifest、TestFlightへ合格する。WorkerにSigning／Upload secretがなく、Signing ServiceにSource／scriptがなく、Upload ServiceにSource／signing keyがないことをservice identityとnegative fixtureで証明する。Approval class、one-shot transaction、Evidence gradingはGovernance ownersへ委譲する。
