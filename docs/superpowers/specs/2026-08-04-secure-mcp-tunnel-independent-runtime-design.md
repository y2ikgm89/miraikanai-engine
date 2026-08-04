# Secure MCP Tunnel Independent Runtime／Logon Startup Design

- 文書状態: approved design
- 正本性: non-normative local-operations design rationale
- 対象: 個人Windows環境の`g-workspace-readonly` Secure MCP Tunnel、独立read-only MCP server、ユーザー単位Scheduled Task
- 決定: 廃止済みSkillへの依存を除去し、専用MCP runtimeを`%LOCALAPPDATA%`へ分離して、ログイン時に最小権限でTunnelを自動起動する
- 承認: 推奨案を2026-08-04にユーザー承認
- Local runtime実装状態: materialized and locally verified on 2026-08-04; real-logon trigger acceptance pending
- Repository Architecture Owner影響: none
- 外部根拠確認日: 2026-08-04

## 1. 結論

`g-workspace-readonly` Profileが廃止済みPersonal Skill配下のPythonを参照しているため、現在の`tunnel-client run`は起動直後に終了する。Skillを復元せず、専用read-only MCP serverを次の独立runtimeへ新設する。

```text
%LOCALAPPDATA%\MCP\g-workspace-readonly\
```

Runtimeは現在インストール済みのCPython 3.14.6から専用`.venv`を作成し、uv 0.12.1で依存を解決、lock、同期する。Tunnel Profileはuv、Skill、Codex内部runtime、system-wide PythonまたはPATHを介さず、専用`.venv\Scripts\python.exe`を絶対Pathで直接起動する。

Tunnelは現在のWindowsユーザーに限定したScheduled Taskでログイン30秒後に起動する。Windows Service、管理者権限、password保存、Task XMLへのsecret埋込み、無限再起動または自動updateは採用しない。

本設計はRepository Engine実装、Architecture Owner、Schema、Registry、Build、CIまたはProduct状態を変更しない。Repositoryには採択理由だけを残し、Personal runtimeとWindows Taskは承認後の実装計画で作成・検証する。

## 2. 現状証拠と根本原因

2026-08-04に次をLocalで確認した。

- 公式`tunnel-client` executableは`%LOCALAPPDATA%\OpenAI\secure-mcp-tunnel\bin\tunnel-client.exe`に存在する。
- versionは`0.0.10`で、OpenAI公開latest releaseと一致する。
- `%APPDATA%\tunnel-client\g-workspace-readonly.yaml`は存在し、Profile load、Tunnel identity、runtime API key reference、health listenerは`doctor`でPASSする。
- `CONTROL_PLANE_API_KEY`はProcess scopeとUser scopeに存在し、Machine scopeには存在しない。
- ProfileのMCP commandは廃止済み`C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\.venv\Scripts\python.exe`を参照する。
- Skill directoryと参照Pythonは存在しない。
- `tunnel-client doctor --profile g-workspace-readonly --explain`の唯一のFAILは`mcp_command_executable`である。
- 起動試行Processは直後に終了し、`/healthz`と`/readyz`は到達不能、残存Tunnel Processは0件である。

根本原因はTunnel client、Profile identity、API key、OpenAI associationまたはportではなく、Profileがlifecycleを終えたSkill所有runtimeへ結合されていたことである。Skillを復元する修正は同じlifecycle couplingを再導入するため採用しない。

## 3. Official factとLocal decision

### 3.1 OpenAI／MCP／Windowsの公式事実

- [OpenAI Secure MCP Tunnel](https://developers.openai.com/api/docs/guides/secure-mcp-tunnels)は、`tunnel-client`がprivate MCP serverへ到達できるnetwork内で稼働し、OpenAIへoutbound HTTPSでpollし、Local MCP requestをforwardする方式を定義する。public inbound portは不要である。
- 同Guideはnamed Profileを`run --profile`で起動し、`doctor`、`/healthz`、`/readyz`、`/metrics`、loopback-only `/ui`で状態を確認することを示す。
- [OpenAI tunnel-client v0.0.10](https://github.com/openai/tunnel-client/releases/tag/v0.0.10)はcontrol-plane poll成功後だけreadyを報告し、stdio recovery、shutdown、OAuth、local proxy、admin UI securityを改善している。
- [MCP Python SDK v2.0.0](https://github.com/modelcontextprotocol/python-sdk/releases/tag/v2.0.0)はstable v2であり、2026-07-28 protocolと以前のrevisionを同じServerから提供する。
- [MCP Python SDK package metadata](https://github.com/modelcontextprotocol/python-sdk/blob/v2.0.0/pyproject.toml)はPython `>=3.10`とPython 3.14 supportを宣言する。
- [uv](https://github.com/astral-sh/uv)は`.python-version`、`uv.lock`、`uv sync --locked`でproject interpreterとlocked environmentを管理できる。
- Microsoft Task Schedulerは[Logon Trigger](https://learn.microsoft.com/en-us/windows/win32/taskschd/logontrigger)、delay、[least-privilege security context](https://learn.microsoft.com/en-us/windows/win32/taskschd/security-contexts-for-running-tasks)、[bounded restart-on-failure](https://learn.microsoft.com/en-us/windows/win32/taskschd/taskschedulerschema-restartonfailure-settingstype-element)を提供する。

### 3.2 本環境のProject decision

次は外部組織の固定推奨ではなく、この個人Windows環境の採用判断である。

- Runtime rootを`%LOCALAPPDATA%\MCP\g-workspace-readonly`へ固定する。
- Python contractを`>=3.14,<3.15`とし、初回materializationは3.14.6で行う。
- uv 0.12.1を初回lock／sync toolとするが、Tunnel実行時dependencyにはしない。
- allowed rootをreal path `G:\workspace`だけに固定する。
- 公開Toolをexact 4 read-only Toolsへ限定する。
- 現在のユーザーのlogin 30秒後にTaskを起動する。
- restartを1分間隔、最大3回へ限定する。
- idle負荷とfile transfer負荷をLocalで実測し、測定なしに性能値を公式値または保証値と呼ばない。

## 4. 比較した方式

### 4.1 必要時だけ手動起動

外部接続時間は最小になるが、ChatGPT／Codex利用前の起動忘れと未readyを毎回operatorが解決する必要がある。日常利用する本環境では可用性が不足するため不採用とする。

### 4.2 ユーザー単位Scheduled Task

現在のユーザーがログインした時だけ起動し、User scope secret、User ACL、low run levelを維持できる。ログイン遅延、single instance、bounded restartも宣言できるため採用する。

### 4.3 Windows Service

login前から稼働できるが、service account、credential delivery、Profile／MCP root ACL、update、shutdown、recoveryの責任が増える。個人developer-mode routeには過剰なため不採用とする。

## 5. Independent MCP runtime

### 5.1 配置と責任

```text
%LOCALAPPDATA%\MCP\g-workspace-readonly\
  .python-version
  pyproject.toml
  uv.lock
  .venv\
  src\readonly_local_files\
    __init__.py
    models.py
    paths.py
    extractors.py
    server.py
  tests\
    test_catalog.py
    test_paths.py
    test_extractors.py
    test_stdio.py
```

- `.python-version`はPython 3.14 seriesを選択する。
- `pyproject.toml`は`requires-python = ">=3.14,<3.15"`とdirect dependencyを宣言する。
- `uv.lock`はtransitive dependency、source artifact、hashを固定する。
- `.venv`はlockからmaterializeしたruntimeであり、手動`pip install`またはsystem packageを混在させない。
- Profile commandは`.venv\Scripts\python.exe -m readonly_local_files.server`だけを参照する。
- API key、Tunnel ID、Profile contentまたはallowed root外PathをMCP command、Tool result、test logへ含めない。

### 5.2 Tool catalog

Serverが公開するToolを次の4個だけに固定する。

1. `list_allowed_directories`
2. `list_directory`
3. `search`
4. `fetch`

`write_file`、`edit_file`、`move_file`、`create_directory`、`delete`、shell、arbitrary process、arbitrary URLまたはcompatibility aliasは公開も実装もしない。全Toolは`readOnlyHint: true`、`destructiveHint: false`、`idempotentHint: true`、`openWorldHint: false`を持つ。Annotationはhintであり、書込みhandlerを実装しないことがsecurity boundaryである。

`search`と`fetch`はOpenAI connector compatibility用の明示的output schemaとstructured resultを返す。`fetch`はread直前のsource bytesからbyte countとSHA-256を計算し、Tool resultのmetadataへ含める。

### 5.3 Read scope

初期runtimeは既存contractを維持し、次をboundedに扱う。

- UTF-8／UTF-16 BOM text、Markdown、source、JSON、YAML、TOML、CSV、TSV。
- signature確認済みPNG、JPEG、WebP、non-animated GIF。
- text-bearing PDF。
- DOCX、PPTX、XLSXのtext／cell value。

legacy Office、macro-enabled Office、encrypted file、scan-only PDF、SVG、未知binary、size limit超過またはresult limit超過はtyped errorでfail closedにする。Parserはmacro、OLE、embedded executable、external linkまたはformulaを実行しない。

### 5.4 Path containment

全Tool inputはroot-relative pathまたはserverが発行したstable IDだけを受け入れる。absolute path、UNC、device path、drive-relative path、`..`、NULを拒否し、Windows case／separatorを正規化する。symlink、junction、reparse point解決後のreal pathが`G:\workspace`外なら拒否する。read直前にstat、file type、size、containmentを再検証する。

## 6. Runtime data flow

```text
Windows user logon
  -> Task Scheduler delay 30s
  -> exact tunnel-client.exe
  -> run --profile g-workspace-readonly
  -> User-scope CONTROL_PLANE_API_KEY reference
  -> outbound HTTPS poll to OpenAI
  -> Profile launches exact .venv Python stdio MCP server
  -> exact 4 read-only Tools
  -> resolved target under G:\workspace
  -> bounded Tool result through the same Tunnel
```

OpenAI product association、ChatGPT workspace permission、app selection、Task authorizationとTool call authorizationは、Local Processがreadyであることとは別の境界として維持する。Local readyだけでChatGPT側のTool availabilityまたはTaskごとのfile read許可を推測しない。

## 7. Tunnel Profile migration

Migrationは次の順序で一度だけ行う。

1. 現Profileの存在、ACL、sanitized configuration、`doctor`結果を確認する。
2. 独立MCP serverをProfileへ接続せず構築し、unit／stdio integration testを通す。
3. 現ProfileをTask-specific backupへcopyし、secretを表示しない。
4. exact旧Tunnel Processが0件であることとhealth port owner不在を確認する。
5. ProfileのMCP commandだけを独立runtimeへ置換し、Tunnel identity、API key reference、association、health addressを維持する。
6. `tunnel-client doctor --profile g-workspace-readonly --explain`をsanitized outputで実行する。
7. Profile migration後のdiffをsecret非表示で検証する。
8. Tunnel／MCP／Task acceptance後にのみbackupのretention dispositionを決める。失敗時はexact backupからProfileだけをrecoverできるようにする。

旧Skill path、fallback Profile、旧Tool aliasまたはdual runtimeは残さない。

## 8. Scheduled Task design

Taskは次のcontractへ固定する。

```yaml
task_name: OpenAI Secure MCP Tunnel - G Workspace Readonly
principal: current exact Windows user
run_level: least privilege
logon_type: interactive token
trigger: current user logon
delay: 30 seconds
action: exact tunnel-client.exe run --profile g-workspace-readonly
working_directory: tunnel-client binary directory
multiple_instances: ignore new
restart_interval: 1 minute
restart_count: 3
execution_time_limit: unlimited
run_only_if_network_available: true
start_when_available: true
hidden: false
```

- 管理者、SYSTEM、LocalServiceまたは別userとして実行しない。
- password、API key、Tunnel ID、Profile bodyをTask definitionへ保存しない。
- `CONTROL_PLANE_API_KEY`は既存User scope referenceだけを使用し、Machine scopeへ複製しない。
- Task definitionはoperatorから見える状態を維持し、対話UIまたは可視consoleを要求しないbackground actionとして登録する。
- 既存Process、port conflictまたはunhealthy stateを自動killしない。
- Taskを再登録する場合はcurrent definitionをsanitized exportし、意図したFieldだけを変更する。
- Task停止、無効化、削除は明示的なoperator actionとし、MCP package updateへ暗黙結合しない。

## 9. Performance and security boundary

### 9.1 Performance

待機中は`tunnel-client`のoutbound long-poll、Python MCP stdio Process、Tunnelが行うLocal Codex検出の子Processが常駐要素になり得る。login contentionを避けるため30秒delayを使用し、Server extractorは必要なTool callまで大型parserをlazy importする。

実装後に同一PCで次を実測する。

- login triggerから`readyz=200`までの時間。
- ready後10分間のTunnel全Process treeのCPU、working set、Process I/O transfer count、system network adapter bytes。
- representative text `fetch`一回と最大許容File一回の追加負荷。
- network断／復帰時のProcess数、retry、CPU、log growth。

Local operational guardrailは、10分idle平均CPU 1%未満、Tunnel全Process treeのcombined working set 256 MiB未満、Process identity／count安定、持続的restart／reconnect loopなしとする。これはMiraikanai local decisionであり、OpenAI、MCP、Pythonまたはuvの公式保証値ではない。超過時はTaskを無効化してon-demand起動へ戻し、原因を調査する。

[Microsoft `GetProcessIoCounters`](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-getprocessiocounters)は指定Processの「すべてのI/O操作」のaccountingを返し、`WriteTransferCount`をfilesystem writeへ限定しない。そのためProcess I/O transferとsystem network adapter aggregateは性能の参考値とし、disk／workspace writeの証拠またはguardrailには使用しない。allowed root全体のbefore／after snapshotも他Processの変更を含み帰属できないため参考値に限定する。read-only性はTool catalog、write handler不在、path containment、source reviewとstdio integration testで別に検証する。

### 9.2 Security

主要riskとmitigationは次である。

| Risk | Mitigation |
| --- | --- |
| read-only contentの外部送信 | exact app／workspace association、Task authorization、4 Tool限定、root containment、metadata／digest確認 |
| User-scope API key露出 | Task argument／XML／logへ含めない、scoped runtime key、rotation、User ACL |
| login persistence対象の改ざん | absolute path、User ACL、binary version／hash、Profile sanitized diff、lock fingerprint |
| root escape | real-path containment、reparse point拒否、read直前再検証 |
| write／shell capability | handlerとdependencyから除去し、catalog testでforbidden Tool 0件を確認 |
| same-user Process compromise | TaskはLimitedだがcurrent user tokenにはworkspace write権限が残る。read-onlyはOS ACL強制ではなくapplication boundaryであり、absolute path、locked dependencies、exact catalog、test、manual update gateでriskを低減する |
| restart storm | single instance、1分間隔、最大3回、既存Processを自動killしない |
| local admin UI exposure | `127.0.0.1`だけを維持し、remote bindを禁止 |
| stale dependency | auto-update禁止、明示的lock refresh、test／doctor／acceptance後だけpromotion |

`G:\workspace`内にTaskで送信してはならないsecret、credential、personal dataまたはlegal-restricted contentが存在する場合、read-onlyであってもallowed root全体を安全とはみなさない。Task authorizationと対象File Manifestで必要範囲を限定する。

## 10. Error handling

次は自動修復せずfail closedにする。

- runtime、Profile、User-scope API key referenceまたはlock欠落。
- Profile command、allowed root、Tool catalog、annotationまたはdependency fingerprint mismatch。
- Tunnel Process duplicate、unexpected executable、port 8080 foreign owner、health／ready timeout。
- path escape、unsupported／encrypted／oversized file、parser failure、result size超過。
- Scheduled Task principal、run level、action、trigger、restartまたはsecret boundary mismatch。

Error outputへAPI key、Tunnel ID、Organization／Workspace ID、Profile body、environment一覧またはroot外Pathを含めない。新規に開始したProcessだけが明確な場合を除き、Processをkillしない。

## 11. Verification strategy

### 11.1 MCP package

- exact 4 Tool catalogとannotation equality。
- forbidden write／delete／shell Tool 0件。
- path normalization、absolute／UNC／device／`..`／reparse escape拒否。
- deterministic directory listing、bounded search、stable ID、fetch byte／SHA-256一致。
- Text、Image、PDF、DOCX、PPTX、XLSX representative extraction。
- unsupported、encrypted、macro、oversized、scan-only、unknown binaryのtyped failure。
- official MCP Clientによるstdio initialize、tools/list、tools/call、unknown Tool failure。
- stdoutがMCP wireだけで、diagnosticはstderrへ限定されること。

### 11.2 Environment and Tunnel

- Python 3.14 interpreter、`uv lock --check`相当、`uv sync --locked`、package import。
- Profile command absolute equality、lock fingerprint、User ACL。
- sanitized `tunnel-client doctor`のFAIL 0件。
- `/healthz=200`、`/readyz=200`、expected Process exactly 1件。
- second startでProcessが増えないこと。
- Tunnel停止後のTask manual runで同じProfileがreadyになること。

### 11.3 Scheduled Task

- principalがcurrent exact user、least privilege、passwordless interactive tokenであること。
- exact Logon Trigger、30秒delay、single-instance、bounded restart、unlimited execution、network condition。
- Task definitionとhistoryへsecretがないこと。
- 登録直後のmanual startと次回実login後のstartを分離して報告する。実login acceptance前にlogon自動起動を実証済みと呼ばない。

### 11.4 Performance

- 10分idle、representative fetch、network断／復帰の測定値を記録する。
- CPU／working set／Process安定性guardrail超過、restart loopまたはunexpected log growthがあればTaskを有効化済みのまま残さない。
- Process I/O transfer、system network bytes、allowed root全体の変化は帰属不能なaggregateとして記録し、disk／workspace writeの証拠へ読み替えない。

## 12. Change boundary

実装対象:

- `%LOCALAPPDATA%\MCP\g-workspace-readonly`の独立runtime、tests、lock、venv。
- `%APPDATA%\tunnel-client\g-workspace-readonly.yaml`のMCP command migration。
- current userのScheduled Task一件。
- Local verification／performance evidence。

対象外:

- Personal Skillの復元または新規Skill作成。
- Codex plugin install、Codex config、Repository Engine code、Architecture Owner、CI。
- Windows Service、SYSTEM account、admin elevation、Machine scope API key。
- `tunnel-client`、Python、uvまたはdependencyの自動update。
- public plugin submission、public HTTPS endpointまたはFirewall inbound rule。
- Profile identity、Tunnel association、allowed rootの無断変更。

## 13. Design acceptance

1. Runtimeが廃止済みSkillおよびCodex lifecycleから独立する。
2. Python 3.14 series、uv lock、専用venv、absolute runtime commandが一意である。
3. Browser側へ公開可能なToolがexact 4 read-only Toolsで、write／shell capabilityが0件である。
4. real path `G:\workspace`外へのreadが拒否される。
5. Profile identity、User-scope API key reference、association、health addressを維持してMCP commandだけを移行する。
6. Tunnelがcurrent user login 30秒後にleast privilegeで起動し、多重起動または無限retryしない。
7. API key、Tunnel identity、Profile bodyをTask、command、log、Tool result、Repositoryへ保存しない。
8. `doctor`、health、ready、stdio Tool、Task manual run、performance guardrailをLocal Evidenceで検証する。
9. 実login後のTrigger acceptanceが未実施なら、その一点をremaining verificationとして明示する。
10. Local runtimeまたはTaskの存在をEngine実装、Architecture materialization、Product qualificationまたはreleaseへ読み替えない。

## 14. 2026-08-04 Local materialization evidence

本節は個人Windows環境の非規範Local evidenceであり、Repository Engine実装、Architecture materialization、Product qualification、ChatGPT workspace authorizationまたはreleaseを表さない。

- 独立runtimeを`%LOCALAPPDATA%\MCP\g-workspace-readonly`へmaterializeし、廃止済みPersonal Skillを復元せず、独立Git `a924e97`までcleanにした。
- 実行contractはPython 3.14.6、uv 0.12.1、MCP 2.0.0、Tunnel client 0.0.10、専用`.venv`、locked dependencyである。
- MCP testは83件PASS。公開catalogは`list_allowed_directories`、`list_directory`、`search`、`fetch`のexact 4 Tools、forbidden Tool 0件、allowed rootはreal path `G:\workspace`一件である。
- Profileはprivate current-user-only backupを保持したままMCP command一行だけを独立runtimeへ変更した。旧Skill参照0件、新command 1件、その他content一致を確認した。
- 公式`doctor --explain --json`は12 PASS、3 SKIP、FAIL 0。on-demand起動とTask manual startの両方で`/healthz=200`、`/readyz=200`、Tunnel executable 1件を確認し、second startはsteady-state Processを増やさず終了した。
- Scheduled Task `OpenAI Secure MCP Tunnel - G Workspace Readonly`はcurrent user、Interactive、Limited、login delay 30秒、IgnoreNew、restart 1分／3回、network required、enabled、password／secret保存なしで登録した。manual startは合格し、Taskは現在enabledである。
- 訂正版10分idle測定は638.867秒／120 samples、7 Processes、平均CPU 0.0021%、最大combined working set 37,928,960 bytes（約36.2 MiB）、Process count stableでLocal guardrailにPASSした。
- 同測定のI/O transferはread 1,247,895 bytes、write 1,647,568 bytes、system network adapter aggregateは943,287,769 bytes、allowed rootのunattributed changesは265件だった。これらはTunnel専用disk／network／workspace writeへ帰属できない参考値である。
- Local Codex検出により`codex app-server`系を含む7 Process treeになったが、Codex pluginまたはSkillはinstallしていない。read-only Python Processの60秒Process別観測ではwrite transfer 0で、長時間増分はLocal Codex検出のstdio／JSON-RPC側へ集中した。
- representative stdio integrationは合格した。ユーザーのnetwork adapterを停止するnetwork断／復帰試験は安全なisolated pathがないため未実施である。
- 次回の実PC loginで、loginから30秒以後の起動、instance 1件、health／ready、restart stormなしを確認するまでは、real-logon trigger acceptanceを`pending-next-login`とする。Profile backupもそのacceptanceまでprivate retentionする。
