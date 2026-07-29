# ChatGPT Pro Advanced Rendering／Multiplayer Corrected Audit

- 取得日: 2026-07-29
- UI model: Pro
- Conversation: https://chatgpt.com/c/6a696ea3-50f8-83ee-9fa1-08f3c0750af5
- 追加入力: [Initial Audit](2026-07-29-chatgpt-pro-advanced-rendering-multiplayer-initial-audit.md)の自己矛盾一覧、Runtime Asset Lifecycle
- 取得方法: ChatGPT UIの回答コピー
- 保存方針: 以下はChatGPT Pro応答全文。Repositoryへの採否は別のadjudication文書で扱う
- 位置づけ: 外部監査入力。Architecture正本、承認Decision、実装／Qualification Evidenceではない

以下は**初回回答の完全自己監査および訂正版（唯一解）**です。
全指摘をRepository正本（添付Markdown）と突合し、矛盾・過剰提案・誤認を排除しています。

---

# 1. 初回回答の訂正表（誤り → 訂正 → 根拠）

| 誤り                           | 訂正                             | 根拠                                        |
| ---------------------------- | ------------------------------ | ----------------------------------------- |
| 「5 Owner」としつつ6列挙             | **4新設Owner（総Owner=59）**        | 新設4件と明記されている                              |
| Advanced Rendering 4→3       | **2新設（ALT / Terrain-Foliage）** | 設計判断の表                                    |
| Scene Representation Owner新設 | **撤回（既存Ownerが所有）**             | World / LOD / Virtual Geometryの正本範囲  /  / |
| Reconstruction Owner新設       | **撤回（Post / Render Graphへ分配）** | PostとRender Graphのhistory責務  /            |
| Prediction Owner分離           | **撤回（Authority内subprofile）**   | Authorityがprediction/rollbackを包含          |
| Lighting=API                 | **撤回（semantic Owner）**         | Lightingは物理意味とshadow intentを所有            |
| Render Graph変更不要             | **誤り → 部分移管必要**                | RT/path/neural semanticを持つ                |
| Runtime Assetにtemporal追加     | **誤り → 非正本**                   | Domain意味を所有しない                            |

---

# 2. 採用 / 撤回した指摘

## 採用

| 種別      | 内容                                    |
| ------- | ------------------------------------- |
| blocker | Terrain/FoliageとWorldの責務衝突            |
| blocker | Render Graphがsemantic RT/pathを保持      |
| major   | Future依存の誤り（co-op / rollback / MMO）   |
| major   | TransportとSessionの混同                  |
| major   | Temporal三重化（RenderGraph/Post/提案Owner） |

## 撤回

| 内容                         | 理由                   |
| -------------------------- | -------------------- |
| Scene Representation Owner | 既存3 Ownerで完全被覆       |
| Reconstruction Owner       | Post/RenderGraphで閉じる |
| Prediction Owner独立         | Authority内で閉包済       |

---

# 3. 唯一の推奨Owner構成（正確ID・境界）

## 新設Owner（4件固定）

### 1. `mirakan.arch.rendering-advanced-light-transport`

**正本**

* GI / reflection / advanced shadow
* technique family / fallback
* semantic denoise（light transport限定）
* scene representation requirement（要求のみ）

**非正本**

* geometry / LOD / world（既存Owner）
* execution（Render Graph）

---

### 2. `mirakan.arch.rendering-terrain-foliage`

**正本**

* Terrain domain（tile/layer/streaming）
* Foliage domain（species/scatter/instance）

**非正本**

* partition（World）
* LOD selection（LOD）

---

### 3. `mirakan.arch.network-transport-connection`

**正本**

* endpoint / handshake / delivery
* connection lifecycle

**非正本**

* gameplay session
* authority

---

### 4. `mirakan.arch.multiplayer-authority-replication`

**正本**

* authority / topology
* replication / RPC
* prediction / rollback（subprofile）
* session lifecycle（join/leave/resync）

**非正本**

* byte transport

---

## 既存Ownerとの確定境界

| Owner            | 維持責務                                  |
| ---------------- | ------------------------------------- |
| Render Graph     | pass/resource/history/execution       |
| Lighting         | Light Source / shadow intent          |
| Post Processing  | effect composition / history intent   |
| World            | Scene / Cell / partition / activation |
| LOD              | representation selection              |
| Virtual Geometry | geometry representation               |
| Runtime Asset    | lifecycleのみ                           |

---

# 4. Future ID（変更6行 + consumer依存）

## 新規6行（正確依存）

| Future ID                                  | Owner           | prerequisite capability                                                               | prerequisite Future |
| ------------------------------------------ | --------------- | ------------------------------------------------------------------------------------- | ------------------- |
| production-terrain                         | terrain-foliage | capability.world.3d                                                                   | []                  |
| production-foliage                         | terrain-foliage | capability.world.3d                                                                   | []                  |
| production-global-illumination-reflections | ALT             | capability.rendering.render-graph-core; capability.render.material.realistic_advanced | []                  |
| headless-dedicated-server-target           | runtime-package | capability.runtime.scheduling; capability.runtime.ecs-e3-cook-load                    | []                  |
| network-transport-connection               | transport       | capability.runtime.scheduling                                                         | []                  |
| multiplayer-authority-replication          | multiplayer     | capability.runtime.scheduling; capability.runtime.ecs-e3-cook-load                    | []                  |

---

## consumer依存（5件以上）

| Consumer                        | 新 prerequisite Future               |
| ------------------------------- | ----------------------------------- |
| aaa-photoreal-rendering         | terrain + foliage + GI              |
| small-cooperative-multiplayer   | transport + multiplayer             |
| rollback-competitive-networking | transport + multiplayer             |
| large-session-network-shooter   | dedicated + multiplayer + rollback  |
| mmo-distributed-world-authority | dedicated + streaming + persistence |

既存誤り：

* co-opがdedicated必須 → 修正済
* rollbackがlarge前提 → 修正済
* MMOがreplicationのみ → 修正済

---

# 5. Handoff Matrix（主要Owner）

| From            | To                 | 内容                         |
| --------------- | ------------------ | -------------------------- |
| Lighting        | ALT                | LightIntent / ShadowIntent |
| World           | Terrain/Foliage    | Cell / partition           |
| Terrain/Foliage | LOD                | representation候補           |
| LOD             | Render Graph       | resolved representation    |
| ALT             | Render Graph       | semantic transport要求       |
| Post            | Render Graph       | history reset              |
| Transport       | Multiplayer        | delivery                   |
| Multiplayer     | Scheduling/ECS     | validated command          |
| Input           | Multiplayer        | InputSnapshot              |
| Multiplayer     | Persistence/Replay | state snapshot             |

---

# 6. Render Graph / Post / ALT の再分配（厳密）

## Render Graphに残す

* RT/path/neural **execution profile**
* history resource
* `RayTracingPortV1`, `NeuralRenderModelV1`

## ALTへ移す

* `render-path-profile.*` semantic ID
* GI/reflection/shadow/path profile selection
* denoise semantic（light transport限定）

## Postに残す

* generic effect / history intent

➡ **三重化は発生しない**

---

# 7. 未定義（Blockerレベル）

## Advanced Rendering

* technique union未定義
* fallback ladder型未定義
* representation requirement型未定義

## Multiplayer

* TransportConnectionState未定義
* MultiplayerSessionState未定義
* NetworkObjectIdentity未定義
* rollback window型未定義

## 共通

* exact Ref不足
* cardinality未定義
* negative fixture不足

---

# 8. 公式一次資料（2026-07-29時点）

## Unreal Engine（5.8, release: 2026-06-17）

* [https://dev.epicgames.com/documentation/en-us/unreal-engine/supported-features-by-rendering-path-for-desktop-with-unreal-engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/supported-features-by-rendering-path-for-desktop-with-unreal-engine)
* [https://dev.epicgames.com/documentation/en-us/unreal-engine/lumen-global-illumination-and-reflections-in-unreal-engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/lumen-global-illumination-and-reflections-in-unreal-engine)
* [https://dev.epicgames.com/documentation/en-us/unreal-engine/path-tracer-in-unreal-engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/path-tracer-in-unreal-engine)
* [https://dev.epicgames.com/documentation/unreal-engine/networking-overview-for-unreal-engine?lang=en-US](https://dev.epicgames.com/documentation/unreal-engine/networking-overview-for-unreal-engine?lang=en-US)

教訓:

* rendering pathごとに機能差（support matrix）
* GI/RT/pathが分離
* server-authority + replication分離

---

## Unity（6.5 / 6000.5, build: 2026-07-29）

* [https://docs.unity3d.com/6000.5/Documentation/Manual/render-pipelines.html](https://docs.unity3d.com/6000.5/Documentation/Manual/render-pipelines.html)
* [https://docs.unity3d.com/6000.5/Documentation/Manual/multiplayer-overview.html](https://docs.unity3d.com/6000.5/Documentation/Manual/multiplayer-overview.html)
* [https://docs.unity3d.com/Packages/com.unity.netcode.gameobjects@2.13/manual/index.html](https://docs.unity3d.com/Packages/com.unity.netcode.gameobjects@2.13/manual/index.html)
* [https://docs.unity3d.com/Packages/com.unity.transport@6.6/changelog/CHANGELOG.html](https://docs.unity3d.com/Packages/com.unity.transport@6.6/changelog/CHANGELOG.html)

教訓:

* Render Pipeline分離
* Transport / Netcode分離
* Dedicated Serverは独立概念

---

## Godot（4.7.1, 2026-07時点）

* [https://docs.godotengine.org/en/4.7/tutorials/rendering/renderers.html](https://docs.godotengine.org/en/4.7/tutorials/rendering/renderers.html)
* [https://docs.godotengine.org/en/4.7/tutorials/3d/global_illumination/index.html](https://docs.godotengine.org/en/4.7/tutorials/3d/global_illumination/index.html)
* [https://docs.godotengine.org/en/4.7/tutorials/networking/high_level_multiplayer.html](https://docs.godotengine.org/en/4.7/tutorials/networking/high_level_multiplayer.html)
* [https://docs.godotengine.org/en/4.7/tutorials/export/exporting_for_dedicated_servers.html](https://docs.godotengine.org/en/4.7/tutorials/export/exporting_for_dedicated_servers.html)

教訓:

* Rendererごとにfeature差
* GIは複数方式
* Transport/RPC/Authority分離

---

# 9. 完了条件（矛盾ゼロ）

以下すべて満たすこと：

1. Owner数 = **59（55 + 4）**
2. 新設Owner = **4件のみ**
3. Future = **30件**
4. 各Futureは**単一Owner**
5. 依存は**DAG**
6. Render/Post/ALTで**責務重複ゼロ**
7. Transport / Session / Authorityが**分離**
8. Scene/LOD/Geometryは**既存Owner維持**
9. temporal意味は**Post/ALTのみ**
10. Render Graphは**execution専用**
11. 「5と言いつつ6」の矛盾なし

---

# 最終結論

> 初回回答の構成は過剰分割と責務侵食により不適合。
> **承認案の4 Ownerを維持し、semanticのみ正確に再配置する構成が唯一の正解。**

これがRepository正本と完全整合する最終形です。
