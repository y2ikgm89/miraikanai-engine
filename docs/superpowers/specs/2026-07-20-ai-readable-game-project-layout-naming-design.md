# Miraikanai Engine AI可読Game Project配置・命名規約

- 文書版: 1.0
- 作成日: 2026-07-20
- 最終更新日: 2026-07-20
- 対象: Miraikanai Engineで制作するGame ProjectのDirectory、File、Asset、Authoring Document、Native Game Code、AI生成物、Import、Staging、Promotion
- 対象外: Engine repository内部のModule／CMake／C ABI命名、Runtime package内部の物理配置、Third-party sourceそのものの内部命名
- 状態: プロジェクト公式の規範設計
- Engine命名正本: [Miraikanai Engine AI可読命名・技術識別子規約](./2026-07-20-ai-readable-engine-naming-convention-design.md)
- Authoring正本: [Miraikanai Engine Authoring Model／Project State規約](./2026-07-19-authoring-model-project-state-design.md)
- Asset正本: [Miraikanai Engine Asset Pipeline／Content Package規約](./2026-07-19-asset-pipeline-content-packaging-design.md)
- Import／AI正本: [Miraikanai Engine Asset Import／AI Authoring／Editor UXアーキテクチャ規約](./2026-07-20-asset-import-ai-authoring-editor-ux-design.md)
- AI権限正本: [Miraikanai Engine AI実装・保守ガバナンス規約](./2026-07-19-ai-engine-development-governance-design.md)
- Game System正本: [Miraikanai Engine Game System／AI Code Generationアーキテクチャ規約](./2026-07-20-game-system-ai-codegen-architecture-design.md)

## 1. 結論

Miraikanai EngineのGame Projectは、**人間とAIで別の最終Directoryを持たない**。人間、AI、Importer、Editor、CLIは同じ正規Project構造、同じ命名Policy、同じStable ID、同じValidation、同じPromotion Gateを使用する。AI由来であることは`ai_generated/`等の最終Pathで表さず、Asset metadata、Provenance、ChangeSet、Receiptで表す。

ただし、AIは正規ProjectへDirectoryまたはFileを直接作成、上書き、移動、削除しない。AIは意味を型付きProposalとして提出し、信頼済みGatewayが次の順で処理する。

```text
Intent／Requirement
  -> typed Proposal
  -> .mirakan/staging
  -> naming／schema／semantic／rights／safety／budget validation
  -> Preview／Diff／dependency closure
  -> Policyに必要なApproval
  -> trusted Gatewayによるatomic promotion
  -> canonical Project source
```

Projectで自由に増やせるのは、許可済み配置Profile内の**対象Directory**と**Source File**だけである。Top-level Directory、Authoring Document path、metadata path、Staging内部名、Stable ID、Derived Artifact pathはEngineが決定する。

## 2. 規範語と優先順位

本書の「必須」「禁止」は機械検査するMUST／MUST NOT、「推奨」はMiraikanai公式Project profileのSHOULD、「許可」はMAYを意味する。

命名と配置の決定権を次の順に固定する。

1. OS、Filesystem、Source形式、Toolchainの必須名と予約名
2. Engine命名正本のProduct stem、case、MCD suffix、曖昧語、Identity規則
3. Authoring、Asset、AI Governance各正本のStable ID、ChangeSet、Staging、Promotion規則
4. 本書のGame Project配置ProfileとSource Asset名
5. Project固有のversioned extension

本書はGame ProjectのPathと名前に限り、旧文書に残る次のlegacy表記を置き換える。

| Legacy | 正規形 |
|---|---|
| `mira.project.json` | `mirakan.project.json` |
| `.mira/` | `.mirakan/` |
| Authoring MCDの`scene.json`等 | `scene.mirakan.json`等 |

Legacy pathは移行入力としてだけ認識し、新規作成、AI提案、Template、Example、Test goldenへ出力しない。自動互換aliasを恒久運用しない。

外部Engineの規則は設計比較資料であり、本書を上書きしない。Unreal Engineの型prefix、Unityの`.meta`、GodotのFilesystem構造をそのまま混在させず、MiraikanaiのStable IDとtyped metadataへ統一する。

## 3. 名前、Path、Identity、表示名を分離する

一つのGame Project対象は次の四つを混同しない。

| 概念 | 目的 | 例 | 変更の影響 |
|---|---|---|---|
| Display name | EditorとGame内で人間へ表示 | `忘れられた森` | 自由に変更可能 |
| Technical slug | 検索、Directory、File、Code | `forgotten_forest` | 明示Migration |
| Stable ID | 永続参照 | UUIDv7 | rename／moveで不変 |
| Revision／Version | 同じIdentityの変更またはContract版 | `asset_revision=7` | 名前へ埋め込まない |

Display nameはUTF-8、Unicode NFCとし、日本語を含めてよい。Technical slugはASCII lowercase `snake_case`とする。永続参照はStable IDまたはversion-bound typed referenceを使用し、Path、表示名、配列位置、content hashをIdentityにしない。

Technical slugの語彙は、Project全体で一つの英語Domain用語へ固定する。日本語表示名をローマ字へ機械変換せず、固有名詞以外は英語の意味語を使う。例として表示名`忘れられた森`は`forgotten_forest`とし、`wasureta_mori`、`forgotten_woods`、`lost_forest`を同一対象の別名として混在させない。略語はEngine Naming PolicyまたはProject glossaryのallowlistにあるものだけを使う。

Project manifestは最低限、次を別Fieldとして持つ。

```text
project_id: UUIDv7
project_slug: forgotten_echoes
display_name: 忘れられた残響
project_revision: uint64
project_naming_policy_version: 1
project_naming_policy_hash: SHA-256
```

Project root Directoryは`<project_slug>/`を推奨するが、Project identityではない。root Directoryを移動または改名しても`project_id`を変更しない。

### 3.1 `project_slug`

`project_slug`は次をすべて満たす。

- 正規表現は`^[a-z][a-z0-9]*(?:_[a-z0-9]+)*$`
- 3～48 ASCII byte
- Game作品を識別する具体的な名詞句
- `mirakan`、`engine`、`editor`、`runtime`だけ、またはそれらを製品stemと誤認させる予約形を使わない
- 日付、担当者名、`new`、`old`、`final`、`copy`、revision番号を含めない

## 4. 正規Project root

正規の論理構造を次へ固定する。物理Directoryは必要になった時点でGatewayが作成できるため、空Directoryを保持するための`.keep`を必須にしない。

```text
<project_slug>/
├─ mirakan.project.json
├─ authoring/
│  ├─ game_spec/
│  ├─ worlds/
│  ├─ scenes/
│  ├─ world_topologies/
│  ├─ levels/
│  ├─ spatial_intents/
│  ├─ procedural_worlds/
│  ├─ map_presentations/
│  ├─ system_implementations/
│  ├─ gameplay/
│  ├─ ui/
│  ├─ localization/
│  ├─ visual_styles/
│  ├─ targets/
│  ├─ decisions/
│  └─ tests/
├─ assets/
│  ├─ source/
│  └─ metadata/
├─ native/
│  └─ game/
│     ├─ include/
│     ├─ modules/
│     ├─ source/
│     └─ tests/
├─ .mirakan/
│  ├─ journal/
│  ├─ snapshots/
│  ├─ recovery/
│  ├─ index/
│  ├─ staging/
│  └─ user/
└─ build/
```

Top-level名はclosed setである。AI、User、Pluginは`content/`、`resources/`、`game_data/`、`ai_generated/`、`temp/`等の代替rootを作らない。新しいrootが必要な場合は`ProjectNamingPolicyV1`のversion追加とMigration、Validator、Template、Documentation更新を一つのChangeSetで行う。

### 4.1 Ownership

| Path | Owner | 手動直接編集 | AI直接write |
|---|---|---:|---:|
| `mirakan.project.json` | Project Gateway | 禁止。Editor／CLI経由 | 禁止 |
| `authoring/` | Authoring Gateway | 原則禁止。外部編集は再取込み | 禁止 |
| `assets/source/` | Project／Asset Broker | 許可。Importとして検出 | 禁止 |
| `assets/metadata/` | Asset Gateway | 禁止 | 禁止 |
| `native/game/` | Source Promotion Service | IDE編集を許可 | 禁止。`SourceDeltaV1`だけ |
| `.mirakan/journal/` | Authoring Gateway | 禁止 | 禁止 |
| `.mirakan/snapshots/` | Authoring Gateway | 禁止 | 禁止 |
| `.mirakan/recovery/` | Editor | 禁止 | 禁止 |
| `.mirakan/index/` | Indexer | 禁止 | 読取Queryだけ |
| `.mirakan/staging/` | Trusted Broker | 禁止 | Tool経由のOutputだけ |
| `.mirakan/user/` | Editor／User profile | Editor経由 | 許可されたUser設定Proposalだけ |
| `build/` | Build／Cook system | 禁止 | 禁止 |

「AI直接write禁止」はAIが制作できないという意味ではない。AIは型付きOperation、Asset candidate、SourceDeltaを提出できるが、最終Pathへの書込みとCommitは信頼済みAuthorityが行う。

### 4.2 Portable source、Local state、Derived output

| 分類 | Path | 公式Git profile |
|---|---|---|
| Portable canonical source | manifest、`authoring/`、`assets/source/`、`assets/metadata/`、`native/game/` | 追跡 |
| Local operational state | `.mirakan/recovery/`、`.mirakan/user/` | 除外 |
| Rebuild可能 | `.mirakan/index/`、`.mirakan/staging/`、`build/` | 除外 |
| Bounded authoring history | `.mirakan/journal/`、`.mirakan/snapshots/` | 既定は除外しProject backupへ含める |

GitはProject databaseまたはUndo systemではない。VCS Adapterはcanonical sourceを同一Project revisionとして扱い、`.mirakan/staging`や`build`を昇格させない。

## 5. 共通Directory／File grammar

### 5.1 人間またはAIが提案するTechnical segment

- DirectoryとFile stemはASCII lowercase `snake_case`
- 正規表現は`^[a-z][a-z0-9]*(?:_[a-z0-9]+)*$`
- Directory segmentは1～64 ASCII byte
- File stemは1～96 ASCII byte
- Project-relative logical pathは`/`区切り、240 UTF-8 byte以下
- 空segment、`.`、`..`、absolute path、drive、UNC、NUL、control characterを禁止
- Windows reserved device name、末尾space／dot、Unicode normalization衝突を禁止
- logical path keyはseparator正規化、Unicode NFC、ASCII case fold後に一意

Engineが生成するUUIDv7 segment、Tool必須名の`CMakeLists.txt`、BCP 47 locale tag、MCD suffixはこのhuman-authored segment grammarの明示例外である。例外は各Contractのgrammarで検査し、AIが自由形式の例外を作らない。

### 5.2 Collectionとsubject

- 複数対象を収容するCollection Directoryは複数形: `characters/`、`levels/`、`textures/`
- 一つの対象を表すsubject Directoryは単数technical slug: `hero/`、`forgotten_forest/`
- 不可算の正規Domain語は固定語として許可: `gameplay/`、`audio/`、`localization/`
- 同じ階層で単数形と複数形を混在させない

### 5.3 禁止する名前

次の語を単独Directory、File stem、subject slugにしない。

```text
base
common
data
final
general
global
helper
info
manager
misc
new
object
old
shared
stuff
temp
thing
util
utils
```

Domain上で意味が閉じる複合語は許可する。`base_color`、`common_controls`、`shared_memory`等は登録済みSemantic IDまたはContract名である場合だけ許可する。便利箱としての`common/`、`shared/`、`misc/`は作らない。

品質状態、作業状態、生成元を名前へ入れない。

```text
# 禁止
hero_final_2.png
hero_ai_generated.png
forest_wip.glb
menu_alice_fix.psd
enemy_20260720.wav

# 許可
hero_body_base_color.png
hero_body_base_color_damaged.png
forgotten_forest_ambience_night.flac
```

`placeholder`、`draft`、`approved`、`production_ready`はmetadata stateであり、File renameで遷移させない。

### 5.4 数字

数字suffixは、Domain上の順序または座標である場合だけ許可する。

```text
hero_body_lod_0.glb
intro_take_03.wav
forest_tile_12_08.png
run_frame_0001.png
```

revision、重複回避、担当者の試行回数には使わない。Source LOD、take、frame、tileの意味と桁数は対応metadataが所有し、名前だけから推測確定しない。

## 6. Source Asset配置

`assets/source`はAsset typeだけで全Projectを横断分割せず、**lifecycle ownerを第一軸、Asset kindを後段**にする。あるsubjectのModel、Texture、Animation、Voiceを近接させ、AIが必要なdependency closureをboundedに取得できるようにする。

### 6.1 固定配置Profile

| Owner kind | Path template | 例 |
|---|---|---|
| Character | `characters/<subject>/<asset_kind>/` | `characters/hero/textures/` |
| Environment | `environments/<subject>/<asset_kind>/` | `environments/forgotten_forest/geometry/` |
| Gameplay feature | `gameplay/<feature>/<subject>/<asset_kind>/` | `gameplay/combat/fireball/effects/` |
| UI surface／system | `ui/<surface>/<asset_kind>/` | `ui/inventory/icons/` |
| Cross-cutting audio | `audio/<category>/<subject>/` | `audio/music/main_theme/` |
| Cinematic | `cinematics/<sequence>/<asset_kind>/` | `cinematics/opening/voice/` |
| Font family | `fonts/<family>/` | `fonts/noto_sans_jp/` |
| Locale-specific media | `localization/<locale>/<bundle>/` | `localization/ja-JP/main_story/` |
| Non-shipping reference | `references/<subject>/` | `references/forest_visual_style/` |
| External package original | `external_packages/<package_slug>/original/` | path-scoped exception |

各`AssetPlacementProfileV1`は`profile_id`、`owner_kind`、`path_template`、required segment、許可`asset_kind_id`集合、shipping classを持つ。`asset_kind`、`category`、`surface`、`feature`はこのCatalogから選ぶ。AIはCatalogにない分類語を生成しない。

特定Character、Environment、Featureと同じlifecycleで変更、削除、配布されるAudio／Effectはsubject配下へ置く。複数Domainを横断してSoundtrack、Master ambience、共通Fontとして管理されるものだけcross-cutting rootへ置く。Ownerを一つに決められない場合は`shared/`へ逃がさず、`MissingAssetOwner`でPromotionを停止する。

初期C1のSource extension allowlistはAsset正本と同じ次の集合とする。

| Asset type | C1 Source extension |
|---|---|
| Texture／Sprite | `.png`、`.jpg`、`.jpeg`、`.exr`、`.ktx2`、`.dds` |
| Mesh／Scene／Skeleton／Animation | `.gltf`、`.glb` |
| Audio | `.wav`、`.flac` |
| Font | `.otf`、`.ttf` |

`.blend`／`.fbx`はC2、`.usd`／`.usdz`はC3のCapability activation後だけ追加できる。`.psd`、`.svg`、任意video、未知binaryはC1へ黙って追加しない。File extension、declared media type、magic／container解析結果の三者が一致しなければImportを拒否する。

Textureの初期`semantic_role_id`は`base_color`、`emissive`、`normal`、`orm`、`mask`、Audioは`sfx`、`ui`、`dialogue`、`music`、`ambience`のclosed setをImport正本から使用する。名前だけでroleを確定せず、metadata、Material slot、channel mapping、User intentと一致させる。

### 6.2 Asset file name

Source Assetの名前は構造化された`AssetTechnicalNameV1`から生成する。

```text
AssetTechnicalNameV1
  subject_slug: required
  purpose_slug: required
  variant_slug: optional
  extension: required, Asset Type allowlist
```

表示形は次とする。

```text
<subject>_<purpose>[_<variant>].<extension>
```

例:

```text
hero_body.glb
hero_body_base_color.png
hero_body_normal.png
hero_run_forward.glb
hero_footstep_stone_01.flac
forgotten_forest_ambience_night.flac
inventory_slot_selected.png
main_theme_exploration.flac
```

`purpose_slug`はsubject内で何を表すかを示す具体的な名詞句または動作句とし、1～4語を推奨する。例は`body`、`body_base_color`、`run_forward`、`footstep_stone`、`slot`である。品質状態、生成元、担当者、revisionをpurposeにしない。

Underscore列を逆parseして意味を推測しない。`AssetTechnicalNameV1`とAsset metadataが各slot、`asset_type`、`semantic_role_id`を保持し、Path rendererがFile名を決定する。これにより`forgotten_forest`のような複合subjectでも曖昧にならない。

Unreal Engine型の`T_`、`SM_`、`BP_`等のtype prefixは採用しない。Miraikanaiはextension、`asset_type`、media type、typed metadataで型を保持するため、名前へ型を重複符号化しない。

### 6.3 2D／3D配置例

```text
assets/source/
├─ characters/
│  ├─ hero/
│  │  ├─ models/
│  │  │  └─ hero_body.glb
│  │  ├─ textures/
│  │  │  ├─ hero_body_base_color.png
│  │  │  └─ hero_body_normal.png
│  │  ├─ animations/
│  │  │  ├─ hero_idle.glb
│  │  │  └─ hero_run_forward.glb
│  │  └─ audio/
│  │     └─ hero_footstep_stone_01.flac
│  └─ forest_slime/
│     ├─ sprites/
│     │  ├─ forest_slime_idle.png
│     │  └─ forest_slime_attack.png
│     └─ audio/
│        └─ forest_slime_attack.flac
├─ environments/
│  └─ forgotten_forest/
│     ├─ geometry/
│     │  └─ forgotten_forest_tree_cluster.glb
│     ├─ textures/
│     │  └─ forgotten_forest_ground_base_color.png
│     ├─ foliage/
│     ├─ effects/
│     └─ audio/
│        └─ forgotten_forest_ambience_night.flac
├─ gameplay/
│  └─ combat/
│     └─ fireball/
│        ├─ icons/
│        │  └─ fireball_icon.png
│        └─ effects/
│           └─ fireball_impact.png
├─ ui/
│  └─ inventory/
│     ├─ icons/
│     └─ backgrounds/
├─ audio/
│  └─ music/
│     └─ main_theme/
│        └─ main_theme_exploration.flac
└─ localization/
   ├─ ja-JP/
   │  └─ main_story/
   └─ en-US/
      └─ main_story/
```

File extension例はCapabilityがActivatedされている場合だけ有効である。未知extensionを名前規則だけで許可しない。

### 6.4 Structured localizationとmedia

翻訳Key、Message、plural、argument schemaは`authoring/localization/<document_id>/localization.mirakan.json`へ保存する。`assets/source/localization/<locale>/`は音声、locale固有画像等のmediaだけに使う。文字列Tableを任意CSVとして二重管理しない。外部翻訳形式はImport／Export projectionであり、Authoring Documentを正本とする。

### 6.5 External package

Third-party package内部の名前を無差別renameしない。原本保持が必要なPackageは`external_packages/<package_slug>/original/`へ隔離し、path-scoped `NamingExceptionRecordV1`、License、Provenance、hashを必須とする。`NamingExceptionRecordV1`はexact path prefix、Package `AssetId`、適用rule、理由、owner、作成revision、失効条件を持ち、AIが作成または範囲拡張しない。

RuntimeまたはGame Authoringはoriginal pathを直接参照しない。Importer／Adapterが正規`AssetId`、typed role、canonical first-party placementへ変換する。安全に参照を書換えられないmulti-file bundleは、原名のまま通常Asset treeへ混入させずImportを拒否またはcontainer形式へ変換する。

## 7. Authoring Document配置

Authoring Documentの物理Pathは表示名またはAI提案名から作らない。GatewayがUUIDv7 Stable IDから決定論的に作る。

```text
authoring/scenes/<scene_id>/scene.mirakan.json
authoring/scenes/<scene_id>/shards/<shard_id>.mirakan.json
authoring/levels/<level_id>/level.mirakan.json
authoring/worlds/<world_id>/world.mirakan.json
authoring/gameplay/<definition_id>/gameplay_definition.mirakan.json
authoring/ui/<ui_document_id>/ui.mirakan.json
authoring/localization/<localization_id>/localization.mirakan.json
authoring/decisions/<decision_id>/decision.mirakan.json
```

AIは`<scene_id>`等を生成または推測せず、`Create*` OperationでGatewayへ発行を要求する。GatewayはIDと結果Pathを返す。Display name変更でPathを変えず、technical pathをUI表示順、World階層、Scene Tree階層として利用しない。

Authoring JSONはUTF-8 without BOM、LF、duplicate key禁止、comment禁止、trailing comma禁止、canonical Schema version必須とする。AIはJSON全文を直接書かず、Stable ID対象のtyped Operationだけを提案する。

## 8. Native Game Code

Game固有C++は`native/game/`だけに置き、Engine sourceと混在させない。

```text
native/game/
├─ CMakeLists.txt
├─ include/
│  └─ <project_slug>/
├─ modules/
├─ source/
│  └─ shaders/
└─ tests/
```

識別子caseはEngine命名正本を継承する。Game固有の正規対応を次へ固定する。

```text
C++ namespace:  mirakan::game::<project_slug>::<domain>
Named Module:   mirakan.game.<project_slug>.<domain>
CMake target:   mirakan_game_<project_slug>_<domain>
CMake alias:    mirakan::game_<project_slug>_<domain>
Header root:    native/game/include/<project_slug>/<domain>/
```

例:

```text
mirakan::game::forgotten_echoes::combat
mirakan.game.forgotten_echoes.combat
mirakan_game_forgotten_echoes_combat
native/game/source/combat/damage_resolution.cpp
```

Game Authoring DocumentからC++ function名、file path、pointer、vtableを参照しない。DefinitionとNative implementationはCapability ID、Command、Event、Snapshot、Save fieldのtyped contractで接続する。

承認済みProject HLSLは対応CapabilityのR3 Gate後だけ`native/game/source/shaders/<domain>/<subject>.hlsl`へ置く。通常制作ではMaterial／VFX等の型付きAuthoringを優先し、任意HLSLをAsset扱いで`assets/source`へ置かない。

AI生成または変更C++／HLSLは`native/game`へ直接writeしない。`SourceDeltaV1`、隔離Staging Worktree、Format、Compile、Test、Review、Source Promotion、昇格後clean Buildを通す。

## 9. AI Staging

### 9.1 正規構造

AI TaskのStagingはTrusted Brokerだけが作成する。

```text
.mirakan/staging/
├─ imports/
│  └─ <job_id>/
└─ ai_tasks/
   └─ <task_id>/
      ├─ task_specification.mirakan.json
      ├─ authorization_envelope.mirakan.json
      ├─ context_manifest.mirakan.json
      ├─ proposal_manifest.mirakan.json
      ├─ authoring_proposals/
      ├─ asset_candidates/
      ├─ source_deltas/
      ├─ previews/
      ├─ diagnostics/
      └─ receipts/
```

`<job_id>`と`<task_id>`はGateway発行のUUIDv7であり、AI会話のtitle、prompt、日時、Provider名をDirectory名にしない。Staging file名はclosed setであり、AIが`result_final_2/`等を追加しない。

Stagingは非信頼領域である。存在するだけではProject Asset、Authoring Document、Source、Build input、Runtime inputにならない。EditorはStaging候補と正式Assetを明確に別stateで表示する。

### 9.2 AIが提出する意味

AIはraw destination pathを権限として提出せず、`ProposedProjectPlacementV1`を提出する。

```text
ProposedProjectPlacementV1
  output_kind
  owner_kind
  owner_id
  subject_id
  subject_slug
  asset_kind_id
  purpose_slug
  semantic_role_id
  variant_slug
  source_media_type
  requested_extension
  intended_consumers
  replacement_intent
```

信頼済みPlacement ResolverがPolicyとCatalogからcanonical pathを再計算する。AI提出の表示Pathと計算結果が異なる場合、計算結果へ黙って補正せず`PlacementMismatch`を返してProposalを更新させる。

### 9.3 Promotion先

| Output | Staging | 正式化するAuthority | Canonical destination |
|---|---|---|---|
| Scene／Level／Gameplay／UI | `authoring_proposals/` | Authoring Gateway | `authoring/<kind>/<stable_id>/...` |
| Image／3D／Audio／Font | `asset_candidates/` | Asset Broker | `assets/source/<placement profile>/...` |
| Asset metadata | Broker内部 | Asset Gateway | `assets/metadata/<asset_id>/asset.mirakan.json` |
| C++／Shader／Test | `source_deltas/` | Source Promotion Service | 許可された`native/game/` path |
| Preview | `previews/` | 昇格しない | Staging限定 |
| Cooked／compressed／LOD／package | Build staging | Build／Cook system | `build/`またはcontent-addressed cache |

Derived Artifactを`assets/source`へSourceのふりをして入れない。AIが生成したLOD、thumbnail、compressed texture、navmesh、shader binaryはDerived Artifactであり、元Sourceと生成契約から再構築する。

## 10. Folder／File追加の正規Operation

### 10.1 Directory

Directory作成そのものをUser intentにしない。次のOperationの副作用として必要なDirectoryだけをGatewayが作る。

- `CreateProjectDocument`
- `CreateAssetSubject`
- `ProposeAssetImport`
- `CreateGameplayDefinition`
- `CreateNativeGameModule`
- `BeginAiAuthoringTask`

AIは空Directoryを作らない。未知のtop-level、owner kind、asset kindを必要とする場合は、Directory作成ではなくPolicy extensionを提案し、owner、schema、migration、Validator、testを要求する。

### 10.2 File

新規Fileは次を満たす場合だけ作成する。

1. 対応するtyped OperationまたはSourceDeltaがある。
2. Owner、subject、kind、role、destinationが解決済みである。
3. canonical path keyが一意である。
4. extensionとmedia typeがActivated Capabilityに一致する。
5. Authoring／metadataならGateway発行IDとSchemaがある。
6. 外部／AI AssetならProvenance、License、Safety recordがある。
7. Preview、Validation、Budget、dependency closureが合格する。
8. 必要なApprovalがProposal Diffと同じhashを承認している。

人間による追加経路も次へ固定する。

| 経路 | 動作 |
|---|---|
| Editor drag-and-drop／Import CLI | `.mirakan/staging/imports/<job_id>/`へ置き、PreviewとApproval後にBrokerが正式配置 |
| 外部Toolが`assets/source`へ直接保存 | File watcherが`PendingImport`として検出。metadata作成とValidation完了まではAsset Catalog、Play、Build、Packageから除外 |
| IDEが`native/game`を編集 | Source changeとして検出し、Format／Build／Test結果を表示。AI生成Sourceと同じPromotion evidence形式を使用できるが、人間の保存自体をAI Approvalへ偽装しない |
| 外部ToolがAuthoring JSONを編集 | 三者比較後にtyped Operationへ変換できる場合だけExternal Tool ChangeSet候補にする |

Importer、AI、File watcherが同じSourceを同時に登録しようとした場合、canonical path keyとSource hashで一つのPending Importへ束ねる。異なる内容ならCollisionとして停止し、最後に書いたProcessを自動勝者にしない。

### 10.3 Move／rename

- Display name変更はPathを変更しない。
- Source Assetのtechnical rename／moveは`AssetId`を維持し、`source_relative_path`とAsset revisionを更新する。
- 参照は`AssetId`で維持し、Path redirectを永続参照として増やさない。
- caseだけのrenameもcanonical key上の一回のMigrationとしてGatewayが処理する。
- `project_slug`変更はnamespace、Module、CMake、include pathを含むProject-wide Migrationであり、通常renameと分ける。

### 10.4 Delete

Folderを再帰的に直接削除しない。削除対象Asset／Document／Source entryをStable IDまたはcanonical pathで列挙し、reference closure、package、Save compatibility、Native bindingを検証する。Commit後に空になったDirectoryだけをGatewayが除去する。

AIによるdelete、置換、shared consumerを持つmove、Project slug変更は人間Reviewを必須とする。

## 11. Collision、deduplication、revision

同じcanonical pathが存在する場合は次の一つに解決する。

| 条件 | 結果 |
|---|---|
| Source hashと意味metadataが同一 | 既存Assetを再利用 |
| 同じ`AssetId`の明示更新 | 新しい`AssetRevision`候補 |
| 異なる`AssetId`、異なる意味 | `AssetPathCollision`で拒否 |
| 既存Assetの明示置換 | consumer closure、Diff、Approval後にrevision更新 |

Collision回避の自動連番を禁止する。

```text
# 禁止
hero.png
hero_2.png
hero_3.png

# 意味が異なるなら許可
hero_portrait_neutral.png
hero_portrait_damaged.png
hero_portrait_winter.png
```

意味のあるvariantを決められない場合、AIは推測せず質問またはBlocking Diagnosticを返す。Content hashはdeduplication keyに利用できるが、Asset identityにはしない。

## 12. AI生成AssetのProvenanceと権利

AI生成Assetも通常Assetと同じ最終Pathへ配置する。生成元は`AssetProvenanceRecordV1`で追跡し、最低限次を保持する。

```text
origin_kind
creator_or_provider
tool_and_model
generation_request_hash
input_asset_refs
terms_snapshot_ref
license_expression
commercial_use_review
content_safety_receipt
c2pa_manifest_hash
modification_chain
task_id
output_sha256
```

Prompt本文にSecretまたは個人情報を含めて保存しない。private generation recordを別権限領域に保持し、Project metadataにはhashと必要な監査参照だけを保存する。

AI自身の「商用利用可能」「安全」「完成」という説明をApprovalとして扱わない。Provider terms、入力Asset権利、Content Safety、Style consistency、Target budget、Scene Previewを信頼済みValidatorとReviewerが確認する。

`generative_ai`というoriginは品質状態ではない。AI生成Assetを人間が修正してもmodification chainを維持し、File名へ`ai`、`human_fixed`を付けない。

## 13. ApprovalとFailure

Risk分類とApprovalの正本はAI実装・保守ガバナンス規約とする。本書では少なくとも次を自動Promotionしない。

- 既存Asset、Document、Codeの削除または置換
- 複数consumerを持つAssetの意味変更
- Collision／Skeleton／Material slot／Localization key／Save schemaの変更
- Native Game Code、Shader source、Build設定、Capability activation
- License、terms、入力権利、Safety結果が不明
- Production packageへ初めて含める外部／AI Asset
- Project root、placement profile、project slug、Naming Policyの変更

失敗時は正規Project revisionと現在のactive Artifactを変更しない。Staging候補を隔離し、stable Diagnostic、evidence、remediation、blocking phaseを返す。部分Promotionを禁止する。

## 14. AI向け機械可読Policy

人間向け本文だけをAIの命名根拠にしない。実装時に`ProjectNamingPolicyV1`をMCD正本として定義し、Editor、CLI、AI Tool、Template、Validator、CIへ同じPolicyからprojectionを生成する。

```text
ProjectNamingPolicyV1
  policy_version
  canonical_product_stem
  manifest_name
  metadata_directory_name
  project_slug_grammar
  technical_segment_grammar
  locale_tag_grammar
  stable_id_path_grammar
  maximum_segment_bytes
  maximum_file_stem_bytes
  maximum_relative_path_bytes
  reserved_root_names
  forbidden_standalone_names
  mcd_suffix
  asset_placement_profiles
  asset_kind_catalog
  semantic_role_catalog
  extension_media_type_allowlist
  source_control_classes
  collision_policy
  promotion_policy
  legacy_input_mappings
```

AI Taskへ渡す`ProjectNamingPolicyProjectionV1`はTask scopeに必要な次だけを含む。

- Policy versionとhash
- 許可されたoutput kindとPathGrant
- 対象owner／subjectのStable IDとtechnical slug
- 使用可能なplacement profile、asset kind、semantic role、extension
- 対象scope内の既存canonical path key
- replacement／delete／rename許可
- 必須ValidationとApproval
- 正例、反例、stable Diagnostic ID

AIは全文Repositoryから規則を推測しない。Projectionにないroot、role、略語、extension、IDを発明した場合はProposalを拒否する。

## 15. Stable Diagnostic

初期Diagnosticを次へ固定する。Message文字列ではなくID、対象Field、evidence、remediationでAIとEditorを接続する。

| Diagnostic ID | 条件 | Remediation |
|---|---|---|
| `MIRAKAN-PROJECT-NAMING-0001` | `InvalidTechnicalSegment` | ASCII lowercase snake_caseへ変更 |
| `MIRAKAN-PROJECT-NAMING-0002` | `ForbiddenStandaloneName` | Domain責務を示す具体名へ変更 |
| `MIRAKAN-PROJECT-NAMING-0003` | `UnknownProjectRoot` | closed rootへ配置、またはPolicy extension |
| `MIRAKAN-PROJECT-NAMING-0004` | `MissingAssetOwner` | owner kindとStable IDを指定 |
| `MIRAKAN-PROJECT-NAMING-0005` | `UnknownAssetKind` | Catalogから選択 |
| `MIRAKAN-PROJECT-NAMING-0006` | `AmbiguousSemanticRole` | roleを質問またはProfileで指定 |
| `MIRAKAN-PROJECT-NAMING-0007` | `AssetPathCollision` | reuse、revision update、意味variantのいずれかを明示 |
| `MIRAKAN-PROJECT-NAMING-0008` | `CaseFoldCollision` | canonical keyが異なるtechnical slugへ変更 |
| `MIRAKAN-PROJECT-NAMING-0009` | `DirectCanonicalWriteDenied` | typed Proposal／SourceDeltaを使用 |
| `MIRAKAN-PROJECT-NAMING-0010` | `MissingAssetProvenance` | Provenance／License／Safety recordを追加 |
| `MIRAKAN-PROJECT-NAMING-0011` | `PlacementMismatch` | Resolver計算結果に基づきProposalを再提出 |
| `MIRAKAN-PROJECT-NAMING-0012` | `LegacyProjectStem` | `mirakan.project.json`／`.mirakan/`へMigration |
| `MIRAKAN-PROJECT-NAMING-0013` | `PathOutsideProject` | Project-relative許可Pathへ限定 |
| `MIRAKAN-PROJECT-NAMING-0014` | `PromotionGateFailed` | evidenceで示されたGateを修正し新Taskで再提出 |

Validatorは勝手にrename、連番追加、Directory移動、別Assetへの参照差替えを行わない。安全なcanonicalizationと意味変更を区別する。

## 16. 代表的なAI制作フロー

「忘れられた森のBossと戦闘Sceneを作る」というRequestを例にする。

1. AIはGameSpec、World、Visual Style、Target budget、既存Character／Asset Catalogを読む。
2. 不足するBoss ownerを`CreateCharacterSubject` Proposalとして提出する。
3. Gatewayが`subject_id`を発行し、technical slug `forest_guardian`を検証する。
4. AIはMesh、Texture、Animation、Audioごとの`AssetRequirement`と複数candidateを生成する。
5. Brokerは`.mirakan/staging/ai_tasks/<task_id>/asset_candidates/`へ候補を隔離する。
6. Placement Resolverは次のdestinationを計算する。

```text
assets/source/characters/forest_guardian/models/forest_guardian_body.glb
assets/source/characters/forest_guardian/textures/forest_guardian_body_base_color.png
assets/source/characters/forest_guardian/animations/forest_guardian_attack_heavy.glb
assets/source/characters/forest_guardian/audio/forest_guardian_roar.flac
```

7. Import、topology、rig、loop、style、rights、safety、Target cook、memory budgetを検証する。
8. AIはScene／Gameplay変更をStable ID対象のtyped Operationとして提案する。
9. EditorはAsset Preview、Scene Preview、Project Diff、dependency closure、費用とRiskを表示する。
10. Approval後、Asset Broker、Authoring Gatewayが同じapproved proposal hashを確認し、dependency closure全体をatomic promotionする。
11. 失敗時は旧Sceneと旧Asset generationを維持し、StagingだけをFailedにする。

AIがDirectory名やUUIDを先に作るのではなく、Game上の意味を提案し、Engineが正規配置を作る。

## 17. 明示的に採用しない方式

- AIがProject全体へのfilesystem write権限を持つ
- 人間用とAI用で最終Directoryを分ける
- `Assets/Textures`、`Assets/Models`のように全Projectをtypeだけで分割する
- `T_`、`SM_`等のtype prefixとtyped metadataを二重管理する
- Display name、Path、content hashをStable identityにする
- `shared/`、`common/`、`misc/`へowner不明Assetを置く
- Collision時に`_2`、`_copy`、日付、担当者名を自動追加する
- AI生成であることをFile名へ永久に埋め込む
- Prompt本文、Provider応答、PreviewをRuntimeまたはProject正本にする
- Staging候補をBuild、Play、Packageから直接参照する
- Derived ArtifactをSource Assetとして保存する
- Folderのrecursive deleteをProject変更Operationにする
- AIがStable ID、Source path、License、Safety合格を推測する

## 18. Migration

### Phase P0: 規範固定

- 本書をGame Project配置・命名の正本にする。
- `ProjectNamingPolicyV1`、Diagnostic ID、layout goldenを固定する。
- 新規Templateは`mirakan.project.json`と`.mirakan/`だけを出力する。

### Phase P1: Legacy inventory

- `mira.project.json`、`.mira/`、旧`.json` Authoring名、type prefix、曖昧Directoryをread-only scanする。
- Path、Asset ID、Document ID、consumer closure、Git statusを含むMigration Planを生成する。
- 自動rename前にcase-fold衝突、reserved name、external bundleを検査する。

### Phase P2: Atomic path migration

- manifest、metadata Directory、MCD suffixを同一Project migrationで切り替える。
- Asset move／renameは`AssetId`を維持し、metadata revisionを更新する。
- Native namespace／Module／target変更はSource PromotionとしてBuild／Testする。
- 旧Path redirectまたは二重writeを恒久化しない。

### Phase P3: AI／Editor enforcement

- Editor作成Dialog、Asset Import、AI Tool、CLI、Templateが同じPolicy projectionを使う。
- Direct canonical write、未知root、collision、legacy stemをstable Diagnosticで拒否する。
- `.mirakan/staging`からのPromotionだけを正式AI追加経路にする。

## 19. Qualification Gate

本書を「実装可能」と判定するには次をすべて自動検証する。

1. Empty 2D、Empty 3D、UI中心、Localizationあり、Native codeありのProject templateが同じroot規則を満たす。
2. Windows／case-sensitive filesystemで同じcanonical path key集合になる。
3. absolute path、UNC、drive、`..`、reserved device name、trailing dot、case-fold collision、Unicode normalization collisionを拒否する。
4. Authoring Documentのcreate／rename／moveでStable ID参照が変化しない。
5. Asset move／rename／reimportで`AssetId`を維持し、revisionとhashだけが契約どおり変化する。
6. 同一Asset dedup、同じIDのrevision、異なるIDのcollisionを三つの別結果へ分類する。
7. AIがroot、Stable ID、path、role、extensionを発明したProposalをすべて拒否する。
8. AI Taskが正規Projectへ直接writeできず、Staging candidateがPlay／Build／Packageから不可視である。
9. Rights、Provenance、Safety、Budget、consumer closureの一つでも不合格ならpartial promotionが0件である。
10. C++ SourceDeltaは隔離Build／Test／Review／Promotion後だけ`native/game`へ反映される。
11. `.mirakan/staging`、`index`、`recovery`、`user`、`build`が公式Git profileとShipping packageへ入らない。
12. 3回の独立AI生成Taskで、最終Proposalのcase違反、legacy stem、曖昧Directory、連番collision回避、架空roleが0件である。
13. Human Editor、AI、CLI、Importerの四経路が同じ不正入力へ同じDiagnostic IDを返す。
14. Migration fixtureが`mira.project.json`／`.mira/`を一度だけ変換し、再実行して差分0となる。
15. Document、Policy projection、Template、Validator goldenに未定義root／kind／role／extensionが0件である。

## 20. 外部公式資料から採用した原則

外部Engineの固有表記ではなく、一次資料で確認できる次の原則だけを採用した。

- Unreal Engine: Project source、Content、Plugin、生成Directoryを分離し、大規模Asset検索のため早期に一貫した命名を定める。[Directory Structure](https://dev.epicgames.com/documentation/unreal-engine/unreal-engine-directory-structure)、[Recommended Asset Naming Conventions](https://dev.epicgames.com/documentation/en-us/unreal-engine/recommended-asset-naming-conventions-in-unreal-engine-projects)
- Unity: Source Assetと再生成可能なLibraryを分離し、Pathとは別のunique IDを`.meta`へ保持して参照を維持する。[Asset metadata](https://docs.unity3d.com/Manual/AssetMetadata.html)、[Introduction to importing assets](https://docs.unity3d.com/Manual/ImportingAssets.html)
- Godot: Filesystemを明示的なProject構造として扱い、関連Scene／Assetをsubjectの近くへ配置し、file／folderへ`snake_case`を使う。[Project organization](https://docs.godotengine.org/en/stable/tutorials/best_practices/project_organization.html)、[Version control systems](https://docs.godotengine.org/en/stable/tutorials/best_practices/version_control_systems.html)

Miraikanai固有の結論は、subject-first配置、UUIDv7 Stable ID、typed metadata、`.mirakan/staging`、atomic promotion、Provenance、Policy projectionを一つの規則へ統合することである。これは他Engineの「公式共通規格」ではなく、上記原則を根拠に定めたMiraikanai Engine公式Game Project規約である。
