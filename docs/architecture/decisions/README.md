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
