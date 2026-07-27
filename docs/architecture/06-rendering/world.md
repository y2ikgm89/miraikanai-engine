# Miraikanai Engine World／Scene／Space／Cell Contract

- 文書ID: mirakan.arch.rendering-world
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: World／Scene／SpaceのSource identity、global composition、persistent entity、optional spatial topology、Cellのplan-local identity、partition／streaming-plan authoring、spatial transition、Loading presentation、procedural／Tilemap／Blockoutの共通意味
- 非正本範囲: 具体World schema、procedural／Tilemap／Blockout catalog、Operation、Fixture、Gameplay progression、Runtime cell phase、ECS schema、Physics／Navigation behavior、Render execution、Save／Replay envelope
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Project State](../03-authoring/project-state.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Collision](../05-simulation/collision.md)
- 関連文書: [Procedural World Catalog／Fixture Candidate](../appendices/procedural-world-catalog-fixture.md)、[Runtime Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[Physics](../05-simulation/physics.md)、[Navigation](../05-simulation/navigation.md)、[Render Graph](render-graph.md)、[LOD](lod.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-27

## 1. 結論と所有境界

Worldは空間、Scene、global composition、persistent entity、任意のspatial topologyを所有する。Gameplay goal、outcome、spawn、進行単位はconsumer-owned stateであり、World activationへ必須にしない。

World activation、Scene activation、Cell streamingはGameplay goalやResultを要求しない。Scene 0件のprocedural-only World、spatial topologyなしのUI補助World、有限進行を持たないcontinuous simulationをvalidとする。

## 2. 正規用語とidentity

| term | meaning |
|---|---|
| World | global compositionとpersistent spatial／nonspatial sourceのauthority |
| Scene | World内で再利用可能なsource composition |
| Space | coordinate／topology profile |
| Cell | 一つのpartition／streaming plan内だけで有効なplan-local単位 |
| Map | user intent。World、Scene、Tilemap、Stage等へtypedに解決する |

World／Scene identityはstable source IDとrevisionであり、display name、path、配列index、Runtime handleを使わない。Cell IDをWorld-global identityまたはpersistent entity IDとして保存しない。

## 3. 「Map」要求の解決規則

`Map`はEngine schema名に固定しない。要求を次へ解決する。

- global spatial composition → World
- reusable authored composition → Scene
- 2D tile source → Tilemap source
- finite gameplay progression → Scenario／Stage
- runtime partition unit → Cell

曖昧な場合は候補を提示して選択を求め、root Scene、Level、Stage、Worldを相互aliasにしない。

## 4. Source Document model

World Sourceはidentity、Space profile、Scene composition、persistent entity source、optional topology、partition plan refs、environment／render refs、activation policyを持つ。

Scene SourceはWorldから独立した再利用identityを持てるが、World-global stateまたはGameplay progressionを所有しない。Persistent entityはSource identityとRuntime Entity handleを分離する。

具体Field候補は[Procedural World Catalog](../appendices/procedural-world-catalog-fixture.md#4-source-document-model)を参照する。補助文書のSchemaをmaterialized current Schemaと扱わない。

## 5. Spatial topology

Topologyは2D、3D、nonspatialをclosed profileで表す。座標系、単位、axis、origin policyはMath／World Space Ownerのtyped refへ解決する。WorldがPhysics、Navigation、Renderingの内部payloadを複写しない。

Nonspatial Worldではanchor、Cell、spatial spawn、streaming fieldをcanonical omissionする。Default 3D空間を補完しない。

## 6. Spatial Partitionとstreaming-plan authoring

Partition planはWorld Source revision、Space profile、algorithm profile、Cell集合、dependency、priority、capacity refを束縛する。Cell identityはplan ID＋plan revision＋local cell IDで解決し、別planへ流用しない。

Streaming plan authoringはSourceを直接分割せずDerived plan候補を作る。Runtime phase、I/O scheduling、shared capacityはRuntime Ownerが決定する。

## 7. Spatial transition intent

Transition intentはsource context、typed destination、requested anchor、subject refs、fallback policyを持つ。World handle、Scene pointer、Cell addressをPublic Portへ出さない。

Destination activation failure時はsource authorityを維持し、half-activated World、orphan Cell、部分subject transferを残さない。

### 7.1 Loading／prefetch presentation

Loading presentationはtransition stateのUI projectionであり、activation authorityではない。Progressは測定可能なwork closureから算出し、偽の100%、無限spinner、UI closeによるactivation commitを禁止する。

## 8. 参照と依存closure

World／Scene SourceはAsset、Schema、Space、Environment、Render、Collision等のtyped refをversion／hash付きで束縛する。Path、display name、latest assetをpackage時に再解決しない。

## 9. Procedural World source

Procedural Worldはseed、generator profile、input closure、determinism class、budget、stable-ID allocation policyをSourceとして宣言する。Generated resultはDerived candidateであり、World SourceまたはProject revisionへ自動昇格しない。

Stable-ID allocation、generated Delta、Manifest、Receipt、Public Commit Closure、Operation／Policy／Fixtureの具体候補は[Procedural World Catalog](../appendices/procedural-world-catalog-fixture.md#9-procedural-world-source)へ分離する。

Ownerが維持する不変条件は次である。

1. 同じgenerator、version、seed、input closure、Target profileから同じcanonical resultを生成する。
2. Stable IDは一つのimmutable candidate内で決定論的に割り当てる。
3. Prepublication failureではpublic objectを0件にする。
4. 成功時はmapping、Delta、Manifest、Receipt、Marker、after Projectを一つのclosureへ束縛する。
5. Runtime生成結果をAuthoring Sourceへ黙示的に逆書込みしない。

## 10. Navigation、Simulation、Renderingとの境界

Worldはgeometry／topology sourceとtyped attachment pointを提供する。Collisionはcollision representation、Physicsはbody／constraint、Navigationはnav data、Renderingはvisibility／draw executionを所有する。

### 10.1 Tilemap source、cook、publication

Tilemapは2D source grid、layer、tile semantic、asset refを所有するが、Physics collider、Navigation grid、Render meshのruntime payloadを所有しない。Cookは各Subsystem projectionを同じTilemap Source revisionへ束縛する。

具体Tilemap Schema、Cook manifest、Fixture候補は[Procedural World Catalog](../appendices/procedural-world-catalog-fixture.md#101-tilemap-sourcecookpublication)を参照する。

### 10.2 Engine-native 3D Blockout

BlockoutはAuthoring geometry sourceであり、final art、Physics body、Navigation mesh、Render meshを兼用しない。Cooked projectionはSource identity、unit、material semantic、Target profileを束縛する。

## 11. Authoring bundleとAI／Editor UX

World authoring bundleは同一Project revisionのSource closureだけを含む。AI／Editorはtyped ChangeSetを提案し、live Runtime World、Editor selection、partial streaming stateをSource authorityへ直接serializeしない。

## 12. Save、Replay、Migration境界

Saveはpersistent identityとOwnerが宣言したStateだけを保存する。World Source、Cell plan、Runtime handleを混同しない。Replayはtransition intent、accepted activation、relevant source／artifact identityを記録する。

Migrationはsource／target schema、consumer inventory、stable identity mapping、fallback、Evidenceを必要とし、似た名前のWorld／Sceneへ黙示fallbackしない。

## 13. Diagnostic、failure、qualification

最低限、unknown ref、wrong revision／hash、invalid topology、plan-local Cell misuse、capacity overflow、generator nondeterminism、stable-ID conflict、partial publication、Target unsupportedをtyped Diagnosticで区別する。

Qualificationはworldless、Scene 0、2D、3D、large partition、transition failure、procedural determinism、Tilemap／Blockout projection、Save／Replay、crash recoveryを含む。具体Fixture候補は[補助Catalog](../appendices/procedural-world-catalog-fixture.md#13-diagnosticfailurequalification)へ分離する。
