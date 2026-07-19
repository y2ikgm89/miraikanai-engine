# Miraikanai Engine Input／Action／Device規約

- 文書版: 1.0
- 作成日: 2026-07-19
- 対象: Keyboard、Mouse、Controller、Touch、Pointer、Action Mapping、Remap、Haptics、Replay
- 状態: プロジェクト公式の規範設計レビュー版
- Runtime規約: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- UI規約: [Miraikanai Engine UI／Text／Localization／Accessibility規約](./2026-07-19-ui-text-localization-accessibility-design.md)
- Windows規約: [Miraikanai Engine Windows Platform／Distribution規約](./2026-07-19-windows-platform-distribution-design.md)
- Mobile規約: [Miraikanai Engine モバイルPlatformアーキテクチャ規約](./2026-07-19-mobile-platform-architecture-design.md)
- Editor規約: [Miraikanai Engine Editor／Workspace／UX規約](./2026-07-19-editor-workspace-ux-design.md)

## 1. 結論

Miraikanai EngineはPlatformのkey code、controller object、touch callbackをGame ruleへ直接渡さない。Platform AdapterがDevice ReadingをEngine値へ正規化し、`InputActionMap`がsemantic Actionへ解決し、`T10_InputLatch`でtick番号付きimmutable `InputSnapshot`を確定する。

Gameplay、UI、AI、NativeGameModuleが参照するのはStable `InputActionId`とSnapshotだけである。Text入力／IMEはAction Inputと別の`ITextInputService`が所有し、keyboard stateから文字を推測しない。

## 2. 決定権と対象外

| 主題 | 正本 |
|---|---|
| Device正規化、Action、Binding、Snapshot、Remap、Haptics | 本書 |
| `T10`、Replay、queue、thread、Platform event source | Runtime規約 |
| Game UI focus／event、Text／IME | UI／Text規約 |
| Android／Apple lifecycle、controller、touch、haptics | Mobile規約 |

C1ではVR motion controller、eye tracking、MIDI、raw HID extension、rhythm-game専用sub-millisecond API、network remote input、anti-cheat input attestationを実装しない。これらは別Device Capabilityとする。

## 3. Device model

### 3.1 Device descriptor

```text
InputDeviceDescriptor
  runtime_device_id
  device_generation
  device_class
  vendor_product_fingerprint
  capability_bits
  player_slot
  connection_state
  battery_state
  display_name
```

`runtime_device_id`はsession内だけ有効で、SaveやProjectへ保存しない。Remap Profileは`DeviceMatchProfile { device_class, optional vendor/product, layout_family }`を使う。

Device classを次で固定する。

- keyboard
- mouse
- gamepad
- touch_surface
- pen
- virtual_accessibility

Unknown classはActionへ自動割当せず、Diagnosticsへ表示する。

### 3.2 Reading

`DeviceReading`はPlatform timestamp、Engine sequence、Device ID／generation、control values、connection flagsを持つ。Adapter callbackは値をpreallocated queueへcopyするだけで、Gameplay、UI tree、Worldを呼ばない。

Digital controlは`up | down`、analogはfinite float `[-1,1]`、triggerは`[0,1]`、pointer／touch positionはsurface pixelとnormalized safe-area座標を両方持つ。NaN、Inf、範囲外、unknown controlをrejectし、clampをDevice calibrationより後のAction processorだけで行う。

## 4. Action model

### 4.1 Action

| Field | 規則 |
|---|---|
| `action_id` | Stable numeric ID。表示名をdispatchに使わない |
| `value_type` | `digital \| axis1 \| axis2 \| pointer2` |
| `scope` | `gameplay \| ui \| editor \| system` |
| `consumption` | `shared \| focused \| exclusive` |
| `default_value` | 型ごとのzero |
| `interactions` | 最大4 |
| `processors` | 最大8、固定順 |
| `required` | Profileごとのbool |
| `replay_policy` | `authoritative \| presentation_only \| excluded` |

Pose、text、arbitrary byte payloadをAction valueへ入れない。

### 4.2 Binding

```text
InputBinding
  binding_id
  action_id
  device_match
  control_path
  modifiers[]
  interaction
  processors[]
  context_id
  priority
```

`control_path`はEngine Input Control Catalogのclosed IDであり、Platform key code文字列を正規dataにしない。Bindingは最大4096／Project、Actionは最大1024、同一ActionのBindingは最大32とする。

### 4.3 C1 interaction

- press
- release
- tap
- hold
- repeat
- chord
- toggle

Tap最大時間は既定0.25 s、Hold最小0.40 s、Repeat初回0.40 s／間隔0.08 sとし、Project Profileで範囲検証できる。Wall clockではなくlatchしたmonotonic input timeで評価し、Replayは結果Action transitionを記録する。

### 4.4 C1 processor

| Processor | 既定 |
|---|---|
| dead zone | stick radial 0.15～0.95、trigger 0.05～0.95 |
| normalize | Device range→Engine range |
| invert | false |
| scale | 1.0、`[-16,16]` |
| clamp | value type範囲 |
| curve | linear。C2で登録済みcurve |
| composite | WASD／D-pad→axis2 |

Processor順序は`normalize -> dead_zone -> invert -> scale -> curve -> clamp -> composite`の登録順で固定し、Bindingごとの任意function chainを許可しない。

## 5. Context、Focus、Consumption

公式Contextを次で固定する。

- `system`
- `editor_global`
- `editor_view`
- `gameplay`
- `game_ui`
- `text_entry`
- `debug`

Active Context stackは`T00`で更新し、tick中は不変とする。Priorityはsystem、text entry、focused UI、gameplay、debugの順で、Debug Shipping無効時は存在しない。

`exclusive` Actionがcontrolをconsumeした場合、低priority Contextへ同じphysical transitionを渡さない。Shared movement stickとUI navigationを同時に有効化する場合は明示`shared` Bindingを必要とする。Focus loss時は全down stateへcanonical releaseを生成し、stuck keyを残さない。

Text entry中もEscape／Cancel等のregistered system Actionだけは利用できる。Keyboard readingをUnicode文字へ変換しない。

## 6. T10 latch

`T10_InputLatch`は次を行う。

1. Device connection／reading queueをtick cutoffまでlatchする。
2. Platform timestamp、Device sequenceでsortし、重複を除去する。
3. disconnect／generation changeを先に適用する。
4. Device calibrationとAction processorを評価する。
5. Context／focus／consumptionを解決する。
6. Action value、started／performed／cancelled transitionを確定する。
7. `InputSnapshot`をcanonical Action ID順でsealする。

```text
InputSnapshot
  tick_id
  input_profile_hash
  active_context_hash
  assigned_device_set_hash
  actions[]
    action_id
    value
    phase
    transition_count
    last_transition_offset
```

Snapshotはtick中immutableで、GameplayはDevice readingを追加pollしない。複数transitionが一tickに入った場合もtap／repeat評価を失わず、Actionごとのbounded transition count最大16を超えたら`InputTransitionOverflow`としてauthoritative sessionをfaultする。

## 7. Platform Adapter

### 7.1 Windows

- GameInputをKeyboard、Mouse、Gamepadのprimary backendとする。
- Device add／remove callbackはinternal workerからbounded connection queueへcopyする。
- Stateは`T10`でGameInput Readingをpollし、必要な過去Readingをchronologicalに取り込む。
- Callback登録はshutdown前に解除し、callback contextをDevice Hubより長生きさせる。
- Win32 pointer／window focusをsurface generation付きeventへ正規化する。
- TextはTSF／IMEへ分離する。

### 7.2 Android

- GameActivity input bufferをframeごとにswapし、一方を処理中にnative producerと共有しない。
- ControllerはPaddleboatを更新し、GameActivity eventをforwardする。
- Touch ID、tool、position、pressure、phaseを正規化し、surface generation不一致eventを破棄する。
- Back gesture／system navigationをGame Actionと混同しない。

### 7.3 Apple

- UIKit touch／pointer、GameControllerのpoll／callbackをDevice Hubへ正規化する。
- Controller profileとUser remapを尊重し、固定vendor layoutを仮定しない。
- App inactive／controller disconnectでrelease／cancelを生成する。
- Haptic engine lifecycle、interruption、resetはPlatform Adapterが処理する。

## 8. TouchとGesture

C1 touchは最大10 contact、contactごとにID、down／move／up／cancel、position、delta、pressure optionalを持つ。

登録済みGestureを次に限定する。

- tap
- double tap
- long press
- drag
- pinch
- rotate

Gesture recognizerはUI layerとGameplay layerで別instanceを使い、同じcontactのownershipをFocus／Consumptionで解決する。AIはraw touch配列を直接Game ruleへ埋め込まず、Touch Control Layoutまたはsemantic Action Bindingを作る。

Virtual stick／buttonはUI DocumentのControlで、safe area、minimum target、dead zone、opacity、handedness、reposition policyを持つ。表示とInput hit regionを同じlayout resultから生成する。

## 9. RemapとAccessibility

- User RemapはProject Defaultを上書きするUser settingで、Project revisionを変更しない。
- Binding captureは次の有効controlを一つ取得し、timeout 10秒、Cancel Actionを常に残す。
- reserved OS shortcut、system command、同Contextのexclusive conflictを警告／拒否する。
- `Restore Defaults`、Action単位reset、Profile export／importを持つ。
- Required Actionがunboundになる変更を拒否する。
- Toggle／hold切替、repeat timing、stick dead zone／sensitivity、left-handed touch layoutをUser settingとして提供する。
- Controllerだけ、keyboardだけ、touchだけでCore menuとPlayを操作できるTarget Profileを検証する。

## 10. Haptics

`HapticCommand`はdevice／player、effect ID、low／high motor amplitude、duration、priority、owner generationを持つ。Amplitude `[0,1]`、duration 0.01～5 s、同時effect最大16／deviceとする。

Hapticsはpresentation-onlyでauthoritative stateへ戻さない。Focus loss、device disconnect、Play stopで全effectを停止する。Unsupported deviceではsilent successにせず`Unsupported` telemetryを返すが、Gameplayを失敗させない。

C1はgamepad rumbleとmobile registered impact／continuous patternを提供する。Custom waveform editor、adaptive trigger、HD haptics authoringはC2。

## 11. ReplayとDeterminism

Replayはnormalized authoritative Action value／transitionとInput Profile hashを記録する。Raw OS timestamp、Device ID、vendor packetを必須Replay dataにしない。

Replay時はPhysical Device readingをGameplayへ混ぜず、Replay `InputSnapshot`をT10 outputとして使用する。Pause／stop等のReplay controlは別System Contextに置く。Profile hash不一致は再生を拒否し、近似mapを行わない。

## 12. Failure policy

| Failure | 結果 |
|---|---|
| Device disconnect | release／cancelを生成、player assignmentをunassignedへ |
| Reading queue overflow | authoritative session fault。入力を黙ってdropしない |
| Stale device generation | reading破棄、counter |
| Invalid value | reading reject、Device diagnostics |
| Required Action unbound | Play prepare／Remap save拒否 |
| Focus loss | 全pressed Action cancel |
| Text／IME unavailable | Text UIをdisabledにし、Action keyboardで文字を推測しない |
| Haptics unavailable | typed Unsupported、Gameplay継続 |
| Surface generation mismatch | pointer／touch event破棄 |

## 13. Capacity、Performance、Memory Budget

`windows_desktop_v1`とC1 mobile profileの論理上限を次に固定する。低性能Targetは同じ上限を黙って縮小せず、Target validationで対応可否を判定する。

| 項目 | C1上限／Gate |
|---|---:|
| Connected logical device | 32 |
| Assigned local player | 8 |
| Control／device | 512 |
| Action／Project | 1,024 |
| Binding／Project | 4,096 |
| Active Context stack | 32 |
| Touch contact | 10 |
| Transition／Action／tick | 16 |
| Device reading queue | 全producer合計16,384 record。1 record最大64 byte、payload領域1 MiB |
| `InputSnapshot` | 最大1,024 Action、canonical serialized size 128 KiB |
| Persistent Input data | 8 MiB。Device descriptor、Action／Binding、calibration、User Remapを含む |
| Latch transient | 4 MiB。sort、deduplicate、gesture、snapshot stagingを含む |
| `T10_InputLatch` CPU | Windows referenceのP95 soft 0.35 ms。Runtime規約の`T00＋T10` 0.50 ms内に含む |

Persistent 8 MiBはRuntime規約のCore World／Saveにあるsnapshot／bridge 32 MiBへ、Latch transient 4 MiBはFrame／Job transientのFrame 32 MiBへchargeする。Runtime message queue予約後のsnapshot／bridge残量22 MiBからInput 8 MiBを差し引き、14 MiBを他のsnapshot／bridge用途へ残す。Queue、Snapshot、gesture recognizer、Platform Adapter allocationを別Domainへ二重計上しない。Mobileも絶対Process budgetはMobile規約を優先するが、この内部上限とtagは維持する。

上限超過は末尾dropやAction切捨てにせず、Cook／Play prepareで静的超過を拒否し、Runtime queue／transition超過はauthoritative session faultとする。Shipping callback／poll pathで一般heap allocation、lock待機、filesystem、log formattingを行わない。

## 14. TestとDefinition of Done

- Keyboard、Mouse、Xbox系／generic controller、touch、penのconnect／disconnect
- press／tap／hold／repeat／chord／toggle、dead zone、composite、conflict
- Focus／Context／Consumption、text entry中のGameplay漏れ
- 同tick最大16 transition、queue overflow、stale generation
- Windows GameInput callback解除／Reading履歴、Android buffer swap、Apple inactive
- Controller／keyboard／touchだけでmenu→Play→pause→exit
- User Remap、required Action、reserved shortcut、left-handed layout
- Replayで同じAction Snapshot／authoritative state hash
- Haptic stop、disconnect、unsupported device
- 60／120 Hz display、60 Hz simulationでinput-to-submit測定

C1完了条件は、同じ2D／3D ProjectがWindows keyboard／mouse／controller、Android touch／controller、Apple touch／controllerでsemantic Actionを共有し、Remap、Text分離、Replay、disconnectを合格することである。

## 15. 一次資料

- [GameInput Introduction](https://learn.microsoft.com/en-us/gaming/gdk/docs/features/common/input/overviews/input-overview)
- [GameInput Readings](https://learn.microsoft.com/en-us/gaming/gdk/docs/features/common/input/overviews/input-readings)
- [GameInput Callbacks](https://learn.microsoft.com/en-us/gaming/gdk/docs/features/common/input/advanced/input-callbacks)
- [Windows IME／Text Input requirements](https://learn.microsoft.com/en-us/windows/apps/develop/input/input-method-editor-requirements)
- [Android GameActivity](https://developer.android.com/games/agdk/game-activity/get-started)
- [Android Paddleboat](https://developer.android.com/games/sdk/game-controller/controller)
- [Apple Game Controller](https://developer.apple.com/documentation/gamecontroller/handling-input-events)
- [Apple Core Haptics](https://developer.apple.com/documentation/corehaptics)

Platform APIのdevice／callback方式はAdapter実装の根拠とし、Action、Context、Replay、RemapはMiraikanaiが独自に所有する。
