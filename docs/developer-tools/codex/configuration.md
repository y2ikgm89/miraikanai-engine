# Codex Configuration Guide for Miraikanai Development

- 文書種別: Engine外Developer Tool Guide
- 状態: review
- 対象Codex CLI: 0.144.5
- 公式資料検証日: 2026-07-21
- 適用範囲: 個人Codex設定、用途別profile、Miraikanai repository設定

## 1. 結論

Codexの公式starting pointはDefault Powerであり、GPT-5.6 SolとMedium reasoningを使う。用途に応じてSmarterまたはFasterへ寄せ、必要な品質を満たす最も低いreasoning effortを選ぶ。これはOpenAIの一般推奨である。

Miraikanaiでは次のproject判断を採用する。通常の個人作業は公式starting point相当、Engine repositoryはSol／High、最難関のread-only調査だけをDeep profileへ分離する。Fast mode、Max、XHighを全作業の既定にしない。設定による品質保証を主張せず、代表taskで速度、使用量、修正品質を再測定する。

本書はEngine Architectureではなく個人用Developer Tool Guideである。[Engine Architecture Index](../../architecture/README.md)の42仕様には含めない。ここにあるモデル名、CLI version、推奨は時点依存であり、Engine contractまたはProduct定数ではない。

## 2. 公式baselineとproject判断

| 主題 | 公式baseline | Miraikanai判断 |
|---|---|---|
| Model | Default PowerはGPT-5.6 Sol | user defaultは`gpt-5.6`、repositoryは`gpt-5.6-sol`へ明示固定 |
| Reasoning | Mediumから開始し、必要な時だけ上げる | user Medium、repository High、Deep Max |
| Config scope | user、trusted project、profile、one-off overrideを分離 | 共通値をuser、Engine固有値をproject、特殊用途をprofileへ置く |
| Approval／sandbox | default permissionから開始し、必要時だけ緩める | interactive作業は`on-request`／`workspace-write` |
| Windows | native sandboxは`elevated`を推奨 | 管理者権限またはsetup不可の場合だけ`unelevated` |
| Web search | local chatの既定はcached | 通常cached、明示的な最新性要求だけlive |
| Agents | thread既定6、nesting depth既定1 | 既定値を維持し、taskに必要な時だけ並列化 |

Solは複雑で曖昧な高価値作業、Terraは日常的な作業とread-heavy scan、Lunaは明確で反復可能な大量処理へ使う。model tierとreasoning effortは別の軸であり、高effortの小型modelが常に低effortの上位modelを上回るとは扱わない。

## 3. 設定layerと優先順位

Codexは同じ設定layerをCLI、IDE extension、ChatGPT desktop appで共有する。優先順位は高い順に次である。

1. CLI flagと`--config`
2. trusted project内の`.codex/config.toml`。rootからcurrent directoryへ近いlayerが勝つ
3. `--profile name`で選択した`$CODEX_HOME/name.config.toml`
4. userの`$CODEX_HOME/config.toml`
5. system config
6. built-in default

Projectがuntrustedならproject `.codex/` layerは読み込まれない。profileはprojectより下位なので、projectと同じkeyを一時的に変える場合はCLI flagまたは`--config`を使う。

Profileは独立した`$CODEX_HOME/<name>.config.toml`へtop-level keyで記述する。`config.toml`内の`[profiles.<name>]` tableまたはtop-level `profile` selectorは使わない。Codex 0.134.0以降はこの旧形式を読み込まないため、互換aliasまたはshimを残さず独立fileへ移す。

## 4. User default

対象は`$CODEX_HOME/config.toml`、通常は`~/.codex/config.toml`である。既存のMCP、plugin、app、hook、notification、shell environment、credential設定を保持し、次のkeyだけをmergeする。

```toml
#:schema https://developers.openai.com/codex/config-schema.json

model = "gpt-5.6"
model_reasoning_effort = "medium"
plan_mode_reasoning_effort = "high"
model_verbosity = "low"
model_reasoning_summary = "none"
personality = "pragmatic"

approval_policy = "on-request"
approvals_reviewer = "user"
sandbox_mode = "workspace-write"
web_search = "cached"

[windows]
sandbox = "elevated"

[agents]
max_threads = 6
max_depth = 1
```

`gpt-5.6`は公式config exampleの一般model指定、Mediumは公式のbalanced defaultである。Miraikanai固有の高推論設定はuser全体へ広げない。管理対象deviceでは`requirements.toml`がuser／project設定を制約できるため、実効値は設定fileだけでなくprovenance付きconfig readで確認する。

## 5. Repository config

対象はrepository rootの`.codex/config.toml`である。trusted projectでだけ読み込まれる。

```toml
#:schema https://developers.openai.com/codex/config-schema.json

model = "gpt-5.6-sol"
model_reasoning_effort = "high"
plan_mode_reasoning_effort = "xhigh"
model_verbosity = "low"
model_reasoning_summary = "concise"
```

これはOpenAIの一般既定ではなく、複数Subsystem、安全境界、長期検証を扱うMiraikanai向けのproject判断である。permission、web search、MCP、agent並列数はrepositoryが強制せず、user／managed policyへ残す。

## 6. 用途別profile

Profileはbase user configへ差分をoverlayする。project内ではproject configが同名keyを上書きするため、profile単独のload試験は競合する`.codex/config.toml`がない一時directoryで行う。

### 6.1 Fast

`$CODEX_HOME/fast.config.toml`:

```toml
model = "gpt-5.6-terra"
model_reasoning_effort = "medium"
plan_mode_reasoning_effort = "medium"
model_verbosity = "low"
model_reasoning_summary = "none"
personality = "pragmatic"
service_tier = "fast"

[features]
fast_mode = true
```

軽量な探索、明確な修正、反復確認に使う。Fast modeはStandardより高いcredit rateを使うため明示選択に限定する。`service_tier`だけを設定してFast modeを有効化したものと扱わない。

### 6.2 Deep

`$CODEX_HOME/deep.config.toml`:

```toml
model = "gpt-5.6-sol"
model_reasoning_effort = "max"
plan_mode_reasoning_effort = "xhigh"
model_verbosity = "medium"
model_reasoning_summary = "concise"
personality = "none"
sandbox_mode = "read-only"
```

最難関の原因分析、設計検証、深いreviewに使う。調査と変更適用を分離し、Maxを通常実装へ流用しない。

### 6.3 Routine

`$CODEX_HOME/routine.config.toml`:

```toml
model = "gpt-5.6-luna"
model_reasoning_effort = "medium"
plan_mode_reasoning_effort = "medium"
model_verbosity = "low"
model_reasoning_summary = "none"
personality = "pragmatic"
```

完了条件が明確な抽出、分類、変換、構造化要約に使う。品質不足の実測なしにreasoningを上げない。

### 6.4 Scan

`$CODEX_HOME/scan.config.toml`:

```toml
model = "gpt-5.6-terra"
model_reasoning_effort = "medium"
plan_mode_reasoning_effort = "medium"
model_verbosity = "low"
model_reasoning_summary = "none"
personality = "none"
sandbox_mode = "read-only"
```

大量file探索、read-only調査、補助情報の要約に使う。

## 7. 未指定にするkeyと削除対象

次は実測でoverrideが必要になるまで追加しない。

- `model_context_window`
- `model_auto_compact_token_limit`
- `tool_output_token_limit`
- `model_supports_reasoning_summaries`
- user defaultの`service_tier`
- user defaultの`review_model`

Codexのmodel catalogがcontext、compaction、tool truncation、capabilityを管理する。手動値はmodel更新後も残るため、利用可能contextを狭めたり早過ぎるcompactionを招いたりする。

削除済みfeature key、deprecatedなweb-search feature toggle、旧profile table、top-level profile selectorを保持しない。現在のtop-level `web_search`と独立profile fileだけを使い、後方互換aliasを設けない。

## 8. 適用手順

1. user configと既存profileのpath、存在有無、SHA-256を取得する。
2. 全対象fileを一つの日時付きbackup directoryへcopyし、元の存在有無とhashをmanifestへ記録する。
3. `git rev-parse --show-toplevel`でrepository rootを確定し、project trustを確認する。
4. user configでは対象keyだけをmergeし、MCP、plugin、app、hook、notification、shell、credential tableを保持する。
5. legacy profile syntaxとdeprecated／removed keyを削除し、4個の独立profile fileを作成する。
6. repository `.codex/config.toml`を作成する。
7. TOML parse、公式JSON Schema、Codex runtimeで検証する。
8. provenance付きconfig readでuser／profile／project／CLIの実効layerを確認する。
9. 新しいCodex taskで通常設定を確認し、代表taskの品質、所要時間、使用量を比較する。

設定を書き換える操作は原子的置換にし、backupが完成する前に元fileを変更しない。credential値またはsecret-bearing MCP環境変数をreportへ出力しない。

## 9. 検証

最低限、次を実行する。

```powershell
codex --version
codex doctor --json
codex features list
codex debug models

$validationRoot = Join-Path ([System.IO.Path]::GetTempPath()) 'codex-profile-validation'
New-Item -ItemType Directory -Force -Path $validationRoot | Out-Null
Push-Location $validationRoot
try {
  codex --profile fast debug prompt-input "profile validation"
  codex --profile deep debug prompt-input "profile validation"
  codex --profile routine debug prompt-input "profile validation"
  codex --profile scan debug prompt-input "profile validation"
} finally {
  Pop-Location
}
```

加えて次を確認する。

- 全TOMLがparseでき、公式`config-schema.json`へ適合する。
- `doctor`のconfig loadと各profileのloadが成功する。
- provenance付きconfig readでrepositoryの5 keyがproject layer由来になる。
- Fast profileで`features.fast_mode`と`service_tier`が同時に有効である。
- legacy profile selector、deprecated／removed key、unknown keyが0件である。
- 対象外tableが変更前hashまたはsemantic diffで不変である。
- Desktop app／CLIでmodel pickerまたはCLI overrideを使った時は、その高優先layerが実効値になる。

CLI version、model catalog、JSON Schema、公式推奨は更新される。upgrade時は同じ検査を再実行し、固定したcontext値や互換aliasを追加して追従しない。

## 10. Rollback

backup manifestに記録したfileだけを元pathへ戻し、復元後SHA-256を照合する。変更前に存在しなかったprofileまたはproject configは、正規化した絶対pathを確認してから削除する。rollback後に`codex doctor --json`とprofile load検査を再実行する。

部分rollbackでbase user configとprofileの世代を混在させない。管理policyまたはproject trustが原因の場合は、それを無効化して迂回せず、実効layerと拒否理由を記録する。

## 11. 公式根拠

2026-07-21にOpenAI Codex Manualを取得し、local cacheがcurrentであることを確認した。さらにCLI `codex-cli 0.144.5`と公式JSON Schema `ConfigToml`で本文のkeyを照合した。

- [Codex Models](https://learn.chatgpt.com/docs/models)
- [Config basics](https://learn.chatgpt.com/docs/config-file/config-basic)
- [Advanced configuration and profiles](https://learn.chatgpt.com/docs/config-file/config-advanced#profiles)
- [Configuration reference](https://learn.chatgpt.com/docs/config-file/config-reference)
- [Sample configuration](https://learn.chatgpt.com/docs/config-file/config-sample)
- [Sandbox and approvals](https://learn.chatgpt.com/docs/agent-approvals-security#sandbox-and-approvals)
- [Windows sandbox](https://learn.chatgpt.com/docs/windows/windows-sandbox)
- [Speed and Fast mode](https://learn.chatgpt.com/docs/agent-configuration/speed)
- [Subagents](https://learn.chatgpt.com/docs/agent-configuration/subagents)
- [Official configuration JSON Schema](https://developers.openai.com/codex/config-schema.json)

公式Manualから確認した事実とMiraikanai固有の採用値を2節で分離した。公開資料にないmodel間の普遍的順位、effort別固定cost、将来のavailabilityは主張しない。
