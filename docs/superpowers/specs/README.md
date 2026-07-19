# Miraikanai Engine 設計文書Index

- 最終更新日: 2026-07-19
- 状態: Subsystem別正式仕様を統合した設計レビュー用Index

## 1. 読む順序

Miraikanai Engineの公式Review setは次の21文書である。上位のProduct判断から共通契約、Subsystem固有契約、検証規約の順に読む。

1. [AIネイティブ独自ゲームエンジン 設計計画書](./2026-07-18-ai-native-game-engine-authoring-design.md)
2. [Miraikanai Engine C++実行コード・構造化ゲームデータ規約](./2026-07-19-cpp-structured-game-data-design.md)
3. [Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)
4. [Miraikanai Engine Authoring Model／Project State規約](./2026-07-19-authoring-model-project-state-design.md)
5. [Miraikanai Engine 実行可能契約・Schema・Codegen規約](./2026-07-19-executable-contract-schema-codegen-design.md)
6. [Miraikanai Engine Runtime連携・寿命・性能規約](./2026-07-19-runtime-integration-lifetime-performance-design.md)
7. [Miraikanai Engine 2D／3D機能計画](./2026-07-19-2d-3d-capability-plan.md)
8. [Miraikanai Engine NativeGameModuleアーキテクチャ規約](./2026-07-19-native-game-module-architecture-design.md)
9. [Miraikanai Engine Asset Pipeline／Content Package規約](./2026-07-19-asset-pipeline-content-packaging-design.md)
10. [Miraikanai Engine Rendering／Render Graphアーキテクチャ規約](./2026-07-19-rendering-render-graph-architecture-design.md)
11. [Miraikanai Engine Collision／Colliderアーキテクチャ規約](./2026-07-19-collision-collider-architecture-design.md)
12. [Miraikanai Engine Physics Dynamics／Navigation／Animation規約](./2026-07-19-physics-navigation-animation-architecture-design.md)
13. [Miraikanai Engine Input／Action／Device規約](./2026-07-19-input-action-device-architecture-design.md)
14. [Miraikanai Engine UI／Text／Localization／Accessibility規約](./2026-07-19-ui-text-localization-accessibility-design.md)
15. [Miraikanai Engine Audio／Mixer／Spatial規約](./2026-07-19-audio-mixer-spatial-architecture-design.md)
16. [Miraikanai Engine Editor／Workspace／UX規約](./2026-07-19-editor-workspace-ux-design.md)
17. [Miraikanai Engine Windows Platform／Distribution規約](./2026-07-19-windows-platform-distribution-design.md)
18. [Miraikanai Engine モバイルPlatformアーキテクチャ規約](./2026-07-19-mobile-platform-architecture-design.md)
19. [Miraikanai Engine Domain Pack／将来Capability規約](./2026-07-19-domain-pack-future-capability-roadmap.md)
20. [Miraikanai Engine AI実装・保守ガバナンス規約](./2026-07-19-ai-engine-development-governance-design.md)
21. [Miraikanai Engine AI検証・評価・来歴規約](./2026-07-19-ai-verification-evaluation-provenance-design.md)

`2026-07-18-codex-config-optimization-design.md`は開発者個人のCodex設定資料であり、Engineの製品・Runtime・Editor・AI契約に対する決定権を持たず、公式Review setへ含めない。

## 2. 文書ごとの決定権

| 文書 | 決定する範囲 |
|---|---|
| AIネイティブ設計計画書 | Product vision、制作体験、AI／人間の関係、Phase、MVP、Review set全体の上位目的 |
| C++実行コード・構造化ゲームデータ規約 | C++／data境界、GameplayDefinition、NativeGameModule、AI実装選択、Script VM不採用 |
| 基盤アーキテクチャ規約 | C++、Memory、Pointer、Module、Dependency、Build、命名、Directory |
| Authoring Model／Project State規約 | Authoring Document、ProjectRevision、ChangeSet、transaction、journal、recovery、唯一の状態変更経路 |
| 実行可能契約規約 | Requirement、Type、Operation、State、Capability、Schema projection、Codegen |
| Runtime連携規約 | Tick、Writer、Command／Event／Snapshot、寿命、Asset version、Budget、Failure |
| 2D／3D機能計画 | 2D／3D Capability範囲、成熟度、Visual Style、各Subsystemの製品上の到達点 |
| NativeGameModule規約 | 公開ABI、Target別link方式、Build／Promotion、GameHost restart、信頼境界 |
| Asset Pipeline／Content Package規約 | Source／Import／Derived／Package、Importer隔離、Catalog／VFS、Cook、Patch／DLC、AI Asset来歴 |
| Rendering／Render Graph規約 | RenderSnapshot、extract、Render Graph、resource／pass／access、Backend、同期、device loss |
| Collision／Collider規約 | Body／Collider／Shape／Material／Filter、Query、Contact／Trigger、Cook、Editor／AI操作 |
| Physics／Navigation／Animation規約 | Dynamics、Nav build／query、2D grid nav、Animation Graph、root motion、固定phase連携 |
| Input／Action／Device規約 | Device sample、semantic action、InputSnapshot、remap、replay、text入力分離、haptics |
| UI／Text／Localization／Accessibility規約 | Retained UI、layout、event／focus、IME、shaping／raster、localization、Game accessibility |
| Audio／Mixer／Spatial規約 | Audio cue／voice／bus、mixer、streaming、spatial audio、callback、device adapter |
| Editor／Workspace／UX規約 | Window／Panel／Dock、Scene／Outliner／Inspector、AI Partner、AI Creator、workspace、Editor accessibility |
| Windows Platform／Distribution規約 | Windows Target、MSIX／folder distribution、signing、crash privacy、package layout |
| モバイル規約 | Android／iOS／iPadOS Target、Adapter、Lifecycle、Build／Signing／Upload分離、実機、Store |
| Domain Pack／将来Capability規約 | Genre Pack境界、初期Pack、将来Capabilityの非対象範囲と設計着手Gate |
| AI実装・保守ガバナンス | AI Task、権限、Risk、質問、Source sandbox、API／MCP／CLI／Plugin |
| AI検証・評価・来歴規約 | Gate、Test、形式モデル、Eval、Receipt、SBOM、Provenance、最新情報 |

同じ主題で矛盾する場合は、その主題の決定権を持つ正式仕様を優先する。上位のProduct目標を下位文書が変更してはならず、下位の具体的な安全規約を上位文書の概略表現で緩和してはならない。複数Subsystemに跨る変更では、各所有文書を同じChangeSetで更新し、Runtime phase、Schema、Memory、Target、Testの整合を確認する。

## 3. 外部規格とプロジェクト規約

外部公式資料はAPI、Platform、標準、Libraryの事実と推奨を提供する。Miraikanai Engine固有のphase、Budget、Risk、Approval、Directory、選択閾値は、本Review setが一次資料を検討して確定したプロジェクト公式規約である。

文書内の「最新」はfloating versionを意味しない。調査基準日に確認したStable／LTS／approved artifactをexact versionとhashで固定し、更新はEvidence、Eval、ADR、Reviewを経由する。

## 4. 設計から実装計画へ進むGate

実装計画書の作成へ進む前に、次をすべて満たす。

- ユーザーが公式Review set 21文書の方向性をReviewし、修正点または承認を返す。
- 文書間Link、Heading anchor、Table、Code fence、規範語の機械検査が通る。
- C0／C1に必要な選択が暗黙Defaultになっておらず、外部artifactから取得する値には取得手順、Owner、Block条件がある。
- Authoring、Runtime、Renderer、Asset、Editor、Input、UI、Audio、Physics、Navigation、Animation、Platform、AIの境界にOwner、入力、出力、phase、budget、failure、testが定義されている。
- Phase 0から2D First Playableまでを、依存関係と検証可能な完了条件を持つ実装taskへ分解できる。

このGateの通過は、**実装計画書を作成してよいことだけ**を意味する。Engine本体の実装開始を承認するものではない。

## 5. 実装開始とPhase完了Gate

Engine本体の実装は、公式Review set承認後に別文書として実装計画書を作成し、その計画書をユーザーがReviewして承認した後に開始する。

- 実装計画書はtaskごとに対象file、public contract、依存task、test、performance／memory gate、rollback、完了条件を記載する。
- Phase 0で生成するMiraikanai Contract Definition、Toolchain lock、Contract compiler、generated code、Gate、Fixtureは、設計Reviewの事前成果物ではなくPhase 0の実装成果物である。
- Phase完了は、該当文書のDefinition of Done、target matrix、conformance／integration／soak／package testが実artifactで合格した時だけ宣言する。
- 2D First Playable、3D First Playable、Android／Apple vertical slice、Production Capabilityは、それぞれ独立したMilestone Gateとして順に判定する。
- 仕様と実装が不一致になった場合、無断で仕様を実装へ合わせず、Evidenceと影響範囲を伴うADR／仕様ChangeSetを先にReviewする。
