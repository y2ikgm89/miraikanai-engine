# ChatGPT Pro Consultation Budget Design

- 設計状態: review
- 対象: 個人Global Skill `collaborating-with-chatgpt-pro`
- 判断日: 2026-07-31
- 外部根拠確認日: 2026-07-31
- 先行設計:
  [ChatGPT Pro Secure MCP Tunnel Skill Update Design](2026-07-30-chatgpt-pro-secure-mcp-tunnel-skill-design.md)

## 1. 結論

ChatGPT Pro相談は、探索の開始点ではなく、CodexがローカルEvidenceを整理した
後のmaterialな判断または最終監査に使う。Skillへ
`consultation_budget`を追加し、次を既定にする。

- Codexが先にlocal test、diff、Owner chain、manifest、公式一次資料を整理する。
- Browserへは現在Outcomeに必要な差分とexact targetだけを渡す。
- 最初の送信は、一つのOutcomeを閉じるcomplete promptとして構成する。
- 追加送信は、未完Response、未解決material finding、新しいmaterial Evidence、
  materially changed candidateの場合だけ許可する。
- Tunnel retrievalは、検索、metadata、bounded contentの順で必要最小限に進める。
- Pro allowance warning、Pro unavailable、task-specific elapsed budget超過は
  lower modeへfallbackせず`blocked`にする。
- 正確性、必須Evidence、引用、task固有validationは、loop削減より優先する。

固定の全Task共通turn数、分数、byte数は設けない。Task固有上限は利用者が
与えた値、visible UI、runtime limit、対象Riskからrunごとに導出する。

## 2. 公式根拠と採用判断

### 2.1 OpenAI公式として採用する事実

[Prompting guidance for GPT-5.6 Sol](https://developers.openai.com/api/docs/guides/prompt-guidance-gpt-5p6)
は次を推奨する。

- 重複instruction、不要なexample、Taskと無関係なToolを削減する。
- Outcome、制約、Evidence、成功条件、停止条件を明確にする。
- core requestを支えるEvidenceがそろったら回答し、必須Factが不足する場合だけ
  最小の追加取得を行う。
- retrieval call数の削減を、正確性、必須Evidence、計算、引用より優先しない。
- long-running workflowではroutine Tool callを逐一説明せず、主要phaseだけを
  更新する。

[Apps in ChatGPT](https://help.openai.com/en/articles/11487775-connectors-in)
は、Appが通常のChatGPT plan rate limitに従い、外部Appは独自limitを持ち得る
と説明する。

[About ChatGPT Pro tiers](https://help.openai.com/en/articles/9793128-chatgpt-pro)
は、一部modelに個別usage allowanceがあり、到達時はresetまで一時利用不能に
なり得ること、表示可能な場合はChatGPTがreset時刻を示すことを説明する。

[Secure MCP Tunnel](https://developers.openai.com/api/docs/guides/secure-mcp-tunnels)
はTunnelをprivate MCP requestのtransportとして説明するが、Global Skill固有の
context budget、追加round条件、監査分割方式、固定料金を規定しない。

### 2.2 本Skillの採用判断

次は公式仕様ではなく、上記の公式原則を本Workflowへ適用する利用者判断である。

- local-first delta audit
- one complete primary prompt
- conditional follow-up gate
- search → metadata → bounded content
- task-derived budget
- Pro unavailable時のfail closed

Skillはこれらを「OpenAI公式が定める固定Workflow」と表現しない。

## 3. 方式比較

| 方式 | 利点 | 欠点 | 判定 |
| --- | --- | --- | --- |
| 全Task共通の固定上限 | 予測しやすく速い | 大規模・高Risk監査を誤って打ち切る | 不採用 |
| local-first adaptive budget | Evidence完全性と利用効率を両立 | Task Contractと観測値が必要 | 採用 |
| Proによる全Repository反復監査 | 探索余地が大きい | context、allowance、時間、Scope露出が増える | 明示的なexhaustive Taskだけ |

## 4. Task Contract

`task_contract`へ次を追加する。

```yaml
consultation_budget:
  strategy: local-first-delta-audit
  primary_send: one-complete-outcome-prompt
  follow_up_gate:
    - incomplete-response
    - unresolved-material-finding
    - new-material-evidence
    - materially-changed-candidate
  retrieval:
    target_mode: exact-target-only
    order:
      - search
      - metadata
      - bounded-content
    recursive-enumeration: denied-by-default
    bulk-read: denied-by-default
  runtime_caps:
    max_elapsed_minutes: task-derived | user-provided | unspecified
    max_browser_turns: task-derived | user-provided | unspecified
    max_returned_bytes: runtime-observed | task-derived | unspecified
```

`one-complete-outcome-prompt`は総round数を1へ固定しない。最初の送信へGoal、
Evidence、評価軸、成功条件、output shape、stop ruleをまとめ、既知情報を小分けに
追送信しないという構成規則である。

`task-derived`はRisk、対象規模、必須Evidence、利用者のterminal conditionから
導出する。Budgetが必須Outcomeを満たせない場合、品質を黙って下げず、利用者へ
最小のScope／時間判断を求める。

## 5. Local-first data flow

```text
user outcome
  -> Codex local authority checks
  -> diff / canonical Owner / dependency closure / official source selection
  -> exact evidence manifest
  -> Browser Pro complete primary prompt
  -> conditional targeted retrieval or follow-up
  -> local adjudication and validation
  -> finish or blocked
```

Local-first stageではTaskに該当する検査だけを行う。Browser監査を置換するための
長大なsummaryは作らず、正本Artifact、revision、path、hash、局所的なdiffを
保持する。

Repository全体のexhaustive auditが明示された場合、Codexが独立Scopeへ分割し、
各Scopeのrequired Evidenceと完了条件を定義する。単一chatへ無制限に追加せず、
distinct outcomeだけfresh chatへ分離する。

## 6. Tunnel retrieval budget

Tunnel rootが広くても、Task Contractのexact target allowlistをretrieval境界に
する。次の順で取得する。

1. 短い識別語で検索し、候補pathと件数を得る。
2. 必要な候補だけmetadata、revision、sizeを確認する。
3. 判断に必要な範囲だけbounded contentを読む。
4. core requestを支えるEvidenceがそろった時点で取得を止める。

recursive tree、bulk multi-file read、全文再読は既定で使わない。次の場合だけ
Task Contractへ明示して許可する。

- exhaustive coverageが成功条件である。
- 対象の分割または検索ではmaterialな見落としRiskを閉じられない。
- runtime limitとauthorizationが対象を完全に覆う。

MCP serverがbounded readを提供しない場合、完全File読取の必要性を評価し、
不要ならchat deliveryまたは`blocked`を選ぶ。Tool名だけからbounded behaviorを
推測しない。

## 7. Browser Proとusage gate

既存のBrowser、Project、memory、model、Pro preflightへ次を追加する。

```yaml
resource_observations:
  pro_allowance_state: available | warning | unavailable | unknown
  reset_time: observed value | unavailable
  elapsed_seconds: observed
  browser_turns: count
  tunnel_tool_calls: count
  returned_bytes: observed | unknown
```

Browser UIがtoken数を表示しない場合、token消費を推定値で埋めない。
`unknown`として、turn、Tool call、observed bytes、elapsed timeを記録する。

Proが利用不能またはreset待ちの場合、Standard、Instant、APIへ自動fallback
しない。利用者が別Task Contractを明示しない限り`blocked`とする。

## 8. Follow-upと停止

追加送信は`follow_up_gate`のいずれかを、visible Responseまたは新Evidenceで
確認した場合だけ行う。同意確認、表現改善、同じ指摘の再質問、固定clean回数の
ために追加しない。

各Result後に次を判定する。

1. core requestを必要Evidence付きで回答できるか。
2. required finding、Owner、date、ID、引用、Artifactが不足しているか。
3. Candidateがmaterialに変化したか。
4. Task固有budget内で次の取得が最小の有効fallbackか。

1が真で2と3が偽なら終了する。必要Evidenceが不足し、budgetまたはPro allowance
で取得不能なら`blocked`とする。

## 9. 更新範囲

Personal Skillだけを更新する。

- `SKILL.md`: local-first stage、usage gate、停止規則のcoreだけ
- `references/prompt-generation-contract.md`: `consultation_budget` schema
- `references/artifact-delivery-contract.md`: retrieval orderとbulk例外
- `references/transcript-contract.md`: `resource_observations`
- `scripts/validate_secure_mcp_contract.ps1`: 新しい契約predicate

`agents/openai.yaml`、Repository `AGENTS.md`、Project route、Tunnel設定、
Filesystem MCP server実装は変更しない。

## 10. Failure処理

| 状態 | 処理 |
| --- | --- |
| Pro allowance warning | Outcome規模とtask-derived budgetを再評価 |
| Pro unavailable / reset待ち | lower modeへfallbackせず`blocked` |
| elapsed budget超過 | active generationを中断せずcompletion gate後に停止 |
| broad retrievalしかない | 必要性を証明できなければfallbackまたは`blocked` |
| required Evidence不足 | 最小の追加取得だけ許可 |
| core request充足済み | 追加roundを行わず終了 |
| exhaustive Task | Scopeを分割し、独立完了条件を持たせる |

Unexpected write、secret、Scope外取得は既存のfail-closed規則を維持する。

## 11. 検証設計

### 11.1 RED

現行Skillをfresh contextへ与え、Pro allowance warningと大規模Repository監査が
同時にあるscenarioを実行する。次のどれかを観測できた場合にREDとする。

- local diff／Owner整理前にBrowser retrievalを開始する。
- exact targetなしでrecursive treeまたはbulk readを選ぶ。
- known Evidenceを複数Promptへ分ける。
- material triggerなしで追加roundを計画する。
- allowance、elapsed、Tool call、returned bytesを記録しない。

### 11.2 GREEN

同じscenarioで次を確認する。

- local-first delta auditを選ぶ。
- complete primary promptを構成する。
- search、metadata、bounded contentを順に選ぶ。
- follow-upを4つのmaterial conditionへ限定する。
- visible resource observationをTranscriptへ残す。
- Pro unavailable時にlower modeへfallbackしない。

### 11.3 Test量

Live Browser Proは検証に使わない。利用枠を消費しないfresh agent application
testとして、no-guidance controlと更新Skillを各5件実行する。各sampleの完全な
payload、response、agent ID、model、timestamp、SHA-256、判定を一時Evidenceへ
保存する。

Static検証では次を確認する。

- budget schemaとretrieval order
- follow-up gateとPro fail-closed
- resource observation fields
- 全Task共通の固定数値上限がない
- 既存のTunnel authorization、write incident、model、route leak、料金保証防止
- relative link、Skill metadata、`quick_validate.py`

## 12. 受入条件

- Browser Pro相談がlocal-firstのmaterial auditへ限定される。
- known Evidenceを小分け追送信せずcomplete primary promptへまとめる。
- Tunnelはexact targetをsearch → metadata → bounded contentで取得する。
- recursive treeとbulk readはexhaustive成功条件がない限り選ばれない。
- follow-upはobservableなmaterial triggerだけで行う。
- Pro allowanceとelapsed stateを観測し、token数を捏造しない。
- Pro unavailable時にlower modeへ黙示fallbackしない。
- 固定のglobal turn、minute、byte capを設けない。
- 正確性、required Evidence、引用、validationをloop削減より優先する。
- 既存のauthorization、privacy、unexpected-write、local verification契約を
  後退させない。
- 公式事実と本Skillの採用判断が分離される。
