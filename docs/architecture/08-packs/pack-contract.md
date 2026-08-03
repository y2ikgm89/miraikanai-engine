# Miraikanai Engine Pack Contract

- 文書ID: mirakan.arch.pack-contract
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Packの4層境界、`PackManifestV1`、source／publisher／distribution trust、Feature／Genre依存、Profile ownership、acquire／install／apply／update／removal、承認済みCompatibility ChangeのPack transaction適用、action-specific Pack lifecycle acceptance、Project／Registry／installed closureのbefore／after lineage、last-valid規則
- 非正本範囲: Product roadmap／Capability成熟度、Product-wide Privacy／Security／license判断、Store／Marketplace運営、migration class／source-target compatibility／reader-writer期間／rollback eligibility、各FeatureのPublic Contract、Genre固有composition、共有MCD／ChangeSet、Subsystem契約は各Ownerを参照
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Product Plan](../00-product/product-plan.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[Project State](../03-authoring/project-state.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)、[Executable Contracts](../02-foundation/executable-contracts.md)
- 関連文書: [Product Plan](../00-product/product-plan.md)、[Product Lifecycle](../00-product/product-lifecycle.md)、[Product Security](../01-governance/product-security.md)、[Product Privacy／Data Governance](../01-governance/product-privacy-data-governance.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Project State](../03-authoring/project-state.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)、[Native Game Module](../03-authoring/native-game-module.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-29

## 1. 結論

Miraikanai Engineの製品構造は次の4層である。

1. `Generic Engine Core`
2. `Reusable Feature Packs`
3. `Genre Packs`（任意）
4. `Game Projects`

`Reference Games`は第5のProduction Runtime層ではない。通常のGame Projectとして選定された検証対象であり、Fixtureと同様にProduction artifactから参照されない。

依存方向は利用側から提供側への一方向とする。

```text
Game Project
  -> optional Genre Pack
  -> Feature Pack
  -> Generic Engine Core
```

Game ProjectはGenre Packを使用せず、Feature Packを直接構成できる。[Product Plan](../00-product/product-plan.md)が採用するcompact 2D RPGと3D technical Referenceはinitial V1のProduct-facing Reference選択であり、Core、Editor、AI、Project C++、Project Shader、Test、Build、Package、Network、ReleaseへRPGまたはShooter Feature／Genre Pack依存を追加しない。Shooterは独立したoptional Genre Packおよびtechnical stress fixtureであり、RPGのpredecessor、Product migration source、Release Receipt sourceではない。ProductのRPG、Shooter、Platformer、Puzzle-Dialogue等のGateはbundled Reference Game coverageを資格化するnonblocking trackであり、Generic EngineのRelease Gate、C++23 Header Shippingまたはproduction-release bindingから参照しない。Reference選択はProduct Planのinitial V1 definitionから直接決まり、旧Installed Product、source／destination migration、Pack ID renameまたはReceipt流用を前提にしない。

## 2. 層と依存規則

| 利用側 | 許可する依存先 | 拒否する依存先 |
|---|---|---|
| Generic Engine Core | Core内の下位契約 | Feature Pack、Genre Pack、Game Project、Reference Game、Fixture |
| Feature Pack | Generic Engine Core、別Feature Pack | Genre Pack、Game Project、Reference Game、Fixture |
| Genre Pack | Generic Engine Core、Feature Pack | 別Genre Pack、Game Project、Reference Game、Fixture |
| Game Project | Generic Engine Core、Feature Pack、任意のGenre Pack | Reference Game、Fixture |
| Reference Game／Fixture | 検証対象のCore、Pack、Project | Production artifactへの逆向き依存 |

Feature Pack間依存だけを有向非巡回グラフ（DAG）として許可する。resolverは自己依存、重複edge、cycle、欠落Pack、互換versionの空intersection、同一Pack identity／versionの異なるhashを拒否する。Genre Pack同士の直接依存は、cycleの有無にかかわらず拒否する。複数Genreの合成はGame Projectが明示的に行い、Genre Pack間の隠れた推移依存にしない。

Profileは独立Packではない。所有Pack artifact内の構成単位であり、独立したmanifest、install state、registry entryを持たず、所有Packの`pack_version`と`content_hash`に含まれる。

## 3. `PackManifestV1`

`PackManifestV1`は次のcanonical fieldを一つずつ持つ。`pack_kind`は`feature | genre`のclosed enumである。

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
  authoring_operation_refs[]:
    McdContractRefV1(kind=operation)
  runtime_port_refs[]
  configuration_profile_refs[]
  composition_recipe_refs[]
  source_template_refs[]
  validator_refs[]:
    exact {validator_id, validator_version, validator_content_hash}
  test_scenario_refs[]:
    exact {fixture_id, fixture_version, fixture_content_hash}
  example_refs[]
  counterexample_refs[]
  ai_vocabulary_refs[]
  ai_planning_recipe_refs[]
  performance_profile_refs[]
  migration_step_refs[0..4096]:
    exact {migration_id, migration_version, migration_content_hash}
  publisher_identity_ref: exact PackPublisherIdentityRefV1
  source_identity_ref: exact PackSourceIdentityRefV1
  requested_permission_refs[]:
    sorted unique exact PackPermissionRequestRefV1
  license_ref: exact LicenseArtifactRefV1
  provenance_ref

PackLocalIdentityV1
  pack_id
  pack_version

PackContractRefV1
  pack_id
  pack_version
  content_hash
```

`pack_id`はPackの論理identity、`pack_version`はSemVer、`content_hash`は自己Fieldを除くcanonical manifestと同梱payloadのcontent identityである。同じ`pack_id`と`pack_version`に異なる`content_hash`を受理しない。全参照は上記version／hashを含むexact identityへ解決し、arrayは参照identityのcanonical orderでserializeする。unknown、missing、duplicate、曖昧な表示名、`latest`の暗黙選択を拒否する。Pack artifactに含まれるOperation、Compatibility Ownerが承認したmigration step、Validator、test scenarioは一件残らず対応inventoryに列挙し、payload探索や命名prefixで補完しない。initial V1の全Pack Manifestは`migration_step_refs=[]`を必須とし、materialized source versionと承認済みCompatibility Changeが存在する公開後のversionだけがnon-empty集合を持てる。

`minimum_engine_contract_ref`は利用可能性の証拠ではない。active TargetでEngine contract、required Capability、Feature Pack closure、Validator、Test、root外Recipe Activation Bindingが指すQualification Receiptがすべて解決して初めてPackを適用できる。

`validator_refs[]`と`test_scenario_refs[]`はPack artifact内のinventory、identity resolution、owner／hash検査の一覧であり、列挙した全Validator／Fixtureを全Recipe共通の実行gateにしない。Pack installは全Receipt-free recordのschema、owner、version、hashを検証するが、Project apply／Cookの実行gateは選択Recipeの`validator_refs[]`とroot外`PackRecipeActivationBindingV1`が指すexact signed Qualification Receiptだけである。Fixture bodyは別owner-typed Qualification subjectだけが解決し、Production Recipe／Runtime Packageは解決しない。Manifest inventoryに存在しても未選択Recipe専用Validator／Fixture、その条件Capability、依存Packをactive closureへ追加しない。

### 3.0 Source、publisher、distribution trust

Pack artifactのcontent integrity、publisher identity、取得元、license、permission、qualificationは別subjectであり、signature成功だけから相互に推測しない。

```text
PackOriginV1
  pack_origin_id: StableId
  pack_origin_version: 1
  origin:
    { kind: bundled,
      bundle_artifact_ref: exact ArtifactRefV1(
        artifact_kind=bundled_pack_origin, schema_version=1) }
    | { kind: first_party_repository | external_repository,
        canonical_repository_uri_utf8: normalized absolute URI }
    | { kind: approved_registry,
        canonical_registry_uri_utf8: normalized absolute URI }
    | { kind: local_artifact,
        local_artifact_ref: exact ArtifactRefV1(
          artifact_kind=local_pack_origin, schema_version=1) }
  pack_origin_content_hash: SHA-256

PackOriginRefV1
  pack_origin_id: StableId
  pack_origin_version: 1
  pack_origin_content_hash: SHA-256

PackTransportSecurityPolicyV1
  transport_policy_id: StableId
  transport_policy_version: 1
  transport_kind: no_network | authenticated_https
  redirect_policy: deny | same_origin_only
  maximum_redirect_count: u8 in 0..8
  network_address_scope: none | public_only
  artifact_hash_required: true
  signature_verification_required: true
  transport_policy_content_hash: SHA-256

PackTransportSecurityPolicyRefV1
  transport_policy_id: StableId
  transport_policy_version: 1
  transport_policy_content_hash: SHA-256

PackIndexSnapshotV1
  index_snapshot_id: StableId
  index_snapshot_version: 1
  pack_source_id: StableId
  pack_source_version: positive u32
  retrieved_at_utc: RFC 3339 UTC
  source_sequence: positive u64
  pagination_complete: bool
  index_entries_artifact_ref: exact ArtifactRefV1(
    artifact_kind=pack_index_entries, schema_version=1)
  entry_count: u32
  entry_set_root_hash: SHA-256
  index_snapshot_content_hash: SHA-256

PackIndexSnapshotRefV1
  index_snapshot_id: StableId
  index_snapshot_version: 1
  index_snapshot_content_hash: SHA-256

PackSigningIdentityV1
  signing_identity_id: StableId
  signing_identity_version: positive u32
  publisher_id: StableId
  signer_subject_ref: exact TrustSubjectRefV1
  signer_role_ref: exact TrustRoleRefV1
  signing_key_id: StableId
  public_key_content_hash: SHA-256
  allowed_pack_scope_content_hash: SHA-256
  signing_identity_content_hash: SHA-256

PackSigningIdentityRefV1
  signing_identity_id: StableId
  signing_identity_version: positive u32
  signing_identity_content_hash: SHA-256

PackSignatureVerificationV1
  signature_verification_id: StableId
  signature_verification_version: 1
  pack_contract_ref: exact PackContractRefV1
  artifact_ref: exact ArtifactRefV1
  signing_identity_ref: exact PackSigningIdentityRefV1
  signature_algorithm_profile_ref:
    exact McdContractRefV1(kind=profile)
  signature_artifact_ref: exact ArtifactRefV1(
    artifact_kind=pack_signature, schema_version=1)
  signature_bytes_sha256: SHA-256
  decision: valid | invalid
  verified_at_utc: RFC 3339 UTC
  diagnostic_refs[0..64]: sorted unique exact DiagnosticCodeRefV1
  signature_verification_content_hash: SHA-256

PackSignatureVerificationRefV1
  signature_verification_id: StableId
  signature_verification_version: 1
  signature_verification_content_hash: SHA-256

PackLicenseReviewV1
  license_review_id: StableId
  license_review_version: 1
  artifact_ref: exact ArtifactRefV1
  license_declaration_ref: exact ArtifactRefV1(
    artifact_kind=pack_license_declaration, schema_version=1)
  distribution_scope: development_only | project_distribution | public_distribution
  decision: approved | rejected
  review_evidence_ref: exact EvidenceRefV1
  diagnostic_refs[0..64]: sorted unique exact DiagnosticCodeRefV1
  license_review_content_hash: SHA-256

PackLicenseReviewRefV1
  license_review_id: StableId
  license_review_version: 1
  license_review_content_hash: SHA-256

PackPermissionReviewV1
  permission_review_id: StableId
  permission_review_version: 1
  artifact_ref: exact ArtifactRefV1
  requested_capability_refs[0..256]:
    sorted unique exact McdContractRefV1(kind=capability)
  approved_capability_refs[0..256]:
    sorted unique exact McdContractRefV1(kind=capability)
  security_review_evidence_refs[0..64]:
    sorted unique exact EvidenceRefV1
  privacy_review_evidence_refs[0..64]:
    sorted unique exact EvidenceRefV1
  decision: approved | rejected
  diagnostic_refs[0..64]: sorted unique exact DiagnosticCodeRefV1
  permission_review_content_hash: SHA-256

PackPermissionReviewRefV1
  permission_review_id: StableId
  permission_review_version: 1
  permission_review_content_hash: SHA-256

PackPublisherIdentityV1
  publisher_id: StableId
  publisher_identity_version: positive u32
  display_identity_artifact_ref: exact ArtifactRefV1
  signing_identity_refs[1..16]:
    sorted unique exact PackSigningIdentityRefV1
  support_channel_refs[1..16]:
    sorted unique exact ProductSupportChannelRefV1
  privacy_disclosure_refs[0..16]:
    sorted unique exact PrivacyDisclosureEntryRefV1
  publisher_identity_content_hash: SHA-256

PackSourceIdentityV1
  pack_source_id: StableId
  pack_source_version: positive u32
  source_kind: bundled | first_party_repository | approved_registry
    | local_artifact | external_repository
  canonical_origin_ref: exact PackOriginRefV1
  transport_security_policy_ref: exact PackTransportSecurityPolicyRefV1
  index_snapshot_ref: null | exact PackIndexSnapshotRefV1
  source_identity_content_hash: SHA-256

PackAcquisitionBindingV1
  acquisition_id: StableId
  acquisition_version: positive u32
  pack_contract_ref: exact PackContractRefV1
  publisher_identity_ref: exact PackPublisherIdentityRefV1
  source_identity_ref: exact PackSourceIdentityRefV1
  acquired_artifact_ref: exact ArtifactRefV1
  signature_verification_ref: exact PackSignatureVerificationRefV1
  license_review_ref: exact PackLicenseReviewRefV1
  permission_review_ref: exact PackPermissionReviewRefV1
  provenance_evidence_ref: exact EvidenceRefV1
  acquired_at_utc: RFC 3339 UTC
  acquisition_binding_content_hash: SHA-256

PackAcquisitionBindingRefV1
  acquisition_id: StableId
  acquisition_version: positive u32
  acquisition_binding_content_hash: SHA-256
```

Manifestのpublisher／sourceはAcquisition Bindingとbyte equalityにする。同じPack ID／version／content hashを別publisherまたは別sourceから取得した場合も別Acquisitionとしてreviewし、以前のtrust decisionを流用しない。redirect、mirror、cache、offline copyはcanonical originを隠さず、最終artifact hashと署名を再検証する。

`PackOriginV1.origin.kind`は`PackSourceIdentityV1.source_kind`とbyte equalityにし、bundled／localは`transport_kind=no_network, redirect_policy=deny, maximum_redirect_count=0, network_address_scope=none`、repository／registryは`transport_kind=authenticated_https, network_address_scope=public_only`を必須にする。Network originのcanonical URIはuserinfo、fragment、non-HTTPS scheme、loopback、link-local、private／reserved addressへの解決を拒否し、redirect後にも同じ検査を繰り返す。`pagination_complete=false`のIndex Snapshot、source identityと異なるsource ID／version、entry artifactのcount／root不一致、rollbackした`source_sequence`または同sequence別entry rootをinstall／update選択に使わない。

Pack Signing Identityはpublisher、AI Security OwnerのTrust Subject／Role、Trust Key Registryのglobally unique key ID、public key bytes、許可Pack scopeを一つのrecordへ束縛する。署名検証時はkey IDからcurrent `TrustKeyV1`をexact一件解決し、Subject／Role／public key hash、RoleとKeyのexact detached purpose `pack_artifact_signature`、有効期間／revocationとSigning Identityを照合する。署名preimageはexact Pack Contract Refと完成Pack artifact ref／content hashをalgorithm Profileのclosed canonical encodingで束縛し、Signature VerificationとAcquisition BindingのPack／artifact refをbyte equalityにする。表示publisher名、certificate subject文字列、同じpublic key bytesまたは過去key versionからIdentityを補完しない。

Index entries artifactはsourceが返した全entryのclosed recordsをcanonical sortした完成集合で、`entry_count`を配列長、`entry_set_root_hash`を完成artifact bytesのSHA-256と一致させる。Signature、License、Permission recordは同じ`acquired_artifact_ref`へexact解決し、BindingのPack contractが指す完成artifact hashとbyte equalityにする。Signature artifact bytesのhashは`signature_bytes_sha256`と一致させ、Signatureの`valid`はlicense／permission／publisher trust／qualificationの承認ではない。Licenseの`approved`は宣言された`distribution_scope`だけに有効で、Development承認をpublic distributionへ昇格しない。Permissionの`approved`ではrequested集合とapproved集合をexact set equalityにし、`rejected`ではapproved集合をemptyにする。Signature／License／Permissionの成功DecisionはDiagnostic集合empty、失敗Decisionは一件以上を必須とする。security／privacy Evidence要件は要求CapabilityのOwner predicateから導出し、空集合を包括承認として扱わない。全content hashは対応する型名のASCII domain separatorと自己hashを除くclosed MCD canonical bytesから計算し、各Refは完成recordへexact解決する。

本節の型はPack trust正本のtarget contractであり、現RepositoryにSchema、Registry、Index、署名検証器、Review、EvidenceまたはAcquisition Bindingがmaterializeしていることを意味しない。不在中はexternal／registry Packのacquire、install、update、applyをfail closedにし、URL、publisher表示名、署名ファイルの存在または文書上の承認からrecordを生成しない。

`source_kind`はtrust levelではない。first-party、bundled、approved registry、署名済みであっても、Engine contract、Target、Capability、license、Privacy、Security、qualificationのGateを免除しない。外部repository URL、search result、display name、download count、rating、同名publisherをidentityとして扱わない。

Permission requestはPackがProject／Runtimeで必要とするnetwork、telemetry、AI、filesystem、device、native code、credential、Player data等のCapabilityをclosed refで宣言する。未宣言accessをdenyし、install時の包括承認を将来の新permissionへ継承しない。permission変更はPack updateのbreaking impactとしてPreviewし、Product SecurityとProduct Privacyのreview Evidenceを必要とする。

Pack discovery／download／update check自体のnetwork flow、account、telemetry、region、retentionは[Product Privacy／Data Governance](../01-governance/product-privacy-data-governance.md)へ登録する。Pack publisherのPrivacy Policy、license、signatureをEngine Product consentまたはGame Player consentとして扱わない。

Pack activation transactionは`PackManifestV1.authoring_operation_refs[]`、active MCD Contract set内でownerが当該PackのOperation LocalRef集合、`TrustedServiceLocalRecordV1.allowed_operation_local_refs[]`へ当該Packが寄与する集合の三者をID／versionでset equalityにする。missing／extra／duplicate／wrong kind／stale version／hash、Service allowlistだけのOperation、ManifestだけのOperationを一件でも検出したらset rootを発行しない。Pack removalも同じtransactionで三集合からexact subsetを除去し、別Pack／Coreのallowlistを変更しない。

Validator gateは異種集合を混ぜず、次の二equalitiesを独立に検査する。

1. `PackManifestV1.validator_refs[] = Validator Registryの当該Pack owner record subset`（Validator ID／version／content hash）。
2. 各Operationについて`OperationValidatorClosureV1.validator_refs[]が宣言するValidator error_refs[] union = closure.reachable_error_refs[] = McdOperationContractV1.errors[]`（Diagnostic ID／code／version／content hash）。

Manifest Validator inventoryをDiagnostic集合、closure reachable error集合をValidator inventoryと比較しない。一方だけの成功を他方へ読み替えず、missing／extra／duplicate／stale refを各gateで別Diagnosticにする。

### 3.1 `CompositionRecipeV1`

Pack全体で常時必要なFeature dependencyと、選択したcompositionだけが必要なFeature dependencyを分離する。`PackManifestV1.required_feature_pack_refs[]`は全Recipe共通のunconditional edge、`CompositionRecipeV1.required_feature_pack_refs[]`は当該RecipeをProjectへ適用する時だけ有効なconditional edgeである。

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

`recipe_hash`は自己Fieldを除くReceipt-free canonical recordのSHA-256であり、所有Packの`content_hash`へ含める。Recipeの`owner_pack_local_identity`は所有Manifestの`{pack_id,pack_version}`とbyte equalityにし、Pack `content_hash`、`PackContractRefV1`、Qualification Receipt／Bindingを含めない。Recipe、Profile、Template、Pack Manifestのhash preimageへQualification Receipt／Binding／Fixtureを含めない。全arrayはexact identityのcanonical orderとし、unknown、duplicate、self dependency、Genre Pack ref、Project／FixtureへのProduction dependency、version／hash conflictを拒否する。`validator_refs[]`はこのRecipe選択時だけapply gateになり、別RecipeのPerception、Stage、full-profile Fixture等を要求しない。Productionはroot外Activation BindingからReceiptのsubject／owner／Recipe／Target／result／freshnessだけを検証し、`PackRecipeQualificationSubjectV1.fixture_refs[]`を解決しない。`fallback_recipe_ref`は同じowner Pack内のRecipeだけを指し、fallback cycleを拒否する。fallbackは元RecipeのGameplay意味を暗黙変更せず、Projectが明示選択した時だけ別のdependency closureを解決する。

生成順は`Pack local identity → receipt-free CompositionRecipeV1／recipe hash → Pack Manifest content hash → PackContractRefV1／Recipe ref → PackRecipeQualificationSubjectV1 → signed Receipt → PackRecipeActivationBindingV1 → Project-owned Activation projection`である。subject hashはASCII `MIRAKAN_PACK_RECIPE_QUALIFICATION_SUBJECT_V1`、binding hashはASCII `MIRAKAN_PACK_RECIPE_ACTIVATION_BINDING_V1`、projection hashはASCII `MIRAKAN_PACK_RECIPE_ACTIVATION_PROJECTION_V1`と各自己Fieldを除くcount／length-framed canonical bytesから計算する。Subject `owner_pack_ref`のpack ID／versionはRecipe `owner_pack_local_identity`とbyte equality、content hashはそのRecipeをinventoryに含む完成Manifestの`content_hash`とbyte equalityにする。Binding Recipe pairはsubjectとbyte equalityでなければならない。PackContractRefをRecipe hash preimageへ戻さず、Receipt／Binding／ProjectionをRecipe、Profile、Manifest、Pack content hashへ戻さない。owner local ID/version、Pack hash、Recipe、Target、fixture、subject hash、signed hash、binding hashのstaleまたはsubstitutionを各一原因でrejectする。

選択Recipe `R`のeffective Feature closureは、所有Manifestの`required_feature_pack_refs[]`、`R.required_feature_pack_refs[]`、両集合から到達するFeature Pack DAGの和集合である。resolverは次を生成する。

```text
RecipeDependencyClosureV1
  selected_recipe_ref
  owner_pack_content_hash
  manifest_required_feature_pack_refs[]
  recipe_required_feature_pack_refs[]
  transitive_feature_pack_refs[]
  resolved_pack_version_and_hash_refs[]
  closure_hash
```

`closure_hash`は上記Fieldのcanonical serializationから計算し、Preview、Project ChangeSet、Cook、Qualification Receipt、Save／Replay headerへ同じ値を伝播する。Pack install時は全Recipe recordのschema、hash、参照kind、所有関係を検証するが、未選択Recipeのconditional dependencyをinstalled closureへ暗黙追加しない。Project apply／Cook時は選択Recipeのeffective closureを原子的に解決し、一件でもmissing、incompatible、unqualified、Target不適合なら`MIRAKAN-PACK-RECIPE-DEPENDENCY_UNRESOLVED`で拒否する。Project revision、active Recipe、registry head、installed closure、Cooked Artifactは変更せず、直前のlast-valid Recipe activationとclosure hashを維持する。partial Recipe applyとmissing Featureのplaceholderを禁止する。

### 3.2 Feature Pack

Feature Packは複数Genre／Projectで再利用する次の要素を提供する。

- Public Contract、Component Schema、Game System Spec
- Validator、Runtime Port、planned authoring action vocabularyまたは完全登録済みMCD Operation ref
- AI vocabulary、planning recipe
- Engine-ownedまたはProject-ownedのreference implementation
- contract fixture、example、counterexample、performance profile

Feature Packの`required_feature_pack_refs[]`は別Feature Packだけを指す。Feature PackはGenre固有Profile、Genre vocabulary、Game Project、Fixtureの内容を参照しない。

### 3.3 Genre Pack

Genre PackはFeature Packを組み合わせる次の要素だけを提供する。

- composition recipe
- Genre vocabularyとGameSpec template
- Genre固有configuration Profile
- reference scenarioとGenre fixture binding

Genre Packは新しい汎用Core契約を作らず、Feature CapabilityのPublic Contract、Schema、State owner、Runtime Portを複写しない。ManifestとRecipeの`required_feature_pack_refs[]`はFeature Packだけを指し、別Genre Packを指せない。条件依存をManifestのunconditional edgeへ昇格させず、使用するRecipe recordへ記録する。

## 4. Installとdependency resolution

Installは次の順で原子的に評価する。

1. artifact bounds、canonical manifest、`content_hash`、publisher、source、Acquisition Binding、license、provenance、signatureを検証する。
2. requested permission、Security／Privacy review、User approval、Target policyを検証する。
3. `minimum_engine_contract_ref`、Target intersection、required Capabilityを検証する。
4. Manifest共通Feature Pack DAGを解決し、Genre間dependency、cycle、version／hash conflictを拒否する。
5. Public Contract、Schema、Profile、`CompositionRecipeV1`、Template、Validator、Fixtureの全参照kind／hash／ownerをload前に検証する。未選択Recipeのconditional dependencyはinstalled closureへ追加しない。
6. 全検証成功時だけPack artifactとregistry revisionを一つのpublicationとしてcommitする。

失敗時はregistry、Project、installed Pack closureを変更せず、直前のlast-valid registry headとartifactを維持する。download済み未検証artifactは非authority quarantineとしてのみ保持できる。部分install、publisher／source置換、permission自動grant、依存の暗黙追加、Genre間のsynthetic edge、未有効Capabilityのplaceholderを禁止する。

Projectへの適用は[Project State](../03-authoring/project-state.md)のPreview／Commit／Undoを使う。PreviewはPack／version／hash、dependency closure、Target、生成または変更するDocument／Asset／Source、承認済みCompatibility Change、conflict、fallback、Qualificationへの影響をexact IDで表示する。Native source templateは[Native Game Module](../03-authoring/native-game-module.md)の別ChangeSet、review、build、promotionへ隔離する。

## 5. Updateとapproved compatibility change

Updateはinstalled base、Projectへ適用済みのinstance、現在の人間／AI変更を使う三者比較Diffで行う。Pack Contractはupdate transactionのatomicity、Preview、last-valid維持、Pack payloadへのstep適用だけを所有する。migration class、source／target version、complete consumer inventory、reader／writer期間、rollback eligibilityは[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)の承認済みCompatibility Changeだけが決定する。

`migration_step_refs[]`がnon-emptyの場合、各stepは一つのapproved Compatibility Change、exact source Pack version／hash、destination Pack version／hash、対象Source／Save／Replay、precondition、rollbackまたは明示的forward-only dispositionへ解決し、Compatibility Changeのordered step集合とset／order equalityにする。公開後のversion変更でPublic Contract、persisted Source、Save／Replay、ProfileまたはRecipeが変わる場合はCompatibility Ownerの判断と再Qualificationを要求する。Pack ContractがSemVer labelだけからclassを推測したり、offline migration、reader、shimまたはrollback eligibilityを独自決定したりしない。Runtime互換shim、削除済みidentityの再利用、旧名alias、Template再展開によるhuman override上書きを禁止する。

update transactionのいずれかが失敗した場合は新revisionをcommitしない。旧Pack version、旧registry head、Project revision、Save／Replay、last-valid Build／Package artifactを維持し、失敗したstepとremediationを返す。Package済みGameへnetworkからPackを暗黙更新しない。

## 6. Removal

Removal planは次をexact IDで列挙する。

- 依存するFeature Pack、Genre Pack、Game Project
- 適用済みTemplate instance、Source、Asset、Save／Replay field
- 削除または置換が必要なProfile、Recipe、Capability、Package
- 必要なapproved Compatibility Change、fallback、再Qualification

live dependencyが一件でも残る場合はremovalを拒否する。Genre Packの削除は共有Feature Packを所有推測で削除せず、別Project／Genreが参照するFeature Packを維持する。Project変更、Cook／Package再生成、registry mutationを別々のcommit境界にし、中間失敗時もlast-valid Projectとartifactを開ける状態を維持する。

Shooter Genre Packを未installまたは削除した状態でも、Core、Editor、AI、Project C++、Project Shader、Test、Build、Packageが成功しなければならない。

### 6.1 Pack lifecycle acceptance

Pack lifecycle stateとReceipt bindingのcontent hashは、それぞれASCII `MIRAKAN_PACK_REGISTRY_STATE_V1`、`MIRAKAN_INSTALLED_PACK_CLOSURE_V1`、`MIRAKAN_PACK_OPERATION_RECEIPT_BINDING_V1`、`MIRAKAN_PACK_LIFECYCLE_ACTION_BINDING_V1`、`MIRAKAN_PACK_LIFECYCLE_ACCEPTANCE_V1`と、自己hashだけを除くclosed count／length-framed canonical bytesからSHA-256で計算する。次のRefは全Fieldを一つのexact backing recordへ解決し、ID-only、display name、`latest`、同version別hashまたは別Project／Registryへfallbackしない。

```text
PackRegistryStateV1
  pack_registry_id: StableId
  pack_registry_revision: positive u64
  expected_previous_registry_state_ref:
    null | exact PackRegistryStateRefV1
  installed_pack_closure_ref: exact InstalledPackClosureRefV1
  registry_state_content_hash: SHA-256

PackRegistryStateRefV1
  pack_registry_id: StableId
  pack_registry_revision: positive u64
  registry_state_content_hash: SHA-256

InstalledPackClosureV1
  installed_pack_closure_id: StableId
  installed_pack_closure_version: positive u64
  installed_pack_entries[0..4096]:
    sorted unique {
      pack_contract_ref: exact PackContractRefV1,
      acquisition_binding_ref: exact PackAcquisitionBindingRefV1,
      resolved_dependency_closure_hash: SHA-256
    }
  installed_pack_closure_content_hash: SHA-256

InstalledPackClosureRefV1
  installed_pack_closure_id: StableId
  installed_pack_closure_version: positive u64
  installed_pack_closure_content_hash: SHA-256

PackOperationReceiptBindingV1
  pack_operation_receipt_binding_id: StableId
  pack_operation_receipt_binding_version: 1
  operation_family_kind:
    pack_acquire | pack_install | pack_apply | pack_update | pack_remove
  operation_ref: exact McdContractRefV1(kind=operation)
  operation_receipt_ref: exact OperationReceiptRefV1
  receipt_payload_type_ref: exact McdContractRefV1(kind=type)
  signed_receipt_purpose_ref: exact McdContractRefV1(kind=policy)
  receipt_subject_contract_refs[1..4096]:
    sorted unique exact McdContractRefV1(kind=type)
  receipt_subject_content_hash: SHA-256
  request_content_hash: SHA-256
  signed_record_content_hash: SHA-256
  owner_typed_receipt_evidence_ref: exact EvidenceRefV1
  pack_operation_receipt_binding_content_hash: SHA-256

PackOperationReceiptBindingRefV1
  pack_operation_receipt_binding_id: StableId
  pack_operation_receipt_binding_version: 1
  pack_operation_receipt_binding_content_hash: SHA-256

PackLifecycleActionBindingV1
  pack_lifecycle_action_binding_id: StableId
  pack_lifecycle_action_binding_version: 1
  required_operation_journey_projection_ref:
    exact RequiredProductOperationJourneyProjectionRefV1
  journey_mode: continuous_e2e | independent_fixture
  journey_scenario_ref: exact QualificationScenarioRefV1
  journey_sequence_index: positive u32
  operation_family_kind:
    pack_acquire | pack_install | pack_apply | pack_update | pack_remove
  operation_ref: exact McdContractRefV1(kind=operation)
  expected_result_branch:
    success | expected_policy_rejection | domain_failure_recovery
  pack_operation_receipt_binding_ref:
    exact PackOperationReceiptBindingRefV1
  product_definition_ref: exact ActiveProductDefinitionRefV1
  toolchain_closure_ref: exact BuildToolchainClosureRefV1
  public_contract_set_ref: exact PublicContractSetRefV1
  candidate_ref: exact PreparedCandidateRefV1
  target_profile_ref: exact TargetProfileRefV1
  journey_locale_scope:
    {kind=not_applicable}
    | {kind=locale_independent}
    | {
        kind=locale_profile,
        locale_profile_ref: exact LocaleProfileRefV1
      }
  journey_reference_dimension_scope:
    {kind=not_applicable}
    | {kind=dimension_independent}
    | {
        kind=reference_dimension,
        reference_dimension: two_d | three_d
      }
  pack_requirement_ref: exact McdContractRefV1(kind=requirement)
  acquisition_binding_ref: null | exact PackAcquisitionBindingRefV1
  source_pack_contract_ref: null | exact PackContractRefV1
  destination_pack_contract_ref: null | exact PackContractRefV1
  before_project_revision_ref: exact ProjectRevisionRefV1
  after_project_revision_ref: exact ProjectRevisionRefV1
  before_pack_registry_state_ref: exact PackRegistryStateRefV1
  after_pack_registry_state_ref: exact PackRegistryStateRefV1
  before_installed_pack_closure_ref: exact InstalledPackClosureRefV1
  after_installed_pack_closure_ref: exact InstalledPackClosureRefV1
  before_dependency_closure_hash: SHA-256
  after_dependency_closure_hash: SHA-256
  compatibility_change_ref: null | exact CompatibilityChangeRefV1
  ordered_migration_step_refs[0..4096]:
    ordered unique exact migration step ref
  update_disposition:
    not_update | rollback_eligible | forward_only
  before_last_valid_build_package_artifact_refs[0..4096]:
    sorted unique exact ArtifactRefV1
  after_last_valid_build_package_artifact_refs[0..4096]:
    sorted unique exact ArtifactRefV1
  failure_evidence_refs[0..4096]:
    sorted unique exact EvidenceRefV1
  pack_lifecycle_action_binding_content_hash: SHA-256

PackLifecycleActionBindingRefV1
  pack_lifecycle_action_binding_id: StableId
  pack_lifecycle_action_binding_version: 1
  pack_lifecycle_action_binding_content_hash: SHA-256

PackLifecycleAcceptanceV1
  pack_lifecycle_acceptance_id: StableId
  pack_lifecycle_acceptance_version: positive u32
  product_definition_ref: exact ActiveProductDefinitionRefV1
  toolchain_closure_ref: exact BuildToolchainClosureRefV1
  public_contract_set_ref: exact PublicContractSetRefV1
  candidate_ref: exact PreparedCandidateRefV1
  target_profile_ref: exact TargetProfileRefV1
  pack_requirement_ref: exact McdContractRefV1(kind=requirement)
  pack_contract_ref: exact PackContractRefV1
  required_operation_journey_projection_ref:
    exact RequiredProductOperationJourneyProjectionRefV1
  journey_mode: continuous_e2e | independent_fixture
  action_binding_refs[5..65535]:
    sorted unique exact PackLifecycleActionBindingRefV1
  security_review_evidence_ref: exact EvidenceRefV1
  privacy_review_evidence_ref: exact EvidenceRefV1
  pack_lifecycle_acceptance_content_hash: SHA-256

PackLifecycleAcceptanceRefV1
  pack_lifecycle_acceptance_id: StableId
  pack_lifecycle_acceptance_version: positive u32
  pack_lifecycle_acceptance_content_hash: SHA-256
```

`PackLifecycleAcceptanceV1`はEngine Release refを含まないpre-release Candidate closureである。`product_definition_ref`は[Product Plan](../00-product/product-plan.md)のexact `ActiveProductDefinitionRefV1`へ全Field byte equalityで解決し、ID／revisionだけ、同名Definition、`latest`、Release labelまたはManifest内の近似値から補完しない。`pack_requirement_ref`は同Definitionの`required_pack_requirement_refs[] ∪ bundled_pack_requirement_refs[]`のexact一memberであり、Product LifecycleのCoverage rowが同じrequirementを同じPack Contractへ束縛しなければならない。Security／Privacy Evidenceは同じPack requirement、permission、source、publisher、Target、Candidateへ限定し、Pack publisherのPrivacy PolicyまたはEngine Product consentで代用しない。

`required_operation_journey_projection_ref`から同じPack requirement／Target／CandidateにapplicableなPack五familyのrequired journey rowを選び、`action_binding_refs[]`の`{family,operation,Target scope,locale scope,Reference dimension scope,scenario,expected branch}` projectionとexact set equalityにする。Pack actionがlocaleまたはdimension independent／not-applicableでもそのclosed branchをAction BindingとReceipt subjectへ保持し、別locale、別dimension、aggregate Claim Scopeまたはnullへcollapseしない。各Action Bindingとowner-typed Evidence／Qualification ReceiptはAI Verification OwnerのVerification Semantic Admissibility Predicate v1を通過し、Scenario applicability、scope vector、subject contract集合、branchをPack subjectからread-backする。各Action Bindingと`PackOperationReceiptBindingV1`はfamily、Operation、Receipt、payload Type、signed purpose Ref、subject contract集合、subject hash、request hash、signed record hash、owner-typed Evidenceをbyte equalityにし、generic Refが解決するowner-specific backing recordをPack Ownerのvalidatorでread-backする。generic `OperationReceiptRefV1`だけ、hash-only Ref、Acquisition Binding、Manifest、download、signature successまたはdisplay action名からPack Operation成功を推測しない。特に`pack_acquire`はexact Operation Receipt Bindingと別の`PackAcquisitionBindingV1`をともに要求し、後者はpublisher／source／artifact／signature／license／permission／provenance inputであってOperation Receiptではない。

Action branchは次のclosed matrixに従う。`acquire`はsource null、destination exact、Acquisition Binding必須でProject／Registry／installed／dependency／last-validのbefore-after byte equalityを要求する。`install`はsource null、destination exact、Acquisition Binding必須、Registryとinstalled closureへdestinationを追加する。`apply`はsource exact、destination exactかつ同一Pack Contract、Project revisionを進め、Registry／installed closureを不変にする。`update`だけがsource／destinationをともに要求し、異なるexact Pack version／hash、exactly one approved `CompatibilityChangeRefV1`、そのChangeが所有するordered migration step集合とのset／order equality、Compatibility Ownerが定めた`rollback_eligible | forward_only`を要求する。`remove`はsource exact、destination nullで、Registry／installed closureからsource exact subsetだけを除去する。全branchのdependency closureは選択Manifest／Recipe closureへexact解決し、action外のField combinationを拒否する。

`journey_mode=continuous_e2e`では同じ`journey_scenario_ref`のsequenceを1からgapなく並べ、各Actionのafter Project／Registry／installed closure／dependency／last-valid集合を次Actionのbeforeとbyte equalityにする。別scenario、別requirement、別Candidate、別Targetまたは別Pack versionのActionをstitchしない。`independent_fixture`では各Actionが固有scenarioとexplicit base stateを持ち、sequence indexは1、別Actionのafterをbaseとして推測せず、複数fixtureを一つの連続journeyへ合成しない。Acceptanceは一つの`journey_mode`だけを許し、同じrecord内で両modeを混ぜない。

`expected_result_branch=success`だけがbranch規則に従うafter stateを公開できる。`expected_policy_rejection | domain_failure_recovery`ではbefore／after Project、Registry、installed closure、dependency closure、Build／Package last-valid集合をすべてbyte equalityにし、non-empty `failure_evidence_refs[]`と同branchを記録するsigned typed Receipt Bindingを要求する。cross-project、cross-registry、cross-requirement、cross-candidate、cross-version、source／destination逆転、Compatibility Change substitutionを拒否する。Product LifecycleはこのRefをsame Release Content Manifestへ集約するだけで、Pack payload、dependency resolutionまたはfailure semanticsを再定義しない。Pack Acceptance、Action Binding、ReceiptまたはEngine Release refをManifest、Acquisition Binding、Recipe、registry state、installed closureまたはdependency closure hashへ戻さない。

## 7. AI discoveryとoperation

AIは自然言語からPack IDを推測してcommitしない。current generic Pack AI Operation setは空で、Capability stateは`not_activated`である。Pack Manifest、Trusted Service allowlist、Provider／MCP Catalogにgeneric Pack AI Operationを登録せず、文書中のfuture vocabularyをOperation IDまたはaliasとして解決しない。

```text
Current generic Pack AI Operation set = {}
Future vocabulary:
  future.packs.action.search
  future.packs.action.read
  future.packs.action.resolve_dependencies
  future.packs.action.explain_composition
  future.packs.action.plan_apply
  future.packs.action.preview_apply
  future.packs.action.validate
  future.packs.action.plan_remove
```

要求は`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`でProject／Pack registry不変として拒否する。future work item `activation.pack.ai_operations.v1`は採用するexact Operation set、MCD全Field、named input／result、Service／Policy／Validator／Diagnostic／Receipt、Risk、authorization intent DAG、private-to-public recovery、Qualificationを同じContract set transactionで完全登録するまでactivateしない。将来の依存、影響、migration、fallbackの説明は`pack_id`、`pack_version`、`content_hash`、Capability ID、Feature Pack edge、Project revisionを含め、Strict Tool Callingに適合しないProviderをCommit Operationへ推測変換しない。

## 8. Diagnosticとfixture

| Failure | Diagnostic ID | 結果 |
|---|---|---|
| Manifest／hash／license／provenance不正 | `MIRAKAN-PACK-MANIFEST_INVALID` | Installを拒否しlast-valid registryを維持 |
| Feature dependency cycle／version conflict | `MIRAKAN-PACK-DEPENDENCY_UNRESOLVED` | closureを拒否しcycleまたは競合rangeを列挙 |
| 選択Recipe dependencyのmissing／incompatible／unqualified | `MIRAKAN-PACK-RECIPE-DEPENDENCY_UNRESOLVED` | Recipe applyを拒否しlast-valid Recipe closureを維持 |
| Genre Pack間dependency | `MIRAKAN-PACK-GENRE_DEPENDENCY_FORBIDDEN` | edgeを拒否しGame Project compositionを案内 |
| Engine contract／Target不適合 | `MIRAKAN-PACK-VERSION_INCOMPATIBLE` | Applyを拒否しshimを生成しない |
| required Capabilityなし | `MIRAKAN-PACK-CAPABILITY_UNAVAILABLE` | Apply／Cookを拒否しTargetとCapabilityを列挙 |
| approved Compatibility Change適用失敗 | `MIRAKAN-PACK-MIGRATION_FAILED` | 新revisionをcommitせずlast-validを維持 |
| live dependencyあり | `MIRAKAN-PACK-REMOVAL_BLOCKED` | Removalを拒否しdependency closureを列挙 |

Pack contract fixtureは最低限、manifest／`CompositionRecipeV1` round-trip、全Field presence、initial V1のempty migration set、feature DAG／cycle／diamond、Manifest共通closureと選択Recipe conditional closureのhash、未選択Recipe dependency非install、Recipe missing／version conflict／Target不適合時のlast-valid維持、fallback cycle、Genre間dependency拒否、Genre PackなしのFeature-only Project、source／publisher／signature／license／permission、install／update／removal rollback、approved Compatibility Change適用失敗時のlast-valid維持、Pack lifecycle acceptanceのsame Candidate／Project revision／Target、Shooter未install時のCore／Editor／AI／Build／Package成功を含む。Reference Game／FixtureがProduction artifactへ逆依存しないこともdependency graphで検査する。

## 9. 完了条件

- initial V1の全Pack Manifestで`migration_step_refs=[]`となり、未materialize predecessorを参照しない。
- source、publisher、signature、license、permission、Security、Privacy、dependency closureがAcquisitionとLifecycle Acceptanceへ一方向に閉じる。
- install、apply、update、removal、last-valid recoveryが同じCandidate、Project revision、Target、Pack identityへ束縛される。
- Compatibility classとrollback eligibilityはCompatibility Ownerだけが決定し、Pack Contractはapproved Changeのtransaction適用だけを所有する。
- Product LifecycleがPack固有Schemaを複写せず、exact `PackLifecycleAcceptanceRefV1`だけをReleaseへ集約できる。
- Schema、Registry、Operation、Fixture、Receipt、Acceptanceが未materialize／未ActivationであることをArchitecture InventoryとClosure Reviewが保持する。
