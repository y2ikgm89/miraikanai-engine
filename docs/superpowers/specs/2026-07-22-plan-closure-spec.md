# Miraikanai Engine 計画書完全化仕様

> **状態: historical input／現行実装正本ではない。** 2026-07-22 review scopeの記録として保持し、current Control Plane／Product契約と矛盾する旧名称・件数を実装へ使わない。

- 状態: ユーザー承認済み
- 承認日: 2026-07-22
- 選択方式: 正本・Registry先行型
- 対象: Architecture Evolution Control Plane、AI-Readable D3D12 Backend、Product Plan、Runtime ECS Contract、および3件の実装計画

## 1. Outcome

計画書だけを読んだ実装AIが、未定義語、暗黙default、未分類ID、依存cycle、測定後に決める合格基準へ遭遇せず、実装、negative test、qualification、package検証まで一方向に実行できる状態を作る。

完了時には次を満たす。

1. Control Plane、D3D12 Backend、Runtime ECS E0に、それぞれ独立して検証可能な実装計画が存在する。
2. Control Plane移行前の43 active specと追加する5正本について、旧依存、新しいdirect `requires`、typed `integrates_with`、相互Contract ID、削除edge、最終topological orderが完全に列挙される。
3. Capability、Product Phase、Target Profile、Requirement、Fixture、Work Packageがclosed registryで参照解決される。
4. Capability activationはTarget別stateを正本とし、集約stateを決定論的に導出する。
5. Capability、Target、Driver、Profile、Fixtureのlogical IDからmaturityとschema／profile versionを除去し、旧IDから新IDへの一回限りの移行表を持つ。
6. D3D12設計はControl Plane後のexact baseline、5新Owner、Package closure、stable diagnostic grammarへ接続される。
7. D3D12 Pipeline keyはpipeline kind、全shader stage、Root Signature、RTV slot順、DSV、sample count、sample quality、全fixed-function stateを閉じる。
8. Enhanced Barrier alias activationで、before-resource barrierがlayout transitionまたはwrite flushを行う場合に`DISCARD`を併用しない規則とnegative fixtureを持つ。
9. Descriptor headroom、HDR／SDR visual tolerance、device recovery、Pipeline Library効果判定を測定開始前に固定する。
10. Product Phase 0～9、2D／3D C2、platformer、puzzle、dialogueのOwner、Work Package、Target、fallback、fixture、Gateが実データ行で閉じる。

## 2. 文書構成

既存設計は責務と判断理由を所有し、実装計画はfile、interface、test、command、expected result、review boundaryを所有する。重複する規則は設計を正本とし、実装計画からexact sectionへ参照する。

更新対象:

- `docs/plans/2026-07-22-architecture-evolution-control-plane-design.md`
- `docs/plans/2026-07-22-ai-readable-d3d12-backend-design.md`
- `docs/architecture/00-product/product-plan.md`
- `docs/architecture/decisions/2026-07-22-runtime-ecs-contract.md`

新規実装計画:

- `docs/plans/2026-07-22-architecture-evolution-control-plane-implementation-plan.md`
- `docs/plans/2026-07-22-d3d12-backend-implementation-plan.md`
- `docs/plans/2026-07-22-runtime-ecs-e0-implementation-plan.md`

## 3. AI可読規則

- 規範Fieldはclosed enum、型、cardinality、ordering、default、failureを同じ定義blockで固定する。
- 未決値marker、後続作業への丸投げ、文脈依存の裁量句、暗黙defaultを禁止する。
- 実測でのみ得られる値は、測定条件と合否式を先に固定し、結果だけをReceiptへ格納する。
- ID、path、command、test名、diagnostic ID、schema majorを完全表記する。
- positive fixtureとnegative fixtureを対にし、fail-open、silent fallback、別Capabilityによる代用を禁止する。
- 計画間handoffはdocument名ではなくexact Git tree hash、schema hash、registry hash、toolchain lock hashでbindする。

## 4. 公式仕様の適用境界

- TypeScript 7.0はstable programmatic APIを持たないため、Architecture lintはcompiler APIへ依存せず、TypeScript 7 CLIでcompileし、正式Artifact生成は`--singleThreaded`を使用する。
- DirectX 12 Agility SDKは2026-07-22時点のstable `Microsoft.Direct3D.D3D12` `1.619.4`、`D3D12SDKVersion=619`を採用候補baselineとし、Toolchain lockでNuGet artifact digestまで固定する。Preview SDKをProduction baselineへ混ぜない。
- D3D12 Pipeline State StreamのPSO identityは`D3D12_PIPELINE_STATE_SUBOBJECT_TYPE`と対応構造をpipeline kind別に閉じ、`DXGI_SAMPLE_DESC.Count`と`Quality`を別Fieldとしてhashへ含める。
- Enhanced BarriersはMicrosoft DirectX-Specsのalias／discard orderingをnegative fixtureまで写す。
- D3D12MAは3.2.0を候補baselineとし、一Device一Allocator、thread-safe既定、budget／statistics、aliasing、`CreateResource3`の境界をToolchain lockとQualificationへ接続する。

外部仕様はAPI制約の根拠であり、Miraikanai固有のOwner、Capability、Package、Gateを外部の公式標準とは表現しない。

## 5. 実行順

1. Control Plane設計と実装計画を閉じる。
2. 43＋5文書のmigration manifestを適用し、lint合格したexact architecture baselineを作る。
3. Runtime ECS E0計画をそのbaselineへbindし、ECS正本追加と既存4文書からのclean migrationを行う。
4. D3D12計画をControl Plane＋ECS後のbaselineへbindし、D3D12正本、Toolchain lock、Windows／Render Graph／Package接続を行う。
5. Product registryと全Qualification Receiptを再検査し、未実装CapabilityをProduction表示しない。

順序を逆転しない。後続計画が前提baseline hashを解決できない場合は開始せず、`diagnostic.architecture.baseline-mismatch`で停止する。

## 6. 監査指摘closure

| 指摘 | Closure authority |
|---|---|
| 3実装計画が存在しない | `docs/plans/2026-07-22-*-implementation-plan.md`の3件 |
| 43＋5文書の依存移行表がない | Control Plane Implementation Plan Appendix A～C。48 node、76 direct edge、29 reciprocal integration |
| Product registry不足 | Product Plan §11.1～§11.7。Target 5、Requirement 16、Fixture 10、Phase 10、Work Package 26、Capability 37 |
| ID migration不足 | Control Plane Implementation Plan Appendix DとD3D12 Implementation Plan Appendix A |
| ActivationのTarget scope不足 | Product Plan §3.2、§11.1、§11.6。`{capability_id,target_id}`が正本 |
| D3D12と5新Owner／Packageが未接続 | D3D12 Design §31～§32、D3D12 Implementation Plan Task 1／12 |
| D3D12 profile／diagnostic ID競合 | D3D12 Design §20.2／§33。旧19 IDをdotted IDへclean replace |
| Pipeline key closure不足 | D3D12 Design §15.2、D3D12 Implementation Plan Task 7 |
| Qualification閾値が測定後決定 | D3D12 Design §29。headroom、visual、recovery、Pipeline Library式を測定前固定 |
| Phase／C2 coverageが実行不能 | Product Plan §11.4～§11.7。Phase 5／9、2D／3D、platformer／puzzle／dialogueを実データ化 |
| Enhanced Barrier aliasing制約不足 | D3D12 Design §16.3、D3D12 Implementation Plan Task 8のnegative fixture |

全指摘は少なくとも一つのpositive fixtureと一つのnegative fixture、またはRegistry orphan／graph lintで検証する。文書追記だけをclosure Evidenceにしない。
