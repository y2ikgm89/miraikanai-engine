# ChatGPT Pro Secure MCP Tunnel Skill Update Design

- 設計状態: approved
- 対象: 個人Global Skill `collaborating-with-chatgpt-pro` と本Repositoryの
  ChatGPT Pro route
- 判断日: 2026-07-30
- 外部根拠確認日: 2026-07-30
- 先行設計:
  [Collaborating with ChatGPT Pro Skill Design](2026-07-29-collaborating-with-chatgpt-pro-skill-design.md)

## 1. 結論

ChatGPT ProへRepository Evidenceを渡す既定経路を、利用可能性とScopeを
検証したSecure MCP Tunnel-backed appにする。Tunnelはlocal Repositoryを
公開せずにChatGPTからprivate MCP serverへ接続する公式経路として扱う。

Tunnel-backed appがTaskに必要なArtifactを完全に提供できない場合だけ、
既存のchat attachment、inline excerpt、review bundle、split partsへ
fallbackする。Tunnelの未接続、Scope不一致、Tool不足を黙って部分Reviewへ
読み替えない。

ChatGPTのTool実行能力は権限を拡大しない。Task Contractで
`local_writes: true`の場合だけ、Scope内の書き込みを依頼できる。変更後は
Codexがlocal source、差分、必要な検証を確認し、ChatGPTの成功表示だけで
完了としない。

## 2. 公式根拠とProject判断

### 2.1 OpenAI公式として採用する事実

[Secure MCP Tunnel](https://developers.openai.com/api/docs/guides/secure-mcp-tunnels)
は次を明示している。

- private、on-premises、developer machine上のMCP serverをChatGPTなどの
  対応OpenAI製品へ接続できる。
- `tunnel-client`はprivate MCP serverへ到達できるtrust boundary内で動作し、
  OpenAIへoutbound HTTPS接続する。
- ChatGPTではdeveloper-mode app作成時にTunnel connectionを選択する。
- App discoveryとTool callには稼働中の`tunnel-client`が必要である。
- Platform tunnel permissionとChatGPT developer-mode accessは別である。
- Secure MCP Tunnelはprivate connectionとdeveloper-mode testing向けであり、
  public plugin submissionの代替ではない。

### 2.2 Project判断

公式文書は、本Skill固有のfallback順序、Task Contract、local writeの許可条件、
Transcript形式、Codexによる再検証を規定しない。これらは利用者Intent、
Repository authority、既存Skill contractに基づくProject判断とする。

Tunnel利用料金、ChatGPT Pro entitlement、write確認UIの挙動は本Skillへ固定
しない。費用または確認挙動がTaskにmaterialな場合、current official source、
visible UI、Platform usageを実行時に確認する。

## 3. 更新範囲

### 3.1 個人Global Skill

次を最小更新する。

- `SKILL.md`
- `references/prompt-generation-contract.md`
- `references/artifact-delivery-contract.md`
- `references/transcript-contract.md`
- 必要な場合だけ`agents/openai.yaml`

`response-completion-gate.md`と`adjudication-and-stop-rules.md`は、Tunnel Tool
callで新しい欠落が見つからない限り変更しない。

### 3.2 Repository route

root `AGENTS.md`はProject固有のrouteとTunnel利用条件だけを保持する。
Global SkillのTask Contract、配送、Transcript、裁定手順を複製しない。

## 4. Access mode

Skillは次の順でEvidence accessを解決する。

1. **Tunnel-backed app**
   - in-app Browser、正しいChatGPT Project、Project-only memory、Proを確認する。
   - 対象chatで期待するappが利用可能か確認する。
   - 読み取り専用preflightとしてallowed root、対象Repository到達性、必要Toolを
     Tool結果から確認する。
   - App名または説明だけからread-only／write-capableを推測しない。
2. **Chat delivery fallback**
   - Tunnelが利用不能、Scope外、Tool不足、形式非対応の場合、既存の
     attachment contractから最小の完全な方式を選ぶ。
3. **Blocked**
   - どの方式でも必要coverage、native structure、受領完全性を確立できない
     場合は、部分結果を完全Reviewと呼ばず停止する。

## 5. Task Contract

`authorized_actions`へTunnel利用を明示する。

```yaml
authorized_actions:
  browser_conversation: true
  browser_file_uploads: scoped | specific | denied
  tunnel_reads: scoped | specific | denied
  tunnel_writes: scoped | specific | denied
  local_reads: true | false
  local_writes: true | false
  external_side_effects: true | false
```

`tunnel_writes`は`local_writes: true`かつ利用者が変更、build、fix、executeを
依頼した場合だけ`scoped`または`specific`になり得る。Review、answer、
research、diagnose、planだけの依頼では`denied`とする。

## 6. Preflightとdata flow

Tunnel方式では送信前またはReview開始前に次を確認する。

1. `tunnel-client`がhealthy、ready、connectedである。
2. ChatGPTの対象Project／chatでappが利用可能である。
3. Tool結果がTask対象を含むallowed rootを示す。
4. Task対象のrootまたは代表Artifactを読み取れる。
5. 必要なread Toolと、書き込みを依頼する場合は必要なwrite Toolがある。
6. Task ContractのScopeとTool引数が一致する。
7. Secret、credential、不要なprivate dataを読み取らせない。

ChatGPTはTunnel経由でTaskに必要なArtifactだけを取得する。Repository全体を
既定で列挙、要約、送信しない。Tool outputとChatGPTの解釈は外部Evidenceで
あり、Repository状態の正本はCodexが読むlocal fileとGit差分である。

## 7. 書き込み契約

ChatGPTへTunnel writeを依頼できる場合も、次を必須とする。

- 対象path、目的、許可された変更範囲をPromptへ明記する。
- Scope外の作成、削除、移動、権限変更、Git、credential、release操作を
  許可しない。
- 破壊的または回復困難な変更は、Task Contractとは別に必要な承認を得る。
- Tool結果から変更対象と成否をTranscriptへ記録する。
- Codexがlocal fileを再読し、`git diff`とTask固有検証を実施する。
- Scope外差分、予期しないbinary、secret、削除を検出した場合は完了扱い
  せず、追加変更を停止する。

ChatGPTがwrite Toolを持つこと、またはUIが確認を表示しなかったことを、
書き込み権限の根拠にしない。

## 8. Transcriptと完了

Tunnel-backed runでは既存のvisible Prompt／Responseに加えて次を記録する。

- delivery mode: `secure-mcp-tunnel`
- visible app identity
- preflightで確認したallowed rootと対象到達性
- ChatGPTが表示したTool call名、Task-relevant引数、結果、error
- read／write分類とTask Contract上の権限
- write後にCodexが確認したlocal diffとvalidation

Token、API key、Tunnel secret、cookie、account identifier、不要な絶対pathは
永続Transcriptへ保存しない。Tool callが欠落、失敗、途中停止した場合は
Response completion gate通過後にだけ修復し、確認不能なら`blocked`とする。

## 9. Failureとfallback

| 状態 | 処理 |
| --- | --- |
| Tunnel未接続 | healthを確認し、復旧不能ならchat deliveryを評価 |
| App未表示 | workspace associationとTunnel Use権限を確認 |
| Scope外 | Scopeを拡張せず、authorized attachmentまたはblocked |
| read Tool不足 | 必要Artifactだけchat delivery |
| write Tool不足 | ChatGPT writeを行わず、許可済みならCodexがlocal実装 |
| unexpected write | 追加Tool callを停止し、local diffを保全して報告 |
| 必要coverage不完全 | 完全Reviewとせずblocked |

Destination、Project、memory mode、Pro、Browserは既存routeどおりfail closedと
し、Tunnel fallbackを理由に変更しない。

## 10. 非目標

- Secure MCP Tunnelの料金が常に無料と保証すること
- App名に`Readonly`が含まれるだけでToolをread-onlyと判断すること
- TunnelにDocker、Kubernetes、VMの特定deployment方式を強制すること
- public plugin submissionへTunnelだけを使用すること
- ChatGPTへRepository authorityまたはGit authorityを移譲すること
- attachment fallbackを削除すること

## 11. 検証

実装前に、現行SkillではTunnelが利用可能でもattachment-onlyと判断する
baseline failureをfresh contextで再現する。更新後は同じscenarioで次を確認
する。

1. Tunnel-backed appを第一候補として選ぶ。
2. App名ではなくallowed root、到達性、Tool catalogを確認する。
3. review-onlyではTunnel writeを拒否する。
4. 明示的change taskではScope内writeだけを許可し、Codex再検証を要求する。
5. Tunnel failure時にattachment fallbackを選び、coverage不足時はblockedに
   する。
6. TranscriptへTool Evidenceとlocal verificationを含める。

Static validationとしてSkill validator、relative link、YAML metadata、
`git diff --check`、対象差分の完全確認を実施する。

## 12. 受入条件

- 個人Skillから「ChatGPT Projectはlocal Repositoryを直接読めない」という
  無条件の記述がなくなる。
- 公式Secure MCP Tunnelを検証済みprivate MCP accessの第一候補にする。
- Repositoryの`AGENTS.md`がTunnel利用を許可し、Global Skillと矛盾しない。
- Readとwriteの権限がTask Contractで分離される。
- 明示されたchange taskだけがScope内Tunnel writeを許可する。
- ChatGPTによる変更をCodexがlocal authorityで再検証する。
- Tunnelが利用不能でも安全なchat delivery fallbackを維持する。
- 料金、entitlement、confirmation UIを未確認の固定仕様として記載しない。
