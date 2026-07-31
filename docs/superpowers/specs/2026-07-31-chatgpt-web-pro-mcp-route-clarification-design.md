# ChatGPT Web Pro MCP Route Clarification Design

- 状態: `approved-for-implementation`
- 正本範囲: Personal Skill `collaborating-with-chatgpt-pro`の現行route contract、
  同Skillの参照契約、static validator、Miraikanai repositoryの現行`AGENTS.md` route
  instruction
- 判断日: 2026-07-31
- 外部根拠確認日: 2026-07-31
- 依存: [Model Context Protocol | ChatGPT Learn](https://learn.chatgpt.com/docs/extend/mcp)
- 先行記録:
  [ChatGPT Pro Readonly Local Files Design](2026-07-31-chatgpt-pro-readonly-local-files-design.md)

## 1. 結論

Secure MCP TunnelのMCP clientはブラウザ版ChatGPTである。CodexはChatGPT Desktop
appの内蔵Browserを操作してブラウザ版ChatGPTを検証するcontrol planeであり、Tunnelを
直接利用するMCP clientではない。

ブラウザ版ChatGPTで`G Workspace Readonly` appを使う各turnは、visible `応答性能`
`Pro`を選択し、collapsed buttonが`Pro`であることを確認してから送信する。`非常に高い`
はfallback、substitute、互換modeとして使用しない。`Pro`、app identity、Tool callの
いずれかを確認できない場合は`blocked`とする。

## 2. 用語と責務境界

| Component | Responsibility | Does not do |
| --- | --- | --- |
| ChatGPT Desktop app内蔵Browserを操作するCodex | Browser route、visible state、app、Tool evidenceを検証する | Secure MCP TunnelをMCP clientとして直接呼ばない |
| ブラウザ版ChatGPT | `G Workspace Readonly` appを有効化し、MCP Toolを呼ぶ | Local Artifactをbrowser attachment、upload、pasteで受け取らない |
| `G Workspace Readonly`／Secure MCP Tunnel | ブラウザ版ChatGPTのread-only Tool callを固定rootのlocal serverへ転送する | write Tool、alternate root、chat-delivery fallbackを提供しない |
| Local read-only MCP Server | `G:\workspace`をexact 4 read-only Toolで読む | BrowserまたはCodexの応答性能を選択しない |

`chatgpt.com`はChatGPT Desktop app内蔵Browserで開くdestinationである。`Codex in-app
Browser`はCodexが操作するBrowser capabilityの名称であって、Tunnel利用主体を指さない。

## 3. 応答性能の分類

| Claim | Classification | Evidence／owner |
| --- | --- | --- |
| ChatGPT Desktop app、Codex CLI、IDE extensionはCodex hostのMCP configurationを共有する。ChatGPT webはlocal Codex configurationを読まない。 | `official-spec` | [OpenAI MCP documentation](https://learn.chatgpt.com/docs/extend/mcp) |
| Browser版ChatGPTのcurrent custom app turnは`応答性能: Pro`、collapsed `Pro`で送信する。 | `user-requirement` | current user instruction |
| `Pro`を確認できない場合、`非常に高い`／`高い`／`中程度`／APIへ変更せず`blocked`にする。 | `project-decision` | least-privilege and no-fallback contract |
| 2026-07-31に`Pro`でToolが注入されなかった、`非常に高い`で一部Tool injectionを観測した。 | `measured`／historical | preceding design; it does not prove current behavior or override the user requirement |

OpenAIの公開資料はこの個別Browser UIの`Pro`選択を一般推奨として規定しない。従って
`Pro`固定はofficial recommendationではなく、このtaskの明示要件として扱う。

## 4. Required behavior

1. CodexはChatGPT Desktop appの内蔵Browserだけを操作し、Chrome、API、別Projectへ
   substituteしない。
2. Browser版ChatGPTでexact Project、memory mode、`応答性能: Pro`、collapsed `Pro`、
   `G Workspace Readonly`のread-only identityを可視確認する。
3. Browser版ChatGPTが`list_allowed_directories`、`list_directory`、`search`、`fetch`を
   呼び、Secure MCP Tunnel経由でlocal serverが返す結果をcoverage contractで照合する。
4. `Pro`、app、Tool catalog、Tool success、root、coverageのどれかが欠ける場合は
   `blocked`として終了する。`非常に高い`へのfallback、attachment、upload、paste、
   Project Source、alternate app、Tunnel writeは許可しない。

## 5. Change scope

変更対象はcurrent behaviorを定義するPersonal Skill、five reference contracts、static
validator、current repository `AGENTS.md`である。過去の計画、実測、review evidenceは
歴史的記録として書き換えない。このdocumentが、過去の`非常に高い` project decisionを
置き換える後方互換性のないsuccess criterionである。

## 6. Verification

- static validatorを先に`Pro`／Browser版ChatGPT／control-plane ownershipを要求する形へ
  更新し、旧Skillに対して失敗することを確認する。
- Skill、参照契約、`AGENTS.md`を最小変更する。
- static validator、Skill validator、Tunnel lifecycle suite、package self-testを通す。
- 新規Browser版ChatGPT Project chatで`Pro`、app identity、exact Tool sequenceを確認する。
  送信はmetadata-onlyで、Local Artifactのattachment、upload、pasteは0回とする。
- Browser UIが`Pro`とapp Tool callを同時に提供しない場合は、未確認の成功を主張せず
  `blocked`として保持する。

## 7. 実施記録（2026-07-31）

| Evidence | Classification | Result |
| --- | --- | --- |
| static contract validator | `measured` | `TOTAL_FAILURES=0`。Browser版ChatGPTがMCP clientであり、Codexはcontrol plane、turnごとに`Pro`を確認する現行contractを検査した。 |
| Secure MCP Tunnel lifecycle suite | `measured` | 35件すべて成功。固定profile、lock、catalog、allowed rootのpreflightを確認した。 |
| local server self-test／test suite | `measured` | exact 4 read-only Toolとforbidden Toolなしを確認。cold-start indexの実測に合わせ、local serverのindex build上限を5秒から10秒へ拡張後、128件すべて成功した。これはlocal implementation decisionであり、OpenAIの公式推奨ではない。 |
| Browser版ChatGPT acceptance | `measured` | ChatGPT Desktop app内蔵BrowserでProject title、memory `プロジェクトのみ`、`応答性能: Pro`、collapsed `Pro`、read-only `G Workspace Readonly`選択を可視確認した。Local Artifactのattachment、upload、paste、Project Sourceは0回。 |
| Browser版ChatGPT Tool evidence | `measured` | primary turnで`G Workspace Readonly`のTool catalogが公開されず、最初の必須`list_allowed_directories`は`REQUIRED_TOOL_CATALOG_UNAVAILABLE`。後続の`search`、`list_directory`、`fetch`は未実行で、Tool cardは0件。 |

以上より、local Tunnelのready状態、Browser app選択、`Pro`選択はそれぞれ確認できたが、同一のbrowser ChatGPT `Pro` turnでexact Tool catalogとcall successを確認できなかった。現在のroute dispositionは`blocked`である。これは`非常に高い`、attachment、upload、paste、API、別appへのfallbackを許可しない。再開には、browser ChatGPT `Pro` turnへ`G Workspace Readonly`の4 Toolが実際に公開されることを確認できるruntime／product側の状態変更が必要である。
