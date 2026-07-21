# Miraikanai Engine Windows Platform／Distribution Contract

- 文書ID: mirakan.arch.platform-windows
- 状態: review
- 正本範囲: Windows Target Profile、process／window／display／lifecycle Adapter、filesystem／user data、Windows package／signing／publication／update、Windows crash／security／qualification
- 非正本範囲: 外部Tool／SDK／OS／graphics version、共通Runtime budget／phase、Asset lifecycle、Renderer／Input／Audio／UI意味、AI authorization／Evidence envelope。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core architecture](../02-foundation/core-architecture.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Naming／project layout](../02-foundation/naming-project-layout.md)、[C++23 modules](../02-foundation/cpp23-modules.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Editor UI Framework](../03-authoring/editor-ui-framework.md)、[Editor workspace／UX](../03-authoring/editor-workspace-ux.md)、[Native game module](../03-authoring/native-game-module.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[Render Graph](../06-rendering/render-graph.md)、[Input](input.md)、[Audio](audio.md)、[UI／Text](ui-text-localization-accessibility.md)、[Mobile Common](mobile-common.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論

`windows_desktop_v1`は[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)のWindows exact baselineを参照するx86-64 desktop Targetである。Editorも最初のGame RuntimeもこのTargetから実装する。OS、graphics runtime、shader model、SDK、compilerの固定値は本書へ複写しない。

Windows固有APIは`engine/platform/windows`と各Backend Adapterに閉じ、正規Project、GameplayDefinition、NativeGameModule、Save、AI ToolへWin32、COM、HANDLE、HRESULT、GameInput、XAudio2、D3D12型を公開しない。

生成Gameの公式Distributionは次の二つである。

- `windows_msix_v1`: Microsoft Store、enterprise、直接配布向けの署名済みfull-trust MSIX
- `windows_managed_layout_v1`: Steam等の外部配布clientがinstall／updateを管理する署名済みDirectory layout

Development用portable layoutはShipping Distributionではない。Miraikanai独自の常駐auto-updater、kernel driver、system serviceをC1で実装しない。

## 2. 決定権と対象外

| 主題 | 正本 |
|---|---|
| Windows OS／ABI、Window、Process、Filesystem、Package、Signing、Crash | 本書 |
| MirakanUi Widget、Docking、Editor Window composition、DirectWrite／TSF／UIA／OLE Adapter契約 | Editor UI Framework規約 |
| D3D12 resource／frame／device loss | Rendering規約 |
| Input、IME、UI／Text、Audio意味 | Player I/O規約 |
| `.mirakanpack`、VFS、Patch／DLC | Asset規約 |
| C++ toolchain、SDK／graphics runtime baseline | Toolchain／Dependencies |
| 共通memory／capacity | Runtime performance／capacity |
| Android／Apple | Android／Apple規約 |

Toolchain ownerが固定したbaseline以外のWindows release／edition、ARM64／emulation、Xbox／GDK、UWP sandbox、Windows Server、Wine／ProtonをC1正式Targetに含めない。将来追加は別Target Profileと実機／package／performance gateを必要とする。

## 3. Target Profile

```text
WindowsDesktopTargetProfileV1
  profile_id = windows_desktop_v1
  toolchain_profile_ref
  architecture = x86_64
  executable_model = win32_full_trust
  renderer_profile_ref
  dpi_awareness = per_monitor_v2
  input_backend = gameinput
  audio_backend = xaudio2
  text_input_backend = tsf_win32
  distribution_profile_id
  content_profile_id
  crash_profile_id
```

起動時にOS build、CPU architecture、D3D feature、SM、driver、memory budget、display、audio／input availabilityを`PlatformCapabilitySignature`へ記録する。Hard requirement不足は起動を止め、quality fallbackでTarget不足を隠さない。

OS Support期間と累積更新要件はToolchain ownerのPlatform policy lockを参照し、本書は固定build、取得先、更新周期を再定義しない。

## 4. Process model

| Process | 権限／役割 |
|---|---|
| `mirakan_editor.exe` | UI、Authoring Command Gateway、Project／Workspace |
| `mirakan_game_host.exe` | Preview Runtime、Project C++、Renderer、Audio |
| `mirakan_worker_host.exe` | Asset／Shader／Contract等のJob entry |
| `mirakan_ai_orchestrator.exe` | Model Provider、MCP、AI Task |
| `mirakan_package_service.exe` | Package assembly／inspection。署名keyなし |
| `mirakan_crash_collector.exe` | optional out-of-process dump／metadata collection |

Editor起動child tree全体は`EditorSessionJob`でkill-on-job-closeとchild process policyを適用するが、AI／Compiler／WorkerまでEditor memoryへ誤計上しないため、このroot Jobへaggregate memory limitを設定しない。EditorHost、GameHost、Source／Asset／Shader Worker、AI Orchestrator、Package Service、Crash Collectorは[Runtime performance／capacity](../04-runtime/performance-capacity.md)のprocess group、allocator tag、working-set telemetry、sandbox capacityへ個別にchargeする。Workerはsuspended起動後にtask別nested Jobへ割り当ててからresumeし、GameHost終了時にProject C++ job、GPU、Audioをjoinし、timeout時はProcess単位で終了する。

Credential、Signing、UploadはBuild Processへ渡さない。Named pipeはUser SID ACL、length prefix、JSON-RPC、message 8 MiB上限、nonce／protocol versionを持つ。

## 5. Window、Display、Lifecycle

### 5.1 Window

- Win32 top-level HWNDをPlatform Adapterが所有する。
- EditorはMain／floating Windowだけにtop-level HWNDを使用し、通常Widgetごとにchild HWNDを作らない。
- DPI Awareness ContextはPer-Monitor V2をProcess起動直後に固定する。
- logical size、physical pixel size、DPI、monitor、refresh、color space、surface generationを分離する。
- resize／DPI／monitor移動はeventへ正規化し、Render surfaceをgeneration付きで再作成する。
- Alt+Enter、borderless、windowedはtyped Display Commandで切り替える。
- Exclusive fullscreenをC1で使用しない。
- minimized、occluded、display offではPresent cadenceを停止／低下させるがSimulation policyを明示する。

### 5.2 Lifecycle state

```text
Starting -> ForegroundActive <-> ForegroundInactive
-> SuspendedForSystemEvent
-> StopRequested -> Stopped
```

Windows desktopではMobile suspendを偽装しない。Session end、shutdown、display／device change、power notification、focus、minimizeを別eventとして正規化する。`WM_CLOSE`はStop Requestを生成し、UI threadで長いSave／Buildを同期実行しない。

Critical Project draft recoveryはEditorのAuthoring規約、Game SaveはSave serviceが原子的に行う。OS shutdown通知を唯一の保存機会にしない。

## 6. Platform Port mapping

| Port | Windows Adapter |
|---|---|
| `IApplicationSurface` | primary Win32 Window＋DXGI surface generation |
| `IUiWindowService` | Editor top-level／owned Window、cursor、monitor、work area、non-client frame |
| `ILifecycleService` | Window／session／power event normalization |
| `IInputDeviceHub` | GameInput device／reading＋Win32 pointer bridge |
| `IUiRenderBackend` | `MirakanUiDrawPacketV1`からD3D12 UI passへの変換 |
| `IUiTextBackend` | DirectWrite text layout、system Font fallback、glyph analysis |
| `ITextInputService` | TSF／IME composition、selection、candidate geometry |
| `IUiAccessibilityBridge` | UI Automation fragment root、provider、control pattern、event |
| `IEditorDataTransferService` | Unicode clipboard、OLE `IDataObject`／drag and drop |
| `IPlatformDialogService` | File／Folder picker、credential／permission、fatal recovery |
| `IAudioDevice` | XAudio2 device／graph／route |
| `IHapticsService` | GameInput haptic／force feedback subset |
| `IContentDeliveryService` | installed package／external platform layout／C2 content group |
| `IDeviceConditionService` | memory budget、power、display、device／driver |
| `IPlatformCrypto` | CNG RNG、hash／signature verification、credential protection |
| `IUserDataStore` | Known Folder＋atomic file protocol |

Game／Project codeはPlatform Portを直接呼ばず、Engine Capabilityのtyped command／snapshotを使う。

## 7. FilesystemとUser data

### 7.1 Root

| Data | Root／規則 |
|---|---|
| Installed Engine／Game | package／distribution layout。Runtime read-only |
| Project source | Userが選択したdirectory。Editorだけread／write |
| Build／Cache | Project `build`またはUser cache。Sourceと分離 |
| Editor config | `FOLDERID_LocalAppData`下のEngine／user |
| Game Save | `FOLDERID_SavedGames`下のPublisher／Game。利用不可時はLocalAppData fallbackをReceiptへ記録 |
| Log／Crash | LocalAppData下、rotation／retention |
| Screenshot／Export | Userが選択した既知FolderまたはFile picker result |

OS pathはUTF-16 Win32境界で扱い、Engine内はvalid NFC UTF-8 logical pathへ変換する。Lossy conversionをしない。Reserved name、reparse point、junction、symlinkをbroker境界で検査し、許可root外へ解決するpathを拒否する。

### 7.2 Atomic save

Saveは同一volumeのtemporary fileへ完全write、flush、hash検証後、`ReplaceFileW`相当の原子的置換を使う。最低一つのprevious backupを保持する。Save headerはProject／Build／schema／content package set／checksumを持つ。

Installed package、Asset Cache、Project sourceへGame Saveを書かない。Crash時の部分Saveを正規slotとして表示しない。

## 8. Windows package

### 8.1 `windows_msix_v1`

full-trust Win32 applicationをMSIXへpackageする。

```text
MSIX
├─ AppxManifest.xml
├─ mirakan_game.exe
├─ D3D12/D3D12Core.dll
├─ content/*.mirakanpack
├─ <locked_runtime_dependency>.dll
├─ assets/
├─ licenses/
└─ [MSIX-generated block map／signature]
```

- `AppxManifest.xml`、`D3D12/D3D12Core.dll`、Third-party runtime basename、MSIX生成物はTool／Platform所有名として原表記を保持する。それ以外のFirst-party artifactとDirectoryはEngine命名正本のlowercase `snake_case`に従う。
- Package identity、publisher、version、architecture、minimum OS、capabilityをTarget／Distribution Profileから生成する。
- default capabilityは0で、実際に必要な宣言だけを[AI Security／Approval](../01-governance/ai-security-approval.md)のRelease decision refから生成する。
- elevation、driver、service、arbitrary startup taskを要求するGameをC1 packageで拒否する。
- Agility DLL version／hash、executable import、Content root hash、source／debug／compiler非混入をinspectionする。
- MSIX packageはinstall前に署名が必要で、Store外はTarget環境が信頼する証明書を用いる。
- Store提出用packageとdirect／enterprise署名packageのIdentity／Signing Receiptを混在させない。

Editor本体もC2 Production Distributionではfull-trust MSIXを推奨する。Phase 0／C1の開発中は署名済みinternal MSIXとportable CI artifactを分け、portable artifactを一般配布物と表現しない。

### 8.2 `windows_managed_layout_v1`

Steam等の外部client向けにはPlatform非依存のDirectory layout、content manifest、launch manifest、redistributable noticeを出す。Install／update／rollback／entitlementは配布clientが所有し、Miraikanai Game内で独自updaterを起動しない。

Layoutの実行fileとDLLへAuthenticode署名を行い、Package Receiptに全file SHA-256を記録する。Platform固有SDK統合は別RepositoryのEngine製品変更であり、authorization classはGovernance ownerを参照する。Game制作Profileから実行または提案せず、NativeGameModuleへ直接linkしない。Baselineに未提供なら`capability_unavailable`とする。

### 8.3 Development package

`windows_development_layout_v1`はlocal Play／CI専用で、PDB、source map、D3D12 Debug Layer／DRED、validation layer、loose Catalog、Debugging規約のD2／D3 instrumentを含められる。各symbol／captureはBuild Receipt、Module hash、Session IDへ関連付け、絶対source pathやcredentialを共有Bundleへ既定で含めない。Shipping manifestと別ID／directoryを使い、Shipping signing／Store uploadへ昇格できない。

## 9. Build、Signing、Publication

```text
Commit済みSource
-> clean Build
-> Target Cook
-> unsigned layout／MSIX content
-> Package Inspection
-> Governance Release decision reference
-> isolated Signing
-> signed Package Inspection
-> Distribution-specific Upload
-> Receipt verification
```

- `clean Build`は[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)の`windows_cmake_ninja_multi_v1` Driver mappingを使用する。IDEからも同じchecked-in entryを使い、別Product Build経路を生成しない。
- `Development`、`Profile`、`Shipping`、`ASan`は同じGeneratorの明示Configurationであり、C++ Profile、Toolchain hash、Configurationが異なるBuild tree／BMIを共有しない。
- Editor、AI、CIはBuild Gatewayを呼び、`ninja`または`cmake -G`を直接Product Build入口として公開しない。
- Build Workerはprivate signing keyを持たない。
- Signing Serviceは固定Package artifact、identity、Governance ownerのRelease decision refだけを受け取る。
- Authenticode／MSIXはSHA-256を使用し、trusted timestamp policyをDistribution Profileへ固定する。
- private keyはPlatform credential store、HSM、またはGovernance ownerが選択したSigning serviceからexportしない。
- `signtool verify`、MSIX package validation、malware scan、SBOM／notice、package executable-content scanを行う。
- Store／Platform upload credentialはSigning keyとSource accessを持たない別Serviceに置く。

Certificate subject、thumbprint、timestamp URL、Store identityはProject bootstrapで実在値を入力し、secretではないreferenceだけをrepositoryへ保存する。

## 10. Update、Patch、DLC

| 更新対象 | C1／C2経路 |
|---|---|
| EXE／DLL／offline shader | Store、MSIX App Installer、または外部配布clientのApplication updateだけ |
| Base `.mirakanpack` | Application update |
| Content Patch／DLC | Asset規約のC2 signed Package＋Distribution service |
| Save schema | Application update同梱のoffline migrator |

C1は自己更新を実装しない。C2でdirect MSIX配布を有効化する場合はMicrosoft App Installer／MSIX update modelを使い、Gameが任意URLからEXEをdownload／executeする独自updaterを作らない。

Patch apply中は旧Packageを上書きせず、完全検証後にmount setを切り替える。rollback可能version、minimum app version、content entitlementをCatalogへ明示する。codeを`.mirakanpack`／DLCへ混入しない。

## 11. Crash、Hang、Diagnostics

### 11.1 C1

- Development／ProfileではWER LocalDumpsまたはTask Manager／ProcDumpによるdump取得手順を提供する。
- Shippingでは`mirakan_crash_collector.exe`を任意で同梱し、out-of-processでminidumpとtyped crash metadataをLocalAppDataへ保存する。
- in-process exception handlerで複雑なallocation、lock、network upload、Project Saveを行わない。
- Dump取得失敗でProcess終了を阻止しない。
- Hang watchdogはGame simulation／render／window heartbeatを監視するが、threadを強制再開して継続しない。

`CrashEnvelopeV1`はapp／Engine／Module／Package hash、OS／driver、Target Capability、last phase、bounded log／breadcrumb、consent stateを持つ。Debugging規約のSession ID、last complete debug chunk、gap、Replay slice、DRED breadcrumbs／page fault、dump／symbol descriptorを参照できるが、同じpayloadを重複保存しない。Project source、AI prompt、credential、full path、user name、Save本文を既定で含めない。

### 11.2 Upload

C1は自動uploadしない。Userがfileを確認して明示exportできる。online uploadはactivated Capabilityと[AI Security／Approval](../01-governance/ai-security-approval.md)のdecision refがある場合だけ有効化し、本書は次のWindows privacy／transport conformanceを検査する。

- 初回説明とopt-in
- 送信前のdata category表示
- TLS、endpoint allowlist、retry cap、delete
- retention、privacy policy、地域
- symbol access分離
- credential／PII redaction
- offline queue size／age上限

Crash consentをAnalytics、AI Provider、Marketing consentとまとめない。

## 12. Security

- Shipping binaryはDEP、ASLR、CFG、CET互換、stack protection、signed executableを必須とする。
- DLL searchはApplication directoryと明示pathへ限定し、current directory／PATH依存loadを禁止する。
- Development DLL loadもabsolute path、hash、manifest、publisher／artifact Receiptを検査する。
- COM、URL scheme、file association、firewall rule、protocol handlerをdefaultで登録しない。
- Package／installerは管理者権限を要求しない。
- Log／dump／Save directoryにACLを適用し、他User書込を許可しない。
- Untrusted Projectを開くだけでC++／shader／importerをEditor Process内実行しない。

## 13. PerformanceとReliability

[Runtime performance／capacity](../04-runtime/performance-capacity.md)のWindows reference environment、共通CPU／GPU／frame envelope、measurement windowをそのまま使用し、本書では複写しない。

- warm-cache起動→操作可能frame P95 5秒、hard 8秒
- Scene reload P95 2秒、hard 3秒
- package mount／Catalog validationを起動時間へ計上する
- window input→submit、audio callback、resize、Alt+Tab、device lossを10分soakへ含める
- package install／upgrade／rollback／uninstall後にUser Saveを保持し、Program filesを残さない
- no-network環境でCore Game／Editorのlocal機能が起動する

## 14. Failure policy

| Failure | 結果 |
|---|---|
| Minimum OS／D3D feature不足 | 起動拒否、必要条件を表示 |
| Package／signature／hash不正 | install／mount／launch拒否 |
| Missing Agility runtime | 起動拒否。System D3D12へ黙ってfallbackしない |
| User data write失敗 | Save成功を表示せずprevious slot維持 |
| Distribution update失敗 | 配布channelが旧versionを維持／rollback |
| Content Patch失敗 | 旧mount set維持 |
| GameHost crash | Editor継続、local crash record |
| Editor crash | Authoring Recovery Diffを次回提示 |
| Device removed | Rendering規約のrecovery、再失敗でGameHost fault終了 |
| Signing／timestamp不合格 | Publication block |

## 15. TestとDefinition of Done

- Toolchain ownerが固定したminimum／fully patched Windows baseline test
- OS／D3D／SM／Enhanced Barrier／driver Capability rejection
- 100／125／150／200% DPI、multi-monitor、resize、Alt+Tab、sleep／session end
- GameInput／DirectWrite／TSF／UI Automation／OLE／XAudio2／D3D12 Adapter conformance
- Project、Save、Cache、Package rootのpath／symlink／ACL／disk full
- MSIX install、launch、upgrade、rollback、uninstall、signature、Store validation
- managed layoutのfile hash、external client smoke、no built-in updater
- EXE／DLL／Content executable scan、DLL hijack、untrusted Project
- WER／Crash Collectorのcrash／hang／dump failure／privacy redaction
- Development／ProfileのPDB／source map／DRED／Debug Session関連付けと、Shipping packageからvalidation layer、IDE attach、raw trace、source path、credentialを除外するscan
- signed Packageをclean VMへinstallし、2D／3D縦切りをplay／save／reload

C1完了条件は、Windows Editorから同一Project revisionをDevelopment Play、signed internal MSIX、managed layoutへBuildし、clean machineでinstall／play／save／crash回収／uninstallでき、Source、credential、debug toolをShippingへ含めないことである。

## 16. 外部依存境界

Windows OS、SDK、graphics runtime、Build／package／signing toolのexact baseline、取得元、integrity、license、外部根拠は[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)だけが所有する。本書はそれらをWindows Adapter、package、qualificationへ写像するEngine contractだけを所有する。
