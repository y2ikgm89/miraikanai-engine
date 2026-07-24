# Miraikanai Engine 計画実現可能性 Review（2026-07-23、current revision）

- scope: Architecture正本、Product／Future definition、AI境界、Control Plane、Critical Path、ECS／D3D12実装計画。
- excluded: Engine runtime実装、Build、実機Qualification、Capability Activation、Release／Shipping実行。
- decision: **planning_go / implementation_and_shipping_no_go**。
- authority: [Product Plan](../architecture/00-product/product-plan.md)、[Control Plane Design](../plans/2026-07-22-architecture-evolution-control-plane-design.md)、[Critical Path](../plans/2026-07-23-critical-path-execution-plan.md)。本Reviewは正本を上書きしない。

## 1. 判定

計画書は、Control Planeを最初に実装し、その後にWork PackageをEvidence付きで進める入力としてはGoである。主な理由は次である。

- Active Product Definition、Future Portfolio、operational stateが別hash domain／署名chainへ分離されている。
- Engine本体とGame Project Sourceを分離し、Project内C++／Shader生成を`Source Task→Staging Proposal→pre-promotion Build／Candidate Test→Independent Review→Code Owner Approval→trusted-internal Promotion→Project ChangeSet登録→promotion後Game Candidate／Package`の一方向DAGへ閉じている。
- AIをModel family別にhard-codeせず、Host／Transport／Provider／Deployment／Model／Authority／Freshnessのsigned profile tupleで扱う。
- hosted ChatGPT Workのplugin remote MCPと、ChatGPT desktop appのCodex hostが使うlocal STDIO／Streamable HTTPを別Host Profileにし、Codex CLI／IDE、Claude、Cursorを含め製品名だけで対応済みにせずsurface／version／binary／TransportごとのConformanceを要求する。設定、権限、Receiptをsurface間で流用しない。
- local inferenceは実現可能なadapter境界を持つが、first-party runtimeはFuture `planning_only`、current materialized instance 0件で、MVP blockerにしていない。
- Open World、MMO、large-session network、AAA、Terrain／Foliage／GI、Console／Web／XR等をplanning-only Futureへ残し、Active MVPを過大表示しない。

ただし実装とShippingはNo-Goである。現変更は計画文書であり、Engine binary、Schema generator、Gateway、署名Service、Build、device test、Activation／Release Receiptを生成していない。

## 2. Current exact closure

| Domain | Current invariant |
|---|---|
| Active Product Definition Registry | 14件 |
| Product Target | 5件 |
| Requirement | 29件 |
| Fixture | 15件 |
| Fallback | 6件 |
| Phase | 10件 |
| Phase Gate | 22件 |
| Release Gate | 5件 |
| Decision Gate | 4件 |
| Work Package | 75件 |
| Capability | 102件 |
| Activation binding | 293件 |
| Risk | 9件 |
| Future Registry | 3件 |
| Future capability | 25件 |
| Future claim | 57件 |
| Future policy | 2件 |
| Contract-active Operation | 10件（Generic Core baseline 6＋installed Core extension World 1＋Genre Pack Shooter 3） |
| Operational Operation | 0件（current Signer Policy `entries=[]`） |

Critical Path coverageは`bootstrap completed 1 + unconditional 70 + conditional deferred 4 = 75`のdisjoint unionである。conditional exact setはCX2、CX3、General Production 3D、Production Release Bindingである。

genesisではActivation 293行すべて`not_activated`、critical Riskはopen、conditional 4 WPはdeferredである。したがってProduction／Shippingは明示的にNo-Goである。

## 3. AI-native feasibility

### 可能にする設計

設計上、AIはMCDとbounded QueryでEngine／Project／Capability／System／Worldを検索し、typed plan／ChangeSetをStagingへ提出し、許可されたBuild／Test／Play／Debug Operationを実行できる。ただし実行可能になるのは、対象Host materialization、Operation familyのatomic Activation、Signer／Trust、Target／Tool conformance、Qualificationが同じclosureで成立した後だけであり、current callable集合は0件である。全routeのAttemptはsigned Reservation／task-scoped current Headで直列化する。Engine Provider／attested managed Hostの生成結果はGeneration Receiptへ、標準外部MCPから受領したTool callはOperation Receipt、Proposalは上流Model attributionなしの`StandardExternalProposalReceiptV1`へ、検証結果はQualification Receiptへ束縛され、人間はDiff、Evidence、Riskを確認してApproval／Commitを行う。

外部Clientは次の三routeへ分離される。

| Route | 設計上のauthority ceiling | current materialized／callable | Activation |
|---|---|---|---|
| standard external MCP | query／proposal | `0件／false` | Phase 5でexact external Host＋MCP Transport＋Tool＋proposal-only Authority＋MCP Grant＋Host/Transport・Schema/Eval Conformance。Provider／Deployment／managed deployment identity／Modelはnull／unattested |
| Engine Provider Adapter | query／proposal | `0件／false` | Phase 4でexact first-party Engine Host＋Provider Runtime／Manifest＋Inference Deployment＋Model＋Tool＋Authority＋Provider-Tool・Schema/Eval Conformance。managed deployment identity／MCP Transport／Grantはnull |
| managed external Host | 将来managed source edit／build | `0件／false` | `future.capability.managed-external-host-execution`でexact external Host＋MCP Transport＋Provider Runtime＋managed deployment identity＋Model＋Tool＋Authority＋Host/Transport・Provider-Tool・Schema/Evalの三Conformance＋pre/post Host Attestation＋Engine Build Receiptを閉じる。現状`planning_only` |

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

長期的な自由度を確保する要点は、Engine CoreをShooter class hierarchyへ固定せず、Reusable Feature Pack、optional Genre Pack、Game Project／Reference Game、data-driven contract、Project Source module、Target adapter、provider／fallbackへ分けることである。有名Engineと同様にProject側native codeを許すが、Engine内部ABIやprivate APIへ直接依存させない。

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
8. **Future 25:** Dossierやprototypeは実現保証ではなく、Decision→Rebaseline→full reset→implementation→production Activation→claim releaseが必要。
9. **Generic contract implementation:** Review 1／2で修正したclosed Contract Set compiler、intent／authorization／request DAG、private-to-public publication、World allocation、owner-typed Qualification分離は文書契約であり、Schema／Registry compiler、Gateway、signing Store、Runtime、Save／Replay、negative fixtureを後続WPで実装する必要がある。

## 7. Verification status

本Reviewで主張できるのは文書内部のclosureだけである。

| Check | Current status |
|---|---|
| Product references／DAG／duplicates／orphan | document audit pass |
| Product exact counts／Activation reachability | document audit pass |
| Active／Future／state separation | document audit pass |
| AI profile／Host／local inference contracts | document audit pass |
| Control Plane design consistency | independent document auditとcurrent全scope再Reviewを完了。残存Critical 0、Important 0、Design／Implementation scoped sync差0 |
| Task 2 Review 1／2 contract remediation | checkpoint commits `efee59f`／`bb49a46`／`76d1aa0`、Foundation correction、全scope audit、独立read-only再Reviewへ反映済み |
| Markdown links／anchors／tables／fences／encoding／patch whitespace | current repository-wide 71 Markdown／545 GFM table／6,135 all rows／5,045 body rows／1,122 fence／1,572 relative link／220 anchor reference audit pass |
| Engine build／tests／device qualification | not run; implementation absent |
| Capability／Release Receipt | 0 current implementation receipts |

旧baseline rangeの最終機械ReceiptはMarkdown 71件、GFM表518件、link 1,773件、local link 1,494件、Product WP DAG 75/75、Future DAG 23/23、Activation 273行、Content ID 32件、追加Control Plane契約30型を検査した。これは下記固定rangeの履歴Evidenceであり、上記current exact closure 293行の検証結果として流用しない。current treeは本更新完了時に別のfresh機械Receiptを発行する。encoding／fence／local link／anchor／table alignment／ID duplicate／DAG／scope／Owner／Design–Implementation集合／旧仕様語／patch whitespaceの問題は0件であった。

### 7.1 Task 2 Review 1／2 remediation（2026-07-24）

対象rangeは`0386626a95f1534389cb4f2a6ec71ff254d06b68..0e08066905a9719664a0e97d33b4aad976a72c26`である。Independent Review 1 `Critical 9／Important 5／Minor 2`、Independent Review 2 `Critical 11／Important 11／Minor 3`とcontroller追加要件を統合し、次をArchitecture正本へ反映した。

- 認可はreceipt-free `operation_intent_hash`を先に作り、Approval／Predelegationを同hashへbindし、binding確定後にfinal `request_hash`を作る。循環形V2は未Activationであったため互換readerを作らない。
- Contract Set rootはMCDだけでなくDiagnostic、Trusted Service、Validator、Operation Validator Closureの全normative local memberをhashする。
- public stateの発行は[Executable Contracts §8](../architecture/02-foundation/executable-contracts.md#8-operation定義)を唯一の現行正本とし、`private durable marker read-back → secret-free PublicCommitClosureV1 candidate → canonical MirakanSignedRecordV1 wrapper → PublicCommitClosureV1＋PublicPublicationMarkerV1＋after stateのatomic public CAS`だけを許可する。Closureなしのpublic authority、Domain独自inline signature、unsigned payload authority、alternate signature、public後rollbackを許可しない。
- Production recordからFixture body refを除き、別owner-typed Qualification recordのexact signed Receiptだけをconsumeする。`owner_layer`とexact owner refでCore／Feature／Genre／Project dependency legalityを検証する。
- World Stable-ID allocationは完全登録したinternal Gateway Operationへ、Scenarioはreceipt-free candidate＋三owner／五gate Receipt DAGとgeneric T00 deliveryへ、Shooterはexact八Validator inventoryと三Operation各17 Diagnostic closureへ閉じた。
- 完全登録できないScenario／Stage authoring、generic Pack AI、Feature authoring、Physics／Navigation／Input selectionはcurrent Manifest／Catalog／allowlistから除外し`not_activated`とした。旧name-only IDをcurrent aliasとして読まない。
- Executable Contracts §§20～21.1の191候補（Discovery／Execution 67件＋既存Domainから回収したauthoring／selection 92件＋AI制作E2E carrier 32件）は24 `PlannedOperationFamilyV1`へ閉じ、全current MCD／Manifest／Service／Policy／Validator／Diagnostic／Receipt／Provider／MCP Tool／alias集合を空、stateを`not_activated`とした。current contract-active 10（Generic Core baseline六＋installed Core extension World一＋Genre Pack Shooter三、operational 0）、conditional legacy migration四、example／pending／rejected十一は互いにdisjointで、family単位またはsigned legacy evidence gate以外の部分Activationを許可しない。

checkpoint 3のfresh assertionsは`19/19 PASS`、対象Scenario／Packのconcrete name-only current Operationは`0`、Shooterのunexpected Operation familyは`0`、対象五文書のunbalanced fenceは`0`、`git diff --check`は`PASS`である。これは文書契約の検査結果であり、Engine実装、Capability Activation、Target Qualification、Release／Shipping Receiptではない。このcheckpoint単独ではTask 2 final closureを主張せず、後続する全scope auditと独立read-only再Reviewの結果を§7.3へ統合する。

### 7.2 Foundation correction（2026-07-24）

Review 1／2後に残っていた参照先不在とdefinition-root循環を、次のexact差分で閉じた。

- current MCD追加は`type.operation.mutation_authorization_binding@2`、`capability.runtime.scheduling@1`、`capability.authoring.command_gateway@1`、`profile.isolation.authoring_command_gateway@1`のexact四recordで、差分は`Type +1、Capability +2、Profile +1`である。`capability.authoring.offline_migration@1`、`profile.isolation.offline_project_migrator@1`、対応Serviceのcurrent集合はexact `[]`で、実migration Operationとのatomic activationまで先行登録しない。
- current active Operation closureはOperation 10、direct reachable Type 27、Policy 23、active-Operation reachable Diagnostic 32、active-Operation reachable Validator 27、Operation Validator Closure 10、Trusted Service 1である。current Diagnostic RegistryはProject load／target selection二件とRuntime Scope Catalog三件のnon-operation Diagnosticを加えたexact 37で、reachable 32と混同しない。`diagnostic.operation.task-not-cancellable`はplanned `operation.task.cancel`専用reserved tokenで、current Diagnostic／Service／Validator／Operationへactivateしていない。
- Owner identityは既存Diagnostic owner hash bytesを維持した`OwnerIdentityLocalRefV1`へ統一し、Core 11、Feature 5、Genre 1のexact 17 active rowを`OwnerIdentityRegistryV1`へ登録した。Feature内訳にはowner-qualified Interaction Space Semantic Registryを所有する`owner.feature.interaction`を含む。旧称`owner.core.ai_security`、`owner.core.runtime_scheduling`、`owner.core.ui_text_accessibility`を追加Ownerとして保存しない。同じID／revisionは全retained Registryでbyte-identical、layer／authority／status差はOwner revision exact `N+1`、row差はRegistry version exact `N+1`とする一般規則を追加した。
- Request hashは式を変更せず、exact一recordの`NamedAlgorithmRegistryV1`へ機械解釈を固定した。recordはExecutable Contracts正本の完成`McdCanonicalBinaryV1` profile/version/hashをinline束縛し、bare serializer名を許可しない。`OperationRequestAlgorithmBindingV1`のversion FieldはClosure-selected positive uint32、current初期値をexact v2とし、将来v3でBinding schemaを変えない。同じAlgorithm ID／versionのcross-Registry再定義を禁止し、bindingを状態変更input、materialization Context、Prepared Receiptへ保存する。Contract Set、Owner Registry、Named Algorithm Registryの三rootだけを`FoundationDefinitionClosureV1`へ閉じ、Owner／Algorithm rootを`ContractSetSnapshotV2` member preimageへ戻さない。
- `MutationAuthorizationBindingV2`は裸refを廃止し、requester、Project descendant／typed subject scopeを持つ`TaskAuthorizationEnvelopeRefV1`、exact intentとquorumを閉じる`OperationMutationApprovalSetRefV1 | OperationPredelegationUseRefV1`、Closure-bound Algorithm bindingへ固定した。R2のcurrent Projector bindingはexact空で、Activationまでdirect Approval Setだけを許す。Grant Wrapper Registry、CSPRNG nonce、累積Consumption Head CASを必須にし、個別Approval、Grant単体、別purpose、scopeだけ一致する別intent、final request hashへの署名循環を拒否する。
- Approval／Grant／Useの三signed wrapperはAI Security Ownerのexact root schema ID、runtime-instance signature slot、固定transport signing Role、`AuthorityQualificationRoleRefV1 {role_ref,role_entry_sha256}`、Core baseline qualification-only exact六Role、subject／scope derivationへ分離した。full Local／Authority Catalogと全fixed transport RoleはTask 0 genesisで一度だけProvisionし、Task 2はschema bytes／verifierを既存Catalogへ一致させ、Engine route ActivationはCatalog／slot／fixed Roleをread-backする。Engine Activationが未登録なら一括追加できるglobal rowはqualification-only六Roleだけで、Project／Task onboardingではactual Identity／qualification Assignment／別transport Assignment／Keyを0～N追加する。Signer Policyはcompleted object hashとself-excluding semantic hashを分離し、Receipt ContextがTrust Head／closure、Governance Policy Config row、Signer Policy、Key／revocationを発行時にpinする。
- `PublishedReceiptMaterializationPolicyV1`をMCDまたは同PolicyがpinするGovernance Registryへ登録するとContract Set／TrustまたはRegistryの自己循環になるため棄却した。Executable Contracts Ownerのunsigned Local Schema Catalog root exact三件として、fixed-logical tagged control policy、per-receipt content-addressed Materialization Policy、Authorization Audit BindingをTask 2へ追加した。Governance Policy ConfigのReceipt required subsetはSigner Policy一row＋Verification Retention／Deterministic Recovery／Store Namespace三rowのexact四row、Trust Registry種類は六、closure memberは15、Contract Set／MCD件数は不変である。Contextはfinal requestの`MutationAuthorizationBindingV2`とbyte equalityなAudit Bindingをrequired保持し、Public Markerからref／hash commitmentまで公開、PII bodyはimmutable access-controlled CASへ分離する。これによりReceiptから認可根拠を監査でき、requestへContextを戻す固定点を作らない。
- Root Scene、Runtime Scope、Performance Scale、Physics Intent Roleの四migrationは、旧schema bytes、source Contract Set／Owner／Named Algorithm／Foundation Closure、decoder、immutable fixtureを証明するsigned `LegacyMigrationInventoryV1`が存在しないためconditional planning state=`not_activated`へ移した。current MCD／Manifest／Service／Policy／Validator／migration専用Diagnostic／Receipt／Provider／MCP／alias／Signer／qualification projectionはexact `[]`である。destination templateだけがrequired `source_foundation_definition_closure_ref`／`retained_source_mcd_ref`とInput–Contribution–Prepared Receipt byte equalityを持ち、実Inventory成立後のatomic activation fixtureで旧別名、ambient retained latest、wrong root、destination downgradeを拒否する。
- Offline Project MigratorのService／Capability／Profile／Trust／Signer current集合はexact `[]`である。四legacy候補の実Inventory gate、または将来の汎用registered schema migration Operationを完全登録する同一transactionだけが、network／process／Host path／Project Sourceを拒否した別process Serviceとして実allowlistと共に初めてmaterializeできる。空allowlist、Profileだけの先行登録、未登録Gateway applyという第二経路を拒否する。
- Capability failure modeのPolicy rejection tokenとProduct fallback foreign keyを区別し、Baseline／RebaselineでFallback六rowの三Field`ProductDiagnosticRefV1`集合とOwner／severity／category／message keyまで固定したProduct Diagnostic六recordをset equalityにする。`fallback.capability.unavailable`は`preserves_semantics=false`かつ`diagnostic.product.capability-unavailable@1`へexact解決し、Product-owned refをFoundation hash preimageへ戻さない。
- Trust authorityは既存六Registryとexact 15-member Trust closureを維持した。genesis Registry bytesからcurrent Trust Head chainの承認済みrow差分をforward replayし、六Registryをkind別authorized stateとset equalityにする。Catalog fixed Roleはrequired subset、qualification-only／offline-recovery Role、runtime Identity／Assignment／Key、approved Revocation／Policyを別predicateで閉じる。後続Service追加はIdentity／Role／generic Assignment／Keyへ各一row、Revocation／Policyへ0 rowであり、Purpose RegistryまたはService Role Registryを新設しない。
- Baseline／Rebaseline CoreはFoundation closure ref／hashとTrust closure ref／hashを別々に必須化した。追加Control Plane関連contractは一型増のexact 30、`FoundationDefinitionClosureV1`はlogical IDを持たないためContent ID Registryはexact 32を維持する。

このcorrectionは計画Schema、Registry、hash DAG、件数の文書closureであり、Engine implementation、Contract compiler実行、Service／Key provisioning、Capability Activation、Qualification Receipt、Releaseを実行済みとはしない。correction単独の時点で未完だった全scope auditと独立read-only再Reviewは後続して実施し、そのfresh結果を§7.3へ記録する。

`PASS`をEngine動作、性能、安全性、商用品質の証明へ拡張しない。

### 7.3 Current tree fresh verification（2026-07-24）

全scope修正と独立再Review後のcurrent treeへ、正本表を機械抽出するfresh検証を実行した。結果は次である。

- Product RegistryはTarget 5、Requirement 29、Fixture 15、Fallback 6、Phase 10、Phase Gate 22、Release Gate 5、Decision Gate 4、Work Package 75、Capability 102、Risk 9、Future Capability 25、Future Claim 57で、duplicate／missing refは0件である。
- Capability Activationはrequired 265、optional 28、合計293である。WP DAGは75 node／171 edge／cycle 0、Future DAGは25 node／9 edge／cycle 0である。
- Future Claim Registry 57件、全Future Entryの`excluded_current_product_claims` union 57件はset equalityで、missing／extra 0件である。
- Operationはplanned family 24、candidate table 24、candidate 191、contract-active 10、conditional legacy migration 4、example／pending／rejected 11のdisjoint union 216件である。intersection、unclassified、unobserved、declared count差はすべて0件である。
- Architecture Dependency SourceはCapability 166、Work Package 171、Fixture 176、Installed Product composition 3、合計516である。Owner 103、edge classはproduction 248、qualification 193、installed 3、cross-cutting 72、documentation 0、runtime-required 125である。
- production graphは64 node／126 unique pair／cycle 0で、Generic CoreからShooterおよび全non-Core ownerへのproduction reachabilityは0件である。Generic Release Gate 5件からShooter／Genre／Reference Gameへの成立依存も0件である。
- 25件のV1型重複候補をField単位で比較し、canonical schema競合は0件である。正規Schema、具体instance、owner制約mirror、symbol inventoryを各記載箇所で明示した。
- fixed 60/1 Cadenceの整数partitionは、advance 1が16,666,666 ns、60 advance合計が1,000,000,000 nsで、16,666,666 nsが20件、16,666,667 nsが40件である。
- 構造検証はMarkdown 71件、fence 1,122、未閉鎖0、GFM表545件／all 6,135行／body 5,045行、列不一致0、relative link 1,572、anchor reference 220、missing 0、`git diff --check -- docs` error 0である。

これはcurrent計画書のsource-level closure、参照整合性、依存非循環性、汎用Core境界を検証した結果である。Engine binary、Schema／Contract compiler、Gateway、Runtime、Build、Test、device qualification、Capability Activation、Release／Shipping Receiptは生成していない。

## 8. Recommended execution order

1. Control Plane development bootstrapを実装・negative testする。
2. Foundation／Headless Authoringを完成する。
3. Windows Editor／D3D12／I/O／UI shellを完成する。
4. Manual 2D→AI MVP-A→external proposal lane→Project Source laneを閉じる。
5. Manual 3D→mobile→C2 2Dを閉じる。
6. CX2／CX3、Production 3D、Production ReleaseはDecision Gate成立後だけ開始する。
7. Future 25は価値／依存／riskで一件ずつ昇格し、MVPへ一括混入させない。

## 9. Final disposition

- **Planning Go:** 現行計画をGate付き実装作業の正本入力として使用できる。
- **Implementation No-Go:** 実装完了、Operation operational、Capability対応済みまたは品質保証済みとは表示できない。これはGate付き実装着手の禁止ではない。
- **Shipping No-Go:** Release Gate、critical Risk、Target production Activationが未成立。
- **Future No-Go:** Future 25を現在利用可能なCapabilityまたは保証済み品質として表示しない。
