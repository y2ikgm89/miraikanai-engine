# Product Qualification and Clean-Break Integration Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Close the approved data-oriented design with one reproducible Performance campaign, one fail-closed Compatibility procedure, and one atomic destination Product Definition projection without changing the current active source definition.

**Architecture:** Performance／Capacity owns the profile, campaign, mandatory metrics, sampling, and promotion rules. Compatibility／Evolution proves whether the unreleased internal cutover may use `source_preserving_recook` and stops on any unresolved consumer. Product Plan adds only a destination projection whose Requirement, Fixture, Gate, risk, Work Package bindings, and ECS owner migration apply atomically with `RuntimeEcsCanonicalizationChangeSetV1`.

**Tech Stack:** Markdown architecture contracts, Product Registry rows, MCD content-addressed refs, PowerShell 7, ripgrep, Git.

## Global Constraints

- Normative design: `docs/superpowers/specs/2026-07-26-data-oriented-runtime-optimization-design.md`.
- Orchestration entry:
  `docs/superpowers/plans/2026-07-26-data-oriented-runtime-optimization-master-plan.md`.
- Prerequisite before Task 0:
  `docs/superpowers/plans/2026-07-26-foundation-value-memory-contract-plan.md`.
- Prerequisite before Task 2:
  complete `docs/superpowers/plans/2026-07-26-runtime-ecs-data-oriented-contract-plan.md`.
- Execute only in the same isolated, explicitly approved, tracked-and-clean
  implementation baseline used by the prerequisite plans. The current
  workspace's pre-existing Architecture changes are not implicitly authorized
  for staging or commit.
- Do not edit or stage unrelated dirty-worktree changes.
- Before each commit, require `git diff --cached --name-only` to contain only that task's listed files.
- Use `git commit --only -- <paths>`; never commit all staged or working-tree changes.
- The current source Product Definition and operational snapshot remain semantically byte-equal. Add destination projection prose and rows; do not rewrite current Registry rows in place.
- The destination projection is not implemented, activated, qualified, or Shipping merely because its architecture text exists.
- Every performance campaign binds one exact Candidate, Target Profile, Contract Set, Toolchain lock, fixture, and input trace.
- Profile literals are closed. Missing runs, metrics, soaks, or identity bindings are `infrastructure_error`, never zero or pass.
- Initial qualification has no fabricated baseline. Later promotion compares two complete campaigns with byte-equal measurement inputs.
- Clean break preserves committed Source and user data. It removes old readers, writers, aliases, generated bindings, and derived Runtime layouts only when the complete Consumer Inventory permits rebuild／recook.
- Do not add production dependencies or change Toolchain pins.

## Execution Preflight

Run once after the Foundation plan and before Task 0, then rerun after the
Runtime plan and before Task 2:

```powershell
$targets = @(
  'docs/architecture/04-runtime/performance-capacity.md',
  'docs/architecture/02-foundation/compatibility-evolution.md',
  'docs/architecture/01-governance/ai-verification-provenance.md',
  'docs/architecture/00-product/product-plan.md'
)
foreach ($path in $targets) {
  git ls-files --error-unmatch -- $path *> $null
  if ($LASTEXITCODE -ne 0) { throw "Target is not in the approved baseline: $path" }
  if (git status --porcelain -- $path) { throw "Target has pre-existing changes: $path" }
}
if (git diff --cached --name-only) { throw 'Index is not empty.' }
```

If this fails, stop before editing and obtain the user's approval for the
baseline commit. Do not commit the currently untracked
Compatibility／Evolution document as though it were created by this plan.

## Cross-Plan Execution Sequence

Use this exact acyclic order:

1. complete the Foundation plan;
2. execute Task 0 of this plan to close the existing unresolved Product
   evidence-owner reference;
3. execute Task 1 of this plan to establish the Performance-owned profile,
   campaign, metric IDs, and Diagnostic;
4. complete the Runtime ECS plan, including its Debugging consumer;
5. return here for Tasks 2–4.

Do not execute Runtime Task 5 before this plan's Task 1, and do not apply the
Product destination projection before all Runtime tasks pass.

## File Map

| File | Responsibility in this plan |
|---|---|
| `docs/architecture/04-runtime/performance-capacity.md` | Qualification profile, campaign, metric IDs, sampling, predicates, promotion |
| `docs/architecture/02-foundation/compatibility-evolution.md` | Consumer inventory, source-preserving recook, cutover rejection |
| `docs/architecture/01-governance/ai-verification-provenance.md` | Cross-Target release-evidence aggregation for the existing Product classes |
| `docs/architecture/00-product/product-plan.md` | Destination Requirement／Fixture／Gate／risk／Work Package and owner projection |
| All files from the prerequisite plans | Final cross-document linkage and ownership audit |

---

### Task 0: Resolve the current Product release-evidence owner

**Files:**
- Modify: `docs/architecture/00-product/product-plan.md:1438-1447`
- Modify: `docs/architecture/01-governance/ai-verification-provenance.md:5-6`
- Modify: `docs/architecture/01-governance/ai-verification-provenance.md:802-851`

**Interfaces:**
- Consumes: the 52-entry Architecture Document Inventory, Product evidence
  class definitions, AI Verification／Provenance release-evidence authority,
  Windows／Android／Apple package-owner Receipts.
- Produces: zero unresolved `mirakan.arch.*` document refs and an independent
  cross-Target Release Evidence owner; no new Architecture document or
  document ID.

- [ ] **Step 1: Reproduce the one unresolved document reference**

```powershell
$files = @(rg --files docs/architecture -g '*.md')
$documentIds = @{}
foreach ($file in $files) {
  $match = Select-String -Path $file -Pattern '^- 文書ID:\s*(\S+)' |
    Select-Object -First 1
  if ($match) {
    $documentIds[$match.Matches[0].Groups[1].Value] = $file
  }
}
$refs = foreach ($file in $files) {
  $content = Get-Content -Raw $file
  foreach ($match in [regex]::Matches(
    $content,
    'mirakan\.(?:arch|decision)\.[a-z0-9-]+'
  )) {
    $match.Value
  }
}
$unresolved = @(
  $refs |
    Sort-Object -Unique |
    Where-Object { -not $documentIds.ContainsKey($_) }
)
if (
  $unresolved.Count -ne 1 -or
  $unresolved[0] -ne 'mirakan.arch.platform-application-package-release'
) {
  throw "Baseline drift: unresolved refs are $($unresolved -join ', ')"
}
Write-Error 'Expected failure: Product release evidence names a nonexistent Architecture owner.'
```

Expected: FAIL after proving the only unresolved ref is
`mirakan.arch.platform-application-package-release`.

- [ ] **Step 2: Replace the two nonexistent owner refs**

In `ProductDecisionGateRegistryV1.evidence_class_definitions[]`, change only
these owner cells:

```text
evidence.class.package-install-offline-rollback-qualification
  owner_document_id = mirakan.arch.ai-verification-provenance

evidence.class.product-release-artifact-plan-valid
  owner_document_id = mirakan.arch.ai-verification-provenance
```

Do not create a 53rd Architecture document. AI Verification／Provenance already
owns Release Evidence, signed Evidence envelopes, Receipt freshness, and
provenance; Windows, Android, and Apple remain the semantic owners of their
Target-specific package, install, offline, signing, upload, rollback, and
device results.

- [ ] **Step 3: Bind AI Verification to the Target package owners**

Add direct dependencies from AI Verification／Provenance to:

```text
../07-platform/windows.md
../07-platform/android.md
../07-platform/apple.md
```

After `### 7.7 StoreUploadReceiptV1`, add
`### 7.8 Product release evidence class aggregation` with these exact
predicates:

```text
evidence.class.package-install-offline-rollback-qualification
  requested_target:
    target.windows.desktop | target.android.mobile | target.apple.mobile
  required_input:
    exact fresh Target-owner package Receipt
    exact package artifact hash and signature state
    clean install and launch result
    offline-run result
    rollback rehearsal result
  equality:
    Candidate, Active Product Definition, Contract Set, Toolchain lock,
    Target Profile, package artifact
  issuer_exclusion:
    wp.product.production-release-binding, its Task, and its Candidate

evidence.class.product-release-artifact-plan-valid
  requested_targets:
    [target.windows.desktop, target.android.mobile, target.apple.mobile]
  required_input:
    content-addressed artifact plan
    Target-lab plan
    signing/upload identity separation
    Store-staging plan
    rollback plan
    exact Windows/Android/Apple owner Review Receipt set
  equality:
    Candidate, Active Product Definition, Contract Set, Toolchain lock,
    Target Profile set
  issuer_exclusion:
    wp.product.production-release-binding, its Task, and its Candidate
```

The first class uses `policy.evidence.target-device.v1`; the second uses
`policy.evidence.contract-ci.v1`. Missing Target-owner Receipt, wrong Target,
mixed Candidate, stale／revoked input, incomplete Target set, or self-issued
Evidence fails closed. The aggregation owns no Target package schema, signing,
upload, Store, device, or rollback semantics.

- [ ] **Step 4: Replace the orphan owner prose**

Replace `Application Package／Release Owner` with
`AI Verification／Provenance Release Evidence Owner`. Add:

```markdown
同OwnerはWindows、Android、Apple各Ownerのfresh Target-specific Receiptをexact set equalityで集約してEvidence classを発行するだけで、Target package schema、signing、upload、rollback、Store policyを所有または上書きしない。`wp.product.production-release-binding`、そのTask、またはCandidateはこのEvidenceを自己発行できない。
```

- [ ] **Step 5: Verify all document refs and owner definitions resolve**

```powershell
$files = @(rg --files docs/architecture -g '*.md')
$documentIds = @{}
foreach ($file in $files) {
  $match = Select-String -Path $file -Pattern '^- 文書ID:\s*(\S+)' |
    Select-Object -First 1
  if ($match) {
    $documentIds[$match.Matches[0].Groups[1].Value] = $file
  }
}
$unresolved = @()
foreach ($file in $files) {
  $content = Get-Content -Raw $file
  foreach ($match in [regex]::Matches(
    $content,
    'mirakan\.(?:arch|decision)\.[a-z0-9-]+'
  )) {
    if (-not $documentIds.ContainsKey($match.Value)) {
      $unresolved += "$file -> $($match.Value)"
    }
  }
}
if ($unresolved.Count) {
  throw "Unresolved Architecture document refs:`n$($unresolved -join "`n")"
}
$productLines = Get-Content 'docs/architecture/00-product/product-plan.md'
foreach ($evidenceClassId in @(
  'evidence.class.package-install-offline-rollback-qualification',
  'evidence.class.product-release-artifact-plan-valid'
)) {
  $needle = '| `' + $evidenceClassId +
    '` | `mirakan.arch.ai-verification-provenance` |'
  $matches = @($productLines | Where-Object { $_.Contains($needle) })
  if ($matches.Count -ne 1) {
    throw "Expected one exact owner row for $evidenceClassId, got $($matches.Count)"
  }
  $verificationMatches = @(
    rg -n --fixed-strings $evidenceClassId `
      'docs/architecture/01-governance/ai-verification-provenance.md'
  )
  if ($verificationMatches.Count -ne 1) {
    throw "Expected one canonical Verification predicate for $evidenceClassId, got $($verificationMatches.Count)"
  }
}
```

Expected: zero unresolved Architecture document refs and both evidence classes
bound to the existing Verification owner.

- [ ] **Step 6: Commit only the owner correction and definition**

```powershell
$paths = @(
  'docs/architecture/00-product/product-plan.md',
  'docs/architecture/01-governance/ai-verification-provenance.md'
)
foreach ($path in $paths) { git diff --check -- $path }
git add -- $paths
$staged = @(git diff --cached --name-only)
if (
  $staged.Count -ne 2 -or
  @($staged | Where-Object { $paths -notcontains $_ }).Count -ne 0
) {
  throw "Unexpected staged paths: $($staged -join ', ')"
}
git commit --only -m "docs: resolve product release evidence owner" -- $paths
```

Expected: two-file commit; no Product Registry ID, evidence-class ID, package
semantic owner, or document count changes.

---

### Task 1: Define the canonical data-oriented qualification profile

**Files:**
- Modify: `docs/architecture/04-runtime/performance-capacity.md:240-275`
- Modify: `docs/architecture/04-runtime/performance-capacity.md:448-480`
- Modify: `docs/architecture/04-runtime/performance-capacity.md:1209-1260`

**Interfaces:**
- Consumes: exact Product `ArtifactCandidateBindingV1`, Target Profile, Contract Set, Toolchain lock, ECS fixture, input trace.
- Produces: `RuntimeDataOrientedQualificationProfileV1`, `RuntimeDataOrientedQualificationCampaignV1`, closed metric IDs, sampling matrix, hard predicates, promotion, and one Diagnostic.

- [ ] **Step 1: Run the missing-profile assertion**

```powershell
$path = 'docs/architecture/04-runtime/performance-capacity.md'
rg -q '^RuntimeDataOrientedQualificationProfileV1$' $path
$rgExit = $LASTEXITCODE
switch ($rgExit) {
  0 { throw 'Precondition failed: data-oriented profile already defined.' }
  1 { break }
  default { throw "rg failed while checking the profile owner (exit $rgExit)." }
}
Write-Error 'Expected failure: Runtime data-oriented qualification has no canonical profile.'
```

Expected: FAIL.

- [ ] **Step 2: Add the exact profile and campaign schemas**

In the integrated qualification section, add:

```text
RuntimeDataOrientedQualificationProfileV1
  profile_version: 1
  target_profile_ref:
    exact {target_profile_id, target_profile_version,
           target_profile_content_hash}
  contract_set_ref: ContractSetRefV1
  toolchain_lock_sha256: SHA-256
  fixture_id: fixture.runtime.ecs-data-oriented-core
  sample_policy: runtime_ecs_warmup_5x120s_median_p95_10m_soak_v1
  chunk_payload_candidates_bytes: [8192, 16384, 32768]
  scenario_ids:
    sequential_motion
    position_only_projection
    lifetime_only_scan
    structural_burst
    archetype_fragmentation
    query_cache_invalidation
  metric_set: runtime_ecs_data_oriented_metrics_v1
  correctness_oracle:
    runtime_ecs_semantic_publication_failure_atomicity_v1
  profile_hash: SHA-256

RuntimeDataOrientedQualificationCampaignV1
  campaign_version: 1
  profile_ref:
    exact {profile_version, target_profile_ref, contract_set_ref,
           toolchain_lock_sha256, profile_hash}
  artifact_candidate_binding_ref: content-addressed ref
  artifact_candidate_binding_sha256: SHA-256
  input_trace_ref: content-addressed ref
  input_trace_sha256: SHA-256
  sample_artifact_ref: content-addressed ref
  sample_artifact_sha256: SHA-256
  correctness_artifact_ref: content-addressed ref
  correctness_artifact_sha256: SHA-256
  result: pass | fail | infrastructure_error
  campaign_hash: SHA-256
```

The profile does not embed a Candidate. The campaign's
`ArtifactCandidateBindingV1` Target member, Contract Set, and Toolchain must be
byte-equal to the profile and to both sample／correctness artifacts.

Compute self-excluding hashes with the MCD canonical encoding and exact domains:

```text
profile_hash =
  SHA-256(
    ASCII "MIRAKAN_RUNTIME_DATA_ORIENTED_QUALIFICATION_PROFILE_V1"
    || uint32_be(len(canonical profile bytes excluding profile_hash))
    || canonical profile bytes excluding profile_hash
  )

campaign_hash =
  SHA-256(
    ASCII "MIRAKAN_RUNTIME_DATA_ORIENTED_QUALIFICATION_CAMPAIGN_V1"
    || uint32_be(len(canonical campaign bytes excluding campaign_hash))
    || canonical campaign bytes excluding campaign_hash
  )
```

Finalize the Contract Set root before materializing either record. Do not
insert a profile or campaign instance into the Contract Set preimage that it
references. Product signed wrappers continue to use RFC 8785 JCS and must not
substitute JCS bytes for these MCD hashes.

- [ ] **Step 3: Register the closed mandatory metric set**

Define `runtime_ecs_data_oriented_metrics_v1` as these exact IDs and value
kinds:

| Metric ID | Value kind |
|---|---|
| `metric.runtime.ecs.callback-general-heap-allocation-count` | `uint64_count` |
| `metric.runtime.ecs.callback-upstream-fallback-count` | `uint64_count` |
| `metric.runtime.ecs.memory-reserved-bytes` | `uint64_bytes` |
| `metric.runtime.ecs.memory-committed-bytes` | `uint64_bytes` |
| `metric.runtime.ecs.memory-live-bytes` | `uint64_bytes` |
| `metric.runtime.ecs.memory-peak-bytes` | `uint64_bytes` |
| `metric.runtime.ecs.chunk-count` | `uint64_count` |
| `metric.runtime.ecs.chunk-row-capacity` | `uint64_count` |
| `metric.runtime.ecs.chunk-occupied-rows` | `uint64_count` |
| `metric.runtime.ecs.chunk-unused-payload-bytes` | `uint64_bytes` |
| `metric.runtime.ecs.archetype-count` | `uint64_count` |
| `metric.runtime.ecs.archetype-fragmentation` | `ratio_rational` |
| `metric.runtime.ecs.selected-row-count` | `uint64_count` |
| `metric.runtime.ecs.contiguous-work-unit-count` | `uint64_count` |
| `metric.runtime.ecs.chunk-transition-count` | `uint64_count` |
| `metric.runtime.ecs.exposed-column-bytes` | `uint64_bytes` |
| `metric.runtime.ecs.useful-selected-payload-bytes` | `uint64_bytes` |
| `metric.runtime.ecs.query-cache-hit-count` | `uint64_count` |
| `metric.runtime.ecs.query-cache-miss-count` | `uint64_count` |
| `metric.runtime.ecs.query-cache-rebuild-count` | `uint64_count` |
| `metric.runtime.ecs.query-cache-invalidation-count` | `uint64_count` |
| `metric.runtime.ecs.structural-moved-row-count` | `uint64_count` |
| `metric.runtime.ecs.structural-copy-bytes` | `uint64_bytes` |
| `metric.runtime.ecs.handle-resolve-p50-ns` | `uint64_duration_ns` |
| `metric.runtime.ecs.handle-resolve-p95-ns` | `uint64_duration_ns` |
| `metric.runtime.ecs.handle-resolve-p99-ns` | `uint64_duration_ns` |
| `metric.runtime.ecs.lease-validation-p50-ns` | `uint64_duration_ns` |
| `metric.runtime.ecs.lease-validation-p95-ns` | `uint64_duration_ns` |
| `metric.runtime.ecs.lease-validation-p99-ns` | `uint64_duration_ns` |
| `metric.runtime.ecs.scenario-cpu-p50-ns` | `uint64_duration_ns` |
| `metric.runtime.ecs.scenario-cpu-p95-ns` | `uint64_duration_ns` |
| `metric.runtime.ecs.scenario-cpu-p99-ns` | `uint64_duration_ns` |
| `metric.runtime.ecs.semantic-result-hash` | `sha256` |
| `metric.runtime.ecs.publication-hash` | `sha256` |
| `metric.runtime.ecs.failure-atomicity` | `bool` |

Define `ratio_rational` as checked `{numerator:uint64, denominator:uint64}`;
denominator zero is `infrastructure_error`. Hardware cache misses, branch
misses, and bandwidth remain supplemental and never stand in for a missing
mandatory ID.

- [ ] **Step 4: Add the exact scenario observations**

Use this closed table:

| Scenario | Required observation |
|---|---|
| `sequential_motion` | Position＋Velocity columns traverse contiguous row ranges |
| `position_only_projection` | Velocity、Lifetime, and cold metadata are not exposed |
| `lifetime_only_scan` | Lifetime traversal does not require Position／Velocity payload access |
| `structural_burst` | bounded create／destroy／add／remove cost and atomic failure |
| `archetype_fragmentation` | archetype count、chunk occupancy、unused bytes、chunk transitions |
| `query_cache_invalidation` | hit、miss、rebuild, and invalidation after structural commit |

State that Position, Velocity, Lifetime, and cold metadata are synthetic bounded
test schemas, not Gameplay Component authorities.

- [ ] **Step 5: Expand the exact sampling matrix**

Define
`runtime_ecs_warmup_5x120s_median_p95_10m_soak_v1`:

1. three payload candidates × six scenarios;
2. five fresh-process measured runs per cell;
3. discard ten deterministic scenario cycles, then measure exactly 120 seconds;
4. choose each run P95 by nearest rank `ceil(0.95 * N)`;
5. sort five run P95 values and use the third;
6. run one additional fresh-process 600-second composite soak per payload after ten discarded cycles;
7. repeat scenarios in declared order with fixed inputs.

Assert the totals:

```powershell
$payloads = 3
$scenarios = 6
$runsPerCell = 5
$runSeconds = 120
$soaks = 3
$soakSeconds = 600
$measuredRuns = $payloads * $scenarios * $runsPerCell
$measuredSeconds = ($measuredRuns * $runSeconds) + ($soaks * $soakSeconds)
if ($measuredRuns -ne 90 -or $soaks -ne 3 -or $measuredSeconds -ne 12600) {
  throw 'Sampling matrix arithmetic mismatch.'
}
```

Missing run／soak, process reuse, scenario reorder, substituted samples,
NaN／infinite values, overflow, identity mismatch, or environment drift makes
the campaign `infrastructure_error`.

- [ ] **Step 6: Add the closed correctness and hard-predicate set**

Define `runtime_ecs_semantic_publication_failure_atomicity_v1` and require:

```text
callback general-heap allocation count = 0
callback upstream fallback count = 0
callback work unit crossing a chunk boundary = 0
unselected or undeclared column access = 0
semantic, publication, and failure oracle mismatch = 0
stale handle, expired lease, or accepted direct structural mutation = 0
required metric missing = 0
```

All predicates must pass for all 90 measured runs and three soaks. The selected
initial layout is 16384 bytes and must pass complete characterization; a better
8192／32768 result does not create an automatic Runtime choice.

- [ ] **Step 7: Add initial qualification and later promotion**

State exactly:

```markdown
Initial qualification invents neither a zero baseline nor an improvement percentage. It passes only when the selected 16 KiB layout completes the full matrix and every hard predicate.

After an initial qualified layout exists, promotion compares baseline and candidate campaigns with byte-equal profile ref、Target、Contract Set、Toolchain、fixture、input trace、sample policy, and correctness oracle. Candidate refs are intentionally different.
```

Promote only if:

- integrated P95 improves by at least 5% and 0.20 ms while peak memory and
  allocation count regress by no more than 5%; or
- peak memory improves by at least 15% while latency, allocation count,
  correctness, fault, load, and presentation remain within existing gates.

Otherwise retain the current qualified layout. Never auto-switch at Runtime.

- [ ] **Step 8: Register the exact Performance Diagnostic**

```text
diagnostic.performance.ecs-required-metric-missing
MIRAKAN-PERFORMANCE-ECS-REQUIRED-METRIC-MISSING
arguments = campaign_hash, scenario_id, payload_bytes, metric_id
result = qualification failure
```

- [ ] **Step 9: Verify and commit only Performance／Capacity**

```powershell
$path = 'docs/architecture/04-runtime/performance-capacity.md'
foreach ($token in @(
  'RuntimeDataOrientedQualificationProfileV1',
  'RuntimeDataOrientedQualificationCampaignV1',
  'MIRAKAN_RUNTIME_DATA_ORIENTED_QUALIFICATION_PROFILE_V1',
  'MIRAKAN_RUNTIME_DATA_ORIENTED_QUALIFICATION_CAMPAIGN_V1',
  'runtime_ecs_data_oriented_metrics_v1',
  'metric.runtime.ecs.failure-atomicity',
  'MIRAKAN-PERFORMANCE-ECS-REQUIRED-METRIC-MISSING',
  '12,600'
)) {
  rg -q --fixed-strings $token $path
  $rgExit = $LASTEXITCODE
  switch ($rgExit) {
    0 { break }
    1 { throw "Missing token: $token" }
    default { throw "rg failed while checking token '$token' (exit $rgExit)." }
  }
}
$metricCount = (rg -o 'metric\.runtime\.ecs\.[a-z0-9-]+' $path | Sort-Object -Unique | Measure-Object).Count
if ($metricCount -ne 35) { throw "Expected 35 mandatory metric IDs, got $metricCount" }
git diff --check -- $path
git add -- $path
$staged = @(git diff --cached --name-only)
if ($staged.Count -ne 1 -or $staged[0] -ne $path) { throw "Unexpected staged paths: $($staged -join ', ')" }
git commit --only -m "docs: define ECS data-oriented qualification" -- $path
```

Expected: one-file commit, two sole-owned schemas, and 35 mandatory metric IDs.

---

### Task 2: Close the clean-break through Compatibility consumer evidence

**Files:**
- Modify: `docs/architecture/02-foundation/compatibility-evolution.md:123-230`
- Modify: `docs/architecture/02-foundation/compatibility-evolution.md:252-291`

**Interfaces:**
- Consumes: `RuntimeEcsCanonicalizationChangeSetV1`, complete `CompatibilityConsumerInventoryV1`, retained Source and user-data boundaries.
- Produces: one conditional `source_preserving_recook` path, two Diagnostics, and a stop condition for retained external consumers.

- [ ] **Step 1: Assert the data-oriented cutover list is incomplete**

```powershell
$path = 'docs/architecture/02-foundation/compatibility-evolution.md'
rg -q 'old generated API signatures that violate' $path
$rgExit = $LASTEXITCODE
switch ($rgExit) {
  0 { throw 'Precondition failed: data-oriented cutover already recorded.' }
  1 { break }
  default { throw "rg failed while checking the cutover inventory (exit $rgExit)." }
}
Write-Error 'Expected failure: Compatibility lacks the complete data-oriented cutover.'
```

Expected: FAIL.

- [ ] **Step 2: Bind the cutover to the complete Consumer Inventory**

Under Runtime ECS canonicalization, require discovery of:

```text
repository worktree
reachable Git history
release registry
distribution registry
Native ABI registry
external API registry
```

Every scope must be `complete` or have a scope-specific passed
`not_applicable` fulfillment; the consumer-record and evidence-binding sets
must satisfy existing exact-equality rules. Empty records are zero only with
`unknown_consumer_state=zero_verified`.

- [ ] **Step 3: Define the conditional change policy**

When every retained consumer can be rebuilt from committed Source, use:

```text
change_class = source_preserving_recook
old_reader_policy = absent
old_writer_policy = absent
alias_policy = forbidden
source_preservation_policy = retain
regeneration_policy = full_recook | full_rebuild
rollback_policy = source_rebuild
```

Committed Source, Asset import documents, and approved Runtime entry documents
are the only regeneration inputs. Stale caches, old package bytes, raw Runtime
handles, chunk rows, addresses, and old generated bindings are prohibited
inputs.

- [ ] **Step 4: Add the exact removal inventory**

Remove atomically from current boundaries:

```text
legacy or suffixless type aliases
object-address or pool-slot identity
pointer-backed inline Component payload
per-Entity virtual update storage
Runtime shared_ptr ownership
alternate Shipping AoS/sparse-set storage
direct mutation during iteration
old query cache entries and persisted row selections
dual Package/Save/Replay Runtime-layout projections
old generated API signatures that violate CppValueTransferPolicyV1
```

Preserve committed Project Source, Asset provenance, published Saves, Native ABI
consumers, distributed Packages, and external API consumers unless their
approved migration says otherwise.

- [ ] **Step 5: Register two exact Compatibility Diagnostics**

```text
diagnostic.compatibility.ecs-consumer-inventory-unresolved
MIRAKAN-COMPATIBILITY-ECS-CONSUMER-INVENTORY-UNRESOLVED
arguments = change_set_ref, discovery_scope, discovery_state
result = clean-break application rejected

diagnostic.compatibility.ecs-retained-external-consumer
MIRAKAN-COMPATIBILITY-ECS-RETAINED-EXTERNAL-CONSUMER
arguments = change_set_ref, consumer_class, consumer_ref
result = switch to separately approved finite migration
```

- [ ] **Step 6: Add the retained-consumer stop condition**

If any release, published Save, Native ABI, distribution, or external consumer
cannot be rebuilt, stop the clean break. Require a separately approved
`versioned_reader_migration` or `external_api_deprecation` with finite reader
window, target release, telemetry, rollback, and removal Gate. Do not infer a
reader or leave it permanently active.

- [ ] **Step 7: Verify and commit only Compatibility／Evolution**

```powershell
$path = 'docs/architecture/02-foundation/compatibility-evolution.md'
foreach ($token in @(
  'source_preserving_recook',
  'old_reader_policy = absent',
  'CppValueTransferPolicyV1',
  'MIRAKAN-COMPATIBILITY-ECS-CONSUMER-INVENTORY-UNRESOLVED',
  'MIRAKAN-COMPATIBILITY-ECS-RETAINED-EXTERNAL-CONSUMER'
)) {
  rg -q --fixed-strings $token $path
  $rgExit = $LASTEXITCODE
  switch ($rgExit) {
    0 { break }
    1 { throw "Missing token: $token" }
    default { throw "rg failed while checking token '$token' (exit $rgExit)." }
  }
}
git diff --check -- $path
git add -- $path
$staged = @(git diff --cached --name-only)
if ($staged.Count -ne 1 -or $staged[0] -ne $path) { throw "Unexpected staged paths: $($staged -join ', ')" }
git commit --only -m "docs: close ECS clean-break inventory" -- $path
```

Expected: one-file commit; unresolved consumers still reject application.

---

### Task 3: Add the atomic destination Product Definition projection

**Files:**
- Modify: `docs/architecture/00-product/product-plan.md:1236-1359`
- Modify: `docs/architecture/00-product/product-plan.md:1360-1555`
- Modify: `docs/architecture/00-product/product-plan.md:1534-1565`
- Modify: `docs/architecture/00-product/product-plan.md:1670-1686`

**Interfaces:**
- Consumes: review-state `RuntimeEcsCanonicalizationChangeSetV1`, an unmaterialized destination migration candidate, owner-reference migration candidate, Performance profile／campaign, and current source Registry rows.
- Produces: exact destination Requirement, Fixture, Phase Gate, Phase additions, Work Package fixture bindings, three owner replacements, one new risk, one existing-risk mitigation replacement, and one Product Diagnostic.

- [ ] **Step 1: Capture the current source-definition checksum**

Before editing, isolate the current source Requirement prose, its
`RequirementRegistryV1` row, and the
`risk.product.memory-pointer-contract-drift` row. Record their exact text in the
execution notes together with the document hash:

```powershell
$path = 'docs/architecture/00-product/product-plan.md'
$before = (Get-FileHash -Algorithm SHA256 $path).Hash
"Pre-edit document SHA-256: $before"
rg -n '^### 11\.[3-7] ' $path
rg -n --fixed-strings '`requirement.foundation.memory-pointer-contract`は、' $path
rg -n --fixed-strings '| `requirement.foundation.memory-pointer-contract` |' $path
rg -n --fixed-strings '| `risk.product.memory-pointer-contract-drift` |' $path
```

The document hash will change because a destination projection is added. The
three captured current source entries must remain byte-equal; review them
separately in Step 8.

- [ ] **Step 2: Add a clearly non-active destination section**

After the existing ECS target-owner mapping, add:

```markdown
### Destination projection: data-oriented ECS core

This subsection is a closed projection of an unmaterialized, unapproved, and unapplied destination migration candidate associated with `RuntimeEcsCanonicalizationChangeSetV1` and its Owner-reference migration. The current Governance profile remains `state=review`, `contract_activation_effect=none`, with `definition_migration_binding_ref` absent. This subsection is not part of the current source Definition and does not change the operational snapshot. Only if a complete Product Definition Migration is separately materialized and approved do the destination projection, Change Set, and Owner-reference migration apply atomically; the source remains byte-equal until that atomic application.
```

- [ ] **Step 3: Add exact destination Registry rows**

Add this table verbatim:

| Registry | Exact addition or replacement |
|---|---|
| `RequirementRegistryV1` | `{requirement_id=requirement.runtime.ecs-data-oriented-core, owner_document_id=mirakan.arch.runtime-entity-component-system, verification_kind=runtime_ecs_data_oriented_qualification, failure_diagnostic_id=diagnostic.product.ecs-data-oriented-core-failed}` |
| `FixtureRegistryV1` | `{fixture_id=fixture.runtime.ecs-data-oriented-core, owner_document_id=mirakan.arch.runtime-entity-component-system, requirement_refs=[requirement.runtime.ecs-data-oriented-core], target_refs=[target.headless.host,target.windows.desktop,target.android.mobile,target.apple.mobile], minimum_duration_seconds=12600}` |
| `PhaseFixtureBindingRegistryV1` | `{gate_id=gate.product.phase-0-ecs-data-oriented-core, phase_id=phase.foundation, fixture_id=fixture.runtime.ecs-data-oriented-core, evaluated_requirement_refs=[requirement.runtime.ecs-data-oriented-core], target_refs=[target.headless.host], candidate_binding_policy_ref=policy.product.same-candidate.v1, freshness_policy_ref=policy.evidence.contract-ci.v1}` |
| `ProductPhaseRegistryV1` `phase.foundation` | append the new Requirement to `outcome_requirement_refs[]` and the new Gate to `exit_gate_refs[]`; `work_package_refs[]` is unchanged |
| `WorkPackageRegistryV1` | append `{kind=product_fixture, fixture_id=fixture.runtime.ecs-data-oriented-core}` to `provided_fixture_refs[]` of `wp.foundation.memory-pointers`, `wp.runtime.scheduling-core`, `wp.runtime.ecs-e0`, `wp.runtime.ecs-e1-storage`, and `wp.runtime.ecs-e2-query-mutation`; all other Fields remain unchanged except the coordinated Owner replacements |

The Headless Phase 0 Gate cannot satisfy Windows, Android, or Apple
qualification. Each later Target must rerun the full 12,600-second fixture with
`policy.evidence.target-device.v1`.

In the same destination projection, replace the acceptance semantics of the
existing `requirement.foundation.memory-pointer-contract` so its definition
closure is exactly:

```text
PointerContractV1
MemoryContractV1
PointerMemoryConsumerBindingV1
CppValueTransferPolicyV1
```

Bind the Requirement to the exact `MemoryContractV1` Type member ref and schema
hash in the same four-Type Contract Set, plus the fresh
`fixture.foundation.memory-pointer-contract` Receipt that proves the
retained-Field、single-`capacity_source`、six-layout/access-Field invariant.
Product Plan must not copy the Memory Field list. Leave the current source
Requirement prose and Requirement Registry row byte-equal until the atomic
migration.

In the destination projection only, replace the `mitigation` Field of
`risk.product.memory-pointer-contract-drift` with this exact value:

```text
the exact four-Type definition closure [PointerContractV1, MemoryContractV1, PointerMemoryConsumerBindingV1, CppValueTransferPolicyV1], bidirectional consumer Matrix, static／negative fixture, supported sanitizer lane, and hot path fallback 0 are bound to the same Phase 0 Candidate Gate
```

Retain every other Field byte-equal to the current source risk row. Do not edit
the current source row in §11.7; this explicit replacement exists only in the
destination projection and applies atomically with the other destination
changes.

- [ ] **Step 4: Add the exact destination owner migration**

Replace
`owner_document_id=mirakan.arch.runtime-scheduling-lifetime` with
`mirakan.arch.runtime-entity-component-system` only for:

```text
wp.runtime.ecs-e0
wp.runtime.ecs-e1-storage
wp.runtime.ecs-e2-query-mutation
```

Do not change `wp.foundation.memory-pointers` or
`wp.runtime.scheduling-core`, add a Work Package／Capability, or alter the
dependency chain.

- [ ] **Step 5: Add the exact ECS risk row**

| Field | Exact value |
|---|---|
| `risk_id` | `risk.product.ecs-data-oriented-regression` |
| `owner_document_id` | `mirakan.arch.runtime-entity-component-system` |
| `affected_work_package_refs[]` | `wp.foundation.memory-pointers; wp.runtime.scheduling-core; wp.runtime.ecs-e0; wp.runtime.ecs-e1-storage; wp.runtime.ecs-e2-query-mutation` |
| `trigger` | missing layout policy、dual Shipping layout、hot callback allocation／fallback、unbounded archetype growth、missing campaign cell／metric, or wrong-Target Receipt substitution |
| `likelihood` | `high` |
| `impact` | `critical` |
| `mitigation` | require the destination Phase 0 Gate、complete campaign、hard predicates, and fresh Target-specific reruns |
| `contingency` | reject the affected ECS Work Package transition and dependent Runtime activation; retain the last qualified layout without an alternate Shipping fallback |
| `monitor_gate_refs[]` | `gate.product.phase-0-ecs-data-oriented-core` |
| `genesis_state` | `open` |
| `revisit_gate_or_date` | `{kind=phase_gate, ref=gate.product.phase-0-ecs-data-oriented-core}` |

State that Product Plan is the only risk-row owner; other documents reference
it without copying trigger, severity, mitigation, or containment.

- [ ] **Step 6: Add Requirement acceptance and Work Package responsibilities**

The destination Requirement requires:

```text
accepted RuntimeComponentLayoutPolicyV1 records for fixture Components
one RuntimeArchetypeLayoutPlanV1 using ecs_chunk_soa_v1
query, lease, and structural contracts
all 35 mandatory metric IDs
every hard predicate passing
8192/16384/32768-byte characterization bound to the same Candidate
no Shipping AoS, sparse-set, object graph, or general-heap fallback
```

Add completion responsibilities:

| Work Package | Added completion responsibility |
|---|---|
| `wp.foundation.memory-pointers` | value-transfer policy、container layout Fields、static and negative Gates |
| `wp.runtime.ecs-e0` | type、owner、diagnostic, and Contract closure |
| `wp.runtime.ecs-e1-storage` | chunk SoA、layout policy、capacity、handle、fragmentation metrics |
| `wp.runtime.ecs-e2-query-mutation` | cached query、contiguous dispatch、allocation-free callback、deferred structural transaction |
| later `wp.runtime.ecs-e7-*` | rerun the qualified profile on the exact Target and device Evidence policy |

The Phase Fixture Gate evaluates its Requirement, fixture, Target, Candidate,
and freshness only. Phase exit separately requires current `complete` lifecycle
heads for every non-deferred listed Work Package, including E1 and E2.

- [ ] **Step 7: Register the Product-owned Target Diagnostic**

```text
diagnostic.product.ecs-target-receipt-mismatch
MIRAKAN-PRODUCT-ECS-TARGET-RECEIPT-MISMATCH
arguments = campaign_hash, expected_target_ref, actual_target_ref
result = Product Gate failure
```

Reference the aggregate
`diagnostic.product.ecs-data-oriented-core-failed` owned and registered by
Runtime ECS; do not duplicate its schema.

- [ ] **Step 8: Verify source rows were not rewritten in place**

```powershell
$path = 'docs/architecture/00-product/product-plan.md'
$diff = git diff --unified=0 -- $path
$removedRegistryRows = $diff | Select-String '^-.*(requirement_id=|fixture_id=|gate_id=|work_package_id=|risk_id=)'
if ($removedRegistryRows) {
  throw "Current Registry rows were removed or replaced:`n$($removedRegistryRows -join "`n")"
}
$sourceEntryMutations = $diff | Select-String '^-.*(`requirement\.foundation\.memory-pointer-contract`は、|\| `requirement\.foundation\.memory-pointer-contract` \||\| `risk\.product\.memory-pointer-contract-drift` \|)'
if ($sourceEntryMutations) {
  throw "Current source Requirement or risk entry changed:`n$($sourceEntryMutations -join "`n")"
}
foreach ($token in @(
  'requirement.runtime.ecs-data-oriented-core',
  'fixture.runtime.ecs-data-oriented-core',
  'gate.product.phase-0-ecs-data-oriented-core',
  'risk.product.ecs-data-oriented-regression',
  'MIRAKAN-PRODUCT-ECS-TARGET-RECEIPT-MISMATCH'
)) {
  rg -q --fixed-strings $token $path
  $rgExit = $LASTEXITCODE
  switch ($rgExit) {
    0 { break }
    1 { throw "Missing token: $token" }
    default { throw "rg failed while checking token '$token' (exit $rgExit)." }
  }
}
$content = Get-Content -Raw $path
$destination = [regex]::Match(
  $content,
  '(?ms)^### Destination projection: data-oriented ECS core\r?\n.*?(?=^### |\z)'
)
if (-not $destination.Success) { throw 'Destination projection section is missing.' }
$riskReplacement = 'the exact four-Type definition closure [PointerContractV1, MemoryContractV1, PointerMemoryConsumerBindingV1, CppValueTransferPolicyV1], bidirectional consumer Matrix, static／negative fixture, supported sanitizer lane, and hot path fallback 0 are bound to the same Phase 0 Candidate Gate'
if (-not $destination.Value.Contains($riskReplacement)) {
  throw 'Destination risk mitigation replacement is missing or not exact.'
}
$outsideDestination = $content.Remove($destination.Index, $destination.Length)
if ($outsideDestination.Contains($riskReplacement)) {
  throw 'Destination risk mitigation replacement escaped the destination section.'
}
```

Expected: the current source Requirement prose, Requirement Registry row, and
current source risk row are byte-equal; only the destination projection carries
the explicit risk replacement and other additions.

- [ ] **Step 9: Verify reference cardinalities and commit only Product Plan**

```powershell
$path = 'docs/architecture/00-product/product-plan.md'
$destination = Get-Content -Raw $path
foreach ($wp in @(
  'wp.foundation.memory-pointers',
  'wp.runtime.scheduling-core',
  'wp.runtime.ecs-e0',
  'wp.runtime.ecs-e1-storage',
  'wp.runtime.ecs-e2-query-mutation'
)) {
  if ($destination -notmatch [regex]::Escape($wp)) { throw "Missing Work Package: $wp" }
}
git diff --check -- $path
git add -- $path
$staged = @(git diff --cached --name-only)
if ($staged.Count -ne 1 -or $staged[0] -ne $path) { throw "Unexpected staged paths: $($staged -join ', ')" }
git commit --only -m "docs: project data-oriented ECS product gate" -- $path
```

Expected: one-file commit and no operational activation.

---

### Task 4: Run the final coordinated Architecture audit

**Files:**
- Verify: every canonical and direct-consumer file from all three plans
- Verify without normative duplication:
  - `docs/architecture/01-governance/architecture-governance.md`
  - `docs/architecture/02-foundation/toolchain-dependencies.md`
  - `docs/architecture/03-authoring/asset-lifecycle.md`
  - `docs/architecture/03-authoring/project-state.md`
  - `docs/architecture/04-runtime/persistence-save.md`
  - Simulation, Rendering, Platform, and Pack documents
  - `docs/architecture/README.md`

**Interfaces:**
- Consumes: all three implementation plans and their scoped commits.
- Produces: current evidence for ownership, linking, Registry closure, diagnostic closure, qualification arithmetic, clean-break safety, and no unrelated edits.

- [ ] **Step 1: Verify all Architecture IDs are unique**

```powershell
$files = @(rg --files docs/architecture -g '*.md')
$ids = foreach ($file in $files) {
  $match = Select-String -Path $file -Pattern '^- 文書ID:\s*(\S+)' | Select-Object -First 1
  if ($match) {
    [pscustomobject]@{ Id = $match.Matches[0].Groups[1].Value; File = $file }
  }
}
$duplicates = $ids | Group-Object Id | Where-Object Count -gt 1
if ($duplicates) { throw "Duplicate document IDs: $($duplicates.Name -join ', ')" }
if ($ids.Count -ne 52) { throw "Expected 52 Architecture document IDs, got $($ids.Count)" }
```

Expected: exactly 52 unique IDs.

- [ ] **Step 2: Verify all document-ID refs and relative Markdown links**

```powershell
$documentIdSet = @{}
foreach ($record in $ids) { $documentIdSet[$record.Id] = $record.File }
$unresolvedDocumentRefs = @()
foreach ($file in $files) {
  $content = Get-Content -Raw $file
  foreach ($match in [regex]::Matches(
    $content,
    'mirakan\.(?:arch|decision)\.[a-z0-9-]+'
  )) {
    if (-not $documentIdSet.ContainsKey($match.Value)) {
      $unresolvedDocumentRefs += "$file -> $($match.Value)"
    }
  }
}
if ($unresolvedDocumentRefs.Count) {
  throw "Unresolved document refs:`n$($unresolvedDocumentRefs -join "`n")"
}
```

```powershell
$broken = @()
foreach ($file in $files) {
  $content = Get-Content -Raw $file
  foreach ($m in [regex]::Matches($content, '\]\((?!https?://|#|mailto:)([^)#]+)(?:#[^)]+)?\)')) {
    $target = Join-Path (Split-Path $file) $m.Groups[1].Value
    if (-not (Test-Path $target)) { $broken += "$file -> $($m.Groups[1].Value)" }
  }
}
if ($broken.Count) { throw "Broken relative links:`n$($broken -join "`n")" }
```

Validate fragments against GitHub-style heading slugs:

```powershell
function Get-ArchitectureHeadingSlug(
  [string]$text,
  [hashtable]$seen
) {
  $slug = [regex]::Replace($text, '<[^>]+>', '').ToLowerInvariant()
  $slug = $slug.Replace('-', 'zzhyphensentinelzz')
  $slug = $slug.Replace('_', 'zzunderscoresentinelzz')
  $slug = [regex]::Replace($slug, '[\p{P}\p{S}]', '')
  $slug = $slug.Replace('zzhyphensentinelzz', '-')
  $slug = $slug.Replace('zzunderscoresentinelzz', '_')
  $slug = [regex]::Replace($slug, '\s+', '-').Trim('-')
  if ($seen.ContainsKey($slug)) {
    $seen[$slug]++
    return "$slug-$($seen[$slug])"
  }
  $seen[$slug] = 0
  return $slug
}

$anchorMaps = @{}
foreach ($file in $files) {
  $seen = @{}
  $anchors = @{}
  foreach ($line in Get-Content $file) {
    if ($line -match '^#{1,6}\s+(.+?)\s*#*\s*$') {
      $anchors[(Get-ArchitectureHeadingSlug $matches[1] $seen)] = $true
    }
  }
  $anchorMaps[(Resolve-Path $file).Path] = $anchors
}

$brokenAnchors = @()
foreach ($file in $files) {
  $content = Get-Content -Raw $file
  foreach ($match in [regex]::Matches($content, '\]\(([^)]+)\)')) {
    $raw = $match.Groups[1].Value.Trim()
    if ($raw -match '^(https?://|mailto:)') { continue }
    $destination = ($raw -split '\s+"')[0]
    if (-not $destination.Contains('#')) { continue }
    $parts = $destination.Split('#', 2)
    $pathPart = $parts[0]
    $fragment = [Uri]::UnescapeDataString($parts[1])
    if (-not $fragment) { continue }
    if ($pathPart) {
      $resolved = [IO.Path]::GetFullPath(
        (Join-Path (Split-Path $file) $pathPart)
      )
    } else {
      $resolved = (Resolve-Path $file).Path
    }
    if (
      -not $anchorMaps.ContainsKey($resolved) -or
      -not $anchorMaps[$resolved].ContainsKey($fragment)
    ) {
      $brokenAnchors += "$file -> $destination"
    }
  }
}
if ($brokenAnchors.Count) {
  throw "Broken heading anchors:`n$($brokenAnchors -join "`n")"
}
```

Expected: zero unresolved document IDs, relative paths, and heading anchors.

- [ ] **Step 3: Verify the three canonical schema owners**

```powershell
$architectureFiles = @(rg --files docs/architecture -g '*.md')
$expect = @{
  '^CppValueTransferPolicyV1$' = 'docs/architecture/02-foundation/memory-pointers.md'
  '^RuntimeComponentLayoutPolicyV1$' = 'docs/architecture/04-runtime/entity-component-system.md'
  '^RuntimeDataOrientedQualificationProfileV1$' = 'docs/architecture/04-runtime/performance-capacity.md'
}
foreach ($entry in $expect.GetEnumerator()) {
  $owners = @(rg -l $entry.Key $architectureFiles | ForEach-Object { $_ -replace '\\','/' })
  if ($owners.Count -ne 1 -or $owners[0] -ne $entry.Value) {
    throw "Owner mismatch for $($entry.Key): $($owners -join ', ')"
  }
}
$domainOwners = @{
  'MIRAKAN_CPP_VALUE_TRANSFER_POLICY_V1' =
    'docs/architecture/02-foundation/memory-pointers.md'
  'MIRAKAN_CPP_VALUE_TRANSFER_BINDING_V1' =
    'docs/architecture/02-foundation/memory-pointers.md'
  'MIRAKAN_RUNTIME_COMPONENT_LAYOUT_POLICY_V1' =
    'docs/architecture/04-runtime/entity-component-system.md'
  'MIRAKAN_RUNTIME_DATA_ORIENTED_QUALIFICATION_PROFILE_V1' =
    'docs/architecture/04-runtime/performance-capacity.md'
  'MIRAKAN_RUNTIME_DATA_ORIENTED_QUALIFICATION_CAMPAIGN_V1' =
    'docs/architecture/04-runtime/performance-capacity.md'
}
foreach ($entry in $domainOwners.GetEnumerator()) {
  $owners = @(
    rg -l --fixed-strings $entry.Key $architectureFiles |
      ForEach-Object { $_ -replace '\\','/' }
  )
  if ($owners.Count -ne 1 -or $owners[0] -ne $entry.Value) {
    throw "Hash-domain owner mismatch for $($entry.Key): $($owners -join ', ')"
  }
}
```

Expected: one full definition per schema at the intended owner.

- [ ] **Step 4: Verify the four-type Pointer／Memory closure**

```powershell
$foundation = @(
  'docs/architecture/02-foundation/executable-contracts.md',
  'docs/architecture/02-foundation/memory-pointers.md'
)
foreach ($type in @(
  'PointerContractV1',
  'MemoryContractV1',
  'PointerMemoryConsumerBindingV1',
  'CppValueTransferPolicyV1'
)) {
  foreach ($file in $foundation) {
    rg -q --fixed-strings $type $file
    $rgExit = $LASTEXITCODE
    switch ($rgExit) {
      0 { break }
      1 { throw "Missing $type in $file" }
      default { throw "rg failed while checking '$file' (exit $rgExit)." }
    }
  }
}

$productPath = 'docs/architecture/00-product/product-plan.md'
$projectionCommit = git log -1 --format=%H -- $productPath
if ($LASTEXITCODE -ne 0 -or -not $projectionCommit) {
  throw 'Cannot resolve the destination-projection commit.'
}
$currentText = (git show "${projectionCommit}:$productPath") -join "`n"
if ($LASTEXITCODE -ne 0) { throw 'Cannot read Product Plan at the projection commit.' }
$baselineText = (git show "${projectionCommit}^:$productPath") -join "`n"
if ($LASTEXITCODE -ne 0) { throw 'Cannot read the pre-projection Product Plan baseline.' }

$riskRowPattern = '(?m)^\| `risk\.product\.memory-pointer-contract-drift` \|.*$'
$baselineRiskRows = [regex]::Matches($baselineText, $riskRowPattern)
$currentRiskRows = [regex]::Matches($currentText, $riskRowPattern)
if ($baselineRiskRows.Count -ne 1 -or $currentRiskRows.Count -ne 1) {
  throw "Expected one current source risk row in baseline and projection; got $($baselineRiskRows.Count) and $($currentRiskRows.Count)."
}
if (-not [string]::Equals(
  $baselineRiskRows[0].Value,
  $currentRiskRows[0].Value,
  [StringComparison]::Ordinal
)) {
  throw 'Current source risk row is not byte-equal to the pre-projection baseline.'
}

$stalePattern = '三Contract|三種manifest|Pointer.Memory.*three|three.*Pointer.Memory'
$staleOptions = [Text.RegularExpressions.RegexOptions]::IgnoreCase
$baselineRiskStale = [regex]::Matches(
  $baselineRiskRows[0].Value,
  $stalePattern,
  $staleOptions
).Count
$baselineAllStale = [regex]::Matches(
  $baselineText,
  $stalePattern,
  $staleOptions
).Count
if ($baselineRiskStale -ne 1 -or $baselineAllStale -ne 1) {
  throw "Baseline must contain exactly one stale cardinality in the current source risk row; got risk=$baselineRiskStale total=$baselineAllStale."
}

$destination = [regex]::Match(
  $currentText,
  '(?ms)^### Destination projection: data-oriented ECS core\n.*?(?=^### |\z)'
)
if (-not $destination.Success) { throw 'Destination projection section is missing.' }
$destinationStale = [regex]::Matches(
  $destination.Value,
  $stalePattern,
  $staleOptions
).Count
$currentRiskStale = [regex]::Matches(
  $currentRiskRows[0].Value,
  $stalePattern,
  $staleOptions
).Count
$currentAllStale = [regex]::Matches(
  $currentText,
  $stalePattern,
  $staleOptions
).Count
$otherStale = $currentAllStale - $currentRiskStale - $destinationStale
if ($currentRiskStale -ne 1 -or $destinationStale -ne 0 -or $otherStale -ne 0) {
  throw "Stale cardinality scope mismatch: source-risk=$currentRiskStale destination=$destinationStale other=$otherStale."
}

$riskReplacement = 'the exact four-Type definition closure [PointerContractV1, MemoryContractV1, PointerMemoryConsumerBindingV1, CppValueTransferPolicyV1], bidirectional consumer Matrix, static／negative fixture, supported sanitizer lane, and hot path fallback 0 are bound to the same Phase 0 Candidate Gate'
if (-not $destination.Value.Contains($riskReplacement)) {
  throw 'Destination does not require the exact four-Type risk mitigation replacement.'
}
```

Expected: the exact four-Type closure is present; the sole stale cardinality is
the baseline-byte-equal current source risk row (one), while the destination and
all other sections contain zero. The destination requires the four-Type
replacement that becomes effective only with the atomic migration.

- [ ] **Step 5: Verify `MemoryContractV1` retained-plus-six shape**

Review the sole schema block and assert one `capacity_source` plus these six
added Fields:

```powershell
$path = 'docs/architecture/02-foundation/memory-pointers.md'
foreach ($field in @('storage_layout:','element_storage:','access_pattern:','growth_policy:','address_stability:','hot_path:')) {
  if ((rg -c --fixed-strings $field $path) -lt 1) { throw "Missing Memory Field: $field" }
}
$capacityCount = (rg -n '^  capacity_source:' $path | Measure-Object).Count
if ($capacityCount -ne 1) { throw "Expected one schema capacity_source, got $capacityCount" }
```

Then manually compare all retained Fields against the pre-change schema and the
approved design. No existing Field may disappear or split.

- [ ] **Step 6: Verify Product bidirectional closure**

Within the destination projection, verify:

- four new Stable IDs are each defined once;
- Requirement→Fixture→Gate→Phase and Gate→Requirement refs resolve;
- all five Work Packages provide the Product fixture;
- only E0／E1／E2 receive the ECS owner replacement;
- risk affected-WP and monitor-Gate refs resolve;
- no Capability or dependency edge was added;
- Headless Gate uses contract CI freshness and later physical Targets require
  fresh device evidence;
- Phase exit independently requires complete lifecycle heads.

Use section-bounded parsing rather than whole-file occurrence counts because
cross-references are intentionally repeated.

- [ ] **Step 7: Verify campaign completeness and arithmetic**

```powershell
$path = 'docs/architecture/04-runtime/performance-capacity.md'
$scenarios = @(
  'sequential_motion',
  'position_only_projection',
  'lifetime_only_scan',
  'structural_burst',
  'archetype_fragmentation',
  'query_cache_invalidation'
)
foreach ($scenario in $scenarios) {
  rg -q --fixed-strings $scenario $path
  $rgExit = $LASTEXITCODE
  switch ($rgExit) {
    0 { break }
    1 { throw "Missing scenario: $scenario" }
    default { throw "rg failed while checking scenario '$scenario' (exit $rgExit)." }
  }
}
$metricCount = (rg -o 'metric\.runtime\.ecs\.[a-z0-9-]+' $path | Sort-Object -Unique | Measure-Object).Count
if ($metricCount -ne 35) { throw "Expected 35 metrics, got $metricCount" }
$runs = 3 * 6 * 5
$seconds = ($runs * 120) + (3 * 600)
if ($runs -ne 90 -or $seconds -ne 12600) { throw 'Campaign arithmetic mismatch.' }
```

Expected: six scenarios, 35 metrics, 90 measured runs, three soaks, and 12,600
measured seconds.

- [ ] **Step 8: Verify all 15 Diagnostic pairs**

Check these owner counts:

```powershell
$diagnosticIds = @(
  'diagnostic.memory.value-transfer-binding-missing',
  'diagnostic.memory.value-transfer-invalid',
  'diagnostic.memory.hot-callback-allocation',
  'diagnostic.memory.hot-callback-upstream-fallback',
  'diagnostic.runtime.ecs-component-schema-revision-required',
  'diagnostic.runtime.ecs-chunk-capacity-zero',
  'diagnostic.runtime.ecs-archetype-permutation-unbounded',
  'diagnostic.runtime.ecs-query-cache-invalid',
  'diagnostic.runtime.ecs-structural-mutation-during-iteration',
  'diagnostic.runtime.ecs-structural-capacity-exceeded',
  'diagnostic.performance.ecs-required-metric-missing',
  'diagnostic.product.ecs-target-receipt-mismatch',
  'diagnostic.compatibility.ecs-consumer-inventory-unresolved',
  'diagnostic.compatibility.ecs-retained-external-consumer',
  'diagnostic.product.ecs-data-oriented-core-failed'
)
foreach ($id in $diagnosticIds) {
  $owners = @(rg -l --fixed-strings $id @(
    'docs/architecture/02-foundation/memory-pointers.md',
    'docs/architecture/04-runtime/entity-component-system.md',
    'docs/architecture/04-runtime/performance-capacity.md',
    'docs/architecture/02-foundation/compatibility-evolution.md',
    'docs/architecture/00-product/product-plan.md'
  ))
  if ($owners.Count -lt 1) { throw "Diagnostic not registered: $id" }
}
```

Manually verify each ID, uppercase hyphenated code, canonical owner, typed
argument list, and required result against design §13. References in consumer
documents do not become registrations.

- [ ] **Step 9: Verify C ABI and Shipping-path exclusions**

```powershell
$native = 'docs/architecture/03-authoring/native-game-module.md'
foreach ($forbidden in @('std::span','Result<T>','std::pmr','shared_ptr')) {
  $abiLines = rg -n 'C ABI' $native
  if (-not $abiLines) { throw 'C ABI section missing.' }
}
```

Perform an actual section-bounded review: the C ABI allowed-shape block must
exclude C++ references, `std::span`, STL／PMR, `Result<T>`, exceptions, and
owner wrappers while the generated C++ adapter consumes the policy.

Across current Shipping success paths, confirm absence of old aliases,
old reader／writer, AoS／sparse-set／object graph fallback, Runtime
`shared_ptr`, and direct structural mutation. Mentions in prohibition,
Compatibility removal inventory, and benchmark comparison are allowed.

- [ ] **Step 10: Verify no normative duplication in indirect consumers**

First verify that AI Verification／Provenance defines the two Product release
evidence-class predicates exactly once, consumes exact Windows／Android／Apple
owner Receipts, and does not own Target package semantics.

Then review Governance, Toolchain, Project State, Asset Lifecycle, Persistence,
Simulation, Rendering, Platform, Pack, and Architecture Index:

- owner-transfer and Receipt-freshness rules still cover the new refs;
- exact compiler／SDK versions remain Toolchain-owned;
- Runtime addresses, rows, spans, and leases are not persistent identity;
- no indirect document defines the three new schemas, sampling thresholds, or
  Product risk row;
- edit an indirect document only for a discovered conflicting owner statement
  or broken direct dependency, and commit that correction alone.

- [ ] **Step 11: Review whitespace, scope, and history**

```powershell
$changed = @(
  'docs/architecture/00-product/product-plan.md',
  'docs/architecture/01-governance/ai-verification-provenance.md',
  'docs/architecture/02-foundation/core-architecture.md',
  'docs/architecture/02-foundation/executable-contracts.md',
  'docs/architecture/02-foundation/memory-pointers.md',
  'docs/architecture/02-foundation/compatibility-evolution.md',
  'docs/architecture/02-foundation/cpp23-modules.md',
  'docs/architecture/03-authoring/gameplay-programming-model.md',
  'docs/architecture/03-authoring/native-game-module.md',
  'docs/architecture/04-runtime/entity-component-system.md',
  'docs/architecture/04-runtime/performance-capacity.md',
  'docs/architecture/04-runtime/scheduling-lifetime.md',
  'docs/architecture/04-runtime/runtime-package.md',
  'docs/architecture/04-runtime/debugging-observability-replay.md',
  'docs/architecture/decisions/2026-07-22-runtime-ecs-contract.md'
)
$firstTaskCommit = git log --format=%H --grep='^docs: route data-oriented foundation contracts$' -1
if (-not $firstTaskCommit) { throw 'First plan commit was not found.' }
$baselineCommit = git rev-parse "$firstTaskCommit^"
git diff --check "$baselineCommit..HEAD" -- $changed
git status --short
git log --oneline -15
```

Expected: no whitespace errors; fourteen scoped document commits if every
planned file required a change, plus the one pre-existing Product owner
correction; unrelated user changes remain intact.

- [ ] **Step 12: Record the completion report**

Report:

- canonical owner changes;
- direct-consumer bindings;
- Product destination rows and non-activation status;
- Consumer Inventory status and any external-consumer stop;
- campaign counts, duration, and result;
- verification commands and exact outcomes;
- any indirect document changed after conflict discovery;
- remaining risks and the next authorized activation step.
