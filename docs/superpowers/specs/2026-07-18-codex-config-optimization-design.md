# Codex 設定最適化 設計書

- 文書版: 1.1
- 作成日: 2026-07-18
- 対象 Codex CLI: 0.144.5
- 対象モデル: GPT-5.6 Sol / Terra / Luna
- 状態: 推奨構成の承認済み設計

## 1. 目的

Codex の実装品質を維持しながら、通常作業で不要な最大推論、過剰な出力、常時 Fast mode、過剰な並列実行を避け、速度と使用量を最適化する。

完了時には次を満たす。

1. 通常作業、複雑なゲームエンジン開発、最難関調査を別の設定層で扱える。
2. 通常作業では GPT-5.6 Sol の Medium、対象リポジトリでは High、明示的な Deep profile だけで Max を使う。
3. Plan mode は通常 High、対象リポジトリでは XHigh を使う。
4. 既定の回答量を Low にして、説明文による出力トークンと次ターンのコンテキスト増加を抑える。
5. reasoning summary は通常 None、対象リポジトリと Deep profile だけ Concise にする。
6. モデル固有のコンテキスト、圧縮、ツール出力上限は Codex のモデルカタログへ追従する。
7. 通常実行は公式の対話向け権限構成を使い、無条件のフルアクセスを既定値にしない。
8. 現在利用中の MCP、プラグイン、デスクトップ設定、通知設定は壊さない。

## 2. 調査結果

### 2.1 現行状態

`C:\Users\y2ikg\.codex\config.toml` は次の挙動を全作業へ適用している。

- `model = "gpt-5.6-sol"`
- `model_reasoning_effort = "max"`
- `plan_mode_reasoning_effort = "xhigh"`
- `model_verbosity = "low"`
- `model_reasoning_summary = "concise"`
- `approval_policy = "never"`
- `sandbox_mode = "danger-full-access"`
- `agents.max_threads = 8`
- `agents.max_depth = 1`
- 削除済み feature の `js_repl = false`

既存 profile は次の三つである。

- `fast.config.toml`: Sol / High / Fast service tier
- `deep.config.toml`: Sol / Max / Read-only
- `scan.config.toml`: Terra / Medium / Read-only

対象リポジトリには `.codex/config.toml` が存在しない。

### 2.2 実環境の検証結果

- `codex --version`: 0.144.5
- `codex doctor --json`: 設定ファイルの構文と読み込みは正常
- 認証方式: ChatGPT
- GPT-5.6 Sol の Codex 実行カタログ:
  - context window: 272,000
  - tool output truncation: 10,000 tokens
  - reasoning: Low / Medium / High / XHigh / Max / Ultra
  - default verbosity: Low
  - default reasoning summary: None
- 公開 JSON Schema の `ReasoningEffort` はモデルカタログが広告する非空文字列を受け入れる。したがって、現在の実行カタログが広告する `max` は Deep profile でスキーマ上も有効である。
- `plan_mode_reasoning_effort` には設計上 `xhigh` を上限として使用し、Ultra は設定ファイルへ固定しない。
- `js_repl` は Removed であり、設定値は無視される。
- 初期 prompt input は約 17.4k 文字で、うちスキルメタデータは約 7.4k 文字、グローバル `AGENTS.md` は約 3.0k 文字である。

### 2.3 公式方針

- Codex の標準 Power は GPT-5.6 Sol / Medium。
- Sol は曖昧・複雑・高価値な作業、Terra は日常作業の能力・速度・価格の均衡、Luna は明確で反復可能な大量処理を対象とする。
- 必要な品質を満たす最も低い reasoning effort を使う。
- High と XHigh は複雑な多段作業、Max は速度と使用量より深さを優先する最難関作業に限定する。
- Fast mode は約 1.5 倍速だが、ChatGPT 認証の GPT-5.6 では標準の 2.5 倍のクレジットを消費する。
- 個人既定値は `~/.codex/config.toml`、リポジトリ固有値は `.codex/config.toml`、用途別差分は `$CODEX_HOME/<name>.config.toml` に分離する。
- `model_context_window`、`model_auto_compact_token_limit`、`tool_output_token_limit` は、手動調整が必要な実測結果がない限り未指定にしてモデル既定値を使う。
- 対話型の標準権限は `approval_policy = "on-request"` と `sandbox_mode = "workspace-write"`。
- Windows native sandbox は `windows.sandbox = "elevated"` が推奨値。

### 2.4 モデルと reasoning effort の比較

モデルの capability tier と reasoning effort は別の軸である。モデルは判断力、曖昧さへの耐性、長期作業の追従性、仕上げ品質を決め、reasoning effort は同じモデルが計画、代替案、検査、修正へ使える推論量を増減させる。高い effort が下位 tier を上位 tier と同一能力にするという公式保証はない。

OpenAI が公開した GPT-5.6 の coding 評価は次のとおりである。これは公開評価構成のモデル間比較であり、`Sol / low` 対 `Terra / max` のような全組み合わせを同一条件で比較した表ではない。

| Coding 評価 | Sol | Terra | Luna |
| --- | ---: | ---: | ---: |
| Artificial Analysis Coding Agent Index v1.1 | 80.0 | 77.4 | 74.6 |
| SWE-Bench Pro | 64.6% | 63.4% | 62.7% |
| DeepSWE v1.1 | 72.7% | 69.6% | 67.2% |
| Terminal-Bench 2.1 | 88.8% | 87.4% | 84.7% |

System Card は reasoning effort を増やすと各モデルの性能が上がる曲線を示し、限定された評価では高 effort の小型モデルが低 effort の大型モデルを上回り得ることも示す。ただし、タスク横断の普遍的な順位は公開されていない。このため、交差比較は次のように扱う。

- `Sol / low` 対 `Terra / max`: 曖昧な設計、広いコード変更、セキュリティ、仕上げ品質では Sol を選ぶ。ただし難しい作業なら Sol を Medium または High に上げる。明確に定義された多段問題では Terra / Max が Sol / Low を上回る場合があるが、通常既定値としては非効率である。
- `Sol / low` 対 `Luna / max`: 複雑な開発は Sol / Low、明確で反復可能な変換・分類・抽出では Luna / Max が勝つ場合がある。通常は Luna / Medium または High で十分かを先に評価する。
- `Terra / max` 対 `Luna / max`: 品質優先は Terra、速度・価格・大量処理優先は Luna。ただし最難関の開発品質が必要なら、両者を Max にする前に Sol / High または XHigh を使う。

ChatGPT 認証のローカルメッセージ目安は、Plus の 5 時間枠で Sol 15–90、Terra 20–110、Luna 50–280 である。実消費は reasoning、コンテキスト、ツール、検索、キャッシュ、作業時間で変わるため、effort 別の固定換算率としては使わない。

## 3. 比較した構成

### 3.1 単一の最大品質設定

全作業で Sol / Max / XHigh Plan を使う現行方式。

- 利点: 選択操作が不要。
- 欠点: 軽い調査、明確な修正、反復作業にも最大推論を使い、遅延と使用量が増える。
- 判断: 採用しない。

### 3.2 モデルと推論を完全に未指定

Codex の推奨モデルと preset に全面追従する方式。

- 利点: 将来の既定値改善へ自動追従できる。
- 欠点: 実行カタログや surface により実効既定値が変わり、品質特性を固定できない。
- 判断: モデル固有の技術上限には採用するが、モデル名と主要 reasoning 値には採用しない。

### 3.3 用途別の階層設定

グローバル、対象リポジトリ、明示 profile を分離する方式。

- 利点: 通常使用量を抑えながら、複雑な実装と最難関作業の品質を明示的に確保できる。
- 欠点: CLI profile を使う場合は `--profile` の選択が必要。
- 判断: 採用する。

## 4. 設定設計

### 4.1 グローバル設定

対象: `C:\Users\y2ikg\.codex\config.toml`

次をグローバル既定値とする。

```toml
#:schema https://developers.openai.com/codex/config-schema.json

model = "gpt-5.6-sol"
model_reasoning_effort = "medium"
plan_mode_reasoning_effort = "high"
model_verbosity = "low"
model_reasoning_summary = "none"
personality = "pragmatic"

approval_policy = "on-request"
approvals_reviewer = "user"
sandbox_mode = "workspace-write"
web_search = "cached"
```

既存の `[windows]` は次を維持する。

```toml
[windows]
sandbox = "elevated"
```

既存の `[agents]` は次へ変更する。

```toml
[agents]
max_threads = 6
max_depth = 1
```

理由:

- Medium は公式の標準 Power と一致する。
- Plan High は設計品質を確保しつつ、全 Plan turn を XHigh に固定しない。
- Low verbosity はコード変更、コマンド結果、最終説明の冗長化を抑える。
- reasoning summary None は通常ターンの追加出力を避ける。推論能力自体は `model_reasoning_effort` が決める。
- 6 threads と depth 1 は公式既定値で、再帰的 fan-out と過剰な並列使用量を避ける。
- Cached web search は通常調査の既定とし、最新性が必要なタスクだけ live search を明示する。

### 4.2 対象リポジトリ設定

作成対象: `G:\workspace\development\GameEngine\mirakanai-engine\.codex\config.toml`

```toml
#:schema https://developers.openai.com/codex/config-schema.json

model = "gpt-5.6-sol"
model_reasoning_effort = "high"
plan_mode_reasoning_effort = "xhigh"
model_verbosity = "low"
model_reasoning_summary = "concise"
```

理由:

- 独自 C++ ゲームエンジン、Editor、AI orchestration は複数の設計制約と安全境界を持つため、通常実装を High とする。
- Plan mode はアーキテクチャ、依存関係、検証計画を扱うため XHigh とする。
- 最終回答は Low のままにし、実装品質と説明量を分離する。
- Concise summary は長期設計で判断根拠を追跡できる最小限の可視性を確保する。

### 4.3 Fast profile

対象: `C:\Users\y2ikg\.codex\fast.config.toml`

```toml
model = "gpt-5.6-terra"
model_reasoning_effort = "medium"
plan_mode_reasoning_effort = "medium"
model_verbosity = "low"
model_reasoning_summary = "none"
personality = "pragmatic"
service_tier = "fast"
```

用途:

- 高速な探索、軽量修正、反復確認。
- `--profile fast` を明示した場合だけ 2.5 倍の ChatGPT クレジット消費を受け入れる。

### 4.4 Deep profile

対象: `C:\Users\y2ikg\.codex\deep.config.toml`

```toml
model = "gpt-5.6-sol"
model_reasoning_effort = "max"
plan_mode_reasoning_effort = "xhigh"
model_verbosity = "medium"
model_reasoning_summary = "concise"
personality = "none"
sandbox_mode = "read-only"
```

用途:

- 最難関の原因分析、設計検証、深いレビュー。
- Max を通常実装へ流用しない。
- Read-only を維持し、深い調査と変更適用を分離する。

### 4.5 Routine profile

作成対象: `C:\Users\y2ikg\.codex\routine.config.toml`

```toml
model = "gpt-5.6-luna"
model_reasoning_effort = "medium"
plan_mode_reasoning_effort = "medium"
model_verbosity = "low"
model_reasoning_summary = "none"
personality = "pragmatic"
```

用途:

- 完了条件が明確な抽出、分類、変換、構造化要約、機械的な小規模変更。
- Luna / Max は既定にせず、Medium で品質不足を実測した場合だけターン単位で上げる。
- Fast tier は併用せず、Luna の標準速度と低い使用量を活かす。

### 4.6 Scan profile

対象: `C:\Users\y2ikg\.codex\scan.config.toml`

```toml
model = "gpt-5.6-terra"
model_reasoning_effort = "medium"
plan_mode_reasoning_effort = "medium"
model_verbosity = "low"
model_reasoning_summary = "none"
personality = "none"
sandbox_mode = "read-only"
```

用途:

- 大量ファイル探索、読み取り中心の調査、補助情報の要約。

## 5. 未指定にする設定

次のキーは追加しない。

- `model_context_window`
- `model_auto_compact_token_limit`
- `model_auto_compact_token_limit_scope`
- `tool_output_token_limit`
- `model_supports_reasoning_summaries`
- グローバルの `service_tier`
- グローバルの `review_model`

理由:

- Codex のモデルカタログがモデルごとの適切な context、compaction、tool truncation を持つ。
- 手動値はモデル更新後も残り、利用可能な context を不必要に狭めたり、圧縮を早めたりする。
- `model_supports_reasoning_summaries` は能力検出を強制上書きするため、通常は設定しない。
- Fast tier と専用 review model は必要な実行で明示し、全ターンへ適用しない。

## 6. 変更しない範囲

次は今回変更しない。

- MCP server の URL、command、環境変数
- プラグインの有効状態
- GitHub app tool の承認設定
- `notify`
- Desktop UI 設定
- project trust 設定
- shell environment policy
- 認証方式と credential store
- グローバル `AGENTS.md`
- Codex の session DB、rollout、memory DB

既存能力を削らず、今回の目的に直接関係する実行既定値だけを変更する。

## 7. 適用手順

1. ユーザー設定四ファイルの SHA-256 と内容を取得し、プロジェクト設定が未作成であることを確認する。
2. `C:\Users\y2ikg\.codex\config.toml` の同一ディレクトリへ、日時付きバックアップを一つ作成する。
3. グローバル設定の対象キーだけを変更し、MCP、plugin、desktop、app、marketplace、shell environment の各 table を保持する。
4. `js_repl = false` と空になった `[features]` table を削除する。
5. `fast.config.toml`、`deep.config.toml`、`scan.config.toml` を設計値へ更新し、`routine.config.toml` を作成する。
6. リポジトリへ `.codex/config.toml` を作成する。
7. TOML parser と公式 JSON Schema で構文・型を検証する。
8. Codex の実行時コマンドで各設定層と profile の読み込みを検証する。
9. 変更前後の差分を確認し、対象外 table が不変であることを検証する。
10. グローバル設定変更を反映するため、新しい Codex task で実効値を確認する。

## 8. 検証

最低限、次を実行する。

```powershell
codex doctor --json
codex features list
codex debug models
codex --profile fast debug prompt-input "profile validation"
codex --profile deep debug prompt-input "profile validation"
codex --profile routine debug prompt-input "profile validation"
codex --profile scan debug prompt-input "profile validation"
```

追加検証:

- Python 3.11 以降の `tomllib` で五つのユーザー設定とプロジェクト設定を parse する。
- 公式 `https://developers.openai.com/codex/config-schema.json` に対して設定を検証する。
- 通常設定では `doctor` の `config.load`、各 profile では `debug prompt-input` の終了コードと JSON parse が正常であることを確認する。
- 通常 profile の model が Sol、Fast と Scan が Terra、Routine が Luna、Deep が Sol であることを確認する。
- 削除済み feature override が残っていないことを確認する。
- `git diff --check` と `git diff` でリポジトリ変更を確認する。

CLI 0.144.5 の `--profile` は `doctor` へ適用できないため、profile の非推論ロード検証には `codex --profile <name> debug prompt-input` を使用する。

## 9. ロールバック

- グローバル設定で起動または検証に失敗した場合は、作成した日時付きバックアップから `C:\Users\y2ikg\.codex\config.toml` を復元する。
- profile の検証に失敗した場合は、変更前に取得した内容へ該当 profile だけを戻す。
- プロジェクト設定が原因の場合は、`.codex/config.toml` を削除してグローバル設定へ戻す。
- ロールバック後に `codex doctor --json` を再実行し、`config.load = ok` を確認する。

## 10. 完了条件

次をすべて満たしたとき完了とする。

1. グローバル、プロジェクト、Fast、Deep、Routine、Scan の設定が設計値と一致する。
2. 全 TOML が parse できる。
3. Codex 0.144.5 が通常設定と各 profile を正常に読み込む。
4. 通常設定に `max`、`service_tier = "fast"`、削除済み feature が存在しない。
5. 対象リポジトリでは Sol / High / XHigh Plan が有効になる。
6. MCP、plugin、desktop、app、marketplace、notify、shell environment の既存設定が保持される。
7. 変更差分、検証結果、バックアップ位置、残存リスクを最終報告する。

## 11. 公式資料

- Codex Models: https://learn.chatgpt.com/docs/models
- Config basics: https://learn.chatgpt.com/docs/config-file/config-basic
- Configuration Reference: https://learn.chatgpt.com/docs/config-file/config-reference
- Advanced Configuration / Profiles: https://learn.chatgpt.com/docs/config-file/config-advanced#profiles
- Sample Configuration: https://learn.chatgpt.com/docs/config-file/config-sample
- Speed / Fast mode: https://learn.chatgpt.com/docs/agent-configuration/speed
- Codex pricing and usage limits: https://learn.chatgpt.com/docs/pricing
- GPT-5.6 launch and coding evaluations: https://openai.com/index/gpt-5-6/
- GPT-5.6 System Card: https://deploymentsafety.openai.com/gpt-5-6
- GPT-5.6 Sol: https://developers.openai.com/api/docs/models/gpt-5.6-sol
- GPT-5.6 Terra: https://developers.openai.com/api/docs/models/gpt-5.6-terra
- GPT-5.6 Luna: https://developers.openai.com/api/docs/models/gpt-5.6-luna
- GPT-5.6 model and reasoning guidance: https://developers.openai.com/api/docs/guides/latest-model
- GPT-5.6 prompting guidance: https://developers.openai.com/api/docs/guides/prompt-guidance-gpt-5p6
