# Advanced Rendering／Multiplayer Research Adjudication

- 実施日: 2026-07-29
- 対象: ChatGPT Pro監査八件、Unreal Engine／Unity／Godot公式資料、Repository正本
- 判定authority: RepositoryのOwner、Product Plan、Decision、current state
- 非正本範囲: Engine API／algorithm／Provider／package versionの採択、実装Task／順序／工数／日程、Capability Activation、Release claim

## 1. 最終採否

採用するArchitecture判断は次である。

- 新設OwnerはAdvanced Light Transport、Terrain／Foliage、Network Transport／Connection、Multiplayer Authority／Replicationの四件だけとし、Owner総数を59件にする。
- Scene／Space／Cell、LOD、Virtual Geometry、Runtime Asset、generic Post、Render Graph execution、Scheduling／ECSの既存authorityを維持する。
- GIとReflectionsは同じAdvanced Light Transport Owner内の独立semantic channelとし、別Future ID、別Target Activation、別fallback、別Qualification、別Product claimを持つ。
- 旧二複合Futureを七つの原子的Futureへclean-breakし、Future Portfolioを`26 - 2 + 7 = 31`行にする。
- TransportはConnectionとsemantic delivery、Multiplayerはgameplay session／authority／replicationを所有し、Dedicated Game Runtime TargetはRuntime Packageが所有する。
- prediction／rollbackはMultiplayerのoptional subprofileであり、large-sessionの一律必須手段にしない。
- Future Target closureは31行とset equalityにし、25行を`single_target`、persistence、small co-op、rollback、large-session、MMO、managed external Hostの六行を`target_role_bundle`にする。bundleはactive／Future prerequisiteをrole別exact Targetへ束縛してDAG末端まで再帰展開し、claimごとのrequired non-empty roleを含む全closureを一つのpromotion／claim release単位にする。
- 31行はすべて`planning_only`、Owner文書は`review`／実装`absent`のままにし、MVP、active Capability、Work Package、Target support、release claimを変更しない。

## 2. ChatGPT Pro監査のadjudication

| 監査入力 | 採用 | 棄却／訂正 |
|---|---|---|
| [Initial Audit](2026-07-29-chatgpt-pro-advanced-rendering-multiplayer-initial-audit.md) | 詳細Future Architectureを今閉じる必要、Transport／Session混同、Render Graphのsemantic責務残存、Future依存の再確認 | 「5 Owner」としながら6件列挙、Scene Representation／Reconstruction／Predictionの新Owner、LightingをAPI扱い、Terrain／Foliage Owner撤回 |
| [Corrected Audit](2026-07-29-chatgpt-pro-advanced-rendering-multiplayer-corrected-audit.md) | 四新設Owner／総59件、既存Scene／LOD／Geometry Owner維持、Transportとgameplay session分離、prediction／rollbackをsubprofile化、semantic selectionだけをALTへ移す | GI＋Reflections複合行、Future 30行、large-sessionへrollback必須、MMOからTransport／Authorityを落とす依存、multiplayer行からTransport前提を落とす案、persistenceをgame serverへ従属させる案 |
| [Atomicity Audit](2026-07-29-chatgpt-pro-advanced-rendering-multiplayer-atomicity-audit.md) | GI／Reflections分離、Future 31行、Reflection claim追加、AAAの同一Target前提をTerrain／Foliage／GI／Reflectionsのexact四件にする | `production-global-illumination-reflections`をalias、umbrella、fallbackまたはpromotion mappingとして残す案 |
| [Target Closure Audit](2026-07-29-chatgpt-pro-advanced-rendering-multiplayer-target-closure-audit.md) | global same-target規則の到達不能、`single_target`／`target_role_bundle`分離、31 Future／59 Owner維持、headlessをdedicated authority roleとDedicated Target前提へ限定する方向 | common Future前提と未束縛conditional listだけのschema、bundle前提へのrole mapping欠落、MMOにheadless authorityとdistributed authorityを同時要求する案、既存managed Host tupleを表せないrole enum／最大7 Target |
| [Final Audit](2026-07-29-chatgpt-pro-advanced-rendering-multiplayer-final-audit.md) | Transport／Replication Envelope全Field hashとclosed Ref、ALT channel-local binding、Dedicated Runtime Entry境界、delivery class集合一致、domain別fallback outcome、assigned authority、Replication／idempotency tagged union、Owner routingの全finding | 修正前bytesへの`公式推奨: 不可`は修正後の状態判定に流用しない |
| [Final Re-audit](2026-07-29-chatgpt-pro-advanced-rendering-multiplayer-final-reaudit.md) | 初回11 findingの解消確認、Future表の列崩壊、Transport duplicate／reassembly identity、Command authority epoch bindingの追加finding | payload hashだけのduplicate identity、暗黙のauthority epoch binding、修正前表構造 |
| [Final Closure Audit](2026-07-29-chatgpt-pro-advanced-rendering-multiplayer-final-closure-audit.md) | Blocker 0、Major 0、31 Future／60 claim／Target reachability／Owner authorityの閉包と`公式推奨: 可` | 唯一残った非意味論Minorの未escape pipeはRepositoryへ残さない |
| [Final Clean Confirmation](2026-07-29-chatgpt-pro-advanced-rendering-multiplayer-final-clean-confirmation.md) | 修正行の3列parse、セル内未escape pipe 0、意味維持、Blocker／Major／Minor 0、`公式推奨: 可` | なし |

Target Closure Auditの方向は採用したが、raw schemaをそのまま正本化していない。Repository版は親行とset equalityなactive／common Future bindingとclaim requirement、profile固有additional Future binding、bundle-to-bundle role mapping、全unordered role relation、JCS profile hash、optional artifact role、claimごとのrequired non-empty role、Promotion Manifestのrole付きActivation keyまで閉じる。MMOはdesktop clientと同一distributed cluster上のauthority／operations、managed Hostはexecution Hostとoptional artifact Targetとして表し、暗黙cross productを作らず、artifact Targetが空のHost executionからmanaged Source build claimを誤解放しない。

反復監査で採用した修正は、Envelope／Plan／Commandのwhole-content identity、channel-local technique binding、domain完全列挙fallback、authority epoch／deduplication domain、Target-role closure、Owner routingというArchitecture契約だけである。実装、Task、Work Package、Target supportまたはrelease claimを追加していない。

Pro出力は外部監査入力であり、自己申告の「最終」「唯一解」または引用表示を承認Evidenceとして扱わない。`公式推奨: 可`もRepositoryの承認Receiptではなく、採否はRepositoryの既存Owner、identity、Target、fallback、claim、failure authorityとの整合とRepository内のfresh verificationで決める。

## 3. 公式資料から採る構造上の教訓

### 3.1 Unreal Engine

- [Supported Features by Rendering Path](https://dev.epicgames.com/documentation/en-us/unreal-engine/supported-features-by-rendering-path-for-desktop-with-unreal-engine)、[Hardware Ray Tracing](https://dev.epicgames.com/documentation/en-us/unreal-engine/hardware-ray-tracing-in-unreal-engine)、[Lumen GI and Reflections](https://dev.epicgames.com/documentation/en-us/unreal-engine/lumen-global-illumination-and-reflections-in-unreal-engine)、[Path Tracer](https://dev.epicgames.com/documentation/en-us/unreal-engine/path-tracer-in-unreal-engine)は、rendering path、GI、reflection、ray capability、reference pathのsupportと制約が一枚の万能booleanではないことを示す。MiraikanaiではAPI名を移植せず、semantic channel、Technique family、Target support、fallback、reference comparisonを分ける。
- [Landscape Overview](https://dev.epicgames.com/documentation/unreal-engine/landscape-overview?lang=en-US)と[Foliage Mode](https://dev.epicgames.com/documentation/en-us/unreal-engine/foliage-mode-in-unreal-engine)は、TerrainとFoliageがWorld、LOD、renderingと接続しつつ別のauthoring／instance意味を持つ比較例になる。Miraikanaiでは両者を一Owner文書内の独立branchにし、World partitionまたはgeneric LODを再所有しない。
- [Networking Overview](https://dev.epicgames.com/documentation/unreal-engine/networking-overview-for-unreal-engine?lang=en-US)、[Iris](https://dev.epicgames.com/documentation/en-us/unreal-engine/introduction-to-iris-in-unreal-engine)、[Replication Graph](https://dev.epicgames.com/documentation/en-us/unreal-engine/replication-graph-in-unreal-engine)は、client／server role、replication system、large-session interest最適化を区別する比較例になる。固有class、設定、互換条件はMiraikanaiの正本へ移植しない。

### 3.2 Unity

- [High Definition Render Pipeline](https://docs.unity3d.com/ja/current/Manual/com.unity.render-pipelines.high-definition.html)と[Terrain](https://docs.unity3d.com/ja/current/Manual/script-Terrain.html)は、Target／pipeline supportとTerrain domainをRenderer一般から区別する比較例になる。
- [Netcode for GameObjects](https://docs.unity3d.com/jp/current/Manual/com.unity.netcode.gameobjects.html)、[Relay integration](https://docs.unity.com/en-us/relay/integration)、[Relay servers](https://docs.unity.com/relay/relay-servers)は、game netcode、transport／relay接続、hosting／allocationを同じauthorityとみなさない比較根拠になる。

### 3.3 Godot

- [Renderers](https://docs.godotengine.org/en/stable/tutorials/rendering/renderers.html)と[Global illumination](https://docs.godotengine.org/en/stable/tutorials/3d/global_illumination/index.html)は、rendererごとに機能差があり、複数GI方式が異なるtrade-offを持つことを示す。MiraikanaiではTargetごとのTechnique resolutionとmeaning-preserving fallbackを要求する。
- [High-level multiplayer](https://docs.godotengine.org/en/4.7/tutorials/networking/high_level_multiplayer.html)は、high-level multiplayer APIとunderlying peer／transportを区別する比較例になる。

## 4. version／日付claimの訂正

- Epic公式ページは確認時にUnreal Engine 5.8 Documentationを表示したが、本設計は特定release dateを根拠にしない。Corrected Auditの`release: 2026-06-17`は公式一次資料で独立確認していないため不採用とする。
- Unityは`current`の公式Manual／Services資料だけを根拠にする。Corrected Auditの`Unity 6.5 / 6000.5`、Netcode `2.13`、Transport `6.6`は本監査で公式にversion closureしていないため、Architecture要件、dependency候補または採択済みversionにしない。
- Godot 4.7.1の存在は[公式maintenance release](https://godotengine.org/article/maintenance-release-godot-4-7-1/)で確認した。ただしMiraikanaiのcanonical ID、SchemaまたはTarget supportはそのversionに依存させない。

## 5. Repositoryへの適用境界

外部Engineから採るのは、責務分離、Target別support、semantic channelの独立性、fallback、authority／replication／transportの区別という構造上の教訓だけである。API、object model、algorithm、default、fixed budget、package version、marketing nameをMiraikanaiのcanonical contractへ移植しない。

本adjudicationはArchitecture文書の判断根拠であり、実装、prototype、Package、Fixture、Receipt、Capability ActivationまたはRelease Evidenceを生成したことを意味しない。
