# Miraikanai Engine Gameplay Programming Model

- 文書ID: mirakan.arch.gameplay-programming-model
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: 構造化GameplayとProject C++の選択境界、manual／AI gameplay proposal parity、GameplayDefinition、GameSystemSpecV1、GameSystemDependencyGraphV1、SystemImplementationSetV1、GameStateOwnerProjectionV1、State owner、typed Command／Event／Snapshot Port、Perception／Interaction／Decision／Action意味、Project-defined System、generated projection／Promotion条件
- 非正本範囲: 具体Schema／Registry／Fixture候補、Native ABI、Project transaction、共有Schema基盤、Runtime scheduling、外部Tool固定、Navigation query、Character Motor、Project固有Interaction結果
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Project State](project-state.md)、[Executable Contracts](../02-foundation/executable-contracts.md)
- 関連文書: [Game Production Loop](game-production-loop.md)、[AI-native C++ Product Identity Decision](../decisions/2026-08-03-ai-native-cpp-product-identity.md)、[Generated Projection／Fixture Candidate Catalog](../appendices/gameplay-generated-projection-fixture-catalog.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Native Game Module](native-game-module.md)、[Runtime ECS](../04-runtime/entity-component-system.md)、[Navigation](../05-simulation/navigation.md)、[Animation](../05-simulation/animation.md)、[Input](../07-platform/input.md)、[Debugging／Replay](../04-runtime/debugging-observability-replay.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-27

Miraikanai EngineはCPU上のEngine／Game実行codeをC++とし、調整頻度の高い挙動と内容をSchema検証可能な`GameplayDefinition`としてAuthoringする。Game Systemは契約固定・実装開放型で、同じPublic Contractに対してstructured definition、bounded Project C++、hybrid、Target-specialized implementationを選べる。

自由にするのはGenre、Core loop、System構成、Algorithm、2D／3D表現である。固定するのはPublic System Contract、State owner、typed Port、lifecycle、Save／Replay意味、Target、Budget参照、Test、failure、fallbackである。

## 1. 構造化dataとC++のdecision matrix

| 条件 | 標準選択 |
|---|---|
| bounded data、closed rule、FSM、parameterized behavior | `GameplayDefinition` |
| hot loop、複雑algorithm、native integration | bounded Project C++ |
| data-driven policy＋native executor | hybrid |
| Target差が契約内に閉じる | Target-specialized implementation set |
| Public Capability外の要求 | `capability_unavailable`で停止 |

Project C++はNative Game Module境界、MCD Port、Memory／Pointer規則、Target Build identityに従う。Runtime code generation、download code、unrestricted scripting、Engine private header、raw pointerのPublic Port露出を認めない。

### 1.1 AI-native gameplay authoring closure

AIと人間は同じ`GameplayDefinition`、`GameSystemSpecV1`、Project C++、Native Game Module descriptorをSourceとして扱う。Editor、CLI、IDE、AIのいずれから開始しても、提案は同じProject revision、Owner、read／write set、Contract set、Validation、semantic diff、Approval、atomic Commitへ収束し、AI専用Gameplay objectまたはAIだけが書けるRuntime stateを作らない。

- bounded data、closed rule、FSM、parameterized behaviorは構造化Sourceを標準とし、複雑algorithm、hot loop、native integrationは公開Contract内のbounded Project C++、両者の分離が有効な場合だけhybridを選ぶ。AIの得意不得意、外部EngineのScript／visual graph慣行またはProvider出力形式で選択を反転しない。
- AI生成C++は通常のProject C++であり、Engine private API、unregistered service、unrestricted reflection、runtime code generationまたは特別な免責領域へ置かない。人間が同じSourceをIDEで読解、変更、build、test、revertできなければ採択しない。
- AIはquery／explainとSource proposalを行えるが、Contract activation、Engine／Adapter変更、approval、Commit authorityを取得しない。Provider不在、拒否、timeoutまたはprotocol非互換時も、既存Sourceをmanual surfaceから保守できなければならない。
- 同じsemantic fixtureをmanual proposal経路とAI proposal経路からValidationし、accepted Source／Cooked／Runtime／Save／Replayの意味を比較する。AIがコードを生成した事実、build成功または自然言語説明だけをGameplay capability、iteration speed、correctnessまたはpromotion Evidenceにしない。

この節はAI Operation、Task authorization、Project transaction、Native ABIまたはFixture Schemaを再定義しない。それぞれ[Executable Contracts](../02-foundation/executable-contracts.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[Project State](project-state.md)、[Native Game Module](native-game-module.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)をOwnerとする。

## 2. `GameplayDefinition`

`GameplayDefinition`はstable identity、schema ref、owner、State declaration、Input／Output Port、rule graph、resource ref、budget ref、failure policyを持つ。表示名、Editor object address、配列index、Runtime Entity handleをSource identityにしない。

Authoring Source、Cooked artifact、Runtime instanceを分離する。CookはSource ref、schema、Contract set、Toolchain、Targetを束縛し、RuntimeはCooked artifactからSourceを推測しない。Save／Replayは宣言済みState ownerとprojectionだけを保存し、live pointer、lease、native objectを含めない。

### 2.1 bounded contract

構造化Gameplayはclosed operator、bounded collection、explicit unit、deterministic orderingを使う。任意code、reflectionによるprivate state access、filesystem／network／process／FFIを許可しない。

### 2.2 Authoring、Cook、Runtime形式

三形式は同じsemantic identityとversionを共有するが、wire layoutとlifetimeは別である。SourceをRuntime memoryへ直接mapせず、Cooked artifactをAuthoring Sourceへ逆書込みしない。

### 2.3 State、Save、Replay、live edit

State fieldごとにOwner、scope、writer、readers、save policy、replay policy、live-edit policyを宣言する。Live editはcompatible fieldだけをtransaction境界で反映し、incompatible schema／owner／scope変更はrestart、recook、migrationのいずれかを明示する。

### 2.4 C1 Perception／Interaction

<a id="InteractionDefinitionV1"></a>
<a id="InteractionRequestV1"></a>
<a id="InteractionSnapshotV1"></a>
<a id="InteractionSpaceBindingV1"></a>
<a id="InteractionSpaceSemanticActivationProjectionV1"></a>
<a id="InteractionSpaceSemanticContributionV1"></a>
<a id="InteractionSpaceSemanticRefV1"></a>
<a id="InteractionSpaceSemanticRegistryV1"></a>
<a id="LogicalInteractionBindingV1"></a>
<a id="SpatialInteractionBindingV1"></a>
<a id="UiInteractionBindingV1"></a>

Perceptionはboundedな視覚／聴覚観測、Interactionは対象発見、prompt semantic、request、競合制御を所有する。Project固有のdoor、pickup、inspect、talk結果はtyped Port consumerが所有する。

Interaction Space Semanticはlogical、spatial、UIの空間差をclosed semantic refとbindingで表す。ContributionはOwner、Target space、activation conditionを宣言し、Registryへatomicに追加する。Unknown semantic、複数active binding、wrong owner／hashを拒否する。

具体Field、Registry row、Fixture候補は[Generated Projection／Fixture Catalog](../appendices/gameplay-generated-projection-fixture-catalog.md#24-c1-perceptioninteraction)を参照する。

### 2.5 Rule／ECAとFinite State Machine

Ruleはevent、pure condition、bounded action proposalからなり、直接Domain stateを書かない。FSMはclosed state／event／transition、initial／terminal state、failureを持つ。競合proposalはOwnerが定めるdeterministic policyで解決する。

### 2.6 Perception→Decision→Action接続

Gameplay AIの方式としてBehavior Tree、Utility、Planner、FSMのいずれかをCore既定にしない。必要なのは方式ではなく、`observation -> bounded memory -> decision proposal -> selected action -> typed command -> subsystem execution -> completion／failure -> explanation`の一方向contractである。

| 段階 | 正本Owner | 境界 |
|---|---|---|
| observation／stimulus | Perception／Interaction | source、subject、observed advance、confidence／completeness、expiryをboundedに公開 |
| bounded memory | Gameplay System／Feature Owner | retention、aging、merge、forget、Save／Replay policyを宣言し、Renderer visibilityやlive queryを記憶として流用しない |
| decision proposal／selection | Gameplay System／Genre／Project policy Owner | read set、goal／rule、candidate action、selection理由、tie policyを所有し、Subsystem private stateを書かない |
| action command | Commandを所有するFeature／Subsystem | exact target、precondition、idempotency、cancel／interrupt policyを検証 |
| execution | Navigation、Animation、Input／Command、Feature等 | typed requestを実行し、decision policyを再定義しない |
| completion／failure | action Owner | accepted、rejected、cancelled、interrupted、failedを一つのterminal resultとして返す |
| explanation／Replay | Debug／Persistence＋各Owner projection | causalityと選択理由をboundedに束縛し、欠損を推測しない |

stimulus expiry、higher-priority Event、target消失、Navigation path invalidation、Animation lock、Feature rejectionは明示的なinterrupt／failure inputである。interruptはselected actionのOwnerへtyped requestとして渡し、Animation、Navigation、Physics、Rendererのprivate stateを直接変更しない。cancel不可boundaryを越えた成功をcancelledへ書き換えず、次action proposalへ解決する。

Perception memory、decision state、selected actionのSave／Replay扱いはOwnerが明示する。deterministic gameplay結果へ影響する入力、RNG binding、selection、accepted Command、terminal resultはReplay projectionへ含め、cache、worker timing、Editor selection、localized explanationをauthoritative stateにしない。non-deterministic inputを使用する場合はrecord／replay可能なsealed observationへ変換するか、deterministic Replay非対応をCapability／Qualificationで明示する。

Debug projectionは「何を観測し、何を記憶し、どの候補をなぜ棄却し、どのactionを選び、どのSubsystemがどの結果を返したか」を同じcausal chainで返す。gap、expired observation、redacted input、unknown policy、missing terminal resultがある場合はcomplete explanationを捏造しない。Genre固有のNPC behaviorはこのcontractを消費し、Generic Gameplay ModelへRPG／Shooter固有decision treeを追加しない。

## 3. `GameSystemSpecV1`

`GameSystemSpecV1`はGame System identity、owner、origin、Capability、State owner、read／write access、phase、Port、implementation set、Save／Replay、Budget、failure、qualificationを宣言する。

同一authoritative fieldのwriterは同一advanceに一つである。Systemは未宣言Component、State、Portへaccessせず、phaseやpriorityだけでwriter authorityを得ない。Implementation Variantは同じPublic ContractとState semanticsを維持する。

本文の列挙はfield-level Schemaの代用ではない。`GameSystemSpecV1`をcurrent Definition Closureへ含める前に、少なくともSystem identity／version、Owner、origin、Capability、Runtime Scope、State classとState owner、Component read／write集合、structural permission集合、State read／write集合、phase集合、Command／Event／Snapshot Port、implementation set、Save／Replay policy、Budget、failure、qualification subjectを持つ一つのbounded canonical Schemaへ解決する。各collectionのelement type、bound、sort、uniqueness、branch制約、self-excluding contract hashが欠ける場合、Contract compiler、ECS manifest、Native descriptor、Package、Save／Replay projectionを生成しない。

### 3.0.1 System graph、Implementation Set、State owner projection

```text
GameSystemDependencyGraphV1
  graph_id: StableId
  graph_version: 1
  project_revision_ref: exact ProjectRevisionRefV1
  contract_set_ref: exact ContractSetRefV1
  system_nodes[1..4096]: sorted unique {
    game_system_contract_ref: exact GameSystemContractRefV1,
    game_system_spec_content_hash: SHA-256,
    runtime_scope_type_ref: exact RuntimeScopeTypeRefV1,
    phase_ids[1..16]: sorted unique TickPhaseId,
    read_component_type_refs[0..4096]:
      sorted unique exact McdContractRefV1(kind=type),
    write_component_type_refs[0..4096]:
      sorted unique exact McdContractRefV1(kind=type),
    structural_permission_refs[0..256]: sorted unique exact ArtifactRefV1,
    read_state_type_refs[0..1024]:
      sorted unique exact McdContractRefV1(kind=type),
    write_state_type_refs[0..1024]:
      sorted unique exact McdContractRefV1(kind=type),
    command_type_refs[0..1024]: sorted unique exact McdContractRefV1(kind=type),
    event_type_refs[0..1024]: sorted unique exact McdContractRefV1(kind=type)
  }
  dependency_edges[0..65536]: sorted unique {
    producer_system_ref: exact GameSystemContractRefV1,
    consumer_system_ref: exact GameSystemContractRefV1,
    edge_kind:
      build | cook | state_read_after_write | command_delivery |
      event_delivery | lifecycle,
    boundary: same_advance | next_advance | activation | shutdown,
    subject_type_ref: exact McdContractRefV1(kind=type) | null
  }
  graph_content_hash: SHA-256

GameSystemDependencyGraphRefV1
  graph_id: StableId
  graph_version: 1
  graph_content_hash: SHA-256

SystemImplementationSetV1
  implementation_set_id: StableId
  implementation_set_version: 1
  project_revision_ref: exact ProjectRevisionRefV1
  contract_set_ref: exact ContractSetRefV1
  game_system_dependency_graph_ref: exact GameSystemDependencyGraphRefV1
  target_profile_ref: exact TargetProfileRefV1
  selections[1..4096]: sorted unique {
    game_system_contract_ref: exact GameSystemContractRefV1,
    implementation_variant_ref: exact ArtifactRefV1,
    implementation_kind:
      gameplay_definition | native_project_source | hybrid |
      target_specialized,
    implementation_variant_hash: SHA-256,
    configuration_content_hash: SHA-256,
    compatibility_evidence_refs[1..64]: sorted unique exact EvidenceRefV1
  }
  implementation_set_content_hash: SHA-256

SystemImplementationSetRefV1
  implementation_set_id: StableId
  implementation_set_version: 1
  implementation_set_content_hash: SHA-256

GameStateOwnerProjectionV1
  state_owner_projection_id: StableId
  state_owner_projection_version: 1
  project_revision_ref: exact ProjectRevisionRefV1
  contract_set_ref: exact ContractSetRefV1
  game_system_dependency_graph_ref: exact GameSystemDependencyGraphRefV1
  entries[1..65536]: sorted unique {
    state_type_ref: exact McdContractRefV1(kind=type),
    authoritative_owner_system_ref: exact GameSystemContractRefV1,
    runtime_scope_type_ref: exact RuntimeScopeTypeRefV1,
    writer_field_refs[1..4096]: sorted unique exact ArtifactRefV1,
    reader_system_refs[0..4096]: sorted unique exact GameSystemContractRefV1,
    publication_boundary_id: TickPhaseId,
    save_replay_policy_ref: exact ArtifactRefV1
  }
  state_owner_projection_content_hash: SHA-256

GameStateOwnerProjectionRefV1
  state_owner_projection_id: StableId
  state_owner_projection_version: 1
  state_owner_projection_content_hash: SHA-256
```

全Refは解決先完成recordとbyte equalityにし、各配列は要素のMCD canonical bytes全体によるunsigned lexicographic strict sort、duplicate 0である。Graphのnode projectionは同Project revision／Contract setのrequired `GameSystemSpecV1`集合とset equalityにし、各nodeのphase、Component、State、Command、Event集合をSpecとbyte equalityにする。edge両端はnode集合内に存在し、self-edge、未宣言subject、read／writeまたはPortから説明できないedgeを拒否する。

`build | cook` edgeおよび`boundary=same_advance` edgeの部分graphはDAGでなければならない。`next_advance | activation | shutdown`を跨ぐ論理cycleは、全edgeが明示され、bounded queue／capacity、fault、Replay requirementが各Owner contractへ解決する場合だけ許可する。同phase callback再入、same-advance write cycle、phase priorityによるcycle隠蔽を許可しない。

Implementation Setの`selections[].game_system_contract_ref` projectionはGraph node集合とset equalityにし、各required SystemへTarget互換、Contract／State semantic互換、passかつ`fresh`なEvidenceを持つ実装exact一件を選ぶ。同じSystemへの0件／複数件、別Target、異なるContract set、失効Evidence、DefinitionとNativeの同時authoritative selectionを拒否する。Play中の選択変更は行わず、新しいRuntime Entry packageとGameHostを要求する。

State owner projectionはGraph全nodeのauthoritative `write_state_type_refs[]`から再計算し、全required authoritative State Typeをexactly onceで覆う。owner 0件／複数件、owner以外のwriter、同じState fieldへの同一advance複数writer、別Scope、reader未宣言、Save／Replay policy欠損を拒否する。Runtime ECS access manifest、Scheduling、Runtime Package、Native descriptorはこのprojectionを消費するがshapeを再定義しない。

### Runtime ECS access cohort projection

Runtime ECS accessはGame SystemのComponent read集合を`RuntimeComponentAccessManifestV1.read_component_refs`、Component write集合を`write_component_refs`、structural permission集合を`structural_permissions`、State read／write集合をstate store bindingへ別々に投影する。State writeをComponent writeへ、Command permissionをstructural permissionへ、phase membershipをwriter authorityへ読み替えない。Gameplay側はECS storage、archetype、query lease、structural transactionを再定義しない。

### Owner identity registry

Owner identityはstable Owner ID、revision、authority document、content hashを束縛する。表示名、path、旧称をRegistry aliasとして保存しない。具体候補は[補助Catalog](../appendices/gameplay-generated-projection-fixture-catalog.md#owner-identity-registry)を参照する。

`owner.core.runtime_ecs`のinitial V1 authority documentは[Runtime ECS](../04-runtime/entity-component-system.md)である。本書はGame System identity、authoring semantics、State／Port、ECS access cohortへのprojectionだけを所有し、Entity／Component storage、archetype、query lease、structural transactionまたはRuntime ECS Owner identityを再定義しない。旧Owner revision、移管元、aliasまたはdual Registryをinitial V1へ作らない。

### 3.1 `RuntimeScopeTypeCatalogV1`

Runtime Scopeはtype identity、instance key schema、owner、lifetime、Save／Replay、activation／deactivation conditionを宣言する。Core closed enumへFeature固有Scopeを埋め込まず、Feature contributionはOwner、version、hash付きrecordとして登録する。

### 3.1.2 Scope依存record

Runtime ScopeのOwner、typed instance key、lifetime、Save／Replay dependencyはinitial V1の正規recordへ直接定義する。過去draftのinline scope、legacy contribution、offline migration、aliasをcurrent Catalogへ登録しない。具体record候補は[補助Catalog](../appendices/gameplay-generated-projection-fixture-catalog.md#312-scope依存record)を参照する。

## 4. State ownershipとtyped ports

StateはOwner、Scope、writer、readers、Save、Replay、publication boundaryを一意に宣言する。Commandはintent、Eventは発生済みfact、Snapshotはseal済み観測を表し、同じTypeを兼用しない。

Portはbounded valueまたはtyped refだけを運び、raw pointer、live lease、native handle、mutable containerを公開しない。Failureはtyped result／Diagnosticで返し、silent dropまたはdefault successにしない。

`presentation_cue_ref`は、そのFieldを含むSchema、validator、Cooker、valid／invalid fixture、Product destinationが同じDefinition closureで採用されるまでplanned fieldであり、current Gameplay MCDへ追加しない。Activation後はexact `SoundCueDefinitionV1`とMixer Bus Stable IDへ解決し、Asset path、display label、native Voice、暗黙のdefault Busへfallbackしない。Gameplay OwnerはT90前にsealするauthoritative Eventとpresentation intentだけを所有し、[Audio](../07-platform/audio.md)がCue選択、playback state、Voice capacity、Mixer gain、loop、Spatial、device failureを所有する。Audio completionまたは失敗をGameplay Event、Timer、Stateへ逆入力しない。

## 5. Project-defined systems

Project-defined SystemはEngineのPublic Capability、MCD Type、Portだけを利用する。Engine core改変、private namespace、unregistered service、unbounded runtime reflectionを要求するSystemは拒否する。

## 6. AI planとcode generation

AIは既存Capability内のSource candidate、validation、explanationを提案できるが、Approval、Contract activation、Engine／Adapter改変、runtime code generationを行わない。生成候補はTask authorization、expected Project revision、semantic intent、Validator closureを束縛する。

具体Generation recordとOperation候補は[補助Catalog](../appendices/gameplay-generated-projection-fixture-catalog.md#6-ai-planとcode-generation)へ分離する。

## 7. Generated bundle

Generated bundleはSource、Contract set、Toolchain、Target、Artifact、manifestを一つのcontent closureへ束縛する。Partial bundle、stale Source、wrong Target、missing manifestをpublicationしない。具体Schema候補は[補助Catalog](../appendices/gameplay-generated-projection-fixture-catalog.md#7-generated-bundle)を参照する。

## 8. ValidationとBuild

ValidationはSchema、cross-ref、Owner、State writer、phase、Port、Save／Replay、Budget、Target、Dependency closureを検査する。Build成功だけでGameplay semanticsまたはQualificationを証明しない。

## 9. Testingとpromotion

Contract、Source、Cooked、Runtime、Save／Replay、AI proposal、failure、recoveryを同じsemantic fixture closureで検証する。Promotionは同一Candidate、Target、Toolchain、Evidence、Approvalを束縛し、別buildの結果を流用しない。

具体Fixture／Promotion record候補は[補助Catalog](../appendices/gameplay-generated-projection-fixture-catalog.md#9-testingとpromotion)へ分離する。

## 10. 禁止する固定階層とunrestricted scripting runtime

Genre、Level、Player、Character、Weapon、QuestをEngine必須階層へ固定しない。Unrestricted scripting VM、eval、download code、native FFIをGameplayDefinitionの一般逃げ道にしない。

[Product Plan](../00-product/product-plan.md)の`future.capability.unrestricted-project-scripting-runtime`はstructured content MOD、sandboxed executable MOD、signed AOT desktop native extension、developer-only executable code（JIT／未署名）を分離検討するincubation umbrellaであり、current GameplayDefinition、Project-defined System、AI code generation、NativeGameModuleを拡張しない。umbrella、表示名、署名、hash一致からOperation、Host API、native権限、Shipping対応を生成しない。
