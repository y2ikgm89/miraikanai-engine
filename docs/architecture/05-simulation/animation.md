# Miraikanai Engine Animation Contract

- 文書ID: mirakan.arch.simulation-animation
- 文書状態: review
- 実装状態: absent
- 検証状態: design-reviewed
- 正本範囲: Animation Source／Cooked Asset、typed graph、2D clip、3D skeleton／skin／clip、instance／pose ownership、event／root motion、IK、retarget、Animation memory／failure、Editor／AI operation、Animation qualification
- 非正本範囲: Runtime phase／Simulation Advance／shared capacity、ECS storage、Save／Replay header・transport、Physics motion resolution、Collision query semantics、Navigation artifact、Rendering skin execution、LOD共通intent、Asset transaction、external dependency version／build pin、AI authorization。各Owner文書を参照する
- 規範依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Math／Core Utilities](../02-foundation/math-core.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)、[Scheduling／Lifetime](../04-runtime/scheduling-lifetime.md)、[Performance／Capacity](../04-runtime/performance-capacity.md)
- 関連文書: [AI Security／Approval](../01-governance/ai-security-approval.md)、[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable contracts](../02-foundation/executable-contracts.md)、[Memory／Pointers](../02-foundation/memory-pointers.md)、[Asset lifecycle](../03-authoring/asset-lifecycle.md)、[Project state](../03-authoring/project-state.md)、[Persistence／Save](../04-runtime/persistence-save.md)、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)、[Runtime performance／capacity](../04-runtime/performance-capacity.md)、[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md)、[Collision](collision.md)、[Physics](physics.md)、[Navigation](navigation.md)、[LOD](../06-rendering/lod.md)、[World](../06-rendering/world.md)
- 根拠区分: project-decision（外部仕様を引用する箇所はofficial-spec、未計測の固定値はprovisional）
- 外部根拠確認日: 2026-07-21

## 1. 結論と所有境界

AnimationはEngine-owned Asset、Graph、Instance、Pose、Event、Root Motion proposal、IK、Retarget contractを公開し、sampling／compression Backendをprivate Adapterへ隔離する。Project C++、GameplayDefinition、AI、Editor、RenderingへVendor job、runtime object、pointer、archive formatを公開しない。dependencyのexact version／commit／license／build optionは[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)だけが所有する。

Animationはauthoritative motion Transformを書かず、Navigationを進めず、Collision queryを再定義しない。root motionは`MotionIntentContributionV1`として[Navigation](navigation.md)のgeneric contribution registryへ登録され、選択済み`MotionExecutorPortV1`だけへ届くproposalである。Animationはregistered resolved motionを受けてposeを確定する。実行順、contribution集約、writerは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)だけが所有する。

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
| `AnimationParameterSnapshot` | gameplayからlatchしたtyped parameter | immutable Simulation Advance input |
| `SkeletonPose` | local／model pose、bounds、socket projection | published pose snapshot |

Cookerはcanonical joint pathとSource Stable IDをartifact内の固定順へ並べ、non-zero `JointRuntimeId`／`ClipRuntimeId`を割り当てる。Runtime IDは対応artifact identityと組でだけ使い、Source、Save identity、別artifact比較へ使用しない。joint名、clip名、function名によるRuntime dispatchを禁止する。Event tag、parameter、semantic boneは[Executable contracts](../02-foundation/executable-contracts.md)のcontract refへ解決する。

### 2.1 Typed Animation Graph

Graph node familyはstate machine、clip sample、2D／1D blend、additive blend、layer／mask、pose cache、IK request、outputをclosed tagged unionで表す。transitionはsource／target Stable ID、typed condition、duration semantics、interrupt policy、event policyを持つ。任意script、function pointer、Vendor nodeをGraphへ埋め込まない。

Graph validationはreachable output、cycle policy、parameter type、clip／skeleton compatibility、mask joint、transition conflict、event duplication、root-motion source一意性を検査する。同priority transitionはStable IDのcanonical orderで決め、authoring insertion順やworker completion順を使わない。

#### 2.1.1 AI-readable Graph closure

`AnimationGraphDefinitionV1`候補はGraph Stable ID、exact Source revision、Target Skeleton contract ref、typed Parameter定義、closed Node／Port／Edge集合、exact一件のOutput node、event authority、root-motion policy、semantic content hashを持つ。集合はTarget／capacity Ownerが登録する上限を持ち、Stable ID byte順へcanonicalizeする。Parameterはexact value type、unit、finite／range policy、default、latch sourceを持ち、display name、Inspector row、文字列parseで型を決めない。

State Machineはmachine／state／transition Stable ID、exact initial state、typed condition AST、priority、duration、blend curve、interrupt、time synchronization、event policyを持つ。C1 condition ASTはlatched Parameter、constant、comparison、finite boolean compositionだけを受理し、arbitrary callback、function名、clock読取り、Renderer visibility、worker stateを参照しない。遷移blend curveとtime synchronizationはclosed値としてSourceへ明示し、Backend defaultへ委ねない。

AI／Editorが読む`AnimationGraphSemanticProjectionV1`候補はGraph／Artifact revision、Node／Port／Edge Stable ref、Parameter type／unit／value source、active state／transition、Blend sample／weight、sync leader／phase、IK request／result／failure、event／root-motion authority、Diagnostic、completeness／gapをbounded queryで返す。node座標、wire bend、zoom、selection、localized label、screen coordinateをsemantic hashまたはCommand targetへ含めない。Human Inspector、AI、CLI、fixtureは同じProjection queryを使用し、AI専用Graphや文字列DSLを第二正本にしない。

#### 2.1.2 Blend Space、同期、event／root motion

`AnimationBlendSpaceSourceV1`候補はBlend Space Stable ID、Target Skeleton ref、1本または2本のtyped axis、sample集合、interpolation、outside-hull policy、synchronization、event policy、root-motion policyを持つ。AxisはParameter ref、unit、finite min／max、`clamp | reject | wrap`を持ち、`wrap`はpositive periodとcanonical originを必須にする。SampleはStable ID、compatible Clip ref、finite axis position、playback scale、optional phase offset、sync marker track refを持つ。

C1 interpolationは1Dの`linear_1d`と2Dの`delaunay_barycentric_2d`だけである。2D triangulationの同円・同一直線tieはSample Stable ID byte順で解決し、Source配列順、Editor挿入順、floating-point libraryの未規定順を使わない。outside-hullは`clamp_to_hull | reject`をSourceへ必須とする。weightはfinite、non-negative、正規化済みにし、canonical Sample Stable ID順でPoseを合成する。

Synchronizationは`none | normalized_phase | marker_track`を明示する。`marker_track`はmarker Stable ID、cycle order、exact clip timebase tickを持ち、全sampleのmarker semantic sequenceが一致しなければCookを拒否する。leaderはeffective weight降順、同値はSample Stable ID byte昇順で一意に選び、worker完了順で変えない。

Gameplay eventはBlend weightやPresentation frameから生成せず、Graph outputごとにexact一件のauthoritative event sourceまたはstate／sync-group event trackを指定する。Presentation eventだけが`dominant_sample`を選べ、同weight tieはSample Stable ID順とする。Animation root motionをBlendする場合はtranslation／rotation delta、weight normalization、hemisphere、failureを固定したqualified named algorithm refを必須とし、そのrefがなければ`executor_driven | disabled`だけを許可する。

### 2.2 2D Animation

2D clipはsprite／region、transform、color、visibility、typed gameplay／presentation event trackを持てる。frame numberではなくnormalized intervalとsource sample timeを保持し、variable presentation rateからauthoritative event cursorを逆算しない。Atlas／spriteのasset identityは[Asset lifecycle](../03-authoring/asset-lifecycle.md)を参照する。

2D blendはcompatible property trackだけを補間し、sprite selection等のdiscrete trackは明示policyで解決する。flip、pivot、socketはtyped propertyで、negative scaleやrenderer-specific flagへ暗黙投影しない。2D root motionを使う場合も3Dと同じproposal／resolution contractを通す。

#### Sprite Flipbook source

次の3型をSprite Flipbookの唯一のSource正本とする。C1手動EditorとQualified importerは同じschemaを生成し、RuntimeはCook済みのStable ID tableを解決する。

```text
SpriteAnimationFrameV1
  sprite_asset_revision_ref: exact Asset revision ref
  sprite_id: StableId
  duration_timebase_ticks: uint32
  untrimmed_pivot_texel: finite float2
  local_offset_m: finite Displacement2f
  collision_pose_ref: optional StableId
  socket_pose_set_ref: optional StableId

SpriteAnimationClipSourceV1
  clip_id: StableId
  clip_timebase_hz: ReducedPositiveRationalV1
  frames: bounded array[1..4096]<SpriteAnimationFrameV1>
  playback_mode: forward | reverse | ping_pong
  loop_mode: once | loop
  event_tracks: bounded array[0..256]<TypedAnimationEventTrackV1>
  default_speed_q16: positive Q16.16

TypedAnimationEventTrackV1
  event_track_id: StableId
  event_contract_ref: exact registered typed Event ref
  classification: gameplay | presentation
  keys: bounded array[0..4096]<{event_key_id: StableId, timebase_tick: uint64, payload_ref: optional exact typed payload ref}>
```

`clip_id`、`sprite_id`、`event_track_id`、event key IDは永続Stable IDである。Frameはexact Sprite Asset revisionとStable Sprite IDを参照し、Texture path、atlas index、frame index、Source配列index、Aseprite tag／layer名、importer tagをRuntime identityにしない。Reimportは旧新revision、Stable Sprite／Clip ID対応、追加／削除／曖昧候補、pivot／trim／event差分をconflict previewへ出し、明示resolutionなしにIDを再採番、近似対応、consumer revision更新しない。

一Clipは1～4,096 frame、`clip_timebase_hz`は既約なexact rationalで`1/1`～`48000/1 Hz`、各`duration_timebase_ticks`は1～60,000とする。全Frame durationのchecked `uint64`和を`total_timebase_ticks`とし、`total_timebase_ticks × clip_timebase_hz.denominator <= 86,400 × clip_timebase_hz.numerator`（24時間以下）をchecked `uint128`で検証する。event keyは`0 <= timebase_tick <= total_timebase_ticks`とし、Simulation CadenceのHz、render rate、importer FPSから補完しない。`default_speed_q16`はQ16.16の`[0.0625, 16.0]`で、0、負値、overflowを拒否する。一時停止はspeed 0へのSource書換えでなく、Animation stateまたはClock Domainのtyped commandを使う。Frameのuntrimmed pivotと`local_offset_m`はfiniteとし、Atlas repack／trim後も見かけ位置、Collider、socketを同じCPU正規座標へCookする。

`forward | reverse | ping_pong`と`once | loop`の組合せをすべて定義する。`once`は終端poseを保持し、`loop`は定義済みseamへ戻る。ping-pongは端frameを折返し時に重複評価せず、3 frameなら`A,B,C,B,A`を一周期とする。ping-pong復路はtraversal kind `reverse`のsub-intervalとして評価し、clipの`AnimationEventTraversalPolicyV1.reverse`（`suppress | emit_reverse`）をそのまま適用する。1 frame clip、reverse、seek、loop跨ぎ、一advance内の複数frame通過でも、既存のauthoritative event cursorと`AnimationEventTraversalPolicyV1`を使い、crossed typed Eventを境界順に高々一度だけ発行する。Editor scrubはisolated preview cursorを使いRuntime Eventを発火しない。

`TypedAnimationEventTrackV1.keys`はtimebase tick、event track ID、event key IDの順でcanonicalizeし、duplicate key ID、clip範囲外timebase tick、未登録Event、payload type mismatchを拒否する。任意関数名やimporter文字列をdispatchしない。Gameplay Eventはvisibility、offscreen、GPU culling、Quality tier、presentation update LODでdrop／重複／遅延させない。

`collision_pose_ref`と`socket_pose_set_ref`は登録済みCPU正規poseだけを参照する。Sprite alpha、GPU pose、Renderer visibilityからHitbox／Hurtbox／socketを生成せず、Presentation poseをauthorityへ戻さない。Sensor切替はCollision Ownerのtyped RuleへEventを渡し、AnimationがColliderを直接writeしない。fixtureはforward／reverse／ping-pong（ping-pong×reverse policy `suppress`／`emit_reverse`別の期待event列を含む）、once／loop、端frame、複数境界、Atlas repack、pivot／offset、CPU collision／socket pose、reimport conflict、Save／Replay event sequenceを固定する。

`SpriteAnimationClipSourceV1`は`AnimationClip2DSourceAsset`の特殊形ではなく独立したSource Asset種であり、適用範囲は排他とする。sprite frame列だけを持つFlipbookは`SpriteAnimationClipSourceV1`を、transform／color等のproperty trackを要する2D clipは`AnimationClip2DSourceAsset`を使い、同じclipを両形式で二重に正本化しない。Graphのclip sample nodeは両Asset種のCooked 2D clipを参照できる。clip内`playback_mode`／`loop_mode`はclip-local sampling semanticsとしてclip sample node評価に適用し、`once`の終端は終端poseを保持する。state遷移の成立はGraphのtyped conditionとtransition policyだけが決定し、clip側fieldがGraph transitionを暗黙に発火・抑止しない。Graphがclipのplayback／loopを上書きする場合はclip sample nodeのtyped fieldで明示する。

### 2.3 3D Skeleton、Skin、Clip

Skeletonはsingle root policy、acyclic parent relation、unique canonical joint path、finite bind pose、invertible bind relationを検証する。ClipはSkeleton contract ref、time interval、typed translation／rotation／scale track、event、optional root trackを持つ。Skinはmesh section、joint remap、normalized finite influenceを持ち、missing jointやout-of-range influenceをsilent repairしない。

Importは[Asset lifecycle](../03-authoring/asset-lifecycle.md)のScene Import IR、conversion report、preview、reimport conflictを使用する。coordinate／unit conversionはSource-to-Engine transformをreceiptへ残し、Runtime sampleで再変換しない。Cooked payloadはTarget非依存のEngine envelope内へ置き、Backend archiveをpublic schemaにしない。

### 2.4 外部Engine比較から採るCoverage

外部EngineのGraph、Blend Space、State Machine、IK、Retarget、Animation Debug機能は、対象範囲の漏れを発見する比較Evidenceとしてだけ使う。外部API、Node名、Class hierarchy、serialization、既定補間、solver defaultをMiraikanaiのidentityまたは採用済みContractへ変換しない。MiraikanaiではStable ID、typed Port、authority分離、deterministic tie、bounded Projection、failure、QualificationをEngine-owned contractとして閉じる。

## 3. Instance、pose、event、root motion

InstanceはEntityごとにGraph state、clock、transition、loop count、event cursor、sampling context、pose scratch ownershipを持つ。sampling contextやscratchをinstance間、worker間で共有しない。Graph AssetとClip Assetはimmutable leaseで共有し、instanceからSource AssetやEditor stateへ参照しない。

Runtimeへの接続はcanonical identifiers `T30_PrePhysics`、`T40_MotionIntent`、`T60_PhysicsIntegrate`、`T80_AnimationFinalize`、`T90_PresentationBuild`、`T110_Publish`を[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)から参照する。順序、Simulation Cadence、writer、publish boundaryは同文書だけが定義し、Animationはphase表や別Cadenceを所有しない。

RuntimeはAnimation instanceへ一つのcanonical `AnimationEvaluationIntervalV1`を供給する。intervalは`interval_id`、`SimulationCadenceProfileRefV1`、`SimulationAdvanceIntervalV1.interval_content_hash`、advance sequence、clock begin／end、direction、traversal kind、loop crossing ordinalを持つ。non-null logical durationは`clip_timebase_hz`とのexact rational積をinstance-local remainder付き整数timebase tickへ変換し、浮動小数の累積または`1/60`固定値を使わない。duration-nullのturn-based／explicit-stepは、別のqualified advance-driven Animation clock policyがexact clip-timebase deltaを供給する場合だけclockを進め、それ以外はno-advanceまたは`cadence_profile_not_qualified`とする。Runtime execution slotごとの役割や順序は本書で定義せず、[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)の供給boundaryを消費する。

同じevaluationに属する`RootMotionProposalV1`、final `SkeletonPose`、`AnimationEvent`抽出は、同一`interval_id`とexact clock begin／endを必ず消費する。root motionだけを別intervalでsampleすること、registered resolved motion inputの受領後にpose／event用clockを再計算すること、presentation frameから別intervalを作ることを禁止する。

Animation clockはcanonical intervalごとに一度だけadvanceする。root-motion sampling、clip sampling、blend、IK、pose、bounds、event extractionはintervalのpure consumerであり、個別にclock／loop count／event cursorをadvanceしない。全outputのvalidation成功後にinstance clockとevent cursorをendへ一回commitする。同じ`interval_id`のretryは同じoutputを返すidempotent evaluationとし、二回目のadvanceを行わない。前interval未commitのまま異なるintervalを受けた場合、または同じIDでbegin／endが異なる場合はinstance faultとする。evaluation failureではclockをadvanceせず、partial pose／event／root motionをpublishしない。

`RootMotionProposalV1`のexact MCD refは`McdContractRefV1 {id=type.animation.root_motion_proposal, version=1, contract_set_hash}`であり、schemaを次へ固定する。IDへPascalCaseまたは`@1`を埋め込まない。

```text
RootMotionProposalV1
  proposal_id
  animation_instance_ref
  interval_id
  local_delta_translation
  local_delta_rotation
  source_state_ref
  source_clip_ref
  artifact_version_ref
```

root-motion modeは`animation | executor_driven | disabled`である。`animation`は`RootMotionProposalV1`をowner登録済み`MotionIntentContributionV1`としてgeneric contribution resolverへ提出し、resolverだけが[Navigation](navigation.md)の`MotionExecutorIntentBatchV1`へ包んでselected executorへ配送する。選択identityはNavigationの`MotionIntentBindingSelectionDocumentV1`とbatch closureだけに存在し、proposalへselected Capability／Provider refを埋め込まない。Animationからselected executorへのdirect submissionは禁止する。selected Providerとcontribution adapterがexact `type.animation.root_motion_proposal` version／Contract set hashを受理できなければActivation前にtyped rejectし、Animation clock、pose、event cursor、last-valid resolved motionを変更しない。`executor_driven`はAnimation root deltaを0としてregistered resolved motionからin-place poseを選び、`disabled`はroot trackをpresentationにも使わない。旧`gameplay_motor`は同じ意味の`executor_driven`へclean migrationし、aliasとして読まない。proposalはauthoritative Transformではない。

現行C1／C2はselected Motion Executorの`resolved_motion_schema_ref`へ適合するgeneration付きsnapshotだけを受け、AnimationがProvider-private state、native Body、Transform componentをqueryしない。`ragdoll`は`future.capability.vehicle-ragdoll-crowd-motion-warping`の予約語であり、同Future Entryがapproved Future-to-Active Promotion Manifest、Control Plane Rebaseline、Active Definition migrationでactive Capabilityへ昇格し、Ragdoll固有Owner contract、Target binding、Authority、Save／Replay schema、positive／negative fixtureが承認されるまで、Animation input、Graph node、operationとして公開しない。昇格前のRagdoll要求は`capability_unavailable`で拒否し、pose、clock、event cursor、Project／Save／Replay stateを変更しない。将来昇格後もPhysicsは`SkeletonPose`へ直接writeせず、versioned Physics pose inputとAnimation poseの合成はAnimation ownerだけが行う。NavigationはAnimation parameter候補をcommand／snapshotで渡せるが、Graph stateを直接変更しない。

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

現行Replayにはparameter snapshot、accepted motion input、Graph／artifact identity、state／clock／cursor、root-motion proposal、Gameplay event、pose hash projectionを供給する。receipt-free Animation Replay Projectionは[Persistence／Save](../04-runtime/persistence-save.md#51-replay-transport-binding)の`RuntimeReplayDomainProjectionRefV1`として一意に参照され、完成した`RuntimeReplayProjectionV1`とのroot外`RuntimeReplayDomainBindingV1`を持つ。Persistence OwnerだけがBindingを`RuntimeReplayBundleManifestV1`へmembershipとして閉じ、Header／binding／Bundle refをProjection base recordへ埋め戻さない。Cadence Profile、sealed Interval、Animation projectionの一致をPersistence validationで検証する。Ragdoll inputは現行Replay schemaへ含めず、Future Entryのactive migration時にversioned schema、旧Save／Replay拒否または明示migration、deterministic fixtureを同時に追加する。recording slotは[Runtime scheduling／lifetime](../04-runtime/scheduling-lifetime.md)のcanonical `T100_ReplayCheckpoint`だけを参照し、Animation固有phaseや旧phase名を設けない。

Animation Save projectionは次のclosed schemaだけを使い、instance memory、runtime ID、pose scratch、native decoder stateを保存しない。

```text
AnimationSaveStateV1
  projection_id: StableId
  projection_version: positive uint32
  schema_version: 1
  animation_instance_identity_ref: exact Save-stable identity ref
  graph_definition_ref: exact revision/content-hash ref
  cooked_animation_artifact_refs[1..64]: exact version/content-hash refs
  cadence_profile_ref: SimulationCadenceProfileRefV1
  last_committed_simulation_advance_interval_ref:
    SimulationAdvanceIntervalRefV1
  last_committed_simulation_advance_interval_sha256: SHA-256
  state_machine_states[0..64]:
    state_machine_node_ref: exact graph-local StableId ref
    active_state_ref: exact graph-local StableId ref
    transition_state:
      kind: none
      | in_progress {
          source_state_ref,
          target_state_ref,
          elapsed_seconds: {numerator:uint64, denominator:positive uint64}
        }
  clip_playback_states[0..256]:
    clip_sample_node_ref: exact graph-local StableId ref
    clip_ref: exact Source StableId + artifact identity ref
    clock_timebase_tick: uint64
    clock_conversion_remainder:
      numerator: uint64
      denominator: positive uint64
    traversal_direction: forward | reverse
    loop_ordinal: uint64
    event_cursors[0..256]: {
      event_track_ref: exact StableId ref,
      timebase_tick: uint64,
      loop_ordinal: uint64
    }
  projection_content_hash: SHA-256
```

各clock remainderは`numerator < denominator`のnon-negative proper fractionで、zeroは`0/1`だけ、non-zeroはgcd 1へcanonicalizeする。`elapsed_seconds`はnon-negative reduced rational、zeroは同じく`0/1`で、選択transition duration以下とする。変換はchecked `uint128` intermediateを使い、既約化後のSave Fieldが`uint64`へ収まらないCadence／Clip／transition組合せをQualificationで拒否する。`projection_content_hash`はASCII `MIRAKAN_ANIMATION_SAVE_STATE_V1`と同Fieldだけを除くclosed canonical bytesから計算する。Saveは完全にcommit済みのAnimation interval後だけ作り、完成Projectionを[Persistence／Save](../04-runtime/persistence-save.md#23-timebase-headerとbinding)のroot外`AuthoritativeSaveDomainBindingV1`によってcompleted Headerへ結ぶ。Headerのstate-owner Projection Ref集合には本Projectionのowner／Type／ID／version／content hashがexact一件存在し、Header／BindingをProjectionへ埋め戻さない。Projectionの`cadence_profile_ref`はHeaderの`simulation_cadence_profile_ref`、Projectionの`last_committed_simulation_advance_interval_ref`／隣接completed SHAはHeaderの同名Fieldとそれぞれbyte equalityにする。Interval Ref内のProfile RefもProjectionの`cadence_profile_ref`と一致しなければならない。state machine stateはmachine Node Stable ID順、clip playback stateはclip sample Node Stable ID順、event cursorはtrack Stable ID順へstrict sortし、重複、clip範囲外tick、Graph／Artifact closure外ref、Graphから導出したactive machine／clip集合との差を拒否する。Pose cache、blend scratch、published poseはGraph／clock stateから再計算するderived stateとしてSaveしない。LoadはGraph／Artifact identity、Header／Binding、Profile ref、last Interval Ref／completed SHA、Projection membership、全machine／clip state、clock／loop／cursor／transition invariantをstagingで検証し、同じremainderから次clock conversionを継続する。current PackageまたはHeaderとの不一致をlatest Artifact、0 remainder、現在時刻へ読み替えず、明示migrationまたは`animation_save_incompatible`で副作用前に拒否する。

## 4. IK、retarget、LOD、memory／failure

### 4.1 IK request、座標空間、評価順

IK requestはrequest／Graph node／Animation instance／canonical intervalのexact ref、`two_bone | look_at_aim | foot_placement`、ordered joint chain、target／pole、position／rotation weight、joint-limit profile、solver profile、evaluation ordinal、failure policyを持つ。target／poleはfinite Transformに加え、`world | entity_local | skeleton_model | joint_local`のspace、space subject ref、generationを必須にし、field欠落時にworldまたはmodel spaceを推測しない。

評価順はTarget SkeletonへのRetarget、base Clip／Blend Space、additive、Layer／Mask、IK stage、local-to-model、authoritative bounds／socket projectionである。同じIK stage内はevaluation ordinal、同値はrequest Stable ID byte順に固定する。chain missing、target non-finite、singularity、stale generationでは`preserve_input_pose | disable_request | fail_instance`の明示policyを使い、NaN poseをpublishしない。solver iteration、epsilon、joint limit、scratch上限はqualified solver／Target profileへexact解決し、Backend defaultへ委ねない。

### 4.2 Foot placement query／result binding

Foot placementはAnimationとCollisionの暗黙callbackで接続せず、Animation-owned `AnimationFootPlacementQueryBindingV1`候補と`AnimationFootPlacementResultBindingV1`候補でgeneric `CollisionQueryRequestV1`／`CollisionQueryResultV1`を束縛する。Query BindingはAnimation instance、target interval、foot semantic／joint、last committed pose generation、authoritative subject Transform generation、Collision scene version、query request ref／hash、foot-placement profile refを持つ。Result Bindingは同じidentity集合、exact result ref／hash、observed scene version、normalized hit selection、result status、binding hashを持つ。

canonical接続はT30でlast committed poseと同advanceのsealed Transformからbounded queryを構築し、T50でgeneric Collision queryを実行し、T60でversion／generation検証済みimmutable Result Bindingをstagingへ公開し、T80で同じtarget intervalのIK requestだけが消費する順である。scene／Transform／pose generationがstale、subject motion bound超過、resultがtruncated／failedの場合は明示failure policyを使い、別足、別interval、last-hit、live Physics queryへ差し替えない。同じintervalのretryは同じQuery／Result hashを使い、query再発行または二重IK適用を行わない。

### 4.3 Retarget、LOD、memory／failure

Retargetはsource／target Skeleton identity、semantic bone／chain mapping、reference pose、translation scale policy、rotation offset、unmapped-joint policyを検証する。runtime fuzzy name matchingを禁止し、Cook時に完全なmapping tableを生成する。missing required semantic、cyclic chain、incompatible root、non-finite offsetはcookを拒否する。Retarget結果はTarget Skeleton上の通常clip／poseとして扱い、source pointerをinstanceへ保持しない。

Animation presentation LODはpose update、presentation bone set、skinning mode、shadow poseを変更できるが、root motion、hitbox／weapon socket、foot contact、Gameplay event、authoritative boundsを低頻度presentation poseから取得しない。visibilityをGraph clock／transition／event cursor停止の入力にしない。共通LOD intent／resolutionは[LOD](../06-rendering/lod.md)、World dormancyは[World](../06-rendering/world.md)へ委譲する。

Memory contractは[Memory／Pointers](../02-foundation/memory-pointers.md)のbindingによりAsset payloadのimmutable lease、instance-local state、frame-local pose scratch、published snapshotを別lifetimeにする。long-lived instanceがframe scratchを保持せず、job完了後にborrowをinvalidateする。allocation、shared pool、queue、memory envelope、backpressureの値と測定は[Runtime performance／capacity](../04-runtime/performance-capacity.md)だけが決定する。

Failure classはmissing／incompatible Asset、Graph validation failure、stale lease、parameter type mismatch、invalid sample interval、event cursor regression、IK／retarget failure、pose non-finite、job cancellation、snapshot publish failureである。Asset failureはlast valid generation、instance-local recoverable failureはdeclared fallback pose、invariant violationはRuntime fault policyを使う。silent bind pose fallbackでGameplay eventやroot motionを継続しない。

## 5. EditorとAI planned actions

Animation EditorはAsset／Graph browser、state／transition graph、timeline、skeleton tree、skin influence view、blend／IK／retarget preview、event／root-motion trace、diagnostic projectionを提供する。表示はSource Documentまたはimmutable Runtime snapshotのProjectionであり、live instanceを独自にmutateしない。手動編集とAIは同じChangeSet、validator、preview、cook、undo／redoを使う。

AnimationのAI-readable entry pointは、全文書の自由検索ではなくboundedな`AnimationCapabilitySemanticProjectionV1`候補とする。ProjectionはOwner文書／revision、Capability／Target activation state、supported Asset／Graph node／Blend interpolation／IK solver、required input contract、current callable Operation ref、disabled reason、question-required condition、relevant Qualification／Diagnostic refを返す。current Operation集合は`[]`であり、planned action名、外部Engine機能名、Editor commandからOperation IDを合成しない。

Animation Debug projectionはGraph／Artifact revision、canonical interval、active state／transition、Parameter snapshot、Blend sample／weight／sync leader、IK Query／Result Binding、pose generation／hash、root-motion proposal／resolution、event cursor／emission、LOD ref、capacity／failure、completeness／gapを持つ。[Debugging／observability／replay](../04-runtime/debugging-observability-replay.md#11-domain-debug-projection)のBindingがmaterializeされるまでEditorはSource previewとdisabled reasonだけを表示し、live trace、Replay同期、validated causeを成功表示しない。Debug projectionからGraph state、Parameter、IK target、Runtime clockをmutateするback-edgeを設けない。

### 5.1 MCD／Tool／Debug Binding Definition Closure

Animationの設計契約、実行可能Tool、Debug連携は別のclosureであり、一つの型名またはEditor表示から他を成立済みと推測しない。

| Closure | Animation Ownerが定義するもの | 外部Ownerとのhandoff | 現在状態 |
|---|---|---|---|
| Runtime／semantic MCD Type | `RootMotionProposalV1`のplanned local ID／schema、Graph／Blend／IK／Foot-placement／Projectionのcandidate payload、bound、canonical order、failure | [Executable contracts](../02-foundation/executable-contracts.md)がMCD envelope、member materialization、Contract set root、generated codec／schemaの一致を所有 | `RootMotionProposalV1`はID／schemaを設計固定した未materialize契約、その他はcandidateでcurrent memberではない |
| Authoring Tool | `AnimationCapabilitySemanticProjectionV1`候補、planned action vocabulary、Animation固有input／result意味、preview差分、不変条件 | Executable Contracts、[AI Security／Approval](../01-governance/ai-security-approval.md)、Editor OwnerがOperationからProvider projectionまでを一つのactivation closureへ閉じる | current callable Operation refはexact `[]`、Capabilityは`not_activated` |
| Debug Binding | `AnimationDebugProjectionV1`候補payload、safe publish point、Animation固有completeness／gap、negative fixture | Debug Ownerが共通Envelope、Binding、Store／Index／Query／Replay連携を所有 | materialized Binding集合はexact `[]`、binding stateは`required_unmaterialized` |

Debug safe publish pointはT80のcommit成功後に公開されたimmutable Animation snapshotである。evaluation failureまたはpartial publishではlast-validをcurrent成功値として偽装せず、対象interval、source generation、omitted field、gap／failureを返す。Debug consumerはrecord／query／compareできるが、Graph state、Parameter、IK target、clock、event cursorへwriteしない。

Definition Closure成立には、MCD Type候補が同じContract set rootのlocal record／envelope／payload／canonical encoding／hash／version／generated projectionへ閉じること、Toolが採用したexact Operation集合についてManifestからaliasまでの完全closureをatomicに持つこと、DebugがAnimation payload Type ref、共通Envelope Type ref、T80 safe-boundary contract、Target、query、capacity、privacy、fixtureを束縛したexact一件のBindingを持つことを要求する。三closureは独立に`not_activated`または`required_unmaterialized`を維持し、一件の成立を他二件のActivation根拠にしない。

次はStable IDでないplanned semantic action vocabularyであり、current MCD Operation familyではない。

- Asset／Graph／instance／pose／event traceのinspect
- 2D clip、Skeleton、Skin、3D Clip、Graph、Retarget Profileのcreate／update
- state、transition、blend、layer、mask、parameter、event、root-motion policyのedit
- IK requestとretarget mappingのproposal／validation
- import／reimport mapping、preview、cook
- fixture生成とqualification report

Animation ownerのcurrent MCD／Owner Manifest／Service allowlist／Policy／Validator／Diagnostic／Receipt／Provider／MCP／alias Operation集合はすべて`[]`、Capability stateは`not_activated`である。future work item `activation.animation.authoring_operations.v1`が採用するexact ID集合と完全closureを一transactionで登録するまでdispatchせず、action名からIDを生成しない。Activation後、AIは「idleからrunへ滑らかに」「足を地面へ合わせる」「別Skeletonへclipを移す」等のintentをtyped actionへ解決し、対象Skeleton／Clip、Gameplay event、root-motion authority、Target、品質差が挙動を変える時だけ質問する。Backend job名、compression library、native archiveを選択肢にしない。assumption、affected Asset／Entity、before／after graph、event／root-motion差、failure fallbackをpreviewする。

Risk分類、authorization、commit可否は[AI Security／Approval](../01-governance/ai-security-approval.md)を参照し、本書でApproval表を複写しない。AI eval evidenceとprovenanceは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を消費する。

## 6. Qualificationと採用しないもの

Unit／schema fixtureはGraph reachability、parameter type、Skeleton hierarchy、Skin influence、Clip interval、event cursor、`RootMotionProposalV1` exact contract ID／Field、root-motion mode、旧`gameplay_motor` migration、closed traversal policy、IK chain、retarget mapping、artifact runtime-ID tableを検査する。Runtime fixtureは2D／3D sampling、transition interruption、blend／additive、loop event、root-motion／selected executor resolution、`animation` modeでproposal schema未受理時のtyped rejectとclock／pose／event cursor／last-valid不変、Physicsなしboard-token／RTS stub、stale resolved motion、provider failure時のlast-valid pose／clock不変、stale Collision result、LOD invariant、asset swap、lease expiry、job cancellation、save／replay hashを含む。Save fixtureはCadence、last Interval Ref／completed SHA、Projection membership、root外Save Domain Bindingを照合し、Replay fixtureはreceipt-free Projection Ref→root Replay Projection→root外Replay Domain Binding→Replay Bundle Manifestを照合する。各一Field差、ProjectionへのHeader／Binding／Bundle埋戻し、別Replay root／Projectionを結ぶBinding、Bundle membershipのmissing／extraをload／Replay前に拒否する。現行Ragdoll negative fixtureは、Ragdoll input／Graph node／operationを`capability_unavailable`で拒否し、pose、clock、event cursor、Project revision、Save／Replay hashが不変であることを検査する。Ragdoll blendのpositive fixtureはFuture Entryが`active`へ昇格するまで登録も実行もしない。

canonical interval fixtureは、root-motion proposal、final pose、event抽出が同じinterval ID／begin／endを持つこと、clock／loop count／event cursorが一度だけcommitされること、同じintervalのretryが同じoutputを返して再advance／再配送しないこと、異なるpayloadを持つduplicate interval IDを拒否することを検査する。failure fixtureはpartial outputをpublishせずclockをadvanceしないことを検査する。

Event fixtureは少なくとも次を固定する。

- forwardでframeを飛び越えた`(begin,end]`のeventをcanonical orderで一度ずつ発火する。
- forwardのloop跨ぎで各loop ordinalのeventを欠落／重複なく発火する。
- reverse `suppress`でevent非発火かつcursorだけがendへ進み、`emit_reverse`で`[end,begin)`を逆canonical orderで一度ずつ発火する。
- seekでGameplay／Presentation eventとroot deltaを抑止し、destination後のforwardで過去eventをbackfillしない。
- Editor scrubでRuntime event／root proposalを発行せず、live instance clock／cursor／state hashを変更しない。
- forward、reverse、seek、Editor scrubの各caseをparallel実行、retry、Replay再生してevent sequence、pose hash、root-motion hashが再現する。

Editor／AI fixtureはGraph diff、timeline preview、import／reimport conflict、question-required intent、unsupported operation、diagnostic remediation、manual／AI equivalenceを検査する。private Backend conformanceは同じSource fixtureからEngine-owned pose／event／diagnosticを生成し、public contractにVendor差が漏れないことを確認する。measurementとcapacity promotionは[Runtime performance／capacity](../04-runtime/performance-capacity.md)、evidence envelopeは[AI Verification／Provenance](../01-governance/ai-verification-provenance.md)を使用する。

Blend Space fixtureは1D boundary／outside range、2D inside／edge／outside hull、duplicate／collinear／cocircular sample、Source並替え、triangulation tie、weight finite／normalization、wrap seam、normalized phase／marker sync、leader tie、Gameplay event source一意性、Presentation dominant sample、unqualified root-motion blend rejectを固定する。Graph fixtureはcondition AST、transition curve／interrupt／sync、nested State Machine、typed Port、unknown Node kind、layout-only changeでsemantic hash不変を検査する。

IK fixtureは四spaceのtarget／pole、generation差、two-bone reach／pole degeneracy／joint limit、look-at axis／limit、複数request order、zero／one／intermediate weightを固定する。Foot placement fixtureはT30 Query Binding、T50 generic execution、T60 Result Binding、T80 consumptionを同じintervalへ束縛し、stale scene／pose／Transform generation、subject motion bound超過、truncated／failed result、retryでfailure policy、query count、pose／clock／event hashが再現することを検査する。

AI-readable fixtureはCapability／Graph／Debug Projectionのbounded query、Stable ref、type／unit、completeness／gap、disabled reasonを検査し、Human／AI／CLIが同じProjection revisionとChangeSetへ収束すること、layout／locale／selection差でsemantic targetが変わらないこと、current Operation `[]`からCommandが生成されないことを固定する。

Definition Closure fixtureはRuntime／semantic Type、Authoring Tool、Debug Bindingのcurrent集合と状態を別々に照合する。未materialize Typeをcurrent MCD memberとして返す、planned actionからOperation IDを生成する、Operation closureの一部だけを登録する、Debug Bindingなしでlive／recorded traceを成功表示する、一つのclosure成立から他を自動activateするcaseを拒否し、Source／Project／Runtime stateを不変にする。

次を採用しない。

- Vendor animation型、job、pointer、archiveのpublic API化
- clip／joint名やfunction名を使うRuntime dispatch
- Rendering visibilityからauthoritative clock／event cursorを停止すること
- Animation TransformとPhysics resolved Transformの二重write
- IK／foot placementからのlive Physics query
- Blend Spaceの補間、同期、event、root motionをBackend defaultへ委ねること
- Graph layout、display name、外部Engine node名からAI Commandを生成すること
- Debug projectionからlive Animation instanceを変更すること
- silent bind-pose fallbackでGameplay event／root motionを継続すること
- Runtime phase／shared capacity、Physics／Navigation／Collision／Rendering責務の再所有
