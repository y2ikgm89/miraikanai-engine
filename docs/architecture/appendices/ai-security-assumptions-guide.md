# AI Security Assumptions／Questions Guide

- 文書ID: mirakan.appendix.ai-security-assumptions-guide
- 文書種別: explanatory supplement
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 親Owner: [AI Security／Approval](../01-governance/ai-security-approval.md)
- 正本範囲: Beginner question／assumptionの説明、Game production recordを用いるsecurity guidance、Project data／Engine境界、negative scenario
- 非正本範囲: Game intent／Brief／Spec／Question／Assumption／Decision／理解closureの型と意味、親Ownerが所有するAuthorization／Capsule、実装Task、実装順序、生成済みArtifactまたはQualification結果
- 規範依存: [親Owner](../01-governance/ai-security-approval.md)
- 関連文書: [Game Production Loop](../03-authoring/game-production-loop.md)、[Architecture Governance](../01-governance/architecture-governance.md)
- 根拠区分: project-decision／provisional。実ArtifactがないRegistry、Catalog、Fixtureは候補
- 外部根拠確認日: 2026-07-27

> 本書はGame production型を定義しない。[Game Production Loop](../03-authoring/game-production-loop.md)のcanonical type／Refをsecurity guidanceへ使用し、[親Owner](../01-governance/ai-security-approval.md)の`AiTaskContextCapsuleV1`／projection bindingを上書きしない。以下のRegistry、Catalog、Fixture候補は対応Artifactが存在するまでcurrent、実装済みまたはqualifiedではない。
## 5. Beginner questions、assumptions、理解条件

[Game Production Loop §4–§6](../03-authoring/game-production-loop.md#4-game-production-subjectとintent-session)は`GameIntentDraftV1 → GameBriefDocumentV1 → GameSpecDocumentV1 → GameUnderstandingClosureV1`と、`GameQuestionRecordV1`／`GameAssumptionRecordV1`／`GameDecisionRecordV1`を一意に所有する。本書はそれらのField、bound、hash、dispositionまたはRef tupleを複写しない。不足型を同名shadow schema、自由JSON、表示名、会話summaryまたは旧candidate名から補完しない。

`AiTaskContextCapsuleV1`はTask Authorizationとactive Operation refの積集合を持つread-only／Disposable projectionで、Capsule自身、Game理解record、自由文のselection reasonまたはcontinuation tokenは権限にならない。Game Understandingを要求するOperationでは、親Owner §5がcanonical `GameUnderstandingClosureRefV1`をcomplete／current／authorized projectionとして検証し、`ready_to_stage`以外、`fresh`でないEvidence、wrong Project／Candidate／TargetをModel呼出し前に拒否する。

質問を次に分類する。

| Class | 動作 |
|---|---|
| Blocking | 回答まで該当Scopeの実装開始不可 |
| High | 推奨案、影響、変更可能性を示し回答を求める |
| Medium | typed Default、根拠、期限、再検証条件を持つaccepted Assumptionを明示して進行可 |
| Low | Project規約から決め、Decisionへ記録 |

初回はBlockingとHighを最大7問へまとめる。超える場合はGame core、Visual、Platform／Businessの順に分割する。初心者向けUIは複雑さを隠せるが、質問、仮定、Risk、Approval、制限を省略してはならない。

大まかなPromptは`GameIntentDraftV1`、Capability／Platform／権利／Online／Save／年齢／Performance検査、`GameQuestionRecordV1`、`GameBriefDocumentV1`、人間確認、First Playable Scope、実装方式、Staging、Gate、Approvalの順で処理する。

AIの「理解した」という自己申告を状態にしない。`GameQuestionRecordV1.resolution`はopen／answered／withdrawnのexact一branchで、回答文字列だけ、callerの「回答済み」、Decision refなしのansweredを拒否する。`GameAssumptionRecordV1`はMedium open質問だけをsourceにし、typed Default、根拠1件以上、有限の`expires_at`、再検証条件1件以上を全て必須にする。根拠0、無期限sentinel、期限切れ、再検証条件0、Closure Question集合のsame-ID different-hash、同一質問への複数active assumptionでは進行しない。条件成立またはEvidence freshness切れ時は、期限前でも新Decision／Assumption hashを発行して再検証する。

`GameUnderstandingClosureV1`の終端条件はcanonical Ownerだけを参照する。Security preflightは`disposition=ready_to_stage`、同一Project lineage、complete exact refs、required Evidenceの`fresh`を確認し、第三state、自由文字列、AI override、別Project／Target／CandidateのClosureを拒否する。Project Shaderを含むSystemは全参照Module／Techniqueに有効な`ShaderUnderstandingClosureV1`と`ProjectShaderQualificationReceiptV1`を必要とし、欠落／`fresh`でない／Target不一致をGame全体の理解で相殺しない。

## 6. Project data、Project C++／Shader／Native module

### 6.1 Immutable Engine boundary

Game制作時のEngineを署名済み、content-addressed、read-only baselineに固定する。Projectはbaseline IDだけでなくEngine、Editor、GameHost、Public SDK、Capability set、Schema／compiler、Validator、Engine-owned Test、Policy、Toolchain、Target Profile、File manifestのroot hashをProjectLockV1へ固定する。

- Engine sourceをProject workspace、AI Context、Source Worker Bundleへ入れない。
- Engine package、SDK、Validator、Policyをread-only mountする。
- engine、editor、runtime、toolchain、policy、Engine-owned schemas／testsへのwriteを拒否する。
- Build、Preview、Packageの前後で署名と全hashを再検証する。
- Project Build scriptによるinclude／link path、Validator、Compiler option上書きを拒否する。
- Baseline mismatch、未知／欠損FileをMIRAKAN-ENGINE-BASELINE_MISMATCHで停止し、AIに修復させない。

Game制作Tool catalogへEngine source patch、Engine module／Extension／Adapter、private API、Validator／Policy／Test変更、Engine Git／Release、binary／signature差替えを登録しない。新しい署名済みEngine Releaseへの更新は明示Migrationであり、その場のEngine変更ではない。

### 6.2 許可するProject artifact

- Game Brief、GameSpec、Requirement、Decision。
- GameSystemSpecV1、Project-defined System Contract。
- GameplayDefinition、World、Scene、Space、UI、Asset、Material、Animation、Audio設定、選択済みPackの`StageDefinitionV1`等のowner-typed Source。Level WorkspaceはProject artifactではない。
- Project Test、Fixture、Benchmark、Replay、Save migration。
- BoundedNativeGameProfileV1に適合するNativeGameModule。
- [Project Shader](../06-rendering/project-shader.md)の`BoundedProjectShaderProfileV1`に適合する`ProjectShaderModuleV1`／`ProjectShaderTechniqueV1` Source、`ShaderFactGraphV1`、`ShaderUnderstandingClosureV1`、Target別`ProjectShaderArtifactSetV1`。
- 生成Bindingの入力となるProject Contract。GeneratorとEngine-owned Schemaは変更不可。

Gameplay／System実装は次の順で検討する。

    既存System／Capability composition
      -> GameplayDefinition
      -> Cook／index／layout最適化
      -> Bounded NativeGameModule
      -> capability_unavailable

Material／Rendering実装は次の順で検討する。

    既存Material／Template／Pass Capability composition
      -> Material Instance／closed typed Graph
      -> Project Shader Function／Node／Library
      -> Project Shading Model／Stage Module
      -> declarative Project Shader Technique／Project Renderer Feature
      -> capability_unavailable

Engine Extension、Engine Adapter、Engine core変更をGame制作のfallbackにしない。

### 6.3 Bounded Native

Project C++はgenerated public SDK、C ABI、許可Primary Moduleだけを使う。次を禁止する。

- Engine private、Platform、Vendor、Editor header／object／pointer／container／allocator／native handle。
- File、socket、process、environment、registry、OS service、dynamic library、native SDKの直接access。
- inline assembly、未知link import、自己書換え、runtime code generation。
- reinterpret cast、所有raw pointer、直接new／delete、malloc／free、custom allocator。
- 独自Thread、mutex、condition variable、TLS。
- wall clock、entropy、global mutable state。
- ABI境界を越えるException、外部Dependency。
- 未宣言Component、phase、queue、scratch、CPU／Memory、State owner、Save／Replay意味。

Source scanだけに依存せず、AST、Module graph、object import、link map、binary import、runtime traceを照合する。C++はmemory-safeを証明された言語ではないため「絶対安全」と表示しない。

### 6.4 Bounded Project Shader

Project ShaderのSource／semantic／resource／pass／AI理解／qualification境界は[Project Shader](../06-rendering/project-shader.md)だけが定義する。ProjectはProfile内でHLSL Function／Node／Shading Model／Stage、宣言済みStorage／UAV相当access、複数Pass Techniqueを追加できるが、Manifest外Pass／Resource／side effect、native GPU resource／descriptor／barrier／queue、Compiler option、Engine private includeへ到達できない。

Targetごとにhardware-VMの隔離Workerでoffline compile／validateし、Source contract、compiler fact、reflection、runtime-use trace、fixtureを照合する。Shipping RuntimeでSource生成、download、JITを行わない。既存`ProjectRenderDomainPortV1`で表現できない新Domain、Port自体、Backend、native execution primitiveが必要なら`capability_unavailable`にする。
