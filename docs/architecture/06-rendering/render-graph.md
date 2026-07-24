# Miraikanai Engine Render Graph Contract

- 文書ID: mirakan.arch.rendering-render-graph
- 状態: review
- 正本範囲: Renderer公開境界、Render Snapshot／View、resource／pass graph、queue／barrier／lifetime execution、surface composition、visibility／geometry execution、lighting pipeline profile、anti-aliasing／temporal execution、Renderer固有failure／qualification
- 非正本範囲: Project Shader Source／semantic Module／Technique Manifest意味／AI理解、Material／Lighting／Post Process／LOD／Worldのauthoring semantics、Runtime phase／shared capacity、Asset transaction、Tool／SDK version、AI authorization、Evidence envelope、共通Schema／projection。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[Product Plan](../00-product/product-plan.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Core architecture](../02-foundation/core-architecture.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Editor UI Framework](../03-authoring/editor-ui-framework.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[Animation](../05-simulation/animation.md)、[Materials](materials.md)、[Project Shader](project-shader.md)、[Lighting](lighting.md)、[Post Processing](post-processing.md)、[LOD](lod.md)、[World](world.md)
- 外部根拠検証日: 2026-07-22

## 1. 結論と所有境界

RendererはProject C++、Gameplay、Editor、AIからnative API object、command list、descriptor index、GPU address、shader binaryを隔離し、Engine-owned handleとimmutable input snapshotだけを受ける。Render Graphはpass、resource、queue、barrier、alias、temporal history、submissionを一意に計画し、Backend Adapterはその計画をnative APIへ写像する。

宣言的`RenderGraphDefinition`はresource／pass／view／dependencyだけを持ち、`render_graph` moduleがcanonical execution planへcompileする。Engine-owned Pass Templateと[Project Shader](project-shader.md)でQualification済みのTechnique ManifestだけをDefinitionへ展開し、callback外access、native barrier／queue signal、Backend objectを埋め込まない。

Runtime phase、Simulation Advance、job dependency、submission lifetimeは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、共通CPU／GPU／memory budgetと測定法は[Runtime performance／capacity](../04-runtime/performance-capacity.md)だけが決定する。本書はRenderer固有のresource pressure、fallback、correctnessを定義するが、共通値や測定envelopeを複写しない。

Materialのshading意味、Lightの物理意味、Post Processのvolume／effect composition、LOD representation選択、World source／streaming planは各同階層Ownerが決定する。Rendererは解決済み入力を実行し、他DomainのSource Documentを解釈しない。

## 2. Module境界と公開Port

ModuleはContracts、Render Extract、Graph Compiler、Resource Registry、Pipeline Cache、Visibility／Geometry、Surface Composite、Backend Adapter、Renderer Qualificationに分ける。Backend AdapterとProvider Adapterはprivateであり、Vendor型、result code、archive、extension structを公開Portへ漏らさない。

`GraphicsDevicePort`はadapter／device／surface Capability query、resource／view／sampler／pipeline作成、descriptor／argument binding、command allocator／buffer取得、logical barrier plan encode、queue submit／serial／completion、present／resize／device fault report、budget／residency telemetryだけを公開する。native型をPort header／MCD／Asset metadata／Project C++へ出さない。

公開入力は次のinventoryに限定する。以後の節と他文書はこの表のIDだけを参照する。種別は`frame`（毎frameのimmutable入力）、`resolved`（Owner Resolverの解決済みPlan）、`cook`（Cook由来artifact）である。`AntiAliasingIntentV1`等のplanned authoring action入力は§11の正本経由であり、frame入力へ混在させない。

| 公開入力 | 種別 | 正本schemaとOwner定義 |
|---|---|---|
| `RenderSnapshot` | frame | 本書§2.1。published simulation／world stateから抽出したimmutable frame input |
| `ViewFamily` | frame | 本書§2.1の`RenderView`集合。同じsurface、render extent policy、AA plan、exposure familyを共有する |
| `ResolvedMaterialBindingV1` | frame | [Materials](materials.md) §5。`CookedMaterialArtifact`とtyped instance bindingの解決結果 |
| `LightSnapshotV1` | frame | [Lighting](lighting.md) §6。`ResolvedLightSet`の唯一のRenderer公開形で、`RenderSnapshot.light_snapshot`として受ける |
| `ResolvedPostProcessPlanV1` | resolved | [Post Processing](post-processing.md)が解決したordered effect composition |
| `ResolvedRepresentationSet` | frame | 本書のframe入力名。[LOD](lod.md)所有の`LodResolutionPlanV1`／`ViewLodContextV1`に基づくruntime選択結果（representationとtransition state）をViewFamilyごとに整列する |
| `WorldRenderPacket` | frame | [World](world.md)のactive cell revisionから生成されたrenderable集合 |
| `ResolvedAntiAliasingPlanV1` | resolved | 本書§9。[Executable contracts](../02-foundation/executable-contracts.md)正本の`AntiAliasingIntentV1`から解決する |
| `ResolvedShadowPlanV1` | resolved | 本書§12。Shadow authoringの解決結果 |
| `RenderRepresentationPlanV1` | cook | 本書§12。Runtime planからCookした分類plan |
| `Renderer2DExecutionPlanV1` | cook | 本書§12。2D packet抽出plan |

SnapshotはSource Stable ID、artifact generation、Transform、bounds、visibility mask、previous presentation stateをEngine handleで参照する。Rendererはauthoritative World／Physics stateを書き戻さない。Animation skin／poseは[Animation](../05-simulation/animation.md)が公開したgeneration付きsnapshotとして読み、GPU skin executionだけを所有する。

Renderableの`RenderObjectKey`は`{renderable_type_id, pipeline_key, material_key, geometry_key, stable_render_id}`のcanonical tupleである。Transparent sortはStyleのdepth／priorityとStable IDを使い、pointer／submission orderをtie-breakにしない。

### 2.1 `RenderSnapshot`と`RenderView`

```text
RenderSnapshot
  schema_version
  snapshot_id
  simulation_cadence_profile_ref: SimulationCadenceProfileRefV1
  simulation_advance_interval_hash: SHA-256
  advance_sequence
  project_revision
  world_generation
  asset_generation_id
  view_family[]
  renderable_2d[]
  renderable_3d[]
  light_snapshot: LightSnapshotV1
  environment
  water_batch[]
  weather_presentation
  snow_surface_batch[]
  vfx_batch[]
  ui_snapshot
  debug_batch
```

Snapshotは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)のpublish contractで全体を一度だけpublish後immutableとする。先頭のCadence Profile ref、Interval hash、advance sequenceは同じpublish対象`SimulationAdvanceIntervalV1`とbyte equalityにし、Presentation側でrateまたはdurationを補完しない。Entity pointer、Component span、native Physics／GPU objectを含めず、ArrayはStable rendering keyでcanonical sortしworker completion順を保存しない。

| `RenderView` field | 型／規則 |
|---|---|
| `view_id` | frame内一意`uint32` |
| `purpose` | `game \| editor \| shadow \| reflection \| thumbnail` |
| `projection` | perspective／orthographicとfinite parameter |
| `view_transform` | right-handed、meter、finite |
| `viewport_px`／`scissor_px` | surface範囲内 |
| `render_scale` | Target Profile範囲 |
| `layer_mask` | registered 64-bit layer |
| `visibility_origin` | camera-relative origin |
| `history_key` | temporal history Stable key |
| `quality_profile_id` | Cooked Profile |

View capabilityのactivationと導入順は[Product Plan](../00-product/product-plan.md)を参照し、本書はViewごとに独立selection／history stateとdomain qualification evidenceだけを提供する。

## 3. Resource modelとRender Graph

`RenderResourceDesc`を次で固定する。`kind`と`ownership`は独立したclosed enumであり、surface／historyをresource kindへ混在させない。

| Field | 規則 |
|---|---|
| `resource_id` | Graph instance内`uint32` |
| `kind` | `texture \| buffer \| acceleration_structure` |
| `ownership` | `transient \| imported \| persistent \| history \| external_surface` |
| `format` | Engine `PixelFormat` closed enum |
| `extent` | absolute、view-relative、named resource-relative |
| `mip_count`／`array_layers`／`sample_count` | Target Capability内 |
| `usage_flags` | attachment、sampled、storage、copy、indirect等 |
| `memory_class` | device local、upload、readback |
| `initial_access` | imported／persistentだけ必須 |
| `clear_value` | attachment clear時だけ、typed |
| `alias_group` | compilerが生成。Authoring入力不可 |
| `debug_name_id` | shipping identityに使わない |

Texture extent、mip、format、usageの組合せをTargetごとに検証し、0 size、overflow、unknown format、usage不整合、readback attachmentを拒否する。Imported resourceはowner、generation、acquire／release conditionを持ち、Graph外resourceを暗黙captureしない。

上位層が保持できるGPU object identityは`RenderResourceHandle { index32, generation32 }`、`PipelineHandle`、`MaterialHandle`だけである。D3D12 resource、`VkImage`、`MTLTexture`等はBackend registryが単一所有し、すべてのhandle lookupでhandle generation、Backend device generation、Asset versionの3つを検証する。一つでも不一致ならstale handle failureとし、native objectを返さない。

resource／pipeline／materialの再利用または破棄は、最後に使用したgraphics、async compute、copyの全queueそれぞれの`GpuSubmissionSerial`がcompletionを通過した後だけ許す。frame count、一queueのserial、CPU submission完了のいずれも代用できない。Device recreate時はBackend device generationを増加し、registry内の`RenderResourceHandle`、`PipelineHandle`、`MaterialHandle`とpending leaseをbulk invalidateする。旧device generationのhandleは全queue完了後のretire処理に限って参照でき、lookup／reuse／new submissionに使えない。

Pass declarationはStable ID、queue class、read／write／read-write access、subresource range、attachment、pipeline key、side-effect class、optional capability requirementを持つ。Pass callbackが宣言外resourceへ触れること、native barrierを発行すること、別queueへworkを隠すことを禁止する。

`RenderPassDescriptor`は`pass_id`、`pass_type`、`queue_class`、`view_scope`、`accesses[]`、`color_attachments[]`、`depth_stencil_attachment`、`execute_template_id`、`parameter_block`、`declared_cost`を持つ。Runtimeのpass type／execute templateはCooked Capability Catalogのclosed IDである。Project／AIはRuntime GPU callback、shader binary、native barrierを追加できないが、offline Source ChangeSetとして`ProjectShaderTechniqueV1`を提案し、Qualification後にCookerがgenerated execute templateとShader artifactへ変換できる。`ResourceAccess`はresource／subresource、`read | write | read_write`、logical stage／usageを明示し、descriptor外accessはvalidation faultである。

Graph Compilerは少なくとも次を検証する。

- use-before-produce、write conflict、cycle、未宣言access、incompatible format／sample、invalid present path。
- resource lifetime、transient alias overlap、temporal history generation、surface generation。
- queue ownership transfer、wait／signal dependency、barrier completeness、pass culling legality。
- Pipeline interfaceとresource binding reflectionの一致。

compile algorithm `R30_CompileRenderGraph`は次の固定順である。

1. resource、pass、view、Target Capabilityをschema validationする。
2. writer一意性、read-before-write、import initial state、feedback loopを検証する。
3. explicit dependencyとresource hazardからpass DAGを構築する。
4. `pass_id` tie-breakでcanonical topological sortする。
5. resourceのfirst／last useとqueue ownershipを計算する。
6. descriptor完全一致かつlifetime非重複のtransient resourceだけalias候補にする。
7. cross-queue signal／waitとBackend-neutral barrier planを作る。
8. compatible attachment passのmerge候補をBackendへ提示する。
9. transient heap、descriptor、command-list capacityを予約する。
10. compile keyとGraph diagnosticsを出力する。

cycle、uninitialized read、同一subresource unordered write、unsupported format、capacity超過はGraph全体を拒否し、resourceを別formatへ黙って変換しない。

同じGraph Definition、Target Profile、Capability Signature、Quality inputからは同じcanonical pass order、resource plan、pipeline key集合を生成する。worker completion順、pointer値、hash-map iteration順を計画へ使わない。

`CanonicalRenderExecutionPlanV1`（compile済みcanonical pass order、resource plan、queue／barrier plan）は本書がOwnerであるlogical planであり、D3D12等の各Backendはそのconsumerとしてnative mappingだけを所有する。Backendがlogical planへpass、resource、依存edgeを追加・並べ替えすることを禁止する。

## 4. Queue、barrier、lifetime execution

queue classはgraphics、async compute、copyをEngine語彙として公開し、実Deviceで利用不能なclassはGraph compile時に意味を保ったqueueへ統合するか、明示的にunsupportedを返す。Queue間dependencyはGraph edgeからだけ生成し、Backend固有signal値を保存形式やdiagnostic identityにしない。

Transient resourceはcompile済みintervalの範囲だけ生存し、aliasはformat／alignment／queue overlap／clear semanticsが互換な場合に限る。Persistent resource、streaming resource、temporal history、swapchain surfaceはgeneration付きleaseで参照し、Device reset、resize、provider change、artifact promotionを跨ぐstale handleを拒否する。

Graph planは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)が公開するRenderer execution slotで実行する。本書は共通phase表、writer順、Simulation Cadenceを再掲せず、slot inputがimmutableであることと、leaseが記録した最後の全queue `GpuSubmissionSerial`完了後にだけ解放することを要求する。

## 5. Frame lifecycle、surface、recovery

Frame contextはinput snapshot revision、ViewFamily revision、Graph plan generation、surface generation、submission serialを束ねる。resize、display-mode change、surface lossはnew generationを発行し、旧generationのworkをretireしてから公開する。旧／新surface resourceの一frame混在を禁止する。

Device loss時は新規submissionを止め、diagnosticをfreezeし、in-flight leaseをfaultedとして回収し、BackendとProviderを再生成する。Source AssetやWorld stateをGPU resourceから復元せず、[Asset lifecycle](../03-authoring/asset-lifecycle.md)のCooked Artifactとpublished snapshotから再構築する。復旧不能ならGameHostを無限retryさせずRenderer faultを公開する。

## 6. 2D／3D／UI composition

World 2D、World 3D、post-processed scene、pixel-locked overlay、Editor UIは別layerとしてGraphへ登録する。depth、color space、alpha convention、sample count、output transferはlayer contractで固定し、UIをtemporal reconstructionやscene exposureへ暗黙入力しない。

Post Processのeffect順、volume blend、history reset要求は[Post Processing](post-processing.md)を正本とする。RendererはそのPlanをCatalog登録済みPass Templateへ展開する。UI／Editor packetのprimitive意味とaccessibilityはそれぞれのOwnerが決定し、本書はsurface compositeとsubmission lifetimeだけを所有する。

`LayerCompositionSummaryV1`は本書がOwnerとして公開するread-only／revisioned projectionであり、最低fieldとしてrevision、view_family_id、layer entry配列（layer ID、depth／color space／alpha convention、sample count、pixel-locked有無、AA除外区分）を持つ。layer IDの語彙は本節のlayer contract登録に従う。[Post Processing](post-processing.md) §7のresolver入力を含む消費側はfield一覧を複写せず書き戻さない。

## 7. Visibilityとgeometry execution

Visibility executionはViewのfrustum、layer mask、World packet bounds、[LOD](lod.md)のResolved Representationを入力とし、candidate生成、occlusion、instance compaction、indirect command generationを行う。representation選択やprojected-error policyをRendererで再計算しない。

GPU-driven pathとCPU reference pathは同じvisible item identity、material binding、geometry generationを生成しなければならない。HZBやocclusion historyはView／surface／projection generationへ束縛し、camera cut、teleport、extent change、history欠損ではconservative visibleへfallbackする。Work expansion機能を使ってもresource lifetime、queue、barrier、budget ownershipはRender Graphから移さない。

実行path IDは`renderer-profile.cpu-direct`、`renderer-profile.gpu-indirect`、`renderer-profile.gpu-meshlet`、`renderer-profile.gpu-work-graph`で、後3者はそれぞれ`renderer-profile.cpu-direct`または`renderer-profile.gpu-indirect`へfallbackできる。HLODのstatic eligibility roleは`decorative_instance`に限り、Gameplay identity／interactionを変更しない。

## 8. Material、Lighting、Post Processとの実行境界

[Materials](materials.md)がMaterial Domain、Shading Model意味、visual style、Material variant keyを所有し、[Project Shader](project-shader.md)がProject Module／Techniqueのsemantic interface、Source profile、Fact Graph、Understanding Closureを所有する。Rendererはoffline生成済みshader artifact、Technique Manifest、Pipeline Interfaceを検証し、runtime source compileや未登録fallback shaderを行わない。

MCD生成`ShaderInterface`はbinding、constant layout、vertex input、attachment expectationを持ち、Pipeline keyはShader Package、render state、vertex layout、attachment format、sample count、Target featureのcanonical hashとする。

[Lighting](lighting.md)はlight type、photometric quantity、color／temperature、shape、range、shadow intentを所有する。RendererはTarget Capabilityに応じたselection、cluster／tile assignment、shadow pass、lighting passをGraph executionとして所有するが、lightの物理値を別単位へ黙って補正しない。

lighting pipeline profileは本書所有のclosed IDであり、`forward_plus_v1 | hybrid_deferred_v1`に固定する。`forward_plus_v1`はGBufferを持たないclustered forward executionで、§9のMSAA Qualified attachment／pipelineと[Materials](materials.md) §4.2のDecal C1 receiverはこのprofileを前提とする。`hybrid_deferred_v1`は`forward_plus_v1`を基底に一部receiverをGBuffer passで実行するoptional profileで、GBuffer attachmentへMSAAを適用しない（§9）。profileは`ResolvedRendererProfileV1`（§12）がTarget Profile、Quality、Capability Signatureから一意に解決し、Target別の既定と品質Tier対応は各Target Profile文書が本IDを参照して固定する。profileをframe途中で切り替えず、変更はSettings Apply／Loading境界のGraph再構築とする。

[Post Processing](post-processing.md)はvolume resolve、effect order、parameter compositionを所有する。RendererはPlanのEffect Catalog IDまたはQualification済みProject Shader Technique IDからresource requirement、history lease、AA接続、surface compositeを検証し、Plan本文からraw pass、Shader Source、native resource、未承認順序変更を受けない。

## 9. Anti-aliasingとtemporal execution

`RendererProfileResolver`は`AntiAliasingIntentV1`をViewFamilyごとの`ResolvedAntiAliasingPlanV1`へ解決する。同じViewFamilyのCameraはraster sample count、temporal Provider、jitter、render／display extentを共有し、異なる方式は別ViewFamily／render targetとする。

| Field | 規則 |
|---|---|
| `source_intent_id`／`source_revision` | 解決元Intent、stale Plan拒否 |
| `view_family_id`／`scope_resolution` | Project／Camera Profileから最終scopeへ解決した結果 |
| `raster_samples` | `1 \| 2 \| 4 \| 8`。2以上はForward+のQualified attachment／pipelineだけ |
| `spatial_method` | `off \| fxaa \| smaa_1x`。一つだけ |
| `temporal_method` | `off`またはexact `upscaler_profile_ref`。後者は`upscaler-profile.mirakan-taa \| upscaler-profile.mirakan-taau`またはQualification済みCatalogのexact Profile IDの一つだけ |
| `render_extent`／`display_extent` | Target／Provider制限内。UI／pixel-locked extentを含めない |
| `jitter_policy` | Engine-owned closed ID。AI／Project dataはsample列を保持しない |
| `history_reset_mask` | camera cut、teleport、extent、surface、projection、Provider、model、AA方式変更のclosed bit |
| `excluded_layers` | `ui \| text \| pixel_locked`を必須集合とする |
| `required_capabilities` | Backend、renderer、sample count、motion／depth／mask、Provider、HDR Capability ID |
| `predicted_cost` | pass GPU、bandwidth、persistent／transient byte、Pipeline variant数 |
| `fallback_chain` | 順序、意味差、User通知、必要な再構築境界 |
| `decision_trace` | 選択理由、却下method、Constraint／Receipt／baseline ID |

方式互換を次のclosed tableに固定する。

| Method | Qualification class | Renderer／入力 | 主用途と制限 |
|---|---|---|---|
| `none` | diagnostic | 全Raster | bit-exact検査、AA対象外layer、User明示だけ |
| `fxaa` | portable | 全Raster、resolved color | spatial fallback。Tone map後、UI合成前 |
| `upscaler-profile.mirakan-taa` | portable 3D | motion、depth、exposure、jitter、history | native extent temporal AA。Pixel／UI／VR low-latencyでは候補外 |
| `msaa_2x`／`msaa_4x` | portable Forward+ | sample可能color／depth、全PSO sample一致 | Geometry edgeとalpha-to-coverage |
| `smaa_1x` | optional | resolved color | Temporal禁止時のspatial候補 |
| `msaa_8x` | optional | Forward+、High／offline実機Gate | 自動選択禁止 |
| `upscaler-profile.mirakan-taau` | optional | `TemporalFrameInputV1` | Engine基準temporal upscale |
| Qualified DLSS／XeSS／FSR／MetalFX | optional | Provider別Input／署名／license／driver | Providerごとに独立Qualification |

`raster_samples > 1`と`temporal_method != off`を同時使用しない。MSAAとFXAA／SMAAの併用は既定禁止、Hybrid Deferred GBufferのMSAAは禁止、MSAA 8xはLow／Medium／MobileのAuto候補外とする。Alpha-to-coverageをtransparent／texture／specular alias対策と表示しない。Temporalはpre-tonemap scene-linear HDR、FXAA／SMAAはTone map後かつUI composite前に実行する。Dynamic resolution、camera cut、teleport、projection／surface／方式／Provider変更ではhistoryを破棄し、MSAA sample変更はSettings Apply／Loading境界でattachment／Pipeline keyを再構築する。

Resolverは`UnsupportedByRenderer | UnsupportedByTarget | InvalidCombination | MissingMotionVectors | MissingTemporalInput | PixelLockedTemporalForbidden | MsaaSampleUnsupported | ProviderNotQualified | BudgetExceeded | ScopeConflict | RebuildBoundaryRequired`を`AntiAliasingResolutionErrorV1`のclosed codeとして返す。未知方式を近似せず、候補、拒否理由、必要Capability／Receipt、意味差をremediationに含める。

`camera_profile`のScopeはCamera単独のnative surfaceを変更せずViewFamilyのPlan候補として解決する。SR provider IDの`upscaler-profile.directsr`／`upscaler-profile.metalfx`はそれぞれ公開Target／input契約に一致する場合だけ選択する。

Temporal historyはViewFamily、algorithm／provider generation、surface generation、extent、projectionへ束縛する。camera cut、teleport、generation／extent／projection／AA方式変更、missing motion inputでは破棄する。Generated frameはauthoritative simulation／render snapshotではなくpresentation outputとして区別し、real frameのmetricへ混ぜない。

`TemporalFrameInputV1`は本書所有のclosed schemaであり、次を持つ。

```text
TemporalFrameInputV1
  schema_version
  view_family_id
  frame_id
  present_id
  render_extent／display_extent
  jitter_offset
  motion_vector_ref
  depth_ref
  exposure_ref
  history_ref
  history_reset_mask
```

`frame_id`／`present_id`はreal frameごとに単調増加し、generated frameへ新しいsimulation identityを割り当てない。`jitter_offset`は`jitter_policy`から導出したEngine-owned値、`history_ref`は`history_key`へ束縛したgeneration付きlease、`history_reset_mask`は`ResolvedAntiAliasingPlanV1`と同じclosed bitである。native handle、descriptor index、Provider内部structを含めない。

Providerはprivate Adapterとして統合し、exact version、hash、license、取得元、build optionは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)だけが固定する。未署名artifact、runtime download、runtime training、無承認更新は本書で別規則を複写せず[AI Security／Approval](../01-governance/ai-security-approval.md)とToolchain ownerへ委譲する。

## 10. Ray、path、neural capability

Ray query、ray-traced shadow／reflection／GI、path preview、neural reconstructionは同じRender Graph resource／queue contractへ従うoptional execution profileであり、別Rendererを形成しない。Capability unavailable、history invalid、provider fault時はProfileに登録されたraster／non-neural pathへ次のGraph generationから切り替える。同一frameへ未宣言passを差し込まない。

closed profile IDは`render-path-profile.rt-shadow`、`render-path-profile.rt-reflection`、`render-path-profile.rtgi`、`render-path-profile.path-trace-reference`、`render-path-profile.path-trace-preview`、`render-path-profile.path-trace-runtime`である。`render-path-profile.rtgi-medium`は`RadianceCachePortV1`の一つのqualified profileとし、上限超過時は未宣言にallocationまたはray数変更を行わず登録済みfallbackへ戻る。

Acceleration structure、model weight、scratch resourceはgeneration付きArtifact／resourceとして扱う。Project C++やAIへnative acceleration handle、arbitrary operator、network accessを公開しない。Path previewをproduction referenceと表示するactivation判断は[Product Plan](../00-product/product-plan.md)が本書のqualification evidenceを消費して決定する。

## 11. Planned action vocabulary、diagnostic、fallback

View／Renderer intent、debug capture要求、qualification run要求はStable IDでないplanned semantic action vocabularyであり、本書のcurrent MCD Operation集合は空である。AAのexact五IDだけは[Executable contracts](../02-foundation/executable-contracts.md#20-ai向けdiscoveryexecution候補のplanning-record未activation)の`planning.operation_family.rendering_aa_discovery@1`に属し、current MCD／Owner Manifest／Service allowlist／Provider／MCP／alias集合は`[]`、Capability stateは`not_activated`である。各ownerのfuture atomic activation work itemが完全なMCD／Service／Policy／Validator／Diagnostic／Receipt／publication closureを登録するまでEditor／AIへdispatchせず、action名からIDを生成しない。Activation後の共通envelope、preview projection、approval classはFoundationとGovernanceの正本を参照し、本書では再定義しない。

Renderer固有diagnosticはGraph／pass／resource／ViewFamily／surface generation、Backend-neutral error code、first failing dependency、fallback dispositionを含む。少なくともgraph invalid、resource exhausted、pipeline unavailable、history invalid、surface lost、device fault、provider unavailableを区別する。native result codeやdriver messageはprivate attachmentとして保存し、stable diagnostic codeにしない。

`RendererProviderErrorV1`は`NotInstalled | UnsupportedDevice | UnsupportedDriver | SignatureInvalid | LicenseNotApproved | VersionMismatch | MissingInput | InvalidFormat | InitializationFailed | ExecutionFailed | HistoryInvalid | SwapchainConflict | BudgetExceeded | DeviceFault`のclosed codeを持つ。AAの互換／排他／scope失敗は`AntiAliasingResolutionErrorV1`を使い、Provider障害と混同しない。Running中のProvider failureは同frameで別Providerへ差し替えずgenerationを停止し、次のLoading境界でContextを再生成する。RT／Neural failureも次frameの登録済みRaster／non-neural Graphへ切り替える。

Quality fallbackは意味を明示し、resolution、optional effect、shadow execution、temporal provider、ray／neural profileの順序付き候補から選ぶ。allocation失敗時のsilent quality reduction、draw skip、default material置換を禁止する。共通backpressureとcapacity判定は[Runtime performance／capacity](../04-runtime/performance-capacity.md)へ従う。

## 12. 関連契約の配置

Rendererは[Lighting](lighting.md)所有の`LightIntentV1`／`LightingStyleProfileV1`／`ResolvedLightPlanV1`、[Post Processing](post-processing.md)所有の`PostProcessIntentV1`／`PostProcessProfileV1`／`ResolvedPostProcessPlanV1`、[LOD](lod.md)所有の`LodIntentV1`／`LodResolutionPlanV1`／`ViewLodContextV1`を解釈し直さず実行する。UI primitiveの`MirakanUiDrawPacketV1`は[Editor UI Framework](../03-authoring/editor-ui-framework.md)、`RuntimeRepresentationPlanV1`は[Runtime performance／capacity](../04-runtime/performance-capacity.md)、共通`RemediationV1`は[Executable contracts](../02-foundation/executable-contracts.md)、Provider lockの`RendererProviderLockV1`は[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)が正本である。

`RenderRepresentationPlanV1`はRuntime planからCookされ、各Source集合を`individual | instanced | spatial_batch | presentation_batch`の一つに分類し、Source Stable ID集合、plan-local Cell、mobility／interaction、geometry／material key、Domain LOD plan、HLOD chain、bounds、resident／visible上限、Target fallback、visual-equivalence hashを持つ。`VisibilityInstanceV1`はgeometry generation、current／previous transform、bounds、量子化済みerror／threshold、previous presentation tier、material packet、layer、stable render IDだけを持つSoAで、Entity／Component pointer／Gameplay tag／Simulation tierを含めない。

2Dは`Renderer2DExecutionPlanV1`により`SpriteRendererComponentV1`または`TileChunkArtifactV1`からAsset version、bounds、layer／order／Y-sort、material instance、atlas page、mask／blend、Stable rendering IDを持つpacketを抽出する。Source rect／Tile ID配列、texture handle、native descriptor indexをSnapshotへcopyしない。

`RendererCapabilitySignatureV1`はBackend、API／shader version、GPU／driver identity、feature bit、memory budget、display mode、SDK／model generation、signed artifact hashを持つ。`RendererCapabilityProjectionV1`はそのAuthoring向けredacted projectionであり、Adapter LUID、driver文字列等のnative識別子をEngine build固定のfield maskで除外した集合だけを公開する。Authoring／AI経路はSignature本体を直接読まずこのprojectionを消費する。`ResolvedRendererProfileV1`はProject要求、Target Profile、そのSignature、Qualification Receiptから一意に解決するroot外Derived projectionで、承認済みfallback順を持つ。Receipt subjectは先に固定したCapability Signature／Target artifact closureだけであり、Resolved Profile／Plan hashを含めない。`RendererOptimizationReceiptV1`は同一input trace／Profile／driver／SDKのBefore／After、capture、visual diffを結ぶ。

Shadow authoringの`ShadowIntentV1`／`ShadowStyleProfileV1`／`ShadowGraphV1`と承認済み`ProjectShadowTechniqueV1`は解決後の`ResolvedShadowPlanV1`だけをRendererへ渡す。`ProjectShadowTechniqueV1`は`ProjectShaderTechniqueV1`のexact specializationで、`injection_port_id = shadow`、`technique_kind = raster | compute | ray | mixed`、必須出力を`shadow_attenuation_linear`とする。`ShadowTechniquePortV1`は[Project Shader](project-shader.md)の`ProjectShaderTechniquePortV1`の`port_id = shadow` entryであり、同じSchemaを使う別名の契約を作らない。このentryが入力semantic、出力、Layer、history、ordering boundaryを固定する。`ShadowGraphV1`はclosed Pass Templateへoffline compileし、native command／barrier、runtime shader compile、未宣言accessを禁止する。

```text
ShadowGraphV1
  graph_id, graph_revision, target_profile_ref
  source_shadow_intent_ref, source_shadow_style_profile_ref
  nodes[1..64] { node_id, registered_pass_template_ref, parameter_set_ref }
  edges[0..128] { source_node_id, output_semantic, target_node_id, input_semantic }
  output_semantic: shadow_attenuation_linear
  graph_hash

ResolvedShadowPlanV1
  plan_id, source_project_revision, target_profile_ref
  shadow_intent_ref, shadow_style_profile_ref, shadow_graph_ref, shadow_graph_hash
  light_binding_refs[], resource_plan_ref, budget_reservation_ref
  project_shadow_technique_ref: optional
  selected_fallback_ref: optional
  capability_signature_hash, qualification_receipt_refs[], plan_hash
```

`ShadowGraphV1`と`ResolvedShadowPlanV1`はRenderer-owned root外Derived Artifactであり、Project／AIが直接編集しない。Graph compilerはnode／edgeのStable ID順でcanonicalizeし、cycle、未登録Template、型不一致semantic、上限超過、`shadow_attenuation_linear`以外の最終出力を拒否する。`qualification_receipt_refs[]`は先に固定したCapability Signature／Technique artifactをsubjectにするdownstream evidenceだけで、Receipt subjectへ`plan_hash`を戻さない。`ResolvedShadowPlanV1`はLighting-owned Source refsと同じProject revision、Target、Capability、budget、Qualificationへ閉じ、欠落時に既定Shadow Graphを生成しない。

Render Graph compilerは[Project Shader](project-shader.md)が所有するTechnique Manifestを通常Passと同じcycle、hazard、lifetime、alias、queue、memory validationへ通す。Manifest申告とShader Fact Graph、reflection、実行時resource useが一致しないArtifactはpromotionを拒否する。Running中の不一致は汎用`ProjectShaderTechniqueValidationFailed`とDomain projectionを発行し、該当Techniqueのそれ以降のpassとsubmissionを停止する。ShadowではDomain projectionを`ShadowTechniqueValidationFailed`とする。同一frameにfallback passを挿入しない。

PlanがGovernanceで承認されたfallback referenceを持つ場合は、次frameのGraph Instanceからそのfallbackへ決定論的に切り替える。承認済みfallbackがなければRenderer faultへ遷移し、該当Planをretry／resumeしない。承認の成立、scope、署名、期限／失効は[AI Security／Approval](../01-governance/ai-security-approval.md)だけが決め、Rendererはそのexact Governance referenceの検証結果だけを消費する。

`RayTracingPortV1`はacceleration-structure build／update、ray query／dispatch、shader／function table、scratch、compaction、timestampだけを公開する。RTGIはEngine-owned `RadianceCachePortV1`を介し、native handleをAssetへ保存しない。`NeuralRenderModelV1`はmodel ID、semantic input／output、architecture version、weight format／SHA-256／provenance、quantization、required feature、scratch／persistent byte、inference cap、fallbackを持ち、runtime download／training／未署名weight／arbitrary operator／network accessを禁止する。

`RendererVisualReceiptV1`はlinear Rec.709 RGB32F比較、UI／pixel-locked bit-exact mask、3D SSIM／RMSE、NaN／Inf、ghost persistenceとframe／camera／exposure／jitter／extent／Provider／driver／SDK／model hashを保存する。`AntiAliasingVisualReceiptV1`は本書のAA reference／baseline、alias energy、edge spread、shimmer、ghost、`unaddressed_alias_class`を追加するDomain projectionである。

AA metricの算出仕様は`AntiAliasingVisualReceiptV1`のDomain projectionとして本書が正本である。比較bufferと色空間は`RendererVisualReceiptV1`のlinear Rec.709 RGB32F比較を継承する。edge maskは静止Reference（§13の4x SSAA downsample）のrelative luminanceへの3×3 Sobel gradient magnitudeから導出し、導出閾値はEngine buildが固定してfixtureごとにReceiptへ記録する。alias energyはedge mask内のReferenceとの差分二乗和、shimmer energyは静止fixtureの連続real frame間差分二乗和のedge mask内積算とする。edge spreadはedge法線方向でluminance遷移が10%から90%へ達する距離のdisplay pixel長とする。P95の母集団はfixtureごとの全edge mask pixel（shimmerは全比較frame pair）であり、§13の閾値はfixture単位で適用する。同一入力から同一metric値を得るdeterministic実装だけを合格証拠にする。

## 13. Qualification

Qualificationはportable raster referenceを必須とし、次のDomain fixtureを持つ。

- Graph cycle、read-before-write、unordered write、subresource overlap、history invalidationのunit／property test。
- 同一Graph入力からcanonical compile plan hashが一致するdeterminism test。
- D3D12（Enhanced Barriersのみ。legacy barrier laneのconformance fixtureを持たない）、Vulkan、Metalのaccess／barrier conformanceと各validation zero-error fixture。
- 2D pixel、Realistic、Toon、Pixel Dioramaのgolden image。
- AA Off／FXAA／SMAA 1x／MSAA 2x・4x・8x／Mirakan TAA／TAAUのthin geometry、foliage、alpha scissor／blend、specular、particle、emissive、skinning、急加速、camera cut、dynamic extent、HDR fixture。
- AA scope conflict、TAA＋MSAA、Deferred MSAA、unsupported sample count、missing motion／depth、pixel-locked temporal、Settings Apply外rebuildのnegative test。
- D3D12／Vulkan／MetalでMSAA color／depth sample count、Pipeline key、resolve order、alpha-to-coverage、surface loss後rebuildが一致するBackend conformance。
- resize、alt-tab、HDR／SDR切替、surface loss、device removed fault injection。
- GPU OOM、descriptor exhaustion、pipeline miss、corrupt shader、stale Asset generationとlast-use serial前に破棄されないlifetime test。
- CPU direct／GPU indirect／meshletのvisible result、occlusion history、camera cut、overflow、fallback一致。
- FOV、projection、resolution、dynamic extent別projected error、CPU／GPU tier、hysteresis、camera cut再選択。
- Cook分類、Gameplay identity保持、mutable objectのstatic batch混入拒否、HLOD Source順序に依存しないArtifact hash、interactive／Physics／Save／animation混入拒否、HLOD on／offのGameplay一致。
- Native TAA／qualified Providerのmotion、depth、exposure、reactive、UI、HDR、dynamic extent、camera cutと、FG Provider一意Swapchain ownership／停止条件。
- Provider署名／hash／missing artifact／unsupported driver／initialization／execution failure／teardown、RT Raster fallback、acceleration structure lifetime、Path convergence／deterministic seed、corrupt／unsigned neural model／non-neural fallback。
- Shadow Graph cycle／上限／unsupported node、未宣言access、interface hash不一致、fallback欠落のnegative testと、各Shadow Planの同一`shadow_attenuation_linear`接続。
- 同じTechnique Manifest／shader reflection／runtime-use trace／Governance referenceから同じ`ShadowTechniqueValidationFailed`を生成し、promotion拒否、該当Plan停止、当該frameへのfallback挿入0件、承認済みfallbackがある場合は次frame切替、ない場合はRenderer faultとなる決定論fixture。
- Project ShaderのRaster／Compute／Ray／mixed Techniqueについて、宣言済みStorage access、Pass DAG、Port入出力、Target fallbackが同じcanonical Graphへcompileされ、Manifest外Pass／Resource／side effect、stale Understanding Closureを拒否するfixture。
- UI／text／pixel-locked layerがdynamic resolution、Temporal Reconstruction、Frame Generationで劣化しないtest。
- AIが未登録Pass、native resource、unsupported Target feature、arbitrary modelを生成できないconformance。

AA visual fixtureは4x linear-resolution SSAA downsampleを静止Reference、AA Offを動的Baselineとする。UI／text／pixel-lockedはbit-exact、NaN／Inf 0 pixel、edge mask内alias energyはOff比20%以上低減、FXAA／SMAA／MSAAのedge spread P95は1.50 display pixel以下、temporal方式は2.00以下、shimmer energy P95はOff比30%以上低減、disocclusion ghostは3 real frameを超えて残らないことを要求する。対象外alias classは失敗でなく明示列挙する。

性能run、visual／replay evidence、receipt envelope、provenance gradingは[Runtime performance／capacity](../04-runtime/performance-capacity.md)と[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を使い、閾値やfieldを複写しない。Renderer固有fixtureはGraph input、expected pass/resource relation、expected output／fallbackだけを所有する。

Release候補はruntime source compile、undeclared resource access、stale generation／Shader Understanding Closure use、critical pipeline miss、device recovery leak、unqualified Provider／Project Technique activationが0件でなければならない。本書はdomain qualification evidenceを出力し、activationと導入順は[Product Plan](../00-product/product-plan.md)が決定する。
