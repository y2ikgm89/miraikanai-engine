# AI-native Production Loop Architecture Reconstruction Design

- 文書状態: proposal
- 実装状態: absent
- 正本性: non-normative design input
- 対象: Miraikanai Engine Architecture Owner群のクリーン再構築
- 非対象: C++実装、MCD Schema file、Registry、Fixture artifact、Receipt、Build、CI、工程、工数、担当
- 外部根拠確認日: 2026-08-03

## 1. 目的

現在のArchitectureが持つ一意Owner、規範依存DAG、fail-closed、typed Operation、Project atomic commit、Runtime package、Product release closureを維持しながら、次の重大gapをtarget design上で閉じる。

1. 自然言語Promptから確認済みGame Brief、GameSpec、Requirement、Task Contextへ至る正本経路がない。
2. Playtestの定性的観測を次のChangeSetへ戻す追跡可能な制作loopがない。
3. AI生成範囲が構成、Asset、Native／Shaderで区別されず、Product claimを誤読できる。
4. compact 2D command RPG First Playable outcomeにexact execution／acceptance projectionがない。
5. Evidence freshness四状態の正本規則と参照先がない。
6. Minimum Executable Coreを構成する必須component、boot input、acceptance closureが一つのtarget contractになっていない。
7. `GameSystemDependencyGraphV1`等、current Ownerへ解決しないexact-looking type参照が残る。
8. stale Markdown fragment、存在しない節参照、Closure Reviewの節順が残る。

文書を増やしただけで実装、materialization、Qualification、Activation、ReleaseまたはProduct completionを主張しない。

## 2. 外部標準とRepository様式

[ISO/IEC/IEEE 42010:2022](https://www.iso.org/standard/74393.html)はArchitecture Descriptionの構造と表現、conceptとrelationship、viewpoint／model kind等の要求を定めるが、Architectureを作るprocess、method、notation、tool、記録format／mediaを規定しない。したがって、外部に唯一の公式Markdown様式があるとは扱わない。

Miraikanaiでは[Architecture Governance](../../architecture/01-governance/architecture-governance.md)のOwner header、一意所有、状態、根拠区分、規範依存DAG、ADR、IndexをArchitecture Description Frameworkとして維持する。外部標準の名称だけを適合宣言にせず、本再構築ではstakeholder concernをProduct outcome、AI production loop、Core executability、Security／Evidence、Target／Release closureへ明示的に対応させる。

## 3. 採択する再構築方式

既存63 Ownerを全面置換せず、責務が欠けているGame production loopだけを新しいAuthoring Ownerとして追加する。既存Ownerへ分散したprovisional shapeを新Ownerへ移し、各既存Ownerは自身の権限、transaction、test、runtime、Product acceptanceだけを保持する。

この方式を採る理由は次である。

- 現在の309規範依存edgeにmissing／cycleがなく、Foundation、Runtime、Rendering、Platform、Product releaseの大部分は既に一意所有されている。
- AI SecurityへGameSpec payload意味を追加するとGovernance authorityとAuthoring meaningが混在する。
- Project StateへPlaytest／creative evaluationを追加するとgeneric transaction OwnerがGame production意味を所有してしまう。
- 全Ownerの全面書換えは閉じている契約まで移動し、正本重複と参照破損を増やす。
- current Repositoryにmaterialized Schema／consumerがないため、新Ownerのinitial V1を直接採用し、旧candidate Schema、alias、dual reader、migration Operationを残さないclean breakが可能である。

## 4. 新しい正本Owner

`docs/architecture/03-authoring/game-production-loop.md`を追加し、文書IDを`mirakan.arch.game-production-loop`とする。

### 4.1 正本範囲

- Game intent sessionとbootstrap／existing Project subject
- Game Intent Draft、Game Brief、GameSpec
- Question、Answer、Assumption、Decisionとsupersession
- Requirement traceabilityとGame understanding closure
- Experience Goal、Playtest Observation、Experience Evaluation
- Iteration DecisionとGame production loop closure
- 上記を操作するtarget Operation familyとcurrent `not_activated`境界
- AI composition、AI external content、AI Project Sourceの生成lane区分

### 4.2 非正本範囲

- Task Authorization、Consent、Risk、Approval、Code Owner、Provider trust: AI Security
- Evidence wrapper、signature、freshness、revocation: AI Verification
- Project identity、Document header、ChangeSet、Commit、bootstrap publication: Project State
- automated test case／runner／result: Developer Testing
- Gameplay System／State／Capabilityのdomain意味: Gameplay Programming Modelと各Domain Owner
- Asset provenance／license／safety: Asset Lifecycle
- Build／Cook／Package／Launch: Core、Runtime Package、Platform
- Product claim、First Playable outcome、Release／Completion: Product PlanとProduct Lifecycle

### 4.3 規範依存

新OwnerはArchitecture Governance、Product Plan、AI Security／Approval、AI Verification／Provenance、Executable Contracts、Project State、Developer Testingへ依存する。既存上位Ownerから新Ownerへの逆向き規範依存は作らない。上位Ownerはgeneric authority／Evidence／transaction contractを所有し、新Ownerがそれらへ適合する。

## 5. Game intentと理解closure

### 5.1 Subjectとsession

`GameProductionSubjectV1`はclosed tagged unionとする。

- `bootstrap`: `project_ref`、exact `bootstrap_request_ref`、exact `bootstrap_profile_ref`を持ち、公開Project revisionはまだ存在しない。
- `existing_project`: exact `ProjectRevisionRefV1`を持つ。

Project bootstrapは公開revision前にstable `ProjectRefV1`を割り当てるが、成功したatomic publicationまでProject aggregate、revision 1、openable partial Projectを公開しない。sessionは一つのsubject、Task lineage、開始時Contract set、Target selection intentへ固定し、bootstrapとexisting Projectを相互変換しない。

### 5.2 Canonical record family

initial V1は次を一意に所有する。

- `GameIntentSessionV1／RefV1`
- `GameIntentDraftV1／RefV1`
- `GameBriefDocumentV1／RefV1`
- `GameQuestionRecordV1／RefV1`
- `GameAssumptionRecordV1／RefV1`
- `GameDecisionRecordV1／RefV1`
- `GameSpecDocumentV1／RefV1`
- `GameRequirementTraceabilityV1／RefV1`
- `GameUnderstandingClosureV1／RefV1`

Draftはoriginal User text hash、locale、subject、Task、bounded attachment／external input refsを保存するEvidence inputで、Project authorityではない。BriefとSpecはProject Stateの共通Document headerを使うAuthoring Documentである。Question、Assumption、Decisionはimmutable revision chainを持ち、同一logical subjectのcurrent headをexpected-previous CASで進める。

Briefはplayer、experience goal、game loop、scope、Target intent、input、save、online、content／rights、accessibility、locale、performance class、known exclusionをclosed fieldとして持つ。SpecはRequirement、System intent、content set、experience goal、test intent、budget intent、style lock、Target bindingをexact refへ分離し、自由proseだけで必須項目を満たさない。

### 5.3 Questionとassumption

Question impactは`blocking | high | medium | low`、resolutionは`open | answered | withdrawn`とする。Blocking／Highはansweredまたはwithdrawn Decisionなしに理解closureへ進めない。Mediumだけが有限期限、typed default、根拠、再検証条件を持つexact一Assumptionで一時的に閉じられる。LowはAIが勝手に無視せず、Decisionでaccept、deferまたはwithdrawする。

`GameUnderstandingClosureV1`はBrief、Spec、全Question current head、active Assumption、Decision、Requirement traceability、System dependency graph、State owner projection、Capability scope、Save／Replay contract set、Test Plan、Evidence admissibilityを同一subject、Project lineage、Contract set、Target集合へ閉じる。全required refを解決し、Blocking／High open 0、unaccepted Medium open 0、Requirement traceability 100%、State owner collision 0、unsupported required Capability 0、stale Evidence 0の場合だけ`ready_to_stage`とする。

## 6. Playtestとcreative iteration

initial V1は次を一意に所有する。

- `GameExperienceGoalSetV1／RefV1`
- `PlaytestSessionDefinitionV1／RefV1`
- `PlaytestObservationSetV1／RefV1`
- `GameExperienceEvaluationV1／RefV1`
- `GameIterationDecisionV1／RefV1`
- `GameProductionLoopClosureV1／RefV1`

Experience Goalは対象player、intended experience、observable indicator、failure signal、priority、applicable Runtime Entry／scenario／Targetを持つ。面白さをCompilerまたはAIが自動合格させる数値へ縮約しない。

Playtest Sessionはexact Game Candidate、Project revision、Runtime Entry、Target、Build／Package／Launch receipt、input／scenario、participant class、consent applicabilityへ固定する。自動`play_scenario`結果はDeveloper Testingのexact Result refとして取り込めるが、人間のObservationを代用しない。

Observationはobserver、timestamp、Experience Goal、reproduction context、observation kind、severity、bounded prose／measurement／evidence refsを持つ。Evaluationは全required GoalとObservation projectionをset equalityにし、`met | partially_met | not_met | not_evaluated`をGoalごとに決定する。`not_evaluated`をpassへ読み替えない。

Iteration Decisionは`accept_candidate | revise | stop | defer`のclosed branchで、`revise`は解決対象Goal／Observation、Requirement delta、許可されたproposal scopeを必須にする。AIの次Taskはexact Iteration Decisionと新Authorizationを入力にし、会話要約または「もっと面白く」だけから旧権限を再利用しない。

Production Loop ClosureはUnderstanding Closure、Candidate、Validation／Test closure、Experience Evaluation、Human Gameplay Approval、Iteration Decisionを束縛する。`accept_candidate`かつ全required technical Gate passかつ有効Human Gameplay Approvalの場合だけplayable acceptanceへ進める。

## 7. AI生成claimの三lane

Product Planと新Ownerは次を別Capability／claimとして扱う。

1. `ai_composed_game`: Gameplay Definition、Project Document、prequalified Pack、`domain_pack_reference | user_provided` Assetを組成する。Beginner First Playableのrequired lane。
2. `ai_generated_external_content`: qualified generation Providerによる画像、音声、3D等をAsset Lifecycleのprovenance、rights、safety、human reviewへ閉じる。MVP-A／Bの成立条件にしない。
3. `ai_generated_project_source`: Native C++／Project Shader Sourceを隔離生成し、Code Owner Assignment、independent Review、Build／Test、Promotionを必須にする。Beginner First Playableのrequired laneにしない。

一laneの成功を他laneのsupport claimへ使わない。「AIがゲームを生成」はclaim scopeにlane集合を必須表示し、`ai_composed_game`だけをoriginal Asset／Native Source生成済みと表現しない。

## 8. First Playable definition

Product Planへreceipt-free `FirstPlayableDefinitionV1`を追加する。initial Product-facing Definitionはexact一件のcompact 2D command RPGで、次を必須にする。

- Title、Map exploration、conversation、command battle、inventory／progress、Save／Load、Settings、Result
- `en-US`と`ja-JP`
- keyboard／controllerのrequired Action、input remap、accessibility minimum
- 2D rendering、UI／Text、Audio、Save、Gameplay Definition、RPG Feature／Genre Packのexact Capability／Pack requirement
- signed Engine／Editor／SDK candidateからbootstrapし、same Project revision系列でauthor、validate、test、package、clean install、launch、offline completion
- `ai_composed_game` journeyとmanual continuity
- Experience Goal、Playtest Evaluation、Human Gameplay Approval
- exact required Target、Runtime Entry、scenario、Requirement、Operation family、Evidence class projection

Shooter Fixture、3D technical Reference、別Genre、別Project、manual-only journeyをこのDefinitionのEvidenceへ使用しない。RPG Fixture／Schema／ReceiptがmaterializeするまでDefinitionはtarget requirementであり、First Playable completionを主張しない。

## 9. Minimum Executable Core closure

Core Architectureへreceipt-free `MinimumExecutableCoreDefinitionV1／RefV1`を追加する。initial closureは次のexact memberを要求する。

- Foundation／Math／Memory policy
- MCD Contract set、Type／Operation／Diagnostic resolver
- Project State readとimmutable Project revision input
- Runtime Entry Package reader、integrity validator、loader staging
- Runtime Orchestrator、Scheduler、Jobs、Runtime ECS、World-less headless branch
- GameHost、WorkerHost、private Platform Adapter、bounded logging／crash evidence
- deterministic clock／seed、typed command／event、fault transition
- public contract／Toolchain lock／Target Profile closure

`MinimumExecutableCoreQualificationV1`はclean configure／build、contract round-trip、headless Runtime Entry load、fixed-step advances、deterministic state hash、cancel／fault、shutdown、leak／sanitizer、wrong Contract／Target／Package negative fixtureを同一Candidateへ束縛する。Editor、Renderer、Physics、Audio、NetworkまたはRPG成功をminimum headless Core bootの代用にせず、headless Core bootだけをProduct First Playableへ読み替えない。

current RepositoryではDefinition、Schema、C++、CMake、Toolchain lock、Fixture、Receiptがabsentであり、stateは`not_materialized`とする。

## 10. Type ownership closure

Gameplay Programming Modelは`GameSystemDependencyGraphV1／RefV1`、`SystemImplementationSetV1／RefV1`、`GameStateOwnerProjectionV1／RefV1`を所有する。GraphはSystem contract、dependency edge、phase、State read／write、command／event、implementation selectionを同一Project revision／Contract setへ閉じ、cycle、複数active State owner、missing implementationを拒否する。

新Game Understanding Closureはこれらのexact Refを消費し、独自System graph／State owner shapeを持たない。Scheduling、Native Module、Runtime Packageは同じGraph ref／hashをbyte equalityで消費する。

Architecture Owner本文に現れるexact type／ref tokenは、次のexact一classへ分類できなければならない。

- 当該Ownerで定義
- direct canonical Owner link付き参照
- non-normative candidateとして明示しcurrent集合から除外
- external official typeとしてversion付き一次資料へ参照

unowned、複数Owner、同名別shape、display nameだけ、appendixからのshadow definitionを許可しない。

## 11. Evidence freshness

AI Verification Ownerは`EvidenceEffectiveStateV1`を`fresh | expired | revoked | invalid`のclosed enumとして所有する。

導出優先順位は`invalid > revoked > expired > fresh`とする。

- `invalid`: schema、signature、purpose、subject、context、hash、required Evidence集合、Trust chainのいずれかが不成立。
- `revoked`: record、subject、issuer、Role、Key、Policyまたはtransitive Evidenceのcurrent revocationが成立。
- `expired`: validかつnon-revokedだがevaluation timeが`expires_at`以上、またはfreshness policyのgeneration／environment条件外。
- `fresh`: valid、non-revoked、期限内で、required current contextとbyte equality。

一件が複数条件に該当する場合は上記優先順位だけを使う。`stale`は独立第五状態にせず、原因がcontext／generation mismatchなら`invalid`、time policy超過なら`expired`へ分類し、Diagnosticで原因を保持する。consumerは`fresh`だけをpositive Evidenceへ数える。

## 12. Operation boundary

新Ownerのtarget Operation familyは次の3 familyへ分ける。

- `game_intent_understanding`: session create、draft capture、question answer／withdraw、assumption accept／replace、brief confirm、spec publish、understanding close
- `game_experience_iteration`: playtest observation record、experience evaluate、iteration decide
- `game_production_read`: bounded inspect、trace、explain

各state-changing OperationはProject Stateのprepared candidate、expected revision、AI Security Authorization／Approval、atomic Commitを使用する。read／explainはmutation Receiptを発行しない。Human Gameplay Approval、Project Commit、Source PromotionはProvider／MCPへ公開しない。

現Repositoryでは全MCD、Owner Manifest、Service allowlist、Policy、Validator、Diagnostic、Receipt type、Provider／MCP／CLI／Editor projection、Activation Evidence集合をexact `[]`、family stateを`not_activated`とする。Operation名の記載だけでdispatch可能と扱わない。

## 13. 既存文書の再配置

- AI Security Assumptions Guide: candidate shapeを削除し、新Ownerの型を使う質問／assumption／security fixture guideへ縮小する。
- AI Security Owner: Game meaningを削除し、Task Capsule、Authorization、Risk、Approval、Code Owner、Consentだけを保持する。
- Project State: Game Brief／Spec等を正規Document kindとして新Ownerへ解決し、Document payloadを複製しない。
- Editor Workspace: beginner workflowとPlaytest Resultを新Ownerのrecord／stateへ束縛する。
- Developer Testing: automated resultとhuman Observationの非代替を明記する。
- Gameplay Programming Model: System Graph、Implementation Set、State Owner Projectionを定義する。
- Asset Lifecycle: 三laneとMVP asset boundaryをProduct claimへ接続する。
- Product Plan: AI generation claim三lane、FirstPlayableDefinition、required journeyを所有する。
- Product Execution Proposal: Shooter代用を除去したまま、FirstPlayableDefinitionのRPG projectionだけを参照する。
- AI Verification Catalog: freshness四状態を親Ownerへ移し、存在しない§10.1参照を削除する。
- Core Architecture: Minimum Executable Core Definition／Qualificationを追加する。
- Closure Review: 新gapをcanonical IDへ登録し、旧§7.9／§7.10を正しいtop-level follow-up節へ再配置する。
- Architecture Index／Review summary: Owner追加、監査input件数／digest、closure状態を更新する。

## 14. Errorとfail-closed規則

- understanding未完了、Playtest未評価、Human Approval不在をAI要約で補完しない。
- unsupported Capabilityを近いPack、外部Engine概念、Native Source生成へ暗黙変換しない。
- Playtest proseだけからRequirement、severity、approval、authorizationを推測しない。
- old Candidate、別Project revision、別Target、別Contract setのEvaluation／Approvalを再利用しない。
- Evidence state不明をfreshにしない。
- Minimum Coreの一部componentをmock、Editor process、別Target Receiptで代用しない。
- target design closureをmaterialization、QualificationまたはProduct completionへ昇格しない。

## 15. 検証Design

正本更新後は少なくとも次を検証する。

1. 全Owner header、Owner ID、Index membershipが一意。
2. 規範依存にmissing／cycleがない。
3. 全relative Markdown path／fragment／typed section linkが解決する。
4. `GameBriefV1`、旧`GameSpecDocument` candidate、旧`RequirementTraceabilityRefV1`等のshadow tokenが残らない。
5. Game production record familyが新Owner以外で再定義されない。
6. `GameSystemDependencyGraphV1`、State owner、Implementation setがGameplay Ownerへexact一件解決する。
7. freshness stateが親Ownerで一意、consumerが`fresh`以外をpositiveへ数えない。
8. FirstPlayableDefinitionがRPG requirement／Target／journeyを持ち、Shooterを参照しない。
9. Minimum Core DefinitionとQualificationがcurrent `not_materialized`状態を保持する。
10. target Operation familyのcurrent materialized／active／operational集合がexact `[]`。
11. `review`文書をimplemented／qualified／active／completeと表現するstatementがない。
12. 外部事実とMiraikanai project-decisionが分離され、確認日と一次資料URLがある。
13. `git diff --check`、`git status --short`、`git diff --stat`と全変更diffの目視確認を行う。

RepositoryにはBuild、test runner、Markdown linter、link checker、Inventory generator、CIがないため、それらを実行済みと主張しない。Architecture固有validatorはread-only scriptとして実行し、artifactを生成しない。

## 16. 完了条件

この再構築は次のすべてが成立した時だけArchitecture target designとして完了する。

- 新Ownerと全consumerの正本境界が一意である。
- PromptからGame Understanding Closure、Proposal、Validation、Playtest、Iteration、Human Approvalまで途切れない。
- AI生成claim三laneとMVP scopeに誤読余地がない。
- compact 2D command RPG First Playableのexact target Definitionがある。
- Evidence freshnessとSystem Graph／State ownerの未定義参照が解消する。
- Minimum Executable Coreのtarget closureとcurrent absent状態が同時に明示される。
- stale fragment／section、Closure Review節順、Index／Review summaryが整合する。
- current Repositoryが実装、Qualification、Activation、Releaseを持たない事実を維持する。

このDesign自体は非正本であり、各項目のcurrent contractは更新後のOwner文書だけが決定する。
