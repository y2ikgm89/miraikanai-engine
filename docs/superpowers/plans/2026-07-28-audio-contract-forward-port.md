# Audio Contract Forward-Port Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `8ba1615`と`3ddf044`から現行Product境界に適合するAudio command、gain、Clock Domain、authoring Schema境界をforward-portする。

**Architecture:** 最新`main`から`codex/audio-contract-forward-port`を作り、Audio Ownerを主変更先とする。Asset LifecycleとGameplay Programming ModelにはAudio固有authorityを複写せず、format adoptionとpresentation cueの接続規則だけを追加する。

**Tech Stack:** Git、Markdown、PowerShell、ripgrep

## Global Constraints

- `docs/superpowers/specs/2026-07-28-unique-commit-migration-design.md`を要求仕様とする。
- `8ba1615`と`3ddf044`をcherry-pick、merge、rebaseしない。
- Audioのcurrent Product tier、named Source format集合、Operation集合を変更しない。
- Gainの単位を裸の`gain`で追加せず、`gain_db`と型を明示する。
- Gameplay ownerはauthoritative Event、Audio ownerはCue／Voice／Mixer／playbackを所有する。

---

### Task 1: Forward-port Audio command and gain contracts

**Files:**
- Modify: `docs/architecture/07-platform/audio.md:66`
- Modify: `docs/architecture/07-platform/audio.md:141`
- Modify: `docs/architecture/07-platform/audio.md:183`
- Modify: `docs/architecture/07-platform/audio.md:294`
- Test: `docs/architecture/07-platform/audio.md`

**Interfaces:**
- Consumes: `AudioRuntimeProfileV1`、`SoundCueDefinitionV1`、Mixer Graph、UI Settings generation
- Produces: Clock Domain contribution、typed gain／ramp／settings、atomic `SetBusGain` semantics

- [ ] **Step 1: Capture the current missing fields**

Run:

```powershell
$file = 'docs/architecture/07-platform/audio.md'
if (Select-String -LiteralPath $file -SimpleMatch 'AudioClockDomainContributionV1') { throw 'Clock contribution already present' }
if (Select-String -LiteralPath $file -SimpleMatch 'AudioGainRampV1') { throw 'Gain ramp already present' }
if (Select-String -LiteralPath $file -SimpleMatch 'layer=runtime | user_settings') { throw 'Layered SetBusGain already present' }
```

Expected: exit 0.

- [ ] **Step 2: Add Clock Domain and pause ownership**

Under section 3, add `### 3.1 Clock Domainとglobal Pause`. Define `AudioClockDomainContributionV1` and `AudioPauseSnapshotV1` as candidate versioned data. State that Audio consumes Runtime Clock Domain selection, global pause does not infer device stop, and audio completion cannot drive authoritative Gameplay.

- [ ] **Step 3: Replace untyped command gain fields**

In section 6, use these exact rows:

```markdown
| `PlayCue` | Cue Asset version、emitter、bus、priority、`gain_db`、pitch、owner generation |
| `SetBusGain` | Bus ID、`layer=runtime \| user_settings`、`gain_db`、ramp、`user_settings`時はSettings generation |
```

Add the rule that runtime and user-settings layers have separate owners. Apply a complete settings batch atomically at an Audio control-block boundary; reject the full batch on Graph、Bus、range、or generation mismatch.

- [ ] **Step 4: Add typed gain and settings values**

In section 8, define these candidate types and bounds:

```text
AudioGainDbV1
  value_db: finite float32 [-96.0, +12.0]

AudioGainRampV1
  duration_ms: uint32 [0, 60_000]
  curve: linear_db | linear_amplitude | equal_power

AudioUserSettingsV1
  schema_version: 1
  mixer_graph_ref: exact Audio Graph ArtifactRefV1
  bus_volumes[1..64]: Bus StableId byte order, no duplicates
  dynamic_range_profile_ref
  mono_output: bool
  settings_content_hash: SHA-256
```

State that mute is a separate bool, four gain layers combine in dB, invalid totals reject without changing the previous generation, and Settings identity／Apply／Revert belongs to UI／Text.

- [ ] **Step 5: Complete the authoring Schema list**

In section 14, ensure the list contains:

```text
AudioRuntimeProfileV1／Ref
AudioClockDomainContributionV1
AudioPauseSnapshotV1
AudioClipAssetV1
AudioImportSettingsV1
SoundCueDefinitionV1
MixerGraphV1
MixerSnapshotV1／Ref
AudioGainDbV1
AudioGainRampV1
AudioUserSettingsV1
SpatialAudioProfileV1
AudioCommandV1
AudioDiagnosticV1
```

Keep the current MCD／Operation／Provider sets exact `[]`.

- [ ] **Step 6: Validate and commit the Audio Owner**

Run:

```powershell
$file = 'docs/architecture/07-platform/audio.md'
$required = @('### 3.1 Clock Domainとglobal Pause','`gain_db`','`layer=runtime \| user_settings`','AudioGainRampV1','AudioPauseSnapshotV1','Operation集合はすべて`[]`')
foreach ($term in $required) {
  if (-not (Select-String -LiteralPath $file -SimpleMatch $term)) { throw "Missing: $term" }
}
git diff --check
git add -- docs/architecture/07-platform/audio.md
git commit -m "docs: forward-port audio command contracts"
```

Expected: one-file commit.

### Task 2: Connect Asset format and Gameplay presentation boundaries

**Files:**
- Modify: `docs/architecture/03-authoring/asset-lifecycle.md:114`
- Modify: `docs/architecture/03-authoring/gameplay-programming-model.md:104`

**Interfaces:**
- Consumes: Audio Owner contracts from Task 1
- Produces: format-adoption fail-closed rule and presentation-cue ownership rule

- [ ] **Step 1: Add the Asset format boundary**

In Asset Lifecycle section 2.1, state that WAV／FLAC parser、validation、Cook rules are target C1 contracts only after Product adoption. `AudioImportSettingsV1`／`AudioImportIRV1` and Toolchain locks do not activate a named format; the current adopted named-format set remains `[]`.

- [ ] **Step 2: Add the Gameplay presentation-cue boundary**

In Gameplay Programming Model section 4, state that `presentation_cue_ref` is a planned field until its Schema／validator／Cooker／fixture／Product destination exist. After activation it resolves to exact `SoundCueDefinitionV1` and Mixer Bus Stable ID. Gameplay owns the sealed Event and presentation intent; Audio owns playback state, Voice capacity, Mixer gain, loop, Spatial, and device failure.

- [ ] **Step 3: Validate and commit the cross-owner links**

Run:

```powershell
$asset = 'docs/architecture/03-authoring/asset-lifecycle.md'
$gameplay = 'docs/architecture/03-authoring/gameplay-programming-model.md'
if (-not (Select-String -LiteralPath $asset -SimpleMatch 'current採用済み名前付きformat集合はexact `[]`')) { throw 'Missing Audio format availability boundary' }
if (-not (Select-String -LiteralPath $gameplay -SimpleMatch '`presentation_cue_ref`')) { throw 'Missing presentation cue boundary' }
if (-not (Select-String -LiteralPath $gameplay -SimpleMatch '`SoundCueDefinitionV1`')) { throw 'Missing Audio owner target' }
git diff --check
git add -- $asset $gameplay
git commit -m "docs: connect audio authoring boundaries"
```

Expected: two-file commit.

### Task 3: Review the isolated Audio ChangeSet

**Files:**
- Review: `docs/architecture/07-platform/audio.md`
- Review: `docs/architecture/03-authoring/asset-lifecycle.md`
- Review: `docs/architecture/03-authoring/gameplay-programming-model.md`

**Interfaces:**
- Consumes: Tasks 1-2 commits
- Produces: PR-ready Audio-only diff

- [ ] **Step 1: Verify exact scope and no Product mutation**

Run:

```powershell
$allowed = @(
  'docs/architecture/03-authoring/asset-lifecycle.md',
  'docs/architecture/03-authoring/gameplay-programming-model.md',
  'docs/architecture/07-platform/audio.md'
)
$changed = @(git diff --name-only main...HEAD)
$unexpected = @($changed | Where-Object { $_ -notin $allowed })
if ($unexpected.Count -or $changed.Count -ne 3) { throw "Unexpected files: $($changed -join ', ')" }
if (git diff --name-only main...HEAD -- docs/architecture/00-product/product-plan.md) { throw 'Product Plan changed unexpectedly' }
git diff --check main...HEAD
```

Expected: exit 0.

- [ ] **Step 2: Verify availability remains fail-closed**

Run:

```powershell
$audio = 'docs/architecture/07-platform/audio.md'
$required = @('Capability stateは`not_activated`','Operation集合はすべて`[]`')
foreach ($term in $required) {
  if (-not (Select-String -LiteralPath $audio -SimpleMatch $term)) { throw "Missing fail-closed statement: $term" }
}
```

Expected: exit 0.
