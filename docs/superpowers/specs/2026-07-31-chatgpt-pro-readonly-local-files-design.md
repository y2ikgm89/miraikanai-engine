# ChatGPT Pro Read-only Local Files Design

- 設計状態: approved
- 対象: 個人Global Skill `collaborating-with-chatgpt-pro`、Secure MCP
  Tunnel Profile `g-workspace-readonly`、Browser版ChatGPT developer-mode app
  `G Workspace Readonly`
- 判断日: 2026-07-31
- 外部根拠確認日: 2026-07-31
- 先行設計:
  [ChatGPT Pro Tunnel-only Delivery Design](2026-07-31-chatgpt-pro-tunnel-only-delivery-design.md)

## 1. 結論

現在の`g-workspace-readonly` Profileが直接起動している汎用
`@modelcontextprotocol/server-filesystem`を廃止し、`G:\workspace`専用の
least-privilege read-only MCP serverへ置き換える。

新Serverが公開するToolは次の4個だけとする。

1. `list_allowed_directories`
2. `list_directory`
3. `search`
4. `fetch`

`write_file`、`edit_file`、`move_file`、`create_directory`、deprecated alias、
汎用shell、任意URL、任意Archive展開、任意Process起動は公開も実装もしない。
Tool annotationだけをsecurity boundaryにせず、書き込みhandlerと書き込み依存を
Server実装から除去する。

Browser添付、upload、promptへのLocal Artifact内容貼り付け、Project Source追加は
引き続き禁止する。Local File内容はTask Contractで許可された対象だけをSecure MCP
TunnelのTool resultとしてChatGPTへ返す。

## 2. 現状証拠と根本原因

2026-07-31のLocal Profile確認では、TunnelのMCP commandは次だった。

```text
cmd /c npx -y @modelcontextprotocol/server-filesystem "G:/workspace"
```

Browser版ChatGPTのapp管理画面では、このServerから14個のActionがimportされていた。
読み取りActionに加えて、少なくとも次の書き込みActionが存在した。

- `create_directory`
- `edit_file`
- `move_file`
- `write_file`

したがって、app display nameとdescriptionに`Readonly`と書かれていても、
実際のMCP Tool boundaryはread-onlyではなかった。

同じBrowser sessionの検証Chatでは、app pillと`Pro`は表示されていたが、
ChatGPTはToolを1回も呼ばず、`catalog_observable: false`、
`tools: []`として`blocked`を返した。一方、管理画面ではAction catalogが存在した。
これは次の2問題が独立していることを示す。

1. 汎用Filesystem Serverがwrite Toolまで公開している。
2. app管理画面のimport済みcatalogと、既存Chat sessionのTool利用状態が一致していない。

既存Skill validatorはMarkdown contractに`read-only`、`list`、`read`が書かれている
ことを検査していたが、Profileの実commandと実際の`tools/list`結果を検査して
いなかった。文書上の契約がruntime capabilityを証明していないことが根本原因である。

## 3. 外部仕様とProject decision

### 3.1 OpenAI公式として採用する事実

[Secure MCP Tunnel](https://developers.openai.com/api/docs/guides/secure-mcp-tunnels)
は、private MCP serverをpublic ingressなしでOpenAI Productへ接続し、
`tunnel-client`がMCP requestをLocal Serverへ転送する方式である。
App discoveryとTool callは、稼働中でconnectedなTunnel clientに依存する。

[Building MCP servers for plugins and API integrations](https://developers.openai.com/api/docs/mcp#create-an-mcp-server)
は、ChatGPT連携向けのread-only compatibilityとして`search`と`fetch`を示し、
Tool output schemaとstructured resultを定義するよう求めている。

[MCP and Connectors](https://developers.openai.com/api/docs/guides/tools-connectors-mcp)
は、MCP ServerのTool definitionがModelへimportされ、その中からToolが呼ばれる
こと、sensitive actionにはapprovalまたはallowlistを使うことを説明している。

公式資料は、この個人環境の`G:\workspace`、4 Tool構成、file type allowlist、
size limit、Skill prompt、特定ChatGPT Project、または後方互換性を規定しない。

### 3.2 この個人環境で採用する判断

- MCP transportは既存Secure MCP Tunnelを継続利用する。
- allowed rootは`G:\workspace`だけとする。
- ChatGPT compatibilityのため`search`と`fetch`を実装する。
- exact path auditのため`list_allowed_directories`と`list_directory`も実装する。
- Serverは公式MCP SDKを使った専用stdio Processとし、runtime dependencyを
  exact versionとlock fileで固定する。
- `npx -y`、floating latest、汎用Filesystem Server、proxy filter、
  ChatGPT側approvalだけによるread-only擬制は採用しない。
- 旧Tool名、write Tool、compatibility proxy、fallback Profileは残さない。

## 4. 方式比較

| 方式 | 利点 | 欠点 | 判定 |
| --- | --- | --- | --- |
| 専用read-only MCP Server | 実装とcatalogの両方からwriteを除去でき、file type処理と検証を所有できる | Serverとextractorの保守が必要 | 採用 |
| 汎用Filesystem MCPの前段proxy | 既存Tool実装を再利用できる | 内部write capabilityとfilter bypass riskが残る | 不採用 |
| ChatGPT permission／approvalだけで制御 | Local実装変更が小さい | write Toolが公開されたままで、approvalはread-only保証ではない | 不採用 |

## 5. Server architecture

### 5.1 配置

Personal Skill内に独立したServer packageを置く。

```text
collaborating-with-chatgpt-pro/
  mcp-server/
    pyproject.toml
    requirements.lock
    src/
      readonly_local_files/
        __init__.py
        server.py
        paths.py
        extractors.py
        models.py
    tests/
      test_catalog.py
      test_paths.py
      test_extractors.py
      test_stdio.py
```

専用Python virtual environmentをSkill packageのruntime directoryに作成する。
MCP SDKとextractor dependencyはexact versionとhash付きlockで固定する。
System-wide package、`pip install`のfloating constraint、Codex内部runtime、
Browser upload parserには依存しない。

Tunnel Profileは、専用virtual environmentのPythonと`server.py`をabsolute
pathで起動し、rootを`G:\workspace`へ固定する。API key、Tunnel ID、secretを
command lineへ追加しない。

### 5.2 Tool catalog

#### `list_allowed_directories`

- input: empty object
- output: exact root `G:\workspace`
- purpose: Browser側がactual allowed rootをTool resultで確認する

#### `list_directory`

- input: root-relative directory path
- output: sorted entry list with relative path、type、size
- recursion: disabled
- hidden entry: 通常Fileと同様に表示する
- purpose: exact target discoveryとmanifest parent coverage

#### `search`

- input: one query string
- output: OpenAI compatibility shapeの`results`
- result fields: stable relative-path ID、title、empty URL、metadata
- behavior: filenameとextractable textに対するbounded search
- purpose: ChatGPTの一般的なLocal document discovery

Local Fileはpublic URLを持たないため、架空のHTTP URLを生成しない。`url`は空文字とし、
ChatGPT citationではなくordinary Tool evidenceとして扱う。

#### `fetch`

- input: `search`結果またはTask Contractが指定したroot-relative ID
- output: OpenAI compatibility shapeの`id`、`title`、`text`、`url`、`metadata`
- metadata: source bytes、source SHA-256、detected MIME、extractor、extraction status
- additional content: allowlisted imageの場合だけMCP image content
- purpose: exact Local Artifact read

`search`と`fetch`は明示的なoutput schemaを持ち、同じ値を`structuredContent`と
JSON-encoded text contentで返す。Binary Artifactのmanifest照合は、`fetch`が
read直前のsource bytesから計算したSHA-256とsource byte countを使う。

全Toolに次のannotationを付与する。

```yaml
readOnlyHint: true
destructiveHint: false
idempotentHint: true
openWorldHint: false
```

AnnotationはUIとapproval分類用のhintであり、Serverの書き込み禁止を代替しない。

## 6. File type policy

### 6.1 直接Text

対象:

- UTF-8／UTF-8 BOM
- UTF-16 LE／BE BOM
- Markdown、source code、JSON、YAML、TOML、CSV、TSV、plain text

NULを含む、encodingを安全に判定できない、またはbinary signatureを持つFileは
Textとしてdecodeしない。

### 6.2 Image

対象:

- PNG
- JPEG
- WebP
- non-animated GIF

ExtensionだけでなくsignatureとMIMEを検証する。Resultはmetadata textとMCP image
contentを返す。SVGはactive content riskを避け、初期実装ではTextとしてもImage
としても返さず`unsupported-media-type`とする。

### 6.3 PDF

PDF parserでTextをLocal抽出し、page orderを保持して`fetch.text`へ返す。
暗号化PDF、parser error、Textを持たないscan-only PDFは`blocked`とする。
初期実装へOCRは入れない。

### 6.4 Modern Office

対象:

- DOCX
- PPTX
- XLSX

DOCXはparagraphとtable、PPTXはslide orderのText、XLSXはsheet orderのcell valueを
Local抽出する。Formulaは式とcached valueを区別してmetadataへ記録する。
external link、macro、embedded executable、OLE objectは実行も取得もせず、
content存在をmetadataで通知する。

旧DOC／PPT／XLS、macro-enabled DOCM／PPTM／XLSM、password-protected Fileは
初期実装では`unsupported-office-format`または`encrypted-file`として`blocked`
にする。LibreOffice automationやOffice COMは使用しない。

### 6.5 Limits

Default limitはServer定数として一元管理する。

- direct text source: 2 MiB
- image source: 10 MiB
- PDF／Office source: 50 MiB
- one Tool result extracted text: 400,000 Unicode characters
- directory listing: 2,000 entries
- search results: 100 entries

Limit超過時はsilent truncationせず、exact limitとobserved sizeを含むtyped errorを返す。
Skillはcomplete Artifact readを要求するTaskでこのerrorを`blocked`として扱う。

## 7. Path security

Tool inputはroot-relative pathまたはroot-relative stable IDだけを受け入れる。
Serverは各callで次を実行する。

1. separatorとcaseをWindows規則で正規化する。
2. absolute path、UNC、device path、drive-relative path、`..`を拒否する。
3. targetのreal pathを取得する。
4. case-insensitiveな`path.relative`相当判定で`G:\workspace`配下を確認する。
5. symlink、junction、reparse pointの解決後にroot外へ出るtargetを拒否する。
6. read直前にstatを取り直し、file typeとsizeを再検証する。

Server processにはwrite API、shell API、network client、または任意Archiveを展開する
APIを持たせない。DOCX／PPTX／XLSX parserはallowlisted package entryだけを
memory内で読み、embedded executableを展開しない。Temporary extractionが必要な
parserは専用temporary directoryだけを使い、`G:\workspace`へFileを作成しない。

## 8. Lifecycle and migration

### 8.1 Controlled breaking migration

1. 新Serverとtestを既存Profileへ接続せず構築する。
2. stdio integration testでexact catalogとrepresentative readを検証する。
3. 現行ProfileをTask-local temporary backupへ退避する。
4. exact旧Tunnel processを停止する。
5. ProfileのMCP commandを新Serverへ置き換える。
6. `tunnel-client doctor --profile g-workspace-readonly --explain`を実行する。
7. lifecycle helperでTunnelを起動し、health／ready／connectedを確認する。
8. Browser appの`更新する`を実行してcatalogを再importする。
9. 新規Project chatで受入試験を行う。
10. 受入後、temporary backupとtest fixtureを削除する。

旧Profile、旧Tool alias、fallback Processはruntimeに残さない。

### 8.2 Ongoing Skill lifecycle

`ensure_secure_mcp_tunnel.ps1`は従来どおりexact Processの再利用と停止時の自動起動を
所有する。これに加えてBrowser navigation前にLocal MCP preflightを行う。

Local MCP preflightは次を検査する。

- Profile commandが専用Serverだけを指す。
- dependency lock fingerprintが期待値と一致する。
- Server self-testのTool名がexact 4件である。
- 4 Toolすべてがread-only annotationを持つ。
- write／edit／move／create／delete／shell名を持つToolが0件である。
- allowed rootがexact `G:\workspace`である。

Mismatch時は既存Tunnelを自動killせず`blocked`とする。構成変更時のProcess replacementは
明示的なmigration workflowだけが行う。

## 9. Skill contract changes

`collaborating-with-chatgpt-pro`のcanonical capabilityを次へ変更する。

```yaml
required_tools:
  - list_allowed_directories
  - list_directory
  - search
  - fetch
write_tools_allowed: []
```

Artifact readのcanonical operationは`fetch`とする。既存の抽象`read`は
compatibility aliasとして残さない。既存の`required_tools: [list, read]`、
汎用Filesystem Tool名、Tool catalogを推測するprompt、catalog未観測でも進むbranchは
削除する。

Primary promptはLocal contentを含めず、Artifact ID、root-relative ID、size、digest、
expected Tool requirementだけを送る。ChatGPTは分析前にexact Tool callを行い、
Tool resultからallowed root、target、coverageを報告する。

## 10. Browser acceptance

受入試験はCodex in-app Browserだけで実施する。

1. Tunnelがhealth、ready、connectedである。
2. app管理画面のAction catalogがexact 4件である。
3. write／edit／move／create／delete Actionが0件である。
4. exact Project `AIネイティブC++ゲームエンジンプロジェクト`を開く。
5. memory mode `プロジェクトのみ`を確認する。
6. 新規chatを作成する。
7. `応答性能`でexact `Pro`を選択し、collapsed button `Pro`を確認する。
8. exact app `G Workspace Readonly`を選択する。
9. Browser添付、upload、paste、Project Sourceを使用しない。
10. `list_allowed_directories`、`list_directory`、`fetch`で既知Markdownを読む。
11. `search`で同Fileを発見し、stable IDが一致することを確認する。
12. Image、PDF、DOCX、PPTX、XLSXのrepresentative fixtureを各1件読む。
13. ChatGPTの回答内容をLocal expected valueと照合する。

Fixtureは専用temporary acceptance directoryに生成し、exact pathを確認してから試験後に
削除する。Repository canonical artifactまたは既存User Fileをfixtureへ転用しない。

既存chatはcatalog cacheを持ち得るため受入証拠に使わない。必ずapp更新後の新規chatを
使用する。

## 11. Error handling

次はtyped errorとして返し、Skillはfallbackせず`blocked`にする。

- `path-outside-root`
- `path-not-found`
- `path-not-file`
- `directory-not-found`
- `unsupported-media-type`
- `unsupported-office-format`
- `encrypted-file`
- `scan-only-pdf`
- `file-too-large`
- `result-too-large`
- `directory-too-large`
- `extractor-failed`
- `catalog-mismatch`
- `profile-mismatch`
- `tunnel-not-ready`

Error resultへsecret、Profile内容、API key、Tunnel ID、環境変数一覧、root外Pathを
含めない。

## 12. Test strategy

### 12.1 RED

実装前に次のfailureを固定する。

- 現行Profileが汎用Filesystem MCPを起動している。
- 現行catalogにwrite Toolが存在する。
- Skill validatorがruntime catalogを検証していない。
- Browser既存chatでrequired Tool callが0件である。

### 12.2 Unit

- exact 4 Tool catalog
- annotation equality
- absolute／UNC／device／drive-relative／`..`拒否
- symlink／junction root escape拒否
- root内case variation許可
- deterministic directory sort
- Text encoding
- Image signature
- PDF Text抽出
- DOCX paragraph／table抽出
- PPTX slide order抽出
- XLSX sheet／cell抽出
- encrypted／macro／legacy format拒否
- source size／result size／entry count limit
- error redaction

### 12.3 Integration

- official MCP clientからstdio Serverへ`initialize`
- `tools/list` exact equality
- 4 Toolのrepresentative `tools/call`
- unknown Toolとwrite Tool callのmethod-not-found
- Process stdoutがMCP protocolだけである
- Profile self-test
- Tunnel doctor、health、ready、connected

### 12.4 Browser

- app catalog exact equality
- new Project chat／project-only memory／Pro
- no upload
- expected Tool calls
- representative file type coverage
- response内容のLocal照合

## 13. 受入条件

- Browser appが公開するActionはexact 4件である。
- write／edit／move／create／delete／shell Actionが存在しない。
- allowed rootはreal path `G:\workspace`だけである。
- root escapeがunit testとintegration testで拒否される。
- Text、Image、PDF、DOCX、PPTX、XLSXのrepresentative fileを読める。
- unsupported、encrypted、macro、legacy、oversized fileがsilent fallbackせず
  typed errorになる。
- Tunnel停止時にSkill helperが固定Profileを起動する。
- Tunnel起動後にruntime catalog fingerprintを検証する。
- Browser版ChatGPTの新規Project chatでexact `Pro`とexact appを選択する。
- Browser添付、upload、paste、Project Source追加が0回である。
- ChatGPTがTool resultだけから既知Local File内容を正しく回答する。
- 既存汎用Filesystem MCP commandとwrite Tool compatibility layerが残らない。
- Skill test、Server test、Tunnel integration、Browser acceptance、
  Repository verificationがすべてpassする。
