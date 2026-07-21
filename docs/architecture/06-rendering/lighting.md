# Miraikanai Engine Lighting Contract

- 文書ID: mirakan.arch.rendering-lighting
- 状態: review
- 正本範囲: Light Source／Component、light type／shape、photometric quantity／unit／color、attenuation／range、shadow intent、Lighting semantic intent／resolver、Lighting固有operation／diagnostic／qualification
- 非正本範囲: Render pass／cluster／queue／shadow execution、Material shading、Environment composition、Runtime shared capacity、Tool version、AI authorization、Evidence envelope、共通Schema／projection。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Math／Core utilities](../02-foundation/math-core.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Render Graph](render-graph.md)、[Materials](materials.md)、[World](world.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論と所有境界

LightingはLightの物理意味とauthoring intentをEngine-owned contractに固定する。人間とAIはlight type、shape、photometric quantity、color、attenuation、range、shadow intentを編集し、Renderer Backend、cluster layout、shadow atlas、pass、descriptorを指定しない。

[Render Graph](render-graph.md)は解決済みLight setのselection、cluster／tile assignment、shadow／lighting passとqueue executionを所有する。[Materials](materials.md)はsurface responseを所有する。本書はEnvironmentやCameraのsource modelを再定義せず、将来のOwnerから与えられるenvironment light／exposure contextをtyped inputとして消費する。

共通Source revision、ChangeSet、operation envelope、projection、approval、Evidenceは[Project state](../03-authoring/project-state.md)、[Executable contracts](../02-foundation/executable-contracts.md)、Governance文書を参照し、本書でfieldやruleを複写しない。

## 2. Module境界

ModuleはLighting Contracts、Light Source Authoring、Photometric Validation、Intent Resolver、Runtime Light Snapshot Bridge、Editor／AI Projection、Lighting Qualificationに分ける。Runtime BridgeはWorld sourceを直接編集せず、published component revisionからimmutable `ResolvedLightSet`を生成する。

Rendererとの公開境界にはBackend-neutral light data、stable source identity、generation、view eligibility hint、shadow intentだけを渡す。native light object、shadow resource、cluster index、GPU addressはprivate execution stateである。

## 3. 正本データモデル

`LightSourceDefinition`は次のDomain semanticsを持つ。

- Stable ID、display label、enabled state、layer／channel mask。
- type: directional、point、spot、area family、environment contribution。
- transform semanticsとshape: direction、position、radius／extent、cone、emitter geometry。
- photometric value、declared unit、color mode、linear color／temperature、tint。
- attenuation model、range policy、near behavior、volumetric participation。
- shadow intent: casts shadow、quality intent、contact／softness intent、bias intent、fallback policy。
- mobility intent、importance class、Target compatibility、authoring metadata ref。

Light typeごとに有効なshape、quantity、unitをclosed matrixで検証する。type不一致fieldを無視せず、unknown enum、非finite値、負の物理量、degenerate direction／shape、invalid cone／rangeを拒否する。

`LightComponent`はWorld entityにSource Definition refとinstance-local overrideを結び、Source Assetを複製しない。override可能fieldはSourceが宣言し、instanceからlight typeやunit semanticsを変更しない。

## 4. 物理単位と数値規約

Lighting authoringはquantityとunitを必ず組にする。luminous intensity、luminous flux、illuminance、luminance等を無名scalarへ統合せず、Light type／shapeに対応するquantityを使う。変換はshapeとemission distributionを含む明示式に基づき、互換でないquantity間を経験的係数で補正しない。

Colorはlinear scene color、chromaticity／temperature、tintの入力modeを区別する。temperatureとRGBの同時authoritative指定を許さず、resolverが一つのcanonical linear emissionへ変換する。display transferやtone mappingは[Post Processing](post-processing.md)の責務でありLight colorへ焼き込まない。

座標、角度、距離、normalization、finite validationは[Math／Core utilities](../02-foundation/math-core.md)を使う。source unitをBackend unitへ変換する処理はprivate Adapterに閉じ、round-trip時に元のquantity／unitを保持する。

## 5. Light eligibilityとRenderer境界

Lightingはview layer、channel、mobility、importance、shadow intent、Target capability requirementを宣言する。どのLightを各Viewへ投入するか、cluster overflow時の実行順、shadow resource割当は[Render Graph](render-graph.md)が所有する。

RendererはLightの物理値をexecution上限へ収めるため黙ってclampしない。capacity不足時はSource identityと拒否／fallback理由を返し、事前宣言されたshadow disable、quality reduction、view exclusionだけを適用する。共通budget、reservation、backpressure、測定値は[Runtime performance／capacity](../04-runtime/performance-capacity.md)を参照する。

Material側はResolved Light semanticsをshading inputとして消費し、Light type、unit、attenuationを再解釈しない。Light linkingはstable layer／channel semanticsを使い、Material名、Entity名、Backend indexによるbindingを禁止する。

## 6. Lighting Intent Resolver

Lighting intentはsubject、mood／readability、key／fill／rim relation、contrast、time／environment context、physical constraint、Target scopeをtyped axesで表す。曖昧な「明るく」「映画的」等は、既存Light編集、Light追加、exposure／post-process変更の候補を区別して質問する。

ResolverはProject revision、selected World／Level scope、existing Light refs、Material／Environment context refs、Target Profile、assumption／question、candidate changes、compatibility resultを束ねたDomain resolutionを返す。共通hash、revision、disposition、projection fieldは[Executable contracts](../02-foundation/executable-contracts.md)の正本を使う。

同じ入力とCatalog revisionから同じcandidate orderを返し、Entity列挙順やviewportの一時状態へ依存しない。物理constraintとartistic intentが競合する場合はsilent conversionをせず、意味差を示した代替案を返す。

## 7. AI／Editor operation

Lighting operationはcreate light、update physical property、apply lighting intent、bind Source Definition、set shadow intent、preview、explain、validateをDomain actionとして登録する。ApplyはGatewayを通じてProject ChangeSetへ変換し、Runtime componentやRenderer resourceを直接変更しない。

Previewは対象revision、World／Level scope、affected Light Stable IDs、before／after physical values、Renderer compatibility、fallback、diagnosticを表示する。Explainは入力intent、unit conversion、採用candidate、assumption、未解決questionを返す。authorization classとhuman approvalは[AI Security／Approval](../01-governance/ai-security-approval.md)だけが決める。

## 8. Diagnosticとfailure

Lighting固有diagnosticはLight Stable ID、property path、type／shape／unit、Target、error code、remediationを含む。少なくともinvalid quantity、unit mismatch、invalid color、degenerate shape、unsupported light type、shadow intent unsupported、stale source、Renderer rejectionを区別する。

missing Source、invalid physical value、unsupported capabilityをdefault light、arbitrary intensity、shadowなしへ黙って置換しない。fallbackはSourceまたはProject profileに宣言され、意味差と適用scopeを表示する。

## 9. Qualificationと完了条件

Qualificationは各Light type／shape／unitのvalidationとconversion、color mode、attenuation、instance override、intent resolution、Renderer bridge、unsupported Target、shadow fallback、stale revisionをfixtureで検証する。

visual／numeric Evidence、Eval、provenance envelopeは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を参照する。本書はLighting inputとexpected physical／semantic resultだけを所有し、共通gradeやreceipt fieldを再掲しない。

単位なしintensity、type不一致field、Backend enumのpublic露出、Renderer実行規則の複写、silent clamp／fallbackが残る実装はRelease候補にしない。Capability maturityと実装順は[Product Plan](../00-product/product-plan.md)だけが所有する。
