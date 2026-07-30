# ChatGPT Pro Secure MCP Tunnel Auto-start Design

- 設計状態: approved
- 対象: 個人Global Skill `collaborating-with-chatgpt-pro`
- 判断日: 2026-07-31
- 外部根拠確認日: 2026-07-31
- 先行設計:
  [ChatGPT Pro Secure MCP Tunnel Skill Update Design](2026-07-30-chatgpt-pro-secure-mcp-tunnel-skill-design.md)

## 1. 結論

`collaborating-with-chatgpt-pro`の開始時に、このWindows環境で承認済みの
`g-workspace-readonly` Secure MCP Tunnelを確認する。既にreadyなら再利用し、
停止中なら既存の公式`tunnel-client`と同Profileを非表示Processとして起動する。

起動処理はSkill内のPowerShell helperへ集約する。Agentが毎回Process名、
Profile、状態確認、待機、失敗処理を組み立てない。Windows Serviceまたは
Scheduled Taskは作成せず、OS login時の常駐化へScopeを広げない。

Local起動成功はTunnel read権限またはEvidence delivery成功を意味しない。
既存Task Contract、ChatGPT側app availability、allowed root、target
reachability、Tool catalogのpreflightはすべて維持する。

## 2. 根拠と採用判断

### 2.1 OpenAI公式として採用する事実

[Secure MCP Tunnel](https://developers.openai.com/api/docs/guides/secure-mcp-tunnels)
は次を説明している。

- App discoveryとMCP Tool callは稼働中の`tunnel-client`に依存する。
- Named Profileは`tunnel-client run --profile <name>`で起動できる。
- `/healthz`、`/readyz`、`/metrics`、loopback-onlyの`/ui`が提供される。
- Clientが接続されていない間、Tunnel requestは再接続まで失敗する。
- DiscoveryまたはTool call failure時はProcessを確認し、
  `tunnel-client doctor --profile <name> --explain`を再実行する。
- 公式公開Release `v0.0.10`では、control-plane poll成功後だけreadyになる。

### 2.2 Localで確認済みの実行条件

2026-07-31時点で次を秘密値を表示せず確認した。

- executable:
  `%LOCALAPPDATA%\OpenAI\secure-mcp-tunnel\bin\tunnel-client.exe`
- version: `0.0.10`
  （[公式最新Release](https://github.com/openai/tunnel-client/releases/latest)と一致）
- profile: `%APPDATA%\tunnel-client\g-workspace-readonly.yaml`
- health listen address: `127.0.0.1:8080`
- ProfileのMCP許可対象: `G:\workspace`

Runtime API key、Tunnel ID、Organization ID、Workspace IDは設計書、helper
output、Skill Transcriptへ保存しない。

### 2.3 本Skillの採用判断

次はOpenAIの固定仕様ではなく、この個人環境向けの採用判断である。

- Profileを`g-workspace-readonly`へ固定する。
- Skill開始時にlocal lifecycle helperを必ず実行する。
- 起動待機上限を30秒にする。
- `Start-Process -WindowStyle Hidden`で可視Windowを作らない。
- 既存の不健康Process、別Processによるport使用、未知Profileを自動修復しない。
- Helper自身が新規起動したProcessだけ、ready timeout時に停止する。

## 3. 方式比較

| 方式 | 利点 | 欠点 | 判定 |
| --- | --- | --- | --- |
| Skill内の決定的helper | 状態判定、多重起動防止、失敗分類を一貫化できる | Windows個人環境に依存する | 採用 |
| Agentが毎回PowerShellを構成 | 追加Fileが不要 | 判断とquotingがrunごとに変動する | 不採用 |
| Windows Service／Scheduled Task | Login後の常時接続が容易 | OS永続設定、更新、停止責任までScopeが広がる | 不採用 |

## 4. Component

### 4.1 Lifecycle helper

次を追加する。

```text
C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\
  scripts\ensure_secure_mcp_tunnel.ps1
```

既定値は環境変数から組み立て、usernameを固定しない。

```yaml
runtime:
  executable: "%LOCALAPPDATA%\\OpenAI\\secure-mcp-tunnel\\bin\\tunnel-client.exe"
  profile: g-workspace-readonly
  profile_file: "%APPDATA%\\tunnel-client\\g-workspace-readonly.yaml"
  health_base_url: "http://127.0.0.1:8080"
  startup_timeout_seconds: 30
```

HelperはProfile Fileの秘密値を読出結果へ出力せず、起動command lineへ秘密値を
展開しない。既存Profile参照だけを`tunnel-client`へ渡す。

### 4.2 Skill contract

`SKILL.md`と`prompt-generation-contract.md`へlocal lifecycle preflightを追加する。
Browser navigationまたはBrowser sendより前にhelperを実行し、成功した場合だけ
既存Secure MCP Tunnel preflightへ進む。

`artifact-delivery-contract.md`の配送順、fallback権限、Tunnel completenessは
変更しない。Lifecycle failureはTunnel completeness成功へ読み替えない。

### 4.3 Static validator

`scripts/validate_secure_mcp_contract.ps1`へ次のpredicateを追加する。

- startup checkがBrowser actionより前に必須である。
- stopped時だけ固定Profileを起動する。
- already-ready時は再利用する。
- unknown／unhealthy Processをkillまたはduplicate-startしない。
- local ready後もapp、scope、reachability、Tool検証を維持する。
- failure時はauthorized fallbackまたは`blocked`である。

## 5. State machine

```text
Skill invoked
  -> resolve exact executable and exact profile
  -> inspect expected tunnel-client process
  -> probe /healthz and /readyz

expected process + health + ready
  -> already-running
  -> continue Browser/Task preflight

no expected process + endpoints unavailable
  -> start exact profile in hidden window
  -> poll health and ready for <= 30 seconds
  -> ready: started
  -> timeout: stop only the process created by this helper
  -> report startup-failed

expected process + not ready
  -> existing-unhealthy
  -> do not start duplicate or kill it

endpoint responds + no expected process
  -> port-conflict-or-foreign-owner
  -> do not start or kill anything
```

`already-running`と`started`だけをlocal lifecycle成功とする。その後に
ChatGPT側のapp identity、allowed root、target reachability、required Toolsを
別に検証する。

## 6. Process起動と出力

Helperは`Start-Process`へargument arrayを渡し、Shell文字列を組み立てない。
起動する論理commandは次だけである。

```text
tunnel-client.exe run --profile g-workspace-readonly
```

Windowはhiddenとする。PID、起動時刻、local health stateはtransient Evidence
として扱う。Process outputが必要な場合はsystem temporary directoryの
task-specific logへ保存し、raw HTTP loggingを有効にしない。

Helperはmachine-readable resultを返す。

```yaml
result:
  status: already-running | started | blocked
  reason: ready | missing-runtime | existing-unhealthy |
          port-conflict | start-failed | ready-timeout
  process_id: observed | not-applicable
  health: true | false
  ready: true | false
  elapsed_ms: integer
```

API key、Tunnel ID、Profile内容、HTTP payload、secret-bearing environment
variableはresultへ含めない。

## 7. Failure処理

| 状態 | 処理 |
| --- | --- |
| executable／Profile欠落 | 起動せず`blocked` |
| 既存Processがready | 再利用 |
| 既存Processが不健康 | duplicate起動・killをせず`blocked` |
| port 8080を未知Processが使用 | kill・port変更をせず`blocked` |
| 新規Processが30秒以内にready | 既存Browser preflightへ進む |
| 新規Processがtimeout／異常終了 | helperが作成したPIDだけ停止して`blocked` |
| Tunnel local lifecycle成功後にapp／Tool確認失敗 | authorized chat fallbackまたは`blocked` |

`doctor --profile g-workspace-readonly --explain`は、起動失敗の診断に使えるが、
診断結果をBrowserへ送らない。永続記録前にsecret、ID、不要な絶対Pathを除く。

## 8. Verification

### 8.1 RED

現行validatorへauto-start contractの必須predicateを追加し、現行Skillで失敗する
ことを確認してからSkillまたはhelperを変更する。

### 8.2 Helper test

実環境で次を順に確認する。

1. Tunnel停止状態でhelperを実行し、`started`になる。
2. `/healthz`と`/readyz`が成功する。
3. 再度helperを実行し、同じProcessを`already-running`として再利用する。
4. Process数が増えない。
5. Helper outputとlogへsecret値が出ていない。

既存の不健康Processまたはport conflictを実環境で故意に作らない。Process／HTTP
判定を注入可能な小さいhelper functionへ分離し、temporary fixtureでfailure
branchを検証する。

### 8.3 Skill validation

- `validate_secure_mcp_contract.ps1`
- `quick_validate.py`
- Relative reference確認
- YAML metadata確認
- Skill差分の完全確認
- `git diff --check`
- `git status --short`
- `git diff --stat`

実際のBrowser ChatGPT consultationはauto-start helperの検証には不要とする。
Local lifecycle成功とChatGPT側Tunnel completenessを混同しない。

## 9. 更新範囲

Personal Skill:

- `SKILL.md`
- `references/prompt-generation-contract.md`
- `scripts/ensure_secure_mcp_tunnel.ps1`（新規）
- `scripts/validate_secure_mcp_contract.ps1`

変更しないもの:

- `agents/openai.yaml`
- Repository `AGENTS.md`
- Tunnel Profile内容
- Tunnel ID、Organization／Workspace association
- Filesystem MCP実装
- Windows Service／Scheduled Task
- `tunnel-client`の自動download／自動update
- Browser delivery fallbackの既存順序

## 10. 受入条件

- Skill開始時に固定Profileのlocal lifecycleが確認される。
- Readyな既存Processを再利用し、多重起動しない。
- 停止中なら既存の公式clientとProfileをhidden Processで起動する。
- 30秒以内にreadyにならない場合は、helperが作成したProcessだけを停止する。
- 既存の不健康Processまたは未知port ownerを自動killしない。
- SecretまたはTunnel identityをcommand、result、Transcriptへ出さない。
- Local ready後もTask authorizationとChatGPT側Tunnel preflightを省略しない。
- Lifecycle failureをTunnel delivery成功と呼ばない。
- Windows常駐設定またはTunnel Profileを変更しない。
