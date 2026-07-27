# Unique Commit Migration Design

## 1. Goal

古いローカルブランチにだけ残る10コミットを、現在の`main`へ直接merge／rebaseせずに内容単位で評価し、現行Architecture GovernanceとProduct Planに適合する設計だけを`main`起点の独立ChangeSetへ移植する。移植完了後は旧ブランチと一時worktreeを安全に削除し、ローカルとGitHubを同じ状態へ収束させる。

## 2. Context

対象ブランチはいずれも`dbcd6f004e8b5d2256b23571013ebf3091ffdbb9`から分岐し、現在の`main`より18コミット古い。分岐後の`main`ではArchitecture文書の責務分離、ADR lifecycle、appendix分離、ECS closure、Navigation、Material契約が更新されている。

そのため、Git上で固有なパッチであっても、現在採用すべき設計とは限らない。旧コミットの大量なOwner文書差分をそのまま適用すると、現行Header、規範依存、正本範囲、Product Registry、Decision Logを巻き戻す危険がある。

## 3. Constraints

- `main`のOwner文書、[Architecture Governance](../../architecture/01-governance/architecture-governance.md)、[Decision Log](../../architecture/decisions/README.md)、[Product Plan](../../architecture/00-product/product-plan.md)を現在の正本として扱う。
- 旧ブランチは参照専用とし、merge、rebase、force-pushしない。
- 移植先は毎回最新`main`から作る。
- 一つのChangeSetへ独立したArchitecture判断を混在させない。
- ADRは一つの重要判断を一ファイルで所有し、Owner文書へ現在のSchema、固定値、runtime挙動を置く。
- `review`文書の存在をCapability activation、実装、Qualification完了の証拠にしない。
- 旧ブランチ由来の`docs/plans/`および`docs/superpowers/specs/2026-07-27-ai-discovery-explain-rendering-design.md`を復活させない。
- 現行Product Planに存在しないCapability、claim、Work Packageを移植だけを理由に追加しない。
- ブランチ削除は必要事項の移植または非採用確認が完了した後に行う。Source commitは手動forward-port後もGit上では未mergeのため、通常削除できない場合は対象commit一覧を提示し、ユーザーが正確に`discard`と確認するまで強制削除しない。

## 4. Considered Approaches

### 4.1 旧ブランチをそのままmergeする

不採用。三ブランチは18コミット遅れており、現在の文書再編と同じファイルを広範囲に変更する。merge conflictを解消できても、古い正本構造や重複契約を意味上復活させる危険がある。

### 4.2 旧ブランチを最新`main`へrebaseしてPRにする

不採用。Git履歴は直線化できるが、一つのブランチにAnimation、AI Asset、Simulation Cadence、Audio、Rendering、Platformなど独立判断が混在する問題を解決しない。レビュー単位も大きすぎる。

### 4.3 必要事項だけを最新`main`へ選択的にforward-portする

採用。旧コミットをEvidenceとして読み、現行Owner文書とProduct Planに照合する。現在も必要な設計だけをテーマ別ChangeSetとして書き直し、既に再実装済み、Product判断がない、または旧計画書だけの内容は移植しない。

## 5. Source Commit Disposition

| Source branch | Commit | Subject | Disposition |
|---|---|---|---|
| `codex/debug-animation-contract-closure` | `b7c6540` | animation and support diagnostics | Animation Owner文書へ選択的に移植する |
| `codex/debug-animation-contract-closure` | `8357e1e` | AI asset memory and async loading | 現行ADR形式へ再構成し、必要なOwner接続だけを移植する |
| `codex/debug-animation-contract-closure` | `fc76b04` | AI asset understanding boundaries | `8357e1e`と同じAI Asset Decision ChangeSetへ統合する |
| `codex/debug-animation-contract-closure` | `35ccd17` | simulation cadence and high refresh | CadenceとPresentationの責務分離だけをDecisionとして移植する。未登録high-refresh Capabilityは追加しない |
| `codex/debug-animation-contract-closure` | `8ba1615` | audio engine contracts | Audio Owner文書へ選択的に移植する |
| `codex/debug-animation-contract-closure` | `3ddf044` | audio authoring schema closure | `8ba1615`と同じAudio ChangeSetで現行Schema表現に合わせる |
| `codex/material-system-plan-contract` | `7a1d7ab` | material authoring contracts | 中核は`2377aa7`で再実装済み。Procedural Texture、Displacement、未登録Work Packageは移植しない |
| `codex/virtualized-geometry-lod-design` | `31ceb1a` | virtualized geometry LOD | 現行Product PlanにCapabilityがないため移植しない |
| `codex/virtualized-geometry-lod-design` | `08151de` | AI discovery and rendering explain design | 削除済み計画書／specだけのため移植しない |
| `codex/virtualized-geometry-lod-design` | `05bf1eb` | rendering and platform gaps | Project Shader等の現行内容と重複し、残りは未登録Capabilityのため直接移植しない |

非移植は「将来永久に不採用」を意味しない。将来Product requirementが承認された場合は、その時点の`main`から新しいDecisionとOwner ChangeSetを作る。旧コミットをcurrent Contractとして再利用しない。

## 6. Destination ChangeSets

### 6.1 Animation contract closure

- Base: 最新`main`
- Source evidence: `b7c6540`
- Primary owner: `docs/architecture/05-simulation/animation.md`
- Scope: typed Animation GraphのAI可読境界、Blend Space／同期／event／root motion、IK request、foot placement、retarget、LOD、memory／failure、Editor／AI planned-action closure
- Exclusions: Product Registryの新規Capability、Operation activation、実装済み主張

### 6.2 AI Asset／Memory／Async alignment decision

- Base: Animation ChangeSet統合後の最新`main`
- Source evidence: `8357e1e`、`fc76b04`
- Decision: `docs/architecture/decisions/2026-07-28-ai-asset-memory-async-alignment.md`
- Owners: Memory／Pointers、Asset Lifecycle、Runtime Package、Architecture README、Decision Log
- Scope: Source Assetからimmutable Artifact、staging、atomic publication、generation handle／lease、retirementまでの責務分離と、AI read／explain／propose境界
- Exclusions: 名前だけのSource format対応、MCD／Operation activation、Artifact／Receiptの存在主張

### 6.3 Simulation Cadence／Presentation separation decision

- Base: AI Asset ChangeSet統合後の最新`main`
- Source evidence: `35ccd17`
- Decision: `docs/architecture/decisions/2026-07-28-simulation-cadence-presentation-separation.md`
- Owners: Scheduling／Lifetime、Performance／Capacity、Input、Persistence／Save、Audio、Camera、VFX Runtime
- Scope: Device reading、Presentation、Simulation Advance、Physics substepを別OwnerのProfile／Intervalとして扱う判断と、既存`SimulationAdvanceIntervalV1`利用境界
- Exclusions: `PresentationFrameIntervalV1`のcurrent Schema化、120／240 Hz Capability追加、Target qualification済み主張

### 6.4 Audio contract closure

- Base: Cadence ChangeSet統合後の最新`main`
- Source evidence: `8ba1615`、`3ddf044`
- Primary owner: `docs/architecture/07-platform/audio.md`
- Related owner: `docs/architecture/03-authoring/asset-lifecycle.md`、`docs/architecture/03-authoring/gameplay-programming-model.md`
- Scope: Audio command field、Cue／Bus境界、Clock Domain／Pause、外部依存境界、authoring Schema closure
- Exclusions: Audio Operation activation、Runtime implementation、Target qualification済み主張

## 7. Integration Flow

1. 現在の`main`が`origin/main`の子孫で、working treeがcleanであることを確認する。
2. 文書検証後、`main`の先行2コミットをfast-forward pushする。
3. ChangeSetごとに`.worktrees/<branch-name>`へ隔離worktreeを作る。
4. 旧コミットのpatchを参照し、最新Owner文書へ必要事項だけを書き直す。
5. Header、Decision Log、正本範囲、規範依存、関連文書、内部linkを検証する。
6. ChangeSetをcommit、pushし、`main`向けPRを作る。
7. PRレビューと検証を通過したChangeSetだけを順番に`main`へ統合する。
8. 各統合後にlocal `main`をfast-forwardし、完了したworktreeと移植先ブランチを削除する。
9. 四ChangeSet統合後、Source Commit Dispositionを再照合する。
10. 旧三ブランチが不要になったことを確認する。`git branch -d`が拒否する未merge Source branchは、対象branch／commitを提示してユーザーの正確な`discard`確認を得た後だけ`git branch -D`で削除する。
11. `git worktree prune`と`git fetch --prune`を実行し、main worktree一件、stale refなしへ収束させる。

## 8. Validation

各ChangeSetは少なくとも次を満たす。

- `git diff --check`がexit 0。
- 変更したOwner文書のHeader fieldがArchitecture Governance指定の順序で存在する。
- ADRがDecision Logに一件だけ登録され、stable document IDと相対linkが一致する。
- 変更したMarkdown内のrepository-relative linkが存在し、fragment付きlinkの対象見出しが一意に解決する。
- 同じSchema、fixed value、Capability、Work Packageを複数Ownerで再定義しない。
- `文書状態=review`、`実装状態=absent`を逸脱する主張を追加しない。
- `docs/plans/`と旧AI Discovery specを復活させない。
- ChangeSetのdiffに、対象外テーマの変更を含めない。

最終状態は次で判定する。

- local `main`、`origin/main`、GitHub `main`のSHAが一致する。
- `git status --short --branch`がcleanかつahead／behindなし。
- `git worktree list --porcelain`がmain worktree一件だけを返す。
- `git worktree prune --dry-run --verbose`が空。
- 明示的なdiscard確認後は旧三ブランチと移植用ブランチがlocal／remoteに存在しない。確認前は旧三ブランチを保全する。

## 9. Risks and Controls

| Risk | Control |
|---|---|
| 旧文書の正本構造を復活させる | cherry-pickせず、最新Owner文書へ手動forward-portする |
| 未採用Capabilityをcurrent設計として追加する | Product Planに現行entryがない項目は移植しない |
| ADRとOwner文書が二重正本になる | ADRは理由と選択、Owner文書はSchemaとruntime意味だけを所有する |
| 大規模PRでレビュー不能になる | 四つの独立ChangeSetへ分割し、順次統合する |
| 旧ブランチを早期削除してEvidenceを失う | 四ChangeSet統合とDisposition再照合まで保持し、未merge branchは正確な`discard`確認後だけ強制削除する |
| worktree／remote branchが残る | 統合ごとにworktree removal、branch delete、pruneを検証する |

## 10. Success Criteria

必要なAnimation、AI Asset／Memory、Cadence／Presentation、Audioの設計が現行ガバナンスに適合した独立ChangeSetとして`main`へ統合され、未登録Capabilityと削除済み計画書が復活していないこと。明示的なdiscard確認後は旧ブランチ・移植ブランチ・補助worktreeも残っていないこと。
