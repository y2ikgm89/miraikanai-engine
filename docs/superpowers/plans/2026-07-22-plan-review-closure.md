# Plan Review Closure Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 2026-07-22の計画書レビューを監査可能なclosureへ変換し、ID、Product Registry、Owner、Control Plane、Runtime ECS、D3D12の計画を依存順に矛盾なく改善する。

**Architecture:** レビュー詳細はimmutableなraw evidenceとして保持し、canonical closure台帳で検証結果と処置を一意に管理する。基盤IDとOwnerを先に閉じ、そのbaselineへProduct Registry、Control Plane、ECS、D3D12を接続し、最後に横断検査を行う。

**Tech Stack:** Markdown、PowerShell 7、Git、ripgrep、公式OpenAI Developer Docs、Context7経由のCMake／Microsoft C++公式資料、Android Developers、Microsoft Learn／DirectX-Specs。

## Global Constraints

- Authority順はユーザー承認済みDecision、active architecture Owner、`docs/superpowers/specs/2026-07-22-plan-closure-spec.md`、外部公式一次資料、レビュー提案とする。
- 外部公式資料は外部API／SDK／Toolchain／Platform制約だけを所有し、Miraikanai固有のID、Owner、Phase、Capabilityを所有しない。
- 機械Diagnostic IDは`diagnostic.<domain>.<condition>`、表示／log codeは`MIRAKAN-<DOMAIN>-<CONDITION>`とする。
- Targetとtoolchain profileは同じlogical ID `target.*`を使用し、profile versionは`profile_version`へ分離する。
- JSON／schema Fieldはsnake_caseとする。MCD `namespace_path`を含むlogical IDはNaming正本のkind別grammarを使い、lower snakeまたはlower kebabの一方をsegmentごとに許可する。
- 数字で始まるID segmentは禁止し、旧`2d-*`／`3d-*`は意味を保つ英字開始形へclean migrationする。
- 既存52ファイルの未コミット差分はレビュー適用結果である。各Taskは対象hunkを読み、無関係なuser変更を置換しない。
- 未固定dependencyの値を推測しない。公式資料で確認できないものは`unverifiable`または`unfixed`として保持する。
- active specの承認状態、Capability activation、Release承認をこの文書改善から自動昇格しない。
- 現在のsessionではユーザーがagent delegationを依頼していないため、実行時は`superpowers:executing-plans`を使用する。

---

## File map

### 新規

- `docs/reviews/2026-07-22-plan-review-closure.md`: canonical finding／decision／disposition台帳。

### Review evidence

- `docs/reviews/2026-07-22-plan-review.md`: 件数、要決定一覧、完了状態の要約。
- `docs/reviews/2026-07-22-plan-review-findings.md`: raw findingと外部検証の証跡。

### Foundation／Product

- `docs/architecture/02-foundation/naming-project-layout.md`: kind別ID grammarとDiagnostic ID/code対応。
- `docs/architecture/02-foundation/executable-contracts.md`: `MirakanDiagnosticV1`とProvider projection。
- `docs/architecture/02-foundation/toolchain-dependencies.md`: Target profile ID、CI実行基盤、外部Toolchain制約。
- `docs/architecture/02-foundation/cpp23-modules.md`: CMake／MSVC module制約とshipping境界。
- `docs/architecture/00-product/product-plan.md`: Requirement／Fixture／Phase／WP／Capability registryとproject risk。
- `docs/architecture/03-authoring/asset-lifecycle.md`: First Playable asset調達とprovenance。
- `docs/architecture/04-runtime/debugging-observability-replay.md`: Support bundle Owner。
- `docs/architecture/08-domain-packs/domain-pack-contract.md`: 同梱reference asset set。

### Platform

- `docs/architecture/07-platform/windows.md`: Target profile参照とC++ module shipping境界。
- `docs/architecture/07-platform/android.md`: Target profile参照と16 KiB package inspection。
- `docs/architecture/07-platform/apple.md`: Target profile参照。

### Control Plane／ECS／D3D12

- `docs/plans/2026-07-22-architecture-evolution-control-plane-design.md`
- `docs/plans/2026-07-22-architecture-evolution-control-plane-implementation-plan.md`
- `docs/architecture/decisions/2026-07-22-runtime-ecs-contract.md`
- `docs/plans/2026-07-22-runtime-ecs-e0-implementation-plan.md`
- `docs/plans/2026-07-22-ai-readable-d3d12-backend-design.md`
- `docs/plans/2026-07-22-d3d12-backend-implementation-plan.md`

---

### Task 1: Review evidenceとclosure台帳を正規化する

**Files:**
- Create: `docs/reviews/2026-07-22-plan-review-closure.md`
- Modify: `docs/reviews/2026-07-22-plan-review.md:3-17`
- Modify: `docs/reviews/2026-07-22-plan-review.md:41-143`
- Modify: `docs/reviews/2026-07-22-plan-review-findings.md:1-6`

**Interfaces:**
- Consumes: 詳細所見の253 finding bullet、要決定46行、設計書§4のclosure schema。
- Produces: `review.plan_review_2026_07_22.finding_NNN`、`decision.plan_review_2026_07_22.<name>`、終端disposition。

- [ ] **Step 1: 現状の監査不一致を再現する**

Run:

```powershell
$detail = Get-Content -Raw docs/reviews/2026-07-22-plan-review-findings.md
foreach ($s in 'CONFIRMED','UNVERIFIED','PLAUSIBLE','REFUTED') {
  "$s=$([regex]::Matches($detail, "(?m)^- \*\*\[$s/").Count)"
}
$summary = Get-Content docs/reviews/2026-07-22-plan-review.md
"decisions=$(($summary | Where-Object { $_ -match '^\d+\. \*\*\[' }).Count)"
"unknown=$(($summary | Where-Object { $_ -match '^\d+\. \*\*\[\?\]' }).Count)"
```

Expected:

```text
CONFIRMED=162
UNVERIFIED=43
PLAUSIBLE=48
REFUTED=0
decisions=46
unknown=9
```

- [ ] **Step 2: closure台帳の固定headerとField contractを作る**

`docs/reviews/2026-07-22-plan-review-closure.md`へ次を追加する。

```markdown
# Miraikanai Engine 計画書レビュー Closure（2026-07-22）

- authority: `docs/superpowers/specs/2026-07-22-plan-review-closure-design.md`
- source findings: 253 retained findings
- omitted refuted evidence: 4 findings reported by the original review but not preserved as detail rows
- terminal validation states: `confirmed | refuted`
- transitional disposition: `pending`
- terminal dispositions: `applied | decision_applied | deferred | refuted`

## Canonical decisions

| decision_id | source_items | decision | authority | state |
|---|---|---|---|---|
| `decision.plan_review_2026_07_22.diagnostic_identity` | 17; 30; 39 | dotted IDとMIRAKAN codeを分離する | closure design §5.1 | `approved` |
| `decision.plan_review_2026_07_22.target_profile_identity` | 14; 24; 38 | profile_idを`target.*`へ統一する | closure design §5.2 | `approved` |
| `decision.plan_review_2026_07_22.id_grammar` | 15; 18; 19 | kind別grammarを採用する | closure design §5.3 | `approved` |
| `decision.plan_review_2026_07_22.layer_vocabulary` | 27; 46 | Render Graphをlayer vocabulary Ownerとする | closure design §5 | `approved` |
| `decision.plan_review_2026_07_22.environment_capability` | 29; 41 | Water／Snow／Weather capabilityをRegistryへ登録する | Product Registry closure | `approved` |
| `decision.plan_review_2026_07_22.semantic_role` | 31; 40 | Gameplay Programming Modelをsemantic_role Ownerとする | Owner一意原則 | `approved` |

## Finding ledger

| finding_id | source_location | original_state | validation_state | severity | category | duplicate_of | decision_id | disposition | changed_documents | evidence |
|---|---|---|---|---|---|---|---|---|---|---|
```

- [ ] **Step 3: 253 retained findingへdeterministic IDを割り当てる**

文書順・行順でfinding bulletを列挙し、`finding_001`から`finding_253`まで割り当てる。重複findingも削除せず、`duplicate_of`でcanonical findingへ接続する。`UNVERIFIED`と`PLAUSIBLE`を推測でconfirmedへ変えない。

Run after editing:

```powershell
$closure = Get-Content -Raw docs/reviews/2026-07-22-plan-review-closure.md
$ids = [regex]::Matches($closure, 'review\.plan_review_2026_07_22\.finding_\d{3}') | ForEach-Object Value | Sort-Object -Unique
"finding_ids=$($ids.Count)"
"first=$($ids[0])"
"last=$($ids[-1])"
```

Expected:

```text
finding_ids=253
first=review.plan_review_2026_07_22.finding_001
last=review.plan_review_2026_07_22.finding_253
```

- [ ] **Step 4: review総括の件数表と説明を監査可能にする**

次の区別を明記する。

```markdown
| retained finding | 253 |
| confirmed | 162 |
| unverified | 43 |
| plausible | 48 |
| refuted detail rows retained | 0 |
| refuted findings reported but not retained | 4 |
```

「対象58 Markdown」は`review_target_markdown`、`active_spec`、`modified_markdown`へ分離し、Gitで再導出できない旧値を現在値として断定しない。「212＋横断22」はfinding件数ではなくedit action件数であることを明記する。

- [ ] **Step 5: 要決定46件をcanonical decision一覧へ置換する**

総括には重複を除いたcanonical decisionだけを掲載し、旧番号はclosure台帳の`source_items`から追跡可能にする。相反していた旧推奨を現行推奨として併記しない。

- [ ] **Step 6: Task 1検証とコミット**

Run:

```powershell
rg -n '^\d+\. \*\*\[\?\]' docs/reviews/2026-07-22-plan-review.md
rg -n 'REFUTED 4|全指摘' docs/reviews/2026-07-22-plan-review-findings.md
git diff --check -- docs/reviews
```

Expected: `[?]`は0件。REFUTED 4件は「明細未収録」としてのみ出現。`git diff --check`はexit 0。

Commit:

```powershell
git add docs/reviews/2026-07-22-plan-review.md docs/reviews/2026-07-22-plan-review-findings.md docs/reviews/2026-07-22-plan-review-closure.md
git commit -m "docs: normalize plan review closure"
```

---

### Task 2: ID、Diagnostic、Target profileの正本を統一する

**Files:**
- Modify: `docs/architecture/02-foundation/naming-project-layout.md:54-93`
- Modify: `docs/architecture/02-foundation/naming-project-layout.md:271-295`
- Modify: `docs/architecture/02-foundation/executable-contracts.md:104-153`
- Modify: `docs/architecture/02-foundation/executable-contracts.md:317-362`
- Modify: `docs/architecture/02-foundation/toolchain-dependencies.md:226-243`
- Modify: `docs/architecture/02-foundation/cpp23-modules.md:1-420`
- Modify: `docs/architecture/07-platform/windows.md:1-60`
- Modify: `docs/architecture/07-platform/android.md:1-80`
- Modify: `docs/architecture/07-platform/apple.md:1-70`
- Modify: `docs/plans/2026-07-22-architecture-evolution-control-plane-design.md:478-508`
- Modify: `docs/plans/2026-07-22-architecture-evolution-control-plane-implementation-plan.md:729-848`

**Interfaces:**
- Consumes: closure design §5.1–§5.4、Product Plan Target registry。
- Produces: kind別grammar、Diagnostic ID/code mapping、toolchain lock schema migration。

- [ ] **Step 1: 現在の二重文法を失敗Evidenceとして記録する**

Run:

```powershell
rg -n -F 'windows_desktop_v1' docs/architecture docs/plans
rg -n -F 'apple_mobile_v1' docs/architecture docs/plans
rg -n 'diagnostic\.<domain>\.<condition>|MIRAKAN-<DOMAIN>-<NAME>' docs/architecture/02-foundation
rg -n 'fixture\.[^` ]*\.(2d|3d)[-_]' docs/architecture docs/plans
```

Expected: 旧profile ID、二つのDiagnostic表現、数字開始fixture IDが検出される。

- [ ] **Step 2: kind別grammar表をNaming正本へ追加する**

追加する表は次とする。

```markdown
| kind | canonical grammar | example |
|---|---|---|
| document | `mirakan.arch.<lower-kebab-path>` | `mirakan.arch.product-plan` |
| operation | `operation.<lower-token-path>` | `operation.build.package.validate` |
| diagnostic ID | `diagnostic.<lower-token-path>` | `diagnostic.product.authoring-roundtrip-failed` |
| diagnostic code | `MIRAKAN-<UPPER-KEBAB-PATH>` | `MIRAKAN-PRODUCT-AUTHORING-ROUNDTRIP-FAILED` |
| target | `target.<lower_snake_or_kebab_path>` | `target.windows.desktop` |
| registry logical ID | `<kind>.<lower-kebab-path>` | `fixture.product.shooter-2d` |
| MCD namespace_path | kind別`lower-token-path` | `render.material_general_2d` |
```

`lower-kebab-path`の各segmentは`[a-z][a-z0-9]*(?:-[a-z0-9]+)*`とし、数字開始を拒否する。schema Field名とRegistry IDのseparatorを混同しない。

`lower-token-path`の各segmentはlower snakeまたはlower kebabのどちらか一方を使い、一segment内の`_`と`-`混在を拒否する。

- [ ] **Step 3: Diagnostic IDとcodeを別Fieldとして固定する**

`MirakanDiagnosticV1`へ次を固定する。

```markdown
| `diagnostic_id` | `diagnostic.<domain>.<condition>`、機械参照のprimary key |
| `code` | `MIRAKAN-<DOMAIN>-<CONDITION>`、log／UI／support用code |
| mapping | Naming正本のtoken変換で一対一。alias、裸PascalCase、backend別別名を拒否 |
```

- [ ] **Step 4: Target profile IDをlogical Targetへ統一する**

`toolchain_lock.profiles[].profile_id`のclosed setを次へ置換する。

```text
target.windows.desktop
target.android.mobile
target.apple.mobile
```

lock schema majorを更新し、旧3値から新3値へのmigration表を追加する。`cpp23-modules.md`と3 Platform文書も同じIDを参照する。

- [ ] **Step 5: 数字開始Registry IDのexact migrationを追加する**

最低限次を同一ChangeSetで置換する。

```markdown
| old | new |
|---|---|
| `fixture.product.2d-shooter` | `fixture.product.shooter-2d` |
| `fixture.product.3d-shooter-arena` | `fixture.product.shooter-arena-3d` |
| `fixture.product.2d-platformer` | `fixture.product.platformer-2d` |
| `fixture.product.2d-puzzle-dialogue` | `fixture.product.puzzle-dialogue-2d` |
| `capability.product.2d_general_production` | `capability.product.general_production_2d` |
| `capability.product.3d_general_production` | `capability.product.general_production_3d` |
| `wp.product.2d-general-coverage` | `wp.product.general-coverage-2d` |
| `wp.product.3d-general-coverage` | `wp.product.general-coverage-3d` |
```

- [ ] **Step 6: 公式Toolchain制約を参照化する**

次を規範ではなく外部制約Evidenceとして記録する。

- CMake `cmake-cxxmodules(7)`: `import std`はNinja系、`CMAKE_EXPERIMENTAL_CXX_IMPORT_STD`、`CXX_MODULE_STD`を要求。
- Microsoft C++ named module tutorial: standard header includeと`import std`の混在を禁止する構成制約。
- Android Developers 16 KiB page-size guide: package内の全native `.so`をalignment検査する。
- OpenAI function calling／Responses migration: strict schema adapterとmodel ID設定値化の根拠。

参照URLは次へ固定する。

```text
https://cmake.org/cmake/help/latest/manual/cmake-cxxmodules.7.html
https://learn.microsoft.com/en-us/cpp/cpp/tutorial-import-stl-named-module
https://developer.android.com/guide/practices/page-sizes
https://developers.openai.com/api/docs/deprecations
https://developers.openai.com/api/docs/guides/function-calling
https://developers.openai.com/api/docs/guides/migrate-to-responses
```

- [ ] **Step 7: Task 2検証とコミット**

Run:

```powershell
rg -n -F 'windows_desktop_v1' docs/architecture docs/plans
rg -n -F 'android_mobile_v1' docs/architecture docs/plans
rg -n -F 'apple_mobile_v1' docs/architecture docs/plans
rg -n 'fixture\.[^` ]*\.(2d|3d)[-_]' docs/architecture docs/plans
git diff --check -- docs/architecture/02-foundation docs/architecture/07-platform docs/plans
```

Expected: 旧IDはmigration表または歴史的引用だけ。数字開始fixture IDはmigration表だけ。diff checkはexit 0。

Commit:

```powershell
git add docs/architecture/02-foundation docs/architecture/07-platform docs/plans/2026-07-22-architecture-evolution-control-plane-*.md
git commit -m "docs: unify architecture identities"
```

---

### Task 3: Product RegistryのPhase／Requirement／Fixture closureを閉じる

**Files:**
- Modify: `docs/architecture/00-product/product-plan.md:116-253`
- Modify: `docs/architecture/00-product/product-plan.md:345-536`
- Modify: `docs/plans/2026-07-22-architecture-evolution-control-plane-design.md:742-779`
- Modify: `docs/plans/2026-07-22-architecture-evolution-control-plane-implementation-plan.md:381-405`

**Interfaces:**
- Consumes: Task 2のcanonical ID、既存Target 5件、Phase 0–9。
- Produces: manual authoring、AI authoring、3D First Playable、MVP completion、Project Source Activationの機械結線。

- [ ] **Step 1: Registry不整合を再現する**

Run:

```powershell
rg -n 'phase.headless-authoring|phase.ai-authoring-mvp-a|phase.manual-3d-mvp-b|requirement.product.c2-3d-coverage' docs/architecture/00-product/product-plan.md
rg -n 'Project Source Activation|support|reset|first-run' docs/architecture/00-product/product-plan.md
```

Expected: Phase 1/4が同じmanual_and_ai Requirementを共有し、Phase 6がC2 Requirementを要求し、MVP chain／Project SourceにRegistry closureがない。

- [ ] **Step 2: Requirement registryへ4件を追加する**

```markdown
| `requirement.product.authoring-roundtrip-manual` | `mirakan.arch.product-plan` | `manual_e2e` | `diagnostic.product.authoring-roundtrip-manual-failed` |
| `requirement.product.first-playable-3d` | `mirakan.arch.product-plan` | `playable_e2e` | `diagnostic.product.first-playable-3d-incomplete` |
| `requirement.product.mvp-completion` | `mirakan.arch.product-plan` | `mvp_completion_e2e` | `diagnostic.product.mvp-completion-incomplete` |
| `requirement.product.project-source-activation` | `mirakan.arch.product-plan` | `source_activation_e2e` | `diagnostic.product.project-source-activation-incomplete` |
```

既存`requirement.product.authoring-roundtrip`はAI経路専用、既存`requirement.product.c2-3d-coverage`はPhase 8専用にする。

- [ ] **Step 3: Fixture registryを更新する**

- `fixture.product.authoring-transaction`はmanual Requirementだけを参照する。
- `fixture.product.shooter-2d`はAI authoring、title-to-result、save-load-replay、MVP completion、Project Source Activationを参照する。
- `fixture.product.shooter-arena-3d`はFirst Playable 3Dとsave-load-replayを参照する。
- C2 3D coverageは`shooter-arena-3d`だけで完了せず、Phase 8のcross-genre matrixが複数fixtureを集約する。

- [ ] **Step 4: Phase registryを更新する**

```markdown
| 1 | `phase.headless-authoring` | `requirement.product.authoring-roundtrip-manual` | `wp.authoring.headless-core` | `fixture.product.authoring-transaction` |
| 4 | `phase.ai-authoring-mvp-a` | `requirement.product.authoring-roundtrip; requirement.product.mvp-completion; requirement.product.project-source-activation` | `wp.authoring.project-native-module; wp.rendering.project-shader; wp.product.ai-authoring-mvp-a` | `fixture.product.shooter-2d` |
| 6 | `phase.manual-3d-mvp-b` | `requirement.product.first-playable-3d` | `wp.domain.shooter-3d` | `fixture.product.shooter-arena-3d` |
```

Phase 8だけが`requirement.product.c2-3d-coverage`を保持する。

- [ ] **Step 5: Project Source CapabilityとWPを登録する**

Capability registryへ次を追加する。

```text
capability.project.native_module
capability.project.shader
```

両CapabilityはPhase 4以前にOwner WP、Target、fallback、fixtureを持つ。既存`native-game-module.md`と`project-shader.md`をOwner文書として参照する。

新規WPは`wp.authoring.project-native-module`と`wp.rendering.project-shader`とし、前者は実在する正本ID `mirakan.arch.native-game-module`、後者は`mirakan.arch.rendering-project-shader`をOwnerにする。MVP-Aでは`target.windows.editor; target.windows.desktop`へ限定し、mobile Targetは別Qualificationまでactivationしない。

- [ ] **Step 6: Registry coverageを検査する**

Run:

```powershell
$p = Get-Content -Raw docs/architecture/00-product/product-plan.md
foreach ($id in @(
  'requirement.product.authoring-roundtrip-manual',
  'requirement.product.first-playable-3d',
  'requirement.product.mvp-completion',
  'requirement.product.project-source-activation',
  'capability.project.native_module',
  'capability.project.shader'
)) {
  "$id=$([regex]::Matches($p, [regex]::Escape($id)).Count)"
}
```

Expected: 各IDは定義行と少なくとも一つのconsumer行に現れる。Registry全体はRequirement 16、Fixture 10、Phase 10、Work Package 26、Capability 37で、Phase↔Work Package、Capability→Owner WP、各Requirement／Fixture参照の未解決が0件になる。

- [ ] **Step 7: Task 3コミット**

```powershell
git diff --check -- docs/architecture/00-product/product-plan.md docs/plans/2026-07-22-architecture-evolution-control-plane-*.md
git add docs/architecture/00-product/product-plan.md docs/plans/2026-07-22-architecture-evolution-control-plane-*.md
git commit -m "docs: close product execution registry"
```

---

### Task 4: 欠落Owner責務とMVP実行前提を閉じる

**Files:**
- Modify: `docs/architecture/00-product/product-plan.md:116-195`
- Modify: `docs/architecture/03-authoring/asset-lifecycle.md:523-632`
- Modify: `docs/architecture/04-runtime/debugging-observability-replay.md:1-900`
- Modify: `docs/architecture/08-domain-packs/domain-pack-contract.md:23-153`
- Modify: `docs/architecture/02-foundation/toolchain-dependencies.md:1-281`
- Modify: `docs/plans/2026-07-22-architecture-evolution-control-plane-design.md:138-220`
- Modify: `docs/plans/2026-07-22-architecture-evolution-control-plane-implementation-plan.md:241-270`

**Interfaces:**
- Consumes: 5新Owner正本の計画、Product MVP chain、Task 3 Registry。
- Produces: CI infrastructure、asset sourcing、support bundle、project riskの一意Owner。

- [ ] **Step 1: Product Planへ開発体制とrisk contractを追加する**

次のFieldを持つ節を追加する。

```markdown
| Field | Rule |
|---|---|
| `team_assumption_state` | ユーザー入力前は`unfixed`。人数・AI利用量を推測しない |
| `planning_capacity` | calendar期間を出さず、相対sizeだけを保持する |
| `phase_estimate` | elapsed timeではなく相対size `S / M / L / XL` |
| `critical_path` | Control Plane → ECS E0 →（Headless AuthoringとD3D12を並行）→ Editor Runtime → 2D Shooter → Project Source Activation → Authoring MVP-A |
| `scope_reduction_order` | C2 advanced rendering → non-Shooter packs → mobile shipping。MVP-A contractは削除しない |
| `risk_owner` | `mirakan.arch.product-plan` |
| `review_cadence` | Phase exitごとに更新し、実測Receiptへ差し替える |
```

- [ ] **Step 2: First Playable asset sourcingを閉じる**

- `asset-lifecycle.md`: sourceは`domain_pack_reference | user_provided | external_generated`のclosed enum。MVP-A/Bでは`domain_pack_reference`と`user_provided`だけを許可し、生成ProviderはC2未満で必須にしない。
- `domain-pack-contract.md`: reference asset setはPack manifest hashへ含め、license／provenance／redistribution policyを必須にする。
- 生成Provider不在をsilent fallbackにせず、要求された場合はtyped diagnosticで拒否する。

- [ ] **Step 3: CI実行基盤OwnerをToolchainへ追加する**

`CiExecutionProfileV1`に次を固定する。

```markdown
| `lane_id` | Verification正本のlane ID |
| `runner_class` | `windows_gpu | windows_hardware_vm | macos_build | android_device | apple_device | portable_linux` |
| `hosting_mode` | `managed | self_hosted` |
| `toolchain_profile_id` | `target.*` |
| `device_matrix_ref` | 実機laneのみ必須 |
| `capacity_state` | `unfixed | qualified | unavailable` |
| `owner` | 調達・保守責任主体。未固定ならlane開始を拒否 |
```

- [ ] **Step 4: SupportBundleV1のOwnerを明示する**

`debugging-observability-replay.md`がschema、redaction、size cap、生成operation、failureを所有する。Platform文書は収集可能Fieldだけを投影し、独自Support Bundle schemaを定義しない。

- [ ] **Step 5: 5新Owner正本Taskを開始可能にする**

Control Plane plan Task 3へ各fileの正確なpath、document ID、`state=review`、`approval_ref=null`、positive／negative validationを記載する。Owner未承認時はWork Packageを`declared`のまま保ち、`diagnostic.architecture.owner-unapproved`で開始を拒否する。

- [ ] **Step 6: Task 4検証とコミット**

Run:

```powershell
rg -n 'CiExecutionProfileV1|team_assumption_state|domain_pack_reference|SupportBundleV1|owner-unapproved' docs/architecture docs/plans
git diff --check -- docs/architecture docs/plans
```

Expected: 各contractは一つのOwner定義とconsumer参照を持つ。

Commit:

```powershell
git add docs/architecture/00-product docs/architecture/02-foundation docs/architecture/03-authoring docs/architecture/04-runtime docs/architecture/08-domain-packs docs/plans/2026-07-22-architecture-evolution-control-plane-*.md
git commit -m "docs: assign plan closure owners"
```

---

### Task 5: Control Plane設計と実装計画を実行可能にする

**Files:**
- Modify: `docs/plans/2026-07-22-architecture-evolution-control-plane-design.md:221-508`
- Modify: `docs/plans/2026-07-22-architecture-evolution-control-plane-design.md:817-1014`
- Modify: `docs/plans/2026-07-22-architecture-evolution-control-plane-implementation-plan.md:28-515`
- Modify: `docs/plans/2026-07-22-architecture-evolution-control-plane-implementation-plan.md:729-871`

**Interfaces:**
- Consumes: Task 2 ID grammar、Task 3 Product Registry、Task 4 Owner表。
- Produces: 到達可能なTask graph、document relation registry生成、hash-bound continuation、baseline receipt。

- [ ] **Step 1: 設計／実装planのField差分を列挙する**

Run:

```powershell
rg -n 'document-relations|continuation|signing key|repository-owned|old ID|Completion Gate|ID grammar' docs/plans/2026-07-22-architecture-evolution-control-plane-*.md
```

Expected: relation registryのconsumerはあるがproducerが不足し、repository-owned signing keyが未定義、old ID 0件Gateと本文移行Taskの境界が不完全。

- [ ] **Step 2: document relation registryのproducerをTask 6へ追加する**

Task 6は次を生成する。

```text
architecture/registry/document-relations.v1.json
schemas/architecture/document-relations.schema.json
```

Schemaは`document_id`、direct `requires`、typed reciprocal `integrates_with`、contract ID、source document hashを必須にする。Task 8AとTask 10はこの出力だけを消費する。

- [ ] **Step 3: continuation署名をhash bindingへ置換する**

repository-owned key、algorithm、secret供給を削除する。payloadは次をhashする。

```text
SHA-256(JCS({request_hash, source_closure_hash, revision, scope, expires_at}))
```

`request_hash`へfield mask、Target、next offsetを含める。これはauthenticity署名ではなく入力binding、破損、誤再利用の検出であり、悪意あるcallerのdigest再計算に対するauthorityを与えないと明記する。Authorityが必要なcommit／approvalは既存Approval Contractへ委譲する。

- [ ] **Step 4: 本文migrationとCompletion Gateを一致させる**

Task 4Bが43 active spec本文、Platform profile ID、Diagnostic ID、numeric-leading IDを更新することを明記し、Appendix Dの全old IDを網羅する。歴史的Decisionとmigration表は0件Gateから除外する。

- [ ] **Step 5: canonical topological orderとbaseline Fieldを一致させる**

DesignとImplementation Planでbaseline receipt Field名、hash対象、node数導出規則を同じ表へ揃える。件数は固定値ではなく生成registryから導出する。

- [ ] **Step 6: Task 5検証とコミット**

Run:

```powershell
rg -n 'repository-owned signing key|signing key profile' docs/plans/2026-07-22-architecture-evolution-control-plane-*.md
rg -n 'document-relations.v1.json|document-relations.schema.json' docs/plans/2026-07-22-architecture-evolution-control-plane-implementation-plan.md
git diff --check -- docs/plans/2026-07-22-architecture-evolution-control-plane-*.md
```

Expected: signing key参照0件。relation registryとschemaはproducer Taskとconsumer Taskの両方に現れる。

Commit:

```powershell
git add docs/plans/2026-07-22-architecture-evolution-control-plane-design.md docs/plans/2026-07-22-architecture-evolution-control-plane-implementation-plan.md
git commit -m "docs: make control plane plan executable"
```

---

### Task 6: Runtime ECS E0のbaseline依存とscopeを閉じる

**Files:**
- Modify: `docs/architecture/02-foundation/naming-project-layout.md:61-72`
- Modify: `docs/architecture/decisions/2026-07-22-runtime-ecs-contract.md:64-165`
- Modify: `docs/architecture/decisions/2026-07-22-runtime-ecs-contract.md:383-955`
- Modify: `docs/architecture/decisions/2026-07-22-runtime-ecs-contract.md:1537-1663`
- Modify: `docs/plans/2026-07-22-runtime-ecs-e0-implementation-plan.md:11-126`
- Modify: `docs/plans/2026-07-22-runtime-ecs-e0-implementation-plan.md:127-376`

**Interfaces:**
- Consumes: Control Plane approved baseline、5新Owner、Task 2 naming。
- Produces: E0 schema freeze条件、実行context、generation関係、段階化Qualification。

- [ ] **Step 1: E0開始Gateをexact baselineへbindする**

開始条件を次へ限定する。

```text
architecture_baseline_receipt_hash != null
architecture_governance.state == approved
compatibility_evolution.state == approved
persistence_save.state == approved
entity_component_system.state == review
```

`entity_component_system`はE0 Taskが作る対象なので開始時にapprovedを要求しない。

- [ ] **Step 2: E0 schema freeze前の未定義を閉じる**

Decisionへ次のclosed規則を追加する。

- enable bitsetはtrivially stored componentのentity slotごとに一bit、tag／singletonには割り当てない。
- `partition_policy`は`single | fixed_range | deterministic_hash`。
- `RuntimeSystemExecutionContextV1`はtick、phase、partition、read snapshot、write batch、diagnostic sinkを保持する。
- store generationとparticipant generationの更新順、stale拒否、publish atomicityを一つのtableで固定する。

- [ ] **Step 3: in-memory type naming例外を正本へ接続する**

`RuntimeEntityHandle`等のin-memory value typeはNamingのversion suffix省略categoryへ登録し、ECS Decision内だけの暗黙例外にしない。

- [ ] **Step 4: C1と後段stress Qualificationを分離する**

- E0/C1: First Playable fixture、30分soak、正本capacity内entity数。
- C2/stress: 100万Entity synthetic、2時間endurance、全archetype handoff matrix。

E0 plan Completion GateからC2/stressを外し、将来Qualification WPへ参照する。

- [ ] **Step 5: contract compiler／CTest ownerをTask 0で閉じる**

Task 0は作成file、command、positive fixture、negative fixture、後続Taskが消費するtarget名を完全記載する。未存在targetを暗黙前提にしない。

- [ ] **Step 6: Task 6検証とコミット**

Run:

```powershell
rg -n 'RuntimeSystemExecutionContextV1|partition_policy|enable bitset|100万|2時間|30分' docs/architecture/decisions/2026-07-22-runtime-ecs-contract.md docs/plans/2026-07-22-runtime-ecs-e0-implementation-plan.md
rg -n 'RuntimeEntityHandle|RuntimeWorldPublicationHandle|serializable = false' docs/architecture/02-foundation/naming-project-layout.md
git diff --check -- docs/architecture/02-foundation/naming-project-layout.md docs/architecture/decisions/2026-07-22-runtime-ecs-contract.md docs/plans/2026-07-22-runtime-ecs-e0-implementation-plan.md docs/superpowers/plans/2026-07-22-plan-review-closure.md
```

Expected: 未定義語がclosed tableへ接続され、E0/C1とC2/stressが別Gateになる。

Commit:

```powershell
git add docs/architecture/02-foundation/naming-project-layout.md docs/architecture/decisions/2026-07-22-runtime-ecs-contract.md docs/plans/2026-07-22-runtime-ecs-e0-implementation-plan.md docs/superpowers/plans/2026-07-22-plan-review-closure.md
git commit -m "docs: close runtime ecs e0 plan"
```

---

### Task 7: D3D12計画のPhase順序と設計／実装差分を閉じる

**Files:**
- Modify: `docs/plans/2026-07-22-ai-readable-d3d12-backend-design.md:155-209`
- Modify: `docs/plans/2026-07-22-ai-readable-d3d12-backend-design.md:490-599`
- Modify: `docs/plans/2026-07-22-ai-readable-d3d12-backend-design.md:762-865`
- Modify: `docs/plans/2026-07-22-d3d12-backend-implementation-plan.md:27-90`
- Modify: `docs/plans/2026-07-22-d3d12-backend-implementation-plan.md:93-386`
- Modify: `docs/plans/2026-07-22-d3d12-backend-implementation-plan.md:413-450`

**Interfaces:**
- Consumes: Control Plane baseline、ECS E0 handoff、Target profile ID、official DirectX constraints。
- Produces: Phase 2で実行可能なbackend Qualificationと後段Product Qualificationの分離。

- [ ] **Step 1: 将来fixture依存を再現する**

Run:

```powershell
rg -n '2D shooter|3D shooter|shooter-2d|shooter-arena-3d|Task 13|Qualification' docs/plans/2026-07-22-*d3d12*.md
```

Expected: Phase 2 backend完了条件がPhase 3／6 content fixtureへ依存している箇所が検出される。

- [ ] **Step 2: Qualificationを二層へ分離する**

```markdown
| Gate | Fixture | Phase |
|---|---|---|
| Backend C1 | empty scene、WARP conformance、hardware smoke、device-loss injection | Phase 2 |
| Product integration | `fixture.product.shooter-2d` | Phase 3/4 |
| 3D integration | `fixture.product.shooter-arena-3d` | Phase 6 |
| C2 matrix | cross-genre／multi-target fixture set | Phase 8 |
```

Task 13はBackend C1だけでD3D12 planを完了し、後段fixtureはProduct Registryが追跡する。

- [ ] **Step 3: compute barrier queue不整合を解消する**

compute sampleはcompute queueで合法なlayoutとsync/accessだけを使用し、Direct queue限定transitionを要求しない。cross-queue handoffはproducer fence、consumer wait、owner queue上のtransitionを明記する。

- [ ] **Step 4: design／implementationのfile mapと型名を一致させる**

Design §24/§25の既存正本更新13件をTask 1.5のexact checklistへ写し、directory、type、window handoff型、diagnostic ID、test target名を同じ表記にする。

- [ ] **Step 5: 公式DirectX制約をEvidenceへ固定する**

- Agility SDK stable artifactはToolchain lockが所有する。
- Enhanced Barrierのsync／access／layout、alias／discard orderingはDirectX-Specsを根拠にする。
- D3D12MAのallocator／budget／aliasing境界はofficial repository releaseとheader contractを根拠にする。
- 公式資料がMiraikanaiの20% headroomやvisual toleranceを推奨するとは記載しない。それらはMiraikanai Qualification policyとする。

- [ ] **Step 6: Task 7検証とコミット**

Run:

```powershell
rg -n 'Phase 2|Phase 3|Phase 6|Phase 8|Backend C1|Product integration' docs/plans/2026-07-22-*d3d12*.md
git diff --check -- docs/plans/2026-07-22-ai-readable-d3d12-backend-design.md docs/plans/2026-07-22-d3d12-backend-implementation-plan.md
```

Expected: Phase 2 completionはPhase 3/6 fixtureを要求せず、後段integration参照は残る。

Commit:

```powershell
git add docs/plans/2026-07-22-ai-readable-d3d12-backend-design.md docs/plans/2026-07-22-d3d12-backend-implementation-plan.md
git commit -m "docs: align d3d12 plan with product phases"
```

---

### Task 8: 全Findingを終端化し横断検査を閉じる

**Files:**
- Modify: `docs/reviews/2026-07-22-plan-review-closure.md`
- Modify: `docs/reviews/2026-07-22-plan-review.md`
- Modify: all files changed by Tasks 2–7 when a validation failure identifies a local defect.

**Interfaces:**
- Consumes: Tasks 1–7のcommitとEvidence。
- Produces: 全253 findingの終端disposition、残余risk、再現可能な検査結果。

- [ ] **Step 1: UNVERIFIED high 10件を最優先で再検証する**

各findingについて原文引用、現在の正本、他Owner文書を照合し、`confirmed`または`refuted`へ変更する。検証失敗を`confirmed`へ昇格しない。

- [ ] **Step 2: 残るUNVERIFIED 33件とPLAUSIBLE 48件を処理する**

- 実害のある参照切れ／Gate不能はconfirmedへ昇格して適用する。
- style改善だけで正本挙動が変わらないものは`deferred`にする。
- 重複はcanonical findingのEvidenceを参照し、同じ修正を二重計上しない。

- [ ] **Step 3: closure終端性を検査する**

Run:

```powershell
$rows = foreach ($line in Get-Content docs/reviews/2026-07-22-plan-review-closure.md) {
  if ($line -notmatch '^\| `review\.plan_review_2026_07_22\.finding_\d{3}` \|') { continue }
  $cells = $line.Split('|')
  [pscustomobject]@{
    validation = $cells[4].Trim().Trim('`')
    category = $cells[6].Trim().Trim('`')
    disposition = $cells[9].Trim().Trim('`')
  }
}
"rows=$($rows.Count)"
"unverified=$(($rows | Where-Object validation -eq 'unverified').Count)"
"unknown_category=$(($rows | Where-Object category -eq '?').Count)"
"terminal=$(($rows | Where-Object disposition -in @('applied', 'decision_applied', 'deferred', 'refuted')).Count)"
```

Expected:

```text
rows=253
unverified=0
unknown_category=0
terminal=253
```

- [ ] **Step 4: Markdownと相対linkを検査する**

Run:

```powershell
git diff --check
$files = rg --files docs -g '*.md'
"markdown_files=$($files.Count)"
$anchors = @{}
foreach ($f in $files) {
  $seen = @{}
  $slugs = [System.Collections.Generic.HashSet[string]]::new([System.StringComparer]::Ordinal)
  $inFence = $false
  foreach ($line in Get-Content $f) {
    if ($line -match '^\s*(```|~~~)') { $inFence = -not $inFence; continue }
    if (-not $inFence -and $line -match '^#{1,6}\s+(.+?)\s*#*\s*$') {
      $heading = $Matches[1] -replace '\[([^\]]+)\]\([^)]+\)', '$1'
      $base = $heading.ToLowerInvariant()
      $base = [regex]::Replace($base, '[`*_~]', '')
      $base = [regex]::Replace($base, '[^\p{L}\p{Nd}\s_-]', '')
      $base = ([regex]::Replace($base, '\s+', '-')).Trim('-')
      $index = if ($seen.ContainsKey($base)) { $seen[$base] + 1 } else { 0 }
      $seen[$base] = $index
      $slug = if ($index -eq 0) { $base } else { "$base-$index" }
      [void]$slugs.Add($slug)
    }
  }
  $anchors[(Resolve-Path -LiteralPath $f).Path] = $slugs
}
$missingLinks = @()
$missingAnchors = @()
foreach ($f in $files) {
  $dir = Split-Path -Parent $f
  $inFence = $false
  foreach ($line in Get-Content $f) {
    if ($line -match '^\s*(```|~~~)') { $inFence = -not $inFence; continue }
    if ($inFence) { continue }
    foreach ($m in [regex]::Matches($line, '(?<!\!)\[[^\]]+\]\(([^)]+)\)')) {
      $raw = $m.Groups[1].Value.Trim().Trim('<','>')
      if ($raw -match '^[a-z]+://' -or $raw -match '^mailto:') { continue }
      $parts = $raw -split '#', 2
      $pathPart = $parts[0]
      $targetPath = if ($pathPart.Length -eq 0) { $f } else { Join-Path $dir $pathPart }
      if (-not (Test-Path -LiteralPath $targetPath)) {
        $missingLinks += "$f -> $raw"
        continue
      }
      if ($parts.Count -eq 2 -and $parts[1].Length -gt 0) {
        $resolved = (Resolve-Path -LiteralPath $targetPath).Path
        $fragment = [System.Uri]::UnescapeDataString($parts[1]).ToLowerInvariant()
        if (-not $anchors[$resolved].Contains($fragment)) {
          $missingAnchors += "$f -> $raw"
        }
      }
    }
  }
}
$missingLinks
$missingAnchors
"missing_links=$($missingLinks.Count)"
"missing_anchors=$($missingAnchors.Count)"
```

Expected: `git diff --check` exit 0、`missing_links=0`、`missing_anchors=0`。

- [ ] **Step 5: old IDとRegistry参照を検査する**

Run:

```powershell
$migrationPlan = 'docs/plans/2026-07-22-architecture-evolution-control-plane-implementation-plan.md'
$migrationLines = Get-Content $migrationPlan
$appendixAuthorityStart = ($migrationLines | Select-String '^## Appendix D:').LineNumber
$appendixStart = ($migrationLines | Select-String '^### D\.1 ').LineNumber
$appendixEnd = ($migrationLines | Select-String '^## Completion Gate$').LineNumber
$idMap = @{}
foreach ($line in $migrationLines[($appendixStart - 1)..($appendixEnd - 2)]) {
  if ($line -notmatch '^\|' -or $line -match '^\|---|Old ID') { continue }
  $values = [regex]::Matches($line, '`([^`]+)`') | ForEach-Object { $_.Groups[1].Value }
  if ($values.Count -ge 2) { $idMap[$values[0]] = $values[1] }
}

$normativeOldIds = @()
$allowedOldIds = @()
foreach ($file in (rg --files docs/architecture docs/plans -g '*.md')) {
  $lineNumber = 0
  foreach ($line in Get-Content $file) {
    $lineNumber++
    foreach ($oldId in $idMap.Keys) {
      if (-not $line.Contains($oldId)) { continue }
      $normalized = $file -replace '\\', '/'
      $allowed =
        ($normalized -eq $migrationPlan -and
          (($lineNumber -ge $appendixAuthorityStart -and $lineNumber -lt $appendixEnd) -or
           $line -match 'old ID|allowed location')) -or
        ($normalized -eq 'docs/architecture/02-foundation/toolchain-dependencies.md' -and
          $line.Contains($idMap[$oldId])) -or
        ($normalized -eq 'docs/plans/2026-07-22-ai-readable-d3d12-backend-design.md' -and
          $line -match '旧|置換|0件') -or
        ($normalized -match '^docs/architecture/decisions/')
      $hit = "$normalized`:$lineNumber $oldId"
      if ($allowed) { $allowedOldIds += $hit } else { $normativeOldIds += $hit }
    }
  }
}

$product = Get-Content -Raw docs/architecture/00-product/product-plan.md
$registry = $product.Substring($product.IndexOf('## 11. Product execution registries'))
$definedIds = [System.Collections.Generic.HashSet[string]]::new([System.StringComparer]::Ordinal)
foreach ($match in [regex]::Matches($registry, '(?m)^\| `(target|requirement|fixture|fallback|wp|capability)\.[^`]+` \|')) {
  [void]$definedIds.Add($match.Value.Split('`')[1])
}
foreach ($match in [regex]::Matches($registry, '(?m)^\| \d+ \| `(phase\.[^`]+)` \|')) {
  [void]$definedIds.Add($match.Groups[1].Value)
}
$registryRefs = [regex]::Matches($registry, '(target|requirement|fixture|fallback|phase|wp|capability)\.[a-z0-9._-]+') |
  ForEach-Object Value | Sort-Object -Unique
$unresolvedRegistryRefs = @($registryRefs | Where-Object { -not $definedIds.Contains($_) })
$reviewMarkers = @(rg -n '\*\*\[\?\]|UNVERIFIED/high|UNVERIFIED/medium' docs/reviews/2026-07-22-plan-review.md docs/reviews/2026-07-22-plan-review-closure.md)

"appendix_d_old_ids=$($idMap.Count)"
"allowed_migration_or_history=$($allowedOldIds.Count)"
"normative_old_id_hits=$($normativeOldIds.Count)"
"unresolved_registry_refs=$($unresolvedRegistryRefs.Count)"
"unresolved_review_markers=$($reviewMarkers.Count)"
$normativeOldIds
$unresolvedRegistryRefs
```

Expected: `normative_old_id_hits=0`、`unresolved_registry_refs=0`、`unresolved_review_markers=0`。old IDはexact migration表、同表を説明するtest／移行文、歴史的Decisionだけに残す。

- [ ] **Step 6: 最終差分をscope別にレビューする**

Run:

```powershell
git diff --stat ba96f1c..HEAD
git diff --name-only ba96f1c..HEAD
git status --short
```

Expected: 変更はこのplanのFile map内。ユーザーの元差分を消す削除や無関係fileを含まない。

- [ ] **Step 7: final closureをコミットする**

```powershell
$reviewAppliedFiles = @(
  'docs/architecture/00-product/product-plan.md',
  'docs/architecture/01-governance/ai-security-approval.md',
  'docs/architecture/01-governance/ai-verification-provenance.md',
  'docs/architecture/02-foundation/core-architecture.md',
  'docs/architecture/02-foundation/cpp23-modules.md',
  'docs/architecture/02-foundation/executable-contracts.md',
  'docs/architecture/02-foundation/math-core.md',
  'docs/architecture/02-foundation/memory-pointers.md',
  'docs/architecture/02-foundation/naming-project-layout.md',
  'docs/architecture/02-foundation/toolchain-dependencies.md',
  'docs/architecture/03-authoring/asset-lifecycle.md',
  'docs/architecture/03-authoring/editor-ui-framework.md',
  'docs/architecture/03-authoring/editor-workspace-ux.md',
  'docs/architecture/03-authoring/gameplay-programming-model.md',
  'docs/architecture/03-authoring/native-game-module.md',
  'docs/architecture/03-authoring/project-state.md',
  'docs/architecture/04-runtime/debugging-observability-replay.md',
  'docs/architecture/04-runtime/performance-capacity.md',
  'docs/architecture/04-runtime/scheduling-lifetime.md',
  'docs/architecture/05-simulation/animation.md',
  'docs/architecture/05-simulation/collision.md',
  'docs/architecture/05-simulation/navigation.md',
  'docs/architecture/05-simulation/physics.md',
  'docs/architecture/06-rendering/camera.md',
  'docs/architecture/06-rendering/environment-surfaces.md',
  'docs/architecture/06-rendering/lighting.md',
  'docs/architecture/06-rendering/lod.md',
  'docs/architecture/06-rendering/materials.md',
  'docs/architecture/06-rendering/post-processing.md',
  'docs/architecture/06-rendering/project-shader.md',
  'docs/architecture/06-rendering/render-graph.md',
  'docs/architecture/06-rendering/vfx-authoring.md',
  'docs/architecture/06-rendering/vfx-runtime.md',
  'docs/architecture/06-rendering/world.md',
  'docs/architecture/07-platform/android.md',
  'docs/architecture/07-platform/apple.md',
  'docs/architecture/07-platform/audio.md',
  'docs/architecture/07-platform/input.md',
  'docs/architecture/07-platform/mobile-common.md',
  'docs/architecture/07-platform/ui-text-localization-accessibility.md',
  'docs/architecture/07-platform/windows.md',
  'docs/architecture/08-domain-packs/domain-pack-contract.md',
  'docs/architecture/08-domain-packs/shooter.md',
  'docs/architecture/decisions/2026-07-22-runtime-ecs-contract.md',
  'docs/developer-tools/codex/configuration.md',
  'docs/plans/2026-07-22-ai-readable-d3d12-backend-design.md',
  'docs/plans/2026-07-22-architecture-evolution-control-plane-design.md',
  'docs/plans/2026-07-22-architecture-evolution-control-plane-implementation-plan.md',
  'docs/plans/2026-07-22-d3d12-backend-implementation-plan.md',
  'docs/plans/2026-07-22-legacy-branch-reconciliation-design.md',
  'docs/plans/2026-07-22-legacy-branch-reconciliation-disposition.md',
  'docs/plans/2026-07-22-runtime-ecs-e0-implementation-plan.md'
)
$closure = Get-Content -Raw docs/reviews/2026-07-22-plan-review-closure.md
$unmapped = $reviewAppliedFiles | Where-Object { $closure -notmatch [regex]::Escape($_) }
if ($unmapped) { throw "review-applied files without closure mapping: $($unmapped -join ', ')" }
git add -- $reviewAppliedFiles
git add -- docs/reviews/2026-07-22-plan-review.md docs/reviews/2026-07-22-plan-review-findings.md docs/reviews/2026-07-22-plan-review-closure.md
git commit -m "docs: finalize plan review closure"
```

---

## Completion Gate

以下をすべて満たしたときだけ完了とする。

1. retained finding 253件すべてに一意IDと終端dispositionがある。
2. REFUTED 4件の明細欠落を明示し、架空のfinding本文を作っていない。
3. 要決定一覧に`[?]`、重複推奨、相反する現行推奨がない。
4. Diagnostic ID/code、Target profile ID、kind別grammarが全正本で一致する。
5. Phase 1/4/6/8のRequirement–Fixture–WP–Target結線が解決する。
6. MVP completion、Project Source Activation、asset sourcing、CI infrastructure、Support Bundle、project riskに一意Ownerがある。
7. Control Planeのrelation registryにproducerがあり、continuationはsecret keyではなくhash bindingを使う。
8. ECS E0はapproved baselineへbindされ、C1とC2/stress Gateが分離される。
9. D3D12 Phase 2 Gateは将来content fixtureへ依存しない。
10. `git diff --check`、relative link検査、old ID検査、closure終端性検査が成功する。
