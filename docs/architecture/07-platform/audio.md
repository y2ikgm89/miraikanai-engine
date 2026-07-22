# Miraikanai Engine Audio／Mixer／Spatial Contract

- 文書ID: mirakan.arch.platform-audio
- 状態: review
- 正本範囲: Audio Assetのdomain意味、Sound Cue、Audio command、Voice、Mixer／Bus／DSP、Spatial、decode／stream／callback、route／interruption、Audio固有capacity／failure／qualification
- 非正本範囲: Source import／cook／promotion transaction、Runtime phase／shared queue・memory budget、Platform lifecycle、external codec／SDK version・hash・license・URL、AI authorization／Evidence envelope。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[Windows](windows.md)、[Mobile Common](mobile-common.md)、[Android](android.md)、[Apple](apple.md)、[UI／Text](ui-text-localization-accessibility.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論

Miraikanai EngineはAudioの論理Voice、Bus、Sound Cue、Spatial、priority、streamingを独自に所有する。XAudio2、Oboe、AVAudioSession／AudioUnitはdevice outputとnative lifecycleのAdapterであり、Game、AI、Saveへnative voice／callbackを公開しない。

Audioはpresentation stateである。Gameplayは「音が終了したcallback」やactual playback cursorをauthoritative判定に使わず、必要なtimingはGameplay clock／typed eventで別管理する。

C1は48 kHz float32 Engine mix、stereo output、one-shot PCM、Opus music stream、Bus gain、limiter、2D pan、3D distance／cone／doppler、device interruption／route recoveryを提供する。

## 2. 決定権と対象外

| 主題 | 正本 |
|---|---|
| Audio Asset意味、Voice、Cue、Mixer、Bus、DSP、Spatial、Streaming | 本書 |
| Runtime slot、Audio control／callback scheduling、shared queue／memory budget | Runtime owners |
| Source import、PCM／Opus cook、Asset generation | Asset規約 |
| Android／Apple device、route、interruption | Mobile規約 |

C1ではvoice chat、microphone capture、speech recognition、MIDI、video sync、third-party DSP plugin、Runtime arbitrary audio graph code、HRTF、convolution reverb、dynamic music authoringを実装しない。HRTF、advanced reverb、voice chatはC2以降の別Capabilityである。

### 2.1 独自実装と公式依存の境界

MiraikanaiはWwise、FMOD、SoLoud等のAudio middlewareをRuntime、Editor、Cookerの正本に採用しない。Audio Asset schema、Sound Cue、logical Voice、priority／virtualization、Bus／Snapshot、DSP Catalog、Spatial、decode scheduling、stream ring、resampler、Mixer、lifecycle、diagnostic、AI／Editor OperationはEngine-owned実装とする。

OS device出力と標準codecは、独自再実装による互換性・安全性・実機差リスクを避けるため、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)が固定するAPI／libraryをprivate Adapter内だけで使用する。

| 依存 | 許可する用途 | Engineが保持する正本 |
|---|---|---|
| XAudio2 | Windows output voice、device、engine／buffer completion callback | PCM ring、logical Voice、Mixer、route policy、callback queue |
| Oboe | Android low-latency output stream、device burst／xrun取得 | Mixer、buffer policy、fallback、route／interruption、telemetry |
| AVAudioSession／AudioUnit | session category／route／interruption、render callback、device format | Mixer、logical clock、PCM ring、recovery、background policy |
| libopus | Cook時encode、検証済みpacketのRuntime decode | stream manifest、packet／chunk index、pre-roll、loop、memory |
| libFLAC | Asset Import Worker内のnative FLAC decode／metadata／integrity確認 | Source validation、channel policy、conversion、Cooked artifact |

libFLACはSource Importにだけlinkし、GameHostへlinkしない。RuntimeはSource WAV／FLAC／Ogg containerをparseせず、Engine validatorとCookerが生成したPCMまたはOpus packet manifestだけを読む。Vendor型、pointer、callback、error codeをMCD、Cooked format、Game API、AI Operationへ公開しない。

外部Dependencyの事実はTarget実機Qualificationを省略する根拠にしない。Dependency更新はToolchain ownerの更新Gate、Adapter conformance、serialized fixture、performance再検証を通す。

### 2.2 `AudioRuntimeProfileV1`

```text
AudioRuntimeProfileV1
  profile_id
  schema_version: 1
  target_profile_ref
  audio_required: bool
  mixer_graph_ref
  output_policy_ref
  fallback_policy_ref
```

Project Sourceの`TargetProfileDocument`がTargetごとにexact一件を所有し、Cooked packageへhash付きで投影する。Play prepareは対象TargetのProfile欠落、重複、Target不一致、unknown Fieldを拒否し、`audio_required`の既定値を推測しない。`audio_required=true`ではdevice／format／routeを準備できなければPlayを拒否または既に開始済みのsessionをfaultにし、`false`ではtyped diagnostic付きsilent deviceを許可する。いずれもAudio completionをGameplay authorityへ昇格させない。

## 3. Architecture

```text
Gameplay／UI／Animation Event
  -> T90 AudioCommand
  -> bounded Audio Command Queue
  -> Audio Control Thread
       -> Voice／Bus／Cue state
       -> Decode／stream request
       -> Engine Mixer
       -> pre-mixed PCM block ring
  -> XAudio2 | Oboe | AudioUnit Adapter
  -> Device
```

Audio control threadがVoice create／destroy、Bus graph、Cue selection、decode scheduling、mix-aheadを所有する。Platform callbackはpreallocated PCM blockをconsumeし、completion／underrunをbounded queueへcopyするだけとする。Callback内でallocation、file I/O、log、mutex待機、device rebuild、Gameplay callbackを行わない。

## 4. Audio Asset

### 4.1 `AudioClipAsset`

| Field | 規則 |
|---|---|
| `asset_id`／revision | Stable Asset |
| `semantic_role` | SFX、UI、dialogue、ambience等のregistered tag |
| `channels` | monoまたはstereo。C1 |
| `source_sample_rate` | Import metadata |
| `cooked_sample_rate` | exactly 48,000 Hz |
| `sample_format` | PCM16 Cooked／mix時float32 |
| `frame_count` | `uint64` |
| `loop_start/end_frame` | optional、0≦start＜end≦frame_count |
| `integrated_loudness_lufs` | measured finite |
| `true_peak_dbfs` | measured finite |
| `streaming_policy` | `resident \| streamed` |
| `decoded_bytes` | budget validation |
| `channel_semantics` | mono、stereo L/R |

10秒以下かつdecoded size 2 MiB以下をresident候補、それ以外をstreamed候補とする。これは自動確定でなくImport previewであり、UI music、loop、latency要件でProfileが明示変更できる。

### 4.2 Source Import

Audio Source ImportはAsset Import／AI Authoring／Editor UX規約の`AudioImportSettingsV1`、`AudioImportIRV1`、Conversion Report、Preview、Reimport Conflictを使用する。

- C1 SourceはWAV PCM／IEEE floatとnative FLACである。WAV compressed codec、unknown channel mask、FLAC CRC不合格を拒否する。libFLACはOgg supportを無効にしたstream decoderだけをImport Workerへlinkし、`STREAMINFO`を必須、non-zero MD5をfull decode時に検証、error callback、frame CRC mismatch、lost sync、trailing undecoded payloadをImport失敗にする。
- RIFF／RF64 bounds、WAVEFORMATEXTENSIBLE valid bits／channel mask／block alignment、FLAC STREAMINFO／frame／CRCを検証する。
- Sourceを48 kHzへCookしても`source_sample_rate`、original frame count、resampler version、変換前後durationをReceiptへ保持する。
- Integrated loudnessとtrue peakは測定metadataであり、明示`gain_policy`なしに自動normalizationをsampleへ焼かない。
- Loopはsample frameで表し、Source marker採用、手動range、codec delay／pre-skip補正をConversion Reportへ記録する。
- Source channelを名前やsemantic roleだけでdownmixせず、typed channel policyとPreview auditionを必須とする。

### 4.3 Stream

C1 stream codecはToolchain lockのlibopusを使う。Cook時に48 kHz、20 ms packetを基本単位とし、`.mirakanpack`内のseek tableとchunk indexを生成する。

`AudioStreamManifest`はduration、channels、pre-roll、packet／chunk offset、granule／frame mapping、loop、payload hashを持つ。RuntimeはOpus file parserやuntrusted Sourceを読まず、検証済みCooked packetだけをdecodeする。

Stream ringは最低250 ms、既定500 ms、最大2秒分をAudio Assetごとに予約する。Music loopはpre-roll／codec delayをCook時に補正し、seam fixtureを合格する。

## 5. Sound Cue

`SoundCueDefinition`はAudio Clipを直接一つ指定する場合と、次のbounded nodeを組み合わせる場合を持つ。

- clip
- random choice
- sequence
- switch by registered semantic parameter
- gain／pitch range
- delay
- loop
- bus send

Node最大64、depth最大8、choice最大32とする。任意loop、Script、callback、Asset path文字列を禁止する。

Random choiceはCue instance seedとdraw countを使い、同じReplay入力でselectionを再現できる。Audio output自体はauthoritative state hashへ含めない。

## 6. Audio command

| Command | 主なField |
|---|---|
| `PlayCue` | Cue Asset version、emitter、bus、priority、gain、pitch、owner generation |
| `StopVoice` | VoiceHandle、fade |
| `StopOwner` | owner generation、fade |
| `PauseVoice`／`ResumeVoice` | VoiceHandle |
| `SetVoiceParameter` | registered parameter、ramp |
| `SetBusGain` | Bus ID、gain、ramp |
| `SetListener` | Listener transform／velocity |
| `SetEmitter` | Emitter transform／velocity／spatial profile |
| `SetSnapshot` | registered Mixer Snapshot、transition |
| `StopAll` | critical command |

Command header／queueは[Runtime performance／capacity](../04-runtime/performance-capacity.md)のAudio command reservationとcritical reserveを使う。`StopVoice`、`StopOwner`、`StopAll`、device recoveryをcriticalとし、低priority `PlayCue`だけをdropできる。

Commandはnative source voice、PCM pointer、file path、function callbackを含めない。

## 7. Voice

### 7.1 State

```text
Free -> Allocated -> Priming -> Playing <-> Paused
-> Stopping -> Finished -> Retiring -> Free
```

`VoiceHandle { index32, generation32 }`はAudio control threadだけが解決する。Completion eventはhandle generationを検査し、reused voiceへ適用しない。

### 7.2 Capacity

| Target | logical active | audible mix | streamed |
|---|---:|---:|---:|
| Windows C1 | 256 | 128 | 16 |
| Mobile Baseline | 128 | 64 | 8 |
| Mobile Standard／High | 192 | 96 | 12 |

CapacityはPlay prepare時にmemoryへchargeし、Play中に増やさない。Audible上限超過時はpriority、estimated loudness、distance、age、Stable Voice keyでcanonical virtualizationする。Virtual Voiceはtime／loop／event cursorを進めるがmixしない。

Voice slot不足時はcritical UI／dialogue reserved 16 slotを維持し、低priority Playを`VoiceCapacity`としてdropする。既存高priority Voiceをrandomに奪わない。

## 8. MixerとBus

### 8.1 Internal format

- internal sample rate: 48,000 Hz
- sample: float32
- internal channel: mono sourceまたはstereo bus
- block: 256 frame reference
- accumulation: float32、denormal flush、finite check
- final: soft clipではなくMaster limiter後に`[-1,1]`

Deviceが48 kHzを受理しない場合、44.1／48／96 kHzのdevice rateへEngine-owned fixed-coefficient polyphase resamplerで最終変換する。既知device channel layoutはmono、stereo、5.1、7.1に固定する。C1のEngine final outputはstereoで、mono deviceへはequal-power downmix、5.1／7.1 deviceへはfront L／R channelへのstereo submitを行う。これ以外のdevice rate（88.2／176.4／192 kHz等）とchannel layoutはnearestへ黙って変えず、Adapter capability不合格とし、挙動は§15の「Device format非対応」に従う。Resampler coefficient、latency、qualityはgolden fixtureで固定する。

### 8.2 C1 Bus graph

```text
Music -----\
SFX --------\
UI ----------> Master -> Limiter -> Output
Dialogue ---/
Ambience ---/
```

Authoring SourceのBus identityはUUIDv7 `StableId`とする。Cookerは一つのexact Audio Graph Artifact内だけで有効な`AudioBusId uint16`へStableId byte順で0～63を割り当てる。0もvalidであり、Masterは数値でなくGraphの`master_bus_ref`で厳密に一つ指定する。`AudioBusId`とGraph `ArtifactRefV1`を組にして扱い、別Graphの同じ数値を比較またはSaveせず、Save／User settingはBus `StableId`を保持する。parentは一つ、cycle禁止、最大64 Bus、depth最大8とする。既定BusはMaster、Music、SFX、UI、Dialogue、Ambience。User volumeはBus gainへ適用し、Game Project revisionを変更しない。

Bus graphの構造変更はPlay開始前またはAudio control block boundaryで互換なSnapshot swapとして行う。一部Nodeだけをhot replaceしない。

### 8.3 C1 DSP

- gain／mute
- linear／equal-power fade
- peak meter
- Master look-ahead limiter
- first-order low-pass／high-pass
- delay line
- reverb send／return用のbounded algorithmic reverb

DSP nodeはEngine Catalogのclosed ID、parameter range、state bytes、CPU costを持つ。Project HLSL／C++／plugin DLLによる任意DSPを禁止する。Graph最大128 node、Voice insert最大4、Bus insert最大8。

## 9. Spatial Audio

### 9.1 ListenerとEmitter

C1 Listenerは一つ。Split-screen／multi-listenerはC2。Transformはright-handed meter、velocity m/s、finiteである。

`SpatialAudioProfile`は次を持つ。

| Field | 規則 |
|---|---|
| `mode` | `non_spatial \| spatial_2d \| spatial_3d` |
| `min/max_distance_m` | `0 < min <= max <= 10000` |
| `rolloff` | linear／inverse／registered curve |
| `inner/outer_cone_rad` | `0..2π` |
| `outer_gain` | `[0,1]` |
| `doppler_factor` | `[0,4]` |
| `stereo_spread_m` | `[0,100]` |
| `occlusion_policy` | off／snapshot |

3D panとdistance attenuationはEngine Mixerが計算する。Dopplerはspeed of sound 343 m/sのReference値をProfileへ保存する。relative velocityがnon-finiteの`SetEmitter`／`SetListener`はcommandをrejectし、Voiceは前回のtransform／velocityを維持する（§15）。supersonic相対速度はShooter規約が許容する正当な入力であり、rejectせずdoppler pitch比率を`[0.5, 2.0]`へ決定的にclampして適用する。

### 9.2 Occlusion

AudioはPhysics Queryをcallback中に行わない。Gameplay／Audio presentation systemが低rateで作った`AudioOcclusionSnapshot`をT90で渡し、Audio control threadがgain／low-passへsmooth適用する。Occlusion結果をGameplay visibilityやdamageへ戻さない。

## 10. Decode、Mix-ahead、Callback

- Audio control threadはdevice block deadlineより先にdecode／mix jobを計画する。
- DecodeはEngine Worker poolへowned compressed chunkとoutput bufferを渡す。
- ResultはAsset version、Voice generation、deadlineを検査してAudio control threadが統合する。
- Callback用PCM ringはpreallocated、既定3 block、minimum 2 blockである。
- AndroidはOboe `PerformanceMode::LowLatency`、`SharingMode::Exclusive`、`Usage::Game`、data callbackを要求し、sample rateを強制せずopen後のnative rateを取得する。48 kHz以外はEngine final resamplerを使う。既定usable bufferは2 burst、PCM ring blockはnative frames-per-burstの整数倍とし、Shared fallback、実buffer、xrunをCapability Signatureへ記録する。
- XAudio2はpre-mixed bufferを一つのEngine output sourceへsubmitし、callbackはbuffer completionだけをqueueへcopyする。
- AppleはAVAudioSession category／route／sample rate確定後にRemoteIO／AudioUnitを構成し、nonblocking render callbackがringをconsumeする。route change、interruption、media service resetはtyped eventへcopyし、Audio control threadで再構成する。

Underrun時は未初期化memoryやlast blockをrepeatせずzero-fillし、counterを上げる。1回のunderrunはpresentation degradation、1秒に3回または10秒に10回でAudio subsystem faultとしてroute rebuildを一度試し、再発時はaudioを停止してGameを継続するか、Projectの`audio_required`ならsession faultにする。

## 11. Route、Interruption、Lifecycle

```text
Stopped -> Opening -> Running -> Interrupted
-> Reconfiguring -> Priming -> Running
```

- Device／route changeはcallback外のAudio control threadで処理する。
- Interruption開始時にVoice logical timeをpauseし、stream decodeをboundedに停止する。
- 再開時は新format／burst／latencyを検証し、ringをpriming後に開始する。
- 古いcallback generationのeventを破棄する。
- Bluetooth等でlatency／sample formatが変わった場合はCapability Signatureを更新する。
- Background policyはMobile規約に従い、許可のないbackground audioを継続しない。

## 12. Dialogue、Localization、Accessibility

- Dialogue CueはLocalized Asset Setを参照し、locale fallback chainをPackage時に解決する。
- Missing required voiceで別言語音声へ黙ってfallbackせず、Project Policyどおりsubtitle-onlyまたはScene load errorにする。
- Subtitle eventはAudio playback callbackでなく、Dialogue／Timelineのauthoritative presentation scheduleからUIへ送る。
- Music、SFX、Dialogue、UIの個別volume、mute、dynamic range profileをUser settingにする。
- mono output optionとcaptions／visual cue向けtyped presentation eventを提供する。

## 13. MemoryとPerformance

Audio allocationは[Runtime performance／capacity](../04-runtime/performance-capacity.md)のAudio child scopeへchargeし、本書はshared parent totalやpartitionを複写しない。Voice／stream profile別capacityとcallback固有deadlineは本書が所有する。

- Callback deadlineは[Runtime performance／capacityの共通Audio operation budget](../04-runtime/performance-capacity.md#7-framelatencysubsystem-budget)をexactに消費し、本書で値を再定義しない
- Callback allocation、blocking lock、I/O、logは0
- Mix blockはdeadlineの50%以下をsoft target
- Control command latency P95は2 audio block以内
- 10分soakでunderrun 0
- Stream seek／loop P95はring deadline内

Mobileはaggregate memory、thermal、device burstをMobile規約で検証し、Windows Voice数を暗黙採用しない。

## 14. AI／Editor操作

AIと人間へ公開するのはAudio Clip Import、Sound Cue、Bus、Mixer Snapshot、Spatial Profile、semantic role、budgetである。

AIは音声buffer、XAudio2 voice、Oboe stream、AudioUnit、DSP pointerを生成／操作しない。Cueを大量生成するChangeSetは同時Voice予測、stream memory、loudness、loop、missing locale、copyright／licenseを検証する。

Audio Editorはwaveform、sample-accurate loop、trim、loudness、true peak、channel、resident／stream cost、Target codec A／B audition、Cue tree、Bus graph、Voice／stream profiler、spatial preview、underrun／route timelineを持つ。Previewも正式Import Planと同じdecode／resample／encode code、専用Preview Voice capacityを使い、Play sessionのBus／Voiceを直接変更しない。

AIのAudio Import操作は`asset.inspect_source`、`asset.propose_import_profile`、`asset.request_preview`、`asset.propose_import_settings_change`、`asset.propose_reimport`へ限定する。Ambiguous channel、loop、gain、dialogue localeは質問を返し、file名から確定しない。

Audio Authoring契約は`schemas/mirakan/audio/`のMCDを正本とし、C++、TypeScript、MCP Operation schema、validator、reference docsを生成する。最低限`AudioRuntimeProfileV1`、`AudioClipAssetV1`、`AudioImportSettingsV1`、`SoundCueDefinitionV1`、`MixerGraphV1`、`MixerSnapshotV1`、`SpatialAudioProfileV1`、`AudioCommandV1`、`AudioDiagnosticV1`をversioned schemaにする。

semantic role、Cue parameter、Bus、DSP node、Snapshot、Spatial curve、diagnostic codeは`AudioSemanticCatalogV1`のStable IDへ登録する。表示名、file path、Editor selection、native handleを参照IDにしない。AIとEditorは同じCatalog revisionとvalidatorを使用し、unknown ID、range外、cycle、capacity超過、Target非対応をCommit前に拒否する。

Import以外のAI操作は`audio.inspect_cue`、`audio.propose_cue`、`audio.validate_cue`、`audio.request_preview`、`audio.inspect_mixer`、`audio.propose_mixer_change`、`audio.explain_budget`、`audio.diff_revision`に限定する。すべてbase revision、target Stable ID、提案patch、predicted budget、diagnostic、preview／validation receiptを返し、直接Commit、直接Play session変更、native object操作を行わない。

各schemaとOperationにはvalid／invalid golden fixtureを用意する。最低限、footstep random Cue、UI critical Cue、streaming music loop、dialogue locale set、Bus cycle、Cue depth超過、unknown parameter、invalid loop、Voice capacity超過、stale revisionを含める。

Runtime AIによる音声生成／downloadはC1／C2 Shippingで禁止する。外部AI生成AudioはAsset規約のStaging、権利、来歴、安全、Importを通す。

## 15. Failure policy

| Failure | 結果 |
|---|---|
| Missing／corrupt required Audio Asset | Scene／Play prepare失敗 |
| Optional Cue不足 | 明示fallbackまたはsilence＋diagnostic |
| Voice／queue capacity | low priority Playをcanonical drop、critical reserve維持 |
| Decode deadline | Voiceをzero／virtualize、counter。stale result破棄 |
| Callback underrun | zero-fill、rebuild policy |
| Route／interruption | logical Voice pause、callback外reconfigure |
| DSP non-finite | blockを出力せずAudio fault、NaNをdeviceへ送らない |
| Audio deviceなし | `audio_required=false`ならsilent device、trueならPlay拒否 |
| Device format非対応（rate／channel layout capability不合格） | `audio_required=false`ならsilent device＋typed diagnostic、trueならPlay拒否。nearest formatへ黙って変えない |
| `SetEmitter`／`SetListener` non-finite | command reject、前回transform／velocity維持、typed diagnostic |
| Callback generation stale | event破棄 |

## 16. Activation境界

Capability maturityと実装順序は[Product Plan](../00-product/product-plan.md)だけが決定する。本書のContract、Headless Mixer、Voice／Cue／Spatial、Import／Cook／Stream、Windows／Android／Apple output Adapter、AI／Editor、cross-platform qualificationは独立fixtureとして検証し、未完成Adapterはsilent successを返さず`UnsupportedTarget`とする。device-independent Mixer／Cue／Cook fixtureはPlatform AdapterなしでCI実行でき、Adapter都合をAudio contractへ逆流させない。

## 17. TestとDefinition of Done

- WAV PCM8／16／24／32、float32、RF64、FLAC、Opus Cook、loop、seek、corrupt packet、stale Asset generation
- WAVEFORMATEXTENSIBLE channel mask、FLAC CRC、Opus pre-skip／output gain／granule mapping
- IETF CELLAR FLAC decoder testbench、RFC 8251 Opus test vector、corrupt／truncated／oversized metadata fuzz corpus
- loudness／true peak golden、44.1／48／96 kHz resample、Target codec A／B PreviewとCook一致
- Cue random／sequence／switch、seed、node／depth cap
- Voice allocate／virtualize／steal／generation／critical reserve
- Bus cycle、Snapshot transition、DSP finite、limiter
- 2D pan、3D distance／cone／doppler／occlusion snapshot
- 44.1／48／96 kHz device conversion golden
- XAudio2 completion、Oboe callback、AudioUnit route／interruption
- callback allocation／lock／I/O 0、P99、underrun 0
- device disconnect、Bluetooth route、background／foreground、GameHost stop
- localization fallback、subtitle timing、User volume／mono
- 10分soak、Runtime ownerのAudio scope、shared command reservation overflow

C1完了条件は、2D／3D縦切りでMusic、SFX、UI、Dialogue、3D spatial音をWindows／Android／Appleへ同じCue／Bus contractで再生し、route、stream、memory、callback hard gateを満たすことである。

## 18. 外部依存境界

Platform audio API、codec library、test corpusのexact release、commit、hash、license、取得元、一次根拠は[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)だけが所有する。Cue、Voice、Bus、DSP Catalog、Spatial、callback制約、failureはMiraikanaiの本書が所有する。
