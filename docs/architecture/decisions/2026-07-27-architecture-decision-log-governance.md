# Miraikanai Engine Architecture Decision Log Governance

- 文書ID: mirakan.decision.architecture-decision-log-governance
- 状態: review
- 正本範囲: Architecture Decision Recordの役割、Decision Logの配置、状態遷移、不変性、汎用形式、現行Architecture正本との責務分離、既存Decisionの整理方針
- 非正本範囲: 各DomainのSchema・固定値・runtime挙動、Architecture Inventory Schema、Owner Registry、MCD／Operation activation、実装Taskの順序。各Owner文書と後続ChangeSetを参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Architecture Document System Restructure](2026-07-21-document-system-restructure.md)
- 外部根拠検証日: 2026-07-27
- 文書種別: Architecture Decision／Decision Log governance
- Decision owner document: `mirakan.arch.architecture-governance`
- Decision日: 2026-07-27
- Supersedes: none

## 1. Context

Miraikanai Engineは、現行Architectureの正本とArchitecture Decisionを同じ`docs/architecture/`配下で管理している。既存Decisionには、採用理由と比較案だけでなく、Domain Ownerが所有すべきSchema、固定値、runtime挙動、移行手順、検証詳細が含まれるものがある。その結果、DecisionとOwner文書のどちらを変更すべきかが曖昧になり、二重正本、参照Graphの肥大化、Accepted後の履歴改変を招く可能性がある。

一方、DecisionをOwner文書へ全文統合して削除すると、なぜその選択をしたか、どの案を退けたか、当時の制約が何であったかをGit履歴の探索なしに発見できなくなる。現行仕様と判断履歴は、同じ情報を二重所有せず、別の責務として共存させる必要がある。

## 2. Decision drivers

1. 現行Schema、固定値、Gate、runtime挙動のOwnerを一件に保つ。
2. 判断のContext、比較案、理由、Consequencesを将来のReviewで発見できるようにする。
3. AcceptedまたはRejectedになったDecision本文を後から現在の説明へ書き換えない。
4. 判断変更時に、旧判断と新判断の関係を明示する。
5. DecisionをコードとArchitecture文書に近い同一Git repositoryで管理する。
6. Decisionを簡潔で一貫した形式にし、設計ガイドや実装計画へ膨張させない。

## 3. Considered options

### 3.1 DecisionをOwner文書へ統合して削除する

不採用とする。現行仕様の一意Ownerは明確になるが、判断理由、比較案、当時の制約、置換関係がGit履歴だけに退避される。Accepted DecisionをDecision Logから除去するため、append-onlyな判断履歴にならない。

### 3.2 全Decisionを単一の`DECISIONS.md`へ統合する

不採用とする。一覧性は上がるが、独立した判断のOwner、Review、Status、Supersedes関係、変更履歴が一ファイルへ結合される。複数判断の同時編集と競合を増やし、Decision単位の不変性を保ちにくい。

### 3.3 汎用形式の個別ADRと現行Owner文書を分離する

採用する。各Decisionは一つの重要な判断を所有し、現行の技術仕様はDomain Owner文書だけが所有する。Decision Logは判断履歴を、Architecture Indexは現行仕様への入口を提供する。

## 4. Decision

### 4.1 配置と責務

`docs/architecture/decisions/`をArchitecture Decision Logの中央配置とする。Decisionは原則として一つの重要な判断を一ファイルで所有する。複数の独立した判断、段階、別Ownerの採否を一つの巨大Decisionへ結合しない。

`docs/architecture/decisions/README.md`は次だけを所有する。

- Decision Logの目的と読み方。
- 汎用Decision template。
- StatusとSupersedesの運用規則。
- 全Decisionへのnavigation。

同READMEは個別判断、Domain Schema、固定値、Gate、runtime挙動を再定義しない。

### 4.2 現行Architectureとの分離

DecisionはContext、Decision drivers、比較案、採用判断、理由、Consequences、置換関係を所有する。Domain Owner文書は現在有効なSchema、型、固定値、状態遷移、runtime挙動、Gate、Qualificationを所有する。

Decision内にOwner文書のSchema表、Field一覧、固定値表、実装手順を複写しない。判断の理解に必要な最小限の要約と相対Linkだけを置く。Owner文書はDecisionを判断理由のinformative referenceとして参照できるが、current Contractのauthority dependencyをDecisionだけへ解決しない。

### 4.3 Lifecycle

既存Headerとの互換を保ち、Decisionの状態を次のように扱う。

| `状態` | ADR lifecycle上の意味 | 本文変更 |
|---|---|---|
| `review` | Proposed | Review指摘を反映できる |
| `normative` | Accepted | 本文を不変にする |
| `rejected` | Rejected | 却下理由を含めて本文を不変にする |
| `superseded` | Superseded | 本文を不変にし、置換先だけを明示する |

AcceptedまたはRejected後に判断内容を変更しない。新しい知見により判断を変える場合は新しいDecisionを`review`で追加し、旧Decisionを`superseded`へ遷移させ、新旧双方からStableな文書IDと相対Linkで関係を辿れるようにする。旧Decisionの本文は当時のContextと判断を維持し、現在の仕様へ書き換えない。

Status、`Supersedes`、`Superseded by`の関係metadataだけは、置換を記録するためのLifecycle変更として更新できる。変更理由と新しい判断は新Decisionが所有する。

### 4.4 汎用Decision format

新規Decisionと、Accepted前に整理する既存Decisionは次の順序を基本とする。

1. Header metadata。
2. Context。
3. Decision drivers／constraints。
4. Considered options。
5. Decision。
6. Consequences。
7. Canonical Owner documents。
8. Supersedes／Superseded by。
9. Officialまたは一次資料。

Decisionは簡潔、断定的、対象判断に限定した記述とする。詳細な設計、Schema、実装手順、Task計画、Qualification fixtureはOwner文書または実装計画へ置き、DecisionからLinkする。

### 4.5 既存Decisionの扱い

| Decision | 現在の扱い | 整理方針 |
|---|---|---|
| `mirakan.decision.architecture-document-system-restructure` | Accepted相当 | 本文を変更・削除しない。旧形式のAccepted履歴として保持する |
| `mirakan.decision.runtime-ecs-contract` | Proposed相当 | 現行Schema・固定値をOwnerへ一意化し、採用理由とConsequencesへ縮約する |
| `mirakan.decision.architecture-decision-log-governance` | Proposed相当 | 本Decisionの承認とmigration完了を同じReview closureで確認してAccepted相当へ遷移する |

Proposed Decisionの整理はAccepted前のReview修正として行う。Accepted Decisionの形式を揃えるためだけの書換えや削除は行わない。

## 5. Consequences

### 5.1 Positive

- 現行仕様と判断履歴の変更理由を分離できる。
- Domain Schema、固定値、Gateの一意Ownerを維持できる。
- 過去の比較案と採用理由をGit履歴の探索なしに発見できる。
- 判断変更をSupersedes chainとして追跡できる。
- Decision ReviewをDomain仕様の全面Reviewから分離できる。

### 5.2 Costs and risks

- Decision LogとOwner文書の両方を更新する変更がある。
- Informative referenceとauthority dependencyの分類Reviewが必要になる。
- 既存Proposed Decisionから重複詳細を除く際、Owner文書への移植漏れが起こり得る。
- Accepted Decisionには旧形式や冗長な記述が残るが、履歴不変性を優先して許容する。

## 6. Migration design

1. [Architecture Governance](../01-governance/architecture-governance.md)へDecision Logの責務、Lifecycle、不変性、Owner文書との分離規則を追加する。
2. `docs/architecture/decisions/README.md`を追加し、汎用template、Status、Decision一覧を定義する。
3. Root [Architecture Index](../README.md)で現行仕様一覧とDecision Logを明示的に分離する。
4. Proposed相当のRuntime ECS Decisionを汎用形式へ整理し、Owner文書と重複するSchema、固定値、実装詳細を除く。
5. DecisionへのHeader dependencyをReviewし、current Contract authorityとして使っている箇所はDomain OwnerまたはGovernance Ownerへ付け替える。
6. 判断理由として必要なDecision Linkは本文のinformative referenceとして保持する。
7. Accepted相当の既存Decision本文を変更せず、全Link、Status、文書ID、正本範囲を検証する。

この順序は設計上のmigration closureであり、実装Taskの詳細手順やCapability activationを定義しない。

## 7. Acceptance criteria

1. `docs/architecture/decisions/README.md`から全Decisionへ到達できる。
2. Root Architecture Indexが現行仕様とDecision Logを別Sectionで示す。
3. DecisionのContext、options、decision、consequences、status、owner、置換関係を一貫して判定できる。
4. Accepted相当の既存Decision本文が変更されていない。
5. Proposed相当DecisionにDomain Schema表、固定値の正本、実装Task Planが残っていない。
6. Schema、固定値、Gate、runtime挙動が一つのDomain Owner文書へ解決する。
7. 全relative Markdown Linkとheading anchorが解決する。
8. 旧Decision pathのredirect、互換stub、単一巨大Decisionを作らない。
9. 作業開始前から存在した未コミット変更を保持する。

## 8. Official guidance

- [Microsoft Azure Well-Architected Framework: Maintain an architecture decision record](https://learn.microsoft.com/en-us/azure/well-architected/architect-role/architecture-decision-record): ADRをappend-only logとして扱い、Accepted recordを書き換えず、新しいrecordからSupersedeする。Decisionを簡潔にし、設計ガイド化しない。
- [AWS Prescriptive Guidance: Architectural decision record process](https://docs.aws.amazon.com/prescriptive-guidance/latest/architectural-decision-records/adr-process.html): Accepted／Rejected ADRをimmutableとし、変更時は新ADRを作り旧ADRをSupersededにする。
- [AWS Prescriptive Guidance: Best practices for using architectural decision records](https://docs.aws.amazon.com/prescriptive-guidance/latest/architectural-decision-records/best-practices.html): 旧ADRをDecision Logへ保持し、中央配置とOwnerを明示する。
- [Google Cloud Architecture Center: Architecture decision records overview](https://docs.cloud.google.com/architecture/architecture-decision-records): Options、requirements、decision、理由を記録し、コードに近いversion controlへ保持する。

## 9. Non-goals

- Accepted Decisionの履歴を現在の説明へ書き換えること。
- 全Decisionを一冊へ結合すること。
- Decisionを削除してGit履歴だけへ退避すること。
- DecisionからMCD、Operation、Tool、Capability、実装Taskをactivateすること。
- Decision Logだけを根拠にcurrent runtime、Contract、Owner Registryの状態を推測すること。
