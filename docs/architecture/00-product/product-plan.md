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

### 5.1 開発体制・見積り・risk contract

実績のない人数、AI利用量、runner capacityからcalendar日程を推測しない。計画値は次のclosed contractで保持し、ユーザー入力と実測Receiptが揃うまで`unfixed`を成功可能な値へ置換しない。

| Field | Rule |
|---|---|
| `team_assumption_state` | ユーザー入力前は`unfixed`。人数、役割、AI利用量を推測しない |
| `planning_capacity` | calendar期間を出さず、相対sizeと依存DAGだけを保持する |
| `phase_estimate` | elapsed timeではなく相対size `S / M / L / XL` |
| `critical_path` | Control Plane → ECS E0 →（Headless AuthoringとD3D12を並行）→ Editor Runtime → 2D Shooter → Project Source Activation → Authoring MVP-A |
| `scope_reduction_order` | C2 advanced rendering → non-Shooter packs → mobile shipping。MVP-A contractは削除しない |
| `risk_owner` | `mirakan.arch.product-plan` |
| `review_cadence` | 各Phase exitで更新し、仮定を同じCandidateの実測Receiptへ差し替える |

`team_assumption_state=unfixed`では担当者名、完了日、同時実行lane数を出力しない。日程または予算の提示が必要になった時点で、team composition、利用可能runner／device、稼働制約、外部依存を入力として別の承認済みplanning revisionを作る。scope削減は上表の順序でProposal化し、Phase 4のRequirement、Project Source Capability、security／package／support closureを暗黙に落とさない。

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

本節の`FutureCapabilityIncubationRegistryV1`は、将来境界を実装DAGから隔離したまま、再検討に必要な決定とQualification形を失わないための唯一のProduct正本である。

```text
FutureCapabilityIncubationRegistryV1
  entries[]:
    future_capability_id
    owner_document_id
    planning_state
    prerequisite_capability_refs[]
    required_decision_kinds[]
    candidate_target_kinds[]
    qualification_fixture_kinds[]
    activation_trigger
    excluded_current_product_claims[]
```

`planning_state`の初期値は全行`planning_only`である。`required_decision_kinds[]`は`product_scope | authority_model | data_model | threat_model | target_program | provider_selection | licensing | operations | qualification`、`candidate_target_kinds[]`は`headless_server | desktop | mobile | console | web | xr | distributed_cluster`、`qualification_fixture_kinds[]`は`contract | deterministic_simulation | network_fault | scale | target_device | content_quality | security | operations`のclosed値だけを受理する。

| future_capability_id | Owner | planning_state | prerequisite capability refs | required decision kinds | candidate target kinds | qualification fixture kinds | activation trigger | excluded current product claims |
|---|---|---|---|---|---|---|---|---|
| `future.capability.offline-large-world-continuous-streaming` | `mirakan.arch.rendering-world` | `planning_only` | `capability.world.3d; capability.environment.core` | `product_scope; data_model; qualification` | `desktop; mobile` | `contract; scale; target_device` | C2 3DのTarget別production Receipt後に、offline World partition、streaming authority、Save整合性、memory budgetの承認Decisionが成立する | `MVP-A; MVP-B; Technology Preview; general 3D production; offline large-world support` |
| `future.capability.headless-dedicated-server-session-transport-replication` | `mirakan.arch.runtime-scheduling-lifetime` | `planning_only` | `capability.runtime.scheduling; capability.runtime.ecs-e3-cook-load` | `product_scope; authority_model; data_model; threat_model; operations; qualification` | `headless_server; distributed_cluster` | `contract; deterministic_simulation; network_fault; security; operations` | headless Target programとserver authority、session transport、replication contract、運用責務の承認Decisionが揃う | `MVP-A; MVP-B; Technology Preview; multiplayer; dedicated-server support` |
| `future.capability.small-cooperative-multiplayer` | `mirakan.arch.product-plan` | `planning_only` | `capability.gameplay.shooter_core` | `product_scope; authority_model; threat_model; qualification` | `desktop; mobile; headless_server` | `deterministic_simulation; network_fault; target_device; security` | dedicated server／replicationのqualified Receipt後に、player count、join／leave、Save、host migrationのProduct Decisionが成立する | `MVP-A; MVP-B; Technology Preview; cooperative multiplayer` |
| `future.capability.rollback-competitive-networking` | `mirakan.arch.product-plan` | `planning_only` | `capability.gameplay.shooter_core` | `product_scope; authority_model; data_model; threat_model; qualification` | `desktop; mobile; headless_server` | `deterministic_simulation; network_fault; scale; security` | deterministic simulation、input delay、rollback window、anti-cheat、ranked operationsの承認Decisionとprototype Receiptが揃う | `MVP-A; MVP-B; Technology Preview; rollback networking; competitive networking` |
| `future.capability.persistence-live-service-moderation-operations` | `mirakan.arch.ai-security-approval` | `planning_only` | `capability.product.general_production_2d; capability.product.general_production_3d` | `product_scope; authority_model; data_model; threat_model; licensing; operations; qualification` | `headless_server; desktop; mobile; distributed_cluster` | `security; operations; network_fault; scale` | persistence、live update、moderation、privacy、retention、incident responseの各Owner Decisionと運用演習Receiptが揃う | `MVP-A; MVP-B; Technology Preview; persistence; live service; moderation` |
| `future.capability.mmo-distributed-world-authority` | `mirakan.arch.product-plan` | `planning_only` | `capability.product.general_production_3d` | `product_scope; authority_model; data_model; threat_model; operations; qualification` | `headless_server; distributed_cluster; desktop` | `deterministic_simulation; network_fault; scale; security; operations` | dedicated server、large-world、persistenceの各Capabilityがproductionとなり、shard／region authorityとfailure recoveryの承認Decisionが成立する | `MVP-A; MVP-B; Technology Preview; MMO; distributed world` |
| `future.capability.vehicle-ragdoll-crowd-motion-warping` | `mirakan.arch.simulation-physics` | `planning_only` | `capability.simulation.physics-3d; capability.simulation.animation-3d` | `product_scope; data_model; provider_selection; licensing; qualification` | `desktop; mobile` | `contract; deterministic_simulation; scale; target_device` | vehicle、ragdoll、crowd、motion warpingを独立Capabilityとして分割したOwner仕様とTarget prototype Receiptが揃う | `MVP-A; MVP-B; Technology Preview; vehicle; ragdoll; crowd; motion warping` |
| `future.capability.production-terrain-foliage-gi` | `mirakan.arch.rendering-environment-surfaces` | `planning_only` | `capability.world.3d; capability.environment.core` | `product_scope; data_model; provider_selection; licensing; qualification` | `desktop; mobile` | `content_quality; scale; target_device` | Terrain、Foliage、GIを別artifact pipelineとfallbackへ分離した仕様、budget Decision、Target比較Receiptが揃う | `MVP-A; MVP-B; Technology Preview; production Terrain; production Foliage; production GI` |
| `future.capability.console-target-program` | `mirakan.arch.product-plan` | `planning_only` | `capability.platform.mobile-lifecycle` | `target_program; threat_model; licensing; operations; qualification` | `console` | `contract; target_device; security; operations` | 特定Console programへの参加承認後に、NDA下Owner、SDK、certification、device lab、release operationsが確定する | `MVP-A; MVP-B; Technology Preview; console support; console shipping` |
| `future.capability.web-target-program` | `mirakan.arch.product-plan` | `planning_only` | `capability.rendering.render-graph-core` | `target_program; data_model; threat_model; provider_selection; licensing; qualification` | `web` | `contract; target_device; security; content_quality` | Web runtime、graphics、storage、threading、download、browser matrixのTarget Decisionとprototype Receiptが揃う | `MVP-A; MVP-B; Technology Preview; web support; web shipping` |
| `future.capability.xr-target-program` | `mirakan.arch.rendering-camera` | `planning_only` | `capability.camera.3d; capability.platform.input-core` | `target_program; data_model; threat_model; provider_selection; licensing; qualification` | `xr` | `contract; target_device; security; content_quality` | XR runtime、tracking、comfort、input、render budget、device matrixのTarget Decisionとprototype Receiptが揃う | `MVP-A; MVP-B; Technology Preview; XR support; XR shipping` |
| `future.capability.commercial-asset-generation-license-quality-qualification` | `mirakan.arch.asset-lifecycle` | `planning_only` | `capability.authoring.ai-core; capability.authoring.asset-save-headless` | `product_scope; provider_selection; licensing; threat_model; qualification` | `desktop; mobile` | `content_quality; security; target_device` | Provider、training／output license、provenance、quality rubric、human review、takedownの承認Decisionとblind evaluation Receiptが揃う | `MVP-A; MVP-B; Technology Preview; commercial asset generation; asset quality guarantee` |
| `future.capability.first-party-local-inference` | `mirakan.arch.ai-security-approval` | `planning_only` | `capability.authoring.ai-core; capability.product.external-agent` | `product_scope; provider_selection; threat_model; licensing; qualification` | `desktop` | `contract; target_device; security` | model／runtime Provider、weight license、hardware floor、sandbox、update／revocation、quality Gateの承認DecisionとTarget Receiptが揃う | `MVP-A; MVP-B; Technology Preview; bundled local model; offline inference guarantee` |

Future IDはactiveな`CapabilityRegistryV1`、`ProductPhaseRegistryV1`、`PhaseFixtureBindingRegistryV1`、Shipping labelから参照してはならない。上表からactive Capabilityへの`prerequisite_capability_refs[]`はincubation側だけが持つ一方向参照であり、CapabilityのActivation、Target対応、Phase schedulingを発生させない。各Entryを実装DAGへ移すには、新しいProduct DecisionでOwner、Target、fallback、Requirement、Fixture、Work Packageを同時に登録し、`planning_only`のまま暗黙activateしない。

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

全Registryは`registry_id`、`format_major=1`、`revision=1`、`entries[]`を持つ。`entries[]`はlogical IDのUTF-8 byte順、重複禁止である。Markdown表は人間の可読性のためPhase順などで提示してよいが、generatorは表をID付き集合として読み、serialized `entries[]`をlogical IDのUTF-8 byte順へ正規化する。validatorは正規化前後のID集合が完全一致することを検査し、表の表示順を保存状態やidentityとして扱わない。参照はexact logical IDだけを受理し、display name、path、配列index、maturity、Phase番号、schema versionをidentityとして使わない。

本節の`CapabilityRegistryV1`はProduct Phase、Target、fallback、Product labelのactivation判定に使うProduct-owned選抜registryであり、全Subsystem内部Capability catalogの複製ではない。Collision、Physics、Animation、Input、Audio等のSubsystem-local C0／C1 contractとdiscovery entryは各Ownerが所有する。ただしProduct Phase、Work Package、fixture、C2 matrix、またはTarget GateからCapability IDを参照する場合は、同じChangeSetで本節へexact行、owner Work Package、Target scope、fallbackを追加しなければならない。Subsystem catalog entryだけをProduct activationの代用にせず、本節未登録IDをProduct-qualifiedとして扱わない。

```text
CapabilityRegistryV1
  entries[]: capability_id, target_product_tier, owner_work_package_ref,
             target_bindings[], fallback_ref, capability_activation_state
  target_bindings[]: target_id, scope(required | optional | excluded)

CapabilityTargetActivationV1
  capability_id, target_id, state, candidate_ref, receipt_refs[], evidence_freshness

ProductPhaseRegistryV1
  entries[]: phase_id, order_index, outcome_requirement_refs[], work_package_refs[], exit_gate_refs[]

WorkPackageRegistryV1
  entries[]: work_package_id, phase_id, owner_document_id, target_refs[], fallback_ref,
             provided_fixture_refs[], required_capability_refs[], requires_work_package_refs[],
             scheduling_state, defer_reason, reconsideration_gate_refs[], blocked_reason_ref

PhaseFixtureBindingRegistryV1
  entries[]: gate_id, phase_id, fixture_id, evaluated_requirement_refs[], target_refs[],
             candidate_binding_policy_ref, freshness_policy_ref

TargetProfileRegistryV1
  entries[]: target_id, owner_document_id, profile_version, qualification_policy_ref

RequirementRegistryV1
  entries[]: requirement_id, owner_document_id, verification_kind, failure_diagnostic_id

FixtureRegistryV1
  entries[]: fixture_id, owner_document_id, requirement_refs[], target_refs[], minimum_duration_seconds

FallbackRegistryV1
  entries[]: fallback_id, owner_document_id, preserves_semantics, failure_diagnostic_id

FutureCapabilityIncubationRegistryV1
  entries[]: future_capability_id, owner_document_id, planning_state,
             prerequisite_capability_refs[], required_decision_kinds[], candidate_target_kinds[],
             qualification_fixture_kinds[], activation_trigger, excluded_current_product_claims[]
```

`scheduling_state`のclosed値は`declared | ready | active | blocked | deferred | complete`である。`defer_reason`と`blocked_reason_ref`は常にFieldを持ち、非該当時は`null`とする。`reconsideration_gate_refs[]`は常に存在し、非deferred時は空配列である。`scheduling_state=deferred`ではnon-empty `defer_reason`と1件以上の`reconsideration_gate_refs[]`、`blocked`ではnon-null `blocked_reason_ref`を必須とする。他stateでこれらを設定した行を拒否する。

`ProductPhaseRegistryV1`の`work_package_refs[]`は、`phase_id`が当該Phaseに一致する全Work Packageの全量列挙である。Phase→Work PackageとWork Package→Phaseの相互参照は双方向で一致させ、片側にのみ現れる参照を拒否する。`WorkPackageRegistryV1`の`provided_fixture_refs[]`は当該Work Packageが実装へ寄与するfixtureの列挙であり、Work Package単独の完了gateではない。RequirementとPhase completionをWork Packageへ複写せず、完了Receiptはappend-onlyな`WorkPackageLifecycleRecordV1`だけが所有する。

`PhaseFixtureBindingRegistryV1`の`evaluated_requirement_refs[]`と`target_refs[]`は参照先Fixtureの各集合のsubsetでなければならず、範囲外参照を拒否する。`candidate_binding_policy_ref=policy.product.same-candidate.v1`はProject revision、Candidate root hash、Contract set hash、Toolchain lock、Target Profileを全Receiptで一致させる。`freshness_policy_ref`はEvidence Ownerの`policy.evidence.contract-ci.v1`または`policy.evidence.target-device.v1`をexact参照し、失効、revoked、入力hash不一致のReceiptをPhase exitへ使用しない。

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
| `requirement.product.authoring-roundtrip-manual` | `mirakan.arch.product-plan` | `manual_e2e` | `diagnostic.product.authoring-roundtrip-manual-failed` |
| `requirement.product.authoring-roundtrip` | `mirakan.arch.product-plan` | `ai_e2e` | `diagnostic.product.authoring-roundtrip-failed` |
| `requirement.product.manual-first-playable-2d` | `mirakan.arch.product-plan` | `manual_playable_e2e` | `diagnostic.product.manual-first-playable-2d-incomplete` |
| `requirement.product.ai-authoring-mvp-a` | `mirakan.arch.product-plan` | `ai_mvp_e2e` | `diagnostic.product.ai-authoring-mvp-a-incomplete` |
| `requirement.product.first-playable-3d` | `mirakan.arch.product-plan` | `playable_e2e` | `diagnostic.product.first-playable-3d-incomplete` |
| `requirement.product.mvp-completion` | `mirakan.arch.product-plan` | `mvp_completion_e2e` | `diagnostic.product.mvp-completion-incomplete` |
| `requirement.product.project-source-activation` | `mirakan.arch.product-plan` | `source_activation_e2e` | `diagnostic.product.project-source-activation-incomplete` |
| `requirement.product.title-to-result` | `mirakan.arch.product-plan` | `playable_e2e` | `diagnostic.product.title-to-result-failed` |
| `requirement.product.save-load-replay` | `mirakan.arch.product-plan` | `state_roundtrip` | `diagnostic.product.save-load-replay-failed` |
| `requirement.product.c2-2d-coverage` | `mirakan.arch.product-plan` | `cross_genre_matrix` | `diagnostic.product.c2-2d-coverage-incomplete` |
| `requirement.product.c2-3d-coverage` | `mirakan.arch.product-plan` | `cross_genre_matrix` | `diagnostic.product.c2-3d-coverage-incomplete` |
| `requirement.product.external-agent-boundary` | `mirakan.arch.ai-security-approval` | `authorization_conformance` | `diagnostic.product.external-agent-boundary-failed` |
| `requirement.product.runtime-generation-boundary` | `mirakan.arch.ai-security-approval` | `threat_model_conformance` | `diagnostic.product.runtime-generation-boundary-failed` |

`requirement.product.manual-first-playable-2d`は手動AuthoringだけでTitle→Result、Save／Load／Replay、Windows package／clean install／offline playを完走する。`requirement.product.ai-authoring-mvp-a`はAIと手動編集の往復、typed ChangeSet、Diff、Approval、Commit、Undo、Replay、Project Source Activation、§5のMVP completion chainを一つのProject historyと同一Candidateで完走する。後者のsub-requirementとして`requirement.product.authoring-roundtrip`、`requirement.product.mvp-completion`、`requirement.product.project-source-activation`をEvidenceへ記録するが、Phase 4 gateはaggregate ID一件だけを評価する。

`requirement.product.mvp-completion`は§5の一方向chain全体、すなわちclean environment相当でのcook、package、install／launch、first-run settings、offline play、checkpoint／resume、diagnosis、`SupportBundleV1`取得、data resetを同一Candidate hashで検証する。`requirement.product.project-source-activation`は`capability.project.native_module`と`capability.project.shader`の両方を実際のFirst Playable挙動から使用し、Source、生成Diff、Approval、Target artifact、Qualification Receiptが同一Project revisionへ閉じる場合だけ成功する。

| fixture_id | Owner | requirement_refs | targets | minimum duration seconds |
|---|---|---|---|---:|
| `fixture.product.headless-contract-smoke` | `mirakan.arch.product-plan` | `requirement.target.headless-determinism` | `target.headless.host` | 60 |
| `fixture.product.authoring-transaction` | `mirakan.arch.project-state` | `requirement.product.authoring-roundtrip-manual` | `target.headless.host` | 300 |
| `fixture.product.windows-empty-scene` | `mirakan.arch.product-plan` | `requirement.target.windows-editor; requirement.target.windows-package` | `target.windows.editor; target.windows.desktop` | 300 |
| `fixture.product.shooter-2d` | `mirakan.arch.domain-pack-shooter` | `requirement.product.manual-first-playable-2d; requirement.product.ai-authoring-mvp-a; requirement.product.authoring-roundtrip; requirement.product.title-to-result; requirement.product.save-load-replay; requirement.product.mvp-completion; requirement.product.project-source-activation; requirement.product.c2-2d-coverage` | `target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | 300 |
| `fixture.product.platformer-2d` | `mirakan.arch.product-plan` | `requirement.product.c2-2d-coverage; requirement.product.save-load-replay` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | 300 |
| `fixture.product.puzzle-dialogue-2d` | `mirakan.arch.product-plan` | `requirement.product.c2-2d-coverage; requirement.product.save-load-replay` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | 300 |
| `fixture.product.shooter-arena-3d` | `mirakan.arch.domain-pack-shooter` | `requirement.product.first-playable-3d; requirement.product.save-load-replay; requirement.product.c2-3d-coverage` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | 300 |
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

| order | phase_id | outcome requirements | work packages | exit gates |
|---:|---|---|---|---|
| 0 | `phase.foundation` | `requirement.target.headless-determinism` | `wp.architecture.control-plane; wp.foundation.cpp23-cx0; wp.foundation.math-memory; wp.runtime.scheduling-core; wp.runtime.ecs-e0; wp.runtime.ecs-e1-storage; wp.runtime.ecs-e2-query-mutation` | `gate.product.phase-0-headless-contract` |
| 1 | `phase.headless-authoring` | `requirement.product.authoring-roundtrip-manual` | `wp.runtime.ecs-e3-cook-load; wp.authoring.project-state-headless; wp.authoring.asset-save-headless; wp.authoring.headless-core` | `gate.product.phase-1-authoring-transaction` |
| 2 | `phase.editor-runtime` | `requirement.target.windows-editor; requirement.target.windows-package` | `wp.runtime.ecs-e4-game-system; wp.rendering.render-graph-core; wp.runtime.d3d12-backend; wp.platform.input-windows; wp.platform.audio-windows; wp.platform.ui-windows; wp.platform.windows-package; wp.product.editor-runtime-windows` | `gate.product.phase-2-windows-empty-scene` |
| 3 | `phase.manual-2d` | `requirement.product.manual-first-playable-2d` | `wp.gameplay.core-c1; wp.runtime.timer; wp.rendering.world-2d; wp.rendering.camera-2d; wp.simulation.collision-2d; wp.simulation.physics-2d; wp.simulation.animation-2d; wp.navigation.path-following; wp.runtime.ecs-e5-2d-integration; wp.domain.shooter-core; wp.domain.shooter-2d` | `gate.product.phase-3-manual-2d` |
| 4 | `phase.ai-authoring-mvp-a` | `requirement.product.ai-authoring-mvp-a` | `wp.authoring.ai-core; wp.runtime.debug-replay-support; wp.runtime.ecs-e6-debug-ai; wp.authoring.prequalified-source-packs; wp.authoring.project-native-module; wp.rendering.project-shader; wp.runtime.ecs-e7-windows-2d; wp.product.ai-authoring-mvp-a` | `gate.product.phase-4-ai-mvp-a` |
| 5 | `phase.external-agent` | `requirement.product.external-agent-boundary` | `wp.product.external-agent` | `gate.product.phase-5-external-agent` |
| 6 | `phase.manual-3d-mvp-b` | `requirement.product.first-playable-3d` | `wp.rendering.world-3d; wp.rendering.camera-3d; wp.simulation.collision-3d; wp.simulation.physics-3d; wp.simulation.animation-3d; wp.runtime.ecs-e5-3d-integration; wp.runtime.ecs-e7-windows-3d; wp.domain.shooter-3d` | `gate.product.phase-6-first-playable-3d` |
| 7 | `phase.mobile` | `requirement.target.android-package; requirement.target.apple-package` | `wp.rendering.vulkan-backend; wp.rendering.metal-backend; wp.platform.android-package; wp.platform.apple-package; wp.platform.mobile-offline` | `gate.product.phase-7-android-lifecycle; gate.product.phase-7-apple-lifecycle` |
| 8 | `phase.production-capability` | `requirement.product.c2-2d-coverage; requirement.product.c2-3d-coverage` | `wp.domain.platformer; wp.domain.puzzle-dialogue; wp.rendering.environment-c2; wp.rendering.vfx-c2; wp.rendering.material-realistic; wp.rendering.material-toon; wp.ui.native-widget; wp.product.general-coverage-2d; wp.product.general-coverage-3d` | `gate.product.phase-8-c2-shooter-2d; gate.product.phase-8-c2-platformer-2d; gate.product.phase-8-c2-puzzle-dialogue-2d; gate.product.phase-8-c2-shooter-3d` |
| 9 | `phase.runtime-generation` | `requirement.product.runtime-generation-boundary` | `wp.product.runtime-generation` | `gate.product.phase-9-runtime-generation-denial` |

Phase 5とPhase 9はこのRegistry行を唯一のscheduling identityとし、本文上の見出しや番号だけで存在を表現しない。

Phase exitのRequirement、Target、Candidate binding、Freshnessは次のbinding表だけが決定する。fixtureの全Requirement／Targetを暗黙評価せず、同じfixtureを別Phaseで使っても別gateのReceiptを代用しない。

| gate_id | phase_id | fixture_id | evaluated requirement refs | target refs | candidate binding policy | freshness policy |
|---|---|---|---|---|---|---|
| `gate.product.phase-0-headless-contract` | `phase.foundation` | `fixture.product.headless-contract-smoke` | `requirement.target.headless-determinism` | `target.headless.host` | `policy.product.same-candidate.v1` | `policy.evidence.contract-ci.v1` |
| `gate.product.phase-1-authoring-transaction` | `phase.headless-authoring` | `fixture.product.authoring-transaction` | `requirement.product.authoring-roundtrip-manual` | `target.headless.host` | `policy.product.same-candidate.v1` | `policy.evidence.contract-ci.v1` |
| `gate.product.phase-2-windows-empty-scene` | `phase.editor-runtime` | `fixture.product.windows-empty-scene` | `requirement.target.windows-editor; requirement.target.windows-package` | `target.windows.editor; target.windows.desktop` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-3-manual-2d` | `phase.manual-2d` | `fixture.product.shooter-2d` | `requirement.product.manual-first-playable-2d` | `target.windows.desktop` | `policy.product.same-candidate.v1` | `policy.evidence.contract-ci.v1` |
| `gate.product.phase-4-ai-mvp-a` | `phase.ai-authoring-mvp-a` | `fixture.product.shooter-2d` | `requirement.product.ai-authoring-mvp-a` | `target.windows.editor; target.windows.desktop` | `policy.product.same-candidate.v1` | `policy.evidence.contract-ci.v1` |
| `gate.product.phase-5-external-agent` | `phase.external-agent` | `fixture.product.external-agent-proposal` | `requirement.product.external-agent-boundary` | `target.headless.host` | `policy.product.same-candidate.v1` | `policy.evidence.contract-ci.v1` |
| `gate.product.phase-6-first-playable-3d` | `phase.manual-3d-mvp-b` | `fixture.product.shooter-arena-3d` | `requirement.product.first-playable-3d` | `target.windows.desktop` | `policy.product.same-candidate.v1` | `policy.evidence.contract-ci.v1` |
| `gate.product.phase-7-android-lifecycle` | `phase.mobile` | `fixture.product.mobile-lifecycle` | `requirement.target.android-package` | `target.android.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-7-apple-lifecycle` | `phase.mobile` | `fixture.product.mobile-lifecycle` | `requirement.target.apple-package` | `target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-8-c2-shooter-2d` | `phase.production-capability` | `fixture.product.shooter-2d` | `requirement.product.c2-2d-coverage` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.contract-ci.v1` |
| `gate.product.phase-8-c2-platformer-2d` | `phase.production-capability` | `fixture.product.platformer-2d` | `requirement.product.c2-2d-coverage` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.contract-ci.v1` |
| `gate.product.phase-8-c2-puzzle-dialogue-2d` | `phase.production-capability` | `fixture.product.puzzle-dialogue-2d` | `requirement.product.c2-2d-coverage` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.contract-ci.v1` |
| `gate.product.phase-8-c2-shooter-3d` | `phase.production-capability` | `fixture.product.shooter-arena-3d` | `requirement.product.c2-3d-coverage` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.contract-ci.v1` |
| `gate.product.phase-9-runtime-generation-denial` | `phase.runtime-generation` | `fixture.product.runtime-generation-denial` | `requirement.product.runtime-generation-boundary` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.contract-ci.v1` |

### 11.5 Work Package registry

表の`scheduling_state`はすべて`declared`である。計画書の存在を`ready`、`active`、`complete`へ読み替えない。

| work_package_id | phase_id | Owner | targets | fallback | provided fixtures | required capabilities | requires WP | scheduling state | defer_reason | reconsideration gates | blocked reason |
|---|---|---|---|---|---|---|---|---|---|---|---|
| `wp.architecture.control-plane` | `phase.foundation` | `mirakan.arch.core-architecture` | `target.headless.host` | `fallback.capability.unavailable` | `fixture.product.headless-contract-smoke` | `[]` | `[]` | `declared` | `null` | `[]` | `null` |
| `wp.foundation.cpp23-cx0` | `phase.foundation` | `mirakan.arch.toolchain-dependencies` | `target.headless.host; target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.headless-contract-smoke; fixture.product.windows-empty-scene` | `capability.foundation.control-plane` | `wp.architecture.control-plane` | `declared` | `null` | `[]` | `null` |
| `wp.foundation.math-memory` | `phase.foundation` | `mirakan.arch.math-core` | `target.headless.host; target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.headless-contract-smoke; fixture.product.windows-empty-scene` | `capability.foundation.cpp23-cx0` | `wp.foundation.cpp23-cx0` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.scheduling-core` | `phase.foundation` | `mirakan.arch.runtime-scheduling-lifetime` | `target.headless.host; target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.headless-contract-smoke; fixture.product.windows-empty-scene` | `capability.foundation.math-memory` | `wp.foundation.math-memory` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e0` | `phase.foundation` | `mirakan.arch.runtime-scheduling-lifetime` | `target.headless.host; target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.headless-contract-smoke; fixture.product.windows-empty-scene` | `capability.runtime.scheduling` | `wp.runtime.scheduling-core` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e1-storage` | `phase.foundation` | `mirakan.arch.runtime-scheduling-lifetime` | `target.headless.host; target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.headless-contract-smoke; fixture.product.windows-empty-scene` | `capability.runtime.ecs-e0-contract` | `wp.runtime.ecs-e0` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e2-query-mutation` | `phase.foundation` | `mirakan.arch.runtime-scheduling-lifetime` | `target.headless.host; target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.headless-contract-smoke; fixture.product.windows-empty-scene` | `capability.runtime.ecs-e1-storage` | `wp.runtime.ecs-e1-storage` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e3-cook-load` | `phase.headless-authoring` | `mirakan.arch.runtime-scheduling-lifetime` | `target.headless.host; target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.authoring-transaction; fixture.product.windows-empty-scene` | `capability.runtime.ecs-e2-query-mutation` | `wp.runtime.ecs-e2-query-mutation` | `declared` | `null` | `[]` | `null` |
| `wp.authoring.project-state-headless` | `phase.headless-authoring` | `mirakan.arch.project-state` | `target.headless.host` | `fallback.capability.unavailable` | `fixture.product.authoring-transaction` | `capability.runtime.ecs-e3-cook-load` | `wp.runtime.ecs-e3-cook-load` | `declared` | `null` | `[]` | `null` |
| `wp.authoring.asset-save-headless` | `phase.headless-authoring` | `mirakan.arch.asset-lifecycle` | `target.headless.host` | `fallback.capability.unavailable` | `fixture.product.authoring-transaction` | `capability.authoring.project-state-headless` | `wp.authoring.project-state-headless` | `declared` | `null` | `[]` | `null` |
| `wp.authoring.headless-core` | `phase.headless-authoring` | `mirakan.arch.project-state` | `target.headless.host` | `fallback.capability.unavailable` | `fixture.product.authoring-transaction` | `capability.authoring.asset-save-headless` | `wp.authoring.asset-save-headless` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e4-game-system` | `phase.editor-runtime` | `mirakan.arch.runtime-scheduling-lifetime` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene` | `capability.runtime.ecs-e3-cook-load; capability.authoring.manual-roundtrip` | `wp.runtime.ecs-e3-cook-load; wp.authoring.headless-core` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.render-graph-core` | `phase.editor-runtime` | `mirakan.arch.rendering-render-graph` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene` | `capability.foundation.cpp23-cx0; capability.runtime.ecs-e0-contract` | `wp.foundation.cpp23-cx0; wp.runtime.ecs-e0` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.d3d12-backend` | `phase.editor-runtime` | `mirakan.arch.rendering-render-graph` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene` | `capability.rendering.render-graph-core` | `wp.rendering.render-graph-core` | `declared` | `null` | `[]` | `null` |
| `wp.platform.input-windows` | `phase.editor-runtime` | `mirakan.arch.platform-input` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene` | `capability.runtime.ecs-e4-game-system` | `wp.runtime.ecs-e4-game-system` | `declared` | `null` | `[]` | `null` |
| `wp.platform.audio-windows` | `phase.editor-runtime` | `mirakan.arch.platform-audio` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene` | `capability.runtime.ecs-e4-game-system` | `wp.runtime.ecs-e4-game-system` | `declared` | `null` | `[]` | `null` |
| `wp.platform.ui-windows` | `phase.editor-runtime` | `mirakan.arch.platform-ui-text-localization-accessibility` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene` | `capability.platform.input-core; capability.platform.audio-core` | `wp.platform.input-windows; wp.platform.audio-windows` | `declared` | `null` | `[]` | `null` |
| `wp.platform.windows-package` | `phase.editor-runtime` | `mirakan.arch.platform-windows` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene` | `capability.rendering.d3d12-cx0; capability.authoring.manual-roundtrip` | `wp.runtime.d3d12-backend; wp.authoring.headless-core` | `declared` | `null` | `[]` | `null` |
| `wp.product.editor-runtime-windows` | `phase.editor-runtime` | `mirakan.arch.product-plan` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene` | `capability.runtime.ecs-e4-game-system; capability.rendering.d3d12-cx0; capability.platform.input-core; capability.platform.audio-core; capability.platform.ui-core; capability.platform.windows-package` | `wp.runtime.ecs-e4-game-system; wp.runtime.d3d12-backend; wp.platform.input-windows; wp.platform.audio-windows; wp.platform.ui-windows; wp.platform.windows-package` | `declared` | `null` | `[]` | `null` |
| `wp.gameplay.core-c1` | `phase.manual-2d` | `mirakan.arch.gameplay-programming-model` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d; fixture.product.shooter-arena-3d` | `capability.runtime.ecs-e4-game-system; capability.product.editor-runtime-windows` | `wp.runtime.ecs-e4-game-system; wp.product.editor-runtime-windows` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.timer` | `phase.manual-2d` | `mirakan.arch.runtime-scheduling-lifetime` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d` | `capability.runtime.scheduling; capability.gameplay.interaction` | `wp.runtime.scheduling-core; wp.gameplay.core-c1` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.world-2d` | `phase.manual-2d` | `mirakan.arch.rendering-world` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d` | `capability.rendering.render-graph-core; capability.gameplay.interaction` | `wp.rendering.render-graph-core; wp.gameplay.core-c1` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.camera-2d` | `phase.manual-2d` | `mirakan.arch.rendering-camera` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d` | `capability.world.2d` | `wp.rendering.world-2d` | `declared` | `null` | `[]` | `null` |
| `wp.simulation.collision-2d` | `phase.manual-2d` | `mirakan.arch.simulation-collision` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-2d; fixture.product.platformer-2d` | `capability.runtime.ecs-e4-game-system; capability.gameplay.interaction` | `wp.runtime.ecs-e4-game-system; wp.gameplay.core-c1` | `declared` | `null` | `[]` | `null` |
| `wp.simulation.physics-2d` | `phase.manual-2d` | `mirakan.arch.simulation-physics` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-2d; fixture.product.platformer-2d` | `capability.simulation.collision-2d` | `wp.simulation.collision-2d` | `declared` | `null` | `[]` | `null` |
| `wp.simulation.animation-2d` | `phase.manual-2d` | `mirakan.arch.simulation-animation` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-2d; fixture.product.platformer-2d` | `capability.gameplay.timer; capability.gameplay.interaction` | `wp.runtime.timer; wp.gameplay.core-c1` | `declared` | `null` | `[]` | `null` |
| `wp.navigation.path-following` | `phase.manual-2d` | `mirakan.arch.simulation-navigation` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.shooter-arena-3d` | `capability.simulation.collision-2d; capability.gameplay.interaction` | `wp.simulation.collision-2d; wp.gameplay.core-c1` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e5-2d-integration` | `phase.manual-2d` | `mirakan.arch.runtime-scheduling-lifetime` | `target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.shooter-2d` | `capability.world.2d; capability.camera.2d; capability.simulation.physics-2d; capability.simulation.animation-2d; capability.gameplay.path_following` | `wp.rendering.world-2d; wp.rendering.camera-2d; wp.simulation.physics-2d; wp.simulation.animation-2d; wp.navigation.path-following` | `declared` | `null` | `[]` | `null` |
| `wp.domain.shooter-core` | `phase.manual-2d` | `mirakan.arch.domain-pack-shooter` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-2d; fixture.product.shooter-arena-3d` | `capability.gameplay.interaction; capability.gameplay.perception; capability.gameplay.timer` | `wp.gameplay.core-c1; wp.runtime.timer` | `declared` | `null` | `[]` | `null` |
| `wp.domain.shooter-2d` | `phase.manual-2d` | `mirakan.arch.domain-pack-shooter` | `target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.shooter-2d` | `capability.gameplay.shooter_core; capability.runtime.ecs-e5-2d-integration; capability.camera.2d` | `wp.domain.shooter-core; wp.runtime.ecs-e5-2d-integration; wp.product.editor-runtime-windows` | `declared` | `null` | `[]` | `null` |
| `wp.authoring.ai-core` | `phase.ai-authoring-mvp-a` | `mirakan.arch.ai-security-approval` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.shooter-2d` | `capability.authoring.manual-roundtrip; capability.domain.shooter-2d` | `wp.authoring.headless-core; wp.domain.shooter-2d` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.debug-replay-support` | `phase.ai-authoring-mvp-a` | `mirakan.arch.runtime-debugging-observability-replay` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.shooter-2d` | `capability.runtime.scheduling; capability.authoring.manual-roundtrip; capability.domain.shooter-2d` | `wp.runtime.scheduling-core; wp.authoring.headless-core; wp.domain.shooter-2d` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e6-debug-ai` | `phase.ai-authoring-mvp-a` | `mirakan.arch.runtime-scheduling-lifetime` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.shooter-2d` | `capability.runtime.ecs-e5-2d-integration; capability.authoring.ai-core; capability.runtime.debug-replay-support` | `wp.runtime.ecs-e5-2d-integration; wp.authoring.ai-core; wp.runtime.debug-replay-support` | `declared` | `null` | `[]` | `null` |
| `wp.authoring.prequalified-source-packs` | `phase.ai-authoring-mvp-a` | `mirakan.arch.native-game-module` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.shooter-2d` | `capability.authoring.ai-core; capability.runtime.ecs-e6-debug-ai` | `wp.authoring.ai-core; wp.runtime.ecs-e6-debug-ai` | `declared` | `null` | `[]` | `null` |
| `wp.authoring.project-native-module` | `phase.ai-authoring-mvp-a` | `mirakan.arch.native-game-module` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.shooter-2d` | `capability.authoring.prequalified-source-packs` | `wp.authoring.prequalified-source-packs` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.project-shader` | `phase.ai-authoring-mvp-a` | `mirakan.arch.rendering-project-shader` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.shooter-2d` | `capability.authoring.prequalified-source-packs; capability.rendering.render-graph-core` | `wp.authoring.prequalified-source-packs; wp.rendering.render-graph-core` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e7-windows-2d` | `phase.ai-authoring-mvp-a` | `mirakan.arch.runtime-scheduling-lifetime` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.shooter-2d` | `capability.runtime.ecs-e6-debug-ai; capability.project.native_module; capability.project.shader; capability.domain.shooter-2d` | `wp.runtime.ecs-e6-debug-ai; wp.authoring.project-native-module; wp.rendering.project-shader; wp.domain.shooter-2d` | `declared` | `null` | `[]` | `null` |
| `wp.product.ai-authoring-mvp-a` | `phase.ai-authoring-mvp-a` | `mirakan.arch.product-plan` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.shooter-2d` | `capability.authoring.ai-core; capability.runtime.debug-replay-support; capability.runtime.ecs-e7-windows-2d; capability.project.native_module; capability.project.shader` | `wp.authoring.ai-core; wp.runtime.debug-replay-support; wp.runtime.ecs-e7-windows-2d; wp.authoring.project-native-module; wp.rendering.project-shader` | `declared` | `null` | `[]` | `null` |
| `wp.product.external-agent` | `phase.external-agent` | `mirakan.arch.ai-security-approval` | `target.headless.host` | `fallback.capability.unavailable` | `fixture.product.external-agent-proposal` | `capability.product.ai-authoring-mvp-a` | `wp.product.ai-authoring-mvp-a` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.world-3d` | `phase.manual-3d-mvp-b` | `mirakan.arch.rendering-world` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.rendering.render-graph-core; capability.gameplay.interaction` | `wp.rendering.render-graph-core; wp.gameplay.core-c1` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.camera-3d` | `phase.manual-3d-mvp-b` | `mirakan.arch.rendering-camera` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.world.3d` | `wp.rendering.world-3d` | `declared` | `null` | `[]` | `null` |
| `wp.simulation.collision-3d` | `phase.manual-3d-mvp-b` | `mirakan.arch.simulation-collision` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.runtime.ecs-e4-game-system; capability.gameplay.interaction` | `wp.runtime.ecs-e4-game-system; wp.gameplay.core-c1` | `declared` | `null` | `[]` | `null` |
| `wp.simulation.physics-3d` | `phase.manual-3d-mvp-b` | `mirakan.arch.simulation-physics` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.simulation.collision-3d` | `wp.simulation.collision-3d` | `declared` | `null` | `[]` | `null` |
| `wp.simulation.animation-3d` | `phase.manual-3d-mvp-b` | `mirakan.arch.simulation-animation` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.simulation.physics-3d; capability.gameplay.timer` | `wp.simulation.physics-3d; wp.runtime.timer` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e5-3d-integration` | `phase.manual-3d-mvp-b` | `mirakan.arch.runtime-scheduling-lifetime` | `target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.world.3d; capability.camera.3d; capability.simulation.physics-3d; capability.simulation.animation-3d; capability.gameplay.path_following` | `wp.rendering.world-3d; wp.rendering.camera-3d; wp.simulation.physics-3d; wp.simulation.animation-3d; wp.navigation.path-following` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e7-windows-3d` | `phase.manual-3d-mvp-b` | `mirakan.arch.runtime-scheduling-lifetime` | `target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.runtime.ecs-e5-3d-integration; capability.runtime.ecs-e6-debug-ai` | `wp.runtime.ecs-e5-3d-integration; wp.runtime.ecs-e6-debug-ai` | `declared` | `null` | `[]` | `null` |
| `wp.domain.shooter-3d` | `phase.manual-3d-mvp-b` | `mirakan.arch.domain-pack-shooter` | `target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.gameplay.shooter_core; capability.runtime.ecs-e7-windows-3d; capability.camera.3d` | `wp.domain.shooter-core; wp.runtime.ecs-e7-windows-3d; wp.rendering.camera-3d` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.vulkan-backend` | `phase.mobile` | `mirakan.arch.rendering-render-graph` | `target.android.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle` | `capability.rendering.render-graph-core; capability.domain.shooter-3d` | `wp.rendering.render-graph-core; wp.domain.shooter-3d` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.metal-backend` | `phase.mobile` | `mirakan.arch.rendering-render-graph` | `target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle` | `capability.rendering.render-graph-core; capability.domain.shooter-3d` | `wp.rendering.render-graph-core; wp.domain.shooter-3d` | `declared` | `null` | `[]` | `null` |
| `wp.platform.android-package` | `phase.mobile` | `mirakan.arch.platform-android` | `target.android.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle` | `capability.rendering.vulkan-backend; capability.runtime.ecs-e7-windows-3d` | `wp.rendering.vulkan-backend; wp.runtime.ecs-e7-windows-3d` | `declared` | `null` | `[]` | `null` |
| `wp.platform.apple-package` | `phase.mobile` | `mirakan.arch.platform-apple` | `target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle` | `capability.rendering.metal-backend; capability.runtime.ecs-e7-windows-3d` | `wp.rendering.metal-backend; wp.runtime.ecs-e7-windows-3d` | `declared` | `null` | `[]` | `null` |
| `wp.platform.mobile-offline` | `phase.mobile` | `mirakan.arch.platform-mobile-common` | `target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle` | `capability.platform.android-package; capability.platform.apple-package` | `wp.platform.android-package; wp.platform.apple-package` | `declared` | `null` | `[]` | `null` |
| `wp.domain.platformer` | `phase.production-capability` | `mirakan.arch.gameplay-programming-model` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.platformer-2d` | `capability.platform.mobile-lifecycle; capability.gameplay.interaction; capability.gameplay.timer; capability.gameplay.path_following; capability.runtime.ecs-e7-windows-2d` | `wp.platform.mobile-offline; wp.gameplay.core-c1; wp.runtime.timer; wp.navigation.path-following; wp.runtime.ecs-e7-windows-2d` | `declared` | `null` | `[]` | `null` |
| `wp.domain.puzzle-dialogue` | `phase.production-capability` | `mirakan.arch.gameplay-programming-model` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.puzzle-dialogue-2d` | `capability.platform.mobile-lifecycle; capability.gameplay.interaction; capability.gameplay.timer; capability.platform.ui-core` | `wp.platform.mobile-offline; wp.gameplay.core-c1; wp.runtime.timer; wp.platform.ui-windows` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.environment-c2` | `phase.production-capability` | `mirakan.arch.rendering-environment-surfaces` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.rendering.environment-core` | `fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d; fixture.product.shooter-arena-3d` | `capability.platform.mobile-lifecycle; capability.rendering.render-graph-core; capability.domain.shooter-3d` | `wp.platform.mobile-offline; wp.rendering.render-graph-core; wp.domain.shooter-3d` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.vfx-c2` | `phase.production-capability` | `mirakan.arch.rendering-vfx-runtime` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.rendering.vfx-core` | `fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d; fixture.product.shooter-arena-3d` | `capability.platform.mobile-lifecycle; capability.rendering.render-graph-core; capability.domain.shooter-3d` | `wp.platform.mobile-offline; wp.rendering.render-graph-core; wp.domain.shooter-3d` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.material-realistic` | `phase.production-capability` | `mirakan.arch.rendering-materials` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.rendering.material-default` | `fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d; fixture.product.shooter-arena-3d` | `capability.platform.mobile-lifecycle; capability.rendering.render-graph-core; capability.domain.shooter-3d` | `wp.platform.mobile-offline; wp.rendering.render-graph-core; wp.domain.shooter-3d` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.material-toon` | `phase.production-capability` | `mirakan.arch.rendering-materials` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.rendering.material-default` | `fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d; fixture.product.shooter-arena-3d` | `capability.platform.mobile-lifecycle; capability.rendering.render-graph-core; capability.domain.shooter-3d` | `wp.platform.mobile-offline; wp.rendering.render-graph-core; wp.domain.shooter-3d` | `declared` | `null` | `[]` | `null` |
| `wp.ui.native-widget` | `phase.production-capability` | `mirakan.arch.platform-ui-text-localization-accessibility` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d; fixture.product.shooter-arena-3d` | `capability.platform.mobile-lifecycle; capability.platform.ui-core` | `wp.platform.mobile-offline; wp.platform.ui-windows` | `declared` | `null` | `[]` | `null` |
| `wp.product.general-coverage-2d` | `phase.production-capability` | `mirakan.arch.product-plan` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d` | `capability.domain.platformer; capability.domain.puzzle-dialogue; capability.environment.core; capability.vfx.system; capability.render.material.realistic_advanced; capability.render.material.toon; capability.ui.native_widget; capability.gameplay.interaction; capability.gameplay.timer; capability.gameplay.path_following` | `wp.domain.platformer; wp.domain.puzzle-dialogue; wp.rendering.environment-c2; wp.rendering.vfx-c2; wp.rendering.material-realistic; wp.rendering.material-toon; wp.ui.native-widget; wp.gameplay.core-c1; wp.runtime.timer; wp.navigation.path-following` | `declared` | `null` | `[]` | `null` |
| `wp.product.general-coverage-3d` | `phase.production-capability` | `mirakan.arch.product-plan` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.domain.shooter-3d; capability.environment.core; capability.vfx.system; capability.render.material.realistic_advanced; capability.render.material.toon; capability.ui.native_widget; capability.gameplay.interaction; capability.gameplay.timer; capability.gameplay.path_following` | `wp.domain.shooter-3d; wp.rendering.environment-c2; wp.rendering.vfx-c2; wp.rendering.material-realistic; wp.rendering.material-toon; wp.ui.native-widget; wp.gameplay.core-c1; wp.runtime.timer; wp.navigation.path-following` | `declared` | `null` | `[]` | `null` |
| `wp.product.runtime-generation` | `phase.runtime-generation` | `mirakan.arch.ai-security-approval` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.runtime-generation-denial` | `capability.product.general_production_2d; capability.product.general_production_3d` | `wp.product.general-coverage-2d; wp.product.general-coverage-3d` | `declared` | `null` | `[]` | `null` |

`wp.authoring.prequalified-source-packs`は初心者向けDefinition-first経路と、事前Qualification済みNative／Shader Packの選択だけを提供する。`wp.authoring.project-native-module`と`wp.rendering.project-shader`で新規AI生成Sourceを採用する場合は、各Ownerの独立したCode owner approval GateとSource／artifact／Receipt hash closureを必須とし、事前PackのReceiptを代用しない。

### 11.6 Capability–Target–Fallback registry

全行の初期ActivationはTargetごとに`not_activated`、`candidate_ref=null`、`receipt_refs=[]`、`evidence_freshness=expired`とする。文書更新だけで昇格しない。Target scopeの`Headless`、`Windows`、`Windows Editor`、`Android`、`Apple`はそれぞれ`target.headless.host`、`target.windows.desktop`、`target.windows.editor`、`target.android.mobile`、`target.apple.mobile`の略記であり、Registry生成時は略記を保存せずexact IDへ展開する。明記しないTargetは暗黙requiredにせず`excluded`として生成する。

| capability_id | tier | owner WP | Target scope | fallback | activation |
|---|---|---|---|---|---|
| `capability.foundation.control-plane` | C0 | `wp.architecture.control-plane` | Headless required; Windows Editor excluded; Windows excluded; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.foundation.cpp23-cx0` | C0 | `wp.foundation.cpp23-cx0` | Headless required; Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.foundation.math-memory` | C0 | `wp.foundation.math-memory` | Headless required; Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.runtime.scheduling` | C0 | `wp.runtime.scheduling-core` | Headless required; Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.runtime.ecs-e0-contract` | C0 | `wp.runtime.ecs-e0` | Headless required; Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.runtime.ecs-e1-storage` | C0 | `wp.runtime.ecs-e1-storage` | Headless required; Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.runtime.ecs-e2-query-mutation` | C0 | `wp.runtime.ecs-e2-query-mutation` | Headless required; Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.runtime.ecs-e3-cook-load` | C0 | `wp.runtime.ecs-e3-cook-load` | Headless required; Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.authoring.project-state-headless` | C0 | `wp.authoring.project-state-headless` | Headless required; Windows Editor excluded; Windows excluded; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.authoring.asset-save-headless` | C0 | `wp.authoring.asset-save-headless` | Headless required; Windows Editor excluded; Windows excluded; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.authoring.manual-roundtrip` | C0 | `wp.authoring.headless-core` | Headless required; Windows Editor excluded; Windows excluded; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.runtime.ecs-e4-game-system` | C0 | `wp.runtime.ecs-e4-game-system` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.rendering.render-graph-core` | C0 | `wp.rendering.render-graph-core` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.rendering.d3d12-cx0` | C0 | `wp.runtime.d3d12-backend` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.platform.input-core` | C0 | `wp.platform.input-windows` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.platform.audio-core` | C0 | `wp.platform.audio-windows` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.platform.ui-core` | C0 | `wp.platform.ui-windows` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.platform.windows-package` | C0 | `wp.platform.windows-package` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.product.editor-runtime-windows` | C0 | `wp.product.editor-runtime-windows` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.world.2d` | C1 | `wp.rendering.world-2d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.camera.2d` | C1 | `wp.rendering.camera-2d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.simulation.collision-2d` | C1 | `wp.simulation.collision-2d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.simulation.physics-2d` | C1 | `wp.simulation.physics-2d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.simulation.animation-2d` | C1 | `wp.simulation.animation-2d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.runtime.ecs-e5-2d-integration` | C1 | `wp.runtime.ecs-e5-2d-integration` | Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.domain.shooter-2d` | C1 | `wp.domain.shooter-2d` | Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.authoring.ai-core` | C1 | `wp.authoring.ai-core` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.runtime.debug-replay-support` | C1 | `wp.runtime.debug-replay-support` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.runtime.ecs-e6-debug-ai` | C1 | `wp.runtime.ecs-e6-debug-ai` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.authoring.prequalified-source-packs` | C1 | `wp.authoring.prequalified-source-packs` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.runtime.ecs-e7-windows-2d` | C1 | `wp.runtime.ecs-e7-windows-2d` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.product.ai-authoring-mvp-a` | C1 | `wp.product.ai-authoring-mvp-a` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.product.external-agent` | C1 | `wp.product.external-agent` | Headless required; Windows Editor excluded; Windows excluded; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.world.3d` | C1 | `wp.rendering.world-3d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.camera.3d` | C1 | `wp.rendering.camera-3d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.simulation.collision-3d` | C1 | `wp.simulation.collision-3d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.simulation.physics-3d` | C1 | `wp.simulation.physics-3d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.simulation.animation-3d` | C1 | `wp.simulation.animation-3d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.runtime.ecs-e5-3d-integration` | C1 | `wp.runtime.ecs-e5-3d-integration` | Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.runtime.ecs-e7-windows-3d` | C1 | `wp.runtime.ecs-e7-windows-3d` | Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.domain.shooter-3d` | C1 | `wp.domain.shooter-3d` | Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.rendering.vulkan-backend` | C1 | `wp.rendering.vulkan-backend` | Android required; Windows Editor excluded; Windows excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.rendering.metal-backend` | C1 | `wp.rendering.metal-backend` | Apple required; Windows Editor excluded; Windows excluded; Android excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.platform.android-package` | C1 | `wp.platform.android-package` | Android required; Windows Editor excluded; Windows excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.platform.apple-package` | C1 | `wp.platform.apple-package` | Apple required; Windows Editor excluded; Windows excluded; Android excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.platform.mobile-lifecycle` | C1 | `wp.platform.mobile-offline` | Android required; Apple required; Windows Editor excluded; Windows excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.domain.platformer` | C2 | `wp.domain.platformer` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.domain.puzzle-dialogue` | C2 | `wp.domain.puzzle-dialogue` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
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
| `capability.project.native_module` | C1 | `wp.authoring.project-native-module` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.project.shader` | C1 | `wp.rendering.project-shader` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` | `not_activated` |
| `capability.product.general_production_2d` | C2 | `wp.product.general-coverage-2d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.product.general_production_3d` | C2 | `wp.product.general-coverage-3d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.product.runtime-generation-boundary` | C3 | `wp.product.runtime-generation` | Windows required; Android required; Apple required | `fallback.capability.unavailable` | `not_activated` |
| `capability.render.material.realistic_advanced` | C2 | `wp.rendering.material-realistic` | Windows required; Android required; Apple required | `fallback.rendering.material-default` | `not_activated` |
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
