# Miraikanai Engine 2D／3D機能計画

- 文書版: 1.0
- 作成日: 2026-07-19
- 対象: 2D／3D Game Runtime、Editor、Asset pipeline、AI Authoring
- 状態: プロジェクト公式の機能範囲と段階設計
- 上位文書: [AIネイティブ独自ゲームエンジン 設計計画書](./2026-07-18-ai-native-game-engine-authoring-design.md)
- 基盤規約: [Miraikanai Engine 基盤アーキテクチャ規約](./2026-07-19-engine-foundation-architecture-design.md)

## 1. 結論

2Dと3Dは同格のFirst-class runtimeとし、2Dを「奥行きが0の3D」として実装しない。Asset、Input、Audio、UI、Script、AI Authoring、Build、Save、Diagnosticsは共有し、描画、Physics、Navigation、Animation authoringは専用Subsystemを持つ。

すべてのkernelを自作する方針は採らない。Miraikanai Engineが独自に所有するのは、公開Capability、正規data model、Editor UX、AI command、validation、lifetime、serialization、fallbackである。Collision solver、Navmesh polygon処理、Script VM、GPU heap suballocationなど、検証済みLibraryが安全性と開発速度を大きく改善する部分はAdapter内で利用する。

## 2. Capability成熟度

Product Phaseと機能の成熟度を混同しないため、各機能を次の四段階で管理する。

| Level | 意味 | 合格条件 |
|---|---|---|
| C0 Foundation | API、schema、validation、debug表示が存在 | Unit／conformance test |
| C1 First Playable | 一つの完成した2Dまたは3D縦切りで利用可能 | 開始から終了までplay可能 |
| C2 Production | 品質tier、profiling、authoring、fallbackを備える | Reference sceneの性能・安定性gate |
| C3 Advanced | 大規模World、高品質表現、特殊genre向け | 専用BenchmarkとDomain Pack |

初期MVPはC0を完成させた後、**2D縦切りをC1、3D compact action arenaをC1**へ順に到達させる。両方を同時に実装しない。AI Authoringの基盤検証を優先する最初の縦切りは2Dとし、Direct3D 12 rendererのProduction検証を行う第二の縦切りを3Dとする。

## 3. 全機能に共通する規約

### 3.1 座標、単位、角度

| 空間 | 規約 |
|---|---|
| 3D World | 右手系、+Y up、meter、radian |
| 3D object forward | +Z。Camera view forwardは−Z |
| 2D World | +X right、+Y up、float logical pixel |
| 2D Physics boundary | `pixels_per_meter`でmeterへ変換。既定値100 |
| UI | origin top-left、+X right、+Y down、DIP |
| Texture UV | origin top-leftのAsset APIへ正規化 |

ScaleをPhysics bodyへ直接適用しない。Collider生成時に固定scaleをbakeし、実行中のvisual scaleとphysics scaleを分離する。単位変換はAdapter境界だけで行い、暗黙変換を禁止する。

glTF importではglTF 2.0の右手系、+Y up、meter、radianを保持する。Objectのfront convention差はimport metadataへ記録し、mesh dataを無理由に再変換しない。

### 3.2 Color、HDR、Alpha

- Lighting計算はscene-linear colorで行う。
- Base color／emissiveなど色textureはsRGBとしてdecodeする。
- Normal、roughness、metallic、depthなどdata textureはlinearとして読む。
- 2D spriteとUIはpremultiplied alphaを正規形式とし、import時に変換する。
- HDR scene bufferはC1でFP16を使用する。
- SDR出力はsRGB、HDR10出力はC2で追加する。
- Tone mapping、exposure、white balanceはCamera／Post Process Profileへ保存する。

### 3.3 時間

- Simulationはfixed tick、renderは可変tick＋interpolationとする。
- 初期fixed tickは60 Hz、最大catch-upは4 step。
- Game pause、UI time、cinematic time、physics timeを明示的なclock domainに分ける。
- Animation curve、particle、audio schedulingはsecondを正規単位とする。
- Randomnessはsystem別seed streamを使い、global random generatorを禁止する。

### 3.4 AIへ公開する編集単位

AIはGPU command、Physics native body、Navmesh polygon、raw shader resourceを直接操作しない。各Subsystemが次を提供する。

```text
CapabilitySchema
AuthoringComponent
TypedCommand
Validator
Preview
RuntimeCompiler
Diagnostics
```

例:

- `SetLightProfile`
- `CreateParticleEmitter`
- `ConfigureNavAgent`
- `AssignPhysicsMaterial`
- `CreateMaterialGraph`
- `BakeEnvironmentLighting`

CommandはStable ID、base project revision、型付き引数、precondition、推定costを持つ。Engineが生成したpreviewとDiffだけをCommit候補にする。

## 4. 共通Engine機能

| Capability | C1 | C2 | C3 |
|---|---|---|---|
| Window／display | Windowed、borderless、resolution、DPI | HDR、multi-monitor、quality auto-detect | Multi-view特殊display |
| Input | Keyboard、mouse、GameInput gamepad、action map、rebinding | Chord、context、haptics、accessibility preset | Specialized device |
| World | Stable ID、transform、component、prefab-like templateではないcomposition recipe | Streaming cell、dependency graph | Large-world origin rebasing |
| Asset | Import、content hash、cook、cache、hot reimport | Background streaming、LOD cook、remote build cache | Distributed cook |
| Save | Versioned save schema、checkpoint、atomic save | Slot、cloud adapter、partial world state | Large-world partition save |
| Audio | XAudio2 voice、bus、spatial emitter、streaming music | X3DAudio、reverb zone、ducking、profiler | Geometry acoustics |
| UI／Text | Layout、style、focus、gamepad nav、DirectWrite text | Localization、IME、screen reader semantics、animation | Advanced vector effects |
| Script | Luau strict、Capability API、quota、debug | Hot reload、profiler、module permission | Restricted runtime content script |
| Native code | Project Capability C++、isolated build、test | Incremental build、source-level profiling | Stable external SDKは1.0後 |
| Gameplay logic／AI | Typed state machine、Rule Graph、Script behavior、seeded random | Blackboard、perception、behavior tree／utility composition | Large-agent simulation policy |
| Camera | 2D／3D camera、viewport、blend、shake | Cinematic track、split view | Multi-camera recording |
| Diagnostics | Log、trace、memory、frame、capture | In-editor profiler、comparison baseline | Automated bottleneck advisor |
| Test | Unit、headless simulation、golden state | Image／audio／performance regression | Large scenario automation |
| Build | Development／Profile／Shipping、signed manifest | Packaging、crash symbols、patch chunks | Multi-platform farm |
| Collaboration | ChangeSet history、text-diffable source、conflict検出 | Git status／diff連携、Asset lock adapter | Remote multi-user authoring |
| Networking | 対象外 | Transport abstractionの調査 | Authoritative multiplayer |

NetworkingをC1/C2へ暗黙に含めない。FPS/TPSというgenreはC1でsingle-playerとして成立させ、multiplayerは独立したC3計画とする。

## 5. 2D RuntimeとEditor

### 5.1 2D Renderer

2Dは専用`CanvasRenderer`を持つ。3D opaque passへquadを混ぜる設計にしない。

#### C1

- Sprite、sprite sheet、texture atlas
- Region、flip、pivot、modulate color
- Nine-slice、tiled sprite
- Layer、explicit order、Y-sort group
- Dynamic batching、texture／sampler grouping
- Orthographic camera、pixel snap、integer scale option
- Parallax layer
- Scissor、mask、blend mode
- Polyline、polygon、basic shape
- 2D normal map
- Directional／point 2D light
- Signed-distance-field 2D shadow occluder
- Sprite material graphの制限subset
- Draw-call、overdraw、atlas occupancy debug view

#### C2

- Instanced sprite、GPU culling
- 2D light clustering
- Soft shadow、shadow atlas
- Vector path tessellation cache
- Render-to-texture、multi-camera composition
- Tilemap専用chunk renderer
- 2D post process stack

Blendの既定はpremultiplied alphaとする。Straight alpha Assetはimport時に変換し、Materialが明示要求した場合だけ別pipelineを使う。

### 5.2 Tilemap

Tilemapは単一巨大arrayではなく、変更・streaming・culling単位のchunkへ分割する。

#### C1

- Multiple tile layer
- Atlas tile、animated tile、collision shape、navigation tag
- 既定32×32 tileのchunk。Projectで変更可能
- Rectangle／line／fill／stamp／selection
- Rule-based terrain paint
- Chunk単位mesh rebuild
- Collider merge
- Paletteとcustom metadata

#### C2

- World streaming
- Background async chunk cook
- Procedural rule preview
- Occlusion／light mask
- Runtime editを許可するtile command

AIはtile IDの巨大配列を直接生成せず、region、rule、seed、constraintを持つ`TileLayoutCommand`を提案する。Engineが実tileへ展開し、接続不良、到達不能、過大変更を検証する。

### 5.3 2D Physics

Box2D 3.xを`Physics2DBackend`内で利用する。

#### C1

- Static／kinematic／dynamic body
- Circle、capsule、box、polygon、chain shape
- Sensor、collision layer／mask
- Distance、revolute、prismatic、weld joint
- Physics material
- Ray cast、shape cast、overlap query
- Contact begin／end、trigger event
- Kinematic character motor
- Fixed 60 Hz、substep設定
- Collider、contact、sleep state debug draw

#### C2

- One-way platform
- Continuous collision quality profile
- Buoyancy／area force
- Ragdoll-like joint authoring
- Large tile collider streaming
- Physics query profiler

Physics eventはtick末にStable IDへ変換して配送する。Box2D pointer、body ID、callback中のWorld変更をGame APIへ露出しない。Event orderingとduplicate抑止をAdapter conformance testで固定する。

### 5.4 2D Navigation

2Dではgrid navigationとpolygon navigationを別backendとして提供する。

#### C1

- Tilemapからwalkable grid生成
- A* query、cost layer、blocked cell
- Agent radiusを考慮したclearance
- Reachability validation
- Path debug overlay

#### C2

- Polygon navigation region
- Dynamic obstacle update
- Hierarchical pathfinding
- Flow field（Simulation Domain Pack）
- Local avoidance

AIがlevelを作る際はspawnからgoalまで、必須interaction point間、escape routeの到達可能性を自動Testする。

### 5.5 2D Animation

#### C1

- Flipbook
- Property track
- Curve／event track
- Tween
- State machineとtransition condition
- Blend parameter
- Preview、scrub、onion skin

#### C2

- Cutout skeleton
- IK chain
- Blend space
- Animation retarget profile
- Root-like 2D displacement

Animation eventは任意関数名を文字列で呼ばず、登録済みtyped Gameplay Eventを発行する。

### 5.6 2D縦切りの合格Scene

最初のC1はtop-down action gameとする。

- Title、settings、play、result
- 一つのtilemap level
- Player movement、attack、damage
- 3 enemy archetype、spawn rule
- Goal、checkpoint、save／load
- 2D light、particle、camera shake、music／SFX
- AIが自然言語からlevel、rule、UI、Scriptを生成
- 人間がInspectorとScriptで修正後、AIが差分を保持して再編集
- 1080p60、10,000 visible sprite、500 dynamic physics bodyのReference stress scene

Stress sceneはgameplay sceneと分離し、同時にすべてを要求しない。Reference hardwareでP95 frame time 16.67 ms以内、GPU／CPUいずれも継続的budget超過なしを合格条件とする。

## 6. 3D RuntimeとEditor

### 6.1 MeshとWorld Rendering

#### C1

- Static mesh、skinned mesh、morph target
- glTF 2.0 import
- Vertex／index buffer cook
- PBR metallic-roughness material
- Normal、occlusion、emissive
- Camera frustum culling
- Manual／generated LOD
- Hardware instancing
- Decal
- Transparent forward pass
- Debug wireframe、bounds、normal、material view

#### C2

- GPU-driven instance culling
- Occlusion culling
- HLOD
- Meshlet／indirect pathはfeature query後に任意
- Streaming cell
- Terrain、foliage、water
- Reflection probe

glTFはinterchange formatであり、Runtime source of truthにしない。Import後は独自Asset schemaへ変換し、cook時にGPU向けformatへ変換する。

### 6.2 Renderer architecture

Render GraphをC0で作り、resource lifetime、state transition、queue、aliasingをcompileする。

- **C1**: Forward+ opaque／masked、forward transparent、depth prepassを基本とする。
- **C2**: Deferred opaque lightingを追加し、Material／quality profile単位でForward+と選択するHybrid rendererにする。
- Compute、copy、direct queueをRender Graphが依存関係からscheduleする。
- Render pathは同じMaterial IRとlighting data contractを使う。
- Ray tracingはC3 optional pathであり、raster fallbackを必須とする。

Forward+を最初にする理由は、透明、MSAA、2D／3D合成を一つの実装で成立させ、最初の縦切りを過剰に広げないためである。多数lightと複雑なopaque materialの実測が必要になった段階でDeferred pathを加える。

### 6.3 LightとShadow

物理量を使用し、無単位の「強さ」だけを公開しない。

| Light | 単位 | C1 | C2 |
|---|---|---|---|
| Directional | lux | 1主光源、複数補助 | 複数shadow caster policy |
| Point | lumen | 対応 | IES profile |
| Spot | lumen／candela | 対応 | IES profile |
| Rectangle／Disk | lumen／nit | 非対応 | LTC近似 |
| Emissive surface | nit | 見た目のみ | probe／GIへの寄与 |

Shadow:

- DirectionalはCascaded Shadow Map。
- Spotはatlas 2D shadow。
- Pointはcubemap shadow。
- Resolution、cascade、bias、update frequencyをLight Quality Profileで制御する。
- Shadow atlasはbudgetを持ち、priority、screen influence、distanceで割り当てる。
- 無効なbias、過大resolution、atlas overflowをValidatorで拒否または明示fallbackする。

AIはlightを無制限に追加できない。Project Profileのvisible light数、shadow caster数、shadow texel budgetをChangeSet validationで検査する。

Global illuminationは次に段階化する。

- C1: direct lighting＋environment IBL。Dynamic GIを前提にしない。
- C2: offline baked lightmap、irradiance probe volume、reflection probe。Static／stationary／dynamic mobilityを明示する。
- C3: hardware非依存のdynamic diffuse GIを研究し、ray tracingは品質向上用の任意backendとする。

Lightmap、probe、IBLはDerived Assetとし、geometry、material、light、baker version、quality settingsのhashから再生成する。AIは`BakeLighting`を要求できるが、bake結果のtexelを直接生成・編集しない。

### 6.4 Sky、Atmosphere、Fog、Cloud、Environment Lighting

これらを別々の装飾機能ではなく、共通の`EnvironmentProfile`として連携させる。

#### C1

- Solid／gradient sky
- HDR environment cubemap
- Directional sun／moon link
- Image Based Lightingのdiffuse irradianceとspecular prefilter
- Height fog＋exponential distance fog
- Automatic／manual exposure
- Environment bake command

#### C2

- Physically based atmosphereのtransmittance、multi-scattering、sky-view LUT
- Aerial perspective
- Froxel volumetric fog
- Local fog volume
- Volumetric cloud ray march
- Weather coverage／density map
- Temporal reprojection、blue-noise jitter
- Dynamic skyからIBLを段階更新
- Cloud shadow map

#### C3

- Multiple atmosphere body
- Advanced weather front
- Large-world cloud streaming
- Hardware ray traced environment option

Quality fallback:

| Tier | Atmosphere | Fog | Cloud |
|---|---|---|---|
| Low | Precomputed sky cubemap | Height／distance fog | 2D cloud layer |
| Medium | Atmosphere LUT | Half-resolution froxel | Reduced-step volumetric |
| High | Atmosphere LUT＋dynamic IBL | Full feature froxel | Temporal volumetric＋shadow |

Cloudとvolumetric fogはtransparent lighting、depth、motion vector、exposureとRender Graph上で明示依存する。履歴bufferが無効になるcamera cut、resolution変更、weather jumpではtemporal historyを破棄する。

### 6.5 Post Processing

#### C1

- Exposure
- Tone mapping
- Bloom
- Color grading LUT
- Vignette
- Depth of fieldの簡易版
- FXAA、TAA
- Motion vector infrastructure

#### C2

- Motion blur
- High-quality depth of field
- Screen-space ambient occlusion
- Screen-space reflection
- Upscaling adapter
- HDR10 output

Effectの順序はPost Process Graphで固定し、AIは登録済みnodeと範囲検証済みparameterだけを編集する。Temporal effectにはhistory invalidation contractを必須とする。

### 6.6 3D Physics

Jolt Physicsを`Physics3DBackend`内で利用する。

#### C1

- Static／kinematic／dynamic rigid body
- Box、sphere、capsule、convex hull、triangle mesh
- Collision layer／mask、sensor
- Ray、shape cast、overlap
- Contact／trigger event
- Character controller
- Fixed step、sleep、continuous collision profile
- Constraintの基本subset
- Debug draw、island、contact profiler

#### C2

- Ragdoll
- Vehicle
- Destructible constraint setup
- Heightfield／large static world streaming
- Soft bodyは検証後に任意

Dynamic bodyへtriangle mesh colliderを許可しない。非uniform scaleはcook時にbakeする。Runtimeでscale変更されたcolliderは自動再cookせず、明示errorにする。

Physics determinismは同一version／platform／thread設定のreplay範囲で検証する。Network lockstep相当のcross-platform determinismは保証しない。

### 6.7 3D Navigation

RecastでEditor／cook時にNavmeshを構築し、DetourでRuntime queryを行う。

#### C1

- Walkable slope、height、step、radiusを持つAgent Profile
- Geometry収集filter
- Tile Navmesh build
- Point projection、path query、straight path
- Off-mesh link
- Area typeとcost
- Navmesh overlay、tile、failed query debug
- Spawn／goal reachability test

#### C2

- TileCacheによるdynamic obstacle
- Partial tile rebuild
- Crowd local avoidance
- Hierarchical long-distance query
- Streaming tile
- Jump／climb／door linkのtyped action

#### Runtime生成の制限

初期RuntimeではAIがpolygonを生成しない。許可するのは既存tile上のcost、off-mesh link、dynamic obstacle、goalの変更だけとする。Navmesh rebuildはEditorまたはserver-controlled bounded regionに限定する。

Navmesh build resultはDerived Assetであり、source geometry、Agent Profile、builder version、settings hashから再生成できる。

### 6.8 3D Animation

ozz-animationをsampling／compression primitiveとしてAdapter内で利用し、Animation Graphは独自に実装する。

#### C1

- Skeleton、clip、compressed track
- Sampling、cross-fade、layer、mask
- State machine
- Parameter／typed event
- Root motion
- Skinning
- Preview、scrub、transition debug

#### C2

- 1D／2D blend space
- Additive animation
- IK、look-at、foot placement
- Retarget profile
- Motion warping
- Ragdoll blend

Game ruleはclip名文字列へ依存せず、Animation Capabilityとsemantic tagを参照する。Root motionの所有者をCharacter MotorまたはAnimationのどちらか一方に明示し、二重適用を禁止する。

### 6.9 3D縦切りの合格Scene

第二C1はsingle-player third-person compact action arenaとする。

- Title、settings、play、result
- Player locomotion、camera、attack、damage
- 3 enemy archetype、Navmesh追跡
- One quest goal、checkpoint、save／load
- Directional、point、spot lightとshadow
- Sky／IBL／height fog
- PBR material、particle、animation、spatial audio
- AIが自然言語からarena、rule、UI、Script、Asset設定を生成
- Inspector、Scene View、Graph、Script、C++ Capabilityの手動修正
- 2,000 visible mesh instance、100 dynamic rigid body、50 navigation agentのstress scene

Reference hardwareで1080p60、P95 frame time 16.67 ms以内、Game runtime CPU memory 2 GiB以内、GPU Project budget内を合格条件とする。

## 7. Particle／VFX

2D／3Dで共通のauthoring graphとcurve systemを使い、Renderer backendだけを分ける。

### C1

- Emitter shape
- Rate／burst
- Lifetime、velocity、acceleration、drag
- Color／size／rotation over life
- Sprite sheet animation
- World／local space
- 2D／3D billboard
- Trailの基本版
- CPU simulation
- Seed固定preview
- Bounds、particle count、overdraw debug

### C2

- GPU compute simulation
- Mesh particle
- Depth／SDF collision
- Light emissionのbudget付きsubset
- Event-driven child emitter
- Ribbon
- Vector field
- LOD、distance culling、fixed budget

GPU particleはvisual effectであり、gameplayの正規状態やSaveへ使用しない。Gameplay判定が必要なeffectはCPUのdeterministic gameplay entityを別に持つ。AIが指定できるparticle上限、spawn rate、light count、collision modeはValidatorでProject budgetに制限する。

## 8. ShaderとMaterial

### 8.1 Compiler pipeline

- HLSL 2021をDXCでcompileする。
- Shader Model 6.6をReference baselineとする。
- Root Signature 1.1を使用する。
- DXC、Agility SDKは検証済みStable versionをlock fileへ固定する。
- Preview Agility SDKをShipping Buildで使用しない。
- Source hash、include hash、define、compiler version、target、optimizationからcache keyを作る。
- Reflection結果を独自`ShaderInterface`へ変換し、RuntimeがDXC型へ依存しないようにする。

### 8.2 Material authoring

| Level | 編集方式 |
|---|---|
| Level 0 | 「濡れた石」「発光する魔法」など自然言語からMaterial intentを生成 |
| Level 1 | Material Graphとparameter |
| Level 2 | Safe custom expression subset |
| Level 3 | Project HLSL module |

Material Graphはtyped IRへcompileし、そこからHLSLを生成する。AIは原則としてGraphとparameterを編集し、HLSL生成はGraphで表現できない要件か明示要求がある場合に限る。

### 8.3 Validation

- Graph cycle、型、resource数、sampler数、instruction estimate
- 禁止include、filesystem path、unbounded loop
- Binding collision、root signature compatibility
- Shader compile、Pipeline State作成
- Reference material preview
- GPU timeoutを避けるisolated compiler process

Shader compile失敗時に直前のvalid PSOを保持するのはEditor previewだけとし、Shipping buildは失敗artifactを含めない。

## 9. Asset pipeline

| Asset | Source／Interchange | Runtime |
|---|---|---|
| Texture | PNG、JPEG、EXR、KTX2、DDS | BC format、mip、streaming metadata |
| 3D | glTF 2.0 | 独自mesh／skeleton／animation package |
| 2D atlas | Image＋atlas metadata | Packed page＋sprite table |
| Audio | WAV、FLAC等 | Platform codec／stream chunks |
| Shader | HLSL／Material Graph | DXIL＋ShaderInterface |
| Nav | Source geometry＋profile | Tiled nav data |
| Physics | Shape authoring data | Cooked shape data |

OpenUSDはC0～C2では採用しない。C3でDCC collaboration interchangeが必要になった場合だけ、別ADRとvertical prototypeを通して採否を決める。採用する場合もProject source of truthにはしない。OpenUSDは一般的なGUID systemや完全なrigging solutionを提供するものではなく、MiraikanaiのStable ID、GameSpec、Behavior Contractを置き換えない。

Importerは別Processで実行し、networkなし、許可pathだけ、timeout、memory capを持たせる。AI生成Assetも同じstaging、license metadata、content safety、import validationを通す。

## 10. Editor機能との対応

### 共通Panel

- Scene／Canvas View
- Hierarchy／Outliner projection
- Inspector
- Asset Browser
- Game／Rule Graph
- Timeline／Animation
- Material／VFX Graph
- Navigation／Physics debug
- Build／Playtest
- Profiler
- Diff／History
- AI Partner

Panelは上下左右edgeでresizeでき、tab docking、split、floating、入替え、複数monitor、pinを持つ。AI Partnerは通常Panelと同様にdock可能で、常時表示をworkspaceごとに保存する。

Editor shellはDear ImGui 1.92.8-dockingを描画基盤に使うが、操作設計とaccessibilityは独自に保持する。

- 1920×1080、2560×1440、100／125／150／200% DPIをlayout test対象にする。
- Mouseだけでなくkeyboard focus、tab order、shortcut、command paletteですべての主要操作へ到達できる。
- Windows UI Automationへ公開するsemantic treeをImGui widget IDと分離して保持する。
- Focus indicator、high contrast、color以外の状態表現、remappable shortcutを必須にする。
- Scene中央領域を常に残し、Panelが画面外へ消えた場合は`Reset Workspace`で回復できる。
- Modal dialogを長時間taskの進捗表示に使わず、cancel可能なbackground jobとして表示する。
- AI提案、Engine validation、実際にCommitされるDiffを視覚的に区別する。

### 保存可能Workspace

- AI Creator: AI Partner、Game Brief、Preview、承認Diffを中心
- Production: Scene、Hierarchy、Inspector、Asset、AI Partner
- Level: Scene、Outliner、Navigation、Physics、AI Partner
- Scripting: Code、API、Test、Console、AI Partner
- Rendering: Scene、Material、VFX、Lighting、GPU profiler
- Debug: Game、Log、Profiler、History、AI Partner

初心者用AI Creatorも同じWorld ModelとChangeSetを使う。上級者向け機能を隠すだけで、別の簡易Project formatや別Editorを作らない。

## 11. Quality ProfileとFallback

| 項目 | Low | Medium | High |
|---|---|---|---|
| Renderer | Forward+ | Forward+ | Hybrid選択可 |
| Shadow | 少数、低解像度 | CSM＋atlas | 高解像度＋area approximation |
| Atmosphere | Cubemap | LUT | LUT＋dynamic IBL |
| Fog | Height | Half-res volumetric | High-quality volumetric |
| Cloud | 2D layer | Reduced volumetric | Full temporal volumetric |
| Particle | CPU／低上限 | GPU standard | GPU high budget |
| Reflection | Probe | Probe＋SSR | Probe＋SSR＋optional RT |
| Anti-alias | FXAA | TAA | TAA／upscaler |

起動時にhardware featureとmemory budgetを検査し、Projectの最低Tier未満なら黙って品質を落とさず、理由と不足Capabilityを表示する。Userが許可した場合だけ下位Tierへfallbackする。

## 12. Domain Packとの境界

Coreはgenreを知らない。次はDomain Packで提供する。

- FPS／TPS: Character weapon、aim、camera、hit reaction
- RPG／Action RPG: Attribute、ability、item、quest、dialogue
- Simulation: Time scale、agent、economy、flow field、large data table
- 2D Action: Platform motor、one-way platform、tile rule
- Strategy: Selection、formation、fog of war、large-agent navigation

Domain PackはCore Capabilityをcompositionし、C++継承階層へgenreを埋め込まない。Packはschema、template、validator、AI vocabulary、test scenarioを一つのversioned packageとして配布する。

## 13. 実装順序

1. Foundation、ID、memory、Result、Job、diagnostics
2. ChangeSet、World Model、Authoring Service、headless test
3. Editor shell、Windows／D3D12 device、Render Graph、Asset cooker
4. 2D Canvas、Input、Audio、UI、Luau、Box2Dとmanual vertical slice
5. TypeScript AI Orchestrator、named-pipe IPC、OpenAI Provider、AI editing loop
6. 外部MCP、Codex／Claude Plugin
7. 3D mesh、PBR、Forward+、Jolt、Recast、animation
8. 3D compact action arena
9. Production lighting、atmosphere、volumetric fog／cloud、GPU VFX
10. Hybrid deferred path、streaming、terrain／foliage
11. Domain Pack拡張
12. 制限付きRuntime generation
13. Multiplayer／large world／ray tracingは個別設計後

2Dと3Dの全機能を先に並行実装しない。共有基盤→2D complete loop→3D complete loopの順で、毎段階にplayable resultを置く。

## 14. 機能完了の定義

機能は「画面に出た」だけでは完了しない。次をすべて満たす。

1. Authoring schemaとtyped commandがある。
2. C++ semantic validatorとbudget validatorがある。
3. Editorで作成、編集、preview、undoできる。
4. AIがCapabilityを発見し、ChangeSetとして提案できる。
5. Runtime packageへcookできる。
6. Debug visualizationとtelemetryがある。
7. Unit、conformance、integration、performance testがある。
8. Invalid input、OOM、device lost、missing assetのfailure behaviorが定義される。
9. Quality tierとfallbackが定義される。
10. Reference sceneでbudgetを満たす。
11. User documentationとAI tool descriptionが同じschema versionを参照する。

## 15. 主要リスクと確定対策

| リスク | 対策 |
|---|---|
| 2Dが3D実装へ従属して使いにくい | 専用Canvas、2D Physics、2D Navigation、専用Editor tool |
| 多genre対応でCoreが巨大化 | Capability＋Domain Pack、継承ではなくcomposition |
| High-end表現で低Tierが破綻 | Quality Profileと明示fallback |
| AIが過大なLight／Particle／Nav変更を作る | Cost宣言、budget validation、preview、approval |
| Physics libraryへProject dataが固定 | Engine componentとAdapter、独自serialization |
| Navmeshがsource of truthになる | Derived Assetとして再生成 |
| GPU particleがgameplayを壊す | Gameplay stateとの分離 |
| Shader自由度がdriver hangを招く | Graph優先、隔離compile、検証、budget |
| 2D／3D同時開発で完成しない | 共有Foundation後に2D、次に3Dの縦切り |
| 有名Engineの模倣になる | 独自World Model、ChangeSet、Capability、Editor projectionを維持 |

## 16. 一次資料

- [Direct3D 12 Programming Guide](https://learn.microsoft.com/en-us/windows/win32/direct3d12/directx-12-programming-guide)
- [Direct3D 12 Memory Management Strategies](https://learn.microsoft.com/en-us/windows/win32/direct3d12/memory-management-strategies)
- [DirectX Shader Compiler](https://github.com/microsoft/DirectXShaderCompiler)
- [DirectX Agility SDK](https://devblogs.microsoft.com/directx/directx12agility/)
- [GameInput Introduction](https://learn.microsoft.com/en-us/gaming/gdk/docs/features/common/input/overviews/input-overview)
- [XAudio2 Programming Guide](https://learn.microsoft.com/en-us/windows/win32/xaudio2/programming-guide)
- [DirectWrite](https://learn.microsoft.com/en-us/windows/win32/directwrite/direct-write-portal)
- [glTF 2.0 Specification](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html)
- [KTX 2.0 Specification](https://registry.khronos.org/KTX/specs/2.0/ktxspec.v2.html)
- [OpenUSD Introduction](https://openusd.org/release/intro.html)
- [Box2D Documentation](https://box2d.org/documentation/)
- [Jolt Physics Documentation](https://jrouwe.github.io/JoltPhysics/)
- [Recast Navigation Repository](https://github.com/recastnavigation/recastnavigation)
- [Luau Embedding](https://luau.org/embedding)
- [Luau Sandboxing](https://luau.org/sandbox)
- [ozz-animation Documentation](https://guillaumeblanc.github.io/ozz-animation/)
- [Godot Dedicated 2D Engine](https://docs.godotengine.org/en/stable/about/list_of_features.html#dedicated-2d-engine)

既存Engineの資料はcoverage確認にだけ使う。Miraikanai Engineの型、Scene、Editor command、serialization、lifecycleを既存製品へ合わせる根拠にはしない。
