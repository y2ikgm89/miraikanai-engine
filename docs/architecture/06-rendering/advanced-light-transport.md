# Miraikanai Engine Advanced Light Transport Contract

- 文書ID: mirakan.arch.rendering-advanced-light-transport
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: diffuse indirect／specular indirect／shadow visibility／reference transportのsemantic channel、channel別technique family、Target support解決、scene representation requirement、light-transport固有history／denoise intent、意味同等fallback、Shadow technique解決、domain diagnostic／qualification
- 非正本範囲: Light Source／photometry／attenuation／Shadow authoring intent、Material response／Project Shader source、Environment radiance source、World／Terrain／geometry／LOD／residency、Render Graph pass／resource／queue／barrier／history allocation／submission、generic Post effect、device API、shared capacity、Evidence envelope、Product quality claim、実装Task／順序。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Render Graph](render-graph.md)、[Lighting](lighting.md)、[Materials](materials.md)、[Post Processing](post-processing.md)、[World](world.md)
- 関連文書: [Product Plan](../00-product/product-plan.md)、[Advanced Rendering／Multiplayer Ownership Decision](../decisions/2026-07-29-advanced-rendering-multiplayer-ownership.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)、[Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)、[Debugging／Observability／Replay](../04-runtime/debugging-observability-replay.md)、[Environment／Surfaces](environment-surfaces.md)、[Terrain／Foliage](terrain-foliage.md)、[LOD](lod.md)、[Virtualized／Continuous Geometry](virtualized-continuous-geometry.md)、[Project Shader](project-shader.md)、[Camera](camera.md)、[Windows](../07-platform/windows.md)、[Mobile Common](../07-platform/mobile-common.md)
- 根拠区分: project-decision／official-documentation comparison（未計測のquality threshold、ray／sample count、distance、cache量、時間／memory値はprovisional）
- 外部根拠確認日: 2026-07-29

## 1. 結論、状態、所有境界

本書は「照明結果をどの方式で得るか」のsemantic authorityを所有する。GI、reflection、advanced shadow、path-traced referenceを一つの`advanced_rendering` booleanへ潰さず、`diffuse_indirect | specular_indirect | shadow_visibility | reference_transport`の独立channelとして解決する。同じViewでGIはsoftware ray、reflectionはhardware ray、shadowはraster、reference比較だけpath tracingというhybridを表現できなければならない。

[Lighting](lighting.md)はLight Source、単位、色、attenuation、mobility、linkingと`ShadowIntentV1`／`ShadowStyleProfileV1`を所有する。[Materials](materials.md)はsurface response、[Environment／Surfaces](environment-surfaces.md)はsky／atmosphere／fog／cloud／water等のradiance source、[World](world.md)と[Terrain／Foliage](terrain-foliage.md)はspatial source、[LOD](lod.md)と[Virtualized／Continuous Geometry](virtualized-continuous-geometry.md)はrepresentation、[Runtime Asset Lifecycle](../04-runtime/runtime-asset-lifecycle.md)はgeneration／residency／leaseを所有する。本書はそれらのexact snapshot／artifact refを読み、意味を書き換えない。

[Render Graph](render-graph.md)は、本書が出力する`ResolvedLightTransportPlanV1`と`LightTransportExecutionRequestV1`をpass／resource／queue／barrier／history resource／submissionへ展開する。`RayTracingPortV1`、`RadianceCachePortV1`、`NeuralRenderModelV1`、native Backend AdapterはRender Graphに残る。本書はnative command、descriptor、acceleration handle、shader table、queue、barrierを公開または保存しない。

本書、型、Profile、Future entryはすべて`review`／`absent`／`planning_only`の設計候補である。MVP、current Capability、Target support、Provider採用、実装開始、Product claimを変更しない。

## 2. 不変条件と共通語彙

1. channel、technique、Target、quality class、Providerは直交軸であり、名前の一致から相互に推測しない。
2. `ray`はscene traversal technique、`path_traced`はtransport estimator family、`neural`は許可済みreconstruction／denoise providerであり、それ自体をProduct qualityまたはsemantic channelにしない。
3. Source／Profile、Cooked representation、Resolved Plan、Runtime Snapshot、Qualification Receiptを別Recordにし、ReceiptをSource／Plan hashへ埋め戻さない。
4. Refはlogical ID、positive version／revision、content hashを持つexact Refとする。表示名、path、`latest`、近いTarget、同名Providerへfallbackしない。
5. `unsupported`、`unqualified`、`qualified`、`degraded`、`fault`を区別する。起動成功、pass成功、reference画像生成を別channelのproduction supportとして扱わない。
6. fallbackはSource Profileに順序付きで列挙し、meaning-equivalence class、Target scope、quality差、再解決境界を持つ。resource不足時のsilent ray／sample削減、別technique挿入、channel無効化を禁止する。
7. semantic profileの解決はpureかつdeterministicであり、GPU timing、worker完了順、現在residentな類似artifact、viewportの一時状態をauthorityにしない。
8. channelは独立にActivation／Qualificationできる。一channelのReceipt、fallback、fault、disableを他channelへ伝播させない。

closed値を次に固定する。

```text
LightTransportChannelV1 =
  diffuse_indirect
  | specular_indirect
  | shadow_visibility
  | reference_transport

LightTransportTechniqueKindV1 =
  disabled
  | baked_field
  | probe_field
  | screen_space
  | software_ray
  | hardware_ray
  | hybrid
  | path_traced

LightTransportSupportStateV1 =
  unsupported
  | supported_unqualified
  | qualified
  | degraded
  | fault

LightTransportMeaningClassV1 =
  reference
  | production_equivalent
  | quality_degraded
  | unavailable
```

`hybrid`は自由な複合名ではない。2～4件の`LightTransportTechniqueRefV1`、channel-local mask、合成operator、優先順、重複coverage拒否規則へ閉じたProfileである。`disabled`は明示branchで、payloadは空、meaning classは`unavailable`に限る。

## 3. Source Profileとidentity

Project sourceは次のrootへ閉じる。

```text
AdvancedLightTransportDocumentV1
  document_id: StableId
  source_revision: positive u64
  profile_refs[1..64]: LightTransportProfileRefV1
  world_bindings[0..1024]: LightTransportWorldBindingV1
  target_policy_ref: exact TargetLightTransportPolicyRefV1
  document_content_hash: SHA-256

LightTransportProfileV1
  profile_id: StableId
  profile_version: positive u32
  owner_ref: exact OwnerRefV1
  visual_style_profile_ref: exact VisualStyleProfileRefV1
  channel_requirements[4]: LightTransportChannelRequirementV1
  technique_bindings[4]: LightTransportTechniqueBindingV1
  target_profile_refs[1..64]: exact TargetProfileRefV1
  budget_intent_ref: exact LightTransportBudgetIntentRefV1
  profile_content_hash: SHA-256
```

`channel_requirements[]`と`technique_bindings[]`はchannel ordinal順、Target refsはexact Target tuple順へstrict sortし、duplicateを拒否する。両配列は四channelとそれぞれset equalityで、同じchannelはexact一件のRequirementとBindingを持つ。Profileがchannelを使わない場合も`disposition=disabled`とdisabled Technique bindingを明示する。配列欠落をdefault enable／disableとして解釈しない。

```text
LightTransportTechniqueBindingV1
  channel: LightTransportChannelV1
  primary_technique_ref: exact LightTransportTechniqueRefV1
  fallback_ladder_ref: exact LightTransportFallbackLadderRefV1

LightTransportFallbackLadderV1
  ladder_id: StableId
  ladder_version: positive u32
  channel: LightTransportChannelV1
  step_refs[0..16]:
    exact LightTransportFallbackStepRefV1
  ladder_content_hash: SHA-256
```

Bindingのprimary TechniqueとLadderは同じchannelを含む。`disposition=disabled`のprimaryは`technique_kind=disabled`、Ladderは空にする。`optional | required`のprimaryはnon-disabledとし、Ladderの第一stepはprimaryを`from_technique_ref`、後続stepは直前stepの`to_technique_ref`をfromとして一つのlinear chainを形成する。分岐、cycle、同じstep refの重複、途中のdisabledからの後続stepを拒否する。triggerが同じ複数候補の優先順はこのstep配列順だけで決まり、Target／runtime状態から並べ替えない。

```text
LightTransportChannelRequirementV1
  channel: LightTransportChannelV1
  requirement:
    { disposition: disabled }
    | { disposition: optional | required,
        source_participation:
          static | dynamic | both,
        receiver_participation:
          opaque | opaque_and_masked | qualified_extended,
        emissive_participation:
          disabled | static | dynamic_qualified,
        translucent_participation:
          excluded | receive_only | qualified_full,
        volumetric_participation:
          excluded | receive_only | qualified_full,
        view_scope:
          camera | view_family | capture | offline_reference,
        quality_intent_ref:
          exact LightTransportQualityIntentRefV1,
        accepted_meaning_classes[1..3]:
          LightTransportMeaningClassV1 }
```

union外Fieldを禁止する。`disposition=disabled`はactive requirement payloadを持たず、resolved channelは`unsupported`、Techniqueなし、meaning `unavailable`となる。`required`はaccepted meaningのqualifiedまたはdegraded Techniqueが得られなければPlan全体を生成せずDiagnosticを返す。`optional`は不成立channelを明示的な`unsupported | supported_unqualified | fault`として保持できるが、別channelのPlanを妨げない。

`reference_transport`は`view_scope=offline_reference | capture`を既定候補とするが、runtime candidateをSource名から推測しない。runtime利用には別Target Profile、budget、fault、Qualificationが必要である。reference成功をpreviewまたはruntime supportと表示しない。

`LightTransportWorldBindingV1`は`{binding_id, binding_version, world_ref, profile_ref, region_selector_ref, binding_content_hash}`を持つ。World名、Camera名、Environment名、active Editor selectionでbindingを推測しない。Region selectorは[World](world.md)のexact Scene／Space／Cell scope refだけを参照し、partitionを本書へ複写しない。

## 4. Technique familyとscene representation requirement

各Techniqueは次のReceipt-free baseを持つ。

```text
LightTransportTechniqueV1
  technique_id: StableId
  technique_version: positive u32
  channel_set[1..4]: LightTransportChannelV1
  definition:
    { technique_kind: disabled }
    | { technique_kind:
          baked_field | probe_field | screen_space |
          software_ray | hardware_ray | path_traced,
        provider_lock_ref: optional exact ProviderLockRefV1,
        required_renderer_capability_refs[0..64],
        channel_bindings[1..4]:
          LightTransportTechniqueChannelBindingV1 }
    | { technique_kind: hybrid,
        members[2..4]:
          { member_technique_ref:
              exact LightTransportTechniqueRefV1,
            channel_mask[1..4]:
              LightTransportChannelV1,
            composition_priority: 1..4 },
        composition_operator_ref:
          exact RegisteredTransportCompositionOperatorRefV1,
        coverage_overlap_policy: reject | explicit_priority,
        provider_lock_ref: optional exact ProviderLockRefV1,
        required_renderer_capability_refs[0..64],
        channel_bindings[1..4]:
          LightTransportTechniqueChannelBindingV1 }
  technique_content_hash: SHA-256

LightTransportTechniqueChannelBindingV1
  channel: LightTransportChannelV1
  representation_requirement_ref:
    exact LightTransportRepresentationRequirementRefV1
  temporal_intent_ref:
    optional exact LightTransportHistoryIntentRefV1
  denoise_intent_ref:
    optional exact LightTransportDenoiseIntentRefV1
  output_semantics[1..16]: registered semantic ID
  target_support_policy_ref:
    exact LightTransportTargetSupportPolicyRefV1
```

union外Fieldを禁止する。`disabled`はactive execution Fieldと`channel_bindings[]`を一つも持たず、`hybrid`だけがmember／mask／composition Fieldを持つ。non-disabled branchの`channel_bindings[]`はchannel uniqueで、channel集合を親`channel_set[]`とset equalityにする。representation、history、denoise、output semantic、Target supportはこのchannel-local bindingからだけ解決し、Technique全体のflat Field、semantic名filter、別channel bindingから推測しない。hybridのmemberは非hybridかつ非disabled、acyclic、同じTargetでeligibleでなければならない。member refはunique、各maskはmemberのchannel setのnon-empty subset、全mask unionは親Techniqueの`channel_set[]`とset equalityにする。`coverage_overlap_policy=reject`ではmask間intersectionを空、`explicit_priority`では`composition_priority`を`1..member count`のexact permutationとし、overlapのwinnerを小さいpriorityへ固定する。`members[]`はpriority順へ正規化する。Provider固有struct、native shader identifier、API extension、command callbackを含めない。`provider_lock_ref`のversion、hash、license、取得元、build optionは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)だけが決める。

```text
LightTransportRepresentationRequirementV1
  requirement_id/version/content_hash
  requirements[0..16]:
    kind:
      none
      | raster_visibility_buffers
      | baked_transport_field
      | probe_volume
      | signed_distance_field
      | acceleration_structure
      | radiance_cache
    source_owner_ref: exact OwnerRefV1
    source_revision_ref: exact versioned domain Ref
    geometry_coverage:
      static | dynamic | skinned | terrain | foliage | qualified_subset
    update_class:
      cook_only | load_time | frame_incremental | qualified_dynamic
    completeness:
      required | optional_with_fallback
    target_limit_ref: exact TargetLimitRefV1
```

World／Terrain／geometry ownerは意味を保ったsource／artifactを提供し、LOD ownerはrepresentationを選択し、Runtime Asset ownerはexact generationをreadyにする。本書はrepresentation requirementを宣言してavailabilityを検証するだけで、World Cell、LOD threshold、geometry page、residency queueを所有しない。missing dynamic／skinned／Terrain／Foliage coverageを静的mesh coverageで代用しない。

`acceleration_structure`はRender Graphの`RayTracingPortV1`が実行するlogical requirementであり、native handleまたはbuild commandではない。`radiance_cache`もRender Graphの`RadianceCachePortV1`へ解決されるlogical roleである。

## 5. Channel解決、Target support、Resolved Plan

Resolverは次の一方向contractを持つ。

```text
resolve(
  exact LightTransportProfileRefV1,
  exact LightingSnapshotRefV1,
  exact MaterialTransportSummaryRefV1,
  exact EnvironmentRadianceSummaryRefV1,
  exact WorldRepresentationSummaryRefV1,
  exact TerrainFoliageRepresentationSummaryRefV1?,
  exact ViewFamilyRequirementRefV1,
  exact RendererCapabilitySignatureRefV1,
  exact TargetProfileRefV1,
  exact LightTransportBudgetEnvelopeRefV1
) -> ResolvedLightTransportPlanV1
   | LightTransportDiagnosticSetV1
```

解決順は(1) exact Ref／Source revision／Target整合、(2) channel requirement、(3) channel別Technique eligibility、(4) representation completeness、(5) Target supportとQualification Binding、(6) budget envelope、(7) fallback ladder、(8) Plan固定とする。Runtime availabilityまたは測定値からSource requirementを逆変更しない。

```text
ResolvedLightTransportPlanV1
  plan_id: content-derived StableId
  base_project_revision: positive u64
  source_profile_ref: exact LightTransportProfileRefV1
  target_profile_ref: exact TargetProfileRefV1
  renderer_capability_signature_ref:
    exact RendererCapabilitySignatureRefV1
  world_scope_ref: exact WorldScopeRefV1
  channel_plans[4]: ResolvedLightTransportChannelPlanV1
  representation_requests[0..32]:
    LightTransportRepresentationRequestV1
  execution_requests[0..64]:
    LightTransportExecutionRequestV1
  history_intents[0..16]: LightTransportHistoryIntentV1
  diagnostic_refs[0..128]
  plan_content_hash: SHA-256
```

```text
ResolvedLightTransportChannelPlanV1
  channel: LightTransportChannelV1
  resolution:
    { support_state: unsupported,
      meaning_class: unavailable,
      selected_fallback_ref:
        optional exact LightTransportFallbackStepRefV1 }
    | { support_state: supported_unqualified,
        selected_technique_ref:
          exact LightTransportTechniqueRefV1,
        selected_fallback_ref:
          optional exact LightTransportFallbackStepRefV1,
        meaning_class:
          reference | production_equivalent | quality_degraded,
        representation_requirement_ref:
          exact LightTransportRepresentationRequirementRefV1 }
    | { support_state: qualified,
        selected_technique_ref:
          exact LightTransportTechniqueRefV1,
        selected_fallback_ref:
          optional exact LightTransportFallbackStepRefV1,
        meaning_class:
          reference | production_equivalent,
        representation_requirement_ref:
          exact LightTransportRepresentationRequirementRefV1,
        qualification_binding_ref:
          exact LightTransportQualificationBindingRefV1 }
    | { support_state: degraded,
        selected_technique_ref:
          exact LightTransportTechniqueRefV1,
        selected_fallback_ref:
          exact LightTransportFallbackStepRefV1,
        meaning_class: quality_degraded,
        representation_requirement_ref:
          exact LightTransportRepresentationRequirementRefV1,
        qualification_binding_ref:
          exact LightTransportQualificationBindingRefV1 }
    | { support_state: fault,
        meaning_class: unavailable,
        failed_technique_ref:
          optional exact LightTransportTechniqueRefV1,
        attempted_fallback_ref:
          optional exact LightTransportFallbackStepRefV1 }
  reason_codes[1..32]: registered diagnostic code
```

union外Fieldを禁止する。`qualified`はrequired representation completenessとexact Target Qualification Binding、`reference | production_equivalent` meaning classを持つ。`degraded`もqualifiedなfallback TechniqueとQualification Bindingを持つ。`supported_unqualified`はeligible Techniqueを説明用に保持できるがQualification Bindingを持たず、`unsupported | supported_unqualified | fault`はexecution requestまたはproduction outputを持たない。non-disabledなselected Techniqueの同channel `LightTransportTechniqueChannelBindingV1`をexact一件解決し、channel Planのrepresentation requirement、history／denoise intent、output semantics、Target support／Qualification subjectをそのbindingと一致させる。各`LightTransportExecutionRequestV1`はchannelを持ち、requestが存在するchannel集合は`qualified | degraded` channel集合とset equality、各該当channelは一件以上のrequestを持つ。requestが参照するTechnique、representation generation、history intent、output semanticは同じchannel binding／Planとset equalityにし、別channel requestを借用しない。全channelが非実行なら`execution_requests=[]`が正規である。channel Planはchannel ordinalのexact四件で、Profileにある四channelとset equalityにする。

`LightTransportExecutionRequestV1`はsemantic input／output、registered Pass Template family、resolution／sample class、View scope、history intent ref、representation generation refs、budget reservation ref、ordering constraintだけを持つ。Render Graphはこれをexecutionへ展開するが、Technique選択、channel meaning、fallback ladderを再解釈しない。

## 6. Shadow、Project Technique、path profile

Lighting-owned `ShadowIntentV1`／`ShadowStyleProfileV1`は本書の`shadow_visibility` channelへ入力される。本書は`ShadowGraphV1`、`ResolvedShadowPlanV1`、Shadow Technique選択を所有し、Render Graphはそのexecution requestだけを所有する。

```text
ShadowGraphV1
  graph_id: StableId
  graph_version: positive u32
  source_shadow_intent_ref: exact ShadowIntentRefV1
  source_shadow_style_profile_ref: exact ShadowStyleProfileRefV1
  nodes[1..64]:
    node_id
    registered_transport_node_ref
    parameter_set_ref
  edges[0..128]:
    source_node_id
    output_semantic
    target_node_id
    input_semantic
  output_semantic: shadow_attenuation_linear
  graph_content_hash: SHA-256

ResolvedShadowPlanV1
  plan_id: content-derived StableId
  parent_light_transport_plan_ref:
    exact ResolvedLightTransportPlanRefV1
  base_project_revision: positive u64
  target_profile_ref: exact TargetProfileRefV1
  shadow_intent_ref: exact ShadowIntentRefV1
  shadow_style_profile_ref: exact ShadowStyleProfileRefV1
  shadow_graph_ref: exact ShadowGraphRefV1
  light_binding_refs[1..4096]
  selected_technique_ref: exact LightTransportTechniqueRefV1
  project_shadow_technique_ref:
    optional exact ProjectShadowTechniqueRefV1
  representation_request_refs[0..16]
  execution_request_refs[1..64]
  selected_fallback_ref:
    optional exact LightTransportFallbackStepRefV1
  qualification_binding_ref:
    exact LightTransportQualificationBindingRefV1
  plan_content_hash: SHA-256
```

`ResolvedShadowPlanV1`は親Planの`shadow_visibility` channelから作るread-only domain projectionで、独立Resolver結果ではない。Source／Target／Project revision、selected Technique／fallback／Qualificationを親channel Planとexact一致させ、`representation_request_refs[]`と`execution_request_refs[]`は親Plan内の同channel request集合とset equalityにする。親が`unsupported | supported_unqualified | fault`ならResolved Shadow Planを生成せず、その状態を親channel Planだけで表す。

`ProjectShadowTechniqueV1`は[Project Shader](project-shader.md)の`ProjectShaderTechniqueV1` specializationとし、`injection_port_id=shadow`、必須出力`shadow_attenuation_linear`、Target／Understanding／Qualification closureを持つ。Project Techniqueがsemantic channel、Light authority、Render Graph commandを追加しない。Graphのcycle、semantic mismatch、未登録node、64／128上限超過、final output違反をrejectする。

path familyは次を区別する。

| profile role | 許可用途 | 禁止する推論 |
|---|---|---|
| `reference` | offlineまたはbounded captureの比較基準 | real-time support、shipping support |
| `preview` | Authoring previewと差分説明 | reference精度、runtime budget適合 |
| `runtime_candidate` | 専用Target／fault／budget Qualification候補 | reference／preview Receiptの流用 |

profile roleはTechnique IDと別Fieldである。同じProvider／shaderを共有してもReceipt subject、Target、budget、fixture、fallbackは別にする。

## 7. Temporal historyとdenoise

本書はlight-transport固有のhistory意味、必要input、reset理由、warm-up、denoise intentを所有する。generic Effect composition／exposureは[Post Processing](post-processing.md)、actual image／buffer allocation、lease、barrier、queueは[Render Graph](render-graph.md)が所有する。

```text
LightTransportHistoryIntentV1
  history_intent_id/version/content_hash
  channel: LightTransportChannelV1
  semantic:
    radiance_accumulation
    | visibility_accumulation
    | reservoir
    | denoise_moment
    | reference_accumulation
  required_input_semantics[1..16]
  initialization:
    clear | source_seed | qualified_reprojection
  reset_reasons[1..16]:
    camera_cut
    | projection_change
    | extent_change
    | world_origin_change
    | source_revision_change
    | representation_generation_change
    | technique_change
    | device_generation_change
    | time_discontinuity
  warm_up_disposition:
    withhold | expose_degraded | fallback
  maximum_history_age_ref: exact BoundedAgePolicyRefV1
```

`LightTransportDenoiseIntentV1`はchannel、signal／noise semantics、required auxiliary inputs、spatial／temporal permission、history intent ref、edge preservation class、output meaning、Target limit、fallbackだけを持つ。任意model、network download、runtime training、raw operator graphを保存しない。neural candidateはRender Graphのexact `NeuralRenderModelV1`とToolchain lockへ解決し、non-neural fallbackを別stepにする。

reset理由のunionが発生したframeでは旧historyを部分再利用しない。Render Graphはreset maskをexecutionへ反映し、本書はphysical resource identityを読まない。history欠落をblack／zeroとしてqualified outputに混ぜない。

## 8. Owner handoff

| Producer | 本書が受け取るもの | 本書が返すもの／Consumer |
|---|---|---|
| Lighting | Light snapshot、Shadow Intent／Style exact refs | channel／Shadow Plan。Light Sourceは変更しない |
| Materials／Project Shader | transport response summary、qualified Technique ref | required shading semantic／Technique selection |
| Environment | radiance／atmosphere／volumetric participation summary | channel participation request |
| World／Terrain／Foliage | exact spatial source／artifact revisions | representation requirements |
| LOD／Virtual Geometry | resolved representation family／generation | completeness判定。tierを再選択しない |
| Runtime Asset Lifecycle | ready generation／lease status | exact generation request。I/O／evictionを決めない |
| Camera／Post | View／exposure family、generic history reset context | channel-local history／output semantics |
| Render Graph | Target Capability signature、execution／fault feedback | immutable execution request。pass／resourceは含めない |
| Performance／Capacity | budget envelope、measurement contract | owner-typed demand／fallback reason |
| Product Plan | planning-only Future／quality rubric ref | Qualification summary。Product claimは決めない |

producer refとPlan refは一方向である。Source／artifactはPlanまたはReceiptを逆参照せず、PlanはExecution Receiptをhash inputにせず、Receiptは完成したsubject ref／hashを持つ。

## 9. Fallback、failure、diagnostic

```text
LightTransportFallbackStepV1
  step_id/version/content_hash
  channel: LightTransportChannelV1
  from_technique_ref: exact LightTransportTechniqueRefV1
  to_technique_ref: exact LightTransportTechniqueRefV1
  trigger_set[1..16]:
    capability_unavailable
    | representation_missing
    | artifact_stale
    | budget_denied
    | history_invalid
    | provider_fault
    | device_fault
  resulting_meaning_class: LightTransportMeaningClassV1
  permitted_target_refs[1..64]
  transition_boundary:
    next_graph_generation | next_view_family | reload_required
  user_visible_explanation_ref: exact DiagnosticExplanationRefV1
```

同じTriggerに複数stepがある場合はLadderの明示順だけを使う。`to_technique_ref`がdisabledなら`resulting_meaning_class=unavailable`、non-disabledなら`unavailable`を禁止する。`qualified`から`degraded`への遷移をtelemetryだけに隠さず、Plan generation、reason、meaning classを変える。同一frameへfallback passを追加せず、宣言されたboundaryで新Plan／Graph generationを作る。

最低限、次を別diagnosticにする。

- Profile／Target／Source revision mismatch
- channel missing／duplicate、Technique branch invalid、inactive payload non-empty
- unsupported Target、Qualification Binding missing／stale／revoked
- representation incomplete、wrong generation、coverage gap
- history invalid、warm-up incomplete、denoise input missing
- budget denied、Provider unavailable／fault、device generation changed
- fallback ladder cycle、meaning class不許可、no qualified fallback
- Shadow Graph cycle／semantic mismatch、Project Technique invalid

fault時に近いProfile、低いTarget、別Provider、現在residentな表現、既定Shadow、screen-space resultへsilent fallbackしない。fallback不能なら該当channelをfaultとし、Profileが許す明示的`disabled` step以外でoutputを捏造しない。

## 10. AI／Editor理解境界

AI／EditorへはSource requirement、各channelのselected／rejected Technique、Target support state、representation requirement、history／denoise intent、fallback ladder、meaning差、budget estimate、Diagnostic、Qualification stateをbounded projectionで示す。native resource、shader table、descriptor、GPU address、raw capture、Provider secretを公開しない。

`create profile`、`select technique`、`preview`、`compare reference`、`explain fallback`はplanned semantic vocabularyであり、current MCD Operation、Service、Tool、Provider surfaceではない。完全なOperation／Authority／Validator／Receipt／publication closureが別途承認されるまで、AIはaction名からIDを生成またはdispatchしない。

AIは`cinematic`、`AAA`、`photoreal`、`real-time ray tracing`等の自由語からTechniqueまたはTarget supportを自動選択しない。channel、dynamic participation、Target、quality／performance intent、meaning-degraded許容を明示し、不足が結果を変える場合だけassumption／questionとして返す。

## 11. Security、capacity、Qualification

Source、Technique metadata、representation metadata、shader／model artifact、history bounds、array count、size／offsetはuntrusted inputとして検証する。Project Shaderとmodel artifactは署名、Target、Toolchain、Understanding／Qualification closureを必要とし、runtime download／compile／training、arbitrary operator、network access、native GPU API exposureを禁止する。共通authorizationは[AI Security／Approval](../01-governance/ai-security-approval.md)、Evidence signature／freshness／revocationは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、共有budget／measurementは[Performance／Capacity](../04-runtime/performance-capacity.md)だけが所有する。

Qualification subjectは少なくとも`{profile_ref, channel, technique_ref, representation_requirement_ref, Target Profile ref, Renderer Capability Signature ref, Toolchain／Provider lock refs, fixture set ref}`へ閉じる。一つのReceiptで別channel、Technique、Target、Provider generation、reference／preview／runtime roleをsupportしない。

fixtureは次を独立に含む。

1. diffuse／specular／shadow／reference各channelのpositive semantic fixture
2. baked、probe、screen-space、software-ray、hardware-ray、hybrid、path roleのapplicable matrix
3. static／dynamic／skinned／Terrain／Foliage、emissive、translucent、volumetric coverage
4. camera cut、teleport、extent／projection／world origin／representation generation変更
5. missing／stale artifact、Target mismatch、Provider loss、device loss、budget denial
6. fallbackごとのmeaning class、visual差、temporal stability、no-silent-change
7. malformed graph、cycle、oversized count、unknown semantic、wrong owner／hash／Receipt
8. reference outputへのnumeric／visual／temporal comparisonとblind content-quality review

`LightTransportQualificationBindingV1`は一つのReceipt-free subjectと1～64件のsigned Receipt refへ閉じ、Root外Activation projectionだけがBindingを参照する。ReceiptまたはBindingをProfile／Technique／Plan content hashへ戻さない。

## 12. 公式資料から採用する構造と限界

- [Unreal Engine: Supported Features by Rendering Path](https://dev.epicgames.com/documentation/en-us/unreal-engine/supported-features-by-rendering-path-for-desktop-with-unreal-engine)はLumen、Nanite、Virtual Shadow Maps、TSR、Path Tracer等のsupportがrendering pathで異なることを示す。本書は一つのadvanced flagではなくchannel／Technique／Target matrixを採用する。
- [Unreal Engine: Hardware Ray Tracing](https://dev.epicgames.com/documentation/en-us/unreal-engine/hardware-ray-tracing-in-unreal-engine)と[Lumen GI and Reflections](https://dev.epicgames.com/documentation/en-us/unreal-engine/lumen-global-illumination-and-reflections-in-unreal-engine)はhardware ray、software path、GI、reflection、shadowの構成とcost／fallbackが一様でないことを示す。本書はchannel別選択を採用する。
- [Unreal Engine: Path Tracer](https://dev.epicgames.com/documentation/en-us/unreal-engine/path-tracer-in-unreal-engine)はreal-time pathとcodeを共有しても用途、制約、fallback representationが異なることを示す。本書はreference／preview／runtime candidateのReceiptを分離する。
- [Godot: Renderers](https://docs.godotengine.org/en/stable/tutorials/rendering/renderers.html)と[Global illumination](https://docs.godotengine.org/en/stable/tutorials/3d/global_illumination/index.html)はrendererごとのfeature差と複数GI方式を示す。本書はunsupported／degradedを明示する。
- [Unity HDRP](https://docs.unity3d.com/ja/current/Manual/com.unity.render-pipelines.high-definition.html)はhigh-fidelity renderingを特定hardware／pipelineの能力として扱う。本書はProduct quality claimとTarget Technique supportを分離する。

外部Engineの名称、API、algorithm、marketing tier、既定値、release時期をMiraikanaiのcanonical IDまたは採用要件にしない。上記は責務分離、support matrix、fallback、qualificationの比較根拠だけである。

## 13. 明示的非目標と設計Closure

非目標は、別Rendererの新設、特定Provider／API／algorithmの選定、全Targetで同一Techniqueを要求すること、ray／path／neuralをProduct claimにすること、Light／Material／World／Terrain／LOD／Render Graph／Postのauthorityを本書へ統合すること、実装Task／順序／担当／工数／日程を定義することである。

本書の設計Closureは次をすべて満たす場合だけ成立する。

1. 四channelが独立Profile、Target support、fallback、Qualificationを持つ。
2. Light Source、Material、Environment、World、Terrain、LOD、geometry、residencyのauthorityが各Ownerに残る。
3. Render Graphへ渡るものがsemantic execution requestで、pass／resource／native objectをSourceへ逆流させない。
4. Shadow semantic Planは本書、Shadow authoringはLighting、Project Technique sourceはProject Shader、executionはRender Graphに一意である。
5. reference／preview／runtime candidateをReceiptとclaimで混同しない。
6. unsupported／unqualified／degraded／faultとmeaning差が機械判定できる。
7. 本書の存在を実装済み、Target対応、Product Activation、AAA／photoreal達成の根拠にしない。
