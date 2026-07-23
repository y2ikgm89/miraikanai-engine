# Miraikanai Engine 計画実現可能性 Review（2026-07-23、current revision）

- scope: Architecture正本、Product／Future definition、AI境界、Control Plane、Critical Path、ECS／D3D12実装計画。
- excluded: Engine runtime実装、Build、実機Qualification、Capability Activation、Release／Shipping実行。
- decision: **planning_go / implementation_and_shipping_no_go**。
- authority: [Product Plan](../architecture/00-product/product-plan.md)、[Control Plane Design](../plans/2026-07-22-architecture-evolution-control-plane-design.md)、[Critical Path](../plans/2026-07-23-critical-path-execution-plan.md)。本Reviewは正本を上書きしない。

## 1. 判定

計画書は、Control Planeを最初に実装し、その後にWork PackageをEvidence付きで進める入力としてはGoである。主な理由は次である。

- Active Product Definition、Future Portfolio、operational stateが別hash domain／署名chainへ分離されている。
- Engine本体とGame Project Sourceを分離し、Project内C++／Shader生成をStaging→Build→Review→Code Owner Approval→Activationへ閉じている。
- AIをModel family別にhard-codeせず、Host／Transport／Provider／Deployment／Model／Authority／Freshnessのsigned profile tupleで扱う。
- ChatGPT／Codex／Claude／Cursor等は製品名だけで対応済みにせず、surface／version／binary／TransportごとのConformanceを要求する。
- local inferenceは実現可能なadapter境界を持つが、first-party runtimeはFutureかつ`not_activated`で、MVP blockerにしていない。
- Open World、MMO、large-session network、AAA、Terrain／Foliage／GI、Console／Web／XR等をplanning-only Futureへ残し、Active MVPを過大表示しない。

ただし実装とShippingはNo-Goである。現変更は計画文書であり、Engine binary、Schema generator、Gateway、署名Service、Build、device test、Activation／Release Receiptを生成していない。

## 2. Current exact closure

| Domain | Current invariant |
|---|---|
| Active Registry | 14件 |
| Product Target | 5件 |
| Requirement | 27件 |
| Fixture | 13件 |
| Fallback | 6件 |
| Phase | 10件 |
| Phase Gate | 20件 |
| Release Gate | 5件 |
| Decision Gate | 4件 |
| Work Package | 75件 |
| Capability | 95件 |
| Activation binding | 273件 |
| Risk | 9件 |
| Future Registry | 3件 |
| Future capability | 23件 |
| Future claim | 52件 |
| Future policy | 2件 |

Critical Path coverageは`bootstrap completed 1 + unconditional 70 + conditional deferred 4 = 75`のdisjoint unionである。conditional exact setはCX2、CX3、General Production 3D、Production Release Bindingである。

genesisではActivation 273行すべて`not_activated`、critical Riskはopen、conditional 4 WPはdeferredである。したがってProduction／Shippingは明示的にNo-Goである。

## 3. AI-native feasibility

### 可能にする設計

AIはMCDとbounded QueryでEngine／Project／Capability／System／Worldを検索し、typed plan／ChangeSetをStagingへ提出し、許可されたBuild／Test／Play／Debug Operationを実行できる。全routeのAttemptはsigned Reservation／task-scoped current Headで直列化する。Engine Provider／attested managed Hostの生成結果はGeneration Receiptへ、標準外部MCPから受領したTool callはOperation Receipt、Proposalは上流Model attributionなしの`StandardExternalProposalReceiptV1`へ、検証結果はQualification Receiptへ束縛され、人間はDiff、Evidence、Riskを確認してApproval／Commitを行う。

外部Clientは次の三routeへ分離される。

| Route | Current最大権限 | 条件 |
|---|---|---|
| standard external MCP | query／proposal | exact external Host＋MCP Transport＋Tool＋proposal-only Authority＋MCP Grant＋Host/Transport Conformance。Provider／Deployment／Modelはnull／unattested |
| Engine Provider Adapter | query／proposal | exact first-party Engine Host＋Provider Runtime／Manifest＋Inference Deployment＋Model＋Tool＋Authority＋Provider-Tool Conformance。MCP Transport／Grantはnull |
| managed external Host | 将来managed source edit／build | exact external Host＋MCP Transport＋Provider Runtime＋Model＋Tool＋Authority＋Host/Transport／Provider-Tool Conformance＋pre/post Host Attestation＋Engine Build Receipt＋専用Future WP。現状`not_activated` |

Approval、Commit、Activation、Signing、ReleaseはどのModel／Hostにも付与しない。

### local AI

local AI対応は推奨する。ただし「モデルを同梱する」ことと「外部Hostがlocal modelを使う」ことを分離する。

- 外部Host local model: standard MCP laneでは上流Modelを暗号学的にattestできないため、Modelはopaque／unattestedのままHost／Transport／Tool／proposal-only Authority／Grant Conformanceで利用する。Model固有保証が必要ならmanaged Host attestationまたはfirst-party local runtimeを使う。
- first-party local inference: Future。runtime／loader artifact、Model Manifest、license Decision、sandbox、OS IPCまたはauthenticated loopback、resource budget、Import／Schema／Tool conformance後にだけActiveへ移行する。
- llama.cpp等は候補Adapterであり固定採用ではない。選定時はexact artifactをpinし、built-in file／shell toolとMCP proxyを無効にしてMiraikanai Gatewayだけを使う。
- Gemma、Kimi、Qwen、DeepSeek、Grok、GLM等は同じprofile contractへ入り、family別Engine branchを作らない。

## 4. Game制作自由度

計画上の自由度は段階的である。

| Stage | 実現を狙う範囲 | 保証しない範囲 |
|---|---|---|
| MVP-A | AI＋手動の2D制作、Title→Result、Save／Load／Replay、Windows Editor／Desktop | Production、汎用3D、Shipping |
| MVP-B | Windows C1 manual 3D first playable | FPS、Production 3D、AAA |
| Mobile | Android／Appleの2D／3D lifecycleを各Target Gateで段階的に閉じる | Desktop合格の流用、Console／Web／XR |
| C2 2D | Shooter＋Platformer＋Puzzle／Dialogueのcross-genre coverage | 全genre、network、large world |
| Deferred C2 3D | 第二の非Shooter 3D fixtureを追加後のgeneral coverage | 条件成立前はShooter C1だけ |
| Future | FPS、Open World、network／MMO、advanced simulation、AAA、new platforms、scripting等 | Active migration／Target qualification前の製品claim |

長期的な自由度を確保する要点は、Engine CoreをShooter class hierarchyへ固定せず、Domain Pack、data-driven contract、Project Source module、Target adapter、provider／fallbackへ分けることである。有名Engineと同様にProject側native codeを許すが、Engine内部ABIやprivate APIへ直接依存させない。

## 5. Control Plane feasibility

初回ceremonyはBC0～BC2、Task 0～9（A）、独立Future Portfolio Approval、Construction Decision、Task 10（B）、Bootstrap Approval（C）、Task 10A（D）、Task 10B（E）の一方向である。Future Approvalはplanning-only別hash domainのcurrentだけをpublishし、Task 10Bより先にActive Product operational currentを作らない。

主な安全条件:

- offline Root、generation-local Root Revocation Head、cross-generation Global Revocation Ledger／Super Head、Recovery Readiness Head、Trust closure、Authority Binding Source Catalog Headを持つ。
- generation 1はout-of-band witness付きcandidate Root Profile／signed Root Head／local・global revocation genesis／candidate Readinessをclosed例外で検証し、4 current pointerのCAS後に通常検証を通すまでAuthorizationを発行しない。rotation／recoveryはRoot／local／global／Readiness／Trust／Catalogの6 pointer CASで、部分publicationを拒否する。
- total-compromise recoveryは旧generation全KeyをGlobal Ledgerへ追記し、新Root／recovery-custodian quorum、cooling-off、typed Incident、destination Trust closureを満たす場合だけ新generationへ移る。pointer巻戻しや旧Root単独署名を許さない。
- development bootstrapとproduction assuranceを分離する。1-of-1 development RootでProduction／Releaseを許可しない。
- vendor assuranceはfresh CRL／OCSP DER、RFC 10007のCRL issuer `keyUsage+cRLSign`、RFC 9919のSHA-256 CertID、RFC 9654 nonce、same-CA delegated responder、closed responder-revocation methodをtyped journalへ束縛する。first revoked Observationはterminalで、pending quarantine obligationと原子的に記録し、N／F／R／Xのexact oneへrouteする。
- Readiness quarantineは通常Change Approvalで迂回せず、原因修正のsame-generation三pointerまたはPolicy不変のassurance Rotation六pointerだけを専用Remediation Authorizationで解除する。Root／Recovery不成立時はfull trust resetへ進む。
- bootstrap時点のAuthorization／Approval expiryはhistorical validityとして検証し、後日の通常expiryでimmutable historyを破壊しない。
- same-definition変更はRebinding、Active Definition変更はRebaseline＋V1 full reset migrationを使う。
- Product operational currentはsingle-pointer CAS、Root／Trust系は設計で列挙したatomic multi-pointer CASを使う。同一candidateのcrash／idempotent retryはsigned payloadのevent time、candidate revocation binding、canonical bytes、signatureを固定する一方、各CAS attemptのjournal `publication_time`はfresh取得し、current authority／expiry／expected parentを直前再検証する。drift／window逸脱時はcandidateを変更せずterminal abortし、fresh Authorizationからnew candidateを作る。
- ReceiptはGit tree外sidecarで、Approval対象へ自己包含しない。
- bootstrap順序はA→current Future Approval F→Fを署名したConstruction Decision→B→C→D→Eであり、Fのrollback、DecisionのFuture closure／Approval差、Coreの旧F固定を拒否する。

この設計は実装可能だが、暗号／trust／recovery／transaction実装の難度は高い。まずdevelopment bootstrap profileで最小Control Planeを完成し、production assuranceは別Gateで昇格するのが現実的である。

## 6. Remaining blockers

1. **Control Plane implementation:** schema catalog、JCS／hash、signature／revocation、lifecycle、Product projection、CAS、negative fixtureが未実装。
2. **Toolchain unresolved dependencies:** C++ strict JSON、runtime JSON Schema validator、SHA-256、unit test framework、MCP server boundary等はexact selection前。
3. **CI／device capacity:** Windows GPU、Android／Apple device、macOS host等のqualified capacity／ownerが必要。
4. **CX2／CX3:** cross-target Evidenceと正式stable C++23 shipping toolchainが成立するまでdeferred。
5. **General Production 3D:** 第二の非Shooter 3D fixtureを含むdestination Active Definition migrationが必要。
6. **Production Release:** Target別support、signing、SBOM／provenance、rollback、device lab、critical Risk closureが必要。
7. **External Host support:** current materialized Host／version／Transport profileは0件。少なくとも一組をPhase 5でConformanceする。
8. **Future 23:** Dossierやprototypeは実現保証ではなく、Decision→Rebaseline→full reset→implementation→production Activation→claim releaseが必要。
9. **Generic contract implementation:** Review 1／2で修正したclosed Contract Set compiler、intent／authorization／request DAG、private-to-public publication、World allocation、owner-typed Qualification分離は文書契約であり、Schema／Registry compiler、Gateway、signing Store、Runtime、Save／Replay、negative fixtureを後続WPで実装する必要がある。

## 7. Verification status

本Reviewで主張できるのは文書内部のclosureだけである。

| Check | Current status |
|---|---|
| Product references／DAG／duplicates／orphan | document audit pass |
| Product exact counts／Activation reachability | document audit pass |
| Active／Future／state separation | document audit pass |
| AI profile／Host／local inference contracts | document audit pass |
| Control Plane design consistency | independent document audit pass: Critical 0、Important 0、Design／Implementation sync差0 |
| Task 2 Review 1／2 contract remediation | checkpoint commits `efee59f`／`bb49a46`／`76d1aa0`へ反映。全scope auditと独立read-only再Reviewはfinal closure前の必須gate |
| Markdown links／anchors／tables／fences／encoding／patch whitespace | repository-wide 67 Markdown audit pass |
| Engine build／tests／device qualification | not run; implementation absent |
| Capability／Release Receipt | 0 current implementation receipts |

最終機械ReceiptはMarkdown 67件、表448件、Product WP DAG 75/75、Future DAG 23/23、Activation 273行、Content ID 32件、追加Control Plane契約29型を検査した。encoding／fence／local link／anchor／table alignment／ID duplicate／DAG／scope／Owner／Design–Implementation集合／旧仕様語／patch whitespaceの問題は0件である。

### 7.1 Task 2 Review 1／2 remediation（2026-07-24）

対象rangeは`0386626a95f1534389cb4f2a6ec71ff254d06b68..0e08066905a9719664a0e97d33b4aad976a72c26`である。Independent Review 1 `Critical 9／Important 5／Minor 2`、Independent Review 2 `Critical 11／Important 11／Minor 3`とcontroller追加要件を統合し、次をArchitecture正本へ反映した。

- 認可はreceipt-free `operation_intent_hash`を先に作り、Approval／Predelegationを同hashへbindし、binding確定後にfinal `request_hash`を作る。循環形V2は未Activationであったため互換readerを作らない。
- Contract Set rootはMCDだけでなくDiagnostic、Trusted Service、Validator、Operation Validator Closureの全normative local memberをhashする。
- public stateはprivate durable marker、canonical `MirakanSignedRecordV1` wrapper、Public Markerの順でのみ可視化する。Domain独自inline signature、unsigned payload authority、alternate signature、public後rollbackを許可しない。
- Production recordからFixture body refを除き、別owner-typed Qualification recordのexact signed Receiptだけをconsumeする。`owner_layer`とexact owner refでCore／Feature／Genre／Project dependency legalityを検証する。
- World Stable-ID allocationは完全登録したinternal Gateway Operationへ、Scenarioはreceipt-free candidate＋三owner／五gate Receipt DAGとgeneric T00 deliveryへ、Shooterはexact八Validator inventoryと三Operation各17 Diagnostic closureへ閉じた。
- 完全登録できないScenario／Stage authoring、generic Pack AI、Feature authoring、Physics／Navigation／Input selectionはcurrent Manifest／Catalog／allowlistから除外し`not_activated`とした。旧name-only IDをcurrent aliasとして読まない。
- Executable Contracts §§20～21.1の159候補（当初Discovery／Execution 67件＋既存Domain文書から回収したauthoring／selection 92件）は18 `PlannedOperationFamilyV1`へ閉じ、全current MCD／Manifest／Service／Policy／Validator／Diagnostic／Receipt／Provider／MCP Tool／alias集合を空、stateを`not_activated`とした。current active 14（bootstrap八＋Domain六）との積集合は空で、family単位以外の部分Activationを許可しない。

checkpoint 3のfresh assertionsは`19/19 PASS`、対象Scenario／Packのconcrete name-only current Operationは`0`、Shooterのunexpected Operation familyは`0`、対象五文書のunbalanced fenceは`0`、`git diff --check`は`PASS`である。これは文書契約の検査結果であり、Engine実装、Capability Activation、Target Qualification、Release／Shipping Receiptではない。全scope auditと二つの独立read-only再Reviewが終わるまでTask 2 final closureを主張しない。

`PASS`をEngine動作、性能、安全性、商用品質の証明へ拡張しない。

## 8. Recommended execution order

1. Control Plane development bootstrapを実装・negative testする。
2. Foundation／Headless Authoringを完成する。
3. Windows Editor／D3D12／I/O／UI shellを完成する。
4. Manual 2D→AI MVP-A→external proposal lane→Project Source laneを閉じる。
5. Manual 3D→mobile→C2 2Dを閉じる。
6. CX2／CX3、Production 3D、Production ReleaseはDecision Gate成立後だけ開始する。
7. Future 23は価値／依存／riskで一件ずつ昇格し、MVPへ一括混入させない。

## 9. Final disposition

- **Planning Go:** 現行計画を実装開始の正本入力として使用できる。
- **Implementation No-Go:** 実行ReceiptがないためEngine機能を完成済みと表示しない。
- **Shipping No-Go:** Release Gate、critical Risk、Target production Activationが未成立。
- **Future No-Go:** Future 23を現在利用可能なCapabilityまたは保証済み品質として表示しない。
