# Miraikanai Engine C++実行コード・構造化ゲームデータ規約

- 文書版: 1.2
- 作成日: 2026-07-19
- 調査基準日: 2026-07-20
- 状態: プロジェクト公式の規範設計レビュー版
- 上位文書: [AIネイティブ独自ゲームエンジン 設計計画書](./2026-07-18-ai-native-game-engine-authoring-design.md)
- 関連基盤: [Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)
- C++言語・Modules詳細: [Miraikanai Engine C++23・Named Modules・`import std`移行規約](./2026-07-20-cpp23-modules-import-std-transition-design.md)
- Authoring詳細: [Miraikanai Engine Authoring Model／Project State規約](./2026-07-19-authoring-model-project-state-design.md)
- Runtime詳細: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- NativeGameModule詳細: [Miraikanai Engine NativeGameModuleアーキテクチャ規約](./2026-07-19-native-game-module-architecture-design.md)
- 契約詳細: [Miraikanai Engine 実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)

## 1. 結論

Miraikanai Engine 1.0の公式実装方式を、次に固定する。

> CPU上で実行するEngineおよびGame固有コードはC++23へ統一する。First-party C++公開境界はNamed Modules、標準Libraryは原則`import std`へ一方向移行する。Scene、Entity、Composition Recipe、Quest、Dialogue、Ability、Behavior、UI Flow、Encounter、調整値は、Miraikanai Engineが所有する検証可能な構造化ゲームデータとして保存する。構造化ゲームデータはoffline Cookし、C++ Runtimeが型付きCompact Binaryとして実行する。

次を採用しない。

- Luau、Lua、C#等のGame scripting runtime。
- 汎用Script VM、bytecode interpreter、JIT、Game向けFFI。
- 汎用プログラミング言語を再発明する独自Script言語。
- Scene、Quest、武器値、UI配置までC++ sourceへ埋め込むsource-only authoring。
- Shipping RuntimeでのC++、Shader、native／managed executableの生成、download、compile、dynamic load。
- AI、Editor、外部ToolによるWorld memory、pointer、vendor objectへの直接書込。

Engine内に`ScriptRuntime`、`Script VM`、`ScriptStateStore`、言語別Capability bindingを設けない。置換後の正規概念は`GameplayDefinition`、`CookedGameplayPackage`、`GameplayStateStore`、`NativeGameModule`である。

## 2. 本書の決定権

本書は次を決定する。

- C++と構造化ゲームデータの責務境界。
- AIが構造化編集とC++生成を選ぶ規則。
- GameplayDefinitionの実行、安全性、保存、Cook、Hot Reload原則。
- NativeGameModuleの公開境界、Build、Preview、Promotion原則。
- Luauおよび汎用Script VMを採用しないこと。

Subsystem固有のfield、phase、memory配分、Target、Store、Testの詳細は各正式仕様を基準とする。同じ主題で矛盾する場合、Game実装方式については本書を優先する。

## 3. 「C++実行コード」の正確な範囲

### 3.1 C++へ統一する範囲

- Foundation、Memory、Job、Diagnostics。
- World／ECS、Serialization、Save／Replay。
- Renderer orchestration、Asset Runtime、Physics、Collision、Navigation、Animation。
- Audio、Input、UI Runtime、VFX、Gameplay Logic。
- Editorのnative shell、Authoring Gateway、Validator、Runtime Compiler。
- Project固有の`NativeGameModule`。

### 3.2 必要な例外

「C++実行コード」はRepository内の全FileをC++へ限定する意味ではない。次の例外は役割を限定して許可する。

| 例外 | 許可範囲 | 禁止 |
|---|---|---|
| GPU program | Material／Shader IR、portable HLSL、Target別DXIL／SPIR-V／metallib | Game／World authority、Runtime任意compile |
| Apple bridge | Objective-C++ Adapter、必要なOS metadata | Gameplay実装、Projectからの直接利用 |
| Android bridge | Java／Kotlin bootstrap、JNI Adapter | Gameplay実装、Projectからの直接利用 |
| C ABI | Platform／vendor／module境界 | owning pointer、Exception、vendor型の公開 |
| TypeScript | Engine外AI Orchestrator、Provider／MCP projection、Build-only Tool | GameHostへの同梱、World authority |
| 構造化File | MCD、GameSpec、World、Asset metadata、GameplayDefinition | 実行可能Source、無検証Runtime反映 |

Texture、Mesh、Animation、Audio、Font等はAssetであり、C++ sourceへ埋め込まない。

## 4. 有名Engineの確認結果と採用判断

| Engine | 公式に確認した構成 | 本Projectへの示唆 |
|---|---|---|
| Unreal Engine | Blueprintのみ、C++のみの両方が可能だが、多くのProjectは併用が有効。Data Assetも正式提供 | C++ onlyは成立するが、調整可能なdata層が必要 |
| CRYENGINE | C#／Luaを使わないC++ only Game DLL sampleを公式提供 | 独自Editor付きEngineでもC++ Game Moduleは実現可能 |
| Unity | Game scriptingはC#。Visual Scriptingを提供。IL2CPPはC# ILをBuild時にC++へ変換 | 生成C++とC++ authoringを混同しない |
| Godot | GDScript、C#、C++ GDExtensionを正式対応し、初心者にはGDScriptを推奨 | 易しい上位制作層は必要だが、言語である必要はない |
| O3DE | C++機能をLuaとScript Canvasへ公開し、反復速度と非Programmer操作を補う | MiraikanaiではAI＋構造化編集がこの役割を担う |

Miraikanai Engineは既存Engineの言語構成やEditor object modelをコピーしない。共通課題であるPerformance、反復速度、Designer操作、Data authoringを確認し、上位制作層をAI統合済み構造化データとして独自に解決する。

## 5. 正規用語

| 用語 | 定義 |
|---|---|
| `GameplayDefinition` | Game固有の挙動・内容を表すversion付き構造化Authoring object |
| `GameplayDefinitionSet` | 同一revisionで整合するDefinitionと依存関係のclosure |
| `CookedGameplayPackage` | DefinitionSetを検証・正規化・索引化したPlatform ABI非依存のcanonical Runtime binary。明示Target Profile／Capability intersectionに対してCookする |
| `GameplayStateStore` | Definition instanceのauthoritative runtime stateを持つEngine-owned store |
| `GameplayTask` | 待機を明示する`{task_id, state_id, wake_tick, bounded_parameters}` |
| `NativeGameModule` | 公開SDKとCapability contractだけを使うProject固有C++ module |
| `GameplayDefinitionChangeSet` | Definitionを変更するrevision付きtransaction |
| `NativeCodeChangeSet` | C++ source、header、CMake、Testを変更する高Risk transaction |
| `Capability` | C++ Runtimeが実装し、MCDで公開する型付き操作・query・event |

`GameplayDefinition`を「Script」と呼ばない。これは任意Codeではなく、Engine-owned C++ systemへ渡す有限・型付き・権限制約付きdataである。

## 6. GameplayDefinitionの範囲

初期のDefinition kindを次に固定する。

- Rule／Event–Condition–Action。
- Finite State Machine。
- Behavior TreeとBlackboard schema。
- Ability、Status Effect、Cooldown、Cost。
- Quest、Objective、Dialogue、Choice。
- Encounter、Spawn Plan、Wave。
- UI Flow、Screen transition、Input action mapping。
- Camera、Audio、VFXのPresentation cue。
- Item、Weapon、Character、Difficulty、Balance table。

Definitionが利用できる操作はMCDのCapability IDで列挙する。filesystem、network、process、clock、pointer、native SDK、dynamic library、arbitrary reflectionを公開しない。

### 6.1 汎用言語化を防ぐ制約

- 任意Source text、eval、FFI、function pointerを持たない。
- 任意loop、recursion、自己書換え、runtime node生成を持たない。
- State cycleはtick／phase境界を越える明示State transitionとして表す。
- 一つのState Machine instanceは一phaseに最大一回だけauthoritative transitionできる。
- Behavior TreeはDefinitionごとに`max_node_visits_per_tick`を必須とし、Target Profileの上限を超える値はCook errorにする。
- Collection、string、blob、Blackboard slot、active taskはMCDで最大数と最大byteを必須にする。
- 待機はcall stackやcoroutineで保持せず`GameplayTask`へ保存する。
- Physics、Navigation、Asset等の非同期処理はversion付きrequest／resultとして扱う。
- World変更はtyped commandへ追加し、現在のcallbackから直接適用しない。

上限を持たないDefinitionはEditor Preview、Cook、Runtime AIの全経路で拒否する。

## 7. AuthoringからRuntimeまでのData Flow

```text
Natural Language／Manual Editor
  -> Requirement Resolver
  -> GameSpec／GameplayDefinitionChangeSet
  -> Structural Validator
  -> Semantic／Capability／Budget Validator
  -> Preview Diff
  -> Authoring Commit
  -> Runtime Compiler
  -> Canonicalize／Resolve／Index／Pack
  -> CookedGameplayPackage
  -> Isolated GameHost Preview
  -> Package／Sign／Ship
```

AI、Editor GUI、CLI、MCP Clientの入力経路は異なっても、`ChangeSet -> validate -> preview -> commit`以後を共有する。UI widget、LLM、Pluginからlive Worldへ直接writeしない。

## 8. Editor形式とRuntime形式

### 8.1 Editor形式

- `/schemas/mira/`のMCDを正本にする。
- text-diffable、version付き、unknown field拒否。
- StableId、Requirement ID、Capability ID、Budget、Target条件を保持する。
- 人間向け名称や説明と、Runtime用数値IDを分離する。

### 8.2 Cook

Runtime Compilerは次をoffline実行する。

- Schemaとsemantic validation。
- StableId／Asset dependencyの解決。
- String keyからnumeric IDへの変換。
- Event別dispatch indexの生成。
- State／nodeのflat array化。
- 定数条件の畳み込み。
- 未到達State、未参照Definition、重複dataの検出。
- Collection capacity、queue contribution、memory upper boundの計算。
- Capability／State layout／dependency hashの生成。
- Canonical binary encodingとcontent hash。

### 8.3 Runtime形式

Shipping GameHostはAuthoring JSONをGameplay実行経路でparseしない。`CookedGameplayPackage`は次を含む。

- Format version、Project revision、Contract set hash。
- Definition set hash、Capability manifest hash、State layout hash。
- Section offset／length／alignment。
- Flat definition table、event index、constant pool。
- Stable runtime handleとAsset version dependency。
- 各Definitionの実行上限とTarget Profile。

Pointer、vtable、native padding、source path、Editor-only説明、Provider情報を保存しない。

## 9. C++ Runtimeでの実行

- C++23のEngine-owned Gameplay SystemがCooked dataを処理する。
- Hot pathはArchetype Chunk＋SoA queryをbatch処理する。
- Entityごとのvirtual `Update()`、個別heap object、文字列dispatchを標準方式にしない。
- Definitionはevent indexから対象だけを選び、全Ruleを毎tick走査しない。
- Workerはimmutable Definition／snapshotからprivate outputを作り、Worldへ直接writeしない。
- Authoritative state writeは`GameplayStateStore`のdelta journal、World変更はtyped commandへ記録する。
- 正常終了と全Validator合格後にだけdeltaとcommandをsealする。

Gameの表現力が固定Capabilityで不足する場合は、Definitionへ汎用命令を追加して言語化せず、CapabilityをC++で追加する。

## 10. AIの実装方式選択

AIはSystemごとに次の順序を必須とする。

1. 既存Component、Asset、GameplayDefinition、Capabilityのcompositionで実現する。
2. DefinitionのCook、index、batching、Asset layoutを最適化する。
3. 表現不能な新規Gameplay Algorithm、または計測済みhot pathだけ`NativeGameModule`を提案する。
4. Platform／Native SDK統合、またはEngine-wide再利用が必要な機能はProject Moduleへ入れず、Engine／Platform AdapterまたはEngine Capability変更をR4として提案する。

Genre名、Gameの総規模、Modelの主観だけでC++を選ばない。選択Proposalは次を持つ。

- Requirementと未充足Capability。
- 同時instance数、呼出頻度、latency、memory、determinism。
- Target intersectionとMobile／Store影響。
- 構造化方式を採用または不採用にする理由。
- Test、Benchmark、Rollback。
- C++公開APIと変更Path。

## 11. NativeGameModule

本章はGame実装方式上の境界を定める。ABI field、entry point、lifecycle、Target別link、Build／Promotion／restart、failureの詳細は[NativeGameModuleアーキテクチャ規約](./2026-07-19-native-game-module-architecture-design.md)を基準とする。

### 11.1 境界

`NativeGameModule`は次の論理公開契約だけを使用できる。

- `mira.foundation`、`mira.runtime.contracts`、`mira.gameplay`、`mira.native_game`とProject生成Contract。
- MCDから生成したwire type、validator、typed command／event。
- 宣言済み`ComponentAccessManifest`のquery／lease。
- Engine allocatorを固定C function tableへ投影した`MiraNativeMemoryPortV1`。STL／PMR object自体は境界へ渡さない。
- Versioned Asset／Entity／Resource handle。

CX0ではContract compilerとCMakeが論理依存を`include/mira/`と生成Headerへ投影し、CX3ではPrimary Named Moduleと`import std`へ投影する。Project SourceがHeader／Module構文を選ぶのではなく、`CppDependencySetV1`、Active `CxxFrontendProfileV1`、Source scanを一致させる。CX3後はEngine C++ Public Header projectionを生成しない。

次を禁止する。

- Engine／vendorのprivate header。
- D3D12、Vulkan、Metal、Box2D、Jolt、Recast等のnative型。
- World pointer、owning raw pointer、global service locator。
- phase外mutation、callback再入、unbounded thread／allocator。
- Project独自のPlatform／Store code。

### 11.2 BuildとPreview

- `NativeCodeChangeSet`はR3とする。
- AI生成Sourceは隔離Source WorkerでBuildする。
- Format、warning-as-error、primary／secondary compiler、static analysis、ASan、unit、integration、performanceを通す。
- Editorへ生成binaryを直接注入しない。
- Windows Developmentは別Processの新しいGameHostが、起動時にversioned C ABI entryを持つGame DLLを一度だけloadする。変更時はGameHostを終了して新Processへ置き換え、in-process unload／reloadを実装しない。
- Windows ShippingはEngineとGame moduleを同一EXEへ静的linkする。Development用DLLまたはPreview artifactをShippingへ昇格しない。
- AndroidはGame moduleをstatic archiveとしてGame shared libraryへlinkし、AppleはGame moduleをapp executableへ静的linkする。C++変更後は完全なrebuild／package／installを必須とする。
- ABI境界へSTL object、exception、RTTI、allocator ownership、Engine／vendor native objectを渡さない。POD wire view、versioned handle、caller-owned buffer、function tableだけを使用する。

隔離Buildは、生成されたnative codeがGameHost内でsandbox化されることを意味しない。Promotion後のC++はProcess内memoryへ到達できる信頼済みCodeであるため、Code owner Reviewと全Gateを通過するまで正規Projectへ昇格しない。

## 12. Performance方針

構造化Authoring自体をRuntime overheadにしない。性能は次で担保する。

- Authoring形式とRuntime binaryの分離。
- Cook時の参照解決、定数畳み込み、索引化、packing。
- SoA、batch query、bounded arena、handle。
- fixed phase、immutable snapshot、private job buffer。
- Runtime allocation、branch、cache miss、contentionのtelemetry。
- Development／Profile／Shippingの分離。
- Profile根拠のないpool化、lock-free化、C++化を禁止。

`windows_desktop_v1`の60 fps基準はCPU／GPU P95 14.00 ms soft、16.67 ms hardとする。`Gameplay Logic`はP95 1.50 ms soft capを維持する。構造化実装がBudgetの80%以下かつdeadline miss 0なら維持し、80～100%はCook／index／layout最適化とC++候補を同一fixtureで比較する。100%超またはdeadline miss発生時はC++候補を作る。C++化の改善が5%未満または測定Noise内なら、構造化実装を維持する。

## 13. Memory方針

Luau廃止により、VM heap、bytecode cache、GC reserve、Script stack、userdata、coroutine、binding copyを持たない。

既存のWindows 2 GiB Engine-controlled CPU budgetにあったScript Domain 128 MiBは削除する。128 MiBは`Unassigned headroom`として保持し、Domainへ自動再配分しない。GameplayDefinitionは次へchargeする。

- Authoring source／Cook staging: Editor／Asset build budget。
- unloaded Cooked package: Asset streaming cache。
- active immutable Definition: Core World／Save budget。
- instance state／task／delta: Core World／Save budget。
- per-job evaluation scratch: Frame／Job transient budget。

Headroom再配分はReference fixture、Before／After、peak memory、allocation count、10分soak、人間Reviewを持つADRだけで行う。

## 14. Save、Replay、Hot Reload

### 14.1 Save／Replay

- Authoritative Gameplay stateは`GameplayStateStore`だけに置く。
- SaveへDefinition source、pointer、C++ object layoutを保存しない。
- SaveはDefinition StableId、State ID、typed field、task、versionを保存する。
- Replayはinput、accepted async result、state delta、command、faultをcanonical順で記録する。
- RNG seedとdraw countをstate hashへ含める。

### 14.2 Live Edit

`GameplayDefinitionSet`はdependency closure全体をstagingし、次を満たす場合だけ`T00`でswapできる。

- Capability manifest hashが一致する。
- State layout hashが一致する。
- Stable State／node IDが維持される。
- Cook、semantic test、deterministic replay fixtureが合格する。

一つでも不一致なら旧versionを維持し、`restart_play`を返す。C1では自動State migrationとNativeGameModule hot unloadを行わない。

## 15. Editor UX

### 15.1 Beginner Workspace

- 自然言語、Game Brief、質問、進捗、Play Test結果。
- TemplateとFormによるGameplayDefinition編集。
- 変更前後のDiff、Risk、戻し方、検証結果。
- C++を表示せずFirst Playableを作成可能。

### 15.2 Standard Workspace

- Viewport、Hierarchy／Outliner、Inspector、Asset、Console、AI Panel。
- GameplayDefinitionのGraph／Table／Form projection。
- AI Panelのdock、pin、auto-hide、別Window。

### 15.3 Advanced Workspace

- Standard機能に加え、C++、Shader、Build、Profiler、Memory、Render Graph、Diagnostics。
- AI生成C++のSource Diff、Requirement、Test、Benchmark、Review状態。
- 手動変更もAI変更と同じChangeSet、Validator、Historyを使用。

Graphは正規dataの一Projectionであり、Graph fileを別の権威あるprogramにしない。

## 16. Editor制作型とRuntime生成型

### 16.1 Editor制作型

AIはGameplayDefinition、Scene、Asset、Material、Test、NativeGameModuleを生成できる。構造化変更はValidator／Cook、C++変更は隔離Build／Reviewを経てから試遊する。

### 16.2 Runtime生成型

Shipping Runtime AIが変更できるのは、出荷済みC++ CapabilityとSchemaが許可する構造化dataだけである。

許可例:

- Dialogue、Quest、NPC intent。
- Spawn plan、Encounter、Wave。
- 数値調整、既存Abilityのcomposition。
- Weather、time、Presentation cue。
- bounded World構成とGameStateDelta。

禁止:

- C++／native／managed code、Shader、bytecodeの生成・download・load。
- Capability、Schema、Budgetの追加。
- raw Physics／Render／Network stateの確定。
- arbitrary process、shell、filesystem、network、FFI。
- Client内へのProvider API key埋込み。

## 17. Failure方針

| Failure | Authoritative変更 | 処理 |
|---|---|---|
| Definition schema／semantic不正 | なし | field単位Diagnostic、Commit拒否 |
| Capability不足 | なし | `UnsupportedCapability`、C++提案は別ChangeSet |
| Cook失敗 | live set変更なし | 旧artifact維持 |
| State layout非互換 | live set変更なし | Play restart要求 |
| Evaluation step／queue budget超過 | tick publishなし | Editor Play停止、Shipping session fault |
| C++ compile／test失敗 | 正規Source／binary変更なし | staging破棄、修正案 |
| C++ crash／sanitizer failure | Promotionなし | Artifact隔離、Receipt不合格 |
| Runtime AI不正data | World変更なし | Proposal拒否、audit ID記録 |

Authoritative event、state delta、commandを性能のため黙ってdropしない。

## 18. TestとRelease Gate

最低限、次を自動化する。

- GameplayDefinition valid／invalid／boundary fixture。
- Canonical JSON／binary round-trip。
- Event index、State transition、Behavior visit上限。
- Deterministic Save／Load／Replay state hash。
- GameplayStateStore transaction rollback。
- Cooked package hash、dependency closure、atomic promotion。
- NativeGameModuleのCX0 Public Header／CX3 Primary Module、CMake／Module DAG、access manifest conformance。
- Generated C++ static analysis、ASan、unit、integration。
- 2D／3D Reference sceneのP50／P95／P99、allocation、cache miss。
- 10分soak、Mobile 30分thermal、2時間endurance。
- Shipping packageにGame／Shader source、Compiler、汎用言語VM、汎用Game bytecode、credentialがなく、承認済みoffline compiled shaderだけが含まれること。
- Runtime AIがSchema外Field、Capability、実行Codeを拒否するnegative test。

## 19. PhaseとMVPへの反映

### Phase 0

- MCDへGameplayDefinition、Capability、State layout、Budgetを定義。
- MCDへ`CxxFrontendProfileV1`と`CppDependencySetV1`を定義し、CX0 Header projectionとCX1 Module projectionを同じContractから生成。
- `mira.foundation`のC++23／Named Module／`import std` ProbeとC++26 readiness CIを構築。
- C++／TypeScript／MCP／Provider／Cooked binary projectionを生成。
- GameplayDefinition validator、canonical codec、minimal evaluator fixture。
- NativeGameModule public SDK、Source Worker、Promotion Gateを定義。

### 2D Manual First Playable

- Rule、Quest、Ability、UI FlowをGameplayDefinitionで作成。
- C++ Gameplay SystemがCookedGameplayPackageを実行。
- Script VMなしでTitleからResultまで完走。

### AI Authoring MVP

- AIが構造化方式を第一選択にする。
- 未充足Capabilityまたは計測済みhot pathだけC++を提案。
- GameplayDefinitionChangeSetとNativeCodeChangeSetを別Riskで表示。
- 初心者がC++を見ずに生成・調整・再試遊できる。
- 上級者が同じProjectでC++を直接編集できる。

## 20. Pre-1.0 Clean変更

実装開始前の設計変更であるため、Luau互換層、migration、deprecated alias、空の`ScriptRuntime` interfaceを残さない。

- Luau dependency、source、license entry、version lockを削除する。
- `engine/scripting/`を作らない。
- MCDのLuau projectionを作らない。
- Script memory／performance／telemetry／testをGameplay Logicへ置換する。
- 既存計画書の「Script＋C++」を「構造化ゲームデータ＋C++」へ置換する。
- 旧Cooked artifactは存在しないためmigration対象にしない。

将来Script言語を検討する場合は、本設計の暗黙拡張ではなく、新規ADR、Threat Model、Memory／Performance、Editor／Debugger、全Target Store gateを持つ別Product判断とする。1.0に汎用`ScriptRuntime`抽象を先回りして実装しない。

## 21. Definition of Done

本方式は次をすべて満たした時だけ実装開始可能とする。

1. 正式仕様にLuau、Game Script VM、ScriptRuntimeの依存が残っていない。
2. GameplayDefinition、CookedGameplayPackage、GameplayStateStore、NativeGameModuleのMCD責務が一意である。
3. Editor形式をShipping hot pathで直接parseしない。
4. 構造化dataの全collection、step、queue、memoryがboundedである。
5. AIの構造化／C++選択がCapabilityとBenchmark Receiptを参照する。
6. C++生成がR3 Source Gateと隔離Buildを通る。
7. Desktop Previewが別GameHost Processを使い、C++ hot unloadへ依存しない。
8. Mobile Shipping Runtimeがdata-only generationを強制する。
9. Gameplay LogicがP95 1.50 ms、全体14.00 ms soft／16.67 ms hardを満たす。
10. Save、Replay、Hot Reload、failureがDefinition versionとState layoutで検証される。
11. Beginner／Advanced Workspaceが同じChangeSetとHistoryを使う。
12. 2D Manual First PlayableがScript VMなしで完走する。
13. AI生成C++の論理依存、実際のimport／include、CMake DAGが一致し、未宣言依存とModule cycleが拒否される。

## 22. 一次資料

- [Epic Games: Coding in Unreal Engine — Blueprint vs. C++](https://dev.epicgames.com/documentation/unreal-engine/coding-in-unreal-engine-blueprint-vs-cplusplus?lang=en-US)
- [Epic Games: Data Assets in Unreal Engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/data-assets-in-unreal-engine)
- [CRYENGINE: Custom C++ Entity Game Sample](https://www.cryengine.com/docs/static/engines/cryengine-5/categories/23756813/pages/25530072)
- [Unity: スクリプトの概要](https://docs.unity3d.com/jp/current/Manual/intro-to-scripting.html)
- [Unity: Native Plugin](https://docs.unity3d.com/ja/current/Manual/plug-ins-native.html)
- [Unity: IL2CPPの概要](https://docs.unity3d.com/jp/current/Manual/il2cpp-introduction.html)
- [Unity: Visual Scripting](https://docs.unity3d.com/ja/current/Manual/com.unity.visualscripting.html)
- [Godot: Scripting languages](https://docs.godotengine.org/en/stable/getting_started/step_by_step/scripting_languages.html)
- [Godot: C++ GDExtension example](https://docs.godotengine.org/en/stable/tutorials/scripting/cpp/gdextension_cpp_example.html)
- [O3DE: Scripting Gameplay](https://docs.o3de.org/docs/user-guide/scripting/)
