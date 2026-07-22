# Runtime ECS E0 Contract Baseline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Engine-owned archetype ECSのE0正本、MCD、generated contract、cross-document ownership、positive／negative fixtureをclean-breakで確定し、E1 storage kernelが推測なく開始できるbaselineを作る。

**Architecture:** E0はRuntime storageを実装せず、Entity／Component／Query／Access／Structural transaction／Cook image／Native ABI／Subsystem Port／Save projection／AI R0 operationのcontractとtest oracleを固定する。ECS固有規則を新Ownerへ一度だけ移し、Memory、Scheduling、Gameplay、Native Moduleは共通原則またはconsumer boundaryだけを保持する。

**Tech Stack:** Miraikanai Contract Definition、JSON Schema Draft 2020-12、C++23 generated types、TypeScript generated projection、MCP JSON Schema、Node.js／TypeScript contract compiler、CTest、TLA+／TLC 1.7.4（commit／publication state machine）。

## Global Constraints

- Task 0の共通tooling準備とTask 1のECS正本bootstrapを除き、Task 3以降はTask 2のexact entry gate通過前に開始しない。対象であるECS自身の`approved`を開始条件にしない。
- 開始条件は`architecture_baseline_receipt_hash != null`、Architecture Governance／Compatibility Evolution／Persistence Saveの各state=`approved`、Entity Component Systemのstate=`review`の五件だけとする。
- `architecture/baselines/control-plane-v1.json`はexact 9 fieldsをread-backし、baseline不一致、Owner非承認、ECS state不一致、dirty treeをそれぞれstable Diagnostic IDで拒否する。
- 新正本IDは`mirakan.arch.runtime-entity-component-system`、Work Packageは`wp.runtime.ecs-e0`、初期stateは`review`、Activationは全Targetで`not_activated`とする。
- Flecs、EnTT、Unity Entities、Unreal Massをdependencyまたは互換APIとして追加しない。
- C1値はchunk payload 16 KiB、base alignment 64 bytes、inline Component最大256 bytes、Entity handle `uint32 index + uint32 generation`、index 0 invalid、generation wrap retireである。
- Runtime storage、raw chunk、Runtime handle、pointer、lease、archetype IDをSave／Replayへ永続化しない。
- Structural operationは`create_entity`、`destroy_entity`、`add_component`、`remove_component`、`set_component_enabled`、`set_entity_enabled`の六つだけで、一batch一commit point、failure前後live digest不変を要求する。
- Query order、archetype／chunk／row iteration、command merge、serializationをcanonical化し、thread完了順、address、filesystem順をidentityにしない。
- AI Runtime operationはR0 read-only captureだけで、mutationはAuthoring ChangeSetを使う。
- 旧`EntityHandle` alias、旧immutable query batch、旧Component State delta、dual manifest、compatibility wrapperを残さない。

---

## 1. File map

| Path | Responsibility |
|---|---|
| `docs/architecture/04-runtime/entity-component-system.md` | ECS唯一のactive Architecture正本 |
| `schemas/runtime/ecs/component-spec.mcd` | Component／Field／presence／external handle contract |
| `schemas/runtime/ecs/entity-handle.mcd` | Handle／World ref／persistent identity contract |
| `schemas/runtime/ecs/archetype-plan.mcd` | Archetype／transition／chunk layout contract |
| `schemas/runtime/ecs/query-access.mcd` | Query normalization、access manifest、lease contract |
| `schemas/runtime/ecs/structural-command.mcd` | 六operation、precondition、atomic batch contract |
| `schemas/runtime/ecs/world-image.mcd` | Root／Section inner image、outer binding contract |
| `schemas/runtime/ecs/save-projection.mcd` | Field ID projection、identity sequence、tombstone contract |
| `schemas/runtime/ecs/native-abi.mcd` | fixed-width C ABI descriptor／view／writer contract |
| `schemas/runtime/ecs/ai-operations.mcd` | authorized R0 capture／query operation |
| `schemas/runtime/ecs/diagnostics.mcd` | stable ECS diagnostic registry |
| `tools/contract_compiler/**` | MCD→C++／TypeScript／JSON Schema／MCP projectionを生成するfirst-party contract compiler CLI |
| `tests/contracts/CMakeLists.txt`、`tests/contracts/runtime_ecs/CMakeLists.txt` | contract test用CMake／CTest harness scaffold |
| `tests/contracts/runtime_ecs/**` | golden bytes、schema、cross-language、negative fixtures |
| `models/runtime_ecs/StructuralCommit.tla` | structural transaction atomicity model |
| `models/runtime_ecs/WorldPublication.tla` | participant prepare／hidden attach／single publish model |

## 2. E0 outputs

E0が生成するpublic contractは次に限定する。

```text
RuntimeComponentSpecV1
RuntimeEntityHandle
RuntimeEntityRefV1
PersistentEntityIdentityV1
RuntimeEntityInitializerSpecV1
RuntimeEntityTemplateV1
RuntimeArchetypePlanV1
RuntimeChunkLayoutFactV1
RuntimeEntityQuerySpecV1
RuntimeComponentAccessManifestV1
RuntimeSystemExecutionContextV1
StructuralCommandBatchV1
RuntimeWorldRootImageV1
RuntimeWorldSectionImageV1
RuntimeWorldPublicationHandle
RuntimePersistentEntityHandoffV1
RuntimeEntityProjectionV1
RuntimeComponentProjectionV1
RuntimeEcsContractGraphV1
```

Suffixなしlatest aliasを生成しない。§2のうち`RuntimeEntityHandle`と`RuntimeWorldPublicationHandle`はwire schemaではなく、[Naming／Project Layout](../architecture/02-foundation/naming-project-layout.md#31-cとschema-type)が定義する`serializable = false`のprocess-local public C++ value type categoryに属する。ECS内だけのsuffix例外Registryは作らず、境界を越える型はversion付きの別Schema typeとして定義する。

## 3. Clean migration inventory

| Legacy owner／symbol | E0処置 | 移行後のlegacy owner |
|---|---|---|
| `memory-pointers.md`の16 KiB chunk、64-byte alignment、256-byte inline cap、archetype説明 | ECS §Chunk layoutへ移動 | 一般allocator、generation handle、lease原則だけを保持 |
| `scheduling-lifetime.md`の`ComponentAccessManifest` | `RuntimeComponentAccessManifestV1`へclean renameしECSへ移動 | Schedulingはphase assignmentとgraph consumerだけを保持 |
| `scheduling-lifetime.md`のarchetype／SoA／location table／lease／Structural command batch | ECS storage／query／transaction節へ移動 | Schedulingはcommit boundaryとscheduler handoffだけを参照 |
| `gameplay-programming-model.md`のComponent composition／System access／SaveReplay field classification | ECS Component／System relationとPersistence projectorへ接続 | GameplayはGameSystem intent、Owner、Command／Event意味だけを保持 |
| `native-game-module.md`の`EntityHandle` | `RuntimeEntityRefV1`へclean replace | 旧alias 0 |
| `native-game-module.md`のimmutable query batch | generated readonly／mutable column descriptorへreplace | 旧batch API 0 |
| `native-game-module.md`のComponent State delta | callback-scoped State view＋Structural／Command writerへreplace | 旧delta API 0 |

移行testは旧symbolの全出現をpath＋lineでsnapshotし、new ownerへ移した後に許可されたhistorical Decision内引用以外0件を要求する。

## 4. Tasks

### Task 0: Contract compilerとcontract test harnessを準備する

**Files:**
- Create: `tools/contract_compiler/package.json`
- Create: `tools/contract_compiler/package-lock.json`
- Create: `tools/contract_compiler/tsconfig.json`
- Create: `tools/contract_compiler/src/main.ts`
- Create: `tools/contract_compiler/src/parse.ts`
- Create: `tools/contract_compiler/src/emit.ts`
- Test: `tools/contract_compiler/test/self-emit.test.mjs`
- Test fixture (positive): `tools/contract_compiler/test/fixtures/valid/minimal-contract.mcd`
- Test fixture (negative): `tools/contract_compiler/test/fixtures/invalid/malformed-contract.mcd`
- Test golden: `tools/contract_compiler/test/golden/minimal-contract/**`
- Create: `tests/contracts/CMakeLists.txt`
- Create: `tests/contracts/runtime_ecs/CMakeLists.txt`

**Interfaces:**
- Consumes: Toolchain lockのJavaScript／TypeScript ESM toolchainと、Executable Contractsのcanonical binary／生成規則。
- Produces: Task 3〜11が消費するCLI `node tools/contract_compiler/dist/main.js compile <input.mcd> --out <directory>`、CMake target `runtime_ecs_contract_tests`、CTest name `runtime_ecs.contract`、compiler／harnessのtoolchain hash。

- [ ] **Step 1: positive fixtureからC++／TypeScript／JSON Schema／MCPを生成してgoldenと比較し、二回生成bytes一致を検査するself-testを書く**
- [ ] **Step 2: negative fixtureがsyntax locationを持つtyped parse failureになるtestを書き、compiler未実装で両testが失敗することを確認する**
- [ ] **Step 3: Node.js標準libraryとlock済みTypeScript CLIだけでcompilerを実装し、production dependencyを追加しない**
- [ ] **Step 4: `runtime_ecs_contract_tests`と`runtime_ecs.contract`をCMake／CTestへ登録し、後続Taskが名前で参照できることを検査する**
- [ ] **Step 5: 次のcommandを順に実行し、compiler／harness toolchain hashをTask 11のReceipt入力として記録する**

```powershell
npm ci --prefix tools/contract_compiler --ignore-scripts --offline --no-audit --no-fund
npx --prefix tools/contract_compiler tsc --build --force --singleThreaded
node --test tools/contract_compiler/test/self-emit.test.mjs
cmake -S tests/contracts -B build/contract-tests -G Ninja -DCMAKE_BUILD_TYPE=Debug
cmake --build build/contract-tests --target runtime_ecs_contract_tests
ctest --test-dir build/contract-tests -R '^runtime_ecs\.contract$' --output-on-failure
```

Expected: self-test PASS、二回生成bytes一致、malformed MCD拒否、CMake targetとCTest nameをexactに発見する。Task 0はECS Decision非依存の共通toolingであり、E0 contract内容を先行実装しない。Task 0のtoolchain hashはE0 Receiptへbindし、既に承認されたControl Plane baselineへ書き戻さない。

### Task 1: ECS active正本とmetadata relationを`review`で作成する

**Files:**
- Create: `docs/architecture/04-runtime/entity-component-system.md`
- Modify: `docs/architecture/README.md`
- Modify: `architecture/registry/document-relations.v1.json`
- Test: `tools/architecture_lint/test/ecs-document.test.mjs`

**Interfaces:**
- Produces: document ID `mirakan.arch.runtime-entity-component-system`。

- [ ] **Step 1: missing owner、duplicate canonical type、non-reciprocal integration testを書く**
- [ ] **Step 2: failureを確認する**
- [ ] **Step 3: Decision §8～§17をactive正本へ移す**

Metadataのdirect `requires`は`["mirakan.arch.runtime-scheduling-lifetime"]`だけとする。次のreciprocal integrationを同じChangeSetで両側へ追加する。

| Peer | Contract IDs |
|---|---|
| `mirakan.arch.architecture-governance` | `contract.architecture.change-set` |
| `mirakan.arch.compatibility-evolution` | `contract.compatibility.ecs-contract-version`; `contract.compatibility.migration-coverage` |
| `mirakan.arch.executable-contracts` | `contract.foundation.mcd-canonical-binary`; `contract.runtime.ecs-contract-graph` |
| `mirakan.arch.memory-pointers` | `contract.foundation.generation-handle`; `contract.foundation.scoped-lease` |
| `mirakan.arch.gameplay-programming-model` | `contract.gameplay.component-access`; `contract.gameplay.system-owner` |
| `mirakan.arch.native-game-module` | `contract.native.ecs-batch-abi`; `contract.native.runtime-entity-ref` |
| `mirakan.arch.runtime-persistence-save` | `contract.persistence.ecs-save-projection`; `contract.persistence.ecs-state-restore` |
| `mirakan.arch.runtime-package` | `contract.package.ecs-contract-set`; `contract.package.world-image-ref` |
| `mirakan.arch.runtime-debugging-observability-replay` | `contract.runtime.ecs-capture`; `contract.runtime.ecs-replay-digest` |

Scheduling metadataには`contract.runtime.ecs-phase-handoff`と`contract.runtime.structural-commit-boundary`をreciprocalに登録する。ECS document IDを既存Ownerの`requires`へ追加しないためcycleを作らない。

同じChangeSetで`wp.runtime.ecs-e0.owner_document_id`を暫定Owner `mirakan.arch.runtime-scheduling-lifetime`から新Owner `mirakan.arch.runtime-entity-component-system`へ置換する。新文書追加前または別ChangeSetでownerだけを変更しない。

- [ ] **Step 4: generated Indexとarchitecture lintを実行する**

Expected: active inventory +1、cycle／redundant／Owner duplicate 0、固定件数表記0。ECS metadataは`state = review`、`approval_ref = null`、全Target activation=`not_activated`であり、正本作成をapprovalまたはCapability昇格とみなさない。

### Task 2: Control Plane baselineとOwner stateのE0 entry gateを追加する

**Files:**
- Modify: `architecture/registry/document-relations.v1.json`
- Modify: `architecture/registry/product.v1.json`
- Test: `tools/architecture_lint/test/ecs-e0-entry-gate.test.mjs`
- Test fixture: `tools/architecture_lint/test/fixtures/ecs-entry/valid.json`
- Test fixture: `tools/architecture_lint/test/fixtures/ecs-entry/baseline-mismatch.json`
- Test fixture: `tools/architecture_lint/test/fixtures/ecs-entry/owner-unapproved.json`
- Test fixture: `tools/architecture_lint/test/fixtures/ecs-entry/ecs-not-review.json`

**Interfaces:**
- Consumes: Task 1のECS `review` metadataと`architecture/baselines/control-plane-v1.json`。
- Produces: booleanではなくexact failure diagnosticを返すE0 entry gate。Task 3〜11はこのgateを必須predecessorにする。

Entry conditionは次の五件のconjunctionだけである。

```text
architecture_baseline_receipt_hash != null
architecture_governance.state == approved
compatibility_evolution.state == approved
persistence_save.state == approved
entity_component_system.state == review
```

- [ ] **Step 1: baseline hash missing／mismatch、三Ownerの各非承認、ECS state非review、dirty treeのnegative testsを書く**
- [ ] **Step 2: `control-plane-v1.json`のexact 9 fieldsの過不足とhash mismatchを検出する**
- [ ] **Step 3: 五条件を評価し、最初のfailureをcanonical diagnostic順で返すgateを実装する**
- [ ] **Step 4: cleanなvalid fixtureでPASSし、ECSが`approved`でなくても`review`なら到達できることを確認する**

baselineのexact fieldsは`git_tree_id`、`architecture_index_sha256`、`document_relation_registry_sha256`、`product_registry_sha256`、`identity_migration_registry_sha256`、`architecture_explain_schema_sha256`、`toolchain_lock_sha256`、`architecture_lint_artifact_sha256`、`lint_version`である。Expected diagnosticはbaseline=`diagnostic.architecture.baseline-mismatch`、Owner=`diagnostic.architecture.owner-unapproved`、ECS state=`diagnostic.architecture.ecs-review-state-required`、dirty=`diagnostic.architecture.dirty-baseline`。Task 0のcompiler／harness hashはE0 Receiptへbindし、Control Plane baselineまたはentry conditionを暗黙拡張しない。

### Task 3: Component、Entity、Archetype MCDをtest-firstで追加する

**Files:**
- Create: `schemas/runtime/ecs/component-spec.mcd`
- Create: `schemas/runtime/ecs/entity-handle.mcd`
- Create: `schemas/runtime/ecs/archetype-plan.mcd`
- Test: `tests/contracts/runtime_ecs/component_entity_archetype_tests.cpp`

**Interfaces:**
- Produces: Component／Field ID、handle、initializer、template、archetype、transition、chunk layout schema。

- [ ] **Step 1: index 0、generation wrap、oversize Component、non-trivial Field、tag／singletonへのenable bitset、unplanned transitionのnegative fixturesを書く**
- [ ] **Step 2: schema missing failureを確認する**
- [ ] **Step 3: Decision §8.1～§8.5のexact fields／bounds／orderingをMCDへ定義する**
- [ ] **Step 4: generated C++／TypeScript／JSON Schemaのshapeを比較する**

Expected: 16 KiB、64-byte、256-byteがECS schemaに一度だけ存在し、Memory文書では値定義0。`enable_bit`対応のtrivially stored Componentだけがentity slotごとにexact 1 bitを持ち、tag／singletonのComponent bitsetは0件。

### Task 4: Query、Access、Structural transaction、Diagnostic registry MCDを追加する

**Files:**
- Create: `schemas/runtime/ecs/query-access.mcd`
- Create: `schemas/runtime/ecs/structural-command.mcd`
- Create: `schemas/runtime/ecs/diagnostics.mcd`
- Test: `tests/contracts/runtime_ecs/query_structural_tests.cpp`

**Interfaces:**
- Produces: normalized query、closed partition policy、`RuntimeSystemExecutionContextV1`、access manifest、lease、六operation batch、ECS Diagnostic ID registry。

- [ ] **Step 1: all／any／none／optional truth table、source term permutation、`single | fixed_range | deterministic_hash`のpositive／negative partition fixtureを書く**
- [ ] **Step 2: non-owner write、iteration中mutation、conflict、capacity failure testを書く**
- [ ] **Step 3: Decision §9～§10のcontractをMCDへ定義し、execution contextのtick、phase、partition、read snapshot、write batch、diagnostic sinkをexact fieldとして生成する**
- [ ] **Step 4: Decision §15のDiagnostic IDとdetection stageを`diagnostics.mcd`のclosed registryへ定義し、unknown／duplicate Diagnostic ID拒否のnegative fixtureを追加する**
- [ ] **Step 5: canonical hashとfailure atomicity fixtureを実行する**

Expected: term順を変えてもquery hash一致、worker数／完了順を変えてもpartition割当てと結果順が一致、execution context field過不足0、invalid batch前後のlive digest一致、operation enumは6件、Diagnostic ID重複0・unknown ID拒否。

### Task 5: Root／Section imageとhash非循環を固定する

**Files:**
- Create: `schemas/runtime/ecs/world-image.mcd`
- Create: `tests/contracts/runtime_ecs/fixtures/world-root-golden.bin`
- Create: `tests/contracts/runtime_ecs/fixtures/world-section-golden.bin`
- Test: `tests/contracts/runtime_ecs/world_image_tests.cpp`

**Interfaces:**
- Produces: inner content image、outer Artifact／Qualification binding、publication handle。

- [ ] **Step 1: self hash、Root final image ref、Receipt-in-inner-image、non-canonical order、stale store／participant generation、partial publishのnegative testsを書く**
- [ ] **Step 2: failureを確認する**
- [ ] **Step 3: Decision §8.6のbinary layout、content hash、Catalog DAGを実装する**
- [ ] **Step 4: encode→decode→re-encode golden byte testを実行する**

Expected: byte identical、trailing byte／unknown enum拒否、hash cycle 0。store generationはowner participant generationと同一counterで、prepare／abortは消費せず、ECSと全storeのpending generationは一回のpublication handle切替えでだけ同時可視になる。

### Task 6: Save／Replay projectionをPersistence Ownerへ接続する

**Files:**
- Create: `schemas/runtime/ecs/save-projection.mcd`
- Modify: `docs/architecture/04-runtime/persistence-save.md`
- Modify: `docs/architecture/04-runtime/debugging-observability-replay.md`
- Test: `tests/contracts/runtime_ecs/save_projection_tests.cpp`

**Interfaces:**
- Produces: lifecycle、composition、enable、Field、tombstone、identity sequence、ephemeral last-issued counterのtyped records。

- [ ] **Step 1: raw handle、archetype ID、chunk bytes、orphan Field、missing migration testを書く**
- [ ] **Step 2: failureを確認する**
- [ ] **Step 3: Decision §11.4とControl Plane Persistence contractへprojectionを定義する**
- [ ] **Step 4: Save→Load→Replay seek digest testを実行する**

Expected: Persistent identity／Field digest一致、ephemeral Entity payload保存0、raw storage保存0。

### Task 7: Native ABIとSubsystem Portを固定する

**Files:**
- Create: `schemas/runtime/ecs/native-abi.mcd`
- Modify: `docs/architecture/03-authoring/native-game-module.md`
- Modify: `docs/architecture/05-simulation/collision.md`
- Modify: `docs/architecture/05-simulation/physics.md`
- Modify: `docs/architecture/05-simulation/navigation.md`
- Modify: `docs/architecture/05-simulation/animation.md`
- Modify: `docs/architecture/06-rendering/camera.md`
- Modify: `docs/architecture/06-rendering/environment-surfaces.md`
- Modify: `docs/architecture/06-rendering/lighting.md`
- Modify: `docs/architecture/06-rendering/lod.md`
- Modify: `docs/architecture/06-rendering/materials.md`
- Modify: `docs/architecture/06-rendering/post-processing.md`
- Modify: `docs/architecture/06-rendering/project-shader.md`
- Modify: `docs/architecture/06-rendering/render-graph.md`
- Modify: `docs/architecture/06-rendering/vfx-authoring.md`
- Modify: `docs/architecture/06-rendering/vfx-runtime.md`
- Modify: `docs/architecture/06-rendering/world.md`
- Modify: `docs/architecture/07-platform/audio.md`
- Modify: `docs/architecture/07-platform/input.md`
- Modify: `docs/architecture/07-platform/ui-text-localization-accessibility.md`
- Test: `tests/contracts/runtime_ecs/native_port_tests.cpp`

**Interfaces:**
- Produces: fixed-width C ABI descriptor、callback-scoped column／State views、typed external handle transaction、publication participant。

- [ ] **Step 1: C bool、compiler enum、pointer retention、callback-after-return、partial participant publish testsを書く**
- [ ] **Step 2: failureを確認する**
- [ ] **Step 3: Decision §12～§13のABI／Port contractをMCDへ定義する**
- [ ] **Step 4: C／C++ layout fingerprintとparticipant failure injectionを実行する**

Expected: ABI size／offset一致、pointer escape 0、partial generation visible 0。

### Task 8: AI R0 operationとsensitivityを固定する

**Files:**
- Create: `schemas/runtime/ecs/ai-operations.mcd`
- Modify: `docs/architecture/01-governance/ai-security-approval.md`
- Modify: `docs/architecture/01-governance/ai-verification-provenance.md`
- Test: `tests/contracts/runtime_ecs/ai_operation_tests.cpp`

**Interfaces:**
- Produces: `RuntimeEcsContractGraphV1` query、bounded capture、continuation、authorization mapping。

- [ ] **Step 1: missing Task Authorization、wrong channel grant、restricted Field、runtime mutation testsを書く**
- [ ] **Step 2: failureを確認する**
- [ ] **Step 3: Decision §14のR0 operationだけを生成する**
- [ ] **Step 4: product internal／local MCP／managed CLI variantsを検査する**

Expected: mutation operation 0、unauthorized read 0、redactionとabsent defaultを区別、bound超過はcontinuationを返す。

### Task 9: 4 legacy ownerからECS固有正本をclean移行する

**Files:**
- Modify: `docs/architecture/02-foundation/memory-pointers.md`
- Modify: `docs/architecture/03-authoring/gameplay-programming-model.md`
- Modify: `docs/architecture/03-authoring/native-game-module.md`
- Modify: `docs/architecture/04-runtime/scheduling-lifetime.md`
- Test: `tools/architecture_lint/test/ecs-clean-migration.test.mjs`

**Interfaces:**
- Consumes: §3 inventory。
- Produces: one-owner contract graph、old symbol occurrence 0。

- [ ] **Step 1: 現在のold phrase／symbol occurrence snapshot testを書く**
- [ ] **Step 2: snapshotがlegacy定義を検出することを確認する**
- [ ] **Step 3: 各定義をECSへ移し、consumerにはOwner linkとContract refだけを残す**
- [ ] **Step 4: clean migration testを実行する**

Expected: `EntityHandle` public alias 0、ECS値重複0、`ComponentAccessManifest` suffixなし定義0、immutable query batch API 0。

### Task 10: Structural／Publication modelをTLCで検証する

**Files:**
- Create: `models/runtime_ecs/StructuralCommit.tla`
- Create: `models/runtime_ecs/StructuralCommit.cfg`
- Create: `models/runtime_ecs/WorldPublication.tla`
- Create: `models/runtime_ecs/WorldPublication.cfg`

**Interfaces:**
- Produces: atomicity、old-generation visibility、single publication invariantのmodel-check Receipt。

- [ ] **Step 1: invariantを定義する**

```text
LiveDigestChangesOnlyAtCommit
FailedBatchPreservesLiveDigest
HiddenGenerationNotVisible
PublicationHandleIsSingleCommitPoint
```

- [ ] **Step 2: 意図的にbroken transitionでTLC violationを確認する**
- [ ] **Step 3: E0 state machineを正しく記述する**
- [ ] **Step 4: TLCを実行する**

Run: `java -jar tools/tla/tla2tools-1.7.4.jar -config models/runtime_ecs/StructuralCommit.cfg models/runtime_ecs/StructuralCommit.tla`

Expected: Model checking completed、invariant violation 0。Publication modelも同じ結果。

### Task 11: E0 conformanceとhandoff baselineを生成する

**Files:**
- Create: `architecture/baselines/runtime-ecs-e0-v1.json`
- Test: `tests/contracts/runtime_ecs/e0_conformance_tests.cpp`

**Interfaces:**
- Produces: E1が消費するexact baseline。

- [ ] **Step 1: Contract／static、correctness oracle、determinism、integration／AIのE0 subsetと、後続C1／C2 Qualification handoffをmatrix化する**
- [ ] **Step 2: C++／TypeScript／JSON Schema／MCP generationを二回行う**
- [ ] **Step 3: 全test、architecture lint、TLCを実行する**
- [ ] **Step 4: baseline JSONを生成しread-backする**

Baselineは`control_plane_baseline_ref`、ECS正本hash、MCD registry hash、generated C++／TypeScript／JSON Schema／MCP hash、golden fixture root hash、TLC model hash、test receipt refsを持つ。

E0はRuntimeを実装しないため、負荷Receiptそのものを完了条件にしない。代わりに次のowner／fixture／負荷条件をbaselineへexact handoffし、後続Work PackageがRuntime実装後に実行する。

| Qualification | Consumer Work Package | Fixture／load | Duration／required evidence |
|---|---|---|---|
| C1 | `wp.gameplay.core-c1` | Product RegistryのFirst Playable fixture。entity数はTarget別canonical C1 capacity以内 | 30分連続soak、leak／generation異常／query cache drift／unbounded archetype増加0 |
| C2／stress | `wp.product.general-coverage-2d`; `wp.product.general-coverage-3d` | 100万Entity synthetic、全archetype、全到達可能persistent handoff matrix | 2時間endurance、C1と同じ異常0、positive／negative handoff coverage 100% |

C2／stress Receiptを`runtime-ecs-e0-v1.json`へ偽装せず、両Consumer Work PackageのCandidate／Target別Qualification Receiptとして保持する。

Expected: 二回生成bytes一致、old symbol 0、orphan contract 0、test error 0。E0は`review`または承認済みContract baselineであり、Runtime Capabilityを自動昇格しない。

## Completion Gate

- Task 2のexact五条件を満たし、Control Plane baselineのexact 9 fieldsと`architecture_baseline_receipt_hash`をread-backできる。
- ECS active正本が`review`またはその後の別approvalで`approved`の一Ownerとして存在し、metadata graphのcycle／redundant／reciprocity errorが0である。E0 baseline生成はapprovalを自動付与しない。
- §2の19型のうちwire contract 17件がMCD、C++、TypeScript、JSON Schema、MCP projectionで一致する。Naming正本のin-memory value type categoryに属する`RuntimeEntityHandle`と`RuntimeWorldPublicationHandle`はwire表現から除外し、C++ layout fingerprintの一致だけを検証する。
- 16 KiB、64-byte、256-byteのECS値がECS正本に一度だけ存在する。
- enable bitset、closed partition policy、execution contextの六責務、store／participant generation表についてpositive／negative fixtureが合格する。
- ECS Diagnostic IDが`diagnostics.mcd`へ一意登録され、Decision §15 registryとの差分、unknown ID、orphan IDが0である。
- 旧`EntityHandle` alias、suffixなしAccess Manifest、immutable query batch、Component State deltaが0件である。
- Query canonicalization、structural atomicity、world image hash非循環、Save projection、Native ABI、AI authorizationのpositive／negative fixtureが合格する。
- TLCの4 invariantがviolation 0である。
- `runtime-ecs-e0-v1.json`の全hashをE1計画がread-backできる。
- C1のFirst Playable／canonical capacity／30分soakと、C2／stressの100万Entity／2時間／全archetype handoff matrixが別Consumer Work Packageへbindされ、C2／stress ReceiptをE0完了条件にしていない。
- Storage kernel、package、C1／C2負荷Receipt、Capability activationをE0完了として偽装していない。
