# Miraikanai Engine 実現可能性 Remediation Review（2026-07-23）

- authority: [実現可能性 Remediation 設計](../superpowers/specs/2026-07-23-plan-feasibility-remediation-design.md)
- execution plan: [実現可能性 Remediation Plan](../superpowers/plans/2026-07-23-plan-feasibility-remediation.md)
- review baseline: Git commit `a87f35fcec9df0e76cb6240234853575a1276337`
- scope: Architecture正本、Product Registry、実装計画、外部一次資料との整合
- excluded: Engine runtime実装、Capability activation、Target Qualification、Release承認
- decision: `conditional_go_for_implementation_bootstrap_only`

## 判定

計画書は、実装計画と最小Bootstrapへ進むための入力としては**条件付きGo**とする。Product、AI操作、Control Plane、D3D12、Critical Path、Future incubationのOwner、Gate、fail-closed条件を同じ正本へ接続し、未成立Capabilityを成立済みと表示しない境界を置いたためである。

公開Shipping、Production readiness、現在利用可能なCapabilityの主張は**No-Go**である。Engine code、build、実機、性能、Security、Package、Releaseの実行Receiptは本変更で生成していない。Activation Registry／Receiptの実装Artifactはまだmaterializeしておらず、現在の有効なActivation Receiptは0件である。materialize時は全required／optional `CapabilityTargetActivationV1`行を`not_activated`、`candidate_ref=null`、`receipt_refs=[]`、`evidence_freshness=expired`で初期化し、文書承認、Work Package登録、Decision Gate成立から自動昇格させない。

## 旧Closureの意味

[2026-07-22 Plan Review Closure](2026-07-22-plan-review-closure.md)のretained Finding 253件がterminalであることは、各Findingの検証状態と処置が終端化されたという**disposition closure**だけを意味する。Engine実装完了、build成功、Target Qualification、Capability readiness、Release readinessを意味しない。

原レビューが件数だけ報告し、本文を保持しなかったREFUTED 4件は、Finding ID、主張、根拠の詳細が一度も保存されていない。今後も推測で再構成せず、欠落した履歴として件数だけを保持する。

## Remediation finding ledger

| finding_id | scope | remediation result | decisive evidence | state／remaining condition |
|---|---|---|---|---|
| `remediation.feasibility.product` | Product schema、DAG、Phase Gate、Mobile、C2 | Work Package schema、Phase固有Gate、Target closure、Mobile 2D／3D GateをProduct正本へ統合し、Critical Pathの74 WPへ接続した | [Product Plan](../architecture/00-product/product-plan.md)、commit `454501c`、`f2c7584`、`05dace5`、[Critical Path](../plans/2026-07-23-critical-path-execution-plan.md) | 実装Bootstrap入力として受理。C2 3Dは第二の非Shooter 3D Fixtureがないため`deferred` |
| `remediation.feasibility.ai-control` | AI Operation、Caller Profile、Code owner | Package、Install、Launch、Smoke、Debug、Support、Reset、Task操作をexact Operationへ閉じ、Host／Transport／Provider／Deployment／Model／Authorityを直交化し、Native／Shader生成をCode owner署名とrevocationへ結んだ | [AI Security／Approval](../architecture/01-governance/ai-security-approval.md)、[Core Architecture](../architecture/02-foundation/core-architecture.md)、[Gameplay Programming Model](../architecture/03-authoring/gameplay-programming-model.md)、commit `1d7d758`、`3a60e4e`、`e8303dc` | Contractは実装可能。Gateway、Policy Service、署名Service、Host conformanceの実装Receiptは未生成 |
| `remediation.feasibility.control-plane` | Bootstrap、Ajv、Readiness、Freshness | canonical schema、offline Draft 2020-12 validation、Bootstrap AuthorityとSigner Roleのexact binding、署名済みTechnical Qualification、Activation行のmissing／duplicate／extra拒否、Target readiness、Evidence freshnessを明文化した | [Control Plane Design](../plans/2026-07-22-architecture-evolution-control-plane-design.md)、[Control Plane Implementation Plan](../plans/2026-07-22-architecture-evolution-control-plane-implementation-plan.md)、commit `192af5c`、`ee29d8d`、`4539f10`、`2519d33` | 実装Bootstrap入力として受理。schema compiler、Ajv lock、署名／revocation、negative fixtureは実装時に実行必須 |
| `remediation.feasibility.d3d12` | D3D12 CX0／CX2／CX3 | MVP production sourceをCX0 self-contained Headerへ固定し、CX1 fixture、CX2 cutover、CX3 Shippingを分離した | [C++23 Modules](../architecture/02-foundation/cpp23-modules.md)、[D3D12 Design](../plans/2026-07-22-ai-readable-d3d12-backend-design.md)、[D3D12 Implementation Plan](../plans/2026-07-22-d3d12-backend-implementation-plan.md)、commit `7f352d8`、`d011950` | CX0の実装計画だけ受理。CX2／CX3は`deferred`、ReleaseはNo-Go |
| `remediation.feasibility.official-evidence` | MCP、OpenAI、Ajv、CMake、MSVC | 外部仕様は公式一次資料で固定し、Miraikanai固有のTTL、Phase、Risk、maturityをVendor推奨と誤記しない境界を明文化した | [Remediation設計 §9](../superpowers/specs/2026-07-23-plan-feasibility-remediation-design.md#9-toolchainと公式根拠)、commit `f39832a` | 実装時にversion、hash、license、公式文書の再確認が必要 |
| `remediation.feasibility.critical-path` | Phase 0～9 Critical Path | Product Registryの全74 WPをCoverageへ保持し、71 executable WPをper-WP taskへmaterialize、3 deferred WPをDecision Gateまで非実行とし、依存、path、fixture、Receipt、rollback、相対sizeを閉じた | [Critical Path Execution Plan](../plans/2026-07-23-critical-path-execution-plan.md)、commit `c2ae9f3`、`1ece354` | 実装順序として受理。74 WPの完了やCapability成立を示すものではない |
| `remediation.feasibility.future` | 将来Capability | Open World、Online／MMO、large-session shooter、FPS、advanced physics／animation、AAA、Terrain／Foliage／GI、Console／Web／XR、Asset生成、first-party local inference、runtime generationを17件のinception dossierへ分離した | [Future Capability Inception Plan](../plans/2026-07-23-future-capability-inception-plan.md)、[Product Plan §8](../architecture/00-product/product-plan.md#8-future-portfolio)、commit `c2ae9f3`、`1ece354` | 全17件`planning_only`。実現性、納期、製品品質、Activationを保証しない |

## 公式一次資料との整合

| external boundary | adopted planning constraint | official source |
|---|---|---|
| MCP Host／Transport | MCP `2025-11-25`をbaselineとし、local STDIOとStreamable HTTPを別Profileとして扱う | <https://modelcontextprotocol.io/specification/2025-11-25>、<https://learn.chatgpt.com/docs/extend/mcp> |
| OpenAI model | direct Providerのexplicit既定`gpt-5.6-sol`はdated snapshotではない。non-snapshot IDとしてProfileへ記録し、解決されたmodel ID、Eval、expiry、Conformance Receiptを同じrevisionへ束縛して変更を管理する | <https://developers.openai.com/api/docs/guides/upgrading-to-gpt-5p6-sol> |
| JSON Schema | Draft 2020-12をAjv 8の2020 entry point、strict、offline local `$ref` allowlistで検証する | <https://ajv.js.org/json-schema.html#draft-2020-12> |
| C++ Modules | CMake 4.4の`import std`制約をCX1へ隔離し、MVPはCX0 Headerを使う | <https://cmake.org/cmake/help/v4.4/manual/cmake-cxxmodules.7.html> |
| MSVC C++23 | MSVC 14.51のpreview modeを正式C++23 Shippingの証拠にしない | <https://devblogs.microsoft.com/cppblog/c23-support-in-msvc-build-tools-14-51/> |

外部一次資料は外部仕様だけを所有する。上表からMiraikanaiのCapability、Security、性能、Release品質が成立したとは推論しない。

## 残るBlockerと非保証範囲

1. **CX3 Shipping:** 正式`/std:c++23`、Experimental token不要のCMake経路、全TargetのBuild／Tooling／ABI／Package／Release Receiptが揃うまで`wp.foundation.cpp23-cx3-shipping`は`deferred`、Capabilityは`not_activated`である。CX0／CX1、MSVC 14.51 preview、Visual Studio Generator由来BMIを代用しない。
2. **C2 3D:** 第二の非Shooter 3D Fixture、C2 Requirement binding、Genre／Rendering／UI provider WP、Owner、Windows／Android／Apple Target closureを一つの承認済みProduct Decisionで登録するまで`wp.product.general-coverage-3d`は`deferred`である。現在の主張上限は`3D First Playable`である。
3. **Future 17件:** inception dossierは調査と設計開始条件を固定するだけで、Engine API、Work Package Activation、Target対応、性能、商用品質Asset、AAA／MMO規模の実現を保証しない。
4. **Local AI:** 外部Host／MCPがlocal modelを使うconformance経路は計画に含むが、first-party local inference runtimeは`planning_only`でありMVP blockerではない。Model名ごとのEngine分岐やsilent cloud fallbackは採用しない。

## Document／commit map

| change group | documents | commits |
|---|---|---|
| Remediation authority | [Design](../superpowers/specs/2026-07-23-plan-feasibility-remediation-design.md)、[Plan](../superpowers/plans/2026-07-23-plan-feasibility-remediation.md) | `f39832a`、`c2ae9f3`、`1ece354` |
| Product execution | [Product Plan](../architecture/00-product/product-plan.md)、[Shooter Domain Pack](../architecture/08-domain-packs/shooter.md) | `454501c`、`f2c7584`、`05dace5` |
| AI control | [AI Security／Approval](../architecture/01-governance/ai-security-approval.md)、[Core Architecture](../architecture/02-foundation/core-architecture.md)、[Executable Contracts](../architecture/02-foundation/executable-contracts.md)、[Editor Workspace](../architecture/03-authoring/editor-workspace-ux.md)、[Gameplay Programming Model](../architecture/03-authoring/gameplay-programming-model.md)、[Native Game Module](../architecture/03-authoring/native-game-module.md)、[Debug／Replay](../architecture/04-runtime/debugging-observability-replay.md) | `1d7d758`、`3a60e4e`、`e8303dc` |
| Control Plane | [Design](../plans/2026-07-22-architecture-evolution-control-plane-design.md)、[Implementation Plan](../plans/2026-07-22-architecture-evolution-control-plane-implementation-plan.md)、[AI Verification](../architecture/01-governance/ai-verification-provenance.md)、[Project State](../architecture/03-authoring/project-state.md)、[Performance／Capacity](../architecture/04-runtime/performance-capacity.md) | `192af5c`、`ee29d8d`、`4539f10`、`2519d33` |
| D3D12／Toolchain | [Toolchain](../architecture/02-foundation/toolchain-dependencies.md)、[C++23 Modules](../architecture/02-foundation/cpp23-modules.md)、[D3D12 Design](../plans/2026-07-22-ai-readable-d3d12-backend-design.md)、[D3D12 Implementation Plan](../plans/2026-07-22-d3d12-backend-implementation-plan.md) | `7f352d8`、`d011950` |
| Execution／Future | [Critical Path](../plans/2026-07-23-critical-path-execution-plan.md)、[Future Inception](../plans/2026-07-23-future-capability-inception-plan.md) | `c2ae9f3`、`1ece354` |

## Verification receipt

このReceiptは計画の内部整合だけを検査し、Engine実装の代替にしない。

| gate | expected invariant | result |
|---|---|---|
| Historical closure boundary | retained Finding 253件がterminal、未保存REFUTED 4件を再構成しない | `PASS` |
| Product registry | duplicate、missing ref、Phase↔WP不整合、DAG cycleが0 | `PASS` |
| Critical Path coverage | Product Registryと実行計画の74 WP／21 Gateがexact一致し、71 executable bindingはunique、3 deferred WPはDecision Gateまで非実行 | `PASS` |
| Future coverage | Product RegistryとFuture planの17 `future_capability_id`がexact一致し、全件`planning_only` | `PASS` |
| Activation boundary | 現在の有効なActivation Receiptは0件。materialization時の全行initial stateが`not_activated`で、文書変更からCapability／Releaseを自動昇格する規則がない | `PASS` |
| Independent semantic review | Control PlaneとExecution／Futureの最終再監査が各`Critical 0 / Important 0` | `PASS` |
| Markdown／patch | table列、local link path、`git diff --check`を最終候補で再実行 | `PASS` |

## Final disposition

- **Conditional Go:** Control Plane bootstrap、contract compiler、ECS E0以降、CX0 D3D12、Headless／Editor／2D／AI／3D／MobileをCritical Path順に実装し、各Gateでnegative fixtureとReceipt read-backを実行する。
- **No-Go:** 現時点のEngine Capability、Target対応、C2 3D、CX3 Shipping、Future 17件、Production／商用品質を利用可能または保証済みと表示する。
