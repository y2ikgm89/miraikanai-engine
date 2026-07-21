# Miraikanai Engine World／Scene／Level／Cell Contract

- 文書ID: mirakan.arch.rendering-world
- 状態: review
- 正本範囲: World／Scene／Level／Cellの用語とsource identity、source composition／partition、streaming-plan authoring、Level transition intent、reference closure、procedural source、Map要求resolution、World固有operation／diagnostic／qualification
- 非正本範囲: Runtime cell activation／phase／shared capacity、ECS／Gameplay component schema、Physics／Navigation behavior、Render／LOD execution、Asset transaction、Save／Replay envelope、Tool version、AI authorization、Evidence envelope、共通Schema／projection。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Math／Core utilities](../02-foundation/math-core.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Gameplay programming model](../03-authoring/gameplay-programming-model.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[Collision](../05-simulation/collision.md)、[Physics](../05-simulation/physics.md)、[Navigation](../05-simulation/navigation.md)、[Animation](../05-simulation/animation.md)、[Render Graph](render-graph.md)、[LOD](lod.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論と所有境界

World authoringはWorld、Scene、Level、Cellを異なるidentityと責務で定義し、Source compositionとTarget別streaming planを分離する。AI、Editor、Project C++はSource Documentとtyped operationを扱い、Runtime cell object、ECS pointer、Renderer resource、Physics／Navigation native handleを直接保存しない。

本書はsourceとstreaming-plan authoringだけを所有する。実行時のcell activation state、phase、job、lifetimeは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、capacity／backpressureは[Runtime performance／capacity](../04-runtime/performance-capacity.md)、representation selectionは[LOD](lod.md)が所有する。

Gameplay component、Physics、Collision、Navigation、Animation、Renderingは各OwnerのDomain dataをWorld entity／cellへ参照で結び、World文書が各Schemaを複写しない。Asset save、promotion、package generationは[Asset lifecycle](../03-authoring/asset-lifecycle.md)へ委譲する。

## 2. 正規用語とidentity

- `World`: Project内の空間／content universeとglobal composition root。
- `Scene`: authoring、ownership、collaborationのための再利用可能なsource shard。
- `Level`: gameplay progression、entry／exit、objective／rule scopeを表すlogical composition。
- `Cell`: streaming、residency、activation planの空間／logical partition unit。
- `Entity source`: Stable IDを持つWorld content record。Runtime entity instanceではない。
- `Layer`: visibility／authoring／variant selectionのorthogonal grouping。LevelやCellの別名ではない。
- `Map`: user request語でありcanonical object typeではない。resolverが意図をWorld／Level／Scene／navigation／presentationへ分類する。

各identityはProject Stable ID、source revision、display labelを分離する。path、filename、display name、array indexをidentityにしない。SceneをLevel、LevelをCell、CellをRuntime chunkと同一視しない。

## 3. 「Map」要求の解決規則

Map要求は少なくともworld authoring、level composition、scene shard、navigation data、presentation／UIの候補へ分類する。Resolverは対象Project revision、requested task、scope、candidate canonical types、question／assumption、affected ownersを返す。

曖昧な「マップを作る」「マップを開く」に対して新しい万能Map assetを生成しない。空間contentならWorld／Scene／Level、pathfindingなら[Navigation](../05-simulation/navigation.md)、画面表示なら将来のUI／Camera Ownerへroutingし、本書ではpresentation schemaを定義しない。

## 4. Source Document model

`WorldSourceDefinition`はWorld Stable ID、coordinate-space ref、root Scene refs、Level catalog refs、Layer catalog、global entity refs、partition intent、default streaming-plan profile refを持つ。

`SceneSourceDefinition`はScene Stable ID、ownership metadata ref、entity source refs、nested Scene instance refs、local Layer refs、external dependency refsを持つ。Scene instanceはsource Scene identityとinstance transform／overrideを分け、nested cycleを拒否する。

Entity sourceはStable ID、Transform source、parent ref、Domain component document refs、Layer／tag、authoring metadata refを持つ。Gameplay／Physics／Navigation／Rendering component fieldは各Owner文書を参照し、World共通recordへflattenしない。

Source Documentのrevision、patch、ChangeSet、Undo／Redo、dirty state、mergeは[Project state](../03-authoring/project-state.md)を正本とする。本書はWorld Domain payloadのvalidationとreference関係だけを所有する。

## 5. World topologyとLevel definition

World topologyはroot coordinate domain、bounded／unbounded intent、region／portal／layer relation、Scene／Level membership、partition constraintsを表す。具体的grid size、runtime page、Backend spatial indexをSource正本にせず、Target別planへ解決する。

`LevelDefinition`はLevel Stable ID、entry／exit intent、required／optional Scene refs、initial Layer state、persistent／ephemeral content policy、transition targets、Gameplay rule set ref、Target compatibilityを持つ。LevelがEntityやScene contentを複製せず、composition refで束ねる。

同じSceneを複数Levelで利用できるが、instance identity、override scope、persistent state ownerを明示する。cross-Level refはpersistent ownerまたはtransition payloadを介し、unloaded instance pointerを保存しない。

## 6. Spatial Partitionとstreaming-plan authoring

Partition intentはspace／logical criteria、Cell identity policy、membership rule、border／overlap rule、always-resident intent、dependency rule、LOD／HLOD relationを持つ。Source authoringは特定Targetのmemory値やqueueを固定しない。

`WorldStreamingPlan`はWorld／Level／Scene source revisions、Target／Quality ref、Cell descriptors、source membership、dependency DAG、residency class、activation prerequisite、representation／HLOD artifact refs、fallback／degradation intent、plan diagnosticを持つ。

Cell descriptorはStable ID、bounds／logical selector、member source refs、dependency refs、priority intent、activation group、persistent state policyを持つ。Cell間dependency cycle、orphan member、duplicate ownership、cross-generation ref、unbounded closureをCook時に拒否する。

共通memory、I/O、worker、queue budget、measurement、backpressureは[Runtime performance／capacity](../04-runtime/performance-capacity.md)だけが所有する。Planはcapacity classとfallback intentを参照するだけで数値を複写しない。Runtime OrchestratorだけがPlanからCell residency／activationを実行する。

## 7. Level transition intent

Level transitionはsource／target Level、trigger intent、loading presentation ref、persistent entity／state policy、required Cell set、precondition、failure／cancel policyをSourceとして定義する。実行phase、writer、async job、timeout、Save checkpointはRuntime Ownerへ委譲する。

transition中に旧／新LevelのEntity identityを再利用せず、persistent identityは明示されたownerとhandoff recordを持つ。target dependencyが不足する場合は部分Activationやdefault Levelへ黙って進まず、blocking reasonと登録済みfallbackを返す。

## 8. 参照と依存closure

全World referenceはStable ID、expected document kind、required／optional、version compatibilityを持つ。CookerはScene nesting、Entity parent、Level composition、Cell membership、Domain component asset、transition targetのclosureをcanonical orderで解決する。

required ref欠損、kind mismatch、cycle、duplicate owner、stale revisionはhard errorとする。optional ref欠損はSourceでfallbackが宣言された場合だけ許可する。Runtimeはdisplay nameやpathからrefを再解決しない。

Asset artifact、Navigation artifact、LOD／HLOD representation、Renderer material／geometryはgeneration付きrefでPlanへ入り、異なるsource／artifact generationを一つのCell activationへ混在させない。

## 9. Procedural World source

Procedural Worldはgenerator Stable ID、typed parameter、seed semantics、input asset refs、bounded output scope、determinism class、generated Source ownership、regeneration／migration policyを持つ。GeneratorはProject Source DocumentへのChangeSetを生成し、Runtime objectやnative resourceを直接生成して正本化しない。

同じgenerator version、input revisions、seed、parameterから同じStable ID assignmentとcanonical outputを生成する。random device、wall clock、worker completion順、network responseをdeterministic generatorの入力にしない。

生成結果は通常のScene／Entity／Cell validationとreviewを通り、手編集領域を無断上書きしない。外部Tool／generator versionとartifact hashは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)を参照する。

## 10. Navigation、Simulation、Renderingとの境界

WorldはNavigation build volume／modifier、Physics／Collision component、Animation component、Render／Material／Light componentへのtyped document refを保持できるが、各Domain fieldや挙動を定義しない。Domain artifactのcook／validation resultをCell dependency closureへ取り込む。

Navigation queryやWorld movement、Physics body activation、Animation sampling、Render visibilityは各OwnerとRuntime schedulingが所有する。World Cell stateをそれらのauthoritative simulation stateとして兼用しない。

[LOD](lod.md)はresident candidateからrepresentationを選び、[Render Graph](render-graph.md)はactive Cell由来の`WorldRenderPacket`を実行する。Worldはselection formula、visibility algorithm、render passを所有しない。

## 11. Authoring bundleとAI／Editor UX

World authoring bundleは対象World／Level／Scene revisions、selected scope、typed Domain document refs、streaming-plan preview ref、validation summaryを束ねる。共通bundle／projection／operation envelopeは[Executable contracts](../02-foundation/executable-contracts.md)の定義を再利用する。

World operationはcreate／update World、Scene、Level、Cell intent、compose Scene、move Entity source、edit Layer、generate partition plan、create transition、preview closure、explain Map resolution、validateをDomain actionとして登録する。Applyは[Project state](../03-authoring/project-state.md)のChangeSetを通じ、Runtime cellを直接操作しない。

Previewは対象revision、composition graph、Cell membership／dependency、Target plan、missing closure、estimated capacity class、fallback、diagnosticを示す。authorization、approval、sandboxは[AI Security／Approval](../01-governance/ai-security-approval.md)だけが決定する。

## 12. Save、Replay、Migration境界

World Source revisionとRuntime instance stateを分離する。SaveはWorld／Level／Scene／Entity Stable IDとcompatible source／artifact generationへDomain stateを投影し、Source Document、Runtime pointer、Cell handleを丸ごと保存しない。

checkpoint時刻、recording、Replay envelope、migration evidenceは[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)を参照する。本書はWorld identity mapping、persistent owner、missing／renamed／split sourceのDomain migration ruleだけを提供する。

Migrationはold／new Stable ID mapping、semantic change、default／manual resolution、orphan dispositionを明示し、display nameやpathの近似一致で自動復元しない。

## 13. Diagnostic、failure、qualification

World固有diagnosticはWorld／Scene／Level／Cell／Entity Stable ID、source revision、reference path、Target、error code、remediationを含む。少なくともmissing ref、kind mismatch、composition cycle、duplicate owner、invalid partition、Cell dependency cycle、unbounded closure、stale plan、unsupported Target、migration unresolvedを区別する。

QualificationはScene composition／nesting、Level reuse／transition、partition determinism、Cell membership／dependency、Target plan、missing asset／Domain artifact、LOD／HLOD link、procedural regeneration、ChangeSet、Save identity migrationをfixtureで検証する。

共通capacity test、Evidence envelope、Eval、provenanceは[Runtime performance／capacity](../04-runtime/performance-capacity.md)と[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を使う。万能Map asset、path identity、Runtime pointer保存、silent missing-ref repair、phase／budget／Domain schema複写が残る実装はRelease候補にしない。Capability maturityと導入順は[Product Plan](../00-product/product-plan.md)だけが所有する。
