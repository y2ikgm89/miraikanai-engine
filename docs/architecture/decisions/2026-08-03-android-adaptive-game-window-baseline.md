# Android Adaptive Game Window Baseline

- 文書ID: mirakan.decision.android-adaptive-game-window-baseline
- 状態: review
- 正本範囲: Android game packageをadaptive orientation／resizable windowへ固定し、game category例外へ依存しない判断理由
- 非正本範囲: exact Mobile Schema／manifest mapping／surface lifecycle／device matrix、GameActivity実装、UI／Camera layout、Fixture／Receipt、実装Task、実装順序。各Owner文書を参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Product Plan](../00-product/product-plan.md)、[Mobile Common](../07-platform/mobile-common.md)、[Android](../07-platform/android.md)
- 外部根拠検証日: 2026-08-03
- 文書種別: Architecture Decision／Android window baseline
- Decision owner document: `mirakan.arch.platform-android`
- Decision日: 2026-08-03
- Supersedes: none

## Context

Mobile Commonの`orientation_policy`／`resize_policy`は値が未確定で、Android generated manifestへの写像も閉じていなかった。Android 16／17のlarge-screen behaviorには`android:appCategory`に基づくgame例外があるが、例外を固定orientationまたはnon-resizableの正当化に使うと、phone、tablet、foldable、multi-window、desktop windowで別挙動を持ち、将来のPlatform変更へ脆弱になる。

## Decision drivers

- phone／tablet／foldableとwindow resizeを一つのProject／packageで扱う。
- orientation lock、hard-coded aspect ratio、size compatibility mode依存を避ける。
- game categoryを正しく宣言しつつ、例外をCapability保証に使わない。
- Surface、Input、safe area、UI state、Saveをresize／rotationで保持する。
- Project自由値からManifest制限を生成しない。

## Considered options

### A. landscapeまたはportraitへ固定する

採用しない。window size変更自体は防げず、large-screen／foldableでletterboxとstate／layout差を増やす。

### B. game例外を使ってnon-resizableにする

採用しない。公式の例外はMiraikanaiのadaptive supportを証明せず、Platform behavior変更と互換modeへ依存する。

### C. Projectごとにmanifest lockを選べるようにする

採用しない。initial V1のTarget semanticsとQualification matrixを分岐させ、同じProduct claimで異なるwindow capabilityを許す。

### D. game category＋adaptive-only baselineにする

採用する。Application categoryはgameへ固定し、orientation／resize／aspect制限を生成せず、全対応をdevice Qualificationで検証する。

## Decision

1. Android Application categoryはEngine-owned `android:appCategory="game"`である。
2. initial Mobile policyは`orientation_policy=adaptive`、`resize_policy=adaptive_required`だけを受理する。
3. generated manifestは`android:resizeableActivity="true"`を持ち、`screenOrientation`、min／max aspect ratio、restricted-resizability opt-outを持たない。
4. rotation、resize、multi-window、fold／unfold、display migrationでProject／World／Save identityを変えない。
5. phone／tablet／foldableと複数aspectを同じpackageでQualificationする。

## Consequences

- Android packageとMobile共通policyの関係が一意になる。
- orientation固定前提のGame contentはinitial Target Qualificationに合格できない。
- adaptive layout、Surface再構成、Input transform、state preservationの検証が必要になるが、本Decisionは実装方法を定めない。
- RepositoryにApplication、manifest generator、device FixtureまたはReceiptは存在せず、本DecisionはAndroid supportを主張しない。

## Canonical Owner documents

- Product／Target identity: [Product Plan](../00-product/product-plan.md)
- common orientation／resize semantics: [Mobile Common](../07-platform/mobile-common.md)
- Android manifest／surface／device qualification: [Android](../07-platform/android.md)

## Supersedes／Superseded by

- Supersedes: none
- Superseded by: none

## Official or primary sources

- [Android adaptive do's and don'ts](https://developer.android.com/develop/adaptive-apps/guides/adaptive-dos-and-donts)
- [App orientation, aspect ratio, and resizability](https://developer.android.com/develop/adaptive-apps/guides/app-orientation-aspect-ratio-resizability)
- [Develop games for all screens](https://developer.android.com/games/develop/all-screens)
- [Support large screen resizability for games](https://developer.android.com/games/develop/multiplatform/support-large-screen-resizability)
