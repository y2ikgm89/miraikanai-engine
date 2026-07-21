# Miraikanai Engine Product Plan

- 文書ID: mirakan.arch.product-plan
- 状態: review
- 正本範囲: Product intent、非交渉原則、Capability成熟度、Portfolio、MVP、Phase順序、製品昇格・停止・完了Gate
- 非正本範囲: Subsystemの型・Field・API・Backend・既定値・Budget、AI権限と承認、Evidence形式。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)
- 外部根拠検証日: 2026-07-21

## 1. Product intent

Miraikanai Engineは、既存EngineへChat機能を付加する製品ではない。人間とAIが同じ型付きAuthoring経路を使い、C++ Engineの信頼済みGatewayだけが検証済み変更をProjectへ確定する、独自のAI-native Game EngineとProjection Editorである。

目標Outcomeは次の一連の制作体験である。

1. 初心者は大まかな自然言語から開始できる。
2. AIはBlocking／High impactの不足要件だけを質問し、推奨案、影響、後から変更可能かを示す。
3. AIが理解した内容、仮定、未対応CapabilityをGame Briefとして人間へ提示する。
4. 承認済みGame BriefからGameSpec、実装計画、短縮されたGame全体と深い代表区間を持つFirst Playableを作る。
5. 構造化Game dataを第一選択とし、公開SDK内で必要かつ適格な場合だけProject C++／Project Shaderを使う。
6. AI、Editor GUI、外部IDE、MCPからの変更は同じChangeSet、Diff、Dry-run、Test、Approval、Undo、Replayへ収束する。
7. 人間の手動変更を正規Project revisionとして再読込し、古い仮定で上書きしない。
8. Editor制作型を完成させた後だけ、同じIRとValidatorを制限付きRuntime生成へ再利用する。

対象Userは初心者からC++を扱う上級者までである。両者は互換性のない別Project形式を使わず、同じ正規状態を異なるWorkspaceで見る。

## 2. 非交渉原則と明示的な非目標

### 2.1 非交渉原則

1. Game状態の正規表現とEditor／AI／Runtimeの表示を分離する。
2. 人間とAIは同じ変更Protocolを使い、AIの推論とEngineの権威判断を分離する。
3. Editorは特権的Writerではなく、正規状態のProjection兼ChangeSet作成者である。
4. 宣言的Source intentとTarget別Runtime artifactを分離する。
5. Genre固有機能を巨大Coreへ混入させず、Domain Packとして追加する。
6. Engineが所有するのは公開Capability、正規data model、編集Protocol、validation、lifecycle、serialization、fallback、Editor UXである。OS、Compiler、Graphics API、検証済みKernelまで再発明しない。
7. 外部LibraryはEngine-owned Adapter内へ隔離し、Project／AI APIへVendor型を漏らさない。
8. Game programming modelはC++23とGameplayDefinitionであり、汎用Game Script VM、JIT、Game向けFFIを持たない。
9. Game制作中のEngine、Editor、公開SDK、Validator、Policyは署名済みread-only baselineである。詳細は[AI Security／Approval](../01-governance/ai-security-approval.md)だけが決定する。
10. 設計名、Tier、Phase、比較対象Engineの存在を「利用可能」の証拠にしない。利用可能性はActivation Evidenceだけで決める。
11. 契約を固定し、最小vertical sliceを作り、実測し、上位bottleneckだけを最適化し、回帰Gateを通した後に昇格する。独立した「最後に最適化するPhase」は置かない。
12. Source intent、Gameplay fidelity floor、Visual StyleをTargetごとに分裂させない。Target差は各OwnerのCompilerとProfileが解決する。

### 2.2 非目標

- AIがC++ object、pointer、memory、GPU resource、Physics native handleを直接操作する経路。
- AIが任意Engine関数、shell、path、URL、console commandを呼ぶ経路。
- Provider Schemaに適合しただけで実行またはCommitを許すこと。
- Chat履歴、Editor view、Runtime WorldをProjectの正規記憶にすること。
- 一行Promptから無検証で完成品を一括生成すること。
- Unity、Unreal Engine、Godotの型階層、Scene形式、Editor操作、名称を模倣すること。
- MVPへMultiplayer、Account、Cloud、広告、課金、Runtime code generationを持ち込むこと。
- 将来Capabilityを理由に空Module、空Directory、placeholder APIを先行作成すること。
- 未選定Provider、Backend、Protocol、Store、Hardware、数値BudgetをRoadmap上で仮固定すること。

## 3. Capability成熟度

### 3.1 C0–C3

TierはProduct上の到達点であり、実装済み状態ではない。

| Tier | Product上の意味 | 最低の完了Evidence |
|---|---|---|
| C0 Foundation | identity、Owner、契約、禁止境界、validation、diagnostic、reference／negative fixtureが存在する | Unit／contract／conformance Gate |
| C1 First Playable | 一つの完成した2Dまたは3D vertical sliceで開始から終了まで利用できる | Playable、Save／Load、Target smoke、回帰Receipt |
| C2 Production | Authoring、品質Tier、profiling、fallback、Target別Qualificationを備える | Reference scene、性能、安定性、fault、release Evidence |
| C3 Advanced | 大規模World、Online、特殊Device、特殊Genre等の独立した汎用化 | 専用仕様、Threat Model、Benchmark、Domain Pack／Target Gate |

C3は「C2より良い実装」の総称ではない。各Capabilityが別仕様、別Prototype、意味同等fallback、個別Qualificationを持つ場合だけ着手する。

### 3.2 Activation state

Activation stateは実ArtifactのEvidenceに基づく現在状態であり、Tierと分離する。

| State | 意味 | AI／Editor／Projectの扱い |
|---|---|---|
| not_activated | 契約未完成、計画だけ、または利用可能なCandidateがない | 通常非表示またはGap説明のみ。Project利用、Package収録、成功扱いを禁止 |
| candidate_locked | CandidateのSource、Contract、Toolchain、Target、Policy、Evidence入力がexact hashで固定された | 隔離Staging／Prototypeだけ。Production ProjectへActivation不可 |
| qualified | 指定Target／Profileの必須Gateに合格し、制約とReceiptが固定された | allowlistされたProjectで個別承認付き利用可。別Targetへ一般化不可 |
| production | Release closureと継続Support Gateまで合格した | Production Manifest範囲で通常選択、Package、Release可 |

通常遷移は次の一方向だけである。

    not_activated -> candidate_locked -> qualified -> production

段階飛越を禁止する。Security incident、重大回帰、license失効、Provider停止、Evidence期限切れでは、productionからqualified、qualifiedからcandidate_locked、candidate_lockedからnot_activatedへ降格できる。降格後は新規選択とPromotionを止め、last-known-good、Migration、Rollback、利用中Projectへの影響をOwnerが明示する。

AIは次を推論してはならない。

- Roadmap記載、Tier、Phase、比較対象Engineの対応を「利用可能」と解釈する。
- OS、GPU、Store、ProviderのbrandからTarget対応を推測する。
- 未対応Capabilityを別機能で近似して成功扱いする。
- Multi-User、Gameplay Networking、Platform Serviceを相互に代用する。
- Presentation結果をauthoritative Gameplayへ逆入力する。

## 4. 2D／3D Capability portfolio

2Dと3Dは同格のFirst-class Runtimeであり、2Dを奥行き0の3Dとして実装しない。Asset、Input、Audio、UI、GameplayDefinition、AI Authoring、Build、Save、Diagnosticsは共有する。Rendering、Physics、Navigation、Animation AuthoringはDimension固有Ownerを持つ。

次表はProduct horizonだけを示す。SubsystemのField、Backend、既定値、Budget、Runtime順序は各Ownerへ移し、本表から推測してはならない。再編時点で実装Receiptを伴わないEntryのActivation stateはnot_activatedである。

| Portfolio group | C0 | C1 | C2 | C3／将来 | 主Owner |
|---|---|---|---|---|---|
| 共通Authoring／Project | 正規Project state、ChangeSet、Asset、Gameplay model | 手動とAIの安全な往復 | 大規模Project、拡張Authoring | Multi-User／UGCは別Entry | [Authoring Owners](#91-authoring) |
| 共通Runtime | scheduling、lifetime、capacity、debug | 2D／3D sliceを同じ契約で実行 | fault／soak／scale Qualification | distributed／onlineは別Entry | [Runtime Owners](#92-runtimesimulation) |
| 2D Presentation | 2D Render／Camera／Material契約 | 2D top-down shooter | lighting、VFX、style、fallback | 特殊表現は個別Entry | [Rendering Owners](#93-rendering) |
| 2D Simulation | Collision、Physics、Grid Navigation、Animation契約 | 2D vertical slice | Target別Kernel／content Qualification | Advanced motionは別Entry | [Simulation Owners](#92-runtimesimulation) |
| 3D Presentation | Mesh／World／Camera／Material／Light契約 | compact third-person arena | advanced lighting、post、environment、VFX | large world／Virtual Productionは別Entry | [Rendering Owners](#93-rendering) |
| 3D Simulation | Collision、Physics、Navigation、Animation契約 | 3D vertical slice | Target別Kernel／content Qualification | Vehicle、Ragdoll等は別Entry | [Simulation Owners](#92-runtimesimulation) |
| Player I/O | Input、Audio、UI／Text／Accessibility契約 | TitleからResultまでのoffline loop | Platform／locale／device matrix | Online serviceと分離 | [Platform Owners](#94-platformdomain-pack) |
| Platform | Windows、Mobile共通境界 | Windows offline package | Android／Apple offline | Linux／macOS／Web／Console／XRは別Entry | [Platform Owners](#94-platformdomain-pack) |
| Domain Pack | Pack契約とShooter Core境界 | 2D／3D Shooter Profile | 2D Action、TPS、RPG、Simulation等 | Multiplayer PackはOnline Gate後 | [Domain Pack Owners](#94-platformdomain-pack) |

## 5. MVP scope

MVPはEngine機能網羅版ではなく、AI Authoringの安全な往復を証明する製品vertical sliceである。

| Milestone | 完了Outcome |
|---|---|
| MVP-A | Phase 4完了。共通Shooter Coreを使う2D top-down shooterで、Prompt、質問、Game Brief、First Playable、AI修正、手動修正、AI再編集を一つのProject historyで完走する |
| MVP-B | Phase 6完了。同じShooter Coreを使う3D compact single-player third-person shooter arenaを追加し、共通基盤が2D専用でないことを証明する |
| Technology Preview | MVP-Bに加え、Release ActivationとProduct completion gateを満たす |

MVP First PlayableはTitleから開始し、Player操作、一つのCore loop、敵／課題／Simulation対象、達成可能なGoal、Result／終了、Save／LoadまたはCheckpointを持つ。AI生成GameplayDefinitionを実際の挙動へ使い、適格なProject Native Capabilityを少なくとも一つ使い、人間の手動変更を保持してAIが追加編集できる。

MVPに含める製品能力は、自由Prompt、Blocking／High質問、Game Brief、GameSpec、typed ChangeSet、構造化Scene／Rule／UI／Asset編集、GameplayDefinition、bounded Project C++／Shaderの隔離検証、Engine生成Diff、Approval、Commit、Undo、Replay、競合防止、First Playable、Cook、Package、Install、offline起動、diagnosis、support bundle、data resetである。

MVPから除外する。

- 商用品質の全Asset自動生成、複数Genre同時対応。
- Multiplayer、Runtime AI生成、Engine CoreのAI自動変更。
- Plugin Marketplace、複数Model ProviderのProduction routing。
- 大規模Open World、Production品質のCloud／GI／Terrain／Foliage。
- Ray tracing必須化、完全自動Release、無承認Commit。

MVP製品完了はEditor上でSceneが動くことではない。clean environment相当で次を一方向に通す。

    create／open -> author -> validate -> play -> save／load
    -> cook -> package -> install／launch -> first-run settings
    -> offline play -> checkpoint／resume -> diagnosis
    -> support／reset -> uninstall／upgrade decision

Network、Account、Provider、Store loginがなくてもTitleからResultまで成立しなければならない。

## 6. 単一Phase sequence

Phaseは次の順序だけをProduct正本とする。Subsystem Ownerはこの順序を増殖、並べ替え、別名化しない。Phase内の技術Task、型、Budgetは各Ownerと実装Planが決める。

| Phase | Product outcome | Exitの要点 |
|---|---|---|
| 0 Foundation契約とToolchain | 契約、Build、Policy、Evidence、測定の最小閉路 | 空Hostとheadless fixtureが固定入力で再現し、未Activationをfail closedにする |
| 1 Headless Authoring Core | Project state、ChangeSet、validate／stage／commit／undo／replay | 同じ入力から同じstate hashを得て、大規模Projectでもbounded queryが成立 |
| 2 Editor Shellと共通Runtime | AIなしでEditor、Runtime、Asset、Debugの共通経路 | 空SceneをWindowsでplay、save、packageできる |
| 3 2D Manual First Playable | 2D top-down shooterを手動制作 | TitleからResult、Save／Load、回帰、Target Gateが成立 |
| 4 AI Authoring MVP-A | 2DでAIと手動編集の往復 | MVP-AとProject Source Activationが成立 |
| 5 外部Agent接続 | local MCP／conformance済みClientを同じGatewayへ接続 | 外部ClientがProposal権限を越えず、credentialをEngineへ渡さない |
| 6 3D First Playable MVP-B | 共通Shooter Coreの3D arena | MVP-B、3D固有Owner、共通契約の非2D依存を実証 |
| 7 Mobile Platform | Android／Apple offline Target | Store／device／thermal／lifecycle GateをTarget別に通す |
| 8 Production CapabilityとDomain Pack | 一CapabilityずつC2へ昇格 | Authoring、diagnostic、fallback、Qualification closureを満たす |
| 9 制限付きRuntime生成 | 署名済みbinary内の許可済みdata変更 | 専用Threat Model、server authority、allowlist、quota、fallbackを承認 |

各Phaseは、契約固定、最小fixture、vertical slice、実測、上位bottleneck改善、同一条件の回帰、Capability昇格の順で閉じる。不合格時はSource intentとlast-known-goodを維持し、OptimizationRequiredまたはnot_activatedへ戻す。

## 7. Promotion、deactivation、Product completion

### 7.1 Promotion gate

Capabilityの昇格には、少なくとも次を必要とする。

1. 一意なOwner、Scope、Target、Requirement、fallback、failure policyがある。
2. SourceとDerived artifact、authoritative stateとPresentationが分離される。
3. AI／Editorが使えるtyped Operationと、禁止推論が定義される。
4. 必須Contract、semantic、policy、build、behavior、performance、fault、security Gateが成功する。
5. Evidenceが同一Source、Contract、Toolchain、Policy、Target、Qualityのexact hashへ結び付く。
6. qualifiedはTarget別Receiptと制約、productionはclean package、install、offline run、support、rollback、release Evidenceを持つ。
7. AIまたは文書の自己申告、別Target、別Quality、古いReceipt、推定値を代用しない。

### 7.2 Deactivation gate

Security incident、重大回帰、license／Provider停止、Target非適合、Evidence失効を検出したOwnerは、新規選択、AI active use、Promotion、Package収録を停止する。現在状態、影響Target、利用Project、last-known-good、rollback／migration、再Qualification条件を記録し、無言のfallbackやGameplay意味変更を行わない。

### 7.3 Product completion gate

Product milestoneは、機能一覧のcheckだけでは完了しない。

- 対象Capability closureが要求されたActivation stateにある。
- clean build／install／launch、offline play、Save／Load、failure diagnosis、rollbackが再現する。
- Platform、Accessibility、Localization、Privacy、License、SBOM、Support、Known limitationsが閉じる。
- AIと手動編集が同じProject history、Diff、Approval、Undo、Replayを使う。
- 未対応Capability、未実測Target、失効Evidenceを明示し、成功表示しない。
- [AI Verification／Provenance](../01-governance/ai-verification-provenance.md)のRelease evidence closureが同一Candidateへ結び付く。

## 8. Future portfolio

次のEntryは境界を早期に認識するが、再編時点ではすべてnot_activatedである。Tierは到達目標であり、実装Schemaではない。正式仕様、Owner、Threat Model、Target、fallback、Qualificationが揃うまで空APIや仮Backendを作らない。

| Future entry | 目標Tier | 先に固定する境界 | not_activatedの間の扱い |
|---|---|---|---|
| Platform／Player Services | C2–C3 | Account、Auth、Achievement、Cloud Save、Commerce等を個別Capability化しGameplay Networkingと分離 | offline／null経路だけ。SDK導入を全Service有効化と解釈しない |
| Extension SDK／Package／Plugin | C2–C3 | data、Editor projection、Import、Platform provider、Native、Engine、Build toolのRisk分離 | 任意binary load、unbounded network、private APIを導入しない |
| Collaboration／VCS／Multi-User | C2–C3 | ChangeSet collaboration、VCS Adapter、remote sessionを三分離 | VCSをProject DB／Undo、Multi-UserをGameplay Networkにしない |
| Timeline／Media／Capture | C2 | Timelineのclock／bindingと、Camera／Animation／Audio／UI／Gameplay／CodecのOwner分離 | Camera sequenceだけを万能Timelineへ拡張しない |
| Terrain／Foliage | C2 | 共通Source revisionからRender、Collision、Navigation、Gameplay、Streaming artifactを別生成 | Renderer effectまたは巨大Managerへ集約しない |
| Multiplayer／Online | C3 | State owner、authority、Command／Event／Snapshot、Transport／Session／Gameplayの分離 | MVP Capability Manifestへ掲載しない |
| Advanced Physics／Animation | C2–C3 | Ragdoll、Vehicle、Retarget、Motion Warping、Crowdを個別Entry化 | Vendor API、Solver値、Node CatalogをEnvelope段階で決めない |
| Platform expansion | C3 | Linux、macOS、Web、Console、XRをTarget別Product closureで扱う | portable compileをProduct対応と表示しない |
| Mod／UGC／LiveOps | C3 | Package、Service、moderation、takedown、signed data update、server authorityの分離 | Patch／DLCやAI Asset stagingを対応扱いにしない |
| Virtual Production | C3候補 | Camera Runtime／Authoringとは別のProduct、Platform、I/O、sync、security境界 | not_activated。C3実装Schema、Protocol、Device defaultを本書に置かない |

## 9. Subsystem ownersとPrimary Product evidence

この節はOwnerへのroutingだけを行う。リンク先の型、Field、Gate、Budgetを本書へ再掲しない。

### 9.1 Authoring

- [Project state](../03-authoring/project-state.md)
- [Asset lifecycle](../03-authoring/asset-lifecycle.md)
- [Editor UI framework](../03-authoring/editor-ui-framework.md)
- [Editor workspace UX](../03-authoring/editor-workspace-ux.md)
- [Gameplay programming model](../03-authoring/gameplay-programming-model.md)
- [Native game module](../03-authoring/native-game-module.md)

### 9.2 Runtime／Simulation

- [Scheduling／lifetime](../04-runtime/scheduling-lifetime.md)
- [Performance／capacity](../04-runtime/performance-capacity.md)
- [Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)
- [Collision](../05-simulation/collision.md)
- [Physics](../05-simulation/physics.md)
- [Navigation](../05-simulation/navigation.md)
- [Animation](../05-simulation/animation.md)

### 9.3 Rendering

- [Render graph](../06-rendering/render-graph.md)
- [Materials](../06-rendering/materials.md)
- [Lighting](../06-rendering/lighting.md)
- [Post processing](../06-rendering/post-processing.md)
- [VFX authoring](../06-rendering/vfx-authoring.md)
- [VFX runtime](../06-rendering/vfx-runtime.md)
- [Camera](../06-rendering/camera.md)
- [Environment／surfaces](../06-rendering/environment-surfaces.md)
- [LOD](../06-rendering/lod.md)
- [World](../06-rendering/world.md)

### 9.4 Platform／Domain Pack

- [Windows](../07-platform/windows.md)
- [Mobile common](../07-platform/mobile-common.md)
- [Android](../07-platform/android.md)
- [Apple](../07-platform/apple.md)
- [Input](../07-platform/input.md)
- [Audio](../07-platform/audio.md)
- [UI／text／localization／accessibility](../07-platform/ui-text-localization-accessibility.md)
- [Domain Pack contract](../08-domain-packs/domain-pack-contract.md)
- [Shooter reference Pack](../08-domain-packs/shooter.md)

### 9.5 Foundation／Governance

- [Core architecture](../02-foundation/core-architecture.md)
- [Toolchain／dependencies](../02-foundation/toolchain-dependencies.md)
- [Executable contracts](../02-foundation/executable-contracts.md)
- [Naming／Project layout](../02-foundation/naming-project-layout.md)
- [C++23 Modules](../02-foundation/cpp23-modules.md)
- [Math core](../02-foundation/math-core.md)
- [Memory／pointers](../02-foundation/memory-pointers.md)
- [AI Security／Approval](../01-governance/ai-security-approval.md)
- [AI Verification／Provenance](../01-governance/ai-verification-provenance.md)

Primary Product Evidenceは、MVP-A／MVP-BのFirst Playable、clean package／install／offline run、AIと手動編集の往復、Save／Load、Undo／Replay、Target別Qualification、failure／rollback、Release closureである。EvidenceのRecord形式、保持、Trace、SBOM、ProvenanceはVerification Ownerだけが決定する。

## 10. 外部比較の使用範囲

Unreal Engine、Unity、Godotの公式資料はCoverageと責務分離の比較Evidenceにだけ使う。MiraikanaiのAPI、Class hierarchy、Scene、Project形式、Editor UX、既定値の根拠にはしない。

- [Unreal Engine Modules](https://dev.epicgames.com/documentation/unreal-engine/unreal-engine-modules)
- [Unity Package Manager](https://docs.unity3d.com/Manual/upm-ui.html)
- [Godot architecture](https://docs.godotengine.org/en/stable/contributing/development/core_and_modules/godot_architecture_diagram.html)

外部資料の時点依存情報はArchitecture定数にせず、更新判断とEvidence freshnessを[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)へ委ねる。
