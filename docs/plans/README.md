# Miraikanai Engine 計画書の実行権限・準備状況

- 状態: current 計画の読取り入口。Product／Control Plane の正本や承認を置換しない。
- 更新日: 2026-07-24
- 対象: `docs/plans/**` と `docs/superpowers/plans/**` の全14文書
- 実装状態: Engine、Control Plane、Gateway、Signer、Runtime、Target Qualification、Capability Activation、Release／Shipping は未実装・未主張。

## 1. 読み方と共通の実行入口

この一覧はナビゲーションであり、実行権限ではない。実行可否、Work Package（WP）の状態、Capability Activation、Release、Shipping は、次の current signed record だけから判定する。

1. [Product Plan](../architecture/00-product/product-plan.md) の current Active Definition と operational snapshot。
2. [Control Plane Design](2026-07-22-architecture-evolution-control-plane-design.md) の `CurrentControlPlaneBaselineBindingV1`、Trust、revocation、Task Authorization。
3. 対象 Owner 文書、exact Toolchain lock、Target／device／runner の Qualification、同一 Staging candidate に束縛された Evidence。

Markdown の checkbox、ファイル名、branch 名、`latest`、過去の Bootstrap Approval は current authority ではない。各 current 計画は、外部 Scheduler が signed snapshot から lifecycle を `declared -> ready -> active` へ正当に遷移させ、必要な人間 Approval と Task Authorization を検証した後だけ開始できる。計画自身は WP を昇格・完了、Capability を activate、Release／Shipping を承認しない。

baseline-scoped な Repository bytes を変更する作業は、isolated Staging candidate、preliminary Evidence、人間 Approval、same-definition Rebaseline、new current binding に対する final Receipt の順で進める。実装済みや製品利用可能を主張するには、さらに current Product snapshot 上の完了／Activation／Target Qualification を read-back する。

## 2. 公式根拠と採用境界

外部資料は技術制約の根拠であり、採用や更新を自動承認しない。exact version、artifact hash、license、取得元、更新 Gate は常に [Toolchain／Dependencies](../architecture/02-foundation/toolchain-dependencies.md) を唯一の正本とする。

| 対象 | 公式根拠 | 計画への適用 |
|---|---|---|
| C++ Modules／`import std` | [CMake C++ Modules](https://cmake.org/cmake/help/latest/manual/cmake-cxxmodules.7.html)、[MSVC `import std`](https://learn.microsoft.com/en-us/cpp/cpp/tutorial-import-stl-named-module) | CMake／generator／MSVC の条件を満たす probe だけに限定し、CX0 production pathへ experimental module を混在させない。 |
| Direct3D 12 Agility | [DirectX 12 Agility SDK](https://devblogs.microsoft.com/directx/directx12agility/) | app-local package layout と exact SDK lock を検証する。WARP と Target minimum OS が未固定の間は、それぞれ D3D12 Task 3 と package qualification を開始しない。 |
| JSON Schema／Ajv | [Ajv JSON Schema docs](https://ajv.js.org/json-schema.html)、[strict mode](https://ajv.js.org/strict-mode.html) | Control Plane lint は Draft 2020-12 と lock 済み `ajv/dist/2020` の strict profile に限る。Runtime validator の採用判断やネットワーク `$ref` を含意しない。 |
| TLA+／TLC | [TLA+ tools](https://github.com/tlaplus/tlaplus) | ECS の model check は Toolchain 正本の exact TLC jar と Java runner だけで実行する。任意 PATH 上の Java、別 jar、未検証 plugin を使わない。 |

公式資料の改訂、artifact replacement、license／support 条件の変化は Evidence を stale にする。実装開始前または更新時に、Toolchain ChangeSet の current lock と公式一次資料を再照合する。

## 3. Current 計画（実行権限ではない）

| 文書 | 役割 | 現在の入口・境界 |
|---|---|---|
| [AI-Readable D3D12 Backend Design](2026-07-22-ai-readable-d3d12-backend-design.md) | D3D12 の詳細設計レビュー | Design review のみ。実装は D3D12 implementation plan の D0／Task Authorization 境界に従う。 |
| [Control Plane Design](2026-07-22-architecture-evolution-control-plane-design.md) | Control Plane の設計正本候補 | `ControlPlaneConstructionAuthorizationV1` だけが construction 着手権限を与える。既存 `development_bootstrap`／`production` assurance profile を変更しない。 |
| [Control Plane Implementation Plan](2026-07-22-architecture-evolution-control-plane-implementation-plan.md) | Control Plane construction 手順 | 外部 Authorization、clean authorized tree、signed inventory が必要。Task 10B 後に CP WP を再び変更しない。 |
| [D3D12 Backend Implementation Plan](2026-07-22-d3d12-backend-implementation-plan.md) | D3D12 実装手順 | D0 は限定された document ChangeSet のみ。Task 2–13 は current technical-document Approval、active WP、Task Authorization、exact Toolchain が必須。 |
| [Runtime ECS E0 Plan](2026-07-22-runtime-ecs-e0-implementation-plan.md) | ECS E0 contract baseline | E0 は current baseline と active WP が必要。新 ECS technical document の human approval は E1 以降の入口であり、E0 の開始前に推測して要求しない。 |
| [Critical Path](2026-07-23-critical-path-execution-plan.md) | Product WP の優先順 consumer | Wave は優先順であり state machine ではない。WP の実行可否は Product snapshot と Owner／Authorization が決める。 |
| [Future Capability Inception](2026-07-23-future-capability-inception-plan.md) | Future Portfolio の調査・昇格手順 | planning-only。prototype／promotion は Future approval、Owner Decision、Toolchain／Target evidence を満たすまで行わない。 |
| [Generic AI-Native Plan Update](../superpowers/plans/2026-07-23-generic-ai-native-engine-plan-update.md) | 文書契約 remediation の作業記録 | partial。Engine implementation、compiler、Gateway、Activation、Release の実行権限ではない。 |

## 4. 解消済みの読み違い

| 論点 | 決定した読取り |
|---|---|
| Control Plane の development／production | 設計 §の既存 `assurance_profile=development_bootstrap` と `production` をそのまま使う。single-pointer CAS や trust 要件の削除を新設しない。 |
| CP 完了後の Wave 0 | Architecture registry／compiler／record store／validator closure は CP Task 6 の pre-publication output である。Task 10B で `wp.architecture.control-plane` が complete になった後は、Wave 0 はそれを read-only で再計算し sidecar Report を発行するだけで、WP や Product snapshot を変更しない。 |
| ECS E0 と E1 | E0 entry は current baseline／active WP と既存 Owner approval で判定する。E0 Task 11 の human approval と Rebaseline が完了して初めて E1–E3 に進める。 |
| MVP-A Thin Slice | 別の activation allowlist や別計画を作らない。Product Plan の `AiCapabilityActivationMatrixV1` と `ExecutionSurfaceBindingV1` を唯一の Activation authority とし、Thin Slice は Wave 4 の全8 family atomic activation 後の qualification witness とする。 |
| Eval の 95% | [AI Verification／Provenance](../architecture/01-governance/ai-verification-provenance.md) が所有する production elevation の条件であり、MVP-A entry 条件へ勝手に前倒ししない。 |

## 5. Historical／superseded（実行禁止）

次の本文は証跡であり、書かれた Task、checkbox、件数、旧 schema 名を current 作業へ使わない。履歴不変性を守るため本文は改変しない。

| 文書 | 状態 |
|---|---|
| [AI-Readable Architecture Comprehension Closure](2026-07-21-ai-readable-architecture-comprehension-closure.md) | historical completion record |
| [Legacy Branch Reconciliation Design](2026-07-22-legacy-branch-reconciliation-design.md) | historical／superseded |
| [Legacy Branch Reconciliation Disposition](2026-07-22-legacy-branch-reconciliation-disposition.md) | immutable historical ledger |
| [Legacy Branch Reconciliation Implementation Plan](2026-07-22-legacy-branch-reconciliation-implementation-plan.md) | completed historical plan |
| [Plan Review Closure](../superpowers/plans/2026-07-22-plan-review-closure.md) | completed historical review plan |
| [Plan Feasibility Remediation](../superpowers/plans/2026-07-23-plan-feasibility-remediation.md) | superseded historical remediation plan |

旧 `WP26`／`Capability37`／`Future17` 等の件数は historical evidence であり、current execution input ではない。current の exact closure は実行時に Product Plan の signed Definition から再生成・照合する。
