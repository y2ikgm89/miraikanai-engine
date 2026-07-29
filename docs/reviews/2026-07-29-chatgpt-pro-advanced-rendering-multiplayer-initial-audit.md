# ChatGPT Pro Advanced Rendering／Multiplayer Initial Audit

- 取得日: 2026-07-29
- UI model: Pro
- Conversation: https://chatgpt.com/c/6a696ea3-50f8-83ee-9fa1-08f3c0750af5
- 入力: 承認済みFuture Architecture Design、Architecture Index、Product Plan、関連Owner文書18件
- 取得方法: ChatGPT UIの回答コピー
- 保存方針: 以下はChatGPT Pro応答全文。Repositoryへの採否は別のadjudication文書で扱う
- 位置づけ: 外部監査入力。Architecture正本、承認Decision、実装／Qualification Evidenceではない

以下は、添付された現行Architecture文書群および「2026-07-29-advanced-rendering-multiplayer-future-architecture-design.md」を前提とした**独立監査結果（最終版）**です。要求どおり、実装・計画には一切踏み込まず、**Architectureの形式的閉包・Owner分割・Future構造の妥当性のみ**を評価します。

---

# 0. 総括（結論先出し）

**結論は一つに固定します。**

> **「今この段階で詳細なFuture Architectureを作る」は条件付きで正当。ただし現行案のままでは過剰分割かつ責務境界が不完全であり、Owner構成は再設計が必要。**

採用すべき最終構成：

* Advanced Rendering：**4 → 3 Ownerへ削減**
* Multiplayer：**2 → 3 Ownerへ再構成（責務再分配）**
* 合計：**5 Owner（固定）**
* Future ID：**原子化維持（30件）だが依存関係を再定義**

---

# 1. 「今詳細Futureを作るべきか」の判定

## 判定: **条件付きD（正当）**

### 理由（代替案比較）

| 案                        | 評価                                          |
| ------------------------ | ------------------------------------------- |
| 既存Owner追記                | ❌ 不可：Render/World/Runtime系に責務リークが発生（既に兆候あり） |
| 薄い境界のみ                   | ❌ 不可：AI可読性・Future整合性が崩壊（依存未確定）              |
| 専用Owner分割（現行案）           | ⚠️ 方向は正しいが粒度不適切                             |
| **専用Owner + 最小閉包設計（推奨）** | ✅ 正当                                        |

### 判断基準

* 現行文書はすでに：

  * Render Graph / Lighting / World / LOD が**強い責務分離**
  * Runtime Package / ECS / Scheduling が**execution authorityを明確化**

→ この状態でFutureを曖昧にすると**循環依存が不可避**

### 結論

* Future詳細化は**必要条件**
* ただし：

  * **「完全仕様」ではなく「閉包保証レベル」までに制限すべき**

---

# 2. Owner構成の監査（最重要）

## 現行案

* Advanced Light Transport
* Terrain / Foliage
* Network Transport / Connection
* Multiplayer Authority / Replication

## 判定: **major（再設計必須）**

理由：

* 責務が**物理軸と機能軸で混在**
* RenderとWorldが分離されていない
* Multiplayerが2分割では不足

---

## 推奨Owner構成（唯一解）

### Advanced Rendering（3 Owner）

#### R1: `Advanced Scene Representation`

**責務**

* Scene structure（BVH / voxel / cluster / virtual geometry integration）
* Terrain / Foliage / World streaming
* LOD / streaming interaction

**非責務**

* 光輸送
* shading
* denoise

---

#### R2: `Advanced Light Transport`

**責務**

* GI / reflections / shadows
* ray tracing / path tracing
* lighting integration（Lighting ownerとの接続点のみ）

**非責務**

* scene topology
* denoise

---

#### R3: `Reconstruction & Temporal Systems`

**責務**

* denoise / history / reprojection
* neural reconstruction
* temporal accumulation

**非責務**

* light generation
* scene構造

---

### Multiplayer（3 Owner）

#### M1: `Network Transport & Session`

**責務**

* endpoint / handshake / delivery
* congestion / reliability
* session lifecycle（create/join/leave）

---

#### M2: `Replication & Simulation Authority`

**責務**

* authority topology
* object identity
* replication / RPC
* relevancy / dormancy

---

#### M3: `Prediction & Recovery Systems`

**責務**

* prediction / reconciliation / rollback
* resync / reconnect / host migration

---

### なぜこの構成か

| 観点    | 理由                            |
| ----- | ----------------------------- |
| AI可読性 | 「状態」「伝播」「補正」の分離               |
| 設計表現力 | temporal系を独立させないと破綻           |
| 将来自由度 | neural / rollback を独立進化可能     |
| 文書整合性 | ECS / Scheduling / Replay と直交 |

---

# 3. 既存Ownerとの衝突監査

## Blocker

### 1. World / Environment / Terrain

* **衝突対象**: world.md / environment-surfaces.md
* 問題:

  * Terrain/Foliageを別Ownerにすると**World責務と二重化**
* 対応:

  * Terrainは**Scene Representationに統合**
  * Worldは**authoring semanticsのみ維持**

---

### 2. Lighting vs Advanced Light Transport

* 問題:

  * GI/shadowがLightingと競合
* 対応:

  * Lighting = **API契約**
  * Advanced = **実装戦略**
    → 完全分離

---

### 3. Render Graph

* 問題:

  * AdvancedがGraph構造に触れる可能性
* 対応:

  * Graphは**絶対Owner固定**
  * Advancedは**node providerのみ**

---

## Major

### 4. Runtime Asset Lifecycle

* denoise history bufferのownership未定義

---

### 5. Scheduling

* temporal accumulationのexecution owner不明

---

### 6. Debug / Replay

* rollbackとreplayの関係未定義

---

## Minor

* LODとvirtual geometryの境界曖昧
* Post-processingとの境界（denoise）

---

# 4. Advanced Rendering責務の完全一意化

| 項目              | Owner                    |
| --------------- | ------------------------ |
| GI              | Advanced Light Transport |
| reflections     | Advanced Light Transport |
| shadows         | Advanced Light Transport |
| ray tracing     | Advanced Light Transport |
| path tracing    | Advanced Light Transport |
| denoise         | Reconstruction           |
| history         | Reconstruction           |
| neural          | Reconstruction           |
| terrain         | Scene Representation     |
| foliage         | Scene Representation     |
| world streaming | Scene Representation     |
| fallback        | 各Owner内に閉じる（外部非公開）       |

---

# 5. Multiplayer責務の完全一意化

| 項目             | Owner       |
| -------------- | ----------- |
| endpoint       | Transport   |
| handshake      | Transport   |
| delivery       | Transport   |
| congestion     | Transport   |
| session        | Transport   |
| authority      | Replication |
| identity       | Replication |
| replication    | Replication |
| RPC            | Replication |
| relevancy      | Replication |
| prediction     | Recovery    |
| rollback       | Recovery    |
| reconnect      | Recovery    |
| host migration | Recovery    |

---

# 6. Future ID監査

## 判定: **major（依存誤りあり）**

### 誤り

#### ❌ small co-op → dedicated server必須

→ **誤り**

* listen serverで成立可能

#### ❌ rollback → large session前提

→ **誤り**

* rollbackはsmall competitiveで最重要

#### ❌ MMO → replication前提のみ

→ **誤り**

* 必須は**interest management + persistence**

---

## 修正依存（確定）

```
Small Co-op
 └ Transport

Competitive
 └ Prediction & Replication

Large Session
 └ Replication + Relevancy

MMO
 └ Replication + Persistence + Streaming
```

---

# 7. 形式仕様不足（必須補完）

## Blocker

* closed tagged union 未定義（network state）
* exact Ref 未定義（network object identity）
* cardinality 未定義（session/player/object）

## Major

* failure / fallback
* reconnect semantics
* denoise fallback

## Minor

* negative fixture不足
* target qualification未定義

---

# 8. 外部Engine比較（一次資料のみ）

## Unreal Engine

* Rendering: Lumen / Nanite 分離
* Networking: Replication Graph + Prediction分離
  → **Scene / Transport / Temporalの分離が明確**

## Unity

* Netcode for GameObjects / Entities
  → TransportとReplicationが分離

## Godot

* SceneTreeベース同期
  → 小規模向けだが責務分離は弱い

### 抽出された教訓

* **Transport / Replication / Prediction は必ず分離**
* **Scene構造とLight輸送は分離**
* **Temporal（history系）は独立させるべき**

---

# 9. 指摘一覧

## Blocker

### B1

* Owner: Terrain/Foliage
* 問題: Worldと責務衝突
* 修正: Scene Representationへ統合

---

### B2

* Owner: Multiplayer
* 問題: Prediction未分離
* 修正: Recovery Owner新設

---

## Major

### M1

* Owner: Lighting
* 問題: Advancedと重複
* 修正: API vs Implementation分離

---

### M2

* Owner: Runtime Asset
* 問題: temporal buffer未定義
* 修正: Reconstructionへ固定

---

### M3

* Owner: Future ID
* 問題:依存誤り
* 修正: 上記依存に修正

---

## Minor

* LOD境界
* Postとの関係
* fallback明文化

---

# 10. 最終推奨Architecture（唯一解）

## Owner（固定）

### Rendering

* R1 Advanced Scene Representation
* R2 Advanced Light Transport
* R3 Reconstruction & Temporal Systems

### Multiplayer

* M1 Network Transport & Session
* M2 Replication & Authority
* M3 Prediction & Recovery

---

## Future ID

* 原子化維持（30）
* 依存再定義済み

---

## 文書更新境界

### 変更必要

* lighting.md（責務分離明記）
* world.md（terrain除外）
* runtime-asset-lifecycle.md（temporal追加）
* debugging-replay.md（rollback統合）

### 変更不要

* render-graph.md
* ECS
* scheduling

---

## 完了条件（厳密）

以下が全て満たされた場合のみ完了：

* Owner間で**責務重複ゼロ**
* 全Futureが**単一Ownerに帰属**
* 依存関係が**DAGで閉じる**
* fallbackが**各Owner内で閉じる**
* Replay / Persistence / Securityと**循環なし**

---

# 最終結論

> 現行案は方向は正しいが「粒度」と「責務分離」が不完全。
> **5 Owner構成への再設計が唯一の正解。**
