# Miraikanai Engine Required Feature Coverage Closure Design

- 状態: approved
- 判断日: 2026-07-29
- 対象: C++ Game Engine必須機能監査から得たProduct Lifecycle、Product Security、再利用Scene composition、Architecture文書間整合性の設計閉包
- 非対象: Engine／Schema／Editor／CI／Testの実装、実装Task、実装順序、担当、工数、日程、Plugin ecosystem、汎用Script VM／JIT、Multiplayer、既存形式向け互換fallback
- 根拠入力: [External Review Summary](../../reviews/README.md)に要約した非規範review。採用済みclosureは[Architecture Plan Closure Review](../../architecture/appendices/architecture-plan-closure-review.md)から各canonical Ownerへ辿る

## 1. 結論

既存Architectureは、監査対象35分野の大半をOwner文書、明示的非目標、またはより狭く安全な意味置換として保持している。新しい汎用Subsystemを横並びに追加する必要はない。完全閉包には次の四変更だけを採用する。

1. `docs/architecture/00-product/product-lifecycle.md`を新設し、第三者DeveloperがProjectを作成してから更新、診断、support、製品releaseを利用するまでのProduct横断compositionとacceptanceを一意に所有させる。
2. `docs/architecture/01-governance/product-security.md`を新設し、Product全体の脅威責任、脆弱性報告、triage、修正release、開示、security incidentを一意に所有させる。
3. `docs/architecture/06-rendering/world.md`の既存`Scene Source`を、再利用Scene instance、nested composition、typed override、explicit rebaseを持つcleanなPrefab意味置換として閉じる。`Prefab`を正規ID、Schema名またはlegacy aliasにはしない。
4. Mobile Application lifecycleとWindows workspace rootの二つの文書矛盾を修正する。

Windows Previewの高速iterationは既存[Native Game Module](../../architecture/03-authoring/native-game-module.md)が、Shipping static link、Preview DLLのprocess起動時一回load、変更時`GameHost` restart、in-process unload／binary replacement／live patch禁止としてすでに閉じている。Hot Reload不足として新しい仕組みを追加しない。

`ArchitectureInventoryV1`、Explain Projection、Schema、Validator、Fixture、Operation、Receipt等がRepositoryに未materializeである事実は、Architecture Closure `ARCH-C03`のblockerとして残す。設計説明の完成を実装、ActivationまたはProduct利用可能性へ読み替えない。

## 2. 判断原則

### 2.1 完全閉包

完全閉包は、機能名を列挙することではなく、次を一意に説明できる状態である。

- canonical Ownerと非Owner
- typed input／outputとexact identity
- authorityとmutation境界
- state transitionとlast-known-good
- failure時の禁止fallback
- Target別Qualificationとnegative fixture
- Evidenceのproducer、consumer、freshness
- current design、materialization、qualification、activationの区別

外部Engineで一般的な名称があっても、MiraikanaiのOwner境界、AI authority、安全性、Product scopeへ合わなければ追加しない。

### 2.2 clean-break

未公開で未materializeの内部Schemaは、正しいV1を直接定義する。旧称alias、二重read、近似version lookup、暗黙migration、silent repairを追加しない。一方、将来一度公開したProject revision、Save、Package、Releaseは既存[Compatibility／Evolution](../../architecture/02-foundation/compatibility-evolution.md)のversioned migration、last-known-good、rollback規則へ従う。clean-breakをUser data破棄の許可にしない。

### 2.3 scope維持

現在のProduct scopeはWindows desktop C1、後続Android／Apple、C++23＋GameplayDefinition、Generic Core＋Feature Packs＋任意Genre Packs＋Projects、compact 2D command RPG Product Referenceである。Shooterはtechnical qualification consumerでありProduct MVPではない。

次は監査入力に現れても現在の必須要件へ昇格しない。

- Plugin marketplaceまたは任意native Plugin ecosystem
- 汎用Game Script VM、JIT、Game向けFFI
- Multiplayer、Account、Cloud、広告、課金
- large open world、virtualized geometryまたはhigh-end renderingをMVP前提にすること
- Runtime code generation

## 3. Owner境界

### 3.1 Product Lifecycle Owner

`mirakan.arch.product-lifecycle`は次だけを正本化する。

- Project create／bootstrapの製品composition
- Project TemplateとSample Projectの製品公開単位
- public API documentation、tutorial、snippet、sampleのrelease binding
- Editor GUI、CLI、headlessから同じtyped Operationへ到達するparity acceptance
- Engine／Project update candidate、repair、support window、NOTICE presentation
- Product Release Gateで必要なend-to-end acceptance

次は参照するだけで複写しない。

| 関心 | canonical Owner |
|---|---|
| `ProjectRevision`、`ProjectChangeSet`、atomic commit | Project State |
| Build／Cook／Package／Signingのdomain semantics | Core、Asset、Runtime Package、各Platform |
| dependency version、license、SBOM、third-party notice source | Toolchain／Dependencies |
| migration class、compatibility window、rollback rule | Compatibility／Evolution |
| `SupportBundleV1`のField、redaction、生成failure | Debugging／Observability／Replay |
| Evidence envelope、署名、retention、revocation | AI Verification／Provenance |
| Product intent、MVP、release／stop／completion Gate | Product Plan |

Product Lifecycle Ownerはdomain Operationを再定義せず、exact refを一つのProduct candidateへ束縛してE2E acceptanceを判定する。

### 3.2 Product Security Owner

`mirakan.arch.product-security`は次を正本化する。

- Product全体のthreat ownership registry
- security baselineとmemory-safetyを含むrisk treatmentの製品責任
- vulnerability report intake、triage、validation、remediation、release、disclosure、closure
- security update decision、affected／fixed release、notification
- product security incident、containment、recovery、recurrence prevention

次は参照するだけで複写しない。

| 関心 | canonical Owner |
|---|---|
| AI task authorization、approval tier、managed host authority | AI Security／Approval |
| dependency lock、license、SBOM generation | Toolchain／Dependencies |
| signing、Store、OS sandbox、privacy integration | 各Platform Owner |
| input validation、memory／handle／resource contract | 各domain Owner |
| Evidence envelope、signature、redactionの共通規則 | AI Verification／Provenance |
| support bundleの収集／redaction schema | Debugging／Observability／Replay |

報告者の文章、scanner severity、外部score、issue labelは入力Evidenceであって権威判断ではない。Product Security Ownerはdomain Ownerの技術判断を上書きせず、case全体のstateとrelease／disclosure closureを所有する。

### 3.3 World Owner

`mirakan.arch.rendering-world`は既存Scene Sourceの再利用compositionを次まで拡張する。

- exact Scene Source revisionを参照するinstance
- stable instance IDとstable source object ID
- acyclic nested composition
- source path／name／array indexに依存しないtyped override
- source revision更新に対するexplicit rebase
- Authoring lineageのCook／Package／Save／Replay／Debugへの追跡
- Runtime artifactがAuthoring Scene Sourceへ依存しないこと

Gameplay progression、spawn rule、Quest、Stage outcomeは各consumer Ownerのままにする。`Prefab`という第二のSource identity、`PrefabV1`、legacy alias、implicit import形式は作らない。

### 3.4 既存Ownerを維持する項目

- Windows Preview iteration: Native Game Module
- Application lifecycle: Scheduling／Lifetime
- presentation surface availability: Scheduling／Lifetime＋Mobile Common
- workspace root: Naming／Project Layout
- Build projection: Core Architecture＋CMake Presets
- Project update compatibility: Compatibility／Evolution
- Package update／signing: 各Platform Owner

## 4. Product Lifecycle contract

完全な正規Schemaと不変条件は [Product Lifecycle Architecture](../../architecture/00-product/product-lifecycle.md) だけが所有する。本節は設計判断の要約であり、recordを複製しない。

- ReleaseはProduct Definition、Toolchain closure、Target、Public Contract、Documentation、Support Windowへexactに閉じる。Securityは完成Releaseを一方向に消費し、Release側へ逆参照を作らない。
- BootstrapはFoundation Definition Closureを満たすTemplateから、GUI／CLI／headless共通のsemantic Operationで全成否のProjectを作る。partial Project、hidden default、fallbackを許さない。
- Documentationはversioned Entry、typed snippet／tutorial qualification、DAGのLink graph、Public Contractとのset equalityをRelease Gateに含める。
- Product updateはCompatibilityが所有するmigration／rollback eligibilityと、Lifecycleが所有するpublication／recovery orchestrationを分離し、同一Candidateのvalidation後に一回だけpromotionする。
- SBOM／NOTICEのpackage set equality、Support Window closure、通知Receipt、Release Acceptanceはversioned exact Refで追跡し、古いartifactや別releaseのEvidenceを流用しない。

## 5. Scene composition contract

完全な正規Schemaと不変条件は [World Architecture](../../architecture/06-rendering/world.md) だけが所有する。本節は設計判断の要約であり、recordを複製しない。

- `Scene Source`を唯一の再利用可能Source identityとし、Prefab schema、Prefab alias、第二のimport形式を作らない。
- Instance、attachment、override、source object、owner document、field contractをversioned exact Refで束縛する。
- Source更新は暗黙追従せず、before／after Source revision、before／result Override Set、各resolutionを持つversioned Rebase Changeを明示的にacceptする。
- Rebase時はinstance内のaccepted change refとchange側のinstance refが相互にexact一致し、source-only change、name／path／index repair、未解決conflictを拒否する。
- Cook後のRuntime Packageはauthoring Scene Source fileやEditor serviceへ依存せず、debug／save／replay用lineageだけをTarget artifactへ閉じる。

## 6. Product Security contract

完全な正規Schemaと不変条件は [Product Security Architecture](../../architecture/01-governance/product-security.md) だけが所有する。本節は設計判断の要約であり、recordを複製しない。

- Threat Ownership Registry、Baseline、Security Case Registry Snapshot、Release Bindingをtyped exact Refで閉じ、Release側との相互hash依存を作らない。
- canonical hashはRFC 8785 JCS前のtyped projection、domain separation、length framingを規定し、全semantic integerを範囲検査済みtagged decimal stringとして表現する。
- 製品severityはresponse policy classへexactに束縛し、scanner名、CVSS、報告者severity、issue priorityから暗黙変換しない。
- Update、Embargo、Disclosure、Incidentはclosed tagged branchで状態別必須Receipt／Evidenceを持つ。closed Incidentは少なくとも1件のconfirmed以降のcaseへ束縛する。
- Case、Decision、Embargo、Disclosure、Incident、Control Changeの相互参照はstrict-ancestor ruleでDAGにし、同一record参照、forward参照、duplicate cycleを拒否する。
- C++23採用は無条件に安全とも不適格ともせず、memory／pointer control、sanitizer、fuzz、static analysis、compiler hardening、dependency isolation、unsafe-boundary inventory、incident learningをbaseline化する。

## 7. failure contract

### 7.1 Product Lifecycle

- release、contract、Target、hash、license、qualificationのexact不一致はmutation前に拒否する。
- `latest`、同名、近いversion、別Target、別Sample、既定Templateへfallbackしない。
- bootstrap failureではcurrent `ProjectRevision`もEditorで開けるpartial Projectも作らない。
- update failureでは旧release／旧Projectをlast-known-goodとして維持する。new Project revision成功を旧Package成功で代用しない。
- public API undocumented、broken link／snippet、stale tutorial、unrunnable Sample、SBOM／NOTICE mismatchはProduct Release Gateを失敗させる。

### 7.2 Scene composition

cycle、exact ref／hash不一致、stable ID欠落、型不一致、削除target、rebase conflict、capacity超過をtyped diagnosticにする。partial commit、name／path／index／similar-field repair、source revisionの暗黙追従を禁止する。

### 7.3 Product Security

untrusted report textからcommand、path、severity、affected release、disclosure dateを自動決定しない。unknown impact、未検証duplicate、未release fix、未完了notification、stale inventoryでcaseをcloseしない。secret、credential、個人情報をadvisoryまたはsupport attachmentへ転記せず、redaction失敗時はpublicationを拒否する。

## 8. Qualification

### 8.1 Product Lifecycle qualification

- clean environmentでProject createから最初のatomic `ProjectRevision`まで
- Editor GUI／CLI／headlessのrequest meaning、result、diagnostic、candidate hash parity
- TemplateおよびSampleからcook、package、install／launch、offline play
- public API snippet、tutorial、link graph、Sampleのsame-release validation
- clean install、supported upgrade、failed upgrade、rollback、repair
- diagnosis、`SupportBundleV1` export、support window、data reset
- SBOM、NOTICE、license presentationのpackage exactnessとUser到達可能性

### 8.2 Scene composition qualification

- simple instance、nested instance、同一Sourceの複数instance
- typed override、source update、explicit rebase、conflict、object delete、field type change
- undo／redo、save／load、replay、cook、package、debug lineage
- crash recovery後のcandidate不変性
- Runtime PackageがAuthoring Scene Sourceなしで起動すること

### 8.3 Product Security qualification

- valid、invalid、malicious、duplicate report
- multi-release impact、dependency vulnerability、embargo、zero-day
- fix candidate、security update、withdraw／rollback、notification
- incident exerciseとreal-incident record
- disclosure correction／withdrawal、redaction failure
- stale SBOM、missing release inventory、unknown impactのnegative fixture

Qualification Receiptの形式、署名、retention、revocationはVerification Owner、domain-specific artifactは各Ownerが所有する。Product Lifecycle／Securityはsame-candidate、same-release、required set、freshnessとacceptanceだけを集約する。

## 9. 文書間整合修正

### 9.1 Mobile lifecycle

[Scheduling／Lifetime](../../architecture/04-runtime/scheduling-lifetime.md)のApplication stateは`Starting | Active | Inactive | Suspended | Terminating`、presentation stateは`absent | active | surface_unavailable`である。[Mobile Common](../../architecture/07-platform/mobile-common.md)からApplication stateの`SurfaceUnavailable`を除き、surface喪失をpresentation stateへ一意に投影する。Application lifecycleとpresentation availabilityを一つのenumに戻さない。

### 9.2 Windows root

[Naming／Project Layout](../../architecture/02-foundation/naming-project-layout.md)はlegacy `build/`を削除し、top-level closed setを`source/ config/ packages/ derived/ intermediate/ staging/ evidence/`としている。[Windows](../../architecture/07-platform/windows.md)の`Build／Cache | Project buildまたはUser cache`をこのroot契約へ合わせ、`build/`を再導入しない。

### 9.3 用語

- `Scene Source`、`SceneCompositionInstanceV1`を正規語にする。
- `Prefab`は外部Engine比較またはUser入力語をScene compositionへ解決する説明でのみ使用し、canonical ID、Schema、aliasにしない。
- `Hot Reload`はUser要求語をNative Game Moduleのrestart-based Preview iterationへ解決する説明に限定する。
- Plugin、Script VM、Multiplayerを現Product requirementへ追加しない。

## 10. Architecture正本へ反映する範囲

### 10.1 新設

- `docs/architecture/00-product/product-lifecycle.md`
- `docs/architecture/01-governance/product-security.md`
- `docs/architecture/decisions/2026-07-29-product-lifecycle-security-ownership.md`

### 10.2 更新

- `docs/architecture/README.md`
- `docs/architecture/00-product/product-plan.md`
- `docs/architecture/01-governance/architecture-governance.md`
- `docs/architecture/01-governance/ai-security-approval.md`
- `docs/architecture/02-foundation/core-architecture.md`
- `docs/architecture/02-foundation/executable-contracts.md`
- `docs/architecture/02-foundation/toolchain-dependencies.md`
- `docs/architecture/02-foundation/compatibility-evolution.md`
- `docs/architecture/03-authoring/project-state.md`
- `docs/architecture/03-authoring/editor-workspace-ux.md`
- `docs/architecture/03-authoring/native-game-module.md`
- `docs/architecture/04-runtime/debugging-observability-replay.md`
- `docs/architecture/06-rendering/world.md`
- `docs/architecture/07-platform/windows.md`
- `docs/architecture/07-platform/mobile-common.md`
- `docs/architecture/appendices/architecture-plan-closure-review.md`
- `docs/architecture/decisions/README.md`

既存Ownerへの更新はowner link、consumer／producer境界、qualification handoff、明白な矛盾修正だけに限定する。Product LifecycleまたはProduct SecurityのSchemaを複写しない。

## 11. 外部公式根拠

| 根拠 | 採用する意味 | 採用しない意味 |
|---|---|---|
| [CMake Presets 4.4](https://cmake.org/cmake/help/v4.4/manual/cmake-presets.7.html) | project／user presetとconfigure／build／test／package／workflow presetをBuild projectionに使える | CMake PresetをMiraikanaiのProject／Build authorityにする |
| [CMake C++ Modules 4.4](https://cmake.org/cmake/help/v4.4/manual/cmake-cxxmodules.7.html) | C++ module依存scanとtoolchain制約をcurrent exact-version設計比較に使う | module対応名だけでcross-target qualification済みとする |
| [Microsoft MSIX signing overview](https://learn.microsoft.com/en-us/windows/msix/package/signing-package-overview) | package identity、valid signature、device trust、bundle署名closure | 署名成功だけでProduct update／support closure済みとする |
| [Microsoft code signing options](https://learn.microsoft.com/en-us/windows/apps/package-and-deploy/code-signing-options) | Windows配布経路別のsigning選択をPlatform Ownerが検証する | Product Lifecycleが証明書／Store policyを所有する |
| [Microsoft package updates](https://learn.microsoft.com/en-us/windows/msix/app-package-updates) | identity、version、differential updateをWindows update qualificationへ入力する | MSIX挙動をcross-platform migration意味へ一般化する |
| [NIST SSDF publications](https://csrc.nist.gov/Projects/ssdf/publications) | SSDF 1.1 Finalをsecure development baseline比較に使う | 1.2 Draftを確定規範として固定する |
| [NIST SP 800-216](https://csrc.nist.gov/pubs/sp/800/216/final) | vulnerability report intake、assessment、management、remediation communicationの比較に使う | 米国連邦機関向け手順を製品policyへそのまま複写する |
| [CISA Product Security Bad Practices](https://www.cisa.gov/sites/default/files/2025-01/joint-guidance-product-security-bad-practices-508c_0.pdf) | Product全体のsecurity ownership、memory-safety roadmap、known exploited issueへの明示責任を設ける | C++を一律禁止する、またはvoluntary guidanceを法的義務とする |

外部根拠はMiraikanai Ownerの代わりではない。Version、確認日、対象範囲をToolchain／Platform／Governanceの各Ownerで保持し、external guidanceの更新だけでcurrent Project contractを暗黙変更しない。

## 12. 完了条件

この設計のArchitecture反映は、次をすべて満たした場合だけ完了とする。

1. 新Owner二文書と所有DecisionがArchitecture Indexから到達できる。
2. Owner文書数、文書ID、規範依存、相対linkが再監査され、重複、未解決、cycleが0件である。
3. Product Lifecycle、Product Security、Scene instance／override／rebaseのOwner、型、data flow、failure、qualificationが一意である。
4. Mobile lifecycleとWindows rootの矛盾が消えている。
5. Native Game Moduleのrestart-based Preview iterationをHot Reload不足として扱っていない。
6. `ARCH-C03`をunmaterialized blockerとして維持し、設計文書を実装済みと表現していない。
7. `ARCH-C21`はProduct Security Owner新設により`closed-in-target-design`へ更新されるが、implementation／operation／qualificationは`absent`のままである。
8. Plugin、汎用Script VM／JIT、Multiplayer、large-world機能をMVPへ追加していない。
9. Required-features監査文書のHot ReloadとScene／Prefab判定を正しいOwner意味へ訂正している。
10. Architecture差分に実装Task、実装順序、担当、工数、日程、互換fallbackが含まれない。

この設計はArchitecture正本の変更境界を承認する文書であり、実装計画ではない。
