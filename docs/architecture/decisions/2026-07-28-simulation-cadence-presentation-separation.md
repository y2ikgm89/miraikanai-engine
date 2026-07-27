# Miraikanai Engine Simulation Cadence／Presentation Separation Decision

- 文書ID: mirakan.decision.simulation-cadence-presentation-separation
- 状態: review
- 正本範囲: Device reading、Presentation、Simulation Advance、Physics substepを別OwnerのProfile／Intervalとして扱う採用判断
- 非正本範囲: Runtime phase／Schema／固定値、Input Schema、Package／Save／Replay Schema、Subsystem budget、Capability activation、実装Task。各Owner文書を参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Input](../07-platform/input.md)、[Physics](../05-simulation/physics.md)
- 外部根拠検証日: 2026-07-28
- 文書種別: Architecture Decision／runtime timing responsibility
- Decision owner document: `mirakan.arch.runtime-scheduling-lifetime`
- Decision日: 2026-07-28
- Supersedes: none

## 1. Context

表示refresh、PlatformのDevice reading、authoritative Simulation Advance、Physics substepは異なる発生源、authority、failure boundaryを持つ。これらを一つのFPSまたはdeltaへ統合すると、表示頻度からGameplayの進行を推測する、raw readingをCommandとして扱う、Physics内部stepごとにInputやeventを増やす、というauthority逆流が生じる。

現行Owner文書はすでに、Inputの`T10_InputLatch`、Schedulingのsealed `SimulationAdvanceIntervalV1`、Presentation-only state、Physicsのouter advance内substepを定義している。本Decisionはその責務分離を採用理由として記録し、Schema、Profile instance、rate、Target Qualificationを再定義しない。

## 2. Decision drivers

1. Device callback／pollの頻度とauthoritative Command受理回数を分離する。
2. Presentation refreshとWorld、Timer、Save、Replayの進行を分離する。
3. 全authoritative consumerが一つのsealed Simulation intervalを参照する。
4. Physics精度の内部partitionがSimulationの外側のboundaryを増やさないようにする。
5. high-refresh Presentationとalternate Simulation cadenceを独立に評価できるようにする。

## 3. Considered options

### 3.1 表示refreshを共通Engine tickとして使う

不採用とする。surface状態、VSync、負荷、Target差でauthoritative進行が変わり、headless、Save、Replay、same-input traceの意味を閉じられない。

### 3.2 Device readingまたはPhysics substepごとにGameplayを進める

不採用とする。raw readingがContext／consumption／latchを迂回し、substepがInput、Timer、event、publicationの追加boundaryになってouter Simulation Advanceの一意性を失う。

### 3.3 各周期をOwner別のtyped boundaryとして接続する

採用する。Input OwnerがDevice readingからimmutable `InputSnapshot`を作り、Scheduling Ownerがsealed `SimulationAdvanceIntervalV1`でauthoritative進行を所有し、Physics Ownerはその一intervalを内部partitionする。Presentationはpublished snapshotだけを消費するnon-authoritative childとする。

## 4. Decision

Device readingはInput Adapterが生成する入力候補であり、consumed authoritative Commandではない。readingはbounded queue、timestamp、sequence、Context、focus、consumptionを経て、`T10_InputLatch`で一つのSimulation Advanceに対応するimmutable `InputSnapshot`へsealされた後だけGameplayへ届く。Presentation用の観測やpreviewが同じreadingを参照しても、そのreadingをT10で受理済みまたは消費済みとみなさない。

Presentationはnon-authoritativeである。published Simulation snapshot、Presentation command、monotonic presentation timeからRender、Camera、UI、Audio、VFX向け結果を作れるが、World、Save、Replay、Gameplay digestを変更しない。surface unavailable、frame drop、異なるdisplay refreshはauthoritative Simulation Advanceの生成数または結果を決めない。

SimulationはScheduling Ownerが生成してsealしたcurrent `SimulationAdvanceIntervalV1`だけをtimebaseとして使う。consumerはrender frame、wall time、表示Hz、Genre、直前deltaからlogical durationを補完しない。Cadence Profileの選択、current instance、alternate branch、Qualificationは[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)が所有する。

Physics substepは一つのouter Simulation Advanceの子である。outer intervalのlogical durationをqualified Profileに従ってexactにpartitionし、substepごとに新しいInput sample、Gameplay Timer、event delivery、Replay checkpoint、World publication boundaryを生成しない。全substepが完了してからouter advanceのcanonical integration／publicationへ一回だけ進む。

high-refresh Presentationとalternate Simulation cadenceは別々のfuture evaluation subjectである。表示refresh、Input device report rate、Physics substep countからSimulation rate、Capability state、Target Qualificationを推測せず、一方の評価または成功を他方の採用根拠にしない。

本Decisionは`review`であり、`contract_activation_effect = none`である。新しいProfile、rate、Schema、Capability、MCD、Operation、Work Package、Qualification Receiptを追加または有効化しない。

## 5. Consequences

### 5.1 Positive

- display、Device、Simulation、Physicsの周期差がGameplay結果へ暗黙に流入しない。
- Input／Save／Replay／Physicsが同じsealed Simulation intervalへ閉じる。
- high-refresh Presentationを検討してもauthoritative cadenceを同時変更する必要がない。
- Physics substepの精度改善とGameplay boundaryの一意性を両立できる。

### 5.2 Costs and risks

- Presentation previewと次のauthoritative acceptは異なる時点になり得るため、別metricで観測する必要がある。
- cadenceまたはsubstepを追加する場合、全consumer、Save／Replay、Target Qualificationのclosureが必要になる。
- Owner間でProfile ref、interval hash、advance sequenceのbyte equalityを維持する必要がある。

## 6. Canonical Owner documents

| Concern | Canonical Owner |
|---|---|
| Device reading、Context／consumption、T10 latch、InputSnapshot | [Input](../07-platform/input.md) |
| Cadence Profile、Simulation interval、phase、publication、Presentation boundary | [Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md) |
| outer intervalのPhysics substep partitionとQualification | [Physics](../05-simulation/physics.md) |
| Product Capability、maturity、Work Package | [Product Plan](../00-product/product-plan.md) |
| Save／Replay timebase binding | [Persistence／Save](../04-runtime/persistence-save.md) |
| cadence／presentation capacityとTarget測定 | [Performance／Capacity](../04-runtime/performance-capacity.md) |

本Decisionは各Ownerのcurrent Schema、fixed value、runtime behaviorを置き換えない。

## 7. Supersedes／Superseded by

- Supersedes: none
- Superseded by: none

## 8. Official or primary sources

- [Unity 6.5 Per-frame updates](https://docs.unity3d.com/6000.5/Documentation/Manual/time-per-frame-updates.html)および[Fixed updates](https://docs.unity3d.com/6000.5/Documentation/Manual/fixed-updates.html): frame updateとfixed physics updateが異なる周期を持つことの比較根拠。
- [Unreal Engine 5.8 Physics Sub-Stepping](https://dev.epicgames.com/documentation/en-us/unreal-engine/physics-sub-stepping-in-unreal-engine): full physics stepの内部partitionとcallback集約の比較根拠。
- [Godot stable Idle and Physics Processing](https://docs.godotengine.org/en/stable/tutorials/scripting/idle_and_physics_processing.html)および[Physics Interpolation Introduction](https://docs.godotengine.org/en/stable/tutorials/physics/interpolation/physics_interpolation_introduction.html): rendered frameとfixed physics tickの分離に関する比較根拠。

これらは責務分離の比較材料であり、外部EngineのAPI、既定rate、delta型、callback順、substep方式をMiraikanai Contractへ移植する根拠ではない。
