# Runtime ECS E0 Contract Baseline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Engine-owned archetype ECSのE0正本、MCD、generated contract、cross-document ownership、positive／negative fixtureをclean-breakで確定し、E1 storage kernelが推測なく開始できるbaselineを作る。

**Architecture:** E0はRuntime storageを実装せず、Entity／Component／Query／Access／Structural transaction／Cook image／Native ABI／Subsystem Port／Save projection／AI R0 operationのcontractとtest oracleを固定する。ECS固有規則を新Ownerへ一度だけ移し、Memory、Scheduling、Gameplay、Native Moduleは共通原則またはconsumer boundaryだけを保持する。

**Tech Stack:** Miraikanai Contract Definition、JSON Schema Draft 2020-12、C++23 generated types、TypeScript generated projection、MCP JSON Schema、Node.js／TypeScript contract compiler、CTest、TLA+／TLC 1.7.4（commit／publication state machine）。

## Global Constraints

- 本Decisionが`ユーザー承認済み`になるまでE0実装を開始しない。計画書の存在を承認とみなさない。
- 開始前に`architecture/baselines/control-plane-v1.json`の全hashをread-backし、不一致、missing、dirty treeを`diagnostic.architecture.baseline-mismatch`で拒否する。
- 新正本IDは`mirakan.arch.runtime-entity-component-system`、Work Packageは`wp.runtime.ecs-e0`、初期stateは`review`、Activationは全Targetで`not_activated`とする。
- Flecs、EnTT、Unity Entities、Unreal Massをdependencyまたは互換APIとして追加しない。
- C1値はchunk payload 16 KiB、base alignment 64 bytes、inline Component最大256 bytes、Entity handle `uint32 index + uint32 generation`、index 0 invalid、generation wrap retireである。
- Runtime storage、raw chunk、Runtime handle、pointer、lease、archetype IDをSave／Replayへ永続化しない。
- Structural operationはcreate、destroy、add、remove、enable_component、enable_entityの六つだけで、一batch一commit point、failure前後live digest不変を要求する。
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
RuntimeChunkLayoutV1
RuntimeEntityQuerySpecV1
RuntimeComponentAccessManifestV1
StructuralCommandBatchV1
RuntimeWorldRootImageV1
RuntimeWorldSectionImageV1
RuntimeWorldPublicationHandleV1
RuntimePersistentEntityHandoffV1
RuntimeEntityProjectionV1
RuntimeComponentProjectionV1
RuntimeEcsContractGraphV1
```

Suffixなしlatest aliasを生成しない。`RuntimeEntityHandle`だけはDecisionで固定したin-memory value typeでありwire schemaではないため`V1` suffix例外とし、Architecture Governanceの例外Registryへ理由とOwnerを登録する。

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

### Task 1: Approval／Control Plane baseline Gateを追加する

**Files:**
- Modify: `architecture/registry/document-relations.v1.json`
- Modify: `architecture/registry/product.v1.json`
- Test: `tools/architecture_lint/test/ecs-e0-entry-gate.test.mjs`

**Interfaces:**
- Consumes: ECS Decision status、Control Plane baseline。
- Produces: booleanではなくexact failure diagnosticを返すE0 entry gate。

- [ ] **Step 1: pending approval、baseline mismatch、dirty treeの3 negative testsを書く**
- [ ] **Step 2: 現状がpending approvalで失敗することを確認する**
- [ ] **Step 3: 承認後だけbaseline read-backへ進むgateを実装する**
- [ ] **Step 4: clean approved fixtureでPASSすることを確認する**

Expected: pending=`diagnostic.architecture.decision-approval-missing`、mismatch=`diagnostic.architecture.baseline-mismatch`、dirty=`diagnostic.architecture.dirty-baseline`。

### Task 2: ECS active正本とmetadata relationを作成する

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

Expected: active inventory +1、cycle／redundant／Owner duplicate 0、固定件数表記0。

### Task 3: Component、Entity、Archetype MCDをtest-firstで追加する

**Files:**
- Create: `schemas/runtime/ecs/component-spec.mcd`
- Create: `schemas/runtime/ecs/entity-handle.mcd`
- Create: `schemas/runtime/ecs/archetype-plan.mcd`
- Test: `tests/contracts/runtime_ecs/component_entity_archetype_tests.cpp`

**Interfaces:**
- Produces: Component／Field ID、handle、initializer、template、archetype、transition、chunk layout schema。

- [ ] **Step 1: index 0、generation wrap、oversize Component、non-trivial Field、unplanned transitionのnegative fixturesを書く**
- [ ] **Step 2: schema missing failureを確認する**
- [ ] **Step 3: Decision §8.1～§8.5のexact fields／bounds／orderingをMCDへ定義する**
- [ ] **Step 4: generated C++／TypeScript／JSON Schemaのshapeを比較する**

Expected: 16 KiB、64-byte、256-byteがECS schemaに一度だけ存在し、Memory文書では値定義0。

### Task 4: Query、Access、Structural transaction MCDを追加する

**Files:**
- Create: `schemas/runtime/ecs/query-access.mcd`
- Create: `schemas/runtime/ecs/structural-command.mcd`
- Test: `tests/contracts/runtime_ecs/query_structural_tests.cpp`

**Interfaces:**
- Produces: normalized query、access manifest、lease、六operation batch。

- [ ] **Step 1: all／any／none／optional truth tableとsource term permutation testを書く**
- [ ] **Step 2: non-owner write、iteration中mutation、conflict、capacity failure testを書く**
- [ ] **Step 3: Decision §9～§10のcontractをMCDへ定義する**
- [ ] **Step 4: canonical hashとfailure atomicity fixtureを実行する**

Expected: term順を変えてもquery hash一致、invalid batch前後のlive digest一致、operation enumは6件。

### Task 5: Root／Section imageとhash非循環を固定する

**Files:**
- Create: `schemas/runtime/ecs/world-image.mcd`
- Create: `tests/contracts/runtime_ecs/fixtures/world-root-golden.bin`
- Create: `tests/contracts/runtime_ecs/fixtures/world-section-golden.bin`
- Test: `tests/contracts/runtime_ecs/world_image_tests.cpp`

**Interfaces:**
- Produces: inner content image、outer Artifact／Qualification binding、publication handle。

- [ ] **Step 1: self hash、Root final image ref、Receipt-in-inner-image、non-canonical orderのnegative testsを書く**
- [ ] **Step 2: failureを確認する**
- [ ] **Step 3: Decision §8.6のbinary layout、content hash、Catalog DAGを実装する**
- [ ] **Step 4: encode→decode→re-encode golden byte testを実行する**

Expected: byte identical、trailing byte／unknown enum拒否、hash cycle 0。

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

- [ ] **Step 1: Contract／static、correctness oracle、determinism、integration／AIのE0 subsetをmatrix化する**
- [ ] **Step 2: C++／TypeScript／JSON Schema／MCP generationを二回行う**
- [ ] **Step 3: 全test、architecture lint、TLCを実行する**
- [ ] **Step 4: baseline JSONを生成しread-backする**

Baselineは`control_plane_baseline_ref`、ECS正本hash、MCD registry hash、generated C++／TypeScript／JSON Schema／MCP hash、golden fixture root hash、TLC model hash、test receipt refsを持つ。

Expected: 二回生成bytes一致、old symbol 0、orphan contract 0、test error 0。E0は`review`または承認済みContract baselineであり、Runtime Capabilityを自動昇格しない。

## Completion Gate

- ECS Decisionが承認済みで、Control Plane baselineの全hashをread-backできる。
- ECS active正本が一Ownerとして存在し、metadata graphのcycle／redundant／reciprocity errorが0である。
- §2の17 contractがMCD、C++、TypeScript、JSON Schema、MCP projectionで一致する。
- 16 KiB、64-byte、256-byteのECS値がECS正本に一度だけ存在する。
- 旧`EntityHandle` alias、suffixなしAccess Manifest、immutable query batch、Component State deltaが0件である。
- Query canonicalization、structural atomicity、world image hash非循環、Save projection、Native ABI、AI authorizationのpositive／negative fixtureが合格する。
- TLCの4 invariantがviolation 0である。
- `runtime-ecs-e0-v1.json`の全hashをE1計画がread-backできる。
- Storage kernel、package、Capability activationをE0完了として偽装していない。
