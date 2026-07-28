# ChatGPT 5.6 Pro 第1次アーキテクチャ監査記録

- 取得日: 2026-07-29
- 対象: `docs/architecture` 全76 Markdown文書
- 入力ZIP SHA-256: `e8167020905042fde042d1302558c0252fb67aba9741f4fb59c7182f139c2882`
- 監査チャット: https://chatgpt.com/c/6a68ddb8-aee4-83ee-b4e3-a0508f4aa2da
- 位置づけ: 外部モデルによる非規範レビュー記録。Owner文書、Decision、実装状態、Qualificationを置換しない。
- 保存形式: ブラウザに表示された回答本文の全文を、見出し装飾を追加せずプレーンテキストとして保持。

---
Miraikanai Engine 第1次アーキテクチャ監査 — Part 1/1

監査日: 2026-07-29
対象: miraikanai-architecture-docs.zip
ZIP SHA-256: e8167020905042fde042d1302558c0252fb67aba9741f4fb59c7182f139c2882

1. 監査前提、読了範囲、評価基準
1.1 ZIP展開・読取確認

ZIPは正常に展開でき、圧縮データの完全性検査でもエラーは検出されなかった。

検査項目	結果
Markdown文書数	76
Markdown以外の収録ファイル	0
総サイズ	4,174,006 bytes
総行数	40,646行
UTF-8 strict decode成功	76／76
NUL文字混入	0
読取不能ファイル	0
ZIP CRC／展開エラー	0

読めないファイルはない。
全76文書について、本文、見出し、Owner Header、規範依存、関連文書、相対リンク、状態表現、Capability／Gate／Evidence／Authority記述を検索・横断照合した。

ただし、本監査が確認したのは添付ZIP内のdocs/architecture相当コーパスだけである。Engine実装、生成Schema、Fixture、Receipt、CI設定、Repository全体、添付外のファイルは監査対象ではない。特に、Decision文書から参照される../../../.codex/config.tomlは添付コーパス外であり、実Repository上の存在を確認できない。

1.2 読了範囲

構成は次のとおりである。

区分	件数	読了範囲
Product Owner	1	00-product/product-plan.md
Governance Owner	3	01-governance/*.md
Foundation Owner	8	02-foundation/*.md
Authoring Owner	6	03-authoring/*.md
Runtime Owner	6	04-runtime/*.md
Simulation Owner	4	05-simulation/*.md
Rendering Owner	12	06-rendering/*.md
Platform Owner	7	07-platform/*.md
Pack Owner	4	08-packs/*.md
Owner文書合計	51	全件
Appendix／Proposal／Review	18	appendices/*.md
Decision関係	6	decisions/*.md
Index	1	README.md
合計	76	全件

読了したOwner文書は以下の51件である。

Product: 00-product/product-plan.md

Governance: 01-governance/architecture-governance.md、ai-security-approval.md、ai-verification-provenance.md

Foundation: 02-foundation/core-architecture.md、toolchain-dependencies.md、executable-contracts.md、compatibility-evolution.md、naming-project-layout.md、cpp23-modules.md、math-core.md、memory-pointers.md

Authoring: 03-authoring/project-state.md、asset-lifecycle.md、editor-ui-framework.md、editor-workspace-ux.md、gameplay-programming-model.md、native-game-module.md

Runtime: 04-runtime/entity-component-system.md、scheduling-lifetime.md、runtime-package.md、persistence-save.md、performance-capacity.md、debugging-observability-replay.md

Simulation: 05-simulation/collision.md、physics.md、navigation.md、animation.md

Rendering: 06-rendering/render-graph.md、materials.md、project-shader.md、lighting.md、post-processing.md、vfx-authoring.md、vfx-runtime.md、camera.md、environment-surfaces.md、lod.md、virtualized-continuous-geometry.md、world.md

Platform: 07-platform/windows.md、mobile-common.md、android.md、apple.md、input.md、audio.md、ui-text-localization-accessibility.md

Packs: 08-packs/pack-contract.md、gameplay-features.md、scenario-stage.md、shooter.md

18 Appendix、5 Decision本文、decisions/README.md、ルートREADME.mdについても全文を照合した。

1.3 状態の扱い

README.mdの§1 目的と非規範性および各Owner Headerに基づき、本監査では次を前提とする。

Owner文書51件の申告状態はすべて文書状態=review。

実装状態はすべてabsent。

Header上の検証状態はdesign-reviewed。

design-reviewedは文書自身の状態申告であり、本監査が独立して採択、実装可能性、動作、性能、Qualificationを証明したことを意味しない。

本文中の型名、Registry、Operation、固定値、Receipt、Gateは、明示された外部Artifactを除き設計候補である。

currentと記載されていても、文書上のcurrent authorityやsource baselineを意味する場合があり、実働実装の存在とは限らない。

target、closed-in-target-design、design-reviewedをaccepted、implemented、active、qualified、productionへ読み替えない。

本監査では記述を以下に分ける。

文書上の事実: 添付文書に明記されている内容。

監査判断: 複数文書の横断照合から導いた不足、危険、閉鎖条件。

外部比較: 公式一次資料を不足発見にのみ使用した結果。Miraikanaiへの設計移植やAPI採用判断ではない。

1.4 評価基準
評価軸	確認内容
Domain coverage	製品水準の2D／3D Engineに必要な領域が、Ownerまたは明示的非目標へ解決されるか
Owner uniqueness	型、状態、Gate、遷移、固定値、診断、Evidenceの正本が一意か
Authority boundary	Source、Derived、Runtime、Presentation、Platform、Governanceの決定権が混ざっていないか
Cross-owner closure	入出力、identity、revision、failure、cancel、rollback、retire、recoveryが送信側と受信側で一致するか
Current／target分離	current authority、target owner、proposal、future、not activatedが混同されないか
Failure atomicity	失敗時にpartial state、silent fallback、stale adoption、authority escalationを生まないか
Gate／Evidence	合否条件、証拠Owner、freshness、revocation、negative evidenceが明確か
Build／Platform closure	Build、Cook、Package、Install、Launch、Signing、Store、Device qualificationが接続されるか
Security／Privacy	AI以外も含むtrust boundary、consent、data handling、vulnerability対応が閉じるか
QA	unit、integration、content、stress、device、session、release gateの関係が閉じるか
Documentation governance	規範依存、Inline根拠、Inventory、用語、リンク、文書分割規則に適合するか
Intentional non-goals	Multiplayer、追加Platform等を「欠落」と誤判定せず、現MVPの必須要件と区別できるか
1.5 外部一次資料の使用範囲

確認日はすべて2026-07-29である。

Runtime Asset

EpicのUnreal Engine 5.8 Asset Managementは、Assetの発見、非同期load、memory保持、unload、disk／memory auditを明示的な実働領域としている。URL: https://dev.epicgames.com/documentation/en-us/unreal-engine/asset-management-in-unreal-engine。Miraikanaiに同じObject modelやAsset IDを導入すべきという意味ではなく、Source／Cookとは別にRuntime取得・保持・解放のauthorityが必要かを検査するために使用した。
Epic Games Developers

Unity Addressables 3.1.0は、EditorでのAsset管理とRuntimeのload／releaseを分け、依存関係、location、memory allocation、非同期load／releaseを管理対象としている。URL: https://docs.unity3d.com/Packages/com.unity.addressables@3.1/manual/index.html、https://docs.unity3d.com/Packages/com.unity.addressables@3.1/manual/LoadingAddressableAssets.html。MiraikanaiへAddressables方式を移植する根拠には使用していない。
Unity マニュアル
+1

GodotのBackground loading資料では、request、status確認、取得時のblocking条件が区別されている。URL: https://docs.godotengine.org/en/latest/tutorials/io/background_loading.html。latest文書であるためVersion固定根拠には使用せず、非同期取得契約にrequestとcompletion以外の状態が必要かを確認する比較材料に限定した。
Godot Engine documentation

QA／Qualification

Unreal Engine 5.8 Automation Test Frameworkは、unit、feature、content stress、functional、screenshot等を区別している。Gauntletは実際のProject sessionを複数Platformで実行・検証する枠として説明されている。URL: https://dev.epicgames.com/documentation/en-us/unreal-engine/automation-test-framework-in-unreal-engine、https://dev.epicgames.com/documentation/en-us/unreal-engine/gauntlet-automation-framework-in-unreal-engine。これはMiraikanaiのテストフレームワーク選定ではなく、Evidence種別と実機session qualificationが単一の「test」で潰れていないかを検査するために使用した。
Epic Games Developers
+1

Gameplay AI

Unreal Engine 5.8のBehavior TreeとAI Perception資料は、perception、記憶・経過時間、decision条件、action、navigation、abort、execution history／debugが別責務として接続される例を示す。URL: https://dev.epicgames.com/documentation/en-us/unreal-engine/behavior-tree-in-unreal-engine---overview、https://dev.epicgames.com/documentation/en-us/unreal-engine/ai-perception-in-unreal-engine。MiraikanaiへBehavior Treeを必須化する根拠には使用せず、既存のPerception／FSM記述から行動選択・中断・診断までが閉じているかを調べる比較材料に限定した。
Epic Games Developers
+1

2D Rendering

Unity 6.2の2D sorting資料は、Sorting Layer、Order、Render Queue、Camera distance等を別の順序決定要素として扱う。URP 2DのTilemap資料は、Tilemap、2D lighting、Transparency sorting、batchingの接続を明示する。URL: https://docs.unity3d.com/6000.2/Documentation/Manual/2d-renderer-sorting.html、https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@16.0/manual/2D/tilemap-renderer-2d-renderer.html。これはUnityのComponent/APIを採用する根拠ではなく、Miraikanaiの2D契約がどの責務を一意所有すべきかを検査する比較材料である。
Unity マニュアル
+1

Product-wide Security

NIST SSDF 1.1は、Secure Software Developmentを個別機能ではなくSDLC全体へ統合する高水準Practiceとして位置付けている。URL: https://csrc.nist.gov/pubs/sp/800/218/final。Miraikanaiへ特定Processをそのまま導入するためではなく、AI Securityだけで製品全体の脆弱性管理が閉じたと扱ってよいかを検査するために使用した。
NISTコンピュータセキュリティリソースセンター

1.6 第1次監査の総合判定

全コーパスは、Owner分離、typed reference、fail-closed、current／target分離、Target別Qualification、Evidence chainという設計思想では非常に充実している。一般的な初期Engine文書よりも、failure、cancel、stale revision、hash、provenance、device loss、accessibility、Platform package境界まで深く扱っている。

一方で、現在の製品MVPを文書上閉じるために必要なのにOwnerが未materializeであるCapabilityが3領域存在する。

RPG-first MVPを成立させるReusable RPG Feature／RPG Genre composition／Reference Game authority

汎用Runtime Asset request／residency authority

汎用Consent Record authority

さらに、Runtime ECSはtarget文書自体の内容不足ではなく、current authorityからtarget Ownerへの移管が未閉鎖であるため、別のCritical blockerとなる。

したがって、現コーパスを次のように評価する。

Architectureの方向性: 強い

Subsystem coverage: 広い

安全境界: 強い

文書上の一意Authority: 一部未閉鎖

Current／target migration: 一部未閉鎖

MVP Architecture closure: 未達

実装／Qualification／Product Activation: 添付文書自身の申告どおりabsentまたは未materialize

2. 製品レベル2D／3D EngineとしてのDomain coverage matrix

以下の「十分」は、review状態のtarget設計として十分という意味であり、実装済みまたはqualifiedという意味ではない。

Domain	判定	主な根拠文書	監査判断
Product intent／製品差別化	被覆・境界修正要	00-product/product-plan.md §1、§2、§5	AI Authoring vertical sliceとMVPの意図は明確。ただしArchitectureとexecution planningが混在
2D／3D Portfolio	十分	product-plan.md §4	2Dをdepth-zero 3Dへ縮退しない方針、2D／3Dの独立Capabilityが明記される
Capability成熟度	十分	product-plan.md §3	C0–C3とActivation stateが分離される。ただし他のstate語彙との統合説明が不足
Architecture Governance	被覆・Closure未了	01-governance/architecture-governance.md	原則は強いが、Inline根拠適合、Inventory、依存層順序に未閉鎖事項
Architecture Inventory	Closure未了	architecture-governance.md §5、README.md §1	Generator、Schema、生成Artifactが存在せず、手動Index
Architecture Explain Projection	Closure未了	architecture-governance.md §5.3	targetは記述されるが、currentの一意な機械解決経路が未materialize
Core layer／Host／Process	十分	02-foundation/core-architecture.md §3–§10	Authoring／Runtime、Host、Gateway、Repository境界が詳細
Executable contracts	十分・materialization未了	02-foundation/executable-contracts.md	operation、state machine、diagnostic、projectionの設計は広い
Compatibility／Evolution	十分	02-foundation/compatibility-evolution.md	clean break、format evolution、migration／recook境界が明確
Naming／Project layout	十分・用語補正要	02-foundation/naming-project-layout.md	技術識別子と配置は詳細。ただしPackage／Pack等の人間語彙衝突が残る
C++ language／Module	十分	02-foundation/cpp23-modules.md	language profile、Module境界、標準Library移行が記述される
Math／Coordinate／Unit	十分	02-foundation/math-core.md	数値、座標、単位、utility境界がある
Memory／Pointer／Handle	十分	02-foundation/memory-pointers.md	ownership、allocation domain、handle、borrow境界が強い
Toolchain／Dependencies	被覆・Closure未了	02-foundation/toolchain-dependencies.md §2、§6、§8	exact pin方針は詳細。必須Dependency、minimum OS、CI／device capacity等が未固定
Project aggregate／Revision	十分	03-authoring/project-state.md §3–§8	ChangeSet、commit、undo、recovery、conflictが明確
Asset Source／Import	十分	03-authoring/asset-lifecycle.md §1–§4	Source、Import、Profile、IR、Preview、Reimportが詳細
Cook／Derived Artifact／Catalog	十分	asset-lifecycle.md §5–§7	Derived、Cooked、Catalog、VFS、Content Package、promotionを被覆
汎用Runtime Asset取得／常駐	必須欠落	04-runtime/runtime-package.md §1、Closure Review §8／§9.1	request、priority、cancel、dependency、residency、eviction等のOwner未決定
Editor UI Core／Shell	十分	03-authoring/editor-ui-framework.md	Tree、Layout、Style、Event、Rendering、IME、Accessibilityまで詳細
Editor Workspace／UX	十分	03-authoring/editor-workspace-ux.md	Workspace、Panel、AI Partner、Undo、long-running task、recoveryを被覆
Game UI／Text	十分	07-platform/ui-text-localization-accessibility.md	UI document、layout、binding、focus、text、glyph、render packetが詳細
Localization	十分	同 §11–§12	locale、fallback、Message AST、font fallback、semantic invarianceを被覆
Accessibility	十分	同 §14、Editor UI §14／§19	semantic tree、Platform bridge、keyboard、screen reader等が明確
Local Settings／Player Profile	十分・Consent参照未解決	同 §15	Settings authorityは明確。ただしConsent Recordの委譲先がない
Gameplay programming model	被覆・Authority移行未了	03-authoring/gameplay-programming-model.md	Definition、System、Perception、Rule／FSMを被覆。ECS current authority問題あり
Reusable Feature Pack	十分	08-packs/gameplay-features.md	Featureのmanifest、Port、State、Save／Replay、failureを被覆
Pack composition／lifecycle	十分	08-packs/pack-contract.md	dependency、install、update、removal、Genre Pack境界が明確
Shooter technical reference	十分	08-packs/shooter.md	Shooter固有compositionは存在する
Scenario／Stage	十分	08-packs/scenario-stage.md	Stage、completion、scope、transition、Save／Replayを詳細化
RPG reusable features	必須欠落	product-plan.md §5.1.1、§9.4	battle、progression、inventory等が必要と明記されるがOwner未materialize
RPG Genre composition	必須欠落	同、pack-contract.md	Genre Packの一般契約はあるがRPG固有Ownerなし
RPG Reference Game authority	必須欠落	product-plan.md §5.0、§9.4	fixture要件はProductにあるが、Reference Gameの正本Ownerなし
Runtime ECS	Critical Closure未了	04-runtime/entity-component-system.md、ECS Closure Review	target設計は豊富。current authority移管とconsumer migrationが未完
Scheduling／Lifetime	十分	04-runtime/scheduling-lifetime.md	phase、cadence、job、message、lease、failureを詳細化
Runtime Entry／Package	被覆・重複Authority危険	runtime-package.md、Scheduling、Project State、Product Plan	subsystem文書はあるがProduct §5.0.1が遷移意味を重複規定
Persistence／Save	十分	04-runtime/persistence-save.md	identity、record set、digest、reconstruction、migrationを被覆
Replay	十分	Persistence、Debugging	semantic projectionとtransport／debug用途が分離される
Debugging／Observability	十分	04-runtime/debugging-observability-replay.md	event、query、causality、breakpoint、capture、crash、support bundleが詳細
Performance／Capacity	被覆・Evidence未了	04-runtime/performance-capacity.md	budget modelは広いがhardware、scale、実測、Target closureが未materialize
Animation	十分	05-simulation/animation.md	source、artifact、graph、pose、event、root motion、retargetを被覆
Collision	十分	05-simulation/collision.md	geometry、filter、query、contact／trigger semanticsを被覆
Physics	十分	05-simulation/physics.md	body、constraint、kinematic、kernel adapter、failureを被覆
Navigation	十分・一部alignment補正	05-simulation/navigation.md、Navigation Review	2D grid、3D navmesh、query、lifetimeを被覆
World／Scene／Partition	十分	06-rendering/world.md	World、Scene、Space、Cell、partition、transition、Tilemap sourceを被覆
2D Sprite runtime	Authority分散	Render Graph §6／§12、Materials、LOD	packet抽出はあるがSprite runtime representationの正本が不明
2D Tile runtime	Authority分散	World §10.1、Asset、Render Graph	source／cook／renderの接続はあるがruntime representationとsorting等が分散
3D Render Graph／RHI	十分	06-rendering/render-graph.md	resource、pass、queue、barrier、surface、recoveryが詳細
Material／Shader	十分	materials.md、project-shader.md	semantic intent、bounded source、compile、qualificationが詳細
Lighting／Post Process	十分	lighting.md、post-processing.md	authoringとexecution boundaryが明確
VFX	十分	vfx-authoring.md、vfx-runtime.md	Source／Artifact／Instance／Simulation／Renderが分離
Camera	十分	camera.md	profile、rig、director、sequenceを被覆
Environment／Surface	十分	environment-surfaces.md	sky、fog、weather presentation、水・surface responseを被覆
LOD	十分	lod.md	representation selection、transition、fallbackを被覆
Virtualized geometry	被覆・Runtime Asset依存未解決	virtualized-continuous-geometry.md §7.2	専用設計は詳細だが汎用runtime asset authority refが未解決
Input	十分	07-platform/input.md	Device、Action、Binding、Context、Remap、Haptics、Replayを被覆
Audio	十分・Runtime Asset接続未了	07-platform/audio.md	Cue、Voice、Mixer、Spatial、Streamを被覆。汎用residencyとの境界未閉鎖
Windows target	被覆・baseline未固定	07-platform/windows.md	process、window、package、signing、update、crash、securityを被覆
Mobile common	十分	07-platform/mobile-common.md	lifecycle、surface、memory、thermal、asset delivery、privacyを被覆
Android	十分・review	07-platform/android.md	package、delivery、permission、device qualificationが詳細
Apple mobile	十分・名称補正要	07-platform/apple.md	実体はiPhone／iPad mobile。macOSは別Future
Build／Cook	十分・target evidence未了	Core、Toolchain、Asset	Gateway、Candidate、Receipt、Cookは詳細
Package／Install／Launch	被覆・横断表不足	Core、Platform文書	個別契約は詳細だがTarget横断の一意なclosure mapが不足
Signing／Store upload	十分・review	Windows／Android／Apple、AI Security	secret separation、unsigned／signed／upload境界が強い
AI Security／Approval	十分	ai-security-approval.md	Risk、Authorization、Sandbox、Approval、Promotionを非常に詳細化
Product-wide Security	Authority分散	Toolchain、Asset、Native、Platform、Debug	AI外の脅威、脆弱性管理、support policyの総合Ownerがない
Consent Record	必須欠落	UI §15、Core §9.1、Debug §14／§15	複数文書が専用正本を前提とするが該当Ownerなし
Verification／Provenance	十分・名称／policy補正要	ai-verification-provenance.md	実際にはAIを超えた全製品Evidence、CI、Releaseを所有
QA policy	被覆・統合不足	Core §12、各Owner Test節、Verification §14	多様なtestはあるが、flaky／quarantine／release-block policyの総合説明不足
Networking／Multiplayer	現MVP非対象	Product §2.2、§8	明示的Future。現MVPの欠落とは判定しない
Account／Cloud／Live Service	現MVP非対象	Product §2.2、§8	明示的Future
Linux／macOS／Web／Console／XR	現MVP非対象	Product §4、§8	独立Future Target Program
Runtime AI generation	現MVP非対象	Product、Mobile	current shipping runtimeでは禁止
Unrestricted scripting／JIT	意図的非目標	Gameplay、Product、Platform	欠落ではなく明示的安全方針
Marketplace／arbitrary plugin	意図的非目標	Product、Pack、Gameplay	current Product scope外
3. 真に欠落している必須Capability
3.1 RPG-first MVPのReusable Feature／Genre／Reference authority
文書上の事実

00-product/product-plan.mdの以下の節は、MVP-Aをcompact 2D command RPGとして固定している。

§5 MVP scope

§5.0 Closed RPG Reference MVP boundary

§5.1.1 RPG-first MVPとcurrent Product Definitionの分離

§9.4 Platform／Packs

§5.0では、Title、Town、Field、Dungeon、Boss、Result／Endingのflowに加え、command battle、progression、equipment、Skill、status effect、Quest、choice、currency、inventory、Shop、Save／Load、en-US／ja-JPをMVP fixtureへ要求している。

§5.1.1は、RPG Product Definition materialization前に次のReusable Owner設計が必要であると明記する。

command battle

actor progression

inventory／equipment

dialogue／quest

currency／shop

RPG Genre composition

Reference Game fixture

source／destination Product Definition Migration

同節は、これらが存在する前にRPG Stable ID、Schema、Capability、Operation、Work Packageを仮登録することを禁止している。

§9.4ではさらに、RPG-first targetを次の四層へ分ける。

Generic Engine Core

Reusable RPG Feature Packs

RPG Genre Pack

RPG Reference Game

そしてReusable Feature Owner、RPG Genre Pack Owner、Reference Game Ownerは未materializeであると明記する。

08-packs/pack-contract.mdの§3.2 Feature Packと§3.3 Genre Packは一般的なPack構造を定義するが、RPG固有Ownerは存在しない。添付76文書のファイル名、H1、Owner一覧にもRPG Feature／Genre／Reference Ownerはない。

監査判断

これは単なる記述不足ではなく、現在のMVP定義を成立させる正本Capabilityの欠落である。

Product PlanにはRPG fixtureの結果要件が非常に詳しく書かれている一方、その結果を成立させるauthoritative state、request、validation、transaction、Save identity、failure、diagnostic、qualificationを所有する文書がない。

Product文書自身が未materializeを正しく明記しているため、虚偽の完成主張ではない。しかし、目的が「実装前の計画書・アーキテクチャ文書の充実化」である以上、この状態ではMVP Architectureは文書上閉じていない。

影響

Product MVPの最重要gameplay domainが一意なOwnerへ解決しない。

UI、Save、Replay、AI Authoring、ECS、Scenario、Inputが何を参照すべきか決められない。

Generic Gameplay FeatureとRPG固有意味の境界が未確定。

ShooterのCombat／Interaction／Pickup semanticsをRPGへ暗黙転用する危険がある。

Product fixtureの数値・flowだけが先行し、authoritative transactionやfailure semanticsがProduct文書へ流入する。

Shooter ReceiptをRPG Evidenceへ誤流用する危険が残る。

候補Owner

Governanceが次の責務分割を決定する必要がある。

再利用可能なbattle／progression／inventory等: Reusable Gameplay Feature Owner群

RPG composition、RPG profile、role mapping、game flow: RPG Genre Owner

original content、balance、World composition、localized presentation、acceptance fixture: RPG Reference Game Owner

Product outcomeとMVP Gate: 既存Product Plan

既存Gameplay Feature Packsだけを無条件に総Ownerへ拡張すると、一つの文書が複数の独立state machineを所有する可能性があるため、分割・統合はArchitecture Governance §7の基準で判断すべきである。

文書上の閉鎖条件

Product §5.0の全RPG要件が、Generic Owner、Reusable Feature Owner、Genre Owner、Reference Ownerのいずれか一つへ解決する。

battle、progression、inventory、quest、economyごとにauthoritative state、request、validation、transaction、Save／Replay、failure、diagnostic、qualificationのOwnerが一意になる。

RPG UIがPresentationに限定され、Gameplay stateを書かないことが双方の文書で一致する。

current Shooter sourceとRPG destinationの全Owner／Capability／Evidence mappingが明示される。

Product PlanからSubsystem固有の重複契約を除いてもMVP acceptanceが追跡可能になる。

missing Ownerが未解決表から除外されるまでRPG Capabilityをnot_activatedとして維持する。

3.2 汎用Runtime Asset request／residency authority
文書上の事実

04-runtime/runtime-package.mdのHeaderと§1 状態と結論は、同文書がTexture、Mesh、Audio、Font等の汎用Runtime Asset request／residency Managerではないと明記する。

同文書は、汎用Runtime Asset authorityをappendices/architecture-plan-closure-review.mdのARCH-C02へ委譲する。

appendices/architecture-plan-closure-review.mdでは以下によりOwner gapが明記される。

§1 監査結論

§4 Runtime計画のcoverage

§8 Architecture closure register

§9.1 Runtime Asset authority

ARCH-C02はopen-decisionであり、最低限必要な責務として次を挙げる。

request identity

Target／Catalog／Package generation

priority／deadline

cancel／idempotency

dependency closure

range I/O

decode／upload／activation

generation／lease

resident／streamed／mip／LOD

eviction／fallback／retire

device loss／memory pressure

metering／diagnostic／qualification

06-rendering/virtualized-continuous-geometry.mdの§7.2 Residency Coordinatorと未解決authorityにも、汎用runtime asset authorityへのbindingが未解決であることが現れる。

Asset LifecycleはSource、Import、Cook、Catalog、Content Packageを所有し、Runtime PackageはRuntime Entry／World Root／Section stagingとpublicationを所有するが、その間の汎用Runtime Asset lifecycleを所有しない。

監査判断

これは既存Reviewでも認識済みのgapであり、第1次監査でもCriticalな必須欠落として再確認された。

Rendering、Audio、World streaming、LOD、Virtualized Geometry、UI glyph／image、VFXはすべてRuntime Asset consumerとなるが、consumerごとの局所Managerを合成して汎用authorityを作ることはできない。

公式資料との比較でも、参照したEngineはいずれもSource／Cookとは別にRuntimeでの取得、非同期状態、依存、memory保持、release等を明示的責務としている。これは特定EngineのAPIを採用する根拠ではなく、このDomain自体を省略できないことを確認する材料である。
Godot Engine documentation
+3
Epic Games Developers
+3
Unity マニュアル
+3

影響

同じAssetに対してRendering、Audio、Worldが別々のrequest identityやcancel semanticsを持つ危険。

Package publicationとRuntime residencyが混同される。

memory pressure、device loss、generation切替、hot reload時のlast-valid保証が閉じない。

priority／deadline／prefetchがWorld、Camera、LOD、Audioごとに競合する。

Virtualized Geometryのroot pin、page eviction、fallbackを汎用memory authorityへ接続できない。

Asset dependency failureがpartial activationを生む可能性を文書上排除できない。

Performance Ownerが測定するqueue／budgetと、実際に制御するAuthorityが分離したままになる。

候補Owner

Governance上の候補は二つある。

独立した汎用Runtime Resource／Asset Owner

Runtime Packageの責務を汎用Runtime Assetまで正式拡張

ただし、後者は現在のRuntime Package §1が明示的に否定しているため、採用するならscope、文書名、依存、分割基準を同時に再判定する必要がある。

Asset LifecycleはSource／Cook正本として維持し、Scheduling、Memory、Performance、Rendering、Audio、Worldはconsumerまたはpolicy inputに限定すべきである。

文書上の閉鎖条件

汎用Runtime Asset lifecycleの一意Ownerが決定される。

Source／Cook／Catalog／Package generationからRuntime requestまでのidentity closureが定義される。

request、dependency、cancel、activation、lease、release、eviction、retire、device loss、memory pressureの状態遷移が一つの正本へ置かれる。

各consumerが独自Managerや独自priority意味を所有しないことが明記される。

Runtime Package publication、World activation、Render snapshot、Audio voice、VFX instanceとの順序境界が接続される。

failure、stale generation、partial dependency、cancel race、over-budget時のlast-valid挙動が閉じる。

Virtualized Geometry §7.2の未解決refが一意なOwnerへ解決する。

ARCH-C02がopen registerから除外できるだけのDecisionとcross-owner consistencyが成立する。

3.3 汎用Consent Record authority
文書上の事実

07-platform/ui-text-localization-accessibility.mdの§15 Player Profile／Settingsは、telemetry、AI Provider、network consentを「専用Consent Record正本」が所有すると明記し、LocalPlayerProfileV1にconsent_record_refs[]を持たせる。

しかし、添付コーパス内にConsent Recordを正本範囲とするOwner文書は存在しない。

一方で、複数文書がConsent Recordを既存authorityとして参照している。

02-foundation/core-architecture.md §9.1 OperationTaskV1

consent_record_ref?

install／reset Receiptのconsent binding

enqueue時／実行直前のconsent再検証

04-runtime/debugging-observability-replay.md

§14 Reproduction、Crash、Hang、remote device

§15 Security、privacy、retention、failure

support bundleのconsent、redaction、retention

07-platform/windows.md

§11 Crash、Hang、Diagnostics

§12 Security

07-platform/mobile-common.md

§7 Memory、thermal、privacy、Runtime AI

ProjectPrivacySpec

07-platform/apple.md

§3 Asset packaging、privacy、runtime content

telemetry、crash、AI prompt、generated contentを別purposeとして扱う

01-governance/ai-security-approval.md

install／resetでconsent必須、別Operationへの権限継承禁止

Mobile CommonはProjectPrivacySpecを定義するが、これはProjectのdata category、purpose、retention、sharing、deletion等の宣言であり、個々の利用者またはOperationに対するgrant／revoke Evidenceの汎用authorityとは一致しない。

DebuggingはSupport Bundle固有のconsent semanticsを所有するが、install、reset、telemetry、AI Provider、network、crash等を横断する汎用Consent authorityではない。

監査判断

Privacyに関する記述全体が欠落しているわけではない。Platform privacy declaration、redaction、support bundle privacyは各文書に存在する。

真に欠落しているのは、複数文書が存在を前提とする汎用Consent Recordの正本Authorityである。

現在は参照型名と利用場面だけが先行し、次の意味が一意に定まらない。

consent subject

purpose

scope

Target／Device／Project binding

grant／deny／revoke／expiry

actor

presentation locale／text version

policy version

evidence

freshness

withdrawal後の処理

parent／child account等の将来境界

crash、telemetry、AI、network、install、resetの相互非継承

影響

UI設定のboolと法的・Security上のconsentが混同される危険。

一つのpurposeへの同意を別purposeへ流用する危険。

install／reset／support bundleのconsent freshnessを同じ基準で検証できない。

revoke後のqueued task、retained data、upload、support bundleがどう扱われるか閉じない。

Windows、Android、Appleで異なるconsent semanticsを持つ危険。

OperationTaskV1が参照する対象のidentity／revision／authorityが解決しない。

Privacy declarationと実際のconsent Evidenceを一致させるOwnerがない。

候補Owner

独立したCross-product Consent／Privacy Owner

または既存のGovernance／Security Ownerを、AI authorizationとは別のpurpose-based Consent authorityまで正式拡張

UI、Mobile Common、Debugging、各Platformは利用場面固有のconsumerまたはpresentation／declaration Ownerに留めるべきであり、汎用Consent正本を分散所有すべきではない。

文書上の閉鎖条件

Consent Recordのsubject、purpose、scope、version、grant、deny、revoke、expiry、evidence、freshness、retention bindingの一意Ownerが決まる。

ProjectPrivacySpec、UI profile、OperationTask、Support Bundle、Crash、AI Provider、Networkの関係が定義される。

purpose間の非継承が明示される。

revoke、policy変更、locale／text version変更、Device交換、Project revision変更時の扱いが閉じる。

consentなし／stale／別purpose／別subject／revokedのnegative conditionが各consumerで同じ意味へ解決する。

Platform固有Manifest／Store declarationとConsent Recordを同一物にせず、照合関係を定義する。

「専用Consent Record正本」という参照が実在するOwnerへ解決する。

4. 存在するが記述が不十分なCapability・契約・Authority
4.1 Runtime ECSのcurrent authority移管
根拠

README.md §1 目的と非規範性

03-authoring/gameplay-programming-model.md §3 GameSystemSpecV2

04-runtime/entity-component-system.md §1 状態と結論、§1.1 Target contract closure

appendices/runtime-ecs-design-closure-review.md §1、§3、§4、§6

appendices/governance-migration-proposals.mdのRuntime ECS canonicalization候補

文書上の事実

READMEは、Runtime ECS文書をtarget Owner候補とし、Migration ChangeSetが承認・適用されるまではGameplay Programming Model revision 1のcurrent authorityを置換しないと明記する。

ECS Closure Reviewには、Owner migration、unresolved refs、canonical layout、query semantics、cache／invalidation、runtime spawn、persistent identity、fixture、Target／device qualification、AI context等の未閉鎖項目が列挙される。

監査判断

ECS文書自体は不足しているのではなく、非常に詳細である。問題は、current authorityとtarget ownerの二重状態を閉じる移管契約が未完なことである。

多くのconsumerがtarget Runtime ECSの型や境界を参照し始める一方、Governance上のcurrent authorityはGameplay Programming Modelに残る。この状態は明示されているため隠れた矛盾ではないが、Architecture baselineとしてはCritical blockerである。

影響

Component identity、query、structural transactionの正本解釈がconsumerごとに変わる。

Save、Runtime Package、Scheduling、Debug、Native Module、AI Projectionが異なるrevisionを参照する危険。

target文書の充実をcurrent化と誤認しやすい。

RPG／Shooter双方のEvidenceがどのECS authorityへ束縛されるか曖昧。

候補Owner

Target semantics: 04-runtime/entity-component-system.md

移管前current authority: 03-authoring/gameplay-programming-model.md

migrationとauthority切替: Architecture Governance／Compatibility

consumer整合: 各Subsystem Owner

文書上の閉鎖条件

current authorityが常に一つである。

target Ownerへの移管条件とall-or-nothing boundaryが明示される。

Closure ReviewのECS-C項目が各Ownerへ解決する。

全consumerの規範参照が同じrevisionへ揃う。

migration前後の型、identity、Save／Replay、diagnostic、AI projectionの扱いが閉じる。

ECS current化だけでRPG、AI Authoring、全Target、shipping readinessを誤表示しない条件が維持される。

4.2 Runtime Entry transition authorityの重複
根拠

00-product/product-plan.md §5.0.1 Title-to-Result scene／screen transition definition closure

03-authoring/project-state.md §3.1.1 RuntimeEntryPointV1、§3.1.2 Runtime Entryのclosed Operation Catalog

04-runtime/scheduling-lifetime.md §4.0 Runtime Entry transition

04-runtime/runtime-package.md §1.1 Runtime Entry package closure

04-runtime/persistence-save.md §4.1 Continue／Runtime Session load

07-platform/ui-text-localization-accessibility.md §8 ViewModelとBinding、§15 Player Profile／Settings

08-packs/scenario-stage.md §11 Stage transition contract

文書上の事実

Product §5.0.1は、Title、Settings、New Game、Continue、HUD／Pause、Town→Field→Dungeon→Boss、Result、Titleへの遷移をOwnerへマッピングする。

同時に、次の詳細もProduct本文で規定する。

destination branchの非公開staging

eligible T00_BoundaryApply

publish後のreverse teardown

commit前cancel

commit後cancel

stale generation

hash不一致

partial World／UI／Stageを表示しない

last-valid generation維持

Sessionとbranch generationの関係

Product本文は別箇所で、Subsystemの型、Field、API、default、budgetを非正本とする。

監査判断

ProductがMVP outcomeとcross-owner scenarioを定義することは妥当である。しかし、Product §5.0.1は単なるacceptance scenarioを超え、Scheduling／Runtime Packageが所有すべきstate transitionとfailure semanticsをかなり具体的に規定している。

そのため、次のauthority tensionが生じる。

ProductがMVP上の「何が起きるべきか」を所有する範囲

Project StateがRuntime Entry定義とOperation Catalogを所有する範囲

Schedulingがtransition commit boundaryを所有する範囲

Runtime Packageがstaging／publicationを所有する範囲

PersistenceがContinue reconstructionを所有する範囲

影響

Subsystem側の契約変更時にProduct側の複写がstaleになる。

cancel／commit／publicationの正本が複数に見える。

Product文書が実質的なRuntime state machine Ownerになる。

一つの変更で多数文書の同義記述を同期する必要が生じる。

Product acceptanceとSubsystem implementation contractの違いをAIが誤読する。

候補Owner

Product outcome／end-to-end scenario: Product Plan

Runtime Entry source definition／selection: Project State

transition order／commit／cancel: Scheduling

package staging／publication: Runtime Package

Continue reconstruction: Persistence

UI Screen navigation: UI

Stage progression: Scenario／Stage

文書上の閉鎖条件

Product Planはobservable outcome、Owner mapping、acceptance invariantへ限定される。

transition state machine、cancel、publication、generation、rollbackは一意Subsystem Ownerへ解決する。

ProductからSubsystemの詳細を削除してもscenario traceが失われない参照構造になる。

各Ownerが同じRuntime Entry identity、Session、branch generation、failure結果を参照する。

重複するmust記述のうち正本と説明引用が区別される。

4.3 Architecture Inventory／Explain Projection／AI Task Context
根拠

README.md §1 目的と非規範性

01-governance/architecture-governance.md §5 InventoryとIndex、§5.3 Architecture Explain Projection

appendices/architecture-plan-closure-review.md §1、§3.1、§8

appendices/executable-contracts-operation-planning-catalog.md

appendices/ai-evidence-envelope-fixture-catalog.md

文書上の事実

READMEは、Architecture InventoryのGenerator、Schema、生成Artifactが存在しないと明記する。

Governanceは将来のInventory、Explain Projectionを定義するが、現在は手動Indexである。

Closure Reviewは、AIによる概念説明は可能だが、Inventory、Explain Projection、Capsule、Schema、query Toolが未materializeであるとする。

監査判断

76文書、51 Owner、複数のcurrent／target、Proposal、Decision、Capability、Operationを持つArchitectureでは、手動Indexだけでは次を安定して解決できない。

一意Owner

current authority

target owner

proposal-only

unresolved ref

Capability state

Gate

Evidence

consumer一覧

external claim freshness

Decision disposition

AI-firstをProduct差別化とするなら、AIが文書全文を毎回推測的に読んでOwnerを決める状態は、Productの安全境界と整合しない。

影響

stale文書を正本と誤認する。

target型をcurrent型として使用する。

Appendix ProposalをOwner正本として扱う。

unresolved decisionをclosedと誤認する。

Owner uniquenessの違反を機械検出できない。

規範依存、リンク、Capability stateのdriftを継続的に検出できない。

候補Owner

Inventory／Explain semantics: Architecture Governance

executable projection／query contract: Executable Contracts

AI Task Contextのauthorization／redaction: AI Security

provenance／freshness: AI Verification

文書上の閉鎖条件

Inventoryが何を正本入力とし、何を生成結果とするか明確になる。

Owner、document state、implementation state、current／target、dependency、fragment、Capability、Gate、Evidence、Decisionを区別する。

stale／duplicate／unresolved／cycle／layer violationの扱いが定義される。

Explain Projectionが自由文要約ではなく、正本refと根拠状態を保持する。

AI Task Contextが必要最小限のbounded projectionとして定義される。

手動Indexと将来Inventoryのauthority関係が明確になる。

4.4 2D Sprite／Tile runtime authority
根拠

00-product/product-plan.md §4 2D／3D Capability portfolio

03-authoring/asset-lifecycle.md §2 Import Profileと検出

06-rendering/render-graph.md §6 2D／3D／UI composition、§12 関連契約の配置

06-rendering/world.md §10.1 Tilemap source、cook、publication

06-rendering/materials.md

06-rendering/lod.md

05-simulation/collision.md

05-simulation/navigation.md

文書上の事実

Render Graphは2D／3D／UIを別layerとして扱い、Renderer2DExecutionPlanV1がSprite／Tile packetを抽出する記述を持つ。

WorldはTilemap source、cook、publicationを扱う。

Asset LifecycleはSprite importを扱い、Materialsには2D domain、LODにはSprite LODが存在する。

しかし、SpriteRendererComponentV1に相当するruntime representation、Tile chunk runtime representation、frame selection、sorting authority、batch／culling、mask／blend、atlas residency、debug projectionを一括して正本範囲とするOwnerは確認できない。

監査判断

2D capabilityは欠落していない。複数文書に十分な要素が存在する。

不足しているのは、2D runtimeを単なるRender Graphのpacket抽出に限定せず、Asset／World／Runtime／Rendering間で一意に接続するauthorityである。

参照したUnity資料でも、2D sortingにはSorting Layer、Order、Queue、Camera distance等の独立要素があり、Tilemapではlighting、sorting、batchingが接続されている。これをコピーする必要はないが、MiraikanaiでもsortingやbatchingのOwnerを暗黙にできないことを示す比較材料となる。
Unity マニュアル
+1

影響

Sprite sort keyの生成元がProject、World、Component、Render Graphのどこか不明。

Tilemap source identityとruntime chunk identityの関係が不明。

Sprite animation／frame selectionとAnimation Ownerの境界が曖昧。

atlas／texture residencyが汎用Runtime Asset gapへ接続しない。

2D lighting、mask、material、pixel snapping等のqualification matrixが分散する。

2Dを3D packetの特殊caseとして実質処理する危険が残る。

候補Owner

Governanceが以下のいずれかを選ぶ必要がある。

独立2D Rendering／Runtime Owner

Render Graphのscopeを2D runtime representationまで正式拡張

WorldとRender Graphの間に2D scene extraction Ownerを置く

WorldはTile source／spatial composition、Assetはimport／cook、Render Graphはexecutionに限定するのが現在の境界と最も整合する。

文書上の閉鎖条件

Sprite／Tile runtime identity、state、lifetime、sorting、visibility、batch、material binding、animation bindingが一意Ownerへ解決する。

Source TilemapからCooked artifact、Runtime chunk、Render packetまでのref closureが成立する。

2Dと3Dの共通部分と固有部分が明記される。

atlas residency、hot reload、device lossを汎用Runtime Asset authorityへ接続する。

2D-specific diagnostic／qualificationが一意Ownerへ解決する。

4.5 Gameplay AIのPerceptionからActionまでの横断契約
根拠

03-authoring/gameplay-programming-model.md §2.4 C1 Perception／Interaction、§2.5 Rule／ECAとFinite State Machine

05-simulation/navigation.md

07-platform/input.md §4.5 Semantic Action-to-Command Binding

05-simulation/animation.md

04-runtime/debugging-observability-replay.md §11 Domain Debug Projection

08-packs/shooter.md

文書上の事実

Gameplay Programming Modelはbounded Perception／Interaction、Rule／ECA、FSMを所有する。

ShooterはGameplay Perception Capabilityを参照する。

Navigation、Input、Animation、Debugには個別のOwner文書がある。

しかし、以下のend-to-end contractがまとまっていない。

observation／stimulus

memory／aging

context

decision／goal selection

action request

navigation／motion request

animation／VFX presentation

interruption／abort

failure／fallback

debug／replay explanation

監査判断

Behavior Treeを導入する必要はない。FSM、Utility、Rule、Planner等の方式選択も本監査の対象外である。

不足しているのは方式ではなく、PerceptionとActionの間のauthoritative data flow、state ownership、interrupt、failure、debugの契約である。

公式資料ではPerceptionがstimulusやmemory agingを持ち、Behavior executionが条件、action、abort、history／debugへ接続されている。Miraikanaiでも同じ方式は不要だが、これらの責務がどこにあるかは閉じる必要がある。
Epic Games Developers
+1

影響

Perception resultをGameplay stateとして保存するのかephemeralにするのか不明。

decisionがNavigation requestを直接操作する可能性。

action interruptionとFSM transitionが別Ownerで競合する。

AI挙動をReplay／Debugで説明できない。

Shooter固有logicがGeneric Gameplay Modelへ流入する。

将来のRPG NPC behaviorでも同じgapが再発する。

候補Owner

Gameplay Programming Modelを主要Owner候補とし、Navigation、Input／Command、Animation、Debugをconsumer／execution Ownerとするのが現状の配置に最も近い。ただし、Gameplay ModelがECS current authorityまで抱えているため、ECS移管と同時に責務過多を再評価する必要がある。

文書上の閉鎖条件

perception、memory、decision、action、interrupt、completion、failureのauthorityが一意になる。

Navigation／Animation／Inputへはtyped requestで接続し、private stateを書かない。

deterministic／non-deterministic要素、Save／Replay対象、Debug projectionを区別する。

Generic contractとGenre-specific decision semanticsを分離する。

AI Authoringが説明・提案できるbounded projectionを定義する。

4.6 Toolchain、Target baseline、Dependency、CI／Device capacity
根拠

02-foundation/toolchain-dependencies.md

§2 Target toolchain baseline

§6 External Dependency baseline

§6.1 必須だが未固定のDependency

§6.2 Known unresolved decision register

§8 Toolchain lock contract

§8.1 CI execution profile

appendices/architecture-plan-closure-review.md §6 Target別Buildと最適化

各Platform文書のQualification節

文書上の事実

Toolchain文書はpin、hash、license、lock、SBOM、CI profile、device matrixのtarget契約を非常に詳細に記述する。

一方で、次が未固定と明記される。

Windows minimum OS

必須C++ Dependencyの一部

CI runner／GPU／macOS build host／mobile device pool

capacity Owner

Target別compiler／linker optimization closure

実際のlock／Receipt

監査判断

これは文書が薄いのではなく、未決定値とtarget設計の境界が残っている状態である。未計測値を推測しない方針は正しい。

問題は、ProductやPlatform文書に多数のexact／minimum／qualified条件がありながら、未固定baselineとの依存関係を一目で追跡できるcurrent target readiness projectionがないことである。

影響

Product targetが開始可能か文書横断で判断しにくい。

exact pin記述をlock済みと誤認する。

minimum OSとSDK／compiler／package requirementが食い違う可能性。

device qualificationをSimulatorや別classへ代替する危険。

provisional performance値をGateへ使う危険。

候補Owner

Tool／SDK／Dependency／lock: Toolchain Owner

Product target readiness: Product Plan

Device／OS／Package条件: 各Platform Owner

Evidence grade／freshness: Verification Owner

budget／measurement: Performance Owner

文書上の閉鎖条件

未固定項目がTargetごとに一意のreadiness statusへ集約される。

Toolchain candidate、source-checked、locked、qualifiedを区別する。

minimum OS、SDK、compiler、package、device matrixのcross-checkが定義される。

CI capacity不在時にどのGateが開始不能か追跡可能になる。

provisional optimizationとProduct claimが分離される。

4.7 QA／Qualification policyの統合
根拠

02-foundation/core-architecture.md §12 Test、CI、AI生成物

01-governance/ai-verification-provenance.md §2、§3、§6、§14 CI lanes

各Owner文書のTest／Qualification節

Platform文書の実機fixture

04-runtime/performance-capacity.md

文書上の事実

コーパスには以下が存在する。

unit

boundary conformance

property／fuzz

artifact golden

migration

concurrency stress

cancel recovery

performance

soak

screenshot／visual

accessibility

clean package

physical device

lifecycle

thermal

security negative

release evidence

したがってQA Capability自体は欠落していない。

一方、これらを製品全体で統合する以下のpolicyは十分明確でない。

test category taxonomy

test ownership

flaky判定

retryの許容範囲

quarantine

expected failure

infrastructure failure

product failure

partial lane success

waiver／exception

historical passのfreshness

release-blocking severity

duplicate evidence

device／session evidenceとunit evidenceの代替禁止

参照したEpic公式資料でも、unit／feature／content stress／screenshotと、実際のProject session／Platform executionが別層として扱われる。Miraikanaiも同じframeworkを必要としないが、Evidence classを区別する必要性は確認できる。
Epic Games Developers
+1

影響

一部laneのpassを全体qualificationへ誤昇格する。

flaky testのretry成功だけでfailureを隠す。

infrastructure unavailableとProduct passを混同する。

simulator、headless、physical device、shipping packageのEvidenceを相互代用する。

Screenshot一致をsemantic correctnessへ読み替える。

過去Receiptをcurrent Candidateへ流用する。

候補Owner

AI Verification／Provenanceは名称に反して既に一般Verification、CI、Release evidenceを所有しているため、第一候補は同Ownerである。ただし、AI専用に見える名称と実際のscopeを先に整理する必要がある。

文書上の閉鎖条件

全Test／Evidence classと代替禁止関係が一意に定義される。

retry、flaky、quarantine、waiver、infrastructure failureの扱いが閉じる。

lane passからProduct／Capability／Target Gateへのaggregation規則が明確になる。

clean package／install／launch／device sessionがunit testで代替できないことが明示される。

Evidence freshness、revocation、Candidate equalityが全laneで一致する。

4.8 Product-wide Security／Vulnerability governance
根拠

01-governance/ai-security-approval.md

02-foundation/toolchain-dependencies.md §7–§9

03-authoring/asset-lifecycle.md §11

03-authoring/native-game-module.md

06-rendering/project-shader.md

04-runtime/debugging-observability-replay.md §15

各Platform Security節

文書上の事実

AI route、Source Worker、credential separation、Signing、Asset validation、Shader、Native Module、Platform package、crash privacyには強いSecurity記述がある。

しかし、全製品横断で次を一意所有する文書は確認できない。

Engine全体のtrust zone一覧

untrusted content／project／package／save／network inputの統合脅威分類

vulnerability intake

severity

triage

patch／backport policy

dependency vulnerability対応

revocation

disclosure

support window

secure default baseline

security regression classification

incidentからArchitecture requirementへのfeedback

AI Securityをそのまま全製品Security Ownerとして読むことは、Headerのscopeと一致しない。

NIST SSDFもSecurity PracticeをSDLC全体へ統合する必要性を示しており、AI経路が詳細であることだけでは製品全体の脆弱性管理が閉じたことにならない。
NISTコンピュータセキュリティリソースセンター

影響

非AI attack surfaceのOwnerが分散する。

Save、Project file、Asset、Package、Plugin相当Pack、remote contentの脅威が別々に評価される。

Dependency脆弱性の更新判断とProduct support判断が接続しない。

Severityやrelease block基準がSubsystemごとに異なる。

Security incident後にどのEvidenceやCapabilityをrevokeするか不明。

候補Owner

独立Product Security Owner

またはAI Security Ownerを、AI-specific sectionとProduct-wide security governanceへ明確に分割・拡張

Toolchain、Asset、Native、Shader、Platformはdomain-specific enforcement Ownerとして維持する。

文書上の閉鎖条件

Product-wide trust boundaryとsecurity responsibility mapが定義される。

vulnerability lifecycle、severity、release impact、revocation、support policyが閉じる。

AI Securityと一般Securityのscopeが明示される。

domain-specific Security節が共通classificationへ参照する。

privacy、consent、credential、content safety、code executionを混同しない。

negative fixtureとincident Evidenceの集約先が一意になる。

4.9 Inline根拠規則への適合
根拠

01-governance/architecture-governance.md §3.1 Inline根拠

Owner文書51件全体

文書上の事実

Governance §3.1は、以下の断定について段落、表、節の先頭にInline根拠を付けるよう要求する。

外部API／support／default／version

固定数値

must／only／deterministic／compatible

Security／Accessibility／Store／Distributionの合否

全Owner文書でliteral 根拠:を集計すると次の結果となる。

Owner文書: 51

根拠:が1件以上ある文書: 9

0件の文書: 42

全Owner文書のliteral 根拠:合計: 16

根拠表記がある9文書は以下である。

01-governance/architecture-governance.md

02-foundation/cpp23-modules.md

02-foundation/toolchain-dependencies.md

04-runtime/performance-capacity.md

05-simulation/collision.md

06-rendering/render-graph.md

07-platform/android.md

07-platform/apple.md

07-platform/ui-text-localization-accessibility.md

監査判断

literal tagがないだけで、各文書のすべての段落が違反しているとは断定できない。Headerの根拠区分、末尾の一次資料、本文リンクで根拠を示している文書もある。

しかし、42文書には固定数値、must、only、deterministic、security、qualification等が大量に存在するため、Headerの包括的な根拠区分だけで§3.1への適合を証明することはできない。

これは個別文書の軽微なeditorial issueではなく、Governance自身が定めたEvidence modelとコーパス実態のsystemic gapである。

影響

project decision、external fact、provisional number、design choiceを局所的に区別できない。

数値や外部Versionの更新時に影響箇所を検索できない。

AIが設計判断を外部事実として説明する。

provisional値をGateへ使用する。

mustのOwnerと根拠が不明になる。

候補Owner

Architecture Governance。各Ownerは自文書内の局所根拠適合を担う。

文書上の閉鎖条件

§3.1の対象となる断定の識別規則が明確になる。

Header-level根拠とInline根拠の役割差が明記される。

固定数値、外部仕様、Security／Accessibility Gate、project decisionが局所的に分類される。

全Owner文書を同じ規則で検査できる。

根拠が不要な記述へ過剰tagを付けることなく、必要箇所のcoverageをEvidenceとして示せる。

4.10 Product Planとexecution planningの境界
根拠

00-product/product-plan.md

§5.1 開発体制・見積り・risk contract

§6 単一Phase sequence

§11 Product execution registries

appendices/product-execution-registry-proposal.md

01-governance/architecture-governance.md

文書上の事実

Product Planには以下が含まれる。

phase sequence

relative size

critical path

scope reduction order

review cadence

Work Package

execution registry参照

文書は日程、担当者名、実績のない人数を推測しないと明記している。また、実装順序を決めないと書かれた箇所もある。

監査判断

無根拠な日程を禁止している点は適切である。しかし、Architecture Owner文書にcritical path、Phase、Work Package lifecycleが置かれることで、以下が混ざっている。

Product outcome

Architecture dependency

Capability activation

execution planning

delivery tracking

特にcritical_pathがArchitecture dependencyのように読める一方、実際にはplanning contractであるため、規範依存DAGと混同しやすい。

影響

Architecture上独立なSubsystemをPhase順序へ固定して解釈する。

execution registryの変更がArchitecture判断に見える。

Product directionと現在の作業状態が混ざる。

Architecture文書の長期安定性がexecution revisionに引きずられる。

AIがPhase／critical pathを実装順として出力する危険がある。

候補Owner

Product intent、MVP outcome、Capability portfolio、Product Gate: Product Plan

execution registry、relative planning state: Proposal／専用planning authority

dependency semantics: Architecture Governanceおよび各Owner

文書上の閉鎖条件

Architecture dependency、Product milestone、planning dependencyが明確に区別される。

Product Planを読んでもcritical pathをSubsystemの実装順と誤認しない。

execution registryが非正本Proposalなのかcurrent planning正本なのか明示される。

Product outcomeからexecution記録を分離してもArchitecture traceが維持される。

Phase変更がSubsystem authorityを変更しないことが明確になる。

4.11 状態語彙とAuthority語彙
根拠

全コーパス、特に以下。

architecture-governance.md §2

product-plan.md §3、§7

ai-security-approval.md §4、§9、§10

ai-verification-provenance.md

asset-lifecycle.md §7

native-game-module.md

gameplay-programming-model.md §9

toolchain-dependencies.md §8.1

各Platform release gate

文書上の事実

コーパスには以下の状態軸が存在する。

document: review等

implementation: absent等

verification: design-reviewed等

architecture: current／target／proposal-only／planning-only

capability: C0–C3

activation: not activated等

CI capacity: unfixed／qualified／unavailable

promotion

technical qualification

release activation

product completion

source promotion

artifact promotion

package promotion

個別文書では多くが丁寧に説明されている。

監査判断

問題は状態数の多さではなく、同じ一般語が異なるsubjectに対して使われることと、全状態軸の関係を一度に説明する正本がないことである。

例としてPromotionは、Asset、Native Source、Gameplay、Product Capability、Release Candidateで異なる意味を持つ。ActivationもCapability、AI route、Product releaseで異なる。

影響

Artifact promotionをCapability activationと誤認する。

technical qualificationをProduct completionと誤認する。

document reviewをdesign approvalと誤認する。

current source baselineをactive implementationと誤認する。

state transitionのOwnerが不明になる。

候補Owner

状態分類とsubject naming: Architecture Governance

人間語彙: Naming／Project Layout

各transition semantics: Domain Owner

文書上の閉鎖条件

各状態軸のsubject、Owner、transition、Evidence、相互非包含関係が一つのtaxonomyへ整理される。

unqualifiedなpromotion、activation、qualifiedの使用を避ける規則がある。

Explain Projectionが複数状態軸を混ぜずに提示できる。

一つの状態の成功から別状態を推測できないことが明記される。

4.12 Build／Package／Install／Launch／Signing／ReleaseのTarget横断closure
根拠

02-foundation/core-architecture.md §9 Build architecture、§9.2 Build Candidate／Testのtyped execution closure

02-foundation/toolchain-dependencies.md

03-authoring/asset-lifecycle.md

07-platform/windows.md

07-platform/android.md

07-platform/apple.md

01-governance/ai-security-approval.md

01-governance/ai-verification-provenance.md

文書上の事実

個別契約は非常に詳細である。

Build Candidate

Candidate Test

Package Receipt

Install

Launch

Smoke

Signing

Upload

Device identity

Approval

consent

hash equality

Project revision

Candidate root

Target Profile

一方、Windows、Android、Appleで必要な前段／後段Evidence集合、共通部分、Platform固有部分を単一のcross-target mapで比較する記述は不足する。

監査判断

Capabilityは存在するが、読者は複数の長文書を横断しなければ、一つのTarget Candidateがどのclosureを満たす必要があるか確認できない。

これは実装計画不足ではなく、Architecture上のEvidence compositionとOwner map不足である。

影響

Platform固有Gateの抜け。

common ReceiptとPlatform Receiptの重複または不足。

Signing成功をrelease成功へ誤昇格。

package inspectionとdevice sessionのどちらかを省略。

Windowsで成立するchainをMobileへそのまま適用。

Support／Reset／Uninstall／UpgradeのProduct completionへの接続漏れ。

候補Owner

共通Build／Operation chain: Core Architecture

Evidence aggregation: Verification

authorization／signing separation: AI Security

Platform固有package／device／store: 各Platform Owner

Product completion: Product Plan

文書上の閉鎖条件

Targetごとの必須Evidence集合、共通集合、禁止代替が一意に追跡できる。

unsigned build、signing、upload、install、launch、smoke、releaseのOwnerが分離される。

Product completionがどのTarget Gateを要求するか明示される。

failure／revocation／stale／device replacement時の再評価範囲が閉じる。

一つのReceiptを別Targetまたは別stageへ流用できないことが横断的に確認できる。

5. ファイル間の矛盾、重複Owner、用語ずれ、循環・欠落依存
5.1 規範依存の層順序矛盾

01-governance/architecture-governance.md §4 規範依存と関連文書は、読む／規範の順序を概ね次のように定め、逆方向依存を禁止する。

Product → Governance → Foundation → Authoring／Runtime → Simulation／Rendering → Platform → Packs

51 Owner文書の規範依存を抽出した結果、cycleはなかったが、定義された層順序に反するedgeが3件ある。

依存元	依存先	問題
00-product/product-plan.md	01-governance/architecture-governance.md	Productが後段Governanceへ規範依存
01-governance/ai-security-approval.md	02-foundation/executable-contracts.md	Governanceが後段Foundationへ規範依存
01-governance/ai-verification-provenance.md	02-foundation/executable-contracts.md	同上
監査判断

内容上は、Product文書がGovernanceの状態規則を使い、Security／VerificationがExecutable Contractsを使うことに合理性がある。

したがって、問題は必ずしも設計依存が誤っていることではなく、Governance §4の層順序規則が実際のmeta-contract dependencyを表現できていないことである。

次のいずれかを文書上決定しなければならない。

規範依存の矢印意味を変更する

Product／Governanceにmeta-layer例外を定義する

該当linkを関連文書または別dependency typeへ再分類する

層順序自体を修正する

例外を黙認したままでは、将来の本当のreverse dependencyを検出できなくなる。

5.2 Product Planの非正本宣言とRuntime Entry詳細のauthority tension

product-plan.md §5.0.1は、Subsystem型・Field・API等を非正本としながら、transition staging、boundary apply、reverse teardown、cancel semantics、generation、last-validの詳細を規定する。

これは文字どおりの論理矛盾ではないが、「Product outcome」と「Runtime state machine」の境界が不明確であり、Scheduling／Runtime Packageとの重複Ownerとして機能する危険がある。

5.3 current Shooter sourceとRPG destination

以下は矛盾ではなく、意図的なtransitional dualityである。

current Registry／Installed composition: Shooter source baseline

Product target direction: RPG-first

未materialize RPG owner: not activated

Shooter ReceiptをRPG Evidenceへ流用しない

根拠:

product-plan.md §5.1.1

pack-contract.md §1

appendices/product-execution-registry-proposal.md

appendices/runtime-ecs-design-closure-review.md ECS-C20

ただし、source／destinationの全row projectionが未materializeであるため、運用上の曖昧さは残る。これは「文書内矛盾」ではなく「明示された移行未閉鎖」と分類する。

5.4 Gameplay ECS current authorityとRuntime ECS target Owner

これも意図的なdual stateであり、隠れた矛盾ではない。

current authority: Gameplay Programming Model revision 1

target Owner: Runtime ECS

migration proposal: 未applied

implementation: absent

ただし、consumer文書がどちらを前提にしているかを機械的に解決できないため、Critical closure gapである。

5.5 重複Ownerまたは重複規範化の危険
Runtime Entry

Product、Project State、Scheduling、Runtime Package、Persistence、UI、Scenario／Stageが同じend-to-end transitionへ関与する。分担自体は必要だが、ProductがSubsystem transition semanticsまで複写している。

Promotion

以下が同じ一般語を使用する。

Product Capability promotion

Asset promotion

Native Source promotion

Gameplay promotion

Package promotion

Release promotion

それぞれsubjectは異なるが、Naming／Governanceでscoped termとして統合されていない。

Activation

以下が区別される必要がある。

Capability activation

AI route activation

Product release activation

Platform target activation

Source activation

Runtime content activation

Qualification

technical qualification

Target qualification

device qualification

feature qualification

Product completion

release gate

個別文書の意味は比較的明確だが、cross-corpus taxonomyがない。

5.6 用語ずれ
PackageとPack

コーパスでは以下が存在する。

Asset Content Package

Runtime Package

Platform package

Package Candidate

Feature Pack

Genre Pack

Support Bundle

Project package相当表現

02-foundation/naming-project-layout.md §2.1は人間語彙の境界を扱うが、Package／Packの全subjectを中央で明確に区別していない。

特に日本語説明中では「パッケージ」がRuntime content、Platform distribution、Feature compositionのいずれか判別しにくい。

Scene

Product §5.0.1は「シーン管理」をEditor上の総称とし、単一SceneManagerを否定する。World文書もMap／Level／Scene／Space／Stageを分離しており、方向は整っている。

一方、ProductのTitle-to-Result説明では一般読者向けのscene／screen表現と、Runtime Entry／World／Stageが混在する。正規語彙と表示語彙の区別を常に局所表示する必要がある。

Apple

07-platform/apple.mdのH1はApple Platform Contractだが、本文のcurrent targetはtarget.apple.mobile、iPhone／iPad、UIKitである。

product-plan.md §4およびFuture portfolioではmacOSは別Future Target Programである。本文を読めば明確だが、文書名とIndex表示だけではApple全体を所有するように見える。

AI Verification／Provenance

文書名はAI限定に見えるが、実際には以下を所有する。

一般Verification lifecycle

Requirement coverage

Performance／reliability

Release Evidence

Supply chain

CI lanes

Product release evidence aggregation

これは内容不足ではなく、Owner discoverability上の名称ずれである。

5.7 循環依存

51 Owner文書の規範依存から抽出したgraphでは、循環依存は検出されなかった。

ただしこれはMarkdown Headerを対象とした解析結果であり、生成Inventoryによる継続的保証ではない。

5.8 欠落依存／リンク

Unicodeを考慮した見出しslug照合で、内部anchor不一致を2件確認した。

appendices/editor-panel-reference-catalog.md §6.3.1

参照: ../06-rendering/world.md#world-level-workspace-boundary

実際の見出し: 06-rendering/world.md §3.1 Level Workspaceの非authority境界

appendices/navigation-design-alignment-review.md §2

参照: ../05-simulation/navigation.md#navigation-current-target-reading

実際の見出し: 05-simulation/navigation.md §1.1 AI／Editor向けcurrent／target読解

添付コーパス外のため検証不能な参照が1件ある。

decisions/2026-07-21-document-system-restructure.md

参照: ../../../.codex/config.toml

添付ZIP外。実Repositoryでは存在する可能性があるため「broken」とは断定せず、「添付範囲では検証不能」とする。

孤立文書は検出されなかった。

5.9 Historical audit数値のfreshness

appendices/architecture-plan-closure-review.md §2には2026-07-28時点の以下の履歴がある。

Markdown 75件

文書ID 73件

Owner 50件

規範依存202 edge

同文書は、2026-07-29にVirtualized Geometryが51件目のOwnerとして追加され、旧数値がhistory Evidenceであってcurrent Inventoryではないことを明記している。

したがって数値矛盾ではない。ただし、51 Ownerを対象とした同等のcurrent再監査Evidenceが存在せず、freshness gapが残っている。

6. Runtime／Editor／Asset／Build／Platform／Security／QAをまたぐ横断gap
6.1 Asset ↔ Runtime ↔ Rendering／Audio／World ↔ Memory

根拠

asset-lifecycle.md §5–§7

runtime-package.md §1、§5.3

scheduling-lifetime.md §5

performance-capacity.md

render-graph.md

audio.md §4.3、§10

world.md §6、§10

virtualized-continuous-geometry.md §7.2

Gap

Cooked artifact availabilityとRuntime residencyの間を所有する汎用authorityがない。

影響

priority、cancel、memory pressure、generation、eviction、device loss、fallbackがconsumerごとに分裂する。

候補Owner

Runtime Asset専用Ownerまたは正式に拡張されたRuntime Package Owner。

文書上の閉鎖条件

全consumerが同じrequest／lease／generation／failure semanticsへ解決し、Performanceはpolicy／measurement、Memoryはallocation rule、Subsystemはconsumerに限定される。

6.2 Editor ↔ Project State ↔ Runtime Entry ↔ UI ↔ Persistence

根拠

project-state.md §3.1.1／§3.1.2

product-plan.md §5.0.1

scheduling-lifetime.md §4.0

runtime-package.md §1.1

persistence-save.md §4.1

ui-text-localization-accessibility.md §8

scenario-stage.md §11

Gap

Runtime Entry selection、transition、staging、publication、UI presentation、Continue reconstructionの一意なcross-owner transition mapがない。ProductがSubsystem詳細を重複規定する。

影響

partial transition、stale generation、cancel semantics、Save reconstructionの重複Authority。

候補Owner

Project State、Scheduling、Runtime Package、Persistenceの明示分担。Productはacceptance Owner。

文書上の閉鎖条件

一つのscenario traceから各Ownerのauthoritative fragmentへ辿れ、同じstate machineを複写しない。

6.3 Asset ↔ World ↔ 2D Rendering ↔ LOD ↔ Collision／Navigation

根拠

asset-lifecycle.md §2

world.md §10.1

render-graph.md §6／§12

materials.md

lod.md

collision.md

navigation.md

Gap

Sprite／Tile sourceからruntime representation、sorting、visibility、collision／navigation extractionまでのcross-owner contractが断片化している。

影響

2Dのauthoritative coordinate、chunk lifetime、render order、collision refresh、navigation invalidationが別々に進化する。

候補Owner

2D runtime representation OwnerをGovernanceで決定。World、Asset、Render、Simulationの境界を参照で接続する。

文書上の閉鎖条件

同じTile／Sprite identityとgenerationを各consumerが参照し、renderとsimulationで別Source解釈を持たない。

6.4 Gameplay AI ↔ Navigation ↔ Input／Command ↔ Animation ↔ Debug

根拠

gameplay-programming-model.md §2.4／§2.5

navigation.md

input.md §4.5

animation.md

debugging-observability-replay.md §11

shooter.md

Gap

Perceptionからdecision、action、interrupt、completion、debugまでのGeneric flowがない。

影響

Genre-specific AIがSubsystem private stateを直接操作し、Replay／Debugで説明できない。

候補Owner

Gameplay Programming Modelを中心に、各execution Ownerへのtyped boundaryを閉じる。

文書上の閉鎖条件

observation、decision、command、execution、failure、explanationが同じStable refsとcausalityへ接続する。

6.5 ECS ↔ Gameplay ↔ Runtime Package ↔ Save ↔ Debug ↔ AI

根拠

gameplay-programming-model.md

entity-component-system.md

ECS Closure Review

runtime-package.md

persistence-save.md

debugging-observability-replay.md

Architecture Explain Projection

Gap

current ECS authorityとtarget Ownerの移管が完了していない。

影響

Component identity、query、structural transaction、Save projection、debug query、AI explanationが異なるrevisionを参照する。

候補Owner

Runtime ECS＋Governance／Compatibility。移管前はGameplay Modelをcurrent authorityとして維持。

文書上の閉鎖条件

consumer集合が同一revisionへ揃い、target記述をcurrentとして先行利用しない。

6.6 Build ↔ Platform ↔ Security ↔ QA

根拠

Core §9

Toolchain §8／§9

AI Security §7／§9／§10

Verification §7／§14

Windows／Android／AppleのBuild／Signing／Qualification節

Gap

共通Build chainとPlatform固有Evidence、Security approval、device session、release gateを一つに比較できるcross-target closure mapがない。

影響

Signing、Upload、Install、Launch、Smoke、device qualificationの一部を省略または相互代用する。

候補Owner

Core、Verification、AI Security、各Platform、Product Planのjoint closure。

文書上の閉鎖条件

各Targetの必須Evidence集合と非代替関係が一意に追跡できる。

6.7 UI／Settings ↔ Security ↔ Debug／Crash ↔ Platform Privacy

根拠

UI §15

Core §9.1

Debug §14／§15

Mobile Common §7

Windows §11／§12

Apple §3

AI Security fixtures

Gap

Consent Record正本がない。

影響

purpose、revoke、freshness、retention、Operation bindingが分散する。

候補Owner

Cross-product Consent／Privacy Owner。

文書上の閉鎖条件

全consumerが同じpurpose-based consent lifecycleへ解決し、Platform declarationと利用者consentを区別する。

6.8 Governance ↔ 全Owner ↔ AI Understanding

根拠

Governance §3、§5、§6

README §1

Closure Review §3

Owner文書全体

Gap

生成Inventoryがなく、Inline根拠適合もsystemicに未確認。

影響

AI、人間、review toolがOwner、current／target、Evidence、dependencyを毎回推測する。

候補Owner

Architecture Governance／Executable Contracts／Verification。

文書上の閉鎖条件

Inventory、Explain Projection、根拠coverage、dependency validationの文書契約が一致する。

6.9 Product ↔ Packs ↔ Reference Evidence

根拠

Product §5.0／§5.1.1／§9.4

Pack Contract §1／§3

Shooter

Product Execution Registry Proposal

Gap

RPG destination Owner群が未materializeで、current Shooter sourceとのatomic migration projectionがない。

影響

Product identity、Reference fixture、Capability、Evidenceのsource／destinationが混在する。

候補Owner

Product、Pack Governance、未materialize RPG Owner群。

文書上の閉鎖条件

Generic、Reusable RPG、RPG Genre、Reference Game、Shooter technical fixtureのEvidenceを相互非代替として追跡できる。

7. Severity付きClosure候補一覧
Severity定義

Critical: 現MVPまたは必須運用経路の正本が欠落・二重化し、Architecture closureを成立させられない。

High: Owner不整合、誤ったGate、Security／Quality／Target差異を生む可能性が高い。

Medium: 直ちにDomain欠落ではないが、用語、freshness、discoverability、保守性を継続的に損なう。

Low: 純粋な表記問題。本監査では独立Low候補を設けず、関連Mediumへ統合した。

MK-CL-001 — RPG-first MVP Owner群の欠落

Severity: Critical

根拠:
00-product/product-plan.md §5.0、§5.1.1、§9.4。
08-packs/pack-contract.md §3.2、§3.3。

影響文書:
Product Plan、Gameplay Feature Packs、Pack Contract、Scenario／Stage、Gameplay Model、UI、Persistence、Runtime ECS、Debug、Verification。

影響:
MVPのbattle、progression、inventory、quest、economyにauthoritative Ownerがなく、Product fixtureだけが先行する。

候補Owner:
Reusable RPG Feature Owner群、RPG Genre Owner、RPG Reference Game Owner。Product Planはoutcome／Gateのみ。

文書上の閉鎖条件:
全RPG要件がGeneric／Reusable／Genre／Referenceのいずれか一つへ解決し、state、request、Save、failure、diagnostic、qualificationが一意になる。

必要Evidence:
Owner responsibility matrix、Product requirement trace、source／destination classification、cross-owner state／request／failure表、Evidence非代替表。

MK-CL-002 — 汎用Runtime Asset authorityの欠落

Severity: Critical

根拠:
04-runtime/runtime-package.md §1、§5.3。
appendices/architecture-plan-closure-review.md ARCH-C02、§9.1。
06-rendering/virtualized-continuous-geometry.md §7.2。

影響文書:
Asset Lifecycle、Runtime Package、Scheduling、Performance、Memory、World、Render Graph、LOD、Virtualized Geometry、Audio、VFX、UI。

影響:
request、cancel、dependency、residency、eviction、device loss、memory pressureが一意Ownerへ解決しない。

候補Owner:
独立Runtime Asset Owner、またはscopeを正式変更したRuntime Package Owner。

文書上の閉鎖条件:
汎用Asset lifecycle、consumer境界、failure、last-valid、generation、lease、budget integrationが一つの正本へ閉じる。

必要Evidence:
Owner Decision、consumer inventory、state transition matrix、failure／cancel truth table、memory pressure／device loss scenario、cross-generation invariant。

MK-CL-003 — 汎用Consent Record authorityの欠落

Severity: Critical

根拠:
07-platform/ui-text-localization-accessibility.md §15。
02-foundation/core-architecture.md §9.1。
04-runtime/debugging-observability-replay.md §14、§15。
07-platform/mobile-common.md §7。
Windows／Apple Privacy節。

影響文書:
UI、Core Operations、AI Security、Debug／Support、Windows、Android、Apple、Mobile Common、Product completion。

影響:
purpose、grant、revoke、expiry、freshness、retention、Operation bindingが未定義。

候補Owner:
Cross-product Consent／Privacy Owner。

文書上の閉鎖条件:
Consent Recordのidentity、purpose、scope、lifecycle、Evidence、non-inheritanceが一意になる。

必要Evidence:
Consent purpose matrix、consumer mapping、grant／revoke state table、policy version binding、negative scenario catalog、privacy declarationとの照合表。

MK-CL-004 — Runtime ECS current／target authority移管未閉鎖

Severity: Critical

根拠:
README.md §1。
03-authoring/gameplay-programming-model.md §3。
04-runtime/entity-component-system.md §1.1。
appendices/runtime-ecs-design-closure-review.md §4、§6。

影響文書:
Gameplay、Scheduling、Runtime Package、Save、Debug、Native Module、Compatibility、AI Projection、RPG／Shooter。

影響:
consumerが異なるECS authority revisionを参照する可能性。

候補Owner:
Runtime ECS、Gameplay Model、Architecture Governance、Compatibility。

文書上の閉鎖条件:
一つのcurrent authority、atomic migration条件、全consumer ref migration、未解決ECS-C項目のOwner解決。

必要Evidence:
current／target owner map、consumer migration inventory、ref resolution report、layout／query／spawn／Save closure review、authority cutover Decision。

MK-CL-005 — Runtime Entry transitionの重複Authority

Severity: High

根拠:
product-plan.md §5.0.1。
project-state.md §3.1.1／§3.1.2。
scheduling-lifetime.md §4.0。
runtime-package.md §1.1。
persistence-save.md §4.1。

影響文書:
Product、Project State、Scheduling、Runtime Package、Persistence、UI、Scenario／Stage。

影響:
staging、commit、cancel、publication、generationの正本が複数に見える。

候補Owner:
Productはoutcome、Project Stateはsource definition、Schedulingはtransition、Runtime Packageはpublication、PersistenceはContinue。

文書上の閉鎖条件:
同一state machineの複写をなくし、Product scenarioからOwner fragmentへ参照可能にする。

必要Evidence:
Runtime Entry authority matrix、event／state ownership表、cancel／failure truth table、Product acceptance trace。

MK-CL-006 — Architecture Inventory／Explain Projection未materialize

Severity: High

根拠:
README.md §1。
architecture-governance.md §5。
architecture-plan-closure-review.md §3.1、§8。

影響文書:
全76文書。

影響:
Owner、current／target、Decision、Capability、Evidenceを機械的に一意解決できない。

候補Owner:
Architecture Governance＋Executable Contracts。Security／Verificationはprojection policyを補助。

文書上の閉鎖条件:
Inventory input、output、authority、staleness、duplicate、unresolved、explain semanticsが文書上閉じる。

必要Evidence:
全Owner inventory snapshot、dependency validation report、current／target projection例、stale／duplicate negative examples。

MK-CL-007 — Inline根拠Governanceへのsystemic不適合

Severity: High

根拠:
architecture-governance.md §3.1。
51 Owner中42件でliteral 根拠:なし。

影響文書:
全Owner。特に外部Version、数値、Security、Accessibility、Store／Distribution条件を含む文書。

影響:
external fact、project decision、provisional value、Gateの根拠を局所解決できない。

候補Owner:
Architecture Governance＋各Owner。

文書上の閉鎖条件:
§3.1対象の全断定を同一ルールで分類し、HeaderとInlineの関係を明示する。

必要Evidence:
Owner別根拠coverage report、未分類断定一覧、外部根拠freshness一覧、provisional数値一覧。

MK-CL-008 — 規範依存の層順序違反

Severity: High

根拠:
architecture-governance.md §4。
違反候補3 edge:
Product→Governance、AI Security→Executable Contracts、AI Verification→Executable Contracts。

影響文書:
Product Plan、Architecture Governance、AI Security、AI Verification、Executable Contracts、README。

影響:
Governance validatorが正当なmeta-dependencyと不正reverse dependencyを区別できない。

候補Owner:
Architecture Governance。

文書上の閉鎖条件:
dependency type、矢印意味、meta-layer例外またはlayer順序を一意に決定し、未承認違反を0件にする。

必要Evidence:
51 Owner dependency graph、layer validation結果、例外一覧、cycle／unresolved report。

MK-CL-009 — 2D Sprite／Tile runtime authorityの分散

Severity: High

根拠:
render-graph.md §6／§12。
world.md §10.1。
Asset、Materials、LOD、Collision、Navigation。

影響文書:
Product 2D portfolio、Asset、World、Render Graph、Materials、LOD、Animation、Collision、Navigation、Runtime Asset。

影響:
runtime identity、sorting、batch、visibility、animation binding、atlas residencyが一意でない。

候補Owner:
2D runtime Owner、または正式拡張されたRender Graph／Scene extraction Owner。

文書上の閉鎖条件:
Sprite／TileのSourceからRuntime packetまでのauthority chainを一意にする。

必要Evidence:
2D owner matrix、sorting authority表、Tile generation trace、render／simulation consistency scenario、2D qualification matrix。

MK-CL-010 — Gameplay AI decision／action横断契約不足

Severity: High

根拠:
gameplay-programming-model.md §2.4／§2.5。
Navigation、Input、Animation、Debug、Shooter。

影響文書:
Gameplay Model、Shooter、将来RPG NPC、Navigation、Animation、Input、Replay／Debug。

影響:
PerceptionからAction、interrupt、failure、explanationまでが閉じない。

候補Owner:
Gameplay Programming Model。Subsystemはexecution consumer。

文書上の閉鎖条件:
observation、memory、decision、command、interrupt、completion、failure、debug authorityを一意化する。

必要Evidence:
end-to-end behavior trace、state ownership表、interrupt／failure scenarios、Replay／Debug projection、Genre-neutrality review。

MK-CL-011 — Product-wide Security／Vulnerability governanceの分散

Severity: High

根拠:
AI Security、Toolchain、Asset、Native Module、Project Shader、Debug、Platform Security節。

影響文書:
全Engine／Editor／Build／Platform／Package domain。

影響:
非AI attack surface、脆弱性、revocation、support policyの総合Ownerがない。

候補Owner:
独立Product Security Owner、またはscopeを明確に拡張したSecurity Governance Owner。

文書上の閉鎖条件:
trust boundary、vulnerability lifecycle、severity、release impact、incident feedbackを横断的に定義する。

必要Evidence:
attack-surface inventory、security responsibility map、vulnerability state model、release／revocation impact matrix、domain security coverage report。

MK-CL-012 — QA／Qualification aggregation policy不足

Severity: High

根拠:
Core §12、Verification §2／§3／§14、各Owner Test節、Platform fixture。

影響文書:
全OwnerのQualification、Product Gate、CI、Release Evidence。

影響:
flaky、retry、quarantine、infrastructure failure、device session、release blockの扱いが統一されない。

候補Owner:
AI Verification／Provenanceの一般Verification authority。

文書上の閉鎖条件:
Test class、Evidence class、aggregation、非代替、freshness、waiver／quarantineを一意化する。

必要Evidence:
test taxonomy、lane-to-gate matrix、flaky／retry policy、device session evidence map、false-pass negative scenarios。

MK-CL-013 — Toolchain／Target／CI／Device baseline未閉鎖

Severity: High

根拠:
toolchain-dependencies.md §6.1、§6.2、§8.1。
Closure Review §6。

影響文書:
Product Target、Windows、Android、Apple、Build、Performance、Verification。

影響:
minimum OS、Dependency、runner、device pool、capacity、optimizationのreadinessを判断できない。

候補Owner:
Toolchain Owner＋各Platform＋Product＋Verification。

文書上の閉鎖条件:
Target別未固定項目、開始不能Gate、Evidence requirementが一意のreadiness projectionへ解決する。

必要Evidence:
Target readiness matrix、Dependency decision register、CI／device capacity record、minimum OS consistency report、provisional／qualified分離表。

MK-CL-014 — Product PlanにArchitectureとexecution planningが混在

Severity: High

根拠:
product-plan.md §5.1、§6、§11。
Product Execution Registry Proposal。

影響文書:
Product Plan、Governance、Toolchain、各Subsystem、Product Registry Proposal。

影響:
Phase、critical path、Work PackageをArchitecture dependencyや実装順と誤認する。

候補Owner:
Product Plan＋Architecture Governance＋専用planning authority候補。

文書上の閉鎖条件:
Product outcome、Architecture dependency、execution trackingを明確に分離する。

必要Evidence:
responsibility split、用語分類、Architecture dependencyとの非同値表、Product requirement trace。

MK-CL-015 — Shooter source／RPG destination projection未materialize

Severity: High

根拠:
product-plan.md §5.1.1。
pack-contract.md §1。
Product Execution Registry Proposal。
ECS Closure Review ECS-C20。

影響文書:
Product、Shooter、Pack Contract、未materialize RPG Owners、Verification、Registry。

影響:
source baseline、destination Product identity、Evidenceの読解が手動になる。

候補Owner:
Product Plan＋Pack Governance＋将来RPG Owners。

文書上の閉鎖条件:
source／destination全row、Owner、Capability、Evidence、non-reuse条件が一つのmigration definitionへ閉じる。

必要Evidence:
source／destination diff、Evidence disposition表、Owner migration map、dual-current negative scenario。

MK-CL-016 — Target横断Build／Release Evidence closure不足

Severity: High

根拠:
Core §9.2、Toolchain、AI Security、Verification、Windows／Android／Apple。

影響文書:
Build、Package、Install、Launch、Smoke、Signing、Upload、Product completion。

影響:
Target固有Evidenceの抜け、別stage／別Target Receipt流用の危険。

候補Owner:
Core＋Verification＋AI Security＋各Platform＋Product。

文書上の閉鎖条件:
各Targetの共通／固有Evidence集合と非代替関係を一意に追跡できる。

必要Evidence:
Target別closure matrix、Receipt predecessor graph、negative substitution cases、Product completion binding。

MK-CL-017 — Promotion／Activation／Qualificationの状態語彙衝突

Severity: Medium

根拠:
Product §3／§7、AI Security §4／§10、Asset §7、Native Module、Gameplay §9、各Platform Gate。

影響文書:
全状態遷移を扱うOwner。

影響:
異なるsubjectのstateを相互に読み替える。

候補Owner:
Architecture Governance＋Naming。

文書上の閉鎖条件:
各状態語のsubject、Owner、Evidence、非包含関係をtaxonomy化する。

必要Evidence:
state-axis matrix、unqualified term usage一覧、Explain Projection例。

MK-CL-018 — Package／Pack語彙の衝突

Severity: Medium

根拠:
Naming §2.1／§6、Asset §6、Runtime Package、Pack Contract、Platform package文書。

影響文書:
Asset、Runtime、Platform、Pack、Product、Support。

影響:
content artifact、runtime closure、distribution、feature compositionを混同する。

候補Owner:
Naming／Project Layout。

文書上の閉鎖条件:
人間語彙とtechnical identifierの対応を定義し、無修飾語の使用規則を決める。

必要Evidence:
vocabulary table、ambiguous usage inventory、cross-doc glossary validation。

MK-CL-019 — AI Verification／Provenanceの名称とscope不一致

Severity: Medium

根拠:
01-governance/ai-verification-provenance.md全体、特に§2、§3、§6、§9、§14。

影響文書:
README、全OwnerのVerification参照、Product Release。

影響:
一般QA／Release Evidence Ownerが発見しにくく、AI専用文書と誤認される。

候補Owner:
Architecture Governance＋同Verification Owner。

文書上の閉鎖条件:
名称、Index説明、scope宣言のいずれから読んでも一般Verification authorityが明確になる。

必要Evidence:
reference inventory、scope mapping、Owner discovery review。

MK-CL-020 — Apple文書のTarget scope名称

Severity: Medium

根拠:
07-platform/apple.md H1／§1。
product-plan.md §4およびmacOS Future entry。

影響文書:
README、Toolchain、Product Future targets、Apple Platform Contract。

影響:
Apple mobileとmacOS desktopのOwnerを混同する。

候補Owner:
Apple Platform Owner＋Naming／Index。

文書上の閉鎖条件:
H1、Index、scope、Target IDが一貫してmobile-only current authorityを示す。

必要Evidence:
Target ownership map、macOS Futureとのnon-overlap確認。

MK-CL-021 — Architecture closure reviewのcurrent freshness不足

Severity: Medium

根拠:
appendices/architecture-plan-closure-review.md §2。

影響文書:
Closure Review、README、Governance、Virtualized Geometry。

影響:
75／50 Owner時点の履歴をcurrent corpus証拠として誤利用する。

候補Owner:
Architecture Governance／Review文書Owner。

文書上の閉鎖条件:
history Evidenceとcurrent Inventoryを明確に分離し、51 Owner current結果への参照を用意する。

必要Evidence:
current count、dependency edge、link、cycle、document stateの再監査結果。

MK-CL-022 — 相対anchor不一致と添付外参照

Severity: Medium

根拠:
editor-panel-reference-catalog.md §6.3.1。
navigation-design-alignment-review.md §2。
decisions/2026-07-21-document-system-restructure.md。

影響文書:
Editor Panel Catalog、World、Navigation Review、Navigation、Decision。

影響:
正本見出しへ到達できず、AI／readerが近似見出しを推測する。

候補Owner:
各source文書Owner＋Architecture Governance。

文書上の閉鎖条件:
内部anchorが実見出しへ解決し、添付外参照には検証境界が明示される。

必要Evidence:
全relative link validation report、out-of-corpus link inventory。

MK-CL-023 — Owner／Appendix長大化と検索性

Severity: Medium

根拠:
architecture-governance.md §7.1のOwner文書原則1000行未満。

1000行超:

01-governance/ai-security-approval.md 1074行

03-authoring/editor-ui-framework.md 1085行

appendices/ai-provider-mcp-security-supplement.md 1071行

appendices/product-execution-registry-proposal.md 1258行

appendices/executable-contracts-operation-planning-catalog.md 1931行

appendices/editor-ui-design-system-catalog.md 2551行

影響文書:
上記文書、README、Inventory／Explain Projection。

影響:
一意Ownerは維持されていても、局所authority、重複、stale sectionを検出しにくい。

候補Owner:
各文書Owner＋Architecture Governance。

文書上の閉鎖条件:
自動分割ではなく、§7の分割／統合基準に照らし、検索可能なfragment、catalog、anchor、Owner境界を保証する。

必要Evidence:
section responsibility map、重複断定検査、fragment reference coverage、document-size exception rationale。

8. 十分に記述されており、むやみに増補すべきでない領域

以下は、本監査で新しい概念やOwnerを安易に追加すべきでない領域である。特定のClosure候補がある場合も、既存設計を全面再構築するのではなく、その未閉鎖点に限定すべきである。

8.1 文書状態と実装状態の分離

README.md §1とarchitecture-governance.md §2は、review、absent、target、current、proposalを区別し、Markdown上の型を実装と扱わない原則を明確にしている。

不足は概念ではなく適用・Inventory・Evidenceである。状態モデルをさらに増やすより、既存状態軸を一貫して説明・検証すべきである。

8.2 AI Security／Approval

ai-security-approval.mdは、Actor、Trust boundary、Task authorization、Risk class、Sandbox、Filesystem、Network、Credential、Provider、MCP、CLI、Preview、Qualification、Approval、Promotion、Failure、negative fixtureまで非常に詳細である。

Product-wide Security Owner不足を理由に、この文書へ無制限に全Security責務を追加すべきではない。AI-specific authorityを維持し、一般Securityとの境界を明確にする方がよい。

8.3 Core Build GatewayとOperation chain

core-architecture.md §9は、OperationTask、Authorization、Project revision、Candidate root、Target、Device、Receipt predecessor、Package→Install→Launch→Smoke、Resetまで詳細に閉じている。

不足はTarget横断projectionとConsent authorityであり、別のBuild operation体系を追加すべきではない。

8.4 Project State／ChangeSet／Undo／Recovery

project-state.mdはProject aggregate、Document、ChangeSet、commit、undo／redo、external edit、conflict、AI／manual editingを簡潔に所有している。

Runtime Entryの重複を解く際もProject Stateの正本を増築し過ぎず、source definitionとauthoring transactionへ限定すべきである。

8.5 Asset Source／Import／Cook／Catalog

asset-lifecycle.mdはSource、Import、Profile、IR、Preview、Reimport、Derived、Cooked、Catalog、VFS、Content Package、promotion、security、qualificationまで十分に広い。

Runtime Asset gapを理由にAsset LifecycleへRuntime residencyを追加すると、Source／CookとRuntime lifetimeが再び混ざる。既存境界を維持すべきである。

8.6 Editor UI Framework／Workspace

Editor UI Framework、Workspace UX、Design System Catalog、Panel Catalogは、UI tree、layout、event、rendering、IME、DPI、dock、accessibility、automation、AI semantic interface、Panel role、failure／recoveryを非常に詳細に扱う。

文書長と一部anchorの問題はあるが、機能概念の追加より、Owner fragment、catalog参照、根拠、navigationの整理を優先すべきである。

8.7 Scheduling／Lifetime／Cadence

scheduling-lifetime.mdはprocess、Play、World、Simulation cadence、phase、clock、pause、Runtime Entry transition、render／audio／asset activation、job lifetime、message order、failureを詳細に扱う。

Runtime AssetやRuntime Entry closureのために別の全体Scheduler概念を追加すべきではない。既存Scheduling authorityへ正しく接続すべきである。

8.8 Persistence／Replay／Debugging

PersistenceはSave identity、record、digest、reconstruction、migrationを所有し、Debuggingはevent、causality、capture、replay transport、crash、support bundleを所有する。両者のsemantic Save／Replayとdiagnostic captureの区別は良好である。

Consent Owner不足をDebugging文書内だけで解決しようとせず、Support Bundle固有consumerとして維持すべきである。

8.9 Collision／Physics／Navigation／Animation

Simulation四領域は、意味、backend境界、artifact、lifetime、failure、qualificationを十分に分離している。

Gameplay AI gapを理由にNavigationやAnimationへdecision authorityを追加すべきではない。これらはtyped requestを実行するSubsystemとして維持するべきである。

8.10 3D Rendering領域

Render Graph、Materials、Project Shader、Lighting、Post Processing、VFX、Camera、Environment、LOD、Virtualized Geometryは広範かつ詳細である。

Runtime Asset gapと2D authority gapを除けば、3D機能をさらに列挙するより、既存CapabilityのTarget matrix、consumer binding、Evidenceを閉じるべきである。

特にVirtualized Geometryは独立設計として十分に深く、汎用Runtime Asset authorityを同文書内で独自生成しないという判断も適切である。

8.11 UI／Text／Localization／Accessibility

Game UI文書はUI document、binding、event、focus、text input、Localization、Text layout、glyph cache、semantic tree、Accessibility、Settings、AI authoring、failure、testを一つのcoherent domainとして扱っている。

Consent Record不足を除き、別のLocalization OwnerやAccessibility Ownerを安易に追加するとauthorityが重複する可能性が高い。

8.12 Platform package／signing separation

Windows、Android、Appleはいずれも、Build、Package、Signing、Upload、Device、Privacy、failure、release gateを詳細に持つ。特にsecret separationとnegative fixtureは強い。

不足は共通cross-target mapとToolchain／Device readinessであり、Platform固有契約を共通文書へ吸収すべきではない。

8.13 Pack layeringの原則

Pack ContractはGeneric Core、Feature Pack、Genre Pack、Project／Referenceの依存方向を明確にし、RPG-firstをCore依存へしないと規定する。

RPG Owner不足は、この原則を変更する理由ではない。むしろ既存の四層分離に従って不足Ownerを閉じるべきである。

9. 次の反証監査で優先検証すべき論点

以下は反証監査の検証優先順位であり、実装順、実装計画、日程、担当、ロードマップではない。

9.1 RPG要件が既存Generic Ownerだけで本当に表現できるか

重点的に反証すべき仮説:

Generic Gameplay Featureだけでbattle、progression、inventory、quest、economyを所有できる。

Scenario／StageだけでRPG progressionを表現できる。

Shooter CombatやPickupを再利用すれば新Ownerは不要。

UI／Save文書だけでRPG state transactionが閉じる。

探すべき反例:

battle commandのatomicityとturn ownership

equipment変更とderived statの同時publication

inventory capacityとcurrency transaction

Quest flagとdialogue choiceのSave identity

item useとbattle／outside-battleのauthority差

Shop reject時のpartial mutation

locale変更がgameplay identityへ影響しないこと

9.2 Runtime Asset authorityが既存Runtime Packageだけで十分か

重点的に反証すべき仮説:

World／Runtime Entry loaderを拡張すれば全Asset consumerを自然に所有できる。

Rendering、Audio、UI、VFXの局所cacheで十分。

Package availabilityがresidencyを暗黙保証できる。

探すべき反例:

Worldless UI-only Runtime EntryでのFont／Texture request

Audio streamingだけが期限を持つ場合

複数ViewのLOD／Virtual Geometry request競合

device loss後のGPU再upload

generation切替中のold lease

memory pressure時のroot／fallback優先度

cancelとdependency completionのrace

hot reloadとSave／Replay非関与

9.3 Consent Recordのpurpose分離とrevoke

重点的に反証すべき仮説:

Settings boolでConsentを表現できる。

Platform Privacy ManifestがConsent Evidenceになる。

Support Bundle consentをCrash／Telemetry／AIへ流用できる。

install consentをresetへ流用できる。

探すべき反例:

policy text version変更

locale変更

consent後のProject purpose変更

queued upload中のrevoke

Device交換

user data reset後のstale Consent ref

AI Provider変更

crash uploadは許可、analyticsは拒否

offline operationとnetwork operationのscope差

9.4 ECS移管が本当にatomicか

重点的に反証すべき仮説:

Runtime ECS文書を正本指定するだけで移管できる。

Component identityだけ移せば他consumerは追従できる。

Save／ReplayはECS layoutから独立なので移行不要。

探すべき反例:

current Gameplay Model typeを参照するSave fixture

target ECS query cacheを前提にするDebug projection

Runtime Package construction recordのrevision差

Native Module access manifest差

persistent identityとruntime spawn identityの衝突

migration途中のdual component owner

RPG／Shooter fixtureが別revisionを参照する状態

9.5 Runtime Entry transitionのfailure atomicity

重点的に反証すべき仮説:

Product §5.0.1とSubsystem文書の意味は完全に同一。

branch replacementとStage transitionは常に明確に区別される。

commit後cancelは全Ownerで同じ意味。

探すべき反例:

destination World成功、UI失敗

Save reconstruction成功、Package generation stale

Stage completion後、Result UI package不足

duplicate transition request

same SessionでTitleへ戻る場合

commit直前のDevice／Target drift

Loading UI終了とpublication failureの競合

source teardown中のfatal diagnostic

9.6 2Dをfirst-classとする契約の実効性

重点的に反証すべき仮説:

Render Graphの2D layer記述だけでfirst-class 2Dが成立する。

SpriteとTileを同じpacket modelで扱える。

2D sortingはpresentationだけなのでGameplay／Worldと無関係。

探すべき反例:

isometric Y-sortとWorld coordinate

Tile chunk replacementとcollision／navigation generation

pixel-art cameraとsubpixel motion

Sprite animation frameとhit shape

atlas hot reload中のrender generation

2D light／mask／material ordering

multiple camera／UI composition

transparent sortingとbatch効率の衝突

9.7 Build／Package／Install／Release Evidenceの非代替性

重点的に反証すべき仮説:

Common Build Receiptが全Targetへ使える。

Simulator passでMobile release gateを部分代替できる。

Signing成功ならpackage contentも正しい。

clean installとupgradeは同じfixtureで十分。

探すべき反例:

-同一Candidate rootでToolchain lock差

package hash同一でもentitlement／manifest差

device generation差

install成功、launch failure

launch成功、offline content不足

upgrade成功、uninstall／reinstallでstale state復活

signing key revocation後の過去Receipt

Store upload成功、device session未実施

9.8 QAのfalse-pass経路

重点的に反証すべき仮説:

retry成功はpassとして問題ない。

screenshot一致はUI correctnessを示す。

unit／headless passでdevice lifecycleも十分。

performance平均値だけでregressionを判定できる。

探すべき反例:

flaky failureがcandidate-specific

infrastructure unavailableをskip扱い

screenshot一致だがfocus graph不正

semantic hash一致だがlocale layout overflow

meanは合格、tail latencyは不合格

warm cacheのみ合格

simulatorのみ合格

test順序依存

stale fixture／golden

quarantine中testをProduct Gateが無視

9.9 Product-wide SecurityのOwner漏れ

重点的に反証すべき仮説:

AI SecurityとPlatform Securityを合成すれば全体Securityが閉じる。

untrusted AssetはImport時だけ検査すればよい。

Package署名がSave／Project／contentの安全性も保証する。

探すべき反例:

malicious Project document

decompression bomb

path／Unicode／case collision

malformed Save

shader／native source isolation failure

signedだがvulnerable dependency

revoked package

support対象外version

local-only attack surface

diagnostic／crashへのsecret流出

incident後のCapability／Receipt revocation

9.10 Gameplay AIの説明可能性と中断

重点的に反証すべき仮説:

Rule／FSMがあればPerceptionからActionまで十分。

Navigation failureはNavigationだけの問題。

AI decision stateはReplay対象外でよい。

探すべき反例:

stimulus expiration中のaction継続

higher-priority eventによるinterrupt

navigation path invalidation

animation lock中のcancel

target消失

perception source conflict

deterministic replayでdecision差

Debugが「なぜその行動を選んだか」を説明できない状態

9.11 Governance validator自身の反証

重点的に検査すべきもの:

3件のlayer-order violationが本当に例外で閉じるか

42文書のInline根拠なしが規則上許容される箇所か

Owner文書1000行原則の例外根拠

Proposalがcurrent正本として参照されていないか

target型がcurrent consumerに流入していないか

unresolved Decisionをclosed-in-target-designと表示していないか

broken anchor以外にUnicode slug差がないか

historical Evidenceをcurrentとして使っていないか

9.12 状態・用語の誤読耐性

人間とAIの双方に対して、次の問いへ一意に答えられるかを検証する必要がある。

reviewだがdesign-reviewedとは何を意味するか

current authorityだが実装absentとは何を意味するか

target Ownerは現在の正本か

qualifiedは何のsubjectか

promotion対象はAsset、Source、Capability、Releaseのどれか

PackageはRuntime、Content、Platform distributionのどれか

AppleはmobileかmacOSを含むか

AI Verificationは一般QAを所有するか

RPG-first approved directionはRPG Capabilityがactiveという意味か

closed-in-target-designはDecision、Schema、Artifact、Evidenceのどこまでを含むか

第1次監査結論

Miraikanai EngineのArchitecture文書は、広さと深さの両方で既に高水準にある。特に、fail-closed、typed identity、revision、Evidence、Authorization、Platform package、Accessibility、failure recoveryを実装前から扱っている点は強い。

現在の主要問題は「機能項目をさらに大量追加すること」ではない。

文書上閉じるべき中心課題は次の四つである。

RPG-first MVPを支える正本Owner群をmaterializeするための判断を閉じること

汎用Runtime Asset request／residency authorityを一意化すること

汎用Consent Record authorityを一意化すること

Runtime ECSのcurrent／target authority移管をatomicに閉じること

その周囲に、Runtime Entry重複、Architecture Inventory、Inline根拠、依存層順序、2D runtime authority、Gameplay AI、Product-wide Security、QA aggregation、Toolchain readinessがHigh severityのclosureとして存在する。

これらが文書上閉じるまでは、Architectureの方向性は強いものの、MVP全体の一意Authority、横断Gate、Evidence chainが完成したArchitecture baselineとは評価できない。
