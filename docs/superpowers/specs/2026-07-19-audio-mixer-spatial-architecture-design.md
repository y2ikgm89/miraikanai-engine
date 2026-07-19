# Miraikanai Engine Audio／Mixer／Spatial規約

- 文書版: 1.0
- 作成日: 2026-07-19
- 対象: Audio Asset、Voice、Mixer、Bus、DSP、Streaming、Spatial、XAudio2／Oboe／AudioUnit Adapter
- 状態: プロジェクト公式の規範設計レビュー版
- Runtime規約: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- Asset規約: [Miraikanai Engine Asset Pipeline／Content Package規約](./2026-07-19-asset-pipeline-content-packaging-design.md)
- Windows規約: [Miraikanai Engine Windows Platform／Distribution規約](./2026-07-19-windows-platform-distribution-design.md)
- Mobile規約: [Miraikanai Engine モバイルPlatformアーキテクチャ規約](./2026-07-19-mobile-platform-architecture-design.md)

## 1. 結論

Miraikanai EngineはAudioの論理Voice、Bus、Sound Cue、Spatial、priority、streamingを独自に所有する。XAudio2、Oboe、AVAudioSession／AudioUnitはdevice outputとnative lifecycleのAdapterであり、Game、AI、Saveへnative voice／callbackを公開しない。

Audioはpresentation stateである。Gameplayは「音が終了したcallback」やactual playback cursorをauthoritative判定に使わず、必要なtimingはGameplay clock／typed eventで別管理する。

C1は48 kHz float32 Engine mix、stereo output、one-shot PCM、Opus music stream、Bus gain、limiter、2D pan、3D distance／cone／doppler、device interruption／route recoveryを提供する。

## 2. 決定権と対象外

| 主題 | 正本 |
|---|---|
| Audio Asset意味、Voice、Cue、Mixer、Bus、DSP、Spatial、Streaming | 本書 |
| `T90` command、Audio control／callback、queue、memory、budget | Runtime規約 |
| Source import、PCM／Opus cook、Asset generation | Asset規約 |
| Android／Apple device、route、interruption | Mobile規約 |

C1ではvoice chat、microphone capture、speech recognition、MIDI、video sync、third-party DSP plugin、Runtime arbitrary audio graph code、HRTF、convolution reverb、dynamic music authoringを実装しない。HRTF、advanced reverb、voice chatはC2以降の別Capabilityである。

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

### 4.2 Stream

C1 stream codecはlibopus 1.6.1を使う。Cook時に48 kHz、20 ms packetを基本単位とし、`.mirapack`内のseek tableとchunk indexを生成する。

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

Command header／queueはRuntime規約の8192 entry、1 MiB payload、critical reserveを使う。`StopVoice`、`StopOwner`、`StopAll`、device recoveryをcriticalとし、低priority `PlayCue`だけをdropできる。

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

Deviceが48 kHzを受理しない場合、44.1／48／96 kHzのdevice rateへEngine-owned fixed-coefficient polyphase resamplerで最終変換する。未知rateやchannel layoutをnearestへ黙って変えず、Adapter capability不合格とする。Resampler coefficient、latency、qualityはgolden fixtureで固定する。

### 8.2 C1 Bus graph

```text
Music -----\
SFX --------\
UI ----------> Master -> Limiter -> Output
Dialogue ---/
Ambience ---/
```

Bus IDはStable numeric ID、parentは一つ、cycle禁止、最大64 Bus、depth最大8とする。既定BusはMaster、Music、SFX、UI、Dialogue、Ambience。User volumeはBus gainへ適用し、Game Project revisionを変更しない。

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

3D panとdistance attenuationはEngine Mixerが計算する。Dopplerはspeed of sound 343 m/sのReference値をProfileへ保存し、relative velocityがnon-finite／supersonic hard範囲外ならeffectをclampせずcommandをrejectする。

### 9.2 Occlusion

AudioはPhysics Queryをcallback中に行わない。Gameplay／Audio presentation systemが低rateで作った`AudioOcclusionSnapshot`をT90で渡し、Audio control threadがgain／low-passへsmooth適用する。Occlusion結果をGameplay visibilityやdamageへ戻さない。

## 10. Decode、Mix-ahead、Callback

- Audio control threadはdevice block deadlineより先にdecode／mix jobを計画する。
- DecodeはEngine Worker poolへowned compressed chunkとoutput bufferを渡す。
- ResultはAsset version、Voice generation、deadlineを検査してAudio control threadが統合する。
- Callback用PCM ringはpreallocated、既定3 block、minimum 2 blockである。
- Androidはnative frames-per-burstの整数倍へblockを選び、低遅延modeとcallbackを使う。
- XAudio2はpre-mixed bufferを一つのEngine output sourceへsubmitし、callbackはbuffer completionだけをqueueへcopyする。
- AppleはAVAudioSession route／sample rate確定後にAudioUnitを構成し、render callbackがringをconsumeする。

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

Windows Audio Domain 128 MiB内訳はRuntime規約どおりdecoded／stream ring 96 MiB、voice／control 16 MiB、reserve 16 MiBとする。

- Callback P99 0.25 ms以下、hard 1.00 ms未満
- Callback allocation、blocking lock、I/O、logは0
- Mix blockはdeadlineの50%以下をsoft target
- Control command latency P95は2 audio block以内
- 10分soakでunderrun 0
- Stream seek／loop P95はring deadline内

Mobileはaggregate memory、thermal、device burstをMobile規約で検証し、Windows Voice数を暗黙採用しない。

## 14. AI／Editor操作

AIと人間へ公開するのはAudio Clip Import、Sound Cue、Bus、Mixer Snapshot、Spatial Profile、semantic role、budgetである。

AIは音声buffer、XAudio2 voice、Oboe stream、AudioUnit、DSP pointerを生成／操作しない。Cueを大量生成するChangeSetは同時Voice予測、stream memory、loudness、loop、missing locale、copyright／licenseを検証する。

Audio Editorはwaveform、loop、loudness、Cue tree、Bus graph、Voice／stream profiler、spatial preview、underrun／route timelineを持つ。Previewも専用Preview Voice capacityを使い、Play sessionのBus／Voiceを直接変更しない。

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
| Callback generation stale | event破棄 |

## 16. TestとDefinition of Done

- PCM／Opus import、loop、seek、corrupt packet、stale Asset generation
- Cue random／sequence／switch、seed、node／depth cap
- Voice allocate／virtualize／steal／generation／critical reserve
- Bus cycle、Snapshot transition、DSP finite、limiter
- 2D pan、3D distance／cone／doppler／occlusion snapshot
- 44.1／48／96 kHz device conversion golden
- XAudio2 completion、Oboe callback、AudioUnit route／interruption
- callback allocation／lock／I/O 0、P99、underrun 0
- device disconnect、Bluetooth route、background／foreground、GameHost stop
- localization fallback、subtitle timing、User volume／mono
- 10分soak、128 MiB／Mobile budget、8192 command overflow

C1完了条件は、2D／3D縦切りでMusic、SFX、UI、Dialogue、3D spatial音をWindows／Android／Appleへ同じCue／Bus contractで再生し、route、stream、memory、callback hard gateを満たすことである。

## 17. 一次資料

- [XAudio2 Introduction](https://learn.microsoft.com/en-us/windows/win32/xaudio2/xaudio2-introduction)
- [XAudio2 Callbacks](https://learn.microsoft.com/en-us/windows/win32/xaudio2/xaudio2-callbacks)
- [XAudio2 Audio Graph](https://learn.microsoft.com/en-us/windows/win32/xaudio2/xaudio2-audio-graph)
- [Android Oboe Low-latency Audio](https://developer.android.com/games/sdk/oboe/low-latency-audio)
- [Apple AVAudioSession](https://developer.apple.com/documentation/avfaudio/avaudiosession)
- [Apple Audio Route Changes](https://developer.apple.com/documentation/avfaudio/responding-to-audio-route-changes)
- [Opus 1.6 API](https://opus-codec.org/docs/opus_api-1.6.pdf)

Platform APIはdevice／callbackの制約を決める。Cue、Voice、Bus、DSP Catalog、Spatial、failureはMiraikanaiが所有する。
