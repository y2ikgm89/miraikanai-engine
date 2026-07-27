# Miraikanai Engine Product Plan

- 文書ID: mirakan.arch.product-plan
- 状態: review
- 正本範囲: Product intent、非交渉原則、Capability成熟度、Portfolio、Algorithm／Performance最適化のProduct優先度、AI制作理解境界、MVP、Phase順序、製品昇格・停止・完了Gate
- 非正本範囲: Subsystemの型・Field・API・Backend・既定値・Budget、AI権限と承認、Evidence形式。各Owner文書を参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Runtime Performance／Capacity](../04-runtime/performance-capacity.md)、[Runtime ECS](../04-runtime/entity-component-system.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Persistence／Save](../04-runtime/persistence-save.md)、[Collision](../05-simulation/collision.md)、[Physics](../05-simulation/physics.md)、[Navigation](../05-simulation/navigation.md)、[Render Graph](../06-rendering/render-graph.md)
- 外部根拠検証日: 2026-07-27

## 1. Product intent

Miraikanai Engineは、既存EngineへChat機能を付加する製品ではない。人間とAIが同じ型付きAuthoring経路を使い、C++ Engineの信頼済みGatewayだけが検証済み変更をProjectへ確定する、独自のAI-native Game EngineとProjection Editorである。

目標Outcomeは次の一連の制作体験である。

1. 初心者は大まかな自然言語から開始できる。
2. AIはBlocking／High impactの不足要件だけを質問し、推奨案、影響、後から変更可能かを示す。
3. AIが理解した内容、仮定、未対応CapabilityをGame Briefとして人間へ提示する。
4. 承認済みGame BriefからGameSpec、実装計画、短縮されたGame全体と深い代表区間を持つFirst Playableを作る。
5. 構造化Game dataを第一選択とし、公開SDK内で必要かつ適格な場合だけProject C++／Project Shaderを使う。Project ShaderはParameter／Graph、`typed_ir`または`bounded_hlsl`のsemantic Module、Stage／Shading Model、declarative Techniqueの順に最小Levelを選び、native GPU APIへ到達しない。`typed_ir`はcanonical IR、`bounded_hlsl`はdeclared／observed contractとCode Owner reviewを必要とし、DXC reflectionだけを意味または挙動の完全証拠にしない。
6. AI、Editor GUI、外部IDE、MCPからの変更は同じChangeSet、Diff、Dry-run、Test、Approval、Undo、Replayへ収束する。
7. 人間の手動変更を正規Project revisionとして再読込し、古い仮定で上書きしない。
8. Runtime structured-data generationはEditor制作型完成だけでは開始せず、`future.capability.runtime-structured-data-generation`のactive移行、Target別Qualification、専用Authority／Threat Model承認後にだけ同じIRとValidatorの再利用を検討する。現在のProduct Phase 9はdeny-onlyである。

対象Userは初心者からC++を扱う上級者までである。両者は互換性のない別Project形式を使わず、同じ正規状態を異なるWorkspaceで見る。

最初のProduct-facing Reference Gameは、Product方向Reviewで承認したcompact 2D command RPGとする。これはGeneric Engine CoreのGenre非依存性、AI Authoringのtyped往復、Save／Load、UI／Text／Localization、data-rich Gameplayを一つのFirst Playableで証明するためのProduct選択であり、RPG意味をCoreへ移す判断ではない。Shooterは高頻度Input、Collision、Physics、ECS、Renderingのtechnical qualification consumerおよび後続Reference Gameとして保持できるが、Product MVP、Generic Engine Release、Core依存の根拠にしない。最初のRPGはMiraikanai独自のReference Gameとし、既存商用RPGの名称、人物、monsters、story、map、music、art、UI、商標その他の保護されたcreative expressionを再利用しない。

## 2. 非交渉原則と明示的な非目標

### 2.1 非交渉原則

1. Game状態の正規表現とEditor／AI／Runtimeの表示を分離する。
2. 人間とAIは同じ変更Protocolを使い、AIの推論とEngineの権威判断を分離する。
3. Editorは特権的Writerではなく、正規状態のProjection兼ChangeSet作成者である。
4. 宣言的Source intentとTarget別Runtime artifactを分離する。
5. 製品構造を`Generic Engine Core`、`Reusable Feature Packs`、任意の`Genre Packs`、`Game Projects`の4層とし、Genre固有機能をCoreへ混入させない。Game ProjectはGenre PackなしでFeature Packを直接利用できる。
6. Engineが所有するのは公開Capability、正規data model、編集Protocol、validation、lifecycle、serialization、fallback、Editor UXである。OS、Compiler、Graphics API、検証済みKernelまで再発明しない。
7. 外部LibraryはEngine-owned Adapter内へ隔離し、Project／AI APIへVendor型を漏らさない。
8. Game programming modelはC++23とGameplayDefinitionであり、汎用Game Script VM、JIT、Game向けFFIを持たない。
9. Game制作中のEngine、Editor、公開SDK、Validator、Policyは署名済みread-only baselineである。詳細は[AI Security／Approval](../01-governance/ai-security-approval.md)だけが決定する。
10. 設計名、Tier、Phase、比較対象Engineの存在を「利用可能」の証拠にしない。利用可能性はActivation Evidenceだけで決める。
11. 契約を固定し、最小vertical sliceを作り、実測し、上位bottleneckだけを最適化し、回帰Gateを通した後に昇格する。独立した「最後に最適化するPhase」は置かない。
12. Source intent、Gameplay fidelity floor、Visual StyleをTargetごとに分裂させない。Target差は各OwnerのCompilerとProfileが解決する。
13. 英語の正規技術語彙、localized presentation、User／Project原文を分離する。Editor表示言語、AI返答言語、Game Project source localeを一つのglobal languageへ統合しない。

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
| C3 Advanced | 大規模World、Online、特殊Device、特殊Genre等の独立した汎用化 | 専用仕様、Threat Model、Benchmark、Pack／Target Gate |

C3は「C2より良い実装」の総称ではない。各Capabilityが別仕様、別Prototype、意味同等fallback、個別Qualificationを持つ場合だけ着手する。

### 3.2 Activation state

Activation stateは実ArtifactのEvidenceに基づくTarget別の現在状態であり、Tierと分離する。正本はcurrent `ProductOperationalStateSnapshotV1`にある`CapabilityTargetActivationStateV1`の`{capability_id, target_id}`行であり、Capabilityに一つだけ置くscalar stateを禁止する。

| State | 意味 | AI／Editor／Projectの扱い |
|---|---|---|
| not_activated | 契約未完成、計画だけ、または利用可能なCandidateがない | 通常非表示またはGap説明のみ。Project利用、Package収録、成功扱いを禁止 |
| candidate_locked | CandidateのSource、Contract、Toolchain、Target、Policy、Evidence入力がexact hashで固定された | 隔離Staging／Prototypeだけ。Production ProjectへActivation不可 |
| qualified | 指定Target／Profileの必須Gateに合格し、制約とReceiptが固定された | allowlistされたProjectで個別承認付き利用可。別Targetへ一般化不可 |
| production | Release closureと継続Support Gateまで合格した | Production Manifest範囲で通常選択、Package、Release可 |

通常遷移は次の一方向だけである。

    not_activated -> candidate_locked -> qualified -> production

集約表示はTarget別stateからだけ導出する。Capability Registryで`required`または`optional`としたTargetのoperational state rowが一件でも欠ける場合はclosure全体をrejectし、`not_activated`を合成しない。closure成功後、`required`行だけの上記順序の最小stateを集約stateとする。`optional`はTarget別support表示だけに使い、集約へ含めない。optional機能をProjectが明示選択する場合はそのTarget行が要求state以上であるか、同じTarget／Candidateへ束縛されたfreshな意味同等fallback Receiptが必要である。`excluded`はrow自体を持たず選択不能である。集約値を保存、手動編集、Target別Receiptの代用に使用してはならない。

保存`state`と実効availabilityを同一視しない。`effective_availability`はcurrent rowに加え、全Receiptのread-time freshness、revocation、Target／Candidate／input hash、license／Provider statusを毎Queryで検証して導出する。required Evidenceが`fresh`以外なら保存stateが`qualified | production`でも即時`unavailable_due_to_evidence`とし、新規選択、Product label、Package、Promotion、Releaseをfail closedにする。署名済みdeactivation transitionは監査上の追随Recordであり、この即時停止を待たせない。

段階飛越を禁止する。Security incident、重大回帰、license失効、Provider停止、Evidence期限切れでは、productionからqualified、qualifiedからcandidate_locked、candidate_lockedからnot_activatedへ降格できる。降格後は新規選択とPromotionを止め、last-known-good、Migration、Rollback、利用中Projectへの影響をOwnerが明示する。

AIは次を推論してはならない。

- Roadmap記載、Tier、Phase、比較対象Engineの対応を「利用可能」と解釈する。
- OS、GPU、Store、ProviderのbrandからTarget対応を推測する。
- 未対応Capabilityを別機能で近似して成功扱いする。
- Multi-User、Gameplay Networking、Platform Serviceを相互に代用する。
- Presentation結果をauthoritative Gameplayへ逆入力する。

## 4. 2D／3D Capability portfolio

2Dと3Dは同格のFirst-class Runtimeであり、2Dを奥行き0の3Dとして実装しない。Asset、Input、Audio、UI、GameplayDefinition、AI Authoring、Build、Save、Diagnosticsは共有する。World Space Profileが一Worldのstructural spatial authorityを選び、Rendering、Physics、Navigation、Animation Authoringはそのexact refまたは互換性だけを消費する。

次表はProduct horizonだけを示す。SubsystemのField、Backend、既定値、Budget、Runtime順序は各Ownerへ移し、本表から推測してはならない。再編時点で実装Receiptを伴わないEntryのActivation stateはnot_activatedである。

| Portfolio group | C0 | C1 | C2 | C3／将来 | 主Owner |
|---|---|---|---|---|---|
| 共通Authoring／Project | 正規Project state、ChangeSet、Asset、Gameplay model | 手動とAIの安全な往復 | 大規模Project、拡張Authoring | Multi-User／UGCは現Future portfolio未登録・非対応。検討時はmembership revisionでEntryを先に追加 | [Authoring Owners](#91-authoring) |
| 共通Runtime | scheduling、lifetime、capacity、debug | 2D／3D sliceを同じ契約で実行 | fault／soak／scale Qualification | distributed／onlineは別Entry | [Runtime Owners](#92-runtimesimulation) |
| 2D Presentation | 2D Render／Camera／Material契約 | compact 2D command RPG | lighting、VFX、style、fallback | 特殊表現は個別Entry | [Rendering Owners](#93-rendering) |
| 2D Simulation | Collision、Physics、Grid Navigation、Animation契約 | RPG World移動とcommand battle state | Target別Kernel／content Qualification | Advanced motionは別Entry | [Simulation Owners](#92-runtimesimulation) |
| 3D Presentation | Mesh／World／Camera／Material／Light契約 | compact 3D technical Reference | advanced lighting、post、environment、VFX | large worldは登録済みFuture。Virtual Productionは現Future portfolio未登録・非対応 | [Rendering Owners](#93-rendering) |
| 3D Simulation | Collision、Physics、Navigation、Animation契約 | 3D vertical slice | Target別Kernel／content Qualification | Vehicle、Ragdoll等は別Entry | [Simulation Owners](#92-runtimesimulation) |
| Player I/O | Input、Audio、UI／Text／Accessibility契約 | TitleからResultまでのoffline loop | Platform／locale／device matrix | Online serviceと分離 | [Platform Owners](#94-platformpacks) |
| Platform | Windows、Mobile共通境界 | Windows offline package | Android／Apple offline | Linux／macOS／Web／Console／XRは独立Future Target Program。active supportではない | [Platform Owners](#94-platformpacks) |
| Packs | 4層依存とFeature Capability境界 | RPG Feature／Genre Packと独立したtechnical fixture | 複数Genre／構造で再利用するFeature Pack | Multiplayer CapabilityはGenre-neutralなOnline Gate後 | [Pack Owners](#94-platformpacks) |

### 4.1 Algorithm／Performance optimization portfolio

次表は同じTarget／Profile／Contract／fixtureで意味同等性と実測差を比較するためのProduct上の優先順位であり、実装順、Work Package、日程、Activation、採用済み経路を表さない。`planning_disposition`は本表だけの列挙で、`candidate_for_qualification | conditional_research | deferred`以外を許さない。行番号、Priority、対象名からStable ID、Operation、Capability、Backend selectionを生成してはならない。

| Priority | 対象 | Product上の候補範囲 | `planning_disposition` | 期待と停止条件 | 主Owner |
|---|---|---|---|---|---|
| 1 | ECS／Memory | chunk SoA、zero-allocation callback、query cache、hot／cold分離、8／16／32 KiB candidateの同一fixture比較 | `candidate_for_qualification` | engine全体へ波及する。semantic oracle、allocation／fallback 0、memory／latency Gateを満たさない候補はrejectする | [Performance](../04-runtime/performance-capacity.md)、[ECS](../04-runtime/entity-component-system.md)、[Memory](../02-foundation/memory-pointers.md)、[Runtime ECS Decision](../decisions/2026-07-22-runtime-ecs-contract.md) |
| 2 | Physics／Collision | Box2D／JoltのTarget別worker候補、sleep profile、Jolt bulk insertion、可能な限り早いdeclarative collision filter | `candidate_for_qualification` | public physics／collision semantics、event順序、Save／Replayを変えずにstep tail latency、挿入、filter costを改善する。Backend固有制約はOwnerが固定する | [Physics](../05-simulation/physics.md)、[Collision](../05-simulation/collision.md) |
| 3 | Navigation | canonical A*のbounded memory再利用、version付き結果cache、Detour sliced query／tile publish境界 | `candidate_for_qualification` | path queryのP95／P99とallocationを安定させる。path cost、tie-break、status、artifact versionが一致しない候補を採用しない | [Navigation](../05-simulation/navigation.md) |
| 4 | Rendering | transient aliasing、GPU culling＋indirect draw、HZB、Target別render-pass最適化 | `candidate_for_qualification` | 大量object sceneで有効なTarget／Profileだけを選ぶ。resource lifetime、visible-set oracle、barrier validation、GPU／CPU時間がGateである | [Render Graph](../06-rendering/render-graph.md) |
| 5 | 高度な探索 | 条件を満たすGridのJPS、immutable weighted graphのALT | `conditional_research` | canonical A*と同じpublic contractを証明できる入力領域だけ。意味が変わる場合は透明な最適化にせず新しいalgorithm／profile contractとする | [Navigation](../05-simulation/navigation.md) |
| 5後段 | 再計画／階層探索等 | D* Lite、HPA*、flow field、local avoidance | `deferred` | repeated replanning、hierarchy abstraction、群集挙動の契約を先に定義する必要があるため、現Portfolioの高速経路として扱わない | [Navigation](../05-simulation/navigation.md) |

Priorityは影響範囲、期待ROI、意味契約の安定性、依存順を表す。数値改善を予測値だけで昇格せず、[Performance／Capacity §8.4](../04-runtime/performance-capacity.md#84-algorithm-optimization-candidate-qualification)のsealed Projectionとfresh Receiptがない行は`not_evaluated`のままにする。

### 4.2 AI production understanding boundary

AIがGame制作時に理解する標準経路は、巨大なMarkdown、全Project dump、live native objectを直接読む方式ではなく、Ownerが正規状態から導出したbounded typed Projectionと登録済みsemantic toolである。AIへ渡せる制作文脈は、目的別に`ArchitectureExplainProjectionV1`、`GameUnderstandingClosureV1`、`AiTaskContextCapsuleV1`、`EditorContextSnapshotV1`、`OptimizationDecisionProjectionV1`へ分割し、各Projectionはschema version、source revision、exact ref／hash、invalidation conditionを持つ。query型Projectionだけがomitted rangeとcursorを明示し、complete型Projectionは欠落Fieldを許さない。Project Sourceと設計文書の必要な抜粋はread-only inputとして利用できるが、Projectionを補完する権威、暗黙default、直接write対象にはしない。

```text
Canonical Project／Contract／Evidence
  -> Owner validation
  -> bounded typed Projection
  -> AI read／explain／propose
  -> registered semantic Operation
  -> ChangeSet
  -> preview／validate／approve／commit
  -> read-back／Receipt／new revision
```

Toolの表示、Provider schema適合、AIの説明成功はAuthorityを付与しない。変更はactiveなOperationが生成したChangeSetだけを通し、raw file edit、任意`set_property`、native handle、memory、shell、`eval`、汎用Python、無制限Project traversal、AI自身によるthreshold緩和／candidate昇格を禁止する。Commit後は新しいProject revision、semantic hash、必要なtest／Receiptをread-backし、要求した変更と一致しなければ成功にしない。現在の設計文書が`review`でOperation集合またはCapabilityが`not_activated`の領域は、AIが説明できても制作利用可能とは扱わない。

current closureではimmutable `ArchitectureInventoryV1`、Architecture Explain Projection、optimization説明用Eval Fixture／Receipt、該当`AiTaskContextCapsuleV1`、optimization explain／propose／select Operationが未materializeまたは未Activationである。したがって§4.2はAI制作経路の目標契約であって現在利用可能な機能一覧ではない。現在状態は[Architecture Governance §4.1／§5](../01-governance/architecture-governance.md#41-現在のapproval-readiness-snapshot)、Capsule schemaとbinding条件は[AI Security／Approval §5](../01-governance/ai-security-approval.md#5-beginner-questionsassumptions理解条件)、optimization ProjectionとEval条件は[Performance／Capacity §8.4](../04-runtime/performance-capacity.md#84-algorithm-optimization-candidate-qualification)を正本とし、本書で件数またはActivationを上書きしない。

Target／ProfileごとのPrimary candidateは、qualified selectionが存在する場合にexact一件、存在しない場合は0件である。reference implementationはsemantic oracle、または明示的に別Qualificationされたsemantic fallbackになり得るが、旧経路と新経路の暗黙併載、deprecated reader、alias、silent fallback、runtime自動切替を正当化しない。benchmark candidateはdispatch不能である。選択を変える場合はprofile／algorithm revisionを新設し、public semanticsが変わる場合は新しいcontract versionとして扱う。Source dataは保持するが、旧挙動を温存する互換layerは作らない。

## 5. MVP scope

MVPはEngine機能網羅版ではなく、AI Authoringの安全な往復を証明する製品vertical sliceである。

| Milestone | 完了Outcome |
|---|---|
| MVP-A | Phase 4完了。Genre非依存Core holdoutとcompact 2D command RPGを使い、Prompt、質問、Game Brief、First Playable、AI修正、手動修正、AI再編集を一つのProject historyで完走する |
| MVP-B | Phase 6完了。MVP-Aを維持したまま3D Shooterをtechnical Referenceとして追加し、3D固有OwnerとFeature契約の非2D依存を検証する。MVP-BはProduct-facing RPG identityまたはGeneric Core依存を変更しない |
| Technology Preview | MVP-Aに加え、Release ActivationとProduct completion gateを満たす。MVP-Bのtechnical Referenceは独立Evidenceとして報告し、Generic Engine Releaseの前提にしない |

Product MVPという語はMVP-AのRPG vertical sliceだけを指す。`MVP-B`はcurrent source Registryとのtraceabilityを保つ既存milestone labelであり、本方向では3D Technical Referenceとして扱う。destination Product DefinitionでlabelまたはIDを変更する場合も§5.1.1のatomic migrationを必要とする。

### 5.0 Closed RPG Reference MVP boundary

Product MVPはMiraikanaiの差別化であるtrusted typed AI Authoring loopを優先する。compact RPGはGame Briefからstructured game、data-driven rule／content、AI／manual round-trip、dialogue／UI／Localization／Accessibility、Save／long-lived state／migration、inventory／progression／cross-system referenceのProduct仮説を強く覆う。compact Shooterがより強く覆うhigh-frequency Input／Collision／PhysicsとECS／Rendering frame-budget stressは独立technical fixtureに残す。この選択はRPGが普遍的に最良のEngine benchmarkであるという主張ではない。

MVP First Playableのcomplete pathを次に固定する。

```text
Title／Continue／Settings
  -> Town
  -> Field
  -> Dungeon
  -> Boss battle
  -> Result／Ending
```

#### 5.0.1 Title-to-Result scene／screen transition definition closure

Product上の「シーン管理」はEditor表示用の総称であり、単一の`SceneManager`型、表示名、filesystem path、配列indexをRuntime authorityにしない。MVPの正規分解は次である。

| Product上の切替 | 正規Owner／契約 | MVPでの一意な扱い |
|---|---|---|
| Title起動 | Project／Runtime Entry | Targetのdefaultとして選択されたUI-only Runtime Entryを一件だけactivateし、その`ui_document_ref`をUI Screen Stackのrootにする |
| Title ↔ Settings | UI | 同じUI-only Runtime Entry内のtyped Screen navigation。Settingsを別World、Stage、Runtime Entryにしない |
| New Game | Runtime／Scheduling | Title UIのregistered `UiCommandId`をauthorized game logic／orchestratorがexact gameplay Runtime Entry refへ解決し、`RuntimeEntryTransitionPortV1`へ`replace_branch` requestを提出する |
| Continue | Persistence／Save＋Runtime／Scheduling | target `SaveCatalogV2`のexact slotから`RuntimeSessionSaveBundleV1`を解決し、保存されたRuntime Entry／Package／State owner projection／optional World／Stage projectionを検証して同じRuntime Entry transitionへ渡す。current V1のdisplay metadataから復元先を推測しない |
| HUD／Pause | Project／UI | gameplay World Runtime Entryへexact `RuntimeEntryPresentationBindingV1`でroot HUD UiDocumentを束縛し、Pauseを同じScreen Stackのmodalとしてpush／dismissする。既存`RuntimeEntryPointV1.entry_kind=world`の`ui_document_ref=null`規則を変更せず、WorldまたはStage stateをUIが直接変更しない |
| Town → Field → Dungeon → Boss | Scenario／Stage＋World | finite gameplay progressionはtyped Stage transition、同一World内の位置変更はWorld spatial transitionを使う。Scene Documentの切替をgame progressionと同一視しない |
| Boss → Result／Ending | Scenario／Stage＋Runtime／Scheduling | completion outcomeからexact Result UI-only Runtime Entryへbranch replacementする。ResultをShooter Game Flow、World、Loading UIの暗黙stateにしない |
| Result → Title | Runtime／Scheduling | registered UI commandからexact Title Runtime Entryへbranch replacementする。新しいPlay Sessionを暗黙生成せず、同じSession内でbranch generationだけを進める |

Runtime Entry replacementはdestination branchを非公開stagingで検証し、次のeligible `T00_BoundaryApply`で一回だけpublishした後にsource branchをreverse teardownする。失敗、stale generation、hash不一致、依存不足、重複request、commit前cancelではsource branchとlast-valid generationを維持し、partial World／UI／Stageを表示しない。commit後cancelは成功を巻き戻さずtyped rejectionとする。Loading表示は進行状況のprojectionであり、遷移成否またはdestination identityの正本ではない。

MVP acceptanceはclean launchから`Title -> Settings -> Title -> New Game -> HUD -> Pause -> Stage progression -> Save -> Result -> Title -> Continue`を同一Candidateで完走し、Continue後に保存されたexact Runtime Entry、Stage instance projection、authoritative state digestへ復元する。keyboard、controller、screen reader、manual test、AI testは同じtyped command／requestと最終Stable ref集合へ収束しなければならない。Scenario／Stage PackなしでもUI-only→World Runtime Entry replacementは成立し、UI-only／headless packageとSave bundleは偽Worldなしでvalidである。

本節はtarget definition closureであり、Operation／Service／Provider／MCPのcurrent activation、Product Phase、Work Package state、Shooter RegistryとRPG destination Product Definitionの切替を行わない。実装開始は既存のActivation／Definition Migration／Qualification gateに従う。

current `SaveCatalogV1`はexact Save Bundle refを持たず、Runtime Entry transition、Presentation Binding、Screen Stackもtarget reviewであるため、本節のContinue／Title-to-Result closureは現時点で`not_activated`である。V1 metadataから仮Continueを作る、既存Runtime Entry V1のworld branch意味を変更する、一部Typeだけを先行current化することを禁止する。

Fixtureは一つのWorld compositionにTown一つ、Field一つ、Dungeon一つ、Boss destination一つを持ち、Loading、Stage transition、Save／Load、ResultはRPG-private代替でなく既存Generic Ownerを使う。Reference Game fixtureは次をすべて含む。

- player-controlled protagonist一人と固定companion一人。recruitまたはparty reorderは持たない。
- `Attack`、`Skill`、`Item`、`Defend`を持つ一つのdeterministic command battle flow。
- regular enemy definition四件とBoss definition一件。
- 宣言済みGameplay valueを変えるdeterministic level-up一回以上と、visibleかつtyped effectを持つequipment変更一回以上。
- Skill／spell definition三件以上と、bounded durationおよびdeterministic recoveryを持つnegative status effect一件以上。
- mandatory Quest一本、Quest flag一件と後続dialogueを変えるbounded choice一件。second endingは作らない。
- currency、price、inventory capacity、accept／rejectを通すbuy-only Shop path一件。
- checkpoint一件とactive battle外のmanual Save／Load。
- locale-invariant gameplay identityを保つ`en-US`／`ja-JP` presentation。
- Windows desktop Targetのkeyboard／controller input。
- 同じsemantic fixtureに対するcomplete manual Authoring runとcomplete AI／manual round-trip run。

上記件数はReference Game fixtureのclosed boundであり、Generic Core limitまたはpublic engine maximumではない。AI生成GameplayDefinitionを実際の挙動へ使い、初心者はDefinition-firstと事前Qualification済みNative／Shader Packだけで完走でき、人間の手動変更を保持してAIが追加編集できる。新規AI生成Project Native／Shader SourceのActivationはMVP-Aの成功条件にせず、Phase 5の独立Gateで検証する。

```text
Game Brief
  -> approved RPG Reference scope
  -> GameSpec
  -> typed RPG and generic Source documents
  -> ChangeSet
  -> preview／validate／approve／commit
  -> cooked Runtime artifacts
  -> play／save／load／replay
  -> package／install／offline completion
  -> sealed Evidence
```

RPG battleはEngine全体のturn-based Simulation cadenceをactivateせず、qualified fixed Simulation profile上のRPG-owned deterministic state machineとしてcommandを既存Runtime boundaryで受理・適用する。Physics、Navigation、Animation、VFX、Input、Replay、Debugはcurrent cadenceを維持し、alternate global cadenceは別Product Capability、全consumer migration、Target別Qualificationなしにactivateしない。

MVPはjob／class change、recruitable／reorderable party、branching ending、crafting、random loot table、procedural dungeon、open-world streaming、complex economy simulation、unrestricted scripting、multiplayer、voice acting／cinematic cutscene、commercial-quality bulk Asset generation、Android／Appleその他Targetをcompletion dependencyとすることを除外する。後続Feature Packを禁止するものではなく、最初のProduct proofへ未Review scopeを混入させないための境界である。

RPG presentationはbattle、Quest、inventory、progression、economyのauthoritative stateを書かず、UIがtyped requestを発行し、owning Gameplay Systemだけが既存Runtime boundaryでvalidate／publishする。AIはbounded semantic projectionを説明し変更を提案できるが、RPG Capability activationを捏造し、Owner validationを迂回し、Runtime memoryを変更し、Shooter成功をRPG Evidenceへ読み替えない。

RPG固有failureはsource preservingかつtypedである。missing RPG owner contractはProduct Definition materialization、missing／stale Feature Pack refはCook／Packageをrejectする。invalid battle command、inventory／currency capacity failure、Quest／dialogue ref mismatchはlast published stateまたはlast valid Project revisionを保ち、partial transactionを作らない。Save schema／content mismatchは承認済みmigrationまたはtyped failureを要求し、類似recordをloadしない。locale mismatchはStable ID、rule、balance、Save identityを変えず、Asset license／provenance欠落はaffected packageをblockして別Assetへsilent substitutionしない。Target／performance Evidence欠落は該当Capabilityを`not_activated`に保つ。

Qualificationは、全Genre／optional Feature Packを外したGenre-neutral Core holdout、TitleからEndingまでのRPG Reference acceptance、高頻度Input／Collision／Physics、ECS、Rendering、soak／failure atomicity／clean packageのtechnical stress fixtureを別Evidence categoryとして保持する。Core holdoutはgenreless WorldとWorldless UI／logic、manual／AI Authoring、Save、Cook、Package、Diagnosticsを通す。RPG acceptanceはdeterministic battle／Replay、progression、equipment、Quest、choice、Shop、Save round-trip、`en-US`／`ja-JP` semantic invariance、manual／AI changeのtyped state収束を通す。一CategoryのReceiptを別Categoryの代用にせず、RPG Referenceをmaximum-scale performance benchmark、technical action fixtureをProduct MVPとして扱わない。

MVPに含める製品能力は、自由Prompt、Blocking／High質問、Game Brief、GameSpec、typed ChangeSet、構造化Scene／Rule／UI／Asset編集、GameplayDefinition、事前Qualification済みNative／Shader Packの選択、Engine生成Diff、Approval、Commit、Undo、Replay、競合防止、First Playable、Cook、Package、Install、offline起動、diagnosis、support bundle（正本は[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md) §14の`SupportBundleV1`）、data resetである。Project C++と`bounded_hlsl`の新規Source隔離検証／Code Owner approval、ならびに`typed_ir`のIR・behavior coverage・Target Qualification gateは、Phase 5の`requirement.product.project-source-activation`が所有する。

MVP Editorは`en-US`を正規source／最終fallback、`en-US`と`ja-JP`をC1表示localeとする。初回表示は対応するsystem localeへ従い、AI返答はEditor表示へ既定連動しながらUser／会話単位で上書きできる。Game Projectの`source_locale`は独立し、Editor localeまたはAI返答localeから推測しない。設定Schema、fallback、Catalog、semantic invariance、Eval Gateは各Owner文書を正本とする。

MVPから除外する。

- 商用品質の全Asset自動生成、複数Genre同時対応。
- job／class change、recruit／party reorder、branching ending、crafting、random loot、procedural dungeon、complex economy。
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
| `critical_path` | Control Plane → ECS E0 →（Headless AuthoringとD3D12を並行）→ Editor Runtime → Genre非依存Core holdout → compact 2D RPG → Authoring MVP-A。Project Source ActivationとShooter technical qualificationは独立lane |
| `scope_reduction_order` | C2 advanced rendering → non-RPG Reference coverage → 3D technical Reference → mobile shipping。MVP-AのGenre非依存Core holdoutとRPG Reference contractは削除しない |
| `risk_owner` | `mirakan.arch.product-plan` |
| `review_cadence` | 各Phase exitで更新し、仮定を同じCandidateの実測Receiptへ差し替える |

`team_assumption_state=unfixed`では担当者名、完了日、同時実行lane数を出力しない。日程または予算の提示が必要になった時点で、team composition、利用可能runner／device、稼働制約、外部依存を入力として別の承認済みplanning revisionを作る。scope削減は上表の順序でProposal化し、Phase 4のMVP-A Requirement、Phase 5の独立Project Source Capability、security／package／support closureを暗黙に落とさない。

### 5.1.1 RPG-first MVPとcurrent Product Definitionの分離

RPG-firstは承認済みProduct target directionであるが、current Registry row、Installed Product composition、active Operation集合、Capability activation row、Work Package lifecycle headをこの文書編集だけで変更しない。現行Shooter IDをRPGへrenameし、Shooter ReceiptをRPG qualificationへ流用し、未定義のRPG Stable ID／Schema／Capability／Operation／Work Packageを仮登録することを禁止する。

RPG Product Definitionをmaterializeする前に、command battle、actor progression、inventory／equipment、dialogue／quest、currency／shopの再利用可能Owner設計、RPG Genre composition、Reference Game fixture、source／destination全rowをexact分類するProduct Definition Migrationをそれぞれ承認する。Migrationは一つのatomic destinationとして適用し、途中のRegistry状態、CoreへのGenre依存、dual current Product Definitionを作らない。それまではcurrent Shooter Registryをsource baselineとして保持し、RPG Capabilityを`not_activated`から成功可能な状態へ推測しない。

RPG battleはcurrent qualified fixed Simulation profile上のdeterministic state machineとして設計し、Engine全体のturn-based cadenceをMVP要件にしない。alternate cadenceは引き続きFuture capabilityであり、全consumer migrationとTarget別Qualificationなしにactivateしない。

### 5.2 Editor Reference DesignのDefinition Closure

独自Editorの画面を実装順、既存Engineの画面名、または見た目だけのmockupから決めない。各Panelが人間、AI、UI Automationに同じ対象・状態・可能な操作を示すため、Reference DesignをPhase 2より前にC0の設計定義として閉じる。AIは画面を画像認識、mouse macro、UIA、screen coordinateで操作せず、Semantic Snapshotとtyped Command／ChangeSetを正規経路にする。この閉包は新しいPanel、Capability、Operation、Work Packageをactivateせず、各既存Ownerが実装する際の受入基準だけを追加する。

当初の五系統は有効だが、排他的な`panel_family`にはしない。Panelは複数のsurface roleを合成できる。`Source`や`History／Diff`をDiagnosticsへ、`Task／Proposal／Approval`をAI Partnerへ暗黙に吸収すると、AIと人間が同じ画面の目的を誤認するため、次の七roleと二つの横断Referenceを正本とする。

| Reference | 代表Panel | Definitionで先に閉じる事項 |
|---|---|---|
| Tree／List | Outliner、Asset Browser | Stable ID selection、階層／行、virtualization、filter／sort、multi-select、keyboard／UIA ItemContainer／VirtualizedItem |
| Property／Form | Inspector、Import Inspector | label／control／unit、mixed・unset、draft／read-only、validation、field単位Diff、commit境界 |
| Canvas | Scene、Game、UI Designer | world／screen座標の区別、overlay、gizmo、camera、selection、safe area、runtime preview境界 |
| Graph／Timeline | Gameplay Graph、Animation Graph、Animation Timeline | viewport LOD、edge／keyframe selection、exact timebase／frame表示、virtual query、keyboard代替、50k node／100k time item時のsemantic paging |
| Diagnostics | Problems、Console、Profiler、Build、Replay | severity、recorded／current、gap／redaction、source／target ref、時系列、再現・retry入口 |
| Source／Text／Diff | Source、History／Diff、Review | 日本語／ASCII／codeのfont role、IME、selection、before／after、conflict、source revision、read-only Source境界。C1はread／select／copy／reviewを先に閉じ、編集・適用・PromotionはSource Capability activation後だけにする |
| Task／Proposal／Review | AI Partner、Question／Decision、Task／Receipt、Approval | intent、assumption、proposal、validation、approval、progress、receipt、stale、accept／rejectの単位。Task／OperationTask、review中／署名済み判断、Proposal／current Projectを混同しない |
| Shell／Docking／Navigation | Menu、Command palette、Toolbar、Status、Dock、Workspace | panel discovery、five-zone dock preview、focus scope、multi-window、recovery、最低Scene領域 |
| Transient／State／Accessibility | tooltip、menu、overlay、notification、全Panel共通 | interaction・availability・editability・selection・validation・freshness・authorityの直交state、focus、High Contrast、200% scale、reduced motion |

各Referenceは次を満たすまで「見た目が決まった」と扱わない。

1. Windows Color Modeに追従するLight／Dark、Windowsの四つのbuilt-in Contrast Theme、200% UI scaleで意味が保たれるsemantic color、typography、icon、density、motion、transparency material policyのtokenと、状態を重ねたときの優先順を定義する。
2. `normal / hover / pressed / focus / disabled / read-only / error / warning / stale`を最低集合とし、AI proposal、Engine validation、User selection、Runtime、approval、taskを別axisで表す。色だけ、iconだけ、またはtooltipだけで意味を伝えない。
3. 人間の表示名とは別にStable target ref、canonical field ref、revision、typed action、disabled reason、virtual collectionのcontinuationを持つAI／Accessibility Semantic projectionを定義する。pixel、screen coordinate、draw primitiveはCommand targetにしない。
4. 全interactive Controlとsemantic composite rootをclosed Widget Pattern Registryのexact `pattern_id`へ一件解決し、anatomy、token、input、focus、state axis、Semantic role、UIA pattern、Action投影を同じinstanceへ束縛する。Patternは`ai_actions[]`、任意AI callback、Project write capabilityを所有せず、具体的ActionはSemantic SnapshotとCommand Registryが所有する。
5. compact／standard／comfortableのrow、field、padding、text、iconを同じlogical unitとscale式で決め、DPI・font/UI scaleによるclip、overlap、off-screen controlを0件にする。
6. motionは状態遷移を隠す飾りにせず、Dock preview、Panel、Task、Proposal、scrollbar chromeだけにboundedなdurationを与える。C1はfirst top-level HWND生成前に`UISettings.AnimationsEnabledChanged`をsubscriptionし、起動時および同event／全`WM_SETTINGCHANGE`後に`SystemParametersInfoW(SPI_GETCLIENTAREAANIMATION)`を再読込する唯一のclient-area判定とする。`FALSE`ならnon-semantic motionを0 ms、最終visual state、static task stageへ縮退する。transparencyは`UISettings.AdvancedEffectsEnabled`／`AdvancedEffectsEnabledChanged`を尊重しても、C1ではimmutableな`effects.editor.opaque-only@1`を維持し、desktop透過、backdrop、Mica、Acrylic、blurを使わない。custom scroll chromeは`UISettings.AutoHideScrollBars`／`AutoHideScrollBarsChanged`を利用者設定として尊重し、`FALSE`はoverflow中の`persistent`、`TRUE`は`ScrollChromeContractV1`の`auto_hide`だけに解決する。visual chromeがhidden／indicatorでもkeyboard／UIA Scroll、logical offset、layout、selection／focus、AI typed command経路を変えない。app-local override、`SPI_SETCLIENTAREAANIMATION`、`UISettingsController.SetAutoHideScrollBars`、registry／Settings write、Progress不明時の擬似percent、reduced motion時の必須animationを禁止し、`TRUE`への復帰後もcancel済みeffectを再生しない。

   - C1のin-window notificationは`SystemParametersInfoW(SPI_GETMESSAGEDURATION)`だけでauto-dismiss deadlineを決め、固定秒数、本文長、severity、hoverからdurationを補正しない。actionなしinformationと永続owner recordを持つcompleted以外は`manual_only`であり、warning／error／blocking／action付き通知はowner recordを必須にする。actionなし短文feedbackはStatus bar一件、状態またはactionを持つnotificationは同じowner Panel内のinline surfaceへ置き、global floating stackを作らない。初期化時と`UISettings.MessageDurationChanged`／全`WM_SETTINGCHANGE`後にSPI値を再読込し、`SPI_SETMESSAGEDURATION`、app-local override、OS Toast／background activationを使わない。notificationのdismiss、UIA provider／activity ID、screen coordinateはAction権限やowner state遷移にならず、UIA notificationはredaction済みのcoalesced announcementだけを発行する。
7. Referenceごとにkeyboard、screen reader、UIA、AI typed command、manual editが同じStable ID／revisionへ収束するnegative caseを含むfixtureを用意する。Screenshotは視覚差の補助evidenceであり、semantic／layout／authorizationの合否を代用しない。
8. 最初の統合Referenceは`Production`のReference 01（Outliner／Scene／Inspector／Problems）とし、2560×1440・100% DPI・standard density・`editor_ui_scale=1.00`でDark（E00）とpaired Light（E13）の双方にAuthoring selectionと、同じStable ID／revisionへexactに関連付いたAI proposal、validation、Runtime、Diagnosticを重ねて検証する。ProblemsのEvidence selectionをObject selectionへ偽装しない。これは新PanelやCapabilityをactivateせず、各roleの視覚・semantic基準だけを先に閉じる。
9. Reference間の状態同期は一つの万能selectionまたはPanel間callbackにせず、Object／Asset／Document element／Evidence／Timeのtyped channel、単一reducer、Panelごとのfollow／pin bindingへ閉じる。select、keyboard focus、reveal、pin、validation、proposal、Runtime、recorded contextを別stateとし、関連対象への遷移はownerが発行したexact relationと登録済みActionだけで行う。
10. 各Reference fixtureはversioned Manifest、明示的なscenario／driver／environment／oracleのtuple集合、再現可能なsourceとvirtual clock、content-addressed Evidence Bundleを持つ。Screenshot、Semantic Snapshot、UIA tree／event、Command／Receipt、performanceを別oracleにし、既存`VerificationReceiptV1`へ束縛する。failing outputからのgolden自動更新、tolerance自動緩和、sleep完了判定、画像だけのpassを禁止する。
11. 最初の具体Manifest `fixture.editor.reference-01@1`はrevision 42のsynthetic source、Production Workspace、14 Environment Profile、9 scenario、input別driver binding、typed checkpoint、七oracleを`coverage.editor.reference-01@1`のexact 166 tupleへ閉じる。runtime read-onlyの否定系はHuman／UIA／AI surfaceでCommandを発行しないbranchと、Gateway直要求をtyped rejectionするbranchを分離する。font／icon、OS／driver／EDID、A／B hardware、comparison／performance profile、Command／Receiptの一件でも未解決ならManifest hashや実行Registry rowを仮生成せず、関連Capabilityを`not_activated`に保つ。全artifactが完成した後だけRegistry rowをpublishし、同じContract set transactionでapplicabilityを`required`へ変更する。完成Manifestが存在してもCapability対象外のsnapshotでは`prohibited`とし、未完成と混同しない。
12. 七oracleのComparison Profile Registryはexact七entryとし、oracle kind、comparator contract、profile ref/hashを閉じる。Visualはpre-present RGBA8 sRGB、1 LSB＋linear aggregate、contrast、critical cue、Reference 01のempty dynamic-region setと30-capture repeatability、UIAはRuntimeId bijectionとtrusted monotonic timestamp値だけのnormalization、PerformanceはA／B別fresh process五run×120秒、nearest-rank P95のmedian、absolute threshold＋5% regression、10分soakを必須にする。初回Baselineとのhash cycleを避けるため、Performance Profileはbaseline非依存のA／B×三workload Harness Qualificationへだけback-referenceし、初回captureではabsolute threshold＋repeatability、Publication後の通常runではabsolute＋relative regressionを評価する。Profile missing時のRunner既定値、event coalescing、hardware混合、平均値、欠測0補完、未承認値に対するrelative pass、tolerance／mask自動拡大を禁止する。
13. Baseline RegistryはCoverage Matrixとexact 166 entryで一致させる。初回はbaseline非依存Execution Definitionで166 artifactを`captured`として観測し、Definition Closureから166件のatomic initialize Change Itemを作り、以後はcurrent `EditorReferenceBaselineHeadV1`／Registryに閉じたreplace Itemだけで変更する。ReviewはGround Truth／Incoming／Differenceとtyped nonvisual diffを同時に示し、selection、focus、viewed、pending decision、signed decision、stale、publicationを別stateにする。Itemごとにdomain ownerとindependent reviewerのdistinct二`ReviewReceiptV1`、24時間以内のfresh authorization、impact setのguard passを要求し、Publication Serviceがcandidate Registry／Manifest／Headをcompileして全166 entryを再検証し、expected source Head CAS／`PromotionReceiptV1` read-back／append-only Change Registry更新を完了した場合だけ新Headにする。`Accept all`、画像click承認、Batch一括Receipt、AI decision、capture=`pass`、closest baseline、tolerance／mask／thresholdだけの緩和、partial initialize、hash cycleを禁止する。

Phase 2はShellと共通UI Runtimeの上記Definition Closureを、実装済み機能だけを対象に`requirement.product.editor-reference-design`として固定する。Reference 01の正確なDock構成、Pattern構成、state scenarioとPanel bindingは[Editor Workspace／UX](../03-authoring/editor-workspace-ux.md)が所有し、font／icon／motion／state token、Widget Pattern Contract／Registry、Cross-Panel State／Selection Contractは[Editor UI Framework](../03-authoring/editor-ui-framework.md)が所有する。Phase 3は手動AuthoringでTree／Form／Canvas／Diagnosticsの合成を、Phase 4はTask／Proposal／Reviewを含むAIの状態差とDiff／Approvalを実際のProject historyで検証する。ただしTask state、`OperationTaskV1` state、Proposal、Validation、Approval、Promotion、Receiptを一つの成功状態へ縮約せず、AIはProposalと説明まで、人間または信頼済みAuthorityだけがApproval／Promotionを行う。Graph／Timeline、Source、Import等の個別機能は対応する既存OwnerのCapability activationに従い、Reference表に載ることだけを利用可能性の根拠にしない。

以下は未決定のまま暗黙fallbackしてはならないDefinition Closure blockerである。完了前にProductのC1 visual／accessibility／performance claimを出さない。

| Blocker | 禁止する早計な扱い | closure owner／受入条件 |
|---|---|---|
| UI／code Font、icon assetのexact version・license・hash・変換形式 | `Segoe UI Variable`／`Yu Gothic UI`のsystem font名、OS build名、Noto／Fluentのrepository名、Screenshotだけで再現可能とみなすこと。CFF2 variable font、別font fallback、emoji／unknown icon、runtime SVG parser、Font 200%の静止画だけでWindows text scale対応済みとすること。Widget／Panel／Command presentationのEditor-owned system icon consumerを別集合として放置すること、source icon名やSVGをUI contractへ直接書くこと。E07–E10／E13のstatic image、起動時palette、scheme表示名だけでruntime Contrast ThemeまたはLight／Dark対応済みとすること。desktop appでunsupportedな`ColorValuesChanged`をtheme通知またはfallbackとして採用すること | Toolchain／Dependencies §2.1＋Editor UI Framework §13.2.1–§13.2.2。Host OS image＋DirectWrite resolved file、Noto Sans Mono CJK JP 2.004 static OTF、Widget／Panel／Command presentationの全Editor-owned system icon consumer=`EditorIconTokenContractV1`=`conversion manifest`のexact mapping、Fluent source ID／variant／usage context、allowlist／converter／16・20 lu output、license／SBOM／notice、同一`style_font_generation`をlockする。Light／Dark、四High Contrast、DPI／UI／Font 200%の既存Reference Profileで日本語／ASCII／code／numeric／pathとiconを照合し、Windows system text scale 1.00–2.25の`TextScaleFactorChanged`でreflow／glyph／UIA更新を検証する。さらに同一running processで`WM_THEMECHANGED`／`WM_SYSCOLORCHANGE`／`WM_SETTINGCHANGE`後にHigh Contrast flagとcurrent Light／Dark modeまたはsystem colorをstable snapshotとして再読込し、Light↔Dark、四built-in themeとuser-customized colorの遷移でnew snapshot／title bar／描画更新とProject／Semantic不変を検証する。一項でも未固定ならPhase 2 visual closureをfail closedにする |
| Windows client-area animation preference | E11のstatic reduced-motion image、`UISettings.AnimationsEnabled`値だけ、`AnimationsEnabledChanged`未subscription、前回値、app-local motion override、`SPI_SETCLIENTAREAANIMATION`、registry／Settings writeから、custom Win32 client areaのreduced-motion対応済みとみなすこと。`TRUE`復帰時にcancel済みmotionをresume／replayすること | Editor UI Framework §13.2.3／§15.5＋Editor Workspace UX §6.9.1。first top-level HWND前に`AnimationsEnabledChanged`をsubscriptionし、初期化時と同event／全`WM_SETTINGCHANGE`後に`SystemParametersInfoW(SPI_GETCLIENTAREAANIMATION)`を再読込してexact snapshot refとEnvironment Profileを照合する。隔離test userの外部controllerで同一running Windowを`TRUE -> FALSE -> TRUE`へ遷移し、event実受信、in-flight dock preview／panel-tab／dock-workspace／indeterminate taskが次presentでstatic final stateになり、Project／Semantic／Command／Receiptをpreference change自身で変更せず、新規triggerだけがfull durationへ戻るEvidenceを記録する。query failureまたはevent subscription failureはcaptureをfail closedにする |
| Windows transparency effects | `AdvancedEffectsEnabled=true`だけ、Mica／Acrylic APIの利用、透過Screenshot、E00–E13の一枚、High Contrastの見た目からcustom surfaceがTransparency effectsを尊重しているとみなすこと。C1のopaque-onlyを将来materialの許可、または`false`を別visual baselineの自動要求へ読み替えること | Editor UI Framework §13.2.4／§15.1＋Editor Workspace UX §6.9.1。`UISettings.AdvancedEffectsEnabled`を初期読込し`AdvancedEffectsEnabledChanged`をsubscriptionしてexact snapshot refをEnvironment Profileへ束縛する。C1は`effects.editor.opaque-only@1`を固定し、High Contrastを含む全surfaceでopaque fallbackを維持する。隔離test userの外部controllerで同一running Windowを`TRUE -> FALSE -> TRUE`へ遷移し、event実受信、snapshot hash、opaque draw descriptor、Project／Semantic／Command／Receipt不変をEvidence化する。read／subscription failureはcaptureをfail closedにする。material追加は別Dependency／policy／fallback／profile ChangeSetなしに許可しない |
| Windows automatic scrollbar preference | `AutoHideScrollBars`のproperty値だけ、visible／hidden Screenshot、scrollbarなしのUIA、app-local常時表示／非表示、`UISettingsController.SetAutoHideScrollBars`、registry／Settings writeからcustom scroll chromeが利用者設定を尊重したとみなすこと。visual chromeのvisibilityをscrollability、selection、keyboard focus、Project write、AI操作権限へ読み替えること | Editor UI Framework §13.2.5／§15.5–§15.6＋Editor Workspace UX §6.9.1。first top-level HWND前に`AutoHideScrollBarsChanged`をsubscriptionし、worker callbackをUI session refreshへqueueしてpropertyを再読込し、exact snapshot／contract refをEnvironment Profileへ束縛する。E00–E13は`FALSE` persistent static profile、隔離test userの外部controllerで同一running Windowを`FALSE -> TRUE -> FALSE`へ遷移する。event実受信、logical extent／viewport／offset、reserved gutter、`persistent`／`revealed`／`indicator`／`hidden` chrome、container UIA Scroll、非scroll axisのUIA値、selection／focus／Project／Semantic／Command／Receipt不変をEvidence化する。read／subscription failureはcaptureをfail closedにする |
| Windows message duration／in-window notification | 固定5秒、文字数・severity・hover由来のduration、`UISettings.MessageDuration`とのAND／OR、static Screenshot、app-local duration、`SPI_SETMESSAGEDURATION`、Toast／notification area／background activation、global floating stack、UIA／activity ID／dismissをAction・owner stateへ読み替えること | Editor UI Framework §13.2.6／§15.6.34＋Editor Workspace UX §6.9.1。`SPI_GETMESSAGEDURATION`を唯一のduration値とし、first top-level HWND前の`MessageDurationChanged` subscriptionと全`WM_SETTINGCHANGE`をrefresh triggerにSPI値を再読込する。actionなしshort feedbackはStatus bar一件、state／action notificationは同じowner Panelのinline surfaceへ置く。E00–E13はexact snapshotと`visible_initial`／persistent owner recordだけを静的比較し、隔離test userの`d1 -> d2 -> d1` runtime probeでdeadline再計算、manual-only不変、UIA kind／processing／activity ID、redaction／coalescing、Project／Semantic／Command／Receipt不変をEvidence化する。read／subscription／UIA event failureはcaptureをfail closedにし、外部App notificationは別Dependency／policy／activation／Evidence ChangeSetなしに導入しない |
| Windows reference hardware、driver、UIのcombined allocation model | node数とMiB上限を別々に掲げて同時成立を推測すること、NVIDIA／AMD差へCPU・monitor・driver driftを混入させること | Performance／Capacity＋Editor UI Framework。共通i5-12400／32 GiB／NVMe、RTX 3060／RX 6600、driver profile、Tree／semantic／text／packetの合計charge式とreference measurementを固定する |
| TSF、OLE drag and drop、UIA virtualizationのlifecycle／thread contract | IME、drag/drop、screen readerを「Win32なので動く」と扱うこと | Editor UI Framework＋Windows。lock、STA、generation、ItemContainer／VirtualizedItem、stale callbackをnegative fixtureで閉じる |
| Widget Pattern Contract／Registry | §9のControl一覧、Panel固有class、Reference screenshotだけからPattern適合を推測すること。未登録variantやPanel固有state axisを許すこと | Editor UI Framework＋Editor Workspace UX。全interactive Control／semantic composite rootのexact Pattern解決、anatomy、token、state、Semantic、UIA、Action投影、必須重畳scenarioをReference 01で閉じる |
| AI semantic actionとVisual evidenceの境界 | screenshot、bounds、Panel名、表示row、Pattern IDをAIの操作権限またはProject write capabilityにすること。Patternへ`ai_actions[]`や任意callbackを持たせること | Editor UI Framework＋AI Security／Approval。Patternはvisual／semantic対応だけを所有し、具体的ActionはCommand RegistryとSemantic Snapshot、変更はtyped action、expected revision、redaction、user-authorized visual evidenceへ分離する |
| Source／Text／Diff Reference Contract | C1のread-only Sourceをeditable editor、UIA／IME、表示行番号、path、外部IDE link、Diff表示だけからProject writeへ読み替えること。read-only、disabled、locked、stale、conflictを一つの表示へ畳み込むこと | Editor Workspace UX＋Editor UI Framework＋Project State＋Source Owner。owner-issued document／revision／range・hunk projection、read／select／copy／edit／apply／commitの分離、外部三者比較、IME／UIAのread-only境界、conflict／stale、Source Capability activation後だけのmutating Action、fixture materialization条件を閉じる |
| Task／Proposal／Review Reference Contract | AI Partnerの会話履歴、Task成功表示、Validation pass、Screenshot、Pattern IDをProposal acceptance、Human Approval、Promotion成功と扱うこと。Task／OperationTask、review pending／signed／stale、Proposal／current Projectを混同すること | Editor Workspace UX＋Editor UI Framework＋AI Security／Approval＋Verification／Provenance。四Pattern、exact ref／hash／revision chain、field／Document／primitive単位accept／reject、partial accept後の再validation、stale／rebase／discard、Approval／Receipt read-back、fixture materialization条件を閉じる |
| Cross-Panel State／Selection Contract | Object、Asset、Graph node、Diagnostic、Runtime objectを一つのglobal listへ混在させること。Panel同士のselection changed callback、focus＝selection、reveal時の暗黙選択、Inspector pin＝lock／authorizationとみなすこと | Editor UI Framework＋Editor Workspace UX＋各projection owner。typed channel、単一reducer、follow／pin、exact relation、revision／generation、AI／UIA event分離を`fixture.cross-panel.attention@1`で閉じる |
| Reference Fixture Manifest／Evidence Contract | Referenceごとの手書きscript、任意sleep、current outputをground truthへ自動採用、hardware差をclosest baselineへ寄せること。ScreenshotまたはAI説明だけを合否にすること | Editor UI Framework＋Editor Workspace UX＋Verification／Provenance。closed Manifest、explicit coverage tuple、typed barrier、複数oracle、baseline変更規則、Evidence Bundle、既存署名Receiptへの一方向bindingを閉じる |
| Reference 01 concrete Manifest | `@1`というID、Markdown表、未計算hash、代表Screenshotだけで実行可能とみなすこと。input固有focus／UIA eventを消して「同じ操作」とすること | Editor Workspace UX＋UI Framework＋Toolchain＋Performance＋Command／Authoring。fixed Stable ID、14 exact Environment Profile、typed clock／oracle subject、surface-disabled／Gateway-rejection分離、166 tuple、materialization／Activation barrierを同じContract setへ閉じる |
| Comparison Profile Registry | 画像tool、UIA client、Performance harnessの既定比較、任意script、正規表現、closest baseline、current output由来toleranceを使うこと。Profile QualificationからProfile自身へback-referenceしhash cycleを作ること | Editor UI Framework＋Editor Workspace UX＋Performance。exact七Profile、typed Difference、Visual数値／repeatability、UIA二Field normalization、A／B別sample／threshold／aggregation／soak、baseline非依存Harness Qualificationを同じContract setへ閉じる |
| Baseline Review／Publication Contract | outputをその場でgoldenまたはpassへ採用すること、画像click／`Accept all`／AIで承認すること、一Batch一署名、初回の部分承認、stale Receipt、`latest`探索、threshold／maskだけを緩和すること | Editor UI Framework＋Editor Workspace UX＋Verification／Approval。baseline非依存initial capture、exact 166-entry Baseline Registry、atomic Change Item、domain owner＋independent reviewer二Receipt、24時間freshness、typed review surface、candidate全166再検証、`EditorReferenceBaselineHeadV1` CAS、Promotion read-back、append-only Change Registryを閉じる |

## 6. 単一Phase sequence

Phaseは次の順序だけをProduct正本とする。Subsystem Ownerはこの順序を増殖、並べ替え、別名化しない。Phase内の技術Task、型、Budgetは各Ownerと実装Planが決める。

| Phase | Product outcome | Exitの要点 |
|---|---|---|
| 0 Foundation契約とToolchain | 契約、Build、Policy、Evidence、測定の最小閉路 | 空Hostとheadless fixtureが固定入力で再現し、未Activationをfail closedにする |
| 1 Headless Authoring Core | Project state、ChangeSet、validate／stage／commit／undo／replay | 同じ入力から同じstate hashを得て、大規模Projectでもbounded queryが成立 |
| 2 Editor Shellと共通Runtime | AIなしでEditor、Runtime、Asset、Debugの共通経路とReference DesignのDefinition Closure | 空SceneをWindowsでplay、save、packageでき、実装済みsurfaceのsemantic／scale／accessibility基準が§5.2へ収束する |
| 3 2D Manual First Playable | Genre非依存Core holdoutとcompact 2D command RPGを手動制作 | TitleからEnding、battle、progression、Quest、Shop、Save／Load、回帰、Windows Target Gateが成立 |
| 4 AI Authoring MVP-A | RPG First PlayableでAIと手動編集の往復。Task／Proposal／Reviewがvalidation、approval、selection、runtime、staleを混同しない | Definition-first／事前Qualification済みPackでMVP-Aが成立し、AI proposalがtyped action／base revision以外からCommitされない |
| 5 外部Agent接続とProject Source Activation | local MCP／conformance済みClientのProposal-only laneと、新規Native／`bounded_hlsl` Source／`typed_ir` laneを分離 | 外部Clientの権限境界、`bounded_hlsl`のCode Owner gate、`typed_ir`のIR・behavior coverage・Target Qualification gateを独立に検証する |
| 6 3D Technical Reference MVP-B | Shooter Genre Packのcompact 3D arenaをtechnical Referenceとして検証 | 3D固有OwnerとFeature契約の非2D依存を実証するが、RPG MVPまたはGeneric Engine Releaseの前提にしない |
| 7 Mobile Platform | Android／Apple offline Target | Store／device／thermal／lifecycle GateをTarget別に通す |
| 8 Production CapabilityとPack | 一CapabilityずつC2へ昇格 | Authoring、diagnostic、fallback、Qualification closureを満たす |
| 9 Runtime生成deny-only境界 | Runtimeからの未許可generation／mutationを拒否し、Project／Save／authoritative Worldを不変に保つ | allowlist、署名baseline、authority、quota、network禁止のnegative fixtureを通す。positive Runtime生成はFuture `planning_only`に留める |

各Phaseは、契約固定、最小fixture、vertical slice、実測、上位bottleneck改善、同一条件の回帰、Capability昇格の順で閉じる。不合格時はSource intentとlast-known-goodを維持し、OptimizationRequiredを状態ではなくDiagnosticとして発行する。Activation stateはcandidate_lockedに留め、利用可能なCandidateを維持できない場合だけnot_activatedへ降格する。

## 7. Promotion、deactivation、Product completion

### 7.1 Promotion gate

Capabilityの昇格には、少なくとも次を必要とする。

1. 一意なOwner、Scope、Target、Requirement、fallback、failure policyがある。
2. SourceとDerived artifact、authoritative stateとPresentationが分離される。
3. AI／Editorが使える完全登録済みMCD Operation identityと、そのnamed inputへ閉じたtyped change primitive、禁止推論が定義される。
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

4章の共通C2と各SubsystemがC2と宣言するCapabilityは、次のread-only projectionへ決定論的に展開する。これはProduct-ownedの保存Registryではなく、approved `CapabilityRegistryV1`とcurrent `ProductOperationalStateSnapshotV1.capability_target_activation_rows[]`を入力とするQuery結果である。

```text
C2CapabilityCoverageMatrixV1 // read-only projection
  row_keys := sort_utf8(
    for capability in CapabilityRegistryV1.entries[]
      where capability.target_product_tier == C2
      for binding in capability.target_bindings[]
        where binding.scope == required || binding.scope == optional
          emit { capability_id, target_id })
  rows[] := exact_join(row_keys, CapabilityRegistryV1, CapabilityTargetActivationStateV1):
    capability_id
    target_id
    owner_work_package_ref
    target_scope
    fallback_ref
    state
    candidate_ref
    receipt_refs[]
    derived_evidence_freshness
```

`target_product_tier=C2`である全Capabilityを例外なく入力集合とし、各`required | optional` bindingからexact `{capability_id, target_id}` keyを一意に導出する。`excluded` bindingは行を生成しない。各keyに対応する`CapabilityTargetActivationStateV1`行が欠落、重複、参照不能、またはC2 Capabilityに対して`row_keys`にないextra Activation keyが存在する場合はprojection全体とProduct labelを拒否し、行の省略、`not_activated`の合成、別Target行の流用を行わない。`state`、`candidate_ref`、`receipt_refs[]`はcurrent operational snapshotからread-onlyで投影する。`derived_evidence_freshness`は保存Fieldではなく、Evidence Ownerのcurrent receipt bytes、input hash、expiry、revocation snapshot、Target identityからQuery時に導出する。matrix固有のstate、Receipt、revision、entry table、`include_in_product_label`を保存しない。Owner、Target scope、fallbackはCapability Registry、defer理由、依存、再検討GateはWork Package Registryからだけ読み、Product labelとmatrix Receiptという二重正本を作らない。

Phase 8の`wp.product.general-coverage-2d`はgenre横断coverageと`capability.product.general_production_2d`を所有するが、projection自体は所有または保存しない。次の三つをすべて、人間の手動Authoring経路とAI Authoring経路の両方で合格した場合だけ公開し、current source RegistryのShooter First PlayableまたはRPG-first MVPのいずれからもC2へ段階飛越しない。

| Playable fixture | Genre固有の完了条件 |
|---|---|
| `fixture.product.shooter-2d` | [Shooter Genre Pack](../08-packs/shooter.md)の`genre.shooter.top_down_2d` core loop、敵、Wave、Boss、score |
| `fixture.product.platformer-2d` | Platformer: 5分以上のTitle→3 room→checkpoint→Result、one-way／moving platform、slope／step／ground snap、continuous collision、room Camera、Flipbook event／hitbox、Game Timer |
| `fixture.product.puzzle-dialogue-2d` | 戦闘に依存しないTitle→3 room→Puzzle／Dialogue／Choice／Item／Interaction→Result、Focus、Text／Font、Timer付きpuzzle、Loading cancel／retry |

三fixtureは共通して、Title-to-Result、cook／package／clean install launch、Save slot／Load／Replay、keyboard／controller／touch、Localization／Accessibilityを同じrunで検証する。WindowsとMobileのQualification ReceiptはTarget別に保持し、一方、別genre、Editor内Play、manualまたはAI片方のReceiptで代用しない。required rowの必須field欠落、要求state未満、未対応Targetが一件でもあれば、個別Capabilityの状態を変えずProduct gateだけをfail closedにする。optional rowはProduct labelの合否へ含めず、Target別supportとして表示する。optional機能をfixture／Projectが選んだ場合だけ、同Targetでの要求stateまたはfresh fallback Receiptを必須にする。

`capability.product.general_production_2d`は登録済みC2 Capability matrixと上記三代表fixtureのbundled Reference Game coverage labelである。これはGeneric Engine Releaseと独立したnonblocking qualification trackであり、三fixture、Shooter／Platformer／Puzzle-DialogueのGenre／Game Project、`wp.product.general-coverage-2d`またはそのReceiptを`ProductReleaseGateRegistryV1.required_*`、`wp.product.production-release-binding`、CX3 shippingの成立条件へ使わない。任意の2D genre、任意scale、大規模World、Online、未登録mechanic、未登録Targetの商用品質を保証しない。三fixture外のProjectは必要Capabilityの明示登録とProject-specific Qualification、またはFuture membership／active migrationを先に完了するまで、このlabelを根拠に対応済みと表示しない。

Local multiplayerは現Active Definitionに存在せず、`future.capability.local-multiplayer-split-screen`のplanning-only候補である。将来activeへ移す場合は、Cameraのsplit viewだけで完了扱いにせず、次のProduct profileでSubsystem closureを追跡する。

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

player assignment、join／leave、Input、Game Flow、UI join prompt／focus、split viewport、Audio listener、[Platform Settingsが所有するlocal profile](../07-platform/ui-text-localization-accessibility.md)とSave bindingを一つのintegrated fixtureで閉じる。いずれかの参照またはReceiptが欠ける場合はsingle-player C2を維持し、Future entryをactiveへ移行せず、local multiplayer対応を主張しない。

## 8. Future portfolio

本節の`FutureCapabilityIncubationRegistryV1`は、将来境界を実装DAGから隔離したまま、再検討に必要な決定とQualification形を失わないための唯一のProduct正本である。

```text
FutureCapabilityIncubationRegistryV1
  entries[]:
    future_capability_id
    owner_document_id
    planning_state
    prerequisite_capability_refs[]
    prerequisite_future_capability_refs[]
    required_decision_kinds[]
    candidate_target_kinds[]
    qualification_fixture_kinds[]
    activation_trigger
    excluded_current_product_claims[]
```

`planning_state`の初期値は全25行`planning_only`である。`required_decision_kinds[]`は`product_scope | authority_model | data_model | threat_model | target_program | provider_selection | licensing | operations | qualification`、`candidate_target_kinds[]`は`headless_server | desktop | mobile | console | web | xr | distributed_cluster`、`qualification_fixture_kinds[]`は`contract | deterministic_simulation | network_fault | scale | target_device | content_quality | security | operations`のclosed値だけを受理する。

| future_capability_id | Owner | planning_state | prerequisite capability refs | prerequisite Future refs | required decision kinds | candidate target kinds | qualification fixture kinds | activation trigger | excluded current product claims |
|---|---|---|---|---|---|---|---|---|---|
| `future.capability.offline-large-world-continuous-streaming` | `mirakan.arch.rendering-world` | `planning_only` | `capability.world.3d; capability.environment.core; capability.product.general_production_3d` | `[]` | `product_scope; data_model; qualification` | `desktop; mobile` | `contract; scale; target_device` | C2 3DのTarget別production Receipt後に、offline World partition、streaming authority、Save整合性、memory budgetの承認Decisionが成立する | `claim.product.feature.general-3d-production; claim.product.feature.offline-large-world-support` |
| `future.capability.alternate-simulation-cadence-and-substep` | `mirakan.arch.runtime-scheduling-lifetime` | `planning_only` | `capability.runtime.scheduling` | `[]` | `product_scope; data_model; target_program; qualification` | `headless_server; desktop; mobile` | `contract; deterministic_simulation; target_device` | alternate fixed／variable／turn-based／explicit-step Cadenceと`SimulationAdvanceIntervalV1`について、Physics／Navigation、Animation、VFX、Input／Replay、Debug hang／step、LOD、Gameplay Feature timer／cadence、Pause、Native ABI、Save、Target budgetを同じProfile hashへ閉じ、60/1以外のCadenceごとに全consumer determinism／Save／Replay／Target Receiptが揃う | `claim.product.feature.alternate-simulation-cadence; claim.product.feature.explicit-step-simulation; claim.product.feature.configurable-physics-substep` |
| `future.capability.headless-dedicated-server-session-transport-replication` | `mirakan.arch.runtime-scheduling-lifetime` | `planning_only` | `capability.runtime.scheduling; capability.runtime.ecs-e3-cook-load` | `[]` | `product_scope; authority_model; data_model; threat_model; operations; qualification` | `headless_server; distributed_cluster` | `contract; deterministic_simulation; network_fault; security; operations` | headless Target programとserver authority、session transport、replication contract、運用責務の承認Decisionが揃う | `claim.product.network.multiplayer; claim.product.network.dedicated-server-support` |
| `future.capability.small-cooperative-multiplayer` | `mirakan.arch.product-plan` | `planning_only` | `capability.runtime.scheduling` | `future.capability.headless-dedicated-server-session-transport-replication` | `product_scope; authority_model; threat_model; qualification` | `desktop; mobile; headless_server` | `deterministic_simulation; network_fault; target_device; security` | dedicated server／replicationのqualified Receipt後に、Genre-neutralなplayer count、join／leave、Save、host migrationのProduct Decisionが成立する | `claim.product.network.cooperative-multiplayer` |
| `future.capability.rollback-competitive-networking` | `mirakan.arch.product-plan` | `planning_only` | `capability.runtime.scheduling` | `future.capability.headless-dedicated-server-session-transport-replication` | `product_scope; authority_model; data_model; threat_model; qualification` | `desktop; mobile; headless_server` | `deterministic_simulation; network_fault; scale; security` | Genre-neutralなdeterministic simulation、input delay、rollback window、anti-cheat、ranked operationsの承認Decisionとprototype Receiptが揃う | `claim.product.network.rollback-networking; claim.product.network.competitive-networking` |
| `future.capability.large-session-network-shooter` | `mirakan.arch.pack-shooter` | `planning_only` | `capability.gameplay.combat; capability.gameplay.ranged_combat; capability.gameplay.encounter_spawn; capability.gameplay.scoring; capability.gameplay.pickup_grant; capability.gameplay.interaction; capability.gameplay.character_locomotion; capability.gameplay.path_following; capability.gameplay.scenario_stage; capability.domain.shooter-2d; capability.domain.shooter-3d; capability.runtime.scheduling` | `future.capability.headless-dedicated-server-session-transport-replication; future.capability.rollback-competitive-networking` | `product_scope; authority_model; data_model; threat_model; operations; qualification` | `headless_server; distributed_cluster; desktop; mobile` | `deterministic_simulation; network_fault; scale; target_device; security; operations` | 二つのGenre-neutralなFuture前提がactiveへ移行した後、Shooter Genre consumerとしてplayer count、server／region authority、replication／rollback、anti-cheat、failure recovery、運用SLOの承認Decisionとscale Receiptが揃う | `claim.product.network.large-session-shooter; claim.product.network.high-player-count-support` |
| `future.capability.persistence-live-service-moderation-operations` | `mirakan.arch.ai-security-approval` | `planning_only` | `capability.product.general_production_2d; capability.product.general_production_3d` | `future.capability.headless-dedicated-server-session-transport-replication` | `product_scope; authority_model; data_model; threat_model; licensing; operations; qualification` | `headless_server; desktop; mobile; distributed_cluster` | `security; operations; network_fault; scale` | persistence、live update、moderation、privacy、retention、incident responseの各Owner Decisionと運用演習Receiptが揃う | `claim.product.operations.persistence; claim.product.operations.live-service; claim.product.operations.moderation` |
| `future.capability.mmo-distributed-world-authority` | `mirakan.arch.product-plan` | `planning_only` | `capability.product.general_production_3d` | `future.capability.headless-dedicated-server-session-transport-replication; future.capability.offline-large-world-continuous-streaming; future.capability.persistence-live-service-moderation-operations` | `product_scope; authority_model; data_model; threat_model; operations; qualification` | `headless_server; distributed_cluster; desktop` | `deterministic_simulation; network_fault; scale; security; operations` | 三つのFuture前提がactiveとなり、shard／region authorityとfailure recoveryの承認Decisionが成立する | `claim.product.network.mmo; claim.product.network.distributed-world` |
| `future.capability.vehicle-ragdoll-crowd-motion-warping` | `mirakan.arch.simulation-physics` | `planning_only` | `capability.simulation.physics-3d; capability.simulation.animation-3d` | `[]` | `product_scope; data_model; provider_selection; licensing; qualification` | `desktop; mobile` | `contract; deterministic_simulation; scale; target_device` | vehicle、ragdoll、crowd、motion warpingを独立Capabilityとして分割したOwner仕様とTarget prototype Receiptが揃う | `claim.product.feature.vehicle; claim.product.feature.ragdoll; claim.product.feature.crowd; claim.product.feature.motion-warping` |
| `future.capability.first-person-shooter-profile` | `mirakan.arch.pack-shooter` | `planning_only` | `capability.gameplay.combat; capability.gameplay.ranged_combat; capability.gameplay.encounter_spawn; capability.gameplay.scoring; capability.gameplay.pickup_grant; capability.gameplay.interaction; capability.gameplay.character_locomotion; capability.gameplay.path_following; capability.gameplay.scenario_stage; capability.domain.shooter-3d; capability.camera.3d; capability.simulation.animation-3d; capability.platform.input-core; capability.platform.audio-core; capability.platform.ui-core` | `[]` | `product_scope; data_model; target_program; qualification` | `desktop; mobile; console` | `contract; deterministic_simulation; target_device; content_quality` | first-person Camera／aim／weapon presentation、authoritative 3D Animation、Input／Audio／UI／comfort／Accessibility、2D／TPSとFeature意味を共有するProfile、独立fixture、Target別performanceを閉じる | `claim.product.feature.fps-support; claim.product.feature.first-person-shooter-profile` |
| `future.capability.production-terrain-foliage-gi` | `mirakan.arch.rendering-environment-surfaces` | `planning_only` | `capability.world.3d; capability.environment.core` | `[]` | `product_scope; data_model; provider_selection; licensing; qualification` | `desktop; mobile` | `content_quality; scale; target_device` | Terrain、Foliage、GIを別artifact pipelineとfallbackへ分離した仕様、budget Decision、Target比較Receiptが揃う | `claim.product.feature.production-terrain; claim.product.feature.production-foliage; claim.product.feature.production-gi` |
| `future.capability.aaa-photoreal-rendering` | `mirakan.arch.rendering-environment-surfaces` | `planning_only` | `capability.world.3d; capability.camera.3d; capability.environment.core; capability.render.material.realistic_advanced; capability.vfx.system; capability.product.general_production_3d` | `future.capability.production-terrain-foliage-gi` | `product_scope; data_model; provider_selection; licensing; target_program; qualification` | `desktop; console` | `contract; content_quality; scale; target_device; operations` | general 3D production後に、quality rubric、content pipeline、lighting／material／environment／VFX fidelity、hardware budget、意味同等fallbackの承認Decisionとblind comparison Receiptが揃う | `claim.product.quality.aaa-visual-parity; claim.product.quality.photoreal-rendering-guarantee` |
| `future.capability.console-target-program` | `mirakan.arch.product-plan` | `planning_only` | `capability.runtime.scheduling; capability.rendering.render-graph-core; capability.platform.input-core; capability.platform.audio-core; capability.platform.ui-core` | `[]` | `target_program; threat_model; licensing; operations; qualification` | `console` | `contract; target_device; security; operations` | 特定Console programへの参加承認後に、NDA下Owner、SDK、graphics／input／audio／UI adapter、certification、device lab、release operationsが確定する | `claim.product.platform.console-support; claim.product.platform.console-shipping` |
| `future.capability.web-target-program` | `mirakan.arch.product-plan` | `planning_only` | `capability.runtime.scheduling; capability.rendering.render-graph-core; capability.platform.input-core; capability.platform.audio-core; capability.platform.ui-core` | `[]` | `target_program; data_model; threat_model; provider_selection; licensing; qualification` | `web` | `contract; target_device; security; content_quality` | Web runtime、graphics、storage、threading、download、browser matrixのTarget Decisionとprototype Receiptが揃う | `claim.product.platform.web-support; claim.product.platform.web-shipping` |
| `future.capability.xr-target-program` | `mirakan.arch.rendering-camera` | `planning_only` | `capability.camera.3d; capability.platform.input-core; capability.platform.audio-core; capability.platform.ui-core; capability.rendering.render-graph-core` | `[]` | `target_program; data_model; threat_model; provider_selection; licensing; qualification` | `xr` | `contract; target_device; security; content_quality` | XR runtime、tracking、comfort、input、render budget、device matrixのTarget Decisionとprototype Receiptが揃う | `claim.product.platform.xr-support; claim.product.platform.xr-shipping` |
| `future.capability.commercial-asset-generation-license-quality-qualification` | `mirakan.arch.asset-lifecycle` | `planning_only` | `capability.authoring.ai-core; capability.authoring.asset-save-headless` | `[]` | `product_scope; provider_selection; licensing; threat_model; qualification` | `desktop; mobile` | `content_quality; security; target_device` | Provider、training／output license、provenance、quality rubric、human review、takedownの承認Decisionとblind evaluation Receiptが揃う | `claim.product.ai.commercial-asset-generation; claim.product.quality.asset-quality-guarantee` |
| `future.capability.first-party-local-inference` | `mirakan.arch.ai-security-approval` | `planning_only` | `capability.authoring.ai-core` | `[]` | `product_scope; provider_selection; threat_model; licensing; qualification` | `desktop` | `contract; target_device; security` | Engine Provider Adapter用model／runtime、weight license、hardware floor、sandbox、update／revocation、quality Gateの承認DecisionとTarget Receiptが揃う。external-agent Host conformanceはoptional integration fixtureでありpromotion prerequisiteではない | `claim.product.ai.bundled-local-model; claim.product.ai.offline-inference-guarantee` |
| `future.capability.managed-external-host-execution` | `mirakan.arch.ai-security-approval` | `planning_only` | `capability.authoring.ai-core; capability.product.external-agent; capability.product.project-source-activation` | `[]` | `product_scope; authority_model; threat_model; provider_selection; operations; qualification` | `desktop; headless_server` | `contract; target_device; security; operations` | exact Host／MCP Transport／Provider Runtime／managed deployment identity／Model／Tool／Authority Profile、Host／Transport・Provider／Tool・Schema／Evalの三Conformance、pre／post Host Attestation、managed Source edit／Build closure、Engine Build Receipt、filesystem／process／network／credential budget、revocation、adversarial fixtureの承認Decisionが揃う。Dossierの`execution_surface_candidates[]`と全Conformance／Build Receiptの`{execution_target_profile_ref, artifact_target_profile_ref: null-or-TargetProfileRefV1}`集合をset equalityにし、headless Hostと列挙済みWindows artifact等をflat Target集合または暗黙cross productから推測しない。mobile artifactは`candidate_target_kinds[]`と必要Future prerequisiteを追加する別Portfolio revisionまで含めない。通常のproposal-only external-agent conformanceを本能力の代用にしない | `claim.product.ai.managed-external-host-execution; claim.product.ai.managed-source-build` |
| `future.capability.runtime-structured-data-generation` | `mirakan.arch.ai-security-approval` | `planning_only` | `capability.product.runtime-generation-boundary` | `[]` | `product_scope; authority_model; data_model; threat_model; operations; qualification` | `desktop; mobile; headless_server` | `contract; deterministic_simulation; security; operations` | 許可Operation、server authority、quota、moderation、rollback、signed baseline、Target別failure policyの承認Decisionとadversarial Receiptが揃う | `claim.product.ai.runtime-generation; claim.product.ai.autonomous-runtime-mutation` |
| `future.capability.proof-carry-forward-definition-migration` | `mirakan.arch.product-plan` | `planning_only` | `capability.foundation.control-plane; capability.product.general_production_2d` | `[]` | `product_scope; data_model; threat_model; operations; qualification` | `headless_server; desktop; mobile` | `contract; deterministic_simulation; security; operations` | retained definition row、policy、binding、candidate、Toolchain、Target、fresh Evidenceと全dependency closureがbyte-exact同一の行だけを新epochへ再署名して継承し、差分closureを行単位resetするmigration schema／adversarial fixtureが承認される | `claim.product.migration.selective-state-carry-forward; claim.product.migration.routine-no-full-reset-upgrade` |
| `future.capability.linux-desktop-target-program` | `mirakan.arch.product-plan` | `planning_only` | `capability.runtime.scheduling; capability.rendering.render-graph-core; capability.platform.input-core; capability.platform.audio-core; capability.platform.ui-core` | `[]` | `target_program; provider_selection; licensing; operations; qualification` | `desktop` | `contract; target_device; operations` | exact distribution／kernel／libc／architecture、graphics／I/O／package adapter、device matrix、release operationsのDecisionとclean-device Receiptが揃う | `claim.product.platform.linux-runtime; claim.product.platform.linux-editor; claim.product.platform.linux-shipping` |
| `future.capability.macos-desktop-target-program` | `mirakan.arch.product-plan` | `planning_only` | `capability.runtime.scheduling; capability.rendering.render-graph-core; capability.platform.input-core; capability.platform.audio-core; capability.platform.ui-core` | `[]` | `target_program; licensing; operations; qualification` | `desktop` | `contract; target_device; security; operations` | exact macOS／architecture、Metal／I/O／package adapter、signing／notarization、device matrix、release operationsのDecisionとclean-device Receiptが揃う | `claim.product.platform.macos-runtime; claim.product.platform.macos-editor; claim.product.platform.macos-shipping` |
| `future.capability.mobile-project-native-shader-source-qualification` | `mirakan.arch.native-game-module` | `planning_only` | `capability.project.native_module; capability.project.shader; capability.platform.mobile-lifecycle` | `[]` | `product_scope; data_model; threat_model; licensing; qualification` | `mobile` | `contract; target_device; security; content_quality` | Android／AppleごとのABI、compiler、shader compiler、sandbox、signing、crash／rollback、Store policyを閉じたSource artifact Receiptが揃う | `claim.product.platform.mobile-project-cpp; claim.product.platform.mobile-project-shader` |
| `future.capability.local-multiplayer-split-screen` | `mirakan.arch.product-plan` | `planning_only` | `capability.gameplay.interaction; capability.platform.input-core; capability.platform.audio-core; capability.platform.ui-core; capability.camera.2d; capability.camera.3d` | `[]` | `product_scope; data_model; target_program; qualification` | `desktop; mobile; console` | `contract; deterministic_simulation; target_device; content_quality` | player assignment、join／leave、split viewport、focus、audio listener、save profile、Accessibilityを一つのTarget別integrated fixtureで閉じる | `claim.product.feature.local-multiplayer; claim.product.feature.split-screen` |
| `future.capability.unrestricted-project-scripting-runtime` | `mirakan.arch.gameplay-programming-model` | `planning_only` | `capability.authoring.ai-core; capability.product.project-source-activation; capability.runtime.debug-replay-support` | `[]` | `product_scope; authority_model; data_model; threat_model; provider_selection; licensing; operations; qualification` | `desktop; mobile` | `contract; deterministic_simulation; target_device; security; operations` | scripting VM／language、sandbox、determinism、hot reload、debug、package、mod trust、Target budgetのDecisionとadversarial／replay Receiptが揃う | `claim.product.feature.unrestricted-scripting; claim.product.feature.arbitrary-runtime-code; claim.product.feature.mod-execution` |

`excluded_current_product_claims[]`と`released_claims[]`は自由文字列でなく、同じFuture closureに含む次の`ProductClaimDefinitionRegistryV1`のexact `claim_id`だけを参照する。Registryのclaim ID集合は25件のFuture行が列挙する`excluded_current_product_claims[]`のunionとset equalityでなければならない。`claim_scope`は`feature | network | platform | operations | ai | quality | migration`のclosed値である。MVP-A、MVP-B、Technology Preview等のmilestoneはActive ProductのPhase／Release Gateだけが所有し、Future claimへ重複登録しない。canonical labelの変更はclaim identityを変えず、ID再利用、未登録label、別言語表示からのID推測を拒否する。

| claim_id | canonical label | claim_scope |
|---|---|---|
| `claim.product.ai.autonomous-runtime-mutation` | autonomous runtime mutation | `ai` |
| `claim.product.ai.bundled-local-model` | bundled local model | `ai` |
| `claim.product.ai.commercial-asset-generation` | commercial asset generation | `ai` |
| `claim.product.ai.managed-external-host-execution` | managed external Host execution | `ai` |
| `claim.product.ai.managed-source-build` | managed Source build | `ai` |
| `claim.product.ai.offline-inference-guarantee` | offline inference guarantee | `ai` |
| `claim.product.ai.runtime-generation` | runtime generation | `ai` |
| `claim.product.feature.arbitrary-runtime-code` | arbitrary runtime code | `feature` |
| `claim.product.feature.alternate-simulation-cadence` | alternate simulation cadence | `feature` |
| `claim.product.feature.configurable-physics-substep` | configurable physics substep | `feature` |
| `claim.product.feature.crowd` | crowd | `feature` |
| `claim.product.feature.first-person-shooter-profile` | first-person shooter profile | `feature` |
| `claim.product.feature.explicit-step-simulation` | explicit-step simulation | `feature` |
| `claim.product.feature.fps-support` | FPS support | `feature` |
| `claim.product.feature.general-3d-production` | general 3D production | `feature` |
| `claim.product.feature.local-multiplayer` | local multiplayer | `feature` |
| `claim.product.feature.mod-execution` | mod execution | `feature` |
| `claim.product.feature.motion-warping` | motion warping | `feature` |
| `claim.product.feature.offline-large-world-support` | offline large-world support | `feature` |
| `claim.product.feature.production-foliage` | production Foliage | `feature` |
| `claim.product.feature.production-gi` | production GI | `feature` |
| `claim.product.feature.production-terrain` | production Terrain | `feature` |
| `claim.product.feature.ragdoll` | ragdoll | `feature` |
| `claim.product.feature.split-screen` | split screen | `feature` |
| `claim.product.feature.unrestricted-scripting` | unrestricted scripting | `feature` |
| `claim.product.feature.vehicle` | vehicle | `feature` |
| `claim.product.migration.routine-no-full-reset-upgrade` | routine no-full-reset definition upgrade | `migration` |
| `claim.product.migration.selective-state-carry-forward` | selective state carry-forward | `migration` |
| `claim.product.network.competitive-networking` | competitive networking | `network` |
| `claim.product.network.cooperative-multiplayer` | cooperative multiplayer | `network` |
| `claim.product.network.dedicated-server-support` | dedicated-server support | `network` |
| `claim.product.network.distributed-world` | distributed world | `network` |
| `claim.product.network.high-player-count-support` | high-player-count support | `network` |
| `claim.product.network.large-session-shooter` | large-session network shooter | `network` |
| `claim.product.network.mmo` | MMO | `network` |
| `claim.product.network.multiplayer` | multiplayer | `network` |
| `claim.product.network.rollback-networking` | rollback networking | `network` |
| `claim.product.operations.live-service` | live service | `operations` |
| `claim.product.operations.moderation` | moderation | `operations` |
| `claim.product.operations.persistence` | persistence | `operations` |
| `claim.product.platform.console-shipping` | console shipping | `platform` |
| `claim.product.platform.console-support` | console support | `platform` |
| `claim.product.platform.linux-editor` | Linux editor | `platform` |
| `claim.product.platform.linux-runtime` | Linux runtime | `platform` |
| `claim.product.platform.linux-shipping` | Linux shipping | `platform` |
| `claim.product.platform.macos-editor` | macOS editor | `platform` |
| `claim.product.platform.macos-runtime` | macOS runtime | `platform` |
| `claim.product.platform.macos-shipping` | macOS shipping | `platform` |
| `claim.product.platform.mobile-project-cpp` | mobile Project C++ | `platform` |
| `claim.product.platform.mobile-project-shader` | mobile Project Shader | `platform` |
| `claim.product.platform.web-shipping` | web shipping | `platform` |
| `claim.product.platform.web-support` | web support | `platform` |
| `claim.product.platform.xr-shipping` | XR shipping | `platform` |
| `claim.product.platform.xr-support` | XR support | `platform` |
| `claim.product.quality.aaa-visual-parity` | AAA visual parity | `quality` |
| `claim.product.quality.asset-quality-guarantee` | asset quality guarantee | `quality` |
| `claim.product.quality.photoreal-rendering-guarantee` | photoreal rendering guarantee | `quality` |

Future IDはactiveな`CapabilityRegistryV1`、`ProductPhaseRegistryV1`、`PhaseFixtureBindingRegistryV1`、Shipping labelから参照してはならない。上表からactive Capabilityへの`prerequisite_capability_refs[]`はincubation側だけが持つ一方向参照であり、CapabilityのActivation、Target対応、Phase schedulingを発生させない。`future.capability.large-session-network-shooter`のactivation triggerが参照する二つのFuture IDもplanning dependencyであってactive DAG edgeではなく、各前提を独立したProduct Decisionでactive Capabilityへ昇格させるまでlarge-session Entryをschedulingしない。各Entryを実装DAGへ移すには、新しいProduct DecisionでOwner、Target、fallback、Requirement、Fixture、Work Packageを同時に登録し、`planning_only`のまま暗黙activateしない。

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

### 9.4 Platform／Packs

- [Windows](../07-platform/windows.md)
- [Mobile common](../07-platform/mobile-common.md)
- [Android](../07-platform/android.md)
- [Apple](../07-platform/apple.md)
- [Input](../07-platform/input.md)
- [Audio](../07-platform/audio.md)
- [UI／text／localization／accessibility](../07-platform/ui-text-localization-accessibility.md)
- [Pack contract](../08-packs/pack-contract.md)
- [Gameplay Feature Packs](../08-packs/gameplay-features.md)
- [Shooter Genre Pack](../08-packs/shooter.md)
- [Scenario／Stage Feature Pack](../08-packs/scenario-stage.md)

RPG-first targetは、Generic Engine Core、command battle／actor progression／inventory・equipment／dialogue・quest／currency・shopを所有するReusable RPG Feature Packs、composition／RPG profile／game flow／command role mapping／Reference fixture bindingを所有するRPG Genre Pack、original content／balance／World composition／localized presentation／acceptance fixtureを所有するRPG Reference Gameの四層に分ける。既存Combat、Interaction、Scenario／Stage、Pickup／Grant、UI、Localization、Save／Replay、Runtime契約はcurrent semanticsが適合する場合だけ再利用し、Shooter契約を暗黙拡張せずGenre意味をCoreへ移さない。Reusable Feature owner、RPG Genre Pack owner、Reference Game ownerは本変更では未materializeであり、focused owner designを承認するまでStable ID、Schema Field、Document ID、Registry、Capability、Operation、Work Packageへ追加しない。

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

Primary Product Evidenceは、MVP-AのRPG First PlayableとGenre非依存Core holdout、MVP-Bの独立3D technical Reference、clean package／install／offline run、AIと手動編集の往復、Save／Load、Undo／Replay、Target別Qualification、failure／rollback、Release closureである。MVP-BまたはShooter EvidenceをMVP-A、RPG、Genre非依存Coreの代用にしない。EvidenceのRecord形式、保持、Trace、SBOM、ProvenanceはVerification Ownerだけが決定する。

## 10. 外部比較の使用範囲

Unreal Engine、Unity、Godotの公式資料はCoverageと責務分離の比較Evidenceにだけ使う。MiraikanaiのAPI、Class hierarchy、Scene、Project形式、Editor UX、既定値の根拠にはしない。比較対象は市場順位の網羅表ではなく、(1) Editor統合AIと段階的permission／Undoを公式公開する商用Engine、(2) reflection／registryから型付きEditor toolを公開する商用Engine、(3) textで差分可能なScene／Resourceと拡張Editor APIを公式公開するopen-source Engine、の三つの制作境界を最小集合で比較できるため選んだ。Engine数、人気、機能数、benchmark順位を設計Evidenceにせず、比較軸を満たさない追加Engineを列挙目的で増やさない。

この比較は非網羅であり、各製品の最新major Editor、AI surface、serialization／reflection APIの公式資料をProduct判断時に再読込する。比較対象の名称または機能が変わってもMiraikanaiの契約を自動変更せず、公式資料で同じ比較軸を再検証し、採用原則が変わる場合だけArchitecture ChangeSetとして扱う。

| Engine | 公式に確認した制作境界 | Miraikanaiへ採る原則 | 採らないもの |
|---|---|---|---|
| Unity 6 | Unity AI Open BetaはEditor内のProject文脈を使うAsk／Plan／Agent、permission level、変更確認／Undoを提示する。Editor data変更は`SerializedObject`がdirty、Undo、Prefab overrideを扱う公式経路である | liveだがboundedなsemantic context、段階的権限、preview／Undo、Editor-owned mutation経路 | AIによるraw Asset／Scene file直接編集、UI上のpermission表示だけをAuthorityとすること |
| Unreal Engine 5.8 | Experimental Unreal MCPはEditor内のToolset Registryから型付きtoolを公開し、Actor／Component／Assetはreflection／Asset Registry／Editor APIで扱う。Python公式資料もOS file APIによるAsset操作を禁止する | reflection-backed projection、登録tool、Editor API、commit後read-back | 任意property setter、汎用Python／console／file edit、localhostであることを認証や承認の代用にすること |
| Godot 4.x | Node／Scene／Resourceとtext形式`.tscn`、`EditorPlugin`／`ClassDB`により構造が観測可能で差分化しやすい。確認した公式資料にはUnity AI／Unreal MCP相当の内蔵AI authoring authorityはない | 透明でdiff可能なcanonical source、型／UID／NodePathを保つEditor-owned validation | textであることを安全性の証明にすること、subresource／順序／UID制約を無視した直接置換 |

scene／screen切替についても公式資料から次の責務分離だけを採用判断へ使う。

| Engine | 公式に確認した切替モデル | Miraikanaiでの対応／差分 |
|---|---|---|
| Unity 6 | `SceneManager`がSceneをSingle／Additiveでloadし、Sceneを越えて保持するObjectは`DontDestroyOnLoad`で明示する | top-level切替と永続Sessionを分離する原則は採る。一方、Scene AssetをTitle／Stage／UI／Worldの共通authorityにはせず、Runtime Entry、Stage、World、UI Screenをtyped Owner契約へ分ける |
| Unreal Engine 5.x | Gameplay FrameworkはGameInstanceをMap load間で保持し、World／LevelをTravel／Streamingする。Common UIはActivatable Widget Stackを持つ | persistent Play Session＋branch travel＋UI Stackの三分離に最も近い。Miraikanaiはさらにexact ref／hash、T00 atomic publish、source保持failureを共通Runtime契約にする |
| Godot 4.5 | `SceneTree`のcurrent sceneを変更し、global stateやscene switching helperはAutoloadで保持できる | 小規模Projectの単純性は参考にするが、global singletonへdestination、Save、Stage、UI authorityを集約せず、registered PortとOwner validatorを必須にする |

三Engineとも画面切替、永続state、UI navigationを完全に同一Objectへ畳み込むことを要求していない。したがってMiraikanaiは単一`SceneManager`を追加せず、§5.0.1の`Runtime Entry replacement + UI Screen Stack + Stage transition + World spatial transition`を正規分解とする。

- [Unity AI: Ask／Plan／Agent](https://unity.com/blog/unity-ai-assistant-ask-plan-agent-mode-explained)
- [Unity AI: Get started](https://unity.com/blog/unity-ai-how-to-get-started)
- [Unity 6 `SerializedObject`](https://docs.unity3d.com/6000.0/Documentation/ScriptReference/SerializedObject.html)
- [Unity 6 additive Scene load](https://docs.unity3d.com/6000.1/Documentation/ScriptReference/SceneManagement.LoadSceneMode.Additive.html)
- [Unity 6 multiple Scene editing](https://docs.unity3d.com/ja/6000.0/Manual/setupmultiplescenes.html)
- [Unity 6 `DontDestroyOnLoad`](https://docs.unity3d.com/6000.0/Documentation/ScriptReference/Object.DontDestroyOnLoad.html)
- [Unreal Engine 5.8 Unreal MCP](https://dev.epicgames.com/documentation/unreal-engine/unreal-mcp-in-unreal-editor)
- [Unreal Engine Reflection System](https://dev.epicgames.com/documentation/unreal-engine/reflection-system-in-unreal-engine)
- [Unreal Engine Asset Registry](https://dev.epicgames.com/documentation/en-us/unreal-engine/asset-registry-in-unreal-engine)
- [Unreal Editor Python](https://dev.epicgames.com/documentation/en-us/unreal-engine/scripting-the-unreal-editor-using-python)
- [Unreal Engine Gameplay Framework](https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-framework-in-unreal-engine)
- [Unreal Engine Level Streaming](https://dev.epicgames.com/documentation/unreal-engine/level-streaming-in-unreal-engine)
- [Unreal Engine Common Activatable Widget Stack](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Plugins/CommonUI/UCommonActivatableWidgetStack)
- [Godot Nodes and Scenes](https://docs.godotengine.org/en/stable/getting_started/step_by_step/nodes_and_scenes.html)
- [Godot `.tscn` format](https://docs.godotengine.org/en/latest/engine_details/file_formats/tscn.html)
- [Godot Editor plugins](https://docs.godotengine.org/en/stable/tutorials/plugins/editor/making_plugins.html)
- [Godot 4.5 `SceneTree`](https://docs.godotengine.org/en/4.5/classes/class_scenetree.html)
- [Godot 4.5 Autoload](https://docs.godotengine.org/en/4.5/tutorials/scripting/singletons_autoload.html)
- [Unreal Engine 5.8 Modules](https://dev.epicgames.com/documentation/unreal-engine/unreal-engine-modules?lang=en-US)
- [Unity 6 Native plug-ins](https://docs.unity3d.com/6000.0/Documentation/Manual/plug-ins-native.html)
- [Godot 4.5 GDExtension](https://docs.godotengine.org/en/4.5/tutorials/scripting/gdextension/what_is_gdextension.html)
- [Unity Hub／Editor language selection](https://docs.unity.com/en-us/hub/add-editor-language)
- [Unreal Engine Localization Overview](https://dev.epicgames.com/documentation/en-us/unreal-engine/localization-overview-for-unreal-engine)
- [Godot EditorSettings language／property-name localization](https://docs.godotengine.org/en/latest/classes/class_editorsettings.html)

外部資料の時点依存情報はArchitecture定数にせず、更新判断とEvidence freshnessを[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)へ委ねる。

## 11. Product execution registries

本節はControl Plane移行時にMCDへ移すProduct-owned機械正本である。それまではMarkdown表を入力とし、表外のID、暗黙行、前方一致、別名を拒否する。

本節のRegistryはcurrent source Product Definitionを表す。§5.1.1のRPG-first target directionを理由に既存行を逐次改変せず、focused owner designとatomic Product Definition migrationが完了するまでは、Shooter fixture、Work Package、Capability参照をsource baselineとして保持する。RPG向けの未定義ID、Availability、Receiptを本節から推測または生成しない。

### 11.1 Registry共通規則

全Registryは`registry_id`、`format_major=1`、`revision`、`entries[]`を持つ。現行初版の`revision`は1であり、以後は同じ`registry_id`／`format_major`内で1ずつ増えるpositive safe integerとする。Active Definition変更時に変更のないRegistryは同じrevision／bytesを再利用し、変更するRegistryだけを`N+1`へ進める。revisionの巻戻し、gap、同revision別bytes、内容変更なしのrevision増加を拒否する。`ActiveProductDefinitionBundleV1.revision`と`FuturePortfolioDefinitionBundleV1.revision`も同じ規則を持ち、いずれかのmember ref／hashまたはbundle Fieldが変わる場合だけexact `N+1`、全bytes不変なら同revisionを再利用する。Active headはsigned operational current pointerとinline `CurrentControlPlaneBaselineBindingV1`で一件に固定し、genesisではBootstrap branch、後続Rebaselineではtagged rebaseline branchを検証する。Future headは`FuturePortfolioApprovalV1`のCASで一件に固定し、同revision別closureを拒否する。`entries[]`はlogical IDのUTF-8 byte順、重複禁止である。Markdown表は人間の可読性のためPhase順などで提示してよいが、generatorは表をID付き集合として読み、serialized `entries[]`をlogical IDのUTF-8 byte順へ正規化する。validatorは正規化前後のID集合が完全一致することを検査し、表の表示順を保存状態やidentityとして扱わない。参照はexact logical IDだけを受理し、display name、path、配列index、maturity、Phase番号、schema versionをidentityとして使わない。

本節の`CapabilityRegistryV1`はProduct Phase、Target、fallback、Product labelのactivation判定に使うProduct-owned選抜registryであり、全Subsystem内部Capability catalogの複製ではない。Collision、Physics、Animation、Input、Audio等のSubsystem-local C0／C1 contractとdiscovery entryは各Ownerが所有する。ただしProduct Phase、Work Package、fixture、C2 matrix、またはTarget GateからCapability IDを参照する場合は、同じChangeSetで本節へexact行、owner Work Package、Target scope、fallbackを追加しなければならない。Subsystem catalog entryだけをProduct activationの代用にせず、本節未登録IDをProduct-qualifiedとして扱わない。

Foundation MCDとのbridgeを次に固定する。`capability.runtime.scheduling@1`はExecutable Contractsに完全なactive MCD recordを持ち、その`supported_targets`は`{target.android.mobile, target.apple.mobile, target.headless.host, target.windows.desktop, target.windows.editor}`である。本節の同ID行はこの五Targetすべてを`required`で持ち、set equalityを必須にする。MCD `status=active`は契約がContract setで参照可能であることだけを意味し、`ProductOperationalStateSnapshotV1`のActivation rowを変更しない。Candidate／Qualification／Production Receiptがないrowは既定どおり`not_activated`であり、MCD recordをReceiptとして扱わない。

`capability.authoring.command_gateway@1`はTrusted Serviceのauthority境界を記述するcurrent C0 Capabilityで、Product選択、Phase label、Target shippingを表すCapabilityではない。このためProduct `CapabilityRegistryV1`に行を持たないことが正規であり、欠落補完、暗黙Product Activation、利用可能表示を行わない。Serviceの起動可否は[AI Security／Approval §7.3](../01-governance/ai-security-approval.md#73-trusted-service-isolation)のservice-authority branch、すなわちcurrent Signer Policy operation集合、Service allowlist、Trust Head／closure、executable／Profile／Contract set／Targetのfresh Technical Qualificationをset equalityにして判定し、Product row不存在を起動不能またはProduct support claimへ読み替えない。`capability.authoring.offline_migration@1`、`service.offline_project_migrator@1`、`profile.isolation.offline_project_migrator@1`のcurrent集合はexact `[]`で、Executable Contracts §8.1.2のlegacy evidence gateまたは将来の汎用schema migration familyを満たすatomic activationだけが初めて三recordを追加できる。将来Product Phase／WP／fixture／Target Gateがいずれかを直接参照する場合だけ、本節の通常ChangeSet規則でProduct-owned行を追加する。

MCD `capability.runtime.scheduling@1.required_capabilities=[]`はCapability契約そのもののsemantic dependencyである。一方、`wp.runtime.scheduling-core`の`requires_work_package_refs[]`と他WPから同WPへのedgeは実装／検証のbuild orderであり、Product Planが所有する。両者を同じ配列として比較せず、MCDが空であることを理由にWP依存を削除したり、WP依存からMCD Capability dependencyを暗黙生成したりしない。

```text
CapabilityRegistryV1
  entries[]: capability_id, target_product_tier, owner_work_package_ref,
             target_bindings[], fallback_ref
  target_bindings[]: target_id, scope(required | optional | excluded)

CapabilityTargetActivationStateV1 // operational state only
  capability_id, target_id, state, candidate_ref, receipt_refs[]

PlannedOperationFamilyRefV1
  planning_record_id
  planning_record_version
  planning_record_hash

ExecutionSurfaceBindingV1
  planned_family_ref: PlannedOperationFamilyRefV1
  activation_work_item_id
  required_operation_ids[1..64]
  required_operation_set_sha256: lowercase hex 64
  execution_host_target_refs[1..5]
  artifact_target_id: null | target_id
  required_execution_state = operational
  provider_projection_requirement =
    none
    | all_activation_declared_external_safe
    | proposal_subset_excluding_trusted_internal

AiCapabilityActivationMatrixV1
  matrix_id = matrix.product.ai-capability-activation.v1
  matrix_version = 1
  execution_surface_required_capability_refs[10]
  deny_only_capability_refs[1]
  rows[10]:
    capability_id
    owner_work_package_ref
    requirement_refs[]
    phase_refs[]
    fixture_refs[]
    risk_refs[]
    state_transition_policy_refs[]:
      ProductOperationalStatePolicyRegistryV1.state_policy_id
    candidate_binding_policy_literal:
      ProductCandidateBindingPolicyLiteralV1
    planning_family_refs[]
    execution_host_rule =
      windows_editor_for_artifact_target
      | headless_proposal_only
    provider_projection_requirement
  matrix_sha256

CapabilityTargetActivationBindingRegistryV1 // immutable promotion definition
  ai_capability_activation_matrix: AiCapabilityActivationMatrixV1
  entries[]:
    capability_id, target_id, owner_work_package_ref
    candidate_lock_policy = owner_wp_task_technical_qualification
    execution_surface_bindings[]: ExecutionSurfaceBindingV1
    qualification_policy = all_evidence_bindings | unavailable_no_gate
    qualification_evidence_bindings[]:
      {provider_work_package_ref, gate_ref, fixture_ref,
       requirement_refs[], freshness_policy_ref}
    qualification_gate_refs[]
    qualification_requirement_refs[]
    production_policy =
      {kind: disabled_current_definition, release_gate_refs: []}
      | {kind: all_release_gates, release_gate_refs[]}
    candidate_binding_policy_ref
    freshness_policy_refs[]

ProductReleaseGateRegistryV1
  entries[]:
    release_gate_id, target_id, fixture_id,
    evaluated_requirement_refs[], required_phase_gate_refs[],
    required_work_package_states[]: {work_package_id, required_state},
    candidate_binding_policy_ref = policy.product.same-candidate.v1,
    freshness_policy_ref,
    critical_risk_predicate = all_impact_critical_effective_mitigated_or_closed

ProductPhaseRegistryV1
  entries[]: phase_id, order_index, outcome_requirement_refs[], work_package_refs[], exit_gate_refs[]

ProductCandidateBindingPolicyLiteralV1 =
  policy.product.same-candidate.v1

WorkPackageRegistryV1
  entries[]: work_package_id, phase_id, owner_document_id, target_refs[], fallback_ref,
             provided_fixture_refs[]: ProvidedFixtureRefV1,
             required_capability_refs[], requires_work_package_refs[],
             scheduling_state, defer_reason, reconsideration_gate_refs[], blocked_reason_ref

ProvidedFixtureRefV1 =
  {kind: product_fixture,
   fixture_id: FixtureRegistryV1.fixture_id}
  | {kind: component_qualification_fixture,
     owner_document_id,
     owner_pack_ref: PackContractRefV1,
     owner_pack_manifest_ref: ArtifactRefV1(
       artifact_kind=pack_manifest, schema_version=1),
     owner_pack_manifest_sha256,
     fixture_id, fixture_version, fixture_content_sha256}

PhaseFixtureBindingRegistryV1
  entries[]: gate_id, phase_id, fixture_id, evaluated_requirement_refs[], target_refs[],
             candidate_binding_policy_ref = policy.product.same-candidate.v1,
             freshness_policy_ref

TargetProfileRegistryV1
  entries[]:
    target_id, owner_document_id, profile_version,
    target_kind =
      headless_server | desktop | mobile | console |
      web | xr | distributed_cluster,
    surface_role = execution_host | artifact_runtime | both,
    qualification_requirement_ref

RequirementRegistryV1
  entries[]: requirement_id, owner_document_id, verification_kind, failure_diagnostic_id

FixtureRegistryV1
  entries[]: fixture_id, owner_document_id, requirement_refs[], target_refs[], minimum_duration_seconds
```

次の二recordはRegistry一覧に必要なentry payloadの投影であり、型の再定義ではない。`FallbackRegistryV1`のclosed canonical schemaは§11.3、`FutureCapabilityIncubationRegistryV1`の唯一のcanonical schemaは§8が所有し、本一覧はそれらとField名・意味を一致させる。

```text
FallbackRegistryV1
  entries[]: ProductFallbackRecordV1 {
    fallback_id, owner_document_id, preserves_semantics,
    failure_diagnostic_ref: ProductDiagnosticRefV1}
  diagnostic_records[]: ProductDiagnosticRecordV1

FutureCapabilityIncubationRegistryV1
  entries[]: future_capability_id, owner_document_id, planning_state,
             prerequisite_capability_refs[], prerequisite_future_capability_refs[], required_decision_kinds[], candidate_target_kinds[],
             qualification_fixture_kinds[], activation_trigger, excluded_current_product_claims[]

ProductClaimDefinitionRegistryV1
  entries[]: claim_id, canonical_label, claim_scope

ProductRiskDefinitionRegistryV1
  entries[]: risk_id, owner_document_id, affected_work_package_refs[], trigger,
             likelihood, impact, mitigation, contingency, monitor_gate_refs[], revisit_gate_or_date,
             genesis_state = open
  revisit_gate_or_date:
    { kind: phase_gate, ref: PhaseFixtureBindingRegistryV1.gate_id }
    | { kind: decision_gate, ref: ProductDecisionGateRegistryV1.gate_id }
    | { kind: date, value: YYYY-MM-DD }

ProductDecisionGateRegistryV1
  evidence_class_definitions[]:
    {class_id, owner_document_id, wrapper_schema_id, signed_record_purpose,
     freshness_policy_ref, subject_policy_ref, required_target_refs[]}
  definition_change_class_definitions[]:
    {class_id, owner_document_id, wrapper_schema_id, signed_record_purpose,
     subject_policy_ref}
  entries[]:
    gate_id, owner_document_id
    genesis_state = blocked
    predicate:
      evaluator_policy = all_of
      required_phase_gate_refs[]
      required_work_package_states[]: {work_package_id, required_state}
      required_evidence_classes[]
      required_definition_change_classes[]
    on_satisfied_action:
      {kind: permit_work_package_transition,
       work_package_id, from_state, to_state, transition_policy_ref}

WorkPackageLifecyclePolicyRegistryV1
  entries[]: transition_policy_id, allowed_edges[], subject_binding_policy,
             prerequisite_policy, receipt_policy, authorization_requirements[],
             bootstrap_only_work_package_ref
  authorization_requirements[]: {edge, authorization_schema, authorization_kind}

ProductOperationalStatePolicyRegistryV1
  entries[]: state_policy_id, change_kind, allowed_edges[], authority_role_ref,
             signed_record_purpose, evidence_policy_refs[], decision_requirements[]
  decision_requirements[]: {edge, requirement}
```

`ProductFallbackRecordV1`、`ProductDiagnosticRefV1`、`ProductDiagnosticRecordV1`のexact Field、hash、set-equality規則は§11.3のclosed shapeをforward-referenceする。この概要からID-only `failure_diagnostic_id`、anonymous Diagnostic、別Registryを生成しない。`RequirementRegistryV1.failure_diagnostic_id`はRequirement評価器の別closed tokenであり、Fallbackのtyped Product Diagnostic refへ機械置換しない。

Work Package表の`provided_fixture_refs[]`列は`ProvidedFixtureRefV1`の可読source notationである。`fixture.product.*`は`kind=product_fixture`として同じActive Definitionの`FixtureRegistryV1`へexact解決する。現行表で許可する`kind=component_qualification_fixture`は、`fixture.feature.combat.contract`、`fixture.feature.ranged_combat.contract`、`fixture.feature.encounter_spawn.contract`、`fixture.feature.scoring.contract`、`fixture.feature.pickup_grant.provider_neutral`、`fixture.feature.interaction.contract`、`fixture.feature.character_locomotion.motion_executor`、`fixture.feature.path_following.executor_stub`、`fixture.feature.scenario_stage.none`のexact九件だけである。先頭八件は`owner_document_id=mirakan.arch.pack-gameplay-features`かつ各対応`feature.combat@1 | feature.ranged_combat@1 | feature.encounter_spawn@1 | feature.scoring@1 | feature.pickup_grant@1 | feature.interaction@1 | feature.character_locomotion@1 | feature.path_following@1`の`PackContractRefV1`、Scenario一件は`owner_document_id=mirakan.arch.pack-scenario-stage`かつ`feature.scenario_stage@1`の`PackContractRefV1`へ解決する。generatorは各current approved owner Pack Manifestから`owner_pack_ref`、完成manifest ref／hash、fixture version／content hashを解決してtagged objectを生成し、Gameplay Featuresのaggregate表示をScenario ownerへ偽装せず、裸IDを保存しない。missing、owner／Pack不一致、version／hash不一致、上記以外のprefixまたはfeature fixtureをDefinition construction errorにする。component fixtureは当該WP実装のQualification入力であってProduct Phase／Release Gateのevidenceではない。Product Gateへ使うには`FixtureRegistryV1`への明示rowと`kind=product_fixture` bindingをActive Definition revisionで追加する。

`policy.product.same-candidate.v1`はRegistry lookup IDではなく、`ProductCandidateBindingPolicyLiteralV1`が許す唯一の固定literalである。そのclosed predicateはProject revision、Candidate root hash、Contract set hash、Toolchain lock hash、Target Profile ref／hash集合のbyte equalityである。`PhaseFixtureBindingRegistryV1`、`ProductReleaseGateRegistryV1`、`CapabilityTargetActivationBindingRegistryV1`の`candidate_binding_policy_ref`はこのliteralだけを受理し、別ID、未登録の説明語、部分一致を拒否する。これに対し`policy.product.activation.promotion.v1`は`ProductOperationalStatePolicyRegistryV1`の登録rowであり、両者を同じRegistry refとして解釈しない。

上記のうち`CapabilityTargetActivationStateV1`だけは定義Registryではない。Product definitionと実行中の観測状態を次の二層へ分離し、Bootstrap Approvalが可変stateをhash対象に含めて自己失効することを禁止する。

```text
ActiveProductDefinitionBundleV1 // immutable within one approved active definition revision
  registry_id, format_major, revision
  capability_registry_ref
  capability_target_activation_binding_registry_ref
  product_release_gate_registry_ref
  product_phase_registry_ref
  work_package_registry_ref
  phase_fixture_binding_registry_ref
  target_profile_registry_ref
  requirement_registry_ref
  fixture_registry_ref
  fallback_registry_ref
  product_risk_definition_registry_ref
  product_decision_gate_registry_ref
  work_package_lifecycle_policy_registry_ref
  product_operational_state_policy_registry_ref

ActiveProductDefinitionClosureV1
  bundle: ActiveProductDefinitionBundleV1
  registry_manifest[]:
    {registry_id, format_major, revision, registry_ref, registry_sha256}

FuturePortfolioDefinitionBundleV1 // independently approved planning-only hash domain
  portfolio_id, format_major, revision
  future_capability_registry_ref
  product_claim_definition_registry_ref
  future_portfolio_policy_registry_ref
  membership_revision_policy_ref
  claim_transition_policy_ref

FuturePortfolioPolicyRegistryV1
  entries[]:
    {policy_id, policy_kind, authority_role_ref,
     signed_record_purpose, required_decision_kind: null | ProductOperationalDecisionV1.decision_kind}

FuturePortfolioDefinitionClosureV1
  bundle: FuturePortfolioDefinitionBundleV1
  registry_manifest[]: {registry_id, format_major, revision, registry_ref, registry_sha256}

FuturePortfolioApprovalPayloadV1
  approval_id
  approval_sequence: positive safe integer
  future_portfolio_definition_sha256
  previous_approval_ref: null | content-addressed ref
  previous_approval_sha256: null | lowercase hex 64
  approver_subject_ref
  approval_authority_ref
  issued_at
  valid_until
  revocation_snapshot_ref

FuturePortfolioApprovalV1
  payload: FuturePortfolioApprovalPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=future_portfolio_definition_approval)

FutureProductClaimReleasePayloadV1
  claim_release_id
  claim_release_sequence: positive safe integer
  previous_release_ref: null | content-addressed ref
  previous_release_sha256: null | lowercase hex 64
  future_capability_id
  destination_active_product_definition_sha256
  promotion_manifest_ref
  promotion_manifest_sha256
  operational_state_snapshot_ref
  operational_state_snapshot_sha256
  activation_keys[]: {capability_id, target_id}
  production_receipt_refs[]
  released_claims[]
  authorization_decision_ref
  issued_at
  valid_until
  revocation_snapshot_ref

FutureProductClaimReleaseV1
  payload: FutureProductClaimReleasePayloadV1
  signed_record: MirakanSignedRecordV1(purpose=future_product_claim_release)

FutureToActivePromotionManifestPayloadV1
  promotion_manifest_id
  source_future_portfolio_definition_sha256
  source_future_portfolio_approval_ref
  source_future_portfolio_approval_sha256
  future_capability_id
  destination_active_product_definition_sha256
  promoted_active_ids[]: {registry_id, row_kind, logical_id, migration_kind}
  active_definition_migration_ref
  active_definition_migration_sha256
  generated_at
  revocation_snapshot_ref

FutureToActivePromotionManifestV1
  payload: FutureToActivePromotionManifestPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=future_to_active_promotion_manifest)

ProductOperationalStateSnapshotPayloadV1
  state_snapshot_id
  active_product_definition_sha256
  control_plane_baseline_binding: CurrentControlPlaneBaselineBindingV1
  sequence
  previous_state_snapshot_ref: null | content-addressed ref
  applied_change:
    {kind: genesis,
     bootstrap_approval_ref, bootstrap_approval_sha256,
     baseline_envelope_ref, baseline_envelope_sha256,
     construction_decision_ref, trust_provisioning_receipt_ref}
    | {kind: operational_transition, transition_wrapper_ref, transition_wrapper_sha256}
    | {kind: wp_lifecycle, lifecycle_wrapper_ref, lifecycle_wrapper_sha256}
    | {kind: active_definition_migration, migration_wrapper_ref, migration_wrapper_sha256}
    | {kind: control_plane_rebaseline, rebinding_wrapper_ref, rebinding_wrapper_sha256}
  capability_target_activation_rows[]: CapabilityTargetActivationStateV1
  decision_gate_evaluations[]: {gate_id, state, evidence_refs[]}
  risk_evaluations[]: {risk_id, state, evidence_refs[]}
  work_package_lifecycle_heads[]:
    {work_package_id, lifecycle_record_ref: null | content-addressed ref, lifecycle_sequence}
  created_at
  revocation_snapshot_ref

ProductOperationalStateSnapshotV1
  payload: ProductOperationalStateSnapshotPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=product_operational_state_snapshot)
```

`active_product_definition_sha256`は`ActiveProductDefinitionBundleV1`と全参照active definition Registryのcanonical bytes closureから決定論的に算出する。`ControlPlaneBootstrapApprovalV1`、baseline、Critical Path Taskはこのhashを`product_registry_sha256`という旧名でなくexact `active_product_definition_sha256`として束縛する。Activation、Decision Gate評価、Risk評価、Work Package lifecycle headの通常更新はdefinition bytesを変更しない。

`control_plane_baseline_binding`はControl Plane設計が所有するclosed `CurrentControlPlaneBaselineBindingV1`をinlineで保持する。初回genesisは`kind=bootstrap`、Active Definition migrationは同payloadのRebaseline Approval／Envelope／Transactionから`kind=rebaseline`を生成し、通常state transition／WP lifecycleはparent値をbyte-exactに保持する。bindingの`active_product_definition_sha256`はsnapshot同Fieldと一致しなければならず、初回Approvalを後続definitionへ流用、別definitionのEnvelope、missing Transaction、任意の`latest` lookupを拒否する。通常Taskは「初回Bootstrapが永続的にcurrent」と仮定せず、このcurrent bindingだけをentry conditionに使う。

`ActiveProductDefinitionClosureV1`は上記2 Fieldだけのclosed objectで、`registry_manifest[]`をregistry IDのunsigned UTF-8 byte順にする。`active_product_definition_sha256 = SHA-256(JCS(ActiveProductDefinitionClosureV1))`とし、文字列連結、filesystem順、別framingを使わない。manifestはBundleが要求する全registryとset equalityで、missing／extra registry、ref先hash差、unknown Fieldを拒否する。全`*_sha256` Fieldはprefixなしlowercase hex 64文字、content-addressed `*_ref`は`sha256:<lowercase-hex-64>`で完成artifactまたは完成signed wrapperを指す。refと隣接hash Fieldは同じbytesへ解決する。Operational snapshotのActivation rowsは`{capability_id,target_id}`、Decisionは`gate_id`、Riskは`risk_id`、WP headは`work_package_id`のunsigned UTF-8 byte順で、全配列を重複なしとする。Receipt／Evidence／Target／policy ref配列もunsigned UTF-8 byte順で、duplicate、unknown、未正規順を拒否する。Markdown表内のsemicolon列は集合の可読表示であってserialization順ではない。generatorはtop-level `entries[]`だけでなく全nested set配列をschema指定keyのunsigned byte順へ正規化し、ID集合が入力表と一致することを確認してからJSONを生成する。validatorは生成済みJSONの未正規順を拒否し、表の表示順をhashへ使わない。

JSONへ保存する全Product `sequence`はinteger `0..9007199254740991`（ECMAScript safe integer）に固定する。genesisだけ0、通常nextはexact `N+1`、上限でのincrementはoverflowとして拒否し、float、負値、指数表記文字列、丸めたuint64を受理しない。Revocation sequenceも同じcanonical表現を使う。

`FuturePortfolioDefinitionClosureV1`はActive closureと同じJCS、manifest set equality、ref／hash、sort規則を別hash domainで適用し、manifestは`FutureCapabilityIncubationRegistryV1`、`ProductClaimDefinitionRegistryV1`、`FuturePortfolioPolicyRegistryV1`のexact 3件とする。`future_portfolio_definition_sha256=SHA-256(JCS(closure))`とし、claim definitionを別fileやMarketing表へ逃がさない。`FuturePortfolioApprovalV1`は`role.future-portfolio-approver.r4`へのactive human assignmentとsingleton purpose keyで署名し、payload／signed recordのsubject、Signer、Role、issued_at、revocation snapshotをexact一致させる。`approval_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:future-portfolio-approval:sha256:<lowercase-hex>`として導出する。初版だけprevious両Field=nullかつ`approval_sequence=1`、後続はcurrent completed approval wrapper ref／hashを持ち、同じapproval chain内で`approval_sequence`をexact `N+1`、current head CASで一件だけ進める。Future Definition内容を変更する場合だけBundle `revision`をexact `N+1`として新hashを承認し、内容不変のexpiry／revocation後renewalは同じdefinition hash／revisionを保持してapproval sequenceだけを進める。内容不変revision bump、内容変更を伴う同revision、sequence gap／branchを拒否する。expired／revoked approvalのportfolioはplanning表示も`unapproved`とし、active Product対応を意味しない。

`FuturePortfolioPolicyRegistryV1`の初期entryは次の2件だけである。Bundleの二つのpolicy refはこのexact IDへ解決し、policy RegistryをFuture closure manifestへ含める。

| policy_id | policy_kind | authority_role_ref | signed_record_purpose | required_decision_kind |
|---|---|---|---|---|
| `policy.future.membership.revision.v1` | `membership_revision` | `role.future-portfolio-approver.r4` | `future_portfolio_definition_approval` | `null` |
| `policy.future.claim.release.v1` | `claim_release` | `role.future-claim-release.r4` | `future_product_claim_release` | `future_claim_release_product` |

planning-only entryの追加、説明修正、claim vocabulary更新、Future dependency変更はFuture membership revisionだけを更新し、current active Product Definition、WP lifecycle、Activation、Phase Evidenceを失効させない。Future entryをactiveへ移すChangeSetだけがdestination Active Product Definitionを変更し、destination `ControlPlaneRebaselineApprovalV1`／Envelope／Transaction、Product migration Decision、full-reset state migrationを必要とする。初回Bootstrap Approvalを再発行または後続Definitionへ流用しない。`prerequisite_future_capability_refs[]`は同じapproved Future closure内のexact IDだけを参照し、自己参照、missing、cycleを拒否する。前提Futureはそれぞれactive definitionへ移行し必要Targetでproductionになるまでconsumer entryをactive migration候補にしない。

Future起源のactive migrationでは`future_promotion_inputs[]`にsource Future closure hash、その時点のcurrent完成`FuturePortfolioApprovalV1` ref／hash、Future ID、追加／保持するactive Registry IDを列挙し、migration authorization Decision、row migration manifestと一緒に承認する。Approvalのpayloadが同じsource Future closure hashを承認し、current approval headで有効、非失効でなければならない。同一definition hashのrenewalでもapproval ref／hash／sequenceを省略しない。配列はFuture ID、内側IDは`{registry_id,row_kind,logical_id}`のunsigned byte tuple順で、duplicate、source entry不存在、row manifestにないID、`removed` IDを拒否する。migration完成後、専用`service.future-to-active-promotion-manifest-publisher`／`role.future-to-active-promotion-manifest-publisher`／singleton purpose `future_to_active_promotion_manifest`が、migration inputのsource Approval ref／hashをbyte-exactに複写した同じmappingと完成migration wrapper ref／hashを`FutureToActivePromotionManifestV1`として署名する。`promotion_manifest_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:future-promotion:sha256:<lowercase-hex>`として導出し、`signed_record.issued_at=payload.generated_at`、revocation snapshotをexact一致させる。これをFuture IDとactive IDsの唯一のorigin mappingとし、名前一致や説明文から由来を推測しない。

`excluded_current_product_claims[]`はplanning中の禁止claimであり、active migrationだけで自動解除しない。解除は、valid `FutureToActivePromotionManifestV1`が列挙するactive Capability、必要Target全Activation `production`、fresh Release Receipt、effective open Critical Risk 0、kind `future_claim_release_product`／subject kind `future_product_claim_release`のfresh R4 `ProductOperationalDecisionV1`を閉じた`FutureProductClaimReleaseV1`を必要とする。Claim payloadのoperational snapshot ref／hashは発行時監査とcommit条件であり、validatorはそのinline rowを`activation_keys[]`でlookupしてproduction、same Candidate、freshness、promotion manifest内Capability／Target closureを再計算する。独立しない`activation_row_refs[]`を発明しない。`claim_release_id`は同Fieldを除くpayload JCS hashから`urn:mirakan:future-claim-release:sha256:<lowercase-hex>`として導出し、`role.future-claim-release.r4`とsingleton purpose keyで署名する。`signed_record.issued_at=payload.issued_at`、revocation snapshot、purpose、Roleをexact一致させる。

Claim release chainは`future_capability_id`ごとに一つで、初回だけprevious両Field=nullかつ`claim_release_sequence=1`、後続はcurrent完成Release wrapper ref／hashとexact `N+1`を持ち、per-Future current-head CASで一件だけ進める。branch、sequence gap、別Futureのprevious、filesystemのlatest探索を拒否する。read-time `effective_release`は発行時snapshotをcurrentと見なさず、Product operational current headから同じ`activation_keys[]`を毎回key lookupし、current headのactive definition hashがpayloadのdestination hashと一致し、全行がproductionかつeffective availability、same Candidate、fresh non-revoked Release Evidence、Risk 0、current Decision／Signer validityを満たす場合だけ列挙claimを解除する。無関係transitionで条件が不変ならReleaseは有効を維持するが、Definition migration、Target降格、Candidate変更、Evidence expiry／revocation、Decision／Release期限切れで即時effective unreleasedへ戻す。Marketing文言やCapability存在だけをrelease Receiptにしない。

`released_claims[]`はsource Future rowの`excluded_current_product_claims[]`に含まれ、同じsource Future closureの`ProductClaimDefinitionRegistryV1`へ解決するnon-empty `claim_id` subsetだけを許し、全解除を主張する場合はset equalityを必須とする。unknown／duplicate／別Future claim、canonical label、Marketing自由文字列を拒否する。`activation_keys[]`はPromotion Manifestが列挙した全promoted Capabilityのdestination `CapabilityRegistryV1.target_bindings[]`のうち`required | optional`全keyから決定論的に導くexact setであり、呼出元がTargetを省略できない。`production_receipt_refs[]`はそのcurrent Activation rows、`CapabilityTargetActivationBindingRegistryV1`、該当`ProductReleaseGateRegistryV1`をproductionで支えるfresh Receiptのexact closureで、missing／extra／別Candidate／別Targetを拒否する。部分claimでもActivation／Production Evidence closureを縮小しない。

`CapabilityTargetActivationBindingRegistryV1`はCapabilityの「どのEvidence provider／Gateと実行surfaceで昇格できるか」を一意にする。`CapabilityRegistryV1`の全`required | optional` bindingとentriesの`{capability_id,target_id}` set equalityを必須とし、`excluded` row、missing、extra、duplicateを拒否する。各rowは次の決定論的規則でmaterializeし、手入力でGateまたはOperationを増減しない。

1. `owner_work_package_ref`はCapability rowとexact一致する。Evidence providerは既定でOwner WP一件、下表のexact keyだけは表のWP一件へ置換する。複数候補、暗黙の依存WP、display name一致を拒否する。
2. providerごとに、`PhaseFixtureBindingRegistryV1`のうちTargetがrowの`target_id`を含み、Fixtureがprovider WPの`provided_fixture_refs[]`内の`kind=product_fixture`に含まれ、Gate Phase orderがOwner WPとprovider WPの両Phase order以上であるGateをcandidate setにする。`kind=component_qualification_fixture`はWP自体のQualificationへだけ使い、このProduct Gate候補へ混ぜない。non-emptyなら最小Phase orderを求め、そのorderにある全Gateを一件ずつ`qualification_evidence_bindings[]`へ展開する。bindingのprovider／gate／fixture／requirements／freshnessは参照先のexact値で、同じ最小Phase／Targetの全Gateを要求する。
3. `qualification_gate_refs[]`は全evidence bindingのGate exact union、`qualification_requirement_refs[]`はRequirement exact union、`freshness_policy_refs[]`はfreshness exact unionである。各unionとbinding間のmissing／extraを拒否する。
4. `AiCapabilityActivationMatrixV1.execution_surface_required_capability_refs[]`は下表のpositive十Capability exact集合、`deny_only_capability_refs[]`は`capability.product.runtime-generation-boundary`一件で、両集合の積は空とする。positive Capabilityの全Target rowでは、行が列挙する各`planning_family_ref`を[Executable Contracts](../02-foundation/executable-contracts.md#20-ai向けdiscoveryexecution候補のplanning-record未activation)の完成`PlannedOperationFamilyV1`へID／version／hashでexact解決し、familyの`reserved_candidate_ids[]`全件を`required_operation_ids[]`へ複写する。`required_operation_ids[]`はNFC UTF-8 Operation ID byte順、duplicate禁止である。`required_operation_set_sha256`はASCII `MIRAKAN_REQUIRED_OPERATION_SET_V1`、Operation count、各Operation IDのNFC UTF-8 bytesをこの順に`uint32_be` length framingしたbytesのSHA-256 lowercase hexであり、family ID、Markdown表示順、delimiter連結、prefix matchをhash入力にしない。familyの一部だけ、別version、別work item、missing／extra Operation、set hash不一致を拒否する。該当しないCapabilityの`execution_surface_bindings[]`はexact `[]`、deny-only Capabilityもpositive bindingを持たない。
5. `required_execution_state=operational`は保存されたActivation stateではなくqualification時のcurrent derived predicateである。各familyについてcurrent Contract Set、Owner Manifest、Trusted Service allowlist、Policy、Validator／closure、Diagnostic、Receipt、Signer Policy、Trust、Service Technical Qualificationの完全集合を[Executable Contracts](../02-foundation/executable-contracts.md)と[AI Security／Approval](../01-governance/ai-security-approval.md)のlike-for-like規則で再計算し、`required_operation_ids[]`全件が同じContract setでoperationalの場合だけ満たす。planning recordの`capability_state=not_activated`、一部Operationだけ、Signer rowだけ、Provider aliasだけ、stale Contract set、Provider投影過多／不足はfalseである。`provider_projection_requirement=none`はProvider／MCP projection exact `[]`、`all_activation_declared_external_safe`はActivation recordで`exposure=external_safe`と確定した全Operationのexact集合、`proposal_subset_excluding_trusted_internal`は同external-safe集合のうちread／query／plan／propose／validate／preview／requestだけのexact集合である。後二者とも`trusted_internal` Commit／Promotion／Approval／Activation／Signing／ReleaseをProviderまたはMCPへ含めず、Operation名やHTTP routeからexposureを推測しない。
6. `execution_host_rule=windows_editor_for_artifact_target`は`execution_host_target_refs=[target.windows.editor]`、`artifact_target_id`を当該Capability rowの`target_id`へする。Android／Apple artifactをWindows Editorで生成しても、Build／Cook／Package／Test ReceiptのTargetはAndroid／AppleのままでWindows Receiptを流用しない。`headless_proposal_only`は`execution_host_target_refs=[target.headless.host]`、`artifact_target_id=null`で、Commit／Promotion／Activationを外部Toolへ投影しない。
7. Evidence binding集合がnon-emptyで、かつ全`execution_surface_bindings[]`がcurrent `operational`なら`qualification_policy=all_evidence_bindings`で全bindingのfresh success、Owner WPと全provider WPのcurrent state=`complete`、same Candidateを必須にする。Evidenceがemptyなら`unavailable_no_gate`、必要実行surfaceが一件でも非operationalなら`not_activated`からの昇格を拒否する。WP完了、fixture pass、説明文のOperation名を実行surfaceの代用にしない。
8. `candidate_locked`はOwner WPのcurrent `CriticalPathTaskV1`と同じCandidateへ閉じたfresh `TechnicalQualificationReceiptV1`だけで許可する。
9. 現DefinitionはCX3 Release／Product Release Gateが未成立のため全row `production_policy={kind=disabled_current_definition, release_gate_refs=[]}`とし、`production`遷移を拒否する。`wp.product.production-release-binding`のsource `production_release_migration_authoring`が生成したpreliminary destination候補をR4 Product migration Decisionで承認した場合だけ、destination Active Definitionの各rowを、そのTargetと一致する`ProductReleaseGateRegistryV1`一件を持つ`{kind=all_release_gates, release_gate_refs=[...]}`へ変更できる。migration V1のfull reset後、同WPをdestination `production_release_binding_qualification`として再実行し、そのWPがcomplete、かつrelease gateがsame Candidateでfresh successの場合だけ`qualified->production`を許可する。Target不一致、empty／複数gate、Release WP未完了、旧definition Receiptのcarryを拒否する。

`wp.product.production-release-binding`のexecution modeはcurrent `CapabilityTargetActivationBindingRegistryV1`全293行から決定論的に導出し、Task入力として署名する。293行すべてが`disabled_current_definition`かつgate集合emptyなら`production_release_migration_authoring`、293行すべてが各Targetのexact Release Gate一件を持つ`all_release_gates`なら`production_release_binding_qualification`である。mixed kind、292以下／294以上、Target違い、empty／複数gateはinvalid Active Definitionであり、同WPを開始しない。

`production_release_migration_authoring`はGateで既にcurrent validatedな`evidence.class.product-release-policy-ready`／`evidence.class.product-release-artifact-plan-valid`の完成Receipt ref／hashをread-backし、内容を新規作成または自己発行しない。その入力をdestination Active 14 Registry closure、全293 policy rowを変更するsigned row migration入力、Target別plan projection、Control Plane Rebaseline入力へbyte-exactに組み込み、preliminary Staging outputとして生成するだけである。source lifecycleは`declared->ready->active`までを許可し、`active->complete`、`ArtifactCandidateBindingV1`、Activation／Release Evidenceへの利用を禁止する。R4 DecisionとA2→B2→C2→D2→T2→L2→P2 migrationがsource active epochを置換し、destinationへsource WP state／Receiptをcarryしない。

`production_release_binding_qualification`はActive Definition、Registry、row migration manifest、Rebaseline artifactを一byteも変更しない。同じdestination Definition／Candidateへfresh Target artifact、package、support、rollback、signing、SBOM／provenance、device-lab、Release Receipt policy Evidenceを閉じ、独立Owner acceptance付き`ArtifactCandidateBindingV1`でのみ`active->complete`へ進む。このcompletionはRelease Gateを自動満足またはActivation／human Release Approvalを発行せず、後続Gateがsame Candidate、freshness、critical Riskを別途再評価する。

| capability_id | target_id | exact evidence provider WP |
|---|---|---|
| `capability.platform.input-core` | `target.windows.editor` | `wp.platform.input-windows` |
| `capability.platform.input-core` | `target.windows.desktop` | `wp.platform.input-windows` |
| `capability.platform.input-core` | `target.android.mobile` | `wp.platform.mobile-io-ui-android` |
| `capability.platform.input-core` | `target.apple.mobile` | `wp.platform.mobile-io-ui-apple` |
| `capability.platform.audio-core` | `target.windows.editor` | `wp.platform.audio-windows` |
| `capability.platform.audio-core` | `target.windows.desktop` | `wp.platform.audio-windows` |
| `capability.platform.audio-core` | `target.android.mobile` | `wp.platform.mobile-io-ui-android` |
| `capability.platform.audio-core` | `target.apple.mobile` | `wp.platform.mobile-io-ui-apple` |
| `capability.platform.ui-core` | `target.windows.editor` | `wp.platform.ui-windows` |
| `capability.platform.ui-core` | `target.windows.desktop` | `wp.platform.ui-windows` |
| `capability.platform.ui-core` | `target.android.mobile` | `wp.platform.mobile-io-ui-android` |
| `capability.platform.ui-core` | `target.apple.mobile` | `wp.platform.mobile-io-ui-apple` |

この12行は`CapabilityTargetActivationBindingRegistryV1` row内のprovider Fieldへmaterializeする正本例外であり、別のoperational stateではない。各keyは対応Capability Target bindingとset inclusion、provider WP Targetと一致し、その他のkeyはOwner WP既定を使う。これにより後段のI/O／Audio／UI Target adapter Receipt要件と同一schemaで判定できる。

`AiCapabilityActivationMatrixV1.rows[]`のcurrent exact十行は次である。`planning family refs`は表示時のbare planning IDで、materialization時は[Executable Contracts §20](../02-foundation/executable-contracts.md#20-ai向けdiscoveryexecution候補のplanning-record未activation)の`{planning_record_id, planning_record_version, planning_record_hash}`へ展開する。配列はcanonical ID順、重複禁止であり、略称、family count、Operation prefix、`latest`参照を保存しない。

| capability_id | owner WP | Requirement refs | Phase | Fixture refs | planning family refs | host rule | Provider projection | Risk refs | Policy binding `{state transition refs; candidate binding literal}` |
|---|---|---|---|---|---|---|---|---|---|
| `capability.authoring.ai-core` | `wp.authoring.ai-core` | `requirement.product.ai-authoring-mvp-a; requirement.product.ai-genre-neutral-authoring` | `phase.ai-authoring-mvp-a` | `fixture.product.genreless-ai-project; fixture.product.shooter-2d` | `planning.operation_family.authoring_discovery; planning.operation_family.game_system_discovery; planning.operation_family.world_discovery; planning.operation_family.gameplay_definition_authoring; planning.operation_family.asset_authoring; planning.operation_family.authoring_changeset_execution` | `windows_editor_for_artifact_target` | `all_activation_declared_external_safe` | `risk.product.ai-tool-safety-code-owner` | `policy.product.activation.promotion.v1; policy.product.same-candidate.v1` |
| `capability.runtime.debug-replay-support` | `wp.runtime.debug-replay-support` | `requirement.product.ai-authoring-mvp-a; requirement.product.ai-genre-neutral-authoring` | `phase.ai-authoring-mvp-a` | `fixture.product.genreless-ai-project; fixture.product.shooter-2d` | `planning.operation_family.build_device_play_debug_task` | `windows_editor_for_artifact_target` | `all_activation_declared_external_safe` | `risk.product.ai-tool-safety-code-owner` | `policy.product.activation.promotion.v1; policy.product.same-candidate.v1` |
| `capability.runtime.ecs-e6-debug-ai` | `wp.runtime.ecs-e6-debug-ai` | `requirement.product.ai-authoring-mvp-a; requirement.product.ai-genre-neutral-authoring` | `phase.ai-authoring-mvp-a` | `fixture.product.genreless-ai-project; fixture.product.shooter-2d` | `planning.operation_family.authoring_discovery; planning.operation_family.game_system_discovery; planning.operation_family.world_discovery; planning.operation_family.gameplay_definition_authoring; planning.operation_family.asset_authoring; planning.operation_family.authoring_changeset_execution; planning.operation_family.build_device_play_debug_task` | `windows_editor_for_artifact_target` | `all_activation_declared_external_safe` | `risk.product.ai-tool-safety-code-owner` | `policy.product.activation.promotion.v1; policy.product.same-candidate.v1` |
| `capability.product.ai-authoring-mvp-a` | `wp.product.ai-authoring-mvp-a` | `requirement.product.ai-authoring-mvp-a; requirement.product.ai-genre-neutral-authoring; requirement.product.authoring-roundtrip; requirement.product.mvp-completion` | `phase.ai-authoring-mvp-a` | `fixture.product.genreless-ai-project; fixture.product.shooter-2d` | `planning.operation_family.authoring_discovery; planning.operation_family.game_system_discovery; planning.operation_family.world_discovery; planning.operation_family.gameplay_definition_authoring; planning.operation_family.asset_authoring; planning.operation_family.authoring_changeset_execution; planning.operation_family.build_candidate_test; planning.operation_family.build_device_play_debug_task` | `windows_editor_for_artifact_target` | `all_activation_declared_external_safe` | `risk.product.ai-tool-safety-code-owner` | `policy.product.activation.promotion.v1; policy.product.same-candidate.v1` |
| `capability.product.external-agent` | `wp.product.external-agent` | `requirement.product.external-agent-boundary` | `phase.external-agent` | `fixture.product.external-agent-proposal` | `planning.operation_family.authoring_discovery; planning.operation_family.game_system_discovery; planning.operation_family.world_discovery; planning.operation_family.gameplay_definition_authoring; planning.operation_family.asset_authoring; planning.operation_family.authoring_changeset_execution; planning.operation_family.build_candidate_test; planning.operation_family.build_device_play_debug_task` | `headless_proposal_only` | `proposal_subset_excluding_trusted_internal` | `risk.product.ai-tool-safety-code-owner` | `policy.product.activation.promotion.v1; policy.product.same-candidate.v1` |
| `capability.project.native_module` | `wp.authoring.project-native-module` | `requirement.product.core-pack-independence; requirement.product.project-source-activation` | `phase.external-agent` | `fixture.product.genreless-ai-project` | `planning.operation_family.native_game_module_source; planning.operation_family.project_source_promotion; planning.operation_family.build_candidate_test; planning.operation_family.authoring_changeset_execution` | `windows_editor_for_artifact_target` | `all_activation_declared_external_safe` | `risk.product.ai-tool-safety-code-owner` | `policy.product.activation.promotion.v1; policy.product.same-candidate.v1` |
| `capability.project.shader` | `wp.rendering.project-shader` | `requirement.product.core-pack-independence; requirement.product.project-source-activation` | `phase.external-agent` | `fixture.product.genreless-ai-project` | `planning.operation_family.project_shader_discovery; planning.operation_family.project_source_promotion; planning.operation_family.build_candidate_test; planning.operation_family.authoring_changeset_execution` | `windows_editor_for_artifact_target` | `all_activation_declared_external_safe` | `risk.product.ai-tool-safety-code-owner` | `policy.product.activation.promotion.v1; policy.product.same-candidate.v1` |
| `capability.product.project-source-activation` | `wp.product.project-source-activation` | `requirement.product.core-pack-independence; requirement.product.project-source-activation` | `phase.external-agent` | `fixture.product.genreless-ai-project` | `planning.operation_family.native_game_module_source; planning.operation_family.project_shader_discovery; planning.operation_family.project_source_promotion; planning.operation_family.build_candidate_test; planning.operation_family.build_device_play_debug_task; planning.operation_family.authoring_changeset_execution` | `windows_editor_for_artifact_target` | `all_activation_declared_external_safe` | `risk.product.ai-tool-safety-code-owner` | `policy.product.activation.promotion.v1; policy.product.same-candidate.v1` |
| `capability.product.general_production_2d` | `wp.product.general-coverage-2d` | `requirement.product.core-pack-independence; requirement.product.c2-2d-coverage` | `phase.production-capability` | `fixture.product.genreless-core-2d; fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d` | `planning.operation_family.authoring_discovery; planning.operation_family.game_system_discovery; planning.operation_family.world_discovery; planning.operation_family.gameplay_definition_authoring; planning.operation_family.asset_authoring; planning.operation_family.authoring_changeset_execution; planning.operation_family.feature_authoring; planning.operation_family.camera_authoring; planning.operation_family.material_authoring; planning.operation_family.vfx_authoring; planning.operation_family.environment_authoring; planning.operation_family.input_binding_selection; planning.operation_family.navigation_binding_selection; planning.operation_family.physics_role_selection; planning.operation_family.build_candidate_test; planning.operation_family.build_device_play_debug_task` | `windows_editor_for_artifact_target` | `all_activation_declared_external_safe` | `risk.product.ai-tool-safety-code-owner` | `policy.product.activation.promotion.v1; policy.product.same-candidate.v1` |
| `capability.product.general_production_3d` | `wp.product.general-coverage-3d` | `requirement.product.first-playable-3d` | `phase.production-capability` | `fixture.product.shooter-arena-3d` | `planning.operation_family.authoring_discovery; planning.operation_family.game_system_discovery; planning.operation_family.world_discovery; planning.operation_family.gameplay_definition_authoring; planning.operation_family.asset_authoring; planning.operation_family.authoring_changeset_execution; planning.operation_family.feature_authoring; planning.operation_family.camera_authoring; planning.operation_family.material_authoring; planning.operation_family.vfx_authoring; planning.operation_family.environment_authoring; planning.operation_family.input_binding_selection; planning.operation_family.navigation_binding_selection; planning.operation_family.physics_role_selection; planning.operation_family.rendering_aa_discovery; planning.operation_family.lighting_discovery; planning.operation_family.post_process_discovery; planning.operation_family.math_semantic_authoring; planning.operation_family.lod_authoring; planning.operation_family.build_candidate_test; planning.operation_family.build_device_play_debug_task` | `windows_editor_for_artifact_target` | `all_activation_declared_external_safe` | `risk.product.ai-tool-safety-code-owner` | `policy.product.activation.promotion.v1; policy.product.same-candidate.v1` |

最終列は配列ではなく、`AiCapabilityActivationMatrixV1`の二Fieldを`{state_transition_policy_refs[]; candidate_binding_policy_literal}`の順に表示したsource notationである。前半は`ProductOperationalStatePolicyRegistryV1`へ解決するexact一件のRegistry ref、後半はRegistry lookupを行わない`ProductCandidateBindingPolicyLiteralV1`の固定literalである。compilerは二Fieldへ別々にmaterializeし、順序交換、片方欠落、二値を同じRegistryへ解決する実装、任意の三値目を拒否する。

`capability.product.general_production_3d`の行は必要実行surfaceを先に固定するが、`wp.product.general-coverage-3d`が`deferred`で第二の非Shooter 3D Fixtureを持たないためcurrent qualificationへ使えない。`gate.product.reconsider-c2-3d`が要求するActive Definition migrationで第二fixture、Requirement、WP、Target bindingを同時追加するまで、Shooter fixture一件を汎用3DのEvidenceとして代用しない。

24 planning familyはpositive十行のunionへexact全件接続する。`rendering_aa_discovery`、`lighting_discovery`、`post_process_discovery`、`math_semantic_authoring`、`lod_authoring`のexact五件はGeneral Production 3D行だけが追加要求し、同rowがdeferredの間はProduct bindingが存在してもActivation／Provider公開／Capability昇格を許可しない。これらをMVP-AまたはGeneral 2Dへ暗黙追加せず、逆にC2 3D移行時に説明名だけで省略しない。matrix compilerは24 family集合と全positive rowのfamily unionをset equalityにし、unboundまたはunknown familyをActive Definition compile errorにする。

`deny_only_capability_refs[]`のcurrent exact一件は`capability.product.runtime-generation-boundary`である。同Capabilityと`wp.product.runtime-generation`のpositive `execution_surface_bindings[]`はexact `[]`とし、`fixture.product.runtime-generation-denial`でRuntime packageにOrchestrator、Provider credential、MCP server、Source Worker、Compiler、Signer、write-capable Gatewayが0件であることだけをQualificationする。将来Runtime generationを追加する場合は本行をsilent変更せず、Future Portfolio→Active Definition migration、Threat Model、Target別Operation family、Signer／Service／Receipt closure、positive／negative fixtureを一つの変更として承認する。

`matrix_sha256`はASCII `MIRAKAN_AI_CAPABILITY_ACTIVATION_MATRIX_V1`、matrix ID／version、positive十Capability集合、deny-only一Capability集合、全十rowのself-excluding canonical bytesを`uint32_be` length framingしてSHA-256する。rowは`capability_id`、内部ref集合はNFC UTF-8 byte順でstrict sortする。matrix row集合とpositive集合、row owner WPとCapability Registry owner WP、Requirement／Phase／Fixtureと各Registry、planning family ID／version／hash／work item／candidate ID setをexact比較する。missing／extra row、同じfamilyの部分subset、Targetをhostとartifactの一Fieldへ圧縮、Runtime generationへのpositive binding、unknown FieldをActive Definition compile errorにする。

このRegistryによりTier、WP existence、provided fixtureだけからActivationを推測しない。optional rowも同じEvidence／execution mappingを持つがProduct aggregateには含めない。current 24 planning family／192候補は全て`not_activated`であり、上表のpositive十CapabilityはWP状態にかかわらず現時点で`qualified`になれない。

Snapshot chainは`sequence=0`、`previous_state_snapshot_ref=null`、`applied_change.kind=genesis`から始める。genesisはfinal Bootstrap Approval、D baseline envelope、construction Decision、Task 0 trust provisioning Receiptをexact参照し、A→B→C→D ancestor／hashをread-backする。全`required | optional` Capability bindingのrowを`not_activated, candidate_ref=null, receipt_refs=[]`で生成し、Decision Gate／Risk表に示す初期state、全WPのnull head／sequence 0を持つ。

ただしControl Plane bootstrap ceremonyの外部公開current headはsequence 0ではない。Task 10Bはfinal D envelopeをread-backし、Approval／Signer／Role／Keyのvalidityとcurrent revocationを再検証した後、candidate生成開始時に一度だけ`bootstrap_transaction_time`とcandidate `revocation_snapshot_ref`を採番する。同一atomic transaction内で、(a)その時刻を`created_at`に持つsigned genesis snapshot sequence 0、(b)それをparentとし同時刻を`recorded_at`に持つ`wp.architecture.control-plane`専用lifecycle sequence 1 `declared->complete`、(c) lifecycle wrapperを`applied_change.kind=wp_lifecycle`として適用し同時刻を`created_at`に持つsnapshot sequence 1を順に生成し、全wrapperのsigned-record issued_atとrevocation snapshotを型規則どおり一致させ、current pointerを(c)へ一度だけCASする。sequence 1ではControl Plane headだけがlifecycle sequence 1、他の全WP headはnull／0である。(a)と(b)は監査ancestorとして保存するが単独currentとして公開せず、いずれかの生成／署名／検証に失敗した場合はcurrent pointerを一切作らない。同じcandidateのcrash／idempotent retryは保存済み`bootstrap_transaction_time`、candidate revocation snapshot、canonical bytes、signatureをbyte-exact再利用し、別event timeで再署名しない。一方、各CAS attemptはjournalのactual `publication_time`をfresh取得し、直前にcurrent Root／local／global／Readiness／Trust／Catalog／Future Approval、Product signerのIdentity／Role／assignment／purpose Key／revocation、expected-empty operational parentをread-backする。candidateが束縛したauthority／validity／publication windowと一件でもdriftした場合はcandidateを変更せずterminal abortし、該当Control Plane規則のfresh Authorization／Task 0またはF renewalからnew candidateを生成する。parentがnon-emptyなら同じcandidateの既存publicationをread-backしexact一致時だけ成功回復、別candidateなら競合quarantineとする。通常Critical Pathはsequence 1のread-back後にsequence 2から開始する。sequence 2以降はstate transition／lifecycle／definition migration／same-definition baseline rebindingのいずれでも、next snapshotの`created_at`を適用payloadの`recorded_at`、`revocation_snapshot_ref`を同payload値に固定する。`applied_change`のref／hashは完成した同一wrapperとexact一致し、同じ適用wrapperまたは同じretryから別event time、別candidate revocation snapshot、別canonical bytes、複数のnext snapshotを生成しない。

以後のnext snapshotは`applied_change`でexactly oneの完成signed transition／lifecycle／definition migration／baseline rebinding wrapperをref／hash束縛し、wrapperのparent、subject、from／toまたはmigration／rebinding projectionとsnapshot diffを一対一で再計算する。複数change、missing wrapper、別parent、指定外row差を拒否する。current snapshot wrapper hashへのcompare-and-swapでexactly one次snapshotだけをcurrentにする。同じprevious wrapper hashから複数candidateが生じた場合は一件だけをacceptし、他を`diagnostic.product.operational-state-fork`で拒否する。同じcanonical payloadの再送は再署名せず既にpublish済みのexact wrapperを返し、同payloadを別signature wrapperとして分岐させない。Snapshot logical IDは`state_snapshot_id`を除く`ProductOperationalStateSnapshotPayloadV1`のRFC 8785 JCS SHA-256から`urn:mirakan:product-state:sha256:<lowercase-hex>`として導出する。`signed_record.subject_sha256`はIDを含む完成payloadのJCS hashであり、wrapper、signature、signed recordをID derivationへ含めない。`previous_state_snapshot_ref`、current head、CAS refはすべて署名検証済み完成wrapperの`sha256:<JCS wrapper lowercase-hex>` content refであり、payload logical IDではない。ref解決時にwrapper hash、signature、payload ID derivationをすべて再計算する。

通常のnext snapshotはexactly oneの署名済みtransitionを適用して生成する。Lifecycle transitionだけは`WorkPackageLifecycleRecordV1`自身をtransitionとして用い、別の重複Recordを作らない。

```text
ProductOperationalStateTransitionPayloadV1
  transition_id
  active_product_definition_sha256
  parent_state_snapshot_ref
  parent_state_snapshot_sha256
  change:
    {kind: activation,
     capability_id, target_id, from_state, to_state,
     prior_candidate_ref: null | exact candidate ref,
     next_candidate_ref: null | exact candidate ref,
     prior_receipt_refs[], next_receipt_refs[]}
    | {kind: decision_gate_evaluation,
       gate_id, from_state, to_state,
       prior_evidence_refs[], next_evidence_refs[]}
    | {kind: risk_evaluation,
       risk_id, from_state, to_state,
       prior_evidence_refs[], next_evidence_refs[]}
  state_policy_ref
  authorization_decision_ref: approved ProductOperationalDecisionV1 ref
  requested_by_subject_ref
  recorded_at
  revocation_snapshot_ref

ProductOperationalStateTransitionV1
  payload: ProductOperationalStateTransitionPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=product_operational_state_transition)

ProductOperationalDecisionPayloadV1
  decision_id
  decision_kind
  definition_binding_kind = current_parent | destination
  active_product_definition_sha256
  owner_document_id
  owner_document_sha256
  subject_kind
  subject_ref
  subject_sha256
  evidence_refs[]
  disposition = approved | rejected
  approver_subject_ref
  approval_authority_ref
  issued_at
  valid_until
  revocation_snapshot_ref

ProductOperationalDecisionV1
  payload: ProductOperationalDecisionPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=product_operational_decision)

ProductOperationApprovalSubjectV1
  schema_id = urn:mirakan:schema:product:operation-approval-subject:v1
  payload_kind = product_state_transition | work_package_lifecycle | active_definition_migration | control_plane_baseline_rebinding | future_product_claim_release
  approval_projection

ProductDefinitionRowMigrationManifestPayloadV1
  manifest_id
  source_active_product_definition_sha256
  destination_active_product_definition_sha256
  rows[]:
    {registry_id,
     row_kind = standard | evidence_class | definition_change_class | decision_gate,
     logical_id, migration_kind,
     source_row_sha256, destination_row_sha256}
  generated_at
  revocation_snapshot_ref

ProductDefinitionRowMigrationManifestV1
  payload: ProductDefinitionRowMigrationManifestPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=active_product_definition_row_migration_manifest)

ActiveProductDefinitionMigrationPayloadV1
  migration_id
  source_active_product_definition_sha256
  destination_active_product_definition_sha256
  source_state_snapshot_ref
  source_state_snapshot_sha256
  destination_rebaseline_approval_ref
  destination_rebaseline_approval_sha256
  destination_rebaseline_envelope_ref
  destination_rebaseline_envelope_sha256
  control_plane_rebaseline_transaction_ref
  control_plane_rebaseline_transaction_sha256
  architecture_definition_migration_binding_ref: optional ArchitectureDefinitionMigrationBindingRefV1
  architecture_definition_migration_binding_sha256: optional SHA-256
  authorization_decision_ref: approved ProductOperationalDecisionV1 ref
  row_migration_manifest_ref
  row_migration_manifest_sha256
  future_promotion_inputs[]:
    {source_future_portfolio_definition_sha256,
     source_future_portfolio_approval_ref, source_future_portfolio_approval_sha256,
     future_capability_id,
     promoted_active_ids[]: {registry_id, row_kind, logical_id, migration_kind}}
  state_policy_ref = policy.product.state.definition-migration.v1
  destination_capability_target_rows[]
  destination_decision_gate_evaluations[]
  destination_risk_evaluations[]
  destination_work_package_heads[]
  control_plane_rebaseline_lifecycle_wrapper_ref
  control_plane_rebaseline_lifecycle_wrapper_sha256
  requested_by_subject_ref
  recorded_at
  revocation_snapshot_ref

ActiveProductDefinitionMigrationV1
  payload: ActiveProductDefinitionMigrationPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=active_product_definition_migration)

ControlPlaneBaselineRebindingPayloadV1
  rebinding_id
  source_state_snapshot_ref, source_state_snapshot_sha256
  active_product_definition_sha256
  source_control_plane_baseline_binding: CurrentControlPlaneBaselineBindingV1
  destination_control_plane_baseline_binding: CurrentControlPlaneBaselineBindingV1(kind=rebaseline)
  rebaseline_approval_ref, rebaseline_approval_sha256
  rebaseline_envelope_ref, rebaseline_envelope_sha256
  rebaseline_transaction_ref, rebaseline_transaction_sha256
  control_plane_lifecycle_wrapper_ref, control_plane_lifecycle_wrapper_sha256
  state_policy_ref = policy.product.state.control-plane-rebaseline.v1
  authorization_decision_ref: approved ProductOperationalDecisionV1 ref
  requested_by_subject_ref
  recorded_at
  revocation_snapshot_ref

ControlPlaneBaselineRebindingV1
  payload: ControlPlaneBaselineRebindingPayloadV1
  signed_record: MirakanSignedRecordV1(purpose=control_plane_baseline_rebinding)
```

`transition_id`は同Fieldを除くpayloadのJCS SHA-256から`urn:mirakan:product-state-transition:sha256:<lowercase-hex>`として導出し、wrapper／signatureを含めない。transitionのparentはcurrent signed snapshot wrapperとexact一致し、next snapshotはtagged `change`が指定する一行以外の全row／headをbyte-exactに保持する。activationのprior candidate／Receiptはparent row、next値はnext rowとexact一致する。Decision／Riskのprior／next Evidenceは対応evaluation rowとexact一致し、Receipt content refの重複なしunsigned UTF-8 byte順集合である。logical Gate IDをEvidence refへ入れない。`requested_by_subject_ref`は変更要求主体であり、`authorization_decision_ref`は当該edgeのPolicyが指定するkindのfresh approved `ProductOperationalDecisionV1`である。要求主体やpublisher署名を承認へ読み替えない。

`ProductOperationalDecisionV1`だけを通常Product state／通常WP lifecycle／baseline rebinding／claim releaseの承認Decision型として受理する。`decision_kind`は`activation_owner_approval | deactivation_r4 | decision_gate_owner_approval | risk_monitoring_owner_approval | risk_acceptance_r4 | risk_mitigation_owner_approval | risk_closure_owner_approval | work_package_owner_transition | work_package_defer_release_product | work_package_owner_acceptance | active_definition_migration_architecture_product | control_plane_baseline_rebinding_product | future_claim_release_product`、`subject_kind`は`product_state_transition | work_package_lifecycle | active_definition_migration | control_plane_baseline_rebinding | future_product_claim_release`のclosed enumである。definition migration／same-definition rebinding内のControl Plane lifecycle Recordは、Control Plane Ownerの`ControlPlaneRebaselineApprovalV1`を専用authorization型として使い、Product Decisionに偽装しない。Product snapshotのbinding切替そのものは対応するProduct Decisionを別途必須にする。

Decisionと実行payloadのhash循環を避けるため、`subject_ref`／`subject_sha256`は実行payload wrapperではなく、closed `ProductOperationApprovalSubjectV1`の完成JCS bytesを指す。`approval_projection`は対象payloadから次のFieldだけを除いたobjectとする。state transitionは`transition_id, authorization_decision_ref, recorded_at, revocation_snapshot_ref`、WP lifecycleは`lifecycle_record_id, authorization_record_ref, recorded_at, revocation_snapshot_ref`、definition migrationは`migration_id, authorization_decision_ref, recorded_at, revocation_snapshot_ref`、baseline rebindingは`rebinding_id, authorization_decision_ref, recorded_at, revocation_snapshot_ref`、Future claim releaseは`claim_release_id, authorization_decision_ref, issued_at, valid_until, revocation_snapshot_ref`を除く。それ以外のFieldは階層、null、配列順を含めbyte-equivalentに保持し、追加Field、説明文、将来のnext snapshot、publisher metadataを入れない。手順は(1) unsigned実行payload候補からprojectionを生成、(2) `ProductOperationApprovalSubjectV1`をcontent-address、(3) Decisionを署名、(4) authorization refを実行payloadへ挿入、(5)実行payload IDとpublisher署名を生成、の一方向だけである。Decision検証時は完成実行payloadからprojectionを再生成してsubject ref／hashと一致させる。

`decision_id`は同Fieldを除くpayloadのJCS SHA-256から`urn:mirakan:product-decision:sha256:<lowercase-hex>`として導出する。commit authorizationでは`disposition=approved`、`decision.issued_at <= execution_payload.recorded_at < decision.valid_until`（claim releaseは`execution_payload.issued_at`）、execution payloadが束縛するrevocation snapshot時点でDecision／Signer／Role／assignment／Keyが非revoked、active definition hash、Owner document bytes hash、approval subject、必要Evidenceが一致するときだけ受理する。`rejected`、commit時期限切れ、別subject、別Owner、別definition、commit時stale Evidenceを流用しない。

Definition bindingはkind別に固定する。通常state transition、通常／defer WP lifecycle、baseline rebinding、Future claim releaseは`definition_binding_kind=current_parent`で、execution parent/current operational snapshotのdefinition hashと一致する。baseline rebindingはsource／destination binding内のdefinition hashも同じcurrent hashでなければならない。`active_definition_migration_architecture_product`だけは`definition_binding_kind=destination`で、まだcurrentでなくてもmigration payload／approval projectionが束縛するdestination definition hashと一致し、同projection内のsource snapshot hashがcurrent sourceを指すことを検証する。他kindでdestination、migration kindでsource hashを入れることを拒否する。これによりdestinationをcurrentと偽装せず、かつ移行前にdestination承認を発行できる。

Architecture Owner transferを含む`ActiveProductDefinitionMigrationPayloadV1`では、`architecture_definition_migration_binding_ref`／hashを対で必須にし、hashはresolved Bindingの`binding_content_hash`とexact一致させる。[Architecture Governance](../01-governance/architecture-governance.md#33-definition-migration-binding)の完成Bindingへexact解決する。Product-only Definition migrationでは両Fieldをcanonical omissionする。Binding→Subjectが持つsource／target active Product Definition hashはpayloadの同名Fieldと、source／target Architecture Inventory、Owner Registry、Contract Set、Foundation Definition Closure、Owner reference migration manifest、Compatibility Change、Consumer Inventory、全Evidence Requirementのpass satisfaction bindingはそれぞれ同じmigration closureと一致しなければならない。Product payloadはBindingを参照するだけでBindingへ自分自身を戻さず、Architecture ChangeSetもProduct wrapperをhash preimageへ戻さない。これによりProduct current pointer切替とArchitecture Owner transferを同じclosureで検証しつつhash cycleを作らない。

後日の通常expiryは、当時正当にcommit済みのimmutable transition／lifecycle／migration chainと署名の履歴整合性を壊さない。Verifierは現在時刻ではなくrecorded／issued時点の上記条件を再現する。ただしrevocationの`effective_at`がcommit時刻以前なら鍵侵害等のretroactive invalidationとしてchainをquarantineし、登録済みrecovery policy以外の新規遷移を停止する。commit後を`effective_at`とするrevocationは過去Recordを監査履歴として維持するが、そのDecisionを新規実行へ再利用できない。Risk acceptance、Future claim release、Capability effective availabilityなどPolicyが継続的権限を要求するprojectionだけはcurrent time／current revocation／current Evidenceで再評価し、失効時にeffective state／claimをfail closedへ戻す。

Decision signerはcurrent `role.product-operational-decision.r4`へのactive assignmentを持つ人間主体でなければならず、assignmentのclosed `decision_scopes[] {owner_document_id, allowed_decision_kinds[]}`にpayloadのOwner／kindがexactに含まれ、Role permissionが`R4_PRODUCT_OPERATIONAL_DECISION`でなければならない。AI、requester、対象実装worker、publisher service、対象Evidence発行者からのindependenceをcurrent Identity／Role／assignment registryで検証する。purpose専用Keyの`allowed_signed_record_purposes`はsingleton `[product_operational_decision]`とし、payloadのapprover／authority／issued_at／revocation snapshotと外側signed recordをexact一致させる。Owner文書IDはcurrent generated Architecture Document Registryへ解決し、`owner_document_sha256`はそのapproved bytesと一致させる。自由なApproval URL、Issue、Markdown文字列、別Decision schemaを代用しない。

Decisionのexpected Ownerはsubjectから次のclosed mappingで導出し、`payload.owner_document_id`／hashとexact一致させる。Activationは`CapabilityTargetActivationBindingRegistryV1`のCapability→owner WP→`WorkPackageRegistryV1.owner_document_id`、WP lifecycleは対象WP rowのOwner、Decision GateはGate rowのOwner、RiskはRisk definition rowのOwnerを使う。Active Definition migrationとsame-definition baseline rebindingのProduct判断は`mirakan.arch.product-plan`を使い、別途`ControlPlaneRebaselineApprovalV1`がArchitecture側を承認する。Future claim releaseはPromotion Manifestが束縛するsource Future closure内の対象Future rowのOwnerを使う。subject kindとDecision kindがこのmappingにない組合せ、caller申告Owner、現在hashだけが一致する別Document、複数Owner候補を拒否する。

Publisherはleast-privilege Service Roleをpurpose別に分離する。Snapshotは`service.product-state.snapshot-publisher`＋`role.product-state.snapshot-publisher`＋purpose `product_operational_state_snapshot`、state transitionは`service.product-state.transition-publisher`＋`role.product-state.transition-publisher`＋purpose `product_operational_state_transition`、WP lifecycleは`service.product-state.wp-lifecycle-publisher`＋`role.product-state.wp-lifecycle-publisher`＋purpose `work_package_lifecycle_transition`、same-definition baseline rebindingは`service.product-state.control-plane-rebinder`＋`role.product-state.control-plane-rebinder`＋purpose `control_plane_baseline_rebinding`を使う。各Role assignment、current revocation snapshot、singleton-purpose non-exportable KeyをServer側で検証し、subject／Role／Key／purpose流用を拒否する。全wrapperの`signed_record.subject_sha256`はIDを含む完成payload JCS hash、`signed_record.revocation_snapshot_ref`はpayloadの同Fieldとexact一致させる。時刻Fieldは型別に、snapshot=`signed_record.issued_at == payload.created_at`、state transition／WP lifecycle／definition migration／baseline rebinding=`issued_at == recorded_at`、Product Decision=`issued_at == payload.issued_at`、row migration manifest=`issued_at == generated_at`とする。存在しないgeneric `payload.issued_at`を推測せず、時刻不一致を拒否する。R4 Owner／Risk／Architecture判断は上記Decision wrapperとして別検証し、publisher ServiceへR4判断権を与えない。requesterまたはOwnerをpublisher signerとして推測しない。

`ProductDefinitionRowMigrationManifestV1`の`migration_kind`は`added | removed | retained`だけである。Registry rowはinlineであり独立content refを発明しない。`rows[]`は`{registry_id,row_kind,logical_id}`のunsigned UTF-8 byte tuple順、重複なしとし、source／destination closureの全row unionとset equalityでなければならない。通常の単一`entries[]` Registryは`row_kind=standard`、`ProductDecisionGateRegistryV1`だけは上記三つの専用kindを使う。validatorは各closureのregistryを`registry_id`と`row_kind`で解決し、rowをlogical IDでexact lookupして`SHA-256(JCS(row))`を再計算する。`added`はsource hash=null／destination hash=non-null、`removed`は逆、`retained`は両方non-nullを必須とし、retainedのbytesが変わる場合も同じlogical IDの旧新hashを明示する。`manifest_id`は同Fieldを除くpayload JCS SHA-256から`urn:mirakan:product-definition-row-migration:sha256:<lowercase-hex>`として導出する。`service.product-state.definition-migrator`／`role.product-state.definition-migrator`のpurpose専用Keyで署名し、source／destination closure、row lookup／hash、sort、ID、署名、current revocationを再計算する。migration payloadのmanifest ref／hashはこの完成signed wrapperを指し、未署名JSON、部分集合、説明文diffを拒否する。

row manifestはProduct Registry rowの完全性を示すだけで、Architecture Owner transfer、Compatibility Consumer Inventory、Owner reference migration manifestを代用しない。Architecture Owner transferを含む場合だけ、それらは`architecture_definition_migration_binding_ref`から別にexact解決する。

Active Product Definition変更は通常transitionを複数回適用してはならず、上記migration wrapper一件をcurrent snapshotへCASする。manifestはsource／destination全Definition rowのadded／removed／retainedをexactly once分類する。現行Migration V1は安全側のfull resetだけを許し、destination Activationは全required／optional bindingを`not_activated, candidate_ref=null, receipt_refs=[]`、Decision／Riskはdefinitionのgenesis evaluation、全通常WP headは`null, sequence=0`へ全面resetする。`wp.architecture.control-plane`だけは完成destination `ControlPlaneRebaselineApprovalV1`、完成destination baseline envelope、完成`ControlPlaneRebaselineTransactionV1`へ束縛し、`policy.product.wp.definition-migration-control-plane-rebaseline.v1`で新しいdestination epoch lifecycle wrapper（previous=null、sequence 1、`declared->complete`）を同じmigrationへ含める。初回Construction Authorization、`ControlPlaneBootstrapApprovalV1`または初回bootstrap policyを再利用しない。source lifecycle／Activation／Gate／Risk Evidence／Receiptはimmutable historyとして保持するがdestination currentへ一件もcarry-forwardせず、Control Plane artifactの再利用可否もrebaseline transaction内でdestination closureへ再検証する。destination snapshotの`previous_state_snapshot_ref`はsource signed wrapper、`sequence=source+1`、`applied_change.kind=active_definition_migration`とし、Definition hashと全state row/headを一つのatomic publicationで切替える。部分carry、複数migration wrapper、Approval前publishを拒否する。

Architecture Owner transferを含む場合は、同じatomic publicationでBindingが参照するArchitecture ChangeSetだけを`applied`へ進め、source Owner revisionをcurrentへ残さない。BindingなしのArchitecture current化、複数Binding、Product migrationと異なるsource／target definition hash、Consumer Inventoryがcomplete／zero verifiedでないCompatibility Change、Evidence Requirementのpass satisfaction binding不足を拒否する。Architecture Owner transferを含まないProduct-only migrationはBindingを持たず、Architecture ChangeSetを進めず、Architecture Consumer InventoryやOwner reference migration manifestを要求しない。

Migration wrapperは`service.product-state.definition-migrator`＋`role.product-state.definition-migrator`＋singleton purpose `active_product_definition_migration`で署名し、`authorization_decision_ref`はkind `active_definition_migration_architecture_product`、subject_kind `active_definition_migration`、同じapproval projectionを持つfresh approved `ProductOperationalDecisionV1`でなければならない。`migration_id`は同Fieldを除くpayloadのJCS SHA-256から`urn:mirakan:active-product-migration:sha256:<lowercase-hex>`として導出する。`signed_record.subject_sha256`はIDを含む完成payload JCS hash、current CAS refは完成signed wrapper JCS hashとし、同payload retryを再署名しない。

`architecture_definition_migration_binding_ref`を持つpayloadは`binding_state=approved`の一件だけを指す。CAS後のread-backで同Binding、Consumer Inventory、Compatibility Change、全Evidence Requirementのpass satisfaction binding、source／target Foundation Definition Closure、Owner reference migration manifestの全refがpayloadと一致し、対応Architecture ChangeSetを`applied`と評価できなければPublicationをrollbackする。Binding record自身の状態を`applied`へ書き換えず、Product-only migrationへBindingを補完しない。

Owner document bytes、Toolchain lock、Local Schema Catalog、Authority Binding Source Catalog、Control Plane artifactの変更がActive Product Definition bytesを変えない場合は、Definition migrationを捏造せず`ControlPlaneBaselineRebindingV1`だけを使う。source snapshotはCAS時のcurrent完成wrapper、payloadとsource／destination bindingの`active_product_definition_sha256`は同一current値、destination bindingは完成Rebaseline Approval／Envelope／Transactionから生成した`kind=rebaseline`でなければならない。Control Planeのcurrent lifecycle headだけを`policy.product.wp.control-plane-rebaseline.v1`の`complete->complete`、same Definition epoch、exact `N+1` Recordへ更新し、Activation全行、Decision／Risk全行、`wp.architecture.control-plane`以外の全WP head、definition hashをparentからbyte-exactに保持する。next snapshotはsource wrapperをprevious、sequence exact `N+1`、`applied_change.kind=control_plane_rebaseline`としてRebinding wrapperを指し、Rebinding wrapperとnext snapshotを一回のexpected-parent CASでpublishする。source drift、Definition hash差、他state変更、CP head未更新／複数更新、初回Bootstrap artifact再利用を拒否する。

Rebindingの`authorization_decision_ref`はkind `control_plane_baseline_rebinding_product`、subject kind `control_plane_baseline_rebinding`、current-parent definition bindingのfresh approved `ProductOperationalDecisionV1`だけを許す。`rebinding_id`は同Fieldを除くpayload JCS SHA-256から`urn:mirakan:control-plane-baseline-rebinding:sha256:<lowercase-hex>`として導出し、上記専用publisherで署名する。Control Plane Rebaseline ApprovalはArchitecture判断、Product Decisionはcurrent Product pointer切替判断、publisherは実行主体であり相互代用しない。

`ProductOperationalStatePolicyRegistryV1`の初期entryは次の7件である。各列はschema Fieldそのものであり、自由記述から値を推測しない。

| state_policy_id | change_kind | allowed_edges[] | authority_role_ref | signed_record_purpose | evidence_policy_refs[] | decision requirements by edge |
|---|---|---|---|---|---|---|
| `policy.product.activation.promotion.v1` | `activation` | `not_activated->candidate_locked; candidate_locked->qualified; qualified->production` | `role.product-state.transition-publisher` | `product_operational_state_transition` | `policy.evidence.contract-ci.v1; policy.evidence.target-device.v1; policy.evidence.release.v1` | 全edge=`activation_owner_approval` |
| `policy.product.activation.deactivation.v1` | `activation` | `production->qualified; qualified->candidate_locked; candidate_locked->not_activated` | `role.product-state.transition-publisher` | `product_operational_state_transition` | `policy.evidence.contract-ci.v1; policy.evidence.target-device.v1; policy.evidence.release.v1` | 全edge=`deactivation_r4` |
| `policy.product.decision-gate.evaluate.v1` | `decision_gate_evaluation` | `blocked->open; open->blocked; open->satisfied; satisfied->open; satisfied->retired` | `role.product-state.transition-publisher` | `product_operational_state_transition` | `policy.evidence.contract-ci.v1; policy.evidence.target-device.v1; policy.evidence.release.v1` | 全edge=`decision_gate_owner_approval` |
| `policy.product.risk.evaluate.v1` | `risk_evaluation` | `open->monitoring; monitoring->open; monitoring->mitigated; open->accepted; monitoring->accepted; mitigated->monitoring; mitigated->closed; accepted->monitoring; accepted->closed` | `role.product-state.transition-publisher` | `product_operational_state_transition` | `policy.evidence.contract-ci.v1; policy.evidence.target-device.v1; policy.evidence.release.v1` | `open->monitoring=risk_monitoring_owner_approval; monitoring->open=risk_monitoring_owner_approval; monitoring->mitigated=risk_mitigation_owner_approval; open->accepted=risk_acceptance_r4; monitoring->accepted=risk_acceptance_r4; mitigated->monitoring=risk_monitoring_owner_approval; mitigated->closed=risk_closure_owner_approval; accepted->monitoring=risk_acceptance_r4; accepted->closed=risk_acceptance_r4` |
| `policy.product.state.genesis.v1` | `genesis` | `genesis_only` | `role.product-state.snapshot-publisher` | `product_operational_state_snapshot` | `[]` | `genesis_only=construction_decision_required` |
| `policy.product.state.control-plane-rebaseline.v1` | `control_plane_rebaseline` | `same_definition_current_binding_to_approved_rebaseline` | `role.product-state.control-plane-rebinder` | `control_plane_baseline_rebinding` | `[]` | `same_definition_current_binding_to_approved_rebaseline=control_plane_baseline_rebinding_product` |
| `policy.product.state.definition-migration.v1` | `active_definition_migration` | `definition_revision_to_approved_revision` | `role.product-state.definition-migrator` | `active_product_definition_migration` | `[]` | `definition_revision_to_approved_revision=active_definition_migration_architecture_product` |

`decision_requirements[]`は`allowed_edges[]`とedge set equalityで、`requirement`は上表のDecision kindまたは`construction_decision_required`だけを受理する。Promotionは`CapabilityTargetActivationBindingRegistryV1`のsame Target／Candidate、Owner WP、exact Gate／Requirement、stateに応じたEvidence policy、fresh Target closure、no revoked evidenceを必須にし、同Registryでdisabledなtransitionを拒否する。Deactivationはsecurity／regression／license／provider／Target／freshness原因、impact、rollback／migrationをDecisionへ閉じ、自動fallbackを禁止する。Decision Gate `satisfied`はtyped definition predicateと全prerequisiteをfresh approved Evidenceで再評価しactionを自動実行しない。Risk `monitoring`は観測計画、`accepted`はR4 Risk Decision、`mitigated`はfresh mitigation Receipt、`closed`はtrigger消滅のfresh Evidenceを必須にする。GenesisはControl Plane §6.1 A／B／C／D、final Bootstrap、construction Decision、全row set equalityを一度だけ検査する。Same-definition Control Plane rebaselineは完成Rebaseline closure、R4 Product Decision、CP lifecycleの一行更新、他state byte equality、single CASを必須にする。Definition migrationはapproved source／destination、destination Rebaseline Approval／Envelope／Transaction、R4 Product Change Decision、全row manifest、全面state reset、Control Plane completion、single CASを必須にする。Architecture Owner transferを含むbranchだけは、さらにapproved Architecture Definition Migration binding、complete Consumer Inventory、approved Compatibility Change、全Evidence Requirementのpass satisfaction binding、Owner reference migration manifestを必須にする。

段階飛越、Target／Candidate差、predicate未成立、unknown state、stale／revoked Evidence、Owner／Role不一致、自由なpolicy ID、Snapshot rowの直接差替えを拒否する。genesisだけはfinal Bootstrap Approvalとoffline governance construction Decisionに閉じた専用`policy.product.state.genesis.v1`で全row set equalityを一度に生成し、通常更新に流用しない。

Activation rowの更新は、`not_activated->candidate_locked`だけが`candidate_ref=null`からnon-null exact Candidateを設定でき、current row `receipt_refs[]`をCandidate lockの現在有効なqualifying closureとして開始する。`candidate_locked->qualified->production`は同じCandidate refをbyte-exactに維持するが、current rowのReceiptは「現在stateを支えるfresh exact closure」へatomic replaceする。旧Receiptを永久unionせず、再Qualificationではexpired／revoked refを新fresh Receiptへ置換できる。transition payloadの`prior_receipt_refs[]`はparent rowと、`next_receipt_refs[]`はnext rowとexact一致し、immutable transition chainが旧新両集合を監査する。隣接降格はlower stateを支えるcurrent closureへ再計算し、`candidate_locked->not_activated`で`candidate_ref=null, receipt_refs=[]`へclearする。別Candidateへの切替は旧Candidateを隣接降格で`not_activated`へ閉じた後、新しいlock transitionとして開始する。`effective_availability`はcurrent rowの`receipt_refs[]`だけを評価し、historical transition Receiptをcurrent合否へ混ぜない。

Risk／Decision Gateの表は可読性のためdefinition列とgenesis評価列を併記するが、generatorは`state`／`evidence refs`をdefinition projectionへ含めずgenesis operational snapshotへ分離する。state／evidence更新は署名済みtransitionと新snapshotで行い、Product DefinitionまたはBootstrap Approvalを更新しない。Definition自体のmembership、predicate、action、Target binding、Phase／WP関係を変える場合だけ新definition revision、二段階Approval、state migrationを必要とする。

Riskの保存stateもread-timeで再評価する。`mitigated`／`closed`は対応するfresh non-revoked mitigation／trigger消滅Evidence、`accepted`はfresh non-revoked kind `risk_acceptance_r4` Decisionがcurrent definition／subjectへ一致する場合だけeffectiveに維持する。根拠がexpired／revoked／input driftなら、current fresh monitoring plan Evidenceがあれば`effective_state=monitoring`、なければ`effective_state=open`とする。保存snapshotを黙って書き換えないが、Release、Activation、WP遷移、Decision Gate、claim releaseは必ずeffective stateを消費し、保存上の`mitigated | accepted | closed`を根拠失効後に許可へ使わない。Risk Ownerはその後、対応する署名済みtransitionで保存stateを追随させる。

Work Packageのcurrent stateは次のsigned append-only chainだけが所有する。

```text
WorkPackageLifecyclePayloadV1
  lifecycle_record_id
  previous_lifecycle_record_ref: null | content-addressed ref
  lifecycle_sequence: integer 1..9007199254740991
  work_package_id
  from_scheduling_state
  to_scheduling_state
  active_product_definition_sha256
  parent_operational_state_snapshot_sha256
  candidate_binding_kind = task_plan | artifact_candidate | baseline_candidate | control_plane_rebaseline_candidate | definition_seed
  candidate_binding_ref
  candidate_binding_hash
  transition_policy_ref
  receipt_refs[]
  blocked_reason_ref: null | exact Diagnostic ref
  authorization_record_ref: approved authorization record ref
  requested_by_subject_ref
  recorded_at
  revocation_snapshot_ref

WorkPackageLifecycleRecordV1
  payload: WorkPackageLifecyclePayloadV1
  signed_record: MirakanSignedRecordV1(purpose=work_package_lifecycle_transition)

ArtifactCandidateBindingV1
  artifact_candidate_id
  origin_task_plan_ref
  origin_task_plan_sha256
  source_revision_ref
  candidate_root_sha256
  contract_set_sha256
  toolchain_lock_sha256
  target_profile_refs[]
  artifact_refs[]

WorkPackageDefinitionSeedBindingV1
  definition_seed_binding_id
  active_product_definition_sha256
  work_package_registry_ref, work_package_registry_sha256
  work_package_id
  work_package_row_sha256

ControlPlaneRebaselineCandidateBindingV1
  candidate_binding_id
  active_product_definition_sha256
  rebaseline_core_ref, rebaseline_core_sha256
  rebaseline_approval_ref, rebaseline_approval_sha256
  rebaseline_envelope_ref, rebaseline_envelope_sha256
  rebaseline_transaction_ref, rebaseline_transaction_sha256
```

`lifecycle_record_id`は同Fieldを除く`WorkPackageLifecyclePayloadV1`のRFC 8785 JCS SHA-256から`urn:mirakan:wp-lifecycle:sha256:<lowercase-hex>`として導出する。`signed_record.subject_sha256`はIDを含む完成payloadのJCS hashであり、wrapper／signatureをIDへ含めない。Signer Role、purpose、`signed_record.issued_at=payload.recorded_at`、revocation snapshotは上記Product state署名規則と一致させる。Lifecycle chain identityは`{active_product_definition_sha256, work_package_id}`のepochである。`lifecycle_sequence`とsnapshot `sequence`は上記safe-integer contractを使い、各Definition epochの最初のLifecycle Recordは1、以後exact `N+1`としoverflowを拒否する。Active Definition migrationではdestination epochを新規開始し、通常WPはnull head／0、Control Planeだけprevious=null／sequence 1のbootstrap Recordとする。別Definition epoch間でprevious refを接続せず、同じsequence値を使ってもdefinition hashが異なるため別chainである。最初のRecordは`previous_lifecycle_record_ref=null`、`from_scheduling_state`をDefinition seedと一致させる。headがnullならcurrent WP stateはDefinition seed、headがnon-nullならverified head payloadの`to_scheduling_state`である。以後はcurrent signed lifecycle wrapper ref、sequence `N`、`to_scheduling_state`をそれぞれprevious ref、`N+1`、次Recordのfromへbyte-exactに引き継ぐ。`previous_lifecycle_record_ref`とsnapshotのlifecycle headは完成signed wrapperのcontent ref、`parent_operational_state_snapshot_sha256`はCAS対象current signed snapshot wrapperのJCS hashであり、payload ID／hashではない。new snapshotの対応headはnew signed lifecycle wrapper ref／sequenceでなければならない。Recordとnext snapshotは同一atomic transactionでpublishする。同じparent／headからの二分岐、sequence gap／duplicate、ID／payload不一致、同じWPに複数current headを拒否する。同一payload retryは再署名せず既存wrapperを返す。`to_scheduling_state=blocked`だけ`blocked_reason_ref`をnon-null registered Diagnosticにし、他stateは`null`とする。

`candidate_binding_kind=task_plan`は`declared -> ready`、`ready -> active`、通常の`ready | active -> blocked`／`blocked -> ready`でcurrent `CriticalPathTaskV1` ref／hashを束縛する。同じWPの`ready->active`、`ready|active->blocked`、`blocked->ready`は直前headと同じtask plan ref／hashをbyte-exactに維持し、別Taskへ切り替える場合は登録済みcancel／supersede policyを追加したActive Definition revisionまで拒否する。`artifact_candidate`は`active -> complete`だけで上記closed `ArtifactCandidateBindingV1`の完成bytesをref／hash束縛し、`origin_task_plan_ref`／hashが直前active headのtask binding、Source revision、Candidate root、Contract set、Toolchain、Target ProfileがTaskとexact一致しなければならない。`artifact_candidate_id`は同Fieldを除くpayloadのJCS SHA-256から`urn:mirakan:artifact-candidate:sha256:<lowercase-hex>`として導出する。`baseline_candidate`はControl Plane専用bootstrap完了Recordだけ、`control_plane_rebaseline_candidate`は同じDefinition内でのControl Plane rebaselineだけに完成`ControlPlaneRebaselineCandidateBindingV1`を束縛する。`definition_seed`は初回`deferred->declared`で完成`WorkPackageDefinitionSeedBindingV1`をref／hash束縛する。同artifactのRegistry ref／hashはcurrent Active Definition closure内のexact `WorkPackageRegistryV1`、WP IDは遷移対象、`work_package_row_sha256=SHA-256(JCS(exact inline row))`でなければならない。`definition_seed_binding_id`は同Fieldを除くJCS hashから`urn:mirakan:work-package-definition-seed:sha256:<lowercase-hex>`、rebaseline candidate IDも同様に`urn:mirakan:control-plane-rebaseline-candidate:sha256:<lowercase-hex>`から導出する。inline rowへの架空content ref、空ref、自由文字列、別Candidateへの置換、origin Taskを持たないartifactを許さない。Active Definition変更時のresetはLifecycle edgeでなく`ActiveProductDefinitionMigrationV1`がatomicに所有する。

`WorkPackageLifecyclePolicyRegistryV1`の初期exact entryは次の5件である。Field値は表から直接生成し、説明文から補完しない。

| transition_policy_id | allowed_edges[] | subject_binding_policy | prerequisite_policy | receipt_policy | authorization requirements by edge | bootstrap_only_work_package_ref |
|---|---|---|---|---|---|---|
| `policy.product.wp.normal.v1` | `declared->ready; ready->active; ready->blocked; active->blocked; blocked->ready; active->complete` | `task_or_artifact` | `normal` | `normal` | `declared->ready=ProductOperationalDecisionV1/work_package_owner_transition; ready->active=ProductOperationalDecisionV1/work_package_owner_transition; ready->blocked=ProductOperationalDecisionV1/work_package_owner_transition; active->blocked=ProductOperationalDecisionV1/work_package_owner_transition; blocked->ready=ProductOperationalDecisionV1/work_package_owner_transition; active->complete=ProductOperationalDecisionV1/work_package_owner_acceptance` | `null` |
| `policy.product.wp.defer-release.v1` | `deferred->declared` | `definition_seed` | `decision_gates` | `no_carry` | `deferred->declared=ProductOperationalDecisionV1/work_package_defer_release_product` | `null` |
| `policy.product.wp.bootstrap-control-plane.v1` | `declared->complete` | `baseline` | `bootstrap` | `bootstrap` | `declared->complete=ControlPlaneConstructionAuthorizationV1/control_plane_construction_authorization` | `wp.architecture.control-plane` |
| `policy.product.wp.control-plane-rebaseline.v1` | `complete->complete` | `baseline` | `same_definition_rebaseline` | `rebaseline` | `complete->complete=ControlPlaneRebaselineApprovalV1/control_plane_rebaseline_approval` | `wp.architecture.control-plane` |
| `policy.product.wp.definition-migration-control-plane-rebaseline.v1` | `declared->complete` | `baseline` | `definition_migration_rebaseline` | `rebaseline` | `declared->complete=ControlPlaneRebaselineApprovalV1/control_plane_rebaseline_approval` | `wp.architecture.control-plane` |

`subject_binding_policy`は`task_or_artifact | definition_seed | baseline`、`prerequisite_policy`は`normal | decision_gates | bootstrap | same_definition_rebaseline | definition_migration_rebaseline`、`receipt_policy`は`normal | no_carry | bootstrap | rebaseline`のclosed enumであり、logical refではない。`authorization_requirements[]`はallowed edgeとset equalityで、各edgeの`authorization_schema`／`authorization_kind`を上表から直接生成する。通常／defer行の`authorization_record_ref`は同じapproval projectionを承認する`ProductOperationalDecisionV1`、初回bootstrap行だけはTask 0～10Bと初回Control Plane completionをscope許可した`ControlPlaneConstructionAuthorizationV1`、二つのrebaseline行だけはsource currentとCore closureを承認した`ControlPlaneRebaselineApprovalV1`である。初回Authorizationは個別lifecycle内容を承認するDecisionではなく、final Bootstrap Approval、baseline candidate、construction Receiptとのclosureでだけ有効なoffline scope authorizationである。`normal` readyは全prerequisite current head=`complete`、approved Owner、current Control Plane approval、qualified task plan、completeはfresh Owner acceptance、Target closure、artifact candidateを必須にする。`defer-release`は全reconsideration gateのread-time effective evaluation=`satisfied`、完成`WorkPackageDefinitionSeedBindingV1`、`receipt_refs=[]`、approved Product Decisionを必須にする。初回`bootstrap`はfinal baseline envelope、Bootstrap Approval、`task-0..task-10a`のpass Construction Receipt exact set、offline `ControlPlaneConstructionAuthorizationV1`へ閉じる。Task 10Bはgenesis／lifecycle／current CASを生成する当事Taskなので、そのReceiptを入力条件へ含めない。atomic publication後にTask 10B pass Receiptを監査記録として発行し、失敗Receiptでも既存currentを巻き戻さずquarantine手順へ進む。`same_definition_rebaseline`はcurrent Control Plane headが`complete`、source／destination Definition hashが同一、完成Rebaseline Core／Approval／Envelope／Transactionと`ControlPlaneRebaselineCandidateBindingV1`が一致する場合だけ`complete->complete`をexact `N+1`で許可する。`definition_migration_rebaseline`は同じ完成artifactをdestination closureへ閉じ、新Definition epochのsequence 1として初回Authorization／Bootstrap Approvalを参照せずsource epoch Receiptをcarryしない。上表外のedge／enum、未登録policy、wrong authorization schema／kind／subject、authorization欠落、completeでtask-plan hashのままを拒否する。`ready`／`complete`等のcurrent値をDefinition表へ書き戻さない。

`scheduling_state`のclosed値は`declared | ready | active | blocked | deferred | complete`である。Definition内の同Fieldはgenesis seedであり、approved definition revision内ではimmutableである。現在stateはcurrent `work_package_lifecycle_heads[]`から導出し、Definition行へwrite-backしない。`defer_reason`と`blocked_reason_ref`は常にFieldを持ち、非該当時は`null`とする。`reconsideration_gate_refs[]`は常に存在し、非deferred時は空配列である。genesis `scheduling_state=deferred`ではnon-empty `defer_reason`と1件以上の`reconsideration_gate_refs[]`、`blocked`ではnon-null `blocked_reason_ref`を必須とする。他stateでこれらを設定した行を拒否する。`reconsideration_gate_refs[]`は`PhaseFixtureBindingRegistryV1.gate_id`または`ProductDecisionGateRegistryV1.gate_id`のexact IDだけを受理する。

`ProductPhaseRegistryV1`の`work_package_refs[]`は、`phase_id`が当該Phaseに一致する全Work Packageの全量列挙である。Phase→Work PackageとWork Package→Phaseの相互参照は双方向で一致させ、片側にのみ現れる参照を拒否する。`WorkPackageRegistryV1`の`provided_fixture_refs[]`は当該Work Packageが実装へ寄与するtyped `ProvidedFixtureRefV1`の列挙であり、Work Package単独の完了gateではない。`kind=product_fixture`だけがPhase／Release evidence候補になり、`kind=component_qualification_fixture`はowner manifestへversion／hash付きで閉じたWP内部Qualification入力である。RequirementとPhase completionをWork Packageへ複写せず、完了Receiptはappend-onlyな`WorkPackageLifecycleRecordV1`だけが所有する。

`PhaseFixtureBindingRegistryV1`の`evaluated_requirement_refs[]`と`target_refs[]`は参照先Fixtureの各集合のsubsetでなければならず、範囲外参照を拒否する。`candidate_binding_policy_ref=policy.product.same-candidate.v1`はProject revision、Candidate root hash、Contract set hash、Toolchain lock、Target Profileを全Receiptで一致させる。`freshness_policy_ref`はEvidence Ownerの`policy.evidence.contract-ci.v1`または`policy.evidence.target-device.v1`をexact参照し、失効、revoked、入力hash不一致のReceiptをPhase exitへ使用しない。物理Device上のplay、cook／package、install／launch、performanceまたはSource artifact使用を含むGateは`policy.evidence.target-device.v1`だけを使う。current OS／driver／device／firmware identityまたはpackage artifact hash／signature／install stateがReceipt発行時からdriftした場合は即時`expired`とし、同じTargetで再実行する。`policy.evidence.contract-ci.v1`のReceiptや別TargetのReceiptを代用しない。

Activationの唯一の保存状態正本はcurrent `ProductOperationalStateSnapshotV1.capability_target_activation_rows[]`の`CapabilityTargetActivationStateV1`である。`CapabilityRegistryV1`の各`target_bindings[]`について、scopeが`required`または`optional`の`{capability_id, target_id}`行をgenesisで生成し、`state=not_activated`、`candidate_ref=null`、`receipt_refs=[]`とする。scopeが`excluded`のTargetにはActivation行を生成せず、選択を拒否する。freshnessをrowへ保存せずEvidence Ownerのcurrent receipt／revocation／expiry／input hashから都度導出する。Capability全体の表示状態はrequired Target行の最小stateから毎回導出するread-only projectionであり、Definition、C2 matrix、Capability表、Product labelへ保存またはwrite-backしない。

`capability.platform.input-core`、`capability.platform.audio-core`、`capability.platform.ui-core`のTarget Activationはportable contractのOwner Receiptだけでは昇格しない。Windows Editor／Desktop行は対応する`wp.platform.input-windows`、`wp.platform.audio-windows`、`wp.platform.ui-windows`の`fixture.product.windows-empty-scene` fresh Receiptを必須とする。Android行は`wp.platform.mobile-io-ui-android`、Apple行は`wp.platform.mobile-io-ui-apple`が同Targetで発行する`fixture.product.mobile-lifecycle`、`fixture.product.shooter-2d`、`fixture.product.shooter-arena-3d`のPhase 7 fresh Receiptを必須とする。Phase 8の`fixture.product.platformer-2d`または`fixture.product.puzzle-dialogue-2d`をPhase 7 Activationの前提にせず、Windows Receiptまたは反対側Mobile TargetのReceiptによる代用を拒否する。

`capability.domain.shooter-2d`と`capability.domain.shooter-3d`はWindows／Android／Appleに別々の`CapabilityTargetActivationStateV1`行を持つ。Phase 3の`gate.product.phase-3-manual-2d`はShooter 2DのWindows行だけ、Phase 6の`gate.product.phase-6-first-playable-3d`はShooter 3DのWindows行だけを昇格候補にする。Phase 7の四つの`*-runtime-2d`／`*-runtime-3d` Gateは対応dimensionとAndroidまたはAppleの一行だけを昇格候補にし、Windows、反対側Mobile Target、別dimensionのReceiptを流用しない。Phase 8 genre GateはC2 coverageを評価するだけでC1 Shooter行を遡及昇格しない。

`decision_gate_evaluations[].state`のclosed値は`blocked | open | satisfied | retired`である。Decision GateはPhase exit Receiptではなく、未登録logical IDを作らずにplanning-only条件を追跡する。`state=satisfied`でも`on_satisfied_action`のChangeSetを自動適用せず、Owner approvalと全Definition validationを別途必須とする。評価state／evidenceはcurrent operational snapshotだけが所有し、`ProductDecisionGateRegistryV1`へ保存しない。

### 11.2 Target Profile registry

| target_id | Owner | profile_version | target_kind | surface_role | qualification_requirement_ref |
|---|---|---:|---|---|---|
| `target.headless.host` | `mirakan.arch.core-architecture` | 1 | `headless_server` | `both` | `requirement.target.headless-determinism` |
| `target.windows.editor` | `mirakan.arch.platform-windows` | 1 | `desktop` | `execution_host` | `requirement.target.windows-editor` |
| `target.windows.desktop` | `mirakan.arch.platform-windows` | 1 | `desktop` | `artifact_runtime` | `requirement.target.windows-package` |
| `target.android.mobile` | `mirakan.arch.platform-android` | 1 | `mobile` | `artifact_runtime` | `requirement.target.android-package` |
| `target.apple.mobile` | `mirakan.arch.platform-apple` | 1 | `mobile` | `artifact_runtime` | `requirement.target.apple-package` |

Profile versionは更新可能なFieldであり、Target IDへ`v1`を埋め込まない。`target_kind`と`surface_role`はID文字列から推測せずRegistry rowのclosed Fieldだけを使う。AI Conformanceのexecution／artifact Target、Product `ExecutionSurfaceBindingV1`はcurrent rowを解決し、execution refは`execution_host | both`、artifact refは`artifact_runtime | both`だけを許す。将来kindまたはroleを変える場合はTarget Profile Registry revisionと全consumer Evidenceを更新し、同ID／versionの意味をin-place変更しない。

Future Dossierはcurrent rowを参照する`current_profile`と、authority-free `proposed_destination_profile`をtagged unionで分離する。後者はFuture Inceptionだけが所有する未登録Target候補であり、本Registryの件数、Target support、Conformance、Build、PackageまたはShippingを変更しない。Future→Active migration時は全proposed候補をdestination Registry rowへ一対一投影し、row migration manifestへ列挙してから全Activationを`not_activated`で開始する。proposal refをcurrent `TargetProfileRefV1`として受理する、現行5 rowへ暗黙追加する、Target kindだけ一致する別Profileへ置換する実装を拒否する。

### 11.3 Requirement、Fixture、Fallback registry

| requirement_id | Owner | verification_kind | failure diagnostic |
|---|---|---|---|
| `requirement.target.headless-determinism` | `mirakan.arch.core-architecture` | `determinism` | `diagnostic.product.headless-determinism-failed` |
| `requirement.foundation.memory-pointer-contract` | `mirakan.arch.memory-pointers` | `pointer_memory_contract_qualification` | `diagnostic.product.memory-pointer-contract-failed` |
| `requirement.target.windows-editor` | `mirakan.arch.platform-windows` | `package_and_launch` | `diagnostic.product.windows-editor-gate-failed` |
| `requirement.target.windows-package` | `mirakan.arch.platform-windows` | `clean_install_offline_run` | `diagnostic.product.windows-package-gate-failed` |
| `requirement.target.android-package` | `mirakan.arch.platform-android` | `store_package_device_run` | `diagnostic.product.android-package-gate-failed` |
| `requirement.target.apple-package` | `mirakan.arch.platform-apple` | `store_package_device_run` | `diagnostic.product.apple-package-gate-failed` |
| `requirement.target.mobile-runtime-2d` | `mirakan.arch.product-plan` | `target_device_playable_e2e` | `diagnostic.product.mobile-runtime-2d-gate-failed` |
| `requirement.target.mobile-runtime-3d` | `mirakan.arch.product-plan` | `target_device_playable_e2e` | `diagnostic.product.mobile-runtime-3d-gate-failed` |
| `requirement.product.authoring-roundtrip-manual` | `mirakan.arch.product-plan` | `manual_e2e` | `diagnostic.product.authoring-roundtrip-manual-failed` |
| `requirement.product.editor-reference-design` | `mirakan.arch.product-plan` | `visual_semantic_accessibility_matrix` | `diagnostic.product.editor-reference-design-incomplete` |
| `requirement.product.authoring-roundtrip` | `mirakan.arch.product-plan` | `ai_e2e` | `diagnostic.product.authoring-roundtrip-failed` |
| `requirement.product.core-pack-independence` | `mirakan.arch.product-plan` | `dependency_and_holdout_matrix` | `diagnostic.product.core-pack-independence-failed` |
| `requirement.product.manual-first-playable-2d` | `mirakan.arch.product-plan` | `manual_playable_e2e` | `diagnostic.product.manual-first-playable-2d-incomplete` |
| `requirement.product.ai-authoring-mvp-a` | `mirakan.arch.product-plan` | `ai_mvp_e2e` | `diagnostic.product.ai-authoring-mvp-a-incomplete` |
| `requirement.product.ai-genre-neutral-authoring` | `mirakan.arch.product-plan` | `ai_holdout_e2e` | `diagnostic.product.ai-genre-neutral-authoring-incomplete` |
| `requirement.product.first-playable-3d` | `mirakan.arch.product-plan` | `playable_e2e` | `diagnostic.product.first-playable-3d-incomplete` |
| `requirement.product.mvp-completion` | `mirakan.arch.product-plan` | `mvp_completion_e2e` | `diagnostic.product.mvp-completion-incomplete` |
| `requirement.product.project-source-activation` | `mirakan.arch.product-plan` | `source_activation_e2e` | `diagnostic.product.project-source-activation-incomplete` |
| `requirement.product.title-to-result` | `mirakan.arch.product-plan` | `playable_e2e` | `diagnostic.product.title-to-result-failed` |
| `requirement.product.save-load-replay` | `mirakan.arch.product-plan` | `state_roundtrip` | `diagnostic.product.save-load-replay-failed` |
| `requirement.product.c2-2d-coverage` | `mirakan.arch.product-plan` | `cross_genre_matrix` | `diagnostic.product.c2-2d-coverage-incomplete` |
| `requirement.product.c2-3d-coverage` | `mirakan.arch.product-plan` | `cross_genre_matrix` | `diagnostic.product.c2-3d-coverage-incomplete` |
| `requirement.product.cpp23-cx2-cross-target` | `mirakan.arch.cpp23-modules` | `cross_target_module_cutover` | `diagnostic.product.cpp23-cx2-cross-target-incomplete` |
| `requirement.product.cpp23-cx3-cross-target` | `mirakan.arch.cpp23-modules` | `cross_target_shipping_language_mode` | `diagnostic.product.cpp23-cx3-cross-target-incomplete` |
| `requirement.product.external-agent-boundary` | `mirakan.arch.ai-security-approval` | `authorization_conformance` | `diagnostic.product.external-agent-boundary-failed` |
| `requirement.product.runtime-generation-boundary` | `mirakan.arch.ai-security-approval` | `threat_model_conformance` | `diagnostic.product.runtime-generation-boundary-failed` |
| `requirement.product.release-headless` | `mirakan.arch.product-plan` | `release_contract_support_rollback` | `diagnostic.product.release-headless-incomplete` |
| `requirement.product.release-windows-editor` | `mirakan.arch.product-plan` | `clean_install_launch_support_rollback_release` | `diagnostic.product.release-windows-editor-incomplete` |
| `requirement.product.release-windows-desktop` | `mirakan.arch.product-plan` | `clean_install_offline_support_rollback_release` | `diagnostic.product.release-windows-desktop-incomplete` |
| `requirement.product.release-android` | `mirakan.arch.product-plan` | `signed_package_device_offline_support_rollback_release` | `diagnostic.product.release-android-incomplete` |
| `requirement.product.release-apple` | `mirakan.arch.product-plan` | `signed_package_device_offline_support_rollback_release` | `diagnostic.product.release-apple-incomplete` |

`requirement.foundation.memory-pointer-contract`は、[Memory／Pointers](../02-foundation/memory-pointers.md)の`PointerContractV1`、`MemoryContractV1`、`PointerMemoryConsumerBindingV1`のdefinition closure、consumer Matrixの正逆参照、live pointer／lease／span／allocator objectの保存・job capture禁止、supported sanitizer lane、hot path fallback 0のbaselineを同一Candidateで検査する。Targetがsanitizerを実行できない場合は、対応するstatic／negative Evidenceとsupported Target laneを別Evidenceとして記録し、未実行laneをpassへ読み替えない。これはPhase 0の計画上のGateであり、MCD、生成物、実装、Activationが既に存在するという主張ではない。

`requirement.product.editor-reference-design`は§5.2の七surface roleと二つの横断Referenceを、排他的なPanel分類や機能Activationにせず、typed style／semantic／Workspace definitionとして閉じる。全interactive Controlとsemantic composite rootはclosed Widget Pattern Registryへ解決し、Patternはvisual、Semantic、UIA、Command投影を同じinstanceへ束縛するが、AI操作権限またはProject writeを所有しない。`fixture.product.windows-empty-scene`では実装済みsurfaceについてLight／Dark／High Contrast、`editor_ui_scale=2.00`、density、state overlay、keyboard／UIA／AI typed commandのmatrixを同一Stable ID／revisionで検証し、未Activation Panelを成功例へ補完しない。同Product fixtureは[Editor UI Framework §15.8](../03-authoring/editor-ui-framework.md#158-reference-fixture-manifestevidence-contract)の`EditorReferenceFixtureRegistryV1`をcurrent Capability snapshotへ解決したrequired Manifest集合と、各Manifestのpass `VerificationReceiptV1`／Evidence Bundleをexact set equalityで含む。最初のcomponent Manifest `fixture.editor.reference-01@1`は[Editor Workspace UX §6.9.1](../03-authoring/editor-workspace-ux.md#691-fixtureeditorreference-011-concrete-manifest)の14 Environment Profile、9 scenario、166 coverage entry、七typed expected subjectとbyte equalityでなければならず、同ManifestのComparison Profile Registryは[Editor UI Framework §15.8.5](../03-authoring/editor-ui-framework.md#1585-comparison-profile-registry)のexact七entryとbyte equalityでなければならない。同ManifestのBaseline RegistryはCoverage Matrixとexact 166 entryで一致し、各expected subjectはatomic Change Item、domain owner＋independent reviewer二`ReviewReceiptV1`、candidate全件検証、source-head CAS、`PromotionReceiptV1` read-backを通ったcurrent Publicationだけから解決する。未materialized logical ID、Markdown表、characterization Profile、Runner既定値、pending decision、AI説明をReceiptへ読み替えない。component Manifestを`FixtureRegistryV1`の新しいProduct fixture rowへ偽装せず、prohibited Manifest、未Activation surfaceのmissing Evidence、別Candidate／environment／contract setのReceiptをpassへ補完しない。Font／icon lock、combined allocation model、TSF／OLE／UIA virtualization、Widget Pattern、AI semanticとVisual evidence、Source／Text／Diff Reference Contract、Task／Proposal／Review Reference Contract、Cross-Panel、Reference Fixture Manifest、Comparison Profile Registry、Baseline Review／Publication Contractの各blockerは§5.2のOwner closureがなければ失敗とし、screenshotだけ、色だけ、Panel名だけ、表示rowだけをEvidenceにしない。`requirement.product.manual-first-playable-2d`は手動AuthoringだけでTitle→Result、Save／Load／Replay、Windows package／clean install／offline playを完走する。`requirement.product.core-pack-independence`はShooterを含む全Genre Packと全optional Feature Packをuninstallした状態でCore／Editor／AI query／Project SDK／Build／Packageが成立し、Core→Feature／Genre required edgeが0件、Genreなし／zero-character／noncombat／UI-only／headlessのholdoutをschema変更なしで表現できることをdependency graphと実行fixtureの両方で検証する。`gate.product.phase-3-genreless-core`は同じCandidate、Active Product Definition、Foundation Closure、Generic Core baseline、Installed Product compositionを束縛し、`result.kind=pass`のfresh `ArchitectureDependencyConformanceReportV1`と`fixture.product.genreless-core-2d`実行Receiptを両方持つまでpassにしない。Report Refだけ、別Candidate、stale Registry、count-only自己申告をEvidenceへ補完しない。`requirement.product.ai-authoring-mvp-a`はDefinition-firstまたは事前Qualification済みNative／Shader Packを使い、AIと手動編集の往復、typed ChangeSet、Diff、Approval、Commit、Undo、Replay、§5のMVP completion chainを一つのProject historyと同一Candidateで完走する。sub-requirementとして`requirement.product.authoring-roundtrip`と`requirement.product.mvp-completion`をEvidenceへ記録するが、`requirement.product.project-source-activation`をPhase 4 gateへ含めない。`requirement.product.ai-genre-neutral-authoring`はGenre／Character／Combat／Objective／Worldを全てoptionalにしたzero-character UI／logic Projectを、同じAI Operation／Validation／Debug／Build DAGで生成、質問解決、Commit、Replay、Packageできることを要求し、Shooter fixtureの成功を代用しない。

`requirement.product.mvp-completion`は§5の一方向chain全体、すなわちclean environment相当でのcook、package、install／launch、first-run settings、offline play、checkpoint／resume、diagnosis、`SupportBundleV1`取得、data resetを同一Candidate hashで検証する。`requirement.product.project-source-activation`は`capability.project.native_module`と`capability.project.shader`の両方を実際のFirst Playable挙動から使用し、Source、生成Diff、Approval、Target artifact、Qualification Receiptが同一Project revisionへ閉じる場合だけ成功する。`requirement.target.mobile-runtime-2d`と`requirement.target.mobile-runtime-3d`はPhase 7のC1実機Requirementであり、同一CandidateのShooter fixtureをAndroidまたはAppleのpackageからclean install／offline起動し、Target別I/O／UI、E7 integration、Save／Load／Replay、Title→Resultまでを実機で完走する。C2 genre coverageや別TargetのReceiptを前提または代用にしない。

五つの`requirement.product.release-*`は単なるbuild成功ではなく、同一Candidate／Toolchain／Target Profileでのclean build、署名済みpackageまたはheadless distribution、clean install／launch、offline run（適用Target）、Support Bundle、rollback rehearsal、license／SBOM／provenance、Release Approvalを閉じる。Headlessは物理Deviceを要求しないが、exact build host profileとdistributable artifactのreproducibilityを要求する。Windows Editor／Desktop、Android、AppleのReceiptは相互流用せず、`policy.evidence.release.v1`でcurrent signer／package／device／support windowを再評価する。

| fixture_id | Owner | requirement_refs | targets | minimum duration seconds |
|---|---|---|---|---:|
| `fixture.product.headless-contract-smoke` | `mirakan.arch.product-plan` | `requirement.target.headless-determinism` | `target.headless.host` | 60 |
| `fixture.foundation.memory-pointer-contract` | `mirakan.arch.memory-pointers` | `requirement.foundation.memory-pointer-contract` | `target.headless.host` | 300 |
| `fixture.product.authoring-transaction` | `mirakan.arch.project-state` | `requirement.product.authoring-roundtrip-manual` | `target.headless.host` | 300 |
| `fixture.product.windows-empty-scene` | `mirakan.arch.product-plan` | `requirement.target.windows-editor; requirement.target.windows-package; requirement.foundation.memory-pointer-contract; requirement.product.editor-reference-design` | `target.windows.editor; target.windows.desktop` | 300 |
| `fixture.product.genreless-core-2d` | `mirakan.arch.product-plan` | `requirement.product.core-pack-independence` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | 300 |
| `fixture.product.genreless-ai-project` | `mirakan.arch.product-plan` | `requirement.product.core-pack-independence; requirement.product.ai-authoring-mvp-a; requirement.product.ai-genre-neutral-authoring; requirement.product.authoring-roundtrip; requirement.product.mvp-completion; requirement.product.project-source-activation; requirement.product.save-load-replay` | `target.windows.editor; target.windows.desktop` | 300 |
| `fixture.product.shooter-2d` | `mirakan.arch.pack-shooter` | `requirement.target.mobile-runtime-2d; requirement.product.manual-first-playable-2d; requirement.product.ai-authoring-mvp-a; requirement.product.authoring-roundtrip; requirement.product.title-to-result; requirement.product.save-load-replay; requirement.product.mvp-completion; requirement.product.project-source-activation; requirement.product.c2-2d-coverage` | `target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | 300 |
| `fixture.product.platformer-2d` | `mirakan.arch.product-plan` | `requirement.product.c2-2d-coverage; requirement.product.save-load-replay` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | 300 |
| `fixture.product.puzzle-dialogue-2d` | `mirakan.arch.product-plan` | `requirement.product.c2-2d-coverage; requirement.product.save-load-replay` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | 300 |
| `fixture.product.shooter-arena-3d` | `mirakan.arch.pack-shooter` | `requirement.target.mobile-runtime-3d; requirement.product.first-playable-3d; requirement.product.title-to-result; requirement.product.save-load-replay; requirement.product.c2-3d-coverage` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | 300 |
| `fixture.product.external-agent-proposal` | `mirakan.arch.ai-security-approval` | `requirement.product.external-agent-boundary` | `target.headless.host` | 120 |
| `fixture.product.mobile-lifecycle` | `mirakan.arch.platform-mobile-common` | `requirement.target.android-package; requirement.target.apple-package; requirement.foundation.memory-pointer-contract` | `target.android.mobile; target.apple.mobile` | 900 |
| `fixture.product.runtime-generation-denial` | `mirakan.arch.ai-security-approval` | `requirement.product.runtime-generation-boundary` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | 300 |
| `fixture.product.release-closure` | `mirakan.arch.product-plan` | `requirement.product.release-headless; requirement.product.release-windows-editor; requirement.product.release-windows-desktop; requirement.product.release-android; requirement.product.release-apple` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | 3600 |
| `fixture.product.cpp23-cx2-cross-target` | `mirakan.arch.cpp23-modules` | `requirement.product.cpp23-cx2-cross-target` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | 1800 |
| `fixture.product.cpp23-cx3-cross-target` | `mirakan.arch.cpp23-modules` | `requirement.product.cpp23-cx3-cross-target` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | 3600 |

`fixture.product.genreless-core-2d`はGenre／Feature Pack ref exact `[]`、Character／Combat／Objective／Completion exact `[]`のCore-only Projectで、明示Runtime Entry、2D World／Camera、deterministic timer、optional Physics／Animation Port、Save／Replayを検証する。`fixture.product.genreless-ai-project`はGenre Pack ref exact `[]`、zero-character／noncombat、WorldなしUI／logic Runtime Entryを既定branchとし、Phase 4ではprequalified Sourceだけ、Phase 5ではCode-owner付きProject C++ moduleとProject Shaderを追加する別Candidate branchを使う。両fixtureはShooter ID／Profile／Game Flow／Assetを入力、fallback、expected outputへ含めず、Shooter install時とuninstall時のContract set／Product dependency graph差を検査する。Pack dependency lint、UI-only、headless noncombatは`core-pack-independence` fixture matrixの別caseであり、一つの偽World／Characterへ近似変換しない。small-model Pack Discoveryはcurrent Operation setが空のためPhase Gateへ成功扱いで登録せず、`activation.pack.ai_operations.v1`のfuture acceptance fixtureとして`not_activated`を維持する。

`FallbackRegistryV1`はfallback entryと、そのentryが参照するProduct-owned Diagnostic recordを同じProduct root内で閉じる。別の15番目のActive Definition Registryを作らず、次のclosed shapeを本Registryのschemaに含める。

```text
ProductDiagnosticRefV1
  diagnostic_id
  diagnostic_version: positive uint32
  diagnostic_content_hash: SHA-256

ProductDiagnosticRecordV1
  diagnostic_id
  diagnostic_version: positive uint32
  owner_document_id
  severity: blocking | error | warning | info
  category
  message_key
  diagnostic_content_hash: SHA-256

ProductFallbackRecordV1
  fallback_id
  owner_document_id
  preserves_semantics: bool
  failure_diagnostic_ref: ProductDiagnosticRefV1

FallbackRegistryV1
  registry_id
  format_major: 1
  revision: positive safe integer
  entries[1..4096]: ProductFallbackRecordV1
  diagnostic_records[1..4096]: ProductDiagnosticRecordV1
```

`diagnostic_content_hash = SHA-256(ASCII "MIRAKAN_PRODUCT_DIAGNOSTIC_RECORD_V1" || uint32_be(len(RFC 8785 JCS(closed record excluding diagnostic_content_hash))) || RFC 8785 JCS(closed record excluding diagnostic_content_hash))`とする。entryはfallback ID、Diagnosticはdiagnostic ID／version順へstrict sortし、`failure_diagnostic_ref`集合と`diagnostic_records[]`をset equalityにする。同じID／version別hash、ID-only lookup、Owner prefix推測、Contract set `DiagnosticCodeRefV1`への型置換を拒否する。下表は可読表示としてDiagnostic IDだけを示すが、保存値は三Fieldの`ProductDiagnosticRefV1`である。

| fallback_id | Owner | preserves_semantics | failure diagnostic |
|---|---|---:|---|
| `fallback.capability.unavailable` | `mirakan.arch.product-plan` | false | `diagnostic.product.capability-unavailable` |
| `fallback.rendering.environment-core` | `mirakan.arch.rendering-environment-surfaces` | true | `diagnostic.rendering.environment-fallback-selected` |
| `fallback.rendering.ibl-baked` | `mirakan.arch.rendering-environment-surfaces` | true | `diagnostic.rendering.ibl-baked-selected` |
| `fallback.rendering.material-default` | `mirakan.arch.rendering-materials` | true | `diagnostic.rendering.material-default-selected` |
| `fallback.rendering.vfx-cpu` | `mirakan.arch.rendering-vfx-runtime` | true | `diagnostic.rendering.vfx-cpu-selected` |
| `fallback.rendering.vfx-core` | `mirakan.arch.rendering-vfx-runtime` | true | `diagnostic.rendering.vfx-core-selected` |

`diagnostic_records[]`のcurrent exact六recordは次である。全rowは`diagnostic_version=1`で、`diagnostic_content_hash`は上記自己除外式から再計算し、表にhash placeholderを保存しない。

| diagnostic_id | owner_document_id | severity | category | message_key |
|---|---|---|---|---|
| `diagnostic.product.capability-unavailable` | `mirakan.arch.product-plan` | `blocking` | `capability` | `diagnostic.product.capability-unavailable.message` |
| `diagnostic.rendering.environment-fallback-selected` | `mirakan.arch.rendering-environment-surfaces` | `warning` | `rendering.environment` | `diagnostic.rendering.environment-fallback-selected.message` |
| `diagnostic.rendering.ibl-baked-selected` | `mirakan.arch.rendering-environment-surfaces` | `warning` | `rendering.environment` | `diagnostic.rendering.ibl-baked-selected.message` |
| `diagnostic.rendering.material-default-selected` | `mirakan.arch.rendering-materials` | `warning` | `rendering.material` | `diagnostic.rendering.material-default-selected.message` |
| `diagnostic.rendering.vfx-cpu-selected` | `mirakan.arch.rendering-vfx-runtime` | `warning` | `rendering.vfx` | `diagnostic.rendering.vfx-cpu-selected.message` |
| `diagnostic.rendering.vfx-core-selected` | `mirakan.arch.rendering-vfx-runtime` | `warning` | `rendering.vfx` | `diagnostic.rendering.vfx-core-selected.message` |

各Fallback rowのOwnerは解決先Diagnostic recordの`owner_document_id`とbyte equalityにする。上表六ID／version集合、Fallbackの`failure_diagnostic_ref`集合、`diagnostic_records[]`集合はexact set equalityで、missing／extra／duplicate、severity／category／message key／Owner差をActive Product Definition compile failureにする。

`preserves_semantics=false`は代替実装ではなくfail-closedを表す。成功Receiptを発行せず、Capabilityを選択不能として説明する。

Executable ContractsのCapability `failure_modes[]`にある`fallback_id`はFoundation Contract setから本RegistryへのProduct foreign keyであり、MCD refではない。Baseline／Rebaseline verifierは同時に束縛されたexact Foundation Definition ClosureとActive Product Definition Bundleについて、全Capability foreign keyを本Registryへexact一件解決する。`fallback.capability.unavailable`は`preserves_semantics=false`かつ、exact `ProductDiagnosticRefV1`が`ProductDiagnosticRecordV1 {diagnostic_id=diagnostic.product.capability-unavailable, diagnostic_version=1, owner_document_id=mirakan.arch.product-plan, severity=blocking, category=capability, message_key=diagnostic.product.capability-unavailable.message}`のclosed recordへ一件解決し、その`diagnostic_content_hash`が直前の`MIRAKAN_PRODUCT_DIAGNOSTIC_RECORD_V1`式で再計算した32-byte SHA-256 digestと一定時間比較で一致することを必須にする。missing／multiple／別diagnostic／hash不一致／stale Product rootでは該当Capabilityをactivateしない。Foundation rootをProduct rootへ戻すhash edge、Product-owned DiagnosticをContract set Diagnosticへ偽装するedgeを作らない。

### 11.4 Product Phase registry

| order | phase_id | outcome requirements | work packages | exit gates |
|---:|---|---|---|---|
| 0 | `phase.foundation` | `requirement.target.headless-determinism; requirement.foundation.memory-pointer-contract` | `wp.architecture.control-plane; wp.foundation.cpp23-cx0; wp.foundation.math-core; wp.foundation.memory-pointers; wp.runtime.scheduling-core; wp.runtime.ecs-e0; wp.runtime.ecs-e1-storage; wp.runtime.ecs-e2-query-mutation` | `gate.product.phase-0-headless-contract; gate.product.phase-0-memory-pointer-contract` |
| 1 | `phase.headless-authoring` | `requirement.product.authoring-roundtrip-manual` | `wp.runtime.ecs-e3-cook-load; wp.authoring.project-state-headless; wp.authoring.asset-save-headless; wp.authoring.headless-core` | `gate.product.phase-1-authoring-transaction` |
| 2 | `phase.editor-runtime` | `requirement.target.windows-editor; requirement.target.windows-package; requirement.foundation.memory-pointer-contract; requirement.product.editor-reference-design` | `wp.runtime.ecs-e4-game-system; wp.rendering.render-graph-core; wp.runtime.d3d12-backend; wp.platform.input-core; wp.platform.audio-core; wp.platform.ui-core; wp.platform.input-windows; wp.platform.audio-windows; wp.platform.ui-windows; wp.platform.windows-package; wp.product.editor-runtime-windows` | `gate.product.phase-2-windows-empty-scene` |
| 3 | `phase.manual-2d` | `requirement.product.manual-first-playable-2d; requirement.product.core-pack-independence` | `wp.gameplay.core-c1; wp.runtime.timer; wp.rendering.world-2d; wp.rendering.camera-2d; wp.simulation.collision-2d; wp.simulation.physics-2d; wp.simulation.animation-2d; wp.navigation.core; wp.runtime.ecs-e5-2d-integration; wp.gameplay.reusable-features-c1; wp.domain.shooter-2d` | `gate.product.phase-3-manual-2d; gate.product.phase-3-genreless-core` |
| 4 | `phase.ai-authoring-mvp-a` | `requirement.product.ai-authoring-mvp-a; requirement.product.ai-genre-neutral-authoring` | `wp.authoring.ai-core; wp.runtime.debug-replay-support; wp.runtime.ecs-e6-debug-ai; wp.authoring.prequalified-source-packs; wp.runtime.ecs-e7-windows-2d; wp.product.ai-authoring-mvp-a` | `gate.product.phase-4-ai-mvp-a; gate.product.phase-4-ai-genreless` |
| 5 | `phase.external-agent` | `requirement.product.external-agent-boundary; requirement.product.project-source-activation` | `wp.product.external-agent; wp.authoring.project-native-module; wp.rendering.project-shader; wp.product.project-source-activation` | `gate.product.phase-5-external-agent; gate.product.phase-5-project-source-activation` |
| 6 | `phase.manual-3d-mvp-b` | `requirement.product.first-playable-3d` | `wp.rendering.world-3d; wp.rendering.camera-3d; wp.simulation.collision-3d; wp.simulation.physics-3d; wp.simulation.animation-3d; wp.runtime.ecs-e5-3d-integration; wp.runtime.ecs-e7-windows-3d; wp.domain.shooter-3d` | `gate.product.phase-6-first-playable-3d` |
| 7 | `phase.mobile` | `requirement.target.android-package; requirement.target.apple-package; requirement.foundation.memory-pointer-contract; requirement.target.mobile-runtime-2d; requirement.target.mobile-runtime-3d` | `wp.rendering.vulkan-backend; wp.rendering.metal-backend; wp.platform.mobile-offline; wp.platform.mobile-io-ui-android; wp.platform.mobile-io-ui-apple; wp.runtime.ecs-e7-android-2d; wp.runtime.ecs-e7-apple-2d; wp.runtime.ecs-e7-android-3d; wp.runtime.ecs-e7-apple-3d; wp.platform.android-package; wp.platform.apple-package` | `gate.product.phase-7-android-lifecycle; gate.product.phase-7-apple-lifecycle; gate.product.phase-7-android-runtime-2d; gate.product.phase-7-apple-runtime-2d; gate.product.phase-7-android-runtime-3d; gate.product.phase-7-apple-runtime-3d` |
| 8 | `phase.production-capability` | `requirement.product.c2-2d-coverage` | `wp.foundation.cpp23-cx2-cutover; wp.foundation.cpp23-cx3-shipping; wp.domain.platformer; wp.domain.puzzle-dialogue; wp.rendering.environment-c2; wp.rendering.vfx-c2; wp.rendering.material-realistic; wp.rendering.material-toon; wp.ui.native-widget; wp.product.general-coverage-2d; wp.product.general-coverage-3d; wp.product.production-release-binding` | `gate.product.phase-8-c2-shooter-2d; gate.product.phase-8-c2-platformer-2d; gate.product.phase-8-c2-puzzle-dialogue-2d` |
| 9 | `phase.runtime-generation` | `requirement.product.runtime-generation-boundary` | `wp.product.runtime-generation` | `gate.product.phase-9-runtime-generation-denial` |

Phase 5とPhase 9はこのRegistry行を唯一のscheduling identityとし、本文上の見出しや番号だけで存在を表現しない。Phase 5の`gate.product.phase-5-external-agent`はProposal-only authorizationだけを評価し、Native／Shader Sourceを作成またはactivateしない。`gate.product.phase-5-project-source-activation`はCode owner付き新規Source laneだけを評価し、external Client接続の成功を代用しない。Phase 8の三Genre gateは`capability.product.general_production_2d`とbundled Reference Game coverageだけを評価する独立trackで、Generic Engine Release、CX3 shipping、production-release bindingをblockしない。Phase 9はdeny-only C0 boundaryであり、Phase 8 coverageの完了またはC2 Product Capabilityへ依存せず、許可されたpositive Runtime structured-data generationは`future.capability.runtime-structured-data-generation`の`planning_only`境界に留める。

Phase exitのRequirement、Target、Candidate binding、Freshnessは次のbinding表だけが決定する。fixtureの全Requirement／Targetを暗黙評価せず、同じfixtureを別Phaseで使っても別gateのReceiptを代用しない。

| gate_id | phase_id | fixture_id | evaluated requirement refs | target refs | candidate binding policy | freshness policy |
|---|---|---|---|---|---|---|
| `gate.product.phase-0-headless-contract` | `phase.foundation` | `fixture.product.headless-contract-smoke` | `requirement.target.headless-determinism` | `target.headless.host` | `policy.product.same-candidate.v1` | `policy.evidence.contract-ci.v1` |
| `gate.product.phase-0-memory-pointer-contract` | `phase.foundation` | `fixture.foundation.memory-pointer-contract` | `requirement.foundation.memory-pointer-contract` | `target.headless.host` | `policy.product.same-candidate.v1` | `policy.evidence.contract-ci.v1` |
| `gate.product.phase-1-authoring-transaction` | `phase.headless-authoring` | `fixture.product.authoring-transaction` | `requirement.product.authoring-roundtrip-manual` | `target.headless.host` | `policy.product.same-candidate.v1` | `policy.evidence.contract-ci.v1` |
| `gate.product.phase-2-windows-empty-scene` | `phase.editor-runtime` | `fixture.product.windows-empty-scene` | `requirement.target.windows-editor; requirement.target.windows-package; requirement.foundation.memory-pointer-contract; requirement.product.editor-reference-design` | `target.windows.editor; target.windows.desktop` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-3-manual-2d` | `phase.manual-2d` | `fixture.product.shooter-2d` | `requirement.product.manual-first-playable-2d; requirement.product.title-to-result; requirement.product.save-load-replay` | `target.windows.desktop` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-3-genreless-core` | `phase.manual-2d` | `fixture.product.genreless-core-2d` | `requirement.product.core-pack-independence` | `target.windows.desktop` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-4-ai-mvp-a` | `phase.ai-authoring-mvp-a` | `fixture.product.shooter-2d` | `requirement.product.ai-authoring-mvp-a; requirement.product.authoring-roundtrip; requirement.product.mvp-completion; requirement.product.title-to-result; requirement.product.save-load-replay` | `target.windows.editor; target.windows.desktop` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-4-ai-genreless` | `phase.ai-authoring-mvp-a` | `fixture.product.genreless-ai-project` | `requirement.product.core-pack-independence; requirement.product.ai-authoring-mvp-a; requirement.product.ai-genre-neutral-authoring; requirement.product.authoring-roundtrip; requirement.product.mvp-completion; requirement.product.save-load-replay` | `target.windows.editor; target.windows.desktop` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-5-external-agent` | `phase.external-agent` | `fixture.product.external-agent-proposal` | `requirement.product.external-agent-boundary` | `target.headless.host` | `policy.product.same-candidate.v1` | `policy.evidence.contract-ci.v1` |
| `gate.product.phase-5-project-source-activation` | `phase.external-agent` | `fixture.product.genreless-ai-project` | `requirement.product.core-pack-independence; requirement.product.project-source-activation` | `target.windows.editor; target.windows.desktop` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-6-first-playable-3d` | `phase.manual-3d-mvp-b` | `fixture.product.shooter-arena-3d` | `requirement.product.first-playable-3d; requirement.product.title-to-result; requirement.product.save-load-replay` | `target.windows.desktop` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-7-android-lifecycle` | `phase.mobile` | `fixture.product.mobile-lifecycle` | `requirement.target.android-package; requirement.foundation.memory-pointer-contract` | `target.android.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-7-apple-lifecycle` | `phase.mobile` | `fixture.product.mobile-lifecycle` | `requirement.target.apple-package; requirement.foundation.memory-pointer-contract` | `target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-7-android-runtime-2d` | `phase.mobile` | `fixture.product.shooter-2d` | `requirement.target.mobile-runtime-2d` | `target.android.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-7-apple-runtime-2d` | `phase.mobile` | `fixture.product.shooter-2d` | `requirement.target.mobile-runtime-2d` | `target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-7-android-runtime-3d` | `phase.mobile` | `fixture.product.shooter-arena-3d` | `requirement.target.mobile-runtime-3d` | `target.android.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-7-apple-runtime-3d` | `phase.mobile` | `fixture.product.shooter-arena-3d` | `requirement.target.mobile-runtime-3d` | `target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-8-c2-shooter-2d` | `phase.production-capability` | `fixture.product.shooter-2d` | `requirement.product.c2-2d-coverage` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-8-c2-platformer-2d` | `phase.production-capability` | `fixture.product.platformer-2d` | `requirement.product.c2-2d-coverage` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-8-c2-puzzle-dialogue-2d` | `phase.production-capability` | `fixture.product.puzzle-dialogue-2d` | `requirement.product.c2-2d-coverage` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-8-cpp23-cx2-cross-target` | `phase.production-capability` | `fixture.product.cpp23-cx2-cross-target` | `requirement.product.cpp23-cx2-cross-target` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-8-cpp23-cx3-cross-target` | `phase.production-capability` | `fixture.product.cpp23-cx3-cross-target` | `requirement.product.cpp23-cx3-cross-target` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |
| `gate.product.phase-9-runtime-generation-denial` | `phase.runtime-generation` | `fixture.product.runtime-generation-denial` | `requirement.product.runtime-generation-boundary` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `policy.product.same-candidate.v1` | `policy.evidence.target-device.v1` |

`gate.product.phase-8-cpp23-cx2-cross-target`と`gate.product.phase-8-cpp23-cx3-cross-target`はCapability qualification専用bindingであり、`ProductPhaseRegistryV1.exit_gate_refs[]`には含めない。Phase bindingがPhase exitから未参照でも、`CapabilityTargetActivationBindingRegistryV1.qualification_gate_refs[]`からexactly one以上参照される上記2 IDだけはorphanではない。その他の未参照binding、またはPhase exitとCapability qualificationのどちらからも参照されないbindingを拒否する。この二GateによりCX2／CX3の5 Target計10行は`unavailable_no_gate`にならず、対応WP完了後に同一Candidate・全Target fresh Receiptを得た場合だけ`qualified`へ進める。C2 3Dの3行は§11.9の承認済み第二fixtureがまだないため意図的に`unavailable_no_gate`を維持する。

`ProductReleaseGateRegistryV1`はPhase exitと別であり、`ProductPhaseRegistryV1.exit_gate_refs[]`へ含めない。現行definitionにも次の5 definition rowを登録するが、Activation bindingのproduction policyはdisabledなので評価結果をproduction遷移へ使えない。Release-binding migration後だけTarget一致rowからexact一件を参照する。

| release_gate_id | target_id | fixture_id | evaluated requirement refs | required phase gates | required WP states | candidate binding | freshness | critical risk policy |
|---|---|---|---|---|---|---|---|---|
| `gate.product.release-headless` | `target.headless.host` | `fixture.product.release-closure` | `requirement.product.release-headless` | `gate.product.phase-0-headless-contract; gate.product.phase-1-authoring-transaction; gate.product.phase-5-external-agent; gate.product.phase-9-runtime-generation-denial` | `wp.foundation.cpp23-cx3-shipping=complete; wp.product.production-release-binding=complete` | `policy.product.same-candidate.v1` | `policy.evidence.release.v1` | `all_impact_critical_effective_mitigated_or_closed` |
| `gate.product.release-windows-editor` | `target.windows.editor` | `fixture.product.release-closure` | `requirement.product.release-windows-editor` | `gate.product.phase-2-windows-empty-scene; gate.product.phase-4-ai-genreless; gate.product.phase-5-external-agent; gate.product.phase-5-project-source-activation; gate.product.phase-9-runtime-generation-denial` | `wp.foundation.cpp23-cx3-shipping=complete; wp.platform.windows-package=complete; wp.product.production-release-binding=complete` | `policy.product.same-candidate.v1` | `policy.evidence.release.v1` | `all_impact_critical_effective_mitigated_or_closed` |
| `gate.product.release-windows-desktop` | `target.windows.desktop` | `fixture.product.release-closure` | `requirement.product.release-windows-desktop` | `gate.product.phase-2-windows-empty-scene; gate.product.phase-3-genreless-core; gate.product.phase-4-ai-genreless; gate.product.phase-5-project-source-activation; gate.product.phase-9-runtime-generation-denial` | `wp.foundation.cpp23-cx3-shipping=complete; wp.platform.windows-package=complete; wp.product.production-release-binding=complete` | `policy.product.same-candidate.v1` | `policy.evidence.release.v1` | `all_impact_critical_effective_mitigated_or_closed` |
| `gate.product.release-android` | `target.android.mobile` | `fixture.product.release-closure` | `requirement.product.release-android` | `gate.product.phase-7-android-lifecycle; gate.product.phase-9-runtime-generation-denial` | `wp.foundation.cpp23-cx3-shipping=complete; wp.platform.android-package=complete; wp.product.production-release-binding=complete` | `policy.product.same-candidate.v1` | `policy.evidence.release.v1` | `all_impact_critical_effective_mitigated_or_closed` |
| `gate.product.release-apple` | `target.apple.mobile` | `fixture.product.release-closure` | `requirement.product.release-apple` | `gate.product.phase-7-apple-lifecycle; gate.product.phase-9-runtime-generation-denial` | `wp.foundation.cpp23-cx3-shipping=complete; wp.platform.apple-package=complete; wp.product.production-release-binding=complete` | `policy.product.same-candidate.v1` | `policy.evidence.release.v1` | `all_impact_critical_effective_mitigated_or_closed` |

各Release Gateは参照Phase Gateのread-time fresh success、WP current state、fixture requirement、Target、same Candidate、release freshness、critical Risk closureを`all_of`で毎回再評価する。Generic Engine Releaseが参照できるPhase Gateはheadless／empty-scene、Genre／Feature Pack未導入の`phase-3-genreless-core`／`phase-4-ai-genreless`、external／Source境界、Target lifecycle、deny-only Phase 9のsubsetだけである。`phase-3-manual-2d`、`phase-4-ai-mvp-a`、`phase-6-first-playable-3d`、Phase 7のShooter fixtureによるruntime gate、Phase 8の三Genre coverage gate、および`wp.product.general-coverage-2d`をRelease predicateへ追加するDefinitionを拒否する。これらはbundled Reference Game／genre coverageの非blocking consumerだけが参照できる。closed predicate `all_impact_critical_effective_mitigated_or_closed`はcurrent `ProductRiskDefinitionRegistryV1`で`impact=critical`の全risk ID集合と評価対象をset equalityにし、各current effective stateが`mitigated | closed`の場合だけ成功する。`open | monitoring | accepted`、missing／extra Risk、expired／revoked Evidence、別Candidateならeffective failであり、保存ActivationがproductionでもRelease／claimを停止する。Phase 9のdeny-only security Gateは全5 Release Gateのglobal prerequisiteであり、未実行・失敗時にTarget別Gateを通さない。Release Gate自身はstateを保存せずEvidenceから投影する。

Planning Decision GateはPhase exit bindingと別Registryであり、`ProductPhaseRegistryV1.exit_gate_refs[]`へ含めない。

| gate_id | Owner | evaluator | required phase gates | required WP states | required evidence classes | required definition-change classes | genesis state | typed on-satisfied action |
|---|---|---|---|---|---|---|---|---|
| `gate.product.reconsider-cpp23-cx2-cutover` | `mirakan.arch.cpp23-modules` | `all_of` | `gate.product.phase-0-headless-contract; gate.product.phase-2-windows-empty-scene; gate.product.phase-7-android-lifecycle; gate.product.phase-7-apple-lifecycle` | `[]` | `evidence.class.cpp23-nonexperimental-import-std; evidence.class.cpp23-cross-target-cutover; evidence.class.cpp23-module-dag-valid; evidence.class.toolchain-lock-exact` | `[]` | `blocked` | `{kind=permit_work_package_transition, work_package_id=wp.foundation.cpp23-cx2-cutover, from_state=deferred, to_state=declared, transition_policy_ref=policy.product.wp.defer-release.v1}` |
| `gate.product.reconsider-cpp23-cx3-shipping` | `mirakan.arch.cpp23-modules` | `all_of` | `gate.product.phase-0-headless-contract; gate.product.phase-2-windows-empty-scene; gate.product.phase-3-genreless-core; gate.product.phase-4-ai-genreless; gate.product.phase-7-android-lifecycle; gate.product.phase-7-apple-lifecycle` | `{work_package_id=wp.foundation.cpp23-cx2-cutover, required_state=complete}` | `evidence.class.cpp23-formal-language-mode; evidence.class.cpp23-nonexperimental-import-std; evidence.class.cpp23-cross-target-release-readiness; evidence.class.package-install-offline-rollback-qualification` | `[]` | `blocked` | `{kind=permit_work_package_transition, work_package_id=wp.foundation.cpp23-cx3-shipping, from_state=deferred, to_state=declared, transition_policy_ref=policy.product.wp.defer-release.v1}` |
| `gate.product.reconsider-c2-3d` | `mirakan.arch.product-plan` | `all_of` | `gate.product.phase-6-first-playable-3d` | `[]` | `evidence.class.active-definition-validation-zero-error` | `definition.change.c2-3d-second-fixture-closure` | `blocked` | `{kind=permit_work_package_transition, work_package_id=wp.product.general-coverage-3d, from_state=deferred, to_state=declared, transition_policy_ref=policy.product.wp.defer-release.v1}` |
| `gate.product.reconsider-production-release-binding` | `mirakan.arch.product-plan` | `all_of` | `gate.product.phase-0-headless-contract; gate.product.phase-2-windows-empty-scene; gate.product.phase-3-genreless-core; gate.product.phase-4-ai-genreless; gate.product.phase-7-android-lifecycle; gate.product.phase-7-apple-lifecycle` | `{work_package_id=wp.foundation.cpp23-cx3-shipping, required_state=complete}` | `evidence.class.product-release-policy-ready; evidence.class.product-release-artifact-plan-valid` | `[]` | `blocked` | `{kind=permit_work_package_transition, work_package_id=wp.product.production-release-binding, from_state=deferred, to_state=declared, transition_policy_ref=policy.product.wp.defer-release.v1}` |

`evidence_class_definitions[]`の初期exact rowsは次の10件である。全行のwrapperは`TechnicalQualificationReceiptV1`、purposeは`technical_qualification_receipt`で、`subject_policy_ref`は同名classのprefixを`evidence.class.`から`policy.subject.`へ置換したclosed IDとする。subject policyはclass ID、Owner、same Candidate、current active definition、current Toolchain、下表Target exact set、class固有入力を持つcanonical closureのJCS hashを要求する。

| class_id | Owner | freshness policy | required Targets | class固有subject入力 |
|---|---|---|---|---|
| `evidence.class.cpp23-nonexperimental-import-std` | `mirakan.arch.cpp23-modules` | `policy.evidence.contract-ci.v1` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | formal compiler mode、`import std` probe、compiler artifact |
| `evidence.class.cpp23-cross-target-cutover` | `mirakan.arch.cpp23-modules` | `policy.evidence.contract-ci.v1` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | module compile／link／load fixture set |
| `evidence.class.cpp23-module-dag-valid` | `mirakan.arch.cpp23-modules` | `policy.evidence.contract-ci.v1` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | generated module DAG、cycle diagnostics |
| `evidence.class.toolchain-lock-exact` | `mirakan.arch.toolchain-dependencies` | `policy.evidence.contract-ci.v1` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | five-profile lock closure、artifact read-back |
| `evidence.class.cpp23-formal-language-mode` | `mirakan.arch.cpp23-modules` | `policy.evidence.contract-ci.v1` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | non-preview language flag／frontend identity |
| `evidence.class.cpp23-cross-target-release-readiness` | `mirakan.arch.cpp23-modules` | `policy.evidence.contract-ci.v1` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | release configuration compile／link plan and probes |
| `evidence.class.package-install-offline-rollback-qualification` | `mirakan.arch.ai-verification-provenance` | `policy.evidence.target-device.v1` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | package、install、offline launch、rollback fixtures |
| `evidence.class.active-definition-validation-zero-error` | `mirakan.arch.product-plan` | `policy.evidence.contract-ci.v1` | `target.headless.host` | 14-registry closure validator report |
| `evidence.class.product-release-policy-ready` | `mirakan.arch.product-plan` | `policy.evidence.contract-ci.v1` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | approved support／rollback／signing／SBOM policy |
| `evidence.class.product-release-artifact-plan-valid` | `mirakan.arch.ai-verification-provenance` | `policy.evidence.contract-ci.v1` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | artifact／Target lab／store staging plan |

`definition_change_class_definitions[]`は一件だけで、`definition.change.c2-3d-second-fixture-closure`をOwner `mirakan.arch.product-plan`、wrapper `ActiveProductDefinitionMigrationV1`、purpose `active_product_definition_migration`、subject policy `policy.subject.c2-3d-second-fixture-closure`へ束縛する。subject policyは下記第二Fixture closure、row manifest、Rebaseline closure、destination snapshot CASをexact検証する。definitionsのclass ID集合は全Gate `required_*_classes[]`のunionとset equalityにし、missing／extra／duplicate class、wrong wrapper／purpose／Owner／subject policy／freshness／Targetを拒否する。

`ProductDecisionGateRegistryV1`の三配列はrow migration時に一つのtagged logical-row unionとして扱う。`evidence_class_definitions[]`は`row_kind=evidence_class`かつ`logical_id`が`evidence.class.` prefix、`definition_change_class_definitions[]`は`row_kind=definition_change_class`かつ`definition.change.` prefix、`entries[]`は`row_kind=decision_gate`かつ`gate.product.` prefixでなければならない。三集合のlogical IDはRegistry全体でglobal unique、相互disjointであり、`ProductDefinitionRowMigrationManifestV1`は各行を`{registry_id,row_kind,logical_id}`でexactly once列挙する。prefixだけからrow内容を合成せず、wrong kind、二配列重複、manifestのkind欠落を拒否する。

`evaluator_policy`のclosed値は現在`all_of`だけである。`required_work_package_states[].required_state`は`complete`だけ、`required_evidence_classes[]`は上表の`evidence.class.*` 10値、`required_definition_change_classes[]`は`definition.change.c2-3d-second-fixture-closure`だけを受理する。Evidence classは自由文ではなく、classごとにOwnerが発行した完成signed Evidence content refの集合へ解決し、全refが同一Candidate、必要Target closure、current Toolchain lock、current active definition hashへ閉じ、freshかつ非revokedでなければならない。`cpp23-cross-target-release-readiness`と`package-install-offline-rollback-qualification`はCX3 WPを開始できるToolchain／candidate package前提であり、CX3 WP自身が後で発行するRelease Receiptを要求しない。`product-release-policy-ready`はProduct Plan Ownerがcompleted CX3、Genre／Feature Pack未導入のCore／AI holdout Gate、Target package Ownerのcurrent Receiptを入力に独立R4承認したsupport window、rollback、signing／SBOM／provenance policy Evidence、`product-release-artifact-plan-valid`はAI Verification／Provenance Release Evidence Ownerが同じprerequisite closureを入力に承認したartifact／Target lab／store staging plan Evidenceである。同OwnerはWindows、Android、Apple各Ownerのfresh Target-specific Receiptをexact set equalityで集約してEvidence classを発行するだけで、Target package schema、signing、upload、rollback、Store policyを所有または上書きしない。`wp.product.production-release-binding`、そのTask、またはCandidateはこのEvidenceを自己発行できない。General 2D／3D、Shooter／Platformer／Puzzle-Dialogue等のReference Game Receiptを両Evidenceの必須入力にしない。両Evidenceは`wp.product.production-release-binding`のdeferred→declaredより前に完成し、同WPまたはそのTask／candidateをproducer／subjectにせず、同WPはref／hashをread-backしてdestination projectionへ組み込むだけである。不足時はGateを`blocked`のまま保つ。実Release Receiptはdestination WP実行後の`fixture.product.release-closure`が発行する。`definition.change.c2-3d-second-fixture-closure`は承認済み`ActiveProductDefinitionMigrationV1`のrow migration manifestが、第二の非Shooter 3D Fixture、`requirement.product.c2-3d-coverage` binding、Genre／Rendering／UI provider WP、Owner、Windows／Android／Apple Target bindingを同一destination closureへ追加し、definition validatorがmissing／extra／duplicate／orphan／cycleを0件とした場合だけ成立する。

Decision評価時は、参照Phase Gateのcurrent fresh success、WP lifecycle headのexact state、Evidence class、definition-change classを`all_of`で毎回再評価する。保存stateが`satisfied`でも、いずれかのEvidenceがexpired／revoked／input hash不一致、Phase Gateが失効、WP stateが不一致になった時点でread-time `effective_state=blocked`とし、defer-releaseへ使わない。`on_satisfied_action`は許可候補を表すだけで自動遷移ではない。exact action一致、current `effective_state=satisfied`、`policy.product.wp.defer-release.v1`、独立Ownerのfresh `ProductOperationalDecisionV1`を揃えた`WorkPackageLifecycleRecordV1`だけが遷移を行い、Activation行の作成または昇格は一切行わない。自然文のpredicateまたはactionを実行入力に使うことを禁止する。

### 11.5 Work Package registry

表の`scheduling_state`は`wp.foundation.cpp23-cx2-cutover`、`wp.foundation.cpp23-cx3-shipping`、`wp.product.general-coverage-3d`、`wp.product.production-release-binding`の4件を`deferred`とし、それ以外を`declared`とする。計画書の存在を`ready`、`active`、`complete`へ読み替えず、deferred WPはPhase exitの完了条件へ含めない。

| work_package_id | phase_id | Owner | targets | fallback | provided fixtures | required capabilities | requires WP | scheduling state | defer_reason | reconsideration gates | blocked reason |
|---|---|---|---|---|---|---|---|---|---|---|---|
| `wp.architecture.control-plane` | `phase.foundation` | `mirakan.arch.core-architecture` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.headless-contract-smoke; fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `[]` | `[]` | `declared` | `null` | `[]` | `null` |
| `wp.foundation.cpp23-cx0` | `phase.foundation` | `mirakan.arch.toolchain-dependencies` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.headless-contract-smoke; fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.foundation.control-plane` | `wp.architecture.control-plane` | `declared` | `null` | `[]` | `null` |
| `wp.foundation.math-core` | `phase.foundation` | `mirakan.arch.math-core` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.headless-contract-smoke; fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.foundation.cpp23-cx0` | `wp.foundation.cpp23-cx0` | `declared` | `null` | `[]` | `null` |
| `wp.foundation.memory-pointers` | `phase.foundation` | `mirakan.arch.memory-pointers` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.foundation.memory-pointer-contract; fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.foundation.math-core` | `wp.foundation.math-core` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.scheduling-core` | `phase.foundation` | `mirakan.arch.runtime-scheduling-lifetime` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.headless-contract-smoke; fixture.foundation.memory-pointer-contract; fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.foundation.memory-pointers` | `wp.foundation.memory-pointers` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e0` | `phase.foundation` | `mirakan.arch.runtime-scheduling-lifetime` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.headless-contract-smoke; fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.runtime.scheduling` | `wp.runtime.scheduling-core` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e1-storage` | `phase.foundation` | `mirakan.arch.runtime-scheduling-lifetime` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.headless-contract-smoke; fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.runtime.ecs-e0-contract` | `wp.runtime.ecs-e0` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e2-query-mutation` | `phase.foundation` | `mirakan.arch.runtime-scheduling-lifetime` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.headless-contract-smoke; fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.runtime.ecs-e1-storage` | `wp.runtime.ecs-e1-storage` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e3-cook-load` | `phase.headless-authoring` | `mirakan.arch.runtime-scheduling-lifetime` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.authoring-transaction; fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.runtime.ecs-e2-query-mutation` | `wp.runtime.ecs-e2-query-mutation` | `declared` | `null` | `[]` | `null` |
| `wp.authoring.project-state-headless` | `phase.headless-authoring` | `mirakan.arch.project-state` | `target.headless.host; target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.authoring-transaction; fixture.product.windows-empty-scene` | `capability.runtime.ecs-e3-cook-load` | `wp.runtime.ecs-e3-cook-load` | `declared` | `null` | `[]` | `null` |
| `wp.authoring.asset-save-headless` | `phase.headless-authoring` | `mirakan.arch.asset-lifecycle` | `target.headless.host; target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.authoring-transaction; fixture.product.windows-empty-scene` | `capability.authoring.project-state-headless` | `wp.authoring.project-state-headless` | `declared` | `null` | `[]` | `null` |
| `wp.authoring.headless-core` | `phase.headless-authoring` | `mirakan.arch.project-state` | `target.headless.host; target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.authoring-transaction; fixture.product.windows-empty-scene` | `capability.authoring.asset-save-headless` | `wp.authoring.asset-save-headless` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e4-game-system` | `phase.editor-runtime` | `mirakan.arch.runtime-scheduling-lifetime` | `target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.runtime.ecs-e3-cook-load` | `wp.runtime.ecs-e3-cook-load; wp.authoring.headless-core` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.render-graph-core` | `phase.editor-runtime` | `mirakan.arch.rendering-render-graph` | `target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.foundation.cpp23-cx0; capability.runtime.ecs-e0-contract` | `wp.foundation.cpp23-cx0; wp.runtime.ecs-e0` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.d3d12-backend` | `phase.editor-runtime` | `mirakan.arch.rendering-render-graph` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene` | `capability.rendering.render-graph-core` | `wp.rendering.render-graph-core` | `declared` | `null` | `[]` | `null` |
| `wp.platform.input-core` | `phase.editor-runtime` | `mirakan.arch.platform-input` | `target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.runtime.ecs-e4-game-system` | `wp.runtime.ecs-e4-game-system` | `declared` | `null` | `[]` | `null` |
| `wp.platform.audio-core` | `phase.editor-runtime` | `mirakan.arch.platform-audio` | `target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.runtime.ecs-e4-game-system` | `wp.runtime.ecs-e4-game-system` | `declared` | `null` | `[]` | `null` |
| `wp.platform.ui-core` | `phase.editor-runtime` | `mirakan.arch.platform-ui-text-localization-accessibility` | `target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.platform.input-core; capability.platform.audio-core` | `wp.platform.input-core; wp.platform.audio-core` | `declared` | `null` | `[]` | `null` |
| `wp.platform.input-windows` | `phase.editor-runtime` | `mirakan.arch.platform-input` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene` | `capability.platform.input-core` | `wp.platform.input-core` | `declared` | `null` | `[]` | `null` |
| `wp.platform.audio-windows` | `phase.editor-runtime` | `mirakan.arch.platform-audio` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene` | `capability.platform.audio-core` | `wp.platform.audio-core` | `declared` | `null` | `[]` | `null` |
| `wp.platform.ui-windows` | `phase.editor-runtime` | `mirakan.arch.platform-ui-text-localization-accessibility` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene` | `capability.platform.ui-core` | `wp.platform.ui-core` | `declared` | `null` | `[]` | `null` |
| `wp.platform.windows-package` | `phase.editor-runtime` | `mirakan.arch.platform-windows` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene` | `capability.rendering.d3d12-cx0` | `wp.runtime.d3d12-backend; wp.authoring.headless-core` | `declared` | `null` | `[]` | `null` |
| `wp.product.editor-runtime-windows` | `phase.editor-runtime` | `mirakan.arch.product-plan` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.windows-empty-scene` | `capability.runtime.ecs-e4-game-system; capability.rendering.d3d12-cx0; capability.platform.input-core; capability.platform.audio-core; capability.platform.ui-core; capability.platform.windows-package` | `wp.runtime.ecs-e4-game-system; wp.runtime.d3d12-backend; wp.platform.input-windows; wp.platform.audio-windows; wp.platform.ui-windows; wp.platform.windows-package` | `declared` | `null` | `[]` | `null` |
| `wp.gameplay.core-c1` | `phase.manual-2d` | `mirakan.arch.gameplay-programming-model` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.genreless-core-2d; fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d; fixture.product.shooter-arena-3d` | `capability.runtime.ecs-e4-game-system` | `wp.runtime.ecs-e4-game-system` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.timer` | `phase.manual-2d` | `mirakan.arch.runtime-scheduling-lifetime` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.genreless-core-2d; fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d` | `capability.runtime.scheduling` | `wp.runtime.scheduling-core` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.world-2d` | `phase.manual-2d` | `mirakan.arch.rendering-world` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.genreless-core-2d; fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d` | `capability.rendering.render-graph-core` | `wp.rendering.render-graph-core` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.camera-2d` | `phase.manual-2d` | `mirakan.arch.rendering-camera` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.genreless-core-2d; fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d` | `capability.world.2d` | `wp.rendering.world-2d` | `declared` | `null` | `[]` | `null` |
| `wp.simulation.collision-2d` | `phase.manual-2d` | `mirakan.arch.simulation-collision` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.genreless-core-2d; fixture.product.shooter-2d; fixture.product.platformer-2d` | `capability.runtime.ecs-e4-game-system` | `wp.runtime.ecs-e4-game-system` | `declared` | `null` | `[]` | `null` |
| `wp.simulation.physics-2d` | `phase.manual-2d` | `mirakan.arch.simulation-physics` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.genreless-core-2d; fixture.product.shooter-2d; fixture.product.platformer-2d` | `capability.simulation.collision-2d` | `wp.simulation.collision-2d` | `declared` | `null` | `[]` | `null` |
| `wp.simulation.animation-2d` | `phase.manual-2d` | `mirakan.arch.simulation-animation` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.genreless-core-2d; fixture.product.shooter-2d; fixture.product.platformer-2d` | `capability.runtime.timer` | `wp.runtime.timer` | `declared` | `null` | `[]` | `null` |
| `wp.navigation.core` | `phase.manual-2d` | `mirakan.arch.simulation-navigation` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.genreless-core-2d; fixture.feature.path_following.executor_stub` | `capability.runtime.ecs-e4-game-system` | `wp.runtime.ecs-e4-game-system` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e5-2d-integration` | `phase.manual-2d` | `mirakan.arch.runtime-scheduling-lifetime` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.genreless-core-2d; fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d; fixture.product.mobile-lifecycle` | `capability.world.2d; capability.camera.2d; capability.simulation.physics-2d; capability.simulation.animation-2d` | `wp.rendering.world-2d; wp.rendering.camera-2d; wp.simulation.physics-2d; wp.simulation.animation-2d` | `declared` | `null` | `[]` | `null` |
| `wp.gameplay.reusable-features-c1` | `phase.manual-2d` | `mirakan.arch.pack-gameplay-features` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.feature.combat.contract; fixture.feature.ranged_combat.contract; fixture.feature.encounter_spawn.contract; fixture.feature.scoring.contract; fixture.feature.pickup_grant.provider_neutral; fixture.feature.interaction.contract; fixture.feature.character_locomotion.motion_executor; fixture.feature.path_following.executor_stub; fixture.feature.scenario_stage.none` | `capability.runtime.timer; capability.simulation.navigation` | `wp.gameplay.core-c1; wp.runtime.timer; wp.navigation.core` | `declared` | `null` | `[]` | `null` |
| `wp.domain.shooter-2d` | `phase.manual-2d` | `mirakan.arch.pack-shooter` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-2d` | `capability.gameplay.combat; capability.gameplay.ranged_combat; capability.gameplay.encounter_spawn; capability.gameplay.scoring; capability.gameplay.pickup_grant; capability.gameplay.interaction; capability.gameplay.character_locomotion; capability.gameplay.path_following; capability.gameplay.scenario_stage; capability.gameplay.perception; capability.runtime.ecs-e5-2d-integration; capability.camera.2d` | `wp.gameplay.reusable-features-c1; wp.runtime.ecs-e5-2d-integration; wp.rendering.camera-2d` | `declared` | `null` | `[]` | `null` |
| `wp.authoring.ai-core` | `phase.ai-authoring-mvp-a` | `mirakan.arch.ai-security-approval` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.genreless-ai-project; fixture.product.shooter-2d` | `capability.authoring.manual-roundtrip` | `wp.authoring.headless-core` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.debug-replay-support` | `phase.ai-authoring-mvp-a` | `mirakan.arch.runtime-debugging-observability-replay` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.genreless-ai-project; fixture.product.shooter-2d` | `capability.runtime.scheduling; capability.authoring.manual-roundtrip` | `wp.runtime.scheduling-core; wp.authoring.headless-core` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e6-debug-ai` | `phase.ai-authoring-mvp-a` | `mirakan.arch.runtime-scheduling-lifetime` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.genreless-ai-project; fixture.product.shooter-2d` | `capability.runtime.ecs-e4-game-system; capability.authoring.ai-core; capability.runtime.debug-replay-support` | `wp.runtime.ecs-e4-game-system; wp.authoring.ai-core; wp.runtime.debug-replay-support` | `declared` | `null` | `[]` | `null` |
| `wp.authoring.prequalified-source-packs` | `phase.ai-authoring-mvp-a` | `mirakan.arch.native-game-module` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.genreless-ai-project; fixture.product.shooter-2d` | `capability.authoring.ai-core; capability.runtime.ecs-e6-debug-ai` | `wp.authoring.ai-core; wp.runtime.ecs-e6-debug-ai` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e7-windows-2d` | `phase.ai-authoring-mvp-a` | `mirakan.arch.runtime-scheduling-lifetime` | `target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.genreless-core-2d; fixture.product.shooter-2d` | `capability.runtime.ecs-e5-2d-integration; capability.runtime.ecs-e6-debug-ai; capability.authoring.prequalified-source-packs` | `wp.runtime.ecs-e5-2d-integration; wp.runtime.ecs-e6-debug-ai; wp.authoring.prequalified-source-packs` | `declared` | `null` | `[]` | `null` |
| `wp.product.ai-authoring-mvp-a` | `phase.ai-authoring-mvp-a` | `mirakan.arch.product-plan` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.genreless-ai-project; fixture.product.shooter-2d` | `capability.authoring.ai-core; capability.runtime.debug-replay-support; capability.authoring.prequalified-source-packs; capability.platform.windows-package` | `wp.authoring.ai-core; wp.runtime.debug-replay-support; wp.authoring.prequalified-source-packs; wp.platform.windows-package` | `declared` | `null` | `[]` | `null` |
| `wp.product.external-agent` | `phase.external-agent` | `mirakan.arch.ai-security-approval` | `target.headless.host` | `fallback.capability.unavailable` | `fixture.product.external-agent-proposal` | `capability.foundation.control-plane` | `wp.product.ai-authoring-mvp-a` | `declared` | `null` | `[]` | `null` |
| `wp.authoring.project-native-module` | `phase.external-agent` | `mirakan.arch.native-game-module` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.genreless-ai-project` | `capability.authoring.ai-core; capability.runtime.ecs-e6-debug-ai` | `wp.authoring.ai-core; wp.runtime.ecs-e6-debug-ai` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.project-shader` | `phase.external-agent` | `mirakan.arch.rendering-project-shader` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.genreless-ai-project` | `capability.authoring.ai-core; capability.runtime.ecs-e6-debug-ai; capability.rendering.render-graph-core` | `wp.authoring.ai-core; wp.runtime.ecs-e6-debug-ai; wp.rendering.render-graph-core` | `declared` | `null` | `[]` | `null` |
| `wp.product.project-source-activation` | `phase.external-agent` | `mirakan.arch.product-plan` | `target.windows.editor; target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.genreless-ai-project` | `capability.product.ai-authoring-mvp-a; capability.project.native_module; capability.project.shader` | `wp.product.ai-authoring-mvp-a; wp.authoring.project-native-module; wp.rendering.project-shader` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.world-3d` | `phase.manual-3d-mvp-b` | `mirakan.arch.rendering-world` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.rendering.render-graph-core` | `wp.rendering.render-graph-core` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.camera-3d` | `phase.manual-3d-mvp-b` | `mirakan.arch.rendering-camera` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.world.3d` | `wp.rendering.world-3d` | `declared` | `null` | `[]` | `null` |
| `wp.simulation.collision-3d` | `phase.manual-3d-mvp-b` | `mirakan.arch.simulation-collision` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.runtime.ecs-e4-game-system` | `wp.runtime.ecs-e4-game-system` | `declared` | `null` | `[]` | `null` |
| `wp.simulation.physics-3d` | `phase.manual-3d-mvp-b` | `mirakan.arch.simulation-physics` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.simulation.collision-3d` | `wp.simulation.collision-3d` | `declared` | `null` | `[]` | `null` |
| `wp.simulation.animation-3d` | `phase.manual-3d-mvp-b` | `mirakan.arch.simulation-animation` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.simulation.physics-3d; capability.runtime.timer` | `wp.simulation.physics-3d; wp.runtime.timer` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e5-3d-integration` | `phase.manual-3d-mvp-b` | `mirakan.arch.runtime-scheduling-lifetime` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d; fixture.product.mobile-lifecycle` | `capability.world.3d; capability.camera.3d; capability.simulation.physics-3d; capability.simulation.animation-3d` | `wp.rendering.world-3d; wp.rendering.camera-3d; wp.simulation.physics-3d; wp.simulation.animation-3d` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e7-windows-3d` | `phase.manual-3d-mvp-b` | `mirakan.arch.runtime-scheduling-lifetime` | `target.windows.desktop` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.runtime.ecs-e5-3d-integration; capability.runtime.ecs-e6-debug-ai` | `wp.runtime.ecs-e5-3d-integration; wp.runtime.ecs-e6-debug-ai` | `declared` | `null` | `[]` | `null` |
| `wp.domain.shooter-3d` | `phase.manual-3d-mvp-b` | `mirakan.arch.pack-shooter` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.gameplay.combat; capability.gameplay.ranged_combat; capability.gameplay.encounter_spawn; capability.gameplay.scoring; capability.gameplay.pickup_grant; capability.gameplay.interaction; capability.gameplay.character_locomotion; capability.gameplay.path_following; capability.gameplay.scenario_stage; capability.gameplay.perception; capability.runtime.ecs-e5-3d-integration; capability.camera.3d` | `wp.gameplay.reusable-features-c1; wp.runtime.ecs-e5-3d-integration; wp.rendering.camera-3d` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.vulkan-backend` | `phase.mobile` | `mirakan.arch.rendering-render-graph` | `target.android.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle` | `capability.rendering.render-graph-core` | `wp.rendering.render-graph-core` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.metal-backend` | `phase.mobile` | `mirakan.arch.rendering-render-graph` | `target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle` | `capability.rendering.render-graph-core` | `wp.rendering.render-graph-core` | `declared` | `null` | `[]` | `null` |
| `wp.platform.mobile-offline` | `phase.mobile` | `mirakan.arch.platform-mobile-common` | `target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle` | `capability.runtime.ecs-e4-game-system` | `wp.runtime.ecs-e4-game-system` | `declared` | `null` | `[]` | `null` |
| `wp.platform.mobile-io-ui-android` | `phase.mobile` | `mirakan.arch.platform-mobile-common` | `target.android.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle; fixture.product.shooter-2d; fixture.product.shooter-arena-3d` | `capability.platform.input-core; capability.platform.audio-core; capability.platform.ui-core` | `wp.platform.input-core; wp.platform.audio-core; wp.platform.ui-core` | `declared` | `null` | `[]` | `null` |
| `wp.platform.mobile-io-ui-apple` | `phase.mobile` | `mirakan.arch.platform-mobile-common` | `target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle; fixture.product.shooter-2d; fixture.product.shooter-arena-3d` | `capability.platform.input-core; capability.platform.audio-core; capability.platform.ui-core` | `wp.platform.input-core; wp.platform.audio-core; wp.platform.ui-core` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e7-android-2d` | `phase.mobile` | `mirakan.arch.runtime-scheduling-lifetime` | `target.android.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle; fixture.product.genreless-core-2d; fixture.product.shooter-2d` | `capability.runtime.ecs-e5-2d-integration; capability.rendering.vulkan-backend; capability.platform.mobile-io-ui-android` | `wp.runtime.ecs-e5-2d-integration; wp.rendering.vulkan-backend; wp.platform.mobile-io-ui-android` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e7-apple-2d` | `phase.mobile` | `mirakan.arch.runtime-scheduling-lifetime` | `target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle; fixture.product.genreless-core-2d; fixture.product.shooter-2d` | `capability.runtime.ecs-e5-2d-integration; capability.rendering.metal-backend; capability.platform.mobile-io-ui-apple` | `wp.runtime.ecs-e5-2d-integration; wp.rendering.metal-backend; wp.platform.mobile-io-ui-apple` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e7-android-3d` | `phase.mobile` | `mirakan.arch.runtime-scheduling-lifetime` | `target.android.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle; fixture.product.shooter-arena-3d` | `capability.runtime.ecs-e5-3d-integration; capability.rendering.vulkan-backend; capability.platform.mobile-lifecycle; capability.platform.mobile-io-ui-android` | `wp.runtime.ecs-e5-3d-integration; wp.rendering.vulkan-backend; wp.platform.mobile-offline; wp.platform.mobile-io-ui-android` | `declared` | `null` | `[]` | `null` |
| `wp.runtime.ecs-e7-apple-3d` | `phase.mobile` | `mirakan.arch.runtime-scheduling-lifetime` | `target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle; fixture.product.shooter-arena-3d` | `capability.runtime.ecs-e5-3d-integration; capability.rendering.metal-backend; capability.platform.mobile-lifecycle; capability.platform.mobile-io-ui-apple` | `wp.runtime.ecs-e5-3d-integration; wp.rendering.metal-backend; wp.platform.mobile-offline; wp.platform.mobile-io-ui-apple` | `declared` | `null` | `[]` | `null` |
| `wp.platform.android-package` | `phase.mobile` | `mirakan.arch.platform-android` | `target.android.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle; fixture.product.shooter-2d; fixture.product.shooter-arena-3d` | `capability.rendering.vulkan-backend; capability.runtime.ecs-e7-android-2d; capability.runtime.ecs-e7-android-3d; capability.platform.mobile-lifecycle` | `wp.rendering.vulkan-backend; wp.runtime.ecs-e7-android-2d; wp.runtime.ecs-e7-android-3d; wp.platform.mobile-offline` | `declared` | `null` | `[]` | `null` |
| `wp.platform.apple-package` | `phase.mobile` | `mirakan.arch.platform-apple` | `target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.mobile-lifecycle; fixture.product.shooter-2d; fixture.product.shooter-arena-3d` | `capability.rendering.metal-backend; capability.runtime.ecs-e7-apple-2d; capability.runtime.ecs-e7-apple-3d; capability.platform.mobile-lifecycle` | `wp.rendering.metal-backend; wp.runtime.ecs-e7-apple-2d; wp.runtime.ecs-e7-apple-3d; wp.platform.mobile-offline` | `declared` | `null` | `[]` | `null` |
| `wp.foundation.cpp23-cx2-cutover` | `phase.production-capability` | `mirakan.arch.cpp23-modules` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.cpp23-cx2-cross-target; fixture.product.headless-contract-smoke; fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.foundation.cpp23-cx0` | `wp.foundation.cpp23-cx0; wp.platform.windows-package; wp.platform.android-package; wp.platform.apple-package` | `deferred` | `CMake 4.4のimport stdがExperimental opt-inかつNinja系限定で、全Target Cutover／Tooling／ABI Receiptが未成立のためCX0 Headerを維持する` | `gate.product.reconsider-cpp23-cx2-cutover` | `null` |
| `wp.foundation.cpp23-cx3-shipping` | `phase.production-capability` | `mirakan.arch.cpp23-modules` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.cpp23-cx3-cross-target; fixture.product.headless-contract-smoke; fixture.product.windows-empty-scene; fixture.product.mobile-lifecycle` | `capability.foundation.cpp23-cx2-cutover` | `wp.foundation.cpp23-cx2-cutover` | `deferred` | `MSVC 14.51はpreviewのC++23だけであり、正式C++23 language mode、非Experimental CMake、全Target Package／Release Receiptが未成立のためRelease Activation不可` | `gate.product.reconsider-cpp23-cx3-shipping` | `null` |
| `wp.domain.platformer` | `phase.production-capability` | `mirakan.arch.gameplay-programming-model` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.platformer-2d` | `capability.gameplay.interaction; capability.runtime.timer; capability.gameplay.path_following` | `wp.gameplay.reusable-features-c1; wp.runtime.timer; wp.runtime.ecs-e7-windows-2d; wp.runtime.ecs-e7-android-2d; wp.runtime.ecs-e7-apple-2d` | `declared` | `null` | `[]` | `null` |
| `wp.domain.puzzle-dialogue` | `phase.production-capability` | `mirakan.arch.gameplay-programming-model` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.puzzle-dialogue-2d` | `capability.gameplay.interaction; capability.runtime.timer; capability.platform.ui-core` | `wp.gameplay.reusable-features-c1; wp.runtime.timer; wp.platform.ui-windows; wp.runtime.ecs-e7-windows-2d; wp.runtime.ecs-e7-android-2d; wp.runtime.ecs-e7-apple-2d` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.environment-c2` | `phase.production-capability` | `mirakan.arch.rendering-environment-surfaces` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.rendering.environment-core` | `fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d; fixture.product.shooter-arena-3d` | `capability.rendering.render-graph-core` | `wp.rendering.render-graph-core; wp.platform.android-package; wp.platform.apple-package` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.vfx-c2` | `phase.production-capability` | `mirakan.arch.rendering-vfx-runtime` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.rendering.vfx-core` | `fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d; fixture.product.shooter-arena-3d` | `capability.rendering.render-graph-core` | `wp.rendering.render-graph-core; wp.platform.android-package; wp.platform.apple-package` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.material-realistic` | `phase.production-capability` | `mirakan.arch.rendering-materials` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.rendering.material-default` | `fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d; fixture.product.shooter-arena-3d` | `capability.rendering.render-graph-core` | `wp.rendering.render-graph-core; wp.platform.android-package; wp.platform.apple-package` | `declared` | `null` | `[]` | `null` |
| `wp.rendering.material-toon` | `phase.production-capability` | `mirakan.arch.rendering-materials` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.rendering.material-default` | `fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d; fixture.product.shooter-arena-3d` | `capability.rendering.render-graph-core` | `wp.rendering.render-graph-core; wp.platform.android-package; wp.platform.apple-package` | `declared` | `null` | `[]` | `null` |
| `wp.ui.native-widget` | `phase.production-capability` | `mirakan.arch.platform-ui-text-localization-accessibility` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d; fixture.product.shooter-arena-3d` | `capability.platform.ui-core` | `wp.platform.ui-windows; wp.platform.mobile-io-ui-android; wp.platform.mobile-io-ui-apple` | `declared` | `null` | `[]` | `null` |
| `wp.product.general-coverage-2d` | `phase.production-capability` | `mirakan.arch.product-plan` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.genreless-core-2d; fixture.product.shooter-2d; fixture.product.platformer-2d; fixture.product.puzzle-dialogue-2d` | `capability.domain.platformer; capability.domain.puzzle-dialogue; capability.environment.core; capability.vfx.system; capability.render.material.realistic_advanced; capability.render.material.toon; capability.ui.native_widget; capability.gameplay.interaction; capability.runtime.timer; capability.gameplay.path_following` | `wp.domain.platformer; wp.domain.puzzle-dialogue; wp.rendering.environment-c2; wp.rendering.vfx-c2; wp.rendering.material-realistic; wp.rendering.material-toon; wp.ui.native-widget; wp.gameplay.reusable-features-c1; wp.runtime.timer` | `declared` | `null` | `[]` | `null` |
| `wp.product.general-coverage-3d` | `phase.production-capability` | `mirakan.arch.product-plan` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.shooter-arena-3d` | `capability.environment.core; capability.vfx.system; capability.render.material.realistic_advanced; capability.render.material.toon; capability.ui.native_widget; capability.gameplay.interaction; capability.runtime.timer; capability.gameplay.path_following` | `wp.rendering.environment-c2; wp.rendering.vfx-c2; wp.rendering.material-realistic; wp.rendering.material-toon; wp.ui.native-widget; wp.gameplay.reusable-features-c1; wp.runtime.timer` | `deferred` | `第二の非Shooter 3D Fixture、C2 Requirement binding、provider WP、Owner、三Target bindingを一つのProduct Decisionでactive Registryへ登録しvalidationを通すまでC2 3Dを評価不能` | `gate.product.reconsider-c2-3d` | `null` |
| `wp.product.production-release-binding` | `phase.production-capability` | `mirakan.arch.product-plan` | `target.headless.host; target.windows.editor; target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.release-closure` | `capability.foundation.cpp23-cx3-shipping` | `wp.foundation.cpp23-cx3-shipping; wp.runtime.ecs-e5-2d-integration; wp.platform.windows-package; wp.platform.android-package; wp.platform.apple-package` | `deferred` | `current Activation Binding全293行からmodeを導出する。全disabledのsourceではproduction_release_migration_authoringとしてpreliminary destinationだけを作りcomplete禁止、全all_release_gatesのdestinationではproduction_release_binding_qualificationとしてDefinition不変のfresh Target closure＋Owner acceptanceでcompleteするまでproduction遷移を禁止する。Genre／Feature Pack未導入のCore holdoutは`wp.runtime.ecs-e5-2d-integration`の`fixture.product.genreless-core-2d`で検証し、bundled Reference Game coverageは別nonblocking trackとする` | `gate.product.reconsider-production-release-binding` | `null` |
| `wp.product.runtime-generation` | `phase.runtime-generation` | `mirakan.arch.ai-security-approval` | `target.windows.desktop; target.android.mobile; target.apple.mobile` | `fallback.capability.unavailable` | `fixture.product.runtime-generation-denial` | `capability.foundation.control-plane` | `wp.architecture.control-plane` | `declared` | `null` | `[]` | `null` |

`wp.rendering.environment-c2`は`capability.rendering.render-graph-core`が提供するtyped `LightSnapshotV1` runtime入力を利用するが、これをLighting discovery／authoringのProduct activationまたは新しいLighting WPとして数えない。Time-of-Day intentがsun／moon Source変更を必要とする場合だけ、[Lighting](../06-rendering/lighting.md)のexact `ResolvedLightPlanRefV1`を同じProject revision／World／Targetへ解決する。`planning.operation_family.lighting_discovery`が未Activation、Planがmissing／stale、対象Light集合が不一致の時はEnvironment側だけの部分変更を成功させず、質問またはCapability未Activationとして停止する。

#### ECS Work Packageのtarget owner mapping

Work Package表の`Owner`列はcurrent generated Architecture Document Registryへ解決する`owner_document_id`であり、文書刷新だけで先行変更しない。したがって`wp.runtime.ecs-e0`から`wp.runtime.ecs-e7-*`の現行Owner列は、`RuntimeEcsCanonicalizationChangeSetV1`が`applied`になるまで旧current authorityを指す。この扱いはECS実装またはOwner移管を許可するものではない。

次表はProductの計画上のtarget責務を示す閲覧用mappingであり、Work Package lifecycle、Capability activation、current Owner Registryを変更しない。

| Work Package群 | target Architecture Owner | 境界 |
|---|---|---|
| `ecs-e0`、`ecs-e1-storage`、`ecs-e2-query-mutation` | [Runtime ECS](../04-runtime/entity-component-system.md) | Entity／Component、archetype、query、access manifest、structural transaction |
| `ecs-e3-cook-load` | [Runtime Package](../04-runtime/runtime-package.md)＋[Asset Lifecycle](../03-authoring/asset-lifecycle.md)＋[Persistence／Save](../04-runtime/persistence-save.md) | World image、artifact catalog、load／reconstruction boundary |
| `ecs-e4-game-system`、`ecs-e5-*-integration` | [Runtime ECS](../04-runtime/entity-component-system.md)＋[Gameplay programming model](../03-authoring/gameplay-programming-model.md) | System binding、Domain integration、current／target Owner transfer |
| `ecs-e6-debug-ai` | [Runtime ECS](../04-runtime/entity-component-system.md)＋[Debugging／Observability／Replay](../04-runtime/debugging-observability-replay.md)＋[AI Security／Approval](../01-governance/ai-security-approval.md) | sealed projection、debug transport、authorization |
| `ecs-e7-*-*` | [Runtime ECS](../04-runtime/entity-component-system.md)＋各Platform Owner | Target qualification。Platform-specific runtimeをECS正本へ複写しない |

### Destination projection: data-oriented ECS core

This subsection is a closed projection of an unmaterialized, unapproved, and unapplied destination migration candidate associated with `RuntimeEcsCanonicalizationChangeSetV1` and its Owner-reference migration. The current Governance profile remains `state=review`, `contract_activation_effect=none`, with `definition_migration_binding_ref` absent. This subsection is not part of the current source Definition and does not change the operational snapshot. Only if a complete Product Definition Migration is separately materialized and approved do the destination projection, Change Set, and Owner-reference migration apply atomically; the source remains byte-equal until that atomic application.

| Registry | Exact addition or replacement |
|---|---|
| `RequirementRegistryV1` | `{requirement_id=requirement.runtime.ecs-data-oriented-core, owner_document_id=mirakan.arch.runtime-entity-component-system, verification_kind=runtime_ecs_data_oriented_qualification, failure_diagnostic_id=diagnostic.product.ecs-data-oriented-core-failed}` |
| `FixtureRegistryV1` | `{fixture_id=fixture.runtime.ecs-data-oriented-core, owner_document_id=mirakan.arch.runtime-entity-component-system, requirement_refs=[requirement.runtime.ecs-data-oriented-core], target_refs=[target.headless.host,target.windows.desktop,target.android.mobile,target.apple.mobile], minimum_duration_seconds=12600}` |
| `PhaseFixtureBindingRegistryV1` | `{gate_id=gate.product.phase-0-ecs-data-oriented-core, phase_id=phase.foundation, fixture_id=fixture.runtime.ecs-data-oriented-core, evaluated_requirement_refs=[requirement.runtime.ecs-data-oriented-core], target_refs=[target.headless.host], candidate_binding_policy_ref=policy.product.same-candidate.v1, freshness_policy_ref=policy.evidence.contract-ci.v1}` |
| `ProductPhaseRegistryV1` `phase.foundation` | append the new Requirement to `outcome_requirement_refs[]` and the new Gate to `exit_gate_refs[]`; `work_package_refs[]` is unchanged |
| `WorkPackageRegistryV1` | append `{kind=product_fixture, fixture_id=fixture.runtime.ecs-data-oriented-core}` to `provided_fixture_refs[]` of `wp.foundation.memory-pointers`, `wp.runtime.scheduling-core`, `wp.runtime.ecs-e0`, `wp.runtime.ecs-e1-storage`, and `wp.runtime.ecs-e2-query-mutation`; all other Fields remain unchanged except the coordinated Owner replacements |

Headless Phase 0 GateはWindows、Android、Apple qualificationを代用しない。各Targetはfull 12,600-second fixtureを`policy.evidence.target-device.v1`で再実行する。

destinationにおける`requirement.foundation.memory-pointer-contract`のdefinition closureはexactly次の4 Typeである。

```text
[PointerContractV1, MemoryContractV1, PointerMemoryConsumerBindingV1, CppValueTransferPolicyV1]
```

同Requirementは、同じfour-Type Contract Set内のexact `MemoryContractV1` Type member ref／schema hashと、retained-Field、single-`capacity_source`、six-layout／access-Field invariantを証明するfresh `fixture.foundation.memory-pointer-contract` Receiptへ束縛する。Memory field listは本書へ複写しない。

destinationだけで`risk.product.memory-pointer-contract-drift.mitigation`を次のexact valueへ置換し、他Fieldはsource rowとbyte-equalに保つ。

```text
the exact four-Type definition closure [PointerContractV1, MemoryContractV1, PointerMemoryConsumerBindingV1, CppValueTransferPolicyV1], bidirectional consumer Matrix, static／negative fixture, supported sanitizer lane, and hot path fallback 0 are bound to the same Phase 0 Candidate Gate
```

Owner-reference migrationは`wp.runtime.ecs-e0`、`wp.runtime.ecs-e1-storage`、`wp.runtime.ecs-e2-query-mutation`の`owner_document_id`だけを`mirakan.arch.runtime-scheduling-lifetime`から`mirakan.arch.runtime-entity-component-system`へ置換する。`wp.foundation.memory-pointers`、`wp.runtime.scheduling-core`、依存chainは変更しない。

destination Requirementはfixture Componentのaccepted `RuntimeComponentLayoutPolicyV1` record、`ecs_chunk_soa_v1`を使う一つの`RuntimeArchetypeLayoutPlanV1`、query／lease／structural contract、全35 mandatory metric ID、全hard predicate、同一Candidateに束縛した8192／16384／32768-byte characterization、Shipping AoS／sparse-set／object graph／general-heap fallback 0を要求する。

| Work Package | Added completion responsibility |
|---|---|
| `wp.foundation.memory-pointers` | value-transfer policy、container layout Fields、static and negative Gates |
| `wp.runtime.ecs-e0` | type、owner、diagnostic, and Contract closure |
| `wp.runtime.ecs-e1-storage` | chunk SoA、layout policy、capacity、handle、fragmentation metrics |
| `wp.runtime.ecs-e2-query-mutation` | cached query、contiguous dispatch、allocation-free callback、deferred structural transaction |
| later `wp.runtime.ecs-e7-*` | rerun the qualified profile on the exact Target and device Evidence policy |

Phase Fixture GateはRequirement、fixture、Target、Candidate、freshnessだけを評価する。Phase exitは別に、E1／E2を含む全non-deferred Work Packageのcurrent lifecycle headが`complete`であることを要求する。

destination `ProductRiskRegistryV1`には次の一行を追加する。risk rowの唯一OwnerはProduct Planであり、他文書はtrigger、severity、mitigation、containmentを複写しない。

| risk_id | owner_document_id | affected_work_package_refs[] | trigger | likelihood | impact | mitigation | contingency | monitor_gate_refs[] | genesis_state | revisit_gate_or_date |
|---|---|---|---|---|---|---|---|---|---|---|
| `risk.product.ecs-data-oriented-regression` | `mirakan.arch.runtime-entity-component-system` | `wp.foundation.memory-pointers; wp.runtime.scheduling-core; wp.runtime.ecs-e0; wp.runtime.ecs-e1-storage; wp.runtime.ecs-e2-query-mutation` | missing layout policy、dual Shipping layout、hot callback allocation／fallback、unbounded archetype growth、missing campaign cell／metric, or wrong-Target Receipt substitution | `high` | `critical` | require the destination Phase 0 Gate、complete campaign、hard predicates, and fresh Target-specific reruns | reject the affected ECS Work Package transition and dependent Runtime activation; retain the last qualified layout without an alternate Shipping fallback | `gate.product.phase-0-ecs-data-oriented-core` | `open` | `{kind=phase_gate, ref=gate.product.phase-0-ecs-data-oriented-core}` |

```text
diagnostic.product.ecs-target-receipt-mismatch
MIRAKAN-PRODUCT-ECS-TARGET-RECEIPT-MISMATCH
arguments = campaign_hash, expected_target_ref, actual_target_ref
result = Product Gate failure
```

aggregate failureはRuntime ECS所有の`diagnostic.product.ecs-data-oriented-core-failed`を参照し、本書でschemaを複製しない。

`wp.authoring.prequalified-source-packs`は初心者向けDefinition-first経路と、事前Qualification済みNative／Shader Packの選択だけを提供する。Phase 5の`wp.authoring.project-native-module`と`wp.rendering.project-shader`で新規AI生成Sourceを採用する場合は、各Ownerの独立したCode owner approval GateとSource／artifact／Receipt hash closureを必須とし、事前PackのReceiptを代用しない。`wp.product.project-source-activation`は両Source WPを集約するが、`wp.product.external-agent`とは依存もRequirementも共有せず、Proposal-only境界をSource生成の成功扱いにしない。

Production owner layerがGeneric Engine CoreであるWPは、`required_capability_refs[]`または`requires_work_package_refs[]`からFeature Pack、Genre Pack、Reference Gameへ到達してはならない。Reference Gameを`provided_fixture_refs[]`の`kind=product_fixture`として置くことはQualification inputの宣言でありproduction dependencyではないが、各Core Capabilityは同じTargetで`fixture.product.genreless-core-2d`または`fixture.product.genreless-ai-project`を使うPack-uninstalled holdout ReceiptなしにActivationできない。`wp.navigation.core`はNavigation query／artifact／provider portと`capability.simulation.navigation`だけを所有し、`capability.gameplay.path_following`は`wp.gameplay.reusable-features-c1`が所有する。InteractionもPublic schemaの正本所在と実装Capability ownerを分け、`capability.gameplay.interaction`のProduct owner WPはFeature Packに固定する。

`wp.runtime.ecs-e7-windows-2d`と`capability.runtime.ecs-e7-windows-2d`のartifact／runtime Targetは`target.windows.desktop`だけである。AI／EditorがそのBuild、Test、Previewを起動するexecution Hostは§10.6の`execution_host_rule=windows_editor_for_artifact_target`で`target.windows.editor`へ別に束縛し、WP／Capability TargetへEditorを重複登録しない。これによりDesktop-only `capability.runtime.ecs-e5-2d-integration`をEditor artifact Capabilityへ偽装せず、Editor Host ReceiptをDesktop runtime Receiptとして流用しない。

`wp.platform.input-core`、`wp.platform.audio-core`、`wp.platform.ui-core`はportable contractを所有し、対応する`*-windows` WPはWindows qualificationだけを所有する。`mobile-io-ui-*`からportable core WPへの`requires_work_package_refs[]`はcontract build-order edgeであり、Windows qualification WPまたはWindows Receiptへの依存ではない。`wp.domain.shooter-2d`と`wp.domain.shooter-3d`はWindows／Android／Apple共通のportable Genre implementationを所有し、Generic Runtime、AI、Debug、Source、Shader、Target backend、E7、Packageのいずれからもrequired edgeを受けない。Android／Appleの実機I/O／UIは同Targetの`wp.platform.mobile-io-ui-android`／`wp.platform.mobile-io-ui-apple`、2D／3D ECS統合はTarget別E7 WPがCore subsystem closureだけをrequired edgeに持ってfresh Phase 7 Receiptを提供する。Shooter fixtureはTarget別E7のQualification caseとして利用できるが、Shooter Capability、Shooter WP、Shooter ReceiptをE7またはPackageの成立条件にしない。Phase 8のPlatformer／Puzzle ReceiptもPhase 7 Activation前提にしない。既存V1 IDの`wp.domain.*`／`capability.domain.*`にある`domain` tokenは互換用opaque segmentであり製品層を表さない。authoritative layer／artifact roleはArchitecture Dependency分類とGeneric Architectureの四層規則だけから解決し、AI／Compiler／Release GateはID substringからGenre、Core、required dependencyを推測しない。

### 11.6 Capability–Target–Fallback registry

本表はCapability identity、Product tier、owner WP、Target scope、fallbackだけを所有し、Activation stateを保存しない。Target scopeの`Headless`、`Windows`、`Windows Editor`、`Android`、`Apple`はそれぞれ`target.headless.host`、`target.windows.desktop`、`target.windows.editor`、`target.android.mobile`、`target.apple.mobile`の略記であり、Registry生成時は略記を保存せずexact IDへ展開する。明記しないTargetは暗黙requiredにせず`excluded`として生成する。

| capability_id | tier | owner WP | Target scope | fallback |
|---|---|---|---|---|
| `capability.foundation.control-plane` | C0 | `wp.architecture.control-plane` | Headless required; Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.foundation.cpp23-cx0` | C0 | `wp.foundation.cpp23-cx0` | Headless required; Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.foundation.cpp23-cx2-cutover` | C0 | `wp.foundation.cpp23-cx2-cutover` | Headless required; Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.foundation.cpp23-cx3-shipping` | C0 | `wp.foundation.cpp23-cx3-shipping` | Headless required; Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.foundation.math-core` | C0 | `wp.foundation.math-core` | Headless required; Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.foundation.memory-pointers` | C0 | `wp.foundation.memory-pointers` | Headless required; Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.runtime.scheduling` | C0 | `wp.runtime.scheduling-core` | Headless required; Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e0-contract` | C0 | `wp.runtime.ecs-e0` | Headless required; Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e1-storage` | C0 | `wp.runtime.ecs-e1-storage` | Headless required; Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e2-query-mutation` | C0 | `wp.runtime.ecs-e2-query-mutation` | Headless required; Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e3-cook-load` | C0 | `wp.runtime.ecs-e3-cook-load` | Headless required; Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.authoring.project-state-headless` | C0 | `wp.authoring.project-state-headless` | Headless required; Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.authoring.asset-save-headless` | C0 | `wp.authoring.asset-save-headless` | Headless required; Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.authoring.manual-roundtrip` | C0 | `wp.authoring.headless-core` | Headless required; Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e4-game-system` | C0 | `wp.runtime.ecs-e4-game-system` | Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.rendering.render-graph-core` | C0 | `wp.rendering.render-graph-core` | Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.rendering.d3d12-cx0` | C0 | `wp.runtime.d3d12-backend` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.platform.input-core` | C0 | `wp.platform.input-core` | Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.platform.audio-core` | C0 | `wp.platform.audio-core` | Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.platform.ui-core` | C0 | `wp.platform.ui-core` | Windows Editor required; Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.platform.windows-package` | C0 | `wp.platform.windows-package` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.product.editor-runtime-windows` | C0 | `wp.product.editor-runtime-windows` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.world.2d` | C1 | `wp.rendering.world-2d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.camera.2d` | C1 | `wp.rendering.camera-2d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.simulation.collision-2d` | C1 | `wp.simulation.collision-2d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.simulation.physics-2d` | C1 | `wp.simulation.physics-2d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.simulation.animation-2d` | C1 | `wp.simulation.animation-2d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.simulation.navigation` | C1 | `wp.navigation.core` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e5-2d-integration` | C1 | `wp.runtime.ecs-e5-2d-integration` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.domain.shooter-2d` | C1 | `wp.domain.shooter-2d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.authoring.ai-core` | C1 | `wp.authoring.ai-core` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.runtime.debug-replay-support` | C1 | `wp.runtime.debug-replay-support` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e6-debug-ai` | C1 | `wp.runtime.ecs-e6-debug-ai` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.authoring.prequalified-source-packs` | C1 | `wp.authoring.prequalified-source-packs` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e7-windows-2d` | C1 | `wp.runtime.ecs-e7-windows-2d` | Windows Editor excluded; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.product.ai-authoring-mvp-a` | C1 | `wp.product.ai-authoring-mvp-a` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.product.external-agent` | C1 | `wp.product.external-agent` | Headless required; Windows Editor excluded; Windows excluded; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.product.project-source-activation` | C1 | `wp.product.project-source-activation` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.world.3d` | C1 | `wp.rendering.world-3d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.camera.3d` | C1 | `wp.rendering.camera-3d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.simulation.collision-3d` | C1 | `wp.simulation.collision-3d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.simulation.physics-3d` | C1 | `wp.simulation.physics-3d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.simulation.animation-3d` | C1 | `wp.simulation.animation-3d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e5-3d-integration` | C1 | `wp.runtime.ecs-e5-3d-integration` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e7-windows-3d` | C1 | `wp.runtime.ecs-e7-windows-3d` | Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e7-android-2d` | C1 | `wp.runtime.ecs-e7-android-2d` | Android required; Windows Editor excluded; Windows excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e7-apple-2d` | C1 | `wp.runtime.ecs-e7-apple-2d` | Apple required; Windows Editor excluded; Windows excluded; Android excluded | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e7-android-3d` | C1 | `wp.runtime.ecs-e7-android-3d` | Android required; Windows Editor excluded; Windows excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.runtime.ecs-e7-apple-3d` | C1 | `wp.runtime.ecs-e7-apple-3d` | Apple required; Windows Editor excluded; Windows excluded; Android excluded | `fallback.capability.unavailable` |
| `capability.domain.shooter-3d` | C1 | `wp.domain.shooter-3d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.rendering.vulkan-backend` | C1 | `wp.rendering.vulkan-backend` | Android required; Windows Editor excluded; Windows excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.rendering.metal-backend` | C1 | `wp.rendering.metal-backend` | Apple required; Windows Editor excluded; Windows excluded; Android excluded | `fallback.capability.unavailable` |
| `capability.platform.android-package` | C1 | `wp.platform.android-package` | Android required; Windows Editor excluded; Windows excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.platform.apple-package` | C1 | `wp.platform.apple-package` | Apple required; Windows Editor excluded; Windows excluded; Android excluded | `fallback.capability.unavailable` |
| `capability.platform.mobile-lifecycle` | C1 | `wp.platform.mobile-offline` | Android required; Apple required; Windows Editor excluded; Windows excluded | `fallback.capability.unavailable` |
| `capability.platform.mobile-io-ui-android` | C1 | `wp.platform.mobile-io-ui-android` | Android required; Windows Editor excluded; Windows excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.platform.mobile-io-ui-apple` | C1 | `wp.platform.mobile-io-ui-apple` | Apple required; Windows Editor excluded; Windows excluded; Android excluded | `fallback.capability.unavailable` |
| `capability.domain.platformer` | C2 | `wp.domain.platformer` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.domain.puzzle-dialogue` | C2 | `wp.domain.puzzle-dialogue` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.environment.aerial_perspective` | C2 | `wp.rendering.environment-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.environment-core` |
| `capability.environment.atmosphere_lut` | C2 | `wp.rendering.environment-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.environment-core` |
| `capability.environment.cloud_shadow` | C2 | `wp.rendering.environment-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.environment-core` |
| `capability.environment.core` | C1 | `wp.rendering.environment-c2` | Windows required; Android required; Apple required | `fallback.rendering.ibl-baked` |
| `capability.environment.dynamic_ibl` | C2 | `wp.rendering.environment-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.ibl-baked` |
| `capability.environment.height_fog` | C2 | `wp.rendering.environment-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.environment-core` |
| `capability.environment.ibl_baked` | C1 | `wp.rendering.environment-c2` | Windows required; Android required; Apple required | `fallback.rendering.environment-core` |
| `capability.environment.intent_resolver` | C1 | `wp.rendering.environment-c2` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.environment.local_fog_volume` | C2 | `wp.rendering.environment-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.environment-core` |
| `capability.environment.sky_hdri` | C1 | `wp.rendering.environment-c2` | Windows required; Android required; Apple required | `fallback.rendering.ibl-baked` |
| `capability.environment.volumetric_cloud` | C2 | `wp.rendering.environment-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.environment-core` |
| `capability.environment.volumetric_fog` | C2 | `wp.rendering.environment-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.environment-core` |
| `capability.gameplay.interaction` | C1 | `wp.gameplay.reusable-features-c1` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.gameplay.path_following` | C1 | `wp.gameplay.reusable-features-c1` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.gameplay.perception` | C1 | `wp.gameplay.core-c1` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.gameplay.combat` | C1 | `wp.gameplay.reusable-features-c1` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.gameplay.ranged_combat` | C1 | `wp.gameplay.reusable-features-c1` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.gameplay.encounter_spawn` | C1 | `wp.gameplay.reusable-features-c1` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.gameplay.scoring` | C1 | `wp.gameplay.reusable-features-c1` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.gameplay.pickup_grant` | C1 | `wp.gameplay.reusable-features-c1` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.gameplay.character_locomotion` | C1 | `wp.gameplay.reusable-features-c1` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.gameplay.scenario_stage` | C1 | `wp.gameplay.reusable-features-c1` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.runtime.timer` | C1 | `wp.runtime.timer` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.project.native_module` | C1 | `wp.authoring.project-native-module` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.project.shader` | C1 | `wp.rendering.project-shader` | Windows Editor required; Windows required; Android excluded; Apple excluded | `fallback.capability.unavailable` |
| `capability.product.general_production_2d` | C2 | `wp.product.general-coverage-2d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.product.general_production_3d` | C2 | `wp.product.general-coverage-3d` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.product.runtime-generation-boundary` | C0 | `wp.product.runtime-generation` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.render.material.realistic_advanced` | C2 | `wp.rendering.material-realistic` | Windows required; Android required; Apple required | `fallback.rendering.material-default` |
| `capability.render.material.toon` | C2 | `wp.rendering.material-toon` | Windows required; Android required; Apple required | `fallback.rendering.material-default` |
| `capability.ui.native_widget` | C2 | `wp.ui.native-widget` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.vfx.bake_cache` | C2 | `wp.rendering.vfx-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-core` |
| `capability.vfx.billboard_3d` | C1 | `wp.rendering.vfx-c2` | Windows required; Android required; Apple required | `fallback.rendering.vfx-cpu` |
| `capability.vfx.extension_operator` | C2 | `wp.rendering.vfx-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-core` |
| `capability.vfx.mesh_ribbon` | C2 | `wp.rendering.vfx-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-cpu` |
| `capability.vfx.particle_cpu` | C1 | `wp.rendering.vfx-c2` | Windows required; Android required; Apple required | `fallback.rendering.vfx-core` |
| `capability.vfx.particle_gpu` | C2 | `wp.rendering.vfx-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-cpu` |
| `capability.vfx.particle_light` | C2 | `wp.rendering.vfx-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-core` |
| `capability.vfx.pattern_catalog` | C1 | `wp.rendering.vfx-c2` | Windows required; Android required; Apple required | `fallback.rendering.vfx-core` |
| `capability.vfx.semantic_intent` | C1 | `wp.rendering.vfx-c2` | Windows required; Android required; Apple required | `fallback.capability.unavailable` |
| `capability.vfx.sprite_2d` | C1 | `wp.rendering.vfx-c2` | Windows required; Android required; Apple required | `fallback.rendering.vfx-cpu` |
| `capability.vfx.system` | C1 | `wp.rendering.vfx-c2` | Windows required; Android required; Apple required | `fallback.rendering.vfx-core` |
| `capability.vfx.trail` | C1 | `wp.rendering.vfx-c2` | Windows required; Android required; Apple required | `fallback.rendering.vfx-cpu` |
| `capability.vfx.visual_collision` | C2 | `wp.rendering.vfx-c2` | Windows required; Android optional; Apple optional | `fallback.rendering.vfx-core` |

`required_capability_refs[]`はconsumer WPの各`target_refs[]`で実行時に必須なCapabilityだけを列挙する。各edgeについてCapability表の同Target scopeが`required`でなければ拒否し、`optional`または`excluded`をrequired edgeに使わない。optional Capabilityを選べるWPはrequired edgeへ混在させず、Owner contractの明示selection policyとfallback refで表し、選択時だけcurrent Target state／fresh fallbackを検査する。headless authoring、build host、Code owner、Phase順序などcross-targetのorderingだけを表す関係は`requires_work_package_refs[]`へ置き、Target availabilityの証拠にしない。`defer_reason`、依存、`reconsideration_gate_refs[]`はWork Packageだけが所有し、Capability表または`CapabilityTargetActivationStateV1`へ複写しない。

現行`WorkPackageRegistryV1`にはoptional Capability選択Fieldがなく、全`required_capability_refs[]` edgeは`required` scopeだけで閉じる。将来WPがoptional selection自体をProduct正本へ持つ場合は、selection policy、fallback、selected／unselected fixtureを含むDefinition schema revisionを先に承認し、自由記述や既存required edgeの読み替えで追加しない。

### 11.7 Product risk registry

`likelihood`のclosed値は`low | medium | high | unknown`、`impact`は`moderate | major | critical`、operational `state`は`open | monitoring | mitigated | accepted | closed`である。`revisit_gate_or_date`は`kind=phase_gate`なら登録済みPhase Fixture Gateへの`ref`、`kind=decision_gate`なら登録済みProduct Decision Gateへの`ref`、`kind=date`なら根拠のあるISO 8601 `YYYY-MM-DD`の`value`だけを受理し、kind不一致Field、unknown kind、裸の文字列を拒否する。Markdown表は厳密な略記`phase_gate:<exact-id>`、`decision_gate:<exact-id>`、`date:<YYYY-MM-DD>`だけを許し、Registry生成時に上記discriminated unionへ展開する。本Revisionは未確定日程を作らず、全行をexact gateで再評価する。表の`monitor gate refs`はDefinitionのlogical gate refsであり、Evidence content refではない。genesis `risk_evaluations[].state=open`、`evidence_refs=[]`とし、後続の署名済みEvaluation transitionだけが実Receipt refを追加する。

| risk_id | Owner | affected work packages | trigger | likelihood | impact | mitigation | contingency | monitor gate refs | genesis state | revisit gate or date |
|---|---|---|---|---|---|---|---|---|---|---|
| `risk.product.control-plane-schema-drift` | `mirakan.arch.core-architecture` | `wp.architecture.control-plane` | Product schema consumerがcanonical Field名、closed値、またはrevisionと不一致になる | `high` | `critical` | Product Registryを唯一の正本とし、bootstrap前にschema hashとexact Field集合を検査する | scheduling startを拒否し、last-known-good schemaへ固定する | `gate.product.phase-0-headless-contract` | `open` | `phase_gate:gate.product.phase-0-headless-contract` |
| `risk.product.cpp23-modules-cmake-constraint` | `mirakan.arch.cpp23-modules` | `wp.foundation.cpp23-cx0; wp.foundation.cpp23-cx2-cutover; wp.foundation.cpp23-cx3-shipping` | CX0 productionへCX1 fixture外の`.ixx`／`.cppm`が混入する、またはCX2／CX3でCMake import std、BMI順序、Tooling、正式C++23、全Target Receiptのいずれかが成立しない | `high` | `critical` | CX0はself-contained Headerへ固定し、CX1 fixtureを隔離し、CX2 cutoverとCX3 Shippingを別deferred WP／Decision Gateで判定する | CX0を維持してCX2／CX3 schedulingとRelease Activationを拒否し、別Generatorまたはpreview Toolchainへfallbackしない | `gate.product.phase-0-headless-contract; gate.product.reconsider-cpp23-cx2-cutover; gate.product.reconsider-cpp23-cx3-shipping` | `open` | `decision_gate:gate.product.reconsider-cpp23-cx3-shipping` |
| `risk.product.memory-pointer-contract-drift` | `mirakan.arch.memory-pointers` | `wp.foundation.memory-pointers; wp.runtime.scheduling-core; wp.runtime.ecs-e0; wp.runtime.ecs-e1-storage; wp.runtime.ecs-e2-query-mutation` | consumer binding、owner、保存／job capture制約、またはsupported sanitizer Evidenceのいずれかが欠落し、local wrapperや暗黙fallbackで迂回される | `high` | `critical` | MCDの三Contract、正逆consumer Matrix、static／negative fixture、supported sanitizer lane、hot path fallback 0をPhase 0の同一Candidate Gateへ束縛する | 当該WPのschedulingと後続Runtime activationを拒否し、欠落Evidenceを別Target／旧ID／aliasで代用しない | `gate.product.phase-0-memory-pointer-contract` | `open` | `phase_gate:gate.product.phase-0-memory-pointer-contract` |
| `risk.product.ai-tool-safety-code-owner` | `mirakan.arch.ai-security-approval` | `wp.authoring.ai-core; wp.authoring.project-native-module; wp.rendering.project-shader; wp.product.project-source-activation` | AI toolが権限外Operationを要求する、または新規Native／`bounded_hlsl`にCode Ownerが割り当てられない、または`typed_ir`のcoverage／Target gateが欠落する | `high` | `critical` | MVP-AをDefinition-first／prequalified Packへ限定し、Native／`bounded_hlsl`を独立GateとCode Owner approvalへ、`typed_ir`をIR・coverage・Target gateへ送る | 新規Source／IR laneをfail closedにし、MVP-AとProposal-only external agentを維持する | `gate.product.phase-4-ai-mvp-a; gate.product.phase-5-project-source-activation` | `open` | `phase_gate:gate.product.phase-5-project-source-activation` |
| `risk.product.target-device-signing` | `mirakan.arch.product-plan` | `wp.platform.windows-package; wp.platform.android-package; wp.platform.apple-package` | 実機、署名資格情報、store package、device lab、またはoffline launch Receiptが揃わない | `high` | `critical` | Target別package WP、同一Candidate binding、target-device freshnessで検査する | 未合格TargetをProduct labelとShippingから除外する | `gate.product.phase-2-windows-empty-scene; gate.product.phase-7-android-lifecycle; gate.product.phase-7-apple-lifecycle; gate.product.phase-7-android-runtime-2d; gate.product.phase-7-apple-runtime-2d; gate.product.phase-7-android-runtime-3d; gate.product.phase-7-apple-runtime-3d` | `open` | `phase_gate:gate.product.phase-7-apple-runtime-3d` |
| `risk.product.asset-license-provenance` | `mirakan.arch.asset-lifecycle` | `wp.authoring.asset-save-headless; wp.product.ai-authoring-mvp-a` | Asset source、license、生成由来、またはSBOM provenanceが同一Project revisionへ閉じない | `medium` | `critical` | Asset import／save ReceiptとAI provenanceをCandidate hashへ束縛する | 問題Assetを隔離し、packageとProduct labelを拒否する | `gate.product.phase-1-authoring-transaction; gate.product.phase-4-ai-mvp-a` | `open` | `phase_gate:gate.product.phase-4-ai-mvp-a` |
| `risk.product.c1-performance-calibration` | `mirakan.arch.runtime-performance-capacity` | `wp.gameplay.core-c1; wp.domain.shooter-2d; wp.domain.shooter-3d` | C1 fixtureのframe、memory、load、soak基準が実測Target profileでcalibrateされていない | `unknown` | `major` | Phase 3／6／7 Candidateで同一入力のTarget別baselineを測定し、推測値をReceiptへ置換する | C1表示を対象Targetで保留し、機能削減を勝手に行わない | `gate.product.phase-3-manual-2d; gate.product.phase-6-first-playable-3d; gate.product.phase-7-android-runtime-2d; gate.product.phase-7-apple-runtime-2d; gate.product.phase-7-android-runtime-3d; gate.product.phase-7-apple-runtime-3d` | `open` | `phase_gate:gate.product.phase-7-apple-runtime-3d` |
| `risk.product.mobile-backend-closure` | `mirakan.arch.rendering-render-graph` | `wp.rendering.vulkan-backend; wp.rendering.metal-backend; wp.platform.mobile-io-ui-android; wp.platform.mobile-io-ui-apple; wp.runtime.ecs-e7-android-2d; wp.runtime.ecs-e7-apple-2d; wp.runtime.ecs-e7-android-3d; wp.runtime.ecs-e7-apple-3d; wp.platform.android-package; wp.platform.apple-package` | Mobile backend、I/O／UI、2D／3D ECS qualificationのいずれかが対象Targetで未合格またはWindows Receiptを流用する | `high` | `critical` | Android／Apple別WP、C1 Shooter実機Gate、CapabilityTargetActivation行を使い、Windows Receipt流用を0件にする | 該当Mobile TargetのpackageとShipping labelを拒否する | `gate.product.phase-7-android-lifecycle; gate.product.phase-7-apple-lifecycle; gate.product.phase-7-android-runtime-2d; gate.product.phase-7-apple-runtime-2d; gate.product.phase-7-android-runtime-3d; gate.product.phase-7-apple-runtime-3d` | `open` | `phase_gate:gate.product.phase-7-apple-runtime-3d` |
| `risk.product.c2-3d-second-fixture` | `mirakan.arch.product-plan` | `wp.product.general-coverage-3d` | 第二の非Shooter 3D Fixtureとprovider closureがactive Registryに存在しない | `high` | `major` | aggregate WPをexact理由付き`deferred`にし、closed Decision Gateだけで再検討する | Product labelを`3D First Playable`に限定し、C2 3D Receiptを発行しない | `gate.product.phase-6-first-playable-3d; gate.product.reconsider-c2-3d` | `open` | `decision_gate:gate.product.reconsider-c2-3d` |
| `risk.product.online-large-world-incubation` | `mirakan.arch.product-plan` | `wp.product.general-coverage-3d; wp.product.runtime-generation` | Online、Large World、またはpositive Runtime generationをplanning-only Decisionなしでactive DAGへ入れようとする | `medium` | `critical` | 対応FutureCapability entryを`planning_only`に保ち、Owner、authority、Threat Model、Target、fallback、Qualificationを一括Decision化する | activationを拒否し、offline／deny-only Product boundaryを維持する | `gate.product.phase-8-c2-shooter-2d; gate.product.phase-9-runtime-generation-denial` | `open` | `phase_gate:gate.product.phase-9-runtime-generation-denial` |

### 11.8 C++23 Cutover／Shipping gate

CX0／CX1はDevelopment、Test、candidate Package、internal Technology Previewだけに使用し、Release Activationへ入力しない。CX0／CX1を使う`capability.rendering.d3d12-cx0`は内部`candidate_locked`を上限とし、`qualified`／`production`への遷移を拒否する。Shipping Configurationのcandidate Package成功は公開Releaseの証拠ではない。

`wp.foundation.cpp23-cx2-cutover`と`capability.foundation.cpp23-cx2-cutover`は全TargetのModule Cutover候補、`wp.foundation.cpp23-cx3-shipping`と`capability.foundation.cpp23-cx3-shipping`は全TargetのRelease有効化を所有する。両Work Packageは初期`deferred`、両CapabilityのTarget行は初期`not_activated`であり、Planning Decision Gateが`satisfied`でもWork Packageの再schedulingだけを許可してActivationを自動昇格しない。

CMakeの非Experimental `import std`、Ninja／Ninja Multi-Config経路、正式`/std:c++23`、全TargetのBuild／Tooling／ABI／Package／Release Receiptのいずれかが欠ける場合はCX0 Headerを維持する。Visual Studio Generator由来BMI、MSVC 14.51の`/std:c++23preview`、CX1 fixture ReceiptをCX2／CX3の代用にしない。

### 11.9 C2 3D gate

`fixture.product.shooter-arena-3d`一件だけではProduct C2 3Dを評価しない。本RevisionのPhase 8 outcomeと`PhaseFixtureBindingRegistryV1`には`requirement.product.c2-3d-coverage`のexit gateが存在せず、`wp.product.general-coverage-3d`は`gate.product.reconsider-c2-3d`が`satisfied`になるまで`deferred`である。第二の非Shooter 3D Fixtureを未登録IDで先取りせず、同Decision Gateのpredicateが要求するFixture、Requirement binding、provider WP、Owner、Windows／Android／Apple Target closureを一つの承認済みChangeSetで登録し、validationが0 errorになった後だけ再schedulingできる。

`capability.product.general_production_3d`のTarget別保存状態はcurrent `CapabilityTargetActivationStateV1`だけが所有する。Decision Gateの`satisfied`、Work Packageの再scheduling、Shooter 3D ReceiptのいずれもTarget行を自動昇格せず、第二fixtureを含む新しいPhase bindingのfresh Receiptが揃うまでProduct labelを`3D First Playable`に限定する。
