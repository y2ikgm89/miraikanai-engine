# Existing Documentation Naming and Directory Unification Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 既存のMiraikanai Engine設計文書を、正規technical stem `mirakan`、正規Game Project layout、AI可読Directory語彙へ全面統一し、legacy表記を明示的Migration入力以外から除去する。

**Architecture:** Engine命名正本とGame Project配置・命名正本をauthorityとし、technical tokenを型別に移行する。機械置換の後、Directory tree、MCD名、Package名、UI名、TAA名、README索引を意味レビューし、全Markdownを横断する残存scanとformat／link検証で完了を証明する。

**Tech Stack:** Markdown、PowerShell、ripgrep、Git

## Global Constraints

- 自然言語製品名は`Miraikanai Engine`、technical stemは`mirakan`／`Mirakan`／`MIRAKAN`だけを使用する。
- C++ root namespaceは`mirakan`、CMake aliasは`mirakan::`、real targetは`mirakan_`、Named Module／Tool aliasは`mirakan.`とする。
- Public Header rootは`include/mirakan/`、Schema rootは`schemas/mirakan/`、MCD suffixは`.mirakan.json`とする。
- Project manifestは`mirakan.project.json`、Engine metadata Directoryは`.mirakan/`とする。
- Git repository slugとdefault clone rootは`mirakan-engine`とする。現在のhosting／local root renameは文書cutover後の実装ChangeSetで行う。
- Content package extensionは`.mirakanpack`、自然言語名は`Mirakan Content Package`とする。
- Engine／Tool／Project componentのprivate implementation Directoryは`source/`とし、repository-owned `src/`を新規正本にしない。
- `Miraikanai`の自然言語名、Third-party URL／path、明示的なlegacy Migration入力は機械置換しない。
- 既存の未コミット変更を保持し、対象token以外を巻き戻さない。
- 文書はUTF-8 without BOM、LF、trailing whitespaceなしを維持する。

---

### Task 1: Baseline inventory and exact migration map

**Files:**
- Read: `docs/superpowers/specs/2026-07-20-ai-readable-engine-naming-convention-design.md`
- Read: `docs/superpowers/specs/2026-07-20-ai-readable-game-project-layout-naming-design.md`
- Read: `docs/superpowers/specs/2026-07-19-engine-foundation-architecture-design.md`
- Read: `docs/superpowers/specs/2026-07-19-authoring-model-project-state-design.md`
- Read: `docs/superpowers/specs/2026-07-19-executable-contract-schema-codegen-design.md`
- Read: `docs/superpowers/specs/2026-07-20-cpp23-modules-import-std-transition-design.md`

**Interfaces:**
- Consumes: Engine命名正本のProduct stem／File／Directory／MCD規則とGame Project正本のlegacy mapping。
- Produces: 後続Taskがそのまま適用するclosed migration map。

- [x] **Step 1: Capture all Engine-owned legacy token classes**

Run:

```powershell
rg -o -g '*.md' -P '\bMira[A-Z][A-Za-z0-9_]*|\bMIRA_[A-Z0-9_]+|(?<![A-Za-z])mira(?:[_\.][a-z0-9_\.]+|::[A-Za-z0-9_:]+|[a-z]+)' docs |
  Sort-Object -Unique
```

Expected: `Mira*` type／product identifiers、`MIRA_*` macro、`mira::`、`mira_`、`mira.`、`.mirapack`を列挙する。

- [x] **Step 2: Fix the migration map**

Apply exactly:

```text
mira::                  -> mirakan::
mira_                   -> mirakan_
mira.                   -> mirakan.
MIRA_                   -> MIRAKAN_
Mira<Type>              -> Mirakan<Type>
MiraUI                  -> MirakanUi
include/mira/           -> include/mirakan/
schemas/mira/           -> schemas/mirakan/
.mira.json              -> .mirakan.json
mira.project.json       -> mirakan.project.json
.mira/                  -> .mirakan/
.mirapack               -> .mirakanpack
Mira Content Package    -> Mirakan Content Package
Mira Contract Definition -> Miraikanai Contract Definition
Mira TAA／Mira TAAU     -> Mirakan TAA／Mirakan TAAU
mira_taa_u_v1           -> mirakan_taau_v1
repository src/         -> source/
```

Do not convert `Miraikanai`, Third-party `github.com/.../src/...`, or quoted legacy inputs in the two naming specifications.

### Task 2: Normalize technical stems across every affected specification

**Files:**
- Modify: `docs/superpowers/specs/2026-07-18-ai-native-game-engine-authoring-design.md`
- Modify: `docs/superpowers/specs/2026-07-19-2d-3d-capability-plan.md`
- Modify: `docs/superpowers/specs/2026-07-19-ai-engine-development-governance-design.md`
- Modify: `docs/superpowers/specs/2026-07-19-ai-verification-evaluation-provenance-design.md`
- Modify: `docs/superpowers/specs/2026-07-19-asset-pipeline-content-packaging-design.md`
- Modify: `docs/superpowers/specs/2026-07-19-audio-mixer-spatial-architecture-design.md`
- Modify: `docs/superpowers/specs/2026-07-19-authoring-model-project-state-design.md`
- Modify: `docs/superpowers/specs/2026-07-19-cpp-structured-game-data-design.md`
- Modify: `docs/superpowers/specs/2026-07-19-domain-pack-future-capability-roadmap.md`
- Modify: `docs/superpowers/specs/2026-07-19-editor-workspace-ux-design.md`
- Modify: `docs/superpowers/specs/2026-07-19-engine-foundation-architecture-design.md`
- Modify: `docs/superpowers/specs/2026-07-19-executable-contract-schema-codegen-design.md`
- Modify: `docs/superpowers/specs/2026-07-19-input-action-device-architecture-design.md`
- Modify: `docs/superpowers/specs/2026-07-19-native-game-module-architecture-design.md`
- Modify: `docs/superpowers/specs/2026-07-19-rendering-render-graph-architecture-design.md`
- Modify: `docs/superpowers/specs/2026-07-19-runtime-integration-lifetime-performance-design.md`
- Modify: `docs/superpowers/specs/2026-07-19-ui-text-localization-accessibility-design.md`
- Modify: `docs/superpowers/specs/2026-07-19-windows-platform-distribution-design.md`
- Modify: `docs/superpowers/specs/2026-07-20-ai-readable-debugging-observability-replay-architecture-design.md`
- Modify: `docs/superpowers/specs/2026-07-20-ai-readable-math-core-utilities-architecture-design.md`
- Modify: `docs/superpowers/specs/2026-07-20-ai-readable-memory-pointer-architecture-design.md`
- Modify: `docs/superpowers/specs/2026-07-20-camera-platform-ai-authoring-virtual-production-architecture-design.md`
- Modify: `docs/superpowers/specs/2026-07-20-cpp23-modules-import-std-transition-design.md`
- Modify: `docs/superpowers/specs/2026-07-20-editor-ui-framework-architecture-design.md`
- Modify: `docs/superpowers/specs/2026-07-20-game-system-ai-codegen-architecture-design.md`
- Modify: `docs/superpowers/specs/2026-07-20-navigation-platform-architecture-design.md`
- Modify: `docs/superpowers/specs/2026-07-20-particle-vfx-architecture-design.md`
- Modify: `docs/superpowers/specs/2026-07-20-physics-ai-semantic-capability-catalog-design.md`
- Modify: `docs/superpowers/specs/2026-07-20-physics-engine-architecture-design.md`
- Modify: `docs/superpowers/specs/2026-07-20-water-surface-platform-architecture-design.md`
- Modify: `docs/superpowers/specs/2026-07-20-weather-snow-surface-architecture-design.md`
- Modify: `docs/superpowers/specs/2026-07-20-world-level-map-ai-authoring-architecture-design.md`
- Modify: `docs/superpowers/specs/README.md`

**Interfaces:**
- Consumes: Task 1 closed migration map。
- Produces: 全Normative technical identifierが`mirakan` stemを使うMarkdown集合。

- [x] **Step 1: Apply boundary-aware mechanical replacements**

For all `docs/**/*.md` except the two naming specifications, replace the exact token classes from Task 1. Use UTF-8 without BOM and LF. Do not perform an unrestricted `Mira` substring replacement.

- [x] **Step 2: Normalize PascalCase acronym output**

Verify and correct:

```text
MirakanUI                  -> MirakanUi
MirakanA* where A is acronym -> Naming Policy PascalCase
Miraikanai                 -> unchanged
```

- [x] **Step 3: Verify no normative old stem remains outside naming migration sections**

Run:

```powershell
rg -n -g '*.md' -P '\bMira[A-Z]|\bMIRA_|mira::|mira_|mira\.|/mira/|mira/|\.mira/|\.mira\.json|\.mirapack' docs
```

Expected: only explicitly labeled legacy／Migration／negative examples in:

```text
docs/superpowers/specs/2026-07-20-ai-readable-engine-naming-convention-design.md
docs/superpowers/specs/2026-07-20-ai-readable-game-project-layout-naming-design.md
docs/superpowers/specs/README.md
docs/superpowers/plans/2026-07-20-existing-documentation-naming-directory-unification.md
```

### Task 3: Normalize repository and Game Project directory structures

**Files:**
- Modify: `docs/superpowers/specs/2026-07-19-engine-foundation-architecture-design.md`
- Modify: `docs/superpowers/specs/2026-07-20-cpp23-modules-import-std-transition-design.md`
- Modify: `docs/superpowers/specs/2026-07-19-native-game-module-architecture-design.md`
- Modify: `docs/superpowers/specs/2026-07-19-authoring-model-project-state-design.md`
- Modify: `docs/superpowers/specs/2026-07-19-editor-workspace-ux-design.md`
- Modify: `docs/superpowers/specs/2026-07-19-executable-contract-schema-codegen-design.md`
- Modify: `docs/superpowers/specs/2026-07-20-ai-readable-game-project-layout-naming-design.md`

**Interfaces:**
- Consumes: canonical `mirakan` stem and Game Project root。
- Produces: 一意なEngine repository tree、component tree、Game Project tree。

- [x] **Step 1: Rewrite Engine repository tree**

Required changes:

```text
schemas/mira/                         -> schemas/mirakan/
orchestrator/src/                     -> orchestrator/source/
tools/contract_compiler/src/          -> tools/contract_compiler/source/
<component>/src/                      -> <component>/source/
include/mira/<component>/             -> include/mirakan/<component>/
```

Fix tree indentation so every `engine/<domain>` sibling has the same depth and brace notation is explicitly documentation shorthand, not a literal Directory name.

- [x] **Step 2: Rewrite Authoring and Editor metadata paths**

Required changes:

```text
mira.project.json                     -> mirakan.project.json
.mira/journal                         -> .mirakan/journal
.mira/recovery                        -> .mirakan/recovery
.mira/user                            -> .mirakan/user
scene.json                            -> scene.mirakan.json
Authoring instance *.json             -> *.mirakan.json where specified by Game Project authority
```

Add the Game Project配置・命名規約 as the path authority in Authoring／Editor specifications.

- [x] **Step 3: Verify repository-owned `src` is gone**

Run:

```powershell
rg -n -g '*.md' '(^|[</`])src/' docs
```

Expected: only Third-party URL／quoted external source path exceptions; no Miraikanai-owned repository path。

### Task 4: Reconcile names with their owning specifications

**Files:**
- Modify: `docs/superpowers/specs/2026-07-19-executable-contract-schema-codegen-design.md`
- Modify: `docs/superpowers/specs/2026-07-19-asset-pipeline-content-packaging-design.md`
- Modify: `docs/superpowers/specs/2026-07-19-2d-3d-capability-plan.md`
- Modify: `docs/superpowers/specs/2026-07-19-mobile-platform-architecture-design.md`
- Modify: `docs/superpowers/specs/2026-07-19-rendering-render-graph-architecture-design.md`
- Modify: `docs/superpowers/specs/2026-07-19-windows-platform-distribution-design.md`
- Modify: `docs/superpowers/specs/README.md`
- Modify: `docs/superpowers/specs/2026-07-20-ai-readable-engine-naming-convention-design.md`
- Modify: `docs/superpowers/specs/2026-07-20-ai-readable-game-project-layout-naming-design.md`

**Interfaces:**
- Consumes: normalized technical stems and Directory trees。
- Produces: 名前のownership、索引、例、Migration記述が相互に矛盾しないReview set。

- [x] **Step 1: Normalize named formats and product features**

Required canonical forms:

```text
Miraikanai Contract Definition (MCD)
Mirakan Content Package V1
.mirakanpack
Mirakan TAA
Mirakan TAAU
mirakan_taa_v1
mirakan_taau_v1
MirakanUi
MIRAKAN-<DOMAIN>-<4桁以上の番号>
MIRAKAN-<DOMAIN>-<CONDITION>
```

- [x] **Step 2: Update the official review index**

Add `Miraikanai Engine AI可読Game Project配置・命名規約` immediately after Engine naming, register `AI可読Capability Portfolio／MVP製品化・将来Roadmap規約` after the 2D／3D plan, `Lighting／AI Authoring規約` after Material, `Post Process／AI Authoring規約` after Lighting, `Scale-Resilient Canonical Architecture規約` after LOD, and `AI可読Shooter Gameplay／Weapon／Projectile規約` after Domain Pack, fix the official Review set count at 46, and ensure numbering remains contiguous.

- [x] **Step 3: Close migration wording**

Engine naming and Game Project naming specifications may retain old tokens only inside tables or paragraphs explicitly marked `Legacy`／`legacy`／`Migration`／`移行入力`／negative example. README may state the legacy-input policy, and this implementation plan may preserve the exact rename map and audit expressions. All normative examples describe only the canonical post-migration state.

### Task 5: Full-document verification

**Files:**
- Verify: `docs/**/*.md`

**Interfaces:**
- Consumes: all migrated Markdown。
- Produces: current evidence that the complete documentation set is unified。

- [x] **Step 1: Run forbidden-token audit**

Run:

```powershell
rg -n -g '*.md' -P '\bMira[A-Z]|\bMIRA[-_]|mira::|mira_|mira\.|/mira/|mira/|\.mira/|\.mira\.json|\.mirapack' docs
```

Expected: only reviewed migration／negative lines in the two naming specifications、README policy、and this plan。

- [x] **Step 2: Run canonical-token presence audit**

Run:

```powershell
rg -n -g '*.md' 'mirakan::|mirakan_|mirakan\.|schemas/mirakan|include/mirakan|\.mirakan/|\.mirakan\.json|\.mirakanpack' docs
```

Expected: all technical surfaces use the canonical stem appropriate to their surface。

- [x] **Step 3: Verify links, encoding, line endings, fences, and placeholders**

For every `docs/**/*.md`:

- decode as strict UTF-8
- reject BOM and CR bytes
- reject trailing whitespace
- require balanced triple-backtick fences
- resolve every relative Markdown `.md` link
- reject `TBD`、`TODO`、`FIXME` introduced by this migration

- [x] **Step 4: Review the complete diff**

Run:

```powershell
git diff --check
git diff --stat
git diff -- docs/superpowers/specs
```

Expected: only naming、Directory、cross-reference、index consistency changes; no user-authored unrelated content removed。

### Task 6: Commit and post-commit audit

**Files:**
- Commit: `docs/superpowers/plans/2026-07-20-existing-documentation-naming-directory-unification.md`
- Commit: LF-only normalization of `docs/superpowers/plans/2026-07-18-codex-config-optimization.md`
- Commit: `.gitattributes`のMarkdown LF enforcement
- Commit: all specification files modified by Tasks 2–4

**Interfaces:**
- Consumes: verified working-tree documentation。
- Produces: one auditable documentation-unification commit。

- [x] **Step 1: Stage only documentation unification files**

Run:

```powershell
git add -- docs/superpowers/specs docs/superpowers/plans/2026-07-20-existing-documentation-naming-directory-unification.md
git diff --cached --check
git diff --cached --name-status
```

Expected: no `.codex`、`.superpowers`、worktree helper、binary、unrelated source file。

- [x] **Step 2: Commit**

Run:

```powershell
git commit -m "docs: unify naming and directory conventions"
```

Expected: commit succeeds。

- [x] **Step 3: Re-run post-commit forbidden-token and status audit**

Run the Task 5 scans against `HEAD` content and confirm the two naming specifications are clean in the working tree. Existing unrelated user changes may remain, but no required documentation-unification change remains uncommitted。
