# ChatGPT Pro Standalone MCP Route Design

- 状態: `approved-for-implementation`
- 正本範囲: Personal Skill `collaborating-with-chatgpt-pro`のBrowser route、
  同Skillの参照契約とstatic validator、Miraikanai repositoryの`AGENTS.md`
  collaboration destination
- 判断日: 2026-07-31
- 外部根拠確認日: 2026-07-31
- 先行記録:
  [ChatGPT Web Pro MCP Route Clarification Design](2026-07-31-chatgpt-web-pro-mcp-route-clarification-design.md)
- 置換対象: 先行記録のProject内チャット固定routeと、visible Tool cardだけを
  Tool実行の必須証拠とする判断

## 1. 結論

`collaborating-with-chatgpt-pro`のBrowser destinationを、特定Project内の新規
チャットから、`https://chatgpt.com/`で開始する通常のstandalone新規チャットへ
後方互換性なく置き換える。

Browser版ChatGPTの応答性能は引き続きvisible `Pro`へ固定し、Local Artifactは
`G Workspace Readonly`からSecure MCP Tunnel経由で取得する。Browserへの添付、
upload、File内容のprompt貼り付け、Project Source、別app、API、低い応答性能への
fallbackは認めない。

Project memoryとProject chat historyはcontext sourceとして使用しない。Taskに必要な
repository contextはTask Contractのexact allowlistへ入れ、Browser版ChatGPT自身が
Tunnel Toolで取得する。この単一路により、動作しないProject capability bindingを
回避しながら、Local FileをBrowserへ配送しない原則と`Pro`要件を同時に満たす。

## 2. 根拠

### 2.1 OpenAI公式として確認した事実

- [Apps in ChatGPT](https://help.openai.com/en/articles/11487775-apps-in-chatgpt)
  は2026-07-31時点で、app互換性がapp、capability、plan、workspace、modelにより
  異なると説明している。旧文面の「Pro modelsを一律除外」は現行本文にない。
- [Developer mode and MCP apps in ChatGPT](https://help.openai.com/en/articles/12584461-developer-mode-and-mcp-apps-in-chatgpt)
  は、Pro userがdeveloper modeでread／fetch権限のMCPを接続できること、local MCP
  serverにはSecure MCP Tunnelを使用することを説明している。
- [Projects in ChatGPT](https://help.openai.com/en/articles/10169521-projects-in-chatgpt)
  はAppsがProjectsで対応すると説明している。
- [Secure MCP Tunnel](https://developers.openai.com/api/docs/guides/secure-mcp-tunnels)
  は、OpenAI側のqueued MCP requestを`tunnel-client`がpollし、local MCP serverへ
  転送して同じ経路で応答を返すことを説明している。

公式資料は、この個人環境でProject内custom MCP Tool catalogがturnへ注入されない
原因や修正時期を説明していない。従ってProject経路の不整合をOpenAIの意図された
制限とは断定せず、現行環境で再現したproduct-path defectとして扱う。

### 2.2 2026-07-31の実測

通常のstandalone chatでは、応答性能`Pro`のまま`G Workspace Readonly`を利用した
時間帯に、local Tunnel admin telemetryで13件のMCP command forwardを観測した。
ChatGPT responseは`G:\workspace`の一覧、既知のacceptance Artifact群、および
`known.md`本文`MCP MARKDOWN ACCEPTANCE 20260731`を返した。

同じTunnel、app、account、応答性能`Pro`を使い、対象Project内で新規chatを作成して
再試験したところ、Browser版ChatGPTは`list_allowed_directories`と`fetch`が利用可能
Toolにないと報告した。試験前後でTunnelの`tools/call` countは18のまま、forward
eventの増分は0だった。

この差分により、Tunnel、local server、read-only root、Pro modelの一般的なTool
能力ではなく、Project chatのcapability bindingが現在の失敗境界であると判断する。

## 3. 方式比較

| 方式 | 利点 | 欠点 | 判定 |
| --- | --- | --- | --- |
| standalone新規chat + `Pro` + Tunnel read | 現行環境でEnd-to-End実通信を確認済み。Project依存を除去できる | Project memoryを利用しない | 採用 |
| Project内chatを維持してfail closed | Project historyを保持する | 現在Tool catalogがturnへ渡らず、目的を達成できない | 不採用 |
| standaloneで取得後にProjectへ移動または再送 | Project整理とMCP取得を両立できる可能性 | route、state、Evidenceが二重化し、貼り付け・再配送の誤用を招く | 不採用 |

## 4. Breaking route contract

正規route Evidenceを次へ置き換える。

```yaml
chat_route:
  browser: codex-in-app-browser
  destination_type: standalone-chat
  start_url: https://chatgpt.com/
  new_chat_required: true
  project_membership: forbidden
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

旧`project_route`、Project URL、Project title、`project-only`／`disabled` memory
verification、Project chat作成、別Project fallback分岐は削除する。互換alias、dual
schema、旧routeの自動変換は実装しない。

送信前に、現在のchatがProject配下でないこと、visible `Pro`がselectedであること、
collapsed buttonが`Pro`であること、およびexact app identityを確認する。送信後のURL
は通常の`/c/<chat-id>`を許可するが、`/g/g-p-...`配下は成功routeとして扱わない。

## 5. Context and Artifact contract

Project memoryの代わりに、現在のTaskから導出した最小context manifestをprimary prompt
へ含める。promptに含められるのはroot-relative Artifact ID、role、revision、expected
bytes、digest、required Tool、評価基準であり、Local Artifact本文ではない。

Browser版ChatGPTは、Task Contractで許可されたArtifactだけを次のexact read-only Tool
で取得する。

- `list_allowed_directories`
- `list_directory`
- `search`
- `fetch`

必要なOwner chain、focused diff、test Evidence、公式根拠をTaskごとにallowlistする。
Project historyからの暗黙context、Browser attachment、upload、paste、Project Source、
Tunnel writeはauthorityにならない。

## 6. Tool execution Evidence

visible Tool cardは、Browser UIが公開している場合は記録するが、成功の唯一の証拠には
しない。次の境界Evidenceを照合する。

1. 送信前: exact standalone route、`Pro`、app identity、Task Contract。
2. Tunnel: 送信直前の`tools/call` countとlast log sequenceを取得する。
3. 送信後: bounded time window内のsanitized count deltaとforward event増分を確認する。
4. Response: expected Artifact ID、bytes、digest、read completeness、failureを照合する。
5. UI: Tool cardがvisibleならcall名、状態、結果を追加Evidenceとして記録する。

Tunnel telemetryから`tunnel_id`、request ID、Profile内容、credential、Local secretを
Transcriptまたはrepositoryへ保持しない。count、時刻、success／failure、必要なTool名
だけをsanitized Evidenceとして扱う。

Tool cardがなくても、Tunnel call増分とexpected Artifact completenessが一致すれば
Tool実行を確認できる。Tunnel増分が0、Artifact coverageが不完全、またはresponseが
既知情報を推測しただけの場合は`blocked`とする。

## 7. Runtime flow

```text
Skill invoked
  -> derive fixed-deny Task Contract and minimum Artifact manifest
  -> run exact Secure MCP Tunnel lifecycle helper
  -> capture sanitized Tunnel telemetry baseline
  -> open ChatGPT Desktop app built-in Browser at https://chatgpt.com/
  -> create a fresh standalone chat
  -> reject any Project-bound route
  -> select and verify exact visible Pro
  -> select and verify G Workspace Readonly
  -> send one complete metadata-only primary prompt
  -> Browser ChatGPT calls authorized read-only MCP Tools
  -> reconcile Tunnel telemetry delta and response Artifact coverage
  -> complete: adjudicate response locally
  -> incomplete: stop as blocked without fallback
```

## 8. Component changes

Personal Skill `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro`:

- `SKILL.md`: Project route workflowをstandalone routeへ置換する。
- `references/prompt-generation-contract.md`: `chat_route` schema、context manifest、
  pre-send route gateを所有する。
- `references/artifact-delivery-contract.md`: Project Source禁止を維持し、standalone
  routeのTunnel-only completenessを定義する。
- `references/response-completion-gate.md`: Browser Tool cardとsanitized Tunnel telemetry
  のEvidence結合を定義する。
- `references/transcript-contract.md`: Project destination／memory fieldsを削除し、
  standalone routeとtelemetry deltaを記録する。
- `references/adjudication-and-stop-rules.md`: Project routeを禁止し、standalone route、
  `Pro`、app、Tunnel、coverage failureをterminal blockerにする。
- `scripts/validate_secure_mcp_contract.ps1`: 新routeを要求し、旧Project routeを拒否する
  RED／GREEN predicateを追加する。

Miraikanai repository:

- `AGENTS.md`: destinationを通常のstandalone新規chatへ置換し、Project title／memory
  gateを削除する。`Pro`、Tunnel-only read、no-fallbackは維持する。

変更しないもの:

- `scripts/ensure_secure_mcp_tunnel.ps1`
- `scripts/test_ensure_secure_mcp_tunnel.ps1`
- Local Filesystem MCP implementationとexact 4 Tool catalog
- Tunnel Profile、allowed root、Browser upload禁止、Tunnel write禁止
- 過去のdesign、plan、review、実測記録

## 9. Validation strategy

### 9.1 RED

static validatorへ先に次のpredicateを追加し、現行Skillで意図どおり失敗することを
確認する。

- `StandaloneChatRouteRequired`
- `ProjectRouteForbidden`
- `ProjectMemoryGateForbidden`
- `ResponsePerformanceProRequired`
- `ExactBrowserAppIdentity`
- `TunnelTelemetryEvidenceRequired`
- `BrowserUploadsAlwaysDenied`
- `NoRouteFallback`

### 9.2 GREEN

Skill、five reference contracts、validator、repository `AGENTS.md`を最小変更し、全
predicateをpassさせる。既存lifecycle helper testとLocal Filesystem MCP testを再実行
し、固定Profile、auto-start、exact 4 Tool、read-only rootへregressionがないことを
確認する。

### 9.3 Live acceptance

通常の新規chatでvisible `Pro`と`G Workspace Readonly`を選択し、既知acceptance
Artifactに対して`list_allowed_directories`と`fetch`を実行する。送信前後のsanitized
Tunnel metrics／log delta、expected bytes／digest、本文markerを照合する。

同じ受入試験でBrowser attachment、upload、paste、Project Source、Tunnel write、API、
Chrome、Project chat、低い応答性能は使用しない。

## 10. 受入条件

- Skillの唯一のBrowser destinationがfresh standalone chatである。
- Project URL、Project identity、Project memory gateが現行contractから消えている。
- Browser版ChatGPTのvisible `Pro`とcollapsed `Pro`を毎send前に確認する。
- `G Workspace Readonly`と`G:\workspace` read-only scopeを確認する。
- Local Artifact本文をBrowserへattach、upload、paste、Project Sourceで配送しない。
- Taskに必要なcontextはexact Tunnel read allowlistから取得する。
- Tool実行はsanitized Tunnel deltaとArtifact completenessで確認する。
- Tunnel、app、route、`Pro`、Tool、coverage failureで別routeへfallbackしない。
- static validator、Skill validator、lifecycle suite、Local MCP suiteがpassする。
- live acceptanceでstandalone `Pro` turnからknown ArtifactのEnd-to-End readが成功する。
