# Miraikanai Engine Animation Contract

- 文書ID: mirakan.arch.simulation-animation
- 状態: review
- 正本範囲: Animation Source／Cooked Asset、typed graph、2D clip、3D skeleton／skin／clip、instance／pose ownership、event／root motion、IK、retarget、Animation memory／failure、Editor／AI operation、Animation qualification
- 非正本範囲: Runtime phase／tick／shared capacity、Physics motion resolution、Collision query semantics、Navigation artifact、Rendering skin execution、LOD共通intent、Asset transaction、external dependency version／build pin、AI authorization。各Owner文書を参照する
- 依存: [文書体系再編Decision](../decisions/2026-07-21-document-system-restructure.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[Collision](collision.md)、[Physics](physics.md)、[Navigation](navigation.md)
- 外部根拠検証日: 2026-07-21

## 1. 結論と所有境界

AnimationはEngine-owned Asset、Graph、Instance、Pose、Event、Root Motion proposal、IK、Retarget contractを公開し、sampling／compression Backendをprivate Adapterへ隔離する。Project C++、GameplayDefinition、AI、Editor、RenderingへVendor job、runtime object、pointer、archive formatを公開しない。dependencyのexact version／commit／license／build optionは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)だけが所有する。

Animationはauthoritative Physics Transformを書かず、Navigationを進めず、Collision queryを再定義しない。root motionは[Physics](physics.md)のCharacter Motorへのproposalであり、resolved motionを受けてposeを確定する。実行順とwriterは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)だけが所有する。

Module境界はContracts、Asset Cook、Graph Runtime、Pose／IK／Retarget Core、private Backend Adapter、Authoring、Editor Projection、Qualification Toolに分ける。Runtime Graphはimmutable Cooked Assetとinstance-local stateだけを読み、Source DocumentやEditor objectを参照しない。

## 2. Asset、Graph、2D／3D model

| Object | 所有内容 | lifetime |
|---|---|---|
| `AnimationClip2DSourceAsset` | typed sprite／property track、event、root／pivot semantics | Project source revision |
| `SkeletonSourceAsset` | canonical joint path、parent、bind pose、semantic bone | Project source revision |
| `SkinSourceAsset` | mesh ref、joint mapping、normalized influence source | Project source revision |
| `AnimationClipSourceAsset` | typed joint track、event、root track、compression intent | Project source revision |
| `RetargetProfileAsset` | source／target semantic mapping、scale／chain policy | Project source revision |
| `CookedAnimationArtifactV1` | immutable clip／skeleton／skin／graph payloadとruntime-ID table | Asset version lease |
| `AnimationGraphDefinition` | typed state、transition、blend、layer、parameter graph | Project source revision |
| `AnimationControllerComponent` | graph ref、parameter set、root-motion mode、retarget ref | World source |
| `AnimationInstanceState` | current state、transition、clock、loop、event cursor | Runtime instance |
| `AnimationParameterSnapshot` | gameplayからlatchしたtyped parameter | immutable tick input |
| `SkeletonPose` | local／model pose、bounds、socket projection | published pose snapshot |

Cookerはcanonical joint pathとSource Stable IDをartifact内の固定順へ並べ、non-zero `JointRuntimeId`／`ClipRuntimeId`を割り当てる。Runtime IDは対応artifact identityと組でだけ使い、Source、Save identity、別artifact比較へ使用しない。joint名、clip名、function名によるRuntime dispatchを禁止する。Event tag、parameter、semantic boneは[Executable contracts](../02-foundation/executable-contracts.md)のcontract refへ解決する。

### 2.1 Typed Animation Graph

Graph node familyはstate machine、clip sample、2D／1D blend、additive blend、layer／mask、pose cache、IK request、outputをclosed tagged unionで表す。transitionはsource／target Stable ID、typed condition、duration semantics、interrupt policy、event policyを持つ。任意script、function pointer、Vendor nodeをGraphへ埋め込まない。

Graph validationはreachable output、cycle policy、parameter type、clip／skeleton compatibility、mask joint、transition conflict、event duplication、root-motion source一意性を検査する。同priority transitionはStable IDのcanonical orderで決め、authoring insertion順やworker completion順を使わない。

### 2.2 2D Animation

2D clipはsprite／region、transform、color、visibility、typed gameplay／presentation event trackを持てる。frame numberではなくnormalized intervalとsource sample timeを保持し、variable presentation rateからauthoritative event cursorを逆算しない。Atlas／spriteのasset identityは[Asset lifecycle](../03-authoring/asset-lifecycle.md)を参照する。

2D blendはcompatible property trackだけを補間し、sprite selection等のdiscrete trackは明示policyで解決する。flip、pivot、socketはtyped propertyで、negative scaleやrenderer-specific flagへ暗黙投影しない。2D root motionを使う場合も3Dと同じproposal／resolution contractを通す。

### 2.3 3D Skeleton、Skin、Clip

Skeletonはsingle root policy、acyclic parent relation、unique canonical joint path、finite bind pose、invertible bind relationを検証する。ClipはSkeleton contract ref、time interval、typed translation／rotation／scale track、event、optional root trackを持つ。Skinはmesh section、joint remap、normalized finite influenceを持ち、missing jointやout-of-range influenceをsilent repairしない。

Importは[Asset lifecycle](../03-authoring/asset-lifecycle.md)のScene Import IR、conversion report、preview、reimport conflictを使用する。coordinate／unit conversionはSource-to-Engine transformをreceiptへ残し、Runtime sampleで再変換しない。Cooked payloadはTarget非依存のEngine envelope内へ置き、Backend archiveをpublic schemaにしない。

## 3. Instance、pose、event、root motion

InstanceはEntityごとにGraph state、clock、transition、loop count、event cursor、sampling context、pose scratch ownershipを持つ。sampling contextやscratchをinstance間、worker間で共有しない。Graph AssetとClip Assetはimmutable leaseで共有し、instanceからSource AssetやEditor stateへ参照しない。

Runtimeへの接続はcanonical identifiers `T30_PrePhysics`、`T40_MotionIntent`、`T60_PhysicsIntegrate`、`T80_AnimationFinalize`、`T90_PresentationBuild`、`T110_Publish`を[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)から参照する。順序、tick frequency、writer、publish boundaryは同文書だけが定義し、Animationはphase表や別tickを所有しない。

RuntimeはAnimation instanceへ一つのcanonical `AnimationEvaluationIntervalV1`を供給する。intervalは`interval_id`、tick ref、clock begin／end、direction、traversal kind、loop crossing ordinalを持つ。Runtime execution slotごとの役割や順序は本書で定義せず、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)の供給boundaryを消費する。

同じevaluationに属する`RootMotionProposal`、final `SkeletonPose`、`AnimationEvent`抽出は、同一`interval_id`とexact clock begin／endを必ず消費する。root motionだけを別intervalでsampleすること、resolved Physics inputの受領後にpose／event用clockを再計算すること、presentation frameから別intervalを作ることを禁止する。

Animation clockはcanonical intervalごとに一度だけadvanceする。root-motion sampling、clip sampling、blend、IK、pose、bounds、event extractionはintervalのpure consumerであり、個別にclock／loop count／event cursorをadvanceしない。全outputのvalidation成功後にinstance clockとevent cursorをendへ一回commitする。同じ`interval_id`のretryは同じoutputを返すidempotent evaluationとし、二回目のadvanceを行わない。前interval未commitのまま異なるintervalを受けた場合、または同じIDでbegin／endが異なる場合はinstance faultとする。evaluation failureではclockをadvanceせず、partial pose／event／root motionをpublishしない。

`RootMotionProposal`はAnimation instance、canonical interval ID、local delta translation／rotation、source state／clip、artifact versionを持つ。root-motion modeは`animation | gameplay_motor | disabled`である。`animation`はMotorがCollisionを解決し、`gameplay_motor`はAnimation root deltaを0としてresolved velocityからin-place poseを選び、`disabled`はroot trackをpresentationにも使わない。proposalはauthoritative Transformではない。

Physics resolved motion／ragdoll inputはgeneration付きsnapshotとして受け、Animationがnative Bodyをqueryしない。RagdollはPhysics pose inputとAnimation poseをtyped weight policyでblendするが、Physicsは`SkeletonPose`へ直接writeしない。NavigationはAnimation parameter候補をcommand／snapshotで渡せるが、Graph stateを直接変更しない。

`AnimationEvent`はevent contract ID、source instance／state／clip、canonical interval ID、normalized time、loop ordinal、payload ref、Gameplay／Presentation classificationを持つ。Event trackは`event_track_id`とregistered typed Event IDを使う。Event traversal policyは次のclosed contractであり、Clipへ全fieldを明示する。

```text
AnimationEventTraversalPolicyV1
  forward: emit_crossed
  reverse: suppress | emit_reverse
  seek: suppress
  editor_scrub: suppress_runtime
```

| `AnimationTraversalKindV1` | Pose／root motion | Event／cursor policy |
|---|---|---|
| `forward` | end poseと`[begin,end]`のroot deltaを同じintervalから評価 | `(begin,end]`のeventをemit。frameを飛び越えても省略しない |
| `reverse` | end poseと逆方向intervalのroot deltaを評価 | `suppress`はeventを発火せずcursorだけendへ移す。`emit_reverse`は`[end,begin)`を逆順走査して一度ずつemit |
| `seek` | destination poseを評価し、Runtimeへ渡すroot deltaは0 | crossed eventをGameplay／Presentationとも発火せず、cursorをdestinationへ置いて後続forwardでbackfillしない |
| `editor_scrub` | isolated Editor preview instanceだけがdestination poseを評価し、root proposalをRuntimeへ送らない | Gameplay／Presentation Runtime eventを発火せずlive instanceのclock／cursorを変更しない。Editor UIはnon-dispatched markerを表示できる |

loop跨ぎはcanonical intervalをtraversal方向のsub-intervalへ分解し、loop ordinalごとに同じpolicyを適用する。forward eventはevent time、track ID、event ID、loop ordinal、reverse eventは逆event time、track ID、event ID、loop ordinalでcanonicalizeする。同じcanonical intervalとpolicyからのretryはeventを再配送しない。Gameplay eventはauthoritative clock／cursorから一度だけ発行し、visibility、render frame drop、off-screen stateで停止しない。Presentation eventのAudio／VFX結果をGameplayへ逆入力しない。

Replayにはparameter snapshot、accepted motion／ragdoll input、Graph／artifact identity、state／clock／cursor、root-motion proposal、Gameplay event、pose hash projectionを供給する。recording ownerは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)のcanonical `T100_ReplayCheckpoint`であり、Animation固有phaseや旧phase名を設けない。

## 4. IK、retarget、LOD、memory／failure

IK requestはsolver kind、ordered chain／joint refs、target／pole transform、weight、iteration class、failure policyをEngine contractで表す。C1 solver familyはtwo-bone、look-at／aim、foot placementを持つ。Collision queryが必要なfoot placementは[Collision](collision.md)のversion付きresultを受け、IKからlive Physics Worldをqueryしない。

IKはbase blend後、local-to-model確定前のpose operationとしてGraph outputに適用する。chain missing、target non-finite、singularity、stale Collision resultでは`preserve_input_pose | disable_request | fail_instance`の明示policyを使い、NaN poseをpublishしない。iteration countやscratch上限はTarget profileとRuntime capacity ownerへ委譲し、共有budget値を本書に複写しない。

Retargetはsource／target Skeleton identity、semantic bone／chain mapping、reference pose、translation scale policy、rotation offset、unmapped-joint policyを検証する。runtime fuzzy name matchingを禁止し、Cook時に完全なmapping tableを生成する。missing required semantic、cyclic chain、incompatible root、non-finite offsetはcookを拒否する。Retarget結果はTarget Skeleton上の通常clip／poseとして扱い、source pointerをinstanceへ保持しない。

Animation presentation LODはpose update、presentation bone set、skinning mode、shadow poseを変更できるが、root motion、hitbox／weapon socket、foot contact、Gameplay event、authoritative boundsを低頻度presentation poseから取得しない。visibilityをGraph clock／transition／event cursor停止の入力にしない。共通LOD intent／resolutionとWorld dormancyは将来のRendering／World ownerへ委譲する。

Memory contractはAsset payloadのimmutable lease、instance-local state、frame-local pose scratch、published snapshotを別lifetimeにする。long-lived instanceがframe scratchを保持せず、job完了後にborrowをinvalidateする。allocation、shared pool、queue、memory envelope、backpressureの値と測定は[Runtime performance／capacity](../04-runtime/performance-capacity.md)だけが決定する。

Failure classはmissing／incompatible Asset、Graph validation failure、stale lease、parameter type mismatch、invalid sample interval、event cursor regression、IK／retarget failure、pose non-finite、job cancellation、snapshot publish failureである。Asset failureはlast valid generation、instance-local recoverable failureはdeclared fallback pose、invariant violationはRuntime fault policyを使う。silent bind pose fallbackでGameplay eventやroot motionを継続しない。

## 5. EditorとAI operations

Animation EditorはAsset／Graph browser、state／transition graph、timeline、skeleton tree、skin influence view、blend／IK／retarget preview、event／root-motion trace、diagnostic projectionを提供する。表示はSource Documentまたはimmutable Runtime snapshotのProjectionであり、live instanceを独自にmutateしない。手動編集とAIは同じChangeSet、validator、preview、cook、undo／redoを使う。

Operation familyは次を持つ。

- Asset／Graph／instance／pose／event traceのinspect
- 2D clip、Skeleton、Skin、3D Clip、Graph、Retarget Profileのcreate／update
- state、transition、blend、layer、mask、parameter、event、root-motion policyのedit
- IK requestとretarget mappingのproposal／validation
- import／reimport mapping、preview、cook
- fixture生成とqualification report

AIは「idleからrunへ滑らかに」「足を地面へ合わせる」「別Skeletonへclipを移す」等のintentをtyped operationへ解決し、対象Skeleton／Clip、Gameplay event、root-motion authority、Target、品質差が挙動を変える時だけ質問する。Backend job名、compression library、native archiveを選択肢にしない。assumption、affected Asset／Entity、before／after graph、event／root-motion差、failure fallbackをpreviewする。

Risk分類、authorization、commit可否は[AI Security／Approval](../01-governance/ai-security-approval.md)を参照し、本書でApproval表を複写しない。AI eval evidenceとprovenanceは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を消費する。

## 6. Qualificationと採用しないもの

Unit／schema fixtureはGraph reachability、parameter type、Skeleton hierarchy、Skin influence、Clip interval、event cursor、root-motion mode、closed traversal policy、IK chain、retarget mapping、artifact runtime-ID tableを検査する。Runtime fixtureは2D／3D sampling、transition interruption、blend／additive、loop event、root-motion／Motor resolution、ragdoll blend、stale Collision result、LOD invariant、asset swap、lease expiry、job cancellation、save／replay hashを含む。

canonical interval fixtureは、root-motion proposal、final pose、event抽出が同じinterval ID／begin／endを持つこと、clock／loop count／event cursorが一度だけcommitされること、同じintervalのretryが同じoutputを返して再advance／再配送しないこと、異なるpayloadを持つduplicate interval IDを拒否することを検査する。failure fixtureはpartial outputをpublishせずclockをadvanceしないことを検査する。

Event fixtureは少なくとも次を固定する。

- forwardでframeを飛び越えた`(begin,end]`のeventをcanonical orderで一度ずつ発火する。
- forwardのloop跨ぎで各loop ordinalのeventを欠落／重複なく発火する。
- reverse `suppress`でevent非発火かつcursorだけがendへ進み、`emit_reverse`で`[end,begin)`を逆canonical orderで一度ずつ発火する。
- seekでGameplay／Presentation eventとroot deltaを抑止し、destination後のforwardで過去eventをbackfillしない。
- Editor scrubでRuntime event／root proposalを発行せず、live instance clock／cursor／state hashを変更しない。
- forward、reverse、seek、Editor scrubの各caseをparallel実行、retry、Replay再生してevent sequence、pose hash、root-motion hashが再現する。

Editor／AI fixtureはGraph diff、timeline preview、import／reimport conflict、question-required intent、unsupported operation、diagnostic remediation、manual／AI equivalenceを検査する。private Backend conformanceは同じSource fixtureからEngine-owned pose／event／diagnosticを生成し、public contractにVendor差が漏れないことを確認する。measurementとcapacity promotionは[Runtime performance／capacity](../04-runtime/performance-capacity.md)、evidence envelopeは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を使用する。

次を採用しない。

- Vendor animation型、job、pointer、archiveのpublic API化
- clip／joint名やfunction名を使うRuntime dispatch
- Rendering visibilityからauthoritative clock／event cursorを停止すること
- Animation TransformとPhysics resolved Transformの二重write
- IK／foot placementからのlive Physics query
- silent bind-pose fallbackでGameplay event／root motionを継続すること
- Runtime phase／shared capacity、Physics／Navigation／Collision／Rendering責務の再所有
