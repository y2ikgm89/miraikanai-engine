# Legacy Branch Reconciliation Design

- Status: approved direction
- Date: 2026-07-22
- Scope: `codex/ai-readable-architecture-closure`、`codex/c1-gameplay-capability-closure`、関連worktree、マージ済みremote branch

## 1. Outcome

古い文書構造に残る未コミット成果を失わず、最新`main`の正規`docs/architecture`／`docs/plans`構造へ意味単位で統合する。統合後は、マージ済みまたは意味移植済みのworktree、local branch、remote branchを削除し、`main`だけが正規成果を保持する状態にする。

## 2. Evidence baseline

開始時点の`main`は`8595833`で、working treeはcleanである。

### 2.1 C1 gameplay branch

- `codex/c1-gameplay-capability-closure`のPR #3はclosed／unmergedである。
- `9f10ec2`から移行判定する識別子の監査台帳は、[Document System Restructure Decision §12.3](../architecture/decisions/2026-07-21-document-system-restructure.md#123-移行対象の完全性)の55型台帳である。全55型は現行`main`で欠落0件である。台帳外の独自抽出件数を監査基準にしない。
- [Document System Restructure Decision §12](../architecture/decisions/2026-07-21-document-system-restructure.md#12-pr-3履歴成果の正本統合)は、PR #3の63 hunkと55型を現行Ownerへ意味移植したことを正本として記録する。
- `5d1f1b8`のCodex設定変更は当時棄却されたが、後続PR #5で`model_reasoning_effort = "xhigh"`が独立承認・マージ済みである。

したがって、最終確認後にこのbranchのlocal／remote refを削除できる。

### 2.2 Architecture comprehension worktree

`codex/ai-readable-architecture-closure`のlinked worktreeには、旧`docs/superpowers/specs` 6文書への207追加／削除行と、241行の未追跡Implementation Planがある。次の6識別子は現行`main`に存在しない。

- `NormativeAuthorityManifestV1`
- `ArchitectureExplainProjectionV1`
- `ExternalEngineConceptResolutionV1`
- `GameHostLoopStateV1`
- `ArchitectureComprehensionCaseV1`
- `ArchitectureComprehensionFixtureV1`

これらを未監査のまま破棄しない。一方、旧pathと旧Review setをそのまま復活させない。

## 3. Compared approaches

### A. Semantic reconciliation and deletion — adopted

dirty worktreeをcheckpoint commitで保全し、各差分を`Preserved`、`Merged`、`Removed`へ分類する。未反映の意味だけを現行Ownerへ移し、最新計画を同じChangeSetで更新してから旧refを削除する。

利点は、正本の一意性、最新path、履歴保全、完全なcleanupを同時に満たすことである。欠点は、機械的なcherry-pickより監査量が多いことである。

### B. Archive old documents

旧文書とpatchをarchiveとして保存し、active仕様へ統合しない。原文は残るが、利用者とAIがarchiveを正本と誤認しやすく、最新計画との同期も保証できないため採用しない。

### C. Discard

dirty worktreeとPR #3 branchを削除する。最短だが、未反映6契約と計画の根拠を失うため採用しない。

## 4. Canonical ownership and name policy

旧名称を無条件にactive schemaとして復活させない。新Control Planeと同じ責務は現行型へmergeし、独立責務だけを新しいV1契約として追加する。

| 旧worktreeの概念 | 現行Owner／Disposition |
|---|---|
| `NormativeAuthorityManifestV1` | `ArchitectureMetadataV1`、document relation registry、generated Architecture Indexへmergeする。別Manifest正本は追加しない |
| `ArchitectureExplainProjectionV1` | Architecture Governanceが生成するbounded read-only projectionとしてControl Plane design／implementation planへ追加し、Project Stateは消費だけを行う |
| `ExternalEngineConceptResolutionV1` | Editor／Workspace UXが入力解決と`question_required`を所有し、Project Stateは解決済みcanonical conceptを持つtyped ChangeSetだけを受理する |
| `GameHostLoopStateV1` | `04-runtime/scheduling-lifetime.md`がouter loop、integer-rational 60 Hz、pause、surface、headless、fault境界を所有する |
| `ArchitectureComprehensionCaseV1` | `01-governance/ai-verification-provenance.md`がCase schema、grader、Evidence closureを所有する |
| `ArchitectureComprehensionFixtureV1` | 同Ownerが240 Case、3 run、hard Gate、infrastructure failure分離を所有する |
| 旧Review set index | 現行`ArchitectureMetadataV1`から生成するIndexへmergeし、固定件数または旧pathを復活させない |

## 5. Plan synchronization

計画書は実装の後追いではなく、各意味移植と同じcommitで更新する。

1. 未追跡の`2026-07-21-ai-readable-architecture-comprehension-closure.md`を現行path、Owner、型名、Control Plane依存へ書き換えて`docs/plans/`へ移す。
2. 各Taskへsource hunk、canonical destination、Disposition、検証command、完了checkboxを記録する。
3. `2026-07-22-architecture-evolution-control-plane-design.md`へbounded Architecture Explainとcomprehension Evidenceの責務を追加する。
4. `2026-07-22-architecture-evolution-control-plane-implementation-plan.md`へschema、generator、CLI、negative fixture、CI Gateのexact Taskを追加する。
5. Runtime ECS E0計画はControl Plane baseline interfaceが変わる場合だけ同じChangeSetで更新し、不要な再記述を行わない。
6. 完了後も計画を削除せず、source branch、統合commit、完了Gateを記録した監査資料として保持する。

## 6. Execution flow

1. dirty worktreeの6変更文書と未追跡計画を、そのbranch上のlocal checkpoint commitへ保存する。
2. 20 hunk／455 changed lineのDisposition ledgerを作成し、未分類を0にする。
3. 最新`main`から作ったintegration branchで、未反映の意味だけを4節のOwnerへ移植する。
4. 5節の計画群を意味移植と同時に更新する。
5. whole-document validationとsource-to-destination coverage auditを実行する。
6. integration branchをpushし、PRを作成して`main`へmergeする。
7. local `main`を同期し、統合内容を再検証する。
8. dirty worktreeをremoveし、`codex/ai-readable-architecture-closure`、`codex/c1-gameplay-capability-closure`、`origin/codex/architecture-document-restructure`の不要refを削除する。

## 7. Failure handling

- checkpoint commitが失敗した場合はworktreeを削除しない。
- source hunkのOwnerが一意に決まらない場合は`Removed`へ分類せず、統合を停止する。
- 新Control Planeと旧型が競合する場合は新Control Planeを正本とし、旧要件をField／validation／fixtureへmergeしたEvidenceをDisposition ledgerへ残す。
- validationまたはPR mergeが失敗した場合は旧worktreeとbranchを保持する。
- branch削除は、merge commitが`origin/main`の祖先であり、working treeがcleanで、source識別子と計画のcoverageが確認できた後だけ行う。

## 8. Verification

- 全Markdownをstrict UTF-8、BOMなし、LF、末尾空白なし、balanced fence、解決可能なrelative linkとして検証する。
- `git diff --check`をintegration前後とmerge後に実行する。
- dirty sourceの20 hunk／455 changed lineを`Preserved`、`Merged`、`Removed`へ全件分類し、未分類0件とする。
- 6識別子が現行Ownerで定義されるか、現行型へmergeされた対応表を持つことを検証する。
- Decision §12.3の55型台帳の全型が現行`main`で欠落0件であることを再確認する。
- 旧`docs/superpowers/specs` path、旧Review set固定件数、重複Owner、互換aliasをactive仕様へ追加しない。
- common credential、local workspace path、未解決placeholder markerを変更対象へ混入させない。
- repositoryに自動test／build manifestがない場合は、その事実をPRと最終報告へ記録する。

## 9. Completion criteria

- 未コミット、未追跡、stashによる未統合成果がない。
- 最新計画が実際のOwner、型、path、実行順、完了状態と一致する。
- integration PRが`main`へmerge済みで、local `main`と`origin/main`が一致する。
- linked worktreeと不要なlocal／remote branchが削除される。
- 保持するbranchは`main`と、別作業で明示的にactiveなbranchだけである。
