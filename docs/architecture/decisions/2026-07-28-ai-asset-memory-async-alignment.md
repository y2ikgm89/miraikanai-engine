# Miraikanai Engine AI-readable Asset／Memory／Async Loading Alignment Decision

- 文書ID: mirakan.decision.ai-asset-memory-async-alignment
- 状態: review
- 正本範囲: AI可読性、外部Asset import、Memory／lifecycle、非同期loadを一つの制作・実行経路として扱う採用判断とOwner責務地図
- 非正本範囲: Product Capability成熟度、Phase／Work Package、Schema・Operation・Diagnostic、Toolchain、Runtime budget／phase、実装Task、Capability activation。各Owner文書を参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Product Plan](../00-product/product-plan.md)、[Architecture Plan Closure Review](../appendices/architecture-plan-closure-review.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Project State](../03-authoring/project-state.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)、[Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)
- 外部根拠検証日: 2026-07-28
- 文書種別: Architecture Decision／cross-owner alignment
- Decision owner document: `mirakan.arch.asset-lifecycle`
- Decision日: 2026-07-28
- Supersedes: none

## 1. Context

Miraikanai EngineはSource Asset、Import設定、Derived Artifact、Runtime Package、live generationを別identityとlifetimeで扱う。一方、AI Authoring、外部format検出、非同期load、memory回収を独立機能として設計すると、Source理解だけでformat対応を主張する、I/O完了だけでpartial stateを公開する、process memoryの解放とpersistent Artifactの削除を混同する、という責務抜けが生じる。

外部EngineもSourceとimported data、未load Asset metadata、background loadを分離している。ただしMiraikanaiのProject revision、typed ChangeSet、Capability activation、generation handle／lease、atomic publicationは各Owner Contractで閉じる必要があり、外部APIをそのままauthorityにできない。

## 2. Decision drivers

1. Source、Import Document、Derived Artifact、Package entry、Runtime generationのidentityを相互に代用しない。
2. AI、Editor、CLIが同じbounded ProjectionとProject ChangeSetを使い、raw file、native object、live pointerを直接変更しない。
3. async completion、cancel、stale result、capacity failureからpartial Worldまたはpartial generationを公開しない。
4. Asset payloadのreachabilityとprocess memoryのlease／retire条件を別Ownerのまま閉じる。
5. Productが採用していないformat、Operation、Capabilityを文書やdecoder名から推測しない。

## 3. Considered options

### 3.1 Source fileとlive objectを共通identityとして直接操作する

不採用とする。Source revision、Cooked payload、Target specialization、active generationの差を表現できず、AI／Editorの直接変更、stale pointer、Package外Source読込みを許してしまう。

### 3.2 Import、memory、async loadを独立Subsystemとして接続する

不採用とする。各Subsystem内部は整理できるが、Source／Toolchain／Targetのinvalidation、staging、atomic publication、generation retirement、Artifact reachabilityのend-to-end条件がOwner間で欠落しやすい。

### 3.3 Immutable Artifactとgeneration境界で閉じた経路にする

採用する。Asset Lifecycleを本Decision ownerとし、Source／Cookは同Owner、汎用Runtime request／generation／residencyは専用[Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md)、Package Entry stagingはRuntime Packageへ分離する。各段階のSchemaとDomain意味は各Owner文書へ残す。AI可読性はこの経路のbounded Projectionであり、別のshadow Asset modelを作らない。

## 4. Decision

採用経路は次である。

```text
Source Asset
  -> typed analysis／Import Profile／Import IR
  -> deterministic Cook／immutable Artifact
  -> Catalog／Runtime Package staging
  -> dependency・integrity・capacity validation
  -> Simulation boundaryでatomic publication
  -> generation handle／lease
  -> retire完了とArtifact reachabilityに基づく回収
```

Source Assetの認識、format scope Qualification、Cook readiness、Package eligibility、Runtime activation、memory residencyは別のstatus dimensionとして扱う。一段階の成功を後続段階の成功へ読み替えず、Project lineage／revision、Target、completeness、stale／omitted stateを保持する。

async I/O、verify、decode、dependency準備はstagingにだけ結果を置く。request時のProject revision、Catalog generation、Target、Contract set、dependency generationをaccept時に再検証し、全条件が成立した完成generationだけをScheduling Ownerのcanonical boundaryでpublishする。cancel、failure、timeout、stale result、queue pressureではSource Worldとlast-valid generationを維持する。

process memoryのreclamationはpersistent Artifactを削除するauthorityではない。Artifact sweepはProject、Catalog、Package、last-valid、Recovery、active readerからのreachabilityをAsset Ownerが判定し、runtime resourceはgeneration handle、lease、submission、reader pinのretire完了をMemory／Scheduling Ownerが判定する。両条件を一つのGC到達性へ畳み込まない。

本Decisionは`review`であり、`contract_activation_effect = none`である。current named Source format集合はexact `[]`、current AI authoring Operation集合はexact `[]`であり、文書、型名、外部Engine機能、Preview成功からformat support、Tool公開、Runtime availabilityを生成しない。

## 5. Consequences

### 5.1 Positive

- Source、Derived、Package、Runtime generationのidentityと責務が分離される。
- AI／Editorの理解と変更がProject revisionおよびOwner Contractへ収束する。
- async completion順、cancel、stale resultによるpartial publicationを防げる。
- Artifact retentionとruntime memory retirementを別々に検証できる。

### 5.2 Costs and risks

- Owner間でSource／Artifact／generation ref、invalidation、publication boundaryを照合する必要がある。
- background I/Oが完了しても、dependency、integrity、capacity、generation検証が終わるまで利用可能にならない。
- Inventory、Projection、Operationが未materializeの間、AIによるAsset説明はreview支援に限定される。
- 汎用Runtime Asset request、priority、deadline、cancel、generation、residency、evictionのtarget OwnerはRuntime Asset Lifecycleへ一意化した。ただし同文書は`review`、実装は`absent`であり、Decision適用、Definition／Port、consumer migration、Qualificationなしにcurrent Manager、SchemaまたはAPIを推測しない。

## 6. Canonical Owner documents

| Concern | Canonical Owner |
|---|---|
| Product上のformat／Target採用とCapability maturity | [Product Plan](../00-product/product-plan.md) |
| Project revision、ChangeSet、Commit／Undo | [Project State](../03-authoring/project-state.md) |
| Source identity、Import、Cook、Artifact、Catalog、reachability | [Asset Lifecycle](../03-authoring/asset-lifecycle.md) |
| generation handle、lease、process memory resource | [Memory／Pointers](../02-foundation/memory-pointers.md) |
| 汎用Runtime Asset request、dependency、priority、deadline、cancel、generation、residency、eviction、recovery | [Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md) |
| Runtime Entry／World Package staging、package dependency closure、atomic publication | [Runtime Package](../04-runtime/runtime-package.md) |
| completion acceptance、publish boundary、retire順 | [Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md) |
| queue、worker、I/O、memory、frame hitch budget | [Performance／Capacity](../04-runtime/performance-capacity.md) |

本Decisionは上記OwnerのSchema、fixed value、runtime behaviorを再定義しない。

## 7. Supersedes／Superseded by

- Supersedes: none
- Superseded by: none

## 8. Official or primary sources

- [Unreal Engine 5.8 Asset Registry](https://dev.epicgames.com/documentation/en-us/unreal-engine/asset-registry-in-unreal-engine): 未load Assetのmetadata取得とasynchronous discoveryを分離する公式資料。
- [Unity 6.5 Asset Database](https://docs.unity3d.com/6000.5/Documentation/Manual/AssetDatabase.html): original Sourceとauthoring／Runtime向けimported counterpartを分離する公式資料。
- [Godot stable Import process](https://docs.godotengine.org/en/stable/tutorials/assets_pipeline/import_process.html): Sourceと内部imported resourceを分離する公式資料。
- [Godot stable Background loading](https://docs.godotengine.org/en/stable/tutorials/io/background_loading.html): background request、status、取得を分離する公式資料。

これらは責務分離とCoverageの比較根拠であり、外部API、object model、path、default、lifetime方式をMiraikanai Contractへ移植する根拠ではない。
