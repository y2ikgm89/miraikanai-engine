# Miraikanai Engine Architecture Decision Log

This README is navigation/template only and does not itself make an Architecture decision.

## 1. Purpose

- This directory records why architecturally significant choices were made.
- Current Schema, fixed values, runtime behavior, Gates, and qualification remain in Owner documents.
- The log is append-only after a Decision becomes `normative` or `rejected`.

## 2. Lifecycle

| 状態 | Meaning | Allowed changes |
|---|---|---|
| `review` | Proposed | Review revisions |
| `normative` | Accepted | Relationship/status metadata only |
| `rejected` | Rejected | Relationship/status metadata only |
| `superseded` | Replaced by a newer Decision | Relationship metadata only |

## 3. Decision Log

| Date | Decision | Status | Scope |
|---|---|---|---|
| 2026-07-21 | [Architecture Document System Restructure](2026-07-21-document-system-restructure.md) | `normative` | document-system restructure |
| 2026-07-22 | [Runtime ECS Contract](2026-07-22-runtime-ecs-contract.md) | `review` | Engine-owned ECS choice |
| 2026-07-27 | [Architecture Decision Log Governance](2026-07-27-architecture-decision-log-governance.md) | `normative` | ADR lifecycle and authority separation |
| 2026-07-28 | [AI-readable Asset／Memory／Async Loading Alignment](2026-07-28-ai-asset-memory-async-alignment.md) | `review` | Asset import, memory lifecycle, async publication |
| 2026-07-28 | [Simulation Cadence／Presentation Separation](2026-07-28-simulation-cadence-presentation-separation.md) | `review` | Runtime timing authority separation |
| 2026-07-29 | [Product Lifecycle／Product Security Ownership](2026-07-29-product-lifecycle-security-ownership.md) | `review` | Product lifecycle and product-wide security ownership |
| 2026-07-29 | [Advanced Rendering／Multiplayer Ownership](2026-07-29-advanced-rendering-multiplayer-ownership.md) | `review` | Advanced light transport、Terrain／Foliage、Transport、Authority／Replication ownership |
| 2026-07-29 | [Android Compile／Target SDK and Vulkan Profile Baseline](2026-07-29-android-release-baseline.md) | `review` | Android SDK role separation and required／optional Vulkan Profile baseline |
| 2026-07-30 | [Product Release／Publication Authority Ownership](2026-07-30-product-release-publication-authority.md) | `review` | Requirement projection、signed release authorization、Platform publication、Product completion authority separation |
| 2026-07-30 | [C++23 Header Shipping／Toolchain Baseline](2026-07-30-cxx23-header-shipping-toolchain-baseline.md) | `review` | initial V1 C++23 Header public surface、Target Shipping frontend、Windows minimum target baseline |
| 2026-07-30 | [Product Data Flow MCD Kind](2026-07-30-product-data-flow-mcd-kind.md) | `review` | Product data-flow identity、MCD common kind、Privacy payload ownership separation |
| 2026-07-30 | [Verification Semantic Admissibility](2026-07-30-verification-semantic-admissibility.md) | `review` | generic verification scope、subject contract、branch and semantic admissibility |
| 2026-08-03 | [AI-native C++ Product Identity](2026-08-03-ai-native-cpp-product-identity.md) | `review` | AI-native C++ product identity、independent design、initial V1 clean-break |
| 2026-08-03 | [Runtime ECS Static Definition／Entity Reference Boundary](2026-08-03-runtime-ecs-static-and-entity-reference-boundary.md) | `review` | static phase identity、snapshot-bound／cross-advance Entity Refのlifetime分離 |
| 2026-08-03 | [Initial Morph Capability Boundary](2026-08-03-initial-morph-capability-boundary.md) | `review` | initial V1／C1／C2からMorphを除外しFuture end-to-end closureへ分離 |
| 2026-08-03 | [glTF Import Dependency Baseline](2026-08-03-gltf-import-dependency-baseline.md) | `review` | cgltf／MikkTSpace／Khronos Validatorのrole分離、single-parser、materialization Gate |
| 2026-08-03 | [MCP Current Protocol Baseline](2026-08-03-mcp-current-protocol-baseline.md) | `review` | current protocol singleton、legacy lifecycle非採用、materialization境界 |
| 2026-08-03 | [Android Adaptive Game Window Baseline](2026-08-03-android-adaptive-game-window-baseline.md) | `review` | game categoryとadaptive orientation／resizable windowの一意なbaseline |

## 4. Template

1. Required Architecture header fields.
2. Decision owner document, Decision date, Supersedes.
3. Context.
4. Decision drivers.
5. Considered options.
6. Decision.
7. Consequences.
8. Canonical Owner documents.
9. Supersedes/Superseded by.
10. Official or primary sources.

## 5. Update rules

- One significant decision per file.
- Name each file `YYYY-MM-DD-<slug>.md` and assign the stable document ID `mirakan.decision.<slug>`.
- Allowed transitions are `review -> normative`, `review -> rejected`, and `normative -> superseded`.
- `rejected` and `superseded` are terminal states.
- Do not rewrite `normative`／`rejected` Decision bodies.
- Create a new Decision for a changed choice. Record `Superseded by` in the old Decision and `Supersedes` in the new Decision, each with the counterpart's stable document ID and relative Markdown link.
- Do not use Decision text as the sole authority for a current Contract.
