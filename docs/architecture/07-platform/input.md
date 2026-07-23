# Miraikanai Engine Input／Action／Device Contract

- 文書ID: mirakan.arch.platform-input
- 状態: review
- 正本範囲: Input device／reading、Action／Binding／Context、latch semantics、Platform input Adapter、touch／gesture、remap／accessibility、haptics、Input replay、Input固有capacity／failure／qualification
- 非正本範囲: Runtime phase／shared queue・memory budget、UI／Text event、Platform lifecycle、Tool／SDK version、Product phase、AI authorization／Evidence envelope。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Editor UI Framework](../03-authoring/editor-ui-framework.md)、[Editor workspace／UX](../03-authoring/editor-workspace-ux.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[Windows](windows.md)、[Mobile Common](mobile-common.md)、[Android](android.md)、[Apple](apple.md)、[UI／Text](ui-text-localization-accessibility.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論

Miraikanai EngineはPlatformのkey code、controller object、touch callbackをGame ruleへ直接渡さない。Platform AdapterがDevice ReadingをEngine値へ正規化し、`InputActionMap`がsemantic Actionへ解決し、`T10_InputLatch`でtick番号付きimmutable `InputSnapshot`を確定する。

Gameplay、UI、AI、NativeGameModuleが参照するのはStable `InputActionId`とSnapshotだけである。Text入力／IMEはAction Inputと別の`ITextInputService`が所有し、keyboard stateから文字を推測しない。

Editor UIのpointer、keyboard、IME、Window eventは`PlatformUiEventV1`へ正規化し、MirakanUi Event／Focus Routerへ渡す。Editor Command操作をGameplayの`T10_InputLatch`へ混入させず、Editor UI event、Game InputSnapshot、Text compositionの三経路を分離する。

## 2. 決定権と対象外

| 主題 | 正本 |
|---|---|
| Device正規化、Action、Binding、Snapshot、Remap、Haptics | 本書 |
| Runtime slot／tick、shared queue／thread／Replay envelope | Runtime owners |
| Game UI focus／event、Text／IME | UI／Text規約 |
| Editor UI event、Focus、Command、TSF integration | Editor UI Framework規約 |
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

相対軸reading（mouse delta、scroll wheel等のdelta control）は、同一Device・同一controlの連続readingをqueue内で決定的にcoalesceする。deltaは加算し、timestamp／sequenceは最後のreadingを採用する。digital transitionとabsolute axisはcoalesceしない。これによりqueue使用量を高polling rate device（例: 8 kHz mouse）と一時的stallの積に比例させず、§13のreading queue上限はcoalescing適用後の最悪流入率×許容stall時間を上回るよう設定する。

## 4. Action model

### 4.1 Action

| Field | 規則 |
|---|---|
| `action_id` | Authoring SourceのUUIDv7 `StableId`。表示名をidentityに使わない |
| `value_type` | `digital \| axis1 \| axis2 \| pointer2` |
| `scope` | `gameplay \| ui \| editor \| system` |
| `consumption` | `shared \| focused \| exclusive` |
| `default_value` | 型ごとのzero |
| `interactions` | 最大4 |
| `processors` | 最大8、固定順 |
| `required` | Profileごとのbool |
| `replay_policy` | `authoritative \| presentation_only \| excluded` |

Pose、text、arbitrary byte payloadをAction valueへ入れない。

Cookerは一つのexact `InputActionMap` Artifact内でAction `StableId`をUUID byte順に並べ、1から`RuntimeActionId uint32`を割り当てる。0はinvalidとし、Artifactへ`StableId`↔`RuntimeActionId`対応表を含める。Runtime dispatchは`RuntimeActionId`、Project Source、User binding、Save／Replay headerはAction `StableId`＋Action Map `ArtifactRefV1`を使い、別Artifactの同じ数値を比較しない。

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

`control_path`はEngine Input Control Catalogのclosed IDであり、Platform key code文字列を正規dataにしない。Bindingは最大4096／Project、Actionは最大1024、同一ActionのBindingは最大32とする。Actionの`interactions`は§4.3 closed setから選ぶ重複なしの許可集合で、各Bindingの`interaction`はその集合のexact一要素を必須とする。Bindingごとのinteraction stateは`{binding_id, device_generation}`で分離し、一つのActionを共有する別Bindingのtap／hold／repeat stateを合成または上書きしない。

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

### 4.5 Semantic Action-to-Command Binding

Input Coreはgenre固有Action listを所有せず、Action roleをProject／Pack所有のtyped Commandへ結ぶ次のgeneric recordとregistryだけを所有する。Project適用時に各ActionのUUIDv7 `StableId`を生成し、role文字列をRuntime dispatchまたはSave identityにしない。

```text
SemanticActionCommandBindingV1
  binding_id: StableId
  owner_ref/hash
  semantic_action_role_ref: McdContractRefV1(kind=semantic_role)
  action_value_schema_ref: McdContractRefV1(kind=type)
  command_schema_ref: McdContractRefV1(kind=type)
  evaluator_policy_ref: McdContractRefV1(kind=policy)
  target_system_ref: GameSystemContractRefV1
  required_context_refs[1..32]
  fixture_refs[1..64]
  binding_content_hash: SHA-256

SemanticActionCommandBindingRegistryV1
  registry_id: input.semantic_action_command_binding.registry.active
  registry_version
  registry_content_hash
  records[0..4096]: SemanticActionCommandBindingV1
```

recordsはrole ID／version、target System ref、binding IDのcanonical byte順へstrict sortし、duplicate、同じrole＋Contextへの複数active binding、stale owner／schema／policy／fixture hash、cross-owner namespace偽装を拒否する。BindingはT10 `InputSnapshot`をT30でregistered typed Commandへ変換するだけで、Domain stateを直接変更しない。Replay、AI、Keyboard／Mouse、Controller、Touchは同じAction／Command経路を使う。個別Genre／Featureのrole、required／optional Action Profile、Command schema、evaluator、fixtureは当該Pack ownerが登録し、Input Coreのschema、binary、fixture inventoryへコピーしない。

## 5. Context、Focus、Consumption

公式Contextを次で固定する。

- `system`
- `editor_global`
- `editor_view`
- `gameplay`
- `game_ui`
- `text_entry`
- `debug`

Active Context stackは`T00`で更新し、tick中は不変とする。Priorityは`system`、`text_entry`、`game_ui`、`gameplay`、`debug`の順で、Debug Shipping無効時は存在しない。

`editor_global`／`editor_view`はEditor Command dispatchに使わず、Play-in-Editor時のGame InputSnapshot経路への転送制御専用とする。`editor_global`はEditor session全体の転送可否、`editor_view`はfocusを持つviewportへの転送可否を制御し、`editor_view`がfocusを持つ間だけ`gameplay`／`game_ui`／`text_entry` Contextを有効化する。Editor UI eventは§1の三経路分離どおり`PlatformUiEventV1`経路が所有し、Shipping Gameは両Contextをactivateしない。

`exclusive` Actionがcontrolをconsumeした場合、低priority Contextへ同じphysical transitionを渡さない。Shared movement stickとUI navigationを同時に有効化する場合は明示`shared` Bindingを必要とする。Focus loss時は全down stateへcanonical releaseを生成し、stuck keyを残さない。

Text entry中もEscape／Cancel等のregistered system Actionだけは利用できる。Keyboard readingをUnicode文字へ変換しない。

## 6. T10 latch

`T10_InputLatch`は次を行う。

1. Device connection／reading queueをtick cutoffまでlatchする。
2. Platform timestamp、Device sequenceでsortし、重複を除去する。
3. disconnect／generation changeを先に適用する。
4. Device calibrationとAction processorを評価する。
5. Context／focus／consumptionを解決する。pointer系controlは前tickの`UiHitTestSnapshot`（pointer-over-UI集合、[UI／Text](ui-text-localization-accessibility.md) §3）を入力とし、UI上のpointer down／up transitionは`game_ui` Contextがconsumeして`gameplay` Contextへ渡さない。
6. Action value、started／performed／cancelled transitionを確定する。
7. `InputSnapshot`をcanonical Action ID順でsealする。

同一Actionへ複数Binding transitionが入る場合、まずPlatform timestamp昇順、Engine sequence昇順、Binding priority降順、`binding_id`のcanonical UUID byte順へ整列し、その順でAction value type固有のregistered foldを適用する。同じphysical transitionはContext／consumption解決後に一度だけ残し、arbitrary last-write-wins、container順、callback順でmergeしない。

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

Snapshotはtick中immutableで、GameplayはDevice readingを追加pollしない。`phase`はfold後に最後に成立したtransitionの`started | performed | cancelled`、transitionがなければ`none`であり、`transition_count`は当該tickで受理した全transition数、`last_transition_offset`は最後の受理位置である。複数transitionが一tickに入った場合もtap／repeat評価を失わず、Actionごとのbounded transition countは最大16とする。超過時は前段のcanonical順の先頭16 transitionを保持し、残りを破棄してActionごとのdropped countとともにtyped `InputTransitionOverflow` diagnosticを発行する。この縮約は決定的でSnapshotへ記録し、authoritative session faultにしない。hitch中に蓄積した正常入力でGameを落とさない。

`input_profile_hash`の対象である「Input Profile」を次で定義する。`InputActionMap` Artifact（`ArtifactRefV1`＋`StableId`↔`RuntimeActionId`対応表）、公式Context定義集合（§5）、interaction timing値（§4.3）、processor既定（§4.4）、composite構成の組であり、hashはこの列挙順のcanonical serializationから計算する。User RemapとDevice構成はInput Profileへ含めない。Replayはnormalized Action value／transitionを記録するため、Remap変更は再生可能性を失わせない。

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

qualificationが要求する「同一input trace」は`SyntheticInputTraceV1`で供給する。tick offset付きのAction `StableId`／value／transition列、Action Map `ArtifactRefV1`、Input Profile hashを持つversioned MCDであり、AI／CIがGameSpec／Test Scenarioから生成する。再生は本節のReplay `InputSnapshot`注入経路を再利用してT10 outputとし、別のFire経路を作らない（§4.5）。記録由来traceとsynthetic traceは同じschemaへ正規化し、同一trace再生が同一のAction Snapshot／authoritative state hashを生むことをfixtureで検証する。Replay envelope（container／transport）はRuntime ownersの正本であり、本書はtrace内容schemaだけを所有する。

## 12. Failure policy

| Failure | 結果 |
|---|---|
| Device disconnect | release／cancelを生成、player assignmentをunassignedへ |
| Reading queue overflow | authoritative session fault。相対軸coalescing（§3.2）適用後の超過に限り、入力を黙ってdropしない |
| Transition／Action／tick超過 | 先頭16 transitionを保持、超過分をtyped `InputTransitionOverflow`＋dropped countへ縮約（§6）。session faultにしない |
| Stale device generation | reading破棄、counter |
| Invalid value | reading reject、Device diagnostics |
| Required Action unbound | Play prepare／Remap save拒否 |
| Focus loss | 全pressed Action cancel |
| Text／IME unavailable | Text UIをdisabledにし、Action keyboardで文字を推測しない |
| Haptics unavailable | typed Unsupported、Gameplay継続 |
| Surface generation mismatch | pointer／touch event破棄 |

## 13. Capacity、Performance、Memory Budget

Input domainの論理上限を次に固定する。低性能Targetは同じ上限を黙って縮小せず、Target validationで対応可否を判定する。共通queue、frame、memory envelopeは[Runtime performance／capacity](../04-runtime/performance-capacity.md)を参照する。

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
| Device reading queue | Input Adapter全producer合計16,384 record。1 record最大64 byte、Input-owned payload arena 1 MiB。相対軸readingは§3.2のcoalescing適用後に計上する |
| `InputSnapshot` | 最大1,024 Action、canonical serialized size 128 KiB |
| Persistent Input data | 8 MiB。Device descriptor、Action／Binding、calibration、User Remapを含む |
| Latch transient | 4 MiB。sort、deduplicate、gesture、snapshot stagingを含む |
| Input latch CPU | Windows referenceのP95 soft 0.35 ms。[Runtime performance／capacity](../04-runtime/performance-capacity.md)のmeasurement window内で測る |

Persistent 8 MiBとLatch transient 4 MiBはRuntime ownerのInput child scopeへchargeする。Queue、Snapshot、gesture recognizer、Platform Adapter allocationを別Domainへ二重計上しない。Mobile aggregate capは[Mobile Common](mobile-common.md)を参照するが、このInput内部上限とtagは維持する。

上限超過は末尾dropやAction切捨てにせず、Cook／Play prepareで静的超過を拒否し、Runtime queue超過はauthoritative session faultとする。transition超過だけは§6のtyped縮約で処理する。Shipping callback／poll pathで一般heap allocation、lock待機、filesystem、log formattingを行わない。

## 14. TestとDefinition of Done

- Keyboard、Mouse、Xbox系／generic controller、touch、penのconnect／disconnect
- press／tap／hold／repeat／chord／toggle、dead zone、composite、conflict
- Focus／Context／Consumption、text entry中のGameplay漏れ
- 同tick最大16 transitionと超過縮約、相対軸coalescing、queue overflow、stale generation
- Windows GameInput callback解除／Reading履歴、Android buffer swap、Apple inactive
- Controller／keyboard／touchだけでmenu→Play→pause→exit
- User Remap、required Action、reserved shortcut、left-handed layout
- Project／Pack所有Action Profileのrequired／optional role、hold／release／repeat、semantic binding 0件／複数、owner／schema／policy hash不一致
- Replayで同じAction Snapshot／authoritative state hash
- Haptic stop、disconnect、unsupported device
- 60／120 Hz display、60 Hz simulationでinput-to-submit測定

C1完了条件は、同じ2D／3D ProjectがWindows keyboard／mouse／controller、Android touch／controller、Apple touch／controllerでsemantic Actionを共有し、Remap、Text分離、Replay、disconnectを合格することである。

## 15. 実装・外部依存境界

Device／Action／Binding／Context／SnapshotのMCD、generated value type、closed Control Catalog、Validator、canonical serialization、Result／DiagnosticはPlatform SDK型を含めない。Windows、Android、Apple Adapterは同じAction resolver、Context、Snapshot、Remap、Replayを再利用し、Game rule、Action ID、Replay formatをforkしない。

OS input／text／haptics API、SDKのexact version、hash、license、取得元は[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)だけが所有する。Adapterは本書のnormalization、lifetime、callback、qualification contractへ写像し、native objectを公開しない。
