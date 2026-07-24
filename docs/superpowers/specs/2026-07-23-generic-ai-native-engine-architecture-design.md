# Generic AI-Native Engine Architecture Design

- 作成日: 2026-07-23
- 状態: ユーザー承認済み設計。Architecture正本へ反映中（2026-07-24契約DAG追補）
- 対象: Engine Core、Pack、Game Project、World／Runtime、Gameplay、AI Discovery、Product Qualification
- 非対象: Engine実装、Capability Activation、実機Qualification、Release／Shipping実行

## 1. 結論

Miraikanai Engineは、Shooterを共通基盤として一般化する設計を採らない。製品構造を次の4層へ固定する。

1. `Generic Engine Core`
2. `Reusable Feature Packs`
3. `Genre Packs`（任意）
4. `Game Projects`

`Reference Games`は第5のRuntime層ではない。各層を検証するために選定された通常のGame Projectであり、Production RuntimeからFixtureまたはReference Gameへの依存を禁止する。

AI Control PlaneはGameplay層ではなく横断Control Planeである。Editor内AI、first-party local inference、cloud Provider、外部MCP Client、CLIは、同じMiraikanai Contract Definition（MCD）、型付きOperation、Validation、Staging、Receiptを使用する。

Shooter-firstは実装順序として維持できるが、Core、Editor、AI、Project C++、Project Shader、Build、Test、Package、Network、Releaseの成立条件にShooter Packを置いてはならない。ProductのShooter／Platformer／Puzzle-Dialogue等のPhase Gateはbundled Reference Game coverageだけを評価するnonblocking trackとし、Generic EngineのRelease Gate、CX3 shipping、production-release bindingはGenre／Feature Pack未導入のCore／AI holdout、external／Source security boundary、Target lifecycle／Package、Toolchain、Risk closureだけから成立させる。

## 2. 層と依存規則

依存方向は利用側から提供側への一方向とする。

```text
Game Project
  -> optional Genre Pack
  -> Feature Pack
  -> Generic Engine Core
```

依存検査は文字列出現数ではなく、Product定義とFoundation compositionを入力に生成する次のclosed artifactだけを使う。このartifactはArchitecture Conformance専用で、Product Activation、Operation dispatch、Build成功、Shippingを単独で許可しない。

### 2.1 Owner、Source、Edgeのclosed schema

```text
ArchitectureDependencyArtifactRoleV1 =
  production | reference_game | qualification_fixture |
  cross_cutting_control_plane

ProductDefinitionRowRefV1
  active_product_definition_sha256: Sha256
  registry_manifest_entry:
    registry_id
    format_major: 1
    revision: positive safe integer
    registry_ref: ContentAddressedRefV1
    registry_sha256: Sha256
  row_kind: standard
  logical_id: StableId
  row_sha256: Sha256

ArchitectureDocumentRefV1
  document_id: StableId
  document_core_ref:
    exact ContentAddressedRefV1(type=ArchitectureDocumentCoreV1)
  document_core_sha256: Sha256
  lifecycle_head_ref:
    exact MirakanSignedRecordRefV1(
      purpose=document_lifecycle_transition)
  lifecycle_head_sha256: Sha256
  lifecycle_sequence: positive safe integer
  approved_document_sha256: Sha256

ArchitectureDocumentRegistryV1
  registry_id: architecture_document.registry.approved
  registry_version: positive uint32
  current_control_plane_baseline_binding:
    CurrentControlPlaneBaselineBindingV1
  entry_count: positive uint32
  entries[1..8192]: ArchitectureDocumentRefV1
  registry_content_hash: Sha256

ArchitectureDocumentRegistryRefV1
  registry_id: architecture_document.registry.approved
  registry_version: positive uint32
  registry_content_hash: Sha256

ArchitectureDependencyOwnerV1
  owner_binding_id: StableId
  owner_binding_version: positive uint32
  owner_source:
    product_work_package:
      work_package_row_ref: ProductDefinitionRowRefV1(
        registry=WorkPackageRegistryV1, row_kind=standard)
      owner_document_ref: ArchitectureDocumentRefV1
    | product_fixture:
      fixture_row_ref: ProductDefinitionRowRefV1(
        registry=FixtureRegistryV1, row_kind=standard)
      owner_document_ref: ArchitectureDocumentRefV1
    | component_qualification_fixture:
      provided_fixture_ref: ProvidedFixtureRefV1(
        kind=component_qualification_fixture)
    | installed_product_control_plane:
      composition_ref: InstalledProductOperationCompositionRefV1
    | foundation_contract_owner:
      owner_ref: OwnerIdentityLocalRefV1
  artifact_role: ArchitectureDependencyArtifactRoleV1
  production_owner_layer:
    core | feature_pack | genre_pack | game_project
    | canonical omission when artifact_role != production
  owner_binding_content_hash: Sha256

ArchitectureWorkPackageOwnerClassificationEntryV1
  work_package_row_ref: ProductDefinitionRowRefV1(
    registry=WorkPackageRegistryV1)
  artifact_role: ArchitectureDependencyArtifactRoleV1
  production_owner_layer:
    core | feature_pack | genre_pack | game_project
    | canonical omission when artifact_role != production

ArchitectureWorkPackageOwnerClassificationV1
  classification_version: positive uint32
  expected_work_package_count: uint32
  entries[1..8192]:
    ArchitectureWorkPackageOwnerClassificationEntryV1
  classification_content_hash: Sha256

ArchitectureDependencyOwnerRefV1
  owner_binding_id: StableId
  owner_binding_version: positive uint32
  owner_binding_content_hash: Sha256

ArchitectureDependencySourceKindV1 =
  product_required_capability | product_required_work_package |
  qualification_fixture | installed_product_composition_member

ArchitectureDependencySourceRecordV1
  source_id: StableId
  source_version: positive uint32
  source_kind: ArchitectureDependencySourceKindV1
  source:
    product_required_capability:
      consumer_work_package_ref: ProductDefinitionRowRefV1(
        registry=WorkPackageRegistryV1)
      required_capability_ref: ProductDefinitionRowRefV1(
        registry=CapabilityRegistryV1)
      provider_work_package_ref: ProductDefinitionRowRefV1(
        registry=WorkPackageRegistryV1)
      applicable_target_refs[1..5]: TargetProfileRefV1
    | product_required_work_package:
      consumer_work_package_ref: ProductDefinitionRowRefV1(
        registry=WorkPackageRegistryV1)
      provider_work_package_ref: ProductDefinitionRowRefV1(
        registry=WorkPackageRegistryV1)
    | qualification_fixture:
      consumer_work_package_ref: ProductDefinitionRowRefV1(
        registry=WorkPackageRegistryV1)
      provided_fixture_ref: ProvidedFixtureRefV1
    | installed_product_composition_member:
      composition_ref: InstalledProductOperationCompositionRefV1
      contribution_origin_ref: OwnerOperationContributionRefV1
      target_contract_owner_ref: OwnerIdentityLocalRefV1
      operation_local_ref_set_hash: Sha256
  source_content_hash: Sha256

ArchitectureDependencySourceRefV1
  source_id: StableId
  source_version: positive uint32
  source_content_hash: Sha256

ArchitectureDependencyEdgeClassV1 =
  production_dependency | qualification_input |
  installed_product_composition | cross_cutting_control_plane |
  documentation_reference

ArchitectureDependencyEdgeV1
  edge_id: StableId
  edge_version: positive uint32
  edge_class: ArchitectureDependencyEdgeClassV1
  from_owner_ref: ArchitectureDependencyOwnerRefV1
  to_owner_ref: ArchitectureDependencyOwnerRefV1
  subject_ref: ArchitectureDependencySourceRefV1
  required_for_runtime: bool
  edge_content_hash: Sha256

ArchitectureDependencySourceSetV1
  source_set_id: architecture_dependency.source_set.current
  source_set_version: positive uint32
  active_product_definition_closure_ref: ContentAddressedRefV1
  active_product_definition_sha256: Sha256
  architecture_document_registry_ref: ArchitectureDocumentRegistryRefV1
  foundation_definition_closure_ref: FoundationDefinitionClosureRefV1
  generic_core_baseline_ref: GenericCoreOperationBaselineRefV1
  installed_product_composition_ref:
    InstalledProductOperationCompositionRefV1
  work_package_owner_classification:
    ArchitectureWorkPackageOwnerClassificationV1
  source_count: uint32
  sources[1..16384]: ArchitectureDependencySourceRecordV1
  source_set_content_hash: Sha256

ArchitectureDependencySourceSetRefV1
  source_set_id: architecture_dependency.source_set.current
  source_set_version: positive uint32
  source_set_content_hash: Sha256

ArchitectureDependencyEdgeRegistryV1
  registry_id: architecture_dependency.edge.registry.current
  registry_version: positive uint32
  source_set_ref: ArchitectureDependencySourceSetRefV1
  owner_count: uint32
  owners[1..8192]: ArchitectureDependencyOwnerV1
  edge_count: uint32
  entries[1..16384]: ArchitectureDependencyEdgeV1
  registry_content_hash: Sha256

ArchitectureDependencyEdgeRegistryRefV1
  registry_id: architecture_dependency.edge.registry.current
  registry_version: positive uint32
  registry_content_hash: Sha256

ArchitectureDependencyDerivedCountsV1
  owner_count: uint32
  source_count: uint32
  product_required_capability_source_count: uint32
  product_required_work_package_source_count: uint32
  qualification_fixture_source_count: uint32
  installed_product_composition_source_count: uint32
  edge_count: uint32
  production_dependency_edge_count: uint32
  qualification_input_edge_count: uint32
  installed_product_composition_edge_count: uint32
  cross_cutting_control_plane_edge_count: uint32
  documentation_reference_edge_count: uint32
  required_for_runtime_edge_count: uint32
  production_graph_node_count: uint32
  production_graph_unique_pair_count: uint32
  production_graph_cycle_count: uint32
  core_to_shooter_production_reachability_count: uint32

ArchitectureDependencyConformanceReportPayloadV1
  report_id: StableId
  evaluated_candidate_ref: PreparedCandidateRefV1
  evaluated_candidate_root_sha256: Sha256
  source_set_ref: ArchitectureDependencySourceSetRefV1
  edge_registry_ref: ArchitectureDependencyEdgeRegistryRefV1
  active_product_definition_sha256: Sha256
  foundation_definition_closure_ref: FoundationDefinitionClosureRefV1
  generic_core_baseline_ref: GenericCoreOperationBaselineRefV1
  installed_product_composition_ref:
    InstalledProductOperationCompositionRefV1
  derived_counts: ArchitectureDependencyDerivedCountsV1
  result:
    {kind: pass, violation_refs: []}
    | {kind: fail, violation_refs[1..1024]}
  qualifier_subject_ref
  qualifier_role_ref =
    role.architecture-dependency-conformance-validator
  issued_at, expires_at, revocation_snapshot_ref

ArchitectureDependencyConformanceReportV1
  payload: ArchitectureDependencyConformanceReportPayloadV1
  signed_record:
    exact MirakanSignedRecordV1(
      purpose=architecture_dependency_conformance)

ArchitectureDependencyConformanceReportRefV1
  exact MirakanSignedRecordRefV1(
    purpose=architecture_dependency_conformance)
```

`ProductDefinitionRowRefV1`は`active_product_definition_sha256`、Registry manifestのexact `{registry_id, format_major, revision, registry_ref, registry_sha256}`、`row_kind`、logical ID、`row_sha256=SHA-256(JCS(exact inline row))`を全Field保存する。`ArchitectureDocumentRefV1`はControl Plane設計が所有する`ArchitectureDocumentCoreV1`とcurrent `DocumentLifecycleRecordV1` headを参照し、Coreの`document_id`、完成UTF-8 file bytes／Git blob／metadata projection、`document_sha256`、Lifecycleの`document_core_ref/hash`を全て解決する。Lifecycle headは同じ`document_id`、同じCore、current sequence、`transition_kind=approve`、`from_state=review`、`to_state=approved`で、独立R4 Document approverのcurrent assignment、singleton purpose `document_lifecycle_transition` Key、署名、期限、revocationが有効でなければならない。`approved_document_sha256`はCoreの`document_sha256`、`lifecycle_head_sha256`は完成signed wrapper bytesとbyte equalityにする。IDだけ、path、display name、`latest`、owner文書prefix、本文hashだけからのApproval補完を許可しない。`owner_source=product_work_package`の`owner_document_ref.document_id`はWork Package rowの`owner_document_id`と一致し、Fixture branchもProductまたはPack側のtyped ownerと一致しなければならない。

`ArchitectureDocumentRegistryV1.entries[]`は、同じcurrent `CurrentControlPlaneBaselineBindingV1`が束縛するActive Product Definitionの全Work Package／Product Fixture rowの`owner_document_id`、九件のcomponent qualification fixtureが束縛するPack owner文書、Foundation Definition Closureの全Owner authority document、およびInstalled Product三Originが解決するOwner authority documentの重複を除いたexact unionである。各IDをcurrent approved Core／Lifecycle headへ一件だけ解決し、entryを`document_id` UTF-8 byte順へstrict sortする。missing、extra、duplicate、未承認、superseded、stale head、別Core approvalを拒否する。`registry_content_hash`はASCII domain `MIRAKAN_ARCHITECTURE_DOCUMENT_REGISTRY_V1`と、自己Fieldを除くclosed recordのRFC 8785 JCS bytesを`uint32_be` length framingして計算する。Registryはcompleted source treeから決定論的に再生成するunsigned derived indexであり、Lifecycle署名を置換せず、`ArchitectureDependencySourceSetV1.architecture_document_registry_ref`だけが同version／hashを参照する。

`ArchitectureWorkPackageOwnerClassificationV1`は`classification_version`、`expected_work_package_count`、全Work Packageのexact row ref、`artifact_role`、conditional `production_owner_layer`、`classification_content_hash`を持つclosed recordである。materialized bytesは75件すべてを明示行として保存し、default layerを持たない。本書では現行75行を次の互いに素な集合で短記する。generatorはActive Product DefinitionのWork Package集合とunionをset equalityにし、Core集合を現在closure内の差集合として61行へ展開してからhashする。後続DefinitionでWork Packageが一件でも増減した場合、旧分類を適用せず新しい全量分類とSource Set versionを要求する。

| artifact role / production layer | current exact Work Package集合 | count |
|---|---|---:|
| `production / feature_pack` | `wp.gameplay.reusable-features-c1` | 1 |
| `production / genre_pack` | `wp.domain.shooter-2d`; `wp.domain.shooter-3d` | 2 |
| `reference_game / omitted` | `wp.domain.platformer`; `wp.domain.puzzle-dialogue` | 2 |
| `cross_cutting_control_plane / omitted` | `wp.architecture.control-plane`; `wp.product.editor-runtime-windows`; `wp.product.ai-authoring-mvp-a`; `wp.product.external-agent`; `wp.product.project-source-activation`; `wp.product.general-coverage-2d`; `wp.product.general-coverage-3d`; `wp.product.production-release-binding`; `wp.product.runtime-generation` | 9 |
| `production / core` | current `WorkPackageRegistryV1` 75行から上記14行を除くexact差集合 | 61 |

既存V1 logical IDの`wp.domain.*`／`capability.domain.*`に含まれる`domain` tokenは互換性維持用のopaque ID segmentであり、`Domain Pack`という製品層またはCore依存を表さない。authoritative分類は本節の`artifact_role`／`production_owner_layer`と`ArchitectureDependencyClassificationRegistryV1`だけから決める。既存IDはhash／registry参照を壊す一括renameを行わず、新規IDでは承認済みnamespace規則により`feature`／`genre`／`reference_game`等の役割を明示する。AI、Compiler、UIはID substringからlayer、Authority、Target、required dependencyを推測しない。

Reference Gameを`production_owner_layer=game_project`へ偽装せず`artifact_role=reference_game`としてlayerを省略する。将来の通常Game Project production artifactだけが`artifact_role=production, production_owner_layer=game_project`を持てる。Profileは独立PackまたはOwnerを生成せず、所有Packのversion／hashに含まれる構成単位とする。

### 2.2 current Source SetとInstalled Productの分離

現行Source Set version 1は次のexact集合である。Product側三集合は[Product Plan §11](../../architecture/00-product/product-plan.md#11-product-execution-registries)の同一`ActiveProductDefinitionClosureV1`から読み、別revisionまたはMarkdown断片を混ぜない。`provided_fixture_refs[]`は15 Product Fixtureと9 component qualification fixtureのtyped unionへ解決し、裸IDを保存しない。

| source kind | canonical derivation | current rows |
|---|---|---:|
| `product_required_capability` | 75 Work Packageの全`required_capability_refs[]`をCapability rowへ解決し、同rowの`owner_work_package_ref`をproviderにする | 166 |
| `product_required_work_package` | 75 Work Packageの全`requires_work_package_refs[]`をexact Work Package rowへ解決する | 171 |
| `qualification_fixture` | 75 Work Packageの全`provided_fixture_refs[]`を`ProvidedFixtureRefV1`へ解決する | 176 |
| `installed_product_composition_member` | current compositionの三Origin contributionを一Origin一sourceで投影する | 3 |
| **合計** | 上記四集合のdisjoint union | **516** |

Installed Product三sourceは次だけである。`operation_local_ref_set_hash`は各Originのstrict sorted `operation_local_refs[]`から計算し、件数だけ、owner IDだけ、Operation prefixだけが一致する代用品を拒否する。

| contribution Origin | target contract owner | layer | Operation count |
|---|---|---|---:|
| `{contribution.owner.core.project_state.runtime_entry_bootstrap,1,73472e663836be538cc609d20821c491a4e1080eb0d05186d1335efb8d727d8a}` | `{owner.core.project_state,1,8629bba70acf6c53968cdbe97a9ccc4043adacb8eef29c9e4303df091f6ee428}` | `core` | 6 |
| `{contribution.owner.core.world.generated_stable_id_allocation,1,58b56dfefe98655ddb6607f00bafd04bfaa74dd25da1f473a1a1bd6991fd940f}` | `{owner.core.world,1,02e558a2c554d54711631421e22d449b92c4339c74d1159157620b5e37dc6625}` | `core` | 1 |
| `{contribution.owner.genre.shooter.target_provider_binding,1,fdf61245348480fb746386f84fd2e56fba52b236922fad7289aa9efe8429732f}` | `{owner.genre.shooter,1,a4c910efb5f04c8d8e5f2980e42e3889dbe73ecbd3f7976e39b4ce072ec01bc4}` | `genre_pack` | 3 |

Source Setは`generic_core_baseline_ref={operation_baseline.generic_engine_core.bootstrap,1,14e723260d7c745ccc79a30060220f01898571bb5aa04d56c162db0a4777a11c}`と`installed_product_composition_ref={operation_composition.product.current,1,722df325f71678faf4b6a1ae7a683e2181529087b37e93c1f2781e7b701e0a05}`を別Fieldでpinする。三Origin edgeは後者をsubjectにする`edge_class=installed_product_composition`だけであり、Generic Core baseline、Owner contribution、Contract Set、Installed compositionの完成bytesへFieldを追加しない。したがって本Registry導入はbaseline六entry／hash、Installed Product十entry／hashを変更せず、Shooter三OperationをCore production dependencyへ変換しない。

Owner集合はWork Package 75、distinct Fixture 24、Installed Product Control Plane 1、上表のFoundation contract owner 3のexact 103件である。同じArchitecture文書が複数Work Packageを所有しても、依存graph nodeはexact Work Package rowへ束縛した別Owner bindingとする。文書単位の一括分類で`mirakan.arch.gameplay-programming-model`配下のCoreとReference Gameを同一layerへ潰さない。

### 2.3 canonical derivationとhash

全closed recordはRFC 8785 JCSを使い、stringをUnicode NFC、SHA-256をprefixなしlowercase hex 64文字、positive integerをJSON safe integerへ固定し、unknown Field、duplicate key、non-canonical numberを拒否する。`Canonical(x)`をself-hash Fieldだけ除いたclosed recordのJCS bytes、`H(domain,x)=SHA-256(ASCII domain || uint32_be(len(Canonical(x))) || Canonical(x))`とする。

- `owner_binding_content_hash = H("MIRAKAN_ARCHITECTURE_DEPENDENCY_OWNER_V1", owner)`
- `classification_content_hash = H("MIRAKAN_ARCHITECTURE_WP_OWNER_CLASSIFICATION_V1", classification)`
- `source_content_hash = H("MIRAKAN_ARCHITECTURE_DEPENDENCY_SOURCE_V1", source)`
- `edge_content_hash = H("MIRAKAN_ARCHITECTURE_DEPENDENCY_EDGE_V1", edge)`
- `source_set_content_hash = H("MIRAKAN_ARCHITECTURE_DEPENDENCY_SOURCE_SET_V1", source_set)`
- `registry_content_hash = H("MIRAKAN_ARCHITECTURE_DEPENDENCY_EDGE_REGISTRY_V1", registry)`

Owner、Source、EdgeのStable IDはそれぞれ`*_binding_content_hash`または`source_content_hash`／`edge_content_hash`を直接再利用せず、logical keyだけを`MIRAKAN_ARCHITECTURE_DEPENDENCY_{OWNER|SOURCE|EDGE}_KEY_V1` domainで同じlength framingにして得たhashから`urn:mirakan:architecture-dependency:{owner|source|edge}:sha256:<lowercase-hex>`として導出する。logical keyはOwnerならtagged `owner_source` identity、Sourceならsource kindと全logical row identity、Edgeなら`edge_class`、二Owner Ref、Source Ref、`required_for_runtime`である。IDをkey preimageまたはcontent hash preimageへ循環させない。

`owners[]`は`owner_binding_id`／version／hash、`sources[]`は`source_id`／version／hash、`entries[]`は`edge_id`／version／hashのunsigned UTF-8 byte順へstrict sortし、同じlogical keyのduplicateまたは同ID／version別hashを拒否する。Source SetとRegistryのcountは実配列長と一致しなければならない。Product Closure、Architecture Document Registry、Foundation Closure、baselineまたはcompositionのRef／hashが一件でもstaleなら部分Registryを生成せず、last completed conformance artifactをcurrent Product Definitionの証拠として流用しない。

### 2.4 production dependency reachability compiler

Compilerは次の順で一度だけ分類する。

1. 75 Work Package、distinct Fixture 24、Installed Product Control Plane一件、Foundation contract owner三件からOwner exact 103件をmaterializeする。
2. 四source kindのset equalityと件数`166 + 171 + 176 + 3 = 516`を検査し、各Sourceをexact一Edgeへ投影する。`product_required_capability`と`product_required_work_package`はconsumer WP Owner→provider WP Owner、`qualification_fixture`はconsumer WP Owner→Fixture Owner、`installed_product_composition_member`はInstalled Product Control Plane Owner→target Contract Ownerに固定し、caller指定endpointを受理しない。
3. `qualification_fixture`は常に`qualification_input`、`installed_product_composition_member`は常に`installed_product_composition`にする。
4. Product dependencyの両endpointのどちらかが`cross_cutting_control_plane`なら`cross_cutting_control_plane`にする。
5. Product dependencyの両endpointのどちらかが`reference_game | qualification_fixture`なら`qualification_input`にする。
6. 残る`product_required_capability | product_required_work_package`だけを`production_dependency`にする。前者だけはconsumerの全TargetでCapability bindingが`required`なら`required_for_runtime=true`、後者はbuild／qualification順序なのでfalseにする。他classは必ずfalseである。
7. `production_dependency`だけからconsumer→providerのdirected multigraphを作り、reachabilityでは同じOwner pairのparallel edgeを一pairへ正規化する。別sourceをRegistryから削除してdeduplicateしない。

current expected projectionは次である。件数差はwarningでなくSource／Compiler failureである。

| derived value | exact current value |
|---|---:|
| Owner records | 103 |
| Edge records | 516 |
| `production_dependency` | 248 |
| `qualification_input` | 193 |
| `cross_cutting_control_plane` | 72 |
| `installed_product_composition` | 3 |
| `documentation_reference` | 0 |
| `required_for_runtime=true` | 125 |
| production graph nodes / unique directed pairs | 64 / 126 |
| production graph cycles | 0 |
| Core root→Shooter production reachability | 0 |

次を機械検査する。

- Coreの`production_dependency`はCoreだけへ到達し、Feature Pack、Genre Pack、Game Project、Reference Game、Fixture、cross-cutting Control Planeへ到達しない。
- Feature Packの`production_dependency`はCoreおよび別Feature Packだけを参照でき、Feature subgraphを含むproduction graphはDAGでなければならない。
- Genre PackはCoreとFeature Packだけを参照できる。Genre Pack同士を直接または推移依存させない。
- Game ProjectはCore、Feature、任意GenreをProject側で合成できるが、複数Genre間のedgeを生成しない。
- Reference Game／Fixtureは`qualification_input`として各層を利用できるが、Production artifactまたは`production_dependency`へ逆流しない。
- `installed_product_composition`はProductが選択したCore／Feature／Genre／Project contributionを列挙できるが、列挙をCore ownershipまたはCore→Pack production dependencyへ変換しない。
- `cross_cutting_control_plane`は複数Ownerを監査・構成できるが、Gameplay／Runtime依存またはproduction ownerを合成しない。
- `documentation_reference`は将来専用Source kindをschema revisionで追加するまでcurrent 0件であり、説明文から自動生成しない。

Generic Core gateのrootは`production_owner_layer=core`の全current Work Package Owner 61件と、Generic Core baseline六entryが解決するexact Contract Ownerである。Baseline Contract Ownerは同じFoundation ClosureのOwner Registryへ解決し、`owner_layer=core`かつauthority documentがCore Work Package bindingのapproved文書と一致することを検証する。この照合はproduction edgeを暗黙生成せず、root集合のidentity検証だけを行う。このroot集合からShooter二WP Ownerまたは`owner.genre.shooter`への`production_dependency`到達数はexact 0でなければならない。Installed Product三edgeをGraphへ混入させ、Core→Shooter到達を作る実装も、逆に三edgeをRegistryから隠す実装も失敗である。

`ArchitectureDependencyConformanceReportV1`はcompleted `PreparedCandidateRefV1`に対するArchitecture compilerの署名済み結果である。`source_set_ref`、`edge_registry_ref`、Active Product Definition、Foundation Closure、Generic Core baseline、Installed Product compositionは同じCandidateから生成した値へbyte equalityにし、`derived_counts`はSource Set／Registryの実配列と上表を再計算する。`result.kind=pass`はOwner 103、Source／Edge 516、Source内訳166／171／176／3、class内訳248／193／3／72／0、runtime-required 125、production graph 64 node／126 unique pair／cycle 0、Core→Shooter reachability 0、全negative fixture成功、`violation_refs=[]`の場合だけ許す。一件でも未評価、stale、count-only自己申告、別Candidate、別Registry、別Product／Foundation closureならpassを発行しない。

`report_id`は同Fieldを除く完成payloadのRFC 8785 JCS SHA-256から`urn:mirakan:architecture-dependency-conformance:sha256:<lowercase-hex>`として導出する。`role.architecture-dependency-conformance-validator`へのcurrent assignmentを持ち、対象Candidateのauthor／publisher／Pack ownerから独立したsubjectだけがsingleton purpose `architecture_dependency_conformance` Keyで署名できる。signed wrapperのsubject hash、signer subject／Role、`issued_at`、revocation snapshotをpayloadへbyte-exact一致させ、`issued_at < expires_at`、current利用時の未失効を要求する。Reportはappend-only Conformance Record Storeへ完成wrapper単位で保存し、Refは同wrapperを一意に解決する。

Productの`requirement.product.core-pack-independence`、`fixture.product.genreless-core-2d`、Critical Path Wave 0は、同じCandidate／Product／Foundation／baseline／compositionを束縛したfresh `result.kind=pass` Report RefをEvidenceへ必須入力として持つ。Reportは依存隔離のEvidenceであってOperation、Activation、Approval、Commit、ReleaseのAuthorityではない。Report Refだけでdispatchする、ReportをActive Product Definition／Contract Set／MCDへ注入する、旧Candidateのpassを新Candidateへ流用する実装を拒否する。

### 2.5 negative fixturesとauthority境界

次のfixtureはArchitecture Conformance artifact専用で、Product `FixtureRegistryV1`、Active Product Definition、Contract Setへ追加しない。

| fixture ID | 一原因mutation | expected result |
|---|---|---|
| `fixture.architecture_dependency.unclassified_work_package` | 76件目のWPをOwner分類なしで追加 | Source Set全体を拒否 |
| `fixture.architecture_dependency.missing_or_extra_source` | 516 sourceから一件削除、または表外sourceを一件追加 | set equality failure |
| `fixture.architecture_dependency.owner_document_spoof` | path／prefixだけ一致する別Owner文書hashへ置換 | Owner resolution failure |
| `fixture.architecture_dependency.fixture_as_production` | `qualification_fixture`を`production_dependency`へ変更 | edge classification failure |
| `fixture.architecture_dependency.ordering_as_runtime` | requires-WP edgeを`required_for_runtime=true`へ変更 | runtime flag failure |
| `fixture.architecture_dependency.core_to_feature` | Core WPからFeature Ownerへのdirectまたはtransitive production edgeを追加 | reachability failure |
| `fixture.architecture_dependency.feature_to_genre` | Feature WPからShooter Ownerへのproduction edgeを追加 | layer rule failure |
| `fixture.architecture_dependency.production_cycle` | production graphへ一つのback edgeを追加 | DAG failure |
| `fixture.architecture_dependency.installed_as_production` | Shooter Origin edgeを`production_dependency`へ変更 | class／Core isolation failure |
| `fixture.architecture_dependency.composition_substitution` | composition Ref、Origin hash、またはOperation set hashを一つ変更 | Source resolution failure |
| `fixture.architecture_dependency.baseline_substitution` | baseline Refを別六entryまたは別hashへ変更 | pinned-root failure |
| `fixture.architecture_dependency.target_scope_mismatch` | required Capabilityをconsumerの一Targetで`optional`または`excluded`へ変更 | Source derivation failure |
| `fixture.architecture_dependency.stale_product_closure` | WPとCapabilityを別Active Definition revisionから混在 | closure equality failure |
| `fixture.architecture_dependency.noncanonical_order` | Owner／Source／Edge配列を一箇所だけ入替え | canonical encoding failure |
| `fixture.architecture_dependency.registry_as_authority` | Edge Registry RefだけでActivationまたはdispatchを許可 | authority boundary failure |
| `fixture.architecture_dependency.product_bundle_injection` | Edge Registryを`ActiveProductDefinitionBundleV1.registry_manifest[]`へ追加 | bundle schema failure |

`ArchitectureDependencyEdgeRegistryV1`と`ArchitectureDependencyConformanceReportV1`は`ActiveProductDefinitionBundleV1`の14 Registry、Future Portfolioの3 Registry、Contract Set、Owner Registry、MCDへ追加しないcross-cutting derived artifact／signed Evidenceである。入力Product Closureを参照しても逆向きにProduct hash preimageへ戻さず、現行Active Product Registry件数、Activation binding 293行、Generic Core baseline六entry／hash、Installed Product十entry／hashを変更しない。Control Plane materialization前のcurrent completed Registry Refとsigned Conformance Reportは存在しないため、文書compilerのsource-level expected projectionだけでEngine実装済みまたはOperation operationalと表示しない。

## 3. Pack契約

`DomainPackManifest`を廃止し、正規型を`PackManifestV1`とする。`pack_kind`は`feature | genre`のclosed enumとする。

```text
PackManifestV1
  pack_id
  pack_version
  pack_kind: feature | genre
  content_hash
  minimum_engine_contract_ref
  supported_target_profile_refs[]
  required_capability_refs[]
  required_feature_pack_refs[]
  provided_capability_refs[]
  public_contract_refs[]
  component_schema_refs[]
  game_system_spec_refs[]
  authoring_operation_refs[]: exact MCD operation ref/version/set root
  runtime_port_refs[]
  configuration_profile_refs[]
  composition_recipe_refs[]
  source_template_refs[]
  validator_refs[]: exact validator ID/version/content hash
  test_scenario_refs[]: exact fixture ID/version/content hash
  example_refs[]
  counterexample_refs[]
  ai_vocabulary_refs[]
  ai_planning_recipe_refs[]
  performance_profile_refs[]
  migration_step_refs[]: exact migration ID/version/content hash
  migration_contribution_refs[]:
    exact registry/contribution ID/version/content hash
  license_ref
  provenance_ref
```

Pack全Recipeに共通するFeature依存だけを`PackManifestV1.required_feature_pack_refs[]`へ置く。選択したcompositionだけが必要とする条件依存は次の正規型へ置く。

```text
CompositionRecipeV1
  recipe_id
  recipe_version
  recipe_hash
  owner_pack_local_identity: exact PackLocalIdentityV1
  required_capability_refs[]
  required_feature_pack_refs[]
  configuration_profile_refs[]
  game_spec_template_refs[]
  action_role_set_refs[]
  source_template_refs[]
  validator_refs[]
  fallback_recipe_ref: CompositionRecipeRef | null

PackRecipeQualificationSubjectV1
  qualification_id
  qualification_version: positive uint32
  owner_pack_ref: exact PackContractRefV1
  recipe_ref/hash: exact CompositionRecipeV1
  target_profile_refs[1..64]
  fixture_refs[1..256]:
    exact {fixture_id, fixture_version, fixture_content_hash}
  input_closure_hash
  result: pass | fail
  qualification_subject_hash: SHA-256

PackRecipeQualificationReceiptV1
  subject: PackRecipeQualificationSubjectV1
  signed_record:
    exact MirakanSignedRecordV1(purpose=pack_recipe_qualification)

PackRecipeQualificationReceiptRefV1
  qualification_id
  qualification_version: positive uint32
  qualification_subject_hash: SHA-256
  signed_record_hash: SHA-256

PackRecipeActivationBindingRefV1
  activation_binding_id
  activation_binding_version: positive uint32
  activation_binding_hash: SHA-256

PackRecipeActivationBindingV1
  activation_binding_id
  activation_binding_version: positive uint32
  recipe_ref/hash: exact receipt-free CompositionRecipeV1
  qualification_receipt_ref: PackRecipeQualificationReceiptRefV1
  activation_binding_hash: SHA-256

PackRecipeActivationProjectionV1
  projection_id
  projection_version: positive uint32
  selected_recipe_ref/hash: exact receipt-free CompositionRecipeV1
  recipe_activation_binding_ref: PackRecipeActivationBindingRefV1
  dependency_closure_ref/hash: exact RecipeDependencyClosureV1
  projection_hash: SHA-256
```

`recipe_hash`は自己Fieldを除くReceipt-free canonical recordのSHA-256で、所有Packの`content_hash`へ含める。`owner_pack_local_identity`は所有Manifestの`{pack_id, pack_version}`とbyte equalityにし、Pack content hash、`PackContractRefV1`、Qualification Receipt／BindingをRecipe hash preimageへ含めない。生成順は`Pack local identity → receipt-free Recipe／recipe hash → Pack Manifest content hash → PackContractRefV1／Recipe ref → Qualification Subject → signed Receipt → root外Activation Binding → Project-owned Activation projection`の一方向だけである。Subjectのowner Pack ID／versionはRecipe local identity、owner Pack content hashは当該Recipeをinventoryに持つ完成Manifest、BindingのRecipe pairはSubjectへbyte equalityにする。Receipt／Binding／ProjectionをRecipe、Profile、Manifest、Pack content hashへ戻す実装を拒否する。

選択Recipeのeffective closureはManifest共通Feature、Recipe required Feature、両者の推移Feature DAGの和集合である。receipt-free canonical Recipe／owner Pack／resolved Pack version＋hashから`closure_hash`を生成し、Preview、Project ChangeSet、Cook、Qualification、Save／Replayへ伝播する。Production applyはroot外`PackRecipeActivationBindingV1`からReceiptのsubject／owner／Recipe／Target／result／freshnessを検証し、Fixture bodyを解決しない。未選択Recipeの条件依存はinstall closureへ追加しない。missing、version／hash conflict、Target不適合、unqualified dependency、fallback cycleはRecipe applyを原子的に拒否し、last-valid Recipe activation、Project revision、registry、closure hash、Artifactを維持する。

`PackManifestV1`はPack payload内のOperation、migration step／contribution、Validator、fixtureを全件列挙する。Manifest Operation集合、Pack ownerのactive MCD Operation集合、Trusted Service allowlist contributionはset equalityである。`validator_refs[]`と`test_scenario_refs[]`はPack内で利用可能なidentity inventoryであり、全Recipe共通の実行gateではない。Recipe apply／qualificationで実行するのは選択済み`CompositionRecipeV1.validator_refs[]`と、root外`PackRecipeActivationBindingV1`が指すexact signed Qualification Receiptだけである。Production Source／Recipe／Registry／Runtime PackageはFixture bodyを解決せず、別owner-typed Qualification Subjectのsigned Receiptだけをsubject／owner／Target／result／freshnessで検証する。

Validatorは異種集合を一つにしない。Manifest Validator inventoryはOwner Validator Registry subsetとValidator ID／version／content hashでset equalityにし、各Operationはclosure ValidatorのDiagnostic union、`reachable_error_refs[]`、Operation `errors[]`をDiagnostic ID／code／version／content hashで別途set equalityにする。

Feature Packは再利用可能なPublic Contract、Schema、Validator、Runtime Port、AI vocabulary、reference implementation、contract fixtureを提供する。Genre PackはFeature Packを組み合わせるcomposition recipe、Genre vocabulary、GameSpec template、Profile、reference scenarioを提供する。Genre Packは新しい汎用Core契約を作らない。

Shooter Packは`pack_kind=genre`とし、次のFeature Packを必要に応じて構成する。

- Combat: Damage、Vital、Faction
- Ranged Combat: Weapon、Shot、Projectile
- Encounter／Spawn
- Scoring
- Pickup／Grant
- Interaction
- Character Locomotion
- Path Following
- Scenario／Stage Flow（有限Gameplay用Recipeだけの条件依存）

`Ready | Playing | Paused | Result`、Shooter Action Role、Shooter固有Camera／Audio／LOD ProfileはShooter Genre Packが所有する。

Shooter manifestの全Recipe共通依存はRanged CombatとそのFeature DAGだけとする。Encounter、Scoring、Pickup、Interaction、Character Locomotion、Path Following、Scenario／Stage、Perceptionは、それらを使用するRecipeの`required_feature_pack_refs[]`／`required_capability_refs[]`へ置く。AI敵、移動、Score、Pickup、finite Stageを持たないminimal Shooter recipeをvalidとし、既存2D／TPS reference Recipeは従来のeffective closureとPerception bindingを維持する。

minimal target-practiceは既存`genre.shooter.top_down_2d | genre.shooter.third_person_3d`のexact 1件とPack-owned immutable `ShooterTargetProviderTemplateV1`を選ぶ。Production bindingはowner-typed `ShooterTargetProviderBindingDocumentV1`、選択正本は`ShooterTargetProviderSelectionDocumentV1`、qualification fixtureはProject Documentではない`ShooterTargetProviderFixtureBindingRecordV1`として分離する。Binding Registryはtrusted Document index内のBinding＋Selection Sourceからreload時に決定的に再materializeするDerived closureで、Registry直接writeやRegistry→Source逆投影を禁止する。Compile／Save／ReplayはSelection Document ref／hash、Binding Document ref／hash、RegistryRef、selected record set hashを束縛する。create／update／selectは三つの完全な`McdOperationContractV1`、named typed input／Result／Prepared semantic Receipt payload／canonical `PublishedDomainReceiptV2` wrapper、exact `MutationAuthorizationBindingV2`、pure pre／post policy、Service／rate policy、17 exact Diagnostic実配列、validator reachable error set equality、request hash／idempotencyを持つ。Shooter ManifestのValidator inventoryとShooter owner Registry subsetはcomposition／perception二件と三Operationのsemantics／postcondition六件のexact八件である。publicationは[Executable Contracts §8](../../architecture/02-foundation/executable-contracts.md#8-operation定義)をexact reuseし、private durable marker read-back後にShooter ownerとreceipt-free Binding／Selection／Registry artifactを束縛する`owner_typed_state_commit`のsecret-free `PublicCommitClosureV1` candidateを作る。Closure Refを含むsigned wrapperの保存／readback後だけ、同じClosure body＋Public Marker＋after Projectを一つのpublic CASで発行する。N create→N+1 reload／select／compile、unrelated N+2でbinding hash不変、cross-project／self-assert spoof、Closure欠落／差替え、各crash windowをfixture化する。`fixture_only`はexact Fixture Registry ownerだけを許可しProduction Source／Registry／Save／Packageへ昇格しない。

次の旧identityを廃止する。

| 旧identity | 移行先 |
|---|---|
| `capability.gameplay.shooter_core` | 上記Feature Capabilityの明示集合 |
| `domain.action_2d` | `genre.shooter.top_down_2d` |
| `domain.tps_single_player` | `genre.shooter.third_person_3d` |
| `DomainPackManifest` | `PackManifestV1` |
| `domain_composition_profile_refs[]` | `composition_recipe_refs[]` |

## 4. Project、World、Scenario

### 4.1 Runtime entry

`ProjectManifest`の単数`root scene`前提を廃止する。各Projectは`runtime_entry_point_refs[1..64]`を持ち、各active Targetでdefault entryを厳密に一つ解決する。

```text
RuntimeEntryPointV1
  entry_point_id
  entry_kind: world | ui | headless
  target_selector_ref
  default_for_selected_targets: bool
  world_ref: WorldDocumentRef | null
  ui_document_ref: UiDocumentRef | null
  startup_game_system_refs[]
  activation_policy_ref
```

`runtime_entry_point_refs[]`はexact `DocumentRef<RuntimeEntryPointDocumentV1>`であり、entryは本文をManifestへ埋め込まない。`RuntimeEntryPointDocumentV1`、`RuntimeTargetSelectorDocumentV1`、`RuntimeEntryActivationPolicyDocumentV1`は共通Document headerの`relative_path`と`content_hash`を持つProject-owned Sourceである。entryの`target_selector_ref`と`activation_policy_ref`は各Documentへのexact ref／schema version／content hashで解決する。

三Documentは`DocumentRef.stable_id == header.document_id == payload.entry_point_id | selector_id | policy_id`を必須とする。`selected_runtime_entry_point_hash`はDocument hashでなく、hash Fieldを持たない`RuntimeEntryPointV1` payloadのMCD canonical semantic hashだけを意味する。Document hashはrefの`content_hash`、selector／policy hashは各hash Field自身を除くpayload hashであり、相互代用しない。

```text
RuntimeTargetSelectorV1
  selector_id
  selector_version
  selector_hash
  target_profile_refs[1..64]

RuntimeTargetSelectorHashPayloadV1
  selector_id
  selector_version
  target_profile_ref_count
  target_profile_refs[1..64]

RuntimeEntryActivationPolicyV1
  policy_id
  policy_version
  policy_hash
  readiness_timeout_ms: uint32[1..120000]
  failure_semantics: reject_activation_keep_last_valid | fault_session_reverse_teardown
  cancel_semantics: preparing_before_first_publish | not_cancelable
  explicit_deactivation_semantics: graceful_reverse_teardown | immediate_reverse_teardown
```

selectorはexact `DocumentRef<TargetProfileDocument>`だけをStable ID／schema／content hash順で持ち、wildcard、tag、表示名、platform名、active Targetの現在値によるlookupを許可しない。`target_profile_ref_count`は配列長と一致し、duplicate／same-ID different-hash／非canonical orderを拒否する。`selector_hash=SHA-256(ASCII "MIRAKAN_RUNTIME_TARGET_SELECTOR_V1" || uint32_be(length(MCD canonical bytes of RuntimeTargetSelectorHashPayloadV1)) || payload bytes)`であり、payloadはhash Fieldを持たない。ID、version、count、ref全Fieldをbindし、ID-only mutation、count mismatch、旧hash再利用を拒否する。entry／selector／policyのcreate／update exact六Operationだけをcurrent MCD Registryへ完全`McdOperationContractV1`として登録する。六recordは共通Envelope、exact input／output／rate-limit／receipt MCD ref、version／hash付きauthority、R2、side effect、idempotency、transaction、12件のpure `policy` pre／post refs、closed `DiagnosticCodeRefV1` set、timeoutを全件明示する。Root SceneとRuntime Scope migrationはPerformance Scale／Physics Intent Role migrationと共に`conditional_legacy_migration` exact四件で、current MCD／Manifest／Service／Policy／Validator／migration専用Diagnostic／Receipt／Provider／MCP／alias／Signer集合はexact `[]`である。Diagnostic refはID／code／version／content hashをcanonical Registryへexact解決し、各active Operationのvalidator error union、reachable errors、`errors[]`をset equalityにする。`request_hash`はExecutable Contractsのcurrent V2定義だけを参照する。Project ownerは六exact active Operation refだけを参照し、metadataを複写しない。Root migration destination Receipt templateはsigned legacy inventory成立後だけWorldをcontent ref、selector／policy／entryをcontent ref＋owner固有semantic hashで記録する。

tagged branchは次を強制する。

- `world`: `world_ref`だけをbranch固有必須refにし、`startup_game_system_refs[0..128]`を許可する。
- `ui`: `ui_document_ref`だけをbranch固有必須refにし、`startup_game_system_refs[0..128]`を許可する。Worldは不要。
- `headless`: `startup_game_system_refs[1..128]`を必須にし、World／UI／surfaceを要求しない。

各active Targetについて`default_for_selected_targets=true`のentryだけを対象に`target_selector_ref`の被覆を検査し、default 0件または2件以上を拒否する。default selector集合とactive Target集合はset equalityで一致させ、inactive extra Targetも拒否する。non-default entryのselector overlapは許可し、benchmark、menu、game、server等を同Targetへ複数登録できる。Runtimeはdefaultまたは明示選択を推測せず、実際の選択結果を`selected_runtime_entry_point_ref`、`selected_runtime_entry_point_hash`、`target_selector_hash`、`activation_policy_hash`、`entry_branch_closure_hash`としてCompile Manifestへ保存する。owner-typed Provider Bindingを一件以上選んだ時だけ`selected_provider_binding_set_hash`をcanonical presentにし、post-commit Project revision／document set hash、Registry ref／hash、Binding Document content hash、revision非依存semantic hashをentry closureへ入れる。Binding payloadへProject revisionを戻さない。World／Topology／streaming hashは選択branchが`world`で対応Sourceが存在する場合だけcanonical presentとし、`ui`／`headless`ではcanonical omissionにする。

`entry_branch_closure_hash`は次の順序付きcanonical inputだけから計算する。

```text
entry_kind
selected_runtime_entry_point_hash
target_selector_hash
activation_policy_hash
target_profile_hash
game_system_dependency_graph_hash
system_implementation_set_hash
selected_provider_binding_set_hash?
world_document_hash?
ui_document_hash?
spatial_topology_hash?
world_streaming_plan_hash?
startup_system_closure_hash?
```

選択branchの全startup system closureは存在する全推移hard dependency、Implementation Variant、State owner relation、Target compatibilityを含む。startup systemsが空でもbranch rootは上記共通hashとbranch固有hashで確定する。Runtime Packageはこのclosureを格納する外側Artifactであり、Package自身のhashをclosure inputへ含めない。未選択branch Fieldや未選択Recipe inventoryも含めない。

`PlayPreparing`は選択entry、Runtime package、System Graph、Target Planと、そのbranch固有closureだけを検証する。`world`はWorld closure、`ui`はUI closure、`headless`はstartup system closureを持つ。branch activation setの準備後に、runtime session配下へWorld instance、UI session、startup systemsをbranch別optionalとして生成する。`Starting`はbranch activation setを準備し、選択された場合だけPresentation child／surfaceを準備する。`surface_unavailable`はApplication stateでなくoptional Presentation child stateで、simulationを暗黙停止しない。`surface_generation`はsurfaceを持つPresentationだけのtagged fieldであり、headless branchへ偽surfaceを作らない。Host／Platform control threadがouter loopを所有し、window threadを暗黙ownerにしない。Input SourceとPresentationが未登録ならinput／render workを省略し、strict headlessではWindow／Surface／RenderSnapshot／Render thread dependencyを0件にする。

stop、fault、restartは実際にactivateしたstartup／UI／World／presentation dependency DAGを常にreverse teardownする。activation policyの明示deactivation値はgraceful／immediateな要求だけを選び、stop／fault時のteardown省略や依存順の再定義には使用しない。Project Document identity／selector-policyの二fixtureとRuntime branch activation／reverse teardownの二fixtureを`RuntimeEntryOwnerIntegrationManifestV1`のexact ref／hash mappingで束ね、`fixture.integration.project-runtime-entry.owner-resolution`を合格させる。

旧`root scene`はhash付き`RootSceneMigrationPlanRefV1`が唯一のpreimageである。Planはlegacy closure、World／selector／activation policy／runtime entryのexact四Document mutation、各active Targetのdefault bindingを持つ。各mutationは`create {allocation_intent, plan_local_ref}`または`update {current_ref, before_hash, preserved_identity}`のclosed branchであり、Stable ID allocation件数とmappingはcreate branch数だけに一致する。Preview、Prepared Candidate、postcondition、Prepared Receipt payloadは同じPlan／allocation mappingを束縛し、[Executable Contracts §8](../../architecture/02-foundation/executable-contracts.md#8-operation定義)の`private durable marker read-back → owner_typed_state_commit PublicCommitClosure candidate → canonical signed wrapper → PublicCommitClosure＋Public Marker＋四Documentのatomic CAS`だけで一括publishする。実行時の暗黙migration、`Level` alias、`ui`／`headless`への近似変換を禁止する。

### 4.2 Generic World

`WorldDocumentV1`は空間、Scene、global composition、persistent entity、任意のspatial topologyだけを所有する。

- `scene_document_refs`は`0..65,535`とし、procedural-only Worldを許可する。
- `spatial_topology_definition_ref`は`0..1`とする。
- Level、Objective、Completion、Encounter、player／party transferをCore Worldから除去する。
- World activation、Scene activation、Cell streamingはObjectiveやResultを要求しない。

`SpatialTopologyDefinitionV1`は`space_nodes[]`、`transition_edges[]`、`activation_entry_refs[]`を持つ。entryは0件を許可し、到達不能spaceは`intentionally_isolated=true`で明示する。World ownerはMCD `type.world.spatial_transition_destination` version 1を登録し、exact World／Topology／Edge／target Space ref、各version／hash、Edge policyに従うoptional／required Anchorを一recordへ閉じる。transition Requestはそのtyped refを参照しFieldを複製しない。transition payloadはtyped `transfer_subject_refs[]`を使用し、Player／Partyを固定しない。

Streaming interestはclosedなPlayer enumではなく、owner登録済みtyped interest-source contract refの集合を使用する。observer、entity、camera、scripted anchor等を登録可能とし、unknown／removed ownerを拒否する。

`WorldAuthoringPlanV1.affected_world_refs`は既存World編集branchで1～64件、新規World作成branchでは`create_document_kinds`がWorld作成を厳密に一件宣言する時だけ0件を許可する。新規Worldの`WorldAuthoringContextV1`はCommit成功後にexact `world_ref`付きで生成し、未発行IDを推測しない。

`ProceduralWorldDefinitionV1`、`WorldAuthoringPlanV1`、precommit `GeneratedWorldSemanticCandidateV1`は同じ`selected_validation_provider_bindings[]`のexact Binding Document ref/content hash、別の`resolved_binding_closure_hash`、output MCD schema集合を持つ。Production recordはFixture bodyでなくexact signed Qualification Receiptを参照する。fresh 3-runはGateway／Brokerを呼ばず、UUIDを持たない連続local ID semantic graph bytes、canonical order、`semantic_graph_hash`、candidate Artifact semantic hashだけのbyte equalityを要求する。

candidateはexact `{project_id, expected_project_revision, document_set_hash}`、immutable `candidate_hash`、`local_id_count`を持つ。合格後、caller-issued `allocation_request_id`をintent subjectへ含め、Authoring Command Gatewayの完全登録済みinternal `operation.world.allocate_generated_stable_ids@1`をexact一回呼ぶ。named Input／Result／Receipt、R3 `MutationAuthorizationBindingV2`、pure pre／post Policy、closed Diagnostic／Validator closure、rate／timeout、Service allowlist、`trusted_internal` exposureを一つのContract set transactionへ登録する。Manifest／prepared Receipt／Publication Stateは同じProject triple、candidate hash、intent／request identityをbindし、`allocated_uuid_count=local_id_count+1`（mapping＋`delta_id`）を検証する。[Executable Contracts §8](../../architecture/02-foundation/executable-contracts.md#8-operation定義)の`private durable marker read-back → World owner／receipt-free Deltaを持つowner_typed_state_commit PublicCommitClosure candidate → canonical signed wrapper → PublicCommitClosure＋Public Marker＋Projectのatomic CAS`だけでpublishし、crash retryは同じClosure／wrapper／Resultへroll-forwardする。

Source、validation output、Preview、Cook、Commitは同じmappingを共有し、二回目allocation、subsystem別mapping、local ID残存を拒否する。final `GeneratedWorldDeltaV1`はallocation ref／hash、両candidate hash、mapping適用済みrecord、self-excluding content hashを持つ。truth tableはabsent+empty=skip、absent+nonempty=reject、selected+required output不足=reject、selected+valid=execute／accept、selected+stale／invalid／failure=Delta全rejectである。hard closureへ含めるのはWorld／Tile／Blockoutが明示選択したgeneric dependency bindingだけであり、Renderer／Collision／Navigationを名前で常時列挙しない。失敗時はlast-valid Source／Artifact／World generationを維持する。五truth-table fixtureと`fixture.world.procedural-core-only`を必須にする。

### 4.3 Optional Scenario／Stage Feature

有限Stage、entry／exit、Objective、Completionが必要なProjectだけがScenario／Stage Feature Packを使用する。

```text
StageDefinitionV1
  stage_id
  world_ref: WorldDocumentRef | null
  content_source_refs[]
  entry_anchor_refs[]
  exit_anchor_refs[]
  stage_game_system_refs[]
  objective_definition_refs[]
  spawn_definition_refs[]
  transition_policy_refs[]
  completion_mode: none | explicit_outcomes
  completion_contract: CompletionContractV1 | null
  save_replay_policy_ref
  fallback_contract
```

`completion_mode=none`ではcompletion contract、objective、resultを要求しない。Stage ScopeはFeature Packが`scope.feature.scenario_stage.instance`として登録する。Coreは`level_instance`を列挙しない。

`world_ref`はrequired nullableである。world Runtime Entryへ結ぶStageだけがexact World refを必須とし、UI-only／headless Stageは`world_ref=null`とowner-typed `content_source_refs[]`を使用できる。`entry_anchor_refs[]`、`exit_anchor_refs[]`、spatial spawn fieldは`world_ref=null`で0件、world branchでだけ参照kindを検証する。

`spawn_definition_refs[]`はspatial spawnだけに使用し、`world_ref=null`では0件にする。worldless Stageの非空間処理は`stage_game_system_refs[]`とowner-typed contentを使う。`StageContentActivationPolicyV1.content_activation_scope`は`none | entry_anchor_closure | listed_content_refs`であり、`none`はcontent／anchor 0件のsystem-only headless Stageをvalidにする。

Stage transition destinationはMCD `type.feature.scenario_stage.transition_destination` version 2のinjective tagged unionとし、`stage | runtime_entry | world_space | ui | headless | session_end`ごとにexact 1 branchだけを許可する。`runtime_entry`はworld／nonspatial、`world_space`はworld／spatial、`ui`と`headless`は各専用branchだけで表す。`StageTransitionContractRefSetV1`はStage owner四refとWorld owner一refのexact 5 refだけを持つ。

cross-owner検証はReceiptを含まない`StageTransitionCrossOwnerCandidateV1`を先にhashし、同じcandidate hashへ三owner Receiptと五gate Receiptを発行し、最後にcandidate＋八Receiptから`final_closure_hash`を作る。Receiptはfinal hashをpayload／subject／signature preimageに含めない。Runtime ownerのregistered `BoundaryDeliveryContractV1`はsealed payloadを次のeligible `T00_BoundaryApply`へexactly once配送し、World／spatial／Locomotion／UI／headlessを仮定しない。このRuntime contractをexact五refへ六件目として混ぜず、owned public inventory、runtime port inventory、cross-owner MCD set、Port closure、Gameplay Features aggregateを五つのlike-for-like gateで個別比較する。

Stage lifecycle Systemは`game_system.extension.feature.scenario_stage`、`owner_layer=feature_pack`、exact Feature owner ref、`system_origin=owner_package`である。Scenario／Stage authoring Operationは完全登録されていないためcurrent setとManifest inventoryを空、Capabilityを`not_activated`とする。旧候補名をaliasとして読まず、将来は一つのexact Operation setと§8.1の認可／Contract Set／publication／honest-activation closureを同じactivation transactionへ登録する。

## 5. Gameplay、Scope、Locomotion

### 5.1 System scope

旧版`GameSystemSpecV1.runtime_instance_scope`のclosed enumを、現行`GameSystemSpecV2.runtime_scope_type_ref: RuntimeScopeTypeRefV1`へ置換する。MCD logical IDは`type.game_system.spec`のままversion 2をcurrentとする。V1は現時点のcurrent Contract Set memberではなく、実legacy inventoryがcanonical bytes／source Closureを証明した場合だけconditional migration inputとしてretainedできるsource candidateである。

Core登録Scopeは次に限定する。

- `scope.core.application`
- `scope.core.runtime_session`
- `scope.core.world`
- `scope.core.entity`
- `scope.core.ui_session`

```text
RuntimeScopeTypeCatalogV1
  catalog_id: runtime_scope.catalog.active
  catalog_schema_version: 1
  catalog_version
  catalog_hash
  contract_set_hash
  dependency_registry_ref: RuntimeScopeDependencyRegistryRefV1(
    registry_id, registry_revision, registry_content_hash)
  dependency_registry_hash
  entries[5..4096]:
    scope_type_ref: RuntimeScopeTypeRefV1(scope_type_id, scope_type_version, scope_type_hash)
    instance_key_schema_ref: McdContractRefV1(kind=type)
    owner_ref: RuntimeScopeOwnerRefV1(owner_id, owner_revision, owner_content_hash)
    lifetime_ref: McdContractRefV1(kind=policy)
    save_replay_policy_ref: McdContractRefV1(kind=policy)
    activation_condition_ref: McdContractRefV1(kind=policy)
    deactivation_condition_ref: McdContractRefV1(kind=policy)
```

Core entryは次のexact 5 rowと完全一致する。

| `scope_type_ref` | `instance_key_schema_ref` | `owner_ref` | `lifetime_ref` | `save_replay_policy_ref` | `activation_condition_ref` | `deactivation_condition_ref` |
|---|---|---|---|---|---|---|
| `scope.core.application` | `type.runtime_scope.key.application_singleton` | `owner.core.runtime` | `policy.runtime_scope.lifetime.process` | `policy.runtime_scope.save_replay.application_none` | `policy.runtime_scope.activation.process_started` | `policy.runtime_scope.deactivation.process_stopping` |
| `scope.core.runtime_session` | `type.runtime_scope.key.runtime_session_uuidv7` | `owner.core.runtime` | `policy.runtime_scope.lifetime.runtime_session` | `policy.runtime_scope.save_replay.runtime_session` | `policy.runtime_scope.activation.entry_ready` | `policy.runtime_scope.deactivation.stop_or_fault` |
| `scope.core.world` | `type.runtime_scope.key.world_instance` | `owner.core.world` | `policy.runtime_scope.lifetime.world_instance` | `policy.runtime_scope.save_replay.world` | `policy.runtime_scope.activation.world_branch_ready` | `policy.runtime_scope.deactivation.world_branch_teardown` |
| `scope.core.entity` | `type.runtime_scope.key.entity_stable_id` | `owner.core.runtime_ecs` | `policy.runtime_scope.lifetime.entity` | `policy.runtime_scope.save_replay.entity_owner_state` | `policy.runtime_scope.activation.entity_created` | `policy.runtime_scope.deactivation.entity_destroyed` |
| `scope.core.ui_session` | `type.runtime_scope.key.ui_session_uuidv7` | `owner.core.ui` | `policy.runtime_scope.lifetime.ui_session` | `policy.runtime_scope.save_replay.ui_session` | `policy.runtime_scope.activation.ui_branch_ready` | `policy.runtime_scope.deactivation.ui_branch_teardown` |

表はID表示だけで、保存値はscope version／hash、MCD version／Contract set hash、owner revision／content hashを全cellに持つ。active Owner／Dependency Registryは全refを実体recordとして解決し、Registry ref／hashのexact equalityとself-excluding canonical Registry hashを検証する。entryはscope ID byte順にsortし、Catalog hashはdomain separator、catalog ID、schema／catalog version、Contract set hash、Dependency Registry ref canonical bytes／同値hash、entry count、七typed refのcanonical bytesをexact inputにして自己hashを除外する。extension ownerは自身のregistered namespaceへscopeを追加できる。Scenario Stage、Encounter、Scoring、Shooter Game Flowの7-Field rowと旧scope migration record／fixtureは各owner文書だけが所有する。Coreからextension scopeへの依存、cross-owner未宣言依存、duplicate、unknown owner、unavailable／removed owner、各dependency version／hash不一致をtyped rejectする。

旧`runtime_instance_scope`はcurrent validatorが常にrejectし、`operation.runtime_scope.migrate_game_system` version 1はsigned legacy inventoryが未成立のconditional destination templateである。current Operation／Contribution／Qualification Binding／Activation Catalog／fixture投影はexact `[]`とする。将来activation時もlogical operation／type／policy IDへschema majorを埋め込まず、Input、選択Contribution、Prepared Receiptのexact Field `retained_source_mcd_ref={type.game_system.spec,version=1,source_contract_set_hash}`をbyte equalityにし、Input／Prepared Receiptのexact `source_foundation_definition_closure_ref`を同じretained Closureへ解決する。destination schemaは`type.game_system.spec` version 2かつdestination Closureとする。Coreはgeneric migration schema／validatorだけを提供し、実Inventoryが証明したCore／Pack mapping、adapter、fixtureを各ownerが下向きに同一activation transactionへ登録する。typed Result／Receiptはrequest／idempotency、before／after Project、両source Field、Source、Contribution、scope／auxiliary refs、Save／Replay mapping、Preview／Validation／Commitをbindし、rejected diagnostics 1～64、Receipt diagnostics 0～64、ephemeral generation非移行を記録する。旧enum aliasまたは`source_system_schema_ref`別名をcurrent resolverへ残さない。

### 5.2 InteractionとGame Flow

共通Interaction rejectionは`game_flow_disallowed`を持たない。`operation_eligibility_policy_ref`によるowner判定と、genericな`policy_denied`を使用する。Shooter Game FlowはShooter Pack内だけで`scope.genre.shooter.game_flow.instance`を読むInteraction policyを提供する。

`InteractionSnapshotV1@1.rejection_reason=game_flow_disallowed`は`InteractionSnapshotV1@2.rejection_reason=policy_denied`へversioned clean migrationする。@2で旧enumをaliasとして受理せず、owner／policy／schema hashを一意に解決できないSource／Save／Replayはmigrationを拒否してlast-validを維持する。

Saveは登録済みState ownerと`SaveReplayContractV1`が宣言したfieldだけを保存する。共通Runtimeが常にGame Flow stateを保存してはならない。

### 5.3 Locomotion

Navigation Path FollowingをCharacter Motorへ固定しない。

```text
MotionExecutorPortV1
  executor_capability_ref: McdContractRefV1(kind=capability)
  movement_profile_schema_ref: McdContractRefV1(kind=type)
  accepted_intent_schema_refs[1..64]: McdContractRefV1(kind=type)
  transport_message_schema_ref:
    McdContractRefV1(id=type.navigation.motion_executor_intent_batch, version=1, contract_set_hash)
  resolved_motion_schema_ref: McdContractRefV1(kind=type)
  compatibility_predicate_ref: McdContractRefV1(kind=policy)
  failure_diagnostic_refs[1..32]
```

Kinematic Motion ExecutorはC1で適格化するProviderの一つである。Vehicle、flying、swimming、RTS unit、board token等は別Capability／Providerとして同じPortへ接続できる。Physics Coreはgenre／Feature非依存のgeneric Provider／adapterだけを提供する。

Navigationはこの7 Field型、`MotionExecutorProviderCatalogV1`／record、`MotionExecutorProviderRecordRefV1`、`MotionExecutorIntentBatchV1`、generic `MotionIntentContributionBindingRegistryV1`の唯一のcanonical ownerである。RecordRefはCatalog ID／version／hash／Contract setとprovider ID／version／content hashを一つにbindし、Batch、Selection State、Save、Replayが同じ型を使う。Catalog recordはtagged owner identity、production／fixture usage、7 Field descriptor、implementation System、Target、Qualification Receiptを持つ。Coreはprovider-neutral resolverだけを所有し、owner固有proposal schema／adapter／binding／positive fixtureは各PackまたはProjectが登録する。Path Followingは`executor_capability_ref`とProvider-owned `movement_profile_ref`を要求し、compatibility policyがintent schema、movement profile、Targetを検証する。path progressは`resolved_motion_schema_ref`の結果だけで判定する。

Batch entriesは1～16件で、各entryが`intent_schema_ref: McdContractRefV1`、payload refまたはinline value identityのtagged union、payload hash、producer Game System ref、proposal IDを持つ。accepted set検査はbatch envelopeでなくentries schema集合だけへ適用する。type spoof、payload hash mismatch、duplicate proposal、noncanonical order、root-motionあり／なしをfixture化する。Gameplay schemaは`type.feature.character_locomotion.gameplay_motion_intent`、Navigationは`type.navigation.movement_intent`、Animationは`type.animation.root_motion_proposal` version 1である。Provider-private派生入力をpublic accepted setへ混ぜない。

`game_system.extension.feature.character_locomotion.contribution`は`owner_layer=feature_pack`としてFeature-owned proposal／adapter contributionだけを所有する。Navigation／Core-owned `game_system.engine.navigation.motion_intent_batch_publisher`が全owner contributionをcanonical mergeし、`MotionExecutorIntentBatchV1`をT40に一度だけ発行する。`MotionIntentContributionBindingRecordRefV1`、RegistryRef、`MotionIntentBindingSelectionDocumentV1`をCompile／Save／Replay closureへ保存し、Character、RTS、board-token、Animationがgeneric publisherをforkしない。Physicsなしboard-token／RTSとFeature PackなしPath fixtureを必須にする。

Physics Kinematic Motion Providerの7 FieldはCapability `capability.motion_executor.physics_kinematic`、Profile `type.physics.kinematic_motion_profile`、generic Navigation intent／contribution envelope、transport batch、resolved `type.physics.kinematic_resolved_motion`、compatibility policy、failure diagnosticsで固定する。owner固有proposalはPack-owned adapter recordを介し、Physics descriptorへFeature type IDをhard-codeしない。board-token／RTS fixture Providerも7 Fieldとowner／usage／Target／dimension／diagnosticを完全登録し、Physics dependency 0件を検証する。

Physics AI意味解決もobject名をCore enumへ固定しない。Core-owned `PhysicsIntentRoleRegistryV1`は六roleのowner、五axis allowed set、Capability、fixture、self-excluding record hashを完全materializeし、Record内へ自身のhash付きRoleRefを埋め戻さない。`PhysicsIntentRoleSelectionDocumentV1`をProject Source正本、Registry／Resolutionを派生とする。旧enum migrationのContribution Registry／Operation／Prepared payload／Receipt／Validator／Manifestはcurrent集合exact `[]`のconditional destination templateであり、実signed inventoryを満たすatomic activation後だけexact一件へ移行し、近似Core roleへ変換しない。

`feature.character_locomotion@1`のPack installはPhysicsを要求しない。Physics Kinematic Motion ProviderはC1 reference recipe／qualification providerの一つに限る。Animation root motionもgeneric contribution registryからselected executorへ届き、Providerへ直接提出しない。`gameplay_motor`旧modeはprovider-neutral modeへclean migrationする。T40はselected executor、T50はactive Physics providerがある場合だけ実行する。missing／incompatible provider、stale result、provider failureはtyped failureとし、last-valid stateを維持する。

## 6. ClockとPause

60 HzはC1／C2で唯一のqualified baselineとして維持するが、Core schema／phase ABIへliteralとして固定しない。

```text
ReducedPositiveRationalV1
  numerator: positive uint32
  denominator: positive uint32
  invariant: gcd(numerator, denominator) = 1

SimulationCadenceProfileRefV1
  profile_id: namespace付きStableId
  profile_version: positive uint32
  profile_content_hash: SHA-256

SimulationCadenceProfileV1
  profile_id: namespace付きStableId
  profile_version: positive uint32
  cadence:
    kind: fixed
      rate_hz: ReducedPositiveRationalV1
      invariant: rate_hz <= 1,000,000,000/1 Hz
      max_catch_up_steps: uint8[1..16]
      overrun_policy: clamp_and_report | fault
    | kind: variable
      minimum_step_seconds: ReducedPositiveRationalV1
      maximum_step_seconds: ReducedPositiveRationalV1
      sample_quantum: nanosecond
      invariant: each bound is an integer 1..4,294,967,295 nanoseconds
      max_steps_per_outer_loop: uint8[1..16]
      sampling_policy: monotonic_elapsed_clamped
      replay_delta_policy: record_exact_rational_delta
    | kind: turn_based
      advance_command_type_ref: exact McdContractRefV1(kind=type)
      max_pending_steps: positive uint16
      wall_time_advance_policy: forbidden
    | kind: explicit_step
      step_request_type_ref: exact McdContractRefV1(kind=type)
      max_steps_per_request: positive uint16
      logical_step_duration_seconds: null | ReducedPositiveRationalV1
      wall_time_advance_policy: forbidden
  physics_substep_profile_ref: null | PhysicsSubstepProfileRefV1
  profile_content_hash: SHA-256

SimulationAdvanceControlRefV1
  control_type_ref: McdContractRefV1(kind=type)
  control_id: namespace付きStableId
  control_version: positive uint32
  control_content_hash: SHA-256

SimulationAdvanceIntervalV1
  cadence_profile_ref: SimulationCadenceProfileRefV1
  advance_sequence: positive uint64
  interval:
    kind: fixed
      logical_duration_seconds: ReducedPositiveRationalV1
    | kind: variable
      logical_duration_seconds: ReducedPositiveRationalV1
      monotonic_sample_sequence: positive uint64
    | kind: turn_based
      accepted_advance_command_ref: SimulationAdvanceControlRefV1
      accepted_advance_command_sha256: SHA-256
      logical_duration_seconds: null
    | kind: explicit_step
      accepted_step_request_ref: SimulationAdvanceControlRefV1
      accepted_step_request_sha256: SHA-256
      request_step_ordinal: positive uint16
      logical_duration_seconds: null | ReducedPositiveRationalV1
  interval_content_hash: SHA-256

SimulationAdvanceIntervalRefV1
  cadence_profile_ref: SimulationCadenceProfileRefV1
  advance_sequence: positive uint64
  interval_content_hash: SHA-256

RuntimeTimeRefV1
  time:
    kind: simulation
      cadence_profile_ref: SimulationCadenceProfileRefV1
      simulation_advance_interval_ref: SimulationAdvanceIntervalRefV1
      simulation_advance_interval_sha256: SHA-256
      advance_sequence: positive uint64
      phase_id: null | TickPhaseId
    | kind: presentation
      render_frame_id: positive uint64
      render_phase_id: null | RenderPhaseId

GameClockDomainProfileRefV1
  profile_id: namespace付きStableId
  profile_version: positive uint32
  profile_content_hash: SHA-256

GameClockDomainProfileV1
  profile_id: namespace付きStableId
  profile_version: positive uint32
  simulation_cadence_profile_ref: SimulationCadenceProfileRefV1
  clock_domain_registry_ref: ClockDomainRegistryRefV1
  clock_domain_activation_projection_ref:
    exact {projection_id, projection_version, projection_content_hash}
  selected_domain_refs[2..64]: ClockDomainRefV1
  pause_support_mode: unsupported | global
  pause_policy_ref: null | exact PausePolicyRefV1
  cinematic_policy_ref
  replay_policy_ref
  profile_content_hash: SHA-256

PausePolicyRefV1
  policy_id: namespace付きStableId
  policy_version: positive uint32
  policy_content_hash: SHA-256

PausePolicyV1
  policy_id: namespace付きStableId
  policy_version: positive uint32
  support_mode: unsupported | global
  global_policy:
    when support_mode=unsupported: null
    | when support_mode=global:
        pause_command_ref
        resume_command_ref
        pause_state_owner: scope.core.runtime_session
        paused_domain_policy_ref
        audio_snapshot_ref
        input_context_ref
        async_completion_policy: queue_until_resume
  policy_content_hash: SHA-256
```

- 上記三base recordのcontent hashは[Runtime scheduling／lifetime](../../architecture/04-runtime/scheduling-lifetime.md#41-clock-domainpausegameplay-timer)のASCII domain、RFC 8785 JCS、`uint32_be` length framingをexact reuseして各自己hash Fieldだけを除外する。`SimulationCadenceProfileRefV1`、`GameClockDomainProfileRefV1`、`PausePolicyRefV1`は完成base recordのID／version／self-excluding content hashからrecord外でmaterializeし、base record自身へ埋め戻さない。
- `fixed`は既約な正の有理数rateかつexact `rate_hz <= 1,000,000,000/1`、`variable`は既約でinteger-nanosecond換算が`1..UINT32_MAX ns`に収まるstep上下限を必須にし、別branch Fieldを混在させない。variable sample差はこのuint32 ns範囲へclampしてから`delta_ns/1,000,000,000`を既約化し、overflow／rounding fallbackを許可しない。
- `turn_based`は登録済みCommand、`explicit_step`は明示step要求だけで進め、wall timeから暗黙進行させない。Interval内のCommand／request Refは`SimulationAdvanceControlRefV1`、隣接SHAは同completed control object全bytesへ解決し、turnの`control_type_ref`は`advance_command_type_ref`、explicitの`control_type_ref`は`step_request_type_ref`とbyte equalityにする。
- Physics substep Refは[Physics](../../architecture/05-simulation/physics.md#21-physics-substep-profile)が所有するcompleted `PhysicsSubstepProfileRefV1`だけを使い、undefined generic Profile ref、表示名、Backend既定へfallbackしない。
- non-null Physics substepはroot外`PhysicsSubstepActivationBindingRefV1`を必須にし、Bindingが解決するCadence Profile Ref、Substep Profile Ref、Target subject、signed Qualification Receiptの`result=pass`／freshnessをRuntime Package、Save、Replayでbyte equalityにする。current binding集合はexact `[]`で、Future Capabilityのatomic Activation前は`cadence_profile_not_qualified`とする。
- Debug／Replay／Domain eventのtime identityはScheduling Ownerのclosed `RuntimeTimeRefV1`だけを使う。simulation branchは同じcompleted IntervalのProfile Ref、Interval Ref／completed SHA／sequence、optional phaseをbyte equality、presentation branchはRender Frame／optional Render Phaseだけとし、裸timestampまたはframe-to-advance推測を作らない。
- SaveはScheduling Ownerのcompleted `AuthoritativeSaveHeaderRefV1`を共通rootにし、exact `GameCandidateBuildReceiptRefV1`／completed signed SHA、Project triple `{project_id, project_revision, project_document_set_hash}`、Contract set、`TargetProfileRefV1`、Game Clock／Cadence Profile、optional Substep Binding、last committed Interval Ref／completed SHA、canonical state-owner Projection Ref集合／hashを固定する。Physics、Animation等はreceipt-free Domain Projectionを先に完成させ、Projection集合からHeader／Refを作り、その後だけroot外`AuthoritativeSaveDomainBindingV1`でHeader RefとProjection Refを結ぶ。Header／BindingをProjection base recordへ埋め戻さない。
- ReplayはDebugging Ownerの開始時immutable `DebugSessionDescriptorRefV1`、completed `AuthoritativeReplayHeaderRefV1`、gapless full canonical `SimulationAdvanceIntervalStreamRefV1`、typed `AuthoritativeReplayRangeArtifactRefV1`集合、Sliceの連続`SimulationAdvanceIntervalRangeClosureV1`を使う。Sessionのend／completeness／gapはDescriptorを更新せず外部`DebugSessionClosureRefV1`へ閉じ、Replay HeaderはClosureをhash入力に戻さない。Header／Slice／signed Receipt間のGame Candidate Build Receipt／completed SHA、Project triple、Target、Session Descriptor、Cadence Profileをbyte equalityにし、Header／Stream／Range Artifact／Range Closure／Slice／Receipt間でSubstep Binding、start／end／count／artifact kind／schema／Ref／completed SHA／set hashを一致させる。Domain Projectionはreceipt-freeでHeader Refを持たず、Projection／Interval Chunk／Range Artifactを完成してからStream／Projection set／Artifact set、Header／Ref、最後にroot外`AuthoritativeReplayDomainBindingV1`を作る。欠落、重複、reorder、branch／control差、別Session／Build／Project／Target、hash cycleを再生前に拒否する。
- C1／C2 active Profileのpre-hash source notationは`{profile_id=simulation.cadence.reference.fixed_60, profile_version=1, cadence={kind=fixed, rate_hz=60/1, max_catch_up_steps=4, overrun_policy=clamp_and_report}, physics_substep_profile_ref=null}`だけとし、compilerがself-excluding content hashを追加した後に外部Profile Refをmaterializeする。
- alternate fixed rate、`variable | turn_based | explicit_step`、別physics substepは`future.capability.alternate-simulation-cadence-and-substep`のActive migrationとTarget別Qualification前に`cadence_profile_not_qualified`とする。
- fixed advance durationは1開始のsequence `s`について`floor(s * 1,000,000,000 * denominator / numerator) - floor((s - 1) * 1,000,000,000 * denominator / numerator)`で先頭0 nsからpartitionし、checked unsigned 128-bit以上で評価する。切り捨てnanosecondの反復加算、float蓄積、overflow、0 ns partitionを許可しない。全authoritative consumerは同一`SimulationAdvanceIntervalV1`のProfile ref／sequence／hashを使い、turn／explicitのnull durationを0または1/60秒へ変換しない。
- Physics／Navigation、Animation、VFX、Input／Replay、Debug step／hang、LOD、Gameplay Feature timer／cadence、Save、Native ABIはProfile ref／hashを記録する。別rateまたは別kindのActivationはこれら全consumerとTarget fixtureを`future.capability.alternate-simulation-cadence-and-substep`で同時に閉じ、一Subsystemだけを60から切り離してsupportedと表示しない。

Pauseは任意Capabilityとする。`GameClockDomainProfileV1.pause_support_mode`は`unsupported | global`のclosed値で、`unsupported`なら`pause_policy_ref=null`、`global`なら同じmodeへ解決するnon-null refを必須にする。`unsupported`のProjectへPause command／stateを生成せず、要求にはtyped `pause_not_supported`を返す。将来のscoped pauseは別Schema versionとQualificationにする。Pause ownerはCoreの`scope.core.runtime_session`を使い、Weapon、Encounter、Shooter Game Flowを共通規約へ記載しない。

## 7. AIが理解できる上位仕様

AIの「理解した」という文章を状態にしない既存方針を維持し、上位入力をfield-level MCDへ閉じる。

### 7.0 共通closed-schema規則

本節の九Schema、すなわち`GameIntentDraftV1`、`GameBriefV1`、`GameSpecDocumentV1`、`QuestionRecordV1`、`AssumptionRecordV1`、`DecisionRecordV1`、`GameUnderstandingClosureV1`、`AiCatalogEntryV1`、`AiTaskContextCapsuleV1`は、同じMCD compiler profileでDraft 2020-12 closed objectへcompileする。全top-level Fieldと全nested Fieldは、型の後ろに`| null`を明記したFieldを含めてpresence requiredである。`| null`を明記しないnull、unknown Field、未定義enum、数値化したboolean、空文字列、tagged unionのsibling Field混在を拒否する。互換Field追加は同じV1へ行わずV2にする。

共通scalarは`StableId=1..128 UTF-8 bytes, ^[a-z][a-z0-9._:-]*$`、`Text<N>=NFC UTF-8 1..N bytes`、`Sha256=lowercase hexadecimal exact 64 chars`、`UtcTimestamp=RFC 3339 UTC exact millisecond precision`、整数は記載bit幅のnon-negative safe JSON integerとする。改行を許す`Text`もNULとC0 controlを拒否する。自由文Fieldは説明・表示・検索だけに使い、Operation、Authority、Approval、Project、Target、Provider、Model、Host、Deploymentまたは実行先を自由文から導出しない。

`*RefV1`は論理ID、schema version、completed object content hashを持つ既存のcontent-addressed typed refであり、logical ID／version／hash／kindを全てbyte equalityでexact一件へ解決する。bare ID、display name、prefix／tag／wildcard lookup、同ID別hash、wrong-kind、unresolved refを拒否する。`ProjectSnapshotRefV1`はexact `{project_id, project_revision, document_set_hash}`である。Ref集合とenum集合は記載上限内のstrict sorted setとし、refは`kind -> logical ID -> schema version -> content hash`、enumはUTF-8 byte順でsortし、duplicateとsame-ID different-hashを拒否する。Text集合はNFC UTF-8 bytes順のstrict sorted setである。`requested_experience_statements[]`と二つの`candidate_*_statements[]`だけは意味順を保つordered sequence、`omitted_ranges[]`は`json_pointer -> start_index -> end_index_exclusive`順である。

全九recordは`schema_version: 1`と`content_hash: Sha256`を持つ。`content_hash = SHA-256(ASCII "MIRAKAN_CLOSED_RECORD_V1/<SchemaName>" || uint32_be(length(McdCanonicalBinaryV1(record excluding content_hash))) || McdCanonicalBinaryV1(record excluding content_hash))`とする。canonical round-trip後もbytesとhashが不変でなければならず、hash Field自身、署名、Receipt、取得順、表示用文字列のnormalization前bytesをpreimageへ混ぜない。外部Refは`{record logical ID, schema_version=1, content_hash}`を持ち、この式の再計算成功後だけmaterializeする。九Schema間のhash DAGは`Intent -> Question`、`Brief -> Intent／Question／Assumption／Decision`、`Spec -> Brief／Decision`、`Assumption -> open Medium Question／Decision`、`Question -> Decision`、`Closure -> Brief／Spec／Question／Assumption／Decision`、`Capsule -> Brief／Spec／Catalog`だけを許可する。`DecisionRecordV1.subject_ref`とOption refは九Schema以外のtyped subject／optionへ限定し、self-ref、back-edge、ClosureまたはCapsuleへのrefを拒否する。Catalog dependency graphはacyclic、Capsule continuationはself／ancestor capsuleを参照不可とする。

### 7.1 `GameIntentDraftV1`

Disposableな入力解決RecordでありProject正本ではない。

```text
GameIntentDraftV1
  schema_version: 1
  draft_id: StableId
  project_ref: ProjectSnapshotRefV1 | null
  source_input_evidence_refs[1..64]: EvidenceArtifactRefV1
  requested_experience_statements[1..64]: Text<2048>
  candidate_requirement_statements[0..256]: Text<2048>
  candidate_constraint_statements[0..128]: Text<2048>
  ambiguity_refs[0..128]: QuestionRecordRefV1
  unsupported_candidate_refs[0..128]: UnsupportedCapabilityCandidateRefV1
  created_at: UtcTimestamp
  content_hash: Sha256
```

`project_ref=null`は新規Project前のdraftだけに許可し、Project-bound Briefへ昇格するときはexact Project snapshotへ束縛する。三statement配列は入力の意味順を保つがduplicate byte stringを拒否する。statement、PromptまたはUnsupported候補の自由文からCapability／Pack／Target IDを合成しない。

### 7.2 `GameBriefV1`

人間確認済みの制作意図を表す。

```text
GameBriefV1
  schema_version: 1
  brief_id: StableId
  project_ref: ProjectSnapshotRefV1
  source_intent_draft_ref: GameIntentDraftRefV1
  experience_summary: Text<4096>
  experience_loop_kinds[1..6]:
    finite | endless | turn_based | continuous_simulation | tool_like | mixed
  audience_requirement_refs[0..64]: RequirementRefV1
  target_profile_refs[0..64]: TargetProfileRefV1
  control_requirement_refs[0..64]: RequirementRefV1
  presentation_requirement_refs[0..128]: RequirementRefV1
  content_requirement_refs[0..256]: RequirementRefV1
  persistence_requirement_refs[0..64]: RequirementRefV1
  online_requirement_refs[0..64]: RequirementRefV1
  accessibility_requirement_refs[0..64]: RequirementRefV1
  business_and_distribution_requirement_refs[0..64]: RequirementRefV1
  explicit_non_goal_refs[0..128]: RequirementRefV1(kind=non_goal)
  accepted_assumption_refs[0..128]: AssumptionRecordRefV1
  decision_refs[1..256]: DecisionRecordRefV1
  unresolved_question_refs[0..128]: QuestionRecordRefV1
  confirmed_by_subject_ref: TrustSubjectRefV1
  confirmed_at: UtcTimestamp
  content_hash: Sha256
```

`experience_loop_kinds[]`は`finite | endless | turn_based | continuous_simulation | tool_like | mixed`のnon-empty subsetとする。Player、Character、Combat、Objective、Completion、World、Genre Packは必須Fieldにしない。

### 7.3 `GameSpecDocumentV1`

```text
GameSpecDocumentV1
  schema_version: 1
  game_spec_id: StableId
  project_ref: ProjectSnapshotRefV1
  game_brief_ref: GameBriefRefV1
  requirement_refs[1..1024]: RequirementRefV1
  runtime_entry_point_refs[0..64]: RuntimeEntryPointDocumentRefV1
  feature_pack_refs[0..128]: PackManifestRefV1(owner_layer=feature_pack)
  genre_pack_refs[0..64]: PackManifestRefV1(owner_layer=genre_pack)
  game_system_spec_refs[0..512]: GameSystemSpecRefV1
  gameplay_definition_refs[0..1024]: GameplayDefinitionRefV1
  content_plan_refs[0..256]: ContentPlanRefV1
  test_scenario_refs[1..1024]: TestScenarioRefV1
  budget_profile_refs[1..128]: BudgetProfileRefV1
  target_profile_refs[0..64]: TargetProfileRefV1
  save_replay_contract_refs[0..128]: SaveReplayContractRefV1
  decision_refs[1..512]: DecisionRecordRefV1
  unsupported_capability_refs[0..256]: CapabilityRefV1
  content_hash: Sha256
```

`genre_pack_refs[]`、World、controllable actor、Objective、Completionは0件を許可する。RequirementからCapability、Pack、System、Implementation、Test、Artifactまでの追跡を`GameUnderstandingClosureV1`が検査する。

### 7.4 質問、仮定、Decision、理解closure

```text
QuestionRecordV1
  schema_version: 1
  question_id: StableId
  project_ref: ProjectSnapshotRefV1
  subject_ref: TypedArtifactRefV1
  question_class: blocking | high | medium | low
  question_text: Text<2048>
  impact_text: Text<2048>
  resolution:
    open: {}
    | answered:
        answer_text: Text<4096>
        answer_evidence_refs[0..32]: EvidenceArtifactRefV1
        decision_ref: DecisionRecordRefV1
        answered_by_subject_ref: TrustSubjectRefV1
        answered_at: UtcTimestamp
    | withdrawn:
        reason_text: Text<2048>
        decision_ref: DecisionRecordRefV1
        withdrawn_by_subject_ref: TrustSubjectRefV1
        withdrawn_at: UtcTimestamp
  created_at: UtcTimestamp
  content_hash: Sha256

AssumptionRecordV1
  schema_version: 1
  assumption_id: StableId
  project_ref: ProjectSnapshotRefV1
  source_question_ref: QuestionRecordRefV1(question_class=medium,resolution=open)
  assumption_class: medium_default
  default_value_ref: TypedAssumptionValueRefV1
  default_summary: Text<2048>
  default_basis_refs[1..32]: EvidenceArtifactRefV1
  valid_from: UtcTimestamp
  expires_at: UtcTimestamp
  revalidation_condition_refs[1..32]: RevalidationConditionRefV1
  acceptance:
    proposed: {}
    | accepted:
        acceptance_decision_ref: DecisionRecordRefV1
        accepted_by_subject_ref: TrustSubjectRefV1
        accepted_at: UtcTimestamp
    | superseded:
        replacement_assumption_ref: AssumptionRecordRefV1
        supersession_decision_ref: DecisionRecordRefV1
        superseded_at: UtcTimestamp
  content_hash: Sha256

DecisionRecordV1
  schema_version: 1
  decision_id: StableId
  project_ref: ProjectSnapshotRefV1
  decision_kind:
    experience | requirement | constraint | implementation |
    target_profile | capability_disposition
  subject_ref: TypedArtifactRefV1
  selected_option_ref: TypedDecisionOptionRefV1
  rejected_option_refs[0..32]: TypedDecisionOptionRefV1
  rationale_text: Text<4096>
  evidence_refs[0..64]: EvidenceArtifactRefV1
  decided_by_subject_ref: TrustSubjectRefV1
  decided_at: UtcTimestamp
  content_hash: Sha256

GameUnderstandingClosureV1
  schema_version: 1
  closure_id: StableId
  project_ref: ProjectSnapshotRefV1
  game_brief_ref: GameBriefRefV1
  game_spec_ref: GameSpecDocumentRefV1
  question_record_refs[0..512]: QuestionRecordRefV1
  assumption_record_refs[0..256]: AssumptionRecordRefV1
  decision_record_refs[1..1024]: DecisionRecordRefV1
  requirement_set_ref: RequirementSetRefV1
  requirement_traceability_ref: RequirementTraceabilityRefV1
  system_graph_ref: SystemGraphRefV1
  state_owner_map_ref: StateOwnerMapRefV1
  capability_scope_ref: CapabilityScopeRefV1
  save_replay_contract_set_ref: SaveReplayContractSetRefV1
  target_profile_refs[0..64]: TargetProfileRefV1
  behavior_budget_refs[1..128]: BudgetProfileRefV1
  test_plan_ref: TestPlanRefV1
  evidence_closure_ref: EvidenceClosureRefV1
  project_shader_understanding_closure_refs[0..256]:
    ShaderUnderstandingClosureRefV1
  unsupported_capability_refs[0..256]: CapabilityRefV1
  unresolved_blocking_question_count: uint16
  unresolved_high_question_count: uint16
  unresolved_medium_question_count: uint16
  unresolved_low_question_count: uint16
  disposition: ready_to_stage | capability_unavailable
  content_hash: Sha256
```

`QuestionRecordV1.resolution`はexact一branchであり、open branchへ回答用Fieldを置かない。終端ClosureではBlocking／High／Low open件数をexact 0にする。Medium open質問は、Closureの`question_record_refs[]`にある同じexact Question refを持ち、`acceptance=accepted`、`valid_from <= closure evaluation time < expires_at`で、全`default_basis_refs[]`と`revalidation_condition_refs[]`が解決可能な`AssumptionRecordV1` exact一件がある場合だけ許可する。Medium Defaultの根拠0件、期限の省略／無期限sentinel、再検証条件0件、期限切れ、same-ID different-hash Question、同一質問への複数active assumptionを拒否する。revalidation condition成立またはEvidence freshness切れ時は、期限前でもAssumptionを再検証して新Decision／Assumption hashを発行するまでClosureをstaleとして再発行できない。

Decisionの`selected_option_ref`だけが選択の機械入力であり、`rationale_text`、Question回答、Assumption summaryをTarget／Operation／Authorityへ解釈しない。DecisionはApproval、Task AuthorizationまたはOperation実行権限を付与しない。Closureの四countは解決済みQuestion集合から再計算して一致させ、Brief／Spec／Question／Assumption／DecisionのProject lineage、Spec→Brief→Intent ref chain、全ref hashを検査する。`ready_to_stage`はBlocking／High／Low open 0、Medium assumption規則成立、Requirement→Capability→Pack／System→Implementation→Test→Artifact追跡100%、State owner重複0、stale Evidence 0、unsupported required Capability 0の場合だけである。required Capabilityが未対応なら`capability_unavailable`とし、同じ終端質問／Assumption規則は緩和しない。

## 8. AI Catalog、Context、Operation

全公開機能から`AiCatalogEntryV1`を生成する。

```text
AiCatalogEntryV1
  schema_version: 1
  catalog_entry_id: StableId
  subject_ref: TypedArtifactRefV1
  production_owner_layer: core | feature_pack | genre_pack | game_project
  production_owner_ref: ProductionOwnerRefV1
  artifact_role:
    production | cross_cutting_control_plane | reference_game |
    fixture | qualification
  architecture_layer_ref: ArchitectureLayerRefV1
  purpose: Text<2048>
  non_goals[0..64]: Text<1024>
  input_contract_refs[0..128]: McdContractRefV1(kind=type)
  output_contract_refs[0..128]: McdContractRefV1(kind=type)
  state_owner_refs[0..64]: StateOwnerRefV1
  runtime_scope_type_refs[0..64]: RuntimeScopeTypeRefV1
  phase_and_lifetime_refs[0..64]: PhaseLifetimeRefV1
  read_set_refs[0..256]: TypedArtifactScopeRefV1
  write_set_refs[0..256]: TypedArtifactScopeRefV1
  dependency_refs[0..256]: AiCatalogEntryRefV1
  incompatibility_refs[0..128]: AiCatalogEntryRefV1
  target_support_refs[0..64]: TargetProfileRefV1
  provider_profile_refs[0..64]: InferenceProviderProfileRefV1
  model_profile_refs[0..128]: InferenceModelProfileRefV1
  host_profile_refs[0..64]: EngineAiHostSecurityProfileRefV1
  deployment_profile_refs[0..64]: InferenceDeploymentProfileRefV1
  maturity_ref: MaturityProfileRefV1
  budget_refs[0..64]: BudgetProfileRefV1
  allowed_operation_refs[0..256]: McdContractRefV1(kind=operation,status=active)
  risk_and_approval_refs[0..64]: RiskApprovalPolicyRefV1
  diagnostic_and_remediation_refs[0..256]: DiagnosticRemediationRefV1
  example_refs[0..64]: ExampleArtifactRefV1
  counterexample_refs[0..64]: CounterexampleArtifactRefV1
  test_fixture_refs[0..128]: TestFixtureRefV1
  deprecation_and_migration_refs[0..64]: MigrationRefV1
  public_project_sdk_refs[0..64]: PublicProjectSdkRefV1
  content_hash: Sha256
```

`production_owner_layer`はproduction code／schemaの一意な所有層、`artifact_role`はCatalog対象Artifactの役割であり、二軸を相互導出しない。`production_owner_ref`はowner layerとexact一致し、Core→Pack／Project reverse dependency規則を満たす。Reference Gameを`production_owner_layer`として追加せず`artifact_role=reference_game`で表し、Fixture／Qualification／cross-cutting Control Planeも同様にProduction所有層から分離する。Provider／Model／Host／Deploymentは独立四集合で、各full tupleのConformance Receiptがある組合せだけを利用できる。brand、model family、Host表示名から別schema branch、Operation、Target、Authorityを生成しない。

Pack Discoveryの旧八actionは完全なMCD登録を持たないため、current Operation set、Manifest、Service allowlist、Provider／MCP Catalogを空にし、Capabilityを`not_activated`とする。旧`operation.packs.*`名をcurrent identityまたはlegacy aliasとして読まない。future vocabularyは`future.packs.action.*`だけに置き、future work item `activation.pack.ai_operations.v1`が採用するexact set、named input／Result／Receipt、Service／Policy／Validator／Diagnostic、Risk、intent DAG、private-to-public publication、Qualificationを同じContract set transactionで閉じるまで要求をfail closedに拒否する。

[Executable Contracts](../../architecture/02-foundation/executable-contracts.md) §§20～21.1の191候補（当初Discovery／Execution 67件＋回収済みDomain authoring／selection 92件＋AI E2E closure 32件）は24の`PlannedOperationFamilyV1`に属し、各familyのcurrent MCD／Manifest／Service／Policy／Validator／Diagnostic／Receipt／Provider／MCP Tool／alias集合はすべて明示的な空配列、stateは`not_activated`である。current contract-active 10は`GenericCoreOperationBaselineV1`のProject State bootstrap六、Installed ProductのCore extension World一、Genre Pack Shooter三であり、current Signer Policyが空なのでoperationalは0である。conditional legacy migration四、example／pending／rejected十一と互いにdisjointで、family固有work itemまたはlegacy evidence gateによるatomic Activation以外の部分登録を許可しない。

```text
AiTaskContextCapsuleV1
  schema_version: 1
  capsule_id: StableId
  task_authorization_ref: TaskAuthorizationEnvelopeRefV1
  game_brief_ref: GameBriefRefV1 | null
  game_spec_ref: GameSpecDocumentRefV1 | null
  selected_catalog_entries[1..256]:
    catalog_entry_ref: AiCatalogEntryRefV1
    selection_reason: Text<1024>
  project_slice_refs[0..1024]: TypedArtifactRefV1
  dependency_refs[0..512]: TypedArtifactRefV1
  constraint_refs[0..256]: RequirementRefV1(kind=constraint)
  target_profile_refs[0..64]: TargetProfileRefV1
  provider_profile_refs[0..64]: InferenceProviderProfileRefV1
  model_profile_refs[0..128]: InferenceModelProfileRefV1
  host_profile_refs[0..64]: EngineAiHostSecurityProfileRefV1
  deployment_profile_refs[0..64]: InferenceDeploymentProfileRefV1
  budget_refs[1..64]: BudgetProfileRefV1
  diagnostic_refs[0..256]: DiagnosticCodeRefV1
  allowed_operation_refs[0..256]:
    McdContractRefV1(kind=operation,status=active)
  source_ref_set_hash: Sha256
  field_mask_paths[1..1024]: CanonicalJsonPointer
  token_estimate: uint32
  omitted_ranges[0..256]:
    source_ref: TypedArtifactRefV1
    json_pointer: CanonicalJsonPointer
    start_index: uint32
    end_index_exclusive: uint32
  continuation:
    complete: {}
    | truncated:
        continuation_ref: AiContextContinuationRefV1
        remaining_token_estimate: uint32
  created_at: UtcTimestamp
  expires_at: UtcTimestamp
  content_hash: Sha256
```

`selected_catalog_entries[]`はCatalog ref順のstrict sorted setで、同じentryを理由違いで重複させない。Brief／Specを含む全source refを共通ref順にsortし、各canonical ref bytesをuint32 length-frameした連結を`source_bytes`とすると、`source_ref_set_hash = SHA-256(ASCII "MIRAKAN_AI_CONTEXT_SOURCE_REF_SET_V1" || uint32_be(source ref count) || uint32_be(length(source_bytes)) || source_bytes)`である。Project Slice取得順やtokenizer順を入れない。`field_mask_paths[]`はRFC 6901 escapeを一意化したUTF-8 byte順strict sorted setで、omitted rangeは`0 <= start_index < end_index_exclusive`とする。`continuation=complete`ではomitted range exact `[]`、`truncated`では1件以上であり、continuation refは同じTask Authorization、source set hash、field mask、expiryへ解決する。

Capsuleはread-only／Disposable projectionで、Project Source、Authorization、Approval、Receipt、CatalogまたはTask stateのAuthorityではない。`allowed_operation_refs[]`はTask Authorization allowlist、選択Catalog entryのactive Operation集合、current Caller／Service projectionの積集合だけをGatewayがtyped refとして生成し、自由文、選択理由、Provider／Model／Host／Deployment名から補完しない。AIへ全Schema、全Project、全設計書、Security署名内部を一括送信せず、Gatewayがsemantic projectionを返してAuthorization、Provenance、Receipt検証を内部で強制する。

Model familyまたはHost brandをEngine機能分岐にしない。同じMCD／Operationへ適合するProvider／ModelをProfileとConformance Receiptで有効化する。Strict Tool Callingへ適合しないModelはread-onlyまたはproposal-onlyに制限し、自然言語をCommit Operationへ推測変換しない。

### 8.1 Contract set、request、Commitの非循環DAG

`ContractSetSnapshotV2`だけをcurrent正本とし、closed local-member unionはMCD、Diagnostic、Trusted Service、Validator、Operation Validator Closureの全normative recordを持つ。Snapshot内部はkind別`ContractSetLocalRefV1`とself-excluding local hashだけを使い、Operation／Service allowlist／Validator closure／Diagnosticの相互edgeに外部set rootやrecord hashを入れない。set root preimageはmember kind、kind固有local identity、local hashをkind→ID→versionのcanonical byte順でstrict sortする。生成順は`LocalRef解決 → self-excluding local record hash → closed member set root → external McdContractRef／DiagnosticCodeRef／TrustedServiceRef／ValidatorRecordRef／closure ref materialization`である。Operation表はroot確定後のexternal projectionで、compiler inputは全intra-set refをkind別LocalRefへ機械投影し、Field／cardinality／logical identityのset equalityをfixtureで検証する。

Operation認可は`MIRAKAN_OPERATION_INTENT_V2 -> MutationAuthorizationBindingV2 -> MIRAKAN_OPERATION_REQUEST_V2`の一方向DAGだけを使う。intent payloadは選択named input type、Operation、Risk、全semantic input Fieldを持ち、`operation_intent_hash`、final `request_hash`、authorization binding全体を除外してself-excluding hashを作る。Task Authorizationはrequester、Task／Operation／Project descendant／typed subject／Path／Budgetのscope上限を署名し、まだ存在しない個別intent hashをsubjectにしない。quorumを満たすOperation Approval SetまたはPolicy ServiceがGrant消費HeadをCASして発行するPredelegation Useだけが同じexact intent hashをsubjectにし、final request hashを参照しない。binding確定後、選択inputの`request_hash`だけを除く全実在Fieldからfinal hashを作る。Task Authorization、Algorithm binding、current Control Plane Baseline bindingのFoundation Closureはbyte equalityにし、retained／stale Closureへ差し替えない。request Algorithm recordはExecutable Contractsの完成`McdCanonicalBinaryV1` profile/version/hashを束縛し、bare serializer名を許可しない。Project-boundはexact Project ref、projectlessはworkspace／catalog／resource等のOwner登録refを使い、sentinel／null Projectを捏造しない。conditional legacy migrationだけはdestination Input schemaがexact required `LegacyMigrationInventoryRefV1`、`source_foundation_definition_closure_ref`、`retained_source_mcd_ref`を持ち、source MCDおよびInput-reachable Contribution／Prepared Receiptの同名Fieldだけを一つのretained source Closureへ、destination execution refをdestination Closureへ解決し、両rootをintentへ含める。Root Scene、Runtime Scope、Performance Scale、Physics Intent Roleの四候補はcurrent全投影集合`[]`で、実Inventory、source Closure resolution、Input–Contribution–Receipt equality、wrong-root／alias／destination-downgrade fixtureを同じFoundation predicateで満たすatomic activationまでdispatchしない。循環shapeのcompat readerを作らず、証明済みretained source recordだけをcandidate入力として扱う。

mutationは[Executable Contracts §8](../../architecture/02-foundation/executable-contracts.md#8-operation定義)を唯一のschema／hash正本とする`immutable final request → OperationAuthorizationAuditBindingV1 → Preview／Validation → per-receipt PublishedReceiptMaterializationPolicyV1／Context → PreparedCandidate／PreparedCommitEnvelope → staged postcondition → private durable commit marker read-back → secret-free PublicCommitClosureV1 candidate → canonical PublishedDomainReceiptV2／MirakanSignedRecordV1 wrapper → PublicCommitClosureV1＋PublicPublicationMarkerV1＋public state／Resultのatomic CAS`という一方向DAGである。private Markerまでは外部current stateを変更しない。Project ChangeSet CommitはClosureの`domain_commitment.kind=project_change_set_commit`へexact `project_change_set_ref`／`candidate_root_sha256`を必須化し、Source primitive 0件でもbranchを残す。その他はregistered Ownerとreceipt-free committed Artifactだけを持つ`owner_typed_state_commit`を使う。Closureのsemantic hashと完成object SHAを分離し、Marker／signed Receiptの隣接hashは後者へ固定する。Materialization PolicyはMCDまたは同PolicyがpinするGovernance Registryへ登録せず、completed object SHAとsemantic hashを分離したcontent-addressed Refにする。Governance Policy ConfigのReceipt required subsetはSigner一row＋Verification Retention／Deterministic Recovery／Store Namespace三rowのexact四rowで、Contextは発行時Trust Head／closure、四row、Signer／Key／revocation、final requestの`MutationAuthorizationBindingV2`とbyte equalityなAuthorization Audit Bindingをpinする。signed wrapperを保存／readbackした後だけ同じClosure body、Public Marker、after stateを一つのpublic CASで発行し、secret-free historical verification graphとAudit ref／hash commitmentを到達可能化する。request／Authorization／ApprovalのPII bodyはimmutable access-controlled CASへ分離し、public inlineしない。unsigned payload、private Marker、Closure candidate store、receipt-store単独存在をauthorityにせず、requestへAudit／Contextを戻す固定点、Materialization PolicyのRegistry自己循環、postconditionがClosure／Marker／公開Receiptを読む循環、Markerがsigned Receipt／Public Markerをhashする循環を禁止する。Closure関連named typeは既存Receipt／Marker root schemaのnested common `$defs`であり、current active Operation 10件、MCD Type／Contract Set member件数を増やさない。

`PreparedCandidateV1`はID、schema、Staging root、before／proposed-after state、prepared Artifact集合からself-excluding content hashを作り、外部`PreparedCandidateRefV1`を完成後にmaterializeする。private Marker後／wrapper保存前のcrashは同じprivate preimageから同じsemantic hash／完成object SHAのClosure candidateを再現またはreadbackし、固定materialization key、issued-at、revocation snapshot、key context、deterministic signing profileから同じwrapper bytesへroll-forwardする。wrapper後／Public Marker前は同じClosure body＋Marker＋after stateを同じexpected predecessorへpublic CASする。既存Closure／wrapperはbyte equalityで再利用し、別Closure、alternate signature、二重publication、overwrite、public後rollbackを禁止する。MarkerなしPrepared artifactは非公開廃棄する。

R2 mutation inputは`MutationAuthorizationBindingV2`の`approval_set | predelegated`を厳密に一つ、R3以上はapproval setだけを持つ。current Projector bindingはexact空なので、atomic ActivationまではR2もdirect Approval Setだけを使う。欠落／expired／quorum不足／scope mismatchは全Domainで`MIRAKAN-APPROVAL-REQUIRED`とし、Operation error、Validator reachable error、Manifest inventoryをset equalityにする。

Input roleはclosed MCD kind `type`の`SemanticActionRoleRefV1`で表し、未定義kindやbare文字列を受理しない。owner-typed `SemanticActionCommandBindingContributionV1`をcanonical mergeしてDerived `SemanticActionCommandBindingRegistryV1`を再materializeし、Production recordはowner Qualification Receiptだけを参照する。Project Sourceの`SemanticActionBindingSelectionDocumentV1`を変更するOperationは本Taskで完全登録されていないためcurrent setを空、Capabilityを`not_activated`とする。Compile／Save／Replayはcurrent Sourceが存在する場合にSelection Document、Action Map、RegistryRef、全RecordRef、role type、Command Systemをexact closureとして保存し、reload後のSource→Registry→Record equalityを再検査する。

## 9. Product ScopeとQualification

現時点で未実装のPlatform／大規模機能を、汎用Schemaが存在することだけで対応済みと表示しない。offline single-player、Windows-first、60 Hz baseline等の現在Scopeは許容するが、恒久Core制約にしない。

性能正本は`ProjectScaleEnvelopeV2`で、`WorkloadDomainTypeRegistryV1`とowner-typed `WorkloadDomainIntentV1`を束ねる。World／spatialは一件でも`required`なら必須、全domainが`forbidden`なら禁止、`optional`だけならnull／exact World intentの双方を許可する。UI-only、strict headless、tooling、resource-onlyは偽World／Gameplay floorなしでvalidである。semantic modeは`authoritative_equivalence | presentation_fidelity | functional_contract | resource_slo | none`のtagged ruleで、`none`は登録済みtool／resource domainだけに許可する。旧V1は完全なmigration Operation／Prepared payload／Receipt／fixtureでV2へ移し、current Source／projectionへ残さない。Qualificationは全owner dimensionを同時発生させ、未登録dimension、schema／unit／hash不一致、必要なequivalence／fidelity／functional／SLO欠落を開始前に拒否する。

最低Reference／holdout集合を次へ固定する。

- Shooter top-down 2D: 最初のGenre vertical slice
- Shooter third-person 3D: 同じFeature契約の3D検証
- Puzzle／Dialogue 2D: Combatなし
- Platformer 2D: Character Locomotionの別Genre検証
- Turn-based zero-character: Character、Physics、Pack-owned completionを要求しない
- Endless continuous simulation: Completion／Resultなし
- UI-only／tool-like: Worldなし
- Headless simulation: Surface／UIなし

Phase 4 AI MVPはShooterだけで汎用性を主張しない。少なくともCombatなし、zero-characterまたはWorldなしの中立Fixtureを一つ同じAI／手動往復Gateへ含める。`general_production_2d`または`general_production_3d`は複数Genre／構造のQualification後だけ表示する。

Network transport、replication、authority、rollbackはGenre-neutral Future Capabilityとする。`small-cooperative-multiplayer`と`rollback-competitive-networking`はShooter Capabilityへ依存しない。`large-session-network-shooter`だけがShooter Genreをconsumerとして参照できる。

Release closureはTarget単位に評価する。Windows／Android／Apple／headlessの個別Releaseを別Target未完了で停止させない。全Target coverage labelは個別Releaseとは別のPortfolio projectionとする。

## 10. 必須Conformance Gate

1. Shooter Packを未installまたは削除した状態でCore、Editor、AI、Project C++、Project Shader、Test、Build、Packageが成功する。
2. exact `ArchitectureDependencySourceSetV1`から再生成したRegistryがOwner 103／Edge 516、class別`248 / 193 / 3 / 72 / 0`、runtime-required 125、production graph 64 node／126 pairと一致し、Generic Core baselineおよびCore／Feature rootからShooter ownerへの`production_dependency`到達が0件である。`qualification_input`、`installed_product_composition`、`cross_cutting_control_plane`、`documentation_reference`は別classとして検査し、Core dependency件数へ加算しない。
3. Genre PackなしでFeature Packだけを使うProjectがvalidである。
4. UI-only ProjectがWorld／spatial contentなし、headless Projectがsurfaceなしでvalidである。
5. finite、endless、turn-based、continuous simulation、tool-like、zero-characterをGame Brief／Specで表現できる。
6. Path FollowingがCharacter Motor以外のconformance stubへ接続できる。
7. C1は60/1だけをqualifiedに保ちながら、Core schema変更なしに別Cadence Profileを追加できる。
8. Pack AI Operationが未Activationの間はPack追加／更新／削除要求をfail closedに拒否し、将来activation時は依存、影響、migration、fallbackをexact IDで説明できる。
9. Local小型Modelを含むConformanceで、未知IDの最終提出0、`question_required`推測回避0、禁止Operation提出0を満たす。
10. Target-local Release closureと全Target Portfolio labelが別判定になる。

## 11. 公式設計から採用する原則

- [Unreal Engine Modules](https://dev.epicgames.com/documentation/unreal-engine/unreal-engine-modules)／[Plugins](https://dev.epicgames.com/documentation/unreal-engine/plugins-in-unreal-engine)のProject-owned module、Runtime／Editor分離、enable／disable可能な依存単位。
- [Unity Packages](https://docs.unity3d.com/6000.0/Documentation/Manual/Packages.html)／[Assembly Definitions](https://docs.unity3d.com/6000.0/Documentation/Manual/assembly-definitions-intro.html)のmanifest、dependency、version、Target別compile、再利用可能なassembly境界。
- [Godot Nodes and Scenes](https://docs.godotengine.org/en/stable/getting_started/step_by_step/nodes_and_scenes.html)のcompositionと、[Godot 4.5 GDExtension](https://docs.godotengine.org/en/4.5/tutorials/scripting/gdextension/what_is_gdextension.html)のEngine再compileを要求しないProject native extension境界。
- [O3DE Gems](https://docs.o3de.org/docs/user-guide/gems/)のcode／asset package、Projectごとの選択、custom feature／gameplay logic追加。
- [Unreal MCP](https://dev.epicgames.com/documentation/unreal-engine/unreal-mcp-in-unreal-editor)が示すtyped parameter／structured result、小さく単一責務のTool、client非依存MCP接続。ただし同機能は公式上もExperimentalでAPI／data formatが変更され得て、loopback既定・認証なし・remote非推奨であるため、MiraikanaiのProduction依存またはSecurity根拠にはしない。

外部EngineのAPI、class hierarchy、serialization、package format互換は採用しない。MiraikanaiのMCD、Stable ID、typed Operation、Gateway、Receiptを正本とする。

上記公式資料は2026-07-24に再確認した。採用するのは分離、composition、manifest、typed Toolという設計原則であり、他Engineの実装済み機能をMiraikanaiの実現Receiptとして流用しない。

## 12. 完了判定

本設計の文書反映だけではEngine実装、Capability Activation、Shippingを証明しない。Architecture正本、Product Registry、current execution plan、AI Eval、Reference Fixture、依存検査が本設計へ同期し、旧identity／旧必須前提へのactive参照が0件になった時点で「計画書更新完了」とする。判定は引き続き`planning_go / implementation_and_shipping_no_go`である。
