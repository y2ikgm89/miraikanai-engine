# Miraikanai Engine Architecture Governance

- 文書ID: mirakan.arch.architecture-governance
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Architecture文書の状態、根拠区分、Inventory、一意所有、規範依存、分割・統廃合、Architecture Decision Log
- 非正本範囲: Product capability、MCD／Operation、実装Task、実装順序、Domain Schema・固定値・runtime挙動、AIの認可・実行route
- 規範依存: none
- 関連文書: [Product Plan](../00-product/product-plan.md)、[AI Security／Approval](ai-security-approval.md)、[AI Verification／Provenance](ai-verification-provenance.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Compatibility／Evolution](../02-foundation/compatibility-evolution.md)、[Governance Migration Proposals](../appendices/governance-migration-proposals.md)
- 根拠区分: project-decision。ADR lifecycleは一次資料を確認済み
- 外部根拠確認日: 2026-07-27

## 1. 結論

Architecture文書は、現在採用する設計、検討中の設計、外部仕様、実装状態、検証結果を混同しない。

文書が存在することは、実装、Schema、Registry、Artifact、生成物、Qualification Receiptが存在することを意味しない。`review`文書に記載された型、固定値、hash、Registry、Operation、Fixtureは、対応するRepository artifactと検証結果が存在しない限り設計候補である。

Architectureの各主張は次のいずれかへ分類する。

1. `official-spec`: 外部の公式仕様、標準、またはVendor一次資料で確認した事実。
2. `project-decision`: Miraikanaiが選択する方式。外部仕様への適合と、選択の妥当性を分けて記録する。
3. `provisional`: Prototype、計測、利用者検証、または承認が未完了の候補値・候補方式。
4. `measured`: 再現可能な条件、入力、Toolchain、Target、結果を持つ計測済み判断。

「公式推奨」「公式方式」は、Miraikanai内部の採用判断を外部組織の推奨と誤読できるため使用しない。Miraikanaiの判断は「本プロジェクトの採用方式」、外部資料の内容は「公式仕様で確認済み」と記載する。

## 2. 文書状態

### 2.1 Owner文書のHeader

`docs/architecture/00-product`から`08-packs`までのOwner文書は、次のHeader fieldを同じ順序で持つ。

```text
- 文書ID: immutable ASCII ID
- 文書状態: draft | review | accepted | deprecated
- 実装状態: absent | partial | implemented
- 検証状態: unreviewed | design-reviewed | prototype-verified | measurement-verified
- 正本範囲: この文書だけが決定する設計範囲
- 非正本範囲: この文書が決定しない範囲
- 規範依存: この文書の成立に必要なOwner文書。DAGでなければならない
- 関連文書: 読解・連携に必要だが成立条件ではない文書。循環可能
- 根拠区分: 本文で使用する根拠分類と例外
- 外部根拠確認日: YYYY-MM-DD | none
```

状態の意味は次のとおりである。

| Field | Value | 意味 |
|---|---|---|
| 文書状態 | `draft` | 構造化前の草案。正本として参照しない |
| 文書状態 | `review` | 設計審査中。採用・実装済みとは解釈しない |
| 文書状態 | `accepted` | Project判断として承認済み。実装状態とは独立 |
| 文書状態 | `deprecated` | current設計ではない。後継または削除理由を示す |
| 実装状態 | `absent` | 対応する実装・生成物をRepositoryで確認できない |
| 実装状態 | `partial` | 一部実装が存在する。適用範囲を本文で列挙する |
| 実装状態 | `implemented` | 対応実装と公開境界が存在する。検証済みとは限らない |
| 検証状態 | `unreviewed` | 一次資料・内部整合性を未確認 |
| 検証状態 | `design-reviewed` | 文書と一次資料を確認したがPrototype／計測結果はない |
| 検証状態 | `prototype-verified` | 対象Targetの再現可能なPrototypeで成立を確認した |
| 検証状態 | `measurement-verified` | 定義済み環境と入力で閾値・予算を計測した |

`implemented`、`prototype-verified`、`measurement-verified`は、本文から到達できるRepository pathまたはimmutable Evidence referenceを必須とする。参照がない状態でこれらへ昇格しない。

### 2.2 ADRの状態

Architecture Decision RecordはOwner文書と異なるlifecycleを持ち、`review | normative | rejected | superseded`を使用する。

- `review`は検討中であり、本文を修正できる。
- `normative`と`rejected`の本文は履歴として不変にする。
- 判断変更時は新しいADRを作成し、旧ADRへ`Superseded by`関係だけを追加する。
- 現在のSchema、固定値、runtime挙動はADRではなくOwner文書が所有する。

この方針は、`normative`（外部guidanceにおけるAccepted）ADRを後から現在の説明へ書き換えず、新しいADRで置換関係を記録するAWSおよびMicrosoftのguidanceと整合する。

## 3. 根拠と断定の規則

### 3.1 Inline根拠

次の主張は、段落、表、または節の冒頭で根拠区分を明示する。

- 外部APIのsupport、制約、default、minimum、deprecation。
- OS、SDK、Compiler、Tool、Libraryのversionと互換条件。
- memory、frame time、queue、timeout、UI timing、容量、件数などの数値。
- 「必須」「禁止」「唯一」「決定論的」「互換」とする設計判断。
- Security、Accessibility、Store、Distributionに関する合否条件。

記載形式は次のいずれかとする。

```text
根拠: official-spec — <一次資料へのlinkと対象version>
根拠: project-decision — <選択理由またはADR>
根拠: provisional — <未検証事項と確定条件>
根拠: measured — <環境、入力、結果、Evidence reference>
```

`review`文書にある未tagの固定値、hash、Registry内容、Fixture件数は`provisional`として扱う。数値の表記がexactでも、実測済みとは解釈しない。

### 3.2 外部根拠確認日

`外部根拠確認日`は、外部資料を最後に確認した日であり、文書全体の正しさ、実装、互換性を保証しない。

- 外部資料を使用しない文書は`none`とする。
- `latest`だけへ依存せず、可能なら対象versionを固定する。
- 公式資料とProject判断を同じ文章で混ぜない。
- Blog、比較記事、検索結果は選択肢の発見に利用できるが、仕様判断は公式一次資料で再確認する。

### 3.3 Hashと生成物

外部配布Artifactのhashは、取得URL、version、byte size、hash algorithmと組で記録できる。

内部Schema、Registry、Fixture、Receipt、Inventoryのhashは、対応ArtifactがRepositoryまたは承認済みimmutable storageに存在する場合だけcurrent値として記載できる。未生成Artifactのhash例は`example`または`provisional`と明示し、current root、lock、baselineと呼ばない。

## 4. 規範依存と関連文書

`規範依存`は、この文書の設計を解釈・検証するために必須となるOwner文書だけを列挙する。

- 規範依存graphはDAGでなければならない。
- Product、Governance、Foundation、Authoring／Runtime、Simulation／Rendering、Platform、Packの順序を逆向きに依存させない。
- 下位層から上位層への説明link、相互運用link、具体例は`関連文書`へ置く。
- 本文中の局所的な正本参照は許可するが、Headerの規範依存と矛盾させない。
- `mirakan.arch.<document-id>#<fragment>`形式の型付き文書参照は、参照先に一意に存在するMarkdown heading slugまたは明示的なASCII `<a id="..."></a>`へ解決しなければならない。Fragmentは大文字小文字を区別するimmutable identifierとして扱い、未定義の型名、表示見出し、重複anchorを参照値へ使わない。

循環が必要に見える場合は、共有契約のOwnerが不明確か、規範依存と関連参照を混同している。循環を正当化するのではなく、共通Ownerへの移管または参照分類の修正を行う。

## 5. InventoryとIndex

### 5.1 現在の状態

2026-07-27時点で、`ArchitectureInventoryV1`を生成するTool、Schema、immutable Inventory artifactはRepositoryに存在しない。

したがって、[Architecture Index](../README.md)は手動管理のnavigationであり、生成済みprojectionではない。Indexは文書件数、Owner、状態、依存の正本ではない。現存ファイルと各Headerがcurrent review対象である。

### 5.2 将来のInventory

Inventoryを導入する場合は、少なくとも次を実ファイルから決定論的に生成する。

```text
ArchitectureDocumentRecord
  document_id
  canonical_path
  document_status
  implementation_status
  verification_status
  normative_dependencies[]
  related_documents[]
  source_content_hash
```

Generatorが存在しない間、README、Markdown表、手入力hashをInventoryとして扱わない。「生成済み」「materialized」「exact projection」という表現も使用しない。

## 6. 一意所有

一つの型、識別子、固定値、Gate、状態遷移、Algorithm、Diagnosticには、正本Ownerを一件だけ割り当てる。

- 他文書は正本へのlinkと利用条件だけを記載する。
- 完全なSchema blockを複写しない。
- 同じ説明を複数文書へ置く場合は、片方を非規範の要約と明示する。
- Generated projectionを使用する場合は、生成元と検証方法を示す。
- Owner移管は旧Ownerから定義を削除し、新Ownerへ移した同じ変更で全参照を更新する。

正本Ownerが未決定の場合は、暫定的な複数定義を作らず、`provisional`な未解決事項として一か所へ記録する。

## 7. 分割・統廃合

### 7.1 分割基準

Owner文書は原則1,000行未満を目安とする。行数だけで分割せず、次の内容が混在した場合に分離する。

1. 安定したArchitecture原則と、未承認の候補設計。
2. 人が読む設計と、生成されるRegistry／Catalog。
3. Contract semanticsと、大量のFixture／Test matrix。
4. Domain共通規則と、Platform／Genre固有の具体例。
5. 現行仕様と、移行・履歴・検討記録。

分割後もOwnerを一意にする。補助文書はHeaderで`正本範囲`を限定し、親Ownerと同じ型を再定義しない。

### 7.2 統合基準

次の場合は文書または節を統合する。

- 同じ型・固定値・Gateを二か所で所有している。
- 一方の文書が他方の要約だけで独立した責務を持たない。
- 分離により相互参照が増え、単独では設計を理解できない。
- 実体のないredirect、legacy stub、mirror Schemaになっている。

`normative`／`rejected` ADRは統合・削除せず、Decision Logに履歴として残す。

### 7.3 Current split status

補助文書は大量の候補Registry、CatalogまたはFixtureを隔離するため、1,000行目安の例外になり得る。ただしHeaderで非正本性と親Ownerを明示し、安定原則を再定義してはならない。

| Document ID | 状態 | 分離対象または結果 |
|---|---|---|
| `mirakan.arch.product-plan` | 分離済み | execution registry proposalを補助文書へ分離 |
| `mirakan.arch.ai-security-approval` | 部分分離 | assumption guideとProvider／MCP supplementを分離。残るSecurity coreは次回追加前に再評価 |
| `mirakan.arch.editor-ui-framework` | 部分分離 | Design System／reference fixture catalogを分離。残るUI coreは次回追加前に再評価 |
| `mirakan.arch.editor-workspace-ux` | 分離済み | Panel／reference catalogを補助文書へ分離 |
| `mirakan.arch.runtime-performance-capacity` | 分離済み | 暫定scale catalogをproposalへ分離 |
| `mirakan.arch.simulation-physics` | 分離済み | AI intent／fixture catalogをproposalへ分離 |
| `mirakan.arch.pack-shooter` | 分離済み | Genre example／difficulty／fixtureをreference catalogへ分離 |
| `mirakan.arch.ai-verification-provenance` | 分離済み | Evidence lifecycle coreとEnvelope／Fixture candidate catalogを分離 |
| `mirakan.arch.executable-contracts` | 分離済み | MCD coreとOperation／未Activation planning candidate catalogを分離 |
| `mirakan.arch.gameplay-programming-model` | 分離済み | Programming Model coreとgenerated projection／Fixture candidate catalogを分離 |
| `mirakan.arch.project-state` | 分離済み | Project transaction coreとTarget readiness／Fixture candidate catalogを分離 |
| `mirakan.arch.rendering-world` | 分離済み | World source semantics coreとprocedural／Tilemap／Blockout candidate catalogを分離 |
| `mirakan.arch.pack-gameplay-features` | 分離済み | common Feature contractとFeature Definition／Fixture candidate catalogを分離 |

## 8. Architecture Decision Log

Decision Logは[decisions/README.md](../decisions/README.md)に置く。

- 判断がArchitecture上重要で、複数の妥当な選択肢がある場合にADRを作成する。
- ADRはContext、選択肢、Decision、理由、Consequences、Owner文書を記録する。
- 実装Task、担当、工数、作業順序、巨大Schema、Fixture一覧をADRへ含めない。
- `normative`または`rejected`になった本文を、現在の設計へ合わせて書き換えない。

## 9. Review checklist

Architecture文書の追加、更新、分割、統合では、次を確認する。

1. Headerの全fieldがあり、状態が実態と一致する。
2. 「公式」とProject判断を区別している。
3. 未実装の型・Registry・Artifactをcurrentまたは生成済みと呼んでいない。
4. 固定値に`official-spec | project-decision | provisional | measured`の根拠がある。
5. 規範依存がDAGで、関連文書と分離されている。
6. 型、固定値、Gate、Diagnosticの正本が一件である。
7. exact Owner／Capability／Target／Contract refが対応Registryまたは完全なrecordへ一意に解決し、ID出現、名前、prefixから未定義refを補完していない。
8. 内部Markdown link、型付き文書fragment、anchorが一意に解決する。
9. 外部linkが対象versionの一次資料へ到達する。
10. `normative`／`rejected` ADRの本文を変更していない。
11. 文書変更を実装完了、Capability activation、Qualification passとして表現していない。

## 10. 一次資料

- [AWS Prescriptive Guidance: Architectural decision record process](https://docs.aws.amazon.com/prescriptive-guidance/latest/architectural-decision-records/adr-process.html)
- [AWS Prescriptive Guidance: Best practices for ADRs](https://docs.aws.amazon.com/prescriptive-guidance/latest/architectural-decision-records/best-practices.html)
- [Microsoft Azure Well-Architected Framework: Maintain an architecture decision record](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record)
