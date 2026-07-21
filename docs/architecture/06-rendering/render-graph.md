# Miraikanai Engine Render Graph Contract

- 文書ID: mirakan.arch.rendering-render-graph
- 状態: review
- 正本範囲: Renderer公開境界、Render Snapshot／View、resource／pass graph、queue／barrier／lifetime execution、surface composition、visibility／geometry execution、anti-aliasing／temporal execution、Renderer固有failure／qualification
- 非正本範囲: Material／Lighting／Post Process／LOD／Worldのauthoring semantics、Runtime phase／shared capacity、Asset transaction、Tool／SDK version、AI authorization、Evidence envelope、共通Schema／projection。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core architecture](../02-foundation/core-architecture.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[Animation](../05-simulation/animation.md)、[Materials](materials.md)、[Lighting](lighting.md)、[Post Processing](post-processing.md)、[LOD](lod.md)、[World](world.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論と所有境界

RendererはProject C++、Gameplay、Editor、AIからnative API object、command list、descriptor index、GPU address、shader binaryを隔離し、Engine-owned handleとimmutable input snapshotだけを受ける。Render Graphはpass、resource、queue、barrier、alias、temporal history、submissionを一意に計画し、Backend Adapterはその計画をnative APIへ写像する。

Runtime phase、tick、job dependency、submission lifetimeは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、共通CPU／GPU／memory budgetと測定法は[Runtime performance／capacity](../04-runtime/performance-capacity.md)だけが決定する。本書はRenderer固有のresource pressure、fallback、correctnessを定義するが、共通値や測定envelopeを複写しない。

Materialのshading意味、Lightの物理意味、Post Processのvolume／effect composition、LOD representation選択、World source／streaming planは各同階層Ownerが決定する。Rendererは解決済み入力を実行し、他DomainのSource Documentを解釈しない。

## 2. Module境界と公開Port

ModuleはContracts、Render Extract、Graph Compiler、Resource Registry、Pipeline Cache、Visibility／Geometry、Surface Composite、Backend Adapter、Renderer Qualificationに分ける。Backend AdapterとProvider Adapterはprivateであり、Vendor型、result code、archive、extension structを公開Portへ漏らさない。

公開入力は次に限定する。

- `RenderSnapshot`: published simulation／world stateから抽出したimmutable frame input。
- `ViewFamily`: 同じsurface、render extent policy、AA plan、exposure familyを共有するView集合。
- `ResolvedMaterialBinding`: [Materials](materials.md)が生成したartifact／parameter binding。
- `ResolvedLightSet`: [Lighting](lighting.md)の物理light stateをRender execution向けに整列した入力。
- `ResolvedPostProcessPlan`: [Post Processing](post-processing.md)が解決したordered effect composition。
- `ResolvedRepresentationSet`: [LOD](lod.md)が選択したrepresentationとtransition state。
- `WorldRenderPacket`: [World](world.md)のactive cell revisionから生成されたrenderable集合。

SnapshotはSource Stable ID、artifact generation、Transform、bounds、visibility mask、previous presentation stateをEngine handleで参照する。Rendererはauthoritative World／Physics stateを書き戻さない。Animation skin／poseは[Animation](../05-simulation/animation.md)が公開したgeneration付きsnapshotとして読み、GPU skin executionだけを所有する。

## 3. Resource modelとRender Graph

Resourceはtexture、buffer、acceleration structure、surface、temporal historyをclosed kindで表し、format class、extent／size、usage、clear semantics、initial／final state、lifetime class、alias eligibility、debug labelを宣言する。Imported resourceはowner、generation、acquire／release conditionを持ち、Graph外のresourceを暗黙captureしない。

Pass declarationはStable ID、queue class、read／write／read-write access、subresource range、attachment、pipeline key、side-effect class、optional capability requirementを持つ。Pass callbackが宣言外resourceへ触れること、native barrierを発行すること、別queueへworkを隠すことを禁止する。

Graph Compilerは少なくとも次を検証する。

- use-before-produce、write conflict、cycle、未宣言access、incompatible format／sample、invalid present path。
- resource lifetime、transient alias overlap、temporal history generation、surface generation。
- queue ownership transfer、wait／signal dependency、barrier completeness、pass culling legality。
- Pipeline interfaceとresource binding reflectionの一致。

同じGraph Definition、Target Profile、Capability Signature、Quality inputからは同じcanonical pass order、resource plan、pipeline key集合を生成する。worker completion順、pointer値、hash-map iteration順を計画へ使わない。

## 4. Queue、barrier、lifetime execution

queue classはgraphics、async compute、copyをEngine語彙として公開し、実Deviceで利用不能なclassはGraph compile時に意味を保ったqueueへ統合するか、明示的にunsupportedを返す。Queue間dependencyはGraph edgeからだけ生成し、Backend固有signal値を保存形式やdiagnostic identityにしない。

Transient resourceはcompile済みintervalの範囲だけ生存し、aliasはformat／alignment／queue overlap／clear semanticsが互換な場合に限る。Persistent resource、streaming resource、temporal history、swapchain surfaceはgeneration付きleaseで参照し、Device reset、resize、provider change、artifact promotionを跨ぐstale handleを拒否する。

Graph planは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)が公開するRenderer execution slotで実行する。本書は共通phase表、writer順、tick frequencyを再掲せず、slot inputがimmutableであることとsubmission fence後にだけleaseを解放することを要求する。

## 5. Frame lifecycle、surface、recovery

Frame contextはinput snapshot revision、ViewFamily revision、Graph plan generation、surface generation、submission serialを束ねる。resize、display-mode change、surface lossはnew generationを発行し、旧generationのworkをretireしてから公開する。旧／新surface resourceの一frame混在を禁止する。

Device loss時は新規submissionを止め、diagnosticをfreezeし、in-flight leaseをfaultedとして回収し、BackendとProviderを再生成する。Source AssetやWorld stateをGPU resourceから復元せず、[Asset lifecycle](../03-authoring/asset-lifecycle.md)のCooked Artifactとpublished snapshotから再構築する。復旧不能ならGameHostを無限retryさせずRenderer faultを公開する。

## 6. 2D／3D／UI composition

World 2D、World 3D、post-processed scene、pixel-locked overlay、Editor UIは別layerとしてGraphへ登録する。depth、color space、alpha convention、sample count、output transferはlayer contractで固定し、UIをtemporal reconstructionやscene exposureへ暗黙入力しない。

Post Processのeffect順、volume blend、history reset要求は[Post Processing](post-processing.md)を正本とする。RendererはそのPlanをCatalog登録済みPass Templateへ展開する。UI／Editor packetのprimitive意味とaccessibilityはそれぞれのOwnerが決定し、本書はsurface compositeとsubmission lifetimeだけを所有する。

## 7. Visibilityとgeometry execution

Visibility executionはViewのfrustum、layer mask、World packet bounds、[LOD](lod.md)のResolved Representationを入力とし、candidate生成、occlusion、instance compaction、indirect command generationを行う。representation選択やprojected-error policyをRendererで再計算しない。

GPU-driven pathとCPU reference pathは同じvisible item identity、material binding、geometry generationを生成しなければならない。HZBやocclusion historyはView／surface／projection generationへ束縛し、camera cut、teleport、extent change、history欠損ではconservative visibleへfallbackする。Work expansion機能を使ってもresource lifetime、queue、barrier、budget ownershipはRender Graphから移さない。

## 8. Material、Lighting、Post Processとの実行境界

[Materials](materials.md)がshader interface、material domain、visual style、variant keyを所有する。Rendererはoffline生成済みshader artifactとPipeline Interfaceを検証し、runtime source compileや未登録fallback shaderを行わない。

[Lighting](lighting.md)はlight type、photometric quantity、color／temperature、shape、range、shadow intentを所有する。RendererはTarget Capabilityに応じたselection、cluster／tile assignment、shadow pass、lighting passをGraph executionとして所有するが、lightの物理値を別単位へ黙って補正しない。

[Post Processing](post-processing.md)はvolume resolve、effect order、parameter compositionを所有する。RendererはPlanのresource requirement、history lease、AA接続、surface compositeを検証し、任意のpass挿入や順序変更を受けない。

## 9. Anti-aliasingとtemporal execution

AA intentはViewFamilyごとの`ResolvedAntiAliasingPlan`へ解決し、一つのViewFamilyでraster sample、temporal provider、jitter sequence、render／display extent policyを共有する。異なる方式を同じsurfaceへ混在させず、必要ならViewFamilyを分離する。

Planはnative raster、multisample raster、spatial filter、temporal reconstructionをclosed familyとして表し、次を宣言する。

- required inputs: color、depth、motion vector、exposure、reactive／transparency mask。
- history keyとreset conditions、render／display extent、jitter ownership。
- incompatible layer／effect／surface conditionsと意味同等fallback。
- Provider Artifact ref、Capability requirement、qualification receipt ref。

Temporal historyはViewFamily、algorithm／provider generation、surface generation、extent、projectionへ束縛する。camera cut、teleport、generation／extent／projection／AA方式変更、missing motion inputでは破棄する。Generated frameはauthoritative simulation／render snapshotではなくpresentation outputとして区別し、real frameのmetricへ混ぜない。

Providerはprivate Adapterとして統合し、exact version、hash、license、取得元、build optionは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)だけが固定する。未署名artifact、runtime download、runtime training、無承認更新は本書で別規則を複写せず[AI Security／Approval](../01-governance/ai-security-approval.md)とToolchain ownerへ委譲する。

## 10. Ray、path、neural capability

Ray query、ray-traced shadow／reflection／GI、path preview、neural reconstructionは同じRender Graph resource／queue contractへ従うoptional execution profileであり、別Rendererを形成しない。Capability unavailable、history invalid、provider fault時はProfileに登録されたraster／non-neural pathへ次のGraph generationから切り替える。同一frameへ未宣言passを差し込まない。

Acceleration structure、model weight、scratch resourceはgeneration付きArtifact／resourceとして扱う。Project C++やAIへnative acceleration handle、arbitrary operator、network accessを公開しない。Path previewをproduction referenceと表示する条件やCapability maturityは[Product Plan](../00-product/product-plan.md)とqualification receiptに従う。

## 11. Operation、diagnostic、fallback

Editor／AIは[Executable contracts](../02-foundation/executable-contracts.md)に登録されたtyped operationを通じ、View／Renderer／AA intent、debug capture要求、qualification run要求を提案する。共通operation envelope、preview projection、approval classはFoundationとGovernanceの正本を参照し、本書では再定義しない。

Renderer固有diagnosticはGraph／pass／resource／ViewFamily／surface generation、Backend-neutral error code、first failing dependency、fallback dispositionを含む。少なくともgraph invalid、resource exhausted、pipeline unavailable、history invalid、surface lost、device fault、provider unavailableを区別する。native result codeやdriver messageはprivate attachmentとして保存し、stable diagnostic codeにしない。

Quality fallbackは意味を明示し、resolution、optional effect、shadow execution、temporal provider、ray／neural profileの順序付き候補から選ぶ。allocation失敗時のsilent quality reduction、draw skip、default material置換を禁止する。共通backpressureとcapacity判定は[Runtime performance／capacity](../04-runtime/performance-capacity.md)へ従う。

## 12. Qualification

Qualificationはportable raster referenceを必須とし、Graph validation、deterministic compile、resource lifetime、queue transfer、surface recovery、visibility equivalence、AA history reset、Provider fault、pixel-locked compositionをTarget Profileごとに検証する。

性能run、visual／replay evidence、receipt envelope、provenance gradingは[Runtime performance／capacity](../04-runtime/performance-capacity.md)と[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を使い、閾値やfieldを複写しない。Renderer固有fixtureはGraph input、expected pass/resource relation、expected output／fallbackだけを所有する。

Release候補はruntime source compile、undeclared resource access、stale generation use、critical pipeline miss、device recovery leak、unqualified provider activationが0件でなければならない。Capability maturityと導入順は[Product Plan](../00-product/product-plan.md)だけが決定する。
