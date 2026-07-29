# Miraikanai Engine Product Plan

- 文書ID: mirakan.arch.product-plan
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Product intent、非交渉原則、Capability成熟度、Portfolio、Algorithm／Performance最適化のProduct優先度、AI制作理解境界、MVP、Phase順序、製品昇格・停止・完了Gate
- 非正本範囲: Subsystemの型・Field・API・Backend・既定値・Budget、AI権限と承認、Evidence形式。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)
- 関連文書: [Product Lifecycle](product-lifecycle.md)、[Product Security](../01-governance/product-security.md)、[Product Execution Registry Proposal](../appendices/product-execution-registry-proposal.md)、[Runtime ECS Design Closure Review](../appendices/runtime-ecs-design-closure-review.md)、[Architecture Plan Closure Review](../appendices/architecture-plan-closure-review.md)、[Architecture Governance](../01-governance/architecture-governance.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Runtime Performance／Capacity](../04-runtime/performance-capacity.md)、[Runtime ECS](../04-runtime/entity-component-system.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Persistence／Save](../04-runtime/persistence-save.md)、[Collision](../05-simulation/collision.md)、[Physics](../05-simulation/physics.md)、[Navigation](../05-simulation/navigation.md)、[Render Graph](../06-rendering/render-graph.md)、[Virtualized／Continuous Geometry](../06-rendering/virtualized-continuous-geometry.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-28

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

AIがGame制作時に理解する標準経路は、巨大なMarkdown、全Project dump、live native objectを直接読む方式ではなく、Ownerが正規状態から導出したbounded typed Projectionと登録済みsemantic toolである。AIへ渡せる制作文脈は、目的別に[Architecture Governance §5.3](../01-governance/architecture-governance.md#53-architecture-explain-projection)所有の`ArchitectureExplainProjectionV1`、`GameUnderstandingClosureV1`、`AiTaskContextCapsuleV1`、`EditorContextSnapshotV1`、`OptimizationDecisionProjectionV1`へ分割し、各Projectionはschema version、source revision、exact ref／hash、invalidation conditionを持つ。query型Projectionだけがomitted rangeとcursorを明示し、complete型Projectionは欠落Fieldを許さない。Project Sourceと設計文書の必要な抜粋はread-only inputとして利用できるが、Projectionを補完する権威、暗黙default、直接write対象にはしない。

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

current closureではimmutable `ArchitectureInventoryV1`、`ArchitectureExplainProjectionV1`、optimization説明用Eval Fixture／Receipt、該当`AiTaskContextCapsuleV1`、optimization explain／propose／select Operationが未materializeまたは未Activationである。したがって§4.2はAI制作経路の目標契約であって現在利用可能な機能一覧ではない。Architecture Inventory／Explain Projectionの状態と契約は[Architecture Governance §5](../01-governance/architecture-governance.md#5-inventoryとindex)、移行全体のreadinessは[Governance Migration Proposals §2.1](../appendices/governance-migration-proposals.md#21-current-readiness)、Capsule schemaとbinding条件は[AI Security／Approval §5](../01-governance/ai-security-approval.md#5-beginner-questionsassumptions理解条件)、optimization ProjectionとEval条件は[Performance／Capacity §8.4](../04-runtime/performance-capacity.md#84-algorithm-optimization-candidate-qualification)を正本とし、本書で件数またはActivationを上書きしない。

Target／ProfileごとのPrimary candidateは、qualified selectionが存在する場合にexact一件、存在しない場合は0件である。reference implementationはsemantic oracle、または明示的に別Qualificationされたsemantic fallbackになり得るが、旧経路と新経路の暗黙併載、deprecated reader、alias、silent fallback、runtime自動切替を正当化しない。benchmark candidateはdispatch不能である。選択を変える場合はprofile／algorithm revisionを新設し、public semanticsが変わる場合は新しいcontract versionとして扱う。Source dataは保持するが、旧挙動を温存する互換layerは作らない。

#### 4.2.1 AI-readable ECS／Memory alignment assessment

ECS／Memory最適化のAI可読性は、概念理解、current／target判別、機械的解決、実行可能性、安全境界を分けて評価する。target designではOwner、identity、storage、query、lease、Candidate、Evidence、selected／rejected理由が分離されており、概念理解と安全境界はstrongである。一方、current authorityはGameplay Programming Model revision 1とRuntime ECS revision 2のtarget Ownerに分かれ、`ArchitectureInventoryV1`、各Projection、Schema、validator、fixture Artifact、Receiptが未materializeであるため、機械的解決はincomplete、実行可能性はabsentである。AIがMarkdownを説明できることを、この不足の代用にしない。

cross-owner連携は次の一方向chainだけを使う。Architecture GovernanceはOwner／文書状態／依存の`ArchitectureExplainProjectionV1`、Memoryはpointer／allocation Contract fragment、Runtime ECSはstorage／query／structural semanticsの`RuntimeEcsContractGraphV1`、Performanceは評価済み候補の`OptimizationDecisionProjectionV1`、AI Securityは`AiTaskContextCapsuleV1`とauthorization、AI VerificationはEval／Receipt／freshnessを所有する。各Ownerは他OwnerのField、状態、権限、Evidenceを複写せず、同じProject lineage、source revision、Target、Contract Set、Toolchain、fixtureへexactに束縛されたread-only Projectionだけを接続する。

共通のAI read／explain／propose以後はauthority別に分岐する。Game Projectのauthoring mutationだけがProject State所有のactive Operationから`ProjectChangeSetV1`、preview／validate／approve／commit、new Project revisionへ進む。Engineのalgorithm／layout CandidateはProject ChangeSetへ変換せず、Performance／Artifact／Governance Ownerのqualification、selection、promotion、Contract revisionへ進む。Project authoring ReceiptをEngine最適化Evidenceへ、Optimization ReceiptをProject Commit authorityへ流用しない。

Runtime ECS固有の有名Engine比較、詳細な整合性判定、未解決Closureは[Runtime ECS Design Closure Review](../appendices/runtime-ecs-design-closure-review.md)をnavigation先とする。Architecture全体のAI可読性、Runtime coverage、Editor／Game分離、Target別Build、汎用Runtime Asset target authority、Build最適化とその他の未解決Closureは[Architecture Plan Closure Review](../appendices/architecture-plan-closure-review.md)をnavigation先とする。両Reviewはproposal appendixであり、Product Capability、Phase、Work Package、Runtime contractまたは外部Engine互換性を追加しない。本節も実装Task、日程、担当、Activation、採用済み最適化を生成しない。

### 4.3 Creative expressionとextension境界

Miraikanaiはゲーム内容の表現自由度と、Engine／Editorへ任意codeを注入する拡張自由度を別axisで扱う。前者を広く保ち、後者を安全性、移植性、再現性のためclosed public Capabilityへ限定する。片方の広さまたは狭さから他方を推測しない。

| axis | targetで許す表現 | 意図的な境界 |
|---|---|---|
| World構造 | 2D、3D、nonspatial、Scene 0件、再利用Scene、procedural、Tilemap、Blockout、continuous／finite composition | 固定`Level` hierarchy、root Scene default、Cellの永続identityを要求しない。`Level Workspace`はEditor presentationだけ |
| Gameplay | `GameplayDefinition`、Feature Pack、任意Genre Pack、Game Project、適格なProject C++23の組合せ | Genre、Player、Character、Weapon、Quest、StageをCore必須階層にしない。Runtime JIT、任意FFI、download codeは許可しない |
| Visual | 2D／3D Render、Material parameter／Graph、`typed_ir`／`bounded_hlsl`、Shading Model、declarative Technique | native graphics API、private Backend handle、未登録Frame StageをProject authorityにしない |
| Authoring | 手動、AI、CLI、MCPが同じSource、Projection、ChangeSet、Diff、Undo、Receiptを使う | screen coordinate、raw file置換、任意property setter、AI自己承認を制作能力にしない |
| Engine／Editor extension | 公開Capability、owner-typed Document、登録済みPanel／Operation、qualified Project Native Module／Shader | C1では任意binary plugin、Marketplace、public Editor Extension SDK、Engine private APIを提供しない |

Product claimとしての「表現可能」は、対象ゲームの意味、状態、入出力、World構造、Visual Styleを公開Capabilityとowner-typed Sourceで保持でき、Target差をSource分裂またはprivate escape hatchなしに解決できることを指す。見た目だけの近似、AIの自然言語説明、外部Tool内だけの再現、未登録pluginまたはEngine改造を成功扱いしない。

安全境界で要求を表現できない場合は`capability_unavailable`として明示し、別Genreの固定template、Level hierarchy、自由JSON、Project C++によるprivate API accessへ黙示fallbackしない。逆に、Capabilityが存在する領域へ特定Genre、Reference Game、Workspace layout、AI providerの都合による不要な制約を追加しない。

current RepositoryではOwner文書が`review`、CapabilityとOperationが未Activation、実装状態が`absent`である。したがって本節はtargetの自由度と制約を定義するが、現在利用可能な制作機能または完成度を主張しない。

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

Product acceptanceとして、Runtime Entry replacementはdestination全体が公開されるかsource branchとlast-valid generationが維持されるかの二択であり、partial World／UI／Stageを表示しない。Loading表示は進行状況のprojectionであって遷移成否またはdestination identityの正本ではない。transition state、eligible boundary、commit／cancel、generation、reverse teardownは[Scheduling／Lifetime §4.0](../04-runtime/scheduling-lifetime.md#40-runtime-entry-transition)、package staging／publicationは[Runtime Package §1.1](../04-runtime/runtime-package.md#11-runtime-entry-package-closure)だけが正本であり、本節のscenario表または文言から別Runtime state machineを生成しない。

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

Phase 5のProject Shader受入はWindows上で`bounded_hlsl`と`typed_ir`を一つ以上ずつ、同じCandidate／Project revisionのFirst Playable挙動に実使用して別々に閉じる。前者はSource隔離、Broker再計算Diff、Code Owner Approval、Target artifact／Receiptを、後者はcanonical IR identity、behavior／change-impact coverage、Target artifact／Receiptを必須にし、一方のmodeの成功を他方へ流用しない。これはProduct acceptanceの確定であり、実装順序、日程、担当を定めない。

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
| `critical_path` | Control Plane → ECS E0 → Headless Authoring → Render Graph core／D3D12 → Editor Runtime → Genre非依存Core holdout → compact 2D RPG → Authoring MVP-A。Phase 1のHeadless Authoring exitはPhase 2 exitより前に閉じる。閉包後はRender Graph core／D3D12と、Phase 2内のPlatform／UI統合を並行できる。Project Source ActivationとShooter technical qualificationは独立lane |
| `scope_reduction_order` | C2 advanced rendering → non-RPG Reference coverage → 3D technical Reference → mobile shipping。MVP-AのGenre非依存Core holdoutとRPG Reference contractは削除しない |
| `risk_owner` | `mirakan.arch.product-plan` |
| `review_cadence` | 各Phase exitで更新し、仮定を同じCandidateの実測Receiptへ差し替える |

`team_assumption_state=unfixed`では担当者名、完了日、同時実行lane数を出力しない。日程または予算の提示が必要になった時点で、team composition、利用可能runner／device、稼働制約、外部依存を入力として別の承認済みplanning revisionを作る。scope削減は上表の順序でProposal化し、Phase 4のMVP-A Requirement、Phase 5の独立Project Source Capability、security／package／support closureを暗黙に落とさない。

### 5.1.1 RPG-first MVPとcurrent Product Definitionの分離

RPG-firstは承認済みProduct target directionであるが、current Registry row、Installed Product composition、active Operation集合、Capability activation row、Work Package lifecycle headをこの文書編集だけで変更しない。現行Shooter IDをRPGへrenameし、Shooter ReceiptをRPG qualificationへ流用し、未定義のRPG Stable ID／Schema／Capability／Operation／Work Packageを仮登録することを禁止する。

RPG Product Definitionをmaterializeする前に、[Gameplay Feature Packs §4.4](../08-packs/gameplay-features.md#44-reusable-rpg-feature-family)のcommand battle、actor progression、inventory／equipment、dialogue／quest、currency／shopのReusable Owner境界、[RPG Genre Pack](../08-packs/rpg.md)のcomposition、Productが所有するReference Game outcome／acceptance binding、source／destination全rowをexact分類するProduct Definition Migrationを同じreview closureへ揃える。Migrationは一つのatomic destinationとして適用し、途中のRegistry状態、CoreへのGenre依存、dual current Product Definitionを作らない。それまではcurrent Shooter Registryをsource baselineとして保持し、RPG Capabilityを`not_activated`から成功可能な状態へ推測しない。

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
| 5 外部Agent接続とProject Source Activation | local MCP／conformance済みClientのProposal-only laneと、新規Native／Project Shader Source laneを分離 | 外部Clientの権限境界を独立に検証し、Windowsでは`bounded_hlsl`一件以上と`typed_ir`一件以上を同じCandidate／Project revisionへ別々のClosure／Receiptで閉じる |
| 6 3D Technical Reference MVP-B | Shooter Genre Packのcompact 3D arenaをtechnical Referenceとして検証 | 3D固有OwnerとFeature契約の非2D依存を実証するが、RPG MVPまたはGeneric Engine Releaseの前提にしない |
| 7 Mobile Platform | Android／Apple offline Target | Store／device／thermal／lifecycle GateをTarget別に通す |
| 8 Production CapabilityとPack | 一CapabilityずつC2へ昇格 | Authoring、diagnostic、fallback、Qualification closureを満たす |
| 9 Runtime生成deny-only境界 | Runtimeからの未許可generation／mutationを拒否し、Project／Save／authoritative Worldを不変に保つ | allowlist、署名baseline、authority、quota、network禁止のnegative fixtureを通す。positive Runtime生成はFuture `planning_only`に留める |

各Phaseは、契約固定、最小fixture、vertical slice、実測、上位bottleneck改善、同一条件の回帰、Capability昇格の順で閉じる。不合格時はSource intentとlast-known-goodを維持し、OptimizationRequiredを状態ではなくDiagnosticとして発行する。Activation stateはcandidate_lockedに留め、利用可能なCandidateを維持できない場合だけnot_activatedへ降格する。

現行Product DefinitionではPhase 5のProject Shaderを`bounded_hlsl`と`typed_ir`の二つの必須sub-closureとして扱う。段階的提供へ変更する場合は、Capability、Requirement、Work Package、Gate、Claimを別Product Definition revisionで分離し、同じGateを片方だけの成功へ読み替えない。

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

Security incidentは[Product Security](../01-governance/product-security.md)のcase／incident closureへ接続し、重大回帰、license／Provider停止、Target非適合、Evidence失効を検出した各domain Ownerと共同で新規選択、AI active use、Promotion、Package収録を停止する。現在状態、影響Target、利用Project、last-known-good、rollback／migration、再Qualification条件を記録し、無言のfallbackやGameplay意味変更を行わない。

### 7.3 Product completion gate

Product milestoneは、機能一覧のcheckだけでは完了しない。

- 対象Capability closureが要求されたActivation stateにある。
- clean build／install／launch、offline play、Save／Load、failure diagnosis、rollbackが再現する。
- Platform、Accessibility、Localization、Privacy、License、SBOM、Support、Known limitationsが閉じる。
- AIと手動編集が同じProject history、Diff、Approval、Undo、Replayを使う。
- 未対応Capability、未実測Target、失効Evidenceを明示し、成功表示しない。
- [AI Verification／Provenance](../01-governance/ai-verification-provenance.md)のRelease evidence closureが同一Candidateへ結び付く。
- [Product Lifecycle](product-lifecycle.md)のbootstrap、Template／Sample、Documentation、GUI／CLI／headless parity、update／repair／support／NOTICE acceptanceが同じEngine release bindingとCandidateへ閉じる。
- [Product Security](../01-governance/product-security.md)の`ProductSecurityReleaseBindingV1`がcurrent baseline、threat ownership、fresh assessment、support windowとtyped `SecurityCaseRegistrySnapshotV1`を同じReleaseへ閉じる。Snapshotは全Case head、空のopen-critical set、security update／disclosure／embargo／incident head、required notification Receiptをexact setとして持ち、generic Evidenceだけでcompletionを主張しない。

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

`planning_state`の初期値は全26行`planning_only`である。`required_decision_kinds[]`は`product_scope | authority_model | data_model | threat_model | target_program | provider_selection | licensing | operations | qualification`、`candidate_target_kinds[]`は`headless_server | desktop | mobile | console | web | xr | distributed_cluster`、`qualification_fixture_kinds[]`は`contract | deterministic_simulation | network_fault | scale | target_device | content_quality | security | operations`のclosed値だけを受理する。

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
| `future.capability.virtualized-continuous-geometry-lod` | `mirakan.arch.rendering-virtualized-continuous-geometry` | `planning_only` | `capability.rendering.render-graph-core; capability.product.general_production_3d` | `[]` | `product_scope; data_model; provider_selection; licensing; target_program; operations; qualification` | `desktop; mobile; console` | `contract; content_quality; scale; target_device; operations` | discrete Mesh LOD／HLODのPlan hashとexact fallbackがqualifiedになり、[Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md)のtarget Owner選択後にexact Definition／Port、Capability、Target qualificationがappliedとなった後、Virtualized／Continuous Geometry OwnerがLOD、Asset Lifecycle、Runtime Asset Lifecycle、Runtime Package、World、Camera、Render Graph、Scheduling／Lifetime、Memory／Pointers、Performance／Capacity、Materials、Animation、Persistence、Debug、Toolchain／Platformのreceipt-free closureを束ね、feature tupleごとのTarget Receipt／active Qualification Binding、provider／license Decision、root／page residency、overflow／fault／device-loss fallback Receiptまで承認する。Product `active`移行とTarget Activation Binding publicationを同一Governance transactionでatomicに行い、active Target／Qualification Binding／Operation集合のいずれかが空なら`planning_only`を維持する | `claim.product.feature.continuous-geometry-lod; claim.product.feature.virtualized-geometry` |
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
| `future.capability.unrestricted-project-scripting-runtime` | `mirakan.arch.gameplay-programming-model` | `planning_only` | `capability.authoring.ai-core; capability.product.project-source-activation; capability.runtime.debug-replay-support` | `[]` | `product_scope; authority_model; data_model; threat_model; provider_selection; licensing; operations; qualification` | `desktop; mobile` | `contract; deterministic_simulation; target_device; security; operations` | 本Entryを直接active移行しない。structured content MOD、sandboxed executable MOD、signed AOT desktop native extension、developer-only executable code（JIT／未署名）を別Future ID／Owner／Target／claimへ分解するProduct Decisionが先に成立し、各分岐のVM／ABI、sandbox、determinism、hot reload、debug、package、mod trust、Target／Store policy、adversarial／replay Receiptが独立して閉じる | `claim.product.feature.unrestricted-scripting; claim.product.feature.arbitrary-runtime-code; claim.product.feature.mod-execution` |

Android／AppleのPlatform文書にProject ShaderのTarget artifact／package schemaが存在しても、`capability.project.shader`のProject-defined Sourceは有効化されない。現行Registryでは両Targetを`excluded`とし、`future.capability.mobile-project-native-shader-source-qualification`を分解・承認してTarget bindingとfresh Qualificationを追加するまで、Engine-ownedまたは事前Qualification済みartifactのpackaging境界だけを参照する。

`future.capability.unrestricted-project-scripting-runtime`は複数の異なるtrust boundaryを発見するためのincubation umbrellaであり、Candidate、active Capability、Work Packageへ直接昇格させない。分解時は実行payload classとして次の四scopeを相互排他にする。(1) Asset、Pack、GameplayDefinition等のstructured content MODは実行codeを含まず、既存Schema／Capability／authority／quota内だけでCookする。(2) sandboxed executable MODは選定済みVM／bytecode、closed Host API、memory／CPU／storage／network budget、determinism、Save／Replay、署名またはcontent trust policyを持ち、native process権限を得ない。(3) signed AOT desktop native extensionはTarget別ABI、publisher trust、Package固定のstatic linkまたはstartup dynamic binding、revocation、crash isolationを閉じ、署名またはhash一致だけでEngine private API、post-package download、mobile対応を許可しない。(4) developer-only executable codeはJITまたは未署名codeを隔離Development Profileだけで扱い、Shipping package、Release claim、MOD配布経路へ流用しない。一つの配布物が複数classを含む場合もpayloadごとに別Capabilityとして判定し、弱いclassのpolicyへ統合しない。

Current Productは上記四scopeをShipping後のMOD／extension／JIT配布Capabilityとしてすべて未対応とする。これはProject開発時にAuthoringしShipping packageへCookしたfirst-party Asset／Pack／GameplayDefinitionを禁止する意味ではない。[Gameplay Programming Model](../03-authoring/gameplay-programming-model.md)のno runtime code generation／download、[Native Game Module](../03-authoring/native-game-module.md)のShipping static link、[Project Shader](../06-rendering/project-shader.md)のoffline compileを維持する。Apple App Review 2.5.2はfeatureを変更するdownload／install／execute codeを原則禁止し、Apple Hardened RuntimeはJITをentitlementなしで禁止する。Androidもdynamic code loadingを高リスクとし、Windowsは未署名public distributionを推奨しないため、一つのcross-platform「unrestricted runtime」へ統合しない。[Apple App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)、[Apple JIT entitlement](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.security.cs.allow-jit)、[Android Dynamic Code Loading](https://developer.android.com/privacy-and-security/risks/dynamic-code-loading)、[Windows code signing options](https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/code-signing-options)

分解Product Decisionは新しいFuture Portfolio revisionでumbrella行をremoveし、各新規行とclaim対応を同じrow migration manifestへ明示する。umbrella IDをactive Capability alias、fallback、共通Operation familyとして残さず、四scopeの一つがQualificationされても他scopeのclaimをreleaseしない。

`excluded_current_product_claims[]`と`released_claims[]`は自由文字列でなく、同じFuture closureに含む次の`ProductClaimDefinitionRegistryV1`のexact `claim_id`だけを参照する。Registryのclaim ID集合は26件のFuture行が列挙する`excluded_current_product_claims[]`のunionとset equalityでなければならない。`claim_scope`は`feature | network | platform | operations | ai | quality | migration`のclosed値である。MVP-A、MVP-B、Technology Preview等のmilestoneはActive ProductのPhase／Release Gateだけが所有し、Future claimへ重複登録しない。canonical labelの変更はclaim identityを変えず、ID再利用、未登録label、別言語表示からのID推測を拒否する。

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
| `claim.product.feature.continuous-geometry-lod` | continuous geometry LOD | `feature` |
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
| `claim.product.feature.virtualized-geometry` | virtualized geometry | `feature` |
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

- [Runtime ECS](../04-runtime/entity-component-system.md)
- [Scheduling／lifetime](../04-runtime/scheduling-lifetime.md)
- [Runtime Package](../04-runtime/runtime-package.md)
- [Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md)
- [Persistence／Save](../04-runtime/persistence-save.md)
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
- [Virtualized／continuous geometry](../06-rendering/virtualized-continuous-geometry.md)
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
- [RPG Genre Pack](../08-packs/rpg.md)

RPG-first targetは、Generic Engine Core、command battle／actor progression／inventory・equipment／dialogue・quest／currency・shopを所有する[Reusable RPG Feature family](../08-packs/gameplay-features.md#44-reusable-rpg-feature-family)、composition／RPG profile／game flow／command role mapping／Reference fixture bindingを所有する[RPG Genre Pack](../08-packs/rpg.md)、original content／balance／World composition／localized presentationを持つ通常のGame ProjectとしてのRPG Reference Gameの四層に分ける。Product PlanはReference GameのProduct outcome、acceptance、Evidence非代替を所有し、Reference contentの別Production Ownerにはならない。既存Combat、Interaction、Scenario／Stage、Pickup／Grant、UI、Localization、Save／Replay、Runtime契約はcurrent semanticsが適合する場合だけ再利用し、Shooter契約を暗黙拡張せずGenre意味をCoreへ移さない。これらはtarget Owner設計のmaterializationであって、Stable ID、Schema Field、Registry、Capability、Operation、Work Package、current Product DefinitionまたはActivationを追加しない。

### 9.5 Foundation／Governance

- [Product Lifecycle](product-lifecycle.md)
- [Core architecture](../02-foundation/core-architecture.md)
- [Toolchain／dependencies](../02-foundation/toolchain-dependencies.md)
- [Executable contracts](../02-foundation/executable-contracts.md)
- [Naming／Project layout](../02-foundation/naming-project-layout.md)
- [C++23 Modules](../02-foundation/cpp23-modules.md)
- [Math core](../02-foundation/math-core.md)
- [Memory／pointers](../02-foundation/memory-pointers.md)
- [AI Security／Approval](../01-governance/ai-security-approval.md)
- [AI Verification／Provenance](../01-governance/ai-verification-provenance.md)
- [Product Security](../01-governance/product-security.md)

Primary Product Evidenceは、MVP-AのRPG First PlayableとGenre非依存Core holdout、MVP-Bの独立3D technical Reference、clean package／install／offline run、AIと手動編集の往復、Save／Load、Undo／Replay、Target別Qualification、failure／rollback、Release closureである。MVP-BまたはShooter EvidenceをMVP-A、RPG、Genre非依存Coreの代用にしない。EvidenceのRecord形式、保持、Trace、SBOM、ProvenanceはVerification Ownerだけが決定する。

## 10. 外部比較の使用範囲

Unreal Engine、Unity、Godotの公式資料はCoverageと責務分離の比較Evidenceにだけ使う。MiraikanaiのAPI、Class hierarchy、Scene、Project形式、Editor UX、既定値の根拠にはしない。比較対象は市場順位の網羅表ではなく、(1) Editor統合AIと段階的permission／Undoを公式公開する商用Engine、(2) reflection／registryから型付きEditor toolを公開する商用Engine、(3) textで差分可能なScene／Resourceと拡張Editor APIを公式公開するopen-source Engine、の三つの制作境界を最小集合で比較できるため選んだ。Engine数、人気、機能数、benchmark順位を設計Evidenceにせず、比較軸を満たさない追加Engineを列挙目的で増やさない。

この比較は非網羅であり、各製品の最新major Editor、AI surface、serialization／reflection APIの公式資料をProduct判断時に再読込する。比較対象の名称または機能が変わってもMiraikanaiの契約を自動変更せず、公式資料で同じ比較軸を再検証し、採用原則が変わる場合だけArchitecture ChangeSetとして扱う。

| Engine | 公式に確認した制作境界 | Miraikanaiへ採る原則 | 採らないもの |
|---|---|---|---|
| Unity 6.3 LTS | Unity AI Open BetaはEditor内のProject文脈を使うAsk／Plan／Agent、permission level、変更確認／Undoを提示する。Editor data変更は`SerializedObject`がdirty、Undo、Prefab overrideを扱う公式経路である | liveだがboundedなsemantic context、段階的権限、preview／Undo、Editor-owned mutation経路 | AIによるraw Asset／Scene file直接編集、UI上のpermission表示だけをAuthorityとすること |
| Unreal Engine 5.8 | Experimental Unreal MCPはEditor内のToolset Registryから型付きtoolを公開し、Actor／Component／Assetはreflection／Asset Registry／Editor APIで扱う。Python公式資料もOS file APIによるAsset操作を禁止する | reflection-backed projection、登録tool、Editor API、commit後read-back | 任意property setter、汎用Python／console／file edit、localhostであることを認証や承認の代用にすること |
| Godot 4.7.1 | Node／Scene／Resourceとtext形式`.tscn`、`EditorPlugin`／`ClassDB`により構造が観測可能で差分化しやすい。確認した公式資料にはUnity AI／Unreal MCP相当の内蔵AI authoring authorityはない | 透明でdiff可能なcanonical source、型／UID／NodePathを保つEditor-owned validation | textであることを安全性の証明にすること、subresource／順序／UID制約を無視した直接置換 |

Rendering abstractionはAI Authoringとは別の比較軸である。同じ三Engineの公式資料から、上位render graph／pipelineと低位graphics abstraction／driverを分離する原則だけを採る。

| Engine | 公式に確認したRendering境界 | Miraikanaiでの対応／差分 |
|---|---|---|
| Unreal Engine 5.8 | RHIはplatform graphics API直上の薄い低位層で、RDGがpass／resource dependency、lifetime、barrier、parallel executionを上位で管理する | `GraphicsDevicePort`＋private Backend AdapterをRHI相当境界、`CanonicalRenderExecutionPlanV1`を上位Graph正本にする。Backendによるlogical pass追加とProject callbackからのnative commandを許さない点はさらに狭い |
| Unity 6.3 LTS | SRP Coreがplatform graphics APIを扱う共通部品を提供し、URP Render Graphではpass入力／出力を宣言してcommand生成を実行関数へ分ける | 再利用可能なpipeline／backend分離とresource宣言を採る。Project scripting APIをGraphics Device境界にせず、Qualification済みPass Templateへ閉じる |
| Godot 4.7.1 | Renderer methodの下にVulkan／D3D12／Metalを抽象化する`RenderingDevice`があり、rendererとgraphics driverを分離する | Engine handleとprivate driverの分離を採る。Renderer profile、Target Profile、Capability Signatureを別契約にし、Backend名から品質profileを推測しない |

Miraikanaiの正本構造は[Render Graph §2.1](../06-rendering/render-graph.md#21-rhi相当境界)であり、外部EngineのClass名、RHI method、command list、feature-level値を互換APIとして導入しない。

scene／screen切替についても公式資料から次の責務分離だけを採用判断へ使う。

| Engine | 公式に確認した切替モデル | Miraikanaiでの対応／差分 |
|---|---|---|
| Unity 6.3 LTS | `SceneManager`がSceneをSingle／Additiveでloadし、Sceneを越えて保持するObjectは`DontDestroyOnLoad`で明示する | top-level切替と永続Sessionを分離する原則は採る。一方、Scene AssetをTitle／Stage／UI／Worldの共通authorityにはせず、Runtime Entry、Stage、World、UI Screenをtyped Owner契約へ分ける |
| Unreal Engine 5.8 | Gameplay FrameworkはGameInstanceをMap load間で保持し、World／LevelをTravel／Streamingする。Common UIはActivatable Widget Stackを持つ | persistent Play Session＋branch travel＋UI Stackの三分離に最も近い。Miraikanaiはさらにexact ref／hash、T00 atomic publish、source保持failureを共通Runtime契約にする |
| Godot 4.7.1 | `SceneTree`のcurrent sceneを変更し、global stateやscene switching helperはAutoloadで保持できる | 小規模Projectの単純性は参考にするが、global singletonへdestination、Save、Stage、UI authorityを集約せず、registered PortとOwner validatorを必須にする |

三Engineとも画面切替、永続state、UI navigationを完全に同一Objectへ畳み込むことを要求していない。したがってMiraikanaiは単一`SceneManager`を追加せず、§5.0.1の`Runtime Entry replacement + UI Screen Stack + Stage transition + World spatial transition`を正規分解とする。

- [Unity AI: Ask／Plan／Agent](https://unity.com/blog/unity-ai-assistant-ask-plan-agent-mode-explained)
- [Unity AI: Get started](https://unity.com/blog/unity-ai-how-to-get-started)
- [Unity 6 release support](https://unity.com/releases/unity-6/support)
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
- [Godot 4.7 `.tscn` format](https://docs.godotengine.org/en/4.7/engine_details/file_formats/tscn.html)
- [Godot Editor plugins](https://docs.godotengine.org/en/stable/tutorials/plugins/editor/making_plugins.html)
- [Godot release archive](https://godotengine.org/download/archive/)
- [Godot 4.7 `SceneTree`](https://docs.godotengine.org/en/4.7/classes/class_scenetree.html)
- [Godot 4.7 Autoload](https://docs.godotengine.org/en/4.7/tutorials/scripting/singletons_autoload.html)
- [Unreal Engine 5.8 Modules](https://dev.epicgames.com/documentation/unreal-engine/unreal-engine-modules?lang=en-US)
- [Unity 6 Native plug-ins](https://docs.unity3d.com/6000.0/Documentation/Manual/plug-ins-native.html)
- [Godot 4.7 GDExtension](https://docs.godotengine.org/en/4.7/tutorials/scripting/gdextension/what_is_gdextension.html)
- [Unreal Engine 5.8 Graphics Programming Overview／RHI](https://dev.epicgames.com/documentation/unreal-engine/graphics-programming-overview-for-unreal-engine)
- [Unreal Engine 5.8 Render Dependency Graph](https://dev.epicgames.com/documentation/en-us/unreal-engine/render-dependency-graph-in-unreal-engine)
- [Unity 6 Scriptable Render Pipeline Core](https://docs.unity3d.com/ja/6000.0/Manual/com.unity.render-pipelines.core.html)
- [Unity 6 URP Render Graph](https://docs.unity3d.com/ja/current/Manual/urp/render-graph-write-render-pass.html)
- [Godot 4.7 Internal rendering architecture](https://docs.godotengine.org/en/4.7/engine_details/architecture/internal_rendering_architecture.html)
- [Unity Hub／Editor language selection](https://docs.unity.com/en-us/hub/add-editor-language)
- [Unreal Engine Localization Overview](https://dev.epicgames.com/documentation/en-us/unreal-engine/localization-overview-for-unreal-engine)
- [Godot EditorSettings language／property-name localization](https://docs.godotengine.org/en/latest/classes/class_editorsettings.html)

外部資料の時点依存情報はArchitecture定数にせず、更新判断とEvidence freshnessを[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)へ委ねる。

## 11. Product execution registries

詳細は[product-execution-registry-proposal](../appendices/product-execution-registry-proposal.md#11-product-execution-registries)へ分離した。本節はnavigationだけを持ち、Catalog／Fixture定義を複写しない。
