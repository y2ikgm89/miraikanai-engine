# Miraikanai Engine Domain Pack／将来Capability規約

- 文書版: 1.0
- 作成日: 2026-07-19
- 対象: 多Genre対応、Domain Pack、Template、AI vocabulary、将来Core Capabilityの設計入口
- 状態: プロジェクト公式の規範設計レビュー版
- Product設計: [AIネイティブ独自ゲームエンジン 設計計画書](./2026-07-18-ai-native-game-engine-authoring-design.md)
- 機能範囲: [Miraikanai Engine 2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)
- Game実装規約: [Miraikanai Engine C++実行コード・構造化ゲームデータ規約](./2026-07-19-cpp-structured-game-data-design.md)
- Native Game規約: [Miraikanai Engine NativeGameModuleアーキテクチャ規約](./2026-07-19-native-game-module-architecture-design.md)
- 契約規約: [Miraikanai Engine 実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)

## 1. 結論

FPS／TPS、RPG／Action RPG、Simulation、Strategy、2D ActionをEngine Coreの継承階層へ組み込まない。CoreはWorld、Render、Physics、Navigation、Animation、Input、UI、Audio、Asset、Gameplay Capabilityを提供し、Genre固有の語彙、Template、Rule、Validator、Test、AI制作手順はversioned `DomainPack`として構成する。

Domain PackはEngine forkでも、任意binary pluginでもない。MCDで検証可能なpackageであり、必要なProject C++ source templateを含める場合も通常の`NativeGameModule` Build／Review／Promotionを通る。

Multiplayer、large world、terrain／foliage、ray tracing、商用品質の内蔵AI Asset生成は「多Genre対応」に暗黙包含しない。それぞれを独立Capabilityとして設計、脅威分析、vertical slice、budget、Target gate後に昇格する。WaterとWeather／Snow SurfaceはこのGateを通過して専用正式仕様へ昇格したが、Production表示は各仕様のC1／C2 Qualification完了後に限る。

## 2. 決定権とDomain Packの正規構造

| 主題 | 正本 |
|---|---|
| Genre語彙、Template、Requirement質問、Validator、reference scenario、Pack更新 | 本書 |
| Core Capabilityの型、phase、memory、Target、failure | 各Subsystem正式仕様 |
| GameplayDefinition／C++選択とScript VM不採用 | Game実装規約 |
| Project C++のBuild、ABI、Promotion | Native Game規約 |
| Pack schema、operation、generated validator | 契約規約 |
| Pack適用、revision、Undo、migration transaction | Authoring Model／Project State規約 |

Domain PackがCore正式仕様の上限を緩和したり、未実装Capabilityを存在するように宣言したりしてはならない。Core Capability追加が必要な場合は本書7節の独立Gateを通し、該当Subsystem正式仕様を先に改訂する。

```text
DomainPackManifest
  pack_id
  pack_version
  minimum_engine_contract
  supported_target_profiles[]
  required_capabilities[]
  optional_capabilities[]
  schema_modules[]
  definition_templates[]
  scene_ui_asset_templates[]
  native_source_templates[]
  validators[]
  test_scenarios[]
  ai_vocabulary[]
  ai_planning_recipes[]
  performance_profiles[]
  migration_steps[]
  license/provenance
```

Pack identityはUUIDv7、versionはSemVer、内容全体はSHA-256で固定する。Pack manifestはRuntimeで任意codeをloadする入口ではない。

### 2.1 Packが所有するもの

- Genre用GameSpec sectionとtyped data
- GameplayDefinition template／composition
- Scene、UI、Input Action、Asset Requirement template
- Capability parameter profile
- Semantic tagとAI vocabulary
- Domain validator、reachability／playability test
- Reference scenarioとPerformance budget
- Tutorial／onboarding／sample
- 必要なNativeGameModule source template

### 2.2 Packが所有しないもの

- World／Entity／Asset／SaveのCore identity
- Renderer／Physics／Nav／Animation等のnative backend
- Platform SDK／Store／Network transport
- AI権限、Command Gateway、validator bypass
- Engine private header、vendor object
- 無制限なScript／expression／callback

Pack間共有が必要な機能はversioned Core Capabilityまたは別のreusable Feature Packへ抽出し、Genre Pack同士を直接linkしない。

## 3. Install、Project適用、更新

1. Pack artifact、license、signature／hash、Engine contract rangeを検証する。
2. 必要CapabilityとTarget intersectionを検証する。
3. Templateから作成するDocument、Asset、Definition、C++ sourceのPreview Diffを生成する。
4. Project既存ID、name、Input、Save、UI、Styleとのconflictを検証する。
5. 一つの`ApplyDomainPackChangeSet`としてCommitする。
6. Cook、Build、Pack test scenarioを実行する。

ProjectはPackのlive mutable objectを参照せず、適用したtemplate instanceと`PackApplicationRecord`を保持する。Pack更新は三者比較Diffで行い、人間override、AI変更、Save fieldを無断で上書きしない。Major updateはoffline migratorと明示承認を必須とする。

一Projectで複数Packを使えるが、各Packのrequired Capability、Input Action、Save field、UI route、semantic tag conflictをApply前に検査する。

## 4. AIとの契約

`AiDomainVocabulary`は次を対応付ける。

```text
user_term
canonical_concept_id
required_questions
candidate_capabilities
default_profile
validation_scenarios
forbidden_assumptions
```

例えば「銃を撃てる」はhitscan／projectile、弾薬、friendly fire、aim、platform input、visual violence表現を分離する。AIはGenre名だけで方式を決めず、Game上のHigh Impact質問だけをUserへ返す。

Packの`AiPlanningRecipe`はPrompt本文ではなく、Requirement→Document／Capability／Testのtyped mappingである。AI Providerが変わっても同じGateway／Validator／Testを使う。

AIはPackを選択または提案できるが、未導入PackのCapabilityを存在するものとして生成しない。Pack追加、Major更新、Native source template適用はRiskとDiffを表示する。

## 5. 初期Domain Pack

### 5.1 `mira.domain.2d_action.c1`

最初の2D top-down action sliceを完成させる必須Packとする。

- player move／aim
- enemy seek／attack
- health／damage／invulnerability window
- projectile／hitbox
- pickup／score
- wave／spawn
- pause／result／restart
- keyboard／mouse／controller／touch Action template
- Pixel 2D Scene／UI／Audio／VFX template
- reachability、60秒play、damage、save／reload test

One-way platform、platformer motor、cutout animationはC2 subprofileとしC1へ混ぜない。

### 5.2 `mira.domain.tps_single_player.c1`

3D compact arena用のsingle-player TPS Packとする。

- third-person camera／camera collision
- Character Motor／locomotion／root motion policy
- aim／lock-on optional
- hitscanとprojectileのtyped weapon
- ammo／reload／fire rate
- health／damage／team／hit reaction
- simple perception／combat behavior
- spawn／checkpoint／result
- controller／keyboard Action template
- 3D arena reachability、aim、camera、60 fps test

Network replication、lag compensation、server authority、matchmaking、voice chatを含めない。FPS Cameraは同じcombat capabilityを使うC2 view／weapon presentation profileであり、networkingの意味ではない。

### 5.3 `mira.domain.rpg_arpg.c1`

TPS C1後のProduction候補とする。

- typed Attribute／derived stat
- Ability／cost／cooldown／cast
- Status effect／stack／duration
- Item／inventory／equipment／loot table
- Quest objective／state／reward
- Dialogue graph／choice／condition
- NPC／faction／interaction
- save／load／migration fixture
- character sheet、inventory、quest、dialogue UI template

Economy、craft、skill treeはC2 Feature Packとする。MMO、online trade、backend account、UGCは含めない。

### 5.4 `mira.domain.simulation.c1`

RPG／ARPG C1後のProduction候補とする。

- simulation clockとtime scale
- bounded agent record／schedule／need
- spatial query／zone
- data table／formula graph
- resource／producer／consumer
- transaction ledger
- deterministic seeded event
- aggregation LOD
- chart／table／inspect UI
- long-run invariant、save、replay、performance test

Agent全体をEntityごとのvirtual updateにせず、SoA batch、event queue、time wheel、bounded queryを使う。Flow field、hierarchical scheduling、economy solverはC2。100万active agent、distributed simulation、cross-platform lockstepはC3である。

## 6. Pack別完了条件

| Pack | Reference result | 必須Gate |
|---|---|---|
| 2D Action C1 | 5分遊べるtop-down action | 2D C1、manual＋AI作成、save、mobile |
| TPS C1 | 5分遊べるcompact arena | 3D C1、60 fps Windows、camera／combat |
| RPG／ARPG C1 | 15分vertical slice | save migration、quest／item／ability invariant |
| Simulation C1 | 30分継続するsmall simulation | deterministic replay、long-run invariant、batch budget |

「Templateが生成できる」だけでは完了しない。初心者PromptからFirst Playable、人間手動調整、AI再編集、Package、Reference Target playtestまでを合格する。

## 7. 将来Core Capability

### 7.1 Multiplayer／Online

状態: **C3、設計未開始。現在のEngine Contractへ含めない。**

Production化前に別正式仕様で最低限、次を決める。

- authority／trust／threat model
- transport、session、identity、encryption、key
- replication schema、interest、bandwidth
- prediction、reconciliation、lag compensation
- server tick、hosting、migration、version compatibility
- cheat／abuse、privacy、moderation
- save／backend／offline behavior
- load／packet loss／latency／security fixture

Single-player FPS／TPSやNPC NavigationをMultiplayer対応と表現しない。

### 7.2 Large World／Streaming World

状態: **C3。C1の10 km hard spatial rangeを超えない。**

必要設計はfloating origin、World cell、stable global coordinate、cross-cell entity、Asset／Nav／Physics／Render streaming、save、teleport、Editor collaboration、precision、memory／I/O budgetである。

### 7.3 Terrain／Foliage

状態: **C2／C3候補。一般Mesh／Materialで代用可能な小規模SceneだけC1。**

- Terrain: height／mesh source、LOD、collision、nav、paint、streaming
- Foliage: scatter rule、seed、instance LOD、wind、collision subset

各SubsystemがRendererだけでなくAsset、Physics、Nav、Editor、AIへ跨るため、個別正式仕様なしに「描画Effect」として追加しない。Waterは[Water Surface Platform規約](./2026-07-20-water-surface-platform-architecture-design.md)、降雪／積雪は[Weather／Snow Surface規約](./2026-07-20-weather-snow-surface-architecture-design.md)へ昇格済みであり、本節の将来候補には含めない。

### 7.4 Ray Tracing

状態: **C3。Raster C1／C2のhard requirementにしない。**

DXR／Vulkan RT／Metal RTのTarget intersection、BLAS／TLAS lifetime、Shader／Material、denoise、fallback、GPU memory、Reference hardwareを別仕様で決める。AIはCapabilityがProduction Manifestへ載るまで選択できない。

### 7.5 Production AI Asset Generation

状態: **C2 Provider選定前。安全なImport受入口はAsset規約C1で実装する。**

画像、Audio、3D、Animationを一つの「生成Asset」機能にまとめず、mediaごとに次を満たす。

- Provider capability／exact model／region／cost／rate limit
- input／output formatとsize
- commercial-use terms snapshot
- training／reference Asset権利とconsent
- content safety／privacy
- Style consistency／technical validation
- C2PA／provenance／license
- retry／provider outage／manual replacement
- offline／no-provider Project behavior
- quality Evalと人間Review

Provider選定はArchitectureの空欄ではなく、C1でProvider-neutral stagingとGateを完成させた後、実在条件を比較してADRで一つずつProduction化する順序である。

## 8. Phase

| 順序 | Deliverable |
|---:|---|
| 1 | Core Subsystemと2D Action C1 |
| 2 | AI Creatorから2D Pack適用、Windows／Mobile Package |
| 3 | 3D C1とTPS single-player C1 |
| 4 | Realistic advanced、Toon、Pixel DioramaのProduction Profile |
| 5 | RPG／ARPG C1 |
| 6 | Simulation C1 |
| 7 | media別AI Asset Provider C2 |
| 8 | Terrain／Foliage C2候補。Water／Snow C1・C2は専用正式仕様のGateで昇格 |
| 9 | Multiplayer、Large World、Ray Tracingは各正式仕様とprototype承認後 |

複数Genreを理由に最初の2D／3D slice前から全Packを同時実装しない。

## 9. Security、Performance、Compatibility

- Pack Sourceはuntrusted inputとしてschema、path、license、signature／hashを検査する。
- Packが任意native binary、Script、post-install executableを含むことを禁止する。
- Native source templateはR3、Engine Capability変更はR4である。
- PackごとにCPU／GPU／memory／queue／Asset予算を持ち、Core budgetを拡張しない。
- Pack schema／Save fieldはStable numeric IDを使い、renameで壊さない。
- Pre-1.0 Major変更はoffline migratorを提供し、互換shimをRuntimeへ積まない。
- Pack removal前にProject reference、Save field、Asset closureを検査し、使用中Packを強制削除しない。

## 10. Failure policy

| Failure | 結果 |
|---|---|
| Manifest／signature／hash不正、dependency cycle | Installを原子的に拒否し、既存Pack registryを維持 |
| Engine／Pack version range不一致 | `IncompatiblePackVersion`で拒否。互換shimを自動生成しない |
| Project Targetに必要Capabilityがない | Apply／Cookを拒否し、不足Capabilityと対応Targetを列挙 |
| 複数PackのID、Input、Save、UI、Asset path衝突 | Conflict Diffを提示し、部分適用しない |
| Template Operationのprecondition不一致 | `ProjectChangeSet`全体を拒否し、最新revisionで再生成を要求 |
| Native source templateを含む | R3 `NativeCodeChangeSet`として隔離Build／Review／Promotionへ送り、Pack適用と同時昇格しない |
| Pack更新migration失敗 | 新revisionをCommitせず、旧Pack versionとProject Documentを維持 |
| Pack removal時にlive reference／Save dependencyあり | Removalを拒否し、依存closureを列挙 |
| 未Productionの将来Capabilityを要求 | Blocking Capability gapとして質問または対応済み代替案を提示し、偽のplaceholder成功にしない |

Package済みGameがPack registryをnetworkから暗黙更新することを禁止する。Data-only Runtime content更新はAsset／Content Package規約の署名、Target、Package、rollbackを通り、Native codeを含めない。

## 11. TestとDefinition of Done

- Pack install、apply、update、remove、conflict、migration
- 複数PackのCapability／Input／Save／UI collision
- AI vocabularyから正しい質問／Requirement／Testを生成
- Beginner Prompt→First Playable→manual edit→AI edit→Package
- Pack Sourceがvalidator／Risk／Native Source Promotionを迂回できない
- Reference scenarioのPerformance、save／load、replay、10分／30分soak
- 未Production CapabilityをAIが選択できない
- Multiplayer等のdeferred機能がC1 Capability Manifestへ混入しない

Domain Pack C1 frameworkは2D ActionとTPS Packが同じCore API、同じProject format、同じAI／manual workflowで独立に適用・更新・検証できた時点で完了する。

## 12. 一次資料

- [Unreal Engine Game Features and Modular Gameplay](https://dev.epicgames.com/documentation/unreal-engine/game-features-and-modular-gameplay-in-unreal-engine)
- [O3DE Gems](https://docs.o3de.org/docs/user-guide/gems/)
- [O3DE Create a Gem](https://docs.o3de.org/docs/user-guide/programming/gems/creating/)
- [Godot Making plugins](https://docs.godotengine.org/en/stable/tutorials/plugins/editor/making_plugins.html)

これらの公式資料は、機能、Asset、Editor拡張を選択可能な単位へ分離する先例の確認に使う。Miraikanaiの`DomainPack`はそれらのPlugin／Gem形式を実装互換で模倣せず、任意binaryをloadしないMCD package、共通ChangeSet、Capability Gate、AI vocabulary、reference scenarioを一体化した独自Product契約とする。
