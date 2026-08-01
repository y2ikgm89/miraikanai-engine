# ChatGPT Pro Collaboration v3 Clean-Break Design

- 文書分類: `non-normative implementation design`
- 状態: `written-spec-review-pending`
- 対象: Personal Skill `collaborating-with-chatgpt-pro`、そのLocal MCP Server、検証suite、repository固有route／retention記述
- 判断日: 2026-08-01
- 外部根拠確認日: 2026-08-01
- 実装状態: `absent`。本書作成時点では既存v2実装がinstallされている
- 後方互換性: `none`。旧schema、旧CLI、旧evidence形式、旧chatを引き継がない
- Architecture authority: none。本書はArchitecture Owner、ADR、実装、qualification、releaseまたはProduct completionの正本ではない

## 1. 結論

Personal Skillを、Browser UIからcall-level evidenceを転記するv2方式から、Local MCP
Serverが認可grantへ結び付いた機械Receiptを発行するv3方式へ置き換える。

品質、Security、監査可能性、網羅性を固定したうえで、重複prompt、不要なTool、毎回の
reference読込、process起動、手動evidence構成だけを削減する。最適化の優先順位は次で固定する。

```text
correctness／security／auditability
  > coverage
  > reproducibility
  > latency／context／usage
```

上位条件と競合する最適化は採用しない。Task固有の制約、成功条件、必要Artifact、試験を
削って速度またはcontext目標を満たすことを禁止する。

## 2. 根拠分類

### 2.1 公式仕様で確認した原則

| 原則 | 根拠 |
| --- | --- |
| Promptは`Goal`、`Context`、`Constraints`、`Done when`を明確にし、結果を変えるcontextだけを与える | [OpenAI Best practices](https://learn.chatgpt.com/guides/best-practices)、[Prompting](https://learn.chatgpt.com/docs/prompting) |
| `AGENTS.md`はdurableなrepository guidanceに使い、短く実用的に保つ | [OpenAI Best practices](https://learn.chatgpt.com/guides/best-practices) |
| Skillは一つの認識可能なgoalに絞り、input、steps、output、stop条件を明確にする | [Build skills](https://developers.openai.com/plugins/build/skills) |
| `SKILL.md`を簡潔にし、policy／schemaは`references/`、決定的処理は`scripts/`へ置く | [Build skills](https://developers.openai.com/plugins/build/skills) |
| Skillはworkflowを、MCP Serverはlive data、authentication、authorization、controlled actionを所有する | [Build skills](https://developers.openai.com/plugins/build/skills) |
| 異なる成果は新しいchatへ分離し、self-containedな作業にはProject外chatを使える | [Projects and chats](https://learn.chatgpt.com/docs/projects) |
| private MCP ServerはSecure MCP Tunnelによりpublic internetへ直接公開せずdeveloper modeで接続できる | [Secure MCP Tunnel](https://developers.openai.com/api/docs/guides/secure-mcp-tunnels) |
| ChatGPT互換のread-only knowledge surfaceではstandard `search`／`fetch`が基礎Toolになる | [Build MCP servers](https://developers.openai.com/plugins/build/mcp-server) |

上表はOpenAIの公開一次資料で確認した原則である。次項の固定route、schema、timeout、
Receipt実装またはretention方式をOpenAI公式推奨とは呼ばない。

### 2.2 Miraikanaiの採用判断

次は`project-decision`である。

- fresh non-Project Browser ChatGPT `Chat` surfaceを使用する。
- response performanceはselected／collapsedともに`Pro`を必須とする。
- exact `G Workspace Readonly` appとSecure MCP Tunnelだけを許可する。
- Local Receipt StoreにPython標準SQLiteを使用する。
- response waitを1,200秒、in-flight drainを30秒、grant TTLを1,800秒とする。
- full transcriptはlocal adjudication／closure後に削除し、compact review summaryだけを残す。

### 2.3 実測事実

2026-08-01の同一Windows環境で次を観測した。

| 項目 | 実測 |
| --- | ---: |
| `validate_skill.ps1` full gate | 137.544秒 |
| `test_validate_consultation_evidence.ps1` | 101.765秒 |
| Local MCP pytest | 153 passed、約22.91秒 |
| 常時参照される現行Skill／contract／runbookの概算 | 約2,791 tokens |
| 6 Artifact consultation prompt | 3,471 UTF-8 bytes |
| 6 Artifact総source size | 56,670 bytes |
| Browser ChatGPT Pro generation | 14分29秒 |
| Tunnel sanitized forward event delta | 24 |

Pro応答はexact completion markerまで到達したが、現行ChatGPT UIは個別MCP
`call_id`、exact Tool／target／status、Artifact digestを表示しなかった。このため、
`consultation-evidence/1`を推測なしで構成できず、acceptanceはfail-closedで`blocked`とした。

追加のlocal code inspectionでは、v2 validatorが
`route.forbidden_path_prefixes[0]`だけを検査し、配列の残りを無視することを確認した。

## 3. 採用案と棄却案

### 3.1 採用: grant-bound transactional Receipt

Local MCP Serverが、各Tool callをfile access前から完了までgrant-bound Receiptとして記録する。
Browser UIはroute、surface、performance、app、visible completionだけを観測し、Tool／Artifact
evidenceの正本にはしない。

この方式は、現在のUIで取得不能なfieldを要求せず、MCP Serverのauthorization boundaryと
evidence boundaryを一致させる。

### 3.2 棄却: v2 validatorの部分補修

全forbidden prefix検査、DOM selector修正、UI Activity解析だけを追加する案は棄却する。
現在のUIがcall-level fieldを公開しないため、selectorを変更しても完全なevidenceを作れない。

### 3.3 棄却: Browserを所有する常駐coordinator

Browser DOM、generation、clipboard、Tunnel、grantを一つのdaemonが制御する案は棄却する。
Browser UI versionへの結合、障害時のcleanup、認証sessionの所有範囲が増え、今回必要な
local evidence改善を超える。

## 4. 責務境界

| Component | 唯一の責務 |
| --- | --- |
| `SKILL.md` | trigger、正常系の短いworkflow、stop／handoff条件 |
| `references/consultation-contract.json` | installed fixed invariantとproject-decision値 |
| `references/runbook.md` | incident、recovery、retention、maintenance時の判断。正常系では読まない |
| `scripts/consultation.ps1` | model-facingな唯一のlifecycle entry point |
| Python consultation core | strict schema、Task／Manifest／Prompt生成、state transition、Receipt snapshot検証 |
| Local MCP Server | exact read authorization、Tool実行、transactional Receipt記録 |
| Secure MCP Tunnel helper | exact profile、runtime、catalog、root、read-only capabilityのreadiness確認 |
| Browser controller | fresh Chatの作成、visible UI invariant確認、prompt送信、stable completion観測 |
| Codex | scope closure、ChatGPT findingのlocal adjudication、repository diff／test／retention closure |
| repository `AGENTS.md` | このrepositoryだけのdestination／fallback／root／retention policy |
| `docs/reviews/README.md` | non-normative compact audit summary |

`agents/openai.yaml`にはCodexが直接使用しないBrowser-side MCP dependencyを偽って宣言しない。
Browser ChatGPTがMCP clientであることをSkill本文に明示し、runtime helperで依存を検証する。

## 5. Clean-break contract

### 5.1 Installed contract

`consultation-contract/3`だけを受理する。rootはexact key setで、未知field、重複JSON
property、型違反を拒否する。

```text
schema
route
response_performance
browser_app
tunnel
delivery
tool_requirements
artifact_integrity
runtime_authorization
receipt_store
deadlines
follow_up_gates
```

`route`は次を所有する。

- `start_url = https://chatgpt.com/`
- `required_surface = Chat`
- `project_membership = denied`
- `forbidden_path_prefixes[]`

`forbidden_path_prefixes`はnon-empty、Ordinal uniqueなstring arrayとし、validationでは全要素を
評価する。先頭要素だけを特別扱いしない。

`tunnel.tools`はexact ordered set `search, fetch`とする。`delivery`はbrowser upload、paste、
Project Source、Project memory、workspace write、alternate routeをすべて`denied`とする。
Receipt DBなど`LOCALAPPDATA`内のcontrol-state writeだけを明示的に許可し、
`G:\workspace`へのwriteとは区別する。

### 5.2 Task Contract

`consultation-task/2`のexact root keyは次とする。

```text
schema
task_id
goal
context[]
constraints[]
done_when[]
output_requirements[]
artifact_ids[]
search_query
stop_conditions[]
```

- `task_id`はnon-empty stable identityとする。
- `goal`は一つのdistinct outcomeだけを表す。
- `context`は結果を変える背景だけを含み、Local Artifact本文を含めない。
- `constraints`、`done_when`、`output_requirements`、`stop_conditions`はnon-empty string arrayとする。
- `artifact_ids`は`G:\workspace` root-relative、forward-slash、Windows identityでuniqueな
  exact file IDだけを含む。directory scope、descendant authorization、exclusion listを廃止する。
- `search_query`は`null`またはnon-empty string一件とする。`null`ならsearch authorizationはない。
- upload／write禁止などinstalled invariantをTaskへ重複記録しない。

Artifact選定前にCodexは対象repositoryのsource of truth、関連code／test／config、Owner／依存、
現在のuser changeを調べ、必要Artifact集合のscope closureを行う。Materialな不足が残る場合は
Manifestを広げるかdistinct outcomeへ分割し、不完全な入力を完全と表現しない。

### 5.3 Artifact Manifest

`artifact-manifest/2`は次を持つ。

```text
schema
task_sha256
artifacts[] {
  artifact_id
  source_bytes
  source_sha256
  extraction_status
}
```

Taskの`artifact_ids`とManifestの`artifact_id`はorderを含むexact equalityとする。
Manifest生成は同じpinned extractorとpath policyを使用する。parent IDと
`authorization_ref`はArtifact IDから導出できるため削除する。

### 5.4 Runtime Authorization

`mcp-read-authorization/2`はgrant、Task、turn、expiry、Task／Manifest digest、optional exact
search query、ordered Artifact metadataを結合する。state digestはdigest fieldを除くstrict
canonical JSONのUTF-8 bytesに対するlowercase SHA-256とする。canonical JSONは全object keyを
Unicode code point順にsortし、insignificant whitespaceなし、UTF-8、BOMなし、non-ASCIIを
escapeしない表現へ固定する。

旧`mcp-read-authorization/1`、unknown schema、future timestamp、expired state、digest mismatch、
case-only duplicate、unsafe pathを拒否する。互換parserまたは自動変換を設けない。

## 6. Tool contract

### 6.1 Catalog

MCP catalogはread-only annotation付き`search`と`fetch`だけを公開する。実装時に現行OpenAI
一次資料のstandard input／output shapeへ再照合し、catalog self-testでexact equalityを検証する。

### 6.2 Canonical plan

expected planはTaskとManifestから一意に導出する。

1. `search_query != null`のときだけ、exact queryで`search`を一回実行する。
2. Manifest順に各Artifact IDを`fetch`でexact一回取得する。

`search_query == null`のsearch、unknown Tool、順序違反、重複、欠落、failed call、Manifest外IDを
すべて`blocked`とする。modelが書く`requirement_id`、手作業の`expected_tool_calls`、
UI由来の`observed_tool_calls`は廃止する。

現行`list_allowed_directories`と`list_directory`は、Manifestで既に確定したroot／IDを再提示する
だけであり、取得可能Artifactを増やさないため削除する。`search`は認可済みManifest内の
independent navigation用途として保持する。

### 6.3 Minimal Tool result

ChatGPTへ返すTool resultはstandard shapeを維持しつつ、Task回答に不要なgrant ID、request ID、
telemetry、内部path、Receipt metadataを含めない。Artifact integrity metadataはLocal Receiptへ
保存し、modelへ同じhashを繰り返し読ませない。

## 7. Transactional Receipt

### 7.1 Storage

Pinned Python 3.14.4に含まれるSQLite 3.50.4を使用し、新しいproduction dependencyは追加しない。
DBはinstalled contractから導出した`LOCALAPPDATA`配下へ置き、`G:\workspace`、repository、skill
directory、Browser-accessible rootとの包含関係を禁止する。

DBはsession tableとcall tableを持つ。session statusは次の一方向遷移だけを許可する。

```text
prepared -> active -> closing -> complete | blocked -> purged
prepared -> aborted -> purged
active   -> closing -> blocked -> purged
```

未知status、逆遷移、identity mismatchを拒否する。
同一Receipt Storeでは`prepared | active | closing`のnon-terminal sessionをexact一件だけ許可し、
processをまたぐ同時Prepare／Arm／Closeは同じnamed lifecycle mutexで直列化する。

### 7.2 Call Receipt

各callは次を記録する。

```text
ordinal
call_id
grant_id
task_id
turn_id
tool
target
invalid_target_sha256
started_at_utc
completed_at_utc
status = pending | success | failed
error_code
source_bytes
source_sha256
extraction_status
```

- `call_id`はserver生成のlowercase canonical UUIDv4とする。
- `ordinal`は同一grant内でtransactionalに単調増加させる。
- canonical targetだけを平文保存する。invalid inputは値を保存せずSHA-256だけを残す。
- `error_code`はstable sanitized codeとし、exception text、absolute path、secretを保存しない。
- source metadataはsuccessful `fetch`だけが持ち、Manifestとexact一致しなければTool自体を失敗させる。

MCP Serverはauthorizationを読んだ後、file／extractorへ触れる前に`pending` Receiptを作成する。
Receipt admissionまたはDB transactionが失敗した場合、file accessへ進まない。Tool resultを返す前に
Receiptを`success`または`failed`へ確定する。completion updateが失敗したcallは成功扱いしない。

### 7.3 Concurrency and close race

Closeは次の順序で行う。

1. DB transactionでsessionを`closing`にし、新しいReceipt admissionを拒否する。
2. exact task／turn／grant identityのauthorization stateをatomicにrevokeする。
3. 既に`pending`となったcallを最大30秒drainする。
4. pendingが残れば`blocked`とする。
5. ordered Receipt snapshotをcanonical JSONとして出力し、そのSHA-256を記録する。
6. snapshot、Task、Manifest、UI observation、telemetryを検証する。

authorizationを読んだ直後にcloseと競合したcallも、Receipt admissionが`closing`を観測すれば
file access前に拒否される。close前にadmit済みのcallだけをdrain対象とする。

## 8. Lifecycle interface

Model-facing public helperは`scripts/consultation.ps1`一つに統合する。

### 8.1 `Prepare`

- strict Taskを生成する。
- full Tunnel lifecycle／profile／runtime／catalog／root checkを実行する。
- Manifestとdeterministic promptを生成する。
- Receipt Storeへgrantを持たない`prepared` sessionを作成する。
- Artifact count、source bytes、prompt bytesをsanitized metricsとして返す。
- authorization grantは作成しない。

### 8.2 Browser configuration

Browser controllerはfresh `https://chatgpt.com/`を開き、Project外の`Chat` surface、selected／collapsed
`Pro`、exact `G Workspace Readonly`、read-only root、exact `search, fetch` catalogを可視確認する。
landing defaultの`Work`をstandalone Chatと同一視しない。

Radio／menu stateはDOM property `checked`ではなく、accessible selected stateを含む可視UI状態で
判定する。UI文言または構造が変わりexact invariantを確認できなければ送信しない。

### 8.3 `Arm`

- lightweight Tunnel readinessを再確認する。
- Browser controllerが渡すpre-send observationをstrict schemaで検証する。
- telemetry baselineを取得する。
- exact prepared Receipt sessionを`active`へ遷移させ、authorization grantをactivateする。
- exact prompt path、task／turn／grant identity、deadlineを返す。

`Arm`成功後はprompt以外のBrowser configurationを変更せず、直ちに一回だけ送信する。
Receipt transitionとauthorization state writeは同じlifecycle mutex内で行う。authorization state
writeが完了しない場合はReceipt sessionを`blocked`へ閉じ、active authorizationが存在しないことを
確認してから失敗を返す。

### 8.4 Wait

response待機上限は`Arm`から1,200秒とする。`今すぐ回答`、model downgrade、別surface、別chatへの
切替を行わない。visible completionは次のすべてを必要とする。

- generation controlが停止済み。
- exact `CONSULTATION_COMPLETE::<task_id>::<turn_id>`がvisible response末尾にある。
- 2秒以上離れた二回の観測で末尾とgeneration stateが変化しない。

期限超過、Browser失敗、marker欠落、UI不安定は`Abort`へ進む。

### 8.5 `Complete`／`Abort`

`Complete`は§7.3のcloseを実施し、Receipt、Artifact、route、performance、app、marker、positive
Tunnel telemetryを検証する。`Abort`も同じclose／revoke gateを通り、理由をsanitized codeで残す。

両Modeは同じsession identityに対してidempotentとする。他sessionのactive grantを削除しない。
内部例外が発生しても`finally`でexact revokeを試みる。revoke identity mismatchまたはrevoke failureは
terminal `authorization-revocation-failed`とし、その後Browser／MCP callを行わない。

### 8.6 `Purge`

Local adjudication、accepted finding反映、dependency／consumer closure、terminal audit、repository
review summary更新が完了した後だけ、exact sessionのReceipt、prompt、response、Manifest、Task、
screenshot等を削除する。別sessionまたは広いdirectoryを対象にしない。

## 9. Prompt and context design

Promptはhelperが次の順に一意生成する。

```text
Goal
Context
Constraints
Done when
Output
Authorized Artifact IDs
Exact Tool plan
Stop conditions
Completion marker
```

- Local本文、absolute path、authorization state、digest、expected bytes、Receipt fieldをpromptへ貼らない。
- Artifact IDは一度だけ列挙し、各ID自身がfetch authorizationであることを一文で示す。
- installed route禁止事項をTaskごとに長く複写せず、回答に必要なTool／stop条件だけを書く。
- Task固有情報を削除、要約またはtruncateしてbyte目標を満たさない。
- 一つのdistinct outcomeを一つのfresh chatで扱う。Materialに異なる成果は新しいTaskへ分割する。

同一6 Artifact baseline fixtureでprompt UTF-8 bytesを現行3,471の70%以下にする。ただしこれは
同一意味内容での重複削減目標であり、quality gateではない。実token使用量はprompt bytesだけから
断定せず、Artifact bytes、Tool call count、wall timeと別々に報告する。

`SKILL.md`と正常系でalways-readとなる本文の合計を4 KiB以下にする。正常系ではcontract JSONと
runbookをmodel contextへ読まない。helperがcontractを機械読取し、runbookはblocked、incident、
recovery、maintenanceのときだけ読む。

## 10. Validation result

`consultation-result/2`はexact key setを持つsanitized JSONとする。

```text
schema
status = complete | blocked | error
reason
task_id
turn_id
grant_id
receipt_snapshot_sha256
affected_artifact_ids[]
affected_call_ids[]
metrics
```

少なくとも次のstable reasonを区別する。

- `scope-incomplete`
- `tunnel-not-ready`
- `route-mismatch`
- `surface-mismatch`
- `response-performance-mismatch`
- `app-identity-mismatch`
- `tool-catalog-mismatch`
- `authorization-invalid`
- `authorization-revocation-failed`
- `response-timeout`
- `response-incomplete`
- `receipt-missing`
- `receipt-pending`
- `tool-plan-mismatch`
- `unauthorized-tool-call`
- `artifact-integrity-mismatch`
- `unexpected-write`
- `validator-runtime-failure`

stdoutは結果JSON一件だけ、stderrはdeveloper test以外でsecret／absolute pathを含めない。
invalid inputはexit 2の`blocked`、内部障害はexit 3の`error`、completeはexit 0とする。

## 11. Quality non-regression gates

速度またはcontext最適化より前に次をすべて満たす。

1. v2が検出できたschema、route、performance、app、catalog、authorization、cardinality、order、
   integrity、write、completion、telemetry異常をv3でも検出する。
2. UIで取得不能だったcall-level identity、status、Artifact integrityをmachine Receiptで追加検証する。
3. Task Artifactの欠落を成功として扱わない。
4. Pro、full response、local adjudication、repository source of truthを維持する。
5. test削除、assertion緩和、error統合、入力truncateでperformance目標を満たさない。
6. 同じfixtureの全既知異常が同じか、より具体的なblocked reasonへ到達する。
7. live acceptanceでresponse proseだけからTool実行を推測しない。

このgateを一つでも満たさない最適化は不採用とする。

## 12. Test design

実装はtest-drivenで行う。最初にv3 behaviorを表す失敗testを追加し、最小実装、refactor、full
verificationの順に進む。

### 12.1 Pure tests

同一Python processで次を検証する。

- contract／Task／Manifest／authorization／observation／Receipt snapshotのstrict parser。
- duplicate JSON property、unknown field、scalar／array substitution、全forbidden prefix。
- Windows OrdinalIgnoreCase identity、Unicode distinct identity、unsafe path、case-only duplicate。
- canonical plan derivation、optional search、fetch order、missing／duplicate／extra／failed call。
- Receipt state transition、transaction rollback、pending drain、close race、server restart。
- source bytes／SHA-256／extraction status mismatch。
- prompt determinism、Artifact ID一回、本文／absolute path／secret非包含。
- sanitized stdout／stderr／result envelope。
- v2以上の61 independent field mutation。

### 12.2 Process-boundary tests

`pwsh` subprocess testは最大6 test caseに限定する。

1. complete exit 0。
2. schema／policy blocked exit 2。
3. runtime failure exit 3。
4. missing input。
5. sanitized output／path secrecy。
6. exact revoke／idempotent abort／mismatched grant protection。

Mutationごとのprocess spawnを廃止し、coverageは減らさない。

### 12.3 MCP tests

- exact `search, fetch` catalogとread-only annotations。
- Receipt作成前のfilesystem sink非到達。
- authorizationなし、expired、tampered、legacy schemaのfail-closed。
- Manifest外、path escape、unknown Tool、invalid argument、duplicate fetch。
- concurrent callとcloseのlinearizable behavior。
- Receipt completion失敗時にsuccess Tool resultを返さない。
- Receipt DBがallowed rootまたはprotected rootと重ならない。

### 12.4 Performance acceptance

同じmachine、同じfixture、warm／cold条件を明記してmedianを比較する。

- evidence mutation suiteは101.765秒baselineの30%以下。
- full gateは60秒以下。
- 6 Artifact promptは3,471 bytes baselineの70%以下。
- normal pathのexposed Tool catalogは4個から2個。
- normal pathのnon-fetch mandatory callは、searchなしTaskで0件。

時間目標を満たさなくても品質gateを緩和しない。原因をprofileし、process／I/O／重複計算だけを
改善する。

### 12.5 Live acceptance

実装後、final skill Artifactを対象にfresh non-Project Browser ChatGPT `Chat`／`Pro`／exact appで
一回のself-auditを行う。次をすべて必要とする。

- exact app catalog `search, fetch`。
- Task／Manifest由来planとのReceipt全件一致。
- Artifact integrity全件一致。
- unknown、duplicate、missing、failed、pending call 0。
- positive Tunnel telemetry delta。
- exact visible-end marker。
- close後にactive authorizationなし。
- local adjudicationとrepository retention closure。

UIがcall cardを保持しないことはfailureではない。Receiptが欠落する場合はfailureとする。

## 13. Files and migration

### 13.1 Personal Skill

実装対象は次である。

- `SKILL.md`
- `agents/openai.yaml`
- `references/consultation-contract.json`
- `references/runbook.md`
- `scripts/consultation.ps1`
- `scripts/ensure_secure_mcp_tunnel.ps1`
- `scripts/validate_skill.ps1`
- Python consultation／Receipt modules
- Local MCP Server catalog／execution instrumentation
- PowerShell／Python tests

旧public helper
`new_consultation_task.ps1`、`new_artifact_manifest.ps1`、
`set_consultation_authorization.ps1`、`get_tunnel_telemetry.ps1`、
`validate_consultation_evidence.ps1`と対応testを削除する。alias、wrapper、schema translationを
残さない。再利用するpure logicはv3 moduleへ移し、旧interfaceを公開しない。

MCP package versionは`4.0.0`へ上げる。install中にactive grantがある場合は変更を開始しない。
変更後はTunnel processをexact profileだけ再起動し、Browser app metadataをRefreshして
`search, fetch`だけであることを確認する。pre-clean-break chatは破棄し、新しいchatだけを使う。

### 13.2 Repository

- `AGENTS.md`のChatGPT Pro routeを`Chat` surface明示、v3 lifecycle委譲、禁止fallback中心へ短縮する。
- `docs/reviews/README.md`へ今回のaudit ID、日付、route／mode、6入力Artifactとdigest、locally
  accepted gap、反映先、closure、exact marker、response digest availability、retention dispositionを
  記録する。
- Architecture Owner／ADRは、今回のSkill implementationを正本化しないため変更しない。

## 14. Retention

現在の`docs/reviews/.transient/chatgpt-pro-skill-optimization-20260801-audit1`は、設計／実装／live
acceptance／local adjudication／summary更新が終わるまで保持する。全文応答は取得不能だったため
復元または推測しない。

closure後は次を順に行う。

1. `docs/reviews/README.md`へcompact summaryを追加する。
2. summaryの件数、digest、marker、blocked／complete判定、反映先を検証する。
3. exact transient consultation directory、Temp Task／Manifest、Receipt sessionを削除する。
4. 削除対象外のtransient data、別session、repository user dataを変更していないことを確認する。

## 15. Completion criteria

次のすべてが成立したときだけimplementationをcompleteと報告する。

1. v3 schema、single lifecycle CLI、transactional Receipt、two-Tool MCP catalogが実装済み。
2. legacy interface／schema／testが削除され、compatibility aliasがない。
3. quality non-regression gate、pure／boundary／MCP／performance testが合格。
4. fresh standalone Chat／Pro live acceptanceがmachine Receiptでcomplete。
5. ChatGPT findingsをCodexがlocal source of truthでadjudicate済み。
6. repository `AGENTS.md`とreview summaryが実態に一致。
7. transient evidenceがretention policyどおり整理済み。
8. full personal-skill corpus diff、repository diff、`git diff --check`、`git status --short`、
   `git diff --stat`を確認済み。

文書の存在または承認だけから、上記implementation、acceptance、qualification、closureを推測しない。
