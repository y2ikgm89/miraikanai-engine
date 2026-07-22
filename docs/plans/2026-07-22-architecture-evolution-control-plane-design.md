# Miraikanai Engine Architecture Evolution Control Plane Design

- 作成日: 2026-07-22
- 状態: 詳細設計レビュー待ち
- 採用方針: Owner正本化（ユーザー承認済み）
- 対象Baseline: branch `codex/runtime-ecs-contract`、43 active architecture specs、Architecture Index、2 Decisions
- 後続Subsystem計画: [AI-Readable Direct3D 12 Backend Design](2026-07-22-ai-readable-d3d12-backend-design.md)
- 対象外: Engine実装code、Build script、MCD compiler実装、既存仕様の承認状態変更

## 1. 結論

Miraikanai Engineの各Subsystem仕様は、single writer、typed command／event／snapshot、Source／Derived分離、Target別Qualification、negative fixture、last-valid rollbackを既に強く定義している。一方で、機能追加や契約変更が複数Subsystemへ波及するときに必要な次の横断Ownerが欠けている。

1. Architecture文書のlifecycle、typed relation、Owner、変更影響を決めるOwner。
2. Public Contract、Project、Save、Domain Pack、ABI、Packageの互換性と進化を決めるOwner。
3. System別Save projectionを一つの永続transactionへ集約するOwner。
4. Cook済みRuntimeをload可能なRuntime Packageへ束ねるOwner。
5. Runtime／Content／Shader／Platform artifactをApplication Package、署名、提出へ束ねるOwner。

本設計では次の5正本を追加し、既存43正本をそれらへ接続する。

- `docs/architecture/01-governance/architecture-governance.md`
- `docs/architecture/02-foundation/compatibility-evolution.md`
- `docs/architecture/04-runtime/persistence-save.md`
- `docs/architecture/04-runtime/runtime-package.md`
- `docs/architecture/07-platform/application-package-release.md`

保留中のRuntime ECS Decisionは、この5正本と既存Ownerの更新が完了するまでactive specへ昇格しない。新ECS正本は上記横断契約を消費し、Save、Package、Compatibility、Document Governanceを再定義しない。

## 2. 完了条件

計画書更新は、文章を追加しただけでは完了としない。次をすべて満たした状態を完了とする。

1. 全active specが一意な文書ID、文書状態、正本範囲、非正本範囲、`requires`、`integrates_with`を持つ。
2. `requires` graphがcycleのないDAGになり、実装順序と変更影響を決定できる。
3. 全cross-document canonical contractに一意なOwnerがあり、consumerはOwnerへの参照だけを持つ。
4. 永続またはwire Schema typeはすべて`PascalCaseV<major>`で、suffixなしlatest aliasが0件である。
5. Capability ID、Work Package ID、Contract ID、Artifact IDがmaturity、document status、表示名をidentityへ埋め込まない。
6. 文書状態、Capability product tier、Capability activation、Work Package scheduling、Contract registration、Target readiness、Evidence freshness、Dependency adoptionが別dimension／state machineとして定義される。
7. `Authoring -> Cook -> Runtime Package -> Application Package -> Sign -> staging upload/read-back -> Game Candidate -> Human Approval／Activation -> Store submit/publish/read-back`が型付きmanifestとReceiptで閉じる。
8. `Gameplay/System Save Contract -> Runtime checkpoint -> Save aggregate -> Platform atomic storage -> staged load/publish`が型付きcontractで閉じる。
9. Domain PackのEngine compatibility range、Pack dependency、resolved lock、migration、exact Qualificationがmanifestから検証できる。
10. Pre-1.0 clean breakとreleased user data保護、Post-1.0 SemVer、migration、support policyの適用対象がartifact classごとに決まる。
11. Product Phase 0～9、Work Package、Capability、Owner、fixture、Target、fallbackの参照にorphanがない。
12. Index件数、Local link／anchor、文書ID、Owner、DAG、型名、ID文法、Phase参照を機械検査できる。
13. Runtime ECS Decisionが存在しないSave Owner、Package Owner、型versionを前提にしない。
14. 外部Tool／SDK／Libraryのrelease、tag object、peeled commit、artifact hashを区別する。
15. Manifest、Package、Candidate、Receiptのidentity graphに自己参照またはhash cycleがなく、全digestの入力byte列を一意に再構成できる。
16. 人間とAIが同じexact SourceからOwner、依存、phase／lifetimeを説明でき、上限超過、stale revision、根拠欠落を正常結果として扱わない。

## 3. 非目標

- 全Subsystem algorithmを本変更で再設計しない。
- 既存の正しいRuntime phase、Budget、Physics／Navigation／Rendering algorithmを横断Ownerへ移さない。
- Public Marketplace、任意binary plugin、Runtime AI生成、Online／Multiplayerを暗黙にActivationしない。
- 後方互換shim、旧Schema alias、二重serializer、二重Package入口を追加しない。
- 文書の`review`を自動的に`approved`へ変更しない。
- 実装がない状態でPerformance、Security、Target互換性を合格済みと表現しない。
- MCD compilerやEngine実装の成功を文書整合性だけから推定しない。

### 3.1 D3D12 Companionとの変更境界

[AI-Readable Direct3D 12 Backend Design](2026-07-22-ai-readable-d3d12-backend-design.md)は、本設計が追加する5横断Ownerの6番目ではない。`docs/architecture/06-rendering/d3d12-backend.md`を一意OwnerとするRendering Subsystemの後続設計である。本横断ChangeSetの範囲をD3D12 Backend詳細へ拡張しない。

依存関係と実行順序を次に固定する。

1. 本設計でArchitecture Governanceのmetadata／relation／Owner grammarを確定する。
2. 本横断ChangeSetを完了し、共有するRendering／Windows／UI／Toolchain文書を一つの整合したbaselineにする。
3. D3D12 Companionを別ChangeSetで反映し、Render Graphとnative BackendのOwner分離、AI Profile分離、MCD projection、Qualification Gateを追加する。
4. D3D12詳細設計の承認後にのみ実装計画を作り、Product Phase 2 Windows vertical sliceより前にTarget Qualificationを閉じる。

共有文書を両ChangeSetで同時編集しない。D3D12 Companionは本設計の完了hashをinput baselineとし、完了前の本横断ChangeSetへD3D12固有型または実装を混入させない。

## 4. 設計原則

### 4.1 一つの意味に一つのOwner

Contractの構造、Field意味、state transition、failure policyを二つの文書で共同所有しない。横断文書はDomain意味を再掲せず、exact contract referenceと必須invariantだけを所有する。

### 4.2 Authority、data、executionを分離する

- Governanceは承認、Policy、Evidence freshnessを所有する。
- DomainはSource意味とDomain固有Stateを所有する。
- Runtimeはstaging、publication、lease、transactionを所有する。
- PlatformはOS storage、package format、signing adapterを所有する。
- Build Gatewayは型付きtaskの順序付けを行うが、各artifactの意味を所有しない。

### 4.3 Strict validationとcompatible evolutionを両立する

未知Fieldを黙って無視する汎用tolerant readerは採用しない。互換性は、明示されたSchema revision、version negotiation、migration、extension pointだけで成立させる。

同一major内のadditive changeでは、新readerが旧writerのartifactを読めることを必須とする。旧readerが新writerを読むforward compatibilityは約束しない。Writerはconsumerが合意したexact revisionを出力し、`latest`を出力しない。

### 4.4 Rebuild可能物とUser dataを分離する

Derived cache、BMI、device cache、Runtime Package、Application Packageはexact Sourceから再生成できる。Project Source、Save、User setting、承認済みGame revisionは再生成不能なUser dataとしてmigrationとrollbackを要求する。

### 4.5 ActivationはEvidenceからだけ決める

文書の存在、Schemaの存在、candidate binaryの存在をCapabilityの`qualified`または`production`とみなさない。Target別Receiptとfreshnessを必要とする。

### 4.6 Identityは結果から前提へ逆参照しない

Build入力を表すsubject、生成artifact、検証Receipt、最終Release recordを別identityにする。前段artifactへ後段Candidate hashを埋め込まず、manifest自身のhashをmanifest内へ持たせない。これにより自己参照hash、再署名によるidentity変化、Target間の循環参照を禁止する。

## 5. 目標Architecture

```text
Architecture Governance
  ├─ document lifecycle / typed relation / owner audit
  └─ change impact closure

Compatibility & Evolution
  ├─ public contract / schema / ID / SemVer
  ├─ migration / deprecation / support policy
  └─ artifact-class compatibility

Authoring Source
  -> Cook / Contract compiler / Content build
  -> Runtime Package
       -> Runtime loader
       -> Persistence compatibility binding
  -> Application Package Assembly
       -> Platform validator
       -> Signing service
       -> Store staging upload and remote read-back
       -> Game Candidate / Human Approval / Activation
       -> Store submit, publish, and remote read-back

Gameplay/System `SaveReplayContractV1`
  -> Persistence coordinator
       -> World/ECS/Domain save projectors
       -> SaveCheckpointV1
       -> PlatformStorageTransactionV1
       -> SaveCommitReceiptV1
```

## 6. 新しい正本仕様

| Path | Stable document ID |
|---|---|
| `01-governance/architecture-governance.md` | `mirakan.arch.architecture-governance` |
| `02-foundation/compatibility-evolution.md` | `mirakan.arch.compatibility-evolution` |
| `04-runtime/persistence-save.md` | `mirakan.arch.runtime-persistence-save` |
| `04-runtime/runtime-package.md` | `mirakan.arch.runtime-package` |
| `07-platform/application-package-release.md` | `mirakan.arch.platform-application-package-release` |

5文書は初回作成時に`state=review`、`approval_ref=null`とする。本設計の承認をEngine仕様の自動承認へ流用しない。

Product RegistryのWork Packageがこの5文書のいずれかを`owner_document_id`として参照する場合、Owner文書が`state=approved`で、non-null `approval_ref`のDecisionが現在bytesのhash／Git blob IDをread-backできるまで`declared`から遷移させない。開始要求は`diagnostic.architecture.owner-unapproved`（code `MIRAKAN-ARCHITECTURE-OWNER-UNAPPROVED`）で拒否し、未承認Ownerの責務をconsumer文書、実装Plan、暫定schemaへ複写して迂回しない。

### 6.1 Architecture Governance

`01-governance/architecture-governance.md`は次だけを所有する。

- Architecture document metadataとlifecycle。
- `requires`、`integrates_with`、`supersedes`の意味。
- Canonical contract Owner宣言とconsumer参照規則。
- Change impact closureと更新単位。
- Index生成、lint、review、approval、supersede Gate。
- Decisionとactive specの関係。
- exact metadata／registry／Source revisionから生成するbounded architecture-explain projection。

AI Risk、Release approval、Evidence内容は既存Governance文書が引き続き所有する。

### 6.2 Compatibility／Evolution

`02-foundation/compatibility-evolution.md`は次だけを所有する。

- Engine public surfaceの分類。
- Pre-1.0とPost-1.0のversion policy。
- Schema major／minor／patch、Field identity、enum、extension point規則。
- Project、Save、Replay、Domain Pack、Native ABI、Runtime／Application Packageの互換方向。
- Migration graph、deprecation、support window、revocation。
- Target非依存の`GameCompatibilitySubjectV1`。Project revision、Engine baseline、Contract／Content／Save／Domain Pack closureを一つの互換subjectへ固定する。
- `EngineContractVersionV1`、`EngineCompatibilityRangeV1`、`DomainPackCompatibilityRangeV1`、`CompatibilityDecisionV1`、`MigrationCoverageV1`、`SupportPolicyV1`。
- Change classificationと必須verification closure。

個々のDomain migration algorithmはDomain Ownerが所有し、本書は登録、順序、coverage、failure atomicityを所有する。

### 6.3 Persistence／Save

`04-runtime/persistence-save.md`は次だけを所有する。

- 全System／World／ECSのSave projection集約。
- Save slot、generation、checkpoint、root manifest。
- Capture boundary、canonical order、hash、size bound。
- Migration plan、load staging、atomic publication、failure recovery。
- Replay開始checkpointとの共有境界。
- Platform storage adapterへ渡すtransaction contract。
- Runtime Package／`GameCompatibilitySubjectV1`とのcompatibility binding。

各Systemのsaved／derived Field意味は`SaveReplayContractV1`と各Domain Owner、Runtime Entity／Component再構築はECS Owner、file root／atomic replace／journal／cloud transportはPlatform Ownerが所有する。

### 6.4 Runtime Package

`04-runtime/runtime-package.md`は次だけを所有する。

- `RuntimePackageManifestV1`。
- `RuntimePackageArtifactV1`。Manifest hash、payload root hash、integrity／trust profileをManifest外で束ねる。
- Runtime loaderの検証順、exact dependency closure、load／rollback。
- Engine／Game binary、Contract set、System implementation set、World root、Content Package、Shader artifact、Save migration setのbinding。
- Development loose layoutとShipping packageが通る共通conformance。
- Runtime Packageのrebuild／invalidation規則。

Content Package formatはAsset Lifecycle、Platform executable formatは各Platform、Build task orchestrationはBuild Gatewayが所有する。

### 6.5 Application Package／Release

`07-platform/application-package-release.md`は次だけを所有する。

- `ApplicationPackageAssemblyManifestV1`。
- `UnsignedApplicationPackageV1`の共通envelope。
- `PackageValidationReceiptV1`、`TargetPackagePreparationTransactionV1`、`StoreStagingUploadReceiptV1`、`StoreStagingReadBackReceiptV1`、`TargetPackagePreparationRecordV1`、`ReleaseTransactionV1`、`StorePublicationReceiptV1`。
- Target packageのBuild、Validate、Authorize、Sign、staging upload／read-backと、Candidate承認後のStore submit、publish、public read-backを分けた一方向state machine。
- Windows／Android／Apple固有packageへのmapping。
- Signing／Upload serviceのinput最小化とidentity separation。
- Compatibility Ownerの`GameCompatibilitySubjectV1`参照。
- `ReleaseSigningReceiptV1`、Store staging Receipt、`TargetPackagePreparationRecordV1`へのhash binding。

署名authorizationと最終`GameCandidateManifestV1`はAI Security／Approval、Evidence envelopeはAI Verification／Provenance、AAB／Apple archive／Windows layoutのformatは各Platform Ownerが所有する。Application Package／ReleaseはCandidateを再定義せず、Target別の完了Recordを供給する。

## 7. Architecture文書Governance

### 7.1 Header format

全active specはH1直後の最初のblockに、次のexact JSON metadataを持つ。Delimiter、key、型を固定し、YAML、Markdown list、別名key、comment、trailing comma、duplicate keyを許可しない。Relationはpathではなくstable document IDを格納し、Index generatorがlinkへ解決する。

```text
<!-- architecture-metadata:v1
{
  "schema": "mirakan.architecture-document-metadata/v1",
  "document_id": "mirakan.arch.compatibility-evolution",
  "state": "review",
  "owner_scope": ["互換性と進化の規則"],
  "non_owner_scope": ["個別Domainのmigration algorithm"],
  "requires": ["mirakan.arch.architecture-governance"],
  "integrates_with": [
    {
      "document_id": "mirakan.arch.executable-contracts",
      "contract_ids": ["contract.foundation.artifact-ref"]
    }
  ],
  "supersedes": [],
  "approval_ref": null,
  "external_evidence_verified_at": "2026-07-22"
}
-->
```

制約は次である。

- `document_id`は`^[a-z][a-z0-9]*(?:\.[a-z][a-z0-9_-]*)+$`に一致し、renameまたは移動でも変えない。
- Arrayは重複なし、`requires`／`supersedes`はdocument ID byte順、`integrates_with`は`document_id` byte順、各`contract_ids`もContract ID byte順とする。`owner_scope`と`non_owner_scope`だけは記述順を意味順として維持する。
- `integrates_with` entryは`document_id`と1件以上のcanonical `contract_ids`だけを持つ。Major／revisionはOwnerのShared canonical contracts表から解決する。Peer側に同じdocument IDと同じContract集合がなければ拒否する。
- `state`は`draft | review | approved | superseded`だけを受理する。
- `approval_ref`は`approved`または`superseded`で必須、それ以外は`null`とする。
- Approval Decisionは対象documentのexact UTF-8 file bytesに対するSHA-256、Git blob ID、承認時刻、承認主体を保持する。Hashをdocument自身へ埋め込まないため自己参照は生じない。
- `external_evidence_verified_at`は外部根拠を使わない文書だけ`null`を許可し、それ以外はUTC基準の`YYYY-MM-DD`とする。

Approval時はDecision IDを先に予約し、documentへ`state=approved`と`approval_ref`を設定した最終bytesを作り、そのSHA-256／Git blob IDをDecisionへ記録して同じChangeSetでcommitする。Read-backで両方を再検証できない場合は`review`のままにする。

従来の`文書ID`、`状態`、`正本範囲`、`非正本範囲`、無型の`依存`は、このblockへ一ChangeSetで置換する。移行中に二つのmetadata入口を併存させない。

### 7.2 Relation semantics

| Relation | 意味 | Cycle | 検査 |
|---|---|---|---|
| `requires` | 本文書のnormative意味を確定する前提Owner | 禁止 | 全体DAG、self edge禁止、存在確認 |
| `integrates_with` | 双方がtyped boundaryで連携するpeer | 許可 | reciprocal edge、境界Contract Owner確認 |
| `supersedes` | 旧文書を置換する一方向関係 | 禁止 | replacement存在、旧文書`superseded` |

External evidence、参考実装、比較対象は本文末のReferenceに置き、document dependencyへ混ぜない。Product PhaseやEvidence Gateは各専用Contract refで結び、`requires`へ代用しない。

`requires`はdirect prerequisiteだけを持ち、別pathで到達できるtransitive edgeを重複登録しない。Architecture lintはtransitive reduction後も同じreachabilityになるedgeをredundantとして拒否する。これにより読む順序と変更影響を過大化させない。

### 7.3 Document lifecycle

```text
draft -> review -> approved -> superseded
          ^          |
          +----------+  normative change
```

| State | 意味 | 許可される行為 |
|---|---|---|
| `draft` | Owner、境界、代替案を作成中 | Prototype計画のみ。Public Contract実装、Promotion禁止 |
| `review` | 内容が完全で検証可能、承認待ち | Fixture／spikeは可能。Production activation禁止 |
| `approved` | Approval refが指すDecisionのread-back済みhashと現在bytesが一致 | Engine機能のimplementation planとActivation candidate作成可能 |
| `superseded` | active正本ではない | `docs/architecture/superseded/`へread-only保持しactive Indexから除外。新規参照禁止 |

承認済みdocumentのfile bytesが一byteでも変われば`approved -> review`へ戻し、`approval_ref=null`として新しいApprovalを要求する。Typo／Link修正を機械的にnormative変更と同一視するわけではないが、exact hashとの矛盾を避けるため承認状態のbypassは設けない。生成Indexだけの変更はactive spec bytesを変えないため各specのApprovalへ影響しない。

### 7.4 Canonical contract ownership

別文書から参照される型はOwner文書の`Shared canonical contracts`表へ一度だけ登録する。

```text
| Contract type | Contract ID | Major | Current revision | Schema hash | Persistence class | Consumers |
```

Consumerは同名型を再定義せず、Owner linkとexact `McdContractRefV1`を使う。MCD実装後はMCD Registryが機械正本となり、Markdown表は生成projectionへ移行する。それまではArchitecture lintが表の一意性とconsumer参照を検査する。

### 7.5 Indexと件数

`active spec`は`docs/architecture/00-product`～`08-domain-packs`にあり、`decisions/`と`superseded/`の配下ではなく、metadataがvalidで、stateが`draft | review | approved`の文書と定義する。`superseded`とDecisionはactive件数へ含めない。

正本件数をDecision本文の完了条件へ固定しない。Indexはdocument metadataから生成し、並び順はDirectory order、文書ID byte順とする。期待件数は検査結果として表示できるが、Architecture invariantにはしない。

### 7.6 Authority discoveryとarchitecture explain projection

旧worktreeにあった別authority Manifestの要件である主題の一意Owner、正本path／anchor、version／content hash、許可されたprojection、MCD参照、変更Review責任は、`ArchitectureMetadataV1`、document relation registry、Shared canonical contracts、生成Architecture Indexへ統合する。別Manifest Schema、互換alias、第二の正本は追加しない。正規authorityはmetadataとregistryのexact entryからのみ解決し、説明文またはAI要約をauthorityへ昇格させない。

Architecture Governanceは、指定Project revisionの構造を人間とAIへ同じEvidenceで提示するread-only／Disposableな`ArchitectureExplainProjectionV1`を生成する。

```text
ArchitectureExplainProjectionV1
  project_id
  project_revision
  contract_set_hash
  scope
  game_system_entries[0..256]
  state_owner_entries[0..256]
  dependency_edges[0..1024]
  runtime_phase_entries[0..256]
  world_entries[0..256]
  level_entries[0..256]
  streaming_entries[0..256]
  capability_entries[0..256]
  target_entries[0..256]
  save_replay_entries[0..256]
  evidence_refs[1..1024]
  omitted_ranges[0..128]
  continuation
```

入力はexact `ArchitectureMetadataV1` set、document relation registry、Product registry、MCD Contract registry、Commit済みWorld／Level／Streaming Source、Target Profile、Project revisionに限定する。各entryはcanonical concept ID、正規Owner document／Contract、optional runtime phaseまたはlifetime、Source StableId、Source content hash、Evidence参照を保持する。各dependency edgeはsource／target canonical concept ID、relation Contract ID、Owner document、Source StableId／hash、Evidence参照を保持する。自然言語説明、Editor配置、外部Engine用語、AIの推測からOwner、edge、phaseを生成しない。

各categoryは256 entry、dependencyは1,024 edge、全canonical encodingは2 MiBを上限とする。上限超過は要約で隠さず`omitted_ranges`とhash-bound `continuation`を返す。Continuation payloadは`request_hash`、`source_closure_hash`、`revision`、`scope`、`expires_at`を持ち、token digestを次に固定する。

```text
SHA-256(JCS({request_hash, source_closure_hash, revision, scope, expires_at}))
```

`request_hash`はcontinuation自体を除くrequest、明示`evaluation_time`、field mask、Target Profile ref、category別next offsetをcanonical化して含み、`source_closure_hash`はmetadata、document relation、Product／Contract registry、World／Target Sourceのhash closureを含む。`expires_at`と検証時刻`evaluation_time`はrequestの明示入力としてcanonical UTCで固定し、generator／encoderがwall clockから生成しない。`expires_at <= evaluation_time`、別revision、scope、field mask、Target、offset、Source closureへの再利用、digest不一致、範囲外offset、必要Evidence欠落は`diagnostic.architecture.explain-continuation-invalid`でfail closedにする。

このdigestはrepository-owned secretやauthenticity署名ではなく、入力binding、破損、誤再利用を検出するread-only cursor integrity値である。悪意あるcallerによるdigest再計算を権限証明として扱わず、continuationからCommit、Approval、Owner変更のauthorityを得ない。authorityを要する操作は既存Approval Contractへ委譲する。同一入力、明示`expires_at`、Source closureからは同一canonical bytesを生成する。

このprojectionはauthority発見と説明のための派生物であり、Project正本、ChangeSet、MCD、Approval、Owner登録を変更しない。変更を行うconsumerはprojection entryを直接Commitせず、canonical Stable ID、typed Operation、expected revisionを正規Gatewayへ再指定する。

`ArchitectureComprehensionCaseV1`と`ArchitectureComprehensionFixtureV1`のSchema、Corpus、grader、昇格GateはAI Verification／Provenanceだけが所有する。Control Planeはexact metadata closure、relation／registry hash、deterministic explain schema hashをEvidence inputとして供給し、Eval期待値または合否を所有しない。Comprehension fixtureはControl Plane baseline read-back後にだけ実行し、stale metadata、omitted Evidence、invalid continuation、存在しないOwner／phaseをnegative Caseとして扱う。

## 8. Compatibility／Evolution model

### 8.1 Version dimensions

| Dimension | Format | 用途 |
|---|---|---|
| Engine release | SemVer | Public capabilityとsupport policy |
| Contract type major | Type suffix `V1`, `V2` | incompatible Schema boundary |
| Contract revision | monotonic `uint32 revision`＋exact `schema_sha256` | 同一major内のadditive／fix revision |
| Contract set | SHA-256 | 一つのBuildが使用するexact closure |
| Native ABI profile | major＋exact profile hash | Native binary load可否 |
| Artifact format | format major／minor＋content hash | Package／Save／Content decode |
| Capability activation | closed state | 利用可能性。Versionではない |

`revision`は同じContract type major内で1から単調増加し、欠番は許すが再利用しない。`schema_sha256`はMCD canonical Schema bytesのSHA-256であり、同じrevisionに別hashを割り当てない。Reader／Writer negotiationと永続参照は両方を記録する。

```text
McdContractRefV1 {
  contract_id, type_major, revision
  schema_sha256, contract_set_sha256
}

ArtifactRefV1 {
  artifact_kind_id
  format_major, format_revision, format_schema_sha256
  content_size
  content_sha256: Sha256DigestV1
}
```

従来の`McdContractRefV1.version`と`ArtifactRefV1.schema_version`はこのFieldへclean置換する。Ref内にrange、bare version、path、URI、`latest`を入れない。Logical Source identityが必要な場合は別subject refで保持し、`ArtifactRefV1`はexact immutable bytesだけを指す。

`latest`、version rangeだけの永続参照、maturityを含むID、document statusを含むIDを禁止する。

### 8.2 Schema evolution

同一major内で許可する変更は次に限定する。

- optional Field追加。ただしdefaultまたはabsence semanticsをOwnerが一意に定義する。
- closed enumを変更しない補足metadata追加。
- 既存の受理集合、既定値、diagnostic identityを変えないvalidator実装修正。以前validだったartifactを拒否する修正は同一majorのfixとして扱わない。
- 既存Field ID、wire type、unit、coordinate、meaningを変えない説明改善。

Major更新を必要とする変更は次である。

- required Field追加。
- Field削除、Field ID変更、wire type／unit／coordinate／meaning変更。
- closed enum value追加または削除。ただし最初からversioned extension registryとして定義したFieldを除く。
- cardinality、range、ordering、default、failure semanticsの非互換変更。
- authority、owner、security boundary、persistent identityの変更。

削除済みField IDとenum codeは永久欠番とし、再利用しない。Suffixなしaliasは作らない。

Unknown Field判定は、Envelopeに記録されたexact type major、revision、schema hashを解決した後に行う。そのSchemaにないFieldは拒否する。新readerは同一majorかつsupport window内の旧revision decoderとfixtureを保持する。Saveは同じepoch内の全Shipping revisionを保持する。旧readerへ未来revisionを読ませない。

Live IPCでrevision negotiationが必要な場合、双方が列挙したexact `{revision, schema_sha256}`集合の最大共通要素を選ぶ。Range、`latest`、hashなしrevisionは使わない。永続artifactはOwnerのcurrent revisionを明記して出力し、暗黙downgradeしない。旧revision writerが必要なら専用encoder、fixture、support期限をCompatibility Registryへ登録する。

### 8.3 Artifact class policy

| Artifact class | Compatibility policy |
|---|---|
| Internal cache／BMI／device cache | 互換なし。exact identity mismatchで破棄し再生成 |
| Derived Artifact／Runtime Package／Application Package | exact Sourceからfull rebuild。旧reader／dual schemaを残さない |
| Project Source | committed revisionを失わない。versioned offline migration、Before／After fixture、rollback必須 |
| Save | 同じ`save_compatibility_epoch`で一度Shippingしたversionを後続releaseが読めることを必須化 |
| Replay | exact baseline／contract setを原則とし、明示Replay migrationがある場合だけcross-version再生 |
| User setting／profile | Saveと同じくmigrationまたは明示reset approvalが必要 |
| Domain Pack Source | declared Engine range、Pack dependency range、migration、exact target qualification |
| Native Game binary | exact ABI profile／contract set。Sourceからrebuildし、暗黙binary shimを作らない |
| Public C++ source API | Post-1.0はSemVer。BMIをToolchain間配布しない |

`save_compatibility_epoch`変更は通常のmajor更新でも自動許可しない。全既存Saveのmigrationを提供するか、User data resetを伴うProduct Decision、明示UI、backup、rollback、Approvalを必要とする。

`SupportPolicyV1`の既定は次とする。

- Pre-1.0の未Shipping内部artifactはclean break可能だが、外部配布済みProject／Save／User settingにはこの例外を適用しない。
- Public C++ source APIとPublic Contractは同じEngine major内の全released minor／patchをsupportする。削除は次majorだけで行い、security emergencyを除き少なくとも一つ前のminorでdeprecation diagnosticとmigration guideを出す。
- Project Sourceはcurrent majorと直前majorからcurrent majorへのdirect offline migrationをsupportする。それ以前は署名・hash固定したarchived migratorをmajor順に適用し、元revisionを上書きしない。
- Saveは同じepochの全Shipping revision、Replayはexact baseline、Domain Packはmanifestが宣言したrangeだけをsupportする。
- Security revokeは通常windowを短縮できるが、Product／Security Decision、影響inventory、safe replacement、User通知、rollback可否を必須とする。

### 8.4 Change classification

| Class | 例 | 必須closure |
|---|---|---|
| `internal_rebuild_only` | cache layout、private adapter | clean rebuild、affected fixture、Package再生成 |
| `additive_contract` | optional Field、new independent capability | old fixture、new fixture、new reader／old writer互換 |
| `migration_required` | Project／Save schema変更 | migrator、Before／After、idempotence、rollback、Package binding |
| `breaking_pre1` | 未公開内部契約のclean break | caller、fixture、docs、artifact全同時更新 |
| `breaking_released` | Public major／Save epoch／ABI break | Product Decision、support plan、migrationまたは明示非互換、全Target Qualification |
| `security_revoke` | compromised key／dependency／provider | fail closed、deactivation、affected inventory、safe replacement、incident Evidence |

`ArchitectureChangeSetV1`はchanged document、contract、Field、Capability、Target、Package、Save、Domain Pack、fixture、migration、approvalのclosureを持つ。影響項目が空であることを「影響なし」の根拠にせず、lintとOwner reviewで照合する。

### 8.5 Canonical byte列とhash

- 永続／wire ContractはExecutable Contractsが所有する`McdCanonicalBinaryV1`で一意のbyte列へencodeする。JSON表示、Markdown、filesystem列挙順、pointer値、locale、時刻をidentity入力にしない。
- Digest型は`Sha256DigestV1`へ統一し、lowercase 64桁hexを表示形式、32 bytesをwire形式とする。Algorithm省略や別algorithmの同じField名を禁止する。
- Manifestは自身のdigestまたは後段artifactのdigestをFieldに持たない。`ManifestArtifactEnvelopeV1`がexact `manifest_ref: ArtifactRefV1`、`payload_root_sha256`、`entry_count`、`trust_profile_ref`を外側で束ねる。Manifest digestは`manifest_ref.content_sha256`だけに置き、別Fieldへ複写しない。Signature refはEnvelopeへ入れず、既存`MirakanSignedRecordV1`がEnvelopeの`ArtifactRefV1`を外側からbindする。
- `payload_root_sha256`はpathをUTF-8 NFC、`/`区切り、case-sensitive logical pathへ正規化し、path byte順に並べた`{path, size, sha256, executable_kind}`のcanonical vectorから計算する。ManifestとEnvelope自身はそのvectorへ含めない。
- `*_refs[]`など集合Fieldは要素canonical bytesのbyte順に並べ、duplicateを拒否する。意味上の順序が必要なsequenceは`order_key`またはdependency DAGをSchemaに明記し、serializer入力順を意味にしない。
- Payload entryはregular fileだけを許可し、空path、absolute path、`.`／`..` segment、backslash、NUL、symlink、hardlink、device、socket、alternate data stream、sparse entry、undeclared nested executableを拒否する。Target固有reserved name／path長制約は各Platform Ownerが追加する。
- Project内のintegrity／trust signatureはcanonical Envelope bytesの`ArtifactRefV1`を`MirakanSignedRecordV1`のJCS subjectへbindする。Platform code／package signingは各formatが定めるbytesを対象にし、署名後のReceiptがcanonical Envelope ref、unsigned artifact ref、signed artifact refを別Fieldでbindする。
- Hash mismatch、duplicate normalized path、case-fold collision、Unicode normalization collision、Manifest／Envelopeの自己entryをfail closedで拒否する。

### 8.6 Game compatibility subject

`GameCompatibilitySubjectV1`はSave、Runtime Package、Application Package、最終Candidateを結ぶTarget非依存のimmutable inputであり、V1 revision 1の必須Fieldを次へ固定する。追加は8.2のadditive規則だけで行う。

```text
project_id, project_revision
engine_release_version, engine_baseline_hash
contract_set_hash, gameplay_contract_set_hash
content_semantic_set_hash, project_shader_semantic_set_hash
save_compatibility_epoch, save_schema_set_hash, migration_set_hash
domain_pack_resolved_lock_hash
compatibility_policy_ref, source_provenance_ref
```

Target profile、native binary、Runtime Package、Application Package、signature、Receipt、Candidate hash、wall-clock timestampは含めない。Canonical subject bytesの`ArtifactRefV1`がexact subject refとなる。入力が一つでも変われば新subjectを作り、二つのsubject間の読込可否はhash equalityではなくartifact class別`CompatibilityDecisionV1`とmigration coverageで決める。

### 8.7 新機能／変更の標準フロー

新機能は既存defaultや既存Contractへ暗黙追加せず、次の順序で導入する。

1. Product Planへmaturityなしのstable Capability ID、`target_product_tier`、Owner、fallback、対象外を登録し、Activationは`not_activated`から開始する。
2. 既存Ownerで意味が閉じるかを確認し、閉じなければArchitecture Decisionで新Ownerと非Owner範囲を決める。共有型をconsumer文書へ先に書かない。
3. `ArchitectureChangeSetV1`でartifact class、Schema、Save、Replay、Runtime Package、Application Package、Target、Domain Pack、Security、Performanceへのimpact closureを列挙する。
4. 新しい独立ContractはV1で追加する。既存Contract変更は8.2の分類に従い、additive revisionまたはnew major＋migrationを選ぶ。
5. Feature選択はCapability Registryとtyped composition manifestだけで行う。散在するboolean、directory scan、registration order、Backend存在をActivation条件にしない。
6. Source、Runtime、Save／Replay、Package、Target adapterをtyped Portで接続し、feature absent、unsupported Target、stale Receipt、migration欠落、partial failureのnegative fixtureを先に定義する。
7. `candidate_locked -> qualified -> production`をTarget別Evidenceで進める。未選択Projectと旧Saveの挙動が変わらないことを回帰fixtureで確認する。

既存機能の削除はFeature flagを消すだけで行わない。Public／User data impactを`breaking_released`として扱い、deprecation、Project／Save migration、fallback、Package cleanup、Target Qualification、Approvalを同じChangeSetで閉じる。

## 9. Stable IDと状態モデル

### 9.1 ID grammar

| ID | 規則 |
|---|---|
| Capability | `capability.<domain>.<name>`。C0～C3、candidate、production、`v1`を含めない |
| Work Package | `wp.<domain>.<name>`。Phaseはidentityへ含めずRegistryの`phase_id`で登録 |
| Operation | `operation.<domain>.<verb_noun>`。Schema majorはOperation contract refに置く |
| Schema Contract | `contract.<domain>.<name>`。Type majorはIDでなくSchema type suffixに置く |
| MCD Domain object | Executable Contractsへ登録した`<kind>.<namespace_path>`。例: `game_system.<domain>.<name>`。version／maturityは専用Field |
| Diagnostic | `diagnostic.<domain>.<condition>` |
| Driver／Profile | logical IDにversionとmaturityを含めない。Versionは専用Field、exact artifactはlockで解決 |

document ID、MCD object、Product Registry logical IDは[Naming／Project Layout §3.2](../architecture/02-foundation/naming-project-layout.md#32-stable-idとoperation)のkind別grammarに従う。すべてのkindで数字開始segment、同一segment内の`-`／`_`混在、maturity／version埋込みを拒否する。Capabilityのmaturity、Work PackageのPhase、Driver／Profileのversionは別Fieldで保持し、昇格、再計画、更新でlogical IDを変えない。旧IDは同じChangeSet内のoffline migration tableで新IDへ一度だけ写像し、runtime aliasを残さない。

### 9.2 State axes

| Axis | Closed state | Owner |
|---|---|---|
| Document lifecycle | `draft | review | approved | superseded` | Architecture Governance |
| Capability product tier | `C0 | C1 | C2 | C3`。到達scopeの分類でありstate transitionではない | Product Plan |
| Capability activation | `not_activated | candidate_locked | qualified | production` | Product Plan |
| Work Package scheduling | `declared | scheduled | active | blocked | complete | deferred` | Product Plan |
| MCD registration | `draft | active | retired` | Executable Contracts |
| Target readiness | `predicted | blocked | qualified` | Project State |
| Target blocked reason | closed `TargetBlockedReasonRegistryV1`の`DiagnosticRefV1`。初期共通値は`optimization_required` | 各Domainが値を登録、Project Stateがenvelopeを所有 |
| Evidence freshness | `fresh | expiring | expired | revoked` | AI Verification／Provenance |
| Dependency adoption | `proposed | locked | qualified | active | rejected | revoked` | Toolchain／Dependencies |

Capabilityは`target_product_tier`と`activation_state`を別Fieldに持つ。C2を目標にしたCapabilityが未実装なら`target_product_tier=C2`かつ`activation_state=not_activated`であり、Tierを実装済み表示に使わない。`C2CapabilityCoverageMatrixV1.lifecycle_state`は廃止し、`capability_activation_state`と`owner_work_package_ref`を参照する。`OptimizationRequired`をCapability activation stateとして使わず、Target readinessの`blocked_reason`として扱う。Dependencyへ`candidate_locked`を使わない。

## 10. Persistence／Save design

### 10.1 Canonical contracts

| Contract | Owner responsibility |
|---|---|
| `SaveSlotManifestV1` | User-visible slot identity、active generation、timestampsの非権威metadata、conflict state |
| `SaveRootManifestV1` | `GameCompatibilitySubjectV1`、Engine／Contract／Content／Domain Pack closure、Save epoch、checkpoint ref |
| `SaveCheckpointV1` | published tick、World set、System record set、RNG／clock、authoritative digest |
| `SaveDomainRecordSetV1` | owner別bounded record envelope。Domain field意味はOwner Contract参照 |
| `SaveMigrationPlanV1` | source／destination revision、ordered steps、owner、precondition、rollback |
| `SaveLoadPlanV1` | exact package／content／migration／capacity／external preparation closure |
| `SaveStoragePolicyV1` | integrity、confidentiality、compression、stored／decoded size bound、backup generation、key profile要求 |
| `SaveStorageEnvelopeV1` | canonical payload digest、stored byte digest、transform profile、nonce／auth tag ref、key version。Key自体は含めない |
| `PlatformStorageTransactionV1` | temporary write、flush、read-back、atomic replaceまたはjournal要求 |
| `SaveCommitReceiptV1` | written generation、read-back hash、commit marker、previous generation |
| `SaveLoadReceiptV1` | migration、staging、digest、publication result |

`SaveCatalogV1`はUI／Settings文書のlocal projectionであり、payload正本ではない。UIは`SaveSlotManifestV1`のbounded projectionを表示し、Save bytesを解釈しない。

### 10.2 Capture sequence

1. GameplayまたはUIがtyped Save requestを発行する。
2. Scheduling Ownerが許可されたpublished tick boundaryを選ぶ。
3. Persistence coordinatorがactive Root、Section、System、Save contract、Content、Domain Pack lockをfreezeする。
4. 各Owner projectorが`SaveReplayContractV1`に列挙されたFieldだけをcanonical recordへ出力する。
5. ECS OwnerはPersistent Identity、lifecycle、composition、enablement、Field projectionを出し、raw handle／chunk／row／paddingを出さない。
6. Persistence coordinatorが全recordのOwner、schema、bound、ordering、digest、cross-referenceを検証する。
7. Canonical payloadとexact `SaveStoragePolicyV1`を一つの`PlatformStorageTransactionV1`でPlatform adapterへ渡す。
8. Platform adapterが`SaveStorageEnvelopeV1`を作り、temporary write、flush、再読込hash、atomic replaceまたはrecoverable journal、commit markerを順に実行する。
9. Plaintext／stored digestとread-backが一致した後だけ`SaveCommitReceiptV1`を確定しactive generationを切り替える。

失敗時は旧active generationを維持する。部分System Save、last-write-wins、同generation競合の自動選択を禁止する。

### 10.3 Load sequence

1. `SaveStorageEnvelopeV1`をbounded parseし、stored size、stored digest、trust／authentication、key versionを検証する。
2. Policy bound内でdecrypt／decompressし、canonical payload sizeとdigestを検証する。
3. `SaveRootManifestV1`のSave epoch、`GameCompatibilitySubjectV1`、Schema set、checkpoint refを検証する。
4. exact Runtime Package、Content Package、Domain Pack lockとmigration coverageを解決する。
5. `SaveMigrationPlanV1`をsource revisionからdestination revisionまで一意に構成する。0経路または複数経路は拒否する。
6. World／System State／external reservationを非公開stagingへ構築する。
7. Domain owner順ではなく、明示migration dependency DAGのtopological orderでmigrationする。
8. Identity、composition、capacity、Owner、Contract、authoritative digestを検証する。
9. 全検査成功後だけ一つのpublication boundaryで新Worldを公開する。
10. 失敗時はstagingを破棄し、Titleまたは旧Worldを維持する。

Platform cloud conflictはtransportが検出し、Persistenceが二つのgenerationをUserへ提示する。Save payloadをfield単位で自動mergeしない。

### 10.4 Replay relation

`ReplaySliceV1.start_checkpoint`は曖昧なinline objectをやめ、exact `ArtifactRefV1`で`SaveCheckpointV1`またはReplay専用の同一checkpoint envelopeを参照する。Replay event streamはcheckpoint以後のaccepted Command／Event、clock、RNG、content generationを持ち、Save slot metadataへ依存しない。

### 10.5 Storage／security envelope

Saveのcanonical payload digestはcompression／encryption前の`McdCanonicalBinaryV1` bytesから計算する。Platform adapterは`SaveStoragePolicyV1`に従い、bounded compressionとplatform protected encryptionを適用し、stored bytesのdigestとauthentication結果を`SaveStorageEnvelopeV1`へ記録する。Integrity検証は全Profileで必須、`confidentiality_mode=none`はProduct／Platform policyが明示許可したoffline profileだけに限定する。

Loadはstored size bound、stored digest、auth tag、key versionを検証してからdecryptし、decoded size／compression ratio bound内でdecompressし、最後にcanonical payload digestとSchemaを検証する。Key materialをSave、Project、log、Receiptへ書かない。Key rotationまたはstorage transform変更はplaintext digestとSave Schemaを変えず、新generationへのrewrap transactionとして行い、旧generationをread-back完了まで維持する。

## 11. Runtime Package design

`RuntimePackageManifestV1` revision 1の必須Fieldを次へ固定する。追加は8.2のadditive規則だけで行う。

```text
runtime_package_id, format_major, format_minor
engine_release_version, engine_baseline_hash
project_revision, game_compatibility_subject_ref
target_profile_ref, toolchain_lock_hash
contract_set_hash, native_abi_profile_hash
system_implementation_set_ref
runtime_world_root_refs[]
content_package_refs[]
project_shader_artifact_refs[]
save_compatibility_epoch
save_schema_set_hash, migration_set_hash
domain_pack_lock_hash
entrypoint_ref, dependency_entries[]
```

Manifest内で`GameCompatibilitySubjectV1`と重複するEngine、Project、Contract、Save、Domain Pack Fieldはすべてsubjectの値とbyte equalityにし、片側だけの更新を拒否する。重複Fieldはloaderのearly diagnosticとTarget package検索用projectionであり、別Authorityではない。

`RuntimePackageArtifactV1`は`ManifestArtifactEnvelopeV1`を使い、`runtime_package_manifest_ref: ArtifactRefV1`、`payload_root_sha256`、entry count、`runtime_trust_profile_ref`、optional `MirakanSignedRecordV1` refをManifest外で保持する。Development profileでもdigest検証は必須で、Shipping profileはTarget policyが要求する署名またはplatform trust Receiptを必須とする。

LoaderはEnvelope、Manifest hash、payload root、trust／signature、Engine baseline、Compatibility subject、Target、ABI、Contract、System、World、Content、Shader、Save migration、Domain Packの順に検証する。順序はdiagnosticの一意性のため固定し、一つでも不一致なら何もactivateしない。

Development loose layoutも同じmanifestとvalidationを使用する。Path scanやDirectory存在だけでload可否を決めない。Runtime Packageはimmutableであり、hot reloadは新generationをstagingしてsafe boundaryで全closureを切り替える。

## 12. Application Package／Release design

### 12.1 Common assembly manifest

`ApplicationPackageAssemblyManifestV1` revision 1の必須Fieldを次へ固定する。追加は8.2のadditive規則だけで行う。

```text
assembly_id, target_profile_ref, distribution_profile_ref
engine_baseline_hash, project_revision, game_compatibility_subject_ref
runtime_package_ref, content_package_refs[]
platform_binary_refs[], shader_artifact_refs[]
resource_manifest_ref, permission_policy_ref
privacy_policy_ref, entitlement_policy_ref
sbom_ref, provenance_ref
normalized_entries[] { path, size, sha256, executable_kind }
```

AssemblyはContent／Shaderを独自選択しない。`engine_baseline_hash`、`project_revision`、`game_compatibility_subject_ref`、Target、Runtime Package refを検証し、`content_package_refs[]`と`shader_artifact_refs[]`はRuntime Package closureとset equalityにする。Platform固有Resourceだけを`resource_manifest_ref`へ追加し、Runtime closureとの差分を一般Contentとして隠さない。

Windows、Android、Appleは共通manifestを参照するTarget固有envelopeを持つ。

- `UnsignedWindowsPackageV1`
- `UnsignedAndroidPackageV1`。既存`UnsignedMobilePackageV1`を明確化して置換する。
- `UnsignedApplePayloadV1`

共通`UnsignedApplicationPackageV1` envelopeは`assembly_manifest_ref: ArtifactRefV1`、`payload_root_sha256`、entry count、Target固有envelope refを持つ。Manifest digestは`assembly_manifest_ref.content_sha256`だけに置く。Manifest、共通envelope、Target固有envelopeはpayload entryへ含めず、8.5の外側hash規則で束ねる。

### 12.2 Target package preparation

```text
planned
  -> built
  -> validated
  -> signing_authorized
  -> signed
  -> signature_verified
  -> staging_uploaded
  -> staging_read_back_verified
  -> prepared
```

`TargetPackagePreparationTransactionV1`は一TargetのCandidate入力を作る。Release Coordinatorだけがstate writerであり、各遷移はprevious state、immutable input hash、Receipt refをcompare-and-swapする。Transaction／attempt identityはUUIDv7 `StableId`、retryは同じtransactionを巻き戻さず、新attempt IDと`parent_attempt_ref`を同じimmutable inputへ結ぶ。Source、manifest、Policy、Targetのhashが変われば新transactionである。任意段階から`failed`へ遷移できる。`staging_kind`は`store_draft | internal_test_track | private_artifact_registry`のclosed enumとする。`staging_uploaded`はいずれも非公開であり、販売、public rolloutを許可しない。

Build identityはSigning／Upload secretを持たない。Signing identityはSource、compiler、arbitrary scriptを受けない。Upload identityはSourceとprivate keyを持たない。Signing Serviceはimmutable unsigned Target artifactと`UnsignedApplicationPackageV1`だけを受け、`assembly_manifest_ref`のcontent digestと`payload_root_sha256`を再計算する。Platform署名は各formatが定めるbytesへ施し、`ReleaseSigningReceiptV1`がcanonical envelope hash、unsigned artifact hash、signed artifact hashを同時にbindする。

`PackageValidationReceiptV1`、`ReleaseSigningReceiptV1`、`StoreStagingUploadReceiptV1`、`StoreStagingReadBackReceiptV1`が同じexact assembly manifest refとunsigned payload rootを参照した場合だけ`TargetPackagePreparationRecordV1`を発行できる。

### 12.3 Candidate identityと循環防止

`GameCompatibilitySubjectV1`はPackage生成前に確定できるTarget非依存入力であり、Runtime Package、Application assembly、Save Rootへ埋め込める。`TargetPackagePreparationRecordV1`は一Targetの署名／staging upload／read-back完了を表す。AI Security／Approval Ownerの`GameCandidateManifestV1`は複数Target recordとwhole-game Attestationを集約する後段Governance recordであり、前段ManifestまたはPackage payloadへ埋め込まない。

```text
GameCompatibilitySubjectV1
  -> RuntimePackageArtifactV1
  -> UnsignedApplicationPackageV1
  -> signed target artifact + Receipts
  -> TargetPackagePreparationRecordV1
  -> AI Security `GameCandidateManifestV1`
```

`TargetPackagePreparationRecordV1`はsubject ref、Target profile ref、assembly manifest ref、unsigned／signed／staged artifact ref、全Receipt ref、release policy refを持つ。AI Security OwnerはCandidateへsubject refとTarget preparation record refsを登録する。Candidate hashは最終Manifestの外側Envelopeで計算するため、Package hashとCandidate hashの循環を作らない。

### 12.4 Store publication transaction

```text
planned
  -> candidate_validated
  -> human_approval_validated
  -> submitted
  -> accepted
  -> rollout_authorized
  -> publication_requested
  -> public_read_back_verified
  -> complete
```

`ReleaseTransactionV1`はexact `GameCandidateManifestV1`、有効な`HumanGameplayApprovalV1`と`GameActivationReceiptV1`、Target preparation record集合、Distribution policy、rollout policyだけを入力にする。全refは同じCandidate hashへ閉じなければならない。Release Coordinatorだけがstate writerで、attempt／retry規則はTarget preparationと同じである。`submitted`以後のStore mutationはUpload／Release identityだけが行い、Source、Build tool、Signing keyを受けない。Store rejectionは`rejected`、技術失敗は`failed`、人間またはPolicyによる公開前中止は`cancelled`とし、いずれもterminalである。Retryは新attempt IDを作る。

`publication_requested`はStore APIがcommandを受理しただけであり公開成功を意味しない。公開channel、version、region、artifact hashをremote read-backし、Candidateと一致した`StorePublicationReceiptV1`発行後だけ`complete`へ進む。段階rolloutの割合変更、停止、rollbackは同じCandidateを指す別`ReleaseRolloutCommandV1`とReceiptで記録し、Build、Package、Candidate identityを変更しない。

## 13. Shared canonical contract Owner整理

### 13.1 新規／変更する横断Contract

| Contract | 唯一のOwner |
|---|---|
| Architecture metadata JSON schema、`ArchitectureChangeSetV1` | Architecture Governance |
| `EngineContractVersionV1`、`EngineCompatibilityRangeV1`、`DomainPackCompatibilityRangeV1`、`CompatibilityDecisionV1`、`MigrationCoverageV1`、`SupportPolicyV1`、`GameCompatibilitySubjectV1` | Compatibility／Evolution |
| `McdContractRefV1`、`ArtifactRefV1`、`Sha256DigestV1`、`ManifestArtifactEnvelopeV1` | Executable Contracts。Compatibility規則を消費するが構造はここだけで定義 |
| `MirakanSignedRecordV1`の共通envelope／hash chain | AI Verification／Provenance。Algorithm、Key用途、authorization policyはAI Security／Approval |
| `CapabilityRegistryV1`、`CapabilityTargetActivationV1`、`ProductPhaseRegistryV1`、`WorkPackageRegistryV1`、`TargetProfileRegistryV1`、`RequirementRegistryV1`、`FixtureRegistryV1`、`FallbackRegistryV1` | Product Plan |
| `SaveSlotManifestV1`、`SaveRootManifestV1`、`SaveCheckpointV1`、`SaveDomainRecordSetV1`、`SaveMigrationPlanV1`、`SaveLoadPlanV1`、`SaveStoragePolicyV1`、`SaveStorageEnvelopeV1`、`PlatformStorageTransactionV1`、Save Receipt群 | Persistence／Save。Key管理／OS crypto実装は各Platform Owner |
| `RuntimePackageManifestV1`、`RuntimePackageArtifactV1` | Runtime Package |
| `ApplicationPackageAssemblyManifestV1`、`UnsignedApplicationPackageV1`、Target mapping rule、`PackageValidationReceiptV1`、`TargetPackagePreparationTransactionV1`、`ReleaseTransactionV1`、`ReleaseSigningReceiptV1`、`StoreStagingUploadReceiptV1`、`StoreStagingReadBackReceiptV1`、`TargetPackagePreparationRecordV1`、`StorePublicationReceiptV1`、`ReleaseRolloutCommandV1` | Application Package／Release |
| `UnsignedWindowsPackageV1`／`UnsignedAndroidPackageV1`／`UnsignedApplePayloadV1`のTarget固有Fieldとformat | Windows／Android／Appleの各Platform Owner |
| `GameCandidateManifestV1`、`GameActivationReceiptV1`、`HumanGameplayApprovalV1` | AI Security／Approval。Target packageの内部構造は再定義せず、`TargetPackagePreparationRecordV1`を参照 |
| `DomainPackManifestV1`、`DomainPackResolvedLockV1` | Domain Pack Contract |

### 13.2 既存の未確定projection

| 現在の型／概念 | 決定するOwner／処置 |
|---|---|
| `TargetProfileRef` | `TargetProfileRefV1`へ改名。構造はExecutable Contracts、値とQualificationは各Platform |
| `TargetCapabilitySnapshotV1` | 共通envelopeはExecutable Contracts、Target固有entryは各Platform |
| `TargetBlockedReasonRegistryV1` | Registry構造と共通値はProject State、Domain固有`DiagnosticRefV1`は各Domain Ownerが一意登録 |
| `LightingBudgetEnvelopeV1` | Performance／Capacity |
| `PostProcessBudgetEnvelopeV1` | Performance／Capacity |
| `EnvironmentLightingSummaryV1` | Environment／Surfaces |
| `MaterialReadabilitySummaryV1` | Materials |
| `CameraPresentationSummaryV1` | Camera |
| `LayerCompositionSummaryV1` | Render Graph |
| `AccessibilityPolicySnapshotV1` | UI／Text／Localization／Accessibility |
| `PostProcessVolumeSummaryV1` | Post Processing |
| `PolicySnapshotV1` | generic名を廃止し`LightingResolutionPolicySnapshotV1`としてLightingが所有 |
| `StableSourcePathV1` | Asset Lifecycleで構造、normalization、case／Unicode規則を定義 |
| `ProviderManifest` | `ProviderManifestV1`としてAI Security／Approvalが所有 |
| `ReleaseTransactionV1` | Application Package／Release |

Consumer文書にはfield一覧を複写せず、Owner link、revision、freshness、read-only、write-back禁止だけを残す。

## 14. Domain Pack evolution

`DomainPackManifest`は`DomainPackManifestV1`へ改名し、V1 revision 1の必須Fieldを次へ固定する。追加は8.2のadditive規則だけで行う。

```text
pack_id, pack_version
engine_compatibility_range: EngineCompatibilityRangeV1
pack_dependencies[]
  pack_id
  version_range: DomainPackCompatibilityRangeV1
  dependency_kind: required | optional
  activation_capability_ref
contract_refs[], capability_refs[]
migration_step_refs[]
qualification_policy_ref
```

`pack_version`はSemVer 2.0.0のvalid versionとし、`production` Activationではpre-releaseを拒否する。Candidate／qualified profileでpre-releaseを使う場合はrangeのprerelease許可flagとQualification policyの両方を必須とする。`EngineCompatibilityRangeV1`と`DomainPackCompatibilityRangeV1`は自由形式文字列やnpmのloose rangeを使わず、normalized lower bound、upper bound、各inclusive flag、prerelease許可flagのtyped tupleとする。Unbounded sideは明示`null`、空集合と全version集合を別stateで表し、Build時にexact versionとartifact hashへ解決する。`dependency_kind=required`では`activation_capability_ref=null`、`optional`ではnon-nullを必須とする。Optional dependencyはそのCapabilityが明示選択された場合だけclosureへ入り、Installed packageやnetwork結果から自動Activationしない。

Install時にrangeを解決しただけではActivationしない。`DomainPackResolvedLockV1`がexact Pack version、artifact hash、Engine baseline、Contract set、dependency closureを固定し、そのexact lockに対するTarget別Qualification Receiptが必要である。

Pack dependency cycle、欠落、lock未選択の複数解、range外、同一Pack ID別hashを拒否する。Resolverは`latest`を自動選択せず、承認済み選択をexact lockへ書く。Patch／minorはcompatibility fixtureを必須とし、persisted Sourceが変わる場合だけ同一major migrationを追加する。Majorはoffline migrationと明示承認を必要とする。Runtime shim、旧名alias、synthetic dependencyを作らない。

Shooter Packの`feature.*.c1`、Profile／Capability IDからmaturityを除去し、C1はRegistryの`target_product_tier`列で表す。現在状態は別軸のActivation stateが持ち、TierをActivation stateで表現しない。

## 15. Product、Phase、Work Package整合

Product Planへ`WorkPackageRegistryV1`を追加する。各entryは次を持つ。

```text
work_package_id, phase_id, owner_document_id
requirement_refs[], capability_refs[]
requires_work_package_refs[]
exit_fixture_refs[], target_refs[]
schedule_state, completion_receipt_refs[]
```

2D coverage Work Packageの正本IDはProduct Plan §11.5の`wp.product.general-coverage-2d`であり、Registryの`phase_id`でPhase 8へ登録する。将来Phaseを移してもIDは変えない。旧表記`WP7a3_2d_product_coverage_c2`はactive specに残存しないため、実装計画Appendix Dへmigration rowを作らない。Capability Coverage MatrixはWork Package scheduleとCapability activationを再定義しない。

`requires_work_package_refs`はcycleを禁止し、PrerequisiteのPhase ordinalがConsumerより後なら拒否する。同Phase内はtopological orderを使う。`schedule_state=complete`は全exit fixtureのfresh Receipt、approved Owner document、Target closureが揃う場合だけ許可する。`deferred`はdefer reasonと再検討Gateを必須とし、Registryから削除しない。

Phase Gateは二種類を明示する。

- `contract_fixture_gate`: 後続Target実装前でもheadless schema／negative fixtureとして実行できる。
- `product_target_gate`: 実Platform、package、install、device Receiptが必要で、該当Phase前には成功扱いにしない。

Phase 4のProject Source Activationは`wp.authoring.project-native-module`（Owner `mirakan.arch.native-game-module`）と`wp.rendering.project-shader`（Owner `mirakan.arch.rendering-project-shader`）をWork Package Registryへ登録し、それぞれ`capability.project.native_module`と`capability.project.shader`を所有させる。両Capabilityは`target.windows.editor`と`target.windows.desktop`だけをrequiredとし、Phase 4の`fixture.product.shooter-2d`がAI Authoring、MVP completion、Source／Diff／Approval／Artifact／Receipt closureを同一Project revisionで検証する。Android／Appleは別Qualificationまで`excluded`であり、Windows Receiptから推論しない。

Scheduling Phase 0のSave／Platform lifecycle項目はcontract fixtureに限定する。Windowsの空Scene save／packageはPhase 2、Android／Apple packageはPhase 7のproduct target gateとする。C++ Modules Phase 0のMobile recipeはcompile／link fixtureであり、Store package合格を意味しない。

## 16. Runtime ECS Decisionとの関係

Runtime ECS設計のarchetype／SoA、closed transition、single writer、access manifest、staged structural transaction、canonical iteration、raw memory非永続化は維持する。ただしactive化前に次を修正する。

昇格先は`docs/architecture/04-runtime/entity-component-system.md`、stable document IDは`mirakan.arch.runtime-entity-component-system`とする。Decision fileをactive specとして数えず、昇格時にDecision refをmetadataのApprovalへ記録する。

1. 「既存Save owner」参照を新Persistence／Save正本へ置換する。
2. ECS Save sectionはComponent／Entity projectionだけを所有し、slot／aggregate／migration orchestration／file transactionを所有しない。
3. `DerivedArtifactManifestV2`は、未実装のsuffixなし型から移るなら`DerivedArtifactManifestV1`としてclean定義する。V2を使う場合は実在するreleased V1とmigration履歴を示す。（ECS Decision本文は現行`DerivedArtifactManifestV1`へ統一済みであり、本項のV2→V1改名分は完了している。）
4. Runtime World Root／Section artifactはRuntime Package manifestとContent Group closureへ接続する。
5. `McdCanonicalBinaryV1`をCompatibility／EvolutionのSchema evolution規則へ接続する。
6. `NativeSystemDescriptorV1`変更をNative ABI compatibility classへ登録する。
7. 16 KiB chunkは`ecs_chunk_soa_v1` profile内の実装値であり、Save、Replay、Artifact identityへ露出しないことを維持する。
8. Decision内の固定文書件数を削除し、生成Indexを参照する。
9. ECS Decision本文のrebaseは全体ChangeSet内で行えるが、ECS active specへの昇格は5新正本とECSが直接`requires`する全Ownerの`approved`後に別Approvalを必要とする。

## 17. Toolchain correction

Toolchain／DependenciesのFreeType 2.14.1はannotated tag objectとpeeled commitを分離する。

```text
tag: VER-2-14-1
tag_object: 3bd82b5f543bc84ccf2b1d0cdb63b95218099ee6
peeled_commit: 526ec5c47b9ebccc4754c85ac0c0cdf7c85a5e9b
```

TypeScript 7.0.2はnative compilerでprogrammatic APIを持たないため、既存どおりCLIだけを使用し、正式Artifactでは`--singleThreaded`を固定する。npm artifactはversion、tarball URL、size、SHA-256、registry integrity、metadata response SHA-256を必須lockとする。`registry_provenance_state`は`verified | not_provided`のclosed stateとし、signature／attestationが存在すれば検証失敗を拒否、存在しなければ`not_provided`を明記してofficial repository tag／commitとartifact digestで補完する。`gitHead`はregistry metadataに存在する場合だけexact値、欠落時は`null`とし推測しない。

OpenAI Model IDはimmutable snapshotが提供される場合はsnapshotを使う。非snapshot IDの場合は同じ出力の再現を主張せず、resolved ID、Provider manifest、request parameters、tool／schema hash、Eval、expiry、Receiptで採用判断を再現する。

## 18. 公式一次資料と検証記録

外部仕様に対しては公式一次資料をconstraintとして使い、Miraikanai固有のOwner分割、Save transaction、Package identityを「外部製品の公式標準」とは表現しない。後者は一次資料と既存43仕様を根拠にした本Projectの設計判断である。

| 検証対象 | 公式一次資料 | 本設計への反映 |
|---|---|---|
| Public release version | [Semantic Versioning 2.0.0](https://semver.org/) | Public APIを宣言し、Post-1.0のbreaking／additive／fixをmajor／minor／patchへ対応 |
| Field進化 | [Protocol Buffers best practices](https://protobuf.dev/best-practices/dos-donts/) | Field number／enum code再利用禁止、required追加・default変更・型変更を非互換扱い。MCD固有規則としてさらにstrict化 |
| Canonical hash | [Protocol Buffers serialization is not canonical](https://protobuf.dev/programming-guides/serialization-not-canonical/) | 汎用serializer出力をhash正本にせず、`McdCanonicalBinaryV1`のcanonical bytesだけを使用 |
| Engine／Extension rangeの比較例 | [Godot `.gdextension` compatibility](https://docs.godotengine.org/en/4.4/tutorials/scripting/gdextension/gdextension_file.html) | minimumだけでなく上限も表せるtyped rangeとexact resolved lockを分離。Godot format自体は採用しない |
| TypeScript 7 | [Microsoft TypeScript 7.0 announcement](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/) | 7.0はstable programmatic APIを持たないためCLIのみ。正式lint buildへ`--singleThreaded`を固定 |
| MCP | [MCP specification 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25) | Protocol versionをexact lockし、Tool schema validation、access control、人間確認を既存Governanceへ接続 |
| OpenAI model | [GPT-5.6 Sol official model reference](https://developers.openai.com/api/docs/models/gpt-5.6-sol) | 2026-07-22時点で別のdated snapshot IDを確認できないため、non-snapshot扱いでresolved ID、Eval、expiry、Receiptを記録 |
| Android toolchain | [AGP 9.3.0 release notes](https://developer.android.com/build/releases/agp-9-3-0-release-notes) | Gradle 9.5.0、Build Tools 36.0.0、JDK 17、maximum API 37を既存pinと照合。明示NDK pinは別lockで維持 |
| Apple toolchain | [Xcode 26.6 release notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-26_6-release-notes) | Xcode 26.6と26.5 SDK群を既存pinと照合 |
| Store staging／publish | [Google Play tracks／draft／staged rollout](https://developers.google.com/android-publisher/tracks)、[App Store submission](https://developer.apple.com/documentation/appstoreconnectapi/app-store-version-submissions)、[Microsoft Store submission API](https://learn.microsoft.com/en-us/windows/apps/publish/store-submission-api) | 非公開preparationと公開transactionを分離し、Target adapterが外部stateを共通closed stateへ写像。公開後はremote read-back Receiptを要求 |
| FreeType tag identity | [FreeType official repository tag](https://gitlab.freedesktop.org/freetype/freetype/-/tags/VER-2-14-1) | `git ls-remote --tags`でtag object `3bd82...`とpeeled commit `526ec...`を区別 |

Context7の`/microsoft/typescript-go`は照会時点でpreview／main branch資料を返し、stable 7.0.2をversion indexへまだ収録していなかった。この時間差は[Microsoftの7.0 GA資料](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0/)と[npm registryの7.0.2 metadata](https://registry.npmjs.org/typescript/7.0.2)で補完し、Context7のpreview記述をGA挙動より優先しない。

Node、npm、CMake、Ninja、LLVM、DXC、vcpkg、Rendering／Physics／Navigation／Text dependencyのexact release、artifact size、digest、licenseは既存[Toolchain／Dependenciesの一次資料表](../architecture/02-foundation/toolchain-dependencies.md#10-context7と公式一次資料)を正本とする。本設計実行時にもremote refとdownloaded artifactを再検証し、設計レビュー日の結果だけを将来Buildの証拠へ流用しない。

## 19. 全active spec影響マトリクス

| Existing spec | 本横断ChangeSetの必須変更／後続boundary |
|---|---|
| `00-product/product-plan.md` | state axes、Work Package Registry、Capability ID、Phase gate分類、C2 Matrix統合 |
| `01-governance/ai-security-approval.md` | `ProviderManifestV1`、Release authorization、`GameCandidateManifestV1`のsubject／Target preparation record参照 |
| `01-governance/ai-verification-provenance.md` | Package／Save／Compatibility Receipt binding、freshness state |
| `02-foundation/core-architecture.md` | Port／Adapter依存方向図、Feature開始Gateの`approved`定義、新Owner参照 |
| `02-foundation/cpp23-modules.md` | compile fixtureとProduct package gate分離、ABI／BMI compatibility |
| `02-foundation/executable-contracts.md` | Target共通ref／snapshot、Contract revision、shared-owner projection |
| `02-foundation/math-core.md` | metadata relation、Compatibility class参照。algorithm変更なし |
| `02-foundation/memory-pointers.md` | metadata relation、ECS固有値の移管前提、ABI／serialization禁止参照 |
| `02-foundation/naming-project-layout.md` | ID grammar、Schema suffix例外0、migration path規則 |
| `02-foundation/toolchain-dependencies.md` | Dependency adoption state、FreeType peeled commit、npm provenance。[D3D12 Companion]後続ChangeSetでAgility SDK／D3D12MAのexact pin、release evidence、runtime inspectionを接続 |
| `03-authoring/asset-lifecycle.md` | suffixless型修正、`StableSourcePathV1`、`DerivedArtifactManifestV1`、Runtime Package handoff |
| `03-authoring/editor-ui-framework.md` | document relation、Package closure、shared contract ref。UI algorithm変更なし。[D3D12 Companion]後続ChangeSetで`ui_d3d12_adapter`を`ui_render_graph_adapter`へ置換し、native ownerの二重化を解消 |
| `03-authoring/editor-workspace-ux.md` | document lifecycle表示、Save／Target readiness表示語彙 |
| `03-authoring/gameplay-programming-model.md` | `SaveReplayContractV1`をPersistenceへ接続、Capability ID、ABI impact |
| `03-authoring/native-game-module.md` | exact ABI compatibility、Package manifest binding、ECS Decision前提 |
| `03-authoring/project-state.md` | Target readiness rename、Project migration、`GameCompatibilitySubjectV1` input、Runtime Package boundary |
| `04-runtime/debugging-observability-replay.md` | Replay start checkpoint、Replay compatibility、SaveとのOwner分離 |
| `04-runtime/performance-capacity.md` | shared Budget Envelope、Qualification／readiness mapping |
| `04-runtime/scheduling-lifetime.md` | Save aggregate Owner移管、capture／load boundary、Phase 0 gate修正 |
| `05-simulation/animation.md` | Save projector／migration、typed relation。algorithm変更なし |
| `05-simulation/collision.md` | Save／Replay projection、typed relation。algorithm変更なし |
| `05-simulation/navigation.md` | Derived artifact／Package／Save relation、Capability ID |
| `05-simulation/physics.md` | Save state projector、Backend adoption state、typed relation |
| `06-rendering/camera.md` | `CameraPresentationSummaryV1` Owner宣言、checkpoint reset relation |
| `06-rendering/environment-surfaces.md` | `EnvironmentLightingSummaryV1` Owner、Capability ID正規化 |
| `06-rendering/lighting.md` | shared projection Owner、generic Policy型改名、Budget ref |
| `06-rendering/lod.md` | Artifact manifest／Runtime Package binding、Capability ID規則 |
| `06-rendering/materials.md` | `MaterialReadabilitySummaryV1` Owner、Artifact compatibility |
| `06-rendering/post-processing.md` | shared projection Owner、Package／history compatibility |
| `06-rendering/project-shader.md` | Runtime／Application Package artifact binding、Frontend compatibility |
| `06-rendering/render-graph.md` | `LayerCompositionSummaryV1` Owner、Target Snapshot、Package validator input。[D3D12 Companion]後続ChangeSetでlogical plan Owner、D3D12 consumer boundary、Enhanced-only conformanceを固定 |
| `06-rendering/vfx-authoring.md` | Capability ID正規化、Source migration、Runtime artifact binding |
| `06-rendering/vfx-runtime.md` | Runtime Package、Save／Replay projector、Capability ref |
| `06-rendering/world.md` | World artifact manifest、Runtime Package、Save checkpoint／migration |
| `07-platform/android.md` | common assembly manifest、`UnsignedAndroidPackageV1`、Play draft／track／rollout mapping、Save adapter |
| `07-platform/apple.md` | common assembly manifest、App Store submission／release mapping、Save adapter |
| `07-platform/audio.md` | Save stable identity、Runtime Package／Application Package entry |
| `07-platform/input.md` | Save／Replay header compatibility、Runtime Package binding |
| `07-platform/mobile-common.md` | `TargetProfileRefV1`、Save adapter boundary、application package common owner、suffixなし`DerivedArtifactManifest`参照（§5.5）の`DerivedArtifactManifestV1`統一 |
| `07-platform/ui-text-localization-accessibility.md` | `AccessibilityPolicySnapshotV1` Owner、Save catalog projection、Package receipt |
| `07-platform/windows.md` | `UnsignedWindowsPackageV1`、common Receipt、Store submission mapping、Save storage adapter。[D3D12 Companion]後続ChangeSetでHWND／OS lifecycleとD3D12 Surface／Device ownerを分離 |
| `08-domain-packs/domain-pack-contract.md` | `DomainPackManifestV1`、Engine range、dependency refs、resolved lock、migration |
| `08-domain-packs/shooter.md` | maturity非依存ID、Pack lock、Save／Package／cross-profile fixture |

Architecture Indexと2 Decisionsも更新対象とする。Decisionは歴史的判断を改竄せず、staleな固定件数には追記で現在の生成Indexを正本とすることを明示する。Runtime ECS Decisionは未承認設計なので本文を直接修正できる。

## 20. 更新順序

一つの巨大な未検証編集として扱わず、次の順序で意味を閉じる。

1. Architecture Governanceを追加し、metadata grammar、state、relation、lint規則を確定する。
2. Compatibility／Evolutionを追加し、型、ID、artifact class、migration規則を確定する。
3. Product Plan、Core、Naming、Executable Contracts、Toolchainを新規則へ合わせる。
4. Persistence／Saveを追加し、Gameplay、Project、Scheduling、Debug、World、Platform、UIを接続する。
5. Runtime Packageを追加し、Asset、Native、World、Shader、Save migrationを接続する。
6. Application Package／Releaseを追加し、Governance、Evidence、Windows、Android、Appleを接続する。
7. Shared projection Ownerを各Domainへ登録し、Rendering／Platform consumerを更新する。
8. Domain PackとShooterをCompatibility、Save、Packageへ接続する。
9. Simulation、Rendering、Platformの残りmetadataとcross-referenceを正規化する。
10. Runtime ECS Decisionを新Ownerへrebaseし、active spec追加前Gateを更新する。
11. Index、Decision追記、全体lint、リンク、anchor、DAG、Owner、type、ID、Phase、E2E traceを検証する。

各stepは局所lintに合格してから次へ進む。中間状態を`approved`と表示せず、全体ChangeSet完成後にreviewを依頼する。

D3D12 Companionは上記のstep 12として本ChangeSetへ追加しない。step 11合格後のexact architecture hashから別ChangeSetを開始し、Product Phase 2の実装開始前に完了させる。

## 21. Architecture lint design

実装計画では、既に固定されたNode.js／TypeScript toolchainだけを使うdependency-free validatorを`tools/architecture_lint/`へ追加する。TypeScript compiler programmatic APIには依存せず、CLIでcompileしたvalidatorをNodeで実行する。

Validatorは次を検査する。

1. active spec inventoryとIndex一致。
2. H1直後のexact JSON metadata、document ID、path、state、approval hashの一意性と整合。
3. `requires`の存在、self edge、cycle、redundant transitive edge。Directory番号をdependency layerと推測しない。
4. `integrates_with`のreciprocal edgeとContract ID集合一致。
5. Local Markdown linkとGitHub互換heading anchor。
6. Shared canonical contract Ownerの一意性とconsumer参照。
7. 永続／wire Schema typeの`V<major>` suffix。
8. suffixなしalias、同名別定義、unknown Owner。
9. Capability／Work Package／Operation／Diagnostic ID grammarとorphan ref。
10. Product Phase ID、Runtime phase ID、Target Profile IDのclosed registry参照。
11. Save、Runtime Package、Application Package、Candidate／Approval／Activation、Store publication Receiptの必須E2E edge。
12. Decisionの固定件数を正本Gateとして使用していないこと。
13. Manifest／Envelope／Package／Candidate参照graphのcycle、self hash、後段identityの前段埋込みがないこと。

出力はstable diagnostic ID、document、line、owner、remediationを持ち、同一入力で同一順序とする。CIは生成Indexとの差分とlint errorが一件でもあれば失敗する。

D3D12固有のsymbol／mapping／descriptor／Tool Catalog検査はCompanionの[Architecture lint追加](2026-07-22-ai-readable-d3d12-backend-design.md#26-architecture-lint追加)が所有し、本validatorの13項へ混入させない。本validatorはD3D12正本が追加された後も、文書ID、Ownerの一意性、typed relation、External Evidence refの存在までを共通Gateとして検査する。

## 22. Verification matrix

| Gate | Positive | Negative |
|---|---|---|
| Document graph | 全`requires`をtopological sort | cycle、self edge、missing docを拒否 |
| Owner | 全shared contractが一Owner | 0 Owner、2 Owner、consumer inline再定義を拒否 |
| Schema | V1型とexact revision解決 | suffixなし、unknown major、Field ID再利用を拒否 |
| Compatibility | new readerがold same-major fixtureを読む | migration欠落、multiple path、range外を拒否 |
| Feature evolution | feature未選択の旧Project／Save／Package挙動が不変 | Backend存在、directory scan、散在booleanによる暗黙Activationを拒否 |
| Save | capture→transform→write→read-back→decrypt／load→digest一致 | partial write、auth／hash mismatch、decompression bound超過、orphan Field、cloud conflict自動mergeを拒否 |
| Runtime Package | exact closureをload | ABI、Contract、Content、Shader、migration mismatchで全体拒否 |
| Application Package／Release | subject→manifest→sign→staging read-back→candidate／approval／activation→publish read-back | manifest外file、identity混在、hash差替え、Candidate hash cycle、未承認公開を拒否 |
| Domain Pack | range＋resolved lock＋Qualification | dependency cycle、lock未選択の複数解、未Qualified Targetを拒否 |
| Product | Work Package→Capability→Receipt closure | orphan WP、maturity入りID、state軸混同を拒否 |
| ECS prerequisite | 新Ownerへのexact ref | 旧Save owner、unversioned manifest、固定件数前提を拒否 |
| D3D12 companion boundary | 後続計画へのexact link、一つのD3D12 Backend Owner | 6番目の横断Owner扱い、Render Graph／Windows／UIでのnative contract再定義を拒否 |

## 23. Riskとmitigation

| Risk | Mitigation |
|---|---|
| 新正本追加で文書数が増える | 固定件数を廃止し、単一責務、1000行目安、生成Indexで管理 |
| 横断型のOwner移動で大量renameが起きる | Pre-implementation clean changeとして一ChangeSetでconsumer、fixture、Decisionを更新 |
| Compatibility規則が開発速度を落とす | Rebuild可能物は互換対象外とし、User data／Public surfaceだけを厳格化 |
| Save仕様がECS詳細へ依存する | Persistenceはaggregate、ECSはprojectionに限定し、raw storageを永続化しない |
| Package仕様がPlatform差を消してしまう | 共通assembly envelopeとTarget固有envelopeを分離する |
| lintがMarkdown表現に過度依存する | exact metadataと明示Registry／Owner／E2E表だけを機械入力にし、本文自然言語を推測しない |
| 全43仕様更新中にcurrent branchのECS成果を失う | ECS Decisionを保持し、最後に新Ownerへrebaseする。既存commitを上書きしない |

## 24. Release／Qualificationでのみ決まる値

本設計を進めるための未決定事項はない。次の値は設計上の未決ではなく、実装またはRelease CandidateでReceiptにより解決するruntime evidenceである。

- 初回Engine release versionとRelease date。
- 実測Performance、Target device、Store validation result。
- 各Capabilityが`qualified`または`production`へ昇格する時期。
- 非snapshot AI Modelの将来resolved ID。

これらを仮値で仕様へ固定せず、exact lock、Candidate、Receipt、Approvalで決定する。

## 25. Review checklist

- 5新正本の責務が重複していない。
- Governance、Compatibility、Persistence、Runtime Package、Application PackageのAuthorityが分離されている。
- User dataとrebuild可能artifactのpolicyが分離されている。
- Pre-1.0、Post-1.0、Project、Save、Replay、Public APIのsupport windowが区別されている。
- Strict unknown rejectionとadditive evolutionの方向が矛盾していない。
- SaveとReplay、Runtime PackageとContent Package、Application PackageとPlatform formatが混同されていない。
- Capability、Work Package、Document、Contract、Target、Evidence、Dependencyのstate axisが分離されている。
- Shared projectionの全Ownerが決まっている。
- Domain Pack rangeとexact Qualificationが両方必要になっている。
- Manifest、Package、Candidateのhash graphに自己参照と循環がない。
- Work PackageのPhase、Capabilityのmaturity、Profileのversionがlogical IDに入っていない。
- ECS Decisionが未存在Ownerを参照しない移行順になっている。
- 全43 active specが影響マトリクスに一度ずつ現れる。
- 実装がないものを合格済みと表現していない。

## 26. Registry closure

Product-owned execution registryは[Product Plan §11](../architecture/00-product/product-plan.md#11-product-execution-registries)を正本とする。Control Planeは構造と参照整合を検査するが、Product tier、Phase outcome、Target scope、fallback選択を再定義しない。

必須Registryは次の7件である。

1. `CapabilityRegistryV1`
2. `CapabilityTargetActivationV1`
3. `ProductPhaseRegistryV1`
4. `WorkPackageRegistryV1`
5. `TargetProfileRegistryV1`
6. `RequirementRegistryV1`／`FixtureRegistryV1`
7. `FallbackRegistryV1`

Capability activationの正本keyは`{capability_id,target_id}`である。required Targetの行が一件でも欠ける場合はaggregateを`not_activated`、全行がある場合は`not_activated < candidate_locked < qualified < production`の最小値とする。AggregateをRegistryへ保存、手動設定、Receiptの代用にしない。

Work Packageは`defer_reason`、`reconsideration_gate_refs[]`、`blocked_reason_ref`を常設Fieldとする。`deferred`で理由または再検討Gate欠落、`blocked`でDiagnostic欠落をfail closedにする。Phase、Capability tier、profile versionをlogical IDへ埋め込まない。

## 27. Exact migration authority

移行開始時の43既存＋5新規文書について、旧依存、新direct `requires`、reciprocal `integrates_with`、Contract ID、削除edge分類、初期canonical topological orderは[Control Plane Implementation Plan Appendix A～C](2026-07-22-architecture-evolution-control-plane-implementation-plan.md#appendix-a-legacy-dependency-inventory)を正本とする。移行後の文書件数とcanonical orderは生成`document-relations.v1.json`から導出し、48を恒久invariantにしない。ID移行は同計画Appendix Dを正本とする。

設計本文の影響マトリクスは変更scopeを説明するものであり、migration manifestの代用ではない。実装はAppendixを`architecture/migrations/control-plane-v1.json`と`architecture/registry/*.json`へ転記し、全legacy edgeと全old IDを一度だけ分類する。

## 28. Baseline handoff

Control Plane完了時に`architecture/baselines/control-plane-v1.json`を生成し、次をexactに記録する。

```text
git_tree_id
architecture_index_sha256
document_relation_registry_sha256
product_registry_sha256
identity_migration_registry_sha256
architecture_explain_schema_sha256
toolchain_lock_sha256
architecture_lint_artifact_sha256
lint_version
```

Baselineへ文書件数やedge件数を重複保存しない。Readerはhash照合済み`document-relations.v1.json`の`documents[]`、`canonical_order[]`、`requires[]`、`integrates_with[]`から件数を導出し、配列間不一致をbaseline mismatchとして拒否する。上記Field集合は実装計画Task 10およびbaseline schemaと完全一致させる。

Runtime ECSとD3D12 Backendはこのread-backが成功するまで開始しない。値は設計時に仮記入せず、clean treeで全lint／test／Index Gateを通したArtifactから生成する。Mismatchは`diagnostic.architecture.baseline-mismatch`で停止し、最新値へ暗黙追従しない。

## 29. 実装計画

実装Task、exact file map、TypeScript interface、test、command、expected result、生成inventory DAG（移行開始時は48文書）、29 initial reciprocal integration、ID migrationは[Architecture Evolution Control Plane Implementation Plan](2026-07-22-architecture-evolution-control-plane-implementation-plan.md)に固定した。本文書の承認だけで実装完了またはCapability activationを宣言しない。
