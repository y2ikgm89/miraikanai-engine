# Miraikanai Engine Windows Platform／Distribution規約

- 文書版: 1.3
- 作成日: 2026-07-19
- 対象: Windows Editor／Game、Process、Window、Platform Port、Filesystem、Package、Signing、Update、Crash
- 状態: プロジェクト公式の規範設計レビュー版
- 基盤規約: [Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)
- Engine命名正本: [Miraikanai Engine AI可読命名・技術識別子規約](./2026-07-20-ai-readable-engine-naming-convention-design.md)
- C++言語・Build規約: [Miraikanai Engine C++23・Named Modules・`import std`移行規約](./2026-07-20-cpp23-modules-import-std-transition-design.md)
- Runtime規約: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- Native Game規約: [Miraikanai Engine NativeGameModuleアーキテクチャ規約](./2026-07-19-native-game-module-architecture-design.md)
- Renderer規約: [Miraikanai Engine Rendering／Render Graphアーキテクチャ規約](./2026-07-19-rendering-render-graph-architecture-design.md)
- Asset規約: [Miraikanai Engine Asset Pipeline／Content Package規約](./2026-07-19-asset-pipeline-content-packaging-design.md)
- Editor規約: [Miraikanai Engine Editor／Workspace／UX規約](./2026-07-19-editor-workspace-ux-design.md)
- Editor UI Framework規約: [Miraikanai Engine 独自Editor UI Framework／Shellアーキテクチャ規約](./2026-07-20-editor-ui-framework-architecture-design.md)
- Debugging規約: [Miraikanai Engine AI可読Debugging／Observability／Replayアーキテクチャ規約](./2026-07-20-ai-readable-debugging-observability-replay-architecture-design.md)
- Player I/O規約: [Input](./2026-07-19-input-action-device-architecture-design.md)／[UI・Text](./2026-07-19-ui-text-localization-accessibility-design.md)／[Audio](./2026-07-19-audio-mixer-spatial-architecture-design.md)
- Mobile規約: [Miraikanai Engine モバイルPlatformアーキテクチャ規約](./2026-07-19-mobile-platform-architecture-design.md)

## 1. 結論

`windows_desktop_v1`はWindows 11 25H2、OS build family 26200以上、x86-64、D3D12、Shader Model 6.6を正式Targetとする。Editorも最初のGame RuntimeもこのTargetから実装する。

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
| C++ toolchain、Agility、memory、security baseline | 基盤規約 |
| Android／Apple | Mobile規約 |

Windows 10、Windows 11 24H2 Home／Pro、ARM64、Windows on ARM emulation、Xbox／GDK、UWP sandbox、Windows Server、Wine／ProtonをC1正式Targetに含めない。将来追加は別Target Profileと実機／package／performance gateを必要とする。

## 3. Target Profile

```text
WindowsDesktopTargetProfileV1
  profile_id = windows_desktop_v1
  minimum_os_build = 26200
  architecture = x86_64
  executable_model = win32_full_trust
  graphics = d3d12_sm66
  enhanced_barriers_required = true
  dpi_awareness = per_monitor_v2
  input_backend = gameinput
  audio_backend = xaudio2
  text_input_backend = tsf_win32
  distribution_profile_id
  content_profile_id
  crash_profile_id
```

起動時にOS build、CPU architecture、D3D feature、SM、driver、memory budget、display、audio／input availabilityを`PlatformCapabilitySignature`へ記録する。Hard requirement不足は起動を止め、quality fallbackでTarget不足を隠さない。

OS Support期間と累積更新要件は`platform_policy.lock.json`へ取得日、Microsoft URL、最小build、期限を固定し、四半期またはRelease前にEvidence Jobで更新する。

## 4. Process model

| Process | 権限／役割 |
|---|---|
| `mirakan_editor.exe` | UI、Authoring Command Gateway、Project／Workspace |
| `mirakan_game_host.exe` | Preview Runtime、Project C++、Renderer、Audio |
| `mirakan_worker_host.exe` | Asset／Shader／Contract等のJob entry |
| `mirakan_ai_orchestrator.exe` | Model Provider、MCP、AI Task |
| `mirakan_package_service.exe` | Package assembly／inspection。署名keyなし |
| `mirakan_crash_collector.exe` | optional out-of-process dump／metadata collection |

Editor起動child tree全体は`EditorSessionJob`でkill-on-job-closeとchild process policyを適用するが、AI／Compiler／WorkerまでEditor memoryへ誤計上しないため、このroot Jobへaggregate memory limitを設定しない。EditorHost＋同時に一つだけのGameHostはRuntime規約のallocator tagとProcess working-set telemetryでaggregate 4 GiB hard gateを適用する。Source／Asset／Shader Workerはsuspended起動後にtask別nested Jobへ割り当ててからresumeし、Source sandbox Profileのcommit memory／CPU／process数上限をOSで強制する。AI Orchestrator、Package Service、Crash CollectorもProcess別budgetを持ち、Editor 4 GiBへ含めない。GameHost終了時にProject C++ job、GPU、Audioをjoinし、timeout時はProcess単位で終了する。

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
- default capabilityは0で、実際に必要な宣言だけをRelease ownerが承認する。
- elevation、driver、service、arbitrary startup taskを要求するGameをC1 packageで拒否する。
- Agility DLL version／hash、executable import、Content root hash、source／debug／compiler非混入をinspectionする。
- MSIX packageはinstall前に署名が必要で、Store外はTarget環境が信頼する証明書を用いる。
- Store提出用packageとdirect／enterprise署名packageのIdentity／Signing Receiptを混在させない。

Editor本体もC2 Production Distributionではfull-trust MSIXを推奨する。Phase 0／C1の開発中は署名済みinternal MSIXとportable CI artifactを分け、portable artifactを一般配布物と表現しない。

### 8.2 `windows_managed_layout_v1`

Steam等の外部client向けにはPlatform非依存のDirectory layout、content manifest、launch manifest、redistributable noticeを出す。Install／update／rollback／entitlementは配布clientが所有し、Miraikanai Game内で独自updaterを起動しない。

Layoutの実行fileとDLLへAuthenticode署名を行い、Package Receiptに全file SHA-256を記録する。Platform固有SDKを統合する場合はEngine Platform AdapterのR4変更であり、NativeGameModuleへ直接linkしない。

### 8.3 Development package

`windows_development_layout_v1`はlocal Play／CI専用で、PDB、source map、D3D12 Debug Layer／DRED、validation layer、loose Catalog、Debugging規約のD2／D3 instrumentを含められる。各symbol／captureはBuild Receipt、Module hash、Session IDへ関連付け、絶対source pathやcredentialを共有Bundleへ既定で含めない。Shipping manifestと別ID／directoryを使い、Shipping signing／Store uploadへ昇格できない。

## 9. Build、Signing、Publication

```text
Commit済みSource
-> clean Build
-> Target Cook
-> unsigned layout／MSIX content
-> Package Inspection
-> Release Authorization
-> isolated Signing
-> signed Package Inspection
-> Distribution-specific Upload
-> Receipt verification
```

- `clean Build`は`windows_cmake_ninja_multi_v1`、checked-in CMake Preset、`Ninja Multi-Config`だけを使用する。Visual Studio IDEはCMake Presetを利用できるが、Visual Studio／NMake／MinGW／MSYS／Unix Makefilesによる別Product Buildを生成しない。
- `Development`、`Profile`、`Shipping`、`ASan`は同じGeneratorの明示Configurationであり、C++ Profile、Toolchain hash、Configurationが異なるBuild tree／BMIを共有しない。
- Editor、AI、CIはBuild Gatewayを呼び、`ninja`または`cmake -G`を直接Product Build入口として公開しない。
- Build Workerはprivate signing keyを持たない。
- Signing Serviceは固定Package artifact、identity、Release Authorizationだけを受け取る。
- Authenticode／MSIXはSHA-256を使用し、trusted timestamp policyをDistribution Profileへ固定する。
- private keyはWindows certificate store、HSM、または承認済みSigning serviceからexportしない。
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

C1は自動uploadしない。Userがfileを確認して明示exportできる。C2 online uploadは次を満たす別Capability承認後だけ有効化する。

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

Windows Reference hardware、2 GiB CPU、5.5 GiB GPU、1080p60、P95 14 msは基盤／Runtime規約を正本とする。

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

- Windows 11 25H2 build 26200系のminimum／fully patched test
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

## 16. 一次資料

- [Windows 11 Release Information](https://learn.microsoft.com/en-us/windows/release-health/windows11-release-information)
- [Windows Packaging Overview](https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/packaging/)
- [What is MSIX?](https://learn.microsoft.com/en-us/windows/msix/overview)
- [Choose a Windows Distribution Path](https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/choose-distribution-path)
- [Prepare a Desktop App for MSIX](https://learn.microsoft.com/en-us/windows/msix/desktop/desktop-to-uwp-prepare)
- [Windows Job Objects](https://learn.microsoft.com/en-us/windows/win32/procthread/job-objects)
- [Collecting User-Mode Dumps](https://learn.microsoft.com/en-us/windows/win32/wer/collecting-user-mode-dumps)
- [Application Recovery and Restart](https://learn.microsoft.com/en-us/windows/win32/recovery/using-application-recovery-and-restart)

MicrosoftのPackaging推奨はDistribution判断の根拠として使う。Miraikanai固有のTarget Profile、Process分離、Content Package、Signing authority、Crash privacyは本書で規範化する。
