# ChatGPT Pro Tunnel-only Delivery Design

- 設計状態: approved
- 対象: 個人Global Skill `collaborating-with-chatgpt-pro`
- 判断日: 2026-07-31
- 外部根拠確認日: 2026-07-31
- 先行設計:
  [ChatGPT Pro Secure MCP Tunnel Auto-start Design](2026-07-31-chatgpt-pro-tunnel-auto-start-design.md)

## 1. 結論

`collaborating-with-chatgpt-pro`のLocal Artifact配送を、Secure MCP
Tunnel-backed `G Workspace Readonly`によるTool readだけへ固定する。
Browser ChatGPTへのFile添付、upload、File内容のprompt貼り付け、archive、
split parts、review bundle、Project Sourceは廃止する。

Task Contractは`browser_file_uploads.mode: denied`を固定値とし、
`artifact.delivery`とTranscriptのEvidence delivery modeは
`secure-mcp-tunnel`だけを受け入れる。既存のchat delivery mode、receipt-only
batch、`BEGIN_REVIEW`、Tunnel failure時のchat fallbackは後方互換性を残さず
削除する。

Browser ChatGPTでは、対象Projectのcomposerにある`応答性能`controlから
visibleな`Pro`を明示選択する。Profileの`Pro`plan badge、推測されたmodel、
`非常に高い`その他の応答性能、selector非表示時の代替証拠は成功条件にしない。

Tunnel停止時は既存のlifecycle helperで起動する。起動、Browser app、
allowed root、target reachability、required read Tools、complete read coverage、
または`Pro`選択のいずれかを検証できない場合、Browser file deliveryへ
切り替えず`blocked`で終了する。

## 2. 根拠と採用判断

### 2.1 OpenAI公式として採用する事実

[GPT-5.6 in ChatGPT](https://help.openai.com/en/articles/20001354-gpt-5-6-in-chatgpt)
は、ChatGPTのmodel pickerでspeedとreasoning levelを選べること、および
`Pro`が難しいTaskと長時間Workflow向けの選択肢であることを説明している。
利用可能な選択肢はplanとworkspace access設定に依存する。

[Secure MCP Tunnel](https://developers.openai.com/api/docs/guides/secure-mcp-tunnels)
は、ChatGPT app discoveryとMCP Tool callが稼働中の`tunnel-client`に
依存すること、Named Profileの起動、health／ready endpoint、および
Client非接続時のrequest failureを説明している。

公式資料は、この個人Skillにおける特定Project、特定Tunnel Profile、
Browser upload禁止、またはfail-closed policyを規定しない。

### 2.2 2026-07-31のBrowser実測

Codex in-app Browserで対象Projectを観測し、composerのresponse selectionだけを
`Pro`へ変更してselected stateを確認した。

- Project title:
  `AIネイティブC++ゲームエンジンプロジェクト`
- plan badge: `Pro`
- composerのcontrol名: `応答性能`
- 観測した選択肢:
  `最速 5.5`、`中程度`、`高い`、`非常に高い`、`Pro`
- 初期観測値: `非常に高い`
- `Pro`選択後のauthoritative signal:
  collapsed composer buttonのvisible textが`Pro`
- Browser app display name: `G Workspace Readonly`
- Browser app description:
  `Read-only access to G:\workspace via local Filesystem MCP.`
- Browser Tool menuに`Secure MCP Tunnel`というdisplay nameはなかった。

確認時はmessage、File添付、upload、Project Source追加を行っていない。
Browser appの表示は接続identityの証拠であり、Tunnel health、Tool
availability、target reachability、complete readの証拠ではない。

### 2.3 本Skillの採用判断

次はOpenAIの固定仕様ではなく、この個人環境向けのProject decisionである。

- Browserの正規Filesystem MCP appを`G Workspace Readonly`へ固定する。
- Local Profileを`g-workspace-readonly`へ固定する。
- Browser file delivery authorityを常にdenyする。
- Local Artifact contentはpromptへ複製せず、ChatGPTがToolで取得する。
- 応答性能はvisibleな`Pro`を直接選択し、選択後のbutton textで再確認する。
- すべてのTunnel／Tool／coverage failureを`blocked`へ閉じる。
- 旧chat delivery schemaとfallback branchの互換層を置かない。

## 3. 方式比較

| 方式 | 利点 | 欠点 | 判定 |
| --- | --- | --- | --- |
| exact app identity + Tool capability + Tunnel-only | 誤配送を防ぎ、File contentをBrowser uploadへ複製しない | Tunnelまたはapp障害時にconsultationを継続できない | 採用 |
| app名を問わないcapability-only routing | app renameへ強い | 別Filesystem appへの誤routeを許し得る | 不採用 |
| Tunnel優先 + chat fallback | 可用性が高い | Userのno-upload要件と単一路契約に反する | 不採用 |

## 4. Breaking contract

### 4.1 Task Contract

正規shapeを次へ置き換える。

```yaml
task_contract:
  browser_file_uploads:
    mode: denied
    allowlist: []
  tunnel_reads:
    mode: scoped | specific
    allowlist: []
  tunnel_writes:
    mode: denied
    allowlist: []
```

`browser_file_uploads.mode`の`scoped`と`specific`は削除する。
`browser_file_uploads.allowlist`は常に空で、User request、retry、Tunnel
failure、size limit、format不足、deadlineによるoverrideを認めない。

`tunnel_reads.allowlist`だけがLocal Artifactのread authorityを表す。
Directory scopeはcanonical absolute rootとTask-local artifact IDへ展開し、
Taskに必要な最小範囲へ限定する。

### 4.2 Artifact manifest

正規manifestを次へ置き換える。

```yaml
artifact:
  id: stable task-local identifier
  repo_relative_path: path shown to ChatGPT
  canonical_path: verified path under allowed root
  role: why this artifact is material
  revision: commit, version, or not-applicable
  sensitivity: non-sensitive | approved-private | blocked
  delivery: secure-mcp-tunnel
  expected_bytes: integer
  sha256: lowercase hex
  required_tools: [list, read]
```

`chat-attachment`、`inline-excerpt`、`review-bundle`、`split-parts`、
`project-source`はenumから削除する。`chat_delivery`branch、attachment
batch、receipt-only prompt、`BEGIN_REVIEW`も削除する。

### 4.3 Project route

正規route Evidenceを次へ置き換える。

```yaml
project_route:
  browser: codex-in-app-browser
  url: exact ChatGPT Project URL
  expected_title: visible Project title
  required_memory: project-only | disabled
  response_performance:
    control_display_name: 応答性能
    required_option: Pro
    option_visibility: visible
    option_selected: true
    collapsed_button_text: Pro
  browser_app:
    display_name: G Workspace Readonly
    description_root: G:\workspace
    access: read-only
  fallback: deny
```

`model_selection_mode: product-selected-pro`と
`quality_reasoning_control_*`は削除する。Model display labelを別に観測
できる場合はTranscriptへ補助Evidenceとして記録できるが、`Pro`選択の代替
証拠にはしない。Model versionはpinしない。

### 4.4 Transcript

Transcriptのrouteとdelivery Evidenceを次へ置き換える。

```yaml
route:
  browser: codex-in-app-browser
  destination: ChatGPT Project title
  memory: project-only | disabled
  execution_date: YYYY-MM-DD
  response_performance_control: 応答性能
  response_performance_selected: Pro
  response_performance_verified: true
  model_display_name: observed visible label | not-exposed
  browser_app_display_name: G Workspace Readonly
evidence_delivery:
  mode: secure-mcp-tunnel
  browser_file_uploads: denied
  upload_attempted: false
  tunnel_profile: g-workspace-readonly
  allowed_root: G:\workspace
  required_tools: [list, read]
  tool_calls: []
  expected_artifact_ids: []
  read_artifact_ids: []
  failed_artifact_ids: []
  completeness: complete | incomplete
```

旧Transcript shapeを正規化するmigrationまたはdual-writeは実装しない。

## 5. Runtime flow

```text
Skill invoked
  -> create exact Task Contract and Tunnel-only artifact manifest
  -> assert browser_file_uploads is denied and empty
  -> run fixed-profile lifecycle helper
     -> already-running | started: continue
     -> blocked: stop
  -> open Codex in-app Browser
  -> verify sign-in, Project title, and memory mode
  -> open response performance menu
  -> select exact visible option Pro
  -> verify collapsed button text Pro
  -> verify G Workspace Readonly identity and read-only G:\workspace scope
  -> activate that app for the new Project chat
  -> send one complete primary prompt containing paths and manifest metadata,
     but no Local Artifact content
  -> ChatGPT calls required list/read Tools
  -> reconcile expected IDs, read IDs, failures, bytes, and digest evidence
  -> complete: begin requested analysis in the same response
  -> incomplete or Tool failure: blocked
  -> adjudicate locally
```

Primary promptはGoal、scope、manifest、review criteria、output schema、
stop ruleを最初から含む。Local File内容をpromptへ貼らない。ManifestのPath、
role、revision、size、digestはcontentではなく配送整合性metadataとして
Task Contractの範囲内で送る。

Tunnel-only modeではchat receipt phaseを作らない。ChatGPTは同じprimary
response内でTool read completenessを確認し、completeな場合だけ分析を行う。

## 6. Failure policy

次のいずれかで即時`blocked`とする。

- lifecycle helperが`already-running`または`started`以外を返す。
- Browser app identityまたはread-only rootを可視確認できない。
- `G:\workspace`または対象Artifactへ到達できない。
- required `list`／`read` Toolがない、拒否される、または失敗する。
- expected Artifactとread Artifactが一致しない。
- expected bytesまたはdigest Evidenceが一致しない。
- unexpected write Toolまたはwrite side effectを観測する。
- 対象Project、memory mode、またはsign-inを確認できない。
- `応答性能`control、visible `Pro` option、selected state、またはcollapsed
  `Pro` buttonを確認できない。
- Pro allowanceが利用できない。

`blocked`時はFile chooserを開かず、Fileをattach／uploadせず、Local File内容を
promptへ貼らず、archiveまたはProject Sourceを作らず、別Project、Chrome、
standalone chat、API、Standard、Instant、その他の低い応答性能へ切り替えない。

Unexpected writeを観測した場合は新規Tool callを止め、変更Pathを記録し、
Codexがlocal diffを検証する。User approvalなしにBrowser ChatGPT由来の変更を
採用しない。

## 7. Component changes

Personal Skill:

- `SKILL.md`
  - WorkflowをTunnel-onlyとexact `Pro` verificationへ簡素化する。
- `references/prompt-generation-contract.md`
  - fixed-deny upload authority、exact route、preflightを定義する。
- `references/artifact-delivery-contract.md`
  - legacy chat modesを削除し、Tunnel manifestとcompletenessだけを所有する。
- `references/transcript-contract.md`
  - Tunnel-only deliveryとexact response performance Evidenceへ置き換える。
- `references/adjudication-and-stop-rules.md`
  - Tunnel、app、Tool、coverage、`Pro` failureをterminal blockerへ追加する。
- `references/response-completion-gate.md`
  - `BEGIN_REVIEW`を削除し、primary／follow-up completionだけを扱う。
- `scripts/validate_secure_mcp_contract.ps1`
  - 新しいbreaking contractを静的検証し、legacy branchを拒否する。

変更しないもの:

- `scripts/ensure_secure_mcp_tunnel.ps1`
- `scripts/test_ensure_secure_mcp_tunnel.ps1`
- `agents/openai.yaml`
- Tunnel Profile内容
- Filesystem MCP実装
- Repository `AGENTS.md`

## 8. Validation strategy

### 8.1 RED

Validatorへ先に次のpredicateを追加し、現行Skillで失敗することを確認する。

- `BrowserUploadsAlwaysDenied`
- `TunnelOnlyArtifactDelivery`
- `NoLegacyChatDeliverySchema`
- `TunnelFailureBlocks`
- `ExactBrowserAppIdentity`
- `ResponsePerformanceProRequired`
- `NoProductSelectedProFallback`
- `TranscriptTunnelOnlyEvidence`

### 8.2 GREEN

ContractとSkillを最小変更し、全predicateをpassさせる。既存lifecycle helper
test 28件を再実行し、auto-start behaviorへregressionがないことを確認する。

### 8.3 Skill validation

- `validate_secure_mcp_contract.ps1`
- `test_ensure_secure_mcp_tunnel.ps1`
- `quick_validate.py`
- YAML metadata parse
- direct relative reference existence
- legacy enum／field scan
- complete changed-file diff review
- `git diff --check`
- `git status --short`
- `git diff --stat`

Live Browser acceptanceではFile添付またはuploadを行わない。新規Project chatで
`Pro` selected state、app identity、Tunnel Tool read、manifest completenessを
確認する。Message送信前にTask Contractのscopeを再確認する。

## 9. 受入条件

- Browser file upload authorityが固定`denied`である。
- Local Artifact contentをBrowser promptへ複製しない。
- Artifact delivery enumが`secure-mcp-tunnel`だけである。
- Tunnel停止時は既存helperが固定Profileを起動する。
- Local ready後にBrowser app、root、Tool、target、coverageを別々に検証する。
- Browser appはexact `G Workspace Readonly`として検証される。
- 応答性能のvisible `Pro`を直接選択する。
- collapsed composer buttonの`Pro`表示をauthoritative signalとして記録する。
- Profile plan badgeまたはselector非表示を`Pro`選択証拠にしない。
- Tunnelまたは`Pro` failureでchat deliveryへfallbackしない。
- Legacy chat schema、receipt protocol、compatibility layerが残らない。
- Transcriptがno-upload、Tunnel completeness、exact `Pro` Evidenceを保持する。
- Validator、helper test、Skill validator、repository verificationがpassする。
