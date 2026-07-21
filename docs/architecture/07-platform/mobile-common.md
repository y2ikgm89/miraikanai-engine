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

`profile_revision`不一致、unknown Capability、禁止permission、Target別aggregate cap超過、minimum deviceとCooked artifact要求の不整合はCook前に拒否する。Android／Appleが共通して生成し、ResolverとQualificationが消費する型は次の`MobileCapabilitySignatureV1`だけである。

```text
MobileCapabilitySignatureV1
  schema_version
  target_profile_ref
  toolchain_profile_ref
  device_identity
  os_capabilities
  cpu_abi
  gpu_capabilities
  memory_class
  display_capabilities
  input_capabilities
  audio_capabilities
  thermal_capabilities
```

各値はEngine-owned enum／valueへ正規化し、vendor objectやdisplay名を永続化しない。Android／Apple ownerは観測値の写像だけを所有し、このfield setを再定義しない。旧`CapabilitySignature`、`PlatformCapabilitySignature`、別綴りのalias、union受理は行わない。

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

`StorePolicyLock`はMobile Store要件の唯一の物理schemaである。Android／Appleはこの型への参照とPlatform固有requirement値だけを所有し、`store_policy.lock`その他の別schemaを定義または受理しない。

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

Platform Adapter破棄順は、callback停止、新規submission停止、queue drain／timeout、resource retire、device／surface解放の順で固定する。順序を入れ替えず、timeout後も未完了submissionが参照するresourceを先に解放しない。停止後に届いたcallbackと古い`SurfaceGeneration`のworkはcommitせず、diagnosticへgeneration、submission serial、timeout reasonを記録する。

## 5. Renderer境界とAsset delivery

Mobileは[Render Graph](../06-rendering/render-graph.md)の`AntiAliasingIntentV1`、`ResolvedAntiAliasingPlanV1`、`AntiAliasingResolutionErrorV1`、`TemporalFrameInputV1`、`ResolvedRendererProfileV1`、`GpuSubmissionSerial`、Provider排他、history reset、real／displayed frame分離を変更しない。本書はTarget Profile／`MobileCapabilitySignatureV1`／memory・thermal signalと、以下のMobile選択policyをResolverへ渡す。Render Graphはexecution、backend barrier、clip／depth、resource lifetime、AA／upscale／frame-generation algorithmを所有する。

Mobile baselineへWindows-only backend、ray／neural technique、unqualified temporal Providerを要求しない。`VirtualShadowBackendV1`、`RayTracedShadowBackendV1`、`ProjectShadowTechniqueV1`はMobile Capability Catalogへ暗黙登録せず、[Render Graph](../06-rendering/render-graph.md)のactivation／resolver contractへ委譲する。unsupported combinationは`UnsupportedByTarget`としてfail closedし、explicit fallbackとomitted reasonを返す。UI／text／pixel-locked layerはWorld dynamic resolutionやtemporal processingの対象外である。Target固有Vulkan／Metal mappingとdevice fixtureは各Platform ownerが持つ。

### 5.1 Frames-in-flightとAA選択

Frames-in-flightは`mobile_baseline`を2、`mobile_standard`と`mobile_high`を3に固定する。Profile外の値を拒否し、全submission完了前にtransient arena、upload ring、binding rangeをresetしない。

- `balanced | low_gpu_cost`: Baseline既定はFXAAであり、実機Gateなしにtemporal methodを自動選択しない。
- `minimum_blur | minimum_ghosting | vr_low_latency`: qualification済みForward+ MSAA 2x／4xだけを候補にする。
- MSAA 8xは`mobile_high`またはoffline capture C2だけに許可し、AI自動選択から除外する。SMAA 1xはC2であり、Baseline必須にしない。
- `pixel_crisp`: pixel-locked layerへWorld AAを適用せず、最終解像度で合成する。
- unsupported combinationは`AntiAliasingResolutionErrorV1::UnsupportedByTarget`で失敗し、明示的fallbackを表示する。

### 5.2 Dynamic resolution

| Profile | 最大pixel数 | 基準解像度 |
|---|---:|---:|
| `mobile_baseline` | 921,600 | 1280×720 |
| `mobile_standard` | 2,073,600 | 1920×1080 |
| `mobile_high` | 3,686,400 | 2560×1440 |

Dynamic resolutionは5%刻み、下限50%とする。直近30 frame中12 frame以上がGPU soft targetを超えるか、Memory／Thermalが`Serious`以上なら直ちに一段下げる。GPUがtargetの80%未満かつMemory／Thermalが`Normal`の状態が15秒連続した場合だけ一段上げ、変更は最大毎秒一段とする。UI、text、pixel-locked layerへ適用しない。temporal method使用時はresolution step、orientation、surface generation、projection、jitter sequence、camera cutのいずれかが変化したframeでhistoryをresetし、reason codeを記録する。

### 5.3 Frame Generation

Frame Generationは`mobile_high`だけに許可し、Provider-offのreal frameがCPU／GPU P95とも16.67 ms以下、real 60 fps、deadline miss 1%以下、30分thermalと2時間enduranceを全て通過した場合だけ候補にする。touch-to-photonは1000 fps以上のhigh-speed cameraで`240 tap×5 run`を採取し、touch contact frameから指定flash regionの最初の輝度変化までを測る。各runのnearest-rank P95のmedianを判定値とし、Provider-off比の劣化8.33 ms以下かつ絶対値83.33 ms以下を要求する。Receiptがなければ失敗する。30 fps入力をdisplayed 60 fpsにした結果を60 fps capabilityと表示しない。pixel-locked 2D、fullscreen menu、loading、pause、camera cut、rotation／resize／surface regenerationでは無効化する。

### 5.4 Mobile graphics quality

Mobile graphics quality profileは`Baseline | Standard | High`のclosed setである。各値は次のMobile選択policyを一括して表し、Android／Apple Adapterはこの表を再定義しない。

| 機能 | Baseline | Standard | High |
|---|---|---|---|
| Renderer | Forward+ | Forward+ | Forward+／optional hybrid |
| Shadow technique | SDF 2D／CSM＋atlas | SDF 2D／cached CSM＋atlas | SDF 2D／cached CSM＋atlas＋選択的PCSS |
| Shadowed directional | 1、2 cascade、1024 atlas | 1、3 cascade、2048 atlas | 1、4 cascade、2048–4096 atlas |
| Visible local lights | 8、local shadowなし | 32、最大2 shadowed | 64、最大4 shadowed |
| Fog | height／distance fog | half-resolution volumetric optional | volumetric高品質 |
| Clouds | 2D layer | reduced volumetric optional | volumetric |
| Particle | CPUまたは限定GPU | GPU particle | GPU particle高budget |
| Reflection | probe | probe＋SSR optional | probe＋SSR |
| Anti-alias／Upscale | FXAA。Qualified MSAA 2xは`minimum_blur`／`minimum_ghosting`／`vr_low_latency`だけ | FXAA fallback。Qualified SMAA 1x、MSAA 2x／4x、Mirakan TAA／TAAU | Mirakan TAAU／Qualified FSR・MetalFX。MSAA 8xは個別Gateだけ |
| Frame Generation | Off | Off | Qualified `mobile_high`だけ |
| RT／Neural | Off | Off | 個別Experimental／Qualification後。Raster fallback必須 |

この表はvendor保証ではなくMiraikanai Engineの品質budgetである。実機測定が不合格ならCapabilityを偽装せず、`High → Standard → Baseline`の順に一段下げ、選択Profile、棄却理由、fallback、omitted reason、Qualification Receiptをdiagnosticへ残す。unknown profileは近い値へ変換せず拒否する。Baselineでも合格しないTargetは`OptimizationRequired`としてShippingを拒否する。2D C1はBaselineで全機能を成立させ、3D C1はscalable subsetを成立させる。Presentation縮退はresolution、shadow、VFX、volumetric、streaming concurrency等だけに適用し、敵味方数、damage、collision、goal、spawn timingその他のauthoritative resultを2D／3Dとも変更しない。

AI／Editorはmethod名を直接推測せず`AntiAliasingIntentV1`を入力し、Rendererの決定的Resolverから得た`ResolvedAntiAliasingPlanV1`、推定GPU／memory／bandwidth費用、fallback、omitted reason、Qualification Receiptを同じChangeSet previewへ表示する。Rendererは表のexecutionを所有し、Mobile Commonはclosed profile値と縮退policyを所有する。

### 5.5 Mobile texture Cook

[Asset lifecycle](../03-authoring/asset-lifecycle.md)のcanonical `DerivedArtifactManifest`とCook／promotion transactionを使用する。Mobile textureのTarget artifact projectionは次の7 fieldを必須とし、新しいgeneric Asset manifest aliasを作らない。

```text
width
height
mip_count
color_space
alpha_mode
target_format
content_hash
```

CookはAsset ID／revision、Source content、texture semantic、Target Profile、Toolchain lockからTarget formatを決定的に選択する。同一入力は同じ`target_format`と`content_hash`を生成し、package inspectionはwidth、height、mip count、color space、alpha mode、target format、content hashをCook済みpayloadと照合する。一項目でも不一致、Target format不足、同一Asset IDのTarget対応重複があればCook／promotionを拒否し、last-valid artifactを維持する。Pixel Art／UI／maskはedgeを壊すblock compressionを避け、RGBA8または用途別losslessを選ぶ。Android／Appleのformat mappingとpackage検証は各Platform ownerが所有する。

RuntimeでBasis／Universal TextureからTarget formatへtranscodeすることを禁止する。Basis／Universal Texture入力を使用する場合もoffline CookでTarget artifactへ確定し、Shipping packageへRuntime transcode path、transcoder、汎用intermediateだけのtextureを含めない。

### 5.6 Asset delivery

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

`DeviceDebugHandshakeV1`はEngine protocol、Build、Project revision、Target、device identity、requested recording tier、retention boundを照合する。認証済みlocal bridgeはUserが選択した一台、一Session、一時間だけ接続し、compatible GameplayDefinitionSet、structured data、already-cooked Assetだけをhot reloadする。二台目または二つ目のSessionを拒否し、一時間到達時は新規captureを停止する。C++、shader、native pluginはrebuild／reinstallする。切断または期限到達時は受信済みcomplete chunkだけをread-only確定し、missing rangeをgapとして残す。Shipping scanはdevice bridge、debug server、IDE attach、validation layer、raw trace、compiler、source path、credentialを拒否する。

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
| texture target選択／7-field manifest／payload不一致 | Cook／promotion拒否、last-valid artifact維持 |
| unknown graphics quality／Baseline不合格 | fallback候補とdiagnosticを表示し、`OptimizationRequired`としてShipping拒否 |
| memory Exhausted／thermal Critical | authoritative stateを維持して縮退、不能ならcheckpoint後safe exit |
| device bridge mismatch／gap | session拒否またはcomplete chunkだけ確定 |
| device bridgeの二台目／二Session目／一時間超過 | 接続拒否またはcapture停止、complete chunkをread-only確定しmissing rangeをgap化 |
| Platform Adapter破棄順違反 | qualification失敗、device／surfaceの早期解放を禁止し順序とtimeout reasonを診断 |
| executable content in delivered Asset | package／mount拒否 |
| Runtime AI validation／moderation failure | Project／Save不変、last-valid content維持 |

共通fixtureはclean／warm start、inactive／background／foreground、process kill recovery、surface loss／rotation／resize／fold、safe-area change、touch／controller／IME／audio interruption、offline delivery interruption／resume／hash mismatch、memory pressure、GPU allocation failure、thermal soak、battery saver、Target fallback、structured-data rollbackを含む。Renderer fixtureはProfile別Frames-in-flight、各AA intentの候補／fallback、30-frame thresholdと15秒回復を含む5% dynamic-resolution遷移、全history-reset reason、Frame Generationのreal／displayed frame分離、`240 tap×5 run`、30分thermal、2時間endurance、無効化場面を検証する。Mobile graphics-quality fixtureは11行×3 profileの全値、unknown拒否、`High → Standard → Baseline`の一段遷移、Baseline不合格、2D C1全機能、3D C1 scalable subset、Presentation縮退前後のauthoritative result一致を検証する。Texture fixtureは同一入力の反復Cookで`target_format`／`content_hash`一致、7 field各tamperの拒否、Target artifact不足／重複、Pixel Art／UI／maskのRGBA8またはlossless、Runtime Basis／Universal Texture transcode path不在を検証する。Device bridge fixtureは一台目／一Session目の59分59秒までを許可し、二台目、二Session目を拒否し、1時間到達でcaptureを停止してcomplete chunkとmissing gapを確定する。Adapter teardown fixtureはcallback中、in-flight submission、queue timeoutを注入し、callback停止からdevice／surface解放までの順序と診断を検証する。

Minimum／Reference実機は同一commit、package、input traceでlifecycle、Save、Input、Audio、graphics golden、memory、thermal、deliveryを測る。Emulator／Simulatorはfunctional smoke専用で、GPU、audio／touch latency、memory、thermalの合否に使わない。Evidence envelope、run grading、provenanceは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)だけが所有する。
