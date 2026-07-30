# Miraikanai Engine Runtime Asset Lifecycle Contract

- 文書ID: mirakan.arch.runtime-asset-lifecycle
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Runtime Asset request identity、dependency closure、staging／activation、generation、residency、lease／release、eviction、retirement、memory pressure／device loss時の回復、共通failure／diagnostic／qualification境界
- 非正本範囲: Source／Import／Cook／Catalog／Content Package、Runtime Entry／World Package publication、共有allocation policy／budget値、Runtime phase、Texture／Mesh／Audio／Font／VFX等のDomain意味、Render／Audio execution、Save／Replay semantics、Platform package delivery、実装Task／順序
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Core Architecture](../02-foundation/core-architecture.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)、[Scheduling／Lifetime](scheduling-lifetime.md)、[Performance／Capacity](performance-capacity.md)
- 関連文書: [Runtime Package](runtime-package.md)、[Advanced Rendering／Multiplayer Ownership Decision](../decisions/2026-07-29-advanced-rendering-multiplayer-ownership.md)、[Debugging／Observability／Replay](debugging-observability-replay.md)、[Render Graph](../06-rendering/render-graph.md)、[Advanced Light Transport](../06-rendering/advanced-light-transport.md)、[Terrain／Foliage](../06-rendering/terrain-foliage.md)、[LOD](../06-rendering/lod.md)、[Virtualized／Continuous Geometry](../06-rendering/virtualized-continuous-geometry.md)、[World](../06-rendering/world.md)、[VFX Runtime](../06-rendering/vfx-runtime.md)、[Audio](../07-platform/audio.md)、[UI／Text／Localization／Accessibility](../07-platform/ui-text-localization-accessibility.md)、[Windows](../07-platform/windows.md)、[Mobile Common](../07-platform/mobile-common.md)
- 根拠区分: project-decision（未計測のcapacity、priority weight、timeout、resident量はprovisional）
- 外部根拠確認日: none

## 1. 結論と状態

本書を、Cook済みAssetをRuntime consumerが要求してから、検証済みgenerationを利用し、leaseを返却し、evict／retireするまでのinitial V1 Runtime Ownerとする。[Runtime Package](runtime-package.md)はRuntime Entry／World Root／Section package closureのstagingとpublicationを維持し、汎用Texture／Mesh／Audio／Font等のrequest／residencyを所有しない。[Asset Lifecycle](../03-authoring/asset-lifecycle.md)はSource、Import、Cook、Catalog、Content Packageの正本を維持し、Runtime lifetimeを所有しない。

全consumerは初期Contract Setから本書のexact Owner／Definition refを直接参照する。旧Manager、移管元Owner、source／target Definition、consumer migration、aliasまたはdual Registryをinitial V1へ定義しない。文書追加だけでSchema、Service、Runtime module、Capability、Operation、Fixture、ReceiptまたはTarget Qualificationが存在するとは扱わず、実装状態は`absent`である。

## 2. 不変条件

1. 一つのRuntime Asset request、generation、lease、failure codeには一つの正本Ownerだけを持つ。
2. Source identity、Cooked artifact identity、Catalog／Package generation、Runtime activation generationを相互に推測せず、exact refで閉じる。
3. dependency closure全体の検証前に一部Assetをactiveとして公開しない。
4. new generationのstaging失敗時はlast-valid generationと既存leaseを維持する。
5. consumerはRuntime Asset Ownerのprivate queue、cache、pointer、native resource handleまたはeviction listを書かない。
6. Runtime Asset OwnerはTexture、Mesh、Audio、Font、VFX、World等のDomain意味、quality selection、render order、voice priority、gameplay stateを所有しない。
7. package availability、disk上の存在、decode完了、GPU upload完了、active、resident、leasedを同義にしない。
8. priority、deadline、budget不足をsilent substitution、partial publication、別Assetへのlatest解決で隠さない。

## 3. Owner境界

| Owner | 正本責務 | 本書との境界 |
|---|---|---|
| Asset Lifecycle | Source、Import、Cook、Catalog、Content Package、promotion | exact Cooked artifact／Catalog／Package generationを供給する |
| Runtime Package | Runtime Entry、World Root／Section closure、branch staging／publication | package closureのconsumerとして必要Asset集合とactivation条件を提出する |
| Runtime Asset Lifecycle | request、dependency、staging、activation generation、residency、lease、eviction、retire | Domain非依存の共通lifecycleだけを所有する |
| Scheduling／Lifetime | apply boundary、job／message order、safe cancellation boundary | 本書の状態を実行するslotを提供し、Asset状態を再定義しない |
| Memory／Pointers | allocation domain、handle／borrow／lease規則 | 本書は割当器や共通pointer taxonomyを複写しない |
| Performance／Capacity | budget、measurement、backpressure predicate | 本書は測定済みでないbyte／latency／queue値を固定しない |
| Domain consumer | Texture／Mesh／Audio／Font／VFX等の意味、compatibility、fallback intent | 共通requestとleaseを消費し、独自residency authorityを作らない |
| Debug／Verification | bounded projection、Evidence、freshness、Qualification | live private queueやraw asset bytesを第二正本にしない |

## 4. Request identityとresolution

Runtime requestは少なくとも、要求対象のexact Cooked artifact ref、Catalog／Content Package generation、Target Profile、requesting owner／consumer、usage class、必要なcompleteness、optional priority／deadline intent、idempotency identityへ閉じる。path、display name、Asset type名、`latest`、現在residentな似たAsset、consumerのnative handleから対象を推測しない。

同じidempotency identityと同じsemantic requestは同じterminal resultへ収束する。同じidentityで対象、generation、Target、completeness、consumer scopeが異なるrequestはconflictとして拒否する。再試行は失敗原因とrequest identityを維持し、別Assetや新しいCatalog generationへ暗黙追従しない。

Requestの概念状態は、受理前、dependency解決中、private staging、activation待ち、active、cancelled、rejected、retiring、retiredを区別する。状態名からService、thread、APIまたは保存形式を生成しない。受理前またはprivate staging中のcancelは副作用を公開せず、activation commit後のcancelは既存leaseを破壊せず新規取得停止またはrelease requestとして扱う。cancelとdependency完了が競合した場合も一つのterminal resultだけを公開する。

## 5. Dependency closure

Runtime dependencyはCook時に確定したexact dependency refと、Runtimeで選択されたTarget／representation bindingから構成する。依存graphはcycle、missing ref、wrong Target、wrong Contract set、stale Catalog／Package generation、同一identity別hashを拒否する。consumer名、file extension、importer、Material名、Shader名、Audio cue名から未登録dependencyを補完しない。

一つのroot requestが複数dependencyを持つ場合、各dependencyをprivate stagingへ置き、rootが要求するcompletenessとDomain validatorを全て満たした後にだけ同じactivation generationへ束縛する。optional dependencyはSource／Cook契約でoptionalと明示され、意味保存fallbackが同じOwner closureへ登録されている場合だけ欠落を許可する。Runtimeの都合でrequired dependencyをoptionalへ降格しない。

## 6. Staging、activation、generation

I/O、range read、decrypt／decompress、decode／transcode、CPU preparation、GPU／device uploadは、各Domain workerとPlatform adapterが実行できるが、共通request identity、generation、cancel、failure、publicationは本書へ返す。各workerのprivate completionをRuntime activeと扱わない。

activationは、同じrequest closure、Catalog／Package generation、Target、Domain compatibility、dependency closure、device generationを再検証し、[Scheduling／Lifetime](scheduling-lifetime.md)のeligible boundaryで一回だけ公開する。Renderer、Audio、UI、VFX、World等が同じAssetについて別々のcurrent generationを公開してはならない。consumer固有のDerived device objectは同じRuntime Asset generationへ束縛し、device object generationをAsset logical identityへ戻さない。

hot reload、Project revision変更、recook、Package差替え、device recoveryはnew generationをprivate stagingする。new generationがactiveになるまでold generationを利用中のleaseはvalidであり、commit後もold leaseが0になるまでretireを待つ。new generationのfailure、cancel、timeout、budget拒否ではold generationを維持し、oldとnewのdependencyを混在させない。

## 7. Residency、lease、release、retirement

residentは検証済みRuntime representationが要求されたmemory classに存在する状態、activeはconsumerが選択できるpublication状態、leasedはconsumer lifetimeに束縛された利用権であり、それぞれ別である。active Assetが一部representationだけresidentである場合は、Source／Cook契約が定義したusable completenessとfallback chainを満たさなければならない。

leaseはAsset identity、activation generation、representation、consumer scope、Device generation、retirement条件へ束縛する。consumerはleaseから得たborrowをjob、frame、callbackまたはdevice generationの許容期間を越えて保持しない。releaseは冪等であり、別generationのlease、既にretiredなidentity、unknown consumerによるreleaseを成功として数えない。

strong root、session root、World／Runtime Entry root、UI-only root、audio deadline root、speculative prefetch等の保持理由を区別する。保持理由はeviction eligibilityへのinputであり、Gameplay、Render、Audioの優先度意味を本書へ移さない。pinの無期限化、consumerによるeviction list直接操作、lease数からAsset重要度を推測することを禁止する。

retirementは新規request受付停止、new lease停止、既存lease drain、device work完了、private resource解放、diagnostic closureの順序を満たす。強制解放が必要なfatal device／process failureではstale handleを無効化し、成功した通常retirement Receiptを捏造しない。

## 8. Priority、deadline、backpressure

priorityとdeadlineはconsumer intentであり、Runtime Asset Ownerが全consumer間のbounded arbitration、fairness、starvation防止、queue capacity、admission controlへ解決する。consumerは独自のglobal priority scaleやprivate emergency laneを定義しない。Audio deadline、visible Render need、World transition root、UI glyph／image、background prefetchを同じ意味へ潰さず、usage classとTarget policyを明示する。

capacity超過時は、未受理requestのreject、speculative requestのcancel、eligible unleased representationのeviction、quality Ownerが事前承認したfallback requestのいずれかをtypedに選ぶ。required root、leased generation、package publication中のdependencyをsilent evictionしない。Performance Ownerが発行したbudget／measurementがない場合、固定byte、queue depth、worker count、timeout、priority weightを本文から推測しない。

## 9. Partial representationとstreaming

mip、LOD、virtual page、audio range、font glyph page、World Section等の部分representationは、完全なAssetの破損状態ではなく、Ownerが宣言したindependently usable unitとcompleteness predicateを持つ場合だけactiveにできる。partial unitのidentity、dependency、generation、lease、eviction、fallbackはroot Assetと同じclosureへ接続する。

Virtualized Geometryのroot／page、LOD representation、Audio stream、Font／UI cache、VFX resourceは、それぞれのDomain Ownerがquality、deadline、semantic fallbackを決める。本書は共通residencyとpressure arbitrationを所有し、各Domainのselection algorithmまたは見た目／音の意味を再定義しない。

## 10. Failure atomicityと回復

| Failure | 正規結果 |
|---|---|
| unknown／stale artifact、Catalog／Package generation差 | requestをrejectし、current active generationを維持 |
| dependency cycle／missing／hash差 | root全体をpublishせず、private stagingを破棄または診断用に隔離 |
| decode／transcode／upload failure | partial representationをactiveにせず、last-validまたは明示fallbackを維持 |
| cancel before activation | stagingをretireし、source／active generationを不変にする |
| cancel after activation | committed resultを取消済みに書き換えず、release／future request停止へ解決 |
| memory pressure | policy上eligibleなunleased representationだけをevictし、required rootとleaseを維持 |
| device loss | logical Asset identityを維持し、旧Device generationのnative objectを無効化してprivate rehydrate |
| hot reload／Package replacement failure | old generationを維持し、old／new dependencyの混成を禁止 |
| consumer compatibility failure | incompatible consumerだけをrejectし、Assetを別Domain意味へ変換しない |
| observability gap | success／resident／evictedを推測せず、unknownまたはpartial evidenceとして表示 |

同じfailureについてconsumerごとに異なるlast-valid、cancel outcome、generation incrementを持たない。fatal invariant違反は該当Session／processのRuntime fault policyへ渡し、似たAsset、zero-filled data、bind pose、default Material、silent Audio dropを共通成功fallbackとして生成しない。

## 11. Consumer境界

| Consumer | Runtime Asset Ownerへ提出するもの | Consumerが所有するもの |
|---|---|---|
| Runtime Package／World | branch／section closure、required root、publication precondition | Runtime Entry／World／Sectionの意味とpublication |
| Render Graph／Materials／LOD | representation need、View／quality intent、device compatibility | render packet、material、visibility、LOD selection |
| Advanced Light Transport | acceleration／radiance／probe／model等のexact representation generation need | semantic channel、Technique、Target support、history／denoise intent、fallback |
| Terrain／Foliage | region／tile／cluster artifact dependency、World activation generation、representation need | Terrain／Foliage Source、identity、domain artifact、Cell binding、domain fallback |
| Virtualized Geometry | root／page dependency、pin／fault intent、fallback representation ref | geometry representation familyとpage meaning |
| Audio | stream／resident need、deadline intent、voice generation binding | cue、voice、mixer、audibility、audio fallback |
| UI／Text | image／font／glyph resource need、locale／style binding | layout、text shaping、focus、semantic accessibility |
| VFX | effect artifact dependency、instance generation binding | simulation、spawn、render semantics |
| Animation | clip／skeleton／sprite artifact need、pose generation binding | clock、pose、event、root motion、frame selection |
| Debug／AI | bounded query、diagnostic、causality ref | Evidence、explanation、redaction、authorization |

Worldless UI-only Runtime Entry、headless logic、Audio-only preload、multiple View、device recoveryでも同じrequest／generation／lease contractを使う。World loaderが存在しないことを理由に別Asset Managerを生成しない。

## 12. Save、Replay、live edit

SaveはRuntime residency、queue position、native handle、lease count、eviction orderを保存しない。Gameplay／World／UI等の正本Ownerが保存するsemantic Asset refは、Load時にexact compatible Cooked artifactとRuntime requestへ再解決する。Replayはrequest／activation／fallbackがGameplay結果へ影響する場合にOwner-typed causalityを記録できるが、physical memory address、worker timing、nondeterministic eviction順をsemantic Replay authorityにしない。

live editはnew Asset generationをstagingし、compatible consumerだけを明示boundaryで切り替える。incompatible Source／Cook／schema／Target変更はrecook、restart、migrationまたはtyped rejectへ解決する。Editor Preview成功をshipping Runtime activationまたはTarget Qualificationとして流用しない。

## 13. Observability、AI理解、Security

bounded Runtime Asset projectionはrequest identity、requesting owner、Artifact／Catalog／Package generation、dependency状態、staging／active／retiring状態、lease countの集約、memory class別charge、priority／deadline class、selected fallback、typed Diagnostic、causal predecessorを返す。raw bytes、filesystem path、credential、private pointer、native descriptor、他ProjectのAsset identityを公開しない。

AIはprojectionからmissing dependency、stale generation、pressure、fallback理由を説明し、Owner-typed remediationを提案できるが、requestを直接activateし、priorityを昇格し、evictionを強制し、missing Evidenceを補完しない。read／explain／propose／executeの認可はAI Security、Evidence freshnessとQualificationはVerification Ownerが決める。

RuntimeはCook済みCatalog／Package／Artifactのintegrity、size／range、signature／promotion状態を再検証し、untrusted path、offset、compressed size、dependency countを信頼しない。Source byte validation、license、malware／format policyはAsset Lifecycle、credentialとnetwork authorizationはAI Security／Platform、共通memory safetyはMemory Ownerへ委譲する。

## 14. Qualification

設計上のQualification分類は次を分離する。

- request identity、idempotency、dependency DAG、cancel race、generation、lease／releaseのcontract test。
- missing／stale／wrong Target／wrong Package／dependency failure／partial stagingのnegative test。
- Worldless UI、World／Section transition、Audio deadline、multi-View LOD、VFX、Font／glyph、hot reloadのconsumer integration。
- memory pressure、starvation、backpressure、eviction、last-valid、old lease drainのstress／soak。
- device loss、surface recreation、GPU reupload、mobile lifecycle、thermal／memory warningのTarget session。
- clean package／install／offline launchからのArtifact resolutionと、別Target／別Candidate Evidence非代替。
- Debug projection、redaction、causality、AI説明のcompletenessとauthorization denial。

Test pass、simulator、Editor Preview、単一consumer、平均latency、warm cacheだけで全Targetをqualifiedにしない。absolute threshold、Hardware Profile、queue capacity、resident byte、deadlineはPerformance／Platformのfresh Evidenceへ束縛し、未materialize値を本書から生成しない。

## 15. Initial V1 design closure

本書のinitial V1 design closureは、次が文書上相互参照できる状態を指す。

1. 全Runtime Asset consumerが本書を唯一のrequest／generation／residency authorityとして参照する。
2. Asset Lifecycle、Runtime Package、Scheduling、Memory、Performanceとの正本／非正本境界が一致する。
3. dependency、cancel、activation、lease、eviction、device loss、hot reloadのfailure atomicityが同じ意味へ解決する。
4. Virtualized Geometryを含む未解決Runtime Asset refが本書へ解決する。
5. 全consumerのOwner／Definition ref、negative scenario、Evidence requirementが同じinitial V1 Architecture revisionへ閉じ、duplicate Owner、近似名fallbackまたは旧aliasが0件である。

このclosureは実装、Service、Schema、Artifact、Fixture、Receipt、Capability ActivationまたはProduct completionを意味しない。

## 16. 明示的に採用しないもの

次を禁止する。

- Asset LifecycleへRuntime residencyを戻す。
- Runtime Package／World loaderを汎用Asset Managerとして暗黙拡張する。
- Renderer、Audio、UI、VFX、LODごとに相互非互換のrequest／cache authorityを作る。
- filesystem path、display name、`latest`、native handleをRuntime Asset identityにする。
- package存在、decode完了、resident、active、leased、qualifiedを同義にする。
- memory pressureでleased／required rootをsilent evictionする。
- missing Assetを似たAsset、default resource、zero dataへsilent substitutionする。
- 文書状態からRuntime module、API、実装Task、実装順、工数、日程を生成する。
