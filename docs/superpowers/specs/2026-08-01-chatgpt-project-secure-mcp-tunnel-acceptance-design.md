# ChatGPT Project Secure MCP Tunnel Acceptance Design

- 状態: `approved-for-implementation`
- 対象: Browser版ChatGPTの新規Projectにおける、既存Secure MCP
  Tunnel-backed `G Workspace Readonly` appのTool discovery／Tool call検証
- 判断日: 2026-08-01
- 外部根拠確認日: 2026-08-01
- 外部根拠:
  - [Secure MCP Tunnel](https://developers.openai.com/api/docs/guides/secure-mcp-tunnels)
  - [Connect and test your plugin](https://developers.openai.com/plugins/deploy/connect-chatgpt)
  - [Apps in ChatGPT](https://help.openai.com/en/articles/11487775-connectors-in)
  - [Projects in ChatGPT](https://help.openai.com/en/articles/10169521-projects-in-chatgpt)
- 関連記録:
  - [ChatGPT Web Pro MCP Route Clarification Design](2026-07-31-chatgpt-web-pro-mcp-route-clarification-design.md)
  - [ChatGPT Pro Standalone MCP Route Design](2026-07-31-chatgpt-pro-standalone-mcp-route-design.md)

## 1. 目的と成功条件

Browser版ChatGPTに新規Project `Secure MCP Tunnel 検証`を作成し、そのProject内の
新規chatで既存の`G Workspace Readonly` appを選択する。Private Local MCP Serverが
公開するread-only ToolをChatGPTが発見し、`G:\workspace`配下の既知Artifactを実際に
読み取れることまでを確認する。

成功には次のすべてを必要とする。

1. `tunnel-client`がhealthy、ready、connectedである。
2. Local MCP Serverがexact Tool catalog
   `list_allowed_directories`、`list_directory`、`search`、`fetch`を公開し、write Toolを
   公開しない。
3. ChatGPT Developer modeの既存`G Workspace Readonly` connectionが、上記Toolと
   metadataをRefresh後に発見する。
4. 新規Project内の新規chatでexact appを追加できる。
5. ChatGPTが少なくとも`list_allowed_directories`と、既知Artifactの取得に必要なToolを
   実行する。
6. ChatGPT応答のArtifact識別子、byte count、SHA-256、非機密markerがLocalで取得した
   manifestと一致する。
7. Browser attachment、upload、Local Artifact本文のprompt paste、Project Source追加、
   write Tool、alternate appを使用しない。

## 2. 根拠分類

| Claim | Classification | Evidence／owner |
| --- | --- | --- |
| Secure MCP Tunnelはprivate MCP Serverをpublic internetへ公開せず、対応OpenAI productから利用可能にする。 | `official-spec` | OpenAI Secure MCP Tunnel guide |
| ChatGPTではDeveloper mode appを作成し、Tunnelを選択してTool／metadataを発見し、新規conversationのTools menuから接続する。 | `official-spec` | OpenAI Connect and test guide |
| Tunnel associationはChatGPT Workspace／Platform Organizationを対象とし、ChatGPT Project単位ではない。 | `official-spec` | OpenAI Secure MCP Tunnel guide |
| Connected appはProject chatで利用できるが、App互換性はmodelを含む利用条件によって異なる。 | `official-spec` | OpenAI Projects／Apps Help Center |
| 新規Project名を`Secure MCP Tunnel 検証`とし、memoryを`プロジェクトのみ`にする。 | `project-decision` | 今回の検証を他Project contextから分離するため |
| 既存`G Workspace Readonly`を再利用し、重複Appを作成しない。 | `project-decision` | 最小変更と既存設定保護 |
| 過去のProject chatではTool catalogがturnへ公開されなかった。 | `measured`／historical | 2026-07-31 route clarification record |
| 新規Projectの`Pro` chatではexact app選択後もsession Tool listが空で、Tool callが0件だった。 | `measured` | 2026-08-01 Task 5 initial attempt |
| 非`Pro` retryでは各取得を示すaggregate live Tool statusが表示され、個別Tool cardは完了後に保持されなかった。 | `measured` | 2026-08-01 Task 5 corrected retry |

公式Help CenterはApp互換性がmodelによって異なると明記している。`Pro` chatでToolが
公開されなかった実測を受け、同じProject、app、prompt、Artifactを保ったまま、
response performanceだけをApp互換の非`Pro` modelへ変更して一度再試行する。これは
今回のacceptanceに限る`project-decision`であり、既存の
`collaborating-with-chatgpt-pro` Skill contractや別Projectの設定を変更しない。

## 3. 採用方式

既存AppをRefreshして新規Projectで検証する。

1. 既存lifecycle helper、`tunnel-client doctor`、Local MCP self-test／testsでLocal側を
   先に検証する。
2. MCP Inspectorでinitialize、Tool list、代表的なread call、invalid input、write Tool不在を
   確認する。Inspectorを実行できない場合はBrowser操作へ進まず`blocked`とする。
3. ChatGPTのDeveloper modeとexact app identityを可視確認し、既存connectionを
   `Refresh`してTool metadataを再取得する。
4. Browser版ChatGPTに新規Projectを作成し、Project内の新規chatでexact appを追加する。
5. `Pro`でTool catalogがsessionへ公開されないことを確認した場合は、新規Project chatの
   response performanceだけをApp互換の非`Pro` modelへ変更し、metadata-only
   acceptance promptを一度再送してTool実行結果とsanitized Tunnel telemetry deltaを
   Local manifestに照合する。

既存connectionの削除、同じTunnelを使う重複App作成、Tunnel Profile変更、Local MCP
Server実装変更、公開HTTPS化、Plugin submissionは今回の範囲外とする。

## 4. データフローと権限境界

```text
Browser ChatGPT Project chat
  -> G Workspace Readonly developer-mode app
  -> OpenAI-hosted Secure MCP Tunnel endpoint
  -> outbound-only tunnel-client
  -> Local read-only MCP Server
  -> allowlisted G:\workspace Artifact
```

- ChatGPT Projectは会話contextの境界であり、MCP認可境界には使用しない。
- `G:\workspace` root、path containment、read-only Tool catalogはLocal MCP Server側で
  強制する。
- Browserへ渡すのはroot-relative Artifact ID、期待byte count、SHA-256、期待markerだけ
  とし、Artifact本文はTool resultとしてのみ返す。
- Durable evidenceへ`tunnel_id`、request ID、Profile本文、API key、cookie、account ID、
  Local secretを保存しない。

## 5. エラー処理

- TunnelがChatGPTに表示されない場合は、対象ChatGPT Workspace association、
  `Tunnels Read + Use`、Developer mode permissionを確認し、別Workspaceへ切り替えない。
- Local Tool catalogが不正な場合はBrowser操作へ進まず、Local側を`blocked`とする。
- App connectionのRefresh後もTool metadataが欠ける場合は、connectionを削除せず、
  discovery failureとして`blocked`にする。
- Project chatでexact appを選択できない、Tool callが発生しない、または結果照合が失敗
  した場合は、standalone chat、別App、upload、paste、公開endpointへfallbackしない。
- `Pro` chatでTool callが発生せずsession Tool listにも公開されない場合に限り、同じ
  Project内の新規chatでApp互換の非`Pro` modelへ変える一変数の再試行を許可する。
- Browserがsign-in、管理者承認、OAuthなど明示的なuser actionを要求した場合は、その場で
  停止して必要な操作だけを依頼する。

## 6. 検証と証拠

実施順序は公式のlocal inspectionからChatGPT connection、conversation testへの流れに
合わせる。

1. Skill static validator、Tunnel lifecycle suite、Local MCP tests／self-testを実行する。
2. Tool discovery画面でexact Tool catalogとmetadataを確認する。
3. 新規Projectのtitle、memory、exact app selectionを可視確認する。
4. 個別Tool cardまたは各取得を特定できるaggregate live Tool statusを確認し、response prose
   だけからTool callを推測しない。
5. 既知ArtifactのLocal manifestとBrowser応答を照合する。
6. sanitized telemetryでacceptance turn中のTool call／forward event増分を確認する。
7. `git diff --check`、`git status --short`、`git diff --stat`を実行し、repository変更が
   この設計／計画／必要なreview summaryだけに限定されることを確認する。

最終結果は`pass`または`blocked`で記録する。Tool callを確認できない場合、App選択や
Tunnel readyだけから成功を推論しない。

## 7. 変更範囲

この設計は一回の新規Project acceptanceを定義する。既存のArchitecture Owner、Local
MCP実装、Tunnel Profile、既存ChatGPT Project、既存chat、現行Personal Skill contract、
repository `AGENTS.md`を変更しない。既存standalone-chat設計／計画は履歴として保持し、
今回のacceptanceへ暗黙に適用しない。

## 8. 実測acceptance記録（non-normative）

実施日: 2026-08-01。これは今回の受入手順の実測証拠であり、Architectureの実装、
qualification、activation、releaseまたはProduct completionを主張しない。

- Local preflightはstatic validatorの`TOTAL_FAILURES=0`、Tunnel lifecycle 35件`PASS`
  （独立`pwsh -File` process exit `0`）、Local MCP pytest 128件passed、self-test
  `PASS`、ensure contractの6 predicate全trueを確認した。
- MCP Inspector 2.0.0はexact 4 read-only Toolを発見し、4件のvalid readはすべて
  `PASS`、path escapeは`path-outside-root`、unknown writeは`tool_not_found`として
  fail-closedを確認した。
- Browser ChatGPTではDeveloper mode enabledの既存`G Workspace Readonly`を一度だけ
  Refreshし、root `G:\workspace`のread-only description、exact 4 Tool、write Tool不在を
  可視確認した。新規Project `Secure MCP Tunnel 検証`は
  `プロジェクト限定メモリ`（Project-only）で作成し、sources／uploadsは0件だった。
- 初回`Pro` chatはsession Tool list 0件で診断上`blocked`となった。同じProject、App、
  prompt、Artifactを維持してresponse performanceだけを非`Pro`の`非常に高い`へ変更した
  一回のretryでは、`list_allowed_directories`、`list_directory`、`fetch`の各取得をUIで
  特定し、unexpected Toolは観測されなかった。
- retryのroot、Artifact ID、`metadata.source_bytes`、`metadata.source_sha256`、先頭見出し、
  completion markerはcanonical manifestと全一致した。aggregate `tools/call`は利用不能
  だったが、sanitized `forward_event_delta=4`は正であり、UI Tool evidenceと併せて照合した。
- acceptance attemptは合計2件で第三試行はない。Tunnel／Appは再作成しておらず、attachment、
  upload、Project Source、Web、alternate Appも使用していない。最終結果は`pass`、terminal
  markerは`CHATGPT_PROJECT_MCP_ACCEPTANCE_20260801`である。
