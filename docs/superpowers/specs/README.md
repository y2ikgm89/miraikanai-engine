# Miraikanai Engine 設計文書Index

- 最終更新日: 2026-07-20
- 状態: Subsystem別正式仕様を統合した設計レビュー用Index。2026-07-20内部整合レビュー済み、ユーザー承認待ち

## 0. 統合計画サマリー

Miraikanai Engineは、AIがEngine内部objectを直接操作するゲームエンジンではない。人間またはAIが「作りたいゲーム状態」と「編集Command」を型付き構造化データとして提案し、C++ `AuthoringCommandGateway`がSchema、semantic、Capability、Policy、budget、base revisionを検証した後だけ、一つの`ProjectRevision`として原子的にCommitする独自AIネイティブゲームエンジンである。

| 主題 | 確定した計画 |
|---|---|
| Product形態 | Editor制作型を先に完成させる。制限付きRuntime生成はPhase 9で別Threat ModelとServer-authoritative Gatewayを設けた後だけ扱う |
| 制作レベル | Level 0自然言語、Level 1 Scene／Inspector／Graph、Level 2 GameplayDefinition、Level 3 Project C++を同一Project形式で往復できる |
| 初回制作 | 大まかなPromptをRequirementへ分解し、Blocking／High Impact不足だけを質問し、Game Brief承認後に薄いゲーム全体と深い代表部分を作る |
| Game実装 | 実行CodeはC++23、頻繁に調整する内容は型付き`GameplayDefinition`。offline CookしたPackageをC++ evaluatorが実行する |
| Script | Luau、Lua、C# Game runtime、汎用Script VM、bytecode interpreter、JIT、Game向けFFIは採用しない |
| 実装方式選択 | AIはSystem単位でGameplayDefinition、C++、または型付きCapability境界での併用を選ぶ。選択理由、budget、benchmarkをDecision Ledgerへ記録する |
| Engine／Editor | Engine、EditorHost、GameHost、Tool、NativeGameModuleのFirst-party CPU codeはC++23。EditorはC++ Projection Editorで、状態の正本ではない |
| C++移行 | CX0のHeader bootstrapから、Named Modules＋`import std`を使うCX3へ一方向移行する。C++26はShippingに使わずreadiness CIだけを持つ |
| Build | CMakeがFirst-party C++ Build定義の正本、Ninjaが高速なC++実行器、独自Build GatewayがEditor／AI／CIの唯一の入口。`build.ninja`は生成物であり公開APIにしない |
| Platform Build | WindowsはNinja Multi-Config、AndroidはGradle→CMake→Single-Config Ninja、Appleはportable C++ archiveをNinja、App shell／resource／最終link／archive／署名をXcodeが所有する |
| AI接続 | 内蔵AIはProvider API、外部Codex／Claude等はlocal MCP、Host Pluginは任意の補助UX。どの接続も検証・Commit権限を持たない |
| Editor UI／UX | C++23の独自`MiraUI Core`＋`MiraEditor Shell`。Retained UIと限定typed Immediate Canvasを併用し、Scene／Canvas、Hierarchy／Outliner、Inspector、Asset、Source、Console、AI Partnerをdock、resize、floating、入替、multi-monitor、複数Workspace保存できる |
| 初心者UX | `AI Creator` Workspaceを同じEditor内に用意し、AI Partnerを常設可能にする。Production／Debug／Art／Level Design等のWorkspaceへ切替可能 |
| Graphics | Windows Direct3D 12、Android Vulkan、Apple Metal。Engine-owned Render Graph、Material／Shader IR、Target別offline shader cookを使う |
| 表現 | 2D、3D、`realistic_basic`、Realistic advanced、Toon、独自`pixel_diorama`を段階実装し、AI Visual Style ResolverがCapabilityとbudget内で選択する |
| Particle／VFX | 単一の型付きVFX Asset／Graph IRから2D／3D・CPU／GPU専用Artifactをoffline生成する。初心者Stack、上級者Graph、AIは同じSourceを編集し、VFX結果をGameplayへ逆入力しない |
| 必須Subsystem | Renderer、Asset、Collision、Physics、Navigation、Animation、Input、UI／Text／Localization／Accessibility、Audio、Particle／VFX、Lighting、Sky／Atmosphere／Fog／Cloudを正式仕様で所有する |
| 外部Library | Engineの正規data model、Capability、validation、lifecycle、serialization、Editor UXは独自所有する。OS、Graphics API、Box2D、Jolt、Recast、GPU allocator等は検証してAdapter内へ隔離し、再発明しない |
| Physics Platform | 独自World／Body／Joint／Character／Command／Save／AI契約をC++23で所有し、2DはBox2D 3.1.1、3DはJolt 5.6.0をprivate kernel候補としてQualification後にProduction昇格する。Game／AIへVendor APIを公開しない |
| Memory／Pointer | RAII、明示所有権、generation handle、phase／epoch lease、memory domain、GPU deferred destruction、Target別budgetを正規規約にする |
| 対象Platform | Windows Editor／Gameを先行し、Android、iOS／iPadOSを順に追加する。Mobile Editor、Linux製品Target、multiplayer、XRは初期対象外 |
| MVP | MVP-AはAIで作る2D top-down action、MVP-Bは3D compact third-person action arena。Android／Apple vertical sliceはその後に行う |
| AIによるEngine保守 | Game制作AIに加えてEngine本体の実装・保守AIを対象にするが、隔離Source Worker、Risk class、Review、Promotion、署名分離を満たすまで高権限操作を公開しない |

### 0.1 要求から正式仕様への対応

| これまでに確認した要求 | 決定権を持つ正式仕様 |
|---|---|
| AI設計図、追加質問、共同／詳細／高水準設計、Editor制作型 | AIネイティブ設計計画書 |
| C++のみの実行層、構造化ゲームデータ、Luau不採用 | C++実行コード・構造化ゲームデータ規約 |
| Memory、Pointer、Architecture、Naming、Directory、Dependency、Build | 基盤アーキテクチャ規約 |
| C++23、C++26 readiness、Modules、`import std`、Ninja | C++23・Named Modules移行規約 |
| AI／手動編集の共通状態、Undo、Recovery、Transaction | Authoring Model／Project State規約 |
| AIが理解できる型、Operation、Capability、Schema、Codegen | 実行可能契約規約 |
| Renderer／Asset／Editor／Input／UI／Audio／2D／3D | 各Subsystem正式仕様、独自Editor UI Framework規約、2D／3D機能計画 |
| Collision、Physics Engine、Joint、Character、Navmesh、Animation | Collision規約、独自Physics Platform規約、Physics／Navigation／Animation連携規約 |
| Light、Fog、Cloud、Sky、Atmosphere、Shader、Material、Toon／Realistic／Pixel Diorama | Rendering規約と2D／3D機能計画 |
| 2D／3D Particle、CPU／GPU VFX、VFX Graph、AI／Editor編集 | 独自Particle／VFX Platform規約と2D／3D機能計画 |
| Android、iOS／iPadOS、Build、Store、実機budget | モバイルPlatform規約 |
| Codex／Claude API、MCP、CLI／Desktop、Plugin、Engine保守AI | AI実装・保守ガバナンス規約 |
| Test、Eval、Provenance、SBOM、最新情報更新 | AI検証・評価・来歴規約 |

### 0.2 2026-07-20内部整合レビュー

- `Miraikanai Engine`の名称へ統一され、旧綴りは公式Review setに残っていない。
- C++23、GameplayDefinition、Script VM不採用、Target、Build Driver、Editor、AI権限、Runtime phaseの主要判断に矛盾する公式経路は見つかっていない。
- 未記入placeholder、実装者任せの保留、暫定Defaultを正式要件として残していない。外部Artifact取得後に確定するhash等は、Owner、取得手順、失敗条件を持つPhase 0作業として区別している。
- NinjaはBuild architecture全体ではなくC++ Build executorであること、Editor統合はCMake File APIとEngine-owned Receiptを使うこと、Phase 0で増分正当性と性能を実測することを今回の見直しで明文化した。
- Editor UIはDear ImGui等の置換前提prototypeを持たず、独自`MiraUI Core`、`MiraEditor Shell`、DirectWrite／TSF／UIA／OLE Adapter、AI Semantic Interfaceを正式経路にした。
- Physicsは「全Solver自作」でも「Vendor API直接公開」でもなく、独自の型付きPlatform契約へBox2D／Joltを隔離する方式に確定した。World／Solver全値、Joint、Character Motor、AI、Save／Replay、Qualificationの決定権を専用仕様へ分離し、Navigation／Animation連携文書との重複を解消した。
- Particle／VFXは4つの別Authoring Systemや万能Runtime Graphではなく、単一Source／型付きIRから2D／3D・CPU／GPU専用Artifactへspecializeする方式に確定した。C1 CPU、C2 GPU、Gameplay分離、SoA、counter RNG、Render Graph、AI／Editor、Budget、Qualificationの決定権を専用仕様へ分離した。
- 本レビューは文書内部の整合確認であり、Engine実装完了または技術成立の実測証明ではない。実装開始には本Review setのユーザー承認と、別のPhase 0実装計画書の承認が必要である。

### 0.3 見直し結論と未実証項目

設計の粒度は、Phase 0実装計画をfile、target、contract、fixture、testへ分解できる段階に達している。一方、次は設計上の未決定ではなく、実artifactで合否を確定する未実証項目である。

| 未実証項目 | 証明するPhase／Gate | 不合格時 |
|---|---|---|
| C++23 Modules／`import std`の全Target Shipping成立 | CX1 Probe、CX2 Candidate、CX3 Cutover Gate | Header bootstrapまたはCX2に留まり、Shippingへ昇格しない |
| Ninjaの増分正当性、no-op性能、memory、中断復旧 | Phase 0 `VerificationReceiptV1` gate `mira.build.ninja_adoption.v1` | Makeへfallbackせず、DAG／dependency／poolを修正して再測定 |
| AppleのNinja C++ archive＋Xcode App shell境界 | Phase 0 C ABI fixture、Mobile Phaseの実archive | Apple CX0または非Promotion Probeに留める |
| AuthoringCommandGatewayとContract compilerの安全性 | Phase 0／1 conformance、negative、crash recovery | Editor／AIのwrite機能を有効化しない |
| 独自MiraUI Core／MiraEditor Shell、D3D12 UI pass、Docking、DirectWrite／TSF／UIA | Phase 2 dependency negative、UI Automation、DPI、IME、keyboard、screen reader、device-loss fixture | Technology Preview配布へ進めない |
| Box2D 3.1.1／Jolt 5.6.0のTarget別Kernel Qualification、Adapter、Joint、Character、Save／Replay | Phase 0 source／build lock、Phase 3の2D C1、Phase 6の3D C1、Mobile Phase実機Gate | 当該Dimension／TargetのPhysics CapabilityをProduction表示しない |
| VFX Graph Compiler、2D／3D CPU specialization、D3D12／Vulkan／Metal GPU Artifact、indirect draw、device recovery | C0 Compiler fixture、Phase 3／6 CPU Reference Effect、Phase 8 GPU／Mobile実機Gate | 該当CPU／GPU VFX CapabilityをProduction表示しない |
| 2D／3Dのframe、memory、visual、physics／navigation連携 | Phase 3／6 reference scene、10分soak、golden capture | 対応CapabilityをC1へ昇格しない |
| Android／AppleのStore、実機、thermal、package | Phase 7 physical device／package／privacy Gate | 対象Platformを配布Targetへ昇格しない |
| AI生成品質、安全性、Provider更新耐性 | MVP-A以降のTask Eval、holdout、adversarial、Promotion Gate | 対象Risk classまたはProviderを有効化しない |

したがって次の正規作業はEngine全体の一括実装ではなく、Phase 0だけを対象とする実装計画書の作成である。2D、3D、Mobile、Production機能は各Milestone Gateの入力が揃ってから別実装計画へ分解する。

## 1. 読む順序

Miraikanai Engineの公式Review setは次の25文書である。上位のProduct判断から共通契約、Subsystem固有契約、検証規約の順に読む。

1. [AIネイティブ独自ゲームエンジン 設計計画書](./2026-07-18-ai-native-game-engine-authoring-design.md)
2. [Miraikanai Engine C++実行コード・構造化ゲームデータ規約](./2026-07-19-cpp-structured-game-data-design.md)
3. [Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)
4. [Miraikanai Engine C++23・Named Modules・`import std`移行規約](./2026-07-20-cpp23-modules-import-std-transition-design.md)
5. [Miraikanai Engine Authoring Model／Project State規約](./2026-07-19-authoring-model-project-state-design.md)
6. [Miraikanai Engine 実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)
7. [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
8. [Miraikanai Engine 2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)
9. [Miraikanai Engine NativeGameModuleアーキテクチャ規約](./2026-07-19-native-game-module-architecture-design.md)
10. [Miraikanai Engine Asset Pipeline／Content Package規約](./2026-07-19-asset-pipeline-content-packaging-design.md)
11. [Miraikanai Engine Rendering／Render Graphアーキテクチャ規約](./2026-07-19-rendering-render-graph-architecture-design.md)
12. [Miraikanai Engine 独自Particle／VFX Platformアーキテクチャ規約](./2026-07-20-particle-vfx-architecture-design.md)
13. [Miraikanai Engine Collision／Colliderアーキテクチャ規約](./2026-07-19-collision-collider-architecture-design.md)
14. [Miraikanai Engine 独自Physics Platform／Dynamicsアーキテクチャ規約](./2026-07-20-physics-engine-architecture-design.md)
15. [Miraikanai Engine Physics／Navigation／Animation連携規約](./2026-07-19-physics-navigation-animation-architecture-design.md)
16. [Miraikanai Engine Input／Action／Device規約](./2026-07-19-input-action-device-architecture-design.md)
17. [Miraikanai Engine UI／Text／Localization／Accessibility規約](./2026-07-19-ui-text-localization-accessibility-design.md)
18. [Miraikanai Engine 独自Editor UI Framework／Shellアーキテクチャ規約](./2026-07-20-editor-ui-framework-architecture-design.md)
19. [Miraikanai Engine Audio／Mixer／Spatial規約](./2026-07-19-audio-mixer-spatial-architecture-design.md)
20. [Miraikanai Engine Editor／Workspace／UX規約](./2026-07-19-editor-workspace-ux-design.md)
21. [Miraikanai Engine Windows Platform／Distribution規約](./2026-07-19-windows-platform-distribution-design.md)
22. [Miraikanai Engine モバイルPlatformアーキテクチャ規約](./2026-07-19-mobile-platform-architecture-design.md)
23. [Miraikanai Engine Domain Pack／将来Capability規約](./2026-07-19-domain-pack-future-capability-roadmap.md)
24. [Miraikanai Engine AI実装・保守ガバナンス規約](./2026-07-19-ai-engine-development-governance-design.md)
25. [Miraikanai Engine AI検証・評価・来歴規約](./2026-07-19-ai-verification-evaluation-provenance-design.md)

`2026-07-18-codex-config-optimization-design.md`は開発者個人のCodex設定資料であり、Engineの製品・Runtime・Editor・AI契約に対する決定権を持たず、公式Review setへ含めない。

## 2. 文書ごとの決定権

| 文書 | 決定する範囲 |
|---|---|
| AIネイティブ設計計画書 | Product vision、制作体験、AI／人間の関係、Phase、MVP、Review set全体の上位目的 |
| C++実行コード・構造化ゲームデータ規約 | C++／data境界、GameplayDefinition、NativeGameModule、AI実装選択、Script VM不採用 |
| 基盤アーキテクチャ規約 | C++、Memory、Pointer、Module、Dependency、`BuildDriverProfileV1`、Ninja／Gradle／Xcode、命名、Directory |
| C++23・Named Modules移行規約 | C++23 Profile、Named Module名、`import std`、CMake、Ninja Generator matrix、Make非対応、BMI、Header例外、AI依存表現、Apple Build分離、CX0→CX3 Cutover Gate |
| Authoring Model／Project State規約 | Authoring Document、ProjectRevision、ChangeSet、transaction、journal、recovery、唯一の状態変更経路 |
| 実行可能契約規約 | Requirement、Type、Operation、State、Capability、Schema projection、Codegen |
| Runtime連携規約 | Tick、Writer、Command／Event／Snapshot、寿命、Asset version、Budget、Failure |
| 2D／3D機能計画 | 2D／3D Capability範囲、成熟度、Visual Style、各Subsystemの製品上の到達点 |
| NativeGameModule規約 | 公開ABI、Target別link方式、Build／Promotion、GameHost restart、信頼境界 |
| Asset Pipeline／Content Package規約 | Source／Import／Derived／Package、Importer隔離、Catalog／VFS、Cook、Patch／DLC、AI Asset来歴 |
| Rendering／Render Graph規約 | RenderSnapshot、extract、Render Graph、resource／pass／access、Backend、同期、device loss |
| 独自Particle／VFX Platform規約 | VFX Asset／Emitter／Graph／IR、2D／3D・CPU／GPU specialization、Runtime、Renderer binding、AI／Editor、Budget、Diagnostic、Qualification |
| Collision／Collider規約 | Body／Collider／Shape／Material／Filter、Query、Contact／Trigger、Cook、Editor／AI操作 |
| 独自Physics Platform／Dynamics規約 | Physics World、Solver Profile、Kernel Qualification、Dynamics、Joint／Constraint、Character Motor、Physics Save／Replay、Physics Editor／AI |
| Physics／Navigation／Animation連携規約 | Physics snapshot／command境界、Nav build／query、2D grid nav、Animation Graph、root motion、固定phase連携 |
| Input／Action／Device規約 | Device sample、semantic action、InputSnapshot、remap、replay、text入力分離、haptics |
| UI／Text／Localization／Accessibility規約 | Retained UI、layout、event／focus、IME、shaping／raster、localization、Game accessibility |
| 独自Editor UI Framework／Shell規約 | MiraUI C++ target、Widget、Retained／Immediate境界、D3D12 UI pass、DirectWrite／TSF／UIA／OLE Adapter、Docking、AI Semantic Interface、禁止GUI dependency |
| Audio／Mixer／Spatial規約 | Audio cue／voice／bus、mixer、streaming、spatial audio、callback、device adapter |
| Editor／Workspace／UX規約 | Window／Panel／Dock、Scene／Outliner／Inspector、AI Partner、AI Creator、workspace、Editor accessibility |
| Windows Platform／Distribution規約 | Windows Target、MSIX／folder distribution、signing、crash privacy、package layout |
| モバイル規約 | Android／iOS／iPadOS Target、Adapter、Lifecycle、Build／Signing／Upload分離、実機、Store |
| Domain Pack／将来Capability規約 | Genre Pack境界、初期Pack、将来Capabilityの非対象範囲と設計着手Gate |
| AI実装・保守ガバナンス | AI Task、権限、Risk、質問、Source sandbox、API／MCP／CLI／Plugin |
| AI検証・評価・来歴規約 | Gate、Test、形式モデル、Eval、Receipt、SBOM、Provenance、最新情報 |

同じ主題で矛盾する場合は、その主題の決定権を持つ正式仕様を優先する。上位のProduct目標を下位文書が変更してはならず、下位の具体的な安全規約を上位文書の概略表現で緩和してはならない。複数Subsystemに跨る変更では、各所有文書を同じChangeSetで更新し、Runtime phase、Schema、Memory、Target、Testの整合を確認する。

## 3. 外部規格とプロジェクト規約

外部公式資料はAPI、Platform、標準、Libraryの事実と推奨を提供する。Miraikanai Engine固有のphase、Budget、Risk、Approval、Directory、選択閾値は、本Review setが一次資料を検討して確定したプロジェクト公式規約である。

文書内の「最新」はfloating versionを意味しない。調査基準日に確認したStable／LTS／approved artifactをexact versionとhashで固定し、更新はEvidence、Eval、ADR、Reviewを経由する。

## 4. 設計から実装計画へ進むGate

実装計画書の作成へ進む前に、次をすべて満たす。

- ユーザーが公式Review set 25文書の方向性をReviewし、修正点または承認を返す。
- 文書間Link、Heading anchor、Table、Code fence、規範語の機械検査が通る。
- C0／C1に必要な選択が暗黙Defaultになっておらず、外部artifactから取得する値には取得手順、Owner、Block条件がある。
- Authoring、Runtime、Renderer、Asset、Editor、Input、UI、Audio、Physics、Navigation、Animation、Platform、AIの境界にOwner、入力、出力、phase、budget、failure、testが定義されている。
- Phase 0から2D First Playableまでを、依存関係と検証可能な完了条件を持つ実装taskへ分解できる。

このGateの通過は、**実装計画書を作成してよいことだけ**を意味する。Engine本体の実装開始を承認するものではない。

## 5. 実装開始とPhase完了Gate

Engine本体の実装は、公式Review set承認後に別文書として実装計画書を作成し、その計画書をユーザーがReviewして承認した後に開始する。

- 実装計画書はtaskごとに対象file、public contract、依存task、test、performance／memory gate、rollback、完了条件を記載する。
- Phase 0で生成するMiraikanai Contract Definition、Toolchain lock、Contract compiler、generated code、Gate、Fixtureは、設計Reviewの事前成果物ではなくPhase 0の実装成果物である。
- Phase完了は、該当文書のDefinition of Done、target matrix、conformance／integration／soak／package testが実artifactで合格した時だけ宣言する。
- 2D First Playable、3D First Playable、Android／Apple vertical slice、Production Capabilityは、それぞれ独立したMilestone Gateとして順に判定する。
- 仕様と実装が不一致になった場合、無断で仕様を実装へ合わせず、Evidenceと影響範囲を伴うADR／仕様ChangeSetを先にReviewする。
