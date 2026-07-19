# Miraikanai Engine Asset Pipeline／Content Package規約

- 文書版: 1.0
- 作成日: 2026-07-19
- 対象: Asset source、Import、Cook、Cache、Catalog、VFS、Streaming、Package、Patch／DLC、AI生成Asset
- 状態: プロジェクト公式の規範設計レビュー版
- Authoring規約: [Miraikanai Engine Authoring Model／Project State規約](./2026-07-19-authoring-model-project-state-design.md)
- 基盤規約: [Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)
- Runtime規約: [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
- 機能範囲: [Miraikanai Engine 2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)
- AI検証規約: [Miraikanai Engine AI検証・評価・来歴規約](./2026-07-19-ai-verification-evaluation-provenance-design.md)
- Windows規約: [Miraikanai Engine Windows Platform／Distribution規約](./2026-07-19-windows-platform-distribution-design.md)
- Mobile規約: [Miraikanai Engine モバイルPlatformアーキテクチャ規約](./2026-07-19-mobile-platform-architecture-design.md)

## 1. 結論

Miraikanai EngineはAssetを「Editorが読めるfile」ではなく、次の四層に分離する。

1. **Source Asset**: 人間／DCC／AIが作る原本
2. **Import Document**: Stable ID、意味、設定、権利、来歴
3. **Derived Artifact**: Target別に決定論的生成したimmutable runtime product
4. **Content Package／Catalog**: 配布、mount、依存、streamingの単位

Runtime、Renderer、Physics、Navigation、Animation、AudioはSource fileを直接読まない。AIもCooked binary、package index、GPU／Physics native objectを直接生成しない。全Sourceは隔離Importer、schema／semantic／budget、Target cook、dependency closure、atomic promotionを通る。

## 2. 決定権と対象外

| 主題 | 正本 |
|---|---|
| Asset identity、metadata、import、cook、cache、catalog、VFS、package、streaming | 本書 |
| Source Document Commit、StableId、ChangeSet | Authoring規約 |
| Asset generation／lease／atomic promotion、memory charge | Runtime規約 |
| Format／Style／Material／Shader／Nav／Physicsの機能範囲 | 2D／3D機能計画と各Subsystem規約 |
| PAD、Apple Background Assets、Store package | Mobile規約 |
| Windows installer／distribution／crash | Windows Platform規約 |

C1／C2ではOpenUSDを正本にせず、FBX importer、Runtime arbitrary import、remote untrusted mod、encrypted DRM package、peer-to-peer content、self-hosted binary updaterを実装しない。これらは個別Capabilityとsecurity designなしに追加しない。

## 3. Asset identityとmetadata

### 3.1 Stable identity

- `AssetId`はRFC 9562 UUIDv7であり、rename／move／再importで変えない。
- `AssetRevision`はSource bytesまたは意味あるImport設定の変更ごとに+1する。
- Runtimeは`AssetVersionHandle { index32, generation32 }`と`ArtifactKey`を使用する。
- path、display name、array index、content hashだけを永続identityにしない。
- 同じSourceから複数Artifactを生成する場合、`AssetId`＋stable `ArtifactRoleId`で識別する。

### 3.2 `AssetSourceDescriptor`

| Field | 型／規則 |
|---|---|
| `asset_id` | UUIDv7 |
| `asset_type` | closed Asset Type ID |
| `asset_revision` | `uint64` |
| `source_relative_path` | Project `assets/source`内、NFC UTF-8 |
| `source_sha256` | file bytes |
| `source_media_type` | allowlist enum |
| `import_profile_id` | typed Profile StableId |
| `import_settings` | Asset Type別MCD object |
| `declared_dependencies` | AssetId＋role |
| `license_record_id` | 必須 |
| `provenance_record_id` | 必須 |
| `safety_receipt_id` | 外部／AI生成時必須 |
| `editor_tags` | 最大64 |

Pathはseparatorを`/`へ正規化し、absolute path、drive、UNC、`..`、empty segment、NUL、reserved device name、case-fold衝突を拒否する。Windows上でもlogical path比較はUnicode NFC＋ASCII case foldによるProject canonical keyを使い、同一keyの二fileを許可しない。

### 3.3 Asset Type

| Type | C1 Source | 主なDerived Artifact |
|---|---|---|
| Texture／Sprite | PNG、JPEG、OpenEXR、KTX2、DDS | Target texture、mip、sprite table、thumbnail |
| Mesh／Scene | glTF 2.0／GLB | mesh stream、meshlet reserved、skin binding、collision／nav source |
| Skeleton／Animation | glTF 2.0 | ozz runtime skeleton／clip、event／root track |
| Audio | WAV、FLAC | PCM16 one-shot、Opus stream chunk、waveform |
| Font | OTF／TTF | validated font blob、glyph coverage index |
| Shader | Engine／承認済みProject HLSL | DXIL、SPIR-V、metallib、ShaderInterface |
| Material／Style | MCD JSON | parameter block、pipeline key、Style Manifest |
| UI／Gameplay／Scene | MCD JSON | runtime package section |
| Physics／Nav | Engine Source Document＋geometry | backend-independent manifest＋Target cooked payload |

SVG、video、FBX、PSD、Blend等はC1 importer allowlistに含めない。利用者はDCCでC1 interchangeへexportする。Format追加はImporter threat model、fixture、license、fuzz、Target cook、memory上限を持つADRを必須とする。

## 4. Import Job

### 4.1 Job identity

```text
AssetImportJob
  job_id
  asset_id
  asset_revision
  source_hash
  import_profile_hash
  importer_id
  importer_version_hash
  target_profile_id
  dependency_artifact_keys[]
  toolchain_lock_hash
  limits
```

Job keyは上記のcanonical tupleからSHA-256で作る。同一keyの成功Artifactを再利用できるが、Source path timestampだけではcache hitにしない。

### 4.2 隔離実行

ImporterはEditor／GameHostと別のWorker Processで実行する。

- network deny
- 読取はbrokerが開いたSource／dependency handleだけ
- 書込はJob固有temporary directoryだけ
- Windows Job Objectまたは各Platform sandboxでprocess treeを制限
- CPU time、wall time、commit memory、output bytes、file countをhard cap
- child process、shell、environment expansion、plugin discoveryを禁止
- untrusted file名をcommand lineへ連結しない
- crash、timeout、OOM、sanitizer failureをJob failureとして隔離

Importer outputは直接Cacheへ移動しない。Output manifest、全file hash、schema、semantic、size、dependency、determinism testが合格した後だけArtifact storeへ原子的にpublishする。

### 4.3 Determinism

同じJob keyをclean workerで二回実行したoutputはbyte-for-byte一致しなければならない。Timestamp、machine path、user、random ID、nondeterministic map順序をArtifactへ含めない。GPU driver依存のoffline buildをC1正規Cookに使わない。

Thumbnail、debug dump、human-readable reportは正式Artifactと分け、Shipping closureへ自動追加しない。

## 5. Asset別validation

| Type | 必須validation |
|---|---|
| Texture | dimension、mip、channel、alpha mode、color space、normal map、block alignment、GPU size |
| Sprite | rect overlap／bounds、pivot、PPU、pixel sampling、atlas padding |
| glTF | accessor bounds、buffer length、node cycle、finite transform、extension allowlist、material support |
| Mesh | index range、degenerate比、vertex／triangle count、bounds、LOD順、skin influence |
| Skeleton | joint count、unique stable joint path、parent cycle、bind pose finite／invertible |
| Animation | duration、strict key order、finite transform、skeleton match、event type、root motion policy |
| Audio | sample rate、channel layout、duration、peak／NaN、loop point、decoded／stream memory |
| Font | table bounds、glyph coverage、embedding permission metadata、corrupt outline |
| Shader | include allowlist、entry/interface、Target compile、resource／instruction budget |
| Material／Style | Interface hash、parameter range、variant、Target Capability、Style lock |
| Physics／Nav | geometry range、cook profile、backend version、query fixture、hard memory cap |

Validatorはunsupported featureを削除して成功させない。明示したfallbackがあり、Preview差分と人間承認がある場合だけ別ChangeSetとしてImport設定を変更する。

## 6. Derived Artifact

### 6.1 Artifact manifest

```text
DerivedArtifactManifest
  artifact_key: sha256
  asset_id
  asset_revision
  artifact_role_id
  target_profile_id
  schema_version
  payload_hash
  payload_size
  alignment
  dependency_keys[]
  importer_id
  importer_version_hash
  toolchain_lock_hash
  capability_requirements[]
  runtime_budget
```

`ArtifactKey`はRuntime規約のcanonical artifact encodingを使う。Payload headerはAssetId／revision／role／Target／schema／sizeを持ち、unknown major、truncated payload、trailing bytes、hash mismatchを拒否する。

### 6.2 Dependency graph

- Hard dependencyは親Artifactと同じgenerationでReadyでなければpromotionできない。
- Optional dependencyは明示fallback AssetIdと意味を持つ。
- Build-only dependencyをRuntime Catalogへ含めない。
- Hard dependency cycleをrejectする。
- Material→Texture、Mesh→Material等のclosureをPackage build前に完全解決する。
- Source、Import設定、Importer、Tool、dependency Artifactのいずれかが変われば影響nodeだけを再Cookする。

Artifact storeはcontent-addressed、immutableである。成功Artifactを上書きせず、新keyで追加する。Garbage collectionはProject revision、package manifest、last-valid generation、active lease、Recovery snapshotからのreachabilityを計算した後だけ行う。

## 7. CatalogとVFS

### 7.1 Runtime Catalog

`MiraAssetCatalogV1`は次を持つ。

| Field | 規則 |
|---|---|
| `catalog_id`／`catalog_version` | UUIDv7／`uint64` |
| `target_profile_hash` | Packageと一致 |
| `package_set_hash` | mount closure |
| `entries` | AssetId＋role順 |
| `dependencies` | ArtifactKey順 |
| `content_groups` | base、optional、level、DLC等 |
| `capability_requirements` | Target起動前検査 |
| `root_hash` | canonical catalog SHA-256 |

Runtime codeは`asset://<uuid>/<role>`の論理参照またはtyped `AssetHandle`を使う。OS pathをGameplayDefinition、Save、Network payloadへ保存しない。

### 7.2 VFS mount

| Layer | Priority | C1／C2規則 |
|---|---:|---|
| Base | 0 | 必須、Platform署名済みApplication package内 |
| Patch | 100～199 | C2。base Package IDと許可revision rangeを必須化 |
| DLC | 200～299 | C2。独立entitlement、dependency、unmount policy |
| Developer Override | 1000 | Developmentだけ。Shipping buildで機能自体を除外 |

同じAssetId＋roleを高priority Catalogが置換できるのは、schema compatibility、Capability、dependency closure、package signature／hashが合格する場合だけである。異なるAssetをpath一致でoverrideしない。Mount／unmountはPackage set全体をStagingし、active Asset leaseがあるDLCを強制unmountしない。

VFSはContent Package読取専用であり、Save、setting、screenshot、crash dumpはPlatform User Data Portへ分離する。

## 8. `.mirapack` C1 format

独自Content Containerを`Mira Content Package V1`として定義する。

```text
Header
  magic "MIRAPAK1"
  endian marker
  format major/minor
  package_id
  package_revision
  target_profile_hash
  block_size = 65536
  index_offset/index_size
  catalog_hash
  package_root_hash

Payload Blocks[0..N)
Index
  artifact entries
  block hash table
  dependency/catalog location
```

- C1 blockはexactly 64 KiB、最終blockだけ短くできる。
- block codecは`stored`だけとする。Texture、Audio等はAsset codecで圧縮済みとし、未承認のcontainer圧縮dependencyを追加しない。
- Artifactは64 KiB boundaryへ配置し、Indexがblock range、logical size、payload hashを持つ。
- Package root hashはHeaderの可変hash fieldを0にしたHeader、Block hash table、Index canonical bytesからSHA-256で作る。
- Indexはbounds、overlap、duplicate、integer overflow、path、dependencyをmount前に検証する。
- RuntimeはPackage全体をmemory mapすることを前提にせず、bounded async range readを使う。

C1の完全性はApplication package署名と`.mirapack` root hashで検証する。Self-hosted remote Patch／DLCの署名、rollback protection、key rotationはC2 Security ADRを承認するまでShipping Capabilityとして公開しない。

## 9. Package build、Patch、DLC

### 9.1 Package build

1. Commit済みProject revisionとTarget Profileを固定する。
2. root Scene、always-loaded UI／font／shader、Content GroupからAsset rootを列挙する。
3. Hard dependency closureを解決し、missing／cycle／Target不一致を拒否する。
4. Artifact hashとLicense／Provenance／Safety Receiptを照合する。
5. groupごとにcanonical AssetId／role順でblockを配置する。
6. Catalog、Package Index、SBOM、Third-party Notice、Build Receiptを生成する。
7. clean Packageを再生成してroot hash一致を検証する。
8. Platform packageへ封入し、Platform署名／Store validationへ渡す。

Developmentのloose Artifact layoutとShipping `.mirapack`は同じCatalog conformanceを通す。

### 9.2 Patch

C2 PatchはArtifact単位で差分を作り、同一hashの64 KiB blockをbaseから再利用できる。Patch apply中にbaseを上書きせず、新Packageを完全検証してからmount setを原子的に更新する。途中失敗、disk full、signature不合格では旧Package setを維持する。

Binary executable、native library、shader binaryをContent Patchとして配布しない。これらはPlatform application updateでのみ更新する。

### 9.3 DLC

DLCはcontent group、entitlement key、base version range、dependency、save behaviorを持つ。DLC不在時に必要なSave fieldはtyped `MissingContent`として扱い、別Assetへ黙って置換しない。DLCの有無でCore Saveを破損させないfixtureを必須とする。

## 10. Streaming

```text
Unloaded -> Requested -> Reading -> Validating -> Decoding
-> DomainPreparing -> Ready -> Active -> Evictable -> Unloaded
```

- Requestはpriority、deadline、budget class、owner generationを持つ。
- I/O completion threadはread完了だけを通知し、decode／transcodeをworkerへ渡す。
- CPU payload、GPU upload、Audio decode、Physics／Nav buildを含むdependency closureがReadyになるまでActiveにしない。
- Simulationは`T00`、Renderingは`R10`、Audioはcontrol block境界でversionをactivateする。
- deadline超過、owner generation不一致、cancel済み結果をpublishしない。
- Evictionはnoncritical、遠距離／低priority、再取得可能なresourceから行い、active leaseを破棄しない。
- last valid generationと一つのretiring generationを超えて保持しない。

Windows Asset streaming cache 768 MiBとDomain別chargeは基盤／Runtime規約を適用する。Mobileは同じ絶対値を使わず、Mobile Profileを正本とする。

## 11. Hot Reload

Source変更は新Asset revisionとImport Jobを作る。依存closure全体をStagingし、次を満たした場合だけpromotionする。

- 全Artifactが同じTarget／Toolchain／Contract lockでReady
- Interface／schema compatibility合格
- Shader／Material、Skeleton／Clip、Physics／Nav等のDomain invariant合格
- memory／GPU／queue予約可能
- Runtime phase boundary到達

一つでも不合格なら旧generation全体を維持する。Textureだけ新、Materialは旧などの部分Version混在を禁止する。互換でない変更は`restart_play`を返す。

## 12. AI生成Assetと権利

### 12.1 C1／C2の確定範囲

- C1は特定の画像／音声／3D生成Providerを製品にbundledしない。
- C1は外部で生成済みのAssetを、通常Sourceと同じImport、権利、来歴、安全、品質Gateで受け入れられる。
- C2でProvider Adapterを追加する場合も、Provider outputを直接Project／Cacheへ入れず`GeneratedAssetStaging`へ置く。
- Runtime AIによるAsset生成、download、shader／native code生成はShippingで禁止する。

これは機能欠落ではなく、Providerの利用規約、地域、費用、Model version、権利条件が変動するため、Production Providerを固定せず安全な受入口を先に完成させる順序である。

### 12.2 Provenance record

AIまたは外部Assetは次を必須とする。

| Field | 規則 |
|---|---|
| `origin_kind` | human、commissioned、licensed_library、generative_ai、procedural |
| `creator_or_provider` | organization／user reference |
| `tool_and_model` | exact version／manifest hash。該当時 |
| `generation_request_hash` | prompt／settingsのprivate record hash |
| `input_asset_refs` | 利用したProject Assetと権利宣言 |
| `terms_snapshot_ref` | 取得日、URL、hash |
| `license_expression` | SPDXまたはProject policy enum |
| `commercial_use_review` | reviewer、decision、scope |
| `content_safety_receipt` | policy version、result |
| `c2pa_manifest_hash` | 存在時。存在しないことも明記 |
| `modification_chain` | parent provenance参照 |

C2PAは検証可能な来歴情報として保持するが、Content Credentialがあるだけで著作権、商用利用、品質を自動承認しない。License不明、入力権利不明、Provider terms snapshotなし、禁止contentはProduction Packageをblockする。

### 12.3 品質workflow

AIはGameSpec、Visual Style lock、technical budgetから`AssetRequirement`を作り、候補生成、技術validation、style consistency、Scene Preview、人間Review、差替えを行う。画像、Audio、3D、Animationそれぞれで次を必須とする。

- dimension／duration／topology／rig／loop等のacceptance criteria
- 複数候補と採用理由
- 既存Assetとのstyle／semantic role比較
- Target cookとmemory／runtime cost
- artifact、provenance、license、safetyの連結
- PlaceholderとProduction Readyの明示区別

AIが「商用品質」と自己宣言してGateを省略できない。

## 13. Failure policy

| Failure | 結果 |
|---|---|
| Import crash／timeout／OOM | Job失敗、temporary隔離、旧Artifact維持 |
| Determinism不一致 | Importer versionをProduction禁止 |
| Unsupported feature | Asset rejectまたは承認済み別ChangeSet |
| Dependency missing／cycle | closure全体reject |
| Corrupt Cache | hashでrejectし再Cook。Sourceへ逆流しない |
| Package index／hash不正 | mount拒否 |
| Streaming deadline | stale result破棄、明示placeholder policyまたはScene load失敗 |
| Promotion不合格 | old generation全体維持 |
| Disk full during Patch | 新Package破棄、旧mount set維持 |
| License／Provenance不足 | Production Package block |
| AI Provider error | Staging Job失敗、Project revision不変 |

Placeholderを許可するAsset roleはProfileで明示し、Collision、Nav、required UI font、Gameplay Definition等のauthoritative／required Assetへ汎用placeholderを使わない。

## 14. TestとRelease Gate

- 全Importerのmalformed／oversized／zip-bomb相当／path traversal／fuzz corpus
- 同一Jobのclean二回CookでArtifact hash一致
- Source／setting／Importer／dependency変更時の正確なincremental invalidation
- Catalog dependency cycle、missing、duplicate、Target mismatch
- `.mirapack` header／index／block bounds／hash／overflow fuzz
- 64 KiB block patch再利用と途中kill／disk full rollback
- Asset generation closureのatomic promotionとlease中retire
- Cache削除後のfull rebuildでPackage root hash一致
- Windows、Android、Apple Target別Texture／Shader／Audio／Mesh Cook
- Streaming queue、cancel、deadline、stale result、memory pressure、surface／audio interruption
- License、SBOM、Notice、Provenance、Safety ReceiptのPackage closure
- AI Assetが通常Importを迂回できず、Production Readyへ自動昇格しないconformance

C1完了条件は、2D／3D縦切りの全AssetをSourceからclean Cookし、Development loose layoutとShipping `.mirapack`で同じCatalogをloadし、hot reload、corrupt input、memory pressure、Package再生成を合格することである。

## 15. 一次資料

- [O3DE Asset Pipeline](https://docs.o3de.org/docs/user-guide/assets/pipeline/)
- [O3DE Asset Cache](https://docs.o3de.org/docs/user-guide/assets/pipeline/asset-cache/)
- [O3DE Product Assets and deterministic generation](https://docs.o3de.org/docs/user-guide/assets/pipeline/product-assets/)
- [glTF 2.0 Specification](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html)
- [KTX 2.0 Specification](https://registry.khronos.org/KTX/specs/2.0/ktxspec.v2.html)
- [C2PA Specifications 2.4](https://spec.c2pa.org/specifications/)

外部EngineのAsset database／bundle形式は採用しない。Sourceとruntime productの分離、dependency tracking、決定論的生成、来歴という原則を根拠に、Miraikanai固有のArtifact、Catalog、Package、promotionを定義する。
