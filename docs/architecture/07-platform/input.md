# Miraikanai Engine Input／Action／Device Contract

- 文書ID: mirakan.arch.platform-input
- 状態: review
- 正本範囲: Input device／reading、Action／Binding／Context、latch semantics、Platform input Adapter、touch／gesture、remap／accessibility、haptics、Input replay、Input固有capacity／failure／qualification
- 非正本範囲: Runtime phase／shared queue・memory budget、UI／Text event、Platform lifecycle、Tool／SDK version、Product phase、AI authorization／Evidence envelope。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Editor UI Framework](../03-authoring/editor-ui-framework.md)、[Editor workspace／UX](../03-authoring/editor-workspace-ux.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[Windows](windows.md)、[Mobile Common](mobile-common.md)、[Android](android.md)、[Apple](apple.md)、[UI／Text](ui-text-localization-accessibility.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論

Miraikanai EngineはPlatformのkey code、controller object、touch callbackをGame ruleへ直接渡さない。Platform AdapterがDevice ReadingをEngine値へ正規化し、`InputActionMap`がsemantic Actionへ解決し、`T10_InputLatch`でSimulation Advance sequence付きimmutable `InputSnapshot`を確定する。Adapter callbackからSnapshotへのcopy、bounded queue、callback captureのPointer／Memory規則は[Memory／Pointers](../02-foundation/memory-pointers.md)のbindingを使い、Platform object、callback pointer、borrowをsnapshot／Replayへ渡さない。

Gameplay、UI、AI、NativeGameModuleが参照するのはStable `InputActionId`とSnapshotだけである。Text入力／IMEはAction Inputと別の`ITextInputService`が所有し、keyboard stateから文字を推測しない。

Editor UIのpointer、keyboard、IME、Window eventは`PlatformUiEventV1`へ正規化し、MirakanUi Event／Focus Routerへ渡す。Editor Command操作をGameplayの`T10_InputLatch`へ混入させず、Editor UI event、Game InputSnapshot、Text compositionの三経路を分離する。

## 2. 決定権と対象外

| 主題 | 正本 |
|---|---|
| Device正規化、Action、Binding、Snapshot、Remap、Haptics | 本書 |
| Runtime slot／Simulation Advance、shared queue／thread／Replay envelope | Runtime owners |
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
SemanticActionRoleRefV1
  role_type_ref: McdContractRefV1(kind=type)

ContextConstraintRefV1
  constraint_id
  constraint_version: positive uint32
  constraint_content_hash: SHA-256
  owner_layer: core | feature_pack | genre_pack | project
  owner_ref: exact GameSystemOwnerRefV1 with matching owner_layer

ContextConstraintRecordV1
  constraint_ref: ContextConstraintRefV1
  allowed_context_ids[1..32]: closed Input Context IDs
  required_context_ids[0..32]: closed Input Context IDs
  excluded_context_ids[0..32]: closed Input Context IDs

SemanticActionCommandBindingRecordV1
  binding_id: StableId
  binding_version: positive uint32
  owner_layer: core | feature_pack | genre_pack | project
  owner_ref: exact GameSystemOwnerRefV1 with matching owner_layer
  semantic_action_role_ref: SemanticActionRoleRefV1
  action_value_schema_ref: McdContractRefV1(kind=type)
  command_schema_ref: McdContractRefV1(kind=type)
  evaluator_policy_ref: McdContractRefV1(kind=policy)
  target_system_ref: GameSystemContractRefV1
  context_constraint_refs[1..32]: ContextConstraintRefV1
  binding_content_hash: SHA-256

SemanticActionCommandBindingRecordRefV1
  registry_ref: SemanticActionCommandBindingRegistryRefV1
  binding_id
  binding_version
  binding_content_hash

SemanticActionCommandBindingRegistryRefV1
  registry_id
  registry_version
  registry_content_hash

SemanticActionCommandBindingContributionV1
  contribution_id
  contribution_version: positive uint32
  contribution_content_hash: SHA-256
  owner_layer: core | feature_pack | genre_pack | project
  owner_ref: exact GameSystemOwnerRefV1 with matching owner_layer
  binding_records[1..1024]: SemanticActionCommandBindingRecordV1

SemanticActionCommandBindingContributionRefV1
  contribution_id
  contribution_version: positive uint32
  contribution_content_hash: SHA-256
  owner_layer: core | feature_pack | genre_pack | project
  owner_ref: exact GameSystemOwnerRefV1 with matching owner_layer

SemanticActionCommandBindingRegistryV1
  registry_id: input.semantic_action_command_binding.registry.active
  registry_version
  registry_content_hash
  contribution_refs[0..4096]:
    SemanticActionCommandBindingContributionRefV1
  records[0..4096]: SemanticActionCommandBindingRecordV1

SemanticActionBindingQualificationSubjectRefV1
  subject_kind: record | contribution
  subject_ref:
    record: SemanticActionCommandBindingRecordRefV1
    | contribution: SemanticActionCommandBindingContributionRefV1

SemanticActionBindingQualificationSubjectV1
  qualification_id
  qualification_version: positive uint32
  owner_layer: core | feature_pack | genre_pack | project
  owner_ref: exact GameSystemOwnerRefV1 with matching owner_layer
  qualified_subject_ref:
    SemanticActionBindingQualificationSubjectRefV1
  target_profile_refs[1..64]
  fixture_refs[1..64]: exact {fixture_id, fixture_version, fixture_content_hash}
  input_closure_hash: SHA-256
  result: pass | fail
  qualification_subject_hash: SHA-256

SemanticActionBindingQualificationReceiptV1
  subject: SemanticActionBindingQualificationSubjectV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=semantic_action_binding_qualification)

SemanticActionBindingQualificationReceiptRefV1
  qualification_id
  qualification_version: positive uint32
  qualification_subject_hash: SHA-256
  signed_record_hash: SHA-256

SemanticActionBindingQualificationBindingRefV1
  qualification_binding_id
  qualification_binding_version: positive uint32
  qualification_binding_hash: SHA-256

SemanticActionBindingQualificationBindingV1
  qualification_binding_id
  qualification_binding_version: positive uint32
  qualified_subject_ref:
    SemanticActionBindingQualificationSubjectRefV1
  qualification_receipt_ref:
    SemanticActionBindingQualificationReceiptRefV1
  qualification_binding_hash: SHA-256

SemanticActionBindingActivationCatalogV1
  catalog_id: input.semantic_action_binding.activation_catalog.active
  catalog_version: positive uint32
  entries[0..8192]:
    exact {
      qualified_subject_ref: SemanticActionBindingQualificationSubjectRefV1,
      qualification_binding_ref:
        SemanticActionBindingQualificationBindingRefV1}
  catalog_hash: SHA-256

SemanticActionBindingSelectionDocumentV1
  common Project Document header
  selection_id: same StableId as Document header
  action_map_ref: ArtifactRefV1
  selected_bindings[0..4096]:
    exact {action_stable_id, SemanticActionCommandBindingRecordRefV1}
  selected_qualification_binding_refs[0..4096]:
    SemanticActionBindingQualificationBindingRefV1
  selection_content_hash: SHA-256
```

Action roleのMCD kindは既存closed kindの`type`である。各role recordは`type.input.semantic_action_role.<owner_namespace>.<role>`のactive MCD type、version、Contract set rootを持ち、そのpayloadがowner、accepted Action value type、Command type、exact `ContextConstraintRefV1`を宣言する。Refの五Fieldは一つのactive `ContextConstraintRecordV1`へexact解決し、latest fallbackを許可しない。Context constraintの三つのContext ID配列はNFC UTF-8 byte順でstrict sortし、duplicate、allowed外required、requiredとexcludedの交差、unknown Context IDを拒否する。未定義`kind=semantic_role`、bare role／Context文字列、Action表示名をcurrent Source／Save／Replayへ受理しない。

record hashはASCII `MIRAKAN_SEMANTIC_ACTION_COMMAND_BINDING_RECORD_V1`、contribution hashはASCII `MIRAKAN_SEMANTIC_ACTION_COMMAND_BINDING_CONTRIBUTION_V1`、それぞれself-excluding Receipt-free canonical bytesを`uint32_be` length framingして計算する。Record／ContributionへQualification Receipt／Bindingを埋め込まない。Contributionはownerの完全snapshotであり、全recordの`owner_layer／owner_ref`がContributionとexact equalityでなければならない。Pack ownerはexact `PackContractRefV1`から写像したowner refとnamespace、Project ownerはcompile対象のexact Project tripleから写像したowner refを要求し、Core namespaceまたはowner文字列の自己申告、Feature／Genre layer spoof、cross-owner recordを拒否する。Production Contribution／RecordはFixture bodyもQualification Receiptも解決しない。

materializerはowner inventoryからownerごとにexact一件のReceipt-free ContributionRefを選び、`owner_layer`のclosed ordinal、owner ID／revision／hash、contribution ID／version／hashの順でstrict sortする。各Contribution内のrecordはrole type ID／version／Contract set hash、context constraint set hash、target System ref、binding ID／versionの順でstrict sortし、flatten後も同じ順へ再sortする。collision keyは`{semantic_action_role_ref, canonical context_constraint_ref set}`であり、異なるownerを含む二件以上、同一binding IDの別hash、同じownerの複数active contributionを優先順位で解決せずRegistry全体を拒否する。Registry hashはASCII `MIRAKAN_SEMANTIC_ACTION_COMMAND_BINDING_REGISTRY_V1`、Registry ID／version、ContributionRef count／全Ref、record count／全Receipt-free record canonical bytesを各`uint32_be` length framingしてSHA-256し、自身だけを除外する。RecordRefはRegistryRef三Fieldとrecord identity／version／hashを一つにbindし、latest recordへfallbackしない。

Registry／RecordRef確定後の唯一の生成順は`receipt-free Record／Contribution → Registry／base refs → Qualification subject → signed Receipt → Qualification binding → Activation Catalog／Selection`である。subject hashはASCII `MIRAKAN_SEMANTIC_ACTION_BINDING_QUALIFICATION_SUBJECT_V1`、binding hashはASCII `MIRAKAN_SEMANTIC_ACTION_BINDING_QUALIFICATION_BINDING_V1`、Activation Catalog hashはASCII `MIRAKAN_SEMANTIC_ACTION_BINDING_ACTIVATION_CATALOG_V1`と各自己Fieldを除くcount／length-framed canonical bytesから計算する。Subject／Bindingの`qualified_subject_ref.subject_kind`と対応`subject_ref` branchは同じtagged valueとしてbyte equality、branch外Refはcanonical omissionにする。subjectのowner layer/refは`record` branchでは参照先Record、`contribution` branchでは参照先Contributionの同Fieldとbyte equalityでなければならない。Bindingの`qualified_subject_ref`はReceipt subjectの同Fieldとbyte equalityにする。Activation Catalog entryの`qualified_subject_ref`はBindingの同Fieldとbyte equalityで、各base tagged subjectへexact一Bindingを対応させる。SelectionのRecordRef集合と`qualified_subject_ref.subject_kind=record`のqualification bindingが解決するRecordRef集合をexact set equalityにする。discriminator外branch、両branch混在、branch外Field残存、Record refをContributionとして解釈するcase、Catalog entryのtagまたはRefだけをBindingから置換するcaseを各一原因fixtureで拒否する。Receipt／Binding／CatalogをRecord、Contribution、Registry hashへ戻さない。Productionはsigned Receiptのsubject／result=`pass`／freshness／revocationだけを検証し、Fixture bodyを解決しない。

owner removalは署名済みPack deactivationまたは同じProject commitに含まれるowner-removal recordだけを根拠に、次Registry versionから当該owner Contribution全体を除く。単にContributionが欠落、timeout、invalid、Qualification失敗になっただけでは削除と解釈しない。merge、sort、collision、owner spoof、removal検証のいずれかに失敗した場合はactive Registry、Runtime lookup、Compile Manifestを部分更新せず、last-valid三点を維持する。

Project Sourceの正本は`SemanticActionBindingSelectionDocumentV1`である。ただし選択write surfaceは本Taskで完全登録されていないため、`operation.input.semantic_action_binding.select`は[Executable contracts](../02-foundation/executable-contracts.md#211-既存domain文書から回収した未登録operation候補)の`planning.operation_family.input_binding_selection@1`に属するexact一候補であり、current MCD、Owner Manifest、Service allowlist、Policy、Validator、Diagnostic、Receipt、Provider／MCP Catalog、generated alias、legacy aliasの各集合をすべて`[]`、Capability stateを`not_activated`とする。選択要求は`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`でSource不変にする。future work item `activation.input.semantic_action_binding_selection.v1`はinitial create/upsertとupdate、selection semantic hash、Action StableId／RecordRef canonical sortとduplicate rule、MCD全Field、Policy／Validator／Diagnostic／canonical signed Receipt、private-to-public recoveryを同じContract set transactionで完全登録するまでactivateしない。Binding Registry、Runtime lookup table、Compile Manifestは既存Source＋active owner Contributionからのみ決定的に再materializeし、Registry直接writeを公開しない。

Compile closureはSelection Document ref／hash、Action Map ArtifactRef、RegistryRef、Activation Catalog ref／hash、全ContributionRef、全RecordRef、全Qualification Binding ref、selection set hashを持つ。SaveはAction StableId、Action Map ArtifactRef、Selection Document ref／hash、RecordRef、Qualification Binding refを保存し、Replay headerはそれらとInput Profile hash、`SimulationCadenceProfileRefV1`、各advanceの`SimulationAdvanceIntervalV1.interval_content_hash`、normalized Action transition／生成Command hashを記録する。Input側Cadence Profile ref／hashはRuntime Replay headerの同Fieldとbyte equalityにし、差があれば再生前に拒否する。Load／Replay／CompileはSource→Registry→Contribution→Record→Qualification subject→signed Receipt→Binding→role type→Context constraint→Command Systemの全ref equalityを再検査する。`fixture.input.semantic-action-binding-roundtrip`は既存Selection reload→Compile→Save／Load→Replayを通し、wrong MCD kind、stale Registry／contribution／record／subject／Receipt／binding／selection、Cadence Profile／interval hash差、subject owner mismatch、Receipt substitution、同role＋Context collision、noncanonical merge、owner removal spoof、cross-owner／layer spoof、ProductionからFixture body参照、SourceとDerived差を各単独原因でrejectする。

BindingはT10 `InputSnapshot`をT30でregistered typed Commandへ変換するだけで、Domain stateを直接変更しない。Replay、AI、Keyboard／Mouse、Controller、Touchは同じAction／Command経路を使う。個別Genre／Featureのrole、required／optional Action Profile、Command schema、evaluator、Contribution、Qualification record／Fixtureは当該Pack ownerが登録し、Input Coreのschema、binary、fixture inventoryへコピーしない。

## 5. Context、Focus、Consumption

公式Contextを次で固定する。

- `system`
- `editor_global`
- `editor_view`
- `gameplay`
- `game_ui`
- `text_entry`
- `debug`

Active Context stackは`T00`で更新し、一advance中は不変とする。Priorityは`system`、`text_entry`、`game_ui`、`gameplay`、`debug`の順で、Debug Shipping無効時は存在しない。

`editor_global`／`editor_view`はEditor Command dispatchに使わず、Play-in-Editor時のGame InputSnapshot経路への転送制御専用とする。`editor_global`はEditor session全体の転送可否、`editor_view`はfocusを持つviewportへの転送可否を制御し、`editor_view`がfocusを持つ間だけ`gameplay`／`game_ui`／`text_entry` Contextを有効化する。Editor UI eventは§1の三経路分離どおり`PlatformUiEventV1`経路が所有し、Shipping Gameは両Contextをactivateしない。

`exclusive` Actionがcontrolをconsumeした場合、低priority Contextへ同じphysical transitionを渡さない。Shared movement stickとUI navigationを同時に有効化する場合は明示`shared` Bindingを必要とする。Focus loss時は全down stateへcanonical releaseを生成し、stuck keyを残さない。

Text entry中もEscape／Cancel等のregistered system Actionだけは利用できる。Keyboard readingをUnicode文字へ変換しない。

## 6. T10 latch

`T10_InputLatch`は次を行う。

1. Device connection／reading queueをadvance cutoffまでlatchする。
2. Platform timestamp、Device sequenceでsortし、重複を除去する。
3. disconnect／generation changeを先に適用する。
4. Device calibrationとAction processorを評価する。
5. Context／focus／consumptionを解決する。pointer系controlは直前advanceの`UiHitTestSnapshot`（pointer-over-UI集合、[UI／Text](ui-text-localization-accessibility.md) §3）を入力とし、UI上のpointer down／up transitionは`game_ui` Contextがconsumeして`gameplay` Contextへ渡さない。
6. Action value、started／performed／cancelled transitionを確定する。
7. `InputSnapshot`をcanonical Action ID順でsealする。

同一Actionへ複数Binding transitionが入る場合、まずPlatform timestamp昇順、Engine sequence昇順、Binding priority降順、`binding_id`のcanonical UUID byte順へ整列し、その順でAction value type固有のregistered foldを適用する。同じphysical transitionはContext／consumption解決後に一度だけ残し、arbitrary last-write-wins、container順、callback順でmergeしない。

```text
InputSnapshot
  cadence_profile_ref: SimulationCadenceProfileRefV1
  simulation_advance_interval_hash: SHA-256
  advance_sequence: positive uint64
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

Snapshotは一advance中immutableで、GameplayはDevice readingを追加pollしない。先頭三FieldはScheduling Ownerが当該T00～T110へ発行した`SimulationAdvanceIntervalV1`とbyte equalityにする。`phase`はfold後に最後に成立したtransitionの`started | performed | cancelled`、transitionがなければ`none`であり、`transition_count`は当該advanceで受理した全transition数、`last_transition_offset`は最後の受理位置である。複数transitionが一advanceに入った場合もtap／repeat評価を失わず、Actionごとのbounded transition countは最大16とする。超過時は前段のcanonical順の先頭16 transitionを保持し、残りを破棄してActionごとのdropped countとともにtyped `InputTransitionOverflow` diagnosticを発行する。この縮約は決定的でSnapshotへ記録し、authoritative session faultにしない。hitch中に蓄積した正常入力でGameを落とさない。

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

Replayはnormalized authoritative Action value／transition、Input Profile hash、exact `SimulationCadenceProfileRefV1`、各`SimulationAdvanceIntervalV1`を記録する。Raw OS timestamp、Device ID、vendor packetを必須Replay dataにしない。

Replay時はPhysical Device readingをGameplayへ混ぜず、Replay `InputSnapshot`をT10 outputとして使用する。Pause／stop等のReplay controlは別System Contextに置く。Profile hash不一致は再生を拒否し、近似mapを行わない。

qualificationが要求する「同一input trace」は`SyntheticInputTraceV1`で供給する。これはAction Map `ArtifactRefV1`、Input Profile hash、exact `SimulationCadenceProfileRefV1`、開始advance sequence、strict昇順の`SimulationAdvanceIntervalV1[]`、各intervalへのadvance offset付きAction `StableId`／value／transition列を持つversioned MCDであり、AI／CIがGameSpec／Test Scenarioから生成する。各IntervalのProfile ref、sequence、hashはtrace headerと一致し、variableのexact duration、turn-basedのaccepted Command、explicit-stepのaccepted request／ordinalを欠落させない。再生は本節のReplay `InputSnapshot`注入経路を再利用してT10 outputとし、別のFire経路を作らない（§4.5）。記録由来traceとsynthetic traceは同じschemaへ正規化し、同一trace再生が同一のAction Snapshot／authoritative state hashを生むことをfixtureで検証する。Replay envelope（container／transport）はRuntime ownersの正本であり、本書はtrace内容schemaだけを所有する。

## 12. Failure policy

| Failure | 結果 |
|---|---|
| Device disconnect | release／cancelを生成、player assignmentをunassignedへ |
| Reading queue overflow | authoritative session fault。相対軸coalescing（§3.2）適用後の超過に限り、入力を黙ってdropしない |
| Transition／Action／advance超過 | 先頭16 transitionを保持、超過分をtyped `InputTransitionOverflow`＋dropped countへ縮約（§6）。session faultにしない |
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
| Transition／Action／advance | 16 |
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
- 同一advance最大16 transitionと超過縮約、相対軸coalescing、queue overflow、stale generation
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
