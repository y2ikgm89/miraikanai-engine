# ChatGPT Pro Advanced Rendering／Multiplayer Atomicity Audit

- 実施日: 2026-07-29
- 対象: Advanced Rendering／MultiplayerのOwner境界、Future Portfolio原子性、GI／Reflections分離
- 保存方針: 以下はChatGPT Pro応答全文。Repositoryへの採否は別のadjudication文書で扱う

---

最終判定

Bを採用します。

Future Portfolioは31行とし、
future.capability.production-global-illuminationと
future.capability.production-reflectionsを別Future行にします。

前回答および承認済み設計判断にある次の結論は、明示的に撤回します。

future.capability.production-global-illumination-reflectionsを一行にする判断

Future総数を30行とする判断

「二つの複合Future IDを六つの原子的IDへ分解した」とする完了条件

Future行には一つのfuture_capability_id、Owner、Target候補、Activation trigger、前提Future、除外Product claimが置かれます。また、昇格後のActivationは{capability_id, target_id}単位です。したがって、Target supportとActivationが独立する二つのsemantic channelを一行へ戻すことは、Future行をProduct上の原子的な昇格単位として扱う構造と両立しません。

product-plan(6)


product-plan(6)

Advanced Light Transport Ownerが同じであることは、一つのFuture行に統合する根拠にはなりません。同Owner内でも、diffuse indirectとspecular indirectは別semantic channelであり、technique、Target capability resolution、fallback ladder、Qualificationを別々に持つ設計です。

2026-07-29-advanced-rendering-m…

1. 前回答の訂正
前回答・現承認案	訂正
production-global-illumination-reflectionsを一行にする	廃止し、GIとReflectionsを二行へ分離する
Renderingの旧一行を三行へ分割	旧一行を四行へ分割する
二つの旧複合行を六つの原子行へ分割	二つの旧複合行を七つの原子行へ分割する
Future総数30	Future総数31
AAAのRendering系Future前提は三件	四件
Reflection用Product claimなし	claim.product.feature.production-reflectionsを新設する
Owner総数59	59のまま。変更なし

承認済み設計判断の§7.1は、旧Rendering複合行をTerrain、Foliage、GI＋Reflectionsの三行へ分けています。これはTerrain／Foliage／GIの複合を解消する一方で、GIとReflectionsという独立channelを再度一つの行へ結合しています。

2026-07-29-advanced-rendering-m…

同文書の完了条件にある「六つの原子的ID」「全30行」も、この訂正に伴って成立しません。

2026-07-29-advanced-rendering-m…

2. AとBの最終比較
評価基準	A: GI＋Reflectionsを一行	B: GIとReflectionsを別行
独立Activation	不合格。同一TargetでGIだけを昇格し、Reflectionsを未昇格にする状態を一つのCapability IDでは表現できない	合格。各CapabilityのTarget行を独立に昇格・降格できる
Product claimの非過大主張	不合格。行を昇格すると名称上GIとReflectionsの両方を成立させたように読める	合格。GI claimとReflection claimを一対一で管理できる
Target差	不合格。例えばGIはTarget Aでqualified、Reflectionsは同Targetでunsupportedという状態を表せない	合格。同一Targetでも異なるActivation state、fallback、Qualificationを保持できる
fallback	不合格。GI fallbackとReflection fallbackの成功／失敗が一つの行のActivation fateへ結合される	合格。各channelのfallback closureを独立評価できる
Future DAG	不合格。GIだけを必要とするconsumerとReflectionsだけを必要とするconsumerが同じ粗い前提を持つ	合格。consumerが必要なsemantic capabilityだけをexact IDで参照できる
promotion manifest	救済にならない。一つのFuture行から二つのActive Capabilityを出すなら、その二つを識別する子IDが必要であり、実質的にBと同じになる	明確。Future IDと昇格先Capabilityの対応を一対一にできる
将来migration	不合格。一方だけが公開された後で分割すると、公開Capability、claim、Target state、consumer prerequisiteのmigrationが必要になる	合格。全体がplanning_only／absentの現在、正しいV1をclean-breakで固定できる
AI可読性	不合格。「行が有効」と「どちらのchannelが有効か」を別途推測させる	合格。IDだけで意味、claim、Target state、依存を一意に追跡できる
Aをpromotion manifestで補う案が成立しない理由

Aの一行から、promotion manifestによって次の二つを別々に昇格させる設計も考えられます。

production-global-illumination-reflections
  -> active capability: global-illumination
  -> active capability: reflections

しかしこの場合、consumerが次の前提を参照したとき、

prerequisite_future_capability_refs:
  future.capability.production-global-illumination-reflections

それが次のどれを要求するか一意になりません。

GIだけ

Reflectionsだけ

GIとReflectionsの両方

manifestが昇格させた任意のsubset

これを解消するには、manifest内またはFuture Registry内にGIとReflectionsの別IDを設け、consumerもそのIDを参照する必要があります。それは構造上Bそのものです。

したがって、Aは「manifestで補える小さな簡略化」ではなく、Future Registryの外側へ原子性を隠す二重表現になります。

3. 正確なFuture総数

旧Future Portfolioは26行です。旧複合行は次の二件です。

Rendering複合行
future.capability.production-terrain-foliage-gi

Multiplayer複合行
future.capability.headless-dedicated-server-session-transport-replication

Bでは次のように置換します。

Domain	旧行数	新行数	増分
Advanced Rendering	1	4	+3
Multiplayer	1	3	+2
合計	2	7	+5

したがって、正確な計算は次です。

26 - 2 + 7 = 31

最終Future行数は31です。

新Ownerは既定どおり4件のままなので、Owner総数は次のままです。

55 + 4 = 59
4. clean-break後の七つの原子的Future ID
Advanced Rendering: 4行
Future ID	Owner
future.capability.production-terrain	mirakan.arch.rendering-terrain-foliage
future.capability.production-foliage	mirakan.arch.rendering-terrain-foliage
future.capability.production-global-illumination	mirakan.arch.rendering-advanced-light-transport
future.capability.production-reflections	mirakan.arch.rendering-advanced-light-transport
Multiplayer: 3行
Future ID	Owner
future.capability.headless-dedicated-server-target	mirakan.arch.runtime-package
future.capability.network-transport-connection	mirakan.arch.network-transport-connection
future.capability.multiplayer-authority-replication	mirakan.arch.multiplayer-authority-replication

次のIDは廃止します。

future.capability.production-terrain-foliage-gi
future.capability.production-global-illumination-reflections
future.capability.headless-dedicated-server-session-transport-replication

これらをalias、umbrella prerequisite、fallback ID、promotion manifest上の別名として残しません。承認済み設計判断も、公開Schemaと実装がabsentである現在は正しいV1を直接設計し、aliasやdual readを残さない方針です。

2026-07-29-advanced-rendering-m…

5. 各Rendering Future行の一意なActivation単位
Future ID	独立してQualification・Activationする意味	含めない意味
future.capability.production-terrain	Production TerrainのSource、artifact、runtime、Target fallback	Foliage、GI、Reflections
future.capability.production-foliage	Production FoliageのSource、artifact、runtime、Target fallback	Terrain、GI、Reflections
future.capability.production-global-illumination	diffuse indirect illumination channel、同channelのtechnique、Target support、fallback、Qualification	specular reflection、advanced shadow、ray tracing一般、path-traced runtime一般
future.capability.production-reflections	specular reflection channel、同channelのtechnique、Target support、fallback、Qualification	diffuse GI、advanced shadow、ray tracing一般、path-traced runtime一般
future.capability.aaa-photoreal-rendering	複数Owner Receiptを集約したAAA／photoreal Product quality claim	個別のTerrain、Foliage、GI、Reflection techniqueやProviderの所有

GIとReflectionsは同じALT Ownerに属しますが、相互の前提にはしません。

future.capability.production-global-illumination
  prerequisite_future_capability_refs = []

future.capability.production-reflections
  prerequisite_future_capability_refs = []

ここでの[]は、両者の間にFuture prerequisite edgeを置かないという意味です。既存Active Capabilityへのprerequisiteは各行で別途保持できますが、GIのActivationをReflectionsへ、ReflectionsのActivationをGIへ従属させません。

6. AAA／photorealの正確なFuture prerequisites

future.capability.aaa-photoreal-renderingのFuture前提は、exactly次の四件です。

prerequisite_future_capability_refs = {
  future.capability.production-terrain,
  future.capability.production-foliage,
  future.capability.production-global-illumination,
  future.capability.production-reflections
}

次は前提に含めません。

future.capability.virtualized-continuous-geometry-lod
ray-tracing technique family
path-traced reference / preview / runtime profile
neural reconstruction technique
特定Provider
特定graphics backend

Virtualized Geometry、ray tracing、path tracing、neural reconstructionは、AAA／photoreal quality rubricを達成する候補になり得ますが、一律のProduct prerequisiteにはしないという既存判断を維持します。

2026-07-29-advanced-rendering-m…

Target別の厳密な成立条件

Target tについてAAA／photorealを昇格させるには、四つの前提がすべて**同じTarget t**で、AAA promotion manifestが要求するActivation stateを満たしていなければなりません。

AAA(t)
  requires Terrain(t)
  requires Foliage(t)
  requires GlobalIllumination(t)
  requires Reflections(t)

次のcross-target合成は禁止します。

GlobalIllumination(desktop)
+ Reflections(console)
!= AAA(desktop)
!= AAA(console)

例えば、あるTargetでGIだけがqualified、Reflectionsがunsupportedである場合は次の状態が正規です。

production-global-illumination(Target) = qualified
production-reflections(Target) = not_activated | unsupported
aaa-photoreal-rendering(Target) = not_activated

Aではこの状態を一つのproduction-global-illumination-reflections行で表現できません。Bでは、Product Planが要求するTarget別Activation stateをそのまま表現できます。

product-plan(6)

現在のAAA行はcandidate targetとしてdesktop; consoleを持つため、AAAの当該Targetを有効な候補として残すなら、四つの前提行もそのTargetをcandidateとして表現可能でなければなりません。前提行に該当Targetが存在しない状態でAAAだけにTargetを列挙することは、到達不能なFuture DAGになります。現在の旧複合行とAAA行のTarget集合には、すでにこの整合確認が必要です。

product-plan(6)

7. exact excluded_current_product_claims[]
推奨する一対一対応
Future ID	excluded_current_product_claims[]
future.capability.production-terrain	{ claim.product.feature.production-terrain }
future.capability.production-foliage	{ claim.product.feature.production-foliage }
future.capability.production-global-illumination	{ claim.product.feature.production-gi }
future.capability.production-reflections	{ claim.product.feature.production-reflections }
future.capability.aaa-photoreal-rendering	{ claim.product.quality.aaa-visual-parity, claim.product.quality.photoreal-rendering-guarantee }

旧Rendering複合行はTerrain、Foliage、GIの三claimだけを持ち、Reflection claimを持っていません。

product-plan(6)

新設するclaim定義

ProductClaimDefinitionRegistryV1には、次の一件を追加します。

claim_id:
  claim.product.feature.production-reflections

canonical_label:
  production reflections

claim_scope:
  feature

Product Planは、Future行のexcluded_current_product_claims[]とProduct Claim RegistryのID集合をset equalityにするため、Reflection Future行だけを追加してclaim IDを追加しない構成は閉じません。

product-plan(6)

claimを混在させない規則
production-global-illumination
  excludes/releases:
    claim.product.feature.production-gi
  does not exclude/release:
    claim.product.feature.production-reflections

production-reflections
  excludes/releases:
    claim.product.feature.production-reflections
  does not exclude/release:
    claim.product.feature.production-gi

AAA行は個別Feature claimを再所有しません。

aaa-photoreal-rendering
  excludes/releases:
    claim.product.quality.aaa-visual-parity
    claim.product.quality.photoreal-rendering-guarantee

  does not directly release:
    claim.product.feature.production-terrain
    claim.product.feature.production-foliage
    claim.product.feature.production-gi
    claim.product.feature.production-reflections

各Feature claimは対応する前提Future行だけが解放し、AAA行はそれらの成立を前提として横断品質claimだけを解放します。

また、次のtechnique名をProduct claimとして暗黙生成しません。

claim.product.feature.ray-tracing
claim.product.feature.path-tracing
claim.product.feature.neural-reconstruction

これらを将来Product claimにする場合は、別のFuture ID、claim ID、Target Activation、fallback、Qualification境界が必要です。現在のALT Owner内ではtechnique familyとして保持します。

8. Future DAGの最終形
future.capability.production-terrain ───────────────┐
future.capability.production-foliage ──────────────┤
future.capability.production-global-illumination ──┼─>
future.capability.production-reflections ──────────┘
    future.capability.aaa-photoreal-rendering

GIとReflectionsの間にedgeはありません。

production-global-illumination -X-> production-reflections
production-reflections -X-> production-global-illumination

このため、将来のconsumerは次のようにexact dependencyを選べます。

Consumer requirement	参照するFuture ID
diffuse indirectだけを必須とする	future.capability.production-global-illumination
specular reflectionだけを必須とする	future.capability.production-reflections
両方を必須とする	上記二件
AAA／photoreal横断claim	Terrain、Foliage、GI、Reflectionsの四件

これにより、GIの方式変更、Reflectionの方式変更、一方だけのTarget追加、一方だけのfallback変更、一方だけの降格が、他方のCapability identityやProduct claimを変更しません。

9. 設計判断文書の正確な訂正点

承認済み設計判断の§7と§13は、次に訂正する必要があります。

§7冒頭

旧：

二つの複合Entryを分解し、26行を30行にする。

訂正：

二つの複合Entryを七つの原子的Entryへclean-breakし、
Future Portfolioを26行から31行にする。
すべてplanning_onlyを維持する。
§7.1 Rendering

旧：

future.capability.production-global-illumination-reflections

訂正：

future.capability.production-global-illumination
future.capability.production-reflections
§13完了条件

旧：

二つの複合Future IDが六つの原子的IDへclean-breakされ、
全30行がplanning_onlyである。

訂正：

二つの複合Future IDが七つの原子的IDへclean-breakされ、
全31行がplanning_onlyである。
10. 最終セルフチェック
項目	最終値
採用案	B
Future総数	31
旧Future総数	26
廃止する旧複合行	2
追加する原子行	7
新Owner数	4
Owner総数	59
Rendering新Future行	4
Multiplayer新Future行	3
GI Owner	mirakan.arch.rendering-advanced-light-transport
Reflections Owner	mirakan.arch.rendering-advanced-light-transport
GI／Reflections間Future edge	なし
AAA Future prerequisites	Terrain、Foliage、GI、Reflectionsの4件
Reflection claim ID	claim.product.feature.production-reflections
production-global-illumination-reflections	廃止、aliasなし
Advanced Rendering／Multiplayerの現状態	planning_only／実装absentのまま
MVP・active Capability・release claimへの追加	なし
最終結論

Aは不採用です。

production-global-illumination-reflectionsは、同じOwnerに属するという理由だけで、独立Activation、Target support、fallback、Qualification、Product claimを一つに結合しています。これは旧Terrain／Foliage／GI複合行を原子化する目的と同じ種類の問題を、ALT Owner内部へ移しただけです。

唯一採用すべき構成はBです。

Future Portfolioを31行とし、
future.capability.production-global-illuminationと
future.capability.production-reflectionsを独立させる。
Ownerは増やさず、両方をmirakan.arch.rendering-advanced-light-transportが所有する。
