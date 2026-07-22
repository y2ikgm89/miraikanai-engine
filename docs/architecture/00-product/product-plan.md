# Miraikanai Engine Product Plan

- 文書ID: mirakan.arch.product-plan
- 状態: review
- 正本範囲: Product intent、非交渉原則、Capability成熟度、Portfolio、MVP、Phase順序、製品昇格・停止・完了Gate
- 非正本範囲: Subsystemの型・Field・API・Backend・既定値・Budget、AI権限と承認、Evidence形式。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)
- 外部根拠検証日: 2026-07-22

## 1. Product intent

Miraikanai Engineは、既存EngineへChat機能を付加する製品ではない。人間とAIが同じ型付きAuthoring経路を使い、C++ Engineの信頼済みGatewayだけが検証済み変更をProjectへ確定する、独自のAI-native Game EngineとProjection Editorである。

目標Outcomeは次の一連の制作体験である。

1. 初心者は大まかな自然言語から開始できる。
2. AIはBlocking／High impactの不足要件だけを質問し、推奨案、影響、後から変更可能かを示す。
3. AIが理解した内容、仮定、未対応CapabilityをGame Briefとして人間へ提示する。
4. 承認済みGame BriefからGameSpec、実装計画、短縮されたGame全体と深い代表区間を持つFirst Playableを作る。
5. 構造化Game dataを第一選択とし、公開SDK内で必要かつ適格な場合だけProject C++／Project Shaderを使う。Project ShaderはParameter／Graph、semantic HLSL Module、Stage／Shading Model、declarative Techniqueの順に最小Levelを選び、native GPU APIへ到達しない。
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

Activation stateは実ArtifactのEvidenceに基づくTarget別の現在状態であり、Tierと分離する。正本は`CapabilityTargetActivationV1`の`{capability_id, target_id}`行であり、Capabilityに一つだけ置くscalar stateを禁止する。

| State | 意味 | AI／Editor／Projectの扱い |
|---|---|---|
| not_activated | 契約未完成、計画だけ、または利用可能なCandidateがない | 通常非表示またはGap説明のみ。Project利用、Package収録、成功扱いを禁止 |
| candidate_locked | CandidateのSource、Contract、Toolchain、Target、Policy、Evidence入力がexact hashで固定された | 隔離Staging／Prototypeだけ。Production ProjectへActivation不可 |
| qualified | 指定Target／Profileの必須Gateに合格し、制約とReceiptが固定された | allowlistされたProjectで個別承認付き利用可。別Targetへ一般化不可 |
| production | Release closureと継続Support Gateまで合格した | Production Manifest範囲で通常選択、Package、Release可 |

通常遷移は次の一方向だけである。

    not_activated -> candidate_locked -> qualified -> production

集約表示はTarget別stateからだけ導出する。Capability Registryで`required`としたTarget行が一件でも欠ける場合は`not_activated`、全行が存在する場合は上記順序の最小stateを集約stateとする。`optional`と`excluded`は集約へ含めない。集約値を保存、手動編集、Target別Receiptの代用に使用してはならない。

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

MVP First PlayableはTitleから開始し、Player操作、一つのCore loop、敵／課題／Simulation対象、達成可能なGoal、Result／終了、Save／LoadまたはCheckpointを持つ。AI生成GameplayDefinitionを実際の挙動へ使い、適格なProject Native CapabilityとProject Shader ModuleまたはTechniqueをそれぞれ少なくとも一つ使い、人間の手動変更を保持してAIが追加編集できる。

MVPに含める製品能力は、自由Prompt、Blocking／High質問、Game Brief、GameSpec、typed ChangeSet、構造化Scene／Rule／UI／Asset編集、GameplayDefinition、bounded Project C++／Shaderの隔離検証、Engine生成Diff、Approval、Commit、Undo、Replay、競合防止、First Playable、Cook、Package、Install、offline起動、diagnosis、support bundle（正本は[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md) §14の`SupportBundleV1`）、data resetである。

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

各Phaseは、契約固定、最小fixture、vertical slice、実測、上位bottleneck改善、同一条件の回帰、Capability昇格の順で閉じる。不合格時はSource intentとlast-known-goodを維持し、OptimizationRequiredを状態ではなくDiagnosticとして発行する。Activation stateはcandidate_lockedに留め、利用可能なCandidateを維持できない場合だけnot_activatedへ降格する。

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

### 7.4 C2 coverageとgenre横断Gate

4章の共通C2と各SubsystemがC2と宣言するCapabilityは、次のProduct-owned matrixへ一件ずつ登録する。

```text
C2CapabilityCoverageMatrixV1
  matrix_id
  revision
  entries[]:
    capability_id
    owner_work_package_ref
    entry_gate_refs[]
    implementation_symbol_refs[]
    validator_refs[]
    fixture_refs[]
    target_refs[]
    fallback_ref
    qualification_receipt_refs[]
    include_in_product_label: bool
    capability_activation_state: not_activated | candidate_locked | qualified | production
    defer_reason
    dependency_refs[]
    reconsideration_gate_refs[]
```

`owner_work_package_ref`、`validator_refs[]`、`fixture_refs[]`、`target_refs[]`、`fallback_ref`、`qualification_receipt_refs[]`、`capability_activation_state`は必須である。一つでも空、参照不能、失効、対象Candidateとhash不一致なら`include_in_product_label=false`とし、Production表示を拒否する。Work Packageから外したCapabilityも削除せず、`capability_activation_state=not_activated`のままowner Work Packageの`scheduling_state=deferred`とdefer理由、依存、再検討Gateを残す。`capability_activation_state`は3.2節のActivation stateと同一の単一軸であり、第二のstate軸を持たない。`production`の場合だけProduct labelへ含める。

Phase 8の`wp.product.general-coverage-2d`がこのmatrixと`capability.product.general_production_2d`を所有する。次の三つをすべて、人間の手動Authoring経路とAI Authoring経路の両方で合格した場合だけ公開し、Phase 3／4のShooter First PlayableからC2へ段階飛越しない。

| Playable fixture | Genre固有の完了条件 |
|---|---|
| `fixture.product.shooter-2d` | [Shooter Pack](../08-domain-packs/shooter.md)の2D top-down core loop、敵、Wave、Boss、score |
| `fixture.product.platformer-2d` | Platformer: 5分以上のTitle→3 room→checkpoint→Result、one-way／moving platform、slope／step／ground snap、continuous collision、room Camera、Flipbook event／hitbox、Game Timer |
| `fixture.product.puzzle-dialogue-2d` | 戦闘に依存しないTitle→3 room→Puzzle／Dialogue／Choice／Item／Interaction→Result、Focus、Text／Font、Timer付きpuzzle、Loading cancel／retry |

三fixtureは共通して、Title-to-Result、cook／package／clean install launch、Save slot／Load／Replay、keyboard／controller／touch、Localization／Accessibilityを同じrunで検証する。WindowsとMobileのQualification ReceiptはTarget別に保持し、一方、別genre、Editor内Play、manualまたはAI片方のReceiptで代用しない。各Capability rowの必須field欠落、`not_activated`、未対応Targetが一件でもあれば、個別Capabilityの状態を変えずProduct gateだけをfail closedにする。

Local multiplayerを有効化する場合は、Cameraのsplit viewだけで完了扱いにせず、次のProduct profileでSubsystem closureを追跡する。

```text
LocalPlaySessionProfileV1
  profile_id
  local_player_slots[]
  player_assignment_policy_ref
  join_leave_policy_ref
  input_profile_refs[]
  game_flow_policy_ref
  ui_join_prompt_ref
  ui_focus_policy_ref
  split_viewport_profile_ref
  audio_listener_policy_ref
  local_player_profile_refs[]
  save_profile_binding_ref
  target_refs[]
  integrated_fixture_refs[]
  qualification_receipt_refs[]
```

player assignment、join／leave、Input、Game Flow、UI join prompt／focus、split viewport、Audio listener、[Platform Settingsが所有するlocal profile](../07-platform/ui-text-localization-accessibility.md)とSave bindingを一つのintegrated fixtureで閉じる。いずれかの参照またはReceiptが欠ける場合はsingle-player C2を維持し、local multiplayer Capabilityだけを未昇格にする。

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
- [Project shader](../06-rendering/project-shader.md)
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
- [Godot architecture](https://docs.godotengine.org/en/stable/engine_details/architecture/godot_architecture_diagram.html)

外部資料の時点依存情報はArchitecture定数にせず、更新判断とEvidence freshnessを[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)へ委ねる。

## 11. Product execution registries

本節はControl Plane移行時にMCDへ移すProduct-owned機械正本である。それまではMarkdown表を入力とし、表外のID、暗黙行、前方一致、別名を拒否する。

### 11.1 Registry共通規則

全Registryは`registry_id`、`format_major=1`、`revision=1`、`entries[]`を持つ。`entries[]`はlogical IDのUTF-8 byte順、重複禁止である。参照はexact logical IDだけを受理し、display name、path、配列index、maturity、Phase番号、schema versionをidentityとして使わない。

```text
CapabilityRegistryV1
  entries[]: capability_id, target_product_tier, owner_work_package_ref,
             target_bindings[], fallback_ref, capability_activation_state
  target_bindings[]: target_id, scope(required | optional | excluded)

CapabilityTargetActivationV1
  capability_id, target_id, state, candidate_ref, receipt_refs[], evidence_freshness

ProductPhaseRegistryV1
  entries[]: phase_id, order_index, outcome_requirement_refs[], work_package_refs[], exit_fixture_refs[]

WorkPackageRegistryV1
  entries[]: work_package_id, phase_id, owner_document_id, target_refs[], fallback_ref,
             fixture_refs[], requires_work_package_refs[], scheduling_state,
             defer_reason, reconsideration_gate_refs[], blocked_reason_ref

TargetProfileRegistryV1
  entries[]: target_id, owner_document_id, profile_version, qualification_policy_ref

RequirementRegistryV1
  entries[]: requirement_id, owner_document_id, verification_kind, failure_diagnostic_id

FixtureRegistryV1
  entries[]: fixture_id, owner_document_id, requirement_refs[], target_refs[], minimum_duration_seconds

FallbackRegistryV1
  entries[]: fallback_id, owner_document_id, preserves_semantics, failure_diagnostic_id
```

`defer_reason`と`blocked_reason_ref`は常にFieldを持ち、非該当時は`null`とする。`reconsideration_gate_refs[]`は常に存在し、非deferred時は空配列である。`scheduling_state=deferred`ではnon-empty `defer_reason`と1件以上の`reconsideration_gate_refs[]`、`blocked`ではnon-null `blocked_reason_ref`を必須とする。他stateでこれらを設定した行を拒否する。

`ProductPhaseRegistryV1`の`work_package_refs[]`は、`phase_id`が当該Phaseに一致する全Work Packageの全量列挙である。Phase→Work PackageとWork Package→Phaseの相互参照は双方向で一致させ、片側にのみ現れる参照を拒否する。`WorkPackageRegistryV1`の`fixture_refs[]`は当該Work Packageが実装を提供するfixtureの列挙であり、Work Package単独の完了gateではない。fixtureの合格判定は、そのfixtureを`exit_fixture_refs[]`で参照するPhaseのexit判定が所有する。

### 11.2 Target Profile registry

| target_id | Owner | profile_version | qualification_policy_ref |
|---|---|---:|---|
| `target.headless.host` | `mirakan.arch.core-architecture` | 1 | `requirement.target.headless-determinism` |
| `target.windows.editor` | `mirakan.arch.platform-windows` | 1 | `requirement.target.windows-editor` |
| `target.windows.desktop` | `mirakan.arch.platform-windows` | 1 | `requirement.target.windows-package` |
| `target.android.mobile` | `mirakan.arch.platform-android` | 1 | `requirement.target.android-package` |
| `target.apple.mobile` | `mirakan.arch.platform-apple` | 1 | `requirement.target.apple-package` |

Profile versionは更新可能なFieldであり、Target IDへ`v1`を埋め込まない。

### 11.3 Requirement、Fixture、Fallback registry

| requirement_id | Owner | verification_kind | failure diagnostic |
|---|---|---|---|
| `requirement.target.headless-determinism` | `mirakan.arch.core-architecture` | `determinism` | `diagnostic.product.headless-determinism-failed` |
| `requirement.target.windows-editor` | `mirakan.arch.platform-windows` | `package_and_launch` | `diagnostic.product.windows-editor-gate-failed` |
| `requirement.target.windows-package` | `mirakan.arch.platform-windows` | `clean_install_offline_run` | `diagnostic.product.windows-package-gate-failed` |
| `requirement.target.android-package` | `mirakan.arch.platform-android` | `store_package_device_run` | `diagnostic.product.android-package-gate-failed` |
| `requirement.target.apple-package` | `mirakan.arch.platform-apple` | `store_package_device_run` | `diagnostic.product.apple-package-gate-failed` |
| `requirement.product.authoring-roundtrip` | `mirakan.arch.product-plan` | `manual_and_ai_e2e` | `diagnostic.product.authoring-roundtrip-failed` |
| `requirement.product.title-to-result` | `mirakan.arch.product-plan` | `playable_e2e` | `diagnostic.product.title-to-result-failed` |
| `requirement.product.save-load-replay` | `mirakan.arch.product-plan` | `state_roundtrip` | `diagnostic.product.save-load-replay-failed` |
| `requirement.product.c2-2d-coverage` | `mirakan.arch.product-plan` | `cross_genre_matrix` | `diagnostic.product.c2-2d-coverage-incomplete` |
| `requirement.product.c2-3d-coverage` | `mirakan.arch.product-plan` | `cross_genre_matrix` | `diagnostic.product.c2-3d-coverage-incomplete` |
| `requirement.product.external-agent-boundary` | `mirakan.arch.ai-security-approval` | `authorization_conformance` | `diagnostic.product.external-agent-boundary-failed` |
| `requirement.product.runtime-generation-boundary` | `mirakan.arch.ai-security-approval` | `threat_model_conformance` | `diagnostic.product.runtime-generation-boundary-failed` |

| fixture_id | Owner | requirement_refs | targets | minimum duration seconds |
|---|---|---|---|---:|
| `fixture.product.headless-contract-smoke` | `mirakan.arch.product-plan` | `requirement.target.headless-determinism` | `target.headless.host` | 60 |
| `fixture.product.authoring-transaction` | `mirakan.arch.project-state` | `requirement.product.authoring-roundtrip` | `target.headless.host` | 300 |
| `fixture.product.windows-empty-scene` | `mirakan.arch.product-plan` | `requirement.target.windows-editor; requirement.target.windows-package` | `target.windows.editor; target.windows.desktop` | 300 |
| `fixture.product.shooter-2d` | `mirakan.arch.domain-pack-shooter` | `requirement.product.title-to-result; requirement.product.save-load-replay` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | 300 |
| `fixture.product.platformer-2d` | `mirakan.arch.product-plan` | `requirement.product.c2-2d-coverage; requirement.product.save-load-replay` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | 300 |
| `fixture.product.puzzle-dialogue-2d` | `mirakan.arch.product-plan` | `requirement.product.c2-2d-coverage; requirement.product.save-load-replay` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | 300 |
| `fixture.product.shooter-arena-3d` | `mirakan.arch.domain-pack-shooter` | `requirement.product.c2-3d-coverage; requirement.product.save-load-replay` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | 300 |
| `fixture.product.external-agent-proposal` | `mirakan.arch.ai-security-approval` | `requirement.product.external-agent-boundary` | `target.headless.host` | 120 |
| `fixture.product.mobile-lifecycle` | `mirakan.arch.platform-mobile-common` | `requirement.target.android-package; requirement.target.apple-package` | `target.android.mobile; target.apple.mobile` | 900 |
| `fixture.product.runtime-generation-denial` | `mirakan.arch.ai-security-approval` | `requirement.product.runtime-generation-boundary` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | 300 |

| fallback_id | Owner | preserves_semantics | failure diagnostic |
|---|---|---:|---|
| `fallback.capability.unavailable` | `mirakan.arch.product-plan` | false | `diagnostic.product.capability-unavailable` |
| `fallback.rendering.environment-core` | `mirakan.arch.rendering-environment-surfaces` | true | `diagnostic.rendering.environment-fallback-selected` |
| `fallback.rendering.ibl-baked` | `mirakan.arch.rendering-environment-surfaces` | true | `diagnostic.rendering.ibl-baked-selected` |
| `fallback.rendering.material-default` | `mirakan.arch.rendering-materials` | true | `diagnostic.rendering.material-default-selected` |
| `fallback.rendering.vfx-cpu` | `mirakan.arch.rendering-vfx-runtime` | true | `diagnostic.rendering.vfx-cpu-selected` |
| `fallback.rendering.vfx-core` | `mirakan.arch.rendering-vfx-runtime` | true | `diagnostic.rendering.vfx-core-selected` |

`preserves_semantics=false`は代替実装ではなくfail-closedを表す。成功Receiptを発行せず、Capabilityを選択不能として説明する。

### 11.4 Product Phase registry

| order | phase_id | outcome requirements | work packages | exit fixtures |
|---:|---|---|---|---|
| 0 | `phase.foundation` | `requirement.target.headless-determinism` | `wp.architecture.control-plane; wp.runtime.ecs-e0` | `fixture.product.headless-contract-smoke` |
| 1 | `phase.headless-authoring` | `requirement.product.authoring-roundtrip` | `wp.authoring.headless-core` | `fixture.product.authoring-transaction` |
| 2 | `phase.editor-runtime` | `requirement.target.windows-editor; requirement.target.windows-package` | `wp.runtime.d3d12-backend; wp.product.editor-runtime-windows` | `fixture.product.windows-empty-scene` |
| 3 | `phase.manual-2d` | `requirement.product.title-to-result; requirement.product.save-load-replay` | `wp.domain.shooter-core; wp.domain.shooter-2d` | `fixture.product.shooter-2d` |
| 4 | `phase.ai-authoring-mvp-a` | `requirement.product.authoring-roundtrip` | `wp.product.ai-authoring-mvp-a` | `fixture.product.shooter-2d` |
| 5 | `phase.external-agent` | `requirement.product.external-agent-boundary` | `wp.product.external-agent` | `fixture.product.external-agent-proposal` |
| 6 | `phase.manual-3d-mvp-b` | `requirement.product.c2-3d-coverage` | `wp.domain.shooter-3d` | `fixture.product.shooter-arena-3d` |
| 7 | `phase.mobile` | `requirement.target.android-package; requirement.target.apple-package` | `wp.platform.mobile-offline` | `fixture.product.mobile-lifecycle` |
| 8 | `phase.production-capability` | `requirement.product.c2-2d-coverage; requirement.product.c2-3d-coverage` | `wp.product.general-coverage-2d; wp.product.general-coverage-3d; wp.domain.platformer; wp.domain.puzzle-dialogue; wp.rendering.environment-c2; wp.rendering.vfx-c2; wp.rendering.material-toon; wp.ui.native-widget; wp.gameplay.core-c1; wp.navigation.path-following; wp.runtime.timer` | `fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d; fixture.product.shooter-arena-3d` |
| 9 | `phase.runtime-generation` | `requirement.product.runtime-generation-boundary` | `wp.product.runtime-generation` | `fixture.product.runtime-generation-denial` |

Phase 5とPhase 9はこのRegistry行を唯一のscheduling identityとし、本文上の見出しや番号だけで存在を表現しない。

Phase exitのTarget範囲は次で導出する。exit判定は`exit_fixture_refs[]`の各fixtureを、`order_index`が当該Phase以下のPhaseの`outcome_requirement_refs[]`に`qualification_policy_ref`が含まれるTargetと、fixture行の`target_refs[]`の積集合に限って要求する。fixture行の`target_refs[]`は全Phase完了時点の到達範囲であり、単独ではPhase exitのTarget範囲を定義しない。11.7節のC2 gateのように対象Targetを明示列挙するgateは、この導出より優先する。

### 11.5 Work Package registry

表の`state`はすべて`declared`である。計画書の存在を`scheduled`、`active`、`complete`へ読み替えない。

| work_package_id | phase_id | Owner | targets | fallback | fixtures | requires WP | state | defer_reason | reconsideration gates | blocked reason |
|---|---|---|---|---|---|---|---|---|---|---|
| `wp.architecture.control-plane` | `phase.foundation` | `mirakan.arch.architecture-governance` | `target.headless.host` | `fallback.capability.unavailable` | `fixture.product.headless-contract-smoke` | `[]` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e0` | `phase.foundation` | `mirakan.arch.runtime-scheduling-lifetime` | `target.headless.host` | `fallback.capability.unavailable` | `fixture.product.headless-contract-smoke` | `wp.architecture.control-plane` | `declared` | `null` | `[]` | `null` |
| `wp.authoring.headless-core` | `phase.headless-authoring` | `mirakan.arch.project-state` | `target.headless.host` | `fallback.capability.unavailable` | `fixture.product.authoring-transaction` | `wp.runtime.ecs-e0` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.d3d12-backend` | `phase.editor-runtime` | `mirakan.arch.rendering-render-graph` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene` | `wp.architecture.control-plane; wp.runtime.ecs-e0` | `declared` | `null` | `[]` | `null` |
| `wp.product.editor-runtime-windows` | `phase.editor-runtime` | `mirakan.arch.product-plan` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene` | `wp.authoring.headless-core; wp.runtime.d3d12-backend` | `declared` | `null` | `[]` | `null` |
| `wp.domain.shooter-core` | `phase.manual-2d` | `mirakan.arch.domain-pack-shooter` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-2d; fixture.product.shooter-arena-3d` | `wp.runtime.ecs-e0` | `declared` | `null` | `[]` | `null` |
| `wp.domain.shooter-2d` | `phase.manual-2d` | `mirakan.arch.domain-pack-shooter` | `target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.shooter-2d` | `wp.domain.shooter-core; wp.product.editor-runtime-windows` | `declared` | `null` | `[]` | `null` |
| `wp.product.ai-authoring-mvp-a` | `phase.ai-authoring-mvp-a` | `mirakan.arch.product-plan` | `target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.shooter-2d` | `wp.domain.shooter-2d` | `declared` | `null` | `[]` | `null` |
| `wp.product.external-agent` | `phase.external-agent` | `mirakan.arch.ai-security-approval` | `target.headless.host` | `fallback.capability.unavailable` | `fixture.product.external-agent-proposal` | `wp.product.ai-authoring-mvp-a` | `declared` | `null` | `[]` | `null` |
| `wp.domain.shooter-3d` | `phase.manual-3d-mvp-b` | `mirakan.arch.domain-pack-shooter` | `target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `wp.domain.shooter-core; wp.product.external-agent` | `declared` | `null` | `[]` | `null` |
| `wp.platform.mobile-offline` | `phase.mobile` | `mirakan.arch.platform-mobile-common` | `target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle` | `wp.domain.shooter-3d` | `declared` | `null` | `[]` | `null` |
| `wp.product.general-coverage-2d` | `phase.production-capability` | `mirakan.arch.product-plan` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d` | `wp.platform.mobile-offline` | `declared` | `null` | `[]` | `null` |
| `wp.product.general-coverage-3d` | `phase.production-capability` | `mirakan.arch.product-plan` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `wp.platform.mobile-offline` | `declared` | `null` | `[]` | `null` |
| `wp.domain.platformer` | `phase.production-capability` | `mirakan.arch.gameplay-programming-model` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.platformer-2d` | `wp.product.general-coverage-2d` | `declared` | `null` | `[]` | `null` |
| `wp.domain.puzzle-dialogue` | `phase.production-capability` | `mirakan.arch.gameplay-programming-model` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.puzzle-dialogue-2d` | `wp.product.general-coverage-2d` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.environment-c2` | `phase.production-capability` | `mirakan.arch.rendering-environment-surfaces` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.rendering.environment-core` | `fixture.product.shooter-2d; fixture.product.shooter-arena-3d` | `wp.product.general-coverage-2d; wp.product.general-coverage-3d` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.vfx-c2` | `phase.production-capability` | `mirakan.arch.rendering-vfx-runtime` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.rendering.vfx-core` | `fixture.product.shooter-2d; fixture.product.shooter-arena-3d` | `wp.product.general-coverage-2d; wp.product.general-coverage-3d` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.material-toon` | `phase.production-capability` | `mirakan.arch.rendering-materials` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.rendering.material-default` | `fixture.product.shooter-2d; fixture.product.shooter-arena-3d` | `wp.product.general-coverage-2d; wp.product.general-coverage-3d` | `declared` | `null` | `[]` | `null` |
| `wp.ui.native-widget` | `phase.production-capability` | `mirakan.arch.platform-ui-text-localization-accessibility` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.puzzle-dialogue-2d` | `wp.product.general-coverage-2d` | `declared` | `null` | `[]` | `null` |
| `wp.gameplay.core-c1` | `phase.production-capability` | `mirakan.arch.gameplay-programming-model` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-2d; fixture.product.puzzle-dialogue-2d; fixture.product.shooter-arena-3d` | `wp.runtime.ecs-e0` | `declared` | `null` | `[]` | `null` |
| `wp.navigation.path-following` | `phase.production-capability` | `mirakan.arch.simulation-navigation` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.platformer-2d; fixture.product.shooter-arena-3d` | `wp.gameplay.core-c1` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.timer` | `phase.production-capability` | `mirakan.arch.runtime-scheduling-lifetime` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d` | `wp.runtime.ecs-e0` | `declared` | `null` | `[]` | `null` |
| `wp.product.runtime-generation` | `phase.runtime-generation` | `mirakan.arch.ai-security-approval` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.runtime-generation-denial` | `wp.domain.platformer; wp.domain.puzzle-dialogue; wp.product.general-coverage-3d` | `declared` | `null` | `[]` | `null` |

### 11.6 Capability–Target–Fallback registry

全行の初期ActivationはTargetごとに`not_activated`、`candidate_ref=null`、`receipt_refs=[]`、`evidence_freshness=expired`とする。文書更新だけで昇格しない。Target scopeの`Windows`、`Android`、`Apple`はそれぞれ`target.windows.desktop`、`target.android.mobile`、`target.apple.mobile`の略記であり、Registry生成時は略記を保存せずexact IDへ展開する。

| capability_id | tier | owner WP | Target scope | fallback | activation |
|---|---|---|---|---|---|
| `capability.environment.aerial_perspective` | C2 | `wp.rendering.environment-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.environment-core` | `not_activated` |
| `capability.environment.atmosphere_lut` | C2 | `wp.rendering.environment-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.environment-core` | `not_activated` |
| `capability.environment.cloud_shadow` | C2 | `wp.rendering.environment-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.environment-core` | `not_activated` |
| `capability.environment.core` | C1 | `wp.rendering.environment-c2` | Windows required; Android required; Apple required | `fallback.rendering.ibl-baked` | `not_activated` |
| `capability.environment.dynamic_ibl` | C2 | `wp.rendering.environment-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.ibl-baked` | `not_activated` |
| `capability.environment.height_fog` | C2 | `wp.rendering.environment-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.environment-core` | `not_activated` |
| `capability.environment.ibl_baked` | C1 | `wp.rendering.environment-c2` | Windows required; Android required; Apple required | `fallback.rendering.environment-core` | `not_activated` |
| `capability.environment.intent_resolver` | C1 | `wp.rendering.environment-c2` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.environment.local_fog_volume` | C2 | `wp.rendering.environment-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.environment-core` | `not_activated` |
| `capability.environment.sky_hdri` | C1 | `wp.rendering.environment-c2` | Windows required; Android required; Apple required | `fallback.rendering.ibl-baked` | `not_activated` |
| `capability.environment.volumetric_cloud` | C2 | `wp.rendering.environment-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.environment-core` | `not_activated` |
| `capability.environment.volumetric_fog` | C2 | `wp.rendering.environment-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.environment-core` | `not_activated` |
| `capability.gameplay.interaction` | C1 | `wp.gameplay.core-c1` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.gameplay.path_following` | C1 | `wp.navigation.path-following` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.gameplay.perception` | C1 | `wp.gameplay.core-c1` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.gameplay.shooter_core` | C1 | `wp.domain.shooter-core` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.gameplay.timer` | C1 | `wp.runtime.timer` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.product.general_production_2d` | C2 | `wp.product.general-coverage-2d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.product.general_production_3d` | C2 | `wp.product.general-coverage-3d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.render.material.toon` | C2 | `wp.rendering.material-toon` | Windows required; Android required; Apple required | `fallback.rendering.material-default` | `not_activated` |
| `capability.ui.native_widget` | C2 | `wp.ui.native-widget` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.vfx.bake_cache` | C2 | `wp.rendering.vfx-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-core` | `not_activated` |
| `capability.vfx.billboard_3d` | C1 | `wp.rendering.vfx-c2` | Windows required; Android required; Apple required | `fallback.rendering.vfx-cpu` | `not_activated` |
| `capability.vfx.extension_operator` | C2 | `wp.rendering.vfx-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-core` | `not_activated` |
| `capability.vfx.mesh_ribbon` | C2 | `wp.rendering.vfx-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-cpu` | `not_activated` |
| `capability.vfx.particle_cpu` | C1 | `wp.rendering.vfx-c2` | Windows required; Android required; Apple required | `fallback.rendering.vfx-core` | `not_activated` |
| `capability.vfx.particle_gpu` | C2 | `wp.rendering.vfx-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-cpu` | `not_activated` |
| `capability.vfx.particle_light` | C2 | `wp.rendering.vfx-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-core` | `not_activated` |
| `capability.vfx.pattern_catalog` | C1 | `wp.rendering.vfx-c2` | Windows required; Android required; Apple required | `fallback.rendering.vfx-core` | `not_activated` |
| `capability.vfx.semantic_intent` | C1 | `wp.rendering.vfx-c2` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.vfx.sprite_2d` | C1 | `wp.rendering.vfx-c2` | Windows required; Android required; Apple required | `fallback.rendering.vfx-cpu` | `not_activated` |
| `capability.vfx.system` | C1 | `wp.rendering.vfx-c2` | Windows required; Android required; Apple required | `fallback.rendering.vfx-core` | `not_activated` |
| `capability.vfx.trail` | C1 | `wp.rendering.vfx-c2` | Windows required; Android required; Apple required | `fallback.rendering.vfx-cpu` | `not_activated` |
| `capability.vfx.visual_collision` | C2 | `wp.rendering.vfx-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-core` | `not_activated` |

`not_activated`行の`defer_reason`は「Phase 8の前提Work PackageとTarget Qualificationが未完了」、依存は対応Owner WP、再検討Gateは`phase.production-capability`開始Gateとする。これら三FieldをRegistry生成時に各行へ物理格納し、省略しない。

### 11.7 C2 3D gate

`wp.product.general-coverage-3d`は`fixture.product.shooter-arena-3d`だけでProduct C2を宣言しない。初回C2 gateは、同fixtureをWindows、Android、Appleの三Targetでmanual／AI Authoring双方から30分soakし、Title→Result、Save／Load／Replay、package／clean install、controller／touch、Localization／Accessibility、device lossまたはlifecycle recoveryを同一Candidate hashで合格させる。第二の非Shooter 3D fixtureがRegistryへ追加されるまでは`capability.product.general_production_3d`を`not_activated`に留め、Product labelを`3D First Playable`に限定する。
