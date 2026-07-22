# Architecture Evolution Control Plane Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 43 active Architecture仕様へ5正本を追加し、48文書をtyped metadata、acyclic direct dependency、closed registry、stable ID、決定論的lint、bounded architecture explainで管理する。

**Architecture:** Markdown本文を意味推測せず、H1直後のexact JSON metadata、checked-in registry JSON、migration manifestだけを機械入力にする。TypeScript 7 compiler APIは使わず、dependency-free TypeScript sourceをCLIでcompileしてNode.jsで実行し、同一Git treeから同一diagnostic順と同一generated Indexを得る。

**Tech Stack:** Node.js 24.18.0 LTS、npm 11.16.0、TypeScript 7.0.2 CLI、`--singleThreaded`、JSON Schema Draft 2020-12、PowerShell 7、Git。

## Global Constraints

- 入力baselineは実行開始時のGit tree hashで固定し、dirty treeでは`diagnostic.architecture.dirty-baseline`を返して停止する。
- `docs/architecture/00-product`～`08-domain-packs`のmetadata valid文書だけをactive specとし、`README.md`、`decisions/`、`superseded/`、`docs/plans/`を除外する。
- Migration開始時の期待inventoryは既存43文書＋新規5文書＝48文書である。移行完了後は固定件数をinvariantにせず生成inventoryを正本とする。
- MetadataはControl Plane Design §7.1のexact JSONだけを受理し、旧Markdown list headerとの併存を禁止する。
- `requires`はdirect prerequisiteだけで、cycle、self edge、missing node、redundant transitive edgeを拒否する。
- `integrates_with`は双方に同じContract ID集合を要求し、片方向、空集合、unknown Contract IDを拒否する。
- Logical IDへmaturity、Phase、schema／profile versionを埋め込まない。旧IDは本計画Appendix Dの一回限りmigrationだけで変換し、runtime aliasを残さない。
- Product stateは`{capability_id,target_id}`単位を正本とし、aggregate stateはrequired Targetの最小stateとして導出する。
- TypeScript 7.0はstable programmatic APIを持たないため、`typescript`をruntime importしない。正式compileは`npx tsc --build --force --singleThreaded`だけを使う。
- 新しいproduction dependencyを追加しない。Markdown、JSON、SHA-256、path、graph処理はNode標準Libraryだけで実装する。
- 実装完了を文書承認、Capability昇格、Release承認へ流用しない。
- 別authority ManifestをSchemaまたは互換aliasとして追加しない。authority discoveryは`ArchitectureMetadataV1`、relation registry、Shared canonical contracts、生成Indexから導出する。

---

## 1. File map

| Path | Responsibility |
|---|---|
| `docs/architecture/01-governance/architecture-governance.md` | Metadata、relation、Owner、lifecycle、lint正本 |
| `docs/architecture/02-foundation/compatibility-evolution.md` | Schema／ID／artifact compatibility、migration正本 |
| `docs/architecture/04-runtime/persistence-save.md` | Save集約、migration、atomic load／publication正本 |
| `docs/architecture/04-runtime/runtime-package.md` | Runtime package manifest、loader、rollback正本 |
| `docs/architecture/07-platform/application-package-release.md` | Target package assembly、sign、staging／publish正本 |
| `schemas/architecture/document-metadata.schema.json` | Architecture metadata V1 schema |
| `schemas/architecture/explain-projection.schema.json` | Architecture explain request／projectionのclosed schema |
| `schemas/architecture/document-relations.schema.json` | document relation registry V1 schema |
| `schemas/architecture/baseline.schema.json` | baseline handoff V1 schema。field過不足を拒否 |
| `schemas/product/product-registries.schema.json` | Product registry V1 schema集合 |
| `architecture/registry/document-relations.v1.json` | 48文書のdirect requires、reciprocal integration、canonical order正本 |
| `architecture/registry/identity-migration.v1.json` | Appendix Dと一致する旧→新ID mapping |
| `architecture/registry/product.v1.json` | Product Plan §11のregistry projection |
| `architecture/migrations/control-plane-v1.json` | legacy header、new metadata、removed edge classificationの監査記録 |
| `tools/architecture_lint/package.json` | offline、private、scriptなしのtool package |
| `tools/architecture_lint/package-lock.json` | `npm ci`が必須とするexact dependency lock |
| `tools/architecture_lint/tsconfig.json` | strict ES module compile設定 |
| `tools/architecture_lint/src/model.ts` | closed data types |
| `tools/architecture_lint/src/parse.ts` | UTF-8、H1、metadata JSON、Markdown link parser |
| `tools/architecture_lint/src/graph.ts` | DAG、transitive reduction、reciprocity検査 |
| `tools/architecture_lint/src/registry.ts` | ID、Owner、Phase、Capability、Target参照検査 |
| `tools/architecture_lint/src/index-generator.ts` | deterministic Architecture Index生成 |
| `tools/architecture_lint/src/explain.ts` | bounded architecture explain parser、generator、canonical encoder、continuation検証 |
| `tools/architecture_lint/src/main.ts` | CLI、diagnostic sort、exit code |
| `tools/architecture_lint/test/*.test.mjs` | Node test runnerによるpositive／negative test |
| `tools/architecture_lint/test/fixtures/explain-invalid/**` | stale／omitted／署名／Evidence／上限の単一原因fixture |
| `tools/architecture_lint/test/fixtures/**` | 1 failureにつき1原因のfixture |

## 2. Public interfaces

```ts
export type DocumentId = string & { readonly __brand: "DocumentId" };
export type ContractId = string & { readonly __brand: "ContractId" };

export interface ArchitectureMetadataV1 {
  readonly schema: "mirakan.architecture-document-metadata/v1";
  readonly document_id: DocumentId;
  readonly state: "draft" | "review" | "approved" | "superseded";
  readonly owner_scope: readonly string[];
  readonly non_owner_scope: readonly string[];
  readonly requires: readonly DocumentId[];
  readonly integrates_with: readonly {
    readonly document_id: DocumentId;
    readonly contract_ids: readonly ContractId[];
  }[];
  readonly supersedes: readonly DocumentId[];
  readonly approval_ref: string | null;
  readonly external_evidence_verified_at: string | null;
}

export interface ArchitectureDiagnosticV1 {
  readonly diagnostic_id: string;
  readonly severity: "error" | "warning";
  readonly document_id: DocumentId | null;
  readonly path: string;
  readonly line: number;
  readonly owner_document_id: DocumentId | null;
  readonly remediation: string;
}

export interface ArchitectureExplainEntryV1 {
  readonly canonical_concept_id: string;
  readonly owner_document_id: DocumentId;
  readonly owner_contract_id: ContractId | null;
  readonly runtime_phase_or_lifetime: string | null;
  readonly source_stable_id: string;
  readonly source_content_sha256: string;
  readonly evidence_refs: readonly string[];
}

export interface ArchitectureExplainDependencyEdgeV1 {
  readonly source_canonical_concept_id: string;
  readonly target_canonical_concept_id: string;
  readonly relation_contract_id: ContractId;
  readonly owner_document_id: DocumentId;
  readonly source_stable_ids: readonly string[];
  readonly source_content_sha256: string;
  readonly evidence_refs: readonly string[];
}

export interface ArchitectureExplainRequestV1 {
  readonly project_revision: string;
  readonly scope: string;
  readonly field_mask: readonly string[];
  readonly target_profile_ref: string | null;
  readonly continuation: string | null;
}

export interface ArchitectureExplainSourceV1 {
  readonly project_id: string;
  readonly project_revision: string;
  readonly contract_set_hash: string;
  readonly architecture_metadata: readonly ArchitectureMetadataV1[];
  readonly document_relation_registry_sha256: string;
  readonly product_registry_sha256: string;
  readonly contract_registry_sha256: string;
  readonly world_source_revision_sha256: string;
  readonly target_source_revision_sha256: string;
  readonly source_entries: readonly ArchitectureExplainEntryV1[];
  readonly source_dependency_edges: readonly ArchitectureExplainDependencyEdgeV1[];
}

export interface ArchitectureExplainProjectionV1 {
  readonly project_id: string;
  readonly project_revision: string;
  readonly contract_set_hash: string;
  readonly scope: string;
  readonly game_system_entries: readonly ArchitectureExplainEntryV1[];
  readonly state_owner_entries: readonly ArchitectureExplainEntryV1[];
  readonly dependency_edges: readonly ArchitectureExplainDependencyEdgeV1[];
  readonly runtime_phase_entries: readonly ArchitectureExplainEntryV1[];
  readonly world_entries: readonly ArchitectureExplainEntryV1[];
  readonly level_entries: readonly ArchitectureExplainEntryV1[];
  readonly streaming_entries: readonly ArchitectureExplainEntryV1[];
  readonly capability_entries: readonly ArchitectureExplainEntryV1[];
  readonly target_entries: readonly ArchitectureExplainEntryV1[];
  readonly save_replay_entries: readonly ArchitectureExplainEntryV1[];
  readonly evidence_refs: readonly string[];
  readonly omitted_ranges: readonly string[];
  readonly continuation: string | null;
}

export function parseArchitectureDocument(path: string, bytes: Uint8Array): ArchitectureMetadataV1;
export function validateDirectDag(nodes: readonly ArchitectureMetadataV1[]): readonly ArchitectureDiagnosticV1[];
export function validateReciprocalIntegrations(nodes: readonly ArchitectureMetadataV1[]): readonly ArchitectureDiagnosticV1[];
export function validateProductRegistries(input: unknown): readonly ArchitectureDiagnosticV1[];
export function generateArchitectureIndex(nodes: readonly ArchitectureMetadataV1[]): string;
export function parseArchitectureExplainRequest(input: unknown): ArchitectureExplainRequestV1;
export function generateArchitectureExplainProjection(request: ArchitectureExplainRequestV1, source: ArchitectureExplainSourceV1): ArchitectureExplainProjectionV1;
export function encodeArchitectureExplainProjection(projection: ArchitectureExplainProjectionV1): Uint8Array;
```

Diagnosticsは`{diagnostic_id,path,line,document_id}`のUTF-8 byte順でsortし、locale、filesystem enumeration、wall clockを使わない。Errorが1件以上ならexit 1、引数／I/O failureはexit 2、cleanはexit 0である。

## 3. Tasks

### Task 1: Baseline inventoryとmigration manifestを固定する

**Files:**
- Create: `architecture/migrations/control-plane-v1.json`
- Test: `tools/architecture_lint/test/migration-manifest.test.mjs`

**Interfaces:**
- Consumes: Git tracked tree、Appendix A～D。
- Produces: `ControlPlaneMigrationV1`。各rowは`path`、`legacy_document_id`、`legacy_dependencies[]`、`new_document_id`、`new_requires[]`、`integration_edge_refs[]`、`replaced_by_integration_edges[]`、`removed_transitive_edges[]`、`removed_reference_only_edges[]`を持つ。

- [ ] **Step 1: failing testを書く**

```js
assert.equal(manifest.rows.length, 48);
assert.equal(new Set(manifest.rows.map(x => x.new_document_id)).size, 48);
assert.deepEqual(manifest.rows.map(x => x.new_document_id), appendixBOrder);
```

- [ ] **Step 2: testを実行し、manifest未存在で失敗することを確認する**

Run: `node --test tools/architecture_lint/test/migration-manifest.test.mjs`

Expected: exit 1、`ENOENT: architecture/migrations/control-plane-v1.json`。

- [ ] **Step 3: Appendix A～DをそのままJSONへ転記する**

`legacy_dependencies[]`はAppendix A、`new_requires[]`はAppendix B、`integration_edge_refs[]`はAppendix Cから生成する。`integration_edge_refs[]`はlegacy edge由来か否かを区別しない。Legacy edgeはAppendix Bの分類式で一意に分類し、`removed_transitive`は`removed_transitive_edges[]`、`replaced_by_integrates_with`（例: PP→SEC）は`replaced_by_integration_edges[]`、`removed_reference_only`は`removed_reference_only_edges[]`へ置く。同じedgeを二分類または無分類にしない。

- [ ] **Step 4: testを再実行する**

Expected: 48 row、43 legacy＋5 `legacy_dependencies=[]`、duplicate 0、unclassified legacy edge 0でPASS。

- [ ] **Step 5: review checkpointを作る**

Run: `git diff --check -- architecture/migrations/control-plane-v1.json tools/architecture_lint/test/migration-manifest.test.mjs`

Expected: outputなし、exit 0。

### Task 2: Metadata／Product registry schemaをtest-firstで追加する

**Files:**
- Create: `schemas/architecture/document-metadata.schema.json`
- Create: `schemas/product/product-registries.schema.json`
- Create: `tools/architecture_lint/test/schema.test.mjs`

**Interfaces:**
- Consumes: Control Plane Design §7、Product Plan §11。
- Produces: unknown property、duplicate logical ID、invalid closed stateを拒否するV1 schema。

- [ ] **Step 1: valid最小fixtureとinvalid fixtureをtestへ記述する**

```js
assert.equal(validateMetadata(validReviewDocument).length, 0);
assert.match(validateMetadata({...validReviewDocument, state: "active"})[0].diagnostic_id,
  /^diagnostic\.architecture\.metadata-invalid$/);
assert.match(validateWorkPackage({...validWp, scheduling_state: "deferred", defer_reason: null})[0].diagnostic_id,
  /^diagnostic\.product\.deferred-reason-missing$/);
```

- [ ] **Step 2: schema未存在のfailureを確認する**

Run: `node --test tools/architecture_lint/test/schema.test.mjs`

Expected: exit 1、schema file `ENOENT`。

- [ ] **Step 3: exact schemaを追加する**

Metadataは`additionalProperties=false`、required 10 key、array duplicate禁止、state closed enumを固定する。Work Packageは`defer_reason`、`reconsideration_gate_refs`、`blocked_reason_ref`を常にrequiredとし、state別`if/then`でnull／non-nullを制約する。ID patternは設計§9.1の2 regex（document ID用と一般logical ID用）を転記し、Appendix DとProduct Plan §11の全新IDが一般logical ID regexに一致するpositive testを加える。

- [ ] **Step 4: testを再実行する**

Expected: valid fixture PASS、unknown key、invalid state、deferred理由欠落、blocked reason欠落の4 negative fixture PASS。

### Task 3: 5つの新Owner正本を追加する

**Files:**
- Create: `docs/architecture/01-governance/architecture-governance.md`
- Create: `docs/architecture/02-foundation/compatibility-evolution.md`
- Create: `docs/architecture/04-runtime/persistence-save.md`
- Create: `docs/architecture/04-runtime/runtime-package.md`
- Create: `docs/architecture/07-platform/application-package-release.md`

**Interfaces:**
- Consumes: Control Plane Design §6、§8、§10～13。
- Produces: Appendix BのIDとrequires、Appendix Cのreciprocal integrationを持つ5 metadata node。

- [ ] **Step 1: 5文書欠落を検査するtestを書く**

```js
for (const path of fiveOwnerPaths) assert.equal(existsSync(path), true, path);
```

- [ ] **Step 2: missing file failureを確認する**

Expected: 5 pathを列挙してFAIL。

- [ ] **Step 3: 各正本を作成する**

各文書は`state=review`、`approval_ref=null`とし、Owner scope、non-owner scope、Shared canonical contracts、failure、qualification、official evidenceを持つ。型はControl Plane Design §13.1から移し、consumerへ複写しない。

- [ ] **Step 4: metadata testを実行する**

Expected: 5文書のschema、ID、requires、integration reciprocityがPASS。

### Task 4: 43文書をexact JSON metadataへ一括移行する

**Files:**
- Modify: Appendix Aに列挙した43 active spec
- Test: `tools/architecture_lint/test/document-inventory.test.mjs`

**Interfaces:**
- Consumes: `ControlPlaneMigrationV1`。
- Produces: 旧`文書ID／状態／正本範囲／非正本範囲／依存`listが0件の48 document set。

- [ ] **Step 1: dual metadataを拒否するtestを書く**

```js
assert.equal(activeDocs.filter(x => x.hasLegacyHeader).length, 0);
assert.equal(activeDocs.filter(x => x.metadataBlockCount !== 1).length, 0);
```

- [ ] **Step 2: 現状43 failureを確認する**

Expected: legacy header 43、new metadata 0でFAIL。

- [ ] **Step 3: 各文書をAppendix B／Cどおり移行する**

H1直後に一つのmetadata blockを置き、旧5 list行を同じpatchで削除する。本文の説明Linkは残すがmetadata `requires`へ再昇格させない。

- [ ] **Step 4: inventory testを実行する**

Expected: active 48、legacy header 0、metadata block 48、unknown ID 0でPASS。

### Task 4B: 43文書本文のold IDと旧型名を置換する

**Files:**
- Modify: Appendix Aに列挙した43 active spec
- Test: `tools/architecture_lint/test/identity-occurrence.test.mjs`

**Interfaces:**
- Consumes: Appendix D、Control Plane Design §13.2、§19。
- Produces: 本文中のAppendix D old ID出現0、§13.2旧型名出現0の48 document set。

- [ ] **Step 1: old ID出現を数えるfailing testを書く**

```js
assert.deepEqual(scanActiveDocs(appendixDOldIds), []);
```

- [ ] **Step 2: 現状failureを確認する**

Expected: `windows_desktop_v1`（`07-platform/windows.md`、`04-runtime/performance-capacity.md`、`02-foundation/toolchain-dependencies.md`）を含むold ID残存でFAIL。

- [ ] **Step 3: 設計§19の文書別必須変更に従い本文を置換する**

Appendix Dのold IDを新stable IDへ、`TargetProfileRef`等の§13.2改名型とsuffixless型を新型名へ、C2 Matrixの`lifecycle_state`参照を`capability_activation_state`と`owner_work_package_ref`参照へ置換する。Task 4のmetadata blockは変更しない。

- [ ] **Step 4: testを再実行する**

Expected: old ID出現0、旧型名出現0でPASS。Completion Gateの「Appendix Dの全old ID出現数が0」は本Taskで到達する。

### Task 5: Parserとstable diagnosticを実装する

**Files:**
- Create: `tools/architecture_lint/package.json`
- Create: `tools/architecture_lint/package-lock.json`
- Create: `tools/architecture_lint/tsconfig.json`
- Create: `tools/architecture_lint/src/model.ts`
- Create: `tools/architecture_lint/src/parse.ts`
- Create: `tools/architecture_lint/test/parse.test.mjs`

**Interfaces:**
- Produces: `parseArchitectureDocument(path, bytes)`。

- [ ] **Step 1: BOM、invalid UTF-8、duplicate metadata、trailing comma、legacy headerのnegative testsを書く**
- [ ] **Step 2: toolchain lock照合済みtarballでnpm cacheを事前充填し、`npm ci --ignore-scripts --offline --no-audit --no-fund`とtestを実行しfailureを確認する**
- [ ] **Step 3: Node標準`TextDecoder("utf-8", {fatal:true})`、line scanner、`JSON.parse`でminimal parserを実装する**
- [ ] **Step 4: `npx tsc --build --force --singleThreaded`を実行する**

Expected: `typescript` runtime import 0、compile error 0。

- [ ] **Step 5: parse testsを実行する**

Expected: positive 1、negative 5がPASS。

### Task 6: DAG、transitive reduction、reciprocal integrationを実装する

**Files:**
- Create: `architecture/registry/document-relations.v1.json`
- Create: `schemas/architecture/document-relations.schema.json`
- Create: `tools/architecture_lint/src/graph.ts`
- Create: `tools/architecture_lint/test/graph.test.mjs`

**Interfaces:**
- Consumes: 48 `ArchitectureMetadataV1`、Appendix B～C。
- Produces: document relation registry、canonical order検証、cycle witness、redundant edge witness、reciprocity diagnostic。

- [ ] **Step 1: cycle、self、missing、redundant、one-way integration fixtureを書く**
- [ ] **Step 2: test failureを確認する**
- [ ] **Step 3: Kahn sortとedge除外DFSを実装する**

Redundant edge `{a,b}`は、そのedgeだけを除いて`a`から`b`へ到達可能な場合に限り報告する。Cycle diagnosticはbyte順最小nodeからDFSし、最初のback-edgeまでのclosed pathを出す。

- [ ] **Step 4: Appendix B graphを検査する**

Expected: nodes 48、edges 76、cycle 0、self 0、missing 0、redundant 0、Appendix Bのcanonical orderが有効なtopological order（各文書の全`requires`先が順序上より前に並ぶ）であること。lintは順序を独自導出せず、同順位のtie-break規則を持たない。

- [ ] **Step 5: document relation registryを転記して検査する**

Appendix Bの`requires`とcanonical order、Appendix Cのreciprocal integrationを`architecture/registry/document-relations.v1.json`へbyte順JSONで転記する。canonical orderは導出物ではなくregistry格納値である。48文書metadataの`requires`／`integrates_with`とregistryの完全一致（過不足0）を検査する。

Expected: registryがschema valid、metadataとの一致、mismatch fixtureがexact diagnosticで失敗。

### Task 7: Product registryとID migrationを実装する

**Files:**
- Create: `architecture/registry/product.v1.json`
- Create: `architecture/registry/identity-migration.v1.json`
- Create: `tools/architecture_lint/src/registry.ts`
- Create: `tools/architecture_lint/test/registry.test.mjs`

**Interfaces:**
- Consumes: Product Plan §11、Appendix D。
- Produces: orphan 0のCapability／Target／Phase／Work Package／Requirement／Fixture／Fallback graph。

- [ ] **Step 1: maturity入りID、missing Target activation、deferred理由欠落、orphan fixtureのnegative testsを書く**
- [ ] **Step 2: failureを確認する**
- [ ] **Step 3: Product Plan §11をbyte順JSONへ転記し、validatorを実装する**
- [ ] **Step 4: aggregate activation testを書く**

```js
assert.equal(aggregate([{scope:"required",state:"production"},{scope:"required",state:"qualified"}]), "qualified");
assert.equal(aggregate([{scope:"required",state:"production"}, null]), "not_activated");
```

- [ ] **Step 5: registry testを実行する**

Expected: Capability／feature 36（既存33 IDの移行＋新規3D Product Capability 1＋Project Source Capability 2）、Requirement 16、Work Package 25、Phase 10、Target 5、orphan 0、maturity-bearing current ID 0、initial aggregateはすべて`not_activated`。

### Task 8: deterministic Index generatorとCLIを実装する

**Files:**
- Create: `tools/architecture_lint/src/index-generator.ts`
- Create: `tools/architecture_lint/src/main.ts`
- Create: `tools/architecture_lint/test/index-generator.test.mjs`
- Modify: `docs/architecture/README.md`

**Interfaces:**
- Produces: `node dist/main.js check`、`node dist/main.js generate-index --check`。

- [ ] **Step 1: shuffled inputから同一bytesを要求するtestを書く**
- [ ] **Step 2: failureを確認する**
- [ ] **Step 3: path layer、document ID byte順でgeneratorを実装する**
- [ ] **Step 4: Indexを生成し、直後の`--check`がdiff 0になることを確認する**

Expected: 2回生成のSHA-256一致、wall-clock文字列0、固定active件数の規範文0。

### Task 8A: bounded architecture explainを実装する

**Files:**
- Create: `schemas/architecture/explain-projection.schema.json`
- Create: `tools/architecture_lint/src/explain.ts`
- Create: `tools/architecture_lint/test/explain.test.mjs`
- Create: `tools/architecture_lint/test/fixtures/explain-invalid/**`
- Modify: `tools/architecture_lint/src/main.ts`

**Interfaces:**
- Consumes: exact Project revision、`ArchitectureMetadataV1` set、document relation registry、Product registry、Contract registry、World／Target Source revision、`ArchitectureExplainRequestV1`。
- Produces: `ArchitectureExplainProjectionV1` canonical bytes、`explain-architecture` CLI、署名付きcontinuation、stable diagnostic。

**Dependencies and ownership:** Tasks 4、6、7、8のmetadata／graph／registry／Indexが完了してから実行する。`ArchitectureComprehensionCaseV1`／`ArchitectureComprehensionFixtureV1`はAI Verification／Provenance Ownerが定義し、本Taskはその入力となるexact projection bytesとhashだけを供給する。

- [ ] **Step 1: schema／parserのfailing testsを書く**

Valid requestに加え、unknown key、空`field_mask`、stale Project revision、category 257 entry、dependency 1,025 edge、Evidence 0件、128超のomitted range、2 MiB超のcanonical encoding、別scope／revisionへ再利用したcontinuation、署名不一致を一原因ずつfixture化する。

- [ ] **Step 2: module未存在でfailureを確認する**

Run: `node --test tools/architecture_lint/test/explain.test.mjs`

Expected: `ERR_MODULE_NOT_FOUND`でexit 1。

- [ ] **Step 3: parser、generator、canonical encoderを実装する**

metadata、relation、registry、Contract、World、Target、Source revisionがrequest revisionと一致する場合だけ生成する。各entryをcanonical concept ID、Owner document、Owner Contract、phase／lifetime、Source StableId、Source content SHA-256、Evidence refで閉じ、各category 256、dependency 1,024、全体2 MiBを上限とする。同順位はUTF-8 byte順でsortし、filesystem列挙順、locale、wall clock、説明文を入力にしない。

- [ ] **Step 4: omissionとcontinuationを実装する**

上限超過は要約へ置換せずexact `omitted_ranges`を返す。Continuation payloadへProject revision、scope、field mask hash、Target Profile ref、Source closure hash、次offsetを含め、repository-owned signing key profileで署名する。別条件への再利用、署名不一致、Source closure driftは`diagnostic.architecture.explain-continuation-invalid`で拒否する。

- [ ] **Step 5: CLI queryを追加する**

```powershell
node tools/architecture_lint/dist/main.js explain-architecture --request request.json --source source.json --output projection.json
```

出力はcanonical UTF-8 bytesだけとし、diagnosticはstderr、validation failureはexit 1、I/O failureはexit 2とする。CLIはProject、Owner、Approval、MCD、ChangeSetを変更しない。

- [ ] **Step 6: deterministic-byteとnegative fixtureを閉じる**

Expected: shuffled input 100回のSHA-256が一致し、stale revision、omitted Evidenceの有効扱い、unsigned continuation、summary由来Owner、上限超過の正常完了がすべてexact diagnosticで失敗する。

### Task 9: CI Gateと全negative fixtureを閉じる

**Files:**
- Create: `tools/architecture_lint/test/fixtures/metadata-invalid/**`
- Create: `tools/architecture_lint/test/fixtures/graph-invalid/**`
- Create: `tools/architecture_lint/test/fixtures/registry-invalid/**`
- Create: `.github/workflows/architecture-lint.yml`
- Modify: `docs/architecture/02-foundation/toolchain-dependencies.md`

**Interfaces:**
- Produces: clean checkoutでoffline install→compile→test→lint→Index checkの一方向job。

- [ ] **Step 1: 既存13 Control Plane lint ruleにarchitecture explainのrevision、Owner、Evidence、category bound、edge bound、byte bound、omission、continuation、determinismの9条件を加え、各positive 1／negative 1を列挙するtest matrixを書く**
- [ ] **Step 2: 各negative fixtureがexact diagnostic IDを一件だけ返すことを確認する**
- [ ] **Step 3: CIをNode 24.18.0、npm 11.16.0、TypeScript 7.0.2 lockへ固定する**

`toolchain-dependencies.md`の公式JavaScript toolchain利用rootへ`tools/architecture_lint/`を追加し、同文書が要求する`private=true`、ES module、exact `engines`、`packageManager=npm@11.16.0`、lockfile SHA-256規則を`tools/architecture_lint/package.json`へ適用する。CI workflowはtoolchain lockが固定するtarball（version、URL、size、SHA-256）を取得し、SHA-256照合後にnpm content-addressed cacheへ事前充填してから`npm ci --offline`を実行する。照合失敗はjob失敗とする。
- [ ] **Step 4: local equivalentを実行する**

```powershell
npm ci --prefix tools/architecture_lint --ignore-scripts --offline --no-audit --no-fund
npx --prefix tools/architecture_lint tsc --build --force --singleThreaded
node --test tools/architecture_lint/test/*.test.mjs
node tools/architecture_lint/dist/main.js check
node tools/architecture_lint/dist/main.js generate-index --check
```

Expected: 全command exit 0、stderr 0、diagnostic error 0、generated diff 0。`explain.test.mjs`を含み、architecture explain negative fixtureの未実行0。

### Task 10: Baseline handoff Receiptを作る

**Files:**
- Create: `architecture/baselines/control-plane-v1.json`
- Create: `schemas/architecture/baseline.schema.json`
- Modify: `docs/architecture/decisions/2026-07-22-runtime-ecs-contract.md`
- Modify: `docs/plans/2026-07-22-ai-readable-d3d12-backend-design.md`

**Interfaces:**
- Produces: `git_tree_sha256`ではなくGit object formatに従う`git_tree_id`、`architecture_index_sha256`、`document_relation_registry_sha256`、`product_registry_sha256`、`identity_migration_registry_sha256`、`architecture_explain_schema_sha256`、`toolchain_lock_sha256`、`architecture_lint_artifact_sha256`、`lint_version`を持つexact handoff。field集合は設計§28と一致し、`schemas/architecture/baseline.schema.json`が過不足を拒否する。

- [ ] **Step 1: dirty tree拒否、hash mismatch、baseline field過不足のtestを書く**
- [ ] **Step 2: clean treeで全Gateを再実行する**
- [ ] **Step 3: baseline JSONを生成し、ECS／D3D12計画へexact refを記録する**
- [ ] **Step 4: baseline read-backを実行する**

Expected: 全hash一致。ECS、D3D12、またはarchitecture comprehension Eval開始時に一つでも不一致なら`diagnostic.architecture.baseline-mismatch`で停止する。

## Appendix A: Legacy dependency inventory

この表は移行開始前43 active specのH1 headerを2026-07-22にread-backした結果である。`D`は`decision:2026-07-21-document-system-restructure`を表し、active document nodeではない。省略記号、wildcard、`ほか`を使用しない。

| Alias | Stable document ID |
|---|---|
| AG | `mirakan.arch.architecture-governance` |
| PP | `mirakan.arch.product-plan` |
| SEC | `mirakan.arch.ai-security-approval` |
| EVD | `mirakan.arch.ai-verification-provenance` |
| COMPAT | `mirakan.arch.compatibility-evolution` |
| CORE | `mirakan.arch.core-architecture` |
| CPP | `mirakan.arch.cpp23-modules` |
| MCD | `mirakan.arch.executable-contracts` |
| MATH | `mirakan.arch.math-core` |
| MEM | `mirakan.arch.memory-pointers` |
| NAME | `mirakan.arch.naming-project-layout` |
| TOOL | `mirakan.arch.toolchain-dependencies` |
| ASSET | `mirakan.arch.asset-lifecycle` |
| EUI | `mirakan.arch.editor-ui-framework` |
| WUX | `mirakan.arch.editor-workspace-ux` |
| GPM | `mirakan.arch.gameplay-programming-model` |
| NGM | `mirakan.arch.native-game-module` |
| PSTATE | `mirakan.arch.project-state` |
| DBG | `mirakan.arch.runtime-debugging-observability-replay` |
| PERF | `mirakan.arch.runtime-performance-capacity` |
| SCHED | `mirakan.arch.runtime-scheduling-lifetime` |
| SAVE | `mirakan.arch.runtime-persistence-save` |
| RPKG | `mirakan.arch.runtime-package` |
| ANIM | `mirakan.arch.simulation-animation` |
| COLL | `mirakan.arch.simulation-collision` |
| NAV | `mirakan.arch.simulation-navigation` |
| PHYS | `mirakan.arch.simulation-physics` |
| CAM | `mirakan.arch.rendering-camera` |
| ENV | `mirakan.arch.rendering-environment-surfaces` |
| LIGHT | `mirakan.arch.rendering-lighting` |
| LOD | `mirakan.arch.rendering-lod` |
| MAT | `mirakan.arch.rendering-materials` |
| POST | `mirakan.arch.rendering-post-processing` |
| SHADER | `mirakan.arch.rendering-project-shader` |
| RG | `mirakan.arch.rendering-render-graph` |
| VFXA | `mirakan.arch.rendering-vfx-authoring` |
| VFXR | `mirakan.arch.rendering-vfx-runtime` |
| WORLD | `mirakan.arch.rendering-world` |
| APPKG | `mirakan.arch.platform-application-package-release` |
| AND | `mirakan.arch.platform-android` |
| APPLE | `mirakan.arch.platform-apple` |
| AUDIO | `mirakan.arch.platform-audio` |
| INPUT | `mirakan.arch.platform-input` |
| MOBILE | `mirakan.arch.platform-mobile-common` |
| UI | `mirakan.arch.platform-ui-text-localization-accessibility` |
| WIN | `mirakan.arch.platform-windows` |
| DP | `mirakan.arch.domain-pack-contract` |
| SHOOT | `mirakan.arch.domain-pack-shooter` |

| Path | ID | Legacy dependencies in source order |
|---|---|---|
| `00-product/product-plan.md` | PP | D, SEC, EVD |
| `01-governance/ai-security-approval.md` | SEC | PP, EVD, MCD, PSTATE, NGM, SHADER |
| `01-governance/ai-verification-provenance.md` | EVD | SEC, PP, MCD, TOOL, PERF, DBG, SHADER |
| `02-foundation/core-architecture.md` | CORE | D, PP, SEC, EVD, TOOL, MCD, NAME, CPP, MATH, MEM, SCHED, PERF, DBG |
| `02-foundation/cpp23-modules.md` | CPP | D, PP, CORE, TOOL, MCD, NAME, MEM |
| `02-foundation/executable-contracts.md` | MCD | D, PP, SEC, EVD, CORE, TOOL, NAME, CPP, MATH, PSTATE, SHADER |
| `02-foundation/math-core.md` | MATH | D, PP, CORE, TOOL, MCD, NAME, MEM |
| `02-foundation/memory-pointers.md` | MEM | D, CORE, TOOL, MCD, NAME, MATH |
| `02-foundation/naming-project-layout.md` | NAME | D, CORE, TOOL, MCD, CPP, PSTATE, SEC |
| `02-foundation/toolchain-dependencies.md` | TOOL | D, PP, SEC, EVD, CORE, MCD, CPP, SHADER |
| `03-authoring/asset-lifecycle.md` | ASSET | D, PP, SEC, EVD, CORE, TOOL, MCD, NAME, PSTATE, WUX |
| `03-authoring/editor-ui-framework.md` | EUI | D, PP, CORE, TOOL, MCD, CPP, MEM, PSTATE, WUX |
| `03-authoring/editor-workspace-ux.md` | WUX | D, PP, SEC, EVD, CORE, NAME, PSTATE, ASSET, EUI, GPM |
| `03-authoring/gameplay-programming-model.md` | GPM | D, PP, SEC, EVD, CORE, TOOL, MCD, NAME, CPP, PSTATE, WUX, NGM |
| `03-authoring/native-game-module.md` | NGM | D, SEC, EVD, CORE, TOOL, MCD, NAME, CPP, MEM, SCHED, PSTATE, GPM |
| `03-authoring/project-state.md` | PSTATE | D, PP, SEC, EVD, CORE, MCD, NAME, ASSET, EUI, WUX, GPM, NGM |
| `04-runtime/debugging-observability-replay.md` | DBG | D, PP, SEC, EVD, CORE, TOOL, MCD, NAME, MEM, PSTATE, ASSET, EUI, WUX, GPM, SCHED, PERF, PHYS, COLL, NAV, ANIM, RG, WORLD, VFXR, ENV, CAM, INPUT, UI, AUDIO |
| `04-runtime/performance-capacity.md` | PERF | D, PP, SEC, EVD, CORE, TOOL, MCD, MATH, MEM, PSTATE, ASSET, GPM, SCHED, DBG, WORLD, LOD |
| `04-runtime/scheduling-lifetime.md` | SCHED | D, PP, SEC, EVD, CORE, TOOL, MCD, MEM, PSTATE, ASSET, GPM, NGM, PERF, DBG, PHYS, NAV, ANIM, RG, WORLD, LOD |
| `05-simulation/animation.md` | ANIM | D, SEC, EVD, TOOL, MCD, ASSET, PSTATE, SCHED, PERF, DBG, COLL, PHYS, NAV, LOD, WORLD |
| `05-simulation/collision.md` | COLL | D, SEC, EVD, TOOL, MCD, MATH, ASSET, GPM, SCHED, PERF, PHYS |
| `05-simulation/navigation.md` | NAV | D, SEC, EVD, TOOL, MCD, ASSET, PSTATE, SCHED, PERF, COLL, PHYS, WORLD |
| `05-simulation/physics.md` | PHYS | D, SEC, EVD, TOOL, MCD, PSTATE, GPM, SCHED, PERF, DBG, COLL, NAV, ANIM |
| `06-rendering/camera.md` | CAM | D, PP, SEC, EVD, MCD, MATH, PSTATE, SCHED, PERF, PHYS, RG, POST |
| `06-rendering/environment-surfaces.md` | ENV | D, PP, SEC, EVD, MCD, MATH, ASSET, PSTATE, SCHED, PERF, COLL, PHYS, RG, MAT, LIGHT, VFXA, VFXR, LOD, WORLD |
| `06-rendering/lighting.md` | LIGHT | D, PP, SEC, EVD, TOOL, MCD, MATH, ASSET, PSTATE, PERF, RG, MAT, POST, WORLD |
| `06-rendering/lod.md` | LOD | D, PP, SEC, EVD, MCD, MATH, ASSET, PSTATE, SCHED, PERF, DBG, ANIM, PHYS, NAV, RG, MAT, WORLD |
| `06-rendering/materials.md` | MAT | D, PP, SEC, EVD, TOOL, MCD, ASSET, PSTATE, PERF, RG, SHADER, LIGHT, POST, LOD |
| `06-rendering/post-processing.md` | POST | D, PP, SEC, EVD, MCD, ASSET, PSTATE, PERF, RG, MAT, SHADER, LIGHT |
| `06-rendering/project-shader.md` | SHADER | PP, SEC, EVD, TOOL, MCD, MATH, ASSET, PSTATE, PERF, DBG, RG, MAT, LIGHT, POST, VFXA |
| `06-rendering/render-graph.md` | RG | D, PP, SEC, EVD, CORE, TOOL, MCD, MEM, ASSET, PSTATE, EUI, SCHED, PERF, DBG, ANIM, MAT, SHADER, LIGHT, POST, LOD, WORLD |
| `06-rendering/vfx-authoring.md` | VFXA | D, PP, SEC, EVD, MCD, ASSET, PSTATE, PERF, RG, MAT, SHADER, LOD, VFXR, ENV |
| `06-rendering/vfx-runtime.md` | VFXR | D, PP, EVD, ASSET, SCHED, PERF, DBG, COLL, PHYS, RG, MAT, LOD, VFXA, ENV |
| `06-rendering/world.md` | WORLD | D, PP, SEC, EVD, TOOL, MCD, MATH, ASSET, PSTATE, GPM, SCHED, PERF, DBG, COLL, PHYS, NAV, ANIM, RG, LOD |
| `07-platform/android.md` | AND | D, SEC, EVD, CORE, TOOL, MCD, ASSET, NGM, SCHED, PERF, DBG, RG, SHADER, MOBILE, INPUT, AUDIO, UI |
| `07-platform/apple.md` | APPLE | D, SEC, EVD, CORE, TOOL, MCD, ASSET, NGM, SCHED, PERF, DBG, RG, SHADER, MOBILE, INPUT, AUDIO, UI |
| `07-platform/audio.md` | AUDIO | D, PP, SEC, EVD, TOOL, MCD, ASSET, PSTATE, SCHED, PERF, DBG, WIN, MOBILE, AND, APPLE, UI |
| `07-platform/input.md` | INPUT | D, SEC, EVD, TOOL, MCD, EUI, WUX, SCHED, PERF, DBG, WIN, MOBILE, AND, APPLE, UI |
| `07-platform/mobile-common.md` | MOBILE | D, PP, SEC, EVD, CORE, TOOL, MCD, MEM, ASSET, GPM, NGM, SCHED, PERF, DBG, RG, INPUT, AUDIO, UI, AND, APPLE |
| `07-platform/ui-text-localization-accessibility.md` | UI | D, PP, SEC, EVD, TOOL, MCD, ASSET, PSTATE, EUI, WUX, NGM, SCHED, PERF, RG, WIN, MOBILE, AND, APPLE, INPUT |
| `07-platform/windows.md` | WIN | D, SEC, EVD, CORE, TOOL, NAME, CPP, ASSET, EUI, WUX, NGM, SCHED, PERF, DBG, RG, SHADER, INPUT, AUDIO, UI, MOBILE |
| `08-domain-packs/domain-pack-contract.md` | DP | PP, SEC, EVD, MCD, PSTATE, ASSET, GPM, NGM, PERF |
| `08-domain-packs/shooter.md` | SHOOT | PP, MCD, ASSET, GPM, SCHED, PERF, DBG, COLL, PHYS, CAM, VFXR, INPUT, AUDIO, UI, DP |

新規AG、COMPAT、SAVE、RPKG、APPKGの`legacy_dependencies[]`は空配列であり、legacy documentが存在したという記録を作らない。

## Appendix B: Final direct `requires` DAG

この順序は`document-relations.v1.json`へ格納するcanonical orderであり、`requires` DAGの有効なtopological orderの一つである。順序は分類やsortの導出物ではなくregistry格納値であり、同順位のtie-break規則を定義しない。lintは「各文書の全`requires`先が順序上より前に並ぶこと」だけを検査する。各`requires`配列はdocument IDのUTF-8 byte順で保存する。表は読みやすさのためAliasを使うが、JSONへAliasを保存しない。

| Order | Alias | Direct requires |
|---:|---|---|
| 1 | AG | `[]` |
| 2 | PP | AG |
| 3 | COMPAT | AG |
| 4 | SEC | PP |
| 5 | EVD | SEC |
| 6 | CORE | PP |
| 7 | TOOL | CORE, EVD |
| 8 | MCD | COMPAT, TOOL |
| 9 | NAME | MCD |
| 10 | MATH | NAME |
| 11 | MEM | MATH |
| 12 | CPP | MEM |
| 13 | PSTATE | MCD |
| 14 | ASSET | NAME, PSTATE |
| 15 | EUI | CPP, PSTATE |
| 16 | GPM | CPP, PSTATE |
| 17 | WUX | ASSET, EUI, GPM |
| 18 | NGM | GPM |
| 19 | SCHED | NGM |
| 20 | PERF | SCHED |
| 21 | DBG | PERF |
| 22 | SAVE | DBG |
| 23 | RPKG | ASSET, SAVE |
| 24 | COLL | ASSET, SCHED |
| 25 | PHYS | COLL |
| 26 | NAV | PHYS |
| 27 | ANIM | PHYS |
| 28 | RG | ASSET, PERF |
| 29 | SHADER | RG |
| 30 | MAT | SHADER |
| 31 | POST | MAT |
| 32 | LIGHT | POST |
| 33 | WORLD | ANIM, NAV, RG |
| 34 | LOD | MAT, WORLD |
| 35 | VFXR | LOD |
| 36 | VFXA | VFXR |
| 37 | ENV | LIGHT, VFXA |
| 38 | CAM | PHYS, POST |
| 39 | APPKG | RPKG |
| 40 | UI | EUI, RG, SAVE |
| 41 | INPUT | EUI, SAVE |
| 42 | AUDIO | ASSET, SAVE |
| 43 | MOBILE | APPKG, INPUT, UI |
| 44 | WIN | APPKG, AUDIO, INPUT, UI |
| 45 | AND | MOBILE |
| 46 | APPLE | MOBILE |
| 47 | DP | RPKG |
| 48 | SHOOT | AUDIO, CAM, DP, INPUT, UI, VFXR |

Graph invariantは48 node、76 direct edge、cycle 0、self edge 0、missing node 0、redundant edge 0である。Legacy edgeの分類は次のtotal functionで決める。

```text
for each legacy edge source -> target:
  if source -> target is in Appendix B:
    retained_direct
  else if target is reachable from source in Appendix B:
    removed_transitive
  else if source/target pair is in Appendix C:
    replaced_by_integrates_with
  else:
    removed_reference_only
```

一edgeが複数分類に一致した場合は上から最初の分類だけを使用する。`removed_transitive_edges[]`はこの式で一意に再生成でき、手動判断を加えない。`D` edgeは常に`removed_reference_only`である。

## Appendix C: Reciprocal integration edge registry

各edgeはAとBのmetadata双方へ、同じ`contract_ids[]`をbyte順で書く。`requires`と同じpairであっても、normative prerequisiteとtyped runtime boundaryの両方が存在する場合は両relationを保持する。

| Edge | A | B | Reciprocal contract IDs |
|---|---|---|---|
| I01 | PP | SEC | `contract.product.capability-activation-approval` |
| I02 | PP | EVD | `contract.product.capability-qualification-receipt` |
| I03 | SEC | EVD | `contract.governance.signed-evidence-envelope` |
| I04 | COMPAT | MCD | `contract.compatibility.game-subject`; `contract.foundation.artifact-ref`; `contract.foundation.compatibility-range` |
| I05 | COMPAT | PSTATE | `contract.compatibility.game-subject`; `contract.project.migration-coverage` |
| I06 | SAVE | DBG | `contract.runtime.replay-start-checkpoint`; `contract.runtime.save-checkpoint` |
| I07 | SAVE | RPKG | `contract.compatibility.game-subject`; `contract.runtime.save-migration-set` |
| I08 | RPKG | APPKG | `contract.compatibility.game-subject`; `contract.package.runtime-package-ref` |
| I09 | APPKG | SEC | `contract.release.game-candidate-input`; `contract.release.target-preparation-record` |
| I10 | APPKG | EVD | `contract.release.package-validation-receipt`; `contract.release.store-readback-receipt` |
| I11 | APPKG | WIN | `contract.package.unsigned-windows-package`; `contract.release.target-package-mapping` |
| I12 | APPKG | AND | `contract.package.unsigned-android-package`; `contract.release.target-package-mapping` |
| I13 | APPKG | APPLE | `contract.package.unsigned-apple-payload`; `contract.release.target-package-mapping` |
| I14 | APPKG | MOBILE | `contract.package.application-assembly`; `contract.release.target-package-mapping` |
| I15 | UI | SAVE | `contract.persistence.player-settings-projection`; `contract.persistence.save-catalog` |
| I16 | INPUT | SAVE | `contract.persistence.input-replay-header`; `contract.persistence.input-settings-projection` |
| I17 | AUDIO | SAVE | `contract.persistence.audio-state-projection` |
| I18 | DP | PP | `contract.product.capability-registry`; `contract.product.work-package-registry` |
| I19 | SHOOT | PP | `contract.product.c2-coverage`; `contract.product.capability-registry` |
| I20 | RG | WIN | `contract.rendering.platform-surface-handoff`; `contract.rendering.target-capability-snapshot` |
| I21 | MOBILE | RG | `contract.rendering.platform-surface-handoff`; `contract.rendering.target-capability-snapshot` |
| I22 | NGM | RPKG | `contract.package.native-game-binary`; `contract.package.runtime-package-manifest` |
| I23 | ASSET | RPKG | `contract.asset.content-package-ref`; `contract.package.runtime-package-manifest` |
| I24 | SHADER | RPKG | `contract.package.runtime-package-manifest`; `contract.rendering.shader-artifact-ref` |
| I25 | WORLD | RPKG | `contract.package.runtime-package-manifest`; `contract.runtime.world-root-artifact` |
| I26 | PHYS | SAVE | `contract.persistence.domain-save-projection`; `contract.persistence.domain-state-restore` |
| I27 | ANIM | SAVE | `contract.persistence.domain-save-projection`; `contract.persistence.domain-state-restore` |
| I28 | NAV | SAVE | `contract.persistence.domain-save-projection`; `contract.persistence.domain-state-restore` |
| I29 | WORLD | SAVE | `contract.persistence.domain-save-projection`; `contract.persistence.domain-state-restore` |

Appendix Cにない本文Linkはmetadata relationではない。新しいtyped boundaryを発見した場合は、両Owner、Contract ID、MCD Owner、positive／negative fixtureを同じChangeSetへ追加し、片側だけを変更しない。

## Appendix D: Exact identity migration

`migration_kind=clean_replace`、`alias_retention=none`、`effective_change_set=control-plane-v1`を全行へ適用する。新IDのschema／profile versionは各Registryの`format_major`または`profile_version`、maturityはCapability Registryの`target_product_tier`へ移す。旧表記`WP7a3_2d_product_coverage_c2`はactive specに出現しないが、数字開始segmentを除去するためProduct Plan §11.5の`wp.product.2d-general-coverage`は`wp.product.general-coverage-2d`へclean replaceする。

### D.1 Capability／feature ID 33件（Shooter Coreを含む）

| Old ID | New stable ID |
|---|---|
| `capability.environment.aerial_perspective_v1` | `capability.environment.aerial_perspective` |
| `capability.environment.atmosphere_lut_v1` | `capability.environment.atmosphere_lut` |
| `capability.environment.cloud_shadow_v1` | `capability.environment.cloud_shadow` |
| `capability.environment.core_v1` | `capability.environment.core` |
| `capability.environment.dynamic_ibl_v1` | `capability.environment.dynamic_ibl` |
| `capability.environment.height_fog_v1` | `capability.environment.height_fog` |
| `capability.environment.ibl_baked_v1` | `capability.environment.ibl_baked` |
| `capability.environment.intent_resolver_v1` | `capability.environment.intent_resolver` |
| `capability.environment.local_fog_volume_v1` | `capability.environment.local_fog_volume` |
| `capability.environment.sky_hdri_v1` | `capability.environment.sky_hdri` |
| `capability.environment.volumetric_cloud_v1` | `capability.environment.volumetric_cloud` |
| `capability.environment.volumetric_fog_v1` | `capability.environment.volumetric_fog` |
| `capability.gameplay.interaction.c1` | `capability.gameplay.interaction` |
| `capability.gameplay.path_following.c1` | `capability.gameplay.path_following` |
| `capability.gameplay.perception.c1` | `capability.gameplay.perception` |
| `capability.gameplay.timer.c1` | `capability.gameplay.timer` |
| `capability.product.2d_general_production_c2` | `capability.product.general_production_2d` |
| `capability.render.material.toon_v1` | `capability.render.material.toon` |
| `capability.ui.native_widget_v1` | `capability.ui.native_widget` |
| `capability.vfx.bake_cache_v1` | `capability.vfx.bake_cache` |
| `capability.vfx.billboard_3d_v1` | `capability.vfx.billboard_3d` |
| `capability.vfx.extension_operator_v1` | `capability.vfx.extension_operator` |
| `capability.vfx.mesh_ribbon_v1` | `capability.vfx.mesh_ribbon` |
| `capability.vfx.particle_cpu_v1` | `capability.vfx.particle_cpu` |
| `capability.vfx.particle_gpu_v1` | `capability.vfx.particle_gpu` |
| `capability.vfx.particle_light_v1` | `capability.vfx.particle_light` |
| `capability.vfx.pattern_catalog_v1` | `capability.vfx.pattern_catalog` |
| `capability.vfx.semantic_intent_v1` | `capability.vfx.semantic_intent` |
| `capability.vfx.sprite_2d_v1` | `capability.vfx.sprite_2d` |
| `capability.vfx.system_v1` | `capability.vfx.system` |
| `capability.vfx.trail_v1` | `capability.vfx.trail` |
| `capability.vfx.visual_collision_v1` | `capability.vfx.visual_collision` |
| `mirakan.feature.shooter_core.c1` | `capability.gameplay.shooter_core` |

`capability.product.general_production_3d`は旧IDが存在しないためmigration rowを作らない。Product Plan Registryへ新規rowとして追加し、§11.7のGateを満たすまで`capability_activation_state=not_activated`、owner Work Packageの`scheduling_state=declared`に留める。`declared_unscheduled`等の複合state値と`lifecycle_state`軸は使用しない。

### D.2 Target、Build Driver、Domain／composition profile

| Class | Old ID | New stable ID | Version／tier destination |
|---|---|---|---|
| Target | `windows_desktop_v1` | `target.windows.desktop` | `profile_version=1` |
| Target | `windows_editor_v1` | `target.windows.editor` | `profile_version=1` |
| Target | `android_mobile_v1` | `target.android.mobile` | `profile_version=1` |
| Target | `apple_mobile_v1` | `target.apple.mobile` | `profile_version=1` |
| Driver | `windows_cmake_ninja_multi_v1` | `driver.windows.cmake-ninja-multi` | `profile_version=1` |
| Driver | `android_gradle_ninja_v1` | `driver.android.gradle-ninja` | `profile_version=1` |
| Driver | `apple_cx0_xcode_v1` | `driver.apple.cx0-xcode` | `profile_version=1` |
| Driver | `apple_modules_probe_ninja_v1` | `driver.apple.modules-probe-ninja` | `profile_version=1` |
| Driver | `apple_modules_ninja_xcode_v1` | `driver.apple.modules-ninja-xcode` | `profile_version=1` |
| Driver | `apple_xcode_cloud_v1` | `driver.apple.xcode-cloud` | `profile_version=1` |
| Domain | `mirakan.domain.2d_action.c1` | `domain.action_2d` | `target_product_tier=C1` |
| Domain | `mirakan.domain.tps_single_player.c1` | `domain.tps_single_player` | `target_product_tier=C1` |
| Profile | `shooter.profile.2d_top_down.c1` | `profile.shooter.top_down_2d` | `target_product_tier=C1` |
| Profile | `2d_top_down_c1` | `profile.shooter.top_down_2d` | same rowへmerge、duplicate sourceを拒否 |
| Profile | `shooter.profile.tps_single_player.c1` | `profile.shooter.tps_single_player` | `target_product_tier=C1` |
| Profile | `tps_single_player_c1` | `profile.shooter.tps_single_player` | same rowへmerge、duplicate sourceを拒否 |

### D.3 Package、renderer、shader profile

| Old ID | New stable ID |
|---|---|
| `android_play_v1` | `package-profile.android.play` |
| `apple_bundle_v1` | `package-profile.apple.bundle` |
| `apple_managed_assets_v1` | `package-profile.apple.managed-assets` |
| `apple_self_hosted_split_v1` | `delivery-profile.apple.self-hosted-split` |
| `windows_development_layout_v1` | `package-profile.windows.development-layout` |
| `windows_managed_layout_v1` | `package-profile.windows.managed-layout` |
| `windows_msix_v1` | `package-profile.windows.msix` |
| `portable_hlsl_2021_v1` | `shader-profile.portable-hlsl-2021` |
| `portable_mobile_v1` | `renderer-profile.portable-mobile` |
| `cpu_direct_v1` | `renderer-profile.cpu-direct` |
| `gpu_indirect_v1` | `renderer-profile.gpu-indirect` |
| `gpu_meshlet_v1` | `renderer-profile.gpu-meshlet` |
| `gpu_work_graph_v1` | `renderer-profile.gpu-work-graph` |
| `directsr_v1` | `upscaler-profile.directsr` |
| `metalfx_v1` | `upscaler-profile.metalfx` |
| `mirakan_taa_v1` | `upscaler-profile.mirakan-taa` |
| `mirakan_taau_v1` | `upscaler-profile.mirakan-taau` |
| `path_trace_preview_v1` | `render-path-profile.path-trace-preview` |
| `path_trace_reference_v1` | `render-path-profile.path-trace-reference` |
| `path_trace_runtime_v1` | `render-path-profile.path-trace-runtime` |
| `rt_reflection_v1` | `render-path-profile.rt-reflection` |
| `rt_shadow_v1` | `render-path-profile.rt-shadow` |
| `rtgi_medium_v1` | `render-path-profile.rtgi-medium` |
| `rtgi_v1` | `render-path-profile.rtgi` |

### D.4 Fixture／contract identity

| Old ID | New stable ID |
|---|---|
| `2d_shooter_c1_v1` | `fixture.product.shooter-2d` |
| `2d_platformer_c2_v1` | `fixture.product.platformer-2d` |
| `2d_puzzle_dialogue_c2_v1` | `fixture.product.puzzle-dialogue-2d` |
| `tps_shooter_c1_v1` | `fixture.product.shooter-arena-3d` |
| `2d_crowded_battle_v1` | `fixture.shooter.crowded-battle-2d` |
| `3d_crowded_battle_v1` | `fixture.shooter.crowded-battle-3d` |
| `d3d12_warp_conformance_v1` | `fixture.rendering.d3d12-warp-conformance` |
| `debug_known_faults_v1` | `fixture.debug.known-faults` |
| `clear_day_v1` | `fixture.environment.clear-day` |
| `temperate_morning_mist_v1` | `fixture.environment.temperate-morning-mist` |
| `humid_distance_haze_v1` | `fixture.environment.humid-distance-haze` |
| `dense_ground_fog_v1` | `fixture.environment.dense-ground-fog` |
| `interior_dust_shafts_v1` | `fixture.environment.interior-dust-shafts` |
| `overcast_volumetric_v1` | `fixture.environment.overcast-volumetric` |
| `stylized_tinted_fog_v1` | `fixture.environment.stylized-tinted-fog` |
| `reference_earth_atmosphere_v1` | `fixture.environment.reference-earth-atmosphere` |
| `even_floor_v1` | `fixture.shooter.even-floor` |
| `explicit_offsets_v1` | `fixture.shooter.explicit-offsets` |
| `world_authoring_cross_view_v1` | `fixture.world.authoring-cross-view` |
| `world_authoring_intent_v1` | `fixture.world.authoring-intent` |
| `world_authoring_semantics_v1` | `fixture.world.authoring-semantics` |
| `loading_progress_contract_v1` | `contract.world.loading-progress` |

### D.5 Numeric-leading logical ID correction

既にRegistryへ導入済みだがNaming正本の数字開始segment禁止に違反するIDは、次のexact mappingでclean replaceする。

| Class | Old ID | New stable ID |
|---|---|---|
| Work Package | `wp.product.2d-general-coverage` | `wp.product.general-coverage-2d` |
| Work Package | `wp.product.3d-general-coverage` | `wp.product.general-coverage-3d` |
| Capability | `capability.product.2d_general_production` | `capability.product.general_production_2d` |
| Capability | `capability.product.3d_general_production` | `capability.product.general_production_3d` |
| Domain | `domain.2d_action` | `domain.action_2d` |
| Profile | `profile.shooter.2d_top_down` | `profile.shooter.top_down_2d` |
| Fixture | `fixture.product.2d-shooter` | `fixture.product.shooter-2d` |
| Fixture | `fixture.product.2d-platformer` | `fixture.product.platformer-2d` |
| Fixture | `fixture.product.2d-puzzle-dialogue` | `fixture.product.puzzle-dialogue-2d` |
| Fixture | `fixture.product.3d-shooter-arena` | `fixture.product.shooter-arena-3d` |

Appendix Dにないmaturity／version-bearing lowercase IDをlintが発見した場合、推測変換せず`diagnostic.architecture.identity-migration-missing`を返す。Reviewerがclass、new ID、version destinationを本表へ追加するまでmigrationを停止する。

## Completion Gate

- 48 active specすべてが一つのexact metadata blockを持つ。
- Appendix Aの全legacy edgeがAppendix B／Cの分類式で一度だけ分類される。
- Appendix Bが48 node、76 edge、cycle／self／missing／redundant各0である。
- Appendix Cの29 edgeが完全にreciprocalで、Contract ID集合が一致する。
- `architecture/registry/document-relations.v1.json`が48文書metadataの`requires`／`integrates_with`およびAppendix Bのcanonical orderと完全一致する。
- Appendix Dの全old ID出現数が0、new IDのorphanが0、runtime aliasが0である。
- Product RegistryはTarget 5、Requirement 16、Fixture 10、Phase 10、Capability 36、Work Package 25を参照解決し、missing Target activationをfail closedにする。
- TypeScript 7.0.2 compileは`--singleThreaded`を使用し、compiler API importが0件である。
- Index二回生成のSHA-256が一致し、Git diffが空である。
- baseline handoffの全hashをECS／D3D12計画がread-backできる。
