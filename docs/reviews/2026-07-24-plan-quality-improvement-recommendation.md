# Miraikanai Engine 計画品質改善 Recommendation（2026-07-24）

- document_id: `mirakan.review.plan-quality-improvement-2026-07-24`
- status: **accepted audit input; B′ amendment applied**（2026-07-24。正本を上書きせず、矛盾する原案は下記 amendment で失効）
- purpose: 計画書コーパスの実現可能性精査結果と、計画品質を上げるための具体的 remediation 方針を、独立AI／人間が監査できる形で固定する
- decision_applied: **B′ — planning quality remediation only / implementation_and_shipping_no_go**
- recommended_approach: **B′ — existing Control Plane profile・Product Activation authorityを保持し、公式根拠・実行入口・承認境界を全current計画へ明記する**
- authority（読取専用）:
  - [Product Plan](../architecture/00-product/product-plan.md)
  - [Architecture Index](../architecture/README.md)
  - [Control Plane Design](../plans/2026-07-22-architecture-evolution-control-plane-design.md)
  - [Control Plane Implementation Plan](../plans/2026-07-22-architecture-evolution-control-plane-implementation-plan.md)
  - [Critical Path Execution Plan](../plans/2026-07-23-critical-path-execution-plan.md)
  - [Future Capability Inception Plan](../plans/2026-07-23-future-capability-inception-plan.md)
  - [Generic AI-Native Architecture Design](../superpowers/specs/2026-07-23-generic-ai-native-engine-architecture-design.md)
  - [Generic AI-Native Plan Update](../superpowers/plans/2026-07-23-generic-ai-native-engine-plan-update.md)
  - [Plan Feasibility Remediation Review](2026-07-23-plan-feasibility-remediation.md)
  - [計画書の実行権限・準備状況](../plans/README.md)
- non_authority: 本 Recommendation は Product Definition、Capability Activation、Release、Shipping、Engine binary の完了を主張しない。件数・ID・schema の正本は Architecture／Product／Control Plane 側にある。新規 implementation plan、single-pointer trust profile、独自 Activation allowlist を作らない。

## 0. B′ 承認済み適用と原案の置換（2026-07-24）

ユーザーは「公式根拠・実行入口・承認境界を全計画へ反映し、既知の矛盾を直す」B′を承認した。本節は原案の R02–R07、R10 と、§5.2／§7／§11 のうち以下と衝突する記述に優先する。Engine 実装の開始や新規 implementation plan の作成を承認するものではない。

| 原案の論点 | B′ の確定処置 |
|---|---|
| R02 の新しい Dev／Production 二段化 | [Control Plane Design](../plans/2026-07-22-architecture-evolution-control-plane-design.md) の既存 `assurance_profile=development_bootstrap`／`production` を保持する。single-pointer CAS、trust closure、1-of-1 開発Rootの新設提案は採用しない。 |
| R03 の新規 Thin Slice 実装計画 | 作成しない。Thin Slice は current [Product Plan](../architecture/00-product/product-plan.md) の `AiCapabilityActivationMatrixV1` と `ExecutionSurfaceBindingV1` を read-back する Wave 4 qualification witness とする。8 family の一部だけを別allowlistで activate しない。 |
| R04 の Wave 0.5 分割 | `wp.architecture.control-plane` 完了後の materialize を禁止する。Registry／compiler／record store／validator closure は Control Plane Task 6 の pre-publication output、Wave 0 は read-only sidecar Report の再計算とする。 |
| R05 の独自 MVP callable allowlist | Product Matrix が唯一の Activation authority。別表は作らず、必要なら Matrix から read-only execution view を生成する。 |
| R06 の 95% Eval 段階化 | [AI Verification／Provenance](../architecture/01-governance/ai-verification-provenance.md) の production elevation 条件を変更しない。95% を MVP-A entry へ前倒しも、既存正本から削除も行わない。 |
| R07 の ECS blocking／descope 選択 | E0 は既存 current baseline／active WP条件で開始候補。E0 Task 11 の human approval と Rebaseline 完了後だけ E1–E3 が開始候補となる。E0 を新規 ECS文書 approval で事前 block しない。 |
| historical 文書 | 6文書は immutable evidence として本文を変更せず、[計画書の実行権限・準備状況](../plans/README.md) で全件を実行禁止として索引する。 |

公式資料は exact adoption の代替ではない。CMake、Microsoft／DirectX、Ajv、TLA+ の適用制約と再検証入口は[計画書の実行権限・準備状況](../plans/README.md)に、version／hash／license は[Toolchain／Dependencies](../architecture/02-foundation/toolchain-dependencies.md)に一意に置く。

## 0.1 監査者への指示（必読）

独立監査者（人間またはAI）は、本文書を「採用前提の結論」として扱わず、次を検証すること。

1. **反証可能な主張だけを採否する。** 「成熟している」「重い」などの形容は、§3 の数値・path・節番号へ辿れる場合だけ採用する。
2. **正本と矛盾する箇所を優先して指摘する。** 本 Recommendation と Product Plan／Critical Path／Executable Contracts が衝突する場合、正本を優先し、本文書を defect として記録する。
3. **件数は current tree を再抽出して照合する。** §2 の数値は 2026-07-24 時点の文書監査値である。差分があれば stale claim として報告する。
4. **実装完了を文書完了と取り違えない。** `document audit pass` は Engine／Gateway／Signer／device Qualification の証拠ではない。
5. **監査出力フォーマット**は §12 に従う。

---

## 1. 結論（Executive verdict）

### 1.1 製品意図との整合

Miraikanai Engine の製品意図は一貫している。

- Chat 付加型ではなく、人間とAIが同じ型付き Authoring 経路を使う AI-native Engine／Projection Editor である（Product Plan §1）。
- AI は Proposal まで。Approval／Commit／Activation／Signing／Release は人間と Engine Gateway の権威である。
- MVP-A は「AI＋手動の安全な往復を 2D Shooter vertical slice で証明する」ことであり、新規 Project C++／Shader 生成を成功条件にしない（Product Plan §5）。

この意図自体は実現可能かつ健全である。問題は意図ではなく、**計画コーパスが「安全な憲法」としては厚い一方、「実行可能な立法・裁判所・工場」としては未稼働**であることである。

### 1.2 Go / No-Go

| 判定 | 結果 | 意味 |
|---|---|---|
| Planning Go | **Yes（条件付き）** | Control Plane → WP 実行の入力として現行計画を使える。ただし §6 の quality remediation を先に入れることを強く推奨する |
| Implementation Go | **No** | binary、Schema compiler、Gateway、Signer、Activation Receipt が無い |
| Shipping Go | **No** | Release Gate、critical Risk closure、Target production Activation 未成立 |
| Future Claim Go | **No** | Future 25 を利用可能 Capability として表示してはならない |
| AI can make games today | **No** | operational Operation = 0、Host materialization = 0 |

条件付きの意味: 現行計画は実装着手の禁止ではない。しかし **ceremony 過大・Thin Slice 不在・Authority 混在**のまま実装に入ると、永遠に planning／governance ループに閉じるか、正本を無視した場当たり実装に分岐するリスクが高い。

### 1.3 実現可能性スコア（仮定付き）

| ID | 観点 | Score | 仮定 | 解釈 |
|---|---|---:|---|---|
| S1 | 計画完全性 | **72 / 100** | Markdown正本・Registry・権限モデルを「計画」と数える | 契約・分離・fail-closed は強い。Eval golden／チーム容量／Thin Slice が弱い |
| S2 | 典型小チームでの MVP-A 納品 | **22 / 100** | 3–6 FTE、フルタイム、既存Engine流用なし、現行契約を大幅に削らない | Phase 0–3 が巨大。Wave 4 で初めて AI。`team_assumption_state=unfixed` |
| S3 | 「AIが広くゲームを作る」ビジョン | **12 / 100** | 多ジャンル・商用Asset生成・Runtime生成・多Target を含む | Future／deny-only／C2 以降に意図的に隔離されている |

スコアは主観を含む。監査者は §1.3 の仮定を変えずに再採点するか、仮定変更を明示して別スコアを出すこと。

---

## 2. 監査対象コーパスと current closure

### 2.1 対象範囲

| 区分 | path | 扱い |
|---|---|---|
| Architecture 正本 | `docs/architecture/**`（約45正本 + Decision） | 製品契約の正本候補。状態は Index 上すべて `review` |
| 実行計画 | `docs/plans/**` | Control Plane / Critical Path / Future / D3D12 / ECS 等 |
| Superpowers | `docs/superpowers/specs/**`, `docs/superpowers/plans/**` | Generic AI-Native、過去 remediation（一部 superseded） |
| Reviews | `docs/reviews/**` | 本系統の監査記録。正本ではない |

### 2.2 現行実行正本クラスタ（読むべき）

次だけを「現在の実行入力」とみなす。

1. Product Plan + Governance（AI Security / Verification）
2. Control Plane Design + Implementation Plan
3. Critical Path Execution Plan
4. Future Capability Inception Plan
5. Generic AI-Native Architecture Design + Plan Update
6. 各 Subsystem Owner 文書（変更対象 Owner のみ）

### 2.3 historical / superseded（実行に使わない）

少なくとも次は historical または superseded である。banner があっても件数表を実行入力に使ってはならない。

| path | 状態 | 注意 |
|---|---|---|
| `docs/superpowers/specs/2026-07-22-plan-closure-spec.md` | historical | 旧件数（WP26 / Cap37 等） |
| `docs/superpowers/specs/2026-07-23-plan-feasibility-remediation-design.md` | superseded | Future 17 等の旧 baseline |
| `docs/superpowers/plans/2026-07-23-plan-feasibility-remediation.md` | superseded | 実行禁止 |
| `docs/plans/2026-07-21-ai-readable-architecture-comprehension-closure.md` | historical complete | 再実行しない |
| `docs/plans/2026-07-22-legacy-branch-reconciliation-*.md` | historical | disposition ledger 含む |
| `docs/superpowers/plans/2026-07-22-plan-review-closure.md` | historical | 旧 path `08-domain-packs` 残存の可能性 |

### 2.4 Current exact closure（2026-07-24 document audit）

出典の一次記録: [Plan Feasibility Remediation Review §2 / §7.3](2026-07-23-plan-feasibility-remediation.md)。

| Domain | Current invariant |
|---|---|
| Active Product Definition Registry | 14 |
| Target | 5 |
| Requirement | 29 |
| Fixture | 15 |
| Fallback | 6 |
| Phase | 10 |
| Phase Gate | 22 |
| Release Gate | 5 |
| Decision Gate | 4 |
| Work Package | 75 |
| Capability | 102 |
| Activation binding | 293（required 265 + optional 28） |
| Risk | 9 |
| Future capability | 25（すべて `planning_only`） |
| Future claim | 57 |
| Contract-active Operation | 10（Core baseline 6 + World 1 + Shooter 3） |
| Operational Operation | **0**（Signer Policy `entries=[]`） |
| Planned Operation family / candidates | 24 / 191（すべて `not_activated`） |
| AI Host materialized / callable | **0 / false**（standard MCP, Engine Provider, managed Host） |

Critical Path coverage: `bootstrap completed 1 + unconditional 70 + conditional deferred 4 = 75`。

---

## 3. AI-native 制作経路の成熟度

目標旅程（Product Plan §1, §5）:

```text
自然言語
  -> Blocking/High 質問
  -> Game Brief 承認
  -> GameSpec / 実装計画
  -> typed ChangeSet（Definition-first / prequalified Pack）
  -> Human Approval
  -> Engine Gateway Commit
  -> Cook / Play / Title→Result
  -> Save/Load/Replay
  -> AI↔手動往復（同じ Project history）
```

| 段 | 成熟度 | 状態 | 根拠 |
|---|---|---|---|
| NL → Intent / 質問 | 部分 | Schema・UXあり。discovery/authoring Op 未Activation | Product Plan §1、editor-workspace-ux §8 |
| Game Brief 承認 | 契約ほぼ完全 | 実行経路なし | Generic Design §7、ai-security-approval |
| Brief → Spec / ChangeSet | 部分 | 型は厚い。callable 空 | project-state §5、Critical Path §7 |
| Approval → Commit | 規則は厚い | Gateway 未実装 | Critical Path §7 |
| Cook / Play / Title→Result | MVP定義は完全 | Phase 0–3 前提 | Product Plan §5–6 |
| AI↔手動往復（MVP-A） | UX規則は厚い | Wave 4 以降 | Critical Path Wave 4 |
| 新規 Native / Shader | 明示的に MVP-A 外 | Phase 5 | Product Plan §5 |

**刺す一文:** ドキュメントは AI-native Engine の憲法としては強い。**「来週 AI で 2D shooter を作る計画」としてはまだ憲法段階**であり、立法（callable Operation）、裁判所（Gateway / Signer）、工場（Editor / Runtime / Pack）は未稼働である。

---

## 4. 計画品質の欠陥一覧（defect-first）

各 defect は次の形を持つ。

- ID
- Severity: `P0`（実装入口または製品主張を壊す） / `P1`（高確率で誤読・停滞） / `P2`（品質・運用）
- Evidence: path と節
- Why it matters
- Proposed fix（§6 へ接続）

### D01 — Control Plane ceremony が全後続の開始条件（P0）

- Evidence: Control Plane Implementation Plan（BC0–Task10B）、Critical Path Wave 0–1、Feasibility Review §5–6
- Why: Root / attestation / CRL・OCSP / multi-pointer recovery 等を Phase 0 前に要求する読みができる。ゲームループ以前に PKI＋governance OS 規模となり、後続 WP の着手を窒息させる。
- Proposed fix: R02（Dev bootstrap / Production trust 二段化）

### D02 — 「AIが作る」証明が Wave 4 まで届かない（P0）

- Evidence: Critical Path Wave 0–4、Product Plan §5.1 critical_path、§6 Phase 0–4
- Why: AI Authoring は手動 2D First Playable 完了後。最小「Prompt → Playable Candidate」縦スライス専用計画が無い。製品ビジョンの中核が実行計画上は後付けになる。
- Proposed fix: R03（AI First Playable Thin Slice 計画）

### D03 — callable surface = 0 のまま詳細契約が製品完成形に見える（P0）

- Evidence: Executable Contracts §§20–21、Feasibility Review §2–3、Architecture Index「全 review」
- Why: contract-active 10 も Signer empty で operational 0。191 candidates / 24 families は予約。読者・エージェントが「実装可能仕様」と誤読しやすい。
- Proposed fix: R01（Authority Map + maturity banner）、R05（MVP callable allowlist）

### D04 — Authority / 件数ドリフトと historical 混在（P1）

- Evidence: `docs/plans/` と `docs/superpowers/` に historical / superseded / executable draft が同居。旧 WP26 / Cap37 / Future17 が残存。
- Why: エージェントが旧件数を実行入力に使う事故面が大きい。`docs/plans/README.md` が無い。
- Proposed fix: R01

### D05 — Wave 0 循環依存（P1）

- Evidence: Critical Path §5 Wave 0（Owner 103 / Source・Edge 516 / Conformance Report pass 要求）
- Why: 依存グラフ生成自体が Control Plane / Document Registry 成果物である可能性が高く、bootstrap 完了前に合格条件が自己参照しやすい。生成器未実装時は紙上 Gate。
- Proposed fix: R04（Wave0 縮小、Conformance を Wave0.5 へ）

### D06 — 未作成正本への critical path 依存（P1）

- Evidence: Product Plan §5.1（ECS E0）、runtime-ecs-contract Decision（ユーザー確認待ち）、参照されるが未作成の Owner 文書（entity-component-system / Save / architecture-governance 等）
- Why: 参照先不在のまま critical path が語られると、実装計画が開始不能または場当たりになる。
- Proposed fix: R07（正本 blocking 化または critical path 外し）

### D07 — チーム仮定 unfixed と相対 Wave の擬似スケジュール（P1）

- Evidence: Product Plan §5.1 `team_assumption_state=unfixed`、Critical Path Wave 0–9
- Why: 人数・device lab・R4 approver を推測しない方針は健全だが、実行不能タスクが並ぶのに「Wave」が直線マイルストーンに見える。進捗判定が不能。
- Proposed fix: R08（assumed planning revision）、R01（Wave は優先順であり state ではないと再掲）

### D08 — Eval / Verification の過大先行（P2）

- Evidence: ai-verification-provenance（V0–V9、holdout、95% 下限、architecture comprehension 大量 case 等）
- Why: 長期的には正しいが、MVP-A 前の必須セットとしては過負荷。紙上 Gate が増える。
- Proposed fix: R06（Eval 段階化: MVP必須 / C2 / Release）

### D09 — 精密さによる実行不能（P2）

- Evidence: Control Plane Design（極大）、Implementation Plan の超密 Task、Generic Design の schema 群
- Why: 精密さ ≠ 実行可能性。「今週何をするか」を抽出できない。独立監査も困難。
- Proposed fix: R02（最小 DoD カード）、R03（Thin Slice の短い計画）

### D10 — D3D12 / ECS 計画の「executable」ラベル誤解（P2）

- Evidence: D3D12 implementation plan「executable draft」、ECS E0 plan（Control Plane baseline 依存）
- Why: Preflight 未充足でも実行可能と読める。
- Proposed fix: R01（ラベルを `blocked on …` に修正する作業項目）

### D11 — Generic plan update の二重真実（P2）

- Evidence: Generic plan update Execution Record（Task2 未 final）vs architecture 側の大量 remediating
- Why: checkbox 未完了と正本差分済みが並立し、完了判定が再帰的。
- Proposed fix: R01、R10（件数単一生成源 / 完了Gate分割）

### D12 — Asset / Pack reference が AI 視覚成果の隠れた前提（P1）

- Evidence: asset-lifecycle、Product Plan MVP（prequalified Pack）、commercial asset generation は Future
- Why: AI が「見た目付きゲーム」を作るには reference Pack 実装が必須だが、Pack 実装も未着手。計画上の優先度が製品ビジョンに対して弱い。
- Proposed fix: R03 の入口条件に prequalified Pack fixture を明示

---

## 5. 検討した改善方針（2–3 approaches）

### 5.1 方針 A — 憲法維持・索引整備（最小変更）

内容:

- `docs/plans/README.md` に Current Authority Map を置く
- historical / superseded の Do-not-use 一覧と旧件数禁止表
- 各正本先頭に maturity banner（document review / contract-active N / operational 0 / implementation absent）
- 契約本文・Wave・ceremony は触らない

長所: 低リスク、誤読防止に即効。
短所: MVP 到達距離は縮まらない。D01/D02 は残る。

採用条件: まず監査誤読だけ止め、契約改訂は別 ChangeSet にする場合。

### 5.2 方針 B（原案、B′で置換） — 二段化 + Thin Slice

内容:

- 既存の `development_bootstrap`／`production` assurance profile を保持し、実行入口を current Authorization と signed snapshot に明記する
- 新規 Thin Slice 計画を作らず、Product Matrixから導く Wave 4 qualification witness とする
- Wave 0 の post-complete materialize を Task 6 pre-publication output と read-only sidecar Reportへ置換する
- MVP callable authority を `AiCapabilityActivationMatrixV1` に一意化し、別allowlistを作らない
- Eval / HITL の既存正本を維持し、95% production elevationをMVP entryへ誤って前倒ししない
- ECS E0 entryとE1以降のhuman-approval／Rebaseline entryを分離する
- team_assumption を `assumed` にする別 planning revision（人数を推測せず、ユーザー入力で固定）

長所: ビジョンと fail-closed を壊さず、実装入口と falsifiable な AI 証明を開ける。
短所: 計画改訂の作業量は中。Rebaseline が必要。

### 5.3 方針 C — MVP 契約圧縮（大胆削減）

内容:

- C1 最小契約以外を Future または deferred appendix へ
- Wave 4 の 8 family atomic を subset allowlist に削減
- Rendering / VFX / Eval 95% を後段へ
- 「手動2D + 最小 AI ChangeSet 往復」を唯一の最初の成功条件にする

長所: 小チーム実現性（S2）が上がる。
短所: 既存正本の大規模 rebaseline。Active/Future set equality の再計算が必要。過去 remediation との差分説明コストが大きい。

### 5.4 採用判定

**本 Recommendation は B′ を適用する。**
方針 A の Authority Map は B′に内包する。方針 C は、ユーザーが明示的に「契約圧縮」を承認した場合のみ、別の Product Definition migration として検討する。

---

## 6. 推奨 Remediation（B′の下での原案照合）

各 remediation は独立に監査・実装できる単位とする。完了条件は falsifiable であること。

### R01 — Current Authority Map と maturity 表示

**成果物:**

1. `docs/plans/README.md`（新規）
2. 必要なら `docs/superpowers/README.md`（historical 索引）
3. 主要正本先頭 banner テンプレート

**README 必須節:**

```text
1. Current executable authorities（path + role）
2. Historical / superseded（do not execute）
3. Do-not-use stale counts（旧件数表）
4. How to read Wave / checkbox（state authority ではない）
5. Link to Product Plan Activation matrix
```

**正本 banner 最小 Field:**

```text
document_status: review | approved | historical | superseded
implementation_status: absent | partial | present
contract_active_operations: <n or n/a>
operational_operations: <n or n/a>
capability_activation: not_activated | mixed | see Product snapshot
claims_forbidden:
  - do_not_claim_shipping
  - do_not_claim_ai_callable_without_activation
```

**完了条件（DoD）:**

- [ ] `docs/plans/README.md` が存在し、§2.2 / §2.3 と set equality で矛盾しない
- [ ] 旧件数（WP26/Cap37/Future17 等）が Do-not-use に列挙される
- [ ] D3D12 / ECS 計画の表紙ラベルが `blocked on Control Plane baseline + WP active` を明示する
- [ ] 独立監査者が「今日実行してよい計画はどれか」を README だけで答えられる

### R02 — Control Plane assurance profile と construction 入口

B′は既存の `assurance_profile=development_bootstrap`／`production` を変更しない。development profile は production release、managed external Host の実行コード受入れ、Production Credential を許可しない。一方、production profile の six-pointer trust／recovery 条件を MVP-A から恣意的に外さない。

construction の唯一の入口は completed `ControlPlaneConstructionAuthorizationV1`、authorized base tree、current signed inventory、current baseline／Trust／revocation の read-back である。profile 名、plan checkbox、過去 Bootstrap Approval、単一 pointer CAS 提案を入口に使わない。

### R03 — MVP-A Thin Slice の qualification witness

新規 implementation plan、独自 allowlist、独自 Product state は作らない。current Product Plan の `AiCapabilityActivationMatrixV1` と `ExecutionSurfaceBindingV1` が8 familyの唯一の Activation authority である。

Thin Slice は、8 family が各family単位で atomic activation され、current operational read-back が成功した**後**に、同一Candidateの fixed prompt、typed ChangeSet Proposal、人間 Approval、trusted-internal Commit、manual edit、play oracle を束縛する Qualification witness として実行する。AI は Approval、Commit、Activation、Signing、Release を行わない。新規 Project C++／Shader、external MCP Production routing、95% production-elevation Gateをこの witness の入口に追加しない。

### R04 — Wave 0 の自己参照排除

Control Plane Task 6 は、Registry、Dependency Source Set／Edge Registry compiler、append-only Conformance Record Store、validator Signer／Key／revocation closureを publication 前に output として完成させる。Task 10B が `wp.architecture.control-plane` を complete にした後、Critical Path Wave 0 はそのcompleted artifactとcurrent snapshotを read-only で再計算し、sidecar `ArchitectureDependencyConformanceReportV1` を発行する。

この Report は Wave 1 のreadiness evidenceに限定し、completed WP、Product snapshot、Activation、Operation Authority を変更しない。Wave 0.5 や post-complete materialization は作らない。

### R05 — MVP callable authority の一意化

`AiCapabilityActivationMatrixV1` と `ExecutionSurfaceBindingV1` を唯一の authority とする。現在 `not_activated` の family／candidate は、計画本文、schema 名、provider alias、Thin Slice witness を理由に callable と表示しない。利用者向けの一覧が必要な場合は、current Product snapshotとMatrixから再生成する read-only view を用い、別Registry、別count、別activation bundleを持たせない。

### R06 — Eval／Verification の承認境界

[AI Verification／Provenance](../architecture/01-governance/ai-verification-provenance.md) が Eval Gate を所有する。95% 下限は production elevation の既存条件として維持するが、本 Recommendation、Wave、Thin Slice witnessがその条件をMVP-A entryへ前倒ししたり、緩和・削除したりしない。MVP-A の Evidenceは Product／AI Verification のcurrent Gateをread-backし、同一Candidate、Toolchain、Target、Approvalに束縛する。

### R07 — ECS E0／E1 の承認境界

ECS E0 の入口は current Control Plane baseline、`wp.runtime.ecs-e0=active`、`wp.runtime.scheduling-core=complete`、既存Ownerのcurrent approval、exact Toolchain／revocationである。E0 Task 11 の affected technical document human approval、same-definition Rebaseline、new current binding、final `TechnicalQualificationReceiptV1`が完成して初めて、E1–E3を後続WPのentry候補にできる。

E0のdraft document、preliminary Evidence、checkbox、future E1の期待をE0 entryに混ぜず、E0をcritical pathからdescopeしない。

### R08 — team_assumption planning revision

**規則（既存 Product Plan §5.1 を尊重）:**

- 人数・AI利用量・device をエージェントが推測して埋めない
- ユーザーが入力した composition だけで `team_assumption_state=assumed` へ遷移する別 revision を作る

**assumed に含める最小 Field:**

```text
roles[]: (engine, editor, AI eval, security custodian, approver)
headcount_or_fte
available_runners / gpu / mobile devices (or explicitly none)
independent_approver_available: true|false
notes
```

**完了条件:**

- [ ] `unfixed` のまま Wave≥N を「実行中」と表示しない規則が Critical Path または Product Plan にある
- [ ] R4 / security custodian 不在時の No-Go が明示される

### R09 — 正本 maturity banner の機械投影（任意だが推奨）

将来、Product snapshot から banner Field を生成する。当面は手動テンプレートでよい。

**完了条件（任意 Gate）:**

- [ ] contract-active / operational 件数が正本先頭と Feasibility Review で一致する生成手順がある

### R10 — 件数単一生成源と完了 Gate 分割

**規則:**

- WP／Capability／Future／Operationの件数を記載するときは、Product Plan の exact closureを参照元、検証日、用途（説明か fixtureか）とともに明示する。Markdown上の数字を lifecycle／Activation／実行権限に使わない。
- Generic plan update は partial documentation remediation と明示し、Taskごとの証跡とcurrent Architecture正本を混同しない。新規 plan への分割は本B′適用の対象外である。

**完了条件:**

- [ ] 旧件数を実行入力に使うテスト（文書lint）がある、または手動監査チェックリストが README にある
- [ ] Generic plan の「未 final」がトップステータスで見える

---

## 7. B′で反映する計画品質の順序

実装コード、新規 implementation plan、Product Definition migrationを作らない範囲の、**計画品質 remediation の順序**である。

```text
1. 全14計画を current／historical に分類する Authority Map を置く。
2. 全current計画の先頭へ、実行入口、Approval boundary、official-source／Toolchain ownership を明示する。
3. Control Plane Task 6／Task 10B と Critical Path Wave 0 を、pre-publication materialization と read-only sidecar Report に整合させる。
4. ECS E0 entry と E1–E3 entry を分離し、D3D12 の WARP／Target OS lock Gateを明記する。
5. Product Matrixを唯一の Activation authority とし、MVP-A Thin SliceをWave 4 qualification witnessとして記述する。
6. Generic planのpartial statusと、Future planのplanning-only境界を明示する。
```

---

## 8. 維持すべき品質（壊してはならないもの）

次は defect ではなく、remediation で回帰させてはならない資産である。

1. Active / Future / claim の set equality と fail-closed
2. Planning と Shipping の明示分離
3. AI ≠ Approval / Commit / Activation / Sign / Release
4. Generic Core が Shooter production graph に依存しないこと
5. name-only Operation を current alias として読まないこと
6. `team_assumption_state=unfixed` のときに人数・日程を捏造しないこと
7. Future 25 を利用可能と表示しないこと

監査者は、B′の適用や将来の別提案が上記を壊していないかを最優先で見る。

---

## 9. 非主張 / 禁止解釈

本 Recommendation は次を主張しない。

- Engine が今日 AI でゲームを作れること
- Control Plane / Gateway / Signer が実装済みであること
- MVP-A の calendar 完了日
- B′を適用すれば実装規模、日程、S2評価が自動的に改善すること
- 方針 A だけで実現可能性が本質的に上がること
- 独立監査なしでの Product / Architecture 正本書き換え許可

---

## 10. 成功した計画品質の定義（meta-DoD）

次をすべて満たしたとき、本 Recommendation の remediation は成功とする。

1. 新規参加者（または独立AI）が `docs/plans/README.md` だけでcurrent計画、historical計画、実行権限の所在を列挙できる。
2. 「今日 AI が呼める Operation は何か」に current Product snapshot と Matrixの read-back 結果だけで答えられる。
3. Control Plane construction、D3D12、ECS、Future の各入口が authorization／approval／toolchain／staging candidate のどこで停止するか明確である。
4. Wave 0 が post-complete materialization や自己参照なしで handoff でき、Reportが権限を変更しない。
5. E0とE1、MVP-A witnessとActivation、MVP-A Evidenceとproduction 95% Gateを混同しない。
6. official-source の制約と Toolchain exact lock の所有者が一意である。
7. 実装実績、Target Qualification、Capability Activation、Release／Shipping を文書改善から主張しない。

---

## 11. 決定記録と残るユーザー入力

| ID | 質問 | 影響する remediation |
|---|---|---|
| Q1 | **解決:** B′を適用する | 全体 |
| Q2 | **解決:** 新規 Thin Slice計画を作らず、Wave 4のqualification witnessにする | R03 |
| Q3 | **解決:** ECS E0／E1の既存承認境界を明記し、descopeしない | R07 |
| Q4 | team composition / device / approver の実際の値 | R08 |
| Q5 | **解決:** 既存 assurance profile を変更しない。Production要件の移動は別の明示的 Product／Control Plane Decision が必要 | R02 |

---

## 12. 独立監査チェックリストと出力フォーマット

### 12.1 チェックリスト

監査者は少なくとも次を実施する。

- [ ] §2.4 件数を current tree から再抽出し、差分を報告する
- [ ] §2.2 の「現行実行正本」に抜け／余分がないか確認する
- [ ] D01–D12 の Evidence path を開き、主張が節内容と一致するか確認する
- [ ] B′または将来明示された別方針が、§8 の回帰禁止事項と衝突しないか列挙する
- [ ] R01–R10 の DoD が反証可能か（主観だけになっていないか）を確認する
- [ ] Product Plan / Critical Path と直接矛盾する提案があれば Critical として報告する
- [ ] 「実装すればすぐ AI がゲームを作れる」方向へ緩和しすぎていないか確認する
- [ ] 英語固有名詞、ID、schema 名の typo / 旧名混入を報告する

### 12.2 監査レポート出力フォーマット

```text
# Plan Quality Recommendation Audit

- auditor: <name or model>
- date: YYYY-MM-DD
- subject: docs/reviews/2026-07-24-plan-quality-improvement-recommendation.md
- subject_git_sha: <sha>
- verdict: approve | approve_with_changes | reject

## Critical findings
- [ID] ...

## Important findings
- [ID] ...

## Minor findings
- [ID] ...

## Count reconciliation
| Domain | Recommendation §2.4 | Re-extracted | Delta |
|...|

## Approach recommendation
- keep B′ | propose A | propose C | hybrid
- rationale: ...

## Required changes before user approval
1. ...
```

### 12.3 Severity 定義（監査側）

| Severity | 定義 |
|---|---|
| Critical | 正本矛盾、安全境界破壊、誤った Go 主張、件数の重大 stale |
| Important | remediation の実現性を落とす欠落、DoD 非反証、Authority 曖昧 |
| Minor | 文言、リンク、重複、typo |

---

## 13. 関連文書

| 文書 | 関係 |
|---|---|
| [2026-07-23 Plan Feasibility Remediation Review](2026-07-23-plan-feasibility-remediation.md) | 実現可能性の現行判定。本 Recommendation はそれを否定せず、計画品質の次手を追加する |
| [2026-07-23 Feasibility Remediation Design](../superpowers/specs/2026-07-23-plan-feasibility-remediation-design.md) | superseded historical。件数を流用しない |
| [Generic AI-Native Architecture Design](../superpowers/specs/2026-07-23-generic-ai-native-engine-architecture-design.md) | 4層と AI Schema の設計正本候補 |
| [Critical Path](../plans/2026-07-23-critical-path-execution-plan.md) | Wave 実行計画。R03/R04 の主変更点 |
| [Product Plan](../architecture/00-product/product-plan.md) | Product intent / MVP / Phase の正本 |

---

## 14. Change log

| Date | Change |
|---|---|
| 2026-07-24 | 初版 draft。方針 B 推奨。独立監査向けチェックリスト付き |
| 2026-07-24 | ユーザー承認済み B′を適用。既存Control Plane profile、Product Matrix、E0／E1境界を保持し、Authority Map／official-source／execution boundary を同期。 |

---

## 15. ユーザー承認記録

```text
approved: yes
approach: B′
notes: 公式根拠・実行入口・承認境界を全計画へ反映し、既知の矛盾を修正する。新規 implementation plan は作成しない。
approver: user
date: 2026-07-24
```
