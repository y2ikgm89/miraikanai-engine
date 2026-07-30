# Miraikanai Engine Terrain／Foliage Contract

- 文書ID: mirakan.arch.rendering-terrain-foliage
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: TerrainとFoliageの独立Source root、Terrain tile／layer／sculpt／stamp／hole／surface projection、Foliage species／variant／placement／deterministic scatter／instance identity、domain cook／artifact／invalidation、World Cell handoff、runtime domain snapshot、domain fallback／diagnostic／qualification
- 非正本範囲: World／Scene／Space／Cell／partition／activation、generic LOD selection、virtualized geometry representation、Runtime artifact request／generation／residency／lease／eviction、Material／Environment／Lighting／Render Graph、Physics body／Collision shape／Navigation mesh authority、Save／Replay envelope、shared capacity、Evidence envelope、Product quality claim、実装Task／順序。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[World](world.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)、[Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md)、[LOD](lod.md)
- 関連文書: [Product Plan](../00-product/product-plan.md)、[Advanced Rendering／Multiplayer Ownership Decision](../decisions/2026-07-29-advanced-rendering-multiplayer-ownership.md)、[Project State](../03-authoring/project-state.md)、[Editor Workspace／UX](../03-authoring/editor-workspace-ux.md)、[Runtime Package](../04-runtime/runtime-package.md)、[Persistence／Save](../04-runtime/persistence-save.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[Debugging／Observability／Replay](../04-runtime/debugging-observability-replay.md)、[Collision](../05-simulation/collision.md)、[Physics](../05-simulation/physics.md)、[Navigation](../05-simulation/navigation.md)、[Render Graph](render-graph.md)、[Advanced Light Transport](advanced-light-transport.md)、[Materials](materials.md)、[Lighting](lighting.md)、[Environment／Surfaces](environment-surfaces.md)、[Virtualized／Continuous Geometry](virtualized-continuous-geometry.md)、[Camera](camera.md)、[Windows](../07-platform/windows.md)、[Mobile Common](../07-platform/mobile-common.md)
- 根拠区分: project-decision／official-documentation comparison（未計測のtile寸法、density、instance／layer上限、distance、時間／memory値はprovisional）
- 外部根拠確認日: 2026-07-29

## 1. 結論、状態、独立Activation

TerrainとFoliageは大規模surfaceとvegetation distributionとして密接に連携するため一つのDomain Ownerへ置くが、Source、Artifact、Runtime Snapshot、Capability、Target Qualification、fallbackを別branchにする。Terrainだけ、Foliageだけ、両方を同じWorldで使用する構成を表現できなければならない。

[World](world.md)はWorld／Scene／Space／Cell、partition、activation set、world origin、generic transitionを所有する。本書はexact World scopeとCell assignmentを受け、Terrain／Foliageのdomain artifactとsnapshotを返す。[LOD](lod.md)はrepresentation tier選択、[Virtualized／Continuous Geometry](virtualized-continuous-geometry.md)はvirtual geometry representation、[Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md)はartifact generation／residency／lease、[Render Graph](render-graph.md)はvisibility／executionを所有する。

[Environment／Surfaces](environment-surfaces.md)はweather、water、snow／wetness conditionを渡し、本書はtyped receiver／response bindingだけを所有する。[Collision](../05-simulation/collision.md)、[Physics](../05-simulation/physics.md)、[Navigation](../05-simulation/navigation.md)は本書と同じSource revisionから各projectionをCookするが、Collider、Body、Nav meshを本書へ移さない。

本書の型とFuture entryは`review`／`absent`／`planning_only`である。current MVP、active World、Target support、production terrain／foliage claim、実装開始を変更しない。

## 2. 不変条件とdomain語彙

1. Terrain SourceとFoliage Sourceは別root、別revision、別artifact generation、別Activationである。一方の成功で他方を対応済みと表示しない。
2. World Cellはplacement／streaming scopeであってTerrain tileまたはFoliage cluster identityではない。一つのCellに複数domain regionを割り当てられ、domain regionは明示mappingなしにCell境界を推測しない。
3. authored instanceとgenerated instanceを別identity classにし、再scatterでauthored identity、Save identity、Gameplay stateを上書きしない。
4. Source operation、Cooked Artifact、Runtime Snapshot、Collision／Navigation／Render projectionを別Recordにする。Runtime native object、resident pointer、LOD選択、draw packetをSourceへ保存しない。
5. seed、mask、coordinate profile、neighbor、Source／dependency revision、Target Profileをartifact keyへ含め、timestamp、thread順、filesystem列挙順、GPU結果をdeterminism authorityにしない。
6. incomplete tile、stale neighbor、partial scatter generation、mixed Source revisionをpublishしない。publication単位は明示したdomain generation closureとする。
7. fallbackでGameplay surface、Collision／Navigation meaning、authored identity、World Cell membership、Save identityを変えない。
8. exact RefはID、positive version／revision、content hashを持つ。path、表示名、`latest`、座標が近いtile／speciesへfallbackしない。

```text
TerrainRepresentationFamilyV1 =
  height_field
  | mesh_surface
  | hybrid_height_mesh

TerrainEditOperationKindV1 =
  sculpt
  | smooth
  | flatten
  | stamp
  | layer_paint
  | visibility_hole
  | region_transform

FoliageInstanceOriginV1 =
  authored
  | generated

FoliagePlacementRuleKindV1 =
  explicit_points
  | density_mask
  | deterministic_scatter
  | surface_sample
  | spline_band

TerrainFoliageSupportStateV1 =
  unsupported
  | supported_unqualified
  | qualified
  | degraded
  | fault
```

`hybrid_height_mesh`は同一領域へ二重surfaceを暗黙生成するbranchではない。regionごとのrepresentation selector、seam policy、overlap拒否、authority surfaceを明示する。`generated` instanceはdeterministic origin keyを持ち、random runtime UUIDをidentityにしない。

## 3. Terrain Source

```text
TerrainDocumentV1
  terrain_id: StableId
  source_revision: positive u64
  coordinate_profile_ref: exact WorldSpaceProfileRefV1
  world_scope_ref: exact WorldScopeRefV1
  region_refs[1..4096]: TerrainRegionRefV1
  layer_catalog_ref: exact TerrainLayerCatalogRefV1
  operation_log_ref: exact TerrainOperationLogRefV1
  target_policy_ref: exact TerrainTargetPolicyRefV1
  document_content_hash: SHA-256

TerrainRegionV1
  region_id: StableId
  region_version: positive u32
  bounds: finite WorldBoundsV1
  representation_family: TerrainRepresentationFamilyV1
  tile_scheme_ref: exact TerrainTileSchemeRefV1
  source_payload:
    { kind: height_field,
      source_ref: exact HeightFieldSourceRefV1 }
    | { kind: mesh_surface,
        source_ref: exact TerrainMeshSourceRefV1 }
    | { kind: hybrid_height_mesh,
        source_ref: exact HybridTerrainSourceRefV1 }
  layer_stack_ref: exact TerrainLayerStackRefV1
  hole_mask_ref: optional exact TerrainMaskRefV1
  world_cell_binding_ref: exact WorldCellBindingRefV1
  region_content_hash: SHA-256
```

`representation_family`と`source_payload.kind`をexact一致させ、union外Fieldを禁止する。height fieldをmesh pathから、meshをheight dataから推測しない。bounds、sample／vertex count、layer count、tile countはTarget policy以下のbounded positive値とし、non-finite coordinate、overflow、degenerate bounds、重複region authorityを拒否する。

```text
TerrainLayerCatalogV1
  catalog_id/version/content_hash
  layers[1..256]:
    layer_id: StableId
    material_surface_ref: exact MaterialSurfaceRefV1
    physical_surface_role_ref: exact PhysicalSurfaceRoleRefV1
    blend_semantics:
      normalized_weight | ordered_overlay | binary_mask
    receiver_roles[0..16]:
      water | snow | wetness | foliage | collision | navigation
```

Material parameter、shader、Physics friction、Navigation cost、snow accumulationをTerrain Layerへ複写しない。Layerは各Ownerのexact semantic refだけを束縛する。unknown Layer、weight NaN／Inf、normalized totalのpolicy違反、同じLayer IDのduplicateをrejectする。

```text
TerrainOperationV1
  operation_id: StableId
  operation_sequence: positive u64
  base_terrain_revision: positive u64
  region_ref: exact TerrainRegionRefV1
  operation:
    { kind: sculpt,
      bounded_shape_ref: exact TerrainEditShapeRefV1,
      parameters_ref: exact TerrainSculptParametersRefV1 }
    | { kind: smooth,
        bounded_shape_ref: exact TerrainEditShapeRefV1,
        parameters_ref: exact TerrainSmoothParametersRefV1 }
    | { kind: flatten,
        bounded_shape_ref: exact TerrainEditShapeRefV1,
        parameters_ref: exact TerrainFlattenParametersRefV1 }
    | { kind: stamp,
        bounded_shape_ref: exact TerrainEditShapeRefV1,
        stamp_source_ref: exact TerrainStampSourceRefV1,
        parameters_ref: exact TerrainStampParametersRefV1 }
    | { kind: layer_paint,
        bounded_shape_ref: exact TerrainEditShapeRefV1,
        layer_ref: exact TerrainLayerRefV1,
        parameters_ref: exact TerrainLayerPaintParametersRefV1 }
    | { kind: visibility_hole,
        bounded_shape_ref: exact TerrainEditShapeRefV1,
        parameters_ref: exact TerrainHoleParametersRefV1 }
    | { kind: region_transform,
        transform_ref: exact TerrainRegionTransformRefV1 }
  affected_tile_keys[1..4096]: TerrainTileKeyV1
  authoring_changeset_ref: exact ChangeSetRefV1
  operation_content_hash: SHA-256
```

Operationは[Project State](../03-authoring/project-state.md)のChangeSetへ入り、undo／redoはSource revisionを戻す新しい正規Commitである。Runtime tile、render buffer、physics bodyへ直接writeしない。union外Field、operation sequence gap、base revision mismatch、scope外tile、branch／parameters mismatch、上限超過をrejectする。`affected_tile_keys[]`はbranchのbounded shapeまたはregion transformから決定論的に再計算した集合とset equalityにする。

## 4. Foliage Sourceとidentity

```text
FoliageDocumentV1
  foliage_id: StableId
  source_revision: positive u64
  world_scope_ref: exact WorldScopeRefV1
  species_catalog_ref: exact FoliageSpeciesCatalogRefV1
  placement_set_refs[0..4096]: FoliagePlacementSetRefV1
  authored_instance_set_ref: exact AuthoredFoliageInstanceSetRefV1
  target_policy_ref: exact FoliageTargetPolicyRefV1
  document_content_hash: SHA-256

FoliageSpeciesV1
  species_id: StableId
  species_version: positive u32
  variant_refs[1..64]: exact FoliageVariantRefV1
  placement_constraint_ref: exact FoliagePlacementConstraintRefV1
  representation_request_ref: exact FoliageRepresentationRequestRefV1
  material_binding_ref: exact MaterialBindingRefV1
  lighting_participation_ref: exact LightingParticipationRefV1
  environment_response_ref: exact FoliageEnvironmentResponseRefV1
  interaction_binding_ref: optional exact FoliageInteractionBindingRefV1
  collision_projection_policy_ref:
    optional exact CollisionProjectionPolicyRefV1
  navigation_projection_policy_ref:
    optional exact NavigationProjectionPolicyRefV1
  species_content_hash: SHA-256
```

Speciesはwind、season、wetness、snow、damage、harvest等のmeaningを所有せず、各Ownerが定義するtyped input／interaction bindingだけを持つ。Game固有health、loot、growth simulationをSpecies Sourceへ埋め込まない。

```text
FoliagePlacementSetV1
  placement_set_id: StableId
  placement_set_version: positive u32
  species_ref: exact FoliageSpeciesRefV1
  world_cell_binding_refs[1..4096]:
    exact WorldCellBindingRefV1
  terrain_surface_selector_ref:
    optional exact TerrainSurfaceSelectorRefV1
  placement_rule:
    { kind: explicit_points,
      point_refs[1..65536]:
        exact FoliagePlacementPointRefV1 }
    | { kind: density_mask,
        density_mask_ref: exact FoliageDensityMaskRefV1,
        sampling_profile_ref:
          exact FoliageSamplingProfileRefV1 }
    | { kind: deterministic_scatter,
        seed_u64,
        distribution_profile_ref:
          exact FoliageDistributionProfileRefV1,
        density_source_ref:
          exact FoliageDensitySourceRefV1 }
    | { kind: surface_sample,
        source_surface_ref: exact SurfaceSourceRefV1,
        sampling_profile_ref:
          exact FoliageSamplingProfileRefV1 }
    | { kind: spline_band,
        spline_ref: exact WorldSplineRefV1,
        band_profile_ref: exact FoliageBandProfileRefV1 }
  exclusion_refs[0..256]: exact FoliageExclusionRefV1
  placement_content_hash: SHA-256
```

各branchはbounded count、finite coordinate、canonical orderingを定義し、union外Fieldを禁止する。`seed_u64`だけをdeterminism根拠にせず、algorithm contract version、species／variant refs、mask／surface／Terrain revision、coordinate profile、Cell binding、exclusion refs、Target-independent canonical sample orderをscatter keyへ含める。

```text
FoliageInstanceIdentityV1
  identity:
    { origin: authored,
      foliage_id: StableId,
      authored_instance_id: StableId }
    | { origin: generated,
        placement_set_ref: exact FoliagePlacementSetRefV1,
        scatter_key_hash: SHA-256,
        canonical_sample_index: bounded u32 }
  identity_content_hash: SHA-256
```

```text
AuthoredFoliageInstanceSetV1
  instance_set_id: StableId
  instance_set_version: positive u32
  source_foliage_id: StableId
  source_revision: positive u64
  instances[0..65536]: AuthoredFoliageInstanceV1
  instance_set_content_hash: SHA-256

AuthoredFoliageInstanceV1
  identity: FoliageInstanceIdentityV1(origin=authored)
  species_ref: exact FoliageSpeciesRefV1
  variant_selection:
    { kind: explicit,
      variant_ref: exact FoliageVariantRefV1 }
    | { kind: species_policy,
        selection_key_hash: SHA-256 }
  world_transform_ref: exact FiniteWorldTransformRefV1
  world_cell_binding_ref: exact WorldCellBindingRefV1
  surface_binding:
    { kind: terrain,
      selector_ref: exact TerrainSurfaceSelectorRefV1 }
    | { kind: world_surface,
        source_ref: exact SurfaceSourceRefV1 }
    | { kind: free_placement }
  instance_content_hash: SHA-256
```

union外Fieldを禁止する。authored identityはSource revisionやtransform編集で変わらず、同じFoliage Document内でStable IDを再利用しない。Source snapshotとinstance recordがrevision／content hashを持つため、stable identityとmutable authored stateを混同しない。generated identityはscatter key変更で新generationとなる。旧generated identityを同位置の新instanceへ近似migrationしない。persistent／Gameplay stateをgenerated instanceへ許可する場合は[Persistence／Save](../04-runtime/persistence-save.md)とGameplay Ownerの別identity／migration Decisionが必要で、位置、species、sample indexだけから旧stateを付け替えない。

## 5. Cooked Artifactとinvalidation

```text
TerrainCookRequestV1
  terrain_ref: exact TerrainDocumentRefV1
  region_ref: exact TerrainRegionRefV1
  tile_key_set[1..4096]: TerrainTileKeyV1
  neighbor_requirement_set[0..16384]:
    exact TerrainNeighborRequirementV1
  target_profile_ref: exact TargetProfileRefV1
  material_catalog_ref: exact MaterialCatalogRefV1
  toolchain_profile_ref: exact ToolchainProfileRefV1
  request_content_hash: SHA-256

CookedTerrainRegionArtifactV1
  artifact_ref: exact DerivedArtifactRefV1
  source_terrain_ref: exact TerrainDocumentRefV1
  source_region_ref: exact TerrainRegionRefV1
  tile_records[1..4096]: CookedTerrainTileRecordV1
  seam_manifest_ref: exact TerrainSeamManifestRefV1
  layer_projection_ref: exact TerrainLayerProjectionRefV1
  representation_candidate_refs[1..16]
  collision_source_projection_ref:
    optional exact CollisionSourceProjectionRefV1
  navigation_source_projection_ref:
    optional exact NavigationSourceProjectionRefV1
  receiver_projection_refs[0..16]
  target_profile_ref: exact TargetProfileRefV1
  artifact_content_hash: SHA-256
```

Terrain publication closureはrequestしたtile set、required neighbor seam records、layer projection、Target Profile、Source revisionのset equalityを必要とする。neighborが別revision、tile欠落、seam proof欠落、projectionだけ新しい場合は全closureをpublishしない。

```text
FoliageCookRequestV1
  foliage_ref: exact FoliageDocumentRefV1
  placement_set_refs[0..4096]:
    exact FoliagePlacementSetRefV1
  world_cell_binding_refs[1..4096]:
    exact WorldCellBindingRefV1
  terrain_artifact_refs[0..4096]:
    exact CookedTerrainRegionArtifactRefV1
  target_profile_ref: exact TargetProfileRefV1
  toolchain_profile_ref: exact ToolchainProfileRefV1
  request_content_hash: SHA-256

CookedFoliageRegionArtifactV1
  artifact_ref: exact DerivedArtifactRefV1
  source_foliage_ref: exact FoliageDocumentRefV1
  source_placement_set_refs[0..4096]:
    exact FoliagePlacementSetRefV1
  world_cell_binding_refs[1..4096]:
    exact WorldCellBindingRefV1
  scatter_generation_ref: exact FoliageScatterGenerationRefV1
  authored_instance_records[0..65536]:
    CookedAuthoredFoliageInstanceRecordV1
  generated_cluster_records[0..65536]:
    CookedGeneratedFoliageClusterRecordV1
  representation_candidate_refs[1..64]:
    exact RepresentationCandidateRefV1
  collision_source_projection_ref:
    optional exact CollisionSourceProjectionRefV1
  navigation_source_projection_ref:
    optional exact NavigationSourceProjectionRefV1
  target_profile_ref: exact TargetProfileRefV1
  artifact_content_hash: SHA-256
```

Foliage Artifactはauthoredとgenerated recordsを別配列にし、identity classを保持する。clusterはstorage／presentation unitであり、Instance identity、World Cell membership、Gameplay identityを統合しない。partial cluster、mixed scatter generation、missing exclusion revision、Terrain surface selector mismatchをrejectする。

invalidationは依存graphから決定する。

| 変更 | Terrain invalidation | Foliage invalidation |
|---|---|---|
| Terrain tile／layer／hole | affected tile＋seam neighbor＋各projection | そのsurfaceを参照するplacement setだけ |
| World Cell mapping／origin contract | binding対象artifact | binding対象cluster／scatter |
| Species／variant | なし | exact species consumer |
| density／mask／seed／exclusion | なし | exact placement setとcluster |
| Material artifact | layer／variant binding projection | exact variant binding |
| Environment state runtime変化 | Source recookなし。typed runtime input | Source recookなし。typed runtime input |
| Collision／Navigation algorithm | Terrain Source不変、projection recook | Foliage Source不変、projection recook |

表示名、file timestamp、Editor selection、runtime visibilityからinvalidationを推測しない。

## 6. World、LOD、Runtime Assetとのhandoff

Worldは次を提供する。

- exact World／Scene／Space／Cell refsとrevision
- coordinate／origin profile
- partition membershipとactivation set
- Runtime Entry／World Packageに含めるartifact closure

本書は次を提供する。

- Terrain region／tileとFoliage placement／clusterのdomain identity
- exact World Cell binding
- representation candidatesとerror／coverage metadata
- Collision／Navigation／Environment／Lightingへのsource projection
- runtime domain snapshot

LOD ownerはcandidateからrepresentation tierを選択し、本書はthreshold、View距離、hysteresis、transitionを決めない。Virtualized Geometry ownerは選択候補のgeometry representationを所有し、本書はpage、cluster tree、inner cut、residencyを決めない。Runtime Asset ownerはCooked Artifactのrequest／dependency／generation／lease／evictionを所有し、本書は別Terrain cache／Foliage cache authorityを作らない。

```text
TerrainRuntimeSnapshotV1
  snapshot_id: content-derived StableId
  world_activation_generation_ref:
    exact WorldActivationGenerationRefV1
  active_region_generations[0..4096]:
    exact CookedTerrainRegionGenerationRefV1
  active_tile_keys[0..65536]: TerrainTileKeyV1
  selected_representation_refs[0..65536]:
    exact ResolvedRepresentationRefV1
  environment_receiver_state_ref:
    optional exact EnvironmentReceiverStateRefV1
  snapshot_content_hash: SHA-256

FoliageRuntimeSnapshotV1
  snapshot_id: content-derived StableId
  world_activation_generation_ref:
    exact WorldActivationGenerationRefV1
  active_region_generations[0..4096]:
    exact CookedFoliageRegionGenerationRefV1
  authored_instance_generation_refs[0..65536]:
    exact AuthoredFoliageInstanceGenerationRefV1
  generated_cluster_generation_refs[0..65536]:
    exact GeneratedFoliageClusterGenerationRefV1
  selected_representation_refs[0..65536]:
    exact ResolvedRepresentationRefV1
  environment_response_state_ref:
    optional exact EnvironmentResponseStateRefV1
  snapshot_content_hash: SHA-256

TerrainRuntimeSnapshotRefV1
  snapshot_id: StableId
  world_activation_generation_ref:
    exact WorldActivationGenerationRefV1
  snapshot_content_hash: SHA-256

FoliageRuntimeSnapshotRefV1
  snapshot_id: StableId
  world_activation_generation_ref:
    exact WorldActivationGenerationRefV1
  snapshot_content_hash: SHA-256

TerrainFoliageRepresentationSummaryV1
  schema_version: 1
  summary_id: content-derived StableId
  summary_generation: positive u64
  project_revision: positive u64
  world_scope_ref: exact WorldScopeRefV1
  target_profile_ref: exact TargetProfileRefV1
  quality_profile_ref: exact QualityProfileRefV1
  world_activation_generation_ref:
    exact WorldActivationGenerationRefV1
  terrain_snapshot_ref: nullable<exact TerrainRuntimeSnapshotRefV1>
  foliage_snapshot_ref: nullable<exact FoliageRuntimeSnapshotRefV1>
  entries[0..65536]:
    domain: terrain | foliage
    subject_ref: exact versioned Terrain／Foliage domain ref
    bounds_ref: exact World-space bounds ref
    mobility_class: static | dynamic | wind_deformed
    opacity_class: opaque | masked | qualified_extended
    coverage_class: surface | volume | sparse_instances
    representation_refs[1..16]:
      exact versioned representation ref
  summary_content_hash: SHA-256

TerrainFoliageRepresentationSummaryRefV1
  summary_id: StableId
  summary_generation: positive u64
  summary_content_hash: SHA-256
```

SnapshotとSummaryはimmutable frame／publication inputで、native buffer、pointer、descriptor、Render pass、Physics body、Nav mesh、Runtime queueを含めない。World activation generationとartifact generationが一致しないSnapshotをpublishしない。Runtime Snapshot Refはsnapshot identity、World activation generation、hashの全Field、Summary Refは解決先三Fieldとbyte equalityにする。Summary entriesは`domain, subject_ref`順、各`representation_refs[]`はexact refのcanonical byte順へstrict sortしてduplicateを拒否し、全entryを通じたrepresentation ref総数をchecked sumで`<= 1048576`とする。Terrain／Foliageがどちらも不在の時だけ両Snapshot refとentriesを空にでき、片方が存在する場合は対応domain entry集合をそのSnapshotのactive subject closureとset equalityにする。Summary hashはASCII `MIRAKAN_TERRAIN_FOLIAGE_REPRESENTATION_SUMMARY_V1`と自己hashを除くclosed recordのcount／presence／length-framed canonical bytesからSHA-256する。

## 7. Environment、Lighting、Collision、Navigation、Gameplay

typed handoffを次に固定する。

| Consumer | 本書が提供するもの | Consumerが所有し続けるもの |
|---|---|---|
| Environment | surface receiver IDs、bounds、layer role、vegetation response binding | weather、water、snow／wetness状態とsimulation／presentation |
| Lighting／Advanced Light Transport | bounds、mobility class、opacity／coverage class、representation availability | Light／GI／reflection／shadow semanticsとTechnique |
| Collision／Physics | revision付きsurface／instance source projection | Collider／shape／body、filter、contact |
| Navigation | revision付きsurface、hole、obstacle source projection | grid／mesh、cost、agent、query |
| Gameplay | typed interaction／damage／harvest bindingとstable authored identity | health、loot、quest、authority、Save policy |
| Render Graph | resolved representationを含むimmutable Snapshot | culling、pass、resource、submission |
| Persistence／Replay | semantic Source／authored identity、generated generation lineage | Save record、migration、Replay envelope |

WeatherまたはseasonからFoliage transform／material responseを得る場合も、Environment input→Foliage response state→Material／Animation presentationの一方向にする。Foliageがweather authorityまたはSimulation cadenceを持たない。

## 8. Streaming、fallback、failure atomicity

本書の`streaming`はWorld Cell activationとRuntime Asset generationにdomain artifactを対応付ける意味であり、I/O scheduler、queue、priority、lease、evictionを所有する意味ではない。

```text
TerrainFoliageFallbackStepV1
  step_id/version/content_hash
  source_binding:
    { domain: terrain,
      source_ref: exact TerrainDocumentRefV1,
      target_policy_ref: exact TerrainTargetPolicyRefV1 }
    | { domain: foliage,
        source_ref: exact FoliageDocumentRefV1,
        target_policy_ref: exact FoliageTargetPolicyRefV1 }
  from_representation_ref: exact RepresentationRefV1
  fallback_target:
    { kind: representation,
      representation_ref: exact RepresentationRefV1 }
    | { kind: unavailable }
  trigger_set[1..16]:
    target_unsupported
    | artifact_missing
    | artifact_stale
    | representation_unavailable
    | memory_pressure
    | device_fault
  meaning_disposition:
    equivalent | presentation_degraded | unavailable
  invariant_outcomes[5..6]:
    invariant:
      world_membership
      | source_identity
      | authored_identity
      | collision_semantics
      | navigation_semantics
      | save_identity
    outcome: preserved | unavailable
  transition_boundary:
    next_activation_generation | reload_required
```

fallbackはexact Source／Target policyに登録し、Runtime AssetまたはRendererが独自に選ばない。Terrain branchのinvariant集合は`{world_membership, source_identity, collision_semantics, navigation_semantics, save_identity}`、Foliage branchはこれに`authored_identity`を加えた集合とし、`invariant_outcomes[].invariant`を各domain集合とset equalityにする。`equivalent | presentation_degraded`では全outcomeを`preserved`、`unavailable`では全invariantを省略なく列挙して各loss／preservationをtypedにする。`fallback_target.kind=unavailable`は`meaning_disposition=unavailable`に限り、representation branchでは`unavailable`を禁止する。presentation fallbackでCollision／Navigation projectionが変わる場合は`equivalent`にできず、同じGameplay session中のsilent transitionを禁止する。Terrainをflat plane、Foliageをempty set、missing materialをdefault materialへ暗黙変換しない。

最低限、次を別failureにする。

- wrong／stale Terrain／Foliage／World／Target ref
- overlapping authority region、invalid coordinate／bounds、tile key overflow
- missing／stale neighbor、seam mismatch、partial tile generation
- layer／mask／hole semantic mismatch
- scatter key mismatch、nondeterministic order、authored／generated identity collision
- unknown species／variant、placement branch mismatch、exclusion conflict
- Cell activation／artifact generation mismatch
- LOD／virtual representation missing、runtime lease unavailable
- Collision／Navigation／Environment projection revision mismatch
- fallback invariant violation、no qualified Target fallback

retryはSource／dependency／Target／capacity changeから新Cookまたはactivation generationを作る。同じfailed generationを部分repairして成功扱いにしない。

## 9. AI／Editor理解境界

AI／Editor projectionはTerrain region／tile／layer／operation、Foliage species／variant／placement／origin、World Cell binding、dependency revision、Cook／runtime state、representation candidate、fallback、diagnostic、Target Qualificationをboundedに示す。raw height／mesh／mask bytes、native buffer、private path、pointer、GPU handleを説明用に公開しない。

自然言語の「広い地形」「森を増やす」「写実的」に対し、World scope、TerrainまたはFoliage、source kind、density／distribution、authored identity、Collision／Navigation、Target、quality／performance intentを区別する。選択が意味を変える不足だけをassumption／questionにし、`production`、`AAA`、`open world`からtile寸法、species、seed、LOD、virtual geometry、GIを自動選択しない。

create／sculpt／paint／scatter／preview／rebuild／explainはplanned semantic vocabularyで、current MCD Operation、Service、Tool、Provider surfaceではない。完全なOperation／Authority／Validator／Receipt／publication closureが承認されるまでaction名からIDを生成またはdispatchしない。

## 10. Security、capacity、Qualification

Source path／bytes、height／mesh／mask、Layer／Species metadata、seed、bounds、count、compressed size、dependency、external DCC outputをuntrusted inputとして扱う。path traversal、decompression bomb、integer overflow、NaN／Inf、oversized count、cyclic dependency、malformed mesh／mask、nondeterministic plugin outputをrejectする。external tool／libraryのversion、hash、license、sandboxは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)と[Asset Lifecycle](../03-authoring/asset-lifecycle.md)だけが所有する。

共通memory／CPU／GPU／I/O measurement、budget、backpressureは[Performance／Capacity](../04-runtime/performance-capacity.md)、Evidence signature／freshness／revocationは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)に委譲する。Terrain／Foliageはdomain demand、upper bound、selected fallback、reasonだけを返す。

Terrain Qualificationは少なくとも次を含む。

1. region／tile create、sculpt、stamp、layer、hole、undo／redo、reimport
2. neighbor seam、negative coordinate、world origin change、Cell境界
3. height field、mesh、hybridのapplicable Source／Artifact／fallback
4. water／snow／wetness、Collision／Navigation projectionのrevision一致
5. cook／load／unload／reload、partial／stale／corrupt artifact、device loss
6. LOD／virtual representationとTarget別content-quality／scale／memory pressure

Foliage Qualificationは少なくとも次を含む。

1. authored instanceとgenerated scatterのidentity／ordering／rebuild
2. species／variant、density／mask／exclusion、seed、Terrain／surface binding
3. Cell activation／streaming、LOD／cull／virtual representation
4. wind／season／wetness／snow response、interaction／damage handoff
5. Collision／Navigation／Lighting participationとpresentation fallback
6. large instance count、invalid mask、stale Terrain、partial cluster、Target pressure

共通negative fixtureはwrong Owner／ID／version／hash、duplicate、sort違反、inactive union payload、count超過、mixed generation、missing exact ref、stale／revoked Qualification Bindingを一原因ずつ検出する。Terrain ReceiptでFoliageを、Foliage ReceiptでTerrainを、desktop Receiptでmobileをsupportしない。

## 11. 公式資料から採用する構造と限界

- [Unreal Engine: Landscape Overview](https://dev.epicgames.com/documentation/unreal-engine/landscape-overview?lang=en-US)はTerrainを専用authoring／rendering systemとして扱い、World Partitionと連携する。本書はTerrain domainとWorld partition authorityを分離する。
- [Unreal Engine: Foliage Mode](https://dev.epicgames.com/documentation/en-us/unreal-engine/foliage-mode-in-unreal-engine)はStatic Mesh FoliageとActor Foliage、painting／placementを区別する。本書はauthored／generated identityとplacement branchを明示する。
- [Unity Terrain](https://docs.unity3d.com/ja/current/Manual/script-Terrain.html)はTerrain data、neighbor、tree／detail等を専用surfaceとして扱う。本書はsource／artifact／neighbor closureを独立させる。

外部EngineのClass名、file format、algorithm、tile／density既定値、API、marketing claimをMiraikanaiのcanonical IDまたは採用要件にしない。比較から採用するのはdomain specialization、World連携、identity、Target／fallback分離だけである。

## 12. 明示的非目標と設計Closure

非目標は、World partitionまたはgeneric Scene Ownerの置換、独自LOD／virtual geometry／Asset Manager／Rendererの新設、Physics／Navigation／Environment／Gameplay authorityの統合、procedural Runtime generationの自動採択、特定Provider／algorithmの選定、実装Task／順序／担当／工数／日程の定義である。

本書の設計Closureは次をすべて満たす場合だけ成立する。

1. TerrainとFoliageが別Source、Artifact、Snapshot、Capability、Target Qualificationで独立Activationできる。
2. World Cell／partition、LOD selection、geometry representation、Runtime Asset lifecycleのauthorityが既存Ownerに残る。
3. authored／generated instance identityとscatter determinismが機械判定できる。
4. Terrain seam／neighborとFoliage generationがpartial publishされない。
5. Collision／Navigation／Environment／Lighting／Render handoffが同じSource／artifact revisionへ閉じる。
6. fallbackがWorld membership、identity、Collision／Navigation、Save meaningをsilent変更しない。
7. 本書の存在を実装済み、large-world対応、production Terrain／Foliage、AAA品質の根拠にしない。
