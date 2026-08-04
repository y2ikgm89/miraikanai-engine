# C1 First Playable Architecture Closure Design

- 文書状態: approved design
- 正本性: non-normative design rationale
- 対象: Product-facing initial First Playableの到達可能Game、exact Target、required gameplay coverage、explicit exclusion、current evidence gap
- 決定: 既存Owner連鎖を最小変更し、approved RPG-first directionの欠落したboundaryをcanonical Ownerへ戻す
- 承認: 推奨案を2026-08-04にユーザー承認
- 実装状態: absent and unchanged
- 実装計画状態: intentionally absent
- Materialization状態: Target／Locale Profile、MCD、RPG／Feature contract、Project、Package、Fixture、Receiptはabsent
- 外部根拠確認日: 2026-08-04

## 1. Status

本設計はArchitecture文書の追記、改善、更新、修正だけを扱う。C++、Shader、Asset、Schema file、Registry instance、Fixture、Receipt、Build system、CI、実装Task、Work Package、順序、工程、工数、担当または日程を作成しない。

本書は採択理由とOwner routingだけを記録する非規範設計入力である。型、fixed value、required set、Product boundary、GateまたはEvidence meaningを定義せず、current contractはリンク先のArchitecture Ownerだけから読む。

## 2. Problem

[Document System Restructure §14.4](../../architecture/decisions/2026-07-21-document-system-restructure.md#144-rpg-first-product-mvp設計の節disposition)は、approved RPG-first Product MVPのplayable flow、gameplay coverage、Runtime cadence、explicit exclusionをProduct PlanへMergedすると記録した。しかしcurrent [Product Plan §5.1](../../architecture/00-product/product-plan.md#51-first-playable-outcome)はcompact 2D command RPGという要約と抽象的Capability集合だけを残し、次が一意でなかった。

- Build／Editor Host、runtime Target、Distribution route。
- TitleからResultまでのReference path。
- 五Reusable RPG Feature familyの全件requiredかsubsetか。
- party、battle command、status、Quest choice、Shop、Saveのminimum semantic coverage。
- Product First Playable C1とInputのcross-target C1／Windowsのcross-distribution C1の違い。
- required gapと、明示的non-goal／Futureの違い。
- Architecture targetと、現Repositoryで実際に作れるGameの違い。

## 3. Considered approaches

### A. Existing Owner chain correction

採用する。Product PlanがProduct boundary、RPGがGenre composition、Gameplay FeatureがFeature meaning、LifecycleがEvidence、Input／Windowsがdomain qualificationを所有する。Closure Reviewはcurrent evidence gapを非規範に追跡する。

### B. New standalone C1 Owner

却下する。First Playable Definition、RPG composition、Feature meaning、Platform qualificationの第二正本を作り、Product → domainのOwner方向を崩す。

### C. RPG Work Package／Fixture／execution Registryを作る

却下する。依頼範囲外の実装計画となり、未materialized IDやEvidenceをArchitecture completenessと混同させる。

## 4. Canonical Owner routing

### 4.1 Product boundary

[Product Plan §5.1](../../architecture/00-product/product-plan.md#51-first-playable-outcome)だけがinitial Product Definition、Game form、Host／runtime Target、Distribution、locale、Input、authoring lane、playable path、required coverageおよびexplicit exclusionを所有する。本書はその値を再定義せず、consumerは同節のexact Definitionからread-backする。

### 4.2 Required gameplay

[Gameplay Feature Packs §4.4](../../architecture/08-packs/gameplay-features.md#44-reusable-rpg-feature-family)だけがReusable RPG Feature family identityとinitial full-family setを所有し、[RPG Genre Pack §4.1](../../architecture/08-packs/rpg.md#41-initial-first-playable-composition)がGenre-facing compositionを供給する。[Product Plan §5.1.2](../../architecture/00-product/product-plan.md#512-playable-path-and-required-gameplay-coverage)はProductに必要なsemantic coverageを選ぶ。instance数、Asset数、dialogue数、test数はQualification／Fixture Ownerへ残し、本書からfixed countを生成しない。

### 4.3 Explicit exclusions

[Product Plan §5.1.3](../../architecture/00-product/product-plan.md#513-explicit-exclusions-and-completion-blockers)だけがinitial First Playableのexplicit exclusionとcompletion blockerを所有する。本書の採択理由からnon-goal、Future subjectまたはrequired setを追加・削除しない。

### 4.4 State boundary

Architecture boundaryを`closed-in-target-design`にできても、current Repositoryで作成・起動できるGameは0件である。Profile record、MCD、contract、implementation、Project、Asset、Package、Fixture、Receipt、Qualification、Activationは独立にabsentと記録する。

## 5. Official-source boundary

Microsoft GameInput、Windows package／distribution、Windows accessibility、WCAG 2.2、RFC 5646、CMake Presets、Unreal packaging、Godot exportの公式一次資料を、Platform／format／accessibility／build・package surfaceの確認に限定して使う。compact RPGの内容、Feature選択、Windows-only C1、MSIX選択、clean-breakはMiraikanaiのproject-decisionであり、外部組織の公式推奨と表現しない。

## 6. Clean-break rule

initial V1はpublic materialization前であるため、predecessor、v0、Shooter→RPG rename、legacy alias、dual reader、migration、managed／portable fallbackを作らない。既存の非規範Execution ProposalをProduct First Playable evidenceへ昇格せず、RPG execution projectionは未作成のまま保持する。

## 7. Document changes

| Document | Change |
|---|---|
| Product Plan | exact Product boundary、playable path、required coverage、exclusion、current availabilityを正本化 |
| RPG Genre Pack | exact五family composition、generic dependency、battle command roleを接続 |
| Gameplay Feature Packs | 五familyのnon-substituting C1 coverageを明確化 |
| Product Lifecycle | exact Host／Target／locale／Input／MSIX Evidence read-backを接続 |
| Input | portable cross-target C1とProduct First Playable subsetを分離 |
| Windows | cross-distribution domain C1とMSIX First Playable subsetを分離 |
| Architecture Plan Closure Review | `ARCH-C149`、current実在性、missing evidence、official-source boundaryを追跡 |

## 8. Acceptance

1. 現計画で到達させるGameが一文とexact boundaryで説明できる。
2. required gameplayとexplicit non-requirementが相互排他的に分類される。
3. Product C1とInput cross-target／Windows cross-distribution domain C1を相互代用できない。
4. Shooter、3D、Mobile、managed／portable layoutをProduct C1 Evidenceへ使えない。
5. external official factとMiraikanai project decisionを混同しない。
6. 実装、実装計画、Work Package、Fixture、Registry、Receipt、Activationを追加または存在主張しない。
7. Owner header、relative link、anchor、canonical ownership、clean-break wordingが整合する。
