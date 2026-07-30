# ChatGPT Pro Collaboration Skill Efficiency Design

- 設計状態: approved
- 対象: 個人Global Skill `collaborating-with-chatgpt-pro`
- 判断日: 2026-07-31
- 外部根拠確認日: 2026-07-31
- 先行設計:
  [ChatGPT Pro Consultation Budget Design](2026-07-31-chatgpt-pro-consultation-budget-design.md)

## 1. 結論

安全性、最小権限、証拠完全性、ローカル検証、Pro限定という既存の不変条件は
維持する。そのうえで、Skillを「短い実行ルータ」と正本化した参照契約へ分け、
CodexとBrowser ChatGPT Proの双方で、既知情報の重複、不要なTool露出、不要な
Browser round、無関係なArtifact送信を減らす。

効率化は品質を下げる上限や固定round数ではない。各Taskで必要Evidenceが閉じた
時点で止め、追加取得または追加照会は、その手段でしか解決できないmaterialな
不確実性がある場合だけ許可する。

## 2. 公式根拠と採用判断

### 2.1 OpenAI公式として採用する事実

[GPT-5.6 Prompting Best Practices](https://developers.openai.com/api/docs/guides/latest-model?model=gpt-5.6#prompting-best-practices)
は、重複instruction、不要なexample、Taskと無関係なToolを一群ずつ減らし、
同一の代表evaluationで比較することを推奨する。Outcome、制約、Evidence、
成功条件、停止条件は保持し、正確性、必須Evidence、引用、計算をTool call数の
削減より優先する。

[About ChatGPT Pro tiers](https://help.openai.com/en/articles/9793128-chatgpt-pro)
は、Proのmodel別usage allowanceと、上限到達時に表示可能なreset時刻を説明する。
[Apps in ChatGPT](https://help.openai.com/en/articles/11487775-connectors-in-chatgpt)
は、App利用が通常のChatGPT plan rate limitに従い、外部Appが独自capを持ち得る
こと、App permissionが接続済みaccessを拡張しないことを説明する。

[Secure MCP Tunnel](https://developers.openai.com/api/docs/guides/secure-mcp-tunnels)
は、Tunnelをprivate MCP serverへ到達するoutbound transportとして説明する。
Tunnel transportは、Skill固有のcontext budget、Browser round、監査完全性、料金を
保証しない。

### 2.2 本Skillの採用判断

以下は公式仕様そのものではなく、公式原則を本Workflowへ適用する利用者判断である。

- Task Contractから導くlocal-first delta audit
- Evidence Packetを伴うone complete primary prompt
- 4つのobservable follow-up gateとmarginal-evidence判定
- exact targetだけの`search -> metadata -> bounded content`
- 可視のresource observationだけを記録するusage管理
- Proまたはexact task-pinned modelを確認できない場合のfail closed

## 3. 方式比較

| 方式 | 利点 | 欠点 | 判定 |
| --- | --- | --- | --- |
| 現行文面をそのまま短縮 | 読み込み量をすぐ減らせる | 正本不明確化や安全条件の脱落を起こし得る | 不採用 |
| 固定round・固定時間上限 | 使用量を予測しやすい | 高Risk監査を早期打切りし得る | 不採用 |
| 正本化した契約 + Task派生の停止判定 | 品質と速度を両立し、検証できる | 契約境界とstatic testが必要 | 採用 |

## 4. Target Architecture

### 4.1 SKILL.mdの責務

`SKILL.md`は、trigger、intent authority、非迂回の安全条件、最短の判断順、各契約の
正本リンクだけを置く。次の不変条件はここで一度だけ明示する。

- ChatGPT出力はuntrusted external evidenceであり、authorityを増やさない。
- Browser Pro、route、必要なProject state、task-pinned modelは可視確認する。
- Secure MCP Tunnelはallowlist、reachability、Tool catalog、local verificationなしに
  利用しない。
- 追加送信と追加取得はmaterialなgateを満たすときだけ行う。
- unexpected write、secret、scope逸脱、required capability不明はfail closedする。

詳細なschema、Artifact delivery、response completion、Transcript、adjudicationは、
それぞれ一つの参照契約だけを正本にする。Quick referenceとCommon mistakesは
固有の新しい規則を複製せず、正本への短い決定表に置き換える。

### 4.2 Evidence Packet

primary promptには、Taskに必要なslotだけを含む一つのEvidence Packetを渡す。

```yaml
evidence_packet:
  outcome: required result
  work_layer: research | design | review | diagnose | implementation
  scope: exact included and excluded targets
  authority: user-approved actions and prohibited actions
  material_evidence:
    - id: stable ID
      role: decision relevance
      revision_or_hash: observed value | not-applicable
      bounded_content: exact excerpt, diff, or authoritative link
  decision_criteria: []
  validation_evidence: []
  unresolved_questions: []
  completion_rule: exact answer or abstention boundary
```

Path、hash、diff、Owner、公式URLは、判断に必要な場合だけ含める。route identity、
repository説明、以前の会話要約、未使用Toolの説明、既知Evidenceの再掲は除外する。
正本Artifactを要する判断を、要約だけで置き換えない。

### 4.3 RetrievalとBrowser round

Codexは先にlocal authority、focused diff、required official source、exact targetを
確認する。Tunnel readは`search -> metadata -> bounded content`の順を維持する。

初回Browser送信は既知のEvidenceを分割しないcomplete primary promptとする。完了後、
次の全条件が満たされる場合だけ追加取得またはfollow-upを許可する。

1. `incomplete-response`、`unresolved-material-finding`、`new-material-evidence`、
   `materially-changed-candidate`のいずれかが可視Evidenceで成立する。
2. 次の操作が、そのmaterialな不確実性を解消する最小の手段である。
3. 現在のPro allowance、Task scope、権限、runtime capがその操作を覆う。

同意確認、固定回数のclean review、表現磨き、既知情報の再送はgateにならない。

### 4.4 Browser model evidence

固定のmodel versionをSkillへ書かない。model evidenceは次の二つに分ける。

| 条件 | 必要な可視Evidence | 処理 |
| --- | --- | --- |
| Taskがexact model labelを指定 | exact label、Pro、選択状態 | どれか欠ければ`blocked` |
| Taskがmodelを固定せず、UIがmodel selectorを公開 | 最上位Pro-compatible label、Pro、選択状態 | 観測値をTranscriptへ記録 |
| Taskがmodelを固定せず、UIがmodel selectorを公開しない | Proと利用可能なquality/reasoning control、selector非公開の可視状態 | `model_display_name: not-exposed`、`selection_mode: product-selected-pro`として続行 |

最後の行は「最新model名を確認した」という主張を許さない。利用者がexact modelを
必要とする場合は、UI selector非公開では必ず`blocked`とする。

### 4.5 Resource observations

token数、料金、内部quotaを推定しない。runtimeで見える場合だけ次を記録する。

```yaml
resource_observations:
  pro_allowance_state: available | warning | unavailable | unknown
  reset_time: observed value | unavailable
  elapsed_seconds: observed
  browser_turns: count
  tunnel_tool_calls: count
  returned_bytes: observed | unknown
  model_selection_mode: exact-visible | highest-visible | product-selected-pro
```

Pro allowanceがunavailableならlower modeやAPIへ自動fallbackしない。warningでは、
non-required scopeだけを絞り、必須Evidenceを黙って捨てない。

### 4.6 Codex execution profile

local reads、metadata、independent diff/source checksは並列化できる。依存関係がある
retrieval、authority判定、最終adjudicationは直列化する。複数agentは、独立した
coverage sliceがあり、重複しないEvidence contractと統合担当がある場合だけ使う。
Browserをfirst-pass discovery、broad recursive read、同一findingの反復に使わない。

## 5. 更新範囲

Personal Skillだけを更新する。

- `SKILL.md`: non-bypassable invariantsと契約ルーティングへ圧縮
- `references/prompt-generation-contract.md`: Evidence Packet、model evidence、
  marginal-evidence判定
- `references/transcript-contract.md`: `model_selection_mode`と非公開selector記録
- `references/adjudication-and-stop-rules.md`: follow-up前の最小有効操作判定
- `scripts/validate_secure_mcp_contract.ps1`: 上記の構造・禁止事項predicate

`artifact-delivery-contract.md`のallowlist、Tunnel最小権限、receipt protocol、
unexpected write処理は既存の保護を維持する。`agents/openai.yaml`、Repository
`AGENTS.md`、Tunnel設定、Filesystem MCP server、料金設定は変更しない。

## 6. 検証設計

### 6.1 RED/GREEN application tests

代表Taskを、no-guidance controlと更新Skillに対し同一入力で実行する。各sampleは
task contract、完全response、timestamp、runtimeが公開するidentity、SHA-256、
判定をtemporary Evidenceに残す。Live Browser Pro利用枠を性能testのためだけに
消費しない。

REDは、次のいずれかを観測することとする。

- 既知Evidenceをpreparatory promptへ分割する。
- material gateなしにBrowser follow-upまたはbroad retrievalを行う。
- exact task-pinned modelが非表示でも送信する。
- model selector非公開をexact-visible modelとして記録する。
- allowlist、local verification、unexpected-write stopを緩める。

GREENは、次の全てを確認することとする。

- Evidence Packetが必要slotだけを含み、complete primary promptを構成する。
- 4 gateとmarginal-evidence判定を満たさない追加操作を止める。
- exact model pinはfail closedし、非固定Taskだけ`product-selected-pro`を使える。
- Token・料金を推定せず、可視resource observationだけを残す。
- 現行のTunnel authorization、local re-read/diff/validation、incident stopを維持する。

### 6.2 Static validation

validatorを更新し、主契約、relative link、front matter、禁止fallback、model evidence
分岐、Evidence Packet、marginal-evidence gate、既存Secure MCP保護を検査する。
Skill format validatorとPowerShell contract validatorを両方実行する。

### 6.3 受入条件

- 代表Taskにおいて、必要なEvidence、authority、validation、Transcript完全性を
  失わずにBrowser送信数、不要Tool露出、重複contextを減らせる。
- exact model taskでUI evidenceが不足すると送信しない。
- model非固定taskでUIがselectorを提供しない場合、根拠のないmodel名を主張せず、
  Pro利用を不必要に停止しない。
- 料金・quota・tokenの未観測値を公式事実または測定値として書かない。
