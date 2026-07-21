# Miraikanai Engine Mobile Common Contract

- 文書ID: mirakan.arch.platform-mobile-common
- 状態: review
- 正本範囲: Mobile共通Target schema、Platform Port境界、lifecycle／surface／save／recovery、renderer接続境界、Asset delivery意味、touch／safe area、memory／thermal policy、device workflow、privacy model、共通qualification
- 非正本範囲: Android／Apple固有profile値・build・package・store・signing、external Tool／SDK version、共通Runtime phase／budget、Asset import／cook／promotion、Renderer内部契約、Input／Audio／UI domain意味、AI authorization／Evidence envelope。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core architecture](../02-foundation/core-architecture.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Gameplay programming model](../03-authoring/gameplay-programming-model.md)、[Native game module](../03-authoring/native-game-module.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[Render Graph](../06-rendering/render-graph.md)、[Input](input.md)、[Audio](audio.md)、[UI／Text](ui-text-localization-accessibility.md)、[Android](android.md)、[Apple](apple.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論と所有境界

MobileはWindows Editor、共通Runtime Contract、private Platform Adapterで構成する。AndroidとAppleでGameSpec、World、Asset ID、GameplayDefinition、portable C++、Input Action、Save schema、Renderer intent、AI ChangeSetをforkしない。OS lifecycle、surface、graphics、audio、text、content delivery、memory／thermalだけをAdapterへ隔離する。

初期Mobileはoffline single-playerをpermissionなしで成立させる。広告、課金、push、platform account／achievement／leaderboard、cloud save、camera／microphone／location、background gameplayは[Product Plan](../00-product/product-plan.md)のactivation対象であり、`PlatformServiceCapability`、permission、privacy、offline／failure、server authorityをOwner仕様で閉じるまでRuntimeへ公開しない。

Android固有のBuild／GameActivity／Vulkan／Play／permission／releaseは[Android](android.md)、Apple固有のBuild／C ABI bridge／Metal／signing／TestFlight／App Storeは[Apple](apple.md)だけが所有する。本書にはGradle、AGP、NDK、Xcode、Provisioning、App Store Connectの値または手順を置かない。

## 2. Target、Distribution、Capability schema

Projectはfree-form platform条件を保存せず、次の型付きcontractを使う。

```text
TargetProfileRef
  profile_id
  profile_revision
  toolchain_profile_ref
  render_quality_tier
  memory_class
  target_fps
  optional_capabilities[]

DistributionProfileRef
  profile_id
  package_id
  version_name
  version_code
  signing_profile_ref
  content_delivery_policy

ProjectMobileSpec
  orientation_policy
  resize_policy
  safe_area_policy
  touch_fallback_policy
  permissions[]
  privacy_declarations[]
  content_safety_profile_ref
```

`profile_revision`不一致、unknown Capability、禁止permission、Target別aggregate cap超過、minimum deviceとCooked artifact要求の不整合はCook前に拒否する。`CapabilitySignature`はOS／device class、GPU feature、memory class、display、input／audio availability、thermal sourceをEngine-owned enum／valueへ正規化し、vendor objectやdisplay名を永続化しない。

Store要件の時点依存dataは共通参照schemaだけを持つ。

```text
StorePolicyLock
  checked_at_utc
  effective_requirements[]
  official_source_refs[]
  reviewer_ref
  toolchain_lock_digest
```

Android／Appleのrequirement値、refresh／submission procedureは各Platform ownerが決定し、external version／URLはToolchain ownerのsource refへ解決する。

旧`MobileSigningReceiptV1`のcross-platform aliasは、signing procedureをCommonへ逆流させるためduplicate-removeする。Androidは`AndroidSigningReceiptV1`、Appleは`AppleSigningReceiptV1`を各Ownerで定義し、共通provenanceへの接続だけを[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)へ委譲する。

共通memory class IDは`mobile_baseline | mobile_standard | mobile_high`、distribution意味は`install | essential | prefetch | on_demand`のclosed setである。profileのOS／ABI／SDK／graphics exact baselineとpackage mappingは各Platform ownerがToolchain refで決定する。

## 3. Platform Portと公開境界

| Port | 共通契約 | owner mapping |
|---|---|---|
| `IGraphicsDevice` | resource／pipeline／submission／present／budget query | [Render Graph](../06-rendering/render-graph.md)とPlatform graphics Adapter |
| `IApplicationSurface` | logical／pixel extent、orientation、safe area、surface generation | Android／Apple |
| `ILifecycleService` | active／inactive／suspended／surface unavailable／terminate／memory pressure | Android／Apple |
| `IInputDeviceHub` | Action、touch、pointer、controller snapshot | [Input](input.md)とPlatform Adapter |
| `ITextInputService` | composition、selection、commit、cancel | [UI／Text](ui-text-localization-accessibility.md)とPlatform Adapter |
| `IAudioDevice` | callback、route、latency、interruption | [Audio](audio.md)とPlatform Adapter |
| `IHapticsService` | named pattern、strength、duration、availability | [Input](input.md)とPlatform Adapter |
| `IContentDeliveryService` | chunk state、progress、verify、mount | 本書の意味、Platform delivery mapping |
| `IDeviceConditionService` | thermal、power、memory pressure、refresh range | 本書のpolicy、Platform signal mapping |
| `IPlatformCrypto` | secure random、hash、signature verify | Toolchain-approved Platform Adapter |

Public header／module／MCD／Saveへnative graphics handle、JNI／Objective-C object、OS enum、callback pointerを出さない。unsupported APIは意味のないsuccessではなく`UnsupportedCapability`を返す。Project C++はPortを直接includeせず、Engine Capabilityのtyped command／snapshotを使う。

Physical directory名は実装配置であり契約identityではない。`platform/contracts`だけが共通value／Portを公開し、Android／Apple AdapterからWorld／Gameplay moduleへ依存しない。Renderer、Audio、Input、ContentのBackendはprivateであり、Projectが選択するのはTarget／Distribution／Capability Profileだけである。

## 4. Lifecycle、surface、save、recovery

共通lifecycle stateはclosed set `Cold | Starting | Active | Inactive | Suspended | SurfaceUnavailable | Terminating`である。exact transition slot、tick、job dependency、lifetimeは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)が所有し、本書はPlatform eventの意味を所有する。

- `Active`: interaction、simulation、presentationを許可する。
- `Inactive`: foregroundだがinteraction不可。authoritative simulationをpauseする。
- `Suspended`: CPU／GPU／audio activityを停止し、復帰用checkpointが完了している。
- `SurfaceUnavailable`: WorldとGameplay stateは保持できるがpresentable surfaceがない。
- `Terminating`: best-effort通知であり、到達を前提にしない。

`SurfaceGeneration`はsurface create／resize／rotation／recreationごとに単調増加する。Render job、pointer／touch event、drawable／presentはcaptureしたgenerationとcurrent generationが一致する場合だけcommitする。不一致jobは破棄し、World、Save、Asset generationをsurface lossで破棄しない。GPU resourceのretire／history resetは[Render Graph](../06-rendering/render-graph.md)へ委譲する。

OS process killとtermination callback不達を前提とする。Saveはexplicit checkpoint、inactive／background transition、重要transaction commit後にgeneration付きtemporary fileへ完全writeし、flush、checksum、journal commit、atomic replaceする。復帰はschema version、content package set、last complete transaction、checksumを検証し、partial／stale generationをactive slotとして表示しない。Source／Derived lifecycle、Cook／promotion／rollbackは[Asset lifecycle](../03-authoring/asset-lifecycle.md)が所有する。

initial Mobile Runtimeはbackground simulation、network tick、Asset decodeを行わず、Platformが許すbounded checkpoint完了だけを使う。background audio／location／Bluetooth等はactivated `PlatformServiceCapability`なしに有効化しない。

## 5. Renderer境界とAsset delivery

Mobileは[Render Graph](../06-rendering/render-graph.md)の`AntiAliasingIntentV1`、`ResolvedAntiAliasingPlanV1`、`AntiAliasingResolutionErrorV1`、`TemporalFrameInputV1`、`ResolvedRendererProfileV1`、`GpuSubmissionSerial`、Provider排他、history reset、real／displayed frame分離を変更しない。本書はTarget Profile／Capability Signature／memory・thermal signalをResolverへ渡し、backend barrier、clip／depth、resource lifetime、AA／upscale／frame-generation algorithmを再定義しない。

Mobile baselineへWindows-only backend、ray／neural technique、unqualified temporal Providerを要求しない。`VirtualShadowBackendV1`、`RayTracedShadowBackendV1`、`ProjectShadowTechniqueV1`はMobile Capability Catalogへ暗黙登録せず、[Render Graph](../06-rendering/render-graph.md)のactivation／resolver contractへ委譲する。unsupported combinationは`UnsupportedByTarget`としてfail closedし、explicit fallbackとomitted reasonを返す。UI／text／pixel-locked layerはWorld dynamic resolutionやtemporal processingの対象外である。Target固有Vulkan／Metal mappingとdevice fixtureは各Platform ownerが持つ。

共通delivery manifestは次である。

```text
AssetChunkManifest
  chunk_id
  delivery: install | essential | prefetch | on_demand
  compressed_size
  installed_size
  sha256
  signature
  dependencies[]
  target_artifacts[]
```

mountはdownload完了だけでなく、size、content hash、signature、dependency closure、Target Profile一致の全合格後にatomic publishする。partial download、old manifest、hash mismatch、executable contentをlive Asset namespaceへ公開しない。Platform deliveryは同じfour-state meaningをPlay／Apple deliveryへ写像し、意味を自動変更しない。texture／audio／shaderのSource、Cook、Target artifact identityは[Asset lifecycle](../03-authoring/asset-lifecycle.md)、format／shader executionはRenderer／Audio ownerが所有する。

## 6. Touch、safe area、display、device workflow

`DisplaySnapshot`はpixel extent、logical extent、scale、safe-area insets、cutout regions、orientation、refresh range、HDR capability、`SurfaceGeneration`を持つ。World render resolutionとUI logical resolutionを分離し、safe area／orientation changeを同じframeのInput mappingより先にpublishする。

Touch OS IDは保存せず、contact開始から終了まで有効なgeneration handleへ変換する。Touch／controller／keyboardは同じ`InputActionId`へbindし、Platform key／buttonをGame ruleへ渡さない。contact cardinality、gesture、remap、haptics、Action semanticsは[Input](input.md)、hit target、large text、layout、Accessibility semanticsは[UI／Text](ui-text-localization-accessibility.md)が所有する。

Editor／AIはTarget／Distribution selector、Capability Matrix、phone／tablet／foldable／safe-area／cutout／orientation preview、touch／controller／software keyboard simulation、World／UI resolution、memory／frame／thermal status、Package Inspector、Platform impact／fallbackを同じProject snapshotから表示する。

`DeviceDebugHandshakeV1`はEngine protocol、Build、Project revision、Target、device identity、requested recording tier、retention boundを照合する。認証済みlocal bridgeは一台、一session、bounded timeだけ接続し、compatible GameplayDefinitionSet、structured data、already-cooked Assetだけをhot reloadする。C++、shader、native pluginはrebuild／reinstallする。切断時はcomplete chunkだけをread-only確定し、欠落rangeをgapとして残す。Shipping scanはdevice bridge、debug server、IDE attach、validation layer、raw trace、compiler、source path、credentialを拒否する。

## 7. Memory、thermal、privacy、Runtime AI

Mobile固有aggregate soft release capは次である。正規`ProcessFootprint`、Engine CPU domain、GPU working setを別々に測り、unified memoryを単純加算しない。Subsystem child allocation、measurement window、reservation／loan／backpressureは[Runtime performance／capacity](../04-runtime/performance-capacity.md)へ接続する。

| Class | Process footprint | Engine CPU | GPU working set | Render transient内数 | Streaming cache内数 | Emergency reserve内数 |
|---|---:|---:|---:|---:|---:|---:|
| `mobile_baseline` | 1,024 MiB | 768 MiB | 384 MiB | 128 MiB | 192 MiB | 64 MiB |
| `mobile_standard` | 1,536 MiB | 1,152 MiB | 640 MiB | 192 MiB | 320 MiB | 96 MiB |
| `mobile_high` | 2,560 MiB | 1,920 MiB | 1,024 MiB | 320 MiB | 512 MiB | 128 MiB |

pressure ratioは`max(process/cap, engine_cpu/cap, gpu/cap)`とする。closed levelは`Normal | Constrained | Severe | Exhausted`で、80%からprefetch停止／cache trim、90%からnonessential eviction／allocation拒否、100%またはOS criticalでcheckpointとsafe degradationを行う。Emergency reserveを通常quality維持へ使わず、authoritative Gameplay、Save、network stateを捨てない。

thermal levelは`Nominal | Warm | Serious | Critical`へ正規化する。Warmはpresentation qualityを一段下げ、Seriousはlower render target、volumetric停止、streaming concurrency低下、Criticalはminimum presentation、nonessential download停止、checkpoint後のsafe exitを許可する。回復は15秒以上安定後に一段ずつ行い、瞬間signalで往復しない。敵味方数、damage、collision、goal、spawn timingを端末都合で変えない。

`ProjectPrivacySpec`はdata category、purpose、retention、third-party sharing、account deletion、child-directed settingを持つ。telemetry、crash、AI prompt、generated contentは別purposeとして扱う。credential、access token、signing key、personal dataをAI prompt、Build log、crash dumpへ含めない。Platform manifest／store declarationとの一致は各Platform ownerが検査する。

Shipping Runtime AIはSchema／Capabilityで許可されたstructured dataだけを変更する。C／C++、native library、platform bytecode、script、shader source／pipeline、dynamic library、FFI、arbitrary Engine callを生成／download／loadしない。Runtime dataは`ContentSafetyProfile`、moderation、age／region policy、rate limit、report、audit ID、rollbackを持ち、network failure時はlast-valid local contentへ戻す。

## 8. Failureと共通qualification

| Failure | 結果 |
|---|---|
| Unsupported Target／Capability | Cook／launch拒否、required capabilityとremediationを表示 |
| stale surface／touch generation | job／event破棄、Worldは維持 |
| process kill／partial Save | last complete generationを復旧、partial slot非公開 |
| Asset chunk hash／signature／dependency mismatch | mount拒否、last-valid namespace維持 |
| memory Exhausted／thermal Critical | authoritative stateを維持して縮退、不能ならcheckpoint後safe exit |
| device bridge mismatch／gap | session拒否またはcomplete chunkだけ確定 |
| executable content in delivered Asset | package／mount拒否 |
| Runtime AI validation／moderation failure | Project／Save不変、last-valid content維持 |

共通fixtureはclean／warm start、inactive／background／foreground、process kill recovery、surface loss／rotation／resize／fold、safe-area change、touch／controller／IME／audio interruption、offline delivery interruption／resume／hash mismatch、memory pressure、GPU allocation failure、thermal soak、battery saver、Target fallback、structured-data rollbackを含む。

Minimum／Reference実機は同一commit、package、input traceでlifecycle、Save、Input、Audio、graphics golden、memory、thermal、deliveryを測る。Emulator／Simulatorはfunctional smoke専用で、GPU、audio／touch latency、memory、thermalの合否に使わない。Evidence envelope、run grading、provenanceは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)だけが所有する。
