# Miraikanai Engine 計画書レビュー総括（2026-07-22）

- レビュー対象（原レビュー申告値）: `docs/` 全58 Markdown（約23,500行）。現行Gitから同じ母集団は再現できないため、closureでは`review_target_markdown`、`active_spec`、`modified_markdown`を別metricとして扱う
- 方法: 19文書グループの精読レビュー、4種の横断検証（リンク／ID／横断契約／E2Eギャップ）、外部依存ピン50項目の公式一次情報Web検証、指摘ごとの敵対的検証、確定指摘の修正適用、適用後の機械検査
- 詳細所見: [2026-07-22-plan-review-findings.md](2026-07-22-plan-review-findings.md)（明細が保持されている253件と外部検証表）
- Closure台帳: [2026-07-22-plan-review-closure.md](2026-07-22-plan-review-closure.md)（Finding ID、canonical decision、検証状態、処置、Evidence）

## 1. 結果総括

| 区分 | 件数 |
|---|---:|
| 詳細所見に保持されたFinding | 253 |
| CONFIRMED | 162 |
| UNVERIFIED | 43 |
| PLAUSIBLE | 48 |
| 詳細所見に保持されたREFUTED行 | 0 |
| 原レビューが報告したが明細未収録のREFUTED | 4 |
| 文書へ適用したedit action（Finding件数とは別metric） | 212＋横断統合22件 |
| 要決定source item／canonical decision | 46／37 |
| 原レビューでの適用見送り | 40 |
| 外部依存ピン検証 | 50項目: 誤り0、懸念4、検証不能1、確認45 |

全体評価: 計画書群は所有権分離・closed enum・fail closed・Evidence駆動の規律が一貫しており、外部依存ピンは50項目すべてで捏造・誤記が検出されなかった（名称不正確1件のみ、修正済み）。主な問題は個別文書の品質ではなく、(1) 文書間の相互参照結線の欠落・不一致、(2) ID体系の二重定義、(3) E2E目標（AIがゲームを作成→検証→ビルド）に必要な正本の空白、の3類型に集中していた。適用後の機械検査では編集52ファイル・リンク1,572件で破損0、文体・表整合の逸脱0を確認した。

## 2. 外部依存ピン検証（懸念・検証不能のみ）

| verdict | 対象 | 内容と対応 |
|---|---|---|
| concern | 単一プロバイダ(OpenAIのみ)前提の AI orchestrator 設計の妥当性 | 単一プロバイダ固定は「実装の単純さ」と引き換えに、モデル退役サイクル(6ヶ月通知)と Responses API 固有機能への結合という2つの事実ベースのリスクを負う。最低限 model ID を設定値化し、プロバイダ呼び出しを薄い境界層(1モジュール)に閉じ込めれば、複数プロバイダ抽象化を今作らなくても将来コストは限定的。マルチプロバイダ抽象を推奨する公式規範は確認できなかった。 |
| concern | 16KBページサイズ要件への対応と計画書での言及 | target API 36ピンのため本要件の適用対象。NDK r29既定で自前コードは対応するが、Oboe/libopus/libFLAC等の全native依存を含む最終arm64-v8a .soのalignment検証(zipalign/ELF segment検査)をPackage inspection Gateに明記すべき。計画書は本要件に言及すべきであり、現状の無言及は仕様ギャップ |
| concern | 4b. 「Android Vulkan Profile 2022」という名称のプロファイル | 実体は存在するが計画書の名称は不正確。「Android Baseline 2022」(VP_ANDROID_baseline_2022)に修正推奨。Vulkan 1.1要求なので4aのvulkan1.1ターゲットと整合。他にVP_ANDROID_baseline_2021、VP_ANDROID_15/16_minimumsも存在 |
| concern | MSVC C++23 modules (import std含む) のサポート状態と既知の制約 | 制約の一次情報: 公式チュートリアル(混在禁止・マクロ非公開・std.ixx手動/ビルドシステム経由コンパイル必要)と https://devblogs.microsoft.com/cppblog/msvc-build-tools-preview-updates-july-2026/ (C1001修正)。存在自体は確認できるが、計画がimport std+モジュールパーティション+サードパーティヘッダを併用するなら14.51では既知バグに当たる可能性があり、14.52安定化までの運用制約として明 |
| unverifiable | minSdk API 29 の妥当性(デバイスカバレッジ・Play要件) | OSバージョン別デバイスカバレッジの公式公開値は現在Web上に存在しない。旧Distribution DashboardページはVulkan/GLESのグラフィックス分布のみ表示し、バージョン分布はGoogle Play Consoleの「Reach and devices」(要ログイン)へ誘導される。PlayポリシーはminSdkを制約しない(要求はtarget APIのみ)。技術的にはAPI 29で64-bit端末のVulkan 1.1が保証されるため合理的だが、カバレッジ数値は公式一次情報 |

懸念4件のうち、Vulkan profile正式名（`VP_ANDROID_baseline_2022`）、MSVC v14.51のC1001既知bug、OpenAIモデル退役サイクルへの備えは本レビューで文書へ反映済み。16 KiBページサイズ要件はandroid.mdが既に正本範囲で所有していることを確認した（research agentの「言及なし」は誤り）。confirmed 45項目の出典URLは詳細所見を参照。

## 3. 適用した修正の類型

主な類型（全量は`git diff`と詳細所見を参照）:

1. **相互参照結線の欠落**: Product Plan §11 Registryの双方向参照不一致（phase↔work package）、dangling参照の解消、AI検証実行用`operation.build.*`／`operation.play.run_fixture`の新設結線。
2. **二重定義の参照化**: C++化性能閾値、Build Driver closed setの正本確定、旧Capability ID（`.c1`付き）の新ID置換、`windows_desktop_v1`等のTarget参照の`target.*`化（lock profile_idは要決定）。
3. **未定義規則・schemaの追記**: FSM/Rule評価順のcanonical規則、鍵Rotation手続き、Encounter/Weapon loadout系schema、`SupportBundleV1`、決定性同値クラス、`EnvironmentLightingSummaryV1`ほか。
4. **未固定依存の明示化**: C++ JSON parser／JSON Schema検証器／SHA-256／C++ test framework／MCP server SDK／Anthropic SDK／Target minimum OS（player）を「未固定」構造で登録（値の捏造なし、採用GateへFeature開始を接続）。
5. **実現可能性の修正**: 未Qualified revisionのPlay一律拒否→Development Play分離、D3D12計画のcompute barrier queue不整合、レイアウト正本（naming-project-layout）へのproject-state §6全面追随。

## 4. 要決定source item（レビュー時点）

以下46件はレビュー時点のsource itemであり、重複・相反を含む。現行判断は[Closure台帳](2026-07-22-plan-review-closure.md)の37 canonical decisionを正本とし、この一覧の推奨を直接適用しない。

1. **[product-index]** Decision(2026-07-21文書体系再構築)は§10.1で正本42件・§12.6で861 pairを固定するが、実態はproject-shader.md含む43件で、README正本指定のGateが恒久failする。
   - 推奨: project-shader.md追加を記録する追補Decisionを作成し§5構造・§6移行表へ由来を追記する。§10.1/§12.6の件数・pair数はハードコードせず「Index掲載数と実ファイル数の一致」としてInventoryから導出する規則へ改訂する。
2. **[product-index]** wp.architecture.control-planeのOwner `mirakan.arch.architecture-governance`は43正本のどの文書ID宣言にも無いdangling参照。Phase 0起点WPのため実装着手時に即解決不能。
   - 推奨: architecture-governance.md新設をDecisionで先に確定するのが本筋。暫定回避なら`mirakan.arch.product-plan`へ仮割当し行へplanned注記、§11.1へowner_document_id実在検証(fail closed)規則を追加する。規則のみ先行追加は現行行が即違反となるため見送った。
3. **[product-index]** Phase 1のauthoring-roundtripはmanual_and_ai_e2eだがAI経路はPhase 4まで無くPhase 1で検証不能。逆にPhase 4のexit fixture(2d-shooter)は同要件を参照せず結線欠落。
   - 推奨: `requirement.product.authoring-roundtrip-manual`(manual_e2e)を分離してPhase 1へ割当、authoring-roundtripはAI側検証としてPhase 4へ移し、exit fixtureへfixture.product.authoring-transactionを追加した上でAI経路を検証するfixture結線を定義する。
4. **[product-index]** Phase 6(3D First Playable/MVP-B)の成果要件がC2要件c2-3d-coverage(cross_genre_matrix)で、§6/§7.4/§11.7のFirst Playable限定方針と矛盾しPhase 6が定義上exit不能。
   - 推奨: 2D側(Phase 3=title-to-result/c2-2d-coverage=Phase 8専用)と対称に`requirement.product.3d-first-playable`(playable_e2e、Windows限定)を新設してPhase 6行とfixture.product.3d-shooter-arenaへ結線し、c2-3d-coverageはPhase 8行専用へ限定する。
5. **[product-index]** §5のMVP完了chain(install→first-run settings→offline→diagnosis→support/reset→uninstall)を実ゲームで検証する§11.3 registry結線が無く、完了chainがどのMilestoneで閉じるかも未定義。
   - 推奨: `requirement.product.mvp-completion`(clean install・offline・diagnosis・support bundle・data resetを含むverification_kind)を§11.3へ新設し、fixture.product.2d-shooterまたは専用fixtureへ結線してPhase 4の成果要件へ含める。
6. **[product-index]** Phase 4 Exit要点「Project Source Activation」(Project Native Capability/Project Shader各1利用)に対応するWP・Requirement・Capability行が§11に無く、表外ID拒否規則下で登録も検証も不能。
   - 推奨: `capability.project.native_module`と`capability.project.shader`をowner WP・必須Target付きで§11.6へ、`requirement.product.project-source-activation`を§11.3へ新設し、Phase 4(phase.ai-authoring-mvp-a)の成果要件へ結線する。
7. **[product-index]** §11.6 Capability registryの網羅性が未定義。Collision/Physics/Animation/Input/Audio等のC0/C1基盤Capability行が無く、physics.mdがRegistry未定義のCapability IDへ依存する実害がある。
   - 推奨: control-plane設計書§26の方向に沿い(b)単一正本と明文化し、Subsystem C0/C1 Capabilityの発番規約と追加手順(Registry行追加+owner WP必須)を定義する。ただし同設計書は未承認のためDecision確定を先行させる。
8. **[product-index]** wp.runtime.ecs-e0の「E0」は承認済正本のどこにも定義が無く、唯一の定義は未承認のRuntime ECS Contract Decision(ユーザー確認待ち)のみで、スコープ境界を承認済正本だけで解決できない。
   - 推奨: ECS Decisionの承認と新正本(04-runtime/entity-component-system.md)追加をPhase 0着手の前提として確定し、承認時にE0定義への正本参照をWP行へ結線する。承認まで当該WPを実質blockedとする扱いはDecision側で明示する。
9. **[product-index]** ランタイムSave dataの集約正本が不在。ECS Decision等がpersistence-save.md等5文書をOwner参照するが全て未作成でREADME正本一覧にも無く、Gameplay Save payload形式・atomicity・集約migrationのOwnerが参照切れ。
   - 推奨: persistence-save.md等の未作成正本をplanned状態でREADMEへ登録するDecisionを起こすか、作成完了までの暫定責務先(project-stateまたはscheduling-lifetime)をREADMEに1行宣言し、ui-text L534の委譲先参照切れを解消する。
10. **[product-index]** 「AIが自然言語からゲームを作成」に必須のAsset(sprite/音/3D)のMVP調達経路(同梱reference/生成Provider/User提供)を所有する正本がdocs/全体に無い。ui-textのC1/C2宣言はUI画像限定。
   - 推奨: MVPは(a)Domain Pack同梱reference asset set+C2まで生成Provider無しを推奨。Product PlanのMVP節へ「First Playable Assetの調達経路」を追記し、Ownerをasset-lifecycleまたはdomain-pack-contractへ割当てる。
11. **[product-index]** エンジン開発自体の体制前提(人数・期間)、E2E到達の現実性・クリティカルパス評価、プロジェクトレベルのリスク登録簿を所有する文書が無い(実施順序自体はPhase sequenceが所有済み)。
   - 推奨: Product Planへ「開発体制と進行前提」節を追加するか別文書を新設し、想定体制(例: solo+AI)、Phase毎概算期間、MVP-Aクリティカルパス、スコープ削減時の降格候補正本、リスク登録簿Ownerをユーザー入力に基づき定義する。
12. **[governance]** CI laneは11本定義済みだが、実行基盤(CIホスティング方式、GPU付きWindows runner、Android／Apple実機device matrix、hardware-VM Sandboxホスト、macOS build host)の計画をどの文書も所有しない(検証済み・is_real=true)。
   - 推奨: docs/architecture/02-foundation/配下(またはdocs/developer-tools/)に「CI／実行基盤」正本を新設し、CIホスティング方式、runner構成(GPU付きWindows、macOS build host、hardware-VM Sandboxホスト)、mobile-device lane用実機device matrixの調達・保守、容量計画と所有者を定義する。toolchain-dependencies.md 2.3のApple Shipping backend限定(apple_xcode_cloud_v1／apple_self_hosted_split_v1)と整合させ、provenance 14のlane表からは新正本を参照する。
13. **[foundation-core]** `McdContractRefV1`の`contract_set_hash`はset全体のMerkle rootのため無関係な文書変更で全永続参照が旧hash化するが、旧hash参照の検証基準（参照先set基準か現行set基準か）とset更新時の一括書換え要否が未定義。
   - 推奨: control-plane計画のref再設計とclass別方針（Project Source=offline migration、Replay=exact contract set必須等）の承認を待ち、承認後にexecutable-contracts §5を新ref形状へ置換して解決規則の正本をcompatibility-evolution.mdへ委譲する。承認前に§5へ一律規則を書かない。
14. **[foundation-core]** Target IDの二重体系: product-plan §11.2は`target.windows.desktop`等を正本とし「Target IDへ`v1`を埋め込まない」と定めるが、toolchain §8は`windows_desktop_v1`/`android_mobile_v1`/`apple_mobile_v1`を要求し、apple.md/windows.md/android.mdも旧IDを使用（検証済みis_real=true）。
   - 推奨: control-plane実装計画D.2の定義済みmigration（`target.*`／`driver.*`系）をtoolchain-dependencies §8とapple.md/windows.md/android.mdへ一括適用し、product-plan §11.2を正本とする。別namespace宣言案はID二重管理を恒久化するため非推奨。
15. **[foundation-core]** MCD ID文法違反: requirementの二重文法（`MIRAKAN-<DOMAIN>-<番号>`形 vs product-planの`requirement.product.authoring-roundtrip`形）、数字先頭segmentの`capability.product.2d_general_production`等がGrammar違反、例示`capability.render.material.toon_v1`の陳腐化（検証済みis_real=true）。
   - 推奨: requirementは共通Envelope文法`requirement.<namespace_path>`へ寄せてMIRAKAN形をDiagnostic code専用に限定（対応表をmigrationで定義）、数字先頭segmentはGrammar改訂ではなくID改名（例: `capability.product.general_production_2d`）を推奨。決定後に例示IDも移行後形式へ一括更新する。
16. **[foundation-lang]** cpp23-modules §4のshipping_allowedの意味(Shipping Configuration build可否かProduction Release可否か)が未定義で、CX3が未出荷外部Toolchainに依存するためMVP必須のCook/Package/Install(signed internal MSIX含む)が無期限blockされ得る。
   - 推奨: shipping_allowedを「Shipping Configuration buildの生成可否」と定義した上で、内部配布・MVP packagingはCX0のDevelopment/Profile構成+署名で許可する例外を§4へ明記し、Production Release昇格のみCX3必須とする。product-plan §5/windows.md C1との結線更新を同一ChangeSetで行う。
17. **[foundation-lang]** Diagnostic識別子の二重文法: naming §3.3のdot形(diagnostic.<domain>.<condition>)と14文書のMIRAKAN-<DOMAIN>-<NAME>形が併存し、照合キーが一意に決まらない。
   - 推奨: dot形をDiagnostic IDの唯一の正本文法、MIRAKAN形をMirakanDiagnosticV1.codeの表記として別概念定義し、diagnostic.<domain>.<condition>↔MIRAKAN-<DOMAIN>-<CONDITION>の1:1機械変換規則をnaming §3.3へ追記する案が最小diff。executable-contracts §12.1のcode所有と両立する。
18. **[foundation-lang]** naming §3.2の一般文法mirakan.<kind>.<domain>.<specific_name>.v<major>と、同文書自身のprefix/versionなしOperation/Diagnostic基本形・docs全域の実ID(準拠形0件)が矛盾し、§14 Commit Gateの文法検査が既存IDをほぼ全拒否する。
   - 推奨: §3.2へkind別closed文法一覧を追加: mirakan.+v<major>は文書ID等の限定kindのみ、operation./diagnostic.は既存基本形を正式文法とし、fixture./wp./requirement./phase./fallback./capability.はprefix・versionなし文法をproduct-plan §11.2の設計(Target IDへv1非埋込)と整合する形で定義する。
19. **[foundation-lang]** (unverified idx0、検証でis_real=true) naming §3.2のsegment=snake_case規定に対し、product-plan §11の複数語segment ID(fixture.product.headless-contract-smoke等)がkebab-caseで矛盾。diagnostic.product.*はnaming所有文法への明確な違反、executable-contracts §5のnamespace_path文法(hyphen不可)とも矛盾。§11.6 capability IDはsnake_caseで§11内でも混在。
   - 推奨: naming §3.2のsnake_caseを正本とし、product-plan §11のkebab segmentをsnake_caseへ機械正規化する(executable-contracts namespace_path文法とも整合)。決定後にproduct-plan.mdの該当ID全件を単一ChangeSetで置換し、ID文法lintの基準文書をnamingへ一本化する。
20. **[foundation-lang]** cpp23-modules §16.2のCX3 Gate必須Build matrixに「portable Linux Clang」laneがあるが、toolchain-dependenciesにLinuxのCompiler/CI image/profileが存在せずprofile_idも3値closedのため、Gateが実行不能。
   - 推奨: toolchain-dependenciesへlinux_ci_v1 profile(Clang exact version/hash/CI image digestは「未固定」構造で定義し、未固定状態ではlane無効の既存§2.1規則を適用)を追加してlaneを維持する。削除案はCX3の検証範囲を弱めるため非推奨。
21. **[authoring-editor]** Timeline分離に伴うpanel_type_idの具体値（例: `motion.timeline`／`diag.timeline`）の文法定義。suggestionはdot区切りgroup prefix案を提示。
   - 推奨: panel_type_id文法を`<group>.<panel>`のdot区切りsnake（例: motion.animation_timeline、diagnostics.debug_timeline）としてEditor UX規約§5.1へclosed規則として追補する。
22. **[authoring-gameplay]** 汎用Definition kind(Rule／ECA、FSM、bounded BT、Ability、Quest／Dialogue、UI Flow、Presentation cue、汎用Balance table)のV1 schemaが未定義。suggestionは「最小kindのV1 schemaを本書へ追加」か「kindごとの確定Phase期限を明記」の二択を提示。
   - 推奨: C1で必要な最小kind(Rule／ECA、FSM)のV1 schemaを§2.4のPerception／Interactionと同じField単位形式で本書へ追加し、残る汎用kindはProduct PlanのPhase結線で確定期限を定める。
23. **[runtime]** Debug容量（Store ring、ingress queue、disk retention、instrumentation overhead閾値）がdebugging §7の委譲先であるperformance-capacityに存在せず、DBG1／C1 qualificationが導出不能。
   - 推奨: performance-capacity §5へ`DebugCapacityRequestV1`（instrumentation tier別ingress entry／arena、Store ring bytes、disk quota、overhead上限%）を定義し、§3のUnassigned headroom 128 MiBを64＋Debug 64へ分割して`target.windows.desktop`のC1既定値を固定する。同時にplausible指摘のhang検出閾値もこの契約へ収容する。
24. **[runtime]** toolchain-dependencies.mdの`toolchain_lock.profiles[].profile_id`（`windows_desktop_v1`等、L224／L226）とcpp23-modules.md L381の参照を、Target IDと同一namespaceとして`target.*`へ置換するか、別namespaceとしてimpl plan D.2の対象外と明記するか。
   - 推奨: 二重綴りの再生産を避けるためprofile_idも`target.*`へ揃える。別namespaceとして残す場合はtoolchain-dependencies.mdへ「profile_idはTarget IDと別namespaceでD.2対象外」の明記とTarget IDへの対応表を必須とする。
25. **[rendering-core]** Shadow authoringの`ShadowIntentV1`／`ShadowStyleProfileV1`／`ShadowGraphV1`／`ResolvedShadowPlanV1`はrender-graph.md §12でのみ出現し、schema定義とOwner帰属が存在しない(指摘2の本体)。
   - 推奨: ShadowIntentV1/ShadowStyleProfileV1/ResolvedShadowPlanV1はlighting.md(正本範囲に「shadow intent」既存)へ帰属させschemaを定義し、closed Pass Templateへcompileする`ShadowGraphV1`のみrender-graph帰属と明記する案を推奨。shadow.md新設はREADME登録を伴うため次点。
26. **[rendering-core]** `VisualStyleProfile.composition`のclosed値集合が未定義のまま残る(指摘3の残余)。
   - 推奨: compositionとcomposition_variantの役割を整理し、compositionの値集合をStyleCapabilityManifestのStyle feature IDとして定義するか、冗長であればcomposition fieldを削除してcomposition_variantへ一本化することを推奨。
27. **[rendering-fx]** layer閉語彙の文書間不一致: post-processing.md §5のPostProcessLayerPolicyV1は7値、render-graph.md §9のexcluded_layersは`ui|text|pixel_locked`の3値で対応表がなく、AA Plan/Post Process Plan互換性検証の仕様が不定。
   - 推奨: plan §13.2のOwner割当どおりrender-graph.mdのLayerCompositionSummaryV1へ7区分を定義してSSoT化し、excluded_layersのui|text|pixel_lockedをui_text/pixel_locked_worldへ写像する対応表をrender-graph.mdへ追加、post-processing.md §5は参照へ置換する。
28. **[rendering-fx]** CameraProfileDocumentV1の`output_policy`の閉語彙が未定義(camera.md §2.1、出現1回のみ)。focus_policyとCameraComfortProfileV1は今回定義済みで、残るのはoutput_policyのみ。
   - 推奨: Camera所有はview出力区分の最小閉語彙(例: `primary_view | secondary_view | offscreen_export`)に限定し、HDR/SDR output transformはrender-graph.md/UI文書への参照とする。
29. **[rendering-world]** Water/Snow/WeatherのCapability IDが未定義のまま、§8のSnowDynamicCapabilityMissing DiagnosticとAI eval（Water capability mismatch、dynamic Snow cost）、§3/§4/§7のbaseline/advanced区分が存在しないCapabilityを前提としている。
   - 推奨: product-plan §11.6へ`capability.water.baseline`(C1)/`capability.water.advanced`(C2)/`capability.snow.dynamic_field`(C2)/`capability.weather.presentation`(C1)をowner WP=wp.rendering.environment-c2、declared_unscheduledで登録し、environment-surfaces §6のclosed集合へ同IDを追記。MVP-A/Bに不要なら未着手と明示。
30. **[rendering-world]** environment-surfaces.md §8のDiagnostic約45件が裸のPascalCase名で、Foundation正本形式`MIRAKAN-<DOMAIN>-<NAME>`（executable-contracts §12.1）に従っていない。同種逸脱はvfx-authoring.md/vfx-runtime.mdにも存在する。
   - 推奨: §8の既存グループ分け通りにMIRAKAN-ENVIRONMENT-*/MIRAKAN-WATER-*/MIRAKAN-SNOW-*/MIRAKAN-WEATHER-*の4 domain segmentへ分割改名し、vfx-authoring.md/vfx-runtime.mdのMIRAKAN-VFX-*改名と同一ChangeSetで適用する。
31. **[rendering-world]** lod.mdの`experience_role`（§2拒否条件・§7 SimulationLodContractV1 field）、`LodIntentV1.semantic_role`、HLOD適格条件の`decorative_instance`がいずれもclosed enum未定義で、SEMANTIC_ROLE_CONFLICT矛盾判定が実装不能。
   - 推奨: gameplay-programming-modelへ`semantic_role`のclosed enum（`decorative_instance`を含む）を正本定義し、`experience_role`は同enumの別名として`semantic_role`へ統一、lod.md §2へconflict判定の組合せ表を追加する。performance-capacity §9の散文言及も同参照へ揃える。
32. **[platform-os]** mobile-common §7の「Render transient内数／Streaming cache内数／Emergency reserve内数」3列の親capが未指定（performance-capacityの委譲先表はwindows_desktop_v1専用、EmergencyはCPU／GPU両envelopeに存在）。
   - 推奨: Render transient→GPU working set列の内数、Streaming cache→Engine CPU列の内数、Emergency reserve→Process footprint列の内数（CPU／GPU横断のcontrolled shutdown専用）として列注記を追加する案を推奨。
33. **[platform-os]** `render_quality_tier`（Baseline|Standard|High）×`memory_class`（mobile_baseline|standard|high）の許容組合せ表が未定義（例: mobile_standardでHighを許すか）。軸の区別とFG行の両条件要求は今回明記済み。
   - 推奨: memory classをquality tierの上限とする表（mobile_baseline→Baselineのみ、mobile_standard→Standard以下、mobile_high→High以下）を§5.4へ追加し、例外は§5.3 FGのように個別Gateを明記する方式を推奨。
34. **[platform-os]** FG合格条件「Provider-off比の劣化8.33 ms以下」は補間型Providerの構造的保留遅延（約8.3 ms）とほぼ等しく、事実上外挿型のみが合格可能。導出根拠と補間frameの「最初の輝度変化」測定定義も未記載。
   - 推奨: 外挿型providerのみ候補と明記して8.33 msを維持する案を推奨（touch latency劣化を実質1表示frame未満に抑えるというmobile FGの品質意図に整合）。補間型を将来許容する場合のみ16.67 msの別laneを追加。
35. **[domain-packs]** Shooter Coreの正本IDが二重体系のまま併存する。shooter.mdは成熟度埋め込みID(mirakan.feature.shooter_core.c1、shooter.profile.*.c1、mirakan.domain.*.c1)を全編で使うが、product-plan §11.6 Registryは`capability.gameplay.shooter_core`(tier別列)を登録し、§11.1は「maturityをidentityとして使わない」と明記する。
   - 推奨: 実装計画Appendix D.1/D.2/D.4の対応表どおりmigration ChangeSetを実行し、shooter.mdのIDを新stable ID(capability.gameplay.shooter_core、profile.shooter.*、domain.*、fixture.product.*)へ一括置換、旧IDはmigration表内のみに残す。16.7/16.8節fixture IDのD.4適用(2d_shooter_c1_v1→fixture.product.2d-shooter等)とProduct Plan fixture IDとの対応明示も同時に行う。
36. **[domain-packs]** `mirakan.domain.2d_action.c1`と`mirakan.domain.tps_single_player.c1`はshooter.mdが参照するが、docs/architecture配下に定義文書(所有schema・Capability・Template)を持たないdangling referenceである(compose方向の表記不一致は今回適用済み)。
   - 推奨: 上記ID migrationと同時に、domain composition(2d_action/tps_single_player)の正本文書を08-domain-packsへ新設して所有schema・必須Capability・Templateを定義し、README正本一覧へ登録する。新設しない判断の場合はshooter.mdの構成図をprofile+feature packのみへ縮退させ、D.2のDomain行も併せて削除する。
37. **[plans-control-plane]** Task 8A Step 4のcontinuation署名に使う「repository-owned signing key profile」が未定義。鍵Owner文書・algorithm・保管・CI供給・rotationがどこにも規定されず、repository内の鍵は改竄防止にならない(idx 11)。
   - 推奨: 署名を廃止し、continuation payloadへrequest hashとSource closure hashをSHA-256でbindする完全性検証方式へ簡素化する(再利用防止はrevision/scope bindで既に達成済み)。採用時は設計§7.6と実装Task 8A Step 4・Step 1のfixture名を同時に修正する。
38. **[foundation-core]** toolchain_lock `profiles[].profile_id` namespace（`windows_desktop_v1`等）とTarget ID文法`target.*`の二重体系の解消方向（toolchain-dependencies L233付近の列挙、cpp23-modules L381の`profiles[apple_mobile_v1]`参照、windows.md L42の`profile_id = windows_desktop_v1`が対象）。
   - 推奨: lock profile_idはtoolchain lock専用namespaceとして現行suffix付きIDを据置き、Target Profile registryの`target.*`との1:1対応表をtoolchain-dependencies §8へ明記する案を推奨。統一する場合は`target.*`へ揃え、lock schema version更新を伴う専用ChangeSetで実施。
39. **[foundation-lang]** Diagnostic ID文法の統一（`MIRAKAN-<DOMAIN>-<NAME>`形式とdotted `diagnostic.*`形式の二重文法）。vfx-authoring/vfx-runtimeの裸PascalCase Diagnostic名改名、render-graphのbackend非依存cycle diagnostic ID付与もこれに従属。
   - 推奨: executable-contracts §12.1の`MIRAKAN-VFX-<NAME>`形式へ寄せ、vfx-authoring・vfx-runtime・environment-surfacesを同一ChangeSetで改名。d3d12計画のdotted ID群との関係整理を同時に決定する。
40. **[authoring-gameplay]** `semantic_role`のclosed enum（`decorative_instance`含む）をgameplay-programming-modelの正本として定義し、lod.md/render-graph.md/performance-capacity.mdからexact参照させる件。
   - 推奨: gameplay-programming-model §2または§3へclosed enumを新設し、render-graph §7の`decorative_instance`等の消費箇所を参照化する。
41. **[rendering-world]** product-plan §11.6へwater/snow/weatherの4 capability行を追加し、実装計画のCompletion Gate「Capability 34」件数を更新する件。
   - 推奨: 推奨行: capability.water.baseline(C1)・capability.water.advanced(C2、fallback→water.baseline相当)・capability.snow.dynamic_field(C2、static mask fallback)・capability.weather.presentation(C1)、全行owner WP=wp.rendering.environment-c2。注意: 今回lifecycle列はactivation列へ置換済みのため、追加時は`capability_activation_state=not_activated`で登録し、defer 3 fieldはowner WP側scheduling規律に従う。
42. **[rendering-core]** materials.md §4.1の`realistic_advanced` compile feature tierに対応するCapability IDのCapability Registry登録（CAPABILITY_NOT_ACTIVATED gate結線）。
   - 推奨: `capability.render.material.realistic_advanced`（C2、fallback.rendering.material-default、activation=not_activated）を追加し、owner WPは既存wp.rendering.material-toonの流用不可なら`wp.rendering.material-realistic`を新設する案を推奨。
43. **[authoring-gameplay]** scheduling-lifetime §4固定tick phase表へのPerception pipeline（候補・Query生成/Query batch処理/結果正規化）の正式割当。
   - 推奨: 候補・Query生成をT30_PrePhysics、Query batch処理をT50_PhysicsStep内のCollision query slot、結果正規化をT60_PhysicsIntegrate後段へ割り当てるADRを起票し、phase表とgameplay-programming-model §2.4の消費参照を同一ChangeSetで確定する。
44. **[plans-misc]** decisions/2026-07-21-document-system-restructure.md §12.4/§12.6項8のreasoning effort記述（high前提）がPR #5のxhigh承認によりstale。
   - 推奨: §12.4へ「本判断はPR #5決定により上書きされた（現行はxhigh）」の追補注記を加え、§12.6項8は歴史的検証項目として注記する追補をDecision gateで承認する。
45. **[ecs-contract]** decisions/2026-07-22-runtime-ecs-contract.mdのV1なしin-memory型5種（RuntimeWorldInstanceHandle等）をnaming規約のV1省略例外Registry（Architecture Governance）へ一括登録する規定の追加。
   - 推奨: in-memory value type群（handle/ordinal/generation table）をカテゴリ単位でV1省略例外として例外Registryへ登録する規定をECS active spec昇格Approvalと同時に追加する。
46. **[rendering-fx]** layer語彙のSSoT化（post-processing側7区分定義とrender-graph `excluded_layers`の対応表）。LayerCompositionSummaryV1自体は構造定義を適用済みで、語彙統一のみ残る。
   - 推奨: Render GraphのLayerCompositionSummaryV1のlayer entry語彙を正本とし、post-processingの7区分は参照＋対応表化する。

## 5. レビューの限界

- minSdk API 29のデバイスカバレッジ数値は公式Web公開が廃止されており検証不能（Play Console内限定）。
- SM6.6 bindless等の高度なHLSL機能が3ターゲット変換（DXIL/SPIR-V/MSL）で成立するかは未検証。toolchain-dependencies §2.4へ検証Ownerを登録済み。
- low severityの未検証指摘48件は詳細所見に残しており、修正は「自明に安全」なもののみ適用した。
- 本レビュー自体もAIによるものであり、要決定事項の適用時は各推奨案の再確認を推奨する。
