# Miraikanai Engine 設計文書Index

- 最終更新日: 2026-07-19
- 状態: 設計レビュー用Index

## 1. 読む順序

実装開始前の公式Review setは次の10文書である。

1. [AIネイティブ独自ゲームエンジン 設計計画書](./2026-07-18-ai-native-game-engine-authoring-design.md)
2. [Miraikanai Engine C++実行コード・構造化ゲームデータ規約](./2026-07-19-cpp-structured-game-data-design.md)
3. [Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)
4. [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
5. [Miraikanai Engine Collision／Colliderアーキテクチャ規約](./2026-07-19-collision-collider-architecture-design.md)
6. [Miraikanai Engine 2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)
7. [Miraikanai Engine モバイルPlatformアーキテクチャ規約](./2026-07-19-mobile-platform-architecture-design.md)
8. [Miraikanai Engine AI実装・保守ガバナンス規約](./2026-07-19-ai-engine-development-governance-design.md)
9. [Miraikanai Engine 実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)
10. [Miraikanai Engine AI検証・評価・来歴規約](./2026-07-19-ai-verification-evaluation-provenance-design.md)

## 2. 文書ごとの決定権

| 文書 | 決定する範囲 |
|---|---|
| AIネイティブ設計計画書 | Product vision、制作体験、Editor、AI／人間の関係、Phase、MVP |
| C++実行コード・構造化ゲームデータ規約 | C++／data境界、GameplayDefinition、NativeGameModule、AI実装選択、Script VM不採用 |
| 基盤アーキテクチャ規約 | C++、Memory、Pointer、Module、Dependency、Build、命名、Directory |
| Runtime連携規約 | Tick、Writer、Command／Event／Snapshot、寿命、Asset version、Budget、Failure |
| Collision／Collider規約 | Body／Collider／Shape／Material／Filter、Query、Contact／Trigger、Cook、Editor／AI操作 |
| 2D／3D機能計画 | Render、Physics、Navigation、Lighting、Particle、Material、Asset、機能成熟度 |
| モバイル規約 | Android／iOS／iPadOS Target、Adapter、Lifecycle、Build／Signing／Upload分離、実機、Store |
| AI実装・保守ガバナンス | AI Task、権限、Risk、質問、Source sandbox、API／MCP／CLI／Plugin |
| 実行可能契約規約 | Requirement、Type、Operation、State、Capability、Schema projection、Codegen |
| AI検証・評価・来歴規約 | Gate、Test、形式モデル、Eval、Receipt、SBOM、Provenance、最新情報 |

同じ主題で矛盾する場合は、その主題の決定権を持つ文書を優先する。上位のProduct目標を下位文書が変更してはならず、下位の具体的な安全規約を上位文書の概略表現で緩和してはならない。

## 3. 外部規格とプロジェクト規約

外部公式資料はAPI、Platform、標準、Libraryの事実と推奨を提供する。Miraikanai Engine固有のphase、Budget、Risk、Approval、Directory、選択閾値は、本Review setが一次資料を検討して確定したプロジェクト公式規約である。

文書内の「最新」はfloating versionを意味しない。調査基準日に確認したStable／LTS／approved artifactをexact versionとhashで固定し、更新はEvidence、Eval、ADR、Reviewを経由する。

## 4. 実装開始条件

Engine実装計画へ進む前に次を満たす。

- ユーザーが10文書の方向性をReviewし、修正点または承認を返す。
- 文書間Link、Heading anchor、Table、Code fence、規範語の機械検査が通る。
- 未確定値が「暗黙Default」ではなく、取得手順、Owner、Block条件を持つ。
- Phase 0で作るMCD、Toolchain lock、Contract compiler、Gate、Fixtureが実装計画へ分解される。
- Engine本体の実装は、上記Review後に別の実装計画として作成する。
