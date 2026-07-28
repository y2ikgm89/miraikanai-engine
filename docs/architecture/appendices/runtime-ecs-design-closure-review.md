# Runtime ECS Design Closure Review

- 文書ID: mirakan.appendix.runtime-ecs-design-closure-review
- 文書種別: proposal appendix
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 親Owner: [Runtime ECS](../04-runtime/entity-component-system.md)
- 正本範囲: Runtime ECS設計監査の結論、current／target区分、cross-owner整合性、未解決Closureの追跡
- 非正本範囲: Runtime ECS semantics、MCD Schema、Product Registry、実装Task、実装順序、担当、工数、生成済みArtifact、承認結果
- 規範依存: [Runtime ECS](../04-runtime/entity-component-system.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Gameplay programming model](../03-authoring/gameplay-programming-model.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)
- 関連文書: [Architecture Governance](../01-governance/architecture-governance.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Project State](../03-authoring/project-state.md)、[Native Game Module](../03-authoring/native-game-module.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Persistence／Save](../04-runtime/persistence-save.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[Governance Migration Proposals](governance-migration-proposals.md)、[Product Plan](../00-product/product-plan.md)、[Product Execution Registry Proposal](product-execution-registry-proposal.md)
- 根拠区分: project-review／official-spec comparison
- 外部根拠確認日: 2026-07-28

> 本書は実装計画ではない。列挙順は実装順序、Product Phase、担当または日程を意味しない。`closed-in-target-design`はtarget文書内の意味が閉じたことだけを表し、Schema、Artifact、Runtime、Receipt、Capability、Owner移管またはProduct Definitionが存在・承認・適用済みであることを意味しない。

## 1. 監査結論

MiraikanaiのRuntime ECSは、Engine-owned archetype／SoA、data-only Component、世代付きEntity handle、cached query、deferred structural mutation、宣言的access、seal済みAI projectionという方向を維持する。この方向はProductのGenre-neutral Core、Memory／Pointer制約、Save／Replay分離、AI authorization境界と整合する。

AI可読性は概念、state、機械解決、実行可能性、安全境界、cross-owner closureを分けて評価する。

| 観点 | 判定 | 根拠 |
|---|---|---|
| 概念理解 | strong | Owner、identity、storage、query、lease、structural transaction、AI projectionが分離されている |
| current／target判別 | explicit but split | current authorityはGameplay programming model revision 1、Runtime ECSはrevision 2のtarget Owner |
| 機械的解決 | incomplete | 未定義Ref、略記配列、未materialize Schema／Artifactが残る |
| 実行可能性 | absent | RepositoryにRuntime実装、生成Schema、validator、fixture artifact、Receiptがない |
| AI安全境界 | strong target design | live World、raw pointer、runtime handle、credentialをAI projectionへ渡さない |
| cross-owner closure | target aligned／open | Owner境界と一方向chainは整合するが、Inventory、Projection、Capsule、Eval、Owner移管が未materializeである |

したがって、AIは本書群から設計意図を説明できるが、未解決Refを補完し、targetをcurrentと解釈し、Markdown recordを実行可能Schemaとして扱ってはならない。完成Closureがない質問は`insufficient_authorized_context`またはdesign-time unresolvedとして返す。

## 2. 外部比較から採る原則

外部EngineはMiraikanaiの正本ではない。採用するのは検証済みの設計原則だけであり、API、型名、chunk size、World semantics、scheduler、binary formatを移植しない。

| 比較対象 | 一次資料で確認した構造 | Miraikanaiが採る原則 | 採らないもの |
|---|---|---|---|
| Unity Entities 1.4 | 同一Component集合のarchetype、chunk内のComponent列、query cache、EntityCommandBufferによるstructural change集約 | archetype／SoA、cached membership、deferred mutation、参照失効の明示 | Unity API、16 KiBの根拠、Job／World semantics |
| Unreal Engine 5.8 | UObject Reflection／serialization、Asset Registryによるtool可読metadata、MassEntityのdata-only Fragment／Archetype／Chunk／Processor／EntityQuery | reflection-backed bounded metadata、dataと処理の分離、query宣言、batch boundary | reflection可視性をAI authorityとみなすこと、任意property mutation、Actor／UObject／Mass API、World subsystem semantics |
| Godot stable | Node／Scene tree／Resource／Inspectorがauthoring構造を可視化し、low-level serverはopaque RIDを使う | 人間とAIが辿れる明示的構造、再利用単位、authoringとnative resource handleの分離 | Node treeをRuntime ECS／Entity identity／memory layoutとして扱うこと、RIDをsemantic projectionへ公開すること |
| Bevy ECS 0.19 | System parameterがaccessを宣言し、Query conflictを検出し、Commandsをdeferする | accessを型／manifestから導出し、競合を機械検査すること | Rust型、Schedule既定値、Bevy World semantics |

一次資料:

- [Unity Entities archetype concepts](https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/concepts-archetypes.html)
- [Unity Entities sync point guidance](https://docs.unity3d.com/Packages/com.unity.entities@1.4/manual/performance-sync-points.html)
- [Unreal Engine 5.8 MassEntity overview](https://dev.epicgames.com/documentation/unreal-engine/overview-of-mass-entity-in-unreal-engine?lang=en-US)
- [Unreal Engine 5.8 Reflection System](https://dev.epicgames.com/documentation/unreal-engine/reflection-system-in-unreal-engine?lang=en-US)
- [Unreal Engine 5.8 Asset Registry](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/AssetRegistry)
- [Godot key concepts](https://docs.godotengine.org/en/stable/getting_started/introduction/key_concepts_overview.html)
- [Godot Resource](https://docs.godotengine.org/en/stable/classes/class_resource.html)
- [Godot optimization using Servers／RID](https://docs.godotengine.org/en/stable/tutorials/performance/using_servers.html)
- [Bevy ECS systems](https://docs.rs/bevy_ecs/latest/bevy_ecs/system/)

### 2.1 AI可読性に対する比較結論

Unity EntitiesとBevyはruntime accessとdata-oriented iteration、Unrealはreflection／Asset metadataとMass data layout、GodotはScene／Resourceのauthoring可視性に強い。Miraikanaiはこれらを一つの外部Engine互換layerへ統合せず、authoringの意味、runtime layout、tool可読metadata、AI authorization、Evidenceを別Ownerのtyped Projectionとして接続する。

外部Engineで型、property、Node、Asset、Archetypeまたはqueryが列挙できることは、Miraikanaiではreadabilityの参考にだけ使う。列挙可能性はcurrent authority、write permission、semantic completeness、performance qualificationまたはCapability Activationを意味しない。逆に、Miraikanaiのstrict Contractが人間またはAIに読めても、生成InventoryとProjectionがない間はoperationalなAI可読性を主張しない。

## 3. Cross-owner整合性

| 接続 | 判定 | 正本境界 |
|---|---|---|
| Product Plan ↔ Runtime ECS | target aligned／current migration absent | ProductはCapability、Phase、Risk、Evidence categoryを所有し、ECSはruntime semanticsを所有する |
| Architecture Governance ↔ Runtime ECS | target aligned／Inventory absent | GovernanceはOwner、文書状態、依存、`ArchitectureExplainProjectionV1`を所有し、ECSはDomain fragmentだけを供給する |
| Gameplay ↔ Runtime ECS | authority transfer pending | Gameplay revision 1がcurrent authority、Runtime ECS revision 2がtarget authority |
| Scheduling ↔ Runtime ECS | target semantics aligned | SchedulingはT110 seal／次のT00 apply、ECSはdelta、preflight、commit、World epochを所有する |
| Memory ↔ Runtime ECS | aligned with open Schema closure | Memoryはpointer／allocator／lease保存禁止、ECSはEntity／Component固有leaseとinvalidationを所有する |
| Project State ↔ Runtime ECS | boundary aligned／Operation closure absent | Project StateはGame Project authoringのChangeSet／Commitだけを所有し、ECS graph、Runtime memory、Engine optimization Candidateを直接write対象にしない |
| Runtime Package ↔ Runtime ECS | boundary aligned／Ref closure open | PackageはWorld imageとbinary、ECSはconstruction setとbuild gateway validationを所有する |
| Persistence ↔ Runtime ECS | boundary aligned／identity evolution open | PersistenceはPersistent Entity Identity、Save record、migrationを所有し、ECS handleは保存しない |
| Native Module ↔ Runtime ECS | target projection aligned | read／write Component、structural permission、State writeを別集合として照合する |
| Performance ↔ Runtime ECS | target acceptance rule aligned | Performanceはcampaign、metric、absolute threshold、promotion predicateを所有する |
| AI Security ↔ Runtime ECS | policy aligned／Capsule and Operation absent | ECSはseal済みgraphを供給し、AI SecurityはTask Capsule、route、authorization、Approvalを所有する |
| AI Verification ↔ Runtime ECS／Performance | target aligned／Eval absent | VerificationはAI理解Eval、Evidence envelope、Receipt、freshnessを所有し、ECS／Performanceはsubjectとpredicateを供給する |
| Debug ↔ Runtime ECS | boundary aligned／E6 capture absent | ECSはpublication-bound graph、Debugはcapture transport、storage、redactionを所有する |
| Platform ↔ Runtime ECS | target mapping only | E7は各Targetのexact Profile、device、Toolchain、Receiptを必要とする |
| Governance ↔ Runtime ECS | process aligned／all transfer inputs absent | Owner移管は完成ChangeSetとDefinition Migrationのatomic適用だけで成立する |

targetのend-to-end連携は次の一方向chainである。

```text
Canonical Architecture／Project／Contract／Evidence
  -> Architecture Inventory／Explain Projection
  -> Memory fragment＋ECS Contract Graph＋Optimization Decision
  -> AiTaskContextCapsule
  -> AI read／explain／propose
     -> Game Project authoring:
        active Project Operation
        -> ProjectChangeSet／preview／validate／approve／commit
        -> read-back／Verification Receipt／new Project revision
     -> Engine algorithm／layout Candidate:
        owner Candidate／qualification
        -> independent selection／promotion／Contract revision
        -> Capability qualification／activation
```

上流ref、Project lineage、source revision、Target、Contract Set、Toolchain、fixture、freshnessのいずれかが一致しない場合は下流へ進めない。Project authoring branchとEngine optimization branchを相互変換せず、Project ChangeSet／Commit ReceiptをEngine Candidateのqualification／promotionへ、Optimization ReceiptをProject mutation authorityへ流用しない。read-only Projection、AI explanation、Proposal、Technical Qualification、human Approval、Commit、Product Activationを一つのstateまたは万能Receiptへ縮約しない。currentではInventory、Explain Projection、ECS Graph Artifact、Optimization説明Eval、Task Capsule、E6 read Operationが未materializeまたは未Activationであり、このchainはtarget alignmentであって利用可能なTool一覧ではない。

## 4. Design closure register

`open-blocker`はtarget ECSのcurrent化または該当CapabilityのActivation前に解決を要する。`proposal-only`はProduct／Governanceの未承認destinationにだけ存在する。`closed-in-target-design`は意味上の衝突を解消済みだが、materializationとQualificationを別に必要とする。

| ID | 論点 | 状態 | Owner／解決条件 |
|---|---|---|---|
| `ECS-C01` | current authorityとtarget Owner移管 | `open-blocker` | Governance。Inventory、Compatibility Change、Owner migration manifest、Definition Closure、Approvalを同一closureへ束縛する |
| `ECS-C02` | `CanonicalValueRefV1`等の参照型 | `open-blocker` | Executable Contracts／Gameplay／Persistence／Runtime ECS。全Refを一意なtop-level定義へ解決する |
| `ECS-C03` | 略記配列、closed enum、branch制約、hash preimage | `open-blocker` | 各Schema Owner。全collectionのelement type、bound、sort、uniquenessとself-excluding hashを閉じる |
| `ECS-C04` | column offset、stride、bitset padding、row capacity、ID生成 | `open-blocker` | Runtime ECS。canonical layout algorithmとgolden vectorを同じContract revisionへ含める |
| `ECS-C05` | structural deltaのdiscriminated union | `closed-in-target-design` | Runtime ECS §7.1。kind別required／forbidden Fieldをexact検査する |
| `ECS-C06` | T00／T110 structural boundary | `closed-in-target-design` | T110で次回batchをsealし、指定された次のT00でprivate working Worldへ一回だけapplyする |
| `ECS-C07` | `world_epoch`とpublication generation | `closed-in-target-design` | Runtime ECS §3.1。前者はworking topology、後者はexternal publicationを数える |
| `ECS-C08` | value write、structural commit、failure atomicity | `closed-in-target-design` | faulted advanceをpublish／再利用せず、last published Worldをexternal authorityとして維持する |
| `ECS-C09` | Query predicate、cache key、invalidation | `open-blocker` | Runtime ECS。pure bounded predicate、cache generation、differential update semantics、golden query resultを閉じる |
| `ECS-C10` | read／write overlap、lease scope、parallel dispatch | `closed-in-target-design` | Runtime ECS＋Scheduling。actual selected row intersectionとComponent access modeで競合を判定する |
| `ECS-C11` | Entity／Component enablementとlifecycle | `closed-in-target-design` | Runtime ECS §7.1。enable値とtarget Componentをunion branchへ含める |
| `ECS-C12` | runtime spawnとPersistent Entity Identity | `open-blocker` | Persistence＋Runtime ECS。template policy、identity request、collision、Save projectionを閉じる |
| `ECS-C13` | Native ABI access mapping | `closed-in-target-design` | Native Module §7.1。read Component、write Component、State write、structural permissionを別集合で照合する |
| `ECS-C14` | concrete fixture、input trace、cardinality、expected hash | `open-blocker` | Runtime ECS＋Verification。logical IDをcontent-addressed fixture artifactへ解決する |
| `ECS-C15` | initial memory／latency acceptance | `closed-in-target-design` | Performance §8.3。承認済みTarget別absolute threshold refがなければProfileをmaterializeしない |
| `ECS-C16` | Runtime ECS diagnostic coverage | `closed-in-target-design` | Runtime ECS §9.2。generation、lease、access、delta shape、commit conflict、publication、identityをtyped failureにする |
| `ECS-C17` | current Phase 0 ECS Gate | `proposal-only` | Product Definition Migration。current Registryへ推測追加しない |
| `ECS-C18` | E7 exact Target／device qualification | `open-blocker` | Platform＋Product。Target Profile、device identity、Toolchain、full campaign Receiptを一対一に束縛する |
| `ECS-C19` | Product Owner indexとRisk Owner | `closed-in-target-design` | Product PlanのRuntime Owner indexへRuntime ECSを追加し、Product Risk rowのOwnerをProduct Planへ統一する |
| `ECS-C20` | RPG-first targetとShooter current Registry | `proposal-only` | Product Definition Migration。Shooter ReceiptをRPG Evidenceへ流用しない |
| `ECS-C21` | E6 end-to-end AI context／Operation／capture | `open-blocker` | Governance＋Runtime ECS＋Performance＋AI Security＋AI Verification＋Debug。Inventory／Explain Projection、ECS Graph、Optimization Decision、Task Capsule、read Operation、redaction、freshness、理解Evalを同じlineage／Target／Contractへ閉じる |
| `ECS-C22` | Component schema evolution | `open-blocker` | Compatibility＋Domain＋Persistence。normal revision、migration、Save／Replay、Native consumerを同一Changeへ束縛する |

## 5. AI-readable contract条件

AI向け説明は次の順で解決する。

1. current Owner RegistryとContract Setからcurrent authorityを確定する。
2. 同じInventory hashの`ArchitectureExplainProjectionV1`からRuntime ECSのOwner、document state、implementation state、依存、consumerを解決する。
3. target／proposal文書は比較または変更候補としてだけ読み、exact Type／Ref／hashが解決できるrecordだけを`RuntimeEcsContractGraphV1`へ含める。
4. pointer／allocation説明はMemory fragment、Candidate状態とselected／rejected理由は`OptimizationDecisionProjectionV1`から解決する。
5. Explain Projection、Memory fragment、ECS Graph、Optimization Decisionを同じProject lineage、source revision、Target、Contract Set、Toolchain、fixture、freshnessの`AiTaskContextCapsuleV1`へ束縛する。
6. redaction後にOwner、status、failure、selected／rejected理由を確定できない場合は推測せず停止する。

`RuntimeEcsContractGraphV1`はread modelであり、MCD正本、Runtime memory、Operation catalog、Capability Activation、Approval、write authorityの代用ではない。Markdown code block、表、logical ID、文書名だけからactive recordまたは実装済みCapabilityを生成しない。

## 6. Current化とCapability表示の条件

Runtime ECS revision 2のcurrent化には、少なくとも次のexact closureを必要とする。

- `ECS-C01`～`ECS-C04`、`ECS-C09`、`ECS-C12`、`ECS-C14`、`ECS-C22`の`open-blocker`解消。
- 全ECS record／Refのmachine-readable Schema、canonical encoding、hash golden vector。
- Gameplay Spec、ECS manifest、Native descriptor、Package、Save／Replay、Debug／AI projectionの正逆参照一致。
- T110 seal→次のT00 apply→T110 publicationと、failure時last-publication維持のdeterministic fixture。
- 同一construction inputからlayout、query plan、structural merge、publication hashが再現するEvidence。
- complete Consumer Inventory、Compatibility Change、Owner migration、Definition Migration、Approval。

E6／E7／RPG-firstはそれぞれのCapabilityまたはProduct表示条件を追加する。ECS core current化だけを理由にAI authoring、全Target対応、RPG MVP、Shipping readinessを表示しない。

E6のAI可読性表示には`ECS-C21`の解消に加え、AI理解Evalのfresh pass Receipt、対象routeのconformance、read OperationのActivationが必要である。Architecture InventoryまたはECS coreだけがcurrent化しても、AI理解、AI提案、AI変更を一括して利用可能と表示しない。

## 7. 本Reviewで整合させる文書

| 文書 | 設計上の修正 |
|---|---|
| Architecture Governance | Owner／state／dependencyのExplain Projectionを所有し、Domain graphまたはAI authorityを所有しない |
| Memory／Pointers | pointer／allocation fragmentだけを供給し、ECS layout、Candidate状態、AI authorizationを複写しない |
| Runtime ECS | currentization gate、epoch／publication、structural union、T110／T00、access conflict、diagnostic、qualificationを明確化 |
| Scheduling／Lifetime | sealとapply、faulted working Worldの扱いをRuntime ECSと一致 |
| Gameplay programming model | `GameSystemSpecV2` field-level closureがcurrent化前提であることを明記 |
| Project State | Game Project向けAI proposalだけをactive Project Operationから`ProjectChangeSetV1`へ収束し、Engine Candidate選定をProject mutationへ変換しない |
| Native Game Module | Component writeとstructural permissionのNative projectionを追加 |
| Performance／Capacity | initial absolute memory／latency threshold recordとCandidate Decision Projectionを所有する |
| AI Security／Approval | Task Capsule、route、authorization、Approvalを所有し、readabilityをwrite authorityへ昇格しない |
| AI Verification／Provenance | AI理解Eval、Evidence envelope、Receipt、freshnessを所有する |
| Runtime Package | Section変更をT110 seal／次のT00 apply／成功T110 publicationへ統一 |
| Persistence／Save | Authoritative digestをexact World epoch／publication generationへ束縛 |
| Product Plan | Runtime／Simulation Owner一覧とAI-readable ECS／Memory alignmentのProduct評価を所有する |
| Product Execution Registry Proposal | ECS Product RiskのOwnerをProduct Planへ統一 |
| Governance Migration Proposals | 本Closure registerをOwner移管前のdesign inputへ追加 |
