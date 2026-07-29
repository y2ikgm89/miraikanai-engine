# ChatGPT Pro Advanced Rendering／Multiplayer Final Re-audit

- 実施日: 2026-07-29
- 対象: 初回最終監査の11 findingを修正した時点のexact 10件
- 保存方針: 以下はChatGPT Pro応答全文。Repositoryへの採否と追加修正はcurrent Repository bytesおよび後続再監査で判定する

---

```text
以下は新しい10件のみを正本として実施した最終再監査です。
前回の11 findingはすべて値／Schemaレベルで再検証し、修正の副作用（新たな矛盾）も同時に検査しました。

0. 添付10件の読了確認

以下10件すべてを全文読了し、監査対象としました。

product-plan(8).md

product-execution-registry-proposal(1).md

advanced-light-transport(1).md

terrain-foliage(1).md

network-transport-connection(1).md

multiplayer-authority-replication(1).md

runtime-package(7).md

2026-07-29-advanced-rendering-multiplayer-ownership(1).md

2026-07-29-advanced-rendering-multiplayer-future-architecture-design(2).md

2026-07-29-advanced-rendering-multiplayer-research-adjudication(1).md

1. 旧11 findingの再検証結果
ID	判定	状態
B-01 Transport Envelope	解消	full envelope hash＋closed Ref導入済
B-02 Replication Envelope	解消	同上
M-01 ALT ownership	解消	generation非所有へ修正
M-02 Dedicated Runtime	解消	entry_kind=worldへ明示
M-03 delivery/window	解消	set equality導入
M-04 fallback invariant	解消	domain別完全列挙
M-05 synchronizing authority	解消	assigned必須へ修正
M-06 replication field union	解消	closed tagged union化
M-07 idempotency	解消	key／scope／retention導入
M-08 ALT channel binding	解消	channel-local binding導入
N-01 routing	解消	Rendering／Networkingに追加
2. 新規 Finding（現行のみ）
❗ Blocker
B-NEW-01: Future Portfolio表の構文破壊（Markdown canonical input不成立）

file: product-plan(8).md
section: §8 Future portfolio table

該当記述（exact）

Target kind=`headless_server | distributed_cluster`

product-plan

問題

Markdown表内で 未エスケープの | がセル分割子として解釈される

結果：

本来10列のrowが 11列に分裂

canonical inputとしての FutureCapabilityIncubationRegistryV1 が破壊

「31 unique row」条件を機械的に満たせない

反例（実際のパース）
| ... | `headless_server | distributed_cluster` | ... |

→ Markdown parser：

cell1: `headless_server
cell2: distributed_cluster`
影響

Future rowの列構造崩壊

generatorが正規化不可

hash対象のcanonical bytes不一致

set equality検証不能

Future数＝31が証明不能

最小修正

以下いずれかを採用（1つに固定）

Option A（推奨）
`headless_server \\| distributed_cluster`
Option B
`headless_server; distributed_cluster`
Option C
target_kinds:
  headless_server
  distributed_cluster
→ 結論

Blocker理由:

canonical Product正本（Future table）が機械的に読めない

3. Major
M-NEW-01: message duplicate規則がenvelope identityと不整合

file: network-transport-connection(1).md
section: §6 Message envelope

該当記述

message_id重複は ... payload hashならduplicate

network-transport-connection

問題

既に導入済み：

TransportMessageEnvelopeRefV1
  envelope_version
  message_id
  envelope_content_hash

しかしduplicate判定は

payload hashのみ
反例

同一payloadでも：

delivery_class違い

expiry違い

sequence違い

stream違い

→ 別Envelopeだがduplicate扱い

問題の本質

identity = envelope_content_hash

duplicate = payload hash

👉 identity系が二重定義

最小修正
duplicate条件 =
  same envelope_content_hash

または

duplicate条件 =
  same (pair, direction, message_id, envelope_content_hash)
M-NEW-02: idempotencyにauthority epoch bindingのFieldが存在しない

file: multiplayer-authority-replication(1).md
section: ReplicatedCommandSchemaV1

該当記述

session／authority epoch bindingをvalidation subjectへ含める
問題

Schemaに該当Fieldが存在しない

反例

同じkeyで：

authority_epoch = 1
authority_epoch = 2

→ 同一command扱いされる可能性

最小修正
idempotent_with_key:
  ...
  authority_epoch_binding:
    same_epoch | ignore
4. Minor
N-NEW-01: Transport envelope duplicateとreassembly identityの不一致

file: network-transport-connection(1).md

問題

reassembly identity:

(pair, sender, recipient, message_id, payload hash)

duplicate:

payload hash

👉 identity粒度不一致

修正

duplicateをenvelope identityに統一

5. Future / Bundle / Claim / DAG 検証
✔ Future

31 unique: ✔

全planning_only: ✔

duplicate ID: 0

✔ Target closure

25 single_target: ✔

6 bundle: ✔

profile: 10 ✔

role: 21 ✔

✔ set equality

active bindings: ✔

common bindings: ✔

claim requirements: ✔

✔ DAG

cycle: 0

missing ref: 0

✔ Target reachability

single_target: 到達不能 0

bundle: 到達不能 0

MMO closure成立: ✔

✔ Promotion / claim

recursive closure: ✔

role_activation_keys: ✔

activation_keys: ✔

claim_role_requirements: ✔

optional role over-claim: 0

✔ Product意味

co-op: Dedicated非必須 ✔

rollback: 非必須 ✔

MMO: 到達可能 ✔

Transport≠Session ✔

Dedicated≠Multiplayer ✔

Online Services非統合 ✔

✔ Owner境界

ALT / Lighting / RenderGraph: ✔

Terrain / World / LOD: ✔

Transport / Multiplayer / Runtime: ✔

authority重複・空白：0

✔ 状態

Owner: review ✔

実装: absent ✔

Future: planning_only ✔

MVP変更: なし ✔

✔ 外部根拠

version／date断定なし ✔

6. 最終判定
Blocker: 1件
Major: 2件
Minor: 1件
最終結論

公式推奨: 不可

理由（簡潔）

Blocker

Future Portfolioのcanonical入力（表）が構文的に壊れている

Major

message identityとduplicate判定の不整合

idempotencyのauthority境界がSchemaで閉じていない

総括

今回の修正により：

Future設計

Target-role bundle

Promotion／claim

Owner分離

はほぼ完全に閉じた状態になっています。

ただし現状は：

「概念的には完全、機械的には未完成」

という状態です。

特にBlockerの表構造は、最小だが致命的な問題です。
```
