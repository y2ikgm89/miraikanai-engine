# Miraikanai Engine 計画書レビュー Closure 設計

- 状態: ユーザー承認済み
- 承認日: 2026-07-22
- 選択方式: 依存順 closure 型
- 入力:
  - `docs/reviews/2026-07-22-plan-review.md`
  - `docs/reviews/2026-07-22-plan-review-findings.md`
  - `docs/superpowers/specs/2026-07-22-plan-closure-spec.md`

## 1. Outcome

レビュー結果を監査可能な closure 台帳へ正規化し、相反・重複する提案を一つの設計判断へ統合する。Product Registry修正をControl Planeの入力として固定し、Control Plane baseline、Runtime ECS、D3D12、最終Product Qualification再検査の順に正本と実装計画を閉じる。

完了時には次を満たす。

1. 非棄却指摘253件について、一意な Finding ID、検証状態、severity、重複関係、処置、変更先、検証Evidenceを追跡できる。
2. 詳細所見に明細が存在しないREFUTED 4件を、存在する指摘として捏造しない。レビュー総括と詳細所見は「明細を保持する253件」と「レビュー過程で棄却されたが明細未収録の4件」を区別する。
3. `UNVERIFIED` 43件はローカル正本で再検証し、`confirmed`または`refuted`へ遷移する。high 10件を先に処理する。
4. 要決定46件の重複と相反を統合し、各canonical decisionを一度だけ適用する。
5. Product Phase、Requirement、Fixture、Work Package、Capability、Target、Owner文書の参照が全件解決する。
6. Control Plane、Runtime ECS E0、D3D12実装計画が、承認済みbaselineと実在するOwnerだけに依存する。
7. リンク、ID文法、Registry参照、件数、Markdown構造の機械検査が成功する。

## 2. Authority と優先順位

判断根拠は次の順序で適用する。

1. ユーザーが承認した設計とDecision。
2. active architecture specの明示的なOwner規則。
3. `2026-07-22-plan-closure-spec.md`の正本・Registry先行方針。
4. 外部API、SDK、Toolchain、Platformについての公式一次資料。
5. レビュー文書の提案。

外部公式資料は外部仕様の事実と制約だけを所有する。Miraikanai固有のID、Owner、Phase、Capability、承認状態を「公式推奨」と表現しない。公式資料がプロジェクト固有判断を規定しない場合は、既存正本との一貫性、移行量、fail-closed性を根拠として判断を記録する。

## 3. 採用方式

### 3.1 選択した方式

依存順 closure 型を採用する。

1. レビュー台帳を正規化する。
2. ID、Inventory、Ownerの基盤判断を閉じる。
3. Product RegistryのPhase／Requirement／Fixture結線をControl Plane入力として閉じる。
4. 欠落Owner責務を既存または計画済み正本へ割り当てる。
5. Control Plane baselineを閉じる。
6. Runtime ECSとD3D12の設計・実装計画を新baselineへ追随させる。
7. Product RegistryとQualificationを再検査し、全横断検査を実行する。

### 3.2 採用しない方式

- 要決定46件の一括適用は、Target profile IDとDiagnostic IDに相反案があるため採用しない。
- CONFIRMEDだけを局所修正する方式は、Control Planeの開始条件となるIDとOwner判断が閉じないため採用しない。
- レビュー文書の件数だけを修正する方式は、実際の正本参照切れを残すため採用しない。

## 4. Closure 台帳

新しいclosure台帳は一行一canonical findingとし、次のFieldを必須にする。

| Field | 規則 |
|---|---|
| `finding_id` | `review.plan_review_2026_07_22.finding_NNN`。各segmentは英字で開始し、欠番を再利用しない |
| `source_locations` | 詳細所見のfile headingと行、重複元を配列で保持 |
| `validation_state` | `confirmed \| refuted`。移行中だけ`unverified`を許可 |
| `severity` | `high \| medium \| low` |
| `category` | `gap \| contradiction \| ambiguity \| link_or_id_error \| infeasibility \| improvement \| tech_stack` |
| `disposition` | `applied \| decision_applied \| deferred \| refuted` |
| `decision_id` | 設計判断が必要な場合のcanonical decision ID。それ以外は`null` |
| `changed_documents` | 実際に変更した文書path。変更なしは空配列 |
| `evidence` | 検索、検査command、公式一次資料、または正本sectionへの参照 |

元の詳細所見は監査証跡として保持する。表現の整理以外に、過去の指摘本文をclosure後の結論へ書き換えない。

## 5. Canonical design decisions

### 5.1 Diagnostic identity

- 機械参照用Diagnostic IDは`diagnostic.<domain>.<condition>`とする。
- `MirakanDiagnosticV1.code`は`MIRAKAN-<DOMAIN>-<CONDITION>`とする。
- IDとcodeの対応は一対一で、domain／condition tokenのcanonical変換表を`naming-project-layout.md`が所有する。
- 裸のPascalCase diagnostic、backend固有の旧code、同義aliasを禁止する。

これにより要決定17、30、39を一つのdecisionへ統合する。

### 5.2 Target と Toolchain profile identity

- logical Target IDは`target.windows.desktop`、`target.windows.editor`、`target.android.mobile`、`target.apple.mobile`、`target.headless.host`を正本とする。
- `toolchain_lock.profiles[].profile_id`も対応するlogical Target IDを使用する。
- schema／profile versionは`profile_version`へ分離し、logical IDへ`v1`を埋め込まない。
- 既存lockからの移行時はtoolchain lock schema majorを更新し、旧profile IDから新Target IDへのexact migration表を要求する。
- 旧`windows_desktop_v1`、`android_mobile_v1`、`apple_mobile_v1`はmigration表だけに残す。

これにより要決定14、24、38を一つのdecisionへ統合する。

### 5.3 Kind別ID grammar

単一の一般regexを全識別子へ適用しない。`naming-project-layout.md`がkind別closed grammarを所有する。

- JSON／schema Field名とMCD `namespace_path` segmentはsnake_caseを使用する。
- `operation.*`、`diagnostic.*`、`target.*`、`capability.*`はdotでnamespaceを分離する。
- `phase.*`、`wp.*`、`fixture.*`、`requirement.*`、`fallback.*`など既存Registry logical IDは、各dot segment内のlowercase kebab-caseを許可する。
- 一つのsegment内でsnake_caseとkebab-caseを混在させない。
- maturity、schema major、profile versionをlogical identityへ埋め込まない。
- 数字だけ、または数字で始まるsegmentを禁止し、`2d`／`3d`は`general_2d`／`general_3d`など意味を保つ語順へ移行する。

### 5.4 Inventory count

- 正本件数とrelation pair件数をDecision本文へ固定値として重複保持しない。
- 生成Inventoryの件数と実在file集合の一致をGateとする。
- 歴史的Decisionの旧件数は改竄せず、生成Inventoryを現行authorityとする追補を追加する。
- レビュー対象Markdown数、active spec数、変更file数を別Fieldとして数える。

## 6. Product Registry closure

次を同一ChangeSetで行う。

1. Phase 1用`manual_e2e` RequirementとPhase 4用AI authoring Requirementを分離する。
2. Phase 4 fixtureからAI authoring経路を検証可能にする。
3. Phase 6へWindows限定の3D First Playable Requirementを追加し、C2 3D coverageをPhase 8専用にする。
4. MVP完了chainをRequirementとfixtureへ結線する。
5. Project Native ModuleとProject Shader activationをCapability、Requirement、Work Packageへ登録する。
6. Capability Registryの追加規則を明文化し、Subsystem正本が未登録Capabilityを参照できないようにする。
7. Phase、Work Package、Requirement、Fixtureの双方向参照とTarget包含をlintする。

## 7. Owner closure

Control Planeで計画済みの次の5正本を先行させる。

1. `architecture-governance.md`
2. `compatibility-evolution.md`
3. `persistence-save.md`
4. `runtime-package.md`
5. `application-package-release.md`

追加責務は新規正本を増やす前に既存Ownerへ割り当てる。

- CI runner、GPU host、macOS build host、mobile device matrixは`toolchain-dependencies.md`が実行基盤profileと調達状態を所有し、Verification文書はlaneとEvidenceだけを所有する。
- First Playable asset調達方針は`asset-lifecycle.md`がprovenanceと取得経路を、`domain-pack-contract.md`が同梱reference asset setを所有する。
- Support bundle schemaは`debugging-observability-replay.md`が所有し、Platform文書は収集可能Fieldとprivacy制約だけを所有する。
- 開発体制、期間、critical path、scope削減候補、project risk registryは`product-plan.md`が所有する。

Owner文書が未作成または未承認の間、そのOwnerを要求するWork Packageは既存の`declared`状態を維持し、開始Gateを`diagnostic.architecture.owner-unapproved`で拒否する。新しいWork Package stateは追加しない。

## 8. Downstream closure

### 8.1 Control Plane

- kind別ID grammarをschemaとnegative fixtureへ反映する。
- 43 active spec本文のidentity migration Taskを維持する。
- document relation registryとschemaの生成Taskを明示する。
- continuation payloadのrepository-owned signing key案を廃止し、request hash、Source closure hash、revision、scopeのSHA-256 bindingへ置換する。

### 8.2 Runtime ECS

- ECS E0は5新Owner正本と生成baselineが承認されるまで開始しない。
- in-memory handle型のversion suffix省略はkind別命名規則へ登録する。
- chunk layout、partition policy、execution context、generation関係をE0 schema freeze前に閉じる。
- C1 GateはFirst Playableに必要な容量と耐久へ限定し、100万Entity／2時間enduranceは後段Qualificationへ分離する。

### 8.3 D3D12

- D3D12 QualificationをPhase 2で実在するempty-scene fixtureとbackend contract fixtureに限定する。
- 2D／3D content fixtureをPhase 2完了条件から外し、後続Product Qualificationへ接続する。
- compute queue barrier、layout compatibility、既存正本更新Taskを設計と実装計画で一致させる。
- 外部API制約はMicrosoft Learn、DirectX-Specs、Agility SDK、D3D12MAの公式一次資料で再確認する。

## 9. Verification

変更後に少なくとも次を実行する。

1. Markdown file／heading／table構造検査。
2. 相対linkとanchorの全件検査。
3. 文書ID、Target、Requirement、Fixture、Fallback、Phase、Work Package、Capability、Owner参照の全件解決。
4. kind別ID grammarのpositive／negative fixture。
5. Phase outcome Requirementがexit fixtureで検証されることのcoverage検査。
6. Work Package Targetがfixture Targetと矛盾しないことの検査。
7. 旧IDがmigration表と歴史的Decision以外に残らないことの検査。
8. closure台帳の全Findingが一つの終端`disposition`を持つことの検査。
9. `git diff --check`と最終差分レビュー。

検査不能な外部事実は外部依存検証表のverdictを`unverifiable`として残し、closure台帳の`validation_state=confirmed`の根拠には使用しない。公式資料とローカル正本が異なる場合、外部API制約は公式資料へ追随し、Miraikanai固有設計の変更はDecisionとして記録する。

## 10. Non-goals

- Engine runtime codeの実装。
- 未承認Capabilityのactivation。
- ECS、D3D12、Control Planeの実装完了宣言。
- multi-provider AI abstractionの先行実装。
- 公式資料が規定しないMiraikanai固有判断を、外部の公式推奨として正当化すること。

## 11. Change order

1. Closure台帳とレビュー文書の件数・用語を整合させる。
2. ID、Inventory、Diagnostic、Target profileのcanonical decisionを正本へ適用する。
3. Product RegistryのPhase／Requirement／Fixture修正をControl Planeのnormative入力として固定する。
4. 5新Owner正本と既存Ownerへの責務割当を閉じる。
5. Control Plane設計・実装計画を更新する。
6. 43＋5文書のmigration manifestを適用し、検査済みbaselineを生成する。
7. Runtime ECS Decision・実装計画を更新する。
8. D3D12設計・実装計画を更新する。
9. Product RegistryとQualificationを再検査し、横断検査結果をclosure Evidenceへ記録する。

この順序を逆転しない。上流baseline、Owner、ID migrationが未解決の場合、下流計画の変更を完了扱いにしない。
